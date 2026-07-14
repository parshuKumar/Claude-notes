# 44 — Iterator Pattern
## Category: LLD Patterns

---

## What is this?

The **Iterator pattern** gives you a standard way to walk through a collection **one element at a time, without knowing how that collection is built inside**. Whether the data is an array, a binary tree, a linked list, or page 7 of a REST API, the consuming code looks identical: `for (const x of thing)`.

Think of a TV remote. You press "next channel" and you get the next channel. You have no idea whether the channels live in an array, a linked list, or are streamed over cable. The remote exposes exactly one verb — **next** — and that's enough.

JavaScript builds this pattern directly into the language. `for...of`, spread (`...`), and destructuring are all just polite ways of calling `next()` in a loop. This doc teaches the protocol properly, then covers the two things most JS developers never learn: **generators** and **async iterators**.

---

## Why does it matter?

Without Iterator, every consumer of a collection has to know its shape:

```javascript
// The code you write when you DON'T have iterators — shape-specific loops everywhere
for (let i = 0; i < arr.length; i++) { use(arr[i]); }           // array
for (let n = list.head; n !== null; n = n.next) { use(n.val); } // linked list
walkInOrder(tree.root, use);                                    // tree — needs recursion
let page = 1, more = true;                                      // paginated API
while (more) {
  const { items, hasMore } = await (await fetch(`/u?page=${page++}`)).json();
  items.forEach(use);
  more = hasMore;
}
```

Four data sources, four completely different traversal styles. Now write one function `countMatching(collection, predicate)` that works on all four. You can't — unless they share an interface. **That interface is the iterator.**

**Interview angle:** Direct questions ("implement an iterator for a binary tree") are common, but the real payoff is the follow-up: *"How would you process a 40 GB CSV in Node without running out of memory?"* Both answers are iterators.

**Real-work angle — this is where it earns its money in Node:** an **async iterator** lets you stream a million database rows or 500 API pages through your program **without ever holding more than one page in memory**. If you've ever crashed a Node process with `JavaScript heap out of memory` while loading a big table, this pattern is the fix. Section 6 is the one to actually internalise.

---

## The core idea — explained simply

### The Library Book-Cart Analogy

You're a researcher who needs every book in a library that mentions "Byzantium."

**Option A — the bad way.** The librarian says: "Sure, here's the entire library." Six thousand books are dumped on your desk. You now (a) need a desk the size of a warehouse, and (b) have to understand the shelving scheme yourself — Dewey Decimal? By author? By colour? You're coupled to the library's internal organisation, and you had to physically hold the whole thing to look at any of it.

**Option B — the iterator.** You ask for a **book cart with an assistant**. You say **"next"**. The assistant hands you one book. You read it, put it down, say "next" again. Eventually the assistant says **"that's all"** and you stop.

You never learned the shelving scheme. You never held more than one book. And the assistant might be fetching books from the basement, or from another branch by courier — you'd never know. The interface is identical either way.

That assistant is the iterator. It has exactly one method: `next()`. Its answer is always the same shape: **"here's a book"** or **"we're done."**

| Library | Iterator pattern | JavaScript |
|---|---|---|
| The library | **Iterable** — the collection | An object with a `[Symbol.iterator]()` method |
| "Give me a cart" | `createIterator()` | `iterable[Symbol.iterator]()` |
| The assistant with the cart | **Iterator** — holds the position | An object with a `next()` method |
| "Next" | `next()` | `it.next()` |
| A book handed to you | The element | `{ value: book, done: false }` |
| "That's all" | Exhausted | `{ value: undefined, done: true }` |
| You, reading | **Client** | `for (const book of library) { ... }` |
| Not knowing the shelving scheme | **Encapsulation** | You never touch `.head`, `.root`, or `.length` |
| Never holding 6,000 books | **Laziness** — one at a time | The whole reason async iterators matter |

Two properties fall out of this, and they are the entire pattern:

1. **Encapsulation** — the client doesn't know the internal structure.
2. **Laziness** — elements are produced on demand, not all up front. This is what makes an *infinite* collection possible, and a 40 GB file processable in 50 MB of RAM.

---

## Key concepts inside this topic

### 1. The problem it solves — painful code first

You write a `Playlist` class backed by an array. Consumers do this:

```javascript
// BAD — the client reaches into the internals
class Playlist {
  constructor() { this.songs = []; }   // public field. Uh oh.
  add(s) { this.songs.push(s); }
}
const p = new Playlist();
p.add('Song A'); p.add('Song B');

// Every consumer now depends on `.songs` being an ARRAY:
for (let i = 0; i < p.songs.length; i++) console.log(p.songs[i]);
const shuffled = [...p.songs].sort(() => Math.random() - 0.5);
```

Six months later you switch the storage to a linked list (for O(1) reordering), or a Map keyed by song ID (for dedup). **Every single consumer breaks.** `.length` is gone. `p.songs[i]` is gone. You leaked your internal representation into every file in the codebase.

```javascript
// GOOD — internals are private, and the class is ITERABLE
class Playlist {
  #songs = [];                        // truly private (# = ES private field)
  add(s) { this.#songs.push(s); return this; }

  // This ONE method makes the class work with for...of, spread, destructuring,
  // Array.from, Map/Set constructors, yield*, and Promise.all — all for free.
  *[Symbol.iterator]() { yield* this.#songs; }
}

const p = new Playlist().add('Song A').add('Song B');
for (const song of p) console.log(song);   // works
console.log([...p]);                       // works
const [first] = p;                         // works
```

Now swap `#songs` for a linked list and change `[Symbol.iterator]` to match. **Not one consumer changes.** That is the pattern.

### 2. The structure — participants

```
Client ──asks──▶ Iterable ──creates──▶ Iterator ──reads──▶ (hidden internals)
                                          │
                                     next() → { value, done }
```

| Participant | Job | In JavaScript |
|---|---|---|
| **Iterable / Aggregate** | The collection | Any object with `[Symbol.iterator]()` |
| **Iterator** | Holds traversal state | An object with `next()` → `{ value, done }` |
| **Client** | The consumer | `for...of`, spread, destructuring, `Array.from`, `yield*` |

Two rules that trip people up:

- **Iterable ≠ Iterator.** An *iterable* produces iterators. An *iterator* is a single, one-shot walk. An array is iterable and can be looped a hundred times; each `for...of` gets a **fresh** iterator with position reset to 0.
- **An iterator should also be iterable** — returning itself from `[Symbol.iterator]()`. That's why you can `for...of` a generator object directly.

### 3. The iteration protocol — the real machinery

**Iterable protocol:** an object is *iterable* if it has a method keyed by the well-known symbol `Symbol.iterator` returning an **iterator**.

**Iterator protocol:** an object is an *iterator* if it has a `next()` returning `{ value, done }`. `done: false` → `value` is the next element. `done: true` → the walk is over.

Built by hand, so you can see the mechanism:

```javascript
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;   // traversal state lives in the ITERATOR, not the iterable —
    const last = this.to;      // which is why two loops over `range` never interfere
    return {
      next() {
        return current <= last ? { value: current++, done: false }
                               : { value: undefined, done: true };
      },
      [Symbol.iterator]() { return this; },  // convention: iterators are iterable too
      // OPTIONAL but important: called when a loop exits EARLY (break/return/throw).
      // This is your cleanup hook — close file handles, DB cursors, sockets HERE.
      return() { console.log('  (cleaned up early)'); return { done: true }; },
    };
  },
};
for (const n of range) console.log(n);   // 1 2 3 4 5
const [a, b] = range;                    // a=1, b=2 — stops early, and return() fires

// THE REVEAL — `for (const x of obj)` is literally just this:
const it = obj[Symbol.iterator]();
let r;
while (!(r = it.next()).done) { const x = r.value; /* your loop body */ }
```

Everything that "just works" on arrays and strings works because they implement this protocol:

| Syntax | What it does under the hood |
|---|---|
| `for (const x of it)` | Calls `next()` until `done` |
| `[...it]` and `Array.from(it)` | Drains the iterator into a new array |
| `const [a, b] = it` | Calls `next()` exactly twice, then calls `return()` |
| `new Set(it)` / `new Map(it)` / `Promise.all(it)` / `f(...it)` | Drains it |
| `yield* it` | Delegates — drains it, yielding each value |

**One protocol. Eight pieces of syntax, free.** That's the payoff for implementing a single method.

### 4. Custom iterables — a BinaryTree and a LinkedList

The classic interview implementation: a BST whose in-order traversal yields values in **sorted order** — written the hard way, with an explicit stack, so you can see the state machine.

```javascript
// ─── BINARY SEARCH TREE, iterator written BY HAND ────────────────
class BinaryTree {
  #root = null;
  insert(value) {
    const node = { value, left: null, right: null };
    if (!this.#root) { this.#root = node; return this; }
    let cur = this.#root;
    while (true) {
      const dir = value < cur.value ? 'left' : 'right';
      if (!cur[dir]) { cur[dir] = node; return this; }
      cur = cur[dir];
    }
  }
  // In-order traversal (left → node → right), done ITERATIVELY.
  // Recursion CAN'T pause — and an iterator MUST pause between elements.
  // So we keep the call stack ourselves, in an explicit array.
  [Symbol.iterator]() {
    const stack = [];
    let cur = this.#root;
    return {
      next() {
        while (cur) { stack.push(cur); cur = cur.left; }  // dive left, remembering the path
        if (stack.length === 0) return { value: undefined, done: true };
        const node = stack.pop();
        cur = node.right;                                 // next: explore the right subtree
        return { value: node.value, done: false };
      },
      [Symbol.iterator]() { return this; },
    };
  }
}

// ─── LINKED LIST — same interface, totally different internals ────
class LinkedList {
  #head = null; #tail = null;
  push(value) {
    const node = { value, next: null };
    if (!this.#head) this.#head = node; else this.#tail.next = node;
    this.#tail = node;
    return this;
  }
  [Symbol.iterator]() {
    let cur = this.#head;
    return {
      next() {
        if (!cur) return { value: undefined, done: true };
        const { value } = cur;
        cur = cur.next;
        return { value, done: false };
      },
      [Symbol.iterator]() { return this; },
    };
  }
}

// ─── THE PAYOFF: one function, ANY iterable ──────────────────────
function countMatching(iterable, predicate) {
  let n = 0;
  for (const item of iterable) if (predicate(item)) n++;
  return n;
}

const tree = new BinaryTree();
[50, 30, 70, 20, 40, 60, 80].forEach(v => tree.insert(v));
const list = new LinkedList().push('a').push('b').push('c');

console.log([...tree]);                              // [20,30,40,50,60,70,80] — SORTED,
                                                     // and the client never saw a tree
console.log(countMatching(tree, v => v > 45));       // 4   (a binary tree)
console.log(countMatching(list, v => v !== 'b'));    // 2   (a linked list)
console.log(countMatching([1, 2, 3], v => v > 1));   // 2   (an array)
console.log(countMatching('hello', c => c === 'l')); // 2   (a string!)
```

`countMatching` contains not one `if` about data structures — and `[...tree]` sorts, `Math.max(...tree)` works, `new Set(tree)` works. **That's the pattern working.**

### 5. Generators — the ergonomic way to write iterators

That hand-written tree iterator took 12 lines and an explicit stack. Generators let the *language* keep the stack for you.

A **generator function** (`function*`) can **pause**. `yield` produces a value and freezes execution — locals, loop position, everything — until someone calls `next()` again. Calling a generator function doesn't run the body; it returns a **generator object**, which is both an iterator **and** iterable. **`yield*` delegates** to another iterable, yielding every value from it — that's the key to recursion.

```javascript
function* countdown(from) {
  while (from > 0) yield from--;   // pauses here, resumes on the next next()
  return 'liftoff';                // becomes { value: 'liftoff', done: true }
}
console.log(countdown(3).next()); // { value: 3, done: false }
console.log([...countdown(3)]);   // [3, 2, 1] — the `return` value is NOT included

class BinaryTree {
  // ... same insert() as before ...

  // The ENTIRE in-order iterator. Four lines. Recursion is back — and it PAUSES.
  *#inOrder(node) {
    if (!node) return;
    yield* this.#inOrder(node.left);   // everything from the left subtree
    yield node.value;                  // then me
    yield* this.#inOrder(node.right);  // then everything from the right subtree
  }
  *[Symbol.iterator]() { yield* this.#inOrder(this.#root); }

  // Another traversal order, nearly free:
  *preOrder(node = this.#root) {
    if (!node) return;
    yield node.value;
    yield* this.preOrder(node.left);
    yield* this.preOrder(node.right);
  }
}
console.log([...tree]);            // [20,30,40,50,60,70,80]  in-order (sorted)
console.log([...tree.preOrder()]); // [50,30,20,40,70,60,80]
```

The explicit stack machine is now the JS engine's problem. **Same behaviour, one third the code, and no chance of an off-by-one bug in the stack juggling.**

Generators are also **lazy, and can be infinite** — which a plain array can never be:

```javascript
function* naturals() { let n = 1; while (true) yield n++; }   // never terminates

function* map(it, fn)   { for (const x of it) yield fn(x); }
function* filter(it, p) { for (const x of it) if (p(x)) yield x; }
function* take(it, n)   { for (const x of it) { if (n-- <= 0) return; yield x; } }

// A lazy pipeline over an INFINITE sequence. Nothing is computed until you pull.
console.log([...take(filter(map(naturals(), n => n * n), n => n % 2 === 0), 5)]);
// [4, 16, 36, 64, 100]
```

Try that with `.map().filter().slice()` on an array of all natural numbers. You can't — the array would have to exist first. Laziness isn't a micro-optimisation; it changes what's *possible*.

### 6. Async iterators — the one that actually matters in Node

You need to process every user in a table with 4 million rows. The naive version:

```javascript
// BAD — loads 4,000,000 rows into RAM. Node dies with:
//   FATAL ERROR: Reached heap limit — JavaScript heap out of memory
const users = await db.query('SELECT * FROM users');
for (const user of users) await sendEmail(user);
```

The fix is an **async iterator**: same protocol, but `next()` returns a **Promise** of `{ value, done }`, keyed by `Symbol.asyncIterator`. You consume it with **`for await...of`**.

```javascript
//   sync:   obj[Symbol.iterator]()      → { next(): { value, done } }
//   async:  obj[Symbol.asyncIterator]() → { next(): Promise<{ value, done }> }
```

**Use case A — paginating an API, lazily.** The caller writes a flat loop and never sees a page, a cursor, or a fetch.

```javascript
class PaginatedApi {
  constructor(baseUrl, pageSize = 100) {
    this.baseUrl = baseUrl;
    this.pageSize = pageSize;
  }
  // An async generator (async function*) — combines async/await with yield.
  async *[Symbol.asyncIterator]() {
    let cursor = null;
    do {
      const url = `${this.baseUrl}?limit=${this.pageSize}${cursor ? `&cursor=${cursor}` : ''}`;
      const res = await fetch(url);              // ONE network call...
      if (!res.ok) throw new Error(`API ${res.status}`);
      const { items, nextCursor } = await res.json();

      // ...then hand out items ONE at a time. The next page is not fetched until
      // the consumer has actually finished this one. That is the laziness.
      for (const item of items) yield item;

      cursor = nextCursor;
    } while (cursor);
  }
}

// The consumer. No pages. No cursors. No fetch. Just a loop.
for await (const user of new PaginatedApi('https://api.example.com/users')) {
  await sendEmail(user);
  // `break` here is safe: it stops the loop AND stops fetching further pages.
  // No wasted API calls. Try getting that from a "fetch everything first" helper.
}
```

Memory used: **one page**, not the whole dataset. And if the consumer `break`s on page 2, pages 3 through 500 are **never requested**.

**Use case B — streaming DB rows.** Real drivers already support this (`pg-query-stream`, `mysql2`'s `.stream()`, MongoDB cursors). Here's the shape, driver-agnostic:

```javascript
async function* streamRows(db, sql, batchSize = 1000) {
  let lastId = 0;
  while (true) {
    // Keyset cursor, NOT `OFFSET` — a large OFFSET makes the DB scan and discard
    // every skipped row, so page 4,000 is thousands of times slower than page 1.
    const rows = await db.query(`${sql} WHERE id > ? ORDER BY id LIMIT ?`, [lastId, batchSize]);
    if (rows.length === 0) return;   // done: the iterator ends
    yield* rows;                     // hand out this batch, one row at a time
    lastId = rows[rows.length - 1].id;
  }
}

// Process 4 million rows in a few hundred KB of memory.
let processed = 0;
for await (const row of streamRows(db, 'SELECT * FROM users')) {
  await sendEmail(row);
  if (++processed % 10_000 === 0) console.log(`${processed} done`);
}
```

**Use case C — Node streams are already async iterables.** This is the fact that makes the whole thing click:

```javascript
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

// Read a 40 GB log file line by line, in CONSTANT memory.
const rl = createInterface({ input: createReadStream('./huge.log'), crlfDelay: Infinity });
let errors = 0;
for await (const line of rl) {   // readline implements Symbol.asyncIterator
  if (line.includes('ERROR')) errors++;
}
```

Every Node `Readable` stream implements `Symbol.asyncIterator`. So do Kafka consumers, the AWS SDK's S3 paginators, and the Fetch API's response body. **Once you know the protocol, you consume all of them with the same three words: `for await of`.**

### 7. When NOT to use it / how it's abused

- **Don't hand-write an iterator when a generator will do.** Explicit `next()` state machines are error-prone. Use `function*` unless you need something exotic.
- **Iterators are single-use.** `const it = arr[Symbol.iterator]()`, then spread it twice → the second gives `[]`. Return a *fresh* iterator from `[Symbol.iterator]()` every time. The common bug: making the class itself the iterator, so a second `for...of` sees an exhausted collection.
- **Random access.** Need `collection[500]`? An iterator walks 500 elements to get there. Iterators are for *sequential* traversal.
- **Don't spread a huge or infinite iterable.** `[...naturals()]` hangs forever. `[...fourMillionRows]` reintroduces the exact memory problem you used the iterator to avoid. Laziness only helps if you *stay* lazy.
- **Performance:** `for...of` over an array is measurably slower than an indexed `for` loop (each step allocates a `{value, done}` object). In a hot numeric inner loop, use the index. Everywhere else, readability wins.
- **Don't forget `return()`** on iterators holding resources. If the consumer `break`s, your DB cursor or file handle leaks unless you close it in `return()` — or use `try/finally` inside a generator, which the engine runs for you on early exit.

### 8. Related patterns and how they differ

| Pattern | Relationship |
|---|---|
| **Composite** (38) | Composite builds trees; Iterator walks them. `yield*` recursion is the natural traversal for a Composite. A classic pair. |
| **Observer** (41) | **Pull vs push.** Iterator = *pull* — the consumer asks and controls the pace. Observer = *push* — the source fires whenever it likes. Exactly the difference between `for await...of` (backpressure free) and `stream.on('data')` (backpressure your problem). |
| **Visitor** (50) | Both add operations over a structure. Iterator hands you elements and lets *you* decide; Visitor pushes a typed operation into each node. Iterator for uniform elements, Visitor for heterogeneous node types. |
| **Producer-Consumer** (51) | An async iterator with an internal buffer IS a producer-consumer channel — with natural backpressure, since the producer can't outrun a consumer that pulls. |

---

## Visual / Diagram description

### Diagram 1 — Class structure: many shapes, one interface

```
                     ┌────────────────────────────────┐
                     │           CLIENT               │
                     │  for (const x of collection)   │
                     │  [...collection]               │
                     └───────────────┬────────────────┘
                                     │ knows ONLY this interface
                                     ▼
                     ┌────────────────────────────────┐
                     │       «protocol» Iterable      │
                     │  [Symbol.iterator]() → Iterator│
                     └───────────────┬────────────────┘
                                     │ implemented by
        ┌───────────────┬────────────┼─────────────┬──────────────┐
        ▼               ▼            ▼             ▼              ▼
 ┌────────────┐  ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐
 │ BinaryTree │  │ LinkedList │ │  Array   │ │  String  │ │ Generator   │
 │ #root      │  │ #head      │ │ (native) │ │ (native) │ │  object     │
 └─────┬──────┘  └─────┬──────┘ └────┬─────┘ └────┬─────┘ └──────┬──────┘
       │ creates       │             │            │              │
       ▼               ▼             ▼            ▼              ▼
 ┌────────────────────────────────────────────────────────────────────┐
 │                    «protocol» Iterator                             │
 │   next()   → { value, done }        ← the ONLY method required     │
 │   return() → { done: true }         ← optional: cleanup on break   │
 └────────────────────────────────────────────────────────────────────┘
```

The client's arrow points at the *protocol*, never at a concrete class. Five wildly different structures — a contiguous array, a tree of pointers, a lazy infinite generator — are interchangeable to the consumer.

### Diagram 2 — Sequence: what `for...of` actually does

```
  Client                  Iterable/Iterator        Internals
    │ ─ tree[Symbol.iterator]() ▶│ creates iterator    │
    │◀─────── iterator ──────────┤ (position = start)  │
    │ ─ next() ─────────────────▶│ dive left, pop ────▶│
    │◀── { value: 20, done: false } ───────────────────┤
    │   [loop body runs with x = 20]                   │
    │ ─ next() ─────────────────▶│ dive left, pop ────▶│
    │◀── { value: 30, done: false } ───────────────────┤
    │            ... repeats ...                       │
    │ ─ next() ─────────────────▶│ stack empty         │
    │◀── { value: undefined, done: true } ─────────────┤
    │   [loop exits] — or, on an early `break`, return() fires
    │                  and your cleanup runs (close cursors, handles)
```

The whole loop is a conversation of `next()` calls. The client never sees the tree, the stack, or a single pointer.

### Diagram 3 — Sync vs async: the memory difference

```
EAGER (loads everything)             LAZY ASYNC ITERATOR (one batch at a time)
┌──────────────────────┐             ┌──────────────────────┐
│  await db.all()      │             │  for await (row of   │
└──────────┬───────────┘             │        stream(db))   │
           │                         └──────────┬───────────┘
           ▼                                    ▼ pull 1 batch
┌──────────────────────┐             ┌──────────────────────┐
│  RAM: 4,000,000 rows │  ~3.2 GB    │  RAM: 1,000 rows     │  ~0.8 MB
│  ████████████████████│  OOM CRASH  │  █                   │   ✓
└──────────────────────┘             └──────────┬───────────┘
                                                ▼ next batch (memory REUSED)
                                     ┌──────────────────────┐
                                     │  RAM: 1,000 rows     │  ~0.8 MB
                                     └──────────────────────┘
```

The arithmetic: 4,000,000 rows × ~800 bytes/row ≈ **3.2 GB** — well past Node's default ~2 GB heap ceiling, so the eager version crashes. Batches of 1,000 rows ≈ **800 KB** resident at any moment, whether the table has 4 million rows or 4 billion. **Constant memory, unbounded data.** That is the single most valuable idea in this document.

---

## Real world examples

### 1. Node.js core — streams are async iterables

Since Node 10, every `stream.Readable` implements `Symbol.asyncIterator`. This is why `for await (const chunk of fs.createReadStream(path))` works, and why `readline` lets you process a multi-gigabyte log file line by line in constant memory. Before this, the same job needed `.on('data')` handlers plus manual `.pause()`/`.resume()` calls to manage backpressure. The iterator protocol gives you backpressure **for free**: the stream can't push data at you, because you're pulling — the next chunk isn't read until your `await`ed loop body finishes.

### 2. AWS SDK v3 — paginators

The AWS JS SDK ships `paginateListObjectsV2`, `paginateScan` (DynamoDB), `paginateQuery`, and dozens more. Each returns an **async iterable** that hides `NextContinuationToken` / `LastEvaluatedKey` bookkeeping entirely — you write `for await (const page of paginateListObjectsV2({ client: s3 }, { Bucket: 'my-bucket' }))` and never touch a token. A bucket with 10 million objects works identically to one with 10: the SDK fetches 1,000 keys per request as you consume them, and stops the moment you `break`.

### 3. Database drivers — cursors

MongoDB's `collection.find()` returns an async-iterable cursor. PostgreSQL's `pg-query-stream` and `mysql2`'s `connection.query(...).stream()` do the same. In each case the DB keeps a server-side cursor open and the driver fetches rows in batches as your loop pulls them. This is how ETL jobs, nightly exports, and migration scripts process tables far larger than available RAM — the Iterator pattern is the entire reason those scripts don't fall over.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| Encapsulation | The client never touches `.root`, `.head`, `.length` — internals stay free to change |
| One interface, N structures | `countMatching()` works on trees, lists, strings, streams, APIs |
| Laziness + infinite sequences | Only compute what's consumed; `break` early and the rest is never produced |
| Constant memory | Process datasets larger than RAM (the async iterator superpower) |
| Free syntax | One method unlocks `for...of`, spread, destructuring, `Array.from`, `Set`, `yield*` |
| Backpressure by default | Pull-based consumption means the producer cannot overwhelm you |

| Cons | The cost you pay |
|---|---|
| Slower than an index loop | Each `next()` allocates a `{value, done}` object — matters only in hot numeric loops |
| Single-use | Iterators are consumed; forgetting this creates silent empty-collection bugs |
| No random access, no `.length` | Element 500 means walking 500 elements; size requires draining |
| Cleanup is subtle | Early `break` needs `return()` / `try...finally`, or you leak file handles and cursors |

**The sweet spot:** Make **every custom collection you write iterable** — one method, and you get the whole language's syntax. Use **generators** wherever you'd otherwise hand-roll a `next()` state machine. Use **async iterators** whenever data arrives in pages, batches, or chunks and the full set might not fit in memory — which in a Node backend is nearly every time you touch a database, an S3 bucket, a file, or a third-party API.

---

## Common interview questions on this topic

### Q1: "What's the difference between an iterable and an iterator?"
**Hint:** An **iterable** *can produce* iterators — it has `[Symbol.iterator]()`. An **iterator** *is* a single stateful walk — it has `next()` returning `{value, done}`. An array is iterable, and every `for...of` over it gets a fresh iterator starting at index 0. Iterators are usually *also* iterable (they return `this`), which is why you can `for...of` a generator object. The classic bug: making the collection itself the iterator, so the second loop over it yields nothing.

### Q2: "Implement an in-order iterator for a binary tree."
**Hint:** Two acceptable answers. **Generator (say this first):** `*inOrder(n) { if (!n) return; yield* this.inOrder(n.left); yield n.value; yield* this.inOrder(n.right); }` — four lines. **By hand (they'll ask next):** you can't use recursion, because an iterator must *pause* between elements, so you keep an explicit stack: push all left children, pop, emit, then move to the popped node's right child. Both are O(n) total, O(h) space.

### Q3: "How would you process a 10 GB CSV / a 4-million-row table in Node without running out of memory?"
**Hint:** Async iterator. `for await (const line of readline.createInterface({ input: fs.createReadStream(path) }))` for the file; a batched cursor generator for the DB. The sentence to say out loud: *"memory stays constant regardless of dataset size, because I only ever hold one chunk."* Then mention backpressure — pull-based consumption means the source can't outrun you. Bonus points: use a keyset cursor (`WHERE id > lastId`) rather than `OFFSET`, because large offsets force the DB to scan and discard every skipped row.

### Q4: "What's the difference between an async iterator and an Observable / EventEmitter?"
**Hint:** **Pull vs push.** `for await...of` is pull — the consumer requests the next value and sets the pace, so backpressure is automatic. `emitter.on('data')` is push — the producer fires whenever it likes and will happily flood a slow consumer, so backpressure becomes your problem. Also: an async iterator is one linear sequence; an EventEmitter can have many listeners. Iterator ↔ Observer is the pull/push pair.

### Q5: "What is `yield*` and when do you use it?"
**Hint:** It **delegates** to another iterable — yields every one of its values in turn, then continues. Two big uses: (1) **recursion in generators** — `yield* walk(node.left)` is how you traverse trees; (2) **composition** — `*[Symbol.iterator]() { yield* this.#items; }` forwards an inner collection's iteration outward in one line.

---

## Practice exercise

### Build a Lazy `EventLog` — Sync, Generator, and Async

Write `eventlog.js`, runnable with `node eventlog.js`. Build it in four stages; **each stage must not change the consumer code from the previous stage.** That constraint is the whole lesson.

**Stage 1 — A hand-written iterator.** Make an `EventLog` class with a private `#events` array (each event: `{ id, type, userId, at }`) and an `add(event)` method. Implement `[Symbol.iterator]()` **by hand** — return an object with a `next()` closing over an index. Prove all of these work: `for...of`, `[...log]`, `const [first, second] = log`, `Array.from(log)`, `new Set(log)`. Then loop over the same log **twice** and confirm you get every event both times. (Nothing the second time? You made the class its own iterator. Fix it — that's the classic bug.)

**Stage 2 — Generators and lazy pipelines.** Rewrite `[Symbol.iterator]` as `*[Symbol.iterator]() { yield* this.#events; }`. Add standalone generators `filter(iterable, pred)`, `map(iterable, fn)`, `take(iterable, n)`. Compose them to get the **first 3 `'login'` events, formatted as strings**. Now prove laziness: put a `console.log` inside `filter` and confirm it runs only for as many events as `take` actually pulled — **not** for the whole log.

**Stage 3 — Custom traversals.** Add `*byUser(userId)` (yields only that user's events) and `*reversed()` (walks backwards). Two new traversal orders, zero changes to any consumer.

**Stage 4 — The async one (the important one).** Write `class RemoteEventLog` with an `async *[Symbol.asyncIterator]()` simulating a paginated API: 47 fake events served in pages of 10, `await new Promise(r => setTimeout(r, 50))` per page to fake latency, and a `console.log('FETCHING PAGE n')` so you can *see* when pages load. Then (a) `for await` over all 47 and count them; (b) `break` after the 12th event and confirm from the logs that **only 2 pages were ever fetched**, not 5.

**Produce:** the file and its console output. **The success criterion for stage 4:** the "FETCHING PAGE" lines appear *interleaved* with your processing output, not all at the start. If they all print up front, you're eager, not lazy — and the whole point has been lost.

---

## Quick reference cheat sheet

- **One line:** Iterator = walk a collection one element at a time without knowing what's inside it.
- **Iterable protocol:** an object with `[Symbol.iterator]()` that returns an iterator.
- **Iterator protocol:** an object with `next()` returning `{ value, done }`. That's the whole contract.
- **`for...of` is sugar** for `const it = obj[Symbol.iterator](); while (!(r = it.next()).done) { ... }`.
- **One method, eight features:** `[Symbol.iterator]` gives you `for...of`, spread, destructuring, `Array.from`, `Set`/`Map` construction, `Promise.all`, argument spreading, and `yield*`.
- **Iterable ≠ Iterator.** Iterables make *fresh* iterators; iterators are *single-use*. Conflating them is the #1 bug.
- **Generators (`function*`, `yield`) are the ergonomic iterator** — they pause and resume, so the engine keeps your traversal state for you. **`yield*` delegates**, and is how you write recursive tree traversal in four lines.
- **Generators are lazy** — nothing runs until you pull, so infinite sequences become possible.
- **Async iterators:** `[Symbol.asyncIterator]()`, `next()` returns `Promise<{value, done}>`, consumed with **`for await...of`**.
- **The killer Node use case:** stream DB rows, API pages, S3 keys, or file lines in **constant memory**, no matter how big the dataset.
- **Node streams, MongoDB cursors, and AWS SDK paginators are already async iterables** — you have the interface for free.
- **Pull, not push** — the consumer sets the pace, so backpressure is automatic (unlike EventEmitter).
- **Clean up on early exit:** `break` triggers the iterator's `return()` (or a generator's `finally`) — close file handles and cursors there.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [43 — Command Pattern](./43-pattern-command.md) — requests as objects; a command queue is often drained with an iterator |
| **Next** | [45 — Template Method Pattern](./45-pattern-template-method.md) — a base class defines the algorithm's skeleton, subclasses fill in the steps |
| **Related** | [38 — Composite Pattern](./38-pattern-composite.md) — builds the trees that `yield*` recursion walks |
| **Related** | [41 — Observer Pattern](./41-pattern-observer.md) — the push counterpart to Iterator's pull; `stream.on('data')` vs `for await...of` |
| **Related** | [51 — Producer-Consumer Pattern](./51-pattern-producer-consumer.md) — an async iterator with a buffer is a channel with built-in backpressure |
