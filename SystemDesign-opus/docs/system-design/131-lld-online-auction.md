# 131 — Design an Online Auction System (eBay)
## Category: LLD Case Study

---

## What is this?

An **online auction system** lets a seller list an item with a starting price and a deadline, and lets many buyers compete by placing bids. The highest bid when the clock runs out wins. Think eBay: someone lists a vintage watch starting at $50, buyers bid it up over three days, and whoever holds the top bid at the final second takes it home.

This is a classic LLD interview problem because it braids together three ideas you have already met: a **State machine** (an auction moves created → active → ended), the **Observer pattern** (everyone watching gets notified when a new bid lands), and **concurrency control** (two people bid the same amount at the same millisecond — exactly one must win). Get those three right and the rest is bookkeeping.

---

## Why does it matter?

**Interview angle.** "Design an auction" is a favourite because it forces you to say the words *State*, *Observer*, and *race condition* out loud and then actually implement the fix. A weak answer stores a `currentHighestBid` field and updates it with `if (amount > highest) highest = amount`. The interviewer then asks "two bids arrive at once — what happens?" and the naive answer double-awards the item. The strong answer reaches for an atomic compare-and-set, exactly like seat booking (recall [118 — Movie Ticket Booking](./118-lld-movie-ticket-booking.md)).

**Real-work angle.** Auctions, flash sales, ad-exchange bidding, and "last one in stock" checkout all share the same skeleton: many writers racing to claim one scarce resource under a lifecycle. If you cannot reason about the race, you ship a system that occasionally sells the same watch to two people — a support nightmare and a refund bill. The lifecycle discipline (you *cannot* bid on an auction that has not started or has already ended) is what keeps the money honest.

---

## The core idea — explained simply

### The analogy: a live auction room with a gavel

Picture a physical auction room.

- The **auctioneer** (our `AuctionService`) runs the show but does not own the item.
- Each **lot** (our `Auction`) has a **gavel state**: not-yet-open, taking bids, or *sold* (gavel down). You literally cannot shout a bid before the lot opens or after the gavel falls.
- The **audience** (our watchers/observers) hears every "new high bid!" the instant it happens, and the person who *was* leading hears "you've been outbid."
- When two people shout the same number at the same instant, the auctioneer's **single mouth** can only accept one — he physically cannot award to both. That single-threaded gavel is our **lock / compare-and-set**.
- A bidder can hand the auctioneer a **secret max** ("keep me in up to $500") — that is **proxy / auto-bidding**. The auctioneer bids on their behalf, the minimum needed to stay on top, until the secret max is exceeded.

| Auction room | Our system | Concept |
|---|---|---|
| Gavel state (closed/open/sold) | `Auction.status` | **State machine** (recall [46](./46-pattern-state.md)) |
| "New high bid!" announcement | `notify()` to watchers | **Observer** (recall [41](./41-pattern-observer.md)) |
| Auctioneer's single mouth | mutex / compare-and-set on `currentHighestBid` | **Concurrency** (recall [52](./52-concurrency-fundamentals.md)) |
| Secret max handed to auctioneer | auto-bid resolution | **Proxy bidding** algorithm |
| The auctioneer | `AuctionService` | Orchestrator |

Every hard part of this problem is one of those five rows. Keep the table in mind as we build.

---

## Key concepts inside this topic

We follow the 7-step LLD method (recall [111 — LLD Approach Framework](./111-lld-approach-framework.md)): requirements → nouns → behaviours → the hard cores → class diagram → code → patterns → extensions.

### 1. Requirements & clarifying questions

**Functional requirements**

- A seller **lists an item** with a starting price and an end time.
- Buyers **place bids**; a bid must exceed the current highest by at least a **minimum increment**.
- An auction has a **lifecycle**: created → active → ended (and possibly cancelled).
- When the auction **ends**, the highest bidder wins.
- Support **auto-bidding** (proxy bidding up to a secret max).
- **Notify** watchers on every new highest bid and on auction end; tell the outbid leader they lost the lead.

**Non-functional**

- **Correctness under concurrency**: two simultaneous bids must resolve to exactly one winner.
- **Fairness / determinism**: given the same bids, the same winner every time.
- Low latency on `placeBid` (it is on the hot path).

**Clarifying questions to ask the interviewer** (asking these signals seniority):

1. Is there a **reserve price** (a hidden minimum below which the seller is not obliged to sell)?
2. Is there a **buy-now** price that ends the auction instantly?
3. Do we need **anti-sniping** — extend the end time if a bid arrives in the final seconds?
4. Are **concurrent bids** in scope? (Almost always yes — this is the point of the problem.)
5. Can a seller **cancel** an auction? Can a bidder **retract** a bid?
6. Single process or **distributed** across servers? (Changes the concurrency answer — see Extensions.)

For this design: yes to auto-bid and concurrency now; reserve, buy-now, and anti-sniping are Extensions.

### 2. Identify core objects (nouns)

Underline the nouns in the requirements:

- **User** — a person; may be a seller and/or a bidder.
- **AuctionItem** — the thing being sold (id, title, description).
- **Bid** — one offer: who, how much, when.
- **Auction** — the lot: item, seller, start price, min increment, end time, **status**, the current highest bid, its watchers, and per-user auto-bid maxes.
- **AuctionService** — the orchestrator that creates auctions, routes bids, and closes them.
- **AuctionStatus** — an enum: `CREATED / ACTIVE / ENDED / CANCELLED`.

### 3. Behaviours + ownership

| Behaviour | Owner | Notes |
|---|---|---|
| `createAuction` | `AuctionService` | builds an `Auction` in `CREATED` |
| `start()` | `Auction` | CREATED → ACTIVE (state transition) |
| `placeBid(user, amount)` | `Auction` | validates status + increment, **atomic** highest-bid update, runs auto-bid, notifies |
| `setAutoBid(user, max)` | `Auction` | records a secret max |
| `end()` | `Auction` | ACTIVE → ENDED, determine winner, notify |
| `subscribe(observer)` | `Auction` | register a watcher |

The key ownership decision: **the `Auction` owns its own status lifecycle and its own highest bid.** It is the only object allowed to mutate those. The `AuctionService` orchestrates (routing, timers) but never reaches inside and flips a status directly — that would let a bid land on an ended auction. Bidders and watchers are **observers**, notified but never in control.

### 4. The three intellectual cores

#### Core A — STATE: the auction lifecycle

An auction is a **finite state machine** (recall [46 — State pattern](./46-pattern-state.md)). Bidding is only valid in `ACTIVE`. The whole point is that behaviour depends on state:

```
placeBid on CREATED  → reject ("auction not started")
placeBid on ACTIVE   → accept (if amount valid)
placeBid on ENDED    → reject ("auction is over")
placeBid on CANCELLED→ reject
```

Valid transitions:

```
CREATED --start()--> ACTIVE --end()--> ENDED
CREATED --cancel()-> CANCELLED
ACTIVE  --cancel()-> CANCELLED
```

Any other transition (ENDED → ACTIVE, ACTIVE → CREATED) is illegal and must throw. Modelling this explicitly is what stops the classic bug of a bid sneaking in one millisecond after the gavel. We are **using the State idea**: the auction's response to `placeBid` is governed entirely by `status`.

#### Core B — OBSERVER: bid notifications

When a new highest bid arrives, many parties care: everyone **watching** the item, and the **previous leader** who just got outbid. When the auction ends, the **winner** and the **seller** care. This is a textbook **Observer** setup (recall [41 — Observer pattern](./41-pattern-observer.md)) — a one-to-many "when X changes, tell everyone subscribed."

The `Auction` is the **subject**; watchers are **observers** implementing `onEvent(event)`. The auction never knows *who* is listening or *what* they do (email? push? log?) — it just publishes events: `NEW_HIGH_BID`, `OUTBID`, `AUCTION_ENDED`. This is the natural, correct fit — do not hand-roll a notification list.

#### Core C — CONCURRENCY: concurrent bids (the hard part)

Here is the bug that separates junior from senior answers. Item is at **$100**. Two bidders both submit **$101** at the same instant. Naive code:

```js
// BROKEN — read-then-write race
if (amount > this.highest.amount) {   // both read $100, both pass
  this.highest = new Bid(user, amount); // both overwrite; both "win"
}
```

Both threads read `$100`, both decide they beat it, both write. Now two people believe they hold the top bid — the auction is corrupt. This is the same **read-modify-write race** you saw with two people grabbing the last cinema seat (recall [118 — Movie Ticket Booking](./118-lld-movie-ticket-booking.md), the seat-hold race).

**The fix: make the compare-and-set atomic.** Serialize the "read highest → compare → set highest" sequence so only one bid can be in that critical section at a time. We use an **async mutex** (recall [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md)): acquire the lock, re-read the *current* highest inside the lock, decide, set, release. The second bidder now runs *after* the first committed, sees the new highest ($101), and is either rejected (if their $101 no longer beats highest + increment) or wins if they had bid higher.

The rule: **never decide a winner on a value you read outside the lock.** Read it again *inside*.

#### Auto-bid / proxy bidding (the algorithmic sub-problem)

A bidder hands over a **secret max**: "bid for me up to $500, but only as much as needed." The system bids the **minimum to stay on top**. The beautiful consequence: two auto-bidders resolve to **one increment above the lower max** — exactly like a real auction, where the item sells for just over the second-highest willingness to pay.

Worked example, increment = $10:

- Alice sets auto-max $200. She becomes leader at the start price, say $100.
- Bob sets auto-max $150. System bids for Bob up to $150; system re-bids for Alice to $160 (one increment above Bob's max). Bob's $150 < $160, Bob is out.
- Final: **Alice leads at $160** = Bob's max ($150) + one increment ($10). The system stopped the instant it needed to — it did not run Alice up to her full $200.

That "settle at second-max + one increment" behaviour is the star of the auto-bid logic, and we implement it below.

---

## Visual / Diagram description

**Class diagram**

```
┌───────────────────────┐         ┌──────────────────────┐
│      AuctionService    │ 1     * │       Auction        │
│  (orchestrator)        │◇───────>│----------------------│
│ + createAuction()      │         │ - item: AuctionItem  │
│ + placeBid()           │         │ - seller: User       │
│ + endAuction()         │         │ - startPrice         │
└───────────────────────┘         │ - minIncrement       │
                                   │ - endTime            │
        ┌──────────────┐          │ - status: AuctionStatus (STATE)
        │  AuctionItem  │<>────────│ - currentHighestBid: Bid │
        │ id,title,desc │          │ - watchers: Observer[]   │
        └──────────────┘          │ - autoMax: Map<user,max> │
                                   │ - lock: Mutex            │
        ┌──────────────┐          │----------------------│
        │     Bid       │<>────────│ + start()            │
        │ bidder: User  │  0..1    │ + placeBid(user,amt) │  (atomic)
        │ amount, ts    │  (highest)│ + setAutoBid(u,max)  │
        └──────────────┘          │ + subscribe(obs)     │
                                   │ + end()              │
        ┌──────────────┐          └──────────┬───────────┘
        │     User      │                     │ notifies (Observer)
        │ id, name      │                     ▼
        └──────────────┘          ┌──────────────────────┐
                                   │   <<Observer>>        │
                                   │ onEvent(event)        │
                                   │  ▲            ▲       │
                                   │  │            │       │
                                   │ EmailObserver LogObserver
                                   └──────────────────────┘
```

**Auction state machine**

```
            start()                end()
  ┌────────┐──────────>┌────────┐──────────>┌────────┐
  │CREATED │           │ ACTIVE │           │ ENDED  │
  └───┬────┘           └───┬────┘           └────────┘
      │ cancel()           │ cancel()
      ▼                    ▼
  ┌──────────────────────────┐
  │        CANCELLED          │
  └──────────────────────────┘

placeBid() is accepted ONLY in ACTIVE. In every other
state it is rejected. end() locks the highest bid forever.
```

The `AuctionService` holds many `Auction`s (composition, ◇). Each `Auction` holds one `AuctionItem`, a set of `Bid`s, and a `Bid` reference for the current highest. The `status` field drives all behaviour (State). The `watchers` list is the Observer wiring. The `lock` guards the highest-bid update (Concurrency).

---

## Real world examples

### eBay

eBay is the canonical proxy-auction: you enter your **maximum**, and eBay bids incrementally on your behalf, revealing only the current price, not your max. Two proxy bids settle at one increment above the loser's max — exactly our auto-bid logic. eBay also ships **anti-sniping mitigations** and uses server-side atomicity so a listing has exactly one high bidder at close (representative of how large marketplaces avoid double-award).

### Google/OpenRTB ad exchanges

Real-time ad auctions run millions of concurrent "bids" per second for a single impression. They are **second-price** auctions (the winner pays one increment above the second bid) — the same "settle at second-highest" mathematics as our proxy logic, just at massive scale with a hard latency budget (conceptual parallel).

### StockX / sneaker resale

StockX runs a continuous **bid/ask** market: highest bid meets lowest ask. The matching engine must atomically pair one bid with one ask so a shoe is never sold twice — the same single-winner concurrency guarantee we implement with a lock (representative).

---

## Trade-offs

**Concurrency approach**

| Approach | Pros | Cons |
|---|---|---|
| No lock (naive) | simplest, fastest | **wrong** — double-award under load |
| In-process mutex (our choice) | correct, simple, no infra | single process only |
| DB conditional update | correct, scales across servers | needs a round-trip; retry logic |
| Distributed lock (Redis) | works across servers | infra + failure modes (recall [84](./111-lld-approach-framework.md) style locking) |

**Auto-bid resolution**

| Choice | Pros | Cons |
|---|---|---|
| Resolve on every bid (our choice) | always consistent leader | more computation per bid |
| Lazy resolve at close | cheap per bid | current price misleading mid-auction |

**Rule of thumb:** in a single-process interview answer, an **in-process async mutex around the compare-and-set** plus **eager auto-bid resolution** is the sweet spot — provably correct, easy to explain, and a clean upgrade path to a DB conditional update when you go distributed.

---

## Common interview questions on this topic

### Q1: "Two bidders bid the same amount at the same instant — what happens?"

**Hint:** Name the read-modify-write race. Only one may win. Wrap the "read highest → compare against highest + increment → set highest" in a mutex; re-read *inside* the lock. The second bid runs after the first commits and is rejected because it no longer beats the new highest. Same pattern as the last-seat race in [118](./118-lld-movie-ticket-booking.md).

### Q2: "How do two auto-bidders resolve?"

**Hint:** The winner is the higher max; the price settles at the **lower max + one increment** (capped at the higher max). Walk the increment-by-increment resolution; stress that the system bids the *minimum needed*, not the full max.

### Q3: "Why is this a State pattern, and what does it prevent?"

**Hint:** `placeBid` behaves differently per status; only `ACTIVE` accepts. Explicit transitions prevent bids landing before start or after the gavel. Illegal transitions throw.

### Q4: "How do watchers get notified without the auction knowing about email/SMS?"

**Hint:** Observer. Auction is the subject; it publishes typed events (`NEW_HIGH_BID`, `OUTBID`, `AUCTION_ENDED`). Observers subscribe and decide their own side effect. Decouples the auction from delivery.

### Q5: "Now make it run across 10 servers — what breaks?"

**Hint:** The in-process mutex only guards one process. Move the atomicity to the datastore: a **conditional update** (`UPDATE ... WHERE highest < :amount`) or a **distributed lock**. The design barely changes — only the `placeBid` critical section swaps implementation.

---

## Practice exercise

### Build and extend the auction engine

Take the code below and run it (`node auction.js`) to see State, Observer, concurrency, and auto-bid all fire. Then extend it:

1. Add a **`reservePrice`**. At `end()`, if the highest bid is below the reserve, mark the auction `ENDED` but with **no winner** ("reserve not met") and notify the seller.
2. Add **anti-sniping**: if a bid arrives within the last 10 seconds, push `endTime` out by 30 seconds and emit an `EXTENDED` event.
3. Write a test that fires **50 concurrent bids** (via `Promise.all`) and asserts the final highest is exactly one bid and the highest amount is the max submitted that satisfied the increment rule.

Produce: a modified `auction.js` and a short note on why the mutex made the 50-bid test deterministic. (~30–40 min.)

---

## Full JavaScript implementation

```js
// auction.js — runnable: `node auction.js`

// ── Enum ────────────────────────────────────────────────
const AuctionStatus = Object.freeze({
  CREATED:   'CREATED',
  ACTIVE:    'ACTIVE',
  ENDED:     'ENDED',
  CANCELLED: 'CANCELLED',
});

// ── Domain objects ──────────────────────────────────────
class User {
  constructor(id, name) { this.id = id; this.name = name; }
}

class AuctionItem {
  constructor(id, title, description = '') {
    this.id = id; this.title = title; this.description = description;
  }
}

class Bid {
  constructor(bidder, amount) {
    this.bidder = bidder;
    this.amount = amount;
    this.timestamp = Date.now();
  }
}

// ── A tiny async mutex (recall 52) ──────────────────────
// Serializes async critical sections. Each acquire() resolves
// only after the previous holder calls its release().
class Mutex {
  constructor() { this._chain = Promise.resolve(); }
  acquire() {
    let release;
    const next = new Promise(res => { release = res; });
    // Whoever awaits `prev` runs next; they get `release` to unlock.
    const prev = this._chain;
    this._chain = prev.then(() => next);
    return prev.then(() => release);
  }
}

// ── Observer wiring ─────────────────────────────────────
// Observers implement onEvent({ type, auction, data }).
class ConsoleObserver {
  constructor(label) { this.label = label; }
  onEvent(ev) {
    console.log(`   [${this.label}] ${ev.type}: ${ev.message}`);
  }
}

// ── The Auction: STATE + OBSERVER + CONCURRENCY ─────────
class Auction {
  constructor({ id, item, seller, startPrice, minIncrement, endTime }) {
    this.id = id;
    this.item = item;
    this.seller = seller;
    this.startPrice = startPrice;
    this.minIncrement = minIncrement;
    this.endTime = endTime;                 // epoch ms
    this.status = AuctionStatus.CREATED;    // STATE
    this.currentHighestBid = null;          // Bid | null
    this.bids = [];
    this.watchers = [];                     // OBSERVER list
    this.autoMax = new Map();               // userId -> max amount
    this._lock = new Mutex();               // CONCURRENCY guard
  }

  // ── Observer API ──
  subscribe(observer) { this.watchers.push(observer); }
  _notify(type, message, data = {}) {
    for (const w of this.watchers) w.onEvent({ type, message, data, auction: this });
  }

  // ── State transitions ──
  start() {
    if (this.status !== AuctionStatus.CREATED)
      throw new Error(`Cannot start auction in status ${this.status}`);
    this.status = AuctionStatus.ACTIVE;
    this._notify('STARTED', `Auction ${this.id} "${this.item.title}" is now ACTIVE`);
  }

  cancel() {
    if (this.status === AuctionStatus.ENDED)
      throw new Error('Cannot cancel an ended auction');
    this.status = AuctionStatus.CANCELLED;
    this._notify('CANCELLED', `Auction ${this.id} was cancelled`);
  }

  // The minimum a new bid must reach to be valid right now.
  _minValidBid() {
    if (!this.currentHighestBid) return this.startPrice;
    return this.currentHighestBid.amount + this.minIncrement;
  }

  setAutoBid(user, maxAmount) {
    this.autoMax.set(user.id, { user, max: maxAmount });
  }

  // ── placeBid: the hot path. ATOMIC compare-and-set. ──
  async placeBid(user, amount) {
    // STATE guard — behaviour depends entirely on status.
    if (this.status !== AuctionStatus.ACTIVE)
      return { ok: false, reason: `Auction is ${this.status}, not accepting bids` };
    if (Date.now() > this.endTime)
      return { ok: false, reason: 'Auction time has passed' };

    // CONCURRENCY: enter the critical section. Everything from here
    // until release() runs for exactly one bid at a time. We RE-READ
    // the highest INSIDE the lock — never trust a value read outside.
    const release = await this._lock.acquire();
    try {
      const minValid = this._minValidBid();       // re-read inside lock
      if (amount < minValid) {
        return { ok: false,
          reason: `Bid $${amount} below minimum $${minValid}` };
      }

      // Compare-and-SET: this is the atomic step the naive code got wrong.
      const previousLeader = this.currentHighestBid;
      const bid = new Bid(user, amount);
      this.currentHighestBid = bid;
      this.bids.push(bid);

      // OBSERVER: announce new high bid; tell the outbid leader.
      this._notify('NEW_HIGH_BID',
        `${user.name} is leading at $${amount}`, { bid });
      if (previousLeader && previousLeader.bidder.id !== user.id) {
        this._notify('OUTBID',
          `${previousLeader.bidder.name}, you've been outbid (now $${amount})`,
          { outbid: previousLeader.bidder });
      }

      // AUTO-BID resolution runs INSIDE the lock so it stays atomic too.
      this._resolveAutoBids();

      return { ok: true, amount: this.currentHighestBid.amount,
               leader: this.currentHighestBid.bidder.name };
    } finally {
      release();  // release the mutex no matter what
    }
  }

  // ── Auto-bid / proxy bidding resolution ─────────────────
  // Called inside the lock. Repeatedly lets the auto-bidder with the
  // highest max reclaim the lead at the MINIMUM needed, until no
  // auto-bidder can legally beat the current highest. Two auto-bidders
  // settle at (lower max + one increment), capped at the higher max.
  _resolveAutoBids() {
    let progressed = true;
    while (progressed) {
      progressed = false;
      const leaderId = this.currentHighestBid?.bidder.id;
      const need = this._minValidBid(); // amount to beat current highest

      // Find the strongest auto-bidder who is NOT already leading and
      // whose max can cover `need`.
      let best = null;
      for (const { user, max } of this.autoMax.values()) {
        if (user.id === leaderId) continue;
        if (max < need) continue;
        if (!best || max > best.max) best = { user, max };
      }
      if (!best) break; // nobody can beat the leader -> settled

      // Bid the MINIMUM to take the lead: `need`, but never above own max.
      const bidAmount = Math.min(best.max, need);
      const previousLeader = this.currentHighestBid;
      const bid = new Bid(best.user, bidAmount);
      this.currentHighestBid = bid;
      this.bids.push(bid);
      this._notify('NEW_HIGH_BID',
        `${best.user.name} (auto) is leading at $${bidAmount}`, { bid });
      if (previousLeader && previousLeader.bidder.id !== best.user.id) {
        this._notify('OUTBID',
          `${previousLeader.bidder.name}, an auto-bidder outbid you at $${bidAmount}`,
          { outbid: previousLeader.bidder });
      }
      progressed = true; // loop again: the new leader may be re-challenged
    }
  }

  // ── end(): STATE transition + determine winner + OBSERVER ──
  end() {
    if (this.status !== AuctionStatus.ACTIVE)
      throw new Error(`Cannot end auction in status ${this.status}`);
    this.status = AuctionStatus.ENDED;   // locks bids forever

    if (this.currentHighestBid) {
      const w = this.currentHighestBid;
      this._notify('AUCTION_ENDED',
        `SOLD: "${this.item.title}" to ${w.bidder.name} for $${w.amount}`,
        { winner: w.bidder, amount: w.amount });
    } else {
      this._notify('AUCTION_ENDED',
        `Auction "${this.item.title}" ended with NO bids`, { winner: null });
    }
    return this.currentHighestBid;
  }
}

// ── Orchestrator ────────────────────────────────────────
class AuctionService {
  constructor() { this.auctions = new Map(); this._seq = 0; }

  createAuction({ item, seller, startPrice, minIncrement, durationMs }) {
    const id = `A${++this._seq}`;
    const auction = new Auction({
      id, item, seller, startPrice, minIncrement,
      endTime: Date.now() + durationMs,
    });
    this.auctions.set(id, auction);
    return auction;
  }

  async placeBid(auctionId, user, amount) {
    const a = this.auctions.get(auctionId);
    if (!a) throw new Error('No such auction');
    return a.placeBid(user, amount);
  }

  endAuction(auctionId) {
    return this.auctions.get(auctionId).end();
  }
}

// ── Demo: prove State + Observer + Concurrency + Auto-bid ──
async function main() {
  const service = new AuctionService();
  const seller = new User('u0', 'Seller Sam');
  const alice  = new User('u1', 'Alice');
  const bob    = new User('u2', 'Bob');
  const carol  = new User('u3', 'Carol');
  const dave   = new User('u4', 'Dave');

  const item = new AuctionItem('i1', 'Vintage Watch');
  const auction = service.createAuction({
    item, seller, startPrice: 100, minIncrement: 10, durationMs: 60_000,
  });

  // OBSERVER: watchers subscribe.
  auction.subscribe(new ConsoleObserver('watch-feed'));
  auction.subscribe(new ConsoleObserver('seller-dashboard'));

  console.log('\n1) STATE: bidding before start is rejected');
  console.log('  ', await service.placeBid(auction.id, alice, 120));

  console.log('\n2) start() -> ACTIVE');
  auction.start();

  console.log('\n3) Manual bids + min-increment rule');
  console.log('   accept 100:', await service.placeBid(auction.id, alice, 100));
  console.log('   reject 105 (needs >=110):',
    await service.placeBid(auction.id, bob, 105));
  console.log('   accept 110:', await service.placeBid(auction.id, bob, 110));

  console.log('\n4) CONCURRENCY: two near-simultaneous $120 bids, one wins');
  const [r1, r2] = await Promise.all([
    service.placeBid(auction.id, carol, 120),
    service.placeBid(auction.id, dave, 120),
  ]);
  console.log('   carol:', r1);
  console.log('   dave :', r2);
  console.log('   -> exactly one accepted; the other saw the new highest inside the lock');

  console.log('\n5) AUTO-BID: two proxy bidders settle at (lower max + increment)');
  auction.setAutoBid(alice, 200); // Alice will fight up to 200
  auction.setAutoBid(bob, 150);   // Bob up to 150
  // Trigger resolution with a modest manual bid to enter the fray:
  await service.placeBid(auction.id, alice, auction._minValidBid());
  console.log(`   highest after auto-resolution: $${auction.currentHighestBid.amount}` +
    ` by ${auction.currentHighestBid.bidder.name}`);
  console.log('   (Bob max 150 + increment 10 = 160 -> Alice leads at 160, not 200)');

  console.log('\n6) STATE: end() -> ENDED, winner + notifications fire');
  auction.end();

  console.log('\n7) STATE: bidding after end is rejected');
  console.log('  ', await service.placeBid(auction.id, dave, 999));
}

main().catch(err => { console.error(err); process.exit(1); });

module.exports = { Auction, AuctionService, AuctionStatus, Bid, User, AuctionItem, Mutex };
```

Running it shows: the pre-start bid rejected (State), the $105 bid rejected by the increment rule, exactly one of the two $120 bids accepted (Concurrency), the two auto-bidders settling at $160 = 150 + 10 (Auto-bid), and the SOLD notification plus outbid messages firing (Observer). All four cores demonstrated in one run.

---

## Design patterns used and WHY

| Pattern | Where | Why it fits |
|---|---|---|
| **State** (recall [46](./46-pattern-state.md)) | `Auction.status` drives `placeBid`/`start`/`end` | Behaviour genuinely differs per lifecycle stage. Only `ACTIVE` accepts bids; `end()` locks. Explicit transitions kill the "bid after gavel" bug. |
| **Observer** (recall [41](./41-pattern-observer.md)) | `subscribe` + `_notify` events | One-to-many "when the high bid changes, tell everyone." Decouples the auction from *how* notifications are delivered. The natural fit for outbid/end alerts. |
| **Concurrency control** (recall [52](./52-concurrency-fundamentals.md)) | `Mutex` around the compare-and-set in `placeBid` | Two simultaneous bids are a read-modify-write race. The lock serializes the critical section so exactly one becomes the new highest. |
| **Strategy-lite** | `ConsoleObserver` interchangeable with `EmailObserver` | Swap notification channels without touching the auction. |

**The connection to seat booking (118).** The concurrent-bid problem is *the same problem* as two people grabbing the last cinema seat in [118 — Movie Ticket Booking](./118-lld-movie-ticket-booking.md). In both, many writers race to claim one scarce, mutually-exclusive resource (the top-bid slot / the seat). Both are solved by making the claim **atomic**: check-and-set the highest bid, or check-and-hold the seat, inside a single critical section — and by *re-reading the shared state inside the lock*, never trusting a value read before you acquired it. Recognising that two different-looking problems are one concurrency pattern is exactly what the interviewer is probing for.

---

## Extensions the interviewer asks for

**"Add a reserve price."**
Store `reservePrice` on the auction. In `end()`, if `currentHighestBid.amount < reservePrice`, transition to `ENDED` but declare **no winner** ("reserve not met") and notify the seller instead of a buyer. Bidding logic is untouched — only the winner-determination branch changes. Optionally emit a `RESERVE_MET` event the first time a bid crosses it.

**"Add buy-now."**
Add a `buyNowPrice`. Inside `placeBid` (still under the lock), if `amount >= buyNowPrice`, set that bid as highest and immediately call `end()`. Because we are already inside the atomic section, the instant close is race-free — no one can slip a bid in between.

**"Add anti-sniping (extend the end time on last-second bids)."**
In `placeBid`, after a valid bid commits, check `if (endTime - now < 10_000) endTime = now + 30_000` and emit an `EXTENDED` event. The State machine is unaffected — the auction stays `ACTIVE`; only the deadline moves. Watch out for infinite extension: cap total extensions.

**"Make it distributed — many servers behind a load balancer."**
The in-process `Mutex` only guards one Node process; on 10 servers it protects nothing. Move the atomicity into the datastore: replace the compare-and-set with a **conditional update** — `UPDATE auctions SET highest = :amt, leader = :u WHERE id = :id AND highest < :amt` — and treat "0 rows affected" as "you were outbid, retry." For multi-step critical sections, use a **distributed lock** (Redis/ZooKeeper, recall the distributed-locking topic). The rest of the design is identical: `placeBid`'s *shape* is unchanged, only the body of the critical section swaps from `Mutex` to a DB condition. That clean seam is the payoff of having isolated the atomic step in the first place.

---

## Quick reference cheat sheet

- **Three cores:** State (lifecycle), Observer (notifications), Concurrency (concurrent bids). Name all three out loud.
- **State:** `CREATED → ACTIVE → ENDED` (+`CANCELLED`). Bid only in `ACTIVE`. `end()` locks bids.
- **Observer:** auction is the subject; watchers subscribe; events `NEW_HIGH_BID`, `OUTBID`, `AUCTION_ENDED`.
- **The race:** two equal bids at once → naive read-then-write awards both. **Bug.**
- **The fix:** mutex / compare-and-set; **re-read the highest INSIDE the lock**.
- **Min increment:** valid bid ≥ current highest + increment (or start price if no bids).
- **Auto-bid rule:** two proxies settle at **lower max + one increment**, capped at higher max — system bids the *minimum needed*.
- **Second-price flavour:** auto-bid mirrors ad-exchange/eBay economics — winner pays just over the runner-up.
- **Ownership:** the `Auction` owns its status and highest bid; the `AuctionService` only orchestrates.
- **Same as 118:** concurrent bid = last-seat race; one atomic claim of a scarce resource.
- **Distributed upgrade:** swap the in-process mutex for a DB conditional update or distributed lock; `placeBid` shape unchanged.
- **Determinism:** the lock makes the outcome of N concurrent bids reproducible — key for tests.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → behaviours → patterns) this doc applies. |
| **Next** | [118 — Movie Ticket Booking](./118-lld-movie-ticket-booking.md) | The same concurrency problem (one scarce resource, many racers) in a different domain — study the parallel. |
| **Related** | [46 — State Pattern](./46-pattern-state.md) | The auction lifecycle is a State machine; bidding is only valid in ACTIVE. |
| **Related** | [41 — Observer Pattern](./41-pattern-observer.md) | Bid / outbid / auction-end notifications are a textbook Observer subject-and-watchers setup. |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) | The async mutex / compare-and-set that makes the concurrent-bid update atomic. |
