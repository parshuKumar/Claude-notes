# 92 — `ConcurrentHashMap` Internals and `CopyOnWriteArrayList`

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: `orderflow`'s idempotency cache — the `ConcurrentHashMap` that stops a retried `POST /orders` from charging the customer twice. This topic is where you discover that the map being thread-safe does not make the *code around it* thread-safe, and that a customer gets charged £40 twice because of two lines that both passed review.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **Java 8+ `ConcurrentHashMap` has no segments.** The Java 7 design — sixteen
> independently-locked sub-maps — was deleted. What replaced it is a single
> `Node<K,V>[] table` where **the lock granularity is one bin**.
>
> **Reads are lock-free.** `get(k)` performs a volatile read of the table slot and walks
> the bin's `next` chain, reading `volatile V val`. No lock. No CAS. No write to shared
> memory of any kind.
>
> **Writes take one of two paths.** If the target bin is **empty**, the write is a single
> **CAS** installing the new node — no lock at all. If the bin is **occupied**, the
> writer **`synchronized`s on the first node in that bin** and mutates under that
> monitor. Two writers hitting different bins never contend.
>
> **`size()` is an estimate.** The count lives in a `long baseCount` plus a striped
> `CounterCell[]` — the same design as `LongAdder` (Topic 95), for the same reason. It
> is summed on read, non-atomically.
>
> And the sentence the whole topic exists for:
>
> **Each OPERATION is atomic. Sequences of operations are not.**
>
> `map.containsKey(k)` is atomic. `map.put(k, v)` is atomic.
> `if (!map.containsKey(k)) map.put(k, v)` is **two** atomic operations with an
> unguarded gap between them, and it races exactly as hard as it would on a `HashMap`.

Two consequences you will use for the rest of your career:

1. **"Is this collection thread-safe?" is the wrong question.** The right one is "is my
   *invariant* protected?" A thread-safe collection protects its own internal structure.
   It knows nothing about the rule you are trying to enforce across two calls.
2. **The atomic composite methods exist for exactly this reason.** `putIfAbsent`,
   `computeIfAbsent`, `compute`, `merge` and `replace(k, old, new)` collapse a
   check-then-act into one bin-locked operation. They are not conveniences. They are the
   only correct form.

---

## The bridge from what you know

### `NO TYPESCRIPT ANALOGUE.`

Say it plainly, because reaching for one would cost you more than it buys.

**JavaScript's `Map` has no concurrency story at all.** There is no thread-safe variant,
no concurrent variant, no lock, and no need for any of them. Node's event loop gives you
run-to-completion: once your synchronous function starts, nothing else in the process
executes until it returns. So this:

```typescript
if (!cache.has(key)) {
  cache.set(key, await computeValue(key));    // note where the await is
}
```

is safe against the *synchronous* interleaving that this document is about. Nothing can
run between `has` and `set` if there is no `await` between them.

There is no library, no `Promise` pattern, and no `AsyncLocalStorage` trick that
corresponds to `ConcurrentHashMap`. There is nothing to transfer.

### The one thing that rhymes, and why it is not the same

The snippet above has a real bug, and it is worth naming because it is the closest thing
you already own to the mental model you need.

```typescript
if (!cache.has(key)) {                        // (1)
  cache.set(key, await computeValue(key));    // (2) -- suspends HERE
}
```

Between (1) and (2), the `await` **yields**. Another request can run, reach (1), also
see `false`, and also start computing. You end up computing twice and setting twice.
This is the "async cache stampede" and you have probably fixed it by caching the
*promise* rather than the value:

```typescript
let inflight = cache.get(key);
if (!inflight) {
  inflight = computeValue(key);   // store the promise, synchronously, before any await
  cache.set(key, inflight);
}
return inflight;
```

Hold on to that fix. It is structurally identical to the correct Java answer, and we
will use it in Example 2.

But the two situations differ in a way you must not blur:

| | Node | Java |
|---|---|---|
| Where can another task interleave? | **Only at an `await`.** You can point at the line. | **Between any two bytecodes.** There is no marker. |
| Is `if (!m.has(k)) m.set(k, v)` safe? | **Yes**, if there is no `await` between them | **No.** Never. Not for any map type. |
| Can two callbacks run at the same instant? | No. One thread. | **Yes.** Two cores, one object. |
| What does "thread-safe collection" buy you? | Nothing exists to buy | Structural integrity only. Not your invariant. |
| Cost of getting it wrong | duplicate work | **duplicate side effects** — a second wallet debit |

**That last row is the whole topic.** In Node the stampede costs you a redundant
computation. In Java the same shape costs you a customer's money, because the thing
inside the gap is not a pure function — it is a wallet debit.

**Verdict: NO ANALOGUE for the collection. A PARTIAL one for the *pattern*, and only if
you keep the "between any two bytecodes" correction firmly attached to it.**

### What you must actively unlearn

| Habit | Why it is fine in Node | Why it is a data-corruption bug in Java |
|---|---|---|
| "I used the concurrent one, so it's safe" | no such distinction exists | The map's internals are safe. Your two-call sequence is not. |
| Read a value, decide, write it back | nothing interleaves without `await` | The OS can suspend you between the read and the write. |
| `map.size()` in an `if` | exact and stable | An **estimate**, computed non-atomically, stale before you read it. |
| Iterate and mutate in the same loop | safe | Legal on a CHM and gives you a **weakly consistent** view — no exception, no snapshot, no guarantee. |
| Copy an array on write to keep readers happy | cheap at your sizes | `CopyOnWriteArrayList` is O(n) **per write**. N writes is O(n²) copying and O(n²) garbage. |

---

## What is this?

`java.util.concurrent` ships several collections designed for concurrent access. Four
matter for backend work, and one is a trap.

### `ConcurrentHashMap<K,V>`

The default concurrent map. Full `Map` implementation, plus:

- **Atomic composites** — `putIfAbsent`, `computeIfAbsent`, `computeIfPresent`,
  `compute`, `merge`, `replace(k, oldV, newV)`, `remove(k, v)`. Each is a single
  bin-locked operation.
- **Weakly consistent iterators** — never throw `ConcurrentModificationException`, but
  also no snapshot. They reflect the map at *some* point during the traversal.
- **No `null` keys or values.** `map.get(k) == null` would otherwise be ambiguous between
  "absent" and "present with null", and a follow-up `containsKey` cannot disambiguate
  because the answer could change in between. `HashMap` allows nulls; CHM throws
  `NullPointerException`. This bites during a migration.
- **Bulk parallel operations** — `forEach`, `search`, `reduce`, each with a
  `parallelismThreshold`. Above the threshold they run on `ForkJoinPool.commonPool()`,
  Topic 25's and Topic 91's shared pool. Pass `Long.MAX_VALUE` to stay sequential.
- **`mappingCount()`** — returns `long`; recommended over `size()` because a CHM can hold
  more than `Integer.MAX_VALUE` mappings. Both are estimates.
- **`ConcurrentHashMap.newKeySet()`** — the concurrent `Set`. Use this, not
  `Collections.synchronizedSet`.

### `CopyOnWriteArrayList<E>` and `CopyOnWriteArraySet<E>`

A list whose backing array is **replaced wholesale on every mutation**. Reads take no
lock and see a consistent immutable snapshot. Writes lock, copy the entire array,
mutate the copy, and publish it by writing a `volatile` reference.

**Reads: free** — one volatile array read, then plain indexing. **Writes: O(n) time and
O(n) allocation**, every single one. **Iterators: true snapshots** — never
`ConcurrentModificationException`, never later changes, and `Iterator.remove()` throws
`UnsupportedOperationException`. Correct for a listener list read on every request and
written at startup; catastrophic for anything written in a loop.

### `ConcurrentSkipListMap` / `ConcurrentSkipListSet` and `ConcurrentLinkedQueue` / `Deque`

The skip lists are the concurrent **sorted** structures. `TreeMap` is not thread-safe and
there is no concurrent red-black tree in the JDK, so you get `NavigableMap` semantics,
lock-free, O(log n) expected. Reach for them when you need `headMap`/`tailMap`/`firstKey`
under concurrency — a time-ordered event buffer, a priority index.

`ConcurrentLinkedQueue`/`Deque` are unbounded, lock-free (Michael–Scott), and
**non-blocking**: `poll()` returns `null` on empty rather than waiting. If you want a
consumer to *wait*, you want a `BlockingQueue` (Topic 93). And "unbounded" means what it
means there too: **the queue is your OOM.**

### `Collections.synchronizedMap(...)` — the trap

It wraps every method in `synchronized (mutex)`, which gives you **one global lock**
instead of per-bin locking — all readers and writers serialise, a scalability wall you
can see in a flame graph at 16 cores. It gives you **the same composition problem,
unfixed** — `containsKey`/`put` still races, with no atomic composites to reach for. And
it gives you **iteration that is not safe at all** unless you hold the mutex yourself:

```java
synchronized (m) {                       // REQUIRED. Documented. Universally forgotten.
    for (var e : m.entrySet()) { ... }   // without this: ConcurrentModificationException
}
```

There is no situation in modern code where `Collections.synchronizedMap` beats
`ConcurrentHashMap`. When you meet one, it is a Java 1.4-era habit nobody revisited.

---

## Why does it matter?

**1. The composition bug is invisible in review and expensive in production.**

`if (!cache.containsKey(key)) { cache.put(key, charge(order)); }` reads as obviously
correct. Both calls are on a documented thread-safe class. The reviewer checks the type,
sees `ConcurrentHashMap`, and approves. The bug fires once in every few thousand
requests, and each occurrence is a duplicate wallet debit.

**2. `orderflow` has exactly the shape.**

The order-placement endpoint accepts an `Idempotency-Key` header. Clients retry on
timeout. Load balancers retry on 5xx. Mobile clients retry when the user taps twice.
Every one of those produces two concurrent, identical requests — and the whole point of
the cache is to make the second one a no-op.

**3. Per-key lock registries are built on this.**

Topic 94's `OrderedLocks` used `locks.computeIfAbsent(key, k -> new ReentrantLock())`.
If that had been `get` then `put`, two threads would receive **two different lock
objects for the same key**, and every lock in the system would silently stop providing
mutual exclusion. The correctness of Topic 94 depends on the correctness of this topic.

**4. Because the internals are the interview.**

"Explain `ConcurrentHashMap`" is asked in almost every senior Java loop, and the
majority of candidates answer with the **Java 7** segment design they read in a blog.
Knowing that segments were removed in Java 8, and what replaced them, is a fast and
reliable signal.

---

## Machine-level reality

### The fields, and what each one means

```java
// java.util.concurrent.ConcurrentHashMap
transient volatile Node<K,V>[] table;       // the bins. Lazily allocated on first put.
private transient volatile Node<K,V>[] nextTable;  // non-null only DURING a resize
private transient volatile long baseCount;  // count, while uncontended
private transient volatile int sizeCtl;     // see the table below -- it is overloaded
private transient volatile int transferIndex;      // resize progress cursor
private transient volatile CounterCell[] counterCells;   // striped count under contention
```

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;             // volatile: this is what makes get() lock-free and correct
    volatile Node<K,V> next;    // volatile: same
}
```

**`volatile V val` and `volatile Node next` are the entire read story.** A reader needs
no lock because every field it traverses is a volatile read, and a volatile read
establishes the happens-before edge (Topic 87) with the writer's volatile write. That is
not an optimisation; it is what makes lock-free reading *correct* rather than merely
fast.

### `sizeCtl` — one int, four meanings

This field is overloaded, and knowing the encoding is a strong interview detail.

| `sizeCtl` value | Meaning |
|---|---|
| **positive**, table non-null | the **resize threshold**: `0.75 * table.length` |
| **positive**, table null | the **initial capacity** to use on first allocation |
| **`-1`** | a thread is currently **initialising** the table; others spin/yield |
| **negative, `< -1`** | the high 16 bits are a resize stamp; the low bits encode `1 + (number of threads currently helping with the transfer)` |

That last row is the interesting one, and it is unusual: **a resize in `ConcurrentHashMap`
is cooperative.** A writer that arrives during a resize does not wait. It calls
`helpTransfer` and moves bins itself. `sizeCtl` is how the map counts its helpers.

### `spread()` — and why the sign bit is reserved

```java
static final int HASH_BITS = 0x7fffffff;    // clears the sign bit

static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

The XOR-shift is the same high-bits-down spreading you met in `HashMap` (Topic 12), and
for the same reason: the bin index is `(n - 1) & hash`, so only the low bits select a
bin, and a hash whose entropy lives in the high bits would collide catastrophically.

The `& HASH_BITS` is new, and it is load-bearing. **Clearing the sign bit reserves all
negative hash values for control nodes:**

| Constant | Value | Node type | What it means |
|---|---|---|---|
| `MOVED` | `-1` | `ForwardingNode` | This bin has been transferred to `nextTable`. A resize is in progress. Follow the pointer, or help. |
| `TREEBIN` | `-2` | `TreeBin` | This bin is a red-black tree, not a linked list. The `TreeBin` object is the lock target. |
| `RESERVED` | `-3` | `ReservationNode` | A placeholder held while `computeIfAbsent`'s mapping function runs on an **empty** bin. |

`RESERVED` is the one to remember, because it explains Trap 2. When `computeIfAbsent`
finds an empty bin, it cannot leave the bin empty while your lambda runs — another
thread would see "absent" and compute the same value. So it CASes in a
`ReservationNode`, **synchronizes on it**, runs your function, and replaces the
reservation with the real node. Your lambda executes **while a monitor is held on a node
that is in the map.**

### The `get` path, step by step

```
get(key):
  1. h = spread(key.hashCode())
  2. tab = table                       (volatile read)
  3. f = tabAt(tab, (n-1) & h)         (Unsafe.getReferenceAcquire -- a volatile read)
  4. if f == null            -> return null
  5. if f.hash == h && keys equal -> return f.val    (volatile read)
  6. if f.hash < 0           -> f.find(h, key)       (forwarding node or tree bin)
  7. otherwise walk f.next, comparing                (all volatile reads)
```

**Zero writes to shared memory. Zero locks. Zero CAS.** Every core keeps the cache line
in Shared state; nothing is invalidated (Topic 95's MESI walk-through). This is why CHM
reads scale close to linearly with cores while a `synchronized` map does not at all.

Step 6 is worth pausing on: a *reader* that arrives during a resize follows the
`ForwardingNode` into `nextTable` and reads from there. It does not block, does not
help, and does not fail.

### The `put` path, step by step

```
putVal(key, value):
  loop:
    tab = table
    if tab is null                     -> initTable(); continue     (sizeCtl == -1 guard)

    f = tabAt(tab, i = (n-1) & h)

    if f == null                       -> casTabAt(tab, i, null, new Node(...))
                                          SUCCESS -> break. NO LOCK WAS TAKEN.
                                          FAIL    -> continue (someone beat us)

    if f.hash == MOVED                 -> tab = helpTransfer(tab, f); continue
                                          (we join the resize instead of waiting)

    otherwise:
      synchronized (f) {               // <-- THE LOCK. The first node of THIS bin.
          re-check tabAt(tab, i) == f  // the bin may have changed before we acquired
          walk the chain / tree
          replace an existing val, or append a new Node
          binCount = length of chain
      }
      if binCount >= TREEIFY_THRESHOLD (8) -> treeifyBin(tab, i)

  addCount(1, binCount)                // updates baseCount or a CounterCell; may resize
```

Five facts fall out, each of which is an interview answer.

**Fact 1 — an insert into an empty bin takes no lock.** One CAS. On a sparsely populated
map, most inserts are lock-free. This is why CHM's write throughput does not fall off a
cliff the way a single-lock map's does.

**Fact 2 — the lock is the first node of the bin, and it is a plain `synchronized`
monitor.** Not a `ReentrantLock`, not a segment, not a striped lock array. That has a
direct diagnostic consequence, below.

**Fact 3 — the re-check inside the `synchronized` block is mandatory.** Between reading
`f` and acquiring its monitor, another thread may have removed `f` or replaced the bin
head, so the code re-reads `tabAt(tab, i)` and restarts if it changed — the same
"validate after acquiring" discipline as `StampedLock` (Topic 94), inside the JDK.

**Fact 4 — treeification has two conditions, and people remember one.** A bin becomes a
red-black tree at 8 nodes **only if the table is at least 64 buckets**
(`MIN_TREEIFY_CAPACITY`); below that the map **resizes instead**, because a long chain in
a small table means the table is too small rather than the hashes adversarial. Untreeify
is at 6, and the gap is deliberate hysteresis against a bin flapping.

**Fact 5 — a resize is cooperative and concurrent.** A writer meeting a `ForwardingNode`
calls `helpTransfer` and moves a range of bins itself; several threads transfer disjoint
ranges tracked by `transferIndex`. No stop-the-world moment, no global lock.

### What CHM contention looks like in a thread dump

Because the bin lock is an **intrinsic monitor**, contention shows up as `BLOCKED`, not
as `WAITING (parking)`. That is the opposite of every `java.util.concurrent` lock you
met in Topic 94, and it is a genuinely useful fingerprint.

*Illustration of the format, not captured output. `<n>`, `<tid>` and `0x...` are
placeholders; your own dump will have real values.*

```
"http-nio-8080-exec-<n>" #<tid> daemon prio=5 os_prio=31 tid=0x... nid=0x... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
        at java.base/java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:<line>)
        - waiting to lock <0x...> (a java.util.concurrent.ConcurrentHashMap$ReservationNode)
        at com.orderflow.orders.IdempotencyCache.recordOrCharge(IdempotencyCache.java:<line>)
```

| What the monitor class is | What it tells you |
|---|---|
| `ConcurrentHashMap$Node` | Ordinary bin contention. Several threads writing keys that hash to the same bin. |
| `ConcurrentHashMap$TreeBin` | A treeified bin — at least 8 collisions in a table of 64+. Suspect a bad `hashCode` (Topic 13) before you suspect load. |
| `ConcurrentHashMap$ReservationNode` | **A `computeIfAbsent` mapping function is running and holding the bin.** If this appears with a *long* duration, your lambda is doing I/O under a lock. That is Trap 3. |

**Many threads `BLOCKED` on `ConcurrentHashMap$Node` with the same monitor identity
means your keys are colliding**, not that the map is slow. Check `hashCode` first.

### `size()` — a striped counter, and why it cannot be exact

```java
// java.util.concurrent.ConcurrentHashMap
private transient volatile long baseCount;
private transient volatile CounterCell[] counterCells;

@jdk.internal.vm.annotation.Contended
static final class CounterCell {
    volatile long value;
}

final long sumCount() {
    CounterCell[] cs = counterCells;
    long sum = baseCount;
    if (cs != null) {
        for (CounterCell c : cs) {
            if (c != null) sum += c.value;
        }
    }
    return sum;
}
```

**This is `LongAdder`'s design, and it is there for `LongAdder`'s reason** (Topic 95). A
single `AtomicLong` count would be a global contention point that undoes the entire
per-bin locking scheme: every write to any bin would CAS the same cache line, and
throughput would collapse with core count. So the count is striped across cells indexed
by the thread's probe, and `CounterCell` is `@Contended` so two cells cannot share a
64-byte cache line (Topic 96 — this is the concrete reason that annotation exists).

Three properties follow, and you must know all three. `size()` is **O(number of cells)**,
bounded by core count — cheap, not free. It is **not atomic**: cells are read one at a
time, so a concurrent `put` into a cell already passed is not counted. Therefore
**`size()` must never appear in a compare-and-act** — `if (map.size() < limit) map.put(...)`
is a check-then-act race with an inaccurate check layered on top; for a bounded cache use
Caffeine (Topic 15) or a `Semaphore` (Topic 97).

For logging, metrics and capacity dashboards all three properties are fine. For a
correctness decision, none of them are.

### `computeIfAbsent` holds the bin lock while your lambda runs

This is the single most consequential implementation detail in the class, and it is
stated plainly in the Javadoc: *"Some attempted update operations on this map by other
threads may be blocked while computation is in progress."*

Two consequences.

**Consequence A — do not do slow things in the mapping function.** A JDBC call, an HTTP
call or a sleep inside `computeIfAbsent` holds an intrinsic monitor on a node that is in
the map for the whole duration. Every other write to that bin blocks, and so does every
resize helper that reaches it. Under load this is a pile of `BLOCKED` threads on
`ConcurrentHashMap$ReservationNode`.

**Consequence B — the mapping function must not touch the same map.** On Java 8 it could
hang or corrupt; from Java 9 the implementation detects many such cases and throws:

```
java.lang.IllegalStateException: Recursive update
```

**"Many", not "all".** The detection triggers when the recursive update collides with
the bin being computed. A recursive update to a *different* bin may succeed, may throw,
or may deadlock depending on the interleaving and on whether a resize is in flight. This
is an implementation behaviour rather than a specified one — **verify on your own JDK
with the proof in the Hands-on section rather than trusting any blog post, including
this one.** The safe rule is absolute: **the mapping function must not touch the map it
is being called on.**

### `CopyOnWriteArrayList` internals

```java
// java.util.concurrent.CopyOnWriteArrayList
private transient volatile Object[] array;      // the ENTIRE state
final transient Object lock = new Object();     // JDK 11+: a plain monitor
                                                // Java 8: a ReentrantLock field
```

The lock object's type changed across releases (it was a `ReentrantLock` in Java 8 and a
plain `Object` monitor later). Confirm on your JDK with the `javap -p` command in the
Hands-on section rather than memorising it.

Every mutation follows one shape:

```java
public boolean add(E e) {
    synchronized (lock) {
        Object[] es = getArray();
        int len = es.length;
        es = Arrays.copyOf(es, len + 1);   // <-- O(n) TIME and O(n) ALLOCATION
        es[len] = e;
        setArray(es);                      // <-- volatile write: publishes atomically
    }
    return true;
}
```

And every read is:

```java
public E get(int index) {
    return elementAt(getArray(), index);   // one volatile read, then plain indexing
}
```

Four properties, and the fourth is the one that hurts:

1. **Readers never block and never see a torn state.** They read the volatile array
   reference once and then work with an immutable snapshot.
2. **The iterator is a true snapshot**, taken at `iterator()` time. It never throws
   `ConcurrentModificationException` — and it never sees anything added afterwards.
   `remove()` on it throws `UnsupportedOperationException`.
3. **The volatile write in `setArray` is the safe publication** (Topic 88). Everything
   the writer did to the new array before the write is visible to any reader that sees
   it.
4. **Every write allocates a new array of the full size.** Adding N elements one at a
   time copies `1 + 2 + ... + N` references — **O(n²) total copying and O(n²) total
   bytes allocated**. At N = 10,000 that is roughly 50 million reference copies and
   400 MB of garbage on a 64-bit JVM with compressed oops. That is an allocation-rate
   problem (Topic 68), a GC problem (Topic 71), and a latency problem, all from one data
   structure choice.

**`addAll` copies once for the whole batch.** If you must build a `CopyOnWriteArrayList`,
build a plain `ArrayList` first and pass it to the constructor or to `addAll`. One copy
instead of N.

### `[JAVA 25]` compact object headers

`-XX:+UseCompactObjectHeaders` shrinks the object header, which changes the size of a
`Node` and of a `CounterCell`, and therefore how many of them fit in a 64-byte cache
line. It does not change any semantics in this document. It **does** change Topic 96's
padding arithmetic, and it changes the memory-per-entry figure you would use for
capacity planning. Re-verify any layout number with JOL under the flag you actually run
with.

---

## Concurrency trace

Two traces, both **before** any correct code. The first is the assigned drill and the
headline bug of this topic.

### Trace 1 — check-then-act on the idempotency cache charges the customer twice

**The setup.** `orderflow` accepts `POST /orders` with an `Idempotency-Key` header. The
service keeps a `ConcurrentHashMap<String, OrderId>` so a retried request returns the
original order instead of placing a second one.

```java
private final Map<String, OrderId> processed = new ConcurrentHashMap<>();

public OrderId place(String idempotencyKey, PlaceOrderCommand cmd) {
    if (processed.containsKey(idempotencyKey)) {      // (1) CHECK
        return processed.get(idempotencyKey);
    }
    OrderId id = doPlace(cmd);                        // (2) ACT -- debits the wallet
    processed.put(idempotencyKey, id);                // (3) RECORD
    return id;
}
```

The customer taps "Pay" once. Their phone's HTTP client times out at 2 seconds and
retries with the **same** idempotency key. Both requests are now in flight against the
same pod.

| Step | Thread A — `http-nio-8080-exec-31` (order intake, key `IK-77c2`) | Thread B — `http-nio-8080-exec-48` (client retry, same key `IK-77c2`) | Shared state / outcome |
|---|---|---|---|
| 1 | `processed.containsKey("IK-77c2")` → **false** | — | key absent |
| 2 | — | `processed.containsKey("IK-77c2")` → **false** | key absent — **both checks have now passed** |
| 3 | enters `doPlace(cmd)` | — | — |
| 4 | reserves 1 unit of SKU-1001 | enters `doPlace(cmd)` | stock 12 → 11 |
| 5 | **debits wallet £40.00**, commits | reserves 1 unit of SKU-1001 | balance −40.00 · stock 11 → 10 |
| 6 | creates order 8812 | **debits wallet £40.00**, commits | **balance −80.00 — the customer has been charged twice** |
| 7 | `processed.put("IK-77c2", 8812)` | creates order 8813 | map now holds 8812 |
| 8 | returns 8812 | `processed.put("IK-77c2", 8813)` | map now holds **8813** — A's entry silently overwritten |
| 9 | — | returns 8813 | the client sees 8813 and discards it as the retry response |
| 10 | — | — | **Two orders, two stock reservations, two debits, one customer, one intent.** The cache now maps the key to the order the client never saw. |

**Outcome, in business terms.**

The customer paid £80 for a £40 order. Two order rows exist. Two units of a
single-unit-per-customer promotional SKU are reserved, so the stock figure is wrong by
one and a genuine customer later gets an out-of-stock error for an item that is
physically present.

Nothing throws. Nothing logs at `WARN` or above. Both requests returned HTTP 201 with a
valid order id. The access log shows two clean 201s from the same client within two
seconds — which looks exactly like a customer who ordered twice on purpose.

The idempotency cache, whose entire reason for existing was to prevent this, contains
one entry mapping the key to order **8813** — the one the client threw away. So a *third*
retry would correctly return 8813, and the reconciliation engineer looking at the cache
three days later will find it in a perfectly consistent state and conclude the cache
worked.

The bug is found by a customer support ticket and a `SELECT user_id, count(*) FROM
payments WHERE created_at > now() - interval '1 day' GROUP BY 1 HAVING count(*) > 1`.

**And the part that makes it senior-level:** the fix below removes the *race*. It does
**not** make the endpoint idempotent, because there are three pods behind the load
balancer and each has its own map. Two retries landing on two different pods reproduce
this trace exactly, with no race inside either JVM. Hold that thought until Example 2.

### Trace 2 — `computeIfAbsent` re-entering the same map

**The setup.** Someone fixes Trace 1 with `computeIfAbsent`, which is the right verb. But
the mapping function needs the customer's tier to price the order, and the tier lookup
is itself cached — **in the same map**, because "it's just a cache, one map is simpler".

```java
private final Map<String, Object> cache = new ConcurrentHashMap<>();

public OrderId place(String key, PlaceOrderCommand cmd) {
    return (OrderId) cache.computeIfAbsent("order:" + key, k -> {
        Tier tier = (Tier) cache.computeIfAbsent(                  // <-- SAME MAP
                "tier:" + cmd.userId(), t -> tierService.lookup(cmd.userId()));
        return doPlace(cmd, tier);
    });
}
```

| Step | Thread A — `http-nio-8080-exec-31` | Thread B — `http-nio-8080-exec-48` | Map internals / outcome |
|---|---|---|---|
| 1 | `computeIfAbsent("order:IK-77c2")` → bin 19 is empty | — | A CASes a `ReservationNode` into bin 19 |
| 2 | `synchronized` on that `ReservationNode`; begins running the lambda | — | **A holds the monitor for bin 19** |
| 3 | lambda calls `computeIfAbsent("tier:55")` → `spread("tier:55")` also lands in **bin 19** | — | same bin, and A already holds its monitor |
| 4 | re-enters the bin: `synchronized` is reentrant, so A does **not** block — it proceeds to mutate a bin whose structure the outer call is mid-way through changing | — | **the map's internal invariants are now being violated from inside** |
| 5 | on Java 9+ the implementation detects the recursive update and throws `IllegalStateException: Recursive update` | — | the reservation is removed; A's request fails with a 500 |
| 5' | *(alternative, and this is the one that costs a night)* the recursion lands in a **different** bin and is not detected | `computeIfAbsent("order:IK-9911")` → also bin 19 → `synchronized` on A's `ReservationNode` → **BLOCKS** | B is `BLOCKED (on object monitor)` on `ConcurrentHashMap$ReservationNode` |
| 6 | still inside the nested lambda, now waiting on `tierService.lookup` — a 400 ms HTTP call | still blocked | **B is blocked on a monitor held across a network call** |
| 7 | 40 more requests arrive whose keys hash to bin 19 | all block | 41 request threads `BLOCKED` on one node |
| 8 | tier service degrades to 3 s under its own load | all still blocked | Tomcat's 200 threads drain |
| 9 | — | — | `GET /products`, which touches none of this, returns 503 |

**Outcome, in business terms.**

In the detected case, a random subset of order placements fail with HTTP 500 and an
exception message — `Recursive update` — that means nothing to anyone reading the alert,
and which does not mention the map, the key, or the lambda.

In the undetected case, the service hangs on one bin. `jcmd Thread.print` shows dozens
of threads `BLOCKED (on object monitor)` on a
`java.util.concurrent.ConcurrentHashMap$ReservationNode`, and **there is no
`Found one Java-level deadlock` section**, because this is not a cycle — it is one
monitor held for a long time by a thread that is waiting on a network. The deadlock
detector is silent. The service is down. Every dashboard shows a healthy database and a
healthy JVM.

The customer sees order placement failing intermittently while the product catalogue,
the login flow and the wallet balance endpoint all work perfectly — which is the worst
possible shape for a support conversation.

---

## Example 1 — minimal

Two shapes, both wrong, then the atomic forms.

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public final class SkuCounters {

    private final Map<String, Integer> soldPerSku = new ConcurrentHashMap<>();

    /** WRONG. Two atomic operations. Increments are lost. */
    public void recordSaleBroken(String sku) {
        Integer current = soldPerSku.get(sku);          // (1) READ
        soldPerSku.put(sku, current == null ? 1 : current + 1);   // (2) WRITE
    }

    /** CORRECT. One atomic operation, under the bin lock. */
    public void recordSale(String sku) {
        soldPerSku.merge(sku, 1, Integer::sum);
    }

    /** Also correct, and clearer when the update is more than an addition. */
    public void recordSaleOfQuantity(String sku, int qty) {
        soldPerSku.compute(sku, (k, v) -> v == null ? qty : v + qty);
    }
}
```

`recordSaleBroken` is `i++` wearing a map (Topic 87). Read, modify, write; three
operations, interleavable at each boundary. Two threads both read `7`, both write `8`,
and one sale vanishes. That the map is a `ConcurrentHashMap` changes nothing at all —
each individual call was already atomic, and the bug is in the gap between them.

### The atomic composites, and when to use which

```java
map.putIfAbsent(k, v);              // insert only if absent. Returns the EXISTING value,
                                    // or null if it inserted. Value must be pre-computed.

map.computeIfAbsent(k, key -> f()); // insert only if absent. f() runs UNDER THE BIN LOCK
                                    // and only when needed. Returns the value either way.

map.computeIfPresent(k, (k2, v) -> g(v));   // update only if present; null return removes

map.compute(k, (k2, v) -> h(v));    // always runs; v is null if absent; null return removes

map.merge(k, initial, (old, given) -> combine(old, given));  // the counter idiom

map.replace(k, expected, updated);  // CAS on a map entry. True only if the current value
                                    // .equals(expected). The composable primitive.
```

Choose by cost: `putIfAbsent` when the value is cheap to build and always needed;
`computeIfAbsent` when it is expensive and must be built at most once; `merge` for a
counter; `compute` for a conditional update that must see `null` for absent; and
`replace(k, old, new)` for "update only if it still equals what I read", where the retry
loop is yours to write.

**Do not use `computeIfAbsent` when the value is cheap.** Constructing a
`new ReentrantLock()` under the bin lock is fine — it is an allocation. Calling a
database under it is not. `putIfAbsent` with a pre-built value never holds a lock over
your code at all; the cost is that you build the value even when it turns out to be
unnecessary. That trade is usually right for cheap values and always wrong for expensive
ones.

### The `replace` retry loop, written correctly

When the update depends on the old value and you refuse to hold the bin lock:

```java
public void applyDiscount(String sku, int pence) {
    Integer current;
    Integer updated;
    do {
        current = prices.get(sku);
        if (current == null) {
            return;                            // nothing to discount
        }
        updated = Math.max(0, current - pence);
    } while (!prices.replace(sku, current, updated));   // retry until nobody raced us
}
```

That is a CAS loop (Topic 95) over a map entry. **It can spin** — under heavy
contention on one key each retry is a wasted round trip, so bound the retries and fail
rather than spinning forever (Topic 95's Trap 5). **It depends on `equals`**, not `==`,
so a broken `equals` (Topic 13) silently does the wrong thing. And **`compute` is usually
clearer**: use the loop only when the mapping function is expensive enough that holding
the bin lock is unacceptable.

---

## Example 2 — production scenario (on the project spine)

### The constraints

From the Topic 65 baseline:

- 100,000 products, 1,000,000 orders, 5,000,000 order lines.
- k6 open-model arrival, 400 rps: 70% catalogue read, 20% order read, **10% order
  placement** — so roughly 40 order placements per second.
- **Three pods** behind the load balancer, each a separate JVM.
- Tomcat request threads: 200 per pod. HikariCP maximum pool size: 20 per pod.
- Client retry policy: 2-second timeout, one immediate retry. The load balancer also
  retries once on a 502.
- Clients send an `Idempotency-Key` header, a UUID generated per user intent.
- Recorded baseline p50/p95/p99 committed in `/docs/java/baselines/`.

At 40 placements per second with a retry on roughly 1% of requests, you get about **24
duplicate-key pairs per minute**. The window for Trace 1 is the duration of `doPlace` —
a transaction that reserves stock and debits a wallet, tens of milliseconds. So the race
is not rare. It is continuous.

### The code that ships and charges customers twice

```java
package com.orderflow.orders;

@Service
public class IdempotentOrderService {

    private final Map<String, OrderId> processed = new ConcurrentHashMap<>();
    private final OrderPlacementService placement;

    public OrderId place(String idempotencyKey, PlaceOrderCommand cmd) {
        if (processed.containsKey(idempotencyKey)) {      // CHECK
            return processed.get(idempotencyKey);         // ...and a second read
        }
        OrderId id = placement.place(cmd);                // ACT: reserves stock, debits wallet
        processed.put(idempotencyKey, id);                // RECORD
        return id;
    }
}
```

There are **four** distinct defects here, and only the first is the headline.

**Defect 1 — check-then-act.** Trace 1. Two threads pass the check and both charge.

**Defect 2 — `containsKey` then `get` is itself a second race.** Even if the entry
exists at the check, it could be evicted before the `get`, and `get` would return
`null`, which this method then returns as an `OrderId`. Always use a single `get` and
test the result for null, never `containsKey` followed by `get`.

**Defect 3 — the map is unbounded.** One entry per idempotency key, forever, for the
life of the JVM. At 40 placements per second that is 3.4 million entries per day,
retained by a `static`-lifetime field. This is Topic 79's leak, exactly: not an
allocation problem, a **reachability** problem. It will not show up in a load test that
runs for fifteen minutes. It will show up as an `OutOfMemoryError` on day four.

**Defect 4 — the map is per-JVM.** Three pods, three maps. A retry that lands on a
different pod finds an empty cache and places the order again, with no race involved at
all. **The in-memory cache cannot be the idempotency guarantee.** Fixing the race makes
the endpoint correct *within one pod* and leaves it broken across the fleet.

### Fix 1 — remove the race, with the promise-caching idiom

The naive atomic fix is `computeIfAbsent(key, k -> placement.place(cmd))`. It removes
the race — but it runs a full order-placement transaction, including a database round
trip and a wallet debit, **while holding an intrinsic monitor on a node in the map**.
That is Trap 3, and at 40 rps with keys colliding it produces the `BLOCKED` pile-up from
Trace 2.

The correct in-JVM shape is the one you already know from Node: **cache the in-flight
work, not the result.**

```java
package com.orderflow.orders;

import java.util.concurrent.*;

@Service
public class IdempotentOrderService {

    /** Key -> the in-flight or completed placement. Bounded and evicted; see Fix 2. */
    private final ConcurrentMap<String, CompletableFuture<OrderId>> inFlight =
            new ConcurrentHashMap<>();

    private final OrderPlacementService placement;
    private final ExecutorService orderExecutor;      // Topic 90/91: bounded, named, metered

    public OrderId place(String idempotencyKey, PlaceOrderCommand cmd) {

        CompletableFuture<OrderId> mine = new CompletableFuture<>();

        // ONE atomic operation. Returns the existing future, or null if we inserted.
        CompletableFuture<OrderId> existing = inFlight.putIfAbsent(idempotencyKey, mine);

        if (existing != null) {
            // Someone else owns this key. Wait for THEIR result. We do no work.
            return join(existing);
        }

        // We own the key. Nothing above held a lock while we did.
        try {
            OrderId id = placement.place(cmd);
            mine.complete(id);
            return id;
        } catch (RuntimeException e) {
            // CRITICAL: a failed placement must not poison the key forever.
            inFlight.remove(idempotencyKey, mine);    // two-arg remove: only if still ours
            mine.completeExceptionally(e);            // release anyone waiting on us
            throw e;
        }
    }

    private OrderId join(CompletableFuture<OrderId> f) {
        try {
            return f.get(5, TimeUnit.SECONDS);        // BOUNDED. Never bare join(). Topic 91.
        } catch (TimeoutException e) {
            throw new IdempotentRequestInProgressException();   // -> HTTP 409, retryable
        } catch (ExecutionException e) {
            throw asRuntime(Futures.rootCause(e));    // Topic 91: unwrap, always
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ServiceInterruptedException(e);
        }
    }
}
```

Seven decisions in that method, each of which you should be able to defend:

1. **`putIfAbsent`, not `computeIfAbsent`.** The value — an empty `CompletableFuture` —
   is an allocation, so building it unconditionally costs nothing. In exchange, **no
   lock is held while `placement.place` runs**. The expensive work happens entirely
   outside the map.
2. **The value is a future, not the result.** The second caller finds the key present
   *immediately*, before the first has finished, and waits on the first caller's result
   instead of starting its own. That is the Node promise-caching fix, transplanted.
3. **`existing != null` means we lost the race**, and the loser does no work at all.
   `putIfAbsent` returning the existing value rather than a boolean is what makes this
   one operation instead of two.
4. **The wait is bounded.** `f.get(5, SECONDS)`, never `f.join()`. An unbounded wait here
   would let one stuck placement consume every Tomcat thread that retries that key
   (Topic 91's Trap 5).
5. **Failure removes the key.** Without `inFlight.remove(key, mine)`, a transient failure
   caches a permanently-failed future and **every** future retry of that idempotency key
   fails forever. This is the single most common bug in hand-rolled request coalescing.
6. **The two-argument `remove(key, value)`.** It removes only if the mapped value is
   still `mine`. The one-argument `remove(key)` would delete a *different* thread's
   entry if the key had already been re-inserted — a check-then-act bug hiding inside
   the fix for a check-then-act bug.
7. **`completeExceptionally` before throwing.** Anyone already parked in `join` is
   released with the real cause rather than timing out five seconds later.

### Fix 2 — bound the map, because unbounded is a leak

`inFlight` is `static`-lifetime state that only grows. Two options, and the second is
the one to ship.

**Option A — evict on completion, with a delay** —
`scheduler.schedule(() -> inFlight.remove(key, mine), 60, TimeUnit.SECONDS)`. Simple, but
it is a hand-rolled cache with a hand-rolled eviction policy and no metrics, and the
scheduler is another executor to size and shut down.

**Option B — use a real cache.** Topic 15's conclusion, applied:

```java
private final Cache<String, CompletableFuture<OrderId>> inFlight = Caffeine.newBuilder()
        .maximumSize(100_000)
        .expireAfterWrite(Duration.ofMinutes(10))     // > the client's total retry window
        .recordStats()                                // Topic 118: hit ratio, evictions
        .build();

// Caffeine's asMap() view IS a ConcurrentMap, so the idiom above is unchanged:
ConcurrentMap<String, CompletableFuture<OrderId>> map = inFlight.asMap();
CompletableFuture<OrderId> existing = map.putIfAbsent(idempotencyKey, mine);
```

You get bounded memory, a TTL derived from the retry window rather than from taste, and
hit/miss/eviction metrics for free. **The eviction policy is a product decision** — it
is "how long after a request do we still promise not to duplicate it" — so write the
number down with its reason, not as a magic constant.

### Fix 3 — the one that actually makes the endpoint idempotent

Everything above fixes the **race**. None of it fixes **Defect 4**, and this is what
separates a senior answer from a correct one. Three pods, three caches: a client retry
routed elsewhere finds an empty map, takes the fast path with no contention whatsoever,
and places a second order. No race, no interleaving, and no amount of `ConcurrentHashMap`
expertise helps.

**The idempotency guarantee has to live where the data lives:**

```sql
ALTER TABLE orders
  ADD COLUMN idempotency_key varchar(64);

CREATE UNIQUE INDEX orders_idempotency_key_uk
  ON orders (idempotency_key)
  WHERE idempotency_key IS NOT NULL;
```

```java
@Transactional
public OrderId place(String idempotencyKey, PlaceOrderCommand cmd) {
    try {
        return placement.placeWithKey(cmd, idempotencyKey);
    } catch (DataIntegrityViolationException e) {
        // The database rejected the duplicate. It is the only arbiter across pods.
        return orders.findByIdempotencyKey(idempotencyKey)
                     .map(Order::id)
                     .orElseThrow(() -> e);      // genuinely a different constraint
    }
}
```

Now the in-memory map is what it should always have been: **an optimisation that avoids
a database round trip in the common case, not a correctness mechanism.** If it is empty,
cold, evicted or on the wrong pod, the endpoint is still correct — it just costs one
more query.

**Say this in an interview and you will separate yourself immediately:**

> "I would fix the check-then-act with `putIfAbsent` and cache the in-flight future so
> the duplicate waits rather than duplicating the work. But I would not call that
> idempotency. An in-memory map is per-JVM, and we run three pods — a retry on another
> pod bypasses it entirely with no race at all. The guarantee has to be a unique
> constraint on the idempotency key in the database, because that is the only thing all
> three pods agree on. The map is a latency optimisation in front of it."

That answer moves the conversation from "can you use `putIfAbsent`" to "do you
understand where a guarantee has to live", which is what the question is actually
probing.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `containsKey` then `put` (the headline)

**Wrong:**

```java
if (!processed.containsKey(key)) {
    processed.put(key, charge(order));
}
```

**Exact symptom:** duplicate side effects at a low, load-dependent rate. In `orderflow`:

- `SELECT user_id, order_total, count(*) FROM payments WHERE created_at > now() -
  interval '1 hour' GROUP BY 1,2 HAVING count(*) > 1` returns rows.
- Two `orders` rows with the same `user_id`, the same total, and timestamps a few
  milliseconds apart.
- **No exception, no log above `INFO`, two HTTP 201s in the access log.**
- The rate scales with concurrency, so it is invisible on a laptop and continuous in
  production. This is the classic "cannot reproduce locally" bug.

**Root cause:** two atomic operations are not one atomic operation. `containsKey`
returns a fact about a moment that has already passed by the time `put` executes. The
map's thread safety guarantees that neither call corrupts the map — it says nothing
about the invariant "at most one charge per key", which spans both calls plus everything
between them.

**Fix:** one atomic operation.

```java
// Value is cheap: build it, then insert conditionally.
OrderId existing = processed.putIfAbsent(key, id);
if (existing != null) { /* someone else won; use theirs */ }

// Value is expensive: compute at most once, under the bin lock.
OrderId id = processed.computeIfAbsent(key, k -> charge(order));   // ONLY if charge() is fast

// Best for this case: cache the in-flight future, so the loser waits instead of working.
// See Example 2, Fix 1.
```

**How to catch it in review:** grep for `containsKey` and `get` immediately followed by a
`put` on the same map. In a concurrent codebase, `containsKey` on a `ConcurrentMap` is
almost always a defect. Make it a checklist item: *"any `ConcurrentMap` read whose result
decides a subsequent write on the same key must be a single atomic method."*

---

### Trap 2 — `computeIfAbsent` re-entering the same map

**Wrong:**

```java
cache.computeIfAbsent(orderKey, k -> {
    Tier tier = cache.computeIfAbsent(tierKey, t -> tierService.lookup(userId));  // SAME MAP
    return place(cmd, tier);
});
```

**Exact symptom:** one of two, and which one you get is not deterministic.

*Symptom A (Java 9+, detected):*

```
java.lang.IllegalStateException: Recursive update
    at java.base/java.util.concurrent.ConcurrentHashMap.computeIfAbsent(ConcurrentHashMap.java:<line>)
    at com.orderflow.orders.IdempotentOrderService.place(IdempotentOrderService.java:<line>)
```

Intermittent HTTP 500s with a message that mentions neither the map nor the key.

*Symptom B (undetected):* the service hangs on one bin. `jcmd <pid> Thread.print` shows
threads `BLOCKED (on object monitor)` waiting to lock a
`java.util.concurrent.ConcurrentHashMap$ReservationNode` or `$Node`, and **no
`Found one Java-level deadlock` section appears** — because a single long-held monitor is
not a cycle.

**Root cause:** `computeIfAbsent` holds the bin's monitor while your mapping function
runs. A nested call to the same map may target the same bin. Because `synchronized` is
reentrant, the nested call does *not* block — it proceeds to mutate a bin whose structure
the outer call is halfway through changing. The JDK added detection for many of these
cases in Java 9 (`IllegalStateException: Recursive update`), **but not for all of them**;
whether a particular recursion is caught depends on bin layout and on whether a resize is
in flight. Treat the detection as a safety net, never as a contract.

**Fix — in order of preference:**

1. **Use two maps.** Different concerns, different maps. The tier cache and the order
   cache have different key spaces, different TTLs and different sizes; sharing one map
   was never a simplification.
2. **Resolve dependencies before the call.** Look up the tier *before* entering
   `computeIfAbsent`, and close over the resolved value:

```java
Tier tier = tierCache.computeIfAbsent(tierKey, t -> tierService.lookup(userId));
return orderCache.computeIfAbsent(orderKey, k -> place(cmd, tier));   // no nesting
```

3. **Keep the mapping function pure and short.** The rule that prevents this and Trap 3
   at once: *a `computeIfAbsent` mapping function may allocate and compute. It may not do
   I/O, take a lock, call into a framework, or touch the map it is called on.*

**Verify the behaviour on your own JDK** with Proof 3 rather than trusting any written
claim about which cases are detected.

---

### Trap 3 — expensive or blocking work inside `computeIfAbsent`

**Wrong:**

```java
Product p = productCache.computeIfAbsent(sku, k -> productRepository.bySku(k));   // JDBC
```

This is the single most common `ConcurrentHashMap` misuse in Spring codebases, and it
looks like the textbook example of what `computeIfAbsent` is for.

**Exact symptom:** under load, a pile of threads `BLOCKED` on one monitor.

- `jcmd <pid> Thread.print` shows N threads in `BLOCKED (on object monitor)` with
  `- waiting to lock <0x...> (a java.util.concurrent.ConcurrentHashMap$ReservationNode)`
  and `ConcurrentHashMap.computeIfAbsent` at the top of the stack.
- `top -H -p <pid>` shows them at **0% CPU**. They are waiting, not working.
- The JFR event `jdk.JavaMonitorEnter` shows a high count with `monitorClass` =
  `java.util.concurrent.ConcurrentHashMap$ReservationNode` or `$Node`, and durations
  matching your database latency.
- Throughput is flat while CPU is low — the signature of a serialisation bottleneck.

**Root cause:** the mapping function runs under the bin's monitor. A 30 ms JDBC call
holds that monitor for 30 ms. Every other thread writing a key **in the same bin** waits.
With 100,000 SKUs and a well-distributed hash the collision probability per pair is low —
but under skew (a handful of hot SKUs, which is exactly the `orderflow` traffic profile)
the hot keys are hit constantly and their bins are always contended. Resize helpers that
reach the bin block too, so a resize during a slow computation stalls writers across the
whole map.

**Fix — three options, in order:**

1. **Use a real cache.** Caffeine's `LoadingCache` is built for exactly this: it
   coalesces concurrent loads for the same key without holding a global or bin lock over
   the loader, and it gives you TTL, size bounds and hit-ratio metrics. This is the
   correct answer for a read-through cache and you should reach for it first.
2. **Cache the future**, as in Example 2's Fix 1 — `putIfAbsent` a `CompletableFuture`,
   then do the slow work outside the map. Correct, and worth understanding even if you
   ship Caffeine, because it is what Caffeine is doing for you.
3. **Accept duplicate work.** For an idempotent, cheap-ish load, a plain
   `get`-then-`load`-then-`putIfAbsent` that occasionally loads twice is fine — as long
   as you have *decided* that duplicates are acceptable. That is a legitimate engineering
   choice; doing it by accident is not.

**The general rule:** the mapping function should be a *computation*, not an *operation*.
If it can block, log a warning, take a lock, or take longer than a few microseconds, it
does not belong there.

---

### Trap 4 — `CopyOnWriteArrayList` used write-heavy

**Wrong:**

```java
private final List<OrderEvent> auditTrail = new CopyOnWriteArrayList<>();

public void record(OrderEvent e) {
    auditTrail.add(e);        // called on EVERY order event -- ~200/sec
}
```

Someone chose it because it is thread-safe and reads never block. Both true.

**Exact symptom:** a latency and GC problem that grows with the list, not with the load.

- Allocation rate climbs steadily over hours. `jcmd <pid> GC.heap_info` and
  `-Xlog:gc*` show young-collection frequency rising with no change in request rate.
- `jcmd <pid> GC.class_histogram` shows `[Ljava.lang.Object;` — object arrays — near the
  top by retained size, with a huge instance count.
- A JFR `jdk.ObjectAllocationSample` recording attributes the allocation to
  `CopyOnWriteArrayList.add` / `Arrays.copyOf`.
- **p99 degrades while p50 does not**, because each `add` is O(n) and n keeps growing.
  The service gets slower the longer it runs, and a restart "fixes" it.

**Root cause:** every mutation copies the entire backing array. N sequential adds copy
`1 + 2 + ... + N` references — **O(n²)** total work and **O(n²)** total bytes allocated.
At 10,000 elements that is roughly 50 million reference copies and, on a 64-bit JVM with
compressed oops, several hundred megabytes of garbage. `CopyOnWriteArrayList` is
designed for **read-mostly, write-almost-never** data: a listener list configured at
startup and read on every request.

**Fix:**

| Requirement | Use |
|---|---|
| Append and iterate, unbounded, FIFO | `ConcurrentLinkedQueue` |
| Append with backpressure and a consumer that waits | `LinkedBlockingQueue` with a bound (Topic 93) |
| Read-mostly, written at startup | **`CopyOnWriteArrayList` is correct — keep it** |
| Set semantics, concurrent | `ConcurrentHashMap.newKeySet()` |
| Sorted, concurrent | `ConcurrentSkipListSet` |
| An audit trail | not a collection at all — a table, or a Kafka topic (Topic 114) |

**The rule of thumb:** `CopyOnWriteArrayList` is right when writes are measured in "per
deploy" and wrong when they are measured in "per second"; if you cannot state the write
rate you cannot justify the choice. And if you must build one, use `addAll` or the
collection constructor — one copy for the batch instead of one per element.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print output
and call it real. What I can give you precisely is what to look for and what each
possible result means.

### Setup

```bash
mkdir -p ~/java-lab/92 && cd ~/java-lab/92
java --version                # expect 21 or 25
jcmd -l                       # you will use this constantly
```

### Proof 1 — settle the internals from your own JDK, not from a blog

```bash
mkdir -p ~/jdk-src && cd ~/jdk-src
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/concurrent/ConcurrentHashMap.java' \
                                  'java.base/java/util/concurrent/CopyOnWriteArrayList.java'

SRC=java.base/java/util/concurrent/ConcurrentHashMap.java

grep -n "static final int MOVED\|static final int TREEBIN\|static final int RESERVED" $SRC
grep -n "static final int HASH_BITS\|static final int spread" $SRC
grep -n "TREEIFY_THRESHOLD\|UNTREEIFY_THRESHOLD\|MIN_TREEIFY_CAPACITY" $SRC
grep -n "class CounterCell\|@Contended\|volatile long baseCount" $SRC
grep -n "synchronized (f)" $SRC | head
grep -n "Recursive update" $SRC
grep -n -i "segment" $SRC | head

javap -p java.util.concurrent.CopyOnWriteArrayList | grep -i "lock\|array"
```

**What to look for:**

| What you see | What it means |
|---|---|
| `MOVED = -1`, `TREEBIN = -2`, `RESERVED = -3` | Confirms the reserved negative hash space, and why `spread` masks the sign bit. |
| `TREEIFY_THRESHOLD = 8`, `UNTREEIFY_THRESHOLD = 6`, `MIN_TREEIFY_CAPACITY = 64` | Both treeify conditions. The 64 is the one candidates forget. |
| `@Contended static final class CounterCell` | The striped counter, cache-line padded — Topic 95's `LongAdder`, Topic 96's reason. |
| Several `synchronized (f)` blocks in `putVal`, `computeIfAbsent`, `replaceNode` | **Bin-level locking on the first node.** This is the mechanical statement, in the source. |
| `throw new IllegalStateException("Recursive update")` present | Your JDK has the Java 9+ detection. Note *where* it is thrown — that tells you which cases it covers. |
| Grep for "segment" finds only a `serialPersistentFields` compatibility stub and comments | **Segments are gone.** If an interviewer describes segments, they are describing Java 7. |
| `CopyOnWriteArrayList` has a `final transient Object lock` | JDK 11+. On Java 8 it is a `ReentrantLock`. Trust your JDK. |

### Proof 2 — reproduce the check-then-act race

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class CheckThenAct {
    static final int THREADS = 32, KEYS = 2_000;

    public static void main(String[] args) throws Exception {
        for (String mode : new String[]{"broken", "putIfAbsent", "computeIfAbsent"}) {
            var map = new ConcurrentHashMap<Integer, String>();
            var charges = new AtomicInteger();          // stands in for "wallet debited"
            var pool = Executors.newFixedThreadPool(THREADS);
            var start = new CountDownLatch(1);          // Topic 97
            var done = new CountDownLatch(THREADS);

            for (int t = 0; t < THREADS; t++) pool.submit(() -> {
                try {
                    start.await();
                    for (int k = 0; k < KEYS; k++) switch (mode) {
                        case "broken" -> {
                            if (!map.containsKey(k)) {          // CHECK
                                charges.incrementAndGet();      // THE CHARGE
                                map.put(k, "v" + k);            // ACT
                            }
                        }
                        case "putIfAbsent" -> {
                            if (map.putIfAbsent(k, "v" + k) == null) charges.incrementAndGet();
                        }
                        case "computeIfAbsent" -> map.computeIfAbsent(k, key -> {
                            charges.incrementAndGet(); return "v" + key;
                        });
                    }
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                finally { done.countDown(); }
            });

            start.countDown(); done.await(); pool.shutdown();
            System.out.printf("%-16s keys=%d charges=%d excess=%d%n",
                    mode, map.size(), charges.get(), charges.get() - KEYS);
        }
    }
}
```

```bash
java CheckThenAct.java
java CheckThenAct.java          # run it several times: the excess VARIES
java -XX:ActiveProcessorCount=8 CheckThenAct.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `broken` reports `excess` well above 0, and a **different** number each run | **The race, reproduced.** Every unit of excess is one duplicate wallet charge. Non-determinism is the point: this is why a passing test proves nothing. |
| `putIfAbsent` and `computeIfAbsent` report `excess = 0`, every run | The atomic forms are correct. `map.size()` should equal `KEYS` in all three cases — **the map is always consistent; only your invariant was not.** |
| `broken` reports `excess = 0` on your machine | Your core count or timing is not producing the interleaving. Raise `THREADS`, raise `KEYS`, or add a `Thread.onSpinWait()` between the check and the act to widen the window. **A zero result is "not observed", never "impossible"** — that is Topic 99's whole lesson. |

Note the last row carefully. This experiment can only ever demonstrate the bug; it can
never demonstrate its absence. The proof of correctness is the happens-before argument
(Topic 86), and `jcstress` (Topic 99) is how you check that argument rather than your
luck.

### Proof 3 — does your JDK detect a recursive `computeIfAbsent`?

```java
import java.util.concurrent.ConcurrentHashMap;

public class RecursiveUpdate {
    public static void main(String[] args) {
        var map = new ConcurrentHashMap<String, String>();
        System.out.println("pid = " + ProcessHandle.current().pid());
        try {
            map.computeIfAbsent("outer", k -> {
                map.computeIfAbsent("inner", k2 -> "innerValue");   // SAME MAP
                return "outerValue";
            });
            System.out.println("completed with no error: " + map);
        } catch (IllegalStateException e) {
            System.out.println("threw: " + e);
        }
    }
}
```

```bash
java RecursiveUpdate.java

# Now force a collision: find two keys whose spread() lands in the same bin of a
# small table, and repeat. Then try it with 16 threads hammering both keys, and if
# it hangs, take a dump:
jcmd <pid> Thread.print -l | grep -B4 -A8 'ConcurrentHashMap'
```

**What to look for:**

| What you see | What it means |
|---|---|
| `threw: java.lang.IllegalStateException: Recursive update` | The Java 9+ detection fired. Good — but it is a safety net, not a contract. |
| `completed with no error` | The two keys landed in **different bins** and the detection did not trigger. **This is the dangerous case**: the code appears to work, and will fail once a resize or a different key distribution puts them in the same bin. |
| The program hangs, and the dump shows `BLOCKED (on object monitor)` on a `ConcurrentHashMap$ReservationNode` with **no** `Found one Java-level deadlock` section | The undetected variant, reproduced. Note carefully that the deadlock detector is silent — this is not a cycle, it is one monitor held too long. Topic 98's decision table exists for exactly this. |

**Write down which of the three you got, and on which JDK.** That single observation is
worth more than any paragraph anyone can write about it, because the behaviour is
implementation detail and it has changed.

### Proof 4 — see bin contention in a thread dump

```java
import java.util.concurrent.ConcurrentHashMap;

public class BinContention {
    static final ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

    public static void main(String[] args) throws Exception {
        System.out.println("pid = " + ProcessHandle.current().pid());
        new Thread(() -> map.computeIfAbsent("hot", k -> {      // holds the bin
            try { Thread.sleep(600_000); } catch (InterruptedException e) { }  // "a JDBC call"
            return "v";
        }), "holder").start();
        Thread.sleep(300);
        for (int i = 0; i < 8; i++)                             // 8 writers, same key
            new Thread(() -> map.computeIfAbsent("hot", k -> "v2"), "waiter-" + i).start();
        Thread.sleep(600_000);
    }
}
```

```bash
java BinContention.java
# other terminal:
jcmd <pid> Thread.print -l > cont.txt
grep 'java.lang.Thread.State' cont.txt | sort | uniq -c | sort -rn
grep -A6 '"waiter-' cont.txt
grep -c 'Found one Java-level deadlock' cont.txt
top -H -p <pid>
```

**What to look for:**

| What you see | What it means |
|---|---|
| Eight threads in `BLOCKED (on object monitor)` with `- waiting to lock <0x...> (a java.util.concurrent.ConcurrentHashMap$ReservationNode)` | **Bin contention, confirmed.** Note the state is `BLOCKED`, not `WAITING` — CHM uses an intrinsic monitor, unlike every `j.u.c` lock in Topic 94. |
| All eight share the **same** monitor identity `<0x...>` | One bin. In production, N threads on one monitor identity means one hot key or one hash collision. |
| The deadlock grep returns `0` | Correct and important. **A long-held monitor is not a deadlock**, so the detector says nothing while the service is unusable. |
| `top -H` shows the waiters at ~0% CPU | They are blocked, not spinning. Throughput flat with low CPU is the signature. |

Keep `cont.txt`. Being able to recognise `ConcurrentHashMap$ReservationNode` in a dump
at 3am is worth an hour of incident time.

### Proof 5 — the `CopyOnWriteArrayList` cost curve

Do **not** measure this with a `System.nanoTime()` loop; the correct JMH harness is in
the Measurement section. What you can do cheaply and honestly is watch the allocation:

```bash
java -Xlog:gc -XX:StartFlightRecording=filename=cowal.jfr,settings=profile YourCowalTest.java
jfr print --events jdk.ObjectAllocationSample cowal.jfr | grep -A4 'CopyOnWriteArrayList'
jcmd <pid> GC.class_histogram | head -20
```

**What to look for:** `[Ljava.lang.Object;` high in the class histogram by instance count
*and* retained size, and allocation samples attributed to `Arrays.copyOf` under
`CopyOnWriteArrayList.add`. That is the O(n²) allocation, visible as garbage rather than
as a benchmark number.

---

## Failure drill

**Assignment (Topic 92, from the master plan's drill map):** a check-then-act race on a
CHM-backed idempotency cache. Two concurrent requests both pass the check and both charge
the wallet. Trace it, then fix it with `putIfAbsent`.

Budget ninety minutes. Part F is where the topic becomes senior-level; do not stop at
Part D.

### Part A — establish the ground truth

1. Bring up the Topic 65 stack with the `load` profile.
2. Re-run the recorded k6 baseline; confirm p50/p95/p99 within ±10% of
   `/docs/java/baselines/`. **If it does not reproduce, stop and fix that first.**
3. Confirm the reconciliation query returns **zero** rows before you start:

```sql
SELECT user_id, total_minor, count(*) AS charges, min(created_at), max(created_at)
FROM payments
WHERE created_at > now() - interval '1 hour'
GROUP BY user_id, total_minor
HAVING count(*) > 1;
```

That query is your instrument. Everything in this drill is measured by it.

### Part B — ship the bug

Deploy the broken `IdempotentOrderService` from Example 2 — `containsKey` then `put`.
Add the idempotency-key plumbing to the controller if it is not already there.

### Part C — drive concurrent duplicates

The baseline k6 script sends unique keys, so it will not reproduce this. Add a scenario
that sends **the same key twice, concurrently**:

```javascript
// load/orderflow-idempotency.js
import http from 'k6/http';
import { check } from 'k6';

export const options = { scenarios: { duplicates: {
  executor: 'constant-arrival-rate', rate: 40, timeInterval: '1s',
  duration: '10m', preAllocatedVUs: 100, maxVUs: 400 } } };

export default function () {
  const key = `IK-${__VU}-${__ITER}`;                 // ONE key, TWO requests
  const body = JSON.stringify({ sku: 'SKU-1001', quantity: 1, userId: 55 });
  const params = { headers: { 'Content-Type': 'application/json', 'Idempotency-Key': key } };

  // http.batch fires both at the same instant -- that is the whole point
  const r = http.batch([['POST', 'http://localhost:8080/orders', body, params],
                        ['POST', 'http://localhost:8080/orders', body, params]]);

  check(r[0], { 'first is 2xx': (x) => x.status < 300 });
  check(r[1], { 'second is 2xx': (x) => x.status < 300 });
}
```

Note what the checks do **not** assert: that both responses carry the same order id. Add
that assertion yourself — it is the one that fails.

### Commands

```bash
k6 run load/orderflow-idempotency.js

PID=$(jcmd -l | grep -i orderflow | cut -d' ' -f1)

# The instrument: duplicate charges
psql -h localhost -U orderflow -c "
  SELECT user_id, total_minor, count(*) FROM payments
  WHERE created_at > now() - interval '10 minutes'
  GROUP BY 1,2 HAVING count(*) > 1;"

# Duplicate orders for one intent
psql -h localhost -U orderflow -c "
  SELECT idempotency_key, count(*) FROM orders
  WHERE created_at > now() - interval '10 minutes'
  GROUP BY 1 HAVING count(*) > 1;"

# The map's own size -- is it growing without bound? (Defect 3)
jcmd $PID GC.class_histogram | grep -E 'ConcurrentHashMap\$Node'

# Thread states, in case you hit Trap 3 as well
jcmd $PID Thread.print -l > d1.txt
grep 'java.lang.Thread.State' d1.txt | sort | uniq -c | sort -rn
```

### What to capture, before reading on

Write these down from your own run. Do not read ahead. (1) How many duplicate-charge rows
does the reconciliation query return over ten minutes? (2) What fraction of the duplicate
pairs is that? (3) What HTTP status did **both** requests return? (4) Do the two responses
carry the same order id? (5) After the run, which order id does the cache hold for a key
that was double-charged — the first or the second? (6) Does anything at all appear in the
application logs at `WARN` or above? (7) What is the `ConcurrentHashMap$Node` instance
count, and how does it change over the ten minutes?

### How to read it

| What you see | What it means |
|---|---|
| A non-zero, **varying** duplicate count | The race, reproduced against the real service. The variation is the point: it is timing-dependent, so a green test suite proves nothing. |
| Both requests returned **HTTP 201** | **The failure is invisible to every HTTP-level monitor you have.** Your error-rate dashboard is flat. This is why the reconciliation query is the instrument. |
| The two responses carry **different** order ids | The client discarded one of two real orders. It has no idea. |
| The cache holds the **second** order id | The later `put` overwrote the first. So a third retry returns the order the client never saw — and the cache looks perfectly consistent to anyone inspecting it afterwards. |
| Nothing above `INFO` in the logs | Confirms Topic 09's lesson: a defect whose only symptom is a duplicated side effect. |
| `ConcurrentHashMap$Node` count rising monotonically and never falling | **Defect 3, the unbounded map.** Extrapolate it to four days and you have an `OutOfMemoryError`. |
| Zero duplicates | Your window is too narrow. Add 50 ms of artificial latency inside `doPlace` — a `pg_sleep`, or a `Thread.sleep` — to widen it, and note that **you have just demonstrated that this bug's visibility depends on downstream latency**, which is why it appears the day the database gets slower. |

### Part D — fix the race

Apply Fix 1 from Example 2: `putIfAbsent` with a `CompletableFuture` value, a bounded
`get(timeout)`, and removal on failure. Re-run the identical k6 script.

Then prove the fix rather than assuming it:

- The reconciliation query must return **zero** rows.
- Both responses must now carry the **same** order id — add that k6 check and watch it
  go green.
- Compare p50/p95/p99 to the recorded baseline. The loser thread now *waits* for the
  winner, so the second request's latency should rise to roughly the first's. **That is
  correct behaviour, and you should be able to explain why it is an improvement over
  returning fast and wrong.**

### Part E — break the fix on purpose

Two variations, both instructive:

1. **Remove the `inFlight.remove(key, mine)` from the failure path.** Then make
   `placement.place` fail for one key (a wallet with insufficient funds). Retry that key
   ten times. **Every retry fails forever**, because the cache holds a permanently-failed
   future. This is the most common bug in hand-rolled request coalescing, and now you
   have produced it deliberately.
2. **Replace the two-argument `remove(key, mine)` with the one-argument `remove(key)`.**
   Under concurrency, one thread's cleanup deletes another thread's fresh entry, and the
   race returns in a subtler form. Reproduce it, and note that you fixed a check-then-act
   with code that itself contained one.

### Part F — the finding that matters: scale to three pods

Everything up to here fixed the race **inside one JVM**. Now run the real topology.

1. Start three instances behind the load balancer (three docker-compose replicas, or
   three ports behind nginx with round-robin).
2. Run the same k6 script. The load balancer will route the two concurrent requests to
   **different pods**.
3. Run the reconciliation query.

**The duplicates come back**, and this time there is no race at all: each pod took the
fast path with an empty cache. Capture the number and sit with it, because this is the
lesson of the topic.

4. Apply Fix 3 — a unique index on `orders(idempotency_key)` plus the
   `DataIntegrityViolationException` handler. Re-run against three pods.
5. Confirm zero duplicates, and confirm in the Postgres logs (`log_min_error_statement`)
   that the constraint is actually being violated and caught — i.e. that the database is
   doing the work you now claim it is doing.

### What the fix proves

Write one paragraph on each:

- **Was the map ever broken?** No. `map.size()` was correct throughout, no entry was
  corrupted, no exception was thrown. State precisely what was broken instead, in one
  sentence.
- **Why did the fix not survive scaling out?** Answer in terms of where the guarantee
  lives, not in terms of Java.
- **What is the in-memory map for, after Fix 3?** If your answer is not "avoiding a
  database round trip on the common path", re-read Example 2.
- **What did the fix cost?** Compare p99 against the baseline. The loser now waits;
  quantify by how much and decide whether it is acceptable.

---

## Measurement

### The instrument for each claim

| Claim | Instrument that makes it falsifiable |
|---|---|
| "We are double-charging" | The reconciliation SQL — `GROUP BY user_id, total HAVING count(*) > 1` |
| "The map is the bottleneck" | JFR `jdk.JavaMonitorEnter` filtered to `monitorClass` starting `java.util.concurrent.ConcurrentHashMap$` |
| "It is bin contention, not slow code" | `jcmd Thread.print` showing N threads `BLOCKED` on the **same** monitor identity, at ~0% CPU in `top -H` |
| "Keys are colliding" | `BLOCKED` on `ConcurrentHashMap$TreeBin` — a treeified bin means 8+ collisions in a 64+ table. Check `hashCode` (Topic 13). |
| "The map is leaking" | `jcmd GC.class_histogram \| grep 'ConcurrentHashMap$Node'` across time; then a heap dump and MAT's dominator tree (Topic 79) |
| "`CopyOnWriteArrayList` is the allocation source" | JFR `jdk.ObjectAllocationSample` attributed to `Arrays.copyOf`; `[Ljava.lang.Object;` high in the class histogram |
| "CHM beats a synchronized map here" | JMH with `@Threads({1,2,4,8,16,32,64})`, `@State(Scope.Benchmark)`, `@Fork(3)` |
| "The fix did not cost latency" | **The recorded Topic 65 p50/p95/p99, re-run identically** |

### Micrometer — what to expose permanently (Topic 118 forward-reference)

`ConcurrentHashMap` has no binder, and that is appropriate: a bare map is not a
component. **A cache is**, so make it one:

```java
Cache<String, CompletableFuture<OrderId>> inFlight = Caffeine.newBuilder()
        .maximumSize(100_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .recordStats()                                   // required for the binder
        .build();

CaffeineCacheMetrics.monitor(registry, inFlight, "idempotency",
        Tags.of("service", "orderflow"));
```

| Metric | What it tells you | Alert? |
|---|---|---|
| `cache.size` (gauge) | Entries held. **Flat is healthy; monotonic growth means your TTL is wrong.** | On approach to `maximumSize` |
| `cache.gets{result="hit"/"miss"}` | Hit ratio. A collapsing ratio means retries are landing on cold pods. | On a sustained ratio drop |
| `cache.evictions` (counter) | Pressure. Non-zero with a size-based bound means the bound is binding. | As a rate |
| `orderflow.idempotency.duplicate.detected` (counter) | **Your own counter.** Increment when `putIfAbsent` returns non-null. This is the metric that would have caught the bug. | On a sudden rate change |
| `orderflow.idempotency.constraint.violation` (counter) | Fix 3's database rejections — cross-pod duplicates. **Non-zero is correct**; a sudden rise means the cache is not working. | As a rate |

*Illustration of the exposition FORMAT, not captured output. `<n>` are placeholders.*

```
# TYPE cache_size gauge
cache_size{cache="idempotency",service="orderflow",} <n>.0
# TYPE cache_gets_total counter
cache_gets_total{cache="idempotency",result="hit",service="orderflow",} <n>.0
# TYPE orderflow_idempotency_duplicate_detected_total counter
orderflow_idempotency_duplicate_detected_total{service="orderflow",} <n>.0
```

**Never tag any of these with the idempotency key, an order id or a SKU** — one series
per key is millions of series and a dead Prometheus (Topic 118's cardinality drill).

### `jcmd` and JFR

```bash
jcmd -l
jcmd $PID Thread.print -l > d1.txt

# Bin contention: BLOCKED, not WAITING. This is the CHM fingerprint.
grep -B2 -A8 'ConcurrentHashMap' d1.txt | grep -A6 'BLOCKED'

# Which monitor class? Node = ordinary bin, TreeBin = collisions, ReservationNode = a
# computeIfAbsent mapping function is holding the bin.
grep -oE 'ConcurrentHashMap\$[A-Za-z]+' d1.txt | sort | uniq -c | sort -rn

# Are they all on ONE monitor? (one hot key) or many? (general load)
grep 'waiting to lock' d1.txt | awk '{print $4}' | sort | uniq -c | sort -rn

grep 'java.lang.Thread.State' d1.txt | sort | uniq -c | sort -rn   # state histogram

# Map growth over time -- run twice, minutes apart
jcmd $PID GC.class_histogram | grep -E 'ConcurrentHashMap\$Node|\[Ljava.lang.Object;'
```

```bash
java -XX:StartFlightRecording=duration=300s,filename=chm.jfr,settings=profile -jar orderflow.jar

jfr summary chm.jfr
jfr print --events jdk.JavaMonitorEnter chm.jfr | head -60
jfr print --json --events jdk.JavaMonitorEnter chm.jfr \
  | jq -r '.recording.events[].values.monitorClass.name' | sort | uniq -c | sort -rn
```

| Event | What to read from it |
|---|---|
| `jdk.JavaMonitorEnter` with `monitorClass = ...ConcurrentHashMap$Node` | Ordinary bin contention. High count with short durations is normal under load. |
| Same, with `...$ReservationNode` and **long** durations | **Trap 3.** A mapping function is doing slow work under the bin lock. |
| Same, with `...$TreeBin` | Treeified bins. Suspect `hashCode` before load. |
| `jdk.ObjectAllocationSample` under `Arrays.copyOf` / `CopyOnWriteArrayList.add` | Trap 4's O(n²) allocation. |

Note that CHM contention appears under `jdk.JavaMonitorEnter` (the `synchronized` event),
**not** `jdk.ThreadPark` (the `j.u.c` event). Searching only the latter — the habit you
picked up in Topic 94 — will make you conclude there is no contention. Check both.

### The standing rule: a naive `System.nanoTime()` loop is wrong

You will be tempted to settle "is `ConcurrentHashMap` faster than
`Collections.synchronizedMap`" like this. Do not.

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    map.put(i, "v");
}
System.out.println((System.nanoTime() - start) / 10_000_000 + " ns/op");
```

Five independent reasons it lies, and you cannot tell which one is lying:

1. **Single-threaded means uncontended.** The entire subject of this document is what
   happens when several threads touch the same bins. A serial loop exercises none of it.
2. **Dead-code elimination.** If nothing reads the map afterwards, C2 may delete work.
3. **Cold JIT and on-stack replacement.** Your average blends interpreted, C1 and C2
   execution in a ratio set by the iteration count you happened to pick.
4. **Resize dominates.** Ten million sequential puts spend most of their time growing the
   table. You have benchmarked `transfer`, not `put`.
5. **Sequential integer keys are the best case.** `spread` distributes them perfectly, so
   you never treeify, never collide, and never measure the behaviour that matters.

**Topic 77 is the full treatment.** The correct harness:

```java
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)                  // ONE shared map -- this IS the point
@Fork(3)                                 // three JVMs: profile pollution shows as variance
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Threads({1, 2, 4, 8, 16, 32, 64})       // the scaling curve IS the result
public class MapScaling {

    @Param({"1000", "100000"})            // small map = more collisions
    int size;

    private ConcurrentHashMap<Integer, String> chm;
    private Map<Integer, String> sync;
    private Map<Integer, String> caffeine;

    @Setup
    public void setup() {
        chm = new ConcurrentHashMap<>();
        sync = Collections.synchronizedMap(new HashMap<>());
        caffeine = Caffeine.newBuilder().maximumSize(size * 2L).<Integer, String>build().asMap();
        for (int i = 0; i < size; i++) {
            chm.put(i, "v" + i); sync.put(i, "v" + i); caffeine.put(i, "v" + i);
        }
    }

    private int nextKey() { return ThreadLocalRandom.current().nextInt(size); }

    @Benchmark public String chmGet()  { return chm.get(nextKey()); }
    @Benchmark public String syncGet() { return sync.get(nextKey()); }

    @Benchmark public void chmComputeIfAbsent(Blackhole bh) {
        bh.consume(chm.computeIfAbsent(nextKey(), k -> "v" + k));
    }
    @Benchmark public void chmPutIfAbsent(Blackhole bh) {
        bh.consume(chm.putIfAbsent(nextKey(), "v"));
    }
}
```

Five annotations doing specific work: **`@State(Scope.Benchmark)`** shares one map across
all threads (with `Scope.Thread` there is no contention and you measured nothing);
**`@Threads({1,...,64})`** makes the *shape of the curve* the result — flat is
contention-limited, rising is scaling, *falling* is Topic 95's pathology; **`@Param`**
varies map size, because collision rate decides the answer; **`Blackhole.consume`** stops
dead-code elimination; **`@Fork(3)`** exposes profile pollution (Topic 74).

**What to expect, stated honestly before you run it:** `chmGet` should scale close to
linearly with threads, because reads take no lock and write nothing to shared memory.
`syncGet` should be flat or falling from two threads upward, because every read
serialises on one monitor. The write benchmarks should scale sub-linearly and degrade
worse at `size = 1000` than at `size = 100000`, because a smaller table means more keys
per bin and more contention. If your results contradict this, trust your machine and
report the numbers — hardware, JDK build and core topology all move these curves.

### `CopyOnWriteArrayList` — the quadratic curve

```java
@BenchmarkMode(Mode.AverageTime) @OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread) @Fork(3)            // per-thread on purpose: this is a COST question
public class CowalGrowth {

    @Param({"100", "1000", "10000"}) int n;

    @Benchmark public List<Integer> cowalAddN() {
        List<Integer> list = new CopyOnWriteArrayList<>();
        for (int i = 0; i < n; i++) list.add(i);       // O(n^2) total
        return list;
    }
    @Benchmark public List<Integer> cowalAddAllOnce() {
        List<Integer> src = new ArrayList<>(n);
        for (int i = 0; i < n; i++) src.add(i);
        return new CopyOnWriteArrayList<>(src);        // O(n) total -- ONE copy
    }
}
```

**The result is the ratio across `@Param` values, not any single number.** n=100 to
n=1000 is 10x the elements; if `cowalAddN` costs roughly 100x more time you have observed
the quadratic directly, while `cowalAddAllOnce` should stay linear. Add `-prof gc` and
compare `gc.alloc.rate.norm` — bytes per operation — which makes the O(n²) allocation
explicit rather than inferred.

### `perf` — and the honest note about macOS

On Linux, the cache-line traffic behind contended bins is directly observable:

```bash
perf stat -e cache-misses,cache-references,LLC-load-misses -- java -jar orderflow.jar
```

**On macOS, `perf` does not exist**, and there is no equivalent exposing the same
hardware counters to a JVM process. Your options: run the experiment in a Linux container
(`--privileged` is usually required for PMU access, and on Apple Silicon under
virtualisation the counters may be unavailable entirely); use
`xctrace record --template 'Time Profiler'` for CPU-time attribution, which does **not**
give cache-miss counters; or accept that the JMH scaling curve is the measurement you can
actually get and treat cache-line reasoning as an *explanation* for the curve rather than
something you independently observed.

**Say which one you did.** "I would confirm with `perf stat` on Linux; on my Mac I can
only infer it from the JMH curve" is a better answer than a confident claim about numbers
you cannot see.

---

## Practice exercises

### 1 — Easy: build your own atomic-method reference

Write a small program that, for each of `put`, `putIfAbsent`, `computeIfAbsent`,
`computeIfPresent`, `compute`, `merge` and `replace(k, old, new)`, exercises the key in
three states — absent, present with a value, and present where the function returns
`null` — and records for each: **what the method returns, what the map contains
afterwards, and whether the mapping function ran.**

Produce a 7 x 3 table. Then answer four questions:

- Which methods can **remove** an entry? (There are more than you expect.)
- Which return the **new** value and which return the **old** one? Which returns `null`
  to mean "I inserted" and which returns `null` to mean "there was nothing"?
- What does `merge` do when the remapping function returns `null`?
- Try `map.put(k, null)` and `map.computeIfAbsent(k, x -> null)`. What happens in each
  case, and why is the asymmetry deliberate?

Keep the table. You will reach for it more often than you expect.

### 2 — Medium: the audit (combines Topics 12, 13, 15, 25, 79, 85, 90, 91, 94)

The fragment below contains **eight** distinct defects drawn from this topic and earlier
ones. Find them all, state the **exact symptom each produces in production** — not "it is
bad practice", but what the on-call engineer sees on a dashboard, in a log, or in a
thread dump — and rewrite it correctly.

```java
@Service
public class PricingCache {

    private static final Map<Sku, BigDecimal> PRICES = new ConcurrentHashMap<>();
    private final List<PriceListener> listeners = new CopyOnWriteArrayList<>();
    private final Map<String, Integer> hitCounts = new ConcurrentHashMap<>();

    public BigDecimal priceFor(Sku sku) {
        if (!PRICES.containsKey(sku)) {
            PRICES.put(sku, pricingService.lookup(sku));       // remote call
        }
        hitCounts.put(sku.code(), hitCounts.getOrDefault(sku.code(), 0) + 1);
        return PRICES.get(sku);
    }

    public void onPriceChange(PriceEvent e) {
        listeners.add(new AuditListener(e));                   // called per event
        PRICES.computeIfAbsent(e.sku(), s -> {
            listeners.forEach(l -> l.notify(e));               // listeners may touch PRICES
            return e.newPrice();
        });
    }

    public boolean isFull() {
        return PRICES.size() >= 100_000;
    }

    public List<Sku> allSkus() {
        return PRICES.keySet().stream().parallel().toList();
    }
}
```

Hints, in no particular order: what `Sku`'s `hashCode` must satisfy for any of this to
work, and what happens if it is mutable (Topic 13); which two lines form a check-then-act;
what a remote call inside a map operation costs (twice over — find both places); what
`getOrDefault` then `put` does at 400 rps; what `listeners.add` per event does to
allocation (Topic 68); whether `listeners.forEach` inside `computeIfAbsent` can reach
`PRICES` and what happens if it does; what `static` means for the lifetime of `PRICES`
(Topic 79); whether `isFull` can be trusted; and which thread pool `.parallel()` uses
(Topic 25, Topic 91).

For each defect also state **which observable signal would first reveal it** — a metric, a
log line, a thread-dump line, a heap dump, or a reconciliation query. **Two of the eight
have no observable signal at all.** Identify which two, and say what you would add.

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A — ground truth.** Bring up the Topic 65 stack. Confirm the recorded baseline
within ±10%, and confirm the duplicate-charge reconciliation query returns zero rows.

**Part B — implement four versions of the idempotency cache**, behind a feature flag so
you can switch without redeploying: (1) **broken** — `containsKey`/`put`;
(2) **computeIfAbsent** — atomic, but running the full placement under the bin lock;
(3) **future-caching** — `putIfAbsent` with a `CompletableFuture`, per Example 2 Fix 1;
(4) **Caffeine + database unique constraint**, per Fix 2 and Fix 3.

**Part C — measure all four at baseline with concurrent duplicate keys.** One table:
duplicate charges per 10 minutes, p50/p95/p99, throughput, thread-state histogram,
`ConcurrentHashMap$Node` count, and threads `BLOCKED` on a CHM monitor. Version 2 should
be *correct* and *slower*; explain the mechanism in one sentence.

**Part D — introduce hash skew.** Give the key wrapper a deliberately terrible `hashCode`
that returns a constant (Topic 13) and re-run version 3. Record threads `BLOCKED` on
`ConcurrentHashMap$TreeBin` and what happened to p99. **Explain why a bad `hashCode` in
one class degrades a data structure two layers away.**

**Part E — scale to three pods.** Re-run versions 3 and 4 behind the load balancer and
record duplicate charges for each. **Version 3 must fail.** State in one paragraph, for a
non-specialist, why a correct concurrency fix did not survive horizontal scaling.

**Part F — the memory question.** Run version 3 for two hours at baseline with no
eviction, capturing `jcmd GC.class_histogram` filtered to `ConcurrentHashMap$Node` and
heap-used from `-Xlog:gc` every fifteen minutes. Plot both, extrapolate to the
`OutOfMemoryError`, and state the date. Then take a heap dump and confirm in MAT's
dominator tree that the map is the dominator (Topic 79).

**Part G — argue both sides, then design.** Make the strongest case for **no** in-memory
idempotency cache at all — just the database constraint — then the strongest case against
yourself, and state the exact condition under which your answer flips (hint: the ratio of
duplicate requests to total, against the cost of a Postgres round trip in your p99
budget). Then design the version you would ship, and say what you would monitor to know
it is working.

---

## Interview questions

### Q1 — "`ConcurrentHashMap` is thread-safe, so this code is fine — right?"

*(The interviewer shows you `if (!map.containsKey(k)) map.put(k, v);`)*

**Mid-level answer:** "Yes, `ConcurrentHashMap` handles concurrent access, so it is safe."

**Senior answer:** "No, and the map's thread safety is exactly what makes it look safe.
Each *operation* is atomic; the *sequence* is not. Two threads can both evaluate
`containsKey` as false before either reaches `put`, and both proceed. The map never
corrupts — `size()` stays right, no entry is torn — but my invariant, 'do this once per
key', is violated.

That matters because of what usually sits between the check and the act. In our order
service it is a wallet debit, so the bug is not a duplicate map entry, it is a customer
charged twice, with two HTTP 201s in the access log and nothing in the application logs.

The fix is one atomic operation instead of two: `putIfAbsent` when the value is cheap to
build, `computeIfAbsent` when it is expensive and must run at most once, `merge` for a
counter, `replace(k, old, new)` when I want a CAS loop.

One caveat on `computeIfAbsent`: the mapping function runs while the bin's monitor is
held, so it must not do I/O and must not touch the same map — on Java 9+ a recursive
update throws `IllegalStateException: Recursive update` in many cases, and in the rest it
just blocks. For our case I would actually cache a `CompletableFuture` with
`putIfAbsent`, so the duplicate request waits for the first one's result and the expensive
work happens outside the map entirely.

And I would say one more thing: none of that makes the endpoint idempotent. The map is
per-JVM and we run three pods, so a retry on another pod duplicates with no race at all.
The guarantee has to be a unique constraint on the idempotency key in the database. The
map is a latency optimisation in front of it."

**What separates them:** distinguishing the map's integrity from the caller's invariant;
naming the business consequence rather than the data-structure consequence; knowing which
atomic method fits which case; the `computeIfAbsent` caveat; and escalating from "fix the
race" to "the guarantee lives in the database". The last point is the one that gets the
offer.

**Follow-up:** "Would `synchronized` around both lines fix it?" Yes, for one JVM, and it
serialises every access to the map — you have thrown away the per-bin locking you were
paying for. `putIfAbsent` is one bin's monitor for a few nanoseconds instead.

---

### Q2 — "Walk me through `ConcurrentHashMap`'s internals."

**Mid-level answer:** "It divides the map into 16 segments, and each segment has its own
lock, so 16 threads can write at once."

**Senior answer:** "That is the **Java 7** design and it was removed in Java 8. Worth
knowing because it is still the most common answer, but it describes a class that no
longer exists.

Since Java 8 there are no segments. There is one `Node[] table`, and **the lock
granularity is a single bin**.

Reads are lock-free: `get` does a volatile read of the table slot and walks the bin,
reading `volatile V val` and `volatile Node next`. No lock, no CAS, no write to shared
memory — so reads scale close to linearly with cores, and the volatile reads are what
give the happens-before edge that makes it correct rather than just fast.

Writes take one of two paths. Empty bin: a single CAS, no lock. Occupied bin: the writer
`synchronized`s on the **first node of that bin** and mutates under that monitor,
re-checking that the bin head has not changed. Two writers on different bins never
contend — and because the lock is an intrinsic monitor, CHM contention shows as `BLOCKED`
in a dump, not `WAITING (parking)`, the opposite of every `j.u.c` lock.

A bin treeifies at 8 nodes, but **only if the table has at least 64 buckets** — below
that it resizes, because a long chain in a tiny table means the table is too small rather
than the hashes adversarial. Untreeify is at 6; the gap is hysteresis.

Resize is cooperative: a writer that meets a `ForwardingNode` — a node with the reserved
hash `MOVED`, which is `-1` — calls `helpTransfer` and moves bins itself. That is why
`spread` masks off the sign bit: negative hashes are reserved for control nodes,
`MOVED = -1`, `TREEBIN = -2`, `RESERVED = -3`.

And `size()` is an estimate: the count is a `baseCount` plus a striped `CounterCell[]`,
each cell `@Contended` so they do not share a cache line. That is `LongAdder`'s design
for `LongAdder`'s reason — a single atomic counter would be a global contention point
that undoes the per-bin locking. So `size()` is fine for a dashboard and must never
appear in a compare-and-act."

**What separates them:** correcting the segments answer *and* dating it; the two write
paths; naming the lock as an intrinsic monitor on the bin head and knowing the
diagnostic consequence; both treeify conditions; cooperative resize; and connecting
`CounterCell` to `LongAdder` and `@Contended`.

**Follow-up:** "Why does `spread` mask the sign bit?" To reserve negative hashes for
`ForwardingNode`, `TreeBin` and `ReservationNode`. It is a tagging scheme in the hash
field.

---

### Q3 — "When would you use `CopyOnWriteArrayList`, and when is it a disaster?"

**Mid-level answer:** "When you have more reads than writes. It copies on write so
readers do not need a lock."

**Senior answer:** "The ratio framing understates it. Every mutation allocates and copies
the **entire** backing array, so N sequential adds are O(n²) in both time and bytes
allocated. At ten thousand elements that is roughly fifty million reference copies and
hundreds of megabytes of garbage — an allocation-rate problem, a GC problem and a p99
problem from one data-structure choice.

So my rule is not 'more reads than writes', it is **writes measured in per-deploy, not
per-second**. The canonical fit is a listener list built at startup and iterated on every
request: reads are one volatile array read, the iterator is a true snapshot, and the
volatile write in `setArray` gives safe publication for free (Topic 88).

The disaster case is anything appended in a loop. The symptom is distinctive — p99
degrades over hours while p50 does not, because each add is O(n) and n keeps growing, and
a restart 'fixes' it; in a class histogram `[Ljava.lang.Object;` sits near the top and
JFR attributes the allocation to `Arrays.copyOf`. For those I would use
`ConcurrentLinkedQueue` if unbounded and non-blocking, a bounded `LinkedBlockingQueue` if
I want backpressure, `ConcurrentHashMap.newKeySet()` for set semantics, or — usually
right for an audit trail — not a collection at all, but a table or a Kafka topic.

And if I do build one, I build an `ArrayList` first and pass it to the constructor. One
copy instead of N."

**What separates them:** quantifying O(n²) in both time *and* allocation; the
"per-deploy, not per-second" rule; the distinctive p99-degrades-while-p50-does-not
symptom and how to confirm it; and naming the correct alternative for each shape.

**Follow-up:** "Why does the iterator not support `remove()`?" Because it iterates a
snapshot array that is no longer the map's state — a removal would have nowhere
meaningful to apply. It throws `UnsupportedOperationException`.

---

### Q4 — "The service is hung and the thread dump shows dozens of BLOCKED threads. Walk me through it."

**Mid-level answer:** "It sounds like a deadlock. I would look for the deadlock section
in the dump and then restart the service."

**Senior answer:** "First: **do not restart.** A restart destroys the only evidence. If
availability is critical, pull the pod out of the load balancer via readiness while
leaving the process alive.

Then, in order. **Three dumps, ten seconds apart** — one gives state, three tell me
whether anything is moving. A **state histogram** of each,
`grep 'Thread.State' | sort | uniq -c`, which puts me in one of four buckets. Then
**search for `Found one Java-level deadlock`** — if it is there the header names both
threads and both resources and I am nearly done.

For this symptom specifically, `BLOCKED` means an **intrinsic monitor**, so it is
`synchronized`, and `ConcurrentHashMap` is a candidate because its bin lock is a plain
monitor. I would look at the monitor class on the `- waiting to lock` lines:
`ConcurrentHashMap$Node` is ordinary bin contention; `$TreeBin` means at least eight
collisions in a table of sixty-four or more, so I would suspect a bad `hashCode` before I
suspect load; and `$ReservationNode` means **a `computeIfAbsent` mapping function is
holding the bin** — which almost always means someone put a database or HTTP call inside
the lambda.

I would also check whether the blocked threads share **one** monitor identity — one
identity is one hot key, many is general load, and those are different problems.

Critically: one monitor held for a long time is **not a cycle**, so the deadlock detector
reports nothing. 'No deadlock section' does not mean 'no hang'. I would corroborate with
`top -H`, because `RUNNABLE` does not mean burning CPU and `BLOCKED` threads at zero CPU
with flat throughput is the serialisation signature. Then I look at what is *not* hung: an
unrelated endpoint failing too means the blast radius is the request-thread pool."

**What separates them:** "do not restart" first; knowing that `BLOCKED` specifically means
an intrinsic monitor and that CHM qualifies; reading the monitor *class* as a diagnosis;
distinguishing one hot key from general load by monitor identity; and knowing that the
deadlock detector is silent for a long-held monitor.

**Follow-up:** "The lambda calls the database. Why is that so bad if the map is only
locked per bin?" Because the bin is held for the whole call, resize helpers reaching that
bin block too, and under skew the hot keys are the ones being hit constantly.

---

### Q5 — "How would you make an endpoint idempotent?"

**Mid-level answer:** "Keep a `ConcurrentHashMap` of processed request ids and skip the
work if the key is already there."

**Senior answer:** "That is the right instinct and the wrong layer, and there are three
separate problems with it.

**First, the composition.** `containsKey` then `put` is a check-then-act race — two
concurrent retries both pass the check and both do the work. The in-JVM fix is
`putIfAbsent` with a `CompletableFuture` as the value, so the loser waits for the winner's
result rather than duplicating it, with a bounded `get(timeout)` and removal of the entry
on failure so a transient error does not poison the key forever.

**Second, the memory.** A map keyed by request id that is never evicted is an unbounded
`static`-lifetime cache — Topic 79's leak. It needs a size bound and a TTL derived from
the client's retry window, which means a real cache with eviction metrics, not a bare map.

**Third, and this is the one that matters: the map is per-JVM.** We run three pods, so a
retry routed elsewhere finds an empty map, takes the fast path with no contention at all,
and duplicates the work. There is no race to fix. The guarantee has to live where all the
instances agree — a unique constraint on the idempotency key, with a handler that catches
the violation and returns the existing result. That is the only arbiter across pods, and
it survives a restart, which the map does not. The map then becomes what it should always
have been: a latency optimisation on the common path.

I would monitor two counters: duplicates caught by the cache, and constraint violations
caught by the database. A rise in the second relative to the first tells me the cache is
not doing its job, and neither number rising when retries are happening tells me the keys
are not what I think they are."

**What separates them:** treating idempotency as a distributed-systems property rather
than a data-structure one; the three-layer breakdown; knowing the failed-future poisoning
trap; and naming the two metrics that prove the design is working.

**Follow-up:** "What if the operation is not a database write — say it publishes to
Kafka?" Then the outbox pattern (Topic 115): write the intent in the same transaction as
the business data, with the unique constraint, and have a separate process publish it.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `ConcurrentHashMap` locks the **first node of a bin** rather than using a fixed array
   of striped locks. Name two advantages of that choice, and one thing it makes harder.
   What would break if the implementation locked the `Node[] table` slot itself instead
   of the node in it?

2. `get()` performs no writes to shared memory at all — no CAS, no lock, no counter
   update. Explain what makes that **correct** and not merely fast, in terms of the
   happens-before edges from Topic 86. Which two `volatile` declarations are doing the
   work, and what would break if `Node.val` were a plain field?

3. `size()` is a striped estimate; a single `AtomicLong` would have been exact. Argue
   the striped choice is correct, then make the strongest case against yourself. Which
   category of caller is harmed, and what should they use instead?

4. A bin treeifies at 8 nodes **only if the table has at least 64 buckets**; otherwise the
   map resizes. Explain why that second condition exists in terms of what a long chain in
   a small table actually tells you, then why untreeify is at 6 rather than 8.

5. `computeIfAbsent`'s mapping function runs under the bin's monitor. That is a
   deliberate design decision with an obvious cost. What does it buy that a
   `putIfAbsent`-with-a-precomputed-value scheme cannot, and under what workload is that
   worth the cost?

6. `CopyOnWriteArrayList` gives you snapshot iterators and lock-free reads for O(n) per
   write. `ConcurrentHashMap` gives you weakly consistent iterators and lock-free reads
   for O(1) per write. Explain why CHM cannot offer snapshot iterators at that price, and
   what it would cost if it tried.

7. Topic 94's `OrderedLocks` used `computeIfAbsent` to build a per-key lock registry.
   Suppose it had used `get` then `put` instead. Describe precisely what breaks — not
   "the map races", but what happens to the mutual exclusion those locks were providing,
   and why that failure would be nearly impossible to diagnose from a thread dump.

---

## Quick reference card

### The mechanical facts

| Question | Answer |
|---|---|
| Segments? | **No.** Removed in Java 8. Segments are the Java 7 design. |
| Read path | Volatile reads only. No lock, no CAS, no shared write. |
| Write path, empty bin | **One CAS.** No lock. |
| Write path, occupied bin | **`synchronized` on the first node of that bin.** |
| Thread state when contended | **`BLOCKED (on object monitor)`** — not `WAITING` |
| Treeify | 8 nodes **and** table >= 64 buckets; untreeify at 6 |
| Resize | Cooperative — writers call `helpTransfer`; `ForwardingNode` has hash `MOVED = -1` |
| `size()` | **Estimate.** `baseCount` + striped `@Contended CounterCell[]` |
| Nulls | **Not allowed**, keys or values. `HashMap` allows them. |
| Iterators | **Weakly consistent.** No `ConcurrentModificationException`, no snapshot. |
| Bulk ops (`forEach`/`search`/`reduce`) | Use `ForkJoinPool.commonPool()` above the parallelism threshold |

### Choosing the atomic method

```java
map.putIfAbsent(k, v)                 // cheap value; returns EXISTING value or null
map.computeIfAbsent(k, k2 -> f(k2))   // expensive value, at most once; f runs UNDER the bin lock
map.computeIfPresent(k, (k2,v) -> g(v))
map.compute(k, (k2,v) -> h(v))        // v is null if absent; null return REMOVES
map.merge(k, 1, Integer::sum)         // the counter idiom
map.replace(k, expected, updated)     // CAS on an entry; you write the retry loop
map.remove(k, expectedValue)          // remove ONLY if the value still matches
ConcurrentHashMap.newKeySet()         // the concurrent Set
```

### The idempotency idiom — the only correct in-JVM shape

```java
CompletableFuture<R> mine = new CompletableFuture<>();
CompletableFuture<R> existing = inFlight.putIfAbsent(key, mine);   // ONE operation
if (existing != null) {
    return existing.get(5, TimeUnit.SECONDS);    // we lost: WAIT, do no work
}
try {
    R r = expensiveWork();                       // no lock held here
    mine.complete(r);
    return r;
} catch (RuntimeException e) {
    inFlight.remove(key, mine);                  // TWO-ARG remove: only if still ours
    mine.completeExceptionally(e);               // release waiters with the real cause
    throw e;
}
```

...and remember that this is a **latency optimisation**, not an idempotency guarantee.
The guarantee is a unique constraint in the database.

### Reading a dump

| Monitor class on `- waiting to lock` | Diagnosis |
|---|---|
| `ConcurrentHashMap$Node` | Ordinary bin contention |
| `ConcurrentHashMap$TreeBin` | 8+ collisions in a 64+ table — check `hashCode` (Topic 13) |
| `ConcurrentHashMap$ReservationNode` | **A `computeIfAbsent` lambda is holding the bin** — Trap 3 |
| All on **one** monitor identity | One hot key |
| Many identities | General write load |

```bash
jcmd $PID Thread.print -l > d1.txt
grep -oE 'ConcurrentHashMap\$[A-Za-z]+' d1.txt | sort | uniq -c | sort -rn
grep 'waiting to lock' d1.txt | awk '{print $4}' | sort | uniq -c | sort -rn
grep 'java.lang.Thread.State' d1.txt | sort | uniq -c | sort -rn
jcmd $PID GC.class_histogram | grep -E 'ConcurrentHashMap\$Node|\[Ljava.lang.Object;'
jfr print --events jdk.JavaMonitorEnter r.jfr | head -60
```

### Gotchas checklist

- [ ] `containsKey` then `put` is a race. So is `get` then `put`. So is `containsKey`
      then `get`.
- [ ] Any read whose result decides a write on the same key must be **one** atomic method.
- [ ] `computeIfAbsent`'s lambda holds the bin lock: **no I/O, no locks, no framework
      calls**.
- [ ] The mapping function must **never** touch the map it is called on.
- [ ] Use the **two-argument** `remove(k, v)` and `replace(k, old, new)` when cleaning up
      an entry you may not still own.
- [ ] `size()` is an estimate. Never use it in a compare-and-act.
- [ ] CHM iterators are weakly consistent, not snapshots. `keySet()` is a live view.
- [ ] No nulls, keys or values. `HashMap` migration will throw `NullPointerException`.
- [ ] CHM contention is `BLOCKED`, not `WAITING`. Check `jdk.JavaMonitorEnter`, not
      `jdk.ThreadPark`.
- [ ] A long-held bin monitor is **not** a deadlock; the detector will say nothing.
- [ ] `CopyOnWriteArrayList` is O(n) per write. Writes per deploy, not per second.
- [ ] Build a `CopyOnWriteArrayList` with `addAll` or the constructor, never in a loop.
- [ ] Bulk `forEach`/`search`/`reduce` use the common pool. Pass `Long.MAX_VALUE` to
      stay sequential.
- [ ] An unbounded map with a request-scoped key is a leak (Topic 79). Bound it and give
      it a TTL.
- [ ] A per-JVM map is not a distributed guarantee. Three pods, three maps.
- [ ] `Collections.synchronizedMap` needs external `synchronized` around iteration, and
      is the wrong choice anyway.
- [ ] A broken or mutable `hashCode` (Topic 13) turns a CHM into a linked list, then a
      tree, then a contention point.

---

## When would I use this at work?

**1. Reviewing any pull request that touches a shared map.**

The diff adds two lines against a `ConcurrentHashMap` and looks obviously correct. You
ask one question: "what happens if two threads run these two lines at the same time?" If
the answer involves a side effect — a charge, a send, an insert — the diff is wrong and
needs a single atomic method. **That question, not a style comment, is what catches the
duplicate wallet debit before it ships.** Then you propose a checklist rule so it stops
depending on the next reviewer noticing: *any `ConcurrentMap` read whose result decides a
write on the same key must be one atomic operation.*

**2. Diagnosing a service that is slow with low CPU.**

Throughput is flat, CPU is 20%, and nothing is obviously broken. You take three thread
dumps, run the state histogram, and see a pile of `BLOCKED` threads on
`ConcurrentHashMap$ReservationNode`. In under two minutes you know that someone put a
database call inside a `computeIfAbsent` lambda, and you can point at the exact line.
**The value is not the fix — it is that you did not spend the afternoon looking at the
database**, which is where the whole team's attention was.

**3. Designing anything that must happen exactly once.**

Payments, notifications, outbound webhooks, order placement. Someone proposes an
in-memory dedupe map. You agree it is a good optimisation, then ask where the guarantee
lives when there are three pods and one of them restarts. That question moves the design
from a data-structure choice to a durability decision — a unique constraint, an outbox, or
an idempotent consumer (Topics 115 and 116) — and the map becomes a cache in front of it.
**"Where does the guarantee live?" is a large part of what separates a senior engineer
from a good one.**

---

## Connected topics

**Prerequisites:**

- **12 — `HashMap` internals.** The bin index `(n-1) & hash`, the power-of-two table, and
  the high-bits XOR spread are the same here. CHM adds the sign-bit mask, per-bin locking
  and cooperative resize on top of a structure you already know.
- **13 — the `equals`/`hashCode` contract.** A broken `hashCode` collapses a CHM into one
  bin, which turns per-bin locking into a single global lock and then into a `TreeBin`.
  A **mutable** key is worse: the entry becomes unreachable but is still retained, so it
  is both a correctness bug and a leak. Every guarantee in this document assumes the
  contract holds.
- **15 — `LinkedHashMap` and LRU.** Why you use Caffeine rather than hand-rolling
  eviction, and why "bounded with a TTL" is a product decision rather than a constant.
- **17 / 88 — immutability and safe publication.** `CopyOnWriteArrayList`'s volatile
  array swap **is** safe publication, and the `final` field freeze is why an immutable
  value in a concurrent map needs no further synchronisation.
- **25 — parallel streams.** CHM's bulk operations and any `.parallelStream()` over
  `keySet()` run on the common pool — Topic 91's shared, `cores − 1`, JVM-wide resource.
- **79 — leaks and reachability.** An unbounded map keyed by anything request-scoped is
  the canonical Java leak: a reachability problem, not an allocation problem, findable
  only in a heap dump.
- **85 — `synchronized`, monitors and inflation.** CHM's bin lock is an ordinary intrinsic
  monitor — a CAS on the mark word uncontended, inflating under contention. That is why
  contention shows as `BLOCKED` and appears under `jdk.JavaMonitorEnter`.
- **86 / 87 — the JMM and `volatile`.** `volatile V val` and `volatile Node next` make
  lock-free reads *correct*. Without those edges a reader could see a stale value
  forever — not briefly, forever.

**This unlocks:**

- **93 — `BlockingQueue`.** The bounded, blocking sibling of `ConcurrentLinkedQueue`. If
  a consumer needs to *wait*, a concurrent collection is the wrong tool.
- **94 — explicit locks.** `computeIfAbsent` is how a per-key lock registry is built
  correctly. If it were `get` then `put`, two threads would get two different lock
  objects for one key and every lock in the system would silently stop excluding.
- **95 — atomics and `LongAdder`.** `CounterCell` **is** `Striped64`. CHM's `size()` and
  `LongAdder` are the same code answering the same question; understanding one gives you
  the other.
- **96 — false sharing.** `@Contended` on `CounterCell` is the concrete reason that
  annotation exists in the JDK, and the clearest real example of padding earning its
  keep.
- **97 — coordination primitives.** A `Semaphore` is the correct way to bound a
  concurrent collection; `map.size() < limit` is not.
- **98 — the concurrency bug taxonomy.** This topic's check-then-act is the canonical
  **race**; Trap 3's held bin monitor is the case where the deadlock detector is silent
  and Topic 98's decision table is what you need instead.
- **99 — jcstress.** How you would *prove* the atomic version correct rather than failing
  to reproduce the race and calling it fixed. Proof 2 can only show the bug.
- **101 — virtual threads.** Millions of virtual threads on one CHM changes the contention
  profile substantially — far more threads, the same number of bins — and a `synchronized`
  bin lock is the shape that historically pinned carriers.
- **110 — Spring Cache and Redis.** The distributed version: the check-then-act reappears
  as `GET` followed by `SET`, fixed with `SET NX` — `putIfAbsent` over the network.
- **116 — idempotency.** The full treatment of Fix 3, and why an in-memory map can only
  ever be an optimisation.
- **118 — metrics and cardinality.** Hit ratio, evictions and a duplicate-detected counter
  are this topic's permanent instrumentation; tagging any of them with the idempotency key
  kills the monitoring system.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted: exactly which recursive `computeIfAbsent` cases your JDK
detects and throws `IllegalStateException: Recursive update` for (it is implementation
behaviour, it has changed, and Proof 3 settles it on your machine in two minutes);
`CopyOnWriteArrayList`'s internal lock type, which was a `ReentrantLock` in Java 8 and a
plain monitor later — `javap -p` settles it; and whether your platform exposes hardware
cache counters at all, which it does not on macOS. Everything else — that segments were
removed in Java 8, that reads are lock-free volatile reads, that a write CASes an empty
bin or `synchronized`s on the first node of an occupied one, that `size()` is a striped
estimate, and above all that atomic operations do not compose into atomic sequences — is
stable, checkable with the commands above, and will still be true the next time somebody
writes two lines that both pass review and charge a customer twice.*
