# 18 — Strings: immutability, the pool, `intern()`, `StringBuilder`, compact strings

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

A `String` is a **printed poster**, not a whiteboard.

You cannot edit a poster. If you want the poster to say something different, you print
a new one. The old poster is still exactly as it was.

So `"hello".toUpperCase()` does not shout at the poster. It prints a second poster.

Now the second idea. The library keeps **one shelf of master copies**. When you write
a poster's text directly in your code — a *literal* — the library checks that shelf. If
that exact text is already there, you get a pointer to the existing master copy. If it
is not, your poster goes on the shelf and becomes the master.

That shelf is the **string pool**.

And the third idea, which is where the bugs live. If you say "print me a fresh poster
of this text" — `new String("hello")` — you get a **brand new poster** that is not the
master copy. It reads identically. It is a different physical object. So "is this the
same poster?" says no, while "does it say the same thing?" says yes.

Finally: building a poster by repeatedly adding one word and reprinting the whole thing
is how you turn a 5-minute job into a 5-hour job. That is string concatenation in a
loop, and it is the most expensive mistake in this document.

---

## The bridge from what you know

### This is one of the closest **HONEST ANALOGUES** in the whole curriculum

I say that rarely. Here it is earned.

JavaScript strings are **immutable**. Exactly like Java's.

```js
const sku = "SKU-4471";
sku[0] = "X";        // silently does nothing
sku.toUpperCase();   // returns a NEW string; sku is unchanged
```

```java
String sku = "SKU-4471";
sku.toUpperCase();   // returns a NEW String; sku is unchanged
```

Same semantics, same reason, same consequences.

V8 also **internalises** string literals. Two identical literals in your JavaScript
source typically end up as the same internal string object, which is why `===` on
strings is fast — V8 can often compare by pointer before falling back to comparing
characters. That is structurally the same trick as Java's string pool.

V8 also has an optimisation for concatenation: `a + b` can produce a **cons string** (a
rope) that holds pointers to the two halves and only flattens into a real character
array when someone reads it. Java does not have that, and the difference matters — see
Trap 1.

**What transfers, cleanly:**
- Strings are immutable, so every "modification" allocates.
- Identical literals are shared behind your back.
- Comparing content is the operation you almost always want.

**What you must unlearn — and this is the whole hazard:**

`===` on JavaScript strings **always compares content**. The specification says so.
Internalisation is an invisible optimisation that can never change the answer.

`==` on Java strings **always compares identity**. Pooling is a visible behaviour that
absolutely changes the answer. Two strings with the same characters may be `==` or may
not be, depending entirely on where they came from.

| You know | Java | Verdict |
|---|---|---|
| Strings are immutable | Strings are immutable | **HONEST ANALOGUE** |
| V8 internalises literals | The string pool interns literals | **HONEST ANALOGUE** — same idea, and both are for memory and fast comparison |
| `===` compares content, always | `==` compares identity, always | **NO ANALOGUE** — this is a new failure mode with no JS counterpart |
| `a + b` may build a cheap cons string | `a + b` materialises a new `String` every time | **PARTIAL** — the JS optimisation makes loop concatenation survivable; in Java it is quadratic |
| One character = one UTF-16 code unit | Same, plus a Latin-1 storage optimisation you can observe | **PARTIAL** — Java has *compact strings*, JS engines have their own one-byte/two-byte representations, but Java's is directly measurable and flag-controllable |
| Template literals `` `total ${x}` `` | `"total %s".formatted(x)` or `+` | **PARTIAL** — no interpolation. Java's string templates were withdrawn before finalising. |

---

## What is this?

Four separate mechanisms that people fuse into one vague idea.

**1. Immutability.** A `String`'s contents never change after construction. Every
method that "changes" a string returns a new one. `String` is a `final` class with a
`private final byte[]` inside it, and it never lets that array escape — the exact
discipline from **Topic 17**.

**2. The string pool.** A JVM-wide table of `String` objects, keyed by content. Every
string *literal* in your source is automatically placed in it at class-load time, and
every occurrence of that literal anywhere in the JVM resolves to the same object.
`intern()` lets you put a runtime-built string in the pool manually.

**3. `StringBuilder`.** A mutable character buffer. This is what you use when you
genuinely need to build a string piece by piece. It grows by doubling, like an
`ArrayList`.

**4. Compact strings.** Since Java 9, a `String` stores its characters as a `byte[]`
plus a one-byte `coder` flag. If every character fits in Latin-1 (roughly, the first
256 Unicode code points), it uses **one byte per character**. Otherwise it falls back
to two bytes per character (UTF-16). One non-Latin-1 character anywhere in the string
doubles the whole string's memory.

---

## Why does it matter?

**Strings dominate real Java heaps.** In a typical service, `String` and its backing
`byte[]` are routinely the two largest entries in a heap histogram — 20–40% is common.
Every log line, every JSON field name, every HTTP header, every SQL statement, every
map key. If you want to reduce a service's memory footprint, strings are where you
look first. You will do this properly with a heap dump in **Topic 79**; the facts you
need to interpret it are here.

**Concatenation in a loop is quadratic.** Building a 1 MB CSV export by `result +=
line` inside a loop copies the entire accumulated string on every iteration. At 5
million iterations this is not "slower" — it is a job that never finishes. This is the
single most common Java performance bug that a code reviewer can catch by eye.

**`==` on strings is a bug that hides in tests.** Your test compares two literals and
passes. Production compares a literal against a value parsed out of JSON and fails.
Same code, different answer, and no exception anywhere.

**`intern()` looks like a memory fix and is often a memory *move*.** It relocates a
heap problem into a JVM-internal hash table that is harder to see, harder to size, and
whose lookup cost you now pay on every call.

---

## Syntax breakdown

### Creating strings

```java
String a = "SKU-4471";                    // literal -> automatically pooled
String b = new String("SKU-4471");        // NEW object on the heap, NOT the pooled one
String c = b.intern();                    // look up / insert in the pool; c == a
String d = String.valueOf(4471);          // "4471"  (null-safe: valueOf(null) -> "null")
String e = new String(bytes, StandardCharsets.UTF_8);   // legitimate: decoding bytes
char[] chars = { 'S', 'K', 'U' };
String f = new String(chars);             // legitimate: building from a char array
```

| Form | When it is right |
|---|---|
| `"literal"` | Always, for constants. |
| `new String("literal")` | **Never.** There is no situation where this is correct. It allocates a duplicate of something you already have. |
| `new String(byte[], Charset)` | Correct and common — decoding network or file bytes. |
| `new String(char[])` | Correct — e.g. after reading a password into a `char[]`. |
| `String.valueOf(x)` | Converting anything to a string safely. |
| `x.intern()` | Only with a measurement in hand. See Trap 3. |

> **A precision point on deprecation.** The `String` constructors are **not**
> deprecated. `new String("x")` is merely pointless. This is different from the wrapper
> constructors — `new Integer(5)`, `new Boolean(true)` and friends — which carry
> `@Deprecated(forRemoval = true)` and produce a compiler warning. They still exist and
> still work on Java 21 and 25, but they are on a removal path. Settle both on your own
> JDK with:
> ```bash
> javac -Xlint:deprecation YourFile.java
> ```
> If `new Integer(5)` warns and `new String("x")` does not, you have confirmed the
> distinction directly rather than trusting me.

### Comparing strings

```java
a.equals(b)                    // content. This is what you want, 99% of the time.
a.equalsIgnoreCase(b)          // content, case-insensitive
a.compareTo(b)                 // lexicographic ordering, for sorting (Topic 14)
a.contentEquals(sb)            // compare a String against a StringBuilder/CharSequence
Objects.equals(a, b)           // null-safe equals; use when either side may be null
a == b                         // IDENTITY. Almost never what you want.
```

### Building strings

```java
StringBuilder sb = new StringBuilder(1024);   // pre-size when you can estimate
sb.append("order:").append(orderId).append(',');
sb.setLength(0);                              // reuse without reallocating
String result = sb.toString();                // one copy, at the end

String.join(",", parts)                       // the right tool for a delimiter
"total %s for order %d".formatted(total, orderId)   // Java 15+; same as String.format
String.format("total %s", total)              // identical result, older style
```

`StringBuffer` is `StringBuilder`'s synchronised twin from Java 1.0. Every method is
`synchronized`, which you virtually never need because a builder is almost always
thread-confined. **`[LEGACY — still asked]`** — an interviewer may ask you the
difference. Answer: same API, `StringBuffer` is synchronised and therefore slower; use
`StringBuilder` unless you have a builder genuinely shared across threads, which is a
design smell in itself.

### Compact strings — the internal shape

Since Java 9 the field layout is, in essence:

```java
public final class String {
    private final byte[] value;   // was char[] before Java 9
    private final byte coder;     // 0 = LATIN1 (1 byte/char), 1 = UTF16 (2 bytes/char)
    private int hash;             // cached hashCode, computed lazily. NOT final.
}
```

Three consequences worth carrying:

1. An all-ASCII string costs roughly **1 byte per character** plus object overhead.
2. Adding a single emoji, accented character or CJK character to a string flips
   `coder` to UTF-16 and **doubles the storage for the whole string**, not just for
   that character.
3. `hashCode` is cached in a non-final field, computed on first call. This is why
   `String` is a fast `HashMap` key after the first lookup, and it is also a
   deliberately-tolerated benign data race (two threads may both compute it and write
   the same value — **Topic 86**).

### The pool, precisely

- Every string **literal** in your source becomes a pooled string when its class is
  loaded. This is required by the Java Language Specification, not an implementation
  choice.
- Compile-time constant expressions are folded and pooled:
  `"SKU-" + "4471"` is one literal to the compiler.
- Runtime concatenation is **not** pooled: `"SKU-" + skuNumber` where `skuNumber` is a
  variable produces a fresh, unpooled `String`.
- `intern()` returns the pooled instance for the string's content, adding it if absent.

**Where the pool physically lives** — this matters and is widely misremembered:

| Era | Where interned strings live | Where the lookup table lives |
|---|---|---|
| Java 6 and earlier | PermGen (fixed size, `OutOfMemoryError: PermGen space`) | native |
| Java 7 onwards | the **main heap**, like any other object | a **native** hash table (`StringTable`) inside the JVM |

So on Java 21: the `String` objects themselves are ordinary heap objects and *can* be
garbage collected when nothing references them. The `StringTable` — the bucket array
used to find them — is a native structure whose size is fixed at JVM startup by
`-XX:StringTableSize`. That split is the whole reason Trap 3 behaves the way it does.

---

## Example 1 — minimal

```java
public class StringIdentity {
    public static void main(String[] args) {
        String literalA = "SKU-4471";
        String literalB = "SKU-4471";
        String built    = new String("SKU-4471");
        String interned = built.intern();

        String prefix   = "SKU-";
        String number   = "4471";
        String runtime  = prefix + number;          // built at runtime

        final String CONST_PREFIX = "SKU-";
        String compiled = CONST_PREFIX + "4471";    // folded at COMPILE time

        System.out.println("literalA == literalB : " + (literalA == literalB));
        System.out.println("literalA == built    : " + (literalA == built));
        System.out.println("literalA == interned : " + (literalA == interned));
        System.out.println("literalA == runtime  : " + (literalA == runtime));
        System.out.println("literalA == compiled : " + (literalA == compiled));
        System.out.println("all .equals each other: "
                + (literalA.equals(built) && literalA.equals(runtime)));
    }
}
```

Predict all six lines before you run it. Then run it.

The one that catches people is `compiled`: because `CONST_PREFIX` is a `final` local
initialised with a literal, the compiler treats `CONST_PREFIX + "4471"` as a
compile-time constant, folds it, and pools the result. Remove the `final` and the
answer flips. Nothing about your logic changed — only whether `javac` could prove the
value at compile time.

That is the point of the example: **`==` on strings is not answering a question about
your data, it is answering a question about your compiler.**

---

## Example 2 — production scenario

`orderflow` needs a nightly CSV export of order lines for the finance team. Roughly 5
million lines.

### The version that ships and then never finishes

```java
package com.orderflow.reporting;

import java.util.List;

public class OrderLineExporter {

    public String toCsv(List<OrderLineRow> rows) {
        String csv = "order_id,sku,quantity,unit_price_minor,line_total_minor\n";

        for (OrderLineRow row : rows) {                    // 5,000,000 iterations
            csv += row.orderId() + ","                     // <-- the defect
                 + row.sku() + ","
                 + row.quantity() + ","
                 + row.unitPriceMinor() + ","
                 + row.lineTotalMinor() + "\n";
        }
        return csv;
    }
}
```

What actually happens on each iteration: the JVM allocates a fresh buffer the size of
everything written so far, copies all of it, appends the new line, and throws the
previous buffer away. Iteration 4,000,000 copies about 200 MB to append 40 bytes.

Total bytes copied is proportional to n², which for 5 million lines is in the
**petabyte** range. The job does not run slowly. It does not run.

The symptom on-call sees: the export pod sits at 100% CPU with a flat, very high
allocation rate; the GC log is wall-to-wall young collections; heap usage sawtooths
against `-Xmx`; and eventually `OutOfMemoryError: Java heap space` with a heap dump
dominated by one enormous `byte[]`. Nothing in the stack trace says "concatenation".

### The corrected version

```java
package com.orderflow.reporting;

import java.io.IOException;
import java.io.Writer;
import java.util.List;

public class OrderLineExporter {

    private static final String HEADER =
            "order_id,sku,quantity,unit_price_minor,line_total_minor\n";

    /**
     * Streams to a Writer. The full CSV is never held in memory at all, so this is
     * O(n) time and O(1) heap regardless of how many rows there are.
     */
    public void writeCsv(List<OrderLineRow> rows, Writer out) throws IOException {
        out.write(HEADER);

        StringBuilder line = new StringBuilder(96);   // one builder, reused

        for (OrderLineRow row : rows) {
            line.setLength(0);                        // reuse, do not reallocate
            line.append(row.orderId()).append(',')
                .append(row.sku()).append(',')
                .append(row.quantity()).append(',')
                .append(row.unitPriceMinor()).append(',')
                .append(row.lineTotalMinor()).append('\n');
            out.write(line.toString());
        }
        out.flush();
    }
}
```

Four decisions, each removing a different cost:

1. **Stream instead of accumulate.** The biggest win is not `StringBuilder` — it is
   never building the whole document. A 200 MB `String` is a problem even if you build
   it efficiently.
2. **One `StringBuilder`, reused** via `setLength(0)`. A new builder per row would be
   5 million allocations; this is one.
3. **Pre-sized to 96 characters**, an estimate of a row's length. Without it the
   builder starts at 16 and doubles, which costs a handful of copies per row.
4. **`append(char)` not `append(String)`** for the single-character separators —
   `','` avoids materialising a one-character `String`. A small win, but free.

> **If you must return a `String`** — for example the caller is a small API response —
> then `StringBuilder` with a size estimate is the right answer and the streaming point
> does not apply. The decision is driven by output size, not by style. The general rule:
> if the output could exceed a few megabytes, stream it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `+` inside a loop

**Wrong:**
```java
String csv = "";
for (OrderLineRow row : rows) { csv += row.sku() + ","; }
```

**Exact symptom:** wall-clock time grows quadratically with input size — 10k rows in
0.2 s, 100k rows in 20 s, 1M rows never completes. CPU pinned at 100%. GC log
(`-Xlog:gc`) shows a very high young-collection rate. A heap histogram
(`jcmd <pid> GC.class_histogram`) shows `byte[]` at the top by a wide margin. Finally
`OutOfMemoryError: Java heap space`.

**Root cause:** `String` is immutable, so `csv += x` cannot append. It must build a new
string from the old one plus `x`, copying everything. Inside a loop this means the JIT
cannot hoist anything out — each iteration genuinely does allocate a builder, copy the
whole accumulated string into it, append, and produce a new `String`.

The precise mechanism, which is the part interviewers probe: since Java 9, `javac`
compiles `+` to an `invokedynamic` call to `StringConcatFactory.makeConcatWithConstants`
rather than to an explicit `StringBuilder` chain. That is generally *faster* than the
Java 8 desugaring, because the generated method handle can size the result exactly and
copy once.

**But it does nothing for the loop.** The concatenation is one expression evaluated per
iteration; whichever strategy implements it, the accumulated string is still copied
every time. So:

> The problem was never `+`. The problem is `+` **accumulating across iterations**.
> `+` inside a single expression, even a long one, is fine — and on Java 9+ it is
> better than a hand-written `StringBuilder`.

**Fix:** a `StringBuilder` outside the loop, or `String.join`, or a `Collector`, or —
best of all — stream to a `Writer` and never accumulate.

```java
String csv = rows.stream().map(OrderLineRow::sku).collect(Collectors.joining(","));
```

**Prove it:** Proof 1 in the hands-on section shows the bytecode difference directly.

---

### Trap 2 — `==` on strings

**Wrong:**
```java
if (order.currency() == "GBP") { applyUkVat(order); }
```

**Exact symptom:** the branch is taken in unit tests and never taken in production. No
exception, no warning, no log line. You find it from a business metric: VAT is not
being applied, or a status filter returns zero rows, or a feature flag never activates.
The code looks obviously correct in review.

**Root cause:** `==` compares object identity. In the test, `order.currency()` came
from a literal in a fixture, so it is the pooled `"GBP"` and `==` is `true`. In
production it came from `ResultSet.getString` or from Jackson parsing JSON — a freshly
allocated, unpooled `String` — so `==` is `false`.

**Fix:**
```java
if ("GBP".equals(order.currency())) { applyUkVat(order); }
```

Literal-first ordering is a deliberate habit: it is null-safe, so a null currency
returns `false` instead of throwing. (Where a null currency *should* be an error, use
`Objects.requireNonNull` explicitly rather than relying on an NPE from a comparison.)

**Better still:** currency is not a `String`. It is an enum, or `java.util.Currency`.
`==` on an enum is correct, fast, and cannot have this bug. Whenever `==`-versus-
`.equals` on a string is a live question, ask first why the value is a string at all.

---

### Trap 3 — `intern()` as a memory fix

**Wrong:**
```java
// "our heap is full of duplicate SKU strings, so let's intern them"
public void record(OrderLineRow row) {
    this.sku = row.sku().intern();
}
```

**Exact symptom:** three of them, and none looks like the change you made.

1. **Throughput drops** on the path that calls `intern()`. `intern()` is a native call
   that hashes the string and takes a lock on a `StringTable` bucket. At millions of
   calls per minute this is measurable, and it shows up in a profile as time in
   `java.lang.String.intern` — a method you can see, at least.
2. **`intern()` itself gets progressively slower.** The `StringTable` has a fixed
   number of buckets, set at startup. Insert millions of distinct strings and the
   average bucket becomes a long linked list, turning an O(1) lookup into an O(n) scan.
   `-XX:+PrintStringTableStatistics` shows exactly this as a rising average and maximum
   bucket length.
3. **Native memory grows** and does not show up in your heap graphs. Your heap
   dashboard looks *better* after the change, which is the trap: you moved the problem
   somewhere your monitoring does not look. Diagnosing it needs
   `jcmd <pid> VM.native_memory` (**Topic 80**).

**Root cause:** `intern()` deduplicates by *content*, which genuinely saves heap when
you have many duplicate strings. But it buys that with a permanent-ish native table
entry and a per-call hash-and-lock. It is the right tool when the set of distinct
values is **small and bounded** (currency codes, order statuses, country codes) and the
wrong tool when it is **large and unbounded** (SKUs, order IDs, user emails, URLs).

Interning an unbounded set of values is not a fix. It is a leak with better PR.

**Fix, in order of preference:**

1. **Do not have the duplicates.** If SKUs repeat, the values should come from a
   `Map<String, Product>` catalogue you already hold, so every row references the same
   `String` instance naturally. This is deduplication by design and costs nothing at
   runtime.
2. **Use an enum** if the set is genuinely small and fixed. Zero strings, zero
   duplication, and `==` becomes correct.
3. **Let the GC do it.** G1 has `-XX:+UseStringDeduplication`, which deduplicates the
   backing `byte[]` of long-lived strings during collection, with no code change and no
   `StringTable` growth. This is the option most teams should reach for and almost
   nobody knows about. It is not free — it costs some GC time — so measure.
4. **`intern()`**, only after 1–3 are unavailable, only for a bounded value set, and
   only with `-XX:+PrintStringTableStatistics` in your before/after evidence.

---

### Trap 4 — a single non-Latin-1 character doubling a string's memory

**Wrong:**
```java
// product descriptions, imported from a supplier feed
String description = row.getString("description");   // e.g. "Café table — oak"
```

**Exact symptom:** memory usage of the product cache is roughly double your estimate,
and only for *some* products. A heap dump shows two populations of `byte[]` with
distinctly different sizes for descriptions of the same character count. The team
concludes that the heap estimate was "just wrong" and raises `-Xmx`.

**Root cause:** compact strings store one byte per character only if **every**
character in the string fits in Latin-1. The `é` in "Café" and the em-dash `—` push
`coder` to UTF-16, so all 17 characters cost two bytes each instead of one. One
character changed the cost of the whole string.

**Fix:** there is no code fix, and that is the point — this is a *capacity planning*
fact, not a bug. What you do about it:

- Size caches by measurement, not arithmetic. Use JOL or a heap dump on realistic data
  that includes your actual international content, not ASCII test fixtures.
- Know that an internationalised product catalogue costs roughly 2× an English-only one
  in string memory, and say so during design rather than during an incident.
- If a specific large field is the problem, store it compressed or out of heap and
  decode on demand — but only with a heap dump proving it is the problem
  (**Topic 79**).

You can verify the mechanism directly with `-XX:-CompactStrings`; see Proof 4.

---

### Trap 5 — string concatenation inside a log call

**Wrong:**
```java
log.debug("placing order " + order + " for customer " + customer + " total " + total);
```

**Exact symptom:** measurable CPU and allocation on a code path where logging is
**disabled**. In a profile you see time inside `Order.toString()` and
`StringConcatFactory` from a line that produces no log output at all. On a hot path
this can be several percent of total CPU, spent producing strings that are immediately
discarded.

**Root cause:** the argument is evaluated before `debug()` is called. Java has no lazy
argument evaluation. The level check happens *inside* `debug`, by which time you have
already built the string and called `toString()` on two domain objects.

**Fix:** SLF4J's parameterised form. The format string is a constant; the arguments are
only formatted if the level is enabled.

```java
log.debug("placing order {} for customer {} total {}", order, customer, total);
```

Or, for a genuinely expensive argument, the supplier form:

```java
log.atDebug().setMessage("order tree: {}").addArgument(() -> order.deepDescription()).log();
```

This is the highest-value habit in this entire document, because it applies to every
log line you will ever write. **Topic 120** covers logging properly.

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM and I am not going to print
output and claim it is real. What follows is the exact command, what to look for, and
how to read each possible outcome.

### Setup

```bash
mkdir -p ~/java-lab/18 && cd ~/java-lab/18
java --version      # expect 21 or 25
```

### Proof 1 — what `+` actually compiles to

`ConcatShapes.java`:
```java
public class ConcatShapes {

    // ONE expression: fine on Java 9+
    static String oneExpression(String sku, int qty) {
        return "sku=" + sku + " qty=" + qty;
    }

    // ACCUMULATING across iterations: the quadratic defect
    static String inALoop(String[] skus) {
        String out = "";
        for (String sku : skus) { out += sku + ","; }
        return out;
    }

    // The correct shape
    static String withBuilder(String[] skus) {
        StringBuilder sb = new StringBuilder(skus.length * 12);
        for (String sku : skus) { sb.append(sku).append(','); }
        return sb.toString();
    }
}
```

```bash
javac ConcatShapes.java
javap -c ConcatShapes.class
```

**What to look for**, method by method:

| Method | What you should find |
|---|---|
| `oneExpression` | A single `invokedynamic` instruction whose name resolves to `makeConcatWithConstants`. No `StringBuilder` anywhere. |
| `inALoop` | An `invokedynamic makeConcatWithConstants` **inside the loop body** — you can tell because the branch at the end of the method jumps back to a target above it. |
| `withBuilder` | `new java/lang/StringBuilder` **before** the loop, `invokevirtual StringBuilder.append` inside it, `StringBuilder.toString` after it. |

**How to read it:**

| What you see | What it means |
|---|---|
| `invokedynamic ... makeConcatWithConstants` | You are on Java 9 or later. `javac` delegates concatenation to `StringConcatFactory`, which generates a method handle at first execution that sizes and copies exactly once. |
| `new java/lang/StringBuilder` in `oneExpression` | You are compiling with a Java 8 `javac`, or with `-XDstringConcat=inline`. This is the **`[LEGACY — still asked]`** shape: Java 8 desugared `+` into an explicit `StringBuilder` chain. |
| The `invokedynamic` in `inALoop` sits between a loop start label and a backward `goto` | This is the proof of the whole trap. One concat instruction, executed n times, each time copying an ever-longer string. The instruction is cheap; the loop makes it quadratic. |

**The point to take away:** `javap` shows you that `oneExpression` and `inALoop` contain
the *same instruction*. The bytecode does not distinguish "good concat" from "bad
concat". Only the loop does. That is why "use `StringBuilder` instead of `+`" is bad
advice and "never accumulate a string across iterations" is good advice.

> **A version-sensitive point I will not guess at.** There used to be a system property
> `-Djava.lang.invoke.stringConcat=<strategy>` for selecting between concatenation
> strategies (`BC_SB`, `MH_INLINE_SIZED_EXACT`, and others). I am not confident which
> strategies, if any, remain selectable on your JDK — several were removed in later
> releases. Settle it yourself:
> ```bash
> java -Djava.lang.invoke.stringConcat=BC_SB ConcatShapes.java
> ```
> If it runs, the property still works on your build. If it fails with an error naming
> the property or the strategy, it does not. Either way you now know, rather than
> believing.

### Proof 2 — pooling and identity

`PoolProof.java`:
```java
public class PoolProof {
    public static void main(String[] args) {
        String literal = "SKU-4471";
        String built    = new String("SKU-4471");
        String runtime  = "SKU-" + args.length + "471";   // args.length is not a constant
        String interned = built.intern();

        System.out.println("literal == built    : " + (literal == built));
        System.out.println("literal == interned : " + (literal == interned));
        System.out.println("literal == runtime  : " + (literal == runtime));
        System.out.println("literal.equals(runtime): " + literal.equals(runtime));
        System.out.println("identityHashCode literal : "
                + System.identityHashCode(literal));
        System.out.println("identityHashCode built   : "
                + System.identityHashCode(built));
        System.out.println("identityHashCode interned: "
                + System.identityHashCode(interned));
    }
}
```

```bash
java PoolProof.java 0 4    # args.length == 2, so runtime builds "SKU-2471" -> not equal
java PoolProof.java 0 4 4 4    # args.length == 4 -> "SKU-4471", equal content
```

**What to look for:** the identity hash codes, and specifically whether `literal` and
`interned` share one while `built` has a different one.

| What you see | What it means |
|---|---|
| `literal == built` is `false` | Expected. `new String` allocated a duplicate. |
| `literal == interned` is `true`, and their identity hashes match | Expected, and it is the definition of `intern()`: it returned the pooled instance. |
| Second run: `literal == runtime` is `false` but `.equals` is `true` | The most important line in the proof. Identical content, different objects, because the runtime-built string was never pooled. This is exactly the production-versus-test asymmetry from Trap 2. |
| `literal == runtime` is `true` in either run | Your JVM or compiler folded something you did not expect. Check whether you accidentally used a constant instead of `args.length`. |

### Proof 3 — look inside the string table

Run any long-lived Java program and ask the JVM to dump `StringTable` statistics at
exit:

```bash
java -XX:+PrintStringTableStatistics PoolProof.java 0 4
```

**What to look for:** a table printed at JVM shutdown containing, for the *StringTable*
section: the number of buckets, the number of entries, the average bucket size, the
maximum bucket size, and the total footprint in bytes.

| What you see | What it means |
|---|---|
| Number of buckets much larger than number of entries; average bucket size well under 1 | A healthy table. Lookups are effectively O(1). This is the normal state for an application that does not call `intern()`. |
| Average bucket size rising above ~2 and maximum bucket size in the tens | The table is overloaded. Every `intern()` call now scans a chain. This is the symptom of Trap 3, and the fix is either to stop interning or to raise `-XX:StringTableSize`. |
| The flag is rejected as unrecognised | Add `-XX:+UnlockDiagnosticVMOptions` before it and retry. Some diagnostic flags require that gate depending on the build. |

To inspect a **running** service instead of one at exit:

```bash
jps -l                                  # find the pid
jcmd <pid> VM.stringtable
jcmd <pid> VM.stringtable -verbose      # per-bucket detail; large output
```

If `jcmd` reports the command as unrecognised or refuses it, start the target JVM with
`-XX:+UnlockDiagnosticVMOptions` — `VM.stringtable` is a diagnostic command and I am
not certain it is ungated on every build.

And to see and change the table's size:

```bash
java -XX:+PrintFlagsFinal -version | grep -i StringTableSize
java -XX:StringTableSize=1000003 -XX:+PrintStringTableStatistics PoolProof.java
```

**How to read the size:** the default is a fixed number chosen at JVM build time (it has
changed across releases, which is precisely why you print it rather than quote it).
`StringTableSize` should be a prime-ish number for good bucket distribution;
`1000003` is prime and is the conventional choice when people raise it. Raising it
costs native memory and fixes nothing unless `PrintStringTableStatistics` showed you
long buckets first.

### Proof 4 — compact strings are real and measurable

`CompactProof.java`:
```java
public class CompactProof {
    public static void main(String[] args) {
        int n = 200_000;
        String[] ascii = new String[n];
        String[] accented = new String[n];

        for (int i = 0; i < n; i++) {
            ascii[i]    = "Product description number " + i + " plain ascii text";
            accented[i] = "Product description number " + i + " café text";  // é
        }

        System.gc();
        Runtime rt = Runtime.getRuntime();
        long used = rt.totalMemory() - rt.freeMemory();
        System.out.println("approx used bytes: " + used);
        System.out.println("keep alive: " + ascii[0].length() + accented[0].length());
    }
}
```

Run it three ways:

```bash
java -Xmx1g CompactProof.java                       # default: compact strings ON
java -Xmx1g -XX:-CompactStrings CompactProof.java   # forced UTF-16 for everything
java -XX:+PrintFlagsFinal -version | grep -i CompactStrings
```

**What to look for:** the difference in reported used bytes between the first two runs.

| What you see | What it means |
|---|---|
| Run 2 uses noticeably more memory than run 1 | Expected. With `-XX:-CompactStrings` every string is UTF-16, so the ASCII half of the data doubles. You have just measured the feature. |
| `CompactStrings` prints as `true` in the flags dump | The default on Java 9+. Confirmed on your build rather than assumed. |
| Almost no difference | Your heap is dominated by something other than the string bytes, or `Runtime.totalMemory()` has not shrunk to the live set. This measurement is crude by design — `Runtime` memory numbers are approximate and GC-timing-dependent. Use JOL or a heap histogram for a real number. |

**Honest caveat on this proof:** `Runtime.totalMemory() - freeMemory()` is a rough
instrument. It measures the heap the JVM currently holds, not the live set, and
`System.gc()` is a request rather than a command. It is good enough to see a ~2×
difference and not good enough for anything finer. The precise tool is a heap histogram:

```bash
java -Xmx1g CompactProof.java &     # note the pid
jcmd <pid> GC.class_histogram | head -20
```

Look at the `[B` (byte array) row's total size. Proper heap analysis is **Topic 79**.

### Proof 5 — the `hashCode` cache

`HashCache.java`:
```java
public class HashCache {
    public static void main(String[] args) throws Exception {
        String key = new String("a-fairly-long-order-reference-string-value");

        java.lang.reflect.Field hashField = String.class.getDeclaredField("hash");
        hashField.setAccessible(true);

        System.out.println("hash field before first hashCode(): " + hashField.get(key));
        key.hashCode();
        System.out.println("hash field after  first hashCode(): " + hashField.get(key));
    }
}
```

```bash
java --add-opens java.base/java.lang=ALL-UNNAMED HashCache.java
```

**What to look for:** whether the field reads `0` before and non-zero after.

| What you see | What it means |
|---|---|
| `0` then a non-zero number | Confirmed: `String` computes its hash lazily and caches it in a mutable field. This is why a `String` map key is cheap after its first use, and it is a deliberate benign race (Topic 86). |
| `InaccessibleObjectException` | You omitted `--add-opens`. Note this: you have just met **Topic 20**'s strong encapsulation from the other side — the JDK will not let you reflect into `java.lang` without explicit permission. |
| A different exception naming the field | The internal field name may differ on your build. Run `javap -p java.lang.String` to see the real field names, then adjust. |

This proof is deliberately reaching into JDK internals, which is exactly what production
code should never do. It is here because it is the fastest way to see the caching
behaviour, and because meeting `--add-opens` now makes Topic 20 land better.

---

## Practice exercises

Write real files, run them, and keep your output.

### 1 — Easy: predict, then verify

Write a program with **ten** string-comparison cases and, for each, write down your
prediction of `==` and `.equals` **before running it**. Include at least:

- two identical literals
- a literal versus `new String(...)` of the same text
- a literal versus the result of `.intern()`
- a literal versus a runtime concatenation using a method parameter
- a literal versus a compile-time-constant concatenation of two `static final String`s
- a literal versus `"SKU-4471".substring(0, 8)` where the substring covers the whole
  string (predict carefully — there is a special case)
- a literal versus a string read from `System.getenv` or `args`
- `"".equals(s)` versus `s.isEmpty()` where `s` is null
- `" ".isBlank()` versus `" ".isEmpty()`
- `" x ".strip()` versus `" x ".trim()` — find an input where they differ

Report your score out of ten, and for every one you got wrong, write the one-sentence
rule you were missing.

### 2 — Medium: string keys under the microscope (combines Topics 01, 12, 13, 17)

Build `SkuIndex`, a `Map<String, Product>` holding 500,000 products with SKU keys of
the form `"SKU-" + i`.

**Part A.** Populate it. Then look up all 500,000 keys, building each lookup key
freshly with `"SKU-" + i` so it is never the same object as the stored key. Confirm
every lookup succeeds, and explain in one sentence why `HashMap` does not care that the
key objects differ, referring to the `equals`/`hashCode` contract (Topic 13).

**Part B.** Now change the lookup to `map.get(key) == storedKey` style identity
comparison and show it failing. Connect this to Trap 2, and to Topic 01's `Integer`
cache — say precisely what the two bugs have in common and where the analogy breaks
down.

**Part C.** Measure `String.hashCode()` cost. Time 10 million `hashCode()` calls on a
**fresh** string each time versus on the **same** string each time. Explain the
difference using Proof 5.

**Part D.** Replace the `String` keys with a `record SkuKey(long number)` (Topic 17 /
Topic 27) and compare the memory footprint using `jcmd GC.class_histogram`. Then answer
honestly: is the saving worth the loss of readability in logs and debugger views?
State the condition under which your answer flips.

### 3 — Hard: production simulation — the `orderflow` export

Build the nightly order-line export three ways and measure all three.

**Part A — three implementations.**
1. `NaiveExporter` — `csv += ...` in the loop, exactly as in Example 2's broken version.
2. `BuilderExporter` — one `StringBuilder`, pre-sized, returns a `String`.
3. `StreamingExporter` — writes to a `BufferedWriter` over a file, holds nothing.

**Part B — measure.** Run each at 10,000 / 100,000 / 1,000,000 rows with:

```bash
java -Xmx512m -Xlog:gc:file=naive.log:time,uptime NaiveExporter 10000
```

Record wall-clock time and the count of young collections in each GC log. **Do not run
the naive version at 1,000,000** unless you want to watch it fail — and if you do, run
it with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./dumps` and keep the dump for
Topic 79.

Plot time against row count for each version. State which curves are linear and which
is not, and give the reason in terms of bytes copied, not in terms of "efficiency".

**Part C — internationalise it.** Make 30% of the product names contain a non-Latin-1
character. Re-run the builder version and compare memory. Confirm your prediction from
Trap 4, and say by how much your original capacity estimate was wrong.

**Part D — the intern experiment.** The export writes each product's category name on
every row, and there are only 40 distinct categories across 1,000,000 rows. Implement
category handling three ways: (a) plain strings from the row, (b) `.intern()` on every
row, (c) an `EnumMap`/enum lookup (Topic 16). Measure heap and throughput for all three,
with `-XX:+PrintStringTableStatistics` on run (b).

Then write the paragraph you would put in a PR description recommending one of them.
It must include a number, and it must say what you would need to see to change your
recommendation.

> These are wall-clock measurements, which **Topic 77** will show you are unreliable for
> small differences. They are perfectly adequate for a quadratic-versus-linear
> difference, which is the point here. Note which of your conclusions actually depend on
> precise timing and which do not.

---

## Interview questions

### Q1 — "Should you use `StringBuilder` instead of `+`?"

**Mid-level answer:** "Yes, `+` creates a new `String` each time, so `StringBuilder` is
faster."

**Senior answer:** "It depends where the `+` is, and since Java 9 the usual advice is
out of date. `javac` compiles `+` to an `invokedynamic` call into `StringConcatFactory`,
which generates a method handle that sizes the result and copies once — for a single
expression that is typically as fast as or faster than a hand-written `StringBuilder`,
and it is more readable. The real problem is `+` that *accumulates across loop
iterations*, because then you copy the whole accumulated string every iteration and
you have a quadratic algorithm. So the rule I actually apply is: `+` freely inside one
expression, never accumulate a string across iterations. And above a few megabytes of
output the right answer is neither — it is to stream to a `Writer` and never hold the
document at all. You can see all of this in `javap -c`; the bytecode for the good and
bad cases contains the same instruction, which is why the rule has to be about the loop
rather than about the operator."

**What separates them:** knowing the Java 9 change, and reframing the rule from
"operator" to "accumulation". The mid answer is a Java 6 answer that people still
repeat.

**Follow-up:** "What changed in Java 9 and why did they do it?" They want
`StringConcatFactory` and the reason: moving the strategy from bytecode `javac` emits
into a runtime-generated method handle means the JDK can improve concatenation without
recompiling anyone's code.

---

### Q2 — "Why is `new String("x") != "x"`?"

**Mid-level answer:** "Because `new` creates a new object, and `==` compares
references."

**Senior answer:** "Because `==` compares identity and they are two objects. The
literal `"x"` is interned at class load — the JLS requires literals to be pooled, so
every occurrence of `"x"` in the whole JVM is one object. `new String("x")` explicitly
allocates a second `String` that copies the same contents, so identity differs while
content matches. `.intern()` on it returns you the pooled one. In practice
`new String(literal)` has no legitimate use — the constructor is not deprecated, unlike
the wrapper constructors, but it is always pointless. The reason this matters beyond
trivia is the asymmetry it creates between tests and production: a test comparing two
literals with `==` passes, and the same comparison against a value from Jackson or JDBC
fails, with no exception to tell you."

**What separates them:** naming the JLS requirement rather than calling pooling an
implementation detail, and immediately going to the test-versus-production failure
shape rather than stopping at the mechanism.

**Follow-up:** "Where does the pool live?" They want: interned `String` objects are on
the heap since Java 7 — before that, PermGen — while the lookup table is a native JVM
structure whose size is fixed by `-XX:StringTableSize`.

---

### Q3 — "Our heap is full of duplicate strings. Should we call `intern()`?"

**Mid-level answer:** "Yes, `intern()` deduplicates them so you only keep one copy."

**Senior answer:** "Probably not, and I'd want to know the cardinality first. `intern()`
helps when the set of distinct values is small and bounded — currency codes, order
statuses. For an unbounded set like SKUs or user emails it makes things worse: you pay a
native hash-and-lock on every call, the fixed-size `StringTable` degrades into long
buckets so `intern()` itself gets slower, and the memory moves from the heap — which
your dashboards watch — into native memory, which they do not. So the graph improves
while the service does not.
What I'd try in order: fix the duplication at the source, so rows reference the same
`String` from a catalogue map I already hold; use an enum if the set is genuinely fixed;
turn on `-XX:+UseStringDeduplication`, which lets G1 dedupe the backing arrays of
long-lived strings during collection with no code change and no `StringTable` growth.
And whichever I pick, I'd want `PrintStringTableStatistics` and a heap histogram
before and after, because 'the heap graph went down' is not the same as 'we use less
memory'."

**What separates them:** the cardinality question, knowing the failure moves to native
memory, and knowing `UseStringDeduplication` exists — most candidates do not.

**Follow-up:** "How would you know whether it worked?" They want a before/after
comparison using a heap histogram plus NMT, not the heap-used graph alone.

---

### Q4 — "What are compact strings and what do they cost you?"

**Mid-level answer:** "Since Java 9 strings use a `byte[]` instead of a `char[]`, so
ASCII strings use half the memory."

**Senior answer:** "A `String` holds a `byte[]` plus a one-byte `coder` flag. If every
character fits in Latin-1 the array is one byte per character; otherwise it is UTF-16 at
two bytes per character. The important consequence is that it is per-*string*, not
per-character: one accented character or one emoji anywhere in the string doubles the
storage for that entire string. So an internationalised product catalogue costs roughly
double an English-only one, and that is a capacity-planning fact rather than something
you can code around. It is measurable — `-XX:-CompactStrings` turns it off, so you can
run both ways and diff a heap histogram. It also means a heap dump from an ASCII test
dataset understates the memory of a real one, which is a very easy way to size a cache
wrongly."

**What separates them:** knowing the flag is per-string rather than per-character, and
turning it into a statement about how you size things and how test data misleads you.

**Follow-up:** "Did anything have to change in `String`'s API?" No — the API is
unchanged and takes `char`; the byte representation is entirely internal, which is why
the change could ship in a point release of the platform without breaking anyone.

---

### Q5 — "`substring` used to be a memory leak. Explain."

**Mid-level answer:** "I think old versions of Java shared the array between the
original string and the substring."

**Senior answer:** "**`[LEGACY — still asked]`** — before Java 7 update 6, `String` held
a `char[]` plus an offset and a count, and `substring` returned a new `String` sharing
the *same* array with different offsets. That made `substring` O(1), which was the
point. The failure mode: parse a 4 MB log line, keep an 8-character substring of it, and
you retain the whole 4 MB array for as long as you hold the substring — a leak with no
visible cause, because the small string looks small in every tool that shows you
lengths. It was changed in 7u6 to copy, making `substring` O(n) but eliminating the
retention. The reason it is still worth knowing: it is a clean example of retention
versus allocation — the memory was allocated legitimately and the bug was that it stayed
reachable, which is exactly how every real Java leak works. It also shows that changing
a complexity guarantee can be the right call when the alternative is an invisible
correctness-of-footprint problem."

**What separates them:** dating the change precisely, naming the retention-versus-
allocation distinction, and treating it as a design-trade story rather than trivia.

**Follow-up:** "So is `substring` free now?" No — it copies, so slicing a large string
repeatedly in a loop is now an allocation cost you can see. Different trade, still a
trade.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `String` is immutable, yet it caches its `hashCode` in a non-final field that it
   writes after construction. Explain why this does not break immutability, and why the
   data race between two threads computing it simultaneously is harmless. What property
   of the computed value makes it harmless?

2. The pool makes `==` accidentally work for literals. Argue that this is *worse* than
   if `==` had never worked for strings at all, using the same reasoning you would apply
   to Topic 01's `Integer` cache.

3. `String` is a `final` class holding a `private final byte[]`. The array is a mutable
   object. Why is `String` nevertheless genuinely, deeply immutable? Name the exact
   discipline it follows, in the vocabulary of Topic 17.

4. Compact strings save memory for Latin-1 text and cost a branch on every character
   access. Under what workload would this be a net loss, and why did the JDK team ship
   it as the default anyway?

5. Java's string templates were proposed and then withdrawn before finalising, so you
   have `formatted()` and `+` rather than `` `${x}` ``. Given what you now know about
   `StringConcatFactory`, what would a string template feature have to be careful about
   that plain interpolation in TypeScript does not?

6. If you could change one thing about Java's string design, would you remove `==` for
   `String` entirely (making it a compile error), or remove automatic literal pooling?
   Argue for one, then make the strongest case against yourself.

7. A colleague replaces every `+` in the codebase with `StringBuilder`, including
   single-expression concatenations, "for performance". Beyond readability, name one
   way this could make the code measurably *slower*.

---

## Quick reference card

### Identity versus content

| Expression | Result | Why |
|---|---|---|
| `"a" == "a"` | `true` | both literals, same pooled object |
| `"a" == new String("a")` | `false` | `new` allocated a second object |
| `"a" == new String("a").intern()` | `true` | `intern()` returns the pooled one |
| `("a" + variable) == "ab"` | `false` | runtime concat is not pooled |
| `(FINAL_A + "b") == "ab"` | `true` | compile-time constant folding |
| `"a".equals(x)` | content | **this is what you want** |

### The APIs that matter

```java
s.equals(t) / s.equalsIgnoreCase(t) / Objects.equals(s, t)
s.isEmpty()          // length == 0
s.isBlank()          // Java 11+: empty or only whitespace
s.strip()            // Java 11+: Unicode-aware trim. Prefer over trim().
s.trim()             // legacy: only strips chars <= U+0020
s.repeat(n)          // Java 11+
s.lines()            // Java 11+: Stream<String> of lines
s.chars()            // IntStream of code units (Topic 23)
s.formatted(args)    // Java 15+: same as String.format(s, args)
String.join(",", list)
String.valueOf(x)    // null-safe; produces "null"
s.intern()           // pool it. See Trap 3 before using.
s.split(regex)       // compiles a regex every call — hoist a Pattern in hot paths
```

### Building strings — pick by output size

| Output size | Use |
|---|---|
| One expression | `+` — it is `invokedynamic` and it is fine |
| A list joined by a delimiter | `String.join` or `Collectors.joining` |
| Kilobytes, built in a loop | `StringBuilder`, pre-sized, declared outside the loop |
| Megabytes or unbounded | Stream to a `Writer`. Do not build a `String` at all. |
| Shared across threads | Rethink the design. `StringBuffer` is the legacy answer. |

### Memory facts

- ASCII string: ~1 byte per character + object header + the `byte[]` header.
- Any non-Latin-1 character: **the whole string** becomes 2 bytes per character.
- Interned strings live on the heap (Java 7+) and are collectable; the `StringTable` is
  native and fixed-size.
- `-XX:+UseStringDeduplication` (G1): dedupes backing arrays during GC, no code change.
- `-XX:StringTableSize=<prime>`: raises the bucket count. Costs native memory.
- `-XX:-CompactStrings`: forces UTF-16. Diagnostic only.

### Gotchas

- [ ] Never `==` between strings. `.equals`, literal-first for null safety.
- [ ] Never accumulate a string across loop iterations.
- [ ] Never `new String("literal")`.
- [ ] Never concatenate inside a `log.debug(...)` argument. Use `{}` placeholders.
- [ ] `intern()` only for a small bounded value set, only with before/after numbers.
- [ ] `split(regex)` recompiles the pattern; hoist a `Pattern` in a hot loop.
- [ ] `strip()` not `trim()` for anything that might contain Unicode whitespace.
- [ ] A `String` is a poor domain type. Currency, status and country want enums.

---

## When would I use this at work?

**1. Code review, on almost every pull request.**
Two patterns you will catch by eye forever after this topic: a string accumulated in a
loop, and a concatenated log argument. Both are invisible to tests, both are real CPU,
and both take five seconds to spot once the shape is in your eye. This is the highest
frequency payoff in Phase 1.

**2. Reading a heap dump when a service is using more memory than budgeted.**
`byte[]` and `String` will be at or near the top of the histogram. Knowing that a
`byte[]` there is *string contents*, that non-Latin-1 content doubles it, and that
duplicate values might be fixable at the source rather than by interning, is the
difference between a diagnosis and a shrug. Topic 79 gives you the tooling; this gives
you the vocabulary to interpret it.

**3. Designing a high-volume export, import or serialisation path.**
The moment someone says "we need to generate a CSV/JSON/XML of everything", the
decision is "stream it or build it", and that decision is made in five minutes at design
time or in a four-hour incident later. Being the person who says "how big is the output
going to get?" before any code is written is a senior behaviour that costs nothing.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and wrappers**: the `Integer` cache is the same shape of bug as the
  string pool — a shared-instance optimisation that makes `==` accidentally work.
- **13 — equals/hashCode contract**: `String`'s implementation of both, and why a
  cached hash is safe for an immutable key.
- **17 — Immutability**: `String` is the JDK's canonical immutable class and follows the
  exact defensive discipline that topic describes. The pool only works *because*
  strings are immutable.

**This unlocks:**
- **19 — Serialization**: strings dominate serialized payloads; `String` has custom
  serialization behaviour, and JSON field names are a string-heavy cost.
- **20 — JPMS**: Proof 5 needs `--add-opens java.base/java.lang=ALL-UNNAMED`, which is
  that topic's central flag.
- **23 — Streams**: `chars()`, `lines()`, and `Collectors.joining` as the idiomatic
  builders.
- **30 — Text blocks**: multi-line string literals, and the interpolation Java does not
  have.
- **69 — Object layout**: the exact byte cost of a `String` and its `byte[]`, verified
  with JOL rather than estimated.
- **76 — Reading bytecode**: `invokedynamic` and `StringConcatFactory` properly, rather
  than "look for this instruction".
- **77 — JMH**: the correct way to measure the concatenation differences you wall-clocked
  in Exercise 3.
- **79 — Memory leaks and heap dumps**: strings will be your largest retained set;
  this is the topic where you learn to attribute them.
- **80 — Off-heap memory**: where the `StringTable` lives, and how to see native growth
  that the heap graph hides.
- **120 — Logging and MDC**: parameterised logging, and why Trap 5 is the most repeated
  performance defect in Java codebases.

---

*Java baseline 21. Compact strings (Java 9) and `StringConcatFactory` concatenation
(Java 9) are both long-settled and unchanged in 21 and 25. The genuinely uncertain
points in this document — which `java.lang.invoke.stringConcat` strategies still exist,
the current default `StringTableSize`, and whether `jcmd VM.stringtable` needs
`-XX:+UnlockDiagnosticVMOptions` on your build — each have a settling command next to
them. Run those rather than trusting the prose.*
