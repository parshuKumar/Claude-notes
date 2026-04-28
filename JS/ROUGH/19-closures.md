# 19 — Closures

## What is this?
A closure is what happens when a function **remembers the variables from the scope where it was created**, even after that outer scope has finished running. Think of it like a backpack — when a function is born inside another function, it packs up the surrounding variables into that backpack and carries them wherever it goes, even long after the outer function is done.

## Why does it matter?
Closures are one of the most powerful features in JavaScript. They let you create private variables, build functions that remember state between calls, and are the foundation behind things like callbacks, event handlers, and module patterns.

## Syntax

```js
function outer() {
  let count = 0;           // variable in outer scope

  function inner() {       // inner function — this becomes a closure
    count++;               // inner "closes over" count — remembers it
    console.log(count);
  }

  return inner;            // return the inner function (not its result)
}

const increment = outer(); // outer() runs and finishes — but count is NOT gone
increment();               // 1  — inner still has access to count
increment();               // 2  — count persists across calls
increment();               // 3
```

## How it works — line by line

1. `function outer()` — defines the outer function with its own scope.
2. `let count = 0` — a local variable inside `outer`. Normally it would disappear when `outer()` finishes.
3. `function inner()` — defined inside `outer`, so it has access to `outer`'s scope.
4. `count++` — `inner` reads and modifies `count` from its parent scope. This is the closure.
5. `return inner` — we return the function itself (no `()`) — not the result of calling it.
6. `const increment = outer()` — `outer()` runs, creates `count`, creates `inner`, and returns `inner`. Even though `outer` is done, `count` stays alive because `inner` still references it.
7. `increment()` — calls `inner`. It finds `count` in its closed-over scope and increments it.
8. Each subsequent call to `increment()` works on the **same** `count` — it persists.

## Example 1 — basic

```js
function makeGreeting(greeting) {
  // greeting is captured in the closure
  function greetUser(userName) {
    console.log(greeting + ", " + userName + "!"); // uses greeting from outer scope
  }
  return greetUser; // return the inner function
}

const sayHello = makeGreeting("Hello");   // greeting = "Hello" is locked in
const sayHey   = makeGreeting("Hey");     // greeting = "Hey" is a separate closure

sayHello("Alice"); // "Hello, Alice!"
sayHello("Bob");   // "Hello, Bob!"
sayHey("Alice");   // "Hey, Alice!"
// Each closure has its OWN copy of greeting — they don't share
```

## Example 2 — real world

```js
// A shopping cart item counter — each cart is independent

function createCart(storeName) {
  let itemCount = 0; // private — nothing outside can touch this directly

  return {
    addItem: function () {
      itemCount++;
      console.log(storeName + " cart: " + itemCount + " item(s)");
    },
    removeItem: function () {
      if (itemCount > 0) itemCount--;
      console.log(storeName + " cart: " + itemCount + " item(s)");
    },
    getCount: function () {
      return itemCount; // only way to read itemCount from outside
    }
  };
}

const nikeCart  = createCart("Nike");   // its own private itemCount = 0
const adidasCart = createCart("Adidas"); // completely separate itemCount = 0

nikeCart.addItem();    // "Nike cart: 1 item(s)"
nikeCart.addItem();    // "Nike cart: 2 item(s)"
adidasCart.addItem();  // "Adidas cart: 1 item(s)"

console.log(nikeCart.getCount());   // 2
console.log(adidasCart.getCount()); // 1
// itemCount is never directly accessible — closures act as private state
```

## Common mistakes

- **Calling the returned function immediately instead of storing it**

  ```js
  // ❌ Wrong — calling outer() AND inner() immediately, losing the closure
  function makeCounter() {
    let count = 0;
    return function () { count++; return count; };
  }
  const result = makeCounter()(); // runs once, result = 1 — no persistent state

  // ✅ Right — store the returned function, then call it multiple times
  const counter = makeCounter(); // store the closure
  counter(); // 1
  counter(); // 2
  counter(); // 3
  ```

- **Thinking each closure call shares the same variable**

  ```js
  // ❌ Wrong assumption — two closures DO NOT share variables
  const counterA = makeCounter();
  const counterB = makeCounter();

  counterA(); // 1
  counterA(); // 2
  counterB(); // ??? many think this continues from 2 — it does NOT

  // ✅ Right — each call to makeCounter() creates a brand-new, separate count
  counterB(); // 1 — completely independent from counterA
  ```

- **Classic loop + closure trap with `var`**

  ```js
  // ❌ Wrong — var is function-scoped, all 3 callbacks share the SAME i
  for (var i = 1; i <= 3; i++) {
    setTimeout(function () {
      console.log(i); // prints 4, 4, 4 — not 1, 2, 3
    }, 1000);
  }

  // ✅ Right — let is block-scoped, each iteration has its OWN i
  for (let i = 1; i <= 3; i++) {
    setTimeout(function () {
      console.log(i); // prints 1, 2, 3 — each closure captures its own i
    }, 1000);
  }
  ```

## Practice exercises

### Exercise 1 — easy
Write a function called `makeMultiplier` that takes a number `factor` as a parameter and returns a new function. The returned function should take a number `value` and return `value * factor`. Create two multipliers: one that doubles (factor 2) and one that triples (factor 3). Test both with a few numbers.

```js
// Write your code here
```

### Exercise 2 — medium
Build a `createTimer` function that uses a closure to track how many seconds have passed. It should return an object with two methods:
- `tick()` — increments the internal seconds count and logs `"Time: X second(s)"`
- `reset()` — sets the seconds count back to 0 and logs `"Timer reset"`

Create one timer, call `tick()` three times, then `reset()`, then `tick()` once more. The seconds should restart from 1 after the reset.

```js
// Write your code here
```

### Exercise 3 — hard
You're building a simple **bank account system** using closures. Write a function `createAccount` that takes `ownerName` and `initialBalance` as parameters. The function should return an object with three methods:
- `deposit(amount)` — adds to the balance, logs `"[ownerName] deposited $X. Balance: $Y"`
- `withdraw(amount)` — deducts from the balance only if funds are sufficient, otherwise logs `"Insufficient funds"`. Logs `"[ownerName] withdrew $X. Balance: $Y"` on success.
- `getBalance()` — returns the current balance

Requirements:
- The `balance` must be private — it should only be accessible through the methods above
- Create two separate accounts and perform operations on each — prove they don't share balance
- Try accessing `balance` directly on the returned object — it should be `undefined`

```js
// Write your code here
```

## Quick reference (cheat sheet)

| Concept | What it means |
|---|---|
| Closure | A function that remembers variables from its outer scope after that scope has closed |
| Created when | An inner function is defined inside an outer function |
| Why variables survive | JS keeps variables alive as long as any function still references them |
| Each closure | Gets its own copy of the closed-over variables — they don't share |
| Common use cases | Private state, factory functions, callbacks, event handlers, memoization |

**Key rules:**
- Return the function itself (`return fn`) — NOT the result (`return fn()`)
- Closures capture variables **by reference**, not by value — the variable can still change
- Every call to the outer function creates a **new, independent** closure
- `let` in loops gives each iteration its own variable — `var` does NOT

## Connected topics
- **18 — Scope**: closures are impossible to understand without scope — they ARE scope remembered over time
- **20 — Higher-order functions**: HOFs return or accept functions — closures power most of them
- **21 — Callback functions**: callbacks often rely on closure to access outer variables
