# 76 — Eventual Consistency in Practice
## Category: HLD Components

---

## What is this?

**Eventual consistency** is a promise that a distributed system makes to you: *"If you stop writing, then after some amount of time, every copy of your data will agree on the same value."* That's it. That's the whole promise.

Think of a rumour spreading through an office. You tell one person that the meeting moved to 3pm. Within ten minutes, everyone knows. But for those ten minutes, if you walked around asking people "when is the meeting?", you'd get **different answers from different people** — and nobody is lying. The information is simply still in flight.

Recall from [08 — Consistency Models](./08-consistency-models.md) that consistency isn't a yes/no switch — it's a **spectrum**, from linearizable (every read sees the latest write, always) down to eventual (reads may see stale data). And recall from [09 — CAP Theorem](./09-cap-theorem.md) that when the network partitions, you must pick availability or consistency. Eventual consistency is what "picking availability" actually *feels like* once you have to build it, ship it, and get paged about it at 2am.

---

## Why does it matter?

Because **almost every system you will ever work on at scale is eventually consistent somewhere**, and most engineers don't realise it until a bug report lands.

You put Redis in front of your database — congratulations, your cache is eventually consistent with your DB. You add a read replica to take load off the primary — your replica is eventually consistent. You push data to a CDN, a search index, a data warehouse, a mobile client's local store. Every one of those is a replica that lags.

**What breaks if you don't understand this:**
- You'll write a `POST /profile` followed by a `GET /profile` in your own frontend and the user will see their **old** data, and you will spend a day blaming your framework.
- You'll pick "Last Write Wins" because it's the default, and quietly lose customer data for months.
- You'll build an inventory system on top of an eventually consistent store and sell 300 tickets to a 200-seat theatre.

**The interview angle:** the moment you say "I'll use Cassandra" or "I'll add read replicas," a good interviewer will ask *"so what happens when a user writes and immediately reads?"* Being able to name the anomaly (read-your-own-writes), name the fix (sticky routing to the leader for N seconds), and name the cost (leader load) is the difference between a hire and a no-hire.

**The real-work angle:** this is where the nastiest, least reproducible production bugs live. "It only happens sometimes" is almost always a replication lag story.

---

## The core idea — explained simply

### The Whiteboard-in-Every-Office Analogy

Your company has offices in **New York, London, and Tokyo**. Each office has a whiteboard with the current stock price of a fictional company written on it. Every whiteboard is supposed to say the same number.

When the price changes, whoever hears it first writes it on **their local whiteboard immediately** — they don't wait for permission — and then sends a courier to the other two offices to update theirs.

Now walk through what actually happens:

- **9:00:00** — NY hears the price is `$42`. NY writes `42`. Couriers depart for London and Tokyo.
- **9:00:01** — Someone in Tokyo looks at the Tokyo whiteboard. It still says `$41`. **They got a stale read.** Nobody did anything wrong.
- **9:00:03** — The courier arrives in Tokyo. Tokyo now says `$42`. The system has **converged**.

That's eventual consistency: **write locally, replicate in the background, converge later.** The system never says "no" to a write. Every office stays open even if the trans-Pacific cable is cut — they just fall further behind.

Now the hard part. What if **two offices write at the same time?**

- **9:00:00** — NY writes `$42`. London writes `$43`. Neither knows about the other.
- **9:00:03** — Couriers cross in mid-air. NY receives "London says 43." London receives "NY says 42."
- **Now what?** Both offices have a value and a conflicting update. Somebody has to decide.

This is a **conflict**, and how you resolve it is the single most important design decision in an eventually consistent system. Everything else in this doc is about that question.

| Analogy | Technical concept |
|---|---|
| An office | A replica / node |
| The whiteboard | The stored value on that replica |
| Writing locally without asking | Accepting a write without coordination (high availability) |
| The courier | Asynchronous replication (gossip, log shipping, anti-entropy) |
| Courier travel time | **Replication lag** — usually ms, can be minutes |
| Reading a stale whiteboard | A **stale read** |
| Trans-Pacific cable cut | A **network partition** |
| Two offices writing at once | A **concurrent write** → a genuine conflict |
| Deciding whose number survives | **Conflict resolution** (LWW / vector clocks / CRDTs / ask the app) |

### What the promise conspicuously does NOT say

Read the definition again, slowly:

> *If no new updates are made to an object, eventually all reads of that object will return the same value.*

Notice the two enormous holes:

1. **It says nothing about WHEN.** "Eventually" has no number attached. In practice, replication lag inside one datacentre is **1–10 ms**. Cross-region is **50–200 ms**. During a partition, a GC pause, or a replica catching up after being down, it can be **minutes**. There is no SLA hiding in the word "eventually."
2. **It says nothing about what you read in the meantime.** Before convergence you might read the new value, the old value, or — with multiple replicas — the new value and *then the old one again*. Time can appear to run backwards.

That second hole is why we have to talk about anomalies.

---

## Key concepts inside this topic

### 1. What it actually feels like to a user

Abstractions are easy to nod along to. Here's eventual consistency as a **support ticket**.

**"I changed my profile photo and it's still the old one."**
Your upload hit the primary. The page reload hit a read replica that hadn't caught up. You see your old face. You refresh again, hit a different replica, and now it's correct. You refresh a third time, land back on the lagging replica, and the old photo is *back*. You file a bug that says "photo randomly reverts."

**"I deleted an email and it came back."**
Your delete landed on replica A. Meanwhile replica B, which never heard about the delete, was down for two minutes. When B rejoins, it gossips its state: "hey, I have this email." A naive merge treats *presence* as data and *absence* as nothing, so the union puts the email back. This is the classic **resurrection** bug, and it's exactly why real systems store a **tombstone** (a "this was deleted at time T" marker) instead of just removing the row.

**"Two people got the same username."**
Both `POST /signup` requests with username `neo` hit different replicas within 50 ms. Both replicas check locally: "is `neo` taken? No." Both accept. Both users get a welcome email. Uniqueness is a **global** invariant, and you cannot enforce a global invariant with only local knowledge. (Fix: usernames must go through a single coordinated path — a strongly consistent store, or a partition where all writes for a given username hash land on one leader.)

**"The like count went 42 → 41 → 43."**
Two replicas each hold a slightly different count. Your load balancer sends consecutive page loads to different replicas. The number *jumps around*. To the user this looks like the site is broken. It's not — it's **non-monotonic reads**, and we fix it below.

### 2. The three anomalies and their targeted fixes

These three have names because they show up constantly. Learn the name, the symptom, and the fix — this trio is interview gold.

#### a) Read-your-own-writes (a.k.a. read-after-write consistency)

**Symptom:** you write, then immediately read, and your own write is missing.

You don't need the system to be globally consistent. You need **one guarantee**: a user always sees *their own* writes. Other users' writes can lag; the user won't notice. Their own writes vanishing is unforgivable.

**Fixes, in increasing order of effort:**

```javascript
// Fix 1: Sticky reads — after a write, route this user to the leader
// for a short window. Cheap, effective, slightly increases leader load.
const WRITE_WINDOW_MS = 10_000; // > p99 replication lag, with headroom

class ReadRouter {
  constructor(redis) {
    this.redis = redis; // shared so ANY app server can see the flag
  }

  async markWrite(userId) {
    // Why Redis and not memory: the next request may hit a different app server.
    await this.redis.set(`recent_write:${userId}`, Date.now(), 'PX', WRITE_WINDOW_MS);
  }

  async pickReplica(userId) {
    const wroteRecently = await this.redis.get(`recent_write:${userId}`);
    return wroteRecently ? this.leader : this.pickReadReplica();
  }
}
```

```javascript
// Fix 2: Version tokens — the write returns a logical timestamp (LSN / commit
// position). The read carries it back and the replica waits until it's caught up.
// More precise than a fixed window: no arbitrary 10s guess.
async function handleWrite(req, res) {
  const { lsn } = await leader.write(req.body);
  res.cookie('read_after_lsn', lsn, { maxAge: 30_000 });
  res.json({ ok: true });
}

async function handleRead(req, res) {
  const requiredLsn = req.cookies.read_after_lsn;
  const replica = pickReadReplica();
  if (requiredLsn && (await replica.replayPosition()) < requiredLsn) {
    return res.json(await leader.read(req.params.id)); // fall back to leader
  }
  res.json(await replica.read(req.params.id));
}
```

```javascript
// Fix 3: Local echo — don't read the server at all for your own write.
// This is what optimistic UI does, and it's the cheapest fix of the three.
async function updateProfile(patch) {
  cache.set('me', { ...cache.get('me'), ...patch }); // trust ourselves
  render(cache.get('me'));                            // instant, correct-looking
  await api.patch('/me', patch);                      // converge in background
}
```

#### b) Monotonic reads (time going backwards)

**Symptom:** you refresh and *lose* information you already saw. The like count goes 43 → 41. A comment you read disappears.

Cause: consecutive reads hit **different replicas with different lag**. Replica A is 50 ms behind; replica B is 3 s behind. You read A, then B.

**Fix: pin each user to one replica**, deterministically, by hashing their ID. One user always reads from the same replica, so they see one consistent timeline. That replica may be stale — but it is stale *monotonically*. Time only moves forward.

```javascript
import { createHash } from 'node:crypto';

// A user always lands on the same replica → their view of time never rewinds.
// Note the failure mode: if that replica dies, the user is rehashed to a new one
// and may briefly go BACKWARDS. Monotonic reads are a best-effort guarantee here.
function pickReplicaFor(userId, replicas) {
  const h = createHash('md5').update(String(userId)).digest();
  return replicas[h.readUInt32BE(0) % replicas.length];
}
```

#### c) Consistent prefix reads (the answer before the question)

**Symptom:** you see an effect before its cause.

```
What actually happened:
  Mr. Poons: "How far into the future can you see, Mrs. Cake?"
  Mrs. Cake: "About ten seconds."

What an observer with a broken prefix sees:
  Mrs. Cake: "About ten seconds."      ← ??? ten seconds of what?
  Mr. Poons: "How far into the future can you see, Mrs. Cake?"
```

Cause: the two writes live on **different partitions** that replicate independently and at different speeds. There is no global ordering across partitions.

**Fix: keep causally-related writes on the same partition.** All messages in one conversation share a partition key (`conversation_id`), so they replicate in order relative to each other. This is the same trick Kafka uses: ordering is guaranteed **within a partition**, never across partitions — so you choose your partition key to match your causality.

### 3. Conflict resolution, part 1 — Last Write Wins (LWW)

Two replicas hold different values for the same key. Pick the one with the **later timestamp**. Discard the other. Done.

This is genuinely appealing: it's O(1), it needs no extra storage, it always converges (as long as everyone uses the same tiebreak rule), and it's what **Cassandra does by default** — every column carries a write timestamp, and the highest one wins.

**THE FLAW: LWW silently discards data, and "later" may be a lie.**

Server clocks drift. NTP typically keeps them within a few milliseconds, but a misconfigured host, a VM pause, or a leap-second bug can put a machine **seconds or minutes** off. Now watch:

```
Server A's clock: 10:00:05  (five seconds FAST — nobody has noticed)
Server B's clock: 10:00:00  (correct)

Real time 10:00:00 → Alice writes profile.bio = "Loves Rust"  → lands on Server A
                     Stored with timestamp 10:00:05

Real time 10:00:02 → Bob (Alice's teammate, editing the same shared doc)
                     writes profile.bio = "Loves Rust and Go" → lands on Server B
                     Stored with timestamp 10:00:02

Replication runs. Conflict on `profile.bio`. LWW compares:
     10:00:05 (Alice, via fast clock A)  >  10:00:02 (Bob, via correct clock B)

WINNER: Alice's older value.
Bob's write — which happened LATER in real time — is silently, permanently GONE.
No error. No log line. No 500. Bob's browser said "Saved ✓".
```

That last line is the whole problem. **LWW does not fail loudly — it fails silently.** Bob has no idea. You have no idea. You'll find out when a customer complains that "the system ate my edit," and you will not be able to reproduce it.

**When LWW is acceptable:**
- The value is a **full immutable snapshot** where losing an intermediate version is fine (a cache entry, a session blob, "user's last seen location").
- Writes to the same key from different clients are **rare** (a single owner mutates it).
- You control the clocks tightly, or better: you use a **logical** timestamp instead of a wall clock.

**When it is not:** anything where two parties can legitimately edit the same thing, and both edits matter.

### 4. Conflict resolution, part 2 — Version vectors (vector clocks)

The core insight: **the timestamp problem is that wall clocks measure time, but what you actually need to know is causality.** Did write B *know about* write A? If yes, B is a descendant of A and can safely overwrite it. If neither knew about the other, they are **concurrent** — a genuine conflict, and no timestamp can resolve it honestly.

A **version vector** captures exactly this. Each replica keeps a counter. The vector is a map `{ replicaName: counter }`. Every time a replica accepts a write, it bumps **its own** counter.

**Comparing two vectors V1 and V2:**

| Condition | Meaning |
|---|---|
| Every entry in V1 ≤ the matching entry in V2, and at least one is strictly less | **V1 happened-before V2.** V2 descends from V1. Safe: keep V2, discard V1. |
| Same, reversed | **V2 happened-before V1.** Keep V1. |
| Vectors are identical | Same version. Nothing to do. |
| V1 has an entry bigger, AND V2 has a different entry bigger | **CONCURRENT.** A real conflict. Neither saw the other. You must merge. |

That last row is the payoff: vector clocks **cannot resolve a conflict for you**, but they can tell you *with certainty* whether one exists. LWW can't — it just guesses, and destroys data when it guesses wrong.

**Worked example.** Two replicas, `A` and `B`. One key: `cart`.

| Step | What happens | Value on A | A's vector | Value on B | B's vector |
|---|---|---|---|---|---|
| 1 | Client writes `[milk]` to A | `[milk]` | `{A:1}` | — | — |
| 2 | A replicates to B | `[milk]` | `{A:1}` | `[milk]` | `{A:1}` |
| 3 | **Partition!** A and B can't talk. | | | | |
| 4 | Client 1 adds eggs on A | `[milk, eggs]` | `{A:2}` | `[milk]` | `{A:1}` |
| 5 | Client 2 adds bread on B | `[milk, eggs]` | `{A:2}` | `[milk, bread]` | `{A:1, B:1}` |
| 6 | **Partition heals.** Compare `{A:2}` vs `{A:1, B:1}`: A's entry is bigger on the left (2 > 1), B's entry is bigger on the right (1 > 0). → **CONCURRENT.** | | | | |
| 7 | Both are kept as **siblings**. Vector of the merged version: `{A:2, B:1}` | | | | |

Note step 6 carefully. If we had used LWW, one of `eggs` or `bread` would be **gone forever**. The vector clock refuses to make that call — and that refusal is the feature.

```javascript
const Ord = Object.freeze({
  BEFORE: 'BEFORE',       // v1 happened-before v2  → v2 wins
  AFTER: 'AFTER',         // v2 happened-before v1  → v1 wins
  EQUAL: 'EQUAL',
  CONCURRENT: 'CONCURRENT' // genuine conflict → must merge
});

class VersionVector {
  constructor(counters = {}) {
    this.counters = { ...counters }; // { replicaId: number }
  }

  /** Called when THIS replica accepts a write. */
  increment(replicaId) {
    const next = { ...this.counters };
    next[replicaId] = (next[replicaId] ?? 0) + 1;
    return new VersionVector(next);
  }

  /** Element-wise max. The version of a merged/reconciled value. */
  merge(other) {
    const keys = new Set([...Object.keys(this.counters), ...Object.keys(other.counters)]);
    const merged = {};
    for (const k of keys) {
      merged[k] = Math.max(this.counters[k] ?? 0, other.counters[k] ?? 0);
    }
    return new VersionVector(merged);
  }

  compare(other) {
    let selfBigger = false;
    let otherBigger = false;
    const keys = new Set([...Object.keys(this.counters), ...Object.keys(other.counters)]);
    for (const k of keys) {
      const a = this.counters[k] ?? 0;
      const b = other.counters[k] ?? 0;
      if (a > b) selfBigger = true;
      if (b > a) otherBigger = true;
    }
    if (selfBigger && otherBigger) return Ord.CONCURRENT; // neither saw the other
    if (selfBigger) return Ord.AFTER;
    if (otherBigger) return Ord.BEFORE;
    return Ord.EQUAL;
  }
}

// --- Replaying the table above ---
const vA = new VersionVector().increment('A');            // {A:1}  step 1
const vA2 = vA.increment('A');                            // {A:2}  step 4
const vB1 = vA.increment('B');                            // {A:1,B:1} step 5

console.log(vA.compare(vA2));    // BEFORE     → safe to overwrite, no conflict
console.log(vA2.compare(vB1));   // CONCURRENT → real conflict, keep both
console.log(vA2.merge(vB1).counters); // { A: 2, B: 1 } → version of the merged cart
```

**What DynamoDB (and its ancestor, Amazon's Dynamo paper) does with this:** when a read finds concurrent siblings, it **returns them all to your application** along with a *context* token, and says, in effect: *"I detected a real conflict. I am not qualified to resolve it — you know your business logic, I don't. Merge them and write the result back with this context."*

For a shopping cart, the merge Amazon chose is **union the items**. Why? Because for a retailer, a customer seeing an item they'd removed is annoying; a customer *losing* an item they wanted to buy costs revenue. So they biased toward keeping things.

That choice has a famous, visible consequence: **a deleted item sometimes comes back into your cart.** In the worked example above, if Client 2's write on B had been "remove milk," the union merge would resurrect the milk. That's not a bug in Dynamo — it's the deliberate business trade-off of an "add-wins" merge, and it's exactly the email-resurrection story from earlier. (The proper fix, again: store a tombstone for the removal so the merge can see that a delete *happened*, rather than seeing only an absence.)

### 5. Conflict resolution, part 3 — CRDTs

**CRDT = Conflict-free Replicated Data Type.** These are data structures designed so that concurrent updates **always merge deterministically, with zero coordination**. There is no conflict to resolve, because the merge function is mathematically guaranteed to produce the same answer no matter what order updates arrive in.

The merge operation must be:
- **Commutative** — `merge(a, b) === merge(b, a)` (order doesn't matter)
- **Associative** — grouping doesn't matter
- **Idempotent** — `merge(a, a) === a` (receiving the same update twice is harmless — crucial, because networks retry)

Get those three properties and convergence is a *theorem*, not a hope.

#### The G-Counter (grow-only counter) — the simplest CRDT that matters

The naive approach fails immediately: if each replica stores a single number `5` and two replicas each increment to `6`, what's the merge? `6`? Then you lost an increment. `11`? Nonsense.

The trick: **don't store a number. Store a vector of per-replica counts.** Each replica only ever increments **its own slot**. The value is the **sum** of all slots. The merge is the **element-wise max** — and max is commutative, associative, and idempotent. Convergence is free.

```javascript
/**
 * G-Counter: a grow-only counter. Perfect for like counts, view counts,
 * page-hit counters — anything that only ever goes up.
 * Each replica owns exactly ONE slot and only ever touches that slot.
 */
class GCounter {
  constructor(replicaId, slots = {}) {
    this.replicaId = replicaId;
    this.slots = { ...slots }; // { replicaId: count }
  }

  increment(by = 1) {
    this.slots[this.replicaId] = (this.slots[this.replicaId] ?? 0) + by;
  }

  value() {
    return Object.values(this.slots).reduce((sum, n) => sum + n, 0);
  }

  /**
   * Element-wise MAX — never sum, or replaying the same gossip message twice
   * would double-count. Max makes merge idempotent, which is why we can retry
   * replication blindly and safely.
   */
  merge(other) {
    for (const [id, count] of Object.entries(other.slots)) {
      this.slots[id] = Math.max(this.slots[id] ?? 0, count);
    }
    return this;
  }
}

// Three replicas take likes during a network partition. Nobody coordinates.
const a = new GCounter('A');
const b = new GCounter('B');
const c = new GCounter('C');

a.increment(); a.increment();  // 2 likes hit replica A
b.increment();                 // 1 like  hit replica B
c.increment(); c.increment(); c.increment(); // 3 likes hit replica C

// Partition heals. Gossip arrives in a RANDOM order, some messages twice.
a.merge(b).merge(c).merge(b);  // duplicate merge of b — harmless (idempotent)
c.merge(a).merge(b);           // totally different order

console.log(a.value()); // 6
console.log(c.value()); // 6  ← same answer, zero coordination, no lost likes
```

Six likes. No locks, no leader, no round-trip, no lost increments — and it would still be 6 if the gossip messages arrived backwards, duplicated, or a week late.

**The G-Set (grow-only set):** same idea, merge = **union**. Elements can be added, never removed. Great for "which tags have ever been applied," "which devices have ever synced."

To support removal you need a **2P-Set** (two sets: added + removed/tombstones; an element is present if it's in `added` and not in `removed`) or an **OR-Set** (each add gets a unique tag, so a later add can beat an earlier remove). Notice you're now storing tombstones — the same answer we kept arriving at.

**Where CRDTs actually run:**
- **Redis Enterprise CRDTs (Active-Active)** replicate counters, sets, and hashes across geo-distributed clusters, merging without a leader.
- **Collaborative editors.** Figma's multiplayer engine, **Automerge**, and **Yjs** use sequence CRDTs so two people typing in the same paragraph at the same time both keep their characters — no lock, no "someone else is editing this," no lost keystrokes.
- **Riak** ships CRDT data types (counters, sets, maps) as first-class citizens.

**The limitation — and it's a real one:** *not every operation can be expressed as a CRDT.* Any operation with a **global invariant** is out. "The balance must never go below zero" cannot be a CRDT, because each replica would have to know the global balance to enforce it — which requires exactly the coordination CRDTs exist to avoid. CRDTs also **grow**: tombstones and per-replica metadata accumulate, and garbage-collecting them safely is genuinely hard.

### 6. Conflict resolution, part 4 — let the application (or the human) decide

Sometimes the honest answer is: **a machine cannot know which version is right.** So surface both and ask.

You already use this every day. It's **git**:

```
<<<<<<< HEAD
const TIMEOUT_MS = 5000;
=======
const TIMEOUT_MS = 30000;
>>>>>>> feature/slow-uploads
```

Git detected concurrent edits to the same lines. It refuses to guess. It hands you both siblings and makes you merge. Google Docs' version history, iCloud's "Keep Both" dialog, and Dynamo's sibling-returning read are all the same pattern.

**Rule:** the *closer* the merge logic sits to the business meaning of the data, the better the merge. A database knows bytes. Your app knows that a shopping cart should union and a "last known GPS position" should take the newest. Push resolution up to whoever knows the most.

### 7. When eventual consistency is fine — and when it absolutely is not

| Fine (use eventual consistency) | Why it's safe |
|---|---|
| Like / view / follower counts | Off-by-three for 200 ms harms nobody. Nobody is auditing it. |
| Social feeds, timelines | A post arriving 2 s late is invisible to the user. |
| **DNS** | The canonical eventually consistent system. TTLs are *literally* a "how stale may this be" contract. |
| Search indexes (Elasticsearch) | A new document being searchable 1 s later is fine. |
| Analytics, dashboards, metrics | Aggregates over huge windows — a few stragglers don't move the number. |
| CDN / cache content | Staleness is bounded by TTL and self-heals on the next fetch. |

| NOT fine (needs strong consistency / coordination) | Why it breaks |
|---|---|
| **Account balances** | Two concurrent withdrawals on different replicas → overdraft. Money is a conserved quantity; you cannot "eventually" un-spend it. |
| **Inventory decrement at zero stock** | Two replicas each see "1 left" and each sell it. You've oversold, and now a human must apologise and refund. |
| **Seat / ticket booking** | Seat 14C cannot belong to two people. Double-booking is a real-world, physical conflict — no merge function exists. |
| **Unique username / email assignment** | A global uniqueness invariant. Local knowledge can never enforce it. |
| Permissions & access revocation | "Eventually revoked" means "briefly still authorised." That's a security incident. |

**The rule of thumb — two questions:**

1. **Can two users briefly see different truths without anything bad happening?**
2. **If the system returns a wrong answer for a few hundred milliseconds, is the cost low and does it self-correct?**

**Two yeses → eventual consistency, and enjoy the availability and the latency.**
**Any no → you need coordination on that specific operation.**

Note that last phrase. It's *per-operation*, not per-system. The same e-commerce app can serve product pages, recommendations, and review counts from eventually consistent replicas (99% of traffic, blazingly fast) while routing the single `decrement_stock` operation through a strongly consistent transaction. **Don't buy strong consistency for the whole system when only one endpoint needs it.**

### 8. Hiding it from the user — the business tactic

Users don't care about your consistency model. They care whether the app **feels** correct. Two techniques do most of the work.

**a) Optimistic UI.** When a user taps "like," you already know what the answer will be. So show it. Increment the heart *instantly*, locally, and fire the request in the background. The user's perceived latency is **zero**. The server converges 40 ms later and nobody ever knows there was a gap. If the request fails, you roll the UI back and show a toast.

```javascript
async function toggleLike(postId, ui, api) {
  const before = ui.getLikeState(postId);
  const optimistic = { liked: !before.liked, count: before.count + (before.liked ? -1 : 1) };

  ui.setLikeState(postId, optimistic); // instant — the user sees THEIR truth

  try {
    // Idempotency key: if the network retries, the server counts this ONCE.
    const server = await api.post(`/posts/${postId}/like`, {
      liked: optimistic.liked,
      idempotencyKey: `${ui.userId}:${postId}:${optimistic.liked}`
    });
    ui.setLikeState(postId, server); // reconcile with the real count
  } catch (err) {
    ui.setLikeState(postId, before); // roll back — this is the part people forget
    ui.toast('Could not save your like. Tap to retry.');
  }
}
```

**b) Idempotent, self-healing background reconciliation.** Don't just hope the system converges — **make it converge on purpose**. Run a background job that recomputes the truth from the source of record and repairs the replicas.

```javascript
/**
 * Anti-entropy: periodically recompute the authoritative count from the
 * source-of-truth event log and repair the fast (approximate) counter.
 * Must be idempotent — it will run thousands of times on the same rows,
 * and it must be safe for two workers to run it concurrently.
 */
async function reconcileLikeCounts({ eventStore, cache, batchSize = 500 }) {
  for await (const postIds of eventStore.staleCounterBatches(batchSize)) {
    const trueCounts = await eventStore.countLikesFor(postIds); // Map<postId, n>
    for (const [postId, trueCount] of trueCounts) {
      const cached = await cache.get(`likes:${postId}`);
      if (Number(cached) !== trueCount) {
        // SET, not INCR: setting to an absolute value is idempotent.
        // INCR is not — a retried INCR double-counts.
        await cache.set(`likes:${postId}`, trueCount);
      }
    }
  }
}
```

This is the pattern behind Dynamo's **read repair** and **anti-entropy** processes, and behind every "recompute the aggregate nightly" cron job you've ever written. Convergence isn't magic — somebody has to actually do it.

---

## Visual / Diagram description

### Diagram 1: The write path and the lag window

```
                         WRITE                          READ (t + 5ms)
                           │                                 │
                           ▼                                 ▼
                    ┌─────────────┐                   ┌─────────────┐
                    │   Client    │                   │   Client    │
                    └──────┬──────┘                   └──────┬──────┘
                           │ POST bio="Loves Rust"           │ GET bio
                           ▼                                 │
                  ┌──────────────────┐                       │
                  │  LEADER (US-East)│                       │
                  │  bio="Loves Rust"│ ◀── ACK sent to       │
                  │  ✅ COMMITTED    │     client IMMEDIATELY│
                  └────────┬─────────┘     (this is why it's │
                           │                available & fast)│
          ┌────────────────┼────────────────┐                │
          │ async          │ async          │ async          │
          │ ~2ms           │ ~80ms          │ ~180ms         │
          ▼                ▼                ▼                │
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
  │ REPLICA 1    │ │ REPLICA 2    │ │ REPLICA 3    │◀───────┘
  │ (US-East)    │ │ (EU-West)    │ │ (AP-South)   │  reads land here
  │ "Loves Rust" │ │ (in flight)  │ │ (in flight)  │
  │ ✅ converged │ │ ⏳ STALE     │ │ ⏳ STALE     │  ← returns the OLD bio
  └──────────────┘ └──────────────┘ └──────────────┘     STALE READ

  ◀────────────── THE INCONSISTENCY WINDOW ──────────────▶
     Usually 1-200ms. During a partition: minutes.
     Eventual consistency guarantees this window CLOSES.
     It guarantees NOTHING about how wide it is.
```

The leader acknowledges the write **before** the replicas have it — that's the entire source of both the speed and the anomaly. Any read that lands on a replica inside the window returns the old value. The width of that window is your **replication lag**, and it is the single most important metric to graph and alert on in an eventually consistent system.

### Diagram 2: A concurrent write, and the three ways to resolve it

```
                    ┌────────────── NETWORK PARTITION ──────────────┐
                    │                                               │
        ┌───────────▼──────────┐                     ┌──────────────▼────────┐
        │      REPLICA A       │      ✂ no link ✂    │      REPLICA B        │
        │  cart = [milk, eggs] │  ◀───────────────▶  │  cart = [milk, bread] │
        │  vector = {A:2}      │                     │  vector = {A:1, B:1}  │
        └───────────┬──────────┘                     └──────────────┬────────┘
                    │                                               │
                    └──────────────► PARTITION HEALS ◀──────────────┘
                                          │
                                          ▼
                            ┌──────────────────────────┐
                            │  Compare {A:2} ↔ {A:1,B:1}│
                            │  A bigger on left,        │
                            │  B bigger on right        │
                            │  ⇒ CONCURRENT — REAL      │
                            │     CONFLICT              │
                            └────────────┬──────────────┘
                                         │
         ┌───────────────────────────────┼───────────────────────────────┐
         ▼                               ▼                               ▼
┌──────────────────┐          ┌────────────────────┐        ┌────────────────────┐
│  1. LWW          │          │  2. SIBLINGS       │        │  3. CRDT           │
│                  │          │     (DynamoDB)     │        │                    │
│ Later timestamp  │          │ Return BOTH to the │        │ Merge is DEFINED   │
│ wins.            │          │ app. App merges.   │        │ by the data type.  │
│                  │          │                    │        │ Union for a set.   │
│ Result:          │          │ Result:            │        │ Result:            │
│ [milk, eggs]     │          │ [milk,eggs,bread]  │        │ [milk,eggs,bread]  │
│                  │          │                    │        │                    │
│ ❌ "bread" is    │          │ ⚠️  You must write │        │ ✅ Automatic,      │
│    GONE. Silent  │          │    the merge — and │        │    deterministic,  │
│    data loss.    │          │    a removed item  │        │    no coordination.│
│ ✅ Zero effort.  │          │    can resurrect.  │        │ ❌ Not every op    │
│                  │          │ ✅ No silent loss. │        │    fits.           │
└──────────────────┘          └────────────────────┘        └────────────────────┘
```

Redraw this from memory. The top half (partition → divergence → concurrent detection) and the three-way fork at the bottom is the entire conflict-resolution chapter of any distributed systems interview, on one whiteboard.

---

## Real world examples

### 1. Amazon DynamoDB — siblings, and the cart that won't forget

The original Dynamo paper (2007) is the source of most of this doc's vocabulary. Amazon's requirement was brutal and specific: **the "Add to Cart" button must never fail**, even during a datacentre partition, because a failed add-to-cart is directly lost revenue.

So Dynamo accepts every write, everywhere, always. It tracks causality with version vectors. When a read finds concurrent siblings, it returns **all of them** to the application with a context token and lets the app merge — because Dynamo knows bytes, and only the cart service knows that carts should **union**.

The visible consequence is exactly the trade-off they chose: an item you removed during a partition can reappear. Amazon decided that a slightly annoying resurrection beats a silently lost purchase. That is a **business decision expressed as a merge function** — and it's the best example in the field of why conflict resolution belongs near the domain logic. (Today's DynamoDB service defaults to eventually consistent reads for speed, and offers an opt-in `ConsistentRead` flag per request — you pay more and get higher latency for it. Consistency is priced per-call.)

### 2. Cassandra — LWW by default, and the clock you'd better be watching

Cassandra stores a timestamp with every **column**, and resolves conflicts with Last Write Wins at column granularity. It's simple, fast, and needs no read-time reconciliation.

It also means Cassandra's correctness depends on your **clocks**. The Cassandra docs are explicit that you must run NTP; clock skew across nodes leads directly to the silent-data-loss scenario in section 3 above. Cassandra also lets you tune consistency **per query** via `ONE / QUORUM / ALL` — a `QUORUM` write plus a `QUORUM` read (where `R + W > N`) gives you read-your-writes without abandoning the AP architecture. And it uses **read repair** and **anti-entropy** (Merkle-tree comparison) to actively push replicas back into agreement rather than waiting for luck.

### 3. Figma — CRDTs so two designers can drag the same rectangle

Figma's multiplayer engine keeps a copy of the document on every client and merges concurrent edits with CRDT-style, conflict-free data structures. There is no lock, no "checkout," no "Bob is editing this file." Two designers can move the same object at the same time and the document converges to one state on everybody's screen.

The design choice worth stealing: Figma **doesn't try to merge everything the same way.** Different properties get different merge semantics — a *position* is a last-writer-wins scalar (the last drag wins; that's what a user expects), while the *set of children of a frame* is a mergeable collection (both people's new layers survive). **The merge strategy is chosen per-field, based on what a human would find least surprising.** That is the real lesson of eventual consistency: convergence is a math problem, but choosing *what to converge to* is a product problem.

---

## Trade-offs

| Dimension | Eventual consistency | Strong consistency |
|---|---|---|
| **Write latency** | Low — ack after one local write (~1–5 ms) | High — must reach a quorum, often cross-region (~50–500 ms) |
| **Read latency** | Low — read the nearest replica | Higher — read the leader or a quorum |
| **Availability during a partition** | Stays up, both sides accept writes | The minority side rejects writes (or blocks) |
| **What can go wrong** | Stale reads, conflicts, anomalies | Timeouts, unavailability, cascading slowness |
| **Developer burden** | **High** — you handle conflicts and anomalies | Low — the DB handles it |
| **Where the complexity lives** | In your application code | In the database |
| **Scales geographically** | Beautifully | Painfully — the speed of light is the ceiling |

| Conflict resolution strategy | Pros | Cons | Use when |
|---|---|---|---|
| **Last Write Wins** | Trivial, O(1), no extra storage, always converges | **Silently destroys data**; depends on clock accuracy | The value is an immutable snapshot with a single natural owner (cache, session, last-known-location) |
| **Version vectors + siblings** | Detects real conflicts with certainty; never loses data silently | You must write the merge; siblings can accumulate; more storage | Data is mergeable and the domain knows how (carts, tag sets, documents) |
| **CRDTs** | Automatic, deterministic, zero coordination, provably converges | Metadata/tombstones grow; not every operation is expressible | Counters, sets, collaborative text, geo-distributed active-active |
| **Ask the user** | Never wrong — a human decides | Terrible UX if it happens often | Conflicts are rare and genuinely ambiguous (git, iCloud) |

**Rule of thumb:** Default to eventual consistency for the **read-heavy 99%** of your traffic — it buys you availability, low latency, and geo-scale almost for free. Then find the small number of operations that guard a **global invariant** (money, stock at zero, seats, unique names) and pay for coordination on **those alone**. Consistency is not a system-wide setting. It's a per-operation budget, and you should spend it where a wrong answer costs real money.

---

## Common interview questions on this topic

### Q1: "What does 'eventually consistent' actually guarantee?"
**Hint:** Exactly one thing: *if writes stop, all replicas converge to the same value.* It guarantees **nothing** about when (no bound — ms normally, minutes during a partition) and **nothing** about what you read in the meantime (you may see stale values, or even go backwards). Say that out loud — most candidates state the promise but never state its limits, and stating the limits is what shows you've operated one.

### Q2: "A user updates their profile and immediately sees the old version. Fix it."
**Hint:** Name the anomaly — **read-your-own-writes**. Then give three fixes in increasing cost: (1) route that user's reads to the leader for ~10 s after their write, with the flag in a shared store like Redis so any app server sees it; (2) return an LSN/version token from the write and have reads wait for the replica to catch up to it, falling back to the leader if not; (3) optimistic UI — render from a local echo of what they just wrote. Then name the cost: fix (1) puts read load back on the leader.

### Q3: "Your system uses Last Write Wins. What's the danger?"
**Hint:** Silent data loss. Two concurrent writes → the "loser" is discarded with no error, no log, no notification. Worse, "later" is decided by a **wall clock**, and clock skew means the write that happened later in real time can lose. Give the concrete scenario (a 5-second-fast server). Then offer the alternatives: version vectors to *detect* the conflict, and a domain-aware merge or a CRDT to *resolve* it.

### Q4: "Explain vector clocks. Why not just use timestamps?"
**Hint:** Timestamps measure *time*; vector clocks measure *causality* — did write B know about write A? Each replica bumps its own counter on write. If every entry of V1 ≤ V2, then V1 happened-before V2 and it's safe to overwrite. If each vector has an entry the other doesn't dominate, they're **concurrent** — a genuine conflict a timestamp would have papered over. Sketch the `{A:2}` vs `{A:1,B:1}` example. Punchline: vector clocks don't resolve conflicts, they *detect* them honestly — and honest detection is the thing LWW can't give you.

### Q5: "What's a CRDT and where would you use one?"
**Hint:** A data type whose merge is commutative, associative, and idempotent — so concurrent updates converge automatically with **no coordination**. Give the G-Counter: each replica increments only its own slot; the value is the sum; the merge is element-wise **max** (max, not sum — otherwise a re-delivered gossip message double-counts). Real uses: Redis Active-Active, Figma/Automerge/Yjs for collaborative editing, Riak data types. State the limit: anything with a global invariant ("balance ≥ 0") can't be a CRDT, and tombstone garbage grows.

### Q6: "Would you use eventual consistency for a payment system?"
**Hint:** Not for the ledger. But don't just say no — **decompose**. The account *balance* and the double-entry ledger write need a strongly consistent transaction (money is conserved; you can't "eventually" un-spend it). But the transaction *history page*, the notification, the fraud-scoring pipeline, the monthly-spend dashboard, and the rewards-points counter are all perfectly happy being eventually consistent. Showing you can split one product along the consistency axis is the answer they're actually looking for.

---

## Practice exercise

### Build a Conflict-Resolving Replicated Store

Write a small Node.js simulation — no database, no network, just classes — that makes you *feel* the anomaly.

**Produce a single file `eventual.js` containing:**

1. **A `Replica` class** holding an in-memory `Map` of `key → { value, versionVector }`. Give it:
   - `write(key, value)` — bumps its own counter in the version vector.
   - `read(key)` — returns the local value (possibly stale — that's the point).
   - `receiveGossip(key, incoming)` — merges an incoming `{value, versionVector}` using your `VersionVector.compare()` from section 4.

2. **A `Cluster` class** with three replicas and:
   - `gossip()` — pushes each replica's state to the others.
   - `partition(replicaId)` / `heal(replicaId)` — a boolean flag that makes `gossip()` skip that replica.

3. **Three scenarios, printed to the console:**
   - **Scenario A (stale read):** write `bio` to replica A, immediately read it from replica C *before* gossiping. Show the stale value. Gossip. Read again. Show convergence.
   - **Scenario B (silent data loss):** implement `resolveLWW()` and drive a conflict where the write that happened **later in real time** carries an **earlier timestamp** (simulate a 5-second-fast clock on one replica). Print the winner. Print, explicitly, the value that was destroyed.
   - **Scenario C (honest conflict):** partition replica B. Write `cart=[milk,eggs]` on A and `cart=[milk,bread]` on B. Heal. Gossip. Assert that `compare()` returns `CONCURRENT`, then merge with a **set union** and print the final cart plus its merged vector `{A:2, B:1}`.

4. **Then answer, in comments at the bottom of the file:**
   - In Scenario C, what would a set-union merge do if B's write had been **"remove milk"**? Which real-world bug is that?
   - Which of your three keys (`bio`, `counter`, `cart`) could be replaced by a **CRDT**, and which one *cannot* be — and why?

**Success criterion:** you can run `node eventual.js` and watch Scenario B destroy data without printing a single error. That silence is the lesson.

---

## Quick reference cheat sheet

- **The promise:** if writes stop, all replicas *eventually* converge. That is the entire guarantee.
- **The two holes:** it says nothing about **when** (ms → minutes), and nothing about **what you read in the meantime**.
- **Replication lag** is the width of the inconsistency window. Graph it. Alert on it. It's your most important consistency metric.
- **Read-your-own-writes:** users forgive stale data from others; they never forgive their *own* write vanishing. Fix: sticky reads to the leader for a short window, an LSN token, or a local echo.
- **Monotonic reads:** never let a user's view of time run backwards. Fix: pin a user to one replica by hashing their ID.
- **Consistent prefix:** never show an effect before its cause. Fix: keep causally-related writes on the **same partition** (choose your partition key to match your causality).
- **LWW** is the default in Cassandra: simple, and it **silently discards data**. "Later" is decided by a wall clock, and clocks lie.
- **Version vectors** detect causality: one vector dominating the other means safe to overwrite; neither dominating means **CONCURRENT** — a real conflict.
- **Vector clocks detect conflicts; they don't resolve them.** Resolution needs domain knowledge.
- **DynamoDB returns siblings** and makes *you* merge. Union-merging a cart is why a deleted item can come back.
- **Deletes need tombstones.** Absence is not data — without a tombstone, a merge will resurrect what you removed.
- **CRDT** = a merge that is **commutative + associative + idempotent** ⇒ convergence is a theorem. G-Counter merges with element-wise **max** (never sum).
- **CRDT limit:** no global invariants (`balance ≥ 0` can't be a CRDT), and tombstone metadata grows forever.
- **Fine eventually consistent:** likes, views, followers, feeds, DNS, search indexes, analytics, caches.
- **Not fine:** account balances, inventory at zero stock, seat booking, unique usernames, permission revocation.
- **The rule of thumb:** can two users briefly see different truths, and is a wrong answer cheap and self-correcting? Two yeses → go eventual.
- **Consistency is per-operation, not per-system.** Buy coordination only for the endpoints guarding a real invariant.
- **Hide it:** optimistic UI (show the user their own action instantly) + **idempotent** self-healing reconciliation (`SET` to an absolute value, never `INCR`).

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [09 — CAP Theorem](./09-cap-theorem.md) — why you were forced to choose availability in the first place |
| **Next** | [77 — Distributed Transactions](./77-distributed-transactions.md) — what you do for the operations that *can't* be eventual (money, stock, seats) |
| **Related** | [08 — Consistency Models](./08-consistency-models.md) — the full spectrum this doc sits at the far end of |
| **Related** | [63 — Database Replication](./63-database-replication.md) — the leader/follower machinery that creates the lag window |
| **Related** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — multi-leader and leaderless setups, where conflicts become unavoidable |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — async events are eventual consistency by construction |
