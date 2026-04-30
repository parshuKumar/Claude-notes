# 37 — Spread with Objects

---

## What is this?

The **spread operator** (`...`) used inside an object literal `{}` copies all enumerable own properties from one object into another. Think of it like photocopying a form — you get a new copy with all the same fields filled in, and then you can add or change fields on the copy without affecting the original.

```js
const original = { name: 'Alice', age: 28 };
const copy      = { ...original };

copy.age = 99;
console.log(original.age); // 28 — original untouched
console.log(copy.age);     // 99
```

---

## Why does it matter?

Spread with objects is how modern JavaScript handles **immutable updates** — creating a modified copy of an object without mutating the original. This pattern is everywhere: Redux state updates, React props, config merging, API data transformation. Mastering it is non-negotiable.

---

## Syntax

```js
const copy   = { ...sourceObj };                      // shallow copy
const merged = { ...obj1, ...obj2 };                  // merge (obj2 wins on conflicts)
const updated = { ...obj, key: newValue };            // copy + override one field
const extended = { ...obj, newKey: newValue };        // copy + add new field
const { unwanted, ...rest } = obj;                    // remove a key (destructuring + rest)
```

---

## How it works — line by line

```js
const defaults = { theme: 'light', lang: 'en', fontSize: 14 };
const userPrefs = { theme: 'dark', fontSize: 18 };

const finalConfig = { ...defaults, ...userPrefs };
// Step 1: spread defaults → { theme: 'light', lang: 'en', fontSize: 14 }
// Step 2: spread userPrefs → theme and fontSize are overwritten, lang stays
// Result: { theme: 'dark', lang: 'en', fontSize: 18 }

console.log(finalConfig);
// { theme: 'dark', lang: 'en', fontSize: 18 }
```

When the same key appears more than once, **the last one wins**.

---

## Example 1 — Shallow copy

```js
const user = {
  name:    'Parsh',
  age:     22,
  country: 'India',
};

const userCopy = { ...user };

userCopy.age = 30;

console.log(user.age);     // 22 — original unchanged
console.log(userCopy.age); // 30
console.log(userCopy);     // { name: 'Parsh', age: 30, country: 'India' }

// Compare: WITHOUT spread — both point to same object
const badCopy = user;
badCopy.age   = 99;
console.log(user.age); // 99 — original is mutated!
```

---

## Example 2 — Merging objects (later keys override earlier ones)

```js
const defaultConfig = {
  theme:         'light',
  language:      'en',
  fontSize:      14,
  notifications: true,
  autoSave:      false,
};

const userConfig = {
  theme:    'dark',    // overrides default
  fontSize: 18,        // overrides default
};

const activeConfig = { ...defaultConfig, ...userConfig };

console.log(activeConfig);
// {
//   theme:         'dark',   ← from userConfig (overrode)
//   language:      'en',     ← from defaultConfig (kept)
//   fontSize:      18,       ← from userConfig (overrode)
//   notifications: true,     ← from defaultConfig (kept)
//   autoSave:      false,    ← from defaultConfig (kept)
// }

// Originals untouched
console.log(defaultConfig.theme); // 'light'
console.log(userConfig.language); // undefined
```

---

## Example 3 — Immutable update (the most common real-world pattern)

Instead of mutating an object, create a new one with the change applied:

```js
const product = {
  id:      'SKU-42',
  name:    'Laptop',
  price:   75000,
  inStock: true,
  rating:  4.5,
};

// Apply a price reduction — DON'T mutate original
const discountedProduct = { ...product, price: 60000, onSale: true };

console.log(product.price);           // 75000 — unchanged
console.log(discountedProduct.price); // 60000
console.log(discountedProduct.onSale);// true — new key added
console.log(discountedProduct);
// {
//   id: 'SKU-42', name: 'Laptop', price: 60000,
//   inStock: true, rating: 4.5, onSale: true
// }
```

This is the pattern used in **Redux reducers** and **React state updates**:

```js
// React-style state update
const currentState = { user: 'Alice', score: 80, lives: 3 };

// Player earns points — don't mutate, return new state
function addScore(state, points) {
  return { ...state, score: state.score + points };
}

const newState = addScore(currentState, 20);
console.log(currentState.score); // 80  — unchanged
console.log(newState.score);     // 100 — new state
```

---

## Example 4 — Removing a property (spread + rest destructuring)

You can't "spread minus a key" directly, but combining rest destructuring with spread gives you a clean way to omit a property:

```js
const user = {
  id:       101,
  name:     'Alice',
  email:    'alice@example.com',
  password: 'hashed_secret',
  role:     'admin',
};

// Remove 'password' — destructure it out, spread the rest
const { password, ...safeUser } = user;

console.log(safeUser);
// { id: 101, name: 'Alice', email: 'alice@example.com', role: 'admin' }
// password is gone from safeUser — safe to expose to frontend

// Remove multiple keys
const { password: _pw, id: _id, ...publicProfile } = user;
console.log(publicProfile);
// { name: 'Alice', email: 'alice@example.com', role: 'admin' }
```

---

## Example 5 — Updating a nested object (the shallow copy trap and how to fix it)

```js
const state = {
  user: {
    name:  'Alice',
    score: 50,
  },
  level: 3,
};

// ❌ WRONG — only the top level is copied, 'user' is still the same reference
const badUpdate = { ...state, level: 4 };
badUpdate.user.score = 999; // mutates state.user.score too!
console.log(state.user.score); // 999 — oops

// ✅ RIGHT — spread the nested object too
const goodUpdate = {
  ...state,
  level: 4,
  user: { ...state.user, score: 100 }, // spread the nested object
};

console.log(state.user.score);       // 50    — original untouched
console.log(goodUpdate.user.score);  // 100
console.log(goodUpdate.level);       // 4
```

**Rule**: For every level of nesting you want to update immutably, you need another spread.

---

## Example 6 — Spread with arrays of objects (batch update pattern)

```js
const todos = [
  { id: 1, text: 'Buy groceries', done: false },
  { id: 2, text: 'Write code',    done: false },
  { id: 3, text: 'Go for a walk', done: true  },
];

// Toggle todo with id=2 — immutably
function toggleTodo(list, targetId) {
  return list.map(todo =>
    todo.id === targetId
      ? { ...todo, done: !todo.done }  // spread + override 'done'
      : todo                           // return unchanged
  );
}

const updated = toggleTodo(todos, 2);

console.log(todos[1].done);   // false — original unchanged
console.log(updated[1].done); // true  — new copy updated
```

---

## Example 7 — Merging with computed property names

```js
const field = 'email';
const value = 'alice@example.com';

const user = {
  name: 'Alice',
  [field]: value,  // computed key — covered fully in topic 40
};

// Spread into another object
const extended = { ...user, verified: true };
console.log(extended);
// { name: 'Alice', email: 'alice@example.com', verified: true }
```

---

## Example 8 — `Object.assign` vs spread

Both copy properties, but there are key differences:

```js
const target  = { a: 1 };
const source1 = { b: 2 };
const source2 = { c: 3 };

// Object.assign — mutates the FIRST argument (target)
Object.assign(target, source1, source2);
console.log(target); // { a: 1, b: 2, c: 3 } — target was mutated

// Spread — always creates a NEW object, never mutates
const result = { ...source1, ...source2 };
console.log(result); // { b: 2, c: 3 } — new object, source1/source2 untouched

// Object.assign to copy without mutating — needs empty first arg
const copy = Object.assign({}, target, source1);

// Summary: prefer spread in modern code; Object.assign is older style
```

| | `Object.assign(target, ...sources)` | `{ ...sources }` |
|---|---|---|
| Mutates | Yes — the first argument | No — always new object |
| Invokes setters | Yes | No |
| Multiple sources | Yes | Yes |
| Readable | Moderate | High |

---

## Tricky things you'll encounter in the real world

### 1. Spread is SHALLOW — nested objects are still shared references

```js
const original = {
  name:    'Alice',
  address: { city: 'Mumbai', pin: '400001' },
};

const copy = { ...original };

copy.name = 'Bob';               // ✅ Safe — primitive, copied by value
copy.address.city = 'Delhi';     // ❌ Mutates original.address too!

console.log(original.name);         // 'Alice' — safe
console.log(original.address.city); // 'Delhi' — mutated!

// Fix: spread the nested object too
const deeperCopy = { ...original, address: { ...original.address } };
deeperCopy.address.city = 'Delhi';
console.log(original.address.city); // 'Mumbai' — now safe
```

---

### 2. Order matters — last spread wins

```js
const base     = { x: 1, y: 2 };
const override = { y: 99, z: 3 };

const a = { ...base, ...override };     // { x: 1, y: 99, z: 3 } — override wins
const b = { ...override, ...base };     // { y: 2, z: 3, x: 1 }  — base wins

// Hardcoded value vs spread — last position wins
const c = { ...base, x: 100 };         // x = 100 — hardcoded wins
const d = { x: 100, ...base };         // x = 1   — spread wins
```

---

### 3. Spreading `null` or `undefined` is safe (unlike arrays)

```js
const extra = null;
const obj   = { a: 1, ...extra };   // { a: 1 } — null/undefined are silently ignored
const obj2  = { a: 1, ...undefined }; // { a: 1 } — same

// This is useful for conditionally adding properties:
const isAdmin = true;
const user = {
  name: 'Alice',
  ...(isAdmin && { role: 'admin', permissions: ['read', 'write'] }),
};
console.log(user);
// { name: 'Alice', role: 'admin', permissions: ['read', 'write'] }

const isAdmin2 = false;
const user2 = {
  name: 'Bob',
  ...(isAdmin2 && { role: 'admin', permissions: ['read', 'write'] }),
};
console.log(user2);
// { name: 'Bob' } — isAdmin2 is false, so false is spread (silently ignored)
```

---

### 4. Spreading an array into an object gives index-keyed properties

```js
const arr = ['a', 'b', 'c'];
const obj = { ...arr };
console.log(obj); // { '0': 'a', '1': 'b', '2': 'c' }
// Rarely useful — usually a mistake. Use [...arr] for array copying.
```

---

### 5. Non-enumerable and prototype properties are NOT copied

```js
function User(name) {
  this.name = name;
}
User.prototype.greet = function() { return `Hi, ${this.name}`; };

const alice = new User('Alice');
const copy  = { ...alice };

console.log(copy.name);    // 'Alice' — own enumerable property: copied
console.log(copy.greet);   // undefined — prototype method: NOT copied
```

---

### 6. Spread does NOT trigger getters — it reads the current value

```js
const source = {
  get expensiveValue() {
    console.log('getter called');
    return 42;
  },
};

const copy = { ...source }; // 'getter called' — getter IS invoked when spreading
console.log(copy.expensiveValue); // 42 — copy has the VALUE, not the getter
// copy.expensiveValue is now a plain data property, not a getter
```

---

## Common mistakes

### Mistake 1 — Thinking spread makes a deep copy

```js
const user = { profile: { name: 'Alice' } };

// ❌ Thinking this is a full independent copy
const copy = { ...user };
copy.profile.name = 'Hacked';
console.log(user.profile.name); // 'Hacked' — shallow copy! nested still shared

// ✅ Spread at every level you need to change
const safeCopy = { ...user, profile: { ...user.profile } };
safeCopy.profile.name = 'Bob';
console.log(user.profile.name);     // 'Alice' — safe now
```

---

### Mistake 2 — Wrong order when overriding a specific key

```js
const user = { name: 'Alice', role: 'viewer' };
const overrides = { role: 'admin' };

// ❌ base spreads AFTER override — base.role overwrites the admin role
const wrong = { ...overrides, ...user }; // { role: 'viewer', name: 'Alice' }

// ✅ base spreads FIRST, overrides spread LAST — overrides win
const right = { ...user, ...overrides }; // { name: 'Alice', role: 'admin' }
```

---

### Mistake 3 — Mutating instead of spreading for updates

```js
const state = { count: 0, label: 'Counter' };

// ❌ Mutates original — breaks immutability
function increment(s) {
  s.count += 1;
  return s;
}

// ✅ Returns a new object — original stays pure
function increment(s) {
  return { ...s, count: s.count + 1 };
}

const newState = increment(state);
console.log(state.count);    // 0 — unchanged
console.log(newState.count); // 1
```

---

## Frequently asked questions

**Q: Is `{ ...obj }` the same as `Object.assign({}, obj)`?**
For plain objects, yes — both create a shallow copy of own enumerable properties. Spread is the modern preferred syntax. `Object.assign` invokes setters on the target object; spread does not.

**Q: How do I deep clone an object?**
- `JSON.parse(JSON.stringify(obj))` — works for plain data (no functions, `undefined`, `Date` objects, circular refs)
- `structuredClone(obj)` — modern API (Node 17+, modern browsers), handles more types including `Date` and circular refs, but not functions

**Q: Can I spread an object into an array?**
No — spreading an object (`...obj`) inside `[]` throws a TypeError. Objects are not iterable.

```js
const obj = { a: 1 };
const arr = [...obj]; // TypeError: obj is not iterable
```

**Q: What does `{ ...obj, key: undefined }` do?**
It copies all of `obj` and then sets `key` to `undefined`. The key still exists on the result (unlike `delete`). Use destructuring + rest if you want to actually remove a key.

**Q: Can I spread multiple objects in one expression?**
Yes — as many as you want, and they're applied left to right:

```js
const result = { ...a, ...b, ...c, extra: 'value' };
```

---

## Practice exercises

### Exercise 1 — Easy: Config builder

You have these two objects:

```js
const serverDefaults = {
  host:     'localhost',
  port:     3000,
  timeout:  5000,
  retries:  3,
  secure:   false,
};

const productionOverrides = {
  host:    'api.myapp.com',
  port:    443,
  secure:  true,
};
```

1. Create `devConfig` — just a copy of `serverDefaults` (no mutations)
2. Create `prodConfig` — merge defaults with production overrides (overrides win)
3. Create `stagingConfig` — start from prodConfig but set `host` to `'staging.myapp.com'` and `timeout` to `10000`
4. Log all three and confirm `serverDefaults` is unchanged

```js
// Write your code here
```

---

### Exercise 2 — Medium: Immutable state updates

You are managing a simple game state:

```js
const gameState = {
  player: {
    name:   'Parsh',
    health: 100,
    level:  1,
  },
  score:   0,
  paused:  false,
  enemies: 5,
};
```

Write these functions — **none of them should mutate `gameState`**, each must return a new state object:

- `addScore(state, points)` — returns new state with `score` increased
- `takeDamage(state, amount)` — returns new state with `player.health` decreased (never below 0)
- `levelUp(state)` — returns new state with `player.level` incremented and `player.health` restored to 100
- `togglePause(state)` — returns new state with `paused` flipped

Test each function and confirm `gameState` is always unchanged after calling them.

```js
// Write your code here
```

---

### Exercise 3 — Hard: Data transformation pipeline

You receive raw user records from a legacy system:

```js
const rawUsers = [
  {
    usr_id:   'U001',
    f_name:   'Alice',
    l_name:   'Chen',
    pwd_hash: 'abc123hashed',
    e_mail:   'alice@example.com',
    meta:     { created: '2024-01-15', verified: true, plan: 'pro' },
  },
  {
    usr_id:   'U002',
    f_name:   'Bob',
    l_name:   'Smith',
    pwd_hash: 'xyz789hashed',
    e_mail:   'bob@example.com',
    meta:     { created: '2024-03-20', verified: false, plan: 'free' },
  },
  {
    usr_id:   'U003',
    f_name:   'Carol',
    l_name:   'Jones',
    pwd_hash: 'def456hashed',
    e_mail:   'carol@example.com',
    meta:     { created: '2025-07-10', verified: true },  // no plan key
  },
];
```

Write `transformUsers(users)` that returns a new array where each user is transformed using **spread and destructuring** (no mutation of originals):

```js
[
  {
    id:        'U001',
    fullName:  'Alice Chen',
    email:     'alice@example.com',
    verified:  true,
    plan:      'pro',
    joinedYear: 2024,
  },
  // ...
]
```

Rules:
- Remove `pwd_hash` (do NOT include it in output)
- Rename fields: `usr_id` → `id`, `e_mail` → `email`
- Combine `f_name` + `l_name` into `fullName`
- Flatten `meta` into the top level — extract `verified`, `plan` (default `'free'` if missing), and `joinedYear` (parse year from `created` string)
- Do not include `meta`, `f_name`, `l_name`, `usr_id`, `e_mail` in the output

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Shallow copy
const copy = { ...obj };

// Merge (last wins on conflict)
const merged = { ...obj1, ...obj2 };

// Copy + override specific key
const updated = { ...obj, key: newValue };

// Copy + add new key
const extended = { ...obj, newKey: 'value' };

// Remove a key (destructuring + rest)
const { unwantedKey, ...rest } = obj;

// Conditional property
const result = {
  ...obj,
  ...(condition && { extraKey: value }),
};

// Immutable nested update
const newState = {
  ...state,
  nested: { ...state.nested, field: newValue },
};
```

| Pattern | Purpose |
|---|---|
| `{ ...a }` | Shallow copy |
| `{ ...a, ...b }` | Merge (b wins) |
| `{ ...a, k: v }` | Copy + override k |
| `const { k, ...r } = a` | Remove key k |
| `{ ...a, nested: { ...a.nested, k: v } }` | Immutable nested update |

---

## Connected topics

- **[32] Spread with Arrays** — same `...` operator, used inside `[]`
- **[36] Object Destructuring** — the counterpart: extracting from objects vs building them
- **[34] Object Basics** — the foundation: CRUD on plain objects
- **[40] Computed Property Names** — using dynamic keys with spread
- **[75] Immutability** — why this pattern matters at scale
