# 30 — Factory Method Pattern

## Category: LLD Patterns

---

## What is this?

The **Factory Method** pattern moves object creation out of your business code and into a dedicated **method whose job is to build things**. Instead of calling `new EmailNotifier()` yourself, you ask a factory: "give me a notifier for `'email'`," and it hands one back.

Think of a **coffee shop**. You don't walk behind the counter, grab the portafilter, and pull your own espresso shot. You say **"one latte, please"** at the register. Someone else knows which machine, which beans, which cup. You get a coffee. You never learned the recipe, and if they swap the espresso machine tomorrow, your order doesn't change.

That's the pattern: **the caller names what it wants; the factory knows how to build it.**

---

## Why does it matter?

Every time you write `new ConcreteClass()` in your business logic, you **weld** that logic to that class. Change the class, change the constructor's arguments, or add a new variant, and you have to go edit every place that said `new`.

**What breaks without it:**
- A giant `switch (type) { case 'email': return new EmailNotifier(...) ... }` block that appears in *six different files*, each slightly out of date with the others.
- Adding a WhatsApp notifier means touching every one of those six files. Miss one, ship a bug.
- Your `OrderService` unit test now needs SMTP credentials, because `OrderService` constructs a real `EmailNotifier` internally.

**The interview angle:** Factory is the most-used creational pattern in real code, and it's the setup for the classic follow-up: *"What's the difference between Simple Factory, Factory Method, and Abstract Factory?"* Most candidates fumble this. You won't, after this doc.

**The real-work angle:** You already use factories constantly in Node — `crypto.createHash('sha256')`, `fs.createReadStream(path)`, `http.createServer()`. Every one of those is a factory function returning a concrete object you never named.

---

## The core idea — explained simply

### The Rental Car Counter Analogy

You land at an airport and walk to the rental counter. You say:

> "I need an **SUV**."

You do **not** say: "Please go to bay 14, retrieve the 2023 Toyota Highlander with VIN 5TDZA..., initialize its odometer, attach the insurance rider, and hand me the keys."

The agent behind the counter — the **factory** — knows all that. She checks stock, picks a specific SUV, sets it up, and hands you keys to *a car*. You drive away. You interact with it through the **universal car interface**: steering wheel, pedals, gear shift. You don't care that it's a Highlander.

Now the value shows up:

- Next month they add **electric vehicles** to the fleet. You say "I need an EV." The counter handles it. **Your driving skills didn't change.**
- They retire the Highlander and buy Kia Sorentos. You say "SUV," you get a Sorento. **You didn't change.**
- The counter is the *single place* that knows which physical car maps to which category. Not fifty travel agents each with their own list.

| Analogy piece | Technical concept |
|---|---|
| "I need an SUV" | The **product type / key** you pass to the factory |
| The rental agent | The **Factory** (`create(type)`) |
| The specific Highlander she picks | The **Concrete Product** (`EmailNotifier`) |
| Steering wheel + pedals | The **Product interface** (base class / shared method signatures) |
| You, the driver | The **Client** — depends only on the interface |
| Adding EVs without retraining drivers | **Open/Closed Principle** — extend without modifying callers |
| The counter being the one place with the fleet list | Creation logic lives in **one** place, not scattered |

The whole trick: **the client depends on the abstraction ("a car"), never on the concrete class ("Highlander").**

---

## Key concepts inside this topic

### 1. The problem it solves — painful code without the pattern

You're building notifications. Version 1, written under deadline:

```javascript
// ---------- BAD: `new` scattered through business logic ----------
import nodemailer from 'nodemailer';
import twilio from 'twilio';

class OrderService {
  async confirmOrder(order, channel) {
    // ... business logic: charge card, decrement stock, write to DB ...

    // ...and then, awkwardly bolted on, the delivery mechanism:
    if (channel === 'email') {
      const transport = nodemailer.createTransport({
        host: process.env.SMTP_HOST,
        port: 587,
        auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS },
      });
      await transport.sendMail({
        to: order.user.email,
        subject: 'Order confirmed',
        text: `Order ${order.id} confirmed.`,
      });
    } else if (channel === 'sms') {
      const client = twilio(process.env.TWILIO_SID, process.env.TWILIO_TOKEN);
      await client.messages.create({
        to: order.user.phone,
        from: process.env.TWILIO_FROM,
        body: `Order ${order.id} confirmed.`,
      });
    } else if (channel === 'push') {
      // ...30 more lines of FCM setup...
    } else {
      throw new Error(`Unknown channel: ${channel}`);
    }
  }
}
```

Count the problems:

1. **`OrderService` knows about SMTP ports and Twilio SIDs.** It has no business knowing that. Its job is orders. (Violates [14 — SRP].)
2. **This exact `if/else` block is copy-pasted** into `ShippingService`, `RefundService`, and `PasswordResetService`. Four copies, already diverging.
3. **Adding WhatsApp** means finding and editing all four. (Violates [15 — OCP].)
4. **You cannot unit-test `confirmOrder`** without real SMTP credentials — it builds the transport itself. (Violates [18 — DIP].)
5. **Construction is expensive and repeated** — a new SMTP transport per order.

### 2. The refactor — Simple Factory

Step one: pull the `switch` out into one class, and make all notifiers share a common shape.

```javascript
// notifications/Notifier.js — the PRODUCT INTERFACE
export class Notifier {
  // JS has no `interface` keyword; a base class that throws is the idiom.
  async send(recipient, message) {
    throw new Error('Notifier.send() must be implemented by a subclass');
  }
}
```

```javascript
// notifications/EmailNotifier.js — a CONCRETE PRODUCT
import { Notifier } from './Notifier.js';

export class EmailNotifier extends Notifier {
  constructor({ transport, from }) {
    super();
    this.transport = transport;   // injected — testable
    this.from = from;
  }

  async send(recipient, message) {
    await this.transport.sendMail({
      from: this.from,
      to: recipient.email,
      subject: message.subject,
      text: message.body,
    });
    return { channel: 'email', to: recipient.email };
  }
}
```

And the factory:

```javascript
// notifications/NotificationFactory.js — the SIMPLE FACTORY
export class NotificationFactory {
  static create(channel, deps) {
    switch (channel) {
      case 'email': return new EmailNotifier(deps.email);
      case 'sms':   return new SmsNotifier(deps.sms);
      case 'push':  return new PushNotifier(deps.push);
      default: throw new Error(`Unsupported channel: ${channel}`);
    }
  }
}
```

Now `OrderService` shrinks to this:

```javascript
// ---------- GOOD ----------
class OrderService {
  constructor(notificationFactory) { this.factory = notificationFactory; }

  async confirmOrder(order, channel) {
    // ... business logic ...
    const notifier = this.factory.create(channel);          // ask, don't build
    await notifier.send(order.user, {
      subject: 'Order confirmed',
      body: `Order ${order.id} confirmed.`,
    });
  }
}
```

`OrderService` no longer knows SMTP exists. **The `switch` still exists — but now there is exactly one of it, in the one class whose job is creation.** That's the entire win, and it's a big one.

Note what did NOT happen: the switch didn't vanish. Anyone who tells you factories "eliminate conditionals" is selling something. They **relocate** conditionals to a place where a conditional is appropriate.

### 3. Simple Factory vs Factory Method — the honest distinction

Here's the thing almost nobody says out loud: **what 95% of engineers call "the Factory pattern" is Simple Factory, which is not a GoF pattern at all.** It's just a good idea. The GoF book's *Factory Method* is a stricter, less common thing.

**Simple Factory** (what you just wrote):
- One class with a `create(type)` method containing a `switch`.
- Adding a product means **editing the factory**. Technically that's still an OCP violation *of the factory*, though you've contained the damage to one file.

**Factory Method** (the real GoF pattern):
- A base class has an **abstract creation method** that subclasses **override**.
- The base class contains an algorithm that *uses* the product but doesn't know which one it is. Subclasses decide.
- Adding a product means **adding a subclass** — you never edit existing code. True OCP.

```javascript
// ---------- GoF FACTORY METHOD ----------
// The CREATOR: contains the workflow, defers the product to a subclass.
class NotificationDispatcher {
  // ↓↓↓ THIS is the "factory method": abstract, overridden by subclasses.
  createNotifier() {
    throw new Error('subclasses must implement createNotifier()');
  }

  // The template workflow. It uses the product without knowing its class.
  async dispatch(recipient, message) {
    const notifier = this.createNotifier();     // ← the deferral happens here
    const start = Date.now();
    try {
      const result = await notifier.send(recipient, message);
      this.onSuccess(result, Date.now() - start);
      return result;
    } catch (err) {
      this.onFailure(err);
      throw err;
    }
  }

  onSuccess(result, ms) { console.log(`sent via ${result.channel} in ${ms}ms`); }
  onFailure(err)        { console.error(`send failed: ${err.message}`); }
}

// CONCRETE CREATORS: each overrides ONE method.
class EmailDispatcher extends NotificationDispatcher {
  constructor(deps) { super(); this.deps = deps; }
  createNotifier() { return new EmailNotifier(this.deps); }   // ← override
}

class SmsDispatcher extends NotificationDispatcher {
  constructor(deps) { super(); this.deps = deps; }
  createNotifier() { return new SmsNotifier(this.deps); }     // ← override
}
```

See the difference? There is **no `switch` anywhere**. Adding WhatsApp = adding `class WhatsAppDispatcher extends NotificationDispatcher` and nothing else. The `dispatch()` workflow — retries, timing, logging — is written **once** in the base class and inherited by everyone.

**When to use which:**

| | Simple Factory | Factory Method (GoF) |
|---|---|---|
| Mechanism | One `create(type)` with a switch | Abstract method overridden per subclass |
| Adding a product | Edit the factory | Add a subclass — touch nothing existing |
| Requires inheritance? | No | **Yes** — that's the whole mechanism |
| Extra shared workflow? | No | Yes — the creator holds a template algorithm |
| How common in real JS | **Very** | Rare (JS prefers passing functions) |

**In an interview, say this:** *"Most people say 'Factory' and mean Simple Factory — a static `create(type)` method. The GoF Factory Method is different: it's an abstract method that subclasses override, so the base class can run an algorithm without knowing the concrete product. In JavaScript I usually reach for Simple Factory or just a factory function, because closures give me the same flexibility without an inheritance hierarchy."*

That answer alone puts you ahead of most candidates.

### 4. Full JavaScript implementation (runnable end to end)

```javascript
// ============================================================
//  notifications.js — complete, runnable: node notifications.js
// ============================================================

/* ---------- PRODUCT INTERFACE ---------- */
class Notifier {
  async send(recipient, message) {
    throw new Error(`${this.constructor.name} must implement send()`);
  }
  get channel() {
    throw new Error(`${this.constructor.name} must implement get channel()`);
  }
}

/* ---------- CONCRETE PRODUCTS ---------- */
class EmailNotifier extends Notifier {
  // `transport` is injected so tests can pass a fake instead of real SMTP.
  constructor({ transport, from = 'noreply@shop.dev' }) {
    super();
    this.transport = transport;
    this.from = from;
  }
  get channel() { return 'email'; }

  async send(recipient, message) {
    if (!recipient.email) throw new Error('recipient has no email address');
    await this.transport.sendMail({
      from: this.from,
      to: recipient.email,
      subject: message.subject,
      text: message.body,
    });
    return { channel: this.channel, to: recipient.email };
  }
}

class SmsNotifier extends Notifier {
  constructor({ client, from }) {
    super();
    this.client = client;
    this.from = from;
  }
  get channel() { return 'sms'; }

  async send(recipient, message) {
    if (!recipient.phone) throw new Error('recipient has no phone number');
    // SMS has a hard 160-char limit — the product owns its own constraints.
    const body = `${message.subject}: ${message.body}`.slice(0, 160);
    await this.client.messages.create({ to: recipient.phone, from: this.from, body });
    return { channel: this.channel, to: recipient.phone };
  }
}

class PushNotifier extends Notifier {
  constructor({ fcm }) { super(); this.fcm = fcm; }
  get channel() { return 'push'; }

  async send(recipient, message) {
    if (!recipient.deviceToken) throw new Error('recipient has no device token');
    await this.fcm.send({
      token: recipient.deviceToken,
      notification: { title: message.subject, body: message.body },
    });
    return { channel: this.channel, to: recipient.deviceToken };
  }
}

/* ---------- THE FACTORY ---------- */
// A registry beats a switch: registering a new channel requires ZERO edits
// to this class, which makes even the FACTORY itself open/closed.
class NotificationFactory {
  #builders = new Map();   // channel → (deps) => Notifier

  register(channel, builder) {
    this.#builders.set(channel, builder);
    return this;                       // chainable
  }

  create(channel) {
    const builder = this.#builders.get(channel);
    if (!builder) {
      throw new Error(
        `Unsupported channel "${channel}". Known: ${[...this.#builders.keys()].join(', ')}`,
      );
    }
    return builder();
  }

  supports(channel) { return this.#builders.has(channel); }
}

/* ---------- THE CLIENT ---------- */
// Note: OrderService knows about Notifier (the interface) and the factory.
// It knows NOTHING about SMTP, Twilio, or FCM.
class OrderService {
  constructor({ notificationFactory }) {
    this.factory = notificationFactory;
  }

  async confirmOrder(order, channels = ['email']) {
    // (pretend: charge card, decrement inventory, persist the order)
    const message = {
      subject: 'Order confirmed',
      body: `Your order ${order.id} for $${(order.totalCents / 100).toFixed(2)} is confirmed.`,
    };

    const results = [];
    for (const channel of channels) {
      const notifier = this.factory.create(channel);   // ← never says `new`
      results.push(await notifier.send(order.user, message));
    }
    return results;
  }
}

/* ---------- COMPOSITION ROOT + DEMO ---------- */
// Fakes stand in for nodemailer / twilio / firebase-admin so this file runs
// with `node notifications.js` and no credentials.
const fakeTransport = { sendMail: async (o) => console.log('  [SMTP]', o.to, '|', o.text) };
const fakeTwilio    = { messages: { create: async (o) => console.log('  [SMS ]', o.to, '|', o.body) } };
const fakeFcm       = { send: async (o) => console.log('  [PUSH]', o.token, '|', o.notification.body) };

async function main() {
  const factory = new NotificationFactory()
    .register('email', () => new EmailNotifier({ transport: fakeTransport }))
    .register('sms',   () => new SmsNotifier({ client: fakeTwilio, from: '+15550000' }))
    .register('push',  () => new PushNotifier({ fcm: fakeFcm }));

  const orders = new OrderService({ notificationFactory: factory });

  const order = {
    id: 'ord_9f21',
    totalCents: 4299,
    user: { email: 'ada@example.com', phone: '+15551234', deviceToken: 'dev_abc' },
  };

  console.log('Confirming order via email + sms + push:');
  const results = await orders.confirmOrder(order, ['email', 'sms', 'push']);
  console.log('Results:', results);

  // --- OCP in action: add a brand-new channel WITHOUT touching any class above.
  class SlackNotifier extends Notifier {
    constructor({ webhook }) { super(); this.webhook = webhook; }
    get channel() { return 'slack'; }
    async send(recipient, message) {
      console.log('  [SLACK]', this.webhook, '|', message.body);
      return { channel: 'slack', to: this.webhook };
    }
  }

  factory.register('slack', () => new SlackNotifier({ webhook: '#orders' }));

  console.log('\nSame OrderService, new channel, zero edits to OrderService:');
  await orders.confirmOrder(order, ['slack']);

  console.log('\nUnknown channel fails loudly:');
  try { factory.create('carrier-pigeon'); }
  catch (e) { console.log('  ' + e.message); }
}

main();
```

Run it and you'll see all four channels fire. **The critical observation:** adding `SlackNotifier` required **zero** modifications to `Notifier`, `NotificationFactory`, or `OrderService`. That is the Open/Closed Principle, made concrete.

### 5. How this enables OCP

Recall from [15 — SOLID: Open/Closed Principle] that code should be **open for extension, closed for modification**: you should add behavior by writing *new* code, not by editing *old, tested, working* code.

Watch the difference in blast radius:

```
WITHOUT factory — add WhatsApp:                WITH factory — add WhatsApp:
┌──────────────────────────┐                   ┌──────────────────────────┐
│ OrderService.js     EDIT │ ← retest          │ OrderService.js       ✓  │ untouched
│ ShippingService.js  EDIT │ ← retest          │ ShippingService.js    ✓  │ untouched
│ RefundService.js    EDIT │ ← retest          │ RefundService.js      ✓  │ untouched
│ PasswordReset.js    EDIT │ ← retest          │ PasswordReset.js      ✓  │ untouched
│                          │                   │ WhatsAppNotifier.js  NEW │ ← only new code
└──────────────────────────┘                   │ main.js  +1 register()   │
   4 files modified                            └──────────────────────────┘
   4 regression risks                             1 new file, 1 wiring line
                                                  0 regression risk
```

The registry-based factory in the code above pushes this even further: even the **factory** doesn't get modified, because registration happens from the outside in `main.js`.

### 6. Real Node.js ecosystem examples

You've been using factories since your first `require`.

**`crypto.createHash(algorithm)`** — the purest example in the standard library:
```javascript
import crypto from 'node:crypto';

const sha = crypto.createHash('sha256');   // ← a factory. You never `new Sha256()`.
const md5 = crypto.createHash('md5');      // ← same call, different concrete class

sha.update('hello').digest('hex');   // both share the Hash interface:
md5.update('hello').digest('hex');   // .update() / .digest()
```
You pass a **string key**; Node returns a concrete `Hash` object backed by an OpenSSL algorithm you never named. Your code depends only on the `Hash` interface. Swap `'sha256'` for `'sha512'` and *nothing else changes*. Textbook Simple Factory.

**`fs.createReadStream(path, opts)`** — returns a `ReadStream`. You get a `Readable` and you `.pipe()` it. Whether it's backed by a file descriptor, a character device, or a FIFO is the factory's problem, not yours.

**Other factories hiding in plain sight:** `http.createServer()`, `net.createConnection()`, `zlib.createGzip()`, `Buffer.from()` (which dispatches on the argument's type — string vs array vs ArrayBuffer — to build the right buffer), `new Intl.NumberFormat(locale)`, and every `mongoose.model(name, schema)` call (which *manufactures a class*).

Notice the naming convention: **Node's factories are called `createX`, not `newX`.** That prefix is a hint you're looking at a factory.

### 7. When NOT to use it / how it's abused

**Don't use a factory when there's only one product.** This is the most common abuse:

```javascript
// ---------- POINTLESS ----------
class UserFactory {
  static create(name, email) { return new User(name, email); }
}
// You have added a class, a file, and an indirection to save typing `new`.
// Just write `new User(name, email)`. YAGNI — see [19 — DRY, KISS, YAGNI].
```
A factory earns its keep when there are **≥2 concrete products** the caller shouldn't choose between directly, **or** when construction is genuinely complicated (parsing config, wiring dependencies, pooling).

**Don't confuse a factory with a Builder.** If the problem is *"my constructor has 15 parameters"*, that's [32 — Builder], not Factory. Factory answers *"which class do I instantiate?"* Builder answers *"how do I assemble one complicated object step by step?"*

**Don't build an AbstractFactoryFactoryProvider.** If your factory takes a factory that produces a factory, you have out-designed the problem. In JavaScript, a **plain function is a perfectly good factory** — you don't need a class:

```javascript
// Perfectly idiomatic. This IS the Factory pattern.
const createNotifier = (channel, deps) => ({
  email: () => new EmailNotifier(deps),
  sms:   () => new SmsNotifier(deps),
}[channel]?.() ?? (() => { throw new Error(`bad channel: ${channel}`); })());
```

**Don't let the factory become a god object.** If `create()` grows to 300 lines of setup for each product, push that complexity *into the products' constructors* or into a Builder. The factory should *choose*, not *assemble*.

### 8. Related patterns and how they differ

This table **is** the interview follow-up. Learn it cold.

| Pattern | Question it answers | Key difference from Factory Method |
|---|---|---|
| **Simple Factory** (not GoF) | "Which of these N classes do I build?" | A `switch` in one method. No inheritance. Most people mean *this* when they say "factory." |
| **Factory Method** (GoF) | "Let subclasses decide which class the base algorithm builds." | Uses **inheritance + overriding**, not a switch. The creator has a workflow that consumes the product. |
| **[31 — Abstract Factory](./31-pattern-abstract-factory.md)** | "Build a whole **family** of related objects that must match." | Factory Method makes **one** product. Abstract Factory makes **several** matching products (`createButton()` + `createCheckbox()`). An Abstract Factory is typically implemented *using* factory methods. |
| **[32 — Builder](./32-pattern-builder.md)** | "How do I construct one complex object step by step?" | Factory picks a *type* in one call; Builder assembles *one* object across many calls, then `.build()`. |
| **[33 — Prototype](./33-pattern-prototype.md)** | "How do I make a new object by cloning an existing one?" | Prototype copies an *instance*; Factory instantiates a *class*. Use when construction is expensive but copying is cheap. |
| **[29 — Singleton](./29-pattern-singleton.md)** | "How do I guarantee one instance?" | Factory returns a **new** object per call (usually); Singleton returns the **same** one. They compose: a factory *can* return a cached singleton. |
| **[42 — Strategy](./42-pattern-strategy.md)** | "How do I swap an *algorithm* at runtime?" | Nearly identical *shape*, different *intent*. Factory is about **creation**; Strategy is about **behavior**. A factory often **produces** strategies. |

**The Factory-vs-Strategy trap:** interviewers love this because the class diagrams look the same (interface + several implementations). The answer: *"Factory's output is an object; Strategy's output is a behavior. If I'm asking 'which object do I need,' that's Factory. If I'm asking 'which algorithm should this object run,' that's Strategy. In practice a factory frequently creates the strategy."*

---

## Visual / Diagram description

### Diagram 1 — Simple Factory (what you'll write 95% of the time)

```
   ┌──────────────────┐
   │   OrderService   │   ← CLIENT: says "give me an 'email' notifier"
   │   (client)       │      knows Notifier + Factory. Knows NOTHING of SMTP.
   └────────┬─────────┘
            │ create('email')
            ▼
   ┌───────────────────────────────┐
   │    NotificationFactory        │   ← FACTORY: the ONE place with the switch
   │───────────────────────────────│
   │ + create(channel) : Notifier  │
   │     'email' → new EmailNotifier
   │     'sms'   → new SmsNotifier │
   │     'push'  → new PushNotifier│
   └───────────────┬───────────────┘
                   │ returns a ...
                   ▼
        ┌────────────────────────┐
        │   «interface»          │   ← PRODUCT: the only type the client sees
        │      Notifier          │
        │────────────────────────│
        │ + send(recip, msg)     │
        │ + get channel()        │
        └───────────▲────────────┘
                    │ extends
      ┌─────────────┼─────────────┬──────────────┐
      │             │             │              │
┌─────┴──────┐ ┌────┴──────┐ ┌────┴──────┐ ┌─────┴──────┐
│EmailNotifier│ │SmsNotifier│ │PushNotifier│ │SlackNotifier│  ← CONCRETE PRODUCTS
│ + send()   │ │ + send()  │ │ + send()  │ │ + send()   │     (add more freely)
└────────────┘ └───────────┘ └───────────┘ └────────────┘
   ▲ nodemailer   ▲ twilio      ▲ FCM         ▲ webhook
```

The client's arrow points at `Notifier` (the abstraction) and at the factory — **never at a concrete product**. That is the whole point.

### Diagram 2 — GoF Factory Method (subclass overriding, no switch)

```
┌────────────────────────────────────────┐
│        NotificationDispatcher          │  ← CREATOR (abstract)
│────────────────────────────────────────│
│ + dispatch(recip, msg)   ← the TEMPLATE algorithm:
│     const n = this.createNotifier();   │     timing, retries, logging.
│     await n.send(...)                  │     Written ONCE. Reused by all.
│                                        │
│ # createNotifier() : Notifier  ← ABSTRACT "factory method"
│        throws — subclass must override │
└──────────────────▲─────────────────────┘
                   │ extends
        ┌──────────┴──────────┐
        │                     │
┌───────┴─────────┐   ┌───────┴─────────┐
│ EmailDispatcher │   │  SmsDispatcher  │   ← CONCRETE CREATORS
│─────────────────│   │─────────────────│
│ createNotifier()│   │ createNotifier()│      each overrides exactly
│  → new Email... │   │  → new Sms...   │      ONE method
└────────┬────────┘   └────────┬────────┘
         │ creates             │ creates
         ▼                     ▼
  ┌─────────────┐       ┌────────────┐
  │EmailNotifier│       │SmsNotifier │     ← PRODUCTS
  └─────────────┘       └────────────┘
```

**Draw both on the whiteboard and the difference is obvious:** Simple Factory has *one* factory class holding a switch. Factory Method has a *hierarchy* of creators, each overriding one method — no switch anywhere.

---

## Real world examples

### 1. Node.js `crypto` — a factory in the standard library

`crypto.createHash(alg)`, `crypto.createCipheriv(alg, key, iv)`, and `crypto.createSign(alg)` are all Simple Factories. You hand in a **string identifier** and get back an object conforming to a known interface (`Hash`, `Cipher`, `Sign`). The concrete implementation lives in OpenSSL and is invisible to you. This is why upgrading your app's hashing from SHA-256 to SHA-512 is a one-character config change and not a refactor: your code depends on the `Hash` interface, not on a `Sha256` class.

### 2. Node.js `fs` streams

`fs.createReadStream()` and `fs.createWriteStream()` manufacture `ReadStream`/`WriteStream` objects. The factory handles opening the file descriptor, choosing buffer sizes, and wiring up the `Readable`/`Writable` machinery. Your code receives an object it can `.pipe()`. The same interface is produced by `zlib.createGzip()`, `net.createConnection()`, and `process.stdin` — which is exactly why `fs.createReadStream(f).pipe(zlib.createGzip()).pipe(res)` composes: every factory in the chain returns something honoring the stream contract.

### 3. Database drivers and ORMs

`mongoose.model('User', schema)` is a factory that manufactures **a class**, not just an object. Knex's `knex({ client: 'pg' })` is a factory choosing between Postgres, MySQL, and SQLite dialect implementations behind one query-builder interface — the entire premise of "swap your database without rewriting queries" rests on that factory. Similarly, cloud SDKs (`new S3Client(config)`, `PubSub.topic()`) expose factory functions so the client code never names a concrete transport class.

---

## Trade-offs

| Benefit | What it buys you |
|---|---|
| **Decouples client from concrete classes** | `OrderService` doesn't know Twilio exists — swap vendors freely ([18 — DIP]) |
| **Creation logic in one place** | One `switch`, not six copies drifting apart ([19 — DRY]) |
| **Enables OCP** | Add a product = add a class + one registration line; touch no existing code |
| **Testable** | Inject a fake factory returning fake products; no SMTP creds in unit tests |
| **Encapsulates complex construction** | Config parsing, pooling, retry wrappers hide inside the factory |

| Cost | What it takes away |
|---|---|
| **More classes/files** | 1 interface + N products + 1 factory instead of one `if` block |
| **Indirection** | "Where is this object actually created?" — one more hop when debugging |
| **The switch doesn't disappear** | It relocates. A Simple Factory still gets edited for each new product |
| **Runtime failure, not compile-time** | `create('emial')` (typo) throws at runtime — a plain `new EmailNotifier()` would fail at import/lint time |
| **Overkill for one product** | A factory over a single class is pure ceremony |

**The sweet spot:** Reach for a factory the moment you have **two or more interchangeable implementations** behind one interface *and* the caller shouldn't care which. Use **Simple Factory** (or a plain factory function) by default. Escalate to **GoF Factory Method** only when the creator also owns a shared workflow the subclasses should inherit. Escalate to **Abstract Factory** only when you need *families* of products that must match — see [31](./31-pattern-abstract-factory.md).

**Rule of thumb:** If you see the same `switch (type) { case ...: return new ... }` in more than one file, you needed a factory yesterday. If you see a factory that only ever returns one class, delete it.

---

## Common interview questions on this topic

### Q1: "What's the difference between Simple Factory, Factory Method, and Abstract Factory?"
**Hint:** The single most common follow-up. **Simple Factory:** one method with a `switch` returning one of N products — not actually a GoF pattern, but what most people mean. **Factory Method:** an *abstract method* on a creator class that subclasses *override*; no switch, uses inheritance, and the creator holds a shared algorithm that consumes the product. **Abstract Factory:** an interface with *several* creation methods that together produce a matching **family** (`createButton()` + `createCheckbox()` from `MacFactory`). One product vs one product-via-subclass vs a family of products.

### Q2: "How does a factory help with the Open/Closed Principle?"
**Hint:** Without it, adding a product means editing every `switch` in every caller — modification, with regression risk in tested code. With a factory, callers depend only on the product *interface*, so adding a product is adding a **new class** plus (at most) one line in the factory. With a **registry-based** factory (`factory.register('slack', builder)`), even the factory stays closed — registration happens in the composition root. Show the blast-radius diagram.

### Q3: "Give me a factory from the Node standard library and explain why it's one."
**Hint:** `crypto.createHash('sha256')`. You pass a **string key**, not a class. It returns an object satisfying the `Hash` interface (`.update()`, `.digest()`) whose concrete implementation you never name. Your code is decoupled from the algorithm — changing to `'sha512'` changes nothing else. Bonus: point out Node's `createX` naming convention marks factories, and cite `fs.createReadStream`, `http.createServer`, `zlib.createGzip`.

### Q4: "Factory vs Strategy — the diagrams look identical. What's the difference?"
**Hint:** Intent, not structure. **Factory is about creation** — "which object do I need?" Its output is an object, and once you have it the factory is done. **Strategy is about behavior** — "which algorithm should this object use?" — and the strategy stays plugged in, swappable at runtime. They compose constantly: a factory is often the thing that *creates* the strategy you inject. If someone asks you to "pick a pricing algorithm at runtime," that's Strategy; the thing that turns the string `'surge'` into a `SurgePricing` object is a Factory.

### Q5: "When would you NOT use a factory?"
**Hint:** When there's only one concrete product — `UserFactory.create(...)` wrapping `new User(...)` is pure ceremony (YAGNI). When construction is trivial and the caller legitimately knows the type it wants. When the real problem is a 15-parameter constructor — that's **Builder**, not Factory. And in JS specifically, remember a **plain function is a valid factory** — you rarely need a class with a static method to earn the pattern's benefits.

---

## Practice exercise

### Kill the Switch — Refactor a Payment Processor

You inherit this (write it out first, exactly as-is, in `payments.js` — feel the badness):

```javascript
class CheckoutService {
  async pay(order, method) {
    if (method === 'card') {
      const stripe = require('stripe')(process.env.STRIPE_KEY);
      return stripe.charges.create({ amount: order.total, currency: 'usd', source: order.token });
    } else if (method === 'paypal') {
      const pp = new PayPalClient(process.env.PP_ID, process.env.PP_SECRET);
      return pp.execute({ total: order.total / 100, currency: 'USD' });
    } else if (method === 'upi') {
      const upi = new RazorpayClient(process.env.RZP_KEY);
      return upi.createPayment({ amount: order.total, vpa: order.vpa });
    }
    throw new Error('bad method');
  }
}
```

**Your task (~30-40 min), producing one runnable file:**

1. **Define the product interface.** A `PaymentProcessor` base class with `async charge(order)` and `get method()`, both throwing `'not implemented'`.
2. **Write three concrete products:** `CardProcessor`, `PayPalProcessor`, `UpiProcessor`. Each takes its SDK client as a **constructor parameter** (inject it — do not `require` inside the class). Each returns a normalized `{ method, transactionId, amountCents, status }`. Notice how each product now owns its own vendor quirks (PayPal wants dollars, Stripe wants cents).
3. **Write a registry-based `PaymentFactory`** with `register(method, builder)` and `create(method)`. Unknown methods must throw an error that *lists the supported methods*.
4. **Rewrite `CheckoutService`** so it takes the factory in its constructor and its `pay()` method is **under 6 lines** with no `if`/`switch` and no vendor names.
5. **Prove OCP:** at the bottom of the file, add a `CryptoProcessor` and register it — **without editing a single line above it** — then charge an order with it.
6. **Prove testability:** write a `testCheckout()` function that constructs `CheckoutService` with a factory whose products are fakes (`{ charge: async () => ({ status: 'ok' }) }`). It must pass with zero network calls and zero API keys.

**What to produce:** `payments.js`, runnable via `node payments.js`, printing a successful charge on each of the four methods plus the passing fake-based test. If step 5 forced you to edit any earlier code, your factory isn't a registry — go back and fix it.

---

## Quick reference cheat sheet

- **Intent:** Create objects without the caller naming the concrete class. The caller asks *what* it wants; the factory decides *which*.
- **Participants:** **Product** (interface) → **Concrete Products** (implementations) → **Factory/Creator** (builds them) → **Client** (asks, never `new`s).
- **Simple Factory:** one `create(type)` with a `switch`. Not GoF. **This is what people mean 95% of the time.** Default to it.
- **GoF Factory Method:** an **abstract method subclasses override**. No switch — inheritance is the mechanism. The creator owns a workflow that consumes the product.
- **The switch never disappears** — it *relocates* to the one class whose job is creation. That's still a huge win (one copy, not six).
- **Registry beats switch:** `factory.register(key, builder)` keeps even the factory closed for modification.
- **Enables OCP** ([15](./15-solid-open-closed.md)): adding a product = new class + one registration line; existing tested code untouched.
- **Enables DIP** ([18](./18-solid-dependency-inversion.md)): client depends on the Product interface, not on Twilio/SMTP/FCM.
- **In Node:** `crypto.createHash(alg)`, `fs.createReadStream()`, `http.createServer()`, `zlib.createGzip()`, `mongoose.model()`. The `createX` prefix is the tell.
- **In JS, a plain function is a valid factory.** You don't need a class to earn the benefits.
- **Don't** use it for a single product (YAGNI), and don't confuse it with **Builder** (complex step-by-step construction) or **Strategy** (swappable *behavior*, not *creation*).
- **vs Abstract Factory:** Factory Method makes **one** product. Abstract Factory makes a **matching family** of products. That's the whole distinction — see [31](./31-pattern-abstract-factory.md).
- **Interview line:** "Most people say Factory and mean Simple Factory. The GoF Factory Method is an overridable method on a creator hierarchy — in JS I'd usually just pass a function."

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [29 — Singleton Pattern](./29-pattern-singleton.md) — controls *how many* objects exist; Factory controls *which kind* |
| **Next** | [31 — Abstract Factory Pattern](./31-pattern-abstract-factory.md) — when one product isn't enough and you need a matching *family* |
| **Related** | [15 — SOLID: Open/Closed Principle](./15-solid-open-closed.md) — the principle a factory exists to serve |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — same class diagram, different intent; factories usually *create* strategies |
