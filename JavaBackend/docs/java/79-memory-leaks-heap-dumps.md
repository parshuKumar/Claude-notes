# 79 — Memory Leaks: Finding Them From a Heap Dump

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: two deliberate leaks in `orderflow`, both run under the Topic 65 baseline load against the real dataset (100k products, 1M orders, 5M order lines). Drill (a): an unbounded `static Map<String, Order>` under `-Xmx512m` until it dies, then Eclipse MAT's dominator tree until you can name the retaining path out loud. Drill (b): a request context in a `ThreadLocal` on a fixed pool, never removed — where retention grows to pool-size × context-size and then **stops**, which is exactly why this one gets closed as "not a leak" and ships.

---

## R0 — READ THIS BEFORE ANY OTHER LINE IN THIS DOCUMENT

**I do not have a JVM, a heap dump, or Eclipse MAT. Nothing in this document is captured
tool output.**

Specifically, you will not find here:

- a MAT dominator-tree screenshot or a MAT figure of any kind,
- a retained size, a shallow size, an object count, or a percentage of heap,
- a histogram with numbers, a dump file size, or a parse time,
- "the leak was 340 MB in a `ConcurrentHashMap$Node[]`" or anything shaped like it.

**Why this rule is at its strictest here:** the entire skill is *reading a dominator tree
you produced yourself and naming a retaining path*. Handing you a conclusion replaces the
skill with a memory. And a fabricated retained size for `orderflow` would send you hunting
for an object that does not exist in your build — you would find something close enough,
stop, and learn nothing.

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see" → "what it means"** table covering plausible and surprising outcomes.

### The labelled exceptions

To read a MAT dominator tree you must know its columns. To read an hprof file you must know
its record structure. In four places I show **structure only** — field names, column
headings, record layout — with every value replaced by `<n>`, `<name>`, `<Class>` or `xxx`,
each carrying the inline label:

> *illustration of the format, not captured output*

Placeholders only. No plausible-looking numbers.

### Spec-level facts I state plainly, each with a confirming command

1. **A Java memory leak is unintended reachability, not unfreed memory.** Confirm: run
   drill (a) and observe that the objects are perfectly reachable through a live path the
   whole time.
2. **MAT's dominator tree answers "if this object were collected, how much would go with
   it".** A histogram does not — it groups by class and sums shallow sizes. Confirm: open
   the same dump in both views (Proof 4) and see that they name different things.
3. **`ThreadLocalMap` entries hold a WEAK reference to the key and a STRONG reference to
   the value.** Confirm: read the JDK source (`ThreadLocal.ThreadLocalMap.Entry`) with the
   command in Machine-level reality §4.
4. **A heap dump is taken at a safepoint and stops the world for its duration.** Confirm:
   take one with `-Xlog:safepoint*` enabled and find the operation in the log (Proof 7).

### THE RULE

> **If your MAT output disagrees with anything in this document, YOUR OUTPUT IS THE
> TRUTH.** The dump of the JVM that actually died is the only authority. Not mine, not a
> blog post's, and not the plausible story you constructed before opening the tool.

---

## Mechanical statement

Read this three times.

> **A Java leak is unintended reachability, not unfreed memory.**
>
> There is no `free()` to forget. The collector reclaims everything unreachable from a GC
> root, correctly, every time. So when memory grows without bound, exactly one thing is
> true: **something you did not think about is still holding a reference.** The object is
> not "leaked" in the C sense; it is *retained*, deliberately and correctly, by a path you
> did not intend to create.
>
> That reframing changes the whole investigation. You are not looking for missing cleanup.
> **You are looking for a path from a GC root to an object that should be dead.**
>
> **MAT's dominator tree answers exactly the question you have.** Object A *dominates*
> object B if **every** path from **any** GC root to B passes through A. The dominator
> tree makes each object's parent its immediate dominator, and the **retained set** of A is
> everything A dominates. Retained size is the sum of that set's shallow sizes — which is,
> in plain words, *"how much memory would be freed if A became unreachable."*
>
> **A histogram is not that question.** A histogram groups objects by class and sums their
> shallow sizes. It reliably tells you that you have a lot of `byte[]`, `char[]`, `String`
> and `Object[]`, which is true of every Java heap that has ever existed and tells you
> nothing about what is holding them. **The histogram names the material; the dominator
> tree names the culprit.**
>
> **And there are exactly four shapes you will meet in practice:**
>
> 1. **An unbounded collection reachable from a static field.** A cache with no eviction.
>    The most common leak in Java, by a wide margin.
> 2. **A listener or callback registered and never unregistered.** The publisher outlives
>    the subscriber and holds it alive.
> 3. **A `ThreadLocal` value never removed, on a thread from a pool.** The thread is a GC
>    root and outlives the request; the value is strongly held from the thread.
> 4. **A non-static inner class (or a lambda capturing `this`) outliving its enclosing
>    instance**, via the synthetic `this$0` field.
>
> Shape 3 is the subtle one, and it is the reason this topic has two drills. Its retention
> is **bounded** at pool-size × context-size and then stops growing — so the graph
> plateaus, the OOM never comes, and it gets closed as "not a leak". It is a leak. It has
> permanently inflated your live set, and **GC cost is a function of the live set** (Topic
> 70), so you now pay for it on every single collection, forever.

The one-line version:

*"Java doesn't leak, it retains. I take a dump, open the dominator tree, find the largest
retained set, and walk the path to GC root."*

---

## The bridge from what you know

### Chrome heap snapshots are an HONEST ANALOGUE — and this is the closest analogy in Phase 8

If you have ever opened DevTools → Memory → "Take heap snapshot" to chase a leak in a
long-lived React page, you already own most of this topic's conceptual vocabulary. The
mapping is close to exact:

| Chrome DevTools Memory | Eclipse MAT | Same thing? |
|---|---|---|
| **Shallow Size** | **Shallow Heap** | Yes — the object's own bytes, excluding what it references |
| **Retained Size** | **Retained Heap** | Yes — everything that would be freed if this object went away |
| **Retainers** pane (the reverse-reference path) | **Path to GC Roots** | Yes — the chain that keeps it alive |
| "Objects retained by X" / dominator view | **Dominator Tree** | Yes — same graph-theoretic dominator concept |
| **Comparison** between two snapshots | **Compare Basket / delta histogram** | Yes — the "what grew" technique |
| GC roots: window, DOM tree, closures | GC roots: static fields, thread stacks, JNI globals, live threads | Same idea, different membership |
| Three-snapshot technique (snapshot, act, snapshot, act, snapshot) | Same technique, same reason | Yes — this transfers 1:1 |

**The core investigative move is identical in both worlds:** *sort by retained size, find
the biggest thing, look at what keeps it alive, and ask why that path exists.* You already
know how to do this. That is a real head start and you should use it with confidence.

The most valuable transferred instinct is this one: **in Chrome you learned that "the
detached DOM node is 40 MB" is not the finding — the finding is *who still points at it*.**
That is exactly the MAT lesson. The retained size tells you where to look; the path to GC
root tells you what to fix.

### Where the analogy runs out — and it runs out at the interesting part

**Difference 1 — threads are GC roots, and there are hundreds of them.**

JavaScript has one main thread. Its roots are the global object, the DOM, and the active
call stack. Java's roots include **every live thread's stack**, plus the thread objects
themselves, plus JNI global references, plus every static field of every loaded class,
plus objects held by the JVM internally.

A thread pool with 200 threads is 200 GC roots that **never die for the lifetime of the
process**. There is no browser equivalent, and it is the entire basis of leak shape 3.

**Difference 2 — `ThreadLocal` on a pooled thread has NO JavaScript equivalent at all.**

Say this out loud, because it is the part of this topic you cannot reason about from your
existing knowledge:

- In Node, request state lives in a closure, in the request object, or in
  `AsyncLocalStorage`. When the request finishes, the closure is unreachable and the state
  goes. The **execution context is per-request** because the runtime is single-threaded and
  the async context is scoped.
- In Java thread-per-request, a `ThreadLocal` value is stored **in a map that is a field of
  the `Thread` object**. The thread does not end at the end of the request; it goes back to
  the pool and picks up the next one. The value stays in the map. **The request's data
  outlives the request by the lifetime of the process.**

Every framework you touch does this: Spring's `SecurityContextHolder` (Topic 56), the MDC
in your logging framework (Topic 120), transaction synchronisation, request-scoped beans
(Topic 38). They all clean up after themselves. **Your code has to as well**, and nothing
in your Node background will remind you.

**Difference 3 — reference strength is a design tool in Java.**

JavaScript has `WeakMap`, `WeakSet`, and more recently `WeakRef` and `FinalizationRegistry`,
and you have probably never used the last two. Java has a full hierarchy — strong, soft,
weak, phantom — plus `ReferenceQueue`, plus `Cleaner`, and library code uses all of them.
`ThreadLocalMap`'s **weak key, strong value** design is precisely the sort of nuance that
produces a leak nobody expects, and it is the subject of drill (b).

**Difference 4 — class loaders and off-heap memory.**

A Java leak can retain an entire `ClassLoader`, and through it every class it loaded and
every static field of every one of those classes — the classic application-server redeploy
leak. And a Java process can grow without the *heap* growing at all, via direct byte
buffers and native allocations (Topic 80). Neither has any browser analogue, and the second
one means **"the heap looks fine" does not mean "memory is fine"**.

**Difference 5 — the dump is a file, and taking it stops the world.**

In Chrome, taking a snapshot pauses a tab you were already looking at. In Java, taking a
heap dump requires a global safepoint (Topic 73) and writes a file roughly the size of your
live heap. On a production service with an 8 GB heap that is a multi-second freeze and 8 GB
of disk. This is an operational decision, not a click.

### Summary

| Transfers cleanly | Does not transfer |
|---|---|
| Shallow versus retained size | Threads as long-lived GC roots |
| Dominators, and "what would be freed" | `ThreadLocal` on a pooled thread |
| Path to GC root / retainers | Reference strengths as a design tool |
| Compare-two-snapshots to find growth | Class-loader retention |
| "Who points at it?" as the real question | Off-heap memory (Topic 80) |
| Sorting by retained size first | The dump being a stop-the-world operation with a file cost |

**Verdict: HONEST ANALOGUE for the tooling and the method; no analogue at all for the Java
leak shapes that involve threads and class loaders.**

---

## What is this?

**A heap dump is a complete snapshot of every object in the Java heap at one instant**,
plus the references between them and the set of GC roots. It is a file, usually in the
**hprof** binary format, and it is typically about the size of your live heap.

**Eclipse MAT (Memory Analyzer Tool)** parses that file, builds indexes, and computes the
dominator tree. It is the standard tool. It is free, it is old, its UI is dated, and it is
excellent at exactly one thing: answering "what is holding this memory".

### The vocabulary, defined precisely

You need all of these, and imprecision about any of them makes the tool unreadable.

| Term | Definition |
|---|---|
| **GC root** | An object reference the collector treats as inherently live: a static field of a loaded class, a local variable on any live thread's stack, a JNI global reference, a live `Thread` object, a monitor currently held, and a few JVM-internal categories |
| **Reachable** | There exists a path of references from some GC root to this object. Reachable = not collectable. **This is the only definition of "in use" the JVM has** |
| **Shallow heap** | The bytes of one object itself: header (Topic 69) plus its fields, aligned. A `HashMap` with a million entries has a *tiny* shallow heap — a handful of fields |
| **Retained set** of A | Every object that would become unreachable if A became unreachable |
| **Retained heap** of A | The sum of the shallow heaps of A's retained set. **"How much would I get back"** |
| **Dominator** | A dominates B if **every** path from **any** GC root to B goes through A |
| **Immediate dominator** | The closest such A. This is B's parent in the dominator tree |
| **Dominator tree** | A tree over the whole heap where each object's parent is its immediate dominator. Sorting its top level by retained heap is the first thing you do, every time |
| **Path to GC root** | The chain of references from a root to a given object. **This is the answer you are looking for** — it names the field, in the class, that must change |
| **Histogram** | Objects grouped by class, with counts and summed shallow heap. Useful for confirming a hypothesis; useless for forming one |
| **Unreachable / garbage** | Objects with no path from a root. MAT can show these if the dump includes them, but they are not your problem |

### Why the dominator tree is the right tool and the histogram is not

Consider one `HashMap` in a static field holding a million `Order` objects.

- **In the histogram:** the map itself is one object with a small shallow heap. Its
  `Node[]` table is one array. The million `Order` objects are a million entries under
  `com.orderflow.Order`. Their `String` fields are under `String` and `byte[]`. The
  histogram's top rows will be `byte[]`, `String`, `Object[]` and
  `java.util.HashMap$Node` — which is what the top rows of **every** Java heap histogram
  look like. You have learned nothing.
- **In the dominator tree:** the `HashMap` dominates its table, which dominates the nodes,
  which dominate the `Order` objects, which dominate their strings. So the map appears near
  the top with a **retained heap that includes all of it**. One row, one culprit, and
  "Path to GC Roots" names the static field.

**That is the whole reason MAT exists**, and the reason the mechanical statement singles it
out.

### What a heap dump is NOT

| It is not | Because |
|---|---|
| A profiler | It has no time dimension. It says what exists now, not where time goes (Topic 78) |
| An allocation record | It does not tell you where an object was allocated. JFR's `jdk.OldObjectSample` and async-profiler's `alloc` mode do |
| A view of native memory | Direct byte buffers, thread stacks, metaspace, code cache and native libraries are all outside it (Topic 80) |
| Cheap | Global safepoint plus a file the size of your live heap |
| A leak detector | It is a snapshot. **A leak is a trend**, and one snapshot cannot show a trend — which is why the multi-dump technique below exists |

---

## Why does it matter?

### Because "we're leaking memory, add more heap" is the default reaction and it is wrong

The mid-level response to a rising heap graph is to raise `-Xmx`. It appears to work — the
OOM moves from Tuesday to Friday — which is the worst possible outcome, because it
converts a diagnosable failure into a slow one that recurs after everyone has moved on.

If the cause is unbounded retention, more heap buys time proportional to the extra heap
and nothing else. Worse, it makes every full GC longer (**GC cost tracks the live set**,
Topic 70), so the service degrades before it dies.

The senior response is: *"Is the live set growing after a full GC, or is this just
allocation the collector hasn't got to yet? Those are different problems."* And that
question is answerable in one command.

### Because the ThreadLocal shape is systematically misdiagnosed

This is why the master plan assigns two drills for this topic.

The unbounded-cache leak is loud: memory climbs, OOM, dump, MAT, done. Painful but
tractable, and the failure is obvious enough that someone will investigate.

The `ThreadLocal` shape is quiet. Retention rises to pool-size × context-size and then
**flattens**. The heap graph looks like a healthy service that warmed up. Nobody
investigates a flat line. Meanwhile:

- your live set is permanently larger, so **every** young and old collection does more work
  (Topic 70);
- your available headroom is permanently smaller, so a traffic spike that used to be fine
  now OOMs;
- and if the context holds a per-request payload — an order with 200 lines, an uploaded
  file, a Postgres result set — the plateau is at a height nobody predicted.

Being the person who recognises a plateau as a leak is a genuine differentiator.

### Because the container OOM and the heap OOM are different failures

`orderflow` runs in a container with a 2 GiB memory limit (Topic 82). Two distinct deaths:

| Failure | What you see | What it means |
|---|---|---|
| `java.lang.OutOfMemoryError: Java heap space` | A Java exception, a stack trace, and — if you configured it — a heap dump | The **heap** could not satisfy an allocation. This topic |
| Container OOMKill (exit 137) | The process vanishes. **No Java exception. No heap dump.** `dmesg` shows the kernel's OOM killer | The **process RSS** exceeded the cgroup limit. Heap is only part of RSS (Topics 80, 82) |

**If the kernel killed you, `-XX:+HeapDumpOnOutOfMemoryError` never fires**, because the
JVM never got to throw anything. Engineers lose days to this. Knowing the difference on
sight, from the exit code, is worth stating in an interview.

### Because you need to be able to do this in fifteen minutes, under pressure

The mastery bar in the master plan is: *from `OutOfMemoryError` to a named retaining path in
under fifteen minutes.* That is a real bar, and it is achievable, because the procedure is
short and always the same:

1. Confirm it is heap retention, not allocation and not native (one command).
2. Get a dump (already configured, or `jcmd`).
3. Open the dominator tree, sort by retained heap.
4. Take the top row. Path to GC Roots, excluding weak/soft references.
5. Name the field. Fix the field.

Everything in this document exists to make those five steps automatic.

---

## Machine-level reality

Four mechanisms: the dump file's structure, how dominators are computed, how path-to-root
works, and the exact `ThreadLocal` retention chain.

### 1. The hprof format — what is actually in the file

A heap dump is a binary file in the **hprof** format. Knowing its shape removes the magic
and, more usefully, tells you what questions it can and cannot answer.

**Header, then a sequence of tagged records:**

```
[header: "JAVA PROFILE 1.0.2\0", identifier size (4 or 8), timestamp]
[record: tag=<n>, time-delta=<n>, length=<n>, body...]
[record: ...]
...
```

*illustration of the format, not captured output — structure only*

The record types that matter:

| Record | Contains | Why you care |
|---|---|---|
| `HPROF_UTF8` | An interned string with an id | Every class name and field name in the dump is one of these |
| `HPROF_LOAD_CLASS` | Class id, name id, class-loader id | **How MAT knows which class loader loaded what** — the basis of class-loader leak analysis |
| `HPROF_FRAME` / `HPROF_TRACE` | Stack frames and traces | Present only if allocation tracing was enabled; usually not |
| `HPROF_HEAP_DUMP_SEGMENT` | The heap itself, as sub-records | The bulk of the file |
| ↳ `GC_CLASS_DUMP` | Superclass, loader, instance size, static fields, field descriptors | **Static fields are here** — leak shape 1 lives in these records |
| ↳ `GC_INSTANCE_DUMP` | Object id, class id, and the raw field values | Every object |
| ↳ `GC_OBJECT_ARRAY_DUMP` | Element ids | Where `HashMap`'s `Node[]` and `ArrayList`'s `Object[]` live |
| ↳ `GC_PRIMITIVE_ARRAY_DUMP` | The raw bytes | `byte[]`, `char[]` — the ones that top every histogram |
| ↳ `GC_ROOT_*` | One record per root, tagged by kind | `JAVA_FRAME`, `JNI_GLOBAL`, `JNI_LOCAL`, `STICKY_CLASS`, `THREAD_OBJ`, `MONITOR_USED`, `THREAD_BLOCK`. **Every path-to-GC-root ends at one of these** |

Three consequences fall straight out:

1. **There is no time in the file.** It is one instant. A leak is a trend, so one dump
   cannot prove a leak — it can only show you a large retained set. That is why you take
   two or three (Proof 5).
2. **There are no allocation sites** unless allocation tracing was on, which it normally is
   not. The dump tells you *what holds* an object, never *who created* it. For creation
   sites use JFR's `jdk.OldObjectSample` or an allocation profile (Topic 78).
3. **Only the Java heap is in it.** Direct byte buffers appear as small `DirectByteBuffer`
   wrapper objects whose native memory is invisible; thread stacks, metaspace and the code
   cache are absent entirely. **A heap dump cannot explain an RSS-versus-heap gap** — that
   is Topic 80.

MAT's first act on opening a dump is to parse it and write a set of index files beside it.
That parse is expensive in time and memory and is why the headless parser exists — see
Hands-on Proof 3.

### 2. Dominator computation — what MAT is doing while it thinks

The object graph is a directed graph with multiple roots. MAT adds a synthetic super-root
pointing at every GC root, then computes the **dominator tree** of that graph.

**The definition again, precisely:** A dominates B if every path from the super-root to B
passes through A. B's **immediate dominator** is the closest such A, and that is B's parent
in the tree.

**The properties that make it the right tool:**

| Property | Consequence for you |
|---|---|
| Every object has exactly one immediate dominator | The dominator tree is a *tree*, so it can be displayed and sorted. The object graph itself is a tangle you cannot read |
| Retained set of A = the subtree rooted at A | Retained heap is a subtree sum, computable once for the whole heap |
| If B is reachable by two independent paths, its dominator is *above* both | **This is the subtlety that surprises people.** An object referenced by both the cache and a live request is not dominated by the cache. Removing the cache would not free it, and MAT correctly declines to attribute it to the cache |
| Computed with a Lengauer–Tarjan-style algorithm | Near-linear, but on a multi-gigabyte heap it is still minutes and gigabytes of working memory |

**The practical implication of the third row deserves emphasis**, because it explains a
result that looks wrong the first time you see it: if your leaking cache holds objects that
are *also* held elsewhere, the cache's retained heap will be **smaller than you expected**,
sometimes much smaller. That is not a bug. It is telling you something true and important:
freeing the cache alone would not free those objects. Use MAT's **"Path to GC Roots →
exclude weak/soft references"** and **"Merge Shortest Paths to GC Roots"** to find every
holder, not just the one you suspected.

**Reading a dominator-tree row.** The columns:

```
Class Name                                        | Objects | Shallow Heap | Retained Heap | Percentage
--------------------------------------------------+---------+--------------+---------------+-----------
<package>.<Class> @ 0x<address>                   |   <n>   |    <n>       |     <n>       |   xx.x%
  |- <field> <package>.<Class> @ 0x<address>      |   <n>   |    <n>       |     <n>       |   xx.x%
  |  |- <field> <package>.<Class> @ 0x<address>   |   <n>   |    <n>       |     <n>       |   xx.x%
```

*illustration of the format, not captured output — column structure only*

| Column | Meaning | How you use it |
|---|---|---|
| **Class Name** | The type, and — crucially — the **field name** through which the parent reaches it | The field names down the tree **are** the retaining path. Read them like a sentence |
| **Objects** | How many objects in this subtree | A large count with a small retained heap is many small objects; the reverse is a few big ones |
| **Shallow Heap** | This object's own bytes | Almost always tiny and almost always irrelevant |
| **Retained Heap** | **What would be freed if this went away** | **Sort by this. Always. First.** |
| **Percentage** | Retained heap as a fraction of the total | Anything above roughly a tenth of the heap in one row deserves an explanation |

**The reading procedure, every time:**

1. Sort the top level by **Retained Heap**, descending.
2. Look at the top three rows. If one row is a large fraction of the heap, that is your
   candidate.
3. **Expand downward** to see what it retains, and read the **field names** on the way — the
   chain of field names is the story.
4. Then go **upward**: right-click → **Path to GC Roots → exclude all phantom/weak/soft
   references**. That gives you the chain from a root down to your object, which is the
   thing you must break.
5. **Name the fix as a field**, not as a class: *"the `static final Map<String,Order>
   RECENT` field in `OrderIdempotencyCache`"*, not *"the order cache"*.

### 3. Path to GC roots — and why the exclusion filter matters

Path to GC Roots does a reverse search from your object through incoming references until
it reaches a root record.

**Why you exclude weak, soft and phantom references:** if the only path to your object is
through a `WeakReference`, then **the object is collectable** and it is not your leak. MAT
will happily show you weak paths, and they are noise — worse than noise, because they look
like an answer. The exclusion is not an optimisation; it is a correctness filter.

**The root kinds you will land on, and what each means:**

| Root kind | Means | Which leak shape |
|---|---|---|
| **System Class** / static field of a class loaded by the bootstrap or app loader | A `static` field is holding it | Shape 1 — unbounded static collection |
| **Thread** (the `Thread` object itself) | A live thread's own fields hold it — **including `threadLocals`** | **Shape 3 — the `ThreadLocal` leak** |
| **Java Local** (a stack frame variable) | A thread is currently executing a method with this on its stack | Usually not a leak — usually a request in flight. Unless a thread is stuck (Topic 94) |
| **JNI Global** | Native code holds a global reference | Native library or agent. Rare and nasty |
| **Busy Monitor** / **Thread Block** | Held for synchronisation | Usually transient |
| **Native Stack** | JVM internals | Usually transient |

**When Path to GC Roots lands on "Thread", stop and think about `ThreadLocal` immediately.**
That is the single highest-value pattern-recognition move in this entire topic.

### 4. The `ThreadLocal` retention chain, exactly

This is the mechanism behind drill (b), and it must be exact, because the *almost*-correct
version of this explanation is what causes the misdiagnosis.

**Read the JDK source rather than trusting me:**

```bash
# The sources jar ships with the JDK.
cd /tmp && unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/lang/ThreadLocal.java' > /dev/null
sed -n '/static class ThreadLocalMap/,/^    }/p' /tmp/java.base/java/lang/ThreadLocal.java | head -60

# And the field on Thread that holds the map:
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/lang/Thread.java' > /dev/null
grep -n "threadLocals\|inheritableThreadLocals" /tmp/java.base/java/lang/Thread.java | head
```

**WHAT TO LOOK FOR in `ThreadLocal.java`:** the declaration of the inner `Entry` class. It
extends `WeakReference<ThreadLocal<?>>` and has a plain field named `value`.

**The chain, link by link:**

```
GC Root: Thread (a live pool thread — a root because it is a live thread)
   |
   +-- field  Thread.threadLocals            -> ThreadLocal.ThreadLocalMap
                |
                +-- field  ThreadLocalMap.table -> Entry[]
                              |
                              +-- Entry[i]  (extends WeakReference<ThreadLocal<?>>)
                                    |
                                    +-- referent (WEAK)   -> the ThreadLocal<?> key
                                    +-- field value (STRONG) -> YOUR OBJECT
```

*illustration of the reference structure, not captured output*

**The four facts that make this a leak, and each one matters:**

1. **The key is weak, the value is strong.** If the `ThreadLocal` object itself becomes
   unreachable, the key is cleared — but **the value is not**, because `Entry.value` is a
   plain strong field.
2. **The `Thread` is a GC root.** Not "reachable from a root" — *is* a root. A pool thread
   is a root for the entire life of the process.
3. **Stale entries (cleared key, live value) are only cleaned opportunistically.** The map
   expunges them during some subsequent `get`, `set` or `remove` when hashing happens to
   land near a stale slot. **There is no background cleaner and no guarantee.** If the
   thread never touches that `ThreadLocal` again, the value is held forever.
4. **Therefore the correct fix is `remove()` in a `finally` block**, on the thread that set
   it, before the thread returns to the pool. Not `set(null)` — that leaves a live entry
   with a null value, which is better but still an entry. `remove()`.

**And the property that causes the misdiagnosis:**

> Retention is **bounded** at (number of pool threads) × (size of one context object).
>
> Once every thread in the pool has served at least one request, every thread has one
> entry. Serving a million more requests **overwrites** each thread's value rather than
> adding to it. The graph rises during warm-up and then goes flat.
>
> **A flat line is what "healthy" looks like. That is why this ships.**

It is still a leak. The live set is permanently inflated by that product, and Topic 70's
central fact is that GC cost is a function of the live set, not of garbage. And the height
of the plateau is a product with a variable second term: if the context holds a per-request
object whose size depends on the request — an order with 200 lines, a result set, an
uploaded file — then the plateau's height is set by the **largest** request each thread has
ever served. Which is unbounded in practice, and only appears under an unusual load pattern,
months later, at 02:00.

### 5. What a heap dump costs, mechanically

- **It requires a global safepoint** (Topic 73). Every application thread stops for the
  duration. The duration scales with the number of objects and the heap size.
- **It writes a file roughly the size of the live heap.** An 8 GB heap writes something on
  that order. Check your disk *before* you trigger it.
- **`GC.heap_dump` performs a full GC first by default**, so the dump contains only
  reachable objects. That is usually what you want (a clean live set) and occasionally not
  (when you want to see what is garbage). `-all` skips the GC.
- **In Kubernetes, the stop-the-world will fail your liveness probe** if the probe's timeout
  is shorter than the dump. The pod gets killed mid-dump and you get a truncated file and a
  restart (Topic 121). Raise the timeout, or take the pod out of rotation, before dumping.

---

## Example 1 — minimal

**The goal:** produce, observe and diagnose a leak in under ten minutes, with code small
enough that there is nothing to argue about.

### The program

```java
package com.orderflow.demo;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * The most common leak in Java: an unbounded cache reachable from a static field.
 * Nothing here is exotic. Someone added a "small" cache and never bounded it.
 */
public final class UnboundedCacheDemo {

    /** GC root: a static field of a loaded class. Lives for the process lifetime. */
    private static final Map<String, byte[]> RECENT_ORDER_PAYLOADS = new HashMap<>();

    public static void main(String[] args) throws InterruptedException {
        int i = 0;
        while (true) {
            String orderId = UUID.randomUUID().toString();
            // Stands in for a serialised order payload.
            RECENT_ORDER_PAYLOADS.put(orderId, new byte[16 * 1024]);
            if (++i % 500 == 0) {
                System.out.println("entries=" + RECENT_ORDER_PAYLOADS.size());
                Thread.sleep(50);
            }
        }
    }
}
```

### Run it so that it dies usefully

```bash
javac -d out UnboundedCacheDemo.java
mkdir -p /tmp/dumps

java -Xmx256m \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/dumps \
     -Xlog:gc*:file=/tmp/gc.log:time,uptime,level,tags \
     -cp out com.orderflow.demo.UnboundedCacheDemo
```

**Every flag earns its place:**

| Flag | Why |
|---|---|
| `-Xmx256m` | Small heap so it dies in seconds instead of minutes. **A small heap is a diagnostic tool** — it converts a slow leak into a fast one |
| `-XX:+HeapDumpOnOutOfMemoryError` | Writes the dump automatically at the moment of failure. **This should be on in every environment you own** |
| `-XX:HeapDumpPath=/tmp/dumps` | A directory you control. In a container this **must** be a mounted volume or the dump dies with the container |
| `-Xlog:gc*` | So you can see the live set failing to shrink, which is the evidence that this is retention rather than allocation |

### WHAT TO LOOK FOR, before you open any tool

**First, in the GC log**, which is where the diagnosis actually happens:

```bash
grep -E "Pause Full|Pause Young" /tmp/gc.log | tail -20
```

The critical reading: **the heap occupancy AFTER each collection.** Every GC log line
reports before-and-after occupancy.

| What you see | What it means |
|---|---|
| Post-collection occupancy climbing steadily, collection after collection | **Retention.** The live set is genuinely growing. This is a leak |
| Post-collection occupancy returning to a stable floor, while pre-collection peaks are high | **Allocation, not retention.** You are producing garbage quickly, which is a different problem (Topics 68, 70, 71) and is not solved by hunting for a leak |
| Full GCs becoming more frequent and longer near the end | Classic pre-OOM death spiral: the collector runs constantly, reclaims almost nothing, and burns CPU |
| `Pause Full` appearing at all under G1 | G1 avoids full GCs; seeing them means it is losing. Strong retention signal |

**This single distinction — post-collection floor rising versus flat — is the most valuable
diagnostic in this document**, because it takes one command and separates "leak" from "GC
tuning" before you spend an hour in MAT.

**Second, the failure itself:**

| What you see | What it means |
|---|---|
| `java.lang.OutOfMemoryError: Java heap space` + a message naming the dump file | Working as intended. Go to MAT |
| `OutOfMemoryError: GC overhead limit exceeded` | The JVM spent most of its time in GC while reclaiming very little. Same root cause, earlier detection |
| The process vanishes with exit code 137 and no exception | **Container OOMKill, not heap OOM.** No dump. The problem may not be the heap at all (Topics 80, 82) |
| `OutOfMemoryError: unable to create native thread` | A thread leak, not an object leak (Topic 98). Different investigation entirely |
| `OutOfMemoryError: Metaspace` | Class-metadata leak — usually a class loader retained, often from repeated dynamic proxy or scripting-engine creation (Topics 40, 67) |
| No dump file despite the flag | Wrong `HeapDumpPath`, no permission, no disk space, or the container died first. **Verify the path is writable before you need it** |

### Open it in MAT

```bash
# GUI: File -> Open Heap Dump -> select the .hprof
# The "Leak Suspects" report is offered on open. Run it - it is a good first pass -
# but do NOT stop there. Go to the dominator tree yourself.
```

**WHAT TO LOOK FOR, in this exact order:**

1. **Overview → Dominator Tree.** Sort by **Retained Heap**, descending.
2. The top row. **Expand it downward** and read the field names.
3. Right-click the top row → **Path to GC Roots → exclude all phantom/weak/soft etc.**
4. Read the root kind at the end of the path.

| What you see | What it means |
|---|---|
| A `java.util.HashMap` at the top with almost the whole heap retained | The expected result. Expand: `HashMap` → `table` (`Node[]`) → nodes → `byte[]` |
| Path to GC Roots ends at **System Class `com.orderflow.demo.UnboundedCacheDemo`** | **The answer.** A static field of that class. Leak shape 1, and now you can name the field |
| The top row is a `byte[]` rather than the map | You are looking at the histogram, not the dominator tree. Switch views |
| The top row is a `Thread` | Leak shape 3. Expand `threadLocals` — this is drill (b)'s shape, and recognising it here is exactly the transferable skill |
| Several rows each with a moderate share, none dominant | Either the retention is spread across many roots, or the objects are multiply-referenced so no single dominator claims them. Use **Merge Shortest Paths to GC Roots** on the *class* rather than one instance |
| Leak Suspects names something you did not expect | Read its reasoning, then verify it in the dominator tree yourself. It is a heuristic, and heuristics are wrong sometimes |

### The sentence you must be able to say

> *"The heap is dominated by a single `HashMap` retained through the static field
> `RECENT_ORDER_PAYLOADS` on `UnboundedCacheDemo`. Every entry ever inserted is still
> reachable because nothing removes them and the class is a GC root for the process
> lifetime. The fix is a bounded cache with an eviction policy, not more heap."*

**Name the field. Not the class, not "the cache" — the field.** That precision is the whole
skill, and it is what makes the fix reviewable.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, **5M order lines** |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap for this investigation | **`-Xmx512m`** — deliberately reduced from the 1200m baseline so the leak manifests in minutes rather than days |
| Collector | G1 |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Dumps | `/dumps`, a mounted volume that survives the container |
| Baseline artefacts | `/docs/java/baselines/<date>-run-01/` |

**Why reduce the heap deliberately.** A leak that takes three weeks to OOM in production
takes twenty minutes at `-Xmx512m` under the Topic 65 load. Shrinking the heap is a
legitimate and under-used diagnostic technique: it changes nothing about the *cause* and
compresses the timeline by the ratio of the heaps. Record that you did it, because the
absolute numbers in the dump are then not production numbers.

### The change that causes it — and it is a reasonable change

`POST /orders` must be idempotent: a client retry must not create a second order. Someone
implements it the obvious way.

```java
package com.orderflow.orders;

import org.springframework.stereotype.Component;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Idempotency guard for POST /orders.
 *
 * Looks fine in review. Passes every test. Has no eviction, and is reachable
 * from a Spring singleton, which is reachable from the ApplicationContext,
 * which is reachable from a static field. It is a GC root for the process lifetime.
 */
@Component
public class OrderIdempotencyCache {

    private final Map<String, Order> byIdempotencyKey = new ConcurrentHashMap<>();

    public Order findExisting(String idempotencyKey) {
        return byIdempotencyKey.get(idempotencyKey);
    }

    public void record(String idempotencyKey, Order order) {
        byIdempotencyKey.put(idempotencyKey, order);      // nothing ever removes
    }
}
```

Three things make this realistic and worth dwelling on:

1. **It is not a `static` field.** It is an instance field of a Spring singleton. The
   retaining path runs through the `ApplicationContext` to a static field somewhere, and
   MAT will show you every link. Engineers who have only ever seen the `static` example do
   not recognise this variant.
2. **The retained value is an `Order` entity** (Topic 48). If it is still attached to a
   persistence context, or holds a lazy `OrderLine` collection proxy (Topic 49), the
   retained set per entry is far larger than "an order". **The dominator tree will show you
   exactly how much larger**, which is the finding.
3. **It passes every test.** Tests insert a handful of keys. The leak is a function of
   *distinct keys over time*, which only load produces.

### Step 1 — configure the JVM so the failure is diagnosable

```yaml
# docker-compose.yml — the load profile (Topic 43)
services:
  orderflow:
    environment:
      JAVA_TOOL_OPTIONS: >-
        -Xms512m -Xmx512m
        -XX:+UseG1GC
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/dumps
        -XX:+ExitOnOutOfMemoryError
        -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags
    volumes:
      - ./dumps:/dumps
      - ./logs:/logs
    deploy:
      resources:
        limits:
          memory: 2g
          cpus: "2"
```

Two flags need justification:

- **`-XX:HeapDumpPath=/dumps` must point at a mounted volume.** Otherwise the dump is
  written into the container's writable layer and vanishes when the container is replaced.
  This is the most common way teams lose the evidence.
- **`-XX:+ExitOnOutOfMemoryError`** makes the JVM die immediately instead of limping along
  in a degraded state throwing OOM from random threads. In a Kubernetes deployment that is
  what you want: fail fast, restart, and preserve the dump. **Do not add it without
  checking your `HeapDumpPath` is writable**, or you exit without evidence.

**Verify before you run the load**, because discovering the path is wrong at the moment of
failure is a wasted afternoon:

```bash
docker exec orderflow sh -c 'touch /dumps/.writetest && ls -la /dumps && rm /dumps/.writetest'
docker exec orderflow df -h /dumps        # need room for roughly the live heap
docker exec orderflow jcmd 1 VM.flags -all | grep -E "HeapDump|MaxHeapSize"
```

### Step 2 — run the load and watch the right number

```bash
k6 run loadtest/baseline.js &

# The wrong thing to watch: current heap usage. It goes up and down; that is GC working.
# The RIGHT thing to watch: heap occupancy AFTER each collection.
tail -f /logs/gc.log | grep -E "Pause Young|Pause Full"

# And the container's view, which is a different number (Topic 82):
watch -n 5 'docker stats --no-stream orderflow'
```

**WHAT TO LOOK FOR — the four-way diagnosis, before you dump anything:**

| What you see | What it means | Where to go |
|---|---|---|
| Post-collection occupancy rising monotonically | **Retention.** Live set growing | This topic. Take the dump |
| Post-collection occupancy flat, pre-collection peaks high, GCs frequent | Allocation rate, not retention | Topics 68, 70, 71 — and an allocation profile (Topic 78) |
| Post-collection occupancy flat, but at a much higher floor than before the change | **The `ThreadLocal` plateau shape.** A bounded leak | Drill (b). Still a leak. Still worth fixing |
| Heap flat, but container RSS climbing until OOMKill | **Not a heap problem.** Direct buffers, metaspace, thread stacks, native libs | Topic 80 — NMT. A heap dump will not help you |

That table is the fifteen-minute mastery bar, and most of it is answered before MAT opens.

### Step 3 — get the dump

If it OOMed, the dump is already there. If you want one before it dies — and you often do,
because a dump taken at 80% full is easier to read than one taken at the moment of
collapse:

```bash
docker exec orderflow jcmd 1 GC.heap_dump -all=false /dumps/orderflow-live.hprof
ls -lh dumps/
```

| Option | Effect | When |
|---|---|---|
| `-all=false` (the default) | Full GC first; dump contains only **reachable** objects | **Almost always.** A clean live set is what you want to analyse |
| `-all=true` | No preceding GC; includes unreachable objects too | Only when you specifically want to see what is garbage |

**Two operational warnings before you run this in an environment that matters:**

1. It is a **global safepoint** for the duration. Every request in flight stalls. On a
   Kubernetes pod, if the liveness probe times out during the dump, the pod is killed and
   you get a truncated file (Topic 121). Take the pod out of rotation, or raise the probe
   timeout, first.
2. It writes a file about the size of the live heap. `df -h` first.

### Step 4 — the MAT session, as a procedure

```bash
# Headless parse first. This is the professional workflow: the parse is the expensive part,
# it needs a lot of memory, and doing it on the machine that owns the dump avoids
# transferring a multi-gigabyte file to your laptop.
/opt/mat/ParseHeapDump.sh /dumps/orderflow-live.hprof \
    org.eclipse.mat.api:suspects \
    org.eclipse.mat.api:overview \
    org.eclipse.mat.api:top_components

ls -la /dumps/       # index files and zipped HTML reports appear beside the dump
```

> **Flagged.** MAT's headless script name (`ParseHeapDump.sh`), its report identifiers, and
> the way you raise its own heap in `MemoryAnalyzer.ini` vary by MAT version. **Check your
> installation** — `ls /opt/mat` and read the `MemoryAnalyzer.ini` — rather than copying a
> path from this document. The **workflow** (parse headlessly, then open the pre-built
> indexes in the GUI) is stable; the invocation is not.

**MAT needs more heap than you think.** It is a Java application analysing a Java heap. Give
it more than the dump size in `MemoryAnalyzer.ini`, or it will OOM analysing your OOM,
which is a rite of passage but a waste of an hour.

**Then, in the GUI, in this order — and it is the same order every time:**

1. **Dominator Tree**, sorted by Retained Heap, descending.
2. Top row. **Expand downward.** Read the field names.
3. **Path to GC Roots → exclude all phantom/weak/soft etc.** Read the root kind.
4. If several instances of one class share the blame: right-click the class in the
   histogram → **Merge Shortest Paths to GC Roots**. This aggregates the paths for all
   instances and is how you find "many objects, one holder".
5. **Only now** look at the histogram, to confirm what the material is.

**WHAT TO LOOK FOR:**

| What you see | What it means | What you do |
|---|---|---|
| A `ConcurrentHashMap` at the top; expanding shows `table` → `Node[]` → `Node` → `Order` | The expected result | Path to GC Roots. Expect: `ConcurrentHashMap` ← `byIdempotencyKey` ← `OrderIdempotencyCache` ← the Spring bean registry ← a static field. **Name the field in the write-up** |
| The retained heap per `Order` is far larger than an `Order`'s fields | The entity drags a graph: `OrderLine` collections, a `Product` reference, possibly a Hibernate proxy holding a session (Topics 48, 49) | **This is a second, independent finding.** Even a bounded cache of entities is a bad idea; cache an immutable projection instead (Topic 47) |
| The top row is a Hibernate `StatefulPersistenceContext` or `SessionImpl` | A persistence context held open — often `open-session-in-view` (Topic 49) or a long-lived `EntityManager` | Different fix entirely. Follow the path to find who holds the session |
| The top row is `java.lang.Thread` | **Leak shape 3.** Expand `threadLocals` → `table` → `Entry` → `value` | That is drill (b). Read on |
| The top row is a `ClassLoader` | Class-loader retention: every class it loaded plus every static field of each. Usually repeated dynamic class generation or a redeploy | Topics 40, 67, 81 |
| No single row dominates; the heap is broad and shallow | Possibly not a leak at all. Re-check the GC log: is the post-collection floor actually rising? | You may be looking at allocation, not retention |
| Retained heap of the suspected cache is **smaller** than you expected | The objects are also referenced elsewhere, so the cache does not dominate them | Correct MAT behaviour, and a real finding. Use **Merge Shortest Paths** to find every holder |

### Step 5 — the fix, and why each option is or is not right

| Option | Verdict |
|---|---|
| Raise `-Xmx` | **No.** Delays the OOM proportionally, makes every full GC longer, and leaves the defect. This is Trap 1 |
| `Map` → `WeakHashMap` | **No.** Weak *keys* only, and the keys here are `String` idempotency keys that may be interned or held by the values. Fragile, and unpredictable eviction timing is a correctness hazard for an idempotency guard |
| Bounded `LinkedHashMap` with `removeEldestEntry` (Topic 15) | **Works, minimally.** Not thread-safe without external synchronisation, which matters here |
| **Caffeine with `maximumSize` + `expireAfterWrite`** | **Yes.** Idempotency has a natural TTL — the client's retry window. Bounded, concurrent, with eviction stats you can put on a dashboard (Topics 15, 110, 118) |
| Redis with a TTL | **Yes, and better across instances.** The bug is worse than a leak: an in-JVM idempotency cache does not work at all behind a load balancer, because the retry may hit a different pod (Topic 110) |
| A unique constraint on `(idempotency_key)` in Postgres | **Best.** Correct by construction, survives restarts, works across instances. The cache becomes an optimisation rather than the mechanism (Topics 52, 115) |

**The senior observation to make in the review:** the leak surfaced a *correctness* bug. An
in-memory idempotency cache is wrong on a multi-instance deployment regardless of its size.
The heap dump found a design defect, not just a memory defect — and saying so is worth more
than the fix.

### Step 6 — prove the fix, do not assert it

```bash
# 1. Re-run the same load with the same reduced heap.
k6 run loadtest/baseline.js &
tail -f /logs/gc.log | grep "Pause"

# 2. The proof: post-collection occupancy reaches a plateau and stays there.
# 3. Then restore the baseline heap and re-run the Topic 65 baseline in full.
# 4. Compare every percentile against /docs/java/baselines/ - within +/-10%.
# 5. Take a heap dump at steady state and confirm the cache's retained heap is bounded.
docker exec orderflow jcmd 1 GC.heap_dump /dumps/orderflow-after-fix.hprof
```

**WHAT TO LOOK FOR:** the post-collection floor flat over a long run, and — the specific
confirmation — the cache's retained heap in the new dump proportional to its configured
maximum size rather than to elapsed time.

---

## Wrong approach → exact symptom → root cause → fix

Five traps.

---

### Trap 1 — treating an OOM as "we need more heap" when it is retention

**Wrong approach.** `OutOfMemoryError` in production. Someone raises `-Xmx` from 1200m to
1800m and closes the ticket.

**Exact symptom — three observations, and the third is decisive:**

1. **The OOM recurs, later, by roughly the ratio of the heaps.** 50% more heap buys ~50%
   more time. That proportionality is the signature of a linear leak.
2. **Full GCs get longer and more frequent before each failure.** GC cost tracks the live
   set (Topic 70); a growing live set means every collection does more work.
3. **The decisive one — post-collection heap occupancy rises monotonically:**

   ```bash
   grep -E "Pause Young|Pause Full" /logs/gc.log | tail -50
   ```

   Every line reports occupancy before and after. **Read the "after".** If the floor after
   each collection rises steadily over hours, the live set is growing and no amount of heap
   will fix it.

**Root cause.** Heap size determines *when* you run out, not *whether*. Unbounded retention
has no equilibrium. Raising `-Xmx` moves the failure and makes every collection more
expensive on the way there — so you degrade for longer before you die.

**The one case where more heap IS the right answer**, and you must be able to distinguish
it: when the live set is **stable but genuinely larger than the heap you provisioned**. A
cache sized for the dataset, a bigger dataset, more concurrent requests each holding a
result set. Signature: post-collection floor is **flat**, but close to `-Xmx`, so the
collector runs constantly. That is a capacity problem, and more heap is the fix (Topics 70,
82, 129).

| What you see in the GC log | Diagnosis | Fix |
|---|---|---|
| Post-collection floor rising over hours | **Leak.** Retention | Find the retaining path |
| Post-collection floor flat and low; frequent young GCs | High allocation rate | Reduce allocation (Topics 68, 78) or tune young gen |
| Post-collection floor flat and near `-Xmx` | Live set genuinely too big for the heap | **More heap, legitimately.** Or shrink the live set |
| Floor flat but at a new, higher level since a release | **The bounded-leak plateau.** Drill (b) | Still a leak. Find it |

**Fix.** Answer the "is the floor rising" question **before** touching `-Xmx`. It is one
grep. Then dump, dominate, name the path.

---

### Trap 2 — reading the histogram and concluding `byte[]` is the problem

**Wrong approach.** Open the dump (or run `jcmd GC.class_histogram`), see the top rows, and
report "we have a huge number of `byte[]` and `String` objects".

**Exact symptom.** Your top rows are, in some order: `byte[]`, `java.lang.String`,
`java.lang.Object[]`, `java.util.HashMap$Node`, `java.lang.Class`. The decisive tell:
**this is what the top of every Java heap histogram looks like, on every application, always.**
If your "finding" would be true of any Java program, it is not a finding.

Second tell: **the histogram has no field names in it**, so it cannot possibly tell you who
holds anything. It is a materials list, not a suspect list.

**Root cause.** A histogram groups by class and sums **shallow** heap. Shallow heap
attributes the bytes to the leaf objects — the arrays that actually hold the data — and
never to the structure that retains them. The `HashMap` doing the retaining has a shallow
heap of a few dozen bytes and will not appear anywhere near the top.

**Prove the difference to yourself — this is Proof 4:**

```bash
# Histogram: what the material is.
jcmd <pid> GC.class_histogram | head -20
```

The column structure:

```
 num     #instances         #bytes  class name (module)
-------------------------------------------------------
   1:          <n>            <n>   [B (java.base@<ver>)
   2:          <n>            <n>   java.lang.String (java.base@<ver>)
   3:          <n>            <n>   [Ljava.lang.Object; (java.base@<ver>)
```

*illustration of the format, not captured output — column structure only*

Then open the same heap in MAT's dominator tree. **Different question, different answer.**

| What you see | What it means |
|---|---|
| Histogram top rows are the usual JDK types | Expected and uninformative. Move to the dominator tree |
| Histogram shows an unexpected **application** class with a huge instance count | **Now the histogram is useful.** Right-click it → Merge Shortest Paths to GC Roots to find the holder |
| Histogram instance counts growing between two snapshots for one class | A genuinely good use of histograms: **the delta**, not the absolute (Proof 5) |
| Dominator tree names a structure the histogram never mentioned | Normal, and the point of the exercise |

**Fix.** Use the histogram for two things only: (1) the **delta** between two snapshots, to
see what is growing; (2) confirming a hypothesis you formed in the dominator tree. Form the
hypothesis in the dominator tree.

---

### Trap 3 — the `ThreadLocal` plateau, closed as "not a leak"

**Wrong approach.** The heap graph rose after a release and then flattened. Someone says
"it stabilised, it's just warm-up", and closes the ticket. This is the misdiagnosis this
topic exists to prevent.

**Exact symptom.** Four observations:

1. **Heap-after-GC rises for a period, then goes flat and stays flat.** No OOM. Ever.
2. **The plateau's height is suspiciously proportional to your thread-pool size.** Halve
   `server.tomcat.threads.max` and the plateau halves. **That is the diagnostic** and it is
   almost conclusive on its own.
3. **The plateau appeared with a specific release.** Compare against the retained baselines.
4. In a dump, **Path to GC Roots ends at `java.lang.Thread`**, through `threadLocals`.

**Root cause.** Exactly the chain from Machine-level reality §4. Each pool thread's
`ThreadLocalMap` holds one strong reference to one context object. Once every thread has
served one request, the retention is complete: further requests **overwrite** rather than
accumulate. Retention = pool size × context size, and then it stops.

**Why it is still a leak, and you must be able to argue this:**

- Your live set is permanently inflated by that product. **GC cost is a function of the
  live set** (Topic 70), so every young and old collection now does more work — forever.
- Your headroom is permanently smaller. The traffic spike that used to fit now does not.
- **The plateau height is not actually a constant.** It is set by the largest context each
  thread has ever held. If the context references a per-request payload — a 200-line order,
  a result set, an uploaded file — the plateau is set by the worst request each thread has
  ever seen. That is unbounded in practice and it materialises months later under an unusual
  load pattern.
- If the value transitively holds a `ClassLoader` (a common shape in application servers),
  you have a class-loader leak wearing a plateau's clothing.

**Prove it — the pool-size experiment, and it takes one restart:**

```bash
# Baseline plateau at the normal pool size.
docker exec orderflow jcmd 1 GC.heap_dump /dumps/pool-200.hprof

# Halve the pool and re-run the identical load.
# application-load.yml: server.tomcat.threads.max: 100
docker exec orderflow jcmd 1 GC.heap_dump /dumps/pool-100.hprof
```

| What you see | What it means |
|---|---|
| The plateau halves when you halve the pool | **Confirmed `ThreadLocal`-per-thread retention.** Nothing else scales with pool size like this |
| The plateau is unchanged | Not per-thread. Look for a shared structure instead |
| The plateau more than halves | Something else scales with threads too — per-thread buffers, or a `ThreadLocal` holding something whose size depends on concurrency |
| Aggregate `Thread` retained heap is a large share of the heap in MAT | Look at `threadLocals` on any one thread and read the `Entry` values |

**Fix.**

```java
public Response handle(HttpServletRequest request) {
    RequestContext.set(buildContext(request));
    try {
        return process(request);
    } finally {
        RequestContext.clear();      // ThreadLocal.remove() - NOT set(null)
    }
}
```

`remove()`, not `set(null)`: `set(null)` leaves a live entry whose value is null, which is
better but still an entry and still a live map slot. `remove()` clears both.

Better still: put the cleanup in a servlet `Filter` or a Spring interceptor so it cannot be
forgotten per handler — this is exactly why `SecurityContextPersistenceFilter` clears the
`SecurityContextHolder` in a `finally` (Topic 56). And best: use a framework-provided
request-scoped mechanism (Topic 38) or `ScopedValue` (Topic 102) rather than raw
`ThreadLocal`, so the lifetime is structural rather than a convention.

---

### Trap 4 — taking a heap dump in production without understanding what it costs

**Wrong approach.** Production is misbehaving. Someone runs `jcmd <pid> GC.heap_dump
/tmp/dump.hprof` on the live pod, during peak.

**Exact symptom — pick the one you get:**

1. **Every request times out for several seconds.** The dump is a global safepoint (Topic
   73), and the log shows it:

   ```bash
   grep -i "heap.*dump\|HeapDumper\|HeapWalk" /logs/safepoint.log
   ```

2. **The pod is killed mid-dump and restarts.** The liveness probe could not be served
   during the stop-the-world, exceeded its failure threshold, and Kubernetes killed the pod
   (Topic 121). You get a truncated, unparseable file and an unplanned restart.
3. **The disk fills.** `/tmp` had less free space than the live heap. Now the JVM cannot
   write logs either.
4. **The dump is written and then lost**, because `/tmp` is the container's writable layer
   and the container was replaced.
5. **MAT cannot open it** because your laptop has less RAM than the dump.

**Root cause.** A heap dump is a stop-the-world walk of the entire object graph plus a
write of a file the size of the live heap. It is a heavyweight operation with an operational
cost, and none of the tooling warns you.

**Fix — the checklist, every time:**

```bash
# 1. How big will it be? Roughly the live heap.
docker exec orderflow jcmd 1 GC.heap_info

# 2. Is there room, on a volume that survives the container?
docker exec orderflow df -h /dumps

# 3. Take the instance out of rotation first, or raise the probe timeouts.
kubectl label pod orderflow-abc123 app-  # remove from the Service selector

# 4. Then dump, to the mounted path.
docker exec orderflow jcmd 1 GC.heap_dump /dumps/orderflow-$(date +%s).hprof

# 5. Parse headlessly ON THE HOST. Do not copy gigabytes to a laptop.
/opt/mat/ParseHeapDump.sh /dumps/orderflow-*.hprof org.eclipse.mat.api:suspects
```

**And the better answer for production, which is to arrange the dump in advance:**

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps            # a mounted volume, verified writable
-XX:+ExitOnOutOfMemoryError        # fail fast, restart clean, keep the evidence
```

Plus JFR running continuously with `jdk.OldObjectSample` enabled, which gives you
leak-candidate objects **with their allocation stack traces** at a fraction of the cost —
see Measurement.

---

### Trap 5 — chasing a heap leak when the heap is not the problem

**Wrong approach.** The container gets OOMKilled repeatedly. Someone takes heap dumps and
finds nothing unusual.

**Exact symptom.** The decisive combination:

- The process **disappears with exit code 137**. No `OutOfMemoryError`. No stack trace. **No
  heap dump**, despite `HeapDumpOnOutOfMemoryError` being set.
- `docker inspect <container> --format '{{.State.OOMKilled}}'` returns `true`.
- The heap graph is **flat and healthy** right up to the death.
- Container RSS climbs steadily while heap does not.

```bash
docker inspect orderflow --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
dmesg | grep -i -E "killed process|oom"
docker exec orderflow cat /sys/fs/cgroup/memory.current
docker exec orderflow jcmd 1 GC.heap_info
```

**Root cause.** RSS is heap **plus** metaspace, code cache, thread stacks, GC data
structures, direct byte buffers, mapped files, and any native library's allocations. The
kernel's OOM killer sees RSS. The JVM's `OutOfMemoryError` sees the heap. **They are
different numbers and they fail differently.** `HeapDumpOnOutOfMemoryError` cannot fire
because the JVM never threw anything — it was killed.

This is Topic 80's territory and Topic 82's, and confusing it with a heap leak costs days.

**Fix — attribute the footprint before you dump anything:**

```bash
# Turn on Native Memory Tracking, then diff over time.
-XX:NativeMemoryTracking=summary

docker exec orderflow jcmd 1 VM.native_memory baseline
# ...wait under load...
docker exec orderflow jcmd 1 VM.native_memory summary.diff
```

**WHAT TO LOOK FOR** in the diff: which category grew.

| Category growing | Meaning | Topic |
|---|---|---|
| Java Heap | Actually a heap problem after all | This topic |
| Class / Metaspace | Class-loader leak — repeated proxy or script class generation | 40, 67, 81 |
| Thread | Thread leak: stacks are ~1 MB each by default | 90, 98 |
| Code | Code cache — often an agent generating many classes | 74, 81 |
| Internal / Other | Direct byte buffers, NIO, Netty | **80** |
| GC | Collector data structures; grows with heap size | 71, 72 |
| Nothing in NMT grows, RSS still does | Native library outside the JVM's accounting, or glibc arena fragmentation | 80, 82 |

**The rule:** *before you open a heap dump, confirm the heap is the thing that grew.* One
`GC.heap_info` and one `docker stats` answer it.

---

## Hands-on proof

### Setup — verify your tools. Do not take versions from this document.

```bash
java -version
jcmd -l
which jmap jcmd jhsdb jfr

ls /opt/mat                              # MAT installation
cat /opt/mat/MemoryAnalyzer.ini          # its own -Xmx: MUST exceed the dump size
/opt/mat/ParseHeapDump.sh --help 2>&1 | head
```

> **I will not name a MAT version or download URL.** Script names and report identifiers
> vary. `ls` and `--help` on your installation are the authority.

### Proof 1 — the four ways to get a dump, and when each applies

```bash
# a) Automatic on heap OOM. Configure this everywhere, always.
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps

# b) On demand, live. Full GC first, reachable objects only.
jcmd <pid> GC.heap_dump /dumps/live.hprof

# c) On demand, including unreachable objects.
jcmd <pid> GC.heap_dump -all=true /dumps/all.hprof

# d) From a core dump, when the process is already gone.
jhsdb jmap --binaryheap --dumpfile /dumps/from-core.hprof --core /cores/core.1234 --exe $(which java)
```

**WHAT TO LOOK FOR:** that (a) is configured in every environment you own, and that its
path is a mounted volume you have verified is writable. Everything else is a fallback.

### Proof 2 — histogram without a dump, which is often enough

```bash
jcmd <pid> GC.class_histogram | head -30
```

This runs a histogram in-process. It is far cheaper than a dump — still a safepoint, but no
file and no full graph walk.

**WHAT TO LOOK FOR:** an **application** class with an implausible instance count. If
`com.orderflow.Order` has orders of magnitude more instances than could plausibly be in
flight, you have a candidate without ever writing a dump.

### Proof 3 — parse headlessly, then open the indexes

```bash
/opt/mat/ParseHeapDump.sh /dumps/orderflow.hprof \
    org.eclipse.mat.api:suspects org.eclipse.mat.api:overview
ls -la /dumps/     # .index files and zipped HTML reports
```

**WHAT TO LOOK FOR:** the index files. Opening the dump in the GUI afterwards is fast
because the expensive work is done. This is how you analyse a dump that is larger than your
laptop.

### Proof 4 — the same heap, two views, two different answers

Open one dump. Look at the **histogram**, write down the top five. Look at the **dominator
tree**, write down the top five. **They will not be the same list.**

| What you see | What it means |
|---|---|
| Histogram: `byte[]`, `String`, `Object[]`. Dominator tree: a named application structure | The expected and instructive result. **This is why the dominator tree exists** |
| Both name the same thing | Possible when one huge array is the leak — an oversized buffer, or a `byte[]` from a file read |
| Dominator tree top row is a class loader | Class-loader retention. The histogram would never have told you |

### Proof 5 — two dumps, and the delta (the technique that actually finds leaks)

**One dump shows a large object. Two dumps show what is GROWING.** Growth is the leak
signal; size is not.

```bash
docker exec orderflow jcmd 1 GC.heap_dump /dumps/t1.hprof
# ... 30 minutes of steady load ...
docker exec orderflow jcmd 1 GC.heap_dump /dumps/t2.hprof
```

In MAT: open both, use **Compare Basket** (add the histogram from each, then compare) to
get a delta histogram.

Cheaper, and often sufficient:

```bash
jcmd <pid> GC.class_histogram > /tmp/h1.txt
sleep 1800
jcmd <pid> GC.class_histogram > /tmp/h2.txt
diff <(awk 'NR>3 {print $4, $2}' /tmp/h1.txt | sort) \
     <(awk 'NR>3 {print $4, $2}' /tmp/h2.txt | sort) | head -40
```

**WHAT TO LOOK FOR:** classes whose instance count grew substantially between the two
snapshots and did not come back down after a GC. That is your leak candidate, identified
without MAT.

| What you see | What it means |
|---|---|
| One application class growing monotonically | Strong candidate. Merge Shortest Paths to GC Roots on it |
| Only JDK classes growing | Look at what holds them — the growth is a symptom |
| Nothing growing, but the heap is | Growth in a few large objects rather than many small ones. Compare **retained heap**, not instance counts |

### Proof 6 — JFR's `jdk.OldObjectSample`, the production leak tool nobody uses

This is the highest-value under-known technique in this document. JFR can sample objects
that **survive** into old age and record **the stack trace where each was allocated** —
which a heap dump can never do.

```bash
jcmd <pid> JFR.start name=leak settings=profile duration=30m filename=/dumps/leak.jfr
# ...wait under load...
jfr summary /dumps/leak.jfr | grep -i oldobject
jfr print --events jdk.OldObjectSample --stack-depth 20 /dumps/leak.jfr | head -60
```

The event's structure:

```
jdk.OldObjectSample {
  startTime      = <timestamp>
  allocationTime = <timestamp>
  objectAge      = <duration>
  lastKnownHeapUsage = <n>
  object         = <Class> : <description of the reference chain>
  arrayElements  = <n>
  root           = <GC root description>
  stackTrace     = [ <Class>.<method>(...) line: <n>, ... ]
}
```

*illustration of the format, not captured output — field names only*

**WHAT TO LOOK FOR:** the `stackTrace` (where it was allocated) together with the `root`
and reference chain (what holds it). **A heap dump gives you the second; only this gives you
both.**

| What you see | What it means |
|---|---|
| Repeated samples of one class with one allocation stack and one root | **The leak, with its creation site.** This is the fastest path from symptom to line number |
| Many different allocation stacks under one root | The holder is generic (a cache); the callers are varied. Fix the holder |
| The event is absent from `jfr summary` | Not enabled by your settings template, or not present on your JDK. Check `jfr metadata` and the `profile` template |
| `objectAge` values clustered near the recording length | Objects allocated early and never released — exactly the leak signature |

> **Flagged uncertainty.** `jdk.OldObjectSample`'s exact field set and its enablement in the
> stock templates vary by JDK version. **`jfr metadata <file>` is the authority for yours.**
> I am confident the event exists and records allocation stack traces for surviving objects;
> I am not confident about its precise field list on your build.

### Proof 7 — confirm the dump is a safepoint

```bash
java -Xlog:safepoint*:file=/tmp/sp.log:time,uptime,level,tags -Xmx256m -cp out ... &
jcmd <pid> GC.heap_dump /tmp/x.hprof
grep -i -E "HeapDump|HeapWalk|GC_HeapDump" /tmp/sp.log
```

**WHAT TO LOOK FOR:** a safepoint operation whose name references the heap dump, with a
substantial "At safepoint" duration (Topic 73). **That duration is application downtime.**
Seeing it once permanently changes how casually you dump production.

### Proof 8 — read the `ThreadLocal` chain in a real dump

Take a dump of any Spring Boot application that has served requests.

1. Dominator tree → find a `java.lang.Thread` row for a request thread.
2. Expand: `Thread` → `threadLocals` → `table` → `Entry` → `value`.
3. Read what is in the values.

**WHAT TO LOOK FOR:** framework contexts you did not put there —
`SecurityContextHolder`'s context, the logging MDC map (Topic 120), Hibernate's
`TransactionSynchronizationManager` resources, request-scoped bean holders (Topic 38).
**They are all supposed to be cleared at the end of the request.** Finding one that is *not*
empty on an idle thread is finding a leak.

| What you see | What it means |
|---|---|
| A handful of entries with null or small values on idle threads | Normal. Frameworks keep their `ThreadLocal` slots and clear the values |
| An entry holding a full request object on an **idle** thread | **A leak.** The thread is between requests and still holds the last one |
| An entry holding an entity or a persistence context | Worse: it drags a whole object graph, possibly a database session (Topics 48, 49) |
| Many entries per thread | Someone is creating `ThreadLocal` instances dynamically. The weak keys may be cleared but the values are not expunged — a nasty variant |

---

## Failure drill

**Two drills, as assigned in the master plan. Do them in order.** Drill (a) teaches the
tool; drill (b) teaches the judgement, and only makes sense once you can drive MAT.

---

### Drill (a) — the unbounded `static Map<String, Order>`

> **Master plan, Section G:** *Unbounded `static Map` cache under load → heap dump → MAT
> dominator tree. **Reachability, not allocation, is the leak.***

**Deliverable:** a written retaining path, from GC root to leaked object, naming the exact
field at each hop.

#### Step 0 — preconditions

- [ ] The Topic 65 baseline reproduces within ±10%.
- [ ] `/dumps` is a mounted volume, verified writable, with room for the live heap.
- [ ] MAT is installed and its own `-Xmx` exceeds the expected dump size.
- [ ] You have recorded the "clean" post-collection heap floor from a good run, so you have
      something to compare against.

#### Step 1 — plant the leak

```java
package com.orderflow.orders;

import org.springframework.stereotype.Component;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class OrderIdempotencyCache {
    /** DELIBERATE LEAK for Topic 79 drill (a). Never evicts. */
    private static final Map<String, Order> RECENT = new ConcurrentHashMap<>();

    public Order findExisting(String key) { return RECENT.get(key); }
    public void record(String key, Order order) { RECENT.put(key, order); }
}
```

Wire `record(...)` into the `POST /orders` path, keyed on a per-request UUID so every
request inserts a new entry.

**Use `static` for drill (a) deliberately**, so the path to root is short and unambiguous
and you learn the tool on an easy case. The instance-field-on-a-singleton variant from
Example 2 is the harder, more realistic version — do that one second, and notice how much
longer the path is.

#### Step 2 — configure the JVM to die usefully

```bash
-Xms512m -Xmx512m -XX:+UseG1GC
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps
-XX:+ExitOnOutOfMemoryError
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags
```

#### Step 3 — run the load and watch the floor, not the peaks

```bash
k6 run loadtest/baseline.js &
watch -n 10 'grep -E "Pause Young|Pause Full" /logs/gc.log | tail -5'
```

**Record, every five minutes, the post-collection occupancy.** Put it in a table. That table
— a rising floor — is your evidence that this is retention and not allocation, and you
produced it before opening any tool.

#### Step 4 — capture

Either wait for the OOM and take the automatic dump, or dump at roughly 80% occupancy:

```bash
docker exec orderflow jcmd 1 GC.heap_dump /dumps/drill-a.hprof
ls -lh dumps/
```

#### Step 5 — MAT, and the fifteen-minute clock

**Start a timer.** The mastery bar is fifteen minutes from dump to named path.

1. `ParseHeapDump.sh /dumps/drill-a.hprof org.eclipse.mat.api:suspects`
2. Open in the GUI. **Dominator Tree.** Sort by Retained Heap, descending.
3. Top row → expand downward, reading field names.
4. Top row → **Path to GC Roots → exclude all phantom/weak/soft etc.**
5. Write the path down.

**The deliverable, `/dumps/drill-a-findings.md`:**

```markdown
# Drill (a) — retaining path

## Evidence this is retention, not allocation
| Time | Post-collection heap occupancy |
|---|---|
| ... | ... |
(Rising floor => live set growing => retention.)

## The dominator tree, top three rows
| Rank | Class | Objects | Retained heap | % of heap |
|---|---|---|---|---|
| 1 | <...> | <n> | <n> | <n>% |

## The retaining path, root to leaf, with the FIELD at each hop
GC Root: System Class com.orderflow.orders.OrderIdempotencyCache
  -> static field RECENT            : java.util.concurrent.ConcurrentHashMap
  -> field table                    : Node[]
  -> element [i]                    : Node
  -> field val                      : com.orderflow.orders.Order
  -> field lines                    : <...>

## Retained heap per Order, and why it is larger than an Order
<Does the entity drag lazy collections, a Product, a session? Topics 48, 49.>

## Root cause, in one sentence
<...>

## Fix, and why not the alternatives
<Bounded cache / Redis TTL / unique constraint. Say why "more heap" is not on the list.>

## Verification
<Post-fix GC log floor flat; post-fix dump shows retained heap bounded by configured
maximum size, not by elapsed time; Topic 65 baseline within +/-10%.>
```

#### What drill (a) proves

1. **The objects were never garbage.** They were reachable, correctly, the whole time. There
   is no bug in the collector and nothing to "free".
2. **The histogram would not have found this.** Confirm by looking at it and seeing the
   usual `byte[]`/`String` top rows.
3. **The fix is a field, not a size.** Naming the field is the deliverable.

---

### Drill (b) — the `ThreadLocal` that plateaus

> **Master plan, Section G:** *`ThreadLocal` never removed on a fixed pool →
> bounded-but-permanent retention. **The subtlest leak shape in Java.***

**Deliverable:** a demonstration that retention equals pool-size × context-size, plus a
written argument for why a plateau is still a leak.

#### Step 1 — plant it, realistically

```java
package com.orderflow.web;

/**
 * DELIBERATE LEAK for Topic 79 drill (b).
 * Set per request. Never removed. On a fixed pool, this is bounded and permanent.
 */
public final class RequestContextHolder {

    private static final ThreadLocal<RequestContext> CONTEXT = new ThreadLocal<>();

    private RequestContextHolder() {}

    public static void set(RequestContext ctx) { CONTEXT.set(ctx); }
    public static RequestContext get()         { return CONTEXT.get(); }

    // The method that exists but is never called. That is the whole bug.
    public static void clear()                 { CONTEXT.remove(); }
}
```

```java
package com.orderflow.web;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
public class RequestContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest http = (HttpServletRequest) req;

        // The context deliberately holds a per-request payload, so the plateau's
        // height depends on the LARGEST request each thread has ever served.
        RequestContextHolder.set(new RequestContext(
                http.getHeader("X-Correlation-Id"),
                http.getHeader("X-Tenant-Id"),
                readBodySnapshot(http)          // e.g. the submitted order, lines and all
        ));

        chain.doFilter(req, res);
        // NO finally. NO clear(). This is the bug, and it looks like an oversight
        // because it is exactly what an oversight looks like.
    }
}
```

Pin the pool so the arithmetic is clean:

```yaml
# application-load.yml
server:
  tomcat:
    threads:
      max: 200
      min-spare: 200      # pre-start them all, so the plateau is reached quickly
```

#### Step 2 — the measurement that makes the point

Run the Topic 65 load and record the post-collection heap floor over time.

**WHAT TO LOOK FOR:** a rise, then a **plateau**. Not an OOM. The plateau is the finding.

| What you see | What it means |
|---|---|
| Floor rises during the first minutes, then flat | **The signature.** Every thread now holds one context; further requests overwrite |
| Floor keeps rising indefinitely | Something else is also leaking, or your pool is still growing. Check `min-spare` and the actual thread count |
| Floor flat from the start | The pool was already warm before you started measuring. Restart and measure from cold |
| Floor rises in steps | Threads being created lazily as concurrency increases. Each step is a new thread's first request |

#### Step 3 — the pool-size experiment (this is the proof)

Run the identical load three times, changing only the pool size:

| Run | `server.tomcat.threads.max` | Plateau height (post-GC floor) |
|---|---|---|
| 1 | 50 | `<n>` |
| 2 | 100 | `<n>` |
| 3 | 200 | `<n>` |

**WHAT TO LOOK FOR:** approximate proportionality to the pool size. Nothing else in a
Java service scales this way, which makes it close to conclusive.

#### Step 4 — confirm the mechanism in a dump

```bash
docker exec orderflow jcmd 1 GC.heap_dump /dumps/drill-b.hprof
```

In MAT:

1. Dominator tree, sorted by retained heap. Look for **many `java.lang.Thread` rows**, each
   with a similar, non-trivial retained heap. That uniformity is itself diagnostic.
2. Expand one: `Thread` → `threadLocals` → `table` → `Entry` → `value`.
3. **Path to GC Roots** on the `RequestContext` → ends at **Thread**, not at a static field.
4. Use **Merge Shortest Paths to GC Roots** on the `RequestContext` class to see all of
   them at once and get the aggregate.

| What you see | What it means |
|---|---|
| One `RequestContext` per request thread, none on non-request threads | Textbook. Exactly pool-size instances |
| The `RequestContext` retains far more than its own fields | It holds a request payload — the plateau height is set by the **largest** request each thread ever served. **Say this in the write-up; it is the part that makes the plateau dangerous** |
| More than one context per thread | Nested or re-entrant setting, or multiple `ThreadLocal` instances. Worse |
| Contexts on threads that are not request threads | Something propagated a `ThreadLocal` across an executor handoff — `InheritableThreadLocal`, or a manual copy (Topics 91, 119, 120) |
| Framework `ThreadLocal` entries with null values | Normal. Frameworks clear values and keep slots |

#### Step 5 — the argument, which is the real deliverable

Write `/dumps/drill-b-findings.md`:

```markdown
# Drill (b) — the bounded leak, and why "it plateaued" is not "it is fine"

## The plateau
| Pool size | Plateau height (post-GC floor) |
|---|---|
| 50  | <n> |
| 100 | <n> |
| 200 | <n> |
Retention is proportional to pool size => per-thread retention => ThreadLocal.

## The retaining path
GC Root: java.lang.Thread "http-nio-8080-exec-<n>"
  -> field threadLocals   : ThreadLocal$ThreadLocalMap
  -> field table          : ThreadLocal$ThreadLocalMap$Entry[]
  -> element [i]          : Entry  (extends WeakReference<ThreadLocal<?>>)
     -> referent (WEAK)   : the ThreadLocal key   <- collectable, and irrelevant
     -> field value (STRONG) : com.orderflow.web.RequestContext   <- the leak
  -> ...whatever RequestContext holds

## Why the key being weak does not save us
<Entry.value is a plain strong field. Stale entries are expunged only opportunistically
during a later get/set/remove that hashes near the stale slot. There is no cleaner and
no guarantee.>

## Why a plateau is still a leak
1. Live set permanently inflated => every collection costs more, forever (Topic 70).
2. Headroom permanently reduced => a spike that used to fit now OOMs.
3. The plateau's height is set by the LARGEST request each thread has served, which is
   not a constant. <state what your context holds>
4. <If applicable: the value transitively retains a ClassLoader / a persistence context.>

## The fix
try { ... } finally { RequestContextHolder.clear(); }   // remove(), not set(null)
Placed in a filter so it cannot be forgotten. Better: a request-scoped bean (Topic 38)
or ScopedValue (Topic 102), where the lifetime is structural rather than conventional.

## Verification
<Post-fix: plateau height no longer scales with pool size; a dump at steady state shows
no RequestContext on idle threads.>
```

#### What drill (b) proves

1. **"It stabilised" is not "it is healthy".** A permanently higher live set is a permanent
   cost you pay on every collection.
2. **The Thread-as-GC-root fact is load-bearing.** Once you internalise that a pool thread
   is a root for the process lifetime, this whole class of bug becomes obvious.
3. **Weak keys do not protect strong values**, and this exact asymmetry is why the JDK's own
   documentation tells you to call `remove()`.
4. **You now recognise the shape from a graph alone** — a step up followed by a flat line,
   coinciding with a release. That recognition is the differentiator.

---

## Measurement

### The four rules

**Rule 1 — measure the post-collection floor, never the current heap.**

Current heap usage is a sawtooth: it rises with allocation and drops at every collection.
It tells you nothing. **The post-collection floor is the live set**, and the live set is
what a leak grows.

```bash
# The one command that separates "leak" from "GC tuning":
grep -E "Pause Young|Pause Full" /logs/gc.log | tail -50
```

Read the occupancy **after** the arrow on each line. Better, plot it:

```bash
# Extract post-collection occupancy over time for a chart.
grep -oE "[0-9]+M->[0-9]+M\([0-9]+M\)" /logs/gc.log \
  | sed -E 's/.*->([0-9]+)M\(.*/\1/' > /tmp/floor.txt
```

> **Flagged.** The exact GC log line format varies by collector and JDK version. Confirm
> yours with `java -Xlog:gc -version` and adapt the pattern — do not assume mine matches.

| Floor behaviour | Diagnosis | Action |
|---|---|---|
| Rising monotonically over hours | Leak | Dump, dominate, name the path |
| Flat and low | Healthy | Any GC pain is allocation rate or tuning (Topics 68, 70, 71) |
| Flat and near `-Xmx` | Live set genuinely too large | More heap or a smaller live set. Legitimately a capacity question (Topic 129) |
| Flat, but stepped up since a release | **Bounded leak** — drill (b)'s shape | Still a leak |
| Sawtooth with a rising floor and lengthening full GCs | Pre-OOM death spiral | Urgent |

**Rule 2 — take at least two dumps, separated by time under load.**

One dump shows what is big. Two show what is **growing**. Growth is the leak signal; size is
not — a big cache that is supposed to be big is not a leak.

**Rule 3 — record the conditions with every dump.**

```bash
{
  date -Is
  docker exec orderflow jcmd 1 VM.uptime
  docker exec orderflow jcmd 1 VM.flags -all
  docker exec orderflow jcmd 1 GC.heap_info
  docker exec orderflow jcmd 1 Thread.print | grep -c '^"'      # thread count
  echo "load: k6 baseline.js; dataset 100k/1M/5M; heap -Xmx512m (reduced from 1200m)"
} > /dumps/drill-a-metadata.txt 2>&1
```

Especially record **that you reduced the heap**, if you did. Otherwise the absolute figures
in the dump will be misread as production numbers six months from now.

**Rule 4 — verify the fix with the same instrument that found the problem.**

Not "the OOM stopped happening" — that is the absence of evidence over too short a window.
The proof is:

1. The post-collection floor is flat over a long run under the same load.
2. A dump at steady state shows the suspect structure's retained heap bounded by its
   configured limit rather than by elapsed time.
3. The Topic 65 baseline percentiles are back within ±10%. *(A leak fix that costs you 15%
   throughput is a different conversation, and you want to have it deliberately.)*

### The instruments, and what each is for

| Instrument | Answers | Cost | Production? |
|---|---|---|---|
| `-Xlog:gc*` post-collection floor | "Is the live set growing?" | Negligible | **Always on** |
| `jcmd GC.heap_info` | Current occupancy by region | Cheap | Yes |
| `jcmd GC.class_histogram` | "What classes, how many" | Safepoint, no file | Yes, sparingly |
| Two histograms, diffed | "What is growing" | Two safepoints | Yes |
| `jcmd GC.heap_dump` | The full object graph | **Safepoint + a file the size of the live heap** | Only deliberately, out of rotation |
| `-XX:+HeapDumpOnOutOfMemoryError` | A dump at the moment of failure | Only on failure | **Always configured** |
| Eclipse MAT dominator tree | "What holds it, and how much would I get back" | Offline; needs lots of RAM | Offline |
| MAT Path to GC Roots | "The exact retaining path" | Offline | Offline |
| JFR `jdk.OldObjectSample` | "What survives, **and where it was allocated**" | Low | **Yes — the best production leak tool** |
| `jcmd VM.native_memory` (NMT) | "Is it even the heap?" | Small overhead when enabled | Yes (Topic 80) |
| Micrometer JVM memory metrics | "Is it growing, over weeks" | Negligible | **Always** (Topic 118) |

### The production configuration you should be running everywhere

```bash
# Diagnose on failure
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps                 # a MOUNTED volume, verified writable
-XX:+ExitOnOutOfMemoryError             # fail fast; restart clean; keep the evidence

# See the live set over time
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50M

# Attribute the footprint when heap is not the answer (Topic 80)
-XX:NativeMemoryTracking=summary

# Continuous JFR so you have evidence from BEFORE the incident
-XX:StartFlightRecording=name=continuous,settings=default,maxsize=200m,maxage=6h,dumponexit=true,filename=/dumps/continuous.jfr
```

Plus, in Micrometer (Topic 118), alert on the **trend of the post-collection floor**, not on
current heap usage:

- `jvm_memory_used_bytes{area="heap",id="G1 Old Gen"}` sampled after collections
- `jvm_gc_pause_seconds` count and sum, watching for the death-spiral shape
- `jvm_gc_live_data_size_bytes` where your Micrometer version exposes it — this is
  **exactly** the post-collection live set, and it is the single best leak alert available

**The alert that catches leaks and nothing else:** live-data-size trending upward over
days, with no corresponding growth in traffic or dataset. That is a leak, and it fires
weeks before the OOM.

### The fifteen-minute procedure, as a card

1. **Is it the heap?** `GC.heap_info` versus container RSS. If RSS grew and heap did not →
   Topic 80, stop.
2. **Is the live set growing?** Post-collection floor in the GC log. Flat → not a leak.
3. **Get a dump.** Automatic from OOM, or `GC.heap_dump` with the disk and rotation
   checklist.
4. **Dominator tree, sort by retained heap.** Top row.
5. **Path to GC Roots, excluding weak/soft.** Name the field.
6. **If the path ends at `Thread`** → `ThreadLocal`. If it ends at a **System Class** →
   a static field. If it ends at a **ClassLoader** → class-loader retention.
7. **Write the path down**, field by field, and fix the field.

### What this topic cannot tell you

| Question | Right tool |
|---|---|
| Where was this object allocated? | JFR `jdk.OldObjectSample`, or an allocation profile (Topic 78) |
| Why is RSS bigger than the heap? | NMT, `jcmd VM.native_memory` (Topic 80) |
| Why is the service slow but not leaking? | Profiling (Topic 78) |
| Is implementation A cheaper than B? | JMH (Topic 77) |
| Why did p99 spike with no GC pause? | `-Xlog:safepoint*` (Topic 73) |
| Are we leaking threads rather than objects? | `Thread.print`, thread count metrics (Topic 98) |
| Are we leaking classes? | Metaspace metrics, NMT `Class` category (Topics 67, 81) |

---

## Practice exercises

### 1 — Easy: build your leak-triage card, and learn your own tools

**Do this:**

1. Run Example 1 with `-Xmx128m`. Get the OOM and the automatic dump.
2. Open it in MAT. Find the retaining path. Time yourself.
3. Repeat with the histogram only (`jcmd GC.class_histogram`) and note that it does not
   answer the question.
4. Record, from your own commands: where MAT is installed, what its own `-Xmx` is, how to
   run the headless parser, and where your `HeapDumpPath` points in each environment you
   own.
5. Verify `HeapDumpPath` is writable in every one of those environments. **Find out now,
   not during an incident.**

**Deliverable:** a one-page triage card: the five diagnostic commands, the four GC-log floor
patterns and what each means, and the four leak shapes with the root kind each produces.

**You have succeeded when:** you can go from "the heap graph looks wrong" to "here is the
field" without looking anything up.

---

### 2 — Medium: the four shapes, all four, in one service
*(combines Topics 13, 15, 21, 38, 40, 48, 49, 56, 67, 68, 70, 78, 120)*

**Goal:** produce every classic leak shape deliberately, so you recognise each from its
dominator tree and its path-to-root.

**Do this in `orderflow`, one at a time, dumping and analysing each before moving on:**

1. **Unbounded static collection** — drill (a). Root kind: System Class.
2. **Unremoved listener** — register an `ApplicationListener` per request with a
   `ConfigurableApplicationContext` and never remove it. Root kind: the context, itself
   reachable from a static. Note how much *each* listener retains via its captured fields.
3. **`ThreadLocal` on a pooled thread** — drill (b). Root kind: Thread.
4. **Non-static inner class outliving its enclosing instance** — give `OrderService` a
   non-static inner `Runnable` submitted to a long-lived scheduled executor. It captures
   `this$0`. Confirm the synthetic field with `javap -p` (Topic 76), then find it in the
   dump. Root kind: the executor's thread or its queue.
5. **Bonus, and the nastiest:** a **mutable key** (Topic 13). Put a `Product` in a
   `HashSet`, mutate its `sku`, and observe that the entry is unreachable by lookup but
   still retained and still counted in `size()`. This is simultaneously a correctness bug
   and a leak, and MAT shows it as ordinary retention with no hint that anything is wrong.

For each: the dominator-tree top row, the path to GC roots with field names, the root kind,
and the fix.

**Then the cross-check:** for shapes 1 and 3, also capture a JFR recording with
`jdk.OldObjectSample` and see whether it names the allocation site. Compare the two
workflows and write down which was faster and why.

**Deliverable:** a reference table — shape, dominator-tree signature, root kind,
path-to-root shape, fix — built entirely from your own dumps.

**You have succeeded when:** you can name the shape from the root kind alone.

---

### 3 — Hard: production simulation — a leak you did not plant
*(against the Topic 65 baseline; combines Topics 48, 49, 65, 70, 71, 78, 79, 80, 82, 118)*

**Setup.** Have a colleague introduce **one** of the following into `orderflow`, without
telling you which, and run the Topic 65 load for an extended period:

- (a) an unbounded cache keyed on something high-cardinality;
- (b) a `ThreadLocal` never cleared;
- (c) an `EntityManager`/persistence context held beyond its transaction (Topics 48, 49);
- (d) a growing `DirectByteBuffer` pool with no released references (Topic 80);
- (e) an `ExecutorService` created per request and never shut down (Topic 98);
- (f) **nothing at all** — a dataset that legitimately grew, so the live set is genuinely
      larger and the correct answer is "more heap" (Topics 70, 129).

**Your task, timed, in this order:**

1. **Classify the failure first.** Heap OOM, container OOMKill, or degraded-but-alive?
   `docker inspect` exit code and `OOMKilled`, plus `GC.heap_info` versus RSS.
2. **Is the live set growing?** Post-collection floor from the GC log. **If it is flat, do
   not open MAT** — you would waste an hour finding a large-but-legitimate structure.
3. **If native:** NMT baseline and diff. Which category grew? Stop here for (d).
4. **If heap:** two histograms 30 minutes apart, diffed. Candidate class?
5. **Dump. Dominator tree. Path to GC roots.** Name the field.
6. **If it is (e):** the histogram will show threads, not objects. `Thread.print | grep -c`
   and the NMT `Thread` category are your instruments, not MAT.
7. **If it is (f):** the correct answer is that there is no leak. **Say so, with evidence.**
   Reporting "no defect found, here is why" is a real skill and the hardest of the six.
8. Fix, re-run the load, verify the floor is flat, re-run the Topic 65 baseline and compare
   every percentile against `/docs/java/baselines/`.

**Deliverable:** an incident write-up: symptom, classification, hypotheses and how each was
excluded, the decisive evidence, the retaining path (if any), the fix, and verification.
Include a section **"the wrong tool I reached for first"** — everyone reaches for MAT too
early at least once, and writing it down is what stops it becoming a habit.

**You have succeeded when:** you correctly identify (d) as native rather than heap without
opening MAT, and correctly identify (f) as not-a-leak without inventing one.

---

## Interview questions

### Q1 — "We're leaking memory. What do you do first?"

**MID-LEVEL answer:** *"Take a heap dump and open it in MAT to see what's using the most
memory."*

Right tool, wrong first step. There are two cheaper questions that must come first, and one
of them frequently makes the dump unnecessary.

**SENIOR answer:** *"Two questions before I dump anything, because a dump is expensive and
half the time it is the wrong tool.*

*First: is it the heap at all? If the container is being OOMKilled — exit 137, no Java
exception, no heap dump despite the flag — then the kernel killed us on RSS, and RSS is heap
plus metaspace plus code cache plus thread stacks plus direct buffers plus native libraries.
A heap dump will show a perfectly healthy heap and I will have wasted an afternoon. One
`GC.heap_info` against `docker stats` answers it, and if they diverge I go to native memory
tracking instead.*

*Second: is the live set actually growing? I grep the GC log for the heap occupancy **after**
each collection. Current heap usage is a sawtooth and means nothing. If the post-collection
floor rises steadily over hours, that is retention and I have a leak. If it is flat, I do not
— I have an allocation-rate problem or a live set that is legitimately too big for the heap
I provisioned, and those have completely different fixes.*

*Only then do I dump. And I'd check first that HeapDumpPath points at a mounted volume with
enough disk, and take the instance out of rotation, because the dump is a global safepoint
that will fail my liveness probe.*

*Once I'm in MAT: dominator tree, sort by retained heap, top row, path to GC roots excluding
weak and soft references. I want to end the sentence with a field name — 'the static
`RECENT` field on `OrderIdempotencyCache`' — not a class name and not 'the cache'."*

**What separates them:** two cheap eliminations before an expensive operation, the container-
versus-heap OOM distinction, the post-collection-floor test, and the operational awareness
that a dump is a stop-the-world with a disk cost. And ending on a field name.

**Follow-up:** *"The floor is flat. Is it a leak?"*
→ *"Not a growing one. It could still be a bounded leak — the `ThreadLocal` shape plateaus
at pool-size times context-size and then stops, so a flat line at a *new, higher* level
since a release is exactly what that looks like. I'd compare the floor against the baselines
from before the release. If the floor is flat and unchanged, there is no leak and I'd go
look at allocation rate or capacity instead."*

---

### Q2 — "Why is the dominator tree better than a histogram?"

**MID-LEVEL answer:** *"The dominator tree shows what's holding the memory, and the
histogram just shows classes."*

Correct but shallow — it does not survive "why does that matter?"

**SENIOR answer:** *"Because they answer different questions, and only one of them is my
question.*

*A histogram groups objects by class and sums their **shallow** heap. Shallow heap is the
object's own bytes, so the memory gets attributed to the leaf objects that hold the data —
`byte[]`, `String`, `Object[]`, `HashMap$Node`. Those are the top rows of every Java heap
histogram ever produced. If my finding would be true of any Java program, it is not a
finding. And the histogram has no field names in it at all, so it structurally cannot tell
me who holds anything.*

*The dominator tree is a different computation. A dominates B if every path from any GC root
to B goes through A. Each object's parent in the tree is its immediate dominator, and the
retained heap of A is the sum of everything in its subtree — which is literally 'how much
memory would come back if A went away'. That is the question I actually have. A leaking
`HashMap` has a shallow heap of a few dozen bytes and a retained heap of most of the heap;
it is invisible in one view and unmissable in the other.*

*There is a subtlety worth knowing: if an object is reachable by two independent paths, its
dominator is above both, so the cache I suspect may show a smaller retained heap than I
expected. That is not a bug — it is telling me that freeing the cache alone would not free
those objects, and I'd use Merge Shortest Paths to GC Roots to find every holder.*

*Histograms are still useful for one thing: the **delta** between two snapshots. Absolute
counts are noise; growth is signal."*

**What separates them:** the precise dominator definition, the shallow-versus-retained
mechanism, the multiply-referenced subtlety, and rehabilitating the histogram for the one
job it is good at.

**Follow-up:** *"MAT's Leak Suspects report already told you the answer. Why go further?"*
→ *"It's a good first pass and I always run it. But it's a heuristic — it looks for a
dominator holding a large share and reports it. It can be confidently wrong when retention
is spread across several roots, or when the biggest retainer is a legitimate cache. I use
it as a hypothesis and verify it in the dominator tree, because what I have to put in the
fix is a specific field, and the report gives me a suspect."*

---

### Q3 — "Memory rose after a release and then flattened. The team closed it as warm-up. Do you agree?"

**MID-LEVEL answer:** *"If it stabilised then it's probably fine — caches and pools warm up
and then level off."*

This is the misdiagnosis. It is also a completely reasonable-sounding answer, which is why
this shape ships.

**SENIOR answer:** *"A plateau is exactly what one specific leak looks like, so I would not
close it on the shape of the graph alone.*

*If a request context is put in a `ThreadLocal` and never removed, and the threads come from
a fixed pool, then each thread ends up holding one context. Once every thread has served one
request, further requests overwrite rather than accumulate. Retention is pool-size times
context-size and then it stops. The graph rises and flattens — indistinguishable from
healthy warm-up.*

*The test is one restart: halve the thread-pool size and re-run the same load. If the plateau
halves, it is per-thread retention. Nothing else in a Java service scales that way.*

*And it is still a leak even though it never OOMs. My live set is permanently larger, and GC
cost is a function of the live set, so every collection now does more work — forever. My
headroom is permanently smaller, so the traffic spike that used to fit now does not. And the
plateau's height is not really a constant: it is set by the **largest** request each thread
has ever served. If the context holds a per-request payload, that height is unbounded in
practice and it materialises months later under an unusual load pattern.*

*The mechanism is worth stating precisely, because the almost-right version is why people
dismiss it: `ThreadLocalMap.Entry` extends `WeakReference` on the **key**, but `value` is a
plain strong field. So the `ThreadLocal` object can be collected and the value is still
strongly held from the thread. Stale entries are only expunged opportunistically during a
later get or set that happens to hash near them. The fix is `remove()` in a `finally`, in a
filter so it cannot be forgotten — not `set(null)`, which leaves a live entry."*

**What separates them:** recognising the plateau as a signature, proposing a decisive
one-restart experiment, arguing that bounded retention is still a defect via the live-set
argument, the weak-key/strong-value asymmetry, and `remove()` versus `set(null)`.

**Follow-up:** *"How would you stop the whole class of bug rather than this instance?"*
→ *"Put the cleanup in a filter or interceptor so it is structural rather than per-handler
— that's exactly why Spring Security clears its holder in a `finally`. Better, don't use raw
`ThreadLocal` for request state: use a request-scoped bean, or `ScopedValue` where the
lifetime is bounded by the structure of the code rather than by a convention. And add a test
that asserts an idle pool thread's `threadLocals` is empty after a request completes — that
is a cheap, permanent regression guard, and almost nobody writes it."*

---

### Q4 — "The container gets OOMKilled but the heap looks fine. What's going on?"

**MID-LEVEL answer:** *"Maybe the heap size is set too high for the container. I'd lower
`-Xmx`."*

Plausible, sometimes even the fix, but arrived at by guessing.

**SENIOR answer:** *"Two different failures with two different symptoms, and the first thing
I'd do is confirm which one this is.*

*A heap OOM throws `java.lang.OutOfMemoryError`, produces a stack trace, and — if
`HeapDumpOnOutOfMemoryError` is configured — writes a dump. A container OOMKill is the
kernel: the process vanishes, exit code 137, `OOMKilled` true in `docker inspect`, a line in
`dmesg`, no Java exception, and crucially **no heap dump**, because the JVM never got to
throw anything. If people are looking for a dump that was never written, that alone can
burn a day.*

*The reason the heap looks fine is that the heap is only part of RSS. RSS is heap plus
metaspace plus the code cache plus thread stacks — around a megabyte each by default, so a
thread leak shows up here — plus GC data structures, plus direct byte buffers, plus mapped
files, plus whatever native libraries allocate. So I'd turn on `-XX:NativeMemoryTracking=summary`,
take a baseline, run the load, and diff. The category that grew names the problem: Class or
Metaspace means a class-loader leak, Thread means a thread leak, Internal or Other usually
means direct buffers, which is the Netty and NIO story.*

*And I'd check the sizing arithmetic separately, because it is a common independent bug:
`-Xmx` set to the container limit guarantees an OOMKill, since heap is not the whole
footprint. The default `MaxRAMPercentage` of 25% is also wrong for a container that exists
to run one JVM. Heap should come from a measured live set plus headroom, not from a rule of
thumb.*

*If NMT shows nothing growing and RSS still climbs, I'd look at native allocator
fragmentation and at libraries outside the JVM's accounting — but that is rare, and I'd want
the NMT diff before going there."*

**What separates them:** distinguishing the two failures by their observable signatures,
knowing RSS's composition, reaching for NMT rather than guessing, and treating the sizing
question as separate from the leak question.

**Follow-up:** *"NMT says the `Internal` category is growing. Where do you look?"*
→ *"Direct byte buffers. The native memory behind a `DirectByteBuffer` is freed by a
`Cleaner` that runs only when the small Java wrapper object is collected — so if the wrappers
are retained, or if the heap simply is not under enough pressure to collect them, native
memory grows while the heap looks calm. I'd set `-XX:MaxDirectMemorySize` so it fails with a
diagnosable error rather than an OOMKill, and I'd look at the NIO and Netty paths — that's
Topic 80."*

---

### Q5 — "Can you take a heap dump in production?"

**MID-LEVEL answer:** *"Yes, `jmap -dump` on the running process."*

Technically true, operationally reckless.

**SENIOR answer:** *"Yes, deliberately, with a checklist — and often I'd rather not need to.*

*What it costs: a heap dump is a global safepoint, so every application thread stops for the
duration, and it writes a file roughly the size of the live heap. On a Kubernetes pod that
means two specific hazards: if the liveness probe times out during the stop-the-world, the
pod is killed mid-dump and I get a truncated file plus an unplanned restart; and if the path
is not a mounted volume, the dump dies with the container.*

*So the checklist is: check the live heap size with `GC.heap_info`, check disk with `df` on a
**mounted** path, take the instance out of the load-balancer rotation or raise the probe
timeouts, then `jcmd GC.heap_dump`, then parse it headlessly on the host rather than copying
gigabytes to a laptop — and MAT needs more heap than the dump size or it will OOM analysing
my OOM.*

*But the better answer is to arrange the evidence in advance so I do not have to do this
under pressure. `HeapDumpOnOutOfMemoryError` with a verified writable mounted path and
`ExitOnOutOfMemoryError`, in every environment. Continuous JFR with a bounded size and age,
`dumponexit`, so I have a recording from **before** the incident. And JFR's
`jdk.OldObjectSample` is the underrated one: it samples objects that survive into old age
and records **where they were allocated**, which a heap dump can never do. Allocation site
plus retaining root, at a fraction of the cost.*

*The thing I would not do is take dumps repeatedly to watch a trend. Each one is a
stop-the-world. Two histograms thirty minutes apart give me the growth signal much more
cheaply, and the JVM memory metrics in Micrometer give it to me for free."*

**What separates them:** the safepoint and disk costs, the Kubernetes probe interaction, the
headless-parse workflow, and — most of all — moving the work from incident time to
configuration time.

**Follow-up:** *"You can't restart or take it out of rotation. It's the only pod. What now?"*
→ *"Then I take the cheap things first: `GC.class_histogram` twice, thirty minutes apart, and
diff the instance counts. That is a safepoint but a much shorter one, no file, and it will
usually name the growing class. Plus a JFR recording with `OldObjectSample`, which is low
overhead and gives me allocation sites. If those narrow it enough to name the field, I never
need the dump — and if they don't, I've at least earned the argument for a maintenance
window with evidence."*

---

## Mental model checkpoint

1. **Define a Java memory leak in one sentence that does not use the word "free".**
   *(Mechanical statement. Unintended reachability: a path from a GC root to an object that
   should be dead.)*

2. **What question does retained heap answer that shallow heap does not? Why does that make
   the histogram the wrong first view?**
   *(What is this? / Trap 2. Retained = what would be freed. The histogram sums shallow heap
   and has no field names, so it names the material, never the culprit.)*

3. **Name the four classic leak shapes, and the GC-root kind each produces in a
   path-to-root.**
   *(Static collection → System Class; unremoved listener → the publisher's root; `ThreadLocal`
   on a pooled thread → Thread; inner class via `this$0` → whatever holds the inner instance.)*

4. **In `ThreadLocalMap`, which reference is weak and which is strong? Why does that
   asymmetry produce a leak, and why doesn't the weak reference save you?**
   *(Machine-level reality §4. Weak key, strong value. Clearing the key does not clear the
   value; stale entries are expunged only opportunistically, with no cleaner and no
   guarantee.)*

5. **Heap graph rises then plateaus. Give the one-restart experiment that identifies the
   cause, and the argument for why a plateau is still a defect.**
   *(Drill (b). Halve the pool; if the plateau halves, it is per-thread retention. Live set
   is permanently inflated, GC cost tracks the live set, headroom is permanently reduced, and
   the plateau's height is set by the largest request each thread has served.)*

6. **A container is OOMKilled and there is no heap dump despite the flag. Explain both facts
   with one mechanism, and name your next command.**
   *(Trap 5. The kernel killed the process on RSS; the JVM never threw, so the dump hook
   never fired. Next: `jcmd VM.native_memory summary.diff`.)*

7. **List the three costs of `jcmd GC.heap_dump` on a production pod, and the two things you
   configure in advance so you rarely need it.**
   *(Global safepoint; a file the size of the live heap; a liveness-probe timeout risk.
   In advance: `HeapDumpOnOutOfMemoryError` to a mounted verified path, and continuous JFR
   with `jdk.OldObjectSample`.)*

---

## Quick reference card

### Getting a dump

```bash
# Always configured, everywhere:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps -XX:+ExitOnOutOfMemoryError

# On demand (full GC first; reachable objects only):
jcmd <pid> GC.heap_dump /dumps/x.hprof
jcmd <pid> GC.heap_dump -all=true /dumps/x-all.hprof     # include unreachable

# From a core file, after the fact:
jhsdb jmap --binaryheap --dumpfile /dumps/x.hprof --core <core> --exe $(which java)
```

### Cheap diagnostics, before you dump

```bash
jcmd <pid> GC.heap_info                     # occupancy by region
jcmd <pid> GC.class_histogram | head -30    # what classes, how many
jcmd <pid> VM.native_memory summary         # is it even the heap? (Topic 80)
jcmd <pid> Thread.print | grep -c '^"'      # thread count (Topic 98)
grep -E "Pause Young|Pause Full" gc.log | tail -50   # the post-collection FLOOR
docker inspect <c> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
```

### MAT

```bash
/opt/mat/ParseHeapDump.sh /dumps/x.hprof org.eclipse.mat.api:suspects
# GUI order, every time:
#   1. Dominator Tree, sort by Retained Heap desc
#   2. Expand the top row; read the FIELD NAMES
#   3. Right-click -> Path to GC Roots -> exclude phantom/weak/soft
#   4. For many instances of one class: histogram -> Merge Shortest Paths to GC Roots
#   5. Histogram LAST, only to confirm
# MemoryAnalyzer.ini -Xmx MUST exceed the dump size.
```

### Dominator-tree columns

```
Class Name                              | Objects | Shallow Heap | Retained Heap | Percentage
<pkg>.<Class> @ 0x<addr>                |   <n>   |     <n>      |      <n>      |   xx.x%
```
*illustration of the format, not captured output — column structure only*

**Sort by Retained Heap. Always. First.**

### The four shapes, and their signatures

| Shape | Path-to-root ends at | Signature | Fix |
|---|---|---|---|
| Unbounded static collection | **System Class** | Floor rises linearly with traffic | Bounded cache, TTL, or move the state out of the JVM |
| Unremoved listener / callback | The publisher's root | Instance count tracks registrations | Unregister in `finally`, or weak listeners |
| **`ThreadLocal` on a pooled thread** | **Thread** | **Plateaus at pool-size × context-size** | `remove()` in `finally`, in a filter |
| Inner class / lambda capturing `this` | Whatever holds the inner instance | `this$0` in the path (`javap -p`, Topic 76) | Make it static; pass what you need |

### Failure signatures

| Symptom | Meaning | Next |
|---|---|---|
| `OutOfMemoryError: Java heap space` | Heap exhausted | Dump → dominator tree |
| `OutOfMemoryError: GC overhead limit exceeded` | Collecting constantly, reclaiming little | Same, earlier |
| `OutOfMemoryError: Metaspace` | Class-metadata leak | Class loaders (Topics 67, 81) |
| `OutOfMemoryError: unable to create native thread` | Thread leak (Topic 98) | `Thread.print`, NMT `Thread` |
| `OutOfMemoryError: Direct buffer memory` | Direct buffers (Topic 80) | NMT, `MaxDirectMemorySize` |
| Exit 137, no exception, no dump | **Container OOMKill** | NMT diff (Topics 80, 82) |

### Gotchas checklist

- [ ] Did I confirm it is the **heap** (heap vs RSS) before dumping?
- [ ] Is the post-collection **floor** rising, or just the sawtooth peaks?
- [ ] Is `HeapDumpPath` a **mounted volume**, and is it writable? (Test it now.)
- [ ] Is there disk for a file the size of the live heap?
- [ ] Did I take the pod out of rotation / raise the probe timeout before dumping?
- [ ] Does MAT have more heap than the dump size?
- [ ] Am I in the **dominator tree**, not the histogram?
- [ ] Did I **exclude weak/soft** in Path to GC Roots?
- [ ] Can I name the **field**, not just the class?
- [ ] If the path ended at `Thread` — did I check `threadLocals`?
- [ ] Did I take a **second** dump to prove growth rather than size?
- [ ] Did I verify the fix with a flat floor **and** the Topic 65 baseline?

---

## When would I use this at work?

**1. The 02:00 page: a service restarting every few hours with rising memory.**
You check the exit code first — heap OOM or OOMKill, because they are different
investigations. You grep the GC log's post-collection floor to confirm retention. The
automatic dump is already on a mounted volume because you configured it months ago. Fifteen
minutes later you post a retaining path with a field name and a one-line fix. The difference
between this and a six-hour incident is almost entirely preparation: the dump flags, the
mounted path, and knowing which two questions to ask before opening MAT.

**2. Reviewing a PR that adds a cache — before it ships.**
Someone adds a `Map` field for idempotency, request deduplication, or "a small lookup". You
ask three questions in the review: what bounds it, what evicts from it, and what is the
retained size of one entry. That third one is the senior question, because a cache of
Hibernate entities retains far more than a cache of DTOs — an entity drags lazy collections,
a `Product`, possibly a session. And often the answer reveals a design bug rather than a
memory bug: an in-JVM idempotency cache does not work behind a load balancer at all.

**3. Explaining a plateau nobody else thinks is a problem.**
The heap graph stepped up after a release and flattened. Everyone moved on. You halve the
thread pool, show that the plateau halves, and explain that the live set is permanently
inflated so every collection now costs more — forever — and that the plateau's height is set
by the largest request each thread has ever served, which is not a constant. Then you fix it
with three lines in a filter. Being the person who reads a flat line as a defect is a
genuinely rare skill, and it is the one this topic's second drill exists to build.

---

## Connected topics

**Prerequisites:**

- **13 — The `equals`/`hashCode` contract:** a mutated key is unreachable by lookup but
  still retained and still counted in `size()`. Simultaneously a correctness bug and a leak,
  and MAT shows it as ordinary retention with no hint that anything is wrong.
- **15 — `LinkedHashMap` and LRU:** the minimal bounded cache, and why you would reach for
  Caffeine instead once eviction policy, TTL and stats become requirements.
- **17 — Immutability and safe publication:** an immutable projection retains far less than a
  mutable entity graph, and it is safe to cache across threads.
- **21 / 22 — Lambdas and method references:** a capturing lambda holds its captured
  references, including `this`. That is leak shape 4 in modern clothing.
- **38 — Bean scopes:** request-scoped beans are the framework's answer to the problem raw
  `ThreadLocal` solves badly, and scoped proxies are how they escape the singleton trap.
- **40 — Proxying:** repeated dynamic proxy generation is a classic metaspace and
  class-loader leak.
- **48 / 49 — Persistence context and lazy associations:** caching an entity retains its
  snapshot, its lazy proxies and potentially its session. This is why "cache the DTO, not the
  entity" is a rule and not a preference.
- **56 — Spring Security's filter chain:** `SecurityContextHolder` is `ThreadLocal`-backed,
  and the framework clears it in a `finally` **because of exactly this topic**. Read that
  filter's source; it is the fix, written by people who learned it the hard way.
- **65 — The load-testing gate:** both drills run under it. Only sustained realistic load
  produces the distinct-key volume and the thread-pool saturation these leaks need.
- **67 — Class loading:** class-loader retention is the fifth leak shape, and the reason a
  metaspace OOM is a different investigation.
- **68 / 70 — TLABs, allocation, GC fundamentals:** **the load-bearing prerequisite.** "GC
  cost is a function of the live set, not of garbage" is the entire argument for why a
  bounded leak still matters.
- **71 / 72 — G1 and low-pause collectors:** the GC log is where you read the post-collection
  floor; the death-spiral shape is visible there before the OOM.
- **73 — Safepoints:** a heap dump is a safepoint operation, and knowing what that costs is
  the difference between a controlled dump and an unplanned pod restart.
- **76 — Reading bytecode:** `javap -p` shows the synthetic `this$0` field on a non-static
  inner class. That is leak shape 4, made visible in the class file.
- **78 — Profiling:** the sibling. A profiler tells you where time goes; a heap dump tells
  you what is retained. Neither substitutes for the other, and JFR's `OldObjectSample`
  bridges them by giving you allocation sites for surviving objects.

**This unlocks:**

- **80 — Off-heap and NMT:** where you go when the heap is healthy and RSS is not. Every
  investigation in this document begins by ruling that out.
- **81 — Instrumentation agents:** agents generate classes at runtime and can themselves
  retain class loaders. An APM agent is a plausible suspect in a metaspace leak.
- **82 — Containers and JVM tuning:** the container-OOMKill-versus-heap-OOM distinction, and
  why `-Xmx` equal to the container limit guarantees the former.
- **90 — `ExecutorService` and pool sizing:** pool threads are the GC roots that make leak
  shape 3 possible, and an unbounded queue is itself a retention structure.
- **98 — Concurrency bug taxonomy:** thread leaks are OOM with a different message, and the
  investigation uses thread counts rather than heap dumps.
- **101 / 102 — Virtual threads, structured concurrency and scoped values:** virtual threads
  are not pooled, so a `ThreadLocal` on one dies with it — which changes leak shape 3
  fundamentally. `ScopedValue` is the structural replacement, where lifetime is enforced by
  the code's shape rather than by a `finally` someone must remember.
- **110 — Spring Cache and Redis:** the correct home for state that must be bounded, shared
  across instances, and evicted on a schedule.
- **118 — Micrometer metrics:** the alert that catches leaks weeks early is the **trend of
  the post-collection live set**, not current heap usage. Setting that up is the most
  valuable thing you can do with this topic.
- **129 — Capacity, cost and latency budgets:** live set plus headroom is how you size a
  heap. A leak is a live set with no equilibrium, which is why it cannot be budgeted around.

---

*Java baseline 21, running on JDK 25. Several things in this document are deliberately
hedged rather than asserted, each with a settling command given inline: Eclipse MAT's
headless script name, report identifiers and configuration file location (`ls` your
installation and read `MemoryAnalyzer.ini`), the exact field set of JFR's
`jdk.OldObjectSample` and whether it is enabled by your settings template (`jfr metadata`),
and the exact GC-log line format your collector and JDK emit (`java -Xlog:gc -version`
before you copy any grep pattern from here).*

*Everything else here — that a Java leak is unintended reachability, that MAT's dominator
tree computes "what would be freed if this went away" while a histogram sums shallow heap by
class, that `ThreadLocalMap.Entry` holds a weak key and a strong value and expunges stale
entries only opportunistically, that a live `Thread` is a GC root, that a heap dump is a
global safepoint writing a file about the size of the live heap, and that a container
OOMKill produces no `OutOfMemoryError` and therefore no automatic heap dump — is documented
behaviour or readable in the JDK source, and each has a confirming command above.*

*No tool output was captured for this document. There are no MAT figures here, no retained
sizes, no object counts, no dump sizes and no percentages of heap. The four format blocks
contain placeholders only. Every number you quote from this topic must come from your own
dump, with its metadata attached.*
