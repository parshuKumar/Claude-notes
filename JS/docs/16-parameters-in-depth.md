# 16 — Parameters in Depth

## What is this?

Parameters are the variables that receive values when a function is called. JavaScript gives you several powerful ways to define them beyond the basic `function fn(a, b)` syntax:

- **Default parameters** — fallback values when an argument is missing
- **Rest parameters** — collect unlimited extra arguments into an array
- **Destructured parameters** — unpack objects or arrays directly in the function signature
- **Parameter validation** — guarding against bad input at the function boundary

```js
// All four in one function
function createOrder({ productName, quantity = 1, ...extras }, ...tags) {
  // productName and quantity destructured from first arg
  // quantity defaults to 1
  // extras collects remaining object properties
  // tags collects all additional arguments
}
```

---

## Why does it matter?

Real-world functions rarely receive simple `(a, b)` arguments. APIs pass objects. Components receive config objects. Utility functions need to handle a variable number of inputs.

Without these features you end up writing defensive boilerplate inside every function:

```js
// Without modern parameter syntax
function createUser(options) {
  const name = options.name;
  const role = options.role !== undefined ? options.role : "viewer";
  const age = options.age;
  // ... just to unpack the arguments
}

// With modern parameter syntax
function createUser({ name, role = "viewer", age }) {
  // unpacked and defaulted in the signature itself
}
```

Learning this properly means your function signatures become self-documenting — a reader can see the shape of the expected input just by looking at the header.

---

## Default Parameters

### Syntax

```js
function functionName(param = defaultValue) { }
```

### How they work

The default is used when the argument is `undefined` — either not passed at all, or explicitly passed as `undefined`.

```js
function sendNotification(message, priority = "normal", retries = 3) {
  console.log(`[${priority.toUpperCase()}] ${message} (retries: ${retries})`);
}

sendNotification("Server down");
// [NORMAL] Server down (retries: 3)

sendNotification("Payment failed", "high");
// [HIGH] Payment failed (retries: 3)

sendNotification("Disk warning", "low", 1);
// [LOW] Disk warning (retries: 1)

sendNotification("Cache miss", undefined, 5);
// [NORMAL] Cache miss (retries: 5)  ← undefined triggers the default
```

### Defaults can reference earlier parameters

```js
function createRect(width, height = width) {
  return { width, height, area: width * height };
}

console.log(createRect(5));     // { width: 5, height: 5, area: 25 }  — square
console.log(createRect(5, 3));  // { width: 5, height: 3, area: 15 }
```

Later default values can reference earlier parameters. Earlier ones cannot reference later ones.

### Defaults can be expressions or function calls

```js
const getDefaultTimeout = () => 5000;

function fetchData(url, timeout = getDefaultTimeout()) {
  console.log(`Fetching ${url} with timeout ${timeout}ms`);
}

fetchData("/api/users");          // timeout = 5000
fetchData("/api/posts", 10000);   // timeout = 10000
```

The default expression runs fresh every time it's needed, not once at definition time.

### The old pattern you'll see in legacy code

```js
// Old ES5 pattern — still appears in older codebases
function greet(name) {
  name = name || "Guest";  // ⚠️ problem: 0, false, "" also trigger the default
  return "Hello, " + name;
}

// ✅ Modern — only undefined triggers default
function greet(name = "Guest") {
  return "Hello, " + name;
}
```

The `||` pattern has a falsy trap — passing `0` or `""` or `false` would incorrectly use the default. The `=` default syntax only triggers on `undefined`.

---

## Rest Parameters

### Syntax

```js
function functionName(param1, param2, ...restName) {
  // restName is a real array of all remaining arguments
}
```

### How they work

```js
function logActivity(userId, action, ...details) {
  console.log(`User ${userId} performed: ${action}`);
  console.log("Details:", details);
}

logActivity("U-001", "login");
// User U-001 performed: login
// Details: []

logActivity("U-002", "purchase", "item-47", 89.99, "promo-applied");
// User U-002 performed: purchase
// Details: ["item-47", 89.99, "promo-applied"]
```

`...details` collects everything after `action` into a real `Array`. Unlike the old `arguments` object, rest parameters:
- Are a proper `Array` (have `.map()`, `.filter()`, etc.)
- Only collect the remaining arguments, not all of them
- Work in arrow functions

### Rules

- Must be the **last** parameter — `function fn(a, ...b, c)` is a SyntaxError
- Only **one** rest parameter per function
- Gets an empty array `[]` if no extra args are passed

### Replacing `arguments`

```js
// Old pattern — arguments is not a real array
function sumOld() {
  return Array.from(arguments).reduce((acc, n) => acc + n, 0);
}

// ✅ Modern — rest parameter is already a real array
const sum = (...numbers) => numbers.reduce((acc, n) => acc + n, 0);

console.log(sum(1, 2, 3, 4, 5));  // 15
console.log(sum(10, 20));          // 30
```

### Combining fixed and rest parameters

```js
function createPlaylist(playlistName, creatorId, ...trackIds) {
  return {
    name: playlistName,
    creator: creatorId,
    tracks: trackIds,
    trackCount: trackIds.length,
  };
}

const playlist = createPlaylist("Morning Focus", "U-88", "T-01", "T-02", "T-03");
console.log(playlist);
// {
//   name: "Morning Focus",
//   creator: "U-88",
//   tracks: ["T-01", "T-02", "T-03"],
//   trackCount: 3
// }
```

---

## Destructured Parameters

### Object destructuring in parameters

Instead of receiving a whole object and then picking properties off it, you destructure directly in the signature:

```js
// Without destructuring
function displayUserCard(user) {
  console.log(`${user.firstName} ${user.lastName} — ${user.role}`);
}

// With destructuring
function displayUserCard({ firstName, lastName, role }) {
  console.log(`${firstName} ${lastName} — ${role}`);
}

const user = { firstName: "Mia", lastName: "Tanaka", role: "editor", id: 42 };
displayUserCard(user);  // "Mia Tanaka — editor"
```

The extra `id` property is simply ignored — you only extract what you name.

### With defaults inside destructuring

```js
function renderProductCard({ name, price, currency = "USD", inStock = true }) {
  const status = inStock ? "In Stock" : "Out of Stock";
  return `${name} — ${currency} ${price.toFixed(2)} [${status}]`;
}

console.log(renderProductCard({ name: "Headphones", price: 149.99 }));
// "Headphones — USD 149.99 [In Stock]"

console.log(renderProductCard({ name: "Cable", price: 9.99, inStock: false }));
// "Cable — USD 9.99 [Out of Stock]"
```

### Renaming while destructuring

```js
// 'status' in the API response should be named 'orderStatus' in our code
function processApiOrder({ id, status: orderStatus, total: orderTotal }) {
  console.log(`Order ${id}: ${orderStatus} — $${orderTotal}`);
}

processApiOrder({ id: "O-123", status: "shipped", total: 89.99 });
// "Order O-123: shipped — $89.99"
```

The syntax `status: orderStatus` means "take the `status` property, call it `orderStatus` locally."

### Default value for the entire parameter

When calling code might pass `undefined` for the whole argument, provide a default for the destructuring itself:

```js
function getUserPermissions({ role = "viewer", canEdit = false } = {}) {
  return { role, canEdit };
}

getUserPermissions({ role: "admin", canEdit: true });  // ✅
getUserPermissions({});                                 // ✅ role="viewer", canEdit=false
getUserPermissions();                                   // ✅ uses {} as default — no crash
```

Without `= {}` at the end, calling `getUserPermissions()` with no argument would throw `TypeError: Cannot destructure property 'role' of undefined`.

### Array destructuring in parameters

```js
function getFirstAndLast([first, ...rest]) {
  const last = rest[rest.length - 1];
  return { first, last, total: rest.length + 1 };
}

console.log(getFirstAndLast(["Monday", "Tuesday", "Wednesday", "Thursday"]));
// { first: "Monday", last: "Thursday", total: 4 }
```

---

## Tricky things you'll encounter in the real world

### 1. `null` does NOT trigger a default parameter — only `undefined` does

```js
function greet(name = "Guest") {
  return `Hello, ${name}`;
}

console.log(greet(undefined));  // "Hello, Guest"  ← default used
console.log(greet(null));       // "Hello, null"   ← null is NOT undefined!
```

This surprises people. If you fetch user data and a field comes back as `null`, the default parameter won't kick in. You'll need to handle `null` explicitly:

```js
function greet(name = "Guest") {
  const displayName = name ?? "Guest";  // ?? handles null AND undefined
  return `Hello, ${displayName}`;
}
```

### 2. Destructuring a parameter that might be `undefined` crashes without a default

```js
// ❌ Crashes if called with no argument
function init({ debug, verbose }) {
  console.log(debug, verbose);
}
init();  // TypeError: Cannot destructure property 'debug' of undefined

// ✅ Safe — default the whole parameter to {}
function init({ debug = false, verbose = false } = {}) {
  console.log(debug, verbose);
}
init();           // false false
init({ debug: true });  // true false
```

This is one of the most common patterns in real JS code — especially for options/config parameters.

### 3. Rest parameter collects the REST, not all

```js
function fn(first, second, ...others) {
  console.log(first);   // "a"
  console.log(second);  // "b"
  console.log(others);  // ["c", "d", "e"]  — only what's left
}

fn("a", "b", "c", "d", "e");
```

`others` doesn't contain `first` and `second` — those were already captured. This is different from the old `arguments` which contained every argument including the named ones.

### 4. Passing an object to a destructuring parameter doesn't require all properties

```js
function configure({ theme, fontSize, language }) {
  console.log(theme, fontSize, language);
}

// Extra properties are silently ignored
configure({ theme: "dark", fontSize: 14, language: "en", debug: true, version: 2 });
// "dark" 14 "en"

// Missing properties become undefined (or use the default if one exists)
configure({ theme: "light" });
// "light" undefined undefined
```

### 5. Default parameters and the temporal dead zone

```js
function fn(a = 1, b = a * 2) {
  return [a, b];
}
console.log(fn());       // [1, 2]
console.log(fn(5));      // [5, 10]
console.log(fn(5, 3));   // [5, 3]  — explicit b overrides the default

// ❌ But you cannot reference a later param in an earlier default
function fn(a = b, b = 2) { }  // ReferenceError — b is in TDZ when a's default runs
```

### 6. Spread vs rest — same syntax, opposite direction

```js
// REST — in function DEFINITION — collects into an array
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}

// SPREAD — in function CALL — expands an array into arguments
const scores = [90, 85, 92, 88];
console.log(Math.max(...scores));  // 92

// Both together
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
const values = [1, 2, 3];
console.log(sum(...values));  // 6
```

They use the same `...` syntax but do the opposite thing depending on context. This confuses people at first.

---

## Common mistakes

### Mistake 1: Putting rest parameter before other parameters

```js
// ❌ SyntaxError — rest must be last
function fn(...args, lastParam) { }

// ✅
function fn(firstParam, ...args) { }
```

### Mistake 2: Thinking `null` triggers a default

```js
// ❌ null slips through
function render(title = "Untitled") {
  return `<h1>${title}</h1>`;
}
render(null);  // "<h1>null</h1>" — not "<h1>Untitled</h1>"

// ✅ Handle null explicitly
function render(title = "Untitled") {
  const safeTitle = title ?? "Untitled";
  return `<h1>${safeTitle}</h1>`;
}
```

### Mistake 3: Forgetting `= {}` default on destructured config parameter

```js
// ❌ Crashes when called with no argument
function init({ port, host }) {
  console.log(port, host);
}
init();  // TypeError

// ✅
function init({ port = 3000, host = "localhost" } = {}) {
  console.log(port, host);
}
init();  // 3000 "localhost"
```

### Mistake 4: Confusing rest (`...` in definition) with spread (`...` in call)

```js
// In a function DEFINITION — collects into array (REST)
function fn(...args) { console.log(args); }

// In a function CALL — expands into individual args (SPREAD)
fn(...[1, 2, 3]);  // same as fn(1, 2, 3)
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `formatGreeting` with these parameters:
- `firstName` (required)
- `lastName` (required)
- `title` — defaults to `""` (empty string)
- `language` — defaults to `"en"`

The function returns a greeting string:
- If `language` is `"en"`: `"Hello, Dr. Sarah Chen"` (or `"Hello, Sarah Chen"` if no title)
- If `language` is `"es"`: `"Hola, Sarah Chen"`
- If `language` is `"fr"`: `"Bonjour, Sarah Chen"`

Test it with at least 5 different combinations of arguments, including cases where defaults kick in.

---

### Exercise 2 — medium

Write a function `buildApiRequest` that accepts a **single config object** with these properties:
- `endpoint` (required)
- `method` — defaults to `"GET"`
- `timeout` — defaults to `5000`
- `headers` — defaults to `{}`
- `body` — defaults to `null`

The function should:
1. Validate that `endpoint` is a non-empty string — if not, throw `new Error("endpoint is required")`
2. Validate that `method` is one of `"GET"`, `"POST"`, `"PUT"`, `"DELETE"` — if not, throw `new Error("Invalid method")`
3. Return a request config object:
   ```js
   {
     url: endpoint,
     method: method.toUpperCase(),
     timeout,
     headers: { "Content-Type": "application/json", ...headers },
     body,
   }
   ```

Also add a default for the whole parameter `= {}` so calling with no argument doesn't crash before the validation runs.

Test:
```js
console.log(buildApiRequest({ endpoint: "/api/users" }));
console.log(buildApiRequest({ endpoint: "/api/orders", method: "post", timeout: 10000 }));
console.log(buildApiRequest({ endpoint: "/api/login", method: "PATCH" })); // should throw
```

---

### Exercise 3 — hard

Build a **report generator function** with the following signature:

```js
function generateSalesReport(
  reportTitle,
  { startDate, endDate, currency = "USD", includeVAT = false } = {},
  ...salesRecords
)
```

Where each `salesRecord` is:
```js
{ region: "North", rep: "Dana Cole", amount: 14200, units: 45 }
```

The function should:
1. Validate that `reportTitle` is a non-empty string
2. Validate that at least one `salesRecord` was passed — if not, return `{ error: "No sales data provided" }`
3. Calculate from the `salesRecords`:
   - Total amount across all records
   - Total units across all records
   - Highest single sale (by amount) — include the `rep` name
   - Average sale amount (rounded to 2 decimal places)
4. If `includeVAT` is `true`, add 20% to the total amount
5. Return a report object:
   ```js
   {
     title: reportTitle,
     period: `${startDate} to ${endDate}`,  // use "N/A" if dates are missing
     currency,
     totalAmount: 42850.00,
     totalUnits: 134,
     averageSale: 10712.50,
     topSale: { rep: "Dana Cole", amount: 14200 },
     vatIncluded: false,
   }
   ```

Test with:
```js
const report = generateSalesReport(
  "Q1 North Region",
  { startDate: "2026-01-01", endDate: "2026-03-31", includeVAT: true },
  { region: "North", rep: "Dana Cole",  amount: 14200, units: 45 },
  { region: "North", rep: "Eli Torres",  amount: 9800,  units: 31 },
  { region: "North", rep: "Pita Havili", amount: 18850, units: 58 }
);
console.log(report);
```

---

## Quick reference

```
Parameters in depth — cheat sheet
─────────────────────────────────────────────────────
DEFAULT PARAMS   function fn(x, y = 10) { }
                 triggered by: undefined (NOT null)
                 can reference earlier params: function fn(a, b = a * 2)
                 can be expressions/function calls

REST PARAMS      function fn(a, b, ...rest) { }
                 rest is a real Array
                 must be LAST parameter
                 one per function
                 replaces the old 'arguments' object

DESTRUCTURED     function fn({ name, role = "viewer" }) { }
                 rename: function fn({ status: orderStatus }) { }
                 default whole param: function fn({ x } = {}) { }
                 array: function fn([first, ...rest]) { }
─────────────────────────────────────────────────────
... in DEFINITION  = REST — collects into array
... in CALL        = SPREAD — expands array into args
─────────────────────────────────────────────────────
null ≠ undefined   null does NOT trigger default params
safe pattern       ({ option = default } = {}) for optional config objects
─────────────────────────────────────────────────────
```

---

## Connected topics

- **14 — Function Declarations and Expressions** — where parameters are introduced
- **15 — Arrow Functions** — arrow functions use all the same parameter syntax; no `arguments` object → rest params required
- **17 — Return Values** — what functions send back; pairs with parameters as the input/output model
- **Destructuring (Modern JS section)** — same `{ }` and `[ ]` syntax, in assignment context
- **Spread operator (Modern JS section)** — the other side of `...`: expanding arrays into arguments
- **Objects (34+)** — destructuring parameters mirrors how you work with objects
