# 96 — Design a Distributed Key-Value Store
## Category: HLD Case Study

---

## What is this?

A **distributed key-value store** is the simplest possible database interface — `put(key, value)`, `get(key)`, `delete(key)` — spread across hundreds of machines so it can hold more data than any single box can fit and keep serving even when machines die.

This is the **capstone** distributed-systems case study. Everything else you've studied converges here. Building a mini **DynamoDB / Cassandra** forces you to combine, in one design:

- **Consistent hashing** ([74 — Consistent Hashing](./74-consistent-hashing.md)) — to decide which machine owns which key.
- **Quorum replication** ([86 — Data Replication Strategies](./86-data-replication-strategies.md)) — to survive machine failure without losing data.
- **The CAP theorem** ([09 — CAP Theorem](./09-cap-theorem.md)) — to make the conscious choice to stay available under partition.
- **Eventual consistency** ([76 — Eventual Consistency](./76-eventual-consistency.md)) — to reconcile the copies that inevitably drift apart.

The API is a lie in its simplicity. `put` and `get` sound like a hash map. The moment the data won't fit on one machine and the machine is allowed to fail, every hard problem in distributed systems shows up at once.

---

## Why does it matter?

Amazon's original **Dynamo** paper (2007) is the single most influential systems paper of its era. It launched DynamoDB, Cassandra, Riak, Voldemort, and the whole "AP database" category. If you understand this one design, you understand the guts of half the databases in production today.

- **It's the interview boss fight.** "Design a distributed key-value store" is asked at every senior/staff loop because it has no shortcuts. You cannot memorize your way through it — you must actually reason about partitions, replicas, and conflicts.
- **It teaches the AP mindset.** Most engineers default to "the database is always consistent." This design forces you to give that up on purpose and get something valuable back: an always-on store that never refuses a write.
- **Every concept is reusable.** The techniques here — LSM-trees, hinted handoff, Merkle trees, vector clocks — reappear in message queues, caches, search engines, and blockchains.

**Interview angle:** the interviewer is watching for whether you *name the CAP tradeoff out loud* and whether you understand *why writes are the hard part*. Candidates who describe an LSM-tree and quorum tuning unprompted are immediately marked as senior.

---

## The core idea — explained simply

### The Coat-Check Analogy

Imagine a coat check at a stadium, not a small theater. One clerk with one rack (a single machine) is fine until the crowd is 80,000 people. Then:

- **One rack can't hold every coat** → you need many racks in many rooms. Something must decide *which room* a coat goes to just from the ticket number. That "which room" function is **partitioning** (consistent hashing).
- **A room can catch fire** → if all copies of a coat were in one burning room, they're gone. So every coat is hung in **three rooms**. That's **replication**.
- **You must never turn a customer away** → even if a room is temporarily locked, you take the coat and hang it in the next open room, leaving a sticky note to move it back later. That's **staying available** (AP) via **sloppy quorum + hinted handoff**.
- **Two clerks might update the same ticket** → you record *who saw what version* so you can tell "this is newer" from "these two happened independently." That's **vector clocks**.

The whole design is: **spread the load, keep redundant copies, never say no, and reconcile the mess afterward.**

---

## Key concepts inside this topic

### 1. Requirements — and the CAP decision

**Functional (the entire API):**

| Operation | Meaning |
|-----------|---------|
| `put(key, value)` | Write/overwrite the value for a key. |
| `get(key)` | Return the value for a key (or the current siblings). |
| `delete(key)` | Remove a key. |

Keys and values are opaque bytes; there are no joins, no secondary indexes, no transactions across keys. That deliberate narrowness is what lets us scale.

**Non-functional (this is where the design lives):**

- **Data too big for one machine.** We assume petabytes — must partition.
- **High availability — always writable.** A write must succeed even during node failures and network partitions. A shopping cart that rejects "add to cart" loses money on every failure.
- **Low latency.** Single-digit-millisecond `get`/`put` at p99.
- **Tunable consistency.** Different callers want different points on the consistency/latency curve — the design must expose a dial.
- **Incremental scale.** Add a node and it starts taking traffic with **zero downtime** and minimal data movement.

**The CAP decision — we choose AP over CP.** (Recall [09 — CAP Theorem](./09-cap-theorem.md): under a network partition you can keep **C**onsistency or **A**vailability, not both.)

We choose **Availability**. The business reason is concrete: an **always-writable cart** that occasionally shows a slightly stale view beats a **perfectly consistent** cart that returns "Service Unavailable" whenever two data centers can't talk. Amazon measured that every rejected write is lost revenue; a rare merge conflict is cheap to reconcile. So: **accept writes always, reconcile later.** Everything downstream (quorums, hinted handoff, vector clocks) exists to make that choice safe.

---

### 2. Single-node design first — the storage engine

Before distributing anything, make *one* node store data well. This is the part most candidates skip and most interviewers love.

**The naive answer: a hash map.** `Map<key, value>` in RAM is O(1) and trivial. But it is **volatile** (a crash loses everything) and **RAM-limited** (can't exceed memory). We need durability and the ability to exceed RAM. So we go to disk — and now the enemy is **random disk writes**, which are orders of magnitude slower than sequential ones. **Writes are the bottleneck.**

Real KV stores (Cassandra, RocksDB, LevelDB, DynamoDB's storage layer) solve this with the **LSM-tree (Log-Structured Merge-tree) + SSTables**. The trick: never do random writes. Turn every write into a sequential append.

**The write path:**

1. **Write-ahead log (WAL).** Every write is first appended to an on-disk commit log — a pure sequential append. This is what makes it **crash-safe**: after a crash we replay the WAL to rebuild in-memory state.
2. **Memtable.** The write is then applied to an in-memory sorted structure (a balanced tree / skip list) called the **memtable**. Writes are fast because they hit RAM; the WAL alone guarantees durability.
3. **Flush to SSTable.** When the memtable exceeds a size threshold, it is flushed to disk as an immutable, **sorted** file called an **SSTable** (Sorted String Table). Because the memtable is already sorted, this flush is one big **sequential write** — the whole point. The old memtable is frozen and a fresh one takes new writes.

Because SSTables are immutable and sorted, we accumulate many of them over time, each a snapshot of some window of writes.

**The read path:**

1. Check the **memtable** (newest data). Hit → return.
2. Otherwise check **SSTables newest-first** (a key can exist in several; the newest wins).
3. To avoid touching every SSTable on disk, each SSTable has a **Bloom filter** in memory.

**Bloom filter (explained properly).** A Bloom filter is a **probabilistic bitset** that answers one question: "is this key *possibly* in this SSTable?" You hash the key with *k* hash functions and set those *k* bits on insert; on lookup you check whether all *k* bits are set.

- **False negatives are impossible.** If any of the *k* bits is 0, the key was *definitely never inserted* → skip this SSTable entirely, zero disk I/O.
- **False positives are possible.** All *k* bits might be set by *other* keys' hashes colliding, so "all bits set" means only **"maybe present"** — you then read the file to be sure, and it might not be there.

So the filter's two answers are exactly **"definitely not present"** or **"maybe present."** It never wrongly says "not present." This lets a read skip the vast majority of SSTables with a cheap in-memory check, turning a potential N-file disk scan into usually one file read.

**Compaction.** Left alone, SSTables pile up: reads get slower (more files to check) and deleted/overwritten keys waste space. **Compaction** is a background job that **merges** several sorted SSTables into one, and because they're all sorted it's a linear merge. During the merge it keeps only the **newest** version of each key and **discards overwritten values and deleted keys (tombstones past their grace period)**. This reclaims space and keeps read amplification bounded.

**LSM vs B-tree (contrast with [62]).** A traditional relational database uses a **B-tree**: it **updates pages in place**, which means a write is a **random disk write** (seek to the page, modify, write back). B-trees are therefore **read-optimized** — a read is a clean O(log n) tree descent. An **LSM-tree is append-only** — writes are sequential and cheap, but a read may consult several SSTables (mitigated by Bloom filters). So:

| | B-tree | LSM-tree |
|--|--------|----------|
| Write | Random, in-place, slower | Sequential append, fast |
| Read | One tree descent, fast | Memtable + several SSTables |
| Space | Fragmentation | Compaction reclaims |
| Best for | Read-heavy, transactional | Write-heavy, high ingest |

**Why Cassandra chose LSM:** it targets write-heavy workloads (time series, event logs, feeds) where sequential-write throughput dominates. This LSM-vs-B-tree answer is a **big interview differentiator** — it shows you understand the physics of disks, not just APIs.

---

### 3. Partitioning — consistent hashing

The data won't fit on one node, so we split keys across nodes. The naive scheme `node = hash(key) % N` is a trap: change **N** (add or remove a node) and *almost every key* remaps → a full data reshuffle and cache stampede.

Instead we use **consistent hashing** ([74 — Consistent Hashing](./74-consistent-hashing.md)). Hash both keys and nodes onto a fixed ring (0 … 2^128−1). A key is owned by the **first node clockwise** from the key's position. Adding or removing a node only moves the keys between that node and its neighbor — about **K/N keys** (K keys, N nodes) — instead of nearly all of them.

**Virtual nodes (vnodes).** A physical node claims **many** points on the ring (say 256), not one. This (a) smooths out load imbalance from random placement, and (b) spreads a departing node's data across *many* remaining nodes instead of dumping it all on one successor. Cassandra uses vnodes exactly this way.

```
                 hash ring (0 .. 2^128-1), clockwise
                          ┌─────────────┐
                     A1 ● │             │ ● B1
                        ╱                 ╲
                  C2 ●                       ● A2
                    │      key K hashes       │
                    │      here ─────►  ● (K) │
                  B2 ●                       ● C1
                        ╲                 ╱
                     A3 ● │             │ ● B3
                          └─────────────┘
   K's owner = first vnode clockwise from K = A2  (physical node A)
   A1/A2/A3 are vnodes of the SAME physical node A, scattered on the ring.
```

---

### 4. Replication — N copies clockwise

Owning a key on one node isn't durable — that node can die. So each key is **replicated to the next N nodes clockwise** on the ring (N is the replication factor, commonly 3). The first node is the **coordinator/owner**; the next N−1 are replicas. This ordered list is the key's **preference list**.

**Critical detail — skip vnodes on the same physical node.** Because a physical node owns many vnodes, the "next N clockwise" vnodes might belong to the *same physical machine*. If you naively took them, all 3 replicas could land on one box — and a single machine failure would lose all copies. So when walking clockwise to build the preference list, you **skip vnodes whose physical node is already in the list**, guaranteeing N *distinct physical nodes*.

**Cross-DC placement.** For disaster tolerance, the preference list is often built to span **data centers / racks** — e.g. N=3 as 2 replicas in DC1 and 1 in DC2 (Cassandra's `NetworkTopologyStrategy`). Now an entire DC can burn down and the key survives.

---

### 5. Consistency — quorum (W + R > N)

Replication raises the question: how many replicas must respond before we call a write or read *successful*? This is the **quorum** dial ([86 — Data Replication Strategies](./86-data-replication-strategies.md)).

- **N** = number of replicas.
- **W** = replicas that must **ack a write** before it's "successful."
- **R** = replicas that must **respond to a read** before we answer.

The magic inequality: **if `W + R > N`, the write set and read set must overlap by at least one node**, so a read is guaranteed to see at least one replica that has the latest write — **strong consistency** on that key. If `W + R ≤ N`, they may not overlap → reads can miss recent writes → **eventual consistency**.

This is a **per-request dial**, not a global setting — the same store serves both.

| Config (N=3) | Guarantee | Good for |
|--------------|-----------|----------|
| W=1, R=1 | `W+R=2 ≤ N` → eventual, fastest, most available | Metrics, logs, "like" counters |
| W=2, R=2 | `W+R=4 > N` → strong; survives 1 node down | General default (DynamoDB-ish) |
| W=3, R=1 | Fast reads, slow/fragile writes (all replicas must ack) | Read-heavy, rarely written config |
| W=1, R=3 | Fast writes, slow reads | Write-heavy ingest, read on demand |

Lower **W** = more available writes but weaker durability; higher **R+W** = stronger consistency at the cost of latency and availability. Because we chose **AP**, the common default is **W=2, R=2, N=3** (strong when all is well) with the ability to drop to **W=1** during failures so writes never stop.

---

### 6. Handling failures

Nodes fail, networks partition, clocks drift. Four mechanisms keep an AP store correct.

**(a) Temporary failure → sloppy quorum + hinted handoff.** Suppose W=2 but one of the 3 preference-list nodes is down. A *strict* quorum would reject the write — unacceptable for an always-writable store. Instead we use a **sloppy quorum**: send the write to the next healthy node *further* clockwise, outside the normal preference list. That stand-in stores the data **with a hint** ("this really belongs to node C"). When C recovers, the stand-in performs **hinted handoff** — it ships the data back to C and deletes its copy. The write never blocked, and durability is eventually restored.

**(b) Permanent failure → anti-entropy with Merkle trees.** If a node is dead for hours, hinted handoff isn't enough; replicas drift. To resync two replicas without shipping the *entire* dataset, each builds a **Merkle tree** — a tree of hashes where leaves hash key ranges and parents hash their children. Two nodes compare **root hashes**; if equal, they're identical — done, zero data transferred. If different, they descend only into the subtrees whose hashes differ, pinpointing exactly which key ranges diverge and exchanging **only those**. This makes reconciliation cost proportional to the *difference*, not the dataset.

**(c) Failure detection → gossip.** There's no central coordinator (that would be a SPOF and a CP bottleneck). Nodes run a **gossip protocol**: each node periodically picks a few random peers and exchanges "who's alive, who's dead, ring state" heartbeats. Information spreads epidemically (log N rounds). This is how nodes learn membership and mark peers down — decentralized, partition-tolerant.

**(d) Conflicts → vector clocks.** With W=1 and concurrent writes, two replicas can hold *different* values for the same key. We need to tell **causally later** (a true update — keep the newer) apart from **concurrent** (two independent writes — a real conflict). A **vector clock** is a map `{nodeId: counter}` attached to each value; a node bumps its own counter when it writes.

Given two versions, compare element-wise:

- If every counter in A ≤ the matching counter in B (and at least one is strictly less) → **A happened-before B** → B is causally newer → **keep B, discard A.**
- If neither dominates (each has some counter the other doesn't) → **concurrent** → a genuine conflict.

**Worked two-node example** (nodes Sx, Sy; shopping cart key):

| Step | Event | Vector clock | Value |
|------|-------|--------------|-------|
| 1 | Client writes via Sx | `{Sx:1}` | `[milk]` |
| 2 | Client updates via Sx | `{Sx:2}` | `[milk, eggs]` |
| 3 | Partition. Client A writes via Sx | `{Sx:3}` | `[milk, eggs, bread]` |
| 4 | Partition. Client B writes via Sy (from step-2 version) | `{Sx:2, Sy:1}` | `[milk, eggs, ham]` |
| 5 | Read compares `{Sx:3}` vs `{Sx:2,Sy:1}` | neither dominates | **concurrent → 2 siblings** |

Step 3's clock has `Sx:3` but no `Sy`; step 4's has `Sy:1` but only `Sx:2`. Neither is ≥ the other → **concurrent**. Dynamo's choice: **return both siblings** to the application and let *it* merge them semantically. For a cart, the app merges by **union** → `[milk, eggs, bread, ham]`.

**The famous cart wart:** union-merge means a **deleted item can reappear** — if the delete happened on one side and a concurrent write on the other, the union resurrects it. Amazon accepted this: an occasionally-reappearing item is a far cheaper bug than a cart that rejects writes.

**Contrast — last-write-wins (LWW).** The simpler alternative: attach a wall-clock timestamp and, on conflict, **keep the one with the higher timestamp**, silently dropping the other. LWW is trivial and needs no sibling logic — but it **silently loses data** (one concurrent write vanishes) and **depends on synchronized clocks** (skew between machines picks the "wrong" winner). Cassandra defaults to LWW (needs NTP); Dynamo chose vector clocks + siblings to never silently lose a write. Know both and when each is acceptable.

---

### 7. Read and write paths, end to end

Any node can be the **coordinator** for any request — the client hits any node (or a smart client picks one on the preference list). The coordinator hashes the key, finds the N replicas, and orchestrates.

**Write path:**

```
  client
    │  put(key, value)
    ▼
 ┌───────────┐   1. hash(key) → ring position
 │coordinator│   2. build preference list → replicas [C, D, E]
 │  (node A) │   3. attach/advance vector clock
 └───────────┘   4. send write to ALL N replicas in parallel
    │  ├──────────────► C  (WAL → memtable) ──► ack
    │  ├──────────────► D  (WAL → memtable) ──► ack
    │  └──────────────► E  (down) ──► sloppy quorum → F stores hint
    ▼
 wait for W acks (e.g. 2). Got C+D → SUCCESS to client.
 (E gets the data later via hinted handoff.)
```

**Read path:**

```
  client
    │  get(key)
    ▼
 ┌───────────┐   1. hash(key) → replicas [C, D, E]
 │coordinator│   2. send read to all (or enough) replicas
 │  (node A) │   3. wait for R responses (e.g. 2)
 └───────────┘
    │  ├──────────◄ C: value v2, clock {Sx:3}
    │  └──────────◄ D: value v1, clock {Sx:2}
    ▼
 4. compare vector clocks:
      - one dominates  → return the newest value
      - concurrent     → return siblings for the app to merge
 5. READ REPAIR: D was stale → coordinator pushes v2 back to D
    (async), so replicas converge over time.
```

**Read repair** is the quiet workhorse: every read that notices a lagging replica fixes it, so hot keys self-heal continuously and anti-entropy only has to handle the cold, rarely-read ones.

---

### 8. Deep dives / edge cases

**Adding a node (scale-out with zero downtime).** The new node picks its vnode positions on the ring. For each position it inserts between an existing vnode and its predecessor, it **takes over the key range** immediately before it. The current owners of those ranges **stream** the relevant SSTables to the new node in the background; once streaming completes, ring metadata flips ownership and the old owners drop their now-redundant copies. Because of vnodes, the incoming load is pulled from **many** existing nodes at once, so no single node is hammered. Only ~K/N keys move.

**The hot-key problem.** Consistent hashing balances keys *count*, not *traffic*. One celebrity key (a viral tweet, a flash-sale SKU) can pin one partition's replicas at 100% while the rest idle — a single key can't be split by hashing. Mitigations: (a) **cache** the hot key in front of the store; (b) **key-splitting** — write to `key:0..key:M` and read/aggregate across them to spread the load; (c) let clients **read from any replica (R=1)** to fan reads across all N copies.

**Tombstones — why deletes are hard.** In an append-only LSM store you **cannot** erase a key in place; the value still sits in old immutable SSTables. So `delete` writes a **tombstone** — a special marker meaning "this key is deleted," with its own timestamp/clock. Reads see the tombstone (newest) and return "not found." Compaction eventually drops the tombstone *and* the shadowed values.

The tricky part is **when** to drop the tombstone. If you purge it too early and a replica that was **down during the delete** comes back, that replica still has the *old value* and no tombstone to tell it otherwise — anti-entropy then treats the old value as "missing on the other side" and **resurrects the deleted data** across the cluster (a "zombie"). To prevent this, tombstones are kept for a **grace period** (`gc_grace_seconds`, default 10 days in Cassandra) that must exceed the max time a node can be down before repair, guaranteeing every replica has learned of the delete before the marker disappears. Deleting a **tombstone**, not a row, and getting the grace window right, is the subtle point interviewers probe.

**Ring-state / metadata management.** Every node must agree on the ring (who owns what, who's up). Options: a **gossip-propagated** decentralized map (Cassandra — no SPOF, eventually consistent membership), or a small **strongly-consistent coordination service** (ZooKeeper/etcd, as some systems use) holding authoritative ring state. The AP purist prefers gossip; the tradeoff is that membership changes take a few gossip rounds to fully propagate, during which two nodes may briefly disagree on ownership (resolved as they converge).

---

## Visual / Diagram description

### Full architecture

```
                    ┌──────────────┐
                    │    Client    │  put/get/delete(key)
                    └──────┬───────┘
                           │ (hits any node; smart client picks
                           │  a coordinator from preference list)
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │ Node A  │  │ Node B  │  │ Node C  │  ... N nodes on the ring
        │(coord)  │  │         │  │         │
        │─────────│  │─────────│  │─────────│
        │ Storage engine per node:          │
        │   WAL → Memtable → SSTables        │
        │           ▲            + Bloom      │
        │           │            + Compaction │
        │  Vector clocks on values            │
        └─────────┘  └─────────┘  └─────────┘
              ▲            ▲            ▲
              └──── gossip (membership, ring state) ────┘
              └──── hinted handoff / read repair ───────┘
              └──── Merkle-tree anti-entropy (background) ┘
```

### The ring (N=3 replication)

```
        Position of key K on the ring, replicated to next
        3 DISTINCT physical nodes clockwise (skipping same-node vnodes):

                     ● N1
              ● N7        ● N2   ◄── K lands here
           ●               ● N3  ─┐
          N6                       ├─ preference list for K:
           ●               ● N4   ─┘   N2, N3, N4  (coordinator + 2 replicas)
              ● N5
                                  W acks needed to write, R responses to read,
                                  W + R > N  ⇒  strong consistency for K.
```

---

## The Node.js implementation

Two of the load-bearing components: the per-SSTable **Bloom filter** and the **vector clock** used for conflict detection.

```javascript
// ── Bloom filter ────────────────────────────────────────────────
// Probabilistic set membership per SSTable.
//  - "definitely not present" (any bit 0)  → skip the file, no disk I/O
//  - "maybe present"          (all bits 1) → read the file to confirm
// False negatives are IMPOSSIBLE; false positives are possible.
class BloomFilter {
  constructor(sizeBits = 1 << 16, numHashes = 4) {
    this.size = sizeBits;
    this.k = numHashes;
    // Bit-packed: one Uint8 holds 8 bits.
    this.bits = new Uint8Array(Math.ceil(sizeBits / 8));
  }

  // k independent hashes via double hashing: h_i = h1 + i*h2  (mod size)
  _hashes(key) {
    let h1 = 2166136261;        // FNV-1a
    let h2 = 5381;              // djb2
    for (let i = 0; i < key.length; i++) {
      const c = key.charCodeAt(i);
      h1 = Math.imul(h1 ^ c, 16777619) >>> 0;
      h2 = ((h2 << 5) + h2 + c) >>> 0;
    }
    const out = [];
    for (let i = 0; i < this.k; i++) {
      out.push(((h1 + i * h2) >>> 0) % this.size);
    }
    return out;
  }

  add(key) {
    for (const bit of this._hashes(key)) {
      this.bits[bit >> 3] |= (1 << (bit & 7));   // set the bit
    }
  }

  // false → key is DEFINITELY not present (skip SSTable)
  // true  → key is MAYBE present (must read SSTable to be sure)
  mightContain(key) {
    for (const bit of this._hashes(key)) {
      if ((this.bits[bit >> 3] & (1 << (bit & 7))) === 0) return false;
    }
    return true;
  }
}

// Read path uses it to avoid disk I/O:
function readFromSSTables(key, sstablesNewestFirst) {
  for (const sst of sstablesNewestFirst) {
    if (!sst.bloom.mightContain(key)) continue; // skip: definitely absent
    const hit = sst.lookupOnDisk(key);          // only now touch disk
    if (hit !== undefined) return hit;           // newest wins
  }
  return undefined;
}

// ── Vector clock ────────────────────────────────────────────────
// Attached to every value. Detects causal order vs. concurrency.
class VectorClock {
  constructor(clock = {}) {
    this.clock = { ...clock };   // { nodeId: counter }
  }

  // A node bumps its own counter each time it coordinates a write.
  increment(nodeId) {
    this.clock[nodeId] = (this.clock[nodeId] || 0) + 1;
    return this;
  }

  // Compare two clocks:
  //  'before'     → this happened-before other (other is newer, keep other)
  //  'after'      → this happened-after  other (this is newer, keep this)
  //  'concurrent' → conflict: return both as siblings
  //  'equal'      → identical version
  compare(other) {
    const ids = new Set([...Object.keys(this.clock), ...Object.keys(other.clock)]);
    let lt = false, gt = false;
    for (const id of ids) {
      const a = this.clock[id] || 0;
      const b = other.clock[id] || 0;
      if (a < b) lt = true;
      if (a > b) gt = true;
    }
    if (lt && gt) return 'concurrent';
    if (lt) return 'before';
    if (gt) return 'after';
    return 'equal';
  }

  // Merge for a resolved value (element-wise max) — used after the
  // application reconciles siblings.
  static merge(a, b) {
    const merged = { ...a.clock };
    for (const [id, c] of Object.entries(b.clock)) {
      merged[id] = Math.max(merged[id] || 0, c);
    }
    return new VectorClock(merged);
  }
}

// Coordinator reconciling R read responses:
function reconcile(responses) {
  // responses: [{ value, clock: VectorClock }]
  let survivors = [];
  for (const r of responses) {
    let dominated = false;
    survivors = survivors.filter((s) => {
      const cmp = s.clock.compare(r.clock);
      if (cmp === 'before') return false;        // s is older than r → drop s
      if (cmp === 'after' || cmp === 'equal') dominated = true; // r is older/equal
      return true;                                // concurrent → keep s
    });
    if (!dominated) survivors.push(r);            // r is newest or concurrent
  }
  // 1 survivor → return it. >1 → siblings; app merges (e.g. cart union).
  return survivors;
}
```

---

## Real world examples

### 1. Amazon DynamoDB
Direct descendant of the Dynamo paper. Managed AP store with **tunable consistency** — the SDK's `ConsistentRead` flag flips a `get` between eventual (cheap, default) and strongly consistent (a quorum read). Under the hood: consistent hashing, replication across Availability Zones, and an LSM storage engine.

### 2. Apache Cassandra
The open-source workhorse. Uses **vnodes**, **gossip** for membership, an **LSM-tree** (memtable + SSTables + compaction + Bloom filters + WAL), per-query **consistency levels** (`ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`), **Merkle-tree repair** (`nodetool repair`), **hinted handoff**, and **LWW** conflict resolution (so it needs NTP-synced clocks). Almost every concept in this doc maps 1:1 to a Cassandra feature.

### 3. Riak
The closest to a literal Dynamo implementation — it kept **vector clocks and sibling resolution** as a first-class feature, exposing conflicts to the application exactly as the paper described.

---

## Trade-offs

- **AP vs CP.** We picked **AP**: always writable, reconcile later. The cost is that a `get` can return stale data or siblings, and the app must sometimes merge. A CP store (e.g. a Spanner-style system) never shows you stale data but refuses writes during partitions. Choose per business need — carts and feeds want AP; bank ledgers want CP.
- **Consistency vs latency (the quorum dial).** Higher `W+R` = stronger consistency but more nodes to wait on = higher latency and lower availability. It's a per-request choice, not one setting.
- **LSM vs B-tree.** LSM buys fast sequential writes at the price of **read amplification** (multiple SSTables) and **compaction** (background CPU/IO). B-trees give fast reads but slow random writes. Match to workload.
- **Vector clocks vs LWW.** Vector clocks never silently lose a write but push merge complexity onto the app and grow with the number of writers. LWW is dead simple but silently drops concurrent writes and trusts wall clocks.
- **Vnodes.** Smooth load and fast rebalancing, at the cost of more ring metadata and more streaming connections during scale events.
- **Decentralized (gossip) vs coordinated (ZooKeeper) membership.** Gossip = no SPOF but eventually-consistent membership; a coordination service = crisp state but a CP dependency to operate and scale.

---

## Common interview questions on this topic

### Q1: "Why an LSM-tree instead of a B-tree for a write-heavy KV store?"
Random disk writes are the bottleneck. LSM turns every write into a sequential append (WAL + memtable flush to sorted SSTables), which is orders of magnitude faster than a B-tree's in-place random page updates. The cost is read amplification, mitigated by Bloom filters and compaction.

### Q2: "How do you keep the store available during a network partition?"
Choose AP. Use **sloppy quorum + hinted handoff** so writes land on a stand-in node when a preference-list node is unreachable, and reconcile with **read repair** and **Merkle-tree anti-entropy** once the partition heals. `W=1` lets writes always succeed.

### Q3: "Two clients update the same key concurrently on different nodes. What happens?"
Each value carries a **vector clock**. On read, the coordinator compares clocks: if one causally dominates, it wins; if they're **concurrent**, the store returns **siblings** and the app merges (e.g. cart union). LWW is the simpler alternative but silently drops one write and needs synced clocks.

### Q4: "Why is `W + R > N` the magic rule?"
It forces the set of nodes acknowledging a write and the set answering a read to **overlap by at least one node**, so every read sees at least one replica with the latest write → strong consistency for that key.

### Q5: "How does deleting a key work, and why is it tricky?"
You write a **tombstone**, not an erase (SSTables are immutable). Tombstones must survive a **grace period** longer than any node's max downtime; purge one too early and a returning stale replica **resurrects** the deleted data via anti-entropy.

### Q6: "You add a node to a hot cluster. Walk me through it with zero downtime."
The node claims vnode positions, takes over the key ranges just before each, and existing owners **stream** those SSTables to it in the background. Ring metadata flips ownership on completion. Because of vnodes, load is pulled from many nodes; only ~K/N keys move; no downtime.

---

## Practice exercise

### Design the storage + replication for a social feed store

You're storing a 100M-user social app's per-user timeline entries in a distributed KV store. Requirements: writes must never fail (posting is sacred), a slightly stale read is fine, and one viral account gets 1000× the traffic.

1. Pick **N, W, R** and justify against "writes never fail" and "stale reads OK." (Hint: N=3, W=1, R=1 or W=2/R=1.)
2. Explain the storage engine you'd choose and why (write-heavy!).
3. The viral account is a **hot key**. Give two concrete mitigations.
4. A data center goes offline for 3 hours, then rejoins. Describe, step by step, how the store reconciles: hinted handoff vs. Merkle-tree repair, and how you'd tune `gc_grace_seconds` so nothing zombies back.
5. Would you use vector clocks or LWW for feed entries? Defend it.

Sketch the ring, the preference list for one user's key, and the write path with your chosen W.

---

## Quick reference cheat sheet

- **API is three calls** — `put`, `get`, `delete`. The simplicity hides every distributed-systems hard problem.
- **Choose AP.** Always writable > perfectly consistent for carts, feeds, sessions. Say it out loud in the interview.
- **Storage = LSM-tree.** WAL (crash safety) → memtable (RAM, sorted) → flush to immutable sorted **SSTables** (sequential write). Reads: memtable then SSTables newest-first, skipped via **Bloom filters**. **Compaction** merges and drops dead keys.
- **Bloom filter** = "definitely not present" or "maybe present." No false negatives.
- **LSM = write-optimized (append-only); B-tree = read-optimized (in-place random writes).** Cassandra picked LSM for write throughput.
- **Partition with consistent hashing + vnodes.** Adding a node moves only ~K/N keys.
- **Replicate to next N distinct physical nodes clockwise** (skip same-node vnodes). Span DCs for disaster tolerance.
- **Quorum: `W + R > N` ⇒ strong consistency.** Per-request dial. Default N=3, W=2, R=2.
- **Temporary failure → sloppy quorum + hinted handoff.** Permanent → **Merkle-tree anti-entropy**. Detection → **gossip**. Conflicts → **vector clocks + siblings** (or LWW, which silently loses data).
- **Read repair** fixes stale replicas on every read.
- **Deletes write tombstones**, kept for a grace period longer than max node downtime, or deleted data resurrects.
- **Hot keys don't shard away** — cache them or split the key.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Foundation** | [74 — Consistent Hashing](./74-consistent-hashing.md) — the ring, vnodes, and ~K/N key movement that make partitioning possible |
| **Foundation** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — quorums, `W + R > N`, and the replication machinery this design tunes |
| **Foundation** | [09 — CAP Theorem](./09-cap-theorem.md) — the AP-vs-CP choice at the heart of this whole design |
| **Foundation** | [76 — Eventual Consistency](./76-eventual-consistency.md) — read repair, anti-entropy, and reconciling replicas that drift |
| **Related** | [105 — HLD Distributed Cache](./105-hld-distributed-cache.md) — same sharding/replication ideas applied to an in-memory cache, and the layer that fronts hot keys |
