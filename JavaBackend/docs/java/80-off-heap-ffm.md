# 80 — Off-Heap Memory: `DirectByteBuffer`, the FFM API, and Native Memory Tracking

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: diagnoses the gap between `orderflow`'s heap usage and the container's RSS — the gap that gets the pod OOMKilled while every heap dashboard says the service is healthy.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> A `DirectByteBuffer` is a **small Java object on the heap** that owns a **large block
> of native memory outside the heap**.
>
> That native block is released by a `Cleaner` — a `PhantomReference` — which only
> runs **after the garbage collector has decided the small Java wrapper object is
> unreachable, and after the reference-processing thread has drained the queue**.
>
> The garbage collector's decision to run at all is driven by **heap occupancy**. The
> wrapper is tens of bytes. The native block is megabytes. **The GC therefore has no
> idea that the expensive thing exists.**
>
> So native memory grows, the heap looks perfectly healthy, no `OutOfMemoryError` is
> thrown, no GC pressure appears in the logs — and the kernel's cgroup OOM killer
> sends `SIGKILL` to a process that, from inside, sees no problem at all.

The container dies. The JVM's last words are nothing, because `SIGKILL` cannot be
caught. There is no heap dump. There is no exception. There is an exit code of 137 and
a Kubernetes event that says `OOMKilled`.

`[JAVA 25]` The Foreign Function & Memory API (`java.lang.foreign`) exists precisely
because this lifetime story is bad. An `Arena` gives you a **deterministic** close.
That is the single most important thing FFM adds over `DirectByteBuffer`, and it is
worth more than the foreign-function-calling half of the API for most backend work.

---

## The bridge from what you know

### The HONEST analogue: Node's `Buffer`

This is one of the genuinely honest analogues in the whole curriculum, and you have
almost certainly already hit the symptom.

```ts
// Node
const buffers: Buffer[] = [];

setInterval(() => {
  buffers.push(Buffer.allocUnsafe(16 * 1024 * 1024));   // 16 MB, OUTSIDE V8's heap
  const heap = process.memoryUsage();
  console.log(
    `heapUsed=${(heap.heapUsed / 1e6).toFixed(1)}MB ` +
    `rss=${(heap.rss / 1e6).toFixed(1)}MB ` +
    `external=${(heap.external / 1e6).toFixed(1)}MB`
  );
}, 1000);
```

Run that and you see `heapUsed` barely move while `rss` climbs without limit. If you
have ever debugged "my Node process RSS is 3 GB but the heap snapshot is 90 MB", you
have already lived this topic's failure mode.

**What transfers, exactly:**

| Fact | Node | Java |
|---|---|---|
| The big allocation lives outside the managed heap | `Buffer` (backed by an `ArrayBuffer` allocated in C++) | `DirectByteBuffer` (backed by `Unsafe.allocateMemory`) |
| The managed heap holds only a small handle | the JS `Buffer` object | the `DirectByteBuffer` Java object |
| The GC frees the native block when the handle dies | V8 external-memory bookkeeping + destructor | `Cleaner` / `PhantomReference` |
| Heap dashboards lie about total footprint | yes | yes |
| The container OOM-kills you with a "healthy" heap | yes | yes |

**Verdict: HONEST analogue. Use it. Lead with it in an interview.**

### Where Java is worse than Node, and why that matters

Node has one advantage you are about to lose. V8 has an **external memory
accounting** hook: `AdjustAmountOfExternalAllocatedMemory`. When you allocate a
`Buffer`, V8 is *told* about the external bytes, and that number participates in V8's
decision to run a GC. It is not perfect, but the GC has been informed.

**HotSpot has no such hook for `DirectByteBuffer`.** The collector's heuristics —
eden occupancy, allocation rate, promotion rate, IHOP for G1 — are all about heap
bytes. A `DirectByteBuffer` wrapper is a tiny, short-lived-looking object. It exerts
almost no pressure. The GC may not run for minutes while native memory triples.

There *is* one compensating mechanism, and it is worth knowing exactly, because it is
also a trap: `java.nio.Bits.reserveMemory` — the code path behind
`ByteBuffer.allocateDirect` — checks the direct-memory budget before allocating. If
the budget is exhausted it calls `System.gc()`, waits, retries with backoff, and
eventually throws `OutOfMemoryError: Direct buffer memory`.

That mechanism only fires **if you have set a budget the JVM can see**. And the
default budget is not the container limit — it is derived from the *max heap*, which
has nothing to do with how much native memory the container can afford. More on that
below, with the command that settles it on your machine.

### What has NO analogue: `Arena` and deterministic native lifetime

Node has no equivalent of `Arena.ofConfined()`. You cannot say, in JavaScript, "this
block of native memory is freed at the end of this lexical scope, on this thread, and
any access afterwards throws a catchable exception rather than segfaulting."

```java
try (Arena arena = Arena.ofConfined()) {
    MemorySegment segment = arena.allocate(16L * 1024 * 1024);
    segment.set(ValueLayout.JAVA_BYTE, 0, (byte) 42);
}   // native memory freed HERE. Deterministically. Not "eventually".
```

**Verdict: NO ANALOGUE.** This is genuinely new, it is the reason FFM matters for
backend work, and it is where you should point when someone asks "why would I use FFM
if I am not calling C?"

### What has NO analogue: `jcmd VM.native_memory`

Node gives you `process.memoryUsage()` — five numbers, one of which (`external`) is a
single undifferentiated bucket. You cannot ask Node "how much of my RSS is the HTTP
parser and how much is zlib?"

The JVM can answer that question, with a per-subsystem breakdown, at runtime, on a
live process, with a before/after diff. That tool is Native Memory Tracking, and
learning to read it is half of this document.

**Verdict: NO ANALOGUE, and it is Java's advantage for once.**

---

## What is this?

"Off-heap memory" means: **memory the process owns that the Java garbage collector
does not manage.** It is still RSS. The kernel still counts it. The cgroup limit still
applies to it. The JVM simply does not move it, compact it, or trace it.

There are four ways your Java process ends up holding it.

**1. `ByteBuffer.allocateDirect(n)` — a `DirectByteBuffer`.**
Native memory obtained through `Unsafe.allocateMemory` (ultimately `malloc`). Handed
back by a `Cleaner` at some unpredictable future point. This is what every NIO
channel, every Netty allocation, and most zero-copy I/O path uses.

**2. `FileChannel.map(...)` — a `MappedByteBuffer`.**
A memory-mapped file. Different cost profile: pages are backed by the page cache, so
they can be evicted under pressure, but they still show in RSS while resident. Also
released by a `Cleaner`.

**3. `java.lang.foreign` — `Arena` and `MemorySegment`.**
`[JAVA 25]` The modern, supported, deterministic way. Also the way you call native
code without writing JNI.

**4. Everything the JVM itself allocates natively, plus every native library you load.**
Metaspace. The code cache. Thread stacks. GC data structures. Symbol tables. Compiler
arenas. The JDBC driver's native bits. `glibc`'s malloc arenas. Your process's RSS is
the sum of all of it, and **the heap is usually not even the largest term** in a
small container.

That last point is the one that gets services killed, so it gets its own equation:

```
RSS  =  committed Java heap
      + Metaspace + compressed class space
      + code cache (JIT-compiled code)
      + thread stacks (touched pages, per thread)
      + GC data structures (card table, remembered sets, mark bitmaps)
      + direct byte buffers + mapped buffers
      + JVM internal (symbols, arenas, NMT's own bookkeeping)
      + native libraries and their malloc arenas
      + the JVM binary and mapped shared libraries
```

**Setting `-Xmx` equal to the container limit means every single one of those other
terms is over budget by construction.** That is Trap 4 below, and it is the most
common JVM-in-a-container mistake in the industry.

---

## Why does it matter?

**1. It is the failure that no dashboard shows.**

Your Grafana panel plots `jvm_memory_used_bytes{area="heap"}`. It reads 40%. The pod
restarts. The panel still reads 40% right up to the restart, because the number was
never wrong — it was answering a different question than the one you needed answered.

**2. The process dies without leaving evidence.**

`SIGKILL` cannot be caught. `-XX:+HeapDumpOnOutOfMemoryError` does not fire, because no
`OutOfMemoryError` was ever thrown. Your shutdown hook does not run. Your in-flight
orders are lost. You get a pod event and an exit code.

**3. It is where the modern Java stack actually lives.**

Netty allocates direct buffers by default. So does the JDK's own NIO. So does gRPC.
So does the Postgres JDBC driver for large results in some configurations. So does
every "zero-copy" claim in every library README. `orderflow` under load is running
direct-buffer traffic on every request whether or not you wrote a line of NIO.

**4. `[JAVA 25]` FFM changes the design space.**

Deterministic native lifetime, safe use-after-free semantics, and native calls without
JNI make off-heap a *design tool* rather than a hazard. A 4 GB product-price index
held off-heap in an `Arena` costs the GC nothing, because the GC never traces it. That
is a real architectural option for `orderflow`'s catalogue, and it is now supported
API rather than `sun.misc.Unsafe`.

**5. It is a top-tier senior interview filter.**

"Heap is at 40%, why did Kubernetes OOMKill the pod?" is asked constantly, and the
answers separate cleanly into "I do not know" and "here is the RSS equation and here
is how I attribute each term."

---

## Machine-level reality

Everything in this section is checkable. Where I am not certain, I say so in one line
and give you the command.

### What a `DirectByteBuffer` actually is

```java
ByteBuffer buf = ByteBuffer.allocateDirect(16 * 1024 * 1024);   // 16 MB
```

What exists after that line:

```
JAVA HEAP                                   NATIVE MEMORY (outside the heap)
+-----------------------------------+       +--------------------------------------+
| DirectByteBuffer instance         |       |                                      |
|   long address  ---------------------->   |   16,777,216 bytes                   |
|   int  capacity = 16777216        |       |   obtained via Unsafe.allocateMemory |
|   int  position, limit, mark      |       |   (ultimately malloc / mmap)         |
|   Object att                      |       |                                      |
|   Cleaner cleaner  ---+           |       +--------------------------------------+
+-----------------------|-----------+
                        |
                        v
    jdk.internal.ref.Cleaner extends PhantomReference<Object>
      referent  = the DirectByteBuffer above
      thunk     = Deallocator { address, capacity }  -> calls Unsafe.freeMemory
      queue     = a JDK-internal reference queue
```

The Java object is on the order of a few dozen bytes. The thing it owns is 16 MB.
**That ratio is the entire bug.** A collector making decisions about a few dozen bytes
cannot make good decisions about 16 MB.

### The `Cleaner` mechanism, step by step

This is the part people hand-wave. Do not hand-wave it — the interview follow-up is
always "so what triggers the free, exactly?"

1. `PhantomReference` is the weakest reference kind. A phantom reference's `get()`
   always returns `null` — you can never resurrect the referent through it. Its only
   purpose is "tell me *after* this object has been determined unreachable".
2. When the collector determines the `DirectByteBuffer` is unreachable, it **clears**
   the phantom reference and **enqueues** it on its reference queue. The native memory
   is still allocated at this instant. Nothing has been freed.
3. A JVM thread then drains the queue and runs the cleaning action. For
   `DirectByteBuffer`, `jdk.internal.ref.Cleaner` is processed by the JDK's
   reference-handler machinery; for the public `java.lang.ref.Cleaner` (Java 9+) it is
   a dedicated cleaner thread you can name and observe.
4. The action calls `Unsafe.freeMemory(address)`. **Now** the native memory is
   released back to the allocator.
5. The allocator (usually `glibc` malloc) may or may not return the pages to the
   kernel. Freeing in the C sense does not necessarily reduce RSS. Large blocks
   obtained via `mmap` typically do return; blocks from the heap arena often do not,
   because malloc keeps them for reuse.

Count the conditions: the object must become unreachable, **and** a collection must
run that notices, **and** the reference queue must be drained, **and** the allocator
must decide to give the pages back. That is four independent things between "I dropped
the reference" and "RSS goes down".

**Step 5 is the one that surprises people most.** You can watch NMT's committed number
for a direct-buffer category fall while `docker stats` RSS does not move, because
glibc kept the arena. On Linux you can sometimes force the issue with
`malloc_trim(0)`, and switching the allocator (`jemalloc`, `tcmalloc` via
`LD_PRELOAD`) changes the behaviour materially. **I am not going to assert which
allocator your base image uses or how it behaves** — Alpine's musl and Debian's glibc
differ substantially here. Settle it with `ldd --version` inside your container and,
if you suspect fragmentation, A/B a run with `LD_PRELOAD` set to jemalloc.

### Why `System.gc()` is not a fix

Every team that hits this proposes `System.gc()`. Here is the full case against it,
which you should be able to give without notes.

1. **It is a request, not a command.** The spec says the JVM makes a best effort.
2. **`-XX:+DisableExplicitGC` turns it into a no-op** — and it is a *popular* flag,
   often set precisely to stop badly-behaved libraries from calling it. If someone set
   it, your fix is silently dead. Worse: it also disables the *JVM's own* internal
   `System.gc()` call inside `Bits.reserveMemory`, which is the only automatic relief
   valve for direct-memory exhaustion. Setting `-XX:+DisableExplicitGC` on a
   NIO-heavy service converts a survivable pause into `OutOfMemoryError: Direct buffer
   memory`.
3. **It is a full, stop-the-world collection on most collectors.** On a multi-gigabyte
   heap that is a latency event you did not budget for, and on `orderflow` at baseline
   load it will show up directly in your p99.
4. **Enqueuing is not cleaning.** `System.gc()` returning does not mean the cleaners
   have run. Reference processing is asynchronous. You have to wait, and there is no
   API that tells you when it is done.
5. **It does not fix the actual defect.** If a reference is genuinely retained —
   a buffer sitting in a `static List`, a pooled object never returned — no number of
   collections will help, because the object is reachable. Reachability, not
   allocation, is the leak (Topic 79's mechanical statement, wearing a different hat).

The honest fix is one of exactly three things: **release the reference**, **bound the
allocation with `-XX:MaxDirectMemorySize`**, or **stop relying on the GC and use an
`Arena` you close yourself.**

### The direct-memory budget, and its dangerous default

There is a JVM-level budget for `ByteBuffer.allocateDirect`, enforced in
`java.nio.Bits`. Every direct allocation reserves against it; every deallocation
releases back to it.

```
-XX:MaxDirectMemorySize=512m
```

**When you do not set it, the JVM derives a default.** The long-standing behaviour is
that the default is taken from the maximum heap size — that is, roughly `-Xmx`.

> **Honest uncertainty, one line:** I am confident the default is *derived* rather
> than unlimited, and that it has historically tracked max heap, but I am not certain
> of the exact derivation on your specific JDK 25 build. **Settle it, do not trust
> me:** run `java -XX:+PrintFlagsFinal -version | grep -i MaxDirectMemorySize` — a
> value of `0` means "derive it", not "unlimited" — and then print the *derived*
> value at runtime from the `BufferPoolMXBean`, shown in the Hands-on section.

Two consequences fall straight out, and both are production incidents:

- **The default has nothing to do with your container limit.** If `-Xmx` is 1 GB and
  the container limit is 1.5 GB, the JVM will cheerfully let you reserve another 1 GB
  of direct memory before it complains. Heap 1 GB + direct 1 GB + everything else is
  well past 1.5 GB. The kernel kills you long before `OutOfMemoryError: Direct buffer
  memory` is thrown.
- **Raising `-Xmx` silently raises the direct-memory ceiling too.** A change that
  looks like "we gave it more heap" also doubled the amount of native memory the
  service is permitted to reserve. This is a genuinely surprising coupling and it
  catches experienced people.

**The rule:** in a container, always set `-XX:MaxDirectMemorySize` explicitly, at a
value you derived from the RSS equation. It is the bound almost nobody sets.

### Netty does its own thing, and you must know that

Netty is the direct-buffer heavyweight in the Java ecosystem, and it does not simply
call `ByteBuffer.allocateDirect`. Netty maintains its own pooled allocator
(`PooledByteBufAllocator`) with per-thread arenas, and on platforms where it can, it
allocates direct memory *without* the JDK's `Cleaner`, doing its own accounting under
`io.netty.maxDirectMemory`.

Practical consequences:

- Netty's usage may **not** count against `-XX:MaxDirectMemorySize`, so bounding the
  JDK budget does not bound Netty.
- Netty's own counter is what you must watch: expose
  `PooledByteBufAllocatorMetric` (arena counts, chunk sizes, active allocations)
  through Micrometer, or read it via JMX.
- A Netty `ByteBuf` leak (a missing `release()`, since Netty is reference-counted)
  looks exactly like this topic's failure: flat heap, rising RSS. Netty ships a leak
  detector — `-Dio.netty.leakDetection.level=paranoid` — which you should run in a
  load test, never in production, because it samples every buffer.

> **Honest uncertainty:** Netty's allocation strategy and whether it bypasses the JDK
> `Cleaner` depend on the Netty version, the platform, and whether `Unsafe` is
> available. **Settle it for your build:** log
> `io.netty.util.internal.PlatformDependent.usedDirectMemory()` and
> `PlatformDependent.maxDirectMemory()` at startup, and set
> `-Dio.netty.leakDetection.level=paranoid` in a load-test run.

This is Topic 103's material in full. You need it here because `orderflow`'s HTTP
layer is Netty the moment you touch WebFlux or a reactive HTTP client.

### `[JAVA 25]` The FFM API: `Arena`, `MemorySegment`, and deterministic lifetime

```java
import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;
import java.lang.foreign.ValueLayout;

try (Arena arena = Arena.ofConfined()) {
    MemorySegment prices = arena.allocate(ValueLayout.JAVA_LONG, 100_000);
    prices.setAtIndex(ValueLayout.JAVA_LONG, 4471, 1999L);
    long price = prices.getAtIndex(ValueLayout.JAVA_LONG, 4471);
}   // freed here, on this line, every time
```

The four arena kinds and what each buys:

| Arena | Lifetime | Thread rules | Use it for |
|---|---|---|---|
| `Arena.ofConfined()` | closed by `close()`, i.e. end of try-with-resources | **one thread only**; other threads get `WrongThreadException` | request-scoped buffers, per-call scratch space |
| `Arena.ofShared()` | closed by `close()` | any thread may access; `close()` performs a thread handshake to prove nobody is inside | a shared off-heap index, closed at shutdown |
| `Arena.ofAuto()` | GC-managed | any thread | when you genuinely want `DirectByteBuffer` semantics — non-deterministic, but safe |
| `Arena.global()` | never | any thread | process-lifetime constants; you are choosing never to free |

**The safety property that makes this usable:** access after close throws
`IllegalStateException`, not a segmentation fault. That is the difference between a
bug you find in a stack trace and a bug that takes down the JVM with a `hs_err_pid`
file. `sun.misc.Unsafe` gave you the second one. FFM gives you the first.

`Arena.ofShared().close()` is not free — it performs a thread-state handshake to
prove that no thread is currently inside a segment access. That is a real cost at
close time and a real safety guarantee. Confined arenas skip it, which is why they
are the default choice for per-request work.

**The status question, answered carefully for both versions:**

| JDK | `java.lang.foreign` status | What you must do |
|---|---|---|
| **21** | **Preview API** (it was in preview through 21; it was finalised in a later release, not in 21) | compile *and* run with `--enable-preview --release 21`; the API may change between releases; not appropriate for production |
| **25** | **Final and standard** | just use it |

> **Say this precisely in an interview:** "FFM was a preview API on 21 and was
> finalised in a later release, so on a 21 baseline I would not ship it — I would use
> `DirectByteBuffer` with an explicit `MaxDirectMemorySize` and a bounded pool, and
> plan the FFM migration for the 25 runtime."

**Settle the status on your machine in ten seconds** — compile a file using `Arena`
with no flags:

```bash
javac ArenaProbe.java
```

| What you see | What it means |
|---|---|
| Compiles cleanly | FFM is a final API on this JDK. You are on 22+ (25 in your case). |
| `error: Arena is a preview API and is disabled by default` | FFM is preview here. You are on 21. Add `--release 21 --enable-preview` to compile and `--enable-preview` to run — and do not ship it. |
| `error: package java.lang.foreign does not exist` | You are on a JDK older than 19. Not your situation, but this is what it looks like. |

**The 21-compatible fallback, written out**, because you have a 21 language baseline:

```java
// Java 21, no preview flags. Deterministic-ish release of a direct buffer.
public final class ScratchBuffer implements AutoCloseable {

    private final ByteBuffer buffer;
    private volatile boolean closed;

    private ScratchBuffer(int bytes) {
        this.buffer = ByteBuffer.allocateDirect(bytes);
    }

    public static ScratchBuffer of(int bytes) { return new ScratchBuffer(bytes); }

    public ByteBuffer buffer() {
        if (closed) throw new IllegalStateException("ScratchBuffer already closed");
        return buffer;
    }

    @Override public void close() { closed = true; /* return to a pool here */ }
}
```

Note what this fallback **cannot** do, and say it out loud: it cannot actually free the
native memory at `close()`. There is no supported API on 21 to free a
`DirectByteBuffer` early. The honest 21 pattern is therefore **pooling** — allocate a
fixed number of buffers at startup, hand them out, take them back, never let the count
grow. A bounded pool converts an unbounded native leak into a fixed, known cost. That
is the whole reason Netty has a pooled allocator.

> There are `sun.misc.Unsafe` and reflection tricks to invoke the cleaner early. They
> work today, they are unsupported, `Unsafe`'s memory-access methods are being
> deprecated for removal, and they will break. Do not put them in `orderflow`.

### `[JAVA 25]` FFM's other half: calling native code without JNI

Not the focus of this topic, but you need to be able to say what it is:

```java
Linker linker = Linker.nativeLinker();
SymbolLookup stdlib = linker.defaultLookup();
MethodHandle strlen = linker.downcallHandle(
        stdlib.find("strlen").orElseThrow(),
        FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS));

try (Arena arena = Arena.ofConfined()) {
    MemorySegment sku = arena.allocateFrom("SKU-4471");
    long len = (long) strlen.invokeExact(sku);
}
```

No C stub, no `System.loadLibrary`, no header generation. `Linker`, `SymbolLookup`,
`FunctionDescriptor` and `MethodHandle` replace all of JNI's boilerplate.

> **One-line uncertainty:** recent JDKs restrict "restricted methods" like
> `Linker.downcallHandle` — calling them without `--enable-native-access=ALL-UNNAMED`
> produces a warning on some releases and is slated to become an error. I am not
> certain of the exact behaviour on your JDK 25 build. **Settle it:** run the above
> once with no flag and once with `--enable-native-access=ALL-UNNAMED` and compare
> stderr.

### Native Memory Tracking: what it does and does not see

```
-XX:NativeMemoryTracking=off        (default)
-XX:NativeMemoryTracking=summary    (per-category totals — what you want)
-XX:NativeMemoryTracking=detail     (adds call-site attribution — heavier)
```

NMT must be enabled **at JVM start**. You cannot turn it on later with `jcmd`. That
is the single most common operational disappointment in this topic: the pod is
misbehaving *now*, and NMT is off, and you must redeploy to get it. **Turn it on by
default in your load-test profile**, and consider leaving `summary` on in production —
its overhead is real but modest, and the JVM reports its own bookkeeping cost back to
you under a `Native Memory Tracking` category so you can see exactly what it costs.

The categories NMT reports include: `Java Heap`, `Class`, `Thread`, `Code`, `GC`,
`Compiler`, `Internal`, `Other`, `Symbol`, `Native Memory Tracking`, `Arena Chunk`,
`Metaspace`, `Logging`, `Arguments`, `Module`, `Safepoint`, `Synchronization`,
`Serviceability`, `Object Monitors`, `Shared class space`. The exact set varies by
JDK version and by which collector is running.

**Which category direct byte buffers land in is the question you actually care about,
and I am not going to guess it for your JDK.** Historically, memory obtained through
`Unsafe.allocateMemory` has been reported under `Other` on recent JDKs and under
`Internal` on older ones. **Settle it experimentally in sixty seconds:** take an NMT
baseline, allocate exactly 512 MB of direct buffers, take a `summary.diff`, and read
which category moved by roughly 512 MB. That is the failure drill below, and it is a
far better way to learn this than memorising a table.

**Three things NMT does NOT see, each of which has bitten someone:**

1. **Memory allocated by third-party native libraries.** A JNI library calling
   `malloc` directly is invisible to NMT. If NMT's total is far below RSS, suspect a
   native library.
2. **`glibc` malloc arena overhead and fragmentation.** NMT counts what the JVM asked
   for, not what the allocator kept.
3. **Anything before NMT initialises.** A small, constant, uninteresting gap.

And one thing NMT reports that people misread: **`reserved` versus `committed`.**

- **Reserved** = address space claimed. Costs no physical memory. A JVM routinely
  reserves far more than it uses.
- **Committed** = backed by the OS, counted against your cgroup when touched.

**Compare `committed` to your container limit. Never `reserved`.** People panic at a
reserved number that is meaningless.

Even `committed` is not RSS. Committed-but-never-touched pages may not be resident.
`-XX:+AlwaysPreTouch` forces the heap to be touched at startup, which makes committed
and resident agree for the heap — at the cost of a slower start. In a container with a
hard limit, that trade is often worth taking, because it converts "OOMKilled at 3am
under peak load" into "fails to start in CI".

### The full RSS equation, with the NMT category for each term

This table is the payload of the whole document. Memorise the shape, not the numbers.

| RSS component | Roughly how big | NMT category | The flag that bounds it |
|---|---|---|---|
| Java heap (committed) | you chose it | `Java Heap` | `-Xmx` / `-XX:MaxRAMPercentage` |
| Metaspace + class space | grows with classes loaded; a Spring app loads a lot | `Class` (and `Metaspace`) | `-XX:MaxMetaspaceSize` — **unbounded by default** |
| JIT code cache | grows as methods compile; larger with an agent attached | `Code` | `-XX:ReservedCodeCacheSize` |
| Thread stacks | per-thread; touched pages only | `Thread` | `-Xss` × thread count — bound the **pools**, not the stack |
| GC structures | card table, remembered sets, mark bitmaps; scales with heap and collector | `GC` | implied by heap size and collector choice |
| JIT compiler arenas | transient, spikes during compilation | `Compiler` | `-XX:CICompilerCount` |
| Direct byte buffers | **unbounded by default in practice** | usually `Other` — **verify** | `-XX:MaxDirectMemorySize` |
| Mapped buffers | size of mapped file regions | usually `Other`/`Internal` — verify | your own discipline |
| Symbols, internal tables | modest, grows with class count | `Symbol`, `Internal` | — |
| Native libraries | JDBC, compression, crypto, Netty native transport | **invisible to NMT** | audit `pmap`, `lsof` |
| NMT's own bookkeeping | small | `Native Memory Tracking` | turn NMT off |

Read that table once more with `orderflow` in mind. A Spring Boot service with
Hibernate loads thousands of classes (Metaspace), runs a Tomcat pool of a couple of
hundred threads (Thread), JIT-compiles a large surface (Code), and does NIO on every
request (Other). **If you set `-Xmx` to the container limit, every one of those rows
is unfunded.**

---

## Example 1 — minimal

Twenty lines that demonstrate the whole mechanical statement.

```java
import java.lang.management.BufferPoolMXBean;
import java.lang.management.ManagementFactory;
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.List;

public class DirectLeakMinimal {

    // The retention. Remove this line and the failure disappears.
    private static final List<ByteBuffer> HELD = new ArrayList<>();

    public static void main(String[] args) throws Exception {
        BufferPoolMXBean directPool = ManagementFactory
                .getPlatformMXBeans(BufferPoolMXBean.class).stream()
                .filter(b -> b.getName().equals("direct"))
                .findFirst().orElseThrow();

        Runtime rt = Runtime.getRuntime();

        for (int i = 1; i <= 200; i++) {
            HELD.add(ByteBuffer.allocateDirect(16 * 1024 * 1024));   // 16 MB each

            long heapUsedMb  = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
            long directMb    = directPool.getMemoryUsed() / (1024 * 1024);
            long directCount = directPool.getCount();

            System.out.printf("iter=%3d  heapUsed=%4d MB   directBuffers=%3d  direct=%5d MB%n",
                    i, heapUsedMb, directCount, directMb);

            Thread.sleep(200);
        }
    }
}
```

```bash
java -Xmx256m DirectLeakMinimal
```

**What to look for:** the `heapUsed` column against the `direct` column.

| What you see | What it means |
|---|---|
| `heapUsed` stays low and roughly flat while `direct` climbs by 16 MB per iteration | **The mechanical statement, demonstrated.** The heap does not know about the native memory. This is the whole topic in one table. |
| `direct` stops climbing and the program throws `OutOfMemoryError: Direct buffer memory` | You hit the direct-memory budget. Note the message names *direct buffer memory*, not heap — that word is the diagnosis. Note also that the JVM tried `System.gc()` first and it did not help, because the buffers are genuinely reachable from `HELD`. |
| The process is killed by the OS before either happens | You are running inside a memory-constrained cgroup and the budget was larger than the limit. **This is the production failure, reproduced.** Exit code 137. |
| `heapUsed` also climbs steadily | Expected and harmless — the `ArrayList` and the wrapper objects cost a little heap. Compare the slopes: heap grows in kilobytes, direct in megabytes. |

**Now delete the `HELD` list** (allocate into a local variable and drop it each
iteration) and re-run. The `direct` column should rise and then fall as collections
happen and cleaners run — but **not immediately, and not smoothly**. That sawtooth,
and specifically the *lag* in it, is the non-determinism the whole topic is about.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline:

- **Container:** `--memory=2g`, `--cpus=2`, Kubernetes `limits.memory: 2Gi`.
- **Dataset:** 100k products, 1M orders, 5M order lines.
- **Load:** the recorded k6 mix — 70% catalogue read, 20% order read, 10% order
  placement — with p50/p95/p99 committed to `/docs/java/baselines/`.
- **JVM flags at baseline:** `-Xmx1g`, G1, no `MaxDirectMemorySize`, no NMT.
- **Requirement that triggers the incident:** product manager asks for a bulk export —
  `GET /orders/export?from=...&to=...` streaming a CSV of orders and lines to the
  client.

### The code that ships and kills the pod

```java
package com.orderflow.orders;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

import java.nio.ByteBuffer;
import java.nio.channels.WritableByteChannel;
import java.time.LocalDate;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@RestController
public class OrderExportController {

    private final OrderExportService exports;

    // "Reuse the buffer per export job so we do not reallocate." Sounds thrifty.
    private final Map<String, ByteBuffer> bufferByJob = new ConcurrentHashMap<>();

    public OrderExportController(OrderExportService exports) { this.exports = exports; }

    @GetMapping(value = "/orders/export", produces = MediaType.TEXT_PLAIN_VALUE)
    public void export(@RequestParam LocalDate from,
                       @RequestParam LocalDate to,
                       @RequestParam String jobId,
                       WritableByteChannel out) throws Exception {

        ByteBuffer buffer = bufferByJob.computeIfAbsent(
                jobId, id -> ByteBuffer.allocateDirect(32 * 1024 * 1024));   // 32 MB

        exports.streamCsv(from, to, buffer, out);
    }
}
```

Three ordinary-looking decisions combine into an outage:

1. `allocateDirect` — chosen because "zero-copy to the socket is faster". It is, and
   that is not the problem.
2. 32 MB per buffer — chosen because the largest export seen in dev was 28 MB.
3. `bufferByJob` keyed by `jobId` — chosen to avoid reallocating on retry. **Nothing
   ever removes an entry.**

### What actually happens under the Topic 65 baseline

The export endpoint is not even in the k6 scenario mix. It is called by an internal
reporting job, a handful of times an hour, with a fresh `jobId` each time.

- Each call adds a permanent 32 MB native allocation.
- The map entry is a few dozen bytes of heap. **Ten calls cost 320 MB of RSS and
  under a kilobyte of heap.**
- G1 sees no pressure. The heap occupancy graph is a flat line. No GC log line is
  unusual. `jvm_memory_used_bytes{area="heap"}` sits where it always sat.
- Nothing throws. `MaxDirectMemorySize` was never set, so the derived budget is
  roughly the 1 GB max heap — far above what the container can survive.
- RSS climbs: 1 GB heap (committed and pre-touched by activity) + Metaspace + code
  cache + 200 Tomcat thread stacks + G1 structures + Netty/NIO direct traffic + the
  accumulating export buffers.
- At some point during the next load run, the cgroup limit is crossed and the kernel
  sends `SIGKILL`.

**The observable symptom, in the order you will actually meet it:**

1. A pod restart with no application error log. The last log line is a normal request.
2. `kubectl describe pod` shows `Last State: Terminated`, `Reason: OOMKilled`,
   `Exit Code: 137`.
3. Your heap dashboard shows 40% right up to the restart.
4. No heap dump exists, because `-XX:+HeapDumpOnOutOfMemoryError` never fired — there
   was no `OutOfMemoryError`.
5. `k6` reports a spike of 5xx and connection errors for the duration of the restart,
   and the p99 for that run is outside your Topic 65 ±10% gate.
6. It happens again the next day, at a different time, correlated with nothing anyone
   can see. The reporting job's schedule is not in anyone's mental model.

**Why the team will blame the wrong thing:** the restart correlates with load, and the
heap looks fine, so the first three hypotheses are "the database", "a thread leak",
and "the last release". None of them are it. Without the RSS equation you will not
even form the correct hypothesis.

### The fix, in three layers

**Layer 1 — make the failure loud instead of fatal.** Do this first, before you know
the cause.

```
-XX:MaxDirectMemorySize=256m
-XX:NativeMemoryTracking=summary
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps
```

Now the same bug throws `OutOfMemoryError: Direct buffer memory` with a stack trace
pointing at `OrderExportController.export`, instead of vanishing under `SIGKILL`. You
have converted an unattributable pod restart into a named line of code. **That trade
is almost always worth it**, and it is the single highest-value flag in this document.

**Layer 2 — fix the actual defect: the unbounded retention.**

```java
package com.orderflow.orders;

import java.nio.ByteBuffer;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

/**
 * A bounded pool of direct buffers. The native footprint of this class is
 * exactly POOL_SIZE * BUFFER_BYTES, forever, and that number goes in the
 * RSS budget alongside heap, metaspace and thread stacks.
 */
public final class ExportBufferPool {

    private static final int POOL_SIZE    = 4;
    private static final int BUFFER_BYTES = 8 * 1024 * 1024;   // 8 MB, not 32

    private final BlockingQueue<ByteBuffer> pool = new ArrayBlockingQueue<>(POOL_SIZE);

    public ExportBufferPool() {
        for (int i = 0; i < POOL_SIZE; i++) {
            pool.add(ByteBuffer.allocateDirect(BUFFER_BYTES));
        }
    }

    /** Blocks with a timeout rather than allocating. Backpressure, not growth. */
    public ByteBuffer borrow() throws InterruptedException {
        ByteBuffer b = pool.poll(2, TimeUnit.SECONDS);
        if (b == null) {
            throw new ExportCapacityExceededException("no export buffer available");
        }
        return b.clear();
    }

    public void giveBack(ByteBuffer b) { pool.offer(b.clear()); }

    public long nativeFootprintBytes() { return (long) POOL_SIZE * BUFFER_BYTES; }
}
```

```java
@GetMapping(value = "/orders/export", produces = MediaType.TEXT_PLAIN_VALUE)
public void export(@RequestParam LocalDate from,
                   @RequestParam LocalDate to,
                   WritableByteChannel out) throws Exception {

    ByteBuffer buffer = pool.borrow();
    try {
        exports.streamCsv(from, to, buffer, out);
    } finally {
        pool.giveBack(buffer);          // the missing half of the original code
    }
}
```

Four things changed and each removes a distinct hazard:

1. **Fixed count.** Native footprint is 32 MB total, known at startup, and it goes on
   the RSS budget line. It cannot grow.
2. **8 MB not 32 MB.** The buffer is streaming scratch space, not a whole-export
   staging area. Streaming means you never need the whole export resident.
3. **`jobId` is gone.** It was never the right key; it was the mechanism of the leak.
4. **Exhaustion is backpressure, not allocation.** The fifth concurrent export waits,
   then fails with a domain exception that maps to a 429 or 503 (Topic 46's
   `ProblemDetail`). That is a product decision made explicitly, which is what
   backpressure always is.

**Layer 3 — `[JAVA 25]` the version with deterministic lifetime.**

```java
import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;

@GetMapping(value = "/orders/export", produces = MediaType.TEXT_PLAIN_VALUE)
public void export(@RequestParam LocalDate from,
                   @RequestParam LocalDate to,
                   WritableByteChannel out) throws Exception {

    try (Arena arena = Arena.ofConfined()) {
        MemorySegment scratch = arena.allocate(8L * 1024 * 1024);
        exports.streamCsv(from, to, scratch.asByteBuffer(), out);
    }   // native memory released HERE, on this line, on this thread, every time
}
```

No pool. No `Cleaner`. No dependence on when the GC feels like running. The buffer is
freed at the closing brace whether the export succeeded, threw, or the client
disconnected mid-stream.

**But read the trade honestly before you reach for it.** Per-request `Arena.allocate`
means a `malloc` and a `free` on every request, whereas the pool reuses. Under
`orderflow`'s export rate — a handful an hour — the allocation cost is irrelevant and
determinism wins outright. Under a rate of thousands per second, pool. **Deterministic
free and zero allocation are different goals; FFM gives you the first, pooling gives
you the second, and you can have both by pooling `MemorySegment`s from a single
long-lived shared `Arena`.**

**The 21-compatible version of Layer 3 is Layer 2.** On a 21 baseline there is no
supported deterministic free, so a bounded pool is the answer, and that is exactly why
the pool pattern is everywhere in Java NIO code.

### The budget line this produces

Write this down for `orderflow` and keep it in the repo next to the baselines:

```
container limit                 2048 MB
  - max heap  (-Xmx)            1024 MB
  - metaspace (-XX:MaxMetaspaceSize)   ??? MB   <- MEASURE with NMT, do not guess
  - code cache                        ??? MB   <- MEASURE
  - thread stacks (200 x -Xss)        ??? MB   <- MEASURE, and bound the pool
  - GC structures                     ??? MB   <- MEASURE
  - direct buffers (-XX:MaxDirectMemorySize)  256 MB  <- you CHOSE this
  - native libs + allocator slack     ??? MB   <- RSS minus NMT total
  ------------------------------------------------
  = headroom                          must be > 0, and > 10% for safety
```

**Every `???` is an NMT command away.** A budget with measured numbers in it is the
artefact that turns this topic from knowledge into engineering, and it is the direct
input to Topic 129's capacity model.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — heap at 40% while the container is OOMKilled

**Wrong:** believing the heap graph is the memory graph.

```yaml
resources:
  limits:
    memory: 2Gi
```
```
-Xmx1g            # "half the container, plenty of room"
```

**Exact symptom, precisely:**

- `kubectl get pod` shows `RESTARTS` incrementing.
- `kubectl describe pod <name>` contains:
  ```
  Last State:  Terminated
    Reason:    OOMKilled
    Exit Code: 137
  ```
- The application log's last line is an ordinary request completing. **No exception, no
  stack trace, no shutdown-hook output.**
- The heap panel reads 40% at the moment of death and does not spike first.
- No heap dump on disk despite `-XX:+HeapDumpOnOutOfMemoryError` being set.
- On the node: `dmesg | grep -i "killed process"` names your JVM PID.
- `cat /sys/fs/cgroup/memory.events` inside a surviving pod shows a non-zero
  `oom_kill` counter (cgroup v2) — this is the cleanest in-container evidence.

**Root cause:** RSS is heap **plus** metaspace, code cache, thread stacks, GC
structures, direct buffers, JVM internals and native libraries. `SIGKILL` from the
cgroup OOM killer is delivered by the kernel to a process the JVM believes is
perfectly healthy. There is nothing for the JVM to report because from the JVM's point
of view nothing went wrong.

**Fix — in this order:**

1. Turn on NMT: `-XX:NativeMemoryTracking=summary`. Redeploy. You cannot diagnose this
   without it.
2. `jcmd <pid> VM.native_memory summary` and compare the **committed** total against
   the container limit.
3. Subtract: `RSS − NMT committed total` = memory the JVM did not allocate. If that
   gap is large, you have a native-library or allocator problem, not a JVM problem.
4. Bound every unbounded term explicitly: `-Xmx`, `-XX:MaxMetaspaceSize`,
   `-XX:MaxDirectMemorySize`, `-XX:ReservedCodeCacheSize`, and the size of every
   thread pool.
5. Only then re-size. Topic 82 is the full treatment.

**The rule to carry:** *heap is a term in the equation, not the equation.*

---

### Trap 2 — never setting `-XX:MaxDirectMemorySize`

**Wrong:** leaving it at the default because "the default is probably sensible".

**Exact symptom, and note that there are two completely different ones:**

- *If the derived budget is smaller than the container's headroom:* you get
  ```
  java.lang.OutOfMemoryError: Direct buffer memory
        at java.base/java.nio.Bits.reserveMemory(Bits.java:...)
        at java.base/java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:...)
        at java.base/java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:...)
  ```
  **This is the good outcome.** You have a stack trace, a heap dump, and a named line
  of code. Note the words `Direct buffer memory` — that phrase means "native", not
  "heap", and it means `MaxDirectMemorySize` and not `-Xmx` is the relevant bound.
- *If the derived budget is larger than the container's headroom* — the common case,
  because the default tracks max heap and not the container limit: **you never see the
  error at all.** The kernel kills you first. Trap 1's symptom.

**Root cause:** the default is derived from the JVM's own max heap, which is unrelated
to how much native memory your container can afford. Nobody set a number, so the
number is wrong by construction.

**Fix:**

```
-XX:MaxDirectMemorySize=256m
```

Pick the value from the RSS budget, not from a blog. Then verify at runtime with the
`BufferPoolMXBean` (Hands-on, Proof 2), and alert on
`jvm_buffer_memory_used_bytes{id="direct"}` crossing 80% of it — Micrometer exposes
this out of the box, and almost nobody graphs it.

**The subtle part worth saying in an interview:** raising `-Xmx` also raises the
default direct-memory ceiling. So "we gave it more heap" is silently also "we let it
reserve more native memory", and the container limit did not move. Setting
`MaxDirectMemorySize` explicitly breaks that coupling.

---

### Trap 3 — "we'll just call `System.gc()`"

**Wrong:**

```java
if (directPool.getMemoryUsed() > threshold) {
    System.gc();                    // "free the buffers"
    Thread.sleep(100);              // "give the cleaners a moment"
}
```

**Exact symptom — three distinct ones, and which you get depends on flags you may not
control:**

1. **It appears to work in dev and not in production.** Production has
   `-XX:+DisableExplicitGC` (a common platform default) and the call is a no-op.
   Nothing in the logs tells you this.
2. **p99 latency degrades in a pattern that correlates with nothing in your request
   mix.** Every trigger is a full stop-the-world collection. On `orderflow`'s baseline
   run this is directly visible as a p99 regression outside the ±10% gate, and in
   `-Xlog:gc` as `Pause Full (System.gc())`.
3. **Native memory does not drop anyway.** Because the buffers are reachable from a
   `static` collection, or because the reference queue has not been drained yet, or
   because glibc kept the freed pages in its arena and RSS did not move.

**Root cause:** four different failure modes wearing one costume.

- `System.gc()` is advisory and can be disabled.
- Enqueuing a phantom reference is not the same as running the cleaner.
- A reachable object is never collected, no matter how many collections you force.
- `free()` is not `munmap()`; RSS need not fall.

**Fix:** find the retention (a heap dump and MAT's dominator tree finds the *wrapper*
objects, which is enough to name the retaining path — Topic 79), then bound the
allocation with a pool, then bound the budget with `MaxDirectMemorySize`. If you
genuinely need release-on-demand, that is what `Arena` is for on 25.

**The one place `System.gc()` legitimately appears:** inside
`java.nio.Bits.reserveMemory`, called by the JVM itself as a last resort before
throwing `OutOfMemoryError: Direct buffer memory`. Which is exactly why
`-XX:+DisableExplicitGC` is more dangerous than it looks on a NIO-heavy service — it
removes the JVM's own relief valve.

---

### Trap 4 — setting `-Xmx` equal to the container memory limit

**Wrong:**

```yaml
resources: { limits: { memory: 1Gi } }
```
```
-Xmx1g
```

**Exact symptom:** the pod survives startup and light traffic, then gets OOMKilled the
first time it is genuinely loaded — typically during your Topic 65 baseline run, at
some point after warm-up, with no application error. Often it survives for hours and
dies during the *second* load run, once the code cache has filled and Metaspace has
finished growing.

**Root cause:** you funded exactly one term of the RSS equation and left every other
term with a budget of zero. Metaspace alone on a Spring Boot + Hibernate application
is substantial and is **unbounded by default**. Add the code cache, 200 thread stacks,
G1's remembered sets and card table, and NIO's direct buffers, and you are over the
limit by design.

The reason this survives light traffic is that all the unfunded terms *grow with
work*: Metaspace grows as more classes load, the code cache grows as more methods
compile, thread stacks grow as the pool grows to serve concurrency, and direct-buffer
usage grows with I/O. A quiet service never reaches its real footprint. This is why the
failure appears "randomly" and why it appears *later* rather than sooner.

**Fix:** size the heap from the **measured live set** (Topic 70) plus GC headroom, and
leave the rest of the container for the rest of the JVM. Then verify with NMT that the
committed total plus native-library slack fits under the limit with headroom. Topic 82
gives the full method and the `MaxRAMPercentage` alternative.

**The reviewer's heuristic, offered as a starting point and not a rule:** in a
container that exists to run one JVM, the heap is a *fraction* of the limit, not the
limit, and the fraction is smaller for small containers because the fixed costs do not
shrink. A 512 MB container has proportionally far less room for heap than a 4 GB one.
Measure; do not extrapolate.

---

### Trap 5 — a `MappedByteBuffer` you never unmapped, blamed on the page cache

**Wrong:**

```java
// Memory-map the product catalogue export for fast random reads.
FileChannel channel = FileChannel.open(catalogPath, StandardOpenOption.READ);
MappedByteBuffer catalog = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
channel.close();                       // closing the channel does NOT unmap
CATALOGS.put(version, catalog);        // and this retains it forever
```

**Exact symptom:** RSS grows with each catalogue version deployed and never falls.
`docker stats` and `kubectl top pod` both show it. The heap is flat. Someone says "it
is just the page cache, that is reclaimable, ignore it" — and they are **half right**,
which is why this one runs for months before anyone fixes it.

The half that is right: clean, file-backed pages *are* reclaimable under memory
pressure. The half that is wrong: **the cgroup memory limit counts page-cache pages
charged to the cgroup**, and while the kernel will try to reclaim them before killing
you, the mapping itself is an address-space reservation the JVM holds and will not
release. On top of that, the old versions are unreachable-but-not-yet-cleaned, or in
this code, reachable forever from `CATALOGS`.

**Root cause:** closing the `FileChannel` does not unmap. The mapping is released by a
`Cleaner`, exactly like a `DirectByteBuffer` — same non-determinism, same
invisibility to GC heuristics — and here it is not even unreachable.

**Fix on 21:** bound the number of retained mappings; evict old versions from
`CATALOGS` explicitly; accept that the actual unmapping is non-deterministic and size
the container for at least two generations of catalogue.

**Fix on 25:** map through an `Arena` so the unmap is deterministic.

```java
try (Arena arena = Arena.ofShared()) {
    MemorySegment catalog = channel.map(
            FileChannel.MapMode.READ_ONLY, 0, channel.size(), arena);
    // ... serve reads from `catalog` ...
}   // unmapped HERE
```

For a long-lived catalogue you would hold the `Arena` as a field and `close()` it when
swapping versions — which gives you an explicit, testable "the old mapping is gone"
moment that the 21 version cannot express at all.

**Diagnostic that settles it in one command:** `pmap -x <pid> | sort -k3 -n | tail -30`
shows the largest mappings with their file backing. A file path in that output that you
recognise as a catalogue version you deployed last month is the entire diagnosis.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and no container runtime, so I
print no output and claim nothing as captured. What follows is the exact command, what
to look for, and how to read each result you might get.

### Setup

```bash
mkdir -p ~/java-lab/80 && cd ~/java-lab/80
java --version                 # expect 21 or 25; note which, it changes the FFM answer
```

### Proof 0 — which JDK, and is FFM final here?

`ArenaProbe.java`:

```java
import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;
import java.lang.foreign.ValueLayout;

public class ArenaProbe {
    public static void main(String[] args) {
        try (Arena arena = Arena.ofConfined()) {
            MemorySegment seg = arena.allocate(1024);
            seg.set(ValueLayout.JAVA_INT, 0, 4471);
            System.out.println("read back: " + seg.get(ValueLayout.JAVA_INT, 0));
        }
        System.out.println("arena closed cleanly");
    }
}
```

```bash
javac ArenaProbe.java && java ArenaProbe
```

| What you see | What it means |
|---|---|
| Compiles and runs, prints `4471` then `arena closed cleanly` | FFM is **final** on this JDK. You may use `Arena` in ordinary code. |
| `error: ... is a preview API and is disabled by default` | FFM is **preview** here — you are on 21. Retry with `javac --release 21 --enable-preview ArenaProbe.java` and `java --enable-preview ArenaProbe`. It will work, and you should still not ship it. |
| A warning about restricted methods or native access | Harmless for pure-memory use; it matters for the `Linker` downcall example. Re-run with `--enable-native-access=ALL-UNNAMED` and compare. |

Now prove the safety property — add this after the try-with-resources:

```java
MemorySegment escaped;
try (Arena arena = Arena.ofConfined()) { escaped = arena.allocate(1024); }
System.out.println(escaped.get(ValueLayout.JAVA_INT, 0));   // use after close
```

**What to look for:** an `IllegalStateException` mentioning that the segment's scope is
already closed.

| What you see | What it means |
|---|---|
| `IllegalStateException` naming a closed scope | **This is the headline FFM safety property.** Under `sun.misc.Unsafe` the same mistake is a segfault and an `hs_err_pid` file. A catchable exception is the entire value proposition. |
| A JVM crash / `hs_err_pid` file | You are not using FFM — check you did not fall back to `Unsafe` somewhere. |

### Proof 1 — the heap/native divergence

Run `DirectLeakMinimal` from Example 1:

```bash
java -Xmx256m DirectLeakMinimal
```

Read the table as described in Example 1. This is the falsifiable form of the
mechanical statement: two columns, one flat, one climbing.

### Proof 2 — find the *actual* direct-memory budget on your JVM

Do not trust my description of the default. Print it.

`DirectBudget.java`:

```java
import java.lang.management.BufferPoolMXBean;
import java.lang.management.ManagementFactory;
import java.nio.ByteBuffer;

public class DirectBudget {
    public static void main(String[] args) {
        System.out.println("Runtime.maxMemory (approx -Xmx) = "
                + Runtime.getRuntime().maxMemory() / (1024 * 1024) + " MB");

        for (BufferPoolMXBean pool
                : ManagementFactory.getPlatformMXBeans(BufferPoolMXBean.class)) {
            System.out.printf("pool=%-8s count=%d used=%d MB capacity=%d MB%n",
                    pool.getName(), pool.getCount(),
                    pool.getMemoryUsed() / (1024 * 1024),
                    pool.getTotalCapacity() / (1024 * 1024));
        }

        // Binary-search the ceiling: keep allocating until it refuses.
        long allocated = 0;
        try {
            while (true) {
                ByteBuffer.allocateDirect(32 * 1024 * 1024);   // deliberately dropped
                allocated += 32;
                if (allocated % 320 == 0) System.out.println("reserved ~" + allocated + " MB");
            }
        } catch (OutOfMemoryError e) {
            System.out.println("REFUSED at ~" + allocated + " MB : " + e.getMessage());
        }
    }
}
```

```bash
java -Xmx512m  DirectBudget
java -Xmx1g    DirectBudget
java -Xmx512m -XX:MaxDirectMemorySize=128m DirectBudget
```

**What to look for:** the `REFUSED at` number in each of the three runs, compared to
each other.

| What you see | What it means |
|---|---|
| Run 2 refuses at roughly double run 1 | **The default budget tracks max heap.** You have just proved the coupling that makes "we raised `-Xmx`" also mean "we raised the native ceiling". This is the most important line in the proof. |
| Run 3 refuses at roughly 128 MB regardless of `-Xmx` | `MaxDirectMemorySize` decouples them. This is the flag you set in production. |
| The message says `Direct buffer memory` | Confirms it is the NIO budget and not the heap. Different word, different flag, different fix. |
| The process is killed instead of throwing | Your machine or container ran out of physical memory before the JVM ran out of budget. **That is the production failure mode in miniature.** Re-run under a container with a limit and watch exit code 137. |
| Run 1 and run 2 refuse at the *same* number | Something is setting `MaxDirectMemorySize` already. Check `JAVA_TOOL_OPTIONS`, `_JAVA_OPTIONS`, and `jcmd <pid> VM.flags -all`. |

Also confirm the flag's own view:

```bash
java -XX:+PrintFlagsFinal -version | grep -i MaxDirectMemorySize
```

*Illustration of the format, not captured output* — a `PrintFlagsFinal` line has five
columns and you read them left to right:

```
   uintx MaxDirectMemorySize   = <n>   {product} {default}
   ^     ^                       ^      ^         ^
   type  flag name               value  category  ORIGIN
```

The last column is the one that matters: `{default}` means nobody set it, `{command
line}` means a flag did, `{ergonomic}` means the JVM derived it from the environment.
A value of `0` with origin `{default}` means "derive at runtime" — which is why you
needed the `BufferPoolMXBean` and the allocation experiment to see the real number.

### Proof 3 — turn on NMT and read a summary

```bash
java -XX:NativeMemoryTracking=summary -Xmx256m DirectLeakMinimal &
JVM_PID=$!
jcmd $JVM_PID VM.native_memory summary
```

*Illustration of the format, not captured output.* An NMT summary section looks like
this shape — I am showing you the **columns**, with `<n>` where a number would be:

```
Native Memory Tracking:

Total: reserved=<n>KB, committed=<n>KB

-                 Java Heap (reserved=<n>KB, committed=<n>KB)
                            (mmap: reserved=<n>KB, committed=<n>KB)

-                     Class (reserved=<n>KB, committed=<n>KB)
                            (classes #<n>)
                            (malloc=<n>KB #<n>)

-                    Thread (reserved=<n>KB, committed=<n>KB)
                            (thread #<n>)
                            (stack: reserved=<n>KB, committed=<n>KB)

-                      Code (reserved=<n>KB, committed=<n>KB)
-                        GC (reserved=<n>KB, committed=<n>KB)
-                     Other (reserved=<n>KB, committed=<n>KB)
```

**How to read those columns, which is the actual skill:**

| Column | Meaning | What to do with it |
|---|---|---|
| `reserved` | address space claimed; costs no physical memory | **Ignore it for capacity planning.** People panic at this number for no reason. |
| `committed` | backed by the OS; counted against the cgroup once touched | **This is the number you compare to the container limit.** |
| `Total: committed` | the JVM's own accounting of everything it allocated | Compare against RSS. The difference is native libraries and allocator slack. |
| `classes #<n>` | how many classes are loaded | A Spring app loads a lot. This is your Metaspace driver (Topic 68). |
| `thread #<n>` | live thread count | Multiply by `-Xss` for the reserved stack figure; note `committed` is usually far lower because untouched stack pages are not resident. |

**What to look for in *your* output:** which section's `committed` is largest after
`Java Heap`. On a small container it is very often `Class` or `Thread`, not anything
you were thinking about.

### Proof 4 — the diff, which is the tool that actually solves incidents

A single summary tells you the state. A **diff** tells you what changed, which is the
question you actually have.

```bash
# 1. take a baseline at a known-good moment
jcmd $JVM_PID VM.native_memory baseline

# 2. do the thing you suspect (run the export endpoint, run k6, wait 10 minutes)

# 3. ask what moved
jcmd $JVM_PID VM.native_memory summary.diff
```

The diff prints the same sections with `+<n>KB` / `-<n>KB` deltas next to each.

| What you see | What it means |
|---|---|
| A large positive delta under `Other` | Almost certainly direct byte buffers (`Unsafe.allocateMemory`). **Confirm** by allocating a known amount and re-diffing — see the Failure drill. |
| A large positive delta under `Class` | Metaspace growth — a classloader leak, or dynamic proxy/bytecode generation churn. Topics 67 and 81. |
| A large positive delta under `Thread` | Thread leak. Cross-check with `jcmd <pid> Thread.print | grep -c "^\"" `. Topic 98. |
| A large positive delta under `Code` | The JIT is still compiling, which is normal during warm-up, or an agent is generating a lot of code (Topic 81). |
| Almost no delta anywhere but RSS grew a lot | **The growth is outside the JVM's accounting.** Native library or allocator fragmentation. Go to `pmap`. |
| `Command executed with 'summary.diff' but NMT is off` | NMT was not enabled at start. You cannot turn it on now. Redeploy with the flag — and add it to your load profile permanently so this never wastes an incident again. |

### Proof 5 — compare NMT's total to actual RSS

Inside the container:

```bash
# resident set size in KB, from the kernel
grep VmRSS /proc/$JVM_PID/status

# the JVM's own accounting
jcmd $JVM_PID VM.native_memory summary | head -5

# the largest individual mappings, with file backing
pmap -x $JVM_PID | sort -k3 -n | tail -30
```

| What you see | What it means |
|---|---|
| RSS is a little above NMT committed | Normal and expected. The gap is the JVM binary, shared libraries, and allocator overhead. |
| RSS is far above NMT committed | Memory the JVM did not allocate: a JNI/native library, or `glibc` arena fragmentation. `pmap` will show large anonymous regions or a recognisable `.so`. |
| RSS is *below* NMT committed | Also normal — committed pages that were never touched are not resident. `-XX:+AlwaysPreTouch` makes the heap's committed and resident agree. |
| `pmap` shows a file you recognise (a catalogue, a mapped index) | Trap 5. You have found an unreleased `MappedByteBuffer`. |

### Proof 6 — watch a `Cleaner` actually run

The public `java.lang.ref.Cleaner` lets you observe the mechanism directly, which
`DirectByteBuffer`'s internal one does not.

```java
import java.lang.ref.Cleaner;

public class CleanerTiming {

    static final Cleaner CLEANER = Cleaner.create();

    static class NativeThing implements AutoCloseable {
        private final Cleaner.Cleanable cleanable;
        NativeThing(int id) {
            // NOTE: the action must NOT capture `this`, or the object is never
            // unreachable and the cleaner never runs. This is THE Cleaner footgun.
            this.cleanable = CLEANER.register(this, () ->
                    System.out.println("  cleaned #" + id + " at " + System.nanoTime()));
        }
        @Override public void close() { cleanable.clean(); }   // deterministic path
    }

    public static void main(String[] args) throws Exception {
        System.out.println("dropping 5 references at " + System.nanoTime());
        for (int i = 0; i < 5; i++) { new NativeThing(i); }

        for (int round = 0; round < 5; round++) {
            System.out.println("round " + round + " (no gc yet)");
            Thread.sleep(500);
        }
        System.out.println("calling System.gc() at " + System.nanoTime());
        System.gc();
        Thread.sleep(2000);
        System.out.println("done");
    }
}
```

```bash
java CleanerTiming
```

**What to look for:** *when* the `cleaned #N` lines appear relative to the "dropping"
line and the `System.gc()` line.

| What you see | What it means |
|---|---|
| No `cleaned` lines during the five rounds, then all five after `System.gc()` | **The mechanical statement, timed.** Dropping the reference did nothing. A collection was required. That gap is where your native memory lived. |
| Some `cleaned` lines appear before `System.gc()` | A young collection happened on its own — fine, and it shows the timing is *unpredictable*, which is the same lesson. |
| No `cleaned` lines even after `System.gc()` | Either the objects are still reachable (check you did not keep them in a list), or your cleaning action accidentally captured `this`. Change the lambda to capture `this` deliberately and watch the cleaner never fire — that is worth doing once, because it is the most common `Cleaner` bug in real code. |

Now add `-XX:+DisableExplicitGC` and re-run:

```bash
java -XX:+DisableExplicitGC CleanerTiming
```

**What to look for:** the `cleaned` lines no longer follow `System.gc()`. You have just
proved Trap 3's first symptom, and you have proved why that flag is dangerous on a
NIO-heavy service.

### Proof 7 — the container-level view

```bash
docker run --rm --memory=512m --memory-swap=512m \
  -v "$PWD":/lab -w /lab eclipse-temurin:25 \
  java -Xmx256m -XX:NativeMemoryTracking=summary DirectLeakMinimal
```

In a second terminal:

```bash
docker stats --no-stream          # watch MEM USAGE / LIMIT climb
```

| What you see | What it means |
|---|---|
| The container's memory usage climbs to the limit and the process disappears | **You have reproduced the production incident locally.** Check `docker inspect <id> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` — expect `true` and `137`. |
| An `OutOfMemoryError: Direct buffer memory` instead | The derived budget was below the container's headroom. Raise `-Xmx` and it will flip to the kill — which is Trap 2's coupling, demonstrated. |
| Nothing happens; the loop finishes | Your limit was generous relative to 200 × 16 MB. Raise the iteration count or lower `--memory`. |

---

## Failure drill

**Mandatory.** Do not read the attribution table until you have produced the numbers
yourself. The point is not the knowledge — it is the memory of watching RSS climb next
to a heap graph that never moved.

### The scenario, exactly as assigned

Allocate direct buffers in a loop without releasing the references. Watch RSS grow
while heap stays flat. Attribute it with `-XX:NativeMemoryTracking=summary` plus
`jcmd <pid> VM.native_memory summary.diff`.

### Part A — the standalone version, to learn the tools

`DirectDrill.java`:

```java
import java.lang.management.BufferPoolMXBean;
import java.lang.management.ManagementFactory;
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.List;

public class DirectDrill {

    private static final List<ByteBuffer> HELD = new ArrayList<>();
    private static final int CHUNK_MB = 32;

    public static void main(String[] args) throws Exception {
        System.out.println("pid = " + ProcessHandle.current().pid());
        System.out.println("PRESS ENTER after you have taken the NMT baseline...");
        System.in.read();

        BufferPoolMXBean direct = ManagementFactory
                .getPlatformMXBeans(BufferPoolMXBean.class).stream()
                .filter(b -> b.getName().equals("direct")).findFirst().orElseThrow();
        Runtime rt = Runtime.getRuntime();

        int target = Integer.parseInt(args.length > 0 ? args[0] : "16");   // chunks
        for (int i = 1; i <= target; i++) {
            HELD.add(ByteBuffer.allocateDirect(CHUNK_MB * 1024 * 1024));
            System.out.printf("allocated %4d MB total | heapUsed=%4d MB | directPool=%4d MB%n",
                    i * CHUNK_MB,
                    (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024),
                    direct.getMemoryUsed() / (1024 * 1024));
            Thread.sleep(250);
        }

        System.out.println("ALLOCATION COMPLETE. Take the summary.diff now, then ENTER.");
        System.in.read();
    }
}
```

**The commands, in order:**

```bash
# terminal 1 — start with NMT on and a small heap so the divergence is stark
javac DirectDrill.java
java -Xmx256m -XX:NativeMemoryTracking=summary DirectDrill 16

# terminal 2 — note the pid printed above, then:
PID=<the pid>
jcmd $PID VM.native_memory baseline
grep VmRSS /proc/$PID/status               # record this number
# ... press ENTER in terminal 1, let it allocate 512 MB ...
grep VmRSS /proc/$PID/status               # record this number
jcmd $PID VM.native_memory summary.diff
jcmd $PID GC.heap_info
```

### Part B — what to write down before reading on

Five numbers and one sentence. Write them in your notes file, not in your head.

1. `VmRSS` before allocation.
2. `VmRSS` after allocating 512 MB.
3. The heap `used` from `GC.heap_info` after allocation.
4. The **NMT category with the largest positive delta** in `summary.diff`, and the
   size of that delta.
5. `directPool` MB from the last printed line.
6. One sentence: *why did the heap not grow?*

### Part C — how to read it

| What you see | What it means |
|---|---|
| RSS grew by roughly 512 MB; heap `used` grew by well under a megabyte | **The drill has fired.** This is the mechanical statement as two numbers you measured. Everything else in this document is an explanation of this pair. |
| The largest NMT delta is under `Other`, at roughly 512 MB | You have just *empirically determined* which NMT category direct buffers land in **on your JDK**. Write it down — it is the thing you will look for at 3am, and it is more reliable than any table I could have written. |
| The largest delta is under `Internal` instead | Also a correct answer, on some JDK versions. Same conclusion: you now know your JDK's answer. |
| The delta is spread across categories or does not obviously match 512 MB | Take a fresh `baseline` immediately before pressing ENTER, so warm-up noise is excluded from the diff. Warm-up allocates a lot of everything. |
| `directPool` from `BufferPoolMXBean` matches the NMT delta | Two independent instruments agreeing. **This is what a solid attribution looks like** — and being able to say "two independent measurements agree" is the difference between a diagnosis and a guess. |
| RSS grew by noticeably *more* than 512 MB | Allocator overhead, plus the JVM committing more heap as the `ArrayList` grows. Both are real and both belong in your RSS budget. |
| RSS grew by noticeably *less* than 512 MB | Some pages were reserved but never touched. `ByteBuffer.allocateDirect` zeroes the buffer, so this is unlikely — if you see it, check whether you are on a system that reports RSS lazily, and re-check after touching each buffer. |

### Part D — the same drill on `orderflow` under the Topic 65 baseline

This is the version that counts, because it is the one that produces a number you can
compare against a recorded baseline.

1. **Add the leaky endpoint** — Example 2's original `OrderExportController` with the
   `bufferByJob` map — to `orderflow`.
2. **Run the container with the flags on:**
   ```bash
   docker run --rm --memory=2g --cpus=2 \
     -e JAVA_TOOL_OPTIONS="-Xmx1g -XX:NativeMemoryTracking=summary -Xlog:gc:file=/tmp/gc.log:time,uptime" \
     -p 8080:8080 orderflow:drill
   ```
3. **Baseline NMT and RSS** before starting load.
4. **Start the recorded Topic 65 k6 scenario** — the same script, the same arrival
   rate, the same dataset. Do not change the mix; the whole point is comparability.
5. **In parallel**, call the export endpoint with a fresh `jobId` every 30 seconds:
   ```bash
   for i in $(seq 1 60); do
     curl -s -o /dev/null "http://localhost:8080/orders/export?from=2026-01-01&to=2026-01-31&jobId=job-$i"
     sleep 30
   done
   ```
6. **Every minute**, record: `grep VmRSS /proc/1/status` inside the container, the heap
   used from `jcmd 1 GC.heap_info`, and `docker stats --no-stream`.
7. **Take `jcmd 1 VM.native_memory summary.diff`** at the five-minute and fifteen-minute
   marks.

**What to capture and compare against `/docs/java/baselines/`:**

| Metric | Where it comes from | What you are looking for |
|---|---|---|
| p50 / p95 / p99 per endpoint | k6 summary | Are they within the ±10% gate *before* the kill? They usually are — that is the point. The service is fine until it is dead. |
| Heap used over time | `GC.heap_info` sampled | Flat. **Flat is the finding.** |
| RSS over time | `/proc/1/status` | A staircase, one step per export call. |
| NMT `Other` delta | `summary.diff` | Should track the number of export calls × 32 MB. |
| Container exit | `docker inspect --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` | `true` and `137`. |
| Heap dump | `/dumps` | **Empty.** No dump was written. Say why out loud. |

### Part E — fix it and re-run the baseline

Apply the Layer 1 + Layer 2 fix (bounded pool, `-XX:MaxDirectMemorySize=256m`, NMT
left on) and run the identical k6 scenario again.

**What the fix proves — and this is the point of the drill, not the fix:**

- The **same** load, the **same** export calls, the **same** container limit, and now
  the pod survives. The only variable was whether the native allocation was bounded.
- The NMT `Other` delta now flattens at 32 MB and stops. You can *see* the bound.
- If you push past the pool's capacity, you get a 503 from your own code with a
  `ProblemDetail` body — a product-visible, monitored, alertable event — instead of a
  `SIGKILL`. **Converting an unattributable kill into a named application error is the
  actual engineering deliverable here.**

Carry one sentence out of this drill: *the heap graph was never wrong; it was never
the graph I needed.*

---

## Measurement

### The instrument for each claim

Every claim in this document maps to an instrument that could falsify it. That mapping
is the difference between engineering and folklore.

| Claim | Instrument that makes it falsifiable |
|---|---|
| "Native memory grows while the heap is flat" | `BufferPoolMXBean` + `GC.heap_info`, sampled over time |
| "It is direct buffers specifically" | `jcmd VM.native_memory baseline` then `summary.diff` |
| "The JVM's accounting explains the RSS" | `NMT Total committed` vs `/proc/<pid>/status VmRSS` |
| "The gap is a native library" | `pmap -x <pid>` sorted by RSS column |
| "The budget defaults from max heap" | Proof 2 — the allocate-until-refused experiment at two `-Xmx` values |
| "`System.gc()` did not free it" | `Cleaner` timing (Proof 6) plus `-Xlog:gc` showing `Pause Full (System.gc())` with no RSS change |
| "The fix bounded it" | The same `summary.diff` flattening, plus the pod surviving the identical k6 run |
| "It did not cost latency" | **The recorded Topic 65 p50/p95/p99, re-run identically** |

That last row is the one people skip. **A memory fix that costs you p99 is not a fix;
it is a trade you made without measuring.** The Topic 65 baseline exists precisely so
that every Phase 8 change can be checked against it.

### The standing rule: a naive `System.nanoTime()` measurement is wrong

You will be tempted to answer "is `allocateDirect` slower than `allocate`?" like this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    ByteBuffer b = ByteBuffer.allocateDirect(1024);
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/op");
```

Four independent reasons this lies, and you cannot tell which one is lying:

1. **Dead-code elimination.** `b` is never used. C2 may prove the allocation has no
   observable effect and delete it. You measure an empty loop.
2. **Escape analysis and scalar replacement.** For a heap `ByteBuffer` the object may
   never be allocated at all (Topic 75). For a direct buffer it cannot be elided,
   because `Unsafe.allocateMemory` is a real side effect — **so the two arms of your
   comparison are optimised by completely different amounts.** This is the specific way
   this particular benchmark lies, and it makes direct buffers look *far* worse than
   they are.
3. **On-stack replacement and cold JIT.** The loop begins interpreted and is replaced
   mid-flight. Your average blends interpreted, C1 and C2 execution in a ratio decided
   by the loop count you happened to choose.
4. **You are measuring the GC and the cleaner too.** A million dropped direct buffers
   means a million cleaner actions and a million `free()` calls, on a schedule you do
   not control, charged to whichever iteration happened to be running.

**Topic 77 is the full treatment.** The correct shape is a JMH harness with
`@State(Scope.Benchmark)`, `Blackhole.consume`, warm-up iterations, and at least
`@Fork(3)` so profile pollution between the two arms shows up as variance rather than
hiding inside a single number.

And the honest framing to lead with, before any benchmark: **you do not choose direct
buffers for allocation speed.** `allocateDirect` is *slower* to allocate than
`allocate` and slower to free. You choose it because it avoids a copy when the bytes
cross into a native I/O call, and because the data does not participate in GC. Those
are the two things to measure — copy elimination and GC pressure — not allocation
cost.

### What to graph in production, permanently

Micrometer exposes all of these out of the box with a Spring Boot Actuator. Almost
nobody puts them on a dashboard, and they are exactly what you need at 3am:

| Metric | Why |
|---|---|
| `jvm_buffer_memory_used_bytes{id="direct"}` | The direct-buffer usage. **Alert at 80% of `MaxDirectMemorySize`.** |
| `jvm_buffer_count_buffers{id="direct"}` | A rising count with flat usage means many small buffers — a different problem shape. |
| `jvm_buffer_memory_used_bytes{id="mapped"}` | Trap 5's early warning. |
| `jvm_memory_used_bytes{area="nonheap"}` | Metaspace + code cache. |
| `process_resident_memory_bytes` (from the OS or a node exporter) | **The number that actually gets you killed.** Graph it on the same panel as heap used. |
| `container_memory_working_set_bytes` (cAdvisor) | What Kubernetes compares to your limit. |

**Put heap used and RSS on the same chart with the container limit as a horizontal
line.** The moment they diverge is the moment this topic becomes relevant, and a
single glance answers the interview question.

---

## Practice exercises

### 1 — Easy: build your own RSS breakdown

Start any JVM with `-XX:NativeMemoryTracking=summary` — use `orderflow` itself if it is
running, otherwise a trivial program that sleeps.

Produce a table with one row per NMT category, columns for `reserved`, `committed`, and
`committed as a percentage of the container limit` (use your machine's RAM if you are
not in a container). Then add three more rows: `NMT total committed`, `VmRSS`, and
`VmRSS − NMT total committed`.

Answer in writing:

1. Which category is second-largest after `Java Heap`? Were you expecting it?
2. What is the `VmRSS − NMT committed` gap, and name at least one plausible source.
3. Re-run with `-Xss256k` instead of the default and report what changed in the
   `Thread` section, in both `reserved` and `committed`. Explain why the two columns
   moved by different amounts.

### 2 — Medium: the audit (combines Topics 01, 15, 68, 69, 70, 79)

The class below contains **six** distinct defects. Four are from this topic; two are
from earlier topics. For each: name the topic, state the **observable** symptom in
production (what does the on-call engineer actually see — a metric, a log line, an exit
code — not "it's bad practice"), and write the fix.

```java
package com.orderflow.catalog;

import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.HashMap;
import java.util.Map;

public class CatalogImageCache {

    private static final Map<Long, ByteBuffer> IMAGES = new HashMap<>();
    private static final Map<Long, Long> BYTES_SERVED = new HashMap<>();

    public ByteBuffer imageFor(Long productId) throws Exception {
        ByteBuffer cached = IMAGES.get(productId);
        if (cached != null) {
            Long served = BYTES_SERVED.get(productId);
            BYTES_SERVED.put(productId, served + cached.capacity());
            return cached;
        }

        Path path = Path.of("/data/images/" + productId + ".jpg");
        FileChannel channel = FileChannel.open(path, StandardOpenOption.READ);
        ByteBuffer buffer = ByteBuffer.allocateDirect((int) channel.size());
        channel.read(buffer);
        channel.close();

        IMAGES.put(productId, buffer);
        BYTES_SERVED.put(productId, 0L);
        return buffer;
    }

    public void evict(Long productId) {
        IMAGES.remove(productId);
        System.gc();                       // "free the native memory"
    }
}
```

Hints, in the order to think about them: one defect throws an NPE on a line with no
visible `null` (Topic 01). One makes the whole class unusable from more than one thread
(Topic 15/68's neighbourhood — think about what `HashMap` does under concurrent
resize). Two are unbounded-growth defects with *different* symptoms — one shows in the
heap dump and one does not. One is a `System.gc()` misunderstanding. One is a sizing
bug that only fires on a large input.

Then answer: with 100k products averaging 200 KB per image, what is the maximum native
footprint of this class as written, and what happens to a 2 GB container?

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A — establish the honest starting point.**
Re-run the Topic 65 k6 baseline against `orderflow` unchanged, with
`-XX:NativeMemoryTracking=summary` added and nothing else. Confirm you are within the
±10% gate. Record, at steady state: p50/p95/p99, heap used, NMT total committed, VmRSS,
and the NMT `committed` for every category. **This is your control. Every later number
is a delta from here.**

**Part B — build the off-heap product-price index.**
`orderflow` reads product prices on 70% of requests. Build a second implementation of
`PriceLookup` that holds all 100k prices off-heap:

- **On 25:** a single long-lived `Arena.ofShared()` holding a `MemorySegment` of
  100,000 `long` values, indexed by a dense product ordinal.
- **On 21:** a `ByteBuffer.allocateDirect(100_000 * 8)`, allocated once at startup and
  never released, with the same ordinal indexing. Note in your write-up exactly what
  you gave up by not having an `Arena`.

Keep the on-heap `Map<Long, Long>` implementation as the control and select between
them with a Spring profile.

**Part C — measure what actually changed.**
Run the identical k6 baseline against both implementations. Report:

| | on-heap `Map<Long,Long>` | off-heap segment |
|---|---|---|
| p50 / p95 / p99 on `GET /products` | | |
| heap used at steady state | | |
| NMT `Java Heap` committed | | |
| NMT `Other` committed | | |
| VmRSS | | |
| young collections per minute (`-Xlog:gc`) | | |
| total GC pause time over the run | | |

**Part D — the honest interpretation.** Answer these in writing, and be willing to
conclude that the off-heap version is not worth it:

1. Did p99 improve, worsen, or stay inside the noise? If you cannot tell, say so — the
   correct answer to a measurement inside the error margin is "no detectable
   difference", not a preference.
2. Did **total RSS** go down, or did you just move bytes from one column to another?
   Boxing (Topic 01) means the on-heap `Map<Long,Long>` costs far more than 800 KB —
   compute what it actually costs from Topic 69's object layout, and compare to the
   800 KB the segment costs.
3. Did GC pause time fall? By how much, and is that fall visible in p99 at all? If your
   p99 is dominated by a Postgres round trip, a GC improvement may be entirely
   invisible — say so.
4. What did you give up? Name at least four things: debuggability in a heap dump (the
   segment is invisible to MAT — Topic 79), no bounds-checked field access, manual
   lifetime management, and the ordinal-mapping layer you now have to maintain.

**Part E — force the failure and prove the bound.**
Deliberately break Part B: allocate a fresh `Arena` (or `DirectByteBuffer`) per request
instead of once at startup, and retain them in a static list. Run the baseline again
under `--memory=2g`.

- Predict, **in writing, before running**: which dies first, `MaxDirectMemorySize` or
  the cgroup? Justify your prediction from the derived-default rule.
- Run it. Record which actually happened and the exit code.
- Now set `-XX:MaxDirectMemorySize=128m` and re-run. Record how the failure mode
  changed and, specifically, **what you now have that you did not have before** (a
  stack trace, a heap dump, a named line of code).

**Part F — argue against yourself.**
You have just built an off-heap index. Make the strongest possible case that the
on-heap `Map` was the right choice for `orderflow`, and state precisely what would have
to be true — about dataset size, live set, GC pause budget, or team experience — for
your answer to flip. Then decide, and write one paragraph you would actually put in a
PR description.

---

## Interview questions

### Q1 — "The heap is at 40%. Why did Kubernetes OOMKill the pod?"

**Mid-level answer:** "Something must be leaking memory outside the heap. Maybe direct
buffers or a native library. I'd increase the memory limit and see if it stops."

**Senior answer:** "Because the heap is one term in RSS and the cgroup limit applies to
RSS. The full set is: committed heap, Metaspace and compressed class space, the JIT
code cache, thread stacks, GC data structures like the card table and remembered sets,
direct and mapped byte buffers, JVM internal structures, and any native library's own
`malloc`. Several of those are unbounded by default — Metaspace and direct memory in
particular.

The kill itself is the kernel's cgroup OOM killer sending `SIGKILL`, which the JVM
cannot catch, so there is no `OutOfMemoryError`, no heap dump, and no shutdown hook.
The absence of evidence *is* the diagnostic signature — an OOM with a stack trace is a
heap problem, an OOM with nothing but exit code 137 is a native problem.

Concretely I would: confirm it with `kubectl describe pod` showing `OOMKilled` and exit
137, or `memory.events` inside the cgroup showing a non-zero `oom_kill` count. Then
redeploy with `-XX:NativeMemoryTracking=summary`, take a `VM.native_memory baseline`
early and a `summary.diff` after the growth, and see which category moved. If NMT's
committed total is close to RSS, it is a JVM subsystem and the diff names it. If RSS is
far above NMT's total, it is a native library or allocator fragmentation and I go to
`pmap`.

Direct byte buffers are the usual answer for a service doing NIO, and
`-XX:MaxDirectMemorySize` is the bound almost nobody sets — its default is derived from
max heap, which has nothing to do with the container limit. Setting it converts a
silent kill into an `OutOfMemoryError: Direct buffer memory` with a stack trace, which
is a strictly better failure. I'd do that before I raised the limit, because raising
the limit on a leak just buys time."

**What separates them:** the mid answer guesses a cause; the senior answer gives the
**equation**, explains **why there is no evidence**, gives a **ranked diagnostic
procedure**, and — the part that gets people hired — proposes making the failure
*louder* before making it *later*.

**Follow-up the interviewer asks:** "You said no heap dump. But we have
`HeapDumpOnOutOfMemoryError` set. Why didn't it fire?" (Because no `OutOfMemoryError`
was thrown. That flag is a heap mechanism; `SIGKILL` bypasses the JVM entirely.)

---

### Q2 — "Walk me through exactly when a `DirectByteBuffer`'s native memory is freed."

**Mid-level answer:** "When the buffer is garbage collected, a `Cleaner` frees it."

**Senior answer:** "There are four steps and each one can stall.

First, the `DirectByteBuffer` must become unreachable — and this is where most real
bugs live, because a buffer sitting in a static map or an unbounded pool is reachable
forever and no amount of GC helps.

Second, a collection must actually run and notice. That is the interesting part: the
GC's decision to run is driven by *heap* occupancy, and the wrapper is a few dozen
bytes while the native block is megabytes. So the collector has no idea the expensive
thing exists. HotSpot has no external-memory accounting hook the way V8 does for Node
`Buffer`s. Native memory can grow for minutes without provoking a single collection.

Third, the phantom reference gets cleared and enqueued, and a JVM thread drains the
queue and runs the cleaning action, which calls `Unsafe.freeMemory`. Enqueuing and
cleaning are separate, asynchronous events — `System.gc()` returning does not mean the
cleaners have run.

Fourth, `free()` is not `munmap()`. The C allocator may keep the pages in its arena, so
RSS need not fall even after the JVM has correctly released everything. That last step
is why people see NMT's number drop while `docker stats` does not move.

There is one automatic relief valve: `Bits.reserveMemory` calls `System.gc()` and
retries with backoff before throwing `OutOfMemoryError: Direct buffer memory` — which
is precisely why `-XX:+DisableExplicitGC` is more dangerous than it looks on a
NIO-heavy service.

The reason FFM's `Arena` matters is that it deletes all four steps. `close()` frees,
deterministically, on that line, and a use-after-close throws
`IllegalStateException` rather than segfaulting."

**What separates them:** four named steps instead of one, the V8-versus-HotSpot
accounting contrast, knowing that `free` is not `munmap`, and knowing that
`DisableExplicitGC` interacts with the JVM's own internal call. Any one of those is a
strong signal.

**Follow-up:** "So would `System.gc()` in a monitoring thread fix it?" (No — and the
candidate should give at least three of: it can be disabled, it is a full pause, it
does not wait for cleaners, and it cannot collect a reachable object.)

---

### Q3 — "How would you size the JVM for a 2 GB / 1 CPU container?"

**Mid-level answer:** "Set `-Xmx` to about 1.5 GB to leave some room for the JVM, and
use G1."

**Senior answer:** "I would not start from a fraction — I would start from two measured
numbers and one flag I know is wrong by default.

The two measurements: the **live set** after a full collection under representative
load, which is the floor the heap can never go below, and the **allocation rate**,
which decides how much headroom above the live set I need to keep collection frequency
sane. Both come from `-Xlog:gc*` on the Topic 65 baseline run. That is Topic 70's work,
and without it any heap number is a guess.

The flag that is wrong by default: `MaxRAMPercentage` is 25, which is a sensible
default for a machine running several JVMs and a bad default for a container that
exists to run exactly one. So I set it explicitly rather than accepting it.

Then I do the subtraction. Container limit minus Metaspace, minus code cache, minus
thread stacks — and thread count matters more than `-Xss` here because a 200-thread
Tomcat pool is a real number — minus GC structures, minus direct buffers, minus native
library slack, and what is left is the heap ceiling. I measure every one of those terms
with `-XX:NativeMemoryTracking=summary` and `jcmd VM.native_memory summary` rather than
guessing, and I bound the unbounded ones explicitly: `MaxMetaspaceSize`,
`MaxDirectMemorySize`, `ReservedCodeCacheSize`.

The 1 CPU part is the half people forget, and it is arguably the bigger risk.
`availableProcessors()` will be 1, which means ergonomics may select the serial
collector rather than G1, the common `ForkJoinPool` gets a parallelism of zero so
parallel streams run on the calling thread, GC threads drop to one, and the
virtual-thread scheduler has one carrier. So I would explicitly choose the collector
rather than accept ergonomics, and I would audit for parallel streams and
`commonPool()` usage. That is Topic 82.

Finally I would validate rather than assert: run the recorded k6 baseline in that exact
container, compare p50/p95/p99 to the committed numbers, and check `VM.native_memory`
committed against the limit with real headroom. If I cannot re-run the baseline inside
the target container, I have not sized anything — I have picked a number."

**What separates them:** starting from measurements rather than a fraction, naming the
subtraction explicitly, knowing `MaxRAMPercentage`'s default and why it is wrong here,
and — the strongest signal — bringing up the CPU-count cascade unprompted when the
question looked like it was only about memory.

**Follow-up:** "What if you cannot measure the live set because the service is not
built yet?" (Then you pick a starting number, ship it with NMT and GC logging on,
alert on RSS versus limit, and treat the first week as the measurement. Say that you
are guessing, and put an expiry date on the guess.)

---

### Q4 — "What does the FFM API give you that `DirectByteBuffer` doesn't?"

**Mid-level answer:** "It's the new API for off-heap memory and for calling native
code. It replaces `sun.misc.Unsafe` and JNI."

**Senior answer:** "Three things, and for backend work the first one is worth more than
the other two combined.

**Deterministic lifetime.** `Arena.ofConfined()` in a try-with-resources frees on the
closing brace. Every time, on that line, whether the body succeeded or threw. That
directly removes the failure mode where native memory grows while the heap looks
healthy, because nothing is waiting on a collector that has no idea the memory exists.

**Safe use-after-free.** Touching a segment after its arena closes throws
`IllegalStateException`. Under `Unsafe` the same mistake is a segfault and an
`hs_err_pid` file with no Java stack trace. A catchable exception versus a JVM crash is
the difference between a bug report and an outage.

**A 2 GB limit that isn't there.** `ByteBuffer` indexes with `int`, so a single buffer
caps out below 2 GB and you end up writing a chunking layer. `MemorySegment` uses
`long` offsets.

Plus the foreign-function half — `Linker`, `SymbolLookup`, `MethodHandle` downcalls —
which removes JNI's C stubs entirely.

The status caveat matters and I would state it upfront: `java.lang.foreign` was a
preview API on 21 and was finalised in a later release. On a 21 baseline I would not
ship it — I would use `DirectByteBuffer` with an explicit `MaxDirectMemorySize` and a
bounded pool, which converts the unbounded leak into a fixed known cost, and plan the
FFM migration for the 25 runtime. And I would check `--enable-native-access` behaviour
on the target JDK before shipping any downcall, because recent releases have been
tightening restricted-method access."

**What separates them:** leading with deterministic lifetime rather than the
native-calling headline, knowing the `int` versus `long` addressing limit, and stating
the preview-versus-final status **precisely for both versions** with a concrete 21
fallback. Vagueness about version status is a reliable tell.

**Follow-up:** "When would you still choose `DirectByteBuffer` on Java 25?" (When you
are handing bytes to an API that demands a `ByteBuffer` — most NIO channels — though
`MemorySegment.asByteBuffer()` bridges that; when you need a pooled allocator someone
else already wrote, such as Netty's; and when the code must also run on a 21 runtime.)

---

### Q5 — "We're seeing RSS grow steadily. NMT's total barely moves. What now?"

**Mid-level answer:** "Maybe NMT isn't tracking everything. I'd take a heap dump and
look for a leak."

**Senior answer:** "A heap dump is the wrong tool for this specific shape, and the fact
that NMT's total is flat is the most informative thing in the question — it tells me
the JVM did not allocate the memory. That rules out heap, Metaspace, code cache, thread
stacks and GC structures in one step, which is most of the search space.

Three candidates remain, in the order I would check them.

**A native library allocating through its own `malloc`.** NMT only accounts for
allocations the JVM makes through its own tracking. A JNI library, a native compression
or crypto codec, or a native transport is invisible to it. `pmap -x <pid>` sorted by
the RSS column, looking for large anonymous regions or a recognisable `.so`, finds it.
On Linux, running under `jemalloc` with profiling enabled via `LD_PRELOAD` will give
you an actual native allocation profile with call sites — that is the tool that
finishes this investigation.

**Allocator fragmentation.** `glibc` creates per-thread arenas and does not return
freed pages to the kernel promptly. A service with a large thread pool and bursty
native allocation can hold a lot of RSS that is genuinely free from the JVM's point of
view. Symptoms: RSS grows in steps and plateaus, and NMT shows healthy churn.
`MALLOC_ARENA_MAX=2` or switching to jemalloc is the usual mitigation, and I would A/B
it under the recorded load baseline rather than just applying it.

**Memory-mapped files.** A `MappedByteBuffer` shows in RSS while resident. `pmap` names
the file, which usually names the feature.

I'd also sanity-check that NMT was on from JVM start — you cannot enable it later, so
'NMT's total barely moves' sometimes means 'NMT was off and I misread the output'."

**What separates them:** treating the *absence* of an NMT delta as positive evidence
that narrows the search, naming `pmap` and jemalloc profiling as the next instruments,
and knowing that glibc arena behaviour is a real and common cause rather than an
exotic one.

**Follow-up:** "How would you tell fragmentation from a genuine native leak?" (A leak
grows without bound and correlates with request volume; fragmentation plateaus. Run
the load, then stop it, then wait — a leak's RSS stays up with no work happening;
fragmentation usually does too, which is why the discriminator is the *shape over
time* under varying load, plus a jemalloc profile that names allocation sites.)

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. V8 tells its garbage collector about external memory when you allocate a Node
   `Buffer`. HotSpot does not do the equivalent for `DirectByteBuffer`. Design the
   hook HotSpot would need. What would it have to measure, where would it plug into
   the collector's heuristics, and name one workload it would make *worse*.

2. `Bits.reserveMemory` calls `System.gc()` before throwing. Everything else in this
   document says `System.gc()` is not a fix. Reconcile those two statements — under
   exactly what condition is the JVM's own call useful, and why is yours not?

3. `Arena.ofShared().close()` performs a thread handshake; `Arena.ofConfined().close()`
   does not. Derive *why* from the access rules of each, without recalling it from the
   text. Then say what would go wrong if shared arenas skipped the handshake.

4. You set `-XX:MaxDirectMemorySize=256m` on a service that also uses Netty. Your
   direct-memory alert never fires, and the pod still gets OOMKilled. Give two
   independent explanations, and say which single command would distinguish them.

5. NMT reports `reserved` and `committed`; the kernel reports RSS. Construct a
   realistic scenario where each of the three is the largest, and one where committed
   exceeds RSS. What does each ordering tell you?

6. A colleague proposes moving `orderflow`'s entire 100k-product catalogue off-heap
   "to reduce GC pressure". Topic 70 says GC cost is a function of the live set. Under
   what measured conditions is this a real improvement, and under what conditions is it
   pure complexity? Name the specific number you would want to see first.

7. `SIGKILL` gives you no heap dump, no shutdown hook, and no log line. Design the
   cheapest possible mechanism that would let you reconstruct *what the JVM was doing*
   at the moment of an OOMKill, given you cannot run code at kill time. (There is more
   than one good answer; at least one involves writing something continuously rather
   than at death.)

---

## Quick reference card

### JVM flags

| Flag | What it does | When to set it |
|---|---|---|
| `-XX:NativeMemoryTracking=summary` | per-category native accounting | **always in load-test profiles; strongly consider production.** Cannot be turned on later. |
| `-XX:NativeMemoryTracking=detail` | adds call-site attribution | during an active investigation only |
| `-XX:MaxDirectMemorySize=<n>m` | hard bound on `allocateDirect` | **always in a container.** Default derives from max heap. |
| `-XX:MaxMetaspaceSize=<n>m` | bounds Metaspace | always in a container — unbounded by default |
| `-XX:ReservedCodeCacheSize=<n>m` | bounds JIT code cache | when NMT shows `Code` is a large term |
| `-Xss<n>k` | per-thread stack size | when NMT `Thread` is large; bound the pool first |
| `-XX:+AlwaysPreTouch` | commits and touches heap at startup | when you want RSS predictable from second one; costs startup time |
| `-XX:+HeapDumpOnOutOfMemoryError` | dump on OOM | always — but note it does **not** fire on `SIGKILL` |
| `-XX:HeapDumpPath=/dumps` | where to write it | always, with a mounted volume |
| `-XX:+ExitOnOutOfMemoryError` | die immediately rather than limp | usually yes in k8s — let the orchestrator restart |
| `-XX:+DisableExplicitGC` | makes `System.gc()` a no-op | **be careful** — it also disables the JVM's direct-memory relief valve |
| `--enable-preview` | required for FFM on **21 only** | never in production on 21 |
| `--enable-native-access=ALL-UNNAMED` | permits FFM restricted methods | on recent JDKs, for downcalls — verify behaviour on yours |
| `-Dio.netty.leakDetection.level=paranoid` | Netty `ByteBuf` leak detection | load tests only, never production |
| `-Dio.netty.maxDirectMemory=<n>` | Netty's own direct budget | when Netty bypasses the JDK budget |

### Diagnostic commands

| Command | What it answers |
|---|---|
| `jcmd <pid> VM.native_memory summary` | where is my native memory right now |
| `jcmd <pid> VM.native_memory baseline` | mark a "before" point |
| `jcmd <pid> VM.native_memory summary.diff` | **what changed since the baseline** — the incident-solving command |
| `jcmd <pid> GC.heap_info` | heap used, committed, and per-region detail |
| `jcmd <pid> VM.flags -all` | every flag's effective value and origin |
| `jcmd <pid> VM.info` | a broad state dump including memory |
| `grep VmRSS /proc/<pid>/status` | the number the kernel will kill you over |
| `pmap -x <pid> \| sort -k3 -n \| tail -30` | the largest individual mappings, with file backing |
| `cat /sys/fs/cgroup/memory.max` | the cgroup v2 limit as the kernel sees it |
| `cat /sys/fs/cgroup/memory.events` | `oom_kill` counter — proof it was the OOM killer |
| `docker inspect <id> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` | `true` / `137` |
| `kubectl describe pod <name>` | `Reason: OOMKilled`, `Exit Code: 137` |
| `java -XX:+PrintFlagsFinal -version \| grep -i MaxDirectMemorySize` | the flag's default and origin |
| `javac ArenaProbe.java` | is FFM preview or final on this JDK |

### Java APIs

```java
ByteBuffer.allocateDirect(n)                       // native, Cleaner-freed, non-deterministic
ByteBuffer.allocate(n)                             // on-heap byte[]
channel.map(MapMode.READ_ONLY, 0, size)            // MappedByteBuffer, also Cleaner-freed

ManagementFactory.getPlatformMXBeans(BufferPoolMXBean.class)
   // -> "direct" and "mapped" pools: getCount(), getMemoryUsed(), getTotalCapacity()

// [JAVA 25] deterministic
Arena.ofConfined()   // one thread, closed by try-with-resources
Arena.ofShared()     // many threads, close() does a handshake
Arena.ofAuto()       // GC-managed, i.e. DirectByteBuffer semantics
Arena.global()       // never freed
arena.allocate(bytes)
segment.get(ValueLayout.JAVA_LONG, offset)
segment.asByteBuffer()                             // bridge to NIO APIs
```

### The RSS equation — say it from memory

```
RSS = heap(committed) + metaspace + code cache + thread stacks
    + GC structures + direct buffers + mapped buffers
    + JVM internals + native libraries + allocator slack
```

### Gotchas checklist

- [ ] Heap graphs do not show RSS. Put them on the same chart with the limit line.
- [ ] `SIGKILL` leaves no heap dump, no exception, no shutdown hook. Exit 137.
- [ ] `-XX:MaxDirectMemorySize` defaults from **max heap**, not the container limit.
- [ ] Raising `-Xmx` silently raises the direct-memory ceiling too.
- [ ] Metaspace is native and **unbounded by default**.
- [ ] `System.gc()` cannot collect a reachable buffer, can be disabled, and is a full pause.
- [ ] `-XX:+DisableExplicitGC` disables the JVM's own direct-memory relief valve.
- [ ] Closing a `FileChannel` does **not** unmap a `MappedByteBuffer`.
- [ ] NMT must be on at **JVM start**. Put it in the load profile now.
- [ ] Compare `committed` to the limit, never `reserved`.
- [ ] NMT does not see native libraries or allocator slack. `RSS − NMT total` is the clue.
- [ ] Netty may bypass the JDK direct budget. Watch its own counters.
- [ ] A `Cleaner` action that captures `this` never runs.
- [ ] FFM is **preview on 21**, final later. Do not ship it on a 21 runtime.

---

## When would I use this at work?

**1. The 3am pod-restart page with no error log.**
`kubectl describe` says `OOMKilled`, exit 137, and the heap dashboard is flat at 40%.
Instead of guessing, you say "RSS is heap plus seven other things; NMT is off so I
cannot attribute it, and the first action is a redeploy with
`-XX:NativeMemoryTracking=summary` and `-XX:MaxDirectMemorySize` set so the next
occurrence throws with a stack trace instead of vanishing." That single sentence turns
a multi-day mystery into a two-deploy investigation, and it is the highest-value thing
in this document.

**2. Reviewing a PR that introduces `allocateDirect`.**
Someone adds a direct buffer for a "zero-copy" export path. You ask three questions:
where is it released, what bounds the total, and what does the container's RSS budget
say. If the answer to any of them is "the GC handles it", you have caught a future
OOMKill in review. The fix — a bounded pool, or an `Arena` on 25 — is five lines and
turns unbounded growth into a fixed line item.

**3. Deciding whether to move a large read-mostly dataset off-heap.**
Product wants the whole 100k-product price index in memory, and GC pause time is
already at the edge of the latency budget. You can compute the on-heap cost from
Topic 69's object layout, compare it to the off-heap cost, measure the GC pause
difference against the Topic 65 baseline, and then say whether the complexity is worth
it — with numbers, and with an honest "the p99 difference is inside the noise" if that
is what you find. That conversation is the difference between an architecture decision
and an architecture opinion, and it feeds straight into Topic 129's capacity model.

---

## Connected topics

**Prerequisites:**

- **65 — the load-test gate.** Every measurement in this document is a delta against
  the recorded p50/p95/p99 and the recorded container limits. Without that baseline
  you have numbers but no comparison, and a number with nothing to compare it to is
  not a measurement.
- **68 — memory areas.** Metaspace is native and outside the heap. That single fact is
  a term in the RSS equation and a common cause of the exact failure in this document,
  and it is unbounded by default.
- **69 — object layout and compressed oops.** Tells you what the *on-heap* alternative
  actually costs, which is the only way to evaluate an off-heap design honestly. The
  32 GB compressed-oops cliff is the same kind of capacity-planning fact as the RSS
  equation: a hard boundary that is invisible until you cross it.
- **70 — GC fundamentals.** The live set is the input to heap sizing, and heap sizing
  is the first term in the RSS subtraction. Also: GC cost is a function of the live
  set, which is the honest argument for and against moving data off-heap.
- **79 — memory leaks and heap dumps.** The retaining-path skill transfers directly.
  The difference is that a heap dump shows you the *wrapper* objects, not the native
  bytes — so MAT points you at the `DirectByteBuffer` instances and NMT tells you how
  much they cost. You need both instruments, and knowing which answers which question
  is the skill.

**This unlocks:**

- **81 — instrumentation agents.** An agent's generated classes land in Metaspace and
  its generated code in the code cache — both native, both terms in this equation. An
  APM agent measurably changes your RSS budget.
- **82 — JVM tuning and container awareness.** The direct continuation. This document
  gives you the equation; Topic 82 gives you the flags, the cgroup mechanics, and the
  CPU-count cascade that makes container sizing a two-dimensional problem.
- **83 — GraalVM native-image.** Native image dramatically changes this equation: no
  Metaspace, no code cache, no JIT compiler arenas, and a much smaller RSS floor. Read
  that as "the terms in the equation change", not as "memory becomes free".
- **101 — virtual threads.** Virtual-thread stacks live on the **heap** as
  continuations rather than as native thread stacks. That moves a term from `Thread` to
  `Java Heap` in NMT, which changes your sizing arithmetic in a way that surprises
  people.
- **103 — NIO and Netty's event loop.** Netty's pooled direct-buffer allocator is the
  single largest consumer of this topic in the real world, and its reference-counted
  `ByteBuf` is a leak shape all of its own.
- **122 — Dockerising Spring Boot.** Layered jars and AppCDS change class-loading and
  therefore the `Class` and `Code` terms, and the container defaults section is this
  topic applied at build time.
- **129 — capacity and cost modelling.** The measured RSS budget from this document is
  a direct input. "How many pods" is downstream of "how many bytes per pod, and which
  of them are unbounded".

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted: the exact derivation of the default `MaxDirectMemorySize`
on your JDK build, which NMT category direct-buffer allocations are reported under on
your JDK, and whether your Netty version bypasses the JDK direct-memory budget. Each
has an experiment in the Hands-on or Failure-drill sections that settles it on your
machine in under five minutes, and the failure drill is designed so that you determine
the NMT category empirically rather than trusting any table. Everything else —
the `Cleaner` mechanics, the RSS equation, the reasons `System.gc()` is not a fix, and
the fact that `SIGKILL` leaves no evidence — has been stable for a decade and will
still be true the next time a pod restarts at 3am with a flat heap graph.*
