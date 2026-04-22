# 11 — for...of

## What is this?

`for...of` is a loop that iterates over the **values** of any iterable — arrays, strings, Maps, Sets, NodeLists, and more.

```js
for (const value of iterable) {
  // do something with value
}
```

It was introduced in ES6 as a cleaner replacement for the classic `for` loop when you only care about **values**, not index numbers.

---

## Why does it matter?

Before `for...of`, iterating an array looked like this:

```js
for (let i = 0; i < products.length; i++) {
  console.log(products[i]);
}
```

That's 3 things to manage: initialize `i`, check `i < length`, update `i++`. And you have to remember to use `products[i]` not `products`.

`for...of` collapses all of that:

```js
for (const product of products) {
  console.log(product);
}
```

One line. You get the value directly. No index arithmetic. No off-by-one errors.

**Use `for...of` when:** you want every value in order and don't need the index.  
**Use classic `for` when:** you need the index number, or need to iterate backwards, or skip elements by index.

---

## Syntax

```js
for (const item of iterable) {
  // item is the current value
}
```

- `item` — the variable name for the current value. You pick the name.
- `iterable` — any object that implements the iterable protocol (arrays, strings, Maps, Sets, etc.)
- `const` is preferred over `let` unless you reassign `item` inside the loop
- The loop ends automatically when there are no more items

---

## How it works — line by line

```js
const orderItems = ["laptop", "mouse", "keyboard"];

for (const item of orderItems) {
  console.log("Packing:", item);
}
```

**Execution trace:**

```
Loop starts → gets iterator from orderItems
Iteration 1: item = "laptop"  → logs "Packing: laptop"
Iteration 2: item = "mouse"   → logs "Packing: mouse"
Iteration 3: item = "keyboard"→ logs "Packing: keyboard"
No more items → loop ends
```

**What's an iterator?**  
Under the hood, `for...of` calls `orderItems[Symbol.iterator]()` — this returns an iterator object with a `.next()` method. Each call to `.next()` returns `{ value: "laptop", done: false }`. When done is `true`, the loop stops.

You don't need to know this to use `for...of`, but it explains *why* it works on strings and Maps too — they all implement `Symbol.iterator`.

---

## Example 1 — basic (array and string)

### Iterating an array

```js
const cartPrices = [29.99, 14.50, 8.00, 49.99];
let totalPrice = 0;

for (const price of cartPrices) {
  totalPrice += price;
}

console.log("Cart total: $" + totalPrice.toFixed(2));
// "Cart total: $102.48"
```

### Iterating a string

```js
const productCode = "ABC123";

for (const char of productCode) {
  console.log(char);
}
// A
// B
// C
// 1
// 2
// 3
```

Strings are iterable too. Each `char` is a single character. This handles Unicode characters correctly — unlike `for (let i = 0; i < str.length; i++)` which can split emoji.

### Getting the index too — entries()

If you need both the index and the value, use `.entries()`:

```js
const menuItems = ["burger", "fries", "shake"];

for (const [index, item] of menuItems.entries()) {
  console.log(`${index + 1}. ${item}`);
}
// 1. burger
// 2. fries
// 3. shake
```

`menuItems.entries()` returns pairs of `[index, value]`. The `[index, item]` syntax is **array destructuring** — it unpacks that pair into two variables. (Covered deeply in the Arrays doc.)

---

## Example 2 — real world

### Processing an order list

```js
const orders = [
  { orderId: "ORD-001", customerName: "Sarah Chen", total: 129.99, isPaid: true },
  { orderId: "ORD-002", customerName: "Marcus Lee", total: 74.50, isPaid: false },
  { orderId: "ORD-003", customerName: "Priya Nair", total: 209.00, isPaid: true },
  { orderId: "ORD-004", customerName: "Jake Torres", total: 55.00, isPaid: false },
];

const unpaidOrders = [];
let totalRevenue = 0;

for (const order of orders) {
  if (order.isPaid) {
    totalRevenue += order.total;
  } else {
    unpaidOrders.push(order.orderId);
  }
}

console.log("Total revenue collected: $" + totalRevenue.toFixed(2));
// "Total revenue collected: $338.99"

console.log("Unpaid orders:", unpaidOrders.join(", "));
// "Unpaid orders: ORD-002, ORD-004"
```

### Iterating a Set (unique values)

```js
const rawTags = ["javascript", "react", "javascript", "node", "react", "css"];
const uniqueTags = new Set(rawTags);  // Set removes duplicates

for (const tag of uniqueTags) {
  console.log(tag);
}
// javascript
// react
// node
// css
```

### Iterating a Map

```js
const stockLevels = new Map([
  ["apples", 42],
  ["bananas", 15],
  ["oranges", 0],
]);

for (const [product, quantity] of stockLevels) {
  if (quantity === 0) {
    console.log(`${product}: OUT OF STOCK`);
  } else {
    console.log(`${product}: ${quantity} units`);
  }
}
// apples: 42 units
// bananas: 15 units
// oranges: OUT OF STOCK
```

Maps are iterable and each iteration gives you `[key, value]` — perfect for destructuring.

---

## Tricky things you'll encounter in the real world

### 1. `for...of` doesn't work on plain objects `{}`

```js
const userProfile = { name: "Alex", age: 28, city: "Berlin" };

for (const value of userProfile) {  // ❌ TypeError: userProfile is not iterable
  console.log(value);
}
```

Plain objects are **not iterable**. To loop over an object, use:

```js
for (const key of Object.keys(userProfile)) {      // ["name", "age", "city"]
for (const value of Object.values(userProfile)) {  // ["Alex", 28, "Berlin"]
for (const [key, value] of Object.entries(userProfile)) {  // [["name","Alex"],...]
```

Or use `for...in` (covered next topic). Knowing which tool to reach for is important.

### 2. `const` vs `let` in `for...of`

```js
// ✅ Use const — you're not reassigning item
for (const product of products) {
  console.log(product);
}

// ✅ Use let only if you're reassigning
for (let price of prices) {
  price = price * 1.1;  // you modified price, so let is needed
  console.log(price);   // but this does NOT modify the original array!
}
```

Reassigning `price` inside the loop does **not** change the original `prices` array. You're just changing your local copy. This is a common misconception.

### 3. `break` and `continue` work inside `for...of`

```js
const transactionAmounts = [12, 55, 340, 2, 8900, 14, 600];

for (const amount of transactionAmounts) {
  if (amount > 5000) {
    console.log("Flagged suspicious transaction:", amount);
    break;  // stops the entire loop
  }

  if (amount < 10) {
    continue;  // skip small amounts, go to next iteration
  }

  console.log("Processing:", amount);
}
// Processing: 12
// Processing: 55
// Processing: 340
// Flagged suspicious transaction: 8900
```

This is an advantage over `.forEach()` — you **cannot** `break` out of a `.forEach()` callback. `for...of` gives you full loop control.

### 4. `for...of` vs `.forEach()` — the critical difference

```js
const discountCodes = ["SAVE10", "SALE20", "PROMO30"];

// forEach — cannot break, cannot use await in a normal function
discountCodes.forEach(code => {
  console.log(code);
});

// for...of — can break, can use await
for (const code of discountCodes) {
  console.log(code);
}
```

**In async code, `for...of` is almost always better:**

```js
// ❌ forEach with async — does NOT wait between iterations
discountCodes.forEach(async (code) => {
  await validateCode(code);  // all 3 fire at once, not sequentially
});

// ✅ for...of with async — waits for each one
for (const code of discountCodes) {
  await validateCode(code);  // waits before moving to the next
}
```

This is one of the most common async bugs in real codebases.

### 5. Destructuring objects inside `for...of`

```js
const employees = [
  { name: "Dana", department: "Engineering", salary: 95000 },
  { name: "Ryan", department: "Design", salary: 80000 },
  { name: "Mei", department: "Marketing", salary: 72000 },
];

for (const { name, salary } of employees) {
  console.log(`${name} earns $${salary.toLocaleString()}`);
}
// Dana earns $95,000
// Ryan earns $80,000
// Mei earns $72,000
```

You can destructure the object directly in the `for...of` header. This is extremely common in real codebases — clean and readable.

### 6. Modifying an array while iterating with `for...of`

```js
const items = [1, 2, 3, 4, 5];

for (const item of items) {
  items.push(item * 2);  // ⚠️ this creates an infinite loop!
  console.log(item);
}
```

`for...of` on an array uses a live iterator. If you push to the array while iterating, the iterator keeps going. **Never modify the array you're iterating.** Create a separate results array instead.

---

## Common mistakes

### Mistake 1: Using `for...of` on a plain object

```js
const config = { theme: "dark", lang: "en" };

// ❌ TypeError
for (const value of config) { ... }

// ✅ Fix
for (const [key, value] of Object.entries(config)) { ... }
```

### Mistake 2: Thinking reassigning the loop variable changes the array

```js
const prices = [10, 20, 30];

for (let price of prices) {
  price = price * 2;  // does NOT change prices array
}

console.log(prices);  // [10, 20, 30] — unchanged!
```

To transform an array, use `.map()` (covered in the Arrays section).

### Mistake 3: Using `for...of` when you need the index

```js
const steps = ["login", "checkout", "confirm"];

// ❌ Awkward — you lose the index
for (const step of steps) {
  console.log("Step ???:", step);
}

// ✅ Use .entries() if you need both
for (const [stepNumber, step] of steps.entries()) {
  console.log(`Step ${stepNumber + 1}: ${step}`);
}
```

### Mistake 4: Forgetting `for...of` can't break out of `.forEach()`

```js
// This is NOT for...of — this is forEach and break won't work:
products.forEach(product => {
  if (product.isDiscontinued) break;  // ❌ SyntaxError: Illegal break statement
});

// ✅ Use for...of instead
for (const product of products) {
  if (product.isDiscontinued) break;  // works fine
}
```

---

## Practice exercises

### Exercise 1 — easy

You have this array of student names:

```js
const studentNames = ["Alice", "Bob", "Charlie", "Diana", "Edward"];
```

Using `for...of`:
1. Log each name with "Student: " prefix
2. Count how many names have more than 5 characters and log the count at the end

Expected output:
```
Student: Alice
Student: Bob
Student: Charlie
Student: Diana
Student: Edward
Names longer than 5 chars: 2
```

---

### Exercise 2 — medium

You have this array of product objects from an online store:

```js
const storeProducts = [
  { name: "Wireless Headphones", category: "Electronics", price: 89.99, inStock: true },
  { name: "Yoga Mat", category: "Fitness", price: 34.99, inStock: false },
  { name: "Coffee Maker", category: "Kitchen", price: 129.99, inStock: true },
  { name: "Running Shoes", category: "Fitness", price: 74.99, inStock: true },
  { name: "Desk Lamp", category: "Office", price: 45.99, inStock: false },
  { name: "Protein Powder", category: "Fitness", price: 54.99, inStock: true },
];
```

Using `for...of` with **object destructuring** in the loop header:
1. Build a separate array called `availableProducts` containing only in-stock items
2. Calculate the total value of all in-stock products
3. Find the most expensive in-stock product and store it in `mostExpensive`
4. Log all three results

---

### Exercise 3 — hard

You are building a **grade report generator** for a school. You have:

```js
const classData = [
  { studentName: "Olivia Hart", scores: [88, 92, 76, 95, 89] },
  { studentName: "Ethan Brooks", scores: [55, 62, 48, 70, 58] },
  { studentName: "Sofia Patel", scores: [97, 99, 95, 100, 98] },
  { studentName: "Liam Nguyen", scores: [73, 68, 80, 77, 71] },
  { studentName: "Zoe Williams", scores: [40, 35, 52, 44, 38] },
];
```

Using `for...of` (possibly nested), write a function `generateReport(classData)` that:

1. For each student, calculates their **average score** (sum of scores ÷ number of scores)
2. Assigns a **letter grade** based on average:
   - 90–100 → "A"
   - 80–89 → "B"
   - 70–79 → "C"
   - 60–69 → "D"
   - Below 60 → "F"
3. Determines if the student **passed** (average ≥ 60)
4. Builds and returns an array of result objects like:
   ```js
   { studentName: "Olivia Hart", average: 88, grade: "B", passed: true }
   ```
5. After calling `generateReport(classData)`, loop through the results and log each one in this format:
   ```
   Olivia Hart — Avg: 88.0 | Grade: B | Status: PASSED
   Ethan Brooks — Avg: 58.6 | Grade: F | Status: FAILED
   ```
6. **Bonus:** Count how many students passed and how many failed, and log a summary at the end:
   ```
   Class summary: 3 passed, 2 failed
   ```

Requirements:
- Use `for...of` for all loops (no classic `for`, no `.forEach()`)
- Use object destructuring in the `for...of` header where it makes the code cleaner
- The `average` in each result object must be rounded to 1 decimal place

---

## Quick reference

```
for...of cheat sheet
─────────────────────────────────────────────────────
BASIC ARRAY          for (const item of arr)
WITH INDEX           for (const [i, item] of arr.entries())
STRING               for (const char of str)
SET                  for (const value of mySet)
MAP                  for (const [key, value] of myMap)
OBJECT KEYS          for (const key of Object.keys(obj))
OBJECT VALUES        for (const value of Object.values(obj))
OBJECT ENTRIES       for (const [k, v] of Object.entries(obj))
─────────────────────────────────────────────────────
USE const            unless you reassign the variable
USE break            to exit early (works — unlike forEach)
USE continue         to skip an iteration
CANNOT USE ON        plain objects {} directly
ASYNC SAFE           for...of respects await; forEach does NOT
─────────────────────────────────────────────────────
for...of vs for      for...of = values only, cleaner syntax
                     for = when you need index arithmetic
for...of vs forEach  for...of = break/continue/await work
                     forEach = functional style, no break
for...of vs for...in for...of = VALUES of iterables
                     for...in = KEYS of objects (next topic)
```

---

## Connected topics

- **12 — for...in** — iterates over object *keys* instead of values; important to know when to use which
- **13 — break and continue** — controlling loop flow (works in `for...of`)
- **Arrays (23+)** — `.map()`, `.filter()`, `.reduce()` are often better than `for...of` for transformations
- **Destructuring (Modern JS section)** — the `[index, value]` and `{ name, age }` syntax used in `for...of` headers
- **Async/Await (52+)** — why `for...of` is the correct loop to use with `await`
- **Sets and Maps (Modern JS section)** — both are iterable with `for...of`
- **Symbol.iterator** — the protocol that makes `for...of` work; any object implementing it becomes iterable
