# 12 — for...in

## What is this?

`for...in` is a loop that iterates over the **enumerable property keys** of an object.

```js
for (const key in object) {
  // key is a string — the name of each property
}
```

It was designed for objects. It answers the question: *"What properties does this object have?"*

---

## Why does it matter?

When you have a plain object and need to inspect or transform its keys, `for...in` is the built-in way to walk through them:

```js
const userSettings = {
  theme: "dark",
  language: "en",
  notifications: true,
  fontSize: 16,
};

for (const setting in userSettings) {
  console.log(setting, "→", userSettings[setting]);
}
// theme → dark
// language → en
// notifications → true
// fontSize → 16
```

Without `for...in`, your alternatives are `Object.keys()` + `for...of`, which is slightly more verbose. `for...in` is the classic, direct way.

**Critical distinction:**
- `for...in` → **keys** of an **object**
- `for...of` → **values** of an **iterable** (array, string, Set, Map)

These are easy to mix up. Get this difference locked in now.

---

## Syntax

```js
for (const key in targetObject) {
  // key is always a string
  const value = targetObject[key];  // bracket notation to access the value
}
```

- `key` — a string containing the property name
- `targetObject` — the object whose keys you're iterating
- You **cannot** use dot notation to access the value because the key is a variable: `targetObject.key` looks for a literal property called "key", not the value of the variable `key`
- Use **bracket notation**: `targetObject[key]`

---

## How it works — line by line

```js
const bookDetails = {
  title: "Atomic Habits",
  author: "James Clear",
  year: 2018,
  pages: 320,
};

for (const key in bookDetails) {
  console.log(key + ": " + bookDetails[key]);
}
```

**Execution trace:**

```
Loop starts → JS collects all enumerable keys of bookDetails
Iteration 1: key = "title"   → bookDetails["title"]  = "Atomic Habits"
Iteration 2: key = "author"  → bookDetails["author"] = "James Clear"
Iteration 3: key = "year"    → bookDetails["year"]   = 2018
Iteration 4: key = "pages"   → bookDetails["pages"]  = 320
No more keys → loop ends
```

**Key is always a string** — even `year` and `pages` which hold numbers. The key `"year"` is the string `"year"`, not the number 2018.

---

## Example 1 — basic

### Display all settings

```js
const appConfig = {
  darkMode: true,
  language: "English",
  autoSave: true,
  fontSize: 14,
  currency: "USD",
};

console.log("Current settings:");
for (const key in appConfig) {
  console.log(`  ${key}: ${appConfig[key]}`);
}
// Current settings:
//   darkMode: true
//   language: English
//   autoSave: true
//   fontSize: 14
//   currency: USD
```

### Check which settings are boolean

```js
const booleanSettings = [];

for (const key in appConfig) {
  if (typeof appConfig[key] === "boolean") {
    booleanSettings.push(key);
  }
}

console.log("Toggle settings:", booleanSettings.join(", "));
// "Toggle settings: darkMode, autoSave"
```

---

## Example 2 — real world

### Comparing two objects for differences

```js
const originalProfile = {
  displayName: "Alexis",
  email: "alexis@email.com",
  bio: "Frontend developer",
  location: "NYC",
};

const updatedProfile = {
  displayName: "Alexis M.",
  email: "alexis@email.com",
  bio: "Senior frontend developer",
  location: "NYC",
};

const changedFields = [];

for (const key in updatedProfile) {
  if (updatedProfile[key] !== originalProfile[key]) {
    changedFields.push({
      field: key,
      before: originalProfile[key],
      after: updatedProfile[key],
    });
  }
}

console.log("Profile changes:", changedFields);
// [
//   { field: "displayName", before: "Alexis", after: "Alexis M." },
//   { field: "bio", before: "Frontend developer", after: "Senior frontend developer" }
// ]
```

### Building a query string from an object

```js
const searchFilters = {
  category: "electronics",
  minPrice: 50,
  maxPrice: 500,
  brand: "sony",
  inStock: true,
};

const queryParts = [];

for (const key in searchFilters) {
  queryParts.push(`${key}=${searchFilters[key]}`);
}

const queryString = "?" + queryParts.join("&");
console.log(queryString);
// ?category=electronics&minPrice=50&maxPrice=500&brand=sony&inStock=true
```

### Counting occurrences (frequency map)

```js
const surveyResponses = ["agree", "disagree", "agree", "neutral", "agree", "disagree"];

// Build frequency map first
const frequency = {};
for (const response of surveyResponses) {
  if (frequency[response] === undefined) {
    frequency[response] = 0;
  }
  frequency[response]++;
}
// frequency = { agree: 3, disagree: 2, neutral: 1 }

// Then use for...in to report results
for (const answer in frequency) {
  console.log(`${answer}: ${frequency[answer]} responses`);
}
// agree: 3 responses
// disagree: 2 responses
// neutral: 1 response
```

This pattern — building a frequency object with `for...of`, then reading it with `for...in` — appears constantly in real code.

---

## Tricky things you'll encounter in the real world

### 1. `for...in` iterates inherited properties too

This is the most dangerous gotcha with `for...in`.

```js
function Vehicle(make, model) {
  this.make = make;
  this.model = model;
}
Vehicle.prototype.describe = function() {
  return `${this.make} ${this.model}`;
};

const myCar = new Vehicle("Toyota", "Camry");

for (const key in myCar) {
  console.log(key);
}
// make
// model
// describe  ← ⚠️ inherited from prototype!
```

`describe` was defined on `Vehicle.prototype`, but `for...in` walks up the prototype chain and includes it. This can cause unexpected behaviour when you're processing the object's own data.

**Fix: always use `hasOwnProperty` guard when using `for...in` on objects you don't fully control:**

```js
for (const key in myCar) {
  if (Object.prototype.hasOwnProperty.call(myCar, key)) {
    console.log(key);  // only make, model
  }
}
```

Why `Object.prototype.hasOwnProperty.call(myCar, key)` instead of `myCar.hasOwnProperty(key)`? Because if someone does `myCar.hasOwnProperty = function() { return true; }` (accidentally or maliciously), your check breaks. Using it from `Object.prototype` directly is safer.

**In modern code, prefer `Object.keys()` + `for...of`** — it only returns own enumerable keys, no prototype chain:

```js
for (const key of Object.keys(myCar)) {
  console.log(key);  // only make, model — safe
}
```

### 2. Using `for...in` on arrays — don't

```js
const scores = [95, 87, 72, 88];

for (const index in scores) {
  console.log(index, scores[index]);
}
// "0" 95
// "1" 87
// "2" 72
// "3" 88
```

Looks like it works. But the indices are **strings** (`"0"`, `"1"`, not `0`, `1`). That breaks arithmetic:

```js
for (const i in scores) {
  console.log(i + 1);  // "01", "11", "21", "31" — string concatenation!
}
```

Worse: if any library adds properties to `Array.prototype`, those appear as extra keys in your loop.

**Rule: never use `for...in` on arrays. Use `for...of` or classic `for`.**

### 3. Key order is mostly reliable but not guaranteed

In modern engines (V8, SpiderMonkey), integer-like keys come first in ascending order, then string keys in insertion order. So:

```js
const weirdObject = { b: 2, a: 1, "3": "three", "1": "one" };

for (const key in weirdObject) {
  console.log(key);
}
// 1    ← integer keys first, sorted numerically
// 3
// b    ← then string keys in insertion order
// a
```

This trips people up when they expect insertion order for all keys. For reliable ordering, use a `Map` instead of a plain object.

### 4. `for...in` on `null` or `undefined` throws

```js
const userData = null;

for (const key in userData) {  // ❌ TypeError: Cannot read properties of null
  console.log(key);
}
```

Always guard:

```js
if (userData && typeof userData === "object") {
  for (const key in userData) {
    console.log(key);
  }
}
```

### 5. Non-enumerable properties are invisible to `for...in`

```js
const product = {};
Object.defineProperty(product, "secretId", {
  value: "XYZ-999",
  enumerable: false,  // hidden from for...in
});
product.name = "Widget";

for (const key in product) {
  console.log(key);  // only "name" — "secretId" is hidden
}
```

Properties added normally (`obj.x = 1`) are enumerable. Properties added with `Object.defineProperty` and `enumerable: false` are invisible. Built-in methods like `.toString()` are also non-enumerable — that's why they don't show up when you loop an object.

---

## Common mistakes

### Mistake 1: Using `for...in` on arrays

```js
const items = ["pen", "notebook", "ruler"];

// ❌ Indices are strings, prototype pollution risk
for (const i in items) {
  console.log(i + 1);  // "01", "11", "21" — wrong!
}

// ✅ Use for...of for arrays
for (const item of items) {
  console.log(item);
}
```

### Mistake 2: Forgetting that keys are always strings

```js
const dimensions = { width: 100, height: 200 };

for (const key in dimensions) {
  // key is "width" and "height" (strings), not the values
  console.log(key * 2);  // NaN — you're doubling the KEY, not the value!
}

// ✅ Access the value with bracket notation
for (const key in dimensions) {
  console.log(dimensions[key] * 2);  // 200, 400
}
```

### Mistake 3: Using dot notation with the key variable

```js
const settings = { volume: 80, brightness: 60 };

for (const key in settings) {
  console.log(settings.key);  // ❌ undefined — looks for literal property "key"
  console.log(settings[key]); // ✅ 80, then 60
}
```

### Mistake 4: Not filtering inherited properties

```js
// If you're looping over objects you didn't create yourself, always guard:
for (const key in someExternalObject) {
  if (Object.prototype.hasOwnProperty.call(someExternalObject, key)) {
    // safe to use key here
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

You have a product object:

```js
const laptopSpec = {
  brand: "Dell",
  model: "XPS 15",
  ramGB: 16,
  storageGB: 512,
  priceUSD: 1299,
};
```

Using `for...in`:
1. Log every key–value pair in the format: `brand: Dell`
2. Count how many properties have a **number** value and log the count

---

### Exercise 2 — medium

You have two snapshots of an employee record — before and after a manager edited it:

```js
const previousRecord = {
  employeeId: "EMP-4421",
  fullName: "Jordan Casey",
  department: "Sales",
  salary: 62000,
  title: "Sales Associate",
  isActive: true,
};

const currentRecord = {
  employeeId: "EMP-4421",
  fullName: "Jordan Casey",
  department: "Marketing",
  salary: 68000,
  title: "Marketing Specialist",
  isActive: true,
};
```

Using `for...in`:
1. Find every field that changed (where the value is different between the two records)
2. For each changed field, log it as: `department changed: "Sales" → "Marketing"`
3. If nothing changed, log: `"No changes detected"`
4. At the end, log how many fields changed in total

---

### Exercise 3 — hard

You are building a **mini inventory report system**. You have a category-to-products map stored as a nested object:

```js
const warehouseInventory = {
  electronics: {
    laptops: 34,
    phones: 128,
    tablets: 57,
    headphones: 92,
  },
  furniture: {
    chairs: 15,
    desks: 8,
    shelves: 22,
    lamps: 41,
  },
  clothing: {
    jackets: 63,
    shoes: 210,
    hats: 88,
    gloves: 145,
  },
};
```

Write a function `buildInventoryReport(inventory)` that uses **nested `for...in` loops** to:

1. For each category, calculate:
   - Total units across all products in that category
   - The product with the **most stock** in that category
   - The product with the **least stock** in that category
2. Return an array of report objects, one per category:
   ```js
   {
     category: "electronics",
     totalUnits: 311,
     highestStock: { product: "phones", units: 128 },
     lowestStock: { product: "laptops", units: 34 },
   }
   ```
3. After calling `buildInventoryReport(warehouseInventory)`, loop through the results and log each in this format:
   ```
   ELECTRONICS (311 units total)
     Most stocked: phones (128)
     Least stocked: laptops (34)
   ```
4. **Bonus:** After reporting all categories, log the single most-stocked product across the **entire** warehouse and which category it belongs to.

Requirements:
- Use `for...in` for all loops over objects
- `buildInventoryReport` must work correctly regardless of how many categories or products exist

---

## Quick reference

```
for...in cheat sheet
─────────────────────────────────────────────────────
BASIC USAGE          for (const key in obj)
ACCESS VALUE         obj[key]  — bracket notation always
OWN KEYS ONLY        Object.keys(obj)  — returns array of own keys
OWN VALS ONLY        Object.values(obj)
OWN KEY+VAL          Object.entries(obj) → use with for...of

PROTOTYPE GUARD      Object.prototype.hasOwnProperty.call(obj, key)
─────────────────────────────────────────────────────
KEYS ARE STRINGS     always — even if key looks numeric
INTEGER KEYS         sorted numerically first (then insertion order)
INHERITED KEYS       included — guard with hasOwnProperty if needed
NON-ENUMERABLE       invisible to for...in
─────────────────────────────────────────────────────
for...in vs for...of  for...in  = KEYS of objects
                      for...of  = VALUES of iterables
for...in vs forEach   for...in  = objects; forEach = array methods
─────────────────────────────────────────────────────
NEVER USE ON ARRAYS  use for...of or classic for instead
```

---

## Connected topics

- **11 — for...of** — use this for arrays, strings, Sets, Maps; not for plain objects
- **13 — break and continue** — work inside `for...in` just like other loops
- **Objects (34+)** — `Object.keys()`, `Object.values()`, `Object.entries()` are often the cleaner choice
- **Prototypes & inheritance (OOP section)** — why inherited properties show up in `for...in`
- **Maps (Modern JS section)** — if you need ordered key-value pairs with guaranteed insertion order, use `Map` + `for...of` instead of an object + `for...in`
- **Object.defineProperty** — how non-enumerable properties are created and why they're invisible to `for...in`
