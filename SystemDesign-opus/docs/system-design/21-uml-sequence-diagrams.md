# 21 — UML Sequence Diagrams — How to Model Interactions
## Category: LLD Fundamentals

---

## What is this?

A **sequence diagram** is a picture of a conversation. It shows *who talks to whom, in what order, over time* — and what happens when someone doesn't answer.

Think of a group chat transcript, but drawn vertically: each participant gets a column, time flows downward, and every message is an arrow from one column to another. Where a class diagram tells you the **nouns** of your system (`Order`, `Payment`, `Inventory`), a sequence diagram tells you the **verbs over time** (`reserve stock`, `charge card`, `roll back`, `emit event`).

---

## Why does it matter?

**If you don't understand this**, you end up with designs that look tidy on paper and explode in production. You can draw a beautiful class diagram with `OrderService`, `PaymentGateway`, and `InventoryService` boxes — and still have no idea what happens when the payment succeeds but the database write fails. Structure without flow hides every interesting bug.

**In an interview**, the moment you draw a sequence diagram you signal three things at once:
1. You know the **order of operations** (you reserve stock *before* charging the card, not after).
2. You know the **failure paths** (what rolls back, and who rolls it back).
3. You know **what's synchronous and what isn't** (the caller waits for payment; it does *not* wait for the confirmation email).

Interviewers probe exactly there. "What if the payment gateway times out?" is the single most common LLD follow-up, and a sequence diagram is the only artifact that has an answer built into it.

**In real work**, sequence diagrams are how you review a distributed flow before writing code, how you onboard someone into a 6-service checkout path, and how you argue for making something async. A whiteboard sketch that shows the caller *not waiting* is worth ten paragraphs of prose.

---

## The core idea — explained simply

### The Restaurant Order Analogy

You sit down at a restaurant. Here's what happens, in order:

1. You tell the **waiter** you want the steak. You *wait* — you don't leave the table.
2. The waiter walks to the **kitchen** and shouts the order. He *waits* for the chef to acknowledge it.
3. The chef checks the **fridge** for steak. If there's none, he tells the waiter immediately and the waiter comes back to you: "Sorry, we're out."
4. If there is steak, the chef starts cooking, and the waiter comes back and says "It's coming."
5. Separately, the chef drops a ticket in a box for the **dishwasher**. He does *not* wait for the dishwasher to do anything. The dishwasher picks tickets up whenever he's free.

That's a sequence diagram. Every element maps directly:

| Restaurant thing | Sequence diagram thing | Meaning |
|---|---|---|
| You, waiter, chef, fridge, dishwasher | **Participants** (columns) | The objects/services involved |
| The vertical line under each person | **Lifeline** (dashed) | That participant existing over time |
| Waiter actively working on your order | **Activation bar** (thin rectangle) | This participant is busy doing this call |
| "I want the steak" (you wait) | **Synchronous message** (solid line, filled arrowhead) | Caller blocks until it gets an answer |
| "It's coming" (waiter answers you) | **Return** (dashed line, open arrowhead) | The reply travelling back |
| Chef drops a ticket in a box | **Asynchronous message** (solid line, open arrowhead) | Caller does *not* wait |
| "We're out of steak" → a different outcome | **`alt` fragment** | An if/else branch in the flow |
| The chef quietly checking his own notes | **Self-call** (arrow looping back) | An object calling its own method |
| A temp waiter hired for the rush, sent home after | **Creation / destruction (the X)** | Object is constructed / disposed |

The whole point of the diagram is that **time is the vertical axis**. Anything drawn lower happened later. That's it. Once you internalise that, you can read any sequence diagram in the world.

---

## Key concepts inside this topic

### 1. Participants and lifelines

Each participant is a box at the top. Below it hangs a **dashed vertical line** — the lifeline — representing that object existing as time passes.

```
┌────────┐   ┌─────────────────┐   ┌──────────────┐
│ Client │   │ OrderController │   │ OrderService │
└───┬────┘   └────────┬────────┘   └──────┬───────┘
    ┊                 ┊                   ┊
    ┊                 ┊                   ┊        time
    ┊                 ┊                   ┊          │
    ┊                 ┊                   ┊          ▼
```

Rules that matter:
- Put participants **left to right in the order they first get called**. Arrows then flow mostly rightward and downward, which is readable. Zig-zagging arrows are the #1 sign of a badly ordered diagram.
- Name them with the **role**, not the file path: `PaymentGateway`, not `src/infra/stripe/StripeClientImpl`.
- An **actor** (a human) is drawn as a stick figure, or just as the leftmost box called `Client` / `User`.

### 2. Activation bars

An **activation bar** (formally an *execution occurrence*) is a thin rectangle drawn *on top of* the lifeline. It means: "this object is currently on the call stack, doing work."

```
    ┌────────┐
    │ Client │
    └───┬────┘
        ┊
       ┌┴┐   ← bar starts when the call arrives
       │ │
       │ │   ← the object is busy for this whole vertical span
       └┬┘   ← bar ends when it returns
        ┊
```

Nested calls create **stacked** bars — if `OrderService` calls `InventoryService` while it's already active, you get a bar inside a bar. This is how you *see* the call stack.

### 3. Synchronous call vs return

- **Synchronous message**: solid line, **filled** (closed) arrowhead `────▶`. "I am calling you and I will wait until you answer."
- **Return**: **dashed** line, **open** arrowhead `◀- - -`. "Here's your answer."

```
 OrderService                InventoryService
     ┊                              ┊
     ├──── reserve(sku, qty) ──────▶┤      solid + FILLED head = sync call
    ┌┴┐                            ┌┴┐
    │ │                            │ │
    │ │◀- - - reservationId - - - -└┬┘      dashed + OPEN head = return
    └┬┘                             ┊
     ┊                              ┊
```

In JavaScript, a synchronous message is simply `const id = await inventory.reserve(...)`. The `await` **is** the activation bar: your function is suspended, waiting.

> Careful: in UML, "synchronous" means **the caller waits for the reply**. An `await`ed Promise counts as synchronous in this sense, even though it doesn't block the Node event loop. What matters for the diagram is: *does the caller continue only after the answer arrives?*

### 4. Asynchronous message

- **Asynchronous message**: solid line, **open** arrowhead `────▷`. "I'm handing this off. I'm not waiting. There may never be a reply."

```
 OrderService                    EventBus
     ┊                              ┊
     ├──── publish(OrderPlaced) ───▷┤      solid + OPEN head = fire and forget
    ┌┴┐                             ┊
    │ │  ← keeps going immediately; NO return arrow ever comes back
    └┬┘                             ┊
```

**The absence of a return arrow is the whole message.** If you draw a return arrow, you've said "the caller waits." If you don't, you've said "the caller moved on." This single visual detail is what separates `await emailService.send()` (checkout now blocks on an SMTP server — terrible) from `bus.publish(...)` (checkout returns in 200 ms — correct).

### 5. Self-call

An object calling its own method: the arrow leaves the lifeline and loops straight back into it, one step lower. It stacks a new activation bar on top of the existing one.

```
 OrderService
     ┊
    ┌┴┐
    │ ├──┐ validate(dto)      ← leaves...
    │ │  │
    │ │◀─┘                    ← ...and comes right back, one notch down
    │ │
    └┬┘
```

Use self-calls sparingly. Draw one only when the private method is *interesting* — a validation that can throw, a fee calculation that's the point of the discussion. Every trivial private helper you draw is noise.

### 6. Object creation and destruction (the X)

- **Creation**: the arrow points at the participant's **box**, and the box is drawn **lower down** — at the vertical position where the object comes into existence. Label the arrow `«create»` or `new Order(...)`.
- **Destruction**: a big **X** at the bottom of the lifeline. The lifeline stops there. Label the incoming arrow `«destroy»` if something explicitly killed it.

```
 OrderService
     ┊
    ┌┴┐                    ┌───────┐
    │ ├─── «create» ──────▶│ Order │   ← box appears HERE, not at the top
    │ │                    └───┬───┘
    │ │                        ┊
    │ ├─── addItem(...) ──────▶┤
    │ │                       ┌┴┐
    │ │                       └┬┘
    │ ├─── «destroy» ─────────▶┼
    └┬┘                        X       ← lifeline ends
     ┊
```

In JS you rarely draw destruction (the garbage collector handles it), but it's genuinely useful for anything with a *closable* lifecycle: a DB transaction, a WebSocket connection, a file handle, a Redis lock. Drawing the X on a `DbTransaction` lifeline shows *exactly* where the commit or rollback happens.

### 7. Combined fragments — `alt`, `opt`, `loop`, `par`

A **combined fragment** is a labelled box drawn around part of the diagram that changes how those messages execute. Four carry 95% of the weight.

**`alt` — if / else.** Two or more compartments separated by a dashed horizontal line. Each has a `[guard]`. Exactly one runs.

```
┌─ alt ──────────────────────────────────────────┐
│ [payment.status == "CAPTURED"]                 │
│    OrderService ──── markPaid() ─────▶ Repo    │
├ - - - - - - - - - - - - - - - - - - - - - - - -┤
│ [else]                                         │
│    OrderService ──── release() ──▶ Inventory   │
│    OrderService ──── markFailed() ───▶ Repo    │
└────────────────────────────────────────────────┘
```

**`opt` — a plain `if` with no else.** One compartment, one guard. It runs or it doesn't.

```
┌─ opt ──────────────────────────────────┐
│ [user.hasCoupon]                       │
│   OrderService ── applyDiscount() ──▶  │
└────────────────────────────────────────┘
```

**`loop` — repetition.** Guard written as `[for each item]` or `loop(1, n)`.

```
┌─ loop ─────────────────────────────────────────┐
│ [for each item in cart]                        │
│   OrderService ── reserve(item) ──▶ Inventory  │
│   OrderService ◀- - - ok - - - - -  Inventory  │
└────────────────────────────────────────────────┘
```

**`par` — parallel.** Compartments separated by dashed lines, but **all of them run**, concurrently, in no guaranteed order. This is `Promise.all()`.

```
┌─ par ──────────────────────────────────────────┐
│   OrderService ── getUser() ──────▶ UserSvc    │
├ - - - - - - - - - - - - - - - - - - - - - - - -┤
│   OrderService ── getPrices() ────▶ PricingSvc │
└────────────────────────────────────────────────┘
        both fire together; we wait for both
```

Two more you'll occasionally meet: `break` (like `opt`, but if the guard is true the *enclosing* fragment is abandoned — an early exit) and `critical` (this region must not be interleaved — a mutex or lock).

---

## Visual / Diagram description

### Diagram 1 — Synchronous order placement, with a payment-failure branch

This is the one to practise until you can draw it in 90 seconds. Time flows **down**. Filled arrowheads are synchronous calls; dashed arrows are returns.

```
 Client    OrderController  OrderService  InventoryService  PaymentGateway  OrderRepository   EventBus
   ┊             ┊               ┊               ┊                ┊               ┊              ┊
   ├─POST /orders▶┤              ┊               ┊                ┊               ┊              ┊
  ┌┴┐            ┌┴┐             ┊               ┊                ┊               ┊              ┊
  │ │            │ ├──┐ validateBody(dto)        ┊                ┊               ┊              ┊
  │ │            │ │◀─┘  (self-call)             ┊                ┊               ┊              ┊
  │ │            │ ├─placeOrder()─▶┤             ┊                ┊               ┊              ┊
  │ │            │ │              ┌┴┐            ┊                ┊               ┊              ┊
  │ │            │ │              │ ├──«create» new Order(PENDING)─────────┐      ┊              ┊
  │ │            │ │              │ │            ┊                ┊    ┌───▼───┐  ┊              ┊
  │ │            │ │              │ │            ┊                ┊    │ Order │  ┊              ┊
  │ │            │ │              │ │            ┊                ┊    └───┬───┘  ┊              ┊
  │ │  ┌─ loop [for each line item] ─────────────────────────────────────┐ ┊      ┊              ┊
  │ │  │         │ │              │ ├─reserve(sku)▶┤                ┊    │ ┊      ┊              ┊
  │ │  │         │ │              │ │             ┌┴┐               ┊    │ ┊      ┊              ┊
  │ │  │         │ │              │ │◀- resId - - └┬┘               ┊    │ ┊      ┊              ┊
  │ │  └───────────────────────────────────────────────────────────────┘ ┊      ┊              ┊
  │ │            │ │              │ │             ┊                ┊      ┊      ┊              ┊
  │ │            │ │              │ ├──── charge(amount, card) ───▶┤      ┊      ┊              ┊
  │ │            │ │              │ │             ┊               ┌┴┐     ┊      ┊              ┊
  │ │            │ │              │ │◀- - PaymentResult - - - - - └┬┘     ┊      ┊              ┊
  │ │            │ │              │ │             ┊                ┊      ┊      ┊              ┊
  │ │  ┌─ alt ───────────────────────────────────────────────────────────────────────────────┐  ┊
  │ │  │ [result.status == CAPTURED]              ┊                ┊      ┊      ┊           │  ┊
  │ │  │         │ │              │ ├── markPaid() ───────────────────────▶┤      ┊           │  ┊
  │ │  │         │ │              │ ├── save(order) ──────────────────────────────▶┤          │  ┊
  │ │  │         │ │              │ │             ┊                ┊      ┊       ┌┴┐         │  ┊
  │ │  │         │ │              │ │◀- - - - - - - - - ok - - - - - - - - - - - -└┬┘         │  ┊
  │ │  │         │ │              │ ├── publish(OrderPlaced) ───────────────────────────────────▷┤
  │ │  │         │ │              │ │        ASYNC — open head, NO return arrow    ┊           │  ┊
  │ │  │◀- 201 {orderId} - - - - -│ │             ┊                ┊      ┊       ┊           │  ┊
  │ │  ├ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -┤  ┊
  │ │  │ [else — declined or timed out]           ┊                ┊      ┊       ┊           │  ┊
  │ │  │  ┌─ loop [for each reservationId] ──────────────────────────────────┐    ┊           │  ┊
  │ │  │  │      │ │              │ ├─release(resId)▶┤              ┊    │   ┊    ┊           │  ┊
  │ │  │  │      │ │              │ │◀- - released - ┤              ┊    │   ┊    ┊           │  ┊
  │ │  │  └─────────────────────────────────────────────────────────────┘    ┊           │  ┊
  │ │  │         │ │              │ ├── save(order = FAILED) ─────────────────────▶┤        │  ┊
  │ │  │         │ │              │ │◀- - - - - - - - - ok - - - - - - - - - - - - ┤        │  ┊
  │ │  │         │ │              │ ├─ throw PaymentDeclinedError ─┐             ┊        │  ┊
  │ │  │◀- 402 {error} - - - - - -│ │             ┊                ┊      ┊       ┊        │  ┊
  │ │  └───────────────────────────────────────────────────────────────────────────────────┘  ┊
  └┬┘            └┬┘             └┬┘             ┊                ┊      ┊       X            ┊
   ┊              ┊               ┊              ┊                ┊      ┊                    ┊
```

**What the diagram is telling you, in prose:**

The client makes one HTTP call and *waits*. The controller validates (a self-call), then delegates to the service. The service creates an `Order` — note the `Order` box appears **partway down**, because it didn't exist before that moment. It reserves stock in a loop, one call per line item. Then it charges the card: one synchronous call whose answer determines everything below it.

The `alt` box is the heart of the diagram. In the happy branch, the order is marked paid, persisted, and an `OrderPlaced` event is fired **asynchronously** (open arrowhead, and crucially **no return arrow** — the service does not wait for anyone to consume it). In the else branch, the service **releases every reservation it made** — this compensating action is the single thing interviewers look for — persists the failed order, and returns a 402.

Notice the ordering decision baked in: **reserve → charge → persist → publish**. If you charged first and *then* discovered stock was gone, you'd have to refund a card: slow, costs a fee, can fail. Reserving first makes the rollback a cheap local operation.

### The matching JavaScript — arrow for arrow

Every arrow above is one line below. Read them side by side.

```js
// order-service.js  — the code twin of Diagram 1

export const OrderStatus = Object.freeze({
  PENDING: 'PENDING',
  PAID: 'PAID',
  FAILED: 'FAILED',
});

export class PaymentDeclinedError extends Error {
  constructor(reason) {
    super(`Payment declined: ${reason}`);
    this.name = 'PaymentDeclinedError';
    this.httpStatus = 402;
  }
}

class Order {
  constructor(userId, items) {                  // ← the «create» arrow
    this.id = `ord_${Date.now()}`;
    this.userId = userId;
    this.items = items;                         // [{ sku, qty, price }]
    this.status = OrderStatus.PENDING;
  }
  get total() {
    return this.items.reduce((sum, i) => sum + i.price * i.qty, 0);
  }
  markPaid(paymentId) {
    this.status = OrderStatus.PAID;
    this.paymentId = paymentId;
  }
  markFailed(reason) {
    this.status = OrderStatus.FAILED;
    this.failureReason = reason;
  }
}

export class OrderService {
  // Dependencies are injected, which is exactly why the diagram has separate
  // lifelines for them — each one is independently swappable and independently
  // able to fail.
  constructor({ inventory, payments, orders, eventBus }) {
    this.inventory = inventory;
    this.payments = payments;
    this.orders = orders;
    this.eventBus = eventBus;
  }

  async placeOrder(userId, items, card) {
    const order = new Order(userId, items);            // «create» Order
    const reservations = [];                            // remember what to undo

    // ── loop [for each line item] ──────────────────────────────────────────
    // Sequential, not Promise.all: we must know precisely which reservations
    // succeeded so the rollback below can release exactly those.
    for (const item of items) {
      const resId = await this.inventory.reserve(item.sku, item.qty); // sync call
      reservations.push(resId);                                        // ← return
    }

    // ── the one synchronous call whose answer branches the whole flow ──────
    const result = await this.payments.charge(order.total, card);

    // ── alt ────────────────────────────────────────────────────────────────
    if (result.status === 'CAPTURED') {                 // [captured]
      order.markPaid(result.paymentId);
      await this.orders.save(order);                    // sync call + return

      // ASYNC message: no await, no return arrow. If the bus is slow or the
      // consumers are down, the customer's checkout is unaffected.
      this.eventBus.publish('OrderPlaced', {
        orderId: order.id,
        userId,
        total: order.total,
      });

      return { httpStatus: 201, orderId: order.id };
    }

    // ── [else] — the compensating branch the interviewer is actually testing ─
    for (const resId of reservations) {                 // loop [for each resId]
      await this.inventory.release(resId);
    }
    order.markFailed(result.reason);
    await this.orders.save(order);
    throw new PaymentDeclinedError(result.reason);      // controller maps → 402
  }
}
```

The controller's self-call, drawn as the little loop-back arrow:

```js
// order-controller.js
export class OrderController {
  constructor(orderService) { this.orderService = orderService; }

  #validateBody(dto) {                                  // ← the self-call arrow
    if (!Array.isArray(dto.items) || dto.items.length === 0) {
      throw Object.assign(new Error('items must be a non-empty array'), { httpStatus: 400 });
    }
  }

  async post(req, res) {
    try {
      this.#validateBody(req.body);
      const out = await this.orderService.placeOrder(
        req.user.id, req.body.items, req.body.card
      );
      res.status(201).json(out);                        // ← the 201 return arrow
    } catch (err) {
      res.status(err.httpStatus ?? 500).json({ error: err.message }); // ← the 402
    }
  }
}
```

### Diagram 2 — The async flow (the caller does NOT wait)

Same business goal, different shape. `OrderService` publishes to a queue and returns *immediately*. A `Worker` picks the job up later, on its own schedule.

```
 Client       OrderService     MessageQueue        Worker      PaymentGateway  OrderRepository
   ┊               ┊                ┊                ┊               ┊               ┊
   ├──placeOrder()▶┤                ┊                ┊               ┊               ┊
  ┌┴┐             ┌┴┐               ┊                ┊               ┊               ┊
  │ │             │ ├─ save(PENDING) ────────────────────────────────────────────────▶┤
  │ │             │ │               ┊                ┊               ┊              ┌┴┐
  │ │             │ │◀- - - - - - - - - - - ok - - - - - - - - - - - - - - - - - - -└┬┘
  │ │             │ ├─ enqueue(job) ▷┤   ASYNC: open head, NO return arrow            ┊
  │ │◀- 202 Accepted {orderId} - - - ┊                ┊               ┊               ┊
  └┬┘             └┬┘               ┊                ┊               ┊               ┊
   ┊               ┊                ┊                ┊               ┊               ┊
   ┊  ▲ CLIENT IS DONE HERE. Latency ≈ 40 ms. It never saw the gateway.              ┊
   ┊  │                             ┊                ┊               ┊               ┊
   ┊  └──────── the client's activation bar has ENDED, but work continues ───────────┊
   ┊                                ┊                ┊               ┊               ┊
   ┊     ······ time gap: 5 ms, or 30 s, or after 3 retries ······                   ┊
   ┊                                ┊                ┊               ┊               ┊
   ┊                                ├── deliver(job) ▷┤              ┊               ┊
   ┊                                ┊               ┌┴┐              ┊               ┊
   ┊                                ┊               │ ├─charge(amt)─▶┤               ┊
   ┊                                ┊               │ │             ┌┴┐              ┊
   ┊                                ┊               │ │◀- result - -└┬┘              ┊
   ┊  ┌─ alt ────────────────────────────────────────────────────────────────────┐   ┊
   ┊  │ [CAPTURED]                  ┊               │ ├─ save(PAID) ────────────────▶┤
   ┊  │                             ┊               │ ├─ ack(job) ─▷┤            │   ┊
   ┊  ├ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -┤   ┊
   ┊  │ [DECLINED]                  ┊               │ ├─ save(FAILED) ──────────────▶┤
   ┊  │                             ┊               │ ├─ nack(job, requeue=false)─▷┤ │
   ┊  │                             ┊               │ │   → dead-letter queue     │   ┊
   ┊  └──────────────────────────────────────────────────────────────────────────┘   ┊
   ┊                                ┊               └┬┘              ┊               ┊
```

**The contrast — and this is the most valuable thing on the page:**

| | Diagram 1 (sync) | Diagram 2 (async) |
|---|---|---|
| Visual tell | The return arrow reaches the client **before** its activation bar ends | The client's bar **ends** while other bars are still running |
| Arrowhead at the boundary | Filled `▶` | Open `▷` |
| Client latency | Sum of *everything* (≈ 800 ms if the gateway is slow) | Just the DB write (≈ 40 ms) |
| Client learns the outcome | In the HTTP response (`201` or `402`) | Later — by polling, webhook, or WebSocket push |
| HTTP status | `201 Created` / `402 Payment Required` | `202 Accepted` — "I heard you; I haven't done it yet" |
| Failure handling | Rollback inside the request; client sees the error | Retries + dead-letter queue; client must be told out-of-band |
| Cost of adding a 6th downstream consumer | +N ms on every checkout | **0 ms** |

When you draw the async version on a whiteboard, **exaggerate the vertical gap** and write "…time passes…" in it. That gap *is* the design decision.

### The matching JavaScript — async twin

```js
// async-order-service.js — the code twin of Diagram 2

export class AsyncOrderService {
  constructor({ orders, queue }) {
    this.orders = orders;
    this.queue = queue;
  }

  async placeOrder(userId, items, card) {
    const order = new Order(userId, items);
    await this.orders.save(order);            // sync call + return: we DO wait for the DB

    // The async arrow. No `await`, no return value used. The caller's
    // activation bar ends on the very next line — that is the whole point.
    this.queue.enqueue('process-order', {
      orderId: order.id,
      amount: order.total,
      card,                                    // in reality: a card *token*, never a PAN
    });

    // 202 Accepted: "accepted for processing", not "done".
    return { httpStatus: 202, orderId: order.id, status: OrderStatus.PENDING };
  }
}

// worker.js — a SEPARATE process. Its lifeline starts far below the client's.
export class OrderWorker {
  constructor({ payments, orders, queue }) {
    this.payments = payments;
    this.orders = orders;
    this.queue = queue;
  }

  start() {
    // deliver(job) — the queue pushes work to us whenever we're free.
    this.queue.subscribe('process-order', (job) => this.handle(job));
  }

  async handle(job) {
    const order = await this.orders.findById(job.data.orderId);

    // Idempotency: a queue may deliver the same job twice. Without this guard
    // a redelivery double-charges the customer. The diagram makes the risk
    // visible — the deliver() arrow can simply happen more than once.
    if (order.status !== OrderStatus.PENDING) return this.queue.ack(job);

    const result = await this.payments.charge(job.data.amount, job.data.card, {
      idempotencyKey: order.id,               // the gateway dedupes retries for us
    });

    if (result.status === 'CAPTURED') {       // ── alt [CAPTURED]
      order.markPaid(result.paymentId);
      await this.orders.save(order);
      return this.queue.ack(job);
    }

    // ── alt [DECLINED] — nobody is on the phone to hear this. Persist it,
    // dead-letter the job, and notify the user out-of-band.
    order.markFailed(result.reason);
    await this.orders.save(order);
    return this.queue.nack(job, { requeue: false });   // → dead-letter queue
  }
}
```

Read the two `placeOrder` methods next to each other. The sync one *contains* the payment call; the async one merely *mentions* it. That difference is one missing return arrow on a whiteboard, and roughly 760 ms of user-visible latency in production.

---

## Real world examples

### Stripe — the webhook flow is literally a sequence diagram

When you call Stripe's PaymentIntents API, the HTTP response tells you the intent was *created*, not that money moved. The real capture may involve a bank and a 3-D Secure redirect. Stripe therefore documents the flow in two phases: your synchronous API call, then an **asynchronous webhook** (`payment_intent.succeeded`) delivered to your server later. That webhook is the open-arrowhead message from Diagram 2, drawn by a payments company in their own docs. Stripe also states webhooks may be delivered **more than once** — which is precisely why the worker above checks `order.status !== PENDING` before charging.

### Kafka-based order pipelines (common at Uber, LinkedIn, and most large e-commerce)

A typical checkout writes the order row, publishes an `OrderPlaced` event to Kafka, and returns. Downstream consumers — fulfilment, analytics, email, fraud scoring, recommendations — each read that event independently. On a sequence diagram this is **one** open-arrowhead message from the producer and then *N separate lifelines* consuming it later. The payoff is visible in the drawing: no return arrow means no latency coupling, so adding a sixth consumer adds **zero milliseconds** to the user's checkout.

### Express middleware — a sequence diagram you already write every day

```js
app.use(authenticate);   // participant 1
app.use(rateLimit);      // participant 2
app.use(orderRouter);    // participant 3
```

Every Express request is a synchronous chain: `Request → authenticate → rateLimit → router → handler → Response`, each link waiting on `next()`. Drawing it with an `alt` fragment (`[token invalid] → 401, short-circuit; [else] → next()`) is the clearest possible explanation of what middleware *is* — and it's a genuinely strong answer if an interviewer asks you to explain it.

---

## Trade-offs

### Sequence diagram vs class diagram

| | Class diagram | Sequence diagram |
|---|---|---|
| Shows | Nouns: classes, fields, relationships | Verbs: calls, order, timing |
| Great at | Structure, inheritance, ownership, cardinality | Flow, failure paths, sync vs async |
| Blind to | *When* anything happens; error handling | The full field/type surface; anything outside this one flow |
| Typical prompt | "Model a parking lot" | "Walk me through what happens when payment fails" |
| Breaks down when | You have 50 classes | You have 15 participants or 4 nested fragments |

**Use both.** Class diagram first (the nouns), then one sequence diagram per *interesting* flow — and "interesting" means "has a failure branch."

### Costs of drawing sequence diagrams

| Pro | Con |
|---|---|
| Makes ordering bugs obvious before you write code | One diagram covers one scenario; a 12-branch flow needs many |
| Forces you to answer "what if this fails?" | Goes stale fast if you commit it and forget it |
| The best possible argument for "make this async" | Nested fragments become unreadable past ~2 levels |
| Communicates a 6-service flow across teams in 30 seconds | Tempting to over-detail — every getter you draw is noise |

**Rule of thumb:** draw a sequence diagram for a flow if — and only if — it crosses **3+ participants** *or* has a **failure path worth rolling back**. Everything else, just read the code. And never put more than ~7 participants on one diagram; if you need more, you've found a missing service boundary, not a diagram problem.

---

## Common interview questions on this topic

### Q1: "When would you draw a sequence diagram instead of a class diagram?"

**Hint:** Class diagram = nouns and structure; sequence diagram = verbs over time. Reach for the sequence diagram when the question is about a *flow across several objects/services*, when there's a *failure path* (rollback, retry, compensation), or when you must show what's async. If you can draw only one for "design checkout", draw the sequence diagram — it implies most of the classes anyway, and it proves you thought about failure.

### Q2: "How do you show on a diagram that a call is asynchronous?"

**Hint:** Two things together: a solid line with an **open** (thin) arrowhead, and — the real tell — **no return arrow**. The caller's activation bar ends immediately. If you draw a dashed return arrow you've claimed the caller waits. Bonus point: note that in UML an `await`ed Promise is still a *synchronous* message, because the caller resumes only after the reply arrives.

### Q3: "In your order flow, what if the payment gateway times out after the charge actually succeeded?"

**Hint:** The classic. Your `alt [else]` branch releases inventory and marks the order FAILED — but the money left the customer's account. Strong answers: (a) send an **idempotency key** so a retry can't double-charge; (b) never decide from a timeout alone — *query* the gateway for the intent's true status, or wait for the webhook; (c) if you rolled back wrongly, an automated **refund/compensation** job repairs it. Say the sentence "a timeout is not a failure, it's an unknown" and you've essentially answered it.

### Q4: "Why reserve inventory before charging the card, rather than after?"

**Hint:** Because the compensating action for a reservation is cheap and local (release a row), while the compensating action for a capture is a refund — slow, costs a fee, can fail, and looks bad to the user. State the general principle out loud: **order your steps so the most expensive-to-undo step is last.**

### Q5: "What's the difference between `alt`, `opt`, `par`, and `loop`?"

**Hint:** `alt` = if/else, several guarded compartments, exactly one runs. `opt` = if with no else, one compartment, runs or doesn't. `loop` = repeat with a guard (`[for each item]`). `par` = several compartments that **all** run concurrently — this is `Promise.all()`. Mention `break` (early exit from the enclosing fragment) if you want to look sharp.

### Q6: "You're at a whiteboard with 4 minutes. How do you draw one fast?"

**Hint:** Give the exact recipe: (1) write the participants across the top **left-to-right in call order** — that alone makes the arrows readable; (2) remember time flows **down**; (3) draw the happy-path arrows only, labelling each with a method name; (4) *then* box the failure branch in an `alt`. **Skip the activation bars entirely if you're rushed** — nobody deducts points for missing rectangles, they deduct points for a missing rollback. If you're short on time, drop the return arrows too, except the one that goes back to the client.

---

## Practice exercise

### "Draw and code the seat-booking flow"

Model a **movie ticket booking**, both as a diagram and as running code. Budget ~35 minutes.

**Participants:** `Client`, `BookingController`, `BookingService`, `SeatLockService`, `PaymentGateway`, `BookingRepository`, `NotificationQueue`.

**Part A — the diagram (15 min).** On paper or in a fenced block, draw the sequence diagram for "user books 3 seats for a show." It must contain:
1. A `loop` over the 3 seats, locking each one via `SeatLockService.lock(seatId, userId, ttl=5min)`.
2. An `opt` fragment applying a coupon only `[if user.hasCoupon]`.
3. An `alt` fragment on the payment result. On `[declined]`, **release every seat lock you acquired** and mark the booking FAILED.
4. Exactly **one asynchronous** message: publishing `BookingConfirmed` to `NotificationQueue` (open arrowhead, **no return arrow**).
5. A `«create»` arrow producing the `Booking` object partway down the page.

**Part B — the code (15 min).** Write `BookingService.book(userId, showId, seatIds)` in JavaScript so that **every arrow in your diagram maps to exactly one line of code**. Track acquired locks in an array so the rollback can release exactly those. In-memory fakes for the other services are fine. Model the statuses as a frozen enum, as in the code above.

**Part C — the contrast (5 min).** Redraw *only* the top half as an **async** flow: `BookingService` writes a `PENDING` booking, enqueues a `ProcessBooking` job, and returns `202 Accepted` immediately. Underneath each version, write one sentence stating the client-visible latency and how the user eventually finds out whether the booking succeeded.

**Deliverable:** two ASCII diagrams plus a `booking-service.js` that runs with `node booking-service.js` and prints both the happy path and the declined path (with the seat locks visibly released).

---

## Quick reference cheat sheet

- **Time flows DOWN.** Anything drawn lower happened later. The one rule you can never break.
- **Participants left-to-right in call order.** Arrows should mostly go right and down. Zig-zag = you ordered them wrong.
- **Lifeline** = dashed vertical line under a participant = "this object exists during this span."
- **Activation bar** = thin rectangle on the lifeline = "this object is on the call stack right now." Nested calls = stacked bars.
- **Synchronous call** = solid line + **filled** arrowhead. The caller waits. In JS: `await svc.doThing()`.
- **Return** = **dashed** line + open arrowhead. Draw it only when the returned value matters; otherwise it's clutter.
- **Async message** = solid line + **open** arrowhead + **NO return arrow**. The missing return arrow *is* the message.
- **Self-call** = arrow loops back into its own lifeline, one notch lower. Draw only the interesting ones.
- **`«create»`** points at the participant's box, drawn partway down. **Destruction** = a big **X** ending the lifeline.
- **`alt`** = if/else (one compartment runs). **`opt`** = if, no else. **`loop`** = repeat. **`par`** = all compartments run concurrently = `Promise.all()`.
- **Every serious flow needs an `alt` with a failure branch.** No failure branch means you narrated the happy path, you didn't design a system.
- **Order steps so the most expensive-to-undo step is last.** Reserve stock → charge card, never the reverse.
- **Class diagram = nouns. Sequence diagram = verbs over time.** Interviewers test the verbs.
- **A timeout is not a failure, it's an unknown.** Idempotency keys + status reconciliation, not blind rollback.
- **Whiteboard under time pressure:** participants across the top in call order → time down → happy-path arrows → *then* box the `alt`. **Skip activation bars if rushed.** Nobody deducts points for missing rectangles; they deduct points for a missing rollback.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [20 — UML Class Diagrams](./20-uml-class-diagrams.md) — the nouns and the structure; sequence diagrams lay the verbs and the timeline on top of those same classes |
| **Next** | [22 — SOLID Principles](./20-uml-class-diagrams.md) — once you can see the call flow, you can see which class is doing too much (SRP) and where to invert a dependency (DIP) |
| **Related** | [02 — HLD vs LLD](./02-hld-vs-lld.md) — sequence diagrams work at both altitudes: service-to-service in HLD, object-to-object in LLD |
| **Related** | [10 — Message Queues](./67-message-queues.md) — the async half of this doc; an open-arrowhead message with no return arrow is exactly what a queue buys you |
