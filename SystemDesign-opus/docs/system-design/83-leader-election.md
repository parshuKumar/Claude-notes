# 83 — Leader Election in Distributed Systems
## Category: HLD Components

---

## What is this?

**Leader election** is how a group of identical servers agree that exactly ONE of them is in charge — and how they automatically pick a new one when that server dies.

Think of a group project with 6 students. If all 6 edit the shared document at once, you get chaos: duplicate paragraphs, contradictory sections, someone deleting someone else's work. So the group picks one person to hold the pen. Everyone else sends suggestions to that person. If the pen-holder gets sick and stops responding, the rest of the group must *notice*, *agree on a replacement*, and *make sure the old pen-holder can't keep writing if they come back*. That last part — preventing two people from thinking they hold the pen — is where distributed systems get genuinely hard.

---

## Why does it matter?

Some jobs must be done by **exactly one** node. Not "usually one." Exactly one.

- **A cron job.** You deploy 10 replicas of your Node service for availability. Each one boots a `node-cron` schedule that emails every customer at 9am. Congratulations: every customer got 10 emails. This is the single most common leader-election bug in real companies.
- **Accepting writes.** If two nodes both accept writes to the same record with no ordering between them, you get lost updates and conflicting state. A single writer gives you a total order for free.
- **Assigning work.** Someone must decide "partition 7 belongs to worker C." If two coordinators disagree, two workers process the same partition.
- **Being the source of truth.** Config, cluster membership, shard maps — one authoritative writer, many readers.

A leader takes a hard coordination problem (N nodes must agree on every decision) and reduces it to an easy one (one node decides, everyone follows). That's the whole trick.

**The cost you pay:** the leader is a bottleneck (all writes funnel through it) and a single point of failure. So the *only* thing that makes leadership viable is **automatic re-election** — the cluster must detect a dead leader and pick a new one in seconds, without a human.

**Interview angle:** "How do you make sure only one node runs this job?" and "What happens during a network partition?" are top-tier follow-ups on almost every HLD question. If you say "we'll have a master node" and can't explain failover and split-brain, you've dug a hole.

**Real-work angle:** you'll hit this the first time you scale a scheduled job past one instance. And you'll almost certainly solve it with a database lease, not by implementing Raft.

---

## The core idea — explained simply

### The Night Shift Manager Analogy

A 24-hour warehouse always has exactly one **shift manager**. The manager is the only person who can assign trucks to loading bays. Everyone else — the workers — just does what the manager says.

The manager wears a badge. The badge is the authority; without it, your orders are ignored.

Now, four things can go wrong, and each maps exactly onto a distributed systems problem:

**1. The manager goes home without telling anyone.** Trucks pile up. Nobody assigns bays. The warehouse stalls.
→ *A leader crashed and nobody noticed. You need **failure detection** (heartbeats).*

**2. Everyone simultaneously decides "I'll be manager!"** Six people grab six badges, all shouting contradictory bay assignments.
→ *A leader crashed and everyone elected themselves. You need an **election protocol**, not a free-for-all.*

**3. A wall collapses, splitting the warehouse in two.** Each half can't see the other. Each half thinks the other half is gone, so each elects its own manager. Two managers. Two contradictory bay assignments for the same truck.
→ ***Split-brain.** The killer. You need a **quorum rule**.*

**4. The manager falls asleep in the break room for 40 seconds.** The rest of the warehouse gives up, elects a new manager. Then the old manager wakes up, doesn't know time passed, and walks out still wearing the badge, shouting orders.
→ *A **GC pause** or VM freeze. The old leader isn't malicious — it's just stale. You need **fencing tokens**.*

Every serious leader-election system is an answer to those four failures.

| Warehouse | Distributed system |
|-----------|--------------------|
| Shift manager | Leader / primary / master |
| Workers | Followers / replicas |
| The badge | Leadership grant (lease, or a term in Raft) |
| "I'm still here" announcement every 30s | Heartbeat |
| Manager didn't announce for 2 minutes | Election timeout fired |
| Wall collapse splits the floor | Network partition |
| Two managers issuing orders | Split-brain |
| Majority of workers must approve the new manager | Quorum (N/2 + 1) |
| Badge has a serial number, and the loading bay only accepts orders with a *higher* serial than the last one it saw | Fencing token |

---

## Key concepts inside this topic

### 1. The naive approaches — and exactly how each one fails

**Approach A: Hardcode the leader.**

```javascript
// config.js
export const IS_LEADER = process.env.NODE_ROLE === 'leader';

if (IS_LEADER) startCronJobs();
```

This works perfectly until `node-1` dies. Then it works perfectly *never*, because no one runs the cron job at all, and you find out on Monday when the invoices didn't go out. **No failover = not a solution.** It's fine for a lab, not for production.

**Approach B: Let every node decide for itself.**

"I'll check if the leader responded to my ping. If not, I'm the leader."

Every node runs this logic *at the same time*, sees the same silence, and reaches the same conclusion: *I'm the leader.* Now you have N leaders. This is a free-for-all, not an election. **This is how you get split-brain by construction.**

**Approach C: Use a database row as a lease. (This is the honest answer for most teams.)**

Your database already provides something extremely hard to build: a single, linearizable, consistent decision point. Use it.

```sql
CREATE TABLE leader_lease (
  resource   TEXT PRIMARY KEY,          -- e.g. 'billing-cron'
  owner_id   TEXT        NOT NULL,      -- which node holds it
  fence      BIGINT      NOT NULL,      -- monotonic token, explained in §6
  expires_at TIMESTAMPTZ NOT NULL       -- lease deadline
);
```

Acquire it with an atomic upsert that only succeeds if the row is free or expired:

```javascript
// Returns the fencing token if we won leadership, or null if someone else holds it.
// The WHERE clause is the entire safety argument: Postgres serializes this row,
// so of 10 nodes racing, exactly one UPDATE sees an expired lease and wins.
async function tryAcquire(db, resource, nodeId, leaseMs) {
  const { rows } = await db.query(
    `INSERT INTO leader_lease (resource, owner_id, fence, expires_at)
     VALUES ($1, $2, 1, now() + ($3 || ' milliseconds')::interval)
     ON CONFLICT (resource) DO UPDATE
       SET owner_id   = EXCLUDED.owner_id,
           -- bump the fence ONLY on a genuine handover, so tokens strictly increase
           fence      = leader_lease.fence + 1,
           expires_at = EXCLUDED.expires_at
       WHERE leader_lease.expires_at < now()      -- previous leader let it lapse
          OR leader_lease.owner_id = EXCLUDED.owner_id  -- or it's us renewing
     RETURNING fence`,
    [resource, nodeId, String(leaseMs)]
  );
  return rows.length ? Number(rows[0].fence) : null;
}
```

This is a real, correct solution. It is what most teams should ship. Its limits are honest ones: your availability is now capped by your database's availability, and it costs one write per renewal interval per resource (trivial — a few writes per second at most). If you already trust Postgres with your money, you can trust it with your leader lock. Don't run Raft to protect a cron job.

The `SELECT ... FOR UPDATE` variant does the same thing with an explicit row lock:

```javascript
// Same idea, different mechanism: take an exclusive row lock inside a transaction,
// inspect the lease, and take it over if it has expired. Slightly more code,
// but easier to reason about when the acquire logic has extra conditions.
await db.query('BEGIN');
const { rows } = await db.query(
  `SELECT owner_id, fence, expires_at FROM leader_lease
    WHERE resource = $1 FOR UPDATE`, [resource]);   // blocks other acquirers
// ... decide, then UPDATE, then COMMIT
```

### 2. Split-brain — the danger everything else exists to prevent

A **network partition** is when the network drops messages between two groups of nodes, while both groups keep running perfectly. Neither side is dead. Each side simply cannot *see* the other.

From inside a partition, "the other nodes are dead" and "I can't reach the other nodes" look **exactly the same**. There is no message you can send to distinguish them — that's the fundamental limit. (Recall from [09 — CAP Theorem](./09-cap-theorem.md) that during a partition you must choose consistency or availability; leader election is that choice made concrete.)

So if each side independently elects a leader, you get **two leaders**, each accepting writes, each certain it's legitimate. When the partition heals, you have two divergent histories and no principled way to merge them. Money has been double-spent. Orders have been double-shipped.

### 3. Quorum — why a majority makes two leaders impossible

The fix is beautiful and one line long:

> **A node is only leader if a strict majority (N/2 + 1) of the cluster voted for it.**

Why this works: two majorities of the same set **must overlap**. In a 5-node cluster, any majority is at least 3 nodes. Two disjoint groups of 3 would need 6 nodes, and you only have 5. So at least one node is in both groups — and that node will not vote for two different leaders in the same term. **Therefore at most one leader can exist. Ever.**

Note what you've traded away: the minority side, which is perfectly healthy, now **refuses to serve writes**. That's not a bug. It's the price of correctness, and it's the CP choice in CAP.

**This is why cluster sizes are odd.** Do the arithmetic:

| Nodes (N) | Quorum (N/2+1) | Failures tolerated (N − quorum) |
|-----------|----------------|----------------------------------|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| **3** | **2** | **1** |
| 4 | 3 | 1 |
| **5** | **3** | **2** |
| 6 | 4 | 2 |
| **7** | **4** | **3** |

Look at rows 3 and 4. A 4-node cluster needs 3 votes and therefore tolerates exactly **one** failure — the same as a 3-node cluster — while costing you an extra machine, an extra thing to patch, and one more node whose failure could matter. A 4-node cluster is strictly worse than a 3-node one. **Even sizes are pure waste.** Same for 6 vs 5. This is the whole reason you see 3, 5, and 7 everywhere and never 4 or 6.

(A 2-node cluster is the worst of all: quorum is 2, so *either* node dying takes the whole thing down. It's less available than a single node.)

### 4. The Bully algorithm — the simple textbook one

Every node has a unique, comparable ID. The rule: **the highest-ID live node wins.**

When node `P` notices the leader is silent:

1. `P` sends an `ELECTION` message to every node with an ID **higher** than its own.
2. **If nobody answers** (all higher nodes are dead), `P` declares itself leader and broadcasts `COORDINATOR` to everyone.
3. **If any higher node answers** with `OK`, `P` gives up and waits — a bigger node has taken over the election.
4. Any node that receives an `ELECTION` from a lower node replies `OK` and then starts its *own* election with the nodes above it.

Worked example — 5 nodes, IDs 1–5. Node 5 is leader. Node 5 crashes.

```
Node 2 notices silence.
  2 → ELECTION → 3, 4, 5
  3 replies OK.  4 replies OK.  5 is dead, silent.
  Node 2 stands down (someone bigger is on it).

Node 3 (having replied OK) starts its own election.
  3 → ELECTION → 4, 5
  4 replies OK.  5 silent.
  Node 3 stands down.

Node 4 starts its own election.
  4 → ELECTION → 5
  5 silent. No replies.
  Node 4 declares itself LEADER, broadcasts COORDINATOR to 1, 2, 3.
```

It's called "bully" because a higher-ID node barges in and takes leadership from a lower one, always.

**Why you won't ship it:**
- It assumes **reliable failure detection**. "Node 5 didn't reply" is treated as "node 5 is dead" — but it may just be a partition. Bully has **no quorum rule**, so a partition produces two leaders, one per side. It is not partition-safe.
- It's **chatty**: O(N²) messages in the worst case when everyone notices at once.
- It **thrashes**: the moment node 5 comes back, it bullies its way back into leadership, causing another disruptive handover for no benefit.

Learn it because it's on exams and it builds intuition. Then use Raft.

### 5. Raft — the modern standard, properly

Raft was explicitly designed to be *understandable*. Its core is three roles and one clever trick.

**The three roles:**

- **Follower** — passive. Responds to the leader and to vote requests. Every node starts here.
- **Candidate** — a follower whose election timeout fired. It is campaigning for votes.
- **Leader** — won a majority. Handles all client writes, replicates them, sends heartbeats.

**Terms — the logical clock.** Time in Raft is divided into numbered **terms** (1, 2, 3, …). Each term has at most one leader. Every message carries its sender's term. The rule that makes everything work:

> If a node sees a term **higher** than its own, it immediately updates its term and reverts to Follower.

A term is not a wall-clock time. It's a monotonically increasing integer that lets any node instantly tell whose information is more recent — a stale leader whose messages carry term 4 gets rejected by a follower already in term 5, and the moment that stale leader hears about term 5, it steps down on its own. No timing assumptions required.

**The election, step by step:**

1. Each follower runs a **randomized election timeout**, typically drawn uniformly from **150–300ms**. Every heartbeat from the leader resets it.
2. If the timeout fires without hearing from a leader, the follower:
   - becomes a **Candidate**,
   - **increments its term** (say 4 → 5),
   - **votes for itself**,
   - sends `RequestVote(term=5, lastLogIndex, lastLogTerm)` to all peers.
3. Each peer grants its vote **at most once per term**, and only if the candidate's log is at least as up-to-date as its own (see below).
4. If the candidate collects votes from a **majority**, it becomes **Leader** and immediately starts sending heartbeats.
5. If it instead hears from a legitimate leader with a term ≥ its own, it reverts to Follower.
6. If nobody wins (split vote), the timeout fires again and a new term starts.

**Why the randomization matters — this is the whole insight.** If every follower used a fixed 200ms timeout, then when the leader died, *all* of them would time out at the same instant, all become candidates, all vote for themselves, and none would reach a majority. They'd retry — and collide again. Forever. Randomizing the timeout means one node almost always fires meaningfully earlier than the rest, wins the votes, and shuts the election down before anyone else even wakes up. **The randomness is what breaks the symmetry.**

**Heartbeats.** The leader sends empty `AppendEntries` RPCs every ~50ms (well under the minimum 150ms election timeout, so followers always hear from it in time). Each one resets every follower's election timer. A live leader therefore *suppresses* elections just by existing.

**Log replication and commit:**

1. A client sends a write to the leader.
2. The leader **appends** it to its own log — uncommitted.
3. It sends `AppendEntries` with that entry to all followers.
4. Once a **majority** of nodes have *stored* the entry, the leader marks it **committed**, applies it to its state machine, and replies to the client.
5. Followers learn of the commit on the next `AppendEntries` and apply it too.

Note the ordering: **replicate to a majority first, then commit, then answer the client.** That's what makes the write durable across a leader crash.

**The log-up-to-date check — why no committed data is ever lost.** A voter refuses to vote for a candidate whose log is *behind* its own. "Up-to-date" is compared as: higher `lastLogTerm` wins; if equal terms, the longer log wins.

Now chain it together. A committed entry lives on a **majority** of nodes. To *win*, a candidate needs votes from a **majority**. Those two majorities must overlap in at least one node — and that node has the committed entry and will refuse to vote for anyone missing it. **Therefore any node that can win an election already has every committed entry.** Committed data cannot be lost. That single argument is the heart of Raft's safety.

**Paxos.** Paxos (Lamport, 1989) is the original consensus algorithm and is equivalent in power to Raft — anything one can do, the other can. It is also famously, painfully hard to understand and, worse, hard to *implement correctly*: the paper describes single-value consensus, and turning that into a real replicated log ("Multi-Paxos") is left largely to the reader, which is exactly where implementations diverge and break. Raft's authors named their paper "In Search of an Understandable Consensus Algorithm" — understandability *was the design goal*. Know that Paxos exists, that it underpins systems like Google's Chubby, and that Raft is what you'd actually reach for.

### 6. Fencing tokens — surviving the zombie leader

Here is the failure that catches everyone.

Your leader holds a 10-second lease. It begins a write. Then the V8 garbage collector stops the world for 15 seconds (or the VM gets live-migrated, or the host is starved of CPU). The process isn't dead — it's **frozen**. It has no idea time is passing.

Meanwhile the lease expires. The other nodes elect a new leader. That leader starts doing leader things. Then the old process wakes up, resumes exactly where it left off — mid-write — still believing it is the leader, because from its perspective no time passed at all.

**You cannot fix this with better timeouts.** There is no timeout short enough to be safe and long enough to be usable, because you cannot bound how long a process might be paused. Any solution based on "surely it won't pause for more than X" is wrong.

The fix is to **make the downstream system reject stale writes**, rather than trying to make the stale leader behave.

Every time leadership is granted, hand out a **fencing token**: a strictly increasing integer (the `fence` column in §1; in Raft, the term serves this role). The leader must include it with every write. The storage system remembers the highest token it has seen and **rejects anything lower**.

```
Time ──────────────────────────────────────────────────────────────▶

Node A  ══leader, fence=33══╗                    ╔══ wakes up, resumes
                            ║  (GC pause 15s)    ║  write(fence=33)
                            ╚════════════════════╝        │
                                                          ▼
Node B                      ┌── lease expires ──┐   ┌───────────────┐
                            │ elected, fence=34 │──▶│   STORAGE     │
                            │ write(fence=34) ✓ │   │ highest = 34  │
                            └───────────────────┘   │               │
                                                    │ 33 < 34       │
                                                    │  → REJECT ✗   │
                                                    └───────────────┘
```

The zombie's write is rejected not because anyone detected the pause — nobody did — but because it carries an old number. **Safety no longer depends on any timing assumption.** That's the whole point. This is the same mechanism you'll see in [84 — Distributed Locking](./84-distributed-locking.md); a leader lease *is* a distributed lock with a longer TTL and a renewal loop.

### 7. A working Node.js lease-based leader elector

This is production-shaped: acquire, renew in the background, release on shutdown, and expose the fencing token.

```javascript
import { EventEmitter } from 'node:events';
import { randomUUID } from 'node:crypto';

/**
 * Leader election by database lease.
 *
 * Safety rests on ONE thing: `tryAcquire` is atomic at the DB level, so of N
 * racing nodes exactly one can win an expired lease. Everything else here is
 * bookkeeping around that single atomic operation.
 */
export class LeaderElector extends EventEmitter {
  constructor(db, { resource, leaseMs = 15_000, nodeId = randomUUID() }) {
    super();
    this.db = db;
    this.resource = resource;
    this.leaseMs = leaseMs;
    // Renew at 1/3 of the lease so we get two full retries before losing it.
    // Renewing at 1/2 leaves only one retry; at 9/10 a single blip costs leadership.
    this.renewMs = Math.floor(leaseMs / 3);
    this.nodeId = nodeId;
    this.isLeader = false;
    this.fence = null;     // our fencing token; null when we are not leader
    this.timer = null;
    this.stopped = false;
  }

  start() {
    this.stopped = false;
    this.#tick();
  }

  async #tick() {
    if (this.stopped) return;
    try {
      const fence = await tryAcquire(this.db, this.resource, this.nodeId, this.leaseMs);
      if (fence !== null) this.#onAcquiredOrRenewed(fence);
      else this.#onLost();
    } catch (err) {
      // A DB error means we CANNOT prove we still hold the lease. The safe
      // assumption is that we do not. Step down immediately, do not gamble.
      this.emit('error', err);
      this.#onLost();
    }
    // Poll fast while a follower (so failover is quick); renew slowly while leader.
    const next = this.isLeader ? this.renewMs : this.renewMs * 2;
    this.timer = setTimeout(() => this.#tick(), next);
  }

  #onAcquiredOrRenewed(fence) {
    if (!this.isLeader) {
      this.isLeader = true;
      this.fence = fence;
      this.emit('elected', fence);       // start the cron job here
    } else if (fence !== this.fence) {
      // Our token changed under us => the lease lapsed and we re-took it. We were
      // NOT continuously leader, so anything mid-flight must be treated as suspect.
      this.fence = fence;
      this.emit('fence-changed', fence);
    }
  }

  #onLost() {
    if (this.isLeader) {
      this.isLeader = false;
      this.fence = null;
      this.emit('demoted');              // stop the cron job here, immediately
    }
  }

  /** Every leader-only side effect must carry this. If null, you are not leader. */
  currentFence() {
    return this.isLeader ? this.fence : null;
  }

  async stop() {
    this.stopped = true;
    clearTimeout(this.timer);
    if (this.isLeader) {
      // Graceful handover: voluntarily expire the lease so a peer can take over
      // in milliseconds instead of waiting out the full TTL.
      await this.db.query(
        `UPDATE leader_lease SET expires_at = now()
          WHERE resource = $1 AND owner_id = $2`,
        [this.resource, this.nodeId]
      );
      this.#onLost();
    }
  }
}
```

Using it — note that the fence is passed *down* into the write:

```javascript
const elector = new LeaderElector(db, { resource: 'billing-cron', leaseMs: 15_000 });
let job = null;

elector.on('elected', (fence) => {
  console.log(`I am the leader (fence=${fence}). Starting billing job.`);
  job = setInterval(() => runBilling(elector), 60_000);
});

elector.on('demoted', () => {
  console.log('Lost leadership. Stopping billing job.');
  clearInterval(job);
  job = null;
});

async function runBilling(elector) {
  // Re-check at the moment of use, not at the moment of scheduling: we may have
  // been demoted in between. This narrows the window but does NOT close it —
  // only the fence check inside the write closes it.
  const fence = elector.currentFence();
  if (fence === null) return;

  // The DB refuses the write if a newer leader has already written with a
  // higher fence. A zombie leader physically cannot corrupt the ledger.
  await db.query(
    `UPDATE billing_state
        SET last_run = now(), fence = $1
      WHERE id = 1 AND fence <= $1`,   // <-- the fence check IS the safety
    [fence]
  );
}

process.on('SIGTERM', () => elector.stop());
```

A Redis variant is the same shape: `SET lease:billing <nodeId> NX PX 15000` to acquire, and `INCR fence:billing` on handover. But be aware — a single Redis node is not a consensus system, and Redlock across multiple Redis nodes is contested precisely because it leans on timing assumptions. Fencing tokens are what save you either way.

### 8. What you should actually use

You almost never implement consensus yourself. You lean on a system that already did it, and you get its correctness proofs for free.

| Tool | How you get a leader | Use when |
|------|----------------------|----------|
| **etcd** | Raft internally; `Lease` + atomic compare-and-swap on a key. Its `revision` number is a ready-made fencing token. | You're on Kubernetes — it's already running. K8s controllers elect leaders this exact way. |
| **ZooKeeper** | Ephemeral sequential znodes (recipe below) | JVM/Kafka/Hadoop ecosystems |
| **Consul** | Sessions + KV lock with a `LockIndex` (a fencing token) | You already run Consul for [72 — Service Discovery](./72-service-discovery.md) |
| **Postgres / MySQL** | Lease row, as in §1 and §7 | You just need one cron job to run once. **Most teams. Start here.** |
| **Redis** | `SET NX PX` + renewal | Speed matters more than absolute safety; pair with fencing |

**The ZooKeeper recipe** (worth knowing by name — interviewers ask):

1. Every candidate creates an **ephemeral sequential** znode under `/election/`. *Ephemeral* = ZooKeeper deletes it automatically when that client's session dies. *Sequential* = ZooKeeper appends a monotonically increasing counter, atomically.
2. You get `/election/n_0000000017`, `/election/n_0000000018`, and so on.
3. **The lowest sequence number is the leader.** Everyone else is a follower.
4. Each follower sets a **watch on the node immediately before its own** — *not* on the leader. If it watched the leader, then when the leader died, every follower would wake up at once (a "herd effect"). Watching your immediate predecessor means exactly **one** node wakes up per failure.
5. Leader crashes → session expires → its ephemeral znode vanishes → the next-lowest node's watch fires → it checks, finds it is now lowest, and becomes leader.

The sequence number is a natural fencing token. Ephemeral nodes give you crash detection for free.

---

## Visual / Diagram description

### Diagram 1: Split-brain, and how quorum stops it

```
        BEFORE PARTITION (5 nodes, quorum = 3)
        ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
        │ N1 │  │ N2 │  │ N3 │  │ N4 │  │ N5 │
        │LEAD│◀─┤FOLL│  │FOLL│  │FOLL│  │FOLL│
        └────┘  └────┘  └────┘  └────┘  └────┘
           heartbeats every 50ms suppress all elections


        DURING PARTITION  ── network splits here ──
     ┌─────────────────────┐ ║ ┌───────────────────────────┐
     │  ┌────┐    ┌────┐   │ ║ │  ┌────┐   ┌────┐   ┌────┐ │
     │  │ N1 │    │ N2 │   │ ║ │  │ N3 │   │ N4 │   │ N5 │ │
     │  └────┘    └────┘   │ ║ │  └────┘   └────┘   └────┘ │
     │                     │ ║ │                            │
     │  MINORITY: 2 nodes  │ ║ │   MAJORITY: 3 nodes        │
     │  N1 needs 3 votes.  │ ║ │   N3 times out, campaigns, │
     │  Can only get 2.    │ ║ │   gets 3 of 5 votes.       │
     │                     │ ║ │                            │
     │  N1 STEPS DOWN.     │ ║ │   N3 → LEADER (term 6).    │
     │  Rejects all writes.│ ║ │   Accepts writes. ✓        │
     │  (unavailable, but  │ ║ │                            │
     │   SAFE)          ✗  │ ║ │                            │
     └─────────────────────┘ ║ └───────────────────────────┘
```

The left side is healthy and running — and still refuses to serve. That refusal is the entire point. Two disjoint groups cannot both reach 3 out of 5, so **the right side is the only side that can ever have a leader.** Exactly one leader exists, cluster-wide, at all times. When the partition heals, N1 and N2 hear term 6, discover they're behind, revert to Follower, and catch up from N3's log.

### Diagram 2: The Raft state machine

```
                       ┌──────────────────────────────────────┐
                       │  starts here, or any time it sees a  │
                       │  message with a HIGHER term          │
                       ▼                                      │
                 ┌───────────┐                                │
      ┌─────────▶│ FOLLOWER  │◀───────────────────────┐       │
      │          └─────┬─────┘                        │       │
      │                │                              │       │
      │   election timeout fires (150–300ms,          │       │
      │   RANDOMIZED — breaks vote-splitting)         │       │
      │                │                    discovers current │
      │                ▼                    leader, or a      │
      │          ┌───────────┐              higher term       │
      │          │ CANDIDATE │──────────────────────┘         │
      │          └─────┬─────┘                                │
      │                │                                      │
      │      wins votes from a MAJORITY                       │
      │      (N/2 + 1, including itself)                      │
      │                │                                      │
      │                ▼                                      │
      │          ┌───────────┐                                │
      └──────────│  LEADER   │────────────────────────────────┘
   discovers a   └─────┬─────┘   sends AppendEntries heartbeats
   higher term         │         every ~50ms, resetting every
                       │         follower's election timer
             split vote: nobody
             got a majority →
             timeout again,
             new term, retry
```

Redraw this from memory before an interview. The three boxes, the five arrows, and the two labels that matter: **randomized timeout** on Follower→Candidate, and **majority** on Candidate→Leader.

---

## Real world examples

### 1. Kubernetes — etcd and the controller-manager

Kubernetes stores all cluster state in **etcd**, which uses Raft internally and is deployed as 3 or 5 nodes (never 4). If etcd loses quorum, the cluster's control plane goes read-only — running pods keep serving, but you can't schedule anything new. That's the CP trade, made deliberately.

Separately, `kube-controller-manager` and `kube-scheduler` run as multiple replicas for availability but must have **only one active instance** — you don't want two schedulers assigning the same pod to two nodes. They achieve this with a **leader election lease**: they contend for a `Lease` object in the API (backed by etcd), renew it every few seconds, and the non-leaders sit idle in standby doing nothing. It is precisely the lease pattern from §7, with etcd instead of Postgres. Kubernetes solves the "10 replicas all running the cron" problem exactly the way you would.

### 2. Kafka — from ZooKeeper to its own Raft

Every Kafka **partition** has one leader broker; all reads and writes for that partition go through it, and followers replicate from it. The controller decides which broker leads which partition.

Historically, Kafka used **ZooKeeper** to elect the controller and store metadata — the ephemeral-znode recipe from §8. Modern Kafka (KRaft mode) dropped ZooKeeper entirely and runs its **own Raft implementation** in a quorum of controller nodes. Same problem, same solution family — they just stopped outsourcing it. Note the fencing idea reappears here as the **leader epoch**: a number that increases on every leadership change, used to reject writes and replication requests from stale leaders.

### 3. PostgreSQL HA (Patroni)

A single-primary Postgres cluster is a leader-election problem wearing a database costume: exactly one node accepts writes, replicas stream from it, and on failure one replica must be promoted — but *never two*. (Recall from [63 — Database Replication](./63-database-replication.md) that single-leader replication assumes exactly one writer; two primaries means silently divergent data.)

**Patroni**, the standard HA tool, doesn't invent consensus. It stores the leader key in etcd/Consul/ZooKeeper with a TTL, and each Patroni agent renews that key while its Postgres is healthy. If the primary's node hangs, its key expires, a replica acquires the key via an atomic compare-and-set, and only *then* promotes itself. Patroni also **fences** the old primary: if an agent can't renew its key, it demotes its own Postgres to read-only before anything else can happen. That's §6, in production.

---

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **Hardcoded leader** | Zero complexity | No failover. Not a real answer. |
| **DB lease row** | Simple, correct, uses infra you already run and trust, easy to debug (it's a row you can `SELECT`) | Availability capped by the DB; the DB becomes a coordination bottleneck; failover takes a full lease TTL |
| **Bully algorithm** | Easy to explain and implement | No quorum → **splits brain on partition**. O(N²) chatter. Thrashes on leader recovery. Don't ship it. |
| **Raft (embedded)** | Provably safe, partition-tolerant, well-understood, fast failover | Real work to run correctly; you now operate a consensus cluster |
| **ZooKeeper / etcd / Consul** | Battle-tested, someone else's correctness proofs, fencing tokens included | Another system to run, monitor, and pay for; adds a network hop |

| Tuning knob | Set it low | Set it high |
|-------------|-----------|-------------|
| **Lease TTL / election timeout** | Fast failover (seconds of downtime) but **spurious elections** — a GC pause or network blip demotes a healthy leader and causes a needless, disruptive handover | Stable leadership, but a real crash leaves you **leaderless for the full TTL** (nothing gets done for 30s) |
| **Cluster size** | 3 nodes: cheap, tolerates 1 failure, fastest consensus (only 2 acks needed) | 7 nodes: tolerates 3 failures, but every write waits for 4 acks → slower, and more network chatter |

**Rule of thumb:** If you need "exactly one node runs this job," use a **database lease with a fencing token** — 50 lines of code, correct, and boring. Reach for Raft/etcd/ZooKeeper only when leadership guards *replicated state* (a database, a log, a shard map) rather than just a side effect. And if you're already on Kubernetes, etcd is right there — use the Lease API instead of building anything. Never run an even-numbered cluster. Never trust a timeout to prevent a zombie leader — that's what fencing tokens are for.

---

## Common interview questions on this topic

### Q1: "You have 10 replicas of a service with a nightly cron job. How do you make sure it runs exactly once?"
**Hint:** Name the bug first (all 10 fire, every customer gets 10 emails), then give the lease answer: an atomic conditional `UPDATE` on a `leader_lease` row with a TTL, renewed in the background at 1/3 the TTL. The winner runs the job; the losers idle. Mention that "exactly once" is aspirational — the job may run twice across a handover, so **make the job idempotent** too. Belt and braces. That last point is what separates a strong answer from an average one.

### Q2: "What is split-brain, and how does quorum prevent it?"
**Hint:** A partition makes each side believe the other is dead, so both elect leaders and accept conflicting writes. Quorum requires a **majority (N/2+1)** to elect. Two disjoint majorities can't exist in one cluster — they'd need more than N nodes — so at most one leader is possible. Say the consequence out loud: **the minority side becomes unavailable even though it's healthy.** That's the CP choice, and interviewers want to hear you name the cost.

### Q3: "Why are Raft clusters always 3, 5, or 7 nodes — never 4?"
**Hint:** Do the arithmetic on the spot. N=3 → quorum 2 → tolerates 1 failure. N=4 → quorum 3 → tolerates **1** failure. Same fault tolerance, one extra machine, more nodes to fail, slower writes (3 acks instead of 2). Even sizes buy you literally nothing.

### Q4: "Your leader has a 10-second lease. It hits a 30-second GC pause. A new leader is elected. Then the old leader wakes up mid-write. What happens?"
**Hint:** This is the zombie-leader question and it's a trap: no timeout tuning saves you, because you cannot bound a pause. The answer is **fencing tokens** — leadership hands out a strictly increasing integer; every write carries it; storage remembers the highest seen and rejects anything lower. The zombie's write fails on arrival. Safety stops depending on timing at all. In Raft the term serves this role; in Kafka it's the leader epoch; in ZooKeeper it's the znode sequence number.

### Q5: "Why is the Raft election timeout randomized?"
**Hint:** With a fixed timeout, all followers time out at the same instant when the leader dies, all become candidates, all vote for themselves, nobody gets a majority, and they retry and collide again — a livelock. Randomizing over 150–300ms means one node almost always fires first and wins before the others wake up. The randomness breaks the symmetry. Add: heartbeats (~50ms) must be well below the minimum timeout (150ms) so a healthy leader reliably suppresses elections.

### Q6: "Do you need consensus for this?"
**Hint:** Usually no — and saying so is a strength. If leadership guards a **side effect** (a cron, a poller, a work assigner), a DB lease plus an idempotent job is enough. You need real consensus when leadership guards **replicated state that must not diverge** — a log, a database, a shard map. Reaching for Raft to protect a cron job is over-engineering, and a good interviewer will say so.

---

## Practice exercise

### Build a leader-elected cron in 3 processes

**Goal:** three Node processes, exactly one of which prints a heartbeat message every second — and when you kill it, another takes over within seconds. About 30–40 minutes.

**Produce:**

1. **Schema.** Create the `leader_lease` table from §1 (Postgres, or SQLite if you want zero setup — the atomic-write argument holds either way).

2. **The elector.** Implement `LeaderElector` from §7. Lease TTL = 6 seconds, renew every 2 seconds. On `elected`, start a `setInterval` printing `[<nodeId>] LEADER fence=<n> tick`. On `demoted`, clear the interval.

3. **Run three copies** in three terminals. Confirm exactly one prints. Kill the leader with `Ctrl+C` and time how long until a new leader starts printing. Then kill the leader with `kill -9` (no graceful `stop()`) and time it again. **Explain the difference** — this is the whole point of the exercise. Write down both numbers.

4. **Simulate a zombie.** Add an env-var-triggered `PAUSE_MS` that blocks the leader's event loop with a busy-wait for 10 seconds (longer than the lease):
   ```javascript
   const until = Date.now() + 10_000;
   while (Date.now() < until) {}   // blocks the event loop, just like a GC pause
   ```
   Watch a second node take leadership while the first is frozen. When the frozen one resumes, does it still think it's the leader? (It will.) Then add a `fence` column to a `job_state` table and make the tick do `UPDATE job_state SET ticks = ticks + 1, fence = $1 WHERE fence <= $1`. Confirm the zombie's update affects **0 rows** while the new leader's succeeds.

5. **Write down** the answer to: *what is the maximum window during which two of your processes both believe they are leader?* (Hint: it's roughly the lease TTL, and it is **not zero** — which is exactly why step 4 matters.)

---

## Quick reference cheat sheet

- **Leader election** = the cluster automatically agreeing on exactly one node in charge, and re-electing when it dies.
- **Why a leader:** turns N-way coordination into one-way instruction. Needed for single-writer ordering, cron jobs, and work assignment.
- **Split-brain** = a partition causes two leaders to be elected; both accept writes; data diverges. This is the danger everything else exists to prevent.
- **Quorum = N/2 + 1.** Two disjoint majorities cannot exist in one cluster, so at most one leader can ever win. This is the entire safety argument.
- **Odd cluster sizes only (3, 5, 7).** N=4 tolerates the same 1 failure as N=3 while costing more and writing slower. Even sizes are waste.
- **The minority side goes unavailable during a partition** — healthy, but refusing writes. That's the price of correctness (the CP choice).
- **Bully algorithm:** highest live ID wins. Simple, chatty (O(N²)), **not partition-safe** (no quorum). Textbook only.
- **Raft roles:** Follower → (election timeout) → Candidate → (majority of votes) → Leader. Higher term seen ⇒ immediately revert to Follower.
- **Terms are logical clocks**, not wall-clock time — they let any node instantly tell whose information is stale.
- **Randomized election timeout (150–300ms)** is what prevents an infinite split vote. Heartbeats every ~50ms suppress elections.
- **Commit rule:** the leader commits an entry only once a **majority has stored it**. Combined with the log-up-to-date vote check, this guarantees any node that can win an election already holds every committed entry — so committed data is never lost.
- **Paxos** is equivalent in power and famously hard to understand; Raft was designed explicitly for understandability.
- **Fencing token** = a strictly increasing number handed out with leadership; downstream systems reject anything lower. This is the ONLY defence against a zombie leader waking from a GC pause.
- **You cannot bound a pause.** Never make safety depend on a timeout. Make it depend on a number.
- **In practice:** use etcd (on K8s), ZooKeeper (ephemeral sequential znodes), Consul, or — most often and most sensibly — a **database lease row**. Don't hand-roll consensus.
- **Also make the job idempotent.** Even a correct election has a handover window; assume the job may run twice.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [84 — Distributed Locking](./84-distributed-locking.md) — a leader lease *is* a distributed lock with a renewal loop; fencing tokens come from there |
| **Next** | [72 — Service Discovery](./72-service-discovery.md) — once you've elected a leader, everyone else has to find out who it is |
| **Related** | [09 — CAP Theorem](./09-cap-theorem.md) — quorum-based election is the CP choice made concrete: the minority partition sacrifices availability for safety |
| **Related** | [63 — Database Replication](./63-database-replication.md) — single-leader replication is the biggest real consumer of leader election; promoting a replica *is* an election |
| **Related** | [08 — Consistency Models](./08-consistency-models.md) — a single leader is the cheapest way to get linearizability, because one node imposes a total order for free |
