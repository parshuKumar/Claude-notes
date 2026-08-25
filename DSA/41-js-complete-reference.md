# JAVASCRIPT — THE COMPLETE DSA REVISION REFERENCE
## Every Idiom, Every Trap, Every Missing Structure — In One Document
### Baseline: Node.js / ES2022 (LeetCode runtime) | Newer features marked inline
### Companion to Patterns 01–38 and the C++/Java References | Interview + Node CP Ready

---

## HOW TO USE THIS DOCUMENT

Your primary language is C++ (you also know Java). JavaScript is the most *different* of the
three for DSA — different enough that the gaps are what bite, not the syntax. So **every place
JS diverges from C++/Java is flagged with a `> **vs C++:**` note**, and the things JS simply
does not give you (a heap, an ordered map, a fast queue) are shipped here as ready-to-paste code.

Three ways to read it:

| Situation | Where to go |
|---|---|
| **Coming to JS from C++/Java** | Read the **C++/Java→JS mental-switch box at the end of §37**, then skim Parts I–IV |
| **Mid-problem, forgot the API** — "how do I get a min-heap in JS again?" | **§33** (Recipe Index); the Heap/Deque classes live in **§14–§15** |
| **Something is wrong or TLEs** | **§36** (Top 30 JS-DSA Mistakes) and **§32** (Debugging & Errors) |

**The rule:** if you catch yourself thinking *"I know this in C++ but not the JS way"* — it
should be in here. If it isn't, add it.

### The 10 things a C++/Java person MUST remember in JS (full list in §37)

1. **Every number is a 64-bit double** — exact integers only to `2^53`; beyond that, precision
   silently breaks (a mod-mul of two 1e9 numbers is already wrong). Use **BigInt** past 2^53.
2. **Bitwise operators truncate to 32-bit signed** — `1 << 31` is negative, `1 << 32 === 1`,
   masks past 31 bits need **BigInt**.
3. **`[10, 2, 1].sort()` → `[1, 10, 2]`** — default sort is lexicographic. Always pass `(a,b)=>a-b`.
4. **No built-in `priority_queue` / `TreeMap` / `TreeSet` / `multiset` / `deque`** — paste your own (§14–§15).
5. **`arr.shift()` is O(n)** — a naive queue is O(n²) → TLE. Use a head-index queue.
6. **`===` not `==`**; objects/arrays compare by reference (`[1] === [1]` is false).
7. **Strings are immutable, no `char` type** — use `s.charCodeAt(i)`, build output with array + `join`.
8. **Recursion overflows around ~10⁴ frames** (`RangeError`) with no big-stack escape — go iterative.
9. **`Array(n).fill([])` shares one reference**; `new Array(n)` has holes, not zeros.
10. **CP I/O:** `readFileSync(0,'utf8')` to read, buffer output and `console.log` once.

### Conventions used throughout

- Code runs on **Node.js 18+ / ES2022** unless tagged (**[ES2020]**, **[ES2023]**, …) with an availability note.
- `// =>` comments show expected output.
- `> **Gotcha:**` flags traps that cause real WA/TLE/RE.
- `> **vs C++:**` flags where JS behaves differently from the C++/Java you already know.
- Complexity is given for every structure operation and algorithm.

---

## TABLE OF CONTENTS

### PART I — CORE LANGUAGE
1. Numbers, BigInt and the IEEE-754 Reality
2. Types, Coercion and Equality
3. Variables, Scope, Hoisting and Closures
4. Functions, Arrow Functions and `this`
5. Objects, Prototypes and Classes
6. Destructuring, Spread, Rest and Modern Syntax
7. Iteration Protocols, Generators, for-of vs for-in

### PART II — ARRAYS, STRINGS AND TYPED ARRAYS
8. Array Fundamentals
9. The Complete Array Method Reference
10. String and String Methods
11. TypedArrays and Pair/Tuple Representations

### PART III — MAP, SET AND THE MISSING STRUCTURES
12. Map and Object-as-Map
13. Set
14. Stack, Queue and Deque
15. PriorityQueue / Binary Heap (JS has none — implement it)
16. Choosing the Right Structure (and the Missing-TreeMap Problem)

### PART IV — SORTING, SEARCHING AND FUNCTIONAL TOOLS
17. Sorting
18. Binary Search (roll your own)
19. Higher-Order Array Methods for Algorithms
20. Comparators and Multi-key Ordering
21. Math and Number Utilities

### PART V — BIT MANIPULATION
22. Bitwise Operators and the 32-bit Reality
23. The Complete Bit-Trick Catalog
24. BigInt Bitwise, Math.clz32, and Wide Masks

### PART VI — NUMBERS, PRECISION, BIGINT AND I/O
25. Number Precision, Safe Integers and Overflow
26. BigInt in Depth
27. Fast I/O in Node.js (the make-or-break CP topic)
28. Randomness, Time, and the Runtime

### PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT
29. The Contest Template and Structure
30. Recursion, the Call Stack, and Iterative Conversion
31. Performance and V8 Gotchas
32. Debugging, Errors and Running

### PART VIII — QUICK REFERENCE AND RECIPES
33. The "How Do I...?" Recipe Index ← **the mid-problem lookup**
34. Master Complexity Table
35. n → Required Complexity Cheat Sheet
36. The Top 30 JS-DSA Mistakes That Cost You Problems
37. 60-Second Warm-Up Drill + C++/Java→JS mental-switch box ← **start here after a break**

---

# PART I — CORE LANGUAGE

## 1. Numbers, BigInt and the IEEE-754 Reality

JS has **exactly one** numeric type for everyday use: `number`, a 64-bit IEEE-754 double. There is no `int`, `long`, `float`, `double`, `short` distinction — every number, whole or fractional, is the same type.

> **vs C++:** In C++/Java you pick the width and whether it's integral. In JS you don't get to choose — `5` and `5.0` are literally the same value. This is the single biggest source of surprise for a C++ mind doing DSA in JS.

### 1.1 Safe integer range

Doubles have a 52-bit mantissa, so only integers up to 2^53 - 1 can be represented exactly.

```js
Number.MAX_SAFE_INTEGER   // => 9007199254740991  (2^53 - 1)
Number.MIN_SAFE_INTEGER   // => -9007199254740991

// Precision silently breaks beyond this — no error, no warning:
9007199254740992 + 1 === 9007199254740992   // => true  (!)
9007199254740993                              // => 9007199254740992 (rounds down)
```

> **Gotcha:** In competitive programming, sums/products that fit in `long long` (up to ~9.2e18) in C++ can silently corrupt in JS past ~9e15. If a problem's constraints allow values near or beyond 2^53 (large factorials, big modpow bases, sums of up to 1e5 numbers each up to 1e15), you likely need `BigInt`.

Other numeric constants:

```js
Number.MAX_VALUE   // => 1.7976931348623157e+308 (largest representable, NOT integer-safe)
Number.EPSILON     // => 2.220446049250313e-16   (smallest diff between 1 and next double)
Infinity           // => Infinity   (e.g. 1/0)
-Infinity          // => -Infinity  (e.g. -1/0)
NaN                // => NaN        (e.g. 0/0, Number('abc'))
```

### 1.2 Division, floor division, modulo

```js
7 / 2          // => 3.5   (NOT integer division like C++'s int/int!)
Math.trunc(7 / 2)   // => 3   (truncate toward zero — matches C++ int division)
Math.floor(7 / 2)   // => 3   (same for positive numbers)
Math.floor(-7 / 2)  // => -4  (floors toward -Infinity — DIFFERS from Math.trunc!)
Math.trunc(-7 / 2)  // => -3  (matches C++'s -7/2 == -3)
(7 / 2) | 0         // => 3   (bitwise-OR-with-0 truncates, but ONLY safe for 32-bit range)
```

> **Gotcha:** `Math.floor` and `Math.trunc` agree for positive numbers but diverge for negatives. C++ integer division truncates toward zero — so `Math.trunc`, not `Math.floor`, is the direct equivalent of C++'s `a / b` on ints.

```js
-7 % 3     // => -1   (result follows the SIGN OF THE DIVIDEND, same rule as C++)
7 % -3     // => 1
```

> **vs C++:** Good news — `%` semantics match C++ exactly (sign follows dividend, unlike Python's `%` which follows the divisor). If you need a always-non-negative modulo (common in DSA, e.g. hashing/circular indices): `((a % n) + n) % n`.

### 1.3 Checks

```js
Number.isInteger(5)        // => true
Number.isInteger(5.0)      // => true  (same value, no separate int type)
Number.isInteger(5.5)      // => false
Number.isSafeInteger(2**53)  // => false (beyond safe range)

Number.isNaN(NaN)          // => true
Number.isNaN('x')          // => false  (does NOT coerce — the SAFE one)
isNaN('x')                 // => true   (global isNaN COERCES 'x' to NaN first — AVOID)
```

> **Gotcha:** Always prefer `Number.isNaN` over the global `isNaN`. The global version coerces its argument to a number first, so `isNaN('foo')` is `true` even though `'foo'` isn't remotely numeric.

### 1.4 BigInt **[ES2020]**

For exact integers beyond `Number.MAX_SAFE_INTEGER` — large factorials, exact 64-bit arithmetic, modpow with huge bases.

```js
const big = 10n;                      // BigInt literal (trailing n)
const big2 = BigInt(10);              // from a Number
const big3 = BigInt("123456789012345678901234567890");

10n + 20n           // => 30n
10n * 10n ** 18n     // => 10000000000000000000n  (exact, no precision loss)

// CANNOT mix BigInt and Number — TypeError:
10n + 5              // TypeError: Cannot mix BigInt and other types
10n + BigInt(5)      // => 15n   (convert first)
Number(10n) + 5      // => 15    (or convert BigInt -> Number, losing exactness if huge)

10n / 3n             // => 3n   (BigInt division TRUNCATES — no fractional part exists)
10n === 10           // => false (different types, strict equality fails)
10n == 10            // => true  (loose equality coerces)
```

> **Gotcha:** BigInt is noticeably slower than `number` and has no `Math` support (`Math.max(1n, 2n)` fails). Only reach for it when a problem's constraints genuinely require exact integers beyond 2^53 (common in "count the exact value" problems, rare in ones that ask for `answer % (1e9+7)`, since that fits in `number` range for typical modulus sizes if you're careful about multiplication overflow — see §1.1).

```js
// Typical CP pattern needing BigInt: exact big multiplication before a final mod
function mulmod(a, b, m) {
  return Number((BigInt(a) * BigInt(b)) % BigInt(m));
}
```

### 1.5 Parsing and formatting

```js
parseInt("42px")        // => 42   (stops at first non-digit char)
parseInt("  42")        // => 42   (leading whitespace ok)
parseInt("42", 2)       // => NaN  (radix 2, but "4" isn't a binary digit) — parses "1010" style
parseInt("1010", 2)     // => 10   (binary "1010" -> decimal 10)
parseInt("0x1F")        // => 31   (auto-detects hex prefix)
parseInt("abc")         // => NaN

parseFloat("3.14abc")   // => 3.14 (stops at first invalid char, keeps decimal point)

Number("42")            // => 42
Number("42px")          // => NaN   (Number() is STRICT — whole string must be numeric)
Number("  42  ")        // => 42    (trims whitespace, unlike parseInt's partial-parse)
Number("")              // => 0     (a classic trap — empty string is NOT NaN)
Number(null)             // => 0
Number(undefined)        // => NaN

+"42"                    // => 42   (unary plus == Number(), fast idiom)
+""                      // => 0
```

> **Gotcha:** `parseInt`/`parseFloat` do partial parsing (stop at first bad char); `Number()` and unary `+` require the *entire* string to be numeric or you get `NaN`. Pick based on whether you want lenient or strict parsing. For DSA input parsing (splitting a line of ints), `Number` or `+` on already-split tokens is usually right; `parseInt` is for strings with known trailing garbage.

```js
(255).toString(16)   // => "ff"     (decimal -> hex string)
(255).toString(2)    // => "11111111"
parseInt("ff", 16)   // => 255      (hex string -> decimal)

(3.14159).toFixed(2)  // => "3.14"  (returns a STRING, not a number!)
Number((3.14159).toFixed(2))  // => 3.14  (convert back if you need a number)
```

## 2. Types, Coercion and Equality

7 primitive types: `number`, `bigint`, `string`, `boolean`, `undefined`, `symbol`, `null` — plus `object` (which covers arrays, functions, plain objects, Map, Set, etc.).

### 2.1 `typeof`

```js
typeof 42          // => "number"
typeof 10n          // => "bigint"
typeof "s"          // => "string"
typeof true         // => "boolean"
typeof undefined    // => "undefined"
typeof Symbol()     // => "symbol"
typeof {}           // => "object"
typeof []           // => "object"   (arrays are objects! use Array.isArray to distinguish)
typeof null         // => "object"   (LONGSTANDING BUG, null is not actually an object)
typeof NaN          // => "number"   (NaN IS a number value, just an invalid one)
typeof function(){} // => "function" (the one exception — functions get their own typeof result)
```

> **Gotcha:** `typeof null === 'object'` is a 25+ year old language bug kept for backward compatibility. To check for null, compare directly: `x === null`. To check for an array, use `Array.isArray(x)`, never `typeof`.

### 2.2 Equality: `===` vs `==`

```js
// ALWAYS use === (strict) — no type coercion, does what a C++ dev expects
1 === 1        // => true
1 === "1"      // => false  (different types, no coercion)
null === undefined  // => false

// == (loose) coerces types before comparing — a minefield, avoid:
0 == ''            // => true   (both coerce to 0)
0 == '0'           // => true
0 == []            // => true   ([] coerces to '' coerces to 0)
0 == false         // => true
'' == false        // => true
null == undefined  // => true   (special-cased — the ONE loose-equality pattern some codebases allow)
[] == false        // => true
[] == ![]          // => true   (! runs first: ![] is false; then [] == false is true)
NaN == NaN         // => false  (NaN never equals anything, even itself, in either == or ===)
```

> **vs C++:** There is no operator overloading, so `==` isn't customizable like in C++ — it follows a fixed, surprising coercion table. The house rule: **use `===`/`!==` unless you have an explicit, deliberate reason for `==`** (the only common legitimate one is `x == null` to catch both `null` and `undefined` at once).

### 2.3 `null` vs `undefined`

```js
let x;                 // x is undefined — declared but not assigned
let y = null;          // y is null — deliberately "no value"

function f(a) { return a; }
f();                    // => undefined  (missing argument)

const obj = { a: 1 };
obj.b;                  // => undefined  (missing property)
obj.a = null;           // deliberate "this exists but has no value"
```

Rule of thumb: `undefined` = the language's "nothing here" (uninitialized var, missing arg, missing prop, function with no `return`). `null` = your code's deliberate "empty/absent" value (e.g., "no parent node" in a linked list).

### 2.4 NaN and `Object.is`

```js
NaN === NaN            // => false  (the famous exception)
Number.isNaN(NaN)      // => true   (the correct way to check)
Object.is(NaN, NaN)    // => true   (Object.is treats NaN as equal to itself)
Object.is(0, -0)       // => false  (distinguishes +0/-0, unlike ===)
0 === -0                // => true   (=== does NOT distinguish them)
```

> **Gotcha:** `Object.is` is like `===` but fixes the two edge cases `===` gets "wrong" for some purposes: `NaN` vs `NaN`, and `+0` vs `-0`. Rarely needed in DSA, but explains `Array.prototype.includes` (uses SameValueZero, so `[NaN].includes(NaN)` is `true`) vs `indexOf` (uses `===`, so `[NaN].indexOf(NaN)` is `-1`).

### 2.5 Truthy / Falsy

Exactly 8 falsy values — everything else is truthy:

```
false, 0, -0, 0n, '', null, undefined, NaN
```

```js
if ([]) console.log("truthy!");   // => "truthy!"  (empty array IS truthy)
if ({}) console.log("truthy!");   // => "truthy!"  (empty object IS truthy)
if ('') console.log("nope");      // (skipped — empty string is falsy)
if (0) console.log("nope");       // (skipped)
```

> **vs C++:** A C++/Java mind reflexively expects an empty container to be "false-ish" (like Python where `[]`/`{}`  are falsy). In JS, `[]` and `{}` are ALWAYS truthy — only an empty **string** is falsy. To test for an empty array, check `arr.length === 0`, never `if (!arr)`.

### 2.6 Short-circuit, `??`, `?.`

```js
// && / || short-circuit like C++
const a = 0 && "never evaluated";   // => 0 (stops at first falsy)
const b = "x" || "default";         // => "x" (stops at first truthy)
const c = 0 || "default";           // => "default"  (0 is falsy, so || falls through — often WRONG)

// ?? [ES2020] — nullish coalescing: falls through ONLY on null/undefined, not on other falsy values
const d = 0 ?? "default";           // => 0  (0 is not null/undefined, so it's KEPT)
const e = null ?? "default";        // => "default"
const f = undefined ?? "default";   // => "default"
const g = '' ?? "default";          // => ''  (empty string kept, unlike ||)
```

> **Gotcha:** `x || defaultVal` is wrong whenever `0`, `''`, or `false` are legitimate values you want to keep — classic bug: `count = input.count || 10` silently replaces a real `0` count with `10`. Use `??` instead: `count = input.count ?? 10`.

```js
// ?. [ES2020] — optional chaining, short-circuits to undefined instead of throwing
const node = { left: null };
node.left?.val          // => undefined  (instead of TypeError: Cannot read properties of null)
node.left?.val ?? -1     // => -1   (common DSA pattern: safe-access-with-default for tree/graph nodes)
node?.method?.()         // optional call — only calls if node.method exists
arr?.[0]                 // optional index access
```

### 2.7 Explicit conversions

```js
String(42)        // => "42"
String(null)       // => "null"
String([1,2,3])    // => "1,2,3"   (array -> comma-joined string)
Number("42")       // => 42
Number(true)        // => 1
Boolean(0)          // => false
Boolean("0")        // => true    (non-empty string is truthy, even "0"!)
!!"0"                // => true    (double-bang idiom for Boolean())
!!0                  // => false
+"42"                // => 42      (unary plus idiom for Number())
"42" | 0              // => 42      (bitwise OR with 0 -> forces int32, truncates decimals)
"3.9" | 0              // => 3
```

> **Gotcha:** `Boolean("0")` is `true` — the string `"0"` is non-empty so it's truthy, even though the number `0` is falsy. Don't confuse the string `"0"` (from `.toString()` or JSON) with the number `0` when branching.

## 3. Variables, Scope, Hoisting and Closures

### 3.1 `let` / `const` / `var`

| | scope | hoisting | reassignable | redeclarable |
|---|---|---|---|---|
| `var` | function | hoisted, init to `undefined` | yes | yes (no error) |
| `let` | block | hoisted, in TDZ | yes | no (SyntaxError) |
| `const` | block | hoisted, in TDZ | no | no (SyntaxError) |

```js
{
  var v = 1;
}
console.log(v);       // => 1  (var leaks out of the block — function/global scoped)

{
  let l = 1;
}
console.log(l);        // ReferenceError: l is not defined  (block-scoped, gone)
```

```js
console.log(v2);   // => undefined  (var is hoisted, usable before declaration, just undefined)
var v2 = 5;

console.log(l2);   // ReferenceError: Cannot access 'l2' before initialization (TDZ)
let l2 = 5;
```

> **`const` means the BINDING is immutable, not the value.** This trips up C++ devs expecting `const` to deep-freeze:

```js
const arr = [1, 2, 3];
arr.push(4);        // OK! mutating contents is fine
arr[0] = 99;         // OK!
arr = [];             // TypeError: Assignment to constant variable. (rebinding the variable itself is not)

const obj = { a: 1 };
obj.a = 2;             // OK — obj's properties are mutable
obj.b = 3;             // OK
```

**Rule: never use `var` in modern code.** Default to `const`; use `let` only when you'll reassign (loop counters, accumulators).

### 3.2 Hoisting summary

```js
foo();                 // => "hoisted!"  (function declarations are FULLY hoisted, body included)
function foo() { console.log("hoisted!"); }

bar();                  // TypeError: bar is not a function (var hoisted as undefined, not the function)
var bar = function() { console.log("expr"); };
```

### 3.3 Global scope note (Node vs browser)

In Node, top-level `let`/`const`/`var`/`function` declared in a module are scoped to that **module** (CommonJS wraps files in a function; ESM has its own module scope) — they do NOT pollute a shared global object like browser `<script>` tags do. In a single competitive-programming file this rarely matters, but it's why `var x` at top-level in a Node script isn't visible as `global.x`.

### 3.4 Closures — THE core concept

A closure is a function bundled with references to its surrounding (lexical) scope, so it can read/update those variables even after the outer function has returned.

**Counter:**
```js
function makeCounter() {
  let count = 0;
  return () => ++count;   // closes over `count`
}
const counter = makeCounter();
counter();   // => 1
counter();   // => 2
counter();   // => 3       (each call remembers/mutates the SAME `count`)
```

**Memoization cache (very common in DP problems):**
```js
function memoFib() {
  const cache = new Map();
  return function fib(n) {
    if (n <= 1) return n;
    if (cache.has(n)) return cache.get(n);
    const result = fib(n - 1) + fib(n - 2);
    cache.set(n, result);
    return result;
  };
}
const fib = memoFib();
fib(40);   // fast — cache persists across calls via closure
```

**The classic `var`-in-loop closure bug:**
```js
// BUG: all three callbacks share the SAME `i` (var is function-scoped, not per-iteration)
const fns = [];
for (var i = 0; i < 3; i++) {
  fns.push(() => console.log(i));
}
fns.forEach(fn => fn());   // => 3, 3, 3   (all see the FINAL value of i, loop already finished)

// FIX: let creates a NEW binding of i for EACH iteration
const fns2 = [];
for (let j = 0; j < 3; j++) {
  fns2.push(() => console.log(j));
}
fns2.forEach(fn => fn());   // => 0, 1, 2   (each closure captured its own iteration's j)
```

> **Gotcha:** This is the #1 closure trap in interview code (e.g., building callbacks/comparators inside a loop, or setTimeout in a loop). Using `let` instead of `var` as the loop variable fixes it for free because `let` gives each iteration its own binding. If you must use an index captured in a callback, `let` (or an IIFE, see below) is the fix.

### 3.5 IIFE (brief)

Immediately Invoked Function Expression — runs once, creates a private scope. Mostly legacy now that `let`/block scope and modules exist, but you'll still see it:

```js
const result = (function () {
  const secret = 42;
  return secret * 2;
})();
console.log(result);   // => 84
// `secret` is not accessible out here
```

## 4. Functions, Arrow Functions and `this`

### 4.1 Three ways to define a function

```js
// Declaration — hoisted, can be called before its definition in the file
function add(a, b) { return a + b; }

// Expression — NOT hoisted (the variable is, the assignment isn't)
const sub = function (a, b) { return a - b; };

// Arrow function — concise, lexical `this`, cannot be a constructor
const mul = (a, b) => a * b;
```

### 4.2 Arrow function specifics

```js
const square = x => x * x;              // single param: parens optional
const add2 = (a, b) => a + b;             // concise body: implicit return
const make = () => ({ a: 1 });            // returning an object literal needs PARENS around it
                                            // (bare {} after => is parsed as a function BODY, not an object)
const block = (a, b) => {                 // block body needs explicit `return`
  const sum = a + b;
  return sum;
};
```

Key differences from regular functions:

| | regular `function` | arrow `=>` |
|---|---|---|
| own `this` | yes (dynamic, call-site dependent) | no — inherits `this` from enclosing scope |
| own `arguments` | yes | no — must use rest params `...args` |
| usable as constructor (`new`) | yes | no — `new (()=>{})()` throws TypeError |
| hoisted (declaration form) | yes | no |

```js
function regular() { console.log(arguments); }
regular(1, 2, 3);   // => [Arguments] { '0': 1, '1': 2, '2': 3 }

const arrow = () => console.log(typeof arguments);  // ReferenceError if no outer `arguments` exists —
                                                        // arrows have NO arguments object of their own
```

### 4.3 Default and rest parameters

```js
function greet(name = "World") { return `Hello, ${name}`; }
greet();          // => "Hello, World"
greet("JS");       // => "Hello, JS"

function sumAll(...nums) {       // rest params — collects remaining args into a real Array
  return nums.reduce((a, b) => a + b, 0);
}
sumAll(1, 2, 3, 4);   // => 10

function log(first, ...rest) {   // rest must be LAST parameter
  console.log(first, rest);
}
log(1, 2, 3);   // => 1 [2, 3]
```

### 4.4 Spread in calls

```js
const nums = [3, 1, 4, 1, 5];
Math.max(...nums);        // => 5   (spread array into individual arguments)
Math.min(...nums, 0);     // => 0   (mix spread with extra args)

function point(x, y, z) { return [x, y, z]; }
point(...[1, 2, 3]);      // => [1, 2, 3]
```

> **Gotcha:** `Math.max(...hugeArray)` can blow the call stack for very large arrays (tens of thousands of elements) since spread turns into that many function arguments. For big arrays, use `arr.reduce((a,b) => Math.max(a,b))` instead.

### 4.5 `this` binding — the 4 rules

`this` in a regular function is determined by **how it's called**, not where it's defined:

```js
const obj = {
  val: 42,
  regular() { return this.val; },
  arrow: () => this?.val,   // `this` here is the ENCLOSING (module/undefined) scope, NOT obj
};
obj.regular();   // => 42   (implicit binding: called as obj.regular(), this = obj)
obj.arrow();      // => undefined  (arrow ignores obj entirely — lexical this)

const detached = obj.regular;
detached();       // TypeError or undefined (default binding: this is undefined in strict/module mode)

const bound = obj.regular.bind(obj);
bound();           // => 42   (explicit binding via bind)

new function() { this.x = 1; };   // `new` binding: this = newly created object (arrows CANNOT do this)
```

The 4 rules, in precedence order: **`new` binding** > **explicit** (`call`/`apply`/`bind`) > **implicit** (`obj.method()`) > **default** (plain call, `this` is `undefined` in strict mode / modules).

> **vs C++:** C++/Java's `this` is always "the current object instance," fixed at the call. JS's `this` is dynamic and re-determined at every call site — the SAME function can have a different `this` depending on how you invoke it. This is why passing `obj.method` as a bare callback (e.g., to `setTimeout`, `.map`, event handlers) silently loses `this`.

```js
class Node {
  constructor(val) { this.val = val; }
  regularGet() { return this.val; }
  arrowGet = () => this.val;   // class field arrow — binds `this` to the instance FOREVER, safe as a callback
}
const n = new Node(5);
const g1 = n.regularGet;
g1();                // TypeError: Cannot read properties of undefined (this lost when detached)
const g2 = n.arrowGet;
g2();                 // => 5   (arrow field keeps `this` bound to n regardless of how it's called)
```

**DSA-relevant takeaway:** if you write a class (e.g. a custom Heap/DSU) and pass a method as a callback (e.g. `arr.sort(myHeap.compare)`), either use an arrow class field, or `.bind(this)`, or (simplest) just use a plain arrow function/closure instead of a class method.

### 4.6 `call` / `apply` / `bind`

```js
function greet(greeting) { return `${greeting}, ${this.name}`; }
const person = { name: "Alice" };

greet.call(person, "Hi");        // => "Hi, Alice"   (args passed individually)
greet.apply(person, ["Hi"]);      // => "Hi, Alice"   (args passed as an array)
const bound = greet.bind(person);
bound("Hi");                       // => "Hi, Alice"   (bind returns a NEW function, permanently bound)
```

### 4.7 Recursion and named function expressions

```js
// Anonymous function expression assigned to const — const name isn't visible INSIDE the function
const fact = function (n) {
  return n <= 1 ? 1 : n * fact(n - 1);   // works because `fact` is captured via closure over the outer const
};

// Named function expression — safer if the const binding might be reassigned/passed around
const fact2 = function factInner(n) {
  return n <= 1 ? 1 : n * factInner(n - 1);   // factInner always refers to itself, independent of outer name
};
```

### 4.8 Higher-order functions

Functions are first-class values — pass them as arguments, return them, store them in arrays/objects. This underlies comparators, callbacks, and functional array methods (see Part on arrays):

```js
const compareAsc = (a, b) => a - b;
[3, 1, 2].sort(compareAsc);   // => [1, 2, 3]

function withLogging(fn) {
  return (...args) => {
    console.log("calling with", args);
    return fn(...args);
  };
}
const loggedAdd = withLogging((a, b) => a + b);
loggedAdd(2, 3);   // logs "calling with [2, 3]", returns 5
```

### 4.9 No function overloading

```js
function f(x) { return x; }
function f(x, y) { return x + y; }   // this SILENTLY REPLACES the first f — no error, no overload resolution
f(1);        // => NaN  (called with (x=1, y=undefined), 1 + undefined = NaN)
```

> **vs C++:** JS has no function overloading by signature — only one `f` can exist per scope, and later definitions win silently. Emulate overloading with default parameters, rest parameters, and runtime type checks (`typeof`/`Array.isArray`) inside a single function body.

## 5. Objects, Prototypes and Classes

### 5.1 Object literals

```js
const p = { x: 1, y: 2 };                 // basic literal

const x = 1, y = 2;
const p2 = { x, y };                       // shorthand property (key name == variable name)

const key = "dynamic";
const p3 = { [key]: 123 };                  // computed key -> { dynamic: 123 }

const p4 = {
  x: 1,
  greet() { return "hi"; },                // method shorthand (no `function` keyword needed)
};
```

### 5.2 Accessing properties

```js
p.x            // => 1        (dot notation — needs a valid identifier key)
p["x"]          // => 1        (bracket notation — required for dynamic/non-identifier keys)
const k = "x";
p[k]             // => 1        (dynamic key access — dot notation CANNOT do this)
p["not-valid-id"]  // needed when the key has hyphens/spaces/starts with a digit
```

### 5.3 Existence checks

```js
const obj = { a: undefined, b: 1 };

"a" in obj                  // => true   (key EXISTS, even though its value is undefined)
obj.a !== undefined          // => false  (TRAP: looks like "a" is missing, but it's actually present!)
obj.hasOwnProperty("a")      // => true   (own property check, ignores prototype chain)
Object.hasOwn(obj, "a")      // => true   (ES2022, preferred modern spelling of hasOwnProperty)

"c" in obj                   // => false  (truly absent)
obj.c                         // => undefined  (accessing a missing key never throws, just undefined)
```

> **Gotcha:** `obj.x !== undefined` is a common but flawed existence check — it gives a false negative when a key exists but was deliberately set to `undefined`. Use `"x" in obj` or `Object.hasOwn(obj, "x")` for a real existence check.

### 5.4 Object utilities

```js
const o = { a: 1, b: 2, c: 3 };

Object.keys(o)      // => ["a", "b", "c"]
Object.values(o)     // => [1, 2, 3]
Object.entries(o)    // => [["a",1], ["b",2], ["c",3]]

for (const [k, v] of Object.entries(o)) { /* k="a", v=1 ... */ }

Object.assign({}, o, { d: 4 });   // => { a:1, b:2, c:3, d:4 }  (shallow merge into a new object)
const copy = { ...o, d: 4 };       // => same result, spread syntax (more idiomatic today)

Object.freeze(o);      // shallow-freezes — top-level props can't be reassigned (silently fails in
                          // non-strict mode, throws in strict mode / modules)
o.a = 99;                 // no-op (or TypeError in strict mode), o.a is still 1
```

> **Gotcha:** `{...obj}` and `Object.assign` are SHALLOW copies. Nested objects/arrays are still shared by reference:
```js
const nested = { arr: [1, 2] };
const shallow = { ...nested };
shallow.arr.push(3);
nested.arr   // => [1, 2, 3]   (the inner array was NOT copied, both point to the same array!)
```

### 5.5 Deleting properties

```js
delete obj.a;    // removes the key entirely (obj no longer has "a" at all, unlike setting undefined)
```

> **Gotcha:** `delete` is slow — it can deoptimize the object's internal shape/hidden class in V8, hurting performance in hot loops. For DSA code needing frequent insert/delete-by-key (e.g. a frequency map, LRU cache), prefer `Map` (`map.delete(key)` is fast and purpose-built) over plain objects.

### 5.6 Iterating objects

```js
const obj2 = { a: 1, b: 2 };

for (const key in obj2) { console.log(key); }   // => "a", "b"  — but ALSO walks inherited enumerable
                                                    // props up the prototype chain (rarely what you want)

// Prefer:
for (const key of Object.keys(obj2)) { /* own keys only, no prototype pollution */ }
for (const [k, v] of Object.entries(obj2)) { /* own keys+values, no prototype pollution */ }
```

> **Gotcha:** `for-in` on a plain object can pick up properties added to `Object.prototype` by other code (or, more commonly, just surprises you by including inherited/non-own keys). `Object.keys/values/entries` only give **own, enumerable** properties — almost always the correct choice.

### 5.7 Prototypes and the prototype chain (brief)

Every object has an internal link (`[[Prototype]]`, accessible via `Object.getPrototypeOf` or the legacy `__proto__`) to another object it "inherits" from. Property lookups walk up this chain until found or the chain ends at `null`.

```js
const animal = { speak() { return "..."; } };
const dog = Object.create(animal);   // dog's prototype is `animal`
dog.speak();     // => "..."   (found via the prototype chain, dog itself has no `speak`)

dog.hasOwnProperty("speak");   // => false  (it's inherited, not own)
Object.getPrototypeOf(dog) === animal;   // => true
```

This is the mechanism `class`/`extends` are built on (see below) — you rarely manipulate the prototype chain directly in DSA code, but understanding it explains why `for-in` walks up, why `hasOwnProperty` matters, and how `class` inheritance actually works under the hood.

### 5.8 ES6 Classes

`class` is syntax sugar over the prototype system — cleaner than manual `Object.create`/constructor-function patterns, and the natural way to write DSA helper types (linked-list Node, DSU, Heap, Trie node, etc.).

```js
class ListNode {
  constructor(val = 0, next = null) {
    this.val = val;
    this.next = next;
  }
}
const head = new ListNode(1, new ListNode(2, new ListNode(3)));
```

```js
class DSU {
  #parent;   // private field [ES2022] — only accessible inside this class, true encapsulation
  #rank;

  constructor(n) {
    this.#parent = Array.from({ length: n }, (_, i) => i);
    this.#rank = new Array(n).fill(0);
  }

  find(x) {
    if (this.#parent[x] !== x) this.#parent[x] = this.find(this.#parent[x]);  // path compression
    return this.#parent[x];
  }

  union(a, b) {
    const ra = this.find(a), rb = this.find(b);
    if (ra === rb) return false;
    if (this.#rank[ra] < this.#rank[rb]) [ra, rb] = [rb, ra]; // wait: needs distinct vars, see note below
    this.#parent[rb] = ra;
    if (this.#rank[ra] === this.#rank[rb]) this.#rank[ra]++;
    return true;
  }

  static create(n) { return new DSU(n); }   // static factory method — called as DSU.create(5), not on an instance
}
```

```js
class MinHeap {
  constructor() { this.data = []; }

  get size() { return this.data.length; }       // getter — accessed like a property: heap.size (no parens)

  push(val) {
    this.data.push(val);
    this.#bubbleUp(this.data.length - 1);
  }

  pop() {
    const top = this.data[0];
    const last = this.data.pop();
    if (this.data.length) { this.data[0] = last; this.#bubbleDown(0); }
    return top;
  }

  #bubbleUp(i) {   // private method [ES2022]
    while (i > 0) {
      const parent = (i - 1) >> 1;
      if (this.data[parent] <= this.data[i]) break;
      [this.data[parent], this.data[i]] = [this.data[i], this.data[parent]];
      i = parent;
    }
  }
  #bubbleDown(i) {
    const n = this.data.length;
    while (true) {
      let smallest = i, l = 2*i+1, r = 2*i+2;
      if (l < n && this.data[l] < this.data[smallest]) smallest = l;
      if (r < n && this.data[r] < this.data[smallest]) smallest = r;
      if (smallest === i) break;
      [this.data[i], this.data[smallest]] = [this.data[smallest], this.data[i]];
      i = smallest;
    }
  }
}
```

`extends` / `super`:

```js
class Shape {
  constructor(name) { this.name = name; }
  area() { return 0; }
}
class Circle extends Shape {
  constructor(r) {
    super("circle");    // MUST call super() before using `this` in a derived class constructor
    this.r = r;
  }
  area() { return Math.PI * this.r ** 2; }   // overrides base method
}
new Circle(2).area();   // => 12.566...
```

```js
class Base {
  static count = 0;                  // static field — belongs to the class, not instances
  static increment() { Base.count++; }  // static method — called as Base.increment(), not on an instance
}
Base.increment();
Base.count;   // => 1
```

> **Gotcha — no operator overloading, and objects compare by reference:**
```js
{ a: 1 } === { a: 1 }        // => false  (different object identities, even with identical contents)
[1, 2] === [1, 2]              // => false  (same reason)
const p1 = { a: 1 };
p1 === p1                       // => true   (same reference)

class Point { constructor(x,y){ this.x=x; this.y=y; } }
new Point(1,1) + new Point(2,2)   // => "[object Object][object Object]" (NO operator overloading —
                                     // + falls back to string coercion, does NOT add coordinates)
```
> There's no `operator==`/`operator+` like C++. To compare objects by value, write your own function (or compare a canonical key, e.g. `JSON.stringify`, or `x1===x2 && y1===y2` for simple cases). To "add" custom objects, write a method (`p1.add(p2)`).

## 6. Destructuring, Spread, Rest and Modern Syntax

### 6.1 Array destructuring

```js
const arr = [1, 2, 3];
const [a, b] = arr;              // a=1, b=2  (extra elements ignored)

let x = 1, y = 2;
[x, y] = [y, x];                  // SWAP with no temp variable — very handy in DSA (two-pointer swaps)
console.log(x, y);                 // => 2 1

const [, second] = arr;            // skip first element with a blank slot -> second = 2
const [first, ...rest] = arr;      // first = 1, rest = [2, 3]
const [p = 10, q = 20] = [5];      // defaults when the source slot is undefined -> p=5, q=20
```

### 6.2 Object destructuring

```js
const obj = { x: 1, y: 2, z: 3 };
const { x, y } = obj;                    // x=1, y=2  (matched by KEY NAME, order doesn't matter)
const { x: px, y: py } = obj;             // renaming -> px=1, py=2
const { w = 99 } = obj;                    // default when key missing -> w=99
const { nested: { deep } } = { nested: { deep: 42 } };   // nested destructuring -> deep=42
```

### 6.3 Destructuring in function parameters

```js
function dist({ x, y }) { return Math.sqrt(x*x + y*y); }
dist({ x: 3, y: 4 });   // => 5

function first2([a, b]) { return a + b; }
first2([10, 20]);        // => 30

function options({ verbose = false, limit = 10 } = {}) { return { verbose, limit }; }
options();                 // => { verbose: false, limit: 10 }   (the `= {}` default handles calling with NO arg)
```

### 6.4 Spread

```js
const a1 = [1, 2, 3];
const copy = [...a1];              // shallow copy of an array
const merged = [...a1, ...[4,5]];   // => [1,2,3,4,5]  concat via spread

Math.max(...a1);                    // => 3  (spread into a call — see §4.4 for the stack-size caveat)

const o1 = { a: 1 }, o2 = { b: 2 };
const mergedObj = { ...o1, ...o2 };    // => { a:1, b:2 }  shallow merge/clone (later spreads win on key conflict)
const withOverride = { ...o1, a: 99 };  // => { a:99 }     explicit keys after spread override it
```

### 6.5 Rest parameters (recap, see §4.3)

```js
function sum(...nums) { return nums.reduce((a,b)=>a+b, 0); }
```

### 6.6 Template literals

```js
const name = "World";
`Hello, ${name}!`;                 // => "Hello, World!"
`1 + 1 = ${1 + 1}`;                  // => "1 + 1 = 2"   (any expression inside ${})

const multiline = `line1
line2`;                              // real newline preserved, no need for \n concatenation
// multiline === "line1\nline2"      // => true
```

### 6.7 Short-circuit assignment **[ES2021]**

```js
let a = null;
a ||= 5;     // a = a || 5           -> a becomes 5 (a was falsy)
let b = 0;
b ||= 5;      // -> b becomes 5 (0 is falsy — careful, see §2.6 caveat)
let c = 0;
c ??= 5;      // c = c ?? 5           -> c STAYS 0 (0 is not null/undefined)
let d = { x: 1 };
d.y &&= 10;    // d.y = d.y && 10     -> d.y stays undefined (undefined is falsy, && short-circuits, no assignment)
d.x &&= 10;    // -> d.x becomes 10 (d.x was truthy, so the assignment happens)
```

### 6.8 Ternary

```js
const max = a > b ? a : b;
const label = n % 2 === 0 ? "even" : "odd";
```

### 6.9 Comma operator

```js
for (let i = 0, j = 10; i < j; i++, j--) { /* two updates per iteration */ }
let x2 = (1, 2, 3);   // => 3  (evaluates all, yields the LAST — rare outside for-loops)
```

### 6.10 Labeled statements — breaking out of nested loops

```js
// Genuinely useful in grid/matrix DSA problems: break/continue an OUTER loop from an inner one
outer:
for (let i = 0; i < grid.length; i++) {
  for (let j = 0; j < grid[i].length; j++) {
    if (grid[i][j] === target) {
      console.log("found at", i, j);
      break outer;      // breaks BOTH loops at once — plain `break` would only exit the inner loop
    }
  }
}

search:
for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) continue search;   // continue the OUTER loop, skipping rest of inner iteration
    console.log(i, j);
  }
}
```

> **vs C++:** C++ has no labeled break/continue — you'd normally use a flag variable or `goto`. JS's labeled statements are the clean built-in equivalent and are idiomatic here (unlike `goto` in C++, which is usually discouraged).

## 7. Iteration Protocols, Generators, for-of vs for-in

### 7.1 for-of vs for-in — THE trap

```js
const arr = ["a", "b", "c"];

for (const val of arr) console.log(val);    // => "a", "b", "c"   (VALUES — right loop for arrays)

for (const idx in arr) console.log(idx);     // => "0", "1", "2"  (KEYS as STRINGS — usually NOT what you want)
```

> **Gotcha:** `for-in` iterates enumerable KEYS/indices (as strings!) — including any inherited enumerable properties up the prototype chain. On an array, that means string indices, not values, plus the risk of picking up unexpected inherited keys. **Never use `for-in` on an array.** Use `for-of` for values, classic indexed `for` when you need the index, or `arr.entries()` for both.

```js
for (const [i, val] of arr.entries()) {
  console.log(i, val);   // => 0 "a", 1 "b", 2 "c"   (index + value together)
}
```

`for-of` works on any **iterable**: arrays, strings, `Map`, `Set`, `arguments`, generators — NOT plain objects (they aren't iterable by default):

```js
for (const ch of "abc") console.log(ch);              // => "a", "b", "c"
const map = new Map([["x",1], ["y",2]]);
for (const [k, v] of map) console.log(k, v);            // => "x" 1, "y" 2  (Map entries destructure naturally)
const set = new Set([1, 2, 3]);
for (const v of set) console.log(v);                      // => 1, 2, 3

for (const x of { a: 1 }) console.log(x);                 // TypeError: {a:1} is not iterable
```

### 7.2 Classic indexed `for` — still fastest for hot loops

```js
for (let i = 0; i < arr.length; i++) { /* ... */ }
```
In hot inner loops (tight numeric work, tens of millions of iterations) the classic `for` loop is typically the fastest option in V8 — faster than `for-of` (which goes through the iterator protocol) and much faster than `forEach` (function-call overhead per element). Reach for it when performance is on the critical path; use `for-of`/array methods otherwise for readability.

### 7.3 `forEach` — can't `break`

```js
[1, 2, 3, 4].forEach(x => {
  if (x === 2) return;      // this only skips the CURRENT callback invocation (acts like `continue`)
  console.log(x);            // => 1, 3, 4   (2 is skipped, but the loop still visits 3 and 4)
});

// There is NO way to break out of forEach early — `break`/`return` don't stop it:
[1, 2, 3, 4].forEach(x => {
  if (x === 2) break;         // SyntaxError: Illegal break statement (break isn't even valid here)
});
```

> **Gotcha:** `forEach` has no early-exit mechanism. If you need to stop iterating partway (very common in search-style DSA code), use a plain `for`/`for-of` loop with `break`, or `.some()` (returns `true` to stop, mimics break) / `.every()` (returns `false` to stop).

```js
// .some() as a break-able "forEach":
[1, 2, 3, 4].some(x => {
  if (x === 3) return true;    // truthy return STOPS iteration
  console.log(x);               // => 1, 2
  return false;
});
```

### 7.4 The iterable/iterator protocol

An object is **iterable** if it has a `[Symbol.iterator]` method returning an **iterator** — an object with `.next()` returning `{ value, done }`.

```js
const range = {
  from: 1, to: 3,
  [Symbol.iterator]() {
    let current = this.from, last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      },
    };
  },
};
[...range];              // => [1, 2, 3]   (spread works because range is iterable)
for (const n of range) console.log(n);   // => 1, 2, 3  (for-of works too)
```

Manual iterator use:
```js
const it = [10, 20][Symbol.iterator]();
it.next();   // => { value: 10, done: false }
it.next();   // => { value: 20, done: false }
it.next();   // => { value: undefined, done: true }
```

### 7.5 `entries()` / `keys()` / `values()`

```js
const arr2 = ["a", "b"];
[...arr2.keys()];      // => [0, 1]
[...arr2.values()];     // => ["a", "b"]
[...arr2.entries()];     // => [[0,"a"], [1,"b"]]

const map2 = new Map([["x",1]]);
[...map2.keys()];        // => ["x"]
[...map2.values()];       // => [1]
[...map2.entries()];       // => [["x",1]]
```

### 7.6 Generators `function*` and `yield`

A generator function returns an iterator lazily — each `yield` pauses execution and hands back a value; the next `.next()` call resumes right after that `yield`.

```js
function* countUp(n) {
  for (let i = 1; i <= n; i++) yield i;
}
for (const x of countUp(3)) console.log(x);   // => 1, 2, 3
[...countUp(3)];                                // => [1, 2, 3]

const gen = countUp(2);
gen.next();   // => { value: 1, done: false }
gen.next();   // => { value: 2, done: false }
gen.next();   // => { value: undefined, done: true }
```

**DSA use case — lazy permutation generation** (avoid materializing all n! permutations up front):

```js
function* permutations(arr) {
  if (arr.length <= 1) { yield arr; return; }
  for (let i = 0; i < arr.length; i++) {
    const rest = [...arr.slice(0, i), ...arr.slice(i + 1)];
    for (const perm of permutations(rest)) {
      yield [arr[i], ...perm];    // yield* below shows the equivalent shorthand
    }
  }
}
for (const p of permutations([1,2,3])) {
  console.log(p);   // yields [1,2,3], [1,3,2], [2,1,3], ... one at a time, lazily
  // could `break` here after finding what you need, without generating the rest — a real advantage
  // over building the full array first
}
```

`yield*` delegates to another iterable/generator (used above implicitly — explicit form):

```js
function* inner() { yield 1; yield 2; }
function* outer() {
  yield 0;
  yield* inner();   // delegates — equivalent to `for (const v of inner()) yield v;`
  yield 3;
}
[...outer()];   // => [0, 1, 2, 3]
```

Generators are also handy for lazy infinite sequences (e.g., an infinite Fibonacci generator you `.next()` on demand) or for writing a custom `[Symbol.iterator]` concisely:

```js
class Range {
  constructor(start, end) { this.start = start; this.end = end; }
  *[Symbol.iterator]() {          // generator method as the iterator implementation — much shorter than §7.4
    for (let i = this.start; i <= this.end; i++) yield i;
  }
}
[...new Range(1, 5)];   // => [1, 2, 3, 4, 5]
```

### 7.7 Array-like vs iterable, and converting

An **array-like** has numeric indices + `.length` but is NOT iterable (e.g., the `arguments` object in old code, or a DOM NodeList in some contexts). An **iterable** implements `Symbol.iterator` (arrays, strings, Map, Set, generators).

```js
function oldStyle() {
  // arguments is array-like but has no .map/.filter/.slice directly
  return Array.from(arguments).map(x => x * 2);   // Array.from converts array-like (or iterable) -> real Array
}
oldStyle(1, 2, 3);   // => [2, 4, 6]

Array.from({ length: 3 }, (_, i) => i * i);   // => [0, 1, 4]  (Array.from with a map function — common
                                                  // idiom for building/initializing an array of size n)
Array.from("abc");                              // => ["a", "b", "c"]  (string -> array of chars)
[..."abc"];                                       // => ["a", "b", "c"]  (spread also converts iterables)

Array.from(new Set([1,1,2,3]));                  // => [1, 2, 3]  (dedupe via Set, then back to array)
```

> **Gotcha:** Spread (`...`) only works on iterables. `Array.from` works on BOTH iterables and array-likes (things with just `.length` + indices, no `Symbol.iterator`), making it the more general tool — reach for `Array.from` when spread throws `TypeError: obj is not iterable`.

### 7.8 Destructuring in for-of over Map entries

```js
const scores = new Map([["alice", 90], ["bob", 85]]);
for (const [name, score] of scores) {
  console.log(`${name}: ${score}`);   // => "alice: 90", "bob: 85"
}
// equivalent, more explicit:
for (const [name, score] of scores.entries()) { /* same thing — .entries() is the default iterator anyway */ }
```

---

# PART II — ARRAYS, STRINGS AND TYPED ARRAYS

## 8. Array Fundamentals

JS arrays are **dynamic, heterogeneous, resizable objects with integer-like string keys** — not a contiguous typed block like C++ `std::vector<T>`/`T[]` or Java `T[]`. Any element can be any type, length changes automatically, and under the hood V8 optimizes contiguous arrays of one type into a fast "packed" internal representation — but the language gives you no such guarantee or control.

> **vs C++/Java:** No compile-time element type, no fixed capacity, no `reserve()`. Mixing types (`[1, "a", null, [2,3]]`) is legal and won't throw.

### Creating arrays

```js
const a = [];                 // empty array
const b = [1, 2, 3];          // array literal
const c = new Array(3);       // length 3, but EMPTY SLOTS (holes), NOT [undefined,undefined,undefined]... looks similar but behaves differently!
console.log(c);               // => [ <3 empty items> ]
console.log(c.length);        // => 3
console.log(c[0]);            // => undefined (reads as undefined, but the slot is a "hole")
```

> **Gotcha:** `new Array(3)` creates **holes**, not zeros. Holes are skipped by `.map`, `.forEach`, `.filter`, `for...of` still visits them as `undefined` via index access but iteration-skipping methods silently skip holes:
```js
const holes = new Array(3);
console.log(holes.map(x => 1));      // => [ <3 empty items> ]  -- map SKIPPED every slot!
const dense = [undefined, undefined, undefined];
console.log(dense.map(x => 1));      // => [ 1, 1, 1 ]          -- real values, map runs
```

### Zero-initializing (the correct way)

```js
const zeros = new Array(5).fill(0);
console.log(zeros); // => [ 0, 0, 0, 0, 0 ]

const zeros2 = Array(5).fill(0); // `new` is optional for Array
console.log(zeros2); // => [ 0, 0, 0, 0, 0 ]
```

### Init with index (the "iota" idiom)

```js
const idx = Array.from({ length: 5 }, (_, i) => i);
console.log(idx); // => [ 0, 1, 2, 3, 4 ]

const squares = Array.from({ length: 5 }, (_, i) => i * i);
console.log(squares); // => [ 0, 1, 4, 9, 16 ]
```

### Array of distinct subarrays — the `.fill([])` trap

```js
// CORRECT: each row is a DISTINCT array
const rowsOk = Array.from({ length: 3 }, () => []);
rowsOk[0].push(1);
console.log(rowsOk); // => [ [ 1 ], [], [] ]

// BUG: .fill([]) puts the SAME array reference in every slot
const rowsBug = Array(3).fill([]);
rowsBug[0].push(1);
console.log(rowsBug); // => [ [ 1 ], [ 1 ], [ 1 ] ]  -- all three "changed"!
```

> **Gotcha:** `.fill(x)` with an object/array argument stores **one reference** repeated N times. Only primitives (numbers, strings, booleans) are safe to `.fill()`. For arrays/objects, always use `Array.from({length:n}, () => ...)` or a loop.

### 2D arrays — the classic shared-row bug

```js
// CORRECT: n distinct row arrays
const gridOk = Array.from({ length: 3 }, () => Array(4).fill(0));
gridOk[0][0] = 9;
console.log(gridOk);
// => [ [ 9, 0, 0, 0 ], [ 0, 0, 0, 0 ], [ 0, 0, 0, 0 ] ]

// BUG: Array(n).fill(row) — every row is the SAME reference
const gridBug = Array(3).fill(Array(4).fill(0));
gridBug[0][0] = 9;
console.log(gridBug);
// => [ [ 9, 0, 0, 0 ], [ 9, 0, 0, 0 ], [ 9, 0, 0, 0 ] ]  -- BUG! all rows mutated
```

> **Gotcha:** This is probably the single most common JS DSA bug for people coming from C++/Java, where `vector<vector<int>> grid(n, vector<int>(m, 0));` deep-copies each row. In JS, `Array(n).fill(x)` never deep-copies — always build 2D arrays with `Array.from({length:n}, () => Array(m).fill(0))`.

### `.length` is a settable property

```js
const arr = [1, 2, 3, 4, 5];
arr.length = 3;
console.log(arr); // => [ 1, 2, 3 ]           -- truncates

arr.length = 5;
console.log(arr); // => [ 1, 2, 3, <2 empty items> ]  -- extends with HOLES, not undefined-filled

arr.length = 0;
console.log(arr); // => []                    -- common idiom to clear an array in place
```

> **vs C++/Java:** There's no `.clear()`/no `vector::clear()` equivalent method name — `arr.length = 0` (or `arr.splice(0)`) is the idiom. `arr = []` does NOT clear the original array — it rebinds the variable to a new array, which matters if other references point at the old one.

### Sparse vs dense — perf note

A "dense" array (every index 0..length-1 populated, same-ish type) is stored by V8 as a fast packed array (near C-array speed). A "sparse" array (holes, or huge gaps, e.g. `arr[10000] = 1` on an empty array) degrades to a slower dictionary-mode object. For CP: always build dense arrays via `fill`/`Array.from`, never `arr[i] = x` on an under-allocated array.

### Indexing — no negative indices

```js
const arr = [10, 20, 30];
console.log(arr[-1]);        // => undefined   -- NOT the last element (unlike Python)
console.log(arr[arr.length - 1]); // => 30
console.log(arr.at(-1));     // => 30           -- [ES2022] .at() DOES support negative indices
console.log(arr.at(-2));     // => 20
```

> **vs Python:** JS `arr[-1]` silently returns `undefined` (no error, no wraparound) because `-1` is treated as a nonexistent object key, not a special index. Use `.at(-1)` [ES2022, Node 16.6+] or `arr[arr.length-1]`.

### Type checking, equality, copying

```js
console.log(Array.isArray([1,2,3])); // => true
console.log(Array.isArray("abc"));   // => false
console.log(typeof [1,2,3]);         // => 'object'  -- typeof is USELESS for arrays, use Array.isArray

// Arrays compare by REFERENCE, never by value
console.log([1,2,3] === [1,2,3]); // => false
const x = [1,2,3];
console.log(x === x);             // => true

// No built-in deep-equal. Options:
console.log(JSON.stringify([1,2,3]) === JSON.stringify([1,2,3])); // => true (caveats: key order for objects, no support for undefined/functions/NaN-as-NaN/Infinity)

function arrEq(a, b) {
  return a.length === b.length && a.every((v, i) => v === b[i]);
}
console.log(arrEq([1,2,3], [1,2,3])); // => true

// Shallow copies — nested arrays/objects are still SHARED
const src = [[1,2], [3,4]];
const c1 = src.slice();
const c2 = [...src];
const c3 = Array.from(src);
c1[0].push(99);
console.log(src[0]); // => [ 1, 2, 99 ]  -- shallow copy shares inner arrays!
```

> **Gotcha:** `slice()`, `[...arr]`, and `Array.from(arr)` all copy only the **top level**. For a true deep copy of nested arrays use `structuredClone(arr)` (Node 17+) or `JSON.parse(JSON.stringify(arr))` (loses functions/undefined/`NaN`... actually `NaN`→`null`, `Infinity`→`null`).

---

## 9. The Complete Array Method Reference

### Full method table

| Method | Signature | What it does | Mutates? | Complexity |
|---|---|---|---|---|
| `push` | `arr.push(...items)` | append to end, returns new length | **mutates** | O(1) amortized per item |
| `pop` | `arr.pop()` | remove & return last element | **mutates** | O(1) |
| `shift` | `arr.shift()` | remove & return first element | **mutates** | **O(n)** |
| `unshift` | `arr.unshift(...items)` | insert at front, returns new length | **mutates** | **O(n)** |
| `slice` | `arr.slice(start, end)` | shallow copy of a range (end excl.) | no (new array) | O(k) |
| `splice` | `arr.splice(start, deleteCount, ...items)` | remove/insert/replace in place, returns removed | **mutates** | O(n) |
| `concat` | `arr.concat(other, ...)` | merge arrays | no (new array) | O(n+m) |
| `join` | `arr.join(sep)` | array → string | no | O(n) |
| `indexOf` | `arr.indexOf(x, from)` | first index of x (`===`), -1 if absent | no | O(n) |
| `lastIndexOf` | `arr.lastIndexOf(x, from)` | last index of x | no | O(n) |
| `includes` | `arr.includes(x, from)` | true/false membership (`SameValueZero`) | no | O(n) |
| `find` | `arr.find(pred)` | first element matching pred, else `undefined` | no | O(n) |
| `findIndex` | `arr.findIndex(pred)` | index of first match, else -1 | no | O(n) |
| `findLast` | `arr.findLast(pred)` **[ES2023]** | last element matching pred | no | O(n) |
| `findLastIndex` | `arr.findLastIndex(pred)` **[ES2023]** | index of last match | no | O(n) |
| `some` | `arr.some(pred)` | true if any element matches | no | O(n) (short-circuits) |
| `every` | `arr.every(pred)` | true if all elements match | no | O(n) (short-circuits) |
| `filter` | `arr.filter(pred)` | new array of matching elements | no | O(n) |
| `map` | `arr.map(fn)` | new array of transformed elements | no | O(n) |
| `reduce` | `arr.reduce(fn, init)` | fold left → right | no | O(n) |
| `reduceRight` | `arr.reduceRight(fn, init)` | fold right → left | no | O(n) |
| `forEach` | `arr.forEach(fn)` | run fn per element, no way to `break` | no | O(n) |
| `flat` | `arr.flat(depth=1)` | flatten nested arrays | no | O(n) |
| `flatMap` | `arr.flatMap(fn)` | map then flatten 1 level | no | O(n) |
| `fill` | `arr.fill(v, start, end)` | overwrite range with v | **mutates** | O(n) |
| `copyWithin` | `arr.copyWithin(target, start, end)` | copy a range within itself | **mutates** | O(n) |
| `reverse` | `arr.reverse()` | reverse in place | **mutates** | O(n) |
| `sort` | `arr.sort(cmp)` | sort in place (see §Sort trap) | **mutates** | O(n log n) |
| `keys` | `arr.keys()` | iterator of indices | no | O(1) to create |
| `values` | `arr.values()` | iterator of values | no | O(1) to create |
| `entries` | `arr.entries()` | iterator of `[index, value]` | no | O(1) to create |
| `Array.from` | `Array.from(iterable/arrLike, mapFn)` | build array from iterable/array-like | no (static) | O(n) |
| `Array.of` | `Array.of(...items)` | build array from args (fixes `Array(7)` ambiguity) | no (static) | O(n) |
| `toSorted` | `arr.toSorted(cmp)` **[ES2023]** | sort, return NEW array | no | O(n log n) |
| `toReversed` | `arr.toReversed()` **[ES2023]** | reverse, return NEW array | no | O(n) |
| `toSpliced` | `arr.toSpliced(...)` **[ES2023]** | splice, return NEW array | no | O(n) |
| `with` | `arr.with(i, v)` **[ES2023]** | return NEW array with index i replaced | no | O(n) |

> **[ES2023] availability:** `findLast`/`findLastIndex`/`toSorted`/`toReversed`/`toSpliced`/`with`/`toSpliced` need Node 20+. LeetCode's Node runtime (18.x as of most recent versions) may lack the `to*`/`with` family — verify before relying on them; `sort()`/`reverse()`/`splice()` in-place equivalents always work.

### Mutates vs returns-new — quick lookup

| Mutates original | Returns new (leaves original alone) |
|---|---|
| `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin` | `slice`, `concat`, `map`, `filter`, `reduce`, `flat`, `flatMap`, `join`, `find*`, `indexOf`, `includes`, `some`, `every`, `toSorted`, `toReversed`, `toSpliced`, `with` |

> **Gotcha:** `sort()` and `reverse()` mutate AND return the same array — easy to accidentally think you got a copy: `const y = x.sort()` means `y === x`.

### Stack via push/pop — O(1)

```js
const stack = [];
stack.push(1);
stack.push(2);
stack.push(3);
console.log(stack.pop()); // => 3
console.log(stack);       // => [ 1, 2 ]
```

### Queue via shift() — O(n), the trap

```js
const q = [1, 2, 3, 4, 5];
console.log(q.shift()); // => 1   -- looks fine, but this is O(n): every remaining element shifts left
```
> **Gotcha:** `shift()`/`unshift()` are O(n) because the engine re-indexes every remaining element. A loop of n `shift()` calls is **O(n²)**, a classic hidden-complexity trap in interviews. For an efficient FIFO queue, use a **two-pointer index into an array** (increment a `head` pointer instead of shifting) or a proper deque structure — covered in Part III (§ Deque / `Map`-as-queue / manual ring buffer).

### slice — copy a range (non-mutating)

```js
const a = [10, 20, 30, 40, 50];
console.log(a.slice(1, 3));  // => [ 20, 30 ]     -- end exclusive
console.log(a.slice(-2));    // => [ 40, 50 ]     -- negative = from end
console.log(a.slice());      // => [ 10, 20, 30, 40, 50 ]  -- common shallow-copy idiom
console.log(a);              // => [ 10, 20, 30, 40, 50 ]  -- unchanged
```

### splice — the versatile in-place editor

```js
const a = [10, 20, 30, 40, 50];

// remove at index 2, 1 element
const removed = a.splice(2, 1);
console.log(removed); // => [ 30 ]
console.log(a);        // => [ 10, 20, 40, 50 ]

// insert at index 1, delete 0
a.splice(1, 0, 99, 98);
console.log(a); // => [ 10, 99, 98, 20, 40, 50 ]

// replace: delete 2 starting at index 0, insert 1
a.splice(0, 2, 'x');
console.log(a); // => [ 'x', 98, 20, 40, 50 ]
```
> **Complexity:** O(n) — shifts all elements after the splice point, same underlying cost as shift/unshift.

### concat, join

```js
console.log([1,2].concat([3,4], [5])); // => [ 1, 2, 3, 4, 5 ]
console.log([1,2,3].join('-'));         // => '1-2-3'
console.log([1,2,3].join(''));          // => '123'
```

### indexOf / includes — and the NaN difference

```js
const a = [1, NaN, 3];
console.log(a.indexOf(NaN));  // => -1     -- indexOf uses === , NaN !== NaN
console.log(a.includes(NaN)); // => true   -- includes uses SameValueZero, treats NaN as findable
```
> **Gotcha:** This is one of the few reasons to prefer `.includes()` over `.indexOf() !== -1` when NaN might be present. Both otherwise behave the same for membership checks.

### find / findIndex / findLast / findLastIndex

```js
const nums = [5, 12, 8, 130, 44];
console.log(nums.find(n => n > 10));       // => 12
console.log(nums.findIndex(n => n > 10));  // => 1
console.log(nums.findLast(n => n > 10));   // => 130  [ES2023]
console.log(nums.findLastIndex(n => n > 10)); // => 3 [ES2023]
```

### some / every

```js
console.log([1,2,3].some(n => n > 2));  // => true
console.log([1,2,3].every(n => n > 0)); // => true
console.log([1,2,3].every(n => n > 1)); // => false
```

### filter, map, reduce, reduceRight

```js
console.log([1,2,3,4].filter(n => n % 2 === 0)); // => [ 2, 4 ]
console.log([1,2,3].map(n => n * n));             // => [ 1, 4, 9 ]

console.log([1,2,3,4].reduce((acc, n) => acc + n, 0));      // => 10
console.log([1,2,3,4].reduce((acc, n) => acc + n));         // => 10 (no init: acc starts as arr[0])
console.log(['a','b','c'].reduceRight((acc, s) => acc + s)); // => 'cba'
```
> **Gotcha:** `.reduce()` without an initial value throws `TypeError: Reduce of empty array with no initial value` on an empty array — always pass an explicit initial value in CP code to avoid edge-case crashes.

### forEach — no break

```js
[1,2,3].forEach(n => console.log(n)); // prints 1, 2, 3
// there is NO way to `break` a forEach loop early —
// return inside the callback only skips that ONE iteration (like `continue`), never exits the loop.
// Use a plain for/for-of loop, or .some()/.every() abusing early-return, if you need early exit.
```

### flat, flatMap

```js
console.log([1, [2, 3], [4, [5, 6]]].flat());     // => [ 1, 2, 3, 4, [ 5, 6 ] ]  -- default depth 1
console.log([1, [2, [3, [4]]]].flat(Infinity));   // => [ 1, 2, 3, 4 ]
console.log([1,2,3].flatMap(n => [n, n * 2]));    // => [ 1, 2, 2, 4, 3, 6 ]
```

### fill, copyWithin

```js
const a = [1,2,3,4,5];
console.log(a.fill(0, 1, 3)); // => [ 1, 0, 0, 4, 5 ]  -- fills [1,3)

const b = [1,2,3,4,5];
console.log(b.copyWithin(0, 3)); // => [ 4, 5, 3, 4, 5 ]  -- copies from index 3 to index 0
```

### reverse

```js
const a = [1,2,3];
a.reverse();
console.log(a); // => [ 3, 2, 1 ]  -- MUTATES
```

### sort — the lexicographic-default trap

```js
const nums = [10, 1, 21, 2];
console.log(nums.sort());          // => [ 1, 10, 2, 21 ]  -- WRONG for numbers! default sort is by UTF-16 STRING comparison
console.log(nums.sort((a,b) => a - b)); // => [ 1, 2, 10, 21 ]  -- correct ascending numeric sort
console.log(nums.sort((a,b) => b - a)); // => [ 21, 10, 2, 1 ]  -- descending
```
> **Gotcha:** `.sort()` with no comparator converts every element to a **string** and sorts lexicographically. `[10, 1, 21, 2].sort()` gives `[1, 10, 2, 21]`, NOT `[1, 2, 10, 21]`. ALWAYS pass a comparator for numbers: `(a,b) => a - b`. Full sort treatment (stability, custom comparators, sorting objects) is in Part IV.
> **vs C++/Java:** `std::sort` needs an explicit `<` or comparator too but never silently stringifies; Java's `Arrays.sort(int[])` is always numeric (only `Arrays.sort(Object[])` needs `Comparable`/`Comparator`). JS's default-to-string behavior is JS-specific and easy to forget.

### keys / values / entries

```js
const a = ['a', 'b', 'c'];
for (const i of a.keys())    console.log(i);        // 0, 1, 2
for (const v of a.values())  console.log(v);        // 'a', 'b', 'c'
for (const [i, v] of a.entries()) console.log(i, v); // 0 'a', 1 'b', 2 'c'
```

### Array.from, Array.of

```js
console.log(Array.from('abc'));            // => [ 'a', 'b', 'c' ]  -- string is iterable
console.log(Array.from(new Set([1,2,2,3]))); // => [ 1, 2, 3 ]
console.log(Array.from({length: 3}, (_, i) => i * 2)); // => [ 0, 2, 4 ]

console.log(Array.of(7));      // => [ 7 ]      -- Array.of always makes an array OF the args
console.log(Array(7));         // => [ <7 empty items> ]  -- but Array(7) makes length-7 holes! ambiguity trap
console.log(Array.of(1,2,3));  // => [ 1, 2, 3 ]
```
> **Gotcha:** `Array(7)` and `Array(1,2,3)` behave differently — a single numeric argument means "length N", multiple arguments means "these are the elements". `Array.of` removes the ambiguity by always treating arguments as elements.

### toSorted / toReversed / toSpliced / with — non-mutating siblings [ES2023]

```js
const a = [3, 1, 2];
const sorted = a.toSorted((x, y) => x - y);
console.log(sorted); // => [ 1, 2, 3 ]
console.log(a);       // => [ 3, 1, 2 ]  -- original untouched

console.log(a.toReversed());     // => [ 2, 1, 3 ]
console.log(a.toSpliced(1, 1, 9)); // => [ 3, 9, 2 ]
console.log(a.with(0, 99));      // => [ 99, 1, 2 ]
console.log(a);                  // => [ 3, 1, 2 ]  -- still untouched
```
> Requires Node 20+ / V8 with ES2023 support. If targeting an older judge (Node 18), fall back to `[...a].sort(cmp)`, `[...a].reverse()`, manual splice-on-copy, etc.

---

## 10. String and String Methods

Strings in JS are **immutable primitives** — same philosophy as Java's `String`, unlike C++'s mutable `std::string`. Every "modifying" operation (`+`, `.slice`, `.replace`, ...) returns a **brand-new string**; nothing changes the original in place.

> **vs C++:** No `s[i] = 'x'` — strings are not mutable buffers. `s[i]` assignment is silently a no-op (non-strict mode) or fails to compile the mental model — it just does nothing.
```js
let s = "hello";
s[0] = 'H';
console.log(s); // => 'hello'  -- silently unchanged! not an error, just ignored
```

### The O(n²) concatenation trap

```js
// BAD in a loop: each += allocates a new string of growing length -> O(n^2) total
let bad = '';
for (let i = 0; i < 5; i++) bad += i;
console.log(bad); // => '01234'  -- correct output, but O(n^2) for large n

// GOOD: push to an array, join once -> O(n)
const parts = [];
for (let i = 0; i < 5; i++) parts.push(i);
const good = parts.join('');
console.log(good); // => '01234'
```
> **Gotcha:** Repeated `s += x` inside a hot loop is the single most common JS-specific perf bug in string-heavy CP problems (e.g. building large output strings, run-length encoding). V8 does have some rope/cons-string optimization for `+=` that softens this in practice, but the array-join pattern is the reliable, guaranteed-safe idiom — always prefer it for anything that could be large.

### No char type

```js
const s = "abc";
console.log(s[1]);        // => 'b'   -- a 1-character STRING, not a char
console.log(s.charAt(1));  // => 'b'
console.log(typeof s[1]);  // => 'string'
```
> **vs C++/Java:** No `char` primitive. Every "character" is a length-1 string. This matters for comparisons (`s[i] === 'a'` works fine) and for arithmetic — you must go through char codes.

### Char arithmetic — charCodeAt / fromCharCode

```js
console.log('a'.charCodeAt(0));         // => 97
console.log(String.fromCharCode(97));   // => 'a'

// common idiom: letter -> 0-25 index
const c = 'd';
console.log(c.charCodeAt(0) - 97);      // => 3   (charCodeAt('a') = 97)

// common idiom: index -> letter
const k = 3;
console.log(String.fromCharCode(97 + k)); // => 'd'

// uppercase equivalent uses 65 ('A')
console.log('Z'.charCodeAt(0) - 65); // => 25
```

### codePointAt / fromCodePoint — unicode note

```js
console.log('a'.codePointAt(0));         // => 97, same as charCodeAt for BMP chars
console.log('😀'.length);                // => 2   -- surrogate pair, length counts UTF-16 code units not "characters"!
console.log([...'😀'].length);           // => 1   -- spread iterates by CODE POINT, correctly counts as 1
console.log(String.fromCodePoint(128512)); // => '😀'
```
> **Gotcha:** `.length`, `charCodeAt`, and `s[i]` all operate on UTF-16 code units, so characters outside the Basic Multilingual Plane (most emoji, some CJK extensions) count as 2. Rarely matters for typical CP problems (usually lowercase a-z), but worth knowing if a problem statement mentions unicode.

### Complete method table

| Method | Signature | What it does | Notes |
|---|---|---|---|
| `length` | `s.length` | number of UTF-16 code units | property, not a method (no `()`) |
| `charAt` | `s.charAt(i)` | char at i, `''` if out of range | never throws |
| `charCodeAt` | `s.charCodeAt(i)` | UTF-16 code unit at i | `NaN` if out of range |
| `at` | `s.at(i)` **[ES2022]** | char at i, supports negative i | `undefined` if out of range |
| `indexOf` | `s.indexOf(sub, from)` | first index of substring, -1 if absent | |
| `lastIndexOf` | `s.lastIndexOf(sub, from)` | last index of substring | |
| `includes` | `s.includes(sub, from)` | true/false | |
| `startsWith` | `s.startsWith(sub, from)` | true/false | |
| `endsWith` | `s.endsWith(sub, endPos)` | true/false | |
| `slice` | `s.slice(start, end)` | substring, supports NEGATIVE indices | end exclusive |
| `substring` | `s.substring(start, end)` | substring, NO negatives (clamped to 0) | swaps args if start>end |
| `substr` | `s.substr(start, len)` | **deprecated**, start+length form | avoid in new code |
| `split` | `s.split(sep, limit)` | string → array | sep can be string OR regex |
| `replace` | `s.replace(pat, repl)` | replace FIRST match (string pat) or per `/g` flag (regex) | returns new string |
| `replaceAll` | `s.replaceAll(pat, repl)` **[ES2021]** | replace ALL matches | string pat needs no `/g`; regex pat MUST have `/g` or throws |
| `toUpperCase` / `toLowerCase` | `s.toUpperCase()` | case conversion | |
| `trim` / `trimStart` / `trimEnd` | `s.trim()` | strip whitespace | |
| `padStart` / `padEnd` | `s.padStart(len, pad)` | pad to length | common for zero-padding numbers |
| `repeat` | `s.repeat(n)` | repeat string n times | throws on negative n |
| `concat` | `s.concat(...strs)` | concatenate | prefer `+`/template literals in practice |
| `match` | `s.match(regex)` | regex match(es) | array or null |
| `matchAll` | `s.matchAll(regex)` | iterator of all matches (needs `/g`) | |
| `search` | `s.search(regex)` | index of first regex match, -1 if none | |
| `localeCompare` | `s.localeCompare(other)` | locale-aware comparison, returns -1/0/1 | slower than `<`/`>`, use for i18n only |

### slice vs substring — the negative-index trap

```js
const s = 'hello world';
console.log(s.slice(-5));      // => 'world'    -- slice supports negative index (from end)
console.log(s.substring(-5));  // => 'hello world' -- substring CLAMPS negatives to 0! totally different result
console.log(s.slice(0, -6));   // => 'hello'
console.log(s.substring(0, 3)); // => 'hel'
```
> **Gotcha:** `substring` and `slice` look interchangeable for positive indices but diverge completely on negative ones. Default to `slice` — it's strictly more capable and its negative-index behavior matches `.at()` and array `.slice()`.

### indexOf / includes / startsWith / endsWith

```js
const s = 'hello world';
console.log(s.indexOf('world'));  // => 6
console.log(s.indexOf('xyz'));    // => -1     -- ALWAYS check for -1, don't assume found
console.log(s.includes('wor'));   // => true
console.log(s.startsWith('hel')); // => true
console.log(s.endsWith('rld'));   // => true
```

### split — string vs regex, char array, empty-string behavior

```js
console.log('a,b,,c'.split(','));    // => [ 'a', 'b', '', 'c' ]   -- keeps empty strings between consecutive delimiters
console.log('a b  c'.split(' '));    // => [ 'a', 'b', '', 'c' ]   -- double space -> empty entry! naive split(' ') trap
console.log('a b  c'.split(/\s+/));  // => [ 'a', 'b', 'c' ]       -- regex \s+ correctly collapses whitespace runs
console.log('hello'.split(''));      // => [ 'h', 'e', 'l', 'l', 'o' ]  -- char array idiom
console.log('a,b,c'.split(',', 2));  // => [ 'a', 'b' ]            -- limit param caps result length
console.log(''.split(','));          // => [ '' ]                 -- splitting empty string gives array with one empty string
```
> **Gotcha:** `str.split(' ')` on input with multiple consecutive spaces produces empty-string entries. If a problem statement says "words separated by spaces" but doesn't guarantee single spaces, use `str.split(/\s+/).filter(Boolean)` or `str.trim().split(/\s+/)`.

### replace / replaceAll — the first-match-only trap

```js
const s = 'a-b-a-b';
console.log(s.replace('a', 'X'));       // => 'X-b-a-b'   -- only the FIRST 'a'!
console.log(s.replace(/a/g, 'X'));      // => 'X-b-X-b'   -- /g flag needed to replace all via regex
console.log(s.replaceAll('a', 'X'));    // => 'X-b-X-b'   -- replaceAll [ES2021] with a plain string replaces all, no /g needed
console.log(s.replaceAll(/a/g, 'X'));   // => 'X-b-X-b'   -- regex arg to replaceAll REQUIRES the /g flag or it throws TypeError
```
> **Gotcha:** `.replace(str, ...)` only ever replaces the first occurrence when the pattern is a plain string — to replace all with `.replace`, you need a regex with `/g`. `.replaceAll` fixes the plain-string case but still demands `/g` if you pass a regex.

### case, trim, pad, repeat

```js
console.log('Hello'.toUpperCase());      // => 'HELLO'
console.log('Hello'.toLowerCase());      // => 'hello'
console.log('  hi  '.trim());            // => 'hi'
console.log('  hi  '.trimStart());       // => 'hi  '
console.log('  hi  '.trimEnd());         // => '  hi'
console.log('5'.padStart(3, '0'));       // => '005'   -- zero-padding idiom
console.log('5'.padEnd(3, '0'));         // => '500'
console.log('ab'.repeat(3));             // => 'ababab'
```

### match, matchAll, search (brief)

```js
console.log('a1b2c3'.match(/\d/g));      // => [ '1', '2', '3' ]
console.log('a1b2c3'.search(/\d/));      // => 1  -- index of first match
for (const m of 'a1b2'.matchAll(/\d/g)) console.log(m[0], m.index); // '1' 1, then '2' 3
```

### String comparison — lexicographic with `<`/`>`

```js
console.log('apple' < 'banana'); // => true
console.log('Zebra' < 'apple');  // => true   -- uppercase letters sort BEFORE lowercase (UTF-16 code unit order, 'Z'=90 < 'a'=97)
console.log('a'.localeCompare('b')); // => -1  -- locale-aware, use only for real i18n sorting
```
> **vs Java:** No need to call `.compareTo()` — JS lets you use `<`, `>`, `<=`, `>=`, `===` directly on strings for lexicographic UTF-16 comparison. Simpler than Java, but remember it's code-unit order, not locale order (careful with mixed case).

### Converting: string ↔ array, string ↔ number

```js
// string -> array of chars
console.log([...'hello']);       // => [ 'h', 'e', 'l', 'l', 'o' ]
console.log('hello'.split(''));  // => [ 'h', 'e', 'l', 'l', 'o' ]  -- equivalent for BMP chars; [...] is unicode-safe (splits by code point)

// array -> string
console.log(['h','e','l','l','o'].join('')); // => 'hello'

// string -> number
console.log(Number('42'));    // => 42
console.log(+'42');           // => 42       -- unary plus, same effect, terser
console.log(parseInt('42px')); // => 42      -- parseInt stops at first non-digit, Number() would give NaN
console.log(Number('42px'));   // => NaN
console.log(parseFloat('3.14abc')); // => 3.14

// number -> string
console.log(String(42));   // => '42'
console.log((42).toString()); // => '42'
console.log(`${42}`);      // => '42'
```

### Reversing a string

```js
const s = 'hello';
console.log([...s].reverse().join('')); // => 'olleh'
// there is no s.reverse() -- strings have no in-place methods at all
```

### Sorting characters

```js
const s = 'dbca';
console.log([...s].sort().join('')); // => 'abcd'   -- default lexicographic sort is CORRECT for single chars
```

### Template literals for building output

```js
const name = 'world', n = 3;
console.log(`Hello, ${name}! You have ${n + 1} items.`); // => 'Hello, world! You have 4 items.'
// multi-line strings without \n concatenation:
const block = `line1
line2`;
console.log(block); // => 'line1\nline2'
```

### Building output efficiently — the CP rule

```js
// Rule: never += in a loop for large output. Push to array, join once.
function buildCSV(rows) {
  const lines = [];
  for (const row of rows) {
    lines.push(row.join(','));
  }
  return lines.join('\n');
}
console.log(buildCSV([[1,2,3], [4,5,6]])); // => '1,2,3\n4,5,6'
```

---

## 11. TypedArrays and Pair/Tuple Representations

**TypedArrays** are fixed-length, single-numeric-type, contiguous-memory array views — the closest JS analogue to a C++ `int arr[n]` or `std::array<int,n>`. Regular JS arrays are flexible but boxed/dictionary-backed under the hood when not "packed"; typed arrays guarantee a raw contiguous numeric buffer, which is both faster and far more memory-efficient for large fixed-size numeric data.

### Type table

| Type | Element size | Range | Notes |
|---|---|---|---|
| `Int8Array` | 1 byte | -128 to 127 | |
| `Uint8Array` | 1 byte | 0 to 255 | |
| `Uint8ClampedArray` | 1 byte | 0 to 255, clamps instead of wraps | used for image/canvas data |
| `Int16Array` | 2 bytes | -32768 to 32767 | |
| `Uint16Array` | 2 bytes | 0 to 65535 | |
| `Int32Array` | 4 bytes | -2^31 to 2^31-1 | most common for CP int arrays |
| `Uint32Array` | 4 bytes | 0 to 2^32-1 | |
| `Float32Array` | 4 bytes | IEEE 754 single | |
| `Float64Array` | 8 bytes | IEEE 754 double | same precision as regular JS numbers |
| `BigInt64Array` | 8 bytes | -2^63 to 2^63-1 | elements are `BigInt`, not `number` |
| `BigUint64Array` | 8 bytes | 0 to 2^64-1 | elements are `BigInt` |

### Creation

```js
const a = new Int32Array(5);
console.log(a); // => Int32Array(5) [ 0, 0, 0, 0, 0 ]   -- ZERO-FILLED by default, unlike `new Array(5)`!

const b = Int32Array.from([1, 2, 3]);
console.log(b); // => Int32Array(3) [ 1, 2, 3 ]

const buf = new ArrayBuffer(16);           // raw 16-byte buffer
const c = new Int32Array(buf);             // view: 4 x int32 over those 16 bytes
console.log(c.length); // => 4
```
> **Gotcha (in your favor here):** Unlike `new Array(n)` which creates holes, `new Int32Array(n)` (and all typed arrays) is **always zero-initialized**. No `.fill(0)` needed.

### When they beat regular arrays in CP

Use typed arrays when: the array is numeric, large (10^5+), and fixed-size — e.g. DP tables, visited/parent arrays, adjacency counts, frequency arrays, BFS distance arrays. Real speedups come from: no boxing (raw ints, not JS `number` objects-in-disguise), guaranteed contiguous memory (cache-friendly), and no dictionary-mode fallback risk. For small or dynamically-resized data, the setup overhead usually isn't worth it — stick with regular arrays plus `Array(n).fill(0)`.

### What they lack vs regular arrays

```js
const t = new Int32Array([3, 1, 2]);

// NO push/pop/shift/unshift/splice — fixed size, cannot resize
console.log(typeof t.push); // => 'undefined'

// HAVE: set, subarray, fill, sort, map, filter, reduce, forEach, indexOf, includes, slice, join, some, every...
t.set([9, 9], 0);          // bulk-write from offset 0
console.log(t);              // => Int32Array(3) [ 9, 9, 2 ]

const view = t.subarray(1, 3); // VIEW (shares memory!), not a copy
view[0] = 100;
console.log(t); // => Int32Array(3) [ 9, 100, 2 ]  -- mutating the subarray mutated t!

console.log(t.slice(1, 3)); // => Int32Array(2) [ 100, 2 ]  -- slice DOES copy (unlike subarray)
```
> **Gotcha:** `.subarray()` returns a live VIEW over the same underlying buffer — writes through it mutate the original. `.slice()` on a typed array does copy, same as regular arrays. This view-vs-copy distinction doesn't exist for regular array `.slice()`/manual indexing.

### Sort is numeric by default — a pleasant difference

```js
const nums = new Int32Array([10, 1, 21, 2]);
console.log(nums.sort()); // => Int32Array(4) [ 1, 2, 10, 21 ]  -- CORRECT numeric sort, no comparator needed!
```
> Unlike regular `Array.prototype.sort`, `TypedArray.prototype.sort()` defaults to ascending **numeric** order — no lexicographic trap. You can still pass a custom comparator for descending, etc.

### Overflow wraps (no exception)

```js
const t = new Int32Array(1);
t[0] = 2147483647;      // Int32 max
t[0] = t[0] + 1;
console.log(t[0]); // => -2147483648  -- silently WRAPS, like C++ signed overflow (UB in C++, well-defined wraparound here)

const u8 = new Uint8Array(1);
u8[0] = 256;
console.log(u8[0]); // => 0  -- wraps mod 256
u8[0] = -1;
console.log(u8[0]); // => 255 -- wraps
```
> **vs C++:** Signed overflow in C++ is undefined behavior; in a `TypedArray`, overflow is well-defined modular wraparound (two's complement), similar in spirit to how it behaves in practice on most C++ compilers, and identical to Java's `int` overflow semantics.

### Typical CP uses

```js
// visited array for grid BFS/DFS
const n = 5, m = 5;
const visited = new Uint8Array(n * m); // 0 = unvisited, 1 = visited; far less memory than Array(n*m).fill(false)

// DP table (1D, flattened)
const dp = new Int32Array(n + 1).fill(-1);

// distance array for BFS
const dist = new Int32Array(n).fill(-1);
```

---

### Pair / Tuple representations

JS has **no built-in tuple type** — unlike C++ `std::pair`/`std::tuple`. The idiomatic choices:

#### (a) `[a, b]` array literal

```js
const p = [3, 7];
const [x, y] = p; // destructure
console.log(x, y); // => 3 7
```
Cheap, easy destructuring, natural for return values (`return [dist, parent]`). **But**: arrays compare by reference, so `[1,2]` cannot be used directly as a `Map`/`Set` key for deduplication — `map.set([1,2], v)` and later `map.get([1,2])` will NOT find it (different array instances).
```js
const m = new Map();
m.set([1,2], 'x');
console.log(m.get([1,2])); // => undefined  -- different array object, reference mismatch
```

#### (b) Encode two ints into one number

```js
// safe when both a, b < ~2^26 (so a*1e6+b, or a shifted encoding, stays within Number.MAX_SAFE_INTEGER = 2^53-1)
function encode(a, b) { return a * 1_000_000 + b; }
function decode(code) { return [Math.floor(code / 1_000_000), code % 1_000_000]; }

console.log(encode(12, 345)); // => 12000345
console.log(decode(12000345)); // => [ 12, 345 ]
```
Fast, usable directly as a `Map`/`Set` key or array index (if small enough), but requires knowing safe bounds for both components in advance.

#### (c) String key — the common visited-coordinates idiom

```js
const visited = new Set();
function key(r, c) { return `${r},${c}`; }

visited.add(key(2, 3));
console.log(visited.has(key(2, 3))); // => true
console.log(visited.has(`2,3`));     // => true  -- string equality works as expected for Set/Map keys
```
> **Gotcha / cost:** String keys are the most common and safest idiom (no overflow-bound math needed, works for any int range including negatives), but each `has`/`add` involves string construction + hashing — measurably slower than a numeric-encoded key or a nested array/typed-array lookup. For hot loops (10^6+ ops), prefer numeric encoding when bounds allow.

#### (d) BigInt packing for 64-bit coordinates

```js
function encodeBig(a, b) {
  return (BigInt(a) << 32n) | BigInt(b >>> 0); // pack two 32-bit ints into one BigInt
}
console.log(encodeBig(5, 10)); // => 21474836490n
```
Handles arbitrarily large components exactly, but `BigInt` arithmetic is significantly slower than `number` arithmetic — reserve for when values genuinely exceed `Number.MAX_SAFE_INTEGER` (2^53-1).

### Decision table

| Need | Best representation |
|---|---|
| Return a pair of values from a function | `[a, b]` array (destructure at call site) |
| Use as `Map`/`Set` key, both ints small (< ~2^26) | encoded number: `a * BASE + b` |
| Use as `Map`/`Set` key, general/negative ints, simplicity > raw speed | string key `` `${a},${b}` `` |
| Use as `Map`/`Set` key, need max raw-int-key speed at any int range | BigInt packing (if beyond safe-int) or numeric encoding (if within) |
| Sort an array of pairs | array of `[a,b]`, custom comparator (see below) |
| Store visited grid cells, dims known & moderate | 2D typed array or flattened `Int32Array`/`Uint8Array` (`r*cols+c` as index) — fastest of all |

### Sorting an array of pairs by first, then second

```js
const pairs = [[3, 2], [1, 5], [3, 1], [1, 2]];
pairs.sort((a, b) => a[0] - b[0] || a[1] - b[1]);
console.log(pairs); // => [ [ 1, 2 ], [ 1, 5 ], [ 3, 1 ], [ 3, 2 ] ]
```
> The `a[0] - b[0] || a[1] - b[1]` idiom: `||` falls through to the second comparison only when the first is `0` (falsy). This chained-comparator pattern extends to any number of tiebreak keys and is the standard CP idiom for multi-key sorts (full sort-stability discussion in Part IV).

---

# PART III — MAP, SET AND THE MISSING STRUCTURES

## 12. Map and Object-as-Map

`Map` (ES6) is insertion-ordered and accepts **any** key type — primitives, objects (by reference identity), functions, even `NaN` (which works as a key because `Map` uses SameValueZero equality, unlike `===`).

### 12.1 Complete method table

| Method / Property | What it does | Complexity |
|---|---|---|
| `new Map()` / `new Map(iterable)` | create, optionally from `[[k,v], ...]` pairs | O(n) for the iterable form |
| `map.set(k, v)` | insert/update; **returns `map`** (chainable) | O(1) avg |
| `map.get(k)` | value for key, or `undefined` | O(1) avg |
| `map.has(k)` | boolean membership test | O(1) avg |
| `map.delete(k)` | removes entry; returns `true` if it existed | O(1) avg |
| `map.size` | **property, not a method** — no `()` | O(1) |
| `map.clear()` | remove all entries | O(n) |
| `map.keys()` | iterator over keys, insertion order | O(1) to get iterator |
| `map.values()` | iterator over values, insertion order | O(1) to get iterator |
| `map.entries()` | iterator over `[k, v]` pairs (default iterator) | O(1) to get iterator |
| `map.forEach((v, k, map) => {})` | iterate — **value first, then key** | O(n) |

```js
const m = new Map();
m.set('a', 1).set('b', 2).set('c', 3); // chainable
console.log(m.size);        // => 3
console.log(m.get('b'));    // => 2
console.log(m.has('z'));    // => false
m.delete('a');
console.log([...m.keys()]); // => [ 'b', 'c' ]
```

> **Gotcha:** `map.size` is a getter property. Writing `map.size()` throws `TypeError: map.size is not a function`.

### 12.2 Iteration

```js
const m = new Map([['a', 1], ['b', 2]]);

// for...of over entries (default iterator) — destructure directly
for (const [k, v] of m) {
  console.log(k, v);
}
// => a 1
// => b 2

// forEach: callback signature is (value, key, map) — VALUE FIRST
m.forEach((v, k) => console.log(k, '->', v));
// => a -> 1
// => b -> 2
```

> **Gotcha:** `forEach`'s callback order is `(value, key)`, matching `Array.forEach`'s `(element, index)` — but the opposite of how you'd naturally say "key, value". Mixing this up silently swaps your variables since both are typically generic names like `v`/`k`.

### 12.3 Frequency-count idiom

The single most common Map use in DSA — counting occurrences:

```js
function countFreq(arr) {
  const freq = new Map();
  for (const x of arr) {
    freq.set(x, (freq.get(x) || 0) + 1);
  }
  return freq;
}

console.log(countFreq([1, 2, 2, 3, 3, 3]));
// => Map(3) { 1 => 1, 2 => 2, 3 => 3 }
```

### 12.4 Map vs plain Object `{}` as a hash map

This is a recurring source of bugs when porting C++/Java `unordered_map` habits to JS objects.

| Aspect | `Map` | Object `{}` |
|---|---|---|
| Key types | ANY type, preserved as-is | Coerced to **string** (or Symbol) |
| Numeric keys | `1` and `'1'` are different keys | `1` and `'1'` collide (both become `'1'`) |
| Object keys | keyed by reference, distinct objects stay distinct | keyed by `String(obj)` → `'[object Object]'` for ALL plain objects — they collide! |
| Prototype pollution | none — clean, empty by default | inherits from `Object.prototype` (`'constructor'`, `'toString'`, `'hasOwnProperty'`, ...) unless created with `Object.create(null)` |
| `.size` | O(1) built-in property | must compute `Object.keys(obj).length` — O(n) |
| Iteration order | guaranteed insertion order, any key type | insertion order EXCEPT integer-like string keys are reordered numerically first |
| Add/delete performance | O(1), no de-opt | `delete obj.k` can de-optimize V8's hidden-class machinery in hot loops |
| JSON | not directly serializable (`JSON.stringify(map)` => `'{}'`) | native JSON shape |

```js
// TRAP 1: numeric-vs-string key collision on plain objects
const obj = {};
obj[1] = 'number-key';
obj['1'] = 'string-key';
console.log(obj);         // => { '1': 'string-key' }  -- only ONE entry, second overwrote first!

const map = new Map();
map.set(1, 'number-key');
map.set('1', 'string-key');
console.log(map.size);    // => 2  -- correctly distinct keys

// TRAP 2: object keys stringify to the same bucket
const badObj = {};
const k1 = { id: 1 }, k2 = { id: 2 };
badObj[k1] = 'first';
badObj[k2] = 'second'; // overwrites — both keys stringify to '[object Object]'
console.log(badObj);   // => { '[object Object]': 'second' }
console.log(Object.keys(badObj).length); // => 1

const goodMap = new Map();
goodMap.set(k1, 'first');
goodMap.set(k2, 'second'); // distinct — keyed by reference
console.log(goodMap.size); // => 2

// TRAP 3: inherited keys on a plain object
const obj2 = {};
console.log('toString' in obj2);       // => true (inherited, not own!)
console.log(obj2.hasOwnProperty('toString')); // => false
// Safe patterns:
const clean = Object.create(null);      // no prototype at all
console.log('toString' in clean);       // => false
const safeMap = new Map();              // Map never has this problem
```

> **Gotcha:** `for...in` over a plain object walks inherited enumerable properties too. Always guard with `Object.hasOwn(obj, key)` (Node 16.9+) or use `Object.keys(obj)` for iteration instead of `for...in`.

**When to use which:**
- Use **`Map`** when: keys are non-string (numbers, objects, coordinate tuples), you add/delete frequently, you need `.size`, or you need reliable insertion order.
- Use a plain **object / `Object.create(null)`** when: data is JSON-shaped with fixed string keys, or you want marginally faster access for pure string keys and don't need iteration order guarantees beyond what objects already give.

### 12.5 Coordinate-pair keys via string encoding

JS has no tuple/pair key type, so grid/graph problems encode `(r, c)` as a string key:

```js
const visited = new Map();
const encode = (r, c) => `${r},${c}`;

visited.set(encode(2, 3), true);
console.log(visited.has(encode(2, 3))); // => true
console.log(visited.has(encode(3, 2))); // => false (order matters in the string)
```

> **vs C++:** `std::map<pair<int,int>, T>` or `unordered_map` with a custom hash just works with a real pair key. JS has no pair type, so string-encoding (or a `Set`/`Map` of `r * COLS + c` as a single integer key) is the idiomatic replacement.

### 12.6 Converting Map ↔ object ↔ array

```js
const map = new Map([['a', 1], ['b', 2]]);

// Map -> array of [k, v] pairs
const arr = [...map];                       // => [ [ 'a', 1 ], [ 'b', 2 ] ]

// Map -> plain object (string keys only — non-string keys get stringified)
const obj = Object.fromEntries(map);         // => { a: 1, b: 2 }

// object -> Map
const backToMap = new Map(Object.entries(obj)); // => Map(2) { 'a' => 1, 'b' => 2 }

// Map -> sorted-by-value array (common DSA pattern: top-K frequent)
const sorted = [...map.entries()].sort((a, b) => b[1] - a[1]);
console.log(sorted); // => [ [ 'b', 2 ], [ 'a', 1 ] ]
```

---

## 13. Set

`Set` (ES6) stores unique values, insertion-ordered, any type. Objects are compared by reference; `NaN` can be stored once (SameValueZero, same as `Map`).

### 13.1 Complete method table

| Method / Property | What it does | Complexity |
|---|---|---|
| `new Set()` / `new Set(iterable)` | create, optionally seeded from an array/iterable (dedupes) | O(n) |
| `set.add(v)` | insert; **returns `set`** (chainable) | O(1) avg |
| `set.has(v)` | membership test | O(1) avg |
| `set.delete(v)` | removes value; returns `true` if it existed | O(1) avg |
| `set.size` | property, not a method | O(1) |
| `set.clear()` | remove all values | O(n) |
| `set.keys()` / `set.values()` | identical iterators over values (alias for consistency with Map) | O(1) to get |
| `set.entries()` | iterator of `[v, v]` pairs (again, for Map-interface consistency) | O(1) to get |
| `set.forEach((v, v2, set) => {})` | iterate values | O(n) |

```js
const s = new Set();
s.add(1).add(2).add(2).add(3); // chainable, dup ignored
console.log(s.size);   // => 3
console.log([...s]);   // => [ 1, 2, 3 ]
console.log(s.has(2)); // => true
s.delete(2);
console.log([...s]);   // => [ 1, 3 ]
```

### 13.2 O(1) membership test — replaces `array.includes`

```js
const arr = Array.from({ length: 100000 }, (_, i) => i);

// SLOW: array.includes is O(n) per call
console.time('array');
const arrHas = arr.includes(99999); // scans up to 100000 elements
console.timeEnd('array'); // measurable

// FAST: Set.has is O(1) avg per call
const set = new Set(arr);
console.time('set');
const setHas = set.has(99999);
console.timeEnd('set'); // negligible
```

Classic BFS/DFS `visited` set:

```js
function bfsVisitedDemo(graph, start) {
  const visited = new Set([start]);
  const queue = [start];
  const order = [];
  let head = 0;
  while (head < queue.length) {
    const node = queue[head++];
    order.push(node);
    for (const next of graph[node] || []) {
      if (!visited.has(next)) {
        visited.add(next);
        queue.push(next);
      }
    }
  }
  return order;
}

console.log(bfsVisitedDemo({ 1: [2, 3], 2: [4], 3: [4], 4: [] }, 1));
// => [ 1, 2, 3, 4 ]
```

> **vs C++/Java:** this is the JS analogue of `unordered_set<int> visited` / `HashSet<Integer> visited`. Same O(1) average-case guarantee.

### 13.3 Dedupe an array

```js
const dup = [3, 1, 2, 3, 1, 4];
const unique = [...new Set(dup)];
console.log(unique); // => [ 3, 1, 2, 4 ]  (order preserved, first occurrence)
```

### 13.4 Set operations — no built-ins (mostly)

Unlike C++ `std::set_union`/`set_intersection`/`set_difference` on sorted ranges, JS `Set` has (historically) **no** built-in union/intersection/difference. You build them manually with spreads + `filter`/`has`.

> **vs C++:** `<algorithm>` gives you `set_union`, `set_intersection`, `set_difference`, `set_symmetric_difference` on sorted ranges. Java has none built-in either (you write the loop, same as JS). **[ES2024]** added real `Set.prototype` methods (`union`, `intersection`, `difference`, `symmetricDifference`, `isSubsetOf`, `isSupersetOf`, `isDisjointFrom`) — available in Node 22+ / very recent V8. **Most judges still run older Node, so don't rely on them** — use the manual idioms below unless you've confirmed the runtime supports them.

```js
const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);

// Union
const union = new Set([...a, ...b]);
console.log([...union]); // => [ 1, 2, 3, 4, 5, 6 ]

// Intersection
const intersection = new Set([...a].filter(x => b.has(x)));
console.log([...intersection]); // => [ 3, 4 ]

// Difference (a - b)
const difference = new Set([...a].filter(x => !b.has(x)));
console.log([...difference]); // => [ 1, 2 ]

// [ES2024, Node 22+] — native, only if the runtime is known to support it:
// console.log([...a.union(b)]);
// console.log([...a.intersection(b)]);
// console.log([...a.difference(b)]);
```

### 13.5 Objects in a Set — reference-identity trap

```js
const s = new Set();
s.add({ x: 1, y: 2 });
s.add({ x: 1, y: 2 }); // a DIFFERENT object, even though "equal" in shape
console.log(s.size); // => 2  -- NOT deduped!
```

> **Gotcha:** For coordinate/state dedup (e.g. visited `(r, c)` cells, or visited board states), never put arrays/objects directly into a `Set` — encode as a string (or a single packed integer) first, exactly as in §12.5.

```js
// Correct coordinate-set pattern
const visited = new Set();
const key = (r, c) => `${r},${c}`;
visited.add(key(0, 0));
console.log(visited.has(key(0, 0))); // => true
console.log(visited.has(key(0, 1))); // => false
```

### 13.6 WeakSet / WeakMap (brief)

`WeakMap`/`WeakSet` hold only object keys, are not iterable, have no `.size`, and allow garbage collection of keys with no other references — useful for memory-safe metadata caches, but essentially never needed in DSA/CP since you can't iterate or size them (algorithms need enumeration). Skip these for contest code; mentioned only so you recognize them if seen.

---

## 14. Stack, Queue and Deque

### 14.1 Stack — trivial, just use an array

```js
const stack = [];
stack.push(1);       // O(1) amortized
stack.push(2);
stack.push(3);
console.log(stack.pop());  // => 3
console.log(stack.at(-1)); // => 2   (peek, O(1), Node 16.6+)
console.log(stack.length); // => 2
```

| Method | What it does | Complexity |
|---|---|---|
| `arr.push(x)` | push to top (end of array) | O(1) amortized |
| `arr.pop()` | pop from top, returns removed element (`undefined` if empty) | O(1) |
| `arr.at(-1)` | peek top without removing | O(1) |
| `arr.length === 0` | empty check | O(1) |

> **vs C++/Java:** direct analogue of `std::stack` / `java.util.Deque` used as a stack (`push`/`pop`/`peek`). No wrapper needed in JS — the array IS the stack.

### 14.2 Queue — THE #1 JS-DSA PERFORMANCE BUG

```js
// NAIVE QUEUE — DO NOT USE FOR BFS ON LARGE INPUT
const queue = [];
queue.push(1);
queue.push(2);
const first = queue.shift(); // removes index 0
```

> **Gotcha — read this twice:** `Array.prototype.shift()` removes the first element and **re-indexes every remaining element**, making it **O(n)** per call. A BFS that does `queue.shift()` once per node turns an O(V+E) algorithm into **O(n^2)** overall. On a graph/grid with 10^5+ nodes this reliably **TLEs** — it is the single most common JS-specific performance bug in competitive programming. `push`+`pop` (stack) are both O(1); `push`+`shift` (naive queue) is O(1) + O(n) = the trap.

**Fix (a): two-pointer / head-index array queue — simplest, fast enough for ~all BFS**

```js
class ArrayQueue {
  constructor() {
    this._data = [];
    this._head = 0;
  }
  push(x) {                 // enqueue, O(1) amortized
    this._data.push(x);
  }
  shift() {                 // dequeue, O(1) amortized — NO re-indexing
    if (this._head >= this._data.length) return undefined;
    const val = this._data[this._head];
    this._data[this._head] = undefined; // drop reference, help GC
    this._head++;
    return val;
  }
  front() {
    return this._head < this._data.length ? this._data[this._head] : undefined;
  }
  get length() {
    return this._data.length - this._head;
  }
  get isEmpty() {
    return this._head >= this._data.length;
  }
}

// BFS usage
function bfs(graph, start) {
  const q = new ArrayQueue();
  const visited = new Set([start]);
  const order = [];
  q.push(start);
  while (!q.isEmpty) {
    const node = q.shift();
    order.push(node);
    for (const next of graph[node] || []) {
      if (!visited.has(next)) {
        visited.add(next);
        q.push(next);
      }
    }
  }
  return order;
}

console.log(bfs({ 1: [2, 3], 2: [4], 3: [4], 4: [] }, 1));
// => [ 1, 2, 3, 4 ]
```

The head pointer never resets, so `_data` only grows — fine for a single BFS run (bounded by V). If you need long-lived reuse (millions of pushes across many operations), periodically compact: `if (this._head > this._data.length / 2) { this._data = this._data.slice(this._head); this._head = 0; }`.

> **Note:** for the overwhelming majority of BFS problems, this head-index array queue is all you need. Reach for the full `Deque` below only when you must push/pop from **both** ends (monotonic deque, sliding-window maximum, 0-1 BFS).

**Fix (b): a complete `Deque` — O(1) at both ends**

Implemented as a doubly-linked list (simplest to get fully correct; avoids capacity-doubling edge cases of a circular buffer):

```js
class DequeNode {
  constructor(val) {
    this.val = val;
    this.prev = null;
    this.next = null;
  }
}

class Deque {
  constructor() {
    this._head = null; // front
    this._tail = null; // back
    this._size = 0;
  }

  get size() { return this._size; }
  get isEmpty() { return this._size === 0; }

  pushBack(val) {                  // addLast, O(1)
    const node = new DequeNode(val);
    if (this._tail === null) {
      this._head = this._tail = node;
    } else {
      node.prev = this._tail;
      this._tail.next = node;
      this._tail = node;
    }
    this._size++;
  }

  pushFront(val) {                 // addFirst, O(1)
    const node = new DequeNode(val);
    if (this._head === null) {
      this._head = this._tail = node;
    } else {
      node.next = this._head;
      this._head.prev = node;
      this._head = node;
    }
    this._size++;
  }

  popBack() {                      // removeLast, O(1)
    if (this._tail === null) return undefined;
    const val = this._tail.val;
    this._tail = this._tail.prev;
    if (this._tail === null) this._head = null;
    else this._tail.next = null;
    this._size--;
    return val;
  }

  popFront() {                     // removeFirst, O(1)
    if (this._head === null) return undefined;
    const val = this._head.val;
    this._head = this._head.next;
    if (this._head === null) this._tail = null;
    else this._head.prev = null;
    this._size--;
    return val;
  }

  front() { return this._head === null ? undefined : this._head.val; }
  back()  { return this._tail === null ? undefined : this._tail.val; }

  // aliases matching common naming conventions
  push(val)  { this.pushBack(val); }   // enqueue-like
  pop()      { return this.popBack(); }
  shift()    { return this.popFront(); }
  unshift(val) { this.pushFront(val); }

  *[Symbol.iterator]() {
    let cur = this._head;
    while (cur !== null) { yield cur.val; cur = cur.next; }
  }

  toArray() { return [...this]; }
}

// Quick self-test
const dq = new Deque();
dq.pushBack(2); dq.pushBack(3); dq.pushFront(1); dq.pushFront(0);
console.log(dq.toArray());  // => [ 0, 1, 2, 3 ]
console.log(dq.popFront()); // => 0
console.log(dq.popBack());  // => 3
console.log(dq.toArray());  // => [ 1, 2 ]
console.log(dq.size);       // => 2
```

**Monotonic-deque example: sliding window maximum (classic use of `Deque`)**

```js
function maxSlidingWindow(nums, k) {
  const dq = new Deque();   // stores INDICES, values kept decreasing front->back
  const res = [];
  for (let i = 0; i < nums.length; i++) {
    while (!dq.isEmpty && dq.front() <= i - k) dq.popFront();
    while (!dq.isEmpty && nums[dq.back()] <= nums[i]) dq.popBack();
    dq.pushBack(i);
    if (i >= k - 1) res.push(nums[dq.front()]);
  }
  return res;
}

console.log(maxSlidingWindow([1, 3, -1, -3, 5, 3, 6, 7], 3));
// => [ 3, 3, 5, 5, 6, 7 ]
```

### 14.3 C++ / Java → JS translation

| C++ | Java | JS equivalent |
|---|---|---|
| `std::stack<T>` | `Deque<T>` used as stack (`push`/`pop`/`peek`) | plain `Array` (`push`/`pop`/`at(-1)`) |
| `std::queue<T>` | `Queue<T>` (typically `ArrayDeque`) | `ArrayQueue` class above (head-index), or `Deque` |
| `std::deque<T>` | `ArrayDeque<T>` | `Deque` class above |
| `q.front()` / `q.back()` | `peek()` / `peekLast()` | `.front()` / `.back()` |
| `q.push()` / `q.pop()` (queue) | `.offer()` / `.poll()` | `.push()` (enqueue) / `.shift()` (dequeue) |

---

## 15. PriorityQueue / Binary Heap (JS has none — implement it)

**JavaScript has NO built-in heap or priority queue.** Unlike C++'s `std::priority_queue` and Java's `java.util.PriorityQueue`, there is nothing in the language or standard library — you must bring your own binary heap. This is the single most important piece of infrastructure code for JS-based competitive programming, because it appears in Dijkstra, k-way merge, top-K, task scheduling, and dozens of other patterns.

### 15.1 Complete, generic, ready-to-paste binary heap

Array-backed binary heap with a comparator, so it works as MinHeap, MaxHeap, or any custom order (e.g. Dijkstra's `[dist, node]` pairs).

```js
class PriorityQueue {
  // comparator(a, b) < 0  => a has higher priority (comes out first)
  // Default: MIN-heap of numbers (a - b)
  constructor(comparator = (a, b) => a - b) {
    this._heap = [];
    this._cmp = comparator;
  }

  size() { return this._heap.length; }
  isEmpty() { return this._heap.length === 0; }
  peek() { return this._heap[0]; }        // O(1) — top() equivalent

  push(val) {                              // offer(), O(log n)
    this._heap.push(val);
    this._siftUp(this._heap.length - 1);
    return this.size();
  }

  pop() {                                  // poll(), O(log n)
    if (this._heap.length === 0) return undefined;
    const top = this._heap[0];
    const last = this._heap.pop();
    if (this._heap.length > 0) {
      this._heap[0] = last;
      this._siftDown(0);
    }
    return top;
  }

  // O(n) heapify — build from an existing array instead of n pushes (n log n)
  static heapify(arr, comparator = (a, b) => a - b) {
    const pq = new PriorityQueue(comparator);
    pq._heap = arr.slice();
    for (let i = Math.floor(pq._heap.length / 2) - 1; i >= 0; i--) {
      pq._siftDown(i);
    }
    return pq;
  }

  _parent(i) { return (i - 1) >> 1; }
  _left(i)   { return 2 * i + 1; }
  _right(i)  { return 2 * i + 2; }

  _siftUp(i) {
    // Bubble the new last element up while it beats its parent.
    while (i > 0) {
      const p = this._parent(i);
      if (this._cmp(this._heap[i], this._heap[p]) < 0) {
        [this._heap[i], this._heap[p]] = [this._heap[p], this._heap[i]];
        i = p;
      } else break; // heap property restored
    }
  }

  _siftDown(i) {
    // Push the root down, always swapping with the higher-priority child.
    const n = this._heap.length;
    while (true) {
      const l = this._left(i);
      const r = this._right(i);
      let best = i;
      if (l < n && this._cmp(this._heap[l], this._heap[best]) < 0) best = l;
      if (r < n && this._cmp(this._heap[r], this._heap[best]) < 0) best = r;
      if (best === i) break; // both children satisfy heap property
      [this._heap[i], this._heap[best]] = [this._heap[best], this._heap[i]];
      i = best;
    }
  }

  toArray() { return this._heap.slice(); } // NOT sorted, just heap-order snapshot
}
```

**Walkthrough of the sift logic:**
- `_siftUp(i)`: a freshly pushed element sits at the last array slot. While it "beats" its parent under the comparator (i.e. `cmp(child, parent) < 0`), swap them and move the index up. Stops as soon as the parent is no worse — that's the invariant restored.
- `_siftDown(i)`: after popping the root and moving the last element there, repeatedly compare the node against BOTH children, pick whichever of {self, left, right} has highest priority, and swap down into that slot. Stops when the node already beats both children.

### 15.2 MinHeap / MaxHeap of numbers

```js
const minHeap = new PriorityQueue((a, b) => a - b); // default anyway
[5, 3, 8, 1, 9, 2].forEach(x => minHeap.push(x));
console.log(minHeap.peek()); // => 1
const sortedAsc = [];
while (!minHeap.isEmpty()) sortedAsc.push(minHeap.pop());
console.log(sortedAsc); // => [ 1, 2, 3, 5, 8, 9 ]

// MaxHeap: just invert the comparator
const maxHeap = new PriorityQueue((a, b) => b - a);
[5, 3, 8, 1, 9, 2].forEach(x => maxHeap.push(x));
console.log(maxHeap.peek()); // => 9
const sortedDesc = [];
while (!maxHeap.isEmpty()) sortedDesc.push(maxHeap.pop());
console.log(sortedDesc); // => [ 9, 8, 5, 3, 2, 1 ]
```

### 15.3 Custom orders — heap of pairs / objects

```js
// Heap of [priority, item] pairs, ordered by priority (min-heap)
const taskPQ = new PriorityQueue((a, b) => a[0] - b[0]);
taskPQ.push([3, 'low urgency']);
taskPQ.push([1, 'urgent']);
taskPQ.push([2, 'medium']);
console.log(taskPQ.pop()); // => [ 1, 'urgent' ]
console.log(taskPQ.pop()); // => [ 2, 'medium' ]

// Heap of objects, min by .dist, tie-break by .node ascending
const pq = new PriorityQueue((a, b) => a.dist - b.dist || a.node - b.node);
pq.push({ dist: 5, node: 2 });
pq.push({ dist: 5, node: 1 });
pq.push({ dist: 2, node: 3 });
console.log(pq.pop()); // => { dist: 2, node: 3 }
console.log(pq.pop()); // => { dist: 5, node: 1 }  (tie broken by node)
```

> **Gotcha:** the comparator MUST return a number (negative/zero/positive), not a boolean. `(a, b) => a < b` looks plausible but breaks the sift logic silently (it coerces `true`/`false` to `1`/`0`, losing the "equal" and "greater" distinction) — always subtract, or use an explicit `if/else if/else` returning `-1/0/1`.

### 15.4 Building a heap from an array in O(n)

```js
const arr = [9, 4, 7, 1, -2, 6, 5];
const heap = PriorityQueue.heapify(arr); // O(n), not O(n log n) from n pushes
console.log(heap.peek()); // => -2
```

> `heapify` sifts down from the last internal node to the root, which is provably O(n) total (not O(n log n)) — useful when you already have all elements up front (e.g. heap-sort, or seeding Dijkstra with all-infinity except source).

### 15.5 Dijkstra with the heap

```js
function dijkstra(n, adj, src) {
  // adj[u] = [[v, weight], ...]
  const dist = new Array(n).fill(Infinity);
  dist[src] = 0;
  const pq = new PriorityQueue((a, b) => a[0] - b[0]); // [distance, node]
  pq.push([0, src]);

  while (!pq.isEmpty()) {
    const [d, u] = pq.pop();
    if (d > dist[u]) continue; // stale entry — lazy deletion, see 15.6
    for (const [v, w] of adj[u]) {
      const nd = d + w;
      if (nd < dist[v]) {
        dist[v] = nd;
        pq.push([nd, v]);
      }
    }
  }
  return dist;
}

const adj = [
  [[1, 4], [2, 1]],
  [[3, 1]],
  [[1, 2], [3, 5]],
  [],
];
console.log(dijkstra(4, adj, 0)); // => [ 0, 3, 1, 4 ]
```

### 15.6 Lazy deletion (no decrease-key)

Like C++ `priority_queue` and Java `PriorityQueue`, this heap has **no `decreaseKey`** operation — you cannot efficiently update an element's priority in place. The standard workaround (used above in Dijkstra) is **lazy deletion**: push a new, better entry without removing the old stale one, then when popping, check if the popped entry is stale (`d > dist[u]`) and simply `continue`/skip it. This costs a bit of extra heap size (`O(E)` entries instead of `O(V)`) but is simpler and just as asymptotically efficient as maintaining a true indexed heap.

### 15.7 Complexity table

| Operation | Complexity |
|---|---|
| `push` / `offer` | O(log n) |
| `pop` / `poll` | O(log n) |
| `peek` / `top` | O(1) |
| `size` / `isEmpty` | O(1) |
| `PriorityQueue.heapify(arr)` (build from array) | O(n) |
| n sequential `push`es (no heapify) | O(n log n) |

> **vs C++:** `priority_queue<int> pq;` is max-heap by default; `priority_queue<int, vector<int>, greater<int>> pq;` for min-heap. Java's `new PriorityQueue<>()` is min-heap by default; pass `Collections.reverseOrder()` for max-heap. JS's `PriorityQueue` above defaults to min-heap via `(a,b)=>a-b`; flip the comparator for max-heap — there is no separate class, just a different comparator argument.

> **Note on npm libraries:** packages like `js-priority-queue`, `heap-js`, `mnemonist`'s Heap, or `@datastructures-js/priority-queue` exist and are well-tested, but most online judges (LeetCode, Codeforces, HackerRank, etc.) run your code **without npm install access** — only built-in Node/JS is available. Hand-rolling the `PriorityQueue` class above (or memorizing it) is the norm for JS competitive programming, exactly as C++ programmers occasionally hand-roll structures STL doesn't provide.

There is also no built-in balanced BST / ordered map / multiset with O(log n) floor/ceiling in JS. See §16 for the workarounds — this is a real, structural gap you plan around rather than paper over.

---

## 16. Choosing the Right Structure (and the Missing-TreeMap Problem)

### 16.1 Decision table: requirement → JS structure

| Requirement | JS structure / technique |
|---|---|
| LIFO access | `Array` — `push`/`pop` |
| FIFO access, bounded run (BFS) | head-index `ArrayQueue` (§14.2) |
| Access/insert/remove both ends | `Deque` class (§14.2) |
| Unique membership test, O(1) | `Set` |
| Key → value, any key type | `Map` |
| Key → value, fixed string-keyed shape (JSON-like) | plain object `{}` |
| Frequency count / multiset by count | `Map<value, count>` |
| Extract-min/max repeatedly | hand-rolled `PriorityQueue` (§15) |
| Sorted order, static data, need rank/binary search | sort once + binary search (see Part IV) |
| Sorted order, need dynamic insert, small n | sorted array + `splice`/`Array#with` insert |
| Sorted order, need dynamic insert, large n, need floor/ceiling/rank | Fenwick tree (BIT) or segment tree over coordinate-compressed values |
| Coordinate/tuple key | string-encode (`` `${r},${c}` ``) or pack into a single integer key |
| Weak, GC-friendly object metadata (rare in DSA) | `WeakMap` / `WeakSet` |

### 16.2 The missing TreeMap/TreeSet/multiset problem

C++ gives you `std::map`/`std::set` (red-black tree, O(log n) insert/erase/find, **and** `lower_bound`/`upper_bound`/ordered iteration) and `std::multiset` (duplicates + same ordered ops). Java gives you `TreeMap`/`TreeSet` with `floorKey`/`ceilingKey`/`higherKey`/`lowerKey`. **JavaScript has none of this.** There is no built-in balanced BST, no ordered map, no ordered set, no multiset, and no O(log n) floor/ceiling anywhere in the standard library. This is the single biggest structural gap versus C++/Java for competitive programming, and every JS-CP solver needs a repertoire of workarounds:

**(a) Sort once + binary search — when the key set is static**
If you know all values up front and never insert afterward, sort them once (O(n log n)) and binary-search for lower/upper bound on each query (O(log n) per query). Cheapest option when it applies. (Full binary-search recipes — including hand-rolled `lowerBound`/`upperBound` since JS has no `std::lower_bound` either — are covered in Part IV.)

**(b) Sorted array + splice-insert — small/moderate n**
```js
function sortedInsert(arr, x) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {               // binary search insertion point
    const mid = (lo + hi) >> 1;
    if (arr[mid] < x) lo = mid + 1; else hi = mid;
  }
  arr.splice(lo, 0, x);           // O(n) shift, but simple & fine for small n
  return lo;
}
const sa = [1, 3, 5, 9];
sortedInsert(sa, 4);
console.log(sa); // => [ 1, 3, 4, 5, 9 ]
```
`splice` insert is O(n) per call (array shift), so this is only appropriate when n stays small (roughly up to a few thousand elements per test) or the number of insertions is small — it is NOT a substitute for a real balanced BST at scale.

**(c) `Map`/object as a frequency multiset — when you don't need ordered navigation**
If you only need "how many copies of X are present" and never need "the smallest element ≥ X", a `Map<value, count>` (as in §12.3) is the right multiset substitute — O(1) add/remove/count, just no ordering.
```js
const multiset = new Map();
const add = (x) => multiset.set(x, (multiset.get(x) || 0) + 1);
const remove = (x) => {
  const c = multiset.get(x);
  if (!c) return;
  if (c === 1) multiset.delete(x); else multiset.set(x, c - 1);
};
add(5); add(5); add(3);
console.log(multiset); // => Map(2) { 5 => 2, 3 => 1 }
remove(5);
console.log(multiset.get(5)); // => 1
```

**(d) Fenwick tree (BIT) / segment tree — dynamic data, need order statistics**
When you need dynamic insert/delete AND O(log n) rank/floor/ceiling/kth-smallest queries (the true TreeMap use case), roll a Fenwick tree or segment tree over coordinate-compressed values. This is the real fix, not a workaround — cross-reference DSA Patterns document, Fenwick Tree / BIT and Segment Tree sections (patterns 27/28), for full implementations.

**(e) npm ordered-container libraries**
Packages like `js-sdsl` (which mirrors C++ STL containers including `OrderedMap`/`OrderedSet`) or `sorted-btree` exist and genuinely replicate TreeMap behavior — but exactly like the heap libraries in §15.7, **most online judges don't allow npm installs**, so treat these as unavailable unless you've confirmed the judge supports them. Know the hand-rolled fallback.

### 16.3 C++ / Java → JS container translation table

| C++ | Java | JS |
|---|---|---|
| `std::vector<T>` | `ArrayList<T>` | `Array` |
| `std::unordered_map<K,V>` | `HashMap<K,V>` | `Map` (or plain object for string keys) |
| `std::map<K,V>` / `std::multimap` | `TreeMap<K,V>` | sorted array + binary search, or Fenwick/segment tree (§16.2d) |
| `std::unordered_set<T>` | `HashSet<T>` | `Set` |
| `std::set<T>` | `TreeSet<T>` | sorted array (dedup on insert), or Fenwick/segment tree |
| `std::multiset<T>` | (`TreeMap<T,Integer>` counts, common Java idiom) | `Map<T, count>` (§16.2c), or sorted array with duplicates |
| `std::stack<T>` | `Deque<T>` as stack | `Array` (`push`/`pop`) |
| `std::queue<T>` | `Queue<T>` (`ArrayDeque`) | head-index `ArrayQueue` (§14.2) |
| `std::deque<T>` | `ArrayDeque<T>` | `Deque` class (§14.2) |
| `std::priority_queue<T>` | `PriorityQueue<T>` | hand-rolled `PriorityQueue`/heap (§15) |
| `std::bitset<N>` | `BitSet` | typed array (`Uint8Array`/`Uint32Array`) or `BigInt` bitmask |
| `std::pair<A,B>` | simple field class / `Map.Entry` | `[a, b]` array, or string key `` `${a},${b}` `` |
| `std::tuple<...>` | record / custom class | array `[a, b, c]` or plain object |
| `std::optional<T>` | `Optional<T>` | `undefined` / `null` sentinel |

### 16.4 Master complexity table — all structures, Parts II & III

| Structure | Access | Search | Insert | Delete | Notes |
|---|---|---|---|---|---|
| `Array` (as list) | O(1) index | O(n) `indexOf`/`includes` | O(1) amortized push, O(n) unshift/splice-middle | O(n) shift/splice-middle, O(1) pop | index access is the strength |
| `Array` (as stack) | O(1) top | — | O(1) amortized push | O(1) pop | |
| head-index `ArrayQueue` | O(1) front | — | O(1) amortized push | O(1) shift (no re-index) | never use raw `shift()` on an array |
| `Deque` (linked-list) | O(1) front/back | O(n) middle | O(1) both ends | O(1) both ends | needed for monotonic-deque problems |
| `Map` | O(1) avg get | O(1) avg has | O(1) avg set | O(1) avg delete | insertion-ordered, any key type |
| plain object `{}` | O(1) avg get | O(1) avg `in`/hasOwn | O(1) avg | O(1) avg (can de-opt) | string/symbol keys only |
| `Set` | — | O(1) avg has | O(1) avg add | O(1) avg delete | insertion-ordered, any value type |
| hand-rolled `PriorityQueue` (heap) | O(1) peek | O(n) arbitrary element | O(log n) push | O(log n) pop | no decrease-key — lazy delete |
| sorted array + binary search | O(1) index | O(log n) lookup/bound | O(n) splice insert | O(n) splice delete | static-data friendly |
| Fenwick tree / BIT | — | O(log n) prefix query | O(log n) point update | O(log n) point update | order statistics, range sum |
| Segment tree | — | O(log n) range query | O(log n) point/range update | O(log n) | most general range structure |

> **vs C++/Java:** the recurring theme across all of Part III — C++ STL and Java's `java.util` hand you `priority_queue`/`PriorityQueue`, `map`/`TreeMap`, `set`/`TreeSet`, `multiset`, and `deque`/`ArrayDeque` all pre-built with the right complexity guarantees. JS gives you exactly two built-in general-purpose containers beyond `Array`: `Map` and `Set` — both hash-based, neither ordered-tree-based. Everything else in this Part (`Deque`, `PriorityQueue`, the sorted-array/BIT workarounds for TreeMap/TreeSet/multiset) is code YOU bring. Internalize the `PriorityQueue` class in §15.1 and the `Deque` class in §14.2 well enough to type them from memory — they are the two pieces of infrastructure every non-trivial JS-CP session eventually needs.

---

# PART IV — SORTING, SEARCHING AND FUNCTIONAL TOOLS

## 17. Sorting

### 17.1 THE trap: default `sort()` is lexicographic string sort

`Array.prototype.sort()` with **no comparator** converts every element to a **string** and compares them **lexicographically** (UTF-16 code unit order). This is the single most common JS-DSA bug — it silently produces wrong answers on numeric arrays instead of crashing.

```js
const arr = [10, 2, 1, 20];
console.log(arr.sort());
// => [ 1, 10, 2, 20 ]   <-- WRONG for numeric ascending order!
// Why: "1" < "10" < "2" < "20" as strings (compares char by char)
```

```js
// More surprises with the default sort:
console.log([100, 25, 3].sort());        // => [ 100, 25, 3 ]  ("100" < "25" < "3")
console.log([-5, -1, -20].sort());       // => [ -1, -20, -5 ] (strings "-1","-20","-5")
console.log([5, 1, 4, 2, 3].sort());     // => [ 1, 2, 3, 4, 5 ] (looks right by luck: single digits)
```

> **Gotcha:** Single-digit test arrays will *accidentally* sort correctly with no comparator, which hides the bug until you feed in a two-digit number in an interview or contest and get baffled by wrong output. **Always** pass a comparator for numbers — no exceptions.

> **vs C++/Java:** `std::sort` on `vector<int>` uses `operator<` (numeric) by default. Java's `Arrays.sort(int[])` is numeric by default too (primitive arrays only — `Arrays.sort(Integer[])` without a comparator uses `compareTo`, which IS numeric for Integer). JS is the odd one out: **every** default sort is string-based, regardless of element type.

### 17.2 The fix — always pass a comparator

```js
const nums = [10, 2, 1, 20];
nums.sort((a, b) => a - b);              // ascending
console.log(nums);                       // => [ 1, 2, 10, 20 ]

nums.sort((a, b) => b - a);              // descending
console.log(nums);                       // => [ 20, 10, 2, 1 ]
```

### 17.3 The comparator contract

`compareFn(a, b)`:
| Return value | Meaning |
|---|---|
| `< 0` (negative) | `a` sorts before `b` |
| `0` | keep relative order (stable) |
| `> 0` (positive) | `b` sorts before `a` |

```js
// The contract in action — you don't need exactly -1/0/1, any signed number works
[3, 1, 2].sort((a, b) => a - b);   // => [1, 2, 3]
```

### 17.4 `sort()` mutates AND returns

Unlike some languages where sort is a free function returning a new container, JS's `Array.prototype.sort` **mutates the array in place** and **also returns the same reference** — convenient for chaining, but easy to trip on if you expected a copy.

```js
const a = [3, 1, 2];
const b = a.sort((x, y) => x - y);
console.log(a);          // => [ 1, 2, 3 ]  (mutated!)
console.log(b === a);    // => true          (same reference returned)
```

> **vs C++:** `std::sort(v.begin(), v.end())` also mutates in place (same behavior) — but C++'s `sort` returns `void`, not the container. Java's `Collections.sort(list)` / `Arrays.sort(arr)` return `void` too. JS returning the mutated array is unique and enables one-liners like `return arr.sort((a,b)=>a-b)[0];`.

### 17.5 Stability — guaranteed since ES2019

`Array.prototype.sort` has been **spec-guaranteed stable** since ES2019 (all evergreen Node/V8 versions comply). Equal elements retain their original relative order. Safe to rely on for multi-key / multi-pass sorting tricks.

```js
const people = [
  { name: 'Bob', age: 30 },
  { name: 'Amy', age: 25 },
  { name: 'Cid', age: 30 },
];
people.sort((a, b) => a.age - b.age);
console.log(people.map(p => p.name));
// => [ 'Amy', 'Bob', 'Cid' ]   (Bob before Cid preserved — both age 30)
```

> **vs C++:** `std::sort` is **NOT** guaranteed stable (`std::stable_sort` is the stable one, usually slower). `std::sort` on equal elements may reorder them. JS's `sort` is always stable — one less thing to worry about, and it's actually the *faster-by-default* choice here since you don't need a separate "stable" variant.

### 17.6 Does `a - b` overflow? (No — for normal ints. Careful with BigInt / huge magnitudes)

JS numbers are IEEE-754 doubles. Unlike **Java's `int` subtraction**, which can silently overflow when writing a comparator like `(a, b) -> a - b` on `int[]` (classic Java interview bug: `Integer.MIN_VALUE - 1` wraps), JS's `a - b` is safe for any values within the *safe integer range* (`±2^53 - 1`, see `Number.MAX_SAFE_INTEGER`), because doubles don't wrap on subtraction — they just lose precision far beyond that range.

```js
console.log(Number.MAX_SAFE_INTEGER);       // => 9007199254740991
console.log(-(2**53) - 1);                  // => -9007199254740993  (no wraparound, just a normal number)
// NOTE: write -(2**53), never -2**53 — unary minus directly before ** is a SyntaxError in JS.
```

> **vs Java:** `(a, b) -> a - b` on `Integer[]` is a **real, famous bug** in Java when `a` and `b` are near `Integer.MIN_VALUE`/`MAX_VALUE` — always use `Integer.compare(a, b)` there. JS has no equivalent overflow trap for ordinary numbers.

**But** for `BigInt` values, or if you ever compare values you *know* exceed `2^53`, subtraction either doesn't type-check (mixing BigInt and Number throws) or loses precision — use explicit comparison instead:

```js
// BigInt comparator — subtraction of BigInts doesn't return a plain comparator-safe number reliably
// and mixing types throws, so compare explicitly:
const bigs = [10n, 2n, 100n, 1n];
bigs.sort((a, b) => (a < b ? -1 : a > b ? 1 : 0));
console.log(bigs);   // => [ 1n, 2n, 10n, 100n ]

// Also safe for regular numbers beyond safe-integer range, or just as a defensive default:
const safeCompare = (a, b) => (a < b ? -1 : a > b ? 1 : 0);
```

> **Gotcha:** `arr.sort((a, b) => a - b)` on a `BigInt[]` throws `TypeError: Cannot mix BigInt and other types, use explicit conversions` — `-` is not defined between BigInt and Number, and even BigInt-BigInt subtraction returns a BigInt, which `sort` doesn't want truncated into a comparator sign in a fragile way. Always use the three-way explicit comparator for BigInt.

### 17.7 Sorting objects / pairs — multi-key `||` idiom

```js
const pairs = [[1, 5], [1, 2], [0, 9], [0, 1]];
pairs.sort((a, b) => a[0] - b[0] || a[1] - b[1]);
console.log(pairs);
// => [ [ 0, 1 ], [ 0, 9 ], [ 1, 2 ], [ 1, 5 ] ]
// Why `||` works: a[0]-b[0] is 0 (falsy) only when first keys tie,
// so JS falls through to the second comparator only on ties.
```

```js
// Three keys
const rows = [
  { dept: 'A', age: 30, name: 'Z' },
  { dept: 'A', age: 30, name: 'B' },
  { dept: 'A', age: 20, name: 'M' },
];
rows.sort((a, b) => a.dept.localeCompare(b.dept) || a.age - b.age || a.name.localeCompare(b.name));
console.log(rows.map(r => `${r.dept}-${r.age}-${r.name}`));
// => [ 'A-20-M', 'A-30-B', 'A-30-Z' ]
```

### 17.8 Sorting strings

Default (no comparator) lexicographic sort works correctly for plain string arrays because string comparison isn't the numeric trap:

```js
console.log(['banana', 'Apple', 'cherry'].sort());
// => [ 'Apple', 'banana', 'cherry' ]   (uppercase sorts before lowercase — UTF-16 code unit order)

console.log(['banana', 'Apple', 'cherry'].sort((a, b) => a.localeCompare(b)));
// => [ 'Apple', 'banana', 'cherry' ]   (locale-aware; same here, but handles accents/case correctly for real text)
```

> **Gotcha:** Default string sort is **case-sensitive** and based on UTF-16 code unit values (`'A'` = 65 < `'a'` = 97), not alphabetical/locale order. Use `.localeCompare()` for human-correct ordering, plain `<`/`sort()` for pure ASCII/competitive-programming string comparisons (faster, and matches C++ `std::string` `<`).

### 17.9 Sorting indices by value (argsort pattern)

```js
const vals = [40, 10, 30, 20];
const idx = [...vals.keys()];              // => [0, 1, 2, 3]
idx.sort((i, j) => vals[i] - vals[j]);
console.log(idx);                          // => [ 1, 3, 2, 0 ]   (indices that would sort vals ascending)
console.log(idx.map(i => vals[i]));        // => [ 10, 20, 30, 40 ]
```

### 17.10 Descending order — two ways

```js
const a1 = [3, 1, 4, 1, 5].sort((a, b) => b - a);
console.log(a1);                            // => [ 5, 4, 3, 1, 1 ]

const a2 = [3, 1, 4, 1, 5].sort((a, b) => a - b).reverse();
console.log(a2);                            // => [ 5, 4, 3, 1, 1 ]  (same result, extra pass — prefer b-a)
```

> **Gotcha:** `.sort((a,b)=>a-b).reverse()` is NOT guaranteed to preserve original relative order among equal elements the same way `(a,b)=>b-a` does — reversing a stable ascending sort actually **reverses** the order of ties too, which flips stability into "last original occurrence first". If tie order matters, sort directly with the descending comparator, don't sort-then-reverse.

### 17.11 TypedArray.sort — numeric by default (contrast!)

`TypedArray.prototype.sort()` (e.g. `Int32Array`, `Float64Array`) sorts **numerically** by default — the opposite default from regular `Array`, because typed arrays can only hold numbers so string-coercion sorting wouldn't make sense.

```js
const ta = new Int32Array([10, 2, 1, 20]);
ta.sort();
console.log(ta);   // => Int32Array(4) [ 1, 2, 10, 20 ]   (numeric by default — no comparator needed!)
```

> **Gotcha:** Don't let this lull you — `new Array(...)` / `[]` literal arrays still need the comparator. Only real `TypedArray`s (rare in everyday DSA code, but common for perf-critical numeric buffers) get numeric-by-default sort.

### 17.12 `toSorted()` — non-mutating sort [ES2023]

```js
const original = [3, 1, 2];
const sorted = original.toSorted((a, b) => a - b);
console.log(original);  // => [ 3, 1, 2 ]   (untouched)
console.log(sorted);    // => [ 1, 2, 3 ]   (new array)
```

> **Availability:** `toSorted` requires Node 20+ (V8 with ES2023 array methods) — **not guaranteed on Node 18**. Check your judge/runtime. Safe portable fallback: `[...arr].sort(cmp)` (spread into a copy first, then sort in place).

```js
// Portable non-mutating sort (works on Node 18 and every judge):
const copy = [...original].sort((a, b) => a - b);
```

### 17.13 Sorting quick-reference table

| Goal | Snippet |
|---|---|
| Ascending numbers | `arr.sort((a,b) => a - b)` |
| Descending numbers | `arr.sort((a,b) => b - a)` |
| Ascending strings (locale) | `arr.sort((a,b) => a.localeCompare(b))` |
| Ascending strings (ASCII/fast) | `arr.sort()` (only if all elements are strings) |
| Sort by object key | `arr.sort((a,b) => a.key - b.key)` |
| Multi-key sort | `arr.sort((a,b) => a.k1-b.k1 \|\| a.k2-b.k2)` |
| Argsort (indices by value) | `[...vals.keys()].sort((i,j)=>vals[i]-vals[j])` |
| Non-mutating sort (portable) | `[...arr].sort(cmp)` |
| Non-mutating sort (ES2023) | `arr.toSorted(cmp)` |
| BigInt-safe compare | `(a,b) => (a<b?-1:a>b?1:0)` |

---

## 18. Binary Search (no built-in — roll your own)

JS ships **no** binary search, `lower_bound`, or `upper_bound` in the standard library. This is a real gap versus C++ (`<algorithm>`: `std::binary_search`, `std::lower_bound`, `std::upper_bound`) and even Java (`Arrays.binarySearch`, `Collections.binarySearch`). In JS-for-DSA you must **hand-roll** these every time — memorize the templates below.

> **vs C++/Java:** No standard equivalent exists. `Array.prototype.indexOf` is linear O(n), not binary search, and does not help. Write your own — always `[lo, hi)` half-open convention below avoids off-by-one bugs.

### 18.1 `lowerBound` — first index with `arr[i] >= x`

```js
function lowerBound(arr, x) {
  let lo = 0, hi = arr.length;           // [lo, hi) half-open
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);   // overflow-safe within 32-bit range
    if (arr[mid] < x) lo = mid + 1;
    else hi = mid;
  }
  return lo;                              // insertion point; arr.length if none
}

const a = [1, 3, 3, 5, 7];
console.log(lowerBound(a, 3));   // => 1  (first index where a[i] >= 3)
console.log(lowerBound(a, 4));   // => 3  (insertion point for 4)
console.log(lowerBound(a, 8));   // => 5  (== arr.length, not found, insert at end)
```

### 18.2 `upperBound` — first index with `arr[i] > x`

```js
function upperBound(arr, x) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (arr[mid] <= x) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

console.log(upperBound(a, 3));   // => 3  (first index where a[i] > 3)
console.log(upperBound(a, 0));   // => 0
```

### 18.3 `binarySearch` — returns index or -1

```js
function binarySearch(arr, x) {
  let lo = 0, hi = arr.length - 1;       // [lo, hi] closed interval here
  while (lo <= hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (arr[mid] === x) return mid;
    if (arr[mid] < x) lo = mid + 1;
    else hi = mid - 1;
  }
  return -1;
}

console.log(binarySearch(a, 5));   // => 3
console.log(binarySearch(a, 4));   // => -1
```

### 18.4 The `>>` overflow caveat — a REAL JS trap

`(lo + hi) >> 1` is the classic C/Java "overflow-safe midpoint" idiom (avoids `(lo+hi)/2` overflowing 32-bit `int`). In JS:

- `>>` (bitwise right shift) coerces operands to **32-bit signed integers** first. It works fine for normal array indices (`< 2^31 ≈ 2.1 billion`), which covers essentially every array length.
- **But** if `lo + hi` somehow exceeds `2^31 - 1` (e.g., you're binary-searching over a numeric range/answer-space up to `1e15`, common in "binary search on the answer" problems), `>>` silently truncates/wraps to a 32-bit value and gives a **garbage midpoint**.

```js
console.log((2_000_000_000 + 2_000_000_000) >> 1);
// => -147483648   WRONG! (4e9 doesn't fit in int32, wraps negative before shifting)

console.log(Math.floor((2_000_000_000 + 2_000_000_000) / 2));
// => 2000000000   CORRECT
```

> **Gotcha:** Since JS numbers are doubles (no native 32-bit int overflow like Java's `int`), the *actual* safe idiom in JS is just `Math.floor((lo + hi) / 2)` or `lo + Math.floor((hi - lo) / 2)` (also avoids `lo+hi` ever overflowing double range in the first place). Reserve `>> 1` for array-index binary search where values are guaranteed small (< 2^31); switch to `Math.floor` for "binary search on the answer" over large numeric ranges.

```js
// Safe midpoint for ANY range, including huge "search the answer" ranges:
function mid(lo, hi) { return lo + Math.floor((hi - lo) / 2); }
```

### 18.5 Binary search on the answer (predicate form)

Two common half-open templates — pick one and stick to it.

```js
// Template A: find the SMALLEST x in [lo, hi) for which predicate(x) is true
// (predicate must be monotonic: false...false, true...true)
function searchSmallestTrue(lo, hi, predicate) {
  while (lo < hi) {
    const m = lo + Math.floor((hi - lo) / 2);
    if (predicate(m)) hi = m;       // m works, try smaller
    else lo = m + 1;                // m doesn't work, go bigger
  }
  return lo;                        // first x where predicate(x) is true
}

// Example: smallest x such that x*x >= 50
console.log(searchSmallestTrue(0, 100, x => x * x >= 50));  // => 8  (8*8=64>=50, 7*7=49<50)
```

```js
// Template B: find the LARGEST x in [lo, hi] for which predicate(x) is true
function searchLargestTrue(lo, hi, predicate) {
  let ans = lo - 1;                 // sentinel: "none found"
  while (lo <= hi) {
    const m = lo + Math.floor((hi - lo) / 2);
    if (predicate(m)) { ans = m; lo = m + 1; }  // record, try bigger
    else hi = m - 1;
  }
  return ans;
}

// Example: largest x such that x*x <= 50
console.log(searchLargestTrue(0, 100, x => x * x <= 50));   // => 7
```

### 18.6 Counting elements in `[l, r]` via `upperBound - lowerBound`

```js
function countInRange(sortedArr, l, r) {
  return upperBound(sortedArr, r) - lowerBound(sortedArr, l);
}

const b = [1, 2, 2, 2, 5, 7, 9];
console.log(countInRange(b, 2, 7));   // => 5  (elements 2,2,2,5,7)
```

### 18.7 Rotated sorted array (brief — cross-ref DSA Pattern 05)

Same `[lo, hi]` binary search, but at each step determine which half is sorted and check if target lies within it:

```js
function searchRotated(arr, target) {
  let lo = 0, hi = arr.length - 1;
  while (lo <= hi) {
    const m = lo + Math.floor((hi - lo) / 2);
    if (arr[m] === target) return m;
    if (arr[lo] <= arr[m]) {               // left half is sorted
      if (arr[lo] <= target && target < arr[m]) hi = m - 1;
      else lo = m + 1;
    } else {                                // right half is sorted
      if (arr[m] < target && target <= arr[hi]) lo = m + 1;
      else hi = m - 1;
    }
  }
  return -1;
}
console.log(searchRotated([4, 5, 6, 7, 0, 1, 2], 0));  // => 4
```

> See Part I / DSA Pattern 05 for the full rotated-array pattern family (find min, count of rotations, search with duplicates).

### 18.8 Sorted array as TreeSet/TreeMap replacement — floor/ceiling idioms

Cross-ref Part III §16 (no built-in ordered set in JS). When you maintain a **sorted array** manually (e.g., via insertion with `splice` for small n, or rebuilding), `lowerBound`/`upperBound` give you `floor`/`ceiling`/`higher`/`lower` semantics:

```js
// ceiling(x): smallest element >= x  — same as lowerBound, guard out-of-range
function ceiling(sortedArr, x) {
  const i = lowerBound(sortedArr, x);
  return i < sortedArr.length ? sortedArr[i] : undefined;
}

// floor(x): largest element <= x  — one step back from upperBound
function floor(sortedArr, x) {
  const i = upperBound(sortedArr, x) - 1;
  return i >= 0 ? sortedArr[i] : undefined;
}

// higher(x): smallest element > x  — exactly upperBound
function higher(sortedArr, x) {
  const i = upperBound(sortedArr, x);
  return i < sortedArr.length ? sortedArr[i] : undefined;
}

// lower(x): largest element < x  — one step back from lowerBound
function lower(sortedArr, x) {
  const i = lowerBound(sortedArr, x) - 1;
  return i >= 0 ? sortedArr[i] : undefined;
}

const s = [1, 3, 5, 7, 9];
console.log(ceiling(s, 4));   // => 5
console.log(floor(s, 4));     // => 3
console.log(higher(s, 5));    // => 7
console.log(lower(s, 5));     // => 3
console.log(ceiling(s, 10));  // => undefined  (guard: nothing >= 10)
console.log(floor(s, 0));     // => undefined  (guard: nothing <= 0)
```

> **Gotcha:** Always guard the index against `-1` and `arr.length` — these are the exact off-by-one spots where floor/ceiling helpers silently return garbage (`undefined` vs. an out-of-bounds `arr[-1]` which JS just gives you as `undefined` anyway, masking bugs elsewhere if you don't check explicitly).

---

## 19. Higher-Order Array Methods for Algorithms

### 19.1 Complete method table

| Method | Purpose | Mutates? | Short-circuits? |
|---|---|---|---|
| `map(fn)` | transform each element → new array | no | no |
| `filter(fn)` | keep elements passing predicate → new array | no | no |
| `reduce(fn, init)` | fold left → single value | no | no |
| `reduceRight(fn, init)` | fold right → single value | no | no |
| `forEach(fn)` | side effects only, no return value | no | no (can't `break`) |
| `some(fn)` | true if ANY element passes (like `any_of`) | no | **yes** |
| `every(fn)` | true if ALL elements pass (like `all_of`) | no | **yes** |
| `find(fn)` | first element passing predicate, or `undefined` | no | yes |
| `findIndex(fn)` | index of first match, or `-1` | no | yes |
| `findLast(fn)` [ES2023] | last element passing predicate | no | yes (from end) |
| `findLastIndex(fn)` [ES2023] | index of last match, or `-1` | no | yes (from end) |
| `flat(depth)` | flatten nested arrays | no | n/a |
| `flatMap(fn)` | map then flatten one level | no | n/a |
| `fill(val, start, end)` | fill range with value | **yes** | n/a |
| `Array.from(obj, mapFn)` | build array from iterable/array-like, optional map | n/a (creates new) | n/a |
| `entries()` | iterator of `[index, value]` pairs | no | n/a |
| `keys()` | iterator of indices | no | n/a |
| `values()` | iterator of values | no | n/a |

### 19.2 `map` / `filter`

```js
console.log([1, 2, 3].map(x => x * x));          // => [ 1, 4, 9 ]
console.log([1, 2, 3, 4, 5].filter(x => x % 2 === 0));  // => [ 2, 4 ]
```

### 19.3 `reduce` / `reduceRight` — the initial value matters

```js
console.log([1, 2, 3, 4].reduce((sum, x) => sum + x, 0));   // => 10
console.log([1, 2, 3, 4].reduce((sum, x) => sum + x));      // => 10 (uses arr[0]=1 as init, still works here)
```

```js
// WITHOUT an initial value, reduce on an EMPTY array THROWS:
try {
  [].reduce((a, b) => a + b);
} catch (e) {
  console.log(e.message);   // => Reduce of empty array with no initial value
}

// WITH an initial value, empty array is safe:
console.log([].reduce((a, b) => a + b, 0));   // => 0
```

> **Gotcha:** Always pass an initial value to `reduce` in DSA code — it's the difference between a crash on an edge-case empty input and a clean `0`/`[]`/`{}` result. This is a very common judge-fails-on-edge-case bug.

```js
// Max via reduce (no spread stack-limit risk, see 19.6)
console.log([3, 7, 2, 9, 4].reduce((m, x) => Math.max(m, x), -Infinity));  // => 9

// Build a frequency object via reduce
const freq = ['a', 'b', 'a', 'c', 'b', 'a'].reduce((acc, ch) => {
  acc[ch] = (acc[ch] || 0) + 1;
  return acc;
}, {});
console.log(freq);   // => { a: 3, b: 2, c: 1 }

// reduceRight — fold from the right (rarely needed, but e.g. right-to-left string building)
console.log(['a', 'b', 'c'].reduceRight((acc, x) => acc + x, ''));   // => 'cba'
```

### 19.4 `forEach` — side effects only, cannot `break`

```js
let total = 0;
[1, 2, 3].forEach(x => { total += x; });
console.log(total);   // => 6
```

> **Gotcha:** `forEach` has **no way to break early** — `return` inside the callback just skips to the next iteration (like `continue`), it does NOT exit the loop. If you need early termination, use a plain `for`/`for...of` loop, or `some`/`every` (abusing their boolean short-circuit), never `forEach`.

```js
// This does NOT stop at 2 — it just "continues":
[1, 2, 3, 4].forEach(x => { if (x === 2) return; console.log(x); });
// => 1
// => 3
// => 4
```

### 19.5 `some` / `every` — short-circuit predicates (any_of / all_of)

```js
console.log([1, 3, 5, 7].some(x => x % 2 === 0));    // => false
console.log([1, 3, 4, 7].some(x => x % 2 === 0));    // => true
console.log([2, 4, 6].every(x => x % 2 === 0));      // => true
console.log([2, 4, 5].every(x => x % 2 === 0));      // => false
```

```js
// Both short-circuit — proof via side effect count
let calls = 0;
[1, 2, 3, 4, 5].some(x => { calls++; return x === 2; });
console.log(calls);   // => 2   (stopped as soon as match found, didn't scan the rest)
```

> **vs C++:** Direct analogs of `std::any_of` / `std::all_of` / `std::none_of` (for `none_of`, negate `every`/`some` appropriately: `!arr.some(pred)`).

### 19.6 `find` / `findIndex` / `findLast` / `findLastIndex`

```js
const arr = [5, 12, 8, 130, 44];
console.log(arr.find(x => x > 10));         // => 12
console.log(arr.findIndex(x => x > 10));    // => 1
console.log(arr.findLast(x => x > 10));     // => 44   [ES2023]
console.log(arr.findLastIndex(x => x > 10)); // => 4    [ES2023]
console.log(arr.find(x => x > 1000));       // => undefined (not -1!)
```

> **Availability:** `findLast`/`findLastIndex` need Node 18+ (present since V8 9.7 / Node 18) — safe on the Node 18+ baseline this reference targets, but verify on older judges (Codeforces custom test / some online judges pin older Node).

### 19.7 `flat` / `flatMap`

```js
console.log([1, [2, 3], [4, [5, 6]]].flat());        // => [ 1, 2, 3, 4, [ 5, 6 ] ]  (default depth 1)
console.log([1, [2, 3], [4, [5, 6]]].flat(2));       // => [ 1, 2, 3, 4, 5, 6 ]
console.log([1, [2, [3, [4]]]].flat(Infinity));      // => [ 1, 2, 3, 4 ]  (fully flatten, any depth)

console.log([1, 2, 3].flatMap(x => [x, x * 2]));
// => [ 1, 2, 2, 4, 3, 6 ]   (map then flatten one level — great for expanding each element into N)
```

### 19.8 `fill`

```js
const grid = new Array(5).fill(0);
console.log(grid);   // => [ 0, 0, 0, 0, 0 ]

const partial = [1, 2, 3, 4, 5];
partial.fill(9, 1, 3);         // fill from index 1 up to (not incl.) 3
console.log(partial);          // => [ 1, 9, 9, 4, 5 ]
```

> **Gotcha:** `fill` with an **object/array** default fills every slot with the **same reference** — a classic 2D-array-init bug (see Part I/II for the correct `Array.from({length:n}, () => new Array(m).fill(0))` pattern; `new Array(n).fill([])` gives `n` references to the SAME inner array).

### 19.9 `Array.from` with map function — the general array-builder

```js
console.log(Array.from({ length: 5 }, (_, i) => i * i));
// => [ 0, 1, 4, 9, 16 ]

console.log(Array.from('hello'));                 // => [ 'h', 'e', 'l', 'l', 'o' ]
console.log(Array.from({ length: 3 }, () => []));  // => [ [], [], [] ]  (3 DISTINCT arrays, unlike fill)
```

### 19.10 `entries` / `keys` / `values`

```js
for (const [i, v] of ['a', 'b', 'c'].entries()) {
  console.log(i, v);
}
// => 0 'a'
// => 1 'b'
// => 2 'c'

console.log([...['x', 'y', 'z'].keys()]);     // => [ 0, 1, 2 ]
console.log([...['x', 'y', 'z'].values()]);   // => [ 'x', 'y', 'z' ]
```

### 19.11 Chaining — readability vs performance tradeoff

```js
const result = [1, 2, 3, 4, 5, 6]
  .filter(x => x % 2 === 0)
  .map(x => x * x)
  .reduce((s, x) => s + x, 0);
console.log(result);   // => 56   (4 + 16 + 36)
```

> **Gotcha:** Each chained `map`/`filter` call allocates a **full intermediate array** and does a **full pass** over the data. `arr.filter(...).map(...).reduce(...)` is 3 passes + 2 allocations. In hot loops in competitive programming (large n, tight TLE), a single plain `for` loop doing the equivalent work in ONE pass with no allocation is measurably faster. Prefer chaining for clarity in normal backend code; drop to a manual `for` loop when n is large (≥ 1e6) or the loop runs inside another loop (O(n²) territory where the constant factor bites).

```js
// Equivalent single-pass version — faster for large n:
let sum = 0;
for (const x of [1, 2, 3, 4, 5, 6]) {
  if (x % 2 === 0) sum += x * x;
}
console.log(sum);   // => 56
```

### 19.12 Common DSA patterns

```js
// Sum
console.log([1, 2, 3, 4].reduce((s, x) => s + x, 0));   // => 10

// Max (reduce — safe for huge arrays, see 21.1 for the spread caveat)
console.log([3, 7, 2, 9].reduce((m, x) => Math.max(m, x), -Infinity));   // => 9

// Frequency map (object)
const f = 'banana'.split('').reduce((acc, c) => (acc[c] = (acc[c] || 0) + 1, acc), {});
console.log(f);   // => { b: 1, a: 3, n: 2 }

// Grouping (array of objects -> Map of arrays)
const items = [{ k: 'x', v: 1 }, { k: 'y', v: 2 }, { k: 'x', v: 3 }];
const groups = items.reduce((m, it) => {
  if (!m.has(it.k)) m.set(it.k, []);
  m.get(it.k).push(it.v);
  return m;
}, new Map());
console.log([...groups.entries()]);   // => [ [ 'x', [ 1, 3 ] ], [ 'y', [ 2 ] ] ]

// Prefix sum array
function prefixSum(arr) {
  const pre = new Array(arr.length + 1).fill(0);
  for (let i = 0; i < arr.length; i++) pre[i + 1] = pre[i] + arr[i];
  return pre;
}
console.log(prefixSum([1, 2, 3, 4]));   // => [ 0, 1, 3, 6, 10 ]
// pre[i] = sum of arr[0..i-1]; range sum(l, r] inclusive-of-l = pre[r+1] - pre[l]
```

---

## 20. Comparators and Multi-key Ordering

### 20.1 The complete comparator mental model

A comparator `(a, b) => number`:
- returns **negative** → `a` should come before `b`
- returns **zero** → order unchanged (stable)
- returns **positive** → `b` should come before `a`

This single function type is reused everywhere in JS: `Array.sort`, a hand-rolled heap's internal ordering, `Intl.Collator.compare`, etc. Learn it once.

### 20.2 Ascending / descending

```js
[5, 3, 8, 1].sort((a, b) => a - b);   // ascending  => [1, 3, 5, 8]
[5, 3, 8, 1].sort((a, b) => b - a);   // descending => [8, 5, 3, 1]
```

### 20.3 Multi-key with the `||` chain

The idiom: chain comparator terms with `||`. Each term is `0` (falsy) on a tie, so `||` falls through to the next key. Non-zero (truthy) short-circuits and that key decides the order.

```js
// 2-key: sort by age asc, then name asc
const people = [
  { age: 30, name: 'Zed' },
  { age: 25, name: 'Amy' },
  { age: 30, name: 'Bob' },
];
people.sort((a, b) => a.age - b.age || a.name.localeCompare(b.name));
console.log(people.map(p => `${p.age}-${p.name}`));
// => [ '25-Amy', '30-Bob', '30-Zed' ]
```

```js
// 3-key: sort by dept asc, then score desc, then name asc
const rows = [
  { dept: 'B', score: 90, name: 'M' },
  { dept: 'A', score: 70, name: 'Z' },
  { dept: 'A', score: 90, name: 'K' },
  { dept: 'A', score: 90, name: 'A' },
];
rows.sort((a, b) =>
  a.dept.localeCompare(b.dept) ||
  b.score - a.score ||
  a.name.localeCompare(b.name)
);
console.log(rows.map(r => `${r.dept}/${r.score}/${r.name}`));
// => [ 'A/90/A', 'A/90/K', 'A/70/Z', 'B/90/M' ]
```

> **Gotcha:** `||` falls through on any **falsy** result, not just exact `0` — but since a real comparator subtraction result of `0` IS the only falsy numeric outcome you'll produce here (negative and positive numbers are both truthy), this is safe **as long as every term is a numeric difference/three-way compare and never something like `NaN`** (which is also falsy and would silently fall through — guard against `NaN` from e.g. subtracting `undefined`).

### 20.4 Comparator for a hand-rolled heap / PriorityQueue

Cross-reference Part III (no built-in `PriorityQueue` in JS). The same `(a, b) => a - b` convention applies: when your binary-heap `siftUp`/`siftDown` compares `cmp(parent, child) > 0` to decide a swap, passing `(a,b)=>a-b` gives a **min-heap** (smaller values bubble to the root), and `(a,b)=>b-a` gives a **max-heap**.

```js
class MinHeap {
  constructor(cmp = (a, b) => a - b) { this.data = []; this.cmp = cmp; }
  push(val) {
    this.data.push(val);
    let i = this.data.length - 1;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (this.cmp(this.data[i], this.data[p]) < 0) {
        [this.data[i], this.data[p]] = [this.data[p], this.data[i]];
        i = p;
      } else break;
    }
  }
  pop() {
    const top = this.data[0];
    const last = this.data.pop();
    if (this.data.length) {
      this.data[0] = last;
      let i = 0;
      while (true) {
        let l = 2 * i + 1, r = 2 * i + 2, smallest = i;
        if (l < this.data.length && this.cmp(this.data[l], this.data[smallest]) < 0) smallest = l;
        if (r < this.data.length && this.cmp(this.data[r], this.data[smallest]) < 0) smallest = r;
        if (smallest === i) break;
        [this.data[i], this.data[smallest]] = [this.data[smallest], this.data[i]];
        i = smallest;
      }
    }
    return top;
  }
  get size() { return this.data.length; }
}

const h = new MinHeap();               // default (a,b)=>a-b => MIN-heap
[5, 1, 8, 2].forEach(x => h.push(x));
console.log(h.pop(), h.pop(), h.pop(), h.pop());   // => 1 2 5 8

const maxH = new MinHeap((a, b) => b - a);   // flip comparator => MAX-heap, same class
[5, 1, 8, 2].forEach(x => maxH.push(x));
console.log(maxH.pop(), maxH.pop());               // => 8 5
```

### 20.5 String comparison: `localeCompare` vs `<`/`>` on UTF-16

```js
console.log('a' < 'b');                 // => true   (UTF-16 code unit compare)
console.log('Z' < 'a');                 // => true   ('Z'=90 < 'a'=97 — NOT alphabetical!)
console.log('café'.localeCompare('cafe')); // => 1 (locale-aware, treats accents sensibly)
console.log('café' < 'cafe');              // => false  (raw code unit compare: 'é'=233 > 'e'=101, but string length/prefix logic differs — always prefer localeCompare for real text)
```

> **vs C++/Java:** `<`/`>` on JS strings is like C++'s `std::string operator<` (byte/code-unit-wise) or Java's raw `String.compareTo` (UTF-16 code unit-wise) — fast and deterministic, good for pure-ASCII competitive programming. `localeCompare` is like Java's `Collator` — use it only when correctness for human/Unicode text matters, since it's slower and locale-dependent (non-deterministic across environments, avoid in CP where judges compare exact output).

### 20.6 Stability leveraged for multi-pass sorts

Because `sort` is stable, you can sort by the **least significant key first**, then the **most significant key**, and the result is correctly multi-key ordered — an alternative to the `||` chain, useful when keys need different comparator logic (e.g., one numeric ascending, one custom):

```js
let data = [
  { grp: 'B', val: 2 },
  { grp: 'A', val: 3 },
  { grp: 'B', val: 1 },
  { grp: 'A', val: 1 },
];
data.sort((a, b) => a.val - b.val);              // pass 1: sort by val
data.sort((a, b) => a.grp.localeCompare(b.grp)); // pass 2: sort by grp (stable — val order preserved within each grp)
console.log(data.map(d => `${d.grp}${d.val}`));
// => [ 'A1', 'A3', 'B1', 'B2' ]
```

### 20.7 No overflow in `a - b` (doubles) — but explicit compare for BigInt

Same point as §17.6, worth repeating here because comparators are exactly where it bites: JS `a - b` on regular numbers never overflows/wraps the way Java's `int` subtraction can. For `BigInt` elements, use the explicit three-way form, since `-` either throws (mixed types) or needs conversion:

```js
const cmpBig = (a, b) => (a < b ? -1 : a > b ? 1 : 0);
console.log([30n, 10n, 20n].sort(cmpBig));   // => [ 10n, 20n, 30n ]
```

### 20.8 Reverse via comparator vs `.reverse()`

```js
const arr1 = [3, 1, 4, 1, 5].sort((a, b) => b - a);        // descending directly, stable
const arr2 = [3, 1, 4, 1, 5].sort((a, b) => a - b).reverse(); // ascending then flip — ties reorder!
console.log(arr1);   // => [ 5, 4, 3, 1, 1 ]
console.log(arr2);   // => [ 5, 4, 3, 1, 1 ]   (same values here, but tie order differs for objects — see 17.10)
```

### 20.9 Comparator quick-reference table

| Goal | Comparator |
|---|---|
| Ascending | `(a, b) => a - b` |
| Descending | `(a, b) => b - a` |
| By object key ascending | `(a, b) => a.key - b.key` |
| By object key descending | `(a, b) => b.key - a.key` |
| Multi-key (2) | `(a, b) => a.k1 - b.k1 \|\| a.k2 - b.k2` |
| Multi-key (3, mixed asc/desc) | `(a, b) => a.k1 - b.k1 \|\| b.k2 - a.k2 \|\| a.k3.localeCompare(b.k3)` |
| By string (fast/ASCII) | `(a, b) => (a < b ? -1 : a > b ? 1 : 0)` |
| By string (locale-correct) | `(a, b) => a.localeCompare(b)` |
| BigInt-safe | `(a, b) => (a < b ? -1 : a > b ? 1 : 0)` |
| Min-heap convention | `(a, b) => a - b` |
| Max-heap convention | `(a, b) => b - a` |

---

## 21. Math and Number Utilities

### 21.1 `Math` object — complete table

| Method / property | Purpose |
|---|---|
| `Math.max(...args)` | variadic max |
| `Math.min(...args)` | variadic min |
| `Math.abs(x)` | absolute value |
| `Math.floor(x)` | round toward −∞ |
| `Math.ceil(x)` | round toward +∞ |
| `Math.round(x)` | round to nearest (half rounds UP, incl. negatives toward +∞ at .5) |
| `Math.trunc(x)` | round toward 0 (drop fractional part) |
| `Math.sign(x)` | -1, 0, or 1 |
| `Math.pow(x, y)` / `x ** y` | exponentiation |
| `Math.sqrt(x)` | square root |
| `Math.cbrt(x)` | cube root |
| `Math.hypot(...args)` | sqrt(sum of squares), avoids intermediate overflow |
| `Math.log(x)` | natural log (ln) |
| `Math.log2(x)` | log base 2 |
| `Math.log10(x)` | log base 10 |
| `Math.LN2` | constant ln(2), for manual log-base-2 via `Math.log(x)/Math.LN2` |
| `Math.exp(x)` | e^x |
| `Math.random()` | pseudorandom float in `[0, 1)` |
| `Math.PI`, `Math.E` | constants |

### 21.2 `max` / `min` — variadic, spread caveat, empty-args trap

```js
console.log(Math.max(3, 7, 2, 9));      // => 9
console.log(Math.min(3, 7, 2, 9));      // => 2
console.log(Math.max(...[3, 7, 2, 9])); // => 9   (spread an array in)
```

```js
// TRAP 1: Math.max() with NO args
console.log(Math.max());   // => -Infinity   (identity element — makes sense but surprises)
console.log(Math.min());   // => Infinity
```

```js
// TRAP 2: spreading a HUGE array can blow the call stack
// Math.max(...arr) expands to a function call with `arr.length` arguments —
// each JS engine has a max argument count (V8: ~65536-125000 depending on version/context).
try {
  const huge = new Array(200000).fill(1);
  console.log(Math.max(...huge));
} catch (e) {
  console.log(e.constructor.name);   // => RangeError  (Maximum call stack size exceeded, or "too many arguments")
}

// FIX: use reduce for large arrays — no argument-count limit
const huge = new Array(200000).fill(1);
console.log(huge.reduce((m, x) => Math.max(m, x), -Infinity));   // => 1
```

> **Gotcha:** The safe threshold for `Math.max(...arr)` is roughly tens of thousands of elements, not millions — in competitive programming with n up to 1e5/1e6, **always** use `reduce` for max/min over a full array, never spread.

> **vs C++:** `std::max_element(v.begin(), v.end())` has no such limit (it's an O(n) loop under the hood, not variadic-call based) — `reduce` is the JS equivalent mental model here, not `Math.max(...)`.

### 21.3 `abs`, `floor`, `ceil`, `round`, `trunc`, `sign`

```js
console.log(Math.abs(-5));      // => 5
console.log(Math.floor(2.7));   // => 2
console.log(Math.floor(-2.7));  // => -3   (toward -Infinity)
console.log(Math.ceil(2.1));    // => 3
console.log(Math.ceil(-2.1));   // => -2   (toward +Infinity)
console.log(Math.round(2.5));   // => 3    (half rounds UP)
console.log(Math.round(-2.5));  // => -2   (NOT -3! rounds toward +Infinity at exactly .5 — see note below)
console.log(Math.trunc(2.7));   // => 2
console.log(Math.trunc(-2.7));  // => -2   (toward 0, NOT -3 — differs from Math.floor for negatives!)
console.log(Math.sign(-42));    // => -1
console.log(Math.sign(0));      // => 0
console.log(Math.sign(42));     // => 1
```

> **Gotcha (round-half-up, not banker's rounding):** `Math.round` uses "round half toward positive infinity" (`x + 0.5` then floor), NOT "round half to even" (banker's rounding, used by some other languages/libraries). This means `Math.round(-2.5)` is `-2`, not `-3`, which surprises people expecting symmetric rounding. If you need banker's rounding, implement it manually.

> **vs C++/Java:** C++'s integer division `-7 / 2` truncates toward 0 (`-3`), matching JS's `Math.trunc(-7/2)` = `-3`, **not** `Math.floor(-7/2)` = `-4`. Java's `/` on ints also truncates toward 0. **This is the #1 negative-integer-division gotcha when porting C++/Java code to JS**: JS has no native integer division operator at all — `/` always produces a float, so you must explicitly pick `Math.trunc` (matches C++/Java `/` semantics for negatives) vs `Math.floor` (mathematical floor division, different for negatives) — see 21.4.

### 21.4 Integer truncation idioms: `Math.floor` vs `x|0` vs `~~x` vs `Math.trunc` — and their differences

```js
console.log(Math.trunc(7 / 2));     // => 3   (toward 0)
console.log(Math.trunc(-7 / 2));    // => -3  (toward 0 — matches C++/Java int division)
console.log(Math.floor(-7 / 2));    // => -4  (toward -Infinity — mathematical floor division, DIFFERENT!)
console.log((-7 / 2) | 0);          // => -3  (bitwise OR with 0 truncates toward 0, same as Math.trunc — for small values)
console.log(~~(-7 / 2));            // => -3  (double bitwise NOT, same truncation trick, same limits)
```

| Idiom | Behavior | Works up to |
|---|---|---|
| `Math.trunc(x)` | truncate toward 0 | any safe integer / any double magnitude |
| `Math.floor(x)` | floor toward -Infinity | any safe integer / any double magnitude |
| `x \| 0` | truncate toward 0 via int32 coercion | **only correct up to ±2^31 − 1 (2,147,483,647)** |
| `~~x` | same as `\| 0`, double bitwise-not | **only correct up to ±2^31 − 1** |

```js
// The REAL trap: |0 and ~~ silently wrap/corrupt beyond 32-bit signed range
console.log(Math.trunc(3_000_000_000));   // => 3000000000  (correct)
console.log(3_000_000_000 | 0);           // => -1294967296  WRONG! (wraps as int32)
console.log(~~3_000_000_000);             // => -1294967296  WRONG! (same wraparound)
```

> **Gotcha:** `|0` and `~~x` are popular "fast truncate" micro-optimizations copied from older JS-perf folklore — they ARE marginally faster than `Math.trunc` in hot loops, but they **corrupt any value outside the 32-bit signed integer range** (roughly ±2.1 billion), which is easy to hit in DSA problems with n up to 1e9/1e18 sums, prefix sums, or products. Default to `Math.trunc`/`Math.floor`; only reach for `|0`/`~~` when you've proven the value is small and you're in a genuinely hot loop.

### 21.5 `pow`, `sqrt`, `cbrt`, `hypot`

```js
console.log(Math.pow(2, 10));    // => 1024
console.log(2 ** 10);            // => 1024   (exponent operator — same result, more idiomatic in modern JS)
console.log(2 ** 0.5);           // => 1.4142135623730951  (fractional exponents work, unlike `<<`/integer pow)
console.log(Math.sqrt(2));       // => 1.4142135623730951
console.log(Math.cbrt(27));      // => 3
console.log(Math.hypot(3, 4));   // => 5   (sqrt(3^2+4^2), avoids manual overflow/precision issues)
console.log(Math.hypot(3, 4, 12)); // => 13  (n-dimensional, sqrt(sum of squares))
```

> **vs C++:** `**` is JS-only syntax (not in C++/Java) — equivalent to `std::pow`/`Math.pow` in Java. Note `**` and `Math.pow` both return **floats** even for integer args (`2 ** 10` is `1024`, a double that happens to print as an integer) — for large integer powers where precision matters, consider `BigInt` exponentiation (`2n ** 10n`).

### 21.6 `log`, `log2`, `log10`, `LN2`, `exp`

```js
console.log(Math.log(Math.E));      // => 1               (natural log)
console.log(Math.log2(8));          // => 3
console.log(Math.log10(1000));      // => 3
console.log(Math.exp(1));           // => 2.718281828459045

// Manual log-base-2 (if you ever need it without Math.log2, or a custom base):
console.log(Math.log(8) / Math.LN2);   // => 3   (same as Math.log2(8))
function logBase(x, base) { return Math.log(x) / Math.log(base); }
console.log(logBase(81, 3));        // => 4
```

### 21.7 `Math.random()` — not seedable, CP note

```js
console.log(Math.random() >= 0 && Math.random() < 1);   // => true  (always in [0, 1))

// Random integer in [0, n)
function randInt(n) { return Math.floor(Math.random() * n); }
console.log(randInt(10) >= 0 && randInt(10) < 10);   // => true

// Random integer in [lo, hi] inclusive
function randRange(lo, hi) { return lo + Math.floor(Math.random() * (hi - lo + 1)); }
```

> **Gotcha:** `Math.random()` is **NOT seedable** in standard JS (unlike C++'s `std::mt19937 gen(seed)` or Java's `new Random(seed)`) — you cannot reproduce a deterministic sequence for testing/debugging with the built-in. For reproducible pseudo-randomness (e.g., stress-testing your solution against a brute force with the same seed), implement your own seeded PRNG (e.g., a small `mulberry32` or `xorshift` function) — a common CP/testing need that JS doesn't cover natively.

```js
// Minimal seedable PRNG (mulberry32) for reproducible tests:
function mulberry32(seed) {
  return function () {
    seed |= 0; seed = (seed + 0x6D2B79F5) | 0;
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
const rng = mulberry32(42);
console.log(rng() === mulberry32(42)());   // => true  (same seed => same first value, reproducible)
```

### 21.8 Integer square root — off-by-one near precision limits, with correction

```js
function isqrt(n) {
  let r = Math.floor(Math.sqrt(n));
  // Correct for floating-point precision drift near perfect squares / large n:
  while (r * r > n) r--;
  while ((r + 1) * (r + 1) <= n) r++;
  return r;
}
console.log(isqrt(50));                 // => 7
console.log(isqrt(49));                 // => 7   (exact perfect square, no drift here)
console.log(isqrt(4_503_599_627_370_496)); // => 67108864  (2^52 — near double precision edge, still correct with the guard)
```

> **Gotcha:** `Math.floor(Math.sqrt(n))` alone can be **off by one** for perfect squares (or values close to them) once `n` approaches `2^53` (`Number.MAX_SAFE_INTEGER` territory) because `Math.sqrt` itself is a floating-point operation subject to rounding error. The `while` correction loop above is cheap (runs 0-1 times almost always) and makes `isqrt` bulletproof — always include it in CP code rather than trusting the raw floor.

### 21.9 GCD / LCM — no built-in, provide helpers

```js
function gcd(a, b) {
  a = Math.abs(a); b = Math.abs(b);
  while (b) { [a, b] = [b, a % b]; }
  return a;
}
function lcm(a, b) {
  if (a === 0 || b === 0) return 0;
  return Math.abs(a / gcd(a, b) * b);   // divide first to reduce overflow risk
}

console.log(gcd(48, 18));   // => 6
console.log(lcm(4, 6));     // => 12
```

```js
// BigInt versions for values beyond safe-integer range:
function gcdBig(a, b) {
  a = a < 0n ? -a : a; b = b < 0n ? -b : b;
  while (b) { [a, b] = [b, a % b]; }
  return a;
}
console.log(gcdBig(48n, 18n));   // => 6n
```

> **vs C++/Java:** C++17 has `std::gcd`/`std::lcm` in `<numeric>`; Java has `BigInteger.gcd` (no primitive built-in `lcm`). JS has neither — always hand-roll, as above.

### 21.10 Fast modular exponentiation (with BigInt for large moduli)

```js
// Regular-number version (safe while base/mod stay well under 2^53, and intermediate mults don't overflow doubles' precision —
// risky above ~1e7 for mod, since (base*base) can exceed safe integer range; prefer BigInt for real CP mod problems)
function modPow(base, exp, mod) {
  base %= mod;
  let result = 1;
  while (exp > 0) {
    if (exp & 1) result = (result * base) % mod;
    base = (base * base) % mod;
    exp = Math.floor(exp / 2);
  }
  return result;
}
console.log(modPow(2, 10, 1000));   // => 24   (2^10 = 1024, 1024 % 1000 = 24)
```

```js
// BigInt version — the SAFE choice whenever mod or base can exceed ~1e7-1e9 (avoids precision loss in base*base)
function modPowBig(base, exp, mod) {
  base = BigInt(base) % BigInt(mod);
  exp = BigInt(exp);
  mod = BigInt(mod);
  let result = 1n;
  while (exp > 0n) {
    if (exp & 1n) result = (result * base) % mod;
    base = (base * base) % mod;
    exp >>= 1n;
  }
  return result;
}
console.log(modPowBig(2, 10, 1_000_000_007));       // => 1024n
console.log(modPowBig(123456789, 987654321, 1_000_000_007n)); // => (large BigInt result)
```

> **Gotcha:** In `modPow` (Number version), `base * base` can exceed `Number.MAX_SAFE_INTEGER` (2^53 ≈ 9e15) when `base` is close to a typical mod like `1e9+7`, silently losing precision in the multiplication BEFORE the `% mod` even runs. Rule of thumb: if `mod > ~3e7` (so that `mod*mod` could approach 1e15+ and risk precision loss with additional multiplications/sums layered on), switch to the **BigInt** version. This is a common source of "works on small tests, WA on large mod" bugs.

### 21.11 Safe modulo for negatives

```js
console.log(-7 % 3);              // => -1   (JS % is REMAINDER, keeps sign of dividend — not true modulo!)
console.log(((-7 % 3) + 3) % 3);  // => 2    (correct mathematical modulo, always non-negative for positive m)

function mod(a, m) { return ((a % m) + m) % m; }
console.log(mod(-7, 3));    // => 2
console.log(mod(7, 3));     // => 1
console.log(mod(-1, 5));    // => 4
```

> **vs C++/Java:** Same trap in C++ (`%` keeps sign of dividend, `-7 % 3 == -1`) and Java (`%` likewise). This is a language-family-wide gotcha, not JS-specific — but worth restating since it's a constant source of off-by-negative bugs in hashing (`hash = ((hash * p + c) % mod + mod) % mod`) and circular-index arithmetic (`(i - 1 + n) % n`).

### 21.12 `Number` methods: `toFixed`, `toString(radix)`, `parseInt`/`parseFloat`

```js
console.log((3.14159).toFixed(2));     // => '3.14'   (returns a STRING, not a number!)
console.log(Number((3.14159).toFixed(2))); // => 3.14  (convert back if you need a number)

console.log((255).toString(16));       // => 'ff'     (hex)
console.log((255).toString(2));        // => '11111111'  (binary)
console.log(parseInt('ff', 16));       // => 255      (parse hex string back to number)
console.log(parseInt('11111111', 2));  // => 255

console.log(parseInt('42px'));         // => 42       (parses leading numeric chars, ignores trailing junk)
console.log(parseFloat('3.14abc'));    // => 3.14
console.log(Number('42px'));           // => NaN      (Number() is strict — whole string must be numeric, unlike parseInt!)
console.log(Number('42'));             // => 42
```

> **Gotcha:** `toFixed` returns a **string**, a frequent source of accidental string concatenation bugs (`(1).toFixed(2) + (2).toFixed(2)` gives `'1.002.00'`, not `3.00`). Wrap in `Number(...)` or use `+` unary (`+x.toFixed(2)`) when you need it back as a number for further arithmetic.

> **vs C++/Java:** `parseInt`/`parseFloat` are lenient (parse a numeric prefix and ignore trailing garbage) — closer to C++'s `strtol`/`std::stoi` (which also stop at the first invalid char) than to Java's `Integer.parseInt` (which **throws** `NumberFormatException` on any trailing non-numeric character). `Number(str)` is the strict one in JS, closer to Java's `parseInt` in spirit (all-or-nothing), but returns `NaN` instead of throwing.

### 21.13 When to reach for BigInt

Cross-ref Part I. Reach for `BigInt` (`123n`) instead of `Number` when:
- Values can exceed `Number.MAX_SAFE_INTEGER` (`2^53 − 1 ≈ 9.007e15`) and you need **exact** integer results (factorials, large combinatorics, big modular arithmetic with large moduli, arbitrary-precision sums).
- You're doing repeated multiplication/exponentiation where intermediate products could exceed the safe integer range even if the final answer (after a mod) would fit — precision loss happens **before** the mod truncates, corrupting the result.

```js
console.log(2 ** 53 === 2 ** 53 + 1);   // => true   !! precision loss — silent wrong answer with plain Number
console.log(2n ** 53n === 2n ** 53n + 1n);  // => false  (BigInt is exact)
```

> **Gotcha:** BigInt is exact but slower and cannot mix with `Number` in arithmetic (`1n + 1` throws `TypeError`) — convert explicitly with `BigInt(x)` / `Number(x)` at the boundary. Don't default to BigInt everywhere; use it only when the value range genuinely demands it (factorials beyond ~18!, sums that can exceed 9e15, big-mod exponentiation as in 21.10).

### 21.14 Math utilities quick-reference table

| Need | Snippet |
|---|---|
| Truncate toward 0 (safe, any range) | `Math.trunc(x)` |
| Floor (toward -Infinity) | `Math.floor(x)` |
| Fast truncate (small values only, < 2^31) | `x \| 0` or `~~x` |
| Max of array (large n) | `arr.reduce((m,x)=>Math.max(m,x), -Infinity)` |
| Max of array (small n) | `Math.max(...arr)` |
| Integer sqrt (exact) | `isqrt(n)` helper (21.8) |
| GCD / LCM | `gcd(a,b)` / `lcm(a,b)` helpers (21.9) |
| Mod exponentiation (small mod) | `modPow(base, exp, mod)` (21.10) |
| Mod exponentiation (large mod) | `modPowBig(base, exp, mod)` with BigInt (21.10) |
| Safe modulo (non-negative) | `((a % m) + m) % m` |
| Random int in `[0, n)` | `Math.floor(Math.random() * n)` |
| Reproducible random | seeded PRNG (mulberry32, 21.7) — `Math.random()` is NOT seedable |
| Number → fixed-decimal string | `x.toFixed(k)` (returns string!) |
| String → int (lenient, stops at junk) | `parseInt(s, radix)` |
| String → number (strict, all-or-nothing) | `Number(s)` |
| Base conversion (number → string) | `x.toString(radix)` |
| Base conversion (string → number) | `parseInt(s, radix)` |

---

# PART V — BIT MANIPULATION

## 22. Bitwise Operators and the 32-bit Reality

**THE defining JS bit fact.** Every number in JS is a 64-bit IEEE-754 double. But every bitwise operator — `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>` — silently converts **both operands** to 32-bit integers before doing anything, then returns a 32-bit result. This conversion is called `ToInt32` (or `ToUint32` for `>>>`'s inputs/output). The double is truncated to an integer, reduced modulo 2³², then reinterpreted as two's-complement.

Consequences that bite constantly:

- Bit ops only ever see the **low 32 bits** of a number. Anything above `2^32 - 1` is silently discarded.
- The *result* of `&`, `|`, `^`, `~`, `<<`, `>>` is a **signed** 32-bit integer, range `[-2^31, 2^31 - 1]` = `[-2147483648, 2147483647]`.
- `>>>` is the one exception: its result is **unsigned**, range `[0, 2^32 - 1]`.

```js
1 << 31            // => -2147483648   (sign bit set — this is NEGATIVE, not 2147483648!)
1 << 32            // => 1             (shift count is taken mod 32, so 32 acts like 0)
1 << 33            // => 2             (mod 32 => shift by 1)

(2 ** 31) | 0       // => -2147483648   (2147483648 truncated/reinterpreted as signed 32-bit)
(2 ** 32) | 0       // => 0             (4294967296 mod 2^32 = 0)
(2 ** 32 + 5) | 0   // => 5             (only the low 32 bits survive)
```

Binary illustration of `1 << 31`:

```
1  = 0000...0001  (32 bits)
shift left by 31 →
     1000...0000
                     ^ this is the sign bit in two's complement
     interpreted as signed int32 = -2147483648
```

> **Gotcha:** Because bit ops truncate to 32 bits, you **cannot** use `&`, `|`, `^`, `<<`, `>>`, `>>>`, or `~` on numbers ≥ `2^31` (or ≤ `-2^31 - 1`) and expect correct results. Bitmask DP/subset tricks that fit fine in C++'s `long long` (64-bit) silently break in JS once `n > 30`. Use BigInt (§24) or an array-based structure instead.

> **vs C++:** In C++, `int` is (usually) 32-bit too, so overflow-by-shift is *also* a bug there (UB even), but you can just declare `long long`/`uint64_t` and the same operators keep working at 64 bits. In JS there is no wider *Number* bit-op — the operators themselves are hard-wired to 32 bits regardless of variable "type" (there are no types). Widening requires switching to an entirely different value kind: `BigInt`.

### The operator table

| Operator | Name | Operand conversion | Result type | Notes |
|---|---|---|---|---|
| `a & b` | AND | ToInt32 both | signed int32 | |
| `a \| b` | OR | ToInt32 both | signed int32 | |
| `a ^ b` | XOR | ToInt32 both | signed int32 | |
| `~a` | NOT | ToInt32 | signed int32 | `~a === -a - 1` |
| `a << b` | left shift | ToInt32(a), ToUint32(b) & 31 | signed int32 | shift count mod 32 |
| `a >> b` | arithmetic (signed) right shift | ToInt32(a), ToUint32(b) & 31 | signed int32 | sign-extends (fills with sign bit) |
| `a >>> b` | logical (unsigned) right shift | ToUint32(a), ToUint32(b) & 31 | **unsigned** int32 | fills with `0`; the ONLY unsigned bit op |

### `1 << 32 === 1` — the shift-count mask

The right-hand operand of any shift is converted with `ToUint32` and then **masked with `& 31`** (5 bits, since a 32-bit shift only ever needs a count in `[0, 31]`). This is why shifting by 32 does nothing (`32 & 31 === 0`) and shifting by 33 acts like shifting by 1.

```js
const n = 5;
console.log(1 << n);        // => 32
console.log(1 << (n + 32)); // => 32  (same! 37 & 31 === 5)
```

> **vs C++:** In C++, shifting by ≥ the bit-width of the type is **undefined behavior** — it might crash, might do this masking, might do something else depending on compiler/arch. JS *defines* the mod-32 (mod-64 for BigInt shifts use the actual bit count, no masking — see §24) behavior, so at least it's consistent, but it is easy to assume "shift by a huge number = huge number" and get bitten.

### `>>` vs `>>>` — signed vs unsigned right shift

```js
const x = -8;
console.log(x >> 1);    // => -4   (arithmetic: sign-extends, fills with 1s, floors toward -Infinity)
console.log(x >>> 1);   // => 2147483644  (logical: treats -8's bit pattern as unsigned 4294967288, then shifts)
```

Binary illustration (`x = -8`, i.e. `0xFFFFFFF8`):

```
-8 as int32:  1111 1111 1111 1111 1111 1111 1111 1000
x >> 1     :  1111 1111 1111 1111 1111 1111 1111 1100  = -4   (sign bit copied in)
x >>> 1    :  0111 1111 1111 1111 1111 1111 1111 1100  = 2147483644  (0 copied in)
```

> **Gotcha:** For non-negative numbers `>>` and `>>>` agree. They diverge only when the sign bit (bit 31) is set — i.e. for negative int32s, or for "logically unsigned" values ≥ 2^31 that you're storing in a plain number and want to reinterpret with `x >>> 0`.

### `x >>> 0`, `x | 0`, `~~x` — the int32-coercion idioms

| Idiom | Effect | Result range |
|---|---|---|
| `x \| 0` | ToInt32 | `[-2^31, 2^31 - 1]` |
| `~~x` | ToInt32 (double NOT) | `[-2^31, 2^31 - 1]` |
| `x >>> 0` | ToUint32 | `[0, 2^32 - 1]` |

```js
3.7 | 0        // => 3   (truncates toward zero, like a cast, NOT Math.floor for negatives)
-3.7 | 0       // => -3  (truncates toward zero — differs from Math.floor(-3.7) === -4)
~~4.9          // => 4
(-1) >>> 0     // => 4294967295   (reinterpret -1's bit pattern as unsigned — classic "uint32 view" trick)
```

> **Gotcha:** `x | 0` / `~~x` are NOT `Math.floor`. They truncate toward zero (like `Math.trunc`), and only work correctly in the int32 range. For large numbers or when you need real flooring, use `Math.floor`/`Math.trunc`/`Math.round` instead — `| 0` is for forcing an *already-integer-ish* value into 32-bit int form quickly (common in hot loops for micro-perf, but prefer clarity over cleverness unless profiling says otherwise).

### Precedence trap: `&`, `|`, `^` bind LOOSER than comparisons

This is the single most common real bug with JS bitwise code, and it is **the same trap as C++**, but JS programmers coming from a mostly-arithmetic mental model of JS get bitten more because they don't expect any "C-like" gotchas.

```js
const x = 5;

// WRONG — parses as: x & (1 === 0)  →  x & false  →  x & 0  →  0
if (x & 1 === 0) console.log("even?"); // never runs correctly

// RIGHT — parenthesize the bitwise expression
if ((x & 1) === 0) console.log("even");
else console.log("odd"); // => "odd"  (5 is odd, correctly detected)
```

Precedence order relevant here (loosest to tightest, partial list):

```
...  ||  &&  |  ^  &  ==/!=/===/!==  </>/<=/>=  <<//>>/>>>  +/-  *//  ~/unary
```

`&`/`^`/`|` are looser than `==`-family and relational operators. **Always parenthesize** a bitwise sub-expression that sits next to a comparison, ternary, or logical operator.

> **vs C++:** Identical precedence trap exists in C/C++ (`if (x & 1 == 0)` is the classic beginner bug there too). If this is muscle memory from C++ already, good — apply the same defensive parens habit in JS.

---

## 23. The Complete Bit-Trick Catalog

All tricks below operate correctly as long as the values involved fit in **31 usable bits** (indices `0..30`, magnitudes up to `2^30`). Bit 31 (the sign bit) is usable for AND/OR/XOR/NOT but breaks arithmetic-feeling tricks like `(1 << k) - 1` and positive-range assumptions — each trick below flags this where relevant. For `n > 30`, jump to §24 (BigInt).

### Quick-reference table

| Trick | Expression |
|---|---|
| Test bit `i` | `(x >> i) & 1` |
| Set bit `i` | `x \| (1 << i)` |
| Clear bit `i` | `x & ~(1 << i)` |
| Toggle bit `i` | `x ^ (1 << i)` |
| Is odd | `(x & 1) === 1` |
| Divide by 2ᵏ (floor, signed) | `x >> k` |
| Multiply by 2ᵏ | `x << k` |
| Clear lowest set bit | `x & (x - 1)` |
| Isolate lowest set bit | `x & -x` |
| Is power of two | `x > 0 && (x & (x - 1)) === 0` |
| Popcount (loop) | Brian Kernighan while-loop |
| Popcount (O(1)) | SWAR `popcount32` (§24) |
| Lowest `k` bits mask | `(1 << k) - 1` (k ≤ 30) |
| Highest set bit index | `31 - Math.clz32(x)` |
| Bit length | `32 - Math.clz32(x)` |
| Trailing zero count | `31 - Math.clz32(x & -x)` |
| Next subset of `mask` | `(s - 1) & mask` |
| All masks over `n` bits | `for (m = 0; m < (1 << n); m++)` (n ≤ 30) |
| int32 forced coercion | `x \| 0`, `~~x` |
| uint32 forced coercion | `x >>> 0` |

### Test / set / clear / toggle a bit

| Operation | Expression | Notes |
|---|---|---|
| Test bit `i` | `(x >> i) & 1` or `(x & (1 << i)) !== 0` | returns `0`/`1` or boolean |
| Set bit `i` | `x \| (1 << i)` | turns bit `i` **on** |
| Clear bit `i` | `x & ~(1 << i)` | turns bit `i` **off** |
| Toggle bit `i` | `x ^ (1 << i)` | flips bit `i` |
| Extract bit `i` as 0/1 | `(x >>> i) & 1` | use `>>>` if `x` might be negative and you want the raw bit, not sign-extension artifacts |

```js
let x = 0b1010; // 10

console.log((x >> 1) & 1);   // => 1   (bit 1 is set)
console.log((x >> 0) & 1);   // => 0   (bit 0 is clear)

x = x | (1 << 0);            // set bit 0
console.log(x.toString(2));  // => "1011"

x = x & ~(1 << 1);           // clear bit 1
console.log(x.toString(2));  // => "1001"

x = x ^ (1 << 3);            // toggle bit 3
console.log(x.toString(2));  // => "1"
```

### Odd/even and divide-by-power-of-two

```js
const isOdd = (n) => (n & 1) === 1;
console.log(isOdd(7));  // => true
console.log(isOdd(-7)); // => true   (works for negatives too — two's complement keeps bit 0 meaningful)

// divide by 2 (signed, floors toward -Infinity, like C++'s arithmetic shift)
console.log(13 >> 1);   // => 6
console.log(-13 >> 1);  // => -7   (NOT -6! floors toward -Infinity, unlike -13/2 === -6.5 → trunc → -6)

// divide by 2 unsigned reinterpretation
console.log((-13) >>> 1); // => 2147483641  (garbage unless you WANT the unsigned reinterpretation)

// multiply by power of two
console.log(5 << 3);    // => 40   (5 * 8)
```

> **Gotcha:** `x >> 1` for **negative** `x` is floor-division by 2, not truncating division. `-13 >> 1 === -7`, but `-13 / 2 === -6.5` and `Math.trunc(-13 / 2) === -6`. This differs from naive expectations carried over from C++ integer division (`-13 / 2 == -6` in C++, truncating toward zero) — the shift and the division round differently for negative numbers in BOTH languages, so don't assume `x >> 1 === x / 2` for negatives in either.

### Clear the lowest set bit — `x & (x - 1)`

```js
let x = 0b10110; // 22
console.log((x & (x - 1)).toString(2)); // => "10100"  (lowest set bit, bit 1, cleared)
```

```
  1 0 1 1 0    (x)
& 1 0 1 0 1    (x - 1: borrow flips trailing 0s to 1s, and the lowest 1 to 0)
-----------
  1 0 1 0 0
```

### Brian Kernighan popcount

Clearing the lowest set bit repeatedly counts set bits in `O(popcount)` instead of `O(32)`.

```js
function popcountBK(x) {
  x = x >>> 0; // normalize to unsigned 32-bit view first
  let count = 0;
  while (x !== 0) {
    x &= x - 1;
    count++;
  }
  return count;
}
console.log(popcountBK(0b10110101)); // => 5
```

> **Gotcha:** JS has **no built-in popcount** (no `__builtin_popcount`, no `Integer.bitCount` like Java). You must write it yourself — this Brian Kernighan loop, or the SWAR trick in §24, or (rarely, for one-offs) a string-based `.toString(2)` count.

### Isolate the lowest set bit — `x & (-x)`

```js
let x = 0b10110000; // 176
console.log((x & -x).toString(2)); // => "10000"  (isolates bit 4, value 16)
```

Why it works: `-x` in two's complement is `~x + 1`. Everything below the lowest set bit of `x` is `0` in `x`, so those bits become `1` in `~x`, then `+1` carries through them all the way up to and including the lowest set bit, cancelling everything else via the AND.

> **Gotcha:** This relies on two's-complement negation, exactly like C++. It works correctly for any `x` in the signed int32 range including negative `x`, but breaks (like everything else in this section) if `x` was meant to represent a value ≥ 2^31.

### Power-of-two check

```js
const isPowerOfTwo = (x) => x > 0 && (x & (x - 1)) === 0;

console.log(isPowerOfTwo(64));  // => true
console.log(isPowerOfTwo(63));  // => false
console.log(isPowerOfTwo(0));   // => false  (0 is NOT a power of two — the x > 0 guard matters)
console.log(isPowerOfTwo(-8));  // => false  (guard also excludes negatives, which would false-positive without it since (-8) & (-9) === 0 is NOT actually true here, but the x>0 guard is the standard safe form)
```

### XOR swap

```js
let a = 5, b = 9;
a ^= b;
b ^= a;
a ^= b;
console.log(a, b); // => 9 5
```

> **Gotcha:** Fails silently if `a` and `b` are the **same variable/same array slot** (`arr[i] ^= arr[i]` zeroes it out instead of leaving it unchanged) — identical trap to C++. Prefer destructuring `[a, b] = [b, a]` in JS; it's just as fast and has none of these edge cases.

### XOR properties → classic patterns

`x ^ x === 0`, `x ^ 0 === x`, XOR is commutative and associative.

**Single Number** (every element appears twice except one):

```js
function singleNumber(nums) {
  return nums.reduce((acc, n) => acc ^ n, 0);
}
console.log(singleNumber([4, 1, 2, 1, 2])); // => 4
```

**Two Single Numbers** (every element appears twice except exactly two):

```js
function singleNumberTwo(nums) {
  const xorAll = nums.reduce((a, n) => a ^ n, 0);       // xorAll = a ^ b, the two uniques
  const diffBit = xorAll & -xorAll;                      // any bit where a and b differ
  let groupA = 0;
  for (const n of nums) {
    if (n & diffBit) groupA ^= n;                        // partition by that bit, XOR within each group
  }
  const groupB = xorAll ^ groupA;
  return [groupA, groupB];
}
console.log(singleNumberTwo([1, 2, 1, 3, 2, 5])); // => [3, 5]  (order may vary)
```

**Missing number** in `[0..n]` (array holds `n` of the `n+1` values):

```js
function missingNumber(nums) {
  let x = nums.length; // start with n
  for (let i = 0; i < nums.length; i++) x ^= i ^ nums[i];
  return x;
}
console.log(missingNumber([3, 0, 1])); // => 2
```

**XOR of range `[0..n]`** — closed form, `O(1)` instead of looping:

```js
function xorUpTo(n) {
  // pattern repeats every 4 based on n % 4
  switch (n % 4) {
    case 0: return n;
    case 1: return 1;
    case 2: return n + 1;
    default: return 0; // n % 4 === 3
  }
}
console.log(xorUpTo(5)); // => 1   (0^1^2^3^4^5 = 1)

function xorRange(l, r) { // XOR of [l..r] inclusive
  return xorUpTo(r) ^ xorUpTo(l - 1);
}
console.log(xorRange(3, 7)); // => 4  (3^4^5^6^7)
```

### Masks: `(1 << k) - 1` and turning bits on/off in bulk

```js
// mask of the lowest k bits, ONLY VALID for k <= 30
const lowMask = (k) => (1 << k) - 1;

console.log(lowMask(4).toString(2));  // => "1111"
console.log(lowMask(0));              // => 0

// turn OFF all bits from position i upward (keep bits 0..i-1)
const keepBelow = (x, i) => x & ((1 << i) - 1);

// turn OFF all bits below position i (keep bit i and above)
const keepAtOrAbove = (x, i) => x & ~((1 << i) - 1);

console.log(keepBelow(0b111111, 3).toString(2));     // => "111"
console.log(keepAtOrAbove(0b111111, 3).toString(2)); // => "111000"
```

> **Gotcha:** `(1 << k) - 1` is only correct for `k` in `[0, 30]`. At `k = 31`, `1 << 31 === -2147483648`, and `-2147483648 - 1` wraps to `2147483647` (positive!) due to double arithmetic *before* any further bit op would re-truncate it — the formula silently produces the wrong mask shape. If you need a full 32-bit all-ones mask, use `-1` directly (`~0 === -1`, all 32 bits set) or `0xFFFFFFFF | 0`. For anything wider, use BigInt.

### Highest set bit / `floor(log2(x))` via `Math.clz32`

`Math.clz32` is JS's **one built-in bit primitive** — count leading zeros in the 32-bit representation. Memorize it; it's the escape hatch for half of "does JS have X built-in bit function" questions.

```js
Math.clz32(1);   // => 31  (0000...0001 has 31 leading zeros)
Math.clz32(0);   // => 32  (no bits at all)
Math.clz32(-1);  // => 0   (0xFFFFFFFF has zero leading zeros)

const floorLog2 = (x) => 31 - Math.clz32(x); // valid for x > 0
console.log(floorLog2(1));    // => 0
console.log(floorLog2(100));  // => 6   (2^6 = 64 <= 100 < 128 = 2^7)
```

### Next / previous power of two

```js
function nextPow2(x) {
  if (x <= 1) return 1;
  return 1 << (32 - Math.clz32(x - 1));
}
console.log(nextPow2(5));   // => 8
console.log(nextPow2(16));  // => 16  (already a power of two)

function prevPow2(x) {
  return 1 << (31 - Math.clz32(x)); // valid for x > 0
}
console.log(prevPow2(100)); // => 64
```

> **Gotcha:** `nextPow2`/`prevPow2` above are only safe while the *result* stays under `2^30` (i.e., input roughly `< 2^30`). Push them near `2^31` and the `1 << ...` inside flips negative, same 32-bit-signed trap as everywhere else in this section.

### Reverse bits (manual — no built-in)

```js
function reverseBits32(x) {
  x = x >>> 0;
  let result = 0;
  for (let i = 0; i < 32; i++) {
    result = (result << 1) | (x & 1);
    x >>>= 1;
  }
  return result >>> 0;
}
console.log(reverseBits32(0b1).toString(2).padStart(32, '0'));
// => "10000000000000000000000000000000"
```

### Count set bits without a builtin (recap + string-based option)

There is no `Integer.bitCount` (Java) or `__builtin_popcount` (C++) equivalent. Options:

```js
// loop-based (Brian Kernighan, shown above) — fastest for sparse bits
popcountBK(0b1011); // => 3

// string-based — clear but slow, fine for one-offs / debugging
const popcountStr = (x) => (x >>> 0).toString(2).split('').filter(c => c === '1').length;
console.log(popcountStr(0b1011)); // => 3

// SWAR bit-twiddling (constant time, no loop) — see popcount32 in §24
```

### Subset enumeration over a bitmask

The classic "iterate every subset of a given mask" trick, used constantly in bitmask-DP / SOS-DP:

```js
function subsetsOf(mask) {
  const subsets = [];
  for (let s = mask; s > 0; s = (s - 1) & mask) {
    subsets.push(s);
  }
  subsets.push(0); // the loop above skips the empty subset — add explicitly if needed
  return subsets;
}
console.log(subsetsOf(0b101).map(s => s.toString(2)));
// => [ '101', '100', '1', '0' ]
```

The `s > 0` loop condition **excludes the empty subset** (`s = 0` would make `(s - 1) & mask` wrap to `mask` itself via `-1 & mask`, looping forever if not guarded). If you need to include the empty subset in the walk itself, use a `do...while`:

```js
function forEachSubsetIncludingEmpty(mask, fn) {
  let s = mask;
  do {
    fn(s);
    s = (s - 1) & mask;
  } while (s !== mask);
}
forEachSubsetIncludingEmpty(0b101, (s) => process.stdout.write(s.toString(2) + ' '));
// => 101 100 1 0
```

**Complexity:** enumerating every subset of every mask over `n` bits is `O(3^n)`, not `O(4^n)`. Each of the `n` bits independently falls into one of 3 states across a `(mask, subset)` pair: *not in mask*, *in mask but not in subset*, *in mask and in subset* — so `sum over all masks of 2^popcount(mask) = sum_{k=0}^{n} C(n,k) * 2^k = 3^n` (binomial theorem). This is the standard justification for "subset-of-subset" DP being tractable up to roughly `n ≈ 15–20`.

> **Gotcha:** This whole pattern is **only valid while `mask` fits in ~30 bits** (`n ≤ 30`), same 32-bit-signed constraint as the rest of this section. For `n > 30`, you need BigInt masks and the arithmetic gets more awkward (no cheap `(s - 1) & mask` two's-complement trick without care — see §24's guidance on BigInt limitations).

### Iterate all masks over n bits

```js
for (let m = 0; m < (1 << n); m++) {
  // process mask m
}
```

> **Gotcha:** Requires `n <= 30`. At `n = 31`, `1 << 31` is negative, so the loop condition `m < (1 << 31)` is `m < -2147483648`, which is **never true** — the loop body never runs, silently producing zero iterations instead of an error. This is a nasty silent bug: no crash, no exception, just a loop that appears to "do nothing."

### Iterate set bits of a mask

```js
let m = 0b10110;
while (m) {
  const b = m & -m;              // isolate lowest set bit
  const i = 31 - Math.clz32(b);  // convert to bit index
  console.log(`bit ${i} is set`);
  m ^= b;                        // clear it and continue
}
// => bit 1 is set
// => bit 2 is set
// => bit 4 is set
```

### Counting-bits DP

Classic `O(n)` table of popcounts using the recurrence `dp[i] = dp[i >> 1] + (i & 1)` (drop the lowest bit by halving, add it back if it was set):

```js
function countBits(n) {
  const dp = new Array(n + 1).fill(0);
  for (let i = 1; i <= n; i++) {
    dp[i] = dp[i >> 1] + (i & 1);
  }
  return dp;
}
console.log(countBits(7)); // => [0, 1, 1, 2, 1, 2, 2, 3]
```

### Binary string ↔ number conversions

Constantly needed for debugging bit tricks and for problems that ask you to manipulate binary directly:

```js
(13).toString(2);              // => "1101"
(13).toString(2).padStart(8, '0'); // => "00001101"   (fixed-width display)
parseInt('1101', 2);           // => 13
Number.parseInt('1101', 2);    // => 13   (same thing, more explicit)

// negative numbers stringify with a leading '-', NOT two's-complement bits:
(-5).toString(2);              // => "-101"   (NOT "...11111011")

// to see the actual two's-complement 32-bit pattern of a negative number:
(-5 >>> 0).toString(2);        // => "11111111111111111111111111111011"
```

> **Gotcha:** `(-5).toString(2)` prints the sign and magnitude (`"-101"`), not the two's-complement bit pattern. If you want to *see* the raw 32-bit pattern (e.g. to sanity-check a bitwise result), coerce through `>>> 0` first to reinterpret as unsigned, then stringify.

### Gray code

`n ^ (n >> 1)` converts a binary count into Gray code, where consecutive values differ by exactly one bit — used in "generate all subsets/permutations with a single-bit-flip transition" problems.

```js
function grayCode(n) {
  return n ^ (n >> 1);
}
for (let i = 0; i < 4; i++) {
  console.log(i.toString(2).padStart(2, '0'), '->', grayCode(i).toString(2).padStart(2, '0'));
}
// => 00 -> 00
// => 01 -> 01
// => 10 -> 11
// => 11 -> 10
```

### The discipline: n > 30 items means NO number bitmasks

If a problem's "choose a subset of `n` items" state needs `n > 30` (e.g., `n` up to 40, 50, 60), a plain JS `Number` bitmask is **unsafe** — the moment you shift or mask past bit 30 you hit sign-bit / 32-bit-truncation bugs described throughout this section. Your two correct options:

1. **BigInt bitmask** — full arbitrary width, same operator syntax, no truncation, but slower and needs `n`-suffixed literals (§24).
2. **Typed array / boolean array** — a `Uint8Array`/plain `boolean[]` indexed set, or a `Uint32Array` used as a manual bitset (one `Uint32` per 32 items) — fastest for very large `n`, no BigInt overhead, but you write the bit-indexing arithmetic by hand.

See the decision table in §24.

---

## 24. BigInt Bitwise, Math.clz32, and Wide Masks

### `Math.clz32` in depth

`Math.clz32(x)` — **c**ount **l**eading **z**eros of `x`'s 32-bit representation. It is the **only** built-in bit-manipulation helper JS ships (no popcount, no trailing-zero-count, no bit-length function — you derive all of those from `clz32`).

```js
Math.clz32(1);          // => 31
Math.clz32(0);          // => 32   (all 32 bits are "leading zeros" when there are no set bits)
Math.clz32(0xFFFFFFFF); // => 0
Math.clz32(-1);         // => 0    (-1's bit pattern is 0xFFFFFFFF)
Math.clz32(2 ** 31);    // => 0    (2^31 truncates to 0x80000000 → sign bit is the only bit set → 0 leading zeros)
```

Quantities you can derive from it (all for `x` interpreted as an unsigned/positive 32-bit pattern unless noted):

| Derived quantity | Formula | Notes |
|---|---|---|
| `floor(log2(x))` | `31 - Math.clz32(x)` | valid for `x > 0` |
| Highest set bit index | `31 - Math.clz32(x)` | same as above |
| Bit length (`0` bits needed) | `32 - Math.clz32(x)` | works for `x === 0` too → `0` |
| Is power of two (alt.) | `x > 0 && Math.clz32(x) === Math.clz32(x - 1) - 1`... | prefer the `x & (x-1)` version — clearer |

### Helper functions you'll want to keep handy

```js
/** Count trailing zeros of a 32-bit int. ctz32(0) === 32 by convention. */
function ctz32(x) {
  x = x | 0;
  if (x === 0) return 32;
  return 31 - Math.clz32(x & -x); // isolate lowest set bit, then find its index
}
console.log(ctz32(0b1000)); // => 3
console.log(ctz32(0));      // => 32

/** Bit length: number of bits needed to represent x (x >= 0). */
function bitLength(x) {
  return 32 - Math.clz32(x >>> 0);
}
console.log(bitLength(0));   // => 0
console.log(bitLength(1));   // => 1
console.log(bitLength(255)); // => 8
console.log(bitLength(256)); // => 9

/** SWAR popcount — O(1), no loop. Correct for the full uint32 range. */
function popcount32(x) {
  x = x - ((x >>> 1) & 0x55555555);
  x = (x & 0x33333333) + ((x >>> 2) & 0x33333333);
  x = (x + (x >>> 4)) & 0x0f0f0f0f;
  return (x * 0x01010101) >> 24;
}
console.log(popcount32(0xFFFFFFFF)); // => 32
console.log(popcount32(0b10110101)); // => 5
```

> **Gotcha:** The `popcount32` multiply step (`x * 0x01010101`) is a **regular double multiplication**, not a bitwise op — it does NOT truncate mid-computation. It stays exact because the intermediate value never exceeds `2^53` (double's exact-integer limit), and only the final `>> 24` re-enters 32-bit-int land. This is why the classic Hacker's-Delight SWAR trick ports cleanly to JS despite JS's "no true 32-bit ints" nature — multiplication and addition are done at full double precision, only the bitwise operators themselves truncate.

### BigInt bitwise — the escape hatch for width > 31 bits

`BigInt` supports `&`, `|`, `^`, `~`, `<<`, `>>` with **full arbitrary width** — no 32-bit truncation, ever. This is what you reach for once your bitmask needs more than ~30 usable bits (subsets over 40, 50, 60+ items; arbitrary-precision bit tricks).

```js
let mask = 0n;
mask |= (1n << 5n);   // set bit 5
mask |= (1n << 49n);  // set bit 49 — NO truncation, unlike a Number would suffer
console.log(mask.toString(2));
// => "10000000000000000000000000000100000"  (bit 49 and bit 5 set)

console.log((mask >> 49n) & 1n); // => 1n   (test bit 49)
console.log((mask >> 5n) & 1n);  // => 1n   (test bit 5)
console.log((mask >> 6n) & 1n);  // => 0n   (test bit 6 — clear)
```

**Set / test / clear / toggle for BigInt masks:**

```js
const setBit    = (mask, i) => mask | (1n << BigInt(i));
const clearBit  = (mask, i) => mask & ~(1n << BigInt(i));
const toggleBit = (mask, i) => mask ^ (1n << BigInt(i));
const testBit   = (mask, i) => ((mask >> BigInt(i)) & 1n) === 1n;

let m = 0n;
m = setBit(m, 3);
m = setBit(m, 40);
console.log(testBit(m, 3));   // => true
console.log(testBit(m, 4));   // => false
console.log(m.toString(2));   // => "10000000000000000000000000000000001000"
```

**Worked example: subset bitmask over 50 items**

```js
function subsetSumBigMask(weights, targetIdxSet) {
  // targetIdxSet: array of indices to include
  let mask = 0n;
  for (const i of targetIdxSet) mask = setBit(mask, i);

  let sum = 0;
  for (let i = 0; i < weights.length; i++) {
    if (testBit(mask, i)) sum += weights[i];
  }
  return sum;
}
const weights = Array.from({ length: 50 }, (_, i) => i + 1); // [1..50]
console.log(subsetSumBigMask(weights, [0, 10, 49])); // => 1 + 11 + 50 = 62
```

### BigInt bitwise gotchas

> **Gotcha:** You **cannot mix** `BigInt` and `Number` in arithmetic or bitwise operators — it throws `TypeError: Cannot mix BigInt and other types`. `1n << 5` is an error; you need `1n << 5n` (or `1n << BigInt(5)`).

```js
try {
  const bad = 1n << 5; // Number 5, not BigInt
} catch (e) {
  console.log(e.message); // => "Cannot mix BigInt and other types, use explicit conversions"
}
```

> **Gotcha:** BigInt has **no `>>>` (unsigned right shift)** — it's a `SyntaxError`/`TypeError` depending on engine, because BigInt is conceptually infinite-precision two's complement with no fixed width, so "unsigned" is meaningless. `BigInt`'s `>>` is always the arithmetic (sign-preserving) shift.

```js
// 5n >>> 1n;  // TypeError: BigInts have no unsigned right shift, use >> instead
```

> **Gotcha:** BigInt shifts are **not** masked mod-anything — `1n << 100n` genuinely shifts by 100 bit positions (unlike `1 << 100` on Numbers, which effectively shifts by `100 & 31 === 4`). This is usually what you want, but it means huge shift counts produce huge BigInts (memory!) instead of wrapping.

```js
console.log((1 << 100));        // => 16   (100 & 31 === 4, so this is 1 << 4)
console.log((1n << 100n).toString(2).length); // => 101  (an actual 101-bit-long number, no wraparound)
```

### Popcount for BigInt

No SWAR shortcut needed for correctness (though a chunked version exists) — simplest is Brian-Kernighan-style or a string count:

```js
function popcountBig(x) {
  let count = 0n;
  while (x > 0n) {
    x &= x - 1n;
    count++;
  }
  return count;
}
console.log(popcountBig((1n << 60n) | (1n << 3n) | 1n)); // => 3n

// string-based alternative
function popcountBigStr(x) {
  return x.toString(2).split('').filter(c => c === '1').length;
}
console.log(popcountBigStr(0b1011n)); // => 3
```

### Performance note

BigInt is **significantly slower** than Number bitwise ops (arbitrary-precision arithmetic has real overhead — allocation, no JIT fast-path the way small-int Numbers get). Rule of thumb:

- `n <= 30`: use plain `Number` bitmasks. Always. This is the fast path and covers the overwhelming majority of competitive-programming bitmask problems.
- `n` in `31..~1000`-ish and you specifically need bitmask semantics (not just "a big set"): use `BigInt`.
- `n` very large (thousands+) and you mainly need membership tests / iteration, not bit-shift tricks: use a **typed array or boolean array** instead — it's faster than BigInt and more memory-predictable.

### Decision table: number mask vs BigInt mask vs array

| Situation | Use | Why |
|---|---|---|
| `n <= 30`, need `<<`, `\|`, `&`, `^`, subset enumeration, bitmask-DP | **Number bitmask** | fastest, native operators, fits int32 |
| `n` in `~31..1000`, need real bit-shift/mask semantics (e.g. bitmask-DP with `n` up to 40-60) | **BigInt bitmask** | correctness > speed here; only option with `<<`/`&`/`\|` semantics at this width |
| Large `n`, just need "is item `i` in the set" / toggling, no shift-heavy bit tricks | **`Uint8Array`/`boolean[]`** indexed set | faster than BigInt, simplest code |
| Very large `n` (memory-sensitive), want compact bitset with manual word indexing | **`Uint32Array`** as a manual bitset (`arr[i >>> 5] |= 1 << (i & 31)`) | most memory-efficient, still Number-op speed per word |
| Need arbitrary-precision integer *arithmetic* (not just bit ops) | **BigInt** | e.g. huge factorials/products where bit width isn't the point |

### Worked bitmask-DP snippet (small n — reference DSA Pattern 24)

Held-Karp style TSP / assignment DP using a `Number` bitmask (`n <= ~20` is the realistic ceiling given `O(2^n * n^2)` or `O(2^n * n)` time, well within the `n <= 30` bitmask-safety window):

```js
function tspMinCost(dist) {
  const n = dist.length;
  const FULL = (1 << n) - 1;
  const dp = Array.from({ length: 1 << n }, () => new Array(n).fill(Infinity));
  dp[1][0] = 0; // start at city 0, mask = {0}

  for (let mask = 1; mask <= FULL; mask++) {
    for (let u = 0; u < n; u++) {
      if (!(mask & (1 << u)) || dp[mask][u] === Infinity) continue;
      for (let v = 0; v < n; v++) {
        if (mask & (1 << v)) continue; // v already visited
        const next = mask | (1 << v);
        const cand = dp[mask][u] + dist[u][v];
        if (cand < dp[next][v]) dp[next][v] = cand;
      }
    }
  }

  let best = Infinity;
  for (let u = 1; u < n; u++) {
    if (dp[FULL][u] !== Infinity) best = Math.min(best, dp[FULL][u] + dist[u][0]);
  }
  return best;
}

const dist = [
  [0, 10, 15, 20],
  [10, 0, 35, 25],
  [15, 35, 0, 30],
  [20, 25, 30, 0],
];
console.log(tspMinCost(dist)); // => 80
```

### Counting-bits DP (recap, tied to bit-length reasoning)

```js
function countBits(n) {
  const dp = new Int32Array(n + 1); // typed array: compact, fast, fine since values are tiny
  for (let i = 1; i <= n; i++) {
    dp[i] = dp[i >> 1] + (i & 1);
  }
  return Array.from(dp);
}
console.log(countBits(10)); // => [0,1,1,2,1,2,2,3,1,2,2]
```

> **vs C++:** In C++ you'd reach for `__builtin_popcount`/`__builtin_clz`/`__builtin_ctz` (GCC/Clang intrinsics) or `<bit>` header's `std::popcount`/`std::countl_zero`/`std::countr_zero` (C++20). In Java, `Integer.bitCount`/`numberOfLeadingZeros`/`numberOfTrailingZeros`. JS gives you exactly **one** of these natively (`Math.clz32`) — everything else (`popcount32`, `ctz32`, `bitLength`) you write yourself, once, and keep in a snippet file. That asymmetry, plus the 32-bit-signed truncation from §22, are the two things to actively remember mid-contest: JS bit tricks are a strict subset of what C++/Java give you for free, and the width ceiling is lower than it looks.

---

# PART VI — NUMBERS, PRECISION, BIGINT AND I/O

## 25. Number Precision, Safe Integers and Overflow

JS has **one** numeric type for all non-BigInt numbers: `number`, an IEEE-754 **double** (64-bit float). There is no separate `int`, `long`, `float`, `double` — `1`, `1.0`, and `1e0` are the exact same value and same type.

> **vs C++/Java:** C++/Java have distinct integer widths (`int32`, `int64`/`long long`) with wraparound overflow. JS has no integer type at all — every "integer" you use is really a double that happens to hold an integer value.

### 25.1 Safe integer range

A double can represent every integer **exactly** only up to `2^53 - 1`. Beyond that, some integers cannot be represented and get silently rounded to the nearest representable double.

```js
console.log(Number.MAX_SAFE_INTEGER);      // => 9007199254740991  (2^53 - 1)
console.log(Number.MIN_SAFE_INTEGER);      // => -9007199254740991
console.log(Number.MAX_SAFE_INTEGER + 1);  // => 9007199254740992  (still "correct" — edge)
console.log(Number.MAX_SAFE_INTEGER + 2);  // => 9007199254740992  (WRONG — should be ...993, lost!)
```

### 25.2 This is NOT overflow-wrap — it is silent precision loss

In C++/Java, exceeding `long long`/`long` range wraps around (undefined behavior in C++ signed overflow, but in practice wraps; Java wraps by spec). In JS, exceeding 2^53 does **not** wrap — it rounds to the nearest representable double, silently producing a wrong-but-plausible-looking number.

```js
console.log(2 ** 53 + 1 === 2 ** 53);   // => true   -- two different math values, same double!
console.log(2 ** 53 + 1);               // => 9007199254740992  (should be ...4740993)

console.log(999999999999999999);        // => 1000000000000000000  (last digits silently lost)
console.log(999999999999999999n);       // => 999999999999999999n (BigInt: exact, see §26)
```

> **Gotcha:** This is a classic silent Wrong-Answer source in competitive programming. There is no crash, no exception — the number just quietly becomes a different, nearby number.

### 25.3 The overflow-before-mod trap (`(a * b) % m`)

In C++, `long long` holds up to ~9.2e18 exactly, so `(a * b) % m` with `a, b < 1e9` is safe (`a * b` up to ~1e18, well within `long long`). In JS, `a * b` for `a, b ~ 1e9` is up to ~1e18, which **exceeds 2^53 (~9.007e15)** — precision is already lost *before* the `%` is even applied.

```js
const a = 1_000_000_000, b = 999_999_999, m = 1_000_000_007;

// WRONG in JS — a * b overflows the safe-integer range before % is applied
console.log((a * b) % m);   // => some number, but NOT the mathematically correct one

// CORRECT — use BigInt for the multiplication (see §26)
console.log((BigInt(a) * BigInt(b)) % BigInt(m));   // => 999999999n (exact)
```

> **vs C++:** `(long long)a * b % m` is exact because `long long` covers the full ~1e18 product. JS's double does **not** — anything with products/sums that can exceed ~9e15 needs BigInt or a mulmod trick (e.g. splitting the multiplication into high/low 32-bit halves) — never trust plain `Number` arithmetic near or past 2^53.

### 25.4 `Number.isSafeInteger` / `Number.isInteger`

```js
Number.isSafeInteger(2 ** 53);       // => false (right at the boundary, already unsafe)
Number.isSafeInteger(2 ** 53 - 1);   // => true
Number.isInteger(5.0);               // => true
Number.isInteger(5.5);               // => false
Number.isInteger("5");               // => false (no coercion, unlike ==)
```

Use `Number.isSafeInteger(x)` as a guard before doing chained arithmetic in DSA problems with large bounds (n up to 1e9, sums up to 1e18, etc.) — if the guard fails, switch to BigInt.

### 25.5 Floating point is NOT exact — never compare with `===`

Binary floating point cannot represent most decimal fractions exactly (same issue as C++/Java `double`/`float`).

```js
console.log(0.1 + 0.2);          // => 0.30000000000000004
console.log(0.1 + 0.2 === 0.3);  // => false

// correct comparison: epsilon tolerance
function almostEqual(a, b, eps = 1e-9) {
  return Math.abs(a - b) < eps;
}
console.log(almostEqual(0.1 + 0.2, 0.3));  // => true

console.log(Number.EPSILON);     // => 2.220446049250313e-16  (smallest diff representable near 1)
```

> **Gotcha:** never write `if (x === y)` for computed floats. Always use an epsilon comparison, exactly like in C++.

### 25.6 `Math.trunc` vs `| 0` — the 32-bit trap

```js
Math.trunc(4.9);      // => 4
Math.trunc(-4.9);     // => -4
(4.9 | 0);             // => 4
(-4.9 | 0);            // => -4

// | 0 secretly converts through a 32-bit signed integer first — breaks past 2^31 - 1
console.log(3000000000 | 0);        // => -1294967296  (WRONG, wrapped as if int32)
console.log(Math.trunc(3000000000)); // => 3000000000  (correct)
```

> **Gotcha:** `| 0` is a common "fast truncate" trick but it silently truncates to **32-bit** range (like a C++ `int32_t` cast), not the 53-bit safe-integer range. Only use `| 0` when you are certain the value fits in a signed 32-bit int (e.g. array indices, small loop counters). Prefer `Math.trunc`/`Math.floor` in general DSA code.

### 25.7 Avoiding floats in DSA — use integer math

When possible, avoid floating point entirely in CP:

```js
// BAD: comparing fractions with floats
function fracLess(a1, b1, a2, b2) {          // a1/b1 < a2/b2 ?
  return a1 / b1 < a2 / b2;                   // precision risk
}

// GOOD: cross-multiplication (assuming positive denominators)
function fracLessExact(a1, b1, a2, b2) {
  return a1 * b2 < a2 * b1;                   // pure integer comparison, exact
}
```

Prefer integer-only formulations: binary search on the answer with integer bounds, cross-multiplication for fraction comparisons, scaling by a fixed power of 10 to work in "cents" instead of decimal dollars, etc.

### 25.8 `-0` vs `0`

```js
console.log(-0 === 0);          // => true   (== and === treat them equal)
console.log(Object.is(-0, 0));  // => false  (Object.is distinguishes them)
console.log(1 / -0);            // => -Infinity  (sign survives division)
console.log(1 / 0);             // => Infinity
```

> **Gotcha:** `-0` mostly behaves like `0`, but `Object.is`, `1/x`, and `JSON.stringify` (`JSON.stringify(-0) === "0"` — actually stringifies as `"0"`, hiding the sign) can expose the difference. Rare in CP but occasionally bites when using `-0` as a sentinel or sorting with a comparator that returns `-0`.

| Concept | JS | C++ | Java |
|---|---|---|---|
| Integer type | none — double up to 2^53 exact | `int`(32b), `long long`(64b) | `int`(32b), `long`(64b) |
| Overflow behavior | silent rounding (no wrap) | wraps (UB for signed, but practically wraps) | wraps by spec |
| Exact int range | ±(2^53 − 1) | ±(2^63 − 1) for `long long` | ±(2^63 − 1) for `long` |
| Float compare | never `===`, use epsilon | never `==`, use epsilon | never `==`, use epsilon |
| Arbitrary precision | `BigInt` | none built-in (need library) | `BigInteger` |

---

## 26. BigInt in Depth

`BigInt` (ES2020) is JS's arbitrary-precision integer type — the answer to needing exact integers beyond 2^53. It is a genuinely separate type from `number`, not an extension of it.

> **vs C++/Java:** C++ has no arbitrary-precision integer in the standard library (you'd write your own bignum or use Boost/`__int128` for a bit more headroom). Java has `java.math.BigInteger`. JS builds it into the language with its own literal syntax and operators, but performance is much worse than `Number` — use it only when you actually need >2^53 precision or exact 64-bit-scale math.

### 26.1 Literals and construction

```js
const a = 123n;                       // BigInt literal — trailing 'n'
const b = BigInt(123);                // from Number (must be a safe integer)
const c = BigInt("123456789012345678901234567890");  // from string — arbitrary size
const d = BigInt("0x1f");             // => 31n  (hex string also works)

console.log(typeof a);                // => 'bigint'
console.log(a);                       // => 123n
```

> **Gotcha:** `BigInt(1.5)` throws `RangeError: The number 1.5 cannot be converted to a BigInt` — only integer-valued Numbers convert cleanly. And `BigInt(2 ** 60)` will silently carry over whatever precision loss the `Number` already had — convert from a **string** when the source might already exceed 2^53.

### 26.2 Arithmetic — `+ - * / % **`

```js
console.log(10n + 3n);   // => 13n
console.log(10n - 3n);   // => 7n
console.log(10n * 3n);   // => 30n
console.log(10n / 3n);   // => 3n   (TRUNCATES toward zero, like C++/Java integer division)
console.log(-10n / 3n);  // => -3n  (truncation, not floor: floor would be -4n)
console.log(10n % 3n);   // => 1n
console.log(-10n % 3n);  // => -1n  (sign follows dividend, same as C++/Java %)
console.log(2n ** 100n); // => 1267650600228229401496703205376n  (exact!)
```

> **vs C++/Java:** BigInt `/` truncates toward zero exactly like C++/Java integer division on signed types — no surprise there. The surprise is everywhere else (mixing types, no `Math`, performance).

### 26.3 Cannot mix BigInt and Number — must convert explicitly

```js
try {
  1n + 1;                              // TypeError!
} catch (e) {
  console.log(e.message);              // => Cannot mix BigInt and other types, use explicit conversions
}

console.log(1n + BigInt(1));           // => 2n   (convert Number -> BigInt)
console.log(Number(1n) + 1);           // => 2    (convert BigInt -> Number)
```

> **Gotcha:** this is a constant source of friction in CP code that mixes plain loop counters (`Number`) with BigInt accumulators — every arithmetic op needs an explicit `BigInt(...)` wrapper on the `Number` side. Decide up front whether a variable is "a BigInt variable" or "a Number variable" and stick to it, converting only at the boundary.

### 26.4 Comparison works across types (unlike arithmetic)

```js
console.log(1n == 1);     // => true   (== does numeric coercion across types)
console.log(1n === 1);    // => false  (=== checks type first — different types!)
console.log(1n < 2);      // => true   (relational operators coerce, no error)
console.log(1n <= 1);     // => true
console.log([3n, 1n, 2n].sort());  // => [ 1n, 2n, 3n ]  (default sort still lexicographic on string form for BigInt too — careful, see below)
```

> **Gotcha:** the default `Array.prototype.sort()` converts elements to strings — this still "works" for single-digit BigInts by luck, but breaks for BigInts of different digit counts. Always pass a comparator: `arr.sort((a, b) => (a < b ? -1 : a > b ? 1 : 0))`.

### 26.5 Converting back to Number

```js
const big = 12345678901234567890n;
console.log(Number(big));            // => 12345678901234567000  (precision LOST — expected, >2^53)
console.log(Number(123n));           // => 123  (safe — fits exactly)
console.log(parseInt("123n"));       // => 123  (parseInt stops at 'n', returns a Number, not BigInt)
```

> **Gotcha:** `Number(bigBigInt)` never throws — it silently rounds, same precision-loss story as §25. Only convert BigInt → Number when you've confirmed the value is within `Number.MAX_SAFE_INTEGER`.

### 26.6 No `Math.*` for BigInt — write your own

```js
Math.sqrt(4n);   // throws TypeError: Cannot convert a BigInt value to a number
```

`Math.sqrt`, `Math.abs`, `Math.max`, etc. all reject BigInt. You must hand-roll integer versions:

```js
function bigAbs(x) {
  return x < 0n ? -x : x;
}

// Integer square root via Newton's method — needed because Math.sqrt rejects BigInt
function isqrt(n) {
  if (n < 0n) throw new RangeError('negative');
  if (n < 2n) return n;
  let x0 = n, x1 = (x0 + 1n) >> 1n;
  while (x1 < x0) {
    x0 = x1;
    x1 = (x0 + n / x0) >> 1n;
  }
  return x0;
}

console.log(isqrt(1_000_000_000_000n));   // => 1000000n
console.log(isqrt(999_999_999_999n));     // => 999999n  (floor of sqrt)
```

### 26.7 Modular exponentiation — THE reason CP reaches for BigInt

```js
// modpow(base, exp, mod): computes (base^exp) % mod exactly, base/exp/mod as BigInt
function modpow(base, exp, mod) {
  base %= mod;
  if (base < 0n) base += mod;
  let result = 1n;
  while (exp > 0n) {
    if (exp & 1n) result = (result * base) % mod;
    base = (base * base) % mod;
    exp >>= 1n;
  }
  return result;
}

console.log(modpow(2n, 10n, 1_000_000_007n));   // => 1024n
console.log(modpow(7n, 1_000_000_000n, 1_000_000_007n));  // => large exact result, e.g. 178571426n
```

### 26.8 Modular inverse via Fermat's little theorem (prime mod)

```js
// modInverse(a, p): a^-1 mod p, valid when p is prime and gcd(a, p) = 1
function modInverse(a, p) {
  return modpow(a, p - 2n, p);
}

const MOD = 1_000_000_007n;
console.log(modInverse(2n, MOD));               // => 500000004n
console.log((2n * modInverse(2n, MOD)) % MOD);  // => 1n  (sanity check)
```

### 26.9 When BigInt is REQUIRED vs when it's a needless slowdown

| Need | Use BigInt? |
|---|---|
| Loop counters, array indices, n ≤ ~1e15 | No — `Number` is fine and much faster |
| Sum/product that can exceed ~9e15 (2 numbers ~1e9 multiplied) | **Yes** |
| `(a * b) % m` with a, b ~1e9 | **Yes** (or a mulmod trick) |
| Modular exponentiation for large exponents | **Yes** |
| Exact factorials beyond ~18! (18! ≈ 6.4e15 already near the edge; 19!+ definitely) | **Yes** |
| 64-bit hashing / rolling hash with large base & mod | **Yes** |
| Bitwise ops that must exceed 32-bit width (see Part V) | **Yes** |
| General array/loop bookkeeping | No |

> **Gotcha (performance):** BigInt arithmetic is **much slower** than `Number` arithmetic — often 10-100x for simple ops, because it's arbitrary-precision under the hood, not a hardware register op. In a hot inner loop processing 1e7+ iterations, blanket-converting everything to BigInt "just in case" can cause a TLE by itself. Use BigInt surgically only where the value can truly exceed 2^53 — keep loop counters and small values in plain `Number`.

### 26.10 BigInt bitwise ops — full width, no 32-bit truncation

Cross-reference: Part V covers `Number` bitwise operators, which silently truncate operands to 32-bit signed integers (`5 | 0`, `~5`, `<<`, `>>>`, etc. all operate on a 32-bit view). BigInt bitwise operators (`& | ^ ~ << >>`) do **not** truncate — they operate on the full (conceptually infinite, two's-complement) value:

```js
console.log(1 << 35);          // => 8  (Number: shift amount mod 32, useless past 31 bits!)
console.log(1n << 35n);        // => 34359738368n  (BigInt: exact, no truncation)

console.log((0xFFFFFFFFn & 0xFn));   // => 15n  (works on full width, no 32-bit clamp)
```

> **vs C++/Java:** this makes BigInt the natural choice in JS for any bitmask/bit-trick DP that needs more than 31 usable bits (`Number`'s safe bitwise range is effectively 32-bit signed) — where C++/Java would just use a wider integer type (`uint64_t`, `long`), JS reaches for BigInt instead. `>>>` (unsigned right shift) has **no BigInt equivalent** — since BigInt has no fixed width, "unsigned" is meaningless; there's nothing to fill with zero bits beyond a boundary.

---

## 27. Fast I/O in Node.js (the make-or-break CP topic)

> **THE RULE:** the friendly `readline` module used line-by-line, and calling `console.log` inside a loop, are **too slow** for large Codeforces/AtCoder-style input and output. Both cause Time Limit Exceeded on problems with 1e5+ lines of input or output. Every serious CP submission in JS should read all of stdin **once** up front and write all of stdout **once** at the end.

### 27.1 The standard fast-input pattern — read all of stdin at once

```js
const data = require('fs').readFileSync(0, 'utf8').split(/\s+/).map(Number);
// fd 0 = stdin. '/dev/stdin' also works on most judges as a fallback:
// const data = require('fs').readFileSync('/dev/stdin', 'utf8').split(/\s+/).map(Number);

let idx = 0;
const next = () => data[idx++];

// Example: first token is n, then n integers follow
const n = next();
const arr = Array.from({ length: n }, next);
console.log(arr);   // => [ ...n numbers... ]
```

- `readFileSync(0, 'utf8')` reads the whole input synchronously as one string — one syscall-level read, not one per line.
- `.split(/\s+/)` splits on any run of whitespace (spaces, tabs, newlines) — perfect for space/newline separated tokens; trims a possible leading empty token if the input starts with whitespace (guard with `.filter(Boolean)` if paranoid).
- `.map(Number)` converts every token to a `Number` eagerly. Skip this (`.split(...)` only, no `.map(Number)`) if input is mixed strings and numbers — convert per-token instead (see §27.2).

### 27.2 Line-based variant (when structure is line-oriented, e.g. a grid)

```js
const lines = require('fs').readFileSync(0, 'utf8').split('\n');

let ptr = 0;
const nextLine = () => lines[ptr++];

const n = Number(nextLine());
const rows = Array.from({ length: n }, nextLine);   // e.g. grid of characters, one row per line
console.log(rows);   // => [ 'row0string', 'row1string', ... ]
```

> **Gotcha:** trailing newline / trailing empty line at EOF is common — `split('\n')` may leave an empty string as the last element. Guard with `.filter(line => line.length > 0)` if you iterate over "all lines" rather than reading an exact known count.

### 27.3 Token-generator / pointer-based reader for mixed int/string input

For problems where tokens are a mix of numbers and words/strings, don't blanket `.map(Number)` — parse per-token, on demand:

```js
const tokens = require('fs').readFileSync(0, 'utf8').split(/\s+/).filter(Boolean);
let p = 0;

const nextStr = () => tokens[p++];
const nextInt = () => Number(tokens[p++]);
const nextBigInt = () => BigInt(tokens[p++]);

// Example: "3 apple 5 banana 2 cherry" — pairs of (count, name)
const pairs = [];
const m = 3;
for (let i = 0; i < m; i++) {
  pairs.push([nextInt(), nextStr()]);
}
console.log(pairs);  // => [ [3,'apple'], [5,'banana'], [2,'cherry'] ]  (illustrative)
```

This "generator-style" reader (a shared pointer + typed `next*` functions) is the standard idiom — it composes with any input shape without re-parsing the whole buffer.

### 27.4 Output: never `console.log` inside a loop

```js
// BAD — TLE risk: each console.log call is a relatively expensive, synchronous flush
for (let i = 0; i < 100000; i++) {
  console.log(i);          // DO NOT DO THIS for large n
}

// GOOD — accumulate into an array, join once, write once
const out = [];
for (let i = 0; i < 100000; i++) {
  out.push(i);
}
console.log(out.join('\n'));                          // one flush total
// or, equivalently and marginally faster (avoids console formatting overhead):
process.stdout.write(out.join('\n') + '\n');
```

> **Gotcha:** `out.push(String(i))` vs `out.push(i)` — `Array.prototype.join` stringifies non-string elements automatically, so pushing raw numbers and joining is fine and avoids a manual `String()` call per element.

### 27.5 The `readline` interface — fine for small/interactive, not for bulk

```js
const readline = require('readline');
const rl = readline.createInterface({ input: process.stdin, terminal: false });

const lines = [];
rl.on('line', (line) => {
  lines.push(line);
});
rl.on('close', () => {
  // all input has been received — process it here
  const n = Number(lines[0]);
  console.log(n);   // => whatever the first line contained, as a Number
});
```

`readline` is event-driven and processes input incrementally as it arrives — useful for genuinely interactive problems (judge sends a line, expects a response, sends the next line based on your answer) or when input is streamed rather than fully available up front. For bulk offline input (the overwhelming majority of CP problems), the `readFileSync`-based synchronous read is simpler and faster — there's no reason to pay event-loop overhead for input that's already sitting fully in stdin before your program starts.

### 27.6 Comparison table

| Method | Speed | When to use |
|---|---|---|
| `readline` (`rl.on('line')`) | Slow-ish (event-loop overhead per line) | Small input, genuinely interactive/streamed problems |
| `readFileSync(0,'utf8').split(...)` | Fast (single read, single parse pass) | **Default choice** for almost all CP problems |
| Manual byte parsing (`readFileSync(0)` as `Buffer`, parse bytes directly) | Fastest, but fiddly | Only for extreme input sizes (1e7+ tokens) where even `.split`/`.map` overhead matters |
| `console.log` per line | Slow (flush per call) | Never for loops — only for a single final line |
| `process.stdout.write(out.join('\n'))` | Fast (single write) | **Default choice** for output |

### 27.7 Parsing integers — the options

```js
Number("42");        // => 42        (general purpose, handles floats/negatives/exponents too)
parseInt("42abc");   // => 42        (stops at first non-digit — sometimes useful, sometimes a footgun)
parseInt("42", 10);  // => 42        (ALWAYS pass radix 10 explicitly — see Part V/leading-zero gotchas)
+"42";               // => 42        (unary plus — same as Number(), terser)
"42" | 0;            // => 42        (fast but 32-bit-truncating — see §25.6, avoid for big values)
BigInt("123456789012345678901234567890");  // exact, arbitrary precision — for values beyond 2^53
```

> **Gotcha:** `parseInt` silently stops at the first invalid character instead of failing (`parseInt("12x34")` → `12`), whereas `Number("12x34")` → `NaN`. For validated token streams (typical CP input), `Number(token)` is the standard, safest choice — reserve `parseInt` for when you deliberately want "parse the leading digits, ignore the rest."

### 27.8 Common input shapes

**Read n, then n numbers on the same or following line(s):**
```js
const data = require('fs').readFileSync(0, 'utf8').split(/\s+/).filter(Boolean).map(Number);
let i = 0;
const n = data[i++];
const arr = data.slice(i, i + n); i += n;
console.log(n, arr);   // => n <n numbers as array>
```

**Read a grid of characters (n rows, each a string):**
```js
const lines = require('fs').readFileSync(0, 'utf8').split('\n').filter(l => l.length > 0);
const n = Number(lines[0]);
const grid = lines.slice(1, 1 + n).map(row => row.split(''));  // char[][] equivalent
console.log(grid.length, grid[0]);  // => n [ 'first','row','chars', ... ]
```

**Read until EOF (unknown number of lines, e.g. multiple test cases with no explicit count):**
```js
const all = require('fs').readFileSync(0, 'utf8').split('\n').filter(l => l.length > 0);
for (const line of all) {
  const nums = line.split(/\s+/).map(Number);
  console.log(nums.reduce((a, b) => a + b, 0));  // process each line, e.g. sum
}
```

### 27.9 LeetCode is a different model — no stdin at all

Codeforces/AtCoder-style judges run your **whole file**, and you own stdin/stdout parsing yourself (everything above). LeetCode (and similar function-signature judges) instead call **a function or class method you implement** — there is no `readFileSync`, no `console.log` for the answer; you `return` the result and the harness handles I/O entirely.

```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function (nums, target) {
  const seen = new Map();
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    if (seen.has(need)) return [seen.get(need), i];
    seen.set(nums[i], i);
  }
  return [];
};

console.log(twoSum([2, 7, 11, 15], 9));  // => [ 0, 1 ]
```

| | Codeforces-style | LeetCode-style |
|---|---|---|
| Input | Raw stdin — you parse it | Function arguments — pre-parsed |
| Output | You write to stdout | You `return` a value |
| Entry point | Whole file runs top to bottom | A named function/class method is called |
| Fast I/O matters? | Yes — critical | No — I/O isn't your code's concern |

### 27.10 Node vs browser I/O

There is no `require('fs')`, no `process.stdout`, no stdin at all in a browser JS environment — those are Node.js globals/modules. CP judges run Node (or a similar server-side runtime), so the `fs`/`process` APIs above are always available; don't confuse this with browser-oriented JS (which would use `prompt()`/DOM/`fetch` instead — irrelevant for competitive programming).

---

## 28. Randomness, Time, and the Runtime

### 28.1 `Math.random()` — not seedable

```js
Math.random();   // => a float in [0, 1)  -- different every call, every run
```

> **Gotcha (CP implication):** `Math.random()` cannot be seeded. This means: (1) you cannot reproduce a failing random test case deterministically — log the actual generated values if you need to debug a randomized-testing failure; (2) you cannot defend against anti-hash-test attacks by fixing a seed (some CP setups seed RNG with time-based entropy specifically to defeat adversarial inputs targeting a known-fixed seed — `Math.random()` already varies per run, but you have zero control over *which* sequence you get, so you can't replay one).

### 28.2 Seeded PRNG — mulberry32

When you need a **reproducible** sequence (deterministic stress-testing, seeded shuffles for comparing two algorithms on identical random input), implement a small PRNG:

```js
function mulberry32(seed) {
  return function () {
    seed |= 0; seed = (seed + 0x6D2B79F5) | 0;
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

const rand = mulberry32(12345);
console.log(rand());   // => 0.6270739405881613  (deterministic for seed 12345, same every run)
console.log(rand());   // => 0.0020329536497592926  (next value in the same fixed sequence)
```

Generating a random integer in `[lo, hi]` from either source:
```js
function randInt(rng, lo, hi) {              // rng: a () => [0,1) function, e.g. Math.random or mulberry32(seed)
  return lo + Math.floor(rng() * (hi - lo + 1));
}
console.log(randInt(mulberry32(1), 1, 6));   // => a deterministic value in [1,6] for seed 1
```

### 28.3 Fisher-Yates shuffle

Needed to defend against adversarial worst-case inputs (e.g. an already-sorted array crafted to trigger O(n^2) behavior in a naive quicksort) by randomizing order before processing:

```js
function shuffle(arr, rng = Math.random) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(rng() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

console.log(shuffle([1, 2, 3, 4, 5]));  // => a random permutation, e.g. [ 3, 1, 5, 2, 4 ]
```

> **vs Java/C++:** V8's built-in `Array.prototype.sort` (Node 11+/V8 7.0+) uses **Timsort**, a stable merge-sort/insertion-sort hybrid with O(n log n) worst case — this is much harder to adversarially trigger O(n^2) on than a naive quicksort (older V8 versions, and classic Java `Arrays.sort` for primitives, used quicksort variants that *are* famously hackable with crafted "killer" inputs). So the "shuffle before sorting" defense is less critical in modern Node than it historically was for Java competitive judges — but it's still good practice for **your own** O(n^2) algorithms (e.g. a hand-rolled quicksort, or any algorithm whose complexity depends on input order) where an adversarial/sorted input could still degrade performance.

### 28.4 Timing / benchmarking

```js
// Wall-clock milliseconds, low resolution, fine for coarse timing
const t0 = Date.now();
// ... work ...
console.log(`Date.now diff: ${Date.now() - t0}ms`);

// Sub-millisecond precision, monotonic (immune to system clock adjustments) — best for benchmarking
const p0 = performance.now();
// ... work ...
console.log(`performance.now diff: ${(performance.now() - p0).toFixed(3)}ms`);

// Nanosecond precision via BigInt, Node-specific, most precise
const h0 = process.hrtime.bigint();
// ... work ...
const h1 = process.hrtime.bigint();
console.log(`hrtime diff: ${h1 - h0}ns`);
```

| API | Resolution | Monotonic? | Notes |
|---|---|---|---|
| `Date.now()` | ~1ms | No (can jump if system clock changes) | Simple, portable, fine for rough timing |
| `performance.now()` | sub-ms | Yes | Best general-purpose benchmarking choice |
| `process.hrtime.bigint()` | ns | Yes | Node-only, highest precision, returns BigInt nanoseconds |

### 28.5 `console.error` → stderr (debug prints without corrupting stdout)

```js
console.error('debug: n =', 42);   // writes to stderr, NOT stdout
console.log('42');                 // writes to stdout — this is what the judge actually reads
```

> **CP trick:** most judges compare only your **stdout** against the expected answer; stderr is ignored (or shown separately for debugging). Use `console.error(...)` freely for debug output while iterating locally — it never pollutes the actual answer stream, so you don't have to remember to strip debug prints before submitting (though it's still good hygiene to remove them for a clean final submission).

### 28.6 Node module basics for CP

```js
const fs = require('fs');               // file/stdin access — readFileSync(0, 'utf8') etc.
const readline = require('readline');   // line-based interactive/streamed input
```

These are the only two modules typically needed for CP I/O. `require` (CommonJS) is the default judge-compatible style; avoid ESM `import` syntax unless you've confirmed the judge supports `"type": "module"` — most CP judges run a single `.js` file directly with CommonJS semantics.

### 28.7 Memory / GC note (brief)

Node's V8 heap has a default size limit (historically ~1.5-4GB depending on Node version and flags) — large CP allocations (e.g. a naive 1e7 x 1e7 2D array) can hit `JavaScript heap out of memory` well before a judge's stated memory limit in MB would suggest. Prefer typed arrays (`Int32Array`, `Float64Array` — see Part V/earlier parts) for large numeric buffers: they're more memory-compact and GC-friendlier than arrays of boxed `Number`/generic objects.

### 28.8 Single-file structure for judges

Codeforces-style judges execute your **entire file** top to bottom as the program — global `const`/`let`, function declarations, and the fast-I/O read/process/write blocks all live in one file with no special entry point required (contrast a C++ `main()` — JS just runs the file's top-level statements in order). LeetCode-style judges instead load your file as a module and invoke **one exported function/class method** they specify — no I/O code, no top-level side effects expected (see §27.9).

```js
'use strict';   // optional but recommended at the top of Codeforces-style submissions

const data = require('fs').readFileSync(0, 'utf8').split(/\s+/).map(Number);
let idx = 0;
const next = () => data[idx++];

// ... solve ...
```

`'use strict'` opts the file into strict mode: catches accidental global-variable creation (assigning to an undeclared variable throws instead of silently creating a global — a real footgun source in a long single-file CP submission), disallows some silent-failure patterns, and is the default automatically inside ES modules and classes. For plain CommonJS single-file scripts it's a cheap, worthwhile safety net to add at the top.

---

# PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT

## 29. The Contest Template and Structure

### 29.1 The full Codeforces-style template

Paste this at the top of every CF/AtCoder-style solution. It reads all of stdin at once (fast — one syscall), tokenizes it, and buffers all output into an array joined once at the end (fast — one syscall out).

```js
'use strict';

// ============ FAST I/O ============
const data = require('fs').readFileSync(0, 'utf8');
let ptr = 0;
const tokens = data.split(/\s+/).filter(Boolean); // split on any whitespace/newlines

function next() {
  return tokens[ptr++];
}
function nextInt() {
  return Number(tokens[ptr++]) | 0; // fast int32 cast; use nextSafeInt for > 2^31
}
function nextSafeInt() {
  return parseInt(tokens[ptr++], 10); // safe up to 2^53
}
function nextFloat() {
  return parseFloat(tokens[ptr++]);
}
function nextBigInt() {
  return BigInt(tokens[ptr++]);
}
function nextIntArray(n) {
  const arr = new Array(n);
  for (let i = 0; i < n; i++) arr[i] = nextInt();
  return arr;
}

// ============ OUTPUT BUFFER ============
const out = [];
function print(x) {
  out.push(x);
}

// ============ PASTE DEPENDENCIES HERE ============
// class MinHeap { ... }   <- from Part III, if you need a priority queue
// class Deque { ... }     <- from Part III, if you need O(1) push/pop both ends
// JS ships NEITHER a PriorityQueue NOR a Deque — unlike Java (java.util.PriorityQueue,
// ArrayDeque) and C++ (std::priority_queue, std::deque). Paste the class in, every time.

// ============ DEBUG ============
const DEBUG = false;
function dbg(...args) {
  if (DEBUG) console.error(...args); // stderr: never pollutes the judge's stdout diff
}

// ============ SOLVE ============
function solve() {
  const n = nextInt();
  const arr = nextIntArray(n);
  dbg('n=', n, 'arr=', arr);

  let sum = 0;
  for (let i = 0; i < n; i++) sum += arr[i];

  print(sum);
}

// ============ MAIN ============
function main() {
  let t = nextInt();
  while (t--) solve();
  console.log(out.join('\n'));
}

main();
```

### 29.2 Line-by-line breakdown

| Piece | Why |
|---|---|
| `readFileSync(0, 'utf8')` | Reads all of stdin (fd 0) in one call. Far faster than `readline` interface for CP-sized input (10^5–10^6 tokens). |
| `data.split(/\s+/).filter(Boolean)` | Splits on any run of whitespace (spaces, tabs, `\n`, `\r\n`); `filter(Boolean)` drops the empty string a leading/trailing split produces. |
| `ptr` + `next()` | A manual token pointer — this **is** your `cin >>` / `Scanner.next()`. Every `next*` call just reads `tokens[ptr++]`. |
| `nextInt()` via `| 0` | `| 0` forces ToInt32 — fast, but **wraps** past ±2^31. Use it only when values fit int32. |
| `nextSafeInt()` via `parseInt` | Safe for anything up to 2^53 (Number's exact-integer range). Prefer this unless you've profiled and need the `|0` trick. |
| `nextBigInt()` | For values that can exceed 2^53 (e.g. products before a mod). See Part I for BigInt rules. |
| `out` array + `print()` | Never `console.log` inside a loop (each call is a flush — see §31, §32). Push strings/numbers, join once. |
| `out.join('\n')` | One string, one write. `console.log` auto-adds the trailing newline. |
| `let t = nextInt(); while (t--) solve();` | The standard CF multi-test-case loop. Omit the `t` line entirely for single-case problems. |
| `main()` at the bottom | JS has no special `main` — it's just a function you call. Nothing runs until you call it (top-level `await` aside). |

> **vs C++:** no `ios::sync_with_stdio(false); cin.tie(0);` — there's no equivalent stream to desync, because `readFileSync` never touches a line-buffered stream at all.

> **vs Java:** no `BufferedReader`/`StreamTokenizer` boilerplate — one `readFileSync` + `split` replaces it. But note Java's `Scanner` is the *slow* path there; this JS approach is closer to Java's `BufferedReader`.

### 29.3 LeetCode-style skeletons

LeetCode never gives you stdin — it calls a function (or a class method) with already-parsed arguments and checks the return value. No I/O boilerplate needed at all.

**Function-signature form** (most common — this is what LeetCode's editor scaffolds):

```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function(nums, target) {
  const seen = new Map(); // value -> index
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    if (seen.has(need)) return [seen.get(need), i];
    seen.set(nums[i], i);
  }
  return [];
};
```

**Class/method form** (used when the problem is stateful — e.g. "design a data structure"):

```js
/**
 * @param {number} capacity
 */
var LRUCache = function(capacity) {
  this.capacity = capacity;
  this.map = new Map(); // insertion order == recency order in a JS Map
};

/**
 * @param {number} key
 * @return {number}
 */
LRUCache.prototype.get = function(key) {
  if (!this.map.has(key)) return -1;
  const val = this.map.get(key);
  this.map.delete(key);
  this.map.set(key, val); // re-insert -> now "most recent"
  return val;
};

/**
 * @param {number} key
 * @param {number} value
 * @return {void}
 */
LRUCache.prototype.put = function(key, value) {
  if (this.map.has(key)) this.map.delete(key);
  else if (this.map.size >= this.capacity) {
    this.map.delete(this.map.keys().next().value); // evict oldest (first inserted)
  }
  this.map.set(key, value);
};
```

> **Gotcha:** LeetCode instantiates your class/calls your function directly — do not wrap it in `main()`, do not read stdin, do not `console.log` the answer (just `return` it).

### 29.4 No macros — use functions

C++ has `#define`/templates for one-liners (`#define pb push_back`, `#define all(x) x.begin(),x.end()`); Java has neither macros nor free functions (everything is a static method on some class). **JS has no macro system at all** — same situation as Java, so the fix is the same: small top-level helper functions, pasted into the template as needed.

```js
function gcd(a, b) {
  while (b) [a, b] = [b, a % b];
  return a;
}

function lcm(a, b) {
  return (a / gcd(a, b)) * b;
}

// Lower bound: first index i such that arr[i] >= target (arr sorted ascending)
function lowerBound(arr, target) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {
    const mid = (lo + hi) >>> 1;
    if (arr[mid] < target) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

// Upper bound: first index i such that arr[i] > target
function upperBound(arr, target) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {
    const mid = (lo + hi) >>> 1;
    if (arr[mid] <= target) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}
```

**Heap import-by-paste reminder:** JS gives you neither `std::priority_queue` (C++) nor `java.util.PriorityQueue` (Java). There is no npm-free built-in — you must paste a `MinHeap`/`MaxHeap` class (binary-heap-over-array, `push`/`pop`/`peek` in O(log n)) at the top of every solution that needs one. See Part III for the full implementation; keep a copy in your snippets file so you can paste it in seconds during a contest.

### 29.5 Debug helper

```js
const DEBUG = false; // flip to true locally, ALWAYS false before submitting

function dbg(...args) {
  if (DEBUG) console.error('[DEBUG]', ...args);
}

dbg('checking state', { i: 3, arr: [1, 2, 3] });
```

> **Gotcha:** `console.error` writes to **stderr**, `console.log` writes to **stdout**. Judges only diff stdout, so `console.error` calls are invisible to the grader — safe to leave sprinkled through code (though `DEBUG=false` still avoids the runtime cost and stderr clutter). Never use `console.log` for debug prints; it corrupts your answer stream.

---

## 30. Recursion, the Call Stack, and Iterative Conversion

### 30.1 The stack limit — and why JS is worse here than Java/C++

Node's V8 engine overflows the call stack at roughly **10^4–10^5 frames** (the exact number depends on frame size — how many locals/closures each call captures — so it's not a fixed constant; simple frames survive longer). The error is:

```
RangeError: Maximum call stack size exceeded
```

> **vs Java:** Java lets you spin up a new `Thread` with an explicit large stack size (`new Thread(null, runnable, "name", 1 << 26)`) to dodge this — a common competitive-programming trick for deep recursion. **Node has no equivalent standard escape hatch.** `node --stack-size=65500` exists but is fragile (can segfault instead of throwing cleanly, because the OS thread stack is smaller than what you asked V8 for) and is **usually unavailable on judges** — you cannot pass flags to `node` on Codeforces/LeetCode.

> **vs C++:** `ulimit -s` can be raised, and many judges configure a large stack for C++ specifically. JS gets no such allowance and the default is smaller to begin with.

**Implication:** any DFS/recursion whose depth scales with `n` (a skewed tree, a linked-list-shaped graph, a chain of 10^5 nodes) **will overflow** in JS even though the equivalent C++/Java submission might survive. You must know how to convert recursive traversals to **iterative, explicit-stack** versions on sight.

### 30.2 Worked example: recursive DFS → iterative DFS (graph)

Recursive (breaks on deep/skewed graphs):

```js
function dfsRecursive(graph, start) {
  const visited = new Set();
  const order = [];

  function visit(u) {
    if (visited.has(u)) return;
    visited.add(u);
    order.push(u);
    for (const v of graph[u]) visit(v); // <-- unbounded recursion depth
  }

  visit(start);
  return order;
}
```

Iterative, explicit stack (safe at any depth, bounded by heap memory not call-stack frames):

```js
function dfsIterative(graph, start) {
  const visited = new Set([start]);
  const order = [];
  const stack = [start]; // explicit stack replaces the call stack

  while (stack.length) {
    const u = stack.pop();
    order.push(u);
    // push neighbors in reverse to visit them in the same order as the recursive version
    const neighbors = graph[u];
    for (let i = neighbors.length - 1; i >= 0; i--) {
      const v = neighbors[i];
      if (!visited.has(v)) {
        visited.add(v);
        stack.push(v);
      }
    }
  }
  return order;
}
```

### 30.3 Worked example: recursive tree traversal → iterative (with post-order work)

Pre-order is trivial to convert (push children, pop, visit — as above). Post-order is the hard case because you need to do work **after** both children return — that's the "post-recursion work" a call stack gives you for free and an explicit stack must simulate.

**Technique: push a frame with a phase marker.**

```js
// Recursive post-order (breaks past ~10^4-10^5 depth on a skewed tree)
function postorderRecursive(root) {
  const result = [];
  function visit(node) {
    if (!node) return;
    visit(node.left);
    visit(node.right);
    result.push(node.val); // post-recursion work
  }
  visit(root);
  return result;
}

// Iterative post-order: simulate the call stack with [node, phase] frames.
// phase 0 = "just arrived, haven't recursed yet"; phase 1 = "children done, do the work"
function postorderIterative(root) {
  const result = [];
  if (!root) return result;
  const stack = [[root, 0]];

  while (stack.length) {
    const frame = stack[stack.length - 1]; // peek, don't pop yet
    const [node, phase] = frame;

    if (phase === 0) {
      frame[1] = 1; // next time we see this frame, do the "after children" work
      if (node.right) stack.push([node.right, 0]);
      if (node.left) stack.push([node.left, 0]);
    } else {
      stack.pop();
      result.push(node.val); // this is the post-recursion work
    }
  }
  return result;
}
```

This "frame + phase" pattern generalizes to **any** recursion with pre- and post-work: each stack entry is `[state, phase]` (or a small object), and the loop dispatches on `phase` instead of relying on the JS call stack to resume where it left off. For a DFS that needs "mark visited on entry, do cleanup/backtracking on exit" (e.g. cycle detection with a `visiting`/`visited` two-color scheme, or computing subtree sizes), use the same shape:

```js
// Iterative DFS with entry/exit work (e.g. subtree size computation)
function subtreeSizes(graph, root) {
  const size = new Map();
  const stack = [[root, -1, 0]]; // [node, parent, phase]

  while (stack.length) {
    const frame = stack[stack.length - 1];
    const [u, parent, phase] = frame;

    if (phase === 0) {
      size.set(u, 1);
      frame[2] = 1;
      for (const v of graph[u]) {
        if (v !== parent) stack.push([v, u, 0]);
      }
    } else {
      stack.pop();
      if (parent !== -1) {
        size.set(parent, size.get(parent) + size.get(u)); // post-recursion aggregation
      }
    }
  }
  return size;
}
```

### 30.4 Tail-call optimization: don't rely on it

ES2015 specifies **proper tail calls** (a self-tail-call shouldn't grow the stack), but **V8 never shipped it** (it was implemented briefly behind a flag in old Safari/JSC only, and even that was later reverted). In practice:

> **Gotcha:** writing "tail-recursive" JS to dodge the stack limit does **nothing** in Node. There is no compiler guarantee — assume every recursive call, tail position or not, consumes a stack frame.

**Trampolining** is a manual workaround: instead of calling the next step recursively, return a *thunk* (a closure describing the next step), and drive the loop from outside.

```js
// Trampoline pattern: turns unbounded recursion into a flat loop
function trampoline(fn) {
  let result = fn();
  while (typeof result === 'function') result = result();
  return result;
}

function factorial(n, acc = 1n) {
  if (n <= 1) return acc;
  return () => factorial(n - 1, acc * BigInt(n)); // return a thunk, not a recursive call
}

trampoline(() => factorial(100000n)); // safe at any depth — no stack growth
```

Trampolining is rarely worth the complexity in a contest — reach for the explicit-stack pattern (§30.2/30.3) first; trampolining is really a functional-programming technique that fits better for simple linear recursions (factorial-shaped) than for tree/graph traversals with branching.

### 30.5 When recursion is fine vs when you must go iterative

| Situation | Max depth | Verdict |
|---|---|---|
| Balanced binary tree, `n ≤ 10^5` | `~log2(n) ≈ 17` | Recursion is fine |
| Balanced BST/segment-tree recursion | `O(log n)` | Recursion is fine |
| Small/bounded `n` (`n ≤ 10^3`–`10^4`) DFS/backtracking | up to `n` | Usually fine, but test |
| Linked list, `n` up to `10^5` | `O(n)` | **Convert to iterative** |
| Skewed/degenerate BST (adversarial input) | `O(n)` | **Convert to iterative** |
| Graph DFS on a general/adversarial graph, `n ≥ 10^4` | up to `O(n)` | **Convert to iterative** |
| Divide-and-conquer that halves each call (merge sort, quicksort avg case) | `O(log n)` | Recursion is fine |

Rule of thumb: if recursion depth is **provably `O(log n)`**, keep the recursive version — it's clearer and V8 handles it fine. If depth can be **`O(n)`** and `n` can exceed roughly `10^4`, convert before you submit, not after a RangeError wastes a submission.

### 30.6 Memoization (top-down DP) and its stack-depth risk

```js
function fib(n, memo = new Map()) {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n);
  const result = fib(n - 1, memo) + fib(n - 2, memo);
  memo.set(n, result);
  return result;
}
```

- A plain object also works as the memo table (`memo[n] ?? (memo[n] = ...)`), but a `Map` avoids key-stringification and prototype-pollution surprises (see Part I/§31 for object vs Map).
- **Stack-depth risk is unchanged by memoization** — memoizing avoids *recomputation*, not recursion depth. `fib(100000)` still recurses 100,000 frames deep the first time each value is computed, and will still overflow. For DP problems where the recursion depth tracks `n` directly (linear-chain DP), either convert to the iterative bottom-up form (almost always possible for DP — process states in dependency order with a `for` loop and an array) or use the explicit-stack pattern from §30.3.
- Bottom-up (tabulation) sidesteps the whole problem and is usually preferred in JS/CP for exactly this reason:

```js
function fibIterative(n) {
  const dp = new Array(n + 1);
  dp[0] = 0; dp[1] = 1;
  for (let i = 2; i <= n; i++) dp[i] = dp[i - 1] + dp[i - 2];
  return dp[n];
}
```

---

## 31. Performance and V8 Gotchas

JS in Node is roughly **2–4x slower than C++** for typical CP workloads (sometimes closer, sometimes worse depending on how idiomatic the code is). Codeforces gives JS the same time limit as C++ on most problems and does **not always** grant extra time — check the problem's language-specific limits before assuming you have headroom. The rules below are what separates "comfortably inside the limit" from "TLE" in V8.

### 31.1 Keep arrays monomorphic and dense

V8 internally tags every array with an **elements kind**: `PACKED_SMI` (small integers, no holes) → `PACKED_DOUBLE` → `PACKED_ELEMENTS` (mixed/objects) → the `HOLEY_*` variants (once any hole appears). Operations get **strictly slower** as an array transitions down this chain, and the transition is **one-way** — an array never gets faster again.

```js
// GOOD: stays PACKED_SMI the whole time — fast
const a = new Array(n).fill(0);
for (let i = 0; i < n; i++) a[i] = i * 2;

// BAD: mixes number and string types -> demotes to PACKED_ELEMENTS (generic, slow)
const b = [];
b.push(1);
b.push('2'); // now every element is boxed/generic

// BAD: creates a hole -> demotes to HOLEY_* permanently for this array
const c = [1, 2, 3];
delete c[1]; // c is now [1, <hole>, 3] — HOLEY_SMI, slower iteration forever

// BAD: sparse assignment creates holes from the start
const d = [];
d[5] = 1; // indices 0-4 are holes
```

> **Gotcha:** pre-size and fill arrays (`new Array(n).fill(0)`) rather than growing them with sparse writes, and never mix value types in one numeric array. Keep an array's contents one consistent type (all numbers, ideally all small integers) for its entire lifetime.

### 31.2 Use typed arrays for large numeric data

`Int32Array`, `Float64Array`, etc. store raw fixed-width values contiguously — no boxing, no elements-kind transitions possible, and much less GC pressure than a regular `Array` of numbers. See Part II for the full typed-array reference; the CP-relevant takeaway:

```js
// Regular array of 10^6 numbers: boxed doubles, more GC churn
const arr = new Array(1_000_000).fill(0);

// Typed array: fixed-width, contiguous, no boxing — meaningfully faster for
// large numeric buffers (adjacency lists' flattened form, DP tables, sieve arrays)
const fast = new Int32Array(1_000_000); // auto-zeroed
```

Use typed arrays for: sieve-of-Eratosthenes boolean/int arrays, large DP tables of fixed-width integers, adjacency lists flattened into CSR form, frequency-count arrays over a bounded alphabet.

### 31.3 Avoid `delete`

```js
const obj = { x: 1, y: 2, z: 3 };
delete obj.y; // BAD: deoptimizes obj's hidden class, forces a slower dictionary mode

obj.y = undefined; // GOOD if you just need "no value" — hidden class stays stable
// or use a Map if keys are added/removed dynamically and frequently
const m = new Map([['x', 1], ['y', 2], ['z', 3]]);
m.delete('y'); // fine — Map is designed for this
```

### 31.4 Avoid the `shift()`/`unshift()` O(n) trap

`Array.prototype.shift()`/`unshift()` re-index every remaining element — **O(n) per call**, so a loop that shifts `n` times is **O(n^2)**, a classic silent TLE. Use a head-index queue instead (full implementation in Part III):

```js
// BAD: O(n^2) total — shift() re-indexes the whole array every call
const q = [1, 2, 3, /* ... */];
while (q.length) {
  const x = q.shift(); // O(n) each call
}

// GOOD: O(1) amortized dequeue via a head pointer
const q2 = [1, 2, 3 /* ... */];
let head = 0;
while (head < q2.length) {
  const x = q2[head++]; // O(1)
}
```

### 31.5 Minimize allocation in hot loops

Chained higher-order methods (`map`/`filter`/`reduce`) each allocate a new intermediate array. Fine outside hot paths; costly inside an `O(n)`-times-repeated inner loop.

```js
// BAD inside a hot loop: 3 intermediate arrays allocated every call
function hot(arr) {
  return arr.map(x => x * 2).filter(x => x > 10).reduce((a, b) => a + b, 0);
}

// GOOD: one pass, zero intermediate allocation
function hotFast(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    const doubled = arr[i] * 2;
    if (doubled > 10) sum += doubled;
  }
  return sum;
}
```

Also reuse scratch arrays/buffers across iterations of an outer loop instead of allocating a fresh one each time (e.g. a `visited` array reset with `.fill(false)` rather than `new Array(n).fill(false)` recreated every BFS call in a multi-test-case problem).

### 31.6 Map vs plain object for hot lookups

| | `Map` | Plain object |
|---|---|---|
| Non-string keys (numbers, objects) | Native, no coercion | Coerced to string keys |
| Frequent add/delete | Optimized for this | Can deoptimize (see `delete` above) |
| Iteration order | Guaranteed insertion order | Guaranteed since ES2015, but integer-like keys sort numerically first (surprising) |
| Raw property access, fixed/stable shape | — | Very fast (hidden-class monomorphic access) |
| `.size` / `Object.keys(obj).length` | O(1) `.size` | O(n) to count |

Use `Map` for hot add/delete cycles and non-string keys (graph adjacency by node id, frequency counters). Use a plain object only when keys are strings and the shape is stable (e.g. a fixed-field record).

### 31.7 String building: never `+=` in a loop

```js
// BAD: O(n^2) — each += may reallocate/copy the growing string
let s = '';
for (let i = 0; i < n; i++) s += chars[i];

// GOOD: push to an array, join once — O(n)
const parts = [];
for (let i = 0; i < n; i++) parts.push(chars[i]);
const s2 = parts.join('');
```

### 31.8 `console.log` buffering

Covered fully in Part VI — the short version: **never** call `console.log` inside a loop over `n` items; each call is a synchronous flush. Push to the `out` array (§29.1) and `console.log(out.join('\n'))` exactly once.

### 31.9 Hidden classes — initialize properties in a consistent order

V8 assigns every object a **hidden class** based on the set and *order* of its properties. Two objects built with properties added in different orders get different hidden classes even if they end up with the same keys — this defeats inline-cache optimizations for functions that operate on "the same kind" of object.

```js
// BAD: inconsistent construction order -> different hidden classes for p1 vs p2
function makePointBad(x, y, tag) {
  const p = {};
  if (tag) p.tag = tag; // sometimes added, sometimes not, sometimes first sometimes never
  p.x = x;
  p.y = y;
  return p;
}

// GOOD: always the same properties in the same order -> one stable hidden class
function makePointGood(x, y, tag = null) {
  return { x, y, tag };
}

// BEST for CP: a class gives every instance the same hidden class by construction
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}
```

Prefer classes (or object literals with all fields present, even if `null`) over incrementally building objects with conditional property adds, especially inside hot loops (e.g. constructing millions of graph-edge objects).

### 31.10 Cache `.length` and repeated property lookups in tight loops

```js
// Slightly slower: arr.length and obj.field re-read every iteration
for (let i = 0; i < arr.length; i++) { /* ... */ }

// Slightly faster in hot loops: cache the length
for (let i = 0, n = arr.length; i < n; i++) { /* ... */ }

// Caching a repeated property/nested lookup also helps in very hot inner loops
const g = graph[u]; // instead of re-reading graph[u] inside the loop below
for (let i = 0; i < g.length; i++) {
  const v = g[i];
  // ...
}
```

Modern V8 often optimizes simple `.length` re-reads away, so this matters most for genuinely hot inner loops (millions+ of iterations), not everywhere — don't over-apply it at the cost of readability.

### 31.11 Complexity rule-of-thumb, adjusted for JS's constant factor

C++ handles roughly `10^8`–`10^9` simple operations/sec; JS in Node realistically handles **`10^7`–`10^8`**. Use this table to sanity-check an approach against `n` and a ~1-2 second limit:

| `n` | Safe complexity in JS (≈1-2s) | Notes |
|---|---|---|
| ≤ 10 | up to `O(n!)`, `O(2^n · n)` | brute force/permutations fine |
| ≤ 20 | `O(2^n)` | bitmask DP territory |
| ≤ 500 | `O(n^3)` | Floyd-Warshall etc. |
| ≤ 5,000 | `O(n^2 log n)` | |
| ≤ 10^4–10^5 | `O(n^2)` only near the low end; prefer `O(n log n)` | `n^2` at `10^5` is `10^10` — too slow |
| ≤ 10^6 | `O(n log n)`, `O(n)` | typed arrays help a lot here |
| ≤ 10^7–10^8 | `O(n)` with a very light constant, or typed arrays | approach the JS ops/sec ceiling |

Treat these as roughly **one order of magnitude tighter** than the equivalent C++ guidance — if an `O(n^2)` approach is borderline in C++ at some `n`, assume it's too slow in JS at that same `n`.

---

## 32. Debugging, Errors and Running

### 32.1 Reading a JS stack trace

```
TypeError: Cannot read properties of undefined (reading 'val')
    at dfs (/home/user/sol.js:12:18)
    at dfs (/home/user/sol.js:15:10)
    at solve (/home/user/sol.js:23:3)
    at main (/home/user/sol.js:30:3)
    at Object.<anonymous> (/home/user/sol.js:33:1)
```

Read it as: **line 1** = error type + message (what went wrong); **subsequent lines** = the call stack, **top frame first** — `dfs` at `sol.js:12:18` is where the crash actually happened, and each line below it is who called whom, ending at your `main()`/module top level. Always look at the **first** (topmost) frame in *your own file* first — that's the actual failure site; frames from deep inside Node internals or when the stack is huge from deep recursion are usually less informative (and with recursion, you'll see the same 2-3 lines repeated `dfs`/`dfs`/`dfs`... all the way up).

### 32.2 Common runtime errors, decoded

| Error | Typical cause | Example / fix |
|---|---|---|
| `TypeError: Cannot read properties of undefined (reading 'x')` | Accessing a property on `undefined`/`null` — **the single most common JS runtime error**. Usually an out-of-bounds array access (`arr[i]` past the end returns `undefined`, not a crash — the crash happens on the *next* `.property` access), a missing map/object entry, or a function that forgot to `return`. | `const node = map.get(key); node.val` when `key` isn't in `map` → `node` is `undefined`. Fix: check `map.has(key)` first, or use `map.get(key)?.val`. |
| `TypeError: x is not a function` | Calling something that isn't callable — typo'd method name, calling an array where you meant a function, shadowed variable. | `arr.lenght()` (typo) |
| `ReferenceError: x is not defined` | Using an undeclared variable, or typo'd identifier. | Forgot `const`/`let`, or misspelled a variable name. |
| `ReferenceError: Cannot access 'x' before initialization` | Temporal Dead Zone — reading a `let`/`const` before its declaration line runs (see Part I). | Common when a function used earlier in the file references a `const` declared later, and gets called too early. |
| `RangeError: Maximum call stack size exceeded` | Recursion too deep — see §30. | Convert to iterative with an explicit stack. |
| `RangeError: Invalid array length` | `new Array(n)` with `n` negative or non-integer (e.g. computed from a bad subtraction). | Guard array-length computations (`Math.max(0, n)`), check for `NaN`. |
| `SyntaxError: Unexpected token` | Malformed code — mismatched brackets, missing comma, stray character (often from copy-pasting between C++/Java syntax). | Check the file/line the error names; often one line above where it's reported. |
| `TypeError: Cannot mix BigInt and other types` | Using `+`, `*`, `<` etc. between a `BigInt` and a `Number` directly. | `5n + 3` throws. Fix: `5n + BigInt(3)`, or convert consistently — see Part I. |

### 32.3 Common-bug checklist for DSA/CP in JS

Run down this list before assuming a bug is "logic," not "JS semantics."

| Bug | Symptom | Fix |
|---|---|---|
| `sort()` with no comparator | `[10, 2, 1].sort()` → `[1, 10, 2]` (lexicographic!) | Always pass a comparator: `arr.sort((a, b) => a - b)` |
| Comparator returns boolean, not number | Silently wrong/unstable sort order — **classic trap** | `(a, b) => a > b` is **WRONG**. Use `(a, b) => a - b` (ascending) |
| `shift()`/`unshift()` in a loop | TLE, not wrong-answer | Use a head-index queue (§31.4, Part III) |
| `console.log` inside a per-element loop | TLE from I/O flushing | Buffer to an array, join+log once (§29.1, §31.8) |
| Bitwise ops beyond 2^31 | Silent truncation/wraparound — bitwise operators coerce to **int32** | Use `BigInt` or avoid bitwise ops on values that can exceed ±2^31 |
| Precision loss beyond 2^53 (e.g. mod-mul) | Wrong numeric answers with no error thrown | Use `BigInt` for products before a modulo when operands can be large |
| `Array(n).fill([])` | All `n` "separate" sub-arrays are actually **the same array** (shared reference) | `Array.from({length: n}, () => [])` |
| `new Array(n)` | Creates `n` **holes**, not zeros — `undefined` on read, and holey elements-kind (slow, see §31.1) | `new Array(n).fill(0)` |
| `==` vs `===` | Type-coercing comparisons produce surprises (`'' == 0` is `true`, `null == undefined` is `true`) | Always use `===`/`!==` unless coercion is deliberate |
| `for...in` over an array | Iterates **string keys** (including inherited/enumerable ones), not guaranteed numeric order, and includes non-index properties | Use `for (let i = 0; i < arr.length; i++)` or `for...of` |
| Floating-point compare with `===` | `0.1 + 0.2 === 0.3` is `false` | Compare with an epsilon: `Math.abs(a - b) < 1e-9` |
| Forgetting `.sort()` mutates | Using `arr.sort()` when you needed the original order preserved elsewhere too | Copy first: `[...arr].sort(...)` (or `arr.toSorted(...)` in Node 20+) |
| Object key string-coercion | Using a non-string/non-symbol as an object key silently `.toString()`s it — `obj[{}]` and `obj[[1,2]]` can collide | Use a `Map` for non-string keys |
| `Math.max(...hugeArray)` | `RangeError` from spreading too many arguments onto the call stack (not the "recursion" RangeError, but same root cause: stack limit) | Reduce manually: `arr.reduce((a, b) => Math.max(a, b))`, or chunk the spread |
| Recursion depth | `RangeError: Maximum call stack size exceeded` on deep input | See §30 — convert to iterative |
| `arr[-1]` | Returns `undefined`, **not** the last element (unlike Python) | Use `arr[arr.length - 1]`, or `arr.at(-1)` |
| Off-by-one in hand-rolled binary search | Infinite loop or wrong boundary — `<=` vs `<`, `mid` vs `mid+1`/`mid-1` | Use the `lowerBound`/`upperBound` templates from §29.4 verbatim rather than re-deriving under time pressure |

### 32.4 Running locally

```bash
# Pipe a file in as stdin, matching how the judge invokes your program
node sol.js < input.txt

# Interactively type input (Ctrl+D / Ctrl+Z to signal EOF)
node sol.js

# Pipe from another command
echo "3
1 2 3" | node sol.js

# Increase V8's stack size (fragile: can segfault instead of a clean RangeError;
# NOT available as an option on most judges — local debugging only)
node --stack-size=65500 sol.js < input.txt

# Attach the inspector for real breakpoint debugging (Chrome DevTools / VS Code)
node --inspect-brk sol.js < input.txt
```

> **Gotcha:** `--stack-size` raises what V8 *thinks* its limit is, but the underlying OS thread stack is a separate, smaller allocation — set it too high and you get a hard segfault instead of a catchable `RangeError`. Treat it as a rough local debugging aid, never a fix to ship.

### 32.5 A tiny stress-testing harness

The classic CP technique: generate random small inputs, run a slow-but-obviously-correct brute force alongside your fast solution, and diff their outputs in a loop until they disagree.

```js
// gen.js — random small test generator
const n = 1 + Math.floor(Math.random() * 8);
const lines = [String(n)];
const arr = Array.from({ length: n }, () => 1 + Math.floor(Math.random() * 20));
lines.push(arr.join(' '));
console.log(lines.join('\n'));
```

```js
// brute.js and fast.js: two solutions to the same problem, same I/O contract as §29.1
```

```bash
# stress.sh — regenerate, run both, diff; stop at the first mismatch
for i in $(seq 1 1000); do
  node gen.js > tmp_in.txt
  node brute.js < tmp_in.txt > tmp_brute.txt
  node fast.js  < tmp_in.txt > tmp_fast.txt
  if ! diff -q tmp_brute.txt tmp_fast.txt > /dev/null; then
    echo "MISMATCH on iteration $i, input:"
    cat tmp_in.txt
    echo "brute:"; cat tmp_brute.txt
    echo "fast:";  cat tmp_fast.txt
    break
  fi
done
echo "done"
```

Run with `bash stress.sh`. This finds edge cases (empty input, duplicate values, boundary sizes) far faster than manual reasoning once a solution "looks right but WAs."

### 32.6 `'use strict'` and `console.assert`

```js
'use strict'; // put at the very top of the file (or omit — ES modules are strict by default)
// Strict mode: undeclared-variable assignment throws instead of silently creating a
// global, duplicate parameter names are a SyntaxError, `this` is undefined instead of
// the global object in a plain function call. Catches real bugs earlier.

x = 5; // 'use strict' turns this typo (missing let/const) into a ReferenceError instead
       // of silently creating a global `x` — exactly the class of bug that wastes contest time
```

```js
console.assert(2 + 2 === 4, 'math is broken'); // logs the message to stderr ONLY if the condition is false
console.assert(1 === 2, 'this fires'); // Assertion failed: this fires
```

> **Gotcha:** `console.assert` does **not** throw and does **not** stop execution — unlike `assert()` in C++ or Java's `assert` keyword, a failed `console.assert` just logs to stderr and the program keeps running. Use it for sanity checks you want visibility into locally, not as a correctness guard the judge will "catch" — a wrong answer with a failed assertion still gets submitted and scored (though since it's stderr, it never affects the judge's stdout diff either way).

---

# PART VIII — QUICK REFERENCE AND RECIPES

## 33. The "How Do I...?" Recipe Index

Task → exact snippet. Grouped. Baseline Node ES2022. Multi-line helpers are pulled out under their table as standalone blocks — paste directly.

### Sorting

| Task | Code |
|---|---|
| Sort numbers ascending | `a.sort((x,y)=>x-y)` |
| Sort numbers descending | `a.sort((x,y)=>y-x)` |
| Sort strings (default is fine, lexicographic) | `a.sort()` |
| Sort objects by one numeric key | `a.sort((x,y)=>x.k-y.k)` |
| Sort objects multi-key (asc k1, then asc k2) | `a.sort((x,y)=>x.k1-y.k1 \|\| x.k2-y.k2)` |
| Sort multi-key mixed directions (asc k1, desc k2) | `a.sort((x,y)=>x.k1-y.k1 \|\| y.k2-x.k2)` |
| Argsort — indices ordered by value | `const idx=[...a.keys()].sort((i,j)=>a[i]-a[j])` |
| Sort by string key, locale-correct | `a.sort((x,y)=>x.s.localeCompare(y.s))` |
| Sort descending, strings | `a.sort((x,y)=>y.localeCompare(x))` |
| Stable sort | native `.sort` is stable since ES2019 — rely on it for tie-preserving multi-pass sorts |
| Sort a copy, keep original order intact | `const sorted=[...a].sort((x,y)=>x-y)` |

### Dedupe / Reverse

| Task | Code |
|---|---|
| Dedupe array (primitives) | `[...new Set(a)]` |
| Dedupe preserving one object per key | `[...new Map(a.map(o=>[o.id,o])).values()]` |
| Count distinct elements | `new Set(a).size` |
| Reverse array (mutates) | `a.reverse()` |
| Reverse array (copy, original intact) | `[...a].reverse()` |
| Reverse string | `[...s].reverse().join('')` |
| Rotate array left by k | `const k2=k%a.length; a.push(...a.splice(0,k2))` |
| Flatten a nested array (one level) | `a.flat()` |
| Flatten fully | `a.flat(Infinity)` |
| Chunk array into size-k pieces | `Array.from({length:Math.ceil(a.length/k)},(_,i)=>a.slice(i*k,i*k+k))` |
| Zip two arrays | `a.map((x,i)=>[x,b[i]])` |
| Range array `[0..n-1]` | `Array.from({length:n},(_,i)=>i)` |
| Range array `[lo..hi]` inclusive | `Array.from({length:hi-lo+1},(_,i)=>lo+i)` |

### Max / Min

| Task | Code |
|---|---|
| Max of array (small/medium, n not huge) | `Math.max(...a)` |
| Max of array (large — avoid call-stack overflow, see §36 #18) | `a.reduce((m,x)=>x>m?x:m,-Infinity)` |
| Min of array | `Math.min(...a)` or reduce with seed `Infinity` |
| Index of max | `a.reduce((bi,x,i)=>x>a[bi]?i:bi,0)` |
| Kth largest, k queries repeated / k small vs large n | maintain a min-heap of size k, push then pop when `size>k`; root = kth largest — class in §15 |
| Kth largest, one-off query | `[...a].sort((x,y)=>y-x)[k-1]` — O(n log n), fine for a single query |
| Second largest distinct value | `[...new Set(a)].sort((x,y)=>y-x)[1]` |

### Sum / Precision

| Task | Code |
|---|---|
| Sum, values stay in safe range | `a.reduce((s,x)=>s+x,0)` |
| Sum that may exceed 2^53 | `a.reduce((s,x)=>s+BigInt(x),0n)` |
| Check whether a value needs BigInt | `Number.isSafeInteger(v)` — `false` means switch to BigInt |
| Running prefix-sum array | `const pre=[0]; for(const x of a) pre.push(pre[pre.length-1]+x);` |
| Range sum `[l,r]` from prefix array | `pre[r+1]-pre[l]` |

### Frequency / Maps

| Task | Code |
|---|---|
| Frequency map (Map, any key type) | `const f=new Map(); for(const x of a) f.set(x,(f.get(x)\|\|0)+1);` |
| Frequency map (plain object, string/int keys only) | `const f={}; for(const x of a) f[x]=(f[x]\|\|0)+1;` |
| Iterate a Map (entries) | `for(const [k,v] of m) {}` |
| Iterate a Map (keys only) | `for(const k of m.keys()) {}` |
| Iterate a Map sorted by key | `[...m.entries()].sort((a,b)=>a[0]-b[0])` |
| Most frequent element | `[...f.entries()].reduce((b,e)=>e[1]>b[1]?e:b)[0]` |
| Map → object | `Object.fromEntries(m)` |
| Object → Map | `new Map(Object.entries(obj))` |
| Map → array of `[k,v]` pairs | `[...m]` or `[...m.entries()]` |
| Array of pairs → Map | `new Map(pairs)` |
| Object → array of values / keys | `Object.values(obj)` / `Object.keys(obj)` |
| Check key exists (Map vs object) | `m.has(k)` vs `k in obj` (not `obj[k]!==undefined` — that misses a stored `undefined`) |

### Adjacency List / Grid Init

| Task | Code |
|---|---|
| Adjacency list, n nodes | `const g=Array.from({length:n},()=>[]);` |
| Weighted adjacency list | `const g=Array.from({length:n},()=>[]); g[u].push([v,w]);` |
| Undirected edge | `g[u].push(v); g[v].push(u);` |
| 2D array init (correct, independent rows) | `Array.from({length:n},()=>Array(m).fill(0))` |
| 2D array init — **shared-ref trap, do not use** | `Array(n).fill(Array(m).fill(0))` — every row is the *same* array object; mutating one mutates all |
| 3D array init | `Array.from({length:n},()=>Array.from({length:m},()=>Array(k).fill(0)))` |
| 2D array deep copy | `grid.map(row=>[...row])` |
| Matrix transpose | `matrix[0].map((_,c)=>matrix.map(row=>row[c]))` |
| Direction arrays, 4-dir (N,E,S,W) | `const dx=[-1,0,1,0], dy=[0,1,0,-1];` |
| Direction arrays, 8-dir | `const dx=[-1,-1,-1,0,0,1,1,1], dy=[-1,0,1,-1,1,-1,0,1];` |
| Use direction arrays in a loop | `for(let d=0; d<4; d++){ const nx=x+dx[d], ny=y+dy[d]; }` |
| In-bounds check | `nx>=0 && nx<n && ny>=0 && ny<m` |

### Strings

| Task | Code |
|---|---|
| String → number (strict) | `Number(s)` or `+s` — returns `NaN` on junk |
| String → number (leading-digits, lenient) | `parseInt(s,10)` — stops at first non-digit, ignores trailing junk |
| Number → string | `String(n)` or `` `${n}` `` |
| Split by a literal char | `s.split(',')` |
| Split on whitespace (any run, trims first) | `s.trim().split(/\s+/)` |
| Split into chars (Unicode-safe, surrogate pairs) | `[...s]` — `s.split('')` breaks multi-byte chars |
| **Split trap** — empty string | `''.split(',')` → `['']`, not `[]`. Guard: `s? s.split(','):[]` |
| Join array into string | `a.join(' ')` |
| char → char code | `s.charCodeAt(i)` |
| char code → char | `String.fromCharCode(code)` |
| `char - 'a'` idiom | `s.charCodeAt(i) - 97` |
| index → char idiom | `String.fromCharCode(97 + i)` |
| Uppercase / lowercase | `s.toUpperCase()` / `s.toLowerCase()` |
| Reverse a string | `[...s].reverse().join('')` |
| Check palindrome | `const r=[...s].reverse().join(''); s===r` |
| `slice` vs `substring` with negative index | `slice(-2)` → last 2 chars. `substring(-2)` clamps negative to `0` → whole string. **Prefer `slice`.** |
| `indexOf` with not-found guard | `const i=s.indexOf('x'); if(i===-1){ /* not found */ }` |
| Build a string efficiently (avoid `+=` in a hot loop) | `const parts=[]; parts.push(x); /* ... */ parts.join('')` |
| Check substring exists | `s.includes('ab')` |
| Trim whitespace | `s.trim()` / `s.trimStart()` / `s.trimEnd()` |
| Pad to fixed width | `s.padStart(5,'0')` / `s.padEnd(5,' ')` |
| Repeat | `'ab'.repeat(3)` → `'ababab'` |
| Check anagram | `[...s1].sort().join('')===[...s2].sort().join('')` |
| Count char occurrences | `[...s].filter(c=>c==='a').length` |

### Combinatorics / Recursion

| Task | Approach |
|---|---|
| All permutations | recursive backtrack — swap-in-place or a `used[]` array; see §30/§7 |
| All subsets, n ≤ 30 | bitmask loop over `0..(1<<n)-1` |
| All subsets, n > 30 | recursion (include/exclude at each index) — a `number` bitmask loses safe precision past 2^31-ish usable bits |
| Iterate submasks of `mask` | `for(let sub=mask; sub>0; sub=(sub-1)&mask){ /* use sub */ }` — misses `sub=0`; add an explicit check first if the empty submask matters |

```js
// bitmask subset enumeration, n <= ~20
for (let mask = 0; mask < (1 << n); mask++) {
  const subset = [];
  for (let i = 0; i < n; i++) if (mask & (1 << i)) subset.push(a[i]);
  // process subset
}
```

### Binary Search

| Task | One-liner / call |
|---|---|
| First index with `a[i] >= target` | `lowerBound(a, t)` |
| First index with `a[i] > target` | `upperBound(a, t)` |
| Floor value (largest `a[i] <= t`, sorted array, no TreeMap) | `const i=lowerBound(a,t); const floor=(i<a.length && a[i]===t)?a[i]:a[i-1];` — watch `i-1` going out of bounds |
| Ceiling value (smallest `a[i] >= t`) | `a[lowerBound(a, t)]` — `undefined` if none exists |
| Count of values in `[l, r]` | `upperBound(a, r) - lowerBound(a, l)` |
| Does target exist | `const i=lowerBound(a,t); i<a.length && a[i]===t` |

```js
function lowerBound(a, t) {          // first index with a[i] >= t
  let lo = 0, hi = a.length;
  while (lo < hi) {
    const mid = (lo + hi) >>> 1;
    if (a[mid] < t) lo = mid + 1; else hi = mid;
  }
  return lo;
}

function upperBound(a, t) {          // first index with a[i] > t
  let lo = 0, hi = a.length;
  while (lo < hi) {
    const mid = (lo + hi) >>> 1;
    if (a[mid] <= t) lo = mid + 1; else hi = mid;
  }
  return lo;
}
```

### Math / Number Theory

| Task | Call |
|---|---|
| gcd | `gcd(a, b)` |
| lcm | `lcm(a, b)` |
| Fast modpow, avoids overflow | `modpow(base, exp, mod)` — all BigInt args |
| Modular multiply, operands may exceed 2^53 | `(BigInt(a) * BigInt(b)) % BigInt(m)` |
| Safe positive modulo (handles negative `a`) | `((a % m) + m) % m` |
| Integer sqrt, float-error corrected | `isqrt(n)` |
| Check power of two | `n > 0 && (n & (n - 1)) === 0` |
| Count set bits (no built-in popcount) | `popcount32(x)` |

```js
function gcd(a, b) { while (b) { [a, b] = [b, a % b]; } return a; }   // iterative — avoids recursion-depth issues
function lcm(a, b) { return a / gcd(a, b) * b; }

function modpow(base, exp, mod) {    // base, exp, mod all BigInt
  base %= mod;
  let r = 1n;
  while (exp > 0n) {
    if (exp & 1n) r = r * base % mod;
    base = base * base % mod;
    exp >>= 1n;
  }
  return r;
}

function isqrt(n) {
  let r = Math.floor(Math.sqrt(n));
  while (r * r > n) r--;
  while ((r + 1) * (r + 1) <= n) r++;
  return r;
}

function popcount32(x) {
  x = x - ((x >>> 1) & 0x55555555);
  x = (x & 0x33333333) + ((x >>> 2) & 0x33333333);
  x = (x + (x >>> 4)) & 0x0f0f0f0f;
  return (x * 0x01010101) >>> 24;
}
```

### Misc Ops

| Task | Code |
|---|---|
| Swap two variables | `[a,b]=[b,a]` |
| Swap array elements | `[a[i],a[j]]=[a[j],a[i]]` |
| Clamp | `Math.max(lo, Math.min(hi, x))` |
| Random int in `[lo,hi]` inclusive | `lo + Math.floor(Math.random()*(hi-lo+1))` |
| Measure runtime | `const t0=performance.now(); /* work */ console.error(performance.now()-t0,'ms');` |
| Deep copy plain data (no functions/cycles) | `structuredClone(obj)` (ES2022+) or `JSON.parse(JSON.stringify(obj))` |

### I/O (CP-style)

| Task | Code |
|---|---|
| Read all input fast | `const data = require('fs').readFileSync(0,'utf8');` |
| Split input into whitespace tokens | `const tok = data.split(/\s+/).filter(Boolean);` |
| Read first line's ints | `const [n,m] = data.split('\n')[0].split(' ').map(Number);` |
| Read a grid of n lines (char matrix) | `const lines=data.split('\n'); const grid=lines.slice(1,1+n).map(l=>l.split(''));` |
| Print all output at once (fast) | `const out=[]; out.push(String(ans)); console.log(out.join('\n'));` |
| Print an array as a space-separated line | `console.log(a.join(' '))` |
| Print multiple lines efficiently | `process.stdout.write(out.join('\n')+'\n')` |

Cross-refs: heap classes — §15. Deque class — §14. TreeMap-substitute patterns — §16. Sort pitfalls — §17.

---

## 34. Master Complexity Table

### Data structures — operation-by-operation

| Structure | Insert end | Insert front | Remove end | Remove front | Random access | Lookup/has | Iterate | Notes / JS caveat |
|---|---|---|---|---|---|---|---|---|
| Array (as list) | O(1)* | O(n) `unshift` | O(1) `pop` | O(n) `shift` | O(1) | O(n) `includes`/`indexOf` | O(n) | `push`/`pop` cheap; `unshift`/`shift` re-index every element |
| Array-as-stack | O(1) `push` | — | O(1) `pop` | — | O(1) | O(n) | O(n) | Ideal — LIFO on the end only, no front ops needed |
| head-index Queue (array + pointer) | O(1) `push` | — | — | O(1) amortized (advance pointer, no splice) | O(1) via `arr[head+i]` | O(n) | O(n) | Never use raw `shift()` as a queue — §36 #3 |
| Deque class (paste, §14) | O(1) | O(1) | O(1) | O(1) | O(1) w/ index math | O(n) | O(n) | Ring buffer or two-stack impl; not built in |
| `Map` | O(1) avg `.set` | — | O(1) avg `.delete` | — | — | O(1) avg `.has`/`.get` | O(n), insertion order preserved | Any key type; no coercion, no reordering surprises |
| object-as-map | O(1) avg | — | O(1) avg `delete obj[k]` | — | — | O(1) avg `in`/`hasOwnProperty` | O(n), mostly insertion order | Integer-like keys get reordered first; all keys coerced to strings |
| `Set` | O(1) avg `.add` | — | O(1) avg `.delete` | — | — | O(1) avg `.has` | O(n), insertion order | No structural dedupe — arrays/objects compared by reference, not value |
| hand-rolled Heap/PriorityQueue | O(log n) push | — | O(log n) pop-root | — | O(1) peek root | O(n) arbitrary lookup | O(n) | Not built in — must paste before use, §15 |
| TypedArray (`Int32Array` etc.) | fixed size | fixed size | fixed size | fixed size | O(1), fast (no boxing) | O(n) | O(n), cache-friendly | Size fixed at construction; assigning auto-truncates to the type |
| sorted-array-as-TreeMap | O(n) (splice) | O(n) | O(n) | O(n) | O(1) | O(log n) via binary search | O(n), sorted | No built-in TreeMap/TreeSet — simulate, or use a Fenwick/segment tree for frequent updates |

\* `push` is amortized O(1); an occasional backing-array resize costs O(n) but averages out over many pushes.

### Method / algorithm complexities

| Operation | Complexity | Notes |
|---|---|---|
| `Array.prototype.sort()` | O(n log n) | V8 uses TimSort; comparator invoked O(n log n) times |
| `arr.includes(x)` / `arr.indexOf(x)` | O(n) | Linear scan — **trap** when called inside another loop → hidden O(n²) |
| `arr.push` / `arr.pop` | O(1) amortized | |
| `arr.shift` / `arr.unshift` | O(n) | **Trap** — shifts every remaining element |
| `arr.splice` | O(n) | Shifts remaining elements past the splice point |
| `Set.has` / `Map.has`/`get`/`set` | O(1) average | Hash-based; worst case O(n), rare/pathological |
| `[...arr]` spread copy | O(n) | |
| `Math.max(...arr)` / `Math.min(...arr)` | O(n) time, but risks call-stack overflow on very large arrays (spread as call args) — §36 #18 |
| Binary search (hand-rolled) | O(log n) | Not built in for arbitrary predicates |
| Heap push / pop (hand-rolled) | O(log n) | Peek root is O(1) |
| Heap build-from-array (heapify) | O(n) | Bottom-up heapify beats n individual inserts (O(n log n)) |
| `JSON.parse`/`stringify` deep copy | O(n) | Plain data only — no functions, `undefined`, or cycles |
| `structuredClone(obj)` | O(n) | ES2022+, handles more types than JSON, still no functions |

**Flag prominently:** `shift()`/`unshift()` are O(n) — never use as a queue in a hot loop. `includes`/`indexOf` are O(n) — never call inside another loop over large data; convert to a `Set`/`Map` first for O(1) lookups.

---

## 35. n → Required Complexity Cheat Sheet

| Constraint on n | Required complexity | Typical technique | Pattern cross-ref |
|---|---|---|---|
| n ≤ 10 | O(n!) or O(2^n · n) | brute-force permutations, full backtracking | Pattern 07 – Backtracking |
| n ≤ 20 | O(2^n) | bitmask DP / bitmask subset enumeration (plain `number` mask fine up to ~30 bits) | Pattern 22 – Bitmask DP |
| n ≤ 100 | O(n^3)–O(n^4) | Floyd–Warshall, triple-nested DP | Pattern 18 – Graph all-pairs / Pattern 25 – Interval DP |
| n ≤ 500 | O(n^3) | dense DP over pairs/triples | Pattern 25 – Interval DP |
| n ≤ 5,000 | O(n^2) | nested loops, simple 2D DP, naive pairwise comparison | Pattern 20 – 2D DP |
| n ≤ 1e5 | O(n log n) | sort-based, heap, binary search, divide & conquer | Pattern 05 – Sorting/Binary Search, Pattern 15 – Heap |
| n ≤ 1e6 | O(n) or O(n log n) with a small constant | single pass, two pointers, sliding window, prefix sums, counting sort | Pattern 01 – Two Pointers, Pattern 02 – Sliding Window |
| n ≤ 1e9 | O(log n), O(√n), or closed-form math | binary search on the answer, modpow, number theory, matrix exponentiation | Pattern 34 – Math/Number Theory |

**Per-tier pitfall to watch:**

| Tier | Common trap at this size |
|---|---|
| n ≤ 20 (bitmask) | Using recursion/subsets when a bitmask loop would be simpler and faster |
| n ≤ 5,000 (O(n^2)) | Accidentally calling `.includes()`/`.indexOf()` inside the O(n) loop, silently making it O(n^3) |
| n ≤ 1e5 (O(n log n)) | Sorting inside a loop instead of once up front |
| n ≤ 1e6 (O(n)) | Using `shift()` for a queue, or `console.log` per iteration — turns O(n) into O(n²) or adds huge constant I/O overhead |
| n ≤ 1e9 (O(log n)/O(√n)) | Iterating up to n directly instead of binary-searching on the answer or using a closed-form/number-theoretic approach |

**JS-specific adjustments:**
- Realistic throughput: **~10^7–10^8 simple ops/sec** in Node — noticeably lower than C++'s ~10^9; budget accordingly on tight judges.
- Some online judges apply the **same time limit as C++/Java** to JS submissions — treat O(n log n) as your practical ceiling for n ≥ 1e6, not "O(n) with a heavy constant."
- Near the limit: prefer **TypedArrays** (`Int32Array`, `Float64Array`) over plain arrays — no boxing, better cache locality, faster iteration.
- Avoid `shift()` and chained `.map().filter().reduce()` allocations inside hot loops — each `.map`/`.filter` call allocates a fresh array.
- Use **BigInt only when precision truly demands it** (values or intermediate products exceeding 2^53) — BigInt arithmetic runs roughly 10-100x slower than native `number` ops.

---

## 36. The Top 30 JS-DSA Mistakes That Cost You Problems

Ranked by how often each one bites in practice. Each entry: symptom you'll actually see, then bad → good.

**1. `arr.sort()` with no comparator on numbers — WA, silently wrong order**
```js
[10, 2, 1].sort();              // -> [1, 10, 2]  (lexicographic!)
[10, 2, 1].sort((a, b) => a - b); // -> [1, 2, 10]
```

**2. Comparator returns boolean instead of a number — WA, inconsistent ordering**
```js
a.sort((x, y) => x > y);   // returns true/false, coerced 1/0 — breaks ties, unreliable
a.sort((x, y) => x - y);   // returns a signed number — correct
```

**3. `shift()` for a queue in a loop — TLE, O(n) per call becomes O(n²) total**
```js
while (q.length) { const x = q.shift(); /* ... */ }          // O(n) per pop
let head = 0; while (head < q.length) { const x = q[head++]; } // O(1) per pop
```

**4. `console.log` inside a large loop — TLE from I/O overhead**
```js
for (const x of results) console.log(x);                 // slow — real syscall per line
console.log(results.join('\n'));                          // one flush
```

**5. Bitwise ops on values beyond 2^31 — silent wrong answer, truncates first**
```js
(3000000000 & 1);           // operand forced to 32-bit signed first — wrong
BigInt(3000000000) & 1n;    // correct for wide values
```

**6. `1 << 31` is negative; `1 << 32 === 1` — off-by-huge-number bugs**
```js
1 << 31;   // -2147483648, not 2147483648
1 << 32;   // 1, shift amount wraps mod 32
```

**7. Precision loss beyond 2^53 (e.g. modular multiply of large ints) — silent WA, no error thrown**
```js
(a * b) % m;                          // wrong once a*b exceeds 2^53
(BigInt(a) * BigInt(b)) % BigInt(m);  // correct
```

**8. `Array(n).fill([])` — silent WA, every slot is the *same* array**
```js
const buckets = Array(n).fill([]);              // buckets[0] === buckets[1] -> true
const buckets = Array.from({ length: n }, () => []);  // independent arrays
```

**9. `new Array(n)` assumed zero-filled — RE/WA, creates holes not zeros**
```js
new Array(n).map(x => 0);       // map SKIPS holes — still all holes
new Array(n).fill(0);           // correctly zero-filled
```

**10. `Array(n).fill(Array(m))` for a 2D grid — silent WA, shared row reference**
```js
const grid = Array(n).fill(Array(m).fill(0));      // grid[0] === grid[1] -> true
const grid = Array.from({ length: n }, () => Array(m).fill(0)); // independent rows
```

**11. `==` instead of `===` — silent WA from coercion**
```js
'' == 0;          // true
null == undefined; // true
'' === 0;         // false — use this
```

**12. `for...in` over an array — WA/slow, wrong iteration semantics**
```js
for (const i in arr) { /* i is a STRING, includes inherited enumerable props */ }
for (const x of arr) { /* correct value iteration */ }
```

**13. Comparing floats with `===` — WA from binary floating-point representation**
```js
0.1 + 0.2 === 0.3;                 // false
Math.abs((0.1 + 0.2) - 0.3) < 1e-9; // true — use an epsilon
```

**14. Forgetting `.sort`/`.reverse`/`.splice` mutate in place — silent corruption/aliasing**
```js
function f(arr) { return arr.sort((a,b)=>a-b); }  // mutates caller's array too
function f(arr) { return [...arr].sort((a,b)=>a-b); } // copy first
```

**15. Assuming object keys keep numeric type — WA in lookups**
```js
const obj = {}; obj[1] = 'x'; obj['1'] === 'x';  // true — same key, both stringified
```
Use a `Map` when key type or insertion order distinction matters.

**16. `[r,c]` array literal as a `Set`/`Map` key — silent WA, dedupe/lookup fails**
```js
const seen = new Set(); seen.add([0,1]); seen.has([0,1]);  // false — different array objects
seen.add(`${0},${1}`); seen.has(`${0},${1}`);               // true — string-encode the key
```

**17. `arr[-1]` expecting Python-style last element — WA, returns `undefined`**
```js
arr[-1];          // undefined, JS has no negative indexing on plain arrays
arr[arr.length-1]; // or arr.at(-1)
```

**18. `Math.max(...hugeArray)` — RE, call-stack overflow on very large arrays**
```js
Math.max(...hugeArray);                          // RangeError past ~10^5-10^6 elements
hugeArray.reduce((m, x) => x > m ? x : m, -Infinity); // safe for any size
```

**19. Deep recursion (e.g. DFS on 1e5 nodes) — RE, stack overflow with no reliable fix**
```js
function dfs(u) { for (const v of g[u]) dfs(v); }   // RangeError past ~10^4 depth
// convert to iterative DFS with an explicit stack instead — §29
```

**20. Mixing `BigInt` and `Number` in one expression — RE, TypeError**
```js
10n + 5;              // TypeError: Cannot mix BigInt and other types
10n + BigInt(5);      // convert consistently first
```

**21. Assuming `substring` behaves like `slice` on negative indices — silent WA**
```js
'hello'.substring(-2); // 'hello' — negative clamps to 0
'hello'.slice(-2);     // 'lo' — counts from the end
```

**22. `parseInt(s)` without radix, or expecting full-string validation — WA**
```js
parseInt('08');      // fine here, but parseInt('0x1A') reads 26 (hex!)
parseInt('12abc');   // 12 — silently ignores trailing junk
Number('12abc');     // NaN — use when the whole string must be a clean number
parseInt(s, 10);     // always pass radix when parseInt is the right tool
```

**23. `map`/`filter`/`forEach` on a sparse array — silently skipped elements**
```js
new Array(3).map(x => 0);   // [ <3 empty items> ] — holes skipped entirely
Array.from({ length: 3 }, () => 0); // [0,0,0] — no holes
```

**24. Forgetting `reduce`'s initial value on a possibly-empty array — RE**
```js
[].reduce((a, b) => a + b);       // TypeError: Reduce of empty array with no initial value
[].reduce((a, b) => a + b, 0);    // 0 — always pass an initial value
```

**25. Treating `[]`/`{}` as falsy — WA in empty-check branches**
```js
if ([]) { /* runs! */ }              // [] is truthy
if (arr.length === 0) { /* correct emptiness check */ }
```

**26. `typeof null === 'object'` — WA in type-guard branches**
```js
typeof null === 'object';   // true — a known JS quirk
x === null;                  // explicit null check, don't rely on typeof
```

**27. `NaN === NaN` is `false` — WA, equality checks silently fail**
```js
NaN === NaN;          // false
Number.isNaN(NaN);    // true — use this
```

**28. `x | 0` / `~~x` for truncation beyond 32-bit range — silent WA**
```js
(3000000000.7 | 0);      // wraps to a negative 32-bit value, wrong
Math.trunc(3000000000.7); // 3000000000 — correct, no 32-bit limit
```

**29. Assuming a built-in heap/TreeMap/deque exists — RE/TLE discovered mid-contest**
JS ships none of these. Keep pasted Heap (§15), Deque (§14), and sorted-array-binary-search (§16) helpers ready *before* the contest starts, not discovered while the clock is running.

**30. Closure capturing a `var` loop variable — silent WA, classic async/closure bug**
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);  // logs 3,3,3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);  // logs 0,1,2 — use let
```

---

## 37. 60-Second Warm-Up Drill + Language-Switch Box

Run this mentally (or paste and adapt) at the start of any session after time away from JS. Each line exercises one construct needed within the first few minutes of a real problem.

```js
const fs = require('fs');
const data = fs.readFileSync(0, 'utf8').split('\n');        // 1. fast input
const [n, m] = data[0].split(' ').map(Number);                // 2. parse first line of ints
const arr = data[1].split(' ').map(Number);                   // 3. parse array of ints

arr.sort((a, b) => a - b);                                     // 4. numeric sort — NEVER bare .sort()

const freq = new Map();                                        // 5. frequency map
for (const x of arr) freq.set(x, (freq.get(x) || 0) + 1);

const visited = new Set();                                     // 6. visited-set for graph/grid traversal
visited.add(`${0},${0}`);                                      // string-encode composite keys

class MinHeap {                                                 // 7. paste-and-use MinHeap (full version §15)
  constructor() { this.a = []; }
  push(v) {
    this.a.push(v);
    let i = this.a.length - 1;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (this.a[p] <= this.a[i]) break;
      [this.a[p], this.a[i]] = [this.a[i], this.a[p]];
      i = p;
    }
  }
  pop() {
    const top = this.a[0], last = this.a.pop();
    if (this.a.length) {
      this.a[0] = last;
      let i = 0;
      while (true) {
        let l = 2 * i + 1, r = 2 * i + 2, s = i;
        if (l < this.a.length && this.a[l] < this.a[s]) s = l;
        if (r < this.a.length && this.a[r] < this.a[s]) s = r;
        if (s === i) break;
        [this.a[s], this.a[i]] = [this.a[i], this.a[s]];
        i = s;
      }
    }
    return top;
  }
  get size() { return this.a.length; }
}
const heap = new MinHeap();
heap.push(5); heap.push(1); heap.push(3);

const grid = Array.from({ length: n }, () => Array(m).fill(0)); // 8. 2D init — independent rows, NOT .fill(Array(m))

for (let mask = 0; mask < (1 << n); mask++) {                   // 9. bitmask subset loop (n <= ~20)
  // process subset `mask`
}

function lowerBound(a, t) {                                     // 10. hand-rolled lowerBound (no built-in)
  let lo = 0, hi = a.length;
  while (lo < hi) {
    const mid = (lo + hi) >>> 1;
    if (a[mid] < t) lo = mid + 1; else hi = mid;
  }
  return lo;
}

const dx = [-1, 0, 1, 0], dy = [0, 1, 0, -1];                    // BFS on a grid, head-index queue
const q = [[0, 0]];
let head = 0;
while (head < q.length) {
  const [x, y] = q[head++];                                      // O(1) dequeue — never q.shift()
  for (let d = 0; d < 4; d++) {
    const nx = x + dx[d], ny = y + dy[d];
    if (nx >= 0 && nx < n && ny >= 0 && ny < m) {
      const key = `${nx},${ny}`;
      if (!visited.has(key)) { visited.add(key); q.push([nx, ny]); }
    }
  }
}

const s = 'abc';
const code0 = s.charCodeAt(0) - 97;                              // 11. char via charCodeAt, 'a'-relative idiom

const out = [];                                                   // 12. buffered output
out.push(String(freq.size));
console.log(out.join('\n'));                                      // single flush at the end
```

### 10-item self-check — if you can't write this cold, re-read the section

| # | Can you write, from memory... | Re-read |
|---|---|---|
| 1 | A numeric sort comparator and a multi-key sort | §17 |
| 2 | A hand-rolled MinHeap/MaxHeap class from scratch | §15 |
| 3 | A correct 2D array init (no shared-row bug) | §33, §36 (#8-10) |
| 4 | lowerBound/upperBound binary search helpers | §19, §33 |
| 5 | A head-index queue (not `shift()`) for BFS | §14, §36 (#3) |
| 6 | Fast `readFileSync(0,'utf8')` input parsing and buffered output | §28, §33 |
| 7 | popcount32 / gcd / modpow from memory | §22-24, §33 |
| 8 | Why `Set`/`Map` can't dedupe `[r,c]` arrays directly | §13, §36 (#16) |
| 9 | The difference between `slice` and `substring` with negative indices | §10, §36 (#21) |
| 10 | Why deep recursion throws RangeError and what to do instead | §29, §36 (#19) |

### C++/Java → JS mental-switch box

Read this every time you switch INTO JS from C++/Java:

| # | Remember |
|---|---|
| 1 | All numbers are IEEE-754 doubles — safe integers only up to 2^53; use `BigInt` beyond that (suffix `n`, e.g. `10n`) |
| 2 | Bitwise operators (`&`, `\|`, `^`, `~`, `<<`, `>>`) are **32-bit signed** — they silently truncate; use `BigInt` for wider bit manipulation |
| 3 | `.sort()` default is **lexicographic** on stringified elements — always pass a comparator for numbers |
| 4 | There is **no built-in heap, TreeMap/TreeSet, or deque** — paste your own (§14-16) before you need it, not mid-problem |
| 5 | `Array.shift()`/`unshift()` are **O(n)**, not O(1) like `std::deque`/`ArrayDeque` — use a head-index pointer or a real deque |
| 6 | Use `===`/`!==`, never `==`/`!=`, to avoid coercion surprises |
| 7 | Strings are immutable and have **no char type** — indexing gives a 1-char string; use `charCodeAt`/`fromCharCode` for char-code arithmetic |
| 8 | Recursion overflows around **~10^4** stack depth (not tunable like `ulimit -s`/a thread's stack size) — go iterative with an explicit stack for deep DFS |
| 9 | For competitive-programming I/O: `readFileSync(0,'utf8')` once for all input, buffer output into an array and `join('\n')` once — not `cin`/`cout` line-by-line, not `console.log` per line |
| 10 | Swap is `[a,b]=[b,a]` — no temp variable, no `std::swap`/`Collections.swap` needed |

---

*JavaScript Complete Reference — companion to DSA Patterns 01–38 and the C++ (doc 39) and Java (doc 40) references.*
*When you forget something and it is not in here, add it. This document should only grow.*
