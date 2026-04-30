# 38 — Object.keys(), Object.values(), Object.entries()

---

## What is this?

These three static methods convert an object's data into an **array**, so you can use all the array methods (`.map()`, `.filter()`, `.reduce()`, etc.) on object data.

- **`Object.keys(obj)`** → array of the object's property names (keys)
- **`Object.values(obj)`** → array of the object's property values
- **`Object.entries(obj)`** → array of `[key, value]` pairs

```js
const scores = { alice: 95, bob: 82, carol: 78 };

Object.keys(scores);    // ['alice', 'bob', 'carol']
Object.values(scores);  // [95, 82, 78]
Object.entries(scores); // [['alice', 95], ['bob', 82], ['carol', 78]]
```

Think of it as giving you three different "views" into the same object — keys only, values only, or both together.

---

## Why does it matter?

Objects don't have array methods built in. You can't call `.map()` or `.filter()` directly on an object. These three methods are the bridge — they let you iterate, transform, and search object data using the full power of array methods you already know.

---

## Syntax

```js
Object.keys(obj)      // → string[]
Object.values(obj)    // → any[]
Object.entries(obj)   // → [string, any][]

// Typical usage pattern:
Object.entries(obj).map(([key, value]) => /* transform */);
Object.entries(obj).filter(([key, value]) => /* condition */);
Object.entries(obj).reduce((acc, [key, value]) => /* accumulate */, initial);
```

---

## How it works — line by line

```js
const product = {
  name:    'Laptop',
  price:   75000,
  inStock: true,
  rating:  4.5,
};

// Object.keys — gives you the property names as strings
const keys = Object.keys(product);
// ['name', 'price', 'inStock', 'rating']

// Object.values — gives you the property values
const values = Object.values(product);
// ['Laptop', 75000, true, 4.5]

// Object.entries — gives you pairs [key, value]
const entries = Object.entries(product);
// [
//   ['name',    'Laptop'],
//   ['price',   75000],
//   ['inStock', true],
//   ['rating',  4.5],
// ]

// Iterating with for...of + destructuring
for (const [key, value] of Object.entries(product)) {
  console.log(`${key}: ${value}`);
}
// name: Laptop
// price: 75000
// inStock: true
// rating: 4.5
```

---

## Example 1 — Counting and checking

```js
const userPermissions = {
  read:    true,
  write:   true,
  delete:  false,
  publish: false,
  admin:   false,
};

// How many permissions does this user have?
const totalPermissions = Object.keys(userPermissions).length;
console.log(totalPermissions); // 5

// How many are currently granted?
const granted = Object.values(userPermissions).filter(v => v === true).length;
console.log(granted); // 2

// Which ones are granted?
const grantedKeys = Object.keys(userPermissions).filter(
  key => userPermissions[key] === true
);
console.log(grantedKeys); // ['read', 'write']

// Better — use entries when you need both key and value
const grantedKeys2 = Object.entries(userPermissions)
  .filter(([key, value]) => value === true)
  .map(([key]) => key);
console.log(grantedKeys2); // ['read', 'write']
```

---

## Example 2 — Transforming object values (map over an object)

`Object.fromEntries()` (ES2019) is the reverse of `Object.entries()` — it converts an array of `[key, value]` pairs back into an object. Together they let you "map" over an object:

```js
const prices = {
  apple:  120,
  banana:  40,
  mango:  200,
  grape:  150,
};

// Apply 10% discount to all prices
const discounted = Object.fromEntries(
  Object.entries(prices).map(([item, price]) => [item, price * 0.9])
);

console.log(discounted);
// { apple: 108, banana: 36, mango: 180, grape: 135 }
console.log(prices.apple); // 120 — original unchanged

// Round to 2 decimal places
const rounded = Object.fromEntries(
  Object.entries(prices).map(([item, price]) => [item, Math.round(price * 0.9)])
);
console.log(rounded);
// { apple: 108, banana: 36, mango: 180, grape: 135 }
```

---

## Example 3 — Filtering an object by value

```js
const inventory = {
  laptop:    5,
  mouse:     0,
  keyboard: 12,
  monitor:   0,
  headset:   3,
};

// Keep only in-stock items (qty > 0)
const inStock = Object.fromEntries(
  Object.entries(inventory).filter(([item, qty]) => qty > 0)
);

console.log(inStock);
// { laptop: 5, keyboard: 12, headset: 3 }

// Keep only items with qty >= 5
const highStock = Object.fromEntries(
  Object.entries(inventory).filter(([, qty]) => qty >= 5)
  // Note the comma: [, qty] means "skip the key, only use value"
);

console.log(highStock);
// { laptop: 5, keyboard: 12 }
```

---

## Example 4 — Filtering an object by key

```js
const config = {
  DB_HOST:     'localhost',
  DB_PORT:     5432,
  APP_NAME:    'MyApp',
  APP_VERSION: '2.1.0',
  SECRET_KEY:  'super-secret-value',
  SECRET_SALT: 'another-secret',
};

// Get only keys that start with 'APP_'
const appConfig = Object.fromEntries(
  Object.entries(config).filter(([key]) => key.startsWith('APP_'))
);
console.log(appConfig);
// { APP_NAME: 'MyApp', APP_VERSION: '2.1.0' }

// Remove keys that contain 'SECRET'
const safeConfig = Object.fromEntries(
  Object.entries(config).filter(([key]) => !key.includes('SECRET'))
);
console.log(safeConfig);
// { DB_HOST: 'localhost', DB_PORT: 5432, APP_NAME: 'MyApp', APP_VERSION: '2.1.0' }
```

---

## Example 5 — Reducing an object (aggregation)

```js
const salesByMonth = {
  Jan: 45000,
  Feb: 52000,
  Mar: 61000,
  Apr: 48000,
  May: 55000,
  Jun: 70000,
};

// Total annual sales
const total = Object.values(salesByMonth).reduce((sum, amount) => sum + amount, 0);
console.log(total); // 331000

// Best month
const bestMonth = Object.entries(salesByMonth).reduce(
  (best, [month, amount]) => amount > best.amount ? { month, amount } : best,
  { month: '', amount: -Infinity }
);
console.log(bestMonth); // { month: 'Jun', amount: 70000 }

// Average
const average = total / Object.keys(salesByMonth).length;
console.log(average.toFixed(2)); // '55166.67'

// Months above average
const aboveAverage = Object.entries(salesByMonth)
  .filter(([, amount]) => amount > average)
  .map(([month]) => month);
console.log(aboveAverage); // ['Mar', 'May', 'Jun']
```

---

## Example 6 — Building objects dynamically with `Object.fromEntries`

```js
// Convert an array of key-value pairs into an object
const pairs = [['name', 'Alice'], ['age', 28], ['city', 'Mumbai']];
const obj = Object.fromEntries(pairs);
console.log(obj); // { name: 'Alice', age: 28, city: 'Mumbai' }

// Convert a Map to a plain object
const map = new Map([['theme', 'dark'], ['lang', 'en']]);
const fromMap = Object.fromEntries(map);
console.log(fromMap); // { theme: 'dark', lang: 'en' }

// Convert URL search params to an object
const params = new URLSearchParams('page=2&limit=10&sort=asc');
const paramsObj = Object.fromEntries(params);
console.log(paramsObj); // { page: '2', limit: '10', sort: 'asc' }
```

---

## Example 7 — Renaming all keys in an object

```js
const legacyUser = {
  usr_name:  'alice92',
  usr_email: 'alice@example.com',
  usr_age:   28,
};

// Remove the 'usr_' prefix from all keys
const cleanUser = Object.fromEntries(
  Object.entries(legacyUser).map(([key, value]) => [
    key.replace('usr_', ''),  // transform key
    value                      // value unchanged
  ])
);

console.log(cleanUser);
// { name: 'alice92', email: 'alice@example.com', age: 28 }
```

---

## Example 8 — Grouping data (before `Object.groupBy`)

```js
const transactions = [
  { id: 1, type: 'income',  amount: 5000 },
  { id: 2, type: 'expense', amount: 1200 },
  { id: 3, type: 'income',  amount: 3000 },
  { id: 4, type: 'expense', amount:  800 },
  { id: 5, type: 'income',  amount: 4500 },
];

// Group by 'type' using reduce
const grouped = transactions.reduce((acc, transaction) => {
  const { type } = transaction;
  return {
    ...acc,
    [type]: [...(acc[type] ?? []), transaction],
  };
}, {});

console.log(grouped);
// {
//   income:  [ {id:1,...}, {id:3,...}, {id:5,...} ],
//   expense: [ {id:2,...}, {id:4,...} ],
// }

// Now use Object.entries to summarise each group
Object.entries(grouped).forEach(([type, items]) => {
  const total = items.reduce((sum, t) => sum + t.amount, 0);
  console.log(`${type}: ₹${total}`);
});
// income: ₹12500
// expense: ₹2000
```

---

## Tricky things you'll encounter in the real world

### 1. These methods only return OWN ENUMERABLE properties

```js
// Inherited properties (from prototype) are NOT included
function Person(name) { this.name = name; }
Person.prototype.greet = function() { return `Hi, ${this.name}`; };

const alice = new Person('Alice');
Object.keys(alice);    // ['name']      — 'greet' is on prototype, excluded
Object.values(alice);  // ['Alice']
```

Non-enumerable properties (like those added via `Object.defineProperty` with `enumerable: false`) are also excluded.

---

### 2. Integer-like keys sort first, numerically

```js
const mixed = { b: 2, 10: 'ten', a: 1, 2: 'two' };
Object.keys(mixed); // ['2', '10', 'b', 'a']
// Integer-like keys (2, 10) come first in numeric order,
// then string keys in insertion order
```

This matches the property ordering rules from topic 34. If order matters, don't use objects — use a Map or array.

---

### 3. `Object.values()` on an array gives the array values (not indexes)

```js
const arr = ['a', 'b', 'c'];
Object.keys(arr);    // ['0', '1', '2']    — indexes as strings
Object.values(arr);  // ['a', 'b', 'c']    — same as the array itself
Object.entries(arr); // [['0','a'],['1','b'],['2','c']]
// Use array methods directly on arrays — these are for objects
```

---

### 4. `Object.fromEntries` is ES2019 — check environment compatibility

It's available in all modern browsers and Node.js 12+. For older environments, you'd use `reduce`:

```js
// Object.fromEntries polyfill equivalent
const entries = [['a', 1], ['b', 2]];

// Modern
Object.fromEntries(entries); // { a: 1, b: 2 }

// Older alternative
entries.reduce((obj, [key, val]) => ({ ...obj, [key]: val }), {});
```

---

### 5. Mutating an object while iterating over it — don't

```js
const obj = { a: 1, b: 2, c: 3 };

// ❌ Dangerous — modifying the source object during iteration
Object.keys(obj).forEach(key => {
  obj[key] *= 2;  // modifying obj while iterating its snapshot of keys
});
// This works here because Object.keys() takes a snapshot first,
// but it's a bad pattern — easy to introduce bugs

// ✅ Build a new object instead
const doubled = Object.fromEntries(
  Object.entries(obj).map(([k, v]) => [k, v * 2])
);
```

---

### 6. Empty object edge cases

```js
const empty = {};
Object.keys(empty);    // []
Object.values(empty);  // []
Object.entries(empty); // []

// Safe to use — no errors, just empty arrays
Object.keys(empty).length === 0; // true — useful "is empty?" check
```

---

## Common mistakes

### Mistake 1 — Using `for...in` when `Object.keys` is more appropriate

```js
const scores = { alice: 95, bob: 82 };

// ❌ for...in also iterates INHERITED properties — can cause bugs
for (const key in scores) {
  console.log(key, scores[key]); // might include prototype properties
}

// ✅ Object.keys only gives OWN properties
Object.keys(scores).forEach(key => {
  console.log(key, scores[key]);
});

// Or if you still use for...in, always guard with hasOwnProperty:
for (const key in scores) {
  if (Object.hasOwn(scores, key)) {
    console.log(key, scores[key]);
  }
}
```

---

### Mistake 2 — Forgetting destructuring in the entries callback

```js
const data = { x: 10, y: 20 };

// ❌ Forgetting to destructure — entry is an array [key, value]
Object.entries(data).forEach(entry => {
  console.log(entry);       // ['x', 10] — you get the pair, not the value
  console.log(entry.value); // undefined — arrays don't have a .value property
});

// ✅ Destructure the pair immediately
Object.entries(data).forEach(([key, value]) => {
  console.log(key, value);  // 'x' 10
});
```

---

### Mistake 3 — Using `Object.keys().map()` when `Object.entries()` is cleaner

```js
const prices = { apple: 100, banana: 50 };

// ❌ Clunky — going back into the object via bracket notation
const result = Object.keys(prices).map(key => `${key}: ₹${prices[key]}`);

// ✅ Use entries — you already have both key and value
const result2 = Object.entries(prices).map(([item, price]) => `${item}: ₹${price}`);
// ['apple: ₹100', 'banana: ₹50']
```

---

## Frequently asked questions

**Q: What's the difference between `Object.keys` and `for...in`?**
`for...in` iterates over ALL enumerable properties including inherited ones from the prototype chain. `Object.keys` only returns the object's **own** enumerable properties. Always prefer `Object.keys` / `Object.entries` for predictable results.

**Q: What is `Object.fromEntries` and when was it added?**
It's the reverse of `Object.entries` — converts an array of `[key, value]` pairs back into an object. Added in ES2019 (ES10). Without it, you'd use `.reduce()` to rebuild an object from entries.

**Q: Do these methods work on class instances?**
Yes — they return the instance's own enumerable properties (the data set in the constructor). They do NOT include class methods (which live on the prototype).

**Q: Can I use these on a `Map`?**
No — Maps aren't plain objects. Use `map.forEach()`, `map.keys()`, `map.values()`, or `map.entries()` (which return iterators). To convert a Map to an array of pairs, use `[...map.entries()]`.

**Q: Is there `Object.entries` for nested objects?**
Not built in. You'd need to write a recursive function or use a library like Lodash.

**Q: How do I check if an object is empty?**
```js
Object.keys(obj).length === 0  // most common
```

---

## Practice exercises

### Exercise 1 — Easy: Product stats

```js
const productRatings = {
  laptop:     4.5,
  mouse:      4.2,
  keyboard:   4.8,
  monitor:    3.9,
  headset:    4.6,
  webcam:     3.7,
};
```

Using `Object.keys`, `Object.values`, and `Object.entries`:
1. Log all product names (keys)
2. Calculate the average rating across all products
3. Find the highest rated product (name + rating)
4. Find all products with a rating below 4.0 (return as array of names)
5. Log how many products there are in total

```js
// Write your code here
```

---

### Exercise 2 — Medium: Transform and filter a cart

```js
const cart = {
  apple:     { price: 120, qty: 3 },
  banana:    { price:  40, qty: 0 },
  mango:     { price: 200, qty: 2 },
  grape:     { price: 150, qty: 0 },
  watermelon:{ price: 300, qty: 1 },
};
```

Write code that:
1. Filters out items with `qty === 0` and returns a new object (use `Object.entries` + `Object.fromEntries`)
2. From the filtered cart, calculates the line total (`price × qty`) for each item — return a new object `{ apple: 360, mango: 400, watermelon: 300 }`
3. Calculates the grand total of the order
4. Returns the most expensive single item (by `price`, not total) — name + price

All using `Object.entries`, `Object.fromEntries`, array methods. No mutation.

```js
// Write your code here
```

---

### Exercise 3 — Hard: Report generator

You receive monthly sales data across multiple regions:

```js
const salesData = {
  north: { Jan: 42000, Feb: 38000, Mar: 55000, Apr: 61000, May: 49000, Jun: 72000 },
  south: { Jan: 35000, Feb: 41000, Mar: 39000, Apr: 52000, May: 60000, Jun: 58000 },
  east:  { Jan: 28000, Feb: 32000, Mar: 45000, Apr: 38000, May: 41000, Jun: 50000 },
  west:  { Jan: 51000, Feb: 48000, Mar: 62000, Apr: 70000, May: 65000, Jun: 80000 },
};
```

Write `generateReport(data)` that returns:

```js
{
  totalByRegion: {
    north: 317000,
    south: 285000,
    east:  234000,
    west:  376000,
  },
  totalByMonth: {
    Jan: 156000,
    Feb: 159000,
    // ... all 6 months
  },
  topRegion:  { name: 'west',  total: 376000 },
  topMonth:   { name: 'Jun',   total: 310000 },
  grandTotal: 1212000,
}
```

Use `Object.entries`, `Object.fromEntries`, `.map()`, `.reduce()`, and `.filter()` — no manual loops.

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
Object.keys(obj)      // → ['key1', 'key2', ...]
Object.values(obj)    // → [val1, val2, ...]
Object.entries(obj)   // → [['key1', val1], ['key2', val2], ...]
Object.fromEntries(entries) // → { key1: val1, ... }  (ES2019)

// Common patterns:
// Iterate
Object.entries(obj).forEach(([k, v]) => console.log(k, v));

// Map values
Object.fromEntries(Object.entries(obj).map(([k, v]) => [k, transform(v)]));

// Filter by value
Object.fromEntries(Object.entries(obj).filter(([, v]) => condition(v)));

// Filter by key
Object.fromEntries(Object.entries(obj).filter(([k]) => condition(k)));

// Sum values
Object.values(obj).reduce((sum, v) => sum + v, 0);

// Check empty
Object.keys(obj).length === 0;

// Get key count
Object.keys(obj).length;
```

| Method | Returns | Use when you need |
|---|---|---|
| `Object.keys(obj)` | `string[]` | Just the property names |
| `Object.values(obj)` | `any[]` | Just the values |
| `Object.entries(obj)` | `[string, any][]` | Both key and value |
| `Object.fromEntries(pairs)` | `object` | Build object from pairs |

---

## Connected topics

- **[34] Object Basics** — the foundation: property access and structure
- **[29] reduce** — heavily used with `Object.entries` for aggregation
- **[27] map** + **[28] filter** — combined with `Object.entries` + `Object.fromEntries`
- **[38 → 12] for...in** — the older alternative (iterates inherited properties too)
- **[40] Computed Property Names** — dynamic keys in `Object.fromEntries` results
