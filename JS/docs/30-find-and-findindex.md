# 30 — find and findIndex

## What is this?
`find` returns the **first element** in an array that passes a test. `findIndex` returns the **index** of that first matching element. Think of it like a librarian searching a shelf — they walk along, check each book against what you asked for, and hand you the first match they find. They stop the moment they find it — they don't keep going to count how many others match.

## Why does it matter?
Every real application needs to look things up: find a user by ID, locate a product in a cart, find an order by its number. `find` and `findIndex` are the clean, expressive tools for these lookups. They replace manual `for` loops with index tracking, and they stop as soon as they find a match — making them more efficient than `filter` for single-item lookups.

---

## Syntax

```js
// find — returns the element itself, or undefined if not found
const element = array.find((element, index, array) => condition);

// findIndex — returns the index (0-based), or -1 if not found
const index = array.findIndex((element, index, array) => condition);
```

**Callback parameters:**

| Parameter | What it is | Required? |
|---|---|---|
| `element` | The current item being tested | ✅ Yes |
| `index` | Its position in the array | ❌ Optional |
| `array` | The original array | ❌ Optional |

**Return values:**

| Method | Found | Not found |
|---|---|---|
| `find` | The element itself | `undefined` |
| `findIndex` | The index (≥ 0) | `-1` |

---

## How it works — line by line

```js
const users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
  { id: 3, name: "Charlie" }
];

const user = users.find((u) => u.id === 2);
```

1. Tests `{ id: 1, name: "Alice" }` — `1 === 2` is false → skip
2. Tests `{ id: 2, name: "Bob" }` — `2 === 2` is true → **return this element, stop**
3. `{ id: 3, name: "Charlie" }` is never even visited

`user` is `{ id: 2, name: "Bob" }`.

```js
const idx = users.findIndex((u) => u.id === 2);
```

Same walk-through — but returns `1` (the index) instead of the element.

```js
const missing = users.find((u) => u.id === 99);
```
Tests all three — none match — returns `undefined`.

```js
const missingIdx = users.findIndex((u) => u.id === 99);
```
Returns `-1`.

---

## Example 1 — basic: finding primitives

```js
const primes = [2, 3, 5, 7, 11, 13, 17, 19];

// First prime greater than 10
const firstBig = primes.find((p) => p > 10);
console.log(firstBig); // 11

// Index of the first even prime (only 2 is even)
const evenIdx = primes.findIndex((p) => p % 2 === 0);
console.log(evenIdx); // 0

// Not found cases
const over100 = primes.find((p) => p > 100);
console.log(over100); // undefined

const negIdx = primes.findIndex((p) => p > 100);
console.log(negIdx);  // -1

// ---- Strings ----
const colours = ["red", "green", "blue", "yellow", "purple"];

const longColour = colours.find((c) => c.length > 5);
console.log(longColour); // "yellow"  ← first match only (not "purple")

const purpleIdx = colours.findIndex((c) => c === "purple");
console.log(purpleIdx); // 4
```

---

## Example 2 — real world: lookup by ID

```js
const products = [
  { id: "p-001", name: "Mechanical Keyboard", price: 89.99,  inStock: true  },
  { id: "p-002", name: "Wireless Mouse",      price: 29.99,  inStock: true  },
  { id: "p-003", name: "4K Monitor",          price: 399.99, inStock: false },
  { id: "p-004", name: "USB-C Hub",           price: 49.99,  inStock: true  }
];

// Find a product by ID (common in cart/checkout logic)
function getProductById(products, id) {
  const product = products.find((p) => p.id === id);

  if (!product) {
    console.error(`Product ${id} not found`);
    return null;
  }

  return product;
}

const keyboard = getProductById(products, "p-001");
console.log(keyboard.name);   // "Mechanical Keyboard"
console.log(keyboard.price);  // 89.99

const ghost = getProductById(products, "p-999");
// Product p-999 not found
console.log(ghost);           // null

// findIndex to update a product in the array
function updateProductPrice(products, id, newPrice) {
  const index = products.findIndex((p) => p.id === id);

  if (index === -1) {
    console.error(`Cannot update — product ${id} not found`);
    return;
  }

  products[index] = { ...products[index], price: newPrice };
  console.log(`Updated ${products[index].name} to $${newPrice}`);
}

updateProductPrice(products, "p-002", 24.99);
// Updated Wireless Mouse to $24.99
console.log(products[1].price); // 24.99
```

---

## Example 3 — real world: cart operations

```js
let cartItems = [
  { id: 101, name: "Keyboard", price: 89.99, qty: 1 },
  { id: 102, name: "Mouse",    price: 29.99, qty: 2 },
  { id: 103, name: "Webcam",   price: 69.99, qty: 1 }
];

// Add item — if already in cart, increment qty; otherwise add new
function addToCart(item) {
  const existingIndex = cartItems.findIndex((i) => i.id === item.id);

  if (existingIndex !== -1) {
    // Item already in cart — update qty
    cartItems[existingIndex] = {
      ...cartItems[existingIndex],
      qty: cartItems[existingIndex].qty + item.qty
    };
    console.log(`${item.name} qty updated to ${cartItems[existingIndex].qty}`);
  } else {
    // New item — add to cart
    cartItems.push(item);
    console.log(`${item.name} added to cart`);
  }
}

addToCart({ id: 102, name: "Mouse", price: 29.99, qty: 1 });
// Mouse qty updated to 3

addToCart({ id: 104, name: "Headset", price: 49.99, qty: 1 });
// Headset added to cart

console.log(cartItems.map((i) => `${i.name} x${i.qty}`));
// ["Keyboard x1", "Mouse x3", "Webcam x1", "Headset x1"]
```

---

## Example 4 — real world: navigation and routing

```js
const routes = [
  { path: "/",         component: "HomePage",     auth: false },
  { path: "/shop",     component: "ShopPage",     auth: false },
  { path: "/cart",     component: "CartPage",     auth: false },
  { path: "/account",  component: "AccountPage",  auth: true  },
  { path: "/admin",    component: "AdminPage",    auth: true  },
  { path: "/checkout", component: "CheckoutPage", auth: true  }
];

function getRoute(pathname) {
  const route = routes.find((r) => r.path === pathname);
  return route || { path: pathname, component: "NotFoundPage", auth: false };
}

function navigate(pathname, isLoggedIn) {
  const route = getRoute(pathname);

  if (route.auth && !isLoggedIn) {
    console.log(`Redirecting to /login — ${pathname} requires authentication`);
    return getRoute("/");
  }

  console.log(`Rendering: ${route.component}`);
  return route;
}

navigate("/shop", false);
// Rendering: ShopPage

navigate("/account", false);
// Redirecting to /login — /account requires authentication

navigate("/account", true);
// Rendering: AccountPage

navigate("/unknown", false);
// Rendering: NotFoundPage
```

---

## Tricky things you'll encounter in the real world

### 1 — `find` returns `undefined` for no match — always guard before using the result

```js
const users = [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }];

const user = users.find((u) => u.id === 99);

// ❌ Crash — accessing property of undefined
console.log(user.name);           // TypeError: Cannot read properties of undefined

// ✅ Guard with a null check
if (user) {
  console.log(user.name);
}

// ✅ Optional chaining (ES2020)
console.log(user?.name);          // undefined — no crash

// ✅ Default value with nullish coalescing
const name = user?.name ?? "Unknown user";
console.log(name);                // "Unknown user"
```

---

### 2 — `findIndex` returns `-1` for no match — check before using as an index

```js
const items = ["a", "b", "c"];
const idx = items.findIndex((x) => x === "z");

// ❌ Using -1 as an index gives the last element (in some contexts)
console.log(items[-1]);    // undefined (JS doesn't support -1 index)
items.splice(idx, 1);      // splice(-1, 1) removes the LAST element! ⚠️
console.log(items);        // ["a", "b"]  ← "c" was removed accidentally

// ✅ Always check for -1 before using the index
if (idx !== -1) {
  items.splice(idx, 1);
}
```

---

### 3 — `find` vs `filter` — first match vs all matches

```js
const orders = [
  { id: 1, status: "pending" },
  { id: 2, status: "shipped" },
  { id: 3, status: "pending" }
];

// find — returns first pending order (stops after id 1)
const firstPending = orders.find((o) => o.status === "pending");
console.log(firstPending); // { id: 1, status: "pending" }

// filter — returns ALL pending orders (visits all 3)
const allPending = orders.filter((o) => o.status === "pending");
console.log(allPending);   // [{ id: 1 }, { id: 3 }]

// Rule: if you expect ONE result → find. If you expect MULTIPLE → filter.
```

---

### 4 — `find` with objects compares by REFERENCE, not by value

```js
const targetUser = { id: 2, name: "Bob" };
const users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
  { id: 3, name: "Charlie" }
];

// ❌ WRONG — comparing object references (always false unless literally the same object)
const found = users.find((u) => u === targetUser);
console.log(found); // undefined  ← even though the content matches

// ✅ RIGHT — compare by a unique property
const found = users.find((u) => u.id === targetUser.id);
console.log(found); // { id: 2, name: "Bob" }  ✅
```

---

### 5 — `findLast` and `findLastIndex` (ES2023)

```js
const log = [
  { level: "info",    msg: "Server started" },
  { level: "warning", msg: "High memory" },
  { level: "info",    msg: "Request received" },
  { level: "error",   msg: "DB connection failed" },
  { level: "info",    msg: "Retrying connection" }
];

// find — first info message (from the left)
console.log(log.find((e) => e.level === "info")?.msg);
// "Server started"

// findLast — last info message (from the right) — ES2023
console.log(log.findLast((e) => e.level === "info")?.msg);
// "Retrying connection"

// findLastIndex — index of last error
console.log(log.findLastIndex((e) => e.level === "error"));
// 3
```

---

### 6 — Performance: `find` stops early; `filter` doesn't

```js
const millionItems = Array.from({ length: 1_000_000 }, (_, i) => ({ id: i }));

// find — stops at the first match (id 500 is found after ~501 iterations)
const found = millionItems.find((item) => item.id === 500);  // fast ✅

// filter — scans ALL 1,000,000 items even though only one matches
const foundArr = millionItems.filter((item) => item.id === 500); // slower ⚠️
```

For single-item lookups, `find` is always preferable to `filter(...)[0]`.

---

## Common mistakes

### ❌ Mistake 1 — Using `filter` for a single-item lookup

```js
const users = [{ id: 1 }, { id: 2 }, { id: 3 }];

// WRONG — filter returns an array; you have to grab [0]; also scans the whole array
const user = users.filter((u) => u.id === 2)[0];

// RIGHT — find returns the element directly and stops early
const user = users.find((u) => u.id === 2);
```

---

### ❌ Mistake 2 — Not checking the result of `find` before using it

```js
const product = products.find((p) => p.id === "p-999");

// WRONG — crashes if product is undefined
console.log(product.name.toUpperCase()); // TypeError

// RIGHT — guard before use
if (!product) {
  console.error("Product not found");
} else {
  console.log(product.name.toUpperCase());
}

// RIGHT — optional chaining for safe access
console.log(product?.name?.toUpperCase() ?? "Unknown");
```

---

### ❌ Mistake 3 — Confusing -1 (not found) with a valid index

```js
const tags = ["js", "node", "react"];
const idx = tags.findIndex((t) => t === "vue");  // -1

// WRONG — treats -1 as if it were a valid index
tags.splice(idx, 1);   // ❌ removes "react" (last element)!

// RIGHT — always check
if (idx !== -1) {
  tags.splice(idx, 1);
}
```

---

## Frequently asked questions

**Q: What's the difference between `find` and `indexOf`?**  
`indexOf` works with primitives and checks by strict equality (`===`). `find` takes a callback and works with any condition — it's the right choice for objects. `indexOf` is fine for checking simple values like strings or numbers.

**Q: What's the difference between `find` and `some`?**  
`some` returns `true`/`false` — does anything match? `find` returns the matching element itself. Use `some` when you only need to know if a match exists. Use `find` when you need to use the matched element.

**Q: Can I use `find` on a string?**  
No — `find` is an Array method. For strings, use `indexOf`, `includes`, or regex.

**Q: What if multiple elements match the condition?**  
`find` returns only the **first** match and stops. `findIndex` returns the index of only the first match. If you need all matches, use `filter`.

**Q: Is there a `findLast`?**  
Yes — `findLast` and `findLastIndex` were added in ES2023. They work the same way but search from right to left, returning the last match.

**Q: Does `findIndex` modify the array?**  
No — neither `find` nor `findIndex` modify the array. They are both read-only methods.

---

## Practice exercises

### Exercise 1 — easy

Given:
```js
const inventory = [
  { sku: "A1", name: "Notebook",      stock: 12, price: 4.99  },
  { sku: "B2", name: "Pen Set",       stock: 0,  price: 8.99  },
  { sku: "C3", name: "Sticky Notes",  stock: 45, price: 3.49  },
  { sku: "D4", name: "Highlighters",  stock: 0,  price: 6.99  },
  { sku: "E5", name: "Binder",        stock: 7,  price: 12.99 }
];
```

1. Use `find` to get the first out-of-stock item (stock === 0). Log its name.
2. Use `find` to get the first item priced over $10. Log its name and price.
3. Use `findIndex` to find the index of the item with sku `"C3"`. Log the index.
4. Try to find an item with sku `"Z9"` — handle the `undefined` result safely using optional chaining.

```js
// Write your code here
```

---

### Exercise 2 — medium

You're building a task management system:

```js
const tasks = [
  { id: 1, title: "Design mockup",   status: "done",        priority: "high",   assignee: "Alice" },
  { id: 2, title: "Write tests",     status: "in-progress", priority: "high",   assignee: "Bob"   },
  { id: 3, title: "Build feature",   status: "todo",        priority: "medium", assignee: "Alice" },
  { id: 4, title: "Code review",     status: "todo",        priority: "high",   assignee: "Charlie" },
  { id: 5, title: "Deploy to staging", status: "todo",      priority: "low",    assignee: "Bob"   }
];
```

Write these functions using `find` and `findIndex`:
- `getTaskById(id)` — returns the task or logs an error and returns `null` if not found
- `updateTaskStatus(id, newStatus)` — uses `findIndex` to locate and update the task in place; logs an error if not found
- `getNextHighPriorityTask()` — returns the first `"todo"` task with `priority === "high"`
- `reassignTask(id, newAssignee)` — finds and updates the assignee for a given task id

Test all four functions.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **binary search function** using `findIndex` logic — then benchmark it against a manual loop.

First write `linearSearch(sortedArr, target)` — uses `findIndex` to find `target`.

Then write `binarySearch(sortedArr, target)` — implements binary search manually (without `findIndex`):
- Keep `low = 0` and `high = arr.length - 1`
- Each iteration: calculate `mid = Math.floor((low + high) / 2)`
- If `arr[mid] === target` → return `mid`
- If `arr[mid] < target` → `low = mid + 1`
- If `arr[mid] > target` → `high = mid - 1`
- If `low > high` → return `-1`

Then generate a sorted array of 10,000 numbers (1 to 10,000), search for `7,777` with both, and compare the number of iterations each approach takes (add a counter to each).

Log: the found index, and how many iterations each method used.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

```js
// find — returns the element
const item = array.find((el) => condition);  // element or undefined

// findIndex — returns the position
const idx = array.findIndex((el) => condition);  // number >= 0 or -1
```

**Return values summary:**

| | Found | Not found |
|---|---|---|
| `find` | The element | `undefined` |
| `findIndex` | Index ≥ 0 | `-1` |
| `findLast` (ES2023) | Last matching element | `undefined` |
| `findLastIndex` (ES2023) | Index of last match | `-1` |

**Choosing the right method:**

| Goal | Method |
|---|---|
| Get first matching element | `find` |
| Get index of first match | `findIndex` |
| Check if ANY match exists | `some` |
| Get ALL matching elements | `filter` |
| Get one primitive by value | `indexOf` / `includes` |

**Key rules:**
- `find` returns `undefined` for no match — always guard before using result
- `findIndex` returns `-1` for no match — always check `!== -1` before using as index
- Both stop at the first match — more efficient than `filter` for single lookups
- For objects, compare by a property value, not by reference (`===` on objects is always false unless same instance)
- Use optional chaining (`?.`) for safe property access on potentially undefined results

---

## Connected topics

- **28 — filter** — returns ALL matches as a new array; use when you need multiple results
- **31 — some and every** — boolean answers: does any/every element match?
- **23 — Array basics** — indexing and how out-of-bounds access works
- **39 — Optional chaining (?.)** — the safest way to access properties on a result that might be `undefined`
