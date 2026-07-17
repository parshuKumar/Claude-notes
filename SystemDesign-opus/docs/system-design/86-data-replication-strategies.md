# 86 — Data Replication Strategies — Quorum, Gossip, Anti-Entropy
## Category: HLD Components

---

## What is this?

**Leaderless replication** is a way of keeping N copies of your data in sync **without any node being "the boss."** Instead of sending every write to a single primary that then fans it out to replicas, the client (or a coordinator node acting on its behalf) sends the write to **several replicas at once** and reads from **several replicas at once**. Consistency comes not from a leader, but from **overlap** — you write to enough nodes and read from enough nodes that the two sets are guaranteed to touch.

The real-world analogy: a rumour. There is no Ministry of Truth that owns the rumour. You tell three friends; later, someone asks four friends what happened. As long as your three and their four **overlap by at least one person**, they hear the newest version. That single sentence is the whole doc.

> **Recall from [63 — Database Replication](./63-database-replication.md):** that topic covered **leader-based** replication — one primary accepts writes, replicas follow the log, and if the primary dies you run a failover election. This doc is the **other family**: Dynamo-style, no leader, no failover, tunable consistency. Keep them mentally separate. They solve the same problem with opposite philosophies.

---

## Why does it matter?

**What breaks without it:** In leader-based replication, the leader is a bottleneck and a liability. Every write funnels through one machine. When that machine dies, you have a **failover gap** — 10 to 30 seconds where the system rejects writes while a new leader is elected. For a shopping cart at Amazon on Black Friday, or a metrics ingest pipeline taking 500k writes/sec, that gap is unacceptable. Amazon's original Dynamo paper was written precisely because "add to cart" must **never** fail, even during a datacenter partition.

Leaderless replication removes the failover step entirely. There is no leader, so there is nothing to fail over. A node being down just means you use a different node.

**The interview angle:** Quorum math (`W + R > N`) is one of the most reliably-asked distributed systems questions at senior level. If an interviewer says "Cassandra," "DynamoDB," "design a key-value store," or "how do you tune consistency," they are fishing for it. Being able to *draw the overlap* and reason about `N=3, W=2, R=2` fluently is table stakes.

**The real-work angle:** Every time you pick a Cassandra consistency level (`ONE` vs `QUORUM` vs `LOCAL_QUORUM`), configure a DynamoDB read as strongly-consistent, or debug why a user "saw an old value," you are living inside this topic. Getting it wrong shows up as ghost data, lost writes, and 3am pages.

---

## The core idea — explained simply

### The Village Notice Board Analogy

A village of 3 scribes (`A`, `B`, `C`) each keeps a copy of the village ledger. There is **no head scribe.** Anyone can walk up to any scribe and update the ledger, and anyone can walk up and read it.

Now, a problem. You go to scribe `A` and say "the harvest is 500 bushels." Only `A` writes it down. Later, your neighbour asks scribe `C` — who still has last week's number. Your neighbour reads **stale data.** The village needs a rule.

**The rule they invent:**
- When you *write*, you must tell **at least 2 of the 3** scribes. (`W = 2`)
- When you *read*, you must ask **at least 2 of the 3** scribes and take the newest answer. (`R = 2`)

Why does this work? There are only 3 scribes. If you told 2 of them, and your neighbour asks 2 of them, **there is no way to pick 2 and 2 out of 3 without sharing at least one scribe.** Pigeonhole principle. That shared scribe has your new number. Your neighbour compares the answers, sees one is newer (a timestamp on the entry), and trusts the newer one.

That's it. That's quorum consensus.

**And the bonus:** if scribe `B` is off sick, the village still functions. Writes go to `A` and `C` (that's 2 — enough). Reads ask `A` and `C` (that's 2 — enough). **Nobody had to hold an election to appoint a new head scribe, because there was never a head scribe.**

| Village thing | Distributed systems thing |
|---|---|
| Scribe | Replica node |
| 3 scribes hold the ledger | **N** — the replication factor |
| "Tell at least 2 scribes" | **W** — the write quorum |
| "Ask at least 2 scribes" | **R** — the read quorum |
| Guaranteed shared scribe | The **overlap** — the reason `W + R > N` works |
| Timestamp on the ledger entry | Version / vector clock, used to pick the winner |
| The person doing the running around | The **coordinator** node |
| Scribe `B` off sick | A node down — and the system keeps serving |
| Sick scribe catching up later | **Read repair / hinted handoff / anti-entropy** |
| Scribes chatting at the market | **Gossip** — how they learn who's alive |

---

## Key concepts inside this topic

### 1. The leaderless write and read path

There is no primary. Any node can act as the **coordinator** for a request — usually whichever node the client's load balancer happened to hit. The coordinator figures out which N nodes own this key (via consistent hashing — a hash ring), fires the request at all N **in parallel**, and waits for the first W (or R) to answer.

```javascript
// A leaderless coordinator. Note: it contacts ALL N replicas,
// but only WAITS for W of them. The rest complete in the background.
class Coordinator {
  constructor(ring, { N = 3, W = 2, R = 2 } = {}) {
    this.ring = ring;   // consistent-hash ring -> which nodes own a key
    this.N = N; this.W = W; this.R = R;
  }

  async put(key, value) {
    const version = Date.now();             // real systems use a vector clock
    const replicas = this.ring.nodesFor(key, this.N);

    // Fire at all N. Resolve as soon as W succeed — don't wait for stragglers.
    const acks = replicas.map(node =>
      node.store(key, { value, version }).then(() => node)
    );
    const winners = await firstNToSucceed(acks, this.W);

    // Nodes that didn't ack in time are NOT rolled back. They will be
    // healed later by read repair / hinted handoff / anti-entropy.
    return { ok: true, version, ackedBy: winners.map(n => n.id) };
  }

  async get(key) {
    const replicas = this.ring.nodesFor(key, this.N);
    const responses = await firstNToSucceed(
      replicas.map(node => node.fetch(key).then(r => ({ node, ...r }))),
      this.R
    );

    // Several replicas answered. They may disagree. Newest version wins.
    const newest = responses.reduce((a, b) => (b.version > a.version ? b : a));

    // READ REPAIR: push the winner back to whoever was behind.
    for (const stale of responses.filter(r => r.version < newest.version)) {
      stale.node.store(key, { value: newest.value, version: newest.version })
        .catch(() => {}); // best-effort, don't block the client's read
    }

    return newest.value;
  }
}

// Resolve when `n` promises succeed; reject only if too many fail to ever reach n.
function firstNToSucceed(promises, n) {
  return new Promise((resolve, reject) => {
    const ok = []; let failed = 0;
    for (const p of promises) {
      p.then(v => { ok.push(v); if (ok.length === n) resolve(ok); })
       .catch(() => { if (++failed > promises.length - n) reject(new Error('quorum not met')); });
    }
  });
}
```

Three things to notice, because they are all interview-relevant:

1. **A failed write is not rolled back.** If only 2 of 3 nodes ack, the write is declared successful and node 3 is simply *behind*. There is no two-phase commit, no abort. Divergence is expected and healed later. This is the philosophical core of Dynamo-style systems: **accept temporary inconsistency, repair it in the background.**
2. **The coordinator contacts all N but waits for W.** So the tail latency you pay is the **Wth-fastest** node, not the slowest.
3. **Reads must resolve conflicts.** Several replicas answer; they may disagree; something has to pick a winner. Last-write-wins by timestamp is the simple option (and can silently drop data); vector clocks / version vectors are the honest option (they detect concurrency and hand siblings back to the app).

---

### 2. Quorum consensus — the heart of it

Three numbers:

- **N** — how many replicas hold each piece of data (the replication factor). Typically 3.
- **W** — how many replicas must **acknowledge a write** before the client is told "OK."
- **R** — how many replicas must **respond to a read** before the client gets an answer.

**The rule:**

> ## `W + R > N`
> If the number of nodes you wrote to plus the number you read from is **greater than** the total number of replicas, then the write set and the read set **must overlap by at least one node.** That overlapping node has the latest value. Since reads take the newest version among the responses, the reader sees the latest write.

It's the pigeonhole principle. You cannot fit two subsets of sizes W and R into a set of size N without them touching, if `W + R > N`.

**Draw this. Every time.**

```
                 N = 3 replicas:  [ A ][ B ][ C ]

   W = 2  (write went to A and B)      R = 2  (read asks B and C)

          WRITE SET                            READ SET
        ┌───────────────┐               ┌───────────────┐
        │               │               │               │
        │   ┌───┐       │   ┌───────┐   │      ┌───┐    │
        │   │ A │       │   │   B   │   │      │ C │    │
        │   └───┘       │   │       │   │      └───┘    │
        │               │   │OVERLAP│   │               │
        └───────────────┼───┤       ├───┼───────────────┘
                        │   └───────┘   │
                        └───────────────┘

        W + R  =  2 + 2  =  4   >   3  =  N     ✔ overlap guaranteed

     Node B is in BOTH sets. B has the newest write.
     The reader gets B's answer, compares versions, and wins.
```

And here is the failure case — the one you must be able to explain:

```
   W = 1  (write went ONLY to A)        R = 1  (read asks ONLY C)

        ┌───────────┐                        ┌───────────┐
        │   ┌───┐   │        ┌───┐           │   ┌───┐   │
        │   │ A │   │        │ B │           │   │ C │   │
        │   └───┘   │        └───┘           │   └───┘   │
        └───────────┘                        └───────────┘
         WRITE SET           (untouched)       READ SET

        W + R  =  1 + 1  =  2   >   3 ?   NO.   ✘ NO OVERLAP

     The reader touched C. C never heard about the write.
     The client reads STALE DATA. This is eventual consistency.
```

#### The configurations, worked out with N = 3

| Config | W + R | Overlap? | Write behaviour | Read behaviour | The trade-off |
|---|---|---|---|---|---|
| **W=3, R=1** | 4 > 3 ✔ | Yes | Must reach **all 3**. One node down or slow ⇒ **writes block/fail** | Ask 1 node, blazing fast | *Read-optimised.* Great for read-heavy, write-rare data (config, feature flags). **Fragile:** zero write fault tolerance. |
| **W=1, R=3** | 4 > 3 ✔ | Yes | Ack from **1 node** — fastest possible write | Must reach **all 3**. One node down ⇒ **reads fail** | *Write-optimised.* Great for high-volume ingest (logs, metrics, sensors). Fragile reads. |
| **W=2, R=2** | 4 > 3 ✔ | Yes | Tolerates **1 node down** | Tolerates **1 node down** | **The balanced default.** Both paths survive a single failure. This is what you say in an interview unless told otherwise. |
| **W=1, R=1** | 2 > 3 ✘ | **No** | Fastest write | Fastest read | *Maximum speed & availability, no consistency guarantee.* You may read stale data until repair catches up. Perfectly fine for a like-counter; catastrophic for a bank balance. |
| **W=3, R=3** | 6 > 3 ✔ | Yes | All 3 | All 3 | Strictest — and **strictly worse than W=2,R=2**: you gain nothing and lose all fault tolerance. A trap answer. |

The general fault-tolerance formula: with `N` replicas, a quorum of `W` tolerates `N − W` node failures on the write path, and `R` tolerates `N − R` on the read path. `N=3, W=2, R=2` ⇒ tolerates 1 failure either way. That's why it's the default.

#### The latency dial

Because the coordinator waits for the **Wth ack**, your write latency is the latency of the **Wth-fastest replica**. Concretely, with three replicas responding in 5ms, 12ms, and 90ms (the last one is GC-pausing):

```
  W = 1  →  wait for the 1st  →  ~5 ms   (weakest consistency)
  W = 2  →  wait for the 2nd  →  ~12 ms  (balanced)
  W = 3  →  wait for the 3rd  →  ~90 ms  (strongest — and you inherit the straggler)
```

**Raising W raises consistency AND raises latency. They are the same knob.** This is exactly the `PACELC` insight from [09 — CAP Theorem](./09-cap-theorem.md): *Else (in the normal case, no partition), you trade **L**atency against **C**onsistency.* Quorum tuning is PACELC's "E" branch made into a config file. Say that sentence in an interview and watch the interviewer's eyebrows go up.

#### Be honest about the limits

`W + R > N` gives you overlap. It does **not** give you linearizability. Real edge cases:

- **Concurrent writes.** Two clients write different values at the same time to overlapping-but-not-identical quorums. Both "succeed." Now the replicas hold genuinely concurrent versions with no happens-before relation. Quorum can't order them — you need vector clocks and a conflict-resolution policy (or you use last-write-wins and silently lose one).
- **A partially-applied write.** The write reached 1 of 3, then the coordinator crashed before hitting W. The client saw an error. But that value is now sitting on one replica — and a later read may or may not surface it. **A failed write is not an undone write.**
- **A write and a read racing.** The write is in flight to 2 nodes; the read hits one that's applied and one that hasn't. The reader may see the new value; a *subsequent* reader may see the old one. Monotonic reads are violated.
- **Node restored from a stale backup**, dropping below the effective quorum count.

> Quorums give you **"probably fresh, and usually provably fresh."** For real linearizability you need consensus (Raft/Paxos) or a compare-and-set primitive layered on top. See [08 — Consistency Models](./08-consistency-models.md).

---

### 3. How the replicas heal — three repair mechanisms

Divergence is not a bug here, it's the design. Nodes go down, writes miss them, and they come back holding stale data. Three mechanisms fix that, and they are complementary — a real system runs all three.

#### (1) Read repair — free, but lazy

When a read collects R responses and they **disagree**, the coordinator already knows the truth (newest version wins). So it pushes the winner back to the stale replicas, asynchronously, after answering the client.

```javascript
// (excerpt from the Coordinator above)
const newest = responses.reduce((a, b) => (b.version > a.version ? b : a));
for (const stale of responses.filter(r => r.version < newest.version)) {
  // Fire-and-forget. The client already has its answer; don't make them wait.
  stale.node.store(key, { value: newest.value, version: newest.version }).catch(() => {});
}
```

- **Cost:** essentially zero. You already paid for the network round-trips.
- **The fatal limitation:** it only repairs data that is **actually being read**. A key that nobody has read since the outage stays stale **forever**. Read repair is great for a hot Twitter timeline and useless for a cold user record from 2019 — which is exactly the record that will be wrong when that user finally logs back in.

#### (2) Hinted handoff — stay writable during a failure

A target replica `C` is down. Instead of failing the write, another node `D` (not normally a replica for this key) accepts it and stores a **hint** — a sticky note that says *"this write isn't mine; it belongs to C; deliver it when C is back."* When `C` recovers, `D` replays the hints to it and deletes them.

```javascript
class Node {
  constructor(id) { this.id = id; this.data = new Map(); this.hints = new Map(); }

  // Normal write for data I own.
  async store(key, record) { this.data.set(key, record); }

  // I'm holding this on behalf of a DOWN node. Not my data — a parcel for a neighbour.
  async storeHint(forNodeId, key, record) {
    if (!this.hints.has(forNodeId)) this.hints.set(forNodeId, []);
    this.hints.get(forNodeId).push({ key, record });
  }

  // Gossip told me `node` is alive again. Deliver its parcels.
  async deliverHints(node) {
    const pending = this.hints.get(node.id) ?? [];
    for (const { key, record } of pending) await node.store(key, record);
    this.hints.delete(node.id);      // only drop them once they're safely delivered
  }
}
```

- **Benefit:** the system stays **writable** through node failures. This is the "always writable" promise of Dynamo — Amazon would rather take your cart addition and reconcile later than tell you "sorry, try again."
- **The nuance — sloppy quorum.** If node `D` (a non-replica) counts toward W, then your W acks did **not** all come from the N nodes that own the key. Your `W + R > N` overlap guarantee is **weakened**: a read of the *real* replicas may not touch the hint-holder. This is a **sloppy quorum** — it trades a consistency guarantee for availability. A **strict quorum** requires W acks from the *designated* N replicas and fails the write otherwise. Cassandra lets you choose (`ANY` = sloppy; `QUORUM` = strict). Know the distinction; it's a favourite follow-up.

#### (3) Anti-entropy — the background scrubber

A periodic background process that picks a peer, compares its dataset against yours, and repairs any differences — **including cold data that nobody reads.** This is what catches the keys read repair will never touch.

The naive version is absurd: "send me all 50 million of your keys and their values so I can diff them." That's terabytes over the wire, repeatedly, forever. The elegant version is a **Merkle tree**.

---

### 4. Merkle trees — comparing millions of keys with a handful of hashes

A **Merkle tree** (hash tree) is a binary tree where:

- **Leaves** = the hash of a *bucket of keys* (a slice of the key range, e.g. all keys hashing into `[0, 1M)`).
- **Internal nodes** = the hash of *its two children's hashes* concatenated. `H(parent) = hash(H(left) + H(right))`.
- **Root** = one hash that fingerprints **the entire dataset**.

The magic property: **if any single key's value changes anywhere in the dataset, the leaf hash changes, which changes its parent, which changes its parent... all the way up to the root.** So the root is a 32-byte summary whose equality means "our datasets are byte-for-byte identical."

```
                       ┌──────────────────┐
                       │   ROOT  = H(1,2) │        <-- compare THIS first
                       │      "9f3a"      │
                       └────────┬─────────┘
                  ┌─────────────┴─────────────┐
        ┌─────────▼────────┐        ┌─────────▼────────┐
        │  N1 = H(L1,L2)   │        │  N2 = H(L3,L4)   │
        │      "c17b"      │        │      "44e0"      │
        └────────┬─────────┘        └────────┬─────────┘
          ┌──────┴──────┐             ┌──────┴──────┐
   ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼───────┐ ┌───▼────────┐
   │ L1 "a1"    │ │ L2 "b2"    │ │ L3 "c3"    │ │ L4 "d4"    │
   │ keys 0-1M  │ │ keys 1-2M  │ │ keys 2-3M  │ │ keys 3-4M  │
   └────────────┘ └────────────┘ └────────────┘ └────────────┘
```

**Walking a comparison between replica A and replica B:**

```
Step 1: A sends B its ROOT hash.
        A root = "9f3a"   B root = "9f3a"   -->  IDENTICAL. Done. ONE hash compared,
                                                  4 million keys verified. Zero data shipped.

        But suppose they differ:
        A root = "9f3a"   B root = "7e21"   -->  DIFFER. Something is wrong. Descend.

Step 2: Compare the two children.
        A: N1="c17b"  N2="44e0"
        B: N1="c17b"  N2="8dd9"
                ^same        ^DIFFERENT
        -->  The left half (keys 0-2M) is PROVABLY IDENTICAL. Never look at it again.
             Recurse ONLY into N2.

Step 3: Compare N2's children.
        A: L3="c3"  L4="d4"
        B: L3="c3"  L4="ff9"
                          ^DIFFERENT
        -->  Divergence is confined to leaf L4: keys 3M-4M.

Step 4: Ship ONLY the keys in bucket L4 (or its sub-buckets) and reconcile them.
```

Four comparisons to find the divergence in a 4-leaf tree. In a real tree of, say, 2^20 leaves (~1 million buckets, each covering a few hundred keys), you find a divergent bucket in **20 comparisons** — `O(log n)` — instead of streaming `O(n)` keys across the network.

```javascript
import { createHash } from 'node:crypto';

const sha = (s) => createHash('sha256').update(s).digest('hex').slice(0, 8);

class MerkleTree {
  // buckets: array of key-ranges, each already hashed to a leaf digest
  constructor(leafHashes) {
    this.levels = [leafHashes];
    let level = leafHashes;
    while (level.length > 1) {
      const parent = [];
      for (let i = 0; i < level.length; i += 2) {
        // A lone last node is promoted (or duplicated) — either is fine, just be consistent.
        parent.push(sha(level[i] + (level[i + 1] ?? level[i])));
      }
      this.levels.push(parent);
      level = parent;
    }
  }
  get root() { return this.levels.at(-1)[0]; }
}

/**
 * Anti-entropy diff. Returns the indexes of the LEAF buckets that differ.
 * Note what this does NOT do: it never touches a value. Only hashes cross the wire.
 */
function diffTrees(a, b) {
  const differing = [];
  const walk = (level, idx) => {
    if (a.levels[level][idx] === b.levels[level][idx]) return; // whole subtree is clean — prune it
    if (level === 0) { differing.push(idx); return; }          // reached a leaf bucket
    walk(level - 1, idx * 2);
    walk(level - 1, idx * 2 + 1);
  };
  walk(a.levels.length - 1, 0);   // start at the root
  return differing;
}

// --- demo ---
const replicaA = new MerkleTree(['a1', 'b2', 'c3', 'd4']);
const replicaB = new MerkleTree(['a1', 'b2', 'c3', 'ff9']);   // one bucket drifted

console.log(replicaA.root === replicaB.root); // false -> work to do
console.log(diffTrees(replicaA, replicaB));   // [3] -> only bucket #3 needs syncing
```

**The two properties that make this worth the space:**
1. **The happy path is one comparison.** Replicas are identical 99.9% of the time. Anti-entropy in the steady state costs one 32-byte hash exchange. That's why you can afford to run it constantly.
2. **The unhappy path prunes aggressively.** Every matching subtree eliminates half the remaining search space, without transferring any of it.

Cassandra builds exactly this during `nodetool repair`. Dynamo used it for replica sync. Git uses the same idea (a Merkle DAG of commits/trees/blobs — that's how `git fetch` figures out what you're missing). Bitcoin puts transactions in a Merkle tree so a light client can verify one transaction's inclusion without downloading the block.

---

### 5. Gossip protocol — how nodes learn who's alive, without a registry

There's no leader, so who keeps the list of nodes? Who knows that node `C` came back? A central registry would be a single point of failure — the exact thing we're trying to eliminate.

Answer: **gossip.** Every second, each node picks **a few random peers** and swaps state with them: "here's what I believe about every node's status and version — tell me yours, and we'll both take the newer info." Information spreads like an infection: 1 knows, then 4, then 16... The technical term really is an **epidemic protocol.**

```javascript
class GossipNode {
  constructor(id) {
    this.id = id;
    // What I believe about everyone. Heartbeat counter + when I last heard it.
    this.view = new Map([[id, { heartbeat: 0, lastSeen: Date.now(), status: 'UP' }]]);
  }

  tick(peers, fanout = 3) {
    // 1. Bump my own heartbeat so others can tell I'm alive.
    this.view.get(this.id).heartbeat++;

    // 2. Pick `fanout` RANDOM peers. Random is the whole trick: no fixed topology
    //    to partition, no coordinator to lose, and coverage grows exponentially.
    for (const peer of pickRandom(peers, fanout)) peer.merge(this.view);

    // 3. Anyone I haven't heard a fresh heartbeat from is suspect, then dead.
    for (const [id, s] of this.view) {
      if (id !== this.id && Date.now() - s.lastSeen > 10_000) s.status = 'DOWN';
    }
  }

  // Merge a peer's view into mine: for each node, the HIGHER heartbeat wins.
  // This makes the merge commutative and idempotent — order of gossip doesn't matter,
  // and receiving the same gossip twice is harmless. (It's a CRDT-ish grow-only merge.)
  merge(theirView) {
    for (const [id, theirs] of theirView) {
      const mine = this.view.get(id);
      if (!mine || theirs.heartbeat > mine.heartbeat) {
        this.view.set(id, { ...theirs, lastSeen: Date.now(), status: 'UP' });
      }
    }
  }
}

const pickRandom = (arr, k) => [...arr].sort(() => Math.random() - 0.5).slice(0, k);
```

#### The spread arithmetic

With fan-out **f = 3**, each round roughly **quadruples** the number of informed nodes (the 1 informed node plus the 3 it told). Informed count ≈ `4^r`.

```
  Cluster of 1000 nodes, fan-out 3:

  Round 0:      1 node knows                    (the write / the "node C is back" fact)
  Round 1:      4                               1 told 3
  Round 2:     16
  Round 3:     64
  Round 4:    256
  Round 5:  ~1000   <-- in theory, everyone

  In practice, some peers picked at random are ALREADY informed (the "coupon
  collector" tail), so full coverage takes a constant factor longer:

  ~O(log_f N) rounds  =  log_4(1000)  ≈ 5 rounds  (idealised)
  ~10 rounds in practice, at 1 gossip/sec  =>  ~10 seconds to full cluster awareness.

  Scale to 100,000 nodes?  log_4(100000) ≈ 8.3 idealised, ~15-20 rounds in practice.
  TEN TIMES the nodes costs only a FEW MORE ROUNDS. That's the logarithm working for you.
```

**Why gossip wins here:**
- **No SPOF.** No registry to lose. Any node can die and the protocol doesn't notice.
- **Constant load per node.** Each node sends `f` messages per round regardless of cluster size — it does *not* get busier as the cluster grows. (Contrast: an all-to-all heartbeat mesh is `O(N²)` messages.)
- **Self-healing.** A partition heals ⇒ gossip resumes ⇒ views reconverge automatically. No operator action.
- **The cost:** membership is only **eventually consistent.** For a few seconds, node `A` may think `C` is up while `B` thinks `C` is down. Your system must tolerate that (and quorum systems do — you just try the node and fall back).

**Used by:** Cassandra (membership + ring state + schema versions), DynamoDB internals, Consul (via SWIM — a refined gossip with indirect probes to reduce false positives), Redis Cluster (its cluster bus is gossip), Riak, and Serf.

---

### 6. Tunable consistency in practice — the actual selling point

Here's the payoff. Because W and R are just numbers on a request, **one application can pick different consistency per query.** No other replication model gives you this dial.

Cassandra's consistency levels (with `N` = replication factor):

| Level | What it means | W+R vs N | Use it for |
|---|---|---|---|
| `ONE` | 1 replica must respond | Weak alone | Metrics, logs, view counts — speed over truth |
| `TWO` / `THREE` | Exactly that many respond | Depends on N | Explicit tuning |
| `QUORUM` | `floor(N/2) + 1` respond (2 of 3) | `QUORUM` write + `QUORUM` read ⇒ **always** `W+R > N` | The default for anything that matters |
| `LOCAL_QUORUM` | A quorum **within the local datacenter only** | Overlap within the DC | Multi-DC: strong locally, no cross-ocean latency |
| `EACH_QUORUM` | A quorum in **every** datacenter (writes) | Strongest cross-DC | Rare; expensive; regulatory use cases |
| `ALL` | Every replica | Strongest, zero fault tolerance | Almost never — one node down and you're offline |
| `ANY` | Any node, **including a hinted-handoff holder** | **Sloppy** — no overlap guarantee | "Never lose a write, don't care when it lands" |

```javascript
// One service. Two writes. Two completely different consistency requirements.
class LedgerService {
  constructor(db) { this.db = db; }

  // MONEY. A stale or lost balance is a lawsuit. Pay the latency.
  // QUORUM write + QUORUM read => 2 + 2 = 4 > 3 => guaranteed overlap.
  async recordPayment(userId, amountCents) {
    return this.db.execute(
      'INSERT INTO payments (user_id, cents, ts) VALUES (?, ?, ?)',
      [userId, amountCents, Date.now()],
      { consistency: 'QUORUM' }              // ~12ms, correct
    );
  }

  // A LIKE COUNT. If it says 41 instead of 42 for two seconds, nobody dies,
  // and this endpoint takes 100x the traffic of payments.
  async incrementLike(postId) {
    return this.db.execute(
      'UPDATE counters SET likes = likes + 1 WHERE post_id = ?',
      [postId],
      { consistency: 'ONE' }                 // ~2ms, eventually correct
    );
  }

  // A user's own profile read, in a multi-region deployment.
  // LOCAL_QUORUM: strong within this region, without a 150ms trans-Atlantic hop.
  async getProfile(userId) {
    return this.db.execute(
      'SELECT * FROM profiles WHERE user_id = ?',
      [userId],
      { consistency: 'LOCAL_QUORUM' }
    );
  }
}
```

**That per-query choice is the entire value proposition.** In a leader-based system, consistency is a property of the *cluster*. In a leaderless system, it's a property of the *request*. You spend strictness where it earns its keep and buy speed everywhere else.

---

### 7. Sloppy quorums and multi-datacenter replication

**Sloppy quorum (recap, because it's the sharp edge):** a *strict* quorum requires W acks from the N nodes that actually **own** the key. A *sloppy* quorum accepts acks from **any** N reachable nodes — home nodes plus stand-ins holding hints. During a partition where a client can reach *some* nodes but not the *right* nodes, a sloppy quorum keeps you writable. But because the stand-ins aren't in the read set, `W + R > N` no longer guarantees overlap — you've bought **durability and availability**, not freshness.

**Multi-datacenter:** replicas are placed across DCs (e.g. `N=6`: 3 in `us-east`, 3 in `eu-west`). A cross-ocean round trip is 80–150ms; requiring a global quorum on every write would make your p99 miserable.

```
   ┌──────────── us-east ────────────┐        ┌──────────── eu-west ────────────┐
   │   ┌────┐    ┌────┐    ┌────┐    │        │   ┌────┐    ┌────┐    ┌────┐    │
   │   │ A1 │    │ A2 │    │ A3 │    │◀──────▶│   │ B1 │    │ B2 │    │ B3 │    │
   │   └────┘    └────┘    └────┘    │  async │   └────┘    └────┘    └────┘    │
   │        LOCAL_QUORUM = 2         │  repl. │        LOCAL_QUORUM = 2         │
   │        ack in ~5 ms  ◀── client │ ~90 ms │        ack in ~5 ms  ◀── client │
   └─────────────────────────────────┘        └─────────────────────────────────┘

   LOCAL_QUORUM: client is acked once 2 of the 3 LOCAL replicas commit (~5ms).
                 The other DC catches up asynchronously (~90ms later).
   Result: strong consistency for users in their own region;
           eventual consistency ACROSS regions.
```

`LOCAL_QUORUM` is the workhorse of every serious multi-region Cassandra deployment: **strong locally, eventual globally.** Pin users to a home region and they get read-your-writes; the cross-region lag only matters if a user hops continents mid-session (or you fail a region over — at which point you may serve slightly stale data, and that's the deal you signed).

---

## Visual / Diagram description

### Diagram 1: The full leaderless write path (N=3, W=2), with node C down

```
                          ┌──────────────┐
                          │    CLIENT    │
                          │ PUT cart:42  │
                          └──────┬───────┘
                                 │ 1. any node can take it — no leader
                                 ▼
                    ┌────────────────────────┐
                    │   COORDINATOR (node D) │
                    │  hash("cart:42") ──▶   │
                    │  owners = [A, B, C]    │
                    └───┬────────┬────────┬──┘
             2. fan out │        │        │  (all 3 in parallel)
              ┌─────────┘        │        └──────────┐
              ▼                  ▼                   ▼
       ┌────────────┐     ┌────────────┐      ┌ ─ ─ ─ ─ ─ ─ ┐
       │  NODE A    │     │  NODE B    │        NODE C
       │  ✔ stored  │     │  ✔ stored  │      │  ✘ DOWN     │
       │  v=1712…   │     │  v=1712…   │        (gossip knew)
       └─────┬──────┘     └─────┬──────┘      └ ─ ─ ─ ─ ─ ─ ┘
             │ ack              │ ack                 │
             └────────┬─────────┘                     │ 3. HINTED HANDOFF
                      ▼                               ▼
              W = 2 acks reached              ┌───────────────┐
              ✔ CLIENT GETS "OK"              │   NODE E      │
              (in ~12 ms — did NOT             │ hint: "this   │
               wait for C)                     │ belongs to C" │
                                               └───────┬───────┘
                                                       │ 4. C returns; gossip
                                                       │    tells E within ~10s
                                                       ▼
                                               ┌───────────────┐
                                               │ NODE C healed │
                                               └───────────────┘
```

**Read it like this:** the client's write is durable and acknowledged after 2 of 3 replicas commit — node `C` being dead never blocks anyone. `C`'s missing write is parked as a hint on `E`. Gossip (running independently, every second) eventually tells `E` that `C` is back, and `E` replays the hint. If `E` *also* died before delivering the hint, **anti-entropy** (Merkle tree comparison between `A`/`B` and `C`) is the backstop that finally repairs `C`. Three layers of healing, each covering the previous one's blind spot.

### Diagram 2: The three repair mechanisms and what each one covers

```
  ┌──────────────────┬────────────────────┬─────────────────────┬──────────────────┐
  │   MECHANISM      │  WHEN IT RUNS      │  WHAT IT FIXES      │  BLIND SPOT      │
  ├──────────────────┼────────────────────┼─────────────────────┼──────────────────┤
  │  READ REPAIR     │ on every read that │ hot data — anything │ COLD DATA.       │
  │                  │ sees a conflict    │ being read          │ never repaired.  │
  ├──────────────────┼────────────────────┼─────────────────────┼──────────────────┤
  │  HINTED HANDOFF  │ at write time,     │ writes that missed  │ hint-holder can  │
  │                  │ when a target is   │ a node during a     │ itself die; hints│
  │                  │ known down         │ SHORT outage        │ expire (~3h)     │
  ├──────────────────┼────────────────────┼─────────────────────┼──────────────────┤
  │  ANTI-ENTROPY    │ background /       │ EVERYTHING,         │ costs CPU + I/O; │
  │  (Merkle trees)  │ scheduled repair   │ incl. cold data,    │ run it off-peak  │
  │                  │                    │ long outages        │                  │
  └──────────────────┴────────────────────┴─────────────────────┴──────────────────┘

  Run all three. Each one's blind spot is the next one's job.
```

---

## Real world examples

### 1. Amazon DynamoDB (and the 2007 Dynamo paper)

The Dynamo paper is the origin of everything in this doc — consistent hashing, `N/W/R` quorums, vector clocks, hinted handoff, Merkle-tree anti-entropy, and gossip membership all appear in it. The driving requirement was the shopping cart: **an "add to cart" must never be rejected**, even during a datacenter partition, because a rejected add is lost revenue. They chose an always-writable store and pushed conflict resolution up to the application (the cart merges divergent versions by unioning the items — which is why an item you deleted could famously reappear; Amazon judged a resurrected item strictly better than a lost one).

Today's DynamoDB exposes a simplified version of the dial: every read is either **eventually consistent** (the default — cheaper, and it may return stale data) or **strongly consistent** (`ConsistentRead: true` — roughly a quorum read, costs 2× the read capacity units and has higher latency). That price difference is `W + R > N` showing up on your AWS bill.

### 2. Apache Cassandra

The most direct, most configurable implementation of everything above. You set the replication factor per keyspace (`N`), and then **every single query** carries its own consistency level (`ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`) — the tunable dial made explicit. Gossip runs once per second for membership and ring state. Hinted handoff is on by default with a window (commonly ~3 hours) after which hints are dropped. `nodetool repair` is the operator-triggered anti-entropy pass, and it builds Merkle trees to diff replica ranges. Cassandra is the reference implementation for interview answers: if you can describe Cassandra's replication, you can describe leaderless replication.

### 3. Consul / Redis Cluster — gossip without the quorum store

Both use gossip for **membership and failure detection** while getting their data consistency elsewhere. Consul runs `SWIM`-style gossip (via the `Serf` library) so every agent knows the cluster roster and node health without a registry — but its *data* lives behind a Raft consensus group (a leader!). Redis Cluster's "cluster bus" is a gossip channel over a second port: nodes exchange `PING`/`PONG` carrying rumours about which nodes are `PFAIL` (suspected) and, once enough nodes agree, `FAIL` (confirmed). This is worth knowing because it shows gossip is a **separable building block** — you can adopt it for membership even in a system that isn't leaderless at all.

---

## Trade-offs

| | Leader-based (topic 63) | Leaderless / Dynamo-style (this doc) |
|---|---|---|
| **Write path** | All writes → one primary | Any node, in parallel to N replicas |
| **Node failure** | Failover election; 10–30s write outage | No failover. Just use other nodes. Zero downtime |
| **Consistency** | Strong (from the primary) by default | **Tunable per query** via W and R |
| **Conflicts** | Impossible (one writer orders everything) | **Possible** — you must resolve them (LWW or vector clocks) |
| **Write throughput ceiling** | The primary's capacity | Scales horizontally with nodes |
| **Operational complexity** | Lower — familiar, one source of truth | Higher — repair, tombstones, conflict handling |
| **Best for** | Transactions, joins, "must be correct" (Postgres) | Always-on, high write volume, geo-distributed |

| Choice | You gain | You give up |
|---|---|---|
| **High W** (e.g. 3 of 3) | Fresher data, more durable writes | Write availability (one node down ⇒ blocked) and latency |
| **High R** (e.g. 3 of 3) | Fresher reads | Read availability and latency |
| **W=1, R=1** | Minimum latency, maximum availability | The overlap guarantee — you WILL read stale data |
| **Sloppy quorum** | Writable through a partition | The `W+R>N` freshness guarantee |
| **Read repair only** | Free repair | Cold data rots. You need anti-entropy too |
| **Frequent anti-entropy** | Fast convergence, no rot | CPU + disk I/O + network on live nodes |
| **Aggressive gossip (high fan-out)** | Faster convergence of membership | More background chatter per node |

**Rule of thumb:** Start at **`N=3, W=2, R=2`** — it's the balanced default, it satisfies `W + R > N`, and it survives one node down on both paths. Then move the dial only where a specific workload demands it: `W=1` for firehose ingest that can tolerate loss, `R=1` for read-heavy data that rarely changes, and `LOCAL_QUORUM` the moment you go multi-region. And say out loud what you gave up.

---

## Common interview questions on this topic

### Q1: "Explain the quorum condition `W + R > N`. Why does it work?"
**Hint:** Pigeonhole. If you write to W of N nodes and read from R of N nodes and `W + R > N`, the two subsets cannot be disjoint — they must share at least one node. That node holds the latest write, and the reader picks the newest version among its R responses. **Draw the two overlapping circles.** Then immediately add the honest caveat: overlap gives you *freshness on that key*, not linearizability — concurrent writes and partially-applied writes still bite.

### Q2: "You have N=3. Walk me through W=3/R=1, W=1/R=3, W=2/R=2, and W=1/R=1."
**Hint:** `W=3,R=1`: fast reads, but a single node down blocks all writes — read-optimised and fragile. `W=1,R=3`: fast writes, fragile reads — good for log/metric ingest. `W=2,R=2`: overlap holds AND both paths tolerate one node down — **the default, say this one.** `W=1,R=1`: `1+1=2` is **not** `> 3`, so no overlap — pure eventual consistency, may read stale data, but it's the fastest and most available. Add: higher W ⇒ you wait for the Wth-fastest node ⇒ consistency and latency are the same knob (PACELC's "else" branch).

### Q3: "A node was down for two hours. How does it catch up?"
**Hint:** Three mechanisms, and say why each alone is insufficient. **Hinted handoff** — peers held the writes it missed and replay them when gossip reports it back up (but hints expire, and the hint-holder can itself die). **Read repair** — reads that see conflicting versions push the newest value back to the stale replica (but only for keys that are actually read; cold data rots forever). **Anti-entropy** — a background Merkle-tree comparison finds and fixes everything, including cold data (but it costs CPU and I/O). Real systems run all three because each one's blind spot is another one's job.

### Q4: "How do you compare two replicas holding 50 million keys without sending 50 million keys?"
**Hint:** Merkle tree. Hash each bucket of the key range into a leaf; hash pairs of hashes upward to a single root. Compare **roots first** — if they match, the datasets are provably identical and you're done after exchanging **one hash**. If they differ, descend only into the subtree whose hash differs; every matching subtree is pruned entirely. Finding a few divergent keys costs `O(log n)` hash comparisons instead of `O(n)` data transfer. Mention that Git and Bitcoin use the same structure.

### Q5: "There's no leader — so how does a node even know which other nodes exist, or that one just died?"
**Hint:** Gossip. Every second each node picks a few (fan-out ~3) **random** peers and exchanges its view of the cluster: heartbeat counters per node, higher counter wins on merge (making the merge commutative and idempotent). Info spreads epidemically — ~`4^r` nodes informed after r rounds — so 1000 nodes converge in roughly 10 rounds / 10 seconds. No central registry, no SPOF, constant message load per node regardless of cluster size. The cost: membership is only *eventually* consistent, so for a few seconds nodes can disagree about who's alive. Used by Cassandra, Consul (SWIM), and Redis Cluster.

### Q6: "What's a sloppy quorum, and what does it cost you?"
**Hint:** A *strict* quorum needs W acks from the N nodes that actually own the key. A *sloppy* quorum accepts acks from any N reachable nodes, including stand-ins holding hints for down nodes. It keeps you writable during a partition — but since the stand-ins aren't in the read set, `W + R > N` no longer guarantees the read touches the newest write. **You bought availability and durability, and paid with freshness.**

---

## Practice exercise

### Tune the Dial: A Quorum Simulator

Build a small Node.js simulation (~30 minutes) that makes the quorum condition tangible. No database, no network — just objects in memory.

**What to build:**

1. A `Replica` class with `store(key, {value, version})`, `fetch(key)`, and an `alive` flag you can toggle. Give each replica a random latency (`await sleep(5 + Math.random() * 50)`) so quorum timing is real.
2. A `Coordinator` class taking `{ N, W, R }`. Its `put()` fires at all N replicas but resolves after W acks; its `get()` resolves after R responses and returns the highest version. Have `get()` also perform **read repair** on any stale replica it saw.
3. A `run()` harness that, for each of the four configurations `(W=3,R=1)`, `(W=1,R=3)`, `(W=2,R=2)`, `(W=1,R=1)`:
   - kills **one** replica,
   - does a `put('x', 'v2')`, then immediately a `get('x')`,
   - and records: **did the write succeed? did the read return the fresh value or a stale one? how many ms did each take?**

**What to produce:** a table with one row per configuration and the columns `W+R > N?`, `write succeeded?`, `read was fresh?`, `write ms`, `read ms`. Then write **three sentences** explaining what you observed:
- why `W=3` failed to write with a node down,
- why `W=1, R=1` returned stale data,
- why `W=2, R=2` did neither.

**The stretch goal:** revive the killed replica, run `get('x')` a few times, and show that **read repair** eventually heals it — then add a `coldKey` that nobody ever reads and prove it *stays* stale. That's the demonstration of exactly why anti-entropy has to exist.

---

## Quick reference cheat sheet

- **Leaderless (Dynamo-style)** = no primary; any node accepts writes; the client/coordinator writes to several replicas and reads from several. No leader ⇒ **no failover step**.
- **N / W / R** = replicas per item / acks needed for a write / responses needed for a read.
- **`W + R > N` ⇒ the read set and write set must OVERLAP** ⇒ at least one node you read has the latest write. This is the whole game. Draw the two circles.
- **`N=3, W=2, R=2` is the default answer.** Overlap holds, and both paths survive one node down.
- **`W=1, R=1`** ⇒ `2 > 3` is false ⇒ **no overlap** ⇒ eventual consistency, possible stale reads. Fine for like-counts, fatal for balances.
- **Latency = the Wth-fastest node.** Higher W ⇒ stronger consistency ⇒ higher latency. This IS the PACELC latency/consistency dial.
- **Quorum ≠ linearizability.** Concurrent writes, partially-applied writes, and read/write races still break it.
- **Read repair** — coordinator writes the newest value back to stale replicas after a conflicting read. Free, but **only fixes data that gets read**. Cold data rots.
- **Hinted handoff** — a stand-in node holds a write for a down node and delivers it on recovery. Keeps you writable. Enables **sloppy quorum**, which trades the overlap guarantee for availability.
- **Anti-entropy** — background replica comparison. The only mechanism that fixes **cold** data.
- **Merkle tree** — leaves hash key-range buckets, parents hash their children, root fingerprints the dataset. **Roots match ⇒ identical, in ONE comparison.** Differ ⇒ recurse into only the differing subtree ⇒ `O(log n)`, not `O(n)`.
- **Gossip** — each node tells `f` random peers its view each round; spreads epidemically; ~`O(log N)` rounds (1000 nodes, fan-out 3 ⇒ ~10 rounds ⇒ ~10s). No SPOF, constant per-node load, eventually-consistent membership.
- **Tunable consistency** — Cassandra's `ONE` / `QUORUM` / `LOCAL_QUORUM` / `ALL`, chosen **per query**: `QUORUM` for a payment, `ONE` for a like. That per-request choice is the selling point.
- **`LOCAL_QUORUM`** — quorum within one datacenter ⇒ strong locally, eventual globally, no 90ms trans-ocean hop on the hot path.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [63 — Database Replication](./63-database-replication.md) — leader-based replication; this doc is its leaderless counterpart, so read that first |
| **Next** | [96 — Design a Key-Value Store](./96-hld-key-value-store.md) — where quorums, gossip, Merkle trees and hinted handoff are assembled into one complete system |
| **Related** | [09 — CAP Theorem](./09-cap-theorem.md) — quorum tuning is PACELC's latency-vs-consistency dial made into a config value |
| **Related** | [08 — Consistency Models](./08-consistency-models.md) — precisely what `W+R>N` does and does *not* buy you |
| **Related** | [76 — Eventual Consistency](./76-eventual-consistency.md) — what you're accepting when `W+R ≤ N`, and how repair drives convergence |
