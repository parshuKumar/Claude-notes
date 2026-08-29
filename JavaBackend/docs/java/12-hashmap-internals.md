# 12 — HashMap Internals: Hashing, Spreading, Buckets, Resize, Treeification

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Picture a cloakroom with 16 numbered pegs.

Someone hands you a coat. You need a rule that turns "this coat" into "peg
number 7", and the rule must give the *same* answer every single time, or you
will never find the coat again.

So:

1. The coat has a label with a long number on it. That is `hashCode()`.
2. You do not have a million pegs, you have 16. So you take the last few digits
   of that number and use those. That is the **bucket index**.
3. You hang the coat on that peg.

Two problems appear immediately, and the whole topic is about them.

**Problem one: two coats want the same peg.** Fine — you hang both on peg 7, one
behind the other. Now finding a coat means going to peg 7 and looking through
everything hanging there. If a hundred coats end up on peg 7, "instant lookup"
has quietly become "read a hundred labels".

**Problem two: the last few digits are a bad summary.** If every coat label ends
in `...000`, then every coat goes to peg 0 and the other fifteen pegs are empty.
The label was fine. Your *rule for shortening it* threw away the useful part.

Java's `HashMap` is the cloakroom, plus two repairs for those two problems:

- It **stirs the label's high digits down into the low digits** before shortening
  it, so a label whose interesting part is at the top does not collapse to peg 0.
  That is the **spread**.
- When one peg gets crowded past a threshold, it stops being a queue and becomes
  a **sorted tree**, so even a hundred coats on one peg costs about seven checks
  instead of a hundred.

And when the whole cloakroom is 75% full, it doubles the number of pegs and
re-hangs everything. That is **resize**.

That is `HashMap`. Everything below is detail.

---

## The bridge from what you know

### What transfers

You use `Map` and `Object` in TypeScript constantly:

```ts
const stockBySku = new Map<string, number>();
stockBySku.set("SKU-4471", 12);
stockBySku.get("SKU-4471");     // 12
```

```java
Map<String, Integer> stockBySku = new HashMap<>();
stockBySku.put("SKU-4471", 12);
stockBySku.get("SKU-4471");     // 12  (boxed — see Topic 01)
```

The *shape* is identical. V8 genuinely does implement `Map` with a hash table,
and `Object` with either hidden classes (for stable shapes) or a dictionary-mode
hash table (once you delete keys or add them dynamically). So the underlying
idea — "hash the key, land in a slot" — really is the same idea.

**Verdict: PARTIAL analogue.** Here is exactly what you must unlearn.

### What does not transfer

| TypeScript / V8 | Java `HashMap` | Verdict |
|---|---|---|
| `Map` key equality is **SameValueZero** — fixed, built into the engine | Key equality is `hashCode()` + `equals()`, **written by you** | **NO ANALOGUE** — you now own the correctness of lookup |
| Two structurally identical objects are two different keys, always | Two structurally equal objects are the *same* key if you say so | **NO ANALOGUE** |
| Load factor, capacity, resize are invisible and untunable | `new HashMap<>(capacity, loadFactor)` — you can set both, and both show up in profiles | **PARTIAL** |
| No such thing as a "bad hash function" — you cannot supply one | A bad `hashCode()` turns O(1) into O(n) and is a denial-of-service vector | **NO ANALOGUE** |
| `Map` iteration order is **guaranteed** insertion order by spec | `HashMap` iteration order is **unspecified** and changes when the map resizes | **NO ANALOGUE — and this one will bite you** |
| No tree fallback; V8 uses open addressing / ordered backing store | Collision chains become red-black trees at a threshold | **NO ANALOGUE** |

That fifth row deserves emphasis, because it is the one you will get wrong from
muscle memory. In JavaScript, `Map` iteration order is a *specified guarantee*:
insertion order, always. In Java, `HashMap` iteration order is an accident of the
table layout. Insert one more entry, trigger a resize, and the order changes.
Code that depends on it is broken code that happens to pass today.

If you want insertion order in Java, that is `LinkedHashMap` — Topic 15.

---

## What is this?

A `HashMap` is an **array of buckets**. Each array slot holds either nothing, a
short **linked list** of entries, or (when crowded) a **red-black tree** of
entries.

Four numbers define its behaviour:

| Name | Value | What it means |
|---|---|---|
| Default capacity | 16 | Number of array slots on first use |
| Load factor | 0.75 | Resize when `size > capacity * 0.75` |
| `TREEIFY_THRESHOLD` | 8 | A crowded bucket becomes a tree |
| `UNTREEIFY_THRESHOLD` | 6 | A thinned tree goes back to a list |

Plus one that people forget:

| `MIN_TREEIFY_CAPACITY` | 64 | Below this table size, **resize instead of treeifying** |

Every entry stores four things:

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;      // the SPREAD hash, cached at insertion time
    final K key;
    V value;
    Node<K,V> next;      // next entry in this bucket
}
```

That `final int hash` field is important and easy to miss. The spread hash is
computed **once, at insertion**, and stored. It is never recomputed — not on
lookup, not on resize. That single design choice is what makes resize cheap and
what makes a mutable key permanently unreachable (Topic 13).

---

## Why does it matter?

Three reasons, in increasing order of career impact.

**1. It is the most-asked senior Java interview topic in Phase 1.**
Not "what is a HashMap" — everyone answers that. The questions are *why a power
of two*, *why the XOR*, *why eight*, *what resize does to iteration order*. Those
questions exist specifically to separate people who have read the source from
people who have read a blog post.

**2. Your `hashCode()` is a performance contract, and you can violate it.**
A `HashMap` gives you O(1) *if* your keys spread. Give it keys that all collide
and every operation becomes O(n) — or O(log n) once treeification saves you.
You cannot cause this in JavaScript. You can trivially cause it in Java, and
attackers can cause it deliberately on any endpoint that puts user-controlled
strings into a map.

**3. Sizing a map is a real capacity decision.**
`new HashMap<>()` starting at 16 and growing to hold 10 million entries performs
about twenty resizes, each one allocating a new array and re-linking every entry.
Knowing that turns a mysterious startup pause into a one-line fix.

---

## Syntax breakdown

Only genuinely new constructs are covered here.

### Bitwise operators you will see in the source

```java
h >>> 16        // unsigned right shift: shift right, fill with ZEROS
h >> 16         // signed right shift:   shift right, fill with the SIGN bit
h ^ x           // XOR: 1 where the bits differ
(n - 1) & hash  // bitwise AND: keeps only the bits set in both
```

| Bit of syntax | What it means | TypeScript equivalent |
|---|---|---|
| `>>>` | Unsigned right shift. Java has no unsigned integers, so `>>>` is how you say "treat this as a bit pattern, not a number". | `>>>` — same operator, same meaning |
| `>>` | Signed right shift. Negative numbers stay negative. | `>>` |
| `^` | XOR | `^` |
| `&` | AND | `&` |
| `0x7fffffff` | Hex literal. `0x` prefix. This particular value is "all bits except the sign bit". | `0x7fffffff` |

These are the same operators you already know from JS, with one difference worth
naming: in JavaScript, bitwise operators coerce to 32-bit ints and back to
float64. In Java, `int` **is** 32 bits, so there is no coercion happening — the
operator is doing exactly what it says on the hardware.

### The diamond and the factory

```java
Map<String, Integer> m = new HashMap<>();              // diamond: infers <String, Integer>
Map<String, Integer> m2 = new HashMap<>(32);           // initial capacity hint
Map<String, Integer> m3 = new HashMap<>(32, 0.5f);     // capacity + load factor
Map<String, Integer> m4 = HashMap.newHashMap(1000);    // sized for 1000 MAPPINGS
```

`HashMap.newHashMap(int)` is the one you want and almost nobody uses. It exists
since Java 19, so it is available on your 21 baseline. The difference matters:

- `new HashMap<>(1000)` means "give me a table of at least 1000 slots". You get
  1024 slots, a threshold of `1024 * 0.75 = 768`, and it **still resizes** before
  you finish inserting 1000 entries.
- `HashMap.newHashMap(1000)` means "I will insert 1000 entries". It works the
  arithmetic backwards for you and picks a capacity that will not resize.

`[LEGACY — still asked]` Before Java 19 the idiom was
`new HashMap<>((int) (expected / 0.75f) + 1)`, or Guava's
`Maps.newHashMapWithExpectedSize(n)`. Interviewers still ask why
`new HashMap<>(1000)` is wrong.

---

## The `get` path, told as a story

This is the section to reread. Everything else is commentary on it.

You call:

```java
Integer stock = stockBySku.get("SKU-4471");
```

Here is every step, in order.

### Step 1 — ask the key for its hash code

```java
int h = "SKU-4471".hashCode();
```

`String.hashCode()` is specified in the Javadoc as
`s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]`. It is a fixed, published
algorithm. It is **not** randomised per JVM run. Remember that; it comes back
under "DoS vector".

You now have a 32-bit `int`. It could be any of about 4.3 billion values,
including negative ones.

### Step 2 — spread it

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

Read it slowly:

- If the key is `null`, the hash is `0`. That is why `HashMap` allows exactly one
  `null` key and why it always lands in bucket 0.
- Otherwise: take the hash code, shift it right 16 places (so the top 16 bits
  move into the bottom 16 positions), and XOR that with the original.

The result: **every one of the low 16 bits now carries information from a high
bit.** The high bits have been "folded down".

Why this is necessary is Step 3.

### Step 3 — turn the hash into a bucket index

```java
int index = (table.length - 1) & hash;
```

The table length is always a power of two — 16, 32, 64, 1024. So
`table.length - 1` is always a run of 1-bits: 15 is `0000...1111`, 1023 is
`0000...1111111111`.

ANDing with that keeps only the **low bits** of the hash and discards everything
else. With a table of 16, only the bottom 4 bits of your hash code have any
effect on where the entry lands. The other 28 bits are thrown away entirely.

That is the whole reason Step 2 exists. Without the spread, a key type whose
hash codes differ only in the high bits — which is extremely common; think an ID
multiplied by a large constant, or a `Long` key holding a timestamp — would map
every single entry to the same bucket. The spread costs one shift and one XOR
and converts that catastrophe into a normal distribution.

Note what the spread is *not*: it is not a good hash function, and it is not
trying to be. It is a cheap fix for one specific failure mode. A genuinely bad
`hashCode()` is still bad after spreading.

### Step 4 — look in the bucket

Three cases.

**Bucket is empty** — `table[index] == null`. Return `null`. Done. This is the
fast path and it is one array read.

**Bucket holds a short list.** Walk it. For each node:

```java
if (node.hash == h && (node.key == key || key.equals(node.key)))
    return node;
```

Three checks, deliberately ordered cheapest-first:

1. `node.hash == h` — an `int` comparison. If the stored spread hashes differ,
   the keys cannot be equal, and we skip `equals()` entirely. This is why the
   hash is cached in the node.
2. `node.key == key` — reference identity. Free, and true surprisingly often
   (interned strings, enum constants, cached boxes from Topic 01).
3. `key.equals(node.key)` — the expensive one, only reached when the first two
   do not settle it.

**Bucket holds a tree.** Walk the red-black tree, comparing by hash first, then
— if the keys implement `Comparable` — by `compareTo`, and otherwise falling back
to a tie-break on identity hash codes. O(log n) instead of O(n).

### Step 5 — return the value

`getNode` returns the `Node`; `get` returns `node.value`, or `null` if `getNode`
returned `null`.

Which is why `map.get(k) == null` is ambiguous: the key might be absent, or the
key might be mapped to `null`. `map.containsKey(k)` is the disambiguator. In
TypeScript you have the same ambiguity with `map.get(k) === undefined` and the
same fix with `map.has(k)`, so this one transfers cleanly.

---

## Why a power of two, precisely

The obvious way to map a hash to a bucket is modulo:

```java
int index = Math.abs(hash) % table.length;
```

That works for any table length and gives a good distribution even for mediocre
hash functions. `HashMap` does not do it. Two reasons:

**1. `&` is far cheaper than `%`.** Integer division and remainder are among the
slowest integer instructions on a modern CPU — many cycles, and not pipelined
the way an AND is. A hash map does this on *every* `get`, `put`, `remove` and
`containsKey`. Making the hot path one AND instead of one division is a real
win at the volumes a hash map operates at.

**2. Resize becomes almost free.** This is the deeper reason, and it is Step 3
of `resize()`, below.

The cost of choosing `&` is that only the low bits survive — which is precisely
the weakness the spread was invented to patch. So: power-of-two table forces
low-bits-only indexing, which forces the XOR spread. The two decisions are one
decision.

`[LEGACY — still asked]` `Hashtable` (the pre-1.2 class) uses `%` with a prime
table size. That is the classical textbook design. `HashMap` deliberately went
the other way and paid for it with the spread. Being able to argue both sides is
a good interview answer.

---

## Resize, step by step

```java
if (++size > threshold)      // threshold = capacity * loadFactor
    resize();
```

With default settings the first resize happens on the 13th insertion
(`16 * 0.75 = 12`), the second on the 25th, and so on.

`resize()` does this:

1. **Double the capacity and double the threshold.** 16 → 32, 12 → 24.
2. **Allocate a new array** of the new size. The old array is now garbage.
3. **Redistribute every entry**, and here is the trick.

For a bucket at index `j` in the old table of size `oldCap`, every entry in it
goes to exactly one of two places in the new table: **`j`** or **`j + oldCap`**.
Nothing else is possible.

Why? Because doubling the table adds exactly one bit to the mask. Index was
`hash & (oldCap - 1)`; it is now `hash & (2*oldCap - 1)`. The only difference is
one additional bit, and that bit is `oldCap`. So:

```java
if ((entry.hash & oldCap) == 0)  // that new bit is 0
    // stays at index j
else
    // moves to index j + oldCap
```

`HashMap` walks each old bucket once, splitting it into a "lo" list and a "hi"
list, then attaches lo at `j` and hi at `j + oldCap`. **No hash is recomputed.
No `equals()` is called. No key is touched.** That is what the cached
`final int hash` in each node buys, and it is only possible because the table
size is a power of two.

Relative order within each list is preserved. That is a Java 8 change: Java 7
reversed the list during transfer, which under concurrent use could form a cycle
and spin a CPU at 100% forever. Java 8 removed that specific failure — but
`HashMap` is still **not thread-safe**, and concurrent writes still produce lost
updates, a wrong `size()`, and entries that vanish. Do not repeat "Java 8 fixed
the HashMap infinite loop" as though it made the class safe. Topic 92 is
`ConcurrentHashMap`.

**Consequence you must remember: resize changes iteration order.** An entry that
iterated third may now iterate thirtieth. If your test asserts on the string
produced by iterating a `HashMap`, it will pass until someone adds a thirteenth
entry.

---

## Treeification, step by step

Java 8 added this. Before it, a bucket was always a linked list, so N colliding
keys meant O(N) lookups — and a remote attacker who could choose your keys could
make N large on purpose.

The rule:

1. A bucket's list grows past `TREEIFY_THRESHOLD` (8).
2. `treeifyBin()` is called. **First it checks the table length.** If
   `table.length < MIN_TREEIFY_CAPACITY` (64), it calls `resize()` instead of
   treeifying, on the theory that a crowded bucket in a tiny table is a sizing
   problem, not a hash-quality problem.
3. Only if the table is already at least 64 slots does the bucket convert to a
   red-black tree.

Ordering inside the tree, in priority order:

- by the stored spread `hash`;
- if hashes tie and the key type implements `Comparable`, by `compareTo`;
- if it still ties, by a deterministic tie-break using class names and
  `System.identityHashCode`.

That last fallback is why treeification works even for keys that are not
`Comparable`: the tree only needs *a* total order, not a meaningful one.

**Untreeify at 6, not 8.** Removals shrink the tree; when it drops to
`UNTREEIFY_THRESHOLD` (6) or fewer, it converts back to a list. (A tree also
untreeifies during resize if its half lands at 6 or fewer nodes.)

The gap between 8 and 6 is **hysteresis**. If both thresholds were 8, a workload
sitting exactly at the boundary would convert list → tree → list → tree on
alternating put/remove, paying the conversion cost every operation. A two-node
gap means you must actually move away from the boundary before paying again.
This is the same reason a thermostat does not switch at exactly the set point,
and it is the answer interviewers are fishing for.

**Why 8 specifically?** The source has a comment about it, and the answer is
statistical. With load factor 0.75 and a decent hash function, bucket occupancy
follows a Poisson distribution with lambda = 0.5. The probability of a bucket
holding 8 entries is roughly 6 in 100 million. So under normal conditions
treeification effectively never happens — which is the point. Trees cost more
memory per node (a `TreeNode` carries parent/left/right/prev pointers on top of
everything a `Node` has). You do not want to pay that in the common case. The
threshold is set where "this is not chance, someone is attacking me or your
`hashCode` is broken" becomes the likely explanation.

> **Honest note on the exact trigger count.** The constant is 8 and "eight" is
> the answer an interviewer wants. But `putVal`'s loop counter does not count the
> head node — the source comment literally reads `// -1 for 1st` — so whether the
> conversion fires as the bucket reaches its 8th or 9th node depends on how you
> count. Do not take my word for it and do not take a blog's. Read `putVal`
> yourself; the command is in Hands-on proof, Proof 1.

---

## Example 1 — minimal

```java
public class SpreadDemo {

    // Reimplementation of HashMap.hash(). Same two lines, made visible.
    static int spread(Object key) {
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }

    static int indexIn(int tableSize, int spreadHash) {
        return (tableSize - 1) & spreadHash;
    }

    public static void main(String[] args) {
        // Keys whose hash codes differ ONLY in the high 16 bits.
        int[] rawHashes = { 0x0000_1234, 0x0001_1234, 0x0002_1234, 0x0003_1234 };

        System.out.println("table size 16, NO spread:");
        for (int h : rawHashes) {
            System.out.printf("  hash=%08x -> bucket %d%n", h, 15 & h);
        }

        System.out.println("table size 16, WITH spread:");
        for (int h : rawHashes) {
            int s = h ^ (h >>> 16);
            System.out.printf("  hash=%08x -> spread=%08x -> bucket %d%n", h, s, 15 & s);
        }
    }
}
```

Run it. The first block puts all four keys in the same bucket, because the low
four bits are identical in all four hash codes. The second block spreads them.
Two operations, one catastrophe averted.

---

## Example 2 — production scenario

`orderflow` has a nightly reconciliation job. It loads every payment from the
last 24 hours into a map keyed by a composite of gateway and gateway reference,
then walks the ledger looking for unmatched entries.

### The version that ships and gets slower every month

```java
public final class PaymentKey {
    private final String gateway;        // "STRIPE", "ADYEN", "WALLET"
    private final long gatewayRef;

    public PaymentKey(String gateway, long gatewayRef) {
        this.gateway = gateway;
        this.gatewayRef = gatewayRef;
    }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PaymentKey other)) return false;
        return gatewayRef == other.gatewayRef && gateway.equals(other.gateway);
    }

    @Override public int hashCode() {
        return gateway.hashCode();          // <-- the defect
    }
}
```

```java
public class Reconciler {
    public Map<PaymentKey, Payment> index(List<Payment> payments) {
        Map<PaymentKey, Payment> byKey = new HashMap<>();       // <-- second defect
        for (Payment p : payments) {
            byKey.put(new PaymentKey(p.gateway(), p.gatewayRef()), p);
        }
        return byKey;
    }
}
```

Two independent defects.

**Defect one: `hashCode()` uses only the gateway.** It is *legal* — equal objects
do have equal hash codes, so the contract in Topic 13 is satisfied. It is
catastrophic for performance. There are three gateways, so every one of 400,000
payments lands in one of **three buckets**. Treeification saves you from O(n) and
gives O(log n), which is the only reason this job finishes at all. Lookups that
should be one array read become a tree descent of depth ~17.

**Defect two: no sizing.** Starting at 16 and growing to hold 400,000 entries
means the map resizes roughly fifteen times. Each resize allocates a new array
and re-links every entry present so far. And because the entries all land in
three buckets, each resize also has to split three enormous trees.

The observable symptom is a batch job whose runtime grows superlinearly with
payment volume, with CPU time concentrated in `HashMap.getNode`,
`HashMap.putVal`, `HashMap.resize` and `TreeNode` methods in a profile. Nothing
throws. Nothing logs. It is just slow, and it gets slower every month as volume
grows.

### The corrected version

```java
public record PaymentKey(String gateway, long gatewayRef) { }
```

That is the whole fix for defect one. A `record` generates `equals` and
`hashCode` from **all** components (Topic 27), so both fields participate.

```java
public class Reconciler {
    public Map<PaymentKey, Payment> index(List<Payment> payments) {
        // Sized for the actual number of mappings, so zero resizes.
        Map<PaymentKey, Payment> byKey = HashMap.newHashMap(payments.size());
        for (Payment p : payments) {
            byKey.put(new PaymentKey(p.gateway(), p.gatewayRef()), p);
        }
        return byKey;
    }
}
```

Three changes and what each buys:

1. **`record`** — `hashCode` now mixes both components, so 400,000 keys spread
   across the table instead of piling into three buckets. Lookups become one
   array read plus usually one `equals`.
2. **`HashMap.newHashMap(size)`** — the table is allocated once at the right
   size. No resizes, no re-linking, no array churn during the job.
3. **`record` again** — you also stop being able to forget `hashCode` when you
   later add a `currency` field to the key. The compiler regenerates both.

> Sizing is only worth doing when you actually know the count. Do not scatter
> capacity hints through a codebase on instinct — an over-large table wastes
> memory and hurts iteration, which walks every slot including empty ones.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a legal but useless `hashCode()`

**Wrong:**
```java
@Override public int hashCode() { return 42; }          // or: return gateway.hashCode();
```

**Exact symptom:** no exception, ever. A `HashMap`-heavy code path that is
mysteriously CPU-bound. In a flame graph (Topic 78) the time sits in
`HashMap.getNode`, `HashMap.putVal`, and `HashMap$TreeNode.*`. Latency scales
with map size instead of staying flat. On small test data it is invisible.

**Root cause:** all keys spread to the same bucket. The map has degenerated into
a single tree (or, below 64 slots, a single linked list).

**Fix:** include every field that `equals` uses. Use a `record`, or
`Objects.hash(a, b, c)`, or the hand-rolled `31 * result + field` form for hot
paths.

> `Objects.hash(...)` is varargs — it allocates an `Object[]` on every call and
> boxes any primitives you pass. That is Topic 01's cost, in a method that runs
> on every lookup. Fine for the 99% case; hand-roll it when a profiler says so.

---

### Trap 2 — `new HashMap<>(expectedEntries)`

**Wrong:**
```java
Map<Long, Order> orders = new HashMap<>(100_000);
```

**Exact symptom:** the map still resizes. You see it as an unexplained
allocation spike and a longer-than-expected startup or batch phase; with GC
logging on (`-Xlog:gc`) you get young collections during what should be a pure
population loop.

**Root cause:** the constructor argument is **table capacity**, not entry count.
100,000 rounds up to a table of 131,072 slots, threshold
`131072 * 0.75 = 98,304` — which is *less than* 100,000. It resizes on the
98,305th insert.

**Fix:**
```java
Map<Long, Order> orders = HashMap.newHashMap(100_000);   // Java 19+, so fine on 21
```

---

### Trap 3 — depending on `HashMap` iteration order

**Wrong:**
```java
Map<String, Integer> lines = new HashMap<>();
// ... populate from an order ...
String csv = String.join(",", lines.keySet());
assertEquals("SKU-1,SKU-2,SKU-3", csv);
```

**Exact symptom:** a test that passes on your machine and fails in CI, or passes
for months and then fails the day someone adds a thirteenth SKU to the fixture.
The failure message shows the same elements in a different order.

**Root cause:** `HashMap` iteration walks the table array in slot order. Slot
assignment depends on the spread hash and the current table size — so it changes
on resize. Nothing about it is specified. Your JavaScript instinct that "Map
iterates in insertion order" is a *guaranteed* fact in JS and a *false* one here.

**Fix:** if order is part of the requirement, say so in the type.
```java
Map<String, Integer> lines = new LinkedHashMap<>();   // insertion order — Topic 15
Map<String, Integer> lines = new TreeMap<>();         // sorted order    — Topic 14
```
If order is not part of the requirement, sort in the assertion instead of
depending on the map.

---

### Trap 4 — a mutable key

**Wrong:**
```java
Map<Cart, Order> draftOrders = new HashMap<>();
Cart cart = new Cart(customerId);
draftOrders.put(cart, order);
cart.addLine(sku, 2);                 // hashCode now different
draftOrders.get(cart);                // null
```

**Exact symptom:** silent data loss. `get` returns `null` for a key you are
holding a reference to. `containsKey` is `false`. `remove` does nothing. And
`size()` still counts the entry, so the map grows forever.

**Root cause:** the spread hash was cached in the `Node` at insertion time and is
never recomputed. After mutation, lookup computes a *new* hash, lands in a
*different* bucket, and finds nothing. The entry is still in the old bucket,
reachable from the map, and now unreachable through the API.

**Fix:** never use a mutable object as a key. This trap is important enough to
be its own topic — it is Topic 13, with a full failure drill.

---

### Trap 5 — user-controlled strings as keys (the DoS vector)

**Wrong:**
```java
@PostMapping("/orders")
public void create(@RequestBody Map<String, Object> attributes) {
    // Jackson has already built a HashMap from arbitrary client-supplied keys
}
```

**Exact symptom:** one HTTP request with a modest body pins a CPU core for
seconds. Under a handful of such requests, p99 latency across *every* endpoint
collapses because the request threads are all busy. A CPU profile shows time in
`HashMap.putVal` and `String.equals`.

**Root cause:** `String.hashCode()` is a published, deterministic, non-randomised
algorithm. Anyone can compute strings that collide — `Aa` and `BB` collide, and
you can generate thousands by concatenation. Java 8's treeification bounds the
damage at O(log n) instead of O(n), which is a genuine mitigation, but building
the tree is not free and the attack is not fully neutralised.

> Some languages (Perl, Python, Ruby) randomise string hashing per process
> specifically to kill this attack. Java does not, because `String.hashCode()`'s
> value is specified in the Javadoc and code in the wild depends on it. Java 7u6
> shipped an opt-in "alternative hashing" mechanism and it was removed again in
> Java 8 when treeification replaced it. `[LEGACY — still asked]`

**Fix:** this is not a `HashMap` fix, it is an input-validation fix. Bound the
number of keys you will accept before you build a map from untrusted input, and
prefer a fixed DTO with named fields over `Map<String, Object>`. Jackson can be
configured with limits; a DTO makes the question moot.

---

## Hands-on proof

Everything here is a command **you** run. I do not have a JVM and will not print
output and call it real. What follows is exactly what to look for and how to read
each possible result.

### Setup

```bash
mkdir -p ~/java-lab/12 && cd ~/java-lab/12
java --version          # expect 21 or 25
echo $JAVA_HOME         # needed for the source read below
```

### Proof 1 — read the actual source (do this first)

Everything I have told you is verifiable in about 200 lines of one file. Your JDK
ships it.

```bash
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/HashMap.java' -d ~/java-lab/12/src
less ~/java-lab/12/src/java.base/java/util/HashMap.java
```

If `src.zip` is missing, your JDK is a JRE-style build; get a full JDK, or read
it online in the OpenJDK repository at
`src/java.base/share/classes/java/util/HashMap.java`.

Read these five things, in this order. In `less`, press `/` then type the search
term.

| Search for | What you are confirming |
|---|---|
| `DEFAULT_INITIAL_CAPACITY` | the four constants, and `MIN_TREEIFY_CAPACITY = 64` |
| `static final int hash(` | the spread is exactly two operations, and `null` maps to 0 |
| `final Node<K,V> getNode` | the cheap-to-expensive check order in Step 4 |
| `putVal` | the `binCount` loop and the `// -1 for 1st` comment — settle the 8-vs-9 question yourself |
| `final Node<K,V>[] resize` | `loHead` / `hiHead` and `(e.hash & oldCap) == 0` |

| What you see | What it means |
|---|---|
| `(h = key.hashCode()) ^ (h >>> 16)` | Confirmed: the spread is one shift and one XOR, nothing more |
| `if (binCount >= TREEIFY_THRESHOLD - 1)` with `binCount` starting at 0 on the head's `next` | The counter skips the head node. Now decide for yourself whether that means 8 or 9, and be able to defend it |
| `if (tab == null \|\| (n = tab.length) < MIN_TREEIFY_CAPACITY) resize();` inside `treeifyBin` | Confirmed: below 64 slots, it grows instead of treeifying |
| `newTab[j] = loHead;` and `newTab[j + oldCap] = hiHead;` | Confirmed: an entry can only stay or move by exactly `oldCap` |

This is the single highest-value 30 minutes in Phase 1. Do not skip it.

### Proof 2 — see the spread do its job

Create `SpreadDemo.java` from Example 1 and run it:

```bash
java SpreadDemo.java
```

| What you see | What it means |
|---|---|
| All four "NO spread" rows show the same bucket number | Confirmed: with a 16-slot table only the low 4 bits matter |
| The "WITH spread" rows show different bucket numbers | Confirmed: the XOR folded high-bit information into the low bits |
| Both blocks show the same bucket | You changed the constants — the four hash codes must differ *only* above bit 16 |

### Proof 3 — watch a bucket treeify

This needs reflection into `java.util`, which JPMS blocks by default (Topic 20).
The `--add-opens` flag is the point of the exercise as much as the output is.

`BucketInspector.java`:
```java
import java.lang.reflect.*;
import java.util.*;

public class BucketInspector {

    // Every instance collides: constant hashCode, but equals is still correct.
    record Colliding(int id) {
        @Override public int hashCode() { return 7; }
    }

    static void report(HashMap<?, ?> map) throws Exception {
        Field tableField = HashMap.class.getDeclaredField("table");
        tableField.setAccessible(true);
        Object[] table = (Object[]) tableField.get(map);

        if (table == null) {
            System.out.println("size=" + map.size() + " table=not yet allocated");
            return;
        }
        int occupied = 0, deepest = 0;
        String deepestType = "-";
        for (Object head : table) {
            if (head == null) continue;
            occupied++;
            int depth = 0;
            Object node = head;
            Field next = node.getClass().getDeclaredField("next");
            next.setAccessible(true);
            while (node != null) {
                depth++;
                node = next.get(node);
            }
            if (depth > deepest) {
                deepest = depth;
                deepestType = head.getClass().getSimpleName();
            }
        }
        System.out.printf("size=%-4d tableLen=%-5d occupied=%-4d deepestBucket=%-3d headType=%s%n",
                map.size(), table.length, occupied, deepest, deepestType);
    }

    public static void main(String[] args) throws Exception {
        HashMap<Colliding, String> map = new HashMap<>();
        for (int i = 1; i <= 200; i++) {
            map.put(new Colliding(i), "v" + i);
            if (i <= 12 || i % 10 == 0) report(map);
        }
    }
}
```

```bash
java --add-opens java.base/java.util=ALL-UNNAMED BucketInspector.java
```

**What to look for:** the `headType` column, and when `tableLen` jumps.

| What you see | What it means |
|---|---|
| `headType=Node` for the early rows | The bucket is still a plain linked list |
| `headType=TreeNode` appears at some point | That bucket converted to a red-black tree |
| `tableLen` doubling several times *before* `TreeNode` appears | Confirmed: `MIN_TREEIFY_CAPACITY=64` forces resizes first. All 200 keys collide, so no resize helps — it just keeps growing until the table reaches 64 |
| `occupied=1` throughout | Confirmed: a constant `hashCode` puts every key in one bucket, no matter how big the table gets |
| `java.lang.reflect.InaccessibleObjectException` | You forgot `--add-opens`. That is JPMS strong encapsulation doing its job — Topic 20 |
| `NoSuchFieldException: next` on a `TreeNode` | Expected in some JDK builds — `TreeNode` inherits `next` from `Node` and `getDeclaredField` does not search superclasses. Walk up with `getSuperclass()` if you hit it |

> Version note: field names inside `HashMap` are implementation detail, not API.
> They have been stable since Java 8, but a future JDK may rename them without
> warning. If this stops compiling on a new JDK, that is information, not a bug
> in the exercise.

### Proof 4 — prove that resize reorders iteration

`OrderChurn.java`:
```java
import java.util.*;

public class OrderChurn {
    public static void main(String[] args) {
        Map<String, Integer> m = new HashMap<>();
        for (int i = 1; i <= 20; i++) {
            m.put("SKU-" + i, i);
            if (i == 12 || i == 13 || i == 20) {
                System.out.println("after " + i + ": " + m.keySet());
            }
        }
    }
}
```

```bash
java OrderChurn.java
```

**What to look for:** compare the `after 12` and `after 13` lines.

| What you see | What it means |
|---|---|
| The order of the first 12 keys is different between the two lines | Confirmed: the 13th insert crossed the threshold (`16 * 0.75 = 12`) and resize relocated entries |
| The order looks the same | Possible — the split may leave the relative order of these particular keys intact. Add more keys, or run with different SKU strings. Absence of a visible change is not a guarantee of stability |
| The order is not insertion order at any point | Confirmed and expected: this is the difference from JavaScript's `Map` |

### Proof 5 — sizing actually matters

`SizingProof.java`:
```java
import java.util.*;

public class SizingProof {
    public static void main(String[] args) {
        int n = 2_000_000;
        String mode = args.length > 0 ? args[0] : "default";
        Map<Integer, Integer> m = switch (mode) {
            case "sized"  -> HashMap.newHashMap(n);
            case "wrong"  -> new HashMap<>(n);      // capacity, not mappings
            default       -> new HashMap<>();
        };
        for (int i = 0; i < n; i++) m.put(i, i);
        System.out.println(mode + " size=" + m.size());
    }
}
```

```bash
java -Xlog:gc:file=default.log -Xmx1g SizingProof.java default
java -Xlog:gc:file=wrong.log   -Xmx1g SizingProof.java wrong
java -Xlog:gc:file=sized.log   -Xmx1g SizingProof.java sized
wc -l default.log wrong.log sized.log
```

**What to look for:** the number of GC lines in each log, and the wall time.

| What you see | What it means |
|---|---|
| `default.log` has clearly the most GC lines | Confirmed: ~20 resizes, each allocating and discarding a table array |
| `sized.log` has the fewest | Confirmed: one allocation, no resize |
| `wrong.log` sits close to `sized.log` but not equal | Confirmed and instructive: `new HashMap<>(n)` avoids *most* resizes but still crosses the threshold once |
| All three are roughly the same | Your `n` is too small for the effect to clear the noise, or the JIT/GC absorbed it. Raise `n`, or lower `-Xmx`. Report what you actually saw |

> You are timing with a wall clock, which Topic 77 will teach you is the wrong
> way to benchmark. The GC *line count* is the more trustworthy signal here
> because it counts events rather than measuring duration.

---

## Practice exercises

Write real files, run them, and keep your output.

### 1 — Easy: find the resize points empirically

Write a program that inserts entries into a `HashMap` one at a time and, using
the reflection technique from Proof 3, prints the table length after each insert.
Print only the insertions where the length **changed**.

Requirements:
- Do not hardcode 16, 12, or 0.75 anywhere.
- Report the insert number at which each resize occurred.
- Then repeat with `new HashMap<>(16, 0.5f)` and with `new HashMap<>(16, 1.0f)`
  and explain, in your own words, what a load factor of 1.0 trades away and what
  it buys.

### 2 — Medium: the audit (combines Topics 01, 06, 10, 11)

Here is a fragment from an `orderflow` pricing service. It contains **five**
distinct defects drawn from Topics 01 through 12. Find them all, state the exact
symptom each one produces in production — what does the on-call engineer see, not
"it's bad practice" — and rewrite it.

```java
public class PriceBook {

    private final Map<Long, Double> priceBySku = new HashMap<>(500_000);
    private final List<Long> recentlyQuoted = new LinkedList<>();

    public double quote(Long sku, Integer quantity) {

        for (Long q : recentlyQuoted) {
            if (q == sku) {
                return 0.0;                       // already quoted this cycle
            }
        }

        Double unit = priceBySku.get(sku);
        double total = unit * quantity;

        recentlyQuoted.add(sku);
        return total;
    }

    public String auditLine() {
        return String.join(",", priceBySku.keySet().stream().map(String::valueOf).toList());
    }
}
```

Hints, so you look in the right places: Topic 01 (two defects), Topic 11 (one),
Topic 12 (two). One of the Topic 12 defects produces a *flaky test*, not a
production incident — say which and why that is arguably worse.

### 3 — Hard: production simulation

Build a small `orderflow` payment reconciler and measure the effect of hash
quality on it.

**Part A.** Write `PaymentKey` twice:
- `PaymentKeyGood` — a `record` over `(String gateway, long gatewayRef)`.
- `PaymentKeyBad` — an identical class with `equals` over both fields, but
  `hashCode()` returning only `gateway.hashCode()`.

Both must satisfy the equals/hashCode contract. `PaymentKeyBad` is legal, just
terrible. Confirm you understand why it is legal before continuing.

**Part B.** Generate 500,000 synthetic payments across 3 gateways. Build a
`HashMap` keyed by each variant, then perform 500,000 lookups (half hits, half
misses). Use `HashMap.newHashMap` for both so sizing is not a confounding
variable.

**Part C.** Run each under an allocation and CPU profile:
```bash
java -XX:StartFlightRecording=duration=60s,filename=good.jfr GoodRun
java -XX:StartFlightRecording=duration=60s,filename=bad.jfr  BadRun
```
Open both in JDK Mission Control. Report:
- Where the CPU time is in each. Name the specific methods.
- Whether `HashMap$TreeNode` appears in the bad run and not the good one.
- Use the Proof 3 inspector to report `occupied` and `deepestBucket` for each map.

**Part D.** Now argue against yourself. The bad version still completes. It is
O(log n), not O(n) — treeification did its job. Under what circumstances would
you *not* fix this? State the condition that flips your answer, in terms of data
volume growth rate and the cost of the change.

> Flight Recorder is Topic 78 and you are using it early and shallowly. That is
> fine. You are collecting one fact — where the CPU went — not learning the tool.

---

## Interview questions

### Q1 — "Why must a HashMap's table be a power of two?"

**Mid-level answer:** "So it can use a bitmask instead of modulo. `(n-1) & hash`
is faster than `hash % n`."

**Senior answer:** "Two reasons, and the second one is the real one. First,
`(n-1) & hash` replaces an integer division, which is one of the slowest integer
operations on the chip, with a single AND — on a path that runs on every get and
put. Second, and more important: doubling the table adds exactly one bit to the
mask, so every entry in old bucket `j` can only go to `j` or `j + oldCap`. That
turns resize into a single pass that splits each bin into a lo-list and a
hi-list, using the cached hash, with no rehashing and no `equals` calls at all.
Modulo with a prime table would give a better distribution but would force a full
rehash on every resize. The cost of the choice is that only the low bits index
the table, which is exactly why the spread function has to fold the high bits
down. The power-of-two decision and the XOR spread are one decision, not two."

**What separates them:** the mid-level answer knows the optimisation. The senior
answer knows that power-of-two is what makes *resize* cheap, and connects it
causally to why the spread exists at all.

**Follow-up the interviewer asks:** "So what does `Hashtable` do differently, and
was `HashMap` right?" They want you to know `Hashtable` uses a prime modulus, and
to argue the trade rather than assert that newer is better.

---

### Q2 — "Explain `h ^ (h >>> 16)`. Why not just use `hashCode()` directly?"

**Mid-level answer:** "It mixes the bits so they're distributed more evenly."

**Senior answer:** "Because the table index is `(n-1) & hash`, and for a
16-element table that is only the low four bits — the other 28 bits of your hash
code are discarded entirely. So any key type whose hash codes vary mainly in the
high bits maps every entry to one bucket. Shifting right by 16 and XORing folds
the high half into the low half, so every low bit now carries information from a
high bit. It costs one shift and one XOR, which is deliberately cheap — this is
not a good hash function and is not trying to be, it is a targeted patch for one
specific failure mode caused by the power-of-two masking. A genuinely bad
`hashCode` is still bad after spreading; the spread only rescues a *good* hash
whose entropy sits in the wrong place."

**What separates them:** naming the *specific* failure mode being patched, and
being explicit that the spread does not rescue a bad `hashCode`. The mid answer
implies the spread is general bit-mixing; it isn't.

**Follow-up:** "Give me a real key type that fails without the spread." Good
answers: `Long` keys holding timestamps or IDs (`Long.hashCode` is
`(int)(value ^ (value >>> 32))`, so a monotonic ID's variation lives low, but a
value scaled by a large power of two puts it high); or any hand-written
`hashCode` that multiplies by a large constant.

---

### Q3 — "Why treeify at 8 and untreeify at 6? Why not both at 8?"

**Mid-level answer:** "Eight is when a bucket is too long, and six is when it's
short enough to go back."

**Senior answer:** "The gap is hysteresis. If both thresholds were 8, a workload
sitting on the boundary would convert list to tree and back on alternating put
and remove, paying the full conversion cost every operation. Two nodes of slack
means you have to genuinely move away from the boundary before paying again.
As for 8 itself: with load factor 0.75 and a decent hash, bucket occupancy is
Poisson with lambda 0.5, so a bucket reaching 8 has probability around six in a
hundred million. The threshold is set where 'this is chance' stops being the
likely explanation and 'your hashCode is broken or someone is attacking you'
starts. That matters because `TreeNode` carries parent, left, right and prev
pointers on top of a `Node`, so treeification is a real memory cost you do not
want to pay in the normal case. And there is a third constant people forget:
`MIN_TREEIFY_CAPACITY` is 64 — below that, a crowded bucket triggers a resize
instead of a treeify, because in a small table crowding is a sizing problem, not
a hash-quality problem."

**What separates them:** naming hysteresis, giving the statistical justification
for 8, and volunteering `MIN_TREEIFY_CAPACITY` unprompted. That third constant is
the one that separates people who read the source from people who read a summary.

**Follow-up:** "What if the keys aren't `Comparable`? How does the tree order
them?" They want: hash first, then `compareTo` if available, then a deterministic
tie-break on class name and `System.identityHashCode`. The tree only needs *a*
total order, not a meaningful one.

---

### Q4 — "Walk me through what `resize()` does. What does it do to iteration order?"

**Mid-level answer:** "It doubles the capacity and rehashes all the entries into
the new table. Iteration order changes because the entries move."

**Senior answer:** "It doubles capacity and threshold, allocates a new array, and
redistributes — but it does not rehash. Every node caches its spread hash as a
final field. Doubling adds one bit to the mask, and that bit is `oldCap`, so each
entry either stays at index `j` or moves to `j + oldCap` depending on
`(hash & oldCap) == 0`. Each bin is walked once and split into a lo-list and a
hi-list, preserving relative order, then attached at those two indices. No
`hashCode` call, no `equals` call, no key dereference. A treeified bin splits the
same way and untreeifies either half that lands at six or fewer nodes.
For iteration order: `HashMap`'s order was never specified, and resize is one of
the things that changes it — which is why any test asserting on the iteration of
a `HashMap` is latently broken. The Java 8 order preservation within a split is
worth knowing for a different reason: Java 7 reversed the list during transfer,
which under concurrent modification could form a cycle and spin a core forever.
Java 8 removed that specific failure — but `HashMap` is still not thread-safe,
and I'd want to correct anyone who says Java 8 'fixed' it."

**What separates them:** "it does not rehash" is the whole answer, and the mid
answer gets it backwards. Then the volunteered correction about Java 7's infinite
loop — offering the nuance rather than repeating the folk version.

**Follow-up:** "So is `HashMap` safe to read concurrently if nobody writes?" Yes,
if the publication is safe (Topic 17/88) and there is genuinely no write. The
interesting part is that "nobody writes" is harder to guarantee than it sounds,
and that `LinkedHashMap` in access order breaks it because `get` mutates
(Topic 15).

---

### Q5 — "How is a bad `hashCode` a denial-of-service vector?"

**Mid-level answer:** "If all the keys collide, lookups become O(n) instead of
O(1), so the map gets slow."

**Senior answer:** "The attack shape is: find an endpoint that puts
attacker-controlled strings into a hash map — a JSON body deserialized into
`Map<String, Object>`, query parameters, HTTP headers — and send keys engineered
to collide. `String.hashCode` is a published, deterministic, non-randomised
algorithm, so collisions are trivially computable; `Aa` and `BB` collide and you
generate thousands by concatenation. Pre-Java-8 that gave quadratic insertion
from one modest request, and because it burns a request thread, a handful of
requests take out the whole service, not just that endpoint. Java 8's
treeification bounds it at O(log n), which is a genuine mitigation but not a
cure — building the tree still costs, and `TreeNode` memory is real. Some
languages randomise string hashing per process to kill it outright; Java can't,
because `String.hashCode` is specified in the Javadoc and the ecosystem depends
on the value. Java 7u6 shipped opt-in alternative hashing and it was pulled in
8 when treeification landed. The actual fix is input-side: bound the key count
you accept from untrusted input, and use a typed DTO instead of
`Map<String, Object>` so the question never arises."

**What separates them:** treating it as a security question with an attack path,
naming *why* Java cannot randomise, and locating the fix at the input boundary
rather than in the collection.

**Follow-up:** "Does using a `record` as your key protect you here?" No — a
`record`'s generated `hashCode` protects you from *your own* bad hash function,
not from an attacker choosing `String` keys. Different problem.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `HashMap` caches the spread hash in each node, which costs 4 bytes per entry.
   Name two things that become possible because of that cache, and one bug that
   becomes possible.

2. The default load factor is 0.75. What would go wrong at 0.5? At 1.0? Which of
   those two would you actually consider changing in production, and what would
   have to be true first?

3. Treeification requires the tree to have *some* total order over keys. Why is a
   tie-break on `System.identityHashCode` acceptable there, when using
   `identityHashCode` in a real `hashCode()` would be a bug?

4. Suppose Java had chosen a prime table size and modulo indexing. Describe how
   `resize()` would have to work instead. What would that cost, and what would it
   buy?

5. `HashMap` allows one `null` key, mapped to bucket 0. `ConcurrentHashMap` and
   `TreeMap` both reject `null` keys. Give a defensible reason for each choice
   rather than calling one of them wrong.

6. A colleague proposes making `hashCode()` more expensive but higher quality —
   say, a proper 64-bit mix — for a hot key type. Under what conditions is that a
   win? Under what conditions does it lose even though the distribution improves?

7. Your JavaScript instinct says `Map` iterates in insertion order and that is a
   *guarantee*. Java made the opposite choice for `HashMap`. What does Java buy
   by refusing to specify iteration order, and what would it have cost to
   guarantee it?

---

## Quick reference card

### The constants

| Constant | Value | Notes |
|---|---|---|
| `DEFAULT_INITIAL_CAPACITY` | 16 | Table is allocated lazily on first put |
| `DEFAULT_LOAD_FACTOR` | 0.75f | Resize when `size > cap * lf` |
| `MAXIMUM_CAPACITY` | 1 << 30 | About 1.07 billion slots |
| `TREEIFY_THRESHOLD` | 8 | List becomes a red-black tree |
| `UNTREEIFY_THRESHOLD` | 6 | Tree becomes a list again (hysteresis) |
| `MIN_TREEIFY_CAPACITY` | 64 | Below this, resize instead of treeify |

### The two lines that are the whole topic

```java
// spread
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// index
int i = (table.length - 1) & hash;
```

### Complexity

| Operation | Good hash | All keys collide, table < 64 | All keys collide, treeified |
|---|---|---|---|
| `get` / `containsKey` | O(1) | O(n) | O(log n) |
| `put` | O(1) amortised | O(n) | O(log n) |
| `remove` | O(1) | O(n) | O(log n) |
| iteration | O(capacity + size) | O(capacity + size) | O(capacity + size) |

That last row is the one people miss: iterating a `HashMap` walks **every table
slot**, empty or not. An over-sized, near-empty map iterates slowly.

### Key APIs

```java
HashMap.newHashMap(n)              // size for n MAPPINGS  (Java 19+, fine on 21)
new HashMap<>(cap, loadFactor)     // cap is SLOTS, not entries
map.getOrDefault(k, d)             // no NPE on unboxing (Topic 01)
map.putIfAbsent(k, v)              // returns the existing value, or null
map.computeIfAbsent(k, fn)         // fn runs only on a miss
map.merge(k, delta, Long::sum)     // the counter idiom
map.compute(k, (k2, v) -> ...)     // null return removes the entry
map.forEach((k, v) -> ...)
map.entrySet()                     // iterate here, not keySet() + get()
Objects.hash(a, b, c)              // varargs: allocates. Fine outside hot paths
```

### Gotchas checklist

- [ ] `HashMap` iteration order is unspecified and changes on resize. Never assert on it.
- [ ] `new HashMap<>(n)` sizes slots, not entries. Use `HashMap.newHashMap(n)`.
- [ ] Keys must be immutable, or at least their hash-relevant fields must be. (Topic 13)
- [ ] `map.get(k) == null` is ambiguous. Use `containsKey` when it matters.
- [ ] `HashMap` is not thread-safe. Java 8 removed one failure mode, not the class of them. (Topic 92)
- [ ] A legal `hashCode` can still be a performance disaster.
- [ ] `Objects.hash(...)` allocates an array and boxes primitives on every call.
- [ ] Iterate `entrySet()`, not `keySet()` followed by `get()` — that is two lookups per entry.
- [ ] Untrusted keys in a map are a DoS surface. Bound them at the input boundary.

---

## When would I use this at work?

**1. Sizing a cache or an index before you write it.**
Product says the catalogue is 800,000 products and you need an in-memory lookup.
You reach for `HashMap.newHashMap(800_000)` rather than `new HashMap<>()`, and
you can say out loud why: twenty resizes, each re-linking every entry present so
far, all during startup, all avoidable with one method call. That is a
five-second decision that removes a startup-latency ticket six months later.

**2. Reading a flame graph and seeing `HashMap.getNode` at the top.**
Most engineers see that and conclude "the map is hot, that's expected". You know
that a well-distributed `HashMap` lookup is one array read plus one `equals`, and
should almost never dominate a profile. So `getNode` at the top means either an
absurd call volume or a degenerate bucket — and you go straight to the key type's
`hashCode` instead of trying to call the map less often.

**3. Reviewing a PR that adds a field to a key class.**
Someone adds `currency` to `PaymentKey`'s `equals` and forgets `hashCode`, or
adds it to `hashCode` and forgets `equals`. You catch it in review. If the class
is a `record`, the problem cannot occur — which is a good enough reason on its
own to push for records as key types. Topic 13 is the full version of this.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing:** `Map<Long, Long>` boxes every key and value, and
  `Long.hashCode()` is `(int)(v ^ (v >>> 32))` — the same fold-the-high-bits-down
  trick, one level down. Also why `Objects.hash` allocates.
- **06 — Type erasure:** why the table is `Node<K,V>[]` and why you cannot have a
  `HashMap<int, int>` at all.
- **10 — Collections Framework:** `Map` is the contract; `HashMap` is one
  implementation decision. Also `ConcurrentModificationException` and the
  `modCount` field you will see all over `HashMap`'s source.
- **11 — List implementations:** the bucket chain is a linked list, and everything
  you learned there about pointer chasing and cache misses applies inside it.

**This unlocks:**
- **13 — equals/hashCode contract:** the correctness rules behind everything here.
  Read it next; these two topics are really one topic split in half.
- **14 — TreeMap and NavigableMap:** what you use when you need order, and why
  `TreeMap` defines equality differently.
- **15 — LinkedHashMap:** `HashMap` plus a linked list across entries, which is
  how you get predictable iteration order and a six-line LRU.
- **16 — EnumMap and EnumSet:** what to use when the key space is known and small
  — no hashing at all.
- **17 — Immutability:** why immutable keys are the only safe keys.
- **27 — Records:** the correct default for a composite map key.
- **69 — Object layout:** the real byte cost of a `Node` versus a `TreeNode`.
- **78 — Profiling:** where `HashMap.getNode` shows up and what it means when it
  does.
- **79 — Memory leaks:** an unbounded `HashMap` used as a cache is the single most
  common Java leak shape.
- **92 — ConcurrentHashMap:** the same structure, made safe — no segments since
  Java 8, lock-free reads, CAS or `synchronized` per bin on write.

---

*Java baseline 21. Nothing in `HashMap`'s core algorithm changed between Java 21
and Java 25 — the design has been stable since Java 8. `HashMap.newHashMap` is
Java 19+, so it is available throughout this curriculum but not on a Java 17
codebase. `[JAVA 25]` Compact object headers shipped as a product feature, which
shrinks the per-entry overhead of every `Node` in the table; if you are capacity
planning a very large map on 25, measure it with JOL rather than assuming the
Java 21 numbers. The 21 fallback is simply the existing header size, which is
what Topic 69 measures.*
