# 43 — Currying

---

## What is this?

**Currying** is the technique of transforming a function that takes multiple arguments into a chain of functions that each take **one argument at a time**. Instead of calling `add(2, 3)`, you call `add(2)(3)`.

Named after mathematician Haskell Curry, it's a core concept in functional programming.

```js
// Normal function — all args at once
function add(a, b) {
  return a + b;
}
add(2, 3); // 5

// Curried version — one arg at a time
function curriedAdd(a) {
  return function(b) {
    return a + b;
  };
}
curriedAdd(2)(3); // 5

// Arrow function shorthand (most common style)
const curriedAdd = a => b => a + b;
curriedAdd(2)(3); // 5
```

Think of it like ordering a custom coffee — you first choose the size, then the type, then extras. Each choice locks in that parameter and returns a "partially configured" order.

---

## Why does it matter?

Currying enables two powerful techniques:

1. **Partial application** — fix some arguments now, supply the rest later. Create specialised functions from general ones without repetition.
2. **Function composition** — curried functions (all taking one argument) compose together cleanly in pipelines.

You'll encounter currying in functional libraries (Ramda, Lodash/fp), React (higher-order components, hooks), event handlers, and any code that builds specialised functions from general ones.

---

## Syntax

```js
// Manual currying
const fn = a => b => c => a + b + c;
fn(1)(2)(3); // 6

// Partial application — lock in first arg(s)
const addFive = fn(5);   // b => c => 5 + b + c
addFive(3)(2);           // 10

// Using a curry helper (shown later)
const curriedFn = curry(originalFn);
curriedFn(1, 2, 3);   // call all at once
curriedFn(1)(2)(3);   // or one at a time
curriedFn(1, 2)(3);   // or any combination
```

---

## How it works — line by line

```js
const multiply = a => b => a * b;
// multiply is a function that takes 'a'
// it returns a NEW function that takes 'b'
// the inner function closes over 'a' (closure!)
// and returns a * b when called

multiply(3);     // returns b => 3 * b  (a partial function with a=3 locked in)
multiply(3)(4);  // 12                  (both args supplied)

// Creating specialised functions via partial application
const double = multiply(2);   // b => 2 * b
const triple = multiply(3);   // b => 3 * b
const times10 = multiply(10); // b => 10 * b

double(5);   // 10
triple(4);   // 12
times10(7);  // 70

// double, triple, times10 are reusable specialised functions
// created from ONE general multiply function
```

---

## Example 1 — Partial application in practice

```js
// General function
const greet = greeting => name => `${greeting}, ${name}!`;

// Specialised functions via partial application
const sayHello   = greet('Hello');
const sayGoodbye = greet('Goodbye');
const sayNamaste = greet('Namaste');

sayHello('Alice');    // 'Hello, Alice!'
sayHello('Bob');      // 'Hello, Bob!'
sayGoodbye('Alice');  // 'Goodbye, Alice!'
sayNamaste('Parsh');  // 'Namaste, Parsh!'

// Without currying — you'd write this each time
greet('Hello', 'Alice');    // one-off
greet('Hello', 'Bob');      // one-off, repetitive

// With currying — create the specialisation ONCE, reuse it many times
const students = ['Alice', 'Bob', 'Carol', 'Dave'];
students.map(sayHello);
// ['Hello, Alice!', 'Hello, Bob!', 'Hello, Carol!', 'Hello, Dave!']
```

---

## Example 2 — Currying with array methods (the key power)

Curried functions slot perfectly into `.map()`, `.filter()`, `.reduce()` because those methods pass one argument per call:

```js
// Curried utility functions
const isGreaterThan = min => num => num > min;
const isLessThan    = max => num => num < max;
const multiplyBy    = factor => num => num * factor;
const add           = amount => num => num + amount;
const startsWith    = prefix => str => str.startsWith(prefix);

const scores   = [45, 78, 92, 61, 88, 33, 55, 97];
const products = ['apple', 'apricot', 'banana', 'avocado', 'cherry'];
const prices   = [100, 250, 75, 400, 180];

// Filter scores above 60
scores.filter(isGreaterThan(60));    // [78, 92, 61, 88, 55, 97]
// Wait — 61 and 55 are not > 60... let me recheck
scores.filter(isGreaterThan(60));    // [78, 92, 61, 88, 97]

// Apply 15% increase to all prices
prices.map(multiplyBy(1.15)).map(n => Math.round(n));
// [115, 288, 86, 460, 207]

// Add shipping cost of 50 to each price
prices.map(add(50));
// [150, 300, 125, 450, 230]

// Products starting with 'a'
products.filter(startsWith('a'));
// ['apple', 'apricot', 'avocado']

// Chain them: scores between 60 and 90
scores
  .filter(isGreaterThan(60))
  .filter(isLessThan(90));
// [78, 61, 88]
```

---

## Example 3 — Three-argument currying

```js
// Three curried arguments
const clamp = min => max => value => Math.min(Math.max(value, min), max);

// General use
clamp(0)(100)(150);  // 100 — clamped to max
clamp(0)(100)(-5);   // 0   — clamped to min
clamp(0)(100)(72);   // 72  — within range

// Partial application — create specialised clamp functions
const clampPercent = clamp(0)(100);   // clamps to 0–100
const clampRGB     = clamp(0)(255);   // clamps to 0–255
const clampScore   = clamp(0)(10);    // clamps to 0–10

clampPercent(150); // 100
clampPercent(-10); //   0
clampRGB(300);     // 255
clampScore(12);    //  10

// Apply to an array
const rawScores = [12, -3, 7, 15, 4, -1, 9];
rawScores.map(clampScore); // [10, 0, 7, 10, 4, 0, 9]
```

---

## Example 4 — Real-world: building a logger

```js
// Curried logger — level is locked in, message is passed later
const log = level => context => message =>
  `[${level.toUpperCase()}] [${context}] ${message}`;

// Specialise by level
const logInfo  = log('info');
const logWarn  = log('warn');
const logError = log('error');

// Specialise further by context (module/component)
const authInfo  = logInfo('Auth');
const authError = logError('Auth');
const dbInfo    = logInfo('Database');
const dbError   = logError('Database');

// Use — clean and consistent
authInfo('User login successful');        // '[INFO] [Auth] User login successful'
authError('Invalid token');              // '[ERROR] [Auth] Invalid token'
dbInfo('Connected to primary DB');       // '[INFO] [Database] Connected to primary DB'
dbError('Connection timeout');           // '[ERROR] [Database] Connection timeout'

// Or build logs for an array of events
const events = ['login', 'view_dashboard', 'logout'];
events.map(logInfo('UserSession'));
// ['[INFO] [UserSession] login', '[INFO] [UserSession] view_dashboard', '[INFO] [UserSession] logout']
```

---

## Example 5 — A general `curry` helper function

Writing curried functions manually with `a => b => c =>` only works when you know the arity (number of arguments) upfront. A `curry` helper makes ANY existing function curriable:

```js
function curry(fn) {
  // Returns a wrapper that collects args until we have enough
  return function curried(...args) {
    if (args.length >= fn.length) {
      // We have all the arguments — call the original function
      return fn.apply(this, args);
    }
    // Not enough args yet — return a function that collects more
    return function(...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
}

// Usage — curry any function
function add(a, b, c) { return a + b + c; }

const curriedAdd = curry(add);
curriedAdd(1, 2, 3); // 6  — all at once
curriedAdd(1)(2)(3); // 6  — one at a time
curriedAdd(1, 2)(3); // 6  — 2 then 1
curriedAdd(1)(2, 3); // 6  — 1 then 2

// Partial application
const addTo10 = curriedAdd(10);
addTo10(5, 3);   // 18
addTo10(5)(3);   // 18
```

---

## Example 6 — Currying for event handling and configuration

```js
// Without currying — repetitive, closure manually each time
document.getElementById('save-btn').addEventListener('click', function() {
  saveData('users', userData);
});
document.getElementById('delete-btn').addEventListener('click', function() {
  deleteData('users', userId);
});

// With currying — create handler factories
const handleSave   = entity => data     => () => saveData(entity, data);
const handleDelete = entity => id       => () => deleteData(entity, id);
const handleFetch  = entity => callback => () => fetchData(entity).then(callback);

// Clean, readable event listener setup
document.getElementById('save-btn')
  .addEventListener('click', handleSave('users')(userData));

document.getElementById('delete-btn')
  .addEventListener('click', handleDelete('users')(userId));
```

---

## Example 7 — Currying vs partial application (the distinction)

These terms are often used interchangeably but they are technically different:

```js
// CURRYING: transform a function into a chain of UNARY (one-arg) functions
const curried = a => b => c => a + b + c;
curried(1);      // b => c => 1 + b + c
curried(1)(2);   // c => 1 + 2 + c
curried(1)(2)(3); // 6

// PARTIAL APPLICATION: fix SOME arguments, leave others open — not necessarily one at a time
function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

function add(a, b, c) { return a + b + c; }

const addFrom5 = partial(add, 5);       // fixes a=5, b and c still open
addFrom5(3, 2);  // 10

const add5and3  = partial(add, 5, 3);   // fixes a=5, b=3, c still open
add5and3(2);     // 10

// Key difference:
// Currying: always unary — one arg at a time
// Partial: can fix multiple args at once
```

---

## Tricky things you'll encounter in the real world

### 1. Currying relies on knowing the function's arity (`fn.length`)

```js
// fn.length counts DECLARED parameters
function add(a, b) { return a + b; }
add.length; // 2

// But rest parameters and default params DON'T count toward length
function tricky(a, b = 10, ...rest) { }
tricky.length; // 1 — only 'a' counts

// This breaks curry helpers that rely on fn.length
const curriedTricky = curry(tricky);
curriedTricky(1);    // won't wait for b — thinks arity is 1
```

---

### 2. `this` context is lost in curried functions

```js
const obj = {
  factor: 3,
  // ❌ Arrow functions don't have their own 'this'
  multiply: x => y => x * y * this.factor, // 'this' is outer scope, not obj
};

// For methods on objects, avoid currying with arrow chains that need 'this'
// Use regular functions or explicit .bind()
```

---

### 3. Debugging curried functions can be harder

Stack traces show nested anonymous functions. Name your curried functions for better debugging:

```js
// ❌ Anonymous — hard to trace
const process = a => b => c => a + b + c;

// ✅ Named — clear in stack traces
const process = function process(a) {
  return function withB(b) {
    return function withC(c) {
      return a + b + c;
    };
  };
};
```

---

### 4. Too many levels of currying kills readability

```js
// ❌ Overkill — 5 levels deep is confusing
const fn = a => b => c => d => e => a + b + c + d + e;
fn(1)(2)(3)(4)(5); // hard to read

// ✅ Group related args
const fn = (a, b) => (c, d) => e => a + b + c + d + e;
fn(1, 2)(3, 4)(5); // clearer grouping
```

---

### 5. Curried functions and `undefined` — accidental partial application

```js
const multiply = a => b => a * b;

// If you forget to pass a arg, you get a function back, not a number
const result = multiply(5); // returns b => 5 * b (a function!)
console.log(result);        // [Function (anonymous)] — not 25!

// This is a feature (partial application) but a bug if you forgot an arg
```

---

## Common mistakes

### Mistake 1 — Calling the curried function without enough sets of parens

```js
const add = a => b => a + b;

// ❌ Only called once — result is a function, not a number
const result = add(2, 3);   // 2 is passed as 'a', 3 is IGNORED
console.log(result);        // b => 2 + b  (function!)

// ✅ Two separate calls
const result2 = add(2)(3);  // 5
```

---

### Mistake 2 — Confusing currying with a function that returns a function for other reasons

```js
// This is NOT currying — it returns a function but doesn't enable partial application of its own args
function makeAdder(x) {
  return function(y) { return x + y; }; // actually IS curried! just written with function keyword
}

// vs a factory that returns a function for configuration:
function createValidator(rules) {
  return function validate(data) {    // NOT currying — different purpose
    return rules.every(rule => rule(data));
  };
}
// Both return functions — only makeAdder is currying its add logic
```

---

### Mistake 3 — Using a curried function where a regular function would be cleaner

```js
// ❌ Over-engineered — no reuse, currying adds no value here
const getFullName = firstName => lastName => `${firstName} ${lastName}`;
getFullName('Alice')('Chen'); // 'Alice Chen'

// ✅ Just use a regular function when you always call with all args at once
function getFullName(firstName, lastName) {
  return `${firstName} ${lastName}`;
}
getFullName('Alice', 'Chen'); // 'Alice Chen'

// Currying is valuable when you need PARTIAL APPLICATION — fixing some args for reuse
const greetAs = title => name => `${title} ${name}`;
const greetDoctor = greetAs('Dr.');  // reused many times
greetDoctor('Alice');  // 'Dr. Alice'
greetDoctor('Smith');  // 'Dr. Smith'
```

---

## Frequently asked questions

**Q: What's the difference between currying and a function that just returns another function?**
All curried functions return functions (when partially applied), but not all functions that return functions are curried. Currying specifically means transforming an n-argument function into a chain of n unary (1-argument) functions.

**Q: Is currying the same as partial application?**
No — they're related but different. Currying always produces unary functions. Partial application fixes any number of arguments at once. A curried function can be partially applied, and `curry()` helpers support both styles.

**Q: Do I need a library like Ramda to use currying?**
No — you can write curried functions manually with `a => b => c => ...` or write a simple `curry()` helper. Libraries like Ramda provide fully curried versions of all utility functions and are used in production FP code.

**Q: Why is it called "currying"?**
After mathematician Haskell Curry, who studied combinatory logic. The concept actually predates him (Schönfinkel described it first), but Curry popularised it, so the name stuck.

**Q: Does currying affect performance?**
Marginal overhead from extra function calls and closures. In 99% of cases, irrelevant. For extremely hot code paths with millions of calls per second, profile before optimising.

**Q: Can I curry async functions?**
Yes — the currying mechanics are the same. The inner function just returns a Promise:

```js
const fetchWithAuth = token => async url => {
  const res = await fetch(url, { headers: { Authorization: `Bearer ${token}` } });
  return res.json();
};

const authedFetch = fetchWithAuth(myToken);
const user = await authedFetch('/api/users/1');
const post = await authedFetch('/api/posts/5');
```

---

## Practice exercises

### Exercise 1 — Easy: Build curried utility functions

Write these curried functions and use them to process the arrays below:

```js
const temperatures = [0, 15, 22, 30, 37, 100];
const words        = ['hello', 'world', 'javascript', 'curry', 'rocks'];
const amounts      = [1000, 2500, 500, 8000, 3200];
```

1. `celsiusToFahrenheit` — curried: takes a `scale` factor and `offset`, returns a function that converts. Use to convert all temps. *(F = C × 1.8 + 32)*
2. `longerThan` — curried: takes `minLength`, returns function that checks if string length > minLength. Use to filter `words`.
3. `applyTax` — curried: takes `taxRate`, returns function that adds tax to an amount. Use to map `amounts` with 18% tax.

```js
// Write your code here
```

---

### Exercise 2 — Medium: Config-driven pipeline

Build a set of curried data processing functions, then use them to build a configurable pipeline:

```js
const employees = [
  { name: 'Alice',  dept: 'Engineering', salary: 90000, yearsExp: 5 },
  { name: 'Bob',    dept: 'Marketing',   salary: 60000, yearsExp: 2 },
  { name: 'Carol',  dept: 'Engineering', salary: 95000, yearsExp: 8 },
  { name: 'Dave',   dept: 'HR',          salary: 55000, yearsExp: 1 },
  { name: 'Eve',    dept: 'Engineering', salary: 85000, yearsExp: 4 },
  { name: 'Frank',  dept: 'Marketing',   salary: 70000, yearsExp: 6 },
];
```

Write these curried functions:
- `filterByDept(dept)` — returns employees in that department
- `salaryAbove(min)` — returns employees with salary > min
- `raiseSalary(percent)` — returns a new array with all salaries increased by percent
- `sortBy(field)` — sorts by a given field (ascending)
- `formatEmployee(labelMap)` — renames fields according to a label map object

Then build a pipeline that gets Engineering employees earning above 85000, gives them a 10% raise, and sorts by salary:

```js
// Write your code here
```

---

### Exercise 3 — Hard: Implement `curry()` and `compose()`

**Part A**: Implement the `curry` helper function from scratch. It should:
- Accept a function `fn` of any arity
- Return a curried version that can be called in any combination (all at once, one at a time, grouped)

```js
function curry(fn) {
  // Write your implementation
}

// Tests your implementation must pass:
const add    = (a, b, c) => a + b + c;
const curried = curry(add);

curried(1)(2)(3);   // 6
curried(1, 2)(3);   // 6
curried(1)(2, 3);   // 6
curried(1, 2, 3);   // 6
```

**Part B**: Implement `compose(...fns)` — takes any number of functions and returns a new function that applies them **right to left** (mathematical function composition):

```js
function compose(...fns) {
  // Write your implementation
}

// f(g(h(x))) === compose(f, g, h)(x)

const double    = x => x * 2;
const addOne    = x => x + 1;
const square    = x => x * x;

const transform = compose(double, addOne, square);
transform(3);   // double(addOne(square(3))) = double(addOne(9)) = double(10) = 20
```

**Part C**: Combine both. Use `curry` and `compose` together to build a price processing pipeline:

```js
// Given these base functions:
const applyDiscount = (percent, price) => price * (1 - percent / 100);
const applyTax      = (rate, price)    => price * (1 + rate / 100);
const roundTo       = (decimals, num)  => Number(num.toFixed(decimals));

// Curry them, then compose a pipeline:
// Apply 10% discount, then add 18% tax, then round to 2 decimal places

const processPrice = /* your composed pipeline */;

processPrice(1000);  // 1000 → 900 → 1062 → 1062.00
processPrice(2500);  // 2500 → 2250 → 2655 → 2655.00
```

```js
// Write your code here
```

---

## Quick reference — cheat sheet

```js
// Manual currying
const fn = a => b => c => expression;
fn(1)(2)(3);

// Partial application — fix first arg
const specialised = fn(fixedArg);
specialised(otherArg);

// Using with array methods
arr.map(curriedFn(config));
arr.filter(curriedPredicate(threshold));

// curry() helper
function curry(fn) {
  return function curried(...args) {
    return args.length >= fn.length
      ? fn.apply(this, args)
      : (...more) => curried(...args, ...more);
  };
}

// compose — right to left
const compose = (...fns) => x => fns.reduceRight((acc, fn) => fn(acc), x);

// pipe — left to right (more readable for pipelines)
const pipe    = (...fns) => x => fns.reduce((acc, fn) => fn(acc), x);
```

| Concept | Description | Example |
|---|---|---|
| Currying | Unary function chain | `a => b => a + b` |
| Partial application | Fix some args, open rest | `const double = multiply(2)` |
| Composition | Output of one → input of next | `compose(f, g)(x) = f(g(x))` |
| Pipe | Same as compose, left-to-right | `pipe(g, f)(x) = f(g(x))` |

---

## Connected topics

- **[19] Closures** — every curried function uses closure to remember earlier args
- **[20] Higher-order Functions** — currying produces and consumes functions
- **[41] Pure Functions** — curried functions are naturally pure and composable
- **[44] Memoization** — curried functions are safe to memoize (pure, deterministic)
- **[15] Arrow Functions** — the `a => b => c =>` syntax is idiomatic with arrow functions
