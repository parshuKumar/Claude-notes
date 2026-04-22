# 13 — break and continue

## What is this?

`break` and `continue` are loop control statements that let you interrupt the normal flow of a loop.

- **`break`** — exits the loop entirely, immediately. Code after the loop runs next.
- **`continue`** — skips the rest of the current iteration and jumps to the next one.

```js
for (const item of items) {
  if (someCondition) break;     // stop the whole loop
  if (otherCondition) continue; // skip this item, go to next
  // normal processing
}
```

They work in `for`, `for...of`, `for...in`, `while`, and `do...while`.

---

## Why does it matter?

Without `break` and `continue`, you'd have to restructure logic with extra flags or nested `if` statements. They make intent explicit and keep loops clean.

**Without `break`:**
```js
let foundUser = null;
let keepSearching = true;

for (const user of userList) {
  if (keepSearching && user.id === targetId) {
    foundUser = user;
    keepSearching = false;
  }
}
```

**With `break`:**
```js
let foundUser = null;

for (const user of userList) {
  if (user.id === targetId) {
    foundUser = user;
    break;  // done — no need to check the rest
  }
}
```

Cleaner, faster (stops early), and the intent is obvious.

---

## Syntax

### break

```js
for (...) {
  if (condition) {
    break;  // exits the innermost loop immediately
  }
}
```

### continue

```js
for (...) {
  if (condition) {
    continue;  // skip to next iteration
  }
  // this only runs if condition was false
}
```

### Labeled break (advanced)

```js
outerLoop: for (...) {
  for (...) {
    if (condition) {
      break outerLoop;  // exits the OUTER loop, not just the inner one
    }
  }
}
```

Labels are rarely needed but exist for nested loop escape.

---

## How it works — line by line

### break example

```js
const transactions = [120, 45, 890, 2300, 15, 670];
const FRAUD_THRESHOLD = 1000;

let flaggedAmount = null;

for (const amount of transactions) {
  console.log("Checking:", amount);
  if (amount > FRAUD_THRESHOLD) {
    flaggedAmount = amount;
    break;
  }
}

console.log("Flagged:", flaggedAmount);
```

**Execution trace:**
```
Iteration 1: amount = 120   → 120 > 1000? No  → continues
Iteration 2: amount = 45    → 45 > 1000? No   → continues
Iteration 3: amount = 890   → 890 > 1000? No  → continues
Iteration 4: amount = 2300  → 2300 > 1000? Yes → flaggedAmount = 2300, BREAK
Loop exits. 15 and 670 are never checked.
console.log → "Flagged: 2300"
```

### continue example

```js
const orderAmounts = [0, 59.99, 0, 120.00, 0, 34.50];
let revenueTotal = 0;

for (const amount of orderAmounts) {
  if (amount === 0) {
    continue;  // skip cancelled/empty orders
  }
  revenueTotal += amount;
}

console.log("Total revenue:", revenueTotal);  // 214.49
```

**Execution trace:**
```
Iteration 1: amount = 0      → continue → skip
Iteration 2: amount = 59.99  → add to total
Iteration 3: amount = 0      → continue → skip
Iteration 4: amount = 120.00 → add to total
Iteration 5: amount = 0      → continue → skip
Iteration 6: amount = 34.50  → add to total
Loop ends → revenueTotal = 214.49
```

---

## Example 1 — basic

### Finding the first match with `break`

```js
const employeeIds = ["E001", "E002", "E003", "E004", "E005"];
const targetId = "E003";
let position = -1;

for (let i = 0; i < employeeIds.length; i++) {
  if (employeeIds[i] === targetId) {
    position = i;
    break;
  }
}

if (position !== -1) {
  console.log(`Found ${targetId} at index ${position}`);
} else {
  console.log(`${targetId} not found`);
}
// "Found E003 at index 2"
```

### Skipping items with `continue`

```js
const productList = [
  { name: "Widget A", isDiscontinued: false },
  { name: "Widget B", isDiscontinued: true },
  { name: "Widget C", isDiscontinued: false },
  { name: "Widget D", isDiscontinued: true },
  { name: "Widget E", isDiscontinued: false },
];

console.log("Active products:");
for (const product of productList) {
  if (product.isDiscontinued) {
    continue;
  }
  console.log(" -", product.name);
}
// Active products:
//  - Widget A
//  - Widget C
//  - Widget E
```

---

## Example 2 — real world

### Search with early exit

```js
const flightSchedule = [
  { flightId: "FL-101", destination: "London", seatsAvailable: 0 },
  { flightId: "FL-102", destination: "Paris", seatsAvailable: 3 },
  { flightId: "FL-103", destination: "London", seatsAvailable: 12 },
  { flightId: "FL-104", destination: "London", seatsAvailable: 1 },
];

function findFirstAvailableFlight(schedule, destination) {
  for (const flight of schedule) {
    if (flight.destination !== destination) continue;
    if (flight.seatsAvailable === 0) continue;
    return flight;  // return also exits the loop — acts like break
  }
  return null;
}

const result = findFirstAvailableFlight(flightSchedule, "London");
console.log(result);
// { flightId: "FL-103", destination: "London", seatsAvailable: 12 }
```

Note: inside a function, `return` naturally exits the loop AND the function. You don't always need an explicit `break`.

### Processing with validation skips

```js
const rawFormSubmissions = [
  { email: "alice@mail.com", age: 28, score: 85 },
  { email: "", age: 22, score: 91 },           // invalid — no email
  { email: "bob@mail.com", age: 17, score: 76 },// invalid — underage
  { email: "carol@mail.com", age: 31, score: 94 },
  { email: "dave@mail.com", age: null, score: 88 }, // invalid — no age
];

const validSubmissions = [];

for (const submission of rawFormSubmissions) {
  if (!submission.email) continue;       // skip: missing email
  if (submission.age === null) continue; // skip: missing age
  if (submission.age < 18) continue;     // skip: underage

  validSubmissions.push(submission);
}

console.log("Valid submissions:", validSubmissions.length);  // 2
```

This is the **guard clause** pattern applied inside a loop — keep processing only what passes all checks.

### Labeled break — escaping nested loops

```js
const seatMap = [
  ["taken",  "taken",  "free",   "taken"],
  ["taken",  "taken",  "taken",  "taken"],
  ["taken",  "free",   "taken",  "free" ],
];

let targetRow = -1;
let targetSeat = -1;

searchLoop: for (let row = 0; row < seatMap.length; row++) {
  for (let seat = 0; seat < seatMap[row].length; seat++) {
    if (seatMap[row][seat] === "free") {
      targetRow = row;
      targetSeat = seat;
      break searchLoop;  // exits BOTH loops immediately
    }
  }
}

console.log(`First free seat: row ${targetRow}, seat ${targetSeat}`);
// "First free seat: row 0, seat 2"
```

Without the label, `break` would only exit the inner loop, and the outer loop would continue searching.

---

## Tricky things you'll encounter in the real world

### 1. `break` only exits the innermost loop

This is the most common `break` mistake:

```js
const grid = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];

for (const row of grid) {
  for (const cell of row) {
    if (cell === 5) {
      console.log("Found 5!");
      break;  // ⚠️ only exits the INNER loop
    }
  }
  console.log("Still in outer loop");  // this still runs!
}
```

If you mean to exit everything, you need a labeled break, or restructure into a function and use `return`.

### 2. `continue` in a `while` loop — update must still happen

```js
let count = 0;

while (count < 5) {
  count++;         // ✅ update is BEFORE continue
  if (count === 3) continue;
  console.log(count);
}
// 1, 2, 4, 5
```

If the increment were after `continue`, you'd get an infinite loop:

```js
let count = 0;

while (count < 5) {
  if (count === 3) continue;  // ⚠️ infinite loop — count never increments past 3
  count++;
  console.log(count);
}
```

The `continue` jumps back to the condition check but `count` is still 3, the condition `3 < 5` is still true, forever.

### 3. `break` and `continue` don't work in `.forEach()`

```js
const numbers = [1, 2, 3, 4, 5];

numbers.forEach(num => {
  if (num === 3) break;     // ❌ SyntaxError: Illegal break statement
  if (num === 2) continue;  // ❌ SyntaxError: Illegal continue statement
});
```

`.forEach()` takes a callback function. `break` and `continue` only work inside loops, not callbacks. Switch to `for...of` when you need them.

### 4. `return` inside a loop acts like `break`

```js
function findExpiredItem(inventory) {
  for (const item of inventory) {
    if (item.isExpired) {
      return item;   // exits the loop AND the function
    }
  }
  return null;  // reached only if no expired item found
}
```

`return` is often the cleanest way to stop a loop inside a function — you get the early exit AND you immediately hand back the result.

### 5. Overusing `continue` can hurt readability

```js
// ❌ Multiple continues — hard to track what actually gets processed
for (const order of orders) {
  if (!order.isPaid) continue;
  if (order.isRefunded) continue;
  if (order.amount < 10) continue;
  if (!order.customerId) continue;
  processOrder(order);
}

// ✅ Better — invert the condition, one clear gate
for (const order of orders) {
  const isValid = order.isPaid
    && !order.isRefunded
    && order.amount >= 10
    && order.customerId;

  if (isValid) {
    processOrder(order);
  }
}
```

Multiple `continue` statements at the top of a loop are readable. But when the logic gets complex, a single `isValid` check is clearer about the full condition required.

### 6. Labeled `continue` — exists but is rare

```js
outerLoop: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) continue outerLoop;  // skips to next iteration of OUTER loop
    console.log(i, j);
  }
}
// 0 0
// 1 0
// 2 0
```

`continue outerLoop` when `j === 1` skips the rest of the inner loop AND the rest of the outer iteration, jumping directly to `i++`. You'll almost never write this, but you may read it.

---

## Common mistakes

### Mistake 1: Expecting `break` to exit all nested loops

```js
// ❌ break only exits the inner loop
for (const category of categories) {
  for (const item of category.items) {
    if (item.id === searchId) {
      foundItem = item;
      break;  // still continues to next category!
    }
  }
}

// ✅ Option 1: labeled break
search: for (const category of categories) {
  for (const item of category.items) {
    if (item.id === searchId) {
      foundItem = item;
      break search;
    }
  }
}

// ✅ Option 2: extract to a function and use return
function findItem(categories, searchId) {
  for (const category of categories) {
    for (const item of category.items) {
      if (item.id === searchId) return item;
    }
  }
  return null;
}
```

### Mistake 2: Infinite loop from `continue` before update in `while`

```js
// ❌ Classic infinite loop
let i = 0;
while (i < 10) {
  if (i % 2 === 0) continue;  // i never changes → infinite loop
  i++;
}

// ✅ Update before continue
let i = 0;
while (i < 10) {
  i++;
  if (i % 2 === 0) continue;
  console.log(i);
}
```

### Mistake 3: Using `break`/`continue` in forEach

```js
// ❌ SyntaxError
items.forEach(item => {
  if (item.isDone) continue;
});

// ✅ Switch to for...of
for (const item of items) {
  if (item.isDone) continue;
}
```

---

## Practice exercises

### Exercise 1 — easy

You have a list of numbers:

```js
const measurements = [4, 7, -3, 12, -8, 5, 19, -1, 6, 23];
```

1. Using a `for...of` loop with `continue`, log only the **positive** numbers (skip negatives and zero)
2. Using a separate `for...of` loop with `break`, find and log the **first number greater than 15** — then stop searching

---

### Exercise 2 — medium

You work for a streaming platform. Here is a list of videos in a playlist:

```js
const playlist = [
  { title: "Intro to CSS", duration: 12, isWatched: true, isFree: true },
  { title: "Box Model", duration: 18, isWatched: false, isFree: true },
  { title: "Flexbox Basics", duration: 24, isWatched: false, isFree: false },
  { title: "Grid Layout", duration: 31, isWatched: false, isFree: false },
  { title: "CSS Variables", duration: 15, isWatched: false, isFree: true },
  { title: "Animations", duration: 27, isWatched: false, isFree: false },
];
```

Write code that:
1. Uses `continue` to **skip already watched** videos
2. Uses `break` to **stop at the first premium (not free) video** — the user hasn't paid
3. Logs each video that the user can and should watch in the format:
   ```
   ▶ Box Model (18 min)
   ```
4. After the loop, logs how many videos the user can watch before hitting the paywall

---

### Exercise 3 — hard

You are building a **seat booking engine** for a cinema. The cinema has a seating grid:

```js
const cinemaSeats = [
  { row: "A", seats: [
    { number: 1, isBooked: true  },
    { number: 2, isBooked: true  },
    { number: 3, isBooked: false },
    { number: 4, isBooked: false },
  ]},
  { row: "B", seats: [
    { number: 1, isBooked: true  },
    { number: 2, isBooked: false },
    { number: 3, isBooked: false },
    { number: 4, isBooked: true  },
  ]},
  { row: "C", seats: [
    { number: 1, isBooked: false },
    { number: 2, isBooked: false },
    { number: 3, isBooked: false },
    { number: 4, isBooked: false },
  ]},
];
```

Write a function `bookSeats(cinemaSeats, numberOfSeats)` that:

1. Uses **nested loops** to find the first row that has **enough consecutive free seats** to fit `numberOfSeats` people sitting together
2. Once found, "books" those seats (set `isBooked: true`) and returns an object:
   ```js
   { success: true, row: "B", seats: [2, 3], message: "Booked 2 seats in row B" }
   ```
3. If no row has enough consecutive free seats, returns:
   ```js
   { success: false, message: "No consecutive seats available for 2 people" }
   ```
4. Use `continue` to skip rows that don't have enough total free seats  
5. Use `break` (or a label) to stop searching once a booking is made

Test it:
```js
console.log(bookSeats(cinemaSeats, 2));  // should book row A seats 3,4 or row B seats 2,3
console.log(bookSeats(cinemaSeats, 4));  // should book row C seats 1–4
console.log(bookSeats(cinemaSeats, 5));  // should fail — no row has 5 consecutive free seats
```

---

## Quick reference

```
break and continue cheat sheet
─────────────────────────────────────────────────────
break           exits the innermost loop immediately
continue        skips rest of current iteration, moves to next
break label     exits the named outer loop
continue label  skips to next iteration of the named outer loop
return          inside a function — exits loop AND function
─────────────────────────────────────────────────────
WORKS IN        for / for...of / for...in / while / do...while
DOES NOT WORK   .forEach() / .map() / .filter() / .reduce()
─────────────────────────────────────────────────────
break only exits         the INNERMOST loop (use label for outer)
continue in while        update variable BEFORE continue or infinite loop
return = break + result  cleaner than break inside a search function
─────────────────────────────────────────────────────
PATTERN: early exit      for...of + break → find first match
PATTERN: filter in loop  for...of + continue → skip unwanted items
PATTERN: nested exit     labeled break or extract to function + return
```

---

## Connected topics

- **09 — for loop** — `break` and `continue` in classic `for` loops; index still increments correctly with `continue`
- **10 — while and do...while** — `continue` before update = infinite loop trap
- **11 — for...of** — the loop most commonly paired with `break`/`continue` in modern code
- **07 — conditionals** — guard clause pattern: `if (!valid) continue` inside loops mirrors early return in functions
- **Functions (14+)** — `return` inside a loop is often the cleanest alternative to `break`
- **Array methods (23+)** — `.find()`, `.findIndex()`, `.some()`, `.every()` often replace loops with `break` entirely
