# 10 — `while` and `do...while`

## What is this?
A `while` loop keeps running a block of code **as long as a condition is true** — and it doesn't care about a counter. Think of it like a security door that stays open while someone is holding a button: it doesn't count how many times it's been opened, it just checks "is the button still held?" before every cycle. `do...while` is the same idea, except the door opens **once first**, then starts checking. Use these loops when you don't know upfront how many iterations you need — only when to stop.

## Why does it matter?
The `for` loop is ideal when you know the exact count. `while` is ideal when you're waiting for a condition to become true — reading user input, polling a server, consuming a queue, searching for a value, running a game loop. Choosing the wrong loop type makes code harder to reason about. Choosing the right one makes your intent clear immediately.

---

## Syntax

```js
// while — checks condition BEFORE each iteration
while (condition) {
  // body
}

// do...while — runs body ONCE, THEN checks condition
do {
  // body
} while (condition);
```

---

## How it works — step by step

### `while`:
```
1. Evaluate condition
2. If true  → run body → go back to step 1
3. If false → exit loop
```

If the condition is false on the very first check, the body **never runs**.

### `do...while`:
```
1. Run body (always, at least once)
2. Evaluate condition
3. If true  → go back to step 1
4. If false → exit loop
```

The body always runs **at least once**, regardless of the condition.

```js
// while — body may never run
let count = 10;
while (count < 5) {
  console.log(count);   // never runs — 10 < 5 is false from the start
}

// do...while — body runs once even though condition is false
let count2 = 10;
do {
  console.log(count2);   // runs once — prints 10
} while (count2 < 5);    // then checks: 10 < 5 = false → exits
```

---

## Deep dive — everything you need to know

### 1. The three parts of a `while` loop — and where they live

A `for` loop packs initialisation, condition, and update into one line. A `while` loop spreads them out:

```js
// for loop — all three in one place
for (let i = 0; i < 5; i++) {
  console.log(i);
}

// Equivalent while loop — initialisation BEFORE, update INSIDE body
let i = 0;              // initialisation — before the loop
while (i < 5) {         // condition
  console.log(i);
  i++;                  // update — inside the body
}
```

They produce identical output. The `while` is more flexible — your update can happen anywhere in the body, conditionally, or not at all (intentional infinite loop).

---

### 2. When to use `while` over `for`

Use `while` when:
- You don't know how many iterations you need
- The loop depends on an external state that changes unpredictably
- You're consuming a data source until it's exhausted

```js
// Consuming a queue until empty — don't know how many items
const messageQueue = ["msg1", "msg2", "msg3"];
while (messageQueue.length > 0) {
  const message = messageQueue.shift();   // removes first item
  console.log("Processing:", message);
}

// Searching for a value — stop when found
const haystack = [14, 7, 3, 99, 42, 5];
let target  = 42;
let index   = 0;
let found   = false;

while (index < haystack.length && !found) {
  if (haystack[index] === target) {
    found = true;
  } else {
    index++;
  }
}

console.log(found ? `Found ${target} at index ${index}` : "Not found");
// "Found 42 at index 4"
```

---

### 3. When to use `do...while`

Use `do...while` when the body **must run at least once** before the condition makes sense:

```js
// Classic use case — input validation (simulated here since we're not in a browser)
// "Ask user for a number, keep asking until they give a valid one"
// In a real app with prompts:
let userInput;
do {
  userInput = prompt("Enter a number between 1 and 10:");
  userInput = Number(userInput);
} while (userInput < 1 || userInput > 10 || Number.isNaN(userInput));
console.log("You entered:", userInput);

// The first prompt MUST run before we can check if the input is valid.
// do...while is the correct loop here — not while.
```

```js
// Menu-driven simulation
let choice;
do {
  // simulate showing a menu and getting a choice
  choice = "3";   // pretend user typed "3" (exit)
  console.log(`You chose: ${choice}`);
} while (choice !== "3");
// Ran once, then exited because choice === "3"
```

---

### 4. Infinite loops — intentional and accidental

**Accidental infinite loop — the most common `while` bug:**
```js
// WRONG — i is never updated → condition never becomes false → hangs forever
let i = 0;
while (i < 5) {
  console.log(i);
  // forgot i++
}
```

**Intentional infinite loop — used in game loops, server processes, CLI tools:**
```js
// Controlled with break
let attempts = 0;

while (true) {
  attempts++;
  console.log(`Attempt ${attempts}`);

  if (attempts >= 3) {
    console.log("Max attempts reached.");
    break;   // the only exit
  }
}
```

The `while (true)` pattern is legitimate and common. The `break` statement is the exit mechanism. Without `break`, you need a way to make the condition become false inside the body.

---

### 5. Multiple exit conditions

```js
// Loop with two ways to exit — found OR exhausted
const inventory = [
  { sku: "A1", name: "Keyboard",  stock: 0  },
  { sku: "B2", name: "Mouse",     stock: 14 },
  { sku: "C3", name: "Monitor",   stock: 0  },
  { sku: "D4", name: "Headphones",stock: 5  },
];

let i = 0;
let firstInStock = null;

while (i < inventory.length && firstInStock === null) {
  if (inventory[i].stock > 0) {
    firstInStock = inventory[i];
  }
  i++;
}

if (firstInStock) {
  console.log(`First in-stock item: ${firstInStock.name}`);   // "Mouse"
} else {
  console.log("Everything is out of stock.");
}
```

---

### 6. Loop with a flag variable

A clean pattern for complex exit conditions:

```js
const numbers  = [3, 7, 2, 11, 5, 13, 8];
let   isPrime  = true;
let   i        = 2;

// Check if numbers[0] (3) is prime — divide by every number from 2 to sqrt
const num = numbers[0];
while (i <= Math.sqrt(num) && isPrime) {
  if (num % i === 0) {
    isPrime = false;   // found a divisor — not prime
  }
  i++;
}

console.log(`${num} is ${isPrime ? "prime" : "not prime"}`);   // "3 is prime"
```

---

### 7. Nested `while` loops

```js
// Simulating a two-pointer approach on a sorted array
const sortedArr = [1, 3, 5, 7, 9, 11];
const target    = 12;
let left  = 0;
let right = sortedArr.length - 1;

while (left < right) {
  const sum = sortedArr[left] + sortedArr[right];
  if (sum === target) {
    console.log(`Pair found: ${sortedArr[left]} + ${sortedArr[right]}`);
    break;
  } else if (sum < target) {
    left++;
  } else {
    right--;
  }
}
```

---

### 8. `do...while` vs `while` — the decision guide

| Scenario | Loop |
|----------|------|
| Show a prompt, then validate | `do...while` |
| Process items in a queue | `while` |
| Count down to zero | `while` or `for` |
| Game: run game loop, show result at end | `do...while` |
| Search for a value | `while` |
| Must run at least once regardless of condition | `do...while` |
| May never need to run | `while` |

---

## Example 1 — basic

```js
// Countdown timer
let secondsLeft = 10;

while (secondsLeft > 0) {
  console.log(`${secondsLeft}...`);
  secondsLeft--;
}
console.log("Time's up!");

// 10... 9... 8... 7... 6... 5... 4... 3... 2... 1... Time's up!
```

```js
// do...while — show at least one loading message
let retries   = 0;
let connected = false;

do {
  retries++;
  console.log(`Connecting... attempt ${retries}`);
  connected = retries >= 3;   // simulates success on 3rd attempt
} while (!connected);

console.log("Connected!");
// Connecting... attempt 1
// Connecting... attempt 2
// Connecting... attempt 3
// Connected!
```

---

## Example 2 — real world

```js
// Paginated data fetcher — simulate fetching pages until there are no more
// (In real code this would use async/await — topic 56 — but the loop logic is the same)

const allPages = [
  { page: 1, data: ["item1", "item2"], hasMore: true  },
  { page: 2, data: ["item3", "item4"], hasMore: true  },
  { page: 3, data: ["item5"],          hasMore: false },
];

let   currentPage  = 0;
const allItems     = [];

do {
  const response = allPages[currentPage];   // simulates a fetch
  allItems.push(...response.data);
  console.log(`Fetched page ${response.page}: ${response.data.join(", ")}`);
  currentPage++;
} while (allPages[currentPage - 1].hasMore);

console.log("All items:", allItems);
// Fetched page 1: item1, item2
// Fetched page 2: item3, item4
// Fetched page 3: item5
// All items: ["item1", "item2", "item3", "item4", "item5"]
```

---

## Common mistakes

### 1. Forgetting to update the loop variable — accidental infinite loop
```js
// WRONG — i never changes, condition never becomes false
let i = 0;
while (i < 10) {
  console.log(i);   // logs 0 forever
  // i++ missing!
}

// RIGHT
let i = 0;
while (i < 10) {
  console.log(i);
  i++;
}
```

### 2. Using `while` when `do...while` is needed
```js
// WRONG — if retries starts at 0 and max is 0, this never runs
let retries = 0;
while (retries < maxRetries) {
  sendRequest();
  retries++;
}
// If the first attempt is mandatory regardless of maxRetries, use do...while

// RIGHT — first attempt always happens
do {
  sendRequest();
  retries++;
} while (retries < maxRetries);
```

### 3. Condition uses `=` instead of `===` or a comparison
```js
// WRONG — assignment instead of comparison — always truthy, infinite loop
let status = "pending";
while (status = "active") {   // assigns "active" to status — truthy forever
  console.log(status);
}

// RIGHT
while (status === "active") {
  console.log(status);
}
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `while` loop as a retry mechanism
```js
// Common pattern in network code
const MAX_RETRIES = 3;
let   attempts    = 0;
let   success     = false;

while (attempts < MAX_RETRIES && !success) {
  attempts++;
  console.log(`Attempt ${attempts}...`);
  success = Math.random() > 0.5;   // 50% chance of success
}

console.log(success ? "Request succeeded" : "All retries exhausted");
```

### Trick 2 — Mutual dependency between condition and body update
```js
// The condition checks something that the body modifies
// Always trace through manually to make sure the exit is reachable

let balance = 1000;
let round   = 0;

while (balance > 0) {
  round++;
  balance -= Math.floor(Math.random() * 300) + 100;   // lose 100–400 each round
  console.log(`Round ${round}: balance $${Math.max(0, balance)}`);
}
console.log(`Game over after ${round} rounds`);
```

### Trick 3 — `while (true)` with `break` is sometimes cleaner than a flag
```js
// WITH a flag — requires reading two things to understand the exit
let keepGoing = true;
while (keepGoing) {
  const value = getNextValue();
  if (value === null) keepGoing = false;
  else process(value);
}

// WITH while(true) + break — the exit condition is at the decision point
while (true) {
  const value = getNextValue();
  if (value === null) break;
  process(value);
}
// The second is often clearer — you see "break" right next to the condition
```

### Trick 4 — `do...while` semicolon is mandatory
```js
// WRONG — missing semicolon after while(...) in do...while
do {
  console.log("hello");
} while (false)   // SyntaxError in strict mode, silent bug otherwise

// RIGHT — semicolon required to terminate the statement
do {
  console.log("hello");
} while (false);
```

### Trick 5 — Loop variable declared outside is accessible after the loop ends
```js
// for loop — i dies after the loop (with let)
for (let i = 0; i < 5; i++) { }
// console.log(i); → ReferenceError

// while loop — i was declared outside, still accessible after
let i = 0;
while (i < 5) { i++; }
console.log(i);   // 5 — accessible and holds the final value

// This is useful when you need the final value (e.g., the index where you stopped searching)
let j = 0;
const data = [10, 20, 30, 40, 50];
while (j < data.length && data[j] !== 30) { j++; }
console.log(j);   // 2 — the index where 30 was found
```

---

## Practice exercises

### Exercise 1 — easy
Write a `while` loop that:
1. Starts with a balance of `$1000`
2. Each iteration, deducts a fixed monthly expense of `$175`
3. Logs the month number and remaining balance each iteration
4. Stops when the balance drops below `$0`
5. After the loop, logs how many months the money lasted

```js
// Write your code here
```

---

### Exercise 2 — medium
You have this unsorted list of support tickets:

```js
const tickets = [
  { id: "T001", priority: "low",      resolved: true  },
  { id: "T002", priority: "high",     resolved: false },
  { id: "T003", priority: "medium",   resolved: true  },
  { id: "T004", priority: "critical", resolved: false },
  { id: "T005", priority: "high",     resolved: false },
  { id: "T006", priority: "low",      resolved: true  },
];
```

Using a `while` loop:
1. Find the **first** unresolved critical or high priority ticket
2. Log its ID and priority
3. Count how many tickets were checked before finding it
4. If no such ticket exists, log `"All urgent tickets are resolved"`

Then write a **separate** `do...while` loop that:
- Simulates escalating the found ticket — logs `"Escalating T00X... attempt N"` 
- Repeats until a simulated `Math.random() > 0.4` check passes (connection established)
- Logs `"Ticket T00X successfully escalated after N attempts"`

```js
// Write your code here
```

---

### Exercise 3 — hard
Build a number guessing game simulation using `do...while`.

```js
const secretNumber = 47;   // the number to guess
const maxGuesses   = 7;
```

Simulate a player guessing by generating numbers: use this sequence to simulate guesses (not random, so results are predictable):

```js
const guesses = [20, 60, 35, 55, 40, 50, 47];
```

The game loop must:
1. Pick the next guess from the array each iteration
2. Log `"Guess N: [number] → Too low"` / `"Too high"` / `"Correct!"`
3. Stop if the guess is correct OR max guesses are reached
4. After the loop, log the outcome:
   - If correct: `"Won in N guesses! Numbers tried: [list]"`
   - If out of guesses: `"Game over — the number was 47. You tried: [list]"`
5. Also log the average of all guesses made (using a `for` loop or accumulation — your choice)

Use `do...while` for the game loop. Use a separate `for` or `while` for the average calculation.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**`while` structure:**
```js
let i = 0;           // initialise before
while (condition) {
  // body
  i++;               // update inside body
}
```

**`do...while` structure:**
```js
do {
  // body — runs at least once
} while (condition);  // ← semicolon required
```

**Decision guide:**
```
Know the count upfront?          → for loop
Don't know count, check before?  → while
Must run at least once?          → do...while
```

**Key rules:**
- Always update the loop variable inside the body — forgetting causes infinite loops
- `while` body may never run if condition starts false
- `do...while` body always runs at least once
- `while (true) { ... break; }` is a valid and often clean pattern
- The loop variable declared outside a `while` is accessible after the loop ends
- `do { } while (cond)` requires a **semicolon** after the closing parenthesis
- Use `&&` in condition to combine multiple stop conditions

---

## Connected topics
- **09 — for loop** — the fixed-count counterpart; understanding both lets you choose the right tool
- **13 — break and continue** — `break` is what gives `while(true)` loops a safe exit
- **11 — for...of** — cleaner syntax when iterating arrays; `while` is better for non-array conditions
- **56 — async/await** — retry loops using `while` are a core async pattern in real apps
