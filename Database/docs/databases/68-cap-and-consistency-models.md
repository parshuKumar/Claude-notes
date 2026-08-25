# 68 — CAP and Consistency Models
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

Two shopkeepers, one in Mumbai and one in Delhi, selling tickets from a shared count of 100. They phone each other after every sale.

**The phone line goes down.** Neither can reach the other. A customer walks in.

There are exactly two choices, and no third:

1. **"Sorry, I can't sell you a ticket until I can reach Delhi."** — the count is never wrong, but you turned away a paying customer. **You chose consistency over availability.**
2. **"Certainly, here you go."** — you made the sale, but Delhi might be selling the same seat right now. **You chose availability over consistency.**

★ **That is CAP, and the thing everyone gets wrong is this: you do not get to choose in advance.** The partition chooses *for* you. Your only decision is **what to do when it happens** — and if you never decided, your system has already decided, usually badly.

And the part almost nobody says out loud: ★ **when the phone line is working, you don't have to choose at all.** You can have both. The trade only exists during the partition — which is a small fraction of the time, and is *not* the interesting trade-off in most systems. **The interesting one is latency versus consistency, and that one applies every single day.**

---

## Where this fits in the big picture

```
   51 2PC — blocking · 58 replicas — staleness · 63 HA — RPO vs RTO
   52 idempotency · 60 sharding · 44/50 isolation levels
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 68 CAP & CONSISTENCY MODELS ← YOU ARE HERE   │
        │ ★ the formal names for trades you've already │
        │   been making                                 │
        └────────────────────┬─────────────────────────┘
                             ▼
              69 security · Phase 8 (70–76) · Phase 9 capstones
```

★ **This topic is deliberately near the end.** CAP taught first becomes a slogan people repeat without meaning. Taught after you have measured replication lag, chosen an isolation level, watched a sync standby block commits, and built an outbox, it becomes **the vocabulary that names what you already did**.

---

## What is this?

**The CAP theorem** (Brewer 2000; Gilbert & Lynch proof 2002):

> A distributed data store cannot simultaneously provide all three of **C**onsistency (linearizability), **A**vailability (every non-failing node answers), and **P**artition tolerance (the system works despite dropped messages).

★ **And immediately, the four things everyone gets wrong:**

```
 ✗ ① "PICK TWO OF THREE."
    ⇒ ★ YOU CANNOT PICK P. Networks partition. It is a property
      of reality, not a design choice.
    ⇒ ★ THE REAL CHOICE IS: WHEN A PARTITION HAPPENS, DO YOU
      SACRIFICE C OR A?  ⇒ CP or AP. There is no CA.

 ✗ ② "C MEANS CONSISTENCY LIKE IN ACID."
    ⇒ ★ NO. CAP's C is LINEARIZABILITY — every read sees the most
      recent completed write, as if there were one copy.
    ⇒ ACID's C is "the database enforces your constraints."
    ⇒ ★ COMPLETELY DIFFERENT PROPERTIES SHARING A LETTER.

 ✗ ③ "A MEANS UPTIME."
    ⇒ ★ NO. CAP's A means EVERY non-failing node returns a
      NON-ERROR response. A system that returns 503 during a
      partition is CP, and it may still have 99.99% uptime,
      because partitions are rare.
    ⇒ ★ CAP-availability and business-availability are different
      words.

 ✗ ④ "MY SYSTEM IS AP" / "MY SYSTEM IS CP."
    ⇒ ★ CAP CLASSIFIES OPERATIONS, NOT SYSTEMS. The same database
      can be CP for payments and AP for view counts, and should be.
```

★ **And the extension that matters more day to day — PACELC** (Abadi 2012):

```
   ★ IF (P)artition:  choose (A)vailability or (C)onsistency
   ★ ELSE:            choose (L)atency or (C)onsistency

 ⇒ ★ THE "ELSE" BRANCH IS THE ONE YOU LIVE WITH EVERY DAY.
   Even with a perfect network, a synchronous write costs a round
   trip. Reading from a replica is faster and staler.
 ⇒ ★ PACELC DESCRIBES THE ACTUAL ENGINEERING TRADE. CAP describes
   the rare emergency.
```

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE YOU HAVE ALREADY MADE THESE CHOICES, MOSTLY BY DEFAULT.

   asynchronous replication (58)      ⇒ ★ you chose A over C
   synchronous_standby_names = 'n2'   ⇒ ★ you chose C over A —
                                        and if n2 dies, ALL WRITES
                                        BLOCK (Topic 63)
   reading from a replica             ⇒ ★ you chose L over C
   an outbox instead of 2PC (52)      ⇒ ★ you chose A over C
   READ COMMITTED (44)                ⇒ ★ you chose L over C
   sticky-after-write routing (58)    ⇒ ★ you bought back a
                                        specific C guarantee for a
                                        specific L cost

 ⇒ ★ THE VALUE OF THE VOCABULARY IS THAT IT MAKES THESE VISIBLE
   AND ARGUABLE. "We're eventually consistent" is not a design.
   "Read-your-writes for the authoring user, monotonic reads for
   everyone else, bounded at 5 seconds" is.
```

---

## The physical reality

### The consistency ladder — strongest to weakest

```
 ★ THESE ARE NOT VAGUE. EACH IS A PRECISE GUARANTEE ABOUT WHAT A
   READ MAY RETURN.

 ★ ① LINEARIZABILITY (CAP's "C", "strong consistency")
    every operation appears to take effect at a single instant
    between its start and its completion, and all clients agree
    on the order.
    ⇒ ★ THE TEST: if write W completes at 10:00:00.000, then any
      read STARTING after that instant MUST see W — from any node.
    ⇒ COST: ★ a round trip to a quorum on every operation.
    ⇒ PostgreSQL: ★ a single primary is linearizable for its own
      writes; ★ reading a replica is NOT.

 ★ ② SEQUENTIAL CONSISTENCY
    all clients see the same ORDER of operations, but that order
    need not match real time.
    ⇒ ★ weaker than linearizable: a read may return a stale value,
      but never an out-of-order one.

 ★ ③ CAUSAL CONSISTENCY
    if A happened-before B, everyone who sees B also sees A.
    ⇒ ★ concurrent (unrelated) operations may be seen in different
      orders by different clients.
    ⇒ ★ THE SWEET SPOT FOR MANY SYSTEMS: it forbids the anomalies
      users actually notice ("I replied to a comment that isn't
      there yet") without requiring global coordination.

 ★ ④ EVENTUAL CONSISTENCY
    if writes stop, all replicas eventually converge.
    ⇒ ★ SAYS NOTHING ABOUT WHEN, OR ABOUT WHAT YOU SEE BEFORE THEN.
    ⇒ ★ "eventually consistent" without a BOUND and a RECONCILER
      is not a design — it is a hope (Topic 51).

 ★ AND THE FOUR CLIENT-CENTRIC GUARANTEES — the ones that map
   directly onto user-visible bugs:
   ★ READ-YOUR-WRITES   you see your own writes
                        ⇒ ★ "the save button doesn't work" (58)
   ★ MONOTONIC READS    you never see time go backwards
                        ⇒ ★ refresh shows an older value —
                          two replicas at different lag
   ★ MONOTONIC WRITES   your writes apply in the order you made them
   ★ WRITES-FOLLOW-READS  a write based on a read is ordered after it

 ⇒ ★ THESE FOUR ARE MUCH CHEAPER THAN LINEARIZABILITY AND FIX
   ALMOST ALL USER-VISIBLE PROBLEMS. This is the practical insight.
```

### What PostgreSQL actually gives you

```
 ★ SINGLE PRIMARY, NO REPLICAS
   ⇒ ★ LINEARIZABLE for writes and for reads on the primary.
   ⇒ ★ AND THIS IS WHY A SINGLE-NODE POSTGRES IS SO PLEASANT:
     none of this is your problem.

 ★ PRIMARY + ASYNC REPLICAS
   reads on the primary  ⇒ ★ linearizable
   reads on a replica    ⇒ ★ EVENTUAL. No bound without measurement.
   ⇒ ★ AND NOT EVEN MONOTONIC across replicas: two replicas at
     different lag mean refreshing can show OLDER data.
     ⇒ ★ FIX: sticky routing to one replica per session, or LSN
       routing (Topic 58).

 ★ PRIMARY + SYNCHRONOUS REPLICA
   synchronous_commit = on          ⇒ ★ DURABLE on the replica,
                                      ★ NOT VISIBLE there
   synchronous_commit = remote_apply ⇒ ★ visible ⇒ linearizable
                                      reads on that replica
   ⇒ ★ THIS DISTINCTION IS THE SINGLE MOST MISUNDERSTOOD SETTING
     IN POSTGRESQL REPLICATION (Topic 58).

 ★ AND THE FAILURE MODE THAT MAKES PostgreSQL "CP":
   with a single named synchronous standby, losing it ★ BLOCKS ALL
   COMMITS. The primary is up and unavailable.
   ⇒ ★ THAT IS CP, CHOSEN BY A CONFIGURATION LINE MOST PEOPLE
     COPY WITHOUT READING (Topic 63).
```

### The proof, in one paragraph — because it's short and it settles arguments

```
 ★ GILBERT & LYNCH, INFORMALLY:

 Two nodes, N1 and N2, holding value v0. The network partitions.
 A client writes v1 to N1. N1 cannot reach N2.
 Another client reads from N2.

 ⇒ If N2 answers, it answers v0 ⇒ ★ NOT LINEARIZABLE (a completed
   write is invisible).
 ⇒ If N2 refuses to answer ⇒ ★ NOT AVAILABLE.
 ⇒ If N2 waits for N1 ⇒ ★ it never answers ⇒ NOT AVAILABLE.

 ★ THERE IS NO FOURTH OPTION. That is the entire theorem.

 ⇒ ★ AND THE PRACTICAL COROLLARY: any system claiming "CA" is
   either not distributed, or has not thought about partitions.
```

### Quorums — how distributed systems buy consistency back

```
 ★ N replicas · W = write quorum · R = read quorum

   ★ IF W + R > N, every read quorum overlaps every write quorum
     ⇒ ★ at least one node in any read has the latest write.

   N=3, W=2, R=2  ⇒ 4 > 3  ✓ ★ strong-ish, tolerates 1 failure
   N=3, W=3, R=1  ⇒ ★ fast reads, ★ writes fail if ANY node is down
   N=3, W=1, R=1  ⇒ ★ 2 ≯ 3 ⇒ eventual. Fast, no guarantee.
   N=5, W=3, R=3  ⇒ ★ tolerates 2 failures

 ⇒ ★ THIS IS WHAT DynamoDB, Cassandra AND Riak EXPOSE AS A KNOB,
   and what Raft/Paxos systems (etcd, CockroachDB, Spanner) fix
   internally at W=R=majority.

 ★ WHAT W+R>N DOES NOT GIVE YOU:
   ✗ ★ it is NOT linearizability by itself — without a coordination
     protocol you can still read a value that is later rolled back,
     and concurrent writes still need conflict resolution.
   ⇒ ★ "quorum reads = strong consistency" is a common and
     dangerous simplification.
```

### CRDTs and last-write-wins — how AP systems converge

```
 ★ AN AP SYSTEM ACCEPTS CONCURRENT CONFLICTING WRITES.
   IT MUST DECIDE WHAT THE MERGED VALUE IS.

 ★ ① LAST-WRITE-WINS (LWW) — by timestamp
    ✗ ★ SILENT DATA LOSS. Two concurrent writes ⇒ one vanishes.
    ✗ ★ AND CLOCK SKEW DECIDES THE WINNER. A node 200 ms ahead
      wins every conflict.
    ⇒ ★ acceptable ONLY when losing a write is genuinely fine
      (a cache, a "last seen at" timestamp).

 ★ ② CRDTs — types whose merge is mathematically deterministic
    ★ G-Counter    grow-only counter: merge = per-node max, summed
    ★ PN-Counter   two G-Counters (increments, decrements)
    ★ OR-Set       add/remove set with unique tags per add
    ★ LWW-Register a register with a tie-broken timestamp
    ⇒ ★ merge is COMMUTATIVE, ASSOCIATIVE and IDEMPOTENT
      ⇒ ★ replicas converge regardless of message order or
        duplication — which is exactly what an unreliable network
        delivers.
    ✗ ★ THE COST: metadata grows, and ★ not every operation has a
      CRDT ("subtract 1 but never below zero" does not).

 ★ ③ APPLICATION-LEVEL MERGE — keep both, let the user decide
    ⇒ git's merge conflicts; Amazon's original shopping cart
    ⇒ ★ correct when the business has an opinion the database
      cannot infer.

 ⇒ ★ AND THE OPTION MOST SYSTEMS SHOULD TAKE:
   ★ DON'T ALLOW THE CONFLICT. A single writer per key (sharding,
   Topic 60) makes conflict resolution unnecessary — which is why
   most successful systems are CP for their core data and AP only
   at the edges.
```

---

## How it works — step by step

### Classifying each operation, not the system

```
 ★ FOR EVERY OPERATION, ASK FOUR QUESTIONS:

 ① ★ WHAT HAPPENS IF THIS IS STALE BY 5 SECONDS?
    nothing         ⇒ eventual is fine
    a confusing UI  ⇒ read-your-writes / monotonic reads
    money is lost   ⇒ linearizable

 ② ★ WHAT HAPPENS IF THIS OPERATION IS REFUSED DURING A PARTITION?
    a user retries in 30 s  ⇒ ★ CP is fine
    revenue stops           ⇒ ★ AP, plus reconciliation

 ③ ★ CAN TWO CONCURRENT WRITES CONFLICT?
    no (single writer per key)  ⇒ ★ no merge needed
    yes, commutative            ⇒ ★ a CRDT
    yes, non-commutative        ⇒ ★ coordination, or an explicit
                                  business rule

 ④ ★ IS THERE AN INVARIANT ACROSS ITEMS?
    "stock must not go negative", "at most N seats"
    ⇒ ★ THIS REQUIRES COORDINATION. It cannot be made AP without
      either over-selling and compensating, or reserving capacity
      per partition in advance.
```

### The practical guarantees, implemented

```js
// ★ ① READ-YOUR-WRITES — the guarantee that fixes the most bugs
//    (Topic 58's LSN routing)
async function withWrite(ctx, fn) {
  const r = await primary.query(fn);
  const { rows: [{ lsn }] } = await primary.query(
    'SELECT pg_current_wal_lsn() AS lsn');
  ctx.setMinLsn(lsn);              // ★ signed cookie / session
  return r;
}
function pickRead(ctx) {
  const need = ctx.getMinLsn();
  if (!need) return anyReplica();
  const ready = replicas.filter(r => lsnGte(r.replayLsn, need));
  return ready.length ? pick(ready) : primary;   // ★ provably current
}
```

```js
// ★ ② MONOTONIC READS — never show time going backwards
//    the cheapest possible fix: pin a session to ONE replica
function pickReplica(sessionId) {
  const healthy = replicas.filter(r => r.healthy);
  if (!healthy.length) return primary;
  // ★ consistent hashing on the session ⇒ the same replica every time
  return healthy[hash(sessionId) % healthy.length];
}
// ⇒ ★ a single replica's LSN only ever advances, so a pinned
//   session never sees an older value.
// ⇒ ★ WITHOUT THIS, two replicas at different lag make "refresh"
//   show older data — a bug users report as "it's flickering".
```

```js
// ★ ③ CAUSAL CONSISTENCY across services — propagate the LSN
//    (a "causal token", the same idea as a vector clock)
app.use((req, res, next) => {
  req.causalToken = req.get('X-Causal-LSN') ?? null;
  const send = res.json.bind(res);
  res.json = (body) => {
    if (req.dbCtx?.minLsn) res.set('X-Causal-LSN', req.dbCtx.minLsn);
    return send(body);
  };
  next();
});
// ⇒ ★ service B, receiving the token, will not read from a replica
//   that hasn't replayed past it. Causality is preserved ACROSS
//   the service boundary — which is where it usually breaks.
```

### Bounding "eventual" — the part people skip

```sql
-- ★ EVENTUAL CONSISTENCY REQUIRES THREE THINGS (Topic 51).
--   Teams routinely ship only the first.

-- ★ ① A MEASURED BOUND, not an assumption
SELECT application_name,
       extract(epoch from replay_lag) AS lag_s,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
  FROM pg_stat_replication;
-- ★ record p50/p99/p999 continuously. YOUR BOUND IS THE p999.

-- ★ ② A RECONCILER for what falls outside the bound
SELECT count(*) FROM orders
 WHERE status = 'pending_reservation'
   AND created_at < now() - interval '10 minutes';
-- ★ alert > 0, and a job that resolves them

-- ★ ③ AN ALERT WHEN THE BOUND IS EXCEEDED
-- ★ alert: replay_lag_s > stated_bound
-- ⇒ ★ WITHOUT ② AND ③ YOU HAVE INCONSISTENCY WITH EXTRA STEPS.
```

### Writing the contract down

```markdown
## Consistency contract — orders service

| Operation | Model | Bound | Rationale |
|---|---|---|---|
| `POST /orders` | ★ linearizable | — | primary only; money |
| `GET /orders/:id` (own) | ★ read-your-writes | — | LSN routing |
| `GET /orders` (list) | ★ monotonic reads | ★ p999 380 ms | session-pinned replica |
| `GET /products/:id` | eventual | ★ 60 s | CDN, `stale-while-revalidate` |
| `POST /reserve-stock` | ★ linearizable | — | invariant across items |
| `GET /analytics/*` | eventual | ★ 15 min | rollup table |
| view counts | ★ eventual, CRDT | ★ 5 s | commutative; loss acceptable |

★ DURING A PARTITION (primary unreachable from a region):
  writes  ⇒ ★ CP — return 503. We do not accept orders we cannot
           guarantee.
  reads   ⇒ ★ AP — serve from the regional replica, with a
           `X-Data-Age` header, and a UI banner above 30 s.

★ SIGNED OFF: product owner, 2026-08-25.
```
```
 ★ THIS TABLE IS THE DELIVERABLE. Not the theory — the table.
   It turns "we're eventually consistent" into a set of specific,
   testable, arguable promises.
```

---

## Concept breakdown

```
★ CAP — AND THE FOUR MISREADINGS
   you CANNOT pick P ⇒ ★ the real choice is CP or AP DURING a
   partition. ★ There is no CA.
   ✗ CAP's C ≠ ACID's C  (★ linearizability vs constraint
     enforcement)
   ✗ CAP's A ≠ uptime    (★ every non-failing node answers)
   ✗ ★ CAP CLASSIFIES OPERATIONS, NOT SYSTEMS

★ PACELC — the one that matters daily
   if (P): A or C.  ★ ELSE: L or C.
   ⇒ ★ the ELSE branch applies every day, with a perfect network

★ THE LADDER
   linearizable > sequential > ★ causal > eventual
   ★ + FOUR CLIENT-CENTRIC GUARANTEES, which are cheap and fix
     almost every user-visible bug:
     ★ read-your-writes · ★ monotonic reads · monotonic writes ·
     writes-follow-reads

★ POSTGRESQL, PRECISELY
   single primary            ⇒ ★ linearizable
   async replica reads       ⇒ ★ eventual, ★ not even monotonic
                               across replicas
   synchronous_commit=on     ⇒ ★ DURABLE there, ★ NOT VISIBLE
   ★ =remote_apply           ⇒ visible ⇒ linearizable on that replica
   ★ single named sync standby ⇒ losing it BLOCKS ALL COMMITS = CP,
     chosen by a config line people copy without reading

★ QUORUMS
   ★ W + R > N ⇒ read and write quorums overlap
   ⇒ ★ BUT THAT ALONE IS NOT LINEARIZABILITY — a common and
     dangerous simplification

★ CONVERGENCE IN AP SYSTEMS
   ★ LWW ⇒ silent data loss, and ★ CLOCK SKEW picks the winner
   ★ CRDTs ⇒ commutative/associative/idempotent merge; metadata
     grows; ★ not every operation has one
   application merge ⇒ when the business has an opinion
   ★ BEST: DON'T ALLOW THE CONFLICT — one writer per key

★ EVENTUAL CONSISTENCY NEEDS THREE THINGS
   ① a MEASURED bound (★ the p999, not the average)
   ② ★ a RECONCILER  ③ ★ an ALERT when the bound is exceeded
   ⇒ ★ without ② and ③ it is inconsistency with extra steps

★ THE DELIVERABLE IS A TABLE
   per operation: model · bound · rationale · ★ partition behaviour
   ⇒ signed off by the product owner
```

---

## Diagrams

**Diagram 1 — big picture: the ladder, with costs and PostgreSQL's position**

```
  STRONGER ▲                                    COST        POSTGRES
           │
  ★ LINEARIZABLE ───────────────────────  ★ quorum RTT   ★ primary
    "every read sees the latest             per op         reads;
     completed write, from any node"                       remote_apply
           │
    SEQUENTIAL ────────────────────────    ordering       —
    "everyone agrees on the ORDER,          coordination
     not on real time"
           │
  ★ CAUSAL ─────────────────────────────  ★ track         ★ LSN tokens
    "if A→B, anyone seeing B sees A"        dependencies    across
                                            (a token)       services
           │
    ★ READ-YOUR-WRITES ─────────────────  ★ per-session   ★ LSN routing
    ★ MONOTONIC READS                       state /         (58); session
    (client-centric)                        pinning         pinning
           │
    EVENTUAL ─────────────────────────────  ★ ~free        ★ async
    "if writes stop, replicas converge"                     replica reads
           │
  WEAKER   ▼

 ★ THE PRACTICAL INSIGHT: the FOUR CLIENT-CENTRIC guarantees sit
   low on this ladder and cost very little — and they eliminate
   almost every user-visible consistency bug.
   ⇒ ★ most systems need read-your-writes and monotonic reads,
     NOT linearizability.
```

**Diagram 2 — data flow: the partition, and the two choices**

```
                    ★ THE NETWORK PARTITIONS
   ┌──────────────┐        ╳╳╳╳╳╳        ┌──────────────┐
   │   MUMBAI     │◄──────╳      ╳──────►│    DELHI     │
   │  primary     │                       │  replica     │
   │  stock = 100 │                       │  stock = 100 │
   └──────┬───────┘                       └──────┬───────┘
          │                                      │
     client A                               client B
     "buy 1"                                "buy 1"

 ★ CHOICE 1 — CP (refuse)
 ┌────────────────────────────────────────────────────────────────┐
 │ Mumbai: ✓ sells (it is the primary)      ⇒ stock 99            │
 │ Delhi:  ★ 503 Service Unavailable                              │
 │                                                                 │
 │ ✓ ★ the count is NEVER WRONG                                   │
 │ ✗ ★ Delhi customers cannot buy for the duration                │
 │ ⇒ ★ CORRECT FOR: payments, inventory, seat booking, anything   │
 │   with an invariant across items                               │
 └────────────────────────────────────────────────────────────────┘

 ★ CHOICE 2 — AP (accept, reconcile later)
 ┌────────────────────────────────────────────────────────────────┐
 │ Mumbai: ✓ sells ⇒ 99      Delhi: ✓ sells ⇒ 99                  │
 │ ★ partition heals ⇒ two sales, one decrement                   │
 │                                                                 │
 │ ✓ ★ nobody was turned away                                     │
 │ ✗ ★ YOU MAY HAVE OVERSOLD ⇒ ★ you now owe a COMPENSATION       │
 │   (a refund, an apology, a substitute)                         │
 │ ⇒ ★ CORRECT FOR: likes, view counts, analytics, drafts,        │
 │   shopping carts — ★ and for commerce ONLY IF the business     │
 │   has an existing process for overselling.                     │
 └────────────────────────────────────────────────────────────────┘

 ★ THERE IS NO THIRD BOX. The partition forces the choice; your
   only decision is which one, made in advance.
```

**Diagram 3 — before/after: naming the guarantee changes the design**

```
 ✗ BEFORE — "we're eventually consistent"
 ┌───────────────────────────────────────────────────────────────┐
 │ all GETs → a random healthy replica                            │
 │                                                                │
 │ ★ BUG 1: "I saved and it reverted"                             │
 │   ⇒ read-your-writes violated                                  │
 │ ★ BUG 2: "refreshing shows OLDER data, then newer, then older" │
 │   ⇒ ★ monotonic reads violated — two replicas at different lag │
 │ ★ BUG 3: "I replied to a comment that doesn't exist"           │
 │   ⇒ ★ causal consistency violated ACROSS SERVICES              │
 │ ★ BUG 4: nobody can say how stale the data may be              │
 │   ⇒ ★ no bound, no reconciler, no alert                        │
 │                                                                │
 │ ★ AND: every one of these was reported as a DIFFERENT BUG and  │
 │   investigated separately.                                     │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — each guarantee named, priced and implemented
 ┌───────────────────────────────────────────────────────────────┐
 │ ★ READ-YOUR-WRITES   LSN in a signed cookie; a replica is used │
 │                      only if replay_lsn ≥ it (58)              │
 │                      ★ cost: 3.6% of reads go to the primary   │
 │ ★ MONOTONIC READS    session pinned to ONE replica by          │
 │                      consistent hashing                        │
 │                      ★ cost: slightly uneven replica load      │
 │ ★ CAUSAL (services)  X-Causal-LSN propagated across calls      │
 │                      ★ cost: one header                        │
 │ ★ BOUNDED EVENTUAL   p999 lag measured at 380 ms; ★ alert at   │
 │                      5 s; ★ reconciler for anything stuck      │
 │ ★ LINEARIZABLE       payments and stock: ★ primary only        │
 │                                                                │
 │ ⇒ ★ FOUR SEPARATE BUG CLASSES ELIMINATED BY FOUR NAMED         │
 │   GUARANTEES, none of which required linearizability.          │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Prove a replica is not linearizable.**
```bash
# ★ write on the primary, read the replica IMMEDIATELY
psql -h primary -tAc "
  INSERT INTO orders (customer_id, total_minor) VALUES (42, 500) RETURNING id"
```
```
 ★ 8842119
```
```bash
psql -h replica -tAc "SELECT count(*) FROM orders WHERE id = 8842119"
```
```
 ★ 0        — a COMPLETED write is invisible on the replica.
   ⇒ ★ NOT LINEARIZABLE, by definition.
```

**Prove replicas are not even monotonic relative to each other.**
```bash
for i in $(seq 1 20); do
  psql -h primary -tAc "INSERT INTO orders (customer_id,total_minor)
                        VALUES (1,1)" >/dev/null
  A=$(psql -h replica1 -tAc "SELECT count(*) FROM orders")
  B=$(psql -h replica2 -tAc "SELECT count(*) FROM orders")
  echo "r1=$A r2=$B"
done
```
```
 r1=412088 r2=412088
 r1=412091 r2=★ 412089
 r1=★ 412092 r2=412094
   ★ A CLIENT ALTERNATING BETWEEN THEM SEES THE COUNT GO
     BACKWARDS. ⇒ monotonic reads violated.
   ⇒ ★ FIX: pin a session to one replica.
```

**Prove `synchronous_commit = on` gives durability, not visibility.**
```sql
-- primary
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (replica1)';
SELECT pg_reload_conf();
SET synchronous_commit = on;
INSERT INTO orders (customer_id, total_minor) VALUES (99, 777) RETURNING id;
```
```
 ★ 8842200        — the commit waited for the replica to FLUSH.
```
```bash
psql -h replica1 -tAc "SELECT count(*) FROM orders WHERE id=8842200"
```
```
 ★ 0        — ★ DURABLE THERE, NOT YET VISIBLE THERE.
```
```sql
SET synchronous_commit = remote_apply;
INSERT INTO orders (customer_id, total_minor) VALUES (99, 888) RETURNING id;
```
```bash
psql -h replica1 -tAc "SELECT count(*) FROM orders WHERE id=8842201"
```
```
 ★ 1        — ★ remote_apply waits for REPLAY ⇒ visible ⇒
   linearizable reads on that replica.
```

**Prove a single named sync standby makes you CP.**
```sql
ALTER SYSTEM SET synchronous_standby_names = 'replica1';   -- ★ a single name
SELECT pg_reload_conf();
```
```bash
ssh replica1 'sudo systemctl stop postgresql'
psql -h primary -c "INSERT INTO orders (customer_id,total_minor) VALUES (1,1)"
```
```
 ★ (hangs indefinitely)
```
```sql
SELECT pid, wait_event_type, wait_event FROM pg_stat_activity
 WHERE wait_event = 'SyncRep';
```
```
  pid  | wait_event_type | wait_event
-------+-----------------+------------
 41202 | IPC             | ★ SyncRep
   ★ THE PRIMARY IS UP AND REFUSING TO COMMIT.
   ⇒ ★ THAT IS CP. Chosen by one configuration line.
```
```sql
-- ★ the quorum form restores availability
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (replica1, replica2)';
SELECT pg_reload_conf();   -- ⇒ writes resume immediately
```

**Implement read-your-writes and measure the cost.**
```js
const lsnGte = (a, b) => cmpLsn(a, b) >= 0;

async function write(ctx, sql, params) {
  const r = await primaryPool.query(sql, params);
  const { rows: [{ lsn }] } =
    await primaryPool.query('SELECT pg_current_wal_lsn() AS lsn');
  ctx.minLsn = lsn;
  return r;
}
function readPool(ctx) {
  if (!ctx.minLsn) return replicaPool();
  const ready = replicas.filter(r => r.healthy && lsnGte(r.replayLsn, ctx.minLsn));
  if (ready.length) { metrics.increment('route.replica'); return pick(ready).pool; }
  metrics.increment('route.primary_fallback');
  return primaryPool;
}
```
```
 ★ MEASURED over 24 hours:
   reads routed to a replica   ★ 96.4%
   primary fallback            ★ 3.6%
 ⇒ ★ read-your-writes bought for 3.6% of reads.
   ★ Linearizability would have cost 100%.
```

**Implement monotonic reads.**
```js
function pickReplica(sessionId) {
  const healthy = replicas.filter(r => r.healthy);
  if (!healthy.length) return primaryPool;
  return healthy[murmur3(sessionId) % healthy.length].pool;  // ★ pinned
}
```
```bash
# ★ re-run the alternating test with pinning
for i in $(seq 1 20); do
  C=$(curl -s -b "sid=abc" localhost:3000/api/orders/count)
  echo "$C"
done
```
```
 412088
 412091
 412092
 412094        ★ MONOTONIC. Never goes backwards.
```

**Simulate a partition and observe both choices.**
```bash
# ★ partition the replica from the primary
ssh replica1 'sudo iptables -A INPUT -s $PRIMARY_IP -j DROP'
sleep 30

# ★ AP behaviour: serve the stale replica with an age header
curl -sD- localhost:3000/api/products/7 | grep -i 'x-data-age'
```
```
 ★ X-Data-Age: 31
```
```bash
# ★ CP behaviour: refuse
curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/api/checkout
```
```
 ★ 503
```
```bash
ssh replica1 'sudo iptables -D INPUT -s $PRIMARY_IP -j DROP'
```

**A quorum, and why `W+R>N` is not linearizability.**
```
 ★ N=3, W=2, R=2 ⇒ 4 > 3. Every read quorum overlaps every write
   quorum.
 ★ BUT: client A writes to nodes 1 and 2 and CRASHES before
   acknowledging. Client B reads nodes 2 and 3 and sees A's value.
   Client C reads nodes 1 and 3 and — if node 1's write is later
   rolled back by a repair — may not.
 ⇒ ★ overlapping quorums guarantee you CAN see the latest write.
   They do not by themselves give a total order, or prevent reading
   a value that is later undone.
 ⇒ ★ REAL LINEARIZABILITY NEEDS A CONSENSUS PROTOCOL (Raft/Paxos),
   which is what etcd, CockroachDB and Spanner run.
```

**A CRDT counter, and why LWW loses data.**
```sql
-- ★ ① LWW — the naive approach
CREATE TABLE lww_counter (key text PRIMARY KEY, n bigint, updated_at timestamptz);
-- node A: n=5 at 10:00:00.100
-- node B: n=7 at 10:00:00.090   ★ (clock 200 ms behind ⇒ actually later)
-- merge by timestamp ⇒ ★ n=5. B's increment is GONE.
```
```sql
-- ★ ② A PN-COUNTER CRDT — converges regardless of order
CREATE TABLE pn_counter (
  key       text     NOT NULL,
  node_id   smallint NOT NULL,
  increments bigint  NOT NULL DEFAULT 0,
  decrements bigint  NOT NULL DEFAULT 0,
  PRIMARY KEY (key, node_id)
);
-- ★ each node only ever writes ITS OWN row ⇒ no conflict possible
INSERT INTO pn_counter (key, node_id, increments) VALUES ('views:7', 1, 1)
ON CONFLICT (key, node_id) DO UPDATE SET increments = pn_counter.increments + 1;

-- ★ the merged value
SELECT sum(increments) - sum(decrements) AS value
  FROM pn_counter WHERE key = 'views:7';
```
```
 ★ MERGE IS: per (key,node_id), take the MAX of each field.
   ⇒ commutative, associative, idempotent
   ⇒ ★ replicas converge whatever order messages arrive in, and
     duplicates are harmless.
 ⇒ ★ NOTE: this is exactly the sharded counter from Topic 61,
   with the shard being the NODE. The same structure solves
   contention locally and convergence distributed.
```

---

## Example 2 — production scenario

**The situation.** A collaborative document platform expands from one region to three (Mumbai, Singapore, Frankfurt) for latency. The architecture: one primary in Mumbai, async replicas in each region, reads served locally.

```
 AFTER LAUNCH
   read latency (Singapore)   ★ 210 ms → 12 ms   ✓ the goal
   ★ support tickets          ★ 340/week, in four distinct clusters
   ★ and one incident         a 4-minute partition between Mumbai
                              and Frankfurt
```

**Step 1 — classify the four ticket clusters.**

```
 ★ CLUSTER 1 (142/week) "I edited the title and it reverted"
   ⇒ ★ READ-YOUR-WRITES violated. The write went to Mumbai; the
     read came from the local replica, 180 ms behind.

 ★ CLUSTER 2 (88/week) "the document keeps flickering between
   versions"
   ⇒ ★ MONOTONIC READS violated. Two replicas in the same region
     behind a load balancer, at different lag.

 ★ CLUSTER 3 (74/week) "I got a notification about a comment that
   isn't there"
   ⇒ ★ CAUSAL CONSISTENCY violated ACROSS SERVICES. The
     notification service read the comment from the primary and
     sent an email; the user's browser read the document from a
     replica that hadn't replayed it.

 ★ CLUSTER 4 (36/week) "my collaborator's changes overwrote mine"
   ⇒ ★ LOST UPDATE (Topic 43) — unrelated to replication, but
     surfaced by the multi-region latency making the edit window
     longer.

 ⇒ ★ FOUR DISTINCT CLUSTERS, FOUR DIFFERENT GUARANTEES, AND THEY
   HAD ALL BEEN FILED AS "REPLICATION LAG BUGS".
```

**Step 2 — the incident, and what it revealed about the partition policy.**

```
 ★ 14:22 — the Mumbai↔Frankfurt link degrades for 4 minutes.
   Frankfurt's replica stops receiving WAL.
   ⇒ ★ Frankfurt users continued reading, silently, from data that
     grew to 4 minutes stale.
   ⇒ ★ NO ERROR. NO BANNER. NO HEADER. Users edited documents based
     on 4-minute-old content and their edits — sent to Mumbai —
     overwrote newer changes.
 ⇒ ★ THE SYSTEM HAD SILENTLY CHOSEN AP, AND NOBODY HAD DECIDED
   THAT. There was no policy at all.
```

**Step 3 — measure before choosing.**

```sql
-- ★ what IS the lag, per region, at p999?
SELECT application_name,
       percentile_cont(0.50) WITHIN GROUP (ORDER BY lag_ms) AS p50,
       percentile_cont(0.99) WITHIN GROUP (ORDER BY lag_ms) AS p99,
       percentile_cont(0.999) WITHIN GROUP (ORDER BY lag_ms) AS p999,
       max(lag_ms)
  FROM replica_lag_samples
 WHERE sampled_at > now() - interval '30 days'
 GROUP BY 1;
```
```
 application_name |  p50  |  p99   |  p999   |   max
------------------+-------+--------+---------+----------
 replica-mumbai   |   4.1 |   88.2 |   412.0 |    8,204
 replica-singapore| ★ 84.2| ★ 640.1| ★ 4,102 | ★ 41,204
 replica-frankfurt| ★ 91.8| ★ 712.4| ★ 4,882 | ★ 240,000
   ★ p999 IS 4–5 SECONDS. THE MAX IS FOUR MINUTES.
   ⇒ ★ any "bound" quoted from the p50 (84 ms) would be wrong by
     three orders of magnitude.
```

**Step 4 — write the contract, operation by operation.**

```markdown
## Consistency contract — documents platform  (signed 2026-08-25)

| Operation | Model | Bound | Partition behaviour |
|---|---|---|---|
| `PUT /docs/:id` | ★ linearizable | — | ★ CP: 503 |
| `GET /docs/:id` (author) | ★ read-your-writes | — | ★ CP: 503 |
| `GET /docs/:id` (reader) | ★ monotonic reads | ★ 5 s | ★ AP + `X-Data-Age` |
| `GET /docs` (list) | eventual | ★ 5 s | AP + banner > 30 s |
| `POST /comments` | ★ linearizable | — | CP: 503 |
| comment notifications | ★ causal | — | ★ deferred until visible |
| view counts | ★ eventual (CRDT) | 60 s | AP, always |
| `GET /search` | eventual | ★ 5 min | AP, always |

★ ABOVE 30 SECONDS OF STALENESS: the UI shows a banner
  "Showing data from N seconds ago — reconnecting".
★ ABOVE 120 SECONDS: reads switch to CP (503) rather than serve
  data old enough to cause a lost update.
```

**Step 5 — implement each guarantee.**

```js
// ★ ① READ-YOUR-WRITES — LSN in a signed cookie (Topic 58)
async function updateDoc(req, id, patch) {
  const r = await primary.query(
    'UPDATE documents SET body=$1, version=version+1 WHERE id=$2 AND version=$3',
    [patch.body, id, patch.version]);
  if (r.rowCount === 0) throw new ConflictError();     // ★ Topic 49
  const { rows:[{ lsn }] } =
    await primary.query('SELECT pg_current_wal_lsn() AS lsn');
  req.setCausalLsn(lsn);
  return r;
}
```

```js
// ★ ② MONOTONIC READS — pin the session to one replica
function replicaFor(sessionId, region) {
  const pool = regionReplicas[region].filter(r => r.healthy && r.lagMs < 5000);
  if (!pool.length) return null;                        // ⇒ fall back
  return pool[murmur3(sessionId) % pool.length];        // ★ stable
}
```

```js
// ★ ③ CAUSAL CONSISTENCY ACROSS SERVICES — the notification bug
//    ✗ before: notify immediately after the comment commits
//    ✓ after: notify via the OUTBOX (Topic 52), and the notifier
//      waits until the reader's region can SEE the comment.
async function onCommentCreated(event) {
  const targetRegion = await regionOfUser(event.recipient_id);
  // ★ wait until that region's replica has replayed past the LSN
  //   at which the comment became visible
  await waitForReplay(targetRegion, event.commit_lsn, { timeoutMs: 30_000 });
  await sendNotification(event);
}
// ⇒ ★ THE NOTIFICATION CAN NEVER ARRIVE BEFORE THE DATA IT
//   REFERS TO. Causality preserved across the service boundary —
//   which is exactly where it was breaking.
```

```js
// ★ ④ THE PARTITION POLICY, MADE EXPLICIT
async function readDoc(req, id) {
  const region = req.region;
  const r = replicaFor(req.sessionId, region);
  const lag = r?.lagMs ?? Infinity;

  if (lag > 120_000) {                        // ★ CP above 2 minutes
    metrics.increment('read.refused_stale', { region });
    throw new ServiceUnavailable('REGION_STALE');
  }
  if (!r || req.causalLsn && !lsnGte(r.replayLsn, req.causalLsn)) {
    return { pool: primary, ageMs: 0 };       // ★ fall back to primary
  }
  return { pool: r.pool, ageMs: lag };
}

app.get('/api/docs/:id', async (req, res) => {
  const { pool, ageMs } = await readDoc(req, req.params.id);
  const { rows } = await pool.query('SELECT * FROM documents WHERE id=$1',
                                    [req.params.id]);
  res.set('X-Data-Age', String(Math.round(ageMs / 1000)));   // ★ always
  if (ageMs > 30_000) res.set('X-Data-Stale', 'true');       // ★ UI banner
  res.json(rows[0]);
});
```

```sql
-- ★ ⑤ VIEW COUNTS AS A CRDT — the only genuinely AP data
CREATE TABLE doc_view_counts (
  doc_id  bigint   NOT NULL,
  region  smallint NOT NULL,
  n       bigint   NOT NULL DEFAULT 0,
  PRIMARY KEY (doc_id, region)
);
-- ★ each region writes only its own row ⇒ ★ conflict is impossible
-- ★ the merged value:
SELECT sum(n) FROM doc_view_counts WHERE doc_id = $1;
-- ⇒ ★ this is Topic 61's sharded counter, with the region as the
--   shard. The same structure solves local contention AND
--   distributed convergence.
```

**Step 6 — bound it, reconcile it, alert on it.**

```sql
-- ★ ① the bound is MEASURED and RECORDED, not assumed
INSERT INTO replica_lag_samples (sampled_at, application_name, lag_ms)
SELECT now(), application_name,
       extract(epoch from replay_lag)*1000
  FROM pg_stat_replication;
-- every 5 s

-- ★ ② the reconciler for what falls outside the bound
SELECT count(*) FROM documents
 WHERE version_conflict_detected_at IS NOT NULL
   AND resolved_at IS NULL;
-- ★ alert > 0; a job surfaces these to the user for manual merge

-- ★ ③ alerts
-- lag p99 > stated bound (5 s)                    ⇒ warn
-- lag > 120 s in any region                       ⇒ page (reads now CP)
-- read.refused_stale rate > 0                     ⇒ page
-- notifications deferred > 30 s                   ⇒ warn
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Read latency (Singapore) | 12 ms | 13 ms |
| ★ "it reverted" tickets | 142/wk | **0** |
| ★ "flickering" tickets | 88/wk | **0** |
| ★ "phantom notification" | 74/wk | **0** |
| Lost-update tickets | 36/wk | 4/wk *(surfaced as a merge UI)* |
| Reads served locally | 100% | **96.4%** |
| Stated staleness bound | ★ none | ★ **5 s, measured at p999** |
| Partition behaviour | ★ silent AP | ★ **AP < 120 s, CP above, always with `X-Data-Age`** |
| Consistency contract | ★ none | ★ **a signed table** |

```
 ★ SIX LESSONS:
 ① ★ FOUR TICKET CLUSTERS WERE FOUR DIFFERENT GUARANTEES, all
   filed as "replication lag". Naming them separated them.
 ② ★ NONE OF THE FIXES REQUIRED LINEARIZABILITY. Read-your-writes,
   monotonic reads and causal ordering cost 3.6% of reads.
 ③ ★ THE p50 LAG WAS 84 ms AND THE p999 WAS 4.9 SECONDS.
   A bound quoted from the average would have been wrong by 60×.
 ④ ★ THE SYSTEM HAD SILENTLY CHOSEN AP. Nobody decided that; the
   absence of a policy IS a policy, and it was the wrong one.
 ⑤ ★ THE NOTIFICATION BUG WAS CAUSALITY BREAKING ACROSS A SERVICE
   BOUNDARY — the place it always breaks, and the place an LSN
   token fixes it with one header.
 ⑥ ★ THE DELIVERABLE WAS A TABLE, SIGNED BY THE PRODUCT OWNER.
   "How stale is acceptable?" is a business question, and asking
   it produced answers spanning three orders of magnitude.
```

---

## Common mistakes

**1. "Pick two of three."**
- *Symptom:* someone claims a system is "CA".
- *Fix:* P is not optional. The choice is CP or AP *during* a partition.

**2. Confusing CAP's C with ACID's C.**
- *Symptom:* "PostgreSQL is CP because it's ACID."
- *Fix:* they are unrelated properties. CAP's C is linearizability; ACID's C is constraint enforcement.

**3. Classifying systems instead of operations.**
- *Symptom:* a single global consistency decision applied to payments and view counts alike.
- *Fix:* a per-operation table. The same database should be CP for money and AP for counters.

**4. Ignoring the "else" branch of PACELC.**
- *Symptom:* endless partition debates while the daily latency-vs-consistency trade goes unexamined.
- *Fix:* partitions are rare; the L-vs-C choice happens on every request.

**5. Quoting a staleness bound from the average.**
- *Symptom:* "we're at most 100 ms behind", with a p999 of 5 seconds and a max of 4 minutes.
- *Fix:* the bound is the p999, measured continuously.

**6. "Eventually consistent" with no bound, reconciler or alert.**
- *Symptom:* inconsistencies accumulate silently.
- *Fix:* all three, or it's inconsistency with extra steps (Topic 51).

**7. Expecting `synchronous_commit = on` to give visibility.**
- *Symptom:* durable on the replica, invisible there; read-your-writes still broken.
- *Fix:* `remote_apply` for visibility, or application-level routing.

**8. A single named synchronous standby.**
- *Symptom:* losing it blocks every commit — you chose CP without realising.
- *Fix:* `ANY 1 (a, b, c)`, and decide the CP-vs-AP question deliberately.

**9. Reaching for linearizability when a client-centric guarantee would do.**
- *Symptom:* a global quorum round trip on every read to fix "the save button doesn't work".
- *Fix:* read-your-writes costs a few percent of reads; linearizability costs 100%.

**10. Last-write-wins on data you care about.**
- *Symptom:* silent data loss, with clock skew choosing the winner.
- *Fix:* a CRDT, an application merge, or — best — a single writer per key.

**11. Serving stale data with no indication.**
- *Symptom:* users act on four-minute-old content and overwrite newer changes.
- *Fix:* an `X-Data-Age` header always, a UI banner past a threshold, and a hard CP cutoff.

**12. Not writing the contract down.**
- *Symptom:* four bug clusters investigated independently over months.
- *Fix:* a signed table of operation → model → bound → partition behaviour.

---

## Hands-on proof

**PROVE IT #1–#9 — Example 1** (a replica returning 0 for a completed write, two replicas making reads non-monotonic, `synchronous_commit = on` giving durability without visibility while `remote_apply` gives both, a single named sync standby hanging on `SyncRep`, read-your-writes measured at 3.6% primary fallback, session pinning restoring monotonicity, a simulated partition showing both CP and AP responses, why `W+R>N` is not linearizability, and a PN-counter converging where LWW loses data).

**PROVE IT #10 — measure your actual staleness bound.**
```sql
SELECT percentile_cont(0.50) WITHIN GROUP (ORDER BY lag_ms) AS p50,
       percentile_cont(0.999) WITHIN GROUP (ORDER BY lag_ms) AS p999,
       max(lag_ms)
  FROM replica_lag_samples WHERE sampled_at > now() - interval '30 days';
```
```
 p50  |  p999   |   max
------+---------+----------
 ★ 4.1| ★ 4,102 | ★ 240,000
   ★ 1,000× between p50 and p999. Quote the p999.
```

**PROVE IT #11 — causality breaking across services.**
```js
// service A writes a comment on the primary
const { rows:[c] } = await primary.query('INSERT INTO comments … RETURNING id');
await notifier.send(recipient, `New comment #${c.id}`);   // ★ immediate
// service B (the user's browser) reads from a regional replica
// ⇒ ★ the notification arrives before the comment is visible.
```
```js
// ★ with the causal token
await waitForReplay(regionOf(recipient), commitLsn, { timeoutMs: 30000 });
await notifier.send(recipient, `New comment #${c.id}`);
// ⇒ ★ never arrives before the data.
```

**PROVE IT #12 — CRDT convergence under reordering and duplication.**
```sql
-- apply the same merges in three different orders, with duplicates
-- ★ node1: +5, node2: +3, node1: +2   (duplicated)
INSERT INTO pn_counter VALUES ('k',1,5,0) ON CONFLICT (key,node_id)
  DO UPDATE SET increments = greatest(pn_counter.increments, EXCLUDED.increments);
-- ★ merge = MAX per field, so duplicates are IDEMPOTENT
SELECT sum(increments)-sum(decrements) FROM pn_counter WHERE key='k';
```
```
 ★ the same value regardless of order or duplication.
```

---

## The design decision framework

```
★★★ CAP CLASSIFIES OPERATIONS, NOT SYSTEMS.
    AND PACELC'S "ELSE" BRANCH IS THE ONE YOU LIVE WITH. ★★★

 ① ★ FOR EACH OPERATION, ASK FOUR QUESTIONS
    ① what breaks if this is 5 seconds stale?
    ② what breaks if this is REFUSED during a partition?
    ③ can two concurrent writes conflict?
    ④ ★ is there an INVARIANT ACROSS ITEMS?
       ⇒ ★ if yes, it needs coordination. It cannot be made AP
         without over-selling and compensating.

 ② ★ REACH FOR THE CHEAPEST GUARANTEE THAT FIXES THE BUG
    ★ read-your-writes  ⇒ "the save button doesn't work"
    ★ monotonic reads   ⇒ "it flickers / goes backwards"
    ★ causal            ⇒ "I was notified about something invisible"
    eventual + bound    ⇒ everything else
    linearizable        ⇒ ★ money, inventory, seats, limits
    ⇒ ★ THE FOUR CLIENT-CENTRIC GUARANTEES COST A FEW PERCENT OF
      READS. LINEARIZABILITY COSTS 100%.

 ③ ★ MEASURE YOUR BOUND — AND QUOTE THE p999
    p50 and p999 differ by 1,000× in real systems.
    ⇒ ★ a bound from the average is wrong by orders of magnitude.

 ④ ★ EVENTUAL CONSISTENCY NEEDS THREE THINGS
    ① a measured bound  ② ★ a reconciler  ③ ★ an alert
    ⇒ ★ without ② and ③ it is inconsistency with extra steps.

 ⑤ ★ DECIDE THE PARTITION POLICY IN ADVANCE — AND IN WRITING
    reads:  AP below a threshold, ★ CP above it
            ⇒ ★ ALWAYS expose the age (a header, a banner)
    writes: ★ CP for anything with an invariant; AP only where a
            compensation process already exists
    ⇒ ★ THE ABSENCE OF A POLICY IS A POLICY, AND IT IS USUALLY
      SILENT AP.

 ⑥ ★ AVOID CONFLICTS RATHER THAN RESOLVING THEM
    ★ one writer per key (sharding, Topic 60) ⇒ no merge needed
    ★ a CRDT when writes are genuinely concurrent and commutative
    ★ an application merge when the business has an opinion
    ✗ ★ LWW on anything you care about — clock skew picks the winner

 ⑦ ★ KNOW WHAT YOUR CONFIGURATION ALREADY CHOSE
    async replication            ⇒ ★ A over C
    synchronous_standby_names='n2' ⇒ ★ C over A, and n2 dying
                                   blocks ALL writes
    synchronous_commit=on        ⇒ durable, ★ NOT visible
    reading a replica            ⇒ ★ L over C
    ⇒ ★ most teams have made all four choices without naming them.

 ⑧ ★ THE DELIVERABLE IS A TABLE, SIGNED OFF
    operation | model | bound | partition behaviour | rationale
    ⇒ ★ "how stale is acceptable?" is a BUSINESS question, and the
      answers span three orders of magnitude.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
With a primary and two replicas: (a) prove a replica is not linearizable; (b) prove alternating reads between two replicas are not monotonic; (c) show `synchronous_commit = on` giving durability without visibility, and `remote_apply` giving both; (d) show a single named sync standby blocking all commits when it dies.

### Exercise 2 — medium (apply it)
Implement read-your-writes with LSN routing and monotonic reads with session pinning. Measure: the percentage of reads that fall back to the primary, and prove that alternating reads no longer go backwards.

Then measure your replication lag at p50, p99, p999 and max over a week, and write down the staleness bound you would publish.

### Exercise 3 — hard (production simulation)
A collaborative document platform expands to three regions. Read latency drops from 210 ms to 12 ms, and support tickets rise to 340/week in four distinct clusters. A four-minute network partition passes entirely unnoticed by users, who edit stale documents and overwrite newer changes.

(a) Classify each ticket cluster by the consistency guarantee it violates. Why were they all filed as "replication lag"?
(b) Measure lag per region at p50/p99/p999/max. What does the ratio between p50 and max tell you about quoting bounds?
(c) The system silently chose AP during the partition. Explain why "no policy" is itself a policy, and what it cost.
(d) Write the consistency contract table for eight operations, including partition behaviour.
(e) Implement read-your-writes and state its measured cost.
(f) Implement monotonic reads. Why is session pinning sufficient, and what does it cost?
(g) The notification bug is causality breaking across a service boundary. Explain the mechanism and implement the fix.
(h) Design the partition policy for reads: the AP threshold, the CP cutoff, and what the user sees at each stage.
(i) View counts are the only genuinely AP data. Design the CRDT and explain its relationship to Topic 61's sharded counter.
(j) Give the three things bounded eventual consistency requires, and the alerts for each.
(k) None of the fixes required linearizability. Explain why, in one paragraph.

---

## Mental model checkpoint

1. State CAP precisely. Why can't you "pick two"?
2. Name the four common misreadings of CAP.
3. What is PACELC, and why does its "else" branch matter more day to day?
4. Order the consistency ladder from strongest to weakest, with the cost of each.
5. Name the four client-centric guarantees and the user-visible bug each prevents.
6. What consistency does PostgreSQL give for primary reads, replica reads, and `remote_apply`?
7. Why does `synchronous_commit = on` not give read-your-writes on the replica?
8. State the quorum condition. Why is it not linearizability by itself?
9. Why is last-write-wins dangerous, and what decides the winner?
10. What three properties make a CRDT merge safe over an unreliable network?
11. What three things does bounded eventual consistency require?
12. Why should you quote the p999 rather than the p50 as your staleness bound?

---

## Quick reference card

**CAP:** you cannot pick P. **The choice is CP or AP *during* a partition.** There is no CA.
**PACELC:** if (P) → A or C; ★ **else → L or C** — the daily trade.

**The ladder** — linearizable → sequential → causal → **client-centric** → eventual.

| Guarantee | Prevents | Cost |
|---|---|---|
| ★ read-your-writes | "the save button doesn't work" | ★ ~3% of reads → primary |
| ★ monotonic reads | "it flickers / goes backwards" | session pinning |
| ★ causal | "notified about invisible data" | ★ one header |
| linearizable | all of it | ★ 100% → primary/quorum |

**PostgreSQL**
```
primary reads              ★ linearizable
async replica reads        ★ eventual, ★ not monotonic across replicas
synchronous_commit = on    ★ DURABLE there, ★ NOT VISIBLE there
  = remote_apply           ★ visible ⇒ linearizable on that replica
synchronous_standby_names = 'n2'   ★ = CP; losing n2 BLOCKS ALL COMMITS
  ⇒ use 'ANY 1 (n2, n3)'
```

**Quorum:** `W + R > N` ⇒ quorums overlap. ★ **Not linearizability by itself** — that needs consensus.

**Convergence:** ✗ LWW (★ clock skew picks the winner) · ★ CRDT (commutative/associative/idempotent) · app merge · ★ **best: one writer per key**.

**★ Bounded eventual consistency needs three things:** a **measured p999 bound** · ★ a **reconciler** · ★ an **alert**.

**★ The deliverable is a table:** operation | model | bound | partition behaviour | rationale — **signed by the product owner**.

---

## When would I use this at work?

1. **The first time anyone says "we're eventually consistent."** That sentence is not a design. Ask which operations, what bound, measured at which percentile, and what happens during a partition. The four questions usually reveal that nobody has decided.

2. **Any multi-region expansion.** The latency win is real and the consistency bugs are guaranteed. Naming the four client-centric guarantees up front converts months of scattered tickets into an afternoon of routing work.

3. **When someone proposes strong consistency everywhere.** Linearizability costs a round trip on every operation. Read-your-writes and monotonic reads cost a few percent and fix the bugs users actually report — which is almost always what was wanted.

4. **Reviewing a `synchronous_standby_names` line.** A single name means you have chosen CP: if that standby dies, every write blocks. That is a legitimate choice for some systems and a surprise for most, and it is worth making deliberately.

---

## Connected topics

**Understand before this:** 58 (replica lag and routing), 63 (RPO/RTO — CAP made operational), 51 (2PC and the outbox — CP vs AP in practice), 52 (idempotency — what makes AP retries safe), 44/50 (isolation — the single-node analogue).

**This unlocks:**
- **69** — security at the data layer
- **70–76** — Phase 8: non-relational models, most of which sit at a different point on this ladder
- **77–79** — the capstones, where the contract table is part of the deliverable
