# 104 — Design Amazon / E-Commerce Platform
## Category: HLD Case Study

---

## What is this?

Amazon is an **online store**. You browse a catalog of products, search for what you want, read a product page, drop things in a cart, and check out — money moves, an order is created, a box shows up at your door.

That one-sentence description hides the fact that "an online store" is not *a* system. It's **a dozen systems wearing a trench coat**: a search engine, a product catalog, a shopping cart, an inventory ledger, a payments platform, an order manager, a recommendation engine, a review system, a notification pipeline. Each of those is a full system-design question in its own right, and each has a *different* consistency, latency, and availability profile.

The single most important thing to say out loud in this interview is this:

> **E-commerce is several loosely-coupled subsystems, and they do not share requirements.** Browsing a product page can be stale by an hour and served from a cache on another continent. Selling the last unit of an item **must be exact, right now, no exceptions.** The whole design is the art of drawing the line between those two worlds and putting the right machinery on each side.

The interviewer will let you sketch the whole map, then say: *"okay — go deep on one."* Nine times out of ten the interesting one is **inventory and checkout**, because that's where the money is and where the concurrency bugs live.

---

## Why does it matter?

This question is asked because it is **integrative**. Feed problems (Instagram, Twitter) test one idea — fanout — very deeply. Amazon tests whether you can hold *many* ideas at once and, crucially, whether you know that **different parts of one product get different guarantees.** That single realization — strong consistency for money and stock, eventual consistency for reviews and "recently viewed" — is the mark of a senior candidate. Juniors try to make the whole system strongly consistent (and it falls over at Black Friday) or eventually consistent (and they oversell the PS5 by 4,000 units).

**In the interview**, the failure modes are the exam. Anyone can draw boxes. The question is: *"10,000 people click Buy on the last 100 units at the same instant. Sell exactly 100. Go."* If you reach for `UPDATE qty = qty - 1` without thinking about the race, you've failed. If you say "atomic conditional decrement, and I'll reserve stock with a hold during checkout, and the checkout is a saga with a compensating refund," you've passed.

**At work**, this is the most transferable case study there is. Every business app is *some* projection of it:
- An ordering flow that must not double-charge → **idempotency keys**.
- A booking system that must not double-book a seat → **the oversell problem**.
- A multi-step workflow across services that must not leave money in limbo → **the saga pattern**.
- A read-heavy content surface that must stay up under a spike → **cache + CDN + graceful degradation**.

Learn Amazon and you've learned the skeleton of nearly every transactional system you'll ever build.

---

## The core idea — explained simply

### The Department Store Analogy

Picture a physical department store, and think about who does what.

**The showroom floor (the browse path).** Bright, huge, full of shoppers just looking. There are display models everywhere, glossy signs, a big catalog you can flip through. If a price tag is five minutes out of date, nobody dies. If the store *photocopies* the catalog and hands copies out at the door so people don't crowd the one master binder — great, that's exactly what you want. **This is the read-heavy, cacheable, eventually-consistent world.** Most people who walk the floor never buy anything.

**The stockroom ledger (the inventory path).** In the back, there is exactly **one** clipboard that says how many units of each item are actually on the shelf. When a sale happens, someone crosses a number off that clipboard. There is **one clipboard on purpose.** If you photocopied the clipboard and let ten cashiers each cross off "last unit" on their own copy, you'd sell the last toaster ten times and have to apologize to nine customers. **This is the write-heavy, single-source-of-truth, strongly-consistent world.**

**The checkout counter (the checkout saga).** A cashier doesn't just take your money. They: put a *hold* on the item so nobody else grabs it while you fumble for your card, run the card, ring up the sale, cross the item off the clipboard, and hand you a receipt. If your card is declined halfway through, they **undo the hold** and put the item back. That ordered dance — with an undo for every step — is a **saga**.

| Analogy | Technical concept |
|---|---|
| The showroom floor, photocopied catalogs | Browse path: CDN + Redis cache, eventual consistency |
| Shoppers just looking | Browse QPS ≫ purchase QPS |
| The one clipboard in the back | Inventory DB — single source of truth, strongly consistent |
| Crossing off a number atomically | Atomic conditional decrement |
| Ten cashiers, ten photocopies of the clipboard | **The oversell bug** |
| Putting an item on hold while you pay | Inventory reservation with a TTL |
| The cashier's ordered dance with undo | The checkout **saga** with compensations |
| Your receipt, filed forever | Order record in a relational DB |
| "People also bought…" cards | Recommendations — eventual, best-effort |

The entire architecture below is just this store, drawn with the showroom and the stockroom on **opposite sides of a consistency wall.**

---

## Key concepts inside this topic

We'll run the 5-step framework: **requirements → estimation → the subsystem map → data & deep design → deep dives**. The twist for a system this broad: after we sketch the whole map, we **go deep on the one hard part — inventory and checkout** — because that's the part an interviewer actually wants to see you sweat.

---

### 1. Requirements

**Clarifying questions to ask out loud (they earn points):**
- Are we designing the *whole* of Amazon, or a slice? (Scope it: "I'll cover the full map, then go deep on the catalog/browse path and, mainly, inventory + checkout. I'll treat search, recommendations, and reviews as their own subsystems and stay shallow on them.")
- Single-seller (like early Amazon) or a marketplace with third-party sellers? (Marketplace multiplies the catalog and inventory complexity; assume mostly first-party, note the difference.)
- Do we handle payment ourselves or integrate a provider (Stripe/Braintree)? (Integrate — say "I will not build a card network.")
- Digital goods or physical, with shipping? (Physical — shipping/fulfillment is real but I'll keep it shallow.)

**Functional requirements (in scope):**
1. **Browse & search** a product catalog; filter and sort.
2. **Product detail page** — description, images, price, availability, reviews.
3. **Cart** — add/remove items, persists across sessions and devices.
4. **Checkout + payment** — turn a cart into a paid order without double-charging.
5. **Inventory tracking** — never oversell; reflect stock on the product page.
6. **Order management** — order lifecycle, history, cancellation/return.
7. **Reviews & ratings.**
8. **Recommendations** — "people also bought," "recently viewed."

**Non-functional requirements — this table decides the whole design:**

| Requirement | Target | Why it drives the design |
|---|---|---|
| **High availability** | 99.99%+ on the buy path | **Downtime = directly lost revenue.** A 1-hour outage on Prime Day is millions of dollars. |
| **Read-heavy browsing** | ~100:1+ browse:buy | Most sessions never purchase. This is why the browse path is cached to death. |
| **Low browse latency** | < 200ms p99 on product pages | A slow page loses the sale before checkout even begins. |
| **STRONG consistency where it counts** | inventory, payment, orders | **You must not oversell the last unit, and must not double-charge.** Non-negotiable. |
| **Eventual consistency where it doesn't** | reviews, recs, recently-viewed | A review showing up 10s late harms nobody. Buys you scale. |
| **Survive massive spikes** | 10x+ on Prime Day / Black Friday | Traffic is bursty and *predictable*. Pre-scale, queue, cache, shed load. |

The two rows that matter most are the two **consistency** rows, and the fact that **they disagree.** The senior insight, stated once and referred back to constantly:

> **This system is not "strongly consistent" or "eventually consistent." It is both, on purpose, in different places.** Browse is eventual (cache-friendly, cheap, fast). Inventory/payment/orders are strong (correct, guarded, sometimes slower). Every design decision below is really the question: *which side of the wall does this belong on?*

---

### 2. Capacity estimation

Show the arithmetic; round hard. The point of this section is to *discover the read/write asymmetry*, which then justifies the caching.

**Users and traffic**
```
DAU                         = 100,000,000
Sessions / user / day       ≈ 2                → 200,000,000 sessions/day
Page views / session        ≈ 20 (lots of browsing)
Page views / day            = 200M × 20 = 4,000,000,000  (4 billion)
Seconds/day                 ≈ 86,400  (call it 100,000 for easy math)

Browse QPS  = 4,000,000,000 / 86,400 ≈ 46,000  →  ~50,000 browse QPS
Peak (Prime Day, ~10x)                        →  ~500,000 browse QPS
```

**Purchases (the writes that matter)**
```
Orders / day                ≈ 5,000,000       (only ~2.5% of sessions buy)
Order QPS   = 5,000,000 / 86,400 ≈ 58          →  ~60 orders/sec
Peak (10x)                                     →  ~600 orders/sec
```

**Now say the loud part:**
```
   Browse : Buy
   50,000 : 60
   ≈ 800 : 1
```
> **Browsing dwarfs buying by ~3 orders of magnitude.** This asymmetry is the whole justification for the design: the browse path is enormous, uniform, and tolerant of staleness → **cache it aggressively** (CDN + Redis). The buy path is comparatively tiny but must be **exactly correct** → spend your consistency budget there. You are cheap where it's hot and careful where it's dangerous.

**Catalog & storage**
```
Catalog size                ≈ 300,000,000 products (marketplace scale)
Metadata / product          ≈ 5 KB (title, attributes, price, seller, ...)
Catalog metadata            = 300M × 5 KB = 1.5 TB      → fits in a sharded DB easily
Images                       ≈ 500 KB/product avg (multiple) → ~150 TB in blob storage + CDN
Search index (Elasticsearch)  a denormalized subset, ~ hundreds of GB to a few TB
```

**Cart storage**
```
Active carts                ≈ 20,000,000 at any time
Per cart                    ≈ 1 KB (a handful of line items)
Cart data                   = 20M × 1 KB = 20 GB       → trivially fits in a KV store / Redis
```
Carts are small, hot, and per-user — a perfect fit for a fast KV store, not the relational DB.

---

### 3. The subsystem map

Before going deep, draw the whole thing. This shows the interviewer you know the territory, and it lets *them* pick where to dive. **Services are split by business capability** (a product-catalog team, a cart team, an inventory team) — the classic microservice boundary, not split by technical layer.

```
                                   ┌──────────────┐
                                   │    Client    │
                                   │ (web / app)  │
                                   └──────┬───────┘
                                          │
                             ┌────────────┴────────────┐
              images/static  │                         │  API (JSON)
                    ┌────────▼────────┐        ┌────────▼────────┐
                    │       CDN       │        │  API Gateway    │  auth, rate-limit,
                    │ (product pages, │        │  / Load Balancer│  routing
                    │  images, JS)    │        └────────┬────────┘
                    └─────────────────┘                 │
        ┌──────────────┬──────────────┬─────────────────┼───────────────┬───────────────┐
        ▼              ▼              ▼                 ▼               ▼               ▼
 ┌────────────┐ ┌────────────┐ ┌────────────┐   ┌────────────┐  ┌────────────┐ ┌────────────┐
 │  Catalog / │ │   Search   │ │    Cart    │   │   Order    │  │   Review   │ │   Recomm-  │
 │  Product   │ │  Service   │ │  Service   │   │  Service   │  │  Service   │ │  endation  │
 │  Service   │ │(Elastic-   │ │            │   │            │  │            │ │  Service   │
 └─────┬──────┘ │  search)   │ └─────┬──────┘   └─────┬──────┘  └─────┬──────┘ └─────┬──────┘
       │        └─────┬──────┘       │                │               │              │
       ▼              ▼              ▼                 ▼               ▼              ▼
 ┌────────────┐ ┌────────────┐ ┌────────────┐   ┌────────────┐  ┌────────────┐ ┌────────────┐
 │ Product DB │ │  Search    │ │ Cart KV    │   │ Order DB   │  │ Review DB  │ │ Rec store  │
 │ (document  │ │  index     │ │ (DynamoDB/ │   │(relational,│  │ (document) │ │ (precomp.) │
 │  store)    │ │            │ │  Redis)    │   │  STRONG)   │  └────────────┘ └────────────┘
 └────────────┘ └────────────┘ └────────────┘   └─────┬──────┘
                                                       │  the buy path binds these three:
       ┌───────────────────────────────────────────────┴─────────────────┐
       ▼                                     ▼                            ▼
 ┌────────────┐                       ┌────────────┐               ┌────────────┐
 │ Inventory  │◀── reserve / commit ──│  Checkout  │── charge ────▶│  Payment   │
 │  Service   │      / release        │Orchestrator│               │  Service   │
 │ (STRONG,   │                       │  (SAGA)    │               │ (Stripe/…) │
 │  the clip- │                       └─────┬──────┘               └────────────┘
 │  board)    │                             │
 └────────────┘                             ▼
                                     ┌────────────┐
                                     │Notification│  email/SMS/push: "order confirmed"
                                     │  Service   │  (async, best-effort)
                                     └────────────┘
```

**Reading the map:**
- **The left half is the browse world** (Catalog, Search) — read-heavy, cache-fronted, eventually consistent. Fed by the CDN.
- **The right half is the buy world** (Cart → Checkout → Inventory + Payment + Order) — write-heavy, strongly consistent, guarded.
- The **Checkout Orchestrator** is the interesting box: it coordinates Inventory, Payment, and Order as a **saga**.
- **Everything is a separate service** so each can scale, fail, and be reasoned about independently — and so that "recommendations is down" never means "you can't check out."

---

### 4. Product catalog and the browse path

This is the easy 80% — get it right quickly so you have time for the hard part.

**The catalog is a great fit for a document store.** Products have wildly varying attributes: a book has an author and ISBN; a t-shirt has size and color; a laptop has RAM and CPU. Forcing that into a fixed relational schema means either 200 mostly-null columns or an EAV nightmare. A **document store (MongoDB/DynamoDB)** lets each product be a self-describing JSON document. Query by `productId`, which is what the product page always does.

```json
{
  "productId": "B08...",
  "title": "Noise-Cancelling Headphones",
  "brand": "Acme",
  "price": { "amount": 19900, "currency": "USD" },
  "attributes": { "color": "black", "batteryHours": 30, "wireless": true },
  "images": ["cdn://.../1.jpg", "cdn://.../2.jpg"],
  "categoryPath": ["Electronics", "Audio", "Headphones"]
}
```

**Cache aggressively — this is a ~50,000 QPS read path.** Three layers:
1. **CDN** caches the rendered product page and all images at the edge. A hot product's page is served from a datacenter near the user in ~20ms, never touching your origin. Prices change rarely, so a short TTL (minutes) plus event-based purge on price change is plenty.
2. **Redis** caches the product document (`product:{id}`) in front of the Product DB, absorbing the CDN misses. ~99% hit rate.
3. The **Product DB** itself only sees the cold tail.

Recall [59 — Caching in Depth](./59-caching-in-depth.md): this is textbook cache-aside on immutable-ish data, with the extra wrinkle that **availability ("In stock" / "Only 3 left") must NOT be baked into the cached page**, because that number lives on the other side of the consistency wall. Render availability from a **separate, fresher call to the Inventory Service** (or a short-TTL counter), so a stale product page never shows stale stock.

**Search is a separate index, not a DB query.** Recall [79 — Search Systems](./79-search-systems.md). You do **not** run `WHERE title LIKE '%headphones%'` against the catalog DB — that doesn't scale, can't rank, and can't do typo-tolerance or facets. Instead:
- The catalog DB is the **source of truth**.
- A pipeline (CDC / events off the catalog) continuously indexes a **denormalized subset** into **Elasticsearch**.
- Search queries hit Elasticsearch for relevance ranking, faceted filters (brand, price range, rating), and autocomplete; the results are `productId`s that you then hydrate from the product cache.
- The index is **eventually consistent** with the catalog — a new product being searchable 30 seconds after it's added is completely fine.

---

### 5. The cart

The cart is a deceptively interesting design choice, and a good place to show judgment about *what guarantees you actually need*.

**Observations about carts:**
- They are **high-write**: every add/remove/quantity-change is a write, and people fiddle constantly.
- They are **ephemeral and low-stakes**: an abandoned cart is the norm, not an error. **It is OK to lose a cart** occasionally — annoying, not catastrophic. Nobody's money moved.
- They are strictly **per-user** and always accessed by user key.

That profile screams **"put it in a fast KV store, not the relational DB."** Storing carts in Postgres wastes your most precious, hardest-to-scale resource (the transactional DB) on high-churn, low-value data.

**Design:** store the cart in **DynamoDB or Redis**, keyed by `cart:{userId}`, as a small document of line items (`productId`, `quantity`, `addedAt`). DynamoDB gives durability + auto-scaling; Redis gives raw speed (often both: Redis as cache, DynamoDB as the durable store).

```json
// cart:{userId}
{ "items": [ { "productId": "B08...", "qty": 2 }, { "productId": "B09...", "qty": 1 } ],
  "updatedAt": "2026-07-18T10:00:00Z" }
```

> **Crucial: the cart holds NO inventory reservation.** Adding to cart does **not** reserve stock — otherwise everyone window-shopping would lock up all the inventory. The cart is just a wish list. Stock is only checked-and-reserved at **checkout** (section 6). A common junior mistake is reserving on add-to-cart; say explicitly that you don't.

**The merge problem.** A logged-out user builds a cart (keyed by an anonymous session/device ID). Then they log in — and they may *already* have a saved cart from last week on their phone. Now you have two carts and must **merge** them into one:
- Union the line items by `productId`.
- On a conflict (same product in both), pick a rule: `max(quantity)` is friendlier than summing (summing surprises people); make it a deliberate, documented choice.
- Merge is a simple, idempotent set-union — do it on the login event, then delete the anonymous cart.

The merge is a small thing, but naming it unprompted signals you've actually thought about real user flows, not just the happy path.

---

### 6. The HARD part — inventory and checkout (go deep here)

Everything above was warm-up. **This is the question.** It's where strong consistency is non-negotiable, and it's the deliberate opposite of the eventually-consistent browse path. Make that contrast explicit: *"the product page can be an hour stale and I don't care; the inventory count cannot be wrong by even one unit."*

#### 6a. The oversell problem

The scenario: a hot item has **100 units left**. A flash sale drops. **10,000 people click Buy within the same second.** You must sell **exactly 100** — not 100 plus an oversell you have to apologize for and refund.

**The naive, broken approach — check then decrement:**
```js
// ❌ BROKEN: a read-modify-write race (a TOCTOU bug)
const row = await db.query('SELECT available FROM inventory WHERE product_id = $1', [pid]);
if (row.available > 0) {                       // 10,000 requests all read "available = 1"
  await db.query('UPDATE inventory SET available = available - 1 WHERE product_id = $1', [pid]);
  // ...and 10,000 of them decrement. available goes to -9,999. You oversold by ~10,000.
}
```
The window between the `SELECT` and the `UPDATE` is the bug. Under high concurrency, thousands of requests read the same "available = 1" before any of them writes. This is a **read-modify-write race** — recall [66 — Database Transactions & Isolation](./66-database-transactions-and-isolation.md); without the right isolation or guard, the check and the write are not atomic together.

**Correct approach 1 — atomic conditional decrement (the workhorse).** Push the check *into* the write, and let the database's row lock serialize it:
```js
// ✅ The decrement and the "> 0" check are ONE atomic statement.
async function decrementStock(pid, qty) {
  const result = await db.query(
    `UPDATE inventory
        SET available = available - $2
      WHERE product_id = $1
        AND available >= $2`,     // the guard lives INSIDE the write
    [pid, qty]
  );
  // rowCount === 0  ⇒  the guard failed (not enough stock). We sold nothing. Reject.
  if (result.rowCount === 0) {
    throw new OutOfStockError(pid);
  }
  return { ok: true };
}
```
The database takes a row lock for the `UPDATE`, so the 10,000 requests are **serialized on that one row**. Exactly 100 of them find `available >= 1` and succeed; the 101st onward see `rowCount === 0` and are cleanly rejected. **No oversell, ever.** The `WHERE available >= qty` guard is the entire trick — never separate the check from the decrement.

**Correct approach 2 — optimistic locking with a version column.** Read the row and its `version`; on write, require the version to be unchanged, and retry on conflict:
```js
// ✅ Optimistic concurrency control — good when you must compute in app code between read and write.
async function reserveOptimistic(pid, qty) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const { available, version } = await db.getInventory(pid);
    if (available < qty) throw new OutOfStockError(pid);
    const result = await db.query(
      `UPDATE inventory SET available = available - $2, version = version + 1
        WHERE product_id = $1 AND version = $3`,   // only if nobody changed it since our read
      [pid, qty, version]
    );
    if (result.rowCount === 1) return { ok: true };  // we won
    // rowCount 0 ⇒ someone else wrote first; loop and retry with fresh data
  }
  throw new ConflictError('too much contention');
}
```
Great for low-to-moderate contention. Under a 10,000-way stampede on one row it degrades (lots of retries) — that's when the single-statement conditional decrement (approach 1) or a **reservation with a hold** (below) wins. Recall [84 — Distributed Locking](./84-distributed-locking.md) for when you need to coordinate a resource that *isn't* a single DB row; here the DB row lock is the cheapest correct lock available, so prefer it.

**Correct approach 3 — reserve with a hold + TTL (best for checkout).** Instead of decrementing `available` at click-time, move stock into a `reserved` bucket with an expiry. This is the one we build the checkout around, next.

> **This is the consistency wall in one paragraph.** The product page said "In stock" and was allowed to be stale. The decrement above is **not** allowed to be stale by even one unit, so it lives on a single strongly-consistent row with a lock around it. Same product, two different worlds, on purpose.

#### 6b. Reserving inventory during checkout (the hold)

When a user *starts* checkout, they need a few seconds (sometimes minutes) to enter shipping and card details. During that window, two people must not both "buy the last unit." So on checkout start, place a **temporary hold**:

```
inventory row:   available = 100   reserved = 0

  user starts checkout, qty 1:
      available -= 1  → 99          reserved += 1 → 1
      write a reservation { id, productId, qty:1, userId, expiresAt: now + 10min }

  ── if the user completes payment ──
      reserved -= 1  (the hold is consumed; stock is truly gone)

  ── if the user abandons / payment fails / TTL expires ──
      available += 1  → 100         reserved -= 1 → 0   (stock returns to the pool)
```

The hold uses the **same atomic conditional decrement** as 6a to move a unit from `available` into a reservation — so the hold itself can't oversell. A **sweeper job** (or a TTL index) scans for expired reservations and releases them back to `available`.

**The timeout trade-off** is a real decision, name it:
- **Too short** (e.g. 60s) → users with slow connections or a fumbled card get their item snatched away mid-checkout. Frustrating, lost sales.
- **Too long** (e.g. 1 hour) → during a flash sale, abandoned holds lock up scarce stock that willing buyers can't reach. Artificial scarcity, lost sales.
- **Typical: 10–15 minutes**, shorter during flash sales. Tune per product velocity.

#### 6c. The checkout as a distributed transaction — the SAGA

Checkout spans **multiple services** (Inventory, Payment, Order, Notification), each with its own database. There is **no distributed ACID transaction** across them — you can't `BEGIN` across Stripe and your Order DB. Recall [77 — Distributed Transactions](./77-distributed-transactions.md): the practical answer at this scale is a **saga** — a sequence of local transactions, each with a **compensating action** that undoes it if a later step fails.

We use an **orchestrated** saga: a Checkout Orchestrator drives the steps and issues compensations. The steps:

```
   reserve inventory ──▶ charge payment ──▶ create order ──▶ commit inventory ──▶ notify
        (hold)               (idempotent)       (PAID)         (consume hold)     (async)

   compensations (run in reverse if a step fails):
        release hold   ◀──   refund      ◀──  cancel order  ◀──   (re-add stock)
```

**The two guardrails that make it correct:**
1. **Idempotency on the payment step** (recall [85 — Idempotency](./85-idempotency.md)). Networks retry. If the "charge" request times out, you don't know whether the charge happened — retrying naively could **double-charge**. So every checkout generates a unique **idempotency key**; the Payment Service (and Stripe) dedupe on it: the same key charges **at most once**, no matter how many retries. This is the single most important line in the whole flow.
2. **Compensations, not rollbacks.** You can't "un-charge" a card with a rollback — you issue a **refund** (a new forward action that reverses the effect). You can't rollback a released inventory hold — you re-acquire stock. Sagas replace rollback with *semantic undo*.

Here is the orchestrator in Node.js:

```js
// checkout-saga.js — orchestrated saga for one checkout.
async function checkout({ userId, cartItems, paymentMethod }) {
  const sagaId = uuid();                      // ← doubles as the payment idempotency key
  const done = [];                            // stack of compensations to run on failure

  try {
    // ── Step 1: RESERVE inventory (atomic hold per item) ──────────────────
    const holds = [];
    for (const item of cartItems) {
      const hold = await inventory.reserve(item.productId, item.qty, {
        userId, ttlSeconds: 900,
      });                                      // uses the conditional decrement of 6a
      holds.push(hold);
      done.push(() => inventory.release(hold.id));   // compensation: release the hold
    }

    // ── Step 2: CHARGE payment (IDEMPOTENT — the retry-safety linchpin) ────
    const amount = priceOf(cartItems);
    const charge = await payment.charge({
      userId, amount, paymentMethod,
      idempotencyKey: sagaId,                  // ← retry with same key ⇒ charges at most ONCE
    });
    done.push(() => payment.refund({ chargeId: charge.id, idempotencyKey: `refund-${sagaId}` }));

    // ── Step 3: CREATE the order record (strongly-consistent relational DB) ─
    const order = await orders.create({
      orderId: sagaId, userId, items: cartItems, chargeId: charge.id, status: 'PAID',
    });
    done.push(() => orders.cancel(order.orderId));

    // ── Step 4: COMMIT inventory (consume the holds — stock is now truly gone)
    for (const hold of holds) await inventory.commit(hold.id);
    // (no compensation needed past the point of no return; the order is PAID + stocked)

    // ── Step 5: NOTIFY (async, best-effort — must NOT fail the checkout) ────
    notifications.send({ userId, type: 'ORDER_CONFIRMED', orderId: order.orderId })
      .catch(err => log.warn('notify failed, will retry async', err));  // fire-and-forget

    return { ok: true, orderId: order.orderId };

  } catch (err) {
    // ── COMPENSATE: run undo actions in REVERSE order (LIFO) ───────────────
    for (const undo of done.reverse()) {
      try { await undo(); }
      catch (e) { log.error('compensation failed — needs reconciliation', e); }
      // a failed compensation is logged for a reconciliation job; never swallowed silently.
    }
    throw new CheckoutFailedError(sagaId, err);
  }
}
```

**Reading the saga:**
- Each forward step pushes its **undo** onto a stack, so failure at step 3 unwinds steps 2 and 1 in reverse (cancel order → refund → release holds).
- **Payment is idempotent**, so the orchestrator can safely retry the charge on a timeout without risk of double-charging.
- **Notification is fire-and-forget** — the order is already confirmed; if the email fails, a background retry sends it later. A notification failure must never fail a paid order.
- **Failed compensations are logged, not swallowed** — they feed a reconciliation job, because "we charged but couldn't create the order" is a money bug that a human/automated process must resolve.

---

### 7. Order service

Once payment succeeds, the order is a **permanent, legal, financial record** — this is the one place you want a **strongly-consistent relational database** (Postgres/MySQL, or a NewSQL like Spanner/CockroachDB at scale). Money and legal records demand ACID, auditability, and the ability to run correct reports. This is the deliberate opposite of the cart (KV, disposable) and the catalog (document, cacheable).

**The order lifecycle is a state machine.** Orders move through well-defined states, and only certain transitions are legal:

```
                    ┌─────────┐
                    │ CREATED │  cart converted, saga started
                    └────┬────┘
                         │ payment succeeds
                         ▼
                    ┌─────────┐        cancel (before shipping)
                    │  PAID   │ ─────────────────────────────┐
                    └────┬────┘                               │
                         │ warehouse picks & ships            ▼
                         ▼                              ┌───────────┐
                    ┌─────────┐                         │ CANCELLED │
                    │ SHIPPED │                         │ (→ refund)│
                    └────┬────┘                         └───────────┘
                         │ carrier confirms delivery
                         ▼
                    ┌───────────┐        customer returns item
                    │ DELIVERED │ ───────────────────────────┐
                    └───────────┘                             ▼
                                                        ┌───────────┐
                                                        │ RETURNED  │
                                                        │ (→ refund)│
                                                        └───────────┘
```

- Store the current `status` plus an **append-only event log** of transitions (`created_at`, `paid_at`, `shipped_at`, …) — you almost always need "when did this order ship?" and an audit trail.
- **Enforce legal transitions in code**: you cannot go `CREATED → DELIVERED` directly, and you cannot `CANCEL` an order that already `SHIPPED` (that's a return, not a cancellation). Guard transitions with a state-machine check.
- CANCELLED and RETURNED both trigger a **refund** (a compensation, exactly like the saga's) and a **stock re-add** to inventory.

---

### 8. Deep dives

#### (a) Flash sale / Black Friday handling

A flash sale is the oversell problem (6a) at maximum intensity plus a traffic spike, and it's **predictable** — which is your biggest advantage.

- **Pre-scale.** You know Prime Day's date. Scale out API, cache, and DB read replicas *in advance*; don't rely on reactive autoscaling that lags the spike.
- **Cache the product page harder than ever.** The hot item's page will be hit millions of times. Serve it fully from the CDN; the origin should see almost none of it.
- **Put a queue / virtual waiting room in front of checkout.** When 500,000 people rush 1,000 units, admit buyers to the checkout flow at a controlled rate (a token/queue) so the inventory row isn't hammered by a 500,000-way stampede. Most will be told "sold out" — which is *correct*, there were only 1,000 units.
- **The inventory row is the bottleneck** and it's a single hot row. Mitigations: the atomic conditional decrement (6a) keeps it *correct* under contention; to keep it *fast*, some designs shard the counter (split 1,000 units into 10 buckets of 100, decrement a random bucket) to spread the lock, or pre-load the count into an atomic Redis counter (`DECR`, reject at ≤ 0) as a fast gate in front of the DB.
- **Shed load gracefully:** better to show "high demand, try again" than to fall over.

#### (b) Search and recommendations (brief)

- **Search:** covered in section 4 — a separate Elasticsearch index fed asynchronously from the catalog, eventually consistent, handling ranking/facets/typo-tolerance. Recall [79 — Search Systems](./79-search-systems.md).
- **Recommendations** ("people also bought," "recently viewed") are **precomputed, eventual, and best-effort.** Batch/stream jobs mine order and click history offline and write per-user and per-product recommendation lists into a fast store. They are read on the product page but are **never on the critical buy path** — if the rec service is down, the page renders without the carousel and the sale still happens.

#### (c) The consistency spectrum across the system (the senior-signal deep dive)

Making this explicit is *the* thing that separates a strong candidate. Draw the wall:

| Subsystem | Consistency | Why |
|---|---|---|
| **Inventory decrement** | **STRONG** | Cannot oversell the last unit. Atomic, locked, single source of truth. |
| **Payment** | **STRONG + idempotent** | Cannot double-charge. Exactly-once effect via idempotency keys. |
| **Orders** | **STRONG (ACID)** | Money and legal records. Auditable, correct. |
| Cart | Read-your-writes, durable-ish | You want to see what you added; losing it occasionally is OK. |
| Product catalog / price | **EVENTUAL** (short TTL) | A minutes-stale price/description is fine; cache hard. |
| Search index | **EVENTUAL** | New product searchable in ~seconds is fine. |
| Reviews | **EVENTUAL** | A review appearing 10s late harms no one. |
| Recommendations / recently-viewed | **EVENTUAL, best-effort** | Approximate and optional; never blocks a purchase. |

> **One product, a spectrum of guarantees.** Spend your (expensive, hard-to-scale) strong-consistency budget only on the three rows at the top. Everything else gets the cheap, fast, scalable eventual-consistency treatment. Saying this sentence, and defending which side each feature is on, is the whole point of the interview.

#### (d) The thundering herd on a hot product page

When a product suddenly goes viral (a celebrity tweets it) and its cache entry **expires at the same moment** it's getting 100,000 req/s, all those requests miss the cache simultaneously and **stampede the Product DB** — which can take the DB down. Recall [59 — Caching in Depth](./59-caching-in-depth.md). Fixes:
- **Single-flight / request coalescing:** only *one* request per key is allowed to recompute on a miss; the rest wait for its result. 100,000 misses become 1 DB read.
- **Stale-while-revalidate:** serve the slightly-stale cached value while one background task refreshes it — readers never see a miss.
- **Jittered / staggered TTLs** so a whole category of products doesn't expire in lockstep.

#### (e) Failure modes and graceful degradation

- **Payment provider down:** don't fail hard. **Queue the charge and retry** with backoff (the idempotency key makes retries safe), and either hold the order in a `PENDING_PAYMENT` state or fail fast to the user with "try again" — never double-charge, never silently lose the order.
- **Inventory DB hot row:** the single row for a viral item becomes a lock-contention hotspot. Mitigate with sharded counters / a Redis atomic pre-gate (deep dive a), and keep the critical section a single atomic statement.
- **Graceful degradation is the theme:** if recommendations, reviews, or "recently viewed" are down, the product page still renders and the customer **can still buy.** Never let a best-effort subsystem block the money path. Circuit-break calls to non-critical services so their slowness can't stall checkout.

---

## Visual / Diagram description

The three diagrams you must be able to redraw from memory:

1. **The subsystem map** (section 3) — a dozen services split by business capability, with the **browse world on the left** (Catalog, Search, fed by the CDN, cacheable/eventual) and the **buy world on the right** (Cart → Checkout Orchestrator → Inventory + Payment + Order, strongly consistent). The Checkout Orchestrator binding Inventory/Payment/Order is the visual heart of the buy path.

2. **The checkout saga** (section 6c) — the forward chain `reserve → charge → create order → commit → notify` with the compensation chain `release ← refund ← cancel` running in reverse underneath. Mark **payment as idempotent** and **notify as async/best-effort**.

3. **The order state machine** (section 7) — `CREATED → PAID → SHIPPED → DELIVERED`, with the two exit branches `CANCELLED` (from PAID) and `RETURNED` (from DELIVERED), both triggering a refund.

The single most important line in all three isn't a box — it's the **consistency wall** running down the middle of diagram 1: stale-OK on the left, exact-or-bust on the right.

---

## Real world examples

### Amazon

Amazon's own architecture is the canonical example of **service-per-capability**: famously, Jeff Bezos's "API mandate" forced every team to expose its data only through service interfaces — which is *why* the catalog, cart, inventory, orders, and payments are independent services with independent datastores. **DynamoDB was literally born from the shopping-cart problem**: Amazon needed an always-writable, highly-available cart store that never rejected an "add to cart" even during partitions — an availability-over-consistency choice that's perfect for a cart and wrong for inventory. That single company demonstrates both sides of the wall in one product.

### Shopify

Shopify runs millions of independent stores and leans hard on the strong-consistency-where-it-counts principle: inventory and checkout are guarded transactional paths (built on sharded MySQL), while storefront rendering is heavily cached and CDN-fronted. Their public engineering writing on **Black Friday / Cyber Monday** is a masterclass in the flash-sale deep dive — pre-scaling, load shedding, and protecting the checkout path from the browse stampede.

### Stripe

Stripe is the reference implementation of **idempotency in payments**: every charge request takes an `Idempotency-Key` header, and Stripe guarantees the same key produces the same result — charging at most once no matter how many times a flaky network makes you retry. This is exactly the guarantee the checkout saga (6c) relies on, and it's why "just integrate a payment provider" is the right interview answer rather than "build a card processor."

---

## Trade-offs

**The central trade-off: one consistency model, or a spectrum?**

| | One model everywhere | A spectrum (what we chose) |
|---|---|---|
| Strong everywhere | Simple mental model; **can't scale the browse path**, falls over at spikes | — |
| Eventual everywhere | Scales beautifully; **oversells and double-charges** — unacceptable | — |
| **Per-subsystem** | — | **Strong for money/stock, eventual for browse/reviews/recs.** More complex, but each part gets exactly the guarantee it needs. |

**Other trade-offs made in this design**

| Decision | You gain | You give up |
|---|---|---|
| Document store for the catalog | Flexible per-product schema, fast key lookups | Rich cross-product relational queries (push those to search) |
| Cache the browse path hard (CDN + Redis) | ~50k QPS served cheaply, low latency | Prices/availability can be slightly stale (render stock separately) |
| Cart in a KV store, no reservation | Cheap, fast, scalable; DB stays free for money | Occasional lost cart; a merge problem on login |
| Reserve inventory with a TTL hold | Two buyers can't grab the last unit mid-checkout | A timeout to tune; a sweeper job; brief artificial scarcity |
| Checkout as a saga (not distributed ACID) | Works across independent services, stays available | No atomic rollback — you write compensations by hand |
| Idempotent payment | Retry-safe; **no double-charge** | Must generate and track idempotency keys |
| Orders in a strong relational DB | ACID, auditable, correct money records | Lower write throughput than a KV store (fine — only ~60 orders/sec) |
| Recommendations best-effort | Buy path never blocked by a non-critical service | Recs can be stale or briefly absent |

**The sweet spot:** **Cache the browse path into oblivion; guard the buy path with atomic decrements, TTL reservations, an idempotent-payment saga, and a strongly-consistent order store.** Two worlds, one consistency wall between them.

**Rule of thumb:** For every feature, ask *"what happens if this is stale or lost?"* If the answer is "a customer is mildly annoyed," it goes on the cheap eventual side. If the answer is "we oversold, double-charged, or lost a legal record," it goes on the expensive strong side. Put the wall exactly there.

---

## Common interview questions on this topic

### Q1: "10,000 people buy the last 100 units at once. How do you not oversell?"

**Hint:** Reject the naive check-then-decrement immediately — name the read-modify-write race. Then give the **atomic conditional decrement**: `UPDATE inventory SET available = available - 1 WHERE product_id = ? AND available >= 1`, and **reject if `rowCount === 0`**. The DB row lock serializes the 10,000 requests; exactly 100 win. Mention optimistic locking (version column) and a TTL reservation as the checkout-time variant. End with the punchline: *"the guard lives inside the write — never separate the check from the decrement."*

### Q2: "A user's card gets charged but the order fails to save. What now?"

**Hint:** This is why checkout is a **saga**, not a distributed transaction. Each step has a compensation; a failure after "charge" but at "create order" triggers a **refund** (a forward action, since you can't rollback a charge) and releases the inventory hold. And you make payment **idempotent** with a key so a retry of a timed-out charge never double-charges. Add: failed compensations aren't swallowed — they go to a reconciliation queue, because that's a money bug.

### Q3: "Why is the cart in DynamoDB/Redis and not your main SQL database?"

**Hint:** Carts are high-write, ephemeral, low-stakes, and always accessed by user key — the exact profile for a KV store. Losing a cart occasionally is fine; you don't want to burn your scarce transactional DB capacity on high-churn wish-list data. Mention: the cart holds **no inventory reservation** (that would lock stock for window-shoppers), and note the **merge problem** when a logged-out cart meets a logged-in one.

### Q4: "The product page shows 'In stock' but it's cached. Isn't that a consistency bug?"

**Hint:** The right answer is that **availability is NOT baked into the cached page** — the page is cached (eventual, fine) but the stock indicator is a separate, fresher call to the Inventory Service (or a short-TTL counter). This is the consistency-wall answer: the *description and price* can be stale, but the *decision to sell* is made against the strongly-consistent inventory row at checkout, so a stale "In stock" at worst means a rejected checkout, never an oversell.

### Q5: "It's Prime Day and one item is going viral. Walk me through what breaks and how you protect it."

**Hint:** Two problems: the **hot product page** (thundering herd → single-flight + stale-while-revalidate + CDN) and the **hot inventory row** (lock contention → atomic single-statement decrement for correctness, plus sharded counters / a Redis atomic pre-gate for speed, plus a **queue/waiting room** in front of checkout to throttle the stampede). Pre-scale because the date is known. And **graceful degradation**: if recs/reviews are down, people can still buy.

---

## Practice exercise

### Build the inventory core and the checkout saga

Write a working Node.js prototype. Use a real Postgres (`docker run -p 5432:5432 -e POSTGRES_PASSWORD=x postgres`) for inventory and orders, and a stub for the payment provider.

**Part 1 — the atomic inventory service.** Implement, backed by an `inventory(product_id, available, reserved, version)` table:
- `reserve(productId, qty, { ttlSeconds })` — atomically move `qty` from `available` into a reservation row with an `expires_at`; reject if insufficient. Use the **conditional decrement** (`WHERE available >= qty`), and return `rowCount === 0` as `OutOfStockError`.
- `release(reservationId)` — return the held qty to `available` (idempotent: releasing twice is a no-op).
- `commit(reservationId)` — consume the hold permanently (decrement `reserved`).
- A **sweeper** that scans for `expires_at < now()` reservations and releases them.

**Part 2 — the checkout saga.** Implement `checkout({ userId, cartItems, paymentMethod })` following the orchestrator in section 6c:
1. Reserve every item (pushing a `release` compensation per item).
2. Charge a **stub payment** that takes an `idempotencyKey` and dedupes on it — prove that calling it twice with the same key charges once.
3. Create a `PAID` order row in Postgres.
4. Commit the holds.
5. On any failure, run compensations in **reverse (LIFO)** and assert the system is back to its starting state (stock restored, refund issued, order cancelled).

**Part 3 — prove it doesn't oversell.** Seed one product with `available = 100`. Then fire **1,000 concurrent** `checkout` calls (`Promise.all`) for `qty = 1`.
- Assert **exactly 100** succeed and **exactly 900** get `OutOfStockError`.
- Assert the final `available === 0` and `reserved === 0` (all holds either committed or released).
- Now **break payment on ~half the successful reservations** (make the stub throw) and assert those holds are **released** (stock returns) and no order was created for them — the saga compensated.

**Deliverable:** the inventory service, the saga, and a one-paragraph write-up answering: *how many units did you sell, and how do you know the atomic conditional decrement prevented the oversell that a check-then-decrement would have caused?* If you can't demonstrate the race with the naive version and its absence with the atomic version, you can't defend the design in an interview.

---

## Quick reference cheat sheet

- **E-commerce is many subsystems, not one** — catalog, search, cart, inventory, order, payment, reviews, recs, notifications. Draw the map, then **go deep on inventory + checkout.**
- **The consistency wall is the whole design.** **STRONG** for inventory, payment, orders. **EVENTUAL** for catalog, search, reviews, recs, recently-viewed. Say which side each feature is on.
- **Browse dwarfs buy** (~800:1). Cache the browse path into oblivion: **CDN → Redis → DB.** Render availability *separately* so a stale page never shows stale stock.
- **Catalog → document store** (varying per-product attributes). **Search → separate Elasticsearch index**, fed async, eventually consistent. Never `LIKE` the catalog DB.
- **Cart → KV store** (DynamoDB/Redis), keyed by user. High-write, ephemeral, OK-to-lose. **No inventory reservation on add-to-cart.** Handle the **login merge**.
- **The oversell fix = atomic conditional decrement:** `UPDATE inventory SET available = available - 1 WHERE product_id = ? AND available >= 1`; **reject if 0 rows.** The guard lives *inside* the write. Never check-then-decrement.
- Alternatives: **optimistic locking (version column)**; **reserve with a TTL hold** during checkout so two buyers can't take the last unit while one enters a card.
- **Checkout = SAGA:** reserve → charge → create order → commit → notify, with **compensations** (release, refund, cancel) run in reverse on failure. You can't rollback across services — you undo semantically.
- **Payment must be IDEMPOTENT** (idempotency key) so a retry never double-charges. This is the single most important line in checkout.
- **Orders → strong relational DB** (ACID; money + legal records). Lifecycle state machine: `CREATED → PAID → SHIPPED → DELIVERED`, with `CANCELLED`/`RETURNED` → refund.
- **Flash sale:** pre-scale, cache the page hard, **queue/waiting-room** in front of checkout, atomic decrement + sharded/Redis counter for the hot row.
- **Thundering herd** on a hot page: single-flight, stale-while-revalidate, jittered TTLs.
- **Graceful degradation:** if recs/reviews are down, people **can still buy.** Never let a best-effort subsystem block the money path.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Related** | [100 — Design Instagram](./100-hld-instagram.md) — the other big case study; contrast its all-eventual feed with this system's strong/eventual split |
| **Related** | [66 — Database Transactions & Isolation](./66-database-transactions-and-isolation.md) — the read-modify-write race behind oversell and why atomicity fixes it |
| **Related** | [77 — Distributed Transactions](./77-distributed-transactions.md) — the saga pattern that drives checkout across independent services |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — the browse-path caches and the thundering-herd defenses |
| **Related** | [79 — Search Systems](./79-search-systems.md) — the separate Elasticsearch index that powers product search |
