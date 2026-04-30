# 42 — Recursion

---

## What is this?

**Recursion** is when a function calls **itself** to solve a problem by breaking it into smaller versions of the same problem. It keeps calling itself until it hits a **base case** — a condition where it stops and returns a value directly, without another recursive call.

Think of it like Russian nesting dolls — each doll contains a smaller version of itself, until you reach the smallest one that contains nothing.

```js
function countdown(n) {
  if (n === 0) {           // base case — stop here
    console.log('Go!');
    return;
  }
  console.log(n);
  countdown(n - 1);        // recursive call — same function, smaller problem
}

countdown(3);
// 3
// 2
// 1
// Go!
```

---

## Why does it matter?

Some problems are **naturally recursive** — trees, nested structures, file systems, mathematical sequences. Recursion lets you express these cleanly in a way that loops simply can't match. It's also the foundation of many algorithms (binary search, merge sort, tree traversal) and you'll encounter it constantly when working with nested data like JSON, DOM trees, or file directories.

---

## The two non-negotiable rules

1. **Base case** — a condition that stops the recursion. Without this, the function calls itself forever → stack overflow.
2. **Recursive case** — the function calls itself with a **simpler/smaller** input, moving toward the base case.

```js
function recursiveFunction(input) {
  // Rule 1: Base case — stop condition
  if (baseCondition) {
    return baseValue;
  }

  // Rule 2: Recursive case — call self with simpler input
  return recursiveFunction(smallerInput);
}
```

---

## How the call stack works

Each function call is pushed onto the **call stack**. Recursive calls keep stacking until the base case is reached, then they unwind (resolve) from the top down.

```js
function factorial(n) {
  if (n === 0) return 1;         // base case
  return n * factorial(n - 1);  // recursive case
}

factorial(4);
```

Call stack growth:
```
factorial(4)  →  4 * factorial(3)
                     3 * factorial(2)
                         2 * factorial(1)
                             1 * factorial(0)
                                 return 1    ← base case hit
```

Call stack unwind (resolving):
```
factorial(0) = 1
factorial(1) = 1 * 1  = 1
factorial(2) = 2 * 1  = 2
factorial(3) = 3 * 2  = 6
factorial(4) = 4 * 6  = 24
```

---

## Example 1 — Factorial

$$n! = n \times (n-1) \times (n-2) \times \ldots \times 1$$

```js
function factorial(n) {
  // Base case: 0! = 1, and 1! = 1
  if (n <= 1) return 1;

  // Recursive case: n! = n × (n-1)!
  return n * factorial(n - 1);
}

console.log(factorial(0)); // 1
console.log(factorial(1)); // 1
console.log(factorial(5)); // 120   (5 × 4 × 3 × 2 × 1)
console.log(factorial(6)); // 720

// Iterative version for comparison:
function factorialLoop(n) {
  let result = 1;
  for (let i = 2; i <= n; i++) result *= i;
  return result;
}
// Same result — recursion is NOT always better for simple loops
// but it IS more natural for problems with branching recursion
```

---

## Example 2 — Fibonacci

Each Fibonacci number is the sum of the two before it: 0, 1, 1, 2, 3, 5, 8, 13, 21...

$$F(n) = F(n-1) + F(n-2)$$

```js
function fibonacci(n) {
  if (n === 0) return 0;           // base case 1
  if (n === 1) return 1;           // base case 2

  return fibonacci(n - 1) + fibonacci(n - 2); // recursive case
}

console.log(fibonacci(0));  // 0
console.log(fibonacci(1));  // 1
console.log(fibonacci(6));  // 8
console.log(fibonacci(10)); // 55

// ⚠️ This naive version is EXPONENTIALLY slow for large n
// fibonacci(40) makes 2^40 calls — freeze!
// Fix with memoization (topic 44) or iteration
```

Call tree for `fibonacci(4)`:
```
fib(4)
├── fib(3)
│   ├── fib(2)
│   │   ├── fib(1) = 1
│   │   └── fib(0) = 0
│   └── fib(1) = 1
└── fib(2)
    ├── fib(1) = 1
    └── fib(0) = 0
```
Notice `fib(2)` is computed twice, `fib(1)` three times — that's the redundancy memoization fixes.

---

## Example 3 — Flatten a nested array

This is where recursion truly shines — a loop struggles with unknown nesting depth.

```js
function flattenArray(arr) {
  let result = [];

  for (const item of arr) {
    if (Array.isArray(item)) {
      // If item is an array, recurse and spread the result
      result = result.concat(flattenArray(item)); // recursive call
    } else {
      result.push(item); // base case: plain value, add it
    }
  }

  return result;
}

const nested = [1, [2, 3], [4, [5, [6, 7]]], 8];
console.log(flattenArray(nested)); // [1, 2, 3, 4, 5, 6, 7, 8]

// Works for ANY depth — a loop can't do this without knowing depth upfront
const deepNest = [[[[[[42]]]]]];
console.log(flattenArray(deepNest)); // [42]

// Note: arr.flat(Infinity) does this built-in, but recursion teaches the pattern
```

---

## Example 4 — Deep object traversal (real-world: JSON processing)

You receive an arbitrarily nested config object and need to find all values for a given key anywhere in the tree:

```js
function findAllValues(obj, targetKey) {
  let found = [];

  for (const [key, value] of Object.entries(obj)) {
    if (key === targetKey) {
      found.push(value); // collect this match
    }

    // If value is an object (not null), recurse into it
    if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
      found = found.concat(findAllValues(value, targetKey)); // recursive call
    }

    // If value is an array, recurse into each element that is an object
    if (Array.isArray(value)) {
      for (const element of value) {
        if (typeof element === 'object' && element !== null) {
          found = found.concat(findAllValues(element, targetKey));
        }
      }
    }
  }

  return found;
}

const config = {
  name:    'App',
  version: '1.0',
  server: {
    host: 'localhost',
    ssl: {
      host: 'secure.app.com',
      port: 443,
    },
  },
  databases: [
    { name: 'primary', host: 'db1.app.com' },
    { name: 'replica', host: 'db2.app.com' },
  ],
};

findAllValues(config, 'host');
// ['localhost', 'secure.app.com', 'db1.app.com', 'db2.app.com']

findAllValues(config, 'name');
// ['App', 'primary', 'replica']
```

---

## Example 5 — Tree traversal (menus, file systems, org charts)

Tree structures are the most natural use of recursion:

```js
const orgChart = {
  name:  'CEO',
  title: 'Chief Executive Officer',
  reports: [
    {
      name:  'CTO',
      title: 'Chief Technology Officer',
      reports: [
        { name: 'Dev Lead',    title: 'Lead Developer',     reports: [] },
        { name: 'DevOps Lead', title: 'DevOps Engineer',    reports: [] },
      ],
    },
    {
      name:  'CFO',
      title: 'Chief Financial Officer',
      reports: [
        { name: 'Accountant', title: 'Senior Accountant', reports: [] },
      ],
    },
  ],
};

// Print the org chart with indentation
function printOrgChart(node, depth = 0) {
  const indent = '  '.repeat(depth);             // 2 spaces per level
  console.log(`${indent}${node.name} — ${node.title}`);

  for (const report of node.reports) {
    printOrgChart(report, depth + 1);            // recurse with depth+1
  }
}

printOrgChart(orgChart);
// CEO — Chief Executive Officer
//   CTO — Chief Technology Officer
//     Dev Lead — Lead Developer
//     DevOps Lead — DevOps Engineer
//   CFO — Chief Financial Officer
//     Accountant — Senior Accountant

// Count total employees in any subtree
function countEmployees(node) {
  if (node.reports.length === 0) return 1;  // base case: leaf node
  return 1 + node.reports.reduce(
    (sum, report) => sum + countEmployees(report), 0
  );
}

console.log(countEmployees(orgChart)); // 6
```

---

## Example 6 — Deep clone (without structuredClone)

```js
function deepClone(value) {
  // Base cases: primitives and null
  if (value === null || typeof value !== 'object') {
    return value;
  }

  // Array: clone each element recursively
  if (Array.isArray(value)) {
    return value.map(item => deepClone(item));
  }

  // Object: clone each property recursively
  const cloned = {};
  for (const [key, val] of Object.entries(value)) {
    cloned[key] = deepClone(val);
  }
  return cloned;
}

const original = {
  name:    'Alice',
  scores:  [95, 87, 91],
  address: { city: 'Mumbai', zip: '400001' },
};

const clone = deepClone(original);

clone.name           = 'Bob';
clone.scores.push(100);
clone.address.city   = 'Delhi';

console.log(original.name);          // 'Alice'      — untouched
console.log(original.scores);        // [95, 87, 91] — untouched
console.log(original.address.city);  // 'Mumbai'     — untouched
```

---

## Example 7 — Binary search (recursive)

```js
function binarySearch(sortedArr, target, low = 0, high = sortedArr.length - 1) {
  // Base case: search space exhausted
  if (low > high) return -1;

  const mid = Math.floor((low + high) / 2);

  if (sortedArr[mid] === target) return mid;          // found!
  if (sortedArr[mid] < target)                         // target is in right half
    return binarySearch(sortedArr, target, mid + 1, high);
  return binarySearch(sortedArr, target, low, mid - 1); // target is in left half
}

const sorted = [2, 5, 8, 12, 16, 23, 38, 56, 72, 91];
binarySearch(sorted, 23);  // 5 (index)
binarySearch(sorted, 99);  // -1 (not found)
binarySearch(sorted, 2);   // 0
```

Each call eliminates half the search space — $O(\log n)$ time complexity.

---

## Tricky things you'll encounter in the real world

### 1. Missing or wrong base case → stack overflow

```js
// ❌ No base case — infinite recursion → RangeError: Maximum call stack size exceeded
function countDown(n) {
  console.log(n);
  countDown(n - 1); // never stops!
}

// ❌ Base case never reached (wrong condition)
function sumTo(n) {
  if (n === 0.5) return 0; // if n starts as integer, 0.5 is never hit
  return n + sumTo(n - 0.1); // floating point → never exactly 0.5
}

// ✅ Always ensure base case is reachable with your inputs
function sumTo(n) {
  if (n <= 0) return 0; // guard with <= not ===
  return n + sumTo(n - 1);
}
```

---

### 2. Stack depth limits (~10,000–15,000 calls in most JS engines)

```js
// ❌ Recursing on a 100,000-element array → stack overflow
function sumArray(arr, index = 0) {
  if (index === arr.length) return 0;
  return arr[index] + sumArray(arr, index + 1);
}

const million = new Array(100000).fill(1);
sumArray(million); // RangeError: Maximum call stack size exceeded

// ✅ Use a loop for large flat iterations
const sum = million.reduce((a, b) => a + b, 0); // fine
```

Recursion is ideal for **branching** (trees, nested data). For **linear** iteration (walking an array), prefer loops or `reduce`.

---

### 3. Mutual recursion — two functions calling each other

```js
function isEven(n) {
  if (n === 0) return true;
  return isOdd(n - 1);  // calls isOdd
}

function isOdd(n) {
  if (n === 0) return false;
  return isEven(n - 1); // calls isEven
}

isEven(4); // true
isOdd(3);  // true
// Works but doubles the stack depth — just use n % 2 === 0 in practice
```

---

### 4. Tail call optimization (TCO) — mostly theoretical in JS

A **tail call** is a recursive call that is the very last operation — nothing is done with its return value except returning it. Some languages optimize this to use constant stack space. Most JS engines do NOT reliably implement TCO, so don't count on it.

```js
// NOT a tail call — multiplication happens AFTER the recursive call returns
function factorial(n) {
  return n * factorial(n - 1); // ← n * ... means something happens after return
}

// Tail call form (accumulator pattern)
function factorialTail(n, acc = 1) {
  if (n <= 1) return acc;
  return factorialTail(n - 1, n * acc); // last operation is the recursive call
}
// In theory TCO-eligible — in practice, JS engines usually still stack-allocate
```

---

### 5. Circular references cause infinite recursion

```js
const a = { name: 'a' };
const b = { name: 'b', ref: a };
a.ref = b; // circular: a → b → a → b → ...

deepClone(a); // infinite recursion if deepClone doesn't check for cycles

// Handle by tracking visited objects:
function deepClone(value, seen = new WeakSet()) {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value)) return '[Circular]'; // ← detect cycle
  seen.add(value);

  if (Array.isArray(value)) return value.map(v => deepClone(v, seen));

  const cloned = {};
  for (const [k, v] of Object.entries(value)) {
    cloned[k] = deepClone(v, seen);
  }
  return cloned;
}
```

---

## Common mistakes

### Mistake 1 — Forgetting the base case

```js
// ❌ Missing base case
function sum(n) {
  return n + sum(n - 1); // calls forever
}

// ✅ Include base case before recursive call
function sum(n) {
  if (n <= 0) return 0;
  return n + sum(n - 1);
}
```

---

### Mistake 2 — Not progressing toward the base case

```js
// ❌ n doesn't change — infinite loop
function countDown(n) {
  console.log(n);
  return countDown(n); // same n every call
}

// ✅ Each call gets closer to base case
function countDown(n) {
  if (n <= 0) return;
  console.log(n);
  return countDown(n - 1); // n shrinks
}
```

---

### Mistake 3 — Ignoring the return value of the recursive call

```js
// ❌ Recursive return value is discarded
function factorial(n) {
  if (n <= 1) return 1;
  n * factorial(n - 1); // computed but NOT returned!
  // function returns undefined
}

// ✅ Return the recursive call's result
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}
```

---

## Frequently asked questions

**Q: When should I use recursion vs a loop?**
Use recursion when the problem has **natural recursive structure** (trees, nested data, divide-and-conquer). Use loops for **linear iteration** over flat data. Rule of thumb: if you can draw the problem as a tree of sub-problems, recursion fits. If it's a straight line, use a loop.

**Q: Can any recursive function be rewritten as a loop?**
Yes — mathematically, any recursion can be expressed iteratively (using an explicit stack). But the recursive version is often much more readable for tree/graph problems.

**Q: Why does my recursive function return `undefined`?**
Almost always because you forgot to `return` the result of the recursive call. Every branch of a recursive function (except `void` functions) must return a value.

**Q: What is "stack overflow"?**
When the call stack exceeds its maximum depth (~10k–15k frames in V8). Each unresolved recursive call occupies a frame. Too many pending calls → RangeError: Maximum call stack size exceeded.

**Q: What is an accumulator pattern in recursion?**
Passing a running result as a parameter so the final answer is built up as you recurse (rather than built up as calls unwind). Makes the function tail-recursive in form:

```js
// Standard: result built on unwind
function sum(arr) {
  if (arr.length === 0) return 0;
  return arr[0] + sum(arr.slice(1));
}

// Accumulator: result built on descent
function sumAcc(arr, acc = 0) {
  if (arr.length === 0) return acc;
  return sumAcc(arr.slice(1), acc + arr[0]);
}
```

**Q: What's the difference between recursion and iteration in performance?**
Loops are generally faster (no function call overhead per step). Recursive calls add stack frame allocation. For large flat structures, always prefer iteration. For tree traversal, the code clarity of recursion usually outweighs the marginal overhead.

---

## Practice exercises

### Exercise 1 — Easy: Classic mathematical recursions

Write three recursive functions (you may NOT use loops):

1. `power(base, exp)` — returns `base` raised to `exp` (only non-negative integer exponents). Must work: `power(2, 0)` → `1`, `power(3, 4)` → `81`

2. `sumDigits(n)` — returns the sum of all digits of a positive integer. `sumDigits(1234)` → `10`, `sumDigits(9)` → `9`

3. `reverseString(str)` — reverses a string recursively. `reverseString('hello')` → `'olleh'`

```js
// Write your code here
```

---

### Exercise 2 — Medium: Nested structure processing

You have a nested category tree from an e-commerce database:

```js
const categories = {
  name: 'All Products',
  children: [
    {
      name: 'Electronics',
      children: [
        { name: 'Phones',    children: [] },
        { name: 'Laptops',   children: [] },
        { name: 'Tablets',   children: [] },
      ],
    },
    {
      name: 'Clothing',
      children: [
        {
          name: 'Men',
          children: [
            { name: 'Shirts', children: [] },
            { name: 'Pants',  children: [] },
          ],
        },
        { name: 'Women', children: [] },
      ],
    },
    { name: 'Books', children: [] },
  ],
};
```

Write three recursive functions:

1. `countCategories(node)` — total number of nodes (including root)
2. `findCategory(node, name)` — returns the node with matching name, or `null` if not found
3. `getCategoryPaths(node, path = '')` — returns array of all full paths like `['All Products', 'All Products > Electronics', 'All Products > Electronics > Phones', ...]`

```js
// Write your code here
```

---

### Exercise 3 — Hard: JSON deep diff

Write `deepDiff(obj1, obj2)` that recursively compares two plain objects and returns an object describing the differences:

```js
const v1 = {
  name:     'Alice',
  age:      28,
  address:  { city: 'Mumbai', zip: '400001' },
  scores:   [95, 87, 91],
  active:   true,
};

const v2 = {
  name:     'Alice',
  age:      29,                              // changed
  address:  { city: 'Delhi', zip: '400001' }, // city changed
  scores:   [95, 87, 91],
  active:   true,
  email:    'alice@example.com',             // added
};
```

The function should return:
```js
{
  age:     { from: 28,       to: 29 },
  address: {
    city:  { from: 'Mumbai', to: 'Delhi' },
  },
  email:   { from: undefined, to: 'alice@example.com' },
}
// Keys with no changes are NOT included
// Nested diffs are reported at their exact path
```

Rules:
- Compare primitives directly
- Recurse into nested objects
- Skip arrays (treat as primitives — compare with `JSON.stringify`)
- Missing keys on either side → `from: undefined` or `to: undefined`

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Recursion template
function recurse(input) {
  // 1. Base case(s) — always first
  if (baseCondition) return baseValue;

  // 2. Recursive case — simpler input, use return value
  return recurse(smallerInput);
}

// Common patterns:

// Factorial
function factorial(n) {
  return n <= 1 ? 1 : n * factorial(n - 1);
}

// Sum of array
function sum(arr) {
  return arr.length === 0 ? 0 : arr[0] + sum(arr.slice(1));
}

// Flatten nested array
function flatten(arr) {
  return arr.reduce((flat, item) =>
    Array.isArray(item) ? [...flat, ...flatten(item)] : [...flat, item], []);
}

// Tree node count
function count(node) {
  return 1 + node.children.reduce((s, c) => s + count(c), 0);
}

// Deep find in object
function deepGet(obj, key) {
  if (key in obj) return obj[key];
  for (const val of Object.values(obj)) {
    if (typeof val === 'object' && val !== null) {
      const found = deepGet(val, key);
      if (found !== undefined) return found;
    }
  }
  return undefined;
}
```

| Situation | Use recursion? |
|---|---|
| Tree / graph traversal | ✅ Yes — natural fit |
| Nested unknown-depth data | ✅ Yes |
| Divide and conquer algorithms | ✅ Yes |
| Flat array / string iteration | ❌ Use a loop |
| Counter / accumulation over flat list | ❌ Use reduce / loop |
| Large N (>10k depth) | ❌ Risk of stack overflow |

---

## Connected topics

- **[20] Higher-order Functions** — recursion often combines with reduce/map
- **[19] Closures** — recursive functions can close over shared state (accumulators)
- **[44] Memoization** — caching recursive call results (solves the fibonacci problem)
- **[29] reduce** — many recursive patterns have a reduce equivalent for flat data
- **[43] Currying** — another advanced functional pattern built on function composition
