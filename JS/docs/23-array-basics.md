# 23 — Array Basics

## What is this?
An array is an **ordered list of values** stored in a single variable. Think of it like a numbered shelf — each slot has an index starting at `0`, and you can put anything on any slot: numbers, strings, objects, even other arrays. The order is preserved, duplicates are allowed, and you can grow or shrink the list freely.

## Why does it matter?
Almost every real program works with collections of data — a list of users, a cart of products, a feed of posts, search results. Arrays are JavaScript's primary tool for holding and working with those collections. Every array method (`map`, `filter`, `reduce`, etc.) that comes in future topics is built on top of the basics covered here.

---

## Syntax

```js
// --- Creating arrays ---
const emptyArray    = [];                           // empty
const scores        = [88, 95, 72, 100];            // numbers
const productNames  = ["Keyboard", "Mouse", "Monitor"]; // strings
const mixed         = [42, "hello", true, null];    // mixed types (allowed, rarely useful)

// --- Accessing elements (zero-indexed) ---
console.log(productNames[0]);   // "Keyboard"   — first element
console.log(productNames[2]);   // "Monitor"    — third element
console.log(productNames[-1]);  // undefined    — negative indices don't work in JS

// --- Modifying elements ---
productNames[1] = "Trackpad";   // replace index 1
console.log(productNames);      // ["Keyboard", "Trackpad", "Monitor"]

// --- Length ---
console.log(productNames.length); // 3   — total number of elements
console.log(productNames[productNames.length - 1]); // "Monitor" — last element
```

---

## How it works — line by line

```js
const scores = [88, 95, 72, 100];
```
Creates an array literal with 4 elements. `const` means the variable `scores` always points to this array — but the array's *contents* can still change (add, remove, modify elements). `const` doesn't freeze the array.

```js
console.log(scores[0]);
```
Arrays are zero-indexed: the first element is at index `0`, the last is at index `length - 1`. Accessing an index that doesn't exist returns `undefined`, not an error.

```js
scores[1] = 99;
```
Direct assignment to an index. If the index exists, it's overwritten. If it's beyond the current length, JS creates a sparse array (holes — covered in Tricky section).

```js
console.log(scores.length);
```
`length` is a live property — it always equals the highest index + 1. It is NOT a method, so no `()`.

---

## Example 1 — basic: working with a list of scores

```js
const examScores = [72, 88, 95, 61, 100, 79];

// Access by index
console.log(examScores[0]);  // 72  — first score
console.log(examScores[5]);  // 79  — last score (index = length - 1)
console.log(examScores[6]);  // undefined — out of bounds, no error

// Length
console.log(examScores.length); // 6

// Last element without knowing the exact index
const lastScore = examScores[examScores.length - 1];
console.log(lastScore); // 79

// Modify a score
examScores[2] = 97;
console.log(examScores); // [72, 88, 97, 61, 100, 79]

// Loop through all scores
for (let i = 0; i < examScores.length; i++) {
  console.log("Score " + (i + 1) + ": " + examScores[i]);
}
// Score 1: 72
// Score 2: 88
// ...

// Find the highest score manually
let highest = examScores[0];
for (let i = 1; i < examScores.length; i++) {
  if (examScores[i] > highest) {
    highest = examScores[i];
  }
}
console.log("Highest:", highest); // 100
```

---

## Example 2 — real world: shopping cart

```js
// Cart as an array of product objects
const shoppingCart = [
  { id: 1, name: "Wireless Mouse",  price: 29.99, qty: 1 },
  { id: 2, name: "Mechanical Keyboard", price: 89.99, qty: 1 },
  { id: 3, name: "USB-C Hub",       price: 49.99, qty: 2 }
];

// Access a specific item
console.log(shoppingCart[0].name);   // "Wireless Mouse"
console.log(shoppingCart[2].qty);    // 2

// Total number of items in cart
console.log(shoppingCart.length);    // 3

// Update quantity of an item
shoppingCart[1].qty = 2;             // bought 2 keyboards
console.log(shoppingCart[1]);        // { id: 2, name: "Mechanical Keyboard", price: 89.99, qty: 2 }

// Calculate total price
let orderTotal = 0;
for (let i = 0; i < shoppingCart.length; i++) {
  const item = shoppingCart[i];
  orderTotal += item.price * item.qty;
}
console.log("Order total: $" + orderTotal.toFixed(2)); // Order total: $299.96

// Find an item by id
let foundItem = null;
for (let i = 0; i < shoppingCart.length; i++) {
  if (shoppingCart[i].id === 3) {
    foundItem = shoppingCart[i];
    break;
  }
}
console.log(foundItem); // { id: 3, name: "USB-C Hub", price: 49.99, qty: 2 }
```

---

## Example 3 — real world: 2D array (array of arrays)

```js
// A weekly schedule — each sub-array is one day's tasks
const weeklyTasks = [
  ["Standup", "Code review", "Deploy"],           // Monday    — index 0
  ["Standup", "Feature development"],             // Tuesday   — index 1
  ["Standup", "Client call", "Documentation"],   // Wednesday — index 2
  ["Standup", "Code review"],                    // Thursday  — index 3
  ["Standup", "Retrospective", "Planning"]       // Friday    — index 4
];

// Access a specific task
console.log(weeklyTasks[0][1]);    // "Code review"  — Monday's second task
console.log(weeklyTasks[2][2]);    // "Documentation" — Wednesday's third task

// How many tasks on Thursday?
console.log(weeklyTasks[3].length); // 2

// Print all tasks for each day
const dayNames = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"];

for (let day = 0; day < weeklyTasks.length; day++) {
  console.log(dayNames[day] + ":");
  for (let task = 0; task < weeklyTasks[day].length; task++) {
    console.log("  - " + weeklyTasks[day][task]);
  }
}
```

---

## Tricky things you'll encounter in the real world

### 1 — Arrays are objects — `typeof` returns `"object"`

```js
const tags = ["js", "node", "react"];

console.log(typeof tags);           // "object"  ← NOT "array"!
console.log(Array.isArray(tags));   // true  ✅ — the correct way to check

// This catches people off guard when validating inputs
function processItems(items) {
  if (typeof items === "object") {    // ❌ objects AND arrays both pass this
    // ...
  }

  if (Array.isArray(items)) {         // ✅ correct check
    // ...
  }
}
```

---

### 2 — `const` does NOT prevent array mutation

```js
const userRoles = ["viewer", "editor"];

// ❌ This throws — you can't reassign the variable
userRoles = ["admin"];               // TypeError: Assignment to constant variable

// ✅ But you CAN mutate the array's contents
userRoles[0] = "admin";             // works fine
userRoles.push("superuser");        // works fine
console.log(userRoles);             // ["admin", "editor", "superuser"]
```

`const` means: "this variable always points to the same array." It says nothing about the array's contents.

---

### 3 — Accessing out-of-bounds returns `undefined`, not an error

```js
const colours = ["red", "green", "blue"];

console.log(colours[10]);   // undefined — no error thrown
console.log(colours[-1]);   // undefined — negative indices not supported in JS

// This can hide bugs — check before using
const colour = colours[10];
console.log(colour.toUpperCase()); // ❌ TypeError: Cannot read properties of undefined
```

Always validate the index or check for `undefined` before using the value.

---

### 4 — Sparse arrays (holes) — created by skipping indices

```js
const sparse = [1, 2, 3];
sparse[7] = "far out";              // indices 3-6 are now holes

console.log(sparse.length);         // 8
console.log(sparse[4]);             // undefined
console.log(sparse);                // [1, 2, 3, <4 empty items>, 'far out']

// Holes behave unexpectedly in loops:
sparse.forEach((item) => console.log(item));
// 1, 2, 3, "far out"  — forEach SKIPS holes

for (let i = 0; i < sparse.length; i++) {
  console.log(sparse[i]);
// 1, 2, 3, undefined, undefined, undefined, undefined, "far out"  — classic for INCLUDES holes
}
```

Never create sparse arrays intentionally. They lead to subtle, hard-to-find bugs.

---

### 5 — Arrays are passed by reference

```js
const originalCart = [{ name: "Mouse", price: 29.99 }];
const cartCopy     = originalCart;          // ⚠️ NOT a copy — same reference

cartCopy.push({ name: "Keyboard", price: 79.99 });

console.log(originalCart.length); // 2  ← original was modified!

// To actually copy: use spread
const trueCopy = [...originalCart];
trueCopy.push({ name: "Monitor", price: 349.99 });
console.log(originalCart.length); // 2  ✅ original untouched
console.log(trueCopy.length);     // 3
```

Note: spread creates a **shallow copy** — the array structure is new, but object elements inside still share references.

---

### 6 — `length` can be set manually — it truncates the array

```js
const items = ["a", "b", "c", "d", "e"];

items.length = 3;               // truncate to 3 elements
console.log(items);             // ["a", "b", "c"]  — d and e are gone!

items.length = 0;               // empty the array
console.log(items);             // []
```

Setting `length` is a valid (if unusual) way to truncate. Setting it to `0` empties the array in place — this mutates the original, unlike reassigning `items = []`.

---

### 7 — Array equality doesn't work with `==` or `===`

```js
const a = [1, 2, 3];
const b = [1, 2, 3];

console.log(a === b);           // false  — different objects in memory
console.log(a == b);            // false  — same reason

const c = a;
console.log(a === c);           // true   — same reference

// To compare array contents, use JSON.stringify (simple cases only)
console.log(JSON.stringify(a) === JSON.stringify(b)); // true ✅
// (doesn't work with undefined, functions, or circular references)
```

---

## Common mistakes

### ❌ Mistake 1 — Off-by-one error (using `length` as the last index)

```js
const users = ["Alice", "Bob", "Charlie"];

// WRONG — users[3] is undefined; users.length is 3, last index is 2
console.log(users[users.length]);     // ❌ undefined

// RIGHT
console.log(users[users.length - 1]); // ✅ "Charlie"
```

---

### ❌ Mistake 2 — Using `for...in` on arrays

```js
const tags = ["javascript", "node", "css"];

// WRONG — for...in gives you STRING keys and may include inherited properties
for (const key in tags) {
  console.log(key);   // "0", "1", "2"  — strings, not numbers
}

// RIGHT — use for...of for values, or classic for loop for index+value
for (const tag of tags) {
  console.log(tag);   // "javascript", "node", "css"
}

for (let i = 0; i < tags.length; i++) {
  console.log(i, tags[i]);   // 0 "javascript", 1 "node", 2 "css"
}
```

---

### ❌ Mistake 3 — Treating `const array = []` as immutable

```js
// WRONG mental model
const favourites = [];
favourites = ["movie"];   // ❌ TypeError — can't reassign

// But this is fine and catches people off guard:
const favourites = [];
favourites.push("movie"); // ✅ contents can change freely
console.log(favourites);  // ["movie"]
```

---

## Frequently asked questions

**Q: Can an array hold different types?**  
Yes — JavaScript arrays are not typed. You can mix strings, numbers, booleans, objects, and even other arrays in one array. In practice, keeping a single type per array makes your code much easier to reason about.

**Q: What's the difference between `array.length` and `array.length - 1`?**  
`length` is the count of elements (always 1 more than the last valid index). The last index is always `length - 1` because arrays start at 0.

**Q: How do I check if something is inside an array?**  
Use `array.includes(value)` for a simple true/false check. For objects, you need `.find()` or `.some()` (covered in later topics) because `includes` checks by reference.

**Q: What happens if I set an index beyond the array's length?**  
A sparse array is created with holes between the last real element and the new index. Avoid this — always use `push()` to add elements at the end.

**Q: Does a `for` loop run on an empty array?**  
No — if `array.length === 0`, the condition `i < array.length` is false from the start and the loop body never executes.

**Q: Is there a maximum array size?**  
Technically yes: `2^32 - 1` elements (about 4.3 billion). You will never hit this limit in practice.

---

## Practice exercises

### Exercise 1 — easy

Create an array `weekTemperatures` with 7 numbers representing daily high temperatures (make them up).

Without using any built-in array methods:
1. Log the temperature on day 1 (index 0) and day 7 (index 6)
2. Log the total number of days
3. Find and log the hottest temperature using a `for` loop
4. Find and log the coldest temperature using a `for` loop
5. Replace day 4's temperature with `32` and log the whole array

```js
// Write your code here
```

---

### Exercise 2 — medium

You have an array of order objects:

```js
const orders = [
  { orderId: "A001", customer: "Alice",   amount: 120.50, shipped: false },
  { orderId: "A002", customer: "Bob",     amount: 89.99,  shipped: true  },
  { orderId: "A003", customer: "Charlie", amount: 245.00, shipped: false },
  { orderId: "A004", customer: "Diana",   amount: 59.99,  shipped: true  },
  { orderId: "A005", customer: "Eve",     amount: 310.00, shipped: false }
];
```

Using only `for` loops and index-based access (no array methods like `filter` or `map`):
1. Calculate the total revenue across all orders
2. Count how many orders have NOT been shipped
3. Find the order with the highest amount and log its `orderId` and `customer`
4. Create a new array `unshippedOrders` containing only the orders where `shipped === false`

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **2D seating chart** for a cinema with 4 rows and 6 seats per row.

Requirements:
1. Create a 2D array `seatChart` where each inner array has 6 `false` values (false = available, true = taken)
2. Write a function `bookSeat(row, seat)` that:
   - Marks `seatChart[row][seat]` as `true`
   - Logs `"Seat [row+1]-[seat+1] booked"` (use 1-based numbering for display)
   - If the seat is already taken, logs `"Seat [row+1]-[seat+1] is already taken"`
   - If the row or seat is out of bounds, logs `"Invalid seat"`
3. Write a function `getAvailableCount()` that returns the total number of available (`false`) seats by looping through the 2D array
4. Write a function `printChart()` that prints each row like: `Row 1: [ ][ ][X][ ][ ][ ]` (X = taken, space = available)

Test it: book seats `(0,2)`, `(1,4)`, `(0,2)` (duplicate), `(5,0)` (invalid), then print the chart and log available count.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Creating arrays:**
```js
const arr = [];                    // empty
const arr = [1, 2, 3];            // literal
const arr = new Array(3).fill(0); // [0, 0, 0] — pre-filled
const arr = Array.from("hello");  // ["h","e","l","l","o"] — from iterable
```

**Reading:**
```js
arr[0]               // first element
arr[arr.length - 1]  // last element
arr.length           // count of elements (not the last index!)
arr[99]              // undefined (no error) if index doesn't exist
```

**Modifying:**
```js
arr[i] = value       // set / overwrite at index i
arr.length = n       // truncate to n elements
```

**Checking:**
```js
Array.isArray(arr)          // true/false — correct way to check
arr.includes(value)         // true/false — is value in the array?
typeof arr === "object"     // true — but misleading (arrays are objects)
```

**Key rules:**
- Arrays are zero-indexed: first = `[0]`, last = `[length - 1]`
- `const` prevents reassignment, NOT mutation
- Out-of-bounds access returns `undefined`, not an error
- Arrays are objects — `typeof` returns `"object"`, use `Array.isArray()`
- Arrays are passed by reference — assign with spread `[...arr]` to copy
- Never use `for...in` on arrays — use `for`, `for...of`, or array methods

---

## Connected topics

- **24 — Array mutation methods** — `push`, `pop`, `shift`, `unshift`, `splice` — adding and removing elements
- **25 — Array non-mutation methods** — `slice`, `concat`, `join`, `flat` — working with arrays without changing the original
- **33 — Array destructuring** — extracting multiple values from an array in one line
