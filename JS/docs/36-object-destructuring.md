# 36 — Object Destructuring

---

## What is this?

**Object destructuring** is a syntax that lets you unpack properties from an object into individual variables in a single line. Think of it like unpacking a delivery box — instead of reaching in one item at a time, you pull out everything you need in one move.

```js
// Without destructuring — repetitive
const user = { name: 'Alice', age: 28, city: 'Mumbai' };
const name = user.name;
const age  = user.age;
const city = user.city;

// With destructuring — clean
const { name, age, city } = user;
console.log(name, age, city); // 'Alice' 28 'Mumbai'
```

---

## Why does it matter?

You'll destructure objects constantly — pulling data from API responses, function parameters, imported modules, React props, and more. It removes repetitive `obj.property` access and makes code significantly cleaner and more readable.

---

## Syntax

```js
const { key1, key2 } = sourceObject;              // basic
const { key1: newName } = sourceObject;           // rename
const { key1 = defaultVal } = sourceObject;       // default value
const { key1: newName = defaultVal } = sourceObject; // rename + default
const { outer: { inner } } = sourceObject;        // nested destructuring
const { key1, ...rest } = sourceObject;           // rest in destructuring
```

---

## How it works — line by line

```js
const product = {
  id:    'SKU-42',
  name:  'Laptop',
  price: 75000,
  brand: 'Dell',
};

const { name, price } = product;
// JS looks for keys 'name' and 'price' in 'product'
// Creates variables 'name' and 'price' with those values
// Keys that don't match are simply ignored

console.log(name);  // 'Laptop'
console.log(price); // 75000
// 'id' and 'brand' are untouched — still on the object
```

---

## Example 1 — Basic extraction

```js
const order = {
  orderId:  'ORD-881',
  customer: 'Parsh',
  total:    4500,
  status:   'shipped',
  paid:     true,
};

// Extract what you need
const { customer, total, status } = order;

console.log(customer); // 'Parsh'
console.log(total);    // 4500
console.log(status);   // 'shipped'
// orderId and paid are ignored — they stay on the object
```

---

## Example 2 — Renaming variables

Sometimes the key name clashes with an existing variable, or the name isn't descriptive enough in context. Use `:` to rename:

```js
const apiResponse = {
  usr_nm: 'alice92',
  pwd:    'hashed_password',
  e_mail: 'alice@example.com',
  tok:    'eyJhbGciOiJIUzI1NiJ9...',
};

// Extract AND rename in one step
const {
  usr_nm: username,
  e_mail: email,
  tok:    authToken,
} = apiResponse;

console.log(username);  // 'alice92'
console.log(email);     // 'alice@example.com'
console.log(authToken); // 'eyJhbGciOiJIUzI1NiJ9...'
// The original keys (usr_nm etc.) are not created as variables
```

---

## Example 3 — Default values

If a key doesn't exist on the object (is `undefined`), you can provide a fallback:

```js
const userSettings = {
  theme:    'dark',
  language: 'en',
  // fontSize is missing
  // notifications is missing
};

const {
  theme,
  language,
  fontSize     = 16,    // key missing → use default 16
  notifications = true, // key missing → use default true
} = userSettings;

console.log(theme);         // 'dark'      — from object
console.log language);      // 'en'        — from object
console.log(fontSize);      // 16          — default used
console.log(notifications); // true        — default used
```

**Important**: defaults only kick in when the value is `undefined`, NOT when it's `null`, `0`, `false`, or `''`:

```js
const config = { timeout: null, retries: 0, debug: false };

const { timeout = 5000, retries = 3, debug = true } = config;

console.log(timeout); // null  — null is NOT undefined, default ignored
console.log(retries); // 0     — 0 is NOT undefined, default ignored
console.log(debug);   // false — false is NOT undefined, default ignored
```

---

## Example 4 — Rename + default combined

```js
const serverConfig = {
  host: 'localhost',
  // port is missing
};

const {
  host: serverHost,
  port: serverPort = 3000,  // rename 'port' to 'serverPort', default 3000
} = serverConfig;

console.log(serverHost); // 'localhost'
console.log(serverPort); // 3000 — default because 'port' is missing
```

---

## Example 5 — Nested object destructuring

```js
const employee = {
  id:   'EMP-12',
  name: 'Priya',
  role: 'Engineer',
  address: {
    city:    'Bangalore',
    pincode: '560001',
    country: 'India',
  },
  salary: {
    base:    90000,
    bonus:   15000,
    currency: 'INR',
  },
};

// Destructure nested objects
const {
  name,
  address: { city, country },        // destructure inside 'address'
  salary:  { base, bonus: bonusPay }, // destructure inside 'salary', rename bonus
} = employee;

console.log(name);     // 'Priya'
console.log(city);     // 'Bangalore'
console.log(country);  // 'India'
console.log(base);     // 90000
console.log(bonusPay); // 15000

// NOTE: 'address' and 'salary' are NOT created as variables here
// Only the nested keys you specified are extracted
console.log(typeof address); // 'undefined' — not a variable
```

---

## Example 6 — Rest in destructuring

Use `...rest` to collect remaining properties into a new object:

```js
const user = {
  id:       101,
  name:     'Alice',
  email:    'alice@example.com',
  password: 'hashed_secret',
  role:     'admin',
};

// Extract sensitive fields, collect the rest
const { password, id, ...publicUser } = user;

console.log(password);   // 'hashed_secret' — extracted (to discard)
console.log(id);         // 101
console.log(publicUser); // { name: 'Alice', email: 'alice@example.com', role: 'admin' }
// publicUser is safe to send to the frontend — no password
```

---

## Example 7 — Destructuring in function parameters

This is one of the most common uses. Instead of receiving a whole object and drilling into it, destructure directly in the parameter list:

```js
// Without destructuring — verbose
function displayProduct(product) {
  console.log(`${product.name} — ₹${product.price} (${product.brand})`);
}

// With destructuring — clean
function displayProduct({ name, price, brand }) {
  console.log(`${name} — ₹${price} (${brand})`);
}

const laptop = { name: 'MacBook', price: 120000, brand: 'Apple', stock: 5 };
displayProduct(laptop); // 'MacBook — ₹120000 (Apple)'  — 'stock' is ignored

// With defaults in parameters
function createUser({ name, role = 'viewer', active = true } = {}) {
  return { name, role, active };
}

createUser({ name: 'Bob' });
// { name: 'Bob', role: 'viewer', active: true }

createUser({ name: 'Admin', role: 'admin' });
// { name: 'Admin', role: 'admin', active: true }

createUser(); // The '= {}' default makes the whole param optional — no error
// { name: undefined, role: 'viewer', active: true }
```

---

## Example 8 — Destructuring in loops

```js
const products = [
  { id: 1, name: 'Keyboard', price: 1200, inStock: true  },
  { id: 2, name: 'Mouse',    price:  800, inStock: false },
  { id: 3, name: 'Monitor',  price: 12000, inStock: true  },
];

// Destructure each object as you loop
for (const { id, name, price, inStock } of products) {
  if (inStock) {
    console.log(`#${id} ${name} — ₹${price}`);
  }
}
// #1 Keyboard — ₹1200
// #3 Monitor — ₹12000

// With array methods
products
  .filter(({ inStock }) => inStock)
  .forEach(({ name, price }) => console.log(`${name}: ₹${price}`));
```

---

## Example 9 — Swapping variables with destructuring

```js
let a = 10;
let b = 20;

// Without destructuring — needs a temp variable
let temp = a;
a = b;
b = temp;

// With destructuring — clean swap, no temp needed
[a, b] = [b, a];  // array destructuring trick for swapping

console.log(a); // 20
console.log(b); // 10
```

---

## Tricky things you'll encounter in the real world

### 1. Destructuring does NOT remove properties from the original object

```js
const config = { host: 'localhost', port: 3000, debug: true };
const { debug } = config;

console.log(debug);        // true — extracted into a variable
console.log(config.debug); // true — still on the original object
console.log(config);       // { host: 'localhost', port: 3000, debug: true } — unchanged
```

Destructuring is a **read operation** — it copies values, doesn't move them.

---

### 2. Destructuring `null` or `undefined` throws a TypeError

```js
const data = null;
const { name } = data; // TypeError: Cannot destructure property 'name' of null

// Guard with default:
const { name } = data ?? {}; // name = undefined — no error
const { name = 'Guest' } = data ?? {}; // name = 'Guest'
```

---

### 3. Nested destructuring fails silently if intermediate key is missing

```js
const user = { name: 'Alice' }; // no 'address' key

// ❌ This throws — trying to destructure 'undefined'
const { address: { city } } = user;
// TypeError: Cannot destructure property 'city' of undefined

// ✅ Provide a default for the intermediate object
const { address: { city } = {} } = user;
console.log(city); // undefined — no error
```

---

### 4. Variable name collision — you MUST rename

```js
const name = 'Bob'; // already declared

const user = { name: 'Alice', age: 28 };

// ❌ SyntaxError — 'name' is already declared with const
const { name, age } = user;

// ✅ Rename to avoid collision
const { name: userName, age } = user;
console.log(userName); // 'Alice'
console.log(name);     // 'Bob' — untouched
```

---

### 5. Destructuring in assignment (without `const`/`let`) needs parentheses

```js
let x, y;

// ❌ SyntaxError — JS interprets { as a block statement
{ x, y } = { x: 1, y: 2 };

// ✅ Wrap in parentheses — forces it to be an expression
({ x, y } = { x: 1, y: 2 });
console.log(x, y); // 1 2
```

---

### 6. Only `undefined` triggers default values — not `null`, `false`, or `0`

```js
const { a = 10 } = { a: null  }; // a = null  (null ≠ undefined)
const { b = 10 } = { b: false }; // b = false (false ≠ undefined)
const { c = 10 } = { c: 0     }; // c = 0     (0 ≠ undefined)
const { d = 10 } = { d: undefined }; // d = 10  (undefined → use default)
const { e = 10 } = {};               // e = 10  (missing → undefined → use default)
```

---

## Common mistakes

### Mistake 1 — Forgetting the syntax when renaming (key: variable, NOT variable: key)

```js
const user = { name: 'Alice' };

// ❌ Wrong direction — tries to rename 'myName' from object (key that doesn't exist)
const { myName: name } = user; // myName = undefined

// ✅ Correct — key on the LEFT, variable name on the RIGHT
const { name: myName } = user; // myName = 'Alice'
```

---

### Mistake 2 — Assuming destructuring modifies the original object

```js
const settings = { theme: 'dark', lang: 'en' };
const { theme } = settings;

theme = 'light'; // this only changes the local variable 'theme'
console.log(settings.theme); // 'dark' — original unchanged

// To update the object:
settings.theme = 'light'; // direct property assignment
```

---

### Mistake 3 — Deeply nesting destructuring unnecessarily

```js
// ❌ Hard to read at a glance
const { a: { b: { c: { d } } } } = deepObject;

// ✅ Break it into steps — clearer
const { a } = deepObject;
const { b } = a;
const { c } = b;
const { d } = c;

// Or use optional chaining for safe access (topic 39)
const d = deepObject?.a?.b?.c?.d;
```

---

## Frequently asked questions

**Q: What happens if I destructure a key that doesn't exist?**
You get `undefined` — same as accessing `obj.missingKey`. No error unless you then try to access a property on that `undefined` value.

```js
const { missing } = { name: 'Alice' };
console.log(missing); // undefined
```

**Q: Can I destructure an array and an object at the same time?**
Yes — nest the patterns:

```js
const data = { users: ['Alice', 'Bob', 'Carol'] };
const { users: [first, second] } = data;
console.log(first);  // 'Alice'
console.log(second); // 'Bob'
```

**Q: Does destructuring work with `Map` or `Set`?**
No — destructuring uses object key lookup, not iteration. For Maps and Sets, use their own methods (`.get()`, `for...of`, `Array.from()`).

**Q: Is there a performance cost to destructuring?**
No meaningful cost. It compiles to simple property lookups — identical to `obj.key`.

**Q: Can I use destructuring with `let` and `var` too?**
Yes — `const`, `let`, and `var` all support destructuring. Use `const` by default unless you need to reassign.

**Q: What's the `= {}` default at the end of a function parameter?**
It makes the entire parameter optional. Without it, calling the function with no argument would try to destructure `undefined` and throw a `TypeError`.

```js
function fn({ name } = {}) { return name; }
fn();             // undefined — no error
fn({ name: 'A' }); // 'A'
```

---

## Practice exercises

### Exercise 1 — Easy: Extract and use API data

You receive this object from an API:

```js
const movieData = {
  title:       'Interstellar',
  director:    'Christopher Nolan',
  year:        2014,
  rating:      8.6,
  genre:       ['Sci-Fi', 'Drama', 'Adventure'],
  streaming:   'Netflix',
  subtitles:   undefined,
  description: 'A team of explorers travel through a wormhole in space.',
};
```

Using **one destructuring statement**:
1. Extract `title`, `director`, `year`, `rating`
2. Rename `streaming` to `platform`
3. Set a default for `subtitles` of `'Not available'`
4. Extract just the first genre into a variable called `primaryGenre` (destructure the array inside the object)

Then log all five variables.

```js
// Write your code here
```

---

### Exercise 2 — Medium: User profile formatter

You receive this nested user object:

```js
const userProfile = {
  userId:   'USR-4421',
  personal: {
    firstName: 'Priya',
    lastName:  'Sharma',
    age:       26,
  },
  contact: {
    email:  'priya@example.com',
    phone:  '+91-9876543210',
    // address is missing
  },
  preferences: {
    theme:         'dark',
    notifications: false,
    language:      'hi',
  },
};
```

Write a function `formatUser(profile)` that:
1. Uses destructuring in the **function parameters** to extract:
   - `userId`
   - `firstName` and `lastName` from `personal`
   - `email` from `contact`
   - `theme` from `preferences`, defaulting to `'light'`
   - `address` from `contact`, defaulting to `'Address not set'`
2. Returns an object like:
   ```js
   {
     id:       'USR-4421',
     fullName: 'Priya Sharma',
     email:    'priya@example.com',
     theme:    'dark',
     address:  'Address not set',
   }
   ```

```js
// Write your code here
```

---

### Exercise 3 — Hard: Order pipeline with destructuring

You receive an array of raw orders from a database:

```js
const rawOrders = [
  {
    order_id:    'ORD-001',
    cust_name:   'Alice',
    items:       [{ sku: 'A1', qty: 2, unit_price: 500 }, { sku: 'B2', qty: 1, unit_price: 1200 }],
    status_code: 1,
    meta:        { source: 'web', coupon: 'SAVE10' },
  },
  {
    order_id:    'ORD-002',
    cust_name:   'Bob',
    items:       [{ sku: 'C3', qty: 3, unit_price: 300 }],
    status_code: 2,
    meta:        { source: 'app', coupon: null },
  },
  {
    order_id:    'ORD-003',
    cust_name:   'Carol',
    items:       [{ sku: 'A1', qty: 1, unit_price: 500 }],
    status_code: 1,
    meta:        { source: 'web' }, // no coupon key
  },
];
```

Write a function `processOrders(orders)` that:
1. Maps each raw order into a clean format using destructuring
2. Status codes: `1` → `'pending'`, `2` → `'processing'`, anything else → `'unknown'`
3. Calculates `total` — sum of `qty × unit_price` for each item
4. Extracts `coupon` from `meta`, defaulting to `'none'`
5. Returns an array of clean order objects:

```js
[
  {
    id:       'ORD-001',
    customer: 'Alice',
    total:    2200,
    status:   'pending',
    coupon:   'SAVE10',
    source:   'web',
  },
  // ...
]
```

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Basic
const { a, b } = obj;

// Rename
const { a: myA } = obj;              // variable is 'myA', not 'a'

// Default
const { a = 10 } = obj;             // use 10 if a is undefined

// Rename + default
const { a: myA = 10 } = obj;

// Nested
const { outer: { inner } } = obj;

// Nested with default on intermediate
const { outer: { inner } = {} } = obj; // safe if 'outer' is missing

// Rest
const { a, b, ...rest } = obj;      // rest gets all remaining keys

// In function params
function fn({ name, age = 18 } = {}) { ... }

// In for...of
for (const { id, name } of arrayOfObjects) { ... }

// Assignment (existing variables) — needs parens
let x;
({ x } = obj);
```

| Pattern | Example | Result |
|---|---|---|
| Basic | `const { name } = user` | `name = user.name` |
| Rename | `const { name: n } = user` | `n = user.name` |
| Default | `const { age = 18 } = user` | `age = 18` if missing |
| Nested | `const { a: { b } } = obj` | `b = obj.a.b` |
| Rest | `const { a, ...r } = obj` | `r = obj without a` |

---

## Connected topics

- **[33] Array Destructuring** — same concept, applied to arrays
- **[35] Object Methods & `this`** — destructuring function params that are objects
- **[37] Spread with Objects** — the counterpart to rest in destructuring (`...`)
- **[39] Optional Chaining `?.`** — safer alternative for deeply nested access
- **[16] Parameters & Arguments** — destructuring directly in function signatures
