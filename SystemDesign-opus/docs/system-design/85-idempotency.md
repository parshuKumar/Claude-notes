# 85 — Idempotency — Designing Safe Retries
## Category: HLD Components

---

## What is this?

An operation is **idempotent** if performing it N times has exactly the same effect as performing it once. Not the same *response* necessarily — the same **effect on the world**.

Think of a light switch. "**Set the switch to ON**" is idempotent: flip it to ON five times and the light is still just... on. Nothing extra happened. But "**toggle the switch**" is NOT idempotent: do it five times and you end up in a completely different state than doing it once. Same button, same finger, wildly different safety properties.

Now replace "light switch" with "charge the customer £500" and you understand why this topic exists.

---

## Why does it matter?

Because **duplicate requests are not a bug you can fix — they are a permanent feature of distributed systems.** You do not get to opt out. Your only choice is whether your system handles them gracefully or corrupts data.

Here is the argument, and I want you to internalise it, because it is the entire point of this document.

**A client sends a request. It gets no response — a timeout, a connection reset, a dead Wi-Fi hop.** What happened on the server?

1. The request **never arrived**. The server has no idea you exist.
2. The request **arrived and was fully processed**, and the *response* was lost on the way back.
3. The request **is still being processed** right now.

From the client's position, **these three are indistinguishable.** There is no packet, no header, no clever trick that tells them apart. This isn't a limitation of your HTTP library — it's a property of networks. The client sees the exact same thing (silence) in all three cases.

So the client has exactly two options, and both are bad:

- **Retry** → if the truth was (2), you just did the thing twice. Double charge.
- **Give up** → if the truth was (1), the operation is silently lost. The customer's order never happened.

There is no third option. You must pick your poison.

**Idempotency is how you make "retry" the safe choice.** If the server handles duplicates correctly, retrying costs you nothing — a duplicate is absorbed harmlessly. And so you retry aggressively, and you never lose operations.

This is also the resolution to a thing you already met in [67 — Message Queues](./67-message-queues.md): **exactly-once delivery is a myth.** Kafka, SQS, RabbitMQ — they all give you *at-least-once* delivery. Your consumer WILL see the same message twice. It's not a question of if.

But you don't actually need exactly-once *delivery*. You need exactly-once **effect**. And:

> **at-least-once delivery + idempotent processing = effectively-once processing**

That equation is the whole game. It's why idempotency is not a nice-to-have — it is the load-bearing wall of every reliable system you will ever build.

**In interviews:** the moment you say "the client retries," a good interviewer will ask "what if the first one actually succeeded?" If you don't have an answer, you fail. Bringing up idempotency keys *unprompted* is one of the strongest senior signals available to you.

**At work:** payments, order creation, queue consumers, webhook receivers, cron jobs, SAGA compensations. Every one of them.

### The horror story

The user taps **"Pay £500"**. The request reaches your server. Your server charges the card. Your server writes to the DB. Your server begins writing the HTTP 200 response... and the user walks into a lift and loses signal.

The mobile app sees a timeout. The mobile app, being well-written, retries. The request arrives again. Your server charges the card. **Again.**

£1000 gone. A furious customer. A chargeback. A postmortem. And the horrible part: **every single component behaved correctly.** The network dropped a packet (normal). The client retried (correct). The server processed the request it received (correct). The only bug was that nobody made the operation idempotent.

---

## The core idea — explained simply

### The Restaurant Order-Ticket Analogy

You're a waiter. You take an order and shout it through the kitchen hatch: *"One steak, well done!"*

The kitchen is loud. You don't hear a confirmation. Did the chef hear you? Maybe. Maybe not.

**The naive waiter** shouts again: *"One steak, well done!"* — and now there might be two steaks. Or one. He genuinely does not know. He has made the situation *worse*, because now he can't even reason about it.

**The professional waiter** does something different. He writes the order on a **numbered ticket** — `#4471` — and spikes it on the rail:

> *"Ticket 4471: one steak, well done."*

Now he can shout it as many times as he likes. The chef's rule is simple: **look at the rail. If ticket 4471 is already there, ignore the shout.** If it's not there, spike it and start cooking.

One steak. Guaranteed. No matter how many times the waiter shouts, no matter how loud the kitchen gets.

And notice what the ticket number bought him: **the freedom to retry without fear.** He can now be paranoid and repeat himself constantly, and paranoia costs him nothing. That's the trade you're making with idempotency — you spend a little storage and a unique index, and you buy the right to retry.

| Restaurant | Distributed system |
|---|---|
| The waiter | The client (mobile app, another service, a queue consumer) |
| Shouting the order | Sending the HTTP request |
| The noisy kitchen | The unreliable network |
| Not hearing a confirmation | Timeout / connection reset / lost response |
| The ticket number `#4471` | The **idempotency key** (a UUID) |
| The ticket rail | The **idempotency store** (a DB table, Redis) |
| "Is 4471 already on the rail?" | Look up the key before processing |
| Spiking the ticket **before** cooking | Atomic `INSERT` with a `UNIQUE` constraint |
| Handing back the same plate | **Replaying the stored response** |
| One steak, no matter what | **Effectively-once processing** |

The key insight, the one people miss: **the ticket is spiked BEFORE the steak is cooked.** If you cooked first and spiked after, two shouts arriving simultaneously would both see an empty rail and both start cooking. Order matters. We'll come back to this — it's the race condition at the heart of the whole design.

---

## Key concepts inside this topic

### 1. Idempotent vs. not — the mental test

Ask yourself one question: **"If I run this twice, is the world any different than if I ran it once?"**

```javascript
// NOT idempotent — the effect compounds every time
balance = balance - 50;              // run twice → -100
counter++;                           // run twice → +2
items.push(newItem);                 // run twice → two items
await sendEmail(user, 'Welcome!');   // run twice → two emails, one annoyed user

// IDEMPOTENT — the effect is absolute, not relative
balance = 100;                       // run twice → still 100
status = 'PAID';                     // run twice → still PAID
items = [newItem];                   // run twice → still one item
await db.query('DELETE FROM sessions WHERE id = $1', [id]); // gone is gone
```

The pattern: **deltas are dangerous, absolutes are safe.** "Subtract 50" depends on what was there before. "Set to 100" does not.

Read-only operations are trivially idempotent — a `GET` changes nothing, so N of them change nothing N times over.

### 2. HTTP methods — which ones are idempotent

The HTTP spec (RFC 9110) makes explicit promises about which methods must be idempotent. Clients, proxies, and load balancers *rely* on these promises — some will automatically retry an idempotent method on a network error.

| Method | Idempotent? | Safe (no side effects)? | Why |
|---|---|---|---|
| `GET` | Yes | Yes | Reading doesn't change anything |
| `HEAD` | Yes | Yes | Same as GET, minus the body |
| `OPTIONS` | Yes | Yes | Pure metadata |
| `PUT` | **Yes** | No | It's a full replace — "set the resource to THIS". Repeating it lands on the same final state |
| `DELETE` | **Yes** | No | Deleting an already-deleted thing leaves it deleted. Return `204 No Content` either way |
| `POST` | **NO** | No | "Create a new thing" — each call creates *another* thing |
| `PATCH` | **NO** | No | Only if the patch body is absolute (`{"status":"PAID"}`) rather than a delta (`{"increment": 1}`). The spec does not guarantee it |

**The honest note nobody tells you:** "idempotent by spec" is a promise *you* have to keep. The spec doesn't enforce anything. If you write `DELETE /orders/42` and it throws `404` on the second call, or `PUT /users/7` and it appends to an audit log each time in a way that matters, **you have broken the contract** — and clients that trusted the spec and auto-retried will now hurt you.

Two clarifications people get wrong:

- **Idempotent ≠ same response.** `DELETE /orders/42` may return `204` then `404`. The *effect* is what must be identical, not the status code. (Though returning `204` both times is friendlier.)
- **Idempotent ≠ safe.** `DELETE` is idempotent but definitely not safe — it destroys data. "Safe" means "no side effects at all."

And **`POST` is the problem child.** It's the method for "create an order", "charge a card", "send a message" — precisely the operations where a duplicate is catastrophic. Which is exactly why the next section exists.

### 3. Idempotency keys — the standard solution

This is the industry-standard fix for non-idempotent operations, and it is **exactly how Stripe's API works.** Every Stripe endpoint that creates something accepts an `Idempotency-Key` header, and you should copy this design wholesale.

**The client's job:**
1. Generate a UUID **per logical operation** — one per "the user tapped Pay", *not* one per HTTP attempt.
2. Send it as a header: `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`
3. On **every retry of that same logical operation, send the SAME key.**

That last point is where people go wrong. If you generate a fresh UUID inside your retry loop, you have built an elaborate no-op. The key must be generated *outside* the loop:

```javascript
import { randomUUID } from 'node:crypto';

async function payWithRetries(amountPence, cardToken) {
  // The key identifies the INTENT, not the attempt.
  // Generated ONCE, outside the loop. This is the whole trick.
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      const res = await fetch('https://api.example.com/payments', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Idempotency-Key': idempotencyKey,   // SAME key every attempt
        },
        body: JSON.stringify({ amountPence, cardToken }),
      });

      // 4xx (except 409) means we did something wrong — retrying won't help.
      if (res.status >= 400 && res.status < 500 && res.status !== 409) {
        throw new Error(`Client error ${res.status}`);
      }
      if (res.ok) return res.json();
    } catch (err) {
      // Network error: we have NO IDEA if the server processed it.
      // Because of the key, we don't have to care. Retry.
    }

    // Exponential backoff + jitter — see section 7.
    const backoffMs = Math.min(1000 * 2 ** attempt, 30_000);
    const jittered = Math.random() * backoffMs;
    await new Promise((r) => setTimeout(r, jittered));
  }

  throw new Error('Payment failed after 5 attempts');
}
```

**The server's job** — four cases, and you must handle all four:

1. Key **absent** → claim it atomically, process the request, then **save the response against the key**.
2. Key **present and complete** → do NOT process again. **Replay the stored response verbatim.**
3. Key **present but still in-flight** (a concurrent duplicate) → return `409 Conflict`, or make the caller wait.
4. Key **present but with a different request body** → return `422`. Someone is reusing a key for a different operation, which is a client bug and you should tell them loudly.

### 4. The store, and the race that will bite you

The schema. Note the constraint — it is the single most important line in this document:

```sql
CREATE TABLE idempotency_keys (
  -- Scope the key to the user. Otherwise tenant A can guess/collide with
  -- tenant B's key and replay THEIR response back to them. This is a
  -- security bug, not just a correctness one.
  user_id        BIGINT      NOT NULL,
  idempotency_key TEXT       NOT NULL,

  request_hash   TEXT        NOT NULL,   -- detects key reuse with a different body
  state          TEXT        NOT NULL,   -- 'IN_PROGRESS' | 'COMPLETED'
  response_code  INT,
  response_body  JSONB,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at     TIMESTAMPTZ NOT NULL,   -- 24h is the typical TTL

  -- THIS is what makes the whole design race-safe.
  PRIMARY KEY (user_id, idempotency_key)
);

CREATE INDEX idx_idem_expiry ON idempotency_keys (expires_at);
```

**Now the race.** Here is code that looks obviously correct and is catastrophically broken:

```javascript
// ============ BAD — check-then-insert. A textbook race condition. ============
async function handlePaymentBAD(req, res) {
  const key = req.headers['idempotency-key'];

  const existing = await db.query(
    'SELECT * FROM idempotency_keys WHERE user_id = $1 AND idempotency_key = $2',
    [req.user.id, key],
  );

  if (existing.rows.length > 0) {
    return res.status(200).json(existing.rows[0].response_body);
  }

  // ---- DANGER ZONE ----
  // Two duplicate requests, arriving 2ms apart, land on two different servers.
  // BOTH ran the SELECT above. BOTH found nothing. BOTH are now here.
  // BOTH will charge the card.
  // No amount of application-level cleverness closes this window.

  const charge = await stripe.charges.create({ amount: req.body.amountPence });
  await db.query('INSERT INTO idempotency_keys ...', [/* ... */]);
  res.status(201).json(charge);
}
```

You cannot fix this with more application code. The gap between "I checked" and "I inserted" is a window, and at scale, *every window gets hit*. Two servers, two connections, two transactions — application memory cannot coordinate them.

**Only the database can.** Make the *insert itself* be the check:

```javascript
// ============ GOOD — insert-first. The UNIQUE constraint arbitrates. ============
import crypto from 'node:crypto';

const IN_PROGRESS = 'IN_PROGRESS';
const COMPLETED = 'COMPLETED';

function hashRequest(body) {
  return crypto.createHash('sha256').update(JSON.stringify(body)).digest('hex');
}

async function handlePayment(req, res) {
  const key = req.headers['idempotency-key'];
  if (!key) {
    return res.status(400).json({ error: 'Idempotency-Key header is required' });
  }

  const userId = req.user.id;
  const reqHash = hashRequest(req.body);

  // STEP 1: Try to CLAIM the key. We do not look first — we just try to insert.
  // If a concurrent duplicate already claimed it, the PRIMARY KEY constraint
  // fires and DO NOTHING makes rowCount = 0. The database is the referee,
  // and it is the only component that can see both requests at once.
  const claim = await db.query(
    `INSERT INTO idempotency_keys
       (user_id, idempotency_key, request_hash, state, expires_at)
     VALUES ($1, $2, $3, $4, now() + interval '24 hours')
     ON CONFLICT (user_id, idempotency_key) DO NOTHING
     RETURNING *`,
    [userId, key, reqHash, IN_PROGRESS],
  );

  if (claim.rowCount === 0) {
    // We LOST the race (or this is a normal retry of a finished request).
    // Someone else owns this key. Go read what they did.
    return replayOrConflict(res, userId, key, reqHash);
  }

  // STEP 2: We WON the claim. We are the one and only processor of this key.
  try {
    const charge = await stripe.charges.create({
      amount: req.body.amountPence,
      source: req.body.cardToken,
      // Belt and braces: pass the key downstream too. Stripe dedupes as well.
      idempotency_key: key,
    });

    const responseBody = { paymentId: charge.id, status: 'succeeded' };

    // STEP 3: Persist the RESPONSE against the key. Every future retry
    // of this key replays exactly this, byte for byte.
    await db.query(
      `UPDATE idempotency_keys
          SET state = $1, response_code = $2, response_body = $3
        WHERE user_id = $4 AND idempotency_key = $5`,
      [COMPLETED, 201, responseBody, userId, key],
    );

    return res.status(201).json(responseBody);
  } catch (err) {
    // Release the claim so a genuine retry can attempt the work again.
    // If we left it IN_PROGRESS forever, the client could never recover.
    await db.query(
      'DELETE FROM idempotency_keys WHERE user_id = $1 AND idempotency_key = $2',
      [userId, key],
    );
    throw err;
  }
}

async function replayOrConflict(res, userId, key, reqHash) {
  const { rows } = await db.query(
    'SELECT * FROM idempotency_keys WHERE user_id = $1 AND idempotency_key = $2',
    [userId, key],
  );
  const record = rows[0];

  // Same key, different body → the client has a bug. Fail loudly.
  if (record.request_hash !== reqHash) {
    return res.status(422).json({
      error: 'This Idempotency-Key was already used with a different request body',
    });
  }

  if (record.state === COMPLETED) {
    // The happy path of a retry: replay the original response verbatim.
    return res.status(record.response_code).json(record.response_body);
  }

  // Still IN_PROGRESS — a concurrent duplicate is being handled right now.
  // Tell the client to come back shortly rather than risk double-processing.
  return res
    .status(409)
    .set('Retry-After', '1')
    .json({ error: 'A request with this Idempotency-Key is currently in progress' });
}
```

Three details worth burning in:

- **Claim before you work, not after.** The `INSERT` happens *before* the card is charged. That's the ticket on the rail before the steak hits the pan.
- **Store the response, not just the key.** A retry must get the *same answer*, not a bare `200 OK`. The client needs that `paymentId`.
- **Expiry.** 24 hours is the industry norm (it's Stripe's). Long enough that any sane retry window is covered; short enough that the table doesn't grow forever. Reap it with a nightly job: `DELETE FROM idempotency_keys WHERE expires_at < now()`.

### 5. Making operations naturally idempotent — usually better

Idempotency keys are a general-purpose bolt-on. But often you can **restructure the operation so duplicates are harmless by construction** — no extra table, no extra lookup, no expiry job. When you can, do this instead.

**a) Conditional / compare-and-swap updates instead of deltas**

```javascript
// BAD — a delta. Run twice, lose £100.
await db.query('UPDATE accounts SET balance = balance - 50 WHERE id = $1', [id]);

// GOOD — CAS. The version acts as a guard. The second run matches 0 rows.
const result = await db.query(
  `UPDATE accounts SET balance = $1, version = version + 1
    WHERE id = $2 AND version = $3`,
  [newBalance, id, expectedVersion],
);
if (result.rowCount === 0) {
  // Either someone else won, or this is our own duplicate. Either way: no harm.
}
```

**b) A natural unique key — let the database reject the duplicate**

The cheapest idempotency in existence. If the data has a natural identity, put a `UNIQUE` on it.

```sql
CREATE TABLE payments (
  id         BIGSERIAL PRIMARY KEY,
  order_id   BIGINT NOT NULL,
  amount     INT    NOT NULL,
  UNIQUE (order_id)          -- one payment per order, enforced by the DB
);
```

```javascript
// The DB does the deduping for you. Free, atomic, no race.
const { rowCount } = await db.query(
  `INSERT INTO payments (order_id, amount) VALUES ($1, $2)
   ON CONFLICT (order_id) DO NOTHING`,
  [orderId, amountPence],
);
if (rowCount === 0) {
  // Duplicate. The payment already exists. This is SUCCESS, not an error.
}
```

**c) Let the CLIENT supply the ID** — the most elegant option available

Instead of `POST /orders` (server generates the ID → every call creates a new order), do:

```
PUT /orders/9f2c1a6e-3b41-4a7d-8c5e-1f0b2d3e4a5b
```

The client generates the UUID. A retry hits **the same resource path**, so it's a replace, not a create. You've converted a non-idempotent `POST` into a naturally idempotent `PUT` — and you needed zero extra infrastructure to do it.

```javascript
app.put('/orders/:orderId', async (req, res) => {
  const { rows } = await db.query(
    `INSERT INTO orders (id, user_id, items, total)
     VALUES ($1, $2, $3, $4)
     ON CONFLICT (id) DO NOTHING
     RETURNING *`,
    [req.params.orderId, req.user.id, req.body.items, req.body.total],
  );

  if (rows.length === 0) {
    const existing = await db.query('SELECT * FROM orders WHERE id = $1', [req.params.orderId]);
    return res.status(200).json(existing.rows[0]);   // retry → same order, no dupe
  }
  return res.status(201).json(rows[0]);
});
```

**d) Deduplicate on message ID in a queue consumer**

Recall from [67 — Message Queues](./67-message-queues.md) that every broker is at-least-once. Your consumer *will* be handed the same message twice. Dedupe on the message's own ID:

```javascript
async function handleMessage(msg) {
  // Claim the message ID first. If we've seen it, we're done.
  const claim = await db.query(
    `INSERT INTO processed_messages (message_id, processed_at)
     VALUES ($1, now())
     ON CONFLICT (message_id) DO NOTHING`,
    [msg.id],
  );
  if (claim.rowCount === 0) {
    return ack(msg);   // Already processed. Ack and move on. Not an error.
  }

  await doTheActualWork(msg.body);
  await ack(msg);
}
```

**e) State machines — make the second attempt a legal no-op**

Model the entity's lifecycle explicitly and only allow legal transitions. A second attempt then simply finds the state already correct.

```javascript
const OrderState = Object.freeze({
  PENDING: 'PENDING',
  PAID: 'PAID',
  SHIPPED: 'SHIPPED',
  CANCELLED: 'CANCELLED',
});

async function markPaid(orderId) {
  // Only PENDING → PAID is legal. The WHERE clause is the guard.
  const { rowCount } = await db.query(
    `UPDATE orders SET state = $1 WHERE id = $2 AND state = $3`,
    [OrderState.PAID, orderId, OrderState.PENDING],
  );

  if (rowCount === 0) {
    const { rows } = await db.query('SELECT state FROM orders WHERE id = $1', [orderId]);
    if (rows[0]?.state === OrderState.PAID) return { alreadyPaid: true };  // idempotent no-op
    throw new Error(`Illegal transition to PAID from ${rows[0]?.state}`);
  }
  return { alreadyPaid: false };
}
```

### 6. Where this bites you in the real world

- **Payment APIs.** The canonical case. Double-charging is the most expensive bug class in fintech.
- **Message queue consumers.** At-least-once means duplicates are *guaranteed*, not merely possible ([67 — Message Queues](./67-message-queues.md)). A consumer without dedupe is a bug waiting for load.
- **Webhook receivers.** Stripe and GitHub will absolutely send you the same webhook twice — if your endpoint is slow, or 5xxs, or their delivery record is lost, they retry. **Dedupe on the event ID** (`evt_...` / GitHub's `X-GitHub-Delivery`). Never on payload content. Never on timestamp.
- **SAGA compensations.** In [77 — Distributed Transactions](./77-distributed-transactions.md), a SAGA is a chain of local transactions with compensating undo steps. The orchestrator retries every step on failure — so **every forward step and every compensation must be idempotent**, or your rollback corrupts more than the failure did.
- **Cron jobs.** The scheduler fires, the pod is OOM-killed mid-run, Kubernetes restarts it. Your "send monthly invoices" job just ran twice.
- **Mobile clients.** Lifts, tunnels, the Tube. Flaky networks are the *normal* condition, not the exception.

### 7. Retries and idempotency are a package deal

Neither half works alone.

- **Retries without idempotency** = duplicate side effects. You built the horror story.
- **Idempotency without retries** = a safety net nobody jumps into. You paid the cost and got no benefit.

The client-side half needs two things:

**Exponential backoff** — double the wait each attempt (1s, 2s, 4s, 8s...) so you don't hammer a struggling server.

**Jitter** — randomise the wait. Without it, 10,000 clients that all failed at the same instant will all retry at the *same instant*, and you've built a synchronised sledgehammer. This is the **thundering herd**, and it is how a brief blip becomes a full outage.

```javascript
async function retryWithBackoff(fn, { maxAttempts = 5, baseMs = 200, capMs = 20_000 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= maxAttempts - 1 || !isRetryable(err)) throw err;

      // Exponential, capped, and FULLY jittered.
      // Full jitter (random across the whole window) beats "backoff + a little noise"
      // at spreading a herd — AWS published the maths on this.
      const window = Math.min(capMs, baseMs * 2 ** attempt);
      await new Promise((r) => setTimeout(r, Math.random() * window));
    }
  }
}

function isRetryable(err) {
  // Retry: network errors, 429, and 5xx. NOT 4xx — a bad request stays bad.
  if (!err.status) return true;                    // network / timeout
  return err.status === 429 || err.status >= 500;
}
```

And retries need a **budget**. Unbounded retries against a dying service turn a partial outage into a total one — which is exactly what [73 — Circuit Breaker](./73-circuit-breaker.md) exists to prevent. Cap your attempts, then trip the breaker.

---

## Visual / Diagram description

### Diagram 1: The ambiguity that makes idempotency mandatory

```
   CLIENT                    NETWORK                     SERVER
     │                          │                           │
     │──── POST /payments ──────┼──────────────────────────▶│
     │                          │                           │  charges card
     │                          │                           │  writes DB
     │                          │◀──── 201 Created ─────────│
     │                          │                           │
     │                          X  ◀── RESPONSE LOST HERE   │
     │                          │                           │
     │   ⏱  TIMEOUT              │                           │
     │                          │                           │
     │   The client now knows EXACTLY NOTHING. Three worlds  │
     │   are consistent with the silence it observed:        │
     │                                                       │
     │     (a) server never got it        → must retry       │
     │     (b) server did it, reply lost  → must NOT retry   │
     │     (c) server is still working    → must NOT retry   │
     │                                                       │
     │   ┌───────────────────────────────────────────────┐  │
     │   │  These are INDISTINGUISHABLE from outside.    │  │
     │   │  No header, no timeout value, no protocol     │  │
     │   │  trick can tell them apart. Ever.             │  │
     │   └───────────────────────────────────────────────┘  │
     │                                                       │
     │   So: retry (risk a duplicate) or give up (risk a    │
     │   lost payment). Idempotency makes retry FREE.        │
```

### Diagram 2: The idempotency-key flow, including the concurrent race

```
  Client A (original)          Client A (retry, 3s later)
        │                              │
        │  Idempotency-Key: K          │  Idempotency-Key: K   (SAME key)
        ▼                              ▼
   ┌─────────────────────────────────────────────┐
   │              LOAD BALANCER                  │
   └───────┬─────────────────────────┬───────────┘
           │                         │
           ▼                         ▼
   ┌──────────────┐          ┌──────────────┐
   │  API Server 1│          │  API Server 2│   ← different boxes!
   └───────┬──────┘          └───────┬──────┘      they share NO memory
           │                         │
           │ INSERT K                │ INSERT K
           │ ON CONFLICT DO NOTHING  │ ON CONFLICT DO NOTHING
           ▼                         ▼
   ┌─────────────────────────────────────────────┐
   │        IDEMPOTENCY STORE (Postgres)         │
   │                                             │
   │   PRIMARY KEY (user_id, idempotency_key)    │  ◀── THE REFEREE.
   │                                             │       The only component
   │   Server 1 → rowCount = 1  → WON the claim  │       that sees BOTH
   │   Server 2 → rowCount = 0  → LOST the claim │       requests at once.
   └─────────────────────────────────────────────┘
           │                         │
           │ WON                     │ LOST
           ▼                         ▼
   ┌──────────────┐          ┌────────────────────────┐
   │ Charge card  │          │ Read the existing row: │
   │ (ONCE)       │          │                        │
   │      │       │          │  COMPLETED  → replay   │
   │      ▼       │          │               stored   │
   │ UPDATE row:  │          │               response │
   │  state=      │          │                        │
   │  COMPLETED   │          │  IN_PROGRESS → 409     │
   │  response=…  │          │                Conflict│
   └──────┬───────┘          └───────────┬────────────┘
          │                              │
          ▼                              ▼
     201 + paymentId              200 + SAME paymentId
     (the card was charged exactly once, either way)
```

**What to take from Diagram 2:** the two servers cannot coordinate with each other. They have separate memory, separate processes, possibly separate continents. The *only* place that can serialise them is the shared database, and the *only* mechanism that does it correctly is a unique constraint on the insert. That's why "check then insert" fails and "insert with `ON CONFLICT`" works — the second one pushes the decision into the one component capable of making it.

---

## Real world examples

### 1. Stripe — the reference implementation

Stripe accepts an `Idempotency-Key` header on all `POST` endpoints. You send a UUID; if you replay the same key, Stripe replays the original response **verbatim** — same status code, same body, same `charge` object — without re-executing the charge. Keys are retained for **24 hours**. Their official SDKs auto-generate a key for you and reuse it across the SDK's internal retries. Stripe also returns an error if you reuse a key with a *different* request body, which is precisely the `422` case in the code above.

This is the design the entire payments industry copied. When an interviewer asks "how would you prevent double charges?", the answer "an idempotency key, the way Stripe does it" is both correct and immediately legible.

### 2. AWS — idempotency baked into the primitives

- **EC2's `RunInstances`** takes a `ClientToken` parameter. Retry with the same token and you get the same instances back, not new ones. Same idea, older name.
- **SQS FIFO queues** take a `MessageDeduplicationId` and suppress duplicates within a **5-minute** window. Useful, but notice the honest limitation — 5 minutes is not 24 hours, and it only helps on the *producer* side. Your consumer still needs its own dedupe.
- **DynamoDB conditional writes** (`ConditionExpression: attribute_not_exists(pk)`) are the NoSQL version of `INSERT ... ON CONFLICT DO NOTHING` — the database is the referee, exactly as in Diagram 2.

### 3. GitHub webhooks — dedupe on the delivery ID

Every GitHub webhook carries an `X-GitHub-Delivery` header containing a unique GUID. GitHub explicitly retries deliveries that fail or time out, so **the same GUID can arrive at your endpoint more than once.** The correct receiver stores the GUID, and drops any delivery whose GUID it has already seen. Stripe does the same with its `evt_...` event IDs.

The wrong implementation — the one you will find in production somewhere near you — hashes the payload body, or checks a timestamp, or just assumes it won't happen. It will happen.

---

## Trade-offs

| Approach | Pros | Cons |
|---|---|---|
| **Idempotency key (header + store)** | Works for any operation; explicit; client controls scope; response replay | Extra table, extra write on every request; TTL/reaping to manage; client must cooperate |
| **Natural unique constraint** | Free, atomic, race-proof, zero extra infra | Needs a genuinely unique business key; can't replay the original response body |
| **Client-supplied ID (`PUT`)** | Elegant; no extra store; naturally RESTful | Requires an API redesign; clients must generate UUIDs correctly |
| **CAS / conditional update** | No extra storage; great for state transitions | Only fits updates, not creates; needs a version column |
| **State machine guard** | Self-documenting; blocks illegal transitions too | Only helps entities with a lifecycle |
| **Do nothing, hope** | Zero work today | Duplicate charges, duplicate orders, a postmortem, a customer refund |

| Cost you pay | What you actually give up |
|---|---|
| An extra DB round-trip per request | ~1-3 ms of latency. Almost always worth it |
| The idempotency table's storage | ~200 bytes/key × 10M requests/day × 1 day TTL ≈ **2 GB**. Trivial |
| Client complexity | Clients must generate and *persist* the key across retries |
| The `409 IN_PROGRESS` case | Some concurrent duplicates get a retryable error, not an answer |

**Rule of thumb:** Make it **naturally idempotent** if you can — a unique constraint or a client-supplied ID beats a bolted-on key every time, because there's nothing to expire and nothing to get wrong. Reach for an **idempotency key** when the operation has an external side effect you cannot take back (charging a card, calling a third-party API, sending an email) and you need to replay the *response*, not just suppress the write. And **any** endpoint that moves money gets a key. No exceptions, no debate.

---

## Common interview questions on this topic

### Q1: "The user taps Pay, the request times out, and the app retries. How do you stop a double charge?"
**Hint:** Idempotency key. Client generates a UUID *per logical payment* (outside the retry loop) and sends it on every attempt. The server does an `INSERT ... ON CONFLICT DO NOTHING` on `(user_id, key)` *before* charging — that atomic claim is what makes it race-safe. Winner charges and stores the response; loser replays the stored response or gets a `409` if still in flight. Then say "this is how Stripe's API works" and watch them nod.

### Q2: "Why can't you just check if the key exists, and insert it if not?"
**Hint:** That's a check-then-act race. Two duplicates land on two servers 2 ms apart, both `SELECT` and find nothing, both proceed, both charge. The servers share no memory — they *cannot* coordinate. Only the database sees both, so the uniqueness decision must happen inside a single atomic DB operation: a `UNIQUE` constraint on the insert. This is the answer that separates senior from mid-level.

### Q3: "Is `DELETE` idempotent? What about `POST` and `PUT`?"
**Hint:** `GET`/`HEAD`/`PUT`/`DELETE` are idempotent by spec; `POST` is not; `PATCH` only if the body is absolute rather than a delta. `DELETE` is idempotent because deleting an already-deleted thing leaves it deleted — return `204` either way. Key nuance to volunteer: *idempotent means same effect, not same response* — and the spec's promise only holds if you actually implemented it that way.

### Q4: "Your queue gives at-least-once delivery. How do you get exactly-once processing?"
**Hint:** You don't get exactly-once *delivery* — it's a myth ([67](./67-message-queues.md)). You get exactly-once *effect*: **at-least-once delivery + idempotent consumer = effectively-once processing.** Dedupe on the message ID with an `INSERT ... ON CONFLICT DO NOTHING` into a `processed_messages` table before doing the work, or make the work itself naturally idempotent (upsert instead of insert, set instead of increment).

### Q5: "How long do you keep idempotency keys, and how do you scope them?"
**Hint:** 24 hours is the industry norm (Stripe's choice) — longer than any sane client retry window, short enough that the table stays small. Scope keys to `(user_id, key)`, not `key` alone: otherwise one tenant can collide with — or deliberately guess — another tenant's key and get *their* response replayed back. That's a data-leak, not just a correctness bug. Reap expired rows with a nightly `DELETE ... WHERE expires_at < now()`.

---

## Practice exercise

### Build a double-charge-proof payment endpoint

Build a small Express service with an in-memory (or SQLite/Postgres) idempotency store, and then **prove** it works under concurrency.

**Part A — the endpoint (~20 min)**

Write `POST /payments`, taking an `Idempotency-Key` header and `{ userId, amountPence }`. It must:
1. Reject requests with no key (`400`).
2. Atomically claim the key with `INSERT ... ON CONFLICT DO NOTHING` on `(user_id, key)` — **before** doing any work.
3. On winning the claim: simulate the charge with a deliberate `await sleep(500)` (this widens the race window so you can actually observe it), then store the response and return `201`.
4. On losing the claim: replay the stored response if `COMPLETED`, or return `409` if `IN_PROGRESS`.
5. Return `422` if the same key arrives with a different `amountPence`.

**Part B — attack it (~15 min)**

Write a script that fires **10 concurrent** requests with the **same** idempotency key using `Promise.all`. Assert:
- Exactly **one** returns `201`.
- Your `chargeCount` counter equals exactly **1**.
- The others returned `200` (replayed) or `409` (in-flight).

**Part C — prove the bug is real (~5 min)**

Now swap your atomic claim for the naive `SELECT`-then-`INSERT` version, rerun Part B, and **watch `chargeCount` climb past 1.** Write down the number you got.

**What to produce:** the working service, the concurrency test, and one sentence explaining why the naive version's charge count was greater than 1. If you can explain that sentence to someone else, you understand idempotency.

---

## Quick reference cheat sheet

- **Idempotent** = doing it N times has the same *effect* as doing it once. Same effect, **not** necessarily the same response.
- **The unfixable truth:** on a timeout, the client cannot distinguish "never arrived" from "processed, reply lost" from "still running." Retry (risk duplicate) or give up (risk loss). There is no third option.
- **The equation:** at-least-once delivery + idempotent processing = **effectively-once processing**. This is what people actually mean by "exactly-once."
- **Idempotent by spec:** `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`. **Not:** `POST`, `PATCH`. And the spec's promise only holds if *you* implemented it that way.
- **Idempotency key:** client generates a UUID **per logical operation** (outside the retry loop!), sends it on every attempt as `Idempotency-Key`.
- **Insert first, work second.** Claim the key *before* the side effect — the ticket goes on the rail before the steak hits the pan.
- **The race is settled by the DATABASE**, never by application code. `INSERT ... ON CONFLICT DO NOTHING` on a `UNIQUE`/`PRIMARY KEY`. Check-then-insert is a race and it *will* fire at scale.
- **Store the response**, not just the key — a retry must get the same `paymentId` back, not a bare `200`.
- **Scope the key to the user:** `PRIMARY KEY (user_id, idempotency_key)`. Global keys are a cross-tenant data leak.
- **Expire keys after ~24h** (Stripe's default). Reap with a nightly job.
- **Prefer natural idempotency:** CAS updates (`SET balance = 100 WHERE version = 5`), unique business keys (`UNIQUE(order_id)`), client-supplied IDs (`PUT /orders/{uuid}`), state-machine guards (`WHERE state = 'PENDING'`).
- **Deltas are dangerous, absolutes are safe.** `balance = balance - 50` is a bug; `balance = 100` is not.
- **Webhooks:** always dedupe on the provider's event ID (`evt_…`, `X-GitHub-Delivery`). Never on payload content.
- **Retries + idempotency are a package deal:** exponential backoff **with jitter** on the client, idempotency on the server. Neither is safe without the other.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [77 — Distributed Transactions](./77-distributed-transactions.md) — SAGAs retry every step, so every step and every compensation must be idempotent |
| **Next** | [107 — HLD: Payment System](./107-hld-payment-system.md) — where idempotency stops being theory and becomes the core of the design |
| **Related** | [67 — Message Queues](./67-message-queues.md) — at-least-once delivery is why your consumer WILL see duplicates; exactly-once delivery is a myth |
| **Related** | [69 — API Design: REST, GraphQL, gRPC](./69-api-design-rest-graphql-grpc.md) — which HTTP methods promise idempotency, and how to design endpoints that keep the promise |
| **Related** | [73 — Circuit Breaker](./73-circuit-breaker.md) — retries need a budget; unbounded retries against a dying service turn a partial outage into a total one |
