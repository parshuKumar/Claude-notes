# 26 — forEach

## What is this?
`forEach` is an array method that **calls a function once for every element** in the array, in order. Think of it like a delivery worker going door-to-door down a street — they knock on every door, do their job at each one, and move on. You hand them instructions (the callback), and they execute those instructions at each stop. `forEach` never skips, never returns early, and never produces a new array.

## Why does it matter?
`forEach` is the cleanest way to iterate over an array when you only care about **doing something** with each element — logging, updating the DOM, sending data — and you don't need a transformed copy back. It replaces the old `for (let i = 0; i < array.length; i++)` pattern with something more readable and intention-revealing.

---

## Syntax

```js
array.forEach(function (element, index, array) {
  // do something with element
});

// Arrow function shorthand (most common)
array.forEach((element, index) => {
  // do something
});

// Only care about the element (index and array are optional)
array.forEach((element) => {
  console.log(element);
});
```

**Callback parameters:**

| Parameter | What it is | Required? |
|---|---|---|
| `element` | The current item | ✅ Yes (in practice) |
| `index` | The position (0-based) | ❌ Optional |
| `array` | The original array | ❌ Optional (rarely used) |

---

## How it works — line by line

```js
const productNames = ["Keyboard", "Mouse", "Monitor"];

productNames.forEach((product, index) => {
  console.log((index + 1) + ". " + product);
});
```

1. `forEach` is called on `productNames`
2. It starts at index `0` — calls the callback with `("Keyboard", 0, productNames)`
3. The callback runs: logs `"1. Keyboard"`
4. Moves to index `1` — calls the callback with `("Mouse", 1, productNames)`
5. Logs `"2. Mouse"`
6. Moves to index `2` — calls the callback with `("Monitor", 2, productNames)`
7. Logs `"3. Monitor"`
8. No more elements — `forEach` finishes and **returns `undefined`**

The key point: `forEach` **always returns `undefined`**. It is purely for side effects.

---

## Example 1 — basic: rendering a list

```js
const notifications = [
  "Your order has shipped",
  "New message from Alice",
  "Scheduled maintenance tonight",
  "Your subscription renews tomorrow"
];

// Log each notification with its position
notifications.forEach((message, index) => {
  console.log(`[${index + 1}/${notifications.length}] ${message}`);
});
// [1/4] Your order has shipped
// [2/4] New message from Alice
// [3/4] Scheduled maintenance tonight
// [4/4] Your subscription renews tomorrow

// Build an HTML string from the array
let listHtml = "<ul>";

notifications.forEach((message) => {
  listHtml += `<li>${message}</li>`;
});

listHtml += "</ul>";
console.log(listHtml);
// <ul><li>Your order has shipped</li><li>New message from Alice</li>...</ul>
```

---

## Example 2 — real world: updating DOM elements

```js
// Attach event listeners to all nav links
const navLinks = document.querySelectorAll(".nav-link");

navLinks.forEach((link) => {
  link.addEventListener("click", (event) => {
    // Remove active class from all links first
    navLinks.forEach((l) => l.classList.remove("active"));
    // Add active class to the clicked link
    event.currentTarget.classList.add("active");
  });
});

// Apply a discount badge to all sale items
const saleItems = document.querySelectorAll(".product-card.on-sale");

saleItems.forEach((card) => {
  const badge = document.createElement("span");
  badge.className = "discount-badge";
  badge.textContent = "SALE";
  card.appendChild(badge);
});
```

`forEach` works on `NodeList` (what `querySelectorAll` returns) as well as arrays.

---

## Example 3 — real world: aggregating data (accumulating into an external variable)

```js
const orders = [
  { orderId: "A001", customer: "Alice",   amount: 120.50, status: "shipped"   },
  { orderId: "A002", customer: "Bob",     amount: 89.99,  status: "pending"   },
  { orderId: "A003", customer: "Charlie", amount: 245.00, status: "shipped"   },
  { orderId: "A004", customer: "Diana",   amount: 59.99,  status: "cancelled" },
  { orderId: "A005", customer: "Eve",     amount: 310.00, status: "pending"   }
];

// Accumulate totals by status
const summary = { shipped: 0, pending: 0, cancelled: 0 };
let grandTotal = 0;

orders.forEach((order) => {
  summary[order.status] += order.amount;
  grandTotal += order.amount;
});

console.log("Shipped total:   $" + summary.shipped.toFixed(2));   // $365.50
console.log("Pending total:   $" + summary.pending.toFixed(2));   // $399.99
console.log("Cancelled total: $" + summary.cancelled.toFixed(2)); // $59.99
console.log("Grand total:     $" + grandTotal.toFixed(2));        // $825.48
```

The mutation here is to `summary` and `grandTotal` — external variables, not the array itself. This is a deliberate side effect.

---

## Example 4 — real world: sending data / side effects

```js
const failedJobs = [
  { id: "job-001", type: "email",  payload: { to: "alice@example.com" } },
  { id: "job-002", type: "report", payload: { reportId: 42 } },
  { id: "job-003", type: "email",  payload: { to: "bob@example.com" } }
];

// Re-queue each failed job (purely side-effect work — no return value needed)
failedJobs.forEach((job) => {
  console.log(`Re-queuing job ${job.id} (type: ${job.type})`);
  // jobQueue.add(job);  // in real code: send to a message queue, API, etc.
});

// Log errors to an external service
const errorLog = [];

failedJobs.forEach((job) => {
  errorLog.push({
    jobId:     job.id,
    timestamp: new Date().toISOString(),
    message:   `Job ${job.id} failed and was re-queued`
  });
});

console.log(errorLog.length); // 3
```

---

## Tricky things you'll encounter in the real world

### 1 — `forEach` ALWAYS returns `undefined` — you cannot use its return value

```js
const prices = [10, 20, 30];

// ❌ WRONG — thinking you can capture a new array from forEach
const doubled = prices.forEach((price) => price * 2);
console.log(doubled); // undefined  ← forEach never returns anything useful

// ✅ RIGHT — use map when you need a new array
const doubled = prices.map((price) => price * 2);
console.log(doubled); // [20, 40, 60]
```

**Rule of thumb:** If you need a new array → use `map`. If you just need to *do something* → use `forEach`.

---

### 2 — You cannot `break` or `return` out of `forEach`

```js
const userIds = [1, 2, 3, 4, 5];

// ❌ WRONG — `return` inside forEach just exits the CALLBACK for that iteration
// it does NOT stop forEach from continuing
userIds.forEach((id) => {
  if (id === 3) return;   // skips id 3, but forEach continues to 4 and 5
  console.log(id);        // logs 1, 2, 4, 5
});

// There is NO way to break out of forEach early
// If you need early exit, use a classic for loop or for...of
for (const id of userIds) {
  if (id === 3) break;    // ✅ actually stops the loop
  console.log(id);        // 1, 2
}
```

---

### 3 — `forEach` skips holes in sparse arrays

```js
const sparse = [1, , , 4, 5];   // holes at indices 1 and 2

sparse.forEach((item, index) => {
  console.log(index, item);
});
// 0 1
// 3 4
// 4 5
// Indices 1 and 2 are silently skipped!

// Classic for loop DOES include holes (as undefined)
for (let i = 0; i < sparse.length; i++) {
  console.log(i, sparse[i]);
}
// 0 1, 1 undefined, 2 undefined, 3 4, 4 5
```

---

### 4 — `forEach` does not wait for async callbacks

```js
const userIds = [1, 2, 3];

// ❌ WRONG — forEach does not await the async callback
// All three fetches are started simultaneously, forEach returns before any finish
userIds.forEach(async (id) => {
  const user = await fetchUser(id);  // async callback — forEach doesn't care
  console.log(user.name);
});
console.log("forEach finished");
// "forEach finished" prints FIRST, then user names arrive in unpredictable order

// ✅ RIGHT — use a for...of loop with await if you need sequential async
for (const id of userIds) {
  const user = await fetchUser(id);  // waits for each one before moving to next
  console.log(user.name);
}

// ✅ RIGHT — use Promise.all + map if you want parallel but need to wait for all
const users = await Promise.all(userIds.map((id) => fetchUser(id)));
users.forEach((user) => console.log(user.name));
```

This is one of the most common async mistakes in JavaScript.

---

### 5 — Modifying the array during `forEach` has unpredictable results

```js
const items = [1, 2, 3, 4, 5];

// ❌ Don't push/splice the array being iterated
items.forEach((item, index) => {
  if (item === 3) items.push(99);  // ⚠️ array grows mid-iteration — forEach uses original length
  console.log(item);
});
// forEach captured the length at start — the 99 is never visited

// Don't rely on this behaviour — never mutate the array you're iterating
```

---

### 6 — `forEach` on a `NodeList` — not the same as an Array

```js
const buttons = document.querySelectorAll("button"); // NodeList, not Array

// ✅ NodeList has forEach — this works
buttons.forEach((btn) => btn.disabled = true);

// ❌ But most other array methods are NOT available on NodeList
buttons.map((btn) => btn.textContent);   // TypeError: buttons.map is not a function

// Convert to Array first to use all array methods
const btnArray = Array.from(buttons);
const labels   = btnArray.map((btn) => btn.textContent);
```

---

## Common mistakes

### ❌ Mistake 1 — Using `forEach` when you need a return value (`map`)

```js
const temperatures = [0, 20, 37, 100];

// WRONG — forEach returns undefined
const fahrenheit = temperatures.forEach((c) => (c * 9) / 5 + 32);
console.log(fahrenheit); // undefined

// RIGHT — map returns a new array
const fahrenheit = temperatures.map((c) => (c * 9) / 5 + 32);
console.log(fahrenheit); // [32, 68, 98.6, 212]
```

---

### ❌ Mistake 2 — Using `return` expecting it to stop forEach

```js
const transactions = [150, -30, 200, -500, 100];

// WRONG — thinking return false stops the loop (like jQuery's .each())
transactions.forEach((amount) => {
  if (amount < -100) return false;  // does NOT stop forEach — just skips this callback
  console.log(amount);              // still logs 100 after the -500 item
});

// RIGHT — if you need to stop early, use for...of
for (const amount of transactions) {
  if (amount < -100) break;
  console.log(amount);  // stops at -500
}
```

---

### ❌ Mistake 3 — Forgetting `forEach` doesn't chain (returns `undefined`)

```js
const items = ["apple", "banana", "cherry"];

// WRONG — trying to chain after forEach
const result = items
  .forEach((item) => console.log(item))
  .map((item) => item.toUpperCase());   // ❌ TypeError: Cannot read properties of undefined

// RIGHT — forEach is a terminal operation, don't chain after it
items.forEach((item) => console.log(item));
const upper = items.map((item) => item.toUpperCase()); // separate statement
```

---

## Frequently asked questions

**Q: When should I use `forEach` vs `map`?**  
Use `forEach` when you want to *do something* (side effects: log, update DOM, send data) and don't need a result array. Use `map` when you want to *transform* each element and get a new array back.

**Q: When should I use `forEach` vs a `for...of` loop?**  
They're nearly identical in behaviour. `for...of` is preferable when you need `break`/`continue` or `await`. `forEach` is slightly more expressive for pure iteration (reads "for each item, do this"). It's a style choice — pick one and be consistent.

**Q: Can I use `forEach` on strings?**  
No — strings are not arrays. Convert to an array first: `[..."hello"].forEach(...)` or `Array.from("hello").forEach(...)`.

**Q: Does `forEach` modify the original array?**  
`forEach` itself doesn't — but your callback can. If your callback mutates the element or calls mutation methods, the array will change. `forEach` is not inherently safe from mutation — that depends entirely on what you do inside the callback.

**Q: What does `forEach` return?**  
Always `undefined`. This is by design — `forEach` is for side effects only. If you find yourself trying to use the return value of `forEach`, switch to `map`, `filter`, or `reduce`.

**Q: Is `forEach` synchronous?**  
Yes — `forEach` itself is synchronous. It calls the callback for each element in order and waits for each synchronous callback to complete before moving to the next. If the callback is `async`, `forEach` doesn't await it (see Tricky #4).

---

## Practice exercises

### Exercise 1 — easy

Given this array of student objects:
```js
const students = [
  { name: "Alice",   grade: 92 },
  { name: "Bob",     grade: 74 },
  { name: "Charlie", grade: 88 },
  { name: "Diana",   grade: 65 },
  { name: "Eve",     grade: 97 }
];
```

Use `forEach` to:
1. Print each student in the format: `"1. Alice — 92/100"`
2. Count how many students scored 80 or above (use an external counter variable)
3. Build and log a single string: `"Top students: Alice, Charlie, Eve"` (only those with grade >= 80)

```js
// Write your code here
```

---

### Exercise 2 — medium

You have an inventory array:
```js
const inventory = [
  { sku: "KB-001", name: "Mechanical Keyboard", stock: 15,  price: 89.99  },
  { sku: "MS-002", name: "Wireless Mouse",       stock: 0,   price: 29.99  },
  { sku: "MN-003", name: "4K Monitor",           stock: 4,   price: 399.99 },
  { sku: "WC-004", name: "1080p Webcam",         stock: 0,   price: 69.99  },
  { sku: "HS-005", name: "Noise-Cancelling Headset", stock: 8, price: 149.99 }
];
```

Use `forEach` to:
1. Build a `stockReport` object where each key is the `sku` and the value is `{ name, stock, status }` — status is `"IN STOCK"` if stock > 0, otherwise `"OUT OF STOCK"`
2. Calculate the total value of all inventory (`stock * price` per item, summed)
3. Collect all out-of-stock item names into an `outOfStockList` array
4. Log the `stockReport`, `totalInventoryValue` (formatted to 2 decimal places), and `outOfStockList`

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **simple event emitter** using `forEach` at its core.

Write a function `createEventEmitter()` that returns an object with:
- `on(eventName, callback)` — registers a callback for an event (multiple callbacks per event allowed)
- `off(eventName, callback)` — removes a specific callback for an event
- `emit(eventName, ...args)` — calls **all** registered callbacks for `eventName` using `forEach`, passing `...args` to each

Requirements:
- Store listeners in a private object (e.g. `{ "login": [fn1, fn2], "logout": [fn3] }`)
- `emit` should use `forEach` to call every listener
- If no listeners are registered for an event, `emit` does nothing silently
- `off` should only remove the exact function reference passed

Test it:
```js
const emitter = createEventEmitter();

function onLogin(user) { console.log("Handler 1: Welcome,", user.name); }
function onLogin2(user) { console.log("Handler 2: Logging activity for", user.name); }

emitter.on("login", onLogin);
emitter.on("login", onLogin2);
emitter.emit("login", { name: "Parsh" });   // both handlers fire

emitter.off("login", onLogin);
emitter.emit("login", { name: "Parsh" });   // only Handler 2 fires
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

```js
array.forEach((element, index, array) => {
  // side effect only — do NOT try to return a new array
});
```

**forEach vs alternatives:**

| Need | Use |
|---|---|
| Do something with each element | `forEach` |
| Transform each element → new array | `map` |
| Filter elements → new array | `filter` |
| Collapse to a single value | `reduce` |
| Need `break` or `continue` | `for...of` |
| Need `await` inside the loop | `for...of` |
| Need index + ability to break | classic `for` loop |

**Key rules:**
- `forEach` always returns `undefined` — never assign its result
- `return` inside forEach exits only the current callback iteration — it does NOT stop the loop
- No `break` in `forEach` — use `for...of` if you need early exit
- `forEach` is synchronous — async callbacks are fired but not awaited
- `forEach` skips holes in sparse arrays
- Works on `NodeList` but NOT all array methods do — use `Array.from()` to convert

---

## Connected topics

- **27 — map** — like `forEach` but returns a new transformed array; use when you need a result
- **28 — filter** — selects elements matching a condition into a new array
- **29 — reduce** — the most powerful iterator; collapses an array to any single value
- **26 → 29 are all HOFs** — they all take a callback function, making them higher-order functions (topic 20)
