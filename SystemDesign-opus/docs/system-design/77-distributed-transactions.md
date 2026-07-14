# 77 — Distributed Transactions — 2PC and SAGA
## Category: HLD Components

---

## What is this?

A **distributed transaction** is a business operation that must succeed or fail as a unit, but whose steps live in **different databases owned by different services**. "Reserve the inventory, charge the card, create the order" is one thought in your head — but three separate writes to three separate machines.

Think of it like **three people signing a contract in three different countries**. In one room, you can watch everyone sign and tear the paper up if anyone refuses. Across three countries, you get a phone call saying "person 2 signed" and then... silence from person 3. Did they sign? Did they die? Is the letter still in the post? A distributed transaction is the set of techniques for getting a trustworthy answer to "did the whole thing happen, or none of it?" when you cannot see all the signatures at once.

---

## Why does it matter?

In a monolith with one database, this is a solved problem and you never think about it:

```javascript
// Monolith. One Postgres. One transaction. The database guarantees all-or-nothing.
await db.transaction(async (tx) => {
  await tx.query('UPDATE inventory SET qty = qty - 1 WHERE sku = $1 AND qty > 0', [sku]);
  await tx.query('INSERT INTO payments (order_id, amount) VALUES ($1, $2)', [orderId, 4999]);
  await tx.query('INSERT INTO orders (id, user_id, status) VALUES ($1, $2, $3)', [orderId, userId, 'PAID']);
});
// If ANY line throws, Postgres rolls back all three. You did nothing. Sleep well.
```

That `db.transaction` gives you **ACID** — Atomicity (all or nothing), Consistency, Isolation (nobody sees half-done work), Durability (once committed, it survives a crash).

Now split those three lines into three microservices with three databases (see [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)). **That guarantee evaporates.** There is no `BEGIN` that spans Inventory-DB, Payment-DB, and Order-DB. So:

- Payment succeeds, order creation crashes → **you charged a customer for nothing.**
- Inventory reserved, payment declined → **you're holding stock nobody bought.** Sold out, zero revenue.
- Order created, inventory reserve silently failed → **you sold a thing you don't have.**

This is the single biggest hidden cost of microservices. Teams split their monolith for deploy independence and then discover they've traded a one-line `COMMIT` for a distributed-systems research problem.

**Interview angle:** The moment you draw more than one database on the whiteboard, "how do you keep these consistent?" is coming. The expected answer is *not* "two-phase commit." It's "saga, orchestrated, with idempotent steps and compensations." If you can also explain *why not 2PC*, you sound senior.

**Work angle:** Every checkout flow, every money transfer, every booking system you will ever touch is a distributed transaction wearing a trenchcoat.

---

## The core idea — explained simply

### The Wedding Planner Analogy

You are planning a wedding. Three vendors must all say yes on the same day: the **venue**, the **caterer**, and the **band**. If you get two of three, you have a disaster — a hall full of hungry guests listening to silence.

**Approach A — The Group Phone Call (this is Two-Phase Commit).**

You get all three vendors on a conference call. You ask: *"Can you hold June 14th? Don't book it for anyone else. Just answer yes or no and then WAIT on the line."* All three say yes and freeze that date. Then you say "confirmed, book it." Everyone books. Beautiful — perfectly atomic.

Now the failures. While the vendors are holding the date and waiting on the line, they **cannot sell June 14th to anybody else**. Every other couple who calls gets told "sorry, tentatively held." And if *your phone dies* after they all said yes but before you said "confirmed" — the vendors are stuck. They can't book someone else (you might come back and confirm). They can't cancel (you might come back and confirm). They sit there, frozen, holding an asset hostage, forever. That is a **blocking protocol with a coordinator that is a single point of failure**, and it's why nobody uses 2PC across microservices.

**Approach B — Call Them One at a Time and Undo If Needed (this is a SAGA).**

You call the venue. Booked. You call the caterer. Booked. You call the band — *"sorry, we're taken."* So you call the caterer back and **cancel**, then call the venue back and **cancel**. You are back where you started.

Notice what you did NOT do: you didn't "roll back." Nothing rewound. You performed a **new, opposing action** — a cancellation — that *semantically* undoes the booking. The caterer's ledger now shows a booking AND a cancellation. History happened. You just neutralized it.

Notice also: for a few minutes, the outside world could see a half-planned wedding. The venue *was* booked, then wasn't. That's the price.

| Analogy | Technical concept |
|---|---|
| The wedding | The distributed transaction (business operation) |
| Venue / caterer / band | Participant services, each with its own database |
| "Hold the date, don't sell it" | PREPARE phase — resource is locked, vote cast |
| "Confirmed, book it" | COMMIT phase |
| Your phone dying mid-call | Coordinator crash → participants **in doubt**, locks held forever |
| Booking one vendor at a time | Saga: a sequence of **local** ACID transactions |
| Calling back to cancel | **Compensating transaction** |
| Guests seeing a half-planned wedding | **No isolation** — intermediate states are visible |
| You, holding the checklist | The **orchestrator** |
| Vendors each calling the next vendor | **Choreography** |

---

## Key concepts inside this topic

### 1. Two-Phase Commit (2PC) — how it actually works

2PC introduces a **coordinator** (also called a transaction manager) and N **participants**. It runs in two rounds.

**Phase 1 — PREPARE (the voting phase).** The coordinator asks every participant: *"Can you commit? Lock your rows, write your changes to your write-ahead log so they survive a crash, and promise me you will be able to commit if I ask."* Each participant replies **YES** (a binding promise — it can no longer refuse) or **NO**.

**Phase 2 — COMMIT or ABORT (the decision phase).** If **every** participant voted YES, the coordinator durably records "COMMIT" and tells everyone to commit. If **any** participant voted NO (or timed out), it records "ABORT" and tells everyone to abort. Participants release their locks only now.

```
                  ┌──────────────┐
                  │ COORDINATOR  │
                  └──────┬───────┘
    PHASE 1              │  (1) prepare?          ┌──────────────┐
    PREPARE /            ├──────────────────────▶ │ Inventory DB │  locks row, writes WAL
    VOTING               ├──────────────────────▶ │ Payment  DB  │  locks row, writes WAL
                         ├──────────────────────▶ │ Order    DB  │  locks row, writes WAL
                         │                        └──────────────┘
                         │  (2) YES / YES / YES
                         │◀────────────────────────────────────────
                         │
                  ┌──────▼────────────────────────────┐
                  │ ALL YES?  → write "COMMIT" to log │   ◀── the point of no return
                  │ ANY NO?   → write "ABORT"  to log │
                  └──────┬────────────────────────────┘
    PHASE 2              │  (3) COMMIT!           ┌──────────────┐
    COMMIT /             ├──────────────────────▶ │ Inventory DB │  commits, RELEASES LOCK
    ABORT                ├──────────────────────▶ │ Payment  DB  │  commits, RELEASES LOCK
                         ├──────────────────────▶ │ Order    DB  │  commits, RELEASES LOCK
                         │                        └──────────────┘
                         │  (4) ACK / ACK / ACK
                         │◀────────────────────────────────────────
                  ┌──────▼───────┐
                  │  DONE        │
                  └──────────────┘

     ⚠  THE DANGER ZONE is between (2) and (3).
        Every participant is holding locks and waiting.
        If the coordinator dies here, they are IN DOUBT — forever.
```

It genuinely gives you atomicity. So why does nobody use it for microservices?

**(a) It's BLOCKING.** Locks are held from PREPARE until COMMIT — across the network, across every participant, for the full round trip. Add a slow participant and *everyone* waits. A transaction that took 2ms in one DB now takes 50–200ms and holds row locks the whole time. Throughput on hot rows collapses.

**(b) The coordinator is a single point of failure — the "in-doubt" problem.** If the coordinator crashes after collecting YES votes but before broadcasting the decision, participants are stuck. They *promised* to be able to commit, so they can't unilaterally abort. They don't know the decision, so they can't commit. They hold their locks and **wait for the coordinator to come back**. Human operators end up manually resolving in-doubt transactions at 3am. This is not a theoretical worry; it is the everyday reality of XA transactions.

**(c) It needs universal protocol support.** Every participant must speak a distributed-transaction protocol (XA / two-phase commit). Postgres, MySQL, and Oracle can. Most NoSQL stores (DynamoDB, Cassandra, MongoDB across shards), Kafka, Redis, S3, and *literally every third-party HTTP API* cannot.

**(d) It doesn't compose with the internet.** You cannot `PREPARE` a payment at Stripe. There is no endpoint where you say "tentatively charge this card and hold the lock while I check with my inventory service." Stripe charges, or it doesn't. The moment a step of your business transaction leaves your datacenter, 2PC is off the table.

**What about 3PC?** Three-Phase Commit adds a "pre-commit" round so participants can time out and make progress without the coordinator. It reduces blocking — but it assumes a synchronous network with bounded delays, which the real world does not provide, and it adds another network round trip to an already slow protocol. It is a textbook curiosity. Nobody ships it.

**When 2PC IS fine:** short-lived transactions inside one datacenter, few participants, all speaking XA, with low contention — e.g. two Postgres shards behind one bank's ledger service. Kafka's exactly-once producer also uses a 2PC-flavoured protocol internally. Outside that box: don't.

### 2. The SAGA pattern — the practical answer

A **saga** is a distributed transaction broken into a **sequence of local transactions**. Each step is a plain, boring, fully-ACID transaction inside **one** service's own database — no distributed locks anywhere. Each step publishes a result. If step N fails, you execute **compensating transactions** for steps N-1, N-2, ... 1, in reverse, to semantically undo them.

The crucial mental shift:

> **You do not roll back. You cannot roll back — the money already moved, the email already left, the warehouse already picked the item.**
> **You apply an OPPOSING ACTION.**

A rollback erases history. A compensation adds new history that cancels out the old. Your payment ledger will show a charge *and* a refund, and that's correct — that's what actually happened in the world.

| Forward action | Compensating action | Note |
|---|---|---|
| Reserve inventory | Release inventory | Clean — pure state reversal |
| Charge card | Refund card | Not free: fees may be lost, customer sees both lines |
| Book seat 14A | Cancel booking 14A | Someone else may have grabbed it in between |
| Create order (status PAID) | Set order status CANCELLED | Prefer a state change to a hard `DELETE` — keep the audit trail |
| Issue loyalty points | Deduct loyalty points | Only safe if the user hasn't spent them (see §4) |
| Increment "orders" counter | Decrement "orders" counter | Commutative — safe in any order |
| **Send confirmation email** | Send "sorry, cancelled" email | **NOT a true undo.** The customer read the first one. |
| **Ship the package** | Recall the package / accept return | Slow, expensive, may fail |
| **Launch the missile** | *(nothing)* | Genuinely not compensable |

**The practical rule this table teaches you: order your steps so the hardest-to-undo action happens LAST.** Reserve inventory (trivially reversible) → charge the card (reversible, but costs you) → create the order → *then* send the email and ship. If a step can't be compensated at all, it must be the final step, after which nothing can fail.

### 3. Choreography SAGA — no central brain

Each service listens for the previous service's event, does its local transaction, and emits its own event. There is no coordinator. This rides on top of your message broker (see [67 — Message Queues](./67-message-queues.md) and [68 — Event-Driven Architecture](./68-event-driven-architecture.md)).

```
HAPPY PATH (events flowing right):

 ORDER SVC          INVENTORY SVC         PAYMENT SVC          SHIPPING SVC
     │                    │                    │                    │
  create order            │                    │                    │
  status=PENDING          │                    │                    │
     │                    │                    │                    │
     ├─ OrderCreated ────▶│                    │                    │
     │                 reserve stock           │                    │
     │                    ├─ StockReserved ───▶│                    │
     │                    │                 charge card             │
     │                    │                    ├─ PaymentDone ─────▶│
     │                    │                    │                 schedule ship
     │◀─────────────────── OrderConfirmed ◀────┴────────────────────┤
  status=CONFIRMED        │                    │                    │


FAILURE PATH (compensation events flowing back left):

     │                    │                    │
     │                    │              charge DECLINED
     │                    │◀── PaymentFailed ──┤
     │             release stock               │
     │◀── StockReleased ──┤                    │
  status=CANCELLED        │                    │
```

```javascript
// choreography — inventory-service/handlers.js
// Each service subscribes to what it cares about and emits what it did.
// Nobody anywhere knows the whole flow. That is the point, and also the problem.

bus.on('OrderCreated', async (evt) => {
  try {
    await inventoryDb.reserve(evt.sku, evt.qty, evt.orderId); // local ACID txn
    await bus.emit('StockReserved', { orderId: evt.orderId, sku: evt.sku, qty: evt.qty });
  } catch (err) {
    await bus.emit('StockReservationFailed', { orderId: evt.orderId, reason: err.message });
  }
});

// This service is also responsible for its OWN compensation.
bus.on('PaymentFailed', async (evt) => {
  await inventoryDb.release(evt.orderId);           // must be idempotent!
  await bus.emit('StockReleased', { orderId: evt.orderId });
});
```

| Choreography pros | Choreography cons |
|---|---|
| No coordinator → no single point of failure | **The business process exists nowhere in the code.** It is smeared across 4 repos. |
| Services are loosely coupled; add a listener without touching others | Nobody can answer "what happens if payment fails at step 3?" without reading every service |
| Naturally fits an existing event bus | Cyclic dependencies creep in (Payment now listens to Inventory *and* Order) |
| Cheap to start | Debugging and monitoring is archaeology across 4 log streams |

Use choreography when the saga is **short (2–3 steps)** and the flow is genuinely simple.

### 4. Orchestration SAGA — one brain, one file

A central **orchestrator** owns the process. It calls step 1, waits, calls step 2, waits... and on failure, walks *backwards* calling compensations. The entire business process is readable top-to-bottom in one place.

```javascript
// saga/OrderSaga.js
// A step = { name, invoke, compensate }.
// invoke:     do the forward work. MUST be idempotent.
// compensate: semantically undo it. MUST be idempotent. MUST tolerate
//             "the forward step may never have actually run."

export class SagaStep {
  constructor({ name, invoke, compensate, retries = 3 }) {
    this.name = name;
    this.invoke = invoke;
    this.compensate = compensate ?? (async () => {}); // read-only steps need no undo
    this.retries = retries;
  }
}

export class OrderSaga {
  /**
   * @param {SagaStep[]} steps  executed in order; compensated in reverse
   * @param {object} log        DURABLE store (Postgres table, not memory!) — see §6
   */
  constructor(steps, log) {
    this.steps = steps;
    this.log = log;
  }

  async execute(sagaId, context) {
    // Resume support: a crashed orchestrator restarts here and skips finished steps.
    const state = (await this.log.load(sagaId)) ?? { completed: [], status: 'RUNNING', context };
    context = state.context;

    for (const step of this.steps) {
      if (state.completed.includes(step.name)) continue; // already done before the crash

      try {
        const result = await this.#withRetry(step, context);
        Object.assign(context, result ?? {}); // steps hand data forward (paymentId, etc.)

        state.completed.push(step.name);
        state.context = context;
        // Persist BEFORE moving on. If we die now, we know exactly where we were.
        await this.log.save(sagaId, state);
      } catch (err) {
        state.status = 'COMPENSATING';
        state.failedStep = step.name;
        state.error = err.message;
        await this.log.save(sagaId, state);

        await this.#compensate(sagaId, state, context);

        state.status = 'FAILED';
        await this.log.save(sagaId, state);
        throw new SagaFailedError(step.name, err, context);
      }
    }

    state.status = 'COMPLETED';
    await this.log.save(sagaId, state);
    return context;
  }

  /** Walk backwards through everything that succeeded, undoing it. */
  async #compensate(sagaId, state, context) {
    const done = [...state.completed].reverse();

    for (const name of done) {
      const step = this.steps.find((s) => s.name === name);
      try {
        await this.#withRetry({ ...step, invoke: step.compensate }, context);
        state.completed = state.completed.filter((n) => n !== name);
        await this.log.save(sagaId, state);
      } catch (err) {
        // A compensation FAILED. This is the worst case in saga-land.
        // You cannot go forward and you cannot go back.
        // Never swallow this: park it for a human. Money is on the line.
        await this.log.markForManualReview(sagaId, name, err.message);
        throw new CompensationFailedError(name, err);
      }
    }
  }

  async #withRetry(step, context) {
    let lastErr;
    for (let attempt = 1; attempt <= step.retries; attempt++) {
      try {
        return await step.invoke(context);
      } catch (err) {
        if (err.permanent) throw err;      // 402 card declined — retrying won't help
        lastErr = err;                     // 503 / timeout — worth another go
        await sleep(2 ** attempt * 100);   // exponential backoff
      }
    }
    throw lastErr;
  }
}

export class SagaFailedError extends Error {
  constructor(step, cause, context) {
    super(`Saga failed at step "${step}": ${cause.message}`);
    this.step = step; this.cause = cause; this.context = context;
  }
}
export class CompensationFailedError extends Error {
  constructor(step, cause) {
    super(`COMPENSATION FAILED at "${step}" — MANUAL INTERVENTION REQUIRED: ${cause.message}`);
    this.step = step; this.cause = cause;
  }
}
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
```

Now the actual business process — **read this top to bottom and you know exactly what the company does when you buy something, including what happens when it goes wrong.** That readability is the entire argument for orchestration.

```javascript
// saga/checkout.js
import { OrderSaga, SagaStep } from './OrderSaga.js';

export function buildCheckoutSaga({ orders, inventory, payments, shipping, mailer }, log) {
  return new OrderSaga([
    new SagaStep({
      name: 'CREATE_ORDER',
      // Semantic lock: the order is visible immediately, but marked PENDING so
      // no other part of the system treats it as real. See §5.
      invoke: async (ctx) => {
        const order = await orders.create({
          id: ctx.orderId, userId: ctx.userId, items: ctx.items,
          total: ctx.total, status: 'PENDING'
        });
        return { orderId: order.id };
      },
      // Never DELETE — a cancelled order is a fact you want in your audit trail.
      compensate: async (ctx) => orders.setStatus(ctx.orderId, 'CANCELLED'),
    }),

    new SagaStep({
      name: 'RESERVE_INVENTORY',
      // Idempotency key = orderId. Calling this twice reserves ONE unit, not two.
      invoke: async (ctx) => {
        await inventory.reserve({ orderId: ctx.orderId, items: ctx.items });
        return { inventoryReserved: true };
      },
      compensate: async (ctx) => inventory.release({ orderId: ctx.orderId }),
    }),

    new SagaStep({
      name: 'CHARGE_PAYMENT',
      // The expensive, externally-visible step. Deliberately placed AFTER the
      // cheap-to-undo one, so a stock-out never triggers a refund.
      invoke: async (ctx) => {
        const charge = await payments.charge({
          idempotencyKey: `charge:${ctx.orderId}`,  // Stripe honours this natively
          userId: ctx.userId, amountCents: ctx.total,
        });
        return { paymentId: charge.id };
      },
      compensate: async (ctx) => {
        if (!ctx.paymentId) return;  // charge may never have landed — tolerate it
        await payments.refund({ idempotencyKey: `refund:${ctx.orderId}`, paymentId: ctx.paymentId });
      },
    }),

    new SagaStep({
      name: 'SCHEDULE_SHIPMENT',
      invoke: async (ctx) => {
        const s = await shipping.schedule({ orderId: ctx.orderId, address: ctx.address });
        return { shipmentId: s.id };
      },
      compensate: async (ctx) => shipping.cancel({ shipmentId: ctx.shipmentId }),
    }),

    new SagaStep({
      name: 'CONFIRM_ORDER',
      invoke: async (ctx) => orders.setStatus(ctx.orderId, 'CONFIRMED'), // lock released
      compensate: async (ctx) => orders.setStatus(ctx.orderId, 'CANCELLED'),
    }),

    new SagaStep({
      name: 'SEND_RECEIPT',
      // LAST ON PURPOSE. An email is NOT compensable — you cannot unsend it.
      // By the time we get here, nothing after it can fail.
      invoke: async (ctx) => mailer.sendReceipt(ctx.userId, ctx.orderId),
      compensate: async () => {},   // there is no undo. That's why it's last.
      retries: 5,
    }),
  ], log);
}
```

| Orchestration pros | Orchestration cons |
|---|---|
| **The whole business process lives in ONE readable file** | A central component you must build, deploy, and keep available |
| Compensation order is explicit and obvious | Risk of the orchestrator becoming a god-object with business logic in it |
| Trivial to monitor: "how many sagas are stuck in COMPENSATING?" | Extra network hop per step (orchestrator → service → orchestrator) |
| Easy to add a step — one entry in an array | Services become slightly more coupled to the orchestrator |
| Testable end-to-end with fakes for each service | |

**The honest recommendation: use orchestration for anything business-critical.** Checkout, payments, booking, provisioning. The "SPOF" objection is weak — you run 3 orchestrator replicas behind a load balancer with the saga log in Postgres, exactly like every other stateless service you own. Meanwhile, the choreography cost — *nobody in the company can tell you what happens when payment fails* — is a permanent tax on every future engineer. Keep choreography for short, low-stakes flows (e.g. "user signed up → send welcome email → warm their cache").

### 5. There is NO isolation — and it will bite you

Look back at ACID. A saga gives you **A**, **C**, **D** — and quietly drops the **I**.

Recall from [66 — Database Transactions and Isolation](./66-database-transactions-and-isolation.md) that isolation means concurrent transactions can't see each other's half-finished work. **A saga cannot provide this.** Between step 2 and step 3, the order row is *committed and visible to the entire company*. Your analytics job sees it. Another user's checkout sees the inventory decrement. A support agent sees an order that is about to be un-created.

This resurrects, at the *business* level, the anomalies your database used to protect you from:

- **Dirty read:** the fraud service reads the order as PAID, but the saga later compensates and refunds it. Fraud has now scored a transaction that never happened.
- **Lost update:** the saga is compensating a loyalty-point grant by writing `points = 500` (the old value it remembers), while the user simultaneously earned 50 points elsewhere. The concurrent update is silently destroyed.

**Countermeasures (from Garcia-Molina's original saga work, and used in practice):**

1. **Semantic lock.** Mark the record with an in-flight status — `status = 'PENDING'`, `funds_on_hold = true`. Every other reader must respect it. This is the workhorse; it's why our `CREATE_ORDER` step writes `PENDING` first and `CONFIRMED` last. You are hand-rolling isolation at the application layer, and you must decide what other readers do when they hit a PENDING row: block, skip, or show a "processing…" state to the user.
2. **Commutative updates.** Design writes so order doesn't matter. `balance = balance - 50` composes safely with a concurrent `balance = balance + 30`; `balance = 450` does not. Prefer deltas over absolute sets, and you kill the lost-update class of bugs outright.
3. **Re-read before compensating.** Never compensate using a value you cached at the start. Re-read current state and apply a *relative* correction. Never refund "the amount I remembered"; refund against `paymentId` and let the payment service tell you what was actually charged.
4. **Pessimistic ordering.** Reorder steps so the step most likely to fail runs *first*, before anything visible has been written. Validate the card (a `$0` auth) before reserving stock, and most failures cost you zero compensation.

### 6. Idempotency is MANDATORY — say it louder

**Every saga step and every compensation WILL run more than once.** Not "might." Will.

The orchestrator calls `payments.charge()`, the charge succeeds, and the *response* is lost to a network timeout. The orchestrator has no idea. It retries. Without idempotency, **you just charged the customer twice.** The same is true of compensations: the orchestrator crashes mid-compensation, restarts, and re-runs `inventory.release()` — releasing 2 units when only 1 was ever reserved, and now your stock count is corrupt forever.

Every `invoke` and every `compensate` must be safe to call N times with the same result as calling it once (see [85 — Idempotency](./85-idempotency.md)).

```javascript
// BAD — a retry double-charges and double-releases. This is a production incident.
async function reserve({ orderId, sku, qty }) {
  await db.query('UPDATE inventory SET reserved = reserved + $1 WHERE sku = $2', [qty, sku]);
}

// GOOD — the idempotency key makes the second call a no-op.
async function reserve({ orderId, sku, qty }) {
  await db.transaction(async (tx) => {
    // A unique index on (order_id, sku) turns a duplicate call into a conflict
    // that we deliberately swallow. The DB is the arbiter, not our memory.
    const { rowCount } = await tx.query(
      `INSERT INTO reservations (order_id, sku, qty)
       VALUES ($1, $2, $3) ON CONFLICT (order_id, sku) DO NOTHING`,
      [orderId, sku, qty]
    );
    if (rowCount === 0) return;  // already reserved by a previous attempt — done.

    const { rowCount: ok } = await tx.query(
      'UPDATE inventory SET reserved = reserved + $1 WHERE sku = $2 AND (qty - reserved) >= $1',
      [qty, sku]
    );
    if (ok === 0) throw Object.assign(new Error('OUT_OF_STOCK'), { permanent: true });
  });
}

// GOOD — compensation is idempotent AND tolerates "never actually reserved".
async function release({ orderId }) {
  await db.transaction(async (tx) => {
    // DELETE...RETURNING is the trick: the row can only be claimed once.
    const { rows } = await tx.query(
      'DELETE FROM reservations WHERE order_id = $1 RETURNING sku, qty', [orderId]
    );
    for (const r of rows) {
      await tx.query('UPDATE inventory SET reserved = reserved - $1 WHERE sku = $2', [r.qty, r.sku]);
    }
    // rows === [] → nothing to undo. Not an error. Just return quietly.
  });
}
```

### 7. Where the saga state lives

An orchestrator that keeps its progress in a JavaScript object **is a bug waiting for a deploy.** Restart the pod and it forgets that it charged someone's card. The saga log must be **durably persisted, and written after every single step transition**, before the next step begins.

```sql
CREATE TABLE saga_instances (
  saga_id       UUID PRIMARY KEY,
  saga_type     TEXT NOT NULL,          -- 'CHECKOUT'
  status        TEXT NOT NULL,          -- RUNNING | COMPENSATING | COMPLETED | FAILED | NEEDS_REVIEW
  completed     JSONB NOT NULL,         -- ["CREATE_ORDER","RESERVE_INVENTORY"]
  context       JSONB NOT NULL,         -- { orderId, paymentId, ... }
  failed_step   TEXT,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON saga_instances (status, updated_at);  -- find stuck sagas
```

That last index earns its keep: a background sweeper queries `status IN ('RUNNING','COMPENSATING') AND updated_at < now() - interval '5 minutes'` and resumes them. Sagas do not fail loudly — **they get stuck silently**, which is why "count of sagas older than 5 minutes still RUNNING" should be a page-worthy alert on day one.

One more trap: if your orchestrator commits to its own DB *and then* publishes a command to a queue, those are two writes — the classic **dual-write problem**. Fix it with the **transactional outbox**: write the command into an `outbox` table *in the same local transaction* as the state update, and let a separate relay poll the table and publish. Same trick, one layer down.

---

## Visual / Diagram description

### Diagram 1: Orchestration saga — forward path and compensation path

```
                          ┌───────────────────────────┐
   POST /checkout ───────▶│   CHECKOUT ORCHESTRATOR   │──────┐ writes state after
                          │  (stateless, 3 replicas)  │      │ EVERY step
                          └───┬───┬───┬───┬───────────┘      ▼
             ┌────────────────┘   │   │   └──────┐     ┌───────────────┐
        (1)  │            (2)     │   │(3)       │(4)  │  SAGA LOG     │
     create  │        reserve     │   │ charge   │ship │  (Postgres)   │
      order  │          stock     │   │  card    │     │ status,       │
             ▼                    ▼   ▼          ▼     │ completed[],  │
      ┌────────────┐   ┌────────────┐  ┌──────────┐    │ context{}     │
      │  ORDER SVC │   │INVENTORY   │  │ PAYMENT  │    └───────────────┘
      │  order_db  │   │ SVC  inv_db│  │ SVC (→Stripe)
      └────────────┘   └────────────┘  └──────────┘
            ▲                 ▲              ✗ DECLINED
            │                 │              │
            │                 │              │  orchestrator sees the failure
            │                 │              │  and walks the completed list
            │                 │              │  BACKWARDS:
            │                 │              │
   C1: status=CANCELLED  C2: release stock   │
            └─────────────────┴──────────────┘
                    COMPENSATION FLOW  ◀── runs in REVERSE order (C2 then C1)
```

The top half is the happy path: the orchestrator calls each service in turn, and after each success it writes `completed: [...]` to the saga log so a crash can resume exactly where it stopped. The bottom half is the failure path: payment is declined, so the orchestrator reads its own `completed` list, reverses it, and calls the compensation for each entry — release the stock, then cancel the order. Every arrow crosses a network boundary and can therefore be retried, which is why every box must be idempotent.

### Diagram 2: The 2PC in-doubt window (draw this to explain why 2PC loses)

```
 time ──▶
          PREPARE            (coordinator dies here)          COMMIT never arrives
             │                          ✗                              │
 Inv DB   ───┼══════════════════════════════════════════════════════════▶  LOCKED
 Pay DB   ───┼══════════════════════════════════════════════════════════▶  LOCKED
 Ord DB   ───┼══════════════════════════════════════════════════════════▶  LOCKED
             │                                                             │
             └────────────── locks held, throughput = 0 ───────────────────┘
                             participants CANNOT abort (they promised)
                             participants CANNOT commit (no decision)
                             → in-doubt until a human intervenes


 SAGA equivalent — nothing is ever locked across the network:

 Inv DB   ──[▪]──────────────────────────  local txn, 3ms, lock released instantly
 Pay DB   ───────────[▪]──────────────────  local txn, 3ms, lock released instantly
 Ord DB   ─────────────────────[▪]────────  local txn, 3ms, lock released instantly
                    ▲
                    └─ but between these, the world SEES intermediate state
```

Draw both of these on a whiteboard and you have made the entire argument: 2PC buys isolation by holding locks across the network and dies with its coordinator; sagas hold no cross-network locks at all and pay for it with visible intermediate state.

---

## Real world examples

### 1. E-commerce checkout (Amazon-style, conceptual)

Amazon's order pipeline is famously **not** a distributed transaction — it's an asynchronous, compensation-driven flow. You click "Buy now" and get an order confirmation *immediately*, with status "Pending." Behind that, payment authorization, inventory allocation, and fulfillment planning proceed independently. If the card later fails, you don't get a rollback — you get an email: *"There was a problem with your payment method."* The order moves to a CANCELLED/ON_HOLD state and any allocated inventory is released. That's a saga: **the semantic lock is the visible "Pending" status**, and the compensation is a state change plus an email. Notice the order of operations too — nothing physically ships until payment settles, because *shipping is the step that's expensive to compensate*.

### 2. Travel booking — flight + hotel + car

This is the canonical saga because 2PC is *physically impossible* here: three different companies' systems, none of which will let you "prepare" a booking. So the booking site does exactly what our orchestrator does — book the flight, book the hotel, book the car, and if the car step fails, **cancel** the hotel and **cancel** the flight. The compensations are real API calls with real consequences: airline cancellation fees, a hotel that has already released the room to someone else, and a window of minutes where you genuinely held three bookings you were about to give up. Notice the *ordering* logic: reputable systems book the **hardest-to-get, most-expensive-to-cancel** item last, and hold a $0 card authorization first so a declined card costs zero compensations.

### 3. Money transfer between banks

Moving $100 from Bank A to Bank B is two local transactions in two ledgers that will never share a transaction manager. The real-world implementation is a saga with a semantic lock: Bank A **debits and holds** (your balance drops, the money sits in a suspense/clearing account — "funds on hold"), then the transfer is submitted to the network, and only on confirmation does Bank B credit. If the credit is rejected, the compensation is a **reversal entry** into your account. This is why a failed transfer shows up as *two lines* on your statement — a debit and a credit — instead of vanishing. The ledger is append-only; you never erase, you only compensate. Money systems are the clearest proof that compensation, not rollback, is how the real world works.

### 4. Productionized saga orchestrators

Do not hand-build the durable state machine if you can avoid it.

- **Temporal** (and its ancestor, Uber's Cadence): you write the saga as **ordinary async JavaScript** in a "workflow" function; Temporal durably records every step's result, so if your process dies mid-flow it *replays* the function and continues from exactly where it stopped. It has first-class saga support — you push compensations onto a list as you go and run them on failure. This is what most serious teams reach for.
- **AWS Step Functions:** you define the saga as a JSON/ASL state machine with explicit `Catch` blocks routing to compensation states. AWS runs the durable log for you.
- **Netflix Conductor**, **Camunda / Zeebe**: same idea, different flavours.

They all solve the same hard part: **durably remembering what you already did.**

---

## Trade-offs

| Dimension | 2PC | Saga |
|---|---|---|
| **Atomicity** | True atomicity | Eventual atomicity (all steps, or all compensated) |
| **Isolation** | Yes — serializable | **None.** Intermediate state is visible |
| **Locks** | Held across the network for the whole transaction | Only inside each local transaction (milliseconds) |
| **Throughput** | Poor under contention | High |
| **Coordinator failure** | Participants stuck **in doubt**, holding locks | Resumable from the durable saga log |
| **Works with 3rd-party APIs (Stripe)** | No | Yes |
| **Works with NoSQL / Kafka / S3** | Mostly no | Yes |
| **Developer complexity** | Low (the protocol hides it) — until it breaks | **High** — you write every compensation by hand |
| **Failure mode** | Blocks | Temporarily inconsistent, then converges |

| Choreography | Orchestration |
|---|---|
| No central component | One central component (run replicas; it's stateless) |
| Loose coupling, easy to bolt on a listener | Slightly tighter coupling to the orchestrator |
| **Process is invisible** — smeared across N services | **Process is one readable file** |
| Compensation logic scattered; easy to miss a case | Compensation order explicit and reviewable |
| Hard to monitor / debug | Trivially monitorable ("sagas stuck in COMPENSATING") |
| Fine for 2–3 simple steps | Correct for 4+ steps or anything touching money |

**Rule of thumb:** If it fits in one database, **use a plain ACID transaction** — do not build a saga to look clever. If it spans services and stays inside your datacenter with XA-capable DBs and low contention, 2PC is *defensible*. For everything else — which is almost everything — **use an orchestrated saga with idempotent steps, explicit compensations, a durable saga log, and the irreversible step last.** And accept, out loud, that you have traded isolation for availability — which means your system is eventually consistent (see [76 — Eventual Consistency](./76-eventual-consistency.md)) and your product must show "Pending" states to users.

---

## Common interview questions on this topic

### Q1: "You have Order, Payment, and Inventory services with separate databases. How do you keep them consistent?"
**Hint:** Name the problem first ("there's no transaction that spans three databases"), then reject 2PC with a reason (blocking + coordinator SPOF + Stripe can't PREPARE), then propose an **orchestrated saga**: sequence of local transactions, compensations in reverse on failure, durable saga log, idempotent steps. Say the sentence "we don't roll back, we apply an opposing action — a refund." Then volunteer the cost: no isolation, so the order sits in a visible PENDING state.

### Q2: "Why doesn't anyone use two-phase commit for microservices?"
**Hint:** Four reasons, in order of force: (1) it's **blocking** — locks are held across the network for the full round trip, killing throughput; (2) the coordinator is a SPOF and a crash after PREPARE leaves participants **in doubt**, holding locks indefinitely with no way to decide; (3) every participant must speak XA, and Kafka/Dynamo/S3 don't; (4) it doesn't compose with third-party APIs — you can't PREPARE a Stripe charge. Bonus: mention 3PC exists, reduces blocking, assumes a synchronous network, and still nobody ships it.

### Q3: "What happens if a compensating transaction itself fails?"
**Hint:** This is the question that separates people who've read about sagas from people who've run them. You retry with exponential backoff (compensations must be idempotent, so retry is safe). If it keeps failing, you **cannot go forward and cannot go back** — so you park the saga in a `NEEDS_REVIEW` state, alert, and escalate to a human. Design principle: make compensations as simple and as likely-to-succeed as possible (a status flip beats a call to a flaky third party), and never let a compensation depend on a service that might be down for the same reason the forward step failed.

### Q4: "Choreography or orchestration — which would you pick, and why?"
**Hint:** Orchestration for anything business-critical. The killer argument isn't technical, it's organizational: with choreography **the business process exists nowhere** — no engineer can tell you what happens when payment fails without reading four repos. The SPOF objection to orchestration is weak (run three stateless replicas, state in Postgres). Concede that choreography is fine for short, low-stakes, 2–3 step flows.

### Q5: "Your saga charges a card, then crashes before recording success. What happens on restart?"
**Hint:** This is exactly why **idempotency is mandatory**. On restart the orchestrator reloads the saga log, sees `CHARGE_PAYMENT` isn't in `completed`, and retries it — with the same idempotency key (`charge:${orderId}`). Stripe recognises the key and returns the *original* charge instead of creating a second one. No double charge. Without the key, you just billed the customer twice. Then add: this is why the saga log must be durable and written after every step, and why a background sweeper must resume sagas stuck in RUNNING.

---

## Practice exercise

### Build a Hotel + Flight booking saga

Build a runnable Node.js orchestrated saga for a travel booking: **reserve flight → reserve hotel → charge card → send itinerary email.**

**What to produce:**

1. **Three fake services** as classes (`FlightService`, `HotelService`, `PaymentService`) with in-memory `Map`s as their "databases." Each exposes a forward method and a compensating method (`reserve`/`cancel`, `charge`/`refund`).
2. Make **every one of those six methods idempotent** using an `idempotencyKey`. Prove it: call `charge()` twice with the same key and assert the balance only moved once.
3. Use the `OrderSaga` + `SagaStep` classes from this doc (copy them) to wire the four steps together with their compensations.
4. Add a `SagaLog` class that persists to a **JSON file on disk** (stand-in for Postgres). It must survive a process restart.
5. **Force the failures** and observe:
   - Make `PaymentService.charge()` throw. Run it. Assert the hotel *and* the flight both got cancelled, in that order (reverse!), and that no email was sent.
   - Make `HotelService.reserve()` throw. Assert only the flight was cancelled and the card was **never charged** — proving your step ordering saved you a refund.
   - Kill the process (`process.exit(1)`) *after* the charge but *before* the log write. Restart. Prove that re-running the saga with the same `sagaId` does **not** double-charge.

**The questions to answer in comments at the bottom of your file:**
- Which step is not truly compensable, and why did you put it last?
- Between "flight reserved" and "card charged," what could another part of the system see? What semantic lock would you add to hide it?

~30–40 minutes. If your third failure test double-charges, you have found the exact bug that puts real companies on the news.

---

## Quick reference cheat sheet

- **Distributed transaction** = one business operation spanning multiple services/databases. There is no `BEGIN...COMMIT` across machines.
- **2PC** = coordinator + PREPARE (vote) phase + COMMIT (decide) phase. Gives true atomicity *and* isolation.
- **2PC is blocking:** locks are held across the network for the entire transaction. Throughput dies.
- **The in-doubt problem:** coordinator crashes after PREPARE → participants hold locks forever, unable to commit or abort. The reason 2PC is dead for microservices.
- **2PC can't reach the internet:** you cannot ask Stripe to "prepare" a charge. **3PC** exists, reduces blocking, assumes a synchronous network, and nobody ships it.
- **Saga** = a sequence of **local** ACID transactions, one per service, with **compensating transactions** to semantically undo on failure.
- **You never roll back — you apply an opposing action.** The charge stays in the ledger; you add a refund.
- **Some actions cannot be compensated** (sent email, shipped package). **Put the irreversible step LAST.**
- **Choreography** = services react to each other's events. No SPOF, but the business process exists nowhere and nobody can debug it.
- **Orchestration** = one central coordinator calls each step and, on failure, calls compensations in reverse. **Use this for anything business-critical.**
- **Sagas have NO isolation.** Intermediate state is visible → dirty reads and lost updates at the *business* level.
- **Countermeasures:** semantic locks (`status = PENDING`), commutative updates (deltas, not absolute sets), re-read before compensating, fail-fast step ordering.
- **Idempotency is MANDATORY.** Every step and every compensation *will* be retried. Use an idempotency key derived from the saga ID.
- **The saga log must be durable** (a Postgres table, not memory), written after every step, with a sweeper that resumes sagas stuck in RUNNING or COMPENSATING.
- **If a compensation fails,** retry with backoff, then park in `NEEDS_REVIEW` and page a human. Never swallow it.
- **Don't hand-roll it in production:** Temporal, AWS Step Functions, Netflix Conductor, Camunda are productionized saga engines.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [76 — Eventual Consistency](./76-eventual-consistency.md) — sagas are *how* you achieve eventual consistency across services; that doc explains the guarantee, this one gives you the mechanism |
| **Next** | [85 — Idempotency](./85-idempotency.md) — the mandatory prerequisite for every saga step and every compensation; without it, retries corrupt your data |
| **Related** | [66 — Database Transactions and Isolation](./66-database-transactions-and-isolation.md) — the ACID guarantees you get for free in one DB, and the "I" that sagas throw away |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — losing the single-database transaction is the biggest hidden cost of splitting up |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — the substrate that choreography sagas run on |
| **Related** | [67 — Message Queues](./67-message-queues.md) — how orchestrator commands and saga events are actually delivered (and redelivered) |
| **Related** | [107 — HLD: Payment System](./107-hld-payment-system.md) — the case study where every idea in this doc becomes load-bearing |
