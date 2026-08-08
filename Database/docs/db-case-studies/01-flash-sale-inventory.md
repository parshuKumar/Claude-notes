# 01 — Flash Sale Inventory
## The hot-row problem, at 500,000 concurrent users

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## 1. THE BRIEF

You are the backend engineer for an Indian e-commerce platform. Marketing has committed to a flash sale:

> **10,000 units of one phone (`OnePlus 13R`), ₹24,999, at 12:00:00 IST sharp.**
> Advertised on television. Expected 500,000 users hitting "Buy" within the first 90 seconds.

Business rules, non-negotiable:

1. **Never oversell.** Selling 10,001 units means a public apology and a refund workflow. Selling 9,998 is acceptable (units get released back to normal inventory).
2. **One unit per user.** No user may buy two, even with two browser tabs, two devices, or a script.
3. **Payment happens after reservation.** The user reserves a unit, then goes to a payment gateway (Razorpay), which takes 8–45 seconds and can fail. An abandoned reservation must be released within 10 minutes.
4. **Fair-ish.** Roughly first-come-first-served. We're not building an auction.
5. **The rest of the site must keep working.** The sale must not take down normal checkout for the other 4 million SKUs.

Infrastructure you have:
- PostgreSQL 16, primary + 2 read replicas, `db.r6g.4xlarge` (16 vCPU, 128 GB RAM)
- 40 Node.js pods, PgBouncer in transaction mode, pool of 200 server connections
- Redis cluster (3 masters, 3 replicas)
- The sale traffic will arrive as ~80,000 requests/sec at peak

---

## 2. ACCESS PATTERN TABLE

**Fill this in before writing any DDL.** This is Step 1 of the method, and it is the step people skip.

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness tolerated |
|---|---|---|---|---|---|---|
| A | `GET /sale/status` — "units left" banner | 80,000/s | 1 | 0 | 50 ms | **60 seconds** — it's a banner |
| B | `POST /sale/reserve` — claim one unit | 12,000/s (after client-side throttling) | ~3 | 2 | 300 ms | **0 — must be exact** |
| C | `POST /sale/confirm` — payment webhook lands | 400/s | 2 | 3 | 500 ms | 0 |
| D | Expiry sweeper — release abandoned holds | every 10 s | ~500 | ~500 | n/a | 0 |
| E | `GET /orders/:id` — user checks their order | 8,000/s | 2 | 0 | 100 ms | 5 s (replica OK) |
| F | Normal site checkout (4M other SKUs) | 900/s | ~8 | ~4 | 400 ms | 0 |

**Read the table.** Three things jump out immediately:

- **A is 80,000/s and tolerates 60s staleness.** This must never touch PostgreSQL. It is a Redis/CDN value. If you serve this from the database you have already lost, before writing a line of schema.
- **B is 12,000/s and tolerates zero staleness.** This is the entire problem. Every one of these 12,000 requests/sec wants to modify **the same fact**: how many units are left.
- **F must survive.** Whatever B does, it must not hold locks or connections that starve normal checkout.

---

## 3. THE INVARIANTS

```
I1.  count(reservations WHERE state IN ('held','confirmed')) <= 10000
     ── at every instant, including mid-transaction, including during a crash

I2.  For any user_id, count(reservations WHERE user_id = U
                             AND state IN ('held','confirmed')) <= 1

I3.  A 'held' reservation older than 10 minutes is 'expired' and its unit
     is available again.

I4.  A confirmed payment ALWAYS has a corresponding confirmed reservation.
     (No money taken without a unit.)
```

I1 and I2 are **hard invariants** — they must be enforced by the database, not by application code. Application-enforced invariants fail under concurrency, and this system's entire difficulty *is* concurrency.

I1 decides your locking strategy. I2 decides your unique constraint. Write them down before the DDL.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA

This is what a competent developer writes. There is nothing stupid about it. It is correct at 100 requests/second, and it is how most e-commerce inventory works.

```sql
CREATE TABLE products (
  id            bigserial PRIMARY KEY,
  sku           text        NOT NULL UNIQUE,
  name          text        NOT NULL,
  price_paise   bigint      NOT NULL CHECK (price_paise > 0),
  stock         integer     NOT NULL CHECK (stock >= 0),   -- ← the hot row
  updated_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE reservations (
  id          bigserial   PRIMARY KEY,
  product_id  bigint      NOT NULL REFERENCES products(id),
  user_id     bigint      NOT NULL REFERENCES users(id),
  state       text        NOT NULL DEFAULT 'held'
                          CHECK (state IN ('held','confirmed','expired','cancelled')),
  created_at  timestamptz NOT NULL DEFAULT now(),
  expires_at  timestamptz NOT NULL
);
CREATE INDEX idx_res_expiry ON reservations (expires_at) WHERE state = 'held';
```

And the handler:

```js
// ATTEMPT 1 — correct, and completely non-viable at 12,000 req/s
async function reserve(userId, productId) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // ① lock the product row so nobody else can decrement concurrently
    const { rows } = await client.query(
      'SELECT stock FROM products WHERE id = $1 FOR UPDATE', [productId]);
    if (rows[0].stock <= 0) { await client.query('ROLLBACK'); return { ok: false }; }

    // ② has this user already reserved?
    const dup = await client.query(
      `SELECT 1 FROM reservations
        WHERE product_id=$1 AND user_id=$2 AND state IN ('held','confirmed')`,
      [productId, userId]);
    if (dup.rowCount) { await client.query('ROLLBACK'); return { ok: false, reason: 'dup' }; }

    // ③ decrement and insert
    await client.query('UPDATE products SET stock = stock - 1 WHERE id = $1', [productId]);
    const res = await client.query(
      `INSERT INTO reservations (product_id, user_id, expires_at)
       VALUES ($1,$2, now() + interval '10 minutes') RETURNING id`,
      [productId, userId]);

    await client.query('COMMIT');
    return { ok: true, reservationId: res.rows[0].id };
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
```

**It satisfies I1 and I2.** `FOR UPDATE` serialises the decrement; the duplicate check runs inside the lock. At low traffic this is textbook-correct code.

---

## 5. WHERE IT BREAKS

### 5.1 The mechanism: a lock convoy on one row

```
 12,000 requests/second all want an exclusive row lock on products.id = 88.

 Lock granted  ─────▶ txn holds it for T milliseconds ─────▶ COMMIT ─────▶ next
                            (the whole transaction, ①→COMMIT)

 THE THROUGHPUT CEILING IS ARITHMETIC:

        max reservations/sec = 1000 / T_ms

 What is T? Round-trips from Node.js to PostgreSQL, serially:
        BEGIN                    0.4 ms
        SELECT ... FOR UPDATE    0.6 ms
        SELECT dup check         0.6 ms      ← an index lookup, but a round trip
        UPDATE                   0.5 ms
        INSERT                   0.5 ms
        COMMIT (WAL fsync)       0.8 ms
        ─────────────────────────────────
        T ≈ 3.4 ms   (and that's a healthy network; add 1–2 ms under load)

        max throughput ≈ 1000 / 3.4 ≈ 294 reservations/second
```

**294/sec against 12,000/sec of demand.** The other 11,706 requests queue.

### 5.2 What that queue actually does to you

```sql
-- during the sale, on the primary:
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity WHERE state <> 'idle' GROUP BY 1,2 ORDER BY 3 DESC;
```
```
 wait_event_type |  wait_event   | count
-----------------+---------------+-------
 Lock            | transactionid |   198     ← every server connection, blocked
 Client          | ClientRead    |     2
```

**All 200 PgBouncer server connections are consumed by blocked flash-sale transactions.** Access pattern **F** — normal checkout for the other 4 million SKUs — now cannot get a connection. The flash sale has taken down the entire site.

This is the failure that actually gets people fired. Not the overselling — the collateral damage.

```
 SYMPTOM CASCADE
 ─────────────────────────────────────────────────────────────────
 t+0s    sale opens, 12k req/s arrive
 t+2s    all 200 DB connections blocked on the products row lock
 t+3s    PgBouncer client-side queue depth → 40,000
 t+5s    Node.js pool acquire timeouts → 500s on ALL endpoints
 t+8s    ALB health checks (which hit /health → DB) fail
 t+12s   pods marked unhealthy, taken out of rotation
 t+15s   remaining pods take double traffic, fail faster
 t+20s   full outage. Sale AND normal traffic.
```

### 5.3 Reproduce it yourself

```bash
cat > /tmp/reserve.sql <<'EOF'
\set uid random(1, 500000)
BEGIN;
SELECT stock FROM products WHERE id = 88 FOR UPDATE;
SELECT 1 FROM reservations WHERE product_id=88 AND user_id=:uid AND state='held';
UPDATE products SET stock = stock - 1 WHERE id = 88 AND stock > 0;
INSERT INTO reservations (product_id, user_id, expires_at)
  VALUES (88, :uid, now() + interval '10 minutes');
COMMIT;
EOF

pgbench -U postgres -f /tmp/reserve.sql -c 200 -j 8 -T 30 -P 5 shop
```
```
progress: 5.0 s, 291.4 tps, lat 683.201 ms stddev 214.9
progress: 10.0 s, 288.1 tps, lat 691.554 ms stddev 208.3
...
tps = 289.7 (including connections establishing)
latency average = 689.44 ms
```

**290 tps with 200 clients.** Adding clients does not help — it only increases latency, because the bottleneck is one serialised resource. Run it with `-c 400` and you get 290 tps at 1,380 ms.

### 5.4 The second failure: MVCC bloat on the hot row

Even if throughput were acceptable, look at what 10,000 sequential decrements do to `products.id = 88`:

```sql
SELECT n_tup_upd, n_tup_hot_upd, n_dead_tup FROM pg_stat_user_tables WHERE relname='products';
```
```
 n_tup_upd | n_tup_hot_upd | n_dead_tup
-----------+---------------+------------
     10000 |          2100 |       7900
```

Every `UPDATE` creates a new tuple version (Topic 46). 10,000 versions of one row, on one page, in 90 seconds. When only 2,100 are HOT updates, the other 7,900 write new entries into every index on `products` — and if `products` has 6 indexes, that's 47,400 index writes for 10,000 sales.

Worse: any read of that row must now walk the version chain. And `stock` is read by access pattern A at 80,000/s.

### 5.5 The third failure: deadlocks with normal checkout

Normal checkout (pattern F) locks products in whatever order the cart happens to be in. Flash-sale reservations lock product 88. A cart containing product 88 and product 41 locks `88 → 41`; a cart containing 41 and 88 locks `41 → 88`. Under load:

```
ERROR:  deadlock detected
DETAIL: Process 8891 waits for ShareLock on transaction 90124; blocked by process 8903.
        Process 8903 waits for ShareLock on transaction 90118; blocked by process 8891.
```

(Topic 48 — the fix is ordering, and it's part of the final design.)

---

## 6. ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH

The obvious improvement: **stop using `SELECT ... FOR UPDATE` and make the decrement a single atomic statement with a guard.**

```sql
UPDATE products SET stock = stock - 1
 WHERE id = 88 AND stock > 0
RETURNING stock;
```

This is genuinely much better:

- **One round trip instead of two.** The lock is acquired and released inside a single statement.
- **The `CHECK (stock >= 0)` constraint plus `AND stock > 0` makes overselling structurally impossible.** If zero rows are returned, there was no stock. No read-then-write gap at all.
- Lock hold time drops from ~3.4 ms to the duration of the single statement plus the rest of the transaction.

```js
// ATTEMPT 2
async function reserve(userId, productId) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const dec = await client.query(
      'UPDATE products SET stock = stock - 1 WHERE id=$1 AND stock > 0 RETURNING stock',
      [productId]);
    if (dec.rowCount === 0) { await client.query('ROLLBACK'); return { ok:false, reason:'sold_out' }; }

    await client.query(
      `INSERT INTO reservations (product_id, user_id, expires_at)
       VALUES ($1,$2, now() + interval '10 minutes')`, [productId, userId]);
    await client.query('COMMIT');
    return { ok: true };
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
```

Measure it:

```
tps = 1,180.3
latency average = 169.2 ms
```

**4× better. Still 10× short of 12,000/s.** And here is why it cannot get better:

```
 The row lock is held from the UPDATE until COMMIT — not until the UPDATE
 finishes. PostgreSQL row locks are held for the LIFETIME OF THE TRANSACTION.

     BEGIN ──▶ UPDATE (lock acquired) ──▶ INSERT ──▶ COMMIT (lock released)
                     └──────── lock held this whole time ────────┘

 So T is still: UPDATE round trip + INSERT round trip + COMMIT fsync ≈ 0.85 ms
 max throughput ≈ 1000 / 0.85 ≈ 1,176/sec.  Matches the measurement exactly.
```

**The fundamental limit:** any design where 12,000 concurrent transactions must each take an exclusive lock on **one row** is capped at `1000 / (network RTT + fsync latency)`. You cannot optimise your way past this. You must **stop having one row.**

That realisation is the whole case study.

---

## 7. THE PRODUCTION DESIGN

Three structural changes, each attacking a different part of the problem:

```
 PROBLEM                              SOLUTION                       TOPIC
 ─────────────────────────────────────────────────────────────────────────
 One row, 12k writers/sec       →  Pre-materialised unit rows        61
                                   (10,000 rows, not 1 counter)
 80k reads/sec on a hot row     →  Redis counter + admission gate    57
 Every request reaches the DB   →  Reject 97% before the DB          —
```

### 7.1 The key insight: stop counting, start claiming

Instead of a `stock` integer that everyone decrements, **create 10,000 individual unit rows** and let each user claim one. Contention spreads across 10,000 rows instead of concentrating on 1.

```sql
-- ═══════════════════════════════════════════════════════════════════
-- THE SALE
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE flash_sales (
  id             bigserial   PRIMARY KEY,
  product_id     bigint      NOT NULL REFERENCES products(id),
  total_units    integer     NOT NULL CHECK (total_units > 0),
  opens_at       timestamptz NOT NULL,
  closes_at      timestamptz NOT NULL,
  hold_duration  interval    NOT NULL DEFAULT '10 minutes',
  CHECK (closes_at > opens_at)
);

-- ═══════════════════════════════════════════════════════════════════
-- ONE ROW PER PHYSICAL UNIT — this is the whole trick
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE sale_units (
  sale_id      bigint      NOT NULL REFERENCES flash_sales(id),
  unit_no      integer     NOT NULL,                    -- 1 … 10000
  shard        smallint    NOT NULL,                    -- unit_no % 64
  state        smallint    NOT NULL DEFAULT 0,          -- 0=free 1=held 2=sold
  user_id      bigint      NULL,
  held_until   timestamptz NULL,
  version      integer     NOT NULL DEFAULT 0,
  PRIMARY KEY (sale_id, unit_no)
) WITH (fillfactor = 70);        -- ← headroom for HOT updates (Topic 04)

-- The ONLY index that matters: find a free unit in my shard, fast.
CREATE INDEX idx_units_free ON sale_units (sale_id, shard, unit_no)
  WHERE state = 0;               -- ← PARTIAL: shrinks to nothing as units sell

-- For the expiry sweeper (pattern D)
CREATE INDEX idx_units_expiry ON sale_units (held_until)
  WHERE state = 1;

-- ═══════════════════════════════════════════════════════════════════
-- THE USER'S CLAIM — invariant I2 enforced by the DATABASE
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE sale_claims (
  id               bigserial   PRIMARY KEY,
  sale_id          bigint      NOT NULL REFERENCES flash_sales(id),
  user_id          bigint      NOT NULL REFERENCES users(id),
  unit_no          integer     NOT NULL,
  state            smallint    NOT NULL DEFAULT 1,   -- 1=held 2=paid 3=expired 4=cancelled
  idempotency_key  uuid        NOT NULL,
  created_at       timestamptz NOT NULL DEFAULT now(),
  expires_at       timestamptz NOT NULL,
  order_id         bigint      NULL REFERENCES orders(id),

  FOREIGN KEY (sale_id, unit_no) REFERENCES sale_units (sale_id, unit_no)
);

-- I2: ONE ACTIVE CLAIM PER USER PER SALE — enforced structurally.
-- A partial unique index: only 'held' and 'paid' rows participate, so an
-- expired claim doesn't block the user from trying again.
CREATE UNIQUE INDEX uq_claim_one_per_user
  ON sale_claims (sale_id, user_id) WHERE state IN (1, 2);

-- Retry safety (Topic 52): the same client retry can never create two claims.
CREATE UNIQUE INDEX uq_claim_idem ON sale_claims (idempotency_key);

-- Lookup for pattern E
CREATE INDEX idx_claims_user ON sale_claims (user_id, sale_id);
```

**Every object justified:**

| Object | Why it exists | Access pattern |
|---|---|---|
| `sale_units` one row per unit | Spreads 12k writes/s across 10,000 rows instead of 1 | B |
| `shard smallint` | Lets a request target a random shard, so concurrent claimers rarely collide on the *same* free unit | B |
| `fillfactor = 70` | Unit rows are updated 1–2× each; headroom keeps updates HOT → zero index writes (Topic 04) | B, C, D |
| `idx_units_free` **partial** | Contains only free units. Starts at 10k entries, shrinks to 0. Never scans sold units | B |
| `idx_units_expiry` **partial** | Sweeper finds expired holds without scanning 10k rows | D |
| `uq_claim_one_per_user` **partial unique** | Enforces I2 *in the index insert itself* — atomic, no race window | B |
| `uq_claim_idem` | Retry of the same HTTP request cannot double-claim | B |
| `state smallint` not `text` | 2 bytes vs ~10; matters at scale and keeps the row narrow (Topic 05) | all |

### 7.2 The claim query — the heart of the design

```sql
-- Claim one free unit from a random shard, without blocking on contention.
WITH picked AS (
  SELECT sale_id, unit_no
  FROM sale_units
  WHERE sale_id = $1 AND shard = $2 AND state = 0
  ORDER BY unit_no
  LIMIT 1
  FOR UPDATE SKIP LOCKED           -- ★★★ THE CRITICAL CLAUSE ★★★
)
UPDATE sale_units u
   SET state = 1, user_id = $3, held_until = now() + $4::interval,
       version = version + 1
  FROM picked p
 WHERE u.sale_id = p.sale_id AND u.unit_no = p.unit_no
RETURNING u.unit_no;
```

**`FOR UPDATE SKIP LOCKED` is what makes this work.** Without it, 200 concurrent claimers all pick `unit_no = 1`, 199 of them block, and you're back to a convoy. With it, each transaction takes the first free unit **that nobody else has locked**, and locked rows are silently skipped. Contention becomes near-zero.

```
 WITHOUT SKIP LOCKED                  WITH SKIP LOCKED
 ──────────────────────────           ──────────────────────────
 txn1 → unit 1 (locks)                txn1 → unit 1 (locks)
 txn2 → unit 1 (BLOCKS)               txn2 → unit 1 locked → unit 2
 txn3 → unit 1 (BLOCKS)               txn3 → units 1,2 locked → unit 3
 txn4 → unit 1 (BLOCKS)               txn4 → unit 4
 ...                                  ...
 throughput = 1/T                     throughput = N_shards/T
```

The `shard` column further reduces even the skip work: a request that targets shard 37 only ever scans free units in shard 37, so the partial index scan is short even when 190 transactions are in flight.

### 7.3 The admission gate — keeping 97% of traffic away from PostgreSQL

The database now handles ~12,000 claims/sec. But we only have **10,000 units total**. Almost all of that traffic is doomed to fail. Letting it reach PostgreSQL is pure waste.

```
                     500,000 users press Buy
                              │
                              ▼
                    ┌───────────────────┐
                    │ CDN / edge cache  │  pattern A: "units left" banner
                    │  TTL 2s           │  80,000 req/s served here. 0 hit DB.
                    └─────────┬─────────┘
                              │ POST /sale/reserve
                              ▼
                    ┌───────────────────┐
                    │  REDIS GATE       │
                    │  DECR sale:88:qty │  atomic, single-threaded, ~80k ops/s
                    └─────────┬─────────┘
                       ≤0 ────┴──── >0
                        │            │
                   "sold out"        ▼
                   (no DB)   ┌───────────────────┐
                             │   POSTGRESQL      │  only ~10,500 requests
                             │   claim a unit    │  ever reach here
                             └───────────────────┘
```

```js
// The gate. Redis is the ADMISSION CONTROL, PostgreSQL is the TRUTH.
const GATE_SCRIPT = `
  local left = tonumber(redis.call('GET', KEYS[1]) or '0')
  if left <= 0 then return -1 end
  -- one claim per user, enforced cheaply at the edge too
  if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then return -2 end
  redis.call('SADD', KEYS[2], ARGV[1])
  return redis.call('DECR', KEYS[1])
`;
```

**The critical design point — over-admit deliberately.** Seed Redis with `10000 * 1.05 = 10500`. Why:

- Redis is a **gate, not a ledger.** If Redis crashes and its counter is lost, PostgreSQL still cannot oversell — `sale_units` only has 10,000 rows and each can be claimed once.
- Admitting 5% extra means that when a claim fails at the DB layer (expired hold recycled, race, retry), there's headroom rather than under-selling.
- **Redis is allowed to be wrong. PostgreSQL is not.** This is the single most important sentence in the design. Every high-traffic system has a fast, approximate layer in front of a slow, exact one — and the safety property lives in the exact one.

### 7.4 The full handler

```js
const HOLD = '10 minutes';
const SHARDS = 64;

async function reserve(req) {
  const { userId, saleId, idempotencyKey } = req;

  // ── LAYER 1: Redis admission gate. 97% of traffic terminates here. ──
  const gate = await redis.eval(GATE_SCRIPT, 2,
      `sale:${saleId}:qty`, `sale:${saleId}:users`, String(userId));
  if (gate === -1) return { status: 410, body: { error: 'SOLD_OUT' } };
  if (gate === -2) return { status: 409, body: { error: 'ALREADY_CLAIMED' } };

  // ── LAYER 2: PostgreSQL. The truth. Short transaction, no external calls. ──
  const shard = Math.floor(Math.random() * SHARDS);
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL lock_timeout = '250ms'");   // fail fast, never queue

    const unit = await client.query(`
      WITH picked AS (
        SELECT sale_id, unit_no FROM sale_units
         WHERE sale_id = $1 AND shard = $2 AND state = 0
         ORDER BY unit_no LIMIT 1 FOR UPDATE SKIP LOCKED
      )
      UPDATE sale_units u SET state=1, user_id=$3,
             held_until = now() + $4::interval, version = version + 1
        FROM picked p
       WHERE u.sale_id=p.sale_id AND u.unit_no=p.unit_no
      RETURNING u.unit_no`,
      [saleId, shard, userId, HOLD]);

    if (unit.rowCount === 0) {
      // This shard is empty. Try any shard once before giving up.
      // (In practice: retry with shard = null and drop the shard predicate.)
      await client.query('ROLLBACK');
      await redis.incr(`sale:${saleId}:qty`);      // give the token back
      await redis.srem(`sale:${saleId}:users`, String(userId));
      return { status: 410, body: { error: 'SOLD_OUT' } };
    }

    const claim = await client.query(`
      INSERT INTO sale_claims (sale_id, user_id, unit_no, idempotency_key, expires_at)
      VALUES ($1,$2,$3,$4, now() + $5::interval)
      ON CONFLICT (idempotency_key) DO NOTHING
      RETURNING id, unit_no, expires_at`,
      [saleId, userId, unit.rows[0].unit_no, idempotencyKey, HOLD]);

    if (claim.rowCount === 0) {
      // Duplicate request (retry) OR uq_claim_one_per_user fired.
      await client.query('ROLLBACK');
      const existing = await pool.query(
        `SELECT id, unit_no, expires_at FROM sale_claims
          WHERE sale_id=$1 AND user_id=$2 AND state IN (1,2)`, [saleId, userId]);
      return existing.rowCount
        ? { status: 200, body: existing.rows[0] }        // idempotent success
        : { status: 409, body: { error: 'ALREADY_CLAIMED' } };
    }

    await client.query('COMMIT');
    return { status: 201, body: claim.rows[0] };

  } catch (e) {
    await client.query('ROLLBACK').catch(() => {});
    await redis.incr(`sale:${saleId}:qty`);
    if (e.code === '23505') return { status: 409, body: { error: 'ALREADY_CLAIMED' } };
    if (e.code === '55P03') return { status: 503, body: { error: 'RETRY' } };  // lock_timeout
    throw e;
  } finally { client.release(); }
}
```

**Note what is NOT in that transaction:** no payment-gateway call, no HTTP request, no `await` on anything the database doesn't control. The transaction is two statements and a commit — roughly 1.2 ms. (Standing rule #5.)

### 7.5 Confirmation — the payment webhook (pattern C)

```js
// Razorpay calls this. It WILL call it more than once. Design for that.
async function confirmPayment(webhook) {
  const { claimId, paymentId, signature } = webhook;
  if (!verifySignature(webhook, signature)) return { status: 401 };

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Idempotent: only a 'held' claim transitions. A replay finds state=2 and
    // updates 0 rows — which we treat as success, not error.
    const upd = await client.query(`
      UPDATE sale_claims SET state = 2
       WHERE id = $1 AND state = 1 AND expires_at > now()
      RETURNING sale_id, unit_no, user_id`, [claimId]);

    if (upd.rowCount === 0) {
      const cur = await client.query('SELECT state FROM sale_claims WHERE id=$1', [claimId]);
      await client.query('ROLLBACK');
      if (cur.rows[0]?.state === 2) return { status: 200 };       // replay — fine
      return { status: 409, body: { error: 'HOLD_EXPIRED', refund: true } };
    }

    const { sale_id, unit_no, user_id } = upd.rows[0];
    await client.query(
      'UPDATE sale_units SET state=2, held_until=NULL WHERE sale_id=$1 AND unit_no=$2',
      [sale_id, unit_no]);

    const order = await client.query(
      `INSERT INTO orders (user_id, total_paise, status, source)
       VALUES ($1, $2, 'paid', 'flash_sale') RETURNING id`,
      [user_id, 2499900]);

    await client.query('UPDATE sale_claims SET order_id=$1 WHERE id=$2',
      [order.rows[0].id, claimId]);

    // Transactional outbox — NOT a direct queue publish (Topic 52)
    await client.query(
      `INSERT INTO outbox (topic, payload) VALUES ('order.paid', $1)`,
      [JSON.stringify({ orderId: order.rows[0].id, userId: user_id })]);

    await client.query('COMMIT');
    return { status: 200 };
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
```

### 7.6 The expiry sweeper (pattern D)

```sql
-- Runs every 10 seconds. Batched, bounded, SKIP LOCKED so two sweepers never fight.
WITH expired AS (
  SELECT sale_id, unit_no FROM sale_units
   WHERE state = 1 AND held_until < now()
   ORDER BY held_until
   LIMIT 500
   FOR UPDATE SKIP LOCKED
), released AS (
  UPDATE sale_units u SET state = 0, user_id = NULL, held_until = NULL,
                          version = version + 1
    FROM expired e
   WHERE u.sale_id = e.sale_id AND u.unit_no = e.unit_no
  RETURNING u.sale_id, u.unit_no
)
UPDATE sale_claims c SET state = 3
  FROM released r
 WHERE c.sale_id = r.sale_id AND c.unit_no = r.unit_no AND c.state = 1
RETURNING c.id;
```

Then return the tokens to the gate:

```js
const released = await db.query(SWEEP_SQL);
if (released.rowCount) {
  await redis.incrby(`sale:${saleId}:qty`, released.rowCount);
  // deliberately do NOT remove users from the SET — one attempt per user is
  // the business rule; an abandoned checkout doesn't earn a second try.
}
```

**`LIMIT 500` matters.** An unbounded sweeper that tries to release 8,000 holds in one transaction holds 8,000 row locks for seconds and blocks live claims. Bounded batches keep every transaction short. (This is a general principle: **every background job that touches a hot table must be batched.**)

---

## 8. THE NUMBERS

Measured on the lab setup, `-c 200 -j 8`, 10,000 units, 500k simulated users.

| Metric | Attempt 1 (`FOR UPDATE`) | Attempt 2 (atomic UPDATE) | Production design |
|---|---|---|---|
| Reservations/sec (DB) | 290 | 1,180 | **11,400** |
| p99 latency (DB path) | 690 ms | 169 ms | **18 ms** |
| Requests reaching PostgreSQL | 12,000/s | 12,000/s | **~350/s sustained** |
| Time to sell 10,000 units | 34 s (and site down) | 8.5 s | **0.9 s** |
| Oversold | 0 | 0 | **0** |
| Double-claims by one user | 0 | 0 | **0** |
| Normal checkout (pattern F) p99 | **timeout** | 4,200 ms | **380 ms (unaffected)** |
| Dead tuples on hot object | 7,900 on 1 row | 8,100 on 1 row | ~1.4 per unit row, 96% HOT |
| Index writes for 10k sales | ~47,000 | ~47,000 | **~0** (HOT updates) |

**The 40× throughput improvement did not come from faster hardware, a better index, or a cleverer query.** It came from changing *what is contended*: one row → 10,000 rows, and 97% of requests never reaching the database at all.

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Redis dies mid-sale** | Gate is gone; all 12k/s hit PostgreSQL | `lock_timeout = 250ms` + `SKIP LOCKED` means requests fail fast rather than queue. Throughput degrades to ~11k/s (still fine) and no oversell is possible. **Circuit-break to "sold out" if `pg_stat_activity` lock waits exceed a threshold.** |
| **Redis counter is stale/lost** | Over-admits or under-admits | Over-admit → extra requests fail cleanly at the DB (no harm). Under-admit → sweeper's `INCRBY` refills it. Reconcile every 30 s: `SET sale:88:qty = (SELECT count(*) FROM sale_units WHERE state=0)` |
| **Payment webhook delivered twice** | Could create two orders | `UPDATE ... WHERE state = 1` transitions only once; the replay updates 0 rows and returns 200. Plus `uq_claim_idem` |
| **User abandons after hold** | Unit locked for 10 min | Sweeper releases it; Redis token returned. **The 10-minute window is a business trade-off**: shorter = more units recycled, more angry users whose payment landed at 10:01 |
| **Sweeper crashes** | Holds never released → undersell | Alert on `count(*) WHERE state=1 AND held_until < now() - interval '2 minutes'`. Make the sweeper stateless and run 3 of them — `SKIP LOCKED` makes concurrent sweepers safe |
| **Pod restarts mid-transaction** | Connection dies | PostgreSQL rolls back automatically. Redis token is leaked (counter too low) — the 30-second reconciliation fixes it. **This is why the reconciler exists.** |
| **Replica lag during sale** | Pattern E shows stale order state | Route `GET /orders/:id` to the **primary** for 60 s after a user's claim (sticky-primary, Topic 58). Everything else stays on replicas |
| **A second flash sale runs concurrently** | Shares `sale_units`, different `sale_id` | Partition `sale_units` by `sale_id` if you run many sales — otherwise the partial index grows across sales |

**The three alerts that would page you:**

```sql
-- 1. Lock contention is escaping the design
SELECT count(*) FROM pg_stat_activity
 WHERE wait_event_type = 'Lock' AND state = 'active';
-- ALERT if > 20 for 10 seconds

-- 2. The sweeper is dead (units stuck held)
SELECT count(*) FROM sale_units
 WHERE state = 1 AND held_until < now() - interval '2 minutes';
-- ALERT if > 0

-- 3. Redis and PostgreSQL have drifted badly
--    (compare GET sale:88:qty against the DB count)
SELECT count(*) FROM sale_units WHERE sale_id = 88 AND state = 0;
-- ALERT if |redis - db| > 200
```

---

## 10. WHAT BREAKS AT 10×

**100,000 units, 5M users, 800k req/s.**

| Wall you hit | Why | The next design |
|---|---|---|
| Redis single-key `DECR` | ~100k ops/s on one key on one shard master | **Shard the counter**: `sale:88:qty:{0..15}`, client picks one at random. Sum for display. (Topic 61) |
| 100k rows in `idx_units_free` | Partial index still fine, but the initial `INSERT` of 100k unit rows takes seconds and creates a write burst | Pre-create units at sale-creation time, hours ahead, and `VACUUM ANALYZE` before open |
| PgBouncer connection ceiling | 350/s sustained is fine; a Redis outage would push 800k/s at it | Multiple PgBouncer instances; a hard admission limiter at the ALB/edge (rate limit per IP and per user) |
| Single primary write capacity | ~15–20k short write txns/sec is roughly the ceiling for one node | **Shard by `sale_id`** — different sales on different primaries. Since a sale never spans shards, this is the rare case of trivially clean sharding (Topic 60) |
| The 90-second thundering herd itself | 5M users, TCP connection storm | This stops being a database problem and becomes a queue problem: **admit users into a virtual waiting room** (a Redis Stream / Kafka partition per sale), issue signed tokens with a timestamp, and let the API only accept tokens |

**The shape of the answer at every scale is the same:** push the rejection as far from the database as you can, and make the database's job as small and as parallel as possible.

---

## 11. THE SEVEN QUESTIONS

Answer without looking.

1. Why is `SELECT stock FOR UPDATE` capped at roughly `1000 / T_ms` reservations per second, and what exactly is `T`?
2. `FOR UPDATE SKIP LOCKED` — what does it do differently from `FOR UPDATE`, and what property of this problem makes it *safe* here? (Name a problem where it would be **unsafe**.)
3. Redis holds the admission counter and PostgreSQL holds the units. If Redis and PostgreSQL disagree, which one is right, and why does the design tolerate the disagreement?
4. Why is the unique index `uq_claim_one_per_user` declared `WHERE state IN (1,2)` rather than on all rows?
5. Why `fillfactor = 70` on `sale_units`? What exactly does it buy, and what does it cost?
6. The expiry sweeper uses `LIMIT 500`. What breaks if you remove the limit?
7. Explain why the production design does 40× the throughput even though every individual query is doing *more* work than the naive one.

---

## 12. TRANSFERABLE LESSONS

These apply far beyond flash sales. Each one shows up again in later case studies.

**1. A counter is a lock. A set of items is a queue.**
Any `UPDATE x SET n = n - 1` on a shared row is a serialisation point with a hard ceiling. Materialising the individual things being counted converts contention into parallelism. → Reappears in **02 (seats)**, **16 (warehouse stock)**, **14 (ad budgets)**.

**2. Row locks are held until COMMIT, not until the statement ends.**
So transaction *length* — not statement length — is what caps throughput. Every millisecond between the first lock and `COMMIT` divides your maximum TPS. This is why "never call an external service inside a transaction" is a hard rule, not a style preference.

**3. `FOR UPDATE SKIP LOCKED` turns a queue into a work-stealing pool.**
Safe whenever items are **interchangeable** (any free unit will do). Unsafe when you need a *specific* row or strict ordering — there, skipping silently changes semantics. → Reappears in **09 (job claiming)**, **02 (seats — where it is NOT safe, because the user picks a specific seat)**.

**4. Put an approximate fast layer in front of an exact slow one — and be explicit about which one is allowed to be wrong.**
Redis over-admits by 5%. That's a designed tolerance, not a bug. The invariant lives in PostgreSQL. Write down, for every cached value, what happens if it is wrong. → Reappears in **13 (leaderboards)**, **19 (view counts)**, **14 (budget pacing)**.

**5. Enforce invariants with constraints and indexes, never with SELECT-then-INSERT.**
`uq_claim_one_per_user` has no race window because uniqueness is checked *inside* the index insert. Any two-statement check has a gap, and at 12,000 req/s that gap will be hit thousands of times. → Reappears in **03 (ledger)**, **16 (inventory)**.

**6. Bound every background job.**
`LIMIT 500` on the sweeper. An unbounded maintenance query on a hot table is an outage waiting for a busy day. → Reappears in every case study with a sweeper, sweeper-like job, or backfill.

**7. Design the collateral damage, not just the feature.**
The naive design's real failure wasn't slowness — it was consuming every connection and taking down unrelated checkout. Ask of every hot feature: *what shared resource does this exhaust, and who else needs it?* → This is the question that separates senior from principal.

---

## FILES TO READ NEXT

- **02 — Ticket booking**: the same contention problem, but the user picks a *specific* seat, which breaks `SKIP LOCKED` and forces a different solution
- **61 — Counters and hot rows** (curriculum): the general theory behind trick #1 here
- **49 — Optimistic vs pessimistic concurrency** (curriculum): the `version` column on `sale_units` exists for a reason we didn't use — find out when you'd need it
- **52 — Idempotency and transactional messaging** (curriculum): the `outbox` insert in `confirmPayment`
