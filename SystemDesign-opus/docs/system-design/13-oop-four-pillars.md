# 13 — Object Oriented Programming — The 4 Pillars for Design
## Category: LLD Fundamentals

---

## What is this?

Object Oriented Programming (OOP) is a way of organizing code around **things** (objects) instead of around **steps** (functions in a script). The "four pillars" — Encapsulation, Abstraction, Inheritance, Polymorphism — are the four tools OOP gives you to keep a growing codebase from turning into mud.

Think of a **car**. You drive it with a steering wheel, a pedal, and a gear stick. You don't reach into the engine to adjust fuel injection timing. The car **hides** its internals (encapsulation), gives you a **simple contract** to operate it (abstraction), a Tesla and a Toyota are both **kinds of** car (inheritance), and you can drive either one without relearning how to drive (polymorphism).

---

## Why does it matter?

Most people learn the four pillars as **syntax trivia** — "encapsulation means make fields private." That's the useless version. The useful version is: **each pillar is a lever you pull to control change.** Software doesn't die from being slow. It dies from being impossible to change safely.

**What breaks without them:**
- No encapsulation → any file in your codebase can set `order.status = 'SHIPPED'` on an unpaid order. You now have a data-corruption bug with 40 possible sources.
- No abstraction → your business logic imports the Stripe SDK directly. You can't test it without hitting Stripe, and you can't add PayPal without rewriting the logic.
- No inheritance / no shared contract → the same 30 lines of validation are copy-pasted into 9 files. You fix a bug in 8 of them.
- No polymorphism → a 400-line `switch (type)` statement that every new feature must edit, and every edit risks breaking the other 12 cases.

**In interviews:** LLD rounds ("design a parking lot", "design an elevator") are graded almost entirely on whether you use these four correctly. The interviewer will deliberately say *"now add a new vehicle type"* to see if your design absorbs the change or requires surgery.

**At work:** Every single design pattern in topics 29–51 is built out of these four pillars. Strategy is polymorphism. Factory is abstraction + polymorphism. Decorator is composition + polymorphism. Learn the pillars and the patterns become obvious instead of memorized.

---

## The core idea — explained simply

### The Hospital Analogy

Walk into a hospital. Watch how it is organized.

**Encapsulation — the pharmacy counter.**
Drugs are behind a locked counter. You cannot walk in and grab morphine. You hand a prescription to the pharmacist, and *she* decides whether to give it to you. The rule ("valid prescription required") lives **next to the thing it protects**. If the rule lived in a policy handbook in a different building, someone would eventually ignore it.

**Abstraction — the job title "surgeon".**
When the hospital hires a surgeon, the contract says: *can operate, can diagnose, can sign off on discharge.* The contract does **not** say which medical school, which technique, which hand they hold the scalpel in. The ER can schedule "a surgeon" without knowing anything else.

**Inheritance — the medical hierarchy.**
`Doctor` → `Surgeon` → `CardiacSurgeon`. Everything true of a Doctor is true of a Cardiac Surgeon (they all have a license, they all have rounds). A Cardiac Surgeon just *adds* more. This is genuinely an **is-a** relationship — a cardiac surgeon **is a** doctor.

**Polymorphism — "page the on-call doctor."**
The nurse calls *"on-call doctor to room 4."* Whoever shows up — surgeon, pediatrician, resident — knows how to `respond()`. The nurse does not branch on the doctor's specialty. **Same message, different behavior, no if-statements at the call site.**

### Mapping the analogy back

| Hospital thing | OOP pillar | What it buys you |
|---|---|---|
| Locked pharmacy counter | **Encapsulation** | Invalid state becomes *unrepresentable*, not just discouraged |
| The job title "surgeon" | **Abstraction** | Callers depend on a contract, not a concrete implementation |
| Doctor → Surgeon → CardiacSurgeon | **Inheritance** | Shared behavior defined once, specialized where needed |
| "Page the on-call doctor" | **Polymorphism** | New doctor types are *added*, not *if-else'd in* |

The four pillars are not four unrelated facts. They're a chain:
**Encapsulation** hides the internals → so you can define an **Abstraction** (a contract) → which many types can satisfy via **Inheritance** (or duck typing) → letting callers use them interchangeably via **Polymorphism**.

---

## Key concepts inside this topic

### 1. Encapsulation — bundle data with the rules that protect it

**Definition:** Keep an object's data private, and expose only methods that enforce the object's rules. The object is responsible for never being in an invalid state.

**The design problem it solves:** *Invariants* — facts that must always be true (a bank balance is never negative; an order can't ship before it's paid). If data is public, the invariant is enforced *by convention*, in every caller. Conventions lose.

**BAD — a public data bag:**

```javascript
// Anyone, anywhere, can put this object into an impossible state.
class BankAccount {
  constructor(ownerId) {
    this.ownerId = ownerId;
    this.balance = 0;
    this.transactions = [];
  }
}

const acct = new BankAccount('u1');

// Somewhere in a controller, 6 months later, at 2am:
acct.balance -= 500;        // balance is now -500. No error. No log entry.
acct.balance = NaN;         // sure, why not
acct.transactions.length = 0; // audit trail: gone
```

There is no single place to fix this bug. The rule "balance must never go negative" is not written anywhere in the code — it lives in someone's head.

**GOOD — private fields (`#`) + methods that enforce the rules:**

```javascript
// `#field` is a real, hard-private field in modern JS (Node 12+).
// It is not accessible outside the class body — not even via bracket notation.
class BankAccount {
  #balance = 0;
  #transactions = [];

  constructor(ownerId) {
    this.ownerId = ownerId; // intentionally public: it never changes and is safe to read
  }

  // Read access is exposed, but as a COPY — callers can't mutate our internals.
  get balance() {
    return this.#balance;
  }

  get history() {
    return [...this.#transactions]; // defensive copy: returning the array itself would leak it
  }

  deposit(amount) {
    this.#assertValidAmount(amount);
    this.#balance += amount;
    this.#record('DEPOSIT', amount);
  }

  withdraw(amount) {
    this.#assertValidAmount(amount);
    // The invariant lives HERE, next to the data it protects. One place. Always runs.
    if (amount > this.#balance) {
      throw new Error(`Insufficient funds: balance ${this.#balance}, requested ${amount}`);
    }
    this.#balance -= amount;
    this.#record('WITHDRAW', amount);
  }

  #assertValidAmount(amount) {
    if (!Number.isFinite(amount) || amount <= 0) {
      throw new Error(`Amount must be a positive finite number, got: ${amount}`);
    }
  }

  #record(type, amount) {
    this.#transactions.push({ type, amount, at: new Date(), balanceAfter: this.#balance });
  }
}

const acct = new BankAccount('u1');
acct.deposit(1000);
acct.withdraw(300);
console.log(acct.balance);      // 700
// acct.#balance = 999;         // SyntaxError — the field does not exist outside the class
acct.balance = 999;             // silently ignored (no setter) — and in strict mode, throws
acct.history.push({ fake: 1 }); // mutates the copy, not our real ledger
console.log(acct.balance);      // still 700
```

**What goes wrong without it:** Bugs stop being local. "Balance went negative" becomes an investigation across the whole codebase instead of a 5-line read of one class.

> **JS note:** `#private` fields are true privacy, enforced by the language. The older conventions — `_balance` (a naming hint only) and closure-based privacy — still appear in older code. `_balance` protects nothing; it's a polite request. Prefer `#`.

---

### 2. Abstraction — expose *what*, hide *how*

**Definition:** Define a stable contract (the *what*) that callers depend on, so the implementation (the *how*) can change freely underneath.

**The design problem it solves:** Coupling to details. If your `OrderService` knows that payments go through Stripe's `charges.create()` API, then Stripe is now permanently welded into your business logic.

Encapsulation hides data. **Abstraction hides decisions.**

**BAD — business logic welded to a vendor:**

```javascript
import Stripe from 'stripe';

class OrderService {
  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_KEY); // hard dependency, created internally
  }

  async checkout(order) {
    // Vendor-specific shapes leak straight into our domain logic:
    const charge = await this.stripe.charges.create({
      amount: order.total * 100,   // Stripe wants cents. Why does OrderService know that?
      currency: 'usd',
      source: order.stripeToken,   // our Order model now has a Stripe-shaped field
    });
    if (charge.status !== 'succeeded') throw new Error('Payment failed');
    order.markPaid(charge.id);
  }
}
```

To unit-test `checkout`, you must mock the Stripe SDK. To add PayPal, you edit `checkout`. To add "pay with store credit," you edit `checkout` again.

**GOOD — depend on an abstract contract:**

```javascript
/**
 * The ABSTRACTION. In Java this would be `interface PaymentGateway`.
 * JS has no `interface` keyword, so we use an abstract base class:
 * it documents the contract AND fails loudly if a subclass forgets a method.
 */
class PaymentGateway {
  /**
   * @param {{ amountCents: number, currency: string, orderId: string }} request
   * @returns {Promise<{ ok: boolean, reference: string }>}
   */
  async charge(request) {
    throw new Error(`${this.constructor.name} must implement charge()`);
  }

  async refund(reference, amountCents) {
    throw new Error(`${this.constructor.name} must implement refund()`);
  }
}

class StripeGateway extends PaymentGateway {
  constructor(stripeClient) { super(); this.stripe = stripeClient; }

  async charge({ amountCents, currency, orderId }) {
    const c = await this.stripe.charges.create({
      amount: amountCents, currency, metadata: { orderId },
    });
    return { ok: c.status === 'succeeded', reference: c.id };
  }

  async refund(reference, amountCents) {
    await this.stripe.refunds.create({ charge: reference, amount: amountCents });
  }
}

class StoreCreditGateway extends PaymentGateway {
  constructor(creditLedger) { super(); this.ledger = creditLedger; }

  async charge({ amountCents, orderId }) {
    const ok = await this.ledger.debit(orderId, amountCents);
    return { ok, reference: `credit:${orderId}` };
  }

  async refund(reference, amountCents) {
    await this.ledger.credit(reference, amountCents);
  }
}

// The business logic now knows NOTHING about Stripe.
class OrderService {
  // Dependency is INJECTED, not constructed. (This is topic 18 — Dependency Inversion.)
  constructor(gateway) {
    this.gateway = gateway;
  }

  async checkout(order) {
    const result = await this.gateway.charge({
      amountCents: order.totalCents,
      currency: order.currency,
      orderId: order.id,
    });
    if (!result.ok) throw new Error('Payment declined');
    order.markPaid(result.reference);
    return order;
  }
}
```

Now: unit-testing `checkout` needs a 5-line fake gateway, not a mocked SDK. Adding PayPal means writing one new class and changing zero lines of `OrderService`.

**What goes wrong without it:** Every vendor change becomes a refactor. Tests need the network. The word "Stripe" appears in 40 files.

---

### 3. Inheritance — share behavior down an *is-a* hierarchy

**Definition:** A subclass automatically gets the fields and methods of its parent, and may add to or override them. `class Dog extends Animal`.

**The design problem it solves:** Duplication of genuinely shared behavior, and expressing a real "is-a" taxonomy.

**BAD — copy-paste "inheritance":**

```javascript
class EmailNotification {
  constructor(recipient, body) { this.recipient = recipient; this.body = body; this.sentAt = null; }
  validate() { if (!this.recipient) throw new Error('No recipient'); }
  async send() {
    this.validate();
    await emailClient.send(this.recipient, this.body);
    this.sentAt = new Date();
    logger.info(`sent to ${this.recipient}`);
  }
}

class SmsNotification {
  constructor(recipient, body) { this.recipient = recipient; this.body = body; this.sentAt = null; }
  validate() { if (!this.recipient) throw new Error('No recipient'); }  // duplicated
  async send() {
    this.validate();                                                     // duplicated
    await smsClient.send(this.recipient, this.body);                     // the ONLY real difference
    this.sentAt = new Date();                                            // duplicated
    logger.info(`sent to ${this.recipient}`);                            // duplicated
  }
}
```

Add "record a delivery metric" and you must edit both. Miss one, and you have a silent gap in your dashboards.

**GOOD — a base class owns the skeleton, subclasses fill the hole:**

```javascript
/**
 * This is the Template Method pattern (topic 45) — the base class
 * fixes the ALGORITHM, subclasses supply the varying STEP.
 */
class Notification {
  #sentAt = null;

  constructor(recipient, body) {
    if (new.target === Notification) {
      throw new Error('Notification is abstract — instantiate a subclass');
    }
    this.recipient = recipient;
    this.body = body;
  }

  get sentAt() { return this.#sentAt; }

  validate() {
    if (!this.recipient) throw new Error('No recipient');
    if (!this.body?.trim()) throw new Error('Empty body');
  }

  // The fixed skeleton. Defined ONCE. Changing it changes every notification type.
  async send() {
    this.validate();
    await this.deliver();   // ← the one varying step
    this.#sentAt = new Date();
    metrics.increment(`notification.sent.${this.channel}`);
    logger.info(`${this.channel} sent to ${this.recipient}`);
  }

  get channel() { throw new Error('Subclass must define channel'); }
  async deliver() { throw new Error('Subclass must implement deliver()'); }
}

class EmailNotification extends Notification {
  get channel() { return 'email'; }

  validate() {
    super.validate();                                   // extend, don't replace
    if (!this.recipient.includes('@')) throw new Error('Invalid email');
  }

  async deliver() { await emailClient.send(this.recipient, this.body); }
}

class SmsNotification extends Notification {
  get channel() { return 'sms'; }

  validate() {
    super.validate();
    if (this.body.length > 160) throw new Error('SMS too long');
  }

  async deliver() { await smsClient.send(this.recipient, this.body); }
}
```

**The trap:** inheritance is the *most abused* pillar. Use it only when the subclass genuinely **is a** kind of the parent, and every parent method makes sense on the child. If you find yourself writing `throw new Error('not supported')` in an override, your hierarchy is wrong (that violates the Liskov Substitution Principle — topic 16), and you should be using **composition** instead (topic 26).

**What goes wrong without it:** Duplicated skeletons, and fixes that land in some copies but not all.

---

### 4. Polymorphism — one message, many behaviors

**Definition:** Different objects respond to the *same* method call in their *own* way, and the caller doesn't need to know or care which one it's holding.

**The design problem it solves:** The growing `if/else` or `switch (type)` — the single biggest source of "I changed one thing and broke three others."

**BAD — the switch that everyone must edit:**

```javascript
class ShippingCalculator {
  calculate(order) {
    switch (order.shippingType) {
      case 'STANDARD':
        return 5.00 + order.weightKg * 0.5;
      case 'EXPRESS':
        return 15.00 + order.weightKg * 1.2;
      case 'OVERNIGHT':
        return 30.00 + order.weightKg * 2.0;
      default:
        throw new Error(`Unknown shipping type: ${order.shippingType}`);
    }
  }
}
// Add DRONE delivery → edit this file.
// Add INTERNATIONAL → edit this file.
// And there is almost certainly a *second* switch on shippingType in the UI,
// and a *third* in the ETA estimator, and they will drift apart.
```

**GOOD — polymorphic dispatch replaces the switch:**

```javascript
class ShippingMethod {
  get code()  { throw new Error('must implement code'); }
  get label() { throw new Error('must implement label'); }
  costFor(order) { throw new Error('must implement costFor()'); }
  etaDays()   { throw new Error('must implement etaDays()'); }
}

class StandardShipping extends ShippingMethod {
  get code()  { return 'STANDARD'; }
  get label() { return 'Standard (5-7 days)'; }
  costFor(order) { return 5.00 + order.weightKg * 0.5; }
  etaDays() { return 6; }
}

class ExpressShipping extends ShippingMethod {
  get code()  { return 'EXPRESS'; }
  get label() { return 'Express (2 days)'; }
  costFor(order) { return 15.00 + order.weightKg * 1.2; }
  etaDays() { return 2; }
}

class OvernightShipping extends ShippingMethod {
  get code()  { return 'OVERNIGHT'; }
  get label() { return 'Overnight'; }
  costFor(order) { return 30.00 + order.weightKg * 2.0; }
  etaDays() { return 1; }
}

// The caller has ZERO branches. This function never changes again.
function quote(order, method) {
  return {
    label: method.label,
    cost: method.costFor(order),
    arrivesInDays: method.etaDays(),
  };
}

// Adding drone delivery = ONE new file. Nothing above is touched.
class DroneShipping extends ShippingMethod {
  get code()  { return 'DRONE'; }
  get label() { return 'Drone (same day)'; }
  costFor(order) { return 12.00 + order.weightKg * 3.0; }
  etaDays() { return 0; }
}

console.log(quote({ weightKg: 2 }, new ExpressShipping()));
// { label: 'Express (2 days)', cost: 17.4, arrivesInDays: 2 }
```

Notice that all four cost formulas, labels, and ETAs for *one* shipping type now live **together in one class**, instead of being smeared across three switch statements. That's cohesion (topic 24) arriving for free.

**What goes wrong without it:** Every new case is a modification to existing, working, tested code — the exact thing the Open/Closed Principle (topic 15) tells you to avoid.

---

### 5. How JavaScript is different from Java/C++ (be honest about this)

Interviewers who come from a Java background sometimes ask JS candidates OOP questions that don't map cleanly. Know the differences.

| Concept | Java / C++ | JavaScript |
|---|---|---|
| **Object model** | Class-based. Classes are a real construct in the type system. | **Prototype-based.** `class` is syntax sugar over prototypes and constructor functions. `class Dog extends Animal` sets `Dog.prototype.__proto__ = Animal.prototype`. |
| **Interfaces** | First-class (`interface Payable`), compiler-enforced. | **None.** No `interface` keyword, no compile-time check. You simulate them with abstract base classes that `throw`, or with TypeScript. |
| **Private** | `private` keyword, compile-time enforced. | `#field` — real runtime privacy. `_field` is convention only. |
| **Abstract classes** | `abstract class`, compiler blocks instantiation. | Simulate with `if (new.target === Base) throw ...`. |
| **Polymorphism** | Requires a shared type (interface or base class). | **Duck typing** — if the object has a `.charge()` method, it works. No shared ancestor required. |
| **Method overloading** | Yes (same name, different signatures). | **No.** One function per name; branch on `arguments.length` or use an options object. |
| **Multiple inheritance** | Interfaces (Java) / multiple base classes (C++). | Single prototype chain only. Use **mixins** (functions that add methods to a class). |

**The most important practical difference: duck typing.**

In Java, to pass an object where a `PaymentGateway` is expected, it must `implement PaymentGateway`. In JS, it just has to *have the methods*:

```javascript
// This is NOT a subclass of PaymentGateway. It doesn't extend anything.
// OrderService will happily use it, because it "quacks like" a gateway.
const fakeGateway = {
  async charge({ amountCents }) {
    return { ok: amountCents < 100_000, reference: 'fake-ref-1' };
  },
  async refund() {},
};

const svc = new OrderService(fakeGateway);
await svc.checkout({ id: 'o1', totalCents: 4500, currency: 'usd', markPaid() {} });
// works — no inheritance involved at all
```

This is why testing in JS is so easy: **your test double doesn't need to inherit from anything.** It's also why JS people often say "just use plain objects and functions."

**So when should you still write the abstract base class in JS?** When the contract is important enough to be worth documenting and enforcing at runtime. `class PaymentGateway { charge() { throw ... } }` gives you: a place to write the JSDoc for the contract, a searchable name (`grep 'extends PaymentGateway'` finds all implementations), and a loud error if someone forgets a method. Those are real benefits. But recognize that you're *simulating* an interface, not declaring one.

**The idiomatic-JS alternatives:**

```javascript
// 1) Plain object / closure as a "strategy" — no class needed at all.
const standardShipping = {
  code: 'STANDARD',
  costFor: (order) => 5.00 + order.weightKg * 0.5,
};

// 2) Mixins for "multiple inheritance" of behavior.
const Timestamped = (Base) => class extends Base {
  #createdAt = new Date();
  get createdAt() { return this.#createdAt; }
};
const Serializable = (Base) => class extends Base {
  toJSON() { return { ...this, type: this.constructor.name }; }
};
class AuditLog extends Timestamped(Serializable(Object)) {}

// 3) A runtime contract check, when you need real enforcement across duck types.
function assertImplements(obj, methods, name) {
  for (const m of methods) {
    if (typeof obj[m] !== 'function') {
      throw new TypeError(`${name}: missing required method "${m}"`);
    }
  }
}
assertImplements(fakeGateway, ['charge', 'refund'], 'PaymentGateway');
```

Rule for JS/Node: **use `class` + `extends` for genuine is-a hierarchies with shared state; use plain objects/closures for pluggable behavior; use TypeScript when you want the compiler to actually check your interfaces.**

---

### 6. The pillars compose — a worked mini-design

Here is all four working together in ~30 lines. This is the shape an interviewer wants to see.

```javascript
// ABSTRACTION: the contract discount rules must satisfy
class DiscountRule {
  get name() { throw new Error('must implement name'); }
  appliesTo(cart) { throw new Error('must implement appliesTo()'); }
  amountOff(cart) { throw new Error('must implement amountOff()'); }
}

// INHERITANCE: concrete rules share the contract and the base's helpers
class PercentOffRule extends DiscountRule {
  constructor(pct, minTotal) { super(); this.pct = pct; this.minTotal = minTotal; }
  get name() { return `${this.pct}% off orders over ${this.minTotal}`; }
  appliesTo(cart) { return cart.subtotal >= this.minTotal; }
  amountOff(cart) { return cart.subtotal * (this.pct / 100); }
}

class BuyTwoGetOneRule extends DiscountRule {
  get name() { return 'Buy 2 get 1 free (cheapest item)'; }
  appliesTo(cart) { return cart.items.length >= 3; }
  amountOff(cart) { return Math.min(...cart.items.map(i => i.price)); }
}

// ENCAPSULATION: Cart guards its own invariants
class Cart {
  #items = [];
  addItem(item) {
    if (item.price < 0) throw new Error('Negative price');
    this.#items.push(item);
  }
  get items() { return [...this.#items]; }
  get subtotal() { return this.#items.reduce((s, i) => s + i.price, 0); }
}

// POLYMORPHISM: zero branching on rule type
function bestDiscount(cart, rules) {
  return rules
    .filter(r => r.appliesTo(cart))
    .map(r => ({ name: r.name, off: r.amountOff(cart) }))
    .reduce((best, d) => (d.off > best.off ? d : best), { name: 'none', off: 0 });
}

const cart = new Cart();
[30, 20, 45].forEach(price => cart.addItem({ price }));
console.log(bestDiscount(cart, [new PercentOffRule(10, 50), new BuyTwoGetOneRule()]));
// { name: 'Buy 2 get 1 free (cheapest item)', off: 20 }  → 20 beats 10% of 95 (9.5)
```

---

## Visual / Diagram description

### Diagram 1: The four pillars as layers of protection

```
                    ┌──────────────────────────────────┐
     CALLER  ─────▶ │        POLYMORPHISM              │
   (knows only      │  "call charge() — I don't care    │
    the contract)   │   which implementation you are"   │
                    └───────────────┬──────────────────┘
                                    │ dispatches to
                    ┌───────────────▼──────────────────┐
                    │        ABSTRACTION               │
                    │  PaymentGateway { charge(req) }   │
                    │  the CONTRACT — the "what"        │
                    └───────┬──────────────┬───────────┘
                            │ extends      │ extends
              ┌─────────────▼───┐    ┌─────▼─────────────┐
              │ StripeGateway   │    │ StoreCreditGateway │   ◀── INHERITANCE
              │                 │    │                    │
              │ ┌─────────────┐ │    │ ┌────────────────┐ │
              │ │#apiKey      │ │    │ │#ledger         │ │   ◀── ENCAPSULATION
              │ │#retryCount  │ │    │ │#balanceCache   │ │       (private, hidden,
              │ └─────────────┘ │    │ └────────────────┘ │        invariants enforced)
              │ charge() {...}  │    │ charge() {...}     │
              └─────────────────┘    └────────────────────┘
```

Read it top-down: the caller sees only the contract. The contract has multiple implementations. Each implementation hides its own private state. Nothing above a line knows anything about what's below it. **That ignorance is the whole point** — it's what lets you swap, test, and extend.

### Diagram 2: Class diagram of the shipping example

```
              ┌─────────────────────────────────┐
              │      ShippingMethod             │   «abstract»
              ├─────────────────────────────────┤
              │ + code : string          {get}  │
              │ + label : string         {get}  │
              ├─────────────────────────────────┤
              │ + costFor(order) : number       │   ← throws if not overridden
              │ + etaDays() : number            │
              └────────────────△────────────────┘
                               │ extends
        ┌──────────────────────┼──────────────────────┬────────────────────┐
        │                      │                      │                    │
┌───────┴────────┐  ┌──────────┴───────┐  ┌───────────┴──────┐  ┌──────────┴───────┐
│StandardShipping│  │ ExpressShipping  │  │OvernightShipping │  │  DroneShipping   │
├────────────────┤  ├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│costFor():      │  │costFor():        │  │costFor():        │  │costFor():        │
│ 5 + kg*0.5     │  │ 15 + kg*1.2      │  │ 30 + kg*2.0      │  │ 12 + kg*3.0      │
│etaDays(): 6    │  │etaDays(): 2      │  │etaDays(): 1      │  │etaDays(): 0      │
└────────────────┘  └──────────────────┘  └──────────────────┘  └──────────────────┘
                                                                   ▲
                                                                   │ NEW: added without
                                                                   │ editing any box
                                                                   │ to its left.

┌───────────────────────────────────────┐
│   quote(order, method)                │  uses ShippingMethod
│   ────────────────────────────────    │  ────────────────────▶  (depends on the
│   return method.costFor(order)        │                          ABSTRACTION only,
│   // zero if/switch statements        │                          never on a subclass)
└───────────────────────────────────────┘
```

The `△` arrow means "inherits from" (UML: hollow triangle pointing at the parent). The key property to notice: `quote()` has an arrow to the **abstract box**, not to any concrete box. That single fact is what makes `DroneShipping` addable without touching `quote()`.

---

## Real world examples

### 1. Node.js Streams — polymorphism in the standard library

Every stream in Node implements the same small contract: a `Readable` has `.pipe()`, `.read()`, and emits `'data'` / `'end'`. A `Writable` has `.write()` and `.end()`.

Because of that shared contract, `fs.createReadStream()` (a file), `http.IncomingMessage` (a network socket), `zlib.createGzip()` (a transform), and `process.stdout` (a terminal) are all interchangeable:

```javascript
import { pipeline } from 'node:stream/promises';
import fs from 'node:fs';
import zlib from 'node:zlib';

await pipeline(
  fs.createReadStream('big.log'),  // could be an HTTP response instead
  zlib.createGzip(),
  fs.createWriteStream('big.log.gz') // could be an S3 upload stream instead
);
```

`pipeline()` contains no `if (source instanceof FileStream)`. It calls `.pipe()` and lets polymorphism do the work. That's why the ecosystem has thousands of custom streams that work with core Node without core Node knowing they exist.

### 2. Express middleware — abstraction over a uniform contract

Every Express middleware is a function with the signature `(req, res, next)`. That signature *is* the abstraction. `express.json()`, `cors()`, `helmet()`, your own `requireAuth`, and a third-party logger all satisfy it, so `app.use()` can accept any of them without special-casing:

```javascript
app.use(helmet());          // third-party
app.use(express.json());    // built-in
app.use(requireAuth);       // yours
```

Express itself has zero knowledge of `helmet`. It knows the contract. (This is also the Chain of Responsibility pattern — topic 47.)

### 3. Stripe's client libraries — encapsulation of retry and auth state

Stripe's Node SDK exposes `stripe.charges.create(...)`. What it hides behind that method call: the API key, connection pooling, idempotency-key generation, exponential-backoff retry on 5xx, request-ID logging, and API-version pinning. Every one of those is internal state that you cannot corrupt from the outside.

If those were public fields, every app using Stripe would eventually have one file that reached in and disabled retries "temporarily." Encapsulation is why the SDK can change its entire retry strategy in a patch release without breaking your app.

---

## Trade-offs

| Pillar | What it buys you | What it costs you |
|---|---|---|
| **Encapsulation** | Invariants enforced in one place; safe refactoring; bugs stay local | More boilerplate (getters); slightly harder to poke at internals while debugging; `#fields` are invisible to `JSON.stringify` and `Object.keys` |
| **Abstraction** | Swap implementations; trivial testing; vendor independence | Indirection — reading the code means jumping between files; a *wrong* abstraction is worse than none (you'll contort code to fit it) |
| **Inheritance** | Shared behavior defined once; expresses real taxonomies | **Tightest coupling in OOP.** A change to the base ripples to every subclass. Deep hierarchies (3+ levels) are nearly unmaintainable |
| **Polymorphism** | New behavior is *added*, not *edited*; kills switch statements | Logic is spread across many files; "where does this actually execute?" gets harder to answer |

| Situation | Reach for... | Not... |
|---|---|---|
| Two things share behavior AND one genuinely *is a* kind of the other | Inheritance | — |
| Two things share behavior but are unrelated kinds | **Composition** (topic 26) | Inheritance |
| You need to swap an algorithm at runtime | Polymorphism / Strategy (topic 42) | A `switch` |
| You have exactly 2 fixed cases that will never grow | A plain `if` | A 3-class hierarchy |
| The data must never be invalid | Encapsulation (`#fields` + methods) | A plain object + validation in the caller |

**Rule of thumb:** Encapsulate always — it's nearly free. Abstract at the boundaries of your system (databases, payment providers, message queues, anything you might replace). Use polymorphism the moment a `switch` on type appears in a *second* place. And treat inheritance as a **last resort**: reach for composition first, and only inherit when the "is-a" is so obvious you'd be embarrassed to argue against it.

---

## Common interview questions on this topic

### Q1: "Explain the four pillars of OOP."
**Hint:** Do not recite definitions — that's what everyone does. Frame each one by the **problem it solves**: encapsulation protects invariants; abstraction decouples callers from implementations; inheritance removes duplication in an is-a hierarchy; polymorphism removes `switch`-on-type. Then give one concrete example (a payment gateway is perfect: it uses all four).

### Q2: "JavaScript doesn't have interfaces. How do you do abstraction then?"
**Hint:** Three honest answers, and say all three. (1) Abstract base classes whose methods `throw new Error('not implemented')` — gives runtime enforcement and a searchable name. (2) Duck typing — any object with the right methods works, which is why JS test doubles are so easy. (3) TypeScript, if you want compile-time checking. Mention that JS is prototype-based and `class` is sugar over prototypes — interviewers love that you know it.

### Q3: "When would you NOT use inheritance?"
**Hint:** When the relationship is "has-a" or "can-do", not "is-a". Classic failure: `class Square extends Rectangle` — a Square can't honor `setWidth()`/`setHeight()` independently, so it breaks Liskov (topic 16). Also: when you'd need multiple inheritance, when the hierarchy would exceed 2-3 levels, or when subclasses start overriding methods with `throw new Error('unsupported')`. In those cases: **composition**.

### Q4: "What's the difference between encapsulation and abstraction? They sound the same."
**Hint:** Encapsulation is about **hiding data** and bundling it with the code that protects it — it's an *implementation* technique. Abstraction is about **hiding decisions** behind a contract — it's a *design* technique. You can encapsulate without abstracting (a private field in a concrete class with no interface). You can't abstract well without encapsulation, because a leaky object can't hide its "how".

### Q5: "Show me polymorphism without inheritance."
**Hint:** Duck typing. Pass any object that has the required method — `{ charge: async () => ({ ok: true }) }` works anywhere a `PaymentGateway` is expected. Also: higher-order functions. `array.sort(comparator)` is polymorphic — `sort` calls whatever comparator you hand it, with no class hierarchy anywhere. In JS, **a function is often the smallest possible strategy object.**

---

## Practice exercise

### The Media Player Refactor

You've inherited this code. It works. It is also a landmine.

```javascript
class MediaPlayer {
  constructor() {
    this.currentFile = null;
    this.volume = 50;
    this.isPlaying = false;
  }

  play(file, type) {
    if (type === 'mp3') {
      console.log(`Decoding MP3 with LAME decoder: ${file}`);
      console.log(`Playing audio at volume ${this.volume}`);
    } else if (type === 'mp4') {
      console.log(`Decoding MP4 with H.264 decoder: ${file}`);
      console.log(`Rendering video frames`);
      console.log(`Playing audio at volume ${this.volume}`);
    } else if (type === 'wav') {
      console.log(`Reading raw WAV PCM: ${file}`);
      console.log(`Playing audio at volume ${this.volume}`);
    } else {
      throw new Error('Unsupported format');
    }
    this.isPlaying = true;
    this.currentFile = file;
  }
}

const p = new MediaPlayer();
p.volume = 500;   // no complaint. speakers are now theoretically on fire.
p.isPlaying = true; // lied to the whole system; nothing is playing
```

**Your task (~30-40 min). Produce one JS file that:**

1. **Encapsulation:** Make `volume` and `isPlaying` private (`#`). Volume must be clamped to 0–100 and only settable through a `setVolume(n)` method that rejects invalid input. `isPlaying` must be read-only from the outside and only flipped by `play()` / `pause()`.
2. **Abstraction:** Define an abstract `MediaFormat` base class with the contract `{ get extension(), decode(file), get hasVideo() }`. Its methods must throw if not overridden.
3. **Inheritance:** Implement `Mp3Format`, `Mp4Format`, and `WavFormat` extending it.
4. **Polymorphism:** Rewrite `MediaPlayer.play(file, format)` so it contains **zero** `if`/`switch` on format type. It should call `format.decode(file)`, then render video only if `format.hasVideo`.
5. **Prove it:** Add a `FlacFormat` **without editing `MediaPlayer` at all.** If you had to touch `MediaPlayer`, your design failed — go back to step 4.
6. **Bonus (duck typing):** Write a `MockFormat` as a **plain object literal** (no `extends`) and pass it to `play()`. Show that it works. Explain in a comment why this is possible in JS but not in Java.

**Deliverable:** the file, plus a 3-line comment at the bottom naming which pillar solved which original problem.

---

## Quick reference cheat sheet

- **Encapsulation** = hide the data, expose the rules. Use `#privateField` in modern JS — `_underscore` is a suggestion, not a lock.
- **Abstraction** = hide the decision behind a contract. Callers depend on *what*, never *how*.
- **Inheritance** = share behavior down an **is-a** hierarchy. If you can't say "X **is a** Y" out loud without wincing, don't inherit.
- **Polymorphism** = same call, different behavior, **no branching at the call site**. It is the cure for `switch (type)`.
- **The chain:** encapsulate → so you can abstract → so many types can implement → so callers can be polymorphic.
- **JS has no `interface`.** Simulate it with an abstract base class that `throw`s, or just use duck typing, or use TypeScript.
- **JS `class` is prototype sugar.** `extends` wires up the prototype chain. Know this; interviewers ask.
- **Duck typing** means your test double needs no base class. `{ charge: async () => ({ ok: true }) }` is a valid gateway.
- **`new.target === Base` → throw** is how you make a class abstract in JS.
- **Return copies** from getters (`[...this.#items]`) or you've leaked your private state anyway.
- **Favor composition over inheritance.** Inheritance is the tightest coupling OOP offers — every base-class change ripples to every subclass.
- **Overriding a method to `throw new Error('unsupported')`** means your hierarchy is wrong. That's a Liskov violation (topic 16).
- **Every design pattern (29–51) is these four pillars rearranged.** Strategy = polymorphism. Factory = abstraction + polymorphism. Decorator = composition + polymorphism.
- **Don't apply all four to a 20-line script.** These are tools for managing *change*. Code that never changes needs none of them.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md) — the last HLD fundamental before we zoom into code |
| **Next** | [14 — SOLID: Single Responsibility Principle](./14-solid-single-responsibility.md) — the first of five principles built directly on these pillars |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — why the third pillar is the one you should use least |
| **Related** | [27 — Abstraction and Interfaces in Design](./27-abstraction-and-interfaces.md) — how to find the *right* abstraction, not just any abstraction |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — polymorphism, formalized into a named pattern |
