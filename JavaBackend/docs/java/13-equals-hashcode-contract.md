# 13 — The equals/hashCode Contract and Silent HashMap Corruption

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Back to the cloakroom from Topic 12.

The rule is simple: **the peg number must be computed from something that never
changes.**

You hang a coat on peg 7 because its label says "blue, wool, size M". Later,
someone dyes the coat red. Now you come back and ask for "red, wool, size M" —
the clerk computes peg 3, looks at peg 3, and says "no such coat".

The coat is still there. It is on peg 7. It is taking up space. The clerk can see
it if you make them walk the whole room. But they will never find it by asking,
and they will never remove it by asking, because every question they ask goes to
peg 3.

That is the entire failure. Nothing crashes. Nothing logs. The cloakroom's
"number of coats" counter still says the coat is there — because it is. You just
cannot get it back.

Two rules follow, and they are the whole topic:

1. **If two things are equal, they must produce the same peg number.** Otherwise
   equal things land in different places and you never find the one you have.
2. **The peg number must not change while the thing is hanging up.** Otherwise
   you lose it, permanently, while still paying rent on it.

Rule 1 is the `equals`/`hashCode` contract. Rule 2 is the mutable-key trap.
Rule 1 is a compile-and-forget mistake. Rule 2 is the one that ends up in a
postmortem.

---

## The bridge from what you know

### There is no analogue. Read this section carefully.

In TypeScript you compare objects one of three ways:

```ts
a === b                                   // reference identity. Built in. Not overridable.
JSON.stringify(a) === JSON.stringify(b)   // structural, fragile, key-order dependent
_.isEqual(a, b)                           // structural, from a library
```

And when you put an object in a `Map` or `Set`:

```ts
const seen = new Set<Product>();
seen.add({ sku: "SKU-4471" });
seen.has({ sku: "SKU-4471" });   // false. Always. Different object.
```

That is **SameValueZero**: for objects it is reference identity, full stop. It is
defined by the ECMAScript spec, it is not a hook, and **you cannot override it**.
There is no `Symbol.equals`. There is no `Symbol.hashCode`. A JS `Map` will never
consider two distinct objects to be the same key, no matter what you write.

So the mental model you arrive with is: *"object keys are identity keys, and
structural comparison is something I do explicitly with a library function."*

In Java, that model is wrong in a specific way:

```java
Set<Product> seen = new HashSet<>();
seen.add(new Product("SKU-4471"));
seen.contains(new Product("SKU-4471"));
// true or false — depending entirely on code YOU wrote in Product
```

**You own the definition of "same key".** `HashSet` and `HashMap` call your
`hashCode()` to find the bucket and your `equals()` to confirm the match. If
those two methods disagree with each other, the collection does not throw, does
not warn, and does not degrade gracefully. It silently loses your data.

| TypeScript | Java | Verdict |
|---|---|---|
| `===` on objects — reference identity, not overridable | `equals()` — you define it | **NO ANALOGUE** |
| No hash-code concept exposed at all | `hashCode()` — you define it, and it must agree with `equals()` | **NO ANALOGUE** |
| `Map`/`Set` key equality is fixed by the spec | Key equality is a method on your class | **NO ANALOGUE** |
| `lodash.isEqual(a, b)` — an explicit call at a call site | `a.equals(b)` — implicit, called by every collection | **PARTIAL** — same idea, radically different blast radius |
| Mutating an object never affects `Set` membership | Mutating a key breaks lookup permanently | **NO ANALOGUE** |

That last row is the one to internalise. In JavaScript, `set.add(obj)` then
`obj.x = 5` then `set.has(obj)` is **still `true`**, because the set is holding
the reference and comparing by reference. In Java the same three lines can give
you `false`. Your instinct is not merely unhelpful here — it is actively wrong,
and it will not fire a warning when it misleads you.

---

## What is this?

Two methods on `java.lang.Object`, which every class inherits:

```java
public boolean equals(Object obj)   // default: this == obj  (reference identity)
public int hashCode()               // default: an identity-derived int
```

The defaults are consistent with each other. The moment you override one, you
take on responsibility for keeping them consistent — and the language will not
help you.

### The `equals` contract

For non-null references `x`, `y`, `z`:

| Clause | Meaning | Typical way it breaks |
|---|---|---|
| **Reflexive** | `x.equals(x)` is `true` | Almost never broken deliberately. Broken accidentally with `NaN` fields, or `==` on `Double` fields |
| **Symmetric** | `x.equals(y)` iff `y.equals(x)` | A subclass adding fields to `equals`; comparing across types |
| **Transitive** | `x.equals(y)` and `y.equals(z)` implies `x.equals(z)` | "Equals with tolerance"; three-class inheritance chains |
| **Consistent** | Repeated calls give the same answer, if nothing changed | Using a mutable field, a random value, or a network call |
| **Null-safe** | `x.equals(null)` is `false`, never a throw | Forgetting the null check (though `instanceof` handles it for you) |

### The `hashCode` contract

| Clause | Meaning |
|---|---|
| **Consistent** | Same object, same run, unchanged fields → same value |
| **Equal implies equal hash** | `a.equals(b)` **must** imply `a.hashCode() == b.hashCode()` |
| **Unequal need not differ** | Distinct objects *may* share a hash code. Legal, and only a performance question |

That middle clause is the load-bearing one. Note the direction. It is a one-way
implication:

- Equal objects **must** have equal hash codes. **Not optional.**
- Equal hash codes do **not** imply equal objects. That is just a collision, and
  Topic 12 explained how the map handles it.

### What "the contract" actually protects

Nothing in the JVM enforces this. No compiler check, no runtime assertion, no
warning. The contract is a promise you make to every hash-based collection in
the JDK — `HashMap`, `HashSet`, `LinkedHashMap`, `LinkedHashSet`,
`ConcurrentHashMap`, `Collectors.toMap`, `Collectors.groupingBy`, the
distinct-element logic in `Stream.distinct()`, and dozens of library and
framework internals you never see.

Break it, and every one of those quietly does the wrong thing.

---

## Why does it matter?

**1. The failure mode is silent data loss, not an exception.**

That distinction is everything. An exception gives you a stack trace, a line
number, and an alert. This gives you an order that is not in the list, a
duplicate charge that "shouldn't be possible", and a reconciliation report that
is off by four rows. You will spend days looking at the wrong layer.

**2. The mutable-key variant is simultaneously a correctness bug and a leak.**

The entry is unreachable through `get`, `contains` and `remove` — but it is still
in the table, so it is still strongly reachable from the map, so the garbage
collector will never free it. The map grows forever with entries you cannot
address. That is Topic 79's leak, produced by Topic 13's bug.

**3. Every framework you are about to learn depends on it.**

Hibernate uses `equals`/`hashCode` for entities in a `Set` association. Jackson
uses it for deduplication. Spring's caching abstraction uses it for cache keys.
Kafka consumers use it for partition assignment bookkeeping. When you get to
Phase 5 and an entity mysteriously appears twice in a `Set<OrderLine>`, this
topic is the answer.

---

## Syntax breakdown

Only constructs that are new to you.

### `instanceof` with a pattern variable

```java
if (o instanceof Product other) {
    // 'other' is in scope here, already cast to Product
}
```

This is **pattern matching for `instanceof`**, standard since Java 16. The old
form was:

```java
if (o instanceof Product) {
    Product other = (Product) o;   // the cast you already knew was safe
}
```

`[LEGACY — still asked]` The old two-line form is what you will see in any
pre-16 codebase.

It is the closest Java gets to TypeScript's control-flow narrowing
(`if (typeof x === "string") { /* x is string here */ }`) — the narrowing is
real, the variable is scoped to the branch, and the compiler tracks it. This is
covered properly in Topic 29.

### `@Override`

```java
@Override public boolean equals(Object o) { ... }
```

An annotation that tells the compiler "I intend this to override something". If
it does not, you get a compile error. Use it on **every** override without
exception — it is the single line of defence against the classic bug below.

```java
public boolean equals(Product other) { ... }   // NOT an override. Overload.
```

That signature takes `Product`, not `Object`. It does not override
`Object.equals`. `HashMap` calls `equals(Object)` and gets the inherited identity
version. Your method is never called by any collection. `@Override` on that
method is a compile error, which is exactly what you want.

### `Objects` helpers

```java
Objects.equals(a, b)          // null-safe: true if both null, false if one null
Objects.hash(a, b, c)         // combines several values into one hash code
Objects.hashCode(a)           // 0 if a is null, else a.hashCode()
Objects.requireNonNull(a)     // throws NPE immediately with a clear message
```

`Objects.hash` is **varargs**. That means every call allocates an `Object[]` and
boxes any primitive you pass — Topic 01's cost, on a method that runs on every
map lookup. Correct, readable, and not free.

### Record

```java
public record Product(String sku, String name, long priceMinor) { }
```

A `record` generates `equals`, `hashCode`, `toString`, a canonical constructor,
and accessors, from all components. The generated `equals` and `hashCode` are
guaranteed consistent with each other. This is the correct default for a map key,
and it is Topic 27.

---

## Example 1 — minimal

```java
import java.util.*;

public class ForgotHashCode {

    static final class Sku {
        private final String code;
        Sku(String code) { this.code = code; }

        @Override public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Sku other)) return false;
            return code.equals(other.code);
        }
        // hashCode deliberately NOT overridden
    }

    public static void main(String[] args) {
        Sku a = new Sku("SKU-4471");
        Sku b = new Sku("SKU-4471");

        System.out.println("a.equals(b)        : " + a.equals(b));
        System.out.println("a.hashCode()==b's  : " + (a.hashCode() == b.hashCode()));

        Set<Sku> set = new HashSet<>();
        set.add(a);
        System.out.println("set.contains(b)    : " + set.contains(b));

        List<Sku> list = new ArrayList<>();
        list.add(a);
        System.out.println("list.contains(b)   : " + list.contains(b));
    }
}
```

Run it. Four lines of output, and the interesting part is that `list.contains(b)`
is `true` while `set.contains(b)` is `false`.

`ArrayList.contains` walks the list calling `equals` on every element — it never
touches `hashCode`, so a broken `hashCode` cannot hurt it. `HashSet.contains`
computes a bucket from `hashCode` *first* and only calls `equals` on what it
finds there. Two objects that are `equals` but land in different buckets never
get compared.

**This is why the bug hides.** Your `List`-based code works. Your `Stream`
filters work. Your `assertEquals` works. Only the hash-based structures fail, and
they fail without a sound.

---

## Example 2 — production scenario

`orderflow` deduplicates inbound payment webhooks. The gateway retries on
timeout, so the same event can arrive several times, and charging a wallet twice
is the worst bug the service can have.

### The version that ships and double-charges

```java
public final class PaymentEvent {

    private final String gateway;
    private final String gatewayRef;
    private String status;               // "PENDING" -> "SETTLED" -> "REFUNDED"
    private Instant lastSeenAt;

    public PaymentEvent(String gateway, String gatewayRef, String status) {
        this.gateway = gateway;
        this.gatewayRef = gatewayRef;
        this.status = status;
        this.lastSeenAt = Instant.now();
    }

    public void markSettled() {
        this.status = "SETTLED";                 // <-- mutates a field used below
        this.lastSeenAt = Instant.now();
    }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PaymentEvent other)) return false;
        return gateway.equals(other.gateway)
            && gatewayRef.equals(other.gatewayRef)
            && status.equals(other.status);       // <-- defect: mutable field
    }

    @Override public int hashCode() {
        return Objects.hash(gateway, gatewayRef, status);   // <-- same defect
    }
}
```

```java
public class WebhookDeduplicator {

    private final Set<PaymentEvent> processed = new HashSet<>();

    public boolean acceptOnce(PaymentEvent event) {
        if (processed.contains(event)) {
            return false;                       // already handled, ignore
        }
        processed.add(event);
        walletService.credit(event);
        event.markSettled();                    // <-- mutation AFTER insertion
        return true;
    }
}
```

Read the sequence carefully.

1. Event arrives with `status = "PENDING"`. `contains` is `false`. It is added.
   Its hash is computed from `("STRIPE", "ref-991", "PENDING")` and it lands in,
   say, bucket 214.
2. The wallet is credited.
3. `markSettled()` changes `status` to `"SETTLED"`. **The object's hash code has
   now changed. The entry has not moved. It is still in bucket 214.**
4. The gateway retries. A *new* `PaymentEvent` arrives, also `"PENDING"`.
   `contains` computes bucket 214, looks there, finds the stored event, calls
   `equals` — and gets `false`, because the stored one now says `"SETTLED"`.
5. The wallet is credited **again**.

### The observable symptoms, in the order you will meet them

- Support tickets: "I was charged twice."
- The `processed` set's `size()` grows monotonically and never stabilises, even
  though the number of *distinct* payments is bounded.
- A heap dump shows a large `HashSet` retaining `PaymentEvent` instances that
  nothing else references.
- No exception. No error log. No metric moves except memory.

### The corrected version

```java
public record PaymentEventKey(String gateway, String gatewayRef) { }
```

```java
public class WebhookDeduplicator {

    private final Set<PaymentEventKey> processed = new HashSet<>();

    public boolean acceptOnce(PaymentEvent event) {
        PaymentEventKey key = new PaymentEventKey(event.gateway(), event.gatewayRef());
        if (!processed.add(key)) {       // add() returns false if already present
            return false;
        }
        walletService.credit(event);
        event.markSettled();             // now harmless: mutates a non-key object
        return true;
    }
}
```

Four changes, each removing a distinct hazard:

1. **A separate key type.** The thing in the set is no longer the thing that
   mutates. This is the structural fix, and it is the one that matters.
2. **`record`.** `equals` and `hashCode` are generated from all components and
   cannot drift apart when someone adds a field.
3. **Only immutable identity fields in the key.** `gateway` and `gatewayRef` are
   assigned at construction and never change. `status` is not identity — it is
   state, and state does not belong in a key.
4. **`add()` returns `false` if present**, so the check and the insert are one
   operation instead of a check-then-act pair.

> This is still not production-complete. An in-memory `Set` that only grows is a
> leak (Topic 79), and idempotency that must survive a restart or a second
> instance needs a unique constraint in the database, not a field in a JVM
> (Topics 92 and 116). The contract bug is fixed; the design work is later.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — overriding `equals` and forgetting `hashCode`

**Wrong:**
```java
@Override public boolean equals(Object o) { /* compares fields */ }
// no hashCode
```

**Exact symptom:**
```java
set.add(a);
set.contains(b);              // false, even though a.equals(b) is true
map.put(a, x);
map.get(b);                   // null
list.contains(b);             // TRUE — which is what makes this confusing
```
No exception. Typically found because a `Stream.distinct()` produced duplicates,
or a `Collectors.groupingBy` created one group per element instead of one group
per key.

**Root cause:** `hashCode` is still `Object`'s identity-based version, so two
`equals` objects produce different hash codes and land in different buckets.
`HashSet.contains` never gets far enough to call `equals`.

**Fix:** always override both, together, from the same fields. Better: use a
`record`. Better still: turn on the compiler/IDE inspection so it is impossible
to merge one without the other.

```bash
# ErrorProne catches this at build time
# Maven: add error_prone_core to maven-compiler-plugin annotationProcessorPaths
# The check name is: EqualsHashCode
```

---

### Trap 2 — the mutable key (the headline)

**Wrong:**
```java
Set<Product> catalogue = new HashSet<>();
Product p = new Product("SKU-4471", "Widget");
catalogue.add(p);
p.setSku("SKU-9902");            // hash-relevant field changed after insertion
```

**Exact symptom:** three things at once, and the combination is diagnostic.
```java
catalogue.contains(p);   // false  <- you are holding the very object
catalogue.remove(p);     // false  <- you cannot delete it either
catalogue.size();        // 1      <- but it is definitely still in there
for (Product q : catalogue) { }   // iterates it fine
```
Plus, over time: unbounded memory growth in a collection whose logical size
should be bounded.

**Root cause:** Topic 12, precisely. The spread hash is computed once and cached
in the `Node` at insertion. Lookup recomputes the hash from the *current* field
values, gets a different number, computes a different bucket index, and looks in
the wrong place. The entry is not lost — it is in the original bucket, strongly
reachable from the map, and addressable only by full iteration.

**Fix, in order of preference:**

1. Make the key immutable. `final` fields, no setters, or a `record`.
2. If the object must be mutable, key on something that is not: an ID, a
   separate key `record`, or a `String`.
3. If you genuinely must mutate a key, `remove` it first and `add` it back after.
   This works and is almost always a sign the design is wrong.

This trap gets its own drill below.

---

### Trap 3 — asymmetric `equals` from a subclass

**Wrong:**
```java
public class Product {
    protected final String sku;
    @Override public boolean equals(Object o) {
        if (!(o instanceof Product p)) return false;
        return sku.equals(p.sku);
    }
    @Override public int hashCode() { return sku.hashCode(); }
}

public class DiscountedProduct extends Product {
    private final int discountPercent;
    @Override public boolean equals(Object o) {
        if (!(o instanceof DiscountedProduct d)) return false;
        return super.equals(o) && discountPercent == d.discountPercent;
    }
    @Override public int hashCode() { return Objects.hash(sku, discountPercent); }
}
```

**Exact symptom:** the answer depends on argument order, and therefore on
insertion order.
```java
Product base = new Product("SKU-1");
DiscountedProduct disc = new DiscountedProduct("SKU-1", 10);

base.equals(disc);     // true   (a DiscountedProduct IS a Product with that sku)
disc.equals(base);     // false  (a Product is not a DiscountedProduct)

List<Product> l = List.of(base);
l.contains(disc);      // depends on which side equals is called from
```
In practice: a `Set` containing both, or containing one, depending on which was
added first. A deduplication step that removes an item in one run and not the
next.

**Root cause:** symmetry is violated. `instanceof` in the superclass accepts
subclasses; `instanceof` in the subclass rejects the superclass.

**Fix — pick one and be able to defend it:**

- **`getClass() != o.getClass()` instead of `instanceof`.** Restores symmetry.
  Cost: a subclass can never be `equals` to its parent, which breaks Liskov
  substitution and — importantly for you — **breaks Hibernate**, because a lazy
  association hands you a runtime-generated proxy subclass whose `getClass()` is
  not your entity class (Topic 49).
- **Make the class `final`.** No subclasses, no problem. This is why value types
  should be `final`, and why `record` is implicitly `final`.
- **Favour composition over inheritance for value types.** A `DiscountedProduct`
  that *has* a `Product` rather than *is* one has no contract to violate.

There is no way to add a value-carrying field to a subclass and preserve
symmetry. That is a genuine, well-known limitation of the contract, not a Java
wart you can code around.

---

### Trap 4 — "equals with tolerance" breaks transitivity

**Wrong:**
```java
@Override public boolean equals(Object o) {
    if (!(o instanceof Money m)) return false;
    return Math.abs(amountMinor - m.amountMinor) < 2;   // "close enough"
}
```

**Exact symptom:** deduplication produces different results depending on
processing order. A `Set` that contains three elements after one insertion order
and two after another. In tests, a flaky assertion on collection size that only
fails under a different data ordering.

**Root cause:** transitivity. With amounts 100, 101, 102: `100.equals(101)` is
true, `101.equals(102)` is true, `100.equals(102)` is false. There is no
consistent hash function for this relation at all — "nearly equal" is not an
equivalence relation, so no `hashCode` can satisfy the contract.

**Fix:** equality is exact, or it is not equality. If you need tolerance, that is
a `Comparator` or a domain method (`isWithin(other, tolerance)`) — an explicit
call at a call site, not a contract every collection in the JDK relies on. And
for money specifically: integer minor units, never a float (Topic 01).

---

### Trap 5 — arrays as fields

**Wrong:**
```java
public final class ShipmentLabel {
    private final byte[] barcode;

    @Override public boolean equals(Object o) {
        if (!(o instanceof ShipmentLabel s)) return false;
        return barcode.equals(s.barcode);          // Object.equals — identity!
    }
    @Override public int hashCode() {
        return Objects.hash(barcode);              // identity hash of the array
    }
}
```

**Exact symptom:** two labels with byte-for-byte identical barcodes are never
equal, and never dedupe. `set.add(a); set.add(b);` gives `size() == 2` for what
should be one label. The same symptom appears with a `record` that has an array
component — the generated `equals` uses `Objects.equals` on each component, which
for arrays is identity.

**Root cause:** arrays in Java do not override `equals` or `hashCode`. They
inherit `Object`'s identity versions. `Objects.hash(someArray)` hashes the
*reference*, not the contents.

**Fix:**
```java
@Override public boolean equals(Object o) {
    if (!(o instanceof ShipmentLabel s)) return false;
    return Arrays.equals(barcode, s.barcode);
}
@Override public int hashCode() {
    return Arrays.hashCode(barcode);
}
```
For nested arrays: `Arrays.deepEquals` / `Arrays.deepHashCode`. Better still,
avoid array fields in value types — use a `List<Byte>`, a `String`, or a wrapper
type with proper semantics. And note the ordering issue: a `List<T>` has
order-sensitive `equals`, a `Set<T>` does not. That is usually what you want, but
be deliberate about it.

---

### Trap 6 — a JPA entity keyed on a generated ID

**Wrong:**
```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @Override public boolean equals(Object o) {
        if (!(o instanceof Order other)) return false;
        return Objects.equals(id, other.id);
    }
    @Override public int hashCode() { return Objects.hash(id); }
}
```

**Exact symptom:** two different, unsaved orders are `equals` to each other
(`id` is `null` on both), so a `Set<Order>` collapses them into one and you lose
an order. Or: you add an unsaved order to a `Set`, the transaction commits, the
ID is assigned, and the order is now unreachable in the set — Trap 2 again,
arriving from a direction you did not expect.

**Root cause:** the identity field is `null` before persist and non-null after,
so it is a mutable field in exactly the way Trap 2 describes — mutated by the
persistence provider rather than by your code, which is why it is so easy to
miss.

**Fix:** assign a business key or a `UUID` in the constructor, so identity exists
from the moment the object does.
```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @Column(nullable = false, updatable = false)
    private final UUID externalId = UUID.randomUUID();

    @Override public boolean equals(Object o) {
        return o instanceof Order other && externalId.equals(other.externalId);
    }
    @Override public int hashCode() { return externalId.hashCode(); }
}
```
This is revisited properly in Topic 48. It is here because it is the single most
common real-world instance of Trap 2 in a Spring codebase.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print
output and call it real.

### Setup

```bash
mkdir -p ~/java-lab/13 && cd ~/java-lab/13
java --version          # expect 21 or 25
```

### Proof 1 — the List/Set split

Save Example 1 as `ForgotHashCode.java` and run it:

```bash
java ForgotHashCode.java
```

| What you see | What it means |
|---|---|
| `a.equals(b): true`, `a.hashCode()==b's: false` | The contract is violated. This is the whole bug in two lines |
| `set.contains(b): false` | Confirmed: `HashSet` went to the wrong bucket and never called `equals` |
| `list.contains(b): true` | Confirmed: `ArrayList` only uses `equals`, so a broken `hashCode` is invisible to it. This is why the bug survives testing |
| `a.hashCode()==b's: true` | You accidentally implemented `hashCode`, or the JVM handed out equal identity hashes (possible but very unlikely). Print both values and check |

### Proof 2 — read the contract in the source of truth

The contract is not folklore. It is in the Javadoc of `java.lang.Object`.

```bash
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/lang/Object.java' -d ~/java-lab/13/src
less ~/java-lab/13/src/java.base/java/lang/Object.java
```

Search (`/`) for `public boolean equals` and read the doc comment above it, then
for `public int hashCode`.

| What you see | What it means |
|---|---|
| The five clauses (reflexive, symmetric, transitive, consistent, null) spelled out | This is the actual contract. Every table in this doc is a restatement of that comment |
| "it is generally necessary to override the `hashCode` method whenever this method is overridden" | Confirmed: the JDK tells you, in the Javadoc, and there is still no compiler check |
| The `hashCode` doc noting that unequal objects producing distinct hashes "may improve the performance of hash tables" | Confirmed: collisions are a performance concern, not a correctness one. That is Topic 12 |

Also read `java.util.HashMap`'s class-level Javadoc:

```bash
unzip -p "$JAVA_HOME/lib/src.zip" 'java.base/java/util/HashMap.java' | head -80
```

**What to look for:** the paragraph beginning "Note that this implementation is
not synchronized" and, more relevantly here, the class doc's warning about
mutable keys. `TreeMap`'s doc has an even more explicit version of the same
warning. Reading the JDK's own hedged language is the fastest way to learn which
guarantees are real.

### Proof 3 — measure the leak

`RetentionProof.java`:
```java
import java.util.*;

public class RetentionProof {

    static final class MutableKey {
        String sku;
        MutableKey(String sku) { this.sku = sku; }
        @Override public boolean equals(Object o) {
            return o instanceof MutableKey k && sku.equals(k.sku);
        }
        @Override public int hashCode() { return sku.hashCode(); }
    }

    public static void main(String[] args) throws Exception {
        Set<MutableKey> set = new HashSet<>();
        for (int i = 0; i < 200_000; i++) {
            MutableKey k = new MutableKey("SKU-" + i);
            set.add(k);
            k.sku = "MUTATED-" + i;      // break it immediately after insertion
        }
        System.out.println("size()          = " + set.size());
        System.out.println("contains first  = " + set.contains(new MutableKey("SKU-0")));
        System.out.println("contains mutated= " + set.contains(new MutableKey("MUTATED-0")));

        System.out.println("Attach a profiler now. PID = " + ProcessHandle.current().pid());
        Thread.sleep(120_000);
    }
}
```

```bash
java -Xmx256m RetentionProof.java &
sleep 5
jcmd $(pgrep -f RetentionProof) GC.class_histogram | head -20
```

| What you see | What it means |
|---|---|
| `size() = 200000` | Confirmed: every entry is still counted. The set has not "lost" anything from its own point of view |
| `contains first = false` | Confirmed: the original key is unreachable, because its hash changed |
| `contains mutated = false` | **This is the important one.** Even the *new* value does not find it — the entry is in the bucket for the *old* hash |
| A large `RetentionProof$MutableKey` row in the histogram | Confirmed: all 200,000 objects are strongly retained by the set and cannot be collected |
| `OutOfMemoryError` before the sleep | Raise `-Xmx` or lower the loop count. This is the leak, arriving faster than expected |

To go further, take a heap dump and open it properly:
```bash
jcmd $(pgrep -f RetentionProof) GC.heap_dump /tmp/leak.hprof
```
Open `/tmp/leak.hprof` in Eclipse MAT and look at the dominator tree. The
`HashSet` will be the dominator of the entire retained set. That is the exact
workflow of Topic 79, and this is the smallest possible example of it.

### Proof 4 — check your codebase for the bug automatically

You should not be finding this by reading. Wire the check into the build.

```bash
# Option A: SpotBugs (standalone, no build changes)
curl -L -o spotbugs.tgz https://github.com/spotbugs/spotbugs/releases/download/4.8.6/spotbugs-4.8.6.tgz
tar xzf spotbugs.tgz
./spotbugs-4.8.6/bin/spotbugs -textui -low target/classes
```

**What to look for:** detector IDs `HE_EQUALS_USE_HASHCODE`,
`HE_INHERITS_EQUALS_USE_HASHCODE`, and `EQ_` prefixed reports.

| What you see | What it means |
|---|---|
| `HE_EQUALS_USE_HASHCODE` | A class defines `equals` but not `hashCode`. Trap 1, found statically |
| `EQ_SELF_NO_OBJECT` | An `equals` overload taking a specific type instead of `Object`. The `@Override` bug |
| No findings but you know there is a bug | SpotBugs cannot see mutable keys — that is a *usage* pattern, not a class-level defect. No static tool catches Trap 2 reliably. This is why the drill below matters |

> Version note: the SpotBugs release URL above will go stale. Check the current
> release on the SpotBugs GitHub releases page rather than trusting the version
> number here. The detector IDs have been stable for years.

For a stricter, build-integrated option, ErrorProne's `EqualsHashCode` check
fails the compile rather than producing a report. That is the version you want in
CI.

---

## Failure drill  `[BONUS]`

This is the drill assigned to Topic 13 in the master plan's failure-drill map:
*mutate a key after insertion into a `HashSet`; capture `contains` false and
`size` unchanged; the fix proves the contract is a data-integrity guarantee.*

Do it. Do not read the outcome and nod. The point of a drill is that you produce
the failure with your own hands so you recognise it later at 2am.

### Setup

```bash
mkdir -p ~/java-lab/13/drill && cd ~/java-lab/13/drill
```

`Drill.java`:
```java
import java.util.*;

public class Drill {

    // A deliberately mutable "product". This is the shape of the bug.
    static final class Product {
        private String sku;
        private final String name;

        Product(String sku, String name) { this.sku = sku; this.name = name; }

        String sku() { return sku; }
        void setSku(String sku) { this.sku = sku; }

        @Override public boolean equals(Object o) {
            return o instanceof Product p
                && sku.equals(p.sku)
                && name.equals(p.name);
        }
        @Override public int hashCode() { return Objects.hash(sku, name); }
        @Override public String toString() { return sku + "/" + name; }
    }

    public static void main(String[] args) {
        Set<Product> catalogue = new HashSet<>();

        Product widget = new Product("SKU-4471", "Widget");
        catalogue.add(widget);

        System.out.println("=== BEFORE MUTATION ===");
        System.out.println("hashCode        : " + widget.hashCode());
        System.out.println("contains(widget): " + catalogue.contains(widget));
        System.out.println("size()          : " + catalogue.size());

        // The single line that breaks everything.
        widget.setSku("SKU-9902");

        System.out.println();
        System.out.println("=== AFTER MUTATION ===");
        System.out.println("hashCode        : " + widget.hashCode());
        System.out.println("contains(widget): " + catalogue.contains(widget));
        System.out.println("remove(widget)  : " + catalogue.remove(widget));
        System.out.println("size()          : " + catalogue.size());
        System.out.println("isEmpty()       : " + catalogue.isEmpty());

        System.out.print("iterating        : ");
        for (Product p : catalogue) System.out.print(p + " ");
        System.out.println();

        // Can you find it by iteration? Yes. Can you remove it that way? Yes.
        Iterator<Product> it = catalogue.iterator();
        while (it.hasNext()) {
            Product p = it.next();
            if (p.sku().equals("SKU-9902")) it.remove();
        }
        System.out.println("after iterator remove, size(): " + catalogue.size());
    }
}
```

### Run it

```bash
java Drill.java
```

### What to capture

Copy the whole output into your notes. Specifically, these five values:

1. `hashCode` before mutation
2. `hashCode` after mutation
3. `contains(widget)` after mutation
4. `remove(widget)` after mutation
5. `size()` after the failed remove

### How to read it

| What you see | What it means |
|---|---|
| The two `hashCode` values differ | The bucket index has changed. Everything below follows from this one fact |
| `contains(widget): true` before, `false` after | **The headline.** You are holding the exact object reference and the set says it is not there |
| `remove(widget): false` | You cannot delete it either. `remove` uses the same bucket-then-equals path as `contains` |
| `size(): 1` and `isEmpty(): false` | The entry was never removed. The set's own bookkeeping is correct and consistent with reality — it *is* still in there |
| Iteration prints `SKU-9902/Widget` | The object is fully reachable. It walks the table array directly and never computes a hash. This is why the entry is retained forever by the GC |
| `after iterator remove, size(): 0` | The only way out. `Iterator.remove()` unlinks the node it is currently positioned on, with no hashing involved |
| `contains` is `true` after mutation | Your `hashCode` does not actually depend on `sku`, or you got a hash collision that happened to land in the same bucket. Print `Objects.hash(...)` for both values and check |

### Now prove the fix

Change one line — make `Product` a record:

```java
record Product(String sku, String name) { }
```

and replace `widget.setSku("SKU-9902")` with the only thing you can now do:

```java
Product mutated = new Product("SKU-9902", "Widget");
// the original 'widget' is unchanged and still findable
```

Re-run.

### What the fix proves

Not "records are nicer". Something more specific:

**The contract is a data-integrity guarantee, and immutability is how you keep
it.** A `record` does not merely save you typing `equals` and `hashCode` — it
makes the bug *unexpressible*, because there is no setter to call. You did not
fix the bug by being careful. You fixed it by removing the operation that causes
it.

That distinction is the senior-level version of this topic. "Be careful with
mutable keys" is advice. "Key types are `record`s so mutation does not compile"
is a design.

### Extension: prove the leak

Add to the drill: loop 500,000 times, mutating each key after insertion, and run
under `-Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp`. Confirm
that the process dies, that the dump's dominator tree names the `HashSet`, and
that every retained `Product` has exactly one incoming reference — from the set
you can no longer address it through. That is Topic 79 in miniature.

---

## Practice exercises

### 1 — Easy: break each clause on purpose

Write five tiny classes, each violating exactly **one** clause of the `equals`
contract: reflexive, symmetric, transitive, consistent, null-safe.

For each one:
- Write a `main` that demonstrates the violation in two or three printed lines.
- Then write a second demonstration showing the *collection-level* consequence —
  a `HashSet` with the wrong size, a `contains` that depends on order, a `List`
  and a `Set` disagreeing.

Requirements:
- No two classes may use the same violation mechanism.
- For the "consistent" one, do not use randomness — use a mutable field, because
  that is the version you will actually meet.

### 2 — Medium: the audit (combines Topics 01, 05, 06, 10, 11, 12)

Here is an `orderflow` inventory reservation cache. It contains **six** distinct
defects drawn from Topics 01 through 13. Find them all, state the exact
observable symptom of each in production, and rewrite it.

```java
public class ReservationCache {

    public static class Reservation {
        public Long orderId;
        public String sku;
        public Integer quantity;
        public String status = "HELD";

        @Override public boolean equals(Object o) {
            if (o == null) return false;
            Reservation r = (Reservation) o;
            return orderId == r.orderId && sku.equals(r.sku) && status.equals(r.status);
        }
    }

    private final Set<Reservation> active = new HashSet<>();
    private final Map<String, Integer> heldBySku = new HashMap<>();

    public void hold(Reservation r) {
        active.add(r);
        Integer held = heldBySku.get(r.sku);
        heldBySku.put(r.sku, held + r.quantity);
    }

    public void confirm(Reservation r) {
        r.status = "CONFIRMED";
        active.remove(r);
    }

    public String report() {
        return String.join(",", heldBySku.keySet());
    }
}
```

Hints so you look in the right places: one defect from Topic 01, one from
Topic 12, three from Topic 13, and one that is a `ClassCastException` waiting for
the wrong argument type. For each defect, say whether it throws, corrupts
silently, or leaks — and which of those three is worst to be on call for.

### 3 — Hard: production simulation

Simulate the double-charge incident from Example 2 end to end, then fix it and
prove the fix.

**Part A — reproduce.** Build the broken `WebhookDeduplicator`. Feed it 50,000
synthetic payment webhooks, where 20% are retries of an earlier event (same
gateway and ref, arriving with `status = "PENDING"`). Count how many times
`walletService.credit` is called. Assert that the count is greater than the
number of distinct events — you have just written a *failing* test that proves
the bug exists.

**Part B — quantify the leak.** Run the same load with
`-Xmx256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/dedup.hprof`.
Report:
- the `size()` of the `processed` set versus the number of distinct events;
- the retained size of that set from the heap dump's dominator tree;
- how many of the retained `PaymentEvent` objects are addressable via
  `contains()`. (Write the loop that checks.)

**Part C — fix and re-prove.** Apply the `record`-key fix. Re-run Part A. The
credit count must now equal the distinct-event count exactly. Re-run Part B. The
set size must now equal the distinct-event count exactly.

**Part D — the harder question.** Your fix works in one JVM. Now:
1. Run two instances of the deduplicator against the same event stream. Does the
   fix still hold? Why not?
2. Restart the process mid-stream. Does it still hold?
3. Write down, in three sentences, what the correct production design is and why
   it cannot live in memory at all.

You are anticipating Topic 116. The point is to notice, on your own, that a
correct `equals`/`hashCode` gives you *local* correctness and that distributed
idempotency is a different problem with a different answer.

---

## Interview questions

### Q1 — "What happens if you override `equals` but not `hashCode`?"

**Mid-level answer:** "The object won't work correctly in a `HashMap` or
`HashSet` — you won't be able to find it, because the hash codes will be
different even though the objects are equal."

**Senior answer:** "Two equal objects get identity-derived hash codes, so they
land in different buckets and `HashSet.contains` never reaches the `equals` call.
The important part is the *shape* of the failure: it is silent. No exception, no
warning, no degraded behaviour you would notice. And it is order-dependent in
which structures it breaks — `ArrayList.contains` still works because it only
calls `equals`, `Stream.filter` still works, your `assertEquals` still passes. So
the bug survives every test that does not specifically exercise a hash-based
collection, which is most of them. That is why I want this caught at build time
by ErrorProne's `EqualsHashCode` check or SpotBugs' `HE_EQUALS_USE_HASHCODE`
rather than by review, and why I default value types to `record`s so the pair
cannot drift apart when someone adds a field."

**What separates them:** the mid answer describes the mechanism. The senior
answer describes *why the bug escapes testing* and moves the fix from discipline
to tooling.

**Follow-up:** "Is the reverse — same `hashCode`, different `equals` — also a
contract violation?" No. Unequal objects sharing a hash code is a collision,
which is legal and only a performance question. The implication is one-way, and
knowing the direction is the point of the question.

---

### Q2 — "Why can't a mutable object be a HashMap key?"

**Mid-level answer:** "Because if you change it, the hash code changes and you
can't find it in the map any more."

**Senior answer:** "The map computes the spread hash once at insertion and caches
it in the node — it never recomputes. So after you mutate a hash-relevant field,
lookup computes a new hash, indexes a different bucket, and finds nothing. The
symptom set is specific and worth memorising: `get` returns null, `contains` is
false, `remove` returns false, and `size()` still counts the entry — all at once,
while you are holding the exact object reference. And it is not just a
correctness bug, it is a leak: the entry is still strongly reachable from the
map's table, so nothing gets collected, and the only way to remove it is
`iterator.remove()` or `clear()`. In a Spring codebase the version I actually see
is a JPA entity whose `equals` uses a `@GeneratedValue` ID — null before the
flush, assigned after — which is the same bug with the mutation performed by
Hibernate instead of by application code."

**What separates them:** naming all four symptoms together (they are diagnostic
as a set), identifying it as a leak as well as a correctness bug, and connecting
it to the JPA case where the mutation is not in your code.

**Follow-up:** "How would you enforce it?" They want `record`s, `final` fields, a
separate key type, and — if you are good — the observation that no static
analyser catches this reliably, because it is a usage pattern rather than a class
defect.

---

### Q3 — "Write `equals` for a class with a subclass. Can you keep it symmetric?"

**Mid-level answer:** "Use `instanceof` and compare the fields. If the subclass
adds a field, override `equals` in the subclass and call `super.equals` first."

**Senior answer:** "You can't, and that is a known limitation rather than
something to code around. If the superclass uses `instanceof`, it accepts
subclass instances, so `parent.equals(child)` is true; if the subclass adds a
field to its own `equals`, `child.equals(parent)` is false. Symmetry is gone, and
the consequence is that a `Set`'s contents depend on insertion order.
Switching the superclass to `getClass() != o.getClass()` restores symmetry but
breaks Liskov substitution, and in a Spring stack it specifically breaks
Hibernate, because a lazy association gives you a runtime-generated proxy
subclass whose `getClass()` is not your entity class. So my actual answer is: for
a value type, make it `final` — which `record` does for you — and use composition
instead of inheritance. If I inherited a hierarchy I cannot change, I'd use
`instanceof` in the superclass, not override `equals` in subclasses at all, and
document that subclasses are not value-distinguished."

**What separates them:** knowing it is genuinely unsolvable rather than offering a
fix that quietly breaks something else, and naming the concrete Hibernate
consequence of the `getClass()` approach.

**Follow-up:** "So what does `record` do about it?" `record` is implicitly
`final`, so the problem cannot arise. That is a design answer, not a workaround.

---

### Q4 — "Two objects are `equals` but land in different buckets. Where's the memory leak?"

**Mid-level answer:** "There isn't one — you just can't find the object."

**Senior answer:** "That's the mutable-key case, and the leak is that the entry is
unreachable through the API but perfectly reachable from the GC's point of view.
The `Node` is still linked into the table array, the table is reachable from the
map, the map is reachable from whatever holds it. So the key, the value, and
anything the value transitively references are all strongly retained forever.
`remove(key)` can't free it because `remove` uses the same broken lookup path.
The map grows without bound while its logical content is bounded, which is the
signature of the classic Java leak: not allocation, *retention*. In a heap dump
the map dominates a large retained set and every entry has exactly one incoming
reference. The only escape is `iterator.remove()` or `clear()`, and if you find
yourself writing either as a fix, the design is wrong."

**What separates them:** "Java doesn't leak, it retains" as a framing, and the
observation that `remove` cannot help because it shares the broken path. The mid
answer stops at the correctness bug and misses that this is simultaneously a
memory incident.

**Follow-up:** "How would you find this from a heap dump?" Dominator tree,
largest retained set, path to GC root. Topic 79.

---

### Q5 — "How do you write a good `hashCode`? Is `Objects.hash` fine?"

**Mid-level answer:** "Use `Objects.hash` with all the fields you use in
`equals`. Or let the IDE generate it."

**Senior answer:** "The correctness rule is the only hard rule: every field used
in `equals`, and no field that isn't. Beyond that it is a performance question,
not a contract question. `Objects.hash` is the right default — it is readable and
it cannot drift out of sync if you are disciplined about editing both methods
together — but it is varargs, so every call allocates an `Object[]` and boxes any
primitives you pass. On a key type used in a hot lookup path I'd hand-roll the
classic `int r = 17; r = 31 * r + field;` form, which allocates nothing.
31 is chosen because it is an odd prime and `31 * i` compiles to `(i << 5) - i`,
so the multiply is nearly free. If the hash is expensive and the object is
immutable, cache it in a field the way `String` does — `String` caches its hash
lazily and uses 0 as the 'not computed' sentinel, which is why the empty string
recomputes every time and nobody cares. And the honest answer for most classes is
`record`, which generates a correct pair from all components and removes the
whole class of drift bugs."

**What separates them:** distinguishing the correctness rule from the performance
rule, knowing `Objects.hash` allocates, and reaching for `record` as the default
rather than as a novelty.

**Follow-up:** "When would caching the hash be wrong?" When the object is
mutable — at which point you have Trap 2 anyway, so the real answer is that the
question exposes a design problem.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The contract says equal objects must have equal hash codes, but not the
   reverse. Why is the implication one-way? What would break if it were
   two-way — and could any implementation even satisfy it?

2. `Object.hashCode()`'s default is identity-derived. What is the argument for
   that default rather than, say, always returning 0? Both are contract-legal.

3. You cannot preserve symmetry when a subclass adds a value field. Java could
   have designed `equals` differently to make this work — describe one such
   design and name what it would have cost.

4. `HashSet.contains` on a mutated key returns `false` for both the old value and
   the new value. Explain why the *new* value also fails, given that the object's
   current fields match it exactly.

5. A colleague suggests `HashMap` should recompute hashes on lookup so mutable
   keys "just work". Give three concrete reasons that is a bad idea, at least one
   of which is not about performance.

6. `record` makes the mutable-key bug unexpressible for the record itself. Give a
   case where a `record` key still breaks the contract, and say what it has in
   common with the array-field trap.

7. In JavaScript you cannot override `Map` key equality, so this entire class of
   bug does not exist. What does JavaScript give up by making that choice? Argue
   that Java's design is worth its cost, then argue the opposite.

---

## Quick reference card

### The contract, in five lines

```
equals:    reflexive, symmetric, transitive, consistent, x.equals(null) == false
hashCode:  consistent within a run
           a.equals(b)  =>  a.hashCode() == b.hashCode()      [MANDATORY]
           a.hashCode() == b.hashCode()  =>  nothing           [collisions are legal]
keys:      must be immutable in every field that equals/hashCode reads
```

### The canonical implementation

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;                       // fast path
    if (!(o instanceof Product other)) return false;  // also handles null
    return priceMinor == other.priceMinor            // primitives: ==
        && Objects.equals(sku, other.sku)            // objects: Objects.equals
        && Arrays.equals(barcode, other.barcode);    // arrays: Arrays.equals
}

@Override
public int hashCode() {
    return Objects.hash(sku, priceMinor, Arrays.hashCode(barcode));
}
```

Or, better:

```java
public record Product(String sku, long priceMinor) { }
```

### Field-type cheat sheet

| Field type | In `equals` | In `hashCode` |
|---|---|---|
| primitive (`int`, `long`, `boolean`) | `a == b` | `Objects.hash(a)` or the value itself |
| `float` / `double` | `Float.compare` / `Double.compare` | `Float.hashCode` / `Double.hashCode` |
| object reference | `Objects.equals(a, b)` | `Objects.hashCode(a)` |
| array | `Arrays.equals(a, b)` | `Arrays.hashCode(a)` |
| nested array | `Arrays.deepEquals` | `Arrays.deepHashCode` |
| `Optional` field | don't — `Optional` is a return type (Topic 26) | — |

### Symptom lookup table

| You observe | Suspect |
|---|---|
| `list.contains(x)` true but `set.contains(x)` false | `equals` overridden, `hashCode` not |
| `set.contains(x)` false while holding `x` | mutated key |
| `set.remove(x)` returns false, `size()` unchanged | mutated key |
| `Set` size depends on insertion order | asymmetric or non-transitive `equals` |
| `Stream.distinct()` leaves duplicates | missing or wrong `hashCode` |
| `Collectors.toMap` throws `IllegalStateException: Duplicate key` | `equals` broader than you meant |
| two unsaved JPA entities are equal | `equals` on a null `@GeneratedValue` ID |
| collection grows unboundedly with bounded logical content | mutated key retaining entries |

### Gotchas checklist

- [ ] Override both, or neither. `record` does both for you.
- [ ] `@Override` on every override — it catches `equals(Product)` vs `equals(Object)`.
- [ ] Every field in `hashCode` must be a field in `equals`, and vice versa.
- [ ] Keys must be immutable in all hash-relevant fields.
- [ ] Arrays need `Arrays.equals` / `Arrays.hashCode`. `record`s do not fix this.
- [ ] `float`/`double` fields need `Double.compare`, not `==` (`NaN`, `-0.0`).
- [ ] `getClass()` gives symmetry but breaks Hibernate proxies.
- [ ] Never put an entity with a generated ID into a `Set` before it is persisted.
- [ ] `Objects.hash(...)` allocates. Fine outside hot paths.
- [ ] Wire `ErrorProne EqualsHashCode` or SpotBugs `HE_*` into CI. Do not rely on review.

---

## When would I use this at work?

**1. Reviewing any new value type.**
Someone adds a class that will end up in a `Set`, a `Map` key, a JPA `Set`
association, or a Kafka key. You ask two questions: is it immutable, and is it a
`record`? If the answer to either is no, you ask why. Ninety percent of this
topic's value is in that thirty-second review habit.

**2. Diagnosing "the order is missing but the count is right".**
A support ticket says an order does not appear in a list, while an admin dashboard
counter says it exists. Most engineers start at the database. You recognise the
signature — reachable by iteration, unreachable by lookup, `size()` unaffected —
and go straight to whatever `Set` or `Map` sits between them, looking for a
mutated key. Minutes instead of a day.

**3. Debugging a Hibernate `Set` association that contains duplicates.**
`Set<OrderLine>` has two identical-looking lines. This is Trap 6 nearly every
time: `equals` is based on a generated ID that was null when the line was added
and non-null after the flush. You will meet this in Topic 48, and you will
recognise it because you built it by hand here.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing:** `Objects.hash` boxes; `Double.compare` versus
  `==` on `double` fields; why `Integer` keys work at all.
- **10 — Collections Framework:** which collections use `equals` only
  (`List`, `ArrayDeque`) versus `hashCode` then `equals` (`HashSet`, `HashMap`).
  That split is why the bug hides.
- **12 — HashMap internals:** the cached `final int hash` in each `Node` is the
  entire mechanism behind the mutable-key trap. If Topic 12 is not solid, reread
  the `get` path before continuing.

**This unlocks:**
- **14 — Comparable and TreeMap:** `TreeMap` defines equality by
  `compareTo() == 0` and ignores `equals` entirely — a *different* contract with
  a different set of ways to break it.
- **16 — Sets and EnumSet:** why `EnumSet` and `EnumMap` sidestep all of this
  (enum identity is guaranteed, and they do not hash at all).
- **17 — Immutability:** the general form of the fix. `final` fields, defensive
  copying, and why immutable keys are the only safe keys.
- **27 — Records:** the correct default for any value type, and the compact
  constructor where validation goes.
- **48/49 — Hibernate:** entity equality, the generated-ID trap, and why
  `getClass()` in `equals` breaks lazy proxies.
- **79 — Memory leaks:** the retention this topic produces, found properly with a
  heap dump and a dominator tree.
- **92 — ConcurrentHashMap:** the same contract, with the added problem that a
  key can be mutated by another thread while a lookup is in flight.
- **116 — Idempotency:** why the correct fix for Example 2 is a unique constraint
  in the database rather than a `Set` in a JVM.

---

*Java baseline 21. The `equals`/`hashCode` contract has not changed since Java
1.0 and will not change — it is a compatibility commitment across the entire
ecosystem. What has changed is the tooling around it: `Objects.hash` arrived in
Java 7, pattern-matching `instanceof` in 16, and `record` in 16, and together they
make the correct implementation short enough that there is no excuse for a
hand-written one on a plain value type. Nothing here differs between Java 21 and
Java 25.*
