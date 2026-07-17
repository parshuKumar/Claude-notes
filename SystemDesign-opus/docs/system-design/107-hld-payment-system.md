# 107 — Design a Payment System
## Category: HLD Case Study

---

## What is this?

A payment system moves money from a customer to a merchant and **keeps a perfect, permanent record of every cent that moved.** Stripe, Adyen, PayPal, Square, the checkout button on Amazon — under the hood they all solve the same problem: take money from one party, give it to another, do it exactly once, and be able to prove, years later, that the books balanced to the penny.

Notice what makes this different from every other system in this course. A feed can show a slightly stale post. A cache can miss. A chat message can arrive 200ms late. **None of that is acceptable here.** If you charge a customer twice, you've stolen from them. If you charge once but forget to record it, you've stolen from yourself. If your ledger says $10,000,050 but the bank settled $10,000,000, you have a $50 hole and — legally — you must find it.

Think of it as a **bank's accounting department, not a web app.** The web app is the thin skin on top. The real system is a machine for producing an **immutable, auditable, always-balanced financial record**, and everything else — the API, the queue, the retries — exists to protect that record from the messy, unreliable, partially-failing distributed world it lives in.

---

## Why does it matter?

This is the case study where **correctness matters more than anything else** — more than latency, more than scale, more than elegance. Every other system earlier in the course chased availability and traded away consistency (the feed, the URL shortener, the chat store all embraced eventual consistency). **A payment system is the anti-thesis of that.** Here you trade latency and even some availability *to buy correctness*, and you say that trade out loud.

- Your normal instinct is **"eventually consistent is fine, users won't notice."** Here it is a lawsuit. Money that is temporarily double-counted is money that got spent twice.
- Your normal instinct is **`UPDATE accounts SET balance = balance + 50`.** That single line is the classic wrong answer. It destroys history, it races under concurrency, and it makes fraud invisible. You will replace it with something append-only.
- Your normal instinct is **"just call the payment API and update my DB."** But those are two different systems, they cannot share one transaction, and the gap between them is exactly where money leaks.

**At work**, the moment you touch billing, subscriptions, marketplace payouts, refunds, or invoicing, you inherit these constraints. Getting them wrong doesn't show up as a bug ticket — it shows up as an angry customer, a chargeback, or a finance team that can't close the books.

**In the interview**, the signal is one thing: *do you understand that money requires a fundamentally different discipline than data?* Candidates who draw "client → payment service → database" and call `UPDATE balance` have failed. Candidates who say "I never touch raw card numbers, I record every movement as balanced double-entry ledger rows, I make every operation idempotent, and I reconcile against the provider nightly" have passed before they've drawn a single box.

---

## The core idea — explained simply

### The Accountant's Ledger Analogy

Imagine an old-fashioned bookkeeper with a big leather-bound ledger and a pen. Their rulebook has four unbreakable laws:

1. **You never erase.** If you make a mistake, you don't scratch it out — you write a *new, correcting line*. The old line stays visible forever. The book is a history, not a snapshot.
2. **Every entry has two sides.** Money never appears or vanishes; it *moves*. So every event is written as (at least) two lines: one account gives (a **debit**), another account receives (a **credit**), and the two amounts are equal. Fifty dollars leaving Alice is the *same event* as fifty dollars arriving at the merchant.
3. **The book must always balance.** Add up every debit and every credit across the entire ledger and the total must be **exactly zero**. If it isn't, you don't have a rounding quirk — you have a *lost transaction*, and you stop everything and find it.
4. **A balance is never stored — it's computed.** To know how much is in an account, you don't look up a "balance" field. You *sum every line that ever touched that account*. The balance is a derived fact, always reconstructable from history.

Now watch a $50 card payment flow through this bookkeeper's world:

- The customer taps **Pay**. Before anything, the bookkeeper tears off a numbered **claim ticket** (an idempotency key) and staples it to the request. If the same ticket ever comes back — because the network hiccuped and the app retried — the bookkeeper doesn't do the work twice; they just hand back the *same receipt* they filed the first time.
- The bookkeeper calls the **card network** (they don't handle the physical cash themselves — a specialist, the *payment processor*, does that and takes on the risk of touching the actual card).
- When the money moves, the bookkeeper writes **two balanced lines** in permanent ink.
- That night, they get a statement from the card network and go **line by line**, ticking off each one against their own ledger. Anything that doesn't match gets circled in red for a human to investigate.

That's the whole system. Everything below is detail.

| Bookkeeper's world | System design thing |
|---|---|
| The leather ledger, written in permanent ink | The **append-only ledger table** — the source of truth |
| Two balanced lines per event (debit + credit) | **Double-entry bookkeeping** — the zero-sum invariant |
| "Sum every line to get the balance" | Balances are **derived**, never `UPDATE`d in place |
| The stapled claim ticket | The **idempotency key** — process once, replay the result |
| The cash specialist who touches the card | The **payment processor** (Stripe/Adyen) — keeps you out of PCI scope |
| The multi-step payment with a call to the card network | A **saga** — a distributed transaction with compensations |
| Nightly line-by-line check against the statement | **Reconciliation** — the safety net that catches drift |

---

## Key concepts inside this topic

We'll run the **5-step framework from [93 — How to Attack an HLD Interview](./93-hld-approach-framework.md)** — requirements → estimation → API/data → architecture → deep dives — but the *center of gravity* is different from every other case study. Here the deep material is **money-movement correctness**: double-entry bookkeeping, idempotency, the payment saga, the outbox, and reconciliation. Those five ideas are the interview.

---

### 1. Requirements

**Clarifying questions to ask first** (ask these out loud — it's scored):

- "Are we the merchant taking payments, or are we building the *processor* like Stripe?" → *Assume we're a platform taking payments and integrating a processor. We do NOT build the card-network connection ourselves.*
- "Cards only, or wallets and bank transfers too?" → *Cards + wallets (Apple Pay / PayPal). Design so adding a method is a new adapter, not a rewrite.*
- "Do we need refunds and partial refunds?" → *Yes. Refunds are first-class, not an afterthought.*
- "Are we in PCI scope — do we store card numbers?" → *No, and that's a deliberate design goal. We tokenize.*

**Functional requirements**

| # | Requirement |
|---|---|
| F1 | Process a payment from a customer to a merchant |
| F2 | Support multiple methods: cards, digital wallets |
| F3 | Handle **refunds** (full and partial) |
| F4 | Record **every** transaction in an immutable **ledger** |
| F5 | **Reconcile** our ledger against the external provider's settlement report |
| F6 | Expose payment status to the customer and merchant |

**Non-functional requirements** — these *are* the design here:

| # | Requirement | What it forces |
|---|---|---|
| N1 | **Correctness above all** — no double charges, no lost money, every cent accounted for | Idempotency everywhere; double-entry ledger; reconciliation |
| N2 | **Strong consistency & durability** | ACID for the ledger write. This is the anti-thesis of the eventually-consistent systems earlier in the course — *say so* |
| N3 | **Auditability** — a complete, immutable financial record for compliance | Append-only ledger, never updated or deleted |
| N4 | **High availability** — you can't refuse a customer's money | Async retries; queue-and-retry when the provider is down |
| N5 | **Reasonable latency** (p99 a few seconds is fine) | We *accept* latency to guarantee correctness — the opposite trade from a feed |

**The scope decision that shapes everything: you stay out of PCI.** The Payment Card Industry Data Security Standard (PCI-DSS) governs anyone who *stores, processes, or transmits raw card numbers* (the PAN — Primary Account Number). That obligation is brutally expensive: hardened networks, annual audits, enormous liability if you're breached. **So you don't touch the card number.** You integrate a payment processor (Stripe, Adyen, Braintree), the customer's card details go **straight from their browser to the processor** (via a hosted field or SDK), and the processor hands you back a **token** — an opaque string like `tok_1P9x...` that represents the card but reveals nothing. You store the token. You are never PCI scope for the PAN, because the PAN never enters your systems. This is not a shortcut; it is the correct, standard architecture, and stating it is a maturity signal.

---

### 2. Capacity estimation

**Traffic** — and the surprising headline: *payment TPS is modest.*

```
Transactions per day         = 10,000,000  (10M — a large payment platform)
Seconds per day              ≈ 100,000     (86,400, rounded for mental math)

Average TPS                  = 10M / 100,000  = 100 TPS
Peak (Black Friday, ~10×)    ≈ 1,000 TPS
```

**Stop and compare.** The chat system in [99] pushed **50,000** messages/sec. A feed serves *hundreds of thousands* of reads/sec. A payment system at 1,000 TPS peak is, by raw-scale standards, *small*. **The challenge here is not throughput — it is correctness and durability.** Say that sentence in the interview; it reframes the whole problem. You are not fighting to scale a firehose. You are fighting to make sure not one drop is ever lost or duplicated.

**Storage — the ledger grows forever and is never deleted.** It is the legal record.

```
Each payment produces multiple immutable ledger entries.
Say ~4 ledger rows per payment (customer debit, merchant credit, fee lines).

Rows per day   = 10M payments × 4 entries    = 40M rows/day
Bytes per row  ≈ 300 B (ids, amounts, account refs, timestamps, metadata)

Per day        = 40M × 300 B                 = 12 GB/day
Per year       = 12 GB × 365                 ≈ 4.4 TB/year
```

A few terabytes a year is *trivial* to store. The point is not the size — it's that **you never delete a single row.** No TTL, no archival deletion, no "clean up old data." Regulators and auditors can demand any transaction from years ago. The ledger is append-only and permanent. Cold rows move to cheaper storage; they never disappear.

---

### 3. The money-movement fundamentals — DOUBLE-ENTRY BOOKKEEPING

This is the heart of the doc and the single thing that separates a real answer from a naive one. Learn it from first principles.

#### The naive (wrong) way

The instinct everyone has:

```sql
-- ❌ THE CLASSIC WRONG ANSWER. Never do this for money.
UPDATE accounts SET balance = balance - 50 WHERE id = 'customer';
UPDATE accounts SET balance = balance + 50 WHERE id = 'merchant';
```

Three fatal flaws:

1. **It destroys history.** After the `UPDATE`, the old balance is gone. You cannot answer "why is this balance what it is?" or "what changed on March 3rd?" There is no audit trail, only a final number.
2. **It races.** Two concurrent updates to the same balance can lose a write (a classic lost-update anomaly — recall [66 — Database Transactions and Isolation](./66-database-transactions-and-isolation.md)). Now the number is simply *wrong*, silently.
3. **You cannot detect corruption.** If a bug adds $50 to the merchant but crashes before subtracting it from the customer, you've *created* $50 out of nothing, and nothing in the system will ever tell you.

#### The correct way: append two balanced entries

**Double-entry bookkeeping** — the technique banks have used for 500 years — says: money never appears or disappears, it only *moves*. So every transaction is recorded as **at least two immutable entries**: a **debit** from one account and a matching **credit** to another. You **never** update a balance. You **append** entries, and a balance is **derived** by summing them.

The governing law, the invariant that makes the whole thing trustworthy:

> **The sum of every entry in the ledger must always equal zero.**

Debits and credits are signed so they cancel. If money left one account, it arrived at another, and the two lines sum to nothing. Across the *entire* ledger — millions of rows — the total is always exactly zero. If it ever isn't, a transaction is broken, and you can *detect it mechanically*.

**Why this is non-negotiable for money:**

- **It's auditable.** Every change has a paired origin. You can trace any dollar to the exact event that moved it, and every movement names both where it came from and where it went.
- **It's immutable.** The ledger is append-only, so history can never be silently altered. Correcting a mistake means appending a *reversing* entry, which itself is a permanent, visible record. (This is exactly the discipline of **event sourcing**, topic 68 — the state is the fold over an append-only log of events, and you never mutate the past.)
- **The zero-sum invariant lets you detect corruption.** You can run a query at any moment — `SELECT SUM(amount) FROM ledger_entries` — and it must return `0`. If it returns anything else, your books don't balance and you know, immediately, that something is wrong. That property is your smoke alarm.

#### The ledger table

```sql
CREATE TABLE ledger_entries (
    entry_id        BIGSERIAL PRIMARY KEY,
    transaction_id  UUID        NOT NULL,   -- groups the paired entries of ONE event
    account_id      VARCHAR     NOT NULL,   -- who this line touches
    direction       VARCHAR     NOT NULL,   -- 'DEBIT' or 'CREDIT'
    amount          BIGINT      NOT NULL,   -- ALWAYS in minor units (cents). Never floats.
    currency        CHAR(3)     NOT NULL,   -- 'USD'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
    -- NOTE: no 'balance' column, and there is NO UPDATE or DELETE on this table, ever.
);
CREATE INDEX ON ledger_entries (account_id);
CREATE INDEX ON ledger_entries (transaction_id);
```

Two rules encoded here that you should say aloud:

- **Amounts are integers in minor units (cents), never floating point.** `0.1 + 0.2 !== 0.3` in floating point; for money that error is a defect. Store `5000`, meaning $50.00, as a `BIGINT`.
- **There is no `balance` column and no `UPDATE`/`DELETE`.** The table is append-only. A balance is a `SUM`.

#### A worked example: a $50 payment

Customer pays a merchant $50. The processor takes a $1.50 fee. As balanced double entries under one `transaction_id`:

| transaction_id | account_id | direction | amount (cents) |
|---|---|---|---|
| `txn_abc` | `customer:alice` | DEBIT | -5000 |
| `txn_abc` | `merchant:store` | CREDIT | +4850 |
| `txn_abc` | `platform:fees` | CREDIT | +150 |

Check the invariant: `-5000 + 4850 + 150 = 0`. **The books balance.** $50 left Alice; $48.50 reached the merchant; $1.50 went to fees. Nothing was created, nothing was lost.

To find the merchant's balance, you never read a field — you sum:

```sql
SELECT SUM(amount) AS balance
FROM ledger_entries
WHERE account_id = 'merchant:store';
```

A **refund** is not an `UPDATE` reversing the row — it's a *new* balanced transaction that moves the money back, leaving the original entries permanently in place. History accumulates; it is never rewritten. That permanence is the whole point.

---

### 4. Idempotency — mandatory, not optional

Recall [85 — Idempotency](./85-idempotency.md): an operation is idempotent if doing it twice has the same effect as doing it once. For payments this is not a nice-to-have — it is the difference between charging someone once and charging them twice.

**The payment horror story, step by step:**

```
1. The user taps "Pay $50".
2. Your server processes it, charges the card... and then the network TIMES OUT
   before the success response reaches the phone.
3. The user's app sees no response. It assumes failure. It RETRIES.
4. Without protection: your server charges the card a SECOND time. The customer
   just paid $100 for a $50 order. You have stolen $50.
```

The retry is not a bug — retries are *correct* behavior on an unreliable network. The bug is a server that treats the retry as a new payment. The fix is the **idempotency key**.

**The solution, in full:**

1. The **client generates a unique key per payment attempt** (a UUID) and sends it in a header (`Idempotency-Key: idem_abc123`) with the *first* request **and every retry** of that same attempt.
2. The server uses a **UNIQUE constraint** on that key. The **first** request wins the insert, does the real work, and stores its result. Every **retry** hits the unique-constraint collision, so instead of charging again, the server **replays the stored result**.

```sql
CREATE TABLE idempotency_keys (
    idem_key     VARCHAR PRIMARY KEY,     -- the UNIQUE constraint IS the lock
    status       VARCHAR NOT NULL,        -- 'IN_PROGRESS' | 'COMPLETED'
    response     JSONB,                   -- the stored result to replay
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```js
// The idempotent charge handler — the single most important piece of code here.
async function handleCharge(req, res) {
  const idemKey = req.headers['idempotency-key'];
  if (!idemKey) return res.status(400).json({ error: 'Idempotency-Key required' });

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Try to CLAIM the key. The UNIQUE PK makes this atomic across all servers.
    // ON CONFLICT DO NOTHING means: if the key already exists, insert 0 rows.
    const claim = await client.query(
      `INSERT INTO idempotency_keys (idem_key, status)
       VALUES ($1, 'IN_PROGRESS')
       ON CONFLICT (idem_key) DO NOTHING
       RETURNING idem_key`,
      [idemKey]
    );

    if (claim.rowCount === 0) {
      // We did NOT win the insert — this is a retry (or a concurrent duplicate).
      await client.query('ROLLBACK');
      const prior = await pool.query(
        `SELECT status, response FROM idempotency_keys WHERE idem_key = $1`,
        [idemKey]
      );
      if (prior.rows[0].status === 'COMPLETED') {
        // REPLAY the original result. We do NOT charge again.
        return res.status(200).json(prior.rows[0].response);
      }
      // Original request is still IN_PROGRESS — tell the client to back off and retry.
      return res.status(409).json({ error: 'in_progress, retry shortly' });
    }

    // We WON the insert → we are the one true processor for this key.
    const result = await processPaymentOnce(client, req.body); // charges + writes ledger

    await client.query(
      `UPDATE idempotency_keys SET status = 'COMPLETED', response = $2 WHERE idem_key = $1`,
      [idemKey, result]
    );
    await client.query('COMMIT');
    return res.status(200).json(result);
  } catch (err) {
    await client.query('ROLLBACK');
    return res.status(500).json({ error: 'processing_failed' });
  } finally {
    client.release();
  }
}
```

**The rule that catches people: EVERY layer must be idempotent, not just the API.**

- Your **API** dedupes with the idempotency key (above).
- Your **queue consumer** must tolerate the same message delivered twice (queues are at-least-once — see step 6). So the consumer also checks "have I already processed this event?"
- Your **call to the payment processor** must carry *its own* idempotency key, so that if *you* retry the provider call, *they* don't double-charge either. **This is exactly how Stripe's API works** — every Stripe request accepts an `Idempotency-Key`, and Stripe guarantees the same key never charges twice.

Idempotency is not one checkbox. It is a property you maintain at *every hop where a retry can happen*, which — in a payment system — is all of them.

---

### 5. The payment flow as a state machine and a SAGA

Recall [77 — Distributed Transactions](./77-distributed-transactions.md). A payment is not one atomic action; it's a *lifecycle* that touches multiple systems.

#### The state machine

A payment moves through well-defined states, and only certain transitions are legal:

```
                 ┌──────────────────────────────────────────────┐
                 │                                              │
   ┌───────────┐ │  authorize   ┌──────────┐  capture  ┌─────────┐  settle  ┌───────────┐
   │ INITIATED ├─┴─────────────▶│ PENDING  ├──────────▶│CAPTURED ├─────────▶│ COMPLETED │
   └─────┬─────┘   (hold funds) │(authed)  │ (take it) └────┬────┘          └─────┬─────┘
         │                      └────┬─────┘                │                     │
         │ decline / error           │ void / expire        │ dispute             │ refund
         ▼                           ▼                       ▼                     ▼
   ┌───────────┐              ┌───────────┐           ┌───────────┐        ┌───────────┐
   │  FAILED   │              │  FAILED   │           │ REFUNDED  │◀───────┤ REFUNDED  │
   └───────────┘              └───────────┘           └───────────┘        └───────────┘
```

- **INITIATED** — we've created the payment intent; nothing has moved.
- **PENDING (authorized)** — the card issuer has placed a *hold* on the funds (an *authorization*), guaranteeing the money is there, but hasn't moved it yet. Auths expire (typically ~7 days).
- **CAPTURED (settled)** — we *capture* the authorized hold; the money is now actually being pulled. (Splitting auth from capture lets you charge only when you ship.)
- **COMPLETED** — funds have settled to the merchant.
- **FAILED** — declined, expired, or errored. No money moved (or the hold was released).
- **REFUNDED** — money is sent back. Reachable from CAPTURED/COMPLETED.

Storing the state explicitly (in a `payments` table) is what makes the flow **resumable** after a crash — see failure modes.

#### Why it can't be one ACID transaction — and the saga

Charging the external provider, writing your ledger, and notifying the merchant span **three different systems**. A single database `BEGIN...COMMIT` cannot wrap a call to Stripe's servers — you can't roll back a network call to another company's API. So you use a **saga**: a sequence of local steps, each with a **compensating action** that undoes it if a later step fails (recall [77]).

```
   ORCHESTRATED SAGA — charge a customer, credit a merchant
   ────────────────────────────────────────────────────────
   Step 1: AUTHORIZE with provider        compensation → VOID the authorization
   Step 2: WRITE ledger entries (local)   compensation → append REVERSING entries
   Step 3: CAPTURE with provider          compensation → REFUND (money already moved!)
   Step 4: Notify merchant                compensation → (best-effort, retriable)

   If step 3 succeeds but step 4's system is down, we RETRY step 4 — we do NOT
   refund, because the money legitimately moved. Compensation is per-step and
   depends on whether money has actually moved yet.
```

**The critical insight about compensating money:** once the money has actually moved (after capture), you **cannot "roll back."** There is no undo. The only compensation for a completed money movement is a **REFUND** — a *new* forward transaction that moves the money back. That's why:

- **Irreversible steps are ordered as late and as carefully as possible.** You authorize (a reversible hold) early; you capture (the irreversible pull) as late as you safely can, so that most failures happen *before* money moves and can be cleanly voided.
- **Compensation before capture is a VOID** (release the hold — cheap, clean, no money moved). **Compensation after capture is a REFUND** (a real reversing money movement, which itself can fail and must be retried).

Say this in the interview: *"A saga's compensation for money is not a rollback — it's a refund, because you can't un-move money. So I order the irreversible step last and make everything before it a reversible hold."*

---

### 6. Asynchronous processing and the outbox pattern

Payments involve **slow external calls** — an authorization round-trip to a card network can take seconds. You cannot hold a synchronous HTTP request open through all of that reliably, and you cannot let a provider's slowness take down your API. So much of the flow runs **asynchronously via a queue** (recall [67 — Message Queues](./67-message-queues.md)): the API validates the request, writes the intent durably, drops an event on a queue, and returns fast; workers pull the event and drive the saga.

But this introduces a subtle, dangerous bug.

#### The dual-write problem

You need to do two things: **update your database** (write the ledger/intent) **and publish an event to the queue**. These are two separate systems, so they cannot share one transaction. Whatever order you pick, a crash in the gap corrupts the system:

```
   Write DB, then publish:   crash after DB write → ledger says charged,
                             but NO event was published → merchant never notified,
                             downstream never runs. Money recorded, nothing happens.

   Publish, then write DB:   crash after publish → event fired, consumers act on it,
                             but the DB write never landed → we notified a merchant
                             about a payment that isn't in our ledger. Phantom money.
```

Either way you've violated the core promise: **you never charge without recording, and never record without the downstream effect.**

#### The fix: the OUTBOX pattern

Write the event to an **outbox table in the SAME local transaction** as the ledger entry. Since both are rows in *your* database, one `COMMIT` makes them atomic — either both land or neither does. A separate **relay** process then reads unpublished outbox rows and pushes them to the queue, marking them sent. (This is again the event-sourcing discipline of topic 68: the durable log of events *is* the DB write.)

```
   ┌─────────────────── ONE ATOMIC DB TRANSACTION ───────────────────┐
   │  INSERT INTO ledger_entries (...)   -- the money record         │
   │  INSERT INTO outbox (event, status='PENDING')  -- the event     │
   │  COMMIT   ← both, or neither. No gap. No dual write.            │
   └────────────────────────────────┬────────────────────────────────┘
                                     │
                     ┌───────────────▼────────────────┐
                     │  OUTBOX RELAY (poll or CDC)     │
                     │  SELECT * FROM outbox            │
                     │    WHERE status='PENDING'        │
                     │  → publish to Kafka/SQS          │
                     │  → UPDATE status='SENT'          │
                     └───────────────┬─────────────────┘
                                     ▼
                            ┌─────────────────┐
                            │  MESSAGE QUEUE  │──▶ saga workers, notifier, ...
                            └─────────────────┘
```

The relay is **at-least-once** — it might publish a row, crash before marking it `SENT`, and publish it again on restart. That's fine, *because every consumer is idempotent* (section 4). This guarantees you **never charge someone without recording it, and never record without eventually driving the downstream effect.** The outbox closes the exact gap where money leaks.

---

### 7. Reconciliation

Everything above — idempotency, double-entry, the saga, the outbox — is defense. **Reconciliation is the safety net that catches whatever slips through anyway.** It is the single feature that turns "we think our books are right" into "we can *prove* our books are right," and it is a hallmark of a mature answer.

**How it works:** periodically — typically **nightly** — you fetch the payment provider's **settlement report** (a file/API listing every transaction *they* believe happened and the money they moved) and compare it, **transaction by transaction**, against **your** ledger.

```
   FOR EACH transaction in the provider's settlement report:
     find the matching entry in OUR ledger by provider_ref
       ├─ present in both, amounts match   → ✅ reconciled, tick it off
       ├─ in provider report, NOT in ours  → ⚠️ they charged, we didn't record it
       ├─ in our ledger, NOT in theirs      → ⚠️ we recorded, they never charged
       └─ present in both, amounts DIFFER   → ⚠️ a discrepancy (fee, FX, partial)
   Anything not a clean ✅ is flagged into an EXCEPTIONS queue for a human.
```

**Why this is essential — and why you can't skip it even with all the defenses above:**

- **Distributed systems drift.** A webhook gets lost. A retry double-fires in a way a bug failed to dedupe. A provider reverses a charge you never heard about. Over millions of transactions, *something* will diverge, no matter how careful the happy path is.
- **It's the only end-to-end check against an independent source of truth.** Every other safeguard checks the system against *itself*. Reconciliation checks it against the *provider's* books — an outside witness. That's what makes it trustworthy.
- **It bounds your exposure.** A discrepancy that would otherwise silently compound gets caught within 24 hours, quantified, and escalated, instead of being discovered by an auditor two years later.

Reconciliation is how you **catch the money errors that slip through despite everything else.** A payment system without it is a system that *hopes* it's correct. A payment system with it is one that *checks*.

---

### 8. Deep dives

#### (a) Handling provider webhooks

Payment confirmation is **asynchronous** and arrives from the provider as a **webhook** (an HTTP callback to your server) — e.g., Stripe POSTs `charge.captured` to your `/webhooks/stripe` endpoint. Three properties make webhooks treacherous, and your handler must survive all three:

- **They can be duplicated.** The provider retries if your endpoint is slow to `200`. So webhook handling **must be idempotent** — key on the provider's event id and process each once.
- **They can be delayed.** A webhook may land seconds or minutes after the event.
- **They can arrive out of order.** You might receive the `captured` event **before** the `authorized` one. Your handler must **tolerate that** — either buffer/reorder by state, or make each handler safe regardless of arrival order (e.g., "captured" upserts the payment into CAPTURED even if no prior "authorized" row exists). Never assume webhooks arrive in causal order.

Also: **verify the webhook signature.** The provider signs the payload; you check it, so an attacker can't POST a fake "you got paid" event.

#### (b) Consistency: why you accept latency for correctness

Unlike the feed systems, which chose availability and eventual consistency, **the ledger write is strongly consistent and synchronous.** You will happily make a payment take an extra second to guarantee the money record is durable and correct before you tell anyone "paid." This is the deliberate inverse of the earlier trade-offs in the course: *there, latency won; here, correctness wins.* The customer would far rather wait two seconds than be charged twice. State this trade explicitly — it shows you know the CAP/PACELC dial exists and that you're turning it the *opposite* way on purpose.

#### (c) Fraud detection (conceptually)

Before you capture, a **real-time fraud scoring** step runs inline: a service scores the transaction on signals (amount, velocity, device fingerprint, geo mismatch, card-testing patterns) and returns approve / review / decline. High-risk transactions are held for manual review or blocked outright. You don't need to build the ML in an interview — you need to *place the step in the flow* (between authorize and capture) and note it must be fast enough not to wreck latency.

#### (d) PCI-DSS and tokenization (high level)

As said in requirements: **you store a token, never the card number.** The card's PAN goes browser → processor directly (via the processor's hosted fields/SDK, so it never touches your servers). The processor returns a **token** representing the card; you store `tok_...` and use it for future charges. Because the PAN never enters your systems, you drop out of the most demanding PCI scope. **Tokenization is the mechanism; staying out of PCI scope is the goal.**

#### (e) Exactly-once effect via at-least-once + idempotency

Say this plainly, because it's a favorite trap: **true exactly-once *delivery* is a myth** in distributed systems. Networks fail, acks get lost, and any real transport is at-least-once (it will occasionally deliver twice) or at-most-once (it will occasionally lose one). You cannot buy exactly-once delivery. What you *can* build is **exactly-once *effect*: at-least-once delivery + idempotent processing.** The message may arrive twice; the idempotency key ensures the money moves exactly once (recall [85]). Every "exactly once" payment system on Earth is really "deliver as many times as you need, process idempotently." Knowing the difference between *delivery* and *effect* is the signal.

---

### 9. Failure modes

Trace what happens when each piece breaks — and show that **no failure loses or duplicates money.**

**The payment provider is down.** You never lose the *intent*. The payment request is written durably (INITIATED) and the saga event sits on the queue; workers **retry with backoff** until the provider recovers. The customer sees "processing," not an error that makes them re-submit (which would risk a double attempt). Queue-and-retry means a provider outage delays payments — it never drops them.

**A webhook is lost.** The provider captured the charge but its confirmation webhook never reached you, so your ledger is momentarily out of sync. **Reconciliation catches it:** that night, the transaction appears in the provider's settlement report but not (fully) in your ledger, gets flagged, and is corrected. The lost webhook becomes a next-day discrepancy, not a permanent hole. (Belt and suspenders: you can also poll the provider for status on stuck payments.)

**Your own service crashes mid-flow.** Because the **saga state is persisted** (the `payments` row records exactly which step completed) and the **ledger + outbox were written in one transaction**, on restart a recovery worker finds payments stuck mid-saga and **resumes** them from the last durable step. And because **every step is idempotent**, re-running a step that had actually succeeded (you just crashed before recording it) is safe — it replays the stored result instead of charging again. Crash-during-flow collapses into "resume from persisted state; idempotency makes the resume harmless."

The unifying theme across all three: **the durable record is written first and the effect is made idempotent, so every failure degrades into a delay or a next-day reconciliation — never a lost or duplicated dollar.**

---

## Visual / Diagram description

The diagrams to memorize and reproduce on a whiteboard:

1. **The ledger table + worked $50 example (section 3)** — the three balanced rows summing to zero. If you can draw this and explain "no balance column, append-only, sum to derive, sum-of-all-entries = 0," you've demonstrated the core competency.
2. **The payment state machine (section 5)** — INITIATED → PENDING → CAPTURED → COMPLETED with FAILED and REFUNDED branches. It shows you know auth-vs-capture and that refund is a forward transition, not a rewind.
3. **The saga + outbox diagram (sections 5–6)** — the ordered steps with per-step compensations, and the one-atomic-transaction outbox that closes the dual-write gap. This is where the distributed-systems maturity shows.

The one thing every diagram must convey is the **asymmetry between reversible and irreversible steps**: holds can be voided cheaply; captured money can only be *refunded*, never rolled back. Almost every subtle bug in payments comes from treating an irreversible money movement as if it were reversible.

---

## Node.js implementation

The two pieces that matter most: the **double-entry ledger write** (atomic, balanced, append-only, with the outbox) and how it plugs into the **idempotent charge handler** from section 4.

```js
// ledger.js — the double-entry write. This is the beating heart of the system.
import { pool } from './db.js';

/**
 * Record a balanced set of ledger entries in ONE atomic transaction,
 * together with an outbox event. Enforces the zero-sum invariant BEFORE
 * writing — if the entries don't balance, we refuse to persist anything.
 */
export async function recordTransaction({ transactionId, entries, event }) {
  // 1. Enforce the invariant IN CODE, before touching the DB.
  //    Every transaction's entries must sum to exactly zero.
  const sum = entries.reduce((acc, e) => acc + e.amount, 0);
  if (sum !== 0) {
    // This is a programming error and must never reach production data.
    throw new Error(`Unbalanced transaction ${transactionId}: entries sum to ${sum}, not 0`);
  }

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // 2. APPEND the entries. Never UPDATE, never DELETE. Amounts are integer cents.
    for (const e of entries) {
      await client.query(
        `INSERT INTO ledger_entries
           (transaction_id, account_id, direction, amount, currency)
         VALUES ($1, $2, $3, $4, $5)`,
        [transactionId, e.account_id, e.amount < 0 ? 'DEBIT' : 'CREDIT', e.amount, e.currency]
      );
    }

    // 3. OUTBOX: write the event in the SAME transaction as the ledger.
    //    This closes the dual-write gap — both land, or neither does.
    await client.query(
      `INSERT INTO outbox (event_type, payload, status)
       VALUES ($1, $2, 'PENDING')`,
      [event.type, JSON.stringify(event.payload)]
    );

    await client.query('COMMIT'); // atomic: ledger + event together
  } catch (err) {
    await client.query('ROLLBACK'); // nothing partial ever survives
    throw err;
  } finally {
    client.release();
  }
}

/**
 * Derive a balance by SUMMING every entry — never by reading a stored field.
 */
export async function getBalance(accountId) {
  const { rows } = await pool.query(
    `SELECT COALESCE(SUM(amount), 0) AS balance
       FROM ledger_entries WHERE account_id = $1`,
    [accountId]
  );
  return Number(rows[0].balance); // integer cents
}
```

```js
// process-payment.js — the "do it exactly once" body called by the idempotent handler.
// (handleCharge in section 4 has already guaranteed this runs at most once per key.)
import { randomUUID } from 'node:crypto';
import { recordTransaction } from './ledger.js';
import { stripe } from './provider.js';

export async function processPaymentOnce(dbClient, { customerId, merchantId, amount, cardToken, idemKey }) {
  const transactionId = randomUUID();
  const feeCents = Math.round(amount * 0.029) + 30; // 2.9% + 30¢, integer math

  // 1. Call the provider — pass OUR idempotency key so THEY never double-charge either.
  //    Order matters: this reversible-ish external effect happens, THEN we record.
  const charge = await stripe.charges.create(
    { amount, currency: 'usd', source: cardToken, capture: true },
    { idempotencyKey: idemKey } // exactly how Stripe's API is meant to be used
  );

  if (charge.status !== 'succeeded') {
    return { status: 'FAILED', reason: charge.failure_message };
  }

  // 2. Record the money movement as BALANCED double entries (+ outbox event).
  //    Customer debited the full amount; merchant credited net; platform credited the fee.
  await recordTransaction({
    transactionId,
    entries: [
      { account_id: `customer:${customerId}`, amount: -amount,           currency: 'USD' },
      { account_id: `merchant:${merchantId}`, amount:  amount - feeCents, currency: 'USD' },
      { account_id: 'platform:fees',          amount:  feeCents,          currency: 'USD' },
    ], // sums to zero: -amount + (amount - fee) + fee = 0
    event: {
      type: 'payment.captured',
      payload: { transactionId, merchantId, amount, providerRef: charge.id },
    },
  });

  return { status: 'CAPTURED', transactionId, providerRef: charge.id };
}
```

Notice how the two guarantees compose: `handleCharge` (section 4) ensures `processPaymentOnce` runs **at most once** per idempotency key; `recordTransaction` ensures the ledger is **always balanced and append-only** and that the downstream event ships **atomically** with the money record. Idempotency protects against *duplication*; double-entry + outbox protect against *loss and phantom effects*. Together they are "no money lost, none created, none double-counted."

---

## Real world examples

### Stripe

Stripe is the reference implementation of almost everything in this doc. Its public API accepts an **`Idempotency-Key`** on every mutating request and guarantees the same key never charges twice — exactly the mechanism in section 4. It splits **authorization** from **capture** (`capture: false` places a hold you settle later), exactly the state machine in section 5. It delivers confirmations via **signed webhooks** that you must handle idempotently and out-of-order. And via **Stripe Elements / hosted fields**, the card number goes browser → Stripe directly and you receive a **token**, keeping you out of PCI scope. If you internalize one real system for this interview, make it Stripe.

### Uber's ledger (and the "money is different" doctrine)

Uber, which moves money between riders, drivers, and Uber itself across dozens of countries and currencies, built a dedicated double-entry ledger platform (internally "LedgerStore") precisely because generic databases with mutable balance columns could not give them the **immutability and auditability** money demands. The lesson companies converge on independently: **money gets its own append-only, double-entry system**, separate from the mutable operational data. You don't bolt correctness onto a CRUD app; you build a ledger.

### PayPal / traditional processors and reconciliation

Every serious payment operator runs **daily reconciliation** against acquiring banks and card-network settlement files. This isn't optional hygiene — it's how discrepancies between "what we think happened" and "what the money network says happened" are caught and resolved within a day. The existence of large "revenue operations" and "payment ops" teams whose whole job is chasing reconciliation exceptions is proof that section 7 is not academic: at scale, *something always drifts*, and reconciliation is the machine that finds it.

---

## Trade-offs

**Balance representation**

| Choice | You gain | You give up |
|---|---|---|
| Append-only double-entry ledger | Full audit trail, immutability, mechanical corruption detection (sum = 0), safe concurrency | More storage (never delete), balance is a `SUM` not a lookup, more upfront design |
| Mutable `balance` column (`UPDATE`) | Trivial to read a balance; less data | No history, lost-update races, undetectable corruption — **disqualifying for money** |

**Consistency posture**

| Choice | You gain | You give up |
|---|---|---|
| Strong consistency, synchronous ledger write | Correctness, durability, "charged means recorded" | Some latency; the ledger DB is on the critical path |
| Eventual consistency (as in the feed systems) | Lower latency, higher availability | **Temporarily wrong money — unacceptable here.** The whole point is *not* to make this trade |

**Delivery semantics**

| Choice | You gain | You give up |
|---|---|---|
| At-least-once queue + idempotent consumers | Exactly-once *effect*, no lost events, survives retries | Must build idempotency at every hop; occasional duplicate deliveries to absorb |
| "Exactly-once delivery" | (nothing — it's a myth) | Chasing it wastes effort; the transport can't guarantee it |

**Card handling**

| Choice | You gain | You give up |
|---|---|---|
| Tokenize via a processor (store `tok_...`) | Out of PCI scope, far less liability, faster to build | Dependence on a processor; fees; some flexibility |
| Store the PAN yourself | Full control | Full PCI-DSS scope, audits, breach liability — almost never worth it |

**Rule of thumb:** **Record before you rely; make every step idempotent; reconcile against an outside witness.** The ledger is the truth, the idempotency key is the guard against duplication, and reconciliation is the proof. Design every path so that a failure becomes a *delay* or a *next-day discrepancy* — never a lost or duplicated dollar.

---

## Common interview questions on this topic

### Q1: "How do you make sure a customer is never charged twice?"

**Hint:** Idempotency keys. The client generates a unique key per payment *attempt* and sends it with the first request and every retry. The server enforces a UNIQUE constraint: the first request wins the insert and does the work; every retry hits the conflict and **replays the stored result** instead of charging again. And it must hold at *every* hop — the API, the queue consumer, and the call to the processor (which itself gets its own idempotency key). Retries are correct behavior on a flaky network; the server's job is to make the second charge a no-op.

### Q2: "How do you store balances? Wouldn't you just UPDATE a balance column?"

**Hint:** No — that's the classic wrong answer. It destroys history, races under concurrency (lost updates), and hides corruption. Use **double-entry bookkeeping**: every transaction is at least two immutable, balanced entries (a debit and a matching credit) appended to a ledger, and the sum of *all* entries is always zero. A balance is **derived** by summing an account's entries, never stored. Immutability gives you audit; the zero-sum invariant lets you *detect* corruption mechanically.

### Q3: "Charging the card, writing your DB, and notifying the merchant are three systems. How do you keep them consistent without one big transaction?"

**Hint:** You can't wrap an external API call in an ACID transaction, so use a **saga**: ordered steps, each with a compensating action. Authorize (a reversible hold) early, capture (the irreversible pull) late. Compensation *before* capture is a **void**; *after* capture it's a **refund**, because you can't un-move money. And for the DB-plus-queue dual write, use the **outbox pattern**: write the event to an outbox table in the *same* transaction as the ledger, then a relay publishes it — so you never record without eventually emitting the event, or vice versa.

### Q4: "The payment provider confirms via webhooks. What can go wrong?"

**Hint:** Webhooks can be **duplicated, delayed, and out of order** — you might get `captured` before `authorized`. So handlers must be **idempotent** (key on the provider event id, process once) and must **tolerate any arrival order** (don't assume causal ordering; upsert into the right state). Also verify the signature so attackers can't forge a "you got paid" event. And if a webhook is *lost* entirely, **reconciliation** the next day catches the divergence.

### Q5: "How do you know your books are actually right?"

**Hint:** **Reconciliation.** Nightly, fetch the provider's settlement report and compare it transaction-by-transaction against your ledger. Charges they have and you don't, charges you have and they don't, and amount mismatches all get flagged into an exceptions queue for a human. It's the only check against an *independent* source of truth — every other safeguard checks the system against itself. Distributed systems drift; reconciliation is how you catch the leaks that survive idempotency, double-entry, and the outbox.

### Q6: "Isn't this just exactly-once delivery?"

**Hint:** No — **exactly-once *delivery* is a myth.** Real transports are at-least-once. What you build is exactly-once **effect**: at-least-once delivery + idempotent processing. The message may arrive twice; the idempotency key ensures the money moves once. Knowing the difference between *delivery* and *effect* is the whole point.

### Q7: "Do you store credit card numbers?"

**Hint:** No, deliberately. The card number (PAN) goes from the browser straight to the processor (Stripe/Adyen) via hosted fields; the processor returns a **token**; you store the token. Because the PAN never touches your servers, you stay out of the most demanding **PCI-DSS** scope. Tokenization is the mechanism; avoiding PCI liability is the reason.

---

## Practice exercise

### Design the ledger, then try to lose money

**Part 1 — The ledger (15 min).** On paper, write the `ledger_entries` schema from memory (no `balance` column, no `UPDATE`). Then write out the balanced entries for these three events, and after each, verify the entries sum to zero:
1. A customer pays a merchant **$120**, processor fee **2.9% + 30¢**.
2. The customer is later **refunded $40** (partial). Remember: this is a *new* transaction, not an edit of the first.
3. Compute the merchant's balance by summing its entries. Show your arithmetic in integer cents.

**Part 2 — The state machine (10 min).** Draw the payment state machine. Then answer: (a) A customer's card is authorized but the merchant never ships — what state does the payment end in, and what compensating action runs? (b) A capture *succeeds* but your merchant-notification service is down — do you refund? Why or why not?

**Part 3 — Break it (20 min).** For each scenario, state precisely whether money can be lost, created, or double-counted, and why (or why not). If your answer is "money is lost," you've moved a durable-write or idempotency step — go find it.
1. Your server charges the card, then crashes *before* writing the ledger.
2. Your server writes the ledger, then crashes *before* publishing the outbox event.
3. The queue delivers the same "capture" event to two workers simultaneously.
4. A webhook says `captured` arrives, but the `authorized` webhook for the same payment never does.
5. The provider's nightly report shows a $50 charge your ledger has no record of.

For scenario 5, write down the exact steps of what your reconciliation job does with it.

**Deliverable:** one page of balanced ledger entries, one state diagram, one page of failure prose. The failure prose is the real exercise — if every scenario ends in "no money lost or duplicated," you've internalized the design.

---

## Quick reference cheat sheet

- **Correctness beats everything.** No double charges, no lost money, every cent accounted for. This is the *anti-thesis* of the eventually-consistent systems earlier in the course — you trade latency and even some availability to buy correctness, and you say so.
- **Payment TPS is modest** (~100 avg, ~1,000 peak). The challenge is **correctness and durability, not scale.**
- **The ledger grows forever and is never deleted.** It's the legal, auditable record. A few TB/year — size is irrelevant, permanence is the point.
- **Double-entry bookkeeping is non-negotiable.** Every transaction = ≥2 balanced entries (debit + credit); the sum of *all* entries is always **zero**.
- **Never `UPDATE balance = balance + x`.** Append immutable entries; **derive** a balance by summing. Append-only = audit + corruption detection (sum ≠ 0 means something's broken).
- **Money is integer minor units (cents), never floats.**
- **Idempotency is mandatory at every hop.** Client sends a unique key per attempt with every retry; server uses a UNIQUE constraint to process once and **replay** the stored result. This is how Stripe's API works.
- **The payment lifecycle is a state machine:** INITIATED → PENDING (authorized) → CAPTURED → COMPLETED, plus FAILED and REFUNDED. Authorize early (reversible hold), capture late (irreversible).
- **Multi-system flow = a SAGA with compensations.** Compensation before capture is a **void**; after capture it's a **refund** — you can't roll back money, you can only move it back.
- **Async via a queue; the OUTBOX pattern fixes the dual-write problem.** Write the event to an outbox table in the *same* transaction as the ledger, then relay it. Never charge without recording, never record without the downstream effect.
- **Webhooks are duplicated, delayed, and out of order.** Handle idempotently, tolerate any arrival order, verify signatures.
- **Reconciliation is the safety net.** Nightly, compare the provider's settlement report line-by-line against your ledger; flag every mismatch. It's the only check against an outside witness.
- **Exactly-once *delivery* is a myth; you build exactly-once *effect*** via at-least-once delivery + idempotency.
- **Stay out of PCI scope: tokenize.** The card number goes browser → processor; you store a token, never the PAN.
- **Every failure degrades into a delay or a next-day discrepancy — never a lost or duplicated dollar.** Provider down → queue and retry. Webhook lost → reconciliation catches it. Service crash → persisted saga state resumes, idempotency makes the resume safe.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Related** | [85 — Idempotency](./85-idempotency.md) — the guarantee that a retried payment charges exactly once; the mechanism behind every "process once, replay the result" in this doc |
| **Related** | [77 — Distributed Transactions](./77-distributed-transactions.md) — the saga and compensating actions that coordinate charging, ledger writes, and merchant notification across systems |
| **Related** | [66 — Database Transactions and Isolation](./66-database-transactions-and-isolation.md) — the ACID guarantees behind the atomic ledger+outbox write, and the lost-update anomaly that kills mutable balances |
| **Related** | [67 — Message Queues](./67-message-queues.md) — the async backbone: at-least-once delivery, the outbox relay, and why consumers must be idempotent |
| **Related** | [104 — HLD: Amazon](./104-hld-amazon.md) — checkout and order placement is the upstream caller of this payment system; where the "place order" flow hands off to "take the money" |
