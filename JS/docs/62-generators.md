# 62 — Generators

## What is this?

A **generator** is a special kind of function that can **pause its execution mid-way and resume later** — yielding a sequence of values one at a time instead of computing and returning everything at once. Think of a generator like a TV show on demand: instead of handing you the entire season at once (a normal function returning an array), it streams one episode at a time, only producing the next episode when you ask for it.

```js
function* count() {   // the * makes it a generator function
  yield 1;
  yield 2;
  yield 3;
}

const gen = count();    // creates a generator object — function body does NOT run yet

gen.next();   // { value: 1, done: false } — runs until first yield
gen.next();   // { value: 2, done: false } — resumes, runs until second yield
gen.next();   // { value: 3, done: false } — resumes, runs until third yield
gen.next();   // { value: undefined, done: true } — function body is exhausted
```

---

## Why does it matter?

Generators are the foundation of several powerful patterns:

- **Lazy sequences** — generate values on demand, never computing what isn't needed. Infinite sequences become trivial.
- **Custom iterators** — the cleanest way to make any object iterable (`Symbol.iterator` + generator)
- **Async/await** — `async/await` is literally syntactic sugar compiled to generators + Promises. Understanding generators means you understand what `async/await` does under the hood.
- **Coroutines** — two-way communication: the caller can send values INTO the generator while receiving values out of it
- **Data pipelines** — compose generators into lazy processing pipelines (like Unix pipes but in JS)
- **State machines** — represent complex stateful logic where each `yield` is a state transition

---

## Syntax

```js
// Generator function declaration
function* generatorFn() {
  yield "first";
  yield "second";
}

// Generator function expression
const gen = function* () {
  yield 1;
  yield 2;
};

// Generator method in a class or object
class MyClass {
  *[Symbol.iterator]() {     // generator as iterator
    yield this.a;
    yield this.b;
  }
}

const obj = {
  *values() {
    yield 1;
    yield 2;
  },
};

// Calling a generator function returns a Generator object (does NOT run the body)
const iterator = generatorFn();

// Calling .next() runs the body until the next yield (or return)
iterator.next();   // { value: "first",  done: false }
iterator.next();   // { value: "second", done: false }
iterator.next();   // { value: undefined, done: true }
```

---

## How it works — step by step

### 1. Generator function vs normal function

```
Normal function:
  Called → runs ENTIRE body → returns one value → done

Generator function:
  Called → returns a Generator object immediately (body NOT running)
  .next() → runs body until next yield → pauses
  .next() → resumes from pause point → runs until next yield → pauses
  .next() → ... repeat ...
  .next() → runs past last yield (or hits return) → { value: undefined, done: true }
```

### 2. The Generator object is both an iterator AND an iterable

A Generator object has:
- `.next()` — run to next yield (iterator protocol)
- `.return(value)` — terminate the generator early, returns `{ value, done: true }`
- `.throw(error)` — inject an error at the current pause point

It also has `[Symbol.iterator]()` that returns `this` — so it is its own iterable.

```js
function* letters() {
  yield "a";
  yield "b";
  yield "c";
}

const gen = letters();

// As iterator: call .next() manually
gen.next();  // { value: "a", done: false }

// As iterable: use for...of, spread, destructuring
for (const letter of letters()) {
  console.log(letter);  // a, b, c
}

console.log([...letters()]);   // ["a", "b", "c"]

const [first, second] = letters();
console.log(first, second);   // "a" "b"
```

### 3. `return` inside a generator

A `return` statement (or reaching the end of the body) sets `done: true`:

```js
function* gen() {
  yield 1;
  return 99;   // done: true, value: 99
  yield 2;     // unreachable
}

const g = gen();
g.next();   // { value: 1, done: false }
g.next();   // { value: 99, done: true }
g.next();   // { value: undefined, done: true }

// IMPORTANT: for...of and spread IGNORE the return value
console.log([...gen()]);  // [1]  — 99 is not included
```

---

## Example 1 — infinite sequences

```js
// An infinite counter — would be impossible with an array
function* naturals(start = 1) {
  let n = start;
  while (true) {    // infinite loop is fine — we only run it when .next() is called
    yield n++;
  }
}

const gen = naturals();
gen.next().value;   // 1
gen.next().value;   // 2
gen.next().value;   // 3
// ... as many as you want

// Take the first n values from any generator:
function take(gen, n) {
  const result = [];
  for (const value of gen) {
    result.push(value);
    if (result.length >= n) break;   // stop consuming; generator stays paused
  }
  return result;
}

take(naturals(), 5);     // [1, 2, 3, 4, 5]
take(naturals(10), 5);   // [10, 11, 12, 13, 14]

// Fibonacci — infinite sequence
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

take(fibonacci(), 8);   // [0, 1, 1, 2, 3, 5, 8, 13]
```

---

## Example 2 — custom iterables via generators

```js
// The cleanest way to implement Symbol.iterator is with a generator

class InfiniteRange {
  constructor(start = 0, step = 1) {
    this.start = start;
    this.step  = step;
  }

  *[Symbol.iterator]() {
    let current = this.start;
    while (true) {
      yield current;
      current += this.step;
    }
  }
}

const evens = new InfiniteRange(0, 2);

for (const n of evens) {
  if (n > 10) break;
  console.log(n);   // 0, 2, 4, 6, 8, 10
}

// ─────────────────────────────────────────────────────────────────────
// Finite custom iterable — binary tree in-order traversal
class BinaryTree {
  constructor(value, left = null, right = null) {
    this.value = value;
    this.left  = left;
    this.right = right;
  }

  *[Symbol.iterator]() {
    if (this.left)  yield* this.left;   // delegate to sub-generator
    yield this.value;
    if (this.right) yield* this.right;
  }
}

const tree = new BinaryTree(
  4,
  new BinaryTree(2, new BinaryTree(1), new BinaryTree(3)),
  new BinaryTree(6, new BinaryTree(5), new BinaryTree(7))
);

console.log([...tree]);   // [1, 2, 3, 4, 5, 6, 7] — in-order traversal!
```

---

## `yield*` — delegating to another iterable

`yield*` delegates to another iterable (or generator), yielding each of its values in turn:

```js
function* concat(...iterables) {
  for (const iterable of iterables) {
    yield* iterable;   // yield all values from each iterable
  }
}

console.log([...concat([1, 2], [3, 4], [5])]);  // [1, 2, 3, 4, 5]
console.log([...concat("abc", [4, 5])]);         // ["a", "b", "c", 4, 5]

// ---

// yield* with another generator
function* inner() {
  yield "x";
  yield "y";
}

function* outer() {
  yield "start";
  yield* inner();    // delegate — yields "x" and "y"
  yield "end";
}

console.log([...outer()]);  // ["start", "x", "y", "end"]

// ---

// yield* returns the return value of the delegated generator
function* inner2() {
  yield 1;
  return "inner-done";   // this becomes the result of yield* in outer
}

function* outer2() {
  const result = yield* inner2();   // result = "inner-done" (the return value)
  yield result;
}

console.log([...outer2()]);  // [1, "inner-done"]
```

---

## Two-way communication — sending values into a generator

This is one of the most powerful and least-known features. The value passed to `.next(value)` becomes the **result of the `yield` expression** inside the generator:

```js
function* calculator() {
  console.log("Generator started");

  const a = yield "Give me the first number";   // pauses; a = whatever .next(value) sends
  const b = yield "Give me the second number";  // pauses; b = whatever .next(value) sends

  yield `Result: ${a} + ${b} = ${a + b}`;
}

const calc = calculator();

calc.next();          // starts generator; returns { value: "Give me the first number", done: false }
calc.next(10);        // sends 10 → a = 10; returns { value: "Give me the second number", done: false }
calc.next(5);         // sends 5  → b = 5;  returns { value: "Result: 10 + 5 = 15", done: false }
calc.next();          // { value: undefined, done: true }
```

**Critical rule:** The value passed to the FIRST `.next()` is always discarded — there is no `yield` expression to receive it yet. The generator hasn't run at all when you call the first `.next()`.

```js
function* gen() {
  const x = yield "first yield";
  console.log("x =", x);       // x = whatever you pass to SECOND .next()
}

const g = gen();
g.next("this is lost");   // first .next() — "this is lost" is IGNORED
g.next("hello");          // sends "hello" → x = "hello"
// logs: x = hello
```

---

## `.return()` and `.throw()` — controlling generators from outside

```js
function* gen() {
  try {
    yield 1;
    yield 2;
    yield 3;
  } finally {
    console.log("cleanup!");  // runs when generator is terminated
  }
}

const g = gen();
g.next();               // { value: 1, done: false }
g.return("done");       // logs "cleanup!"; returns { value: "done", done: true }
g.next();               // { value: undefined, done: true }

// ---

// .throw() injects an error at the current yield point
function* safeGen() {
  try {
    yield "a";
    yield "b";
  } catch (err) {
    console.log("caught:", err.message);
    yield "recovered";
  }
}

const sg = safeGen();
sg.next();              // { value: "a", done: false }
sg.throw(new Error("oops"));
// logs: caught: oops
// returns: { value: "recovered", done: false }
sg.next();              // { value: undefined, done: true }
```

---

## Example 3 — lazy data pipeline (like Unix pipes)

```js
// Each generator in the pipeline only processes data when downstream asks for it.
// Nothing is loaded into memory all at once.

function* csvLines(rawText) {
  for (const line of rawText.split("\n")) {
    if (line.trim()) yield line;
  }
}

function* parseCSV(lines) {
  let headers = null;
  for (const line of lines) {
    const values = line.split(",").map(v => v.trim());
    if (!headers) {
      headers = values;
    } else {
      yield Object.fromEntries(headers.map((h, i) => [h, values[i]]));
    }
  }
}

function* filterRows(rows, predicate) {
  for (const row of rows) {
    if (predicate(row)) yield row;
  }
}

function* mapRows(rows, transform) {
  for (const row of rows) {
    yield transform(row);
  }
}

// Pipeline: read → parse → filter → transform
const raw = `name,age,city
Alice,30,NYC
Bob,17,LA
Carol,25,Chicago
Dave,15,Miami`;

const pipeline = mapRows(
  filterRows(
    parseCSV(csvLines(raw)),
    row => parseInt(row.age) >= 18
  ),
  row => ({ ...row, age: parseInt(row.age) })
);

for (const row of pipeline) {
  console.log(row);
}
// { name: 'Alice', age: 30, city: 'NYC' }
// { name: 'Carol', age: 25, city: 'Chicago' }
// Memory: only ONE row is in memory at a time — even if the CSV is 10GB!
```

---

## Example 4 — state machine

```js
// Generators are natural state machines — each yield is a state

function* trafficLight() {
  while (true) {
    yield "green";   // 30s
    yield "yellow";  // 5s
    yield "red";     // 30s
  }
}

function* orderStateMachine(orderId) {
  console.log(`Order ${orderId} created`);
  yield "PENDING";

  console.log(`Order ${orderId} payment received`);
  yield "PAID";

  console.log(`Order ${orderId} shipped`);
  yield "SHIPPED";

  console.log(`Order ${orderId} delivered`);
  return "DELIVERED";   // final state — done: true
}

const order = orderStateMachine("ORD-001");
order.next().value;   // "PENDING"   — logs: "Order ORD-001 created"
order.next().value;   // "PAID"      — logs: "Order ORD-001 payment received"
order.next().value;   // "SHIPPED"   — logs: "Order ORD-001 shipped"
order.next().value;   // "DELIVERED" — logs: "Order ORD-001 delivered", done: true
```

---

## Generators and async/await — the connection

`async/await` is compiled to a generator + Promise runner. Understanding this explains all of `async/await`'s behaviour:

```js
// This async function:
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  const user     = await response.json();
  return user;
}

// Is approximately equivalent to this generator + runner:
function* fetchUserGen(id) {
  const response = yield fetch(`/api/users/${id}`);
  const user     = yield response.json();
  return user;
}

// A "Promise runner" (similar to what the JS engine does for async/await):
function run(generatorFn, ...args) {
  const gen = generatorFn(...args);

  function step(result) {
    const { value, done } = gen.next(result);
    if (done) return Promise.resolve(value);
    return Promise.resolve(value).then(step, (err) => {
      gen.throw(err);  // inject the rejection as an error at the yield point
    });
  }

  return step(undefined);
}

// run(fetchUserGen, 1)  ≈  fetchUser(1)
```

This is exactly what Babel compiled `async/await` to before native support, and what libraries like `co` (used with early Koa.js) implemented.

---

## Generator utilities

```js
// ── take(n) — get first n values ─────────────────────────────────
function take(iterable, n) {
  const result = [];
  for (const val of iterable) {
    result.push(val);
    if (result.length >= n) break;
  }
  return result;
}

// ── map — lazy map over any iterable ─────────────────────────────
function* map(iterable, fn) {
  for (const val of iterable) {
    yield fn(val);
  }
}

// ── filter — lazy filter ──────────────────────────────────────────
function* filter(iterable, predicate) {
  for (const val of iterable) {
    if (predicate(val)) yield val;
  }
}

// ── zip — interleave two iterables ───────────────────────────────
function* zip(iter1, iter2) {
  const a = iter1[Symbol.iterator]();
  const b = iter2[Symbol.iterator]();
  while (true) {
    const { value: va, done: da } = a.next();
    const { value: vb, done: db } = b.next();
    if (da || db) return;
    yield [va, vb];
  }
}

// ── range — like Python's range ───────────────────────────────────
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) {
    yield i;
  }
}

// Compose them all:
const result = take(
  filter(
    map(range(0, 100), x => x * x),  // [0, 1, 4, 9, 16, 25 ...]
    x => x % 2 === 0                  // [0, 4, 16, 36 ...]
  ),
  5
);
console.log(result);  // [0, 4, 16, 36, 64]
```

---

## Async generators

Async generators combine generators with async/await — they can `yield` Promises and `await` inside:

```js
async function* paginate(endpoint, token) {
  let url  = `${endpoint}?limit=100`;

  while (url) {
    const response = await fetch(url, {    // await inside async generator
      headers: { Authorization: `Bearer ${token}` },
    });

    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    const { data, nextPage } = await response.json();
    yield data;                             // yield the page of results

    url = nextPage ?? null;
  }
}

// Consume with for-await-of:
async function processAllUsers(token) {
  for await (const page of paginate("https://api.example.com/users", token)) {
    for (const user of page) {
      await processUser(user);
    }
    console.log(`Processed page of ${page.length} users`);
  }
}

// Only loads one page at a time — memory safe for millions of records
```

---

## Tricky things

### 1. First `.next()` value is ALWAYS discarded

```js
function* gen() {
  const x = yield "yielded";
  return x;
}

const g = gen();
g.next("ignored");   // first call — "ignored" is LOST; returns { value: "yielded", done: false }
g.next("kept");      // sends "kept" → x = "kept"; returns { value: "kept", done: true }
```

### 2. `for...of` and spread ignore the `return` value

```js
function* gen() {
  yield 1;
  return "final";
  yield 2;   // unreachable
}

console.log([...gen()]);            // [1] — "final" is NOT included
for (const v of gen()) console.log(v);  // logs: 1
// To get the return value, use .next() manually until done: true
```

### 3. Generators cannot be restarted

```js
function* gen() { yield 1; yield 2; }

const g = gen();
[...g];       // [1, 2] — exhausted
[...g];       // []     — already done! Cannot restart.

// To restart: create a new generator
const g2 = gen();   // fresh iterator
[...g2];      // [1, 2]
```

### 4. Arrow functions cannot be generators

```js
const gen = () => { yield 1; };    // SyntaxError: Unexpected strict mode reserved word
const gen = *() => { yield 1; };  // SyntaxError

// Generator functions must use the function* syntax:
const gen = function* () { yield 1; };   // ✓
```

### 5. `yield` is only valid inside the generator itself — not in nested callbacks

```js
function* gen() {
  [1, 2, 3].forEach(n => {
    yield n;   // SyntaxError: yield is only valid inside generator functions
  });
}

// Fix — use for...of:
function* gen() {
  for (const n of [1, 2, 3]) {
    yield n;   // ✓
  }
}
// Or yield*:
function* gen() {
  yield* [1, 2, 3];   // ✓
}
```

### 6. `yield*` returns the `return` value of the delegated generator (not the yielded values)

```js
function* inner() {
  yield 1;
  yield 2;
  return "inner returned";   // THIS is what yield* evaluates to
}

function* outer() {
  const result = yield* inner();   // yields 1 and 2 to caller; result = "inner returned"
  yield result;                    // yields "inner returned" to caller
}

console.log([...outer()]);  // [1, 2, "inner returned"]
```

---

## Common mistakes

### Mistake 1 — Calling a generator function and expecting it to run immediately

```js
function* greet() {
  console.log("Hello!");
  yield "done";
}

// WRONG — expects console.log to run
const g = greet();   // nothing logs — body has NOT run

// RIGHT — first .next() triggers execution
g.next();   // logs: "Hello!", returns { value: "done", done: false }
```

### Mistake 2 — Reusing an exhausted generator

```js
function* gen() { yield 1; yield 2; }

const g = gen();
console.log([...g]);   // [1, 2]
console.log([...g]);   // [] — exhausted! Should have created a new generator.

// RIGHT — if you need multiple iterations, call the factory again
const values = () => gen();   // wrap in a function to create fresh generators
console.log([...values()]);   // [1, 2]
console.log([...values()]);   // [1, 2]
```

### Mistake 3 — Using `yield` inside a regular function within a generator

```js
// WRONG
function* gen() {
  const items = [1, 2, 3];
  items.forEach(item => yield item);   // SyntaxError

  // WRONG — setTimeout callback
  setTimeout(() => { yield "late"; }, 1000);  // SyntaxError
}

// RIGHT — use for...of or yield*
function* gen() {
  for (const item of [1, 2, 3]) {
    yield item;   // ✓
  }
}
```

### Mistake 4 — Ignoring the two-way nature and never sending values in

```js
// A generator designed to receive values but called like a one-way iterator:
function* accumulator() {
  let total = 0;
  while (true) {
    const amount = yield total;
    if (amount === null) return total;
    total += amount;
  }
}

// WRONG — never sending values, getting undefined accumulation
const acc = accumulator();
acc.next();    // { value: 0, done: false }
acc.next();    // amount = undefined → total = NaN
acc.next();    // total = NaN

// RIGHT — send values in
const acc2 = accumulator();
acc2.next();        // start — { value: 0, done: false }
acc2.next(10);      // amount = 10 → total = 10; { value: 10, done: false }
acc2.next(5);       // amount = 5  → total = 15; { value: 15, done: false }
acc2.next(null);    // return total → { value: 15, done: true }
```

---

## Practice exercises

### Exercise 1 — easy

Write these generator functions:
1. `range(start, end, step = 1)` — yields integers from `start` up to (but not including) `end`, incrementing by `step`
2. `repeat(value, times)` — yields `value` exactly `times` times (or infinitely if `times` is undefined)
3. `cycle(array)` — yields items from the array cyclically, forever: `[1,2,3]` → 1, 2, 3, 1, 2, 3, ...

Test them with `for...of` (with a break condition for infinite ones) and spread.

```js
// Write your code here
```

### Exercise 2 — medium

Implement a lazy pipeline utility:

```js
// pipe(...generators)(source) — composes generator transforms left-to-right

function* map(fn)         { /* yield fn(x) for each x */ }
function* filter(pred)    { /* yield x only if pred(x) */ }
function* take(n)         { /* yield first n values */ }
function* flatMap(fn)     { /* yield* fn(x) for each x */ }
function* tap(fn)         { /* yield x unchanged, call fn(x) as side effect */ }

function pipe(...transforms) {
  return function(source) {
    // apply each transform in sequence
  };
}

// Test:
const result = pipe(
  map(x => x * x),
  filter(x => x % 2 === 0),
  tap(x => console.log("passing:", x)),
  take(4)
)(range(1, 100));

console.log([...result]);
// logs: passing: 4, passing: 16, passing: 36, passing: 64
// result: [4, 16, 36, 64]
```

```js
// Write your code here
```

### Exercise 3 — hard

Build an **async generator-based job queue** for a Node.js backend:

```js
// JobQueue class:
// - constructor(concurrency = 3) — max concurrent jobs
// - add(fn) — adds an async function (job) to the queue; returns a Promise for the result
// - async *[Symbol.asyncIterator]() — async generator that yields job results as they complete
// - get stats() — returns { pending, running, completed, failed }

// processJobs(queue) — async generator that:
//   - Receives jobs from a source
//   - Yields { jobId, result } or { jobId, error } for each completed job
//   - Respects the concurrency limit
//   - Handles individual job failures without stopping the queue

// Usage:
const queue = new JobQueue(3);

// Add 10 "jobs" (simulated with delays):
for (let i = 1; i <= 10; i++) {
  queue.add(() => simulateWork(i));
}

for await (const { jobId, result, error } of queue) {
  if (error) console.error(`Job ${jobId} failed:`, error.message);
  else       console.log(`Job ${jobId} done:`, result);
}

console.log(queue.stats);  // { pending: 0, running: 0, completed: 8, failed: 2 }
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax / Detail |
|---|---|
| Generator function | `function* gen() { yield value; }` |
| Generator method | `*methodName() { yield value; }` |
| Generator expression | `const g = function* () { yield 1; }` |
| Arrow generators | NOT POSSIBLE — must use `function*` |
| Call generator | `const iter = gen()` — body does NOT run |
| Advance | `iter.next()` → `{ value, done }` |
| Send value in | `iter.next(val)` — val becomes result of `yield` expression |
| Terminate early | `iter.return(val)` → `{ value: val, done: true }` |
| Inject error | `iter.throw(err)` — throws at current yield point |
| `yield` | Pauses, emits value to caller |
| `yield*` | Delegates to another iterable; evaluates to its return value |
| `return` in generator | Sets `done: true`; value ignored by `for...of` and spread |
| First `.next()` arg | Always discarded — no yield expression to receive it |
| Exhausted generator | Cannot be restarted — create a new one |
| `yield` in callback | SyntaxError — use `for...of` or `yield*` instead |
| `for...of` | Works on generators (ignores return value) |
| Spread `[...]` | Works on generators (ignores return value) |
| Destructuring | Works on generators |
| Infinite generators | Safe — only compute when `.next()` is called |
| Async generator | `async function* gen() { yield await fetch(...) }` |
| Consume async gen | `for await (const val of gen()) { }` |

---

## Connected topics

- **52 — Event loop** — generators pause execution without blocking the thread; async/await is generators + Promises
- **56 — async/await** — `async/await` is generators + a Promise runner under the hood
- **60 — Symbol** — `Symbol.iterator` and `Symbol.asyncIterator` are what make generators iterable
- **61 — WeakMap/WeakSet** — generators combined with WeakMap enable memory-safe lazy computation
- **63 — Proxy and Reflect** — Proxy traps can intercept generator method calls
