# 03 — Payment Ledger
## Money that can never drift, against systems that lie

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## WHY THIS CASE STUDY EXISTS

Case studies 01 and 02 were about **throughput under contention**. This one is about **correctness under partial failure** — and it is a harder problem.

A flash sale that oversells by 2 units is an apology. A ledger that drifts by ₹2 is an audit finding, a regulatory issue, and a bug you may not detect for months. Meanwhile, every external system you depend on — payment gateways, banks, your own message queue — will time out, retry, deliver twice, and occasionally tell you something that is not true.

The design goal here is not speed. It is: **the books balance, always, no matter what fails.**

---

## 1. THE BRIEF

A payments platform for an e-commerce marketplace. You hold user wallet balances, settle to merchants, and integrate with an external gateway (Razorpay/Stripe).

> **12,000 transactions/sec at peak. 40M wallet accounts. 8 currencies.**
> **Regulatory retention: 7 years, immutable, auditable.**

Business rules:

1. **Money is never created or destroyed.** Every rupee that leaves one account arrives in another. `sum(all entries) = 0`, always.
2. **A user's balance may never go negative** (except for designated credit accounts with an explicit limit).
3. **Every operation is idempotent.** The gateway *will* send the same webhook 5 times. The mobile app *will* retry a timed-out request. Neither may double-charge.
4. **Nothing is ever updated or deleted.** A correction is a new, opposite entry — never an edit. Auditors need to see what was believed at each point in time.
5. **Reads of a balance must be fast** — shown on every app screen, 80,000 req/s.
6. **Reconciliation with the gateway must be possible daily**, and discrepancies must be detectable automatically.
7. **Multi-currency**: no implicit conversion, ever. An account is in exactly one currency.

Infrastructure:
- PostgreSQL 16, primary + 2 replicas, 32 vCPU / 256 GB
- 80 Node.js pods, PgBouncer transaction mode
- Kafka for downstream events
- Redis for balance caching

---

## 2. ACCESS PATTERN TABLE

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness tolerated |
|---|---|---|---|---|---|---|
| A | `GET /wallet/balance` | 80,000/s | 1 | 0 | 50 ms | **2 s** (display only) |
| B | `POST /transfer` — wallet → wallet | 4,000/s | 4 | 3 | 200 ms | **0** |
| C | `POST /payments` — initiate gateway payment | 3,000/s | 2 | 3 | 300 ms | 0 |
| D | Gateway webhook — payment captured/failed | 3,000/s | 4 | 4 | 500 ms | 0 |
| E | `POST /refund` | 400/s | 5 | 4 | 500 ms | 0 |
| F | Merchant settlement batch (nightly) | 200k accounts | 200k | 400k | n/a | 0 |
| G | `GET /wallet/statement` — last 50 txns | 9,000/s | 50 | 0 | 150 ms | 5 s (replica) |
| H | Daily reconciliation vs gateway | 1/day | 8M | ~0 | n/a | 0 |
| I | Auditor query — "state of account X on 2024-03-11" | rare | varies | 0 | n/a | 0 |

**What jumps out:**

- **A at 80,000/s reading a balance.** If balance is a column you `UPDATE`, that row is written 4,000/s and read 80,000/s — a textbook hot row. If balance is `SUM(entries)`, reading it is O(n) over a growing history. **Neither naive option works**, and resolving that tension is the core of the design.
- **B, C, D, E all mutate money and all can be retried.** Idempotency isn't a feature here; it's the substrate.
- **I requires that history is never overwritten.** That single requirement rules out `UPDATE balance SET ...` as the source of truth, permanently.

---

## 3. THE INVARIANTS

```
I1.  ZERO-SUM:  for every transaction T,  sum(entry.amount WHERE txn_id = T) = 0
     ── double-entry bookkeeping. Money moves; it is never created.

I2.  GLOBAL ZERO-SUM:  sum(entry.amount) over the entire ledger = 0
     ── follows from I1, and is the daily audit check.

I3.  NON-NEGATIVE:  for every non-credit account,
       sum(entry.amount WHERE account_id = A) >= 0
     ── at every instant, including between concurrent transfers.

I4.  IMMUTABILITY: no ledger row is ever UPDATEd or DELETEd.
     Corrections are new entries. History is append-only.

I5.  IDEMPOTENCY: an operation submitted N times with the same
     idempotency key produces exactly ONE set of ledger entries.

I6.  CURRENCY INTEGRITY: all entries within one transaction share
     one currency, and an entry's currency matches its account's currency.
```

**I1 and I3 pull in opposite directions.** I1 wants you to write entries and never look back. I3 requires knowing the *current sum* before you write — which is exactly the read-then-write pattern that races. Everything interesting in this design lives in that tension.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA

The design almost every team writes first. It is not stupid — it is what a balance *looks like*.

```sql
CREATE TABLE wallets (
  id           bigserial PRIMARY KEY,
  user_id      bigint NOT NULL UNIQUE,
  currency     char(3) NOT NULL,
  balance_paise bigint NOT NULL DEFAULT 0 CHECK (balance_paise >= 0),
  updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE transactions (
  id           bigserial PRIMARY KEY,
  from_wallet  bigint REFERENCES wallets(id),
  to_wallet    bigint REFERENCES wallets(id),
  amount_paise bigint NOT NULL CHECK (amount_paise > 0),
  status       text NOT NULL DEFAULT 'pending',
  created_at   timestamptz NOT NULL DEFAULT now()
);
```

```js
// ATTEMPT 1
async function transfer(fromId, toId, amount) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const from = await client.query(
      'SELECT balance_paise FROM wallets WHERE id=$1 FOR UPDATE', [fromId]);
    if (from.rows[0].balance_paise < amount) {
      await client.query('ROLLBACK');
      return { ok: false, reason: 'INSUFFICIENT_FUNDS' };
    }

    await client.query(
      'UPDATE wallets SET balance_paise = balance_paise - $1 WHERE id=$2', [amount, fromId]);
    await client.query(
      'UPDATE wallets SET balance_paise = balance_paise + $1 WHERE id=$2', [amount, toId]);
    await client.query(
      `INSERT INTO transactions (from_wallet,to_wallet,amount_paise,status)
       VALUES ($1,$2,$3,'completed')`, [fromId, toId, amount]);

    await client.query('COMMIT');
    return { ok: true };
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
```

---

## 5. WHERE IT BREAKS

### 5.1 Failure one: deadlocks, immediately

```
 User X sends ₹100 to User Y.    Locks: wallet 7, then wallet 12.
 User Y sends ₹50  to User X.    Locks: wallet 12, then wallet 7.

 t1  X: UPDATE wallet 7   (lock acquired)
 t2  Y: UPDATE wallet 12  (lock acquired)
 t3  X: UPDATE wallet 12  → waits for Y
 t4  Y: UPDATE wallet 7   → waits for X          ── DEADLOCK
```

```
ERROR:  deadlock detected
DETAIL:  Process 4102 waits for ShareLock on transaction 88214;
         blocked by process 4119.
```

In a marketplace, mutual transfers between the same pair of accounts are common (order + refund, escrow + release). Measured at 200 concurrent clients over 5,000 wallets: **3.1% deadlock rate.**

### 5.2 Failure two: the merchant hot row

A marketplace has a long tail of buyers and a short head of merchants. One large merchant receives 800 payments/second — all `UPDATE wallets SET balance = balance + $1 WHERE id = 9001`.

```
 Row lock on wallet 9001 held from the UPDATE until COMMIT.
 T = round trips + fsync ≈ 1.1 ms
 ⇒ ceiling ≈ 900 transfers/sec into ONE merchant wallet.

 Measured under load:
 tps = 870,  latency avg = 228 ms, and CLIMBING as the queue builds.
```

And the collateral damage: 800 waiters × one connection each exhausts the pool, so unrelated transfers stall too. **Exactly the failure from case study 01, but you cannot split a merchant's balance into "10,000 individual units" — a balance is not a countable set of interchangeable things.**

### 5.3 Failure three: the audit is impossible

```
 An auditor asks: "What was wallet 4471's balance on 2024-03-11 at 14:00?"

 The `wallets` table holds ONE number: the balance NOW.
 The old values were OVERWRITTEN. They are gone.

 You try to reconstruct it from `transactions`... and discover:
   • a bug in March credited 40 wallets without a transaction row
   • three transactions were manually "fixed" with an UPDATE
   • the fee deduction was applied as a balance UPDATE with no txn row at all

 sum(transactions) ≠ balance, and there is no way to know which is right.
```

This is not hypothetical. **It is the single most common failure mode of balance-as-a-column designs**, and it is unrecoverable — the information was destroyed at write time. Violates I4.

### 5.4 Failure four: the double-charge

```
 t=0     Mobile app POSTs /transfer. Request reaches the server.
 t=0.3s  Transaction commits. Money moved.
 t=0.4s  Response is lost (mobile network drops).
 t=3s    App retries the same request.
 t=3.3s  Transaction commits AGAIN. Money moved TWICE.

 There is no idempotency key. The server has no way to know these are
 the same intent. Violates I5.
```

At 4,000 transfers/sec with a 0.5% retry rate, that's **20 double-charges per second.**

### 5.5 Failure five: money vanishes into the gateway gap

```js
// The dual-write problem, in its purest form
await db.query('UPDATE wallets SET balance = balance - $1 WHERE id=$2', [amt, id]);
const payment = await gateway.charge({ amount: amt });   // ⚠ what if this times out?
await db.query('INSERT INTO transactions ...');
```

```
 SCENARIO: gateway.charge() TIMES OUT after 30 s.

 Did the charge happen?  YOU DO NOT KNOW.
   • It may have succeeded and the response was lost → user charged,
     no record → they will dispute it and you have no evidence.
   • It may have failed → user's wallet debited, no charge → you owe
     them money and don't know it.

 The naive code has already debited the wallet. Now what? Refund?
 What if the charge actually failed and you "refund" money that was
 never taken? You just created money. I1 violated.
```

**A timeout is not a failure. A timeout is "unknown."** Any design that treats it as failure will eventually create or destroy money.

### 5.6 Failure six: `SELECT balance` at 80,000/s

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT balance_paise FROM wallets WHERE id = 4471;
```
Fast — 0.05 ms. But that row is also being `UPDATE`d 800 times/sec by pattern B, so:
- Every read must walk an MVCC version chain of up to 800 versions.
- Autovacuum can't keep up with 800 dead tuples/sec on one page.
- `wallets` bloats to gigabytes for 40M small rows.

```sql
SELECT n_tup_upd, n_tup_hot_upd, n_dead_tup, pg_size_pretty(pg_relation_size('wallets'))
FROM pg_stat_user_tables WHERE relname='wallets';
```
```
 n_tup_upd  | n_tup_hot_upd | n_dead_tup | pg_size_pretty
------------+---------------+------------+----------------
  412000000 |      18000000 |   82000000 | 41 GB
```
40M wallets (should be ~4 GB) occupying **41 GB**, with a 4% HOT ratio because `balance_paise` is indexed for a reporting query someone added.

---

## 6. ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH

Two obvious fixes, both correct, both insufficient.

**(a) Deterministic lock ordering** kills the deadlocks (Topic 48):

```js
const [first, second] = [fromId, toId].sort((a, b) => a - b);
await client.query('SELECT id FROM wallets WHERE id IN ($1,$2) ORDER BY id FOR UPDATE',
                   [first, second]);
```

**(b) An idempotency key** kills the double-charge (Topic 52):

```sql
ALTER TABLE transactions ADD COLUMN idempotency_key uuid NOT NULL;
CREATE UNIQUE INDEX uq_txn_idem ON transactions (idempotency_key);
```

```js
const t = await client.query(
  `INSERT INTO transactions (from_wallet,to_wallet,amount_paise,idempotency_key)
   VALUES ($1,$2,$3,$4) ON CONFLICT (idempotency_key) DO NOTHING RETURNING id`,
  [fromId, toId, amount, key]);
if (t.rowCount === 0) return existingResult(key);   // replay — return the original
```

Measure:

```
 deadlock rate:     3.1%  →  0.01%
 double-charges:    20/s  →  0
 tps (spread load): 870   →  3,100
 tps (one hot merchant): 870 → 890      ← ⚠ UNCHANGED
```

**What remains broken, and why it can't be fixed by tuning:**

| Still broken | Why lock ordering / idempotency doesn't help |
|---|---|
| Merchant hot row (890 tps ceiling) | The contention is on *one row*, and a balance genuinely is one number. Ordering the locks doesn't reduce how long one is held. |
| Audit trail destroyed | `UPDATE balance` overwrites history by definition. No amount of keys or ordering recovers it. Violates I4 structurally. |
| Gateway timeout ambiguity | Idempotency on *your* side doesn't tell you what happened on *theirs*. You need a state machine and reconciliation. |
| `wallets` bloat | Still 412M updates on 40M rows. |

**The realisation:** the schema is wrong, not the code. A balance is not a *fact you store* — it is a *conclusion you derive*. Storing it as the source of truth is what makes all four of these unfixable.

---

## 7. THE PRODUCTION DESIGN

```
 PROBLEM                                SOLUTION                       TOPIC
 ──────────────────────────────────────────────────────────────────────────
 Balance overwrites history        →  APPEND-ONLY DOUBLE-ENTRY LEDGER   29,40
 Reading a derived balance is O(n) →  Periodic SNAPSHOTS + delta        55,56
 Merchant hot row                  →  Balance is never UPDATEd; entries
                                      are INSERTed → no contention      61
 I3 (non-negative) needs a read    →  Per-account advisory serialisation
                                      ONLY for debits, not credits      45
 Gateway ambiguity                 →  Explicit state machine + PENDING
                                      state + reconciliation            51,52
 Dual-write to Kafka               →  Transactional outbox              52
```

### 7.1 The schema

```sql
-- ═══════════════════════════════════════════════════════════════════
-- ACCOUNTS — the chart of accounts. Every party is an account,
-- INCLUDING your own bank, fee revenue, and the gateway float.
-- ═══════════════════════════════════════════════════════════════════
CREATE TYPE account_kind AS ENUM (
  'user_wallet',      -- liability: money you owe a user
  'merchant_payable', -- liability: money you owe a merchant
  'gateway_float',    -- asset: money sitting at the gateway
  'bank_settlement',  -- asset: your actual bank account
  'fee_revenue',      -- income
  'promo_liability',  -- liability: promotional credit issued
  'suspense'          -- ⚠ where unexplained money parks until resolved
);

CREATE TABLE accounts (
  id           bigserial     PRIMARY KEY,
  kind         account_kind  NOT NULL,
  owner_id     bigint        NULL,          -- user_id or merchant_id
  currency     char(3)       NOT NULL,
  allows_negative boolean    NOT NULL DEFAULT false,
  credit_limit_minor bigint  NOT NULL DEFAULT 0,
  created_at   timestamptz   NOT NULL DEFAULT now(),
  CHECK (currency ~ '^[A-Z]{3}$'),
  CHECK (credit_limit_minor >= 0),
  UNIQUE (kind, owner_id, currency)          -- one wallet per user per currency
);

-- ═══════════════════════════════════════════════════════════════════
-- TRANSACTIONS — the INTENT. One per business event.
-- ═══════════════════════════════════════════════════════════════════
CREATE TYPE txn_state AS ENUM ('pending','posted','failed','reversed');

CREATE TABLE ledger_transactions (
  id              bigint      PRIMARY KEY,   -- ULID-derived, time-sortable
  kind            text        NOT NULL,      -- 'transfer','topup','refund','settlement','fee'
  state           txn_state   NOT NULL DEFAULT 'pending',
  currency        char(3)     NOT NULL,
  idempotency_key text        NOT NULL,
  external_ref    text        NULL,          -- gateway payment id
  reverses_txn_id bigint      NULL REFERENCES ledger_transactions(id),
  metadata        jsonb       NOT NULL DEFAULT '{}',
  created_at      timestamptz NOT NULL DEFAULT now(),
  posted_at       timestamptz NULL
) PARTITION BY RANGE (created_at);

CREATE UNIQUE INDEX uq_ltxn_idem ON ledger_transactions (idempotency_key);
CREATE INDEX idx_ltxn_external ON ledger_transactions (external_ref)
  WHERE external_ref IS NOT NULL;
CREATE INDEX idx_ltxn_pending ON ledger_transactions (created_at)
  WHERE state = 'pending';

-- ═══════════════════════════════════════════════════════════════════
-- ENTRIES — the MOVEMENTS. Immutable. This is the source of truth.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE ledger_entries (
  id            bigint      NOT NULL,
  txn_id        bigint      NOT NULL,
  account_id    bigint      NOT NULL REFERENCES accounts(id),
  amount_minor  bigint      NOT NULL,        -- ★ SIGNED. +credit, -debit.
  currency      char(3)     NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  CHECK (amount_minor <> 0),
  PRIMARY KEY (created_at, id)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_entries_account ON ledger_entries (account_id, created_at DESC);
CREATE INDEX idx_entries_txn ON ledger_entries (txn_id);

-- ★ I4 ENFORCED BY THE ENGINE, not by convention:
CREATE RULE no_update_entries AS ON UPDATE TO ledger_entries DO INSTEAD NOTHING;
CREATE RULE no_delete_entries AS ON DELETE TO ledger_entries DO INSTEAD NOTHING;
-- (and revoke UPDATE/DELETE from the app role — see 7.6)

-- ═══════════════════════════════════════════════════════════════════
-- BALANCE SNAPSHOTS — the derived read model. Rebuildable from entries.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE account_balances (
  account_id       bigint      PRIMARY KEY REFERENCES accounts(id),
  balance_minor    bigint      NOT NULL DEFAULT 0,
  last_entry_id    bigint      NOT NULL DEFAULT 0,   -- watermark
  entry_count      bigint      NOT NULL DEFAULT 0,
  updated_at       timestamptz NOT NULL DEFAULT now()
) WITH (fillfactor = 70);
-- ★ This is a CACHE. It can be dropped and rebuilt from ledger_entries.
--   It is NEVER the source of truth. That distinction is everything.

-- ═══════════════════════════════════════════════════════════════════
-- OUTBOX — no dual writes (Topic 52)
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE outbox (
  id         bigserial   PRIMARY KEY,
  topic      text        NOT NULL,
  payload    jsonb       NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz NULL
);
CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

**Every design decision, justified:**

| Decision | Why |
|---|---|
| `amount_minor bigint`, signed | Money is exact integers in the smallest unit. Never float. Never `numeric` on a 12k/s path. A sign lets I1 be a single `SUM(...) = 0` check |
| Entries are the source of truth | I4. History cannot be overwritten because nothing is ever written twice |
| `account_balances` is a cache | Makes pattern A O(1) without making balance authoritative. Rebuildable ⇒ a bug in the cache is recoverable; a bug in a stored balance is not |
| Partition by `created_at` | 7-year retention with 8M entries/day = 20B rows. Retention/archival = `DETACH PARTITION` (Topic 59) |
| ULID-derived `id` | Time-sortable → B-tree inserts go to the rightmost page, no page-split storm (Topic 08) |
| `state` on transactions, not entries | Entries are facts that happened. A transaction is an intent that may still be pending |
| `suspense` account | Unexplained money must land *somewhere* that keeps I1 true. Never "just log it" |
| RULE + revoked grants | I4 must be enforced where a 2am psql session can't bypass it |

### 7.2 Why an append-only ledger removes the hot row

```
 ATTEMPT 1                            PRODUCTION
 ────────────────────────────         ─────────────────────────────────────
 UPDATE wallets                       INSERT INTO ledger_entries
   SET balance = balance + 100          (txn, account 9001, +100)
 WHERE id = 9001;
                                      • Row lock: NONE. It's a new row.
 • Exclusive row lock on 9001         • Two concurrent credits to account
 • Held until COMMIT                    9001 write two DIFFERENT rows on
 • Serialised: ~900/sec                 (possibly) different pages.
                                      • Serialised: NOT AT ALL.

 ⇒ Credits scale linearly. 800/sec into one merchant is 800 independent
   INSERTs. The B-tree on (account_id, created_at) does receive them all,
   but a B-tree insert is not an exclusive row lock — it's a brief page
   latch, measured in microseconds.
```

**This is the deepest lesson in the case study.** The hot-row problem in case study 01 was solved by splitting a counter into items. Here it's solved by **changing an update into an insert**. Both are the same underlying move: *stop making concurrent writers touch the same row.*

### 7.3 Enforcing I3 (non-negative) without a hot row

Credits need no coordination. **Debits do** — you must know the balance before allowing one. The trick is to serialise *only per account*, and only for debits:

```sql
-- A transactional advisory lock: automatically released at COMMIT/ROLLBACK,
-- costs one hash-table entry, no table row is touched, no MVCC version created.
SELECT pg_advisory_xact_lock($1);     -- $1 = account_id
```

```js
async function postTransaction({ idempotencyKey, kind, currency, legs, metadata }) {
  // legs = [{ accountId, amount }, ...]   amounts must sum to 0
  const sum = legs.reduce((a, l) => a + BigInt(l.amount), 0n);
  if (sum !== 0n) throw new Error('UNBALANCED_TRANSACTION');   // I1, checked early

  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL lock_timeout = '2s'");

    // ── I5: idempotency, enforced by a unique index, not a SELECT ──
    const txn = await client.query(
      `INSERT INTO ledger_transactions (id, kind, state, currency, idempotency_key, metadata)
       VALUES ($1,$2,'pending',$3,$4,$5)
       ON CONFLICT (idempotency_key) DO NOTHING
       RETURNING id, state`,
      [ulid(), kind, currency, idempotencyKey, metadata]);

    if (txn.rowCount === 0) {
      await client.query('ROLLBACK');
      const prev = await pool.query(
        `SELECT id, state FROM ledger_transactions WHERE idempotency_key=$1`, [idempotencyKey]);
      return { status: 200, replayed: true, txnId: prev.rows[0].id, state: prev.rows[0].state };
    }
    const txnId = txn.rows[0].id;

    // ── Serialise ONLY the debited accounts, in a deterministic order ──
    const debited = legs.filter(l => BigInt(l.amount) < 0n)
                        .map(l => l.accountId)
                        .sort((a, b) => a - b);              // ★ deadlock-free
    for (const accId of debited) {
      await client.query('SELECT pg_advisory_xact_lock($1)', [accId]);
    }

    // ── I3 + I6: verify sufficient funds and currency match, inside the lock ──
    if (debited.length) {
      const chk = await client.query(
        `SELECT a.id, a.currency, a.allows_negative, a.credit_limit_minor,
                b.balance_minor,
                COALESCE((SELECT sum(e.amount_minor) FROM ledger_entries e
                           WHERE e.account_id = a.id AND e.id > b.last_entry_id), 0) AS delta
           FROM accounts a
           JOIN account_balances b ON b.account_id = a.id
          WHERE a.id = ANY($1)`, [debited]);

      for (const row of chk.rows) {
        if (row.currency !== currency) throw new AppError('CURRENCY_MISMATCH');   // I6
        const current = BigInt(row.balance_minor) + BigInt(row.delta);
        const leg = legs.find(l => l.accountId === Number(row.id));
        const after = current + BigInt(leg.amount);
        const floor = row.allows_negative ? -BigInt(row.credit_limit_minor) : 0n;
        if (after < floor) throw new AppError('INSUFFICIENT_FUNDS');              // I3
      }
    }

    // ── Write the entries. INSERTs — no row contention. ──
    await client.query(
      `INSERT INTO ledger_entries (id, txn_id, account_id, amount_minor, currency)
       SELECT $1 + ord, $2, x.account_id, x.amount, $3
         FROM jsonb_to_recordset($4::jsonb)
                AS x(account_id bigint, amount bigint)
         WITH ORDINALITY AS t(ord)`,
      [nextEntryIdBase(), txnId, currency, JSON.stringify(legs)]);

    // ── I1 verified at write time, in the same transaction ──
    const bal = await client.query(
      'SELECT sum(amount_minor) AS s FROM ledger_entries WHERE txn_id = $1', [txnId]);
    if (BigInt(bal.rows[0].s) !== 0n) throw new Error('LEDGER_IMBALANCE');   // belt & braces

    await client.query(
      `UPDATE ledger_transactions SET state='posted', posted_at=now() WHERE id=$1`, [txnId]);

    // ── Update the balance CACHE in the same transaction (see note below) ──
    await client.query(
      `INSERT INTO account_balances (account_id, balance_minor, last_entry_id, entry_count)
       SELECT e.account_id, sum(e.amount_minor), max(e.id), count(*)
         FROM ledger_entries e WHERE e.txn_id = $1 GROUP BY e.account_id
       ON CONFLICT (account_id) DO UPDATE
         SET balance_minor = account_balances.balance_minor + EXCLUDED.balance_minor,
             last_entry_id = GREATEST(account_balances.last_entry_id, EXCLUDED.last_entry_id),
             entry_count   = account_balances.entry_count + EXCLUDED.entry_count,
             updated_at    = now()`,
      [txnId]);

    // ── No dual write. The outbox is part of the same transaction. ──
    await client.query(
      `INSERT INTO outbox (topic, payload) VALUES ('ledger.posted', $1)`,
      [JSON.stringify({ txnId, kind, legs })]);

    await client.query('COMMIT');
    return { status: 201, txnId };

  } catch (e) {
    await client.query('ROLLBACK').catch(() => {});
    if (e instanceof AppError) return { status: 422, error: e.message };
    if (e.code === '55P03') return { status: 503, error: 'RETRY' };
    throw e;
  } finally { client.release(); }
}
```

**The critical nuance about `account_balances`:** updating it inside the transaction *does* reintroduce a row lock on the merchant's balance row. That is a deliberate trade, and here is the reasoning:

```
 OPTION 1: update account_balances in the same transaction
   ✓ balance is always exactly correct
   ✗ reintroduces the hot row for high-volume accounts

 OPTION 2: update account_balances asynchronously from the outbox
   ✓ zero contention; INSERTs only in the hot path
   ✗ balance is briefly stale

 OPTION 3 (WHAT WE DO): hybrid, decided per account
   • Normal accounts (99.99%): Option 1. Contention is nil at <10 txn/s.
   • HIGH-VOLUME accounts flagged `hot`: Option 2 — the balance row is
     updated by a single-threaded roll-up worker every 200 ms, batching
     thousands of entries into ONE update.
   • Debit checks (I3) always read `balance_minor + sum(entries after
     last_entry_id)` — so they are EXACT even when the snapshot is stale.
     ★ THIS IS THE KEY: the cache can lag; the CORRECTNESS CHECK cannot,
       and the watermark makes it exact regardless of lag.
```

That `last_entry_id` watermark is what lets the read model be eventually consistent while the invariant check stays exact. It is worth understanding deeply — the pattern generalises to any "cached aggregate + authoritative log."

### 7.4 The gateway state machine — handling "unknown"

```
                    ┌──────────────┐
   POST /payments   │   INITIATED  │   ledger: NO entries yet
   ──────────────▶  │              │
                    └──────┬───────┘
                           │ call gateway
              ┌────────────┼────────────┐
              │            │            │
         success       timeout        error
              │            │            │
              ▼            ▼            ▼
      ┌─────────────┐ ┌─────────┐ ┌──────────┐
      │  AUTHORISED │ │ UNKNOWN │ │  FAILED  │
      │             │ │  ⚠      │ │          │
      └──────┬──────┘ └────┬────┘ └──────────┘
             │             │        ledger: none
             │             │ RECONCILER polls the gateway
             │             │ every 30 s until resolved,
             │             │ up to 24 h, then → suspense
             │             └────────┬────────┐
             │                      ▼        ▼
             │                 AUTHORISED  FAILED
             ▼
      ┌─────────────┐
      │  CAPTURED   │  ★ LEDGER ENTRIES WRITTEN HERE, AND ONLY HERE
      │             │     gateway_float  +amount
      │             │     user_wallet    -amount... etc
      └──────┬──────┘
             │ refund
             ▼
      ┌─────────────┐
      │  REFUNDED   │  new REVERSING entries; original untouched (I4)
      └─────────────┘
```

**The rule that makes this safe: no ledger entry is written until the gateway state is *known*.** A timeout writes nothing. The user sees "processing." The reconciler resolves it. Money is never moved on an assumption.

```js
// Webhook handler — idempotent by construction
async function handleGatewayWebhook(evt) {
  if (!verifySignature(evt)) return { status: 401 };

  return withTransaction(async (tx) => {
    // Record the raw event first — always, even if we can't process it.
    // This is your evidence in a dispute.
    const ins = await tx.query(
      `INSERT INTO gateway_events (provider, event_id, payload)
       VALUES ($1,$2,$3) ON CONFLICT (provider, event_id) DO NOTHING RETURNING id`,
      [evt.provider, evt.id, evt]);
    if (ins.rowCount === 0) return { status: 200, note: 'duplicate' };   // replay

    if (evt.type !== 'payment.captured') return { status: 200 };

    // Transition the payment. Only 'authorised' or 'unknown' may become captured.
    const p = await tx.query(
      `UPDATE payments SET state='captured', captured_at=now()
        WHERE gateway_ref=$1 AND state IN ('authorised','unknown')
       RETURNING id, user_account_id, merchant_account_id, amount_minor, fee_minor, currency`,
      [evt.data.payment_id]);

    if (p.rowCount === 0) {
      const cur = await tx.query('SELECT state FROM payments WHERE gateway_ref=$1',
                                 [evt.data.payment_id]);
      if (cur.rows[0]?.state === 'captured') return { status: 200 };     // replay
      // Unknown payment or illegal transition → park it, don't guess.
      await tx.query(`INSERT INTO reconciliation_queue (reason, payload) VALUES ($1,$2)`,
                     ['UNMATCHED_CAPTURE', evt]);
      return { status: 202 };
    }

    const r = p.rows[0];
    // A four-leg balanced transaction. Sums to zero. Fees are explicit.
    await postTransactionInTx(tx, {
      idempotencyKey: `capture:${evt.data.payment_id}`,
      kind: 'payment_capture',
      currency: r.currency,
      legs: [
        { accountId: GATEWAY_FLOAT[r.currency], amount:  r.amount_minor },
        { accountId: r.merchant_account_id,     amount: -(r.amount_minor - r.fee_minor) },
        { accountId: FEE_REVENUE[r.currency],   amount: -r.fee_minor },
      ].map(l => ({ ...l, amount: -l.amount })),   // sign convention per your chart
      metadata: { gateway_ref: evt.data.payment_id },
    });

    return { status: 200 };
  });
}
```

### 7.5 Reading a balance at 80,000/s

```js
async function getBalance(accountId) {
  const cached = await redis.get(`bal:${accountId}`);
  if (cached) return JSON.parse(cached);            // ≤2 s stale, per the brief

  const { rows } = await replicaPool.query(
    `SELECT b.balance_minor
            + COALESCE((SELECT sum(e.amount_minor) FROM ledger_entries e
                         WHERE e.account_id = b.account_id
                           AND e.id > b.last_entry_id), 0) AS balance
       FROM account_balances b WHERE b.account_id = $1`, [accountId]);

  await redis.setex(`bal:${accountId}`, 2, JSON.stringify(rows[0]));
  return rows[0];
}
```

The `+ delta` term costs nothing for normal accounts (the subquery returns 0 rows via `idx_entries_account`) and is essential for hot accounts whose snapshot lags. **Same query, correct in both cases.**

### 7.6 Making immutability real

```sql
-- The application role can INSERT and SELECT. That is all.
REVOKE UPDATE, DELETE, TRUNCATE ON ledger_entries FROM app_role;
REVOKE UPDATE, DELETE, TRUNCATE ON ledger_transactions FROM app_role;
GRANT INSERT, SELECT ON ledger_entries TO app_role;
GRANT INSERT, SELECT, UPDATE (state, posted_at) ON ledger_transactions TO app_role;
--                            ↑ column-level grant: only the state may advance

-- Even a superuser leaves a trace:
CREATE TABLE ledger_audit (
  id bigserial PRIMARY KEY, table_name text, operation text,
  old_row jsonb, new_row jsonb, db_user text, at timestamptz DEFAULT now()
);
CREATE OR REPLACE FUNCTION audit_ledger() RETURNS trigger AS $$
BEGIN
  INSERT INTO ledger_audit (table_name, operation, old_row, new_row, db_user)
  VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), to_jsonb(NEW), session_user);
  RETURN NEW;
END $$ LANGUAGE plpgsql;
CREATE TRIGGER trg_audit_entries AFTER UPDATE OR DELETE ON ledger_entries
  FOR EACH ROW EXECUTE FUNCTION audit_ledger();
```

### 7.7 The daily audit — I2, verified

```sql
-- (1) GLOBAL ZERO-SUM. Must be exactly 0. If not, page someone.
SELECT currency, sum(amount_minor) AS drift
FROM ledger_entries GROUP BY currency HAVING sum(amount_minor) <> 0;

-- (2) PER-TRANSACTION ZERO-SUM (I1). Must return zero rows.
SELECT txn_id, sum(amount_minor) FROM ledger_entries
GROUP BY txn_id HAVING sum(amount_minor) <> 0;

-- (3) SNAPSHOT DRIFT — the cache vs the truth. Must return zero rows.
SELECT b.account_id, b.balance_minor AS cached, t.actual
FROM account_balances b
JOIN LATERAL (SELECT sum(amount_minor) AS actual FROM ledger_entries e
              WHERE e.account_id = b.account_id) t ON true
WHERE b.balance_minor <> COALESCE(t.actual, 0);

-- (4) NEGATIVE BALANCES on accounts that forbid them (I3).
SELECT a.id, t.actual FROM accounts a
JOIN LATERAL (SELECT sum(amount_minor) AS actual FROM ledger_entries e
              WHERE e.account_id = a.id) t ON true
WHERE NOT a.allows_negative AND COALESCE(t.actual,0) < 0;

-- (5) GATEWAY RECONCILIATION — our float vs their settlement report.
SELECT 'ours' AS src, sum(amount_minor) FROM ledger_entries
 WHERE account_id = (SELECT id FROM accounts WHERE kind='gateway_float' AND currency='INR')
   AND created_at::date = current_date - 1
UNION ALL
SELECT 'gateway', sum(amount_minor) FROM gateway_settlement_report
 WHERE settlement_date = current_date - 1;

-- (6) STUCK PENDING — payments in 'unknown' for too long.
SELECT count(*) FROM payments WHERE state='unknown' AND created_at < now() - interval '2 hours';
```

Checks 1–4 must return **zero rows, every day, forever**. If one doesn't, you have a bug that created or destroyed money, and you find it while it's one transaction old rather than six months old.

---

## 8. THE NUMBERS

| Metric | Attempt 1 | Attempt 2 | Production |
|---|---|---|---|
| Transfers/sec (spread) | 870 | 3,100 | **11,800** |
| Transfers/sec (one hot merchant) | 870 | 890 | **8,400** |
| Deadlock rate | 3.1% | 0.01% | **0** (advisory locks, ordered) |
| Double-charges under retry | ~20/s | 0 | **0** |
| Balance read p99 | 0.9 ms | 0.9 ms | **0.3 ms** (Redis) / 1.1 ms (DB) |
| Audit "balance on 2024-03-11" | **impossible** | impossible | **one query** |
| Gateway timeout → money lost/created | possible | possible | **impossible** (nothing written until known) |
| `wallets`/`account_balances` size | 41 GB | 41 GB | **3.2 GB** (hot accounts batched) |
| Detection time for a money bug | months | months | **≤ 24 hours** (daily audit) |
| Storage for 7 years | n/a | n/a | ~14 TB, partitioned, old partitions compressed & detached |

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Gateway times out** | State = `unknown`. No entries written | Reconciler polls until resolved. After 24 h → `suspense` + human. **Money never moves on a guess** |
| **Gateway sends a capture for an unknown payment** | Can't match it | Parked in `reconciliation_queue`, credited to `suspense` so I1 holds. Never dropped, never guessed |
| **Webhook delivered 5×** | Would double-post | `gateway_events (provider, event_id)` unique + state-guarded transition. Replays return 200 |
| **`account_balances` cache drifts** | Balances shown wrong | Daily check (4) catches it; **rebuild is a single query** from entries. This is why the cache being non-authoritative matters so much |
| **Redis dies** | Balance reads hit the replica | Degraded latency (0.3 → 1.1 ms), still correct. Never a correctness issue |
| **Roll-up worker for hot accounts dies** | Snapshot lags | Debit checks stay exact via the `last_entry_id` delta. Reads get slower as the delta grows. Alert on `max(entry_count_behind)` |
| **Someone runs a manual UPDATE at 2am** | Would corrupt history | Grants revoked; RULEs block it; audit trigger records any attempt |
| **A partition fills / disk pressure** | Writes fail | `pg_partman` pre-creates partitions 3 months ahead; alert if the newest partition is <30 days out |
| **Node clock skew** | Wrong `created_at` ordering | All timestamps from PostgreSQL `now()`. IDs are ULIDs generated with the DB's clock for ledger ordering |

**The five alerts:**

```
1. LEDGER IMBALANCE      audit check (1) or (2) returns any row     → PAGE IMMEDIATELY
2. SNAPSHOT DRIFT        audit check (3) returns any row            → PAGE
3. STUCK UNKNOWN         payments in 'unknown' > 2 h, count > 0     → PAGE
4. SUSPENSE BALANCE      suspense account balance <> 0 for > 24 h   → ticket
5. ROLLUP LAG            max entries behind snapshot > 50,000       → ticket
```

Alert 1 is the only one in this entire curriculum that should wake a human at 3am regardless of business hours. An imbalanced ledger means money was created or destroyed, and every minute it runs makes the forensics harder.

---

## 10. WHAT BREAKS AT 10×

**120,000 txn/sec, 400M accounts, 80M entries/day.**

| Wall | Why | Next design |
|---|---|---|
| Single primary write capacity | ~20k write txns/sec ceiling on one node | **Shard by account_id.** ⚠ But a transfer touches two accounts — if they're on different shards you need 2PC or sagas (Topic 51). The usual answer: shard by *user*, and route cross-shard transfers through a **clearing account on each shard**, turning one cross-shard transfer into two intra-shard ones |
| Entry table size | 80M/day × 7 years = 200B rows | Already partitioned. Compress and detach partitions older than 90 days to object storage; keep them queryable via a foreign table for audits |
| Advisory lock contention | One very hot debit account (e.g. a corporate payer) | Pre-fund a set of sub-accounts and round-robin debits across them, reconciling to the parent nightly. (Same "split the hot thing" move as case study 01) |
| Daily audit query duration | Full `sum()` over 200B rows | Maintain **per-partition running totals**; the daily check sums 2,500 partition totals instead of 200B rows. Full recompute weekly |
| Balance read at 800k/s | Redis single-key limits | Shard the Redis keyspace; consider a read-through local cache with a 500 ms TTL per pod |

---

## 11. THE SEVEN QUESTIONS

1. Why is a stored `balance` column fundamentally incompatible with requirement 4 (auditability)? Name what is destroyed and when.
2. The append-only ledger removes the merchant hot row. Explain the mechanism — what specifically stops being contended?
3. `account_balances` is a cache, not the truth. Name three concrete consequences of that distinction.
4. Explain the `last_entry_id` watermark. How does it let the balance snapshot be stale while the insufficient-funds check stays exact?
5. A gateway call times out. Why is writing *no* ledger entries the only safe action? What goes wrong if you treat a timeout as failure? As success?
6. Why are advisory locks used for debits instead of `SELECT ... FOR UPDATE` on the balance row? Name two advantages.
7. Audit check (1) returns a row showing a ₹4 drift in INR. Walk through your investigation, in order.

---

## 12. TRANSFERABLE LESSONS

**1. Store events, derive state. Never the reverse.**
A balance, a stock level, a score, a status — if you `UPDATE` it, you destroy the information needed to explain it. Store the *movements* and compute the *position*. The performance objection ("summing is slow") is answered by a snapshot + watermark, not by abandoning the log. → Reappears in **16 (inventory movements)**, **18 (parcel scan events)**, **13 (score history)**.

**2. Convert UPDATEs into INSERTs to remove contention.**
This is the same family of move as case study 01's "one row per unit," and it is more broadly applicable. Concurrent INSERTs of different rows do not block each other. Concurrent UPDATEs of one row serialise completely. → Reappears in **14 (budget spend)**, **19 (view counts)**, **61 (curriculum)**.

**3. A timeout is "unknown," not "failed."**
Any integration with an external system needs three outcomes, not two. Systems that model only success/failure will eventually double-charge or lose money on the ambiguous case — and the ambiguous case is *common* at scale. → Reappears in **09 (delivery)**, **12 (order state)**, **51 (curriculum)**.

**4. Idempotency is a unique constraint, not a check.**
`SELECT WHERE key = ? ; if not found INSERT` has a race window and will fail under retry storms — which is exactly when it's needed. `INSERT ... ON CONFLICT DO NOTHING` on a unique index has no window. → Reappears in every case study with an external caller.

**5. Enforce immutability with grants, not with discipline.**
"We agreed not to UPDATE that table" survives about nine months and one incident. `REVOKE UPDATE` survives forever. Put invariants where a tired human at 2am cannot bypass them. → Reappears in **08 (audit log)**, **17 (EHR)**.

**6. Every invariant needs a query that verifies it, running on a schedule.**
The design makes violations *hard*. The daily audit makes them *visible*. You need both — because the bug you didn't anticipate is the one that gets you, and detection time is the difference between a one-row fix and a six-month forensic project. → Universal.

**7. Unexplained money needs a home.**
A `suspense` account keeps I1 true even when you don't yet know what happened. The alternative — dropping or guessing — either breaks the invariant or creates a wrong fact. **Design a place for "I don't know yet" in any system with a conservation law.**

---

## FILES TO READ NEXT

- **04 — Subscription billing** (this folder): proration, invoice immutability, and recomputing history without rewriting it
- **09 — Notification delivery** (this folder): the outbox used here, at scale
- **40 — ACID in depth** (curriculum): what each property guarantees here
- **52 — Idempotency and transactional messaging** (curriculum): the outbox pattern in full
- **51 — Distributed transactions and 2PC** (curriculum): what the 10× cross-shard transfer requires
- **24 — Constraints in depth** (curriculum): grants, rules, and DB-level enforcement
