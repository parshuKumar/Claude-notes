# 44 — Memoization

---

## What is this?

**Memoization** is an optimization technique where you **cache the result of a function call** and return the cached result when the same inputs are called again — instead of recomputing.

Think of it like a student who solves a hard math problem and writes the answer on a sticky note. Next time someone asks the same question, they just read the note instead of solving it again.

```js
// Without memoization — recomputes every time
function square(n) {
  console.log(`Computing ${n}²`);
  return n * n;
}

square(5); // logs 'Computing 5²', returns 25
square(5); // logs 'Computing 5²' again — wasted work!

// With memoization — computes once, caches result
const memoSquare = memoize(square);

memoSquare(5); // logs 'Computing 5²', returns 25
memoSquare(5); // returns 25 from cache — no log, no recomputation
```

---

## Why does it matter?

Some computations are **expensive** — recursive algorithms, complex calculations, data transformations. If a pure function is called repeatedly with the same arguments, memoization converts a slow repeated computation into a fast cache lookup. It's used in React (`useMemo`, `React.memo`), dynamic programming algorithms, and any performance-critical pure function.

**Critical requirement**: memoization is only valid for **pure functions** — same input must always produce same output. Memoizing an impure function (one that reads external state) gives wrong cached results.

---

## Syntax

```js
// Manual cache with a Map
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Usage
const memoFn = memoize(expensiveFunction);
memoFn(42);    // computed
memoFn(42);    // from cache
```

---

## How it works — line by line

```js
function memoize(fn) {
  const cache = new Map();         // stores key → result pairs, persists via closure

  return function(...args) {       // wrapper function replaces the original
    const key = JSON.stringify(args); // convert args to a string key
                                      // [5] → '[5]', ['a','b'] → '["a","b"]'

    if (cache.has(key)) {          // check: have we computed this before?
      return cache.get(key);       // YES — return the cached result instantly
    }

    const result = fn.apply(this, args); // NO — compute the actual result
    cache.set(key, result);              // store it for next time
    return result;                       // return the fresh result
  };
}

const square = n => n * n;
const memoSquare = memoize(square);

memoSquare(4); // cache miss → computes 16, caches '[4]' → 16, returns 16
memoSquare(5); // cache miss → computes 25, caches '[5]' → 25, returns 25
memoSquare(4); // cache HIT → returns 16 immediately, no computation
memoSquare(5); // cache HIT → returns 25 immediately
```

---

## Example 1 — Solving the Fibonacci performance problem

From topic 42, naive recursive Fibonacci is exponentially slow:

```js
// ❌ Slow — recomputes fib(2), fib(3)... thousands of times
function fibSlow(n) {
  if (n <= 1) return n;
  return fibSlow(n - 1) + fibSlow(n - 2);
}

console.time('slow');
fibSlow(40); // ~1 second+ — millions of redundant calls
console.timeEnd('slow');

// ✅ Memoized — each value computed exactly ONCE
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const fib = memoize(function fibMemo(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // calls the memoized version!
});

console.time('fast');
fib(40); // essentially instant — 40 unique computations, all cached
console.timeEnd('fast');

// Cache is reused across calls
fib(40); // instant — still in cache
fib(41); // only computes fib(41) once, reuses fib(40) from cache
```

**Key**: the recursive calls inside `fibMemo` must call `fib` (the memoized wrapper), not `fibMemo` directly.

---

## Example 2 — Expensive data transformation

```js
// Simulating an expensive computation (sorting + filtering a large dataset)
function processLargeDataset(filters) {
  console.log(`Processing with filters: ${JSON.stringify(filters)}`);
  // Simulate expensive work
  const data = Array.from({ length: 10000 }, (_, i) => ({ id: i, value: Math.random() }));
  return data
    .filter(item => item.value > filters.threshold)
    .sort((a, b) => b.value - a.value)
    .slice(0, filters.limit);
}

const memoProcess = memoize(processLargeDataset);

// First call — expensive
memoProcess({ threshold: 0.9, limit: 5 }); // logs "Processing..."

// Same filters again — instant
memoProcess({ threshold: 0.9, limit: 5 }); // no log — cache hit

// Different filters — computed fresh
memoProcess({ threshold: 0.8, limit: 10 }); // logs "Processing..."
```

---

## Example 3 — Memoization with a closure cache (inline, no helper)

Sometimes you want to memoize one specific function without a helper:

```js
// Self-contained memoized function using closure
const getCountryInfo = (() => {
  const cache = {};   // cache lives in closure

  return function(countryCode) {
    if (cache[countryCode]) {
      console.log(`Cache hit: ${countryCode}`);
      return cache[countryCode];
    }

    console.log(`Fetching: ${countryCode}`);
    // Simulate expensive lookup (in reality this would be a DB/API call)
    const data = {
      IN: { name: 'India',         currency: 'INR', timezone: 'Asia/Kolkata' },
      US: { name: 'United States', currency: 'USD', timezone: 'America/New_York' },
      GB: { name: 'United Kingdom', currency: 'GBP', timezone: 'Europe/London' },
    }[countryCode] ?? { name: 'Unknown', currency: '???', timezone: 'UTC' };

    cache[countryCode] = data;
    return data;
  };
})();

getCountryInfo('IN'); // Fetching: IN  → { name: 'India', ... }
getCountryInfo('US'); // Fetching: US  → { name: 'United States', ... }
getCountryInfo('IN'); // Cache hit: IN → { name: 'India', ... }
getCountryInfo('IN'); // Cache hit: IN → { name: 'India', ... }
```

---

## Example 4 — Memoization with expiry (TTL — time to live)

In real applications, cached data can become stale. A TTL-aware memoizer invalidates cache entries after a timeout:

```js
function memoizeWithTTL(fn, ttlMs = 5000) {
  const cache = new Map(); // key → { value, expiresAt }

  return function(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();

    if (cache.has(key)) {
      const { value, expiresAt } = cache.get(key);
      if (now < expiresAt) {
        return value; // cache hit — not yet expired
      }
      cache.delete(key); // expired — remove it
    }

    const result = fn.apply(this, args);
    cache.set(key, { value: result, expiresAt: now + ttlMs });
    return result;
  };
}

// Exchange rate: cache for 10 seconds
const getExchangeRate = memoizeWithTTL(function(from, to) {
  console.log(`Fetching rate: ${from} → ${to}`);
  // Simulate API call
  const rates = { 'USD-INR': 83.5, 'EUR-INR': 89.2, 'GBP-INR': 104.1 };
  return rates[`${from}-${to}`] ?? null;
}, 10000);

getExchangeRate('USD', 'INR'); // Fetching rate...  → 83.5
getExchangeRate('USD', 'INR'); // from cache        → 83.5
// After 10 seconds...
getExchangeRate('USD', 'INR'); // Fetching rate again (expired)
```

---

## Example 5 — Memoization in React (conceptual)

React's `useMemo` and `React.memo` are the built-in memoization tools:

```js
// useMemo — memoize an expensive calculation in a component
// (Conceptual — requires React environment)

function ProductList({ products, searchTerm }) {
  // Without useMemo — recalculates on EVERY render, even if products and searchTerm didn't change
  const filteredProducts = products.filter(p =>
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  );

  // With useMemo — only recalculates when products or searchTerm changes
  const filteredProducts2 = React.useMemo(
    () => products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    ),
    [products, searchTerm] // dependency array — same concept as cache key
  );

  return filteredProducts2.map(p => <div key={p.id}>{p.name}</div>);
}

// React.memo — memoize an entire component
// Only re-renders if props change
const PriceTag = React.memo(function PriceTag({ price, currency }) {
  console.log('PriceTag rendering');
  return <span>{currency} {price}</span>;
});
```

Understanding memoization from first principles makes React's hooks click immediately.

---

## Example 6 — Comparing cache strategies

```js
// Strategy 1: Simple object cache (fastest, but object keys are strings)
function memoizeSimple(fn) {
  const cache = {};
  return function(arg) {    // single arg only
    if (arg in cache) return cache[arg];
    return (cache[arg] = fn(arg));
  };
}

// Strategy 2: Map cache (handles any key type, multiple args via JSON key)
function memoizeMap(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// Strategy 3: WeakMap cache (for object arguments — allows GC of keys)
function memoizeWeak(fn) {
  const cache = new WeakMap();
  return function(arg) {   // arg must be an object
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}

// When to use each:
// memoizeSimple — single primitive arg, performance-critical
// memoizeMap    — multiple args or mixed types, general purpose
// memoizeWeak   — object arg, want cache to be GC'd when object is no longer referenced
```

---

## Example 7 — Limitations of `JSON.stringify` as a cache key

```js
const memoFn = memoize(someFunction);

// Problem 1: Functions are lost
memoFn({ callback: () => {} });
memoFn({ callback: () => {} }); // Different function reference — but key is same! '{"callback":undefined}'

// Problem 2: Undefined values are stripped
memoFn({ a: 1, b: undefined }); // key: '{"a":1}'
memoFn({ a: 1 });               // key: '{"a":1}' — same key! wrong cache hit

// Problem 3: Key order matters
memoFn({ a: 1, b: 2 }); // key: '{"a":1,"b":2}'
memoFn({ b: 2, a: 1 }); // key: '{"b":2,"a":1}' — different keys! cache miss

// Problem 4: Circular references throw
const circular = { a: 1 };
circular.self = circular;
memoFn(circular); // TypeError: Converting circular structure to JSON

// Solution: use a custom key serializer for complex cases
function memoizeCustomKey(fn, keyFn = JSON.stringify) {
  const cache = new Map();
  return function(...args) {
    const key = keyFn(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

---

## Tricky things you'll encounter in the real world

### 1. Memoizing impure functions gives wrong results

```js
let discount = 0.1;
function getDiscountedPrice(price) {
  return price * (1 - discount); // reads external state — IMPURE
}

const memoPrice = memoize(getDiscountedPrice);

memoPrice(1000); // computes 900, caches 1000 → 900
discount = 0.2;  // discount changed!
memoPrice(1000); // returns 900 from cache — WRONG! should be 800

// Rule: ONLY memoize pure functions
```

---

### 2. Memory leaks — the cache grows without bound

```js
const memoFetch = memoize(fetchUserData);

// Called with millions of different user IDs
for (let id = 0; id < 1000000; id++) {
  memoFetch(id); // cache grows to 1 million entries — memory leak!
}

// Solution: bounded cache (LRU — Least Recently Used)
function memoizeLRU(fn, maxSize = 100) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      // Move to end (most recently used)
      const value = cache.get(key);
      cache.delete(key);
      cache.set(key, value);
      return value;
    }

    const result = fn.apply(this, args);

    if (cache.size >= maxSize) {
      // Delete the oldest entry (first item in Map = least recently used)
      cache.delete(cache.keys().next().value);
    }

    cache.set(key, result);
    return result;
  };
}
```

---

### 3. The cache key problem for object arguments

```js
// Same data, different objects — cache miss!
const processUser = memoize(fn);
processUser({ id: 1, name: 'Alice' });
processUser({ id: 1, name: 'Alice' }); // different object reference, same JSON
                                        // JSON.stringify handles this correctly
                                        // but it's slow for large objects

// If you control the inputs, use a primitive key:
processUser(userId);           // primitive key = fast and unambiguous
```

---

### 4. Memoization and recursion — the self-referencing trap

```js
// ❌ WRONG: recursive function calls the unmemoized version of itself
const memoFib = memoize(function(n) {
  if (n <= 1) return n;
  return memoFib(n - 1) + memoFib(n - 2); // calls memoFib ← OK only if this is the memoized one
});
// This works IF memoFib is the memoized wrapper AND the function body references memoFib

// ❌ This DOESN'T work:
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // calls fib — the unmemoized original
}
const memoFib2 = memoize(fib); // wraps fib, but fib still calls itself
memoFib2(40); // still slow — internal calls bypass the memo cache
```

---

## Common mistakes

### Mistake 1 — Memoizing a function that depends on external state

```js
// ❌ Wrong: getUser reads from external 'users' array
const users = [{ id: 1, name: 'Alice' }];
function getUser(id) {
  return users.find(u => u.id === id);
}
const memoGetUser = memoize(getUser);
memoGetUser(1); // { id: 1, name: 'Alice' } — cached

users[0].name = 'Bob'; // users array changed
memoGetUser(1); // returns { id: 1, name: 'Alice' } — STALE CACHE
```

---

### Mistake 2 — Not realising the cache persists for the lifetime of the memoized function

```js
const memoFn = memoize(expensiveCalc);
memoFn(42);         // computed and cached
// ... 1 hour later
memoFn(42);         // still returns the 1-hour-old cached result

// If data changes over time, use TTL-based memoization (Example 4)
// or manually clear the cache with a cache.clear() method on the wrapper
```

---

### Mistake 3 — Memoizing functions with side effects expecting them to run once

```js
// ❌ console.log is a side effect — will only run on first call
const memoLog = memoize(n => {
  console.log(`Processing ${n}`); // side effect
  return n * 2;
});

memoLog(5); // logs 'Processing 5', returns 10
memoLog(5); // returns 10 from cache — NO log printed
// The side effect was skipped — which may or may not be what you want
```

---

## Frequently asked questions

**Q: What's the difference between memoization and caching?**
Memoization is a specific form of caching applied to function results, keyed by function arguments. "Caching" is the broader term — it covers HTTP caches, database query caches, CDN caches, etc. Memoization always refers to function-level result caching.

**Q: When does memoization NOT help?**
- When the function is called rarely (overhead > savings)
- When inputs are almost always unique (cache never hits)
- When the function is already fast (premature optimization)
- When the function is impure (gives wrong results)
- When inputs are large objects and serializing them is expensive

**Q: Is memoization the same as dynamic programming?**
Memoization is one technique used to implement dynamic programming (specifically "top-down DP"). Dynamic programming is the broader approach of breaking problems into overlapping subproblems and storing solutions. Memoization adds a cache to recursion to achieve this.

**Q: What is the space-time tradeoff?**
Memoization trades **memory (space)** for **speed (time)**. You use more memory to store cached results, but save computation time. This is always the core tradeoff — sometimes the memory cost isn't worth it.

**Q: How do I clear the memoization cache?**
Add a `.clear()` method to the returned function:

```js
function memoize(fn) {
  const cache = new Map();
  function memoized(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  }
  memoized.clear = () => cache.clear();
  return memoized;
}

const memoFn = memoize(expensiveFn);
memoFn(42);
memoFn.clear(); // bust the cache
memoFn(42); // recomputes
```

**Q: Does memoization work with async functions?**
Yes — the Promise is what gets cached. Second call returns the same Promise (already resolved):

```js
function memoizeAsync(fn) {
  const cache = new Map();
  return async function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = await fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

---

## Practice exercises

### Exercise 1 — Easy: Memoize mathematical functions

Using the `memoize` helper below (copy it into your solution), memoize these three functions and prove the cache works by logging when computation actually runs:

```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

Functions to memoize:
1. `isPrime(n)` — returns `true` if n is prime (use trial division — add a `console.log('computing...')` inside)
2. `nthTriangular(n)` — returns the nth triangular number: $1 + 2 + ... + n$ (add a `console.log` inside)
3. `digitSum(n)` — recursively sums digits of n: `digitSum(1234) = 10`

Prove the cache works:
```js
// Each should only log 'computing...' ONCE per unique input
isPrimeMemo(97);   // computes
isPrimeMemo(97);   // from cache — no log
isPrimeMemo(89);   // computes
isPrimeMemo(97);   // from cache — no log
```

```js
// Write your code here
```

---

### Exercise 2 — Medium: Memoized API simulator

Simulate a rate-limited API where each "request" is expensive. Build a memoized fetch function with these features:
- Cache results by URL
- Track cache stats: hits, misses, total calls
- Add a `.stats()` method to the memoized function
- Add a `.invalidate(url)` method to remove a specific URL from cache

```js
// Simulate expensive API call
function fakeApiCall(url) {
  console.log(`[API] Fetching: ${url}`);
  const responses = {
    '/users/1': { id: 1, name: 'Alice', role: 'admin' },
    '/users/2': { id: 2, name: 'Bob',   role: 'viewer' },
    '/products/1': { id: 1, name: 'Laptop', price: 75000 },
  };
  return responses[url] ?? { error: 'Not found' };
}

const cachedFetch = memoizeWithStats(fakeApiCall);

cachedFetch('/users/1');    // API called
cachedFetch('/users/1');    // from cache
cachedFetch('/users/2');    // API called
cachedFetch('/users/1');    // from cache
cachedFetch('/products/1'); // API called

console.log(cachedFetch.stats());
// { hits: 2, misses: 3, total: 5, hitRate: '40.00%' }

cachedFetch.invalidate('/users/1');
cachedFetch('/users/1');    // API called again (cache cleared for this URL)
```

```js
// Write your code here
```

---

### Exercise 3 — Hard: LRU memoize + benchmarking

**Part A**: Implement an LRU (Least Recently Used) memoize function with a configurable max cache size. When the cache is full, evict the least recently used entry.

```js
function memoizeLRU(fn, maxSize = 5) {
  // Write your implementation
  // Hint: Map maintains insertion order — use this for LRU tracking
}
```

Test it:
```js
const memoFn = memoizeLRU(n => {
  console.log(`Computing ${n}`);
  return n * n;
}, 3); // max 3 entries

memoFn(1); // Computing 1
memoFn(2); // Computing 2
memoFn(3); // Computing 3
memoFn(1); // from cache (moves 1 to most recent)
memoFn(4); // Computing 4 — cache full, evict LRU (which is 2)
memoFn(2); // Computing 2 — 2 was evicted!
memoFn(3); // from cache (3 is still there)
```

**Part B**: Benchmark memoized vs non-memoized Fibonacci for `n = 35`:

```js
// Use console.time / console.timeEnd to compare:
// 1. Plain recursive fib(35)
// 2. Memoized fib(35) — first call (cold cache)
// 3. Memoized fib(35) — second call (warm cache)
```

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Core memoize helper
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// With cache.clear()
memoized.clear = () => cache.clear();

// With TTL
function memoizeTTL(fn, ms) {
  const cache = new Map();
  return function(...args) {
    const key  = JSON.stringify(args);
    const now  = Date.now();
    const hit  = cache.get(key);
    if (hit && now < hit.exp) return hit.val;
    const val = fn.apply(this, args);
    cache.set(key, { val, exp: now + ms });
    return val;
  };
}

// LRU (Map preserves insertion order)
// — delete + re-set to move to most-recent
// — cache.keys().next().value is the LRU entry
```

| Technique | Use case |
|---|---|
| Basic memoize | Pure fn, infinite valid cache |
| TTL memoize | Data that becomes stale |
| LRU memoize | Bounded memory, many unique inputs |
| WeakMap memoize | Object args, allow GC of unused entries |

**When to memoize:**
- ✅ Pure function (same input → same output always)
- ✅ Called repeatedly with same args
- ✅ Computation is measurably slow
- ❌ Impure function
- ❌ Called rarely or always with unique args
- ❌ Already fast (don't optimize prematurely)

---

## Connected topics

- **[41] Pure Functions** — memoization ONLY works correctly with pure functions
- **[19] Closures** — the cache Map lives in a closure — this is how it persists
- **[42] Recursion** — memoized recursion = top-down dynamic programming
- **[43] Currying** — curried functions are pure and safe to memoize
- **[44 → React] useMemo / React.memo** — memoization baked into React's rendering model
