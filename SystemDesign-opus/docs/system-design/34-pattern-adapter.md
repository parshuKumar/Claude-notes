# 34 — Adapter Pattern (aka Wrapper)

## Category: LLD Patterns

---

## What is this?

The **Adapter** pattern makes two incompatible interfaces work together by putting a small translator between them. Your code speaks one "language" (a method shape it expects), the other object speaks a different one, and the adapter sits in the middle converting one call into the other.

The analogy is literally in the name: you land in London with a laptop charger that has a flat two-pin European plug, and the wall socket has three rectangular UK holes. You don't rewire your laptop and you don't rewire the building. You buy a **travel adapter** — a cheap piece of plastic whose entire job is to accept one shape on the input side and present a different shape on the output side. Electricity flows unchanged; only the *interface* was converted.

---

## Why does it matter?

Without Adapter, third-party code leaks into every corner of your app. You call `stripe.paymentIntents.create({ amount, currency, payment_method })` in your checkout controller, in your subscription renewal job, in your refund handler, and in your admin panel. Then the company signs a deal with PayPal, or Stripe deprecates an API version, and you get to change **eleven files** and re-test all of them. Every one of those call sites is a place a bug can hide.

With Adapter, your application only ever knows about *your* interface — `PaymentGateway.charge(amount, token)`. Behind it, one small class knows Stripe's dialect and another knows PayPal's. Swapping vendors becomes a one-line change in a factory, and your business logic never learns that Stripe exists.

**In interviews:** Adapter comes up the moment you say "we'll integrate a third-party payment/SMS/storage provider," because the follow-up is always *"what if we need a second provider?"* — and the expected answer is an interface plus adapters. It's also one of the three structural patterns (Adapter, Decorator, Facade) interviewers ask you to **distinguish**, since all three wrap an object.

**At work:** Adapter is the single most useful pattern for keeping a codebase testable. Once your app depends on `PaymentGateway` rather than the Stripe SDK, you drop in a `FakeGateway` in tests and never touch the network.

---

## The core idea — explained simply

### The Travel Adapter Analogy

You are in a hotel room in London.
- **Your laptop charger** = your application code. It has a fixed plug shape (a fixed set of calls it wants to make). You *could* cut the cable and solder on a new plug — but that destroys the charger, and you'd do it again in the next country.
- **The UK wall socket** = the third-party SDK. Fixed shape too, and definitely not yours to change — it's the building's wiring.
- **The travel adapter** = the Adapter class. Your charger's shape on one side, the socket's shape on the other. No logic of its own; it just translates.

The key insight: **neither the charger nor the socket has to know the adapter exists.** The charger thinks it's plugged into a European socket. The socket thinks a British plug went in. The adapter maintains that illusion for both.


| Travel analogy | Adapter pattern term | In our payment example |
|----------------|---------------------|------------------------|
| Your laptop charger | **Client** | `CheckoutService` — the code that wants to charge a card |
| The plug shape the charger has | **Target interface** | `PaymentGateway` with `charge(amount, token)` |
| The UK wall socket | **Adaptee** | The Stripe SDK with `paymentIntents.create({...})` |
| The travel adapter | **Adapter** | `StripeAdapter` — implements `PaymentGateway`, calls Stripe inside |
| Electricity | The actual work | Money actually moving |

The payoff: buying a second adapter (Japan, the US) is cheap and doesn't touch your charger. Writing a second adapter (`PayPalAdapter`) is cheap and doesn't touch your business logic.

---

## Key concepts inside this topic

### 1. The problem it solves — start with the pain

Here is checkout code with no adapter. It works — and it is a trap.

```javascript
// ❌ BAD — the Stripe SDK's shape has leaked into business logic
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_KEY);

class CheckoutService {
  async checkout(order, cardToken) {
    // Stripe wants amounts in the smallest currency unit (cents),
    // wants snake_case keys, and returns an object with a `.status` string.
    const intent = await stripe.paymentIntents.create({
      amount: Math.round(order.totalDollars * 100),
      currency: 'usd',
      payment_method: cardToken,
      confirm: true,
    });

    if (intent.status !== 'succeeded') {
      throw new Error(`Payment failed: ${intent.last_payment_error?.message}`);
    }
    return { chargeId: intent.id };
  }
}
```

Three things are wrong here, and none of them are obvious until it's too late:

1. **Stripe's vocabulary is now your vocabulary.** `paymentIntents`, `confirm: true`, `status === 'succeeded'`, cents-vs-dollars — all of that Stripe-specific knowledge is sitting inside a class that should only care about orders.
2. **You cannot add a second provider without an `if`.** The day PayPal arrives, this method sprouts `if (provider === 'stripe') { … } else if (provider === 'paypal') { … }` — and so does `refund()`, and `getStatus()`, and every other money-touching method. Cost grows as **providers × call-sites**. That's the smell Adapter exists to remove.
3. **You cannot unit-test it.** Testing `checkout()` means either hitting Stripe's network sandbox or monkey-patching the SDK module.

### 2. The structure — who plays which role

Adapter has exactly four participants — learn these names, interviewers use them.

| Participant | Meaning | In our example |
|-------------|---------|----------------|
| **Client** | The code that wants something done | `CheckoutService` |
| **Target** | The interface the client is written against | `PaymentGateway` (abstract base class) |
| **Adaptee** | The existing/foreign class with the wrong shape | `Stripe` SDK, `PayPal` SDK |
| **Adapter** | Implements Target, holds an Adaptee, translates calls | `StripeAdapter`, `PayPalAdapter` |

The rule of thumb: **the Target interface is designed by YOU, for YOUR needs.** This is the part beginners get wrong — they design the target interface to look like the SDK they happen to be using first, and then the "adapter" for the second vendor is a nightmare of impedance mismatch. Design the interface around what your *domain* needs (`charge`, `refund`, `getStatus`) and let each adapter do the ugly work.

### 3. Object adapter (composition) vs class adapter (inheritance)

There are two textbook flavours.

```javascript
// CLASS ADAPTER — the adapter IS-A adaptee. Needs multiple inheritance in
// most languages; JS's single inheritance makes this awkward and rare.
class StripeClassAdapter extends StripeSDK {       // inherits the ADAPTEE
  async charge(amountDollars, token) {             // bolts on the TARGET method
    const intent = await this.paymentIntents.create({ /* ... */ });
    return { id: intent.id, status: intent.status };
  }
}

// OBJECT ADAPTER — the adapter HAS-A adaptee. This is the JS way.
class StripeAdapter extends PaymentGateway {       // implements the TARGET
  constructor(stripeClient) {
    super();
    this.stripe = stripeClient;                    // composition: adaptee injected
  }
  async charge(amountDollars, token) { /* translate + delegate */ }
}
```

| | Class adapter (inheritance) | Object adapter (composition) |
|---|---|---|
| Relationship | Adapter **is-a** Adaptee | Adapter **has-a** Adaptee |
| Adapt subclasses of the Adaptee? | No — bound to one concrete class | Yes — inject any object with the right methods |
| Needs multiple inheritance? | Usually yes (JS doesn't have it) | No |
| Testability | You inherit the SDK's constructor and network calls | You inject a fake adaptee freely |

**In JavaScript, always prefer the object adapter.** JS has single inheritance, most SDKs are factory functions or plain objects rather than extensible classes, and composition lets you inject a mock in tests. (Recall from [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) that inheritance locks the relationship at compile time while composition lets you decide at runtime.)

### 4. Full JavaScript implementation

Complete and runnable with `node adapter-demo.js`. The two "SDKs" at the top are fake stand-ins that mimic the real APIs' shapes closely enough to make the point.

```javascript
// ---- PART 0 — The ADAPTEES: two third-party SDKs we do not control.
// Different method names, argument shapes, units (cents vs dollars),
// success signals, and error styles. We cannot change any of it.

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const rand = () => Math.random().toString(36).slice(2, 10);

class StripeSDK {
  // Takes CENTS (integer) and snake_case keys. SIGNALS FAILURE by RETURNING
  // a status string — it does not throw.
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.paymentIntents = {
      create: async ({ amount, currency, payment_method }) => {
        await sleep(20);
        if (payment_method === 'tok_declined') {
          return { id: 'pi_fail', status: 'requires_payment_method',
                   last_payment_error: { message: 'Your card was declined.' } };
        }
        return { id: `pi_${rand()}`, status: 'succeeded', amount, currency };
      },
    };
    this.refunds = { create: async ({ payment_intent, amount }) =>
      ({ id: `re_${rand()}`, status: 'succeeded', payment_intent, amount }) };
  }
}

class PayPalSDK {
  constructor(clientId, secret) { this.clientId = clientId; this.secret = secret; }
  // PayPal takes DOLLARS as a decimal STRING, nests everything inside
  // purchase_units, and THROWS on failure instead of returning a status.
  async captureOrder(orderToken, { purchase_units }) {
    await sleep(25);
    if (orderToken === 'PP-DECLINED') {
      const err = new Error('INSTRUMENT_DECLINED');
      err.details = [{ issue: 'INSTRUMENT_DECLINED' }];
      throw err;
    }
    return { result: { id: `PAY-${rand()}`, status: 'COMPLETED', purchase_units } };
  }
  async refundCapture(captureId, { amount }) {
    await sleep(25);
    return { result: { id: `REF-${rand()}`, status: 'COMPLETED', captureId, amount } };
  }
}

// ---- PART 1 — The TARGET. WE design this, in OUR domain language:
// dollars as a number, a token in, a Receipt back, a PaymentError on
// failure. No vendor word appears anywhere in it.

class PaymentError extends Error {
  constructor(message, { retryable = false, providerCode = null } = {}) {
    super(message);
    this.name = 'PaymentError';
    this.retryable = retryable;
    this.providerCode = providerCode;
  }
}

class Receipt {
  constructor({ chargeId, amountDollars, provider }) {
    Object.assign(this, { chargeId, amountDollars, provider, createdAt: new Date().toISOString() });
  }
}

/** The Target. In JS we express an "interface" as a base class whose
 *  methods throw — subclasses MUST override them. */
class PaymentGateway {
  /** @returns {Promise<Receipt>} @throws {PaymentError} */
  async charge(amountDollars, token) { throw new Error('charge() must be implemented'); }
  /** @returns {Promise<Receipt>} */
  async refund(chargeId, amountDollars) { throw new Error('refund() must be implemented'); }
}

// ---- PART 2 — The ADAPTERS. One per vendor. Each is an OBJECT adapter:
// it HOLDS the SDK rather than inheriting from it.

class StripeAdapter extends PaymentGateway {
  constructor(stripeClient) { super(); this.stripe = stripeClient; } // adaptee injected

  async charge(amountDollars, token) {
    // TRANSLATE IN #1: our dollars -> Stripe's integer cents. Math.round guards
    // against float dust (19.99 * 100 === 1998.9999999999998).
    const intent = await this.stripe.paymentIntents.create({
      amount: Math.round(amountDollars * 100),
      currency: 'usd',
      payment_method: token,   // TRANSLATE IN #2: our `token` -> their key name
      confirm: true,
    });

    // TRANSLATE OUT #1: Stripe's status string -> our thrown PaymentError.
    if (intent.status !== 'succeeded') {
      throw new PaymentError(intent.last_payment_error?.message ?? 'Charge failed',
                             { retryable: false, providerCode: intent.status });
    }
    // TRANSLATE OUT #2: Stripe's PaymentIntent -> our Receipt.
    return new Receipt({ chargeId: intent.id, amountDollars, provider: 'stripe' });
  }

  async refund(chargeId, amountDollars) {
    const res = await this.stripe.refunds.create({
      payment_intent: chargeId, amount: Math.round(amountDollars * 100),
    });
    return new Receipt({ chargeId: res.id, amountDollars, provider: 'stripe' });
  }
}

class PayPalAdapter extends PaymentGateway {
  constructor(paypalClient) { super(); this.paypal = paypalClient; }

  async charge(amountDollars, token) {
    try {
      // TRANSLATE IN: our plain number -> PayPal's nested decimal-string shape.
      const res = await this.paypal.captureOrder(token, {
        purchase_units: [{ amount: { value: amountDollars.toFixed(2), currency_code: 'USD' } }],
      });
      if (res.result.status !== 'COMPLETED') {
        throw new PaymentError('Charge not completed', { providerCode: res.result.status });
      }
      return new Receipt({ chargeId: res.result.id, amountDollars, provider: 'paypal' });
    } catch (err) {
      if (err instanceof PaymentError) throw err;
      // TRANSLATE OUT: PayPal THROWS where Stripe RETURNS a status. Normalising
      // both here is what lets the client catch exactly ONE error type.
      const issue = err.details?.[0]?.issue ?? 'UNKNOWN';
      throw new PaymentError(`PayPal rejected the payment (${issue})`,
                             { retryable: issue === 'INTERNAL_SERVER_ERROR', providerCode: issue });
    }
  }

  async refund(chargeId, amountDollars) {
    const res = await this.paypal.refundCapture(chargeId, {
      amount: { value: amountDollars.toFixed(2), currency_code: 'USD' },
    });
    return new Receipt({ chargeId: res.result.id, amountDollars, provider: 'paypal' });
  }
}

// A third implementation that talks to nothing at all — this is WHY Adapter
// makes code testable. Your test suite never touches the network.
class FakeGateway extends PaymentGateway {
  async charge(amountDollars, token) {
    if (token === 'FAIL') throw new PaymentError('Simulated decline');
    return new Receipt({ chargeId: `fake_${rand()}`, amountDollars, provider: 'fake' });
  }
  async refund(chargeId, amountDollars) {
    return new Receipt({ chargeId: `fakeref_${rand()}`, amountDollars, provider: 'fake' });
  }
}

// ---- PART 3 — The CLIENT. Zero vendor knowledge. It could not tell you
// whether it is talking to Stripe, PayPal, or a fake.

class CheckoutService {
  constructor(gateway) { this.gateway = gateway; } // depends on TARGET, never an SDK

  async checkout(order, token) {
    const receipt = await this.gateway.charge(order.totalDollars, token);
    order.status = 'PAID';
    order.chargeId = receipt.chargeId;
    return receipt;
  }

  async cancel(order) {
    if (order.status !== 'PAID') throw new Error('Only paid orders can be cancelled');
    const receipt = await this.gateway.refund(order.chargeId, order.totalDollars);
    order.status = 'REFUNDED';
    return receipt;
  }
}

// The ONE place that knows which vendor is live. Swapping providers is a
// config change here — CheckoutService is never retested, or even re-read.
function makeGateway(name) {
  switch (name) {
    case 'stripe': return new StripeAdapter(new StripeSDK('sk_test_123'));
    case 'paypal': return new PayPalAdapter(new PayPalSDK('client_id', 'secret'));
    case 'fake':   return new FakeGateway();
    default: throw new Error(`Unknown payment provider: ${name}`);
  }
}

// ---- PART 4 — DEMO
async function main() {
  for (const provider of ['stripe', 'paypal', 'fake']) {
    const checkout = new CheckoutService(makeGateway(provider));
    const order = { id: 'ord_1', totalDollars: 19.99, status: 'PENDING' };

    const receipt = await checkout.checkout(order, provider === 'paypal' ? 'PP-OK' : 'tok_visa');
    console.log(`[${provider}] charged $${receipt.amountDollars} -> ${receipt.chargeId}`);

    const refund = await checkout.cancel(order);
    console.log(`[${provider}] refunded -> ${refund.chargeId}, order is ${order.status}`);
  }

  // Failure path: two totally different vendor error styles, ONE error type out.
  for (const [provider, badToken] of [['stripe', 'tok_declined'], ['paypal', 'PP-DECLINED']]) {
    try {
      await new CheckoutService(makeGateway(provider))
        .checkout({ id: 'ord_2', totalDollars: 5, status: 'PENDING' }, badToken);
    } catch (err) {
      console.log(`[${provider}] ${err.name}: ${err.message} (code=${err.providerCode})`);
    }
  }
}

main();
```

Run it: identical-shaped output for every provider. That's the whole point.

### 5. A real Node.js ecosystem example

Adapters are everywhere in Node, usually under a different name.

- **Database drivers behind an ORM.** Knex and Sequelize define one query interface, then ship a `mysql2` adapter, a `pg` adapter, an `sqlite3` adapter. You write `db.select('*').from('users')`; the adapter turns it into that driver's dialect. Target = the query builder API; adaptees = the raw drivers.
- **`passport` strategies.** Passport's target is `verify(credentials, done)`. Every OAuth provider has a wildly different token dance; each `passport-*` package is an adapter hiding it behind that one callback shape.
- **Winston / Pino transports.** Your app calls `logger.info(msg)`; a transport adapts it into a file write, an HTTP POST to Datadog, or a syslog packet.
- **`node-fetch` / `undici`.** They adapt Node's stream-and-callback `http.request` into the browser's promise-based `fetch()`. Different interface, same capability — textbook adapter.

### 6. When NOT to use it / how it's abused

**Don't use Adapter when you own both sides.** If the "incompatible" interface is your own class, just *edit it*. An adapter around code you control is pure indirection.

**The lowest-common-denominator trap.** The classic abuse: your target interface can only express features that *all* vendors support, so you lose Stripe's idempotency keys because PayPal has no equivalent. Fix: design the target for your domain, and let adapters expose optional capabilities via a `supports(feature)` check — don't cripple the interface to force symmetry.

**The leaky adapter.** If `PaymentGateway.charge()` takes an argument called `payment_method_id` or returns a raw `PaymentIntent`, you haven't adapted anything — the vendor's shape is still leaking. The test: could you swap vendors without touching the client? If not, the adapter is fake. (Related: **adapter chains.** An adapter around an adapter around a wrapper means someone kept patching instead of redesigning. Two hops max.)

### 7. Related patterns and how they differ (the #1 interview follow-up)

All three of these wrap another object. Interviewers ask you to tell them apart because most candidates can't.

| Pattern | Interface after wrapping | Purpose | One-line test |
|---------|-------------------------|---------|---------------|
| **Adapter** | **Different** from the wrapped object | Make an existing interface usable by a client that expects a different one | "The shapes don't match" |
| **Decorator** | **Same** as the wrapped object | Add behaviour (logging, retry, caching) without changing the interface | "Same method, more stuff happens" |
| **Facade** | **New and simpler**, over MANY objects | Hide the complexity of a whole subsystem behind one convenient entry point | "Too many moving parts" |
| **Proxy** | **Same** as the wrapped object | Control *access* (lazy load, auth check, rate limit, remote call) | "Same method, but should you even be calling it?" |
| **Bridge** | Designed up front, both sides vary | Separate abstraction from implementation *before* they exist | "Adapter is a rescue; Bridge is a plan" |

The two the interviewer will actually press you on:

**Adapter vs Facade.** An adapter converts *one* interface into *another specific one that a client already demands*. A facade invents a *new, simpler* interface over *several* classes that nobody demanded. Adapter = **compatibility**; Facade = **convenience**. See [36 — Facade](./36-pattern-facade.md).

**Adapter vs Decorator.** A decorator must keep the same interface, because decorators stack: `new Retry(new Logging(new Cache(x)))` only works if all four are interchangeable. An adapter deliberately *changes* the interface, so adapters don't stack. See [35 — Decorator](./35-pattern-decorator.md).

**Adapter vs Bridge.** Same diagram, opposite timing. Bridge is designed *before* either side exists, so abstraction and implementation vary independently. Adapter is applied *after the fact*, to code you didn't get to design.

---

## Visual / Diagram description

### Diagram 1: The Adapter class structure

```
        ┌──────────────────────────┐
        │      CheckoutService      │  ← CLIENT: knows ONLY the Target.
        │  - gateway: PaymentGateway│    Has never heard of Stripe.
        │  + checkout(order, token) │
        └────────────┬──────────────┘
                     │ depends on
                     ▼
        ┌──────────────────────────┐
        │  «interface» PaymentGateway│ ← TARGET: designed by YOU,
        │  + charge(dollars, token) │    in YOUR domain language.
        │  + refund(id, dollars)    │
        └────────────┬──────────────┘
                     │ implements
        ┌────────────┴───────────────┬────────────────────┐
┌───────▼────────────┐    ┌──────────▼─────────┐  ┌───────▼──────────┐
│   StripeAdapter    │    │   PayPalAdapter    │  │   FakeGateway    │
│     (ADAPTER)      │    │     (ADAPTER)      │  │   (for tests)    │
│ - stripe: StripeSDK│    │ - paypal: PayPalSDK│  │ + charge()       │
│ + charge()         │    │ + charge()         │  │ + refund()       │
│ + refund()         │    │ + refund()         │  └──────────────────┘
└───────┬────────────┘    └──────────┬─────────┘
        │ HAS-A (object adapter)     │ HAS-A
        ▼                            ▼
┌────────────────────┐    ┌────────────────────────┐
│     StripeSDK      │    │       PayPalSDK        │  ← ADAPTEES:
│     (ADAPTEE)      │    │       (ADAPTEE)        │    third-party.
│ .paymentIntents    │    │ + captureOrder(tok,{}) │    You cannot
│    .create({...})  │    │ + refundCapture(id,{}) │    change these.
│ .refunds.create()  │    │                        │
└────────────────────┘    └────────────────────────┘
```

Read it top to bottom. The client points at the *interface*, never at the boxes on the bottom row. The adapters are the only place in the whole codebase where the words `paymentIntents` or `captureOrder` appear — delete `StripeAdapter` and nothing else breaks except one line in `makeGateway()`.

### Diagram 2: The translation happening inside one call

```
CheckoutService              StripeAdapter                  StripeSDK
      │  charge(19.99, "tok_visa") │                             │
      ├───────────────────────────▶│                             │
      │                            │ ══ TRANSLATE IN ══          │
      │                            │ 19.99 dollars → 1999 cents  │
      │                            │ token → payment_method      │
      │                            │ add currency, confirm       │
      │                            │ paymentIntents.create({     │
      │                            │   amount:1999,              │
      │                            │   payment_method:'tok_visa'})│
      │                            ├────────────────────────────▶│
      │                            │ {id:'pi_9x', status:'succeeded'}
      │                            │◀────────────────────────────┤
      │                            │ ══ TRANSLATE OUT ══         │
      │                            │ status!=='succeeded'        │
      │                            │   → throw PaymentError      │
      │                            │ else → new Receipt(...)     │
      │  Receipt{chargeId:'pi_9x'} │                             │
      │◀───────────────────────────┤                             │
```

The adapter does exactly three jobs and you can see all three: **convert the arguments in**, **delegate**, **convert the result (and the errors) out**. If it is doing business logic on top of that, it has stopped being an adapter.

---

## Real world examples

### 1. Shopify — payment gateway abstraction

Shopify merchants can take money through dozens of processors (Shopify Payments, PayPal, Klarna, Affirm, regional processors in India and Brazil). Shopify's checkout does not contain a branch per processor. It defines a payment-gateway contract — authorize, capture, void, refund — and each processor integration is an adapter implementing that contract. The `activemerchant` library (Shopify's, open source) is literally a library of ~100 payment adapters behind one `Gateway` interface. Adding a new processor means writing one adapter class; the checkout flow is untouched.

### 2. Vercel / Next.js — the adapter layer for deployment targets

Next.js runs the same application on AWS Lambda, Node servers, Cloudflare Workers, and Deno. Those runtimes have incompatible request/response objects: Node gives you `IncomingMessage`/`ServerResponse` (streams and callbacks), Workers give you the WHATWG `Request`/`Response` (promises and web streams). Next.js defines its own internal request abstraction and ships **adapters** that convert each runtime's native objects into it. Your `export default function handler(req, res)` never learns which cloud it woke up on.

### 3. Terraform / Pulumi — cloud provider plugins (conceptual)

Infrastructure-as-code tools declare one resource model ("a VM with 4 GB of RAM") and adapt it to AWS's EC2 API, GCP's Compute API, and Azure's ARM API — three completely different request formats, auth schemes, and error vocabularies. Each provider plugin is an adapter, and the core engine that computes the plan/diff contains no cloud-specific code at all.

---

## Trade-offs

| Pro | What it buys you | Con | What it costs you |
|-----|------------------|-----|-------------------|
| **Decoupling** | Business logic depends on your interface, not a vendor's SDK | **More classes** | An interface + N adapters instead of one direct call |
| **Swappable vendors** | Change providers by changing one factory line | **Indirection** | Debugging steps through one more layer |
| **Testability** | Inject a `FakeGateway`; tests never touch the network | **Lowest-common-denominator risk** | A bad target interface hides features you wanted |
| **Single Responsibility** | Translation lives in one class, not across call sites | **Maintenance** | Vendor changes their API → someone updates the adapter |
| **Open/Closed** | Add a vendor by adding a class, not editing existing ones | **Overkill for one vendor** | Never swapping and never testing? It's ceremony |
| **Normalised errors** | Every vendor's exotic failure becomes one `PaymentError` | | |

**Rule of thumb:** Write an adapter the moment a third-party API's shape would otherwise appear in more than one file of your business logic — or the moment you want to unit-test the code that calls it. Those two triggers cover ~95% of real cases. Do NOT write one for code you own and can simply change.

---

## Common interview questions on this topic

### Q1: "What's the difference between Adapter and Facade? They both wrap things."
**Hint:** Adapter **converts** an interface — the client already demands a specific shape (`charge(amount, token)`) and the adaptee doesn't have it. Facade **simplifies** — it invents a brand-new, easier entry point over several collaborating classes, and nobody was demanding that exact shape. Adapter usually wraps *one* object; a facade usually coordinates *many*. Adapter = compatibility. Facade = convenience.

### Q2: "Object adapter or class adapter — which and why?"
**Hint:** Object adapter (composition), essentially always, and in JS you barely have a choice: no multiple inheritance. Composition lets one adapter work with any object that has the adaptee's methods (so you can inject a mock in tests), lets you adapt subclasses of the adaptee for free, and avoids inheriting the adaptee's constructor, state, and network behaviour. Class adapter's only real edge is that it can *override* adaptee methods — rarely what you want.

### Q3: "How do you design the target interface?"
**Hint:** Around your **domain**, not around whichever SDK you happened to integrate first. Ask: what does my application actually need to do? (`charge`, `refund`, `getStatus`). If your interface method is named `createPaymentIntent`, you've just renamed Stripe and gained nothing. Watch out for the lowest-common-denominator trap: don't strip a feature just because one vendor lacks it — expose a capability check instead.

### Q4: "You have 12 payment providers. Won't you have 12 adapters to maintain?"
**Hint:** Yes — and that's the point: the cost is *linear* in providers and *isolated*. Without adapters, each provider adds a branch to every payment-touching method, so cost is providers × call-sites. Also: adapters are individually trivial and independently testable. If several share logic (auth, retries, HTTP), extract a shared `BaseHttpGateway` — but keep the translation per-vendor.

### Q5: "Where does Adapter show up in the Node ecosystem?"
**Hint:** Knex/Sequelize dialect drivers, Passport strategies, Winston/Pino transports, `node-fetch` adapting `http.request` into the browser `fetch` shape, any ORM's "connector" package. Naming a real one out loud is worth a lot.

---

## Practice exercise

### The Two-Provider Storage Adapter

Build a file-storage abstraction with adapters for two very different backends. Target: ~30 minutes. Produce one runnable `storage.js` containing:

1. **The Target interface** — `class FileStorage` with three methods that throw if not overridden: `upload(key, buffer, contentType) -> { url, key, size }`, `download(key) -> Buffer`, `delete(key) -> boolean`.

2. **Two fake adaptee SDKs** (~15 lines each, mimicking the real shapes):
   - `S3SDK` — `putObject({ Bucket, Key, Body, ContentType })`, `getObject({ Bucket, Key })` returning `{ Body }`, `deleteObject({ Bucket, Key })`. PascalCase keys; needs a bucket name everywhere.
   - `LocalDiskSDK` — `writeFile(path, buffer)`, `readFile(path)`, `unlink(path)`. Filesystem paths, no concept of a URL, and it **throws** `ENOENT` on a missing file rather than returning null.

3. **Two adapters** — `S3Adapter` (holds an `S3SDK` + bucket name) and `LocalDiskAdapter` (holds a `LocalDiskSDK` + base directory). Each must translate `key` into the backend's addressing scheme, produce a `url` (`https://<bucket>.s3.amazonaws.com/<key>` vs `file:///<basedir>/<key>`), and **normalise the "not found" case** so both throw the same `FileNotFoundError` you define.

4. **A client** — `class AvatarService` with `setAvatar(userId, buffer)` uploading to `avatars/${userId}.png`, returning the URL. Plus a **`main()` demo** running it against *both* adapters with identical-shaped output, and one download of a missing key showing both backends produce the same `FileNotFoundError`.

**The test you must pass:** grep `AvatarService` for `S3`, `Bucket`, or `writeFile`. One hit means the adapter is leaking — fix the target interface.

---

## Quick reference cheat sheet

- **Adapter** converts one interface into another so two incompatible classes can work together. Also called **Wrapper**.
- **Four participants:** Client (wants the work done) → Target (the interface it's written against) → Adapter (implements Target, holds the Adaptee) → Adaptee (the foreign class with the wrong shape).
- **Design the Target for YOUR domain**, not as a rename of the SDK you integrated first. If your method is called `createPaymentIntent`, you've renamed Stripe and gained nothing.
- **Object adapter (composition, has-a)** beats **class adapter (inheritance, is-a)** — and in JS, single inheritance basically forces the choice anyway.
- **An adapter does exactly three things:** translate the arguments in, delegate, translate the result and the errors back out. Nothing else — no business logic.
- **Normalise errors too** — the half everyone forgets. One vendor returns a status string, another throws; the client should only ever catch one error type.
- **Trigger to write one:** a vendor's vocabulary is about to appear in more than one file, OR you want to unit-test the calling code without a network.
- **Don't adapt code you own** — just change it. Indirection with no payoff is pure cost.
- **Beware the lowest-common-denominator interface** — don't drop a feature just because one vendor lacks it; expose a `supports(feature)` check instead.
- **Adapter vs Decorator:** Adapter *changes* the interface; Decorator *keeps* it and adds behaviour (so decorators stack, adapters don't).
- **Adapter vs Facade:** Adapter converts one interface (compatibility, one object); Facade simplifies a whole subsystem (convenience, many objects).
- **Adapter vs Proxy / Bridge:** Proxy keeps the interface and controls *access*. Bridge has an identical diagram but opposite timing — Bridge is planned up front, Adapter is a rescue after the fact.
- **Node examples:** Knex/Sequelize dialects, Passport strategies, Winston transports, `node-fetch` over `http.request`.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [33 — Prototype Pattern](./33-pattern-prototype.md) — the last of the creational patterns, before we move into structural ones |
| **Next** | [35 — Decorator Pattern](./35-pattern-decorator.md) — the other great wrapper: same interface, extra behaviour |
| **Related** | [36 — Facade Pattern](./36-pattern-facade.md) — the pattern most often confused with Adapter in interviews |
| **Related** | [26 — Composition Over Inheritance](./26-composition-over-inheritance.md) — why the object adapter beats the class adapter |
