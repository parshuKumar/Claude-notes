# 41 — Pure Functions vs Side Effects

---

## What is this?

A **pure function** is a function that:
1. Given the same inputs, **always returns the same output**
2. Has **no side effects** — it doesn't change anything outside itself

A **side effect** is any change a function makes to the world outside its own scope — modifying a variable, mutating an object, writing to the DOM, making a network request, logging to the console.

```js
// Pure — same input always gives same output, nothing outside changes
function add(a, b) {
  return a + b;
}
add(2, 3); // always 5, forever

// Impure — depends on/changes external state
let total = 0;
function addToTotal(n) {
  total += n;  // side effect: modifies external variable
  return total;
}
addToTotal(5); // 5
addToTotal(5); // 10 — same input, different output!
```

Think of a pure function like a vending machine — put in coins, get the same snack every time. An impure function is like asking a person — the answer depends on their mood (external state).

---

## Why does it matter?

Pure functions are:
- **Predictable** — easy to reason about
- **Testable** — no setup/teardown needed, just input → output
- **Composable** — can be combined reliably
- **Memoizable** — safe to cache results (topic 44)
- **Parallelizable** — no shared state to conflict

Understanding the boundary between pure and impure is the first step toward writing code that doesn't have mysterious bugs caused by shared mutable state.

---

## What counts as a side effect?

```js
// Modifying an external variable
let count = 0;
function increment() { count++; } // side effect

// Mutating a parameter
function addItem(cart, item) {
  cart.push(item); // side effect — mutates the input array
  return cart;
}

// Modifying the DOM
function showError(msg) {
  document.getElementById('error').textContent = msg; // side effect
}

// Making a network request
function fetchUser(id) {
  return fetch(`/api/users/${id}`); // side effect
}

// Writing to localStorage / sessionStorage
function saveTheme(theme) {
  localStorage.setItem('theme', theme); // side effect
}

// Logging
function calculate(a, b) {
  console.log('calculating...'); // side effect (even this!)
  return a + b;
}

// Reading from external mutable state
let discount = 0.1;
function getPrice(price) {
  return price * (1 - discount); // impure — depends on external 'discount'
}
```

---

## Syntax — what makes a function pure

```js
// ✅ Pure: all data comes from parameters, return value only output
function pureFunction(input1, input2) {
  // only uses input1 and input2
  // only creates local variables
  // returns a value
  // no mutations, no I/O, no external reads
  return someComputation(input1, input2);
}
```

---

## How it works — line by line

```js
// IMPURE version
let items = ['apple', 'banana'];

function addFruit(fruit) {
  items.push(fruit);      // ← mutates the EXTERNAL 'items' array
  return items;           // ← returns the same mutated array
}

addFruit('mango');
console.log(items); // ['apple', 'banana', 'mango'] — external state changed

// PURE version
function addFruitPure(currentItems, fruit) {
  return [...currentItems, fruit]; // ← creates and returns a NEW array
                                   // ← currentItems is untouched
}

const items2 = ['apple', 'banana'];
const result = addFruitPure(items2, 'mango');
console.log(items2); // ['apple', 'banana']          — unchanged
console.log(result); // ['apple', 'banana', 'mango'] — new array
```

---

## Example 1 — Identifying pure vs impure

```js
// ✅ Pure
function double(n)        { return n * 2; }
function greet(name)      { return `Hello, ${name}`; }
function sum(arr)         { return arr.reduce((a, b) => a + b, 0); }
function clamp(n, min, max) { return Math.min(Math.max(n, min), max); }
function toUpperCase(str) { return str.toUpperCase(); }

// ❌ Impure
let logHistory = [];
function logAction(action) {
  logHistory.push(action);          // mutates external array — side effect
  return logHistory;
}

let userId = 1;
function nextUserId() {
  return ++userId;                  // reads AND modifies external variable
}

function getTax(amount) {
  const rate = config.taxRate;      // reads external mutable state
  return amount * rate;
}

function saveUser(user) {
  localStorage.setItem('user', JSON.stringify(user)); // I/O side effect
}
```

---

## Example 2 — Transforming impure to pure

### Before: mutating the array parameter

```js
// ❌ Impure — mutates the input array
function sortStudents(students) {
  students.sort((a, b) => a.name.localeCompare(b.name)); // sorts IN PLACE
  return students;
}

const myClass = [
  { name: 'Zara', grade: 90 },
  { name: 'Alice', grade: 85 },
  { name: 'Mike', grade: 92 },
];

sortStudents(myClass);
console.log(myClass); // already sorted — original mutated!

// ✅ Pure — sort a copy
function sortStudentsPure(students) {
  return [...students].sort((a, b) => a.name.localeCompare(b.name));
}

const result = sortStudentsPure(myClass);
console.log(myClass);  // original order preserved
console.log(result);   // sorted copy
```

---

### Before: reading external state

```js
let taxRate = 0.18;

// ❌ Impure — depends on external variable
function calcPrice(amount) {
  return amount + amount * taxRate;
}

// ✅ Pure — tax rate is a parameter
function calcPricePure(amount, rate) {
  return amount + amount * rate;
}

calcPricePure(1000, 0.18); // 1180 — always predictable
calcPricePure(1000, 0.28); // 1280 — easy to test different rates
```

---

### Before: mutating an object parameter

```js
// ❌ Impure — modifies the passed-in cart
function applyDiscount(cart, percent) {
  cart.items.forEach(item => {
    item.price = item.price * (1 - percent / 100); // mutates each item
  });
  return cart;
}

// ✅ Pure — builds new objects at every level
function applyDiscountPure(cart, percent) {
  return {
    ...cart,
    items: cart.items.map(item => ({
      ...item,
      price: item.price * (1 - percent / 100),
    })),
  };
}
```

---

## Example 3 — Pure functions are easy to test

```js
// ✅ Pure function
function formatCurrency(amount, currency = 'INR') {
  return `${currency} ${amount.toLocaleString('en-IN', { minimumFractionDigits: 2 })}`;
}

// Testing is trivial — no mocks, no setup, no state to reset
console.log(formatCurrency(75000));        // 'INR 75,000.00'
console.log(formatCurrency(1500.5, 'USD')); // 'USD 1,500.50'
console.log(formatCurrency(0));            // 'INR 0.00'

// ❌ Impure function — testing requires setup
let discount = 0;
function getPriceWithDiscount(price) {
  return price * (1 - discount);
}
// To test this, you must set 'discount' before each test and reset it after
// Tests can interfere with each other — a nightmare in large codebases
```

---

## Example 4 — Separating pure logic from side effects

Real programs NEED side effects — fetching data, rendering UI, writing files. The goal isn't to eliminate side effects but to **isolate** them from your pure logic.

```js
// Pure — business logic (testable, no I/O)
function calculateOrderTotal(items, discountPercent, taxRate) {
  const subtotal = items.reduce((sum, { price, qty }) => sum + price * qty, 0);
  const discount = subtotal * (discountPercent / 100);
  const tax      = (subtotal - discount) * taxRate;
  return subtotal - discount + tax;
}

// Pure — data transformation (testable)
function formatOrder(order, total) {
  return {
    id:        order.id,
    customer:  order.customerName,
    total:     total,
    timestamp: new Date().toISOString(), // ← subtle: Date() makes this impure!
    // Better: pass timestamp as a parameter to keep it pure
  };
}

// Impure — side effects contained here (fetch, DOM, localStorage)
async function submitOrder(order) {
  // Pure calculation isolated — easy to test independently
  const total     = calculateOrderTotal(order.items, order.discount, 0.18);
  const formatted = formatOrder(order, total);

  // Side effect: network request (unavoidable, but isolated)
  const response = await fetch('/api/orders', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(formatted),
  });

  // Side effect: DOM update (unavoidable, but isolated)
  if (response.ok) {
    document.getElementById('status').textContent = 'Order placed!';
  }
}
```

**Pattern**: Push side effects to the edges of your system. Keep the core logic pure.

---

## Example 5 — Function composition with pure functions

Pure functions compose cleanly because their outputs are predictable:

```js
// All pure
const trim       = str => str.trim();
const toLowerCase = str => str.toLowerCase();
const removeSpaces = str => str.replace(/\s+/g, '-');
const addSlash   = str => `/${str}`;

// Compose — output of one becomes input of the next
function createSlug(title) {
  return addSlash(removeSpaces(toLowerCase(trim(title))));
}

createSlug('  Hello World  '); // '/hello-world'
createSlug('  JS Tips & Tricks  '); // '/js-tips-&-tricks'

// Or with reduce for cleaner composition
function compose(...fns) {
  return input => fns.reduce((acc, fn) => fn(acc), input);
}

const toSlug = compose(trim, toLowerCase, removeSpaces, addSlash);
toSlug('  Advanced JavaScript  '); // '/advanced-javascript'
```

Impure functions can't be safely composed — their hidden state dependencies make the combined behaviour unpredictable.

---

## Example 6 — When side effects are necessary and how to handle them

Side effects are unavoidable in real programs. The key is to:
- **Make them explicit** — obvious from the function name and signature
- **Isolate them** — don't mix pure logic with side effects in the same function
- **Make them predictable** — same inputs should reliably produce the same effects

```js
// ❌ Hidden side effect — function name doesn't hint at mutation
function getFilteredUsers(users) {
  activeUsers = users.filter(u => u.active); // surprise! modifies outer variable
  return activeUsers;
}

// ✅ Explicit side effect — name signals it's doing I/O
async function fetchAndStoreUser(userId) {
  const user = await fetch(`/api/users/${userId}`).then(r => r.json());
  localStorage.setItem(`user_${userId}`, JSON.stringify(user));
  return user;
}

// ✅ Separate pure logic from I/O
function filterActiveUsers(users) {         // pure
  return users.filter(u => u.active);
}

function renderUserList(users) {            // impure — DOM side effect
  const list = document.getElementById('user-list');
  list.innerHTML = users.map(u => `<li>${u.name}</li>`).join('');
}

// Use them together — pure transformation first, then side effect
const activeUsers = filterActiveUsers(allUsers); // pure — testable
renderUserList(activeUsers);                      // side effect — at the edge
```

---

## Tricky things you'll encounter in the real world

### 1. `Math.random()` and `Date.now()` make functions impure

```js
// ❌ Impure — different output every call, even with same input
function generateId(prefix) {
  return `${prefix}_${Math.random().toString(36).slice(2)}`;
}

// ❌ Impure — depends on current time
function isExpired(expiryDate) {
  return new Date() > expiryDate; // uses current time — external state
}

// ✅ Inject the random/time value as a parameter to make it pure
function generateId(prefix, randomValue) {
  return `${prefix}_${randomValue}`;
}

function isExpired(expiryDate, now) {
  return now > expiryDate;
}

// Caller handles the impure part:
generateId('user', Math.random().toString(36).slice(2));
isExpired(someDate, new Date());
```

---

### 2. Methods on built-in types — some are pure, some mutate

```js
// Array methods — PURE (return new array)
arr.map()     // ✅ pure
arr.filter()  // ✅ pure
arr.slice()   // ✅ pure
arr.concat()  // ✅ pure
arr.reduce()  // ✅ pure

// Array methods — IMPURE (mutate in place)
arr.push()    // ❌ mutates
arr.pop()     // ❌ mutates
arr.sort()    // ❌ mutates
arr.reverse() // ❌ mutates
arr.splice()  // ❌ mutates

// String methods — always pure (strings are immutable in JS)
str.toUpperCase() // ✅ always returns new string
str.trim()        // ✅ always returns new string
str.replace()     // ✅ always returns new string
```

---

### 3. Closures can make functions impure — be aware

```js
let multiplier = 3;

// ❌ Impure — closes over mutable external variable
const scale = n => n * multiplier;
scale(5); // 15 now, but if multiplier changes later...
multiplier = 10;
scale(5); // 50 — same input, different output!

// ✅ Pure — multiplier is a parameter
const scalePure = (n, factor) => n * factor;
scalePure(5, 3);  // always 15
scalePure(5, 10); // always 50
```

---

### 4. Object/array parameters — passing by reference means mutation is a side effect

```js
// ❌ Subtle side effect — adds property to the passed-in object
function addTimestamp(order) {
  order.createdAt = new Date().toISOString(); // mutates the parameter!
  return order;
}

// ✅ Pure — creates new object
function addTimestamp(order) {
  return { ...order, createdAt: new Date().toISOString() };
  // Still impure due to Date() — but at least no mutation
}

// ✅ Fully pure (inject time)
function addTimestamp(order, timestamp) {
  return { ...order, createdAt: timestamp };
}
```

---

## Common mistakes

### Mistake 1 — Mutating array/object parameters without realising it

```js
// ❌ push, sort, splice, delete all mutate the original
function getTopThree(scores) {
  return scores.sort((a, b) => b - a).slice(0, 3); // sort mutates scores!
}

const allScores = [88, 55, 92, 71, 100, 63];
getTopThree(allScores);
console.log(allScores); // [100, 92, 88, 71, 63, 55] — original sorted!

// ✅ Sort a copy
function getTopThree(scores) {
  return [...scores].sort((a, b) => b - a).slice(0, 3);
}
```

---

### Mistake 2 — Thinking a function is pure because it returns a value

```js
// ❌ Returns a value BUT has a side effect too — NOT pure
function logAndDouble(n) {
  console.log(`Doubling ${n}`); // side effect
  return n * 2;
}

// A function must have NO side effects AND be deterministic to be pure
```

---

### Mistake 3 — Calling `new Date()` or `Math.random()` inside "pure" functions

```js
// ❌ Looks pure but isn't — output changes every second
function createUser(name) {
  return {
    name,
    createdAt: new Date().toISOString(), // changes every call!
  };
}

// ✅ Inject the timestamp
function createUser(name, timestamp) {
  return { name, createdAt: timestamp };
}
createUser('Alice', new Date().toISOString()); // impurity is at the call site
```

---

## Frequently asked questions

**Q: Does every function need to be pure?**
No — side effects are necessary. The goal is to keep as much of your logic in pure functions as possible, and push side effects to specific, clearly-labelled places. A good ratio is: pure functions for calculations and transformations, impure functions for I/O and state persistence.

**Q: Is `console.log` a side effect?**
Yes — it writes to an output stream (the console), which is external to the function. In practice, `console.log` in application code is tolerated, but in truly pure utility functions it's avoided.

**Q: Are arrow functions automatically pure?**
No — arrow functions are just a shorter syntax. They can be pure or impure depending on what they do.

**Q: What's referential transparency?**
A function is "referentially transparent" if you can replace any call to it with its return value without changing the program's behavior. Pure functions are referentially transparent. Example: `add(2, 3)` can always be replaced with `5` anywhere in your code.

**Q: Do pure functions exist in frameworks like React?**
Yes — React functional components and hooks are designed around this concept. Components that take props and return JSX are essentially pure functions. `useState` and `useEffect` are the explicit side-effect hooks.

**Q: What's a "pure" component in React?**
A component where the same props always produce the same rendered output, with no side effects in the render itself. `React.memo` and `PureComponent` use this to skip unnecessary re-renders.

---

## Practice exercises

### Exercise 1 — Easy: Audit these functions

Below are 6 functions. For each one, state whether it is **pure or impure**, and explain **why** (what makes it impure if it is). Then **rewrite any impure ones** to be pure.

```js
// Function A
function multiply(a, b) {
  return a * b;
}

// Function B
let basePrice = 100;
function getPrice(multiplier) {
  return basePrice * multiplier;
}

// Function C
function reverseString(str) {
  return str.split('').reverse().join('');
}

// Function D
function addTag(tagList, newTag) {
  tagList.push(newTag);
  return tagList;
}

// Function E
function getFullName(user) {
  return `${user.firstName} ${user.lastName}`;
}

// Function F
function updateScore(player) {
  player.score += 10;
  return player;
}
```

```js
// Write your analysis and rewrites here
```

---

### Exercise 2 — Medium: Refactor to pure

This shopping cart module mixes pure logic with side effects and mutations everywhere. Refactor it so that:
- All calculation logic is in pure functions
- Side effects (console output) are isolated and explicit
- No input parameters are mutated

```js
// ❌ Original messy code
let cart = [];
let totalAmount = 0;
const TAX_RATE = 0.18;

function addItem(name, price, qty) {
  cart.push({ name, price, qty });
  totalAmount += price * qty;
  console.log(`Added ${name} to cart. Running total: ₹${totalAmount}`);
}

function removeItem(name) {
  const index = cart.findIndex(item => item.name === name);
  if (index !== -1) {
    totalAmount -= cart[index].price * cart[index].qty;
    cart.splice(index, 1);
    console.log(`Removed ${name}. Running total: ₹${totalAmount}`);
  }
}

function applyTax() {
  totalAmount = totalAmount * (1 + TAX_RATE);
  console.log(`Total after tax: ₹${totalAmount.toFixed(2)}`);
}
```

Write the refactored version:
```js
// ✅ Write your pure refactored version here
```

---

### Exercise 3 — Hard: Pipeline architecture

Build a data processing pipeline that strictly separates pure transformations from side effects.

You receive raw order data and need to:
1. Validate each order (pure)
2. Calculate totals (pure)  
3. Apply tier discounts: `premium` → 15%, `standard` → 5%, `free` → 0% (pure)
4. Format for display (pure)
5. "Save" the result (simulate with console.log — the only impure step)

```js
const rawOrders = [
  { id: 'ORD-1', customer: 'Alice', tier: 'premium',  items: [{ name: 'Laptop', price: 75000, qty: 1 }, { name: 'Mouse', price: 800, qty: 2 }] },
  { id: 'ORD-2', customer: 'Bob',   tier: 'standard', items: [{ name: 'Keyboard', price: 1200, qty: 1 }] },
  { id: 'ORD-3', customer: null,    tier: 'free',     items: [] },                 // invalid — no customer
  { id: 'ORD-4', customer: 'Carol', tier: 'free',     items: [{ name: 'Mousepad', price: 350, qty: 3 }] },
];
```

Write:
- `validateOrder(order)` → returns `{ valid: true, order }` or `{ valid: false, reason: '...' }`
- `calculateTotal(order)` → returns order with `subtotal` added
- `applyDiscount(order, tiers)` → returns order with `discount` and `finalTotal` added
- `formatOrder(order)` → returns a clean display string
- `processOrders(orders, tiers)` → orchestrates the pipeline, skips invalid orders, logs results

```js
const discountTiers = { premium: 0.15, standard: 0.05, free: 0 };

processOrders(rawOrders, discountTiers);
// Should log formatted summary for ORD-1, ORD-2, ORD-4
// Should log skip reason for ORD-3
```

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Pure function checklist:
// ✅ Same input → always same output
// ✅ No external variable reads (unless const/frozen)
// ✅ No mutation of parameters
// ✅ No I/O (console, DOM, fetch, localStorage)
// ✅ No Math.random() or Date() without injection

// Transforming impure → pure:
// Mutation: arr.push(x)          → return [...arr, x]
// Mutation: arr.sort()           → return [...arr].sort()
// Mutation: obj.key = val        → return { ...obj, key: val }
// External read: uses outerVar   → add as parameter
// I/O: console.log / fetch       → move to caller / wrapper

// Side effect isolation pattern:
const result = pureTransform(data);    // pure
sideEffect(result);                    // impure — at the edge
```

| Type | Characteristic | Examples |
|---|---|---|
| Pure | Same input → same output, no side effects | `add`, `map`, `filter`, `toUpperCase` |
| Impure (mutation) | Modifies input/external state | `push`, `sort`, `splice`, `delete obj.k` |
| Impure (I/O) | Reads/writes external systems | `fetch`, `console.log`, DOM, localStorage |
| Impure (random/time) | Non-deterministic | `Math.random()`, `Date.now()` |

---

## Connected topics

- **[19] Closures** — closures over mutable state make functions impure
- **[20] Higher-order Functions** — composing pure functions is safe and powerful
- **[37] Spread with Objects** — how to update objects without mutating (pure pattern)
- **[44] Memoization** — only works correctly with pure functions
- **[75] Immutability** — the broader philosophy this topic feeds into
