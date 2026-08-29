# 17 — Immutability: `final` semantics, defensive copying, safe publication

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Imagine a shelf with a label on it.

The **label** says which box to go and look in. The **box** has stuff inside it.

`final` glues the *label* down. Nobody can peel it off and point it at a different
box. That is all it does.

It does **not** lock the box. Anyone who follows the label can walk to the box, open
it, and take things out or put things in.

So "I made the field `final`, so my object is immutable" is exactly as wrong as
"I glued the label down, so nobody can steal the contents."

There is a second thing, and it is stranger.

Imagine two people building and reading the shelf at the same time. Person A is still
putting things into the box while Person B walks up and reads the label. In a
single-worker warehouse this cannot happen — one person does one thing at a time. In
a **multi-worker** warehouse it happens constantly, and B can read a label that points
at a box that is only half-filled.

Java has a rule about this: if every slot in the box is `final`, then the moment
anyone can see the label, the box is guaranteed complete. That rule is called the
**final-field freeze**, and it is the single most useful thing `final` does.

---

## The bridge from what you know

### `readonly` / `as const` / `Object.freeze` — **PARTIAL analogue**

You have three tools in TypeScript/JavaScript that look like `final`. None of them is
the same thing. Here is the honest mapping.

```ts
class Order {
  readonly id: string;                 // compile-time only
  readonly lines: OrderLine[] = [];    // the ARRAY is still mutable
}

const status = "PAID" as const;        // compile-time only, a literal type

const frozen = Object.freeze({ id: "A" });   // runtime, shallow
frozen.id = "B";                             // silently ignored (or throws in strict mode)
```

| Your tool | What it actually does | `final` compared to it |
|---|---|---|
| `readonly` | A **compile-time** check. It is **erased**. The emitted JavaScript has no trace of it. A plain JS caller, a cast, or `JSON.parse` output can write the field and nothing complains. | `final` is enforced by `javac` **and** recorded in the class file as an `ACC_FINAL` flag that the JVM itself honours. It is not erased. |
| `as const` | Narrows a *type*. Nothing to do with mutation at runtime. | No relation. Java's nearest equivalent is a `static final` constant, which the compiler may inline into every call site. |
| `Object.freeze` | A **runtime**, **shallow** freeze of one object's own properties. Closest in spirit — but it freezes the *target*, which `final` never does. | `final` freezes the *reference*, `Object.freeze` freezes the *target*. They are opposites, and neither is deep. |

**What transfers:** the instinct that "immutable" is a design goal worth having, and
the knowledge that all three of your tools are shallow. That last part transfers
perfectly — Java's shallowness will not surprise you.

**What you must unlearn:** `readonly` being purely a lint-level nicety. In Java, a
`final` field is a *runtime* fact that the JIT compiler is allowed to optimise
against, and that other threads depend on for correctness.

### Safe publication — **NO TYPESCRIPT ANALOGUE**

I want to be literal about this, because a wrong analogy here costs more than no
analogy at all.

**There is no TypeScript or JavaScript equivalent of safe publication. None. The
concept cannot be expressed in a single-threaded runtime.**

Here is why, mechanically.

Your runtime has **run-to-completion** semantics. When a function starts, it runs to
its end (or to an `await`) before any other JavaScript executes. There is exactly one
thread touching the heap. So a partially-constructed object is a thing that only
*your own code* can observe, and only if you deliberately hand out `this` from inside
a constructor and then look at it. Nobody else exists to look.

In Java, threads are preempted by the operating system between any two bytecodes.
Thread A can be halfway through a constructor when the OS suspends it and lets Thread
B run. If Thread B can reach that object, B can read fields that are still zero and
null.

Worse: even without preemption, the compiler, the JIT and the CPU are all permitted to
**reorder** the writes. The write that publishes the reference can become visible
before the writes that filled in the fields. This is legal and it happens on real
hardware — most readily on ARM chips, which includes Apple Silicon.

"Safe publication" is the set of techniques that make that impossible. `final` fields
are one of them, and the cheapest one.

> Full treatment: **Topic 86** (happens-before), **Topic 87** (`volatile`), and
> **Topic 88** (`final` fields and safe publication, with `jcstress` proof). This
> topic gives you the rule and the reason. Topic 88 gives you the machine.

### `SharedArrayBuffer` — the one place you *have* met this

If you have used `SharedArrayBuffer` with `Atomics` in Node worker threads, you have
met a real memory model with real visibility rules. That is an **HONEST ANALOGUE**,
and it is the only one. If you have not used it, you have never had a data race in
your life, and you should assume your intuition about "obviously the object is built
by then" is worthless in Java.

---

## What is this?

There are **three separate levels** of immutability in Java, and people mix them up
constantly. Name them separately and the whole topic becomes simple.

### Level 1 — reference immutability (`final`)

The variable cannot be pointed at a different object.

```java
final List<String> skus = new ArrayList<>();
skus = new ArrayList<>();   // COMPILE ERROR: cannot assign a value to final variable
skus.add("SKU-4471");       // completely fine. The list changed.
```

### Level 2 — shallow immutability

Every field is `final`, and no method changes any field. But a field may *point at*
something mutable, and that thing can still change.

A `record` gives you exactly this and no more. (**Topic 27**.)

### Level 3 — deep immutability

Every field is `final`, and every field's *value* is itself immutable — either a
primitive, a `String`, another deeply-immutable object, or a **copy** of a mutable
thing that nobody else holds a reference to.

This is the level you actually want for a domain value like an `Order`. Getting from
Level 2 to Level 3 is what **defensive copying** is for.

### And the fourth, orthogonal thing — safe publication

Even a deeply-immutable object can be *observed broken* by another thread if the
reference to it is handed over incorrectly. Immutability and safe publication are
different properties. You need both.

The good news: for a class where **all** fields are `final` and `this` never escapes
the constructor, the Java Memory Model gives you safe publication for free, with no
locks, no `volatile`, no cost.

---

## Why does it matter?

Four things, in the order you will meet them.

**1. Your "immutable" class is not immutable, and a caller corrupts it.**
This is the number-one bug of this topic. You write an `Order` with `final` fields,
you return `this.lines` from a getter, and a caller three layers away calls
`order.getLines().clear()`. Your stored total no longer matches your lines. No
exception, no log line. You find it from a reconciliation report.

**2. A `HashMap` or `HashSet` silently loses entries.**
If an object's fields can change after it was used as a key, its `hashCode` changes,
and lookup goes to the wrong bucket. The entry is unreachable but still counted and
still retained. That is **Topic 13**, and immutability is the structural fix for it.

**3. Thread safety, for free or not at all.**
An immutable object needs no locks, no `volatile`, no `synchronized`. It cannot be in
an inconsistent state because it has no state transitions. This is the cheapest
concurrency strategy that exists, and it is the reason the whole industry moved
towards value objects. The moment your object is mutable, every thread that touches it
is a design problem (Phase 9).

**4. Another thread sees a half-built object.**
Rare, intermittent, load-dependent, hardware-dependent, and effectively undebuggable
from logs. `final` fields eliminate a whole class of these for free. This is the
argument for constructor injection in Spring (**Topic 39**) that most people repeat
without understanding.

---

## Syntax breakdown

### `final` in its four positions

```java
public final class Money {                     // 1. final CLASS: cannot be subclassed

    private final long amountMinor;            // 2. final FIELD: assign once, in the
    private final String currency;             //    declaration or in the constructor

    public Money(long amountMinor, String currency) {
        this.amountMinor = amountMinor;
        this.currency = currency;
    }

    public final Money plus(Money other) {     // 3. final METHOD: cannot be overridden
        final long total = this.amountMinor    // 4. final LOCAL: cannot be reassigned
                         + other.amountMinor;
        return new Money(total, currency);
    }
}
```

| Position | What it forbids | Why you would use it |
|---|---|---|
| `final class` | subclassing | Stops someone breaking your invariants by overriding a method. `String`, `Integer` and `LocalDate` are all `final` for this reason. |
| `final` field | reassignment after construction | The important one. Also triggers the memory-model freeze. |
| `final` method | overriding | Protects a method that the constructor or another final method depends on. |
| `final` local / parameter | reassignment inside the method | Readability, and it is *required* for a variable captured by a lambda or anonymous class — see below. |

### Blank finals

A `final` field does not have to be assigned at its declaration. It must be assigned
**exactly once** by the time every constructor finishes. The compiler tracks this
precisely.

```java
private final String currency;

public Money(long amountMinor) {
    if (amountMinor < 0) {
        this.currency = "GBP";       // assigned on this path
    } else {
        this.currency = "GBP";       // and on this one
    }
    // this.currency = "USD";        // COMPILE ERROR: might already have been assigned
}
```

### `final` vs "effectively final"

A lambda or an anonymous class may only capture a local variable that is **never
reassigned**. It does not have to carry the `final` keyword — the compiler works out
whether it *could* have. That is "effectively final".

```java
String sku = order.sku();                 // never reassigned -> effectively final
Runnable task = () -> log.info(sku);      // legal

String status = "NEW";
status = "PAID";                          // reassigned
Runnable bad = () -> log.info(status);    // COMPILE ERROR: must be final or effectively final
```

This is not the same as your JavaScript closures, which capture the *variable* and see
later writes to it. Java captures the *value* at capture time, which is exactly why it
insists the variable never change — so the two models can never disagree.

### The defensive-copy toolkit

```java
List.copyOf(source)                // -> a genuinely immutable copy. Rejects nulls.
Set.copyOf(source)
Map.copyOf(source)
List.of("A", "B")                  // -> an immutable list built from values

new ArrayList<>(source)            // -> a mutable copy. Sometimes what you want internally.
Collections.unmodifiableList(x)    // -> an unmodifiable VIEW of x. NOT a copy. See Trap 3.

Arrays.copyOf(array, array.length) // arrays have no immutable form; you must copy
source.clone()                     // for arrays this is fine and idiomatic

Instant / LocalDate / LocalDateTime      // immutable. Use these.
java.util.Date / java.util.Calendar      // MUTABLE. Legacy. Never expose one.
BigDecimal / String / all 8 wrappers     // immutable
```

The one you must internalise:

> **`List.copyOf(x)` gives you a snapshot. `Collections.unmodifiableList(x)` gives you
> a read-only window onto a list that someone else can still change.**

---

## Example 1 — minimal

A shipping address that thinks it is immutable.

```java
import java.util.List;

public final class ShippingAddress {

    private final String postcode;
    private final List<String> lines;

    public ShippingAddress(String postcode, List<String> lines) {
        this.postcode = postcode;
        this.lines = lines;              // <-- stores the caller's list
    }

    public String postcode()      { return postcode; }
    public List<String> lines()   { return lines; }   // <-- hands it back out
}
```

```java
public class AddressLeak {
    public static void main(String[] args) {
        var mutableLines = new java.util.ArrayList<>(List.of("12 Mill Road", "Cambridge"));

        var address = new ShippingAddress("CB1 2AB", mutableLines);
        System.out.println("before: " + address.lines());

        mutableLines.add("SURPRISE");        // attack 1: through the reference we kept
        address.lines().add("ALSO THIS");    // attack 2: through the getter

        System.out.println("after : " + address.lines());
    }
}
```

Run it. The class is `final`. Every field is `final`. There is no setter anywhere.
And the address changed twice.

The fix is two copies — one **in**, one **out**:

```java
    public ShippingAddress(String postcode, List<String> lines) {
        this.postcode = java.util.Objects.requireNonNull(postcode);
        this.lines = List.copyOf(lines);     // copy IN: we now own our data
    }

    public List<String> lines() { return lines; }   // already immutable; safe to return
```

With `List.copyOf`, the copy-out becomes unnecessary — that is the whole point of
holding an immutable field. If you had stored a mutable `ArrayList` internally for
performance, you would need `List.copyOf(this.lines)` on the way out as well.

---

## Example 2 — production scenario

`orderflow` needs an `Order` aggregate that is safe to put in a cache, safe to use as
a map key, safe to hand to a metrics thread, and safe to log.

### The version that ships and then corrupts data

```java
package com.orderflow.orders;

import java.math.BigDecimal;
import java.util.Date;
import java.util.List;

public final class Order {

    private final long orderId;
    private final long customerId;
    private final List<OrderLine> lines;
    private final BigDecimal total;
    private final Date placedAt;
    private final String[] appliedPromotionCodes;

    public Order(long orderId, long customerId, List<OrderLine> lines,
                 Date placedAt, String[] appliedPromotionCodes) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.lines = lines;                                // leak 1
        this.placedAt = placedAt;                          // leak 2 (Date is mutable)
        this.appliedPromotionCodes = appliedPromotionCodes; // leak 3 (arrays are mutable)
        this.total = lines.stream()
                          .map(OrderLine::lineTotal)
                          .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public List<OrderLine> lines()        { return lines; }                  // leak 4
    public Date placedAt()                { return placedAt; }               // leak 5
    public String[] promotionCodes()      { return appliedPromotionCodes; }  // leak 6
    public BigDecimal total()             { return total; }                  // safe: BigDecimal is immutable
}
```

Six leaks. Every field is `final`. The class is `final`. It looks immutable in review.

What goes wrong in production: the fulfilment service calls
`order.lines().removeIf(line -> line.quantity() == 0)` to tidy up before printing a
picking slip. It has now removed lines from an order that is sitting in the order
cache with a `total` computed from the lines that used to be there. The invoice
total no longer matches the sum of the invoice lines. Finance notices four days later.

### The corrected version

```java
package com.orderflow.orders;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.Objects;

public final class Order {

    private final long orderId;
    private final long customerId;
    private final List<OrderLine> lines;      // deeply immutable: copyOf + OrderLine is immutable
    private final BigDecimal total;           // immutable type
    private final Instant placedAt;           // immutable type
    private final List<String> promotionCodes;  // List, not array — arrays cannot be immutable

    public Order(long orderId,
                 long customerId,
                 List<OrderLine> lines,
                 Instant placedAt,
                 List<String> promotionCodes) {

        if (orderId <= 0)     throw new IllegalArgumentException("orderId must be positive");
        if (lines.isEmpty())  throw new IllegalArgumentException("an order must have at least one line");

        this.orderId        = orderId;
        this.customerId     = customerId;
        this.lines          = List.copyOf(lines);            // copy IN
        this.placedAt       = Objects.requireNonNull(placedAt, "placedAt");
        this.promotionCodes = List.copyOf(promotionCodes);   // copy IN

        // computed AFTER the copy, so it can never disagree with our own lines
        this.total = this.lines.stream()
                               .map(OrderLine::lineTotal)
                               .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public long orderId()               { return orderId; }
    public long customerId()            { return customerId; }
    public List<OrderLine> lines()      { return lines; }            // already immutable
    public Instant placedAt()           { return placedAt; }         // Instant is immutable
    public List<String> promotionCodes(){ return promotionCodes; }
    public BigDecimal total()           { return total; }

    /** Change is expressed as a NEW object, never as a mutation. */
    public Order withExtraLine(OrderLine extra) {
        var newLines = new java.util.ArrayList<>(this.lines);
        newLines.add(extra);
        return new Order(orderId, customerId, newLines, placedAt, promotionCodes);
    }
}
```

Six specific decisions, each removing a specific hazard:

1. `List.copyOf` in the constructor — nobody else holds a reference to our list.
2. `Instant` instead of `java.util.Date` — `Date` has `setTime()`. It should never
   appear in a new codebase.
3. `List<String>` instead of `String[]` — an array can never be made immutable, only
   copied on every access. Using a `List` removes the problem instead of managing it.
4. `total` computed from `this.lines` (the copy), not from the parameter.
5. Validation in the constructor, so an invalid `Order` cannot exist at all.
6. `withExtraLine` returns a new `Order`. This is what "immutable" means as a
   *behaviour*, not just as a field modifier.

And the free bonus: because **every** field is `final` and `this` never escapes the
constructor, this object is **safely published**. You can hand it to another thread
through a plain field with no synchronisation and that thread is guaranteed to see all
six fields correctly initialised.

> **Topic 27** shows how a `record` writes most of this for you — but a record gives
> you Level 2 (shallow) immutability only. The `List.copyOf` calls still have to go in
> the record's compact constructor by hand. Records do not make this topic go away.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the leaked internal collection

**Wrong:**
```java
private final List<OrderLine> lines;
public List<OrderLine> getLines() { return lines; }
```

**Exact symptom:** no exception. Ever. What you observe instead is a *business*
inconsistency: `order.total()` does not equal the sum of `order.lines()`; an order
that had 3 lines when it was cached has 2 lines when it is invoiced; a
`Collections.sort(order.getLines())` call somewhere reorders lines permanently in the
cache. In tests it passes, because tests construct an order and read it once.

**Root cause:** `final` protects the field, not the object it points at. Returning the
reference hands a caller full write access to your internals.

**Fix:** copy in the constructor (`List.copyOf`), and if you must hold a mutable list
internally, copy again in the getter. Never return the live reference.

**How to prove you fixed it:** write a test that calls the getter and tries to mutate.
It should throw `UnsupportedOperationException`.

```java
@Test
void linesCannotBeMutatedByCallers() {
    Order order = anOrderWithTwoLines();
    assertThrows(UnsupportedOperationException.class,
                 () -> order.lines().add(anotherLine()));
}
```

---

### Trap 2 — `final` on a field whose type is mutable

**Wrong:**
```java
public class PaymentAttempt {
    private final Map<String, String> gatewayMetadata = new HashMap<>();

    public Map<String, String> metadata() { return gatewayMetadata; }
}
```

**Exact symptom:** two threads writing to `gatewayMetadata` concurrently produce, in
increasing order of nastiness: a lost entry; a `ConcurrentModificationException`
thrown from a *reader* iterating the map; or, on a `HashMap` resize, an infinite loop
that pins a CPU core at 100% with a thread stuck in `HashMap.getEntry` in the thread
dump. That last one is a real and famous production failure mode.

**Root cause:** `final` said "this reference never changes". It said nothing about the
map's contents, and `HashMap` is not thread-safe.

**Fix:** `this.gatewayMetadata = Map.copyOf(metadata);` at construction. If the map
genuinely must be mutated after construction, this class is not immutable — say so,
and use `ConcurrentHashMap` and a lock discipline instead of pretending (Phase 9).

---

### Trap 3 — `Collections.unmodifiableList` on a list you do not own

**Wrong:**
```java
public Order(List<OrderLine> lines) {
    this.lines = Collections.unmodifiableList(lines);   // a VIEW, not a COPY
}
```

**Exact symptom:** `order.lines().add(x)` correctly throws
`UnsupportedOperationException` — so your test passes and you believe you are safe.
Meanwhile the caller who *passed in* the list still holds it, calls
`theirList.add(x)`, and `order.lines()` now contains the new element. The order
changed through a door you thought you had locked.

**Root cause:** `Collections.unmodifiableList` wraps the *live* list. It removes the
mutating methods from your view; it does not remove them from anyone else's.

**Fix:** `List.copyOf(lines)`, which snapshots. Or, if you are on a codebase that
predates Java 10, `Collections.unmodifiableList(new ArrayList<>(lines))` — copy first,
*then* wrap.

> `List.copyOf` also rejects `null` elements, which `new ArrayList<>(x)` does not. That
> is usually what you want in a domain object, but it is a behaviour change worth
> knowing about when you migrate old code.

---

### Trap 4 — `this` escaping the constructor

**Wrong:**
```java
public final class InventoryWatcher implements PriceListener {

    private final long productId;
    private final BigDecimal threshold;

    public InventoryWatcher(long productId, BigDecimal threshold, PriceBus bus) {
        bus.register(this);              // <-- 'this' escapes BEFORE construction ends
        this.productId = productId;
        this.threshold = threshold;
    }

    @Override public void onPrice(BigDecimal price) {
        if (price.compareTo(threshold) < 0) { /* threshold may still be null here */ }
    }
}
```

**Exact symptom:** an intermittent `NullPointerException` inside `onPrice`, on a field
that is `final` and assigned in the constructor and could not possibly be null. It
happens under load, roughly never in tests, and disappears when you add a log line.
On x86 it may be rare. On ARM (Apple Silicon, Graviton) it is much more likely.

**Root cause:** the final-field freeze happens at the **end** of the constructor.
Publishing `this` before that point voids the guarantee entirely. Another thread —
here, a `PriceBus` dispatch thread — can invoke `onPrice` on a half-built object.

**Fix:** never let `this` leave the constructor. Register afterwards, from a factory
method:

```java
public static InventoryWatcher startWatching(long productId, BigDecimal threshold, PriceBus bus) {
    InventoryWatcher watcher = new InventoryWatcher(productId, threshold);  // fully built
    bus.register(watcher);                                                  // then published
    return watcher;
}
```

The subtle version of this trap: starting a thread from a constructor, or calling any
overridable method from a constructor (a subclass override runs before the subclass's
own fields are assigned). Both are the same defect wearing a different hat.

---

### Trap 5 — publishing a mutable object through a plain field

**Wrong:**
```java
public class PricingConfigHolder {
    private static PricingConfig current;             // not final, not volatile

    public static void reload(PricingConfig fresh) { current = fresh; }
    public static PricingConfig current()           { return current; }
}
```

**Exact symptom:** two shapes, both awful.
- A reader thread returns `null` from `current()` long after `reload` completed on
  another thread — sometimes *forever*, because the JIT is allowed to hoist the read
  out of a loop and never re-read it.
- A reader gets a non-null `PricingConfig` whose own fields read as `null`/`0`.

**Root cause:** a plain field write establishes no happens-before edge. There is no
guarantee that any other thread ever sees it, and no ordering between the writes that
built `fresh` and the write that published it. "Eventually" is not in the
specification.

**Fix:** make the holder field `volatile`, and make `PricingConfig` deeply immutable
with all-`final` fields.

```java
private static volatile PricingConfig current;
```

`volatile` gives the ordering for the *reference*. All-`final` fields give the freeze
for the *contents*. You want both. This is **Topics 86–88**, and it is the reason
those topics exist.

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM, so I am not going to
print output and call it real. What I can give you exactly is the command, what to
look for, and how to read every outcome you might get.

### Setup

```bash
mkdir -p ~/java-lab/17 && cd ~/java-lab/17
java --version      # expect 21 or 25
```

### Proof 1 — `final` does not protect the target

`FinalIsShallow.java`:
```java
import java.util.*;

public class FinalIsShallow {
    public static void main(String[] args) {
        final List<String> skus = new ArrayList<>(List.of("SKU-1001"));

        skus.add("SKU-4471");            // legal
        skus.set(0, "REPLACED");         // legal
        skus.clear();                    // legal
        // skus = new ArrayList<>();     // uncomment: compile error

        System.out.println("mutations through a final reference: " + skus);
    }
}
```

```bash
java FinalIsShallow.java
```

**What to look for:** the program runs and prints an empty list.

| What you see | What it means |
|---|---|
| It compiles and prints `[]` | Correct. `final` permitted three mutations and forbade zero of them. |
| Compile error on the `skus = ...` line after you uncomment it | Correct, and it is the *only* thing `final` was ever doing here. |

### Proof 2 — copy versus view

`CopyVersusView.java`:
```java
import java.util.*;

public class CopyVersusView {
    public static void main(String[] args) {
        List<String> source = new ArrayList<>(List.of("A", "B"));

        List<String> view = Collections.unmodifiableList(source);
        List<String> copy = List.copyOf(source);

        source.add("C");     // mutate the ORIGINAL

        System.out.println("view : " + view);
        System.out.println("copy : " + copy);

        try { view.add("D"); } catch (UnsupportedOperationException e) {
            System.out.println("view.add threw  : " + e.getClass().getSimpleName());
        }
        try { copy.add("D"); } catch (UnsupportedOperationException e) {
            System.out.println("copy.add threw  : " + e.getClass().getSimpleName());
        }
    }
}
```

```bash
java CopyVersusView.java
```

**What to look for:** whether `view` and `copy` printed the same thing.

| What you see | What it means |
|---|---|
| `view` has 3 elements, `copy` has 2 | The expected result, and the entire lesson. The view tracks the source; the copy is a snapshot. |
| Both `.add` calls threw `UnsupportedOperationException` | Correct. Both *look* immutable from the outside. Only one *is*. |
| `view` and `copy` both have 2 elements | You mutated something other than `source`. Re-read the program. |

### Proof 3 — `final` is in the class file, not just in the compiler

`FinalFlag.java`:
```java
public class FinalFlag {
    private final int orderId;
    private int retryCount;
    public FinalFlag(int orderId) { this.orderId = orderId; }
}
```

```bash
javac FinalFlag.java
javap -p -v FinalFlag.class | sed -n '1,60p'
```

**What to look for:** in the field listing, `orderId` should carry an access-flags line
that includes `ACC_FINAL`, and `retryCount` should not.

| What you see | What it means |
|---|---|
| `ACC_PRIVATE, ACC_FINAL` on `orderId`, `ACC_PRIVATE` only on `retryCount` | The expected result. `final` survives compilation as a fact the JVM can act on. Contrast this with `readonly`, which leaves no trace in emitted JavaScript at all. |
| Nothing about `ACC_FINAL` anywhere | You probably ran `javap` without `-v`. The flags only appear in verbose mode. |

I am telling you which flag to look for, not quoting exact output — the surrounding
format varies by JDK. If your listing differs from this description, trust your
listing. Reading class files properly is **Topic 76**.

### Proof 4 — can reflection break a `final` field?

This is genuinely version-sensitive and I would rather you settle it on your own JDK
than take my word for it.

`FinalReflection.java`:
```java
import java.lang.reflect.Field;

public class FinalReflection {
    private final int orderId = 42;

    public static void main(String[] args) throws Exception {
        FinalReflection target = new FinalReflection();
        Field f = FinalReflection.class.getDeclaredField("orderId");
        f.setAccessible(true);
        try {
            f.set(target, 99);
            System.out.println("write SUCCEEDED, field reads: " + f.getInt(target));
            System.out.println("but the inlined constant reads: " + target.orderId);
        } catch (IllegalAccessException e) {
            System.out.println("write REJECTED: " + e.getMessage());
        }
    }
}
```

```bash
java FinalReflection.java
```

**Honest statement of what I am confident about:** reflective writes to `final` fields
of **records** and of **hidden classes** are rejected on Java 17 and later. Reflective
writes to `static final` fields have been progressively locked down since Java 12
(the old `Field.class.getDeclaredField("modifiers")` hack no longer works). Ordinary
instance `final` fields have historically been writable after `setAccessible(true)`.
Whether that is still true on *your* exact JDK build is what this program settles.

| What you see | What it means |
|---|---|
| `write SUCCEEDED` and the two printed values differ | Reflection wrote the field, but the compiler had already inlined the constant `42` at the read site. This is why the JVM treats `final` as an optimisation licence, and why doing this in real code produces impossible-looking bugs. |
| `write SUCCEEDED` and both values are `99` | The write took effect and nothing was inlined. Also fine — inlining is a compiler choice, not a guarantee. |
| `write REJECTED: Can not set final ...` | Your JDK forbids it. Note the exact message and the JDK version; that is your ground truth for this platform. |

Either way, the rule for real code is the same: **reflection is not a supported way to
mutate a `final` field, and a library that does it will break on a future JDK.**

### Proof 5 — the safe-publication race (honest limits)

I am not going to give you a small program that reliably shows a partially-constructed
object, because **there is no such program**. The race is legal but rare, and whether
you observe it depends on the JIT's decisions and on your CPU's memory model.

What I can tell you honestly:

- On **x86/x64** the hardware memory model (TSO) hides most store reordering, so a
  naive reproduction attempt will almost certainly print nothing interesting. You will
  conclude, wrongly, that the problem is theoretical.
- On **aarch64** — Apple Silicon, AWS Graviton — the model is weaker and the race is
  observable far more often. If you are on an M-series Mac, you are on the hardware
  where this actually shows up.

The correct tool is `jcstress`, which runs millions of interleavings and reports an
outcomes table. That is **Topic 88**, and it has the full harness.

What you *can* do today, in one minute, is establish which hardware memory model you
are working on:

```bash
java -XX:+PrintFlagsFinal -version | grep -i 'UseCompressedOops\|UseZGC'
uname -m
```

| What you see | What it means |
|---|---|
| `arm64` / `aarch64` | Weak memory model. Publication races are observable here. Your intuition from x86 code review does not transfer. |
| `x86_64` | Strong (TSO) memory model. Many publication bugs are latent and will surface when the service is deployed to Graviton or an M-series developer machine. |

**Why this matters even without a reproduction:** you cannot test your way to
publication correctness. You establish it by construction — all-`final` fields, no
escaping `this` — and you verify it by reading the code, not by running it.

---

## Practice exercises

Write real files, run them, and keep your output.

### 1 — Easy: find and prove the leaks

Take the broken `Order` from Example 2 (the six-leak version). Write a single test
class `OrderLeakTest` that **demonstrates each of the six leaks with an assertion**.

Requirements:
- Each test method must fail on the broken version and pass on the fixed version.
- No test may use reflection. Every leak must be reachable through the public API.
- For the `Date` leak, use `setTime()`. For the array leak, write to index 0.
- Name each test after the symptom a user would report, not after the mechanism.
  (`invoiceTotalStopsMatchingLines`, not `linesGetterReturnsLiveReference`.)

Then answer in writing: which of the six leaks would a code reviewer most likely
*miss*, and why?

### 2 — Medium: the mutable key (combines Topics 10, 12, 13, 16)

Build `ProductCatalogue`:

```java
public class Product {
    private String sku;                 // deliberately mutable
    private String name;
    private ProductCategory category;   // an enum
    // constructor, getters, setters, equals(), hashCode() based on sku
}
```

**Part A.** Put 1,000 `Product` objects into a `HashSet<Product>` and into a
`Map<ProductCategory, Set<Product>>`. Confirm `contains` works for all of them.

**Part B.** Pick one product and call `setSku("CHANGED")`. Now show, with printed
output:
- `set.contains(thatProduct)` returns `false`
- `set.remove(thatProduct)` returns `false`
- `set.size()` is unchanged
- iterating the set still finds the object

Explain which clause of the `equals`/`hashCode` contract you violated, and why
`HashSet` cannot detect this. (Topic 13.)

**Part C.** Fix it by making `Product` immutable. Use `EnumMap<ProductCategory, ...>`
rather than `HashMap` for the category index and say why (Topic 16). Then explain
what you had to change about the *calling code* that used to call `setSku`, and
whether that change made the calling code better or worse.

**Part D.** Argue the other side. Name one realistic situation in `orderflow` where a
mutable entity is the right call, and say what discipline you would put around it.

### 3 — Hard: production simulation — the immutable order aggregate

**Part A — build it.** Write the `orderflow` order aggregate properly:
`Order`, `OrderLine`, `Money`, `ShippingAddress`, `PromotionCode`. Rules:
- Deep immutability throughout. No `java.util.Date`, no arrays in public signatures.
- Money as `long` minor units inside `Money` (Topic 01), never `double`.
- All mutating operations expressed as `withX(...)` methods returning new instances.
- A `Builder` for construction, because a 7-argument constructor is unreadable — and
  the builder must produce a fully-validated object or throw.

**Part B — attack it.** Write `OrderMutationAttackTest` with at least eight distinct
attempts to mutate a constructed `Order` through its public API. Every one must fail.
Include: the collection getter, the nested `OrderLine`'s collection getter, the
`Money`, the address lines, and an attempt to subclass `Order` and override a getter.

**Part C — the publication argument.** Write a short note (10–15 lines) that a
reviewer could read, answering: *if this `Order` is put into a
`ConcurrentHashMap<Long, Order>` by one thread and read by 200 request threads, what
guarantees do we have and where do they come from?* Name the specific mechanisms.
Then state honestly what you are **not** sure about and would confirm at Topic 88.

**Part D — the cost.** `withExtraLine` copies the whole line list. For an order with
200 lines, added one at a time, that is 20,100 element copies. Measure the wall-clock
cost of building a 200-line order that way versus mutating an `ArrayList`. Then answer:
- Is the difference large enough to matter for `orderflow`'s actual order sizes?
- At what order size would your answer flip?
- If it did flip, what would you change — and would you give up immutability, or would
  you keep it and change the *construction* pattern instead?

> You are wall-clock timing again, which **Topic 77** will show you is unreliable. Do
> it anyway, record the number, and come back to it after JMH.

---

## Interview questions

### Q1 — "What does `final` mean on a field?"

**Mid-level answer:** "It means the field can only be assigned once, in the constructor
or at declaration, and can't be reassigned after that."

**Senior answer:** "Two things, and the second is the interesting one. First, it
prevents reassignment of the *reference* — it says nothing about the object the
reference points at, so a `final List` is fully mutable. Second, it has Java Memory
Model semantics: there's a freeze action at the end of the constructor, so any thread
that obtains a reference to a correctly-constructed object is guaranteed to see the
correctly-initialised final fields, with no synchronisation at all. That's the basis of
safe publication, and it's why an all-final class is thread-safe for free. The
guarantee is void if `this` escapes the constructor."

**What separates them:** the mid answer is the syntax rule. The senior answer names the
memory-model guarantee, states its precondition, and knows the shallowness is a
separate fact from the freeze.

**Follow-up the interviewer asks:** "What would void that guarantee?" They want
`this` escaping the constructor — registering a listener, starting a thread, or calling
an overridable method from inside the constructor.

---

### Q2 — "Here's an immutable class. Is it?"

```java
public final class Order {
    private final List<OrderLine> lines;
    public Order(List<OrderLine> lines) { this.lines = lines; }
    public List<OrderLine> lines() { return lines; }
}
```

**Mid-level answer:** "No — the getter returns the internal list, so a caller can
mutate it. You should return `Collections.unmodifiableList(lines)`."

**Senior answer:** "No, and there are two separate holes, not one. The getter leaks
the reference out, and the constructor accepts a reference the caller still holds — so
even if I fix the getter, the original caller can still mutate it. I'd use
`List.copyOf` in the constructor, which fixes both at once because the field is then
genuinely immutable and safe to return directly.
`Collections.unmodifiableList` only fixes the getter; it's a view over the same list,
so the constructor hole stays open. And there's a third level: this is only immutable
if `OrderLine` is. Shallow immutability with a mutable element type buys you nothing."

**What separates them:** seeing the constructor hole, knowing view-versus-copy, and
raising the depth question unprompted.

**Follow-up:** "Would making this a `record` fix it?" The answer is no — a record gives
you shallow immutability and generates the leaky accessor for you. The `List.copyOf`
still has to go in the compact constructor. (Topic 27.)

---

### Q3 — "Why is constructor injection better than field injection in Spring?"

**Mid-level answer:** "It makes the class easier to test because you can construct it
without a container, and it fails fast if a dependency is missing."

**Senior answer:** "Three reasons, and the third is the one people skip. One: you can't
construct an invalid object, so a missing dependency is a startup failure rather than
an NPE on the first request. Two: it makes the god-class visible — nine constructor
parameters is a design signal, nine `@Autowired` fields is invisible. Three: the fields
can be `final`, which means the bean is safely published. A singleton bean is read by
every request thread with no synchronisation anywhere; with field injection the fields
are non-final and written after construction, so strictly speaking you're relying on
the container's own happens-before edges rather than on the object itself. Making the
fields final removes that reliance entirely."

**What separates them:** connecting a framework style rule to the memory model. Almost
nobody does this, and it is exactly the depth the question is probing for.

**Follow-up:** "Does that mean field injection is actually broken?" The honest answer
is no in practice — Spring's container publishes the bean after population, and that
establishes the edge — but the guarantee then comes from the framework rather than from
your class, which is a worse place for it to live. Saying "in practice it works, but
here's why I still wouldn't" is the strong answer.

---

### Q4 — "`readonly` in TypeScript versus `final` in Java. Same thing?"

**Mid-level answer:** "Roughly, yes — both stop you reassigning."

**Senior answer:** "Related but not the same, in three ways. `readonly` is erased —
it exists only for the type checker, so the emitted JavaScript has no enforcement and
any untyped caller can write the property. `final` is recorded as `ACC_FINAL` in the
class file and honoured by the JVM. Second, `final` carries memory-model meaning that
`readonly` has no concept of, because a single-threaded runtime has nothing to say
about visibility across threads. Third, they're both shallow, which is the one part
that does transfer cleanly. If I wanted the closest runtime analogue to `final` on the
JS side it'd be `const`, but that's local-only and also shallow, so it maps to a `final`
local variable rather than to a `final` field."

**What separates them:** distinguishing compile-time erasure from runtime enforcement,
and naming the memory-model dimension as something that simply does not exist on the
other side rather than fudging an equivalence.

**Follow-up:** "What about `Object.freeze`?" They want you to notice it freezes the
*target*, not the reference — the opposite axis from `final`, and still shallow.

---

### Q5 — "Immutability costs allocations. When would you not use it?"

**Mid-level answer:** "When performance matters and you're creating a lot of objects."

**Senior answer:** "Rarely, and only with a measurement. The default is immutable,
because the cost is allocation of short-lived objects, which the JVM is extremely good
at — young-generation allocation is a pointer bump and collection of dead young objects
is close to free. The cases where I'd genuinely reconsider: a large object mutated in a
tight loop where each copy is O(n) — a 200-line order rebuilt per line is quadratic —
and there the fix is usually a mutable *builder* that produces one immutable result,
not a mutable domain object. Second case: a genuinely large buffer, where I'd use a
mutable structure confined to one thread and never publish it. What I would not do is
give up immutability on a domain aggregate that's shared across threads, because the
concurrency cost of getting that wrong dwarfs the allocation cost of getting it right."

**What separates them:** naming builders as the escape hatch that keeps the invariant,
distinguishing thread-confined mutation from shared mutation, and refusing to trade
correctness for an unmeasured allocation.

**Follow-up:** "How would you measure whether it mattered?" JMH plus an allocation
profiler (Topics 77 and 78), not a stopwatch.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A `record` gives you `final` fields automatically. Explain, in one sentence each,
   why that still does not make a record holding a `List` immutable, and why it *does*
   make it safely publishable.

2. `String` is `final` as a class, and its internal byte array is `final` as a field —
   yet the array is a mutable object. Why is `String` still genuinely immutable? What
   specific discipline is `String` following that your `Order` must also follow?

3. The final-field freeze happens at the end of the constructor. What, precisely, goes
   wrong if a superclass constructor calls an overridable method that a subclass has
   overridden to read a subclass field?

4. You are reviewing a PR that adds `Collections.unmodifiableList` to five getters.
   Every test passes. Under what circumstances is this PR a genuine fix, and under what
   circumstances is it worse than no change at all?

5. Safe publication has no TypeScript analogue because the runtime is single-threaded.
   Yet Node has `worker_threads` and `SharedArrayBuffer`. Why does the analogue still
   not hold for ordinary Node code, and what exactly changes when you add
   `SharedArrayBuffer`?

6. If you could add one thing to Java's design here, would you make fields `final` by
   default (requiring an explicit `mutable` keyword), or would you add deep immutability
   as a language feature? Argue for one, then make the strongest case against yourself.

7. An immutable object is thread-safe. Does that mean a `List<Order>` of immutable
   orders is thread-safe? Justify your answer as a code-review rule, not as a fact.

---

## Quick reference card

### What `final` does and does not do

| | `final` field | |
|---|---|---|
| Prevents reassigning the reference | **yes** | compile-time error |
| Prevents mutating the target object | **no** | this is the trap |
| Recorded in the class file | **yes** | `ACC_FINAL` |
| Enables JIT constant-folding | **yes** | especially `static final` |
| Guarantees visibility to other threads | **yes, at end of constructor** | void if `this` escapes |
| Applies to array elements | **no** | `final int[] a; a[0] = 9;` is legal |

### Copy versus view

```java
List.copyOf(x)                                    // COPY. Immutable. Rejects nulls.
Set.copyOf(x) / Map.copyOf(x)                     // COPY. Immutable.
List.of(a, b) / Map.of(k, v)                      // Immutable, built from values.
Collections.unmodifiableList(x)                   // VIEW. Tracks x. Not a copy.
Collections.unmodifiableList(new ArrayList<>(x))  // COPY then wrap. Pre-Java-10 idiom.
new ArrayList<>(x)                                // MUTABLE copy. For internal use.
Arrays.copyOf(a, a.length) / a.clone()            // arrays: copy is the only option
```

### Types to use and to avoid

| Use | Avoid | Why |
|---|---|---|
| `Instant`, `LocalDate`, `LocalDateTime`, `Duration` | `java.util.Date`, `Calendar` | the old ones have setters |
| `BigDecimal`, `String`, wrappers | — | already immutable |
| `List<String>` field | `String[]` field | arrays can never be immutable |
| `Money` value object with `long` minor units | `double` | Topic 01 |
| `record` + `List.copyOf` in the compact constructor | bare `record` with a `List` | shallow ≠ deep |

### The immutable-class checklist

- [ ] Class is `final` (or all constructors are private with a static factory).
- [ ] Every field is `private final`.
- [ ] No setters and no method that changes a field.
- [ ] Every mutable constructor argument is **copied in**.
- [ ] Every mutable field is **copied out** — or is already immutable, which is better.
- [ ] No `java.util.Date`, no arrays in the public API.
- [ ] Validation happens in the constructor, so an invalid instance cannot exist.
- [ ] `this` never escapes the constructor: no listener registration, no thread start,
      no call to an overridable method.
- [ ] Change is expressed as `withX()` returning a new instance.
- [ ] There is a test that *attempts* mutation and asserts it fails.

### Gotchas

- `final` is shallow. Always. In every position.
- `Collections.unmodifiableXxx` is a view, not a copy.
- `List.copyOf` of an already-immutable list may return the same instance — that is
  fine and is an optimisation, not a leak.
- A `final` array's *elements* are mutable.
- A `static final` field is inlined at compile time for primitives and `String`
  constants, so changing it requires recompiling every caller, not just the declaring
  class. (Binary compatibility — Topic 32's neighbourhood.)
- Immutable ≠ safely published if `this` escaped. Two separate properties.

---

## When would I use this at work?

**1. Designing any DTO, event payload, or domain value object.**
Every request DTO, every Kafka event, every cache entry in `orderflow` should be
deeply immutable. This is not a preference — it is what makes the object safe to put in
a `ConcurrentHashMap`, safe to hand to an `@Async` method, and safe to log without
racing the logger. You will make this decision on your first day and every day after.

**2. Reviewing a pull request.**
Someone writes `public List<OrderLine> getLines() { return this.lines; }`. You catch it
in review instead of in a four-day-old finance discrepancy. The tell is easy to train:
any getter that returns a `List`, `Map`, `Set`, array, or `Date` is a leak until proven
otherwise.

**3. Diagnosing an impossible NullPointerException.**
A `final` field that is assigned in the constructor reads as `null` in production, once
a week, under load. Instead of adding logging and hoping, you go straight to "did
`this` escape the constructor?" and grep for listener registration, thread starts and
overridable calls inside constructors. That is a five-minute diagnosis instead of a
two-week ghost hunt. The full machinery is Topics 86–88; the instinct starts here.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and wrappers**: wrappers are immutable, which is *why* sharing
  cached `Integer` instances is safe at all.
- **10 — Collections framework**: the `List`/`Set`/`Map` contracts you are copying.
- **13 — equals/hashCode contract**: a mutable key is the sharpest reason to prefer
  immutability. Immutability is the structural fix for that whole bug class.
- **16 — Enum collections**: enums are the perfect immutable key type.

**This unlocks:**
- **18 — Strings**: `String` is the canonical immutable class in the JDK. The pool only
  works *because* strings are immutable.
- **19 — Serialization**: deserialization bypasses your constructor for ordinary
  classes, which means it bypasses your defensive copies and your validation. Records
  fix this; ordinary `Serializable` classes need `readObject` to redo the work.
- **27 — Records**: the language feature that writes Level 2 immutability for you, and
  the compact constructor where your `List.copyOf` calls belong.
- **39 — Dependency injection styles**: `final` fields are the memory-model argument for
  constructor injection.
- **79 — Memory leaks**: an immutable object cached forever is still a leak. Immutability
  fixes correctness, not retention.
- **86 — Happens-before**: why "the object was created before it was used" is not a
  statement about other threads.
- **87 — `volatile`**: the tool for publishing the *reference* safely.
- **88 — `final` fields and safe publication**: the full treatment, with a `jcstress`
  harness that shows the race this topic only describes.
- **92 — Concurrent collections**: immutable values are what make a
  `ConcurrentHashMap<Long, Order>` actually safe rather than merely non-throwing.

---

*Java baseline 21. Nothing in this topic's core semantics changed between 21 and 25 —
the final-field freeze has been specified since Java 5's JSR-133 memory model, and
`List.copyOf` has existed since Java 10. The one genuinely moving part is how much
reflection is still permitted to write `final` fields; Proof 4 settles that on your own
JDK rather than trusting a version number.*
