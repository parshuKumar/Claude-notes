# 24 — Array Mutation Methods

## What is this?
Mutation methods are array methods that **modify the original array in place** — they don't return a new array, they change the one you already have. Think of it like editing a physical to-do list on paper: you cross things off, add new items, and rearrange — the list itself is changed. The five core mutation methods are `push`, `pop`, `shift`, `unshift`, and `splice`.

## Why does it matter?
Every time you add a product to a cart, remove a notification, or reorder items in a list, you're using mutation methods. They're the most direct and performant way to change the contents of an array. Knowing *which* ones mutate (and which don't) is critical — accidentally mutating an array you didn't mean to touch is one of the most common sources of bugs in JavaScript.

---

## Syntax

```js
const items = ["a", "b", "c"];

items.push("d");          // add to end       → ["a","b","c","d"]
items.pop();              // remove from end  → ["a","b","c"]
items.unshift("z");       // add to start     → ["z","a","b","c"]
items.shift();            // remove from start→ ["a","b","c"]
items.splice(1, 1);       // remove at index  → ["a","c"]
items.splice(1, 0, "x");  // insert at index  → ["a","x","c"]
items.splice(1, 1, "y");  // replace at index → ["a","y","c"]
```

---

## Method-by-method breakdown

### `push(...items)` — add to the end

```js
const notifications = ["Email from Alice", "PR comment from Bob"];

const newLength = notifications.push("Deployment finished");
// Returns: the new length (3)

console.log(notifications);
// ["Email from Alice", "PR comment from Bob", "Deployment finished"]

// Push multiple at once
notifications.push("Server alert", "New sign-up");
console.log(notifications.length); // 5
```

**Returns:** the new `length` of the array (not the item added).

---

### `pop()` — remove from the end

```js
const browserHistory = ["/home", "/products", "/cart", "/checkout"];

const removedPage = browserHistory.pop();
// Returns: the removed element

console.log(removedPage);      // "/checkout"
console.log(browserHistory);   // ["/home", "/products", "/cart"]

// Pop on an empty array returns undefined — no error
const empty = [];
console.log(empty.pop());      // undefined
```

**Returns:** the removed element (or `undefined` if the array is empty).

---

### `unshift(...items)` — add to the start

```js
const priorityQueue = ["task-B", "task-C"];

const newLength = priorityQueue.unshift("task-A");
// Returns: the new length (3)

console.log(priorityQueue);    // ["task-A", "task-B", "task-C"]

// Add multiple — they're inserted in the order given
priorityQueue.unshift("task-URGENT-1", "task-URGENT-2");
console.log(priorityQueue);
// ["task-URGENT-1", "task-URGENT-2", "task-A", "task-B", "task-C"]
```

**Returns:** the new `length`. ⚠️ `unshift` is slower than `push` on large arrays — all existing elements must be re-indexed.

---

### `shift()` — remove from the start

```js
const ticketQueue = ["Ticket-001", "Ticket-002", "Ticket-003"];

const nextTicket = ticketQueue.shift();
// Returns: the removed element

console.log(nextTicket);     // "Ticket-001"
console.log(ticketQueue);    // ["Ticket-002", "Ticket-003"]
```

**Returns:** the removed element (or `undefined` if empty). ⚠️ `shift` is slower than `pop` on large arrays — all remaining elements must be re-indexed.

---

### `splice(start, deleteCount, ...itemsToInsert)` — the Swiss army knife

`splice` can **remove**, **insert**, or **replace** elements anywhere in the array.

```js
const playlist = ["Track-1", "Track-2", "Track-3", "Track-4", "Track-5"];

// --- Remove ---
const removed = playlist.splice(1, 2);    // start at index 1, remove 2 items
console.log(removed);    // ["Track-2", "Track-3"]  ← returns what was removed
console.log(playlist);   // ["Track-1", "Track-4", "Track-5"]

// --- Insert (deleteCount = 0) ---
playlist.splice(1, 0, "New-Track-A", "New-Track-B");
console.log(playlist);
// ["Track-1", "New-Track-A", "New-Track-B", "Track-4", "Track-5"]

// --- Replace (remove 1, insert 1) ---
playlist.splice(2, 1, "Replaced-Track");
console.log(playlist);
// ["Track-1", "New-Track-A", "Replaced-Track", "Track-4", "Track-5"]

// --- Remove from end (negative index) ---
playlist.splice(-1, 1);   // start 1 from the end, remove 1
console.log(playlist);
// ["Track-1", "New-Track-A", "Replaced-Track", "Track-4"]
```

**Returns:** an array of the removed elements (empty array `[]` if nothing was removed).

---

## How it works — line by line

```js
const playlist = ["Track-1", "Track-2", "Track-3", "Track-4", "Track-5"];
const removed = playlist.splice(1, 2);
```

- `start = 1` → begin at index 1 ("Track-2")
- `deleteCount = 2` → remove 2 elements starting there ("Track-2", "Track-3")
- No items to insert → only deletion
- Returns the array of removed items: `["Track-2", "Track-3"]`
- `playlist` is now `["Track-1", "Track-4", "Track-5"]` — mutated in place

```js
playlist.splice(1, 0, "New-Track-A", "New-Track-B");
```

- `start = 1` → insert before index 1
- `deleteCount = 0` → remove nothing
- Items `"New-Track-A"`, `"New-Track-B"` are inserted at position 1
- Original elements from index 1 onward shift right

---

## Example 1 — basic: using push/pop as a stack

A **stack** is a "last in, first out" data structure — like a pile of plates.

```js
// Browser navigation stack — back button behaviour
const pageStack = [];

// User navigates forward
pageStack.push("/home");
pageStack.push("/shop");
pageStack.push("/product/42");
pageStack.push("/cart");

console.log(pageStack);
// ["/home", "/shop", "/product/42", "/cart"]

// User clicks Back — pop the current page
const current = pageStack.pop();
console.log("Left:", current);    // "/cart"
console.log("Now on:", pageStack[pageStack.length - 1]); // "/product/42"

pageStack.pop();
console.log("Now on:", pageStack[pageStack.length - 1]); // "/shop"
console.log(pageStack);           // ["/home", "/shop"]
```

---

## Example 2 — basic: using shift/unshift as a queue

A **queue** is "first in, first out" — like a line at a checkout.

```js
// Print job queue
const printQueue = [];

// Jobs arrive
printQueue.push("invoice-001.pdf");
printQueue.push("report-Q1.pdf");
printQueue.push("contract-draft.pdf");

console.log(printQueue);
// ["invoice-001.pdf", "report-Q1.pdf", "contract-draft.pdf"]

// Printer processes the next job (first in, first out)
function processNextJob() {
  if (printQueue.length === 0) {
    console.log("Queue empty");
    return;
  }
  const nextJob = printQueue.shift();
  console.log("Printing:", nextJob);
}

processNextJob(); // Printing: invoice-001.pdf
processNextJob(); // Printing: report-Q1.pdf
console.log(printQueue); // ["contract-draft.pdf"]
```

---

## Example 3 — real world: shopping cart management

```js
let cart = [
  { id: 101, name: "Keyboard",  price: 89.99 },
  { id: 102, name: "Mouse",     price: 29.99 },
  { id: 103, name: "Webcam",    price: 69.99 },
  { id: 104, name: "Headset",   price: 49.99 }
];

// Add item to cart
function addToCart(item) {
  cart.push(item);
  console.log(item.name + " added. Cart size: " + cart.length);
}

// Remove item by id
function removeFromCart(itemId) {
  const index = cart.findIndex((item) => item.id === itemId);

  if (index === -1) {
    console.log("Item not found in cart");
    return;
  }

  const removed = cart.splice(index, 1); // remove 1 item at found index
  console.log(removed[0].name + " removed from cart");
}

// Insert an urgent/priority item at the top of the cart
function addPriorityItem(item) {
  cart.unshift(item);
  console.log(item.name + " added as priority item (top of cart)");
}

// Replace an item (e.g. wrong size → correct size)
function replaceItem(itemId, newItem) {
  const index = cart.findIndex((item) => item.id === itemId);

  if (index === -1) {
    console.log("Item not found");
    return;
  }

  cart.splice(index, 1, newItem);        // remove 1, insert newItem
  console.log("Replaced item at position " + (index + 1));
}

addToCart({ id: 105, name: "Monitor Stand", price: 39.99 });
// Keyboard added. Cart size: 5

removeFromCart(102);
// Mouse removed from cart

addPriorityItem({ id: 100, name: "Gift Wrap", price: 4.99 });
// Gift Wrap added as priority item (top of cart)

replaceItem(103, { id: 103, name: "Webcam Pro", price: 99.99 });
// Replaced item at position 3

console.log(cart.map((item) => item.name));
// ["Gift Wrap", "Keyboard", "Webcam Pro", "Headset", "Monitor Stand"]
```

---

## Tricky things you'll encounter in the real world

### 1 — All five methods mutate — they change the original array

```js
const originalList = ["a", "b", "c"];
const ref = originalList;          // same reference — NOT a copy

ref.push("d");
console.log(originalList);         // ["a", "b", "c", "d"]  ← mutated!

// If you need to preserve the original, copy first
const safeCopy = [...originalList];
safeCopy.push("e");
console.log(originalList);         // ["a", "b", "c", "d"]  ← untouched
```

---

### 2 — `push` vs `concat` — mutation vs new array

```js
const cart = ["Keyboard", "Mouse"];

// push — mutates, returns new length
cart.push("Monitor");             // cart is now ["Keyboard", "Mouse", "Monitor"]

// concat — does NOT mutate, returns a NEW array
const newCart = cart.concat("Webcam");
console.log(cart);    // ["Keyboard", "Mouse", "Monitor"]  — unchanged
console.log(newCart); // ["Keyboard", "Mouse", "Monitor", "Webcam"]

// In modern code, spread is preferred over concat:
const updatedCart = [...cart, "Webcam"];
```

---

### 3 — `splice` with negative start index counts from the end

```js
const logs = ["log-1", "log-2", "log-3", "log-4", "log-5"];

// Remove the last 2 logs
logs.splice(-2, 2);
console.log(logs); // ["log-1", "log-2", "log-3"]

// Insert before the last element
logs.splice(-1, 0, "injected-log");
console.log(logs); // ["log-1", "log-2", "injected-log", "log-3"]
```

---

### 4 — `splice` with no deleteCount removes everything from start to end

```js
const steps = ["step-1", "step-2", "step-3", "step-4"];

// Remove from index 2 to end (omit deleteCount)
const removed = steps.splice(2);
console.log(removed); // ["step-3", "step-4"]
console.log(steps);   // ["step-1", "step-2"]
```

If you omit `deleteCount`, `splice` removes ALL elements from `start` to the end.

---

### 5 — `pop` and `shift` on empty arrays return `undefined` silently

```js
const queue = [];

const next = queue.shift();
console.log(next);           // undefined — no error thrown

// Bug-prone:
console.log(next.toUpperCase()); // ❌ TypeError: Cannot read properties of undefined

// Fix — always check before using the return value
if (queue.length > 0) {
  const next = queue.shift();
  console.log(next.toUpperCase());
}
```

---

### 6 — `push` with an array pushes the ARRAY as one element (not spreading it)

```js
const tags = ["js", "node"];
const newTags = ["react", "css"];

// ❌ Pushes the whole array as a single element (nested array)
tags.push(newTags);
console.log(tags); // ["js", "node", ["react", "css"]]

// ✅ Spread to push individual items
tags.push(...newTags);
console.log(tags); // ["js", "node", "react", "css"]
```

---

### 7 — Performance: `push`/`pop` are O(1), `shift`/`unshift` are O(n)

`push` and `pop` operate at the end — no re-indexing needed, always fast.  
`shift` and `unshift` operate at the start — every existing element gets a new index, which takes longer as the array grows.

For very large arrays (thousands+ of items) used as queues, consider a different data structure. For typical UI work (dozens to low hundreds of items), the difference is imperceptible.

---

## Common mistakes

### ❌ Mistake 1 — Using the return value of `push`/`unshift` thinking it's the new item

```js
const items = ["a", "b"];

// WRONG — push returns the new length (3), not the added item
const result = items.push("c");
console.log(result);  // 3  ← not "c"!

// RIGHT — if you need the added item, just use it directly
items.push("c");
const addedItem = items[items.length - 1]; // "c"
```

---

### ❌ Mistake 2 — Forgetting `splice` returns the removed items as an array

```js
const users = ["Alice", "Bob", "Charlie"];

// WRONG mental model — thinking splice returns the remaining array
const result = users.splice(1, 1);
console.log(result);  // ["Bob"]  ← removed items, NOT the remaining array
console.log(users);   // ["Alice", "Charlie"]  ← this is what remains
```

---

### ❌ Mistake 3 — Mutating an array while iterating over it

```js
const orderIds = [101, 102, 103, 104, 105];

// ❌ DANGEROUS — splicing inside a for loop messes up indices
for (let i = 0; i < orderIds.length; i++) {
  if (orderIds[i] % 2 === 0) {
    orderIds.splice(i, 1);   // removes element, array shifts — you skip the next element!
  }
}
console.log(orderIds); // [101, 103, 105] — seems right but worked by accident

// ✅ SAFE — iterate backwards so splice doesn't affect unvisited indices
for (let i = orderIds.length - 1; i >= 0; i--) {
  if (orderIds[i] % 2 === 0) {
    orderIds.splice(i, 1);   // safe — only affects items already visited
  }
}

// ✅ BEST — use filter to create a new array (non-mutating)
const oddIds = orderIds.filter((id) => id % 2 !== 0);
```

---

## Frequently asked questions

**Q: When should I use `push`/`pop` vs `shift`/`unshift`?**  
Use `push`/`pop` (end of array) whenever possible — they're faster. Use `shift`/`unshift` (start of array) when order specifically requires FIFO behaviour (queues) or when you need something at the front.

**Q: What does `splice` return if I don't remove anything?**  
An empty array `[]`. `splice(2, 0, "new")` inserts "new" at index 2, removes nothing, and returns `[]`.

**Q: Is there a non-mutating version of `splice`?**  
Yes — `toSpliced()` (ES2023) returns a new array with the change applied, leaving the original untouched. For older code, combine `slice` and `concat` (covered in topic 25).

**Q: Can I use `splice` to remove ALL elements?**  
Yes: `array.splice(0)` removes everything and returns all elements. But `array.length = 0` or `array.splice(0, array.length)` is clearer for intent.

**Q: What's the difference between `delete array[i]` and `splice`?**  
`delete array[i]` sets the slot to `undefined` (creates a hole) but does NOT change `length`. `splice` actually removes the element and re-indexes everything after it. Always prefer `splice` for removing elements.

---

## Practice exercises

### Exercise 1 — easy

Start with this array:
```js
const taskList = ["Design mockup", "Write tests", "Build feature", "Code review", "Deploy"];
```

Using only mutation methods:
1. Add `"Write documentation"` to the end
2. Add `"Emergency hotfix"` to the start
3. Remove the last task and log what was removed
4. Remove the first task and log what was removed
5. Log the final array and its length

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a **playlist manager** with the following:

```js
const playlist = [
  { id: 1, title: "Bohemian Rhapsody", artist: "Queen" },
  { id: 2, title: "Hotel California",  artist: "Eagles" },
  { id: 3, title: "Stairway to Heaven", artist: "Led Zeppelin" },
  { id: 4, title: "Imagine",            artist: "John Lennon" }
];
```

Write these functions using mutation methods:
- `addTrack(track)` — add to the end
- `removeTrack(id)` — remove by id using `splice` (find the index first with a `for` loop)
- `moveToTop(id)` — move a track to position 0 (remove it, then `unshift` it)
- `insertAfter(id, newTrack)` — insert `newTrack` immediately after the track with matching `id`

Test: remove track 2, move track 4 to top, insert a new track after track 1.

```js
// Write your code here
```

---

### Exercise 3 — hard

Implement a **fixed-size log buffer** — a circular buffer that keeps only the last N log entries.

Write a function `createLogBuffer(maxSize)` that returns an object with:
- `log(message)` — adds a message to the buffer. If adding it would exceed `maxSize`, remove the oldest entry first (`shift`) before adding the new one (`push`). Log `"[BUFFER FULL] Dropping oldest entry"` when this happens.
- `getLogs()` — returns a copy of the current log array
- `clear()` — empties the buffer
- `size()` — returns the current number of entries

Create a buffer of size 3, add 5 log messages, and after each `log()` call print the current buffer contents.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Method | Where | Mutates? | Returns |
|---|---|---|---|
| `push(...items)` | End | ✅ Yes | New `length` |
| `pop()` | End | ✅ Yes | Removed element |
| `unshift(...items)` | Start | ✅ Yes | New `length` |
| `shift()` | Start | ✅ Yes | Removed element |
| `splice(start, del, ...ins)` | Anywhere | ✅ Yes | Array of removed items |

**`splice` quick patterns:**
```js
arr.splice(i, 1)           // remove 1 element at index i
arr.splice(i, 0, x)        // insert x before index i (remove 0)
arr.splice(i, 1, x)        // replace element at index i with x
arr.splice(-n, n)          // remove last n elements
arr.splice(i)              // remove everything from i to end
```

**Stack (LIFO):** `push` to add, `pop` to remove  
**Queue (FIFO):** `push` to add, `shift` to remove

**Key rules:**
- All five methods mutate the original array — copy first if you need to preserve it
- `push`/`unshift` return the new **length**, NOT the added item
- `pop`/`shift` return the **removed item** (or `undefined` if empty)
- `splice` returns an **array** of removed items (empty array if nothing removed)
- Never `splice` inside a forward `for` loop — iterate backwards or use `filter`
- Use `push(...array)` (spread) to push multiple items; `push(array)` nests the whole array

---

## Connected topics

- **25 — Array non-mutation methods** — `slice`, `concat`, `join`, `flat` — operations that return a new array and leave the original untouched
- **23 — Array basics** — indexing, length, and how arrays work in memory
- **28 — filter** — the clean, non-mutating way to remove elements based on a condition
