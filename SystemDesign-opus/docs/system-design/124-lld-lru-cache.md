# 124 — Design an LRU Cache
## Category: LLD Case Study

---

## What is this?

An **LRU cache** (Least Recently Used) is a fixed-size box that remembers the most recently touched items and quietly throws away the one you haven't touched in the longest time when it runs out of room.

Think of the pile of clothes on the chair in your bedroom. The chair holds maybe ten things. Every morning you grab a shirt, wear it, and toss it back on **top** of the pile. When the pile gets too tall to add another item, which one do you remove? The one crushed at the very **bottom** — the thing you haven't worn in weeks. That is exactly an LRU cache: recently-used items float to the top, and the least-recently-used item at the bottom gets evicted first.

The whole challenge — and the reason this is one of the most-asked coding-interview questions on earth — is doing both "grab any item instantly" **and** "find the bottom of the pile instantly" at the same time. That tension is the entire problem.

---

## Why does it matter?

**Interview angle.** "Implement an LRU cache" is a rite of passage. It shows up at Google, Meta, Amazon, and virtually every mid-to-senior interview loop. It is beloved because it is small enough to finish in 30 minutes yet it forces you to combine **two** data structures — a hash map and a doubly-linked list — and reason about why neither alone is enough. Interviewers watch for whether you reach for the O(1) design or fumble into an O(n) scan. If you can explain *why* a doubly-linked list (and not an array or a singly-linked list), you have already passed.

**Real-work angle.** LRU is not a toy. It is the **eviction policy** baked into real caches everywhere: Redis offers `allkeys-lru`, your CPU's L1/L2 caches approximate LRU in hardware, operating-system page caches use LRU-like clocks, and most application-level caches (Guava, Caffeine, `lru-cache` on npm) default to it. Recall from [59 — Caching in Depth](./59-caching-in-depth.md) that every cache with a bound must answer one question: *when I'm full, who gets kicked out?* LRU is the most common answer, and knowing how it works under the hood is what lets you tune it — or replace it when it misbehaves.

What breaks if you don't understand this? You write a cache that either grows without bound (memory leak, OOM crash at 3 a.m.) or evicts with an O(n) scan that turns your fast cache into a slow one.

---

## The core idea — explained simply

### The Coat-Check Room Analogy

Picture a coat-check room at a busy theatre. There is a long **conveyor rack** and a **receptionist with a ledger**.

- The **rack** is one long line of hooks. Coats near the **front** were just handed in or just retrieved; coats near the **back** have been hanging untouched all night.
- The **ledger** is a lookup table: "ticket #57 → the coat currently on hook near the middle." Without the ledger, finding your specific coat means walking the whole rack.

When you **drop off** a coat, the receptionist hangs it at the **front** and writes your ticket in the ledger. When you **retrieve** a coat, she reads the ledger to find it instantly, then — because you clearly still care about this coat — she **moves it back to the front** of the rack. When the rack is completely full and a new coat arrives, she grabs whatever is at the very **back** (nobody has touched it all night), removes it, and erases its ticket from the ledger.

Two tools, two jobs. The **ledger gives instant lookup by ticket**. The **rack gives instant knowledge of what's oldest and lets her move a coat to the front**. Neither tool alone is enough: a ledger can't tell you what's oldest, and a rack alone would force her to walk it to find your coat.

| Analogy | Technical concept |
|---------|-------------------|
| Coat-check room | The `LRUCache` |
| A single coat on a hook | A `Node` (holds key + value) |
| The conveyor rack (ordered line) | Doubly-linked list (usage order) |
| Front of the rack | Most-recently-used (MRU) end / head |
| Back of the rack | Least-recently-used (LRU) end / tail |
| The receptionist's ledger | The hash `Map` (key → Node) |
| Reading the ledger | `map.get(key)` — O(1) lookup |
| Moving a coat to the front | Splicing a node to the head — O(1) |
| Removing the back coat when full | Evicting the tail node — O(1) |
| Total number of hooks | `capacity` |

Everything below is that analogy made precise in code.

---

## Key concepts inside this topic

This follows the [111 — LLD Approach Framework](./111-lld-approach-framework.md): requirements → objects → behaviours → class diagram → code → patterns → extensions. Because this is fundamentally a **data-structure problem**, we weight it heavily toward the structure and the implementation.

### 1. Requirements & clarifying questions

Before writing a line, pin down the contract. State it back to the interviewer:

**Functional requirements:**

| Operation | Behaviour |
|-----------|-----------|
| `get(key)` | Return the value if present, and mark that key **most-recently-used**. Return a "miss" sentinel (`-1` in the classic LeetCode framing, or `undefined`) if absent. |
| `put(key, value)` | Insert a new key, or update an existing key's value. Either way, mark it **most-recently-used**. If inserting would exceed `capacity`, first **evict the least-recently-used** entry. |

**The hard requirement — the one the whole question exists to test:**

> **Both `get` and `put` must run in O(1) time** — constant time, independent of how many items are in the cache.

If your solution is O(n) — for example, scanning an array to find the oldest item — you have technically "solved" it but failed the interview. O(1) is the point.

**Clarifying questions to ask (asking these earns real points):**

- **Is the capacity fixed at construction, or resizable later?** (Assume fixed unless told otherwise.)
- **What should `get` return on a miss?** `-1`, `null`, `undefined`? (LeetCode says `-1`; idiomatic JS says `undefined`.)
- **Do we need thread-safety / concurrency?** In single-threaded Node this is usually "no," but flag it — it matters for the extensions.
- **Is there a TTL (time-to-live / expiry) per entry?** Classic LRU has none; real caches often do.
- **Are keys always present-and-valid, or must we handle updates to existing keys specially?** (We must — updating must not grow the size.)
- **Can `capacity` be 0?** A defensive edge case worth a one-line guard.

### 2. Identify core objects (the nouns)

The object list here is small but precise — resist adding more.

| Object | Responsibility |
|--------|----------------|
| `LRUCache` | The public facade. Owns capacity, the map, and the two sentinels. Exposes `get` / `put`. |
| `Node` | A single doubly-linked-list node holding `key`, `value`, `prev`, and `next`. |
| `Map` (built-in) | Internal index: `key → Node`. Gives O(1) lookup so `get` can find a node without walking the list. |

Notice there is no `EvictionPolicy` interface, no `Strategy` object, no factory. This problem is deliberately lean. Over-engineering it with patterns is a common way to *lose* points — more on that in section 7.

### 3. The core insight — WHY hash map + doubly-linked list

This is the whole problem. If you understand this section, everything else is typing. We need two things to be fast simultaneously:

**Requirement A — O(1) lookup by key.** Given a key, find its value instantly. A **hash `Map`** does exactly this. Done.

**Requirement B — O(1) eviction of the oldest, and O(1) "mark as newest".** We must (i) know which entry is least-recently-used and remove it instantly, and (ii) take any just-accessed entry and move it to the "most recent" position instantly. This is an **ordering** requirement — we need to maintain items in usage order and reorder cheaply.

A `Map` alone cannot do Requirement B efficiently: a plain hash map has no notion of "oldest." So we add a second structure that maintains **usage order as a line**: most-recently-used at the **head**, least-recently-used at the **tail**.

**Why a doubly-linked list, and not an array or a singly-linked list?** This is the exact question interviewers probe. Walk through each candidate:

| Structure | Move an item to front | Remove the oldest | Verdict |
|-----------|----------------------|-------------------|---------|
| **Array / dynamic list** | O(n) — shifting every element over | O(1) at one end, but "move from middle to front" is O(n) | Fails: middle removal shifts everything |
| **Singly-linked list** | To splice a node out you need its **predecessor**, and a singly-linked list can only reach the predecessor by walking from the head — O(n) | O(n) to find the node before the tail | Fails: can't reach the predecessor in O(1) |
| **Doubly-linked list** | O(1) — the node has both `prev` and `next`, so you splice it out and relink in constant time | O(1) — the tail node's `prev` gives you the new tail instantly | **Wins** |

The killer detail is **middle removal**. On a `get`, the node you just accessed can be anywhere in the middle of the list, and you must yank it out and move it to the head. To splice a node out of a list in O(1) you must rewire **both** its neighbours: `node.prev.next = node.next` and `node.next.prev = node.prev`. That requires pointers to **both** neighbours — which is precisely what "doubly-linked" gives you. A singly-linked list only knows `next`, so it can't find `node.prev` without an O(n) walk. An array can't remove from the middle without an O(n) shift.

**The bridge between the two structures:** the `Map` stores `key → the Node object itself` (not the value — the actual node). So on `get(key)`, you do `map.get(key)` to land directly on the node in O(1), read its value, **and** you already hold the node, so you can splice it to the head in O(1). The two structures share the same node objects. That sharing is what makes both operations O(1) at once.

```
      THE MAP (index)                 THE DOUBLY-LINKED LIST (usage order)
   key -> Node reference

   ┌───────┬──────────┐            HEAD (MRU)                     TAIL (LRU)
   │  "a"  │  ●───────┼──┐        ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
   ├───────┼──────────┤  └──────► │ a:10 │◄─►│ c:30 │◄─►│ b:20 │◄─►│ d:40 │
   │  "b"  │  ●───────┼───────────┼──────┼───┼──────┼──►│      │   │      │
   ├───────┼──────────┤           └──────┘   └──────┘   └──────┘   └──────┘
   │  "c"  │  ●───────┼──────────────────────►                        ▲
   ├───────┼──────────┤                                               │
   │  "d"  │  ●───────┼───────────────────────────────────────────────┘
   └───────┴──────────┘        "a" was used most recently; "d" is next to be evicted
```

The Map points *into* the list. Look up any key → jump straight to its node → read it and reorder it, all O(1).

### 4. The sentinel-node trick

Naive linked-list code is a minefield of `null` checks: "what if the list is empty?", "what if I'm removing the only node?", "what if I'm removing the head?" Every one of those is a special case, and every special case is a place bugs hide.

The fix is **sentinel nodes** (also called dummy or guard nodes). Instead of `head` and `tail` pointing at real data nodes, we create two permanent, contentless nodes — a dummy `head` and a dummy `tail` — and wire them together at construction:

```
   head(dummy) ◄────► tail(dummy)          // an "empty" cache
```

Every real node lives *between* these two sentinels. Now:

- The **most-recently-used** node is always `head.next`.
- The **least-recently-used** node (the eviction target) is always `tail.prev`.
- Inserting at the front is always "put it between `head` and `head.next`" — the `head` sentinel is *never* null, so there is no "empty list" special case.
- Removing any node is always "link its two neighbours" — and because the neighbours might be sentinels rather than null, the same three lines of code work whether the node is in the middle, at the front, or is the only real node.

The sentinels never move and never hold data; they exist purely so that **every real node is guaranteed to have a non-null `prev` and `next`**. That single guarantee erases every null-check branch. This is the detail that separates clean, correct LRU code from the buggy version that crashes on the empty-list edge case. Interviewers notice when you reach for sentinels unprompted.

### 5. Class / structure diagram

```
 ┌─────────────────────────────────────────────────────────────┐
 │                        LRUCache                              │
 ├─────────────────────────────────────────────────────────────┤
 │ - capacity : number                                         │
 │ - map      : Map<key, Node>                                 │
 │ - head     : Node   (dummy sentinel, MRU side)              │
 │ - tail     : Node   (dummy sentinel, LRU side)             │
 ├─────────────────────────────────────────────────────────────┤
 │ + get(key)          : value | undefined                     │
 │ + put(key, value)   : void                                  │
 │ - _addToFront(node) : void   (insert just after head)       │
 │ - _remove(node)     : void   (splice node out)              │
 │ - _moveToFront(node): void   (_remove then _addToFront)     │
 │ - _evictLRU()       : void   (remove tail.prev + map delete)│
 └─────────────────────────────────────────────────────────────┘
                     │ owns many
                     ▼
 ┌───────────────────────────────────────┐
 │                Node                   │
 ├───────────────────────────────────────┤
 │  key   : any    (stored so eviction   │
 │                  can delete from map) │
 │  value : any                          │
 │  prev  : Node | null                  │
 │  next  : Node | null                  │
 └───────────────────────────────────────┘

  Structure at runtime (capacity 3, after put(a),put(b),put(c), get(a)):

  head ⇄ [a] ⇄ [c] ⇄ [b] ⇄ tail
  (MRU) ───────────────► (LRU, evicted next)
```

### 6. Full JavaScript implementation

Here is the complete, runnable hash-map + doubly-linked-list design. Every method is O(1). Read the comments — they explain the *why*, especially the subtle bits.

```javascript
'use strict';

/**
 * A single node in the doubly-linked list.
 * CRITICAL DETAIL: we store the `key` inside the node, not just the value.
 * Why? When we evict the tail node, we must ALSO delete it from the Map.
 * The Map is keyed by `key`, so the evicted node must carry its own key back
 * to us — otherwise we'd have a node to remove from the list but no way to
 * remove its entry from the Map. Forgetting this is the single most common
 * bug in LRU implementations: the list shrinks but the Map leaks forever.
 */
class Node {
  constructor(key, value) {
    this.key = key;      // needed for eviction -> map.delete(node.key)
    this.value = value;
    this.prev = null;
    this.next = null;
  }
}

class LRUCache {
  constructor(capacity) {
    if (!Number.isInteger(capacity) || capacity <= 0) {
      throw new Error('capacity must be a positive integer');
    }
    this.capacity = capacity;
    this.map = new Map();            // key -> Node   (O(1) lookup)

    // Dummy sentinels. head.next is the MRU node; tail.prev is the LRU node.
    // They never hold data — they exist so every real node has non-null
    // neighbours, which removes every null-check special case.
    this.head = new Node(null, null);
    this.tail = new Node(null, null);
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  // --- private helpers -----------------------------------------------------

  /** Splice a node OUT of the list. Works for any node because its neighbours
   *  are guaranteed non-null (worst case they are the sentinels). O(1). */
  _remove(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  /** Insert a node immediately after head, i.e. at the MRU front. O(1). */
  _addToFront(node) {
    node.prev = this.head;
    node.next = this.head.next;
    this.head.next.prev = node;   // old first node now points back at us
    this.head.next = node;        // head now points at us
  }

  /** Mark a node as most-recently-used: pull it out, re-insert at front. O(1). */
  _moveToFront(node) {
    this._remove(node);
    this._addToFront(node);
  }

  /** Evict the least-recently-used node (tail.prev) from BOTH structures. O(1). */
  _evictLRU() {
    const lru = this.tail.prev;      // the real node just before the tail sentinel
    this._remove(lru);
    this.map.delete(lru.key);        // this is why Node stores its key
    return lru.key;
  }

  // --- public API ----------------------------------------------------------

  /** Return the value for key and mark it MRU, or undefined on a miss. O(1). */
  get(key) {
    const node = this.map.get(key);
    if (node === undefined) return undefined;   // miss
    this._moveToFront(node);                     // touching it makes it MRU
    return node.value;
  }

  /** Insert or update key -> value, mark it MRU, evict LRU if over capacity. O(1). */
  put(key, value) {
    const existing = this.map.get(key);

    if (existing !== undefined) {
      // Update path: change the value and move to front.
      // IMPORTANT: do NOT grow size — the key already exists.
      existing.value = value;
      this._moveToFront(existing);
      return;
    }

    // Insert path: make a new node at the front and index it.
    const node = new Node(key, value);
    this._addToFront(node);
    this.map.set(key, node);

    // Only AFTER inserting do we check overflow. If we're over capacity,
    // evict the oldest. (We never exceed capacity by more than 1, so one
    // eviction is always enough.)
    if (this.map.size > this.capacity) {
      this._evictLRU();
    }
  }
}

// --- demo: the classic LeetCode sequence, capacity 2 -----------------------

function main() {
  const cache = new LRUCache(2);
  const log = (label, v) => console.log(`${label.padEnd(28)} => ${v}`);

  cache.put(1, 1);                          // cache: {1=1}
  cache.put(2, 2);                          // cache: {1=1, 2=2}
  log('get(1)  expect 1', cache.get(1));    // returns 1; 1 is now MRU, 2 is LRU
  cache.put(3, 3);                          // capacity full -> evicts key 2
  log('get(2)  expect undefined', cache.get(2)); // 2 was evicted -> miss
  cache.put(4, 4);                          // evicts key 1 (least recently used)
  log('get(1)  expect undefined', cache.get(1)); // 1 was evicted -> miss
  log('get(3)  expect 3', cache.get(3));    // returns 3
  log('get(4)  expect 4', cache.get(4));    // returns 4

  // Bonus: prove update does NOT grow size.
  const c2 = new LRUCache(2);
  c2.put('a', 1);
  c2.put('b', 2);
  c2.put('a', 100);                         // update existing key, size stays 2
  c2.put('c', 3);                           // evicts 'b' (a was just touched)
  log("get('a') expect 100", c2.get('a'));  // 100
  log("get('b') expect undefined", c2.get('b')); // evicted
  log("get('c') expect 3", c2.get('c'));    // 3
}

main();
```

**Expected output** (this is the well-known correct sequence — run it and match):

```
get(1)  expect 1             => 1
get(2)  expect undefined     => undefined
get(1)  expect undefined     => undefined
get(3)  expect 3             => 3
get(4)  expect 4             => 4
get('a') expect 100          => 100
get('b') expect undefined    => undefined
get('c') expect 3            => 3
```

**The pragmatic JS shortcut — a Map-only LRU.** Here is a detail that surprises interviewers: JavaScript's built-in `Map` **preserves insertion order**, and `Map` iteration yields keys oldest-first. That gives us "who is oldest?" for free. We can move a key to the "newest" position by deleting it and re-setting it (re-insertion pushes it to the end of the iteration order), and we can find the oldest with `map.keys().next().value`. The whole cache collapses to about 15 lines:

```javascript
class LRUCacheMap {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map();
  }

  get(key) {
    if (!this.map.has(key)) return undefined;
    const value = this.map.get(key);
    this.map.delete(key);      // remove...
    this.map.set(key, value);  // ...and re-insert to move it to the "newest" end
    return value;
  }

  put(key, value) {
    if (this.map.has(key)) this.map.delete(key); // so re-set moves it to the end
    this.map.set(key, value);
    if (this.map.size > this.capacity) {
      const oldestKey = this.map.keys().next().value; // first key = least recently used
      this.map.delete(oldestKey);
    }
  }
}
```

**Be honest about which one to use where.** In real production Node code, the `Map`-only version is what you'd actually write — it's shorter, correct, and O(1) because `Map.delete`/`set`/`has` are all O(1) and `keys().next()` is O(1). The hash-map + doubly-linked-list version exists to **demonstrate the data-structure understanding** in an interview. Many interviewers will explicitly say "don't use the language's ordered map — show me the linked list," precisely because the DLL version is the one that proves you understand *why* two structures are needed. Know both, and know when each is expected.

### 7. Design patterns / concepts used — and why (honestly)

This is more of a **data-structure problem than a design-pattern problem**, and a strong candidate says so out loud rather than forcing patterns onto it. Cramming in a Strategy or Factory here signals you're pattern-matching interview buzzwords instead of solving the problem. That said, a few genuine concepts apply:

| Concept | Where it appears | Why |
|---------|------------------|-----|
| **Composition** (recall [23 — Class Relationships](./23-class-relationships.md)) | `LRUCache` *has-a* `Map` and *has-many* `Node`s | The cache is built by composing simpler structures — the textbook "has-a" relationship. |
| **Facade** | The public `get` / `put` surface | Hides the messy pointer-rewiring behind two clean methods. Callers never see the linked list. |
| **Sentinel / Null Object** | The dummy `head` and `tail` | A guard-object idiom: dummy objects stand in for "the boundary" so callers never null-check. |
| **Encapsulation** | `_remove`, `_addToFront` are private-by-convention | Invariants (the list and map staying in sync) are protected inside the class. |

**Where it fits in real systems.** An LRU cache is the **eviction policy** that lives *inside* a larger cache. Recall from [59 — Caching in Depth](./59-caching-in-depth.md) that any bounded cache must decide who to evict; LRU is the default answer. Zoom out to [105 — Design a Distributed Cache](./105-hld-distributed-cache.md) and this exact structure is what runs on each individual cache node — the distributed layer handles sharding and replication across machines, but on any single node, an LRU (or a cousin like LFU) decides what stays in RAM. The little data structure you just built is a real component of systems serving millions of requests per second.

### 8. Extensions the interviewer will ask for

Interviewers love "now add X." Here's how this design absorbs the common ones:

**"Add a TTL (expiry) per entry."** Store an `expiresAt` timestamp on each `Node`. In `get`, after finding the node, check `if (Date.now() > node.expiresAt) { this._remove(node); this.map.delete(key); return undefined; }` — a *lazy* expiry that costs nothing until you touch the key. For eager cleanup, run a periodic sweep or keep a second structure ordered by expiry. The DLL design absorbs this cleanly because the node already exists as a place to hang metadata.

**"Make it thread-safe / async-safe."** Single-threaded Node rarely needs this, but if multiple async flows mutate the cache between `await`s you can get interleaving bugs. Wrap `get`/`put` in a mutex/lock so the read-modify-write of the pointers is atomic (recall the locking discussion in [123 — Design a Rate Limiter (LLD)](./123-lld-rate-limiter.md), which faces the same shared-counter race). The structure is unchanged; you're just serializing access to it.

**"Implement LFU (Least Frequently Used) instead."** Harder. LFU evicts the item accessed the *fewest* times, not the least recently. You need a **frequency count** per node plus **frequency buckets**: a `Map<frequency, DoublyLinkedList>` grouping all nodes of the same count, and a `minFreq` pointer to the smallest bucket. On access, bump the node's count and move it from its old bucket to the `count+1` bucket. To evict, remove the LRU node from the `minFreq` bucket (LFU breaks frequency ties with LRU). It's the same "map + linked lists + sentinels" toolkit, just one dimension richer — which is exactly why interviewers escalate to it.

**"Bound by memory instead of item count."** Instead of `map.size > capacity`, track a running `currentBytes` and each node stores its own `size`. On `put`, add the new node's size; then **evict in a loop** (`while (currentBytes > maxBytes) this._evictLRU()`) because one large insert might require kicking out several small entries. Eviction returns the freed node so you can subtract its bytes. Same skeleton, a different overflow predicate.

Every one of these reuses the core: a `Map` for O(1) lookup and a doubly-linked list of sentinel-guarded nodes for O(1) ordering. That reuse is the sign of a good core design.

---

## Visual / Diagram description

Below is the life of a `get(key="b")` on a full cache, drawn so you can reproduce it on a whiteboard. The key `"b"` is buried in the middle; the operation must pull it to the front, all in O(1).

```
 BEFORE  get("b")        head ⇄ [a] ⇄ [b] ⇄ [c] ⇄ tail
                         (MRU)                  (LRU)
 Map: { a:●, b:●, c:● }        ▲     ▲     ▲
         │   │   │             │     │     │
         └───┼───┼─────────────┘     │     │
             └───┼───────────────────┘     │
                 └─────────────────────────┘

 STEP 1  map.get("b")  -> jump straight to node [b]           (O(1) lookup)

 STEP 2  _remove([b]): rewire its two neighbours
            [a].next = [c]         [c].prev = [a]
         head ⇄ [a] ⇄ [c] ⇄ tail         ([b] now dangling)

 STEP 3  _addToFront([b]): splice between head and [a]
         head ⇄ [b] ⇄ [a] ⇄ [c] ⇄ tail

 AFTER   get returns [b].value; "b" is MRU, "c" is now next to be evicted
```

**How to read it.** Step 1 uses the Map to teleport to the node — no walking the list. Step 2 splices the node out using its `prev` and `next` (the reason we need a *doubly*-linked list). Step 3 re-inserts it right after the `head` sentinel. Because the sentinels are always present, no step ever touches a `null` pointer, and every step is a fixed number of pointer assignments — hence O(1).

---

## Real world examples

### Redis — `allkeys-lru` / `volatile-lru`

Redis, the most widely deployed in-memory cache, offers LRU as a `maxmemory-policy`. When Redis hits its memory ceiling, `allkeys-lru` evicts the least-recently-used key across the whole dataset. Interestingly, Redis uses an **approximate** LRU (sampling a handful of keys and evicting the oldest among the sample) rather than a perfect doubly-linked list, because tracking exact LRU order for millions of keys costs memory and CPU. This is a representative real-world tradeoff: exact LRU is beautiful in an interview, but at scale, "approximately LRU" is often good enough and much cheaper. (Behaviour per Redis documentation.)

### The `lru-cache` npm package (used by npm itself)

The `lru-cache` module is one of the most-downloaded packages on npm and powers caching inside npm's own CLI and countless Node tools. Under the hood it maintains, conceptually, exactly the structure in this document — an index plus a usage-ordered list — and layers on TTL, max-size-by-bytes, and disposal callbacks (all of the "extensions" from section 8). It's a real-world confirmation that the toy you built is the production skeleton. (Representative of its documented feature set.)

### CPU hardware caches (L1/L2/L3)

Your processor's caches evict cache lines using LRU-like policies implemented in silicon. Because true LRU is expensive in hardware, CPUs typically use approximations such as "pseudo-LRU" (a tree of bits) or NRU (not-recently-used). The concept is identical to what you've learned — keep what was touched recently, drop the coldest line — just realized in gates instead of pointers. (Conceptual; exact policy varies by chip.)

---

## Trade-offs

**Hash-map + doubly-linked list vs. the Map-only shortcut:**

| | Hash map + DLL | Map-only (insertion-order) |
|---|----------------|----------------------------|
| Lines of code | ~70 | ~15 |
| Time complexity | O(1) get/put | O(1) get/put |
| Demonstrates data-structure skill | Yes (the interview goal) | No — hides the mechanism |
| Language-agnostic | Yes | No — relies on JS Map ordering |
| Bug surface | Higher (pointer rewiring) | Lower |
| What to write in production JS | Rarely | Usually this |

**LRU as an eviction policy vs. alternatives:**

| Policy | Evicts | Good when | Weak when |
|--------|--------|-----------|-----------|
| **LRU** | Least recently *used* | Recent access predicts future access (temporal locality) | A one-time full scan flushes the whole cache ("cache pollution") |
| **LFU** | Least *frequently* used | Popularity is stable over time | Slow to adapt; an old hit stays sticky; more memory/complexity |
| **FIFO** | Oldest *inserted* | Simplicity; order doesn't matter | Ignores usage entirely — evicts hot items |
| **Random** | A random entry | Extremely cheap, no bookkeeping | Unpredictable hit rate |

**The sweet spot:** In an interview, build the hash-map + doubly-linked-list version — it's what the question is testing, and the sentinel trick keeps it correct. In production Node, reach for a battle-tested library (`lru-cache`) or the Map-only shortcut, and choose LRU as your default eviction policy unless you have a specific reason (stable popularity → LFU, a scan-heavy workload → something scan-resistant).

---

## Common interview questions on this topic

### Q1: "Why do you need both a hash map AND a linked list? Why not just one?"

**Hint:** The map gives O(1) lookup by key but has no concept of "oldest." The linked list maintains usage order (and lets you move a node to the front / evict the tail in O(1)) but finding a specific key in it would be an O(n) walk. Each covers the other's weakness; together, both `get` and `put` are O(1). State the two requirements (lookup vs. ordering) and assign one structure to each.

### Q2: "Why a doubly-linked list specifically — not a singly-linked list or an array?"

**Hint:** On a `get`, the accessed node is in the *middle* and must move to the front. Splicing a node out in O(1) requires rewiring both neighbours, so you need pointers to both `prev` and `next` — that's what "doubly" means. A singly-linked list can't reach the predecessor without an O(n) walk; an array can't remove from the middle without an O(n) shift. Middle removal is the crux.

### Q3: "Why store the key inside the Node when the Map is already keyed by it?"

**Hint:** When you evict the tail node, you must also delete its entry from the Map — and `map.delete(...)` needs the key. You arrive at the eviction with a *node* (from `tail.prev`), so the node must carry its own key back to you: `map.delete(evictedNode.key)`. Without it, the list shrinks but the Map leaks forever. This is the most commonly missed detail; name it proactively.

### Q4: "What do the dummy head/tail sentinels buy you?"

**Hint:** They guarantee every real node has non-null `prev` and `next`, which erases every edge case: empty list, single element, removing the head, removing the tail. Insert-at-front and remove-node become the same three lines regardless of position. It's the difference between clean code and a null-pointer crash on the empty-list case.

### Q5: "The interviewer says 'now make eviction based on total memory, not item count.' What changes?"

**Hint:** Track a running byte total and store each node's size. On `put`, add the size; then evict in a *loop* (`while (bytes > maxBytes)`), because a single large insert may need to remove several small entries — unlike count-based eviction where one insert triggers at most one eviction. The core map + DLL is untouched; only the overflow predicate changes.

---

## Practice exercise

### Build and test an LRU cache from scratch (~30 min)

Do NOT look at the code above while you work — that's the point.

1. In a file `lru.js`, implement `Node` and `LRUCache` (the hash-map + doubly-linked-list version) with **dummy head/tail sentinels**. Implement `get`, `put`, and the private helpers `_remove`, `_addToFront`, `_moveToFront`, `_evictLRU`.
2. Write a `main()` that runs the classic capacity-2 sequence: `put(1,1)`, `put(2,2)`, `get(1)`→`1`, `put(3,3)`, `get(2)`→miss, `put(4,4)`, `get(1)`→miss, `get(3)`→`3`, `get(4)`→`4`. Print each result and confirm it matches.
3. Add one test the classic sequence misses: `put` an **existing** key with a new value and assert the cache size did **not** grow.
4. Now write the ~15-line `Map`-only version and run the *same* test sequence through it. Confirm identical outputs.
5. **Stretch:** add a per-entry TTL to the DLL version (lazy expiry on `get`) and write a test using a short timeout that proves an expired key returns `undefined`.

**What to produce:** one file that prints the classic sequence's results from both implementations, plus your update-doesn't-grow-size assertion passing. If both implementations agree and every assertion holds, you've internalized the design.

---

## Quick reference cheat sheet

- **LRU cache** — a fixed-capacity cache that evicts the **least-recently-used** entry when full.
- **The two-structure design** — a hash **Map** (key → Node) for O(1) lookup + a **doubly-linked list** for O(1) usage-ordering. Neither alone suffices.
- **Map's job** — O(1) "find the value for this key" and jump straight to its node.
- **List's job** — maintain order (MRU at head, LRU at tail); move-to-front and evict-tail in O(1).
- **Why doubly-linked** — middle removal in O(1) needs pointers to *both* neighbours (`prev` and `next`).
- **Why not an array** — removing/moving from the middle is O(n) (shifting).
- **Why not singly-linked** — can't reach a node's predecessor in O(1).
- **Sentinel nodes** — dummy `head` and `tail` so every real node has non-null neighbours; kills all edge cases.
- **Store the key in the Node** — so eviction can `map.delete(node.key)`; forgetting this leaks the Map.
- **`get(key)`** — Map lookup → move node to front → return value (or `undefined`/`-1` on miss). O(1).
- **`put(key,value)`** — update-in-place (don't grow size) or insert-at-front; if `size > capacity`, evict tail. O(1).
- **JS Map shortcut** — `Map` preserves insertion order; delete+re-set to refresh, `keys().next().value` for oldest. ~15 lines, real-world choice.
- **Interview vs. production** — DLL version proves understanding; Map-only version is what you'd actually ship in Node.
- **Real uses** — Redis `allkeys-lru`, npm's `lru-cache`, OS page caches, CPU pseudo-LRU. It's the eviction policy inside real caches.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [59 — Caching in Depth](./59-caching-in-depth.md) | Establishes *why* caches need an eviction policy; LRU is the most common answer to "who gets evicted?" |
| **Next** | [105 — Design a Distributed Cache](./105-hld-distributed-cache.md) | Scales this single-node structure across many machines; each node still runs an LRU-like policy internally. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The requirements → objects → code method this document follows. |
| **Related** | [23 — Class Relationships](./23-class-relationships.md) | The `LRUCache` *has-a* `Map` and *has-many* `Node`s — the composition relationship in action. |
| **Related** | [123 — Design a Rate Limiter (LLD)](./123-lld-rate-limiter.md) | Another O(1)-obsessed LLD; shares the shared-state / locking concern raised in the thread-safety extension. |
