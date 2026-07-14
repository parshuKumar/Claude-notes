# 36 — Facade Pattern

## Category: LLD Patterns

---

## What is this?

The **Facade** pattern puts one simple, friendly entry point in front of a complicated subsystem. Behind that entry point there might be six classes talking to each other in a precise order — but the caller sees a single method and doesn't have to know any of it.

The analogy is starting a car. You turn a key (or push a button). What actually has to happen: the ECU wakes up, the fuel pump pressurises the rail, the ignition coil charges, the starter motor engages the flywheel, the crankshaft turns, the injectors fire in sequence, the spark plugs ignite, and once the engine catches, the starter must *disengage* or it'll be destroyed. You do exactly none of that. The key is the facade. And crucially — the mechanic can still open the bonnet and touch every one of those parts directly. **A facade doesn't lock the subsystem away; it just gives you the convenient path.**

---

## Why does it matter?

Without a facade, the complexity of a subsystem leaks into everyone who uses it. Placing an order requires calling InventoryService, then PaymentService, then ShippingService, then NotificationService — **in that order**, with **compensating rollbacks** if step 3 fails after step 2 succeeded. If that orchestration lives in your HTTP controller, then it also lives in your admin panel's "create order for customer" button, and in your Shopify import job, and in your integration tests. Four copies of a delicate sequence. Three of them will drift.

A facade gives that sequence exactly one home.

**In interviews:** Facade is the pattern you name when the interviewer says *"this controller is doing too much"* or *"how do you keep the API layer thin?"* It's also the third member of the structural trio (Adapter, Decorator, Facade) that interviewers ask you to distinguish — and the one most people get wrong, because they think "facade = hiding things" when it actually means "facade = a convenient default path."

**At work:** Every SDK you've ever used is a facade. `fetch()` is a facade. Your `services/` layer, if you have a good one, is a set of facades. It's probably the pattern you'll write most often without ever calling it by name.

---

## The core idea — explained simply

### The Starting-the-Car Analogy

You sit in your car and turn the key. The car starts.

Here is what your key **actually triggered**, in strict order:

1. The **ECU** (engine control unit) boots and runs self-checks on the sensors.
2. The **fuel pump** runs for ~2 seconds to bring the fuel rail up to pressure — *before* anything else, or there's nothing to burn.
3. The **ignition system** charges the coils.
4. The **starter motor** engages the flywheel and cranks the engine, drawing ~200 amps.
5. The **injectors** spray fuel into the cylinders, timed to the crankshaft position sensor.
6. The **spark plugs** fire, in firing order, at exactly the right crank angle.
7. The ECU detects the engine has caught, and **disengages the starter motor** — if it didn't, the spinning engine would destroy it.

Seven subsystems. A strict order. A rollback-ish step at the end (disengage, or you break something expensive). And a driver who knows **one thing**: turn the key.

Now the part everyone forgets: **the bonnet still opens.** A mechanic can put a pressure gauge on the fuel rail, pull a spark plug, or bench-test the starter motor directly. The key didn't *hide* those components — it didn't make them private, it didn't wrap them, it didn't remove them from the car. It just gave the 99% use case a one-move interface.

That is exactly a facade.

| Car | Facade pattern term | In the order example |
|-----|--------------------|----------------------|
| The driver | **Client** | `OrderController` (your HTTP handler) |
| The ignition key / start button | **Facade** | `OrderFacade.placeOrder(...)` |
| Fuel pump, ignition, starter, injectors, ECU | **Subsystem classes** | `InventoryService`, `PaymentService`, `ShippingService`, `NotificationService`, `EmailClient` |
| The precise start-up sequence | The facade's **orchestration logic** | reserve → charge → ship → notify, with compensation |
| The mechanic opening the bonnet | **Direct subsystem access** (still allowed!) | An admin tool calling `InventoryService.adjustStock()` directly |

The one-sentence definition to memorise: **a facade is a convenience, not a wall.**

---

## Key concepts inside this topic

### 1. The problem it solves — the 40-line controller

Here's the "before". This is real code shape — a controller that grew orchestration.

```javascript
// ❌ BAD — an HTTP controller doing five services' worth of choreography
app.post('/orders', async (req, res) => {
  const { userId, items, cardToken, address } = req.body;

  // 1. Check and reserve stock
  const reservations = [];
  try {
    for (const item of items) {
      const stock = await inventoryService.getStock(item.sku);
      if (stock < item.qty) {
        // ...and now unwind whatever we already reserved
        for (const r of reservations) await inventoryService.release(r.id);
        return res.status(409).json({ error: `${item.sku} out of stock` });
      }
      const r = await inventoryService.reserve(item.sku, item.qty);
      reservations.push(r);
    }
  } catch (e) {
    for (const r of reservations) await inventoryService.release(r.id);
    return res.status(500).json({ error: 'Reservation failed' });
  }

  // 2. Price it (tax rules by region — hope you got this right in the other 3 call sites)
  const subtotal = items.reduce((s, i) => s + i.price * i.qty, 0);
  const tax = subtotal * (address.country === 'US' ? 0.0875 : 0.20);
  const total = Math.round((subtotal + tax) * 100) / 100;

  // 3. Charge
  let receipt;
  try {
    receipt = await paymentService.charge(total, cardToken);
  } catch (e) {
    for (const r of reservations) await inventoryService.release(r.id);   // compensate #1
    return res.status(402).json({ error: 'Payment declined' });
  }

  // 4. Ship
  let shipment;
  try {
    shipment = await shippingService.createShipment(userId, items, address);
  } catch (e) {
    await paymentService.refund(receipt.chargeId, total);                 // compensate #2
    for (const r of reservations) await inventoryService.release(r.id);   // compensate #1
    return res.status(500).json({ error: 'Shipping unavailable' });
  }

  // 5. Commit the reservations, notify, email
  for (const r of reservations) await inventoryService.commit(r.id);
  await notificationService.push(userId, 'Order placed!');
  await emailClient.send(await userService.getEmail(userId), 'Your receipt', `Total $${total}`);

  res.json({ orderId: shipment.orderId, total, tracking: shipment.trackingNumber });
});
```

What's actually wrong here:

- **The controller knows the business.** It knows reservations come before payment. It knows US tax is 8.75%. It knows shipping failure means refund *and* release. That is domain knowledge sitting in the HTTP layer.
- **It cannot be reused.** The admin panel needs to place orders too. So does the CSV importer. So does the mobile BFF. Each will copy this, and each copy will get the compensation order slightly wrong.
- **It cannot be tested** without spinning up Express and mocking five services through HTTP.
- **The compensation logic is duplicated and fragile.** Count the `for (const r of reservations) await inventoryService.release(r.id)` lines: four. Add a sixth step and you'll add a fifth.
- **The client is coupled to five classes.** Change any one service's method signature and the controller breaks.

### 2. The structure — who plays which role

| Participant | Role | In the order example |
|-------------|------|----------------------|
| **Facade** | One class with a small number of high-level methods. Knows the subsystem; the subsystem does not know it. | `OrderFacade` |
| **Subsystem classes** | The real workers. Independent, each with a Single Responsibility. **They have no reference to the facade.** | `InventoryService`, `PaymentService`, `ShippingService`, `NotificationService`, `EmailClient` |
| **Client** | Talks to the facade for the common path. May *still* talk to the subsystem directly when it needs to. | `OrderController`, admin tools, jobs |
| **Additional Facade** (optional) | A second facade for a different use case, so one facade doesn't become a god object. | `ReturnsFacade` |

Two rules that define a correct facade:

1. **Dependencies point one way.** Facade → subsystem. If a subsystem class ever needs to call back into the facade, your layering is inverted and you have a cycle.
2. **The facade adds no new functionality.** It only *sequences* what the subsystem already does. If your facade is computing tax with its own algorithm, that logic belongs in a `PricingService` the facade calls. A facade that grows real logic has become a god object — the #1 way this pattern gets abused.

### 3. Full JavaScript implementation

Runnable with `node facade-demo.js`. The subsystem services are deliberately real-shaped: they have their own quirks, their own errors, and no awareness of each other.

```javascript
// ===================================================================
// THE SUBSYSTEM — five independent services. Each is useful on its own.
// None of them knows the others exist. None of them knows the Facade exists.
// ===================================================================

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const id = (p) => `${p}_${Math.random().toString(36).slice(2, 8)}`;

class OutOfStockError extends Error {
  constructor(sku) { super(`SKU ${sku} is out of stock`); this.name = 'OutOfStockError'; this.sku = sku; }
}
class PaymentDeclinedError extends Error {
  constructor(reason) { super(`Payment declined: ${reason}`); this.name = 'PaymentDeclinedError'; }
}

class InventoryService {
  constructor(stock) { this.stock = new Map(Object.entries(stock)); this.reservations = new Map(); }

  async reserve(sku, qty) {
    await sleep(5);
    const have = this.stock.get(sku) ?? 0;
    if (have < qty) throw new OutOfStockError(sku);
    this.stock.set(sku, have - qty);                    // hold the units
    const rid = id('res');
    this.reservations.set(rid, { sku, qty });
    console.log(`  [inventory] reserved ${qty}x ${sku} (${rid})`);
    return rid;
  }
  async commit(rid) {                                   // sale confirmed, units gone for good
    this.reservations.delete(rid);
    console.log(`  [inventory] committed ${rid}`);
  }
  async release(rid) {                                  // compensation: give the units back
    const r = this.reservations.get(rid);
    if (!r) return;
    this.stock.set(r.sku, (this.stock.get(r.sku) ?? 0) + r.qty);
    this.reservations.delete(rid);
    console.log(`  [inventory] RELEASED ${r.qty}x ${r.sku} (${rid})`);
  }
}

class PricingService {
  // Tax rules live HERE, not in the facade and definitely not in the controller.
  static TAX = { US: 0.0875, GB: 0.20, DE: 0.19 };
  price(items, country) {
    const subtotal = items.reduce((s, i) => s + i.price * i.qty, 0);
    const rate = PricingService.TAX[country] ?? 0;
    const tax = subtotal * rate;
    return { subtotal, tax, total: Math.round((subtotal + tax) * 100) / 100 };
  }
}

class PaymentService {
  async charge(amount, token) {
    await sleep(10);
    if (token === 'tok_declined') throw new PaymentDeclinedError('insufficient funds');
    const chargeId = id('ch');
    console.log(`  [payment]   charged $${amount.toFixed(2)} -> ${chargeId}`);
    return { chargeId, amount };
  }
  async refund(chargeId, amount) {
    await sleep(10);
    console.log(`  [payment]   REFUNDED $${amount.toFixed(2)} for ${chargeId}`);
    return { refundId: id('re') };
  }
}

class ShippingService {
  async createShipment(userId, items, address) {
    await sleep(10);
    if (address.country === 'AQ') throw new Error('We do not ship to Antarctica');
    const shipmentId = id('shp');
    console.log(`  [shipping]  created ${shipmentId} to ${address.country}`);
    return { shipmentId, trackingNumber: id('TRK').toUpperCase(), etaDays: 3 };
  }
}

class NotificationService {
  async push(userId, message) {
    await sleep(3);
    console.log(`  [push]      -> ${userId}: "${message}"`);
  }
}

class EmailClient {
  async send(to, subject, body) {
    await sleep(3);
    console.log(`  [email]     -> ${to} | ${subject}`);
  }
}

class UserService {
  constructor(users) { this.users = new Map(Object.entries(users)); }
  async get(userId) {
    const u = this.users.get(userId);
    if (!u) throw new Error(`No such user: ${userId}`);
    return u;
  }
}

// ===================================================================
// THE FACADE — one method the outside world calls. It OWNS the sequence
// and the compensation, and NOTHING else. Notice: no tax maths, no
// stock arithmetic, no SMTP. It only orchestrates.
// ===================================================================
class OrderFacade {
  constructor({ inventory, pricing, payments, shipping, notifications, email, users }) {
    this.inventory = inventory;
    this.pricing = pricing;
    this.payments = payments;
    this.shipping = shipping;
    this.notifications = notifications;
    this.email = email;
    this.users = users;
  }

  /**
   * The ONE public entry point. Everything the caller needs, in one call.
   *
   * The sequence is deliberate:
   *   reserve stock  -> price -> charge -> ship -> commit stock -> notify
   * Reserve BEFORE charging (never take money for something you can't send).
   * Commit stock only AFTER shipping succeeds (the last irreversible step).
   *
   * `undo` is a stack of compensating actions — a hand-rolled Saga. If any
   * step throws, we unwind in reverse order. This is the delicate bit that
   * used to be copy-pasted into four controllers.
   */
  async placeOrder({ userId, items, cardToken, address }) {
    const undo = [];
    try {
      const user = await this.users.get(userId);

      // 1. Reserve every line item.
      for (const item of items) {
        const rid = await this.inventory.reserve(item.sku, item.qty);
        undo.push(() => this.inventory.release(rid));
        item.reservationId = rid;
      }

      // 2. Price it (delegated — the facade does not know tax rates).
      const { subtotal, tax, total } = this.pricing.price(items, address.country);

      // 3. Take the money.
      const receipt = await this.payments.charge(total, cardToken);
      undo.push(() => this.payments.refund(receipt.chargeId, total));

      // 4. Book the shipment.
      const shipment = await this.shipping.createShipment(userId, items, address);

      // 5. Point of no return — commit the stock.
      for (const item of items) await this.inventory.commit(item.reservationId);

      // 6. Tell the human. Notifications are best-effort: a failed email must
      //    NOT roll back a paid, shipped order. So they sit outside `undo`.
      await this._notifyBestEffort(user, shipment, total);

      return {
        orderId: shipment.shipmentId,
        tracking: shipment.trackingNumber,
        etaDays: shipment.etaDays,
        subtotal, tax, total,
      };
    } catch (err) {
      // Unwind in REVERSE — refund before releasing stock, mirroring how we built up.
      console.log(`  [facade]    ! ${err.name}: ${err.message} — rolling back ${undo.length} step(s)`);
      for (const compensate of undo.reverse()) {
        try { await compensate(); }
        catch (e) { console.error(`  [facade]    !! compensation failed: ${e.message}`); }
      }
      throw err;   // the facade normalises the flow, it does not swallow failures
    }
  }

  async _notifyBestEffort(user, shipment, total) {
    try {
      await this.notifications.push(user.id, `Order placed! Tracking ${shipment.trackingNumber}`);
      await this.email.send(user.email, 'Your receipt', `Total: $${total.toFixed(2)}`);
    } catch (e) {
      console.warn(`  [facade]    notification failed (order still valid): ${e.message}`);
    }
  }
}

// ===================================================================
// THE CLIENT — the controller is now 6 lines and knows NOTHING about
// reservations, tax, or compensation. Compare this to the 40-line version.
// ===================================================================
function makeOrderController(orderFacade) {
  return async function handlePost(req, res) {
    try {
      const result = await orderFacade.placeOrder(req.body);   // <-- one call
      res.status(201).json(result);
    } catch (err) {
      const status = err instanceof OutOfStockError ? 409
                   : err instanceof PaymentDeclinedError ? 402
                   : 500;
      res.status(status).json({ error: err.message });
    }
  };
}

// ===================================================================
// DEMO
// ===================================================================
async function main() {
  const subsystem = {
    inventory: new InventoryService({ 'BOOK-1': 5, 'MUG-2': 1 }),
    pricing: new PricingService(),
    payments: new PaymentService(),
    shipping: new ShippingService(),
    notifications: new NotificationService(),
    email: new EmailClient(),
    users: new UserService({ u_1: { id: 'u_1', email: 'ada@example.com' } }),
  };
  const facade = new OrderFacade(subsystem);

  const items = () => [{ sku: 'BOOK-1', qty: 2, price: 12.0 }, { sku: 'MUG-2', qty: 1, price: 8.0 }];

  console.log('\n--- 1. Happy path ---');
  const ok = await facade.placeOrder({
    userId: 'u_1', items: items(), cardToken: 'tok_visa',
    address: { country: 'US', line1: '1 Main St' },
  });
  console.log('  RESULT:', ok);

  console.log('\n--- 2. Payment declined: stock is released, no money taken ---');
  try {
    await facade.placeOrder({
      userId: 'u_1', items: items(), cardToken: 'tok_declined',
      address: { country: 'US', line1: '1 Main St' },
    });
  } catch (e) { console.log('  caught:', e.name); }

  console.log('\n--- 3. Shipping fails AFTER payment: refund + release ---');
  try {
    await facade.placeOrder({
      userId: 'u_1', items: items(), cardToken: 'tok_visa',
      address: { country: 'AQ', line1: 'Research Base' },
    });
  } catch (e) { console.log('  caught:', e.message); }

  console.log('\n--- 4. The bonnet still opens: an admin restocks DIRECTLY ---');
  // The facade did NOT hide InventoryService. Advanced callers still reach in.
  subsystem.inventory.stock.set('MUG-2', 50);
  console.log('  [admin] MUG-2 stock is now', subsystem.inventory.stock.get('MUG-2'));

  // And the HTTP layer is now trivial:
  const controller = makeOrderController(facade);
  console.log('\n--- 5. Via the (6-line) controller ---');
  await controller(
    { body: { userId: 'u_1', items: items(), cardToken: 'tok_visa', address: { country: 'GB' } } },
    { status: (c) => ({ json: (b) => console.log(`  HTTP ${c}`, b) }) }
  );
}

main();
```

**Before vs after, side by side:**

| | Before (no facade) | After (facade) |
|---|---|---|
| Controller length | ~40 lines | ~6 lines |
| Classes the controller couples to | 5 services + tax rules | 1 (`OrderFacade`) |
| Copies of the compensation logic | 4, drifting | 1 |
| Reusable by admin panel / CSV import? | Copy-paste | `new OrderFacade(...)` |
| Unit-testable without HTTP? | No | Yes — construct with fakes, call `placeOrder` |
| Where does "US tax is 8.75%" live? | In the controller | In `PricingService`, where it belongs |

### 4. A facade does NOT hide the subsystem

This is the part beginners get wrong, and it's a genuine interview differentiator.

A facade is **not** encapsulation. It does not make the subsystem private, and it must not try to. Look at step 4 of the demo: an admin tool reaches straight into `InventoryService` to restock. That's *allowed*. That's *correct*.

```javascript
// The 99% path — use the facade.
await orderFacade.placeOrder({ userId, items, cardToken, address });

// The 1% path — a bulk-import job needs to reserve without charging.
// It reaches into the subsystem directly. This is NOT a violation.
for (const item of csvRows) {
  await inventory.reserve(item.sku, item.qty);
}

// An ops script needs to force-refund without touching an order at all.
await payments.refund('ch_abc123', 49.99);
```

The car analogy holds all the way down: the ignition key is the convenient path, but the bonnet is not welded shut. A mechanic can bench-test the starter motor. If you *did* weld the bonnet shut, the car would be unmaintainable — and a facade that truly hides its subsystem forces every unanticipated use case to either (a) bloat the facade with a new method or (b) work around it entirely.

**Where a facade *does* enforce something:** it defines the **sanctioned** path. Teams often add a lint rule saying "HTTP controllers may import facades, not subsystem services" — the *policy* is enforced socially/statically, not by the pattern. The pattern itself only offers convenience.

**Corollary for interviews:** if someone asks "so a facade hides complexity?" — the precise answer is *"it hides complexity from callers who don't want it, without removing access for callers who do."*

### 5. A real Node.js ecosystem example

**`fetch()` — the facade over the HTTP stack.** When you write:
```javascript
const res = await fetch('https://api.example.com/users/1');
const user = await res.json();
```
you have just triggered: DNS resolution, TCP socket creation, a TLS handshake (certificate chain validation, cipher negotiation), HTTP/2 connection pooling and stream multiplexing, header serialisation, chunked transfer decoding, gzip/brotli decompression, redirect following, and character-encoding detection. Node exposes all of those as separate modules — `dns`, `net`, `tls`, `http2`, `zlib`. `fetch()` is a facade that sequences them for the common case. And true to the pattern, **it doesn't hide them**: when you need connection-pool control or a custom TLS agent, you drop down to `undici`'s `Agent` or raw `http.request`. The bonnet opens.

**The AWS SDK's top-level clients.** `new S3Client().send(new PutObjectCommand({...}))` is a facade over credential resolution (env vars → shared config file → EC2 instance metadata → STS assume-role), SigV4 request signing, region endpoint construction, retry with adaptive backoff, and multipart upload chunking. You *can* reach in — `@aws-sdk/signature-v4` is a separate published package you can use alone. Again: convenient path, not a wall.

**`express()` itself.** `const app = express()` is a facade over `http.createServer`, a router, a middleware stack, and a view engine. And `app.listen()`? It's literally `http.createServer(this).listen(...)` — a two-line facade method. Meanwhile `req` and `res` are still the raw Node `IncomingMessage`/`ServerResponse` underneath, so you can always reach past Express to `res.socket`.

**Others:** Mongoose over the MongoDB driver, `nodemailer`'s `transporter.sendMail()` over SMTP/OAuth/MIME encoding, `jsonwebtoken`'s `sign()` over base64url + HMAC + JSON serialisation.

### 6. When NOT to use it / how it's abused

**The god-object facade.** The #1 abuse. `OrderFacade` starts with `placeOrder()`. Then someone adds `cancelOrder()`, `refundOrder()`, `updateAddress()`, `applyCoupon()`, `exportToCsv()`, `recalculateTax()`. Two thousand lines later it's the thing you were trying to avoid. **Fix:** multiple small facades, one per coherent use case (`OrderFacade`, `ReturnsFacade`, `SubscriptionFacade`). The GoF book calls this "Additional Facades" and it's the sanctioned escape hatch.

**Facade with logic in it.** The facade's job is **sequencing**, not computing. The moment `OrderFacade` contains `const tax = subtotal * 0.0875`, you have business logic in the coordination layer — and now `PricingService` and `OrderFacade` disagree about tax. Push it down.

**A facade over one class.** If your "facade" wraps a single service and just forwards calls, it's not a facade — it's dead indirection. (It might be a legitimate **Adapter** if it's converting the interface, or a **Proxy** if it's controlling access. If it's neither, delete it.)

**The pass-through facade.** If `facade.reserve(sku, qty)` just calls `inventory.reserve(sku, qty)` with no orchestration, no simplification, and no value added, every method like that is pure cost. A facade earns its existence by turning N calls into 1.

**Don't facade a subsystem that's already simple.** Two classes and three calls doesn't need a coordination layer. Wait until the pain is real.

**Watch for hidden coupling.** A facade doesn't reduce the *number* of dependencies in the system — it just relocates them. Your controller now depends on one class instead of five, which is a real win for the controller. But `OrderFacade` still depends on five services, and if it becomes the only place anyone calls them from, it becomes a change bottleneck. That's usually a fine trade; just know you made it.

### 7. Related patterns and how they differ (the #1 interview follow-up)

| Pattern | What it wraps | What it produces | Intent |
|---------|--------------|------------------|--------|
| **Facade** | **Many** objects | A **new, simpler** interface | **Convenience** — one easy path to a common workflow |
| **Adapter** | Usually **one** object | A **specific, pre-existing** interface the client demands | **Compatibility** — the shapes don't match |
| **Decorator** | **One** object | The **same** interface, plus behaviour | **Enhancement** — same call, extra work |
| **Proxy** | **One** object | The **same** interface | **Access control** — should you even reach it? |
| **Mediator** | **Many** objects | Nothing new for outsiders | **Decoupling peers** — the objects talk *through* it |
| **Singleton** | — | — | Facades are often *implemented* as singletons; don't confuse the two |

The three the interviewer will press on:

**Facade vs Adapter.** Both wrap. But: (a) an **adapter's target interface is dictated by an existing client** — the client already says `charge(amount, token)` and the SDK doesn't have it. A **facade invents a brand-new interface** that nobody was demanding; it exists purely because the workflow was tedious. (b) An adapter typically wraps **one** adaptee; a facade coordinates **several** subsystem classes. (c) Adapter = *"I can't use this."* Facade = *"I can use this, but it takes 40 lines."* See [34 — Adapter](./34-pattern-adapter.md).

**Facade vs Mediator.** These are genuinely close and it's a great interview question. Both sit in the middle of several objects. The difference is **who knows whom**:
- **Facade:** one-directional. The facade knows the subsystem; the subsystem classes have **never heard of the facade** and would work perfectly without it. Remove the facade and everything still functions — callers just have more typing to do.
- **Mediator:** bidirectional. The colleagues **know about the mediator** and are *designed to communicate through it* instead of with each other. Remove the mediator and the colleagues cannot talk at all. Think of a chat room: `User.send()` calls `chatRoom.broadcast()`. The users are *dependent* on the mediator.

So: **Facade simplifies access to a subsystem (subsystem is unaware). Mediator centralises communication between peers (peers are aware).** See [48 — Mediator](./48-pattern-mediator.md).

**Facade vs Decorator.** A decorator keeps the **same** interface and adds behaviour to **one** object, and decorators **stack**. A facade creates a **new** interface over **many** objects, and facades don't stack. See [35 — Decorator](./35-pattern-decorator.md).

---

## Visual / Diagram description

### Diagram 1: Before and after

```
BEFORE — the client is coupled to the whole subsystem
════════════════════════════════════════════════════

  ┌──────────────────┐
  │ OrderController  │───────┬────────┬────────┬────────┬────────┐
  │  (40 lines of    │       │        │        │        │        │
  │   orchestration) │       │        │        │        │        │
  └──────────────────┘       ▼        ▼        ▼        ▼        ▼
                        ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐
  ┌──────────────────┐  │Invent- ││Payment ││Shipping││Notific-││ Email  │
  │  AdminPanel      │─▶│  ory   ││Service ││Service ││ ation  ││ Client │
  │ (same 40 lines,  │  │Service ││        ││        ││Service ││        │
  │  copy-pasted)    │  └────────┘└────────┘└────────┘└────────┘└────────┘
  └──────────────────┘       ▲        ▲        ▲
                             │        │        │
  ┌──────────────────┐       │        │        │
  │  CsvImportJob    │───────┴────────┴────────┘
  │ (40 lines again, │
  │  order slightly  │      3 clients × 5 services = 15 edges.
  │  wrong)          │      3 copies of the compensation logic.
  └──────────────────┘


AFTER — one facade, one home for the sequence
═════════════════════════════════════════════

  ┌──────────────────┐
  │ OrderController  │──┐
  │    (6 lines)     │  │
  └──────────────────┘  │
                        │      ┌────────────────────────────────┐
  ┌──────────────────┐  │      │        OrderFacade              │
  │  AdminPanel      │──┼─────▶│                                 │
  └──────────────────┘  │      │  + placeOrder({...})            │
                        │      │      1. inventory.reserve()     │
  ┌──────────────────┐  │      │      2. pricing.price()         │
  │  CsvImportJob    │──┘      │      3. payments.charge()       │
  └──────────────────┘         │      4. shipping.createShipment()│
           │                   │      5. inventory.commit()      │
           │                   │      6. notify + email          │
           │                   │      ✗ on error → undo.reverse()│
           │                   └───┬───┬────┬────┬────┬──────────┘
           │                       │   │    │    │    │
           │                       ▼   ▼    ▼    ▼    ▼
           │              ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐
           │              │Invent- ││Payment ││Shipping││Notific-││ Email  │
           └─────────────▶│  ory   ││Service ││Service ││ ation  ││ Client │
             direct access│Service ││        ││        ││Service ││        │
             STILL ALLOWED└────────┘└────────┘└────────┘└────────┘└────────┘
             (the bonnet
              still opens)      THE SUBSYSTEM — none of these five classes
                                has any idea the OrderFacade exists.
```

Two things to notice, and say them out loud on a whiteboard:
1. The arrows from the facade point **down only**. No subsystem class points back up. That one-way flow is what makes it a facade and not a mediator.
2. The dashed path from `CsvImportJob` straight into `InventoryService` is **not a bug**. The facade is the convenient path, not the only path.

### Diagram 2: The sequence inside `placeOrder()`, including rollback

```
Controller   OrderFacade    Inventory   Pricing   Payment   Shipping   Notify
    │             │             │          │         │         │          │
    │ placeOrder()│             │          │         │         │          │
    ├────────────▶│             │          │         │         │          │
    │             │ reserve()   │          │         │         │          │
    │             ├────────────▶│  push undo: release(rid)     │          │
    │             │◀────rid─────┤          │         │         │          │
    │             │ price()     │          │         │         │          │
    │             ├────────────────────────▶│        │         │          │
    │             │◀─────{total: 40.10}─────┤        │         │          │
    │             │ charge(40.10)          │         │         │          │
    │             ├─────────────────────────────────▶│         │          │
    │             │◀────receipt──── push undo: refund(chargeId)│          │
    │             │ createShipment()       │         │         │          │
    │             ├───────────────────────────────────────────▶│          │
    │             │                        │         │      ✗ THROWS      │
    │             │◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤          │
    │             │                        │         │         │          │
    │             │   ══ ROLLBACK: undo.reverse() ══           │          │
    │             │ refund(chargeId)       │         │         │          │
    │             ├─────────────────────────────────▶│         │          │
    │             │ release(rid)           │         │         │          │
    │             ├────────────▶│          │         │         │          │
    │◀──throw─────┤             │          │         │         │          │
    │ HTTP 500    │             │          │         │         │          │
    ▼             ▼             ▼          ▼         ▼         ▼          ▼
```

The compensating actions unwind in **reverse** order — refund before release, mirroring how they were pushed onto the stack. This is a hand-rolled Saga, and it is *the* reason the sequence deserves its own class: getting this right once is hard, getting it right in four copy-pasted controllers is impossible.

---

## Real world examples

### 1. Stripe's SDK — `stripe.checkout.sessions.create()`

One method call gives you a hosted payment page. Behind it: a customer object may be created, a payment intent is initialised, line items are validated against your product catalogue, currency and tax rules are resolved (Stripe Tax), 3-D Secure requirements are evaluated per region, and a signed session URL is generated. Stripe *also* exposes every one of those as its own API resource — `customers`, `paymentIntents`, `taxRates`, `prices` — so an advanced integrator can build the flow by hand. Exactly the facade contract: one easy path, subsystem still reachable.

### 2. Node's `fetch()` over the HTTP stack

Covered in detail above: `fetch()` sequences DNS, TCP, TLS, HTTP/2 multiplexing, decompression, and redirect handling — all of which Node exposes separately as `dns`, `net`, `tls`, `http2`, `zlib`. When `fetch()` isn't enough (custom agents, connection pool tuning, raw sockets), you drop to `undici` or `http.request`. The facade never blocked you.

### 3. Kubernetes' `kubectl apply -f deploy.yaml`

One command. Behind it: the YAML is parsed and validated against the OpenAPI schema, credentials are resolved from kubeconfig, a three-way strategic merge patch is computed against the live object and the last-applied annotation, the request is authenticated (TLS client certs / OIDC / token), authorised (RBAC), passed through admission webhooks and mutating controllers, then persisted to etcd — after which a dozen controllers reconcile it. And `kubectl` deliberately keeps the bonnet open: `kubectl get --raw`, `kubectl proxy`, and the client-go library all let you talk to the API server directly. Facade, not wall.

---

## Trade-offs

| Pros | What it buys you |
|------|------------------|
| **Thin clients** | Your controller drops from 40 lines to 6 |
| **One home for the workflow** | The delicate sequence + compensation exists once, not four times |
| **Decoupling** | Clients depend on 1 class, not 5. Refactor a service freely behind the facade |
| **Reusability** | The admin panel, the CSV job, and the mobile BFF all call the same method |
| **Testability** | Construct the facade with fakes; test the whole workflow with no HTTP, no network |
| **Discoverability** | A new engineer reads `OrderFacade` and understands the business flow in one file |
| **Escape hatch preserved** | Advanced callers still reach the subsystem — no capability is lost |

| Cons | What it costs you |
|------|-------------------|
| **God-object risk** | The single most common failure. Facades accrete methods until they're 2000 lines |
| **Extra layer** | One more file, one more hop when debugging |
| **Coupling relocated, not removed** | The facade still depends on 5 services — it becomes a change bottleneck |
| **Can hide real cost** | `placeOrder()` looks cheap; it's actually 6 network calls. Callers stop thinking about latency |
| **Tempting to over-simplify** | If the facade only exposes the common path, the 1% use case gets bolted on badly |
| **Not a security boundary** | It doesn't *prevent* subsystem access, so don't use it to enforce anything |

**The sweet spot:** One facade **per coherent use case**, not per subsystem. `OrderFacade.placeOrder()` and `ReturnsFacade.processReturn()` — two small classes — beat one `CommerceFacade` with 14 methods. The facade **orchestrates and nothing else**: if you catch it doing arithmetic, computing tax, or formatting HTML, push that down into a subsystem class where it belongs.

---

## Common interview questions on this topic

### Q1: "What's the difference between Facade and Adapter?"
**Hint:** **Intent and cardinality.** Adapter converts an interface **because the client already demands a specific shape** and the wrapped object doesn't have it — it's about *compatibility*, and it usually wraps **one** object. Facade invents a **brand-new, simpler** interface that nobody demanded, over **several** collaborating classes — it's about *convenience*. The one-liner: Adapter = "I *can't* use this." Facade = "I *can* use this, but it takes 40 lines."

### Q2: "Does a facade hide the subsystem?"
**Hint:** **No — and this is the point most candidates miss.** A facade provides the *convenient* path; it does not make the subsystem inaccessible. An admin tool or a bulk-import job can still call `InventoryService.reserve()` directly, and that's correct and intended. Compare the car: the ignition key doesn't weld the bonnet shut. If you want to *enforce* that nobody bypasses the facade, that's a module-boundary/lint-rule/access-modifier concern, not something the pattern gives you.

### Q3: "Facade vs Mediator — they both sit between multiple objects."
**Hint:** **Who knows whom.** Facade is **one-directional**: the facade knows the subsystem; the subsystem classes have never heard of the facade and work fine without it. Mediator is **bidirectional**: the colleagues hold a reference to the mediator and are *designed* to talk through it — remove the mediator and they can't communicate at all (a chat room where `user.send()` calls `room.broadcast()`). Facade simplifies *access*; Mediator centralises *communication between peers*.

### Q4: "How do you stop a facade from becoming a god object?"
**Hint:** Three rules. (1) **One facade per use case**, not per subsystem — `OrderFacade`, `ReturnsFacade`, `SubscriptionFacade`, each small. GoF explicitly sanctions "Additional Facades." (2) **The facade orchestrates, it does not compute.** If it contains tax maths, that belongs in `PricingService`. (3) **Cap it** — if it's past ~5 public methods or ~200 lines, split it. The smell: unrelated methods sharing a class only because they touch the same services.

### Q5: "Where would you *not* use a Facade?"
**Hint:** When the subsystem is already simple (two classes, three calls — a coordination layer is pure cost); when it would wrap a single class with no orchestration (that's dead indirection, or it's secretly an Adapter/Proxy); when the caller genuinely needs fine-grained control (a facade would just be a leaky pass-through); and when you'd be tempted to use it as a *security* boundary — it isn't one, since the subsystem stays reachable.

---

## Practice exercise

### The Video Upload Facade

You're building the upload flow for a video platform. Target: ~35 minutes. Produce one runnable `upload.js`.

**1. Write the subsystem — five independent classes**, each ~10 lines, each unaware of the others:
- `StorageService` — `async putObject(key, buffer) -> { url }`, `async deleteObject(key)`
- `TranscodingService` — `async submit(sourceUrl) -> jobId`, `async getStatus(jobId) -> 'QUEUED'|'DONE'|'FAILED'`, `async cancel(jobId)`. Make it fail for any file whose buffer length exceeds some limit.
- `ThumbnailService` — `async generate(sourceUrl) -> thumbUrl`
- `MetadataRepository` — `async save(video)`, `async updateStatus(videoId, status)`, `async delete(videoId)`
- `SearchIndexer` — `async index(video)` — but make this one **throw ~30% of the time**

**2. Write `VideoUploadFacade.upload({ userId, filename, buffer })`** which must:
- Store the raw file (push a compensating `deleteObject`)
- Save metadata with status `PROCESSING` (push a compensating `delete`)
- Submit a transcoding job (push a compensating `cancel`)
- Generate a thumbnail
- Flip metadata status to `READY`
- Index it for search — **best effort**: a failed search index must **not** roll back a perfectly good uploaded video. Log a warning and carry on.
- On any *other* failure, unwind the compensations in reverse and rethrow.
- Return `{ videoId, playbackUrl, thumbUrl }`

**3. Write the "before"** — a `uploadControllerBad(req, res)` that does all of the above inline, with the compensation logic copy-pasted. Count its lines. Then write `uploadControllerGood(req, res)` using the facade. Put the line counts in a comment.

**4. In `main()`, demonstrate all four:**
- Happy path
- Transcoding failure → prove the stored object and the metadata row were both cleaned up
- Search-index failure → prove the video is **still READY** (best-effort steps don't roll back)
- **Direct subsystem access:** an admin script that calls `TranscodingService.submit()` on an already-uploaded video to re-encode it — *without going through the facade*. Add a comment explaining why this is allowed and what it proves about facades.

**The question to answer in a closing comment:** you now want a `VideoUploadFacade.reprocess(videoId)` method. Do you add it to this facade, or create a second one? Justify it in two sentences.

---

## Quick reference cheat sheet

- **Facade** = one simple entry point in front of a complicated subsystem. Turn the key; don't sequence the fuel pump yourself.
- **Four participants:** Client → Facade → Subsystem classes (which are **unaware of the facade**) → optional Additional Facades.
- **Dependencies point one way.** Facade → subsystem, never back. A subsystem class calling the facade means you've built a cycle, not a facade.
- **A facade does NOT hide the subsystem.** It's the *convenient* path, not the *only* path. The bonnet still opens.
- **A facade orchestrates; it does not compute.** Tax maths in your facade means the logic escaped from `PricingService`.
- **The value is in the sequence.** N calls in a strict order, with compensation on failure → 1 call. That's the win.
- **Compensation unwinds in reverse** — refund before releasing stock. Keep an `undo` stack; it's a hand-rolled Saga.
- **Best-effort steps sit outside the undo stack** — a failed confirmation email must never roll back a paid, shipped order.
- **God-object is the #1 abuse.** Prevent it with **one facade per use case**, capped at ~5 methods.
- **A "facade" over one class is not a facade** — it's dead indirection, or it's really an Adapter or a Proxy.
- **Facade vs Adapter:** Facade = "I *can* use this, but it's tedious" (convenience, many objects, new interface). Adapter = "I *can't* use this" (compatibility, one object, an interface the client already demands).
- **Facade vs Mediator:** Facade's subsystem doesn't know it exists (one-way). Mediator's colleagues *depend* on it to talk to each other (two-way).
- **Facade vs Decorator:** Decorator keeps the same interface on one object and stacks. Facade makes a new interface over many and doesn't.
- **Node examples:** `fetch()` over dns/net/tls/http2/zlib; the AWS SDK's `S3Client`; `express()` over `http.createServer`; `mongoose` over the Mongo driver.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [35 — Decorator Pattern](./35-pattern-decorator.md) — wraps one object, keeps its interface; Facade wraps many and invents a new one |
| **Next** | [37 — Proxy Pattern](./37-pattern-proxy.md) — the last of the "wrapper" structural patterns: same interface, but controls access |
| **Related** | [34 — Adapter Pattern](./34-pattern-adapter.md) — the pattern most often confused with Facade; convert vs simplify |
| **Related** | [48 — Mediator Pattern](./48-pattern-mediator.md) — also sits between many objects, but the colleagues *know about it* and depend on it |
