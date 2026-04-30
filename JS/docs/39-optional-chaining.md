# 39 — Optional Chaining (`?.`)

---

## What is this?

**Optional chaining** (`?.`) lets you safely access deeply nested properties without crashing when an intermediate value is `null` or `undefined`. Instead of throwing a `TypeError`, it short-circuits and returns `undefined`.

Think of it as a cautious explorer — before opening a door, it checks "does this door exist?" If not, it stops and says "undefined" rather than walking into a wall.

```js
// Without optional chaining — crashes if user.address is null
const city = user.address.city; // TypeError if address is null/undefined

// With optional chaining — safely returns undefined
const city = user?.address?.city; // undefined — no crash
```

---

## Why does it matter?

Real-world data is messy. API responses have optional fields. Users may not have filled in every profile field. Deeply nested config objects may be partially set. Optional chaining eliminates an entire category of defensive `if` checks and makes accessing uncertain data clean and readable.

---

## Syntax

```js
obj?.property                  // property access
obj?.[expression]              // bracket notation / dynamic key
obj?.method()                  // method call
obj?.method?.()                // method that may not exist
arr?.[index]                   // array element access

// Chaining multiple levels
obj?.a?.b?.c?.d
```

---

## How it works — line by line

```js
const user = {
  name:    'Alice',
  address: {
    city: 'Mumbai',
  },
};

user?.name;              // 'Alice'      — user exists, reads name
user?.address?.city;     // 'Mumbai'     — both exist, reads city
user?.address?.pincode;  // undefined    — pincode key doesn't exist, no error
user?.phone?.number;     // undefined    — phone doesn't exist, short-circuits
user?.phone?.number?.toString(); // undefined — stops at phone, doesn't continue

// Compare without optional chaining:
user.phone.number;       // TypeError: Cannot read properties of undefined
```

When `?.` encounters `null` or `undefined` on the LEFT side, it immediately returns `undefined` and stops evaluating the rest of the chain. This is called **short-circuiting**.

---

## Example 1 — API response with optional fields

```js
// Data from an API — not every user has filled in all fields
const user1 = {
  id:      101,
  name:    'Priya',
  profile: {
    bio:     'Frontend developer',
    website: 'https://priya.dev',
    social: {
      twitter: '@priyatweets',
      github:  'priya-codes',
    },
  },
};

const user2 = {
  id:   102,
  name: 'Bob',
  // profile is missing entirely
};

const user3 = {
  id:     103,
  name:   'Carol',
  profile: {
    bio: 'Designer',
    // social is missing
  },
};

// Safe access — same code works for all three users
function getUserTwitter(user) {
  return user?.profile?.social?.twitter ?? 'No Twitter account';
}

console.log(getUserTwitter(user1)); // '@priyatweets'
console.log(getUserTwitter(user2)); // 'No Twitter account'
console.log(getUserTwitter(user3)); // 'No Twitter account'
```

---

## Example 2 — Optional method calls (`?.()`)

Use `?.()` when the method itself might not exist on the object:

```js
const logger = {
  log(msg) { console.log(`[LOG] ${msg}`); },
};

const silentLogger = {
  // no log method
};

// ✅ Safe method call — won't crash if log doesn't exist
logger.log?.('Hello');        // '[LOG] Hello'
silentLogger.log?.('Hello');  // undefined — no error, method just doesn't exist

// Real-world: optional callback
function processData(data, onComplete) {
  // do processing...
  onComplete?.('Done!');  // only call if onComplete was provided
  // cleaner than: if (typeof onComplete === 'function') onComplete('Done!');
}

processData({ name: 'test' });                  // no callback — fine
processData({ name: 'test' }, msg => console.log(msg)); // 'Done!'
```

---

## Example 3 — Optional bracket notation (`?.[]`)

Use when the key is in a variable or is a number (array index):

```js
const config = {
  database: {
    host: 'localhost',
    port: 5432,
  },
};

const section = 'database';
const key     = 'host';

// Dynamic key access — safe
config?.[section]?.[key];     // 'localhost'
config?.['missing']?.['host']; // undefined — 'missing' key doesn't exist

// Array element access
const users = ['Alice', 'Bob', 'Carol'];
users?.[0];    // 'Alice'
users?.[10];   // undefined — out of bounds, no error

const noArray = null;
noArray?.[0];  // undefined — no error (without ?. this would throw)
```

---

## Example 4 — Optional chaining with `??` (nullish coalescing)

`?.` and `??` are designed to work together. `?.` safely traverses; `??` provides a fallback when the result is `null` or `undefined`:

```js
const order = {
  id:       'ORD-100',
  customer: {
    name: 'Parsh',
    // address is missing
  },
  discount: null,
};

const city      = order?.customer?.address?.city ?? 'City not provided';
const discount  = order?.discount ?? 0;
const notes     = order?.notes ?? 'No special notes';
const firstName = order?.customer?.name?.split(' ')?.[0] ?? 'Customer';

console.log(city);      // 'City not provided'
console.log(discount);  // 0      — null triggers the fallback
console.log(notes);     // 'No special notes'
console.log(firstName); // 'Parsh'
```

---

## Example 5 — Optional chaining in loops and array methods

```js
const orders = [
  { id: 1, customer: { name: 'Alice', address: { city: 'Mumbai' } } },
  { id: 2, customer: { name: 'Bob'   /* no address */ } },
  { id: 3 /* no customer */ },
];

// Safe mapping — no crashes on incomplete data
const cities = orders.map(order => order?.customer?.address?.city ?? 'Unknown');
console.log(cities); // ['Mumbai', 'Unknown', 'Unknown']

// Filter only orders WITH a city
const ordersWithCity = orders.filter(order => order?.customer?.address?.city);
console.log(ordersWithCity.length); // 1
```

---

## Example 6 — Optional chaining with DOM elements

One of the most common uses in frontend code:

```js
// Without optional chaining — crashes if element doesn't exist in the DOM
document.getElementById('submit-btn').addEventListener('click', handler);
// TypeError if #submit-btn doesn't exist on this page

// With optional chaining — safe
document.getElementById('submit-btn')?.addEventListener('click', handler);

// Reading values safely
const inputValue = document.querySelector('#search-input')?.value ?? '';
const modalTitle = document.querySelector('.modal-title')?.textContent?.trim() ?? '';

// Chained DOM traversal
const firstListItem = document.querySelector('ul')?.firstElementChild?.textContent;
```

---

## Example 7 — Short-circuiting prevents side effects

Because `?.` short-circuits, functions on the right side of a `?.` are NOT called if the left side is nullish:

```js
let callCount = 0;

function expensiveOperation() {
  callCount++;
  return 'result';
}

const obj = null;

obj?.someMethod(expensiveOperation()); // expensiveOperation IS called — args are evaluated first
// Be careful: argument expressions are evaluated before ?.

// But chained calls stop:
const result = obj?.process?.();
// obj is null → short-circuits → process() is never called

console.log(result);    // undefined
console.log(callCount); // 1 (expensiveOperation was still called in first example)
```

---

## Tricky things you'll encounter in the real world

### 1. `?.` only guards against `null` and `undefined` — not other falsy values

```js
const obj = {
  count: 0,
  name:  '',
  flag:  false,
};

obj?.count;   // 0     — 0 is not null/undefined, ?. passes through
obj?.name;    // ''    — empty string passes through
obj?.flag;    // false — false passes through

// Don't confuse with ??
obj?.count ?? 'default'; // 0       — 0 is not null/undefined
obj?.missing ?? 'default'; // 'default' — missing is undefined
```

---

### 2. Cannot use `?.` on the LEFT side of an assignment

```js
const user = { profile: {} };

// ❌ SyntaxError — can't assign to an optional chain
user?.profile?.name = 'Alice';

// ✅ Check then assign the traditional way
if (user?.profile) {
  user.profile.name = 'Alice';
}
```

---

### 3. Optional chaining doesn't make a missing key appear

```js
const obj = {};
const val = obj?.missing;  // undefined — doesn't add 'missing' to obj
console.log('missing' in obj); // false — key still doesn't exist
```

---

### 4. `?.` with `delete`

```js
const user = { profile: { bio: 'Hello' } };

delete user?.profile?.bio;    // ✅ Works — deletes bio if path exists
delete user?.missing?.prop;   // No error — short-circuits at 'missing'

console.log(user.profile.bio); // undefined — bio was deleted
```

---

### 5. Stacking `?.` — when you need it vs overkill

```js
const data = { user: { name: 'Alice' } };

// ✅ Appropriate — each step could realistically be null/undefined
data?.user?.name?.toLowerCase();

// ❌ Overkill — if you KNOW data is always an object, don't guard it
data?.user?.name; // when you KNOW data is not null/undefined: data.user?.name

// Use ?. only where null/undefined is actually possible
// Overusing it hides bugs — a TypeError can tell you something went wrong
```

---

### 6. Optional chaining and TypeScript

In TypeScript, `?.` also narrows types. If `?.` returns `undefined`, TypeScript knows the chain was cut. This makes it a powerful tool for type safety beyond just runtime safety.

---

## Common mistakes

### Mistake 1 — Using `&&` chains instead of `?.` (old pattern)

```js
const user = { address: { city: 'Mumbai' } };

// ❌ Old verbose guard chain
const city = user && user.address && user.address.city;

// ✅ Modern — clean and reads naturally
const city2 = user?.address?.city;

// The && pattern also has a subtle bug:
// if city is 0, '' or false, && stops early — ?.  doesn't have this issue
const config = { timeout: 0 };
config && config.timeout; // 0  — correct
config?.timeout;          // 0  — correct (same result here, but safer pattern)
```

---

### Mistake 2 — Confusing `?.` with `?` (ternary) or `??` (nullish coalescing)

```js
// ?. — optional chaining: access nested props safely
user?.address?.city

// ?? — nullish coalescing: provide fallback for null/undefined
user?.address?.city ?? 'Unknown'

// ? : — ternary operator: if/else shorthand
user ? user.address.city : 'Unknown'

// They work together but are different things
```

---

### Mistake 3 — Expecting `?.` to handle ALL errors

```js
const obj = { fn: 'not a function' };

// ?.() only guards against fn being null/undefined
// If fn exists but is NOT a function, you still get a TypeError
obj.fn?.();  // TypeError: obj.fn is not a function

// ✅ Guard the type too if needed:
typeof obj.fn === 'function' && obj.fn();
```

---

## Frequently asked questions

**Q: What is the difference between `?.` and `&&` for checking nested properties?**
`?.` is cleaner and specifically handles `null`/`undefined`. The `&&` operator also short-circuits on all falsy values — so `obj && obj.count` returns `0` if `count` is `0` (not what you want). `obj?.count` correctly returns `0`. Use `?.` in modern code.

**Q: Does `?.` work with function calls at the top level?**
Yes — `fn?.()` calls `fn` only if `fn` is not null/undefined. If `fn` doesn't exist, it returns `undefined`. If `fn` exists but is not a function, it still throws.

**Q: Can I use `?.` in a loop condition?**
Yes, but be careful — if `?.` returns `undefined`, comparisons like `undefined > 5` return `false`, which might silently skip iterations rather than throw an error.

**Q: When should I NOT use `?.`?**
When you expect the value to always be there. Overusing `?.` can swallow bugs — if a property is always supposed to exist and it's missing, a `TypeError` is useful. Only use `?.` where `null`/`undefined` is a **normal, expected** case.

**Q: Is optional chaining the same in all environments?**
It was added in ES2020. It's fully supported in all modern browsers and Node.js 14+. For older environments, Babel can transpile it.

**Q: Does `?.` work on `null` and `undefined` only, or also on `false`/`0`/`''`?**
Only `null` and `undefined`. Accessing a property on `false`, `0`, or `''` still works because those are primitives that have prototype methods (e.g., `(0).toString()` is valid). The `?.` only stops on the two explicitly "nothing" values.

---

## Practice exercises

### Exercise 1 — Easy: Safe user data display

You receive these user objects from an API. Some have incomplete data:

```js
const users = [
  {
    id: 1,
    name: 'Alice',
    contact: { email: 'alice@example.com', phone: '+91-9876543210' },
    preferences: { theme: 'dark', language: 'en' },
  },
  {
    id: 2,
    name: 'Bob',
    contact: { email: 'bob@example.com' },
    // preferences missing
  },
  {
    id: 3,
    name: 'Carol',
    // contact missing entirely
    // preferences missing
  },
];
```

Write a function `formatUserCard(user)` that safely returns:
```js
{
  name:     'Alice',
  email:    'alice@example.com',     // or 'Not provided'
  phone:    '+91-9876543210',        // or 'Not provided'
  theme:    'dark',                  // or 'light' (default)
  language: 'en',                    // or 'en' (default)
}
```

Use `?.` and `??` — no `if` statements allowed.

```js
// Write your code here
```

---

### Exercise 2 — Medium: Order summary with optional fields

```js
const orders = [
  {
    id: 'ORD-001',
    customer: { name: 'Parsh', tier: 'premium' },
    items: [
      { name: 'Laptop', price: 75000 },
      { name: 'Mouse',  price:   800 },
    ],
    discount: { type: 'percent', value: 10 },
    shipping: { method: 'express', cost: 299 },
  },
  {
    id: 'ORD-002',
    customer: { name: 'Alice' },      // no tier
    items: [
      { name: 'Keyboard', price: 1200 },
    ],
    // no discount
    shipping: { method: 'standard', cost: 99 },
  },
  {
    id: 'ORD-003',
    customer: { name: 'Bob', tier: 'free' },
    items: [
      { name: 'Mousepad', price: 350 },
    ],
    discount: { type: 'flat', value: 50 },
    // no shipping (pickup)
  },
];
```

Write `summariseOrder(order)` that returns:
```js
{
  id:           'ORD-001',
  customer:     'Parsh',
  tier:         'premium',           // or 'standard' if missing
  itemCount:    2,
  subtotal:     75800,
  discountAmt:  7580,                // 10% of subtotal, or flat value, or 0 if no discount
  shippingCost: 299,                 // or 0 if no shipping
  total:        68519,               // subtotal - discountAmt + shippingCost
}
```

Use `?.` throughout for safe access.

```js
// Write your code here
```

---

### Exercise 3 — Hard: Config path resolver

You have a deeply nested config object where any level might be missing:

```js
const appConfig = {
  server: {
    http: {
      host: 'localhost',
      port: 3000,
    },
    https: {
      host:    'secure.myapp.com',
      port:    443,
      ssl: {
        certPath: '/etc/ssl/cert.pem',
        keyPath:  '/etc/ssl/key.pem',
      },
    },
  },
  database: {
    primary: {
      host:     'db.myapp.com',
      port:     5432,
      name:     'production_db',
      poolSize: 10,
    },
    // replica is missing
  },
  cache: null,  // explicitly disabled
  features: {
    darkMode:     true,
    betaFeatures: false,
  },
};
```

Write a function `getConfigValue(config, path, defaultValue)` where `path` is a dot-separated string like `'server.https.ssl.certPath'` or `'database.replica.host'`.

The function should:
1. Split the path into parts
2. Walk the object using optional bracket notation `?.[]` at each step
3. Return the value at that path, or `defaultValue` if any step is `null`/`undefined`

```js
getConfigValue(appConfig, 'server.https.ssl.certPath', 'no cert'); // '/etc/ssl/cert.pem'
getConfigValue(appConfig, 'database.replica.host', 'no replica');  // 'no replica'
getConfigValue(appConfig, 'cache.driver', 'memory');               // 'memory' (cache is null)
getConfigValue(appConfig, 'features.darkMode', false);             // true
getConfigValue(appConfig, 'server.http.port', 80);                 // 3000
```

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Property access
obj?.prop                    // undefined if obj is null/undefined
obj?.a?.b?.c                 // chain — stops at first null/undefined

// Dynamic / bracket
obj?.[key]                   // undefined if obj is null/undefined
arr?.[index]                 // undefined if arr is null/undefined

// Method call
obj?.method()                // undefined if obj is null/undefined
obj.method?.()               // undefined if method doesn't exist
obj?.method?.()              // both guards

// With nullish coalescing fallback
obj?.prop ?? 'default'       // 'default' if prop is null/undefined

// Callback guard
callback?.()                 // call only if callback is not null/undefined

// Delete with optional chaining
delete obj?.prop             // no error if obj is null/undefined
```

| Syntax | Purpose | Returns when nullish |
|---|---|---|
| `obj?.prop` | Property access | `undefined` |
| `obj?.[key]` | Dynamic key | `undefined` |
| `obj?.method()` | Object might be null | `undefined` |
| `obj.method?.()` | Method might not exist | `undefined` |
| `obj?.a ?? 'x'` | Access + fallback | `'x'` |

---

## Connected topics

- **[34] Object Basics** — the foundation; `?.` makes property access safe
- **[04] Operators** — `??` (nullish coalescing) is the natural partner to `?.`
- **[36] Object Destructuring** — alternative for extracting nested values safely
- **[57] Error Handling** — `?.` reduces the need for try/catch on property access
- **[66] Selecting DOM Elements** — `?.` is essential when elements may not exist
