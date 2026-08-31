# 69 — Object Layout: Headers, Alignment, Compressed Oops, and Measuring Real Size

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you answer "how much heap does one `orderflow` catalogue entry actually cost, and how many can I hold in 1200 MB" — with a number you measured, not a number you guessed. Every claim here is checked against the Topic 65 baseline in `/docs/java/baselines/`.

---

## Mechanical statement

Read this three times. The rest of the document is an elaboration of it.

> **Every Java object begins with a header.** On 64-bit HotSpot with compressed oops
> enabled, that header is **12 bytes**: an 8-byte **mark word** and a 4-byte **class
> word**. An array carries a third header field — a 4-byte **length** — making an array
> header **16 bytes**.
>
> **After the header come the instance fields, and the JVM chooses their order.** Not
> your declaration order. HotSpot reorders fields to minimise padding, packing wide
> types first and filling the gaps with narrow ones, with superclass fields laid out
> before subclass fields.
>
> **The whole object is then padded up to a multiple of 8 bytes.** That is
> `ObjectAlignmentInBytes`, and 8 is its default. An object whose fields end at byte 20
> occupies 24. Four bytes you paid for and cannot use.
>
> **A reference — an "oop", an ordinary object pointer — is 8 bytes on a 64-bit
> machine. Compressed oops make it 4.** The JVM stores a 32-bit value and reconstructs
> the real address by shifting left by 3 (because every object starts on an 8-byte
> boundary, the low three bits are always zero and carry no information) and adding a
> base. Thirty-two bits shifted by three addresses **2^35 bytes — 32 GiB**. Above that,
> the encoding cannot represent the heap, and the JVM **silently turns compressed oops
> off**.
>
> **Which is why a 31 GB heap can hold more objects than a 33 GB one.**

Five consequences follow directly, and you should be able to derive each one:

1. **An `Integer` is not four bytes.** It is a 12-byte header plus a 4-byte `int`,
   already a multiple of 8, so **16 bytes** — a 4× overhead over the primitive. That is
   the whole reason `List<Integer>` costs multiples of `int[]`.
2. **The header dominates small objects.** For anything holding less than about 20
   bytes of actual data, most of what you allocated is bookkeeping.
3. **Adding a field to a class sometimes costs 8 bytes and sometimes costs nothing.**
   It depends on whether there was a padding gap to fill. You cannot reason about this
   from the source; you must measure it.
4. **Crossing the compressed-oops threshold makes every reference field and every class
   word twice as wide.** Your live set grows, with no code change, purely from a heap
   flag. You added RAM and lost capacity.
5. **The only honest answer to "how big is this object" is the output of a tool.** Not
   arithmetic, not a table in a document, not this document. Arithmetic is how you form
   a hypothesis. JOL is how you settle it.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

This is the literal, unhedged version. There is nothing to transfer. Read this section
as a list of questions you have never been able to ask, rather than as a mapping.

**1. You cannot ask V8 how many bytes an object occupies.**
There is no API. `Object.keys(x).length` counts properties. `JSON.stringify(x).length`
counts characters in a serialisation that has nothing to do with the in-memory
representation. A heap snapshot in Chrome DevTools gives you a "shallow size" and a
"retained size" per object, which is the closest thing that exists — and it is a
debugging view of an implementation detail, not a documented, stable, programmatically
queryable fact. There is no V8 equivalent of `ClassLayout.parseInstance(x)` that prints
byte offsets of fields. **In Java, object size is a measurable, reproducible property of
a class, and you are now expected to know it for the classes on your hot paths.**

**2. There is no compressed-oops equivalent to reason about.**
V8 has pointer compression, and it is genuinely similar in spirit — 32-bit tagged values
inside a 4 GB cage. But you do not configure it, you do not choose it, you do not have a
threshold to plan around, and crucially **you do not have a decision to make about heap
size that silently changes the width of every reference in your program.** No Node
engineer has ever had to say "we cannot go above 31 GB of heap because the thirty-second
gigabyte costs us capacity". That sentence is unintelligible in Node and is a normal
capacity-planning statement in Java.

**3. There is no object header to reason about.**
V8 objects have a map pointer (the hidden class) and properties, and Node's memory
advice stops at "objects have overhead, don't make millions of them". Java's header has
a *documented job*: the mark word holds identity hash code, GC age bits, and lock state,
which is why Topic 85 (`synchronized`, monitors, lock inflation) is a story about bits
in this header, and why a GC that moves objects writes a forwarding pointer into it.
**The header is not just overhead. It is the JVM's per-object working memory, and three
other topics in this phase are about what it holds.**

**4. There is no field-reordering question.**
Your TypeScript object's property order affects V8's hidden-class transitions, which is
a real performance concern — but it is about *shape identity*, not about *byte layout*.
Java's reordering is about padding and cache lines, and it is the reason two classes
with identical fields declared in different orders can have identical size (the JVM
fixed it for you) while two classes with one extra `boolean` differ by 0 or 8 bytes.

**5. "Sizeof" is not a language feature in either language — but only Java has a tool that
answers it exactly.** JOL, from the OpenJDK project itself, reads the live VM's layout
decisions and prints them. This is not an estimate.

| You know | Java | Verdict |
|---|---|---|
| Object property count | Object header + field offsets + padding | **NO ANALOGUE** |
| DevTools heap snapshot "shallow size" | `ClassLayout.parseInstance(x).instanceSize()` | **NO ANALOGUE** — the Java one is exact, stable and scriptable |
| DevTools "retained size" | `GraphLayout.parseInstance(x).totalSize()` and MAT's retained set | **PARTIAL at best** — see the Trap section; they answer *different* questions |
| V8 pointer compression (automatic, invisible) | Compressed oops with a heap-size threshold you must plan around | **NO ANALOGUE** |
| Hidden-class shape transitions | Field reordering for alignment | **NO ANALOGUE** — different mechanism, different consequence |
| `--max-old-space-size` | `-Xmx` — which now also decides your reference width | **NO ANALOGUE** for the second half of that sentence |

> **The theme of this topic:** you have been handed a ruler. Everything about object
> representation that was previously an unmeasurable folk belief in your career is now a
> number you can print in ten seconds. The obligation that comes with the ruler is that
> **"I think this is small" stops being an acceptable answer in a design review.**

---

## What is this?

An "object layout" is the byte-by-byte arrangement of one Java object in the heap. It has
three parts, in this order:

| Part | Size (64-bit HotSpot, compressed oops on) | What it holds |
|---|---|---|
| **Mark word** | 8 bytes | identity hash code, GC age, lock state / monitor pointer, GC forwarding pointer during evacuation |
| **Class word** (klass pointer) | 4 bytes compressed, 8 uncompressed | pointer to the class metadata in metaspace — the object's type at runtime |
| **Array length** | 4 bytes, **arrays only** | the element count |
| **Instance fields** | sum of field widths, JVM-chosen order | your data |
| **Padding** | 0–7 bytes | to reach a multiple of `ObjectAlignmentInBytes` |

So: **object header = 12 bytes. Array header = 16 bytes.** Those are the two numbers to
carry in your head, and both are spec-level facts of 64-bit HotSpot with compressed oops
enabled — but **verify, don't trust me**:

```bash
java -XX:+PrintFlagsFinal -version | grep -E "UseCompressedOops|UseCompressedClassPointers|ObjectAlignmentInBytes"
```

Field widths, which you already know but must have exact for the arithmetic:

| Type | Bytes |
|---|---|
| `boolean`, `byte` | 1 |
| `char`, `short` | 2 |
| `int`, `float` | 4 |
| `long`, `double` | 8 |
| any reference | **4 with compressed oops, 8 without** |

### The two independent compression flags

People conflate these. They are separate:

- **`UseCompressedOops`** — compresses *references between objects*: your fields, array
  elements, everything of reference type.
- **`UseCompressedClassPointers`** — compresses the *class word in the header* from 8
  bytes to 4, backed by a dedicated region called **compressed class space**
  (`-XX:CompressedClassSpaceSize`, default 1 GB of reserved virtual address space).

Historically the second implies the first is on. Turning compressed oops off has usually
turned compressed class pointers off as well, taking the header from 12 to 16 bytes.
**Do not memorise the coupling — print both flags on your JDK.**

### `[JAVA 25]` Compact object headers

Recent JDKs add **compact object headers**, which fold the class word into the mark word
and take the object header from **12 bytes to 8**. It arrived as an experimental feature
and has been progressing toward a product feature; on JDK 25 the flag is
`-XX:+UseCompactObjectHeaders`.

> **One-line version note, and it is a genuine uncertainty:** I am **not** asserting
> whether `UseCompactObjectHeaders` is on by default on your JDK 25 build, or whether it
> still requires `-XX:+UnlockExperimentalVMOptions` — that has moved between releases and
> differs by vendor build. **Settle it in ten seconds with
> `java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders` and then confirm
> the consequence with JOL**, because if it is on, **every size prediction in this
> document is 4 bytes too large for non-array objects, and some objects will land in a
> smaller alignment bucket.** That is not a rounding error; for a heap full of small
> objects it is a double-digit percentage of your live set.

This is the single most important reason the drill in this document is *measurement-shaped*
rather than *table-shaped*. A table of object sizes has a shelf life. The habit of running
JOL does not.

---

## Why does it matter?

**1. Because capacity planning is arithmetic, and you cannot do arithmetic without the
unit.**

`orderflow` holds a product catalogue cache (Topic 37 built it, Topic 110 will make it
Redis-backed). Someone will ask "can we cache all 100k products in-process?" That is a
multiplication: *cost per cached entry × 100 000*, compared against the heap you have
after subtracting your live set and your headroom. If you do not know the cost per entry
to within ~20%, you are not answering the question, you are voting on it.

**2. Because the difference between `List<Integer>` and `int[]` is 4–5×, and that is
often the whole problem.**

Topic 01 told you boxing allocates. This topic tells you *how much*, once you count the
array of references as well as the boxes. A heap-dump histogram (Topic 79) dominated by
`java.lang.Long` is one of the most common findings in Java production work, and the fix
is a representation change you can only justify with sizes.

**3. Because the 32 GB compressed-oops cliff is a fact you will be the only person in
the room who knows.**

It is genuinely counter-intuitive: **you add memory and lose effective capacity.** Every
reference field in every live object doubles in width the moment the encoding cannot
cover the heap. On a reference-dense object graph — which a Hibernate-backed service
absolutely is — the live set can grow substantially from that alone. This is a
capacity-planning fact, not trivia, and it is Trap 1.

**4. Because "it's just a boolean" is a claim you can now falsify.**

Adding a field to `Product` costs 0 bytes if it lands in an existing padding gap, and 8
bytes if it does not. Multiply by the number of instances live at peak. In a design
review, "we measured: this adds 8 bytes × 100k cached products = 800 KB, which is 0.07%
of heap, ship it" ends the discussion in one sentence. So does the opposite finding.

**5. Because the humongous threshold in Topic 71 is measured against the object's REAL
size.**

A `new byte[1_048_576]` is not 1 MB on the heap. It is 16 bytes of header plus 1 048 576
bytes of payload, which is *over* 1 MB, which means with a 2 MB region size it is over
the half-region humongous threshold when you thought you were exactly at it. Off-by-one
in this arithmetic is the difference between an eden allocation and an old-gen humongous
allocation. Topic 71's entire failure drill turns on this.

---

## Machine-level reality

### The mark word — and a forward reference you should hold

The mark word is 8 bytes of **multiplexed** state. It does not hold one thing; it holds
whichever thing is currently relevant, distinguished by tag bits at the bottom. Over an
object's life it can hold:

| At this moment | The mark word holds |
|---|---|
| Freshly allocated, never hashed, never locked | mostly zeros plus the "unlocked" tag |
| After `System.identityHashCode(x)` or a default `hashCode()` | the **identity hash code**, computed once and stored, never recomputed |
| Surviving a young collection | the object's **age** (a small number of bits — this is Topic 68's tenuring threshold, physically) |
| Locked by a thread | a pointer to a **lock record** on the locking thread's stack, or the "inflated" tag plus a pointer to a heavyweight **monitor** |
| Being moved by the collector | a **forwarding pointer** to the object's new address |

> **Forward reference to Topic 85.** When you write `synchronized (order)`, the lock
> state goes into this word. When contention forces the lock to *inflate*, the JVM
> allocates a native `ObjectMonitor` and stores a pointer to it here. This is why
> "`synchronized` is cheap when uncontended and expensive when contended" is a statement
> about *these eight bytes*, and why Topic 85 is a direct continuation of this section.
> It is also why an object that has had its identity hash code computed cannot use
> certain lock optimisations — the word can only hold one thing at a time. Two features
> competing for the same 64 bits is the kind of mechanism you can only see from here.

There is a diagnostic that dumps the header bits, and JOL prints them for you:

```bash
java -jar jol-cli.jar internals java.lang.Object
```

**WHAT TO LOOK FOR:** the first two rows of the layout table, named for the mark and
class words. On a JDK with compact object headers enabled you will see a single 8-byte
header row instead of 8+4. That single observation settles the version question above.

### The class word and compressed class space

The class word points at the `Klass` structure in **metaspace** — native memory, outside
`-Xmx`, Topic 68's subject. Compressed class pointers store a 32-bit offset into a
dedicated **compressed class space** region, which is why `-XX:CompressedClassSpaceSize`
exists and why it has its own `OutOfMemoryError` message (`Compressed class space`)
distinct from a heap OOM. If you ever see that exact message, the answer is class
loading (Topic 67), not heap.

### The array length word — and why arrays cost 16, not 12

`length` is stored, not derived. It has to be — `arr.length` is a field read, and bounds
checks on every array access read it. So an array header is 12 + 4 = 16 bytes, and:

```
array_size = 16 + (element_width × length), rounded up to a multiple of 8
```

with `element_width` being 4 for reference arrays under compressed oops and 8 without.
**This is the formula behind "an `Object[]` of 1000 elements costs 4 KB before you put
anything in it."**

### Field reordering — what HotSpot actually does

You declare fields in one order. HotSpot lays them out in another. The goals are: keep
every field naturally aligned (an 8-byte `long` should not straddle an 8-byte boundary),
waste as little padding as possible, and keep a superclass's fields at fixed offsets so
subclass layouts can be computed independently.

The observable rules, which you should treat as *strong tendencies to be verified rather
than a specification*:

1. **Superclass fields come before subclass fields.** A subclass cannot renumber its
   parent's offsets, because code compiled against the parent uses those offsets.
2. **Within a class, wider types tend to be allocated first**: `long`/`double`, then
   `int`/`float`, then `short`/`char`, then `byte`/`boolean`, then references. The exact
   policy has changed across JDK versions.
3. **Gaps get filled.** If the header plus a superclass's fields leaves a 4-byte hole
   before the next 8-byte boundary, the algorithm drops an `int` into it rather than pad.
4. **The end is padded** to `ObjectAlignmentInBytes`.

The practical consequence, and the reason this is in the "machine-level reality" section
rather than trivia: **you cannot predict the size of a class with many mixed-width fields
by adding up the widths.** You can predict the *lower bound*. JOL tells you the truth,
including a line that names exactly how many bytes went to internal gaps and how many to
final alignment.

> **Forward reference to Topic 96 (false sharing).** The `@jdk.internal.vm.annotation.Contended`
> annotation deliberately *inserts* padding — typically 128 bytes — around a field so that
> two hot fields cannot share a cache line. That is the same mechanism as alignment
> padding, used on purpose, at a much larger granularity, to solve a hardware contention
> problem. `-XX:-RestrictContended` is what makes it available outside the JDK. When you
> get to Topic 96 and see `LongAdder`'s padded cells, you will be looking at deliberate
> object layout.

### Alignment — and the one flag that moves the cliff

`ObjectAlignmentInBytes` defaults to 8. Every object starts on an 8-byte boundary, so
the low 3 bits of any object address are always zero. Compressed oops exploit exactly
that: store `address >> 3` in 32 bits, recover `value << 3`.

```
addressable_heap = 2^32 × ObjectAlignmentInBytes
                 = 2^32 × 8 = 2^35 bytes = 32 GiB
```

Raise the alignment to 16 and the reach doubles to 64 GiB — at the cost of up to 15 bytes
of padding per object instead of 7. Bad for a heap of small objects, fine for large ones.
**A real lever almost nobody knows exists.** Measure both arms before using it.

```bash
java -XX:ObjectAlignmentInBytes=16 -Xmx40g -XX:+PrintFlagsFinal -version | grep -E "UseCompressedOops|ObjectAlignmentInBytes"
```

**WHAT TO LOOK FOR:** whether `UseCompressedOops` is still `true` at a heap size where
the default alignment would have disabled it.

### The three compressed-oops modes

The JVM does not have one compressed-oops implementation; it picks one of three, and the
choice changes the *cost* of a reference dereference:

| Mode | When | Decoding cost |
|---|---|---|
| **32-bit / unscaled** | heap fits entirely below 4 GB | none — the 32-bit value *is* the address |
| **Zero-based** | heap base can be placed at address 0 | a shift only |
| **With base (shifted + base)** | heap base is nonzero | a shift **and** an add, on every decode |
| **Disabled** | heap too large (~above 32 GB) | none — but references are 8 bytes wide |

Print which one you got:

```bash
java -Xmx<size> -Xlog:gc+heap+coops=info -version
```

**WHAT TO LOOK FOR:** a line naming the heap address range and the compressed-oops mode.

| What you see | What it means |
|---|---|
| A line mentioning zero-based compressed oops | Best case. Cheapest decode, 4-byte references. |
| A line mentioning a nonzero heap base | Still 4-byte references, slightly more expensive decoding. Usually irrelevant; occasionally visible in a very pointer-chasing-heavy microbenchmark. |
| **No such line at all, at a large `-Xmx`** | Compressed oops are **off**. Confirm with `PrintFlagsFinal`. This is the cliff, and you are over it. |
| Compressed oops off at a heap you expected to be safe | The threshold is **not exactly 32 GB** — the JVM must also place the compressed class space and reserve address ranges, so the real cutoff is somewhat *below* 32 GB and varies by platform and JDK. Binary-search it on your own runtime; see Hands-on Proof 5. |

### The 32 GB cliff as a capacity-planning fact

State it the way you would state it in a design review:

> Below roughly 32 GB of heap, every reference costs 4 bytes and the object header costs
> 12. Above it, references cost 8 and the header costs 16. For an object graph with a
> typical density of reference fields, that inflates the live set by a meaningful
> double-digit percentage. **Therefore, in the range from about 31 GB to somewhere in the
> forties, adding heap can reduce the number of objects you can hold.** The correct
> responses are: stay below the threshold; or raise `ObjectAlignmentInBytes` and measure;
> or jump well past the threshold so the extra raw gigabytes outweigh the inflation; or
> run more, smaller JVMs.

The last option is what large deployments actually pick: two 24 GB JVMs hold more objects
than one 48 GB JVM, and each has a smaller live set, which by Topic 70's central rule makes
every pause cheaper. **"Shard the JVM" is a GC decision before it is an availability one.**

`orderflow` at the Topic 65 baseline runs with `-Xmx1200m`, so you are nowhere near this.
That is exactly why you must learn it deliberately: you will not stumble into it, and the
first time it matters will be a Friday.

---

## Example 1 — minimal

Everything here is a **prediction to be verified**, not a measurement. The arithmetic is
mechanical; do it yourself before reading each answer.

### Setup — get JOL

Two ways. Both are worth having.

**As a command-line tool** (no code needed, inspects classes without instantiating them):

```bash
# Fetch the JOL command-line jar (any recent version; check Maven Central for the latest).
# groupId org.openjdk.jol, artifactId jol-cli, classifier full
mvn dependency:get -Dartifact=org.openjdk.jol:jol-cli:LATEST:jar:full \
    -Ddest=jol-cli.jar

java -jar jol-cli.jar internals java.lang.Integer
java -jar jol-cli.jar internals java.lang.String
java -jar jol-cli.jar internals java.util.HashMap
java -jar jol-cli.jar externals java.lang.String
```

**As a test-scope dependency**, which is how you will use it against `orderflow`'s own
classes:

```xml
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version><!-- latest from Maven Central --></version>
  <scope>test</scope>
</dependency>
```

> On JDK 17+ JOL uses `Unsafe` and may need the module opened. If you see an
> `InaccessibleObjectException` or a reflective-access warning, add:
> `--add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED`
> and, if your build complains about `jdk.internal.misc`,
> `--add-exports java.base/jdk.internal.misc=ALL-UNNAMED`. JOL prints a clear diagnostic
> when it cannot introspect; do not ignore it and trust the number anyway.

### The probe

```java
package com.orderflow.lab.layout;

import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.info.GraphLayout;
import org.openjdk.jol.vm.VM;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Prints the layout of the five things you were asked to predict.
 * Run it. Do NOT trust any size written in a document, including this one.
 */
public final class LayoutProbe {

    public static void main(String[] args) {
        // 1. What VM am I actually on? Print this FIRST, every time.
        System.out.println(VM.current().details());

        Integer boxedInt   = 12345;                 // deliberately outside the -128..127 cache
        Long    boxedLong  = 1234567890123L;
        String  sku        = "SKU-0000012345";      // 14 chars, all LATIN1
        Map<String, String> emptyMap = new HashMap<>();

        List<Long> tenLongs = new ArrayList<>();
        for (long i = 1000; i < 1010; i++) {
            tenLongs.add(i);                        // boxing: 10 distinct Long objects
        }

        System.out.println(ClassLayout.parseInstance(boxedInt).toPrintable());
        System.out.println(ClassLayout.parseInstance(boxedLong).toPrintable());
        System.out.println(ClassLayout.parseInstance(sku).toPrintable());
        System.out.println(ClassLayout.parseInstance(emptyMap).toPrintable());
        System.out.println(ClassLayout.parseInstance(tenLongs).toPrintable());

        // Shallow size is one question. Deep size is a different question.
        System.out.println("String  deep: " + GraphLayout.parseInstance(sku).totalSize());
        System.out.println("HashMap deep: " + GraphLayout.parseInstance(emptyMap).totalSize());
        System.out.println("List    deep: " + GraphLayout.parseInstance(tenLongs).totalSize());
        System.out.println(GraphLayout.parseInstance(tenLongs).toFootprint());
    }
}
```

```bash
javac -d out --release 21 -cp jol-core.jar src/test/java/com/orderflow/lab/layout/LayoutProbe.java

java -cp out:jol-core.jar \
     --add-opens java.base/java.lang=ALL-UNNAMED \
     --add-opens java.base/java.util=ALL-UNNAMED \
     com.orderflow.lab.layout.LayoutProbe
```

### The shape of what JOL prints

```
<fully qualified class name> object internals:
OFF  SZ                TYPE DESCRIPTION               VALUE
  0   8                     (object header: mark)     <mark word bits>
  8   4                     (object header: class)    <klass pointer>
 12   4                 int <FieldName>               <value>
Instance size: <N> bytes
Space losses: <A> bytes internal + <B> bytes external = <A+B> bytes total
```

***Illustration of the format, not captured output. `OFF`, `SZ`, `<N>`, `<A>`, `<B>` are
placeholders showing field positions so you can read your own output.***

The four things to read, in this order:

| Column / line | What it tells you |
|---|---|
| `OFF` | Byte offset of the field from the start of the object. **The proof that reordering happened**: compare against your declaration order. |
| `SZ` | Field width in bytes. A reference showing `4` is your compressed-oops confirmation, in the most direct form available. |
| `Instance size` | **The shallow size.** This is what "how big is one of these" means. |
| `Space losses: A internal + B external` | `internal` = padding inserted *between* fields to keep alignment; `external` = padding at the end to reach the object alignment. **`external` is the number that tells you whether one more small field is free.** |

### The five predictions

Do these before running anything. Write your answers down. The rules you need are all
above: 12-byte object header, 16-byte array header, 4-byte references, pad to 8.

**1. `Integer`**

One `int value` field.

```
12 (header) + 4 (int) = 16.  Already a multiple of 8.  → PREDICT 16 bytes.
```

This one you will almost certainly get right, and it is the answer to the interview
staple. Note the shape of the sentence: **"an `Integer` is 16 bytes, which is 4× the 4
bytes of an `int`, and that ratio is the whole cost of `List<Integer>`."**

**2. `Long`**

One `long value` field.

```
12 (header) + 8 (long) = 20.  Pad to 24.  → PREDICT 24 bytes, with 4 bytes of external loss.
```

Note what just happened: a `Long` holds twice the data of an `Integer` and costs only 1.5×
as much, because the `Integer` was paying for padding it did not need. **Object cost is not
linear in data size.** That is a genuinely useful thing to know when choosing a
representation.

**3. `String`**

This is the first place most people go wrong, in **two** independent ways.

`String`'s fields on a modern JDK (verify with `jol-cli internals java.lang.String`):
`byte[] value`, `int hash`, `byte coder`, `boolean hashIsZero`.

```
12 (header) + 4 (value ref) + 4 (int hash) + 1 (coder) + 1 (hashIsZero) = 22.  Pad to 24.
  → PREDICT 24 bytes SHALLOW.
```

**Error mode one:** you predicted from an older mental model of `String` (`char[] value`
plus `int hash` only, or with an extra `offset`/`count` from the pre-Java-7 era) and got a
different field list. The class has changed several times. **Print the fields; do not
recall them.**

**Error mode two, and this is the important one:** 24 bytes is the size of the `String`
*object*. It does not include the `byte[]` holding the characters, which is a **separate
object**. For a 14-character LATIN1 string:

```
byte[] = 16 (array header) + 14 (bytes) = 30.  Pad to 32.
Deep total = 24 + 32 = 56 bytes for a 14-character string.
```

**Compact strings** (Java 9+) are why that is 14 bytes and not 28: a `String` whose
characters all fit in Latin-1 is stored one byte per character, with `coder = LATIN1`.
Introduce a single emoji or a non-Latin-1 character and the whole array switches to UTF-16
at two bytes per character. **A `String` can double in heap cost because of one character
in it.** For `orderflow`, product names and descriptions in a multi-language catalogue is
exactly where that bites.

**4. An empty `HashMap`**

This is where you should expect to be wrong, and it is the most instructive of the five.

`new HashMap<>()` — what is in it?

The trap: **the bucket table is `null`.** `HashMap` allocates its `Node[] table` lazily, on
the first `put`. "Empty `HashMap`" therefore means "a `HashMap` object with a null table",
and its shallow size is just its own fields plus `AbstractMap`'s.

Fields to account for: from `AbstractMap` — `keySet`, `values` (2 references). From
`HashMap` — `table`, `entrySet` (2 references), `size`, `modCount`, `threshold` (3 `int`s),
`loadFactor` (1 `float`).

```
12 (header) + 4×4 (four refs) + 4×3 (three ints) + 4 (float) = 12 + 16 + 12 + 4 = 44.
Pad to 48.  → PREDICT 48 bytes shallow, deep size EQUAL to shallow, because table is null.
```

Now the follow-up that makes the point:

```
After ONE put:
  table = new Node[16]      → 16 (array header) + 16 × 4 (refs) = 80 bytes
  one Node                  → 12 (header) + 4 (int hash) + 4 (key ref) + 4 (value ref) + 4 (next ref) = 28 → pad to 32
  → the map's own footprint jumps by 112 bytes to hold ONE entry
```

**A one-entry `HashMap` costs on the order of 160 bytes before you count the key and the
value objects themselves.** That is the number behind Topic 12's material and behind every
"we cached it in a map" design decision you will ever review. And `Map<Long, Long>` — the
counter idiom Topic 01 warned about — is 32 (Node) + 24 (key `Long`) + 24 (value `Long`) =
**80 bytes of object, plus 4 bytes of table slot, to store 16 bytes of numbers.**

**5. An `ArrayList` of 10 boxed `Long`s**

```
ArrayList object: 12 (header) + 4 (elementData ref) + 4 (int size)
                              + 4 (int modCount, inherited from AbstractList) = 24.  → 24 bytes.

elementData: after ten adds on a default-constructed list, capacity is 10
             (DEFAULT_CAPACITY), so Object[10] = 16 + 10×4 = 56 bytes.

Ten Long objects: 10 × 24 = 240 bytes.

→ PREDICT deep ≈ 24 + 56 + 240 = 320 bytes.
```

Compare against `long[10]`:

```
16 (array header) + 10 × 8 = 96 bytes.
```

**320 versus 96 — a factor of 3.3 for ten elements**, and the ratio *increases* with size
because `ArrayList`'s own 24-byte overhead amortises away while the 24-bytes-per-box does
not. At scale the steady-state ratio is `(4 bytes of slot + 24 bytes of box) / 8 bytes of
primitive = 3.5×` for `Long`, and `(4 + 16) / 4 = 5×` for `Integer`. **That is the "4–5×"
figure from Topic 01, now derived rather than asserted.**

### Two ways this prediction can be wrong, and both are worth triggering

**(a) `ArrayList` growth.** If you built the list a different way — `new ArrayList<>(20)`,
or 11 adds instead of 10 — the backing array is a different size and your prediction is
off by tens of bytes with no visible cause. `ArrayList` grows by roughly 1.5× and never
shrinks. **`GraphLayout` will show you the real array length; your arithmetic will not.**

**(b) The `Integer`/`Long` cache.** `Long.valueOf(5)` returns a shared, cached instance for
values in −128..127 (Topic 01). If you had built the list with small values, `GraphLayout`
counts each *distinct* object once — so ten cached boxes for values 1..10 are ten distinct
cached objects and still count ten times, but adding **the same** `Long` ten times counts
it once and your deep size collapses. **`GraphLayout` measures a graph, not a multiset.**
This is the single most common way people misread a deep-size number.

Run `toFootprint()` to see this directly:

```
<class name> instance footprint:
COUNT  AVG  SUM   DESCRIPTION
    1  <n>  <n>   java.util.ArrayList
    1  <n>  <n>   [Ljava.lang.Object;
   10  <n>  <n>   java.lang.Long
                  (total)
```

***Illustration of the format, not captured output. `<n>` are placeholders.***

**WHAT TO LOOK FOR:** the `COUNT` column. If it says 10 `Long`s, you have ten distinct
objects. If it says 1, you added the same box repeatedly and every deep-size conclusion you
were about to draw is wrong.

### Reading your actual output

| What you see | What it means |
|---|---|
| `Integer` instance size 16, `Long` 24 | Standard 64-bit HotSpot with compressed oops. Your baseline mental model is confirmed. |
| `Integer` instance size **12**, `Long` **16** | **Compact object headers are ON.** The header is 8 bytes, not 12. Every prediction above is 4 bytes high for non-arrays. Re-derive, and note the flag in your capacity model. |
| `Integer` instance size **16** but the class word shows `SZ 8` | Compressed **class** pointers are off while compressed oops may still be on. Unusual; check both flags. |
| `Integer` instance size **24** | Compressed oops off entirely — header is 16. Check `-Xmx`; you are over the cliff, or someone passed `-XX:-UseCompressedOops`. |
| `String` has fields you did not predict | Expected. The class evolves. This is why the drill is measurement-shaped. |
| Empty `HashMap` deep size == shallow size | Correct, and it proves the lazy table. |
| Empty `HashMap` deep size **larger** than shallow | You did not create it with `new HashMap<>()`. `Map.of()`, `new HashMap<>(map)`, or a `put` you forgot about. |
| `Space losses: 0 internal + 4 external` on `Long` | The 4 bytes of tail padding. Confirms the alignment rule, and tells you a 4-byte field could be added for free. |
| A large `internal` loss on one of `orderflow`'s entities | Field widths that do not pack. Usually many `boolean`s or a `long` after an odd number of `int`s. Worth a look if the class has 100k live instances; irrelevant otherwise. |

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Collector | G1 (verify it, do not assume it) |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Arrival rate | open model, roughly 400 requests per second total |
| Latency budget | endpoint p99s within the recorded baseline |
| Baseline artefacts | `/docs/java/baselines/` |

### The proposal

Catalogue reads are 70% of traffic, and the product table is nearly static. Someone
proposes the obvious thing:

> "Let's cache the whole product catalogue in-process. 100k products, it's just SKUs and
> prices, it'll be a few megabytes. We can drop the Redis round-trip on the hot path
> entirely."

This is a good instinct and might be a good decision. It is also a **capacity claim**, and
the entire point of this topic is that you can now check it before it becomes a 3am page.

### The class being cached

```java
package com.orderflow.catalog;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.Map;

public final class ProductSummary {
    private final long id;                     // 8
    private final String sku;                  // ref → String → byte[]
    private final String name;                 // ref → String → byte[]
    private final BigDecimal price;            // ref → BigDecimal → BigInteger?
    private final String currency;             // ref
    private final long categoryId;             // 8
    private final int  stockOnHand;            // 4
    private final boolean active;              // 1
    private final boolean discontinued;        // 1
    private final Instant updatedAt;           // ref → Instant
    private final List<String> tags;           // ref → ArrayList → Object[] → Strings
    private final Map<String, String> attributes; // ref → HashMap → Node[] → Nodes → Strings
    // ... constructor and accessors omitted
}
```

### The arithmetic you do in the design review, out loud

**Step 1 — shallow size of `ProductSummary`.**

```
12 (header)
+ 8 + 8   (two longs)
+ 4       (int stockOnHand)
+ 1 + 1   (two booleans)
+ 4 × 7   (seven references: sku, name, price, currency, updatedAt, tags, attributes)
= 12 + 16 + 4 + 2 + 28 = 62  → pad to 64
```

**PREDICTION: 64 bytes shallow.** That is the number people quote, and it is off by more
than an order of magnitude from the truth, because **seven of the twelve fields are
references to objects that are not counted.**

**Step 2 — the graph, which is the number that actually matters.**

Estimate each referenced subtree, using the rules from Example 1:

| Field | Objects reached | Rough cost |
|---|---|---|
| `sku` | `String` (24) + `byte[]` for ~14 LATIN1 chars (32) | ~56 |
| `name` | `String` (24) + `byte[]` for ~40 chars; **UTF-16 if any non-Latin-1 character** (56 LATIN1 / 96 UTF-16) | 80–120 |
| `price` | `BigDecimal` — an object with an `int scale`, a `long intCompact`, a possibly-null `BigInteger`, and a cached `String` — and a `BigInteger` is itself an object plus an `int[]` | 40–120, **measure it, do not guess** |
| `currency` | shared `String` if interned/constant-pooled; a distinct one per product if built per row | 0 or ~48 |
| `updatedAt` | `Instant`: header + `long seconds` + `int nanos` | ~24 |
| `tags` | `ArrayList` (24) + `Object[]` (16 + 4n) + n `String`s (~48 each) | ~90 for n=1, ~240 for n=4 |
| `attributes` | `HashMap` (48) + `Node[16]` (80) + n `Node`s (32 each) + 2n `String`s | ~300 for n=4 |

**A plausible total is several hundred bytes per product, not 64.** I am deliberately not
committing to a single figure, because the answer depends entirely on your actual data —
tag counts, attribute counts, name lengths, whether names are Latin-1, and whether
`currency` strings are shared. **That is the finding.** The honest output of this step is a
range and a plan to measure.

**Step 3 — measure it, on real data.**

```java
package com.orderflow.catalog;

import org.openjdk.jol.info.GraphLayout;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * Runs under the 'sizing' profile ONLY. Never on a production path.
 * Loads a representative sample from the real catalogue and measures the graph.
 */
@Component
@Profile("sizing")
public class CatalogSizingProbe implements CommandLineRunner {

    private final ProductQueryService products;

    public CatalogSizingProbe(ProductQueryService products) {
        this.products = products;
    }

    @Override
    public void run(String... args) {
        // A RANDOM sample, not the first N. The first N are sorted by id and
        // systematically unrepresentative of tag/attribute cardinality.
        List<ProductSummary> sample = products.randomSample(1000);

        // Measure the sample AS A WHOLE, so shared Strings are counted once -
        // which is what a real cache would also do.
        long totalBytes = GraphLayout.parseInstance(sample.toArray()).totalSize();

        System.out.printf("sample=%d bytes=%d avg=%.1f%n",
                sample.size(), totalBytes, totalBytes / (double) sample.size());
        System.out.println(GraphLayout.parseInstance(sample.toArray()).toFootprint());
    }
}
```

```bash
docker compose run --rm \
  -e SPRING_PROFILES_ACTIVE=sizing \
  -e JAVA_TOOL_OPTIONS="--add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED" \
  orderflow
```

**WHAT TO LOOK FOR:** the `toFootprint()` breakdown, class by class.

| What you see | What it means | What you do |
|---|---|---|
| `java.lang.String` and `[B` dominate the footprint | Normal for a catalogue. Text is the payload. | Consider whether `name`/`description` need to be in the cache at all, or whether the cache should hold ids and the text should stay in Redis. |
| `java.math.BigDecimal` is a surprisingly large share | Each price is an object graph, not a number. | Store minor units as a `long` and a currency code; construct `BigDecimal` only at the API boundary. This is usually a 3–4× reduction on that field. |
| `java.util.HashMap$Node` count ≫ product count | The `attributes` map is the dominant cost, and per-entry overhead is the reason. | A sorted `String[]` pair array, or a shared interned key set, or drop attributes from the cache. |
| Average per product is close to your shallow estimate | You are measuring a shared graph — most `String`s are the same objects. **Check the `COUNT` column.** | Verify with a second, disjoint sample. If the average jumps, you had accidental sharing. |
| Average per product is 10× your shallow estimate | Expected, and it is the whole lesson. | Multiply out and compare against heap. |
| The probe itself OOMs or takes minutes | `GraphLayout` walked into something enormous. See Trap 5. | Detach the DTOs from the persistence context first; never `parseInstance` a Hibernate entity. |

**Step 4 — the decision, stated as arithmetic.**

```
per_product_bytes  = <measured average from Step 3>
cache_bytes        = per_product_bytes × 100_000
heap               = 1200 MB
live_set_baseline  = <the floor of old-gen occupancy after mixed/full GCs at baseline load, Topic 68>
headroom_required  = enough that G1 never hits evacuation failure (Topic 71)

VERDICT: cache_bytes + live_set_baseline + headroom_required  <  heap ?
```

If `per_product_bytes` lands in the low hundreds, 100k products is somewhere in the tens of
megabytes, and against a 1200 MB heap the cache is affordable — **but you must then re-run
the Topic 65 baseline**, because you have just permanently increased the live set, and by
Topic 70's central rule *every young collection now costs more, forever*. The cache is not
free even when it fits.

If it lands near a kilobyte, 100k products is ~100 MB — around 8% of heap for a cache, plus
the GC cost of a permanently larger live set. That is a real trade, and now it is a
conversation with numbers.

### The change that would have caused an incident

The version of this story where nobody does the arithmetic:

1. Cache ships. Staging has 5k products seeded, works beautifully. p99 improves.
2. Production has 100k products, and production product names are multi-language, so a
   large fraction are UTF-16 — **double the `byte[]` cost of the biggest field**.
3. The cache is a `ConcurrentHashMap` field on a singleton bean, so it is reachable from a
   GC root forever (Topic 70) and every entry is promoted to old gen (Topic 68).
4. Old-gen occupancy floor rises. G1's concurrent cycles start earlier and more often.
   Mixed collections have more to copy. p99 degrades — **on every endpoint, including the
   ones that do not touch the cache**, because pauses are global.
5. Under peak load, G1 runs out of free regions to evacuate into and does a `Pause Full`.
   Multi-second stall. Page.
6. The postmortem says "GC issue" and someone proposes tuning `MaxGCPauseMillis`, which by
   Topic 71's Trap 1 makes it worse.

**The root cause is in this document:** nobody knew the per-entry size, so nobody knew the
live set was about to grow by 100 MB, so nobody re-ran the baseline. Object layout is not
an academic topic. It is the input to the capacity decision that prevents that entire chain.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — adding memory past ~32 GB and LOSING effective capacity

**Wrong approach.** A reporting service (`orderflow`'s analytics sibling, Topic 25's
parallel-stream workload) is running at `-Xmx30g` and occasionally hitting long GC pauses
under month-end load. The team has the budget, so they move to a larger instance and set
`-Xmx48g`. Reasonable, obvious, and wrong.

**Exact symptom.**

- Old-generation occupancy after a full collection — the live-set floor from Topic 68 —
  rises by a large fraction, **with no data change and no code change**. The same query
  over the same rows now retains meaningfully more heap.
- Young collections take longer: Object Copy in the `gc+phases=debug` breakdown is up,
  because every surviving object is bigger and every reference field being scanned is
  wider.
- `jcmd <pid> GC.class_histogram` shows the same *instance counts* as before with larger
  *byte totals* per class. That is the fingerprint. **Same objects, more bytes.**
- The out-of-memory condition they were trying to escape is not as far away as the
  arithmetic promised, and on the worst month-end runs it is not further away at all.
- Nothing in the GC log says "compressed oops disabled". There is no warning. The JVM
  makes the change silently.

**Root cause.** At `-Xmx48g` the heap cannot be addressed by a 32-bit shifted oop, so the
JVM disabled compressed oops. Every reference field went from 4 bytes to 8. Every object's
class word went from 4 bytes to 8, taking the header from 12 to 16. On a reference-dense
graph — Hibernate entities, `HashMap` nodes, `ArrayList` backing arrays, `String` objects
pointing at `byte[]`s — a large share of the live set *is* references and headers. **They
bought 18 GB of raw heap and spent a substantial part of it on pointer width.**

**Fix.**

1. **Confirm it first.** Never fix on a hypothesis:
   ```bash
   jcmd <pid> VM.flags -all | grep -i CompressedOops
   java -Xmx48g -XX:+PrintFlagsFinal -version | grep -i CompressedOops
   java -Xmx48g -Xlog:gc+heap+coops=info -version
   ```
2. **Find your own threshold** — it is not exactly 32 GB, because the JVM must also reserve
   the compressed class space and place the heap base. Binary-search it (Hands-on Proof 5).
3. **Then choose, deliberately:**

| Option | When it is right | Cost |
|---|---|---|
| Stay at or below your measured threshold (e.g. `-Xmx30g`) | The live set fits, and it usually does | You leave RAM on the table — but it was never free RAM |
| `-XX:ObjectAlignmentInBytes=16` with `-Xmx48g` | Objects are large on average, so extra tail padding is a small fraction | Up to 15 bytes padding per object instead of 7; **measure the live set both ways before committing** |
| Go far past the threshold — 64 GB, 96 GB | The raw gigabytes genuinely outweigh the inflation | Larger live set per collector instance, so longer pauses (Topic 70); often the wrong answer for a latency-sensitive service |
| **Run two or three smaller JVMs** | Almost always, for a service | Operational complexity — and smaller live sets per JVM, which makes every pause cheaper |

**What the fix proves:** heap size is not a scalar you can raise monotonically. There is a
discontinuity in it, and you are expected to know where.

---

### Trap 2 — assuming `List<Integer>` costs the same as `int[]`

**Wrong approach.** `orderflow`'s recommendation path needs, per request, the set of
product ids a user has recently viewed, and per background job, an in-memory index of
`productId → viewCount`. It gets written the natural way:

```java
// The natural, idiomatic, and expensive version.
private final Map<Long, Long> viewCounts = new ConcurrentHashMap<>();
private final List<Integer>   recentIds  = new ArrayList<>();

public void recordView(long productId) {
    viewCounts.merge(productId, 1L, Long::sum);   // allocates a boxed Long per increment
}
```

**Exact symptom.**

- A heap dump (Topic 79) opened in MAT shows `java.lang.Long` and `java.lang.Integer` in
  the top three classes by *retained* size — often above the domain entities the service
  exists to serve.
- `jcmd <pid> GC.class_histogram` shows instance counts in the millions for `Long`.
- Allocation rate (Topic 68's formula) is high and dominated by boxing, visible directly in
  an async-profiler allocation profile (Topic 78) as `java.lang.Long.valueOf`.
- The heap the team sized from "we're storing 5 million longs, that's 40 MB" is exhausted
  at a fraction of the expected element count.

**Root cause.** Three costs stack, and only the first is obvious:

```
long primitive in a long[]      : 8 bytes
Long object                     : 24 bytes  (12 header + 8 long + 4 padding)
+ the reference to it           : 4 bytes   (an Object[] slot, or a HashMap.Node field)
+ HashMap.Node wrapper          : 32 bytes  (12 header + hash + key ref + value ref + next ref, padded)
```

So a `Map<Long, Long>` entry is roughly **80 bytes of object plus a 4-byte table slot for
16 bytes of numbers — around 5×**. A `List<Integer>` is `(4 slot + 16 box) / 4 primitive =
5×`. A `List<Long>` is `(4 + 24) / 8 = 3.5×`. **That is the "4–5×" number, and now you can
derive it for any element type instead of repeating it.**

And the `merge` above allocates a **new** `Long` on every increment past the cache range,
so this is an allocation-rate problem as well as a live-set problem.

**Fix.** In increasing order of intrusiveness:

| Fix | Use when |
|---|---|
| `long[]` / `int[]` directly | You control the index space and the size is known |
| An open-addressed primitive map — **fastutil**, **Eclipse Collections**, **HPPC** (`Long2LongOpenHashMap`) | You need map semantics on primitive keys. This is the standard answer and typically a 3–5× live-set reduction with no `Node` objects at all |
| `LongAdder` per key instead of boxed counters | High-contention counters; Topic 95 |
| Don't hold it on-heap at all | Redis (Topic 110), or off-heap (Topic 80) |
| Nothing | **Fewer than a few thousand entries.** A 5× overhead on 2 KB is 10 KB. Do not rewrite anything for that. |

**The senior move is the last row.** The cost is real, the ratio is real, and it is
**irrelevant below a threshold you should state out loud**. "This is 5× the primitive cost,
which at our cardinality is 40 KB, so we keep the readable version" is a stronger review
comment than either "boxing is fine" or "never box".

---

### Trap 3 — "it's just one more field", or: padding is invisible in source

**Wrong approach.** A ticket asks for a `boolean priceOverridden` flag on `ProductSummary`,
the class cached 100k times. Someone adds it. Someone else, in a different sprint, adds
`boolean taxExempt`. A third adds `long supplierId`. Each change is reviewed on its own and
each looks free.

**Exact symptom.** Confusing and inconsistent:

- The first `boolean` changes the cache's measured footprint by **zero bytes**.
- The second `boolean` also changes it by zero.
- The `long` adds **8 bytes per instance** — 800 KB across the cache.
- A fourth change, adding one more `int`, adds **8 bytes**, not 4.
- The team concludes fields are free, then concludes fields are expensive, then stops
  reasoning about it at all, which is the actual damage.

**Root cause.** Alignment. The class had 2 bytes of tail padding, so two `boolean`s landed
in existing slack and cost nothing. The `long` needed 8-byte alignment and a fresh 8-byte
slot. The `int` pushed past a boundary and dragged 4 bytes of new padding with it. **The
per-field cost is a function of what is already in the class, and it is not visible in
source.**

**Fix.** Make it visible, cheaply and automatically.

```bash
# Before and after, on the class you actually care about.
java -cp out:jol-core.jar -jar jol-cli.jar internals com.orderflow.catalog.ProductSummary
```

**WHAT TO LOOK FOR:** the `Space losses: A internal + B external` line.

| What you see | What it means |
|---|---|
| `external` loss of 4 or more | There is slack at the end. A field of that width or narrower is genuinely free. Say so in the review. |
| `external` loss of 0 | The next field of any width costs at least 8. Say that in the review too. |
| Large `internal` loss | Fields are not packing. Usually a `long` or a reference sandwiched between odd-width fields. Rarely worth chasing — **unless** instance counts are in the millions. |
| Instance size crossed a multiple of 8 after your change | You added 8 bytes, not 1. Multiply by live instance count before you argue about it. |

And then **encode the decision as a test**, so it cannot regress silently:

```java
package com.orderflow.catalog;

import org.junit.jupiter.api.Test;
import org.openjdk.jol.info.ClassLayout;

import static org.assertj.core.api.Assertions.assertThat;

class ProductSummaryLayoutTest {

    /**
     * This class is cached 100k times. Its shallow size is a capacity input.
     * If this test fails, someone added a field: re-run the sizing probe and
     * update the capacity model in /docs/java/baselines/ before changing this number.
     *
     * NOTE: this number is JDK- and flag-dependent (compressed oops, compact object
     * headers). It is pinned to the runtime declared in our Dockerfile, on purpose.
     */
    @Test
    void shallowSizeIsPinned() {
        long size = ClassLayout.parseClass(ProductSummary.class).instanceSize();
        assertThat(size)
            .as("ProductSummary shallow size - see docs/java/baselines/catalog-cache-sizing.md")
            .isEqualTo(/* the value YOU measured on YOUR runtime */ 0L);
    }
}
```

> That test is deliberately written to fail until you fill in your own measured number.
> **Do not copy a size from a document into an assertion.** The assertion's job is to
> notice change, and it can only do that if the baseline came from your machine.

---

### Trap 4 — reporting a `String`'s shallow size as its memory cost

**Wrong approach.** A capacity model for the catalogue cache is built by summing
`ClassLayout.parseInstance(...).instanceSize()` over the fields of `ProductSummary`. It
comes out at 64 bytes per product, so 100k products is "about 6.4 MB, negligible".

**Exact symptom.** The cache is enabled and the measured heap growth is an order of
magnitude larger than the model predicted. Nobody can explain the gap, so the model is
abandoned rather than fixed — and with it, the habit of modelling at all.

**Root cause.** Shallow size stops at the object boundary. `ProductSummary` holds seven
references; the model counted 4 bytes for each reference and 0 bytes for each *target*. The
targets are `String`s, each of which is *itself* two objects (the `String` and its `byte[]`),
plus a `BigDecimal` which may be two or three, plus collections which are three or more. The
model measured the *tip* of the graph.

There is a second, subtler version of the same mistake in the other direction: summing deep
sizes per object and adding them up. That **double-counts every shared object** — the
interned currency string, the shared empty array, the cached small `Long`. Shared structure
is exactly what makes a real cache cheaper than the sum of its parts.

**Fix.** Know which of the three questions you are asking, and use the matching tool.

| Question | Tool | Gotcha |
|---|---|---|
| "How big is one instance of this class?" | `ClassLayout.parseInstance(x).instanceSize()` — or `parseClass(C.class)` without an instance | Excludes everything it points at |
| "How big is this object graph, counting each object once?" | `GraphLayout.parseInstance(x).totalSize()` | Follows **all** references — see Trap 5 |
| "How big is this collection of objects, with sharing counted once?" | `GraphLayout.parseInstance(a, b, c...)` or `parseInstance(list.toArray())` on the **whole sample at once** | This is the one that models a cache correctly |
| "If I dropped this, how much would come back?" | **Not JOL.** MAT's *retained size* on a heap dump, Topic 79 | Retained ≠ deep: retained excludes objects still reachable elsewhere |

**The sentence to remember:** *deep size is "everything I can reach"; retained size is
"everything that would die with me". For a cache-sizing decision you want retained, and
JOL's deep size is an upper bound on it.*

---

### Trap 5 — `GraphLayout` on a live entity walks into the whole application

**Wrong approach.** Wanting to know how big an `Order` is, someone writes the obvious probe
inside a `@Transactional` service method:

```java
// DO NOT DO THIS on a managed entity inside a transaction.
Order order = orderRepository.findById(id).orElseThrow();
System.out.println(GraphLayout.parseInstance(order).totalSize());
```

**Exact symptom.** One of these three, and which one you get is luck:

- The call takes minutes and prints a number in the **hundreds of megabytes**, which is
  obviously not the size of one order.
- The process throws `OutOfMemoryError` — from the *measuring* tool, while the application
  itself is healthy.
- It triggers a storm of lazy-loading SELECTs, or a `LazyInitializationException` (Topic
  49), because walking the object graph touches Hibernate proxies and each touch is a query.

**Root cause.** `GraphLayout` follows every reference transitively, and a managed entity is
not an island. `Order` → persistence context → session → `EntityManagerFactory` → session
factory → connection pool → configuration → class loaders. **In a live Spring application
almost every object is transitively connected to almost every other object.** The
"footprint of an `Order`" is not a question the reference graph can answer.

This is also the honest reason the Trap 4 table sends you to MAT: retained size on a heap
dump *is* the question "what would die with this", and a live-graph walk cannot compute it.

**Fix.**

1. **Measure detached, plain data.** Map to a DTO or record first, outside the transaction:
   ```java
   OrderSummary dto = orderMapper.toSummary(order);   // plain fields, no proxies
   long bytes = GraphLayout.parseInstance(dto).totalSize();
   ```
2. **Measure a batch of DTOs at once**, so shared strings are counted once, exactly as
   `CatalogSizingProbe` above does.
3. **For anything involving the persistence context, use a heap dump and MAT** (Topic 79).
   Dominator tree, retained size, path to GC roots. That is the right tool and JOL is not.
4. **Never run `GraphLayout` on a production request path.** It is a diagnostic, it walks
   arbitrary amounts of graph, and it holds references while it does so.

**What the fix proves:** the reference graph in a running application is one connected blob.
Any tool that answers "how big is this" by walking outward must be told where to stop, and
JOL cannot be. That is a property of the graph, not a bug in JOL.

---

## Hands-on proof

Every one of these is under two minutes. Do all six before the failure drill.

### Setup

```bash
mkdir -p ~/jvm-lab && cd ~/jvm-lab
mvn dependency:get -Dartifact=org.openjdk.jol:jol-cli:LATEST:jar:full -Ddest=jol-cli.jar
mvn dependency:get -Dartifact=org.openjdk.jol:jol-core:LATEST -Ddest=jol-core.jar
java -version
```

### Proof 1 — establish what your VM actually chose

```bash
java -XX:+PrintFlagsFinal -version | grep -E "UseCompressedOops|UseCompressedClassPointers|ObjectAlignmentInBytes|CompactObjectHeaders"
```

**WHAT TO LOOK FOR:** four values, written down before you predict anything.

| What you see | What it means |
|---|---|
| `UseCompressedOops = true`, `ObjectAlignmentInBytes = 8` | The standard configuration all the arithmetic in this document assumes. |
| `UseCompactObjectHeaders = true` | **Headers are 8 bytes, not 12.** Every non-array prediction here is 4 bytes high. Re-derive before the drill; this is the point of the drill. |
| `UseCompactObjectHeaders` absent from the output | Your JDK does not have the flag. You are on the 12-byte header. Fine. |
| `UseCompressedOops = false` at a small `-Xmx` | Somebody's `JAVA_TOOL_OPTIONS` or a base image is setting it. Find out why before doing anything else. |
| `ObjectAlignmentInBytes = 16` | Someone has already made the Trap 1 trade. Ask who and why; there should be a measurement. |

### Proof 2 — the header, in isolation

```bash
java -jar jol-cli.jar internals java.lang.Object
```

**WHAT TO LOOK FOR:** an object with **no fields at all**, so its entire size is header
plus padding. This is the cleanest possible measurement of your header size.

| What you see | What it means |
|---|---|
| Instance size 16 | 12-byte header + 4 bytes of padding. The classic configuration. |
| Instance size 8 | 8-byte compact header, no padding needed. Compact object headers are on. |
| Instance size 16 with an 8-byte class word | Compressed class pointers off; header is 16, no padding. |

### Proof 3 — prove field reordering happens

```java
package com.orderflow.lab.layout;

import org.openjdk.jol.info.ClassLayout;

public class ReorderProbe {
    // Declared deliberately in the worst possible order for packing.
    static class Awkward {
        boolean a;
        long    b;
        boolean c;
        int     d;
        boolean e;
        Object  f;
    }

    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(Awkward.class).toPrintable());
    }
}
```

**WHAT TO LOOK FOR:** the `OFF` column versus your declaration order.

| What you see | What it means |
|---|---|
| `long b` at a low offset, `boolean`s grouped at the end | Reordering confirmed. The JVM packed wide fields first and filled the tail with the narrow ones. |
| Fields in exactly declaration order | Either your declaration order already packed perfectly, or your JDK's layout policy differs. Try a more awkward order before concluding anything. |
| `Space losses` mostly `internal` | Something forced a gap — commonly a superclass field boundary. |
| Total size smaller than the naive sum of widths | You never had to pay for the padding you assumed. This is the point. |

### Proof 4 — the compressed-oops reference width, seen directly

```bash
# Any class with a reference field will do; String has one.
java -jar jol-cli.jar internals java.lang.String
java -XX:-UseCompressedOops -jar jol-cli.jar internals java.lang.String
```

**WHAT TO LOOK FOR:** the `SZ` column on the `byte[] value` row, and the total instance
size, in both runs.

| What you see | What it means |
|---|---|
| `SZ 4` on the reference, larger `SZ 8` in the second run | Direct, unambiguous proof of what compressed oops do. **This is the two-command version of the whole topic.** |
| Instance size grows in the second run | Header grew too (class word 4 → 8), plus the wider reference, plus re-padding. |
| The second command errors or warns that the flag is obsolete | Some JDKs restrict or deprecate manual control. Then use the `-Xmx`-based method in Proof 5, which is the real-world trigger anyway. |

### Proof 5 — find YOUR compressed-oops threshold

Do not take 32 GB on faith. It is an upper bound, not the answer.

```bash
for size in 28g 29g 30g 31g 32g 33g 34g; do
  printf "%s: " "$size"
  java -Xmx$size -XX:+PrintFlagsFinal -version 2>/dev/null \
    | grep -E "^ *bool UseCompressedOops" \
    | awk '{print $4}'
done
```

> This reserves virtual address space, not physical memory, so it usually runs on a laptop.
> If a size fails to start at all, that is a different limit — record it and move on.

**WHAT TO LOOK FOR:** the exact `-Xmx` at which the value flips from `true` to `false`.

| What you see | What it means |
|---|---|
| Flip somewhere below 32 GB | Expected. The JVM must also reserve the compressed class space and place the heap base. **Write your number down; it is your real ceiling.** |
| Flip exactly at 32 GB | Also plausible depending on JDK and platform. |
| `true` all the way to 34 GB | Check `ObjectAlignmentInBytes` — someone raised it, doubling the reach. |
| Different answers on your laptop and in the container | **Then the container's is the one that counts.** Run this inside the image you deploy, with the same base image and the same `JAVA_TOOL_OPTIONS`. |

### Proof 6 — see the cost of the cliff on a real structure

```java
package com.orderflow.lab.layout;

import org.openjdk.jol.info.GraphLayout;
import java.util.HashMap;
import java.util.Map;

public class CliffProbe {
    public static void main(String[] args) {
        Map<String, String> m = new HashMap<>();
        for (int i = 0; i < 100_000; i++) {
            m.put("SKU-" + i, "warehouse-" + (i % 50));
        }
        System.out.println("deep bytes = " + GraphLayout.parseInstance(m).totalSize());
        System.out.println(GraphLayout.parseInstance(m).toFootprint());
    }
}
```

```bash
# Arm A: comfortably under your measured threshold.
java -Xmx4g -cp out:jol-core.jar com.orderflow.lab.layout.CliffProbe

# Arm B: comfortably over it. Use a size that Proof 5 showed disables compressed oops.
java -Xmx<over-your-threshold> -cp out:jol-core.jar com.orderflow.lab.layout.CliffProbe
```

**WHAT TO LOOK FOR:** the ratio between the two `deep bytes` figures, and the per-class
`AVG` column in the footprints.

| What you see | What it means |
|---|---|
| Arm B is meaningfully larger, same `COUNT`s | **The cliff, measured on your own machine, on a realistic structure.** Same objects, more bytes. Record the ratio; it is the input to Trap 1's decision table. |
| The ratio is small | Your structure is payload-dominated (long strings) rather than reference-dominated. That is itself a finding: the cliff hurts pointer-heavy graphs most. |
| Arm B refuses to start | The machine does not have the address space. Run it in a container with a larger limit, or accept Proof 5's flag output as sufficient evidence. |
| Identical sizes | Compressed oops did not actually turn off. Re-check with `PrintFlagsFinal` at that exact `-Xmx`. |

---

## Failure drill

**Mandatory.** This drill is measurement-shaped: **its success condition is that you are
wrong at least once.** If every prediction matched, you either got lucky or you did not
predict — go back and add three more classes until something surprises you.

Do not read past Step 3 until you have written your predictions down. Predicting *after*
seeing the answer feels identical to predicting before, and teaches nothing. This is the
single most-skipped instruction in this document and the one that matters most.

### The assignment, restated from the master plan

> Predict the size of `Integer`, `Long`, `String`, an empty `HashMap`, and an `ArrayList`
> of 10 boxed `Long`s. Then verify with JOL. **Be wrong at least once.**

### Step 0 — establish the ground rules of your own runtime

```bash
java -version
java -XX:+PrintFlagsFinal -version | grep -E "UseCompressedOops|UseCompressedClassPointers|ObjectAlignmentInBytes|CompactObjectHeaders"
java -jar jol-cli.jar internals java.lang.Object
```

Write down four numbers, on paper, before predicting anything:

```
object header bytes  = ____   (from java.lang.Object's instance size minus its padding)
array header bytes   = ____   (object header + 4)
reference bytes      = ____   (4 if compressed oops on, else 8)
alignment bytes      = ____   (ObjectAlignmentInBytes)
```

**If your header is 8 rather than 12, every prediction in Example 1 is 4 bytes high.** Do
not "correct for it" by mentally subtracting — re-derive each one. The re-derivation is
where the learning is.

### Step 1 — predict, in writing, with your working shown

Fill this in on paper or in a scratch file. **Show the arithmetic, not just the answer** —
when you are wrong, you need to be able to see *which term* was wrong.

| # | Thing | Fields you believe it has | Your arithmetic | Predicted shallow | Predicted deep |
|---|---|---|---|---|---|
| 1 | `Integer.valueOf(12345)` | | | | |
| 2 | `Long.valueOf(1234567890123L)` | | | | |
| 3 | `"SKU-0000012345"` (14 chars) | | | | |
| 4 | `new HashMap<>()` | | | | |
| 5 | `new HashMap<>()` **after one `put`** | | | | |
| 6 | `ArrayList` of 10 distinct boxed `Long`s | | | | |
| 7 | `long[10]` | | | | |
| 8 | `com.orderflow.catalog.ProductSummary` | | | | |

Rows 5, 7 and 8 are not in the assignment. Add them anyway: 5 is where the lazy table
surprises you, 7 is the comparison that makes rows 6 and 7 mean something, and 8 is the one
that transfers to your job.

**Commit to a number for every cell.** "About 20-something" is not a prediction and cannot
be wrong, which means it cannot teach you anything.

### Step 2 — predict two harder ones

These exist to guarantee the "be wrong" condition, because almost nobody gets both:

| # | Thing | Predicted deep |
|---|---|---|
| 9 | `"café"` — four characters, one non-ASCII | |
| 10 | `new ArrayList<>()` with 11 `Long`s added one at a time | |

Row 9 is the compact-strings trap: is `é` inside Latin-1 or not, and what does that do to
the `byte[]`? Row 10 is the `ArrayList` growth trap: the eleventh add triggers a grow, and
the new capacity is not 11.

### Step 3 — verify

Run `LayoutProbe` from Example 1, extended with rows 5, 7, 8, 9 and 10.

```bash
java -cp out:jol-core.jar \
     --add-opens java.base/java.lang=ALL-UNNAMED \
     --add-opens java.base/java.util=ALL-UNNAMED \
     com.orderflow.lab.layout.LayoutProbe
```

For row 8, against the real class:

```bash
java -jar jol-cli.jar internals com.orderflow.catalog.ProductSummary
# (put orderflow's jar on the classpath: -cp target/orderflow.jar)
```

### Step 4 — what to capture

For every row where you were wrong, capture **all four** of these. The last one is the one
that makes the drill worth doing.

1. The predicted number.
2. The measured number.
3. The **specific term in your arithmetic** that was wrong — not "I was off by 8", but
   "I assumed `String` had a `char[]`, and it has a `byte[]` plus a `coder` byte plus a
   `hashIsZero` boolean".
4. **The general rule you can now state** that would have prevented it.

### Step 5 — how to read your errors

| Where you were wrong | The rule you just learned |
|---|---|
| `Integer` or `Long` off by exactly 4 | Compact object headers, or you forgot tail padding. Check Step 0's header number. |
| `String` field list wrong | **JDK library classes change.** `String` lost `offset`/`count`, then gained `coder` and `hashIsZero`. Never predict a library class's fields from memory; print them. |
| `String` deep size far too small | You forgot the `byte[]` is a separate object with its own 16-byte header. **The most common single error in this drill.** |
| Empty `HashMap` deep > shallow predicted | You assumed the table is allocated in the constructor. It is not — `HashMap` allocates lazily on first `put`, and that is a deliberate design choice for the many maps that are created and never filled. |
| `HashMap` after one `put` badly underestimated | You counted the `Node` and forgot the 16-slot table, or vice versa. One entry costs a `Node` **plus** the whole table. |
| `ArrayList` of 10 off by tens of bytes | Backing-array capacity is not element count. Default capacity is 10; growth is ~1.5×; **capacity never shrinks**. |
| Row 10 (11 elements) badly wrong | The grow happened. Predict capacity, not size. |
| Row 9 (`"café"`) too small | Compact strings: one non-Latin-1 character promotes the **entire** array to UTF-16, two bytes per character. |
| `ProductSummary` shallow wrong | Count the references at 4 bytes each **and** re-pad at the end. Or you forgot a superclass. |
| `ProductSummary` deep wildly wrong | Expected, and the real lesson: **you cannot predict deep size of a domain object.** Its cost is in strings and collections whose contents you do not know. Deep size is measured, never derived. |
| **You were right about everything** | Then the drill failed. Add `BigDecimal`, `Instant`, a record with a `List` component, and a Hibernate entity. Something in there will surprise you. |

### Step 6 — turn one error into a permanent artefact

Pick the error that would have cost the most in production — for most people that is the
`String` deep-size one or the `HashMap` one — and write the capacity model it invalidates:

```
/docs/java/baselines/catalog-cache-sizing.md

  Measured on: <JDK version, vendor, container image>
  Flags:       UseCompressedOops=<>, ObjectAlignmentInBytes=<>, UseCompactObjectHeaders=<>
  Method:      GraphLayout over a random 1000-product sample, measured as one graph
  Result:      <bytes> per product (min / median / p95 across three disjoint samples)
  Projection:  100_000 products => <MB>
  Heap:        1200 MB, measured live-set floor <MB> (Topic 68 method)
  Verdict:     <fits / does not fit>, with <MB> headroom
  Re-measure when: the JDK changes, ProductSummary gains a field, or the catalogue's
                   language mix changes (compact strings)
```

### What the drill proves

Three things, in increasing order of importance:

1. **Object size is knowable exactly**, and the tool takes ten seconds. There is never a
   good reason to guess.
2. **Your intuition is systematically biased low**, because intuition counts payload and
   the JVM charges you for headers, references, padding, and wrapper nodes.
3. **The classes you did not write are the ones that surprise you.** You will predict your
   own DTO reasonably well and be wrong about `String`, `HashMap` and `BigDecimal` — which
   are precisely the classes that dominate a real service's heap. **That asymmetry is why
   capacity models must be measured rather than derived.**

---

## Measurement

### The instrument for this topic

**JOL. Not a heap dump, not a profiler, not arithmetic.** JOL comes from the OpenJDK
project and reads the running VM's actual layout decisions. It is the ground truth, and
its answers are exact rather than statistical.

The three entry points, and the question each answers:

```java
// 1. SHALLOW: how big is one instance of this class?
ClassLayout.parseInstance(obj).toPrintable();     // full table with offsets
ClassLayout.parseInstance(obj).instanceSize();    // just the number
ClassLayout.parseClass(Product.class).toPrintable();  // no instance needed

// 2. DEEP: how big is everything reachable from here, counting each object once?
GraphLayout.parseInstance(obj).totalSize();
GraphLayout.parseInstance(obj).toFootprint();     // per-class COUNT / AVG / SUM
GraphLayout.parseInstance(a, b, c).totalSize();   // shared structure counted ONCE

// 3. WHAT VM AM I ON? Print this first, every single time.
VM.current().details();
```

And from the command line, with no code and no instance:

```bash
java -jar jol-cli.jar internals  <fully.qualified.ClassName>   # layout of instances
java -jar jol-cli.jar externals  <fully.qualified.ClassName>   # reachable graph
java -jar jol-cli.jar estimates  <fully.qualified.ClassName>   # size under several VM modes
```

`estimates` deserves its own mention: it prints what the class would cost under different
compressed-oops and alignment configurations **without you having to run those VMs**. That
is the fastest way to put a number on Trap 1's cliff for one of your own classes.

### The decision table: which size do you want?

This is the part people get wrong, and it is a *thinking* error rather than a tooling error.

| The question you are actually asking | The measurement | The tool |
|---|---|---|
| "How much does adding this field cost?" | Shallow size, before and after | `ClassLayout` |
| "Can we cache 100k of these?" | Deep size **of a representative sample measured as one graph** | `GraphLayout.parseInstance(sample.toArray())` |
| "Why is our heap full?" | Retained size by dominator | **MAT on a heap dump** — Topic 79, not JOL |
| "Which code allocates the most?" | Allocation profile | **async-profiler alloc mode** — Topic 78, not JOL |
| "Is this array humongous under G1?" | Real array size including header and padding | `ClassLayout` on the array — Topic 71 |
| "Did the JIT eliminate this allocation?" | **Not a size question at all** | Topic 75, allocation profiler |

### Why a naive `System.nanoTime()` measurement is WRONG here

You will be tempted to measure object size by timing or by differencing free memory:

```java
// DO NOT DO THIS. Every number it produces is noise.
System.gc();
long before = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
Object[] objects = new Object[1_000_000];
for (int i = 0; i < objects.length; i++) {
    objects[i] = new ProductSummary(/* ... */);
}
System.gc();
long after = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
System.out.println("bytes per object = " + (after - before) / 1_000_000);
```

Five independent reasons it lies, and you cannot tell which one is lying:

1. **`System.gc()` is a *request*, not a command.** It may do nothing at all, it may do a
   concurrent cycle that has not finished when you read the numbers, and
   `-XX:+DisableExplicitGC` turns it into a no-op entirely. Your "before" and "after" are
   taken at unknown points in the collector's state machine.
2. **Floating garbage.** Concurrent collectors do not reclaim everything that is dead at
   the moment you ask. Heap-used-after-GC is the live set **plus** an unknown amount of
   floating garbage (Topic 68's Trap 5, and Topic 71's SATB discussion).
3. **TLAB waste.** Each thread's allocation buffer has unusable tail space (Topic 68).
   Heap "used" includes retired TLAB remainders that hold nothing.
4. **You measured the graph, not the object.** Your million `ProductSummary` objects share
   `String`s in ways that depend on how you constructed them. Change the construction and
   the "size per object" changes with no class change.
5. **The JIT may have deleted work you thought you were measuring.** Not here, because the
   array keeps everything alive — but move one line and it can, and Topic 75 is the full
   treatment. This is exactly the failure mode Topic 77 exists to prevent.

**And a sixth reason specific to this topic: you do not need to estimate.** The exact answer
is available. Differencing heap counters to learn an object's size is like weighing a truck
to find out how many boxes are in it when the manifest is in your hand.

Topic 77 is the full treatment of why timing loops lie. Read it before you write any
benchmark you intend to act on.

### Where JMH *is* the right tool in this topic

Only for one question: **does a representation change actually pay off end-to-end?**
Object size predicts memory, not speed. A primitive array is smaller *and* usually faster —
better cache locality, no pointer chasing, no boxing allocation — but "usually" is not
"measurably, in my workload, by this much".

```java
package com.orderflow.bench;

import it.unimi.dsi.fastutil.longs.Long2LongOpenHashMap;
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.random.RandomGenerator;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)                                    // 3 separate JVMs: defeats profile pollution
@State(Scope.Benchmark)
public class ViewCounterRepresentationBenchmark {

    @Param({"10000", "1000000"})
    public int entries;

    private Map<Long, Long>       boxed;
    private Long2LongOpenHashMap  primitive;
    private long[]                probeKeys;

    @Setup(Level.Trial)
    public void setUp() {
        boxed     = new HashMap<>();
        primitive = new Long2LongOpenHashMap();
        var rnd   = RandomGenerator.getDefault();
        probeKeys = new long[4096];
        for (int i = 0; i < entries; i++) {
            boxed.put((long) i, (long) i);
            primitive.put((long) i, (long) i);
        }
        for (int i = 0; i < probeKeys.length; i++) {
            probeKeys[i] = rnd.nextInt(entries);      // random => defeats cache-friendly scans
        }
    }

    @Benchmark
    public void boxedLookup(Blackhole bh) {
        for (long k : probeKeys) {
            bh.consume(boxed.get(k));                  // note: k is boxed at the call site too
        }
    }

    @Benchmark
    public void primitiveLookup(Blackhole bh) {
        for (long k : probeKeys) {
            bh.consume(primitive.get(k));
        }
    }
}
```

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar ViewCounterRepresentationBenchmark -prof gc -rf json -rff repr.json
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Holds the maps in fields JMH controls, so C2 cannot constant-fold them |
| `Blackhole.consume(...)` | Defeats dead-code elimination — without it the lookups can vanish entirely |
| `@Warmup(iterations = 5)` | Lets C2 compile and reach steady state before anything is recorded |
| `@Fork(3)` | Three separate JVMs; exposes run-to-run variance and stops one benchmark's profile polluting the other (Topic 74) |
| `@Param` | Small and large maps are different regimes — one fits in cache, one does not |
| Random probe keys | A sequential scan measures prefetching, not lookup |
| `-prof gc` | **The important one here.** Reports allocated bytes per operation. This is where the boxing shows up. |

**WHAT TO LOOK FOR:** two numbers, and the second matters more.

| What you see | What it means |
|---|---|
| `primitiveLookup` faster, and `-prof gc` shows near-zero bytes/op while `boxedLookup` allocates | The expected result, now proven. The allocation is from boxing the `long` key at the `Map<Long,Long>.get` call site. |
| Both allocate near zero | Escape analysis scalar-replaced the boxes (Topic 75). Genuinely possible, and a reason to check rather than assume. |
| Times overlap within the confidence intervals | **You measured nothing.** Report that. A representation change that does not show up at your cardinality is not worth the dependency. |
| `boxedLookup` faster at 10 000 entries | Plausible — `HashMap` is extremely well optimised and small maps live in cache. The win appears at scale. Report the crossover, not a verdict. |

Report the confidence interval JMH prints, never the point estimate. **If the intervals
overlap, you measured nothing.**

### The numbers to track continuously, in production

Not during a drill — permanently:

| Number | Where from | Why |
|---|---|---|
| Live-set floor (MB) | Old-gen occupancy after mixed/full collections, over a long run (Topic 68's method) | The denominator of every capacity decision, and the thing an object-size regression moves |
| `UseCompressedOops` at startup | Log it in your startup banner from `HotSpotDiagnosticMXBean` | So that nobody ever discovers Trap 1 during an incident |
| Top classes by retained size | Periodic heap dump in a non-production environment, MAT (Topic 79) | Catches a representation problem before it is a capacity problem |
| Pinned shallow size of your cached types | The JOL assertion test from Trap 3 | Makes "we added a field" a visible capacity event in the diff |

---

## Practice exercises

### 1 — Easy: build your own object-size reference card

Produce a table, measured on **your** runtime, for these classes. For each: shallow size,
deep size, and the number of distinct objects in the graph.

`Object`, `Integer`, `Long`, `Double`, `Boolean.TRUE`, `Character`, `String` of length 0 /
10 / 100 (all ASCII), `String` of length 10 containing one emoji, `byte[0]`, `byte[1]`,
`byte[7]`, `byte[8]`, `long[10]`, `Object[10]`, `ArrayList` empty / with 1 / with 10 /
with 11 elements, `HashMap` empty / with 1 / with 12 / with 13 entries, `HashSet` empty,
`BigDecimal.valueOf(19.99)`, `Instant.now()`, `UUID.randomUUID()`, `Optional.of(x)`,
`java.time.LocalDate.now()`.

Then answer, from your own table:

1. Why do `byte[7]` and `byte[8]` cost the same? What does `byte[9]` cost?
2. Why does the `HashMap` jump between 12 and 13 entries? (Topic 12: load factor 0.75 on a
   16-slot table.)
3. What is the cheapest way to store 1000 booleans, and how much does each candidate cost:
   `boolean[]`, `Boolean[]`, `List<Boolean>`, `BitSet`, a single `long[16]`?
4. `Optional.of(x)` versus `x` — what is the per-value overhead, and at what call rate would
   you care? (Topic 26 said "never a field"; now you can put a number on why.)
5. Which of these classes surprised you, and what rule would have prevented the surprise?

Commit the table to `/docs/java/baselines/object-sizes-<jdk-version>.md` with the JDK
version and flags in the header. **It expires when the JDK changes.** Say so in the file.

### 2 — Medium: the audit (combines Topics 01, 11, 12, 18, 21, 25, 27, 65, 68)

Audit `orderflow` for representation cost. This is a real code review with a real output.

1. **Find every boxed collection on a retained path.** Grep for `Map<Long`, `Map<Integer`,
   `List<Long>`, `List<Integer>`, `Set<Long>` in fields (not locals — locals die young and
   Topic 68 says that is nearly free).
2. For each, **measure it** with `GraphLayout` at realistic cardinality from your seeded
   dataset, and compute the primitive-representation cost as a comparison.
3. **Find every `String` field on a class with more than 10 000 live instances.** For each:
   is it Latin-1 in production? What is its average length? Is it a small set of repeated
   values that should be an `enum` (Topic 16) or interned (Topic 18)?
4. **Find every `HashMap` with fewer than 4 expected entries.** Each costs ~48 bytes plus a
   16-slot table plus a `Node` per entry. `Map.of()` (immutable, specialised) or two fields
   is usually smaller. Measure before changing anything.
5. **Find the largest single allocation on any hot path** and check it against Topic 71's
   humongous threshold for `-Xmx1200m`. Remember the array header: `new byte[N]` is `N + 16`
   bytes, padded — **so an array you sized to exactly the threshold is over it.**
6. **Re-derive the catalogue-cache decision** from Example 2 with your measured numbers.
7. Write up the findings as a table: site, current bytes, proposed bytes, live instances at
   peak, total saving, risk of the change. **Sort by total saving.** Then draw a line where
   the saving stops being worth the diff, and defend the line.

The deliverable is the line, not the table. Anyone can list costs; the judgment is knowing
which ones do not matter.

### 3 — Hard: production simulation against the baseline

Prove, end to end, that an object-layout change moves a production metric.

1. **Re-run the Topic 65 baseline** and confirm you are within ±10%. If not, stop — the
   gate rule applies and nothing downstream is measurable.
2. **Record the live-set floor** using Topic 68's method: old-gen occupancy after mixed
   collections, over a ten-minute run at baseline load.
3. **Implement the catalogue cache** from Example 2 as a `ConcurrentHashMap<Long,
   ProductSummary>` on a singleton bean, populated at startup for all 100k products.
4. **Predict, before running:** the new live-set floor (old floor + measured cache deep
   size), the change in young-collection frequency (Topic 68: eden size / allocation rate is
   unchanged, so frequency should be roughly unchanged), and the change in young-collection
   *duration* (Topic 70: cost tracks the live set — but the cache is in **old** gen, so
   young pauses should barely move while **mixed** collections get more expensive). Write
   all three predictions down.
5. **Run the unchanged baseline load for ten minutes** with
   `-Xlog:gc*,gc+heap=debug:file=gc-cached.log:time,uptime,level,tags`.
6. **Measure:** new live-set floor, young-collection frequency and duration, mixed-collection
   duration, p50/p95/p99/p999 per endpoint, Full GC count.
7. **Compare against your predictions and explain every gap.** A prediction you can explain
   after the fact is worth more than one that happened to be right.
8. **Now make the representation change:** replace `BigDecimal price` with `long
   priceMinorUnits` + `String currencyCode`, drop `attributes` from the cached summary, and
   intern or share the `currency` strings. Re-measure the cache's deep size.
9. **Re-run.** Report the delta in live-set floor and in p99, with confidence intervals from
   three runs of each arm.
10. **Write the one-paragraph verdict** you would put in a PR description. It must contain:
    the measured per-entry size before and after, the live-set delta, the p99 delta with its
    uncertainty, and an honest statement of whether the change was worth it.

**The most valuable possible outcome of this exercise is "the p99 did not move".** That is
a real finding: it means the live set was not the constraint, and you have just saved
yourself from a class of speculative optimisation for the rest of your career. Write it up
with the same care you would write a win.

---

## Interview questions

### Q1 — "How much memory does an `Integer` take?"

**MID-LEVEL answer:** "Four bytes — it's an `int` wrapped in an object. Maybe a bit more
for the object overhead."

**SENIOR answer:** "On 64-bit HotSpot with compressed oops, **16 bytes**: a 12-byte header —
8-byte mark word plus 4-byte class word — plus the 4-byte `int`, which happens to land
exactly on the 8-byte alignment boundary so there's no tail padding. That's **4× the
primitive**.

Two things I'd add, because the interesting part isn't the number.

First, **the number is only half the cost in a collection.** In a `List<Integer>` you also
pay 4 bytes for the reference in the backing `Object[]`, so it's 20 bytes per element versus
4 for an `int[]` — that's the 5× figure people quote. For `Long` it's 24 + 4 versus 8, about
3.5×. And in a `Map<Long, Long>` it's worse again, because each entry also carries a
`HashMap.Node`, which is another 32 bytes: roughly 80 bytes of object to store 16 bytes of
numbers.

Second, **I'd verify rather than assert**, because this number moves. Compact object
headers take the header from 12 bytes to 8 on recent JDKs, which makes an `Integer` 12
bytes. Turning compressed oops off — which happens automatically above roughly 32 GB of
heap — takes the header to 16. So the honest answer is 16 bytes *on the runtime I checked*,
and the check is `java -jar jol-cli.jar internals java.lang.Integer`, which takes ten
seconds.

Where this actually matters for me: it's the input to a capacity decision. When someone
asks 'can we hold this in memory', the answer is a multiplication, and this is the unit."

**What separates them:** the mid answer confuses the primitive with the wrapper. The senior
answer gives the number, **decomposes it** into header plus payload plus alignment, extends
it to the collection cost that makes it matter, **names the two configurations that change
it**, and offers the verification command. The single strongest move is refusing to assert
a version-dependent number without saying which runtime it was measured on.

**Interviewer's follow-up:** *"So should we ban boxed collections?"* — No. The overhead is a
ratio, and a ratio is meaningless without a cardinality. Five times a few kilobytes is a few
kilobytes. I'd care above roughly a million elements, or on an object retained for the
process's life, and below that I'd take the readable version every time. What I would ban
is *not knowing which case you're in.*

---

### Q2 — "Our GC pauses got worse after we doubled the heap. Why?"

**MID-LEVEL answer:** "A bigger heap means there's more to scan, so collections take longer.
We should probably tune the pause target down, or switch to a low-pause collector like ZGC."

**SENIOR answer:** "There are three quite different mechanisms that produce that sentence,
and they have three different fixes, so my first job is to find out which one I'm looking
at.

**One — the compressed-oops cliff, which is specific to one heap range.** If the heap
crossed roughly 32 GB, the JVM silently disabled compressed oops. Every reference field went
from 4 bytes to 8 and every object header from 12 to 16. **The live set grew with no code
change**, and since GC cost tracks the live set rather than the garbage, every collection
got more expensive. This is the counter-intuitive one: you added memory and lost effective
capacity. I check it in one command — `jcmd <pid> VM.flags -all | grep CompressedOops` — and
the fingerprint in a class histogram is the same instance counts with larger byte totals.

**Two — a bigger heap means a bigger young generation**, if the young gen is sized as a
percentage of heap, which under G1 it is. More eden means each young collection covers more
allocation, so collections are less frequent but each one has more surviving objects to copy.
Object Copy dominates a young pause. That's usually a good trade for throughput and a bad
one for p99, and it's visible in the `gc+phases=debug` breakdown.

**Three — old gen got bigger, so the concurrent cycle and any mixed or full collection has
more to do.** If they were already close to a Full GC, doubling the heap postponed the
problem and made the eventual pause much worse.

**What I'd actually do:** get the GC log with `-Xlog:gc*,gc+heap=debug` from before and after
on the same load, compare the live-set floor — old-gen occupancy after mixed collections —
and compare the phase breakdown. If the live set jumped without a data change, it's
compressed oops. If Object Copy grew proportionally, it's young sizing. If it's Full GCs,
it's old gen and I read Topic 71's failure modes.

**And I'd separate GC pause from stop-the-world duration before concluding anything**, with
`-Xlog:safepoint*`. A reported pause is GC work at the safepoint; the time to *reach* the
safepoint is a different number with a different cause, and I've seen teams tune GC for
weeks on a problem that was a counted loop."

**What separates them:** the mid answer has one model (bigger = slower) and reaches for a
flag. The senior answer holds **three competing hypotheses**, names the evidence that
distinguishes them, knows the compressed-oops cliff exists at all, and refuses to attribute
a pause to GC before separating GC time from time-to-safepoint.

**Interviewer's follow-up:** *"Suppose it is the compressed-oops thing. What do you do?"* —
Four options, and I'd measure before choosing. Come back below the threshold — find the exact
one by binary-searching `-Xmx` against `PrintFlagsFinal`, it's usually a bit under 32 GB.
Or raise `ObjectAlignmentInBytes` to 16, which doubles the addressable range at the cost of
more padding per object — good for large objects, bad for small ones, and I'd measure the
live set both ways. Or go far enough past the threshold that the raw gigabytes outweigh the
inflation. Or, most likely for a service, **run two smaller JVMs**, which also gives each
collector a smaller live set and therefore cheaper pauses.

---

### Q3 — "We need to hold 200 million small records in memory. Design the representation."

**MID-LEVEL answer:** "I'd use a `HashMap<Long, Record>` and give the JVM a big heap —
maybe 64 GB — and use G1 or ZGC to keep pauses down."

**SENIOR answer:** "Before designing anything I'd want two numbers: **the per-record size in
the representation we're considering**, and **the access pattern** — point lookups by key,
range scans, or full iteration. Those two decide everything.

Take the naive design and cost it. A `HashMap<Long, Record>` entry is a `Node` at 32 bytes,
a boxed `Long` key at 24, a table slot at 4, plus the record. Call the record five fields:
header plus fields plus padding, say 40 bytes if it's all primitives, several hundred if it
has strings. **So before the payload we're paying about 60 bytes per entry in pure
container overhead. Times 200 million, that's 12 GB of nothing but bookkeeping.**

Now the thing most people miss: **a heap that size is over the compressed-oops threshold.**
Above roughly 32 GB every reference goes to 8 bytes and every header to 16. Our 60 bytes of
overhead becomes closer to 90. **We'd be spending well over 20 GB on container overhead
alone**, and every GC cycle would be tracing 200 million objects' worth of references.

So the design goes in a different direction:

1. **Structure of arrays, not array of structures.** Parallel primitive arrays — a `long[]`
   of keys, a `long[]` of one field, an `int[]` of another. Zero per-record object overhead,
   zero headers, perfect cache locality on scans, and **the GC has almost nothing to trace**:
   a handful of huge arrays rather than 200 million objects. That last point is the big one —
   Topic 70's rule is that cost tracks the live set, and specifically the number of live
   *references* to chase.
2. **A primitive open-addressed map** — fastutil's `Long2IntOpenHashMap` mapping key to index
   — if we need lookup by key. No `Node`s, no boxing.
3. **Or off-heap entirely.** Topic 80: FFM `Arena` or memory-mapped segments, so the data
   isn't in the GC's graph at all. That's the honest answer at 200 million records, and it
   trades GC pressure for manual lifetime management and a harder debugging story.
4. **Or don't hold it in one JVM.** Shard across several processes. Smaller live set each,
   all comfortably under the compressed-oops threshold, and cheaper pauses everywhere.

**And I'd prototype before committing.** Build 1% of it — 2 million records — in two
candidate representations, measure the deep size with JOL, measure the GC behaviour under a
representative access pattern, and extrapolate. That's a day of work and it's the difference
between a design and a guess."

**What separates them:** the mid answer picks a data structure and buys hardware. The senior
answer **costs the container overhead separately from the payload**, spots that the required
heap crosses the compressed-oops threshold and that this makes the overhead worse, knows that
GC cost scales with object *count* and reference density rather than raw bytes, and proposes
a prototype instead of a decision.

**Interviewer's follow-up:** *"Your parallel-arrays idea makes the code much uglier. Is that
worth it?"* — At 200 million, yes, and I'd contain the ugliness behind an accessor API so
callers still see a record-shaped view. At 200 thousand, absolutely not — 60 bytes of
overhead times 200 000 is 12 MB, which is noise, and I'd use the `HashMap` and spend the
effort elsewhere. **The cardinality is what makes the trade, not the technique.**

---

### Q4 — "What's actually in a Java object header, and why would I care?"

**MID-LEVEL answer:** "There's some metadata the JVM uses — a pointer to the class, and some
GC information. It's around 12 or 16 bytes."

**SENIOR answer:** "Two words on 64-bit HotSpot with compressed oops: an **8-byte mark word**
and a **4-byte class word**, so 12 bytes, and arrays add a 4-byte length making 16.

The class word is the simple one — a pointer to the class metadata in metaspace, compressed
into 4 bytes via compressed class pointers, which is why there's a separate 'compressed class
space' region with its own `OutOfMemoryError`.

**The mark word is the interesting one, because it's multiplexed.** It holds different things
at different times, distinguished by tag bits: the identity hash code once it's been computed
and from then on forever; the object's GC age, which is what the tenuring threshold compares
against; the lock state, which is either a pointer to a lock record on a thread's stack or,
once inflated, a pointer to a heavyweight monitor; and during evacuation, a forwarding
pointer to where the object was moved.

I care for four concrete reasons.

**Sizing.** 12 bytes on every object means small objects are mostly overhead. A `Boolean` is
16 bytes to hold one bit. That's the arithmetic behind every boxed-collection conversation.

**Locking.** Topic 85's story is entirely about these bits — 'uncontended `synchronized` is
cheap' means 'it's a compare-and-swap on the mark word', and 'contention is expensive' means
'the JVM had to allocate a real monitor and store a pointer to it here'.

**Because features compete for the same 64 bits.** Computing an identity hash code and using
certain lock optimisations can conflict, because the word can only hold one thing at a time.
That's the kind of interaction you can only see from the layout.

**And because it's changing.** Compact object headers fold the class word into the mark word
and take the header from 12 bytes to 8. On a heap full of small objects that's a
double-digit percentage of the live set. I'd check `-XX:+PrintFlagsFinal -version | grep -i
CompactObjectHeaders` on any runtime before I quoted a size."

**What separates them:** the mid answer knows there is a header. The senior answer knows what
is in it, knows it is **multiplexed rather than a fixed record**, connects it to three other
topics (GC ages, locking, capacity), and knows it is currently changing across JDK versions.

**Interviewer's follow-up:** *"Where does the identity hash code live before it's computed?"*
— It doesn't exist. It's computed lazily on first request and then stored in the mark word,
because it must be stable for the object's lifetime and the object may move — so it can't be
derived from the address. That's also why `System.identityHashCode` on a huge number of
objects has a cost people don't expect, and why a collection keyed on identity hash forces
that computation on everything you put in it.

---

### Q5 — "How would you prove how much memory our cache actually uses?"

**MID-LEVEL answer:** "I'd look at the heap usage in our monitoring before and after
enabling the cache, or take a heap dump and look at the class histogram."

**SENIOR answer:** "Depends which of two questions we're asking, and they have different
answers.

**If the question is 'how much does one entry cost' — JOL.** I'd take a random sample of
real entries, not the first N, because the first N are usually sorted by id and
systematically unrepresentative. Then `GraphLayout.parseInstance(sample.toArray())` measured
**as one graph**, so shared objects — interned currency strings, cached small `Long`s — are
counted once, which is what a real cache does too. Divide by the sample size. I'd do three
disjoint samples and report a range, because entry cost varies with string lengths and
collection sizes.

**If the question is 'how much would we get back by dropping the cache' — a heap dump and
MAT.** Deep size counts everything reachable; **retained size counts what would actually die
with it**, which is the number that answers a capacity question. Those differ by exactly the
shared structure, and for a cache holding references to objects that are also live elsewhere
the gap can be large. That's Topic 79.

**What I would specifically not do** is difference heap counters around a `System.gc()`.
That's the intuitive approach and it's unreliable for four reasons: `System.gc()` is a
request that may be a no-op, concurrent collectors leave floating garbage so
heap-used-after-GC is live set plus an unknown, retired TLABs count as used, and you end up
measuring the whole graph's sharing rather than the cache's cost. It gives you a number, and
you can't tell how wrong it is.

**And I'd record the runtime with the measurement**, because object sizes depend on
compressed oops and on compact object headers, so a number without a JDK version and a flag
list is not reproducible. Then I'd pin the shallow size of the cached class in a unit test
with JOL, so that when someone adds a field it shows up as a **capacity event in the diff**
rather than as a surprise in production three months later."

**What separates them:** the mid answer reaches for aggregate monitoring. The senior answer
**distinguishes deep size from retained size and matches each to the question it answers**,
knows the sampling trap and the shared-structure trap, can articulate exactly why the
intuitive `System.gc()` method is unreliable, and turns the one-off measurement into a
regression test.

**Interviewer's follow-up:** *"The measurement says 400 bytes per entry and we want a million
entries. What do you tell the team?"* — 400 MB, which I'd then compare against three things:
our heap, our measured live-set floor, and our GC headroom. And I'd say the cost isn't only
the 400 MB — it's a permanent increase in the live set, which by Topic 70's rule makes every
collection more expensive from then on. So the decision isn't "does it fit", it's "does it
fit *and* what does it do to p99", and the only way to answer the second half is to re-run
the load baseline with it enabled.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. An `Integer` is 16 bytes and a `Long` is 24. A `Long` holds twice the data for 1.5× the
   cost. **Derive the general rule** about when adding a field to a class is free, and state
   the one JOL output line that tells you whether the next field is free in a specific class.

2. Compressed oops encode a reference as `address >> 3`. Derive the 32 GiB limit from first
   principles — 32 bits, shift of 3 — and then explain why the *observed* threshold on a
   real JVM is somewhat **below** 32 GB, and what else the JVM has to fit in that address
   space.

3. `GraphLayout.parseInstance(a)` plus `GraphLayout.parseInstance(b)` does not equal
   `GraphLayout.parseInstance(a, b)`. Explain exactly when they differ and by how much. Then
   say which of the three numbers is the right one for sizing a cache, and why.

4. A `String` object is 24 bytes regardless of its contents. Explain how a four-character
   string and a four-thousand-character string can both be 24 bytes, and what that tells you
   about the difference between shallow and deep size. Then predict which of `"cafe"` and
   `"café"` has the larger deep size, and by how much, and say why.

5. `new HashMap<>()` has a null table. Construct a realistic `orderflow` scenario where this
   lazy allocation saves a meaningful amount of memory, and one where it makes a
   `GraphLayout` measurement misleading enough to cause a wrong decision.

6. A colleague proposes raising `ObjectAlignmentInBytes` to 16 so the service can use a
   48 GB heap with compressed oops. Derive the two competing effects on the live set, say
   which class of object graph makes the trade good and which makes it bad, and name the
   single measurement that decides it.

7. Topic 71 says a G1 humongous allocation is anything at least half a region. Your region
   size is 2 MB, so the threshold is 1 MB. Someone writes `new byte[1024 * 1024]` believing
   it is exactly at the threshold and therefore safe. **Explain precisely why they are
   wrong**, using this document's arithmetic, and give the largest `byte[]` length that is
   genuinely not humongous.

---

## Quick reference card

### The arithmetic

```
object header    = 12 bytes   (8 mark + 4 class)      // compressed oops on, 64-bit HotSpot
array header     = 16 bytes   (8 mark + 4 class + 4 length)
reference        =  4 bytes   compressed / 8 uncompressed
alignment        =  8 bytes   (ObjectAlignmentInBytes)

object_size      = round_up(header + sum(field_widths) + internal_padding, alignment)
array_size       = round_up(16 + element_width × length, alignment)

compressed oops reach = 2^32 × ObjectAlignmentInBytes = 32 GiB at the default alignment
```

> **Version note, one line:** compact object headers
> (`-XX:+UseCompactObjectHeaders`) reduce the object header from 12 bytes to 8 on recent
> JDKs; **I am not asserting whether it is on by default, or still experimental, on your
> JDK 25 build** — settle it with
> `java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders` and confirm the
> consequence with `java -jar jol-cli.jar internals java.lang.Object`.

### Common sizes — as PREDICTIONS to verify, never as facts to quote

| Thing | Predicted shallow | Predicted deep | Verify with |
|---|---|---|---|
| `Object` | 16 | 16 | `jol-cli internals java.lang.Object` |
| `Integer` | 16 | 16 | `jol-cli internals java.lang.Integer` |
| `Long` / `Double` | 24 | 24 | `jol-cli internals java.lang.Long` |
| `String` (n ASCII chars) | 24 | 24 + round_up(16 + n, 8) | `ClassLayout` + `GraphLayout` |
| `String` (any non-Latin-1 char) | 24 | 24 + round_up(16 + 2n, 8) | as above |
| `byte[n]` | round_up(16 + n, 8) | same | `ClassLayout.parseInstance(arr)` |
| `Object[n]` | round_up(16 + 4n, 8) | plus the elements | `GraphLayout` |
| `new HashMap<>()` | 48 | 48 (table is null) | `GraphLayout.totalSize()` |
| `HashMap` with 1 entry | 48 | + 80 (table) + 32 (Node) + key + value | `toFootprint()` |
| `HashMap.Node` | 32 | plus key and value | `jol-cli internals java.util.HashMap$Node` |
| `ArrayList` | 24 | + backing array + elements | `toFootprint()` |
| `long[10]` | 96 | 96 | `ClassLayout` |
| `List<Long>` of 10 | 24 | ≈ 320 | `toFootprint()` — **check the COUNT column** |

**Every number in that table is arithmetic from the rules above, not a measurement. Two of
them are probably wrong on your JDK. Run the tool.**

### Ratios worth carrying in your head

```
List<Integer> vs int[]      ≈ (4 slot + 16 box) / 4  = 5×
List<Long>    vs long[]     ≈ (4 slot + 24 box) / 8  = 3.5×
Map<Long,Long> per entry    ≈ 32 Node + 24 key + 24 value + 4 slot = 84 bytes for 16 of data
String        vs raw bytes  ≈ 24 + 16 header bytes of overhead per string
compressed oops off         ≈ live set grows by a double-digit % on a reference-dense graph
```

### Commands

```bash
# What am I running? Print this before quoting any size.
java -XX:+PrintFlagsFinal -version | grep -E "UseCompressedOops|UseCompressedClassPointers|ObjectAlignmentInBytes|CompactObjectHeaders"
jcmd <pid> VM.flags -all | grep -iE "compressedoops|objectalignment|compactobjectheaders"

# Which compressed-oops mode did I get, and where is the heap based?
java -Xmx<size> -Xlog:gc+heap+coops=info -version

# Layout of a class, with no instance and no code.
java -jar jol-cli.jar internals  java.lang.String
java -jar jol-cli.jar externals  java.lang.String
java -jar jol-cli.jar estimates  com.orderflow.catalog.ProductSummary

# Find YOUR compressed-oops threshold.
for s in 28g 30g 31g 32g 33g; do printf "%s " $s; \
  java -Xmx$s -XX:+PrintFlagsFinal -version 2>/dev/null | grep "^ *bool UseCompressedOops" | awk '{print $4}'; done

# In code.
ClassLayout.parseInstance(x).toPrintable()      // shallow, with offsets
ClassLayout.parseClass(C.class).instanceSize()  // shallow, no instance needed
GraphLayout.parseInstance(x).totalSize()        // deep, each object once
GraphLayout.parseInstance(arr).toFootprint()    // per-class COUNT / AVG / SUM
VM.current().details()                          // the configuration you measured under

# JOL needs these on JDK 17+.
--add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED

# The other tools, for the questions JOL does not answer.
jcmd <pid> GC.class_histogram      # instance counts and byte totals (triggers a full GC)
jcmd <pid> GC.heap_dump /dumps/x.hprof   # then MAT: retained size, dominator tree (Topic 79)
```

### How to read a JOL layout table — field guide

```
<class> object internals:
OFF  SZ   TYPE DESCRIPTION            VALUE
<o>  <n>       (object header: mark)  <bits>
<o>  <n>       (object header: class) <bits>
<o>  <n>  <T>  <fieldName>            <value>
<o>  <n>       (object alignment gap)
Instance size: <N> bytes
Space losses: <A> bytes internal + <B> bytes external = <A+B> bytes total
```

***Illustration of the format, not captured output. All values are placeholders.***

| Line / column | How you use it |
|---|---|
| `OFF` | Field offset. Compare against declaration order to see reordering. |
| `SZ` | Field width. A reference at `4` proves compressed oops are on. |
| `(object header: mark)` | 8 bytes. Topic 85 lives here. |
| `(object header: class)` | 4 bytes compressed, 8 not. Absent on compact headers — folded into the mark word. |
| `(object alignment gap)` | Tail padding. **If this is ≥ 4, one more `int` is free.** |
| `Instance size` | The shallow answer. What "how big is one" means. |
| `internal` loss | Padding between fields. Only worth chasing at millions of instances. |
| `external` loss | Tail padding. The "is the next field free" number. |

### Gotchas checklist

- [ ] Print the VM configuration before quoting any size. Never recall one.
- [ ] Header is 12 for objects, **16 for arrays** — the length word is real.
- [ ] Pad to 8. An object ending at 20 bytes occupies 24.
- [ ] Shallow ≠ deep ≠ retained. Know which question you are asking.
- [ ] `GraphLayout` on a Hibernate entity walks into the whole application. Use DTOs.
- [ ] Measure a **sample as one graph**, so sharing is counted once — like a real cache.
- [ ] Random sample, not the first N. The first N are sorted and unrepresentative.
- [ ] `HashMap`'s table is null until the first `put`.
- [ ] `ArrayList` capacity ≠ size, grows ~1.5×, and never shrinks.
- [ ] One non-Latin-1 character doubles a `String`'s `byte[]`.
- [ ] `new byte[N]` is `N + 16` bytes — check it against Topic 71's humongous threshold.
- [ ] Above ~32 GB of heap, compressed oops turn off **silently**. No warning, no log line.
- [ ] Find your own threshold; it is below 32 GB and platform-dependent.
- [ ] A 5× overhead on 10 KB is 50 KB. State the cardinality before proposing a rewrite.
- [ ] Pin the shallow size of hot cached classes in a test so field additions are visible.

---

## When would I use this at work?

**1. Someone proposes an in-process cache and quotes a size from intuition.**
You spend ten minutes with `GraphLayout` on a random sample of real entries and come back
with a measured per-entry cost, a projection, and the live-set delta. Two outcomes are both
wins: either the number is small and the cache ships with a documented capacity model, or the
number is 10× the estimate and you prevented an incident whose postmortem would have said
"GC issue". This is the single highest-frequency use of this topic, and it takes less time
than the meeting where it gets debated.

**2. Capacity planning above 30 GB of heap.**
A team asks for bigger instances to fix GC pauses. You ask what heap they are moving to, and
if it crosses the compressed-oops threshold you explain — with `PrintFlagsFinal` output from
their own runtime, not a blog post — that they will lose a chunk of the capacity they are
buying, and you present the four options with their trade-offs. This is a 20-minute
conversation that changes a hardware decision, and you will very likely be the only person
in the room who knows the cliff exists.

**3. Reviewing a PR that adds a field to a high-cardinality class.**
`ProductSummary`, an entity with 5 million rows, a cache value type. You run
`jol-cli internals` before and after, look at the external space loss, and comment with the
actual per-instance delta and the total at peak cardinality. Usually the answer is "this is
free, ship it", and saying so with a number is worth more than saying it with a shrug —
because the time it is *not* free, the same habit catches it, and by then the assertion test
from Trap 3 catches it automatically.

---

## Connected topics

**Prerequisites:**

- **01 — Primitives, wrappers, autoboxing**: this topic supplies the missing number. Boxing
  allocates *16 bytes for an `Integer`, 24 for a `Long`*, and with the reference slot that is
  the 4–5× figure that makes `List<Integer>` a capacity problem rather than a style
  preference. Everything Topic 01 asserted, you can now derive and verify.
- **11 — List implementations**: `ArrayList`'s backing `Object[]` is 16 + 4n bytes, its
  capacity is not its size, it grows ~1.5× and never shrinks. `LinkedList`'s per-node object
  is the reason it loses on memory as decisively as it loses on cache locality.
- **12 — HashMap internals**: a `Node` is 32 bytes and the table is 16 + 4n. That is why a
  small map is expensive per entry, why the table being lazily allocated matters, and why
  the resize at load factor 0.75 is a memory event as well as a rehash.
- **18 — Strings, the pool, compact strings**: a `String` is two objects, and the `coder`
  byte decides whether the second one costs n or 2n bytes. `intern()` trades heap for a
  lookup — and now you can put a number on both sides of that trade.
- **21 — Lambdas and `invokedynamic`**: a capturing lambda allocates an object per
  evaluation, and this topic tells you what that object costs.
- **25 — Parallel streams**: parallel work multiplies allocation and boxing across threads;
  the per-element overhead you compute here is what gets multiplied.
- **27 — Records**: a record is a normal object with normal layout. A record with a `List`
  component is shallowly immutable and its deep size is unbounded — measure it.
- **65 — The load-testing gate**: the recorded baseline is the control for the hard exercise.
  Without it, "the cache didn't hurt p99" is an opinion.
- **66 — JVM architecture**: the heap that these objects live in, and the metaspace that the
  class word points into.
- **68 — Heap generations and TLABs**: allocation is a pointer bump *of this many bytes*.
  Object size is the multiplier that converts an allocation *count* into an allocation
  *rate*, and the GC age lives in the mark word this topic dissects.

**This unlocks:**

- **70 — GC fundamentals**: cost tracks the live set, and the live set is measured in bytes.
  This topic is where the bytes come from. Compressed oops turning off is a live-set event.
- **71 — G1 in depth**: the humongous threshold is compared against the object's **real**
  size — header, array length word, padding included. `new byte[1MB]` is over 1 MB.
- **72 — ZGC and Shenandoah**: coloured pointers put metadata *in the reference itself*,
  which is a direct consequence of how references are represented — and it is why
  generational ZGC and compressed oops have historically not coexisted. Check on your JDK.
- **73 — Safepoints**: object headers are rewritten with forwarding pointers during
  evacuation, which is one of the reasons relocation needs the world stopped or a barrier.
- **74 — JIT and tiered compilation**: field offsets are baked into compiled code, which is
  why layout is fixed at class-load time and why redefinition (Topic 81) is restricted.
- **75 — Escape analysis**: a scalar-replaced object has **no layout at all** — its fields
  become registers and the header is never written. This topic is the cost that Topic 75
  eliminates.
- **76 — Bytecode**: `getfield` and `putfield` resolve to the offsets you just printed.
- **77 — JMH**: the only correct way to answer "is the smaller representation also faster",
  and the reason the `System.gc()`-differencing approach above is fiction.
- **78 — Profiling**: async-profiler's allocation mode names the *site*; JOL names the
  *size*. You need both to convert an allocation profile into an allocation rate in MB/s.
- **79 — Heap dumps and MAT**: retained size answers "what dies with this", which is the
  question JOL's deep size only bounds. When old gen grows, this is the tool.
- **80 — Off-heap memory**: the escape hatch when per-object overhead and GC tracing cost
  more than manual lifetime management. FFM `Arena` has no headers and no alignment padding
  you did not choose.
- **82 — JVM tuning and containers**: `MaxRAMPercentage` decides `-Xmx`, which decides
  whether compressed oops are on, which decides your live set. This document's arithmetic is
  downstream of that one flag.
- **83 — GraalVM native image**: a different object model with a build-time-computed heap;
  the contrast sharpens what HotSpot's layout is actually for.
- **85 — `synchronized` and monitors**: the mark word is the lock. Lock inflation is a state
  change in the eight bytes this document opened with, and Topic 85 is its full treatment.
- **96 — False sharing**: `@Contended` is deliberate padding at cache-line granularity —
  the same mechanism as alignment, used to solve a hardware problem.
- **101 — Virtual threads**: a virtual thread's stack is a heap object that grows and
  shrinks. Millions of them make object-size arithmetic a scheduling concern.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately hedged
rather than asserted: whether `UseCompactObjectHeaders` is enabled by default on your JDK 25
build, the exact `-Xmx` at which compressed oops are disabled on your platform, and the
precise field list of `java.lang.String` on your JDK. Each has a command in the Hands-on
section that settles it in under a minute. **No object size in this document was measured —
every size given is arithmetic derived from the stated rules, offered as a prediction for you
to falsify with JOL.** The JOL and footprint tables shown are labelled illustrations of the
output format with placeholder values; none of them is captured output. What has been stable
since 64-bit HotSpot got compressed oops and will still be true at 2am: objects have a
header, arrays have a length word, everything is padded to the alignment, references are
narrower below the threshold than above it, and the tool is faster than the argument.*
