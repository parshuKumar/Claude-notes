# 09 — CAP Theorem — The Impossible Triangle
## Category: HLD Fundamentals

---

## What is this?

The CAP theorem says that when your data lives on **more than one machine**, and the network between those machines breaks, you must choose: either keep answering requests with possibly-stale data, or stop answering and protect correctness. **You cannot do both.**

Think of two bank tellers in two different branches sharing one ledger over a phone line. The phone line dies. A customer walks up to each teller at the same time and both want to withdraw the last $100. Each teller has exactly two options: say "sorry, I can't reach head office, come back later" (safe but unhelpful), or hand over the money and hope the other teller didn't (helpful but now the account is overdrawn). That's CAP. There is no third door.

---

## Why does it matter?

Because **every distributed database you will ever pick has already made this choice for you**, and if you don't know which choice it made, you will be surprised in production at 3 AM.

- You put your inventory count in Cassandra and oversell 400 concert tickets, because Cassandra chose availability and two replicas both said "1 ticket left."
- You put your session store in a strongly-consistent cluster, one node loses network, and suddenly **nobody can log in** — because the cluster chose consistency and refused to serve.

**In interviews:** CAP is the single most name-dropped and most *misunderstood* theorem in system design. Interviewers use it as a filter. Saying "I'll pick AP because availability matters" without saying *during a partition* marks you as someone who memorized a triangle picture. Saying "P isn't optional, so the real question is what we do in the seconds a partition is active, and honestly PACELC is the better model here" marks you as someone who has operated a distributed system.

**At work:** CAP is the vocabulary you use in design review when someone says "let's just use MongoDB with `w: 1` for the payments ledger." You need to be able to explain, precisely, what they just gave up.

---

## The core idea — explained simply

### The Two Warehouses Analogy

You run an online shop. You have **two warehouses** — one in Mumbai, one in Delhi. Both warehouses keep a copy of the same inventory sheet, so that customers in either city get fast answers. Every time one warehouse sells an item, it phones the other to say "decrement the count."

Life is good. Then a cable is cut. **Mumbai and Delhi can no longer phone each other.** Neither warehouse has crashed — both are healthy, both have staff, both can still talk to their own local customers. They just can't talk to *each other*.

A customer walks into Mumbai and asks to buy the last laptop. Mumbai's manager has two choices:

**Choice A — "Refuse until I can confirm" (Consistency)**
> "I'm sorry, I can't reach Delhi right now. I can't guarantee this laptop is still available. Please wait."

The customer gets **no answer** (or an error). But you will *never* sell the same laptop twice. Mumbai has chosen **CP**: it sacrificed availability to protect correctness.

**Choice B — "Sell it and sort it out later" (Availability)**
> "Sure! Here's your laptop."

The customer is happy. But Delhi might have sold that same laptop 3 seconds ago. You now have a conflict to resolve when the cable is repaired. Mumbai has chosen **AP**: it sacrificed consistency to stay useful.

**There is no Choice C.** You cannot both answer immediately *and* guarantee the answer is globally correct, when you physically cannot reach the other warehouse. That's not an engineering limitation — it's information-theoretically impossible. Information can't travel over a cut cable.

### Mapping the analogy back

| Analogy | Technical concept |
|---|---|
| The two warehouses | Two nodes / replicas of a distributed database |
| The shared inventory sheet | Replicated data |
| The phone line between them | The network link |
| The cable being cut | A **network partition** (P) |
| "I can't answer until I confirm" | **Consistency** (C) — refuse rather than be wrong |
| "Sure, here you go" | **Availability** (A) — always answer, even if stale |
| Selling the same laptop twice | A consistency violation (a *divergent* read) |
| Reconciling after the cable is fixed | Conflict resolution / anti-entropy / read-repair |

---

## Key concepts inside this topic

### 1. Defining C, A, and P precisely (this is where people fail)

The three letters have **specific formal meanings** in CAP that differ from their everyday usage. Get these exactly right.

**C — Consistency = Linearizability**

Not ACID's "C." Not "the data is nice and valid." In CAP, **Consistency means linearizability**: the whole distributed system behaves as if there is exactly **one single copy** of the data, and every operation happens **instantaneously at some point** between when you called it and when it returned.

Concretely: if a write completes at 10:00:00.000, then **any** read that starts at 10:00:00.001 — from *any* node — must see that write or something newer. Never something older.

> Recall from [08 — Consistency Models] that linearizability is the *strongest* consistency model — the top of the spectrum. CAP's C is that top rung, not the whole ladder.

**A — Availability = every non-failing node returns a non-error response**

Not "99.99% uptime." Not "the site loads." In CAP, **Availability means: every request that reaches a node that has not crashed must receive a non-error response, in finite time.**

The crucial word is **every**. If your system has 5 nodes, 3 of them are on the majority side of a partition and answer fine, and 2 are on the minority side and return `ERR: no quorum` — **that system is NOT available in the CAP sense**, even though 60% of your nodes are happily serving traffic. CAP's A is a brutally strict, all-or-nothing definition.

This is why CAP's "A" and your SLA's "availability" are different things. A CP database can absolutely have 99.99% real-world uptime — because partitions are rare.

**P — Partition tolerance = the system keeps operating despite dropped messages**

The system continues to function even when the network arbitrarily **drops or delays messages** between nodes. A partition is not "a node crashed" — a crashed node is just a slow node that never replies. A partition is when **healthy nodes cannot reach each other** but each can still reach some clients.

### 2. Why P is not a choice — and what that really means

Here is the sentence that separates people who understand CAP from people who've seen the triangle:

> **You do not get to choose P. The network chooses P for you.**

Networks fail. Switches reboot. A cloud provider's availability zone gets isolated. A `iptables` rule is fat-fingered. A GC pause makes a node unreachable for 8 seconds, which is indistinguishable from a partition. Amazon, Google, and Microsoft all publish postmortems about partitions inside their own datacenters.

So if your data is on more than one machine, **P will happen to you.** You cannot opt out. Which means:

```
CAP is NOT: "pick any 2 of 3"
CAP IS:     "P is forced on you, so during a partition you pick C or A"
```

And critically — **the choice only applies while a partition is active.** When the network is healthy, a well-built system gives you *both* C and A. A CP database like etcd is perfectly available on a normal Tuesday. An AP database like Cassandra is perfectly consistent on a normal Tuesday (if you give it a moment). The trade-off is a *failure-mode* trade-off, not a steady-state one.

### 3. Walking a two-node partition, step by step

Two replicas, N1 and N2. They replicate to each other. Value `v = "A"` on both. Client X talks to N1, client Y talks to N2.

```
t0   N1 [v=A]  <────── replication ──────>  N2 [v=A]
                        (healthy)

t1   THE NETWORK PARTITIONS.  N1 and N2 cannot reach each other.
     Both are alive. Both can still be reached by their clients.

     N1 [v=A]  <──────  ✂  CUT  ✂  ──────>  N2 [v=A]

t2   Client X → N1:  WRITE v = "B"
t3   Client Y → N2:  READ  v = ?
```

Now N1 must decide, at t2, what to do with that write, and N2 must decide, at t3, what to do with that read.

**If the system is CP (chooses Consistency):**

```
t2   N1: "I cannot replicate this write to N2. If I accept it, N2 will
          serve a stale 'A' and linearizability is broken."
     N1 → Client X:  ERROR / timeout / "no quorum"        ← NOT AVAILABLE

t3   N2: "I don't know if N1 has newer data. If I answer 'A' I might be lying."
     N2 → Client Y:  ERROR / timeout                      ← NOT AVAILABLE

     Correctness preserved. Nobody ever sees a wrong value.
     But the system just told two healthy clients "no."
```

**If the system is AP (chooses Availability):**

```
t2   N1: "I'll take the write locally and replicate it when the network heals."
     N1 → Client X:  200 OK, v = "B"                      ← AVAILABLE
     N1 now holds v="B", N2 still holds v="A".  They have DIVERGED.

t3   N2: "I'll serve what I have."
     N2 → Client Y:  200 OK, v = "A"                      ← STALE READ

     Everyone gets an answer. But Y read "A" AFTER X's write of "B"
     succeeded. Linearizability is broken.

t4   NETWORK HEALS.  Now: conflict resolution.
     - Last-write-wins by timestamp?  (may silently lose data)
     - Vector clocks → hand both versions to the app to merge?
     - CRDTs → merge automatically by construction?
     This cleanup work is the real price of AP.
```

Note what the AP system did **not** do: it did not corrupt anything or crash. It made a defensible engineering choice and **moved the cost from "downtime" to "reconciliation complexity."** That's the whole theorem in one line.

### 4. CP systems, AP systems, and the CA myth

**CP — sacrifice availability during a partition**

| System | How it behaves during a partition |
|---|---|
| **ZooKeeper** | Uses Zab consensus. Minority-side nodes stop serving writes and can be configured to stop serving reads. No quorum, no service. |
| **etcd** | Raft consensus. A node that loses contact with the leader's quorum refuses writes. This is exactly why Kubernetes control planes freeze during a partition rather than split-brain. |
| **HBase** | Each region is served by exactly one RegionServer. If that server is unreachable, that region's data is simply **unavailable** until reassignment. Consistency by construction. |
| **MongoDB (default)** | A replica set has one primary. On partition, the minority side steps down its primary and accepts **no writes**. With `readConcern: "majority"` / `writeConcern: "majority"`, it is meaningfully CP. |
| **Spanner / CockroachDB** | Raft/Paxos per range. Minority ranges stall. They aim for very high availability by making partitions rare and quorums fast — but under a true partition, they choose C. |

**AP — sacrifice consistency during a partition**

| System | How it behaves during a partition |
|---|---|
| **Cassandra** | Leaderless. Any replica can accept a write. During a partition, both sides keep accepting writes; hinted handoff + read repair + anti-entropy reconcile later. Conflicts resolved by last-write-wins on cell timestamps. |
| **DynamoDB / the original Dynamo paper** | Same lineage. Quorums are *tunable* (`W`, `R`, `N`) — you can dial toward CP by requiring `W + R > N`, or toward AP with `W=1, R=1`. (Today's managed DynamoDB defaults to strong-consistency-capable reads; the *design lineage* is the AP one.) |
| **CouchDB** | Built for offline-first replication. Every node accepts writes; conflicting document revisions are **kept side by side** and the application picks a winner. |
| **Riak** | Vector clocks and sibling values — hands you both versions and says "you decide." |

**CA — the box that doesn't exist (in a distributed system)**

The "CA" corner of the triangle means: consistent and available, but *not* partition tolerant. The only way to be genuinely CA is to **not be distributed at all**:

- A single PostgreSQL box with no replicas. Zero network between nodes → zero partitions possible → C and A trivially. But zero fault tolerance too: that box dies, everything dies.
- A single-node Redis. Same story.

Any system that markets itself as "CA and distributed" is really a CP system that hasn't thought hard about partitions, or an AP system that hasn't admitted it. **The moment you add a second node, CA is off the table.**

### 5. What interviewers actually probe (the misunderstandings)

**Misunderstanding 1: "You pick 2 of 3."**
No. You get P for free (unwillingly). You pick between C and A **during a partition only**.

**Misunderstanding 2: "MongoDB is CP, so it's less available than Cassandra."**
Not in the everyday sense. MongoDB replica sets run at very high uptime. CAP's "A" only describes the *partition* case. Confusing CAP-availability with SLA-availability is the #1 tell of a memorized answer.

**Misunderstanding 3: "CAP's C is the C in ACID."**
No. ACID's C means "transactions preserve database invariants (constraints, FKs)." CAP's C means linearizability — a *distributed* property about what reads see. A single-node MySQL is ACID-consistent and trivially linearizable; that says nothing about CAP.

**Misunderstanding 4: "My system is AP, so it's eventually consistent, so it's fine."**
Eventual consistency is a *guarantee about the end state*, not about how bad the middle state gets. "Eventually" can mean 200 ms or, with a stuck hinted-handoff queue, an hour. Ask what the convergence bound is.

**Misunderstanding 5: "We're CP so we never lose data."**
CP protects *linearizability*, not durability. A CP system with `fsync` disabled will still lose acknowledged writes on power failure. Different axis entirely.

**Misunderstanding 6: "The database is CP so my app is consistent."**
Your app probably reads from a **cache** in front of the database. Recall from [08 — Consistency Models] that a cache is a replica. Putting Redis in front of etcd gives you an AP read path with a CP write path. CAP applies to the *system as the user experiences it*, not to one box in your diagram.

### 6. PACELC — the honest upgrade

CAP has a real flaw: **it only says anything about the partition case**, which is maybe 0.001% of your system's life. It is silent about the other 99.999%. That's a strange thing for a model to be silent about.

**PACELC** (Daniel Abadi, 2012) fixes this:

> **If there is a Partition (P), choose Availability (A) or Consistency (C).**
> **Else (E) — when the network is fine — choose Latency (L) or Consistency (C).**

The "else" half is the part you actually feel every single day. Even with a perfectly healthy network, **making a write visible on 3 replicas across 3 availability zones costs a real network round trip** — a few milliseconds inside a region, 100+ ms across continents. You can pay that latency for consistency, or skip it and be fast but stale.

```
                    ┌─────────────────────────────┐
                    │  Is the network partitioned? │
                    └──────────┬────────┬─────────┘
                          YES  │        │  NO ("Else")
                    ┌──────────▼──┐  ┌──▼─────────────┐
                    │  PA  or  PC │  │  EL   or   EC  │
                    │             │  │                │
                    │ answer  vs  │  │ answer  vs     │
                    │ stale   be  │  │ fast    wait   │
                    │         safe│  │         for    │
                    │             │  │         quorum │
                    └─────────────┘  └────────────────┘
```

Classifying real systems under PACELC is far more informative than CAP:

| System | PACELC | Read it as |
|---|---|---|
| **Cassandra** (default tuning) | **PA/EL** | Stays up during a partition; even when healthy, prefers low latency over strong consistency. |
| **DynamoDB** (eventually-consistent reads) | **PA/EL** | Same shape. Strongly-consistent reads flip you toward EC (and cost 2× the read capacity — *the latency/consistency price made literally into a bill*). |
| **MongoDB** (majority concerns) | **PC/EC** | Refuses on the minority side; waits for majority ack even when healthy. |
| **PostgreSQL with sync replication** | **PC/EC** | Commit blocks until the standby acks. |
| **PostgreSQL with async replicas** | **PC/EL** | Primary is the source of truth (CP-ish), but replicas serve stale reads for speed. |
| **Spanner** | **PC/EC** | Pays TrueTime commit-wait latency to buy external consistency. |

**When an interviewer says "CAP," the strongest possible move is:** answer CAP correctly, then say *"but CAP only describes the partition case — PACELC is the model I'd actually design with, because the latency-vs-consistency trade during normal operation is the one that shows up in my p99 every day."*

---

## Visual / Diagram description

### Diagram 1: The triangle — and why one corner is a lie

```
                          C
                    (Linearizability)
                         ╱   ╲
                        ╱     ╲
                       ╱       ╲
                      ╱   CP    ╲
                     ╱  ZooKeeper╲
                    ╱   etcd      ╲
                   ╱    HBase      ╲
                  ╱  MongoDB(maj)   ╲
                 ╱                   ╲
                ╱   ┌─────────────┐   ╲
               ╱    │  "CA"       │    ╲
              ╱     │  ─────────  │     ╲
             ╱      │ Single-node │      ╲
            ╱       │ Postgres.   │       ╲
           ╱        │ NOT a       │        ╲
          ╱         │ distributed │         ╲
         ╱          │ system.     │          ╲
        ╱           └─────────────┘           ╲
       ╱                   AP                  ╲
      ╱                Cassandra                ╲
     ╱                 DynamoDB                  ╲
    ╱                  CouchDB                    ╲
   ╱                   Riak                        ╲
  ╱───────────────────────────────────────────────── ╲
 A                                                     P
(Every non-failing                          (Survives dropped
 node responds)                              messages between nodes)
```

The triangle is drawn everywhere, and it's *misleading*: it implies three symmetric options. In reality the bottom-right vertex **P is mandatory** for any multi-node system, so you are always somewhere on the right-hand edge — the CP↔AP edge. The "CA" corner is only reachable by giving up distribution entirely.

### Diagram 2: The partition decision, as a live system

```
        NORMAL OPERATION                  ┃        PARTITION ACTIVE
                                          ┃
   Client A          Client B             ┃   Client A          Client B
      │                 │                 ┃      │                 │
      │ write v=B       │ read v          ┃      │ write v=B       │ read v
      ▼                 ▼                 ┃      ▼                 ▼
  ┌───────┐         ┌───────┐             ┃  ┌───────┐  ✂ ✂ ✂  ┌───────┐
  │ Node1 │◀───────▶│ Node2 │             ┃  │ Node1 │  ✂cut✂  │ Node2 │
  │ v = A │  repl   │ v = A │             ┃  │ v = A │  ✂ ✂ ✂  │ v = A │
  └───────┘         └───────┘             ┃  └───┬───┘         └───┬───┘
                                          ┃      │                 │
  Both C and A hold.                      ┃      │  MUST CHOOSE    │
  No trade-off needed!                    ┃      ▼                 ▼
                                          ┃   ┌──────────────────────────┐
  (But note: how long did the             ┃   │ CP: both return ERROR.   │
   write take to become visible           ┃   │     v stays A everywhere.│
   on Node2? THAT is PACELC's             ┃   ├──────────────────────────┤
   "Else" half.)                          ┃   │ AP: N1 returns 200 (v=B) │
                                          ┃   │     N2 returns 200 (v=A) │
                                          ┃   │     → DIVERGED, fix later│
                                          ┃   └──────────────────────────┘
```

Draw this on a whiteboard as two panels. The left panel is the world 99.999% of the time — no trade-off exists there, which is precisely why CAP alone is not enough. The right panel is the world CAP describes, and the fork in the road at the bottom is the entire theorem.

---

## Real world examples

### 1. Amazon's shopping cart — deliberately AP

The 2007 Dynamo paper describes an explicit business decision: **an "add to cart" must never fail.** A rejected add-to-cart is lost revenue, immediately and measurably.

So Dynamo (and the DynamoDB lineage that followed) accepts writes on *any* reachable replica during a partition. Two replicas can independently accept adds. On heal, Dynamo does **not** silently pick a winner — it hands the application **both versions** (tracked with vector clocks), and the cart service **merges them by taking the union of items**. The famous consequence: a *deleted* item can reappear in your cart, because a union of {A, B} and {A} is {A, B}. Amazon judged a resurrected item to be a far cheaper bug than a failed add.

That is CAP reasoning driven by a P&L, not by a textbook.

### 2. Kubernetes' etcd — deliberately CP

The Kubernetes control plane stores all cluster state in **etcd**, which uses Raft. If the etcd cluster loses quorum (say, 2 of 3 members are isolated), the minority side **refuses writes**.

Consequence: during a partition, you **cannot schedule new pods or change cluster state.** That looks like an outage. It is a *deliberate* one. The alternative — two halves of the cluster each believing they're in charge — would mean **split-brain**: the same Deployment scaled by two controllers, duplicate pods, two Services claiming the same IP. Kubernetes' designers correctly decided that a frozen control plane is vastly better than a schizophrenic one.

Notice the escape hatch: **already-running pods keep running.** The *data plane* stays up while the *control plane* goes CP. That's a common and excellent architectural move — put CP only where you truly need it.

### 3. Cassandra at scale — tunable, but AP at heart

Cassandra lets you set consistency **per query** via `W` (replicas that must ack a write) and `R` (replicas that must respond to a read), against `N` replicas:

- `W=1, R=1` → maximum availability and lowest latency, weakest guarantee. Classic AP/EL.
- `W=QUORUM, R=QUORUM` with `W + R > N` → any read set overlaps any write set, so you'll see the latest write. This is a **strong-ish** setting — but note that during a partition, the minority side now **fails** those quorum queries. You have dialed yourself toward CP for those queries.

This is the practical lesson: **CP vs AP is often a per-query knob, not a per-database identity.** Write the session cache at `ONE`; write the billing ledger at `QUORUM`. Real systems mix.

---

## Trade-offs

| | **CP (consistency during partition)** | **AP (availability during partition)** |
|---|---|---|
| **During a partition** | Minority side returns errors / times out | Every reachable node answers |
| **What you can promise users** | "The answer you get is always correct" | "You always get an answer" |
| **Failure mode** | Downtime for some users | Stale or conflicting data |
| **Complexity lives in** | Consensus (Raft/Paxos), leader election, quorum math | Conflict resolution, vector clocks, CRDTs, read repair |
| **Write latency (healthy net)** | Higher — must reach a quorum | Lower — one replica can ack |
| **Good for** | Money, inventory counts, locks, config, leader state, unique constraints | Carts, likes, view counts, feeds, sensor data, logs, presence |
| **Bad for** | High-throughput ingest, cross-region writes, offline clients | Anything where two conflicting truths cause real-world harm |

| Approach | Benefit | Cost you actually pay |
|---|---|---|
| Go CP everywhere | Simple mental model, no reconciliation code | Cross-region writes get slow; partitions become user-visible outages |
| Go AP everywhere | Always up, fast writes, easy multi-region | You now own the conflict-resolution problem *forever*, in application code |
| Mix per data type | Each dataset gets what it needs | Two mental models in one codebase; engineers must know which is which |
| Mix per query (tunable) | Maximum flexibility | Very easy to get wrong silently — one `W=1` on the wrong table and your ledger drifts |

**Rule of thumb:** Ask one question about the data — **"if two users see different answers for three seconds, does anyone get hurt?"** If the answer involves money, inventory, identity, or locks → **CP**. If the answer is "the like count is briefly off by one" → **AP**. And **never apply one answer to the whole system** — a real product almost always needs both, on different tables.

---

## Common interview questions on this topic

### Q1: "Explain CAP. Which two would you pick?"
**Hint:** Gently reject the premise. "P isn't something you pick — the network imposes it. Any system with more than one node must tolerate partitions. So the real choice is C or A, *and only while a partition is active*. When the network is healthy, a good system gives you both." Then name the trade for your specific system. This reframing is the answer they're listening for.

### Q2: "Is MongoDB CP or AP?"
**Hint:** "With the default replica-set setup and `writeConcern: majority` / `readConcern: majority`, it's CP — the minority side steps down its primary and refuses writes rather than risk split-brain. But it's tunable: with `w:1` and reads allowed from secondaries, you get stale reads and it starts behaving AP-ish. The honest answer is that CP/AP is a *configuration*, not a logo on the box." Bonus: mention its PACELC class is PC/EC.

### Q3: "Why is CA impossible for a distributed system?"
**Hint:** Because P is an environmental fact, not a feature. Once a partition happens, a node that cannot reach its peers has only two options — refuse (lose A) or answer from local state (lose C). CA is only achievable by never having a network between nodes, i.e. by being a single machine. And a single machine has zero fault tolerance, which is usually *why* you went distributed.

### Q4: "How does CAP relate to ACID and BASE?"
**Hint:** Different axes, colliding vocabulary. ACID's **C** = "the DB's invariants (constraints, FKs) hold after a transaction" — a *single-node* property. CAP's **C** = linearizability — a *distributed* property about what reads observe. BASE (Basically Available, Soft state, Eventually consistent) is essentially the design philosophy of an AP system. Say this out loud; interviewers specifically probe for the ACID-C / CAP-C conflation.

### Q5: "What's PACELC and why is it better?"
**Hint:** CAP only says anything during a partition, which is a vanishingly small fraction of your system's uptime — it's silent about the other 99.999%. PACELC adds the "Else" clause: when there's no partition, you *still* trade **Latency vs Consistency**, because replicating a write to a quorum costs a real round trip. Cassandra is PA/EL; Spanner and MongoDB(majority) are PC/EC. The EL/EC choice is what shows up in your p99 latency graph every day.

### Q6: "Design a system that needs both strong consistency and high availability."
**Hint:** You can't have both *under partition* — but you can (a) **shrink the blast radius**: make partitions rare (multi-path networking, same-AZ quorums) and short; (b) **split by data type**: CP store for the ledger, AP store for the feed; (c) **degrade gracefully**: on partition, go read-only rather than fully down; (d) **use a fast consensus with an odd node count across 3 AZs** so a single-AZ partition still leaves a majority alive and serving. That last one is what "highly available CP" actually means in practice.

---

## Practice exercise

### The Ticket Booking Partition Drill

You're designing a concert-ticket booking system (a mini Ticketmaster). It has these five data stores:

1. **Seat inventory** — which of the 20,000 seats for a show are taken
2. **User session / login state**
3. **The "how many people are viewing this event right now" counter**
4. **Payment ledger** — money charged, refunds issued
5. **Event metadata** — venue name, date, artist, image URLs

**Do this (do not just read it — write it down):**

**Part A.** For each of the five stores, write one sentence answering: *"If two users see different answers for 3 seconds, who gets hurt and how badly?"* Then label it **CP** or **AP**. Justify each in one line.

**Part B.** For the two you labelled CP, name a concrete database (from the CP list in this doc) and say what happens to a user on the minority side of a partition. Write the literal error message you'd show them.

**Part C.** For the AP ones, describe how you'd reconcile conflicting values after the network heals. Be specific: last-write-wins? A counter CRDT? Union of sets?

**Part D.** A ticket goes on sale and 50,000 people hit "buy" in 10 seconds. Explain in 3-4 sentences why a *pure* CP seat-inventory store becomes a bottleneck, and describe one mitigation (hint: think about *partitioning the inventory itself* — do all 20,000 seats need to be in one consensus group?).

**Deliverable:** one page. If you can defend Part D to a skeptical colleague, you understand CAP better than most candidates.

---

## Quick reference cheat sheet

- **CAP** = you cannot have Consistency + Availability + Partition tolerance simultaneously **during a partition**.
- **C in CAP = linearizability** — the system acts as if there's one single copy of the data. **NOT** ACID's C.
- **A in CAP = every non-failing node returns a non-error response.** Brutally strict. NOT your 99.99% SLA.
- **P in CAP = the system survives dropped/delayed messages between nodes.**
- **P is NOT optional.** The network decides. If you have 2+ nodes, you have P.
- **So CAP is really: CP or AP, and only *during* a partition.** Healthy network = you get both.
- **CA = single node only.** Any "distributed CA" claim is marketing.
- **CP examples:** ZooKeeper, etcd, HBase, MongoDB (majority concerns), Spanner, CockroachDB.
- **AP examples:** Cassandra, Dynamo/DynamoDB lineage, CouchDB, Riak.
- **Tunable is normal:** Cassandra's `W + R > N` dials you toward C; `W=1,R=1` dials you toward A.
- **PACELC is the honest model:** if **P**, choose **A** or **C**; **E**lse, choose **L**atency or **C**onsistency.
- **PACELC classes:** Cassandra/DynamoDB = **PA/EL**. MongoDB(majority)/Spanner = **PC/EC**. Postgres w/ async replicas = **PC/EL**.
- **The deciding question:** "If two users see different answers for 3 seconds, does anyone get hurt?" Money/inventory/locks → CP. Likes/feeds/counters → AP.
- **Real systems mix.** One product uses CP for the ledger and AP for the feed. Never answer "my system is AP" about a whole product.
- **Caches are replicas.** Redis in front of a CP database gives your users an AP read path. CAP applies to what the *user* experiences.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [08 — Consistency Models](./08-consistency-models.md) — the full spectrum of consistency; CAP's "C" is the strongest rung on that ladder |
| **Next** | [10 — Networking Fundamentals](./10-networking-fundamentals.md) — the network that partitions; you can't reason about P without knowing how TCP actually fails |
| **Related** | [76 — Eventual Consistency in Practice](./76-eventual-consistency.md) — how AP systems actually reconcile: vector clocks, LWW, CRDTs |
| **Related** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — quorums (W+R>N), gossip, Merkle trees — the machinery behind the CP/AP dial |
| **Related** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — why CAP's "A" and your SLA's "availability" are different numbers |
