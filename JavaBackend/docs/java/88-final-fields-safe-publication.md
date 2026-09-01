# 88 — `final` Field Semantics and Safe Publication

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is why a thread in `orderflow` can hold a non-null `Order` reference whose `total` is zero and whose `status` is null — an order that exists and has nothing in it.

---

## Before anything else — what is and is not in this document

**I do not have a JVM and I have never run jcstress on your machine. There is not a single
captured test result in this document, and I will never present anything as if there were.**

Specifically, you will not find here:

- a jcstress outcome table with sample counts or frequencies in it,
- a statement that a particular outcome "occurs about 0.1% of the time",
- a claim that any bug in this document *will* reproduce on your machine,
- a timing, a throughput figure, or a percentile.

**Why the rule is decisive here.** Publication bugs are rare-event bugs. If I gave you a
frequency, you would run the test, not see it, and conclude your code is safe. **That
conclusion would be wrong and it would ship.** The frequency of a race is a property of a
machine, a JDK, a core layout and a run — never of the code.

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in the output *you* generate,
- a **"what you see" → "what it means"** table covering the plausible outcomes and the
  surprising ones.

### The labelled exception

To read a jcstress report you must know its columns. In one place I show the **column
structure** of an outcome table, with every value replaced by `<n>` or `xxx`, carrying the
inline label:

> *illustration of the format, not captured output*

Placeholders only. No plausible-looking numbers, ever.

### The two honesty rules that govern everything below

These are stated in Topics 86 and 87 and they are load-bearing here. Read them twice.

**Rule 1 — a `FORBIDDEN` outcome with zero observations means "not observed on this machine,
this JDK, this run". It does NOT mean "proven impossible".**

jcstress is a falsifier, not a prover. It hammers an interleaving window and reports what it
saw. Zero observations of a forbidden outcome is **failure to falsify** your argument — a real
and valuable result, and not proof. **The proof is the happens-before argument.** jcstress is
how you catch that argument being wrong; never how you establish it. If you cannot write the
argument in words, a green run tells you nothing you should act on.

**Rule 2 — x86 is Total Store Order and hides bugs that aarch64 exposes. Here is which way
that cuts.**

x86-TSO forbids store-store and load-load reordering in hardware; aarch64 — your Apple
Silicon Mac — permits both. This document's bug depends precisely on a store-store reordering
being visible to another thread: the object's field writes observed *after* the write of the
reference. Therefore:

- **Your Mac is the better instrument for this topic.** If a publication bug is going to be
  observed anywhere, it is more likely to be observed on aarch64.
- **A green run on x86 CI is weak evidence.** It is close to no evidence for this specific
  bug class, because the hardware forbids the reordering the bug needs. A team that tests
  only on x86 and deploys to Graviton or Ampere has tested nothing relevant.
- **And even on aarch64: never claim the bug "will" reproduce.** The compiler may not have
  reordered anything on this run. The window may be too narrow. The JIT may not have
  compiled the code. **"I did not observe it" and "it cannot happen" are different
  statements and you must never conflate them.**

The correct sentence, which you should be able to say out loud: *"I ran it on aarch64 and
observed the outcome / did not observe the outcome; the reason the code is correct is the
happens-before argument, not the test result."*

### `[JAVA 25]` compact object headers, in one line

JDK 25 makes compact object headers a product feature, shrinking the object header. That
changes object *sizes* and any `B/op` or heap figure you measure — **it changes nothing about
final-field semantics or the freeze.** Settle sizes with JOL, semantics with the JLS.

---

## Mechanical statement

> **The end of a constructor with `final` fields carries a freeze action: any thread that
> sees the reference is guaranteed to see correctly-initialised final fields, with no
> synchronisation. Non-final fields carry no such guarantee — a reader can see a non-null
> reference to an object whose fields are still default values.**

Six mechanical claims, each of which you must be able to defend:

1. **The freeze is a real, specified action.** It is not folklore and not an implementation
   detail. The Java Language Specification's final-field semantics say that a `final` field
   correctly initialised in a constructor is guaranteed visible to any thread that obtains a
   reference to the object *without* that thread doing anything.

2. **The guarantee is one-directional and unusual.** Every other happens-before edge in the
   JMM (Topic 86) requires **both** sides to participate — a volatile write and a volatile
   read, a monitor unlock and a monitor lock. **Final fields are the only mechanism where
   the reader does nothing.** That is why they are so valuable and why they feel like magic.

3. **It applies to `final` fields only, and to what they reach at construction end.** A
   `final` reference field guarantees the *reference* is visible, and — importantly — the
   objects reachable from it as of the freeze. It guarantees nothing about later mutation of
   those objects.

4. **"Correctly initialised" excludes any object whose constructor leaked `this`.** If the
   reference escapes before the constructor finishes, another thread can hold it before the
   freeze, and the guarantee evaporates for that reference.

5. **Non-final fields have no freeze.** The constructor's field writes and the publishing
   write of the reference are ordinary stores. The compiler may reorder them, the CPU may
   make them visible out of order, and a reader with no happens-before edge may see the
   reference before the fields.

6. **Reading a non-null reference is not evidence of anything.** The reader sees a reference
   because a store became visible. It has no information about which *other* stores became
   visible. This is the sentence that makes the whole topic click.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

**A single-threaded runtime cannot observe a partially-constructed object.**

This is not "the analogy is imperfect". There is no analogy. In JavaScript, by the time any
other code can reach a reference, the constructor has run to completion, because there is no
other code running concurrently. Run-to-completion semantics make the entire question
unaskable. You have never seen this bug, you have never had to prevent it, and you have no
intuition for it — because your runtime made it structurally impossible.

The one place your intuition might reach: a `this`-leak inside a JS constructor, where a
callback registered there observes a half-built object. That is a real JS bug and **not the
same bug** — that callback runs *later*, on a subsequent turn of the loop, after the
constructor has finished, and what it sees is deterministic. Here we mean a **different
thread, at the same instant, on a different core, with its own cache, seeing your
constructor's writes in a different order than you issued them.**

### Do not map this onto the event loop. Explicitly.

This is the topic where the wrong model produces the most confident wrong code. If you think
*"the object is created before anyone can use it"*, you are reasoning from run-to-completion.
In Java that is **false from another thread's perspective.** Two threads are genuinely
simultaneous; there is no ordering between them unless you create one.

### The one thing that half-transfers: `readonly`

TypeScript's `readonly` and Java's `final` both prevent reassignment and both are shallow —
a `readonly` array can be mutated, and a `final List` field can have elements added.

**Where it stops:** `readonly` is erased at compile time and has no runtime existence. `final`
has **memory-model meaning** — it changes what other threads are guaranteed to see. No
TypeScript keyword changes what another thread observes, because there is no other thread.

**This is the payoff Topic 17 promised.** Topic 17 said: "`final` prevents reassignment of
the reference, not mutation of the target. Its real power is JMM: correctly-constructed final
fields are guaranteed visible to other threads without synchronisation." This document is that
sentence in full, with the trace and the test.

### What actually transfers

- **Preferring immutable data** transfers, and here it buys thread safety for free rather than
  merely preventing accidental mutation.
- **Your instinct that "construct then fill in" is bad design** transfers, and gains a
  mechanical justification: a two-phase constructor is a publication hazard, not just a smell.
- **Defensive copying** transfers, and is what makes the freeze guarantee mean something for
  collections.

---

## What is this?

### The definition, precisely

**Publication** is making a reference available to another thread: storing it in a static
field, an instance field of a shared object, a collection, a queue, a `Future`; passing it to
a thread's constructor; returning it from a getter another thread calls.

**Safe publication** means any thread that obtains the reference is guaranteed to see the
object fully constructed. **Unsafe publication** means it might not.

"Might" is doing enormous work here. It does not mean "briefly, then it catches up". It means
a reader can observe a non-null reference to an object whose fields hold their default values
— `0`, `null`, `false` — and nothing in the specification says it will ever see anything else
without an edge.

### Why the reference and the fields can separate

Three independent mechanisms, any one of which suffices. **You must name all three**, because
fixing one does not fix the others.

1. **The compiler may reorder.** Javac emits nearly literal bytecode (Topic 76), but C2 may
   reorder stores with no data dependency between them (Topics 74–75, 86) — and the
   constructor's field stores and the reference store are independent.
2. **The CPU may reorder, or make stores visible out of order.** On aarch64 stores from one
   core can become visible to another in a different order. On x86-TSO they cannot — Rule 2,
   and why your Mac is the better instrument.
3. **The store buffer.** A core's writes sit in a store buffer before reaching cache
   coherence, and two writes can drain in either order absent a barrier.

**And a fourth:** even with no reordering at all, a reader with no happens-before edge is
under no obligation to observe *any* particular write. Absent an edge, a read may return a
stale value indefinitely — the JMM does not promise "eventually" (Topic 86).

### The final-field freeze

When a constructor that writes `final` fields completes normally, a **freeze action** occurs
for each. The specification's guarantee, stated carefully:

> If a thread obtains a reference to an object **after** that object's constructor completed,
> and the reference was not made available to that thread before the freeze, then that thread
> is guaranteed to see the correctly-initialised values of the object's `final` fields — and
> the values of anything reachable from those `final` fields as of the freeze.

Three conditions hidden in that sentence, all of which can be violated:

| Condition | How it is violated |
|---|---|
| The constructor **completed normally** | Not a problem in practice, but a constructor that throws leaves an object no one should hold |
| The reference was **not published before the freeze** | **`this` escaping from the constructor.** Trap 5. This is the common violation |
| The fields are **`final`** | Any non-final field is outside the guarantee entirely — including in the same object |

Mechanically, HotSpot implements the freeze with a **`StoreStore` barrier at the end of the
constructor**: the constructor's stores are ordered before any store that follows, including
the publishing store of the reference. That barrier is the whole cost, and on most
architectures it is very cheap or free.

**Note what the reader does: nothing.** No volatile read, no lock, no barrier. It is the only
mechanism in the JMM with that property — which makes it the cheapest correct publication in
Java, and makes "make the fields final" a high-leverage change.

### What the freeze does NOT give you

Be precise here; this is where people over-claim.

| It guarantees | It does not guarantee |
|---|---|
| The `final` field's value as of construction end | Anything about **non-final** fields in the same object |
| Objects reachable through the `final` field, as of the freeze | Any **later mutation** of those objects |
| Visibility without reader-side synchronisation | Atomicity of anything |
| Correct publication of *this* object | Correct publication of an object stored into a `final` collection *after* construction |

**The trap that follows from row two is Trap 2**, and it is the most common real-world
version: a `final List<OrderLine>` field. The reference is frozen; the *contents* are not, if
you add to them after construction. `final` froze the reference, not the data.

### The five safe-publication idioms

Memorise these. They are the answer to the interview question and they are the review
checklist.

| # | Idiom | Mechanism | Cost | When |
|---|---|---|---|---|
| 1 | **Static initializer** | Class initialisation holds a lock and carries an edge (Topic 67) | Free; lazy per-class | Constants, singletons, immutable tables |
| 2 | **`volatile` field or `AtomicReference`** | Volatile write→read edge (Topic 87) | Cheap read, more expensive write | A reference replaced over time |
| 3 | **`final` field** | **The freeze. No reader-side cost at all** | Essentially free | **The default. Prefer this** |
| 4 | **Guarded by a lock** | Monitor unlock→lock edge (Topic 85) | Contention-dependent | Mutable state needing atomicity too |
| 5 | **Into a thread-safe collection** | The collection's own internal edges (Topic 92) | Collection-dependent | Handing objects to other threads |

Idiom 5 is worth spelling out because people distrust it: putting an object into a
`ConcurrentHashMap`, a `BlockingQueue` or a `CopyOnWriteArrayList` before another thread
retrieves it **safely publishes it** — a documented guarantee of the concurrent collections,
not a side effect. **Meta-rule:** reach for idiom 3 first; it is the only one that asks
nothing of the reader and cannot be got wrong at the call site.

---

## Concurrency trace

**Read this before any correct code.** This is the bug, step by step, with nothing hidden.

### The scenario

`orderflow` has a hot-order cache: the most recently placed order for a customer is kept in a
plain field so a follow-up `GET /orders/latest` can serve it without a database round trip.
Thread A is the request thread placing an order. Thread B is a different request thread
reading the cache.

```java
// THE BROKEN CODE. Do not copy this. It is here to be traced.
package com.orderflow.orders;

public final class LatestOrderCache {

    // PLAIN field. Not volatile. No lock. This is the publication channel.
    private Order latest;

    public void publish(Order order) { this.latest = order; }
    public Order latest()            { return this.latest; }
}

// The object being published. NOTE: the fields are NOT final.
public final class Order {
    private long id; private long totalCents; private String status;   // no `final`

    public Order(long id, long totalCents, String status) {
        this.id = id; this.totalCents = totalCents; this.status = status;
    }
    public long id() { return id; }
    public long totalCents() { return totalCents; }
    public String status() { return status; }
}
```

### The interleaving

Two threads, time running downward. **The right column is what Thread B actually observes**,
not what you intended.

| # | Thread A — places the order | Thread B — reads the cache |
|---|---|---|
| 1 | Allocates memory for a new `Order`. All fields hold **default values**: `id=0`, `totalCents=0`, `status=null`. | — |
| 2 | — | Reads `cache.latest`. Gets `null`. Returns "no cached order". Correct, uninteresting. |
| 3 | Constructor: writes `id = 90210`. **Store sits in A's store buffer.** | — |
| 4 | Constructor: writes `totalCents = 4599`. **Also in the store buffer.** | — |
| 5 | Constructor: writes `status = "PLACED"`. **Also in the store buffer.** | — |
| 6 | Constructor returns. **No freeze action — the fields are not `final`, so no `StoreStore` barrier is emitted.** | — |
| 7 | `cache.latest = order` — writes the **reference** into the shared plain field. | — |
| 8 | **The store of the reference drains to memory before the three field stores.** Permitted: no barrier, no data dependency, and on aarch64 the hardware allows store-store reordering. | — |
| 9 | — | Reads `cache.latest`. **Gets a non-null reference.** |
| 10 | — | Reads `order.id()`. **Sees `0`.** The store from step 3 is not visible yet. |
| 11 | — | Reads `order.totalCents()`. **Sees `0`.** |
| 12 | — | Reads `order.status()`. **Sees `null`.** |
| 13 | — | Returns an order object to the caller: `{ id: 0, totalCents: 0, status: null }` |
| 14 | Field stores finally drain. Memory is now consistent. | Too late. Thread B has already responded. |

### What to take from this trace

**Step 9 is the sentence that matters: Thread B read a non-null reference, and that fact
carries no information about any other write.** B did not read the reference "after" the
constructor in any meaningful sense — there is no "after" between two threads without an edge.

Note what the trace does **not** require: no unlucky context switch (the threads run genuinely
simultaneously on different cores); nothing exotic from the JIT (step 8 is permitted by aarch64
hardware alone); and no load (only that the window is hit).

And note step 6, the fix in negative: **had the fields been `final`, step 6 would emit a
`StoreStore` barrier**, step 8's reordering would be forbidden, and steps 10–12 would be
impossible for any thread obtaining the reference after the freeze.

### The outcome in business terms

`orderflow` returns HTTP 200 with an order whose `id` is 0, whose total is £0.00, and whose
status is `null`.

Downstream, worst last:

- The customer's app renders an order for zero pounds. Support tickets.
- Reconciliation sums `totalCents` across cached orders and under-reports revenue.
- A `switch` on `status` hits the `null` case and throws an NPE at a place the code proves
  cannot be null — the constructor sets it and there is no setter.
- The payment service is asked about order `0` and logs an error nobody can trace.
- **And it does not reproduce.** Under load, on some machines, on some architectures, at some
  rate. Closed as "could not reproduce" twice before anyone reads the code.

**That last bullet is why this topic is ELITE.** The bug is invisible to testing, invisible to
review by anyone who has not learned it, and its symptoms point everywhere except the cause.

---

## Example 1 — minimal

The smallest complete pair: the broken version and the fixed one, side by side, with nothing
else in them.

```java
package com.orderflow.lab.pub;

/** BROKEN. Non-final fields, published through a plain field.
 *  A reader can see a non-null reference with default field values. */
final class UnsafeOrder {
    private long totalCents; private String status;               // NOT final
    UnsafeOrder(long totalCents, String status) {
        this.totalCents = totalCents; this.status = status;
    }
    long totalCents() { return totalCents; }
    String status() { return status; }
}

/** CORRECT. Final fields => freeze at constructor end => StoreStore barrier.
 *  Any thread obtaining this reference after construction sees both fields. */
final class SafeOrder {
    private final long totalCents; private final String status;   // FINAL
    SafeOrder(long totalCents, String status) {
        this.totalCents = totalCents; this.status = status;
    }
    long totalCents() { return totalCents; }
    String status() { return status; }
}

/** Even better in Java 21: a record. All components are implicitly final,
 *  so you get the freeze for free and cannot forget it. */
record RecordOrder(long totalCents, String status) { }
```

**The difference is one keyword per field, and it changes the memory model.** Nothing else
about these classes differs — same shape, same size, same accessors.

**WHAT TO LOOK FOR:**

```bash
javac -d out com/orderflow/lab/pub/*.java
# Does the CONSTRUCTOR differ? Look at the end of each <init>.
javap -c -p -cp out com.orderflow.lab.pub.UnsafeOrder | sed -n '/<init>/,/^$/p'
javap -c -p -cp out com.orderflow.lab.pub.SafeOrder   | sed -n '/<init>/,/^$/p'
# Is the field flagged final in the class file? This IS visible.
javap -p -cp out com.orderflow.lab.pub.SafeOrder
javap -p -cp out com.orderflow.lab.pub.UnsafeOrder
```

| What you see | What it means |
|---|---|
| `javap -p` shows `private final long totalCents;` on `SafeOrder` and no `final` on `UnsafeOrder` | Correct. The `ACC_FINAL` flag is what C2 reads to decide whether to emit the freeze barrier |
| The two `<init>` bytecode bodies look essentially identical | **Expected, and it is the lesson.** The freeze is not a bytecode instruction. It is a constraint the JIT honours when generating machine code, based on the field's `final` flag. **Bytecode cannot show you this** |
| You went looking for a barrier opcode and found none | Correct — there is no such opcode. Barriers are emitted by the JIT into machine code. Seeing them needs `-XX:+PrintAssembly` with `hsdis`, which most JDKs do not bundle |

**An important negative result: `javap` settles Topic 76's questions and cannot settle this
one.** Different question, different instrument. The instrument here is jcstress.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m`, G1 (confirm with `jcmd VM.flags -all`) |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Threads | Thread-per-request; many request threads truly concurrent on 2 vCPU |
| Baseline | `/docs/java/baselines/` — p50/p95/p99/p999 per endpoint |

### The code that ships

A promotion-pricing cache. Prices change rarely; reads are 70% of the load mix; someone
correctly identifies the database round trip as waste and adds a cache.

```java
package com.orderflow.pricing;

import java.util.HashMap;
import java.util.Map;

/** The pricing snapshot handed to every request thread. */
public final class PricingSnapshot {

    private Map<Long, Long> unitPriceCents;   // NOT final
    private long            version;          // NOT final
    private String          promotionCode;    // NOT final

    public PricingSnapshot(Map<Long, Long> prices, long version, String promotionCode) {
        this.unitPriceCents = new HashMap<>(prices);   // defensive copy - good instinct
        this.version        = version;
        this.promotionCode  = promotionCode;
    }
    public long priceFor(long productId) {
        Long p = unitPriceCents.get(productId);        // NPE risk if map is null
        return p == null ? 0L : p;
    }
    public long version() { return version; }
    public String promotionCode() { return promotionCode; }
}

@Component
public final class PricingCache {

    // PLAIN field. Read by every request thread. Written by the refresh thread.
    private PricingSnapshot current = new PricingSnapshot(Map.of(), 0L, "NONE");

    /** Called every 60 seconds by a @Scheduled task on ITS OWN THREAD. */
    public void refresh(Map<Long, Long> prices, long version, String code) {
        this.current = new PricingSnapshot(prices, version, code);   // UNSAFE PUBLICATION
    }

    /** Called by every request thread, thousands of times per second. */
    public PricingSnapshot current() { return this.current; }
}
```

**Everything about this looks careful** — defensive copy, `final` class, map copied rather
than aliased. **And it is broken in two independent ways:**

1. **The fields are not `final`**, so there is no freeze and no `StoreStore` barrier at
   construction end.
2. **The publishing field `current` is not `volatile`**, so there is no release/acquire edge
   between the refresh thread's write and the request threads' reads.

Either fix alone closes the hole. Fixing both is what you should actually do, for reasons
below.

### What you observe, in the order you observe it

1. Fine for weeks. The cache works, p99 improves. It is a good change.
2. A rare NPE inside `priceFor`, dereferencing `unitPriceCents` — **a field the constructor
   unconditionally assigns and no setter modifies.** The stack trace is impossible.
3. Closed as "transient". It happens twice more over a month.
4. A handful of orders priced at `0`. Finance notices before engineering does.
5. `promotionCode` logged as `null` in a field declared non-nullable, breaking a consumer.
6. Someone notices the incidents cluster **within a second of a cache refresh**.
7. **On x86 staging it never reproduces.** On aarch64 production nodes it does. That is
   Rule 2, in production, costing real money.

### Why each symptom happens

| Symptom | Mechanism |
|---|---|
| NPE on `unitPriceCents` | A request thread saw the new `PricingSnapshot` reference before the store of `unitPriceCents` was visible. The field held its default: `null` |
| Price of `0` | It saw a non-null but *empty* map — the `HashMap` reference visible but its internal `table` array not yet, or the map itself partially constructed. **The freeze does not reach into a non-final field's target** |
| `promotionCode` null | Same as the NPE, different field |
| Clusters at refresh | The window exists only around the publishing write. Between refreshes every thread reads a long-settled reference |
| Only on aarch64 | x86-TSO forbids the store-store reordering. **Staging tested nothing relevant** |

### The diagnosis, as commands

```bash
# 1. Is the publishing field volatile? Is the published object's state final?
#    This is a code question, and it is faster than any tool.
javap -p -cp target/classes com.orderflow.pricing.PricingSnapshot
javap -p -cp target/classes com.orderflow.pricing.PricingCache
#    LOOK FOR: 'final' on every field of PricingSnapshot, 'volatile' on PricingCache.current.
#    Absent => you have found it. Stop here and fix it.

# 2. Encode the bad outcome as a jcstress test and run it. This is the real diagnosis.
mvn clean verify
java -jar target/jcstress.jar -t OrderUnsafePublication

# 3. Record what you ran it on. This is PART OF THE RESULT, not metadata.
java --version && uname -m && grep -A2 jcstress pom.xml

# 4. Correlate the incidents with refresh timing, to confirm the window.
grep -E 'pricing.refresh|NullPointerException' /var/log/orderflow/app.log \
  | awk '{print $1, $2}' | sort | uniq -c
```

### The fixes, in the order you should apply them

**Fix 1 — make every field `final`.** This is the change that fixes the whole class of bug
and costs nothing at runtime.

```java
public final class PricingSnapshot {
    private final Map<Long, Long> unitPriceCents;   // FINAL
    private final long            version;          // FINAL
    private final String          promotionCode;    // FINAL

    public PricingSnapshot(Map<Long, Long> prices, long version, String code) {
        this.unitPriceCents = Map.copyOf(prices);   // unmodifiable AND copied
        this.version = version; this.promotionCode = code;
    }   // ... accessors unchanged
}
```

Note `Map.copyOf` rather than `new HashMap<>(...)`: the copy is defensive **and** the result
is unmodifiable, so nothing can add to it after the freeze. **That closes Trap 4 at the same
time.**

**Fix 2 — make the publishing field `volatile`.** Belt and braces, and it buys something the
freeze does not: it guarantees that a reader eventually sees the *new* reference rather than
an indefinitely stale one (Topic 86's "not eventually — possibly never").

```java
    private volatile PricingSnapshot current = new PricingSnapshot(Map.of(), 0L, "NONE");
```

**Why both, when either closes the trace?** They answer different questions. `final` fields
guarantee that *if* you see the reference the object is fully built — and say nothing about
whether you see the new reference at all. `volatile` guarantees you see the new reference
promptly — and says nothing about objects published by other routes. The `final` fields
protect the object however it is published, including through code nobody has written yet.
**Defence in depth, and a volatile read on a reference costs close to nothing (Topic 87).**

**Fix 3 — prefer a record.**

```java
public record PricingSnapshot(Map<Long, Long> unitPriceCents, long version, String promotionCode) {
    public PricingSnapshot {
        unitPriceCents = Map.copyOf(unitPriceCents);   // compact constructor: copy + freeze
    }
    public long priceFor(long productId) {
        return unitPriceCents.getOrDefault(productId, 0L);
    }
}
```

Record components are implicitly `final`, so **the freeze cannot be forgotten by the next
person to edit the class.** The compact constructor is where the defensive copy belongs
(Topic 27). This is the version to ship.

**Fix 4 — encode the invariant as a jcstress test and keep it.** So the next refactor cannot
quietly remove a `final`.

**Fix 5 — run the Topic 65 load profile on aarch64** and re-run the incident correlation.
**Do not treat a clean run as proof.** Rule 1.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — double-checked locking without `volatile`

**Wrong approach.** The classic lazy singleton, written from memory:

```java
public class PromotionTableHolder {
    private static PromotionTable instance;              // NOT volatile

    public static PromotionTable getInstance() {
        if (instance == null) {                          // first check, unsynchronized
            synchronized (PromotionTableHolder.class) {
                if (instance == null) {                  // second check
                    instance = new PromotionTable();     // UNSAFE PUBLICATION
                }
            }
        }
        return instance;
    }
}
```

**Exact symptom.** A rare `NullPointerException` or a rare wrong result from a field of
`PromotionTable` that the constructor unconditionally sets. Under load, on a multi-core
machine, on aarch64 much more readily than on x86. Never in a unit test.

**Root cause.** The `synchronized` block serialises *writers*. It does nothing for the
**first check**, which is outside the lock and has no happens-before edge. A second thread can
read a non-null `instance` there and use a `PromotionTable` whose constructor's writes are not
yet visible. The lock prevents two instances; it does not prevent observing a
partially-constructed one.

**Fix — three options, in order of preference:**

```java
// 1. BEST for a static singleton: the holder idiom (safe publication idiom #1).
//    Class initialisation is lazy AND carries a happens-before edge (Topic 67).
public class PromotionTableHolder {
    private static class Holder { static final PromotionTable INSTANCE = new PromotionTable(); }
    public static PromotionTable getInstance() { return Holder.INSTANCE; }
}
// 2. If you genuinely need DCL: the volatile is MANDATORY, not an optimisation.
private volatile PromotionTable instance;
// 3. And make PromotionTable's own fields final regardless. Belt and braces.
```

**The review rule:** *any* double-checked locking without `volatile` on the checked field is
a bug, unconditionally, with no exceptions for "it's only read once" or "it worked in
testing". Topic 87 introduced this; here you can also see the *second* fix — final fields on
the published object — which is why this trap appears in both documents.

---

### Trap 2 — a `final` field holding a mutable object

**Wrong approach.** Believing `final` made the object immutable.

```java
public final class OrderSnapshot {
    private final List<OrderLine> lines;      // final! must be safe, surely
    public OrderSnapshot() { this.lines = new ArrayList<>(); }   // frozen: the REFERENCE
    public void addLine(OrderLine line) { lines.add(line); }     // called from any thread
    public List<OrderLine> lines() { return lines; }
}
```

**Exact symptom.** Readers see the list reference reliably — the `final` did its job — but
observe **missing elements, a wrong `size()`, a `ConcurrentModificationException`, or, in
`ArrayList`'s case, an `ArrayIndexOutOfBoundsException` or a `null` element from a resize
race.** Intermittent, load-dependent, unreproducible.

**Root cause.** The freeze guarantees the field's value at construction end and what it
reaches **as of the freeze** — an empty list. Every element added afterwards is an ordinary
mutation of a shared, non-thread-safe object with no synchronisation. **`final` froze the
reference, not the contents.** Topic 17's shallow-immutability point with a memory-model
consequence.

**Fix.**

```java
// 1. Populate in the constructor and make it genuinely immutable.
public final class OrderSnapshot {
    private final List<OrderLine> lines;
    public OrderSnapshot(List<OrderLine> lines) {
        this.lines = List.copyOf(lines);    // copy AND unmodifiable, before the freeze
    }
    public List<OrderLine> lines() { return lines; }   // safe to return: nobody can mutate it
}
// 2. If it must change over time, replace the whole snapshot behind a volatile
//    reference rather than mutating in place. Copy-on-write at the object level.
private volatile OrderSnapshot snapshot;
// 3. If it must be mutated concurrently, use a concurrent collection - and know that
//    gives per-operation atomicity, not compound atomicity (Topic 92).
```

**`List.copyOf` before the freeze is the whole fix**, one method call. **Review rule:** a
`final` field of a mutable type is only as safe as what you do with the object — check the
constructor copies it and nothing mutates it afterwards.

---

### Trap 3 — publishing `this` from a constructor

**Wrong approach.** Registering the object with something while it is still being built.
Three common shapes, all of which look reasonable:

```java
public class InventoryListener {
    private final long reorderThreshold;
    public InventoryListener(EventBus bus, long threshold) {
        bus.register(this);              // (a) THIS ESCAPES. The bus can dispatch NOW.
        this.reorderThreshold = threshold;
    }
}
public class OrderReaper {
    private final Duration timeout;
    public OrderReaper(Duration timeout) {
        new Thread(this::reap).start();  // (b) THIS ESCAPES via the method reference.
        this.timeout = timeout;          //     The new thread may run before this line.
    }
    private void reap() { /* uses timeout */ }
}
public class PaymentProcessor {
    private final Gateway gateway;
    public PaymentProcessor(Registry registry) {
        registry.add("payments", this);  // (c) THIS ESCAPES into a shared map.
        this.gateway = registry.gateway();
    }
}
```

**Exact symptom.** A `final` field observed as `null` or `0` — which is supposed to be
impossible. In shape (b), a `NullPointerException` on `timeout` inside `reap`, on a field
declared `final` and assigned in the constructor. The stack trace makes no sense and the
first three people who look at it conclude the JVM is broken.

**Root cause.** The freeze happens at the **end** of the constructor. If a reference to
`this` becomes visible to another thread before that point, that thread's reference was
obtained before the freeze and **the final-field guarantee does not apply to it** — the
specification's guarantee is explicitly conditional on the reference not being published
prematurely. **You have opted out of the one mechanism that would have saved you.** Note
subtlety (b): `this::reap` captures `this`, and so do anonymous inner classes, non-static
inner classes, and any lambda touching an instance field (Topic 21).

**Fix — the two-step factory. Construct fully, then publish:**

```java
public final class InventoryListener {
    private final long reorderThreshold;
    private final String warehouseCode;

    private InventoryListener(long threshold, String code) {   // private
        this.reorderThreshold = threshold; this.warehouseCode = code;
    }                                                          // <-- FREEZE happens here

    public static InventoryListener createAndRegister(EventBus bus, long threshold, String code) {
        InventoryListener listener = new InventoryListener(threshold, code);
        bus.register(listener);        // publication strictly AFTER the freeze
        return listener;
    }
}
```

**The review rule, short and catching all three shapes:** *no constructor may pass `this` —
explicitly, or implicitly via a lambda, method reference, inner class, or a started `Thread` —
to anything that outlives the constructor.* If you need registration, use a static factory; in
Spring, `@PostConstruct` serves the same purpose (Topic 37).

---

### Trap 4 — "we made the field `volatile`, so the object is safely published"

**Wrong approach.** Adding `volatile` to the publishing field and concluding the problem is
solved, without touching the published object.

```java
private volatile PricingSnapshot current;   // volatile: good
// ... but PricingSnapshot's fields are still non-final, and something else
//     publishes it through a different route.
```

**Exact symptom.** The original symptom disappears from the one code path you fixed. Then the
same corruption appears on a different path — an object put into a `HashMap` cache, handed to
a `CompletableFuture` continuation with no executor, stored in a `ThreadLocal` copied across
threads, or passed to a logging framework's async appender.

**Root cause.** `volatile` makes *that one field* a safe channel. It says nothing about the
object. **The object is only intrinsically safe to publish if its state is `final`.** Every
new route needs its own edge, and routes multiply as the codebase grows.

**Fix.** Make the object's fields `final` — then it is safe through *every* route, including
ones written next year by someone who has not read this document.

```java
// Safe no matter how it is published. This is why final fields are idiom #3
// and why they are the one to reach for first.
public record PricingSnapshot(Map<Long, Long> prices, long version, String code) {
    public PricingSnapshot { prices = Map.copyOf(prices); }
}
```

**The principle worth carrying:** *`volatile` secures a channel; `final` secures the object.*
Securing the object scales; securing channels does not, because you must remember every time.

---

### Trap 5 — reading a green jcstress run as proof

**Wrong approach.** Writing the test, declaring the bad outcome `FORBIDDEN`, running it,
seeing zero observations, and reporting "verified thread-safe".

**Exact symptom.** Two shapes, both bad:

- The bug ships anyway, because the test never hit the window on your machine — and shows up
  on production hardware with a different core layout.
- The team develops a false model: "we jcstress our concurrent code, so it is correct",
  which quietly replaces the discipline of writing happens-before arguments.

**Root cause.** **jcstress is a falsifier.** It samples an interleaving window and reports
what it saw. Absence of an observation is absence of evidence, not evidence of absence — and
the chance of hitting the window depends on the JDK, core count, core layout (performance
versus efficiency cores on Apple Silicon), other load, and the architecture. **On x86-TSO the
store-store reordering this bug needs is forbidden by the hardware, so a green x86 run is
close to no evidence at all here.**

**Fix — a three-part discipline, all three required:** (1) **write the happens-before argument
in words first** — "Thread B's read of the reference is ordered after Thread A's freeze
because …"; if you cannot complete that sentence the code is not correct regardless of any
test result. (2) **Run jcstress on aarch64**, the machine most likely to falsify you — and on
x86 too if you deploy there, recording which is which. (3) **Report results with their
conditions attached**: "not observed on JDK 25, jcstress `<version>`, aarch64, `-m stress`" is
a reportable result; "verified safe" is a claim the tool cannot support.

**The sentence to use in a design review:** *"The test did not falsify the argument on
aarch64. The reason the code is correct is the freeze at constructor end."* Two sentences,
correctly ordered — evidence, then proof.

---

## Hands-on proof

### Setup

```bash
mkdir -p ~/java-lab/88 && cd ~/java-lab/88
java --version && uname -m && sysctl -n hw.ncpu   # aarch64 = the better instrument here
```

**Record JDK version, jcstress version and `uname -m`. All three are part of every result.**

### Proof 1 — the `final` flag is visible; the barrier is not

```bash
javac -d out com/orderflow/lab/pub/*.java
javap -p -cp out com.orderflow.lab.pub.SafeOrder
javap -p -cp out com.orderflow.lab.pub.UnsafeOrder
javap -c -p -cp out com.orderflow.lab.pub.SafeOrder | sed -n '/<init>/,/^$/p'
```

| What you see | What it means |
|---|---|
| `final` present on `SafeOrder`'s fields, absent on `UnsafeOrder`'s | The `ACC_FINAL` flag. This is the input C2 uses to decide on the freeze barrier |
| The two constructors' bytecode is otherwise the same shape | **The lesson.** The freeze is a JIT-level constraint, not a bytecode instruction |
| No barrier opcode anywhere | Correct. There is no such opcode. Barriers appear in machine code, not bytecode |

### Proof 2 — get a jcstress project

```bash
mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jcstress \
  -DarchetypeArtifactId=jcstress-java-test-archetype \
  -DgroupId=com.orderflow \
  -DartifactId=orderflow-publication \
  -Dversion=1.0

cd orderflow-publication
grep -A2 jcstress pom.xml
```

> **Version note.** I have deliberately not pinned a jcstress version — it and the archetype
> move independently of the JDK, and any version I quote will be stale. Take the current one
> from `github.com/openjdk/jcstress`'s README and pin it in your POM. **Do not copy a version
> number out of a teaching document, this one included.**

### Proof 3 — the R16 test: unsafe publication of an `Order`

`src/main/java/com/orderflow/OrderUnsafePublication.java`:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.Actor;
import org.openjdk.jcstress.annotations.Description;
import org.openjdk.jcstress.annotations.JCStressTest;
import org.openjdk.jcstress.annotations.Outcome;
import org.openjdk.jcstress.annotations.State;
import org.openjdk.jcstress.infra.results.II_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE_INTERESTING;

/**
 * orderflow: an Order published through a PLAIN field, with NON-FINAL fields.
 * Results are (r1, r2) = (totalCents, statusCode) as seen by the reader:
 *   -1, -1  reader saw null; it ran before the publish. Correct, uninteresting.
 *  4599, 7  reader saw a fully-constructed Order. Intended.
 *     0, 0  NON-NULL reference with DEFAULT field values. THE BUG.
 *  4599, 0 and 0, 7  partial visibility. Also the bug, also legal.
 */
@JCStressTest
@Description("Unsafe publication: non-final fields written in a constructor, "
           + "reference stored in a plain field. The reader can see the reference "
           + "before the field writes.")
@Outcome(id = "-1, -1", expect = ACCEPTABLE,
         desc = "Reader saw null. It simply ran first. Correct and uninteresting.")
@Outcome(id = "4599, 7", expect = ACCEPTABLE,
         desc = "Reader saw a fully-constructed Order. The intended outcome.")
@Outcome(id = "0, 0", expect = ACCEPTABLE_INTERESTING,
         desc = "PARTIALLY-CONSTRUCTED ORDER OBSERVED. Non-null reference, both "
              + "fields at their default values. orderflow serves an order worth "
              + "nothing, with a null status.")
@Outcome(id = "4599, 0", expect = ACCEPTABLE_INTERESTING,
         desc = "PARTIAL VISIBILITY. Total arrived, status did not.")
@Outcome(id = "0, 7", expect = ACCEPTABLE_INTERESTING,
         desc = "PARTIAL VISIBILITY. Status arrived, total did not.")
@State
public class OrderUnsafePublication {

    /** The publication channel: a PLAIN field. Not volatile. No lock. */
    Order published;

    /** The published object. Fields are NOT final -> no freeze at constructor end. */
    static class Order {
        int totalCents;      // NOT final
        int statusCode;      // NOT final
        Order() {
            this.totalCents = 4599;
            this.statusCode = 7;      // 7 == PLACED
        }
    }

    @Actor
    public void orderPlacementThread() {
        published = new Order();      // construct, then publish. No barrier between them.
    }

    @Actor
    public void orderReadThread(II_Result r) {
        Order o = published;          // plain read
        if (o == null) {
            r.r1 = -1;
            r.r2 = -1;
        } else {
            r.r1 = o.totalCents;      // may see 0
            r.r2 = o.statusCode;      // may see 0
        }
    }
}
```

And the fixed version, in the same project, so the outcome tables sit side by side:

`src/main/java/com/orderflow/OrderFinalFieldPublication.java`:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.II_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.FORBIDDEN;

/**
 * The same channel - still a PLAIN field - but the Order's fields are FINAL.
 * The freeze at constructor end orders the field writes before any subsequent
 * store, including the store of the reference.
 * Read Honesty Rule 1 before interpreting a FORBIDDEN row with zero samples.
 */
@JCStressTest
@Description("Final fields: the constructor freeze orders the field writes before "
           + "the publishing store. The reader needs no synchronisation.")
@Outcome(id = "-1, -1", expect = ACCEPTABLE,
         desc = "Reader saw null. It ran first. Correct.")
@Outcome(id = "4599, 7", expect = ACCEPTABLE,
         desc = "Fully-constructed Order. The only non-null outcome the "
              + "happens-before argument permits.")
@Outcome(id = "0, 0", expect = FORBIDDEN,
         desc = "Partially-constructed object seen through final fields. If this is "
              + "EVER observed, the freeze argument is wrong or the test is wrong.")
@Outcome(id = "4599, 0", expect = FORBIDDEN, desc = "Partial visibility of final fields.")
@Outcome(id = "0, 7",    expect = FORBIDDEN, desc = "Partial visibility of final fields.")
@State
public class OrderFinalFieldPublication {

    Order published;                  // still a PLAIN field - deliberately

    static class Order {
        final int totalCents;         // FINAL
        final int statusCode;         // FINAL
        Order() {
            this.totalCents = 4599;
            this.statusCode = 7;
        }                             // <-- FREEZE. StoreStore barrier emitted here.
    }

    @Actor
    public void orderPlacementThread() {
        published = new Order();
    }

    @Actor
    public void orderReadThread(II_Result r) {
        Order o = published;
        if (o == null) { r.r1 = -1; r.r2 = -1; }
        else           { r.r1 = o.totalCents; r.r2 = o.statusCode; }
    }
}
```

Read what each part is doing:

| Element | Why it is there |
|---|---|
| `@JCStressTest` | Marks the class for the annotation processor. Without `mvn verify` it never runs |
| `@State` on the class | The class **is** the state; a fresh instance per trial, many thousands of them |
| `II_Result` | Two `int` results, `r1`/`r2`. The `id` strings must match the printed tuple exactly, `", "` separator included |
| Two `@Actor` methods | Two threads. **jcstress schedules them, varying affinity and JIT config; you do not** |
| **No `@Arbiter`** | Deliberate. An arbiter runs *after* both actors with edges from both, so it always sees the settled object — it would hide the bug. **The reading actor must race the writer.** The key design difference from Topic 87's counter test |
| `-1, -1` as `ACCEPTABLE` | The reader legitimately ran first. Undeclared, the run fails as `UNKNOWN` |
| `0, 0` as `ACCEPTABLE_INTERESTING` (unsafe test) | Legal — that is the complaint — and it is what you are hunting, so the report should shout |
| The same ids as `FORBIDDEN` (final-field test) | Declares the correctness claim. **If jcstress observes one, the test fails and your argument was wrong** |
| Both tests use a **plain** publishing field | The *only* variable is `final`. Changing two things proves nothing |

### Proof 4 — build and run

```bash
mvn clean verify                          # NOT `mvn compile` - the processor must run

java -jar target/jcstress.jar -h          # do this once; the CLI has changed across versions
java -jar target/jcstress.jar -l          # list the tests it found

java -jar target/jcstress.jar -t OrderUnsafePublication
java -jar target/jcstress.jar -t OrderFinalFieldPublication

# Confirm against your version with -h before relying on these:
java -jar target/jcstress.jar -t OrderUnsafePublication -m stress   # longest, most thorough
java -jar target/jcstress.jar -t OrderUnsafePublication -m quick    # shortest
java -jar target/jcstress.jar -t Order -v                           # verbose per configuration
java -jar target/jcstress.jar -t Order -r results/                  # HTML report directory
```

### Proof 5 — how to read the outcome table

jcstress prints a table per test configuration. Its **column structure**:

```
RESULT      SAMPLES     FREQ       EXPECT  DESCRIPTION
 -1, -1         <n>      xxx%   ACCEPTABLE  Reader saw null. It ran first.
   0, 0         <n>      xxx%  INTERESTING  PARTIALLY-CONSTRUCTED ORDER OBSERVED.
   0, 7         <n>      xxx%  INTERESTING  PARTIAL VISIBILITY.
4599, 0         <n>      xxx%  INTERESTING  PARTIAL VISIBILITY.
4599, 7         <n>      xxx%   ACCEPTABLE  Fully-constructed Order.
```

*illustration of the format, not captured output*

**I have no JVM, so `<n>` and `xxx%` are placeholders.** I am not going to invent sample
counts. A plausible frequency would teach you to expect a particular rate, and the rate is
exactly the thing that varies by machine, JDK, core layout and architecture.

| Column | What it is |
|---|---|
| `RESULT` | The result tuple, formatted the same way as your `@Outcome` `id`. **If your `id` string does not match this formatting exactly, the outcome shows as `UNKNOWN`** |
| `SAMPLES` | How many times this exact outcome was observed across the whole run |
| `FREQ` | That count as a percentage of all observations. **Not a performance number and not a production probability** |
| `EXPECT` | The grade you assigned. `UNKNOWN` means you did not declare it, and the test fails — correctly |
| `DESCRIPTION` | Your `desc` text echoed back. Write it for a reader who is not you |

### Proof 6 — what your results mean

| What you see | What it means |
|---|---|
| `OrderUnsafePublication`: any of `0, 0`, `0, 7`, `4599, 0` observed | **You have directly observed a partially-constructed object.** Record it with JDK, jcstress version and `uname -m`. This is the result the drill exists to produce |
| `OrderUnsafePublication`: only `-1, -1` and `4599, 7` | **NOT evidence that the code is safe.** The window was not hit. Try `-m stress`, close other applications, and re-read Honesty Rule 1. On x86 this is close to the *expected* result, because TSO forbids the needed reordering |
| `OrderFinalFieldPublication`: only `-1, -1` and `4599, 7`; the `FORBIDDEN` rows have zero samples | **The outcome you want, read correctly: "not observed on this machine, this JDK, this run."** It is failure to falsify the freeze argument. It is a real result and it is **not proof** |
| `OrderFinalFieldPublication`: a `FORBIDDEN` outcome observed even once | **Stop.** In order of likelihood: your test is wrong, your understanding of the freeze is wrong, or you have found something remarkable. Re-read the test first — check the fields really are `final` and that `this` does not escape |
| An outcome you did not declare, marked `UNKNOWN` | The test fails, correctly. Work out how that outcome is reachable **before** declaring it. The thinking is the point of the tool |
| The build errors before any test runs | The annotation processor did not run. Use `mvn clean verify`, and confirm `target/jcstress.jar` exists |
| Wildly different results between two runs on the same machine | **Normal and important.** jcstress frequencies are not stable quantities. Never report one run's `FREQ` as a property of the code |
| Everything green on x86 CI, interesting outcomes on your Mac | **Expected, and it is Rule 2.** The aarch64 result is the stronger evidence. Say so explicitly when you report it |

---

## Failure drill

### The assignment, restated from the master plan

> Unsafe publication of a partially-constructed `Order` via a plain field, verified with
> jcstress.

**What the drill proves:** that a reader can hold a non-null reference to an object whose
fields are still default values, that `final` fields close the hole with no reader-side cost,
and that the evidence for both is conditional on your machine.

### The two rules that govern this drill

Restated because they decide how you write up the result:

1. **A `FORBIDDEN` outcome with zero observations means "not observed on this machine, this
   JDK, this run" — never "proven impossible".** The proof is the happens-before argument.
2. **x86-TSO hides this bug; aarch64 exposes it.** Your Mac is the better instrument. **And
   even on aarch64, never claim the bug "will" reproduce.**

### Step 0 — the control

```bash
java --version && uname -m && sysctl -n hw.ncpu
grep -A2 jcstress pom.xml
```

Write these into your notes file before running anything. They are part of the result.

### Step 1 — write the happens-before argument before you write the test

On paper, in words, both directions. **Do this first. Write the test first and you will
reverse-engineer a justification from whatever the tool prints.**

- **Unsafe version:** "There is no happens-before edge between Actor 1's writes to
  `totalCents`/`statusCode` and Actor 2's reads of them. The publishing write is a plain store
  with no barrier before it. Therefore Actor 2 may observe the reference without observing the
  field writes, and `0, 0`, `0, 7` and `4599, 0` are all legal."
- **Final-field version:** "The constructor writes two `final` fields and completes normally,
  so a freeze occurs for both, implemented as a `StoreStore` barrier. That barrier orders both
  field writes before the subsequent store of the reference. Actor 2 obtains the reference only
  after that store, and it was not published before the freeze, so Actor 2 sees both values.
  Therefore only `-1, -1` and `4599, 7` are legal."

**Those two paragraphs are the deliverable.** Test results are supporting evidence.

### Step 2 — build and run both tests

```bash
mvn clean verify
java -jar target/jcstress.jar -t OrderUnsafePublication      | tee /tmp/88-unsafe.txt
java -jar target/jcstress.jar -t OrderFinalFieldPublication  | tee /tmp/88-final.txt

# If the unsafe test shows nothing interesting, escalate before concluding anything.
java -jar target/jcstress.jar -t OrderUnsafePublication -m stress | tee /tmp/88-unsafe-stress.txt
```

### Step 3 — the volatile-channel variant, for completeness

Add a third test identical to `OrderUnsafePublication` except the publishing field is
`volatile` and the fields stay non-final; declare the partial outcomes `FORBIDDEN`.

| Test | Fields | Channel | What it demonstrates |
|---|---|---|---|
| `OrderUnsafePublication` | non-final | plain | The bug |
| `OrderFinalFieldPublication` | **final** | plain | **The freeze alone closes it** |
| `OrderVolatileChannelPublication` | non-final | **volatile** | The release/acquire edge alone closes it |

**Three tests, one variable each.** That is what makes it an argument rather than an anecdote.

### Step 4 — the `orderflow` version

Take `PricingSnapshot` and `PricingCache` from Example 2 and encode the real invariant.

```java
@JCStressTest
@Description("orderflow PricingCache: a refresh thread publishes a new PricingSnapshot "
           + "through a plain field while a request thread prices a line.")
@Outcome(id = "-1", expect = ACCEPTABLE,         desc = "Reader saw the initial snapshot.")
@Outcome(id = "1999", expect = ACCEPTABLE,       desc = "Reader saw the new price. Intended.")
@Outcome(id = "0", expect = ACCEPTABLE_INTERESTING,
         desc = "PRICED AT ZERO. The reader saw the new snapshot reference with a null or "
              + "empty price map. orderflow sells the product for nothing.")
@State
public class PricingSnapshotPublication {
    // ... mirror the production shape: plain field, non-final snapshot fields,
    //     one actor refreshing, one actor calling priceFor(42).
    //     Catch NPE in the reader and encode it as a distinct result value
    //     rather than letting it escape - an escaping exception is not an outcome.
}
```

**Two design notes:** catch the NPE in the reading actor and map it to a distinct result value
— an exception escaping an actor is not an outcome jcstress can grade. And keep the production
shape; if you simplify the map away you are testing a different program, and the point is that
`orderflow`'s actual class is unsafe. Then fix it — `record` for final components, `Map.copyOf`
in the compact constructor, `volatile` on the cache field — and re-run with the bad outcomes
declared `FORBIDDEN`.

### Step 5 — what to capture

| Artefact | Command | Why |
|---|---|---|
| JDK, jcstress version, `uname -m`, core count | `java --version`, `uname -m`, `sysctl -n hw.ncpu` | **A result without these is not reportable** |
| The two happens-before arguments, in words | your notes | **The actual proof**, written before the tests |
| Outcome tables: unsafe, unsafe `-m stress`, final-field, volatile-channel | `-t <name>` | The bug; failure to falsify; each mechanism isolated |
| Outcome tables, `orderflow` version, before and after | `-t PricingSnapshotPublication` | The production invariant |
| The same runs on x86, if you have access | CI, or a cloud box | **Label which is which.** The contrast is the lesson |
| `javap -p` on the fixed classes | `javap -p` | Proves the `final` flags are actually there |

### Step 6 — how to read it, and how to write it up

Three paragraphs, in this order, and the order is not negotiable:

1. **The argument.** Why the fixed code is correct, in happens-before terms. No test results.
2. **The evidence.** What you ran, on what, and what you observed — "the `FORBIDDEN` outcomes
   had zero samples on JDK 25, jcstress `<version>`, aarch64, `-m stress`" — an observation
   with its conditions attached.
3. **What you are not claiming.** That zero observations is not proof; that an x86 run would
   be weaker evidence; that the frequency you saw is not a production probability.

**Paragraph three is the deliverable.** Anyone can run a tool; reporting its limits correctly
is the skill.

### What the drill proves

A reader can hold a **non-null reference to a partially-constructed object** — not
theoretically; you either observed it or you know exactly why you did not. **`final` fields
close the hole with zero reader-side cost**, which is why they are the first thing to reach
for. And **the evidence is conditional on the machine**, which is why the argument, not the
test, is the proof.

---

## Measurement

### The instruments for this topic

| Question | Instrument | Authority |
|---|---|---|
| Is this class safe to publish? | **The happens-before argument, in words** | **Highest. This is the proof** |
| Can I falsify my argument? | jcstress, on aarch64, with `-m stress` | High as a falsifier; **zero as a prover** |
| Are the fields actually `final`? | `javap -p` | Definitive, and takes five seconds |
| Is the publishing field `volatile`? | `javap -p`, or reading the source | Definitive |
| What barriers did the JIT emit? | `-XX:+PrintAssembly` with `hsdis` | Ground truth, **and hsdis is not bundled with most JDKs** |
| Does the object cost what I think? | JOL | Definitive. `[JAVA 25]` compact headers change it |
| What does the fix cost at runtime? | JMH with `@Threads` (Topic 77) | The only honest answer to "is `final` slow" |
| Did it change the service's p99? | Topic 65 load profile vs `/docs/java/baselines/` | The system-level answer |

### Why a naive `System.nanoTime()` measurement is WRONG here

Two different naive measurements suggest themselves and both are wrong, differently.

**The first: timing a loop to see whether `final` costs anything.**

```java
// WRONG. Name the failures before reading on.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    new SafeOrder(4599, "PLACED");     // result unused
}
System.out.println((System.nanoTime() - start) / 10_000_000 + " ns/op");
```

1. **Dead-code elimination.** The result is unused; C2 may delete the loop entirely.
2. **Escape analysis (Topic 75).** The object never escapes, so it may be scalar-replaced —
   **and a scalar-replaced object has no constructor barrier to measure, because there is no
   object.** You measure the absence of the thing you set out to measure.
3. **No warm-up.** The freeze barrier is emitted by the JIT; the interpreter has none.
4. **Single-threaded.** A `StoreStore` barrier is invisible with one thread and no coherence
   traffic. **The interesting cost, if any, is under contention** — that needs `@Threads`.
5. **Wrong quantity.** The question is "is it correct", which no timing harness answers.

**The second, and worse: a hand-rolled two-thread loop to "see if the bug happens".**

```java
// WRONG, AND DANGEROUSLY SO.
new Thread(() -> { while (true) cache.publish(new UnsafeOrder(4599, "PLACED")); }).start();
new Thread(() -> { while (true) { var o = cache.latest(); if (o != null && o.totalCents() == 0)
                                    System.out.println("BUG"); } }).start();
```

6. **No control over the interleaving.** jcstress deliberately varies affinity, JIT
   configuration and scheduling. Your loop does one thing.
7. **`System.out.println` in the reader is a synchronised call**, inserting a happens-before
   edge that can suppress the very bug you are hunting.
8. **No grading.** jcstress makes you *declare* acceptable outcomes and fails on undeclared
   ones. Your loop silently ignores outcomes you did not think of — the valuable ones.
9. **Not seeing "BUG" will feel like proof.** It is not, for all of Rule 1's reasons.

**The correct instruments: jcstress for correctness, JMH with `@Threads` for cost, and the
happens-before argument for proof.**

### The cost question, answered properly

"Does making fields `final` slow us down?" — the honest answer is that the freeze is a
`StoreStore` barrier at constructor end, which on common architectures is very cheap or free,
and that **you should measure rather than assume** if it is on a path that matters:

```java
@BenchmarkMode(Mode.AverageTime) @OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark) @Fork(3) @Warmup(iterations = 5) @Measurement(iterations = 5)
public class PublicationCostBench {
    @Param({"4599"}) int totalCents;

    @Benchmark @Threads(1)  public Object one()    { return new SafeOrder(totalCents, "PLACED"); }
    @Benchmark @Threads(8)  public Object eight()  { return new SafeOrder(totalCents, "PLACED"); }
}
```

```bash
java -jar target/benchmarks.jar PublicationCostBench -prof gc -f 3 -wi 5 -i 5
```

**Return the object** — deliberately, unlike Topic 75. Here you *want* it to escape, because
a scalar-replaced object has no barrier to measure. **The correct benchmark shape inverts
between the two topics; the benchmark must match the question.**

**The practical answer that ends the discussion:** the cost is one barrier per construction;
the alternative is a data race. **You will not win an argument for the data race on
performance grounds** — and if you construct so many objects that the barrier is measurable,
you have an allocation-rate problem (Topics 68, 75), not a `final` problem.

### What to track in production

There is no metric for "we published something unsafely". Track proxies:

| Signal | Where from | Why |
|---|---|---|
| `NullPointerException` on a field with no setter | Error aggregation, grouped by stack | **The single highest-signal indicator of unsafe publication.** Almost nothing else produces it |
| Errors clustering around a cache refresh or a config reload | Log correlation by timestamp | The window exists only around a publishing write |
| Zero or default values in business data | Data-quality checks on totals, statuses, ids | The business-visible form of the bug |
| Divergence between x86 CI and aarch64 production | Compare error rates by node architecture | **Rule 2, as an operational signal** |

**Put the first row in your error dashboard as a named query.** "NPE on a field the
constructor always assigns" is a signature; once you know it you find these bugs in minutes
rather than months.

---

## Practice exercises

### 1 — Easy: classify twelve publications

Take twelve real publication sites in `orderflow`: a `@Component` field assigned in
`@PostConstruct`; a `static final` table; a `ConcurrentHashMap.put`; a value handed to a
`CompletableFuture`; a field written by a `@Scheduled` refresh; an object put on a
`BlockingQueue`; a listener registered in a constructor; a `ThreadLocal` value; a record
returned from a service; a lazily-initialised singleton; a value passed to a `Thread`
constructor; a getter returning a mutable collection.

For each, produce a row:

| Site | Published how | Which of the five idioms (or none) | Are the object's fields final? | Safe? | Cheapest fix |

**Rules:** every "safe" must name the specific happens-before edge. "Probably fine" is not an
entry — write "unknown, needs analysis" and count how many of those you have.

**Success criterion:** you can name the edge for every site you called safe, and found at
least one where the answer is genuinely "no" or "unknown".

### 2 — Medium: the audit (combines Topics 13, 17, 21, 27, 37, 39, 67, 85, 86, 87)

Audit every class in `orderflow` that is shared across threads.

1. List every class whose instances are reachable from more than one thread.
2. For each, `javap -p` and record which fields are `final` and which are not.
3. For each non-final field, decide: does it need to change after construction? If not,
   **make it final** and note the change.
4. Find every constructor that publishes `this` — explicitly, or via a lambda, method
   reference, inner class, or started `Thread`. Grep is a start:
   ```bash
   grep -rn 'this)' --include=*.java src/main/java | grep -A2 -B2 'public .*(' 
   grep -rn 'new Thread' --include=*.java src/main/java
   grep -rn 'register(this\|add(this\|subscribe(this' --include=*.java src/main/java
   ```
   Convert each to a static factory or a `@PostConstruct`.
5. Find every `final` field of a mutable type. Confirm the constructor copies it defensively
   and nothing mutates it afterwards.
6. Find every double-checked locking site and confirm `volatile`.
7. Produce a table: class, fields made final, `this`-leaks fixed, mutable-final fields
   defended, DCL sites checked — plus **one sentence of happens-before justification per
   class**.

**Success criterion:** every shared class is either fully `final`-fielded or has a written
justification. And you found at least one real bug.

### 3 — Hard: production simulation against the baseline

1. Reproduce the Example 2 `PricingCache` bug as the drill describes, all three variants.
2. Deploy the **broken** version to a load-test environment and run the unchanged Topic 65
   profile for 30 minutes, refreshing the cache every 5 seconds rather than 60 — **widening
   the window without changing the code's shape**.
3. Instrument the read path to detect and **count** the bad state: null map, empty map, a
   zero price for a product known to have one. Count it; do not just log it.
4. Run on **aarch64** and, if you have access, on **x86**. Compare the counts.
5. Fix it — record with `final` components, `Map.copyOf` in the compact constructor,
   `volatile` on the cache field — and re-run everything identically.
6. Measure the fix's cost: JMH `@Threads` on the construction path, plus p50/p95/p99 against
   `/docs/java/baselines/`.
7. Write it up in the three-paragraph form.

**Success criterion:** a count of bad observations in a running service, a side-by-side
architecture comparison, a fix whose cost you measured, and a write-up that refuses to
over-claim. **Zero bad observations on your hardware is also a pass** — provided you say so
plainly, explain why it does not mean the code is safe, and note whether you were on x86.

---

## Interview questions

### Q1 — "What is safe publication?"

**MID-LEVEL.** "Making sure an object is fully constructed before other threads use it —
usually by synchronising or making it volatile."

**SENIOR.** "Publication is making a reference visible to another thread; safe publication
means any thread obtaining it is guaranteed to see the object fully constructed. It needs a
name because the naive assumption fails: without a happens-before edge a thread can read a
non-null reference to an object whose fields still hold their defaults — zero and null. The
reference store and the field stores are independent, so the compiler can reorder them, the
store buffer can drain them out of order, and on a weakly-ordered architecture like aarch64
the hardware permits the reordering to be observed. Reading a non-null reference carries no
information about any other write. Five idioms make publication safe: static initializer,
volatile or `AtomicReference`, `final` field, guarded by a lock, or into a thread-safe
collection. I reach for `final` first, because it is the only one where the reader does
nothing — the freeze means the object is safe through *any* route, including ones written
later by someone who has not thought about this. The other four secure a channel; `final`
secures the object."

**What separates them:** the senior answer says **what goes wrong** rather than what to do,
names the mechanism at three levels, lists all five idioms, and explains **why `final` is
categorically different** — it asks nothing of the reader, so it scales to routes that do not
exist yet.

**Follow-up:** *"Give me the sentence that makes this click."* → **"Reading a non-null
reference tells you nothing about any other write."** People implicitly believe seeing the
reference means the constructor finished. Between two threads with no edge, there is no
"finished" and no "before".

---

### Q2 — "What does `final` actually do, beyond preventing reassignment?"

**MID-LEVEL.** "It stops you reassigning the field, and it's a hint to the JIT that it can
optimise better."

**SENIOR.** "It has memory-model semantics, which is the part people miss. When a constructor
that writes `final` fields completes normally there's a freeze action for those fields —
HotSpot implements it as a `StoreStore` barrier at constructor end, so the constructor's
writes are ordered before any store that follows, including the store publishing the
reference. Any thread obtaining the reference after construction sees the correctly
initialised final fields, and what they reach as of the freeze, **without doing anything on
the reader side.** That's unique: every other edge needs both parties — a volatile write and
a volatile read, a monitor unlock and a lock. Two conditions though. It doesn't apply if
`this` escaped the constructor, because some thread then got the reference before the freeze
— that's why registering a listener or starting a thread in a constructor is a real bug, not
a style issue. And it's shallow: a `final List` guarantees the reference and the list's state
as of the freeze, not anything added afterwards. So it's `final` plus `List.copyOf` in the
constructor."

**What separates them:** the senior answer knows this is **specified behaviour with a
mechanical implementation**, knows the guarantee is **one-sided** — the property that makes it
valuable — and volunteers both conditions under which it fails. "A hint to the JIT" is a wrong
model that leads to treating `final` as optional.

**Follow-up:** *"Does making a field `final` cost anything?"* → A `StoreStore` barrier per
construction, which on common architectures is very cheap or free — measure with JMH at
realistic thread counts if it is on a hot path. But the comparison is against a data race, so
it is not an argument you lose on performance grounds. And if you construct so many objects
that the barrier is measurable, you have an allocation-rate problem, not a `final` problem.

---

### Q3 — "Why does double-checked locking need `volatile`?"

**MID-LEVEL.** "Without volatile the second thread might not see that the first thread set
the instance, so you could get two instances."

**SENIOR.** "That's not the failure — the `synchronized` block prevents two instances fine,
because both writers serialise on the monitor. The failure is the **first check**, which is
deliberately outside the lock and has no happens-before edge with the write inside it. So a
second thread can see a non-null `instance` and return it while the constructor's writes are
not yet visible, and the caller uses a partially-constructed object. `volatile` fixes it: the
write inside the lock is a release, the read at the first check is an acquire. This was the
pre-Java-5 broken idiom and the Java 5 memory model is what made the volatile version
correct. Two practical points: for a static singleton I'd use the holder idiom instead — a
static nested class with a `static final` field, because class initialisation carries an edge
and it's lazy, correct and impossible to get wrong. And I'd make the singleton's own fields
`final`, closing the same hole from the other side and protecting it through every other
publication route too."

**What separates them:** the senior answer correctly identifies that **the bug is the
unsynchronized read, not a lost update**, and that the observable symptom is a
partially-constructed object rather than two instances. That is the distinction that shows
the person understands the memory model rather than having memorised "DCL needs volatile".

**Follow-up:** *"Would making the singleton's fields final let you drop the volatile?"* →
For the *publication* hazard, yes — the freeze orders the constructor writes before the
reference store, so a reader seeing the reference sees the fields. But I would keep the
`volatile` anyway, because without it a thread can read a stale `null` indefinitely and
re-enter the lock, and because relying on the reader-side subtlety makes the code fragile to
someone later adding a non-final field. **Both, and say why in a comment.**

---

### Q4 — "This works on our x86 CI and fails in production on Graviton. Why?"

**MID-LEVEL.** "ARM is a different architecture, so timing is different and the race shows up
more often."

**SENIOR.** "It's not timing, it's the hardware's memory model. x86 is Total Store Order: it
forbids store-store and load-load reordering in hardware, so many publication bugs are simply
unobservable there even though the code is incorrect by the JMM. aarch64 permits both. A
publication bug needs exactly a store-store reordering to be observed — the reference store
visible before the field stores — which x86 forbids and aarch64 permits. So the CI result
isn't 'the code passed', it's 'we ran the experiment on hardware that can't produce the
failure'. That cuts a specific way: the aarch64 failure is the **stronger** evidence, not the
flakier one, and I'd treat the green x86 run as close to no evidence for this bug class.
Practically: write the interleaving out, find the missing edge — most likely a plain
publishing field and non-final fields on the published object — encode the bad outcome as a
jcstress test, run it on aarch64, and push for CI on the architecture we deploy to. One thing
I wouldn't say is that the bug 'will reproduce' on ARM; it's a rare-event bug, and not
observing it there wouldn't mean the code is fine either."

**What separates them:** the senior answer gets the **direction of the evidence** right —
the ARM failure is stronger evidence, not weaker — which is counterintuitive and is what most
teams get backwards. It also refuses to over-claim in the other direction. Timing is a folk
explanation; the memory model is the mechanism.

**Follow-up:** *"Your Mac is Apple Silicon. What does that mean for how you work?"* → It
makes my laptop a better instrument than the CI fleet for this bug class, so I run
publication jcstress tests locally and treat local interesting outcomes as important. It also
means a "works on my machine" report from me is stronger than one from an x86 colleague — the
opposite of the usual assumption.

---

### Q5 — "How would you verify a class is safe to publish?"

**MID-LEVEL.** "Write a test that runs it on multiple threads and check it doesn't fail."

**SENIOR.** "Two things, in a specific order, and the order matters. First the argument: I
write out in words why the code is correct in happens-before terms — 'the constructor writes
final fields and completes, so a freeze occurs; the freeze orders those writes before the
publishing store; the reader obtains the reference only after that store and it wasn't
published before the freeze; therefore the reader sees the initialised values.' If I can't
complete that sentence, the code isn't correct and no test result changes it. Second the
falsification: encode the bad outcomes as a jcstress test, grade them `FORBIDDEN`, run it on
aarch64 because x86-TSO hides this bug class, and report the result with its conditions
attached — JDK version, jcstress version, architecture, mode. What I don't do is call a green
run proof: `FORBIDDEN` with zero samples means 'not observed on this machine, this JDK, this
run'. It's failure to falsify, which is a real result and not the same thing. jcstress is how
I catch my argument being wrong, not how I establish it. A hand-rolled two-thread loop is
strictly worse — no control over interleaving, no outcome grading, and any `println` in the
reader inserts an edge that can suppress the bug you're hunting."

**What separates them:** the senior answer puts **the argument before the test** and states
the epistemics of the tool correctly. Most people invert this and treat a green concurrency
test as verification, which is the single most expensive misunderstanding in concurrent
programming. The `println` detail is the tell that the person has actually tried the naive
approach.

**Follow-up:** *"What if you can't get jcstress running?"* → Then I still have the argument,
which is the proof, and I have `javap -p` to confirm the fields are actually `final` and the
publishing field is actually `volatile` — which catches the majority of real bugs in five
seconds. The tool is valuable; it is not the load-bearing part.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Reading a non-null reference gives **no information about any other write**. Derive from
   that alone why a null check on a shared field is not a safety check, and construct the
   `orderflow` symptom it produces.

2. The freeze happens at the **end** of the constructor. Derive why `bus.register(this)` on a
   constructor's first line destroys the guarantee for a field assigned on its third — and
   name three ways `this` escapes that do not contain the word `this`.

3. Final-field semantics are the only happens-before mechanism where **the reader does
   nothing**. Explain why that makes `final` scale better across a codebase than `volatile`,
   and construct the maintenance scenario where the difference bites.

4. A `final List` field guarantees the reference and the list's state as of the freeze. State
   precisely which subsequent operations fall outside, and give the one-call fix.

5. x86-TSO forbids store-store reordering; aarch64 permits it. **State which way that cuts for
   evidence**, and why an interesting outcome on aarch64 beats a green run on x86 — and why
   neither is proof.

6. A jcstress `FORBIDDEN` outcome with zero samples. **List everything that rules in and
   everything it rules out**, and name the one artefact that would actually establish
   correctness.

7. Your `PricingSnapshot` has final fields but the cache field is a plain reference. Derive
   what is now guaranteed and what is not, and decide whether you would also add `volatile` —
   with the reason, not the reflex.

---

## Quick reference card

### The model

```
PUBLICATION = making a reference visible to another thread.
SAFE PUBLICATION = any thread obtaining it sees a fully-constructed object.

WITHOUT AN EDGE: reader can see a NON-NULL reference with DEFAULT fields (0/null/false).
  Three mechanisms, any one sufficient:
    compiler reordering | store buffer drain order | hardware (aarch64 permits it)
  Plus: absent an edge a read may be stale INDEFINITELY. Not "eventually".

THE FREEZE: constructor writes final fields, completes normally
    => freeze action per final field => HotSpot StoreStore barrier at constructor end
    => any thread obtaining the reference AFTER construction sees them.
       READER DOES NOTHING. Unique in the JMM.
  CONDITIONS: `this` must NOT escape the constructor.
              SHALLOW - reference frozen, later mutation is not.

THE FIVE SAFE PUBLICATION IDIOMS:
  1. static initializer            (class-init edge, Topic 67)
  2. volatile / AtomicReference    (release-acquire, Topic 87)
  3. FINAL FIELD                   (the freeze - PREFER THIS)
  4. guarded by a lock             (unlock-lock edge, Topic 85)
  5. into a thread-safe collection (its internal edges, Topic 92)

volatile secures a CHANNEL.  final secures the OBJECT.
```

### The two honesty rules

```
1. FORBIDDEN with zero samples = "not observed on this machine, this JDK, this run".
   NOT "proven impossible". The proof is the happens-before argument.
   jcstress is a FALSIFIER, never a prover.
2. x86 is Total Store Order and HIDES this bug class. aarch64 EXPOSES it.
   => your Apple Silicon Mac is the BETTER instrument
   => a green x86 CI run is close to NO evidence here
   => and even on aarch64: NEVER claim the bug "will" reproduce.
```

### jcstress

```bash
mvn archetype:generate -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jcstress \
  -DarchetypeArtifactId=jcstress-java-test-archetype \
  -DgroupId=com.orderflow -DartifactId=orderflow-publication -Dversion=1.0
# Pin the CURRENT version from github.com/openjdk/jcstress README, not from here.
mvn clean verify                      # NOT `mvn compile` - the processor must run
java -jar target/jcstress.jar -h      # once; the CLI has changed across versions
java -jar target/jcstress.jar -l      # list discovered tests
java -jar target/jcstress.jar -t OrderUnsafePublication
java -jar target/jcstress.jar -t OrderUnsafePublication -m stress   # confirm with -h
java -jar target/jcstress.jar -t Order -r results/                  # HTML report
```

| Annotation | Role |
|---|---|
| `@JCStressTest` | Marks the class for the annotation processor |
| `@State` | The class **is** the state; a fresh instance per trial |
| `@Actor` | One thread. jcstress schedules them, not you |
| `@Arbiter` | Runs after both actors with edges from both. **Omit it for publication tests** — it would hide the bug |
| `@Outcome(id, expect, desc)` | `id` must match the printed tuple format **exactly** |
| `ACCEPTABLE` / `ACCEPTABLE_INTERESTING` / `FORBIDDEN` / `UNKNOWN` | Undeclared outcomes fail the run — correctly |

### Diagnostic commands

```bash
# Are the fields actually final? Is the publishing field volatile? Five seconds.
javap -p -cp target/classes com.orderflow.pricing.PricingSnapshot
javap -p -cp target/classes com.orderflow.pricing.PricingCache
# Find this-escapes in constructors.
grep -rn 'new Thread\|register(this\|add(this\|subscribe(this' --include=*.java src/main/java
# Find double-checked locking without volatile.
grep -rn -B3 'synchronized' --include=*.java src/main/java | grep -A3 'if (.* == null)'
# The conditions that are part of every result.
java --version && uname -m && sysctl -n hw.ncpu
# Object size. [JAVA 25] compact headers change it; semantics are unchanged.
java -cp jol-cli.jar org.openjdk.jol.Main internals com.orderflow.orders.Order
# Cost of the fix, at realistic thread counts. Return the object - you WANT it to escape.
java -jar target/benchmarks.jar PublicationCostBench -prof gc -f 3
```

### Gotchas checklist

- [ ] A non-null reference tells you **nothing** about any other write.
- [ ] `final` on the fields; `volatile` on the channel. Prefer `final`; often do both.
- [ ] No constructor publishes `this` — via lambda, method reference, inner class or `Thread`.
- [ ] `final` is **shallow**: `List.copyOf` / `Map.copyOf` in the constructor.
- [ ] Records give you final components for free and cannot be forgotten. Use them.
- [ ] DCL without `volatile` is a bug, unconditionally.
- [ ] No `@Arbiter` in a publication test — it hides the bug.
- [ ] Zero `FORBIDDEN` samples is **not proof**. Write the happens-before argument.
- [ ] Run on aarch64; label every result with JDK, jcstress version and architecture — and
      never say the bug "will" reproduce.
- [ ] `[JAVA 25]` compact headers change sizes, not semantics. Settle with JOL.

---

## When would I use this at work?

**1. Reviewing any class shared between threads — which is more of them than people think.**

You have a three-second check that catches an entire bug class: *are all the fields `final`,
and does the constructor leak `this`?* `javap -p` answers the first, a glance at the
constructor the second. Because the fix — `final` fields, or converting the class to a record
— is nearly free with no downside, this is one of the few review comments that is never
contentious. **Over a year it prevents a category of production bug that is close to
impossible to diagnose after the fact**, because its symptoms point everywhere but the cause.

**2. Diagnosing a `NullPointerException` on a field that cannot be null.**

The signature — an NPE on a field the constructor unconditionally assigns, no setter,
clustering around a cache refresh or config reload, commoner on ARM nodes than x86 — is now
something you recognise on sight. Everyone else is hunting a code path that sets the field to
null, and there isn't one. **Going straight to "is this object safely published?" turns a
multi-week mystery into a ten-minute fix.**

**3. Setting the team's standard for concurrent code.**

Two norms that outlast you. First: **every shared class carries a one-line happens-before
justification in a comment**; if it cannot be written, the class is not ready. Second:
**concurrency claims are reported with their conditions** — "not observed on JDK 25, jcstress
`<version>`, aarch64" rather than "verified thread-safe". The second changes how the team
reads every concurrency result it ever gets, and costs nothing to introduce. **Most teams
argue thread safety on intuition; you can make them argue on evidence, while being clear
about what the evidence does and does not support.**

---

## Connected topics

**Prerequisites:**

- **13 — equals/hashCode**: a mutable key silently corrupts a `HashMap`; an unsafely-published
  object silently corrupts everything downstream. Same shape — no exception, wrong data.
- **17 — Immutability, `final`, safe publication**: **this document is Topic 17's promised
  payoff.** Topic 17 said `final` has memory-model meaning; this is that meaning in full.
- **21 — Lambdas and capture**: `this::method` captures `this`, as does a non-static inner
  class. That is how `this` escapes a constructor without the word appearing in the diff.
- **27 — Records and shallow immutability**: components are implicitly `final`, so a record
  gets the freeze for free and cannot lose it by accident. The surviving trap is a record
  holding a mutable collection — `Map.copyOf` in the compact constructor.
- **37 — Bean lifecycle**: `@PostConstruct` is Spring's two-step factory; the bean is fully
  constructed before it runs.
- **39 — Constructor injection**: makes fields `final`, giving safe publication for free.
- **65 — The load baseline**: where the window is wide enough for the bug to appear.
- **67 — Class loading**: why a static initializer is a safe publication idiom — class
  initialisation holds a lock and carries an edge.
- **74–75 — JIT**: the compiler's licence to reorder is one of the three mechanisms, and the
  freeze barrier is emitted into machine code, which is why bytecode cannot show it to you.
- **85 — `synchronized` and monitors**: idiom 4, and the DCL trap's partial fix.
- **86 — The JMM I**: happens-before, the edge list, and "not eventually — possibly never".
  The final-field freeze is one of those edges; this document is its full treatment.
- **87 — `volatile` and barriers**: idiom 2 and the `StoreStore`/`StoreLoad` vocabulary, plus
  the DCL trap from the channel side. **This document is that trap from the object side.**

**This unlocks:**

- **92 — Concurrent collections**: idiom 5. A `ConcurrentHashMap.put` safely publishes — a
  documented guarantee, not a side effect. Plus the reminder that atomicity does not compose.
- **94 — Explicit locks**: AQS's `volatile int state` is how `ReentrantLock` gets its edge.
  Same mechanism as idiom 2, different ergonomics.
- **99 — jcstress**: the full treatment of the tool used here, including its own statement of
  the two honesty rules.
- **101 — Virtual threads**: mount and unmount carry happens-before edges, so a virtual
  thread's writes survive carrier migration. Publication semantics are unchanged; what changes
  is how many threads can race on one object.
- **122 — Docker and startup**: config reloads and cache warm-ups are exactly where
  publication windows live, and where the incidents cluster.
- **129 — Capacity and cost**: an unsafe publication that corrupts business data is a
  correctness incident with a financial number attached — the argument for the five-second
  review check.

---

*Java baseline 21, running on JDK 25. Three things here are deliberately hedged rather than
asserted: the exact machine instructions HotSpot emits for the freeze on your aarch64 build
(settle it with `-XX:+PrintAssembly` and `hsdis`, not bundled with most JDKs), the current
jcstress version and CLI options (take them from github.com/openjdk/jcstress, not from here),
and whether any test here will surface its interesting outcome on your machine. **No sample
count, frequency, timing or percentile in this document was measured — I have no JVM.** The
outcome table shown is a column-format illustration with `<n>` placeholders, labelled as such.
Two rules govern every experiment here: a `FORBIDDEN` outcome with zero observations means
"not observed on this machine, this JDK, this run", never "proven impossible" — the proof is
the happens-before argument, and jcstress is how you catch that argument being wrong; and
x86's Total Store Order hides the reordering this bug class needs, which aarch64 permits,
making your Apple Silicon Mac the better instrument and a green x86 CI run close to no
evidence — while never licensing the claim that the bug "will" reproduce anywhere. What has
been stable since Java 5 and will still be true at 2am: the end of a constructor with final
fields carries a freeze action, so any thread obtaining the reference afterwards sees
correctly-initialised final fields with no synchronisation; non-final fields carry no such
guarantee; and reading a non-null reference tells you nothing whatsoever about any other
write.*
