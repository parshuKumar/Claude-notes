# 28 — filter

## What is this?
`filter` is an array method that **tests every element against a condition and returns a new array containing only the elements that pass**. Think of it like a bouncer at a club — every person in the queue is checked against the entry rules, and only those who meet the criteria get in. The ones who don't pass are ignored; the original queue is untouched.

## Why does it matter?
Any time you need to show only the items that match a search query, display only in-stock products, remove invalid entries, or narrow down a dataset — that's `filter`. It replaces the "create empty array, loop, if condition push, return" pattern with a single, declarative expression. It is one of the most-used array methods in real codebases.

---

## Syntax

```js
const newArray = array.filter(function (element, index, array) {
  return true;   // include this element
  return false;  // exclude this element
});

// Arrow function shorthand (most common)
const newArray = array.filter((element) => condition);
```

**How the callback works:**
- Return **`true`** (or any truthy value) → element **is included** in the result
- Return **`false`** (or any falsy value) → element **is excluded**
- The original array is never modified
- The result array can be **shorter** than (or the same length as) the original — never longer

**Callback parameters:**

| Parameter | What it is | Required? |
|---|---|---|
| `element` | The current item | ✅ Yes (in practice) |
| `index` | The position (0-based) | ❌ Optional |
| `array` | The original array | ❌ Optional (rarely used) |

---

## How it works — line by line

```js
const scores = [88, 45, 92, 31, 76, 60, 55];
const passing = scores.filter((score) => score >= 60);
```

1. `filter` creates a new empty array internally
2. Calls callback with `88` → `88 >= 60` is `true` → included
3. Calls callback with `45` → `45 >= 60` is `false` → excluded
4. Calls callback with `92` → `true` → included
5. Calls callback with `31` → `false` → excluded
6. Calls callback with `76` → `true` → included
7. Calls callback with `60` → `60 >= 60` is `true` → included
8. Calls callback with `55` → `false` → excluded
9. Returns `[88, 92, 76, 60]`

`scores` is still `[88, 45, 92, 31, 76, 60, 55]` — completely untouched.

---

## Example 1 — basic: filtering numbers and strings

```js
const temperatures = [-5, 0, 12, 20, -3, 33, 8, -1, 25];

// Only freezing temperatures
const freezing = temperatures.filter((t) => t < 0);
console.log(freezing); // [-5, -3, -1]

// Only warm temperatures
const warm = temperatures.filter((t) => t >= 20);
console.log(warm); // [20, 33, 25]

// ---- Strings ----
const usernames = ["alice_dev", "BOB", "charlie123", "DIANA", "eve_42", "FRANK"];

// Only lowercase-starting usernames
const lowerUsers = usernames.filter((name) => name[0] === name[0].toLowerCase());
console.log(lowerUsers); // ["alice_dev", "charlie123", "eve_42"]

// Only names longer than 4 characters
const longNames = usernames.filter((name) => name.length > 4);
console.log(longNames); // ["alice_dev", "charlie123", "DIANA", "eve_42", "FRANK"]

// Originals untouched
console.log(temperatures.length); // 9
console.log(usernames.length);    // 6
```

---

## Example 2 — real world: filtering a product catalogue

```js
const products = [
  { id: 1, name: "Mechanical Keyboard", category: "peripherals", price: 89.99,  inStock: true,  rating: 4.7 },
  { id: 2, name: "Wireless Mouse",      category: "peripherals", price: 29.99,  inStock: true,  rating: 4.2 },
  { id: 3, name: "4K Monitor",          category: "displays",    price: 399.99, inStock: false, rating: 4.8 },
  { id: 4, name: "USB-C Hub",           category: "accessories", price: 49.99,  inStock: true,  rating: 3.9 },
  { id: 5, name: "Webcam Pro",          category: "peripherals", price: 99.99,  inStock: true,  rating: 4.5 },
  { id: 6, name: "Desk Mat XL",         category: "accessories", price: 34.99,  inStock: false, rating: 4.1 },
  { id: 7, name: "LED Desk Lamp",       category: "accessories", price: 44.99,  inStock: true,  rating: 4.3 }
];

// Only in-stock items
const available = products.filter((p) => p.inStock);
console.log(available.map((p) => p.name));
// ["Mechanical Keyboard", "Wireless Mouse", "USB-C Hub", "Webcam Pro", "LED Desk Lamp"]

// Only peripherals
const peripherals = products.filter((p) => p.category === "peripherals");
console.log(peripherals.map((p) => p.name));
// ["Mechanical Keyboard", "Wireless Mouse", "Webcam Pro"]

// Under £50, in stock, rated above 4.0
const budgetPicks = products.filter(
  (p) => p.price <= 50 && p.inStock && p.rating > 4.0
);
console.log(budgetPicks.map((p) => p.name));
// ["Wireless Mouse", "LED Desk Lamp"]

// Originals untouched
console.log(products.length); // 7
```

---

## Example 3 — real world: search / query filtering

```js
function searchProducts(products, query) {
  const q = query.toLowerCase().trim();

  if (!q) return products;  // empty query → return all

  return products.filter((product) =>
    product.name.toLowerCase().includes(q) ||
    product.category.toLowerCase().includes(q)
  );
}

console.log(searchProducts(products, "key").map((p) => p.name));
// ["Mechanical Keyboard"]

console.log(searchProducts(products, "acc").map((p) => p.name));
// ["USB-C Hub", "Desk Mat XL", "LED Desk Lamp"]

console.log(searchProducts(products, "").map((p) => p.name));
// All 7 products
```

---

## Example 4 — real world: data cleanup — remove invalid entries

```js
// Raw data from a form submission or CSV import — messy
const rawSubmissions = [
  { name: "Alice",   email: "alice@example.com",  age: 28  },
  { name: "",        email: "noemail",             age: 17  },
  { name: "Charlie", email: "charlie@example.com", age: 25  },
  { name: null,      email: "nobody@example.com",  age: -1  },
  { name: "Diana",   email: "diana@example.com",   age: 30  },
  { name: "Eve",     email: "",                    age: 22  }
];

// Keep only valid submissions
const validSubmissions = rawSubmissions.filter((submission) => {
  const hasName  = submission.name && submission.name.trim().length > 0;
  const hasEmail = submission.email && submission.email.includes("@");
  const isAdult  = submission.age >= 18;
  return hasName && hasEmail && isAdult;
});

console.log(validSubmissions);
// [
//   { name: "Alice",   email: "alice@example.com",   age: 28 },
//   { name: "Charlie", email: "charlie@example.com",  age: 25 },
//   { name: "Diana",   email: "diana@example.com",    age: 30 }
// ]
console.log(`${validSubmissions.length} of ${rawSubmissions.length} submissions are valid`);
// "3 of 6 submissions are valid"
```

---

## Example 5 — real world: chaining filter + map + join

```js
const employees = [
  { name: "Alice",   dept: "Engineering", salary: 95000, active: true  },
  { name: "Bob",     dept: "Marketing",   salary: 72000, active: true  },
  { name: "Charlie", dept: "Engineering", salary: 110000, active: false },
  { name: "Diana",   dept: "Engineering", salary: 88000, active: true  },
  { name: "Eve",     dept: "HR",          salary: 68000, active: true  }
];

// Active Engineering team — names formatted, sorted alphabetically
const engineeringRoster = employees
  .filter((emp) => emp.dept === "Engineering" && emp.active)  // filter step
  .map((emp) => `${emp.name} ($${emp.salary.toLocaleString()})`)  // transform step
  .sort()  // alphabetical
  .join(", ");  // final string

console.log(engineeringRoster);
// "Alice ($95,000), Diana ($88,000)"
```

Each step in the chain is clean, readable, and purposeful.

---

## Tricky things you'll encounter in the real world

### 1 — Falsy return values accidentally exclude elements

```js
const data = [0, 1, 2, false, 3, null, 4, undefined, 5, ""];

// ❌ WRONG — trying to filter out null and undefined but also losing 0, false, ""
const clean = data.filter(Boolean);
console.log(clean); // [1, 2, 3, 4, 5]
// 0, false, "", null, undefined all removed — is that what you wanted?

// ✅ RIGHT — be explicit about what you're removing
const noNullOrUndefined = data.filter((x) => x !== null && x !== undefined);
console.log(noNullOrUndefined); // [0, 1, 2, false, 3, 4, 5, ""]
```

`filter(Boolean)` is a convenient shorthand — but it removes ALL falsy values (0, "", false, null, undefined, NaN). Only use it when you want all of those removed.

---

### 2 — `filter` returns an empty array (not `null`/`undefined`) when nothing matches

```js
const premiumUsers = [
  { name: "Alice", plan: "premium" },
  { name: "Bob",   plan: "premium" }
];

const freeUsers = premiumUsers.filter((u) => u.plan === "free");
console.log(freeUsers);        // []  — empty array, not null
console.log(freeUsers.length); // 0
console.log(Array.isArray(freeUsers)); // true

// Always safe to chain — no crash
freeUsers.map((u) => u.name);  // []  — no error
freeUsers.forEach((u) => ...); // runs zero times — no error
```

---

### 3 — `filter` does not modify the original — changes to the filtered array don't affect the source

```js
const allOrders = [
  { id: 1, status: "pending" },
  { id: 2, status: "shipped" },
  { id: 3, status: "pending" }
];

const pendingOrders = allOrders.filter((o) => o.status === "pending");

// Mutating the OBJECTS inside pendingOrders affects allOrders too (same references!)
pendingOrders[0].status = "processing";

console.log(allOrders[0].status); // "processing"  ⚠️ — allOrders was affected!

// Fix: clone the objects when you need independence
const safePending = allOrders
  .filter((o) => o.status === "pending")
  .map((o) => ({ ...o }));   // spread creates new objects

safePending[0].status = "processing";
console.log(allOrders[0].status); // "pending"  ✅
```

---

### 4 — `filter` is not the right tool to both remove AND transform — chain it with `map`

```js
const cartItems = [
  { name: "Keyboard", price: 89.99, qty: 1, inStock: true  },
  { name: "Mouse",    price: 29.99, qty: 2, inStock: false },
  { name: "Monitor",  price: 399.99, qty: 1, inStock: true }
];

// ❌ WRONG — mixing filtering and transformation inside one filter callback
const result = cartItems.filter((item) => {
  if (!item.inStock) return false;
  item.total = item.price * item.qty;  // ❌ mutating inside filter
  return true;
});

// ✅ RIGHT — filter first, then map to transform (separate concerns)
const result = cartItems
  .filter((item) => item.inStock)
  .map((item) => ({ ...item, total: parseFloat((item.price * item.qty).toFixed(2)) }));

console.log(result);
// [
//   { name: "Keyboard", price: 89.99, qty: 1, inStock: true, total: 89.99  },
//   { name: "Monitor",  price: 399.99, qty: 1, inStock: true, total: 399.99 }
// ]
```

---

### 5 — Performance: filtering large arrays with complex conditions

```js
// ❌ Calling an expensive function on every element in a large array
const results = bigArray.filter((item) => expensiveCheck(item));

// ✅ Cache values before filtering when possible
const processed = bigArray.map((item) => ({
  ...item,
  score: expensiveCheck(item)   // compute once
}));
const results = processed.filter((item) => item.score > threshold);
```

---

### 6 — Using the `index` parameter to filter by position

```js
const logs = ["alpha", "beta", "gamma", "delta", "epsilon"];

// Every other element (even indices)
const everyOther = logs.filter((_, index) => index % 2 === 0);
console.log(everyOther); // ["alpha", "gamma", "epsilon"]

// Skip the first and last
const middle = logs.filter((_, index) => index > 0 && index < logs.length - 1);
console.log(middle); // ["beta", "gamma", "delta"]

// The `_` convention means "I don't need the element, only the index"
```

---

## Common mistakes

### ❌ Mistake 1 — Forgetting to `return` in a multi-line callback

```js
const orders = [{ amount: 100 }, { amount: 50 }, { amount: 200 }];

// WRONG — no return → callback returns undefined (falsy) → everything filtered out
const bigOrders = orders.filter((order) => {
  order.amount > 100;   // ❌ just an expression — not returned
});
console.log(bigOrders); // []  ← everything excluded!

// RIGHT
const bigOrders = orders.filter((order) => {
  return order.amount > 100;  // ✅
});
// Or with implicit return:
const bigOrders = orders.filter((order) => order.amount > 100); // ✅
```

---

### ❌ Mistake 2 — Modifying elements inside the `filter` callback

```js
const users = [{ name: "Alice", active: true }, { name: "Bob", active: false }];

// WRONG — mutating inside filter creates a confusing mix of concerns
const activeUsers = users.filter((user) => {
  user.lastChecked = Date.now();   // ❌ side effect inside a filter callback
  return user.active;
});

// RIGHT — filter is for selecting; keep callbacks pure
const activeUsers = users.filter((user) => user.active);
// Do transformations separately with map
```

---

### ❌ Mistake 3 — Using `filter` to check if something exists (use `some` instead)

```js
const roles = ["viewer", "editor", "admin"];

// WRONG — using filter just to check existence (overkill)
const hasAdmin = roles.filter((r) => r === "admin").length > 0;

// RIGHT — use `some` for existence checks
const hasAdmin = roles.some((r) => r === "admin");  // ✅ stops at first match
```

---

## Frequently asked questions

**Q: Does `filter` return a new array or modify the original?**  
Always returns a new array. The original is never modified by `filter` itself. However, if you mutate the objects inside the callback, those mutations affect the originals (since objects are passed by reference).

**Q: What happens if no element passes the condition?**  
`filter` returns an empty array `[]` — not `null`, not `undefined`. It is always safe to chain further operations on the result.

**Q: Can I use `filter` to deduplicate an array?**  
Yes: `arr.filter((item, index) => arr.indexOf(item) === index)`. But `[...new Set(arr)]` is more efficient and readable for primitive values.

**Q: What's the difference between `filter` and `find`?**  
`filter` returns ALL matching elements as an array. `find` returns the FIRST matching element (not an array). Use `find` when you expect at most one match.

**Q: Can I chain `filter` multiple times?**  
Yes: `arr.filter(conditionA).filter(conditionB)`. But combining conditions into a single `filter` is more efficient (one pass instead of two).

**Q: Is `filter(Boolean)` safe to use?**  
Only when you genuinely want to remove ALL falsy values (null, undefined, 0, "", false, NaN). If you only want to remove null/undefined, write `filter((x) => x != null)` (loose inequality catches both).

---

## Practice exercises

### Exercise 1 — easy

Given:
```js
const numbers = [4, 15, 8, 23, 42, 7, 16, 3, 30, 11, 50, 1];
```

Use `filter` to produce four separate arrays:
1. `evenNumbers` — only even numbers
2. `greaterThan15` — only numbers greater than 15
3. `singleDigit` — only numbers with a single digit (< 10)
4. `divisibleBy3` — only numbers divisible by 3

Log all four and confirm the original is untouched.

```js
// Write your code here
```

---

### Exercise 2 — medium

You run a job board. Filter the listings based on user preferences:

```js
const jobListings = [
  { id: 1,  title: "Frontend Developer",    company: "Acme",     salary: 85000,  remote: true,  experienceYears: 2 },
  { id: 2,  title: "Backend Engineer",      company: "Globex",   salary: 110000, remote: false, experienceYears: 5 },
  { id: 3,  title: "UI/UX Designer",        company: "Initech",  salary: 72000,  remote: true,  experienceYears: 3 },
  { id: 4,  title: "DevOps Engineer",       company: "Umbrella", salary: 120000, remote: true,  experienceYears: 6 },
  { id: 5,  title: "Junior JS Developer",   company: "Acme",     salary: 55000,  remote: true,  experienceYears: 0 },
  { id: 6,  title: "Full Stack Developer",  company: "Globex",   salary: 95000,  remote: false, experienceYears: 4 },
  { id: 7,  title: "Data Analyst",          company: "Initech",  salary: 78000,  remote: true,  experienceYears: 2 }
];
```

Write a function `filterJobs(listings, preferences)` where `preferences` is an object that may have any combination of:
- `minSalary` — exclude jobs below this
- `remoteOnly` — if `true`, only remote jobs
- `maxExperience` — exclude jobs requiring more years than this
- `keyword` — only jobs whose title includes this string (case-insensitive)

Test with: `{ minSalary: 75000, remoteOnly: true, maxExperience: 3, keyword: "developer" }`

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **multi-stage data pipeline** that cleans, filters, and reports on a dataset of user activity logs.

```js
const activityLogs = [
  { userId: 1,  action: "login",    timestamp: "2026-04-01T08:00:00Z", duration: 0,   success: true  },
  { userId: 2,  action: "purchase", timestamp: "2026-04-01T09:15:00Z", duration: 120, success: true  },
  { userId: 1,  action: "search",   timestamp: "2026-04-01T10:00:00Z", duration: 5,   success: true  },
  { userId: 3,  action: "login",    timestamp: "2026-04-02T07:30:00Z", duration: 0,   success: false },
  { userId: 2,  action: "logout",   timestamp: "2026-04-02T11:00:00Z", duration: 0,   success: true  },
  { userId: 4,  action: "purchase", timestamp: "2026-04-02T14:00:00Z", duration: 95,  success: true  },
  { userId: 1,  action: "purchase", timestamp: "2026-04-03T09:00:00Z", duration: 200, success: true  },
  { userId: 3,  action: "login",    timestamp: "2026-04-03T08:00:00Z", duration: 0,   success: true  },
  { userId: 5,  action: "search",   timestamp: null,                   duration: -1,  success: true  },
  { userId: null, action: "purchase", timestamp: "2026-04-04T10:00:00Z", duration: 50, success: true }
];
```

Using `filter` (and `map` where needed):
1. **Clean** — remove logs where `userId` is null/undefined OR `timestamp` is null OR `duration < 0`
2. **Successful purchases only** — from the cleaned logs, keep only `action === "purchase"` AND `success === true`
3. **High-value sessions** — from successful purchases, keep only those with `duration >= 100`
4. **Report** — produce a final array of strings: `"User 2 made a high-value purchase (120s) on 2026-04-01"`

Log each stage's count and the final report array.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

```js
const result = array.filter((element, index) => {
  return boolean;   // true = keep, false = discard
});
```

**Common filter patterns:**

```js
// By value
arr.filter((x) => x > 10)
arr.filter((x) => x !== null)
arr.filter((x) => x !== null && x !== undefined)  // remove null/undefined
arr.filter(Boolean)                                // remove ALL falsy values

// By property
arr.filter((item) => item.active)
arr.filter((item) => item.status === "shipped")
arr.filter((item) => item.price < 50 && item.inStock)

// By string content
arr.filter((item) => item.name.includes("search"))
arr.filter((item) => item.email.endsWith("@example.com"))

// By position
arr.filter((_, i) => i % 2 === 0)          // even indices only
arr.filter((_, i) => i < 5)                // first 5 elements

// Deduplicate primitives
arr.filter((v, i, a) => a.indexOf(v) === i)
```

**Key rules:**
- Returns a new array — never modifies the original
- Result can be shorter than original — never longer
- Empty array `[]` returned when nothing matches (not null/undefined)
- Callback must return a truthy/falsy value — missing `return` = everything excluded
- Don't mutate elements inside `filter` callbacks
- Use `some` to just check if anything matches; use `find` for the first match
- Chain `filter` → `map` when you need to both select AND transform

---

## Connected topics

- **27 — map** — transforms every element; chain `filter().map()` to select then reshape
- **29 — reduce** — can replicate `filter` and do much more; the Swiss army knife
- **30 — find and findIndex** — returns the first matching element instead of all matches
- **31 — some and every** — boolean checks across the array; `some` is faster than `filter().length > 0`
