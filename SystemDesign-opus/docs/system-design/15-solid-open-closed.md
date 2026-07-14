# 15 — SOLID — Open/Closed Principle (OCP)

## Category: LLD Fundamentals

---

## What is this?

The Open/Closed Principle says software entities — classes, modules, functions — should be **open for extension, but closed for modification.** In plain English: **you should be able to add new behaviour by ADDING new code, not by EDITING existing, working, already-tested code.**

Think of a power strip on your wall. When you buy a new lamp, you do not call an electrician to rip open the wall and rewire the circuit. You plug the lamp into an empty socket. The wall is **closed** — nobody touches the wiring. The socket is **open** — anything with the right plug shape works. The socket shape is the *contract*; the lamp is the *extension*.

Bad code has no sockets. Every new lamp means opening the wall.

---

## Why does it matter?

Because **editing working code is how you break working code.**

Every time you open a file that already has passing tests and paying customers depending on it, you are gambling. You might fat-finger a `break` statement. You might reorder a condition. You might change a shared variable that three other cases quietly relied on. The compiler will not catch it. The code review might not catch it.

Here is the compounding cost, made concrete. Suppose your `DiscountCalculator` has a `switch` with 6 customer types, and adding a 7th requires editing it:

```
Cost of adding customer type #7 to a switch-based calculator:
  - Edit the shared file                              (merge conflict risk)
  - Re-run and re-verify all 6 existing test suites   (they touch the same file)
  - Full regression of the pricing path               (money is involved)
  - Code review by the owner of every existing case   (they don't trust you)
  → ~2 days, and a nonzero chance of a production pricing bug.

Cost with polymorphism:
  - Add one new file                                  (no merge conflict — nobody else's file)
  - Add one new test file                             (nobody else's tests re-run)
  - Register it
  → ~2 hours, and existing pricing logic is byte-for-byte untouched.
```

**In interviews:** OCP is the principle behind the single most common LLD follow-up question in existence: *"Now the product team wants to add a new [payment method / notification channel / discount type]. What changes in your design?"* If your honest answer is "I edit the switch statement in `PaymentProcessor`," you have failed the question. If your answer is "I add a `CryptoPayment` class implementing `PaymentMethod` and register it — nothing existing changes," you have passed.

**At real work:** OCP is what makes plugin systems, middleware stacks, and payment gateways possible. It is also, applied blindly, one of the top causes of unreadable over-engineered code — which we will be honest about later.

---

## The core idea — explained simply

### The Power Socket Analogy

You buy a house. In the living room there is one lamp, hard-wired directly into the wall.

**Month 1:** You want a TV. Call the electrician. He opens the wall, splices a new wire into the circuit, and connects the TV. It works — but while he was in there, he loosened the lamp's connection. The lamp now flickers.

**Month 2:** You want a heater. Electrician again. Opens the wall again. This time, he mislabels a wire, and the TV and the heater share a fuse that trips whenever both are on.

**Month 3:** Every new appliance means an electrician, a wall being opened, and a 30% chance something that used to work stops working.

Now imagine the house had been built with **sockets**.

**Month 1:** TV plugs in. The lamp does not care. The lamp does not even know the TV exists.
**Month 2:** Heater plugs in. The wall was never touched. Zero risk to the lamp or the TV.
**Month 3:** You buy something that did not exist when the house was built — a phone charger. It still works, because it satisfies the **contract**: two prongs, right voltage.

The house is **closed for modification** (you never open the wall) and **open for extension** (you can plug in devices the builder never imagined).

| Analogy piece | Technical meaning |
|---|---|
| Opening the wall to add each appliance | Editing a `switch` / `if-else` chain for every new case |
| The lamp flickering after the TV install | A regression in an old feature caused by a new feature |
| The socket | The **abstraction** — a base class or interface |
| The plug shape + voltage | The **contract** — the method signature every extension must honour |
| Plugging in a new appliance | Adding a new subclass / strategy file |
| A charger the builder never imagined | A new behaviour added years later with zero changes to old code |
| The fuse box knowing which sockets exist | The **registry / factory** that maps a key to an implementation |

**The key insight most people miss:** OCP does not say "never change code." Bug fixes change code. Refactors change code. OCP is specifically about **adding new variations of an existing concept**. Adding the 7th kind of discount should not require editing the code that handles the first six.

---

## Key concepts inside this topic

### 1. The problem — a `switch` that grows forever

Here is a discount calculator that started innocent and got ugly. Every line of this is real-world realistic.

```javascript
// ❌ BAD — every new customer type means EDITING this file.
class DiscountCalculator {
  calculate(order, customer) {
    switch (customer.type) {
      case 'REGULAR':
        return 0;

      case 'PREMIUM':
        return order.total * 0.10;

      case 'GOLD':
        return order.total * 0.20;

      // Month 4: "Add students!"      → edit the file, redeploy, retest everything
      case 'STUDENT':
        return order.total * 0.15;

      // Month 7: "Add employees!"     → edit the file again
      case 'EMPLOYEE':
        return order.total * 0.30;

      // Month 9: "Add veterans, but capped at ₹500!"
      case 'VETERAN':
        return Math.min(order.total * 0.25, 500);

      // Month 11: "Add partners — 15%, but ONLY on electronics."
      case 'PARTNER': {
        const electronics = order.items
          .filter((i) => i.category === 'ELECTRONICS')
          .reduce((sum, i) => sum + i.price * i.qty, 0);
        return electronics * 0.15;
      }

      default:
        return 0;
    }
  }
}
```

**Every problem in this file, named:**

1. **Adding a case means editing shipped code.** This file is on the money path. Every edit is a redeploy of the pricing engine.
2. **Every edit re-risks all seven cases.** Add a `case` and forget a `return`? JavaScript falls through to the next case. Now `VETERAN` customers get the `PARTNER` discount and finance calls you at 2am.
3. **The file grows without bound.** In 3 years it is 400 lines with `PARTNER_TIER_2` and `LEGACY_GOLD_GRANDFATHERED`.
4. **Merge conflicts.** Two teams adding two customer types in the same sprint both edit line 30.
5. **The full test suite must re-run.** All 7 cases live in one file, so a change to one forces re-verification of all — you cannot argue "my change couldn't have affected GOLD" when the code is in the same function.
6. **The `switch` will appear again.** Watch: soon you'll also need `getDiscountLabel(customer.type)` for the receipt, and `isDiscountStackable(customer.type)`. Now you have **three parallel switches** that must be kept in sync. Forget one and the receipt says "Regular customer — 30% off". This is the deepest symptom: **the switch metastasises.**

### 2. The fix — polymorphism (the mechanism OCP actually runs on)

Replace "*ask what type it is, then decide*" with "*ask the object to do it.*" That single reversal is the whole trick.

```javascript
// ✅ GOOD — discount-strategy.js
// The ABSTRACTION. This is the "socket". It should almost never change.
export class DiscountStrategy {
  /** @returns {number} the discount amount in the order's currency */
  calculate(order) {
    throw new Error('DiscountStrategy.calculate() must be implemented');
  }

  /** Human-readable label for the receipt. */
  get label() {
    throw new Error('DiscountStrategy.label must be implemented');
  }
}
```

```javascript
// ✅ GOOD — strategies/regular.js
import { DiscountStrategy } from '../discount-strategy.js';

export class RegularDiscount extends DiscountStrategy {
  calculate(order) { return 0; }
  get label() { return 'No discount'; }
}
```

```javascript
// ✅ GOOD — strategies/premium.js
import { DiscountStrategy } from '../discount-strategy.js';

export class PremiumDiscount extends DiscountStrategy {
  calculate(order) { return order.total * 0.10; }
  get label() { return 'Premium member — 10% off'; }
}
```

```javascript
// ✅ GOOD — strategies/veteran.js
import { DiscountStrategy } from '../discount-strategy.js';

// Notice: the cap lives WITH the rule it belongs to. In the switch version,
// this Math.min was floating in a giant function next to unrelated logic.
export class VeteranDiscount extends DiscountStrategy {
  static MAX_DISCOUNT = 500;

  calculate(order) {
    return Math.min(order.total * 0.25, VeteranDiscount.MAX_DISCOUNT);
  }

  get label() { return 'Veteran — 25% off (max ₹500)'; }
}
```

```javascript
// ✅ GOOD — strategies/partner.js
import { DiscountStrategy } from '../discount-strategy.js';

export class PartnerDiscount extends DiscountStrategy {
  calculate(order) {
    const electronicsTotal = order.items
      .filter((item) => item.category === 'ELECTRONICS')
      .reduce((sum, item) => sum + item.price * item.qty, 0);
    return electronicsTotal * 0.15;
  }

  get label() { return 'Partner — 15% off electronics'; }
}
```

And the calculator — which now **never changes again**:

```javascript
// ✅ GOOD — discount-calculator.js
export class DiscountCalculator {
  // It doesn't know WHICH strategy. It doesn't care. That's the point.
  calculate(order, strategy) {
    const discount = strategy.calculate(order);
    return {
      subtotal: order.total,
      discount,
      label: strategy.label,
      payable: order.total - discount,
    };
  }
}
```

**Now watch the "add a new type" cost.** Marketing wants a `BLACK_FRIDAY` discount:

```javascript
// ✅ strategies/black-friday.js — A NEW FILE. Nothing existing is opened.
import { DiscountStrategy } from '../discount-strategy.js';

export class BlackFridayDiscount extends DiscountStrategy {
  calculate(order) { return order.total * 0.40; }
  get label() { return 'Black Friday — 40% off'; }
}
```

`DiscountCalculator` was not opened. `PremiumDiscount` was not opened. `VeteranDiscount` was not opened. Their tests do not need to re-run, because their bytes did not change. **That is OCP.** And notice the metastasising-switch problem is gone for free: `label` lives next to `calculate` in the same class, so they can never drift out of sync.

### 3. The remaining leak — who picks the strategy?

Be honest with yourself. Something, somewhere, still has to turn the string `'GOLD'` into a `GoldDiscount` object. If that something is:

```javascript
// ❌ We just moved the switch. It's still a switch.
function getStrategy(type) {
  switch (type) {
    case 'REGULAR': return new RegularDiscount();
    case 'PREMIUM': return new PremiumDiscount();
    case 'GOLD':    return new GoldDiscount();
    // ...still editing this file for every new type.
  }
}
```

...then you have improved things (the *rules* are now isolated and testable) but you have **not achieved full OCP** — one file still gets edited per new type.

This is worth stating plainly because a lot of tutorials stop here and pretend the problem is solved. It is not. **The switch has been pushed to the edge of the system, and shrunk from "complex business logic" to "one line of object construction."** That is a genuine, large improvement — the risky part (the pricing math) is now closed. But there is a way to close the last file too.

### 4. Full OCP — the self-registering registry (plugin pattern)

Make the strategies **register themselves**. Now adding a type means adding exactly one file and importing it once.

```javascript
// ✅ discount-registry.js
export class DiscountRegistry {
  #strategies = new Map();

  register(customerType, strategyInstance) {
    if (this.#strategies.has(customerType)) {
      throw new Error(`Duplicate discount strategy for "${customerType}"`);
    }
    this.#strategies.set(customerType, strategyInstance);
    return this; // chainable
  }

  resolve(customerType) {
    const strategy = this.#strategies.get(customerType);
    if (!strategy) {
      // Fail loudly. A silent 0% discount is a business bug that hides for months.
      throw new Error(`No discount strategy registered for "${customerType}"`);
    }
    return strategy;
  }

  registeredTypes() {
    return [...this.#strategies.keys()];
  }
}

// A single shared registry instance the whole app plugs into.
export const discountRegistry = new DiscountRegistry();
```

Each strategy file now registers itself on import — the file is entirely self-contained:

```javascript
// ✅ strategies/gold.js
import { DiscountStrategy } from '../discount-strategy.js';
import { discountRegistry } from '../discount-registry.js';

class GoldDiscount extends DiscountStrategy {
  calculate(order) { return order.total * 0.20; }
  get label() { return 'Gold member — 20% off'; }
}

// The "plug going into the socket". This one line is the whole registration.
discountRegistry.register('GOLD', new GoldDiscount());
```

```javascript
// ✅ strategies/index.js — the ONE place that grows. One import line per type.
import './regular.js';
import './premium.js';
import './gold.js';
import './student.js';
import './veteran.js';
import './partner.js';
import './black-friday.js';   // ← adding a type = adding these 2 lines total
```

```javascript
// ✅ checkout.js — the calling code, which NEVER changes again.
import './strategies/index.js';               // side-effect import: everything registers
import { discountRegistry } from './discount-registry.js';
import { DiscountCalculator } from './discount-calculator.js';

const calculator = new DiscountCalculator();

export function priceOrder(order, customer) {
  const strategy = discountRegistry.resolve(customer.type);
  return calculator.calculate(order, strategy);
}

// --- demo ---
const order = {
  total: 3000,
  items: [
    { name: 'Headphones', category: 'ELECTRONICS', price: 2000, qty: 1 },
    { name: 'T-shirt',    category: 'APPAREL',     price: 1000, qty: 1 },
  ],
};

console.log(priceOrder(order, { type: 'GOLD' }));
// { subtotal: 3000, discount: 600, label: 'Gold member — 20% off', payable: 2400 }

console.log(priceOrder(order, { type: 'PARTNER' }));
// { subtotal: 3000, discount: 300, label: 'Partner — 15% off electronics', payable: 2700 }

console.log(discountRegistry.registeredTypes());
// [ 'REGULAR', 'PREMIUM', 'GOLD', 'STUDENT', 'VETERAN', 'PARTNER', 'BLACK_FRIDAY' ]
```

**This is full OCP.** The pricing engine is genuinely closed. To add a discount type, you write one new file and add one import line to a manifest — and if you load strategies by scanning a directory (`fs.readdirSync('./strategies')`), even the manifest disappears. Zero existing files touched.

**The honest cost of the registry version:** you have traded a compile-time guarantee for a runtime one. With a `switch`, a typo in a type name is at worst obvious in one place. With a registry, forget the import line in `index.js` and `resolve('BLACK_FRIDAY')` throws **at runtime, in production, during the Black Friday sale**. This is why `resolve()` throws loudly instead of returning a default — and why a startup smoke test that asserts every expected type is registered is not optional.

### 5. OCP is the PRINCIPLE. Strategy and Factory are the MECHANISM.

Students constantly confuse these three. Get this straight and you will answer the interview follow-up cleanly:

| | What it is | What it says |
|---|---|---|
| **OCP** (this topic) | A **principle** — a goal, a property you want your code to have | "Adding behaviour should not require editing existing code." |
| **Strategy** ([42](./42-pattern-strategy.md)) | A **design pattern** — a concrete code structure | "Put each algorithm in its own class behind a common interface; swap them at runtime." |
| **Factory** ([30](./30-pattern-factory-method.md)) | A **design pattern** — a concrete code structure | "Move object creation into a dedicated place so callers don't hard-code `new ConcreteThing()`." |

The relationship, in one line:

```
        OCP  =  the GOAL ("don't edit working code to add a variation")
         │
         ├── achieved BY ──▶  Strategy  (the varying behaviour lives in swappable classes)
         └── achieved BY ──▶  Factory / Registry  (the CHOICE of which class is also centralised)
```

You can achieve OCP with other mechanisms too — higher-order functions, event listeners, middleware chains, dependency injection, a config-driven table. Strategy and Factory are the two most common, not the only two. **Always name the principle when the interviewer asks "why did you use Strategy here?" The answer is: "to satisfy OCP — so a new discount type doesn't require reopening the pricing engine."**

### 6. The honest counter-argument — when a `switch` is correct

Now the part most tutorials refuse to say: **a `switch` with 3 stable cases is fine. Leave it alone.**

```javascript
// ✅ This is GOOD CODE. Do not "fix" it.
function formatBytes(bytes, unit) {
  switch (unit) {
    case 'KB': return (bytes / 1024).toFixed(2);
    case 'MB': return (bytes / 1024 ** 2).toFixed(2);
    case 'GB': return (bytes / 1024 ** 3).toFixed(2);
    default:   return String(bytes);
  }
}
```

Turning this into `ByteFormatterStrategy` + `KilobyteFormatter` + `MegabyteFormatter` + `ByteFormatterRegistry` would be **absurd**. Four files, an abstraction, and a registry to express what a five-line function says perfectly. There is no fourth unit coming. The set is closed by physics.

The failure mode of over-applied OCP has a name: **speculative generality**. You build a plugin architecture for a system that will only ever have two plugins. The costs are real:

- **Indirection tax.** To answer "what discount does a GOLD customer get?", a new engineer must find the registry, find the registration, find the file, read the class. In the `switch` version, they read one line.
- **Abstractions built on guesses are usually wrong.** You designed `DiscountStrategy.calculate(order)`. Six months later, Marketing wants a discount that depends on the customer's *purchase history*, not just the order. Your interface signature is wrong. Now you edit the base class and **every single strategy** — the exact modification OCP was supposed to prevent. **A wrong abstraction is more expensive than a switch statement.**
- **You cannot see the whole logic in one place.** Sometimes "all pricing rules visible in one function" is genuinely the more debuggable design.

This ties directly to **YAGNI** ([19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md)): *You Aren't Gonna Need It.* Don't build the extension point for a change that hasn't been asked for.

### 7. The rule: abstract on the SECOND or THIRD change, not the first

Here is the practical decision procedure that experienced engineers actually use.

```
  Change #1 → Just write the switch / if-else. It's honest and readable.
              You do not yet know what varies. Any abstraction now is a GUESS.

  Change #2 → Write the second case. STILL just add it to the switch.
              Now you have two data points. Start noticing the shape of what varies.
              (Some teams abstract here. Defensible, but early.)

  Change #3 → STOP. This is the signal. The variation is now a PROVEN, RECURRING axis
              of change — not a hypothesis. You have three real examples, so you can
              design the abstraction from EVIDENCE instead of imagination.
              → NOW extract the Strategy. The interface you design will actually fit,
                because you built it around three real cases, not one imagined one.
```

This is sometimes called the **Rule of Three**. The reasoning is precise: with one example you cannot see what varies; with two you might be seeing a coincidence; with three you can see the *axis* of variation and design an interface that will fit the fourth.

**The two trigger questions to ask, honestly:**

1. **"Has this actually changed twice already?"** — Not "might it change." *Has* it. History is evidence; the roadmap is a rumour.
2. **"Is the set open or closed?"** — Payment methods, notification channels, discount types, file exporters, auth providers: **open sets** — the business will keep adding to them forever. Days of the week, HTTP methods, chess pieces, byte units: **closed sets** — they are finished. **Abstract open sets. Switch on closed sets.**

That second question is the one that gets you the job, because it means you understand OCP is about *predicting where change actually happens*, not about eliminating `switch` from the language.

---

## Visual / Diagram description

### Diagram 1: BEFORE — every new type reopens the same file

```
   New requirement:  "Add STUDENT"    "Add VETERAN"    "Add PARTNER"
                            │                │                │
                            │                │                │
                            └────────────────┼────────────────┘
                                             │
                                             ▼   ⚠ EDIT. REDEPLOY. RETEST ALL.
                        ┌──────────────────────────────────────┐
                        │        DiscountCalculator            │
                        ├──────────────────────────────────────┤
                        │  calculate(order, customer) {        │
                        │    switch (customer.type) {          │
                        │      case 'REGULAR':  ...            │  ← risk
                        │      case 'PREMIUM':  ...            │  ← risk
                        │      case 'GOLD':     ...            │  ← risk
                        │      case 'STUDENT':  ...   ← added  │
                        │      case 'VETERAN':  ...   ← added  │
                        │      case 'PARTNER':  ...   ← added  │
                        │    }                                 │
                        │  }                                   │
                        └──────────────────────────────────────┘
                          Grows forever. One missing `return` and
                          VETERAN silently gets PARTNER's discount.
```

### Diagram 2: AFTER — the calculator is closed; strategies plug in

```
                          ┌───────────────────────────┐
                          │    DiscountCalculator     │      CLOSED
                          ├───────────────────────────┤   ✓ never edited again
                          │ + calculate(order,        │
                          │             strategy)     │
                          └─────────────┬─────────────┘
                                        │ depends on the ABSTRACTION,
                                        │ never on a concrete class
                                        ▼
                          ╔═══════════════════════════╗
                          ║   «abstract»              ║      THE SOCKET
                          ║   DiscountStrategy        ║   (the contract)
                          ╠═══════════════════════════╣
                          ║ + calculate(order): number║
                          ║ + get label(): string     ║
                          ╚═════════════▲═════════════╝
                                        │  extends
        ┌──────────────┬────────────────┼────────────────┬──────────────────┐
        │              │                │                │                  │
┌───────┴──────┐┌──────┴───────┐┌───────┴──────┐┌────────┴──────┐┌──────────┴────────┐
│ RegularDisc. ││ PremiumDisc. ││  GoldDisc.   ││ VeteranDisc.  ││ BlackFridayDisc.  │
├──────────────┤├──────────────┤├──────────────┤├───────────────┤├───────────────────┤
│+calculate()  ││+calculate()  ││+calculate()  ││+calculate()   ││ +calculate()      │
│  → 0         ││  → 10%       ││  → 20%       ││  → 25% cap 500││  → 40%            │
│+label        ││+label        ││+label        ││+label         ││ +label            │
└──────────────┘└──────────────┘└──────────────┘└───────────────┘└───────────────────┘
                                                                  ▲
                                                    NEW FILE ─────┘   OPEN
                                                    Nothing above was touched.

                          ┌───────────────────────────┐
                          │     DiscountRegistry      │   the fuse box:
                          ├───────────────────────────┤   maps 'GOLD' → GoldDiscount
                          │ - strategies: Map         │
                          │ + register(type, strategy)│   ← each file self-registers
                          │ + resolve(type): Strategy │   ← throws if unknown
                          └───────────────────────────┘
```

**How to read it:** the arrow from `DiscountCalculator` points at the **abstract** box, never at a concrete strategy. That is the whole game — the calculator's source code contains no mention of `GOLD`, `PARTNER`, or `BlackFridayDiscount`, so it physically *cannot* need editing when one is added. Adding `BlackFridayDiscount` grows the diagram sideways at the bottom; every box above it is untouched. If you can redraw this diagram from memory, you understand OCP.

---

## Real world examples

### 1. Express.js middleware — OCP as an architecture

Express's request pipeline is closed. Its core `Router` was written years ago and does not know your auth logic exists. You extend it by *adding* a function:

```javascript
app.use(myNewRateLimiter);   // extension: a new function
app.use(myNewAuthCheck);     // extension: another new function
```

The Express source code is never edited. The abstraction is the contract `(req, res, next) => void`. Every middleware ever written — `helmet`, `cors`, `morgan`, yours — plugs into that one socket. That is why the npm ecosystem around Express exists at all: a closed core with an open extension point.

### 2. Payment gateways (Stripe, Razorpay) and their SDK adapters

Real e-commerce backends define a `PaymentMethod` abstraction with `charge(amount, token)` and `refund(chargeId)`. Concrete classes: `CardPayment`, `UPIPayment`, `NetBankingPayment`, `WalletPayment`. When Razorpay or Stripe adds a new method (say, a new BNPL provider), the checkout service adds a class — the order flow, cart, and invoicing code stay closed. Teams that skipped this and wrote `if (method === 'card') { stripe... } else if (method === 'upi') { razorpay... }` inside the checkout controller find that adding a payment method means editing the highest-risk file in the company.

### 3. Node.js `Stream` and the `EventEmitter` contract

Node's core stream machinery is closed. You extend it by subclassing `Readable`/`Writable`/`Transform` and implementing `_read()` / `_write()` / `_transform()` — the contract. `zlib.createGzip()`, `crypto.createCipheriv()`, `fs.createReadStream()`, and your custom CSV parser all satisfy the same socket, which is why `pipeline(a, b, c)` composes things Node's authors never anticipated. Similarly, `EventEmitter` is open for extension (`.on('anything', handler)`) and closed for modification — you never edit the emitter to support a new event name.

---

## Trade-offs

| Aspect | `switch` / `if-else` chain | Polymorphic strategies + registry |
|---|---|---|
| **Adding a new case** | Edit existing, shipped, tested file | Add a new file; touch nothing existing |
| **Risk to existing behaviour** | Real — fallthrough, shared state, typos | Near zero — old bytes unchanged |
| **Readability for 3 simple cases** | Excellent — all logic visible at once | Worse — indirection for no benefit |
| **Readability for 12 complex cases** | Terrible — a 400-line function | Excellent — one small file per rule |
| **Files to open to trace one case** | 1 | 3 (caller → registry → strategy) |
| **Merge conflicts** | Frequent (everyone edits one file) | Rare (everyone adds their own file) |
| **Test blast radius per change** | The whole file's suite | Just the new file's suite |
| **Failure mode** | Missing `case` → silent wrong result | Missing registration → loud runtime throw |
| **Cost if you guessed the interface wrong** | None — just edit the switch | High — you must change the base class **and every strategy** |

**The sweet spot:** apply OCP to **open sets** that have **already changed at least twice** — payment methods, notification channels, discount rules, file exporters, auth providers. Keep the `switch` for **closed sets** and for variation that has never actually varied.

**Rule of thumb:** **Abstract on the second or third change, not the first.** The first time, write the switch. The third time you edit the same switch for the same reason, stop and extract the Strategy — you now have three real examples, so the abstraction you design will be built on evidence rather than a guess. And remember: a *wrong* abstraction costs more than the `switch` it replaced.

---

## Common interview questions on this topic

### Q1: "What does 'open for extension, closed for modification' actually mean in code?"
**Hint:** New behaviour = new file/class. Existing, working, tested source files are not opened. Give the concrete contrast: adding a 7th discount type means editing `DiscountCalculator` (violation) versus adding `BlackFridayDiscount.js` and registering it (compliance). Name the mechanism: **polymorphism** — replace "check the type, then decide" with "ask the object."

### Q2: "You replaced the switch with strategies — but something still has to pick the strategy. Didn't you just move the switch?"
**Hint:** This is the good interviewer's trap, and the honest answer scores points. Say: *"Yes — a naive factory just relocates it. But it shrinks the switch from complex business logic to one line of object construction, so the risky code is closed. To close the last file, use a self-registering registry: each strategy calls `registry.register('GOLD', new GoldDiscount())` on import, so adding a type touches zero existing files."* Then volunteer the cost: the registry trades a compile-time check for a runtime one, so you need a startup assertion that every expected type is registered.

### Q3: "What's the relationship between OCP, the Strategy pattern, and the Factory pattern?"
**Hint:** **OCP is the principle (the goal). Strategy and Factory are mechanisms (the how).** Strategy makes the *behaviour* pluggable; Factory/Registry makes the *choice of which behaviour* pluggable. You need both for full OCP. Also note OCP can be satisfied without either — middleware chains, higher-order functions, and event listeners are all OCP-compliant extension points.

### Q4: "Is a `switch` statement always a code smell?"
**Hint:** Absolutely not — and say so confidently, because a candidate who applies principles blindly is a liability. A switch over a **closed set** (HTTP methods, byte units, days of the week) is correct and clearer than any abstraction. The smell is a switch over an **open set** that the business keeps adding to. Ask yourself: "will this list keep growing forever?" If yes → abstract. If it's finished → switch.

### Q5: "When do you decide to introduce the abstraction?"
**Hint:** The **Rule of Three**. First occurrence: write it inline. Second: notice it. Third: extract. Abstracting on the first change is speculative generality (YAGNI) and the interface you invent will probably be wrong — because you designed it from one example. With three real cases you can see the actual axis of variation. Add the killer line: *"a wrong abstraction is more expensive than duplication, because fixing it means editing the base class and every subclass — the exact thing OCP was meant to prevent."*

---

## Practice exercise

### Close the Notification Sender

You inherit this. It works. It is also about to be asked to support Slack, WhatsApp, and in-app notifications.

```javascript
// ❌ Refactor me.
class NotificationSender {
  async send(channel, user, message) {
    switch (channel) {
      case 'EMAIL':
        if (!user.email) throw new Error('no email on file');
        await sendGrid.send({ to: user.email, subject: message.title, html: message.body });
        break;

      case 'SMS':
        if (!user.phone) throw new Error('no phone on file');
        // SMS has a hard 160-char limit — must truncate
        await twilio.messages.create({ to: user.phone, body: message.body.slice(0, 160) });
        break;

      case 'PUSH':
        if (!user.deviceToken) return; // silently skip — user has no app installed
        await fcm.send({ token: user.deviceToken, notification: { title: message.title, body: message.body } });
        break;

      default:
        throw new Error(`Unknown channel: ${channel}`);
    }
  }
}
```

**What to produce (aim for ~35 minutes):**

1. **A `NotificationChannel` base class** in JavaScript with the contract every channel must honour. Think carefully about the signature — the three existing channels behave differently (SMS truncates, PUSH can silently no-op, EMAIL throws). Design the interface to accommodate all three **without special-casing any of them**. Hint: you probably want both `canSend(user)` and `send(user, message)`.
2. **Three concrete channel classes** — `EmailChannel`, `SmsChannel`, `PushChannel` — each in its own file, each closed over its own quirks (the 160-char truncation belongs *inside* `SmsChannel`, not in the caller).
3. **A `NotificationRegistry`** where each channel self-registers, plus a `NotificationService` that resolves and sends. Make `resolve()` throw loudly on an unknown channel.
4. **Add `SlackChannel` and prove OCP holds.** Write it as a new file. Then state explicitly, in a comment: *which existing files did you have to open?* If the answer is anything other than "the manifest import list," your design leaks.
5. **The honest paragraph.** Write 3–4 sentences answering: *was this refactor worth it?* Consider: if this app will only ever have EMAIL and SMS forever, was the original switch actually better? Under what condition does your answer flip? Name the specific evidence that would justify the abstraction.

---

## Quick reference cheat sheet

- **OCP** = "Open for extension, closed for modification." **Add new code; don't edit working code.**
- **The mechanism is polymorphism** — replace *"check the type, then decide"* with *"ask the object to do it."*
- **The smell:** a `switch` / `if-else` chain over a type that the business keeps adding to.
- **The worse smell:** the *same* switch appearing in three places (`calculate`, `getLabel`, `isStackable`) that must stay in sync.
- **OCP is the PRINCIPLE. Strategy (42) and Factory (30) are the MECHANISM.** Never confuse the goal with the tool.
- **A naive factory just relocates the switch** — you shrink it from business logic to object construction. Real, but partial.
- **Full OCP = self-registering registry.** Each strategy calls `registry.register(key, impl)` on import. New type = new file, zero edits.
- **The registry's cost:** a forgotten registration fails at **runtime**, not compile time. Add a startup assertion.
- **Open sets** (payment methods, notification channels, discount types, exporters) → **abstract**.
- **Closed sets** (HTTP methods, byte units, days of the week, chess pieces) → **switch is correct**. Leave it alone.
- **Rule of Three:** write the switch on change #1, tolerate it on #2, extract the Strategy on #3.
- **A wrong abstraction costs more than a switch** — fixing it means editing the base class AND every subclass.
- **Trigger question:** *"Has this actually changed twice already?"* History is evidence; the roadmap is a rumour.
- **Interview line that wins:** "I'd apply OCP here because payment methods are an open set that has already grown twice — but I'd leave the byte-unit switch alone, because that set is closed."

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [14 — SOLID: Single Responsibility](./14-solid-single-responsibility.md) — SRP splits classes by actor; OCP keeps those classes closed as behaviour grows |
| **Next** | [16 — SOLID: Liskov Substitution](./16-solid-liskov-substitution.md) — OCP only works if subclasses are truly substitutable for their base |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) — the primary MECHANISM for achieving OCP |
| **Related** | [30 — Factory Method Pattern](./30-pattern-factory-method.md) — closes the last leak: who decides which strategy to instantiate |
| **Related** | [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) — the guardrail against applying OCP speculatively |
