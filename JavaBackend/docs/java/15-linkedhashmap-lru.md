# 15 — LinkedHashMap and an LRU Cache

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Third and last version of the cloakroom.

Topic 12: coats on numbered pegs. Fast to find, but if you walk the room the
coats come at you in peg order, which is meaningless.

Topic 14: coats on a shelf in alphabetical order. Meaningful order, but you pay
for it on every single hang-up.

Now: coats on numbered pegs **and** a piece of string threaded through every coat
in the order they arrived.

Finding a coat is still the fast peg lookup — the string changes nothing about
that. But if you want to walk the coats in arrival order, you just follow the
string. You get both.

Now the trick that makes this topic worth its own chapter.

Change one rule: **every time someone looks at a coat, you unthread it and
re-thread it at the very end of the string.** Do nothing else.

Follow the consequence. The coat at the *front* of the string is now, by
construction, the coat nobody has looked at for the longest time. If the
cloakroom is full and you must throw one out, you do not need to search, sort, or
timestamp anything. You cut the front of the string.

That is a **least-recently-used cache**, and in Java it is one constructor
argument and one four-line method override.

---

## The bridge from what you know

### What transfers

Iteration order.

```ts
const m = new Map<string, number>();
m.set("SKU-3", 3);
m.set("SKU-1", 1);
m.set("SKU-2", 2);
[...m.keys()];      // ["SKU-3", "SKU-1", "SKU-2"] — insertion order, GUARANTEED
```

```java
Map<String, Integer> m = new LinkedHashMap<>();
m.put("SKU-3", 3);
m.put("SKU-1", 1);
m.put("SKU-2", 2);
m.keySet();          // [SKU-3, SKU-1, SKU-2] — insertion order, guaranteed
```

`LinkedHashMap` is the Java collection whose behaviour matches your default
mental model of a JavaScript `Map`. If you have been writing `HashMap` and
assuming insertion order, this is the class you actually meant.

**Verdict: HONEST ANALOGUE** — for insertion-order mode only.

### What does not transfer

| TypeScript | Java `LinkedHashMap` | Verdict |
|---|---|---|
| `Map` iterates in insertion order | Same, in the default mode | **HONEST ANALOGUE** |
| No access-order mode exists | `accessOrder = true` reorders on every `get` | **NO ANALOGUE** |
| No eviction hook of any kind | `removeEldestEntry` is called after every insert | **NO ANALOGUE** |
| An LRU cache needs a library or ~40 lines of hand-written doubly-linked-list code | An LRU cache is six lines | **NO ANALOGUE** |
| `map.get()` is a pure read | With `accessOrder`, `get()` **structurally mutates the map** | **NO ANALOGUE — and this is a concurrency bug waiting to happen** |
| Single-threaded, so shared mutable state is a non-issue | Two threads calling `get()` can corrupt the linked list | **NO ANALOGUE** |

The LRU trick genuinely has no JavaScript counterpart. In Node you would reach
for `lru-cache` from npm, or hand-roll a `Map` plus manual delete-and-reinsert
(which works, because JS `Map` preserves insertion order and `delete`+`set` moves
a key to the end — but it is your code, not the platform's).

The last two rows are the ones to hold on to. Your entire Node mental model says
"a read is a read". Here, in access-order mode, it is not. That fact is the
source of the most surprising bug in this topic.

---

## What is this?

`LinkedHashMap` **extends** `HashMap`. Everything from Topic 12 is still true:
same table, same spread hash, same `(n-1) & hash` indexing, same resize, same
treeification, same load factor.

It adds exactly one thing: a **doubly-linked list threaded through every entry**,
independent of which bucket the entry is in.

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;      // the linked list — the "string"
    // inherited from Node: hash, key, value, next  <- 'next' is the BUCKET chain
}
```

Two separate link structures per entry, doing two separate jobs:

| Field | Job |
|---|---|
| `next` | the collision chain within one bucket (Topic 12) |
| `before` / `after` | the iteration order across the whole map |

Plus two fields on the map itself: `head` and `tail`.

### The three hooks

`HashMap` has three empty methods that exist purely so `LinkedHashMap` can
subclass it without duplicating any logic:

```java
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

`LinkedHashMap` overrides all three:

- `afterNodeAccess` — in access-order mode, unlink the entry and re-link it at the
  tail. This is the "move to the end of the string" step.
- `afterNodeInsertion` — call `removeEldestEntry(head)`; if it returns `true`,
  remove the head. This is the eviction hook.
- `afterNodeRemoval` — unlink from the doubly-linked list.

That is the entire class. It is roughly 800 lines of which most is Javadoc.

### The two modes

```java
new LinkedHashMap<>()                          // insertion order
new LinkedHashMap<>(16, 0.75f, false)          // insertion order (explicit)
new LinkedHashMap<>(16, 0.75f, true)           // ACCESS order
```

That third argument is the whole feature.

| Mode | An entry moves to the end when... |
|---|---|
| insertion order (default) | it is first inserted. A `put` over an existing key does **not** move it |
| access order | it is `get`, `getOrDefault`, `put`, `merge`, `compute`, `computeIfAbsent` (on a hit), `putIfAbsent`, or `replace` |

**Not** counted as an access, in either mode:

- `containsKey` / `containsValue`
- `entrySet()` / `keySet()` / `values()` and iterating them
- `forEach`
- `size()`, `isEmpty()`
- `getOrDefault` on a **miss**

That list matters. `containsKey` not counting as an access is the single most
common way a hand-rolled LRU quietly stops being an LRU.

---

## Why does it matter?

**1. It is the collection you actually meant most of the time.**

Almost every place a `HashMap` appears in a codebase, the developer was
implicitly assuming a stable iteration order — for a log line, a serialized JSON
body, a test assertion, a report. `HashMap` does not give you one (Topic 12).
`LinkedHashMap` does, for two extra references per entry and no change in
lookup cost. When order matters at all, it should be in the type.

**2. It is the smallest correct LRU cache in any mainstream language.**

Six lines. Not a library, not a dependency, not a design discussion. That makes
it the right answer for a bounded per-request memo, a single-threaded batch job's
working set, or a test double for a real cache.

**3. Knowing why you would *not* ship it is the senior half of the topic.**

`LinkedHashMap` LRU is not thread-safe, has no TTL, no statistics, no eviction
listener, no refresh, no weighting, and LRU is a mediocre eviction policy on real
access patterns. Being able to write it in six lines *and* explain why you would
reach for Caffeine instead is exactly the Mid → Senior gap the curriculum is
targeting.

---

## Syntax breakdown

### Anonymous subclass with an instance initialiser

```java
Map<String, Product> cache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override protected boolean removeEldestEntry(Map.Entry<String, Product> eldest) {
        return size() > 500;
    }
};
```

That `{ ... }` after the constructor call creates an **anonymous subclass**. You
are not configuring a `LinkedHashMap`; you are declaring a new nameless class
that extends it and overrides one method, and instantiating it in the same
expression.

There is no TypeScript equivalent. The closest is
`class extends LinkedHashMap { ... }` as an expression, which TS/JS does support
(`const C = class extends B {}`), but you would never write it inline like this.

`[LEGACY — still asked]` The same syntax with a *second* brace block is the
"double-brace initialisation" idiom:
```java
Map<String, String> m = new HashMap<>() {{ put("a", "1"); put("b", "2"); }};
```
Do not use it. It creates an anonymous class per site, each holding a hidden
reference to the enclosing instance — a classic leak shape (Topic 79) and a
serialization hazard. Use `Map.of(...)` instead. Interviewers ask about it
specifically to see whether you know why it is bad.

### `protected` and overriding

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest)
```

`protected` means visible to subclasses and to the same package (Topic 03). The
JDK's default implementation returns `false` — so `LinkedHashMap` never evicts
unless you subclass it and say otherwise. This is a **template method**: the JDK
calls a method you supply. It is the same pattern as a lifecycle hook, and it is
the only extension point on the class.

### `SequencedMap` (new in Java 21)

Java 21 added `SequencedCollection`, `SequencedSet` and `SequencedMap`. Since
`LinkedHashMap` implements `SequencedMap`, you get:

```java
map.firstEntry();       // the eldest — your LRU eviction candidate
map.lastEntry();        // the most recently inserted/accessed
map.pollFirstEntry();   // remove and return the eldest
map.pollLastEntry();
map.putFirst(k, v);
map.putLast(k, v);
map.reversed();         // a live reversed VIEW of the map
map.sequencedKeySet();
```

`LinkedHashSet` gets the `SequencedSet` equivalents: `getFirst`, `getLast`,
`addFirst`, `addLast`, `removeFirst`, `removeLast`, `reversed`.

Before Java 21, getting the first entry of a `LinkedHashMap` meant
`map.entrySet().iterator().next()`. That is the Java 17 fallback if your team is
not on 21 yet.

> **One point of genuine uncertainty.** I am fairly confident that `putFirst` and
> `putLast` throw `UnsupportedOperationException` on a `LinkedHashMap` in
> **access-order mode**, because reordering by position conflicts with the mode's
> own reordering rule. I cannot verify that without a JVM. Proof 5 below is three
> lines and settles it. `firstEntry`, `lastEntry`, `pollFirstEntry` and
> `reversed()` work in both modes.

---

## Example 1 — minimal

```java
import java.util.*;

public class OrderVsAccessOrder {
    public static void main(String[] args) {

        Map<String, Integer> insertion = new LinkedHashMap<>();
        Map<String, Integer> access    = new LinkedHashMap<>(16, 0.75f, true);
        Map<String, Integer> hash      = new HashMap<>();

        for (Map<String, Integer> m : List.of(insertion, access, hash)) {
            m.put("SKU-A", 1);
            m.put("SKU-B", 2);
            m.put("SKU-C", 3);
        }

        System.out.println("insertion, fresh : " + insertion.keySet());
        System.out.println("access,    fresh : " + access.keySet());
        System.out.println("hash,      fresh : " + hash.keySet());

        // Touch the first key in each.
        insertion.get("SKU-A");
        access.get("SKU-A");
        hash.get("SKU-A");

        System.out.println();
        System.out.println("insertion, after get(A) : " + insertion.keySet());
        System.out.println("access,    after get(A) : " + access.keySet());
        System.out.println("hash,      after get(A) : " + hash.keySet());

        // containsKey does NOT count as an access.
        access.containsKey("SKU-B");
        System.out.println("access, after containsKey(B) : " + access.keySet());
    }
}
```

Run it. Three things to notice, in increasing order of importance:

1. Both `LinkedHashMap`s start in insertion order. The `HashMap` does not.
2. After `get("SKU-A")`, the access-order map has moved `SKU-A` to the end. The
   insertion-order map has not. **A read changed the map.**
3. `containsKey("SKU-B")` did nothing. If you built an LRU on top of
   `containsKey` followed by `get`, only the `get` would count — and if you used
   `containsKey` alone as a "did we cache this" check, your recency tracking
   would be silently wrong.

---

## Example 2 — production scenario

`orderflow`'s `ProductCatalogService` serves product detail. Every order-placement
request looks up 1–20 products, the catalogue has 800,000 rows, and the database
round trip is the dominant cost of the endpoint.

### The version that ships and then causes an incident

```java
@Service
public class ProductCatalogService {

    private final ProductRepository repository;

    // "Small cache, no big deal."
    private final Map<String, Product> cache = new HashMap<>();

    public Product bySku(String sku) {
        Product cached = cache.get(sku);
        if (cached != null) {
            return cached;
        }
        Product loaded = repository.findBySku(sku);
        cache.put(sku, loaded);
        return loaded;
    }
}
```

Four defects, each with a distinct symptom.

**Defect 1 — unbounded.** There is no eviction. The map grows to hold every SKU
ever requested. With 800,000 products it will eventually hold all of them, plus
whatever each `Product` transitively references. Symptom: heap usage climbing
slowly over days, an eventual `OutOfMemoryError`, and a heap dump whose dominator
tree names this exact map. This is the single most common Java leak shape and it
is Topic 79's drill.

**Defect 2 — not thread-safe.** This is a Spring singleton (Topic 38) serving
concurrent requests. `HashMap.put` from two threads can lose an update, corrupt
`size()`, or leave the table in a state where a subsequent `get` never
terminates. Symptom: intermittent wrong results, a `size()` that does not match
reality, and — occasionally — a request thread pinned at 100% CPU forever.

**Defect 3 — no TTL.** A price change lands in the database. This cache never
notices. Symptom: customers quoted stale prices, indefinitely, with no way to
flush short of a restart. Support tickets that nobody can reproduce because the
other pod has a different cache.

**Defect 4 — caches nulls as misses.** `repository.findBySku` returns `null` for
an unknown SKU. `cache.put(sku, null)` stores it. The next call's
`cache.get(sku) != null` check fails, so it hits the database again — the cache
never absorbs negative lookups. A scanner probing random SKUs walks straight
through to the database on every request. Symptom: database load that does not
correlate with legitimate traffic.

### Step one — the six-line bounded LRU

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxEntries;

    public LruCache(int maxEntries) {
        super(16, 0.75f, true);                      // <- access order
        this.maxEntries = maxEntries;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxEntries;                  // <- the eviction policy
    }
}
```

That is the whole thing. Six meaningful lines. Read each one:

| Line | What it does |
|---|---|
| `extends LinkedHashMap<K, V>` | inherit the hash table and the linked list |
| `super(16, 0.75f, true)` | the `true` is access-order mode. Without it this is a FIFO cache, not an LRU |
| `removeEldestEntry` | called by `afterNodeInsertion` after **every** insert |
| `return size() > maxEntries` | `size()` is post-insert, so `>` gives you exactly `maxEntries`. `>=` would give `maxEntries - 1` |
| `eldest` | the head of the linked list — the least recently accessed entry |

It bounds memory and evicts sensibly. It does **not** fix thread safety, TTL, or
null-caching. Do not ship it into a Spring singleton.

### Step two — what you actually ship

```java
@Service
public class ProductCatalogService {

    private final ProductRepository repository;
    private final Cache<String, Optional<Product>> cache;

    public ProductCatalogService(ProductRepository repository, MeterRegistry meters) {
        this.repository = repository;
        this.cache = Caffeine.newBuilder()
                .maximumSize(50_000)
                .expireAfterWrite(Duration.ofMinutes(10))
                .refreshAfterWrite(Duration.ofMinutes(2))
                .recordStats()
                .build();
        CaffeineCacheMetrics.monitor(meters, cache, "product-catalogue");
    }

    public Optional<Product> bySku(String sku) {
        return cache.get(sku, k -> Optional.ofNullable(repository.findBySku(k)));
    }
}
```

What each piece buys, mapped back to the four defects:

| Feature | Fixes |
|---|---|
| `maximumSize(50_000)` | Defect 1 — bounded, with W-TinyLFU admission rather than plain LRU |
| thread-safe by design | Defect 2 — Caffeine is built on `ConcurrentHashMap` (Topic 92) |
| `expireAfterWrite(10m)` | Defect 3 — stale prices self-correct within a bounded window |
| `Optional<Product>` as the value | Defect 4 — a miss is a cached `Optional.empty()`, not a `null` |
| `cache.get(key, loader)` | the loader runs **once per key** even under concurrent misses — no thundering herd |
| `recordStats()` + metrics | you can see hit rate. An unmeasured cache is a guess |
| `refreshAfterWrite(2m)` | a stale-but-usable entry is served while it refreshes in the background |

### The honest note you must be able to give

**Caffeine is what you ship. `LinkedHashMap` is what you understand.**

Concretely, here is what Caffeine has that six lines cannot:

- **W-TinyLFU eviction**, not LRU. It keeps a compact frequency sketch and admits
  a new entry only if it is likely to be used more than the victim it would
  displace. The failure mode this fixes is real and common: a single large scan —
  a reporting query, a sitemap crawl, a batch job — walks a million SKUs through
  a plain LRU and evicts the entire genuinely-hot working set, because LRU has no
  notion that those million entries were each seen exactly once. Hit rate goes to
  zero and stays there. W-TinyLFU refuses to admit them.
- **Time-based expiry** — `expireAfterWrite`, `expireAfterAccess`, and a variable
  expiry per entry.
- **Asynchronous refresh** so a hot key is never a latency cliff at expiry.
- **Statistics** — hit rate, miss rate, load time, eviction count — which is the
  only way to know your cache is doing anything at all.
- **Weight-based bounds** (`maximumWeight`), because 50,000 small products and
  50,000 large ones are not the same memory footprint.
- **Removal listeners**, for when eviction must trigger cleanup.
- **Concurrency**, built on `ConcurrentHashMap`, with amortised bookkeeping that
  does not serialise readers.

And the thread-safety point deserves its own paragraph, because it is
counter-intuitive:

> **In access-order mode, `get()` is a structural modification.** It unlinks and
> re-links a node in the doubly-linked list. Two threads calling `get()`
> concurrently — with no `put` anywhere in sight — can corrupt the list, produce
> a wrong iteration order, drop entries, or loop forever. `Collections.synchronizedMap`
> around it works, but it serialises every read, which defeats most of the reason
> you built a cache. This is the trap: `LinkedHashMap` in access-order mode is
> the one collection where "we only read it concurrently" is not a safe defence.

Guava's `Cache` is the older sibling of Caffeine, by the same author; Caffeine
supersedes it and Spring's `@Cacheable` supports both. If you are on Spring Boot,
`spring-boot-starter-cache` plus `caffeine` on the classpath auto-configures it —
Topic 42.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — forgetting `accessOrder = true`

**Wrong:**
```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    public LruCache(int max) {
        super(16, 0.75f);              // <-- no third argument: INSERTION order
        this.max = max;
    }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > max; }
}
```

**Exact symptom:** the cache works, evicts, and stays bounded — but the hit rate
is much lower than expected and does not improve with size. Your hottest keys get
evicted while cold ones survive. Nothing throws. If you are not recording
statistics, you never find out at all.

**Root cause:** without access order, the linked list is in insertion order, so
`eldest` is the oldest *inserted* entry regardless of how often it has been used.
You have built a FIFO cache and called it an LRU.

**Fix:** `super(16, 0.75f, true)`. And record hit-rate statistics, because
otherwise this class of bug is undetectable.

---

### Trap 2 — `containsKey` before `get`

**Wrong:**
```java
if (cache.containsKey(sku)) {
    return cache.get(sku);
}
```

**Exact symptom:** subtly worse hit rate than the same code written differently.
No error. Very hard to spot in review because the code looks more careful, not
less.

**Root cause:** `containsKey` does not trigger `afterNodeAccess`, so it does not
count as a use. In this snippet the following `get` *does*, so the recency is
still updated — but any code path that calls `containsKey` **without** a
following `get` silently fails to mark the entry as used. It is a latent bug that
activates the moment someone adds an early return.

**Fix:** one lookup, not two — which is faster anyway.
```java
Product cached = cache.get(sku);
if (cached != null) return cached;
```
Or, if `null` is a legal value, `cache.getOrDefault(sku, SENTINEL)`.

---

### Trap 3 — sharing it across threads

**Wrong:**
```java
@Service
public class PricingService {
    private final Map<String, BigDecimal> cache = new LruCache<>(10_000);   // singleton
}
```

**Exact symptom:** intermittent and varied. Wrong values returned. `size()` that
exceeds `maxEntries`. Entries that vanish. A `ConcurrentModificationException`
from an iteration that had no visible concurrent write. Occasionally a request
thread stuck at 100% CPU that a thread dump shows parked inside
`LinkedHashMap.afterNodeAccess` or `HashMap.getNode`. All of it load-dependent,
none of it reproducible locally.

**Root cause:** `LinkedHashMap` is not thread-safe, and in access-order mode even
concurrent `get()` calls mutate the shared doubly-linked list. Two threads
re-linking the same node race on `before`/`after` pointers.

**Fix, in order of preference:**
1. Use Caffeine. This is what it is for.
2. If you cannot add a dependency and the cache is small and hot-path-cold:
   `Collections.synchronizedMap(new LruCache<>(n))` — correct, but every read
   takes the same lock, and you must also synchronise manually around any
   iteration.
3. Confine it to one thread — a per-request memo, a single-threaded batch job's
   working set. This is genuinely fine and needs no library.

---

### Trap 4 — `removeEldestEntry` with a side effect or a stale bound

**Wrong:**
```java
@Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    if (size() > max) {
        auditLog.record("evicting " + eldest.getKey());   // I/O inside the hook
        externalIndex.remove(eldest.getKey());            // mutating something else
        return true;
    }
    return false;
}
```

**Exact symptom:** every `put` to the cache becomes as slow as your audit log.
Under load, the cache is the bottleneck. Worse: if `externalIndex` is another map
guarded by a lock, you have just created a lock ordering you did not design
(Topic 94), and if it re-enters this cache you get a stack overflow or a
deadlock.

**Root cause:** `removeEldestEntry` is called from inside `afterNodeInsertion`,
which is inside `putVal`, which is on the hot path of every insert. It is a
predicate. It is supposed to answer a question, not do work.

**Fix:** the method returns a boolean and does nothing else.
```java
@Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > maxEntries;
}
```
If you need a genuine eviction listener, that is a Caffeine feature
(`removalListener`), and Caffeine runs it off the calling thread by default.

Also note: `removeEldestEntry` returning `true` removes exactly **one** entry per
insert. If you lower `maxEntries` at runtime from 10,000 to 100, the map does not
shrink to 100 — it shrinks by one entry per subsequent insert. If you need it to
shrink now, you must remove entries yourself.

---

### Trap 5 — `LinkedHashMap` iteration cost, in both directions

**Wrong (direction one):**
```java
Map<Long, Order> index = new HashMap<>(4_000_000);   // huge, mostly empty
for (Map.Entry<Long, Order> e : index.entrySet()) { ... }
```

**Exact symptom:** an iteration that is far slower than the entry count suggests.
Profiling shows time in `HashMap$HashIterator.nextNode`.

**Root cause:** `HashMap` iteration walks the **entire table array**, skipping
nulls. Cost is O(capacity + size), not O(size). A 4-million-slot table with 1,000
entries costs 4 million array reads to iterate.

**Fix:** `LinkedHashMap` iteration follows the linked list and is O(size) exactly.
For a large, sparse, frequently-iterated map, `LinkedHashMap` is genuinely
*faster* to iterate than `HashMap` — which surprises people who assume it is
strictly more expensive.

**Wrong (direction two):**
```java
Map<Long, OrderLine> lines = new LinkedHashMap<>();   // 50 million entries, never iterated
```

**Exact symptom:** roughly 8–16 extra bytes of heap per entry versus `HashMap`,
which at 50 million entries is hundreds of megabytes for nothing.

**Root cause:** two extra reference fields (`before`, `after`) per `Entry`.

**Fix:** use `LinkedHashMap` when order or eviction is a requirement. Not by
default, and not "just in case".

---

## Hands-on proof

Everything here is a command **you** run. I do not have a JVM and will not
fabricate output.

### Setup

```bash
mkdir -p ~/java-lab/15 && cd ~/java-lab/15
java --version          # expect 21 or 25
```

### Proof 1 — access order versus insertion order

Save Example 1 as `OrderVsAccessOrder.java` and run it:

```bash
java OrderVsAccessOrder.java
```

| What you see | What it means |
|---|---|
| Both `LinkedHashMap` rows show `[SKU-A, SKU-B, SKU-C]` initially | Confirmed: insertion order, guaranteed, in both modes |
| After `get(A)`, the access row shows `[SKU-B, SKU-C, SKU-A]` | Confirmed: **a read reordered the map**. This is the entire feature and the entire hazard |
| After `get(A)`, the insertion row is unchanged | Confirmed: `accessOrder=false` ignores reads |
| After `containsKey(B)`, the access row is unchanged | Confirmed: `containsKey` is not an access. This is Trap 2 |
| The `HashMap` row is in some other order entirely | Confirmed: Topic 12. Do not rely on it |

### Proof 2 — the six-line LRU actually evicts the right thing

`LruProof.java`:
```java
import java.util.*;

public class LruProof {

    static class LruCache<K, V> extends LinkedHashMap<K, V> {
        private final int max;
        LruCache(int max) { super(16, 0.75f, true); this.max = max; }
        @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > max; }
    }

    static class FifoCache<K, V> extends LinkedHashMap<K, V> {
        private final int max;
        FifoCache(int max) { super(16, 0.75f, false); this.max = max; }   // no access order
        @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > max; }
    }

    public static void main(String[] args) {
        for (Map<String, Integer> cache : List.of(new LruCache<String,Integer>(3),
                                                  new FifoCache<String,Integer>(3))) {
            cache.put("A", 1);
            cache.put("B", 2);
            cache.put("C", 3);
            cache.get("A");            // A is now the most recently USED
            cache.put("D", 4);         // forces one eviction

            System.out.printf("%-10s after put(D): %s%n",
                cache.getClass().getSimpleName(), cache.keySet());
        }
    }
}
```

```bash
java LruProof.java
```

| What you see | What it means |
|---|---|
| `LruCache after put(D): [C, A, D]` | Confirmed: **B** was evicted — the least recently *used*. A survived because `get("A")` refreshed it |
| `FifoCache after put(D): [B, C, D]` | Confirmed: **A** was evicted — the oldest *inserted*. The `get` was ignored. This is Trap 1, demonstrated |
| Both show the same keys | You passed the same boolean to both constructors. Check the `super(...)` calls |
| Either has 4 entries | `removeEldestEntry` is not being called — check the `@Override` compiles, and that the signature is exactly `Map.Entry<K,V>` |

The `@Override` annotation is the safety net here: without it, a typo in the
method name or signature compiles fine and silently never evicts.

### Proof 3 — read the source

```bash
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/LinkedHashMap.java' -d ~/java-lab/15/src
less ~/java-lab/15/src/java.base/java/util/LinkedHashMap.java
```

Read these four things:

| Search for | What you are confirming |
|---|---|
| `static class Entry<K,V> extends HashMap.Node` | the `before`/`after` fields, and that `next` is inherited and still means the bucket chain |
| `void afterNodeAccess` | the unlink-and-relink-at-tail logic, and the `if (accessOrder && (last = tail) != e)` guard |
| `void afterNodeInsertion` | `removeEldestEntry(first)` and `removeNode(...)` — one entry per insert |
| `protected boolean removeEldestEntry` | the default `return false`, and the Javadoc's own LRU example |

| What you see | What it means |
|---|---|
| `afterNodeAccess` starts with `if (accessOrder && ...)` | Confirmed: in insertion mode the method returns immediately. Zero cost |
| `afterNodeAccess` writes `p.before`, `p.after`, `head`, `tail` | Confirmed: a `get` performs **four or more field writes on shared state**. This is why concurrent reads are unsafe |
| `afterNodeInsertion` removes exactly one node | Confirmed: lowering the bound at runtime does not shrink the map immediately |
| The class Javadoc contains a worked LRU example | Confirmed: this use is blessed by the JDK, not a hack |

Now do the same for `HashMap`, and find the three empty hook methods:
```bash
grep -n "afterNode" ~/java-lab/12/src/java.base/java/util/HashMap.java
```
Seeing three empty methods in `HashMap` whose only purpose is to be overridden by
`LinkedHashMap` is a good lesson in itself about designing for extension.

### Proof 4 — break it with two threads

`ConcurrentBreak.java`:
```java
import java.util.*;
import java.util.concurrent.*;

public class ConcurrentBreak {
    static class LruCache<K, V> extends LinkedHashMap<K, V> {
        private final int max;
        LruCache(int max) { super(16, 0.75f, true); this.max = max; }
        @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > max; }
    }

    public static void main(String[] args) throws Exception {
        Map<Integer, Integer> cache = new LruCache<>(1000);
        for (int i = 0; i < 1000; i++) cache.put(i, i);

        int threads = 8;
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        CountDownLatch go = new CountDownLatch(1);

        for (int t = 0; t < threads; t++) {
            pool.submit(() -> {
                go.await();
                for (int i = 0; i < 2_000_000; i++) {
                    cache.get(ThreadLocalRandom.current().nextInt(1000));   // READS ONLY
                }
                return null;
            });
        }
        go.countDown();
        pool.shutdown();
        boolean done = pool.awaitTermination(60, TimeUnit.SECONDS);

        System.out.println("finished in time : " + done);
        System.out.println("size()           : " + cache.size());
        int counted = 0;
        for (Map.Entry<Integer, Integer> e : cache.entrySet()) counted++;
        System.out.println("counted by iter  : " + counted);
    }
}
```

```bash
java ConcurrentBreak.java
```

Run it several times. This is a race, so it is not deterministic.

| What you see | What it means |
|---|---|
| `size()` and `counted by iter` disagree | Confirmed: the linked list has been corrupted by concurrent **reads**. The table and the list now disagree about what is in the map |
| `finished in time : false` | A thread is stuck — most likely looping in a corrupted `before`/`after` chain. Take a thread dump with `jcmd <pid> Thread.print` and look for a thread inside `LinkedHashMap` |
| `ConcurrentModificationException` from the final iteration | Confirmed: `modCount` changed during iteration, from reads. Fail-fast doing its job (Topic 10) |
| Everything looks fine, every run | Entirely possible — races are timing-dependent. Raise the thread count and the iteration count, and try on a machine with more cores. **Not observing it is not evidence of safety** |
| It crashes with `NullPointerException` inside `LinkedHashMap` | Also a valid outcome. The list is corrupt |

Then change one thing — `super(16, 0.75f, false)` — and re-run.

| What you see | What it means |
|---|---|
| Now stable across runs | Confirmed: in insertion-order mode `get` does not touch the list, so read-only concurrency does not corrupt it. **This still does not make the class thread-safe** — add one concurrent `put` and you are back to Topic 12's problems |

That last row is the point. The fix is not "use insertion order". The fix is
"use a class designed for concurrency".

### Proof 5 — settle the SequencedMap question

```java
// SeqCheck.java
import java.util.*;

public class SeqCheck {
    public static void main(String[] args) {
        LinkedHashMap<String,Integer> ins = new LinkedHashMap<>();
        LinkedHashMap<String,Integer> acc = new LinkedHashMap<>(16, 0.75f, true);
        ins.put("A", 1); acc.put("A", 1);

        System.out.println("insertion firstEntry : " + ins.firstEntry());
        System.out.println("access    firstEntry : " + acc.firstEntry());
        System.out.println("access    reversed   : " + acc.reversed());

        for (var m : List.of(ins, acc)) {
            try {
                m.putFirst("Z", 26);
                System.out.println("putFirst OK  -> " + m.keySet());
            } catch (UnsupportedOperationException e) {
                System.out.println("putFirst threw UnsupportedOperationException");
            }
        }
    }
}
```

```bash
java SeqCheck.java
```

| What you see | What it means |
|---|---|
| `putFirst OK` on the insertion map, `UnsupportedOperationException` on the access map | My stated expectation is confirmed: position-based insertion conflicts with access-order mode |
| `putFirst OK` on both | My expectation was wrong. Believe your JVM, note it, and treat that as the answer |
| `firstEntry` works on both | Expected — this is your LRU eviction candidate, available without an iterator |
| A compile error on `firstEntry` | You are not on Java 21. `SequencedMap` arrived in 21. Fall back to `map.entrySet().iterator().next()` |

This is exactly the kind of question you should never accept from a document,
including this one. Three lines and a `java` command beats any amount of
confident prose.

---

## Practice exercises

### 1 — Easy: prove the eviction policy

Write a cache with `maxEntries = 3`. Insert A, B, C. Then perform a specific
sequence of `get` and `put` calls of your own design, and **predict in a comment,
before running, which key will be evicted** at each step. Run it and check.

Requirements:
- At least one step must use `containsKey` and you must predict correctly that it
  changes nothing.
- At least one step must use `put` on an **existing** key. Predict whether that
  counts as an access in each mode, then verify.
- Do the whole exercise twice, once with `accessOrder=true` and once `false`, and
  produce a table of the two eviction sequences side by side.

### 2 — Medium: the audit (combines Topics 01, 10, 12, 13)

Here is an `orderflow` session cache with **six** defects from Topics 01–15. Find
them all, state the exact production symptom of each, and rewrite it.

```java
@Service
public class SessionCache {

    private static final Map<Long, CustomerSession> SESSIONS = new LinkedHashMap<>(1000) {
        @Override protected boolean removeEldestEntry(Map.Entry<Long, CustomerSession> e) {
            return size() >= 1000;
        }
    };

    public CustomerSession get(Long customerId) {
        if (SESSIONS.containsKey(customerId)) {
            return SESSIONS.get(customerId);
        }
        return null;
    }

    public void touch(Long customerId, CustomerSession session) {
        session.lastSeen = Instant.now();
        SESSIONS.put(customerId, session);
    }

    public int activeCount() {
        int n = 0;
        for (Long id : SESSIONS.keySet()) {
            if (SESSIONS.get(id).lastSeen.isAfter(Instant.now().minusSeconds(300))) n++;
        }
        return n;
    }
}
```

Hints so you look in the right places: one defect from Topic 01, one from
Topic 12, one from Topic 13, and three from this topic. One of them is a
`ConcurrentModificationException` that only fires under a specific access order —
say which and why. Another causes the cache to hold 999 entries instead of 1000
and nobody ever notices; explain why that one is worth fixing anyway.

### 3 — Hard: production simulation

Build the `orderflow` product cache three ways and measure the difference that
eviction policy makes.

**Part A — the workload.** Generate a realistic access trace over 800,000 SKUs:
- 80% of requests hit a hot set of 5,000 SKUs, chosen with a Zipf-like skew.
- 20% hit uniformly random SKUs from the full catalogue.
- Then, halfway through the trace, inject a **scan**: 200,000 sequential unique
  SKUs, each requested once. This simulates a reporting job or a sitemap crawl.

**Part B — three caches, same bound (10,000 entries):**
1. `LinkedHashMap` with `accessOrder=false` (FIFO).
2. `LinkedHashMap` with `accessOrder=true` (LRU).
3. Caffeine with `maximumSize(10_000)` and `recordStats()`.

Instrument all three with a hit counter. For Caffeine, use its own `stats()` and
check your counter agrees with it.

**Part C — report:**
- Overall hit rate for each.
- Hit rate **during** the scan.
- Hit rate in the 100,000 requests **immediately after** the scan. This is the
  number that matters, and it is where LRU and W-TinyLFU diverge sharply.
- Explain, in your own words, why LRU's post-scan hit rate collapses and
  Caffeine's does not.

**Part D — the concurrency question.** Now run the LRU version from 8 threads.
Report `size()` versus a count obtained by iteration, and whether the run
completes. Then wrap it in `Collections.synchronizedMap` and re-run, measuring
throughput. Answer:
1. Did synchronising fix correctness?
2. What did it cost?
3. At what read volume does that cost stop being acceptable, and how would you
   find that number honestly rather than by guessing?

> Part D's timings are wall-clock and therefore untrustworthy — Topic 77 explains
> why. Report them anyway, and note next to each number that you do not yet trust
> it. Being explicit about the confidence level of your own measurements is a
> senior habit worth building early.

---

## Interview questions

### Q1 — "Implement an LRU cache."

**Mid-level answer:** "I'd use a `HashMap` plus a doubly-linked list. The map
gives O(1) lookup and the list tracks recency — on a get I move the node to the
head, and when I'm over capacity I evict the tail." *(Then writes 40 lines and
gets the pointer updates slightly wrong.)*

**Senior answer:** "Two answers, and which one is right depends on what you are
asking. If you want to see whether I know the data structure: it is a hash map
for O(1) lookup plus a doubly-linked list for O(1) recency updates, and the
subtlety is that the map's value must be the list *node*, not the value, so
unlinking is O(1) rather than a list scan. If you want the code I would actually
write in Java:

```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int max;
    LruCache(int max) { super(16, 0.75f, true); this.max = max; }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) { return size() > max; }
}
```

because `LinkedHashMap` already is a hash map plus a doubly-linked list, the
`true` puts it in access-order mode, and `removeEldestEntry` is a hook the JDK
calls after every insert. That said, I would not ship either one into a shared
service. It is not thread-safe — in access-order mode `get()` mutates the linked
list, so even concurrent reads corrupt it — and it has no TTL, no stats, and no
eviction listener. Production is Caffeine, and I would want to say why: LRU is a
poor policy under scans, and W-TinyLFU's admission filter is the fix."

**What separates them:** offering both answers and naming which question is being
asked; knowing the map must hold the node; and volunteering the thread-safety and
policy limitations without being prompted.

**Follow-up:** "Why is `get()` mutating a problem if we only read?" Because
"read-only" is the defence people reach for, and here it is wrong. If they smile
at this, you have hit the point of the question.

---

### Q2 — "What's the difference between HashMap, LinkedHashMap and TreeMap?"

**Mid-level answer:** "`HashMap` has no order, `LinkedHashMap` keeps insertion
order, `TreeMap` keeps sorted order. `HashMap` and `LinkedHashMap` are O(1),
`TreeMap` is O(log n)."

**Senior answer:** "That is the ordering axis, and it is right, but there are two
more axes that matter more in practice.

**Key identity.** `HashMap` and `LinkedHashMap` decide 'same key' with
`hashCode` then `equals`. `TreeMap` decides it with `compare() == 0` and never
calls `equals` at all. So the same insertions can give different sizes — that is
Topic 14's `BigDecimal` case.

**Iteration cost.** `HashMap` iteration is O(capacity + size) because it walks the
whole table array including empty slots. `LinkedHashMap` follows its linked list,
so it is O(size) exactly. For a large sparse map that is iterated often,
`LinkedHashMap` is genuinely faster to iterate, which surprises people who assume
it is strictly heavier. It is heavier per entry — two extra references — but
cheaper to traverse.

And `LinkedHashMap` has a capability the others do not: access-order mode plus
`removeEldestEntry` makes it a one-class LRU. That is also its unique hazard,
because in that mode a `get` is a structural modification."

**What separates them:** three axes instead of one, and specifically the
iteration-cost point, which almost nobody volunteers and which is directly
useful.

**Follow-up:** "When would you pay for `LinkedHashMap` over `HashMap` if you do
not need eviction?" Any time iteration order is observable — a serialized
response, a log line, a report, a test assertion. Order in the output should be
order in the type, not an accident of the hash function.

---

### Q3 — "Is your `LinkedHashMap` LRU thread-safe? What would you use instead?"

**Mid-level answer:** "No, it's not thread-safe. I'd wrap it in
`Collections.synchronizedMap`."

**Senior answer:** "No, and the reason is worse than people expect. In
access-order mode `get()` calls `afterNodeAccess`, which unlinks the node and
re-links it at the tail — that is several writes to `before`, `after`, `head` and
`tail`. So a workload with *no writes at all* still races. Two threads calling
`get` concurrently can corrupt the list, produce a `size()` that disagrees with
iteration, drop entries, or spin forever in a broken chain. 'We only read it' is
the defence people offer and it is exactly wrong here.

`Collections.synchronizedMap` does make it correct, and I would take it for a
low-traffic cache, but it serialises every read on one lock, which removes most
of the point of caching in a concurrent service, and you still have to
synchronise manually around iteration. `ConcurrentHashMap` is thread-safe but has
no ordering and no eviction, so it does not solve this problem either.

Production answer is Caffeine: built on `ConcurrentHashMap`, with amortised
bookkeeping so readers do not serialise, W-TinyLFU rather than LRU, TTL,
statistics and removal listeners. The `LinkedHashMap` version is the one I use
inside a single request or a single-threaded batch job, where confinement is the
concurrency strategy and adding a dependency would be the worse choice."

**What separates them:** knowing the read path itself is the race, naming the
concrete cost of the synchronized wrapper, and giving a case where the simple
version is still the right call.

**Follow-up:** "What about `ConcurrentLinkedHashMap`?" It was a real library and
its author, Ben Manes, went on to write Guava's cache and then Caffeine. Knowing
that lineage is a small signal that you have actually looked into this.

---

### Q4 — "Why is LRU a bad eviction policy? What's better?"

**Mid-level answer:** "LRU is usually fine. Maybe LFU is better for some
workloads."

**Senior answer:** "LRU's weakness is scan resistance. It assumes recency
predicts future use, so a single pass over a large key space — a reporting query,
a batch job, a crawler, a cache-warming script someone wrote — walks through the
cache and evicts the entire genuinely-hot working set, because every one of those
one-time keys looks maximally recent at the moment it arrives. Hit rate goes to
near zero and stays there until the hot set is faulted back in one miss at a
time. In a service that is a latency cliff with no obvious cause, and the trigger
is often something on a cron that nobody associates with the API.

Plain LFU has the opposite problem: it needs unbounded counters and it never
forgets, so a key that was hot last month blocks a key that is hot today.

W-TinyLFU, which is what Caffeine implements, combines them. It keeps a compact
approximate frequency sketch — a count-min sketch with periodic aging — and uses
it as an **admission** filter: a new entry is only admitted if its estimated
frequency beats the entry it would evict. So scan traffic is rejected at the door
rather than being admitted and then displacing hot keys. There is a small LRU
window in front for genuine recency, hence 'W' for windowed.

If I were choosing without measuring, I would take Caffeine's default. If it
mattered, I would replay a real access trace against both policies and compare
hit rates — which is the exercise, not a debate."

**What separates them:** naming scan resistance as the specific failure, knowing
W-TinyLFU is an *admission* policy rather than an eviction policy, and ending on
"replay a real trace" rather than an opinion.

**Follow-up:** "Where does the cache-warming script fit into this?" It is the
same failure with friendlier intentions — warming a cache with a full scan under
LRU can leave it in a worse state than cold.

---

### Q5 — "What does `removeEldestEntry` return, and what should you not do in it?"

**Mid-level answer:** "It returns true if the eldest entry should be removed.
Usually `size() > maxEntries`."

**Senior answer:** "It is a predicate, called from `afterNodeInsertion` after
every insert, and `eldest` is the head of the linked list — the oldest by
insertion, or the least recently used in access-order mode. Return `true` and
the JDK removes it. Three things worth knowing beyond that.

First, the comparison is `size() > max`, not `>=`, because `size()` has already
been incremented by the insert. `>=` gives you a cache of `max - 1`, which is a
bug nobody notices for years.

Second, it removes exactly **one** entry per insert. So lowering the bound at
runtime does not shrink the map — it shrinks by one per subsequent put. If you
need it to shrink now, you evict explicitly.

Third, and this is the one I would flag in review: it must not do work. It runs
inside `putVal`, on the hot path. I have seen audit logging and secondary-index
updates in there, which makes every insert as slow as the slowest thing it calls,
and if the secondary structure has its own lock you have created a lock order
nobody designed. If you genuinely need an eviction callback, that is
`removalListener` in Caffeine, which runs off the calling thread by default."

**What separates them:** the `>` versus `>=` off-by-one, the one-per-insert
behaviour, and treating "no side effects in a predicate" as a reviewable rule
with a concrete consequence.

**Follow-up:** "Could you implement a TTL with `removeEldestEntry`?" Only badly —
it can inspect the eldest entry's timestamp, but it is only called on insert, so
a cache with no writes never expires anything, and it can only ever remove the
single eldest. Recognising why that does not work is the answer.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `LinkedHashMap` extends `HashMap` and adds two reference fields per entry.
   Estimate the memory overhead per entry, then say at what map size that
   overhead would change your design.

2. `HashMap` declares three empty `afterNode*` methods that it never uses itself.
   What does that tell you about how the JDK authors thought about extension, and
   what is the cost of that design choice?

3. `containsKey` does not count as an access but `getOrDefault` does — on a hit.
   Construct a defensible rationale for that distinction, then argue it is
   inconsistent.

4. In access-order mode, `get()` performs several writes to shared state. Java
   could have made these updates lock-free, or lazy, or batched. Pick one of
   those and describe what it would change about the class's guarantees.

5. `removeEldestEntry` removes at most one entry per insert. Give a workload
   where that is exactly right and one where it is a problem, and say how you
   would detect the second case.

6. LRU's scan-resistance failure and Topic 12's hash-collision DoS are both
   "adversarial input degrades a data structure". Compare them: which is easier
   to trigger accidentally, and which is worse when it happens?

7. Node's `lru-cache` package has millions of weekly downloads; Java has this in
   the standard library and most Java developers still reach for Caffeine. What
   does that tell you about where the actual difficulty in caching lies?

---

## Quick reference card

### The six-line LRU

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxEntries;
    public LruCache(int maxEntries) {
        super(16, 0.75f, true);                 // true = ACCESS order
        this.maxEntries = maxEntries;
    }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxEntries;             // '>' not '>=' — size() is post-insert
    }
}
```

Not thread-safe. No TTL. No stats. Ship Caffeine.

### What counts as an access (accessOrder = true)

| Counts | Does not count |
|---|---|
| `get` (hit) | `containsKey` / `containsValue` |
| `getOrDefault` (hit) | iteration of `keySet` / `values` / `entrySet` |
| `put` (new or existing) | `forEach` |
| `putIfAbsent`, `replace` | `size`, `isEmpty` |
| `merge`, `compute`, `computeIfAbsent` (hit) | `getOrDefault` (miss) |

### Java 21 `SequencedMap` on LinkedHashMap

```java
map.firstEntry();        // eldest — the LRU victim
map.lastEntry();         // most recently used
map.pollFirstEntry();    // evict manually
map.reversed();          // live reversed view
map.putFirst(k, v);      // may throw UnsupportedOperationException in access-order mode
```
Java 17 fallback for `firstEntry`: `map.entrySet().iterator().next()`.

### Cost comparison

| | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---|---|---|---|
| `get` / `put` | O(1) | O(1) | O(log n) |
| iteration | O(capacity + size) | **O(size)** | O(size) |
| memory per entry | baseline | +2 references | +3 references, +colour |
| order | none, unspecified | insertion or access | sorted |
| key identity | `hashCode` + `equals` | `hashCode` + `equals` | `compare() == 0` |
| null key | one allowed | one allowed | none |

### Caffeine, the parts you will use

```java
Cache<String, Optional<Product>> cache = Caffeine.newBuilder()
    .maximumSize(50_000)                       // or .maximumWeight(...) + .weigher(...)
    .expireAfterWrite(Duration.ofMinutes(10))
    .refreshAfterWrite(Duration.ofMinutes(2))  // requires a LoadingCache
    .recordStats()
    .removalListener((k, v, cause) -> { })
    .build();

cache.get(key, k -> load(k));   // loader runs once per key, even under concurrent misses
cache.stats().hitRate();
```

### Gotchas checklist

- [ ] `accessOrder=true` or it is a FIFO cache wearing an LRU label.
- [ ] `size() > max`, not `>=`.
- [ ] `containsKey` is not an access. Do a single `get`.
- [ ] In access-order mode, `get()` mutates. Concurrent reads are unsafe.
- [ ] `removeEldestEntry` is a predicate. No I/O, no locks, no side effects.
- [ ] It removes one entry per insert. Lowering the bound does not shrink it now.
- [ ] Two extra references per entry. Do not use it as a default `Map`.
- [ ] An unbounded cache is a leak (Topic 79), not a cache.
- [ ] `null` values make "miss" and "cached null" indistinguishable. Cache an `Optional`.
- [ ] No stats means you do not know whether the cache works. Record them.

---

## When would I use this at work?

**1. Any map whose iteration order is visible to a human or a machine.**
A JSON response body, a CSV export, a log line listing failed items, a diff in a
test failure. If someone will read the order, it should be
`LinkedHashMap` — not because `HashMap` is wrong, but because "the order happens
to be stable today" is not a property you want load-bearing. This costs two
references per entry and removes an entire category of flaky test.

**2. A per-request or per-job memo.**
You are processing 50,000 order lines and each one needs a product lookup, with
heavy repetition. A `LruCache<>(2000)` confined to that one thread is correct,
zero-dependency, and obviously bounded. Reaching for Caffeine here would be
adding a concurrent data structure to a single-threaded problem. Knowing when
*not* to use the production tool is part of using it well.

**3. The design conversation about a shared cache.**
Someone proposes adding a `static Map` cache to a Spring singleton. You now have
five specific questions ready: what bounds it, what expires it, is it thread-safe,
what is the hit rate, and what happens on the second pod. Each of those maps to a
concrete defect from this topic. That conversation, held before the code merges,
is worth more than any amount of later debugging.

---

## Connected topics

**Prerequisites:**
- **10 — Collections Framework:** `Map` contract, fail-fast iteration and
  `modCount` — which is what makes Proof 4's `ConcurrentModificationException`
  appear.
- **11 — List implementations:** the doubly-linked list threaded through the
  entries is exactly the structure you studied there, including its cache-locality
  weakness.
- **12 — HashMap internals:** required. `LinkedHashMap` **is** a `HashMap` plus a
  list; every word about spreading, buckets, resize and treeification still
  applies.
- **13 — equals/hashCode:** unchanged. Key identity is still `hashCode` + `equals`.

**This unlocks:**
- **16 — Sets and EnumSet:** `LinkedHashSet` is this class with dummy values, and
  is the right answer for "a set that iterates predictably".
- **17 — Immutability:** an immutable snapshot of a cache, and why safe
  publication matters for anything shared between threads.
- **26 — Optional:** caching `Optional.empty()` to make negative lookups
  cacheable, as in Example 2.
- **38 — Bean scopes:** why a field on a Spring singleton is shared across every
  concurrent request, which is what turns Trap 3 from theory into an incident.
- **51 — Hibernate caching:** the same problems one layer down — L1, L2, and why
  a cache is only safe if every writer goes through it.
- **79 — Memory leaks:** an unbounded cache is *the* canonical Java leak. This
  topic is the fix; that topic is the diagnosis.
- **92 — ConcurrentHashMap:** what Caffeine is built on, and why a concurrent map
  alone is still not a cache.
- **118 — Metrics:** `recordStats()` plus `CaffeineCacheMetrics` — an unmeasured
  cache is an assumption.

---

*Java baseline 21. `LinkedHashMap` itself has been unchanged in behaviour since
Java 8. The `SequencedMap` interface — `firstEntry`, `lastEntry`,
`pollFirstEntry`, `putFirst`, `putLast`, `reversed` — is new in Java 21, so it is
available on this baseline but not on a Java 17 codebase; the fallback there is
`entrySet().iterator().next()`. Nothing in this topic differs between Java 21 and
Java 25. The one claim I could not verify without a JVM is whether `putFirst` /
`putLast` throw in access-order mode; Proof 5 settles it in three lines.*
