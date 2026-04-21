# 07 — Conditionals

## What is this?
A conditional is a way to tell your program "do this only if a certain situation is true." Think of it like a security guard at a door: they check your ID (the condition), and based on what they see, they either let you in or send you elsewhere. JavaScript gives you two main tools for this: `if/else` (for flexible, range-based decisions) and `switch` (for matching one value against many specific cases).

## Why does it matter?
Every real program makes decisions. Is the user logged in? Is the cart empty? Did the API return an error? Conditionals are the backbone of all program logic. Knowing when to use `if/else` vs `switch`, how to write clean early returns, and how to avoid deeply nested "pyramid of doom" code will make your logic readable and maintainable.

---

## Syntax

```js
// if / else if / else
if (condition) {
  // runs if condition is truthy
} else if (anotherCondition) {
  // runs if the first was falsy and this is truthy
} else {
  // runs if ALL conditions above were falsy
}

// switch
switch (expression) {
  case value1:
    // runs if expression === value1
    break;
  case value2:
    // runs if expression === value2
    break;
  default:
    // runs if no case matched
}
```

---

## How it works — line by line

- `if (condition)` — JS evaluates `condition`. If it's **truthy** (not just `true` — anything truthy), the block runs.
- `else if (anotherCondition)` — Only checked if the previous `if` was false. You can chain as many as needed.
- `else` — The catch-all. Runs only if every condition above was falsy. Optional — you don't always need one.
- `switch (expression)` — JS evaluates `expression` **once**, then compares it using **strict equality (`===`)** against each `case`.
- `break` — Exits the switch. Without it, execution **falls through** to the next case (intentional or a bug — see below).
- `default` — The switch's catch-all. Equivalent to `else`. Optional but recommended.

---

## Deep dive — everything you need to know

### 1. Conditions evaluate truthiness, not just `true`/`false`

Any value can be a condition — JS converts it to boolean:

```js
const username = "parsh";
const cartCount = 0;
const sessionToken = null;

if (username) {
  console.log("Logged in");    // runs — non-empty string is truthy
}

if (cartCount) {
  console.log("Has items");    // DOESN'T run — 0 is falsy
}

if (!sessionToken) {
  console.log("Not authenticated");   // runs — null is falsy, !null is true
}
```

Recall from topic 03: the falsy values are `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Everything else is truthy.

---

### 2. `switch` uses strict equality — no coercion

```js
const input = "1";

switch (input) {
  case 1:
    console.log("Number one");    // DOESN'T run — "1" !== 1 (strict)
    break;
  case "1":
    console.log("String one");    // runs — "1" === "1"
    break;
}
```

This is a common source of bugs when the switched value comes from a form (string) but the cases use numbers.

---

### 3. Fall-through — intentional vs accidental

Without `break`, a matched case **falls through** to the next case and runs it too, regardless of whether it matches.

**Accidental fall-through (bug):**
```js
const day = "Monday";

switch (day) {
  case "Monday":
    console.log("Start of week");   // runs
    // forgot break!
  case "Tuesday":
    console.log("Second day");      // ALSO runs — this is a bug
    break;
}
```

**Intentional fall-through (useful pattern):**
```js
const day = "Saturday";

switch (day) {
  case "Saturday":
  case "Sunday":
    console.log("Weekend!");   // runs for both Saturday and Sunday
    break;
  case "Monday":
  case "Tuesday":
  case "Wednesday":
  case "Thursday":
  case "Friday":
    console.log("Weekday");
    break;
  default:
    console.log("Unknown day");
}
```

Stacking cases without `break` between them is the clean way to handle "multiple values, same outcome."

---

### 4. `if/else` vs `switch` — when to use which

| Situation | Use |
|-----------|-----|
| Checking ranges or complex conditions (`> 90`, `=== null && isActive`) | `if/else` |
| Checking one variable against many specific exact values | `switch` |
| Only 2-3 branches | `if/else` |
| 4+ branches on the same variable | `switch` (cleaner to read) |
| Conditions involve different variables | `if/else` |
| Need fall-through grouping | `switch` |

```js
// BETTER with if/else — range check
const score = 72;
if (score >= 90)      { grade = "A"; }
else if (score >= 75) { grade = "B"; }
else if (score >= 60) { grade = "C"; }
else                  { grade = "F"; }

// BETTER with switch — exact value match
const statusCode = 404;
switch (statusCode) {
  case 200: console.log("OK");           break;
  case 201: console.log("Created");      break;
  case 400: console.log("Bad Request");  break;
  case 401: console.log("Unauthorized"); break;
  case 404: console.log("Not Found");    break;
  case 500: console.log("Server Error"); break;
  default:  console.log("Unknown");
}
```

---

### 5. Early returns — the antidote to deeply nested if/else

"Pyramid of doom" is what happens when you keep nesting conditions:

```js
// BAD — nested pyramid
function processOrder(order) {
  if (order) {
    if (order.isValid) {
      if (order.isPaid) {
        if (order.inStock) {
          console.log("Dispatching order");
        } else {
          console.log("Out of stock");
        }
      } else {
        console.log("Payment pending");
      }
    } else {
      console.log("Invalid order");
    }
  } else {
    console.log("No order");
  }
}
```

**Early returns** (guard clauses) invert the condition and return immediately, keeping the main logic at the top level:

```js
// GOOD — guard clauses + early returns
function processOrder(order) {
  if (!order)           return console.log("No order");
  if (!order.isValid)   return console.log("Invalid order");
  if (!order.isPaid)    return console.log("Payment pending");
  if (!order.inStock)   return console.log("Out of stock");

  console.log("Dispatching order");   // only reaches here if all checks passed
}
```

This is one of the most important clean code patterns. Every guard clause eliminates one level of nesting.

---

### 6. Single-line `if` — when it's acceptable vs when it's not

```js
// Acceptable — truly one action, fits on one line, easy to read
if (!user) return;
if (isDev) console.log(debugInfo);

// NOT acceptable — hiding an action without braces in a block
if (isAdmin)
  deleteAllUsers();   // this works, but the visual indentation is misleading
  logAction();        // this line is NOT in the if — it always runs! Common bug.

// ALWAYS use braces when the body is non-trivial
if (isAdmin) {
  deleteAllUsers();
  logAction();
}
```

---

### 7. `switch` with expressions and blocks

```js
// switch can use any expression — not just variables
switch (true) {
  case score >= 90:
    console.log("A");
    break;
  case score >= 75:
    console.log("B");
    break;
  default:
    console.log("C or below");
}
// switch(true) is valid but unusual — if/else is cleaner for ranges
```

```js
// Declaring variables in switch cases — requires a block
switch (status) {
  case "active": {
    const label = "Active User";     // block scope — avoids variable collision
    console.log(label);
    break;
  }
  case "banned": {
    const label = "Banned User";     // can reuse the name — different block
    console.log(label);
    break;
  }
}
```

---

### 8. Ternary as a conditional expression

Already covered in topic 04, but worth placing here in context:
```js
// Use ternary for simple, one-value decisions in expressions
const message = isLoggedIn ? "Welcome back!" : "Please log in.";

// Don't use ternary for multi-statement logic — use if/else
// WRONG
isAdmin ? (deleteUser(), logAction()) : showError();   // confusing, avoid

// RIGHT
if (isAdmin) {
  deleteUser();
  logAction();
} else {
  showError();
}
```

---

## Example 1 — basic

```js
// Access control based on user role and age
const userRole = "editor";
const userAge  = 17;
const isPremium = false;

// Role check
if (userRole === "admin") {
  console.log("Full access granted.");
} else if (userRole === "editor") {
  console.log("Edit access granted.");
} else if (userRole === "viewer") {
  console.log("Read-only access.");
} else {
  console.log("Unknown role. Access denied.");
}

// Age check using early return pattern (simulated inline)
const ageStatus = userAge >= 18 ? "Adult content allowed" : "Restricted — minors only";
console.log(ageStatus);

// Feature gate
if (!isPremium) {
  console.log("Upgrade to Premium to unlock this feature.");
}
```

---

## Example 2 — real world

```js
// HTTP response handler using switch
function handleApiResponse(statusCode, data) {
  // Guard clauses first
  if (!statusCode) return console.log("Error: No status code received.");

  switch (statusCode) {
    case 200:
    case 201:
      console.log("Success:", data);
      break;

    case 400:
      console.log("Bad request — check your input data.");
      break;

    case 401:
      console.log("Unauthorised — please log in again.");
      break;

    case 403:
      console.log("Forbidden — you don't have permission.");
      break;

    case 404:
      console.log("Not found — the resource doesn't exist.");
      break;

    case 429:
      console.log("Rate limited — slow down your requests.");
      break;

    case 500:
    case 502:
    case 503:
      console.log("Server error — try again later.");
      break;

    default:
      console.log(`Unhandled status code: ${statusCode}`);
  }
}

handleApiResponse(201, { id: 42, name: "New item" });   // "Success: { id: 42, name: 'New item' }"
handleApiResponse(404, null);                            // "Not found..."
handleApiResponse(503, null);                            // "Server error..."
```

---

## Common mistakes

### 1. Omitting `break` in switch (accidental fall-through)
```js
// WRONG — both cases run when status is "pending"
const status = "pending";
switch (status) {
  case "pending":
    console.log("Waiting for payment");   // runs
  case "failed":
    console.log("Payment failed");        // ALSO runs — missing break!
    break;
}

// RIGHT
switch (status) {
  case "pending":
    console.log("Waiting for payment");
    break;
  case "failed":
    console.log("Payment failed");
    break;
}
```

### 2. Using `=` (assignment) instead of `===` in a condition
```js
// WRONG — assigns 5 to x, then evaluates truthiness of 5 (always true)
let x = 0;
if (x = 5) {
  console.log("This always runs");    // bug — always true
}

// RIGHT
if (x === 5) {
  console.log("x is 5");
}
```

### 3. Assuming `else if` checks even when a previous branch ran
```js
// WRONG mental model — once a branch runs, the rest are skipped
const score = 95;
if (score >= 60) { console.log("C"); }      // runs (95 >= 60 is true)
if (score >= 75) { console.log("B"); }      // ALSO runs (separate if)
if (score >= 90) { console.log("A"); }      // ALSO runs (separate if)
// Output: "C", "B", "A" — wrong! Should only print "A"

// RIGHT — use else if to make branches mutually exclusive
if (score >= 90)      { console.log("A"); }
else if (score >= 75) { console.log("B"); }
else if (score >= 60) { console.log("C"); }
else                  { console.log("F"); }
// Output: "A" — correct
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `switch` compares with `===`, not `==`
```js
// This is a real gotcha with form values
const userInput = "2";   // string from a form

switch (userInput) {
  case 1: console.log("One"); break;    // WON'T match — 1 !== "2"
  case 2: console.log("Two"); break;    // WON'T match — 2 !== "2"
  case "2": console.log("Two"); break;  // MATCHES — "2" === "2"
}
// Always convert form input to the right type before switching on it
```

### Trick 2 — `if` without braces and multi-line indentation
```js
// This is the most common subtle bug for beginners — looks like both lines are in the if
if (isAdmin)
  console.log("Admin logged in");
  console.log("Redirecting...");    // always runs — NOT in the if block

// The rule: if your if body is more than one action, always use braces
```

### Trick 3 — `default` doesn't have to be last in a switch
```js
switch (action) {
  default:
    console.log("Unknown action");
    break;               // still need break even on default if it's not last
  case "start":
    console.log("Starting");
    break;
  case "stop":
    console.log("Stopping");
    break;
}
// Valid JS — though placing default last is the convention
```

### Trick 4 — Comparing to `null` with `==` catches both `null` and `undefined`
```js
// This is one of the accepted uses of loose equality
function greet(name) {
  if (name == null) {   // catches both null and undefined
    return "Hello, Guest!";
  }
  return `Hello, ${name}!`;
}

greet(null)       // "Hello, Guest!"
greet(undefined)  // "Hello, Guest!"
greet("Parsh")    // "Hello, Parsh!"
```

### Trick 5 — A `switch` with no `break` anywhere as an intentional flow pattern
```js
// "Waterfall" pattern — each case accumulates behaviour
// (unusual but valid — used in some state machines)
function getPermissions(role) {
  const perms = [];
  switch (role) {
    case "admin":
      perms.push("delete");    // falls through
    case "editor":
      perms.push("write");     // falls through
    case "viewer":
      perms.push("read");      // falls through
      break;
  }
  return perms;
}

getPermissions("admin")    // ["delete", "write", "read"]
getPermissions("editor")   // ["write", "read"]
getPermissions("viewer")   // ["read"]
```

---

## Practice exercises

### Exercise 1 — easy
Write a grade classifier. Given:
```js
const examScore = 83;
```

Using `if/else if/else`, log the grade and a message:
- 90–100 → `"A — Excellent"`
- 75–89  → `"B — Good"`
- 60–74  → `"C — Passing"`
- 40–59  → `"D — Needs work"`
- Below 40 → `"F — Failed"`

Also log whether the student passed (score >= 60).
```js
// Write your code here
```

---

### Exercise 2 — medium
Build a shipping cost calculator using `switch`. Given:

```js
const shippingZone    = "international";
const orderWeight     = 4.5;    // kg
const isPremiumMember = true;
```

Zones and base rates:
- `"local"` → $3 flat
- `"domestic"` → $5 + $1 per kg over 2kg
- `"international"` → $15 + $2 per kg over 1kg
- anything else → log error and stop

After calculating the base cost:
- Premium members get 20% off shipping
- Log the base cost, discount, and final cost with 2 decimal places

```js
// Write your code here
```

---

### Exercise 3 — hard
You're building a user authentication flow. Write the logic using **guard clauses** (early returns as `console.log` + stop the flow with a variable or flag — since we haven't covered functions in depth yet).

Given:
```js
const user = {
  username:       "parsh_dev",
  password:       "secure123",
  isEmailVerified: true,
  isBanned:        false,
  failedAttempts:  2,
  plan:            "premium"
};

const enteredPassword = "secure123";
const maxFailedAttempts = 3;
```

Implement these checks **in order**, stopping at the first failure (use a `let blocked = false` flag and wrap each check in `if (!blocked)`):
1. If `user` doesn't exist → `"Error: User not found"`
2. If account is banned → `"Access denied: Account suspended"`
3. If failed attempts >= max → `"Account locked — too many failed attempts"`
4. If password doesn't match → `"Incorrect password"` (also increment failedAttempts)
5. If email not verified → `"Please verify your email before logging in"`
6. If all pass → log a personalised welcome: `"Welcome back, parsh_dev! You are on the Premium plan."`

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**`if/else` structure:**
```js
if (condition) { }
else if (condition) { }
else { }
```

**`switch` structure:**
```js
switch (value) {
  case exact:
    // code
    break;
  default:
    // code
}
```

**Key rules:**
- `switch` uses `===` — no coercion. Match types carefully.
- Always write `break` unless fall-through is intentional.
- Stack cases without `break` for "multiple values, same outcome."
- Use guard clauses (early returns) to avoid deeply nested if/else.
- Use `if/else` for ranges and complex conditions; `switch` for exact value matching.
- Never use `=` (assignment) inside a condition — use `===`.
- Braces are optional for single-line if bodies, but always use them if there's any chance of confusion.

**Guard clause pattern:**
```js
if (!conditionA) return logError("A");
if (!conditionB) return logError("B");
// happy path here
```

**Accepted `==` use:**
```js
if (value == null) { }   // catches both null and undefined
```

---

## Connected topics
- **08 — Truthy and Falsy** — conditions evaluate truthiness, not just `true`/`false`; this is the direct next topic
- **04 — Operators** — `&&`, `||`, `!`, ternary, and comparison operators are what you put inside conditions
- **17 — Return values** — early returns in functions are the real version of the guard clause pattern
