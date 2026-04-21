# 03 — Type Coercion

## What is this?
Type coercion is JavaScript automatically converting a value from one type to another behind the scenes — without you asking it to. Think of it like a translator at a meeting: when two people speak different languages, the translator silently converts what one says so the other can understand. Sometimes the translation is perfect. Sometimes it's hilariously wrong. JS does this silently, which is why it causes so many bugs.

## Why does it matter?
Coercion is behind some of the most infamous JS bugs — `"5" + 1` giving `"51"`, `0 == false` being `true`, and `[] == false` somehow also being `true`. Every JS developer has been burned by coercion at least once. Understanding it means you can predict JS's behaviour instead of being surprised by it, and you'll know exactly when to use `==` vs `===`.

---

## Two kinds of coercion

### Implicit coercion — JS does it automatically
```js
"5" + 1        // "51"  — JS converted 1 to a string and concatenated
"5" - 1        // 4     — JS converted "5" to a number and subtracted
true + 1       // 2     — JS converted true to 1
false + 1      // 1     — JS converted false to 0
null + 1       // 1     — JS converted null to 0
undefined + 1  // NaN   — JS tried to convert undefined to a number, failed
```

### Explicit coercion — YOU do it intentionally
```js
Number("42")       // 42      — string to number
String(42)         // "42"    — number to string
Boolean(0)         // false   — number to boolean
parseInt("42px")   // 42      — string to integer, stops at non-numeric
parseFloat("3.14") // 3.14    — string to float
```

---

## Syntax

```js
// Implicit — happens automatically inside expressions
let result = "10" - 2;          // 8   — string coerced to number
let joined = "Price: " + 99;    // "Price: 99" — number coerced to string

// Explicit — you control the conversion
let age     = Number("25");     // 25
let label   = String(100);      // "100"
let isValid = Boolean("");      // false
```

---

## How it works — line by line

- `"10" - 2` → JS sees `-` and knows it's arithmetic. It tries to convert both sides to numbers. `"10"` becomes `10`, so the result is `8`.
- `"Price: " + 99` → JS sees `+` with a string on the left. It switches to **string concatenation mode** and converts `99` to `"99"`, giving `"Price: 99"`.
- `Number("25")` → You explicitly ask JS to convert `"25"` to a number. Returns `25`.
- `Boolean("")` → You explicitly ask if an empty string is truthy. It's falsy, so returns `false`.

---

## The `+` operator is the dangerous one

`+` has **two jobs**: addition (numbers) and concatenation (strings).

The rule JS follows:
> If **either** operand is a string, `+` becomes concatenation. Everything else gets converted to a string.

```js
1 + 2           // 3        — both numbers, addition
"1" + 2         // "12"     — left is string, concatenation
1 + "2"         // "12"     — right is string, concatenation
1 + 2 + "3"     // "33"     — evaluated left to right: (1+2)=3, then 3+"3"="33"
"1" + 2 + 3     // "123"    — "1"+2="12", then "12"+3="123"
```

Every other arithmetic operator (`-`, `*`, `/`, `%`) does NOT have the string mode — they always try to convert to number:
```js
"10" - 2    // 8
"10" * 2    // 20
"10" / 2    // 5
"10" % 3    // 1
```

---

## `==` vs `===` — the most important distinction in JS

### `===` — Strict equality (no coercion)
Compares **both value AND type**. If the types differ, it returns `false` immediately. No conversion.

```js
5 === 5          // true  — same value, same type
5 === "5"        // false — same value, different types
null === null    // true
null === undefined  // false — different types
```

### `==` — Loose equality (with coercion)
Compares values **after** attempting type conversion. JS follows a specific set of rules:

```js
5 == "5"         // true  — "5" is converted to 5
0 == false       // true  — false is converted to 0
1 == true        // true  — true is converted to 1
null == undefined // true — special rule: these two only equal each other
null == 0        // false — null only equals null or undefined, nothing else
"" == false      // true  — both convert to 0
```

### The `==` conversion rules (simplified)
1. If both types are the same → compare directly (no coercion needed)
2. `null == undefined` → always `true` (and nothing else equals `null` or `undefined` loosely)
3. If comparing number and string → convert string to number, then compare
4. If comparing boolean and anything → convert boolean to number first (`true` → 1, `false` → 0)
5. If comparing object and primitive → convert object to primitive first

### The rule: always use `===` in your code
```js
// Use == only in ONE legitimate case: checking for both null and undefined at once
if (value == null) { ... }   // true for both null AND undefined — intentional shorthand
```

---

## `typeof` and coercion together

`typeof` always returns a string. This is important when you use it in comparisons:

```js
typeof 42 === "number"    // true  — correct
typeof 42 == "number"     // true  — also works, but use === by habit
typeof 42 === "Number"    // false — case sensitive! "number" not "Number"
```

---

## How JS converts to each type

### To Number:
```js
Number("")          // 0
Number("  ")        // 0   — whitespace-only strings become 0
Number("42")        // 42
Number("42px")      // NaN — has non-numeric characters
Number(true)        // 1
Number(false)       // 0
Number(null)        // 0
Number(undefined)   // NaN
Number([])          // 0   — empty array becomes 0 (bizarre)
Number([5])         // 5   — single-element array unwraps
Number([1,2])       // NaN — multi-element array can't convert
Number({})          // NaN
```

### To String:
```js
String(42)          // "42"
String(true)        // "true"
String(false)       // "false"
String(null)        // "null"
String(undefined)   // "undefined"
String([1,2,3])     // "1,2,3"
String({})          // "[object Object]"
```

### To Boolean:
There are exactly **6 falsy values** in JS. Everything else is truthy.
```js
Boolean(false)      // false
Boolean(0)          // false
Boolean(-0)         // false
Boolean(0n)         // false — BigInt zero
Boolean("")         // false — empty string
Boolean(null)       // false
Boolean(undefined)  // false
Boolean(NaN)        // false

// Everything else is truthy, including:
Boolean("0")        // true  — string "0" is NOT falsy (it's a non-empty string)
Boolean(" ")        // true  — space is not empty
Boolean([])         // true  — empty array is truthy!
Boolean({})         // true  — empty object is truthy!
Boolean(-1)         // true  — any non-zero number
```

This connects directly to topic 08 (Truthy/Falsy) — but knowing it now helps you understand coercion fully.

---

## Example 1 — basic

```js
// Seeing coercion in action — predict each result before looking at the comment
console.log("5" + 1);       // "51"  — + with string = concatenation
console.log("5" - 1);       // 4     — - always numeric, "5" coerced to 5
console.log(true + 1);      // 2     — true coerced to 1
console.log(false + 1);     // 1     — false coerced to 0
console.log(null + 1);      // 1     — null coerced to 0
console.log(undefined + 1); // NaN   — undefined coerces to NaN
console.log("" + 1);        // "1"   — empty string + number = string
console.log(+"42");          // 42    — unary + converts string to number
console.log(+true);          // 1
console.log(+false);         // 0
console.log(+null);          // 0
console.log(+undefined);     // NaN
```

---

## Example 2 — real world

```js
// A form reads ALL values as strings — classic source of coercion bugs
const rawQuantity = "3";    // came from an HTML input — it's a string
const pricePerUnit = 29.99;  // this is a number

// BUG — string + number = concatenation
const wrongTotal = rawQuantity + pricePerUnit;
console.log(wrongTotal);    // "329.99" — not what you want!

// FIX — explicitly convert before using
const quantity   = Number(rawQuantity);               // 3 (number)
const totalPrice = quantity * pricePerUnit;           // 89.97
console.log(`Total: $${totalPrice.toFixed(2)}`);      // "Total: $89.97"

// Another common real-world coercion trap
const userInput = "";         // user left a field blank
if (userInput == false) {
  console.log("No input");   // this runs — "" == false is true via coercion
}
// Better:
if (userInput === "") {
  console.log("No input");   // explicit and clear
}
```

---

## Common mistakes

### 1. Using `+` to add numbers when one might be a string
```js
// WRONG — input from a form is always a string
const itemsInCart = "3";
const newItem     = 1;
console.log(itemsInCart + newItem);   // "31" — string concatenation

// RIGHT — convert explicitly first
console.log(Number(itemsInCart) + newItem);   // 4
```

### 2. Using `==` and being surprised
```js
// WRONG — these look like they should be false
console.log(0 == false);    // true
console.log("" == false);   // true
console.log(null == 0);     // false — wait, but null == undefined is true?

// RIGHT — use === to avoid all coercion surprises
console.log(0 === false);   // false
console.log("" === false);  // false
```

### 3. Forgetting that `Number()` on an empty or invalid string gives 0 or NaN
```js
// WRONG assumption
console.log(Number(""));        // 0  — not NaN! Could silently corrupt a calculation
console.log(Number("hello"));   // NaN

// RIGHT — validate before converting
const raw = "";
const value = raw.trim() === "" ? null : Number(raw);   // explicitly handle empty case
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `[] + []` and `[] + {}`
These are famous JS interview questions. They look absurd but have logical explanations:
```js
[] + []    // ""       — both arrays convert to "", "" + "" = ""
[] + {}    // "[object Object]"  — [] → "", {} → "[object Object]"
{} + []    // 0 or "[object Object][]" — depends on context (block vs expression)
```
You won't write this intentionally, but you'll see it in interviews. The lesson: `+` on non-primitives triggers object-to-primitive conversion.

### Trick 2 — Unary `+` is a quick string-to-number trick
```js
const input = "42";
const num   = +input;    // 42 — shorthand for Number(input)
// Common in real code — but explicit Number() is clearer for beginners
```

### Trick 3 — Comparing across types with `>`/`<`
```js
"10" > 9     // true  — "10" coerced to 10
"10" > "9"   // false — both are strings! String comparison: "1" < "9" lexicographically
// This bites people sorting numbers stored as strings
```

### Trick 4 — `+` on arrays in template literals
```js
const tags = ["js", "web"];
console.log("Tags: " + tags);        // "Tags: js,web" — array coerced to string
console.log(`Tags: ${tags}`);        // "Tags: js,web" — same thing in template literal
// This is occasionally useful but mostly a source of bugs when you expect array behaviour
```

### Trick 5 — `parseInt` vs `Number` on edge cases
```js
parseInt("42px")     // 42  — stops at the first non-numeric character
Number("42px")       // NaN — fails completely because "42px" isn't a valid number
parseInt("")         // NaN — empty string returns NaN (unlike Number("") which gives 0)
parseInt("0x1F")     // 31  — parses hexadecimal
parseInt("011", 10)  // 11  — always pass radix 10 to avoid octal parsing in old JS
```

---

## Practice exercises

### Exercise 1 — easy
Without running the code first, predict the output of every line. Then verify by running it.

```js
console.log("10" + 5);
console.log("10" - 5);
console.log("10" * "2");
console.log(true + true);
console.log(null + 5);
console.log(undefined + 5);
console.log("" + false);
console.log(1 + 2 + "3");
console.log("1" + 2 + 3);
```

Write your predictions as comments next to each line, then log the actual results and compare.
```js
// Write your code here
```

---

### Exercise 2 — medium
A checkout form returns all values as strings. Fix the following code so all calculations are correct:

```js
const rawPrice    = "149";
const rawQuantity = "3";
const rawDiscount = "10";   // percentage

// These are all broken — fix them
const subtotal      = rawPrice * rawQuantity;       // should be 447
const discountAmt   = subtotal * rawDiscount / 100; // should be 44.7
const finalAmount   = subtotal - discountAmt;       // should be 402.3
const receiptLine   = "Total: $" + finalAmount;     // should be "Total: $402.3"

console.log(subtotal);
console.log(discountAmt);
console.log(finalAmount);
console.log(receiptLine);
```

Some of the calculations may already work due to coercion — figure out which ones and explain why. Convert explicitly where needed.
```js
// Write your code here
```

---

### Exercise 3 — hard
You are reviewing a colleague's code. There are **4 type coercion bugs** hidden in the following snippet. Find each one, explain what the bug is, what JS actually produces, and rewrite the entire block correctly:

```js
function processOrder(orderId, quantity, unitPrice, coupon) {
  // Check 1 — is quantity a valid number?
  if (quantity == true) {
    console.log("Quantity provided");
  }

  // Check 2 — is coupon missing?
  if (coupon == false) {
    console.log("No coupon applied");
  }

  // Check 3 — calculate total
  const total = unitPrice + quantity;
  console.log("Total: " + total);

  // Check 4 — compare order ID
  if (orderId == 1001) {
    console.log("Priority order");
  }
}

processOrder("1001", "2", 49.99, "");
```
```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**The `+` rule:**
> One side is a string → concatenation. Otherwise → addition.

**Arithmetic operators `-`, `*`, `/`, `%`:**
> Always convert to number. No string mode.

**`==` vs `===`:**
```
===  → no coercion. Types must match. Always use this.
==   → with coercion. Only use for null/undefined check: value == null
```

**Explicit conversion:**
```js
Number(x)      // to number (strict — NaN on failure)
parseInt(x)    // string to integer (lenient — stops at first non-numeric)
parseFloat(x)  // string to float (lenient)
String(x)      // to string
Boolean(x)     // to boolean
+x             // shorthand for Number(x) — unary plus
```

**The 6 falsy values:**
```
false, 0, -0, 0n, "", null, undefined, NaN
```

**Common coercion results to memorise:**
```
"5" + 1        → "51"   (concatenation)
"5" - 1        → 4      (numeric)
null + 1       → 1      (null → 0)
undefined + 1  → NaN    (undefined → NaN)
true + 1       → 2      (true → 1)
false + 1      → 1      (false → 0)
"" == false    → true   (both → 0)
null == undefined → true (special rule)
null == 0      → false  (null only equals null/undefined)
NaN == NaN     → false  (always)
```

---

## Connected topics
- **02 — Data Types** — coercion converts between these types; you need to know all 8 types first
- **08 — Truthy and Falsy** — coercion to boolean is how JS evaluates conditions; directly related
- **04 — Operators** — the `??` (nullish coalescing) and `||` operators rely on coercion to work
