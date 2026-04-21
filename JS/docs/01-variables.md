# 01 — Variables

## What is this?
A variable is a named container that holds a value in your program. Think of it like a labelled box in a storage room — you put something inside, give the box a name, and later you can find it, open it, or swap out what's inside. In JavaScript, you create variables using three keywords: `var`, `let`, and `const`.

## Why does it matter?
Almost every program stores and reuses data — a user's name, a price, a score. Variables are how you hold onto that data while your code runs. Choosing the right keyword (`var`, `let`, or `const`) affects whether your data can change and where in your code it's accessible.

## Syntax
```js
var   oldSchoolName = "Alice";   // function-scoped, hoisted, avoid in modern JS
let   currentScore  = 42;        // block-scoped, can be reassigned
const MAX_LIVES     = 3;         // block-scoped, cannot be reassigned
```

## How it works — line by line
- `var oldSchoolName = "Alice"` — Declares a variable using the old keyword. It gets hoisted to the top of its function (or global scope), which can cause confusing bugs. Avoid it in new code.
- `let currentScore = 42` — Declares a variable that lives only inside the nearest `{ }` block. You can change its value later with `currentScore = 99`.
- `const MAX_LIVES = 3` — Declares a variable whose binding cannot be reassigned. Once set, you can't do `MAX_LIVES = 5`. Use this by default unless you know the value will change.

## Example 1 — basic
```js
// Storing a user's name and greeting them
const userName = "Parsh";          // name won't change, so use const
let loginCount = 0;                // count will change, so use let

loginCount = loginCount + 1;       // user logs in — increment the count

console.log(userName);             // "Parsh"
console.log(loginCount);           // 1
```

## Example 2 — real world
```js
// A simple product in an online store
const productName  = "Mechanical Keyboard";   // product name never changes
const productPrice = 89.99;                   // price is fixed at declaration

let stockCount     = 120;                     // stock changes as people buy
let isOnSale       = false;                   // sale status can be toggled

// A customer buys one unit
stockCount = stockCount - 1;                  // update stock
isOnSale   = true;                            // flash sale just started

console.log(`${productName} — $${productPrice}`);   // "Mechanical Keyboard — $89.99"
console.log(`Stock left: ${stockCount}`);            // "Stock left: 119"
console.log(`On sale: ${isOnSale}`);                 // "On sale: true"
```

## Common mistakes

### 1. Using `var` and getting surprised by hoisting
```js
// WRONG — var is hoisted, gives undefined instead of an error
console.log(discount);   // undefined (not an error — confusing!)
var discount = 10;

// RIGHT — let throws a clear ReferenceError if used before declaration
console.log(discount);   // ReferenceError: Cannot access 'discount' before initialization
let discount = 10;
```

### 2. Trying to reassign a `const`
```js
// WRONG
const taxRate = 0.18;
taxRate = 0.20;   // TypeError: Assignment to constant variable

// RIGHT — use let if the value needs to change
let taxRate = 0.18;
taxRate = 0.20;   // works fine
```

### 3. Thinking `const` makes objects fully immutable
```js
// WRONG assumption — the object's contents CAN change
const userProfile = { name: "Parsh", age: 25 };
userProfile.age = 26;            // this works — you're changing a property, not rebinding the variable
userProfile = { name: "Bob" };   // TypeError — you can't rebind the variable itself

// RIGHT mental model — const locks the variable reference, not the object contents
```

## Practice exercises

### Exercise 1 — easy
Declare three variables to represent a movie ticket booking:
- The cinema name (should never change)
- The seat number (can change if the customer swaps seats)
- Whether the ticket has been scanned at the door (starts as false)

Print all three values to the console.
```js
// ✅ Ideal solution
const cinemaName = "PVR Cinemas";   // name of the cinema never changes — const
let seatNumber   = "F7";            // seat can be swapped — let
let isScanned    = false;           // scan status changes at entry — let

console.log(cinemaName);   // "PVR Cinemas"
console.log(seatNumber);   // "F7"
console.log(isScanned);    // false
```


---

### Exercise 2 — medium
You're tracking a player in a game. The player has a username (fixed), a level that starts at 1, and a score that starts at 0.

Write code that:
1. Declares all three variables with the right keyword
2. Simulates the player earning 500 points
3. Simulates the player levelling up (level increases by 1)
4. Logs a message like: `"dragonSlayer99 — Level 2 — Score: 500"`

Use a template literal for the final log.
```js
// ✅ Ideal solution
const playerName = "dragonSlayer99";   // username never changes — const
let level        = 1;                  // level increases — let
let score        = 0;                  // score increases — let

score = score + 500;   // player earns 500 points
level = level + 1;     // player levels up

console.log(`${playerName} — Level ${level} — Score: ${score}`);
// "dragonSlayer99 — Level 2 — Score: 500"
```

---

### Exercise 3 — hard
You're building the first lines of a checkout system. Declare variables for:
- The customer's full name
- The original price of an item (e.g. 120)
- A discount percentage (e.g. 15 — meaning 15%)
- A boolean for whether the customer is a premium member

Then calculate:
- The discount amount (original price × discount / 100)
- The final price after the discount is applied

Log a receipt-style message to the console, something like:
```
Customer: Parsh Shah
Original price: $120
Discount (15%): -$18
Final price: $102
Premium member: true
```

Pick the right keyword (`let` vs `const`) for every variable based on whether it would logically change in a real checkout flow.
```js
// ✅ Ideal solution
const customerFullName = "Parsh Shah";   // name doesn't change mid-checkout — const
const originalPrice    = 120;            // source-of-truth price never changes — const
const discountPercent  = 15;             // discount rate is fixed for this order — const
const isPremium        = true;           // membership status is fixed — const

// Calculated values — also won't change once computed, so const is correct
const discountAmount = originalPrice * (discountPercent / 100);   // 18
const finalPrice     = originalPrice - discountAmount;            // 102

console.log(`Customer: ${customerFullName}`);
console.log(`Original price: $${originalPrice}`);
console.log(`Discount (${discountPercent}%): -$${discountAmount}`);
console.log(`Final price: $${finalPrice}`);
console.log(`Premium member: ${isPremium}`);

// Key lesson: discountPercent and discountAmount are SEPARATE variables.
// Never overwrite a variable with a different kind of value — it kills readability.
```

---

## Quick reference (cheat sheet)

| Keyword | Scope | Hoisted? | Reassignable? | When to use |
|---------|-------|----------|---------------|-------------|
| `var`   | Function | Yes (as `undefined`) | Yes | Avoid — legacy code only |
| `let`   | Block `{}` | No (TDZ error) | Yes | When the value will change |
| `const` | Block `{}` | No (TDZ error) | No (rebind) | Default choice — use unless value changes |

**Key rules to remember:**
- Prefer `const` → use `let` only when you know the value will change → never use `var`
- `const` on an object/array doesn't freeze the contents, only the variable binding
- Block scope means the variable dies when its `{ }` closes
- TDZ = Temporal Dead Zone — the period before a `let`/`const` is declared where accessing it throws an error

## Connected topics
- **02 — Data Types** — variables hold values; data types define what kind of value that is
- **18 — Scope** — deep dive into how block, function, and global scope work and why it matters
- **03 — Type Coercion** — understanding what happens when JS automatically converts the type of a variable's value
