# 33 — Array Destructuring

## What is this?

**Array destructuring** is a syntax that lets you unpack values from an array into individual variables in a single statement.

```js
// without destructuring
const colours = ['red', 'green', 'blue'];
const first  = colours[0];
const second = colours[1];
const third  = colours[2];

// with destructuring — same result, one line
const [first, second, third] = colours;
```

The left side mirrors the shape of the array on the right. JS matches by **position**.

---

## Why does it matter?

Arrays are everywhere — function return values, API responses, `Object.entries()`, `useState` in React, regex matches, coordinate pairs. Destructuring removes the noise of repeated index access and makes your intent clear at a glance.

```js
// without
const result = getUserLocation();
const lat  = result[0];
const lng  = result[1];
const city = result[2];

// with
const [lat, lng, city] = getUserLocation();
```

---

## Syntax

```js
// basic
const [a, b, c] = array;

// skip elements
const [first, , third] = array;   // note the empty slot

// default values
const [x = 0, y = 0] = array;

// rest — collect remaining elements
const [head, ...tail] = array;

// swap variables
[a, b] = [b, a];

// nested arrays
const [[x1, y1], [x2, y2]] = points;

// from a function return
const [err, data] = riskyOperation();
```

---

## How it works — line by line

```js
const scores = [95, 82, 77, 60, 45];

const [top, second, , fourth] = scores;

// Position 0 → top    = 95
// Position 1 → second = 82
// Position 2 → skipped (empty slot between commas)
// Position 3 → fourth = 60
// Position 4 → not requested, ignored

console.log(top);    // 95
console.log(second); // 82
console.log(fourth); // 60
```

---

## Example 1 — basic patterns

```js
// ── basic ──────────────────────────────────────────────
const [r, g, b] = [255, 128, 0];
console.log(r, g, b); // 255 128 0

// ── skipping ───────────────────────────────────────────
const [,, blue] = [255, 128, 0];
console.log(blue); // 0

// ── default values ─────────────────────────────────────
const [width = 100, height = 100, depth = 1] = [1920, 1080];
console.log(width, height, depth); // 1920 1080 1

// ── rest ───────────────────────────────────────────────
const [champion, runnerUp, ...others] = ['Alice', 'Bob', 'Carol', 'Dave', 'Eve'];
console.log(champion);  // 'Alice'
console.log(runnerUp);  // 'Bob'
console.log(others);    // ['Carol', 'Dave', 'Eve']

// ── swap ───────────────────────────────────────────────
let x = 10, y = 20;
[x, y] = [y, x];
console.log(x, y); // 20 10
```

---

## Example 2 — real world: functions that return multiple values

JS functions can only return one value. The conventional workaround is to return an array (or object). Destructuring makes consuming it clean.

```js
function minMax(numbers) {
  const sorted = [...numbers].sort((a, b) => a - b);
  return [sorted[0], sorted[sorted.length - 1]];
}

const temperatures = [22, 15, 31, 27, 8, 29];
const [coldest, hottest] = minMax(temperatures);

console.log(`Coldest: ${coldest}°C, Hottest: ${hottest}°C`);
// Coldest: 8°C, Hottest: 31°C
```

```js
// Another pattern: error-first tuple (popular in Go-style JS)
function parseJSON(str) {
  try {
    return [null, JSON.parse(str)];
  } catch (err) {
    return [err.message, null];
  }
}

const [error, data] = parseJSON('{"name":"Parsh"}');
if (error) {
  console.error('Parse failed:', error);
} else {
  console.log(data.name); // 'Parsh'
}

const [error2, data2] = parseJSON('{bad json}');
console.log(error2); // 'Unexpected token...'
```

---

## Example 3 — real world: `Object.entries()` in a loop

`Object.entries()` returns an array of `[key, value]` pairs. Destructuring each pair directly in the loop variable is idiomatic JS.

```js
const productPrices = {
  laptop:     999,
  mouse:       29,
  keyboard:    79,
  monitor:    349,
};

// Without destructuring
for (const entry of Object.entries(productPrices)) {
  console.log(entry[0], entry[1]);
}

// With destructuring — much cleaner
for (const [product, price] of Object.entries(productPrices)) {
  console.log(`${product}: $${price}`);
}
// laptop: $999
// mouse: $29
// keyboard: $79
// monitor: $349

// Apply a 10% discount to each
const discounted = Object.fromEntries(
  Object.entries(productPrices).map(([product, price]) => [product, price * 0.9])
);
console.log(discounted);
// { laptop: 899.1, mouse: 26.1, keyboard: 71.1, monitor: 314.1 }
```

---

## Example 4 — real world: regex matches

`String.prototype.match()` returns an array where index 0 is the full match and subsequent indices are capture groups. Destructuring lets you name the captures.

```js
const logLine = '2026-05-01 14:32:07 ERROR Failed to connect to database';

const pattern = /^(\d{4}-\d{2}-\d{2}) (\d{2}:\d{2}:\d{2}) (\w+) (.+)$/;
const [, date, time, level, message] = logLine.match(pattern);
//      ^ skip the full match at index 0

console.log(date);    // '2026-05-01'
console.log(time);    // '14:32:07'
console.log(level);   // 'ERROR'
console.log(message); // 'Failed to connect to database'
```

---

## Example 5 — real world: React `useState` pattern

React's `useState` hook returns a two-element array. This is one of the most-seen destructuring patterns in modern front-end code.

```js
// React (shown for pattern recognition — you don't need React knowledge yet)
const [count, setCount] = useState(0);
const [isOpen, setIsOpen] = useState(false);
const [user, setUser] = useState(null);

// The naming convention is always: [value, setter]
// This is only possible and clean because of array destructuring
```

---

## Example 6 — real world: coordinate and geometry data

```js
// GPS coordinates
const waypoints = [
  [28.6139, 77.2090],   // Delhi
  [19.0760, 72.8777],   // Mumbai
  [12.9716, 77.5946],   // Bangalore
];

for (const [lat, lng] of waypoints) {
  console.log(`Lat: ${lat}, Lng: ${lng}`);
}

// CSV parsing — split returns an array
const csvRow = 'Parsh,22,Developer,Mumbai';
const [name, age, role, city] = csvRow.split(',');
console.log(name, age, role, city);
// Parsh 22 Developer Mumbai

// Destructuring with type conversion
const [, rawAge] = csvRow.split(',');
const parsedAge = Number(rawAge);
console.log(parsedAge + 1); // 23
```

---

## Tricky things you'll encounter in the real world

### 1. Destructuring `undefined` or `null` throws a TypeError

```js
// If the array is undefined, destructuring crashes
function getConfig() {
  return null; // maybe an API returned nothing
}

const [host, port] = getConfig();
// TypeError: Cannot destructure property of null

// Fix: use a fallback with || or ??
const [host, port] = getConfig() ?? [];
// OR provide defaults in the destructure itself
const [host = 'localhost', port = 3000] = getConfig() ?? [];
console.log(host, port); // 'localhost' 3000
```

---

### 2. Default values only apply when the element is `undefined`, NOT `null` or `0`

```js
const [a = 'default'] = [undefined]; // a = 'default'  ✅
const [b = 'default'] = [null];      // b = null        ← null is not undefined
const [c = 'default'] = [0];         // c = 0           ← 0 is not undefined
const [d = 'default'] = [];          // d = 'default'   ✅ (missing = undefined)
```

---

### 3. Rest must always be last

```js
// ❌ Invalid syntax
const [...head, last] = [1, 2, 3]; // SyntaxError

// ✅ Rest must be the final element
const [first, ...rest] = [1, 2, 3]; // rest = [2, 3]
```

---

### 4. Destructuring does not consume or modify the original array

```js
const arr = [1, 2, 3];
const [x, y] = arr;
console.log(arr); // [1, 2, 3] — untouched
```

---

### 5. Nested destructuring can get hard to read — know when to stop

```js
// Readable — two levels is usually fine
const [[x1, y1], [x2, y2]] = [[0, 0], [10, 20]];

// Hard to read — prefer intermediate variables
const [[[a, b], c], [d, [e, f]]] = [[[1, 2], 3], [4, [5, 6]]];
// Better:
const [firstGroup, secondGroup] = [[[1, 2], 3], [4, [5, 6]]];
const [[innerPair, c2]] = [firstGroup];
```

---

### 6. Swapping without a temporary variable

The swap pattern `[a, b] = [b, a]` creates a temporary array on the right side, then destructures it. It's clean but has a subtle rule: you cannot use `const` here if the variables were already declared.

```js
let a = 1, b = 2;
[a, b] = [b, a];   // ✅ — reassigning existing let variables
console.log(a, b); // 2 1

// ❌ This won't work:
const c = 1, d = 2;
[c, d] = [d, c]; // TypeError: Assignment to constant variable
```

---

### 7. Array destructuring on strings

Strings are iterable, so you can destructure them character by character.

```js
const [first, second, ...rest] = 'hello';
console.log(first);  // 'h'
console.log(second); // 'e'
console.log(rest);   // ['l', 'l', 'o']
```

---

## Common mistakes

```js
// ❌ Using object destructuring syntax on an array
const { 0: first, 1: second } = ['a', 'b']; // works but looks wrong
const [first, second] = ['a', 'b'];          // use this instead

// ❌ Forgetting to skip with empty commas
const arr = [1, 2, 3, 4];
const [, , third] = arr;  // ✅
const [third] = arr;       // ❌ — this gives you 1, not 3

// ❌ Expecting rest to give a value when empty
const [a, b, ...rest] = [1, 2];
console.log(rest); // [] — empty array, NOT undefined

// ❌ Declaring the variable twice
const [x] = [1];
const [x] = [2]; // SyntaxError: Identifier 'x' has already been declared

// ❌ Destructuring inside a condition without assignment
// if (const [a, b] = arr) — INVALID SYNTAX
// Destructure first, then use the variables
```

---

## Frequently asked questions

**Q: What is the difference between array destructuring and object destructuring?**
Array destructuring assigns by **position** — the name you give the variable is up to you. Object destructuring assigns by **key name** (covered in topic 36).

**Q: Can I rename variables when destructuring an array?**
There are no keys to rename — the variable name is whatever you choose. `const [myVar] = [42]` — `myVar` is your name for position 0.

**Q: Does destructuring work on `Map`, `Set`, and other iterables?**
Yes — anything iterable. `const [first] = new Set([10, 20, 30])` gives `10`.

**Q: Is `const [a, b] = arr` the same as `let [a, b] = arr`?**
`const` means you cannot reassign `a` or `b`. `let` allows reassignment. Use `const` by default; use `let` only if you need to reassign (like in the swap pattern).

**Q: Can I use destructuring in function parameters?**
Yes — this is very common:
```js
function display([title, score]) {
  console.log(`${title}: ${score}`);
}
display(['Maths', 95]);
```

---

## Practice exercises

### Exercise 1 — easy

Given this data:

```js
const rgb = [204, 85, 0];
const leaderboard = ['Alice', 'Bob', 'Carol', 'Dave', 'Eve', 'Frank'];
const csvLine = '42,London,Engineer,true';
```

Using only destructuring:

1. Extract `r`, `g`, `b` from `rgb`
2. Extract `gold`, `silver`, `bronze` and `rest` (remaining players) from `leaderboard`
3. Parse `csvLine` — split it, then destructure into `id`, `city`, `role`, `isActive` (convert `isActive` to a real boolean)
4. Write a one-liner to swap `gold` and `silver` without a temp variable

---

### Exercise 2 — medium

You receive raw API responses in this shape:

```js
const responses = [
  [null, { id: 1, name: 'Laptop',   price: 999 }],
  ['Not found', null],
  [null, { id: 2, name: 'Mouse',    price: 29  }],
  ['Unauthorized', null],
  [null, { id: 3, name: 'Monitor',  price: 349 }],
];
```

Each item is `[error, data]`.

1. Write `processResponse([error, data])` — logs `'Error: <msg>'` if there's an error, otherwise logs `'Product: <name> — $<price>'`
2. Use `responses.forEach` with `processResponse` to handle all of them
3. Write `getSuccessful(responses)` — returns only the data objects from successful responses (where error is `null`). Use destructuring inside `filter` and `map`.
4. Write `getTotalValue(responses)` — sums the prices of all successful products

---

### Exercise 3 — hard

Build a `parseCSV(csvString)` function that parses a multi-line CSV string into an array of structured objects.

```js
const csv = `id,name,age,city,salary
1,Alice,30,London,75000
2,Bob,25,Paris,62000
3,Carol,35,Berlin,88000
4,Dave,28,Tokyo,70000`;

const result = parseCSV(csv);
console.log(result);
// [
//   { id: 1, name: 'Alice', age: 30, city: 'London', salary: 75000 },
//   { id: 2, name: 'Bob',   age: 25, city: 'Paris',  salary: 62000 },
//   ...
// ]
```

Requirements:

1. Use destructuring to extract the header row from the data rows
2. Use destructuring inside `.map()` to unpack each row into named variables
3. Convert `id`, `age`, and `salary` to numbers (not strings)
4. Add a bonus `isHighEarner` boolean field — `true` if salary > 70000
5. The function must work for any CSV — don't hardcode column names

---

## Quick reference — cheat sheet

```js
// ── basic ─────────────────────────────────────────────
const [a, b, c]    = [1, 2, 3];

// ── skip ──────────────────────────────────────────────
const [, second]   = [1, 2, 3];      // skip first
const [, , third]  = [1, 2, 3];      // skip first two

// ── default values ────────────────────────────────────
const [x = 0, y = 0] = [10];         // x=10, y=0

// ── rest ──────────────────────────────────────────────
const [head, ...tail] = [1, 2, 3];   // head=1, tail=[2,3]

// ── swap ──────────────────────────────────────────────
[a, b] = [b, a];

// ── nested ────────────────────────────────────────────
const [[x1, y1]] = [[5, 10]];

// ── in function params ────────────────────────────────
function fn([a, b]) { ... }

// ── in for...of ───────────────────────────────────────
for (const [key, val] of Object.entries(obj)) { ... }

// ── safe fallback ─────────────────────────────────────
const [a, b] = maybeNull ?? [];

// ── error-first tuple ─────────────────────────────────
const [err, data] = riskyFn();
if (err) { /* handle */ }
```

---

## Connected topics

| Topic | Why it connects |
|---|---|
| [32-spread-arrays.md](32-spread-arrays.md) | Rest (`...tail`) uses the same `...` syntax as spread |
| [36-object-destructuring.md](36-object-destructuring.md) | Same concept but by key name instead of position |
| [38-object-keys-values-entries.md](38-object-keys-values-entries.md) | `Object.entries` returns `[key, value]` pairs — destructured in every loop |
| [27-map.md](27-map.md) | Destructuring in `.map()` callbacks is extremely common |
| [21-callback-functions.md](21-callback-functions.md) | Error-first tuple pattern mirrors error-first callbacks |
