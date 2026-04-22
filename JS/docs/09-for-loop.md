# 09 — The `for` Loop

## What is this?
A `for` loop is a way to run a block of code a set number of times. Think of it like a factory conveyor belt: you set how many items to process, the belt moves one step at a time, does the job, and stops when it's done. It's the most precise loop in JavaScript — you control exactly where it starts, when it stops, and how it steps.

## Why does it matter?
Loops are how programs handle repetition without you writing the same code over and over. The `for` loop is the foundation — once you understand its three-part structure deeply, every other loop (while, for...of, forEach) will make immediate sense. It also unlocks working with arrays by index, which is essential for nearly every real algorithm.

---

## Syntax

```js
for (initialisation; condition; update) {
  // body — runs every iteration
}
```

```js
//       ┌─ runs once before loop starts
//       │         ┌─ checked before EVERY iteration — if false, loop stops
//       │         │          ┌─ runs after EVERY iteration
//       ↓         ↓          ↓
for (let i = 0; i < 5; i++) {
  console.log(i);   // 0, 1, 2, 3, 4
}
```

---

## How it works — step by step

Given `for (let i = 0; i < 5; i++)`:

1. **Initialisation** — `let i = 0` runs once. `i` starts at 0.
2. **Condition check** — `i < 5` is evaluated. If `true`, run the body. If `false`, exit the loop.
3. **Body runs** — `console.log(i)` executes.
4. **Update** — `i++` runs. `i` is now 1.
5. **Back to step 2** — check condition again. Repeat until condition is `false`.
6. When `i` reaches 5, `i < 5` is `false` → loop exits.

```
Iteration 1: i=0, 0<5=true  → body runs → i becomes 1
Iteration 2: i=1, 1<5=true  → body runs → i becomes 2
Iteration 3: i=2, 2<5=true  → body runs → i becomes 3
Iteration 4: i=3, 3<5=true  → body runs → i becomes 4
Iteration 5: i=4, 4<5=true  → body runs → i becomes 5
             i=5, 5<5=false → STOP
```

Total iterations: **5**. Range: **0 to 4** (not 0 to 5).

---

## Deep dive — everything you need to know

### 1. Loop variable scope — always use `let`, never `var`

```js
// WRONG — var is function-scoped, leaks outside the loop
for (var i = 0; i < 3; i++) { }
console.log(i);   // 3 — i leaked out! This is a real source of bugs.

// RIGHT — let is block-scoped, dies when the loop ends
for (let i = 0; i < 3; i++) { }
console.log(i);   // ReferenceError: i is not defined — correct behaviour
```

This matters especially in async code (covered in topic 52) — `var` in loops is responsible for one of the most infamous JS bugs.

---

### 2. Accessing array elements by index

The most common use of `for` loops — iterating over an array with index access:

```js
const products = ["Keyboard", "Mouse", "Monitor", "Headphones"];

for (let i = 0; i < products.length; i++) {
  console.log(`${i + 1}. ${products[i]}`);
}
// 1. Keyboard
// 2. Mouse
// 3. Monitor
// 4. Headphones
```

**Why `i < products.length` and not `i <= products.length`?**

Array indices go from `0` to `length - 1`. If length is 4, valid indices are 0, 1, 2, 3. Using `<=` would try to access `products[4]` which is `undefined`.

```js
const arr = ["a", "b", "c"];   // length = 3, indices = 0, 1, 2
arr[3]   // undefined — index 3 doesn't exist
```

---

### 3. Counting backwards — decrementing loop

```js
const countdown = [5, 4, 3, 2, 1];

for (let i = countdown.length - 1; i >= 0; i--) {
  console.log(countdown[i]);
}
// 1, 2, 3, 4, 5 — reversed

// Or count a variable down directly
for (let seconds = 5; seconds >= 1; seconds--) {
  console.log(`${seconds}...`);
}
console.log("Launch!");
```

---

### 4. Stepping by more than 1

The update expression can be anything — not just `i++`:

```js
// Step by 2 — only even numbers
for (let i = 0; i <= 10; i += 2) {
  console.log(i);   // 0, 2, 4, 6, 8, 10
}

// Step by 10
for (let i = 0; i <= 100; i += 10) {
  console.log(i);   // 0, 10, 20, ... 100
}

// Multiply each step
for (let i = 1; i <= 1024; i *= 2) {
  console.log(i);   // 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024
}
```

---

### 5. Nested loops — iterating 2D structures

A loop inside a loop. The inner loop runs **completely** for every single iteration of the outer loop.

```js
// Multiplication table (3x3)
for (let row = 1; row <= 3; row++) {
  for (let col = 1; col <= 3; col++) {
    process.stdout.write(`${row * col}\t`);   // \t is a tab character
  }
  console.log();   // new line after each row
}
// 1  2  3
// 2  4  6
// 3  6  9

// Total iterations: outer(3) × inner(3) = 9
```

**Performance note:** With large datasets, nested loops become slow fast. O(n²) complexity. Always ask if there's a flatter solution.

```js
// Building a seating chart
const rows  = ["A", "B", "C"];
const seats = [1, 2, 3, 4];

for (let r = 0; r < rows.length; r++) {
  for (let s = 0; s < seats.length; s++) {
    console.log(`Seat ${rows[r]}${seats[s]}`);
  }
}
// Seat A1, Seat A2, Seat A3, Seat A4, Seat B1 ...
```

---

### 6. The three parts are all optional

Any or all of the three parts can be omitted (though this is unusual):

```js
// Omit initialisation — if i is already declared outside
let i = 0;
for (; i < 3; i++) {
  console.log(i);
}

// Omit update — update inside body instead
for (let i = 0; i < 3; ) {
  console.log(i);
  i++;   // update here
}

// Omit all three — infinite loop (requires break to exit)
for (;;) {
  console.log("infinite");
  break;   // exits after first iteration
}
```

The `for(;;)` infinite loop is a real pattern in event loops, game loops, and server processes — they rely on a `break` inside the body to exit.

---

### 7. Computed values inside the loop — cache `length`

In performance-sensitive code, avoid recalculating `array.length` on every iteration:

```js
const bigArray = new Array(1_000_000).fill(0);

// Slightly less optimal — .length is accessed every iteration
for (let i = 0; i < bigArray.length; i++) { }

// Optimal — cache length in initialisation
for (let i = 0, len = bigArray.length; i < len; i++) { }
```

In practice this micro-optimisation rarely matters for arrays under 100k items — modern JS engines optimise it — but you'll see this pattern in older or performance-critical codebases.

---

### 8. Off-by-one errors — the most common loop bug

An off-by-one error is when your loop runs one too many or one too few times.

```js
const items = ["a", "b", "c"];   // 3 items, indices 0, 1, 2

// Too many — accesses items[3] = undefined
for (let i = 0; i <= items.length; i++) {
  console.log(items[i]);   // a, b, c, undefined
}

// Too few — misses the last item
for (let i = 0; i < items.length - 1; i++) {
  console.log(items[i]);   // a, b  (misses c)
}

// Correct
for (let i = 0; i < items.length; i++) {
  console.log(items[i]);   // a, b, c
}
```

**Mental model:** If you want `n` items (0-indexed), the condition is `i < n` (not `i <= n`).

---

### 9. Modifying an array while looping — dangerous

```js
const numbers = [1, 2, 3, 4, 5];

// DANGEROUS — removing items while iterating shifts indices
for (let i = 0; i < numbers.length; i++) {
  if (numbers[i] % 2 === 0) {
    numbers.splice(i, 1);   // removes item at i, array shrinks, i skips next item
  }
}
console.log(numbers);   // [1, 3, 5] — looks right but only by luck in this case

// SAFE — iterate backwards when splicing
for (let i = numbers.length - 1; i >= 0; i--) {
  if (numbers[i] % 2 === 0) {
    numbers.splice(i, 1);   // removing from the end doesn't affect unvisited indices
  }
}

// BETTER — use filter instead (topic 28)
const odds = numbers.filter(n => n % 2 !== 0);
```

---

## Example 1 — basic

```js
// Building a simple times table
const multiplier = 7;

console.log(`--- ${multiplier} Times Table ---`);

for (let i = 1; i <= 10; i++) {
  const result = multiplier * i;
  console.log(`${multiplier} × ${i} = ${result}`);
}

// 7 × 1 = 7
// 7 × 2 = 14
// ...
// 7 × 10 = 70
```

---

## Example 2 — real world

```js
// Processing an order list and calculating totals
const orderItems = [
  { name: "Keyboard",   price: 89.99, qty: 1 },
  { name: "Mouse",      price: 29.99, qty: 2 },
  { name: "USB Hub",    price: 19.99, qty: 3 },
  { name: "Monitor",    price: 299.99, qty: 1 },
];

let subtotal    = 0;
let totalItems  = 0;

console.log("Order Summary:");
console.log("─".repeat(40));

for (let i = 0; i < orderItems.length; i++) {
  const item      = orderItems[i];
  const lineTotal = item.price * item.qty;

  subtotal   += lineTotal;
  totalItems += item.qty;

  console.log(
    `${(i + 1)}. ${item.name.padEnd(12)} × ${item.qty}  $${lineTotal.toFixed(2)}`
  );
}

console.log("─".repeat(40));
console.log(`Total items: ${totalItems}`);
console.log(`Subtotal:    $${subtotal.toFixed(2)}`);
console.log(`Tax (18%):   $${(subtotal * 0.18).toFixed(2)}`);
console.log(`Total:       $${(subtotal * 1.18).toFixed(2)}`);
```

---

## Common mistakes

### 1. Using `<=` instead of `<` with array length
```js
const arr = [10, 20, 30];

// WRONG — tries to access arr[3] which is undefined
for (let i = 0; i <= arr.length; i++) {
  console.log(arr[i]);   // 10, 20, 30, undefined
}

// RIGHT
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);   // 10, 20, 30
}
```

### 2. Forgetting that the loop variable is scoped — reusing `i`
```js
// WRONG — var i leaks, causes confusion in nested loops
for (var i = 0; i < 3; i++) {
  for (var i = 0; i < 2; i++) {   // same i! overwrites outer i
    console.log("inner", i);
  }
  // outer i is now 2 after inner loop, so outer loop ends after one outer iteration
}

// RIGHT — let is scoped, each loop has its own i
for (let i = 0; i < 3; i++) {
  for (let i = 0; i < 2; i++) {   // different i — inner is block-scoped
    console.log("inner", i);
  }
}
// Even better — use different variable names: i, j, k
```

### 3. Infinite loop — condition never becomes false
```js
// WRONG — i is never updated, loop never ends, browser/Node freezes
for (let i = 0; i < 5; ) {
  console.log(i);
  // forgot i++ here!
}

// RIGHT
for (let i = 0; i < 5; i++) {
  console.log(i);
}
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — Iterating with index AND value at the same time
```js
const fruits = ["apple", "banana", "cherry"];

// Access both the index and the value cleanly
for (let i = 0; i < fruits.length; i++) {
  console.log(`[${i}] ${fruits[i]}`);
}
// [0] apple
// [1] banana
// [2] cherry

// for...of gives value but not index (topic 11)
// forEach gives both (topic 26)
// The classic for loop is the only one where you naturally have both
```

### Trick 2 — Using loop index to access multiple arrays in sync
```js
const names   = ["Parsh", "Amit", "Neha"];
const scores  = [92, 78, 88];
const passing = [true, true, true];

for (let i = 0; i < names.length; i++) {
  const status = passing[i] ? "PASS" : "FAIL";
  console.log(`${names[i]}: ${scores[i]} — ${status}`);
}
// Parsh: 92 — PASS
// Amit:  78 — PASS
// Neha:  88 — PASS
```

### Trick 3 — Loop doesn't run at all if condition starts false
```js
const items = [];   // empty array

for (let i = 0; i < items.length; i++) {
  console.log(items[i]);   // never runs — 0 < 0 is false immediately
}
console.log("Done");   // runs — loop was simply skipped
```

### Trick 4 — Building a string or array with a loop
```js
// Building a string progressively
let output = "";
for (let i = 1; i <= 5; i++) {
  output += i;   // "1", "12", "123", "1234", "12345"
}
console.log(output);   // "12345"

// Building an array progressively
const squares = [];
for (let i = 1; i <= 5; i++) {
  squares.push(i * i);
}
console.log(squares);   // [1, 4, 9, 16, 25]
```

### Trick 5 — `break` and `continue` in for loops (preview of topic 13)
```js
// break — exit the loop entirely
for (let i = 0; i < 10; i++) {
  if (i === 5) break;
  console.log(i);   // 0, 1, 2, 3, 4
}

// continue — skip this iteration, move to next
for (let i = 0; i < 10; i++) {
  if (i % 2 === 0) continue;   // skip even numbers
  console.log(i);               // 1, 3, 5, 7, 9
}
```

---

## Practice exercises

### Exercise 1 — easy
Write a `for` loop that:
1. Counts from 1 to 20
2. For multiples of 3, logs `"Fizz"`
3. For multiples of 5, logs `"Buzz"`
4. For multiples of both 3 and 5, logs `"FizzBuzz"`
5. For everything else, logs the number

(This is the classic FizzBuzz — a real interview question. Figure it out from first principles.)
```js
// Write your code here
```

---

### Exercise 2 — medium
You have this array of transactions:

```js
const transactions = [
  { id: "T001", type: "credit",  amount: 5000 },
  { id: "T002", type: "debit",   amount: 1200 },
  { id: "T003", type: "debit",   amount: 300  },
  { id: "T004", type: "credit",  amount: 2500 },
  { id: "T005", type: "debit",   amount: 800  },
  { id: "T006", type: "credit",  amount: 1000 },
];
```

Using only a `for` loop (no array methods), calculate and log:
1. Total credits
2. Total debits
3. Net balance (credits - debits)
4. The number of each type
5. The ID of the largest single transaction

```js
// Write your code here
```

---

### Exercise 3 — hard
Build a seating reservation system using nested `for` loops.

```js
const rows      = 4;
const seatsPerRow = 6;
const reserved  = ["A3", "A4", "B1", "C5", "C6", "D2"];
```

Generate and log a seating chart where:
- Rows are labelled A, B, C, D (use `String.fromCharCode(65 + rowIndex)` to get the letter)
- Seats are numbered 1 through 6
- Reserved seats show `[X]`, available seats show `[ ]`
- Each row is printed on one line, like: `A: [ ] [ ] [X] [X] [ ] [ ]`

After the chart, log:
- Total seats
- Reserved count
- Available count
- Availability percentage (e.g. `"75.00% available"`)

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Structure:**
```js
for (let i = 0; i < n; i++) { }       // count up n times
for (let i = n - 1; i >= 0; i--) { }  // count down
for (let i = 0; i < arr.length; i++) { } // iterate array
```

**Condition rules:**
```
i < arr.length      → visits indices 0 to length-1 (correct)
i <= arr.length     → visits one too many (off-by-one bug)
i < arr.length - 1  → visits one too few (misses last item)
```

**Step variations:**
```
i++     → step by 1
i--     → step back by 1
i += 2  → step by 2
i *= 2  → double each step
```

**Scope:**
```
let i  → block-scoped, dies after loop (always use this)
var i  → function-scoped, leaks out (never use in loops)
```

**Key rules:**
- Arrays are 0-indexed. Loop from `0` to `< length`.
- Use `let`, never `var` for loop variables.
- Nested loops: total iterations = outer × inner. Can be expensive.
- Iterating backwards (i--) is safe when using `splice` inside the loop.
- If condition is false before first check — loop body never runs. No error.

---

## Connected topics
- **10 — while and do...while** — loops without a fixed counter; use when you don't know upfront how many iterations you need
- **13 — break and continue** — controlling loop flow mid-iteration
- **11 — for...of** — cleaner syntax when you only need values, not indices
- **24 — Array mutation methods** — `push`, `splice` etc. are what you use inside loops to build/modify arrays
