# 28 — What Are Design Patterns and Why They Exist
## Category: LLD Patterns

---

## What is this?

A **design pattern** is a *named, reusable solution to a problem that keeps showing up* in software design. It is not a library you install, and it is not code you copy-paste. It is a **shape you learn to recognize** — like how a chess player recognizes "that's a fork" or a doctor recognizes "that's a stress fracture."

The real-world analogy: think of **cooking techniques**. "Braise," "sauté," "emulsify," "deglaze." None of these are recipes. They're named techniques that solve recurring problems (tough meat, water in the pan, oil and vinegar that won't mix). A chef doesn't invent "slowly cooking meat in liquid in a covered pot" from scratch each time — she says "I'll braise it" and every other chef in the kitchen instantly knows what will happen. Design patterns are the braise-and-sauté of object-oriented design.

---

## Why does it matter?

**If you don't know patterns, three things happen:**

1. **You reinvent them badly.** Every experienced engineer eventually writes an object that lets other objects subscribe to its changes. If you've never heard of "Observer," you'll invent a half-broken version of it, with a memory leak because you forgot to let subscribers unsubscribe.
2. **You can't talk about design.** Design review becomes 20 minutes of "so this class has a list of these other classes and when the first one changes it loops over them and calls a function they gave it earlier..." instead of 4 seconds of "it's an Observer."
3. **You can't read other people's code.** Half the frameworks you use — Express, Node streams, Redux, Passport — are patterns wearing a costume. If you know the pattern, the framework becomes obvious. If you don't, it's magic.

**The interview angle:** LLD/machine-coding rounds are *explicitly* pattern rounds. "Design a parking lot" is really "do you know Strategy and Factory?" "Design an ATM" is really "do you know State?" Interviewers listen for you to *name* the pattern. Naming it is worth points. Naming it *and* justifying why the alternative was worse is worth more.

**The real-work angle:** Patterns are how you make a codebase survive people leaving. A new hire who knows patterns can open a file, see `PaymentStrategy`, and understand it in 30 seconds. That's the entire value proposition.

---

## The core idea — explained simply

### The Building-Architect Analogy (this is literally where patterns came from)

In 1977, an architect named **Christopher Alexander** published *A Pattern Language*. He wasn't writing about software — he was writing about **buildings and towns**. He had noticed something: good buildings, all over the world, across centuries, kept solving the same human problems in the same ways.

For example, he described a pattern he called **"Light on Two Sides of Every Room"**:

> **Problem:** Rooms lit from one side only feel gloomy, and people's faces are hard to read because they're backlit and in shadow. People instinctively avoid these rooms.
>
> **Solution:** Give every room windows on at least two walls.
>
> **Consequence:** More exterior wall per room, so the building has a less compact footprint and costs more.

Look at the shape of that. A **name**. A **problem**. A **context** where the problem arises. A **solution**. And, crucially, a **cost** — the pattern doesn't pretend it's free.

Alexander wrote 253 of these. Nobody in software noticed for about 15 years.

Then in **1994**, four engineers — **Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides**, forever after known as the **"Gang of Four" (GoF)** — published *Design Patterns: Elements of Reusable Object-Oriented Software*. They did to software what Alexander did to architecture: they went looking through real, working C++ and Smalltalk codebases, found the structures that kept reappearing, and **gave them names**. They found 23.

That book is why, today, you can say "just put a Strategy there" in a code review and four people nod.

### Mapping the analogy back

| Alexander's building patterns | GoF software patterns |
|-------------------------------|------------------------|
| "Light on Two Sides of Every Room" | "Observer" |
| Problem: gloomy, unusable rooms | Problem: object A must react to changes in B without B depending on A |
| Context: any room people sit in | Context: event-driven code, UI, pub-sub |
| Solution: windows on two walls | Solution: B keeps a subscriber list; A registers a callback |
| Cost: more exterior wall, higher build cost | Cost: indirection; hard to trace flow; leaks if you forget to unsubscribe |
| **Not a blueprint** — you still design your own house | **Not code** — you still write your own classes |

That last row is the one people miss. A pattern is **not a snippet**. You don't `npm install observer`. The pattern is the *idea*; your code is a fresh instance of the idea, shaped to your problem.

### The one-sentence definition to memorize

> **A design pattern is a named, reusable solution to a recurring design problem within a particular context — described together with its consequences.**

Every word is doing work:
- **Named** — so a team can share it in conversation.
- **Reusable** — it's a shape, not a specific implementation.
- **Recurring** — if a problem happens once, it doesn't need a pattern. It needs a fix.
- **Within a context** — Singleton is right for a DB pool and wrong for a User. Context decides.
- **Consequences** — every pattern *costs* something. Usually indirection. A pattern that claims no downside is a sales pitch, not a pattern.

---

## Key concepts inside this topic

### 1. The three GoF categories

The Gang of Four sorted their 23 patterns by **what kind of problem they solve**:

| Category | The question it answers | Mental hook |
|----------|------------------------|-------------|
| **Creational** | *How do objects get created?* | The factory floor |
| **Structural** | *How do objects get composed into bigger things?* | The Lego instructions |
| **Behavioral** | *How do objects talk to each other and divide responsibility?* | The org chart |

That's it. If you remember nothing else, remember: **create, compose, communicate.**

### 2. All 23 patterns, one line each

**Creational — how objects get created (5)**

| Pattern | Category | Solves |
|---------|----------|--------|
| **Singleton** | Creational | Exactly one instance of a thing must exist, and everyone must share it (a DB connection pool, a config object). |
| **Factory Method** | Creational | Let a subclass decide *which* concrete class to instantiate, so callers never `new` a concrete type. |
| **Abstract Factory** | Creational | Create a *family* of related objects that must match each other (all-AWS clients, all-dark-theme widgets). |
| **Builder** | Creational | Assemble a complex object step by step, so you don't get a constructor with 15 arguments. |
| **Prototype** | Creational | Make new objects by *cloning* an existing one, when construction is expensive but copying is cheap. |

**Structural — how objects compose (7)**

| Pattern | Category | Solves |
|---------|----------|--------|
| **Adapter** | Structural | Make an existing class fit an interface it wasn't built for (a translator between two APIs). |
| **Bridge** | Structural | Split an abstraction from its implementation so both can vary without a subclass explosion. |
| **Composite** | Structural | Treat a single object and a *tree of objects* through the same interface (files and folders). |
| **Decorator** | Structural | Wrap an object to add behavior at runtime, without touching or subclassing the original. |
| **Facade** | Structural | Put one simple door in front of a complicated subsystem. |
| **Flyweight** | Structural | Share the identical parts of a million objects so you don't run out of memory. |
| **Proxy** | Structural | Stand in front of an object to control access to it (lazy-load, cache, authorize, log). |

**Behavioral — how objects communicate (11)**

| Pattern | Category | Solves |
|---------|----------|--------|
| **Chain of Responsibility** | Behavioral | Pass a request down a line of handlers until someone deals with it (middleware). |
| **Command** | Behavioral | Turn "do this action" into an *object*, so it can be queued, logged, retried, or undone. |
| **Interpreter** | Behavioral | Represent a small language's grammar as classes and evaluate expressions in it. |
| **Iterator** | Behavioral | Walk a collection's elements one at a time without exposing how the collection is stored. |
| **Mediator** | Behavioral | Stop N objects from all knowing each other; route everything through one hub. |
| **Memento** | Behavioral | Capture an object's internal state so it can be restored later (undo). |
| **Observer** | Behavioral | When one object changes, automatically notify everyone who signed up to hear about it. |
| **State** | Behavioral | Let an object change its behavior when its internal state changes — kill the giant `switch`. |
| **Strategy** | Behavioral | Make an algorithm swappable at runtime by putting each variant behind a common interface. |
| **Template Method** | Behavioral | Fix the *skeleton* of an algorithm in a base class; let subclasses fill in specific steps. |
| **Visitor** | Behavioral | Add a new operation across a class hierarchy without editing any of those classes. |

**Reality check on which ones matter.** In 2026 JavaScript work, you will use **Strategy, Observer, Factory, Adapter, Decorator, Facade, Singleton, Command, Chain of Responsibility, State, Builder, Iterator, Proxy** constantly. You will almost never write **Interpreter** or **Flyweight** by hand, and **Visitor** shows up mainly if you touch ASTs (Babel, ESLint). Learn all 23 names; go deep on the first list.

### 3. Patterns you already use in Node without knowing it

This is the section that makes patterns click. You have been writing these for years.

**`EventEmitter` = Observer**

```javascript
import { EventEmitter } from 'node:events';

const orders = new EventEmitter();

// Two independent subscribers. The emitter has NO IDEA who they are —
// that decoupling IS the Observer pattern.
orders.on('order:placed', (order) => sendConfirmationEmail(order));
orders.on('order:placed', (order) => decrementInventory(order));

orders.emit('order:placed', { id: 'ord_1', total: 4999 });
```
The emitter holds a subscriber list, and `emit` loops over it. That is Observer, verbatim from the 1994 book.

**Express middleware = Chain of Responsibility**

```javascript
app.use(requestLogger);       // handler 1 — logs, then calls next()
app.use(authenticate);        // handler 2 — may STOP the chain with a 401
app.use(rateLimit);           // handler 3
app.post('/orders', createOrder);

function authenticate(req, res, next) {
  if (!req.headers.authorization) {
    return res.status(401).end();   // handled here — chain stops
  }
  req.user = decode(req.headers.authorization);
  next();                            // not my problem — pass it down the chain
}
```
Each handler either handles the request or passes it on. `next()` *is* the chain link.

**Streams `.pipe()` = Decorator**

```javascript
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';
import { createCipheriv } from 'node:crypto';

createReadStream('backup.sql')
  .pipe(createGzip())                      // wraps the stream, adds compression
  .pipe(createCipheriv('aes-256-ctr', key, iv))  // wraps again, adds encryption
  .pipe(createWriteStream('backup.sql.gz.enc'));
```
Each `.pipe()` wraps the previous stream in a new object with the *same* streaming interface but *extra* behavior. Wrapping to add behavior while keeping the interface — that is Decorator.

**`crypto.createHash()` = Factory**

```javascript
import { createHash } from 'node:crypto';

const h = createHash('sha256');  // you never `new Sha256Hash()`
h.update('hello');
console.log(h.digest('hex'));
```
You pass a *string*, and a function hands you back a fully built object of some concrete class you never named. That's a factory.

**`Symbol.iterator` = Iterator**

```javascript
class Playlist {
  #tracks = [];
  add(t) { this.#tracks.push(t); return this; }

  // Callers walk the playlist WITHOUT knowing it's backed by an array.
  // Swap #tracks for a linked list tomorrow and no caller changes.
  *[Symbol.iterator]() {
    for (const t of this.#tracks) yield t;
  }
}

const p = new Playlist().add('Kashmir').add('Black Dog');
for (const track of p) console.log(track);   // works, because Iterator
```

**`module.exports = new X()` = Singleton**

```javascript
// db.js — Node's module cache makes this a Singleton for free.
class Database {
  constructor() { this.pool = createPool({ max: 20 }); }
}
export default new Database();   // every importer gets the SAME instance
```
Node caches modules, so this object is created once per process. You've been writing Singletons since your first `logger.js`.

**The point:** patterns are not exotic. They're the names for things good code already does.

### 4. The decision guide — symptom → pattern

Don't start from "which pattern should I use?" Start from **"what hurts?"**

| Symptom in your code | Reach for |
|----------------------|-----------|
| A big `if/else` or `switch` picking between *algorithms* (shipping cost, sort order, compression) | **Strategy** |
| A big `if/else` or `switch` on a *status field*, and each branch also changes the status | **State** |
| Long `switch` deciding *which class to construct* from a string/enum | **Factory Method** |
| You must construct a *matched set* of objects (all-AWS, all-dark-theme) and mixing them breaks things | **Abstract Factory** |
| A constructor with 8+ params, half optional, and callers pass `null, null, true, null` | **Builder** |
| Object X must react when Y changes, but Y must not know X exists | **Observer** |
| A third-party SDK's method names don't match the interface your code expects | **Adapter** |
| You want optional add-ons (retry, cache, log, gzip) stackable in any order | **Decorator** |
| Calling a subsystem takes 6 setup calls in the right order, and everyone copy-pastes them | **Facade** |
| Requests must pass a sequence of steps, any of which may stop it (auth, validate, rate-limit) | **Chain of Responsibility** |
| You need undo/redo, a job queue, or an audit log of actions | **Command** (+ **Memento** for undo state) |
| You need to control *access* to an object — lazy-load it, cache it, check permissions first | **Proxy** |
| You have a tree (folders, org chart, UI nodes) and want to treat leaf and branch identically | **Composite** |
| N objects all reference each other and the dependency graph is a hairball | **Mediator** |
| Two independent dimensions of variation (Shape × Renderer) causing `CircleSvgRenderer`, `SquareCanvasRenderer`… | **Bridge** |
| Millions of near-identical objects blowing up memory | **Flyweight** |
| Base algorithm is fixed but two steps differ per subclass | **Template Method** |
| Expensive-to-build object, and you need many near-copies of it | **Prototype** |

Print that table. It's most of what an LLD interview tests.

### 5. The dangers — patternitis and its cousins

Patterns have a dark side, and it's worse than not knowing them.

**(a) Patternitis — treating the book as a checklist.** The moment you learn 23 patterns, every problem starts looking like it needs one. Symptom: a PR that adds four interfaces and three factories to make a function that returns a boolean.

**(b) Cargo-culting — copying the shape without the reason.** You saw a `StrategyFactory` at your last job so you add one here, even though there is exactly one strategy and there will only ever be one.

```javascript
// BAD — a Strategy with one strategy. This is a function wearing a costume.
class TaxStrategy { calculate(amount) { throw new Error('not implemented'); } }
class IndiaTaxStrategy extends TaxStrategy {
  calculate(amount) { return amount * 0.18; }
}
class TaxCalculator {
  constructor(strategy) { this.strategy = strategy; }
  calc(a) { return this.strategy.calculate(a); }
}
new TaxCalculator(new IndiaTaxStrategy()).calc(100);

// GOOD — until a second tax regime actually exists:
const calculateTax = (amount) => amount * 0.18;
calculateTax(100);
```
Add the Strategy on the day the **second** algorithm arrives. Not before. That's not laziness — that's [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) doing its job.

**(c) Forcing a Singleton where a plain module works.** This is the single most common JS pattern crime.

```javascript
// BAD — a Java-flavoured Singleton, transplanted into JS where it buys nothing.
class Config {
  static #instance = null;
  static getInstance() {
    if (!Config.#instance) Config.#instance = new Config();
    return Config.#instance;
  }
}
Config.getInstance().get('PORT');

// GOOD — the ES module cache already guarantees one instance per process.
// config.js
export const config = Object.freeze({ port: Number(process.env.PORT ?? 3000) });
```
Java needs `getInstance()` because it has no module-level singletons. **Node does.** Importing the same module twice gives you the same object. Writing `getInstance()` in Node is ceremony with no payoff — and it makes the thing harder to mock in tests.

**(d) AbstractFactoryFactoryBuilder syndrome.** Stacking patterns until the abstraction is deeper than the problem. If a reader has to open five files to find the line that does the work, the patterns have failed. **Indirection is the price you pay for flexibility. If you're not buying flexibility, don't pay.**

**The rule that keeps you safe:**

> Patterns are a vocabulary for **recognizing structure you already need** — not a checklist to apply. You should discover the pattern in your problem, then name it. You should not pick a pattern and then hunt for a place to put it.

### 6. What a pattern description actually contains

When you read a pattern (in this course, in the GoF book, on refactoring.guru), look for these six fields. When you *design* one, you should be able to fill them in:

| Field | Example (Strategy) |
|-------|-------------------|
| **Name** | Strategy |
| **Intent** | Define a family of interchangeable algorithms behind one interface. |
| **Problem / motivation** | A class has a growing `switch` over algorithm variants; adding a variant edits the class. |
| **Participants** | `Context` (holds a strategy), `Strategy` (interface), `ConcreteStrategyA/B` |
| **Consequences** | + open for extension; + testable in isolation. − one more class per algorithm; − the context must be handed a strategy from somewhere. |
| **Related patterns** | State (same shape, different intent — State transitions itself); Template Method (same goal via inheritance, not composition). |

If you can rattle off Intent + Participants + Consequences + one Related pattern for a given pattern, you will pass the pattern portion of any LLD interview. That's the study target.

---

## Visual / Diagram description

### Diagram 1: The pattern map — create, compose, communicate

```
                    ┌───────────────────────────────────────┐
                    │      23 GoF DESIGN PATTERNS           │
                    └────────────────┬──────────────────────┘
             ┌───────────────────────┼───────────────────────┐
             │                       │                       │
             ▼                       ▼                       ▼
   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐
   │   CREATIONAL     │   │   STRUCTURAL     │   │    BEHAVIORAL        │
   │  "how objects    │   │  "how objects    │   │  "how objects        │
   │   get created"   │   │   compose"       │   │   communicate"       │
   ├──────────────────┤   ├──────────────────┤   ├──────────────────────┤
   │ Singleton        │   │ Adapter          │   │ Chain of Responsib.  │
   │ Factory Method   │   │ Bridge           │   │ Command              │
   │ Abstract Factory │   │ Composite        │   │ Interpreter          │
   │ Builder          │   │ Decorator        │   │ Iterator             │
   │ Prototype        │   │ Facade           │   │ Mediator             │
   │                  │   │ Flyweight        │   │ Memento              │
   │      (5)         │   │ Proxy            │   │ Observer             │
   │                  │   │                  │   │ State                │
   │                  │   │      (7)         │   │ Strategy             │
   │                  │   │                  │   │ Template Method      │
   │                  │   │                  │   │ Visitor       (11)   │
   └──────────────────┘   └──────────────────┘   └──────────────────────┘
        hides `new`          wraps / nests           routes messages
```

**What it shows:** the whole catalogue on one whiteboard. In an interview, if asked "what are the categories of design patterns," draw exactly this. The bottom line of each box is the *tell*: Creational patterns exist to hide the `new` keyword; Structural patterns wrap objects in other objects; Behavioral patterns decide who calls whom.

### Diagram 2: The lifecycle of a pattern in real code (how it should arrive)

```
  DAY 1                DAY 40                    DAY 41                   DAY 90
┌─────────┐         ┌────────────┐          ┌───────────────┐       ┌───────────────┐
│ One      │         │ Second     │          │  You NAME     │       │ Third variant │
│ algorithm│  ────▶  │ variant    │  ────▶   │  the shape:   │ ────▶ │ = 1 new file, │
│          │         │ arrives.   │          │  "Strategy"   │       │ 0 edits to    │
│ Just a   │         │ You add an │          │  Extract the  │       │ existing code │
│ function │         │ if/else.   │          │  interface.   │       │               │
└─────────┘         └────────────┘          └───────────────┘       └───────────────┘
     ▲                     ▲                        ▲                       ▲
  YAGNI —            STILL FINE.              THE RIGHT              The payoff:
  no pattern         Two branches             MOMENT. The            Open/Closed
  yet.               is not yet               problem has            (see topic 15).
                     a problem.               proven it recurs.

   ┌──────────────────────────────────────────────────────────────────────────┐
   │  PATTERNITIS = jumping straight from DAY 1 to DAY 41. You pay all the    │
   │  indirection cost on day one and collect the benefit... possibly never.  │
   └──────────────────────────────────────────────────────────────────────────┘
```

**What it shows:** patterns are *discovered*, not *scheduled*. The correct trigger for introducing a pattern is the arrival of the **second or third** variation, not the anticipation of it. Draw this for yourself the next time you feel the urge to add an interface "because we might need it."

---

## Real world examples

### 1. Express.js — Chain of Responsibility, in production, at internet scale

Express's entire architecture is one pattern. The `app` holds an ordered array of handler functions; a request is passed to the first one, and each handler decides whether to end the response or call `next()` to pass it along. Error-handling middleware (the 4-argument `(err, req, res, next)` form) is a *second* chain the request falls into when someone calls `next(err)`.

This is why you can `npm install helmet cors morgan compression` from four different authors and stack them in any order: they all conform to the chain's single interface, `(req, res, next)`. The pattern *is* the plugin ecosystem.

### 2. Node.js core streams — Decorator, plus Iterator

`zlib.createGzip()`, `crypto.createCipheriv()`, and `stream.Transform` all take a stream and return a stream. Same interface in, same interface out, extra behavior in the middle — textbook Decorator. Because the interface is preserved, `.pipe()` chains are arbitrarily long and order-independent in exactly the way Decorator promises.

And since Node 10, streams are also async-iterable (`for await (const chunk of stream)`) — that's `Symbol.asyncIterator`, the Iterator pattern, letting you walk a stream without knowing anything about its internal buffering.

### 3. Babel and ESLint — Visitor, the pattern nobody writes by hand

Both tools parse your JavaScript into an **AST** (Abstract Syntax Tree — a tree of node objects like `FunctionDeclaration`, `CallExpression`). A plugin doesn't traverse the tree itself. Instead it hands the traverser an object of *visitor* methods:

```javascript
// A Babel plugin. You never walk the tree — you declare what to do at each node type.
export default function myPlugin() {
  return {
    visitor: {
      CallExpression(path) { /* called for every function call in the file */ },
      Identifier(path)     { /* called for every identifier */ },
    },
  };
}
```

That's the Visitor pattern: the tree knows how to walk itself, and *you* supply the new operation without ever modifying the node classes. It's how thousands of third-party Babel/ESLint plugins add new behavior to a class hierarchy they don't own.

---

## Trade-offs

| Using design patterns | Benefit | Cost |
|-----------------------|---------|------|
| **Shared vocabulary** | "Put a Strategy there" replaces 3 minutes of explanation. Onboarding gets faster. | Only works if the whole team knows the vocabulary. Half a team knowing patterns is worse than none. |
| **Battle-tested** | You inherit 30 years of other people's bugs-already-found. Edge cases are documented. | The GoF book was written for C++/Smalltalk in 1994. Some patterns (Iterator, Command, Prototype) are *language features* in JS and shouldn't be hand-rolled. |
| **Design becomes discussable** | You can argue about a design on a whiteboard before writing code. | Encourages Big Design Up Front if you're not careful. |
| **Extensibility** | New variant = new file, no edits to old files (Open/Closed). | Every pattern adds **indirection**. More files, more jumps in the debugger, harder stack traces. |

| Skipping patterns | Benefit | Cost |
|-------------------|---------|------|
| **Just write the function** | Fastest possible path to shipping. Zero indirection. Easiest to read *today*. | The 4th `if/else` branch lands and now the change is expensive and risky. |
| **No interfaces** | Nothing to mock, nothing to wire up. | Testing gets hard — you can't substitute a fake for the real payment gateway. |

**The sweet spot:** Write the simplest thing that works. When the **second** variation of a behavior appears, write the `if/else`. When the **third** appears — or when you find yourself editing a class you shouldn't have to touch — *that* is the moment to extract the pattern. Recognize, then name, then refactor. Never the reverse.

**Rule of thumb:** If you can't state, in one sentence, what would get *harder* if you removed the pattern, you don't need the pattern.

---

## Common interview questions on this topic

### Q1: "What is a design pattern? Is it a library?"
**Hint:** No. It's a *named, reusable solution to a recurring problem in a context*, along with its consequences. It's a description of a shape, not code you install. Trace the lineage: Christopher Alexander's 1977 *A Pattern Language* for buildings → the Gang of Four's 1994 book applied the idea to OO software and catalogued 23. Mention that a proper pattern description always includes the *cost*, not just the benefit — that's what separates a pattern from a slogan.

### Q2: "What are the three categories, and give one example of each?"
**Hint:** **Creational** — how objects get created (Singleton, Factory Method, Abstract Factory, Builder, Prototype). **Structural** — how objects compose into bigger structures (Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight). **Behavioral** — how objects communicate and split responsibility (Strategy, Observer, State, Command, Chain of Responsibility, Iterator, Template Method, Mediator, Memento, Visitor, Interpreter). The memory hook: **create, compose, communicate.** Bonus points: say the counts (5 / 7 / 11 = 23).

### Q3: "Name a design pattern you've used in Node.js — without a framework helping you."
**Hint:** Don't just say "Singleton." Say something with a *reason*. Good answer: "Strategy, for our payment providers — we had a `switch` over Stripe/Razorpay/PayPal in the checkout service; every new provider meant editing checkout. I extracted a `PaymentProvider` interface with `charge(amountInPaise, token)`, and now a provider is one new file, and checkout hasn't changed in a year." Then mention the ones you get for free: `EventEmitter` is Observer, Express middleware is Chain of Responsibility, `.pipe()` is Decorator.

### Q4: "When are design patterns a bad idea?"
**Hint:** When you apply them speculatively. Name the failure modes: **patternitis** (checklist-driven design), **cargo-culting** (copying a shape without its reason), and JS-specific ones — hand-rolling a `getInstance()` Singleton when the ES module cache already gives you one instance per process, or hand-rolling an Iterator when `Symbol.iterator` and generators exist. Close with the rule: a pattern buys flexibility and pays for it in indirection. If you're not buying flexibility you actually need, you're just paying.

### Q5: "Strategy and State have almost the same class diagram. What's the difference?"
**Hint:** The *structure* is identical (a context delegates to a swappable object behind an interface); the **intent** differs. In **Strategy**, the client picks the algorithm and the strategies don't know about each other — swapping is the client's decision, and it's usually set once. In **State**, the *states themselves* decide the next state and tell the context to transition — the object's behavior changes over its own lifetime, driven from inside. One-liner: "Strategy is *how* I do a thing; State is *what I currently am*." Interviewers love this one because it proves you understand that a pattern is defined by its *intent*, not its UML.

---

## Practice exercise

### The Pattern Hunt (~30 min)

You will not write a pattern from scratch. You'll do the more valuable thing: **find the ones already living in your code.**

**Part A — Hunt in a real codebase (15 min).**
Open any Node project you've worked on (or clone a small Express app). Find and write down **three concrete examples** of patterns already present. For each, produce exactly four lines:

```
Pattern:      Observer
File/line:    src/services/orderService.js:42
Evidence:     emitter.on('order:paid', ...) with 3 independent subscribers
Why it's OK:  the order service must not know that email + inventory + analytics exist
```

Look specifically for: `EventEmitter` (Observer), `app.use()` (Chain of Responsibility), `.pipe()` (Decorator), any `create*()` function returning an object (Factory), any `module.exports = new Thing()` (Singleton), any `for...of` over your own class (Iterator).

**Part B — Diagnose a smell (15 min).**
Find **one** `switch` statement or `if/else` chain with 3+ branches in that codebase. Then write a short paragraph answering:

1. Which pattern from the symptom table would remove it — **Strategy**, **State**, or **Factory Method**? (Careful: the answer depends on whether the branches select an *algorithm*, a *lifecycle state*, or a *class to construct*.)
2. Sketch the interface — just the method signature, e.g. `calculate(order) -> number`.
3. **Argue the other side.** In two sentences, make the case that the `switch` should be left exactly as it is. If that case is convincing, you've just learned the most important lesson in this document.

**What to produce:** one text file with three 4-line pattern findings and one diagnosis paragraph. Keep it. When you finish topics 29–52, come back and see how many more patterns you can now spot in the same file.

---

## Quick reference cheat sheet

- **Definition:** a design pattern is a *named, reusable solution to a recurring problem in a context* — described with its consequences. Not a library. Not a snippet.
- **Origin:** Christopher Alexander, *A Pattern Language* (1977, buildings, 253 patterns) → the **Gang of Four** (Gamma, Helm, Johnson, Vlissides), *Design Patterns* (1994, software, **23** patterns).
- **The three categories:** **Creational** (5) = how objects get created · **Structural** (7) = how objects compose · **Behavioral** (11) = how objects communicate. Memory hook: **create, compose, communicate.**
- **The three reasons patterns matter:** (a) shared **vocabulary**, (b) they're **battle-tested**, (c) they make design **discussable** before code exists.
- **Every pattern has a cost**, and the cost is almost always **indirection**. A pattern with no stated downside is a sales pitch.
- **You already use them:** `EventEmitter` = Observer · Express `app.use()` = Chain of Responsibility · `.pipe()` = Decorator · `crypto.createHash()` = Factory · `Symbol.iterator` = Iterator · `export default new Db()` = Singleton.
- **Start from the symptom, not the catalogue.** `switch` over algorithms → **Strategy**. `switch` on status → **State**. `switch` picking a class to `new` → **Factory Method**. 8-param constructor → **Builder**. Mismatched third-party API → **Adapter**.
- **Strategy vs State:** identical diagram, different intent. Strategy = *how* I do a thing (client picks). State = *what I currently am* (the state transitions itself).
- **The rule of three:** first variant → just write it. Second → an `if/else` is fine. Third → *now* extract the pattern.
- **Patternitis** = applying patterns as a checklist. **Cargo-culting** = copying the shape without the reason. Both produce codebases where you open five files to find one line of logic.
- **JS-specific trap:** don't hand-roll `getInstance()` — the ES module cache is already a Singleton, and a module is easier to mock.
- **The safe framing:** patterns are a vocabulary for **recognizing structure you already need**, not a checklist to apply. Discover it, then name it — never the reverse.
- **Interview study target per pattern:** Intent + Participants + Consequences + one Related pattern and how it differs.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [27 — Abstraction and Interfaces in Design](./27-abstraction-and-interfaces.md) — patterns are built out of abstractions; you need that skill first |
| **Next** | [29 — Singleton Pattern](./29-pattern-singleton.md) — the first (and most abused) pattern, and where the catalogue begins |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — the GoF's own headline principle; most patterns are composition in disguise |
| **Related** | [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) — the antidote to patternitis; YAGNI is what stops you applying a pattern on day one |
