# 04 — Operators

## What is this?
Operators are the symbols that tell JavaScript what to *do* with values — add them, compare them, combine conditions, pick between them. Think of operators like the verbs in a sentence: without them, you just have a list of nouns (values) with no action. JavaScript has several categories of operators, and each one has rules about types, precedence, and coercion that determine the exact result.

## Why does it matter?
You use operators in every single line of logic you write. Understanding how they work — especially the non-obvious ones like `??`, `||`, `&&` for value selection, and the ternary for inline decisions — separates code that just works from code that is clean, predictable, and bug-free.

---

## Categories of operators

| Category | Symbols | Purpose |
|----------|---------|---------|
| Arithmetic | `+` `-` `*` `/` `%` `**` | Math on numbers |
| Assignment | `=` `+=` `-=` `*=` `/=` `%=` `**=` | Assign and update values |
| Comparison | `==` `!=` `===` `!==` `>` `<` `>=` `<=` | Compare values, return boolean |
| Logical | `&&` `\|\|` `!` | Combine or negate conditions |
| Ternary | `condition ? a : b` | Inline if/else |
| Nullish coalescing | `??` | Fallback only for null/undefined |
| Optional chaining | `?.` | Safe property access (topic 39, introduced here) |
| typeof / instanceof | `typeof` `instanceof` | Type checking |
| Comma | `,` | Evaluate multiple expressions |
| Spread / Rest | `...` | Topics 32/33 — introduced here for context |

---

## Syntax

```js
// Arithmetic
const sum        = 10 + 3;     // 13
const remainder  = 10 % 3;     // 1  — modulo (leftover after division)
const power      = 2 ** 8;     // 256 — exponentiation

// Assignment shorthand
let score = 100;
score += 50;     // score = score + 50 → 150
score -= 20;     // score = score - 20 → 130
score *= 2;      // score = score * 2  → 260
score /= 4;      // score = score / 4  → 65
score **= 2;     // score = score ** 2 → 4225
score %= 100;    // score = score % 100 → 25

// Comparison
5 === 5          // true
5 !== "5"        // true  — strict not-equal
10 >= 10         // true
"b" > "a"        // true  — string comparison is alphabetical (lexicographic)

// Logical
true && false    // false — AND: both must be true
true || false    // true  — OR: at least one must be true
!true            // false — NOT: flips the boolean

// Ternary
const status = score > 50 ? "Pass" : "Fail";

// Nullish coalescing
const displayName = userName ?? "Guest";   // "Guest" only if userName is null or undefined
```

---

## How it works — line by line

- `10 % 3` → Divides 10 by 3 (result: 3 remainder **1**). Returns the remainder. Use it to check even/odd, wrap around arrays, etc.
- `2 ** 8` → 2 to the power of 8. Cleaner than `Math.pow(2, 8)`.
- `score += 50` → Shorthand for `score = score + 50`. All compound assignment operators follow this pattern.
- `"b" > "a"` → Strings are compared character by character using Unicode code points. `"b"` (98) > `"a"` (97) → `true`. This behaves unexpectedly with numbers stored as strings.
- `true && false` → Returns `false`. Both sides must be truthy for AND to return truthy.
- `true || false` → Returns `true`. Only one side needs to be truthy for OR to return truthy.
- `!true` → Flips the boolean. `!false` → `true`, `!0` → `true`, `!""` → `true`.
- `condition ? a : b` → If condition is truthy, evaluates and returns `a`. Otherwise returns `b`. Inline if/else.
- `userName ?? "Guest"` → Returns `userName` unless it's `null` or `undefined`. If it is, returns `"Guest"`.

---

## Deep dive — the things that actually trip people up

### 1. `&&` and `||` don't just return `true`/`false` — they return VALUES

This is the most commonly misunderstood thing about logical operators. `&&` and `||` don't return booleans — they return one of their **operands**.

**`||` — returns the first TRUTHY value it finds, or the last value if none are truthy:**
```js
"Parsh" || "Guest"    // "Parsh"    — "Parsh" is truthy, stop and return it
""      || "Guest"    // "Guest"    — "" is falsy, keep going, return "Guest"
null    || "Guest"    // "Guest"    — null is falsy
0       || 42         // 42         — 0 is falsy
false   || false      // false      — nothing truthy, return the last value
```

**`&&` — returns the first FALSY value it finds, or the last value if all are truthy:**
```js
"Parsh" && "Admin"    // "Admin"    — "Parsh" is truthy, keep going, return "Admin"
""      && "Admin"    // ""         — "" is falsy, stop and return it
null    && "Admin"    // null       — null is falsy, stop immediately
true    && 42         // 42         — true is truthy, keep going, return 42
```

This is called **short-circuit evaluation** — JS stops evaluating as soon as the result is determined.

**Real-world use of `||` for defaults:**
```js
const userCity = savedCity || "Mumbai";   // use saved city, fall back to Mumbai
```

**Real-world use of `&&` to guard against null:**
```js
const firstName = user && user.firstName;  // only access .firstName if user exists
// If user is null, returns null. If user exists, returns user.firstName.
```

---

### 2. `??` (Nullish Coalescing) vs `||` — they are NOT the same

This is a critical distinction:

- `||` uses falsy check — triggers for `false`, `0`, `""`, `null`, `undefined`, `NaN`
- `??` uses nullish check — triggers **only** for `null` and `undefined`

```js
const itemCount = 0;

const display1 = itemCount || "No items";   // "No items" — WRONG! 0 is falsy
const display2 = itemCount ?? "No items";   // 0          — CORRECT! 0 is not null/undefined
```

```js
const temperature = 0;
const tempDisplay = temperature || "Unknown";   // "Unknown" — BUG: 0°C is a valid temp
const tempFixed   = temperature ?? "Unknown";   // 0         — correct
```

**Rule:** Use `??` when `0`, `false`, or `""` are valid values that should NOT trigger the fallback.

---

### 3. Ternary operator — inline decisions

The ternary is a compact if/else that returns a value. It's not just shorthand — it's an **expression** (it produces a value), while `if/else` is a **statement** (it doesn't).

```js
// Syntax: condition ? valueIfTrue : valueIfFalse

const age          = 20;
const accessLevel  = age >= 18 ? "Adult" : "Minor";   // "Adult"

// Nested ternary — valid but use sparingly (hurts readability)
const score     = 72;
const grade     = score >= 90 ? "A"
                : score >= 75 ? "B"
                : score >= 60 ? "C"
                : "F";
// "C"

// Using ternary inside a template literal
const itemCount = 3;
console.log(`You have ${itemCount} item${itemCount === 1 ? "" : "s"} in your cart.`);
// "You have 3 items in your cart."
```

**When NOT to use ternary:** When the logic has side effects or is complex. Use `if/else` then.
```js
// BAD — ternary with side effects, hard to read
isAdmin ? deleteUser() : showError();

// BETTER
if (isAdmin) {
  deleteUser();
} else {
  showError();
}
```

---

### 4. Operator precedence — which runs first?

JS evaluates operators in a specific order, just like BODMAS in maths.

```js
2 + 3 * 4       // 14 — multiplication before addition
(2 + 3) * 4     // 20 — parentheses override precedence

true || false && false   // true — && runs before ||
// Equivalent to: true || (false && false) → true || false → true
```

**Precedence order (high to low, simplified):**
1. `()` — parentheses
2. `**` — exponentiation (right to left)
3. `*` `/` `%` — multiplication, division, modulo
4. `+` `-` — addition, subtraction
5. `>` `<` `>=` `<=` — comparison
6. `===` `!==` `==` `!=` — equality
7. `&&` — logical AND
8. `||` — logical OR
9. `??` — nullish coalescing
10. `? :` — ternary
11. `=` `+=` etc — assignment (right to left)

**Rule:** When in doubt, use parentheses to be explicit.

---

### 5. The `%` (modulo) operator — more useful than it looks

```js
// Check if a number is even or odd
10 % 2 === 0    // true  — even
11 % 2 === 0    // false — odd
11 % 2 === 1    // true  — odd

// Wrap around — keep a value within a range
const totalSlides  = 5;
let   currentSlide = 4;

currentSlide = (currentSlide + 1) % totalSlides;   // 0 — wraps back to start
// Goes: 0 → 1 → 2 → 3 → 4 → 0 → 1 → ...

// Check if divisible
15 % 5 === 0    // true — 15 is divisible by 5
```

---

### 6. Increment and Decrement — `++` and `--`

```js
let counter = 5;

counter++;   // post-increment — returns 5 THEN increments to 6
++counter;   // pre-increment  — increments to 7 THEN returns 7

counter--;   // post-decrement — returns 7 THEN decrements to 6
--counter;   // pre-decrement  — decrements to 5 THEN returns 5
```

The difference between `++x` and `x++` only matters when you use the result in an expression:
```js
let a = 5;
let b = a++;   // b = 5 (old value), then a becomes 6
let c = ++a;   // a becomes 7 first, then c = 7

// In isolation (own line), both do the same thing — just increment
counter++;     // perfectly fine
++counter;     // same outcome when not used as part of an expression
```

---

### 7. `!` (NOT) and `!!` (double NOT)

```js
!true     // false
!false    // true
!0        // true  — 0 is falsy, NOT flips to true
!""       // true  — empty string is falsy
!"hello"  // false — non-empty string is truthy, NOT flips to false
!null     // true

// !! (double NOT) — converts any value to its boolean equivalent
!!0         // false
!!"hello"   // true
!!null      // false
!![]        // true  — empty array is truthy
!!undefined // false

// Used to convert a value to boolean explicitly
const hasItems = !!cartItems.length;   // true if length > 0, false if 0
```

---

## Example 1 — basic

```js
// A simple quiz scorer
const correctAnswers = 7;
const totalQuestions = 10;
const passMark       = 5;

const percentage  = (correctAnswers / totalQuestions) * 100;   // 70
const hasPassed   = correctAnswers >= passMark;                // true
const grade       = percentage >= 80 ? "Distinction"
                  : percentage >= 60 ? "Pass"
                  : "Fail";

console.log(`Score: ${correctAnswers}/${totalQuestions}`);     // "Score: 7/10"
console.log(`Percentage: ${percentage}%`);                     // "Percentage: 70%"
console.log(`Result: ${grade}`);                               // "Result: Pass"
console.log(`Passed: ${hasPassed}`);                           // "Passed: true"
```

---

## Example 2 — real world

```js
// A user dashboard — handle missing data gracefully
const loggedInUser = {
  username:  "parsh_dev",
  credits:   0,
  bio:       "",
  plan:      null
};

// ?? — correct fallback (doesn't trigger on 0 or "")
const displayCredits = loggedInUser.credits ?? "Loading...";   // 0 — correct
const displayBio     = loggedInUser.bio     ?? "No bio yet";   // "" — oops, still empty

// For bio, empty string should show the fallback — so || is correct here
const displayBio2    = loggedInUser.bio     || "No bio yet";   // "No bio yet" — correct

const planLabel      = loggedInUser.plan    ?? "Free";         // "Free" — null triggers ??

// && to guard access
const isVerified     = loggedInUser.username && loggedInUser.username.length > 3;  // true

// Ternary in a message
const welcomeMsg = `Welcome back, ${loggedInUser.username}! You are on the ${planLabel} plan.`;
// "Welcome back, parsh_dev! You are on the Free plan."

console.log(displayCredits);   // 0
console.log(displayBio2);      // "No bio yet"
console.log(planLabel);        // "Free"
console.log(welcomeMsg);
```

---

## Common mistakes

### 1. Using `||` when `??` is needed — accidentally swallowing valid falsy values
```js
// WRONG — treats 0 as "no value"
const retryCount = userSetting || 3;   // if userSetting is 0, you get 3 — wrong!

// RIGHT — 0 is a valid setting, only replace null/undefined
const retryCount = userSetting ?? 3;
```

### 2. Misreading operator precedence
```js
// WRONG — looks like it checks (a === 1 OR a === 2)
if (a === 1 || 2) { ... }   // ALWAYS true — this is (a === 1) || (2), and 2 is always truthy

// RIGHT
if (a === 1 || a === 2) { ... }
```

### 3. Modifying a variable while using it in a ternary or expression
```js
// WRONG — confusing and error-prone
let x = 5;
let y = x++ + ++x;    // post then pre-increment — hard to reason about

// RIGHT — keep increments on their own lines
let x = 5;
x++;
x++;
let y = x + x;
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `&&` as a conditional renderer (common in React)
```js
const isAdmin = true;
const panel   = isAdmin && "<AdminPanel />";
// If isAdmin is true, returns "<AdminPanel />"
// If isAdmin is false, returns false (not the string)
// Used everywhere in JSX: {isAdmin && <AdminPanel />}
```

### Trick 2 — Chaining `??`
```js
const value = userInput ?? savedValue ?? defaultValue;
// Returns the first non-null/undefined value in the chain
```

### Trick 3 — `??` cannot be mixed with `&&`/`||` without parentheses
```js
// SyntaxError — JS doesn't know precedence between ?? and ||
const val = a || b ?? c;      // SyntaxError

// RIGHT — use parentheses to be explicit
const val = (a || b) ?? c;
const val = a || (b ?? c);
```

### Trick 4 — `**` (exponentiation) is right-associative
```js
2 ** 3 ** 2    // 2 ** (3 ** 2) = 2 ** 9 = 512  (NOT (2**3)**2 = 64)
// Exponentiation chains from right to left — opposite of most operators
```

### Trick 5 — Comparing strings with `>` and `<` uses Unicode, not length
```js
"banana" > "apple"    // true — 'b' (98) > 'a' (97)
"10" > "9"            // false — '1' (49) < '9' (57) — string comparison!
10 > 9                // true — numeric comparison, works as expected

// This is why sorting an array of number-strings without a sort function gives wrong results
["10", "9", "2"].sort()   // ["10", "2", "9"] — lexicographic, not numeric
```

---

## Practice exercises

### Exercise 1 — easy
Given these variables:
```js
const temperature = 0;
const windSpeed   = null;
const humidity    = 85;
const cityName    = "";
```

Write expressions using the correct operator (`||` or `??`) for each:
1. Display temperature — show "Unknown" only if not provided (0 is valid)
2. Display windSpeed — show "Calm" if no data
3. Display cityName — show "Unnamed location" if empty or not set
4. Display humidity — show "N/A" only if not provided (0 would be valid humidity)

Log all four results.
```js
// Write your code here
```

---

### Exercise 2 — medium
You're building a ticket pricing system. Using ternary operators and arithmetic:

```js
const basePrice    = 500;
const isMember     = true;
const ticketCount  = 3;
const promoCode    = null;
```

Write code that calculates and logs:
1. Member discount: members get 15% off, non-members get 0% off — use ternary
2. Promo discount: if a promoCode exists, apply an extra 10% off the already-discounted price — use `??` or `&&`
3. Final total for all tickets
4. A summary line: `"3 tickets × $425 = $1275.00 (Member discount applied)"`

Use `.toFixed(2)` for prices. All logic must use operators — no if/else allowed.
```js
// Write your code here
```

---

### Exercise 3 — hard
You're reviewing a colleague's authentication logic. Find **all operator-related bugs** (there are 5), explain each one, and rewrite the entire function correctly:

```js
function checkAccess(userId, role, sessionActive, failedAttempts) {
  // 1. User is valid if userId exists and is not zero
  const isValidUser = userId || role;

  // 2. Admin has full access, others need active session
  const hasAccess = role === "admin" || "moderator" ? true : sessionActive;

  // 3. Account is locked after 3 failed attempts
  const isLocked = failedAttempts > 3;

  // 4. Display remaining attempts — failedAttempts could be 0
  const remainingAttempts = (3 - failedAttempts) || "No attempts left";

  // 5. Grant access only if valid user, has access, and not locked
  const canLogin = isValidUser && hasAccess || !isLocked;

  console.log(`Can login: ${canLogin}`);
}

checkAccess(0, "moderator", true, 0);
// Expected: Can login: true
// Actual with bugs: likely wrong
```
```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Arithmetic:**
```
+   addition / string concat
-   subtraction
*   multiplication
/   division
%   modulo (remainder)
**  exponentiation
++  increment (pre/post)
--  decrement (pre/post)
```

**Comparison — always use `===` and `!==`:**
```
===  strict equal (no coercion)
!==  strict not-equal
>  <  >=  <=  comparison (uses coercion on mixed types)
```

**Logical:**
```
&&   AND — returns first falsy, or last value
||   OR  — returns first truthy, or last value
!    NOT — flips to boolean
!!   double NOT — converts to boolean
```

**Key rules:**
```
??  → fallback for null/undefined ONLY
||  → fallback for any falsy value (false, 0, "", null, undefined, NaN)
?:  → ternary — inline if/else, returns a value

&&  short-circuits on first falsy
||  short-circuits on first truthy

?? cannot be chained with || or && without parentheses
** is right-associative — chains right to left
```

**Operator precedence (remember with PAMD-CE-AND-OR-NC-T-A):**
```
( )          → Parentheses
** * / %     → Exponent, Multiply, Divide, Modulo
+ -          → Add, Subtract
> < >= <=    → Comparison
=== !== == != → Equality
&&           → AND
||           → OR
??           → Nullish
? :          → Ternary
= += -= ...  → Assignment
```

---

## Connected topics
- **03 — Type Coercion** — `||`, `&&`, and comparison operators trigger coercion; understanding coercion is prerequisite
- **08 — Truthy and Falsy** — `||`, `&&`, and `??` rely on truthy/falsy evaluation to decide what to return
- **07 — Conditionals** — operators are the building blocks of every condition in if/else and switch
