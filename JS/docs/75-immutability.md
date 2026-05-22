# 75 — Immutability

---

## 1. What is this?

**Immutability** means that once a value is created, it cannot be changed. Instead of modifying existing data, you create a new copy with the changes applied. The original stays untouched.

**Analogy — Git commits:**
You never go back and edit an old commit. You make a new commit on top. The history is immutable — every previous state is preserved. Immutable data in JavaScript works the same way: you always produce a new version, the old version still exists unchanged.

**Analogy — accounting:**
A double-entry bookkeeping system never erases entries. Mistakes are corrected by adding a new correcting entry. The full history is always preserved. Mutable state is like erasing — immutable state is like appending.

---

## 2. Why does it matter?

- **Predictability** — a function that doesn't mutate its inputs can't cause surprise side effects in other parts of the code
- **Debugging** — if data never changes in place, you always know what a value was at any point in time (no "who changed this?")
- **React / state management** — React uses shallow equality checks (`===`) to decide if a component should re-render. If you mutate an array in place, React sees the same reference and doesn't re-render. You must return a new array.
- **Concurrency** — immutable data is inherently thread-safe (relevant in Node.js worker threads and upcoming JS parallel features)
- **Undo/redo, time-travel debugging** — trivial when every state change produces a new snapshot instead of overwriting

---

## 3. Primitive vs Reference Types

Primitives are **already immutable** in JavaScript:

```js
let str = "hello";
str.toUpperCase(); // returns "HELLO" — str is unchanged
console.log(str);  // "hello"

let num = 42;
num.toFixed(2);    // returns "42.00" — num unchanged
console.log(num);  // 42
```

Objects and arrays are **mutable by default** — operations mutate in place:

```js
const user = { name: "Alice", age: 30 };
user.age = 31;        // mutates the original object
console.log(user);    // { name: "Alice", age: 31 } — changed!

const nums = [1, 2, 3];
nums.push(4);         // mutates in place
console.log(nums);    // [1, 2, 3, 4] — changed!
```

---

## 4. Immutable Object Patterns

### Spread — shallow copy with changes

```js
const user = { name: "Alice", age: 30, role: "user" };

// ✅ Return new object, original untouched
const updatedUser = { ...user, age: 31 };

console.log(user);        // { name: "Alice", age: 30, role: "user" } — unchanged
console.log(updatedUser); // { name: "Alice", age: 31, role: "user" }
```

### Nested objects — the shallow copy trap

```js
const state = {
  user: { name: "Alice", address: { city: "London" } },
  count: 0
};

// ❌ Shallow copy — nested object is still shared
const next = { ...state, count: 1 };
next.user.address.city = "Paris"; // mutates state.user.address too!
console.log(state.user.address.city); // "Paris" — oops

// ✅ Spread all the way down for the path you're changing
const next = {
  ...state,
  count: 1,
  user: {
    ...state.user,
    address: {
      ...state.user.address,
      city: "Paris"    // only this is new
    }
  }
};
console.log(state.user.address.city); // "London" — untouched
```

### `Object.assign` — same result as spread

```js
// These are equivalent:
const next = { ...original, key: newValue };
const next = Object.assign({}, original, { key: newValue });

// Object.assign mutates the first argument — never pass the original as first
// ❌ Mutates original
Object.assign(original, { key: newValue });

// ✅ Pass empty object as target
Object.assign({}, original, { key: newValue });
```

### Removing a property immutably

```js
const user = { id: 1, name: "Alice", password: "secret" };

// ❌ delete mutates the object
delete user.password;

// ✅ Destructure and collect the rest
const { password, ...publicUser } = user;
console.log(publicUser); // { id: 1, name: "Alice" }
console.log(user);       // { id: 1, name: "Alice", password: "secret" } — untouched
```

---

## 5. Immutable Array Patterns

### The mutating array methods (avoid or wrap)

| Mutating method | Immutable replacement |
|---|---|
| `push(x)` | `[...arr, x]` |
| `pop()` | `arr.slice(0, -1)` |
| `unshift(x)` | `[x, ...arr]` |
| `shift()` | `arr.slice(1)` |
| `splice(i, n, x)` | `[...arr.slice(0,i), x, ...arr.slice(i+n)]` |
| `sort()` | `[...arr].sort()` |
| `reverse()` | `[...arr].reverse()` |
| `arr[i] = x` | `arr.map((v, j) => j === i ? x : v)` |

### Add to array

```js
const items = [1, 2, 3];

// ✅ Add to end
const next = [...items, 4];          // [1, 2, 3, 4]

// ✅ Add to beginning
const next = [0, ...items];          // [0, 1, 2, 3]

// ✅ Add at specific index
const i = 1;
const next = [...items.slice(0, i), 99, ...items.slice(i)]; // [1, 99, 2, 3]
```

### Remove from array

```js
const items = [1, 2, 3, 4, 5];

// ✅ Remove by index
const next = items.filter((_, i) => i !== 2); // [1, 2, 4, 5]

// ✅ Remove by value
const next = items.filter(x => x !== 3);       // [1, 2, 4, 5]

// ✅ Remove first occurrence by value
const idx  = items.indexOf(3);
const next = idx === -1 ? items : [...items.slice(0, idx), ...items.slice(idx + 1)];
```

### Update item in array

```js
const users = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
];

// ✅ Update by id
const next = users.map(u => u.id === 2 ? { ...u, name: "Bobby" } : u);
// [{ id: 1, name: "Alice" }, { id: 2, name: "Bobby" }]
// original users array unchanged
```

### Sort and reverse without mutation

```js
const nums = [3, 1, 4, 1, 5];

// ❌ Mutates original
nums.sort((a, b) => a - b);

// ✅ Copy first
const sorted   = [...nums].sort((a, b) => a - b);
const reversed = [...nums].reverse();
// nums still [3, 1, 4, 1, 5]

// ES2023 — toSorted(), toReversed(), toSpliced(), with() — immutable by design
const sorted   = nums.toSorted((a, b) => a - b);   // new array, no mutation
const reversed = nums.toReversed();                  // new array, no mutation
const replaced = nums.with(2, 99);                   // new array with index 2 = 99
```

---

## 6. `Object.freeze` and `Object.isFrozen`

`Object.freeze` prevents any modifications to an object:

```js
const config = Object.freeze({
  apiUrl:  "https://api.example.com",
  timeout: 5000,
  retries: 3
});

config.timeout = 9999; // silently fails in sloppy mode
"use strict";
config.timeout = 9999; // TypeError: Cannot assign to read only property

console.log(config.timeout); // 5000 — unchanged

Object.isFrozen(config); // true

// Freeze also prevents:
delete config.apiUrl;        // fails
config.newKey = "value";     // fails
Object.defineProperty(config, "x", { value: 1 }); // TypeError
```

### Shallow freeze — the limitation

```js
const state = Object.freeze({
  user: { name: "Alice" } // nested object is NOT frozen
});

state.user = {};         // ❌ fails — top level is frozen
state.user.name = "Bob"; // ✅ succeeds — nested object is NOT frozen!
console.log(state.user.name); // "Bob"
```

### Deep freeze

```js
function deepFreeze(obj) {
  Object.getOwnPropertyNames(obj).forEach(name => {
    const value = obj[name];
    if (value && typeof value === "object") {
      deepFreeze(value); // recurse into nested objects
    }
  });
  return Object.freeze(obj);
}

const config = deepFreeze({
  db: { host: "localhost", port: 5432 },
  api: { url: "https://example.com" }
});

config.db.host = "remote"; // silently fails
console.log(config.db.host); // "localhost" — protected
```

`Object.freeze` is useful for **configuration objects and constants** that should never change at runtime.

---

## 7. `const` Is Not Immutability

This is one of the most common misconceptions:

```js
const user = { name: "Alice" };

// ❌ Misconception: const makes this immutable
user.name = "Bob"; // ✅ works fine — const only protects the binding
console.log(user.name); // "Bob"

user = { name: "Charlie" }; // ❌ TypeError: Assignment to constant variable
```

`const` prevents **reassigning the variable** — it does not prevent **mutating the object the variable points to**.

| | Prevents reassignment | Prevents mutation |
|---|---|---|
| `const` | ✅ yes | ❌ no |
| `Object.freeze` | ❌ no (you can still reassign) | ✅ yes (shallow) |
| `deepFreeze` | ❌ no | ✅ yes (deep) |
| `const` + `Object.freeze` | ✅ yes | ✅ yes (shallow) |

---

## 8. Immutability in State Management

This is where immutability matters most in frontend development.

### React useState — why mutation breaks it

```js
// ❌ WRONG — mutating state directly
const [items, setItems] = useState([1, 2, 3]);

const addItem = (item) => {
  items.push(item);     // mutates the array
  setItems(items);      // same reference — React sees no change, doesn't re-render!
};

// ✅ CORRECT — return new array
const addItem = (item) => {
  setItems([...items, item]); // new array reference — React re-renders
};

// ✅ CORRECT — use functional update form
const addItem = (item) => {
  setItems(prev => [...prev, item]);
};
```

### Redux-style reducer pattern

```js
// A reducer is a pure function: (state, action) => newState
// It must NEVER mutate state — always return a new object

function todoReducer(state = { items: [], filter: "all" }, action) {
  switch (action.type) {
    case "ADD_TODO":
      return {
        ...state,
        items: [...state.items, { id: Date.now(), text: action.text, done: false }]
      };

    case "TOGGLE_TODO":
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.id ? { ...item, done: !item.done } : item
        )
      };

    case "DELETE_TODO":
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.id)
      };

    case "SET_FILTER":
      return { ...state, filter: action.filter };

    default:
      return state; // always return state for unknown actions
  }
}
```

---

## 9. Real-World Examples

### Example 1 — Immutable config with Object.freeze

```js
// config.js
const config = deepFreeze({
  server: {
    port:    process.env.PORT ? Number(process.env.PORT) : 3000,
    host:    "0.0.0.0",
  },
  db: {
    url:     process.env.DATABASE_URL ?? "postgresql://localhost/myapp",
    pool:    { min: 2, max: 10 }
  },
  jwt: {
    secret:  process.env.JWT_SECRET,
    expires: "7d"
  }
});

export default config;

// Any accidental mutation in app code throws TypeError in strict mode
// config.db.url = "something"; // TypeError in strict mode
```

### Example 2 — Immutable state updates in a cart

```js
class CartStore {
  #state;

  constructor() {
    this.#state = { items: [], coupon: null };
  }

  getState() {
    return this.#state; // read-only — callers shouldn't mutate this
  }

  addItem(product, qty = 1) {
    const existing = this.#state.items.find(i => i.id === product.id);

    this.#state = {
      ...this.#state,
      items: existing
        ? this.#state.items.map(i =>
            i.id === product.id ? { ...i, qty: i.qty + qty } : i
          )
        : [...this.#state.items, { ...product, qty }]
    };
  }

  removeItem(productId) {
    this.#state = {
      ...this.#state,
      items: this.#state.items.filter(i => i.id !== productId)
    };
  }

  applyCoupon(code) {
    this.#state = { ...this.#state, coupon: code };
  }

  getTotal() {
    return this.#state.items.reduce((sum, i) => sum + i.price * i.qty, 0);
  }
}
```

### Example 3 — Functional data pipeline (no mutation)

```js
// All functions return new arrays/objects — none mutate their inputs

const processOrders = (orders) =>
  orders
    .filter(o => o.status === "completed")
    .map(o => ({ ...o, total: o.items.reduce((s, i) => s + i.price * i.qty, 0) }))
    .map(o => ({ ...o, tax: +(o.total * 0.2).toFixed(2) }))
    .sort((a, b) => b.total - a.total)  // ❌ sort mutates — fixed below
    ;

// ✅ Sort without mutation
const processOrders = (orders) =>
  [...orders]                            // copy before sort
    .filter(o => o.status === "completed")
    .map(o => ({ ...o, total: o.items.reduce((s, i) => s + i.price * i.qty, 0) }))
    .map(o => ({ ...o, tax: +(o.total * 0.2).toFixed(2) }))
    .sort((a, b) => b.total - a.total);
```

### Example 4 — Avoiding accidental mutation in Express handlers

```js
// ❌ Mutating the DB result directly is dangerous —
//    other code referencing the same object sees the change
app.get("/users/:id", async (req, res) => {
  const user = await db.findUser(req.params.id);
  delete user.password;       // mutates the cached DB object!
  res.json(user);
});

// ✅ Create a new object without the sensitive field
app.get("/users/:id", async (req, res) => {
  const user = await db.findUser(req.params.id);
  const { password, ...publicUser } = user;
  res.json(publicUser);
});

// ✅ Or use a serialiser
function serializeUser({ password, refreshToken, ...rest }) {
  return rest;
}
res.json(serializeUser(user));
```

### Example 5 — Undo/Redo with history stack

```js
class HistoryManager {
  #past    = [];
  #present = null;
  #future  = [];

  constructor(initialState) {
    this.#present = initialState;
  }

  getState() { return this.#present; }

  push(newState) {
    this.#past    = [...this.#past, this.#present]; // preserve old state
    this.#present = newState;
    this.#future  = []; // clear redo stack on new action
  }

  undo() {
    if (!this.#past.length) return;
    this.#future  = [this.#present, ...this.#future];
    this.#present = this.#past[this.#past.length - 1];
    this.#past    = this.#past.slice(0, -1);
  }

  redo() {
    if (!this.#future.length) return;
    this.#past    = [...this.#past, this.#present];
    this.#present = this.#future[0];
    this.#future  = this.#future.slice(1);
  }

  canUndo() { return this.#past.length > 0; }
  canRedo() { return this.#future.length > 0; }
}

const history = new HistoryManager({ text: "" });
history.push({ text: "Hello" });
history.push({ text: "Hello World" });
history.undo();
history.getState(); // { text: "Hello" }
history.redo();
history.getState(); // { text: "Hello World" }
```

---

## 10. Tricky Things

### 1. Spread is shallow — nested objects are still shared

```js
const a = { x: { y: 1 } };
const b = { ...a };

b.x.y = 99;           // mutates a.x.y too — same reference!
console.log(a.x.y);   // 99

// ✅ Spread all levels you're changing
const b = { ...a, x: { ...a.x, y: 99 } };
```

### 2. `Array.from` and `[...arr]` are shallow copies too

```js
const matrix = [[1, 2], [3, 4]];
const copy   = [...matrix];

copy[0].push(99);           // mutates matrix[0] too
console.log(matrix[0]);     // [1, 2, 99]

// ✅ Deep copy with structuredClone (modern, native)
const copy = structuredClone(matrix);
copy[0].push(99);
console.log(matrix[0]);     // [1, 2] — safe
```

### 3. `structuredClone` — the modern deep copy

```js
// ✅ Native deep clone — works with nested objects, arrays, Dates, Maps, Sets
const original = {
  name: "Alice",
  created: new Date(),
  tags: ["admin", "user"],
  meta: { visits: 42 }
};

const clone = structuredClone(original);
clone.meta.visits = 100;
console.log(original.meta.visits); // 42 — fully independent

// ⚠️ structuredClone does NOT handle: functions, class instances, undefined values in objects
// For those, use a library (lodash cloneDeep) or manual reconstruction
```

### 4. `Object.freeze` is synchronous — frozen objects can still have mutable prototype methods

```js
const arr = Object.freeze([1, 2, 3]);
arr.push(4); // TypeError: Cannot add property 3, object is not extensible
arr[0] = 99; // TypeError: Cannot assign to read only property '0'
arr.length;  // 3 — reading is fine
```

### 5. Immer — write mutable-looking code that produces immutable updates

```js
import { produce } from "immer";

const state = {
  users: [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }],
  count: 2
};

// You write mutating code — Immer intercepts and produces a new object
const next = produce(state, draft => {
  draft.users[0].name = "Alicia"; // looks mutable — isn't
  draft.count++;
  draft.users.push({ id: 3, name: "Charlie" });
});

console.log(state.users[0].name); // "Alice" — unchanged
console.log(next.users[0].name);  // "Alicia" — new object
```

Immer is widely used in Redux Toolkit, and is worth knowing for complex nested updates.

---

## 11. Common Mistakes

### Mistake 1 — Thinking `const` means immutable

```js
// ❌ This mutates — const doesn't protect against it
const users = [];
users.push({ id: 1 });  // perfectly valid — users is mutated

// ✅ Use spread to produce new arrays
const nextUsers = [...users, { id: 1 }];
```

### Mistake 2 — Mutating function parameters

```js
// ❌ Mutating a parameter mutates the caller's object
function addRole(user, role) {
  user.roles.push(role); // mutates the original
  return user;
}

// ✅ Return a new object
function addRole(user, role) {
  return { ...user, roles: [...user.roles, role] };
}
```

### Mistake 3 — Forgetting `.sort()` and `.reverse()` mutate

```js
// ❌ Mutates the original array silently
const sorted = myArray.sort();

// ✅ Copy first
const sorted = [...myArray].sort();
// or ES2023:
const sorted = myArray.toSorted();
```

### Mistake 4 — `JSON.parse(JSON.stringify(obj))` as deep clone

```js
// Works for plain data but has significant limitations:
const obj = {
  fn:        function() {},  // lost — functions are not JSON-serialisable
  date:      new Date(),     // becomes a string, not a Date object
  undef:     undefined,      // lost — undefined is not JSON-serialisable
  map:       new Map(),      // becomes {} — Maps/Sets are not JSON-serialisable
  circular:  null,
};
obj.circular = obj;          // JSON.stringify throws on circular references

// ✅ Use structuredClone for deep copies of plain objects/arrays/dates/maps/sets
const copy = structuredClone(obj);
```

### Mistake 5 — Over-copying — spreading unnecessarily

```js
// ❌ Spreading on every access is wasteful when you're only reading
function getTotalPrice(cart) {
  const copy = [...cart.items]; // unnecessary — you're only reading
  return copy.reduce((s, i) => s + i.price, 0);
}

// ✅ Only copy when you need to mutate or return a new version
function getTotalPrice(cart) {
  return cart.items.reduce((s, i) => s + i.price, 0);
}
```

---

## 12. Practice Exercises

### Easy — immutable todo updates

Write three pure functions that operate on a todos array without mutating it:
- `addTodo(todos, text)` — returns new array with a new todo `{ id, text, done: false }`
- `toggleTodo(todos, id)` — returns new array with that todo's `done` flipped
- `removeTodo(todos, id)` — returns new array without that todo

Log the original array after each operation to prove it was not mutated.

---

### Medium — immutable state reducer

Write a `cartReducer(state, action)` function that handles these actions:
- `{ type: "ADD_ITEM", item: { id, name, price } }`
- `{ type: "REMOVE_ITEM", id }`
- `{ type: "UPDATE_QTY", id, qty }` (qty 0 = remove)
- `{ type: "APPLY_COUPON", code }`
- `{ type: "CLEAR_CART" }`

Rules:
- Never mutate `state`
- Return the same `state` reference for unrecognised actions
- Initial state shape: `{ items: [], coupon: null }`
- Write five test calls that verify the original state is unchanged after each dispatch

---

### Hard — mini state manager with history

Build a `Store` class that:
- Holds state and a reducer function
- `dispatch(action)` — calls reducer, stores new state, saves old state to history
- `getState()` — returns current state (frozen with `Object.freeze`)
- `undo()` — reverts to previous state
- `redo()` — re-applies undone state
- `subscribe(fn)` — calls `fn(newState, oldState)` after every dispatch or undo/redo
- Limit history to 50 entries

Test it with the cart reducer from above.

---

## 13. Quick Reference

```
PRIMITIVES — already immutable (strings, numbers, booleans, null, undefined, Symbol)

IMMUTABLE OBJECT UPDATES
  { ...original, key: newVal }           update property
  { ...original, ...patch }              merge patch
  const { key, ...rest } = original      remove property (rest = without key)
  Object.assign({}, original, patch)     same as spread merge

IMMUTABLE ARRAY UPDATES
  [...arr, item]                          add to end
  [item, ...arr]                          add to start
  arr.filter(x => x !== item)            remove by value
  arr.filter((_, i) => i !== idx)        remove by index
  arr.map(x => x.id === id ? {...x, ...changes} : x)  update by id
  [...arr].sort(fn)                      sort (copy first!)
  arr.toSorted(fn)                       sort — ES2023, non-mutating
  arr.toReversed()                       reverse — ES2023, non-mutating
  arr.with(index, newVal)               replace at index — ES2023, non-mutating

FREEZING
  Object.freeze(obj)      shallow freeze — top-level only
  deepFreeze(obj)         recursive freeze
  Object.isFrozen(obj)    check if frozen

DEEP COPY
  structuredClone(obj)    native deep clone (no functions, no class instances)
  [...arr]                shallow array copy
  { ...obj }              shallow object copy
  JSON.parse(JSON.stringify(obj))  deep clone (loses Date/Map/undefined/functions)

CONST VS FREEZE
  const   = binding is fixed, object can still mutate
  freeze  = object can't mutate, binding can still be reassigned

KEY RULES
  Never mutate function parameters
  Copy before sort/reverse
  Spread all the way down for nested updates
  structuredClone > JSON roundtrip for deep copies
  Return same reference for no-op updates (equality checks)
```

---

## 14. Connected Topics

- **07 — Variables** — `const` vs `let` and the binding vs mutation distinction
- **09 — Objects** — spread, `Object.assign`, `Object.freeze`
- **10 — Arrays** — mutating vs non-mutating array methods
- **42 — Classes** — private fields (`#`) protect internal state from external mutation
- **72 — try/catch** — `Object.freeze` throws `TypeError` in strict mode on mutation attempts
- **76 — Module pattern** — immutable public API, private mutable internals
- **React / Redux** — shallow equality checks make immutable updates mandatory for re-renders
- **Immer** — library that makes complex immutable updates ergonomic
