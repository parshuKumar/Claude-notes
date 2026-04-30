# 32 — Spread Operator (Arrays)

---

## What is this?

The **spread operator** (`...`) "spreads" an array's elements out into individual values. It unpacks an array wherever a list of values is expected.

```js
const nums = [1, 2, 3];

console.log(nums);    // [1, 2, 3]  ← the array itself
console.log(...nums); // 1 2 3      ← three individual values
```

Three dots. That's it. But it unlocks an enormous range of patterns.

---

## Why does it matter?

Before spread (pre-ES6), copying arrays, merging them, or passing them to functions required verbose workarounds like `.concat()`, `.apply()`, and manual loops. Spread made those operations a single readable line:

- Copy an array without sharing references
- Merge multiple arrays into one
- Pass array elements as function arguments
- Add items to arrays immutably (without mutation)
- Clone and modify arrays in one expression

---

## Syntax

```js
// Inside an array literal — spreading an array into another array
const combined = [...arr1, ...arr2];

// Inside a function call — spreading array elements as arguments
someFunction(...arr);

// Spreading a string into characters
const chars = [..."hello"]; // ['h', 'e', 'l', 'l', 'o']

// Spreading any iterable (arrays, strings, Sets, Maps, NodeLists)
const unique = [...new Set([1, 2, 2, 3])]; // [1, 2, 3]
```

---

## How it works — line by line

```js
const a = [1, 2, 3];
const b = [4, 5, 6];

const merged = [...a, ...b];
// Step 1: ...a expands to  1, 2, 3
// Step 2: ...b expands to  4, 5, 6
// Step 3: array literal wraps them → [1, 2, 3, 4, 5, 6]

console.log(merged); // [1, 2, 3, 4, 5, 6]
```

The spread operator works inside `[...]` array literals and `fn(...)` function calls. It does **not** work as a standalone expression.

---

## Example 1 — Copying an Array (shallow clone)

```js
const original = [1, 2, 3];

// ❌ Assignment — both variables point to the same array
const ref = original;
ref.push(4);
console.log(original); // [1, 2, 3, 4]  ← original was mutated!

// ✅ Spread — creates a new array with the same values
const copy = [...original];
copy.push(99);
console.log(original); // [1, 2, 3, 4]  ← untouched
console.log(copy);     // [1, 2, 3, 4, 99]
```

---

## Example 2 — Merging Arrays

```js
const fruits  = ['apple', 'banana'];
const veggies = ['carrot', 'broccoli'];
const grains  = ['rice', 'oats'];

// Old way
const allOld = fruits.concat(veggies).concat(grains);

// Spread way — cleaner and you can insert items anywhere
const all = [...fruits, 'mango', ...veggies, ...grains];
console.log(all);
// ['apple', 'banana', 'mango', 'carrot', 'broccoli', 'rice', 'oats']
```

---

## Example 3 — Passing Array Elements as Function Arguments

Some functions expect individual arguments, not an array. `Math.max` is the classic example:

```js
const scores = [44, 92, 78, 55, 100, 61];

// ❌ Wrong — passes the whole array as one argument
console.log(Math.max(scores)); // NaN

// ✅ Spread — passes each element as a separate argument
console.log(Math.max(...scores)); // 100
console.log(Math.min(...scores)); // 44

// Same pattern with any function
function add(a, b, c) { return a + b + c; }
const vals = [10, 20, 30];
console.log(add(...vals)); // 60
```

---

## Example 4 — Immutable Array Operations (React / state management style)

When you cannot mutate state directly, spread lets you build modified copies:

```js
const cart = ['laptop', 'mouse', 'keyboard'];

// Add an item (without push)
const withWebcam = [...cart, 'webcam'];
// cart is still ['laptop', 'mouse', 'keyboard']

// Remove an item (without splice)
const withoutMouse = cart.filter(item => item !== 'mouse');

// Insert at a specific index (without splice)
const index = 1;
const withHeadphones = [
  ...cart.slice(0, index),
  'headphones',
  ...cart.slice(index),
];
console.log(withHeadphones); // ['laptop', 'headphones', 'mouse', 'keyboard']
```

---

## Example 5 — Converting Iterables to Arrays

```js
// String → array of characters
const letters = [..."JavaScript"];
console.log(letters); // ['J','a','v','a','S','c','r','i','p','t']

// Set → array (removes duplicates)
const tags = new Set(['js', 'css', 'js', 'html', 'css']);
const uniqueTags = [...tags];
console.log(uniqueTags); // ['js', 'css', 'html']

// NodeList → array (so you can use array methods)
// const divs = [...document.querySelectorAll('div')];
// divs.forEach(div => div.classList.add('active'));

// Map entries → array of [key, value] pairs
const map = new Map([['a', 1], ['b', 2]]);
const pairs = [...map];
console.log(pairs); // [['a', 1], ['b', 2]]
```

---

## Tricky things you'll encounter in the real world

### 1. Spread is a SHALLOW copy — nested arrays/objects are still shared

```js
const matrix = [[1, 2], [3, 4], [5, 6]];
const copy   = [...matrix];

copy[0].push(99); // mutating the nested array

console.log(matrix[0]); // [1, 2, 99]  ← original was affected!
console.log(copy[0]);   // [1, 2, 99]

// They share the same inner arrays — spread only copies the outer level.
```

To deep-clone, use `JSON.parse(JSON.stringify(arr))` (for simple data) or a library like `structuredClone`:

```js
const deepCopy = structuredClone(matrix);
deepCopy[0].push(99);
console.log(matrix[0]); // [1, 2]  ← safe now
```

---

### 2. Spreading a large array into a function call can exceed the call stack

```js
const huge = new Array(1_000_000).fill(1);

// This can throw "Maximum call stack size exceeded" in some environments
const max = Math.max(...huge); // ⚠️ potentially dangerous

// Safer alternative:
const max2 = huge.reduce((a, b) => Math.max(a, b), -Infinity);
```

The spec says there's no hard limit, but in practice engines have limits. For very large arrays, prefer `reduce`.

---

### 3. Spread creates a new array — the original is not modified

```js
const arr = [3, 1, 2];
const sorted = [...arr].sort((a, b) => a - b);

console.log(arr);    // [3, 1, 2]  ← untouched
console.log(sorted); // [1, 2, 3]

// Without spread, .sort() mutates in place:
// arr.sort() would change arr itself
```

This pattern — `[...arr].sort()` — is extremely common when you want a sorted copy.

---

### 4. Spread doesn't work on non-iterables

```js
const obj = { a: 1, b: 2 };

// ❌ — objects are not iterable in array context
const bad = [...obj]; // TypeError: obj is not iterable

// ✅ — use Object.values() or Object.entries() first
const vals = [...Object.values(obj)]; // [1, 2]

// NOTE: Spread DOES work on objects inside object literals: { ...obj }
// That's covered in the Object Spread topic (33).
```

---

### 5. Order matters when spreading multiple arrays

```js
const defaults  = [1, 2, 3];
const overrides = [10, 20];

// overrides comes last → its values win at their positions
const merged1 = [...defaults, ...overrides]; // [1, 2, 3, 10, 20]

// Think carefully about where you place each spread
const withPrefix = [0, ...defaults];  // [0, 1, 2, 3]
const withSuffix = [...defaults, 99]; // [1, 2, 3, 99]
```

---

### 6. Spread vs `Array.from` — when to use which

Both convert iterables to arrays. The difference:

```js
// Spread — concise, but no transform step
const arr1 = [...'hello']; // ['h', 'e', 'l', 'l', 'o']

// Array.from — accepts a mapping function as second argument
const arr2 = Array.from('hello', char => char.toUpperCase());
// ['H', 'E', 'L', 'L', 'O']

// Array.from for array-like objects (have .length but aren't iterable)
const arrayLike = { 0: 'a', 1: 'b', length: 2 };
const arr3 = Array.from(arrayLike); // ['a', 'b']
const arr4 = [...arrayLike];        // TypeError — not iterable
```

Use spread for quick conversions. Use `Array.from` when you need a transform or the source is array-like (not iterable).

---

## Common mistakes

```js
// ❌ Trying to spread inside a non-literal context
const spread = ...arr; // SyntaxError — spread only works inside [] or fn()
// ✅ Wrap in an array literal:
const spread = [...arr];

// ❌ Thinking spread deep-clones
const nested = [[1, 2], [3, 4]];
const clone  = [...nested];
clone[0][0] = 999;
console.log(nested[0][0]); // 999 — shared reference!
// ✅ Use structuredClone for deep cloning

// ❌ Forgetting to wrap in [] when combining
const a = [1, 2], b = [3, 4];
const wrong = ...a, ...b;   // SyntaxError
const right = [...a, ...b]; // [1, 2, 3, 4]

// ❌ Using spread on very large arrays in Math.max
Math.max(...new Array(500_000)); // might crash
// ✅ Use reduce for large arrays
```

---

## Frequently asked questions

**Q: Is `[...arr]` the same as `arr.slice()`?**
For a simple flat array, yes — both produce a shallow copy. `[...arr]` is more readable and also works on any iterable, not just arrays.

**Q: Can I spread multiple arrays in one expression?**
Yes: `[...a, ...b, ...c, 99, ...d]` — mix spreads, literals, and values freely.

**Q: Does spread work on `arguments` (the old function arguments object)?**
Yes: `[...arguments]` converts it to a real array. But prefer rest parameters (`...args`) in modern code.

**Q: What's the difference between spread (`...`) and rest (`...`)?**
Same syntax, different context:
- **Spread**: *expands* an array/iterable into individual values → used in function calls and array/object literals
- **Rest**: *collects* individual values into an array → used in function parameters and destructuring
```js
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); } // REST
const copy = [...nums]; // SPREAD
```

**Q: Can I spread an async iterable?**
Not with `[...]`. Use `for await...of` to collect async iterable values.

---

## Practice exercises

### Easy — Array Utilities

Given these arrays:

```js
const evens = [2, 4, 6, 8];
const odds  = [1, 3, 5, 7];
const extra = [10, 20];
```

Using only spread (no `.concat`, no `.push`, no loops):

1. Create `all` containing all numbers from `evens`, `odds`, and `extra` in that order.
2. Create `withZero` — a copy of `evens` with `0` added at the very beginning.
3. Create `sandwiched` — `extra` inserted between `evens` and `odds`.
4. Find the maximum value across all three arrays in one expression.

---

### Medium — Immutable Cart

```js
let cart = [
  { id: 1, name: 'Laptop',   qty: 1 },
  { id: 2, name: 'Mouse',    qty: 2 },
  { id: 3, name: 'Keyboard', qty: 1 },
];
```

Write these functions — each must return a **new array** (do not mutate `cart`):

```js
function addItem(cart, item) { /* ... */ }
// Adds item to the end; returns new cart

function removeItem(cart, id) { /* ... */ }
// Removes item with matching id; returns new cart

function insertAt(cart, index, item) { /* ... */ }
// Inserts item at the given index; returns new cart
```

Test:
```js
const withHeadset = addItem(cart, { id: 4, name: 'Headset', qty: 1 });
const noMouse     = removeItem(cart, 2);
const inserted    = insertAt(cart, 1, { id: 5, name: 'Webcam', qty: 1 });

console.log(cart.length);      // 3  — original unchanged
console.log(withHeadset.length); // 4
console.log(noMouse.length);     // 2
console.log(inserted[1].name);   // 'Webcam'
```

---

### Hard — Deep Array Merge

Write a function `mergeUnique(...arrays)` that:
1. Accepts any number of arrays using rest parameters
2. Merges all arrays into one
3. Removes duplicates (each value appears only once)
4. Preserves the original order of first appearance
5. Does not use `Set` directly — implement the deduplication manually using array methods

```js
const a = [1, 2, 3, 2];
const b = [3, 4, 5];
const c = [1, 5, 6, 7];

console.log(mergeUnique(a, b, c)); // [1, 2, 3, 4, 5, 6, 7]
console.log(mergeUnique([1, 1, 1], [1, 2])); // [1, 2]
console.log(mergeUnique()); // []
```

---

## Quick reference — cheat sheet

```
[...arr]              → shallow copy of arr
[...a, ...b]          → merge two arrays
[x, ...arr]           → prepend x
[...arr, x]           → append x
[...arr].sort()       → sorted copy (arr unchanged)
[...arr].reverse()    → reversed copy (arr unchanged)
fn(...arr)            → pass elements as arguments
[...'hello']          → ['h','e','l','l','o']
[...new Set(arr)]     → deduplicated array
[...map]              → array of [key, value] pairs

Spread is SHALLOW — nested references are shared
```

---

## Connected topics

- **[33] Array Destructuring** — the counterpart to spread; pulls values out of arrays
- **[37] Spread Operator (Objects)** — same `...` syntax, used with object literals
- **[24] Array Mutation Methods** — spread lets you do these immutably
- **[25] Array Non-Mutation Methods** — many patterns combine spread with these
- **[18] Scope** — spread creates new bindings; understanding scope helps reason about copies
