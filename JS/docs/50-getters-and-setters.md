# 50 — Getters and Setters

## What is this?

Getters and setters are special methods that let you **intercept property reads and writes** on an object. A getter looks like a property when you read it (`obj.fullName`) but secretly runs a function to compute or return a value. A setter looks like a property assignment (`obj.fullName = "Alice Smith"`) but secretly runs a function to validate, transform, or store the value. Think of them like a receptionist at a front desk — every request to get or set something passes through them first, so they can apply rules, format data, or refuse invalid input before anything reaches the internal storage.

## Why does it matter?

Getters and setters give you **controlled access** to an object's internal state without changing the external API. You can start with a plain property today and swap it for a getter/setter tomorrow — code that accesses `person.age` never needs to change, but now you can add validation, logging, or computation behind the scenes. This is the foundation of **encapsulation** — one of the core principles of object-oriented programming.

---

## Syntax

```js
class Circle {
  constructor(radius) {
    this._radius = radius;   // _ signals "internal storage — use the getter/setter"
  }

  get radius() {             // getter — accessed like: circle.radius (no parentheses)
    return this._radius;
  }

  set radius(value) {        // setter — triggered like: circle.radius = 5
    if (value < 0) throw new RangeError("Radius cannot be negative.");
    this._radius = value;
  }

  get area() {               // computed getter — no corresponding setter
    return Math.PI * this._radius ** 2;
  }

  get diameter() {
    return this._radius * 2;
  }

  set diameter(value) {      // setter that writes to a different backing field
    this._radius = value / 2;
  }
}

const c = new Circle(5);
console.log(c.radius);     // 5         — getter called
console.log(c.area);       // 78.539...  — computed getter, no backing field
c.radius = 10;             // setter called, validates then stores
console.log(c.diameter);   // 20         — derived from radius
c.diameter = 6;            // setter updates radius via diameter
console.log(c.radius);     // 3
```

---

## How it works — line by line

```js
get radius() {
  return this._radius;
}
```
`get` declares a getter. When any code reads `someCircle.radius`, this function runs and its return value is used. No `()` at the call site — it looks like a plain property.

```js
set radius(value) {
  if (value < 0) throw new RangeError("...");
  this._radius = value;
}
```
`set` declares a setter with exactly **one parameter** — the value being assigned. When any code writes `someCircle.radius = 10`, this function runs with `value = 10`.

```js
get area() {
  return Math.PI * this._radius ** 2;
}
```
A **read-only computed property** — there's no backing field, the value is calculated fresh each time. Because there's no matching `set area(...)`, trying to assign to it either fails silently (non-strict) or throws (strict mode / classes).

---

## Getters and setters in object literals

You don't need a class — getters and setters work directly in object literals too:

```js
const person = {
  _firstName: "Alice",
  _lastName:  "Smith",

  get fullName() {
    return `${this._firstName} ${this._lastName}`;
  },

  set fullName(value) {
    const parts = value.trim().split(" ");
    if (parts.length < 2) throw new Error("Full name must have at least two words.");
    this._firstName = parts[0];
    this._lastName  = parts.slice(1).join(" ");
  },

  get initials() {
    return `${this._firstName[0]}.${this._lastName[0]}.`;
  }
};

console.log(person.fullName);       // "Alice Smith"
console.log(person.initials);       // "A.S."

person.fullName = "Bob Johnson";
console.log(person.fullName);       // "Bob Johnson"
console.log(person.initials);       // "B.J."

person.fullName = "Madonna";        // Error: Full name must have at least two words.
```

---

## Example 1 — validation with setter

```js
class UserProfile {
  constructor(username, age) {
    // Use setters in constructor to run validation immediately
    this.username = username;   // triggers set username()
    this.age      = age;        // triggers set age()
  }

  get username() {
    return this._username;
  }

  set username(value) {
    if (typeof value !== "string")    throw new TypeError("Username must be a string.");
    if (value.trim().length < 3)      throw new RangeError("Username must be at least 3 chars.");
    if (/\s/.test(value))             throw new Error("Username cannot contain spaces.");
    this._username = value.toLowerCase().trim();
  }

  get age() {
    return this._age;
  }

  set age(value) {
    if (!Number.isInteger(value))     throw new TypeError("Age must be an integer.");
    if (value < 13 || value > 120)    throw new RangeError("Age must be between 13 and 120.");
    this._age = value;
  }

  get isAdult() {            // computed — no setter
    return this._age >= 18;
  }

  toString() {
    return `${this._username} (age ${this._age}) — ${this.isAdult ? "adult" : "minor"}`;
  }
}

const user = new UserProfile("Alice99", 25);
console.log(user.toString());    // "alice99 (age 25) — adult"
console.log(user.isAdult);       // true

user.age = 17;
console.log(user.isAdult);       // false — getter recomputes every time

new UserProfile("AB", 25);       // RangeError: Username must be at least 3 chars.
new UserProfile("alice", 10);    // RangeError: Age must be between 13 and 120.
```

---

## Example 2 — lazy computation with getter caching

For expensive computed properties, you can calculate once and cache the result:

```js
class TextAnalyzer {
  constructor(text) {
    this._text      = text;
    this._wordCache = null;   // null = not yet computed
  }

  get text() {
    return this._text;
  }

  set text(value) {
    this._text      = value;
    this._wordCache = null;   // invalidate cache when text changes
  }

  get words() {
    if (this._wordCache === null) {
      console.log("(computing words...)");   // only logged on first access
      this._wordCache = this._text
        .toLowerCase()
        .match(/\b[a-z]+\b/g) || [];
    }
    return this._wordCache;
  }

  get wordCount() {
    return this.words.length;
  }

  get uniqueWords() {
    return new Set(this.words).size;
  }

  get mostCommon() {
    const freq = {};
    for (const word of this.words) {
      freq[word] = (freq[word] || 0) + 1;
    }
    return Object.entries(freq).sort((a, b) => b[1] - a[1])[0]?.[0] ?? null;
  }
}

const ta = new TextAnalyzer("the quick brown fox jumps over the lazy dog the");
console.log(ta.wordCount);     // (computing words...) → 10
console.log(ta.wordCount);     // (no log) → 10   — cached
console.log(ta.uniqueWords);   // 8
console.log(ta.mostCommon);    // "the"

ta.text = "hello hello world"; // clears cache
console.log(ta.wordCount);     // (computing words...) → 3  — recomputed
```

---

## Example 3 — real world: reactive-style data binding

A simplified version of what Vue/React do — automatically tracking when a value changes:

```js
class ReactiveValue {
  #value;
  #listeners = [];

  constructor(initialValue) {
    this.#value = initialValue;
  }

  get value() {
    return this.#value;
  }

  set value(newValue) {
    const oldValue = this.#value;
    if (newValue === oldValue) return;    // no change — don't notify
    this.#value = newValue;
    for (const listener of this.#listeners) {
      listener(newValue, oldValue);       // notify all subscribers
    }
  }

  onChange(callback) {
    this.#listeners.push(callback);
    return () => {                        // returns an unsubscribe function
      this.#listeners = this.#listeners.filter(l => l !== callback);
    };
  }
}

const count = new ReactiveValue(0);

const unsubscribe = count.onChange((newVal, oldVal) => {
  console.log(`count changed: ${oldVal} → ${newVal}`);
});

count.value = 1;   // count changed: 0 → 1
count.value = 2;   // count changed: 1 → 2
count.value = 2;   // (no output — same value, setter short-circuits)

unsubscribe();
count.value = 3;   // (no output — unsubscribed)
console.log(count.value);  // 3
```

---

## `Object.defineProperty` — getters/setters the low-level way

Before ES6 class syntax, getters and setters were defined via `Object.defineProperty`. You'll see this in older code and libraries:

```js
const product = { _price: 100 };

Object.defineProperty(product, "price", {
  get() {
    return this._price;
  },
  set(value) {
    if (value < 0) throw new RangeError("Price cannot be negative.");
    this._price = value;
  },
  enumerable:   true,    // shows up in for...in and Object.keys
  configurable: true,    // can be redefined or deleted
});

console.log(product.price);    // 100
product.price = 250;
console.log(product.price);    // 250
product.price = -10;           // RangeError

// Also works for getters only (read-only property):
Object.defineProperty(product, "discountedPrice", {
  get() { return this._price * 0.9; },
  enumerable: true,
  configurable: false,   // cannot redefine
});
console.log(product.discountedPrice);  // 225
```

---

## Checking if a property is a getter/setter

You can inspect property descriptors to see if a property is a getter/setter:

```js
class Temp {
  get celsius() { return this._c; }
  set celsius(v) { this._c = v; }
}

const descriptor = Object.getOwnPropertyDescriptor(Temp.prototype, "celsius");
console.log(descriptor);
/*
{
  get: [Function: get celsius],
  set: [Function: set celsius],
  enumerable: false,
  configurable: true
}
*/

// For a regular property:
const obj = { name: "Alice" };
console.log(Object.getOwnPropertyDescriptor(obj, "name"));
/*
{
  value: "Alice",
  writable: true,
  enumerable: true,
  configurable: true
}
*/
// Note: getter/setter descriptors have 'get'/'set' but NOT 'value'/'writable'
// Regular property descriptors have 'value'/'writable' but NOT 'get'/'set'
```

---

## Inheritance and getters/setters

Getters and setters are inherited just like regular methods — the child class can override them:

```js
class Animal {
  constructor(name) {
    this._name = name;
  }
  get name() {
    return this._name;
  }
  set name(value) {
    if (!value) throw new Error("Name cannot be empty.");
    this._name = value;
  }
}

class Dog extends Animal {
  get name() {
    return `Dog: ${super.name}`;   // call parent getter with super
  }
  // Setter NOT overridden — inherited Animal's setter still applies
}

const d = new Dog("Rex");
console.log(d.name);    // "Dog: Rex"  — child getter used
d.name = "Max";         // Animal's setter runs — validates
console.log(d.name);    // "Dog: Max"
d.name = "";            // Error: Name cannot be empty.
```

---

## Tricky things

### 1. Getter without setter — silent fail in non-strict, error in class

```js
class Box {
  get size() { return 10; }
  // No setter defined
}

const b = new Box();
b.size = 20;   // In class (strict mode): TypeError: Cannot set property size of #<Box>
               // In plain object literal (non-strict): silently ignored

console.log(b.size);  // still 10
```

### 2. Infinite loop — getter calling itself

```js
// WRONG
class User {
  get name() {
    return this.name;    // calls get name() again → infinite recursion → stack overflow
  }
}

// RIGHT — store in a backing field with underscore
class User {
  get name() {
    return this._name;   // reads the private backing field
  }
  set name(v) {
    this._name = v;
  }
}
```

### 3. Using `this.prop = value` in the constructor calls the setter

```js
class Temperature {
  set celsius(v) {
    if (v < -273.15) throw new RangeError("Below absolute zero.");
    this._celsius = v;
  }
  get celsius() { return this._celsius; }
}

class BodyTemp extends Temperature {
  constructor(value) {
    super();
    this.celsius = value;   // this triggers the setter — validation runs at construction
  }
}

new BodyTemp(37);     // OK
new BodyTemp(-500);   // RangeError: Below absolute zero. — caught at construction time
```

This is intentional and good practice: run validation immediately in the constructor by using the setter.

### 4. Getters are NOT cached by default

```js
class Expensive {
  get result() {
    console.log("(computing...)");
    return Array.from({ length: 1000 }, (_, i) => i).reduce((a, b) => a + b);
  }
}

const e = new Expensive();
console.log(e.result);  // (computing...) 499500
console.log(e.result);  // (computing...) 499500  — computed AGAIN
console.log(e.result);  // (computing...) 499500  — and again
```

Unless you implement manual caching (see Example 2), the getter runs every time.

### 5. Getter defined on prototype — shadowing by instance property

```js
class Config {
  get theme() { return "light"; }
}

const c = new Config();
console.log(c.theme);    // "light" — runs getter

// Assigning directly creates an OWN property that shadows the getter
Object.defineProperty(c, "theme", { value: "dark", writable: true, configurable: true });
console.log(c.theme);    // "dark" — own property found first, getter bypassed
```

### 6. No parameters in getter, exactly one in setter

```js
class Wrong {
  get size(unit) { ... }     // SyntaxError — getters cannot have parameters
  set size() { ... }         // SyntaxError — setters must have exactly one parameter
  set size(a, b) { ... }     // SyntaxError — setters cannot have more than one parameter
}
```

---

## Common mistakes

### Mistake 1 — Forgetting the backing field (infinite recursion)

```js
// WRONG — self-referencing getter
class Product {
  get price() { return this.price; }   // infinite loop
  set price(v) { this.price = v; }     // infinite loop
}

// RIGHT — use a separate backing field
class Product {
  get price()  { return this._price; }
  set price(v) {
    if (v < 0) throw new RangeError("Price must be non-negative.");
    this._price = v;
  }
}
```

### Mistake 2 — Calling the getter with parentheses

```js
class Circle {
  get area() { return Math.PI * this._r ** 2; }
}
const c = new Circle();
c._r = 5;

// WRONG
console.log(c.area());   // TypeError: c.area is not a function

// RIGHT
console.log(c.area);     // 78.539...
```

### Mistake 3 — Not invalidating cache when data changes

```js
// WRONG — stale cache
class Cart {
  constructor() {
    this.items = [];
    this._total = null;
  }
  addItem(price) {
    this.items.push(price);
    // forgot to reset this._total = null
  }
  get total() {
    if (this._total === null) {
      this._total = this.items.reduce((a, b) => a + b, 0);
    }
    return this._total;
  }
}

const cart = new Cart();
cart.addItem(10);
console.log(cart.total);   // 10 — computed
cart.addItem(20);
console.log(cart.total);   // 10 — WRONG: stale cache, should be 30

// RIGHT — reset cache in addItem
addItem(price) {
  this.items.push(price);
  this._total = null;   // invalidate
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a class `Rectangle` with backing fields `_width` and `_height`. Add getters and setters for both that throw a `RangeError` if the value is not a positive number. Add read-only (getter-only) computed properties `area` and `perimeter`. Add a getter `isSquare` that returns `true` if `width === height`. Verify that:
- Setting a negative width throws
- `area` and `perimeter` update automatically when you change width or height
- `isSquare` works correctly

```js
// Write your code here
```

### Exercise 2 — medium

Create a class `Password` that stores a password securely. Requirements:
- The raw password is stored in a **private** field `#raw`
- A setter `set password(value)` validates: at least 8 chars, contains at least one uppercase letter, one digit, one special character (`!@#$%^&*`). Throw descriptive errors for each rule violated.
- A getter `get password()` always returns a masked version: first 2 chars visible, rest replaced with `*` (e.g., `"MyPass!1"` → `"My******"`)
- A getter `get strength()` returns `"weak"`, `"medium"`, or `"strong"` based on length (< 10 = weak, 10–14 = medium, 15+ = strong)
- A method `verify(input)` returns `true` if `input === this.#raw`

```js
// Write your code here
```

### Exercise 3 — hard

Build a class `SmartArray` that wraps a regular array and adds computed getters. The internal array is stored in a private field `#data`. Expose:
- `set items(arr)` — setter that validates: input must be an array; if not, throw. Also clears all computed caches.
- `get items()` — returns a copy of the array (not the original — callers shouldn't mutate internal state)
- `get length()` — count
- `get sum()` — cached: sum of all numbers (cache invalidated when `items` is set)
- `get average()` — cached: average (return `null` if empty)
- `get min()` / `get max()` — cached
- `get sorted()` — returns a **new** sorted array (does NOT sort `#data` in place)
- `get unique()` — returns array with duplicates removed
- `push(...values)` — appends to `#data` and invalidates all caches; returns `this`
- `filter(fn)` — returns a **new** `SmartArray` containing only matching items

Test with: create a SmartArray, push several numbers, read `sum`, `average`, `min`, `max`, `sorted`, `unique`, then replace `items` entirely and show caches are fresh.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax / Rule |
|---|---|
| Define getter | `get propName() { return ...; }` — in class or object literal |
| Define setter | `set propName(value) { ... }` — exactly one parameter |
| Access getter | `obj.propName` — no parentheses |
| Trigger setter | `obj.propName = newValue` — looks like plain assignment |
| Backing field convention | `_propName` (or private `#propName`) to avoid getter/setter loop |
| Read-only property | Getter with no matching setter |
| Write-only property | Setter with no matching getter (rare) |
| Getter in constructor | `this.prop = val` in constructor calls the setter if one exists |
| Getter NOT cached | Runs every time it's accessed — add manual caching if expensive |
| Cache invalidation | Reset cache variable to `null` whenever backing data changes |
| Inherited | Yes — child overrides with same `get`/`set` keyword; call parent with `super.propName` |
| `super` in getter | `return super.propName` — valid |
| `Object.defineProperty` | Low-level way to add getters/setters to existing objects |
| Descriptor check | `Object.getOwnPropertyDescriptor(obj, "prop")` — has `get`/`set` keys, not `value`/`writable` |
| Getter parameter count | Exactly 0 — syntax error otherwise |
| Setter parameter count | Exactly 1 — syntax error otherwise |

---

## Connected topics

- **47 — ES6 Classes** — Getters and setters are defined inside classes; that topic covers the overall class structure.
- **51 — Private class fields (#)** — Getters/setters are most powerful when paired with private backing fields. The `#` syntax is the modern, truly private alternative to `_`.
- **46 — Prototypes** — Getters defined in a class live on the prototype, not on each instance. Understanding this prevents confusion about where the getter function actually lives.
