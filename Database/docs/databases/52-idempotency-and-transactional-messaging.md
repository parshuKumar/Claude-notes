# 52 — Idempotency and Transactional Messaging
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

You hand a cashier your card. The terminal freezes. **Did it charge you or not?**

You genuinely cannot tell. Neither can the cashier. The safe move is to try again — but if the first one *did* go through, you've now paid twice.

So the shop changes how it works. Every purchase gets a **slip with a unique number on it**, written before anything happens. The cashier's rule: *"before charging, check whether a slip with this number is already in the box. If it is, don't charge — just tell them the outcome that's written on it."*

Now retrying is **free**. You can retry ten times. The first attempt writes the slip and charges; the other nine find the slip and report what happened. **The uncertainty didn't go away — it became harmless.**

And the second half. The shop also has to tell the warehouse to ship your item. The naive move is: charge the card, then phone the warehouse. But if the phone line is down after the charge, you've been charged and nothing ships.

So instead: **when they write the slip, they write a "to-do card" into the same box, in the same motion.** A runner comes by every few seconds, takes the to-do cards, phones the warehouse, and marks them done. If the runner is asleep, the cards pile up — nothing is lost. If the runner phones twice, the warehouse checks its own slip box and ignores the duplicate.

That's the whole topic: ★ **an idempotency key makes retries harmless, and an outbox makes "did my message get sent?" a local transactional question instead of a distributed one.**

---

## Where this fits in the big picture

```
   43/44 anomalies & isolation  ·  48 deadlocks (40P01)  ·  50 SSI (40001)
   51 2PC — ★ why you're not using distributed transactions
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 52 IDEMPOTENCY & TRANSACTIONAL MESSAGING     │
        │ ★ what makes ALL of Phase 5's retries safe   │
        └────────────────────┬─────────────────────────┘
                             ▼
                  ★ PHASE 5 COMPLETE
              → 53–61 denormalisation & scale
              → 76 CDC · case studies 03, 09, 12
```

Every topic in this phase told you to **retry**: `40001` from REPEATABLE READ, `40001` from SSI, `40P01` from deadlocks, a lost connection, a timeout. **This topic is the precondition that makes all of that safe.** Without it, "just retry" is how you double-charge customers.

---

## What is this?

**Idempotency** — an operation is idempotent if performing it N times has the same effect as performing it once.

```
 ✓ IDEMPOTENT      UPDATE orders SET status='paid' WHERE id=7;
                   DELETE FROM sessions WHERE id=$1;
                   INSERT … ON CONFLICT DO NOTHING;
                   PUT /orders/7  {status: "paid"}

 ✗ NOT IDEMPOTENT  UPDATE accounts SET balance = balance - 500 …;
                   INSERT INTO ledger_entries …;
                   POST /payments {amount: 500}
                   ⇒ ★ every "add", "subtract", "append" and "send"
```

**Transactional messaging** — the problem of making a database write and a message send atomic, when they are two different systems.

```
 ★ THE DUAL-WRITE PROBLEM — you cannot escape it by reordering:

   ① db.commit(); broker.publish();
      ⇒ crash between them ⇒ ★ order exists, nobody was told

   ② broker.publish(); db.commit();
      ⇒ crash between them ⇒ ★ everyone was told about an order
        that doesn't exist  — WORSE

   ③ 2PC across both
      ⇒ ★ blocking, availability multiplies, Kafka's XA support is
        poor (Topic 51)

 ⇒ ★ THE ANSWER: DON'T WRITE TO TWO SYSTEMS.
   Write the message INTO YOUR DATABASE, in the SAME transaction.
   A separate process delivers it later.
   ⇒ THE TRANSACTIONAL OUTBOX.
```

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE YOU CANNOT AVOID THE UNCERTAINTY. YOU CAN ONLY MAKE IT
   HARMLESS.

 EVERY ONE OF THESE LEAVES YOU NOT KNOWING WHAT HAPPENED:
   • a timeout on a POST                (did it commit?)
   • a connection reset mid-COMMIT      (★ did it commit?)
   • a load-balancer 502                (did it reach the server?)
   • a mobile client retrying on flaky 4G
   • a webhook provider's at-least-once delivery
   • a Kafka consumer rebalance replaying an offset
   • ★ YOUR OWN retry loop for 40001 / 40P01

 ⇒ ★ AND ONE OF THEM IS UNSOLVABLE IN PRINCIPLE:
   if the connection drops while COMMIT is in flight, the
   TRANSACTION MAY HAVE COMMITTED. The server did the work; the
   acknowledgement was lost. No amount of client-side cleverness
   recovers this.
 ⇒ THE ONLY FIX IS FOR THE OPERATION TO BE SAFE TO REPEAT.
```

```
 ★ THE COST OF GETTING IT WRONG, IN ONE LINE:
   duplicate charges are the single most reputation-damaging class
   of bug a payments system can ship, and they are caused by a
   missing UNIQUE constraint.
```

---

## The physical reality

### Why the database is the only place the key can live

```
 ★ AN IDEMPOTENCY CHECK MUST BE ATOMIC WITH THE WORK IT GUARDS.

 ✗ IN REDIS / MEMCACHED
     if (await redis.get(key)) return cached;
     await doWork();                    ← ★ crash HERE
     await redis.set(key, result);
   ⇒ the work happened, the key was never recorded
   ⇒ the retry does it AGAIN
   ⇒ ★ and Redis is a SEPARATE FAILURE DOMAIN — you have
     re-created the dual-write problem inside your solution.

 ✗ IN APPLICATION MEMORY
   ⇒ gone on restart; not shared across instances

 ✓ ★ IN THE SAME DATABASE, IN THE SAME TRANSACTION
     BEGIN;
       INSERT INTO idempotency_keys (key) VALUES ($1);  -- ★ UNIQUE
       … the work …
     COMMIT;
   ⇒ the key and the work commit together or not at all
   ⇒ ★ ENFORCED BY A UNIQUE INDEX, WHICH IS A B-TREE PAGE-LEVEL
     GUARANTEE (Topic 11) — not application logic that a code path
     can forget.
```

### What actually happens on a concurrent duplicate

```
 TWO REQUESTS WITH THE SAME KEY ARRIVE 2 ms APART.

 t1  A: BEGIN; INSERT INTO idempotency_keys (key) VALUES ('k1');
        ⇒ B-tree: find the leaf page for 'k1', insert the entry
        ⇒ ★ the index entry is written but A is UNCOMMITTED

 t2  B: BEGIN; INSERT INTO idempotency_keys (key) VALUES ('k1');
        ⇒ B-tree: finds A's uncommitted entry for 'k1'
        ⇒ ★ B CANNOT DECIDE YET. If A commits ⇒ conflict.
          If A aborts ⇒ no conflict.
        ⇒ ★ B BLOCKS on A's transaction ID (XactLockTableWait)

 t3  A: … does the work … COMMIT;

 t4  B: wakes up, re-checks
        ⇒ A committed ⇒ ★ ERROR: duplicate key value violates
          unique constraint "idempotency_keys_pkey"  (SQLSTATE 23505)

 ⇒ ★ B BLOCKED FOR THE FULL DURATION OF A'S TRANSACTION.
   THIS IS THE SUBTLETY THAT SURPRISES PEOPLE: a duplicate request
   does not fail fast — it waits for the original to finish, then
   fails. If A takes 3 seconds (calling a payment gateway), B waits
   3 seconds.
 ⇒ AND THAT IS EXACTLY WHAT YOU WANT: when B wakes up, the answer
   EXISTS and B can return it. Failing fast would mean returning
   "in progress" with nothing useful.
 ⇒ ★ BUT IT MEANS: NEVER HOLD AN IDEMPOTENCY KEY OPEN ACROSS A SLOW
   EXTERNAL CALL. (See "the three-state pattern" below.)
```

### The outbox — what it guarantees and what it does not

```
 BEGIN;
   INSERT INTO orders …;                 ← your state
   INSERT INTO outbox (payload, key) …;  ← ★ the intent to publish
 COMMIT;                                 ← ★ ONE fsync, both or neither

 ★ WHAT THIS GUARANTEES:
   if the order exists, the event WILL eventually be published.
   if the order doesn't exist, the event will NEVER be published.
   ⇒ NO DUAL WRITE. The atomicity is LOCAL.

 ★ WHAT IT DOES NOT GUARANTEE:
   ① EXACTLY-ONCE DELIVERY. The relay may publish, then crash
      before marking the row published ⇒ it publishes again.
      ⇒ ★ AT-LEAST-ONCE IS THE BEST ANY SYSTEM CAN DO across a
        network. "Exactly-once" always means "at-least-once
        delivery + idempotent consumer."
   ② ORDERING across aggregates. Two orders' events may arrive
      out of order.
      ⇒ if you need per-aggregate ordering, partition the broker
        by aggregate_id AND have the relay publish in id order
        per aggregate.
   ③ LOW LATENCY. The relay polls. Typical lag 50–500 ms.
      ⇒ if that's too slow, use logical decoding (Topic 76) —
        the same guarantee, sub-10 ms, no polling, no outbox table.

 ★ THE OUTBOX TABLE IS A QUEUE, AND QUEUES BLOAT (Topic 47):
   every row is INSERTed then UPDATEd then DELETEd.
   ⇒ high churn ⇒ dead tuples ⇒ ★ aggressive per-table autovacuum
     is MANDATORY, not optional.
   ⇒ or: partition by day and DROP.
```

### Why `INSERT … ON CONFLICT DO NOTHING` is not always enough

```
 ★ THE SUBTLE BUG THAT PASSES CODE REVIEW.

 INSERT INTO processed_events (key) VALUES ($1) ON CONFLICT DO NOTHING;
 -- rowCount === 0 ⇒ "already processed, skip"

 ⇒ ★ THIS IS CORRECT ONLY IF THE FIRST ATTEMPT'S WORK IS IN THE
   SAME TRANSACTION AS THE KEY INSERT.

 ✗ IF NOT — e.g. the key is inserted, then the work happens after
   COMMIT — you get:
     attempt 1: key inserted, COMMIT, ★ then the process crashes
                before doing the work
     attempt 2: ON CONFLICT DO NOTHING ⇒ rowCount 0 ⇒ "already
                done" ⇒ ★ SKIPPED. THE WORK NEVER HAPPENS.
   ⇒ ★ SILENT DATA LOSS, and it looks like correct dedupe code.

 ⇒ ★ THE RULE: THE KEY AND THE WORK MUST COMMIT TOGETHER.
   If they cannot (because the work involves an external call),
   you need the THREE-STATE pattern, not a boolean.
```

### The three-state pattern — for work that includes an external call

```
 ★ WHEN THE OPERATION CALLS A PAYMENT GATEWAY, YOU CANNOT HOLD A
   TRANSACTION OPEN ACROSS IT (Topics 45, 49). So you need three
   states, not "present/absent":

   'in_progress'  — reserved; the work may or may not have happened
   'succeeded'    — done; here is the stored response
   'failed'       — done; here is the stored error

 THE FLOW:
 ① TXN 1: INSERT … (key, status='in_progress', request_hash)
          ON CONFLICT DO NOTHING; COMMIT;
    rowCount 0 ⇒ someone else owns it:
      • status='succeeded'/'failed' ⇒ ★ RETURN THE STORED RESPONSE
      • status='in_progress'        ⇒ ★ RETURN 409 "in progress,
        retry shortly" — do NOT do the work
 ② the external call, ★ outside any transaction, with the SAME
    idempotency key passed to the provider
 ③ TXN 2: apply the local effects AND
          UPDATE idempotency_keys SET status='succeeded',
                 response=$1 WHERE key=$2;
          COMMIT;   ← ★ atomic together

 ★ THE CRASH BETWEEN ② AND ③ IS THE HARD CASE:
   the gateway charged; we have no record.
   ⇒ ★ THIS IS WHY YOU PASS YOUR KEY TO THE PROVIDER. On retry,
     Stripe/Razorpay return the ORIGINAL charge rather than making
     a new one. Their idempotency and yours must use the SAME key.
   ⇒ plus a sweeper: any 'in_progress' older than N minutes gets
     RECONCILED against the provider's API. ★ Not guessed.

 ★ AND THE REQUEST HASH: store a hash of the request body. If the
   same key arrives with a DIFFERENT body, that is a client bug —
   return 422, don't silently return the old response.
```

---

## How it works — step by step

### The complete idempotent endpoint

```sql
CREATE TABLE idempotency_keys (
  key            text        PRIMARY KEY,
  request_hash   text        NOT NULL,      -- ★ detects key reuse
  status         text        NOT NULL
                   CHECK (status IN ('in_progress','succeeded','failed')),
  response_code  int,
  response_body  jsonb,
  created_at     timestamptz NOT NULL DEFAULT now(),
  completed_at   timestamptz
);
CREATE INDEX idx_idem_stale ON idempotency_keys (created_at)
  WHERE status = 'in_progress';            -- ★ for the sweeper
```

```js
async function createPayment(req, res) {
  const key = req.get('Idempotency-Key');
  if (!key) return res.status(400).json({ error: 'IDEMPOTENCY_KEY_REQUIRED' });
  const hash = sha256(canonicalJson(req.body));

  // ── ① claim the key ────────────────────────────────────────────
  const claim = await withTransaction(t => t.query(
    `INSERT INTO idempotency_keys (key, request_hash, status)
     VALUES ($1, $2, 'in_progress')
     ON CONFLICT (key) DO NOTHING
     RETURNING key`, [key, hash]));

  if (claim.rowCount === 0) {
    const { rows: [prev] } = await pool.query(
      'SELECT * FROM idempotency_keys WHERE key = $1', [key]);

    // ★ same key, different body ⇒ a client bug. Say so.
    if (prev.request_hash !== hash)
      return res.status(422).json({ error: 'IDEMPOTENCY_KEY_REUSED' });

    if (prev.status === 'in_progress')
      return res.status(409).json({ error: 'IN_PROGRESS', retryAfter: 2 });

    // ★ replay the stored response, byte for byte
    return res.status(prev.response_code).json(prev.response_body);
  }

  // ── ② the external call — ★ OUTSIDE any transaction ───────────
  let charge;
  try {
    charge = await gateway.charge({
      amountMinor: req.body.amount_minor,
      idempotencyKey: key,          // ★ THE SAME KEY. This is what
    });                             //   makes the crash-between case
  } catch (e) {                     //   recoverable.
    await pool.query(
      `UPDATE idempotency_keys
          SET status='failed', response_code=502,
              response_body=$1, completed_at=now()
        WHERE key=$2`, [{ error: e.code }, key]);
    return res.status(502).json({ error: e.code });
  }

  // ── ③ local effects + completion, atomically ──────────────────
  const body = await withTransaction(async (t) => {
    const { rows: [p] } = await t.query(
      `INSERT INTO payments (order_id, amount_minor, gateway_ref, status)
       VALUES ($1,$2,$3,'captured') RETURNING id, status`,
      [req.body.order_id, req.body.amount_minor, charge.id]);

    await t.query(
      `INSERT INTO ledger_entries (account_id, amount_minor, kind, ref)
       VALUES ($1,$2,'payment',$3)`,
      [req.body.account_id, req.body.amount_minor, p.id]);

    // ★ the outbox write, same transaction
    await t.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type,
                           payload, idempotency_key)
       VALUES ('payment',$1,'PaymentCaptured',$2,$3)`,
      [p.id, { payment_id: p.id, order_id: req.body.order_id },
       `payment-captured:${p.id}`]);

    const out = { payment_id: p.id, status: p.status };
    await t.query(
      `UPDATE idempotency_keys SET status='succeeded', response_code=201,
              response_body=$1, completed_at=now() WHERE key=$2`,
      [out, key]);
    return out;
  });

  res.status(201).json(body);
}
```

```js
// ── ④ the sweeper — ★ the piece everyone forgets ────────────────
async function sweepStuckKeys() {
  const { rows } = await pool.query(
    `SELECT key FROM idempotency_keys
      WHERE status='in_progress' AND created_at < now() - interval '5 minutes'
      LIMIT 100`);
  for (const { key } of rows) {
    // ★ ASK THE PROVIDER. Do not guess.
    const charge = await gateway.lookupByIdempotencyKey(key);
    if (charge) await completeFromCharge(key, charge);
    else await pool.query(
      `UPDATE idempotency_keys SET status='failed', response_code=504,
              response_body='{"error":"TIMEOUT"}', completed_at=now()
        WHERE key=$1 AND status='in_progress'`, [key]);
  }
}
```

### Naturally idempotent SQL — prefer this over keys

```sql
-- ★ ① UPSERT — the most useful idempotent primitive
INSERT INTO user_preferences (user_id, theme, updated_at)
VALUES ($1,$2, now())
ON CONFLICT (user_id) DO UPDATE
   SET theme = EXCLUDED.theme, updated_at = now();

-- ★ ② UNIQUE on a natural business key — no separate key table
CREATE UNIQUE INDEX uq_ledger_ref ON ledger_entries (source, source_ref);
INSERT INTO ledger_entries (source, source_ref, …) VALUES ('stripe', $1, …)
ON CONFLICT (source, source_ref) DO NOTHING;
-- ⇒ ★ the WEBHOOK'S OWN ID is the idempotency key. Nothing extra.

-- ★ ③ ABSOLUTE, NOT RELATIVE
UPDATE orders SET status='shipped' WHERE id=$1;          -- ✓ idempotent
UPDATE counters SET n = n + 1 WHERE id=$1;               -- ✗ not
UPDATE counters SET n = $2 WHERE id=$1 AND version=$3;   -- ✓ (Topic 49)

-- ★ ④ CONDITIONAL STATE TRANSITIONS — idempotent AND race-safe
UPDATE orders SET status='shipped', shipped_at=now()
 WHERE id=$1 AND status='paid'
RETURNING id;
-- rowCount 0 ⇒ already shipped, or not payable. ★ Check which.

-- ★ ⑤ DELETE and SET are naturally idempotent; INSERT and += are not.
```

---

## Concept breakdown

```
IDEMPOTENCY — N times ≡ once
├── ✓ naturally: UPDATE …SET x=$1 · DELETE · UPSERT · PUT
├── ✗ never:     x = x + 1 · plain INSERT · POST · "send"
└── ★ THE KEY MUST LIVE IN THE SAME DATABASE, ENFORCED BY UNIQUE
     ⇒ Redis re-creates the dual-write problem inside your fix

★ THE UNSOLVABLE UNCERTAINTY
   a connection lost during COMMIT may have committed
   ⇒ no client-side cleverness recovers it
   ⇒ ★ the ONLY fix is that repeating is safe

★ WHAT HAPPENS ON A CONCURRENT DUPLICATE
   the second INSERT BLOCKS on the first transaction, then gets
   23505 ⇒ ★ it waits as long as the original takes
   ⇒ so NEVER hold a key open across a slow external call

★ THE THREE-STATE PATTERN (external calls)
   in_progress → succeeded / failed, ★ with the stored response
   ① claim (txn 1) ② call, outside a txn, ★ same key to the
   provider ③ effects + completion (txn 2) ④ ★ a SWEEPER that
   RECONCILES stuck keys against the provider — never guesses
   + ★ a request hash, so key reuse with a different body is a 422

THE DUAL-WRITE PROBLEM
   db-then-publish ⇒ lost event · publish-then-db ⇒ phantom event
   2PC ⇒ blocking (Topic 51)
   ⇒ ★ TRANSACTIONAL OUTBOX: write the event INTO the database

THE OUTBOX
├── ✓ guarantees: event exists ⟺ state exists. ★ Local atomicity.
├── ✗ does NOT give: exactly-once · cross-aggregate ordering ·
│    low latency (poll lag 50–500 ms)
├── relay: SELECT … FOR UPDATE SKIP LOCKED ⇒ N relays, no contention
├── ★ consumer MUST dedupe — at-least-once is the best possible
└── ★ IT IS A QUEUE ⇒ IT BLOATS ⇒ aggressive autovacuum or
     partition-and-drop (Topic 47)

"EXACTLY-ONCE" DOES NOT EXIST ON A NETWORK
   ★ it always means at-least-once delivery + an idempotent consumer

SAGA — when a sequence must be undone
   forward steps + ★ compensating actions, each idempotent
   ⇒ compensations RARELY restore the exact prior state
     (a refund is not an un-charge) — that is a business decision

★ THE THREE THINGS AN EVENTUALLY-CONSISTENT DESIGN NEEDS
   ① at-least-once delivery  ② consumer dedupe
   ③ ★ A RECONCILER + A LAG ALERT   ← the one teams omit
```

---

## Diagrams

**Diagram 1 — big picture: the dual-write problem and its only real answer**

```
 ✗ DB THEN PUBLISH
   BEGIN; INSERT orders; COMMIT;  ──✓
                                    ★ ⚡ CRASH
   broker.publish()               ──✗
   ⇒ the order exists. Nobody was ever told. ★ SILENT.

 ✗ PUBLISH THEN DB
   broker.publish()               ──✓
                                    ★ ⚡ CRASH
   BEGIN; INSERT orders; COMMIT;  ──✗
   ⇒ everyone acted on an order that does not exist. ★ WORSE.

 ✗ 2PC ACROSS BOTH
   ⇒ blocking, 0.999ⁿ availability, poor broker support (Topic 51)

 ✓ ★ TRANSACTIONAL OUTBOX — one system, one transaction
   ┌──────────────────────────────────────────────────────────────┐
   │  BEGIN;                                                       │
   │    INSERT INTO orders  (…);        ← your state               │
   │    INSERT INTO outbox  (…);        ← ★ the intent to publish  │
   │  COMMIT;                           ← ★ ONE fsync. Atomic.     │
   │                                                               │
   │        ┌─────────── outbox table ───────────┐                 │
   │        │ id │ payload │ published_at        │                 │
   │        │  1 │  {...}  │ 2026-08-18 09:14    │                 │
   │        │  2 │  {...}  │ ★ NULL              │ ← pending       │
   │        └──────────────┬─────────────────────┘                 │
   │                       │                                       │
   │      relay (separate process, ★ crash-safe)                   │
   │        SELECT … WHERE published_at IS NULL                    │
   │         ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED          │
   │              │                                                │
   │              ▼                                                │
   │           broker ──► consumer ──► ★ DEDUPES on the key        │
   └──────────────────────────────────────────────────────────────┘
     ★ crash anywhere ⇒ at worst a DUPLICATE, never a LOSS.
       And duplicates are harmless because the consumer dedupes.
```

**Diagram 2 — data flow: the three-state key across a gateway call**

```
  CLIENT              YOUR SERVICE                 GATEWAY
    │                      │                          │
    │ POST /payments       │                          │
    │ Idempotency-Key: k1  │                          │
    │─────────────────────►│                          │
    │                      │ ① TXN1: INSERT k1        │
    │                      │    status='in_progress'  │
    │                      │    ★ COMMIT — the claim  │
    │                      │      is durable NOW      │
    │                      │                          │
    │                      │ ② charge(key=k1) ───────►│  ★ THE SAME
    │                      │                          │    KEY
    │                      │                          │  charges once
    │                      │◄──── charge_id ──────────│
    │                      │                          │
    │                      │ ★ ⚡ CRASH HERE = the    │
    │                      │   hard case: charged,    │
    │                      │   no local record        │
    │                      │                          │
    │                      │ ③ TXN2: INSERT payment   │
    │                      │    + ledger + outbox     │
    │                      │    + UPDATE k1           │
    │                      │      status='succeeded'  │
    │                      │      response={...}      │
    │                      │    ★ COMMIT — all atomic │
    │◄──── 201 ────────────│                          │
    │                      │                          │
    │ ★ RETRY, same key    │                          │
    │─────────────────────►│ INSERT k1 ⇒ 0 rows       │
    │                      │ status='succeeded'       │
    │◄─ 201, SAME BODY ────│ ★ replayed, not redone   │
    │                      │                          │
    │                      │ ④ SWEEPER (every minute):│
    │                      │   'in_progress' > 5 min? │
    │                      │   ──lookup(key=k1) ─────►│ ★ ASK,
    │                      │◄──── charge or null ─────│   DON'T GUESS
    │                      │   ⇒ complete or fail it  │
```

**Diagram 3 — before/after: a webhook handler**

```
 ✗ BEFORE — 0.3% of payments double-credited
 ┌───────────────────────────────────────────────────────────────┐
 │ app.post('/webhooks/razorpay', async (req, res) => {          │
 │   const e = req.body;                                          │
 │   await db.query(                                              │
 │     `UPDATE accounts SET balance_minor = balance_minor + $1    │
 │       WHERE id = $2`, [e.amount, e.account_id]);   ★ RELATIVE  │
 │   await db.query(                                              │
 │     `INSERT INTO ledger_entries (…) VALUES (…)`);  ★ APPEND    │
 │   res.sendStatus(200);                                         │
 │ });                                                            │
 │                                                                │
 │ ⇒ Razorpay retries on any non-2xx AND on timeout               │
 │ ⇒ a 3-second GC pause ⇒ their timeout ⇒ ★ redelivery           │
 │ ⇒ ★ the balance is credited TWICE                              │
 │ MEASURED: 0.31% of payment webhooks double-applied            │
 │           ₹4.2 lakh over-credited in 5 months                  │
 │           ★ discovered by a customer, not by monitoring        │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — structurally impossible
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE UNIQUE INDEX uq_ledger_source                          │
 │   ON ledger_entries (source, source_event_id);                │
 │                                                                │
 │ app.post('/webhooks/razorpay', async (req, res) => {           │
 │   const e = verifySignature(req);          ★ always first      │
 │   const applied = await withTransaction(async (t) => {         │
 │     const { rowCount } = await t.query(                        │
 │       `INSERT INTO ledger_entries                              │
 │          (source, source_event_id, account_id, amount_minor)   │
 │        VALUES ('razorpay', $1, $2, $3)                         │
 │        ON CONFLICT (source, source_event_id) DO NOTHING`,       │
 │       [e.id, e.account_id, e.amount]);                         │
 │     if (rowCount === 0) return false;   ★ already applied      │
 │                                                                │
 │     await t.query(                                             │
 │       `UPDATE accounts SET balance_minor = balance_minor + $1  │
 │         WHERE id = $2`, [e.amount, e.account_id]);             │
 │     await t.query(`INSERT INTO outbox (…) VALUES (…)`);        │
 │     return true;                        ★ ALL ONE TRANSACTION  │
 │   });                                                          │
 │   res.sendStatus(200);      ★ 200 even on duplicate — stops    │
 │ });                          the provider retrying             │
 │                                                                │
 │ MEASURED: 0 duplicates in 8 months, 41M webhooks              │
 │           ★ no key table needed — THEIR event id IS the key    │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE accounts (id bigint PRIMARY KEY, balance_minor bigint NOT NULL);
INSERT INTO accounts VALUES (1, 100000);

CREATE TABLE ledger_entries (
  id             bigserial PRIMARY KEY,
  account_id     bigint      NOT NULL REFERENCES accounts(id),
  amount_minor   bigint      NOT NULL,
  source         text        NOT NULL,
  source_event_id text       NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX uq_ledger_source ON ledger_entries (source, source_event_id);
```

**Prove a natural key makes a webhook idempotent.**
```sql
-- first delivery
BEGIN;
INSERT INTO ledger_entries (account_id, amount_minor, source, source_event_id)
VALUES (1, 50000, 'razorpay', 'evt_ABC123')
ON CONFLICT (source, source_event_id) DO NOTHING;
```
```
INSERT 0 1        ★ rowCount 1 → proceed
```
```sql
UPDATE accounts SET balance_minor = balance_minor + 50000 WHERE id=1;
COMMIT;

-- redelivery of the SAME event
BEGIN;
INSERT INTO ledger_entries (account_id, amount_minor, source, source_event_id)
VALUES (1, 50000, 'razorpay', 'evt_ABC123')
ON CONFLICT (source, source_event_id) DO NOTHING;
```
```
INSERT 0 0        ★ rowCount 0 → SKIP the credit
```
```sql
ROLLBACK;
SELECT balance_minor FROM accounts WHERE id=1;
```
```
 balance_minor
---------------
        150000        ★ credited ONCE. Two deliveries, one effect.
```

**Prove the concurrent case blocks then fails.**
```sql
-- session 1                          -- session 2
BEGIN;
INSERT INTO ledger_entries
  (account_id, amount_minor, source, source_event_id)
VALUES (1,50000,'razorpay','evt_XYZ');
                                      BEGIN;
                                      INSERT INTO ledger_entries
                                        (account_id, amount_minor, source,
                                         source_event_id)
                                      VALUES (1,50000,'razorpay','evt_XYZ');
                                      -- ⏸ ★ BLOCKS on session 1's XID
COMMIT;
                                      -- wakes up:
-- ERROR:  duplicate key value violates unique constraint
--         "uq_ledger_source"                      ★ SQLSTATE 23505
```
```sql
-- and with ON CONFLICT it becomes a clean rowCount 0 instead:
-- session 2 would report INSERT 0 0 rather than an error.
```

**Prove the non-idempotent shape is broken.**
```sql
UPDATE accounts SET balance_minor = 100000 WHERE id=1;
-- three "retries" of a relative update
UPDATE accounts SET balance_minor = balance_minor + 50000 WHERE id=1;
UPDATE accounts SET balance_minor = balance_minor + 50000 WHERE id=1;
UPDATE accounts SET balance_minor = balance_minor + 50000 WHERE id=1;
SELECT balance_minor FROM accounts WHERE id=1;
```
```
 balance_minor
---------------
      ★ 250000        expected 150000. ₹1,000 created from nothing.
```

**The idempotency key table, three states.**
```sql
CREATE TABLE idempotency_keys (
  key            text PRIMARY KEY,
  request_hash   text NOT NULL,
  status         text NOT NULL CHECK (status IN ('in_progress','succeeded','failed')),
  response_code  int,
  response_body  jsonb,
  created_at     timestamptz NOT NULL DEFAULT now(),
  completed_at   timestamptz
);

-- claim
INSERT INTO idempotency_keys (key, request_hash, status)
VALUES ('idem_9f2a', 'sha256:abc…', 'in_progress')
ON CONFLICT (key) DO NOTHING;
```
```
INSERT 0 1        ★ we own it
```
```sql
-- … external call … then complete
UPDATE idempotency_keys
   SET status='succeeded', response_code=201,
       response_body='{"payment_id":881}', completed_at=now()
 WHERE key='idem_9f2a';

-- a retry
INSERT INTO idempotency_keys (key, request_hash, status)
VALUES ('idem_9f2a', 'sha256:abc…', 'in_progress')
ON CONFLICT (key) DO NOTHING;
```
```
INSERT 0 0
```
```sql
SELECT status, response_code, response_body FROM idempotency_keys WHERE key='idem_9f2a';
```
```
  status   | response_code |    response_body
-----------+---------------+----------------------
 succeeded |           201 | {"payment_id": 881}
   ★ replay this. Do not redo the work.
```

**The outbox, end to end.**
```sql
CREATE TABLE outbox (
  id              bigserial PRIMARY KEY,
  aggregate_type  text        NOT NULL,
  aggregate_id    bigint      NOT NULL,
  event_type      text        NOT NULL,
  payload         jsonb       NOT NULL,
  idempotency_key text        NOT NULL UNIQUE,
  created_at      timestamptz NOT NULL DEFAULT now(),
  published_at    timestamptz
);
CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published_at IS NULL;

-- ★ per-table autovacuum — MANDATORY on a queue table (Topic 47)
ALTER TABLE outbox SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_vacuum_threshold    = 1000,
  autovacuum_vacuum_cost_delay   = 0
);

-- the write path
BEGIN;
INSERT INTO ledger_entries (account_id, amount_minor, source, source_event_id)
VALUES (1, 25000, 'internal', 'txn-5501') RETURNING id \gset
INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload, idempotency_key)
VALUES ('ledger', :id, 'LedgerEntryCreated',
        jsonb_build_object('entry_id', :id), 'ledger-created:' || :id);
COMMIT;

-- the relay — run this in several sessions at once
BEGIN;
SELECT id, event_type, payload, idempotency_key
  FROM outbox WHERE published_at IS NULL
 ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED;
-- … publish …
UPDATE outbox SET published_at = now() WHERE id = ANY($1);
COMMIT;
-- ★ N relays, zero contention, no double-claiming (Topic 45)
```

**Monitor the lag.**
```sql
SELECT count(*) AS pending,
       coalesce(extract(epoch from now() - min(created_at)), 0)::int AS oldest_s
  FROM outbox WHERE published_at IS NULL;
```
```
 pending | oldest_s
---------+----------
      12 |        0        ★ alert if oldest_s > 60
```

**Retention — don't `DELETE` (Topics 47, 59).**
```sql
-- ✗ DELETE FROM outbox WHERE published_at < now() - interval '7 days';
--   ⇒ millions of dead tuples, index churn, no space returned

-- ✓ partition by day, DROP the old ones
CREATE TABLE outbox_p (LIKE outbox INCLUDING ALL) PARTITION BY RANGE (created_at);
CREATE TABLE outbox_p_2026_08_18 PARTITION OF outbox_p
  FOR VALUES FROM ('2026-08-18') TO ('2026-08-19');
DROP TABLE outbox_p_2026_08_11;   -- ★ instant, no dead tuples, ~0 WAL
```

---

## Example 2 — production scenario

**The situation.** A food-delivery platform. Order placement charges a card, decrements restaurant capacity, and notifies the dispatch service. A customer support thread reaches engineering:

```
 "I was charged three times for one order."
   → 41 similar tickets in 6 weeks
   → ★ ₹2.8 lakh refunded
   → ★ 1 chargeback and a payment-provider risk-review flag
```

**Step 1 — find every place a retry can happen.**

```js
// mobile client
async function placeOrder(cart) {
  return retry(() => fetch('/api/orders', {
    method: 'POST', body: JSON.stringify(cart)
  }), { attempts: 3, timeout: 8000 });     // ★ retries on TIMEOUT
}

// server
app.post('/api/orders', async (req, res) => {
  const order = await withTransaction(async (t) => {
    const { rows: [o] } = await t.query(
      `INSERT INTO orders (customer_id, restaurant_id, total_minor, status)
       VALUES ($1,$2,$3,'pending') RETURNING id`, [...]);
    await t.query(
      `UPDATE restaurants SET open_orders = open_orders + 1 WHERE id=$1`, [...]);
    return o;
  });
  const charge = await gateway.charge({ amountMinor: req.body.total_minor });  // ★ 3–9 s
  await pool.query('UPDATE orders SET status=$1, gateway_ref=$2 WHERE id=$3',
                   ['paid', charge.id, order.id]);
  await dispatch.notify(order.id);                                             // ★ another network call
  res.json(order);
});
```

```
 ★ SIX SEPARATE DEFECTS, EACH OF WHICH ALONE CAUSES DUPLICATES:
 ① no idempotency key anywhere
 ② the client retries on TIMEOUT — the case where the request
   most likely DID succeed
 ③ ★ gateway.charge() takes 3–9 s and the client timeout is 8 s
   ⇒ the retry fires almost exactly when the charge succeeds
 ④ open_orders = open_orders + 1 is RELATIVE ⇒ triple-counted
 ⑤ ★ dispatch.notify() is a dual write — a crash after the charge
   means paid, undispatched, and no record of intent
 ⑥ ★ NO KEY IS PASSED TO THE GATEWAY, so the gateway cannot
   deduplicate either
```

**Step 2 — measure it properly.**

```sql
SELECT customer_id, restaurant_id, total_minor,
       count(*) AS dupes,
       min(created_at) AS first, max(created_at) AS last,
       max(created_at) - min(created_at) AS spread
  FROM orders
 WHERE created_at > now() - interval '30 days'
 GROUP BY customer_id, restaurant_id, total_minor,
          date_trunc('minute', created_at)
HAVING count(*) > 1
 ORDER BY dupes DESC LIMIT 5;
```
```
 customer_id | restaurant_id | total_minor | dupes |  spread
-------------+---------------+-------------+-------+----------
       88412 |          1204 |       84500 |   ★ 3 | 00:00:16
       41209 |           882 |      112000 |     2 | 00:00:08
```
```
 ★ SPREAD OF 8–16 SECONDS = exactly the client's retry schedule.
   This is not a race between users. It is one user's client,
   retrying.
```

```sql
-- and the true blast radius
SELECT count(*) FILTER (WHERE dupes > 1) AS duplicate_groups,
       sum(total_minor) FILTER (WHERE dupes > 1)/100.0 AS rupees_over_charged
  FROM (SELECT customer_id, total_minor, count(*) AS dupes FROM orders
         WHERE created_at > now() - interval '180 days'
         GROUP BY 1, 2, date_trunc('minute', created_at)) x;
```
```
 duplicate_groups | rupees_over_charged
------------------+---------------------
              412 |          ★ 284,100
```

**Step 3 — the fix, all six defects.**

```sql
-- ① the key table
CREATE TABLE idempotency_keys (
  key           text PRIMARY KEY,
  request_hash  text NOT NULL,
  status        text NOT NULL CHECK (status IN ('in_progress','succeeded','failed')),
  response_code int,
  response_body jsonb,
  created_at    timestamptz NOT NULL DEFAULT now(),
  completed_at  timestamptz
);
CREATE INDEX idx_idem_stale ON idempotency_keys (created_at) WHERE status='in_progress';

-- ② the outbox
CREATE TABLE outbox (
  id bigserial PRIMARY KEY,
  aggregate_type text NOT NULL, aggregate_id bigint NOT NULL,
  event_type text NOT NULL, payload jsonb NOT NULL,
  idempotency_key text NOT NULL UNIQUE,
  created_at timestamptz NOT NULL DEFAULT now(), published_at timestamptz
);
CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
ALTER TABLE outbox SET (autovacuum_vacuum_scale_factor=0.01,
                        autovacuum_vacuum_cost_delay=0);

-- ③ ★ open_orders becomes derivable, not accumulated
--    a relative counter can never be made idempotent; remove it.
DROP … ; -- open_orders column
CREATE INDEX idx_orders_open ON orders (restaurant_id)
  WHERE status IN ('pending','paid','preparing');
-- ⇒ count(*) over a partial index. Always correct, never drifts.
```

```js
// ④ the client — ★ generate the key ONCE, reuse across retries
async function placeOrder(cart) {
  const key = crypto.randomUUID();          // ★ NOT inside retry()
  return retry(() => fetch('/api/orders', {
    method: 'POST',
    headers: { 'Idempotency-Key': key, 'Content-Type': 'application/json' },
    body: JSON.stringify(cart),
  }), { attempts: 3, timeout: 15000 });     // ★ > the gateway's p99
}
```

```js
// ⑤ the server
app.post('/api/orders', async (req, res) => {
  const key = req.get('Idempotency-Key');
  if (!key) return res.status(400).json({ error: 'IDEMPOTENCY_KEY_REQUIRED' });
  const hash = sha256(canonicalJson(req.body));

  const claim = await withTransaction(t => t.query(
    `INSERT INTO idempotency_keys (key, request_hash, status)
     VALUES ($1,$2,'in_progress') ON CONFLICT (key) DO NOTHING RETURNING key`,
    [key, hash]));

  if (claim.rowCount === 0) {
    const { rows: [prev] } = await pool.query(
      'SELECT * FROM idempotency_keys WHERE key=$1', [key]);
    if (prev.request_hash !== hash)
      return res.status(422).json({ error: 'IDEMPOTENCY_KEY_REUSED' });
    if (prev.status === 'in_progress')
      return res.status(409).json({ error: 'IN_PROGRESS', retryAfter: 3 });
    return res.status(prev.response_code).json(prev.response_body);
  }

  // ★ the gateway gets OUR key — so it dedupes too
  let charge;
  try {
    charge = await gateway.charge({
      amountMinor: req.body.total_minor, idempotencyKey: key });
  } catch (e) {
    await pool.query(
      `UPDATE idempotency_keys SET status='failed', response_code=502,
              response_body=$1, completed_at=now() WHERE key=$2`,
      [{ error: e.code }, key]);
    return res.status(502).json({ error: e.code });
  }

  const body = await withTransaction(async (t) => {
    const { rows: [o] } = await t.query(
      `INSERT INTO orders (customer_id, restaurant_id, total_minor,
                           status, gateway_ref)
       VALUES ($1,$2,$3,'paid',$4) RETURNING id, status`,
      [req.body.customer_id, req.body.restaurant_id,
       req.body.total_minor, charge.id]);

    // ★ no more dispatch.notify() — an outbox row instead
    await t.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type,
                           payload, idempotency_key)
       VALUES ('order',$1,'OrderPaid',$2,$3)`,
      [o.id, { order_id: o.id, restaurant_id: req.body.restaurant_id },
       `order-paid:${o.id}`]);

    const out = { order_id: o.id, status: o.status };
    await t.query(
      `UPDATE idempotency_keys SET status='succeeded', response_code=201,
              response_body=$1, completed_at=now() WHERE key=$2`, [out, key]);
    return out;
  });
  res.status(201).json(body);
});
```

```js
// ⑥ the sweeper — ★ reconcile, never guess
async function sweep() {
  const { rows } = await pool.query(
    `SELECT key FROM idempotency_keys
      WHERE status='in_progress' AND created_at < now() - interval '3 minutes'
      LIMIT 200`);
  for (const { key } of rows) {
    const charge = await gateway.lookupByIdempotencyKey(key);   // ★ ASK
    if (charge?.status === 'captured') {
      await completeOrderFromCharge(key, charge);   // idempotent itself
      metrics.increment('idem.swept_recovered');
    } else {
      await pool.query(
        `UPDATE idempotency_keys SET status='failed', response_code=504,
                response_body='{"error":"GATEWAY_TIMEOUT"}', completed_at=now()
          WHERE key=$1 AND status='in_progress'`, [key]);
      metrics.increment('idem.swept_failed');
    }
  }
}
```

**Step 4 — the dispatch consumer dedupes.**

```js
async function onOrderPaid(event) {
  await withTransaction(async (t) => {
    const { rowCount } = await t.query(
      `INSERT INTO processed_events (idempotency_key, consumer)
       VALUES ($1,'dispatch') ON CONFLICT DO NOTHING`, [event.key]);
    if (rowCount === 0) return;                    // ★ already handled
    await t.query(
      `INSERT INTO dispatch_jobs (order_id, status) VALUES ($1,'queued')
       ON CONFLICT (order_id) DO NOTHING`, [event.order_id]);
  });
}
```

**Step 5 — the monitoring that would have caught it in week one.**

```sql
-- ① ★ the duplicate detector, run hourly — this is the alert that
--    should have existed from day one
SELECT count(*) FROM (
  SELECT customer_id, total_minor
    FROM orders WHERE created_at > now() - interval '1 hour'
   GROUP BY 1, 2, date_trunc('minute', created_at)
  HAVING count(*) > 1) x;
-- ★ alert: > 0

-- ② outbox lag
SELECT extract(epoch from now() - min(created_at))::int
  FROM outbox WHERE published_at IS NULL;              -- ★ alert > 60

-- ③ stuck keys
SELECT count(*) FROM idempotency_keys
 WHERE status='in_progress' AND created_at < now() - interval '5 minutes';
-- ★ alert > 0

-- ④ sweeper recoveries — ★ a NON-ZERO rate here is the leading
--    indicator of a gateway or timeout problem
-- metric: idem.swept_recovered
```

**Step 6 — retention.**

```sql
-- ★ both tables are queues. Neither may use DELETE for retention.
DELETE FROM idempotency_keys
 WHERE completed_at < now() - interval '30 days';    -- ✗ bloat

-- ✓ partition both by created_at, drop old partitions (Topics 47, 59)
DROP TABLE outbox_2026_07;
DROP TABLE idempotency_keys_2026_07;
-- ⇒ ★ retention window must EXCEED the client's maximum retry
--   horizon, or an old retry finds no key and does the work again.
```

**Step 7 — results, 6 months.**

| | Before | After |
|---|---|---|
| Duplicate orders | 412 / 180 days | **0** |
| Over-charged | ₹2.84 lakh | **₹0** |
| Chargebacks | 1 (+ risk flag) | 0 |
| Dispatch notifications lost | ~0.2% (dual write) | **0** |
| `open_orders` drift | up to 14 | **n/a** — derived |
| Detection | ★ customer tickets | 3 alerts, hourly |
| Sweeper recoveries | n/a | 4–11/day (★ real gateway timeouts, now recovered automatically) |
| p99 on `POST /orders` | 4,100 ms | 3,900 ms (unchanged — the gateway dominates) |

```
 ★ THREE LESSONS:
 ① THE BUG WAS ARCHITECTURAL, NOT A MISTAKE. Every individual
   line was reasonable. The system had no notion of "this request
   is the same request."
 ② ★ THE CLIENT'S RETRY TIMEOUT (8 s) WAS SHORTER THAN THE
   GATEWAY'S p99 (9 s). That single misconfiguration turned a
   theoretical risk into a 0.3% duplicate rate. ★ ALWAYS SET
   CLIENT TIMEOUTS ABOVE THE DEPENDENCY'S p99.
 ③ ★ THE SWEEPER FINDS 4–11 GENUINELY STUCK PAYMENTS PER DAY.
   Those existed before too — they were simply lost, silently.
   The pattern didn't create that failure; it made it VISIBLE
   and RECOVERABLE.
```

---

## Common mistakes

**1. Storing the idempotency key outside the database.**
- *Symptom:* duplicates that survive the "fix"; a crash between the work and the key write.
- *Engine-level why:* the key and the work must commit atomically. Redis is a separate failure domain — it re-creates the dual-write problem.
- *Fix:* a table in the same database, guarded by `UNIQUE`.

**2. Generating the key inside the retry loop.**
- *Symptom:* every retry gets a fresh key and does the work again.
- *Fix:* generate once, at the top of the logical operation.

**3. A boolean "processed" flag instead of three states.**
- *Symptom:* a crash after marking the key but before the work leaves the operation permanently skipped — **silent data loss**.
- *Fix:* `in_progress` / `succeeded` / `failed`, plus a sweeper.

**4. No sweeper for stuck keys.**
- *Symptom:* `in_progress` rows accumulate; retries get 409 forever; money is in limbo.
- *Fix:* reconcile against the provider's API. **Never guess.**

**5. Not passing the key to the external provider.**
- *Symptom:* the crash between "gateway charged" and "we recorded it" is unrecoverable.
- *Fix:* use the *same* key for your table and the provider's `Idempotency-Key`.

**6. Client timeout shorter than the dependency's p99.**
- *Symptom:* retries fire precisely when the original is about to succeed.
- *Fix:* client timeout > dependency p99, with a margin.

**7. Publishing to a broker inside the transaction.**
- *Symptom:* the transaction rolls back but the event was already sent — a phantom event.
- *Fix:* the outbox. Publish only after commit, from a separate process.

**8. An outbox with no lag alert or reconciler.**
- *Symptom:* the relay dies; events silently stop; nobody notices for days.
- *Fix:* alert on `min(created_at)` age; a reconciler for aggregates stuck in an intermediate state.

**9. `DELETE`-based retention on outbox / key tables.**
- *Symptom:* the highest-churn tables in the system bloat catastrophically.
- *Fix:* partition and `DROP`; aggressive per-table autovacuum meanwhile (Topics 47, 59).

**10. A key retention window shorter than the client's retry horizon.**
- *Symptom:* a mobile client retries a 40-day-old queued request; the key is gone; the work runs again.
- *Fix:* retention must exceed the maximum retry horizon of every client.

**11. Returning a non-2xx for a detected duplicate.**
- *Symptom:* the provider treats it as failure and retries harder.
- *Fix:* return `200`/`201` with the stored response. Duplicate detection is success.

**12. Relative updates on anything a retry can touch.**
- *Symptom:* counters drift; balances inflate.
- *Fix:* absolute values, conditional transitions, or derive the value from rows.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (natural-key dedupe, the concurrent duplicate blocking then failing with `23505`, the relative-update triple-apply, the three-state key table, the outbox with `SKIP LOCKED`, lag monitoring, per-table autovacuum, partition-based retention).

**PROVE IT #9 — the crash between key and work.**
```sql
-- ✗ the broken shape: key committed separately from the work
BEGIN; INSERT INTO processed_events (key) VALUES ('e1'); COMMIT;
-- ★ crash here
-- retry:
INSERT INTO processed_events (key) VALUES ('e1') ON CONFLICT DO NOTHING;
-- ⇒ rowCount 0 ⇒ "already done" ⇒ ★ THE WORK NEVER HAPPENS

-- ✓ the correct shape: one transaction
BEGIN;
  INSERT INTO processed_events (key) VALUES ('e2') ON CONFLICT DO NOTHING;
  -- if rowCount 0, ROLLBACK and skip
  UPDATE accounts SET balance_minor = balance_minor + 100 WHERE id=1;
COMMIT;
-- ★ crash anywhere ⇒ neither happened ⇒ the retry does both
```

**PROVE IT #10 — the relay is crash-safe (at-least-once, never lost).**
```sql
BEGIN;
SELECT id FROM outbox WHERE published_at IS NULL
 ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED;
-- ★ kill the relay process here, before UPDATE
```
```sql
SELECT count(*) FROM outbox WHERE published_at IS NULL;
-- ★ unchanged. The row lock was released by the abort; the next
--   relay picks it up. Nothing is lost. It may be published twice —
--   which is why the consumer dedupes.
```

**PROVE IT #11 — N relays never double-claim.**
```bash
for i in 1 2 3 4; do
  psql -d shop -c "BEGIN; SELECT id FROM outbox WHERE published_at IS NULL
     ORDER BY id LIMIT 5 FOR UPDATE SKIP LOCKED; SELECT pg_sleep(2); COMMIT;" &
done
wait
# ★ each session returns a DIFFERENT set of 5 ids. Zero overlap,
#   zero waiting.
```

**PROVE IT #12 — measure outbox bloat without tuning.**
```sql
INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload, idempotency_key)
SELECT 'x', g, 'E', '{}', 'k'||g FROM generate_series(1,500000) g;
UPDATE outbox SET published_at = now();
SELECT n_live_tup, n_dead_tup,
       pg_size_pretty(pg_total_relation_size('outbox')) AS size
  FROM pg_stat_user_tables WHERE relname='outbox';
```
```
 n_live_tup | n_dead_tup |  size
------------+------------+--------
     500000 |   ★ 500000 | 214 MB
   ★ every row was inserted then updated ⇒ 100% dead-tuple ratio
     on one pass. This is why a queue table needs aggressive
     autovacuum and partition-based retention.
```

---

## The design decision framework

```
★★★ ASSUME EVERY WRITE WILL BE DELIVERED MORE THAN ONCE. ★★★
    Design so that is boring.

 ① CAN THE OPERATION BE MADE NATURALLY IDEMPOTENT?
    ⇒ ★ ALWAYS ASK FIRST. It costs nothing.
    ✓ absolute, not relative:  SET status='paid'  not  n = n + 1
    ✓ UPSERT:                  ON CONFLICT DO UPDATE
    ✓ conditional transition:  WHERE id=$1 AND status='pending'
    ✓ ★ a UNIQUE on a NATURAL business key
       (the provider's event id, the invoice number, the order id)
       ⇒ NO SEPARATE KEY TABLE NEEDED. This is the best outcome.
    ✓ derive counters from rows instead of accumulating them
    ⇒ if you get here, you are done.

 ② IF NOT, USE AN EXPLICIT IDEMPOTENCY KEY
    ✓ ★ in the SAME database, with a UNIQUE constraint
    ✓ ★ generated ONCE by the client, reused across retries
    ✓ ★ THREE states, not a boolean
    ✓ ★ a request hash ⇒ same key + different body = 422
    ✓ ★ the stored response, replayed byte-for-byte
    ✓ ★ a SWEEPER that RECONCILES with the provider — never guesses
    ✓ ★ retention > the longest client retry horizon
    ✓ ★ 2xx on a detected duplicate (a non-2xx makes it retry harder)

 ③ IF AN EXTERNAL CALL IS INVOLVED
    ✓ ★ pass YOUR key as the provider's Idempotency-Key
    ✓ ★ the call happens OUTSIDE any transaction
    ✓ ★ client timeout > the dependency's p99, with margin
    ✓ a sweeper for the crash-between window

 ④ IF A MESSAGE MUST BE SENT
    ✗ never publish inside the transaction
    ✗ never publish before it
    ✗ ★ never 2PC across the DB and the broker (Topic 51)
    ✓ ★ THE OUTBOX: same transaction, separate relay
    ✓ relay with FOR UPDATE SKIP LOCKED ⇒ N relays, no contention
    ✓ ★ THE CONSUMER MUST DEDUPE. Always. Non-negotiable.
    ✓ need <10 ms? logical decoding instead of polling (Topic 76)

 ⑤ TREAT BOTH TABLES AS QUEUES
    ★ they have the highest churn in your system
    ✓ per-table autovacuum: scale_factor 0.01, cost_delay 0
    ✓ ★ partition by day, DROP — never DELETE (Topics 47, 59)

 ⑥ THE THREE ALERTS — WITHOUT THESE YOU HAVE INCONSISTENCY WITH
    EXTRA STEPS
    ✓ ★ outbox lag:      age of the oldest unpublished row > 60 s
    ✓ ★ stuck keys:      'in_progress' older than 5 min > 0
    ✓ ★ a DUPLICATE DETECTOR on your actual business entities
      ⇒ the one that would have caught the 6-month leak

 ⑦ "EXACTLY-ONCE" IS A MARKETING TERM
    ★ across a network it always means:
      at-least-once delivery + an idempotent consumer.
    ⇒ if someone claims exactly-once, ask where the dedupe lives.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
For each, say whether it is idempotent and rewrite the ones that are not:
(a) `UPDATE accounts SET balance = balance - 500 WHERE id=1` · (b) `UPDATE orders SET status='shipped' WHERE id=7` · (c) `INSERT INTO ledger (…) VALUES (…)` · (d) `DELETE FROM sessions WHERE id=$1` · (e) `POST /payments {amount: 500}` · (f) `INSERT … ON CONFLICT DO NOTHING`

Then demonstrate the triple-apply of (a) and prove your rewrite is safe.

### Exercise 2 — medium (apply it)
Build the complete outbox: schema with partial index, per-table autovacuum, a write path, a relay using `SKIP LOCKED`, and a deduplicating consumer.

Then prove: (a) four relays never double-claim; (b) killing a relay mid-batch loses nothing; (c) a redelivered event applies once; (d) after 500,000 published rows, measure the dead-tuple ratio and explain why partitioning is required.

### Exercise 3 — hard (production simulation)
A food-delivery platform has 412 duplicate orders over 180 days and ₹2.84 lakh over-charged, discovered from support tickets. The client retries on timeout with an 8-second budget; the payment gateway's p99 is 9 seconds. The server has no idempotency key, uses `open_orders = open_orders + 1`, and calls `dispatch.notify()` after the charge.

(a) Enumerate all six independent defects and explain how each alone produces duplicates.
(b) Write the detection query. Explain what an 8–16 second spread between duplicates tells you about the cause.
(c) Explain precisely why the 8-second client timeout against a 9-second p99 turns a theoretical risk into a 0.3% duplicate rate.
(d) Design the three-state key table. Justify each state, the request hash, and the stored response.
(e) Write the endpoint. Mark exactly which parts are inside a transaction and which are not, and say why for each.
(f) Explain what happens if the process crashes between the gateway call and the local commit — and the two mechanisms that make it recoverable.
(g) Write the sweeper. Explain why it must reconcile against the provider rather than time out.
(h) `open_orders` is a relative counter. Explain why no idempotency scheme can fix it, and give the alternative.
(i) Replace `dispatch.notify()` with an outbox. Write the relay and the deduplicating consumer.
(j) Write the three alerts, with thresholds. Which one would have caught this in week one?
(k) Both new tables are queues. Give the retention design and explain why the window must exceed the client's retry horizon.
(l) The sweeper now finds 4–11 stuck payments per day. Were those failures created by the new design? Explain.

---

## Mental model checkpoint

1. Define idempotency. Give three naturally idempotent SQL shapes and three that never are.
2. Why must the idempotency key live in the same database as the work? What breaks with Redis?
3. What happens when two requests with the same key arrive 2 ms apart? How long does the second wait?
4. Why is a boolean "processed" flag insufficient? What is the exact silent-data-loss sequence?
5. Name the three states and the role of each. What does the sweeper do, and what must it never do?
6. State the dual-write problem in both orderings. Why is publish-then-commit worse?
7. What does the outbox guarantee? Name three things it does *not*.
8. Why must the consumer deduplicate even with a correct outbox?
9. Why is the outbox table a bloat risk, and what are the two mitigations?
10. Why must the key retention window exceed the client's retry horizon?
11. What does "exactly-once delivery" actually mean?

---

## Quick reference card

**Naturally idempotent — prefer these**
```sql
UPDATE orders SET status='shipped' WHERE id=$1 AND status='paid';   -- conditional
INSERT … ON CONFLICT (natural_key) DO NOTHING;                      -- ★ best
INSERT … ON CONFLICT (id) DO UPDATE SET …;                          -- upsert
DELETE FROM sessions WHERE id=$1;
-- ✗ never: x = x + 1 · plain INSERT · "send"
```

**The three-state key**
```sql
CREATE TABLE idempotency_keys (
  key text PRIMARY KEY,
  request_hash text NOT NULL,                          -- ★ 422 on reuse
  status text NOT NULL CHECK (status IN ('in_progress','succeeded','failed')),
  response_code int, response_body jsonb,
  created_at timestamptz NOT NULL DEFAULT now(), completed_at timestamptz);
```
```
① claim (txn)  ② external call (★ no txn, ★ same key to provider)
③ effects + completion (txn)  ④ ★ sweeper reconciles stuck keys
```

**The outbox**
```sql
BEGIN; INSERT INTO orders …; INSERT INTO outbox …; COMMIT;   -- ★ one fsync
```
```sql
SELECT … FROM outbox WHERE published_at IS NULL
 ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED;               -- ★ N relays
CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
ALTER TABLE outbox SET (autovacuum_vacuum_scale_factor=0.01,
                        autovacuum_vacuum_cost_delay=0);     -- ★ it's a queue
```

**The three alerts**
```sql
SELECT extract(epoch from now()-min(created_at))::int FROM outbox
 WHERE published_at IS NULL;                                  -- ★ > 60 s
SELECT count(*) FROM idempotency_keys
 WHERE status='in_progress' AND created_at < now()-interval '5 min';  -- ★ > 0
-- ★ + a duplicate detector on your real business entities
```

**Rules:** key generated **once** by the client · key + work in **one transaction** · external call **outside** it · **same key to the provider** · client timeout **> dependency p99** · consumer **always** dedupes · retention **>** retry horizon · **2xx on duplicates** · **partition and DROP**, never `DELETE`.

**"Exactly-once" = at-least-once delivery + idempotent consumer.** There is no other kind.

---

## When would I use this at work?

1. **Every `POST` that costs money or sends something.** An `Idempotency-Key` header is table stakes for a payments API, and the reason Stripe, Razorpay and every serious provider require one. If your API doesn't accept one, your clients cannot safely retry — which means they retry unsafely.

2. **Every webhook handler.** Providers deliver at-least-once by design. A `UNIQUE (source, source_event_id)` with `ON CONFLICT DO NOTHING` makes the handler correct in three lines and needs no key table at all.

3. **Any time someone proposes writing to a database and a message broker in one operation.** The outbox is the answer, and it's simpler than the alternatives people reach for. It also removes the reason anyone was considering 2PC (Topic 51).

4. **Whenever you add a retry loop.** Everything in Phase 5 told you to retry `40001` and `40P01`. This is the topic that makes those retries safe — the transaction body must be idempotent, or you've traded a visible failure for an invisible corruption.

---

## Connected topics

**Understand before this:** 24 (`UNIQUE` — the enforcement mechanism), 44/48/50 (the retryable errors this makes safe), 45 (`SKIP LOCKED` — the relay pattern), 49 (optimistic concurrency — retries need idempotency), 51 (2PC — what this replaces).

**This unlocks:**
- **76** — CDC and logical decoding: the outbox without a table, at sub-10 ms
- **59** — partitioning: retention for outbox and key tables
- **Case study 03** — the payment ledger, built entirely on these primitives
- **Case study 09** — notification delivery: at-least-once end to end
- **Case study 12** — food-delivery orders: sagas and compensation in full

---

> ### ★ PHASE 5 COMPLETE — Transactions & Concurrency (39–52)
>
> **What you can now do:** name any concurrency anomaly from its symptom · choose an isolation level from measured evidence rather than folklore · read a lock queue and resolve an incident in one query · explain bloat, wraparound and HOT updates from first principles · eliminate deadlocks structurally · choose between atomic, pessimistic and optimistic control by measuring `p` · argue a team out of 2PC with arithmetic · and make every retry in the system safe.
>
> **Next: Phase 6 — Denormalisation & Scale (53–61).** Everything so far assumed one machine and a correct schema. Now: when correctness must bend for speed, and how to bend it without breaking it.
