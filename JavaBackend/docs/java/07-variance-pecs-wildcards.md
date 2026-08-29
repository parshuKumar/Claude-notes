# 07 — Generics III — Variance, PECS, and Wildcard Capture

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

You run a warehouse. You have two kinds of crate.

**A crate you are allowed to take things OUT of.**
The label says "fruit". Inside might be apples, might be pears. You do not know
which. But whatever you pull out, it is definitely *some kind of fruit*. So taking
things out is safe.

Putting something in is **not** safe. If you drop a banana into a crate labelled
"fruit", and that crate is really my apple crate, my apple crate now has a banana in
it. I will find it later and be very confused.

**A crate you are allowed to put things IN to.**
The label says "this crate accepts apples, or anything more general than apples".
Maybe it is the apple crate. Maybe it is the fruit crate. Maybe it is the
everything-crate. An apple fits in all three, so putting an apple in is safe.

Taking something out is **not** safe. You asked for an apple, but you might be
holding the everything-crate, and what you pull out might be a hammer.

That is the whole topic:

> **A crate you read from can be labelled more general. A crate you write to can be
> labelled more specific. Never both at once.**

Java spells "crate you read from" as `? extends`, and "crate you write to" as
`? super`. The rest of this doc is why, and what happens when you get it wrong.

---

## The bridge from what you know

### The partial analogue: TypeScript variance

You already have this. TypeScript 4.7 added explicit variance annotations:

```ts
interface Producer<out T> { get(): T; }      // covariant  — T only comes out
interface Consumer<in T>  { put(x: T): void; } // contravariant — T only goes in
interface Box<in out T>   { get(): T; put(x: T): void; } // invariant — both
```

And even before that, TypeScript inferred variance structurally:

```ts
let apples: string[] = ["gala"];
let things: object[] = apples;   // allowed — TS arrays are covariant
things.push(42 as any);          // and now `apples` contains a number
```

So you already know:
- what covariance means (`Producer<Apple>` is usable as `Producer<Fruit>`),
- what contravariance means (`Consumer<Fruit>` is usable as `Consumer<Apple>`),
- and that covariant mutable containers are unsound.

**Verdict: PARTIAL analogue.** The concepts transfer exactly. Three things do not.

### Difference 1 — where you declare it

TypeScript (and Kotlin, and Scala) let you declare variance **once, on the type**.
`Producer<out T>` is covariant everywhere, forever.

Java has **no declaration-site variance.** A `List<T>` is a `List<T>` and nothing
more. Instead, Java makes you declare variance **at every use site**, in each method
signature, with a wildcard:

```java
void report(List<? extends Product> products) { ... }   // covariant HERE, only
```

This is why Java signatures are noisier than yours. It is also more flexible: the
same `List` can be used covariantly in one method and contravariantly in another.

### Difference 2 — why Java generics are invariant

In TypeScript, unsoundness is free. The types are erased and *nothing checks
anything at runtime*, so a bad assignment just produces wrong data silently.

Java made the opposite choice for generics: **`List<String>` is simply not a
`List<Object>`, and the compiler refuses.** The reason is Topic 06 — erasure.

At runtime, a `List<String>` and a `List<Object>` are the *same class*, `ArrayList`.
There is no type token inside. So if Java allowed the assignment, there would be no
runtime check available to stop the bad write:

```java
List<String> names = new ArrayList<>();
List<Object> things = names;      // COMPILE ERROR — and thank goodness
things.add(42);                   // would have been an unchecked, undetectable write
String first = names.get(0);      // would ClassCastException here, far from the cause
```

Java's compiler is the only line of defence, so the compiler is strict.

### Difference 3 — arrays

Java arrays are **reified**: an array object remembers its component type at runtime.
And Java arrays *are* covariant — a language design decision from 1995, before
generics existed, made so that `Arrays.sort(Object[])` could sort anything.

Because the type is remembered, Java can and does check every write:

```java
Object[] things = new String[3];   // legal — arrays are covariant
things[0] = 42;                    // compiles fine, throws at runtime
```

The throw is `java.lang.ArrayStoreException: java.lang.Integer`.

That is the trade in one image:

| | Variance | Runtime type known? | Bad write caught |
|---|---|---|---|
| Java generics | invariant (wildcards opt in) | **no** (erased) | at **compile** time |
| Java arrays | covariant, always | **yes** (reified) | at **runtime**, as `ArrayStoreException` |
| TS arrays | covariant, always | no | **never** |

Josh Bloch (who wrote the collections framework) calls array covariance a mistake.
It is: it converts a type error into a runtime exception, and it costs a type check
on *every single array store* whose static type could be a supertype.

---

## What is this?

Three words, defined once:

**Invariant** — `List<String>` and `List<Object>` have no subtype relationship at
all. Neither can be used where the other is expected. This is Java's default.

**Covariant** — if `String` is a subtype of `Object`, then `Producer<String>` is
usable as a `Producer<Object>`. Java expresses this with `? extends`.

**Contravariant** — if `String` is a subtype of `Object`, then `Consumer<Object>` is
usable as a `Consumer<String>`. Java expresses this with `? super`.

A **wildcard** (`?`) is Java's way of saying "some specific type that I am not going
to name". Not "any type" — *one* type, unknown to the compiler.

**Wildcard capture** is what the compiler does when it needs to reason about that
unknown type: it invents a temporary name for it, usually printed as `CAP#1`, and
type-checks against that name.

**PECS** is the mnemonic: **P**roducer **E**xtends, **C**onsumer **S**uper. If the
parameter produces values for you, use `extends`. If it consumes values from you,
use `super`. It is a mnemonic, not a rule — the actual rule is "what direction do
the values flow?", and once you think in flows you will not need the mnemonic.

---

## Why does it matter?

Three concrete things, in ascending order of how much they cost you.

**1. Your APIs are unusable without it.**
Write `void priceAll(List<Product> items)` and a caller holding a
`List<DigitalProduct>` cannot call it. They will either copy the list (allocation
and a bug when they forget the copy is a snapshot) or add a cast and a
`@SuppressWarnings`. Both are your fault, in your signature.

**2. Compile errors that read like line noise.**
`incompatible types: CAP#1 cannot be converted to CAP#2` is the single most
confusing error message a Java newcomer meets. Ten minutes of this topic turns it
into a one-second diagnosis.

**3. `ArrayStoreException` in production.**
It appears when generic code passes an array through an `Object[]` parameter. The
stack trace points at a library, not at your code, and the fix is at the call site
three frames up.

---

## Syntax breakdown

Only constructs that are genuinely new. Topic 05 covered `<T>`, bounds like
`<T extends Number>`, and generic methods; Topic 06 covered erasure and raw types.

### The three wildcard forms

```java
List<?>              anything;      // unbounded  — "a List of some unknown type"
List<? extends Product> producers;  // upper bounded — unknown type, at most Product
List<? super  Product> consumers;   // lower bounded — unknown type, at least Product
```

| Form | Read as | You may GET | You may PUT |
|---|---|---|---|
| `List<?>` | list of *some* type | `Object` | nothing except `null` |
| `List<? extends Product>` | list of Product-or-subtype | `Product` | nothing except `null` |
| `List<? super Product>` | list of Product-or-supertype | `Object` | `Product` and its subtypes |
| `List<Product>` | list of exactly Product | `Product` | `Product` and its subtypes |

The two `nothing except null` rows are the part that surprises people. Read the
reason from the ELI5 crate: the compiler does not know *which* subtype the list
holds, so it cannot prove any value you offer is acceptable. `null` is the sole
exception because `null` is a member of every reference type.

### `? extends` vs a plain bound — they are different things

```java
<T extends Product> void a(List<T> items)     // T is NAMED. You can use it.
void b(List<? extends Product> items)         // the type is UNNAMED.
```

Both accept the same arguments. The difference is whether the method body (and the
return type) can refer to the type. Use the named form when you need the name; use
the wildcard when you do not, because it is a smaller, clearer signature.

### Wildcards nest, and the nesting is not intuitive

```java
List<List<Product>>            exactly;   // list of lists of exactly Product
List<? extends List<Product>>  outerCo;   // unknown list-type, holding exact Products
List<List<? extends Product>>  innerCo;   // exact List type, holding Product-or-sub
```

`List<List<String>>` is **not** a `List<List<Object>>`, and it is **not** a
`List<? extends List<Object>>` either — invariance applies at every level.
`List<List<String>>` *is* a `List<? extends List<String>>`.

### The capture-helper idiom

This is the one piece of syntax you will not guess. You cannot write into a
`List<?>`, so this fails:

```java
public static void swapFirstTwo(List<?> list) {
    Object tmp = list.get(0);
    list.set(0, list.get(1));    // COMPILE ERROR
    list.set(1, tmp);            // COMPILE ERROR
}
```

The fix is to give the unknown type a name by delegating to a generic method. The
compiler *captures* the wildcard into the method's type variable `T`:

```java
public static void swapFirstTwo(List<?> list) {
    swapHelper(list);                    // capture happens here
}

private static <T> void swapHelper(List<T> list) {
    T tmp = list.get(0);
    list.set(0, list.get(1));            // fine — the type has a name now
    list.set(1, tmp);
}
```

Nothing was cast. Nothing was suppressed. The compiler simply needed a name to hang
its reasoning on, and the private helper supplies one. This is exactly what
`java.util.Collections.swap` does internally.

### The recursive bound with `? super`

You saw `<T extends Comparable<T>>` in Topic 05. Production code almost always wants
the looser version:

```java
public static <T extends Comparable<? super T>> T max(Collection<? extends T> items)
```

Why `? super T`? Because if `PhysicalProduct extends Product` and only `Product`
implements `Comparable<Product>`, then `PhysicalProduct` is **not** a
`Comparable<PhysicalProduct>` — it is a `Comparable<Product>`. The tight bound
rejects it; the `? super` bound accepts it. This exact signature is what
`Collections.max` uses, and now you know why it looks like that.

---

## Example 1 — minimal

```java
import java.util.*;

public class VarianceMinimal {

    // PRODUCER: we only READ from this list.
    static double sumAll(List<? extends Number> numbers) {
        double total = 0;
        for (Number n : numbers) {     // safe: whatever it holds, it is a Number
            total += n.doubleValue();
        }
        // numbers.add(1);             // would not compile — and that is correct
        return total;
    }

    // CONSUMER: we only WRITE into this list.
    static void fillWithIntegers(List<? super Integer> sink) {
        for (int i = 1; i <= 3; i++) {
            sink.add(i);               // safe: an Integer fits any Integer-supertype
        }
        // Integer first = sink.get(0);  // would not compile — might be List<Object>
        Object first = sink.get(0);      // this is all you get back
    }

    public static void main(String[] args) {
        List<Integer> ints    = List.of(1, 2, 3);
        List<Double>  doubles = List.of(1.5, 2.5);

        System.out.println(sumAll(ints));       // works
        System.out.println(sumAll(doubles));    // also works — this is the payoff

        List<Number> sink = new ArrayList<>();
        fillWithIntegers(sink);                 // List<Number> accepted
        List<Object> objectSink = new ArrayList<>();
        fillWithIntegers(objectSink);           // List<Object> accepted too
        System.out.println(sink + " " + objectSink);
    }
}
```

Delete the `? extends` from `sumAll` and the two calls stop compiling. That is the
whole value proposition, in one edit you should actually perform.

---

## Example 2 — production scenario

`orderflow` prices an order. Pricing is a chain of rules — a base price, a volume
discount, a loyalty adjustment, tax. Each rule reads the order lines and *emits*
adjustments into a shared sink. Different callers want the adjustments collected
into different containers: the checkout path wants a `List<PriceAdjustment>` to
show the customer; the audit path wants a `List<AuditEvent>` where
`PriceAdjustment` is one kind of `AuditEvent`.

```java
package com.orderflow.pricing;

import java.util.*;

// ---------- domain ----------

sealed interface OrderLine permits PhysicalLine, DigitalLine {
    String sku();
    int quantity();
    long unitPriceMinor();          // money as minor units — Topic 01, Trap 5
}

record PhysicalLine(String sku, int quantity, long unitPriceMinor, double weightKg)
        implements OrderLine { }

record DigitalLine(String sku, int quantity, long unitPriceMinor, String downloadUrl)
        implements OrderLine { }

// PriceAdjustment is one kind of auditable event.
interface AuditEvent { String description(); }

record PriceAdjustment(String reason, long deltaMinor) implements AuditEvent {
    public String description() { return reason + " " + deltaMinor; }
}

// ---------- the API surface ----------

public interface PricingRule {

    /**
     * Reads order lines, writes adjustments.
     *
     *  lines  is a PRODUCER of OrderLine   -> ? extends
     *  sink   is a CONSUMER of adjustments -> ? super
     */
    void apply(List<? extends OrderLine> lines,
               Collection<? super PriceAdjustment> sink);
}
```

Now the two implementations, and the two very different callers.

```java
package com.orderflow.pricing;

import java.util.*;

public final class VolumeDiscountRule implements PricingRule {

    private final int threshold;
    private final int percentOff;

    public VolumeDiscountRule(int threshold, int percentOff) {
        this.threshold  = threshold;
        this.percentOff = percentOff;
    }

    @Override
    public void apply(List<? extends OrderLine> lines,
                      Collection<? super PriceAdjustment> sink) {

        for (OrderLine line : lines) {          // READ — allowed
            if (line.quantity() >= threshold) {
                long gross = line.unitPriceMinor() * line.quantity();
                long delta = -(gross * percentOff) / 100;
                sink.add(new PriceAdjustment(                 // WRITE — allowed
                        "volume>" + threshold + " on " + line.sku(), delta));
            }
        }
        // lines.add(...)  would not compile. Correct: a rule must not mutate the order.
    }
}
```

```java
package com.orderflow.pricing;

import java.util.*;

public final class PricingEngine {

    private final List<PricingRule> rules;

    public PricingEngine(List<PricingRule> rules) {
        this.rules = List.copyOf(rules);
    }

    public void priceInto(List<? extends OrderLine> lines,
                          Collection<? super PriceAdjustment> sink) {
        for (PricingRule rule : rules) {
            rule.apply(lines, sink);
        }
    }

    public static void main(String[] args) {
        PricingEngine engine = new PricingEngine(List.of(
                new VolumeDiscountRule(10, 5),
                new VolumeDiscountRule(50, 12)));

        // Caller 1 — checkout. It holds a list of ONE concrete line type.
        List<PhysicalLine> shipment = List.of(
                new PhysicalLine("SKU-4471", 60, 1999L, 0.4),
                new PhysicalLine("SKU-9002", 3,  4500L, 1.2));

        List<PriceAdjustment> forCustomer = new ArrayList<>();
        engine.priceInto(shipment, forCustomer);        // needs ? extends OrderLine

        // Caller 2 — audit. It collects into a broader container.
        List<AuditEvent> auditTrail = new ArrayList<>();
        engine.priceInto(shipment, auditTrail);         // needs ? super PriceAdjustment

        System.out.println(forCustomer);
        System.out.println(auditTrail);
    }
}
```

Now count what each wildcard bought:

| Wildcard | Removing it breaks | Business consequence |
|---|---|---|
| `? extends OrderLine` on `lines` | Caller 1 — `List<PhysicalLine>` is rejected | Every caller must build a `List<OrderLine>` copy. Copies drift from the real order; someone eventually prices a stale snapshot. |
| `? super PriceAdjustment` on `sink` | Caller 2 — `List<AuditEvent>` is rejected | The audit path collects into its own list and merges. Two lists that must be kept in sync is a bug factory. |
| Both together | nothing compiles for anyone but the exact types | The API is technically correct and practically unusable. |

And note what the wildcards *prevent*: a rule physically cannot add a line to the
order or read a `PriceAdjustment` back out of the sink. Those are design invariants,
enforced by the compiler, for free, because you chose the right wildcard.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — invariant parameter, unusable API

**Wrong:**
```java
public long totalWeight(List<PhysicalLine> lines) { ... }
```
called as
```java
List<OrderLine> lines = order.lines();
long w = totalWeight(lines);
```

**Exact symptom — a compile error of this shape:**
```
error: incompatible types: List<OrderLine> cannot be converted to List<PhysicalLine>
```
and the mirror-image case, `List<PhysicalLine>` into a `List<OrderLine>` parameter,
gives the same shape. Exact wording varies slightly by javac version; the phrase
`incompatible types` and two `List<...>` types is the signature to recognise.

**Root cause:** generics are invariant. `List<Sub>` is not a `List<Super>` and
never will be, because erasure leaves no runtime check to make it safe.

**Fix:** decide the direction of flow. If the method only reads:
```java
public long totalWeight(List<? extends OrderLine> lines)
```

**Anti-fix to refuse in review:** `totalWeight(new ArrayList<>(lines))` at the call
site, or a raw `List` parameter. Both compile. Both move the problem.

---

### Trap 2 — trying to write into a producer

**Wrong:**
```java
static void addFreeGift(List<? extends OrderLine> lines) {
    lines.add(new DigitalLine("GIFT-001", 1, 0L, "https://..."));
}
```

**Exact symptom:**
```
error: no suitable method found for add(DigitalLine)
    method List.add(CAP#1) is not applicable
      (argument mismatch; DigitalLine cannot be converted to CAP#1)
  where CAP#1 is a fresh type-variable:
    CAP#1 extends OrderLine from capture of ? extends OrderLine
```

That `CAP#1` is wildcard capture made visible. The compiler invented a name for the
unknown type so it could tell you why your value does not fit it.

**Root cause:** the list might really be a `List<PhysicalLine>`. Adding a
`DigitalLine` would corrupt it. The compiler cannot know, so it refuses everything
except `null`.

**Fix:** if the method genuinely needs to add, it is a consumer, not a producer.
Change the signature to `List<? super DigitalLine>` — or, more honestly, to
`List<OrderLine>`, and make the caller's list actually be one. Do not
`@SuppressWarnings` your way past this; the compiler is right.

---

### Trap 3 — array covariance, `ArrayStoreException`

**Wrong:**
```java
PhysicalLine[] shipment = new PhysicalLine[10];
OrderLine[] view = shipment;                      // legal: arrays are covariant
view[0] = new DigitalLine("SKU-1", 1, 0L, "u");   // compiles cleanly
```

**Exact symptom, at runtime:**
```
java.lang.ArrayStoreException: com.orderflow.pricing.DigitalLine
	at com.orderflow.pricing.Shipping.pack(Shipping.java:41)
```
The message names the class you tried to store, not the array type, which is
mildly unhelpful the first time you see it.

**Root cause:** the array object remembers it is really a `PhysicalLine[]`. Every
store through a reference whose static type is a supertype gets a runtime check.
This one failed.

**Where it actually bites you:** varargs and `toArray`. A `List<PhysicalLine>` gives
you `toArray(new PhysicalLine[0])`, but generic library code often has only an
`Object[]`, hands it around, and something writes the wrong type into it three frames
away from your code.

**Fix:** prefer `List` over arrays in any API that has a type parameter. Generics
and arrays do not mix — Topic 06 told you `new T[]` is illegal for exactly this
family of reasons. If you must have an array, do not widen its static type.

---

### Trap 4 — `? super` on a return type

**Wrong:**
```java
public List<? super PriceAdjustment> collectAdjustments(Order order) { ... }
```

**Exact symptom:** no compile error here. The pain is at every call site:
```java
var adjustments = engine.collectAdjustments(order);
long total = adjustments.stream()
        .mapToLong(a -> a.deltaMinor())   // error: cannot find symbol: deltaMinor()
        .sum();
```
because the element type is `Object`. Callers "fix" it with
`(PriceAdjustment) a`, and then a `ClassCastException` appears the day someone puts
something else in the list.

**Root cause:** a wildcard on a *return* type pushes the unknown outward. The
caller now has strictly less information than you did. Wildcards belong on
**parameters**, where they widen what you accept; on return types they narrow what
you give.

**Fix:**
```java
public List<PriceAdjustment> collectAdjustments(Order order)
```

**The rule:** be liberal in what you accept (wildcards on parameters), specific in
what you return (no wildcards). Yes — this is Postel's law, and you already apply it
to HTTP APIs.

---

### Trap 5 — capture mismatch across two calls

**Wrong:**
```java
static void rotate(List<?> a, List<?> b) {
    b.addAll(a);      // error
}
```

**Exact symptom:**
```
error: incompatible types: List<CAP#1> cannot be converted to Collection<? extends CAP#2>
```

**Root cause:** each `?` is captured **separately**. `CAP#1` and `CAP#2` are two
different fresh type variables, and the compiler has no evidence they are the same
type — because they need not be. `rotate(stringList, integerList)` is a legal call.

**Fix:** name the type once so both parameters share it.
```java
static <T> void rotate(List<T> a, List<T> b) { b.addAll(a); }
```

**How to recognise this class of error instantly:** if you see two `CAP#` numbers in
one message, you wrote two wildcards where you meant one type variable.

---

## Hands-on proof

Everything here is a command **you** run. I do not have a JVM and will not print
output and call it real. What follows is the exact file, the exact command, what to
look for, and how to read every outcome.

### Setup

```bash
mkdir -p ~/java-lab/07 && cd ~/java-lab/07
java --version      # expect 21 or 25
```

### Proof 1 — invariance is a compile-time wall

`Invariance.java`:
```java
import java.util.*;

public class Invariance {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>();
        List<Object> things = names;    // the line under test
        things.add(42);
        String first = names.get(0);
        System.out.println(first);
    }
}
```

```bash
javac Invariance.java
```

**What to look for:** compilation must FAIL, at the `List<Object> things = names;`
line.

| What you see | What it means |
|---|---|
| `error: incompatible types: List<String> cannot be converted to List<Object>` | Expected. Java generics are invariant, and the compiler stopped you before any runtime damage. |
| It compiles | You are not compiling the file you think you are, or you edited the type to `List<?>`. Re-check. |
| A *warning* rather than an error | You are using a raw `List` somewhere. Add `-Xlint:rawtypes -Xlint:unchecked` and read what it says. |

Now change the line to `List<? extends Object> things = names;` and recompile. It
compiles — and `things.add(42)` now fails instead. Two different errors, same
underlying protection.

### Proof 2 — arrays are covariant and pay at runtime

`ArrayVariance.java`:
```java
public class ArrayVariance {
    public static void main(String[] args) {
        String[] names = new String[3];
        Object[] view  = names;             // covariant: compiles
        System.out.println("assignment ok, array class = " + view.getClass().getName());
        view[0] = Integer.valueOf(42);      // compiles; runtime decides
        System.out.println("store ok — you should not see this line");
    }
}
```

```bash
java ArrayVariance.java
```

**What to look for:** the first `println` runs, then an exception.

| What you see | What it means |
|---|---|
| `array class = [Ljava.lang.String;` then `ArrayStoreException: java.lang.Integer` | Expected. The array remembered its real component type and refused the store. This is reification. |
| No exception | You changed `String[]` to `Object[]` at the allocation. The static type is irrelevant; the *allocated* type is what is checked. |
| A compile error | You added a generic type parameter somewhere. Arrays of a type variable are illegal — that is Topic 06. |

**How to read it:** compare with Proof 1. Same category of mistake. Generics caught
it at compile time and cost nothing at runtime; arrays caught it at runtime and cost
a type check on every store. That comparison *is* the argument that array covariance
was a mistake.

### Proof 3 — see wildcard capture in the error message

`Capture.java`:
```java
import java.util.*;

public class Capture {
    static void addOne(List<? extends Number> xs) {
        xs.add(1);
    }
    static void copy(List<?> a, List<?> b) {
        b.addAll(a);
    }
}
```

```bash
javac -Xdiags:verbose Capture.java
```

**What to look for:** two errors, and the tokens `CAP#1` and `CAP#2`.

| What you see | What it means |
|---|---|
| `CAP#1 extends Number from capture of ? extends Number` | The compiler named the unknown type to explain the failure. `add` needs a `CAP#1`; you offered an `Integer`; it cannot prove they match. |
| Two different `CAP#` numbers in the `copy` error | Each wildcard captured separately. Two wildcards are never assumed to be the same type. This is the fingerprint of Trap 5. |
| No `CAP#` at all, just `incompatible types` | Drop `-Xdiags:verbose` and add it back; the compact diagnostic mode hides the capture detail. |

### Proof 4 — wildcards erase to their bound

`Erased.java`:
```java
import java.util.*;

public class Erased {
    public void a(List<? extends Number> xs) { }
    public void b(List<? super Integer> xs)  { }
    public void c(List<?> xs)                { }
    public <T extends Number> void d(List<T> xs) { }
}
```

```bash
javac Erased.java
javap -s Erased.class          # erased descriptors
javap -v Erased.class | grep -A1 Signature
```

**What to look for:**

| Where | What to look for | What it means |
|---|---|---|
| `javap -s` | every one of the four methods has the descriptor `(Ljava/util/List;)V` | All four erase to the same thing. At the bytecode level there is no variance at all — this is Topic 06's erasure, and it is exactly why the compiler must be strict. |
| `javap -v` … `Signature` | strings like `(Ljava/util/List<+Ljava/lang/Number;>;)V` | Generic information *is* kept, in a side attribute, for reflection and for compiling against the class. `+` means `? extends`, `-` means `? super`, `*` means `?`. |

**How to read it:** the `Signature` attribute is metadata; the JVM does not enforce
it. Only `javac` reads it. So the guarantee is entirely compile-time, which is the
one-sentence answer to "why can't Java just check variance at runtime?".

### Proof 5 — the capture helper compiles when the direct version does not

`SwapProof.java`:
```java
import java.util.*;

public class SwapProof {

    // Uncomment to see it fail:
    // static void swapDirect(List<?> list) {
    //     Object tmp = list.get(0);
    //     list.set(0, list.get(1));
    //     list.set(1, tmp);
    // }

    static void swap(List<?> list) { swapHelper(list); }

    private static <T> void swapHelper(List<T> list) {
        T tmp = list.get(0);
        list.set(0, list.get(1));
        list.set(1, tmp);
    }

    public static void main(String[] args) {
        List<String> skus = new ArrayList<>(List.of("SKU-4471", "SKU-9002"));
        swap(skus);
        System.out.println(skus);
    }
}
```

```bash
java SwapProof.java
# then uncomment swapDirect and:
javac SwapProof.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| The program prints the two SKUs in swapped order | The capture helper worked. No cast, no `@SuppressWarnings` — the private method's `T` is the name the compiler needed. |
| `swapDirect` fails with `List.set(CAP#1,...)` errors | Expected. Same code, no name for the type, no compilation. The only difference between the two is a name. |

> **Version note:** wildcard capture and inference have been refined across releases
> (notably Java 8's target typing and the Java 18-era inference fixes). A snippet
> that fails on one JDK may compile on another. If you hit a disagreement, settle it
> with `javac --version` on both machines before assuming your understanding is
> wrong.

---

## Practice exercises

Write real files, compile them, run them, and keep the errors you hit.

### 1 — Easy: fix the signatures

Here are five signatures. For each one, state (a) whether values flow **in**, **out**,
or **both**, (b) what wildcard it should have, and (c) one concrete caller it
currently rejects.

```java
void archive(List<Order> orders);                       // reads only
void collectSkus(List<OrderLine> lines, Set<String> out); // reads lines, writes out
long sum(Collection<Long> amounts);                     // reads only
void register(Map<String, Product> catalog);            // writes only
List<Product> filter(List<Product> in, Predicate<Product> p);  // reads in, returns new
```

Then write a `main` that calls each fixed version with a subtype collection, and
confirm it compiles. Note which of the five needed **no** wildcard, and say why.

### 2 — Medium: combines Topics 01, 05 and 06

Implement this method, then answer the questions.

```java
public static <T extends Comparable<? super T>> T maxOf(Collection<? extends T> items)
```

Requirements:
- Throw `NoSuchElementException` on an empty collection. Do not return `null`.
- Do not use `Collections.max`, `Stream.max`, or `sorted`.
- It must compile and run against `List<Integer>`, `List<String>`, and a
  `List<PhysicalLine>` where `PhysicalLine` implements `Comparable<OrderLine>`
  (**this last one is the point** — the tight bound `<T extends Comparable<T>>`
  rejects it).

Then answer, in writing:
1. Why `? super T` in the bound rather than `Comparable<T>`? Give the exact call
   that breaks under the tight bound.
2. Why `? extends T` on the parameter rather than `Collection<T>`?
3. Compile with `javac -s` — sorry, with `javap -s` on the class — and state what
   the erased descriptor of `maxOf` is. Which of your two wildcards survives
   erasure? (Topic 06.)
4. If `T` were `Long` and the collection had 10 million elements, what does Topic 01
   tell you about the cost of every `compareTo` call in your loop? Would you change
   anything? Justify either answer.

### 3 — Hard: production simulation on `orderflow`

Build a small pricing pipeline and then break it deliberately.

**Part A — build.** Implement `PricingRule`, `PricingEngine`, `VolumeDiscountRule`
and a `LoyaltyRule` from Example 2. Add a third rule, `TaxRule`, that must run
**last** and needs to see the adjustments the earlier rules produced. Decide: does
`TaxRule.apply` need `List<? extends PriceAdjustment>` as an extra parameter, or
does the sink need to become readable? Write down the trade-off before you code it.

**Part B — break it three ways.** Produce each of these and record the exact
compiler or runtime message:
1. A rule that tries to add an `OrderLine` to the `lines` parameter.
2. A caller that passes a `List<AuditEvent>` where `List<? extends PriceAdjustment>`
   is required.
3. An `ArrayStoreException` reached through this pipeline. You will need an array
   somewhere — a `PriceAdjustment[]` returned from a `toArray` call, widened to
   `AuditEvent[]`, then written to.

**Part C — the API review.** Someone proposes changing `apply` to:
```java
<T extends OrderLine> void apply(List<T> lines, List<PriceAdjustment> sink);
```
Write the review comment. Is `T` used? Does it accept fewer callers, more, or the
same? Which of the two parameters got *worse*? Would you approve it?

**Part D — the honest bit.** Now argue against yourself. Wildcards made this API
accept more callers, at the cost of a signature that a new team member has to stop
and parse. Name one concrete situation in `orderflow` where you would deliberately
write the invariant `List<OrderLine>` instead, and defend it.

---

## Interview questions

### Q1 — "Why is `List<String>` not a `List<Object>`, when `String[]` *is* an `Object[]`?"

**Mid-level answer:** "Generics are invariant. Arrays are covariant. That is just how
Java works."

**Senior answer:** "Both design decisions are about where the safety check can live.
Arrays are reified — the array object knows its component type — so the JVM can check
every store and throw `ArrayStoreException`. Covariance is unsound, but at least it
is detected. Generics are erased, so there is no runtime type to check against; the
only place a bad write could be caught is at compile time, which means the compiler
has to be strict, which means invariance. Array covariance was added in Java 1.0 so
that `Arrays.sort(Object[])` could exist before generics did — it is widely regarded
as a mistake, and it costs a check on every array store through a widened reference."

**What separates them:** the mid answer states two facts. The senior answer explains
that they are the *same* decision resolved differently under different runtime
information, and can date the historical reason.

**Follow-up the interviewer asks:** "So where does `ArrayStoreException` actually
bite in real code?" They want to hear varargs, `toArray`, or generic library code
passing an `Object[]` — not a textbook two-liner.

---

### Q2 — "Explain PECS. Then explain it without using the mnemonic."

**Mid-level answer:** "Producer Extends, Consumer Super. If you read from it use
`extends`, if you write to it use `super`."

**Senior answer:** "Without the mnemonic: ask which direction values cross the
boundary. If values come *out* of the parameter into my method, I only need a
guarantee about their upper bound, so `? extends` — I can call `Product` methods on
whatever comes out. If values go *in* from my method, I need a guarantee that the
container can hold what I am giving it, so `? super`. If both directions happen, no
wildcard is possible and the parameter must be invariant — which is a design signal
that the parameter is doing two jobs. And the corollary people forget: wildcards
belong on parameters. On a return type they just push an unknown out to the caller,
who then casts."

**What separates them:** deriving the rule from data flow rather than reciting it,
plus the two corollaries (both-directions means invariant; never on return types).

**Follow-up:** "`Collections.copy(List<? super T> dest, List<? extends T> src)` — walk
me through why each wildcard is where it is." A senior candidate should also note the
signature is a single generic method with `T` naming the relationship between the two
parameters, which is why `T` exists at all.

---

### Q3 — "This does not compile. Explain the error, and fix it two ways."

```java
static void addDefaultLine(List<? extends OrderLine> lines) {
    lines.add(new DigitalLine("GIFT-001", 1, 0L, "https://x"));
}
```

**Mid-level answer:** "You cannot add to a `? extends` list. Change it to
`List<OrderLine>`."

**Senior answer:** "The error mentions `CAP#1` — a fresh type variable the compiler
invented for the captured wildcard. The list might really be a `List<PhysicalLine>`,
so no specific value can be proved safe; only `null` is assignable to every reference
type. Two fixes with different meanings: `List<? super DigitalLine>` if the method's
job is genuinely to consume, which keeps it usable by callers holding
`List<OrderLine>` or `List<Object>`; or plain `List<OrderLine>` if the parameter is
really the order's own line list and the caller should be holding the exact type. I
would pick based on whether this method is a library helper or part of the order
aggregate — and I would not reach for `@SuppressWarnings`, because the compiler is
correct here."

**What separates them:** naming capture, explaining the `null` exception, and
choosing between fixes on design grounds rather than on whichever compiles.

**Follow-up:** "Why is `null` allowed?"

---

### Q4 — "When would you write `List<?>` instead of `List<Object>`?"

**Mid-level answer:** "They are basically the same, `List<?>` is shorter."

**Senior answer:** "They are opposites. `List<Object>` is a list that genuinely holds
`Object` — you can add anything to it, and only a `List<Object>` can be passed. It is
invariant, so `List<String>` is rejected. `List<?>` is a list of some unknown single
type — you can pass any `List` at all, and you can add nothing but `null`. So
`List<?>` is the right parameter type for a method that only inspects size, iterates
as `Object`, or clears. I use it for utility code, and I reach for `List<? extends
Something>` the moment I need to call a method on the elements."

**What separates them:** understanding that `List<?>` is about *what callers you
accept*, and `List<Object>` is about *what the list holds* — and that the second is
almost never what you meant.

**Follow-up:** "What about raw `List`?" The expected answer: raw types disable
generic checking entirely, including on unrelated methods of the same object, and
produce unchecked warnings — it is not a shorter `List<?>`, it is an opt-out from the
type system. (Topic 06.)

---

### Q5 — "You are designing a repository interface for `orderflow`. Write `saveAll`."

**Mid-level answer:**
```java
void saveAll(List<Order> orders);
```

**Senior answer:**
```java
<S extends T> List<S> saveAll(Iterable<S> entities);   // roughly Spring Data's shape
// or, if the return is not needed:
void saveAll(Collection<? extends T> entities);
```
"`Collection` rather than `List` because I do not need ordering or index access and I
should not force the caller to have one. `? extends T` because the parameter is a
pure producer — I read entities out of it and never add. If I need to return the
saved entities with their generated IDs, I name the type variable instead of using a
wildcard, because the return type must be specific — `List<? extends T>` coming back
would make every caller cast. That is the general rule: wildcards widen parameters,
never return types."

**What separates them:** choosing the widest sensible *interface* as well as the
right variance, and articulating the parameter-vs-return-type asymmetry unprompted.

**Follow-up:** "Spring Data's actual signature returns `<S extends T> List<S>`. Why a
type variable there rather than a wildcard?" Because the caller needs the concrete
subtype back — a wildcard would erase exactly the information the caller came for.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Java could have made arrays invariant in Java 5, when generics arrived, and kept
   the old behaviour only for raw code. It did not. What would have broken, and was
   the compatibility argument stronger than the safety argument? Argue both sides.

2. Kotlin has declaration-site variance (`class Producer<out T>`) *and* use-site
   variance (`Producer<out T>` as a parameter). Java has only use-site. Name one
   concrete advantage Java's choice has, not just the disadvantages.

3. `List<? extends Product>` lets you add `null` but nothing else. Is `null` actually
   safe there? Construct the argument that it is, then construct the argument that
   permitting it was a mistake.

4. Why does `Collections.max` use `Comparable<? super T>` rather than
   `Comparable<T>`? Give a concrete `orderflow` class hierarchy where the difference
   decides whether your code compiles.

5. Two wildcards in one signature capture to two different types. But a caller might
   genuinely be passing the same list twice. Should the compiler be smarter here?
   What would it cost to make it so?

6. TypeScript's arrays are covariant and unsound with *no* runtime check at all —
   strictly worse than Java's arrays. Yet TypeScript codebases do not seem to
   suffer for it in the way you would predict. Why not? What does that tell you
   about where variance bugs actually come from?

7. You have a `Map<String, List<? extends OrderLine>>`. Explain, precisely, what you
   can and cannot do with a value pulled out of that map, and why the wildcard being
   nested changes nothing about the rules.

---

## Quick reference card

### The three wildcards

```java
List<?>                  // unknown type. Read as Object. Write only null.
List<? extends Product>  // Product or a subtype. Read as Product. Write only null.
List<? super  Product>   // Product or a supertype. Read as Object. Write Product+subs.
List<Product>            // exactly Product. Read and write Product.
```

### PECS, as a decision table

| The parameter... | Wildcard | Example from the JDK |
|---|---|---|
| only gives values to me | `? extends T` | `Collections.copy(dest, List<? extends T> src)` |
| only takes values from me | `? super T` | `Collections.copy(List<? super T> dest, src)` |
| does both | none — invariant | `Collections.swap(List<?>, int, int)` uses capture instead |
| I only need its size/iteration as Object | `?` | `Collections.frequency(Collection<?>, Object)` |
| is a return type | **never a wildcard** | `List<T> subList(...)` |

### Rules that hold without exception

- Generics are **invariant**. Wildcards are the only way to opt out, at the use site.
- Arrays are **covariant** and **reified**. Bad stores throw `ArrayStoreException`.
- You can put `null` into anything. That is the only universal write.
- Two `?` in one signature are two different types. Name a `T` if they must match.
- Wildcards on parameters. Type variables on return types.
- If a type variable appears **once** in a signature, it should probably be a
  wildcard instead.

### Error-message decoder

| Message fragment | What it means |
|---|---|
| `incompatible types: List<A> cannot be converted to List<B>` | invariance — you need a wildcard on the parameter |
| `CAP#1 extends X from capture of ? extends X` | you tried to write into a producer |
| two different `CAP#` numbers | two wildcards where you meant one type variable |
| `ArrayStoreException: <class>` | array covariance — check for a widened array reference |
| `unchecked call to ... as a member of the raw type` | you have a raw type; that is Topic 06, not this one |

### Signature-attribute symbols (`javap -v`)

```
+   ? extends       Ljava/util/List<+Ljava/lang/Number;>;
-   ? super         Ljava/util/List<-Ljava/lang/Integer;>;
*   ?               Ljava/util/List<*>;
```

---

## When would I use this at work?

**1. The first time you publish an interface other teams implement.**
Every parameter is a decision about who can call you. Getting `? extends` right on a
`Collection` parameter is the difference between "any team can pass their list" and
"every team writes `new ArrayList<>(theirList)` and a copy bug eventually follows".
You do this once, at design time, and it is permanent — changing a published
signature later is a binary-compatibility event.

**2. Reading a stack trace with `ArrayStoreException` in it.**
It almost never appears in the code that caused it. Knowing that it means "an array
was widened to a supertype somewhere upstream" takes you straight to the varargs or
`toArray` call, instead of staring at the throwing line for twenty minutes.

**3. Reviewing a `@SuppressWarnings("unchecked")`.**
Most of them are someone losing an argument with variance. The review question is
always "which direction does this value flow, and did you pick the wildcard that
matches?" About half of them turn out to be a missing `? extends` and can be deleted
outright — which matters, because every suppression you leave in place is a place
the compiler has stopped helping you.

---

## Connected topics

**Prerequisites:**
- **05 — Generics I**: type parameters, bounds, generic methods. This topic assumes
  you can already read `<T extends Comparable<T>>`.
- **06 — Type erasure**: the *reason* generics are invariant. If erasure is not solid,
  reread it before this one — the whole argument rests on "there is no runtime type".
- **02 — Nominal typing**: why Java cannot fall back on structural compatibility the
  way TypeScript does.

**This unlocks:**
- **10 — Collections Framework**: every collection interface method you will read
  (`addAll`, `containsAll`, `removeAll`) has wildcards in it, and now they are
  readable.
- **11 — List implementations**: `List.copyOf(Collection<? extends E>)` and friends.
- **14 — Comparable vs Comparator**: `Comparator.comparing` is a masterclass in
  variance — `Comparator<? super T>` appears everywhere for the reason in Q4.
- **21–24 — Lambdas, method references, Streams**: `Stream.map(Function<? super T, ?
  extends R>)`. Every functional interface parameter in the JDK is contravariant in
  its input and covariant in its output, and you will now read those signatures at
  speed instead of skipping them.
- **27–29 — Records, sealed types, pattern matching**: sealed hierarchies plus
  wildcards is how you model a closed set of `orderflow` events with a covariant
  handler API.
- **47 — Spring Data JPA**: `<S extends T> List<S> saveAll(Iterable<S>)` — Q5's
  signature, in a framework you will use daily.

---

*Java baseline 21. Nothing in this topic changed between 21 and 25 at the language
level; wildcard **inference** has been quietly improved across releases, so a snippet
that fails to compile on one JDK may succeed on a newer one. If two machines
disagree, check `javac --version` first. Declaration-site variance has been proposed
for Java repeatedly and has never landed — do not plan around it.*
