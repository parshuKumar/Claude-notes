# 01 — Primitives, Wrappers, Autoboxing, and the Integer Cache

## Phase: 1 — Core Language
## Category: FOUNDATION
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Imagine a warehouse.

A **primitive** is a number written directly on the shelf label. Cheap. Instant to read.

A **wrapper** is that same number printed on a card, put inside a box, and the box
placed on the shelf. Now the label just says "go look in box #4471".

Both hold the number 5. But:

- The box costs materials to make (memory).
- Reading the number means walking to the box and opening it (an extra hop).
- And here is the part that bites people: **two different boxes can both contain a 5.**

So if you ask "is this the same box?" instead of "is the number inside the same?",
you get the wrong answer. That is the entire bug you are here to learn about.

---

## The bridge from what you know

### What transfers

TypeScript has one number type: `number`. It is a 64-bit float. Always.

```ts
let quantity = 5;        // float64
let price    = 19.99;    // float64
let orderId  = 90210;    // float64
```

Java makes you choose:

```java
int quantity = 5;
double price = 19.99;
long orderId = 90210L;
```

That is the first real difference: **Java makes you pick a size and a shape**, and
picking wrong has consequences you can measure.

### The honest analogue you already have (but never see)

You actually *do* have boxing in JavaScript. V8 stores small integers as **Smi**
(small integer) — the value is packed directly into the pointer, no object at all.
Larger numbers and all floats become a **HeapNumber** — a real object on the heap.

So V8 does the same trick Java does. **Verdict: PARTIAL analogue.**

What transfers: the idea that "a number" can be either a raw value or a heap object,
and the heap object costs more.

What does **not** transfer: in JavaScript this is completely invisible. `5 === 5` is
`true` no matter how V8 stored them. In Java, boxing is visible in the type system
(`int` vs `Integer`), visible in your code, and **`==` will lie to you.**

### What does not transfer at all

| TypeScript | Java | Verdict |
|---|---|---|
| `number` (one type) | 8 primitives + 8 wrappers | **PARTIAL** — you now make a choice you never made before |
| `===` compares number values, always | `==` on wrappers compares *identity* | **NO ANALOGUE** — this is a new failure mode |
| `null` on a number is a type error under `strictNullChecks` | `Integer` can be `null` and explodes when unboxed | **PARTIAL** — the hole exists, but Java's compiler will not catch it for you |
| `bigint` for large integers | `long`, then `BigInteger` | **PARTIAL** |
| Numbers in arrays are just numbers | `List<Integer>` boxes every element | **NO ANALOGUE** — a real cost with no JS equivalent |

---

## What is this?

Java has **8 primitive types** that hold a raw value directly, and **8 wrapper
classes** that hold the same value inside an object.

**Autoboxing** is the compiler quietly converting between them for you — `int` to
`Integer` and back — so you can put a number into a `List`, which can only hold
objects.

The **Integer cache** is a small pool of pre-made `Integer` objects for the values
−128 to 127. Because those objects are reused, `==` accidentally works inside that
range and accidentally fails outside it.

---

## Why does it matter?

Three things break in production, and all three are quiet:

1. **A comparison that passes every test and fails on real data.**
   Order IDs 1, 2, 3 in your tests are inside the cache. Order ID 4,829,113 in
   production is not. Same code. Different answer.

2. **A `NullPointerException` on a line with no visible null.**
   `int stock = stockLevels.get(sku);` throws when the key is missing. The stack
   trace points at a line where you never wrote `null` anywhere.

3. **Memory and speed you did not budget for.**
   An `int` is 4 bytes. An `Integer` is about 16 bytes, plus a pointer to reach it.
   In a list of 10 million, that is not a rounding error — it is the difference
   between fitting in memory and not.

---

## Syntax breakdown

### The 8 primitives

```java
byte    b   = 100;              // 8 bits.  -128 .. 127
short   s   = 30000;            // 16 bits. -32,768 .. 32,767
int     i   = 2_000_000_000;    // 32 bits. about +/- 2.1 billion   <- your default
long    l   = 9_000_000_000L;   // 64 bits. about +/- 9.2 quintillion
float   f   = 19.99f;           // 32-bit decimal. Rarely worth it.
double  d   = 19.99;            // 64-bit decimal.                  <- your default
char    c   = 'A';              // 16 bits. A UTF-16 code unit, NOT a byte.
boolean ok  = true;             // true / false.
```

Line by line, the bits that are new to you:

| Bit of syntax | What it means |
|---|---|
| `2_000_000_000` | Underscores in number literals. Purely for reading. The compiler ignores them. |
| `9_000_000_000L` | The trailing `L` says "this literal is a `long`". **Without it the compiler treats it as an `int` and refuses to compile**, because the value does not fit in 32 bits. |
| `19.99f` | The trailing `f` says "float". A bare `19.99` is a `double`. |
| `'A'` (single quotes) | A `char`. Double quotes `"A"` is a `String`. These are different types, and unlike TypeScript the quote style is not a matter of taste. |

### The 8 wrappers

Each primitive has an object version. Note the capital letters, and note that two of
them are not simply the primitive name capitalised:

```
byte    -> Byte
short   -> Short
int     -> Integer      // not "Int"
long    -> Long
float   -> Float
double  -> Double
char    -> Character    // not "Char"
boolean -> Boolean
```

### Autoboxing and unboxing

```java
Integer boxed   = 42;        // autoboxing:  int -> Integer
int     unboxed = boxed;     // unboxing:    Integer -> int
```

You wrote no conversion code. The compiler inserted it. Specifically, it rewrote
your two lines into roughly this:

```java
Integer boxed   = Integer.valueOf(42);   // autoboxing calls valueOf(...)
int     unboxed = boxed.intValue();      // unboxing calls intValue()
```

**That rewrite is the whole topic.** `Integer.valueOf` is where the cache lives, and
`intValue()` is where the `NullPointerException` comes from. You will confirm both
with `javap` in the hands-on section below.

### Why wrappers exist at all

Java's collections hold **objects**, not primitives. So this is illegal:

```java
List<int> quantities = new ArrayList<>();   // does not compile
```

and this is what you write instead:

```java
List<Integer> quantities = new ArrayList<>();
quantities.add(5);      // autoboxed to Integer.valueOf(5)
```

> This limitation comes from **type erasure**, which is Topic 06. For now: generics
> only work with objects, so numbers in collections must be boxed. Project Valhalla
> aims to remove this cost. It has not landed as of Java 25 — check its status when
> you actually need it, rather than planning around it.

---

## The Integer cache — the exact rule

When you autobox, the compiler calls `Integer.valueOf(int)`. That method does **not**
always create a new object. It keeps a pre-built array of `Integer` objects and hands
you one from that array when it can.

**The cached range is −128 to 127, inclusive.** That is guaranteed by the language
spec, not just an implementation detail of one JVM.

```java
Integer a = 127;
Integer b = 127;
a == b;              // true  -> both point at the SAME cached object

Integer c = 128;
Integer d = 128;
c == d;              // false -> two SEPARATE objects, each holding 128
```

Nothing about the number 128 is special. It is just past the edge of the cache.

### The other wrappers

| Wrapper | Cached range | Tunable? |
|---|---|---|
| `Byte` | all values (there are only 256) | no |
| `Short` | −128 .. 127 | no |
| `Integer` | −128 .. 127 | **yes** — the upper bound only |
| `Long` | −128 .. 127 | no |
| `Character` | 0 .. 127 | no |
| `Boolean` | the `TRUE` and `FALSE` constants | no |
| `Float` | **nothing is cached** | no |
| `Double` | **nothing is cached** | no |

The `Integer` upper bound is tunable with `-XX:AutoBoxCacheMax=<n>`. You will use
that flag in the hands-on section to *prove* the cache is what is causing the
behaviour. **Do not ever use it to fix a bug.** Changing a JVM flag so that `==`
starts working is hiding a defect, not repairing one, and it will not survive the
next person who reads the code.

### The rule you actually follow

> **Never use `==` on wrapper types. Use `.equals()`, or unbox to primitives first.**

```java
Long a = 4_829_113L;
Long b = 4_829_113L;

a == b;                         // false. Two objects.
a.equals(b);                    // true.  Compares the values.
a.longValue() == b.longValue(); // true.  Two primitives, real value comparison.
```

---

## Example 1 — minimal

```java
public class CacheBoundary {
    public static void main(String[] args) {
        Integer a = 127, b = 127;
        Integer c = 128, d = 128;

        System.out.println("127 == 127 : " + (a == b));
        System.out.println("128 == 128 : " + (c == d));
        System.out.println("128 .equals: " + c.equals(d));
    }
}
```

Run it. One of those three lines is `false`, and it is the one that looks most
obviously true. That is the point.

---

## Example 2 — production scenario

You are writing the deduplication check for an order-intake endpoint. Customers retry
on timeout, so the same order can arrive twice. You must not charge twice.

### The version that ships and then quietly fails

```java
public class OrderDeduplicator {

    private final List<Long> recentOrderIds = new ArrayList<>();

    public boolean isDuplicate(Long incomingOrderId) {
        for (Long seen : recentOrderIds) {
            if (seen == incomingOrderId) {      // <-- the defect
                return true;
            }
        }
        recentOrderIds.add(incomingOrderId);
        return false;
    }
}
```

Your unit tests use order IDs 1, 2 and 3. Every test passes. The code is correct
**for all IDs from −128 to 127** and wrong for every other value.

Production order IDs start at 4,829,113. Every duplicate check returns `false`. Every
retried order is treated as new. Customers get charged twice.

### The corrected version

```java
public class OrderDeduplicator {

    // A Set gives O(1) membership and uses equals()/hashCode(), not ==.
    private final Set<Long> recentOrderIds = new HashSet<>();

    public boolean isDuplicate(long incomingOrderId) {   // primitive: cannot be null
        return !recentOrderIds.add(incomingOrderId);     // add() returns false if present
    }
}
```

Three changes, and each removes a different hazard:

1. `Set` instead of `List` — membership is now `equals()`-based, so identity never
   enters the picture. It is also O(1) instead of O(n).
2. `long` instead of `Long` in the parameter — a primitive cannot be `null`, so the
   NPE described below is impossible here.
3. `add()` returns `false` when the element was already present, so there is no
   separate check-then-act step.

> This is still not production-complete: a `Set` that only ever grows is a memory
> leak, and idempotency across service restarts needs a database constraint, not
> memory. Those are Topics 79 and 116. The boxing bug is fixed.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `==` on wrappers

**Wrong:**
```java
if (orderId == cachedOrderId) { ... }   // both are Long
```

**Exact symptom:** the branch is never taken in production, but is always taken in
tests. No exception, no log line, no error metric. You find it from a business
number: duplicate-charge complaints, or a cache hit rate that sits flat at 0%.

**Root cause:** `==` on objects compares references. Small values happen to share a
cached object, so tests with small fixture IDs pass.

**Fix:** `orderId.equals(cachedOrderId)`, or better, make the variables `long`.

---

### Trap 2 — unboxing a `null`

**Wrong:**
```java
Map<String, Integer> stockBySku = inventory.currentStock();
int available = stockBySku.get("SKU-4471");    // SKU not in the map
```

**Exact symptom:**
```
java.lang.NullPointerException: Cannot invoke "java.lang.Integer.intValue()"
  because the return value of "java.util.Map.get(Object)" is null
```
pointing at a line where you never mentioned `null`.

> Java 14+ gives you that helpful message naming the exact method. On older JVMs the
> message is empty and this is genuinely painful to diagnose — which is why it is
> worth learning to recognise the shape on sight.

**Root cause:** `Map.get` returns `null` for a missing key. The compiler inserted
`.intValue()` to unbox it into your `int`. `null.intValue()` throws.

**Fix:**
```java
int available = stockBySku.getOrDefault("SKU-4471", 0);
```

---

### Trap 3 — the ternary that unboxes

**Wrong:**
```java
Integer discountPercent = promotion.discountPercent();   // may be null
int applied = customer.isVip() ? discountPercent : 0;
```

**Exact symptom:** `NullPointerException` — *even when `isVip()` returns false and
you never touch `discountPercent`.*

**Root cause:** the two branches have different types (`Integer` and `int`). Java
resolves that by unboxing the `Integer` branch, and it decides this while working out
the type of the whole expression — before any branch is selected. So the unboxing
happens either way.

**Fix:** make both branches the same type.
```java
int applied = customer.isVip() && discountPercent != null ? discountPercent : 0;
```

This one is genuinely surprising, and it is a favourite interview question.

---

### Trap 4 — boxing in a hot loop

**Wrong:**
```java
Map<Long, Long> unitsSoldByProduct = new HashMap<>();

for (OrderLine line : lines) {                       // 5 million lines
    Long current = unitsSoldByProduct.get(line.productId());
    unitsSoldByProduct.put(line.productId(),
                           current == null ? line.quantity() : current + line.quantity());
}
```

**Exact symptom:** the job is slower than it should be, and the GC log shows a high
young-collection rate. CPU time shows up in allocation, not in your logic.

**Root cause:** `current + line.quantity()` unboxes twice, adds, then boxes the
result — a fresh `Long` every iteration, because running totals exceed 127 almost
immediately. Five million short-lived objects.

**Fix (idiomatic Java):**
```java
unitsSoldByProduct.merge(line.productId(), line.quantity(), Long::sum);
```

**Fix (only if it is genuinely on the hot path):** a primitive-keyed map from a
library such as Eclipse Collections, fastutil or HPPC. They store `long` keys and
`long` values in plain arrays, with no boxing at all.

> Do not do the second one on a hunch. Boxing is a *real* cost but it is rarely
> *your* cost. "Measure first" is not a platitude here — Topic 77 (JMH) exists
> because timing this by hand gives you a wrong answer. Fix the algorithm first;
> reach for a specialised collection when a profiler tells you to.

---

### Trap 5 — `double` for money

**Wrong:**
```java
double orderTotal = 0.10 + 0.20;   // 0.30000000000000004
```

**Exact symptom:** invoice totals off by one penny. Reconciliation reports that never
balance. A customer who paid £19.99 charged £19.990000000000002.

**Root cause:** `double` is binary floating point. It cannot represent 0.1 exactly,
for the same reason base-10 cannot represent 1/3 exactly. This is identical to
JavaScript's `0.1 + 0.2 !== 0.3` — you have already met this bug.

**Fix:** store money as `long` in minor units (pence, cents), or use `BigDecimal`
with an explicit scale and rounding mode. Never `double`, never `float`.

```java
long totalInPence = 1999L;                   // simple, fast, exact
BigDecimal total  = new BigDecimal("19.99"); // note: String, not double
```

`new BigDecimal(19.99)` — with a `double` argument — puts the exact bug you were
avoiding straight back in. Always construct from a `String`.

---

## Hands-on proof

Everything below is a command **you** run. I am not going to print output and claim
it is real — I do not have a JVM. What I can give you precisely is what to look for,
and what each possible result means.

### Setup

```bash
mkdir -p ~/java-lab/01 && cd ~/java-lab/01
java --version     # confirm your JDK. Should be 21 or 25.
```

### Proof 1 — the cache boundary is real

`CacheProof.java`:
```java
public class CacheProof {
    public static void main(String[] args) {
        for (int value : new int[] { 126, 127, 128, 129 }) {
            Integer a = value;
            Integer b = value;
            System.out.printf("%4d :  == %-6s  .equals %s%n", value, a == b, a.equals(b));
        }
    }
}
```

```bash
java CacheProof.java
```

**What to look for:** the `==` column. Read the result like this:

| What you see | What it means |
|---|---|
| `true` for 126 and 127, `false` for 128 and 129 | The expected result. You have observed the cache boundary directly. |
| `true` on all four rows | Something is caching more than the default. Check `JAVA_TOOL_OPTIONS` and `_JAVA_OPTIONS` for `-XX:AutoBoxCacheMax`. |
| `.equals` is `true` on every row | Correct, and it is the point: `.equals` does not care about the boundary at all. |

### Proof 2 — move the boundary, prove the cause

```bash
java -XX:AutoBoxCacheMax=1000 CacheProof.java
```

**What to look for:** all four rows should now read `true` in the `==` column.

**Why this matters:** if a JVM flag can change whether your comparison works, then
your comparison was never comparing what you thought it was. This is the strongest
evidence available that `==` tests object identity and not value.

Confirm the flag actually took effect:
```bash
java -XX:AutoBoxCacheMax=1000 -XX:+PrintFlagsFinal -version | grep -i autobox
```

Again: this flag is a **diagnostic**, never a fix.

### Proof 3 — watch the compiler insert the calls

`BoxingBytecode.java`:
```java
public class BoxingBytecode {
    public static int total(Integer a, int b) {
        Integer sum = a + b;
        return sum;
    }
}
```

```bash
javac BoxingBytecode.java
javap -c BoxingBytecode.class
```

**What to look for** in the disassembly of `total`:

- `invokevirtual java/lang/Integer.intValue` — unboxing `a` so it can be added.
- `invokestatic  java/lang/Integer.valueOf` — boxing the result into `sum`.
- a second `Integer.intValue` — unboxing `sum` for the `int` return.

**How to read it:** you wrote one `+` and one `return`. The bytecode contains three
method calls you never typed. That is what "the compiler does it for you" actually
costs, and it is why boxing shows up in profiles as work you cannot find in your
source.

> I am telling you which instructions to look for, not quoting exact output. If your
> listing differs from this description, trust your listing — paste it and I will read
> it. Reading bytecode properly is Topic 76.

### Proof 4 — the null unbox, with the real stack trace

`NullUnbox.java`:
```java
import java.util.*;

public class NullUnbox {
    public static void main(String[] args) {
        Map<String, Integer> stockBySku = new HashMap<>();
        stockBySku.put("SKU-1001", 12);

        int available = stockBySku.get("SKU-4471");   // absent key
        System.out.println(available);
    }
}
```

```bash
java NullUnbox.java
```

**What to look for:** an NPE whose message names `Integer.intValue()` and says it was
called on the return value of `Map.get`. Two things worth keeping:

1. The word `intValue` appears in an exception for code where you never typed it.
   Seeing `intValue` in an NPE means "something unboxed a null" — that is now a
   one-second diagnosis for you.
2. If your JDK gives an NPE with **no** message, you are on an older JVM or have
   `-XX:-ShowCodeDetailsInExceptionMessages` set. Turn it back on.

### Proof 5 — measure the memory difference

```bash
# download once
curl -o jol-cli.jar \
  https://repo1.maven.org/maven2/org/openjdk/jol/jol-cli/0.17/jol-cli-0.17-full.jar

java -jar jol-cli.jar internals java.lang.Integer
```

**What to look for:** the total instance size, and how much of it is object header
versus your actual 4-byte `int`.

**How to read the result:** on a 64-bit JVM with compressed references you should see
a 12-byte header, 4 bytes of payload, 16 bytes total. That is a 4× overhead for
storing a number — and it does not count the 4-or-8-byte pointer in the array that
points at it. This is exactly why `List<Integer>` is not a free substitute for
`int[]`.

Object layout in full is Topic 69. You are just collecting the fact today.

---

## Practice exercises

Write real files, run them, and paste your code and any output.

### 1 — Easy: find the boundary yourself

Write a program that **discovers** the cache boundary rather than assuming it. Loop
upward from 0, box each value twice, and print the first value for which `==` is
`false`. Then go downward from 0 to find the lower edge.

Requirements:
- Do not hardcode 127 or −128 anywhere.
- Print both boundaries.
- Then run the same program with `-XX:AutoBoxCacheMax=500` and report what changed
  and — importantly — what **did not** change.

### 2 — Medium: the audit

Here is a fragment from an order-processing service. It contains **five** distinct
defects from this topic. Find them all, explain the exact symptom each produces in
production (not "it's bad practice" — what does the on-call engineer actually see?),
and rewrite it correctly.

```java
public class OrderProcessor {

    private final Map<Long, Integer> stockBySku = new HashMap<>();
    private final List<Long> processedOrderIds = new ArrayList<>();

    public double processOrder(Long orderId, Long sku, Integer quantity) {

        for (Long processed : processedOrderIds) {
            if (processed == orderId) {
                return 0.0;
            }
        }

        Integer available = stockBySku.get(sku);
        if (available > quantity) {
            stockBySku.put(sku, available - quantity);
        }

        double unitPrice = 0.10;
        double total = unitPrice * quantity;

        Integer discount = lookupDiscountPercent(sku);
        double applied = (quantity > 10) ? discount : 0;

        processedOrderIds.add(orderId);
        return total - (total * applied / 100);
    }

    private Integer lookupDiscountPercent(Long sku) {
        return null;   // no promotions configured yet
    }
}
```

Draw on what you already have: your TypeScript instinct about `===`, and your SQL
background about how money should be stored.

### 3 — Hard: production simulation

Build a small inventory counter, then break it, then measure it.

**Part A.** Write `InventoryCounter` with a `Map<Long, Long>` tracking units sold per
product ID, and a method `record(long productId, long quantity)` that accumulates.
Process 5,000,000 synthetic order lines across 100,000 product IDs.

**Part B.** Write the accumulation twice:
- **Version 1:** the naive `get` / null-check / add / `put` loop from Trap 4.
- **Version 2:** `merge` with `Long::sum`.

**Part C.** Run both with GC logging on and compare:

```bash
java -Xlog:gc:file=v1.log:time,uptime -Xmx512m InventoryCounter v1
java -Xlog:gc:file=v2.log:time,uptime -Xmx512m InventoryCounter v2
```

Report the number of young collections in each log, and the total time. Then answer
in your own words: **did the difference come from fewer objects, or from something
else?** Be honest if the difference is small — "barely any difference, here are the
numbers" is a better answer than a confident wrong one.

**Part D.** Now argue the other side. Given your measurements, would you actually
replace this with a primitive-keyed map from fastutil in a real codebase? State the
condition under which your answer flips.

> Note: you are deliberately timing with a wall-clock stopwatch here, which Topic 77
> will teach you is the wrong way to benchmark. That is intentional. You will come
> back to this exercise after JMH and see how wrong it was.

---

## Interview questions

### Q1 — "What does `Integer a = 127, b = 127; a == b;` evaluate to? What about 128?"

**Mid-level answer:** "`true` for 127 and `false` for 128, because Java caches
Integers from −128 to 127."

**Senior answer:** "`true`, then `false`. Autoboxing compiles to `Integer.valueOf`,
which returns a shared instance for −128 to 127, so `==` accidentally succeeds in
that range. Outside it you get distinct objects and `==` compares references. The
real point is that `==` on a wrapper never tests what you want — and the 127 case
passing is the *more* dangerous one, because it means unit tests with small fixture
IDs pass while production data fails."

**What separates them:** the mid-level answer recites the range. The senior answer
identifies *why the correct-looking case is the dangerous one*, and connects it to how
the bug escapes testing.

**Follow-up the interviewer asks:** "Can you change that behaviour?" They want
`-XX:AutoBoxCacheMax`. They are then checking whether you volunteer, unprompted, that
using it as a fix would be wrong.

---

### Q2 — "This line throws a NullPointerException. Where is the null?"

```java
int available = stockBySku.get("SKU-4471");
```

**Mid-level answer:** "The map doesn't contain the key, so `get` returns null."

**Senior answer:** "Same cause, plus the mechanism: the compiler inserted
`.intValue()` to unbox into the `int`, and that call is the one that throws. On Java
14+ the message names `Integer.intValue()` explicitly, which is how you recognise the
shape instantly. The fix is `getOrDefault` — but the design question is whether a
missing SKU genuinely means zero stock or means a data error. Silently defaulting to
0 can hide the real problem."

**What separates them:** naming the inserted call, then treating "what should a
missing key mean?" as a design decision rather than jumping to a one-liner.

**Follow-up:** "When is defaulting to 0 the wrong choice?"

---

### Q3 — "Why does this throw even when `isVip()` returns false?"

```java
Integer discount = null;
int applied = customer.isVip() ? discount : 0;
```

**Mid-level answer:** "I'd have to run it — I'd expect it to give 0."

**Senior answer:** "It throws. The two branches are `Integer` and `int`, so the
conditional's type is worked out by numeric promotion, which unboxes the `Integer`
branch. That decision is part of typing the expression, so it applies regardless of
which branch is actually selected. Making both branches `Integer`, or null-checking
explicitly, avoids it."

**What separates them:** knowing that the *type of the expression* is decided before
the branch is — a language-semantics answer rather than a runtime guess.

**Follow-up:** "How would you have caught this before production?" Good answers name
a static analyser — ErrorProne, NullAway, IntelliJ's inspection — rather than "more
tests", because you cannot write a test for a case you did not think of.

---

### Q4 — "How much memory does a `List<Integer>` of 10 million elements use versus `int[]`?"

**Mid-level answer:** "More, because of object overhead."

**Senior answer:** "`int[]` is about 40 MB — 4 bytes each plus a small header.
`List<Integer>` is roughly 4× to 5× that: each `Integer` is about 16 bytes — 12-byte
header plus 4-byte payload, aligned — and the backing array holds a 4-byte compressed
reference to each one. So call it 200 MB, plus the GC cost of tracing 10 million
extra objects, plus cache misses from pointer-chasing instead of a linear scan. If
every value happened to be inside the cache range you'd get sharing and far less, but
that is not a case worth planning around. I'd verify with JOL rather than trusting my
arithmetic."

**What separates them:** a number with the reasoning behind it, mentioning *memory
locality* and not just size, and offering to verify rather than asserting.

**Follow-up:** "At what point would you reach for a primitive collection library?"
They are testing whether you optimise on evidence or on reflex.

---

### Q5 — "Your team stores prices as `double`. Talk me through it."

**Mid-level answer:** "You should use `BigDecimal` for money."

**Senior answer:** "`double` is binary floating point, so 0.1 and 19.99 aren't
representable exactly — the same reason `0.1 + 0.2 !== 0.3` in JavaScript. Errors
accumulate across line items and surface as reconciliation breaks, which are
expensive to investigate. Two viable fixes: `long` minor units, which is fast and
exact and my default on a high-volume transaction path; or `BigDecimal` with an
explicit scale and rounding mode, which is better once you need division, tax and
multiple currencies. If it's `BigDecimal`, construct from a `String` — `new
BigDecimal(19.99)` puts the bug straight back. And the real work is the migration:
the existing rows already hold drifted values."

**What separates them:** two options with a stated preference and a reason, the
`String` constructor trap, and thinking about migrating existing data rather than only
about new code.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The spec guarantees caching for −128 to 127 but allows a JVM to cache more. Why
   guarantee a range at all, rather than saying nothing? What would break if the range
   were left entirely unspecified?

2. `Float` and `Double` have no cache. Give the reason — and connect it to why
   `Double.NaN == Double.NaN` is `false` while
   `Double.valueOf(Double.NaN).equals(Double.valueOf(Double.NaN))` is `true`.

3. If you could change one thing about Java's design here, would you remove
   autoboxing entirely (forcing explicit conversion), or remove `==` on objects
   (forcing `.equals`)? Argue for one, then make the strongest case against yourself.

4. You are reviewing a PR that changes a method signature from `Integer getQuantity()`
   to `int getQuantity()`. Under what circumstance is that change a bug rather than an
   improvement?

5. TypeScript has no boxing you can observe, yet V8 boxes numbers internally. What
   does Java gain by making boxing visible in the type system? What does it lose?

6. A colleague says "boxing is slow, so I replaced every `List<Integer>` in the
   codebase with `int[]`." What is wrong with that reasoning *even if every individual
   replacement is genuinely faster*?

7. `Boolean` caches `TRUE` and `FALSE`, so `Boolean a = true; Boolean b = true; a == b`
   is always `true`. Does that make `==` safe for `Boolean`? Justify your answer as a
   *code review rule*, not just as a fact.

---

## Quick reference card

### Sizes and defaults

| Type | Bits | Range | Default value |
|---|---|---|---|
| `byte` | 8 | −128 .. 127 | `0` |
| `short` | 16 | −32,768 .. 32,767 | `0` |
| `int` | 32 | about ±2.1 billion | `0` |
| `long` | 64 | about ±9.2 quintillion | `0L` |
| `float` | 32 | ~7 decimal digits | `0.0f` |
| `double` | 64 | ~15 decimal digits | `0.0` |
| `char` | 16 | ` ` .. `￿` | ` ` |
| `boolean` | — | `true` / `false` | `false` |

Wrappers all default to `null`. That single difference is the source of Trap 2.

### Key APIs

```java
Integer.valueOf(int)              // what autoboxing calls; uses the cache
Integer.parseInt(String)          // String -> int (no object created)
Integer.valueOf(String)           // String -> Integer
someInteger.intValue()            // what unboxing calls; NPEs on null
Integer.compare(a, b)             // safe comparison, no boxing
Integer.MAX_VALUE / MIN_VALUE
Math.addExact(a, b)               // throws ArithmeticException on overflow
Objects.requireNonNullElse(x, 0)
map.getOrDefault(key, 0)          // the fix for Trap 2
map.merge(key, delta, Long::sum)  // the fix for Trap 4
```

### Costs at a glance

| | `int` | `Integer` |
|---|---|---|
| Memory | 4 bytes | ~16 bytes + a reference to reach it |
| Can be `null` | no | yes |
| `==` | compares value | compares identity |
| In a collection | impossible | required |
| Arithmetic | direct | unbox → compute → box |

### Gotchas checklist

- [ ] Never `==` between wrappers. `.equals()`, or unbox first.
- [ ] Unboxing a `null` throws. Look for `intValue` / `longValue` in NPE messages.
- [ ] A ternary mixing `Integer` and `int` unboxes **both** branches.
- [ ] `long` literals need the `L` suffix.
- [ ] Money is `long` minor units or `BigDecimal`. Never `double`.
- [ ] `new BigDecimal(19.99)` is wrong; `new BigDecimal("19.99")` is right.
- [ ] `int` overflows silently. `Math.addExact` throws instead.
- [ ] `-XX:AutoBoxCacheMax` is for diagnosis. Never for a fix.

---

## When would I use this at work?

**1. Reviewing a pull request.**
Someone writes `if (userId == currentUserId)` where both are `Long`. You catch it in
review instead of in a duplicate-charge incident three months later. This is the most
common way this topic pays for itself, and it costs you nothing once the pattern is in
your eye.

**2. Reading a stack trace at 2am.**
An NPE mentions `Integer.intValue()`. You go straight to a map lookup or an unboxed
field, instead of hunting for an explicit `null` that does not exist anywhere in the
file. Minutes instead of an hour.

**3. Sizing a service before it exists.**
Product asks whether a 50-million-row in-memory index will fit in an 8 GB container.
`List<Long>` versus `long[]` is the difference between roughly 1 GB and roughly
200 MB — and that is the difference between "yes" and "we need a different
architecture". You will do this properly in Topics 69 and 129; this is where the
instinct starts.

---

## Connected topics

**Prerequisites:** none. This is the first topic.

**This unlocks:**
- **06 — Type erasure**: *why* collections cannot hold primitives at all.
- **12 — HashMap internals**: what `hashCode` on a boxed `Integer` returns, and why
  that matters for how keys spread across buckets.
- **13 — equals/hashCode contract**: the general rule behind "use `.equals`".
- **17 — Immutability**: wrappers are immutable, and that is precisely why sharing
  cached instances is safe in the first place.
- **23 — Streams**: `IntStream` and `mapToInt` exist specifically to avoid the boxing
  cost you just measured.
- **69 — Object layout**: the exact 16 bytes, verified rather than quoted.
- **77 — JMH**: how to benchmark boxing correctly, and why your Exercise 3 timings
  were not trustworthy.
- **95 — Atomics**: `AtomicInteger` versus `Integer` — one is mutable and thread-safe,
  the other is neither.

---

*Java baseline 21. Nothing in this topic changed between 21 and 25. Autoboxing has
been stable since Java 5 and the cache range is fixed by the spec, so what you learn
here will still be true in a decade — with the possible exception of Project Valhalla
eventually making the cost disappear, which has not happened yet.*
