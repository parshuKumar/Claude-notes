# 24 — Coupling and Cohesion — The Two Forces of Good Design
## Category: LLD Fundamentals

---

## What is this?

**Coupling** is how much your modules depend on each other. **Cohesion** is how focused a single module is on one job. You want **low coupling** (modules can change independently) and **high cohesion** (each module does one thing well).

Think of a good kitchen. Cohesion is *"everything in the knife drawer is a knife"* — you open one drawer and find exactly the class of thing you expected. Coupling is *"can I replace the fridge without rewiring the oven?"* In a well-designed kitchen, yes. In a badly designed one, the fridge, the oven, and the dishwasher share a single tangled circuit, and touching one trips all three.

These two forces have been the definition of "good design" since 1974, and they have not aged a day.

---

## Why does it matter?

Coupling and cohesion are the *measurable* version of the vague word "clean". When a senior engineer says your code is messy, they almost always mean one of these two things, and the fix is different for each:

- **High coupling** → you change one file and three unrelated tests break. You can't unit-test anything without spinning up a database. A new engineer can't understand any file without reading five others. Refactoring is terrifying because you can't predict the blast radius.
- **Low cohesion** → you open `UserService.js` and it is 2,000 lines that send emails, resize avatars, generate PDF invoices, and talk to Stripe. Nobody can find anything. Every feature touches this file, so every PR conflicts with every other PR.

**In interviews:** this is the vocabulary you use to *justify* your design. When the interviewer asks "why did you split that into two classes?", the answer isn't "it felt cleaner" — it's *"because those two responsibilities change for different reasons, and keeping them together gives me low cohesion and forces every consumer to depend on both."* Naming the force is what separates a mid-level answer from a senior one.

**At work:** it is the single best predictor of whether a codebase will still be pleasant in two years. Every architectural pattern you will ever learn — microservices, hexagonal architecture, event-driven design, dependency injection — is, underneath, a specific technique for lowering coupling while keeping cohesion high.

---

## The core idea — explained simply

### The Office Departments Analogy

Imagine a 200-person company. There are two ways to organise it.

**Company A — high coupling, low cohesion.**
There are no real departments. There's "Team 1" and "Team 2", and the work was split by who was free on Tuesday. Team 1 does *some* accounting, *some* recruiting, and the office plants. Team 2 does *the other half* of accounting, customer support, and the Christmas party. To close the books, Team 1 has to walk over and ask Team 2 for numbers, then Team 2 has to ask Team 1 to re-approve them, three times. Nobody can go on holiday. Every decision requires a meeting with both teams.

- **Low cohesion:** each team's work has nothing in common with itself. "Accounting and office plants" is not a job.
- **High coupling:** neither team can complete anything without constantly reaching into the other.

**Company B — low coupling, high cohesion.**
There's a Finance department, a Recruiting department, and a Support department. Finance does finance. All of it. Only it. When Support needs a refund issued, they don't walk into Finance's spreadsheets — they file a **refund request** through a defined process, and Finance handles it. Support doesn't know or care *how* Finance does it. Finance could switch from Excel to SAP tomorrow and Support wouldn't notice.

- **High cohesion:** everything in Finance is finance.
- **Low coupling:** departments talk through a small, defined interface (the request form), not by reaching into each other's internals.

### Mapping it back

| Office | Code | The force |
|---|---|---|
| "Accounting + office plants" in one team | `UserService` that also sends email and resizes images | **Low cohesion** (bad) |
| "Finance does all finance, only finance" | `InvoiceService` that only computes and stores invoices | **High cohesion** (good) |
| Support reads Finance's private spreadsheet | Module A reaches into `moduleB.internalCache._items` | **Content coupling** (worst) |
| Support files a refund request form | `financeService.requestRefund({ orderId, amountCents })` | **Data coupling** (best) |
| Everyone reads one shared whiteboard of company state | Every module reads/writes a global `config` / `window.state` | **Common coupling** (bad) |
| Support tells Finance *how* to do its job, step by step | `finance.process(order, { skipTax: true, mode: 'legacy' })` | **Control coupling** (bad) |

**The key insight that ties them together:** cohesion is about **what's inside one box**. Coupling is about **the wires between boxes**. And they pull in the same direction — when you split a low-cohesion module into focused pieces, coupling usually *drops*, because each consumer now depends only on the small piece it actually needs.

---

## Key concepts inside this topic

### 1. Coupling — the six levels, worst to best

Coupling is a *spectrum*, not a boolean. Learn the names — interviewers use them, and each one has a distinct fix.

#### (a) Content coupling — WORST
Module A reaches directly into module B's internal state and modifies it. B has no idea it happened.

```javascript
// A is mutating B's private guts. If B ever renames `_cache` or changes it to a
// Map, A silently breaks. B's author has no idea A exists.
cacheModule._cache['user:1'] = { name: 'Alice' };   // content coupling
```

**Fix:** make it private (`#cache`) and expose a method: `cacheModule.set('user:1', user)`.

#### (b) Common coupling (global coupling) — very bad
Many modules read and write the same shared global.

```javascript
global.appConfig = { retries: 3 };
// authService.js
global.appConfig.retries = 5;         // silently changes behaviour for EVERYONE
// paymentService.js — now retries 5 times and nobody knows why
```

**Why it's poison:** the dependency is invisible. Nothing in `paymentService`'s signature tells you it depends on `authService`. You cannot test either in isolation, and the bug reproduces only in the exact order the modules happened to load.

**Fix:** pass config in explicitly (`new PaymentService({ retries: 3 })`). That's [18 — Dependency Inversion](./18-solid-dependency-inversion.md).

#### (c) Control coupling — bad
A passes a **flag** that tells B *which branch to take*. The caller is now steering B's internal logic, which means the caller knows B's internals.

```javascript
// BAD: the boolean is a remote control for someone else's if-statement.
report.generate(data, true);           // true = ...what? PDF? summary? draft?
report.generate(data, false, true, 2); // this is where careers go to die
```

**Why it's bad:** every new mode means a new flag *and* a change in every caller. The combinatorics explode: 4 booleans = 16 code paths, of which you tested 3.

**Fix:** split into distinct methods (`generatePdf(data)`, `generateCsv(data)`), or pass a **strategy object** instead of a flag.

#### (d) Stamp coupling — mediocre
B receives a whole fat object but only needs two fields of it.

```javascript
// BAD: sendWelcomeEmail now depends on the ENTIRE User shape. Rename any field
// on User and this function's tests need a full User fixture to even compile.
function sendWelcomeEmail(user) {
  mailer.send(user.email, `Hi ${user.firstName}`);
  // ...but `user` also carries passwordHash, addresses, paymentMethods, 40 fields
}
```

**Why it matters:** you've made `sendWelcomeEmail` depend on a 40-field structure to use 2 of them. That's 38 fields' worth of change that can break it, and a security smell (you just handed a password hash to the mail module).

**Fix:** pass only what's needed.

#### (e) Data coupling — GOOD, the target
B receives exactly the primitive data it needs. Nothing more.

```javascript
// GOOD: depends on two strings. That's the entire contract. Trivially testable.
function sendWelcomeEmail(email, firstName) {
  mailer.send(email, `Hi ${firstName}`);
}
```

#### (f) No coupling / message coupling — the ideal, where it fits
Modules never call each other at all; they communicate through events or messages and don't even know the other exists.

```javascript
// The order module doesn't know email exists. It just announces a fact.
eventBus.emit('order.placed', { orderId: 'o1', email: 'a@b.com' });

// Somewhere else, entirely independently:
eventBus.on('order.placed', ({ email }) => sendWelcomeEmail(email, 'there'));
```

This is the loosest coupling achievable — and it's exactly why HLD uses message queues (Kafka, SQS) between services. **Note the cost:** you trade compile-time safety and traceability for decoupling. You can no longer "find all callers".

**The ladder, one line each:**

```
WORST  ┌─ Content   : A mutates B's private internals        b._cache[k] = v
       │  Common    : A and B share a mutable global         global.config.x = 5
       │  Control   : A passes a flag steering B's branches  b.run(data, true)
       │  Stamp     : A passes a fat object; B uses 2 fields b.email(wholeUser)
       │  Data      : A passes exactly the primitives needed b.email(addr, name)
BEST   └─ Message   : A emits an event; B may or may not hear bus.emit('x', {})
```

### 2. Cohesion — the seven levels, best to worst

Cohesion asks: **why are these things in the same module?**

#### (a) Functional cohesion — BEST
Every element contributes to exactly one well-defined task.

```javascript
// Everything here computes tax. Nothing else. One reason to exist.
class TaxCalculator {
  calculate(amountCents, region) { /* ... */ }
  #rateFor(region) { /* ... */ }
  #applyExemptions(amountCents, region) { /* ... */ }
}
```

#### (b) Sequential cohesion — very good
The output of one function is the input to the next; they form a pipeline.

```javascript
// parse → validate → normalise. Each feeds the next. Legitimately one module.
class CsvImporter {
  parse(raw)          { /* returns rows */ }
  validate(rows)      { /* returns validRows */ }
  normalise(validRows){ /* returns records */ }
}
```

#### (c) Communicational cohesion — good
Functions grouped because they all operate on the **same data**.

```javascript
// All of these read/write the same Order aggregate. Reasonable grouping.
class OrderRepository {
  async findById(id) { /* ... */ }
  async save(order)  { /* ... */ }
  async delete(id)   { /* ... */ }
}
```

#### (d) Procedural cohesion — mediocre
Grouped because they always run in a certain order, but they don't share data.

```javascript
// These run in sequence at startup, but they have nothing to do with each other.
class Bootstrap {
  loadConfig() {}      // touches disk
  connectDb() {}       // touches network
  warmCache() {}       // touches redis
}
```
Tolerable in a `main()` / composition root. A smell anywhere else.

#### (e) Temporal cohesion — weak
Grouped only because they happen **at the same time**.

```javascript
// The ONLY thing these share is "runs on shutdown". Zero conceptual relationship.
function onShutdown() {
  flushLogs();          // logging concern
  closeDbPool();        // persistence concern
  cancelCronJobs();     // scheduling concern
  notifySlack();        // ops concern
}
```

#### (f) Logical cohesion — bad
Grouped because they're "the same *kind* of thing", then selected with a flag. Notice this *creates control coupling* in every caller — the two forces are linked.

```javascript
// BAD: "it's all input handling!" — no. These share a category, not a purpose.
function handleInput(type, payload) {
  if (type === 'keyboard') { /* ... */ }
  else if (type === 'mouse') { /* ... */ }
  else if (type === 'file')  { /* ... */ }   // nothing in common with the others
}
```

#### (g) Coincidental cohesion — WORST
Grouped for no reason at all. You know it by its name.

```javascript
// utils.js — the junk drawer. Every codebase has one. It should not exist.
export function formatDate(d) {}
export function calculateTax(amt) {}
export function isValidEmail(s) {}
export function retryWithBackoff(fn) {}
export function hexToRgb(h) {}
```
`utils.js`, `helpers.js`, `common.js`, `misc.js` — these filenames are *literally the confession of coincidental cohesion*. When every module imports `utils.js`, and `utils.js` changes for five unrelated reasons, you've coupled your entire codebase to a file with no purpose.

```
BEST   ┌─ Functional     : one job                      TaxCalculator
       │  Sequential     : output → input pipeline      parse→validate→normalise
       │  Communicational: same data                    OrderRepository
       │  Procedural     : same order of execution      Bootstrap
       │  Temporal       : same moment in time          onShutdown()
       │  Logical        : same category + a flag       handleInput(type, ...)
WORST  └─ Coincidental   : no reason whatsoever         utils.js
```

### 3. The relationship to SRP (14) and DIP (18)

These aren't three separate ideas. **Cohesion and coupling are the *forces*; SOLID principles are the *techniques* that manage them.**

- **[14 — Single Responsibility Principle](./14-solid-single-responsibility.md)** is *literally the definition of high cohesion*, stated as a rule you can apply. "A class should have one reason to change" = "a class should be functionally cohesive". If you can describe a class without using the word "and", it's cohesive. If `UserService` "manages users **and** sends emails **and** resizes avatars", you have three reasons to change, three responsibilities, and low cohesion.

- **[18 — Dependency Inversion Principle](./18-solid-dependency-inversion.md)** is *the main technique for reducing coupling*. The reason `new PostgresUserRepo()` inside `UserService` is bad is that it couples a high-level policy (user rules) to a low-level detail (Postgres). Inject an abstraction instead, and `UserService` is now coupled only to an *interface* — which never changes — instead of to Postgres, which changes constantly.

The one-liner to remember: **SRP raises cohesion. DIP lowers coupling. Together they are the whole game.**

### 4. Measurable smells — stop guessing, start counting

"Loose coupling" sounds fuzzy. It isn't. Here are questions with **numeric answers** you can run on your own repo today.

| Smell | The question | Bad answer |
|---|---|---|
| **Change amplification** | "To add one field to `User`, how many files must I touch?" | > 3 |
| **Fan-out** | "How many things does this module import?" | > 7–10 for a domain class |
| **Fan-in on a junk file** | "How many files import `utils.js`?" | Anything. Delete `utils.js`. |
| **Test setup cost** | "How many mocks do I need to unit-test this one method?" | > 2–3 |
| **The `and` test** | "Describe this class in one sentence." | You needed the word "and" |
| **Ripple tests** | "I changed module A. How many *unrelated* test files went red?" | > 0 |
| **Boolean params** | "How many boolean/flag parameters does this method take?" | ≥ 2 (control coupling) |
| **Git co-change** | "Which files always appear in the same commit as this one?" | Files in *other* modules |

That last one is the best diagnostic in existence and you can run it right now:

```bash
# Files most often committed alongside src/services/order.js.
# High co-change with files in OTHER modules = hidden coupling you can't see in the imports.
git log --format='%H' --follow -- src/services/order.js \
  | xargs -I{} git show --name-only --format='' {} \
  | sort | uniq -c | sort -rn | head -20
```

If `src/services/order.js` is always committed together with `src/email/templates.js`, those two are coupled — regardless of what the import graph claims.

### 5. Worked refactor: a tightly-coupled, low-cohesion module

Here's the module you've met a hundred times.

```javascript
// ============================================================
// BEFORE — high coupling, low cohesion.
// ============================================================
const { Pool } = require('pg');
const stripe = require('stripe')(process.env.STRIPE_KEY);
const nodemailer = require('nodemailer');

global.appSettings = { taxRate: 0.18, currency: 'INR' };   // (b) COMMON coupling

class OrderService {
  constructor() {
    // (b) DIP violation → coupled to Postgres, Stripe, and SMTP forever.
    // You cannot unit-test a single line of this class without a live database,
    // a Stripe key, and an SMTP server.
    this.db = new Pool({ connectionString: process.env.DATABASE_URL });
    this.mailer = nodemailer.createTransport({ host: 'smtp.example.com' });
  }

  // (c) CONTROL coupling: two booleans = four code paths, and the caller has to
  // know what they mean. `placeOrder(u, i, true, false)` is unreadable.
  async placeOrder(user, items, isExpress, skipEmail) {
    // --- concern 1: validation
    if (!items || items.length === 0) throw new Error('empty order');

    // --- concern 2: pricing + tax (business logic, buried in here)
    let subtotal = 0;
    for (const it of items) subtotal += it.priceCents * it.qty;
    const tax = Math.round(subtotal * global.appSettings.taxRate);
    let shipping = isExpress ? 29900 : 0;
    const total = subtotal + tax + shipping;

    // --- concern 3: payment (talks to Stripe directly)
    const charge = await stripe.charges.create({
      amount: total, currency: global.appSettings.currency,
      source: user.stripeToken,                    // (d) STAMP: needs 1 field of `user`
    });

    // --- concern 4: persistence (raw SQL, right here in the business logic)
    const res = await this.db.query(
      'INSERT INTO orders (user_id, total_cents, charge_id) VALUES ($1,$2,$3) RETURNING id',
      [user.id, total, charge.id],
    );

    // --- concern 5: notification
    if (!skipEmail) {
      await this.mailer.sendMail({
        to: user.email,
        subject: 'Order confirmed',
        html: `<h1>Thanks ${user.firstName}!</h1><p>Total: ₹${total / 100}</p>`,
      });                                          // --- concern 6: HTML templating
    }
    return res.rows[0].id;
  }
}
```

Score it honestly:
- **Cohesion: coincidental-to-procedural.** This class validates, prices, taxes, charges cards, writes SQL, sends SMTP, and renders HTML. Describe it in one sentence — you can't, without six "and"s.
- **Coupling:** common (`global.appSettings`), control (two booleans), stamp (whole `user`), plus hard construction coupling to `pg`, `stripe`, and `nodemailer`.
- **Change amplification:** switch from Stripe to Razorpay → edit this file. Change the tax rate rule → edit this file. Change the email copy → edit this file. Every team edits the same file. Every PR conflicts.
- **Test setup cost:** infinite. You cannot test the tax math without a database.

Now the refactor. Each concern becomes a **functionally cohesive** unit, and they're wired together with **data coupling** and **injected abstractions**.

```javascript
// ============================================================
// AFTER — low coupling, high cohesion.
// ============================================================

// --- 1. PRICING: functionally cohesive. Pure. No I/O. Testable in microseconds.
//        No globals — the tax rate is DATA that comes in, not a global it reads.
class PricingPolicy {
  constructor({ taxRate, expressShippingCents }) {
    this.taxRate = taxRate;
    this.expressShippingCents = expressShippingCents;
  }

  // DATA coupling: primitives in, a plain object out. That's the whole contract.
  priceOrder(items, shippingSpeed) {
    const subtotal = items.reduce((s, it) => s + it.priceCents * it.qty, 0);
    const tax = Math.round(subtotal * this.taxRate);
    const shipping = shippingSpeed === 'express' ? this.expressShippingCents : 0;
    return { subtotal, tax, shipping, total: subtotal + tax + shipping };
  }
}

// --- 2. ABSTRACTIONS (DIP). OrderService will depend on THESE, never on Stripe/pg.
class PaymentGateway {
  async charge({ amountCents, currency, token }) { throw new Error('not implemented'); }
}
class OrderRepository {
  async save(order) { throw new Error('not implemented'); }
}
class Notifier {
  async orderConfirmed({ email, firstName, totalCents }) { throw new Error('not implemented'); }
}

// --- 3. Concrete adapters. Each is the ONLY file that knows a vendor's name.
//        Swapping Stripe for Razorpay now touches exactly ONE file.
class StripeGateway extends PaymentGateway {
  constructor(stripeClient) { super(); this.stripe = stripeClient; }
  async charge({ amountCents, currency, token }) {
    const c = await this.stripe.charges.create({ amount: amountCents, currency, source: token });
    return { chargeId: c.id };
  }
}

class PostgresOrderRepository extends OrderRepository {
  constructor(pool) { super(); this.pool = pool; }
  async save({ userId, totalCents, chargeId }) {
    const res = await this.pool.query(
      'INSERT INTO orders (user_id, total_cents, charge_id) VALUES ($1,$2,$3) RETURNING id',
      [userId, totalCents, chargeId],
    );
    return res.rows[0].id;
  }
}

class EmailNotifier extends Notifier {
  constructor(mailer, templates) { super(); this.mailer = mailer; this.templates = templates; }
  async orderConfirmed({ email, firstName, totalCents }) {
    // Templating lives in ITS own cohesive module, not smeared into order logic.
    await this.mailer.sendMail({
      to: email,
      subject: 'Order confirmed',
      html: this.templates.orderConfirmation({ firstName, totalCents }),
    });
  }
}

// --- 4. The orchestrator. Now functionally cohesive: it does ONE thing —
//        it sequences the steps of placing an order. It knows no vendors.
class OrderService {
  constructor({ pricing, gateway, repository, notifier }) {
    this.pricing = pricing;
    this.gateway = gateway;
    this.repository = repository;
    this.notifier = notifier;
  }

  // No boolean flags. `shippingSpeed` is a value, not a remote control.
  // We take the exact fields we need, not the whole User (kills stamp coupling —
  // and note that passwordHash can no longer accidentally reach the mail server).
  async placeOrder({ userId, email, firstName, paymentToken, items, shippingSpeed = 'standard' }) {
    if (!items?.length) throw new Error('Order must contain at least one item');

    const { total } = this.pricing.priceOrder(items, shippingSpeed);
    const { chargeId } = await this.gateway.charge({
      amountCents: total, currency: 'INR', token: paymentToken,
    });
    const orderId = await this.repository.save({ userId, totalCents: total, chargeId });

    await this.notifier.orderConfirmed({ email, firstName, totalCents: total });
    return orderId;
  }
}

// --- 5. The composition root — the ONE place that knows every concrete class.
//        All the ugly coupling is concentrated here, where it does no damage.
function buildOrderService(deps) {
  return new OrderService({
    pricing:    new PricingPolicy({ taxRate: 0.18, expressShippingCents: 29900 }),
    gateway:    new StripeGateway(deps.stripe),
    repository: new PostgresOrderRepository(deps.pgPool),
    notifier:   new EmailNotifier(deps.mailer, deps.templates),
  });
}
```

**What we actually bought:**

| Metric | Before | After |
|---|---|---|
| Test the tax math | needs Postgres + Stripe + SMTP | `new PricingPolicy({taxRate:0.18}).priceOrder(items,'express')` — pure, 0 mocks |
| Swap Stripe → Razorpay | edit `OrderService` (the core business file) | write one new `RazorpayGateway`, change one line in `buildOrderService` |
| Change the email copy | edit `OrderService` | edit `templates` — `OrderService` never recompiles |
| Boolean flags | 2 (→ 4 untested paths) | 0 |
| "Describe the class" | 6 "and"s | "It sequences the steps of placing an order." |
| Unit-test `placeOrder` | impossible | 4 fake objects, 5 lines, runs in 1ms |

Notice: **suppressing the flags removed control coupling, injecting the abstractions removed construction coupling, passing exact fields removed stamp coupling, and deleting the global removed common coupling** — and doing all four *automatically* raised cohesion, because each concern had to move into its own home. The two forces move together.

---

## Visual / Diagram description

### Diagram 1: The four quadrants

```
                        HIGH COHESION
                              ▲
                              │
   ┌──────────────────────────┼──────────────────────────┐
   │  THE BIG BALL OF MUD-ish │      ★ THE GOAL ★         │
   │                          │                          │
   │  Focused modules, but    │  Focused modules that     │
   │  wired to each other's   │  talk through small,      │
   │  internals. Every change │  stable interfaces.       │
   │  ripples.                │  Change one, others       │
   │                          │  don't notice.            │
   │  "Good classes, awful    │                          │
   │   dependency graph."     │  "Finance does finance,   │
   │                          │   via a request form."    │
◀──┼──────────────────────────┼──────────────────────────┼──▶
HIGH│                          │                     LOW COUPLING
COUPLING                       │                          │
   │      THE DISASTER        │     THE JUNK DRAWER      │
   │                          │                          │
   │  Giant god-classes that  │  Independent modules that │
   │  each do everything AND  │  each do 5 random things. │
   │  reach into each other.  │  Nothing breaks, but      │
   │                          │  nothing is findable.     │
   │  Legacy code. Rewrite    │                          │
   │  candidate.              │  "utils.js, x40"          │
   └──────────────────────────┼──────────────────────────┘
                              │
                              ▼
                        LOW COHESION
```

### Diagram 2: The refactor, drawn

```
BEFORE — one god-class, wired to the world (high coupling, low cohesion)

     global.appSettings ◀─────────────┐  (common coupling — invisible dependency)
                                      │
   ┌──────────────────────────────────┴──────────────────────────────┐
   │                        OrderService                             │
   │  ┌──────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐  │
   │  │ validate │ │ price  │ │  tax   │ │  SQL   │ │ HTML email │  │
   │  └──────────┘ └────────┘ └────────┘ └────────┘ └────────────┘  │
   │        ▲            ▲          ▲         ▲            ▲         │
   │        └────────────┴──────────┴─────────┴────────────┘         │
   │              6 unrelated reasons to change this file            │
   └───────┬──────────────────┬───────────────────┬──────────────────┘
           │ new Pool()       │ require('stripe') │ createTransport()
           ▼                  ▼                   ▼
      ┌─────────┐        ┌─────────┐        ┌─────────┐
      │ Postgres│        │ Stripe  │        │  SMTP   │   ← hard-wired.
      └─────────┘        └─────────┘        └─────────┘     Untestable.


AFTER — focused modules, wired through abstractions (low coupling, high cohesion)

                       ┌────────────────────────┐
                       │      OrderService      │   "sequences an order"
                       │  (one reason to change)│   ← knows NO vendor names
                       └───┬────────┬────────┬──┘
              depends on   │        │        │   depends on ABSTRACTIONS only
             ┌─────────────┘        │        └──────────────┐
             ▼                      ▼                       ▼
    ┌────────────────┐    ┌──────────────────┐    ┌────────────────┐
    │ PaymentGateway │    │ OrderRepository  │    │    Notifier    │  (interfaces)
    │  (interface)   │    │   (interface)    │    │  (interface)   │
    └───────△────────┘    └────────△─────────┘    └───────△────────┘
            │ implements           │ implements           │ implements
    ┌───────┴────────┐    ┌────────┴─────────┐    ┌───────┴────────┐
    │ StripeGateway  │    │ PostgresOrderRepo│    │ EmailNotifier  │  (adapters)
    └───────┬────────┘    └────────┬─────────┘    └───────┬────────┘
            ▼                      ▼                      ▼
        ┌────────┐            ┌─────────┐            ┌────────┐
        │ Stripe │            │ Postgres│            │  SMTP  │
        └────────┘            └─────────┘            └────────┘

    ┌──────────────────────────────────────────────────────────────┐
    │  PricingPolicy — pure, no I/O, no globals. Tested in 1ms.     │
    └──────────────────────────────────────────────────────────────┘

    The arrows now point INWARD at abstractions (that's DIP, doc 18).
    Each box has exactly one reason to change (that's SRP, doc 14).
```

Draw this on a whiteboard and you have simultaneously explained coupling, cohesion, SRP, and DIP. Interviewers love it because it's four answers in one picture.

---

## Real world examples

### 1. Unix pipes — the original high-cohesion, low-coupling design

`grep`, `sort`, `wc`, `cut` — each program does **one thing** (perfect functional cohesion) and knows **nothing** about the others (near-zero coupling). They communicate through the narrowest possible interface: a stream of bytes on stdout.

```bash
cat access.log | grep ' 500 ' | cut -d' ' -f1 | sort | uniq -c | sort -rn | head
```

Six programs cooperate on a real task, and not one of them has ever heard of the other five. You can replace `sort` with a completely different implementation and nothing else changes. This is **data coupling** (pure text in, pure text out) plus **functional cohesion**, and it's why these tools have survived fifty years and outlived the machines they were written for.

### 2. Microservices — coupling and cohesion at the HLD level

The entire microservices argument is this doc, scaled up. A service boundary drawn well is a **cohesion** boundary: the Payments service owns everything about payments and nothing else. The communication rule — *"you may not read another service's database, only call its API or consume its events"* — is a hard ban on **content coupling** and **common coupling**.

And the classic microservices failure mode is exactly this doc's failure mode: the **distributed monolith**, where services were split by *layer* rather than by *capability* (low cohesion), so every user-facing change requires deploying six services in a specific order (high coupling). You now have all the pain of a monolith plus network latency. The lesson: splitting into services does not lower coupling. Splitting along *cohesion* boundaries does.

### 3. React's move from mixins to hooks

Early React had **mixins**, which injected methods and state into a component from the outside. That's close to **common coupling** — two mixins could silently clash over the same state key, and you could not tell where a method came from by reading the component. React's own post-mortem named the problems: implicit dependencies, name collisions, and snowballing complexity.

Hooks replaced them with **explicit data coupling**: `const [x, setX] = useState(0)` — the dependency is a value you named yourself, right there in the function body. Nothing is injected. Nothing can collide. The composition is visible in the code you're reading. Same capability, dramatically lower coupling.

---

## Trade-offs

| Direction | You gain | You give up |
|---|---|---|
| **Lowering coupling** (interfaces, DI, events) | Independent change, easy testing, swappable implementations | More files, more indirection; "where does this actually happen?" becomes a real question; an event-driven flow can be genuinely hard to trace in production |
| **Raising cohesion** (splitting classes) | Findable, understandable, one-reason-to-change modules | More classes; over-split code makes you open six files to follow one flow; can become anaemic (classes with no behaviour, just data) |
| **Tolerating some coupling** | Fewer files, dead-simple call graph, faster to write | Change amplification; untestable units; refactoring becomes scary |
| **Tolerating some low cohesion** | Fast to ship the first version | The god-class arrives around month six and never leaves |

The cost of over-correcting is real and has a name: **abstraction astronaut** code, where a 3-line function is wrapped in a factory, a strategy, an interface, and a DI container. `AbstractSingletonProxyFactoryBean` is not a joke — it exists.

**The sweet spot:**
- **Always** kill content coupling and common coupling. There is no defensible reason for a shared mutable global or reaching into another module's privates. These are free wins.
- **Always** kill boolean-flag parameters (control coupling). Free win.
- **Introduce an interface when there is a second implementation, or when a real thing (DB, network, clock) makes tests slow.** Not before. One implementation behind an interface "just in case" is YAGNI ([19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md)).
- **Rule of thumb:** if you cannot describe a module in one sentence without the word "and", split it. If changing one behaviour makes you touch more than three files, your coupling is too high. Those two sentences will carry you through most design reviews.

---

## Common interview questions on this topic

### Q1: "What's the difference between coupling and cohesion?"
**Hint:** Coupling is **between** modules — how much they depend on each other (want it LOW). Cohesion is **within** one module — how focused it is on a single job (want it HIGH). One-liner: *"cohesion is what's inside the box; coupling is the wires between the boxes."* Then make it concrete: `utils.js` = low cohesion; a class that does `new PostgresPool()` in its constructor = high coupling.

### Q2: "Can you have low coupling and low cohesion at the same time? Is that good?"
**Hint:** Yes, and no. That's the "junk drawer" quadrant — a codebase of forty independent `helpers.js` files that each do five unrelated things. Nothing ripples when you change one, but nothing is findable and logic gets duplicated because nobody knows where anything lives. Low coupling alone is not the goal; you need both. Draw the four-quadrant diagram.

### Q3: "How would you actually measure coupling in a real codebase?"
**Hint:** Give **numbers**, not vibes. (1) *Change amplification*: to add one field, how many files do I touch? (2) *Fan-out*: how many imports does this module have? (3) *Test setup cost*: how many mocks to unit-test one method? (4) The killer — *git co-change analysis*: `git log --name-only` and count which files always appear in the same commit. Files in *different* modules that always change together are coupled, no matter what the import graph says. Mentioning that last one signals real experience.

### Q4: "How does this relate to SOLID?"
**Hint:** They're the same idea at different altitudes. **SRP is the definition of high cohesion** ("one reason to change" = "one job"). **DIP is the main tool for low coupling** (depend on abstractions, so your business logic never knows Postgres exists). ISP lowers coupling too — a fat interface forces clients to depend on methods they don't use. SOLID is a set of *techniques*; coupling and cohesion are the *forces* those techniques exist to manage.

### Q5: "Here's a 500-line `UserService`. Walk me through refactoring it."
**Hint:** Don't start cutting. Start by **listing the reasons it changes**: "the marketing team changes email copy; the infra team changes the database; the finance team changes tax rules." Each reason is a responsibility, and each responsibility becomes a module. Then wire them with injected abstractions rather than `new`. Then name what you removed: "I eliminated common coupling by deleting the global, control coupling by replacing two booleans with distinct methods, and construction coupling by injecting the repository." Naming the coupling types out loud is what scores.

---

## Practice exercise

### The Coupling Audit (~35 min)

**Part A — Audit real code (15 min).** Open any project you've written (or any open-source repo). Pick the single largest file in `src/`. Then produce a table with these five numbers:

1. **Fan-out** — count its `import`/`require` statements.
2. **The `and` count** — write a one-sentence description of the file. How many times did you need "and"?
3. **Change amplification** — pick a plausible small feature ("add a `phoneNumber` field to User"). Count how many files you'd touch.
4. **Mock count** — pick its most important method. How many things would you have to fake to unit-test it?
5. **Co-change** — run the `git log` co-change command from section 4 on it. List the top 5 files it always ships with. Are any of them in a *different* module?

**Part B — Classify (10 min).** For each dependency in that file, label the coupling type: content / common / control / stamp / data / message. Then classify the file's cohesion: functional / sequential / communicational / procedural / temporal / logical / coincidental. Be honest.

**Part C — Refactor one thing (10 min).** Pick the **single worst** coupling you found and fix only that one. Most likely candidates, in order of payoff:
- A boolean parameter → split into two methods, or take a strategy/value object.
- A `new SomeConcreteThing()` inside a constructor → accept it as a constructor argument instead.
- A whole fat object passed where two fields were needed → pass the two fields.

**What to produce:** the 5-number table, the coupling/cohesion labels, and a real diff (even 10 lines) of the one thing you fixed. Then re-run the mock count on the refactored method — if it didn't go down, you fixed the wrong thing.

---

## Quick reference cheat sheet

- **Coupling = between modules.** How much they depend on each other. **Want LOW.**
- **Cohesion = within a module.** How focused it is on one job. **Want HIGH.**
- **The image:** cohesion is what's *inside* the box; coupling is the *wires between* boxes.
- **Coupling, worst → best:** Content → Common (globals) → Control (flags) → Stamp (fat objects) → **Data (primitives)** → Message (events).
- **Cohesion, best → worst:** **Functional** → Sequential → Communicational → Procedural → Temporal → Logical → Coincidental.
- **`utils.js` / `helpers.js` / `common.js`** are confessions of coincidental cohesion. Name modules after what they *do*.
- **A boolean parameter is control coupling.** Two booleans = four code paths, three of them untested. Split the method.
- **A shared mutable global is common coupling** — an invisible dependency that makes order-of-loading a bug source. Inject it instead.
- **SRP = high cohesion**, stated as a rule ([14](./14-solid-single-responsibility.md)). **DIP = low coupling**, stated as a rule ([18](./18-solid-dependency-inversion.md)).
- **The `and` test:** if you can't describe the class in one sentence without "and", split it.
- **The blast-radius test:** "if I change X, how many files do I touch?" More than 3 → your coupling is too high.
- **The mock test:** more than 2–3 mocks to unit-test one method → too coupled. Tests are a coupling detector you already own.
- **Git co-change is the best real-world coupling metric.** Files that always commit together are coupled, whatever the imports say.
- **They move together:** fixing coupling usually raises cohesion, because each concern is forced into its own home.
- **Don't over-correct.** An interface with one implementation, added "just in case", is YAGNI. Add abstraction when there's a second implementation or a slow dependency.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [23 — Class Relationships](./23-class-relationships.md) — the four kinds of wire; this doc is about how *few* of them you should string |
| **Next** | [25 — Design by Contract](./25-design-by-contract.md) — once modules are decoupled, the contract at each boundary is what keeps them honest |
| **Related** | [14 — SOLID: Single Responsibility](./14-solid-single-responsibility.md) — high cohesion, stated as an actionable rule |
| **Related** | [18 — SOLID: Dependency Inversion](./18-solid-dependency-inversion.md) — the primary technique for driving coupling down |
| **Related** | [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) — the guardrails that stop "lower the coupling" from becoming abstraction astronautics |
