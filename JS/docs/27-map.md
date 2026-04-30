# 27 — map

## What is this?
`map` is an array method that **calls a function on every element and returns a brand new array** containing the results. Think of it like a factory conveyor belt — each item goes in, gets transformed (painted, reshaped, labelled), and comes out the other end as a new item. The original items on the input belt are never touched.

## Why does it matter?
`map` is the most-used array method in modern JavaScript. Every time you need to convert a list of raw data into a list of display values, transform API responses, generate JSX elements in React, or reshape objects — `map` is the tool. It replaces the pattern of "create empty array, loop, push transformed value, return array" with a single, expressive line.

---

## Syntax

```js
const newArray = array.map(function (element, index, array) {
  return transformedElement;  // MUST return something — this becomes the new element
});

// Arrow function shorthand (most common)
const newArray = array.map((element) => transformedElement);

// Implicit return (no curly braces needed for single expressions)
const newArray = array.map((element) => element * 2);
```

**Callback parameters:**

| Parameter | What it is | Required? |
|---|---|---|
| `element` | The current item | ✅ Yes (in practice) |
| `index` | The position (0-based) | ❌ Optional |
| `array` | The original array | ❌ Optional (rarely used) |

**Critical rule:** The callback MUST `return` a value. If you forget `return`, the new array is filled with `undefined`.

---

## How it works — line by line

```js
const prices = [10, 25, 8, 40];
const discounted = prices.map((price) => price * 0.9);
```

1. `map` creates a new empty array internally
2. Calls the callback with `(10, 0, prices)` → callback returns `9` → stored at index 0
3. Calls the callback with `(25, 1, prices)` → returns `22.5` → stored at index 1
4. Calls the callback with `(8, 2, prices)` → returns `7.2` → stored at index 2
5. Calls the callback with `(40, 3, prices)` → returns `36` → stored at index 3
6. Returns the new array `[9, 22.5, 7.2, 36]`

`prices` is still `[10, 25, 8, 40]` — completely untouched.

The new array is **always the same length** as the original. Every element must map to exactly one output element.

---

## Example 1 — basic: transforming values

```js
const examScores = [72, 88, 95, 61, 100, 79];

// Convert to letter grades
const letterGrades = examScores.map((score) => {
  if (score >= 90) return "A";
  if (score >= 80) return "B";
  if (score >= 70) return "C";
  if (score >= 60) return "D";
  return "F";
});

console.log(letterGrades); // ["C", "B", "A", "D", "A", "C"]

// Add 5 bonus points to each score (capped at 100)
const bonusScores = examScores.map((score) => Math.min(score + 5, 100));
console.log(bonusScores);  // [77, 93, 100, 66, 100, 84]

// Format for display
const displayScores = examScores.map((score, index) => `Student ${index + 1}: ${score}%`);
console.log(displayScores);
// ["Student 1: 72%", "Student 2: 88%", ...]

// Originals untouched
console.log(examScores);   // [72, 88, 95, 61, 100, 79]
```

---

## Example 2 — real world: transforming API response data

```js
// Raw data from an API — often messy, inconsistently shaped
const apiUsers = [
  { user_id: 101, first_name: "alice",   last_name: "SMITH",   email_address: "alice@example.com",   is_active: 1 },
  { user_id: 102, first_name: "BOB",     last_name: "johnson", email_address: "bob@example.com",     is_active: 0 },
  { user_id: 103, first_name: "charlie", last_name: "BROWN",   email_address: "charlie@example.com", is_active: 1 }
];

// Transform into the shape your UI actually needs
const users = apiUsers.map((rawUser) => ({
  id:       rawUser.user_id,
  fullName: capitalise(rawUser.first_name) + " " + capitalise(rawUser.last_name),
  email:    rawUser.email_address.toLowerCase(),
  isActive: rawUser.is_active === 1
}));

function capitalise(str) {
  return str.charAt(0).toUpperCase() + str.slice(1).toLowerCase();
}

console.log(users);
// [
//   { id: 101, fullName: "Alice Smith",   email: "alice@example.com",   isActive: true  },
//   { id: 102, fullName: "Bob Johnson",   email: "bob@example.com",     isActive: false },
//   { id: 103, fullName: "Charlie Brown", email: "charlie@example.com", isActive: true  }
// ]
```

---

## Example 3 — real world: building UI elements

```js
// Product data
const products = [
  { id: 1, name: "Mechanical Keyboard", price: 89.99,  inStock: true  },
  { id: 2, name: "Wireless Mouse",      price: 29.99,  inStock: true  },
  { id: 3, name: "4K Monitor",          price: 399.99, inStock: false },
  { id: 4, name: "USB-C Hub",           price: 49.99,  inStock: true  }
];

// Generate HTML card strings (same pattern used in React to generate JSX)
const productCards = products.map((product) => {
  const stockLabel = product.inStock
    ? `<span class="badge in-stock">In Stock</span>`
    : `<span class="badge out-of-stock">Out of Stock</span>`;

  return `
    <div class="product-card" data-id="${product.id}">
      <h3>${product.name}</h3>
      <p class="price">$${product.price.toFixed(2)}</p>
      ${stockLabel}
      <button ${product.inStock ? "" : "disabled"}>Add to Cart</button>
    </div>
  `.trim();
});

// Render all cards
const container = document.getElementById("product-grid");
container.innerHTML = productCards.join("");
```

---

## Example 4 — real world: reshaping objects (pick/rename fields)

```js
const rawOrders = [
  { order_id: "ORD-001", cust_name: "Alice",   total_amount: 120.50, order_status: "shipped",   created_at: "2026-04-01" },
  { order_id: "ORD-002", cust_name: "Bob",     total_amount: 89.99,  order_status: "pending",   created_at: "2026-04-03" },
  { order_id: "ORD-003", cust_name: "Charlie", total_amount: 245.00, order_status: "delivered", created_at: "2026-04-05" }
];

// For a dashboard table — only the columns we need, renamed cleanly
const tableRows = rawOrders.map(({ order_id, cust_name, total_amount, order_status }) => ({
  id:       order_id,
  customer: cust_name,
  amount:   "$" + total_amount.toFixed(2),
  status:   order_status.charAt(0).toUpperCase() + order_status.slice(1)
}));

console.log(tableRows);
// [
//   { id: "ORD-001", customer: "Alice",   amount: "$120.50", status: "Shipped"   },
//   { id: "ORD-002", customer: "Bob",     amount: "$89.99",  status: "Pending"   },
//   { id: "ORD-003", customer: "Charlie", amount: "$245.00", status: "Delivered" }
// ]
```

---

## Tricky things you'll encounter in the real world

### 1 — Forgetting `return` fills the array with `undefined`

```js
const numbers = [1, 2, 3, 4];

// ❌ WRONG — forgot return (with curly braces, implicit return doesn't apply)
const doubled = numbers.map((n) => {
  n * 2;    // expression evaluated, result discarded — no return!
});
console.log(doubled); // [undefined, undefined, undefined, undefined]

// ✅ RIGHT — explicit return
const doubled = numbers.map((n) => {
  return n * 2;
});

// ✅ RIGHT — arrow function implicit return (no curly braces)
const doubled = numbers.map((n) => n * 2);
```

This is the single most common `map` mistake.

---

### 2 — Returning an object with implicit return requires extra parentheses

```js
const ids = [1, 2, 3];

// ❌ WRONG — JS thinks the { } is a function body, not an object literal
const result = ids.map((id) => { id: id, label: "Item " + id });
// SyntaxError or [undefined, undefined, undefined]

// ✅ RIGHT — wrap the object literal in ( ) to force expression mode
const result = ids.map((id) => ({ id: id, label: "Item " + id }));
console.log(result);
// [{ id: 1, label: "Item 1" }, { id: 2, label: "Item 2" }, { id: 3, label: "Item 3" }]
```

Whenever you implicitly return an object from an arrow function, wrap it in `( )`.

---

### 3 — `map` always produces an array of the same length — it cannot filter

```js
const scores = [88, 45, 92, 31, 76];

// ❌ WRONG — trying to use map to filter (skip low scores)
const passing = scores.map((score) => {
  if (score >= 60) return score;
  // No return for failing scores → returns undefined
});
console.log(passing); // [88, undefined, 92, undefined, 76]

// ✅ RIGHT — use filter to remove, then map to transform
const passing = scores
  .filter((score) => score >= 60)
  .map((score) => score + " pts");
console.log(passing); // ["88 pts", "92 pts", "76 pts"]
```

If you need to both filter AND transform, chain `filter` → `map`.

---

### 4 — `map` on array of objects — shallow copy of the object

```js
const products = [
  { name: "Keyboard", price: 89.99, specs: { weight: "1.2kg" } }
];

// map creates a new array AND new outer objects IF you spread
const updated = products.map((product) => ({
  ...product,          // new object — modifying `updated[0].price` won't affect original
  price: product.price * 0.9
}));

// BUT nested objects are still shared references
updated[0].specs.weight = "1.5kg";
console.log(products[0].specs.weight); // "1.5kg"  ⚠️ — original mutated!

// Fix: spread nested objects too
const safeUpdate = products.map((product) => ({
  ...product,
  price: product.price * 0.9,
  specs: { ...product.specs }   // shallow copy of nested object
}));
```

---

### 5 — `parseInt` passed directly to `map` — the index-as-radix gotcha

```js
const stringNums = ["1", "2", "3", "10"];

// ❌ WRONG — map passes (value, index, array); parseInt uses second arg as radix
const nums = stringNums.map(parseInt);
console.log(nums); // [1, NaN, NaN, 3]  ← completely broken

// Why:
// parseInt("1",  0) → 1   (radix 0 treated as 10)
// parseInt("2",  1) → NaN (base-1 is invalid)
// parseInt("3",  2) → NaN ("3" doesn't exist in base 2)
// parseInt("10", 3) → 3   ("10" in base 3 = 1*3 + 0 = 3)

// ✅ RIGHT — wrap in an arrow function so only the value is passed
const nums = stringNums.map((str) => parseInt(str, 10));
console.log(nums); // [1, 2, 3, 10]
```

---

### 6 — Chaining map after map — consider combining into one pass

```js
const rawPrices = ["$10.50", "$25.00", "$8.99"];

// Two separate passes — works but less efficient
const stripped = rawPrices.map((p) => p.replace("$", ""));
const numbers  = stripped.map((p) => parseFloat(p));

// One pass — more efficient, same result
const numbers = rawPrices.map((p) => parseFloat(p.replace("$", "")));
console.log(numbers); // [10.5, 25, 8.99]
```

---

### 7 — Using `map` for side effects only (wrong tool)

```js
const userNames = ["Alice", "Bob", "Charlie"];

// ❌ WRONG — using map purely for side effects (result array ignored)
userNames.map((name) => console.log(name));

// ✅ RIGHT — use forEach when you don't need a return value
userNames.forEach((name) => console.log(name));
```

Using `map` for side effects creates a throwaway array that wastes memory and signals wrong intent.

---

## Common mistakes

### ❌ Mistake 1 — Missing `return` inside a multi-line callback

```js
const orders = [{ id: 1, amount: 100 }, { id: 2, amount: 200 }];

// WRONG — no return statement
const summaries = orders.map((order) => {
  const label = "Order #" + order.id;
  label + ": $" + order.amount;   // ❌ not returned!
});
console.log(summaries); // [undefined, undefined]

// RIGHT
const summaries = orders.map((order) => {
  const label = "Order #" + order.id;
  return label + ": $" + order.amount;  // ✅
});
```

---

### ❌ Mistake 2 — Forgetting `( )` when returning an object literal

```js
const items = [{ id: 1, name: "Mouse" }];

// WRONG — { } parsed as function body
const result = items.map((item) => { id: item.id });  // undefined

// RIGHT
const result = items.map((item) => ({ id: item.id })); // ✅
```

---

### ❌ Mistake 3 — Mutating the original objects inside `map`

```js
const products = [{ name: "Keyboard", price: 100 }];

// WRONG — mutating the original object instead of creating a new one
const discounted = products.map((product) => {
  product.price = product.price * 0.9;  // ❌ modifies the original!
  return product;
});

console.log(products[0].price); // 90  ← original corrupted

// RIGHT — return a new object
const discounted = products.map((product) => ({
  ...product,
  price: product.price * 0.9   // ✅
}));
console.log(products[0].price); // 100  ← original untouched
```

---

## Frequently asked questions

**Q: What's the difference between `map` and `forEach`?**  
`map` returns a new array of transformed values — use it when you need a result. `forEach` always returns `undefined` — use it for side effects only (logging, DOM updates, sending data).

**Q: Does `map` mutate the original array?**  
`map` itself does not mutate the original array. However, if your callback mutates the original objects inside the array (rather than returning new ones), the originals will change. Always return new values/objects from your callback.

**Q: Can I use `map` to add or remove elements?**  
No — `map` always produces an array of the exact same length as the original. To remove elements, use `filter`. To both transform and filter, chain `filter().map()`.

**Q: Can I use `map` on a string?**  
Not directly. Convert to an array first: `[..."hello"].map(...)` or `Array.from("hello").map(...)`.

**Q: Is `map` synchronous?**  
Yes — like `forEach`, `map` is synchronous. If your callback is `async`, the returned array will be filled with Promises, not resolved values. Use `Promise.all(array.map(asyncFn))` to handle async mapping.

**Q: Can I `break` out of `map`?**  
No. `map` always processes every element. If you need early exit, use a `for...of` loop.

---

## Practice exercises

### Exercise 1 — easy

Given:
```js
const temperatures = [0, 10, 20, 30, 37, 100];
```

Use `map` to produce three new arrays:
1. `fahrenheit` — convert each Celsius value to Fahrenheit: `(C * 9/5) + 32`
2. `labels` — format each as a string like `"0°C = 32°F"` (using both original and converted value)
3. `categories` — `"freezing"` if < 0°C, `"cold"` if < 15°C, `"warm"` if < 30°C, `"hot"` otherwise

Log all three arrays and confirm the original is untouched.

```js
// Write your code here
```

---

### Exercise 2 — medium

You receive this raw API response:
```js
const rawProducts = [
  { product_id: "p-001", product_title: "mechanical keyboard",  base_price: 89.99,  discount_pct: 10, available: true  },
  { product_id: "p-002", product_title: "wireless mouse",       base_price: 29.99,  discount_pct: 0,  available: true  },
  { product_id: "p-003", product_title: "4k monitor",           base_price: 399.99, discount_pct: 20, available: false },
  { product_id: "p-004", product_title: "usb-c hub",            base_price: 49.99,  discount_pct: 5,  available: true  }
];
```

Use `map` to transform each item into a clean product object:
- `id` → from `product_id`
- `name` → from `product_title`, title-cased (capitalise each word)
- `originalPrice` → `base_price` rounded to 2 decimal places
- `salePrice` → `base_price` after `discount_pct`% off, rounded to 2 decimal places
- `hasDiscount` → `true` if `discount_pct > 0`
- `status` → `"Available"` or `"Sold Out"` based on `available`

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **CSV report generator** using `map` and `join`.

You have:
```js
const salesData = [
  { rep: "Alice",   region: "North", q1: 45000, q2: 52000, q3: 48000, q4: 61000 },
  { rep: "Bob",     region: "South", q1: 38000, q2: 41000, q3: 44000, q4: 39000 },
  { rep: "Charlie", region: "East",  q1: 55000, q2: 58000, q3: 62000, q4: 70000 },
  { rep: "Diana",   region: "West",  q1: 29000, q2: 34000, q3: 31000, q4: 37000 }
];
```

Use `map` to produce:
1. A `summaryData` array where each object adds: `annualTotal` (sum of q1–q4), `avgQuarterly` (rounded to 2dp), and `topQuarter` (the quarter name — "Q1"/"Q2"/"Q3"/"Q4" — with the highest value)
2. A `csvLines` array of strings — one per rep — formatted as: `"Alice,North,45000,52000,48000,61000,206000,51500.00,Q4"`
3. Prepend a header row and produce the final `csvOutput` string (joined with `"\n"`)

Log the final `csvOutput` string.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

```js
const result = array.map((element, index) => {
  return transformedValue;   // must return — result becomes element in new array
});
```

**Implicit return patterns:**
```js
array.map((x) => x * 2)               // primitive — no braces needed
array.map((x) => x.name)              // property access
array.map((x) => `Hello, ${x.name}`)  // template literal
array.map((x) => ({ id: x.id }))      // object — MUST wrap in ( )
```

**Common patterns:**

| Goal | Pattern |
|---|---|
| Double every number | `arr.map((n) => n * 2)` |
| Extract one property | `arr.map((item) => item.name)` |
| Rename/reshape objects | `arr.map(({ a, b }) => ({ x: a, y: b }))` |
| Format to string | `arr.map((item) => \`${item.name}: $${item.price}\`)` |
| Convert type | `arr.map((s) => parseFloat(s))` — don't pass `parseFloat` directly |
| Async map | `Promise.all(arr.map(async (item) => await fetchData(item)))` |

**Key rules:**
- Always `return` a value — missing `return` gives `[undefined, ...]`
- Arrow function implicit return: no braces → no `return` needed
- Returning an object with implicit return → wrap in `( )`
- `map` is non-mutating — it never changes the original array
- `map` always produces the same length as the input — use `filter` to remove items
- Don't pass named functions directly (e.g. `parseInt`) — extra args cause bugs
- Don't use `map` for side effects — use `forEach`

---

## Connected topics

- **28 — filter** — removes elements that don't match a condition; commonly chained after or before `map`
- **29 — reduce** — can replicate `map` and do much more; the most powerful iterator
- **26 — forEach** — for side effects only; the "map but returns nothing" counterpart
- **20 — Higher-order functions** — `map` is a HOF; understanding HOFs explains why the callback pattern works
