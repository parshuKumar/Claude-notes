# 32 — Spread with Arrays

## What is this?

The **spread operator** (`...`) expands an iterable (like an array) into individual elements in place. Think of it as unpacking a box — instead of handing someone the box, you lay out every item from the box individually.

```js
const nums = [1, 2, 3];

console.log(nums);      // [1, 2, 3]   ← the box
console.log(...nums);   // 1 2 3       ← the items laid out
```

This single syntax covers four common operations without any loops: **copying arrays**, **merging arrays**, **inserting elements**, and **passing array items as function arguments**.

---

## Why does it matter?

Before spread (ES6), developers had to use `.concat()`, `.apply()`, and `.slice()` for these jobs. Spread replaces all of them with a single consistent syntax:

```js
// old way to merge two arrays
const merged = arr1.concat(arr2);

// old way to pass array elements as arguments
Math.max.apply(null, numbers);

// new way — both problems solved with ...
const merged  = [...arr1, ...arr2];
const biggest = Math.max(...numbers);
```

You will see spread in virtually every modern JS codebase.

---

## Syntax

```js
// copy
const copy = [...originalArray];

// merge
const merged = [...arr1, ...arr2];

// insert elements
const withExtra = [first, ...middle, last];

// pass as function arguments
someFunction(...argsArray);

// spread inside array literal
const combo = [0, ...arr1, 99, ...arr2];
```

---

## How it works — line by line

```js
const colours = ['red', 'green', 'blue'];
const moreColours = ['yellow', ...colours, 'purple'];

// Step 1: JS evaluates ...colours → 'red', 'green', 'blue'
// Step 2: builds the new array with those values in place
// Result: ['yellow', 'red', 'green', 'blue', 'purple']

console.log(moreColours);
// ['yellow', 'red', 'green', 'blue', 'purple']

// The original is untouched
console.log(colours);
// ['red', 'green', 'blue']
```

---

## Example 1 — copying an array (shallow copy)

Spread is the most common way to make a copy of an array before mutating it.

```js
const original = [10, 20, 30];

// WITHOUT spread — same reference, mutation affects both
const ref = original;
ref.push(40);
console.log(original); // [10, 20, 30, 40]  ← original changed!

// WITH spread — new array, independent
const copy = [...original];
copy.push(99);
console.log(original); // [10, 20, 30, 40]  ← untouched
console.log(copy);     // [10, 20, 30, 40, 99]
```

---

## Example 2 — merging arrays

```js
const frontend = ['HTML', 'CSS', 'JavaScript'];
const backend  = ['Node.js', 'Express', 'SQL'];
const devops   = ['Docker', 'CI/CD'];

// merge all into one
const fullStack = [...frontend, ...backend, ...devops];
console.log(fullStack);
// ['HTML', 'CSS', 'JavaScript', 'Node.js', 'Express', 'SQL', 'Docker', 'CI/CD']

// add items before/after while merging
const curriculum = ['Intro', ...frontend, 'Break', ...backend, 'Final Project'];
console.log(curriculum);
// ['Intro', 'HTML', 'CSS', 'JavaScript', 'Break', 'Node.js', 'Express', 'SQL', 'Final Project']
```

---

## Example 3 — real world: immutable state updates (React / Redux style)

In front-end frameworks, state must not be mutated directly. Spread lets you create updated copies cleanly.

```js
const state = {
  todos: [
    { id: 1, text: 'Learn spread', done: false },
    { id: 2, text: 'Build a project', done: false },
    { id: 3, text: 'Get a job', done: false },
  ],
};

// Add a new todo — don't push onto existing array
function addTodo(todos, newTodo) {
  return [...todos, newTodo];
}

// Remove a todo by id — don't splice existing array
function removeTodo(todos, id) {
  return todos.filter(todo => todo.id !== id);
}

// Mark a todo as done — don't mutate the object
function completeTodo(todos, id) {
  return todos.map(todo =>
    todo.id === id
      ? { ...todo, done: true }   // spread on object (covered in topic 37)
      : todo
  );
}

const withNew     = addTodo(state.todos, { id: 4, text: 'Deploy', done: false });
const withRemoved = removeTodo(withNew, 2);
const withDone    = completeTodo(withRemoved, 1);

console.log(withDone);
// [
//   { id: 1, text: 'Learn spread', done: true },
//   { id: 3, text: 'Get a job',    done: false },
//   { id: 4, text: 'Deploy',       done: false },
// ]

// original state is untouched
console.log(state.todos.length); // 3
```

---

## Example 4 — real world: passing array as function arguments

Some functions expect individual arguments, not an array. Spread bridges the gap.

```js
const temperatures = [22, 18, 31, 27, 15, 29, 24];

// Math.max / Math.min don't accept an array
console.log(Math.max(temperatures));    // NaN — wrong
console.log(Math.max(...temperatures)); // 31  — correct

// Same with Math.min
console.log(Math.min(...temperatures)); // 15

// Custom function that takes individual args
function createTag(tag, ...classes) {
  return `<${tag} class="${classes.join(' ')}">`;
}

const classList = ['btn', 'btn-primary', 'large'];
console.log(createTag('button', ...classList));
// <button class="btn btn-primary large">
```

---

## Example 5 — real world: building a notification feed

A real app pattern — merging notification batches from different sources, deduplicating, and sorting.

```js
const serverNotifications = [
  { id: 'n1', message: 'New follower',   timestamp: 1714500000 },
  { id: 'n2', message: 'Your post liked',timestamp: 1714503600 },
];

const pushNotifications = [
  { id: 'n3', message: 'Sale starts now', timestamp: 1714507200 },
  { id: 'n1', message: 'New follower',    timestamp: 1714500000 }, // duplicate
];

// merge both sources
const combined = [...serverNotifications, ...pushNotifications];

// deduplicate by id
const seen = new Set();
const feed = combined.filter(n => {
  if (seen.has(n.id)) return false;
  seen.add(n.id);
  return true;
});

// sort newest first
feed.sort((a, b) => b.timestamp - a.timestamp);

console.log(feed);
// [
//   { id: 'n3', message: 'Sale starts now', timestamp: 1714507200 },
//   { id: 'n2', message: 'Your post liked', timestamp: 1714503600 },
//   { id: 'n1', message: 'New follower',    timestamp: 1714500000 },
// ]
```

---

## Tricky things you'll encounter in the real world

### 1. Spread makes a **shallow** copy — nested objects are still shared

This is the #1 spread gotcha. Spread only copies one level deep.

```js
const original = [{ name: 'Alice', scores: [90, 85] }, { name: 'Bob', scores: [70, 75] }];
const copy      = [...original];

// Adding to the copy's array is fine
copy.push({ name: 'Carol', scores: [88] });
console.log(original.length); // 2 — untouched

// But mutating a nested object affects BOTH
copy[0].name = 'CHANGED';
console.log(original[0].name); // 'CHANGED' ← original was affected!

// Why? copy[0] and original[0] point to the same object in memory
console.log(copy[0] === original[0]); // true
```

To deep-copy, use `structuredClone` (modern) or `JSON.parse(JSON.stringify(...))` (older):

```js
const deepCopy = structuredClone(original);
deepCopy[0].name = 'CHANGED';
console.log(original[0].name); // 'Alice' — safe
```

---

### 2. Spread creates a new array — it does NOT freeze or protect it

```js
const source = [1, 2, 3];
const copy   = [...source];

copy.push(4);          // fine — copy is a new array
source.push(99);       // fine — source is unchanged copy-wise

console.log(source);   // [1, 2, 99]
console.log(copy);     // [1, 2, 3, 4]
```

---

### 3. Spreading a non-iterable throws a TypeError

```js
const num = 42;
const broken = [...num]; // TypeError: num is not iterable

// Only iterables work: arrays, strings, Sets, Maps, NodeLists, arguments
const chars   = [..."hello"];            // ['h', 'e', 'l', 'l', 'o']
const setArr  = [...new Set([1, 2, 1])]; // [1, 2]  — deduplication trick
const mapKeys = [...new Map([['a', 1], ['b', 2]]).keys()]; // ['a', 'b']
```

---

### 4. Spread vs `concat` — behaviour with nested arrays

`concat` with a nested array adds the inner array as one element. Spread does too, unless you spread the inner array as well.

```js
const a = [1, 2];
const b = [[3, 4], 5];

console.log(a.concat(b));  // [1, 2, [3, 4], 5]  ← nested array stays nested
console.log([...a, ...b]); // [1, 2, [3, 4], 5]  ← same — spread only one level

// To flatten the nested array too:
console.log([...a, ...b.flat()]); // [1, 2, 3, 4, 5]
```

---

### 5. `arguments` object needs spread — it is not a real array

Inside a regular function, `arguments` is array-like but lacks array methods. Convert it with spread:

```js
function sum() {
  // arguments.reduce(...)  ← TypeError — not a real array
  return [...arguments].reduce((acc, n) => acc + n, 0);
}

console.log(sum(1, 2, 3, 4)); // 10

// Better: use rest parameters instead (topic 33 adjacent)
function sum2(...nums) {
  return nums.reduce((acc, n) => acc + n, 0);
}
```

---

### 6. Order matters when spreading for overrides

When spread is used to build an array with potential duplicates or overrides, position determines what wins.

```js
const defaults  = ['en', 'metric', 'light'];
const userPrefs = ['fr'];

// user preference comes last — it is at index 0 but defaults fill the rest
// this is more relevant for objects (topic 37), but for arrays:
const config = [...defaults, ...userPrefs]; // ['en', 'metric', 'light', 'fr']

// if your logic depends on position, think carefully about which array goes first
```

---

## Common mistakes

```js
// ❌ Forgetting spread when passing array to Math.max
Math.max([1, 2, 3]);    // NaN
Math.max(...[1, 2, 3]); // 3  ✅

// ❌ Thinking spread is a deep copy
const arr = [{ x: 1 }];
const copy = [...arr];
copy[0].x = 99;
console.log(arr[0].x); // 99 — original mutated!

// ❌ Spreading inside a non-array context
const wrong = ...arr;   // SyntaxError — spread only works inside [] {} or function calls

// ❌ Using spread to deduplicate (spread doesn't deduplicate)
const dupes   = [1, 1, 2, 2, 3];
const nodupes = [...dupes]; // [1, 1, 2, 2, 3] — still has duplicates
// Correct way:
const unique = [...new Set(dupes)]; // [1, 2, 3] ✅

// ❌ Spreading a large array into a function with argument limits
// Browsers have a maximum argument count (~65,536). For huge arrays, use reduce.
const hugeArray = new Array(100000).fill(1);
Math.max(...hugeArray); // may throw "Maximum call stack size exceeded"
// Use: hugeArray.reduce((a, b) => Math.max(a, b), -Infinity);
```

---

## Frequently asked questions

**Q: What is the difference between spread (`...arr`) and rest parameters (`...args`)?**
Same syntax, opposite purpose. **Spread** expands an array into individual values. **Rest** collects individual values into an array. Spread is used in expressions; rest is used in function parameter lists. (Rest parameters are covered as part of the functions topics.)

**Q: Does spread work with strings?**
Yes — a string is iterable. `[...'hello']` gives `['h', 'e', 'l', 'l', 'o']`. Useful for character-level operations.

**Q: Can I spread multiple arrays in one expression?**
Yes, as many as you want: `[...a, ...b, ...c, ...d]`.

**Q: Is spread slower than `concat`?**
For small arrays, negligibly. For very large arrays (hundreds of thousands of elements), `concat` can be faster because it's a native optimised method. In practice, this rarely matters.

**Q: Does `[...arr]` work the same as `arr.slice()`?**
Yes, both produce a shallow copy. `[...arr]` is considered more readable and idiomatic in modern code.

---

## Practice exercises

### Exercise 1 — easy

Given these arrays:

```js
const fruits   = ['apple', 'banana', 'cherry'];
const veggies  = ['carrot', 'broccoli'];
const grains   = ['rice', 'oats'];
```

Using only spread (no `concat`, no `push`):

1. Create `allFoods` — all three arrays merged in order
2. Create `withExtras` — `allFoods` but with `'mango'` at the start and `'quinoa'` at the end
3. Create `fruitsOnly` — a copy of `fruits` and confirm it is a different reference (`fruitsOnly !== fruits`)
4. Find the alphabetically last item across all three arrays using `Math.max` — wait, that won't work. Find the last item alphabetically by spreading into a function that uses `localeCompare`. (Hint: write a small helper.)

---

### Exercise 2 — medium

You are managing a playlist app. All operations must be **non-mutating** (the original playlist must never change).

```js
const playlist = [
  { id: 1, title: 'Bohemian Rhapsody', artist: 'Queen',   duration: 354 },
  { id: 2, title: 'Hotel California',  artist: 'Eagles',  duration: 391 },
  { id: 3, title: 'Stairway to Heaven',artist: 'Led Zep', duration: 482 },
  { id: 4, title: 'Smells Like Teen Spirit', artist: 'Nirvana', duration: 301 },
];
```

Implement these functions using spread (and whatever other methods you need):

1. `addTrack(playlist, track)` — returns new playlist with `track` appended
2. `removeTrack(playlist, id)` — returns new playlist without the track with that id
3. `insertTrackAt(playlist, track, index)` — inserts `track` at the given index, shifting others right
4. `moveToTop(playlist, id)` — returns new playlist with that track moved to position 0

---

### Exercise 3 — hard

Build a `Pipeline` class that chains data transformations. Each transformation is a function. The class stores the data immutably — every operation returns a new `Pipeline` instance (never mutates the existing one).

```js
// Expected usage:
const result = new Pipeline([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  .filter(n => n % 2 === 0)       // [2, 4, 6, 8, 10]
  .map(n => n * n)                 // [4, 16, 36, 64, 100]
  .take(3)                         // [4, 16, 36]  ← first N elements
  .append([999, 1000])             // [4, 16, 36, 999, 1000]
  .prepend([0, 1])                 // [0, 1, 4, 16, 36, 999, 1000]
  .value();                        // returns the final array

console.log(result); // [0, 1, 4, 16, 36, 999, 1000]
```

Implement the `Pipeline` class with: `filter`, `map`, `take`, `append`, `prepend`, and `value`. Each method except `value` must return a new `Pipeline` instance. Use spread for `append` and `prepend`.

---

## Quick reference — cheat sheet

```js
// ── copy ──────────────────────────────────────────────
const copy = [...arr];                     // shallow copy

// ── merge ─────────────────────────────────────────────
const merged = [...a, ...b];
const combo  = [0, ...a, 99, ...b, -1];   // insert items anywhere

// ── function arguments ────────────────────────────────
Math.max(...arr);
fn(...args);

// ── convert iterables to arrays ───────────────────────
[...'hello']              // ['h','e','l','l','o']
[...new Set([1,1,2])]     // [1, 2]  ← deduplication
[...nodeList]             // real array from DOM NodeList
[...map.entries()]        // [[k,v], ...]

// ── spread does NOT ───────────────────────────────────
// deep copy nested objects  → use structuredClone()
// deduplicate               → use new Set()
// flatten nested arrays     → use .flat()
```

---

## Connected topics

| Topic | Why it connects |
|---|---|
| [25-array-non-mutation-methods.md](25-array-non-mutation-methods.md) | `slice` and `concat` are the pre-spread equivalents |
| [33-array-destructuring.md](33-array-destructuring.md) | Rest in destructuring uses the same `...` syntax |
| [37-spread-objects.md](37-spread-objects.md) | Same operator applied to objects — same shallow-copy gotcha |
| [29-reduce.md](29-reduce.md) | Use `reduce` instead of spread for huge arrays in argument position |
| [23-array-basics.md](23-array-basics.md) | Arrays by reference — why copying matters |
