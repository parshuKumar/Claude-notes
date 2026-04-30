# 25 — Array Non-Mutation Methods

## What is this?
Non-mutation methods are array methods that **return a new value without changing the original array**. Think of it like a photocopier — you feed in the original document, it gives you a modified copy, and the original stays exactly as it was. The four core non-mutation methods are `slice`, `concat`, `join`, and `flat`.

## Why does it matter?
In real codebases you often need to work with array data without corrupting the source — especially when the same array is used in multiple places, passed to components, or stored in state. Non-mutation methods let you derive new arrays freely without defensive copying everywhere. Understanding which methods mutate and which don't is one of the most practical distinctions in JavaScript.

---

## Syntax

```js
const original = [1, 2, 3, 4, 5];

const sliced  = original.slice(1, 3);    // [2, 3]           — original untouched
const joined  = original.concat([6, 7]); // [1,2,3,4,5,6,7]  — original untouched
const str     = original.join(" - ");    // "1 - 2 - 3 - 4 - 5"
const nested  = [[1, 2], [3, 4]];
const flat    = nested.flat();           // [1, 2, 3, 4]      — original untouched

console.log(original); // [1, 2, 3, 4, 5]  — unchanged by all of the above
```

---

## Method-by-method breakdown

### `slice(start, end)` — extract a portion of an array

Returns a **shallow copy** of a portion of the array from `start` up to (but not including) `end`.

```js
const menuItems = ["Home", "Shop", "About", "Blog", "Contact", "FAQ"];

// Basic slice
console.log(menuItems.slice(1, 4));   // ["Shop", "About", "Blog"]
                                       // indices 1, 2, 3 — NOT 4

// Omit end — go to the end of the array
console.log(menuItems.slice(2));      // ["About", "Blog", "Contact", "FAQ"]

// Negative indices — count from the end
console.log(menuItems.slice(-2));     // ["Contact", "FAQ"]  — last 2
console.log(menuItems.slice(-3, -1)); // ["Blog", "Contact"] — 3rd from end up to (not including) last

// Copy the entire array (common pattern)
const copy = menuItems.slice();       // all elements, new array
console.log(copy === menuItems);      // false — different array object

// Original is untouched
console.log(menuItems.length);        // 6
```

**Returns:** a new array. Original is never modified.

---

### `concat(...arrays)` — merge arrays together

Returns a **new array** combining the original with any values or arrays passed in.

```js
const featuredProducts = ["Keyboard", "Mouse"];
const newArrivals      = ["Webcam", "Headset"];
const clearanceItems   = ["Old Monitor", "USB Hub"];

// Combine two arrays
const shopAll = featuredProducts.concat(newArrivals);
console.log(shopAll);
// ["Keyboard", "Mouse", "Webcam", "Headset"]

// Combine multiple arrays at once
const fullCatalogue = featuredProducts.concat(newArrivals, clearanceItems);
console.log(fullCatalogue);
// ["Keyboard", "Mouse", "Webcam", "Headset", "Old Monitor", "USB Hub"]

// Add individual values alongside arrays
const withExtras = featuredProducts.concat(newArrivals, "Monitor Stand", "Cable Tidy");
console.log(withExtras);
// ["Keyboard", "Mouse", "Webcam", "Headset", "Monitor Stand", "Cable Tidy"]

// All originals untouched
console.log(featuredProducts); // ["Keyboard", "Mouse"]
console.log(newArrivals);      // ["Webcam", "Headset"]
```

**Modern alternative — spread operator (preferred):**
```js
const fullCatalogue = [...featuredProducts, ...newArrivals, ...clearanceItems];
```
Both produce the same result. Spread is more common in modern code.

---

### `join(separator)` — convert an array to a string

Returns a **string** formed by concatenating all elements with the given separator between them.

```js
const breadcrumbs  = ["Home", "Electronics", "Laptops", "MacBook Pro"];
const csvRow       = ["Alice", "28", "Engineer", "London"];
const urlPathParts = ["api", "v2", "users", "profile"];
const tags         = ["javascript", "nodejs", "react"];

console.log(breadcrumbs.join(" > "));   // "Home > Electronics > Laptops > MacBook Pro"
console.log(csvRow.join(","));          // "Alice,28,Engineer,London"
console.log(urlPathParts.join("/"));    // "api/v2/users/profile"
console.log(tags.join(", "));          // "javascript, nodejs, react"

// Default separator is a comma
console.log(breadcrumbs.join());        // "Home,Electronics,Laptops,MacBook Pro"

// Empty string separator — concatenate with nothing between
console.log(["H", "e", "l", "l", "o"].join("")); // "Hello"

// Numbers are converted to strings automatically
console.log([1, 2, 3].join(" + "));    // "1 + 2 + 3"

// null/undefined become empty strings
console.log([1, null, 3, undefined, 5].join("-")); // "1--3--5"
```

**Returns:** a string. The array is not modified.

---

### `flat(depth)` — flatten nested arrays

Returns a **new array** with sub-arrays flattened into it up to the specified `depth`.

```js
// One level deep (default depth = 1)
const weeklyTasks = [
  ["Standup", "Code review"],
  ["Feature work", "Testing"],
  ["Deploy", "Retrospective"]
];

console.log(weeklyTasks.flat());
// ["Standup", "Code review", "Feature work", "Testing", "Deploy", "Retrospective"]

// Deeper nesting requires higher depth
const deeplyNested = [1, [2, [3, [4]]]];

console.log(deeplyNested.flat());    // [1, 2, [3, [4]]]      — only 1 level
console.log(deeplyNested.flat(2));   // [1, 2, 3, [4]]        — 2 levels
console.log(deeplyNested.flat(3));   // [1, 2, 3, 4]          — 3 levels
console.log(deeplyNested.flat(Infinity)); // [1, 2, 3, 4]     — all levels

// flat removes holes in sparse arrays
const sparse = [1, , 3, , 5];
console.log(sparse.flat()); // [1, 3, 5]  — holes removed ✅
```

**Returns:** a new array. Original is not modified.

---

### `flatMap(fn)` — map then flat (depth 1)

A very common combo: transforms each element AND flattens one level. Equivalent to `.map().flat()` but more efficient.

```js
const sentences = ["Hello world", "JavaScript is great", "Keep learning"];

// Split each sentence into words, then flatten into one word list
const words = sentences.flatMap((sentence) => sentence.split(" "));
console.log(words);
// ["Hello", "world", "JavaScript", "is", "great", "Keep", "learning"]

// Compare with .map() alone:
const notFlat = sentences.map((sentence) => sentence.split(" "));
console.log(notFlat);
// [["Hello","world"], ["JavaScript","is","great"], ["Keep","learning"]]

// Real-world use: expand each order into individual line items
const orders = [
  { orderId: "A1", items: ["Keyboard", "Mouse"] },
  { orderId: "A2", items: ["Monitor"] },
  { orderId: "A3", items: ["Webcam", "Headset", "Hub"] }
];

const allItems = orders.flatMap((order) => order.items);
console.log(allItems);
// ["Keyboard", "Mouse", "Monitor", "Webcam", "Headset", "Hub"]
```

---

## Example 1 — real world: pagination with `slice`

```js
function paginate(items, pageNumber, itemsPerPage) {
  // Calculate the start and end indices for this page
  const start = (pageNumber - 1) * itemsPerPage;
  const end   = start + itemsPerPage;

  const pageItems  = items.slice(start, end);    // never mutates items
  const totalPages = Math.ceil(items.length / itemsPerPage);

  return {
    items:       pageItems,
    currentPage: pageNumber,
    totalPages,
    hasNext:     pageNumber < totalPages,
    hasPrev:     pageNumber > 1
  };
}

const allProducts = [
  "Keyboard", "Mouse", "Monitor", "Webcam", "Headset",
  "USB Hub", "Desk Mat", "Cable Tidy", "Monitor Arm", "LED Strip"
];

const page1 = paginate(allProducts, 1, 3);
console.log(page1.items);      // ["Keyboard", "Mouse", "Monitor"]
console.log(page1.totalPages); // 4
console.log(page1.hasNext);    // true

const page3 = paginate(allProducts, 3, 3);
console.log(page3.items);      // ["USB Hub", "Desk Mat", "Cable Tidy"]

// allProducts is completely untouched
console.log(allProducts.length); // 10
```

---

## Example 2 — real world: building URLs and CSV with `join`

```js
// Build a URL from parts
function buildApiUrl(baseUrl, ...pathSegments) {
  const cleanSegments = pathSegments.map((s) => s.replace(/^\/|\/$/g, "")); // strip slashes
  return baseUrl.replace(/\/$/, "") + "/" + cleanSegments.join("/");
}

console.log(buildApiUrl("https://api.example.com", "users", "42", "orders"));
// "https://api.example.com/users/42/orders"

// Build a CSV row
function toCSVRow(fields) {
  return fields
    .map((field) => {
      const str = String(field);
      return str.includes(",") ? `"${str}"` : str;   // wrap in quotes if contains comma
    })
    .join(",");
}

const headers = ["Name", "Age", "City", "Job Title"];
const row1    = ["Alice", 28, "London", "Software Engineer"];
const row2    = ["Bob",   34, "New York", "Product Manager, Senior"];

console.log(toCSVRow(headers)); // "Name,Age,City,Job Title"
console.log(toCSVRow(row1));    // "Alice,28,London,Software Engineer"
console.log(toCSVRow(row2));    // "Bob,34,New York,"Product Manager, Senior""
```

---

## Example 3 — real world: merging data sources with `concat`

```js
async function getFullNotificationFeed(userId) {
  // In real code these would be await fetch(...)
  const systemAlerts = [
    { type: "system", message: "Scheduled maintenance tonight", priority: "high" }
  ];

  const userMessages = [
    { type: "message", message: "Alice sent you a file", priority: "normal" },
    { type: "message", message: "Bob mentioned you in a comment", priority: "normal" }
  ];

  const activityUpdates = [
    { type: "activity", message: "Your PR was approved", priority: "high" },
    { type: "activity", message: "Deployment completed", priority: "normal" }
  ];

  // Merge all sources — none of the originals are mutated
  const allNotifications = systemAlerts.concat(userMessages, activityUpdates);

  // Sort high priority first (sort is mutating, but on the NEW merged array — that's fine)
  allNotifications.sort((a, b) => (a.priority === "high" ? -1 : 1));

  return allNotifications;
}
```

---

## Tricky things you'll encounter in the real world

### 1 — `slice` is shallow — nested objects are still shared references

```js
const users = [
  { id: 1, name: "Alice", settings: { theme: "dark" } },
  { id: 2, name: "Bob",   settings: { theme: "light" } }
];

const copy = users.slice();         // new array, but same object references inside

copy[0].settings.theme = "light";   // ⚠️ mutates the object inside the original!
console.log(users[0].settings.theme); // "light" — original changed!

// Fix: deep clone objects if you need true independence
const deepCopy = users.map((user) => ({
  ...user,
  settings: { ...user.settings }   // spread nested object too
}));
```

---

### 2 — `concat` does NOT flatten nested arrays more than one level

```js
const a = [1, [2, 3]];
const b = [[4, 5], 6];

console.log(a.concat(b));
// [1, [2, 3], [4, 5], 6]  — arrays inside are kept intact

// concat flattens only the TOP level of arrays you pass in:
console.log([1].concat([2, [3]])); // [1, 2, [3]]  — [3] is not flattened
```

---

### 3 — `join` converts everything to a string — including objects (badly)

```js
const items = [{ name: "Mouse" }, { name: "Keyboard" }];
console.log(items.join(", "));
// "[object Object], [object Object]"  ❌ — useless

// Fix: map to strings first
console.log(items.map((item) => item.name).join(", "));
// "Mouse, Keyboard"  ✅
```

---

### 4 — `flat` only goes one level deep by default

```js
const data = [[1, 2], [3, [4, 5]]];

console.log(data.flat());    // [1, 2, 3, [4, 5]]  — inner [4,5] not flattened!
console.log(data.flat(2));   // [1, 2, 3, 4, 5]    — correct

// When unsure how deep: use Infinity
console.log(data.flat(Infinity)); // [1, 2, 3, 4, 5]
```

---

### 5 — Chaining non-mutation methods

Because each method returns a new array (or value), you can chain them:

```js
const rawData = [
  "  Alice  ",
  "",
  "  Bob  ",
  null,
  "  Charlie  ",
  undefined,
  "  Diana  "
];

const cleanNames = rawData
  .filter(Boolean)                   // remove falsy (null, undefined, "")
  .map((name) => name.trim())        // trim whitespace
  .slice(0, 3)                       // take first 3
  .join(", ");                       // format as string

console.log(cleanNames); // "Alice, Bob, Charlie"
```

Each step works on the output of the previous one — the original `rawData` is never changed.

---

### 6 — `slice()` with no arguments copies the whole array

```js
const original = [1, 2, 3, 4, 5];
const copy = original.slice();      // common shallow copy pattern

copy.push(6);
console.log(original); // [1, 2, 3, 4, 5]  — untouched
console.log(copy);     // [1, 2, 3, 4, 5, 6]

// In modern code, spread is more common for the same effect:
const copy2 = [...original];
```

---

## Common mistakes

### ❌ Mistake 1 — Expecting `slice` to mutate the original

```js
const items = ["a", "b", "c", "d"];

// WRONG — thinking slice removes elements from items
items.slice(1, 2);
console.log(items); // ["a", "b", "c", "d"]  — nothing changed!

// RIGHT — capture the return value
const portion = items.slice(1, 2);
console.log(portion); // ["b"]
console.log(items);   // ["a", "b", "c", "d"]  — still intact
```

---

### ❌ Mistake 2 — Confusing `slice` and `splice`

```js
const arr = [10, 20, 30, 40, 50];

// slice — non-mutating, returns a portion
arr.slice(1, 3);    // returns [20, 30], arr unchanged

// splice — MUTATING, modifies the original
arr.splice(1, 2);   // removes [20, 30] from arr, arr is now [10, 40, 50]

// Memory trick: splIce → Incision (it cuts into the array)
//               slIce  → a slice of the array (read-only copy)
```

---

### ❌ Mistake 3 — Using `concat` when you mean `push` (and vice versa)

```js
const cart = ["Keyboard"];

// WRONG — using concat but not capturing the result (silent no-op)
cart.concat(["Mouse"]);          // new array created, immediately discarded
console.log(cart);               // ["Keyboard"]  — unchanged!

// RIGHT — capture it
const updatedCart = cart.concat(["Mouse"]);

// Or just use push if you want to mutate
cart.push("Mouse");
```

---

## Frequently asked questions

**Q: What does "shallow copy" mean?**  
A shallow copy creates a new array, but the elements inside it are still references to the same objects as the original. Changing a primitive value in the copy doesn't affect the original. But mutating a nested object through the copy WILL affect the original, because both arrays point to the same object in memory.

**Q: When should I use `concat` vs spread (`[...a, ...b]`)?**  
They do the same thing. Spread is shorter and more readable in modern code. `concat` is useful when combining many arrays dynamically (e.g. `[].concat(...arrayOfArrays)`).

**Q: Can I chain `slice` and `join`?**  
Yes — `array.slice(0, 3).join(", ")` is perfectly valid. Each method returns a value the next can operate on.

**Q: What's `flatMap` and when should I use it?**  
`flatMap` runs a `.map()` then a `.flat(1)` in one step. Use it whenever your map callback returns an array and you want the results flattened into a single array — for example splitting sentences into words.

**Q: Does `flat` remove `undefined`/`null` values?**  
No — it only flattens nested arrays. But it does remove holes in sparse arrays.

**Q: What is the non-mutating alternative to `splice`?**  
Combine `slice` and spread: `[...arr.slice(0, i), newItem, ...arr.slice(i + 1)]`. Or use the newer `toSpliced()` method (ES2023).

---

## Practice exercises

### Exercise 1 — easy

Given this array:
```js
const sentence = ["The", "quick", "brown", "fox", "jumps", "over", "the", "lazy", "dog"];
```

1. Use `slice` to extract just `["brown", "fox", "jumps"]`
2. Use `slice` to get the last 3 words
3. Use `join` to turn the full array into the sentence `"The quick brown fox jumps over the lazy dog"`
4. Use `join` with `" | "` to format just the first 4 words

All operations should leave the original `sentence` array unchanged. Log `sentence` at the end to confirm.

```js
// Write your code here
```

---

### Exercise 2 — medium

You manage a blog platform. You have:
```js
const publishedPosts = [
  { id: 1, title: "Intro to JS",         tags: ["js", "beginner"] },
  { id: 2, title: "Understanding Scope", tags: ["js", "intermediate"] },
  { id: 3, title: "Node.js Basics",      tags: ["node", "beginner"] }
];

const draftPosts = [
  { id: 4, title: "Closures Deep Dive",  tags: ["js", "advanced"] },
  { id: 5, title: "REST API Design",     tags: ["node", "api"] }
];
```

Without mutating either original array:
1. Combine both arrays into `allPosts` using `concat`
2. Get only the first 3 posts from `allPosts` using `slice`
3. Use `flatMap` to get a flat list of ALL unique tags across all posts (use `flat` or `flatMap`, then deduplicate using `[...new Set(tags)]`)
4. Use `join` to produce a comma-separated string of all post titles

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **non-mutating array utility library** — a plain object with methods that each return a new array (never modify the input):

```js
const ArrayUtils = {
  // insertAt(arr, index, item) — returns new array with item inserted at index
  // removeAt(arr, index) — returns new array with element at index removed
  // replaceAt(arr, index, newItem) — returns new array with element at index replaced
  // move(arr, fromIndex, toIndex) — returns new array with element moved from one index to another
  // chunk(arr, size) — splits arr into sub-arrays of given size
  //   e.g. chunk([1,2,3,4,5], 2) → [[1,2],[3,4],[5]]
};
```

Implement all five methods using only `slice`, `concat`, spread, and loops. No mutation methods allowed.

Test each one, and confirm the original array is always untouched.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Method | What it does | Returns | Mutates? |
|---|---|---|---|
| `slice(start, end)` | Extract portion of array | New array | ❌ No |
| `concat(...vals)` | Merge arrays/values | New array | ❌ No |
| `join(sep)` | Array → string | String | ❌ No |
| `flat(depth)` | Flatten nested arrays | New array | ❌ No |
| `flatMap(fn)` | Map + flat(1) in one step | New array | ❌ No |

**`slice` patterns:**
```js
arr.slice(1, 4)     // indices 1, 2, 3 (NOT 4)
arr.slice(2)        // from index 2 to end
arr.slice(-3)       // last 3 elements
arr.slice(-3, -1)   // 3rd from end up to (not including) last
arr.slice()         // full shallow copy
```

**`join` patterns:**
```js
arr.join()          // "a,b,c"  (default comma, no space)
arr.join(", ")      // "a, b, c"
arr.join("/")       // "a/b/c"
arr.join("")        // "abc"
```

**Key rules:**
- None of these methods change the original — always capture the return value
- `slice` vs `splice`: slice = read-only copy, splice = cuts into the original
- Shallow copy: `arr.slice()` or `[...arr]`
- Chaining works: `arr.filter(...).slice(0,5).join(", ")`
- `flat(Infinity)` flattens no matter how deep
- `join` on array of objects gives `[object Object]` — map to strings first

---

## Connected topics

- **24 — Array mutation methods** — the counterpart: `push`, `pop`, `shift`, `unshift`, `splice`
- **27 — map** — transforms every element, returns a new array (non-mutating)
- **28 — filter** — returns a new array of matching elements (non-mutating)
- **33 — Array destructuring** — another clean, non-mutating way to extract values from arrays
