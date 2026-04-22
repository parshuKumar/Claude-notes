# 08 — Truthy and Falsy Values

## What is this?
In JavaScript, every single value — not just booleans — has an inherent "truthiness." When JS needs to make a yes/no decision (inside an `if`, a `while`, a `&&`, or a `||`), it converts whatever value you gave it into a boolean. Values that convert to `true` are called **truthy**. Values that convert to `false` are called **falsy**. Think of it like a bouncer who doesn't check your actual ID type — they just decide "legit" or "not legit" based on what you show them.

## Why does it matter?
Truthy/falsy is the invisible engine behind every condition you write. You'll use it to write cleaner, shorter conditions — and you'll use it to avoid some of the nastiest silent bugs in JS, like treating `0` as "no value" or an empty array as "no items." Understanding this topic makes every topic from conditionals to logical operators to React rendering click into place.

---

## The complete list of falsy values — memorise these 8

There are **exactly 8 falsy values** in JavaScript. Everything else — literally everything — is truthy.

```js
false         // the boolean false
0             // the number zero
-0            // negative zero (yes, it exists)
0n            // BigInt zero
""            // empty string (single quotes, double quotes, or backticks — all falsy if empty)
null          // intentional absence of value
undefined     // unassigned value
NaN           // Not a Number
```

Every other value — including empty arrays `[]`, empty objects `{}`, the string `"0"`, the string `"false"`, negative numbers, `Infinity` — is **truthy**.

---

## Proving it with `Boolean()` and `!!`

You can convert any value to its boolean equivalent two ways:

```js
// Method 1 — Boolean() function
Boolean(0)          // false
Boolean("")         // false
Boolean(null)       // false
Boolean(undefined)  // false
Boolean(NaN)        // false
Boolean(false)      // false

Boolean(1)          // true
Boolean("hello")    // true
Boolean([])         // true  ← empty array is TRUTHY
Boolean({})         // true  ← empty object is TRUTHY
Boolean("0")        // true  ← the string "0" is TRUTHY
Boolean("false")    // true  ← the string "false" is TRUTHY
Boolean(-1)         // true
Boolean(Infinity)   // true

// Method 2 — double NOT (!!)
!!0          // false
!!""         // false
!!"hello"    // true
!![]         // true
!!{}         // true
```

`!!` is the shorthand you'll see in real codebases. `!!value` is exactly `Boolean(value)`.

---

## How JS uses truthy/falsy in conditions

Any condition in an `if`, `while`, ternary, `&&`, or `||` goes through this conversion:

```js
const cartItems  = [];      // empty array — truthy
const username   = "";      // empty string — falsy
const loginCount = 0;       // zero — falsy
const userObject = null;    // null — falsy

if (cartItems)  { console.log("Cart is truthy"); }    // runs — [] is truthy!
if (username)   { console.log("Has username"); }      // doesn't run — "" is falsy
if (loginCount) { console.log("Has logins"); }        // doesn't run — 0 is falsy
if (userObject) { console.log("Has user"); }          // doesn't run — null is falsy
```

---

## Deep dive — the traps that catch everyone

### Trap 1 — Empty array `[]` and empty object `{}` are TRUTHY

This surprises almost every JS beginner.

```js
const items = [];

if (items) {
  console.log("Items is truthy");   // THIS RUNS — [] is truthy
}

// If you want to check if an array is empty, check its LENGTH:
if (items.length === 0) {
  console.log("Cart is empty");     // correct check
}

// Or with truthy shorthand (length of 0 is falsy):
if (!items.length) {
  console.log("Cart is empty");     // also correct
}
```

```js
const config = {};

if (config) {
  console.log("Config is truthy");   // runs — {} is truthy
}

// To check if an object has no properties:
if (Object.keys(config).length === 0) {
  console.log("Config is empty");
}
```

---

### Trap 2 — The string `"0"` and `"false"` are TRUTHY

```js
const flag = "false";   // you might get this from a config file or API

if (flag) {
  console.log("Flag is truthy");    // runs — "false" is a non-empty string
}

// This bites people reading boolean flags from localStorage or query strings
const darkMode = localStorage.getItem("darkMode");   // returns "true" or "false" (strings)

if (darkMode) { }             // WRONG — "false" is truthy, always enters the block
if (darkMode === "true") { }  // RIGHT — explicit string comparison
```

---

### Trap 3 — `0` is falsy, which breaks numeric checks

```js
const userScore = 0;   // valid score — player just started

if (userScore) {
  console.log("Score:", userScore);   // doesn't run — 0 is falsy!
}

// WRONG — treats 0 as "no score"
const display = userScore || "No score yet";    // "No score yet" — wrong for a valid 0

// RIGHT — use ?? which only triggers on null/undefined
const display = userScore ?? "No score yet";    // 0 — correct

// Or check explicitly:
if (typeof userScore === "number") {
  console.log("Score:", userScore);   // runs — type check, not truthiness
}
```

---

### Trap 4 — `NaN` is falsy but you shouldn't rely on it for validation

```js
const parsed = parseInt("hello");   // NaN

if (!parsed) {
  console.log("Invalid input");     // runs — NaN is falsy
}

// BUT — this also triggers for 0:
const parsed2 = parseInt("0");      // 0
if (!parsed2) {
  console.log("Invalid input");     // runs — 0 is also falsy! Bug.
}

// RIGHT — use Number.isNaN() for NaN checks
if (Number.isNaN(parsed)) {
  console.log("Not a valid number");
}
```

---

### Trap 5 — Negative zero (`-0`) is falsy

```js
const velocity = -0;

if (!velocity) {
  console.log("Not moving");   // runs — -0 is falsy
}

// -0 === 0 is true in JS
console.log(-0 === 0);           // true
console.log(Object.is(-0, 0));   // false — Object.is distinguishes -0 from 0
// You'll rarely need this, but good to know it exists
```

---

### Trap 6 — `document.getElementById()` returns `null` when not found — guard against it

```js
// This pattern is everywhere in DOM code
const button = document.getElementById("submitBtn");

// Without guard — TypeError if button doesn't exist
button.addEventListener("click", handler);   // TypeError: Cannot read properties of null

// With truthy guard
if (button) {
  button.addEventListener("click", handler);   // safe
}

// Or modern optional chaining (topic 39):
button?.addEventListener("click", handler);   // safe, no error if null
```

---

## Truthy/falsy in logical operators — the full picture

This ties together topics 04 and this topic. `&&` and `||` don't return `true`/`false` — they return one of their operands after evaluating truthiness.

```js
// || — returns the first TRUTHY value, or the last value if all are falsy
""        || "default"      // "default"  — "" is falsy, return next
0         || 42             // 42         — 0 is falsy, return next
null      || "fallback"     // "fallback"
"Parsh"   || "Guest"        // "Parsh"    — truthy, return immediately
false     || 0 || ""        // ""         — all falsy, return last

// && — returns the first FALSY value, or the last value if all are truthy
""        && "value"        // ""         — falsy, stop immediately
"Parsh"   && "Admin"        // "Admin"    — both truthy, return last
null      && doSomething()  // null       — short-circuits, doSomething never called
"user"    && user.getName() // user.getName() — left is truthy, evaluate right
```

**Real-world patterns:**
```js
// Default value with ||
const displayName = user.nickname || user.fullName || "Anonymous";

// Guard with &&
const firstName = user && user.profile && user.profile.firstName;

// These two lines do the same thing in different situations:
const label1 = count || "None";    // use when 0 means "no value"
const label2 = count ?? "None";    // use when 0 is a valid value
```

---

## Using `!` to convert truthy/falsy to boolean

```js
const username = "";

!username         // true  — "" is falsy, NOT flips to true
!!username        // false — double NOT gives the actual boolean equivalent

// Common pattern: convert a value to a clean boolean
const hasUsername = !!username;        // false
const hasItems    = !!cart.length;     // false if length is 0, true otherwise
const isLoggedIn  = !!sessionToken;    // false if null/undefined, true otherwise
```

---

## Example 1 — basic

```js
// Testing each falsy value and some truthy surprises
const testValues = [
  false, 0, -0, 0n, "", null, undefined, NaN,  // all falsy
  "0", "false", [], {}, -1, Infinity, " "       // all truthy (surprises!)
];

for (const val of testValues) {
  console.log(`${String(val).padEnd(12)} → ${val ? "TRUTHY" : "FALSY"}`);
}

// Output:
// false        → FALSY
// 0            → FALSY
// 0            → FALSY   (-0 displays as 0)
// 0            → FALSY   (0n displays as 0)
//              → FALSY   (empty string)
// null         → FALSY
// undefined    → FALSY
// NaN          → FALSY
// 0            → TRUTHY  (string "0")
// false        → TRUTHY  (string "false")
//              → TRUTHY  ([] displays as empty)
// [object Obje → TRUTHY  ({})
// -1           → TRUTHY
// Infinity     → TRUTHY
//              → TRUTHY  (space " ")
```

---

## Example 2 — real world

```js
// A user profile renderer — handling missing/empty data gracefully
const userProfile = {
  displayName:  "",           // user cleared their name — falsy
  bio:          null,         // never set — falsy
  followerCount: 0,           // real value — falsy, but valid!
  tags:         [],           // no tags yet — truthy (empty array!)
  website:      "https://parsh.dev"  // set — truthy
};

// WRONG — all falsy checks, but followerCount and tags behave unexpectedly
const nameDisplay      = userProfile.displayName  || "No name set";      // "No name set" ✓
const bioDisplay       = userProfile.bio          || "No bio yet";       // "No bio yet"  ✓
const followerDisplay  = userProfile.followerCount || "No followers";    // "No followers" ✗ — 0 is valid!
const hasTagsDisplay   = userProfile.tags         ? "Has tags" : "No tags"; // "Has tags" ✗ — array is truthy even if empty!

// RIGHT — use the right tool for each case
const followerFixed    = userProfile.followerCount ?? "No followers";    // 0 ✓
const hasTagsFixed     = userProfile.tags.length > 0 ? "Has tags" : "No tags";  // "No tags" ✓
const websiteDisplay   = userProfile.website || "No website";            // "https://parsh.dev" ✓

console.log({ nameDisplay, bioDisplay, followerFixed, hasTagsFixed, websiteDisplay });
```

---

## Common mistakes

### 1. Checking an array's existence instead of its contents
```js
// WRONG
const orders = [];
if (orders) {
  renderOrders(orders);   // runs even with empty array — renders nothing, looks broken
}

// RIGHT
if (orders.length > 0) {
  renderOrders(orders);
}
```

### 2. Using `||` when `??` is needed for numeric values
```js
// WRONG — 0 is falsy, so || replaces it
const retryLimit = config.retries || 3;   // if config.retries is 0, you get 3 — wrong

// RIGHT
const retryLimit = config.retries ?? 3;   // 0 stays 0, only null/undefined get replaced
```

### 3. Reading a boolean from localStorage and trusting truthiness
```js
// WRONG — localStorage always stores strings
localStorage.setItem("notifications", "false");
const notif = localStorage.getItem("notifications");   // "false" — a string

if (notif) {
  enableNotifications();   // runs! "false" (string) is truthy
}

// RIGHT
if (notif === "true") {
  enableNotifications();
}
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `0` in JSX/React renders nothing (it's falsy)
```js
// In React (just awareness for now):
const count = 0;
return <div>{count && <Badge />}</div>
// Renders "0" on the page — not nothing — because 0 is returned as a value
// Fix: use ternary instead
return <div>{count > 0 ? <Badge /> : null}</div>
// This is why React devs use ternary for JSX conditions, not &&
```

### Trick 2 — `document.all` is falsy (historical quirk)
```js
Boolean(document.all)   // false — despite being an object!
// document.all is the only object in JS that is falsy
// This exists for legacy browser-detection code compatibility
// You'll never use it — just don't be surprised if you see it mentioned
```

### Trick 3 — `Object.is()` for strict sameness including -0 and NaN
```js
Object.is(NaN, NaN)    // true  — unlike ===
Object.is(-0, 0)       // false — unlike ===
Object.is(1, 1)        // true  — same as ===
// Object.is() is the "true equality" — used internally by React to detect state changes
```

### Trick 4 — Truthy/falsy in `while` loops
```js
// Loop until value is falsy — useful for consuming a queue
const messageQueue = ["msg1", "msg2", "msg3"];

while (messageQueue.length) {          // 3 → 2 → 1 → 0 (falsy) → stops
  const msg = messageQueue.shift();    // removes and returns first item
  console.log("Processing:", msg);
}
// Cleaner than: while (messageQueue.length > 0)
```

### Trick 5 — Truthy check on a function return value
```js
// APIs often return null on failure — guard pattern
function getUserById(id) {
  const users = { 1: "Parsh", 2: "Amit" };
  return users[id] || null;   // returns the name, or null if not found
}

const user = getUserById(3);
if (user) {
  console.log(`Found: ${user}`);
} else {
  console.log("User not found");   // runs — null is falsy
}
```

---

## Practice exercises

### Exercise 1 — easy
Without running the code, predict `TRUTHY` or `FALSY` for each value. Then verify:

```js
const values = [
  0, 1, -1, "", " ", "0", "false",
  null, undefined, NaN, [], {}, false, true,
  Infinity, -Infinity, 0n, 1n
];
```

Write a loop that logs each value and whether it's truthy or falsy. Mark which ones surprised you.
```js
// Write your code here
```

---

### Exercise 2 — medium
You're rendering a user's profile card. Given:

```js
const profile = {
  name:       "",
  age:        0,
  bio:        null,
  website:    undefined,
  followers:  0,
  following:  142,
  posts:      [],
  isPremium:  false
};
```

For each field, write the logic that:
1. `name` — show `"Anonymous"` if empty
2. `age` — show `"Age not set"` only if truly not provided (null/undefined), but show `0` if age is actually 0
3. `bio` — show `"No bio"` if null or undefined
4. `website` — show `"No website"` if not set
5. `followers` — show `"0 followers"` correctly (not replaced by a fallback)
6. `posts` — show `"No posts yet"` if the array is empty (not just falsy)
7. `isPremium` — show `"Free plan"` or `"Premium"` using a ternary — but isPremium is `false`, not `null`

Log all 7 values with a label.
```js
// Write your code here
```

---

### Exercise 3 — hard
You are building a form validator. Given raw form data (everything is a string or missing):

```js
const formData = {
  username:  "parsh_dev",
  email:     "",
  age:       "0",
  password:  "abc",
  terms:     "false",   // string, from a checkbox that wasn't ticked
  referral:  null
};
```

Write validation logic that checks each field and collects error messages into an array:

1. `username` — required, must be at least 3 characters
2. `email` — required (not empty string)
3. `age` — must be a number >= 18. Parse it first. Treat "0" as age 0, not as falsy.
4. `password` — must be at least 8 characters
5. `terms` — must be the **string** `"true"` (don't use truthiness — `"false"` is truthy!)
6. `referral` — optional, skip validation if null or undefined

After collecting all errors:
- If the errors array has items, log each error
- If no errors, log `"Form is valid — submitting..."`

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**The 8 falsy values — memorise these:**
```
false, 0, -0, 0n, "", null, undefined, NaN
```

**Common truthy surprises:**
```
"0"        → truthy  (non-empty string)
"false"    → truthy  (non-empty string)
[]         → truthy  (empty array is an object)
{}         → truthy  (empty object)
-1         → truthy  (non-zero number)
Infinity   → truthy
" "        → truthy  (space is not empty)
```

**Conversion tools:**
```
Boolean(x)   → explicit boolean
!!x          → shorthand for Boolean(x)
!x           → inverted boolean
```

**When to use what:**
```
if (x)            → is x truthy? (simple existence check)
if (x == null)    → is x null OR undefined?
if (x === null)   → is x exactly null?
if (x === "")     → is x an empty string?
if (x.length > 0) → does array/string have items?
if (Number.isNaN(x)) → is x NaN?
x || default      → fallback for any falsy value
x ?? default      → fallback only for null/undefined
```

**Key rules:**
- Empty array `[]` and empty object `{}` are **truthy** — check `.length` or `Object.keys().length`
- `"0"` and `"false"` are **truthy** — always compare strings explicitly with `===`
- `0` is **falsy** — use `??` not `||` when 0 is a valid value
- `NaN` is **falsy** — but use `Number.isNaN()` for explicit NaN checks
- `document.all` is the only object that is falsy — historical quirk, ignore in practice

---

## Connected topics
- **07 — Conditionals** — every `if` condition relies on truthy/falsy evaluation
- **04 — Operators** — `&&`, `||`, `!`, `??` all use truthy/falsy to decide what to return
- **03 — Type Coercion** — truthy/falsy is JS converting a value to boolean — direct application of coercion
