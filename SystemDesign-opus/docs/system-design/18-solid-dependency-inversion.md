# 18 — SOLID — Dependency Inversion Principle (DIP)

## Category: LLD Fundamentals

---

## What is this?

The Dependency Inversion Principle says two things:

1. **High-level modules should not depend on low-level modules. Both should depend on abstractions.**
2. **Abstractions should not depend on details. Details should depend on abstractions.**

In plain English: your **business logic** (placing an order, calculating a refund) should not know that MySQL exists, or that you use SendGrid for email. It should know only that *"something can save an order"* and *"something can send an email."* The concrete MySQL and SendGrid code plugs into those slots from outside.

Think of a **wall socket**. Your lamp doesn't depend on the power station. The lamp depends on the **socket standard** — 3 pins, 230V. The power station also conforms to that standard. Neither knows about the other. Swap coal for solar and the lamp doesn't notice. **The socket is the abstraction, and it points *away* from both of them.**

---

## Why does it matter?

Because of one word: **testability**. And after that: **replaceability**.

Look at this — you have written this, everyone has written this:

```javascript
// ❌ The most common shape of code in the world.
const db = require('./mysql-database');
const mailer = require('./sendgrid-mailer');

class OrderService {
  async placeOrder(userId, items) {
    const order = await db.query('INSERT INTO orders ...');   // ← hard-wired to MySQL
    await mailer.send(userId, 'Order confirmed');             // ← hard-wired to SendGrid
    return order;
  }
}
```

Now try to unit-test `placeOrder`. You **cannot**, in any honest sense. To run that one function you need:

- a live MySQL server, with the right schema, seeded
- a SendGrid API key, and network access
- and every test run sends a real email to a real address

So what happens in practice? Nobody unit-tests it. They write a slow, flaky integration test, or they write no test at all, or they reach for a horror like `jest.mock('./sendgrid-mailer')` and monkey-patch the module registry. That last one *works*, and it's a warning light: **you're using a testing-framework hack to compensate for a design flaw.**

**Interview angle:** DIP is where "I know SOLID" becomes "I've actually built things." The killer follow-up is always *"okay, so how do you test it?"* — and DIP is the answer.

**Real-work angle:** DIP is what lets you swap Postgres for DynamoDB, SendGrid for SES, or Stripe for Adyen by writing **one new file** and changing **one line of wiring** — instead of a two-week grep-and-replace across your business logic.

---

## The core idea — explained simply

### The "New Chef" Analogy

You own a restaurant. Your **head chef** (high-level: the business logic) needs ingredients.

**Setup A — the chef depends on a specific farm (BAD).**

The chef's recipe card literally says: *"Drive to Marco's Farm on Route 9. Enter through the red gate. Marco keeps the tomatoes in the third barn."*

Now:
- Marco retires. **Every recipe card must be rewritten.**
- You want to *test* the recipe in a training kitchen? You can't — the recipe requires a 40-minute drive to a real farm.
- You want a second restaurant in another city? The recipes are useless there.

The chef — your most valuable, most stable asset — is **coupled to a detail that changes far more often than he does.** That's the inversion problem in one sentence.

**Setup B — the chef depends on a "supplier" contract (GOOD).**

The recipe card now says: *"Ask the supplier for 2kg of tomatoes."* A **supplier** is anyone who can answer `getTomatoes(2)`.

- Marco is a supplier. So is the new farm. So is the supermarket. So is a **cardboard box of plastic tomatoes in the training kitchen.**
- Swapping Marco for someone else: you change *who you hand the chef*, not the recipes.
- Testing the recipe: hand the chef the plastic tomatoes. The recipe runs in 3 milliseconds.

The chef never knew Marco's name. **That's dependency inversion.**

### Mapping the analogy back

| Analogy | Code |
|---|---|
| The head chef | **High-level module** — `OrderService`, your business logic |
| Marco's Farm, the specific red gate | **Low-level module** — `MySQLDatabase`, `SendGridMailer` |
| "Ask the supplier for X" | The **abstraction / port** — `OrderRepository`, `Mailer` |
| Handing the chef a supplier at the start of the shift | **Constructor injection** |
| The restaurant manager who decides which supplier to hand over | The **composition root** / IoC container |
| Plastic tomatoes in the training kitchen | The **fake / stub** in your unit test |
| The chef never learns Marco's name | High-level module has **zero `require`** of any concrete detail |

### Why "inversion"? What is being inverted?

This is the part people memorise without understanding. Here's the answer.

**Normally**, control flows downward and dependency flows the same way: `OrderService` **calls** MySQL, so `OrderService` **depends on** MySQL. Arrow points down.

```
   OrderService  ──depends on──▶  MySQLDatabase
   (high-level)                   (low-level, changes often)
```

**After inversion**, control *still* flows downward — `OrderService` still ends up calling MySQL at runtime — but the **dependency arrow now points UP**, from MySQL to the abstraction that the business layer owns:

```
   OrderService  ──depends on──▶  OrderRepository  «abstraction»
   (high-level)                          ▲
                                         │ implements  ← ARROW FLIPPED
                                  MySQLOrderRepository
                                  (low-level, changes often)
```

**The direction of the source-code dependency has been inverted relative to the direction of the call.** That's it. That's the whole name.

And the key detail everyone misses: **the abstraction belongs to the high-level module, not the low-level one.** `OrderRepository` is defined by and for the business logic — it speaks the business's language (`save(order)`, `findByUser(userId)`), not the database's language (`executeQuery`, `getConnection`). If your "abstraction" has a method called `runSQL()`, you haven't inverted anything; you've just wrapped MySQL in a thin coat of paint.

---

## Key concepts inside this topic

### 1. The BAD version — and exactly why it's untestable

```javascript
// ===== src/services/order-service.js =====
// ❌ BAD. High-level business logic reaching DOWN into concrete details.

const db = require('../infra/mysql-database');       // ← concrete
const mailer = require('../infra/sendgrid-mailer');  // ← concrete
const stripe = require('../infra/stripe-client');    // ← concrete

class OrderService {
  async placeOrder(userId, items) {
    const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);

    // The BUSINESS RULE — the only part that's actually valuable:
    if (total <= 0) throw new Error('Order total must be positive');
    if (items.length > 50) throw new Error('Max 50 items per order');

    const charge = await stripe.charge(userId, total);
    if (charge.status !== 'CAPTURED') throw new Error('Payment failed');

    const order = await db.query(
      'INSERT INTO orders (user_id, total, status) VALUES (?, ?, ?)',
      [userId, total, 'CONFIRMED']
    );

    await mailer.send({
      to: await db.query('SELECT email FROM users WHERE id = ?', [userId]),
      subject: 'Order confirmed',
      body: `Your order #${order.insertId} for $${total} is confirmed.`
    });

    return { orderId: order.insertId, total };
  }
}

module.exports = { OrderService };
```

Count the problems:

| Problem | Consequence |
|---|---|
| `require`s three concrete infrastructure modules at the top | **Importing this file boots a MySQL pool and a Stripe client.** Just importing it. |
| The business rules (`total > 0`, `max 50 items`) are buried in I/O | You cannot test the rules without the I/O |
| To test, you need a DB, an SMTP key, and a Stripe key | Tests are slow (seconds), flaky (network), and **send real emails** |
| Swapping MySQL → Postgres means editing this file | Business logic changes because a *storage* decision changed. Nonsense. |
| `OrderService` speaks SQL | The high-level module knows the low-level module's private language |

That fourth row is the deepest one. **Your business logic changed for a reason that has nothing to do with the business.** That's the smell DIP exists to kill.

### 2. Define the abstractions — "ports"

An abstraction here is called a **port** (from hexagonal architecture — more in section 5). It's the socket in the wall. In JS, a port is a base class with throwing methods, or honestly just a documented duck-typed shape.

Critically: **name the methods in the language of the business, not the technology.**

```javascript
// ===== src/core/ports.js =====
// ✅ OWNED BY THE BUSINESS LAYER. They know nothing about SQL, HTTP, or SMTP.
//    Read them aloud: they describe INTENT, not mechanism.

class OrderRepository {
  async save(order)  { throw new Error('Not implemented'); }
  async findById(id) { throw new Error('Not implemented'); }
}

class UserRepository {
  async findById(id) { throw new Error('Not implemented'); }
}

class Mailer {
  // NOT `sendSmtp()`. NOT `postToSendGridApi()`. Just: send a message.
  async send({ to, subject, body }) { throw new Error('Not implemented'); }
}

class PaymentGateway {
  // Returns { status: 'CAPTURED' | 'DECLINED', reference }
  async charge(userId, amount) { throw new Error('Not implemented'); }
}

module.exports = { OrderRepository, UserRepository, Mailer, PaymentGateway };
```

Notice these ports are **small** — that's [17 — ISP](./17-solid-interface-segregation.md) doing its job. A fat port would make every fake huge and defeat the purpose. And each one must be honestly implementable by *every* adapter — that's [16 — LSP](./16-solid-liskov-substitution.md). The three principles interlock: **ISP keeps the ports small, LSP keeps the adapters honest, DIP points the arrows at them.**

### 3. The GOOD version — constructor injection

```javascript
// ===== src/core/order-service.js =====
// ✅ GOOD. Notice what's missing: there is NOT A SINGLE `require` of
//    mysql, sendgrid, or stripe. This file has zero infrastructure imports.

class OrderService {
  // The dependencies arrive from OUTSIDE. The service never constructs them,
  // never imports them, never knows their concrete type.
  constructor({ orderRepo, userRepo, mailer, paymentGateway }) {
    this.orderRepo = orderRepo;
    this.userRepo = userRepo;
    this.mailer = mailer;
    this.paymentGateway = paymentGateway;
  }

  async placeOrder(userId, items) {
    const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);

    // The business rules are now front and centre — and independently testable.
    if (total <= 0) throw new Error('Order total must be positive');
    if (items.length > 50) throw new Error('Max 50 items per order');

    const user = await this.userRepo.findById(userId);
    if (!user) throw new Error('User not found');

    const charge = await this.paymentGateway.charge(userId, total);
    if (charge.status !== 'CAPTURED') throw new Error('Payment failed');

    const order = await this.orderRepo.save({
      userId, items, total, status: 'CONFIRMED', paymentRef: charge.reference
    });

    await this.mailer.send({
      to: user.email,
      subject: 'Order confirmed',
      body: `Your order #${order.id} for $${total} is confirmed.`
    });

    return { orderId: order.id, total };
  }
}

module.exports = { OrderService };
```

Read that file again and ask: **what technology does it use?** You can't tell. That's the goal. This file would work identically on Postgres, DynamoDB, or a JSON file on disk — and you could port it to a completely different company's infrastructure without touching a line.

Now the **adapters** — the low-level details that plug into the ports. They live in a separate folder, and the dependency arrow points *up* from them:

```javascript
// ===== src/adapters/mysql-order-repository.js =====
// ← low-level requires the HIGH-LEVEL's abstraction. That is the INVERSION.
const { OrderRepository } = require('../core/ports');

class MySQLOrderRepository extends OrderRepository {
  constructor(pool) { super(); this.pool = pool; }

  async save(order) {
    const [res] = await this.pool.execute(
      'INSERT INTO orders (user_id, total, status, payment_ref) VALUES (?, ?, ?, ?)',
      [order.userId, order.total, order.status, order.paymentRef]
    );
    return { ...order, id: res.insertId };
  }

  async findById(id) {
    const [rows] = await this.pool.execute('SELECT * FROM orders WHERE id = ?', [id]);
    return rows[0] ?? null;
  }
}

// ===== src/adapters/stripe-payment-gateway.js =====
const { PaymentGateway } = require('../core/ports');

class StripePaymentGateway extends PaymentGateway {
  constructor(stripe) { super(); this.stripe = stripe; }
  async charge(userId, amount) {
    const res = await this.stripe.charges.create({
      amount: Math.round(amount * 100), currency: 'usd', customer: userId
    });
    // The adapter's job: TRANSLATE the vendor's language into the port's language.
    return { status: res.paid ? 'CAPTURED' : 'DECLINED', reference: res.id };
  }
}
```

That translation line is the whole point of an adapter. Stripe says `res.paid`. Your business says `'CAPTURED'`. The adapter is the only place in your entire codebase that knows both dialects — **it's a translator, and it's the only thing that has to change when Stripe changes.**

### 4. The composition root — where the wiring actually happens

Someone has to say "MySQL goes in the OrderRepository slot." That someone is the **composition root**: one file, at the very edge of your app, that is allowed to know about everything.

```javascript
// ===== src/main.js  — THE COMPOSITION ROOT =====
// This is the ONLY file that knows both the abstractions AND the concretes.
// It is the "restaurant manager" who hands the chef a supplier.

const mysql = require('mysql2/promise');
const sgMail = require('@sendgrid/mail');
const Stripe = require('stripe');

const { OrderService } = require('./core/order-service');
const { MySQLOrderRepository } = require('./adapters/mysql-order-repository');
const { MySQLUserRepository } = require('./adapters/mysql-user-repository');
const { SendGridMailer } = require('./adapters/sendgrid-mailer');
const { StripePaymentGateway } = require('./adapters/stripe-payment-gateway');

async function buildApp() {
  // 1. Construct the low-level details.
  const pool = await mysql.createPool(process.env.DATABASE_URL);
  sgMail.setApiKey(process.env.SENDGRID_KEY);
  const stripe = new Stripe(process.env.STRIPE_KEY);

  // 2. Wrap them in adapters.
  const orderRepo = new MySQLOrderRepository(pool);
  const userRepo = new MySQLUserRepository(pool);
  const mailer = new SendGridMailer(sgMail);
  const paymentGateway = new StripePaymentGateway(stripe);

  // 3. INJECT them into the high-level module.
  const orderService = new OrderService({ orderRepo, userRepo, mailer, paymentGateway });

  return { orderService };
}

module.exports = { buildApp };
```

Now, the promised payoff. **Swap MySQL for Postgres.** What changes?

- Write `src/adapters/postgres-order-repository.js` — **one new file.**
- In `main.js`, change `new MySQLOrderRepository(pool)` → `new PostgresOrderRepository(pool)` — **one line.**
- `OrderService`: **untouched.** Its tests: **untouched, and still passing.**

That's the entire promise of DIP, and it is not theoretical — this is how real teams do database migrations without a rewrite.

### 5. Dependency Injection, IoC, and IoC containers — in plain English

These three terms get used interchangeably and they are *not* the same thing. Here's the honest breakdown:

| Term | What it actually means | One-liner |
|---|---|---|
| **DIP** (Dependency Inversion **Principle**) | A **design principle**: depend on abstractions, and let the abstraction be owned by the high-level module | *The goal* |
| **DI** (Dependency **Injection**) | A **technique**: pass dependencies in from outside instead of constructing them inside (`new`) or importing them (`require`) | *The mechanism* |
| **IoC** (Inversion of **Control**) | The **general idea** that something *else* decides when/what — your code gets called rather than doing the calling. DI is one flavour of IoC | *The umbrella* |
| **IoC container** | A **library** (`awilix`, `tsyringe`, `inversify`, NestJS's built-in) that auto-wires the graph for you so you don't hand-write the composition root | *The convenience* |

**You do not need an IoC container to do DIP.** The `main.js` above is DIP, fully, with zero libraries. Containers only start to earn their keep when the object graph is big enough that hand-wiring becomes error-prone — typically 30+ services.

Here's what a container buys you, using `awilix`:

```javascript
// With an IoC container: you REGISTER the parts; it figures out the wiring
// by matching constructor parameter names to registration names.
const { createContainer, asClass, asValue } = require('awilix');

const container = createContainer();
container.register({
  pool:           asValue(pool),
  orderRepo:      asClass(MySQLOrderRepository).singleton(),
  userRepo:       asClass(MySQLUserRepository).singleton(),
  mailer:         asClass(SendGridMailer).singleton(),
  paymentGateway: asClass(StripePaymentGateway).singleton(),
  orderService:   asClass(OrderService).singleton(),
});

const orderService = container.resolve('orderService');  // whole graph built automatically
```

**But be honest about the cost:** the container makes the dependency graph *implicit*. A missing registration is now a runtime error instead of a `TypeError` you'd have caught immediately. For most Node apps, a hand-written composition root is clearer. **Start with hand-wiring; reach for a container when the wiring file becomes the problem, not before.**

#### Hexagonal architecture (ports and adapters)

This is just DIP taken to the whole-application level, and it's the architecture NestJS, Spring, and most serious backend codebases converge on. Here's the whole idea:

- Your **core** (domain + use cases) sits in the middle. It has **zero imports from any framework, database, or vendor SDK.**
- Around it are **ports** — the abstractions the core defines: `OrderRepository`, `Mailer`.
- Around those are **adapters** — the concrete implementations: `MySQLOrderRepository`, `ExpressHttpController`, `KafkaEventPublisher`.
- **Every arrow points inward.** Nothing in the core points out.

The rule you can enforce with a lint rule and win the argument forever:

> **`src/core/` may not `require` anything from `src/adapters/` or `node_modules` (except pure utilities).**

If that rule holds, your business logic is portable, testable in milliseconds, and outlives every framework you'll use.

"Driving" adapters (HTTP controller, CLI, cron) call *into* the core. "Driven" adapters (DB, mailer, payment) are called *by* the core through ports. Same core, either direction — that's why the same use case can be exposed over HTTP, over a queue, and over a CLI without duplicating a line of logic.

### 6. The test that only becomes possible after inversion

This is the payoff. Before DIP, this test literally could not be written. Now it's ten lines.

```javascript
// ===== tests/order-service.test.js =====
const { OrderService } = require('../src/core/order-service');

// The "plastic tomatoes". Fakes — no DB, no network, no API keys, no mocking library.
const makeDeps = (overrides = {}) => ({
  userRepo: { findById: jest.fn(async () => ({ id: 'u1', email: 'ana@shop.com' })) },
  orderRepo: { save: jest.fn(async (o) => ({ ...o, id: 999 })), findById: jest.fn() },
  paymentGateway: { charge: jest.fn(async () => ({ status: 'CAPTURED', reference: 'ch_1' })) },
  mailer: { send: jest.fn(async () => {}) },
  ...overrides,
});

describe('OrderService.placeOrder', () => {
  test('places an order, charges the card, and emails the user', async () => {
    const deps = makeDeps();
    const service = new OrderService(deps);

    const result = await service.placeOrder('u1', [
      { price: 10, qty: 2 },   // 20
      { price: 5,  qty: 1 },   // 5
    ]);

    expect(result).toEqual({ orderId: 999, total: 25 });
    expect(deps.paymentGateway.charge).toHaveBeenCalledWith('u1', 25);
    expect(deps.orderRepo.save).toHaveBeenCalledWith(
      expect.objectContaining({ total: 25, status: 'CONFIRMED', paymentRef: 'ch_1' })
    );
    expect(deps.mailer.send).toHaveBeenCalledWith(
      expect.objectContaining({ to: 'ana@shop.com' })
    );
  });

  test('does NOT save the order or email anyone if payment is declined', async () => {
    const deps = makeDeps({
      paymentGateway: { charge: jest.fn(async () => ({ status: 'DECLINED' })) },
    });
    const service = new OrderService(deps);

    await expect(service.placeOrder('u1', [{ price: 10, qty: 1 }]))
      .rejects.toThrow('Payment failed');

    // THIS is the assertion you could never make before. It proves a real
    // business invariant: a declined payment leaves NO trace.
    expect(deps.orderRepo.save).not.toHaveBeenCalled();
    expect(deps.mailer.send).not.toHaveBeenCalled();
  });

  test('rejects orders over 50 items before touching payment', async () => {
    const deps = makeDeps();
    const service = new OrderService(deps);
    const items = Array(51).fill({ price: 1, qty: 1 });

    await expect(service.placeOrder('u1', items)).rejects.toThrow('Max 50 items');
    expect(deps.paymentGateway.charge).not.toHaveBeenCalled();   // fail fast, no money moved
  });
});
```

Look at what you gained:

| | Before DIP | After DIP |
|---|---|---|
| Runtime of these 3 tests | ~8 seconds (DB + 2 network calls) | **~15 ms** |
| Infrastructure needed | MySQL, SendGrid key, Stripe key | **none** |
| Flakiness | network timeouts, dirty test DB | **zero** |
| Can you assert "payment declined ⇒ nothing saved"? | Not without inspecting a real DB | **One line** |
| Real emails sent | yes 😬 | none |
| Mocking library needed | `jest.mock()` module hackery | **none** — just plain objects |

That last row is worth pausing on. **If you find yourself reaching for `jest.mock('../infra/db')`, that's not a testing technique — it's a symptom.** You're using the module loader to work around a dependency you should have injected. DIP makes the mocking library unnecessary.

---

## Visual / Diagram description

### Diagram 1: BEFORE — dependencies point down into the details

```
   ┌───────────────────────────────────────────────────────────┐
   │                      OrderService                          │
   │                     (HIGH-LEVEL)                           │
   │                                                            │
   │   require('../infra/mysql-database')   ──────────┐         │
   │   require('../infra/sendgrid-mailer')  ────────┐ │         │
   │   require('../infra/stripe-client')    ──────┐ │ │         │
   │                                              │ │ │         │
   │   placeOrder() { ... business rules ... }    │ │ │         │
   └──────────────────────────────────────────────┼─┼─┼─────────┘
                                                  │ │ │
                       dependency arrows point DOWN into concrete details
                                                  │ │ │
                    ┌─────────────────────────────┘ │ └──────────────┐
                    ▼                               ▼                ▼
        ┌───────────────────┐        ┌────────────────────┐  ┌────────────────┐
        │  MySQLDatabase    │        │  SendGridMailer    │  │  StripeClient  │
        │   (LOW-LEVEL)     │        │   (LOW-LEVEL)      │  │  (LOW-LEVEL)   │
        │                   │        │                    │  │                │
        │  needs: live DB   │        │  needs: API key    │  │  needs: API key│
        │  needs: schema    │        │  needs: network    │  │  needs: network│
        └───────────────────┘        └────────────────────┘  └────────────────┘

   ⚠️  To construct/import OrderService, you must have ALL THREE running.
   ⚠️  Swapping MySQL → Postgres means EDITING OrderService.
```

### Diagram 2: AFTER — the arrows are inverted

```
   ┌────────────────────────────────────────────────────────────────────┐
   │                        THE CORE  (src/core/)                        │
   │                                                                     │
   │   ┌──────────────────────────────────────────────────────────┐     │
   │   │                    OrderService                            │     │
   │   │                   (HIGH-LEVEL)                             │     │
   │   │   constructor({ orderRepo, mailer, paymentGateway })       │     │
   │   │   ZERO infrastructure requires                             │     │
   │   └────────┬──────────────┬─────────────────┬─────────────────┘     │
   │            │              │                 │                        │
   │      depends on     depends on        depends on                     │
   │            ▼              ▼                 ▼                        │
   │   ┌────────────────┐ ┌──────────┐  ┌──────────────────┐             │
   │   │OrderRepository │ │  Mailer  │  │ PaymentGateway   │  «PORTS»    │
   │   │  «abstraction» │ │«abstract»│  │  «abstraction»   │             │
   │   │  + save()      │ │ + send() │  │  + charge()      │             │
   │   │  + findById()  │ │          │  │                  │             │
   │   └───────▲────────┘ └────▲─────┘  └────────▲─────────┘             │
   └───────────┼───────────────┼─────────────────┼───────────────────────┘
               │               │                 │
          implements      implements        implements     ◀── ARROWS NOW
               │               │                 │             POINT UP
   ┌───────────┴───────────────┴─────────────────┴───────────────────────┐
   │                     ADAPTERS  (src/adapters/)                        │
   │  ┌────────────────────┐ ┌──────────────────┐ ┌────────────────────┐ │
   │  │MySQLOrderRepository│ │  SendGridMailer  │ │StripePaymentGateway│ │
   │  │    (LOW-LEVEL)     │ │   (LOW-LEVEL)    │ │    (LOW-LEVEL)     │ │
   │  └────────────────────┘ └──────────────────┘ └────────────────────┘ │
   │                                                                      │
   │  ┌────────────────────┐ ┌──────────────────┐ ┌────────────────────┐ │
   │  │InMemoryOrderRepo   │ │   FakeMailer     │ │  FakeGateway       │ │
   │  │   (FOR TESTS)      │ │   (FOR TESTS)    │ │  (FOR TESTS)       │ │
   │  └────────────────────┘ └──────────────────┘ └────────────────────┘ │
   └──────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ wires everything together
                       ┌────────────┴────────────┐
                       │  main.js                │
                       │  «COMPOSITION ROOT»     │
                       │  the ONLY file that     │
                       │  knows about both sides │
                       └─────────────────────────┘
```

**The single most important thing in this diagram:** the ports live *inside* the core box. They are owned by the business logic, not by the database. Every arrow crossing the core's boundary points **inward**. Draw that box on a whiteboard and you've drawn hexagonal architecture.

### Diagram 3: The runtime call vs. the source dependency

```
   SOURCE-CODE DEPENDENCY               RUNTIME CALL FLOW
   (compile time — who imports whom)    (what actually happens)

   OrderService                          OrderService
        │                                     │
        │ depends on                          │ calls .save()
        ▼                                     ▼
   OrderRepository «port»                OrderRepository «port»
        ▲                                     │
        │ implements                          │ dispatches to
        │                                     ▼
   MySQLOrderRepository                  MySQLOrderRepository
                                              │
                                              ▼
                                         [ actual MySQL ]

   ⬆ arrow points UP                    ⬇ control flows DOWN
```

**These two diagrams point in opposite directions, and that is the entire meaning of the word "inversion."** Control still flows down to MySQL at runtime; the *source-code dependency* has been flipped to point up. Being able to say this sentence out loud is what a DIP interview question is really testing.

---

## Real world examples

### 1. NestJS — DIP as the framework's core mechanic

NestJS (the dominant "enterprise" Node framework) is built on constructor injection and a built-in IoC container. You declare what you need and the framework wires it:

```javascript
@Injectable()
class OrderService {
  constructor(
    @Inject('ORDER_REPOSITORY') private orderRepo,
    @Inject('MAILER') private mailer,
  ) {}
}
```

Then a module file (the composition root) binds the token `'ORDER_REPOSITORY'` to a concrete class. In tests, `Test.createTestingModule({...}).overrideProvider('ORDER_REPOSITORY').useValue(fakeRepo)` swaps in a fake with one line. The reason NestJS won the enterprise-Node market is largely that it made DIP the default instead of an act of discipline.

### 2. Express itself — the injected `app`

A very ordinary DIP win that most Node devs do without naming it:

```javascript
// ❌ BAD — the module builds and listens on a real port at import time.
const app = express();
app.listen(3000);
module.exports = app;

// ✅ GOOD — the routes depend on an INJECTED service, and the server
//    is started by the composition root, not by the module.
function createOrderRoutes(orderService) {           // ← injected
  const router = express.Router();
  router.post('/orders', async (req, res) => {
    const result = await orderService.placeOrder(req.user.id, req.body.items);
    res.status(201).json(result);
  });
  return router;
}
```

This is exactly why `supertest` works: because `app` is a value you *build*, not a server that *starts itself*, your test can drive it in-process with no port, no network, and no teardown. The whole `supertest` ecosystem exists because Express (mostly) got the inversion right.

### 3. Database drivers behind a repository — the migration that didn't hurt

The pattern is well documented across the industry: teams that put their persistence behind a repository port (Prisma/TypeORM/Knex hidden behind `OrderRepository`) can change database engines by writing one new adapter. Teams that scattered raw SQL through their services do a rewrite.

The concrete difference shows up in the diff. In an inverted codebase, a Postgres→DynamoDB migration touches `src/adapters/` and `main.js` and nothing else — and every single business-logic test keeps passing *unmodified*, which is precisely how you prove the migration didn't change behaviour. In a non-inverted codebase, the same migration touches every service file, and you have no green test suite to trust while you do it. **DIP's real gift isn't flexibility — it's a trustworthy safety net during exactly the changes that scare you most.**

---

## Trade-offs

| Approach | Pros | Cons |
|---|---|---|
| **Direct `require` of concretes (no DIP)** | Dead simple; no wiring; easy to follow with jump-to-definition | Untestable without real infra; swapping a vendor means editing business logic; slow, flaky tests |
| **Constructor injection + hand-wired composition root** | Full DIP benefits; the wiring is explicit and greppable; zero libraries; **the sweet spot for most Node apps** | You must maintain the wiring file; a little boilerplate per service |
| **IoC container (awilix / NestJS / inversify)** | Auto-wires large graphs; scoped lifetimes (per-request); less boilerplate at scale | Dependency graph becomes implicit and hard to trace; missing registrations become *runtime* errors; a real learning curve |
| **`jest.mock()` on module imports** | Requires zero refactor today | Not a design — a workaround. Brittle (breaks when you move a file); doesn't help you swap vendors in production; hides the coupling instead of removing it |

| You give up… | To get… |
|---|---|
| The convenience of `require`-ing a DB right where you use it | Business logic that boots in 0 ms and tests in 15 ms |
| A little indirection (one more hop to find the real impl) | The ability to swap MySQL → Postgres by writing one file |
| Some ceremony in the wiring file | A test suite that never sends a real email at 3am |

**When NOT to invert:** don't create a port for something with exactly one implementation that will never change and that you never need to fake in tests. A `formatCurrency()` utility does not need an `ICurrencyFormatter` port. Invert at the boundaries where the outside world lives — **I/O, network, time, randomness, third-party vendors** — and leave the pure functions alone. (That's [19 — YAGNI](./19-dry-kiss-yagni.md) keeping DIP honest.)

**Rule of thumb:** *If it does I/O, or if it's someone else's SDK, put a port in front of it. If it's pure logic, don't.* Then ask the question that settles every argument: **"Can I unit-test my business rules with no network and no database?"** If yes, you've inverted enough. If no, you haven't.

---

## Common interview questions on this topic

### Q1: "What exactly is being *inverted* in Dependency Inversion?"
**Hint:** The **source-code dependency arrow**, relative to the direction of the runtime call. Normally `OrderService` calls MySQL, so it also *depends on* MySQL — both arrows point down. After inversion, control still flows down at runtime, but the *source dependency* now points **up**: `MySQLOrderRepository` depends on `OrderRepository`, which the business layer owns. Say the phrase "control flows down, dependency points up" and you've nailed it.

### Q2: "What's the difference between DIP, DI, and an IoC container?"
**Hint:** **DIP** is the *principle* (depend on abstractions owned by the high-level module). **DI** is the *technique* that achieves it (pass dependencies in rather than `new`/`require` them inside). An **IoC container** is a *library* that automates the wiring. You can — and for most Node apps, should — do DIP with plain constructor injection and zero libraries.

### Q3: "How do you know you've actually inverted, and not just added an indirection layer?"
**Hint:** Two tests. **(a)** Does the abstraction speak the *business's* language (`save(order)`) or the *technology's* (`runQuery(sql)`)? If it's the latter, you wrapped MySQL, you didn't invert. **(b)** Who *owns* the abstraction — is it defined in the core, next to the business logic, or in the infra folder next to the driver? A port that lives in `src/infra/` is a red flag. The abstraction must belong to the high-level module.

### Q4: "Where do you draw the line — should everything be behind an interface?"
**Hint:** No. Invert at the **boundaries with the outside world**: databases, HTTP clients, message queues, mailers, payment vendors, the clock, the random-number generator, the filesystem. Don't invert pure functions or value objects — a `Money` class or a `slugify()` helper needs no port. The heuristic: *if it does I/O or is a third-party SDK, port it; if it's pure logic, leave it.* Over-porting is YAGNI, and it makes the codebase harder to read for zero benefit.

### Q5: "I can just use `jest.mock()` to fake the database. Why bother with DIP?"
**Hint:** This is the trap question, and the right answer is confident: *"`jest.mock()` lets me fake a dependency in tests, but it doesn't let me swap one in production."* The coupling is still there — it's just been papered over in the test environment. Module mocking is also brittle (it breaks when you move a file), and it means my test file has to know my implementation's import paths, which is coupling to a *file layout*. DIP gives me both testability **and** replaceability; module mocking gives me a fragile half of one.

---

## Practice exercise

### Invert the Notification Service

Here's the code. It works. It ships. It also has zero tests, and you're about to find out why.

```javascript
// notification-service.js
const twilio = require('twilio')(process.env.TWILIO_SID, process.env.TWILIO_TOKEN);
const sgMail = require('@sendgrid/mail');
const redis = require('redis').createClient({ url: process.env.REDIS_URL });
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

sgMail.setApiKey(process.env.SENDGRID_KEY);

class NotificationService {
  async notify(userId, message) {
    // Business rule 1: respect the user's channel preference.
    const { rows } = await pool.query(
      'SELECT email, phone, preferred_channel, quiet_hours FROM users WHERE id = $1',
      [userId]
    );
    const user = rows[0];
    if (!user) throw new Error('User not found');

    // Business rule 2: never notify during quiet hours.
    const hour = new Date().getHours();
    if (user.quiet_hours && hour >= 22) return { sent: false, reason: 'QUIET_HOURS' };

    // Business rule 3: rate limit — max 5 notifications per user per hour.
    const key = `notif:${userId}:${hour}`;
    const count = await redis.incr(key);
    await redis.expire(key, 3600);
    if (count > 5) return { sent: false, reason: 'RATE_LIMITED' };

    // Deliver.
    if (user.preferred_channel === 'SMS') {
      await twilio.messages.create({ to: user.phone, from: '+15550000', body: message });
    } else {
      await sgMail.send({ to: user.email, from: 'noreply@app.com', subject: 'Update', text: message });
    }
    return { sent: true, channel: user.preferred_channel };
  }
}
```

**Produce these (~40 min):**

1. **List every reason this class is untestable.** There are at least five, and one of them is *not* a `require` statement. (Strong hint: read business rule 2 again. `new Date()` is a hidden dependency on the outside world — the clock. How do you write a test that runs at 23:00?)

2. **Define the ports.** Write `ports.js` with the abstractions the *business* needs. Name them in business language. There should be roughly four, and one of them will be a `Clock`. Keep them small — apply ISP.

3. **Rewrite `NotificationService`** with constructor injection. **Success criterion: the file has zero `require` statements** other than possibly a pure utility.

4. **Write the adapters:** `PostgresUserRepository`, `RedisRateLimiter`, `TwilioSmsSender`, `SendGridEmailSender`, `SystemClock`. Note that SMS and email are two implementations of the same port — that's a Strategy in disguise ([42 — Strategy Pattern](./42-pattern-strategy.md)).

5. **Write the composition root** (`main.js`) that wires the real thing.

6. **Write the Jest tests that were previously impossible.** At minimum, these four:
   - notifies via SMS when that's the user's preference
   - **returns `QUIET_HOURS` at 23:00** — using a `FakeClock`, with no waiting and no mocking library
   - **returns `RATE_LIMITED` on the 6th call in an hour** — and asserts that **no SMS was sent**
   - throws for an unknown user, and asserts the sender was never touched

   Every test must run in **under 50ms total**, with **no network, no DB, no Redis, and no `jest.mock()`**.

7. **The final proof.** Add an in-memory `PushNotificationSender` adapter. Count the lines you changed in `NotificationService`. The answer must be **zero** — and if it isn't, your ports are wrong.

---

## Quick reference cheat sheet

- **DIP in one line:** high-level modules and low-level modules both depend on **abstractions** — and the abstraction is **owned by the high-level module**.
- **The inversion:** control still flows **down** to MySQL at runtime; the **source-code dependency arrow now points up**. Say that sentence in the interview.
- **DIP ≠ DI ≠ IoC container.** Principle → technique → library. You need the first two; the third is optional.
- **The abstraction must speak business language.** `save(order)` ✅. `runQuery(sql)` ❌ — that's just MySQL wearing a hat.
- **Ownership test:** if the port lives in `src/infra/` next to the driver, you haven't inverted. Ports belong in the core.
- **The killer symptom:** you can't unit-test your business rules without a database. That's DIP screaming.
- **`jest.mock()` on a module import is a symptom, not a solution** — it fakes the dependency in tests but leaves it hard-wired in production.
- **Constructor injection is the workhorse:** `constructor({ orderRepo, mailer })`. That's 90% of DIP, with no library.
- **Composition root:** exactly one file (`main.js`) is allowed to know both abstractions and concretes. Everything else stays ignorant.
- **Hexagonal / ports-and-adapters** = DIP at app scale. Core in the middle, ports around it, adapters outside, **all arrows point inward**.
- **Invert at the boundaries** — I/O, network, third-party SDKs, the clock, randomness. **Not** pure functions.
- **The clock is a dependency.** So is `Math.random()`. Inject them, or you'll never test time-dependent rules.
- **The payoff, measured:** swapping a database becomes *one new file + one line of wiring*, with your entire business test suite still green and untouched.
- **The enforceable rule:** `src/core/` may not import from `src/adapters/` or from vendor SDKs. Lint it. Win the argument forever.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [17 — SOLID: Interface Segregation Principle](./17-solid-interface-segregation.md) — ISP keeps the ports small; DIP is what you inject them through |
| **Next** | [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) — the counterweight: don't invert what will never change |
| **Related** | [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md) — DIP is the single most powerful tool for reducing coupling |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — an injected port with several adapters *is* a Strategy; DIP and Strategy are the same shape at different scales |
