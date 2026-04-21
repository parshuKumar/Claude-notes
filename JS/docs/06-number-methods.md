# 06 — Number Methods

## What is this?
JavaScript has a single `number` type for all numeric values — integers, decimals, negative numbers, and special values like `Infinity` and `NaN`. On top of that, JS gives you two toolkits: methods directly on number values (like `.toFixed()`), and the `Math` object — a built-in library of mathematical functions. Think of `Math` like a scientific calculator that's always in your pocket, and number methods as the formatting buttons on a basic calculator.

## Why does it matter?
Any app that touches money, scores, measurements, or statistics needs number handling done right. Rounding errors in financial calculations, failing to detect invalid input, or displaying `3.9999999999` instead of `4.00` — these are real production bugs caused by not knowing how JS handles numbers.

---

## How JS stores numbers — float precision (read this first)

JavaScript uses **64-bit floating point** (IEEE 754) to store all numbers. This means some decimal numbers cannot be represented exactly in binary — the same way you can't write 1/3 as a finite decimal.

```js
0.1 + 0.2             // 0.30000000000000004 — NOT 0.3
0.1 + 0.2 === 0.3     // false — this will shock you the first time
```

This is not a JS bug — it's a consequence of how binary floating point works. Every language that uses IEEE 754 (Python, Java, C#) has this same issue.

**How to handle it:**
```js
// Option 1 — round to known decimal places
(0.1 + 0.2).toFixed(2)           // "0.30" — string!
Number((0.1 + 0.2).toFixed(2))   // 0.3   — back to number

// Option 2 — work in integers (multiply up, divide down)
const priceCents = 10 + 20;      // 30 cents — no floating point issue
const priceDollars = priceCents / 100;   // 0.3 — exact

// Option 3 — use a precision library (decimal.js, big.js) for financial apps
```

---

## Number instance methods

These are called on a number value or variable. They always return a **string** (not a number) — important to know.

### `toFixed(digits)`
Rounds to `digits` decimal places and returns a **string**.

```js
const price = 19.9;
price.toFixed(2)         // "19.90" — string, padded with zero
price.toFixed(0)         // "20"    — rounds to nearest integer
price.toFixed(4)         // "19.9000"

const tax = 1.005;
tax.toFixed(2)           // "1.00" — NOT "1.01" — floating point rounding quirk
// The actual stored value of 1.005 is slightly below 1.005 in binary, so it rounds down

// Always convert back to number if you need to do more math
const roundedPrice = Number(price.toFixed(2));   // 19.9 (number)
const asFloat      = parseFloat(price.toFixed(2));  // 19.9 (number) — same thing
```

### `toPrecision(digits)`
Formats a number to a total number of **significant digits** (not decimal places).
```js
const n = 123.456;
n.toPrecision(5)    // "123.46" — 5 significant digits total
n.toPrecision(2)    // "1.2e+2" — switches to exponential notation when needed
n.toPrecision(7)    // "123.4560" — pads with zero
```

### `toString(radix?)`
Converts a number to a string. Optional `radix` converts to different bases.
```js
(255).toString()      // "255"  — decimal (base 10)
(255).toString(16)    // "ff"   — hexadecimal (base 16)
(255).toString(2)     // "11111111" — binary (base 2)
(255).toString(8)     // "377"  — octal (base 8)
(10).toString(2)      // "1010"
```

### `valueOf()`
Returns the primitive number value of a Number object. Rarely used directly.
```js
const n = new Number(42);    // Number object (avoid this — use primitives)
n.valueOf()                  // 42 — primitive
```

---

## Number static methods

Called on `Number` itself, not on a value.

### `Number.isNaN(value)`
Returns `true` only if the value is exactly `NaN`. Does **not** coerce.

```js
Number.isNaN(NaN)           // true
Number.isNaN("NaN")         // false — the string "NaN" is not NaN
Number.isNaN(undefined)     // false — undefined is not NaN
Number.isNaN(0 / 0)         // true  — 0/0 produces NaN

// The old global isNaN() is dangerous — it coerces first
isNaN("hello")              // true  — "hello" gets coerced to NaN, then checked
Number.isNaN("hello")       // false — no coercion, "hello" is not literally NaN
// Always use Number.isNaN() — never the global isNaN()
```

### `Number.isFinite(value)`
Returns `true` if the value is a finite number (not `Infinity`, `-Infinity`, or `NaN`).
```js
Number.isFinite(42)          // true
Number.isFinite(Infinity)    // false
Number.isFinite(-Infinity)   // false
Number.isFinite(NaN)         // false
Number.isFinite("42")        // false — no coercion (unlike global isFinite())
Number.isFinite(1 / 0)       // false — 1/0 is Infinity in JS (no ZeroDivisionError!)
```

### `Number.isInteger(value)`
```js
Number.isInteger(42)      // true
Number.isInteger(42.0)    // true  — 42.0 is the same as 42 in JS
Number.isInteger(42.5)    // false
Number.isInteger("42")    // false — no coercion
```

### `Number.isSafeInteger(value)`
Safe integers are those that can be exactly represented in IEEE 754: from -(2^53 - 1) to (2^53 - 1).
```js
Number.isSafeInteger(9007199254740991)    // true  — Number.MAX_SAFE_INTEGER
Number.isSafeInteger(9007199254740992)    // false — beyond safe range
Number.isSafeInteger(42)                  // true
```

### `Number.parseInt(string, radix?)` and `Number.parseFloat(string)`
Same as the global `parseInt` and `parseFloat`, but attached to `Number`. Prefer these.

---

## Global parsing functions

### `parseInt(string, radix)`
Reads a string from left to right and extracts an **integer**, stopping at the first non-numeric character. Always pass `10` as the radix to avoid unexpected octal/hex parsing.

```js
parseInt("42")         // 42
parseInt("42.9")       // 42    — stops at decimal point
parseInt("42px")       // 42    — stops at "p"
parseInt("px42")       // NaN   — starts with non-numeric
parseInt("")           // NaN   — empty string
parseInt("0xFF", 16)   // 255   — hexadecimal
parseInt("1010", 2)    // 10    — binary to decimal
parseInt("10")         // 10    — always pass radix 10 for safety!
parseInt("010")        // 10    — some older engines parse "010" as octal (8) — radix prevents this
parseInt("010", 10)    // 10    — explicit base 10, safe everywhere
```

### `parseFloat(string)`
Like `parseInt` but keeps the decimal portion.
```js
parseFloat("3.14")        // 3.14
parseFloat("3.14abc")     // 3.14 — stops at non-numeric
parseFloat("abc3.14")     // NaN  — starts with non-numeric
parseFloat("  3.14  ")    // 3.14 — leading/trailing whitespace is ignored
parseFloat(true)          // NaN  — booleans aren't parseable
```

### `Number(value)` — strict conversion
Converts a value to a number. Returns `NaN` on failure. Does NOT stop partway through like `parseInt`.
```js
Number("42")       // 42
Number("42px")     // NaN   — strict, no partial parsing
Number("")         // 0     — empty string becomes 0 (gotcha!)
Number(" ")        // 0     — whitespace-only string becomes 0
Number(true)       // 1
Number(false)      // 0
Number(null)       // 0
Number(undefined)  // NaN
Number([5])        // 5     — single-element array
Number([])         // 0     — empty array becomes 0
```

---

## The `Math` object

`Math` is a built-in object (not a class, not a function) — just a namespace with properties and methods.

### Rounding methods — know all four, they behave very differently
```js
Math.round(4.5)    // 5  — rounds to nearest, ties go up
Math.round(4.4)    // 4
Math.round(-4.5)   // -4 — ties go UP (toward +Infinity), not toward zero

Math.floor(4.9)    // 4  — always rounds DOWN (toward -Infinity)
Math.floor(-4.1)   // -5 — "down" means toward -Infinity, not toward zero!

Math.ceil(4.1)     // 5  — always rounds UP (toward +Infinity)
Math.ceil(-4.9)    // -4

Math.trunc(4.9)    // 4  — removes the decimal part, always toward zero
Math.trunc(-4.9)   // -4 — toward zero, unlike floor
```

**The difference between `floor` and `trunc` for negatives:**
```js
Math.floor(-3.2)   // -4 — floor goes DOWN
Math.trunc(-3.2)   // -3 — trunc goes toward ZERO
// For positive numbers they're identical. For negatives, they differ.
```

### `Math.random()`
Returns a pseudo-random float between **0 (inclusive)** and **1 (exclusive)**.
```js
Math.random()           // 0.7341893... — different every time

// Random integer between 0 and n-1
Math.floor(Math.random() * 6)     // 0, 1, 2, 3, 4, or 5

// Random integer between min and max (inclusive)
function randomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
randomInt(1, 10)    // 1, 2, 3, 4, 5, 6, 7, 8, 9, or 10
randomInt(1, 6)     // simulates a dice roll
```

### Min, Max, and Absolute value
```js
Math.max(3, 1, 4, 1, 5, 9)   // 9
Math.min(3, 1, 4, 1, 5, 9)   // 1
Math.max()                    // -Infinity — no arguments
Math.min()                    // Infinity  — no arguments

// With an array — use spread operator
const scores = [85, 92, 78, 95, 88];
Math.max(...scores)   // 95
Math.min(...scores)   // 78

Math.abs(-42)    // 42 — absolute value
Math.abs(42)     // 42
```

### Power, Square root, and Cube root
```js
Math.pow(2, 10)      // 1024 — same as 2 ** 10
Math.sqrt(144)       // 12
Math.sqrt(2)         // 1.4142135623730951
Math.cbrt(27)        // 3  — cube root
Math.cbrt(125)       // 5
```

### Logarithms and constants
```js
Math.PI          // 3.141592653589793
Math.E           // 2.718281828459045 — Euler's number
Math.LN2         // 0.6931471805599453 — natural log of 2
Math.SQRT2       // 1.4142135623730951

Math.log(Math.E)  // 1   — natural logarithm
Math.log2(8)      // 3   — log base 2
Math.log10(1000)  // 3   — log base 10
```

### Sign and integer check
```js
Math.sign(-5)     // -1
Math.sign(0)      // 0
Math.sign(5)      // 1
// Useful when you need direction without magnitude
```

---

## Number constants
```js
Number.MAX_VALUE           // 1.7976931348623157e+308 — largest representable number
Number.MIN_VALUE           // 5e-324 — smallest positive number (close to 0, NOT most negative)
Number.MAX_SAFE_INTEGER    // 9007199254740991  — 2^53 - 1
Number.MIN_SAFE_INTEGER    // -9007199254740991
Number.POSITIVE_INFINITY   // Infinity
Number.NEGATIVE_INFINITY   // -Infinity
Number.NaN                 // NaN
Number.EPSILON             // 2.220446049250313e-16 — smallest difference between two numbers
```

**`Number.EPSILON` for safe float comparison:**
```js
// WRONG — floating point makes this unreliable
0.1 + 0.2 === 0.3    // false

// RIGHT — check if difference is smaller than epsilon
Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON   // true — effectively equal
```

---

## Example 1 — basic

```js
// Formatting prices for a receipt
const unitPrice = 49.999;
const quantity  = 3;

const subtotal  = unitPrice * quantity;          // 149.997
const tax       = subtotal * 0.18;               // 26.99946
const total     = subtotal + tax;                // 176.99646

console.log(`Subtotal: $${subtotal.toFixed(2)}`);   // "Subtotal: $150.00"
console.log(`Tax (18%): $${tax.toFixed(2)}`);        // "Tax (18%): $27.00"
console.log(`Total: $${total.toFixed(2)}`);          // "Total: $177.00"

// Checking input validity
const userInput = "abc";
const parsed    = parseFloat(userInput);
if (Number.isNaN(parsed)) {
  console.log("Invalid price entered.");
}
```

---

## Example 2 — real world

```js
// A dice-based game score calculator
function rollDice(sides = 6) {
  return Math.floor(Math.random() * sides) + 1;
}

const rolls       = [rollDice(), rollDice(), rollDice()];
const total       = rolls.reduce((sum, n) => sum + n, 0);
const average     = total / rolls.length;
const highest     = Math.max(...rolls);
const isAllSame   = rolls.every(n => n === rolls[0]);

console.log(`Rolls: ${rolls.join(", ")}`);
console.log(`Total: ${total}`);
console.log(`Average: ${average.toFixed(1)}`);
console.log(`Highest: ${highest}`);
console.log(`Jackpot (all same): ${isAllSame}`);

// Pixel dimension formatter
function toPx(value) {
  return `${Math.round(value)}px`;
}

console.log(toPx(14.7));    // "15px"
console.log(toPx(14.2));    // "14px"
```

---

## Common mistakes

### 1. Forgetting `toFixed()` returns a string
```js
// WRONG — adds strings instead of numbers
const a = (1.005).toFixed(2);    // "1.00" — string
const b = (2.005).toFixed(2);    // "2.01" — string
console.log(a + b);              // "1.002.01" — string concatenation!

// RIGHT — convert back to number first
console.log(Number(a) + Number(b));   // 3.01
```

### 2. Using global `isNaN()` instead of `Number.isNaN()`
```js
// WRONG — global isNaN coerces the argument first
isNaN("hello")       // true — "hello" coerces to NaN, passes check
isNaN(undefined)     // true — undefined coerces to NaN

// RIGHT — Number.isNaN does not coerce
Number.isNaN("hello")       // false — "hello" is a string, not NaN
Number.isNaN(undefined)     // false — undefined is not NaN
Number.isNaN(NaN)           // true  — only thing that passes
```

### 3. `Math.round` vs `Math.floor` confusion with negatives
```js
// WRONG assumption — floor always goes "toward zero"
Math.floor(-2.1)   // -3 — NOT -2! Floor always goes toward -Infinity

// RIGHT mental model
Math.floor(-2.1)   // -3 — down toward -Infinity
Math.trunc(-2.1)   // -2 — toward zero
Math.ceil(-2.9)    // -2 — up toward +Infinity
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `1 / 0` doesn't throw in JS
```js
console.log(1 / 0);    // Infinity — not an error!
console.log(-1 / 0);   // -Infinity
console.log(0 / 0);    // NaN
// Always validate denominators before dividing in real code
const safeDiv = (a, b) => b !== 0 ? a / b : null;
```

### Trick 2 — `Number("")` is `0`, not `NaN`
```js
Number("")     // 0 — empty string silently becomes 0
Number(" ")    // 0 — whitespace-only also becomes 0
// This means you can't use Number() alone to detect empty input
// Always check if the string is empty first:
const input = "";
const value = input.trim() === "" ? NaN : Number(input);
```

### Trick 3 — Floating point with `toFixed()` is not always predictable
```js
(1.005).toFixed(2)    // "1.00" — not "1.01"!
(1.015).toFixed(2)    // "1.01" — not "1.02"!
// This is a floating point representation issue — 1.005 is stored as slightly less
// For financial calculations, multiply to integers or use a library
```

### Trick 4 — `Math.random()` is NOT cryptographically secure
```js
// For random game elements, shuffles, test data — fine
Math.random()

// For security tokens, passwords, session IDs — NEVER use Math.random()
// Use: crypto.getRandomValues() in browser, or crypto.randomBytes() in Node.js
```

### Trick 5 — `parseInt` without radix can surprise you
```js
parseInt("08")     // 8 in modern JS, but was 0 in old JS (octal!)
parseInt("0x10")   // 16 — auto-detects hex prefix
parseInt("010")    // 10 in modern JS, 8 in some old engines
// Always pass radix: parseInt("08", 10) → always 8, everywhere
```

---

## Practice exercises

### Exercise 1 — easy
Given:
```js
const rawTemperatures = ["22.6", "18.333", "30.1", "-5.89", "abc"];
```

For each item:
1. Parse it to a number
2. If it's not a valid number (`NaN`), log `"Invalid reading"`
3. If it's valid, log the temperature rounded to 1 decimal place with `"°C"` appended

Expected output:
```
22.6°C
18.3°C
30.1°C
-5.9°C
Invalid reading
```

```js
// Write your code here
```

---

### Exercise 2 — medium
Build a tip calculator function (don't worry about function syntax — just write the logic for one specific case):

```js
const billAmount    = "84.50";    // from a form — it's a string
const tipPercent    = "15";       // from a form — it's a string
const numberOfPeople = 3;
```

Calculate and log:
1. Bill as a number (validate it — if NaN, log error and stop)
2. Tip amount (rounded to 2 decimal places)
3. Total bill with tip
4. Amount per person (rounded up to nearest cent — use `Math.ceil` with factor of 100)
5. Whether each person's share is above $30 (true/false)

Format all money values with 2 decimal places and a `$` prefix.
```js
// Write your code here
```

---

### Exercise 3 — hard
You are building a statistics module for a quiz app. Given:

```js
const scores = [72, 88, 95, 61, 88, 74, 100, 55, 88, 79];
```

Calculate and log ALL of the following:
1. **Total** — sum of all scores
2. **Average** — rounded to 2 decimal places
3. **Highest** and **Lowest** score
4. **Range** — highest minus lowest
5. **Median** — sort the array, take the middle value (or average of two middle values for even-length arrays)
6. **Mode** — the score that appears most often (88 appears 3 times)
7. **Pass rate** — percentage of scores >= 70, formatted as `"X.XX%"`
8. **Grade distribution** — count how many are: A (90+), B (75–89), C (60–74), F (below 60)

Use `Math` methods where appropriate. No external libraries.
```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Number instance methods (return strings):**
```
n.toFixed(2)        → "3.14"   — fixed decimal places
n.toPrecision(4)    → "3.142"  — significant digits
n.toString(16)      → "ff"     — convert to different base
```

**Number static methods:**
```
Number.isNaN(x)          → true only if x is literally NaN (no coercion)
Number.isFinite(x)       → true if finite number
Number.isInteger(x)      → true if whole number
Number.isSafeInteger(x)  → true if within 2^53 - 1
```

**Parsing (string → number):**
```
Number("42")         → 42    (strict — NaN on any non-numeric chars)
parseInt("42px", 10) → 42    (lenient — stops at first non-numeric)
parseFloat("3.14x")  → 3.14  (lenient — stops at first non-numeric)
Number("")           → 0     (gotcha — empty string is 0, not NaN)
```

**Math rounding:**
```
Math.round(x)   → nearest integer (ties go up)
Math.floor(x)   → always down (toward -Infinity)
Math.ceil(x)    → always up   (toward +Infinity)
Math.trunc(x)   → toward zero (removes decimal)
```

**Math utility:**
```
Math.max(a, b, ...)   → largest
Math.min(a, b, ...)   → smallest
Math.abs(x)           → absolute value
Math.sqrt(x)          → square root
Math.pow(x, y)        → x to the power y (same as x ** y)
Math.random()         → 0 to <1 (not crypto-safe)
Math.sign(x)          → -1, 0, or 1
```

**Math constants:**
```
Math.PI    → 3.14159...
Math.E     → 2.71828...
```

**Key rules:**
- `toFixed()` returns a **string** — wrap with `Number()` if you need a number
- Always use `Number.isNaN()` — never the global `isNaN()`
- `Math.floor(-2.1)` → `-3`, not `-2` — floor goes toward -Infinity
- `0 / 0` → `NaN`, `1 / 0` → `Infinity` — no errors thrown
- Use `Math.ceil(x * 100) / 100` to round UP to 2 decimal places

---

## Connected topics
- **05 — String Methods** — `toFixed()` returns a string; parsing functions bridge the two topics
- **03 — Type Coercion** — `Number()`, `parseInt()`, and `parseFloat()` are explicit coercion tools
- **29 — reduce** — used to sum arrays; pairs naturally with Math methods for statistics
