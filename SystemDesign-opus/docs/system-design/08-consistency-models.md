# 08 — Consistency Models — Strong, Eventual, and Everything Between

## Category: HLD Fundamentals

---

## What is this?

A **consistency model** is the *promise* a distributed system makes about what you will see when you read data that lives on more than one machine.

Here's the whole problem in one sentence: **the moment your data exists on two computers, "the current value" stops being a well-defined thing.** Server A might have the new value while server B still has the old one, and there is no god's-eye view that says which one is "right." A consistency model is the contract that says exactly how much staleness, reordering, and weirdness you're allowed to see — so you can build correct software on top of it.

Think of a rumour spreading through an office. The moment you tell one person, there's a window where some people know and some don't. **Strong consistency** = a magic megaphone: the instant you speak, *everyone* has heard, simultaneously. **Eventual consistency** = the rumour spreads person-to-person; give it five minutes and everyone will know, but right now Bob in accounting is confidently telling people the old version.

---

## Why does it matter?

Because if you don't pick a consistency model deliberately, you get one **accidentally** — and it's almost always the weakest one, discovered in production at 2am.

**What breaks without this knowledge:**
- A user posts a comment, the page reloads, and their comment **isn't there.** They post it again. Now there are two. (Read-your-own-writes violation — the single most common consistency bug on the internet.)
- Your bank app shows a balance of $500 on one screen and $300 on another because two reads hit two different replicas.
- Two people buy the last concert ticket. Both get a confirmation email. One of them is going to be very angry at a venue door.
- A user deletes an embarrassing photo, refreshes, and **sees it again.** They panic-delete it three more times.

**Interview angle:** Almost every distributed-system design question funnels into this. "How do you handle a user seeing their own post immediately when the feed is served from a read replica?" "Is your inventory count strongly consistent?" "What happens if two writes race?" If your answer is "we'll use a database," you've said nothing. This doc gives you the vocabulary to say something.

**Real-work angle:** Every time you add a read replica, a cache, or a CDN, you have *silently chosen a weaker consistency model*. Recall from [06 — Latency vs Throughput](./06-latency-and-throughput.md) that a cross-continent round trip costs ~150ms. Strong consistency across regions means paying that 150ms **on every write, sometimes twice**. Weaker consistency is how you go fast — and every fast system on the internet is quietly built on it.

---

## The core idea — explained simply

### The Shared Whiteboard Analogy

Imagine a company with offices in **New York**, **London**, and **Tokyo**. Each office has a whiteboard, and the whiteboards are supposed to show the same thing: the current price of a product.

Someone in New York walks up and writes **$99** (previously it was $120).

**What should happen in London and Tokyo?** There are several possible contracts, and each is a real consistency model:

**Contract A — "Nobody reads until everyone is updated" (Strong / Linearizable).**
Before the New York write is accepted, we phone London and Tokyo, wait for them to write $99 too, and only then tell New York "done." During that phone call, *nobody anywhere* can read the price — the boards are locked. Result: it is **impossible** for anyone, anywhere, to ever see the old price after the write succeeded. Beautiful. Also: the write took 150ms (Tokyo is far away), and if the Tokyo office's phone line is down, **the write cannot happen at all.** Nobody can change the price anywhere on Earth because Tokyo is unreachable.

**Contract B — "Write locally, tell the others when you can" (Eventual).**
New York writes $99 on its board immediately and says "done" in 2ms. A courier is dispatched to London and Tokyo. For the next few hundred milliseconds, **London and Tokyo are showing $120 while New York shows $99.** They are all "correct" according to their own board. Eventually — hence the name — everyone converges on $99. Result: writes are blazing fast, the system keeps working even if Tokyo is unreachable, and a customer in Tokyo just bought the item at the old price.

**Contract C — "At least *you* see your own change" (Read-your-own-writes).**
The New York employee who wrote $99 will *always* see $99 when they look, even if they walk over to the Tokyo office. Everyone else might still see $120 for a while. This sounds like a weird half-measure, and it is — but it's exactly the right trade for social media, and it's what fixes the "I posted a comment and it vanished" bug.

**Contract D — "Cause before effect" (Causal).**
Alice writes "Is the price still $120?" on the board. Bob, having *read Alice's question*, writes "No, it's $99." Causal consistency guarantees that **anyone who sees Bob's answer must already have seen Alice's question.** You never see a reply before the thing it's replying to. But two *unrelated* writes (New York changing a price, Tokyo changing a description) may appear in different orders in different offices — and that's fine, because nobody cares.

### Mapping the analogy back

| Whiteboard | System | What you get | What you pay |
|---|---|---|---|
| The three boards | Database replicas | Redundancy, local reads | They can disagree |
| Phone everyone before acknowledging | **Synchronous replication / consensus (Raft, Paxos)** | Strong consistency | Slow writes; a partition blocks writes entirely |
| Write locally, courier later | **Async replication / gossip** | Fast writes, always available | Stale reads, conflicts |
| The moment where boards disagree | **Replication lag** (usually ms; can be seconds) | — | The window where bugs live |
| "I always see my own change" | **Read-your-own-writes / sticky routing** | Sane UX, cheap | Only helps *that one user* |
| "Replies never precede questions" | **Causal consistency** | Conversations make sense | Needs version vectors to track |
| "Everyone converges given time" | **Eventual consistency** | Max availability & speed | You must handle conflicts |

**The single sentence to remember:**
> **Consistency is a spectrum, and you buy it with latency and availability. There is no free strong consistency.**

---

## Key concepts inside this topic

### 1. Why this problem exists at all (replication lag)

You have one database. Reads are slow because everyone hammers it. So you add a **read replica** — a second copy of the database that serves reads. The primary handles writes and streams them to the replica.

```
   WRITE                                READ
     │                                    │
     ▼                                    ▼
 ┌─────────┐   async replication    ┌─────────┐
 │ PRIMARY │ ─────────────────────▶ │ REPLICA │
 │ x = 99  │      (takes 50ms)      │ x = 120 │  ← STALE for 50ms
 └─────────┘                        └─────────┘
```

That 50ms window is **replication lag**, and it is where every consistency bug in your career will live. Under normal load it's a few milliseconds. Under heavy write load, during a schema migration, or when the replica is doing a backup, it can balloon to **seconds or minutes**. A user's write completes, they immediately read, the read is routed to the replica, and their data isn't there.

This is not a bug in the database. This is **the physics of having more than one machine.** Consistency models are the vocabulary for deciding what to do about it.

### 2. The spectrum — from strongest to weakest

Here is the map. Everything below is a point on this line:

```
 STRONGER ◀───────────────────────────────────────────────────────────▶ WEAKER
 (slower, less available)                        (faster, more available)

 Linearizable ─▶ Sequential ─▶ Causal ─▶ Read-your-writes ─▶ Monotonic ─▶ Eventual
     │              │            │             │                │            │
  "one global    "one global  "cause      "you see your    "you never    "converges
   order, in      order, not   before      own changes"     go back in    someday"
   real time"     necessarily  effect"                       time"
                  real time
```

Each model to the left of another is **strictly stronger** — it guarantees everything the weaker one does, plus more. And each step to the left costs you latency, availability, or both.

Let's walk them.

### 3. Strong consistency (Linearizability) — "it behaves like one machine"

**The promise:** the system behaves *as if* there were exactly one copy of the data. Once a write completes, **every** subsequent read — from any client, on any replica — returns that new value or a newer one. Never an older one.

The formal name is **linearizability**. The intuition: every operation appears to take effect **instantaneously at some single point in time** between when you called it and when it returned. There is one global timeline, and it respects **real time**.

**The story.** You transfer $500 from savings to checking. The transfer returns success. You immediately refresh your checking balance on a different device, connected to a different data centre. Under linearizability, you are **guaranteed** to see the $500. Anything else would be a bug, and for a bank, an unacceptable one.

**How it's achieved:** the write is not acknowledged until enough replicas have durably accepted it, coordinated by a consensus protocol (Raft, Paxos) or synchronous replication.

```
 Client                 Leader              Follower-1          Follower-2
   │                      │                     │                   │
   ├─ write(x=99) ───────▶│                     │                   │
   │                      ├─ replicate ────────▶│                   │
   │                      ├─ replicate ─────────────────────────────▶
   │                      │◀──── ACK ───────────┤                   │
   │                      │◀──── ACK ───────────────────────────────┤
   │                      │  (majority reached — now it's committed)│
   │◀──── OK ─────────────┤                     │                   │
   │                      │                     │                   │
   │  ANY read after this point, on ANY replica, returns 99.
```

**The costs — both of them matter:**
1. **Latency.** The write can't finish until a majority of replicas confirm. If your replicas span continents, that's the speed of light: ~150ms per round trip, and consensus often needs more than one.
2. **Availability.** If the leader can't reach a majority (a network partition), it **must refuse to serve writes.** It cannot know whether another leader has been elected on the other side of the partition, and serving a write anyway could produce two conflicting histories. Refusing to serve is the *only* safe move. This is exactly the trade-off that [09 — CAP Theorem](./09-cap-theorem.md) formalizes: **during a partition, you cannot have both consistency and availability.**

**Where you find it:** etcd, ZooKeeper, Google Spanner, single-node PostgreSQL/MySQL, DynamoDB with `ConsistentRead: true`, Redis with `WAIT`. Use it for: money, inventory counts, unique-username claims, distributed locks, leader election, anything where a wrong answer is worse than a slow answer.

### 4. Sequential consistency — "one order, but not necessarily *the* real-time order"

**The promise:** all replicas see all operations in **the same order**, and each individual client's operations appear in the order that client issued them. But that global order **doesn't have to match real time.**

**The story.** Alice writes `x=1` at 10:00:00.000. Bob writes `x=2` at 10:00:00.005 (5ms later, in wall-clock reality). Under **linearizability**, everyone must agree the final value is `2`, because Bob's write really did happen after Alice's. Under **sequential consistency**, the system is allowed to order them as `[Bob's x=2, then Alice's x=1]` and settle on `1` — as long as *every* replica agrees on that same order.

Is that useful? It's still very strong: no replica ever disagrees with another about the sequence of events, so you never get two replicas with different final values. It's just that a client who *knows* (out of band — say, Bob phoned Alice) that Bob wrote later can be surprised.

Sequential consistency is mostly a **theoretical waypoint** — it's the model of memory in multi-threaded programming. In practice, distributed systems people say "strong consistency" and mean linearizability. Know the distinction so you can name it if an interviewer probes; don't build your design around it.

### 5. Causal consistency — "you never see the reply before the question"

**The promise:** operations that are **causally related** are seen by everyone in the same order. Operations that are **concurrent** (unrelated) may be seen in different orders by different people, and that's OK.

"Causally related" means: if operation B *could have been influenced by* A — because the same client did A then B, or because someone **read** A before doing B — then B causally follows A.

**The story — the one everyone recognizes.** On a social network:
1. **Alice** posts: *"I lost my job today 😞"*
2. **Bob** reads it and replies: *"Congratulations!!"*

Wait — Bob is being cruel? No. Bob actually replied to a *different* post. But here's the horror scenario **without** causal consistency: Carol's replica receives Bob's reply **before** it receives Alice's post. Carol sees a comment thread that makes no sense, or worse, sees Bob's "Congratulations!!" attached to a post about losing a job that arrived a moment later.

Real version, from an actual class of bug:
1. Alice posts a photo.
2. Alice deletes the photo.
3. Bob's replica gets the **delete** before the **post** (they took different network paths). The delete is a no-op — there's nothing to delete. Then the post arrives. **The photo Alice deleted is now permanently visible on Bob's replica.**

Causal consistency prevents exactly this. The delete carries metadata saying "I causally depend on the post," so Bob's replica *holds the delete in a buffer* until the post arrives, then applies both in order.

**How it's achieved:** each write carries a **version vector** (or vector clock) — a small map of "what I had seen when I made this write." A replica delays applying a write until it has already applied everything that write depends on. (Full mechanics live in [76 — Eventual Consistency in Practice](./76-eventual-consistency.md).)

```js
/**
 * Causal delivery, in miniature. Each write carries the set of
 * write-IDs the author had already seen. We refuse to apply a write
 * until all of its dependencies are locally applied.
 */
class CausalReplica {
  constructor(name) {
    this.name = name;
    this.applied = new Set();  // ids of writes we have applied
    this.store = new Map();
    this.pending = [];         // writes waiting on their causes
  }

  /** A write: { id, deps: [ids], key, value } */
  receive(write) {
    this.pending.push(write);
    this.drain();
  }

  /** Repeatedly apply anything whose causes are all satisfied. */
  drain() {
    let progress = true;
    while (progress) {
      progress = false;
      for (let i = 0; i < this.pending.length; i++) {
        const w = this.pending[i];
        const ready = w.deps.every(d => this.applied.has(d));
        if (!ready) continue;              // still waiting on a cause — hold it
        this.store.set(w.key, w.value);
        this.applied.add(w.id);
        this.pending.splice(i, 1);
        progress = true;                   // applying one may unblock another
        break;
      }
    }
  }
}

const carol = new CausalReplica('carol');

// The network delivers the DELETE first — out of causal order.
carol.receive({ id: 'w2', deps: ['w1'], key: 'photo:7', value: null });     // delete
console.log(carol.store.has('photo:7'), carol.pending.length); // false, 1  ← buffered, not lost

// Now the original POST arrives. Both get applied, in the right order.
carol.receive({ id: 'w1', deps: [], key: 'photo:7', value: 'beach.jpg' });  // post
console.log(carol.store.get('photo:7'), carol.pending.length); // null, 0   ← correctly deleted
```

**Why it's a great deal:** causal consistency is the **strongest model you can have while remaining available during a network partition.** You get "the world makes sense" without paying the consensus tax. Used by: COPS, Facebook's TAO (approximately), MongoDB's causally consistent sessions, Azure Cosmos DB's "Consistent Prefix"/"Session" tiers.

### 6. Read-your-own-writes — "at minimum, don't gaslight the author"

**The promise:** after *you* write something, *you* will always read it back. Says nothing about what other people see.

**The story — the classic bug.**
1. Alice posts a comment. The write goes to the **primary**. Success!
2. The page reloads and fetches the comments. The load balancer routes this read to a **read replica**.
3. The replica hasn't received the write yet (50ms of replication lag).
4. **Alice's comment is not on the page.** She assumes it failed and posts it again.
5. Both writes eventually replicate. **Now there are two identical comments.**

This bug has shipped at basically every company. It is invisible in testing (localhost has zero replication lag) and painfully visible in production.

**Three ways to fix it, cheapest first:**

**Fix 1 — Read from the primary for a short window after a write.** Simple, effective, and it doesn't overload the primary because only recently-active writers pay the cost.

```js
/**
 * Read-your-own-writes via sticky routing.
 * After a user writes, route THEIR reads to the primary for a
 * window slightly longer than the observed replication lag.
 */
class ReadYourWritesRouter {
  constructor({ primary, replicas, stickyWindowMs = 500 }) {
    this.primary = primary;
    this.replicas = replicas;
    this.stickyWindowMs = stickyWindowMs;
    this.lastWriteAt = new Map();   // userId -> timestamp. Use Redis in a real fleet;
                                    // an in-memory map only works if the same node
                                    // handles the user's next request.
  }

  async write(userId, key, value) {
    await this.primary.set(key, value);
    this.lastWriteAt.set(userId, Date.now());
  }

  async read(userId, key) {
    const last = this.lastWriteAt.get(userId) ?? 0;
    const insideWindow = Date.now() - last < this.stickyWindowMs;

    // The user wrote recently → they might not see themselves on a replica.
    // Pay the primary-read cost for THIS user only.
    if (insideWindow) return this.primary.get(key);

    // Everyone else gets a cheap replica read.
    const replica = this.replicas[Math.floor(Math.random() * this.replicas.length)];
    return replica.get(key);
  }
}
```

**Fix 2 — Write-through the UI.** Don't re-fetch at all. Optimistically render the comment you just submitted from local state. This is what React apps do, and it's why the bug often *looks* fixed while still being present in the API.

**Fix 3 — Logical timestamps.** The write returns a version number (an LSN, a Lamport timestamp). The client sends it with the next read: *"give me a view at least this fresh."* The replica waits — or forwards to the primary — until it has caught up. This is exactly what MongoDB's **causally consistent sessions** and DynamoDB's consistent reads do.

**Warning:** "the user" is not always "the device." Alice writes on her phone, then opens her laptop. If your sticky window is keyed to a session or a server node, the laptop read hits a stale replica and the bug is back. Key it to the **user ID**, in a shared store (Redis).

### 7. Monotonic reads — "time must not run backwards"

**The promise:** if you read a value once, you will never subsequently read an **older** value. You may see stale data — but never *increasingly* stale data.

**The story.**
1. Alice refreshes a post. Her read hits **Replica-1** (caught up). She sees **12 comments**.
2. She refreshes again 2 seconds later. The load balancer round-robins her to **Replica-2**, which is 30 seconds behind. She sees **8 comments**.
3. **Four comments just vanished.** Alice is now convinced your site is broken. She is right.

Without monotonic reads, users **time-travel backwards** every time they bounce between replicas of differing lag. This feels far worse than plain staleness — a slightly-old page is fine; a page that *un-loads* content is terrifying.

**The fix is delightfully cheap:** make each user **sticky to one replica**. Hash their user ID to pick a replica and always send them there. They may see stale data, but the staleness never *increases*, because a single replica only ever moves forward.

```js
// Monotonic reads via consistent user→replica pinning.
// Same user → same replica → the replica only ever moves forward in time.
function pickReplica(userId, replicas) {
  let hash = 0;
  for (const ch of String(userId)) hash = (hash * 31 + ch.charCodeAt(0)) | 0;
  return replicas[Math.abs(hash) % replicas.length];
}
// Caveat: if that replica dies, the user is re-pinned to another one and
// CAN go backwards once. Accept it, or track a version watermark per user.
```

Note that this is a *different* guarantee from read-your-own-writes. Read-your-writes is about **your writes**. Monotonic reads is about **other people's writes** — it says the world you observe only moves forward. You usually want both, and pinning by user ID gives you a decent approximation of both at once.

### 8. Monotonic writes — "my own edits apply in the order I made them"

**The promise:** *If one client issues write A and then write B, every replica applies A before B.* Your own writes are never reordered relative to each other.

**The story.** You're editing a blog draft from your laptop.

1. `10:00:00` — you save. Title becomes `"Untitled draft"`.
2. `10:00:04` — you save again. Title becomes `"Why Quorums Work"`.

Both writes are accepted. But write #1 was routed to a node in Virginia and write #2 (after a load-balancer reshuffle) went to a node in Oregon. They replicate at different speeds, and a third replica happens to receive **#2 first, then #1**. It applies them in arrival order. Final stored title: **`"Untitled draft"`**.

Your later edit was silently overwritten by your *earlier* one. You reload, see the old title, and conclude the save button is broken. This is the "**my changes keep reverting**" bug, and it is the most infuriating one on this list because the data isn't corrupt — it's just *from the past*.

```
   You ──write A ("Untitled draft")──▶ [ Node-VA ] ─────slow link (2s)────┐
                                                                          ▼
   You ──write B ("Why Quorums Work")─▶ [ Node-OR ] ──fast link (50ms)──▶ [ Replica-3 ]

   Replica-3 applies in ARRIVAL order:  B  then  A
   Final value: "Untitled draft"   ← your newer edit lost to your older one
```

**The practical fixes:**

1. **Route all writes from one session through the same leader/partition.** If A and B go through the same node, that node imposes an order and ships them down one log. This is why systems hash a session (or a user ID) to a fixed partition. Cheapest fix, and the one most systems actually use.
2. **Attach a per-client sequence number**, and have replicas refuse to apply a write out of order (buffer it, or drop it if a newer one from the same client already landed).

```js
// Per-session sequence numbers. The client stamps every write with a
// counter it increments locally; replicas refuse to go backwards.
class MonotonicWriter {
  constructor(sessionId) {
    this.sessionId = sessionId;
    this.seq = 0;
  }

  async write(store, key, value) {
    // ++this.seq is issued in program order, so B always carries a
    // strictly higher seq than A — no matter which node each one lands on.
    return store.put(key, value, { sessionId: this.sessionId, seq: ++this.seq });
  }
}

class Replica {
  constructor() {
    this.data = new Map();
    this.lastSeqBySession = new Map(); // sessionId -> highest seq applied
  }

  put(key, value, { sessionId, seq }) {
    const lastSeq = this.lastSeqBySession.get(sessionId) ?? 0;

    // Already applied a NEWER write from this same client? Then this one
    // arrived late and is stale. Dropping it is correct — applying it
    // would resurrect the user's older edit on top of their newer one.
    if (seq <= lastSeq) return { applied: false, reason: 'stale-out-of-order' };

    this.data.set(key, value);
    this.lastSeqBySession.set(sessionId, seq);
    return { applied: true };
  }
}
```

Note how narrow this guarantee is: it says nothing about *other* clients' writes interleaving with yours. If your colleague edits the same draft, monotonic writes will not save you — that's a genuine **write conflict**, and you need causal consistency or explicit conflict resolution (topic 76).

Monotonic writes, monotonic reads, and read-your-own-writes are collectively called **session guarantees**: they only promise things about *one client's own view of its own session*. That narrowness is exactly why they're cheap — no consensus required, just routing and a counter — and why they deliver most of the *perceived* correctness of strong consistency for a tiny fraction of the cost.

### 9. Eventual consistency — "converges, given enough quiet"

**The promise, stated honestly:** *"If no new writes are made, then eventually all replicas will converge to the same value."*

Read that carefully. It is a remarkably weak promise. It says **nothing** about:
- **How long** "eventually" is (100ms? 10 minutes? It's unbounded in the definition.)
- What you see **in the meantime** (anything — old values, values that go backwards, values that flip-flop).
- What happens when **two writes conflict** (that's a separate problem you must solve yourself).

**The story.** You update your profile picture. The write hits a replica in Virginia. Your friend in Singapore refreshes your profile and sees... the old picture. Then refreshes again and sees the new one. Then hits a different edge cache and sees the old one *again*. Eventually — usually within a second — the whole world sees the new one. **Nobody is harmed.** A profile picture is not worth 150ms of cross-continent consensus on every write.

**The conflict problem.** Two people edit the same shopping cart from two devices, simultaneously, on two different replicas. Both writes succeed locally. Now the replicas exchange gossip and discover they disagree. What's the answer?

- **Last-Write-Wins (LWW):** compare timestamps, newest wins. Simple, and it **silently destroys data** — the other write is just gone. Clock skew makes it worse. Cassandra's default.
- **Merge / CRDTs:** define the data so conflicts *can't* happen. A shopping cart as a set → union the two carts. A like-counter as a grow-only counter → sum the increments. This is what Amazon's Dynamo paper described ("the cart merges, and a deleted item may reappear — but you never lose an add").
- **Keep both, let the app decide:** Riak-style siblings. Correct, but you've now pushed the problem into your business logic.

Conflict resolution is a whole topic. It's [76 — Eventual Consistency in Practice](./76-eventual-consistency.md), and it covers vector clocks, LWW, and CRDTs properly.

**Where you find it:** DNS (the canonical example — a TTL is literally a bounded staleness window), CDNs, Cassandra/DynamoDB by default, S3 (historically), every read-replica setup, every cache you've ever added.

### 10. Quorum intuition — W + R > N (a preview)

Here's the beautiful trick that lets you **tune** consistency instead of choosing one extreme.

Suppose you store every piece of data on **N = 3** replicas.

- **W** = how many replicas must **acknowledge a write** before you call it successful.
- **R** = how many replicas you must **read from** before you return an answer (you take the newest of what you get back).

**The rule:**

```
   W + R > N   ⟹   the write set and the read set MUST overlap
                    ⟹  at least one replica you read from has the latest write
                    ⟹  strong (quorum) consistency
```

Why? Pigeonhole principle. If you wrote to `W` replicas and read from `R` replicas out of `N`, and `W + R > N`, those two sets cannot be disjoint. **At least one replica is in both.** That replica has the new value, and since you take the newest value returned, you get it.

Draw it:

```
       N = 3 replicas:   [ R1 ]   [ R2 ]   [ R3 ]

  W=2, R=2  →  W + R = 4 > 3   ✓ STRONG
       write to:  ███R1███ ███R2███
       read from:          ███R2███ ███R3███
                              ▲
                    OVERLAP — R2 has the new value. Guaranteed fresh.

  W=1, R=1  →  W + R = 2 ≯ 3   ✗ EVENTUAL
       write to:  ███R1███
       read from:                   ███R3███
                    NO OVERLAP — R3 may be stale. Fastest possible, weakest guarantee.
```

Now you can **dial the knob per operation:**

| Config (N=3) | W+R | Guarantee | Best for |
|---|---|---|---|
| **W=3, R=1** | 4 > 3 | Strong. Writes slow (all replicas), reads instant. | Read-heavy: config, feature flags |
| **W=2, R=2** | 4 > 3 | Strong. Balanced. **The standard default.** | General purpose |
| **W=1, R=3** | 4 > 3 | Strong. Writes instant, reads slow. | Write-heavy logging with rare strong reads |
| **W=1, R=1** | 2 ≯ 3 | **Eventual.** Fastest, always available. | Metrics, view counts, profile pics |
| **W=2, R=1** | 3 ≯ 3 | Eventual (just barely misses). | — |

And here's the punchline that connects to the next topic: **W=2 with N=3 means you can lose one replica and still write.** But if a partition leaves you able to reach only **one** replica, a `W=2` system **must refuse the write**. It chose consistency over availability. Set `W=1` and it will happily accept the write — choosing availability over consistency. That choice, forced on you by the network, is [09 — CAP Theorem](./09-cap-theorem.md).

Full mechanics — sloppy quorums, hinted handoff, read repair, anti-entropy — are in [86 — Data Replication Strategies](./86-data-replication-strategies.md).

---

## Visual / Diagram description

### Diagram 1: The consistency spectrum, with costs

```
 STRONGEST                                                              WEAKEST
 (slow, coordinated, fragile under partition)   (fast, uncoordinated, partition-proof)
 ═══════════════════════════════════════════════════════════════════════════════════▶

 ┌──────────────┐ ┌────────────┐ ┌──────────┐ ┌─────────────┐ ┌──────────┐ ┌──────────┐
 │ LINEARIZABLE │ │ SEQUENTIAL │ │  CAUSAL  │ │ READ-YOUR-  │ │MONOTONIC │ │ EVENTUAL │
 │              │ │            │ │          │ │   WRITES    │ │  READS   │ │          │
 │ "one copy,   │ │ "one order,│ │ "cause   │ │ "you always │ │ "never   │ │ "we all  │
 │  real time"  │ │  any time" │ │  before  │ │  see your   │ │  go back │ │  agree…  │
 │              │ │            │ │  effect" │ │  own edits" │ │  in time"│ │  someday"│
 └──────────────┘ └────────────┘ └──────────┘ └─────────────┘ └──────────┘ └──────────┘
   Raft, Paxos,     (mostly         COPS,        Sticky           Sticky      Cassandra
   Spanner,          theory /        TAO,         primary          replica     DynamoDB
   etcd, ZK          CPU memory)     Cosmos DB    reads            per user    DNS, CDN

 ◀── needs consensus ──▶│◀────────── available during a partition ──────────────────▶
                        │
      ✗ A partition BLOCKS writes  │  ✓ Keep accepting writes on BOTH sides
        (see topic 09 — CAP)       │    (then you must resolve conflicts — topic 76)
```

### Diagram 2: The bug — read-your-own-writes violated, and fixed

```
  ✗ BROKEN — Alice's comment vanishes                ✓ FIXED — sticky primary read
  ─────────────────────────────────────              ────────────────────────────────

    Alice                                              Alice
      │ 1. POST /comment                                 │ 1. POST /comment
      ▼                                                  ▼
  ┌────────┐                                         ┌────────┐
  │   LB   │                                         │   LB   │  marks: alice wrote
  └───┬────┘                                         └───┬────┘  at t=0 (in Redis)
      │                                                  │
      ▼                                                  ▼
  ┌─────────┐  50ms lag  ┌─────────┐               ┌─────────┐  50ms lag ┌─────────┐
  │ PRIMARY │───────────▶│ REPLICA │               │ PRIMARY │──────────▶│ REPLICA │
  │ +comment│  (in-flight)│ (stale) │               │ +comment│           │ (stale) │
  └─────────┘            └────┬────┘               └────┬────┘           └─────────┘
                              │                         │
      │ 2. GET /comments (10ms later)                   │ 2. GET /comments (10ms later)
      ▼                       │                         │    LB sees alice wrote <500ms ago
  ┌────────┐                  │                    ┌────────┐   → route to PRIMARY
  │   LB   │──── routes to ───┘                    │   LB   │──────────┘
  └────────┘     the replica                       └────────┘
      │                                                  │
      ▼                                                  ▼
   [ no comment ]  ← 💥 Alice re-posts.            [ comment is there ]  ← ✓
   [ duplicate!  ]     Now there are two.
```

### Diagram 3: The quorum overlap that makes strong consistency work

```
                          N = 5 replicas
              ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
              │ R1 │  │ R2 │  │ R3 │  │ R4 │  │ R5 │
              └────┘  └────┘  └────┘  └────┘  └────┘

  WRITE with W=3   ▓▓▓▓▓▓  ▓▓▓▓▓▓  ▓▓▓▓▓▓
                   (v=99)  (v=99)  (v=99)     R4, R5 still hold v=120

  READ  with R=3                   ░░░░░░  ░░░░░░  ░░░░░░
                                   (v=99)  (v=120) (v=120)
                                      ▲
                                   OVERLAP — R3 is in BOTH sets.
                                   We read {99, 120, 120}, take the NEWEST
                                   version → 99.  ✓ Guaranteed correct.

    W + R = 3 + 3 = 6  >  N = 5     ✓ strong
    W + R = 2 + 2 = 4  ≯  N = 5     ✗ eventual — the sets can miss each other entirely
```

You cannot draw two sets of size 3 from 5 slots without them touching. That's it — that's the whole proof, and it's why `W + R > N` works.

---

## Real world examples

### 1. Amazon DynamoDB — consistency as a per-request parameter

DynamoDB makes the trade-off an **API argument**. Every `GetItem` call takes a `ConsistentRead` boolean:

```js
// Eventually consistent read — the default. Cheaper (0.5 RCU), faster,
// and may return data from a replica that hasn't caught up yet.
await ddb.getItem({ TableName: 'Users', Key: { id: { S: 'u1' } } });

// Strongly consistent read — 2x the cost (1 RCU), higher latency,
// and NOT available across regions in a global table.
await ddb.getItem({ TableName: 'Users', Key: { id: { S: 'u1' } }, ConsistentRead: true });
```

This is the design lesson: **consistency is not a property of your database, it's a property of each operation.** Use the strong read for "check the account balance before a withdrawal." Use the eventual read for "render the user's display name." Choosing strong-everywhere means paying double for reads that never needed it.

### 2. Facebook / Meta — the read-your-own-writes problem at planet scale

Facebook's architecture historically routed all writes to a primary region and served reads from local read-replicas worldwide (their TAO graph store sits in front of MySQL). This is an eventually-consistent read path — which produces exactly the "I commented and it disappeared" bug for a user in Singapore whose write went to Virginia.

Their documented approach is a **sticky/session-based mechanism**: after a user performs a write, their subsequent reads for a short window are routed to a cache/replica known to have the write (or to the primary), giving that user read-your-own-writes while everyone else continues to get fast, cheap, eventually-consistent local reads. It's the exact pattern in the `ReadYourWritesRouter` above, at a scale of billions of users. The takeaway: **you don't need global strong consistency; you need each user to not be gaslit about their own actions.**

### 3. Google Spanner — buying linearizability with atomic clocks

Spanner is the famous counter-example that says "you *can* have global strong consistency, if you're willing to buy hardware." Google put **GPS receivers and atomic clocks in every datacenter** to bound clock uncertainty to a few milliseconds. Its `TrueTime` API doesn't return a timestamp — it returns an *interval* `[earliest, latest]` that provably contains the true time.

To commit a transaction, Spanner **deliberately waits out the uncertainty interval** (a "commit wait") before acknowledging. That wait — typically a handful of milliseconds — is the price of a globally consistent, externally-consistent (linearizable) ordering of transactions across continents. It's the purest illustration of the central trade in this doc: **Spanner did not get strong consistency for free. It literally bought the latency with a hardware clock budget, and then paid it, on every commit.**

---

## Trade-offs

| Model | Guarantee | Latency cost | Available during partition? | Use it for |
|---|---|---|---|---|
| **Linearizable (strong)** | Reads always see the latest write | High — consensus round trips (~ms local, ~150ms+ global) | **No** — must reject writes | Money, inventory, unique IDs, locks, leader election |
| **Sequential** | Global order, but not real-time order | High | No | Mostly theoretical / CPU memory models |
| **Causal** | Cause always precedes effect | Low-medium — version vectors, small metadata | **Yes** (strongest model that stays available) | Comment threads, chat, collaborative editing, social graphs |
| **Read-your-own-writes** | You see your own edits | Low — sticky routing only | Yes | Any user-generated content (posts, comments, profile edits) |
| **Monotonic reads** | You never see older data than before | Very low — pin user to a replica | Yes | Feeds, timelines, anything a user refreshes |
| **Eventual** | Converges *eventually* | Lowest — write locally, return | Yes | View counts, likes, DNS, CDN content, profile pictures, analytics |

| Choice | Pros | Cons |
|---|---|---|
| **Strong everywhere** | Simple to reason about — behaves like one machine. No conflict logic to write. | Slow writes; writes stop during a partition; doesn't scale across regions without exotic hardware; you pay for consistency on data that never needed it |
| **Eventual everywhere** | Fast, cheap, always available, scales globally | You *will* ship the duplicate-comment bug; you must write conflict-resolution logic; "eventually" is unbounded and your users notice |
| **Mixed (per-operation)** | Pay only where correctness demands it | You must actually *think* about each piece of data. Requires discipline and a DB that supports it. |

| Quorum config (N=3) | Guarantee | Trade |
|---|---|---|
| **W=2, R=2** | Strong (`4 > 3`) | Balanced; survives 1 dead replica for both reads and writes. **The safe default.** |
| **W=3, R=1** | Strong (`4 > 3`) | Instant reads; **any** dead replica blocks all writes |
| **W=1, R=1** | Eventual (`2 ≯ 3`) | Fastest and most available; stale reads, conflicts |

**The sweet spot:** **Mixed, chosen per data type.** Ask one question of every field in your data model: *"If a user sees a 5-second-old value here, does anything bad happen?"* If the answer is "someone loses money / double-books a seat / gets a duplicate account," use **strong**. If the answer is "they see 41 likes instead of 42," use **eventual** — and then bolt on **read-your-own-writes + monotonic reads via sticky routing**, because those two are nearly free and they fix 95% of the *perceived* weirdness. Most systems are 95% eventual and 5% strong, and the whole skill is knowing which 5%.

---

## Common interview questions on this topic

### Q1: "What's the difference between strong and eventual consistency?"
**Hint:** Strong (linearizable) = once a write completes, *every* subsequent read from *any* replica returns it or something newer — the system behaves as if there's one copy. Eventual = replicas converge *if writes stop*, but in the meantime you can read stale data, and you must resolve conflicts yourself. The cost is the point: strong requires coordination (consensus round trips) and must **refuse writes during a network partition**; eventual requires no coordination and stays available. Name the price out loud — that's what separates a memorized answer from an understood one.

### Q2: "A user posts a comment and doesn't see it after refreshing. What happened and how do you fix it?"
**Hint:** Classic **read-your-own-writes violation**. The write went to the primary; the read was load-balanced to a replica that hasn't caught up (replication lag). Fixes, cheapest first: (1) route that user's reads to the **primary** for a short window after their write, keyed by **user ID in a shared store** (not session or node, or it breaks when they switch devices); (2) optimistically render the comment client-side; (3) have the write return a version token/LSN and require the read to be "at least this fresh." Bonus: mention that this is worse than it looks — the user re-posts, and now you have duplicates, which is why you also want idempotency (topic 85).

### Q3: "What does W + R > N mean, and why does it work?"
**Hint:** N = replicas per item, W = replicas that must ack a write, R = replicas you read from. If `W + R > N`, the write set and the read set must **overlap** (pigeonhole principle) — so at least one replica you read from has the newest write, and taking the newest version returned gives you a fresh read. With N=3, `W=2, R=2` is the standard strong config. `W=1, R=1` gives eventual consistency and maximum speed. The killer follow-up you should preempt: **a higher W means you need more replicas alive to accept a write** — which is exactly how the quorum setting becomes a CAP choice during a partition.

### Q4: "Give me an example where causal consistency matters but strong consistency is overkill."
**Hint:** A comment thread. If Bob's reply is delivered to a replica before Alice's original post, the thread is nonsense — or worse, a `delete` arrives before the `create` it deletes, and the deleted item becomes permanently visible. Causal consistency prevents that by tagging each write with its dependencies and buffering until they're satisfied. But you do *not* need every replica on Earth to agree on the exact global ordering of two unrelated users' posts — that would cost consensus latency for zero user benefit. **Causal is the strongest model that still survives a network partition**, which makes it the best deal on the spectrum.

### Q5: "Your read replicas have 30 seconds of lag. What guarantees can you still offer users, and how?"
**Hint:** You can't offer strong consistency, but you can offer the two that actually fix the *perceived* bugs, and both are cheap: **read-your-own-writes** (sticky the writer to the primary for a window) and **monotonic reads** (hash-pin each user to one replica so they never bounce to a *more* stale one and watch data disappear). Then: expose the staleness (a "last updated 30s ago" label beats silent lies), and identify the operations that genuinely need freshness (checkout, balance, inventory) and route *only those* to the primary. Also: 30 seconds of lag is itself an incident — go find out whether it's write volume, a long-running transaction, or a single-threaded apply on the replica.

### Q6: "Is a cache a consistency problem?"
**Hint:** Yes — a cache is just an under-managed replica with no replication protocol. The moment you add Redis in front of Postgres, you have two copies of the data and a staleness window governed by your TTL and invalidation strategy. Everything in this doc applies: your cache TTL *is* your bounded-staleness guarantee; cache invalidation is a conflict-resolution problem; and a user reading through a cache after their own write hits the exact read-your-own-writes bug. This is why "there are only two hard things in computer science: cache invalidation and naming things" is really a joke about consistency.

---

## Practice exercise

### The Consistency Audit — Design a Social Feed

You are designing a **social media app** (posts, likes, comments, follower counts, DMs, and a user profile). It runs in **3 regions** (US, EU, Asia), each with a full read replica. Writes go to a primary in the US. Replication lag is normally **80ms**, but spikes to **8 seconds** under load.

**Part A — Classify every piece of data (20 min).**

Build this table and fill in every row. For each, state the **weakest consistency model that is acceptable**, and **one concrete sentence describing the bad thing that happens** if you go weaker than that.

| Data | Weakest acceptable model | What breaks if weaker |
|---|---|---|
| A user's own new **post** | ? | ? |
| Someone else's **post** in your feed | ? | ? |
| **Like count** on a post | ? | ? |
| **Comment thread** ordering | ? | ? |
| **Follower count** | ? | ? |
| **Direct message** delivery | ? | ? |
| **Username** (must be globally unique) | ? | ? |
| **Blocked-users list** (used to filter the feed) | ? | ? |
| A user's **account balance** (for paid tips) | ? | ? |

Two of those rows are traps. Think hard about **blocked-users** (what happens if the block hasn't replicated when the feed renders?) and **username uniqueness** (what happens if two people claim `@alice` on two replicas simultaneously?).

**Part B — Fix the two worst bugs (15 min).**

1. **The vanishing comment.** Alice (in the EU) comments on a post. Her write goes to the US primary. Her page re-fetches from the EU replica, 80ms later. Write the routing logic (as JS pseudocode or a diagram) that guarantees she sees her own comment. State exactly what you key the stickiness on, and explain what breaks if you key it on the HTTP session instead of the user ID.

2. **The time-travelling feed.** Bob refreshes and sees 12 comments, refreshes again and sees 8. Explain in one sentence why this happens, name the consistency model being violated, and give the cheapest fix. Then state the one scenario where your fix **still** lets him go backwards, and decide whether you care.

**Part C — Quorum math (10 min).**

You migrate the **username registry** to a quorum store with **N = 5** replicas.
3. List every `(W, R)` pair that gives you **strong consistency**. Show that `W + R > 5` for each.
4. For `W=3, R=3`: how many replicas can you lose and still accept writes? How many and still serve reads?
5. A network partition splits the replicas **3-and-2**. With `W=3`, which side can still register a new username? What must the other side do, and what does that tell you about the CAP theorem (topic 09)?

**Produce:** the completed table, the routing pseudocode, and short written answers to 3-5. One page.

---

## Quick reference cheat sheet

- **Consistency model = the promise about what you'll read** when data lives on more than one machine. Not "is the data valid" — that's a database constraint, a different thing.
- **The root cause is replication lag.** Primary writes, replica catches up 50ms later, and every bug in this doc lives in that window.
- **The spectrum:** Linearizable → Sequential → **Causal** → Read-your-own-writes → Monotonic reads → Eventual. Left = safer + slower. Right = faster + weirder.
- **Linearizable (strong)** = "behaves like one copy, in real time." Requires consensus. **Blocks writes during a partition.** Use for money, inventory, unique names, locks.
- **Causal** = "cause always precedes effect" — you never see a reply before the question, or a delete before the create. **The strongest model that survives a partition.** The best deal on the spectrum.
- **Read-your-own-writes** = "you always see *your* edits." Fixed with **sticky primary reads keyed on user ID**. Fixes the #1 real-world consistency bug: the vanishing comment → the duplicate comment.
- **Monotonic reads** = "you never go backwards in time." Fixed by **pinning each user to one replica**. Fixes the disappearing-comments-on-refresh horror.
- **Eventual** = "converges *if writes stop*." Says nothing about how long, or what you see meanwhile, or how conflicts resolve. Powers DNS, CDNs, Cassandra, DynamoDB defaults.
- **Conflicts are the hidden cost of eventual.** LWW (simple, silently loses data), CRDTs/merge (correct, restrictive), or keep-both-and-ask-the-app. See topic 76.
- **`W + R > N` ⟹ strong.** The read set and write set must overlap (pigeonhole). N=3: `W=2,R=2` is the default. `W=1,R=1` is eventual and fastest.
- **A higher W = fewer partitions you can survive.** This is where quorum tuning becomes a CAP decision.
- **A cache is an unmanaged replica.** Every guarantee in this doc applies to your Redis layer and your CDN, whether you thought about it or not.
- **You cannot buy strong consistency for free.** Spanner bought it with atomic clocks and a mandatory commit wait. Everyone else pays with latency or availability.
- **The design question for every field:** *"If a user sees a 5-second-old value here, does anything actually break?"* Most systems are **95% eventual, 5% strong** — the skill is knowing which 5%.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — the replicas you added for availability are exactly what created this problem |
| **Next** | [09 — CAP Theorem](./09-cap-theorem.md) — the formal statement of the trade you've just been previewing: during a partition, you must pick consistency **or** availability |
| **Related** | [76 — Eventual Consistency in Practice](./76-eventual-consistency.md) — the deep dive: vector clocks, last-writer-wins, CRDTs, and how conflicts actually get resolved |
| **Related** | [63 — Database Replication](./63-database-replication.md) — where replication lag comes from, sync vs async, and how failover works |
| **Related** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — quorums (W+R>N) in full detail, plus gossip, read repair, and Merkle trees |
| **Related** | [66 — Database Transactions and Isolation Levels](./66-database-transactions-and-isolation.md) — the *single-node* cousin of this topic: isolation is to transactions what consistency is to replicas |
