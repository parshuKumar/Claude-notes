# 31 — some and every

## What is this?

`some` and `every` are array methods that test whether elements in an array satisfy a condition.

- **`some(fn)`** — returns `true` if **at least one** element passes the test. Stops as soon as it finds one.
- **`every(fn)`** — returns `true` if **all** elements pass the test. Stops as soon as it finds one that fails.

Both always return a **boolean** (`true` or `false`). They never return an element, an index, or a new array.

---

## Why does it matter?

Before these methods existed, developers wrote `for` loops with flags:

```js
// old way
let hasAdmin = false;
for (let i = 0; i < users.length; i++) {
  if (users[i].role === 'admin') {
    hasAdmin = true;
    break;
  }
}
```

`some` and `every` replace that pattern with a single readable line. You use them constantly for:

- Form validation (are all fields filled in?)
- Permission checks (does the user have any required role?)
- Stock checks (is every item in the cart available?)
- Feature flags (are all required features enabled?)
- Data quality checks (does any record have a missing field?)

---

## Syntax

```js
array.some(callback)
array.every(callback)

// callback receives: (currentElement, index, array)
```

Both accept the same callback shape as `filter`, `map`, and `forEach`.

---

## How it works — line by line

### `some` — short-circuits on first `true`

```js
const scores = [42, 67, 55, 91, 38];

const hasHighScore = scores.some(score => score >= 90);
// iteration 1: 42 >= 90 → false, keep going
// iteration 2: 67 >= 90 → false, keep going
// iteration 3: 55 >= 90 → false, keep going
// iteration 4: 91 >= 90 → TRUE  ← stops here, returns true
// iteration 5: never runs

console.log(hasHighScore); // true
```

### `every` — short-circuits on first `false`

```js
const ages = [22, 31, 19, 25, 17];

const allAdults = ages.every(age => age >= 18);
// iteration 1: 22 >= 18 → true, keep going
// iteration 2: 31 >= 18 → true, keep going
// iteration 3: 19 >= 18 → true, keep going
// iteration 4: 25 >= 18 → true, keep going
// iteration 5: 17 >= 18 → FALSE ← stops here, returns false

console.log(allAdults); // false
```

---

## Example 1 — basic usage

```js
const numbers = [3, 7, 1, 8, 4, 6];

// some: is there any even number?
const hasEven = numbers.some(n => n % 2 === 0);
console.log(hasEven); // true  (8 passes)

// every: are all numbers positive?
const allPositive = numbers.every(n => n > 0);
console.log(allPositive); // true

// every: are all numbers greater than 5?
const allAboveFive = numbers.every(n => n > 5);
console.log(allAboveFive); // false  (3 fails immediately)

// some: does any number exceed 100?
const anyLarge = numbers.some(n => n > 100);
console.log(anyLarge); // false  (scans all — none pass)
```

---

## Example 2 — real world: form validation

A signup form with multiple fields. Before submitting, check that all required fields are filled and that no field exceeds a length limit.

```js
const formFields = [
  { name: 'username',  value: 'parsh_dev',      required: true  },
  { name: 'email',     value: 'parsh@mail.com',  required: true  },
  { name: 'password',  value: 'secret123',       required: true  },
  { name: 'bio',       value: '',                required: false },
];

// Are all required fields filled in?
const allRequiredFilled = formFields
  .filter(field => field.required)
  .every(field => field.value.trim().length > 0);

console.log(allRequiredFilled); // true

// Does any field exceed 100 characters?
const anyTooLong = formFields.some(field => field.value.length > 100);
console.log(anyTooLong); // false

// Combine into a submit check
function canSubmit(fields) {
  const requiredFilled = fields
    .filter(f => f.required)
    .every(f => f.value.trim() !== '');

  const noFieldTooLong = fields.every(f => f.value.length <= 100);

  return requiredFilled && noFieldTooLong;
}

console.log(canSubmit(formFields)); // true
```

---

## Example 3 — real world: cart stock check

An e-commerce checkout. Before processing payment, verify every item in the cart is in stock and there are no zero-quantity items.

```js
const cart = [
  { productId: 'A1', name: 'Keyboard',   quantity: 1, inStock: true  },
  { productId: 'B2', name: 'Mouse',       quantity: 2, inStock: true  },
  { productId: 'C3', name: 'Monitor',     quantity: 1, inStock: false },
  { productId: 'D4', name: 'USB Hub',     quantity: 0, inStock: true  },
];

const allInStock     = cart.every(item => item.inStock);
const hasZeroQty     = cart.some(item => item.quantity === 0);
const hasOutOfStock  = cart.some(item => !item.inStock);

console.log(allInStock);    // false  (Monitor is out of stock)
console.log(hasZeroQty);    // true   (USB Hub has qty 0)
console.log(hasOutOfStock); // true

function canCheckout(cartItems) {
  if (!cartItems.every(item => item.inStock)) {
    return { ok: false, reason: 'Some items are out of stock' };
  }
  if (cartItems.some(item => item.quantity <= 0)) {
    return { ok: false, reason: 'Cart contains items with invalid quantity' };
  }
  return { ok: true };
}

console.log(canCheckout(cart));
// { ok: false, reason: 'Some items are out of stock' }
```

---

## Example 4 — real world: permission system

A role-based access control (RBAC) system. A user has an array of roles, and a resource requires certain roles to be accessed.

```js
const currentUser = {
  id: 'u_99',
  name: 'Parsh',
  roles: ['editor', 'moderator'],
};

const adminPanel = {
  requiredRoles: ['admin'],
  anyRoleAllowed: false,  // must have ALL required roles
};

const dashboard = {
  requiredRoles: ['editor', 'viewer', 'admin'],
  anyRoleAllowed: true,   // having ANY of these roles is enough
};

function canAccess(user, resource) {
  if (resource.anyRoleAllowed) {
    // user needs at least one matching role
    return resource.requiredRoles.some(role => user.roles.includes(role));
  } else {
    // user must have every required role
    return resource.requiredRoles.every(role => user.roles.includes(role));
  }
}

console.log(canAccess(currentUser, adminPanel));  // false (no 'admin' role)
console.log(canAccess(currentUser, dashboard));   // true  ('editor' matches)
```

---

## Tricky things you'll encounter in the real world

### 1. `every` on an empty array always returns `true` (vacuous truth)

This catches everyone off-guard. If the array is empty, `every` has nothing to disprove, so it returns `true`.

```js
[].every(n => n > 1000);  // true  ← unexpected!
[].some(n => n > 1000);   // false ← correct instinct
```

This matters when the array might be empty — e.g., a cart with no items should not pass an "all items in stock" check.

```js
// WRONG — an empty cart passes checkout
function canCheckout(cart) {
  return cart.every(item => item.inStock);
}
canCheckout([]); // true — you'd charge the user for nothing!

// RIGHT — guard against empty
function canCheckout(cart) {
  return cart.length > 0 && cart.every(item => item.inStock);
}
canCheckout([]); // false
```

---

### 2. They return booleans, not elements — don't confuse with `find` / `filter`

```js
const users = [{ id: 1, active: true }, { id: 2, active: false }];

// WRONG: trying to use some() like find()
const activeUser = users.some(u => u.active);
console.log(activeUser); // true  ← not the user object, just true/false

// RIGHT: use find() when you need the element
const activeUser2 = users.find(u => u.active);
console.log(activeUser2); // { id: 1, active: true }
```

---

### 3. The relationship between `some` and `every`

`some` and `every` are logical opposites through negation:

```js
const arr = [1, 2, 3, 4];

// these two lines always produce the same result:
arr.some(n => n > 2);          // true
!arr.every(n => n <= 2);       // true  ← same thing

// these two lines always produce the same result:
arr.every(n => n > 0);         // true
!arr.some(n => n <= 0);        // true  ← same thing
```

Pick the form that reads most naturally for your use case.

---

### 4. Short-circuit means your callback may not run for every element

This is a feature, but it can cause bugs if your callback has side effects (logging, mutation).

```js
let callCount = 0;

const result = [10, 20, 30, 40].every(n => {
  callCount++;
  console.log(`checking ${n}`);
  return n < 25;
});

// Output:
// checking 10
// checking 20
// checking 30  ← stops here (30 >= 25 is false)
// 40 is never checked

console.log(callCount); // 3, not 4
console.log(result);    // false
```

Never put critical side effects (like sending analytics) inside `some`/`every` callbacks.

---

### 5. Checking for existence vs checking a property — two common patterns

```js
const tags = ['javascript', 'node', 'react'];

// Pattern A: does the array contain a specific value?
const hasReact = tags.some(tag => tag === 'react');
// same as:
const hasReact2 = tags.includes('react');
// use includes() for primitives — it's cleaner

// Pattern B: does any object in the array meet a condition?
const users = [{ name: 'Alice', verified: false }, { name: 'Bob', verified: true }];
const hasVerified = users.some(u => u.verified);
// you MUST use some() here — includes() won't work on objects
```

---

### 6. Using `index` and `array` parameters in the callback

All array methods pass `(element, index, array)` to the callback. This lets you compare elements to neighbours or check position.

```js
const prices = [10, 12, 9, 14, 16];

// Is the array sorted in ascending order?
const isSorted = prices.every((price, index, arr) => {
  if (index === 0) return true;          // skip first (no previous element)
  return price >= arr[index - 1];        // each price >= the one before it
});

console.log(isSorted); // false  (9 < 12 breaks it)
```

---

## Common mistakes

```js
// ❌ Forgetting that every() on [] is true
const valid = [].every(x => x > 0); // true — probably wrong

// ❌ Expecting some/every to return the element
const found = [1, 2, 3].some(n => n === 2); // true, not 2

// ❌ Using some() when you need every()
// "Check that all passwords are strong enough"
const strongEnough = passwords.some(p => p.length >= 8); // WRONG — only one needs to pass
const strongEnough2 = passwords.every(p => p.length >= 8); // RIGHT

// ❌ Putting async code inside some/every
const result = items.every(async item => {
  const data = await fetchItem(item.id); // async — always returns a Promise (truthy)
  return data.inStock;                   // this line is never evaluated synchronously
});
// result is always true — Promises are truthy objects
// Use a manual for...of loop with await for async validation

// ❌ Mutating the original array inside the callback
arr.some(item => {
  arr.push(item * 2); // don't do this — behaviour is unpredictable
  return item > 10;
});
```

---

## Frequently asked questions

**Q: When should I use `some` vs `includes`?**
Use `includes` for simple primitive value checks (`arr.includes('admin')`). Use `some` when you need to test a condition on each element's properties (`arr.some(u => u.role === 'admin')`).

**Q: Can I use `some` or `every` on a string?**
No — they are array methods. To test string characters, convert first: `[...'hello'].some(c => c === 'l')`.

**Q: What's the difference between `some` and `find`?**
`some` returns a boolean. `find` returns the actual element (or `undefined`). Use `some` when you only need to know *whether* something exists, `find` when you need *what* it is.

**Q: Does `some` / `every` work on Sets or Maps?**
No — not directly. Convert first: `[...mySet].some(...)` or `[...myMap.values()].every(...)`.

**Q: Can I stop `every` early and still get `false` without a falsy return?**
The only way to short-circuit `every` is to return a falsy value from the callback. There is no `break` mechanism. That's by design — use a `for...of` loop if you need more control.

---

## Practice exercises

### Exercise 1 — easy

You have this array of products:

```js
const products = [
  { id: 1, name: 'Laptop',   price: 999,  inStock: true,  category: 'electronics' },
  { id: 2, name: 'Notebook', price: 4,    inStock: true,  category: 'stationery'  },
  { id: 3, name: 'Headphones',price: 79,  inStock: false, category: 'electronics' },
  { id: 4, name: 'Pen',      price: 1,    inStock: true,  category: 'stationery'  },
  { id: 5, name: 'Monitor',  price: 350,  inStock: true,  category: 'electronics' },
];
```

Write the following using `some` or `every` (no loops):

1. `isAnyOutOfStock` — `true` if any product is out of stock
2. `allElectronicsExpensive` — `true` if every electronics item costs more than $50
3. `hasAffordableOption` — `true` if any product costs less than $10
4. `allInStock` — `true` if every product is in stock (guard against empty array too)

---

### Exercise 2 — medium

You are building a multi-step wizard form. Each step has fields, and each field has a value and validation rules.

```js
const steps = [
  {
    stepName: 'Personal Info',
    fields: [
      { name: 'firstName', value: 'Parsh',         required: true,  maxLength: 50  },
      { name: 'lastName',  value: 'Shah',           required: true,  maxLength: 50  },
      { name: 'nickname',  value: '',               required: false, maxLength: 20  },
    ],
  },
  {
    stepName: 'Account Details',
    fields: [
      { name: 'username', value: 'parsh_dev',       required: true,  maxLength: 30  },
      { name: 'password', value: 'abc',             required: true,  maxLength: 100 },
      { name: 'confirm',  value: 'abc',             required: true,  maxLength: 100 },
    ],
  },
  {
    stepName: 'Preferences',
    fields: [
      { name: 'theme',    value: 'dark',            required: false, maxLength: 20  },
      { name: 'language', value: '',                required: false, maxLength: 10  },
    ],
  },
];
```

Write these functions using `some` and `every`:

1. `isStepComplete(step)` — returns `true` if all required fields in the step have a non-empty trimmed value and no field exceeds its maxLength
2. `isFormReady(steps)` — returns `true` if every step is complete
3. `hasAnyError(steps)` — returns `true` if any field in any step is required, non-empty, but exceeds its maxLength (a "too long" error)

---

### Exercise 3 — hard

Build a `Validator` class that validates an array of data objects against a schema. The schema defines rules for each field.

```js
const userSchema = {
  fields: {
    username: { required: true, minLength: 3, maxLength: 20, pattern: /^[a-z0-9_]+$/ },
    email:    { required: true, pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ },
    age:      { required: false, min: 13, max: 120 },
  },
};

const users = [
  { username: 'parsh_dev', email: 'parsh@mail.com', age: 22 },
  { username: 'ab',        email: 'bad-email',      age: 17 },
  { username: 'UPPERCASE', email: 'ok@ok.com'                },
  { username: 'valid_one', email: 'also@valid.com', age: 30 },
];
```

Implement:

1. `validateRecord(record, schema)` — returns `true` if the record passes all rules in the schema
2. `allValid(records, schema)` — returns `true` if every record is valid
3. `anyInvalid(records, schema)` — returns `true` if any record fails
4. `getInvalidRecords(records, schema)` — returns the array of records that fail (use `filter` + `some`/`every`)

---

## Quick reference — cheat sheet

```js
// ── some ────────────────────────────────────────────────
arr.some(fn)           // true if ANY element passes fn
                       // false if NONE pass (or array is empty)
                       // short-circuits on first true

// ── every ───────────────────────────────────────────────
arr.every(fn)          // true if ALL elements pass fn
                       // false if ANY element fails
                       // short-circuits on first false
                       // CAUTION: [].every(anything) → true

// ── choosing the right method ───────────────────────────
// Need a boolean?              → some / every
// Need the element?            → find
// Need all matching elements?  → filter
// Simple value existence?      → includes  (primitives only)

// ── logical equivalence ─────────────────────────────────
arr.some(fn)      ===  !arr.every(x => !fn(x))
arr.every(fn)     ===  !arr.some(x => !fn(x))

// ── empty array behaviour ───────────────────────────────
[].some(fn)       // false  (nothing passes)
[].every(fn)      // true   (nothing fails — vacuous truth)

// ── common patterns ─────────────────────────────────────
// any role match
roles.some(role => user.roles.includes(role))

// all required fields filled
fields.filter(f => f.required).every(f => f.value.trim() !== '')

// array is sorted ascending
arr.every((v, i, a) => i === 0 || v >= a[i - 1])

// guard empty array in every()
arr.length > 0 && arr.every(fn)
```

---

## Connected topics

| Topic | Why it connects |
|---|---|
| [28-filter.md](28-filter.md) | `filter` uses the same callback shape; `some` is like a boolean version of `filter` |
| [30-find-and-findindex.md](30-find-and-findindex.md) | `find` returns the element; `some` returns whether it exists |
| [27-map.md](27-map.md) | Often chained before `some`/`every` to transform values first |
| [29-reduce.md](29-reduce.md) | `reduce` can replicate `some`/`every` but is overkill for boolean checks |
| [32-spread-arrays.md](32-spread-arrays.md) | Spread used to convert Sets/Maps before applying `some`/`every` |
