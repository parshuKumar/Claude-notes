# 29 — reduce

## What is this?
`reduce` is an array method that **processes every element one by one and collapses the entire array into a single value**. That "single value" can be a number, a string, an object, another array, or anything else. Think of it like a snowball rolling down a hill — it picks up each piece of snow (element) as it goes, growing bigger and bigger, until it reaches the bottom with everything accumulated inside.

## Why does it matter?
`reduce` is the most powerful and versatile array method. It can replicate `map`, `filter`, `forEach`, and more — all in one pass. It's the right tool any time you need to calculate a total, group items by a property, build an object from an array, count occurrences, or collapse nested structures. Every senior JavaScript developer uses it constantly.

---

## Syntax

```js
const result = array.reduce(function (accumulator, element, index, array) {
  return updatedAccumulator;  // returned value becomes the new accumulator
}, initialValue);

// Arrow function shorthand
const result = array.reduce((acc, element) => acc + element, 0);
```

**Parameters:**

| Parameter | What it is |
|---|---|
| `accumulator` (acc) | The running result — starts as `initialValue`, updated each iteration |
| `element` (current) | The current array element being processed |
| `index` | The current index (optional) |
| `array` | The original array (optional, rarely used) |
| `initialValue` | The starting value of the accumulator — **always provide this** |

**Critical rule:** Always provide an `initialValue`. Omitting it causes subtle bugs with empty arrays and changes the starting index (covered in Tricky section).

---

## How it works — line by line

```js
const cartTotals = [29.99, 89.99, 49.99, 9.99];
const grandTotal = cartTotals.reduce((acc, price) => acc + price, 0);
```

Step by step — `acc` starts at `0` (the `initialValue`):

| Iteration | `acc` (before) | `price` | `acc + price` | `acc` (after) |
|---|---|---|---|---|
| 1 | `0` | `29.99` | `29.99` | `29.99` |
| 2 | `29.99` | `89.99` | `119.98` | `119.98` |
| 3 | `119.98` | `49.99` | `169.97` | `169.97` |
| 4 | `169.97` | `9.99` | `179.96` | `179.96` |

Final result: `179.96`

The accumulator is the "running total". Each call to your callback must **return the new accumulator** — forgetting `return` means `acc` becomes `undefined` on the next iteration.

---

## Example 1 — basic: sum, product, min, max

```js
const numbers = [3, 7, 2, 9, 4, 6, 1];

// Sum
const sum = numbers.reduce((acc, n) => acc + n, 0);
console.log(sum); // 32

// Product
const product = numbers.reduce((acc, n) => acc * n, 1);
console.log(product); // 3*7*2*9*4*6*1 = 9072

// Maximum
const max = numbers.reduce((acc, n) => (n > acc ? n : acc), numbers[0]);
console.log(max); // 9

// Minimum
const min = numbers.reduce((acc, n) => (n < acc ? n : acc), numbers[0]);
console.log(min); // 1

// String concatenation
const words = ["JavaScript", "is", "genuinely", "great"];
const sentence = words.reduce((acc, word) => acc + " " + word);
// No initial value — first element becomes acc, iteration starts at index 1
console.log(sentence); // "JavaScript is genuinely great"
```

---

## Example 2 — real world: order summary with multiple fields

```js
const orderItems = [
  { name: "Mechanical Keyboard", price: 89.99,  qty: 1, category: "peripherals" },
  { name: "Wireless Mouse",      price: 29.99,  qty: 2, category: "peripherals" },
  { name: "4K Monitor",          price: 399.99, qty: 1, category: "displays"    },
  { name: "USB-C Hub",           price: 49.99,  qty: 2, category: "accessories" },
  { name: "Desk Mat",            price: 24.99,  qty: 1, category: "accessories" }
];

const orderSummary = orderItems.reduce(
  (acc, item) => {
    const lineTotal = item.price * item.qty;

    return {
      subtotal:   acc.subtotal + lineTotal,
      itemCount:  acc.itemCount + item.qty,
      lineCount:  acc.lineCount + 1
    };
  },
  { subtotal: 0, itemCount: 0, lineCount: 0 }   // initial value is an object
);

console.log(orderSummary);
// { subtotal: 654.93, itemCount: 8, lineCount: 5 }

const tax  = orderSummary.subtotal * 0.1;
const total = orderSummary.subtotal + tax;
console.log(`Subtotal: $${orderSummary.subtotal.toFixed(2)}`); // $654.93
console.log(`Tax (10%): $${tax.toFixed(2)}`);                  // $65.49
console.log(`Total: $${total.toFixed(2)}`);                    // $720.42
```

---

## Example 3 — real world: groupBy — group items by a property

```js
const employees = [
  { name: "Alice",   dept: "Engineering", salary: 95000  },
  { name: "Bob",     dept: "Marketing",   salary: 72000  },
  { name: "Charlie", dept: "Engineering", salary: 110000 },
  { name: "Diana",   dept: "HR",          salary: 68000  },
  { name: "Eve",     dept: "Marketing",   salary: 78000  },
  { name: "Frank",   dept: "Engineering", salary: 88000  }
];

// Group employees by department
const byDepartment = employees.reduce((acc, employee) => {
  const dept = employee.dept;

  if (!acc[dept]) {
    acc[dept] = [];   // create the array for this dept if it doesn't exist yet
  }

  acc[dept].push(employee);
  return acc;
}, {});

console.log(byDepartment);
// {
//   Engineering: [Alice, Charlie, Frank],
//   Marketing:   [Bob, Eve],
//   HR:          [Diana]
// }

// Now you can easily get engineering headcount and total salary
const engTeam      = byDepartment["Engineering"];
const engHeadcount = engTeam.length;
const engTotalSalary = engTeam.reduce((sum, emp) => sum + emp.salary, 0);
console.log(`Engineering: ${engHeadcount} people, $${engTotalSalary.toLocaleString()} total`);
// Engineering: 3 people, $293,000 total
```

---

## Example 4 — real world: count occurrences

```js
const pageViews = [
  "/home", "/shop", "/home", "/product/42", "/shop",
  "/home", "/cart", "/shop", "/product/42", "/checkout"
];

const viewCounts = pageViews.reduce((acc, page) => {
  acc[page] = (acc[page] || 0) + 1;   // increment or initialise to 0 then 1
  return acc;
}, {});

console.log(viewCounts);
// { "/home": 3, "/shop": 3, "/product/42": 2, "/cart": 1, "/checkout": 1 }

// Find the most-visited page
const mostVisited = Object.entries(viewCounts).reduce(
  (acc, [page, count]) => (count > acc.count ? { page, count } : acc),
  { page: "", count: 0 }
);
console.log(`Most visited: ${mostVisited.page} (${mostVisited.count} views)`);
// Most visited: /home (3 views)
```

---

## Example 5 — real world: flatten nested array (replicate `flat`)

```js
const weeklyTasks = [
  ["Standup", "Code review"],
  ["Feature work", "Testing"],
  ["Deploy", "Retrospective"]
];

const allTasks = weeklyTasks.reduce((acc, dayTasks) => {
  return acc.concat(dayTasks);  // or: [...acc, ...dayTasks]
}, []);

console.log(allTasks);
// ["Standup", "Code review", "Feature work", "Testing", "Deploy", "Retrospective"]
```

---

## Example 6 — real world: pipeline — replicate filter + map in one pass

```js
const transactions = [
  { id: 1, type: "credit", amount: 500.00 },
  { id: 2, type: "debit",  amount: 120.50 },
  { id: 3, type: "credit", amount: 1200.00 },
  { id: 4, type: "debit",  amount: 89.99  },
  { id: 5, type: "credit", amount: 75.00  }
];

// In one pass: keep only credits, format as strings
const creditSummaries = transactions.reduce((acc, tx) => {
  if (tx.type !== "credit") return acc;          // skip non-credits (filter)
  acc.push(`Credit #${tx.id}: +$${tx.amount.toFixed(2)}`); // transform + push (map)
  return acc;
}, []);

console.log(creditSummaries);
// ["Credit #1: +$500.00", "Credit #3: +$1200.00", "Credit #5: +$75.00"]
```

---

## Tricky things you'll encounter in the real world

### 1 — Always provide an `initialValue`

```js
const values = [5, 10, 15];

// With initialValue — accumulator starts at 0, all 3 elements are processed
values.reduce((acc, n) => acc + n, 0);  // 0+5 → 5+10 → 15+15 = 30

// Without initialValue — first element (5) becomes acc, iteration starts at index 1
values.reduce((acc, n) => acc + n);     // 5+10 → 15+15 = 30  (same result here)

// ❌ DANGER — empty array with no initialValue throws a TypeError
[].reduce((acc, n) => acc + n);  // TypeError: Reduce of empty array with no initial value

// ✅ SAFE — empty array with initialValue returns the initialValue
[].reduce((acc, n) => acc + n, 0);  // 0 — safe
```

**Always provide `initialValue`.** It costs nothing and prevents crashes on empty arrays.

---

### 2 — Forgetting to `return` the accumulator

```js
const nums = [1, 2, 3, 4];

// ❌ WRONG — forgot to return acc after modifying it
const result = nums.reduce((acc, n) => {
  acc += n;   // modifies the local variable, but...
              // implicit return is undefined — acc becomes undefined next iteration!
}, 0);
console.log(result); // NaN or undefined

// ✅ RIGHT — always return the updated accumulator
const result = nums.reduce((acc, n) => {
  return acc + n;
}, 0);

// ✅ Also right — implicit return (arrow function, no braces)
const result = nums.reduce((acc, n) => acc + n, 0);
```

This is the single most common `reduce` mistake.

---

### 3 — Mutating the accumulator object vs returning a new one

```js
const items = [{ cat: "A" }, { cat: "B" }, { cat: "A" }];

// ✅ Mutation approach — works fine for plain accumulator objects
const grouped = items.reduce((acc, item) => {
  if (!acc[item.cat]) acc[item.cat] = 0;
  acc[item.cat]++;
  return acc;   // ← returning the SAME object (mutation is ok here — it's our accumulator)
}, {});
// { A: 2, B: 1 }

// ❌ WRONG — returning a brand new object each time (works but very inefficient for large arrays)
const grouped = items.reduce((acc, item) => {
  return {
    ...acc,
    [item.cat]: (acc[item.cat] || 0) + 1
  };
}, {});
```

For accumulator objects, mutate-and-return is both correct and efficient. Spread on every iteration creates O(n²) work.

---

### 4 — Using `reduce` to build an array — remember to return `acc`

```js
const data = [1, -2, 3, -4, 5];

// ❌ WRONG — pushing to acc but not returning it
const positives = data.reduce((acc, n) => {
  if (n > 0) acc.push(n);
  // ← forgot return acc! → acc becomes undefined on next iteration
}, []);

// ✅ RIGHT
const positives = data.reduce((acc, n) => {
  if (n > 0) acc.push(n);
  return acc;   // always return the accumulator
}, []);
console.log(positives); // [1, 3, 5]
```

---

### 5 — `reduceRight` — processes from right to left

```js
const path = ["home", "users", "parsh", "documents"];

// reduce — left to right
const leftPath = path.reduce((acc, segment) => acc + "/" + segment);
console.log(leftPath); // "home/users/parsh/documents"

// reduceRight — right to left
const rightPath = path.reduceRight((acc, segment) => acc + "/" + segment);
console.log(rightPath); // "documents/parsh/users/home"

// Useful for composing functions right-to-left (like mathematical composition)
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);

const process = compose(
  (x) => x * 2,    // applied third
  (x) => x + 10,   // applied second
  (x) => x - 1     // applied first
);
console.log(process(5)); // ((5-1) + 10) * 2 = 28
```

---

### 6 — Early exit: `reduce` cannot stop early

```js
// ❌ reduce always processes every element — there's no way to break out
// Even if you find what you need on the first element, all others are still visited

// For early exit, use find, some, or a for...of loop
const users = [{ id: 1 }, { id: 2 }, { id: 3 }];

// Wrong tool for "find first match"
const found = users.reduce((acc, user) => {
  if (user.id === 2) return user;
  return acc;
}, null);
// Works, but visits ALL elements even after finding id 2

// Right tool
const found = users.find((user) => user.id === 2); // stops at first match
```

---

## Common mistakes

### ❌ Mistake 1 — No `return` in callback (most common)

```js
const prices = [10, 20, 30];

// WRONG — no return, acc becomes undefined after first iteration
const total = prices.reduce((acc, price) => {
  acc + price;   // ❌ expression evaluated, not returned
});
console.log(total); // NaN

// RIGHT
const total = prices.reduce((acc, price) => acc + price, 0); // ✅
```

---

### ❌ Mistake 2 — Wrong initialValue type

```js
const words = ["hello", "world", "foo"];

// WRONG — initial value is 0 (number), but we want a string
const sentence = words.reduce((acc, word) => acc + " " + word, 0);
console.log(sentence); // "0 hello world foo"  ← "0" prepended

// RIGHT — initial value matches what you're building
const sentence = words.reduce((acc, word) => acc + " " + word, "");
console.log(sentence); // " hello world foo"  ← leading space

// BETTER — no initial value (first element becomes acc, no extra space)
const sentence = words.reduce((acc, word) => acc + " " + word);
console.log(sentence); // "hello world foo"  ✅
```

---

### ❌ Mistake 3 — Using `reduce` when `map` or `filter` is clearer

```js
const numbers = [1, 2, 3, 4, 5];

// WRONG use of reduce — it works, but hides intent
const doubled = numbers.reduce((acc, n) => { acc.push(n * 2); return acc; }, []);

// RIGHT — use map, which communicates intent clearly
const doubled = numbers.map((n) => n * 2); // ✅ obviously a transformation
```

Use `reduce` when no other method fits cleanly. Prefer `map`, `filter`, `forEach` when they do the job.

---

## Frequently asked questions

**Q: When should I use `reduce` instead of `map` or `filter`?**  
Use `reduce` when you need a single output that is NOT a 1-to-1 transformation of every element: totals, counts, grouped objects, combined results from multiple fields, or when you need to filter AND transform in one pass. If `map` or `filter` cleanly express what you need, use those instead.

**Q: Can the accumulator be an array?**  
Yes — the initial value can be any type: number, string, object, array, Map, Set. Whatever makes sense for what you're building.

**Q: What happens if I `reduce` an empty array without an initial value?**  
`TypeError: Reduce of empty array with no initial value`. Always provide an initial value to be safe.

**Q: Can I use `reduce` to build a Map or Set?**  
Yes:
```js
const uniqueTags = posts.reduce((acc, post) => {
  post.tags.forEach((tag) => acc.add(tag));
  return acc;
}, new Set());
```

**Q: Is `reduce` slower than a `for` loop?**  
Marginally, because of function call overhead per iteration. For typical UI work this is irrelevant. For processing millions of items in tight loops, a `for` loop is faster. Optimise only when you measure a real bottleneck.

**Q: What's `reduceRight`?**  
Same as `reduce` but processes elements from right to left (last index to first). Useful for right-to-left function composition.

---

## Practice exercises

### Exercise 1 — easy

Given:
```js
const salaries = [48000, 72000, 95000, 61000, 83000, 54000, 110000];
```

Use `reduce` to calculate:
1. `totalPayroll` — sum of all salaries
2. `averageSalary` — totalPayroll divided by the number of employees (round to 2dp)
3. `highestSalary` — the maximum value
4. `lowestSalary` — the minimum value
5. `aboveAverage` — count of salaries strictly above the average

Log all five results.

```js
// Write your code here
```

---

### Exercise 2 — medium

Given this array of blog posts:
```js
const posts = [
  { id: 1, title: "Intro to JS",         tags: ["js", "beginner"],             views: 1200, published: true  },
  { id: 2, title: "Understanding Scope", tags: ["js", "intermediate"],          views: 850,  published: true  },
  { id: 3, title: "Node Basics",         tags: ["node", "beginner"],            views: 430,  published: false },
  { id: 4, title: "Closures Deep Dive",  tags: ["js", "advanced"],              views: 2100, published: true  },
  { id: 5, title: "REST API Design",     tags: ["node", "api", "intermediate"], views: 660,  published: true  },
  { id: 6, title: "Draft Post",          tags: ["js"],                          views: 0,    published: false }
];
```

Use a **single `reduce`** call (no other array methods) to produce one object:
```js
{
  totalViews: number,           // sum of views for published posts only
  publishedCount: number,       // count of published posts
  tagFrequency: { js: 3, ... }, // count of each tag across ALL posts
  mostViewedTitle: string       // title of the post with highest views (published only)
}
```

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **shopping cart reducer** — the same pattern used in Redux state management.

Write a function `cartReducer(state, action)` where:
- `state` is the current cart: `{ items: [], couponCode: null }`  
  Each item: `{ id, name, price, qty }`
- `action` is an object: `{ type, payload }`

Handle these action types:
- `"ADD_ITEM"` — if item with same `id` exists, increment `qty`; otherwise push new item
- `"REMOVE_ITEM"` — remove item by `payload.id`
- `"UPDATE_QTY"` — set `qty` of item with `payload.id` to `payload.qty`; if qty ≤ 0, remove the item
- `"APPLY_COUPON"` — set `couponCode` to `payload.code`
- `"CLEAR_CART"` — return `{ items: [], couponCode: null }`

The function must NEVER mutate `state` — always return a new state object.

Then use `reduce` to replay a sequence of actions from an initial state:
```js
const actions = [
  { type: "ADD_ITEM",    payload: { id: 1, name: "Keyboard", price: 89.99, qty: 1 } },
  { type: "ADD_ITEM",    payload: { id: 2, name: "Mouse",    price: 29.99, qty: 1 } },
  { type: "ADD_ITEM",    payload: { id: 1, name: "Keyboard", price: 89.99, qty: 1 } }, // qty becomes 2
  { type: "APPLY_COUPON", payload: { code: "SAVE10" } },
  { type: "UPDATE_QTY",  payload: { id: 2, qty: 3 } },
  { type: "REMOVE_ITEM", payload: { id: 1 } }
];

const initialState = { items: [], couponCode: null };

const finalState = actions.reduce(cartReducer, initialState);
console.log(finalState);
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

```js
const result = array.reduce((acc, element, index) => {
  return updatedAcc;   // MUST return — becomes acc for the next iteration
}, initialValue);      // always provide this
```

**Common patterns:**

```js
// Sum
arr.reduce((acc, n) => acc + n, 0)

// Count occurrences
arr.reduce((acc, x) => ({ ...acc, [x]: (acc[x] || 0) + 1 }), {})

// Group by property
arr.reduce((acc, item) => {
  (acc[item.key] = acc[item.key] || []).push(item);
  return acc;
}, {})

// Flatten one level
arr.reduce((acc, sub) => acc.concat(sub), [])

// Filter + map in one pass
arr.reduce((acc, item) => {
  if (condition(item)) acc.push(transform(item));
  return acc;
}, [])

// Build an object (index by id)
arr.reduce((acc, item) => ({ ...acc, [item.id]: item }), {})

// Max / min
arr.reduce((acc, n) => (n > acc ? n : acc), arr[0])
```

**Key rules:**
- Always provide `initialValue` — prevents crashes on empty arrays
- Always `return` the accumulator — forgetting `return` is the #1 mistake
- Accumulator can be any type: number, string, object `{}`, array `[]`, Map, Set
- For accumulator objects: mutate-and-return is efficient; spreading every iteration is O(n²)
- `reduce` always processes every element — no early exit; use `find`/`some` for early exit
- Use `reduce` when no other method fits; prefer `map`/`filter` when they do

---

## Connected topics

- **27 — map** — `reduce` can replicate `map`; use `map` when the intent is purely transformation
- **28 — filter** — `reduce` can replicate `filter`; chain `filter + map` for clarity, use `reduce` when combining both in one pass matters
- **34 — Object basics** — `reduce` often produces objects; understanding object literals and property access is essential
- **44 — Memoization** — a common pattern that uses an object (like `reduce`'s accumulator) to cache computed results
