# 34 — Object Basics

---

## What is this?

An **object** is a collection of related data stored as **key-value pairs**. Think of it like a real-world identity card — it holds multiple pieces of information (name, age, address) all grouped under one thing. Instead of creating five separate variables for a user, you create one object with five properties.

```js
// Five messy variables — hard to manage
const userName    = 'Alice';
const userAge     = 28;
const userEmail   = 'alice@example.com';
const userCountry = 'India';
const userActive  = true;

// One clean object
const user = {
  name:    'Alice',
  age:     28,
  email:   'alice@example.com',
  country: 'India',
  active:  true,
};
```

---

## Why does it matter?

Nearly everything in real-world JavaScript is an object — API responses, DOM elements, configuration, form data, database records. Understanding how to create, read, update, and delete object properties is the most foundational skill for working with data in JS.

---

## Syntax

```js
// Object literal (most common way to create)
const objectName = {
  key1: value1,          // property: any valid identifier
  key2: value2,
  'key with spaces': 3,  // quoted key — when key has special chars or spaces
};

// Read (dot notation)
objectName.key1;

// Read (bracket notation)
objectName['key1'];

// Update
objectName.key1 = newValue;

// Add new property
objectName.newKey = newValue;

// Delete
delete objectName.key1;
```

---

## How it works — line by line

```js
const product = {        // create an object and store it in 'product'
  name: 'Laptop',        // key 'name' → value 'Laptop'
  price: 75000,          // key 'price' → value 75000
  inStock: true,         // key 'inStock' → value true
};

product.name;            // 'Laptop'  — dot notation reads the 'name' property
product['price'];        // 75000     — bracket notation, same result

product.price = 70000;   // update: 'price' key now holds 70000
product.discount = 5;    // add: new key 'discount' added with value 5

delete product.inStock;  // remove: 'inStock' key is gone entirely

console.log(product);
// { name: 'Laptop', price: 70000, discount: 5 }
```

---

## Ways to create objects

### 1 — Object literal (most common)

```js
const book = {
  title:  'Atomic Habits',
  author: 'James Clear',
  pages:  320,
};
```

### 2 — `new Object()` (avoid — verbose, no benefit)

```js
const book = new Object();
book.title  = 'Atomic Habits';
book.author = 'James Clear';
// Same result as literal, but ugly
```

### 3 — Object.create() (for prototype-based patterns — covered in topic 46)

```js
const bookProto = {
  describe() {
    return `${this.title} by ${this.author}`;
  }
};
const book = Object.create(bookProto);
book.title  = 'Atomic Habits';
book.author = 'James Clear';
```

### 4 — Factory function (creates objects with a function)

```js
function createProduct(name, price) {
  return {
    name,           // shorthand: same as name: name
    price,
    inStock: true,
  };
}
const laptop = createProduct('Laptop', 75000);
const phone  = createProduct('Phone',  30000);
```

---

## Reading properties — dot vs bracket notation

### Dot notation

```js
const order = { id: 101, status: 'shipped', total: 4500 };

order.id;       // 101
order.status;   // 'shipped'
order.total;    // 4500
order.missing;  // undefined — accessing a key that doesn't exist: no error, just undefined
```

### Bracket notation

Use when the key is:
- Stored in a variable
- Contains spaces or special characters
- Computed dynamically

```js
const order = { id: 101, status: 'shipped' };

const key = 'status';
order[key];           // 'shipped' — key is a variable, can't use dot here
order['id'];          // 101

// Dynamic key access — real use case
function getField(obj, fieldName) {
  return obj[fieldName]; // can't use obj.fieldName — fieldName is a variable
}
getField(order, 'status'); // 'shipped'
getField(order, 'id');     // 101
```

```js
// Keys with special characters (rare, but you'll see it in API data)
const data = {
  'first-name': 'Alice',
  'user id':    42,
};
data['first-name']; // 'Alice' — must use brackets, dot would be a syntax error
data.first-name;    // SyntaxError (JS reads it as data.first MINUS name)
```

---

## Example 1 — Create, read, update, delete in action

```js
// CREATE — a user profile object
const userProfile = {
  username:  'parsh_dev',
  email:     'parsh@example.com',
  age:       22,
  isPremium: false,
};

// READ
console.log(userProfile.username);   // 'parsh_dev'
console.log(userProfile['email']);   // 'parsh@example.com'
console.log(userProfile.phone);      // undefined — key doesn't exist, no error

// UPDATE — user upgrades to premium
userProfile.isPremium = true;
console.log(userProfile.isPremium);  // true

// ADD — user adds a bio later
userProfile.bio = 'Learning JS every day.';
console.log(userProfile.bio);        // 'Learning JS every day.'

// DELETE — remove email for privacy
delete userProfile.email;
console.log(userProfile.email);      // undefined — key is gone
console.log(userProfile);
// { username: 'parsh_dev', age: 22, isPremium: true, bio: 'Learning JS every day.' }
```

---

## Example 2 — Real world: product catalog entry

```js
// Simulating data you'd receive from a backend API
const product = {
  id:          'SKU-4821',
  name:        'Wireless Headphones',
  brand:       'SoundMax',
  price:       2499,
  currency:    'INR',
  stock:       14,
  rating:      4.3,
  tags:        ['audio', 'wireless', 'sale'],
  dimensions: {             // nested object
    weight: '250g',
    color:  'matte black',
  },
};

// Read top-level
console.log(product.name);          // 'Wireless Headphones'
console.log(product.price);         // 2499

// Read nested object
console.log(product.dimensions.color);        // 'matte black'
console.log(product['dimensions']['weight']); // '250g'

// Read array inside object
console.log(product.tags[0]);       // 'audio'
console.log(product.tags.length);   // 3

// Update — apply a discount
product.price    = 1999;
product.onSale   = true;

// Conditionally add a key
if (product.stock === 0) {
  product.soldOut = true;
}
// product.soldOut is NOT added above — stock is 14, condition is false

console.log(product.price);   // 1999
console.log(product.onSale);  // true
console.log(product.soldOut); // undefined
```

---

## Example 3 — Checking if a property exists

Never rely on `undefined` to check existence — a property could intentionally hold `undefined`.

```js
const config = {
  theme:    'dark',
  language: 'en',
  timeout:  undefined,   // intentionally undefined
};

// BAD — can't distinguish "key missing" from "key = undefined"
if (config.timeout === undefined) {
  console.log('no timeout set'); // prints — but the key IS there
}

// GOOD — 'in' operator checks if key EXISTS, regardless of value
'timeout'  in config;   // true  — key exists (even though value is undefined)
'fontSize' in config;   // false — key doesn't exist

// ALSO GOOD — hasOwnProperty (checks only own properties, not inherited)
config.hasOwnProperty('theme');    // true
config.hasOwnProperty('toString'); // false — inherited from Object.prototype

// MODERN — Object.hasOwn() (preferred over hasOwnProperty)
Object.hasOwn(config, 'theme');    // true
Object.hasOwn(config, 'toString'); // false
```

---

## Example 4 — Shorthand property names (ES6)

When a variable name matches the key name, you can write it once:

```js
const name    = 'Alice';
const age     = 28;
const country = 'India';

// OLD verbose way
const user = {
  name:    name,
  age:     age,
  country: country,
};

// NEW shorthand (ES6) — same result
const user = { name, age, country };

console.log(user); // { name: 'Alice', age: 28, country: 'India' }
```

Real-world use: building objects from function parameters:

```js
function createUser(name, email, role) {
  return { name, email, role };   // clean shorthand
}
const admin = createUser('Bob', 'bob@example.com', 'admin');
// { name: 'Bob', email: 'bob@example.com', role: 'admin' }
```

---

## Example 5 — Iterating over an object

Objects aren't arrays — you can't use `forEach` directly. Use these instead:

```js
const scores = {
  alice: 95,
  bob:   82,
  carol: 78,
};

// for...in — iterates over keys
for (const player in scores) {
  console.log(`${player}: ${scores[player]}`);
}
// alice: 95
// bob: 82
// carol: 78

// Object.keys() → array of keys
Object.keys(scores);   // ['alice', 'bob', 'carol']

// Object.values() → array of values
Object.values(scores); // [95, 82, 78]

// Object.entries() → array of [key, value] pairs
Object.entries(scores);
// [['alice', 95], ['bob', 82], ['carol', 78]]

// Combine with array methods
const topScorers = Object.entries(scores)
  .filter(([player, score]) => score >= 80)
  .map(([player, score]) => player);
console.log(topScorers); // ['alice', 'bob']
```

*(Full coverage of Object.keys/values/entries in topic 38.)*

---

## Example 6 — Nested objects and safe access

```js
const order = {
  id:       'ORD-991',
  customer: {
    name:    'Parsh',
    address: {
      city:    'Mumbai',
      pincode: '400001',
    },
  },
  items: [
    { sku: 'A1', qty: 2 },
    { sku: 'B3', qty: 1 },
  ],
};

// Deep access
order.customer.name;                  // 'Parsh'
order.customer.address.city;          // 'Mumbai'
order.items[0].sku;                   // 'A1'
order.items[1].qty;                   // 1

// DANGER — accessing nested key on missing parent throws TypeError
order.shipping.carrier;               // TypeError: Cannot read properties of undefined

// SAFE — optional chaining (covered in depth in topic 39)
order.shipping?.carrier;              // undefined — no error
order.customer?.address?.pincode;     // '400001'
```

---

## Tricky things you'll encounter in the real world

### 1. Objects are reference types — assignment copies the reference, not the data

```js
const original = { score: 100 };
const copy      = original;     // copy points to the SAME object in memory

copy.score = 999;
console.log(original.score); // 999 — original is changed!

// To get a real shallow copy, use spread:
const safeCopy = { ...original };
safeCopy.score = 1;
console.log(original.score); // 999 — original unchanged
```

---

### 2. `delete` only removes own properties — and it's rarely needed

```js
const user = { name: 'Alice', temp: 'delete me' };
delete user.temp;
console.log(user); // { name: 'Alice' }

// delete returns true even if the key didn't exist
delete user.nonExistent; // true — no error

// delete does NOT work on variables:
const x = 5;
delete x; // false — can't delete variables, only object properties
```

In modern code, instead of deleting properties, prefer building a new object without that key:

```js
const { temp, ...cleanUser } = user; // destructuring + rest (topic 36)
console.log(cleanUser); // { name: 'Alice' }
```

---

### 3. Property order in objects

Since ES2015, integer-like keys sort first (numerically), then string keys in insertion order, then Symbol keys:

```js
const weird = {
  b:   2,
  a:   1,
  100: 'hundred',
  2:   'two',
};

Object.keys(weird); // ['2', '100', 'b', 'a']
// integer keys (2, 100) first in numeric order, then string keys in insertion order
```

Don't rely on property order for logic — use arrays when order matters.

---

### 4. `const` objects are NOT immutable

`const` prevents reassigning the variable — it does NOT prevent mutating the object:

```js
const config = { theme: 'dark' };
config.theme = 'light'; // ✅ allowed — mutating a property, not reassigning
config = {};             // ❌ TypeError — can't reassign a const variable

// For a truly frozen object:
const frozen = Object.freeze({ theme: 'dark' });
frozen.theme = 'light'; // silently fails (no error in non-strict mode)
console.log(frozen.theme); // 'dark' — unchanged
```

---

### 5. Computed property names (preview)

You can use an expression as a key using `[]`:

```js
const field = 'email';
const user = {
  name:    'Alice',
  [field]: 'alice@example.com',   // computed: key is the VALUE of field
};
console.log(user.email); // 'alice@example.com'
// Covered fully in topic 40.
```

---

### 6. Accessing a property that doesn't exist returns `undefined`, not an error

```js
const user = { name: 'Alice' };
console.log(user.age);       // undefined — not an error
console.log(user.age + 1);   // NaN — undefined + 1 = NaN
console.log(user.age.years); // TypeError — can't access property of undefined

// Guard before deep access:
const age = user.age ?? 'unknown'; // 'unknown' if undefined/null
```

---

## Common mistakes

### Mistake 1 — Using dot notation with a variable key

```js
const field = 'name';
const user  = { name: 'Alice' };

// ❌ Wrong — looks for a key literally named 'field'
console.log(user.field);   // undefined

// ✅ Right — bracket notation evaluates the variable
console.log(user[field]);  // 'Alice'
```

---

### Mistake 2 — Thinking assignment copies the object

```js
// ❌ Wrong mental model
const a = { x: 1 };
const b = a;           // "I have a copy now"
b.x = 99;
console.log(a.x);      // 99 — surprise! a was mutated

// ✅ Right — use spread for a shallow copy
const c = { ...a };
c.x = 42;
console.log(a.x);      // 99 — a is safe
```

---

### Mistake 3 — Checking existence with `undefined` comparison

```js
const settings = { darkMode: undefined };

// ❌ Unreliable
if (settings.darkMode === undefined) {
  console.log('key missing'); // prints — but the key IS there
}

// ✅ Use 'in' operator
if (!('darkMode' in settings)) {
  console.log('key missing'); // does NOT print — key exists
}
```

---

### Mistake 4 — Forgetting `delete` doesn't deeply remove

```js
const user = {
  profile: { name: 'Alice', age: 28 }
};

delete user.profile.name; // removes name from nested object — works
delete user.profile;      // removes the profile key entirely — works

// But delete does NOT work on nested values via top-level delete:
delete user; // false — can't delete a variable binding
```

---

## Frequently asked questions

**Q: When should I use dot notation vs bracket notation?**
Default to dot notation — it's cleaner. Use bracket notation only when the key is in a variable, has special characters, or is computed dynamically.

**Q: Can object keys be numbers?**
Yes — but they're converted to strings internally. `obj[1]` is the same as `obj['1']`.

**Q: What's the difference between `null` and an empty object `{}`?**
`null` means "intentionally no value" — `typeof null` is `'object'` (a historical bug in JS). An empty object `{}` is a real object with no properties. They behave very differently: you can add properties to `{}` but accessing any property on `null` throws a `TypeError`.

**Q: How do I copy an object including nested objects (deep copy)?**
Spread (`{ ...obj }`) only copies one level deep. For deep copies:
- Simple data: `JSON.parse(JSON.stringify(obj))`
- Modern (Node 17+ / modern browsers): `structuredClone(obj)`
- Note: both methods fail on functions, `undefined` values, and circular references.

**Q: Can I have duplicate keys in an object?**
No — keys must be unique. If you write the same key twice, the last value wins and JS won't warn you:

```js
const bad = { name: 'Alice', name: 'Bob' }; // no error!
console.log(bad.name); // 'Bob' — last value wins
```

**Q: What happens when I access a property on `undefined` or `null`?**
A `TypeError` is thrown immediately. This is one of the most common runtime errors in JS:

```js
const user = null;
user.name; // TypeError: Cannot read properties of null (reading 'name')

// Guard with optional chaining:
user?.name; // undefined — no error (topic 39)
```

---

## Practice exercises

### Exercise 1 — Easy: Build and manipulate a user object

Create an object called `studentProfile` with these properties:
- `name` — your name
- `grade` — a number (e.g. 10)
- `subjects` — an array of 3 subject strings
- `passed` — boolean `true`

Then:
1. Log the student's name using dot notation
2. Log the second subject using bracket notation on the array
3. Update `grade` to 11
4. Add a new property `school` with any school name
5. Delete the `passed` property
6. Log the final object

```js
// Write your code here
```

---

### Exercise 2 — Medium: Order processing

You receive this order object:

```js
const order = {
  orderId:   'ORD-5521',
  customer:  'Priya Sharma',
  items: [
    { product: 'Keyboard', qty: 1, price: 1200 },
    { product: 'Mouse',    qty: 2, price:  800 },
    { product: 'Mousepad', qty: 1, price:  350 },
  ],
  status:    'pending',
  createdAt: '2026-05-01',
};
```

Write code that:
1. Logs the customer name
2. Logs the product name of the second item
3. Calculates the **total price** (qty × price for each item, summed) — use a loop or array method
4. Adds a `total` property to the order with that calculated value
5. Updates `status` to `'confirmed'`
6. Checks whether `order.discount` exists using the `in` operator — log `true` or `false`
7. Logs the final order object

```js
// Write your code here
```

---

### Exercise 3 — Hard: Config merger

You're building a settings system. You have a `defaultConfig` object and a `userConfig` object. The user config only contains settings the user has explicitly changed.

```js
const defaultConfig = {
  theme:         'light',
  fontSize:      14,
  language:      'en',
  notifications: true,
  autoSave:      false,
  sidebar:       'left',
};

const userConfig = {
  theme:    'dark',
  fontSize: 18,
  autoSave: true,
};
```

Write a function `mergeConfig(defaults, overrides)` that:
1. Creates a merged config where user values override defaults (do NOT mutate either object)
2. Returns the merged config
3. Also returns a `changedKeys` array listing only the keys the user overrode

The function should return an object like:
```js
{
  merged:      { theme: 'dark', fontSize: 18, language: 'en', ... },
  changedKeys: ['theme', 'fontSize', 'autoSave'],
}
```

Test it:
```js
const result = mergeConfig(defaultConfig, userConfig);
console.log(result.merged);
console.log(result.changedKeys);
// Make sure defaultConfig and userConfig are unchanged after the call
```

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// CREATE
const obj = { key: value, key2: value2 };

// READ
obj.key;           // dot notation
obj['key'];        // bracket notation
obj[variable];     // dynamic key via variable

// UPDATE / ADD
obj.key   = newValue;
obj[key]  = newValue;

// DELETE
delete obj.key;

// CHECK EXISTENCE
'key' in obj;                  // true/false (includes inherited)
Object.hasOwn(obj, 'key');     // true/false (own properties only)

// SHORTHAND (ES6)
const name = 'Alice';
const user = { name };         // same as { name: name }

// COPY (shallow)
const copy = { ...obj };

// ITERATE
for (const key in obj) { ... }
Object.keys(obj)               // → array of keys
Object.values(obj)             // → array of values
Object.entries(obj)            // → array of [key, value] pairs

// FREEZE (immutable)
const frozen = Object.freeze(obj);
```

| Operation | Syntax |
|---|---|
| Create | `const obj = { k: v }` |
| Read (dot) | `obj.key` |
| Read (bracket) | `obj['key']` or `obj[variable]` |
| Update/Add | `obj.key = newVal` |
| Delete | `delete obj.key` |
| Check exists | `'key' in obj` |
| Shallow copy | `{ ...obj }` |
| All keys | `Object.keys(obj)` |

---

## Connected topics

- **[35] Object Methods & `this`** — adding functions inside objects; what `this` refers to
- **[36] Object Destructuring** — cleaner syntax for extracting multiple properties at once
- **[37] Spread with Objects** — copying and merging objects with `...`
- **[38] Object.keys / values / entries** — iterating over objects as arrays
- **[39] Optional Chaining `?.`** — safe access for deeply nested objects
