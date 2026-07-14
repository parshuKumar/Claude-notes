# 74 — Consistent Hashing — The Key to Distributed Systems
## Category: HLD Components

---

## What is this?

Consistent hashing is a way of deciding **which server owns which piece of data** so that when you add or remove a server, *almost nothing moves*. Instead of a formula that depends on the number of servers (which changes every time you scale), you map both servers and keys onto an imaginary **circle**, and each key belongs to the first server you meet walking clockwise from it.

The everyday analogy: imagine a circular table of people, and a pile of letters scattered around the edge. Each letter is delivered to the first person clockwise from where it landed. If one person leaves the table, only *their* letters need re-delivering — everyone else's mail is untouched. That's the whole trick.

---

## Why does it matter?

Because the naive alternative — `server = hash(key) % numberOfServers` — has a failure mode that can **take your entire company down during a routine capacity increase**.

The moment you change `numberOfServers`, the modulo changes for nearly every key. Every cached value is suddenly looked up on the wrong box, every lookup is a miss, and 100% of your read traffic — traffic your cache was absorbing so your database never saw it — slams into the database at once. Databases sized for a 5% miss rate do not survive a 100% miss rate. This has caused real, headline-grade outages.

**Interview angle:** This is one of the most-asked HLD component questions. It shows up as its own question ("explain consistent hashing"), and as a required sub-answer inside *Design a Distributed Cache*, *Design a Key-Value Store*, *Design a CDN*, and any sharding discussion. Interviewers love it because it has a crisp problem, a crisp solution, and a crisp second-order problem (uneven load) with a crisp second-order fix (virtual nodes). If you can walk all four steps, you look senior.

**Real-work angle:** You will meet it inside Cassandra, DynamoDB, Riak, Memcached client libraries, CDN edge selection, sticky-session load balancers, and any time you write a sharding layer yourself. Knowing it means you can add capacity to a live cache tier without a cold-start stampede.

---

## The core idea — explained simply

### The Cloakroom Analogy

You run the cloakroom at a huge venue. Guests hand you coats; you must remember which of your 4 coat racks each coat went on, using nothing but the guest's ticket number.

**The naive system:** `rack = ticketNumber % 4`. Ticket 1234567 → rack 3. Simple, fast, perfectly balanced. It works beautifully — until the venue gets busier and you wheel in a **5th rack**.

Now the rule is `ticketNumber % 5`. Ticket 1234567 → rack 2. But the coat is physically hanging on rack 3. Every single guest whose ticket's remainder changed now gets sent to the wrong rack, finds nothing, and you have to go hunt through all 5 racks by hand. That "hunt by hand" is your database.

**The consistent-hashing system:** Instead of numbering racks 0–4, you arrange the racks **around the wall of a circular room** at fixed positions. A coat with ticket number T is hung at position T on the wall, and then walked clockwise to the first rack it reaches. When you wheel in a 5th rack, you park it somewhere on the wall. It only takes over the coats sitting on the arc **between it and the rack behind it**. Every other coat in the room stays exactly where it is.

**Mapping the analogy back:**

| Cloakroom | Distributed system |
|-----------|-------------------|
| Circular room wall | The **hash ring** — the output space of the hash function, `0 … 2³²-1`, bent into a circle |
| Coat rack | A **node** (cache server, DB shard, storage node) |
| Rack's position on the wall | `hash(serverId)` — where the server lands on the ring |
| Guest's ticket number | `hash(key)` — where the key lands on the ring |
| "Walk clockwise to the first rack" | The **lookup rule**: successor node owns the key |
| Wheeling in a new rack | **Scaling out** — it steals only the arc behind it |
| A rack collapsing | **Node failure** — its arc is inherited by the next rack clockwise |
| Splitting one big rack into 150 small hooks scattered around the room | **Virtual nodes** — the fix for uneven load |

The single sentence to remember: **modulo hashing couples every key to the *count* of servers; consistent hashing couples each key only to its *neighbouring* server.**

---

## Key concepts inside this topic

### 1. The disaster: what naive modulo hashing actually does

You have 4 cache servers, `S0 S1 S2 S3`, and you route with `server = hash(key) % 4`. Let's take 10 real keys and do the arithmetic explicitly. (The hash column is just the output of your hash function — treat these as given.)

| Key | hash(key) | `% 4` → server | `% 5` → server | Moved? |
|-----|-----------|----------------|----------------|--------|
| `user:1`  | 1234567 | **3** | 2 | MOVED |
| `user:2`  | 2345678 | **2** | 3 | MOVED |
| `cart:9`  | 3456789 | **1** | 4 | MOVED |
| `sess:a`  | 4567890 | **2** | 0 | MOVED |
| `post:7`  | 5678901 | **1** | 1 | stays |
| `img:42`  | 6789012 | **0** | 2 | MOVED |
| `vid:11`  | 7890123 | **3** | 3 | stays |
| `doc:88`  | 8901234 | **2** | 4 | MOVED |
| `key:99`  | 9012345 | **1** | 0 | MOVED |
| `tag:13`  | 1023456 | **0** | 1 | MOVED |

**8 out of 10 keys moved. 80%.**

That is not bad luck with a small sample — it is the *expected* result. When you go from `N` servers to `N+1`, a key keeps its home only if `hash(key) % N === hash(key) % (N+1)`, which happens for roughly **1/(N+1)** of keys. So the fraction that moves is:

```
fraction remapped ≈ N / (N+1)

4 → 5 servers:    4/5  = 80%   of all keys move
9 → 10 servers:   9/10 = 90%   of all keys move
99 → 100 servers: 99/100 = 99% of all keys move
```

Note the perverse property: **the bigger and more successful your cluster, the worse the damage.** At 100 servers, adding one more invalidates 99% of your cache.

**Now walk through the consequence, because this is the part interviewers want to hear:**

1. You add server S4 at 10:00. Config rolls out.
2. 80% of cache lookups now hash to a server that has never seen that key → **cache miss**.
3. Say you were serving 100,000 reads/sec at a 95% hit rate. The DB was comfortably absorbing 5,000 reads/sec.
4. Post-deploy, the miss rate jumps toward ~80%. The DB now sees ~80,000 reads/sec — a **16× spike, instantly**.
5. The DB's connection pool saturates. Query latency climbs from 2 ms to 2,000 ms. Requests queue. App servers hold connections open and run out of threads.
6. Retries pile on top. This is a **cache stampede** (also called a thundering herd), and it usually ends with the database on the floor and a very long incident review.

And the truly nasty part: **the exact same catastrophe happens when a server DIES**, unplanned, at 3 a.m. Going 4 → 3 servers remaps ~3/4 of the keyspace. You do not get to choose when this happens.

Consistent hashing exists to make step 2 read "**~20% of keys move**" instead of "~80%", and to make the server-death case "**~25% move**" instead of "~75%".

### 2. The hash ring

Take your hash function's output space. For a 32-bit hash, that's the integers `0` to `2³² - 1` (that's 4,294,967,295 — about 4.3 billion slots). Now **bend that line into a circle** so that the position after `2³² - 1` wraps back around to `0`.

Two placements happen on this circle:

- **Hash each SERVER onto the ring.** Position = `hash("10.0.0.1:11211")`. The server ID can be an IP, a hostname, a node name — anything stable.
- **Hash each KEY onto the ring.** Position = `hash("user:1")`.

Then one rule, and one rule only:

> **To find a key's server, start at the key's position and walk CLOCKWISE until you hit the first server. That server owns the key.** If you fall off the end of the ring, wrap around to position 0 and keep walking.

That's it. That "first server clockwise" is called the key's **successor node**.

Here is the ring with 4 servers. The numbers around the outside are ring positions (I'm showing them as percentages of `2³²` to keep the diagram readable — position 0 = top, and clockwise increases):

```
                              0 / 2^32  (wrap point)
                                   │
                    ┌──────────────┴──────────────┐
              ┌─────┤          RING               ├─────┐
         ╱────┘     └──────────────┬──────────────┘     └────╲
       ╱                           │                          ╲
     ╱                        ● S0 @ 5%                        ╲
    │                              ╲                            │
    │   ○ key "tag:13" @ 92%         ╲                          │
    │            ▲                     ╲                        │
    │             ╲                      ╲                      │
    │              ╲ walks clockwise      ╲                     │
    │               ╲ past 100% → 0% → S0  ╲                    │
    │                                       ╲                   │
    │                                        ● S1 @ 30%         │
    │                                          ▲                │
    │                                         ╱                 │
    │  ● S3 @ 78%                            ╱                  │
    │        ▲                              ╱                   │
    │         ╲                   ○ key "user:1" @ 22%          │
    │          ╲                                                │
    │           ○ key "img:42" @ 61%                            │
    │                    ╲                                      │
     ╲                    ╲──────────▶ ● S2 @ 65%              ╱
       ╲                                                      ╱
         ╲────┐                                        ┌────╱
              └────────────────────────────────────────┘

  LEGEND:   ●  = server (node) hashed onto the ring
            ○  = key hashed onto the ring
            ▲/▶ = direction of the clockwise walk to find the owner
```

Read the four lookups off the diagram:

| Key | Ring position | Walk clockwise, first server hit | Owner |
|-----|---------------|----------------------------------|-------|
| `user:1` | 22% | next server at 30% | **S1** |
| `img:42` | 61% | next server at 65% | **S2** |
| `tag:13` | 92% | nothing until 100%, wrap to 0%, next at 5% | **S0** (wrap-around!) |
| a key at 70% | 70% | next server at 78% | **S3** |

Equivalently, each server **owns the arc that ends at it** — the span from the previous server (exclusive) clockwise up to itself (inclusive):

```
  S0 owns (78%  → 5%]   ... wrapping through 0     = 27% of the ring
  S1 owns ( 5%  → 30%]                             = 25% of the ring
  S2 owns (30%  → 65%]                             = 35% of the ring
  S3 owns (65%  → 78%]                             = 13% of the ring
```

Notice the ownership of a key depends **only on the two servers around it**, not on how many servers exist in total. That is the entire source of the property we want.

### 3. Why it works — the arithmetic of adding a node

Now add **S4**, and suppose it hashes to **position 45%**. Position 45% sits inside S2's arc (30% → 65%).

```
BEFORE:   ...─── S1@30% ──────────────────────────── S2@65% ───...
                        └──────── S2 owns this ─────┘

AFTER:    ...─── S1@30% ────── S4@45% ───────────── S2@65% ───...
                        └─ S4 ─┘└──── S2 still owns ────┘
                        (stolen)
```

**Only the keys landing in `(30%, 45%]` change owner.** They move from S2 to S4. That's 15% of the ring in this example. Every key in `(45%, 65%]` stays on S2. Every key owned by S0, S1, S3 is **completely untouched** — those servers don't even know a deployment happened.

Averaged out (and with virtual nodes making the average actually hold — see below), a new node lands with an arc of about **1/N of the ring**, so about **K/N keys move**, where K = total keys and N = the new number of nodes.

Here is the contrast, side by side. Assume **K = 10,000,000 cached keys**, going from **4 → 5 servers**:

| | Modulo hashing (`hash % N`) | Consistent hashing |
|---|---|---|
| Formula for keys remapped | `K × N_old/(N_old+1)` | `K / N_new` |
| Keys remapped, 4 → 5 | 10,000,000 × 4/5 = **8,000,000** | 10,000,000 / 5 = **2,000,000** |
| % of cache invalidated | **80%** | **20%** |
| Servers whose data is disturbed | **all 4** | just the **1** (or few) whose arcs were split |
| Extra DB load after the change | ~16× baseline → outage | ~4× baseline → survivable, and it decays as the cache refills |
| Same numbers at 100 → 101 servers | **99%** of keys move | **~1%** of keys move |

That last row is the punchline. **With modulo, scaling gets more dangerous as you grow. With consistent hashing, it gets safer.**

And when a server **dies** (5 → 4), the dead node's arc is inherited by its clockwise successor; only that node's ~1/N of keys are lost. Nothing else on the ring notices.

### 4. The problem with the basic ring: uneven distribution

Go back and look at the arc sizes from the diagram above:

```
  S0: 27%   ████████████████████████████
  S1: 25%   █████████████████████████
  S2: 35%   ███████████████████████████████████
  S3: 13%   █████████████
```

That's a real problem. S2 is doing **2.7× the work of S3** — it stores 2.7× the data, absorbs 2.7× the requests, and will hit its memory limit and start evicting long before S3 breaks a sweat. And this wasn't bad luck; with only 4 random points on a circle, wildly unequal arcs are *normal*. The expected size of the largest arc with N random points is roughly `ln(N)/N` of the ring — for N=4 that's ~35%, not the 25% you wanted. It gets no better in practice: cluster operators regularly saw 2–3× spreads on unvirtualized rings.

There's a **second, worse** problem: **failure cascades**.

```
  BEFORE:   S1@30% ────────────── S2@65% ─────── S3@78%
                    S2 owns 35%           S3 owns 13%

  S2 DIES:  S1@30% ─────────────────────────────  S3@78%
                    S3 now owns 30% → 78% = 48% of the ring!
```

When S2 dies, **100% of S2's load lands on exactly ONE server — S3**. Not spread across the survivors: all of it, on one box. S3 goes from 13% of the ring to 48% — its traffic nearly **quadruples in an instant**. If S3 was already at 40% CPU, it is now at 160% CPU, which is to say: it dies too. And then *its* load — now 48%+ of the ring — lands entirely on S0. This is a **cascading failure**, and it can eat the whole cluster in under a minute.

So the basic ring solves remapping but leaves you with lumpy load and a single-heir failure model. Both problems have the same fix.

### 5. Virtual nodes (vnodes) — the fix

**The idea:** stop placing each physical server on the ring **once**. Place it **many times** — typically 100 to 200 — by hashing decorated versions of its ID:

```javascript
// Instead of one point:
ring.place(hash("S0"));

// place 150 points, all owned by the same physical server S0:
for (let i = 0; i < 150; i++) {
  ring.place(hash(`S0#${i}`));   // "S0#0", "S0#1", ... "S0#149"
}
```

Each of those 150 points is a **virtual node** (vnode). All 150 map back to the same physical machine. With 4 servers × 150 vnodes you now have **600 points** scattered around the ring instead of 4, and each physical server owns **150 small arcs sprinkled all over the circle** rather than one giant contiguous slab.

```
  WITHOUT VNODES — 4 big arcs, one per server:

   ┌──────────────┬───────────┬──────────────────┬──────┐
   │      S0      │    S1     │        S2        │  S3  │   ← wildly uneven
   └──────────────┴───────────┴──────────────────┴──────┘
   0%                                                  100%


  WITH VNODES — each server owns many small arcs, interleaved:

   ┌──┬─┬───┬──┬────┬─┬──┬───┬─┬───┬──┬─┬────┬──┬─┬───┬──┐
   │S2│S0│S3 │S1│ S0 │S2│S1│S3 │S0│S2 │S3│S1│ S2 │S0│S3│S1│S0│
   └──┴─┴───┴──┴────┴─┴──┴───┴─┴───┴──┴─┴────┴──┴─┴───┴──┘
   0%                                                  100%
        (this is 17 of the 600 arcs; the real ring has 600)

   Sum of all S0 arcs ≈ 25%.  Sum of all S1 arcs ≈ 25%.  etc.
```

**Benefit (a) — the load evens out.** This is just the law of large numbers. One random arc has enormous variance; the *sum of 150 independent random arcs* has tiny variance. Empirically:

| Vnodes per server | Typical spread between the busiest and quietest of 4 servers |
|-------------------|--------------------------------------------------------------|
| 1 (no vnodes)     | 2× – 3× — often much worse |
| 10                | ~1.3× |
| 100               | ~1.10× |
| **150** (common)  | **~1.05–1.08×** — i.e. every server within a few % of 25% |
| 1000              | ~1.02×, but the ring costs more memory and lookups get slightly slower |

**Benefit (b) — failure spreads instead of crushing.** When S2 dies, you remove its **150 separate points**, scattered all over the ring. Each of those 150 orphaned arcs is inherited by whichever vnode happens to be clockwise of it — and with 450 surviving vnodes belonging to S0, S1 and S3 interleaved randomly, those arcs land **roughly evenly on all three survivors**:

```
   NO VNODES:  S2 dies → 100% of S2's load → S3 alone.   S3: +270% traffic → dies → cascade.

   VNODES:     S2 dies → S2's 150 arcs → split ~1/3 each to S0, S1, S3.
                         each survivor absorbs ~33% of S2's load
                         each survivor goes from 25% → ~33% of the ring
                         a +33% traffic bump. Survivable. No cascade.
```

That is the single most important operational property of vnodes: **it converts a fatal 4× spike on one node into a benign 1.33× bump on every node.**

**Benefit (c) — weighting heterogeneous hardware.** Your fleet isn't uniform: maybe you have two 64 GB boxes and two 128 GB boxes. Vnodes give you a dial. Give the big boxes **twice the vnodes**:

```javascript
ring.addNode('small-1', { vnodes: 150 });   // 64 GB  → ~1 share
ring.addNode('small-2', { vnodes: 150 });   // 64 GB  → ~1 share
ring.addNode('big-1',   { vnodes: 300 });   // 128 GB → ~2 shares
ring.addNode('big-2',   { vnodes: 300 });   // 128 GB → ~2 shares
// Ring share ≈ 150:150:300:300 → 16.6% : 16.6% : 33.3% : 33.3%
```

Load now follows capacity. You cannot do this at all with `hash % N`.

**Cost:** memory (600–3,000 ring entries instead of 4 — still trivial, a few hundred KB) and a slightly slower lookup, `O(log V)` where V = total vnodes. `log2(600) ≈ 10` comparisons. Nothing.

### 6. The implementation — a real `ConsistentHashRing` in Node.js

The one design decision that matters: **how do you do the "walk clockwise" step efficiently?** A linear scan over every vnode is `O(V)` — with 600–3,000 vnodes and 100k lookups/sec, that's tens of millions of comparisons per second, and it's on the hot path of every single request.

The fix: keep the vnode positions in a **sorted array**, and use **binary search** for the successor. Lookup becomes **`O(log V)`**. Adds/removes are `O(V log V)` (re-sort), but those happen at deploy time, not per request.

```javascript
// consistent-hash-ring.js
'use strict';

/**
 * FNV-1a, 32-bit. A real, fast, non-cryptographic hash with good avalanche
 * behaviour — flipping one input bit scrambles ~half the output bits, which is
 * exactly the "spread points uniformly around the ring" property we need.
 * (MD5 truncated to 32 bits works too and is what several Memcached clients use;
 *  FNV-1a is chosen here because it needs no crypto import and is much faster.)
 */
function fnv1a32(str) {
  let hash = 0x811c9dc5;                 // FNV offset basis
  for (let i = 0; i < str.length; i++) {
    hash ^= str.charCodeAt(i);
    // hash *= 16777619 (FNV prime), done with shifts to stay in 32-bit int range
    hash = (hash + ((hash << 1) + (hash << 4) + (hash << 7) + (hash << 8) + (hash << 24))) >>> 0;
  }
  return hash >>> 0;                     // force unsigned: 0 .. 2^32 - 1
}

class ConsistentHashRing {
  /**
   * @param {object}   opts
   * @param {number}   opts.vnodes  virtual nodes per physical node (default 150)
   * @param {function} opts.hash    string -> uint32
   */
  constructor({ vnodes = 150, hash = fnv1a32 } = {}) {
    this.defaultVnodes = vnodes;
    this.hash = hash;

    // The ring itself: positions sorted ascending. Binary search needs this invariant.
    this.positions = [];               // number[]  — uint32 ring positions
    this.ownerAt = new Map();          // position -> physical node id
    this.vnodeCount = new Map();       // node id  -> how many vnodes it has (for removal + weighting)
  }

  /**
   * Add a physical node. `weight` multiplies its vnode count — a box with 2x the
   * RAM gets weight 2 and therefore ~2x the ring share, and ~2x the keys.
   */
  addNode(nodeId, { weight = 1 } = {}) {
    if (this.vnodeCount.has(nodeId)) throw new Error(`node already on ring: ${nodeId}`);

    const count = Math.round(this.defaultVnodes * weight);
    this.vnodeCount.set(nodeId, count);

    for (let i = 0; i < count; i++) {
      // Decorating the id is what scatters one physical node across the whole ring.
      let pos = this.hash(`${nodeId}#${i}`);
      // Collisions are astronomically rare but not impossible; probe forward so we
      // never silently drop a vnode or let two nodes fight over one position.
      while (this.ownerAt.has(pos)) pos = (pos + 1) >>> 0;
      this.ownerAt.set(pos, nodeId);
      this.positions.push(pos);
    }
    this.positions.sort((a, b) => a - b);   // keep the binary-search invariant
    return this;
  }

  /** Remove a physical node (it died, or you're scaling in). Its arcs go to its successors. */
  removeNode(nodeId) {
    if (!this.vnodeCount.has(nodeId)) return this;

    for (const [pos, owner] of this.ownerAt) {
      if (owner === nodeId) this.ownerAt.delete(pos);
    }
    this.positions = this.positions.filter((p) => this.ownerAt.has(p));
    this.vnodeCount.delete(nodeId);
    return this;
  }

  /**
   * THE core operation. Hash the key onto the ring, then walk CLOCKWISE to the
   * first vnode — implemented as a binary search for the smallest position >= h.
   * O(log V). If nothing is >= h we've run off the end of the ring, so we wrap
   * around to index 0 — that wrap IS the "circle" in consistent hashing.
   */
  getNode(key) {
    if (this.positions.length === 0) return null;

    const h = this.hash(key);
    let lo = 0;
    let hi = this.positions.length;       // half-open [lo, hi)

    while (lo < hi) {
      const mid = (lo + hi) >>> 1;
      if (this.positions[mid] < h) lo = mid + 1;
      else hi = mid;
    }

    const idx = lo === this.positions.length ? 0 : lo;   // <-- the wrap-around
    return this.ownerAt.get(this.positions[idx]);
  }

  /** Successor list — the next R DISTINCT physical nodes clockwise. This is how */
  /** Dynamo/Cassandra pick replicas: the key's owner plus the next R-1 nodes.    */
  getNodes(key, replicas = 1) {
    if (this.positions.length === 0) return [];

    const h = this.hash(key);
    let lo = 0, hi = this.positions.length;
    while (lo < hi) {
      const mid = (lo + hi) >>> 1;
      if (this.positions[mid] < h) lo = mid + 1; else hi = mid;
    }

    const out = [];
    const seen = new Set();
    for (let step = 0; step < this.positions.length && out.length < replicas; step++) {
      const pos = this.positions[(lo + step) % this.positions.length];
      const owner = this.ownerAt.get(pos);
      if (!seen.has(owner)) { seen.add(owner); out.push(owner); }  // skip vnodes of a node we already have
    }
    return out;
  }

  get nodes() { return [...this.vnodeCount.keys()]; }
  get ringSize() { return this.positions.length; }
}

module.exports = { ConsistentHashRing, fnv1a32 };
```

### 7. The demo that proves it — measure the keys that actually move

Reading the theory is not the same as believing it. Run this. It builds a 4-node ring, hashes 10,000 keys, prints the distribution, then adds a 5th node and counts **exactly how many keys changed owner**.

```javascript
// demo.js
const { ConsistentHashRing } = require('./consistent-hash-ring');

const KEYS = Array.from({ length: 10_000 }, (_, i) => `user:${i}:session`);

function distribution(ring, keys) {
  const counts = new Map(ring.nodes.map((n) => [n, 0]));
  for (const k of keys) counts.set(ring.getNode(k), counts.get(ring.getNode(k)) + 1);
  return counts;
}

function printDistribution(label, counts, total) {
  console.log(`\n${label}`);
  const ideal = 100 / counts.size;
  for (const [node, n] of [...counts].sort()) {
    const pct = (n / total) * 100;
    const bar = '█'.repeat(Math.round(pct));
    console.log(`  ${node.padEnd(10)} ${String(n).padStart(5)} keys  ${pct.toFixed(2).padStart(5)}%  ${bar}`);
  }
  console.log(`  (perfectly even would be ${ideal.toFixed(2)}% each)`);
}

// ---- Step 1: 4 nodes, 150 vnodes each -------------------------------------
const ring = new ConsistentHashRing({ vnodes: 150 });
['cache-A', 'cache-B', 'cache-C', 'cache-D'].forEach((n) => ring.addNode(n));

console.log(`Ring has ${ring.ringSize} vnodes for ${ring.nodes.length} physical nodes.`);
printDistribution('DISTRIBUTION WITH 4 NODES:', distribution(ring, KEYS), KEYS.length);

// Snapshot who owns what, so we can diff after scaling.
const before = new Map(KEYS.map((k) => [k, ring.getNode(k)]));

// ---- Step 2: add a 5th node, then diff ------------------------------------
ring.addNode('cache-E');
printDistribution('DISTRIBUTION AFTER ADDING A 5th NODE:', distribution(ring, KEYS), KEYS.length);

let moved = 0;
for (const k of KEYS) if (before.get(k) !== ring.getNode(k)) moved++;

const movedPct = (moved / KEYS.length) * 100;
console.log(`\n=== THE WHOLE POINT ===`);
console.log(`Keys remapped by CONSISTENT HASHING : ${moved} / ${KEYS.length} = ${movedPct.toFixed(2)}%`);
console.log(`Theoretical optimum (K / N_new)      : ${(100 / 5).toFixed(2)}%`);

// The same scaling event under naive modulo hashing, for contrast:
const { fnv1a32 } = require('./consistent-hash-ring');
let modMoved = 0;
for (const k of KEYS) if (fnv1a32(k) % 4 !== fnv1a32(k) % 5) modMoved++;
console.log(`Keys remapped by MODULO hashing      : ${modMoved} / ${KEYS.length} = ${((modMoved / KEYS.length) * 100).toFixed(2)}%`);
```

**Representative output** (your exact numbers will vary by a few tenths of a percent — the hash is deterministic but the arcs are effectively random):

```
Ring has 600 vnodes for 4 physical nodes.

DISTRIBUTION WITH 4 NODES:
  cache-A     2571 keys  25.71%  ██████████████████████████
  cache-B     2438 keys  24.38%  ████████████████████████
  cache-C     2519 keys  25.19%  █████████████████████████
  cache-D     2472 keys  24.72%  █████████████████████████
  (perfectly even would be 25.00% each)

DISTRIBUTION AFTER ADDING A 5th NODE:
  cache-A     2065 keys  20.65%  █████████████████████
  cache-B     1943 keys  19.43%  ███████████████████
  cache-C     2038 keys  20.38%  ████████████████████
  cache-D     1971 keys  19.71%  ████████████████████
  cache-E     1983 keys  19.83%  ████████████████████
  (perfectly even would be 20.00% each)

=== THE WHOLE POINT ===
Keys remapped by CONSISTENT HASHING : 1983 / 10000 = 19.83%
Theoretical optimum (K / N_new)      : 20.00%
Keys remapped by MODULO hashing      : 8007 / 10000 = 80.07%
```

Three things to take from that output, and they are the three things to say out loud in an interview:

1. **The distribution is even** — every node within ~1% of its fair share, because of 150 vnodes each.
2. **~19.8% of keys moved, essentially the theoretical minimum of 1/5.** You cannot do better; those keys *must* move for the new node to take any load at all.
3. **Modulo moved 80.07%** — matching the `N/(N+1) = 4/5` prediction exactly. Four times the damage, and every one of those is a cache miss hitting your database.

Notice also that the ~1,983 keys that moved went **entirely to cache-E**. Look at the before/after rows: A, B, C and D each *shrank*, but no key moved *between* A, B, C and D. Only the new node gained. That is exactly what "only the arcs behind the new vnodes are stolen" means.

---

## Visual / Diagram description

### Diagram 1: The ring in a real request path

This is what you draw on the whiteboard when the interviewer says "how does a client know which cache server to talk to?"

```
   ┌──────────────┐   GET user:1
   │  App Server  │
   │  (Node.js)   │
   └──────┬───────┘
          │  1. key = "user:1"
          ▼
   ┌─────────────────────────────────────────────┐
   │       ConsistentHashRing (in-process)        │
   │                                              │
   │  2. h = fnv1a32("user:1")      = 0x3C1A...   │
   │  3. binary search sorted positions[600]      │
   │     for the smallest pos >= h   → O(log V)   │
   │  4. that vnode is "cache-B#87"               │
   │  5. strip the "#87" → physical node cache-B  │
   └──────────────────────┬───────────────────────┘
                          │  6. connect to cache-B directly
        ┌─────────┬───────┴────┬──────────┬──────────┐
        ▼         ▼            ▼          ▼          ▼
   ┌────────┐┌────────┐  ┌────────┐ ┌────────┐ ┌────────┐
   │cache-A ││cache-B │  │cache-C │ │cache-D │ │cache-E │
   │        ││ ◀ HIT  │  │        │ │        │ │ (new)  │
   └────────┘└────────┘  └────────┘ └────────┘ └────────┘
```

The critical thing this diagram shows: **there is no coordinator, no lookup service, no routing table hop.** Every client computes the same answer independently from the same ring, in microseconds, in-process. Add a node to the ring config and every client independently reaches the same new conclusion. That decentralisation is *why* Dynamo-style systems adopted it.

### Diagram 2: The vnode ring, unrolled

The circle is hard to draw under interview pressure. Unroll it into a line (remembering the right edge wraps to the left edge) — it's easier to draw and shows the same thing:

```
 pos 0                                                              2^32-1 ─┐
   │                                                                        │
   ├──┬────┬──┬─────┬───┬──┬────┬──┬───┬─────┬──┬───┬────┬──┬────┬───┬─────┤ (wraps)
   │A1│ C4 │B7│ D2  │A9 │C1│ B3 │D8│A4 │ C7  │B2│D5 │ A6 │C9│ B8 │D1 │ A2  │
   └──┴────┴──┴─────┴───┴──┴────┴──┴───┴─────┴──┴───┴────┴──┴────┴───┴─────┘
      ▲                                    ▲
      │                                    │
   key "x" hashes here                key "y" hashes here
   → walks right → lands in C4        → walks right → lands in C7
   → owner = cache-C                  → owner = cache-C
                                        (same physical node, different vnode!)

   ── Now cache-C DIES. Delete every C* vnode: ──

   ├──┬───────┬──┬─────┬────────┬────┬──┬───┬────────┬──┬────────┬──┬────────┤
   │A1│  B7   │B7│ D2  │   A9   │ B3 │D8│A4 │   B2   │D5│   A6   │B8│  D1    │
   └──┴───────┴──┴─────┴────────┴────┴──┴───┴────────┴──┴────────┴──┴────────┘
        ▲                                    ▲
        │                                    │
   key "x" now → B7 (cache-B)          key "y" now → B2 (cache-B)... or D5, or A6
                                        depending on where it fell.

   C's arcs were absorbed by A, B and D — SPREAD OUT, not dumped on one heir.
```

Compare that to the no-vnode case, where deleting one big `C` block hands **every single one of C's keys to whichever single node sits to its right.**

---

## Real world examples

### 1. Amazon DynamoDB (and the Dynamo paper that started it)

The 2007 Dynamo paper is where most engineers first met consistent hashing, and it introduced the vnode refinement explicitly. Dynamo places storage nodes on a ring, and a key is stored on its **coordinator** (the first node clockwise) **plus the next N-1 distinct physical nodes clockwise** — that ordered list is called the key's **preference list**. That's the `getNodes(key, replicas)` method in the implementation above, and it is why replication and partitioning fall out of the *same* data structure. The paper also calls out the two problems we covered — non-uniform load and non-uniform failure inheritance — and names virtual nodes as the fix, including using vnode counts to weight heterogeneous hardware.

### 2. Apache Cassandra

Cassandra's ring is the direct descendant of Dynamo's. Every node owns a set of **token ranges** (arcs) on a ring, and the partition key is hashed (by default with Murmur3) to find its range. Since Cassandra 1.2, `num_tokens` in `cassandra.yaml` sets the vnodes per node — the default was long **256**, and modern guidance has moved to **16** (a lower count reduces the blast radius of a node failure across the cluster, at some cost in evenness, and interacts better with rack-aware replica placement). Bootstrapping a new node streams only the ranges it takes over — this is exactly "only K/N keys move", implemented as a live data migration.

### 3. Memcached client libraries

Memcached servers know nothing about each other — there is no cluster protocol. **All of the sharding logic lives in the client.** The classic `libketama` algorithm (originating at Last.fm, then ported into most Memcached clients across languages) is consistent hashing with vnodes: it hashes `"server:port-N"` for many N, uses MD5, takes 4 x 32-bit points from each MD5 digest, and sorts them into a ring. Every app server builds the identical ring from the identical server list, so they all agree on placement without ever talking to each other. If you have used a Node Memcached client with a "ketama" or "consistent" hashing option, this is exactly the code you were running.

### 4. Redis Cluster — a related but *different* approach (be honest about this in interviews)

Redis Cluster does **not** use a hash ring. It uses **16,384 fixed hash slots**. The key's slot is `CRC16(key) mod 16384`, and each master node is *assigned a set of slot numbers* (e.g. node A holds slots 0–5460). The cluster gossips a slot→node map, and clients cache it.

| | Consistent hashing (ring) | Redis Cluster (fixed slots) |
|---|---|---|
| Key → bucket | `hash(key)` → position → walk clockwise | `CRC16(key) % 16384` → slot number |
| Bucket → node | implicit, from ring geometry | an **explicit map**, gossiped between nodes |
| Adding a node | new vnodes land on the ring; keys move automatically | an operator/orchestrator **explicitly reassigns slots** and migrates them |
| Rebalancing control | emergent — you get what the hash gives you | precise — you can move exactly the slots you want |
| Keys moved when scaling | ~1/N | ~1/N (same result! you move ~1/N of the slots) |

**They achieve the same goal by different means.** Both add a level of indirection so key placement isn't tied to `N`. The ring's indirection is *geometric and implicit*; slots are *tabular and explicit*. The slot approach is easier to reason about and to control (you can migrate one slot at a time and watch it), which is why it's popular in systems with a coordinator. The ring approach needs no coordinator at all, which is why it's popular in leaderless systems like Dynamo and Cassandra. **Calling Redis Cluster "consistent hashing" is a common and forgivable slip — but knowing the difference is a genuine senior signal.**

### 5. CDNs and sticky load balancers

CDN edge tiers use consistent hashing to decide which *edge cache* in a POP holds a given object, so a cache-fill for `/video/123.ts` always goes to the same peer and the object isn't duplicated across every machine in the rack. Load balancers do the same for sticky routing: HAProxy's `balance hash` with `hash-type consistent`, NGINX's `hash $request_uri consistent`, and Envoy's `RING_HASH` and `MAGLEV` policies all hash a request attribute onto a ring so the same user or the same URL keeps landing on the same backend — and so that removing one backend doesn't reshuffle everyone else's stickiness. (Maglev, from Google, is a related "consistent hashing with better lookup and even better balance" scheme worth name-dropping.)

---

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **`hash(key) % N`** | Dead simple; one line; perfectly even; O(1); zero memory | Adding/removing a node remaps ~N/(N+1) of keys — a cache-wipe and a DB stampede. Gets *worse* as you grow. Cannot weight unequal servers. |
| **Ring, no vnodes** | Only ~K/N keys move on a change; no coordinator needed | Badly uneven arcs (2–3× spread is normal); a dead node dumps 100% of its load on ONE neighbour → cascading failure |
| **Ring + vnodes (150ish)** | Even load (±few %); failure load spreads across many nodes; supports weighted/heterogeneous hardware; still no coordinator | More memory (V ring entries); lookup is O(log V) not O(1); rebalancing still physically moves data; ring config must be identical on every client |
| **Fixed slots (Redis Cluster)** | Explicit, auditable, operator-controllable migration; easy to reason about; same ~1/N movement | Needs a slot→node map to be distributed and kept fresh (gossip or a coordinator); fixed slot count (16,384) is a hard ceiling on node count |

**Two things consistent hashing does NOT solve. Say these unprompted — it's the fastest way to sound like you've actually operated one:**

1. **Hot keys.** If `celebrity:profile` gets 500,000 reads/sec, consistent hashing faithfully sends *all 500,000* to the one node that owns that key's arc. Perfect balance of *keys* is not balance of *traffic*. Fixes are orthogonal to the ring: **replicate the hot key across many nodes** and pick one at random on read; **cache it locally in the app process** for a few seconds; or **key-split it** (`celebrity:profile#0..9`, write to all, read one at random). A ring can't help you — one key has exactly one position.
2. **Rebalancing still moves real data.** "Only 20% of keys move" still means 20% of your dataset is copied over the network. On a 10 TB cluster that is **2 TB of streaming**, which takes hours, competes with live traffic, and must be throttled. Consistent hashing minimises the movement; it doesn't make it free.

Also worth naming: **the ring must be identical on every client.** If app server #7 has a stale node list, it will compute a different owner and read/write the wrong box. In client-side setups (Memcached) this makes config rollout a correctness concern, not just an ops one.

**Rule of thumb:** if data is *partitioned across a changing set of nodes* and *moving data is expensive* (a cache, a shard map, a sticky session), use consistent hashing with **100–200 vnodes per physical node**. If the node set is genuinely fixed forever, plain modulo is fine and simpler — but node sets are almost never fixed forever.

---

## Common interview questions on this topic

### Q1: "Why not just use `hash(key) % N`? Walk me through what breaks."
**Hint:** Do the arithmetic out loud. Going N → N+1 remaps ~N/(N+1) of keys — 80% at 4→5, 99% at 100→101. Every remapped key is a cache miss. If you were at a 95% hit rate serving 100k reads/sec, the DB goes from 5k to ~80k reads/sec instantly — a 16× spike that saturates the connection pool and takes the DB down. Then add: **the same thing happens when a node dies unexpectedly**, so this isn't just a deploy-time problem. Consistent hashing bounds the movement to ~1/N.

### Q2: "What are virtual nodes and why do you need them? Isn't the basic ring enough?"
**Hint:** Two independent problems. (1) **Balance:** with only N random points, arcs are wildly uneven — expected largest arc is ~ln(N)/N, so a 4-node ring routinely has one node holding 40–50%. 150 vnodes per node collapses the spread to a few percent by the law of large numbers. (2) **Failure blast radius:** without vnodes, a dead node's entire load lands on exactly one successor, which may then fall over — a cascade. With vnodes, its 150 scattered arcs are inherited by many different neighbours, so each survivor absorbs only ~1/(N-1) of the dead node's load. Bonus: vnode counts let you weight bigger machines.

### Q3: "How do you find the successor node efficiently? What's the complexity?"
**Hint:** Keep vnode positions in a **sorted array** and **binary search** for the smallest position ≥ `hash(key)`; if you run past the end, wrap to index 0 — that wrap *is* the circle. Lookup is **O(log V)** where V = N × vnodesPerNode (600 vnodes → ~10 comparisons). Node add/remove is O(V log V) but happens at deploy time, not per request. A linear scan is O(V) and is the wrong answer. (A balanced BST / red-black tree gives the same O(log V) — Java's `TreeMap.ceilingEntry` is the canonical version.)

### Q4: "Does consistent hashing solve the hot-key problem?"
**Hint:** **No** — and saying so confidently is the point of the question. It balances *keys*, not *traffic*. One viral key still maps to one position and therefore one node, which can be melted by that key alone. Fixes are separate: replicate the hot key to R nodes and read from a random one; add a short-TTL in-process cache in front of the ring; or shard the key itself into `key#0..key#9`. Mention that detecting hot keys (count-min sketch / heavy hitters on the client) is a prerequisite.

### Q5: "Redis Cluster uses consistent hashing, right?"
**Hint:** Careful — this is a trap. Redis Cluster uses **16,384 fixed hash slots**: `CRC16(key) % 16384`, with an explicit, gossiped slot→node assignment map. It is not a ring and there's no clockwise walk. It solves the *same* problem (decouple key placement from node count, so scaling moves only ~1/N of data) with an explicit table instead of ring geometry — which trades away coordinator-free operation for precise, auditable, one-slot-at-a-time migration. Cassandra and DynamoDB are the real ring users.

### Q6: "How many virtual nodes should you use?"
**Hint:** 100–200 per physical node is the standard answer, and 150 is the number most often cited (libketama-era Memcached clients land in this range). Justify it with the balance/cost curve: 1 vnode gives a 2–3× spread; 100 gives ~10%; 150 gives ~5%; 1,000 gives ~2% but costs memory and a slightly deeper binary search. Note the counter-pressure: Cassandra *lowered* its default from 256 to 16, because a very high vnode count means every node shares range boundaries with almost every other node, which enlarges the blast radius of correlated failures and complicates rack-aware replica placement. So: more vnodes = better balance, but more coupling.

---

## Practice exercise

### Build the ring, then break it

Type out `consistent-hash-ring.js` and `demo.js` from section 6 and 7 (type them — don't paste; you want the binary search in your fingers for the whiteboard). Then run these five experiments and **write down the number you got for each**:

1. **Prove the disaster.** Set `vnodes: 1` (no virtual nodes) and print the 4-node distribution. Record the busiest and quietest node. Is the spread over 2×? Now bump to `vnodes: 150` and print it again. This one comparison is the entire justification for vnodes and you now own it as a fact, not a claim.

2. **Prove the fix.** With `vnodes: 150`, add a 5th node and print the % of the 10,000 keys that moved. You should get ~20%. Then repeat starting from 10 nodes and adding an 11th — you should get ~9%. Confirm the trend: **the bigger the cluster, the smaller the disruption.** Modulo does the opposite.

3. **Kill a node.** With `vnodes: 1`, snapshot ownership, then `ring.removeNode('cache-C')` and count how many of C's keys went to each survivor. You should find **one survivor took ~all of them**. Now do the same with `vnodes: 150` — the load should split roughly in thirds. You have just reproduced the cascading-failure mechanism and its cure.

4. **Weight the fleet.** Add `cache-A` and `cache-B` with `{ weight: 1 }` and `cache-BIG` with `{ weight: 2 }`. Verify `cache-BIG` gets roughly 50% of keys and the other two get ~25% each.

5. **Simulate the stampede (the payoff).** Assume 100,000 reads/sec and a 95% cache hit rate. Compute, for both strategies, the reads/sec that hit the database in the minute after you add the 5th node:
   - Modulo: `100,000 × (0.80 + 0.05×0.20)` ≈ ? reads/sec to the DB
   - Consistent hashing: `100,000 × (0.20 + 0.05×0.80)` ≈ ? reads/sec to the DB
   - How many × over the 5,000 reads/sec baseline is each? **Write the two numbers down.** They are the two numbers you say in the interview.

**Deliverable:** a short README with the five numbers, plus one paragraph explaining to a teammate why you would not ship a Memcached tier without this.

---

## Quick reference cheat sheet

- **The problem:** `hash(key) % N` remaps **N/(N+1)** of all keys when you add one node — **80%** at 4→5, **99%** at 100→101. Every remap is a cache miss; the DB eats the whole miss storm and dies.
- **The same disaster happens on node DEATH**, uninvited, at 3 a.m. That's why this isn't optional.
- **The ring:** bend the hash output space (`0 … 2³²-1`) into a circle. Hash **servers** onto it. Hash **keys** onto it.
- **The one rule:** a key belongs to the **first server clockwise** from it (its *successor*). Fall off the end → wrap to 0.
- **The win:** adding a node steals only the arc **between it and its predecessor**. Only **~K/N** keys move — 20% instead of 80%, and it *improves* as the cluster grows.
- **Basic ring's flaw #1 — balance:** N random points make wildly unequal arcs; one node can own 40–50% of the ring.
- **Basic ring's flaw #2 — cascade:** a dead node dumps **100% of its load on ONE neighbour**, which can then die too.
- **Virtual nodes:** hash each physical server onto the ring **100–200 times** (`S0#0`, `S0#1`, …). Each server owns many small scattered arcs.
- **Vnodes fix both:** load evens out (law of large numbers), and a dead node's arcs are absorbed by **many** neighbours (~1/(N-1) each), not one.
- **Vnodes also weight hardware:** a 2× bigger box gets 2× the vnodes and therefore ~2× the keys.
- **Implementation:** sorted array of ring positions + **binary search** for the successor → **O(log V)** lookup. Never linear-scan the ring.
- **Replication falls out for free:** walk clockwise past the owner to collect the next R **distinct** physical nodes — that's Dynamo's *preference list*.
- **Used by:** DynamoDB/Dynamo, Cassandra (`num_tokens`), Memcached clients (**ketama**), CDN edge selection, HAProxy/NGINX/Envoy sticky routing (`RING_HASH`).
- **Redis Cluster is NOT a ring:** it's **16,384 fixed hash slots** (`CRC16(key) % 16384`) with an explicit gossiped slot→node map. Same goal, different mechanism. Knowing this distinction is a senior signal.
- **It does NOT solve hot keys.** One viral key = one ring position = one node. Fix with key replication, key-splitting, or a local in-process cache.
- **Rebalancing still moves real bytes.** 20% of 10 TB is 2 TB across the network. Minimised ≠ free. Throttle it.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [64 — Database Sharding](./64-database-sharding.md) — consistent hashing is *how* you decide which shard a row lives on, and how you add a shard without rewriting the world |
| **Next** | [75 — Data Partitioning](./75-data-partitioning.md) — the broader menu of partitioning strategies (range, hash, directory) that consistent hashing sits inside |
| **Related** | [105 — HLD: Design a Distributed Cache](./105-hld-distributed-cache.md) — the interview question where you *must* produce this from memory, ring diagram and all |
| **Related** | [59 — Caching In Depth](./59-caching-in-depth.md) — cache misses, stampedes, and the hot-key problem the ring deliberately does not solve |
