# 68 — Event-Driven Architecture
## Category: HLD Components

---

## What is this?

Event-Driven Architecture (EDA) is a style of building systems where components communicate by **announcing facts that already happened** instead of **calling each other and waiting for an answer**.

Think of a newspaper. The newsroom prints "EARTHQUAKE HIT TOKYO" and drops the paper on the street. It does not phone every reader individually, and it does not care whether the insurance company, the seismologist, or nobody at all reads it. Anyone interested picks it up and reacts. That's an event. Compare that to calling your bank and saying "transfer $500 to Alice" — that's a **command**: one specific recipient, and you very much stay on the line to hear whether it worked.

An event-driven system is a system built mostly out of newspapers instead of phone calls.

---

## Why does it matter?

Without events, every new feature forces you to modify old code. Your `OrderService` charges the card. Then someone wants an email — you edit `OrderService`. Then loyalty points — you edit `OrderService`. Then a fraud check, an analytics ping, a warehouse reservation, a Slack alert. Now `placeOrder()` is a 400-line function that calls seven services in sequence, takes 3 seconds, and fails entirely if the Slack API is down. That is the disease. Events are the cure: `OrderService` publishes `OrderPlaced` and stops caring.

**Interview angle:** Almost every distributed-system design question ends up here. "How do you decouple the order flow from notifications?" "What happens if the payment succeeds but the DB write fails?" "Choreography or orchestration?" If you can talk crisply about commands vs events, the outbox pattern, and why you'd *not* use event sourcing, you sound like someone who has actually run a distributed system.

**Real-work angle:** The single most common production bug in event-driven systems is the **dual-write problem** — your DB says the order exists, Kafka never got the event, and now the warehouse will never ship it. You will hit this. The outbox pattern (later in this doc) is the fix, and knowing it puts you ahead of most mid-level engineers.

---

## The core idea — explained simply

### The Restaurant Kitchen Analogy

Walk into a busy restaurant kitchen.

**The command path.** The waiter walks up to the grill chef and says: *"Table 4 wants one medium-rare steak."* This is a **command**. It is imperative (do this), it is addressed to exactly one person (the grill chef, not the pastry chef), the waiter expects a result (a steak), and if the chef says "we're out of steak," the waiter absolutely cares — he has to go back and tell Table 4.

**The event path.** Later, the chef slams a bell and shouts: *"ORDER 47 IS UP!"* This is an **event**. It is a past-tense fact — order 47 *is already plated*. He shouts it into the room. He does not know who is listening. Maybe the waiter grabs it. Maybe the expediter checks the garnish. Maybe the manager notes the ticket time for the weekly report. Maybe nobody is listening at all and the food sits under the heat lamp. The chef does not wait, does not check, does not care. He's already on the next ticket.

Now here's the payoff. The restaurant hires a new "food photographer" for Instagram. To make her work, do you retrain the chef? No. She just stands in the kitchen and listens for the bell. **The chef's job did not change at all.** That's the entire promise of event-driven architecture: you can add consumers without touching the producer.

And here's the honest cost. The owner walks in and asks: *"What happens, exactly, when a customer orders a steak?"* Nobody can answer. The knowledge is scattered across the grill chef, the sauce chef, the expediter, the waiter, and the photographer — each of whom only knows their own reaction to one bell. That is the debugging pain of EDA, and it is real.

| Kitchen | Technical concept |
|---|---|
| Waiter tells grill chef "make a steak" | **Command** — imperative, one handler, sender expects a result |
| Chef shouts "ORDER 47 IS UP!" | **Event** — past-tense fact, broadcast, publisher doesn't care who hears |
| The bell + the room | **Message broker** (Kafka, RabbitMQ, SNS/SQS) |
| Photographer starts listening, chef unchanged | **Adding a consumer without touching the producer** |
| Two waiters both grab order 47 | **Duplicate delivery** — you need idempotency |
| Ticket rail with every order of the night, in order | **Event log / event sourcing** |
| Nobody can explain the full steak flow | **Choreography's debugging problem** |
| Head chef with a checklist driving each ticket | **Orchestration** |

---

## Key concepts inside this topic

### 1. Commands vs Events — get this crisp

This is the foundation. Most confused event-driven designs are confused because someone published a "command" and called it an event, or vice versa.

| | **Command** | **Event** |
|---|---|---|
| **Grammar** | Imperative — `ChargePayment` | Past tense — `PaymentCharged` |
| **Meaning** | "Please do this" | "This happened. It's a fact." |
| **Recipients** | Exactly **one** handler | **Zero or many** listeners |
| **Sender knows receiver?** | Yes — addressed to a specific service | No — publisher has no idea who subscribes |
| **Can it be rejected?** | Yes — the handler can refuse/validate/fail | No — you can't reject the past |
| **Does the sender care about the outcome?** | **Yes** — it expects a result or an error | **No** — fire and forget |
| **Coupling** | Sender → Receiver (tight-ish) | Publisher → nobody (loose) |
| **Adding a new handler** | Ambiguous — who's supposed to do it? | Free — just subscribe |
| **Typical transport** | HTTP/gRPC call, or a queue with one consumer group | Pub/sub topic with many subscribers |

**A worked contrast:**

```javascript
// COMMAND: imperative, one handler, I want a result, I care if it fails.
// If this throws, the user sees "Payment failed. Try another card."
const receipt = await paymentService.chargePayment({
  orderId: 'ord_991',
  amountCents: 4999,
  cardToken: 'tok_abc',
});

// EVENT: a fact that already happened. Published to a topic.
// Zero, one, or twelve services may react. The publisher will never know.
await broker.publish('payments', {
  type: 'PaymentCharged',            // past tense — NOT "ChargePayment"
  eventId: 'evt_7f3a...',            // unique — consumers dedupe on this
  occurredAt: '2026-07-12T10:04:11Z',
  data: { orderId: 'ord_991', amountCents: 4999, receiptId: 'rcpt_55' },
});
```

**The naming test:** if you can't name it in past tense without it sounding silly, it's probably a command in disguise. `SendEmail` is a command. `OrderPlaced` is an event. `EmailRequested` is... a command wearing an event costume, and that's actually fine and common — just be honest that only one service should handle it.

### 2. Request-response vs event-driven — what you gain and what you lose

Recall the classic synchronous flow: service A calls service B over HTTP and blocks until B answers. Simple, debuggable, and it works fine — right up until you have eight downstream calls.

**What you GAIN by going event-driven:**

- **Loose coupling.** The producer doesn't import, know about, or deploy alongside the consumer. Teams ship independently.
- **Independent scaling.** Email sending is slow (200ms each)? Run 50 email consumers. Order intake is fast? Run 3. Each consumer scales to its own workload, not to the slowest hop.
- **Add consumers for free.** The analytics team wants order data. They subscribe. **You write zero code.** This is the single biggest organizational win.
- **A natural audit trail.** The event log is a literal, ordered history of everything that happened. "Why is this order in this state?" — replay the events.
- **Resilience to downstream outages.** If the email service is down for 20 minutes, orders still get placed. Events pile up in the broker and drain when it recovers. In the synchronous world, the email service being down would have taken checkout down with it.
- **Latency for the user.** Checkout returns in 80ms after writing the order + publishing the event, instead of 2.4s waiting for six services.

**What you LOSE — and this part is not free:**

- **No simple "did it work?"** In request-response, a 200 means it worked. In EDA, `publish()` returning successfully means *the broker accepted the message*. It says nothing about whether the warehouse reserved stock. End-to-end success becomes something you have to *build* (correlation IDs, status endpoints, timeouts, sagas).
- **Eventual consistency everywhere.** The user places an order and immediately hits "My Orders" — and it isn't there yet, because the read-model consumer hasn't processed the event. You now have a UX problem, not just a technical one.
- **Debugging is genuinely painful.** A synchronous bug gives you one stack trace. An event-driven bug gives you: "the event was published, service B logged that it consumed it, but service C never saw it." You need distributed tracing (correlation IDs on every event) or you are blind.
- **Ordering complexity.** Kafka guarantees order *within a partition*, not across partitions. If `OrderPlaced` and `OrderCancelled` land in different partitions, a consumer can see the cancel before the place. You fix this by partitioning on a key (e.g. `orderId`) so all events for one order go to one partition, in order.
- **Duplicate delivery.** Almost every broker gives **at-least-once** delivery, not exactly-once. A consumer that crashes after doing the work but before acknowledging will get the message again. Every consumer must be **idempotent** — processing the same event twice must have the same effect as processing it once (see `85-idempotency.md`). Plus the operational weight: you now run Kafka, and Kafka needs care.

**Rule of thumb:** use request-response when the caller needs the answer *right now* to continue (auth check, payment authorization, inventory reservation at checkout). Use events for everything that happens *because* something happened (emails, analytics, search indexing, loyalty points, cache invalidation).

### 3. Choreography vs orchestration

Once you have multi-step business flows across services, you must choose who is in charge.

**Choreography** — nobody is in charge. Each service listens for events and reacts by emitting its own events. It's a dance where everyone knows their own steps.

```javascript
// Payment service — reacts to an order, emits a payment fact.
broker.subscribe('orders', 'payment-svc', async (event) => {
  if (event.type !== 'OrderPlaced') return;
  const receipt = await charge(event.data.orderId, event.data.totalCents);
  await broker.publish('payments', {
    type: 'PaymentCharged',
    data: { orderId: event.data.orderId, receiptId: receipt.id },
  });
});

// Inventory service — reacts to a payment, emits a stock fact.
broker.subscribe('payments', 'inventory-svc', async (event) => {
  if (event.type !== 'PaymentCharged') return;
  await reserveStock(event.data.orderId);
  await broker.publish('inventory', {
    type: 'StockReserved', data: { orderId: event.data.orderId },
  });
});

// Shipping service subscribes to 'inventory'. And so on, forever.
```

Elegant on a slide. But now answer this question: **"What actually happens when an order is placed?"** You cannot. The flow exists nowhere — not in any file, not in any diagram, only in the emergent behaviour of eight services. To trace it you grep eight repos for `subscribe(`. Worse: when payment succeeds but inventory is out of stock, *who refunds the card?* Someone has to write a compensating listener, and it lives in a service that has nothing to do with payments.

**Orchestration** — one component (an "orchestrator" or "saga") explicitly drives the steps and knows the whole plan.

```javascript
// The whole business flow lives in ONE readable place.
class OrderSaga {
  constructor({ payments, inventory, shipping, repo }) {
    Object.assign(this, { payments, inventory, shipping, repo });
  }

  async run(orderId) {
    const done = [];                              // for compensation on failure
    try {
      const receipt = await this.payments.charge(orderId);        // command
      done.push(() => this.payments.refund(receipt.id));          // undo step

      await this.inventory.reserve(orderId);                      // command
      done.push(() => this.inventory.release(orderId));           // undo step

      await this.shipping.schedule(orderId);                      // command
      await this.repo.setState(orderId, 'CONFIRMED');
    } catch (err) {
      // Run the undo steps in reverse — a "compensating transaction".
      // Note: you cannot ROLLBACK across services; you can only apologise.
      for (const undo of done.reverse()) await undo();
      await this.repo.setState(orderId, 'FAILED', err.message);
      throw err;
    }
  }
}
```

Notice the orchestrator sends **commands** (imperative, one handler, cares about the result) — while the services it drives may still *publish* events for everyone else. The two styles coexist.

| | **Choreography** | **Orchestration** |
|---|---|---|
| Control | Distributed — each service decides | Central coordinator decides |
| Message style | Events (broadcast) | Commands (addressed) + events |
| "What happens on order?" | Unanswerable without reading 8 repos | Read one class |
| Adding a step | Just subscribe — no existing code changes | Edit the orchestrator |
| Failure handling / rollback | Smeared across services, easy to get wrong | Explicit compensating steps in one place |
| Coupling | Lowest | Orchestrator knows all participants |
| Risk | Cyclic event storms, invisible logic | The orchestrator becomes a bottleneck / god object |
| Debuggability | Poor | Good — one state machine, one log |

**The honest recommendation:** use **orchestration for business-critical multi-step flows** (checkout, onboarding, refunds, anything with money or compensation). Use **choreography for simple fan-out reactions** (send an email, index in Elasticsearch, bump a counter) where each listener is independent and nobody needs to roll anything back. Teams that choreograph a 9-step payment flow always regret it around month eight. See `77-distributed-transactions.md` for the saga pattern in depth.

### 4. Event sourcing — the log IS the database

Normally you store **current state**: a row that says `balance = 320`. Every update destroys the previous value. Event sourcing inverts this: you store the **append-only sequence of events** as the source of truth, and derive current state by replaying them.

```javascript
// The events. Immutable, past-tense facts. This is the ONLY thing we persist.
const Events = Object.freeze({
  ACCOUNT_OPENED: 'AccountOpened',
  DEPOSITED: 'Deposited',
  WITHDRAWN: 'Withdrawn',
});

class BankAccount {
  constructor(id) {
    this.id = id;
    this.balance = 0;
    this.version = 0;          // how many events we've applied
    this.pending = [];         // new events not yet persisted
  }

  // --- The fold. Pure. No validation, no side effects. Just: state + event -> state.
  // This must NEVER throw: replaying history that already happened must always succeed.
  apply(event) {
    switch (event.type) {
      case Events.ACCOUNT_OPENED: this.balance = event.data.openingCents; break;
      case Events.DEPOSITED:      this.balance += event.data.amountCents; break;
      case Events.WITHDRAWN:      this.balance -= event.data.amountCents; break;
      default: break;            // unknown/retired event types are ignored, not fatal
    }
    this.version += 1;
    return this;
  }

  // --- Rebuild any account from its history. THIS is what "state" means now.
  static replay(id, history) {
    return history.reduce((acc, e) => acc.apply(e), new BankAccount(id));
  }

  // --- Commands: validate against CURRENT state, then RECORD a new fact.
  // Business rules live here, not in apply().
  withdraw(amountCents) {
    if (amountCents <= 0) throw new Error('Amount must be positive');
    if (amountCents > this.balance) throw new Error('Insufficient funds');
    this.#record({ type: Events.WITHDRAWN, data: { amountCents } });
  }

  deposit(amountCents) {
    if (amountCents <= 0) throw new Error('Amount must be positive');
    this.#record({ type: Events.DEPOSITED, data: { amountCents } });
  }

  #record(event) {
    event.occurredAt = new Date().toISOString();
    this.apply(event);        // update in-memory state immediately
    this.pending.push(event); // and queue the fact for durable append
  }
}

// --- Usage
const acct = BankAccount.replay('acc_1', [
  { type: Events.ACCOUNT_OPENED, data: { openingCents: 10_000 } },
  { type: Events.DEPOSITED,      data: { amountCents:   5_000 } },
  { type: Events.WITHDRAWN,      data: { amountCents:   2_000 } },
]);
console.log(acct.balance);  // 13000  <- derived by folding the log, never stored
acct.withdraw(3_000);
console.log(acct.balance);  // 10000
console.log(acct.pending);  // [ { type: 'Withdrawn', data: { amountCents: 3000 }, ... } ]
// The repository appends acct.pending to the event store. Nothing is ever UPDATEd.
```

**The superpowers:**

- **A complete audit log, for free.** Not a `changes` table someone remembers to write to — the log *is* the truth. Regulators love this; it's why banking and trading systems use it.
- **Time travel.** Want the balance as of March 3rd? `replay(id, history.filter(e => e.occurredAt < '2026-03-04'))`. You can reconstruct *any* past state exactly.
- **Rebuild a new read model from history.** Product wants a "monthly spending by category" screen that never existed. You write a new projection and replay five years of events through it. In a state-based system that data is simply gone forever — you never recorded it.
- **Debug by replay.** A customer's balance is wrong. Replay their events one at a time and watch exactly which one broke it.

**The real costs — and they are heavy:**

- **Schema evolution.** In 2031 you'll be replaying an event written in 2026 whose shape you've since changed. You can never "migrate" the past — the log is immutable. You must either version event types (`WithdrawnV2`) and keep upcasting functions forever, or write tolerant `apply()` handlers that cope with missing fields. This debt only grows.
- **GDPR vs immutability.** A user invokes "delete my data." Your source of truth is an append-only log that you have sworn never to mutate. These are in direct conflict. The standard escape is **crypto-shredding**: encrypt each user's personal fields with a per-user key, store keys in a separate mutable keystore, and on deletion **destroy the key**. The events remain, but the PII inside them is now permanently unreadable ciphertext.
- **Replay time.** An account with 4 million events takes minutes to load. Fix: **snapshots**.
- **Most systems do not need this.** If you don't have a hard audit requirement, a genuine need for time travel, or a domain where the *history* is the product (ledgers, trading, order books, version control) — use a normal database. Event sourcing is a specialist tool with a permanent tax, not a default.

**Snapshots — the fix for replay time:**

```javascript
// Every N events, persist the derived state. Then replay only what came after.
const SNAPSHOT_EVERY = 500;

async function load(id, store) {
  const snap = await store.getLatestSnapshot(id);        // { version, balance } | null
  const acct = new BankAccount(id);
  if (snap) {
    acct.balance = snap.balance;
    acct.version = snap.version;
  }
  // Only the tail of the log — 500 events max, not 4 million.
  const tail = await store.getEventsAfter(id, acct.version);
  for (const e of tail) acct.apply(e);
  return acct;
}

async function save(acct, store) {
  await store.append(acct.id, acct.pending);   // the events are the truth
  acct.pending = [];
  if (acct.version % SNAPSHOT_EVERY === 0) {
    // A snapshot is a CACHE, never the truth. Delete them all and the system
    // still works — it just gets slow. That property is what keeps you safe.
    await store.putSnapshot(acct.id, { version: acct.version, balance: acct.balance });
  }
}
```

### 5. CQRS — separate the write model from the read model

CQRS (Command Query Responsibility Segregation) says: **the model you write with and the model you read with do not have to be the same model.**

Writes go to a normalized, validated, consistency-focused store. Reads are served from **projections** — denormalized views shaped for one specific query, built by consuming the events the write side produced.

```javascript
// WRITE SIDE — normalized, enforces invariants, one row per fact.
async function handlePlaceOrder(cmd) {
  const order = Order.create(cmd);         // validates: stock, price, address
  await orderRepo.save(order);             // normalized tables
  await broker.publish('orders', order.pullEvents());
}

// READ SIDE — a projection. Purpose-built, denormalized, joins already done.
// This exists so the "My Orders" page is a single key lookup, not a 5-table join.
broker.subscribe('orders', 'orders-projection', async (event) => {
  if (event.type === 'OrderPlaced') {
    await readDb.orderSummaries.insert({
      orderId:    event.data.orderId,
      userId:     event.data.userId,
      status:     'PLACED',
      itemNames:  event.data.items.map(i => i.name), // denormalized on purpose
      totalCents: event.data.totalCents,
    });
  }
  if (event.type === 'OrderShipped') {
    await readDb.orderSummaries.update({ orderId: event.data.orderId },
      { status: 'SHIPPED', trackingId: event.data.trackingId });
  }
});

// The query is now trivially fast and trivially scalable.
const myOrders = await readDb.orderSummaries.find({ userId });
```

**Why it pairs naturally with event sourcing:** event sourcing gives you a write model that is a stream of facts, and *no* queryable current state. You literally cannot run `SELECT * FROM orders WHERE status='SHIPPED'` against an event log. So you *need* projections — and projections are exactly the read side of CQRS. The two patterns fit like a hand in a glove, which is why they're almost always taught together. (But note: **you can do CQRS without event sourcing**, and most teams should. A plain Postgres write model that emits events to build a denormalized read view is CQRS, and it's far cheaper.)

**The cost — read-your-own-writes.** The projection is updated *after* the write, by an async consumer. Typical lag is 50–500ms. So:

1. User clicks "Place order." Write succeeds. `202 Accepted` in 60ms.
2. Browser redirects to "My Orders."
3. The projection hasn't caught up. **The order isn't there.** The user thinks it failed and orders again.

Three ways to handle it in the UI, in increasing order of effort:

- **Optimistic UI.** The write side returns the `orderId`; the client immediately renders the order from what it already knows, without re-fetching. Cheapest, works well, hides the lag entirely.
- **Poll / subscribe.** Return `202` plus a status URL. The client polls (or listens over a WebSocket) until the projection reports `PLACED`. Honest, shows a spinner, good for slow flows like refunds.
- **Read-your-writes routing.** Stamp the client with the version/offset it just wrote (e.g. a cookie holding the Kafka offset) and have the read API either wait for the projection to reach that offset or fall back to the write store. Most correct, most work.

The one thing you must not do is pretend the lag doesn't exist. See `76-eventual-consistency.md`.

### 6. The dual-write problem and the Outbox pattern

**This is the most practically valuable part of this document.** It's the bug that will actually bite you.

Here is the code almost everyone writes first:

```javascript
// ❌ BROKEN. This is the dual-write bug.
async function placeOrder(input) {
  const order = await db.orders.insert(input);          // (1) write to Postgres
  await kafka.publish('orders', {                       // (2) write to Kafka
    type: 'OrderPlaced',
    data: { orderId: order.id, userId: order.userId },
  });
  return order;
}
```

It looks fine. It is not fine. You are writing to **two systems that do not share a transaction**, and there is a window between them:

- **Crash between (1) and (2):** the order exists in Postgres. Kafka never got the event. The warehouse will never ship it, the customer never gets an email, and **nothing will ever fix this** — no retry, no cron, no alert. The order silently rots. This is the worst outcome because it is invisible.
- **Kafka is down at (2):** if you throw, the user sees an error even though the order *was* saved. If you swallow it, you get case 1.
- **"I'll just publish first, then write":** now a crash gives you an event for an order that doesn't exist. Consumers will `404` on it forever.
- **"I'll wrap it in a try/catch and roll back the DB":** the rollback itself can fail. And if Kafka actually *did* accept the message but the ack was lost, you've now rolled back an order the warehouse is already picking.

**You cannot make two independent systems atomic.** Distributed 2-phase commit (XA) is the textbook answer and it is, in practice, slow, poorly supported by Kafka, and operationally miserable. Nobody does it.

**The Outbox pattern** sidesteps the whole thing with one insight: *make the event write part of the same database transaction as the business write.* Then a separate process moves events from the DB to the broker.

```sql
-- The outbox lives in the SAME database as your business tables.
CREATE TABLE outbox (
  id           BIGSERIAL PRIMARY KEY,
  aggregate_id TEXT        NOT NULL,          -- e.g. the order id; use as the Kafka key
  event_type   TEXT        NOT NULL,          -- 'OrderPlaced'
  payload      JSONB       NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  sent_at      TIMESTAMPTZ                    -- NULL = not yet published
);

-- The relay only ever scans unsent rows, so keep that index tight.
CREATE INDEX outbox_unsent_idx ON outbox (id) WHERE sent_at IS NULL;
```

```javascript
// ✅ THE WRITE. One transaction. Both rows commit, or neither does.
// There is no window. The database's own atomicity is doing the work for us.
async function placeOrder(input) {
  return db.transaction(async (tx) => {
    const order = await tx.orders.insert(input);

    await tx.outbox.insert({
      aggregate_id: order.id,                    // partition key -> per-order ordering
      event_type:   'OrderPlaced',
      payload:      {
        eventId: crypto.randomUUID(),            // consumers dedupe on this
        orderId: order.id,
        userId:  order.userId,
        totalCents: order.totalCents,
      },
    });

    return order;   // COMMIT: either the order AND the event exist, or nothing does.
  });
}
```

```javascript
// ✅ THE RELAY. A separate loop (or process) that drains the outbox into Kafka.
// Guarantee: AT-LEAST-ONCE. It may publish a row twice; it will never lose one.
async function relayLoop() {
  for (;;) {
    // FOR UPDATE SKIP LOCKED lets you run several relay instances safely —
    // each grabs a different batch instead of fighting over the same rows.
    const rows = await db.query(`
      SELECT id, aggregate_id, event_type, payload
      FROM outbox
      WHERE sent_at IS NULL
      ORDER BY id                    -- id order == commit order == event order
      LIMIT 100
      FOR UPDATE SKIP LOCKED
    `);

    if (rows.length === 0) { await sleep(200); continue; }

    for (const row of rows) {
      await kafka.publish('orders', {
        key:   row.aggregate_id,      // same key -> same partition -> ordered per order
        value: { type: row.event_type, ...row.payload },
      });

      // If we crash HERE — published but not marked — the next loop republishes it.
      // That is fine and expected: consumers are idempotent (they dedupe on eventId).
      // Losing an event is unacceptable; sending it twice is merely annoying.
      await db.query('UPDATE outbox SET sent_at = now() WHERE id = $1', [row.id]);
    }
  }
}

// Housekeeping: the outbox is a queue, not an archive. Prune it or it grows forever.
setInterval(() => db.query(
  `DELETE FROM outbox WHERE sent_at < now() - INTERVAL '3 days'`,
), 60 * 60 * 1000);
```

Read the trade carefully: you have **converted an impossible problem (atomicity across two systems) into a solved one (at-least-once delivery + idempotent consumers).** You did not eliminate duplicates — you made duplicates the *only* failure mode, and duplicates are something you know how to handle.

**CDC / Debezium — the productionized version.** Polling the outbox works, but it costs you a query every 200ms and adds latency. The industrial version is **Change Data Capture**: Debezium tails the database's own replication log (Postgres WAL / MySQL binlog), sees the outbox `INSERT` the instant it commits, and publishes it to Kafka. No polling, sub-second latency, no relay code to maintain, and the ordering comes straight from the commit log. Same pattern, better plumbing. If you're already running Kafka Connect, use it.

---

## Visual / Diagram description

### Diagram 1: Choreography vs Orchestration

```
CHOREOGRAPHY — no central brain. Each service listens and reacts.

              ┌──────────────────── EVENT BUS (Kafka) ────────────────────┐
              │                                                            │
              │   orders            payments           inventory           │
              └──▲─────┬──────────────▲──────┬────────────▲─────┬──────────┘
                 │     │              │      │            │     │
        publishes│     │subscribes    │      │            │     │
      OrderPlaced│     ▼   PaymentCharged    ▼  StockReserved   ▼
          ┌──────┴───┐   ┌──────────┐   ┌───────────┐   ┌───────────┐
          │  Order   │   │ Payment  │   │ Inventory │   │ Shipping  │
          │ Service  │   │ Service  │   │  Service  │   │  Service  │
          └──────────┘   └──────────┘   └───────────┘   └───────────┘
                              │               │               │
                         publishes        publishes       publishes
                      PaymentCharged   StockReserved   ShipmentBooked

   "What happens when an order is placed?"  -> nobody can tell you.
   The flow exists in no file. Refund-on-failure logic ends up smeared
   across three services that shouldn't know about each other.


ORCHESTRATION — one coordinator drives the steps and owns the plan.

                            ┌──────────────────────────┐
                            │      Order Saga          │
                            │   (state machine)        │
                            │                          │
                            │  1. charge payment       │
                            │  2. reserve stock        │
                            │  3. book shipment        │
                            │  X. on failure: undo 2,1 │
                            └───┬───────┬──────────┬───┘
                                │       │          │
                 command:charge │       │          │ command:schedule
                                │       │ command:reserve
                                ▼       ▼          ▼
                        ┌──────────┐ ┌──────────┐ ┌──────────┐
                        │ Payment  │ │Inventory │ │ Shipping │
                        │ Service  │ │ Service  │ │ Service  │
                        └────┬─────┘ └────┬─────┘ └────┬─────┘
                             │            │            │
                             └────────────┴────────────┘
                                     replies / events
                                          │
                                          ▼
                                 saga advances or compensates

   The whole business flow is ONE readable class with ONE state machine.
   You can query "which sagas are stuck in step 2?" — try that with choreography.
```

The top diagram shows events flowing *outward* — each service publishes a fact and someone downstream happens to be listening. Notice there is no arrow that represents "the order flow"; it only exists as a chain of coincidences. The bottom diagram shows commands flowing *out from the centre* and results flowing back, with one box that knows the whole plan and the undo steps.

### Diagram 2: The Outbox Flow

```
   ┌─────────────────────────────────────────────────────────────────┐
   │                      ORDER SERVICE                               │
   │   placeOrder()                                                   │
   │       │  BEGIN TRANSACTION ───────────────────────────┐          │
   │       ├──▶ INSERT INTO orders  (business data)        │  ATOMIC  │
   │       │                                               │  both or │
   │       └──▶ INSERT INTO outbox  (OrderPlaced event)    │  neither │
   │          COMMIT ──────────────────────────────────────┘          │
   └───────────────────────────┬──────────────────────────────────────┘
                               ▼
              ┌────────────────────────────────────┐
              │        POSTGRES (one DB)           │
              │  ┌──────────┐   ┌───────────────┐  │
              │  │  orders  │   │    outbox     │  │
              │  │  id ...  │   │ id | sent_at  │  │
              │  │          │   │  1 |  NULL ◀──┼──┼── not published yet
              │  │          │   │  2 |  NULL    │  │
              │  └──────────┘   └───────┬───────┘  │
              └─────────────────────────┼──────────┘
                                        │
                     poll (SELECT ... WHERE sent_at IS NULL)
                     or CDC (tail the WAL — Debezium)
                                        │
                               ┌────────▼─────────┐
                               │   RELAY PROCESS  │
                               │  read → publish  │
                               │  → mark sent_at  │
                               └────────┬─────────┘
                                        │  at-least-once
                                        ▼
                          ┌──────────────────────────┐
                          │      KAFKA  "orders"     │
                          └───┬──────────┬───────┬───┘
                              │          │       │
                        ┌─────▼───┐ ┌────▼───┐ ┌─▼────────┐
                        │ Email   │ │Warehouse│ │Analytics │
                        │Consumer │ │Consumer │ │Consumer  │
                        └─────────┘ └────────┘ └──────────┘
                         (each one must be IDEMPOTENT — dedupe on eventId,
                          because the relay may replay a row it published
                          just before crashing)
```

The crucial thing to see: **there is no moment in time where the order exists but the event does not.** The transaction boundary guarantees it. The relay may be slow, may crash, may send a duplicate — but it can never *lose* an event, because the event is sitting durably in the same database as the order, waiting.

---

## Real world examples

### 1. Uber — Kafka as the company's nervous system

Uber runs one of the largest Kafka deployments in the world, moving trillions of messages a day. Trip lifecycle facts — `TripRequested`, `DriverAccepted`, `TripCompleted` — are published once and consumed by dozens of independent systems: dynamic pricing, driver payouts, fraud detection, ETA models, the data lake, and near-real-time analytics. The trip service does not know that the ML feature pipeline exists. That is exactly the "add a consumer without touching the producer" property, applied at company scale — a new data-science team can start consuming trip events without a single line of change in the trip service.

### 2. LinkedIn — where Kafka came from

Kafka was built at LinkedIn precisely because the point-to-point integration graph had become unmanageable: every new system needed a custom pipeline into every other system. The fix was to invert it — every system publishes its facts to a central, durable, replayable log, and every consumer reads from that log at its own pace. The durable-and-replayable part matters: a consumer that was broken for six hours can rewind its offset and reprocess, which is impossible with a fire-and-forget bus.

### 3. Banking and trading ledgers — event sourcing where it genuinely belongs

Double-entry bookkeeping has been event sourcing since the 15th century: you never edit a ledger entry, you post a *correcting* entry. Modern payment ledgers (and systems built on them) follow the same rule — the balance is a derived number, and the immutable list of postings is the truth. This is the domain where event sourcing's costs are worth paying, because a regulator can and will ask "prove what this balance was on March 3rd, and show every transaction that produced it." Note the contrast with a typical CRUD product: your average SaaS dashboard has no such requirement, and event sourcing there is pure tax.

---

## Trade-offs

| Aspect | Request-Response (sync) | Event-Driven (async) |
|---|---|---|
| Coupling | Caller must know the callee | Publisher knows nobody |
| Latency for the user | Sum of all downstream hops | Just the write + publish |
| "Did it work?" | Trivial — check the status code | Hard — needs tracing, sagas, status endpoints |
| Downstream outage | Takes you down with it | Events queue up and drain later |
| Adding a consumer | Modify the producer | Free |
| Debugging | One stack trace | Distributed tracing or you're blind |
| Consistency | Strong, immediate | Eventual |
| Duplicate handling | Not needed | Mandatory — every consumer must be idempotent |

| Choice | You gain | You pay |
|---|---|---|
| **Choreography** | Lowest coupling, no central component | Flow is invisible; rollback logic smeared everywhere |
| **Orchestration** | Readable flow, explicit compensation, debuggable | A central component that knows everyone |
| **Event sourcing** | Perfect audit log, time travel, rebuildable read models | Schema evolution forever, GDPR pain, replay cost, high complexity |
| **CQRS** | Reads scale independently, purpose-built query models | Two models to maintain; eventual consistency hits the UX |
| **Outbox** | Kills the dual-write bug permanently | At-least-once delivery (duplicates), a relay to run, a table to prune |

**The sweet spot:** Use events for reactions and fan-out; use synchronous calls when the caller genuinely needs the answer to proceed. Orchestrate anything with money or compensation; choreograph simple notifications. **Always** use the outbox pattern the moment you write to a DB and publish to a broker in the same operation — it is cheap and it prevents a class of silent, unrecoverable data loss. Adopt event sourcing only when the history *is* the product; otherwise you're paying a permanent tax for a feature you'll never use.

---

## Common interview questions on this topic

### Q1: "What's the difference between a command and an event?"
**Hint:** A command is imperative (`ChargePayment`), goes to exactly one handler, and the sender expects a result and cares if it fails. An event is a past-tense fact (`PaymentCharged`), is broadcast to zero-or-many listeners, and the publisher neither knows nor cares who consumes it. You can reject a command; you cannot reject the past. The give-away: commands couple sender to receiver, events don't.

### Q2: "Your service writes an order to Postgres and publishes to Kafka. What can go wrong?"
**Hint:** This is the dual-write problem — the classic trap. If the process crashes between the two writes, the order exists but the event is gone forever, and nothing will ever repair it. Two systems cannot be made atomic (and 2PC/XA is impractical with Kafka). The fix is the **outbox pattern**: insert the event into an `outbox` table inside the same DB transaction as the order, then have a relay (polling, or CDC via Debezium) publish it and mark it sent. That converts an unsolvable atomicity problem into at-least-once delivery, which you handle with idempotent consumers.

### Q3: "Choreography or orchestration for a checkout flow?"
**Hint:** Orchestration. Checkout is business-critical, multi-step, and needs compensation (if inventory fails after the card is charged, someone must refund). Choreography would smear that refund logic across services and leave nobody able to answer "what happens when an order is placed?". Say you'd use a saga orchestrator with explicit compensating transactions — and then add that you'd still *publish* events (`OrderPlaced`) for the fan-out consumers like email and analytics. The two coexist.

### Q4: "When would you NOT use event sourcing?"
**Hint:** Almost always. Say it plainly — that's the answer they want to hear from a senior candidate. The costs are permanent: schema evolution of years-old events, GDPR deletion vs an immutable log (mention crypto-shredding as the escape hatch), replay time (mitigated by snapshots), and a much steeper learning curve for the team. Use it when the history is the product — ledgers, trading, audit-heavy domains. For a CRUD SaaS app, a normal database plus an events table is enough.

### Q5: "With CQRS, a user places an order and doesn't see it on their orders page. What do you do?"
**Hint:** That's read-your-own-writes violation caused by projection lag (typically 50–500ms). Three options, in increasing cost: optimistic UI (render from what the client already knows — cheapest and usually enough); poll or subscribe on a status endpoint until the projection catches up; or read-your-writes routing, where the client carries the version/offset it wrote and the read API waits for the projection to reach it (or falls back to the write store). Don't pretend the lag doesn't exist — design the UX around it.

---

## Practice exercise

### Build the Outbox

Build a tiny, runnable Node.js simulation — no Kafka, no Postgres, just in-memory objects. Roughly 30 minutes.

**What to produce:**

1. A fake `Database` class with two "tables" (plain arrays): `orders` and `outbox`, and a `transaction(fn)` method that either commits both writes or discards both. Implement it by staging writes in a temp buffer and only merging on success.
2. A fake `Broker` class with `publish(topic, event)` and `subscribe(topic, handler)`.
3. `placeOrder(input)` — inserts the order **and** the `OrderPlaced` event into the outbox, inside one transaction.
4. A `relay()` function that reads unsent outbox rows, publishes them, and marks `sentAt`.
5. Two subscribers on the `orders` topic: an `EmailConsumer` and a `WarehouseConsumer`. Each prints what it did.

**Now prove the point — three experiments:**

- **A.** Make the relay throw *after* publishing but *before* marking `sentAt`. Run it again. What happens? (You should see the same event delivered twice.) Now make `EmailConsumer` idempotent by keeping a `Set` of processed `eventId`s, and show that the duplicate causes no second email.
- **B.** Write the **broken** version — insert the order, then publish directly, with a `throw new Error('crash')` in between. Show that the order exists but no consumer ever hears about it, and that *nothing in your system can ever detect or repair this*. This is the bug you're trying to make impossible.
- **C.** Add a third consumer — `AnalyticsConsumer` — and confirm you changed **zero lines** of `placeOrder()`. Write down, in one sentence, why that was free.

If experiment B doesn't make you slightly uncomfortable, run it again and stare at it.

---

## Quick reference cheat sheet

- **Command** = imperative, one handler, sender wants a result and cares if it fails (`ChargePayment`).
- **Event** = past-tense fact, broadcast to zero-or-many, publisher doesn't know or care who listens (`PaymentCharged`). If it reads as an instruction, it's a command.
- **The big win:** add a new consumer without touching the producer — an *organizational* win as much as a technical one.
- **The big loss:** there is no simple "did the whole thing work?" — you must build it with correlation IDs, tracing, and sagas.
- **At-least-once is the norm.** Exactly-once is mostly a marketing word. Every consumer must be **idempotent** — dedupe on `eventId`.
- **Ordering** is guaranteed only *within a partition* — key by the entity id (`orderId`) so one entity's events stay ordered.
- **Choreography** = each service reacts; elegant, but nobody can explain the flow and rollback logic gets smeared everywhere.
- **Orchestration** = a saga drives the steps; visible, debuggable, compensatable. **Use it for anything involving money.**
- **Event sourcing** = store events, derive state by replaying them. Superpowers: audit log, time travel, rebuildable read models.
- **Event sourcing costs:** schema evolution forever, GDPR vs immutability (→ **crypto-shredding**), replay time (→ **snapshots**, which are a cache and never the truth). Most systems should not do this.
- **CQRS** = separate write model from read projections. Pairs with event sourcing because a log can't be queried — but you can (and usually should) do CQRS *without* it.
- **CQRS cost:** the read model lags. Handle read-your-own-writes with optimistic UI, polling, or offset-based routing.
- **Dual-write problem:** DB write + broker publish are not atomic. A crash between them loses the event *silently and forever*.
- **Outbox pattern:** write the event to an `outbox` table **in the same transaction** as the business data; a relay publishes it and marks it sent. **CDC/Debezium** is the productionized version (tail the WAL, no polling).

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [67 — Message Queues](./67-message-queues.md) — the transport layer (Kafka, RabbitMQ, SQS) that every event-driven system is built on |
| **Next** | [77 — Distributed Transactions](./77-distributed-transactions.md) — sagas and compensating transactions, the deep dive on orchestration |
| **Related** | [85 — Idempotency](./85-idempotency.md) — mandatory reading; at-least-once delivery means your consumers *will* see duplicates |
| **Related** | [76 — Eventual Consistency](./76-eventual-consistency.md) — the price of CQRS read models and async projections |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — events are what make service boundaries actually hold |
| **Related** | [41 — Observer Pattern](./41-pattern-observer.md) — the same publish/subscribe idea, in-process, at class level |
| **Related** | [43 — Command Pattern](./43-pattern-command.md) — commands as first-class objects, the LLD mirror of this doc's command/event split |
| **Related** | [87 — Batch vs Stream Processing](./87-batch-vs-stream-processing.md) — what you do with the event stream once you have one |
