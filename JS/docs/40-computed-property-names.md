# 40 — Computed Property Names

---

## What is this?

**Computed property names** let you use an **expression** as an object key by wrapping it in square brackets `[]` inside an object literal. Instead of hardcoding the key name, the key is calculated at runtime from a variable, function call, or any expression.

```js
const field = 'email';

// Without computed property — key is literally the string 'field'
const obj1 = { field: 'alice@example.com' };
console.log(obj1); // { field: 'alice@example.com' }  ← key is 'field', not 'email'

// With computed property — key is the VALUE of the variable 'field'
const obj2 = { [field]: 'alice@example.com' };
console.log(obj2); // { email: 'alice@example.com' }  ← key is 'email'
```

Think of it like filling out a form — instead of writing the label yourself, you look it up from a reference and write whatever it says.

---

## Why does it matter?

Computed properties are essential whenever you need to build objects with **dynamic keys** — keys that aren't known until runtime. This comes up constantly in form handling, API data transformation, state management, configuration builders, and any code that maps over keys programmatically.

---

## Syntax

```js
const key = 'someKey';

const obj = {
  [key]:              value,          // variable as key
  [expression]:       value,          // any expression
  [`prefix_${key}`]:  value,          // template literal as key
  [fn()]:             value,          // function call result as key
  [obj.prop]:         value,          // property value as key
};
```

---

## How it works — line by line

```js
const field   = 'username';
const prefix  = 'user';
const index   = 3;

const profile = {
  [field]:             'alice92',       // key = 'username'
  [`${prefix}Id`]:     101,             // key = 'userId'
  [`item_${index}`]:   'third item',    // key = 'item_3'
  [field.toUpperCase()]: 'ALICE92',     // key = 'USERNAME'
};

console.log(profile);
// {
//   username:  'alice92',
//   userId:    101,
//   item_3:    'third item',
//   USERNAME:  'ALICE92',
// }
```

JS evaluates the expression inside `[]` first, converts the result to a string (if it's not a Symbol), and uses that as the key.

---

## Example 1 — Dynamic form field update

The most classic use case — updating a specific field in a form state object without knowing which field at write time:

```js
// WITHOUT computed properties — ugly switch/if chain
function updateField(formData, fieldName, value) {
  if (fieldName === 'name') {
    return { ...formData, name: value };
  } else if (fieldName === 'email') {
    return { ...formData, email: value };
  } else if (fieldName === 'age') {
    return { ...formData, age: value };
  }
  // ...grows forever as fields are added
}

// WITH computed properties — one line, handles ANY field
function updateField(formData, fieldName, value) {
  return { ...formData, [fieldName]: value };
}

const form = { name: '', email: '', age: 0, city: '' };

const afterName  = updateField(form, 'name',  'Alice');
const afterEmail = updateField(form, 'email', 'alice@example.com');
const afterAge   = updateField(form, 'age',   28);

console.log(afterName);
// { name: 'Alice', email: '', age: 0, city: '' }
console.log(afterEmail);
// { name: '', email: 'alice@example.com', age: 0, city: '' }
```

This is the **exact pattern used in React** for controlled form inputs:

```js
// React-style input handler
function handleChange(event) {
  const { name, value } = event.target;
  setFormData(prev => ({ ...prev, [name]: value }));
  // [name] is the input's name attribute — computed key!
}
```

---

## Example 2 — Building lookup tables / indexes

```js
const products = [
  { id: 'SKU-01', name: 'Laptop',   price: 75000 },
  { id: 'SKU-02', name: 'Mouse',    price:   800 },
  { id: 'SKU-03', name: 'Keyboard', price:  1200 },
];

// Build a lookup table: id → product (O(1) access instead of O(n) find)
const productById = products.reduce((lookup, product) => ({
  ...lookup,
  [product.id]: product,   // computed key = product's id
}), {});

console.log(productById);
// {
//   'SKU-01': { id: 'SKU-01', name: 'Laptop',   price: 75000 },
//   'SKU-02': { id: 'SKU-02', name: 'Mouse',    price:  800  },
//   'SKU-03': { id: 'SKU-03', name: 'Keyboard', price: 1200  },
// }

// Now lookup is instant:
productById['SKU-02'].name;  // 'Mouse'
productById['SKU-01'].price; // 75000
```

---

## Example 3 — Grouping data with computed keys

```js
const transactions = [
  { id: 1, category: 'food',      amount: 450 },
  { id: 2, category: 'transport', amount: 120 },
  { id: 3, category: 'food',      amount: 320 },
  { id: 4, category: 'shopping',  amount: 1800 },
  { id: 5, category: 'transport', amount:  85 },
  { id: 6, category: 'food',      amount: 210 },
];

// Group by category — category name becomes the key dynamically
const grouped = transactions.reduce((groups, tx) => {
  const existing = groups[tx.category] ?? [];
  return {
    ...groups,
    [tx.category]: [...existing, tx],  // computed key = tx.category
  };
}, {});

console.log(grouped);
// {
//   food:      [{ id:1,... }, { id:3,... }, { id:6,... }],
//   transport: [{ id:2,... }, { id:5,... }],
//   shopping:  [{ id:4,... }],
// }

// Sum by category
const totals = Object.fromEntries(
  Object.entries(grouped).map(([cat, txs]) => [
    cat,
    txs.reduce((sum, tx) => sum + tx.amount, 0),
  ])
);
console.log(totals);
// { food: 980, transport: 205, shopping: 1800 }
```

---

## Example 4 — Prefixed/namespaced keys

```js
// Building CSS variable objects
const theme = 'dark';
const colors = {
  [`--${theme}-bg`]:      '#1a1a2e',
  [`--${theme}-text`]:    '#eaeaea',
  [`--${theme}-accent`]:  '#6c63ff',
};
console.log(colors);
// { '--dark-bg': '#1a1a2e', '--dark-text': '#eaeaea', '--dark-accent': '#6c63ff' }

// Building prefixed API fields
function buildPayload(entityType, fields) {
  return Object.fromEntries(
    Object.entries(fields).map(([key, value]) => [`${entityType}_${key}`, value])
  );
}

buildPayload('user', { name: 'Alice', age: 28 });
// { user_name: 'Alice', user_age: 28 }

buildPayload('product', { name: 'Laptop', price: 75000 });
// { product_name: 'Laptop', product_price: 75000 }
```

---

## Example 5 — Computed keys in class methods and object methods

```js
const PREFIX = 'handle';

const eventHandlers = {
  [`${PREFIX}Click`](event) {
    console.log('clicked', event.target);
  },
  [`${PREFIX}Hover`](event) {
    console.log('hovered', event.target);
  },
  [`${PREFIX}Submit`](event) {
    event.preventDefault();
    console.log('form submitted');
  },
};

eventHandlers.handleClick({ target: 'button' });  // 'clicked button'
eventHandlers.handleHover({ target: 'link' });     // 'hovered link'
```

---

## Example 6 — Dynamic validation schema builder

```js
// Build a validation rules object where field names are computed
function buildValidationSchema(fields) {
  return fields.reduce((schema, { name, required, minLength }) => ({
    ...schema,
    [name]: {
      required:  required ?? false,
      minLength: minLength ?? 0,
    },
  }), {});
}

const schema = buildValidationSchema([
  { name: 'username',  required: true,  minLength: 3 },
  { name: 'email',     required: true,  minLength: 0 },
  { name: 'password',  required: true,  minLength: 8 },
  { name: 'bio',       required: false, minLength: 0 },
]);

console.log(schema);
// {
//   username: { required: true, minLength: 3 },
//   email:    { required: true, minLength: 0 },
//   password: { required: true, minLength: 8 },
//   bio:      { required: false, minLength: 0 },
// }

// Use it:
function validate(data, schema) {
  const errors = {};
  for (const [field, rules] of Object.entries(schema)) {
    if (rules.required && !data[field]) {
      errors[field] = `${field} is required`;          // computed key in errors!
    } else if (data[field]?.length < rules.minLength) {
      errors[field] = `${field} must be at least ${rules.minLength} characters`;
    }
  }
  return errors;
}

console.log(validate({ username: 'al', password: 'abc' }, schema));
// { username: 'username must be at least 3 characters', email: 'email is required', password: 'password must be at least 8 characters' }
```

---

## Example 7 — Symbol as computed property key

Symbols create truly unique keys that can't clash with string keys:

```js
const ID      = Symbol('id');
const VERSION = Symbol('version');

const record = {
  name:    'Config',
  [ID]:      42,            // Symbol key — won't show in for...in or Object.keys
  [VERSION]: '1.0.0',
};

console.log(record[ID]);       // 42
console.log(record[VERSION]);  // '1.0.0'
console.log(record.name);      // 'Config'

// Symbol keys are hidden from normal enumeration
Object.keys(record);           // ['name'] — Symbol keys excluded
JSON.stringify(record);        // '{"name":"Config"}' — Symbols excluded

// Access only via the original Symbol reference
Object.getOwnPropertySymbols(record); // [Symbol(id), Symbol(version)]
```

*(Symbols are covered fully in topic 60.)*

---

## Tricky things you'll encounter in the real world

### 1. The expression is evaluated ONCE at object creation time

```js
let key = 'a';
const obj = { [key]: 1 };  // key is evaluated now — obj = { a: 1 }

key = 'b';
console.log(obj);  // { a: 1 } — changing key later doesn't affect obj
```

The computed key is baked in at the time the object literal is executed.

---

### 2. Keys are always converted to strings (unless Symbol)

```js
const num = 42;
const bool = true;
const arr = [1, 2];

const obj = {
  [num]:  'number key',   // key becomes '42'
  [bool]: 'bool key',     // key becomes 'true'
  [arr]:  'array key',    // key becomes '1,2' (array.toString())
};

console.log(obj);
// { '42': 'number key', 'true': 'bool key', '1,2': 'array key' }

Object.keys(obj); // ['42', 'true', '1,2']
```

Objects as keys also get converted to strings:

```js
const objKey = { id: 1 };
const container = { [objKey]: 'value' };
console.log(container); // { '[object Object]': 'value' }
// All objects stringify to '[object Object]' — use Map for object keys
```

---

### 3. Duplicate computed keys — last one wins, no warning

```js
const key = 'name';
const obj = {
  [key]:  'Alice',
  name:   'Bob',   // same key! last value wins
};
console.log(obj.name); // 'Bob' — 'Alice' is silently overwritten
```

---

### 4. Computed keys work in destructuring too

```js
const field = 'email';

// Extract a dynamic key from an object
const user = { name: 'Alice', email: 'alice@example.com' };
const { [field]: userEmail } = user;   // key is computed, rename to userEmail

console.log(userEmail); // 'alice@example.com'

// This is the destructuring equivalent of user[field]
```

---

### 5. Inside class definitions — computed method names

```js
const action = 'greet';

class User {
  constructor(name) { this.name = name; }

  [`${action}User`]() {           // computed method name
    return `Hello, ${this.name}`;
  }
}

const alice = new User('Alice');
alice.greetUser(); // 'Hello, Alice'
```

---

## Common mistakes

### Mistake 1 — Forgetting brackets and using the variable name as a literal key

```js
const fieldName = 'email';
const value     = 'alice@example.com';

// ❌ Wrong — key is literally 'fieldName', not 'email'
const obj = { fieldName: value };
console.log(obj); // { fieldName: 'alice@example.com' }

// ✅ Correct — brackets make it computed
const obj2 = { [fieldName]: value };
console.log(obj2); // { email: 'alice@example.com' }
```

---

### Mistake 2 — Using an object as a key expecting uniqueness

```js
const key1 = { id: 1 };
const key2 = { id: 2 };

// ❌ Both objects stringify to '[object Object]' — second overwrites first
const obj = {
  [key1]: 'first',
  [key2]: 'second',
};
console.log(obj); // { '[object Object]': 'second' }

// ✅ Use Map for object keys
const map = new Map();
map.set(key1, 'first');
map.set(key2, 'second');
map.get(key1); // 'first'
map.get(key2); // 'second'
```

---

### Mistake 3 — Not renaming in destructuring with computed keys

```js
const key = 'name';
const user = { name: 'Alice' };

// ❌ SyntaxError — computed key in destructuring REQUIRES a rename
const { [key] } = user;

// ✅ Computed key in destructuring always needs ': aliasName'
const { [key]: userName } = user;
console.log(userName); // 'Alice'
```

---

## Frequently asked questions

**Q: Can I use a function call as a computed key?**
Yes — the function is called and its return value (converted to string) becomes the key:

```js
function getKey() { return 'dynamicKey'; }
const obj = { [getKey()]: 42 };
console.log(obj); // { dynamicKey: 42 }
```

**Q: Can computed property names be used in class bodies?**
Yes — for method names. You can't use them for class fields in the same way (class field computed names have limited support). Works cleanly for methods.

**Q: Are computed keys the same as bracket notation for reading?**
Same square bracket syntax, different context:
- **Writing** (object literal): `{ [expr]: value }` — computed property name
- **Reading** (property access): `obj[expr]` — bracket notation access
They both evaluate the expression, just one sets a key and the other reads one.

**Q: What's the difference between `{ [key]: val }` and `{ [key]: key }`?**
`{ [key]: val }` — key name is dynamic (from `key`), value is `val`.
`{ [key]: key }` — key name is dynamic, value is also the contents of `key` (the same string).

**Q: Does computed property naming affect performance?**
No meaningful difference for typical use. Engines optimize it. Only matters in extremely hot loops with millions of object creations.

---

## Practice exercises

### Exercise 1 — Easy: Dynamic object builder

Write a function `buildObject(keys, values)` that takes two arrays and returns an object where each key from `keys` maps to the corresponding value from `values`.

```js
buildObject(['name', 'age', 'city'], ['Alice', 28, 'Mumbai']);
// { name: 'Alice', age: 28, city: 'Mumbai' }

buildObject(['x', 'y', 'z'], [10, 20, 30]);
// { x: 10, y: 20, z: 30 }
```

Then write `buildPrefixedObject(prefix, data)` that takes a prefix string and an object, and returns a new object where every key is prefixed:

```js
buildPrefixedObject('user_', { name: 'Alice', age: 28 });
// { user_name: 'Alice', user_age: 28 }

buildPrefixedObject('db_', { host: 'localhost', port: 5432 });
// { db_host: 'localhost', db_port: 5432 }
```

```js
// Write your code here
```

---

### Exercise 2 — Medium: Generic state updater

You're building a utility for managing application state. Write a function `setState(currentState, updates)` where `updates` is an array of `{ field, value }` objects:

```js
const initialState = {
  username:  '',
  email:     '',
  age:       0,
  isPremium: false,
  theme:     'light',
};

const updates = [
  { field: 'username',  value: 'parsh_dev' },
  { field: 'email',     value: 'parsh@example.com' },
  { field: 'isPremium', value: true },
];

const newState = setState(initialState, updates);
console.log(newState);
// { username: 'parsh_dev', email: 'parsh@example.com', age: 0, isPremium: true, theme: 'light' }

// Original must be unchanged
console.log(initialState.username); // ''
```

The function must:
1. Not mutate `currentState`
2. Apply all updates in order (later updates override earlier ones for the same field)
3. Use computed property names — no if/switch logic

```js
// Write your code here
```

---

### Exercise 3 — Hard: Analytics event tracker

You're building an analytics module. Events come in a stream and you need to build a summary object dynamically.

```js
const eventStream = [
  { type: 'pageView',  page: '/home',    userId: 'U1' },
  { type: 'click',     element: 'btn-1', userId: 'U1' },
  { type: 'pageView',  page: '/about',   userId: 'U2' },
  { type: 'purchase',  itemId: 'SKU-1',  userId: 'U1', amount: 2500 },
  { type: 'pageView',  page: '/home',    userId: 'U3' },
  { type: 'click',     element: 'btn-2', userId: 'U2' },
  { type: 'purchase',  itemId: 'SKU-2',  userId: 'U3', amount: 800 },
  { type: 'pageView',  page: '/home',    userId: 'U2' },
  { type: 'click',     element: 'btn-1', userId: 'U3' },
  { type: 'purchase',  itemId: 'SKU-1',  userId: 'U2', amount: 2500 },
];
```

Write `buildAnalyticsSummary(events)` that returns:

```js
{
  // Count of each event type — keys are dynamic
  eventCounts: {
    pageView: 4,
    click:    3,
    purchase: 3,
  },

  // Count of views per page — keys are the page paths
  pageViews: {
    '/home':  3,
    '/about': 1,
  },

  // Count of clicks per element — keys are element ids
  clicksByElement: {
    'btn-1': 2,
    'btn-2': 1,
  },

  // Total revenue and count per item
  purchasesByItem: {
    'SKU-1': { count: 2, revenue: 5000 },
    'SKU-2': { count: 1, revenue:  800 },
  },

  totalRevenue: 5800,
}
```

Use computed property names throughout. No hardcoded keys for counts/groupings.

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Basic computed key
const key = 'name';
const obj = { [key]: 'Alice' };             // { name: 'Alice' }

// Template literal key
const type = 'user';
const obj2 = { [`${type}Id`]: 101 };        // { userId: 101 }

// Function call as key
const obj3 = { [getKey()]: 'value' };

// In spread + update pattern
const updated = { ...state, [fieldName]: newValue };

// In reduce (building lookup)
arr.reduce((acc, item) => ({ ...acc, [item.id]: item }), {});

// In destructuring (requires alias)
const { [key]: alias } = obj;

// Symbol key
const SYM = Symbol('key');
const obj4 = { [SYM]: 'hidden' };
```

| Use case | Pattern |
|---|---|
| Variable as key | `{ [variable]: value }` |
| Template key | `{ [\`prefix_${v}\`]: value }` |
| Update one field | `{ ...obj, [field]: newVal }` |
| Build lookup | `reduce((acc, item) => ({ ...acc, [item.id]: item }), {})` |
| Dynamic destructure | `const { [key]: alias } = obj` |

---

## Connected topics

- **[34] Object Basics** — property access; `[]` notation is the read equivalent
- **[37] Spread with Objects** — the `{ ...obj, [key]: val }` pattern is used everywhere
- **[36] Object Destructuring** — computed keys work in destructuring too
- **[29] reduce** — most lookup/grouping patterns use reduce + computed keys
- **[60] Symbol** — the only non-string type that can be a valid unique object key
