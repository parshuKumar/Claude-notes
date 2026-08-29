# 66 — JVM Architecture vs V8: Class Loader, Runtime Data Areas, Execution Engine

## Phase: 8 — JVM Internals
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: no new `orderflow` capability. This is the map. Every diagnosis in Topics 67–83 names one of the areas defined here, and you will spend this document learning which `jcmd` subcommand shows you each one on the service you already have running under load.

---

## Mechanical statement

Read this twice. Everything else in this document is an elaboration of it.

> A `.class` file is a byte array. The JVM **loads** it, **verifies** every method's
> bytecode against the type rules, **prepares** its static fields, and **resolves**
> its symbolic references to other classes. The result is stored as metadata in a
> native memory area outside the Java heap.
>
> When a thread calls a method, the JVM pushes a **frame** onto *that thread's own
> stack*. The frame holds the local variable array and the operand stack for that
> single invocation, and nothing else. Objects created by that method do **not** live
> in the frame. They live on a **heap shared by every thread in the process**.
>
> The execution engine starts by **interpreting** bytecodes one at a time, counting
> how often each method is entered and how often each loop branches backwards. When a
> counter crosses a threshold, the method is handed to a **compiler** that emits
> machine code into a shared **code cache**, and subsequent calls jump straight into
> that machine code.
>
> **What is per-thread and what is shared is the origin of every concurrency problem
> in Phase 9.** It is also the origin of every memory problem in Phase 8, because the
> things that exhaust are in different places and each has its own error message and
> its own flag.

---

## The bridge from what you know

You know V8 well. That is genuinely useful here, and I am going to be exact about
where it helps and where it installs a wrong model.

### The shape is an HONEST ANALOGUE — say this out loud once so you stop being surprised by it

People will tell you the JVM is a completely different kind of runtime from V8. At the
level of *architecture*, that is not true, and pretending it is will make you memorise
things you could have transferred.

| V8 | HotSpot JVM | Verdict |
|---|---|---|
| Parses JS source, emits bytecode | Reads `.class` bytecode produced by `javac` | **PARTIAL** — same idea, but the JVM never sees your source |
| **Ignition** — bytecode interpreter | **Template interpreter** — bytecode interpreter | **HONEST ANALOGUE** |
| **Sparkplug** — fast baseline compiler | **C1** — fast, lightly-optimising compiler | **HONEST ANALOGUE** |
| **Maglev** — mid-tier optimiser | (no separate tier; C1 levels 1–3 cover the ground) | **PARTIAL** |
| **TurboFan** — speculative optimiser | **C2** — speculative optimiser | **HONEST ANALOGUE** |
| Deoptimises when a speculation is violated | Deoptimises via an *uncommon trap* | **HONEST ANALOGUE** |
| Generational GC: new space (scavenger, semi-spaces) + old space, **Orinoco** doing concurrent/parallel/incremental work | Generational GC: eden + survivor spaces + old gen, G1 doing concurrent/parallel/incremental work | **HONEST ANALOGUE** — this is closer than most Java engineers realise |
| Hidden classes / shapes describing object layout | Klass metadata describing object layout | **PARTIAL** |
| Inline caches at call sites | Inline caches at call sites | **HONEST ANALOGUE** |

So: tiered compilation, a generational copying collector, deoptimisation, inline
caches. **You already have all of these ideas.** You have simply never been able to
see any of them.

### The three differences that actually matter

Do not let the shared shape hide these. They are the entire reason Phase 8 and Phase 9
exist as separate phases.

**Difference 1 — the heap is shared across preemptively-scheduled threads.**

A V8 **isolate** has one heap and one JavaScript thread. Your callback runs to
completion. Nothing can observe your object half-built, because nothing else is
running. `worker_threads` gives you more isolates, but each has its *own* heap and
they communicate by copying (or by an explicitly shared `SharedArrayBuffer`).

A JVM has one heap and many threads, all of which can touch any object reachable from
their stacks, and the operating system can suspend any of them **between any two
bytecodes**.

This has consequences you will meet before Phase 9:

- The GC cannot simply run "between callbacks", because there is no such moment. It
  has to bring every thread to a **safepoint** first (Topic 73).
- Allocation itself is a shared-resource problem, which is why each thread gets its
  own private slice of eden — a **TLAB** (Topic 68).
- The compiler is allowed to reorder your reads and writes, and the only thing that
  constrains it is the Java Memory Model (Topic 87).

**Verdict: NO ANALOGUE.** Do not map "the event loop guarantees run-to-completion"
onto anything here. It does not exist.

**Difference 2 — the JVM has a bytecode verifier.**

V8 compiles source that its own parser produced moments earlier. It can trust the
bytecode because it wrote it.

The JVM accepts a `.class` file from anywhere: your compiler, someone else's compiler,
a jar downloaded from Maven Central, a byte array generated at runtime by CGLIB, an
APM agent's `ClassFileTransformer`. It cannot trust any of it. So before a class is
usable, a **verifier** proves things about every method: the operand stack never
overflows or underflows, types on the stack match what each instruction expects,
branches land on instruction boundaries, `final` classes are not extended, local
variables are not read before assignment.

That is why `java.lang.VerifyError` exists and why there is no such thing in Node.
It also costs startup time, which is one of the things AppCDS and the AOT cache
(Topic 83) reduce.

**Verdict: NO ANALOGUE.**

**Difference 3 — and this is the theme of the whole phase — the collector is pluggable and everything is observable.**

Say it plainly:

> **In Node you had almost no visibility. Here you have logs, dumps and flags — and
> therefore an obligation to use them.**

In Node, "we have a memory problem" is a sentence that ends in a shrug or in
`--max-old-space-size`. You cannot ask V8 how big your live set is. You cannot ask it
how many bytes an object occupies. You cannot choose a different collector. You can
take a heap snapshot in DevTools, and that is roughly where the road ends.

In the JVM:

- `-Xlog:gc*` prints every collection with before/after occupancy and a cause.
- `jcmd <pid> GC.heap_info` prints the current generation sizes.
- `jcmd <pid> GC.heap_dump` writes the entire object graph to a file you can open in
  a tool and query.
- `-XX:+UseZGC` / `-XX:+UseG1GC` / `-XX:+UseSerialGC` swap the *entire garbage
  collector* on the command line.
- JOL tells you an object's exact byte layout.
- `jcmd <pid> VM.native_memory summary` attributes memory the heap does not account
  for.

None of this is optional knowledge for a senior Java engineer, because the tools
exist. "I don't know why it OOMed" is an acceptable answer in Node and is not an
acceptable answer here. That change of obligation — not any single fact — is what
Phase 8 is for.

**Verdict: NO ANALOGUE.** You are gaining a capability you have never had.

### What about `--max-old-space-size`?

You will reach for `-Xmx` as its equivalent. It is a **PARTIAL** analogue, and Topic 68
spends real time on the gap. The short version: `--max-old-space-size` is essentially
the only heap knob you had. `-Xmx` is one of several dozen, it bounds only the *Java
heap*, and the Java heap is **not** the process footprint. Metaspace, the code cache,
thread stacks, direct byte buffers and GC bookkeeping all live outside it. Sizing
`-Xmx` from your container's memory limit is one of the traps below, and it is the
single most common way a healthy-looking Java service gets OOM-killed in Kubernetes.

---

## What is this?

The JVM specification defines a small number of **runtime data areas**. Everything the
JVM does at runtime happens in one of them. Learning their names is not trivia — it is
how you turn a symptom into a search space in one step.

There are exactly four things you need to hold:

1. **The class loader subsystem** — turns bytes into a usable class. Loading,
   linking (verify, prepare, resolve), initialisation. Topic 67 is entirely about this.
2. **The runtime data areas** — where things live. Some are per-thread, some are
   shared by the whole process.
3. **The execution engine** — interpreter, JIT compilers, and the garbage collector.
4. **The native interface** — JNI and, since Java 22, the FFM API (Topic 80), which
   is how the JVM reaches code it did not compile.

The rest of this document is (2) and (3) in detail, because (1) is Topic 67 and (4)
is Topic 80.

---

## Why does it matter?

**1. Every memory error you will ever see names a specific area, and the fixes are
completely different.**

`OutOfMemoryError: Java heap space`, `OutOfMemoryError: Metaspace`,
`OutOfMemoryError: unable to create native thread`, `OutOfMemoryError: Direct buffer
memory`, `StackOverflowError`, `CodeCache is full`. Six symptoms. Six different areas.
Six different flags. If you only know "the heap", you will raise `-Xmx` for all six
and fix one of them.

**2. It tells you what a thread dump and a heap dump each contain — and what they
cannot contain.**

A thread dump shows you every thread's *stack*: which frames are live, what each
thread is blocked on. It tells you nothing about object sizes. A heap dump shows you
every object on the *heap* and its references. It tells you almost nothing about what
the CPU is doing. Knowing the split means you pick the right instrument on the first
try at 2am.

**3. It is the vocabulary for the next eighteen topics.**

"Humongous allocation" (71) is a heap-region fact. "Time to safepoint" (73) is a
thread-stack fact. "Scalar replacement" (75) is a fact about an object that never
reached the heap at all. "Metaspace leak" (81) is a class-loader fact. Every one of
those sentences is meaningless without this map, and obvious with it.

**4. It is where your Node instincts are most dangerous.**

Not because they are wrong, but because they are *almost* right. The shape transfers;
the threading model does not. An engineer who assumes run-to-completion writes a
counter without synchronisation, sees it pass every test on a single-threaded test
runner, and then under-counts by 3% under k6 load with no exception, no log line and no
error metric. That bug is Topic 87. Its root cause is on this page.

---

## Machine-level reality

### The runtime data areas, and the only table you need to memorise

| Area | Per-thread or shared | What lives there | Exhaustion symptom | Flag that bounds it | Command that shows it |
|---|---|---|---|---|---|
| **JVM stack** (frames) | **per-thread** | one frame per active method call: local variable array, operand stack, return address | `StackOverflowError` | `-Xss` | `jcmd <pid> Thread.print` |
| **PC register** | **per-thread** | address of the bytecode currently executing | — | — | implicitly, in `Thread.print` |
| **Native method stack** | **per-thread** | C stack frames for JNI / FFM calls | `StackOverflowError` or a hard JVM crash with an `hs_err_pid` file | OS / `-Xss` | the `hs_err_pid*.log` file |
| **Heap** | **SHARED** | every object and every array | `OutOfMemoryError: Java heap space` | `-Xms` / `-Xmx` / `-XX:MaxRAMPercentage` | `jcmd <pid> GC.heap_info` |
| **Metaspace** (the spec calls it the *method area*) | **SHARED** | class metadata, method bytecode, the runtime constant pool, `Klass` structures | `OutOfMemoryError: Metaspace` | `-XX:MaxMetaspaceSize` | `jcmd <pid> VM.metaspace` |
| **Code cache** | **SHARED** | machine code emitted by C1 and C2, plus interpreter stubs and adapters | `CodeCache is full. Compiler has been disabled.` — a **warning**, not an error, and throughput collapses | `-XX:ReservedCodeCacheSize` | `jcmd <pid> Compiler.codecache` |
| **String table / symbol table** | **SHARED** | interned `String`s and the symbols classes are named by | usually shows up as heap or native growth | `-XX:StringTableSize` | `jcmd <pid> VM.stringtable` |

Print that table. It is the single highest-value thing on this page.

### Two honest corrections to the textbook version

Most tutorials say "static fields live in the method area". That was true before Java
8. It is not how HotSpot works now, and the difference is visible in a heap dump.

**Correction 1 — static field *values* are on the heap.** Since JDK 8 removed PermGen,
class *metadata* (the `Klass`, the method bytecode, the constant pool) lives in native
metaspace, but the **storage for static fields** lives in the class's
`java.lang.Class` mirror object, which is an ordinary heap object. This is why a
static `Map` that grows without bound shows up in a heap dump rooted at a `Class`
(Topic 79), and why a class-loader leak shows up as *both* metaspace growth and heap
growth.

**Correction 2 — "method area" is a specification word, not a HotSpot word.** The JVM
spec defines a method area and deliberately does not say where it lives. HotSpot
implements it as metaspace, in native memory, sized in chunks per class loader. Other
JVMs may do something else. When you read the spec and the HotSpot docs side by side
and they seem to disagree, this is usually why.

Both of these are implementation facts about HotSpot, not spec guarantees. Confirm the
metaspace side on your own JVM:

```bash
jcmd <pid> VM.metaspace
```

and note that the per-classloader breakdown it prints is exactly the shape of a
classloader-leak diagnosis in Topic 81.

### A frame, precisely

When your `OrderService.placeOrder(...)` calls `InventoryService.reserve(...)`, the
JVM pushes a frame containing:

- a **local variable array**, indexed from 0. For an instance method, slot 0 is
  `this`; then the parameters; then your locals. A `long` or a `double` occupies
  **two** consecutive slots. Everything else occupies one.
- an **operand stack**, which is where the bytecode instructions actually compute.
  The JVM is a stack machine: `iadd` pops two ints and pushes their sum. The maximum
  depth is computed by `javac` and stored in the class file, which is one of the
  things the verifier checks.
- **frame data**: a reference to the runtime constant pool, and the return address.

Two consequences you can use immediately:

- A local variable holding an object holds a **reference**, not the object. The object
  is on the shared heap. This is the mechanical reason a "local" object can be seen by
  another thread the instant you store it into a field. It is also why **GC roots**
  (Topic 70) include every thread's stack.
- Recursion depth is bounded by `-Xss`, per thread. The default is platform-dependent
  — commonly around 1 MB on 64-bit Linux, but do not take my word for it:

```bash
java -XX:+PrintFlagsFinal -version | grep -i ThreadStackSize
```

That prints the value in kilobytes on HotSpot. Verify, do not trust me.

### Where the JVM stack ends and the heap begins — the sentence that matters

> **Frames are per-thread and die when the method returns. Objects are shared and die
> when nothing can reach them.**

An object created inside a method does not go away when the method returns. It becomes
unreachable *if* no one kept a reference — and then a collector, at some unspecified
later moment, notices. That gap between "unreachable" and "collected" is the entire
subject of Topic 70.

There is one important exception, and it is Topic 75: if C2 can *prove* an object never
escapes its allocating method, it can **scalar-replace** it — the object is never
allocated at all and its fields become CPU registers. That is a real optimisation you
will verify with a profiler, not a hope.

### The class loader subsystem, in one paragraph

Three built-in loaders form a delegation chain: **bootstrap** (the core JDK classes,
written in native code, shown as `null` in Java), **platform** (JDK modules that are
not core), **application** (your classpath). A request to load a class is delegated to
the parent first, and only if the parent cannot find it does the child try. A class's
identity at runtime is the pair **(fully-qualified name, defining class loader)** —
which is why the same `com.orderflow.Order` loaded by two different loaders produces a
`ClassCastException` whose message appears to say a class cannot be cast to itself.

That is the whole of Topic 67, which you should read next. It is a direct continuation.

### The execution engine — how a method gets faster

Every method starts at **tier 0: interpreted**. The interpreter maintains two counters
per method: an invocation counter, and a back-edge counter (how many times a loop
jumped backwards). When the sum crosses a threshold, the method is queued for
compilation.

The tiers, as HotSpot names them:

| Tier | Compiler | What it does |
|---|---|---|
| 0 | interpreter | executes bytecode directly; collects basic counts |
| 1 | C1 | compiled, **no profiling** — used for trivial methods where profiling is not worth it |
| 2 | C1 | compiled, limited profiling — used when the C2 queue is long |
| 3 | C1 | compiled, **full profiling** — the normal warm-up path |
| 4 | **C2** | fully optimised, profile-guided, speculative |

The normal life of a hot method is **0 → 3 → 4**. Tier 3 code is slower than tier 1
code because it is busy recording which types actually arrive at each call site and
which branches are actually taken. C2 then uses that profile to speculate: "this call
site has only ever seen `CardPaymentGateway`, so inline it directly and install a
guard". If the speculation is later violated, an **uncommon trap** fires and execution
falls back to the interpreter, which is called **deoptimisation**. Topic 74.

**On-stack replacement (OSR)** is the case that confuses people: a method containing a
long-running loop can be compiled and swapped in *while it is still executing*, with
the current frame's state migrated into the compiled version. This is why a program
consisting of one giant `main` loop can speed up mid-run.

Compiled code goes into the **code cache**, which since JDK 9 is segmented into three
parts (non-nmethod stubs, profiled nmethods, non-profiled nmethods). When it fills,
HotSpot prints a warning and *disables the compiler*. Your service does not crash. It
just quietly reverts toward interpreted speed and your p99 falls off a cliff. That is
one of the traps below.

### V8's tiering versus HotSpot's, honestly

The shapes match. The differences that will bite you:

- **V8 caches compiled code across runs** in some embedders; HotSpot's code cache is
  per-process and starts empty every time. That is why JVM warm-up is a deployment
  concern and Node's is much less of one. AppCDS and `[JAVA 25]` the AOT cache
  (Project Leyden) exist to reduce exactly this. Topic 83.
- **HotSpot compiles on background threads.** `-XX:CICompilerCount` sets how many.
  Your application thread does not stop to compile; it keeps interpreting until the
  compiled version is installed. So warm-up is a *latency* story, not a stall.
- **HotSpot's profile is per-JVM and permanent for the life of the process.** A call
  site polluted early stays polluted. This is why a benchmark that runs several
  implementations in one JVM lies (Topic 77) and why a rarely-used code path can
  permanently degrade a hot one (Topic 74).

### What the verifier actually checks, and why you meet it

Between loading and initialisation, every method is verified. Modern class files carry
a `StackMapTable` attribute — a summary, written by `javac`, of the types on the
operand stack and in the local variable array at each branch target. The verifier uses
it to do a fast single-pass type check instead of the slow iterative dataflow analysis
older class files required.

You meet the verifier in exactly three situations:

1. **A bytecode-manipulating library produced invalid bytecode.** `VerifyError` at
   class load. Suspects: an old CGLIB/ASM/ByteBuddy against a newer class-file
   version, or an APM agent (Topic 81).
2. **A class file compiled for a newer JDK.** That is `UnsupportedClassVersionError`,
   not `VerifyError`, and the message names both versions.
3. **Someone tried to turn verification off** to speed up startup.
   `-Xverify:none` / `-noverify` were deprecated in JDK 13. On modern JDKs they are
   ignored, with a warning. I am not going to assert the exact behaviour on your build
   — run `java -Xverify:none -version` and read what it prints. Either way: do not do
   this. You are disabling the thing that stops malformed bytecode from corrupting the
   VM.

### Why this split is the origin of every Phase 9 problem

Hold the two columns together:

**Per-thread, therefore automatically safe:** locals, parameters, the operand stack,
the program counter. No other thread can see them. A method that only touches locals
and primitives is thread-safe for free, and that is the deep reason "make it a local
variable" is such an effective fix.

**Shared, therefore requiring a memory model:** every object, every array, every
static field's storage, every interned string, all compiled code. The moment a
reference to an object crosses from one thread's stack to another's — by being stored
in a field, put in a collection, or passed to an executor — that object becomes a
correctness question.

Phase 9's entire content is the rules for that second column: `volatile` and visibility
(86), atomicity (87), safe publication (88), lock ordering (94), false sharing (96).
None of it would exist if the heap were per-thread. In V8 it effectively is, which is
why JavaScript has almost no memory model and Java has a famous one.

---

## Example 1 — minimal

The goal is to see the split with your own eyes: a per-thread failure that one thread
survives, and a shared failure that takes everyone down.

`src/main/java/com/orderflow/lab/AreaTour.java`:

```java
package com.orderflow.lab;

import java.util.ArrayList;
import java.util.List;

public class AreaTour {

    // A static field. Its STORAGE is on the heap, in the Class mirror.
    // Everything it references is shared by every thread in this JVM.
    static final List<byte[]> SHARED = new ArrayList<>();

    static int depth = 0;

    // Per-thread: each recursive call pushes a frame onto THIS thread's stack.
    static void recurse() {
        depth++;
        recurse();
    }

    public static void main(String[] args) throws Exception {

        Thread worker = new Thread(() -> {
            try {
                recurse();
            } catch (StackOverflowError e) {
                System.out.println("worker: StackOverflowError at depth ~" + depth);
                System.out.println("worker: I am still alive and so is the JVM");
            }
        }, "worker");

        worker.start();
        worker.join();

        System.out.println("main: still running after worker overflowed its stack");

        // Now exhaust something SHARED.
        try {
            while (true) {
                SHARED.add(new byte[1024 * 1024]);   // 1 MB, retained forever
            }
        } catch (OutOfMemoryError e) {
            System.out.println("main: OutOfMemoryError after retaining "
                    + SHARED.size() + " MB");
        }
    }
}
```

Run it with a small, explicit heap so the second half finishes quickly:

```bash
java -Xmx64m -Xss256k AreaTour.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `worker: StackOverflowError at depth ~N`, then `main: still running` | Stacks are **per-thread**. One thread blew its stack; the JVM and every other thread were unaffected. |
| The depth `N` changes when you change `-Xss` | Confirms `-Xss` is the bound, and confirms the stack is a fixed-size reservation per thread, not a growable heap structure. |
| `main: OutOfMemoryError after retaining ~N MB`, with N close to your `-Xmx` minus overhead | The heap is **shared** and **bounded**. Note the number is *lower* than 64: some of your heap is not available for your objects. |
| `depth` printed as a wildly different number on two runs of the same command | Expected. Frame size depends on which methods are on the stack and whether they are compiled. Do not treat this as a precise measurement. |
| No `StackOverflowError` at all, program hangs | You are on a JVM where the recursion got tail-call-shaped by the JIT, or `-Xss` is very large. Increase the recursion cost by adding a `long[] pad = new long[8];` local. |

**The thing to actually take away:** you just crashed one thread without crashing the
process, then crashed the process from a single thread. That asymmetry is the map.

---

## Example 2 — production scenario (on the project spine)

### The constraints

This is your Topic 65 baseline. State it precisely, because every number below is
relative to it:

- `orderflow` in a container: `--memory=2g`, `--cpus=2`.
- Dataset: 100,000 products, 1,000,000 orders, 5,000,000 order lines in Postgres.
- k6, open-model arrival rate, scenario mix 70% catalogue read / 20% order read /
  10% order placement.
- Baselines committed in `/docs/java/baselines/`: p50, p95, p99, p999, throughput and
  error rate per endpoint, plus the JVM flags used.
- JVM flags at baseline: whatever you recorded. If you did not record them, the gate
  was not met — go back and record them, because everything below is a comparison.

### The incident

Two weeks after the baseline, the team adds a reporting endpoint. It is a small change.
The deploy goes out. Then:

- p99 on `GET /products` goes from your baseline number to roughly four times it.
- The change is not in the catalogue path. It touches nothing that endpoint uses.
- Heap usage looks *fine*. `GC.heap_info` shows plenty of room. GC pause times are
  unchanged from baseline.
- Container memory sits at about 1.9 GB of the 2 GB limit and occasionally the pod
  restarts with reason `OOMKilled`.
- Nothing in the application log is unusual.

An engineer who only knows "the heap" is now stuck. They will raise `-Xmx`, which will
make the OOMKills *more* frequent, and they will not understand why.

### The reasoning, using only the table above

Walk the areas in order and ask which one explains *all* the symptoms.

**Is it the heap?** No. GC logs are unchanged and `GC.heap_info` shows headroom. If it
were the heap, you would see higher GC frequency or longer pauses in `-Xlog:gc*`.

**Is it a database problem?** Check, because it usually is — but a database regression
would show up as time spent waiting, and it would not explain the container memory
growth or the fact that an *unrelated* endpoint slowed down.

**Which areas are shared, so that a change in one endpoint can degrade another?** From
the table: heap, metaspace, code cache, string table. That is the candidate list, and
it is short.

**Which of those, when exhausted, degrades throughput without throwing?** Exactly one:
the **code cache**. When it fills, HotSpot logs a warning and turns the JIT compiler
off. Methods that were already compiled keep running compiled. Methods that had not yet
reached tier 4 stay at tier 3 or in the interpreter, **forever**. And methods that get
deoptimised are never recompiled.

That last point is what makes it look unrelated: the catalogue endpoint was fine when
it warmed up, then some deoptimisation kicked it back to the interpreter, and there was
no compiler left to bring it back.

**Which of those explains container memory at 1.9 GB with a healthy heap?** Metaspace,
code cache and thread stacks are all outside `-Xmx`. If someone sized `-Xmx` at 1.8 GB
"because the container has 2 GB", the process footprint was always going to exceed the
limit. That is Trap 2 below and Topic 82 in full.

### The commands that settle it, in order

```bash
# 1. Find the JVM inside the container.
docker compose exec orderflow jcmd -l

# 2. Code cache occupancy and whether compilation is still enabled.
docker compose exec orderflow jcmd <pid> Compiler.codecache

# 3. Metaspace, broken down per class loader.
docker compose exec orderflow jcmd <pid> VM.metaspace

# 4. Heap, to rule it in or out.
docker compose exec orderflow jcmd <pid> GC.heap_info

# 5. Everything the JVM believes about its own configuration.
docker compose exec orderflow jcmd <pid> VM.flags -all

# 6. Attribute the footprint the heap does not explain.
#    Requires the JVM to have been started with -XX:NativeMemoryTracking=summary.
docker compose exec orderflow jcmd <pid> VM.native_memory summary
```

And for the next deploy, so that you have the evidence instead of having to reconstruct
it:

```bash
-Xlog:codecache+sweep=info,class+load=info,gc*:file=/logs/jvm.log:time,uptime,level,tags:filecount=5,filesize=100M
```

### What to look for, and what each outcome means

| What you see | What it means | Where it goes next |
|---|---|---|
| `Compiler.codecache` shows the code cache near its reserved size, and the log contains `CodeCache is full. Compiler has been disabled.` | Confirmed. The JIT is off. This is your p99 regression and it will never recover without a restart. | Raise `-XX:ReservedCodeCacheSize`, then find *why* code volume grew — usually an agent (81), a flood of lambdas or generated proxies (40), or `-XX:-UseCodeCacheFlushing` set by someone. |
| `VM.metaspace` shows the class-loader count growing every hour | A class-loader leak. Metaspace never shrinks meaningfully and the heap grows too, because each loader retains `Class` mirrors. | Topic 81, then Topic 79's dominator tree. |
| `GC.heap_info` shows old gen near max and `-Xlog:gc*` shows frequent full collections | It *is* the heap after all, and the reporting endpoint retains something. | Topic 70 for the live-set arithmetic, Topic 79 for the dump. |
| `VM.native_memory summary` shows a large `Thread` category | Thread stacks. Count your threads. `threads × -Xss` is real, reserved, per-thread memory. | Topic 90. |
| `VM.native_memory summary` shows a large `Internal` or `Other` with direct buffers | `DirectByteBuffer` growth — heap looks fine, RSS does not. | Topic 80. |
| Everything looks normal and p99 is still bad | You have ruled out the JVM. Now it is the database, the pool, or the downstream. That is a *result*, not a failure. | Topics 55, 109, 111. |

### The point of the scenario

You did not guess. You had a list of seven places memory and time can go, you knew
which are shared and which are not, and you eliminated five of them with four commands.
That is the entire value of this topic, and it is why it comes before the eighteen
topics that follow.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — raising `-Xmx` to fix `OutOfMemoryError: Metaspace`

**Wrong:**
```bash
java -Xmx4g -jar orderflow.jar     # was -Xmx2g; the OOM said "Metaspace"
```

**Exact symptom:** the application still dies with
`java.lang.OutOfMemoryError: Metaspace`, at roughly the same uptime as before. Heap
graphs in your dashboard look completely healthy right up to the crash. Restarting
buys you the same number of hours every time.

**Root cause:** metaspace is **native memory outside the Java heap**. `-Xmx` bounds the
heap and has no effect on it whatsoever. The underlying cause is almost always
repeated class loading by loaders that are never released: hot redeploys in an app
server, a scripting engine compiling classes per request, a bytecode-generating library
in a loop, or a test harness creating a fresh Spring context per test class.

**Fix:** first, get evidence, not a bigger number.

```bash
jcmd <pid> VM.metaspace                        # per-classloader breakdown
jcmd <pid> VM.classloader_stats                # loader count and class count
java -Xlog:class+load=info -jar orderflow.jar  # what is being loaded, and repeatedly
```

If the class-loader count grows monotonically, you have a loader leak and
`-XX:MaxMetaspaceSize` only changes *when* you die. Fix the leak (Topic 81). If the
count is stable and simply large — a big Spring application with many generated proxies
is legitimately tens of thousands of classes — then raising `-XX:MaxMetaspaceSize` is a
correct fix, and you should also raise the container memory limit, because metaspace
counts against it.

---

### Trap 2 — sizing `-Xmx` from the container memory limit

**Wrong:**
```yaml
# k8s: limits.memory: 2Gi
env:
  - name: JAVA_OPTS
    value: "-Xmx2g"        # "the container has 2 GB, so give the heap 2 GB"
```

**Exact symptom:** the pod is killed by the kernel. `kubectl describe pod` shows
`Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. There is **no**
`OutOfMemoryError` in your application log, no stack trace, no heap dump — the JVM was
never given the chance to notice. Restart, run fine for a while, killed again.

**Root cause:** **the heap is not the process footprint.** Resident memory is heap +
metaspace + code cache + thread stacks + direct byte buffers + GC bookkeeping (card
tables, remembered sets, mark bitmaps) + the JVM's own C++ structures + any native
libraries. A JVM with a 2 GB heap in a 2 GB container has a footprint that exceeds the
limit by design.

**Fix:** size the heap from your *measured* live set plus headroom, not from the
container limit. Leave real room for everything else. Then confirm what the JVM
actually chose:

```bash
jcmd <pid> VM.flags -all | grep -E 'MaxHeapSize|MaxMetaspaceSize|ReservedCodeCacheSize'
jcmd <pid> VM.native_memory summary       # needs -XX:NativeMemoryTracking=summary
```

Prefer `-XX:MaxRAMPercentage` over a hard `-Xmx` in containers so the value scales when
the limit changes. Note that the *default* `MaxRAMPercentage` is a small fraction —
appropriate for a machine running many processes, wrong for a container that exists to
run one JVM. Full treatment in Topic 82; the arithmetic is in Topic 70.

---

### Trap 3 — raising `-Xss` to "fix" a `StackOverflowError`

**Wrong:**
```bash
java -Xss16m -jar orderflow.jar      # the parser blew the stack on one deep document
```

**Exact symptom:** the original `StackOverflowError` goes away. Some weeks later, under
peak load, you get `java.lang.OutOfMemoryError: unable to create native thread` — or,
in a container, another silent OOMKill. Heap is fine. The thread count in your metrics
is high but not absurd.

**Root cause:** `-Xss` is **per thread**. Every thread the JVM creates reserves that
much address space, and touches a growing part of it. At 200 threads, 1 MB stacks are
200 MB; 16 MB stacks are 3.2 GB. You did not fix a stack problem, you multiplied it by
your thread count. And the recursion that overflowed at 1 MB will still overflow at
16 MB on a slightly deeper input — you bought a constant factor against an unbounded
problem.

**Fix:** find the recursion.

```bash
jcmd <pid> Thread.print > threads.txt
grep -c '"' threads.txt                  # rough thread count
```

Look at the overflowing thread's frames: a genuinely deep recursion (a recursive
descent parser, a deeply nested JSON document, a cyclic object graph in a serialiser)
needs an iterative rewrite or an input depth limit, not a bigger stack. A *runaway*
recursion — the same two or three frames repeating — is a plain bug. Only raise `-Xss`
when the depth is genuinely required and bounded, and then count your threads and
multiply. Topic 90 covers thread footprint properly.

---

### Trap 4 — carrying the run-to-completion assumption across

**Wrong:**
```java
@Service
public class OrderMetrics {
    private long ordersPlaced;                       // no volatile, no lock

    public void recordPlacement() { ordersPlaced++; }   // "it's just a counter"
    public long total() { return ordersPlaced; }
}
```

In Node this is correct. Your callback runs to completion; nothing interleaves.

**Exact symptom:** under the k6 baseline at 10% order placement, `total()` reports
fewer orders than the database contains. The gap is small — a fraction of a percent —
and it grows with concurrency. There is **no exception, no log line, no error metric**.
Under a single-threaded integration test it is always exactly right.

**Root cause:** `ordersPlaced++` is three bytecodes — read, add, write — and the OS can
suspend the thread between any two of them. Two threads read the same value and both
write back the same increment. The heap is shared; the scheduler is preemptive. There
is additionally a *visibility* problem: without `volatile` there is no guarantee that
one thread's write is ever seen by another at all.

**Fix (immediate):** `AtomicLong`, or better `LongAdder` under high contention.

```java
private final LongAdder ordersPlaced = new LongAdder();
public void recordPlacement() { ordersPlaced.increment(); }
public long total() { return ordersPlaced.sum(); }
```

**Fix (real):** internalise that "shared heap + preemptive scheduling" means every
mutable object reachable from two threads is a correctness question. Topics 84 through
96 are that fix, in full. This trap is here in Topic 66 because the *architectural* fact
that causes it is on this page, and because a TypeScript engineer will otherwise write
this code in week one and not find it for a year.

---

### Trap 5 — measuring the JVM before it has warmed up

**Wrong:** deploy, run k6 for 60 seconds, record p99, compare against last week's
baseline, conclude the change made things slower.

**Exact symptom:** p99 is dramatically worse than baseline for the first tens of
seconds and then converges. Two runs of the identical build produce different numbers
depending on how long they ran. Someone "optimises" code that was never slow.

**Root cause:** at process start, **zero** classes are loaded and **zero** methods are
compiled. The first execution of every code path pays: class loading, verification,
static initialisation (Topic 67), interpretation, tier-3 profiling, then tier-4
compilation. On top of that, the connection pool is empty, the Hibernate second-level
cache is cold, the OS page cache has none of your data, and Postgres has not planned
your queries. The JVM's contribution is real and it is not the only contribution.

**Fix:**

1. Make the load test's warm-up phase explicit and *exclude it from the recorded
   percentiles*. Your Topic 65 baseline should already say how long that is.
2. Prove the JIT's share rather than assuming it:

```bash
# count the classes loaded during the run
java -Xlog:class+load=info:file=classload.log -jar orderflow.jar
wc -l classload.log

# see compilation happening; the volume tapers as the service warms
java -XX:+PrintCompilation -jar orderflow.jar 2>&1 | tee compile.log
```

3. Compare the *shape* of the latency curve over time, not a single number. If p99
   drops sharply in the first minute and then flattens, that is warm-up. If it is flat
   and high, it is not.
4. Reduce it, once you have measured it: AppCDS, and `[JAVA 25]` the AOT cache.
   Topic 83.

> A naive `System.nanoTime()` around a call inside your service is **not** a valid
> measurement of any of this, for reasons Topic 77 spends a whole document on. You will
> measure a mixture of cold-JIT state, dead-code elimination and on-stack replacement,
> in unknown proportions.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM here and I am not going to
print output and call it real. What I can give you exactly is the command, what to look
for in it, and how to read every plausible result — including the surprising ones.

### Setup

```bash
mkdir -p ~/java-lab/66 && cd ~/java-lab/66
java --version                 # 21 or 25; the rest of this doc assumes one of those
jcmd -l                        # every JVM this user can see, with pids
```

If `jcmd -l` prints nothing while a JVM is running, you are in a different container,
a different user, or a different PID namespace than the JVM. Inside Docker:

```bash
docker compose exec orderflow jcmd -l
```

### Proof 1 — take the full tour of the areas on your running service

Start the `orderflow` baseline, then run each of these against it. Do all seven; the
point is to associate a symptom with a command, and that only sticks if you have seen
each one print something.

```bash
PID=$(jcmd -l | grep orderflow | awk '{print $1}')

jcmd $PID VM.version           # exact JVM build; put this in every bug report
jcmd $PID VM.uptime
jcmd $PID VM.flags             # non-default flags only  <- start here, always
jcmd $PID VM.flags -all        # every flag with its final value
jcmd $PID VM.system_properties
jcmd $PID GC.heap_info         # heap and its generations
jcmd $PID VM.metaspace         # class metadata, per loader
jcmd $PID Compiler.codecache   # JIT output, per segment
jcmd $PID VM.stringtable       # interned strings
jcmd $PID Thread.print         # every thread's stack
jcmd $PID VM.classloader_stats
jcmd $PID help                 # the full list; there are more than you expect
```

**What to look for, and what it means:**

| Command | What to look for | What it means |
|---|---|---|
| `VM.flags` | the *short* list — only what differs from default | This is your service's actual configuration, including anything injected by `JAVA_TOOL_OPTIONS` or a base image. Surprises here are common and expensive. |
| `VM.flags -all` and `grep UseG1GC`, `grep UseZGC` | which collector is actually on | Do **not** assume. Defaults are version-dependent (see the version note below). |
| `GC.heap_info` | the generation names printed | G1 prints regions and young/old counts. ZGC does not have the same shape at all. The *format tells you the collector*, which is a useful cross-check. |
| `VM.metaspace` | total committed, and the per-loader table | Committed metaspace growing over hours with a stable workload is a loader leak. |
| `Compiler.codecache` | used vs reserved, per segment | Near-full is a warning sign long before the "compiler disabled" message appears. |
| `Thread.print` | how many threads, and what they are named | Thread naming discipline pays for itself here. Nameless `Thread-47` entries are a smell. |
| `jcmd $PID help` | subcommands you did not know existed | `GC.run`, `GC.heap_dump`, `JFR.start`, `Thread.dump_to_file`, `VM.native_memory`. You will use all of them in Topics 71–82. |

### Proof 2 — watch classes being loaded, lazily

```bash
java -Xlog:class+load=info:file=classload.log:time,uptime -jar orderflow.jar
```

Let it start, then hit one endpoint you have never hit in this process, then check:

```bash
wc -l classload.log
tail -50 classload.log
```

**What to look for:** classes loaded *after* startup completed, at the moment you first
called that endpoint.

| What you see | What it means |
|---|---|
| Thousands of classes at startup, then a burst on the first request to a new endpoint | Correct and expected. Class loading is **lazy** (Topic 67). Your first request to any path is more expensive than the second, forever. |
| Classes loaded from `jrt:/java.base` early on | The bootstrap loader, serving the core JDK from the runtime image. |
| The same class name appearing more than once with different loaders | Two class loaders defined it. This is either legitimate (a container isolating applications) or the cause of a `ClassCastException` that reads like nonsense. Topic 67. |
| Class loading continuing steadily forever under constant load | A generator is producing classes at runtime. Metaspace will grow. Find it now. |

### Proof 3 — see the effect of the JIT by turning it off

```bash
# A: normal, tiered compilation
java -jar orderflow.jar

# B: interpreter only. Everything else identical.
java -Xint -jar orderflow.jar
```

Run the *same* k6 script against both. Do not use a stopwatch and do not use
`System.nanoTime()` inside the app.

**What to look for:** the difference in throughput and p99 between A and B, from the
load generator's own report.

| What you see | What it means |
|---|---|
| B is dramatically slower — a large multiple, not a few percent | Expected. You have just measured what the JIT is worth on *your* workload. This is the most vivid single demonstration of the execution engine there is. |
| B is only modestly slower | Your service is dominated by something the JIT cannot help: Postgres round-trips, network, lock contention. That is a genuinely useful finding — it tells you the JVM is not your bottleneck, and Topic 78 will tell you what is. |
| B fails to start or times out | The load test's timeouts are tuned for compiled code. Raise them; you are measuring startup, not steady state. |

Then look at compilation volume directly:

```bash
java -XX:+PrintCompilation -jar orderflow.jar 2>&1 | tee compile.log
wc -l compile.log
grep -c ' 4 ' compile.log      # rough count of tier-4 (C2) compilations
```

> Careful with that `grep`: `PrintCompilation`'s column layout is not a stable API and
> differs across JDK versions. Use it to see *relative* volume over time, not as a
> precise metric. `-Xlog:jit+compilation=debug` is the more structured route on modern
> JDKs. Topic 74 does this properly.

### Proof 4 — prove the heap is shared and the stack is not

Run Example 1 above with several `-Xss` values and record the recursion depth:

```bash
java -Xss256k -Xmx64m AreaTour.java
java -Xss512k -Xmx64m AreaTour.java
java -Xss1m   -Xmx64m AreaTour.java
```

**What to look for:** the depth roughly scaling with `-Xss`, and `main` surviving every
time.

| What you see | What it means |
|---|---|
| Depth roughly doubles as `-Xss` doubles | Frames are fixed-size and the stack is a fixed reservation. Now you can estimate: `depth × frame size ≈ -Xss`. |
| Depth is not proportional | Frame size is not constant — the JIT can compile `recurse()` and change its frame layout mid-run. Do not over-fit. |
| `main` survives every stack overflow | The stack is per-thread. This is the point. |
| The `OutOfMemoryError` count is well below your `-Xmx` in MB | Correct. Survivor space, GC reserve and object headers all consume heap you cannot allocate into. Topics 68 and 69 explain exactly where it went. |

### Proof 5 — attribute the footprint the heap does not explain

Restart with native memory tracking on:

```bash
java -XX:NativeMemoryTracking=summary -Xmx1g -jar orderflow.jar
```

Then, under load:

```bash
jcmd $PID VM.native_memory summary
jcmd $PID VM.native_memory baseline      # take a baseline
# ... run more load ...
jcmd $PID VM.native_memory summary.diff  # what grew since the baseline
```

**What to look for:** the categories, and their relative sizes. `Java Heap`, `Class`
(metaspace), `Thread` (stacks), `Code`, `GC`, `Compiler`, `Internal`, `Symbol`.

| What you see | What it means |
|---|---|
| Total committed noticeably larger than `-Xmx` | Correct and expected. This is the number your container limit must accommodate. If you have never looked at it, look now. |
| `Thread` large | `threads × -Xss`. Count threads with `Thread.print`. |
| `Class` growing in the diff under steady load | Classes are still being loaded. Loader leak or a runtime generator. |
| `GC` surprisingly large | Card tables, remembered sets and mark bitmaps scale with heap size. This is part of why a bigger heap is not free. Topics 70 and 71. |
| `Internal` or `Other` growing | Direct buffers are the usual suspect. Topic 80. |

> NMT itself has overhead — modest for `summary`, higher for `detail`. It is normally
> acceptable to leave `summary` on in production, but measure that claim against your
> own baseline rather than taking it from me.

### Proof 6 — settle the version-dependent defaults on YOUR JDK

This is the habit I most want you to build. Never assert a default; print it.

```bash
java -XX:+PrintFlagsFinal -version | grep -E 'UseG1GC|UseZGC|UseSerialGC|UseParallelGC'
java -XX:+PrintFlagsFinal -version | grep -E 'MaxHeapSize|InitialHeapSize|MaxRAMPercentage'
java -XX:+PrintFlagsFinal -version | grep -E 'ReservedCodeCacheSize|MaxMetaspaceSize'
java -XX:+PrintFlagsFinal -version | grep -E 'ThreadStackSize|CICompilerCount|TieredCompilation'
java -XX:+PrintFlagsFinal -version | grep -E 'UseCompressedOops|ObjectAlignmentInBytes'
```

**How to read it:** the marker after each value tells you where it came from.
A value marked as a product default is what the JVM picked for you; a value marked as
having come from the command line is what you set. That distinction is exactly what
`jcmd VM.flags` (short form) shows you for a running process, and it is the first thing
to check when a service behaves differently in two environments.

> **Version note, stated once and hedged deliberately.** G1 has been the default
> collector since JDK 9 and remains so on the JDKs this curriculum targets. Generational
> ZGC became the default *mode of ZGC* in JDK 23+, so `-XX:+UseZGC` gives you the
> generational variant on JDK 25 while on JDK 21 it does not unless you also pass
> `-XX:+ZGenerational`. **Do not take these as facts about your machine.** Run the
> commands above, or `jcmd <pid> VM.flags` on the running service, and read the answer.
> Defaults also change with available memory and CPU count — a JVM on a small container
> may select a different collector than the same JVM on your laptop.

---

## Failure drill

**Mandatory.** Produce the failure yourself and write down what you saw before reading
the analysis. The point is not the fact; it is the memory of watching one thread die
while the JVM shrugged, and then watching one thread take the whole process down.

### The scenario

You are going to prove the per-thread / shared split empirically, on the running
`orderflow` service rather than on a toy, and then map every exhaustion mode to the
command that would have diagnosed it.

### Part A — exhaust a per-thread area while the service keeps serving

Add a deliberately dangerous endpoint. **Behind a profile, so it cannot ship.**

`src/main/java/com/orderflow/lab/AreaDrillController.java`:

```java
package com.orderflow.lab;

import org.springframework.context.annotation.Profile;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping("/lab/areas")
@Profile("drill")                       // NEVER active in any real profile
public class AreaDrillController {

    /** Per-thread: blows only the request thread's stack. */
    @GetMapping("/stack")
    public String stack() {
        try {
            return "depth reached " + descend(0);
        } catch (StackOverflowError e) {
            return "StackOverflowError caught on " + Thread.currentThread().getName();
        }
    }

    private int descend(int n) {
        long[] pad = new long[4];       // make each frame a realistic size
        pad[0] = n;
        return descend(n + 1) + (int) pad[0];
    }

    /** Shared: retains until the whole heap is gone. */
    private static final List<byte[]> RETAINED = new ArrayList<>();

    @GetMapping("/heap")
    public String heap() {
        while (true) {
            RETAINED.add(new byte[4 * 1024 * 1024]);   // 4 MB, never released
        }
    }
}
```

Start the service with the drill profile and a small heap so Part B finishes quickly.
Use a **separate** container or a **separate** local run — do not do this against the
instance holding your recorded baseline.

```bash
SPRING_PROFILES_ACTIVE=drill \
java -Xmx512m -Xss512k \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/dumps \
     -Xlog:gc*:file=/tmp/drill-gc.log:time,uptime,level,tags \
     -jar target/orderflow.jar
```

Now, with k6 running your normal baseline scenario against it at a low rate so the
service is genuinely serving traffic:

```bash
# 1. Take a "before" snapshot of every area.
PID=$(jcmd -l | grep orderflow | awk '{print $1}')
jcmd $PID GC.heap_info      > /tmp/before-heap.txt
jcmd $PID Thread.print      > /tmp/before-threads.txt
jcmd $PID VM.metaspace      > /tmp/before-metaspace.txt
jcmd $PID Compiler.codecache > /tmp/before-codecache.txt

# 2. Blow one request thread's stack.
curl -s localhost:8080/lab/areas/stack

# 3. Immediately confirm the service is still healthy.
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/products?page=0
jcmd $PID Thread.print > /tmp/after-stack-threads.txt
```

**What to capture, written down before you continue:**

1. The exact response body from `/lab/areas/stack`.
2. The HTTP status of the `/products` call immediately afterwards.
3. Whether k6 recorded **any** errors during the window.
4. The thread count before and after.

### Part B — exhaust the shared area

```bash
# 4. This one will not return.
curl -s --max-time 120 localhost:8080/lab/areas/heap &

# 5. While it runs, watch the heap fill.
watch -n 1 "jcmd $PID GC.heap_info"

# 6. When it dies, look at what k6 saw and what the JVM wrote.
ls -la /tmp/dumps
grep -i 'Full' /tmp/drill-gc.log | tail -20
```

**What to capture:**

5. The error the `/lab/areas/heap` request returned, if any.
6. What happened to the **unrelated** `/products` requests k6 was making — error rate,
   p99 — during the last few seconds before the OOM.
7. Whether a heap dump file was written.
8. The pattern in the GC log in the final seconds.

### How to read it

| What you see | What it means |
|---|---|
| `/lab/areas/stack` returns the caught-`StackOverflowError` message, `/products` returns 200 immediately after, k6 records zero errors | **Part A has fired correctly.** A per-thread area was completely exhausted and the shared process was unaffected. This is the mechanical meaning of "per-thread". |
| `/lab/areas/stack` returns a 500 instead | Your framework's error handling caught it above your code. Still fine — the point is that the *process* survived. Check the log for the `StackOverflowError`. |
| The thread that overflowed disappears from a later `Thread.print` | It was a pooled request thread; the container may retire and replace it. Note that a `StackOverflowError` on a pooled thread can leave `ThreadLocal` state behind — that is Topic 79's second drill, and this is your first sight of it. |
| During Part B, **`/products` latency rises sharply and then requests start failing**, before any OOM | **This is the lesson.** One endpoint consumed a shared resource and degraded every other endpoint. The GC log will show collections getting more frequent and reclaiming less: that is a rising live set (Topic 70). |
| `OutOfMemoryError: Java heap space` appears on threads that were **not** the allocating one | Also the lesson. The heap is shared, so whichever thread happens to request the next allocation is the one that gets the error. **The thread that reports the OOM is usually not the thread that caused it.** Remember this; it misleads people constantly. |
| A heap dump file appears in `/tmp/dumps` | Good — your flags are correct for production. Opening it is Topic 79. Note how large it is; you must have disk space for a full heap, and a sidecar to ship it. |
| No heap dump appears | `-XX:HeapDumpPath` is not writable, or the container was OOM-killed by the kernel before the JVM could react. The second case is Trap 2 and Topic 82. |
| The process is killed with exit code 137 and there is **no** `OutOfMemoryError` at all | The **kernel** killed it, not the JVM. Your `-Xmx` plus non-heap footprint exceeded the container limit. This is a different failure from a Java OOM and needs a different fix. |

### Part C — map every area to its command, from memory

Close this document. On a blank page, write the seven areas from the big table, and for
each: per-thread or shared, its exhaustion symptom, its bounding flag, and the `jcmd`
subcommand that shows it. Then check yourself against the Quick reference card below.

If you cannot do this from memory, you will not do it at 2am. That is the whole
deliverable of Topic 66.

### What the drill proves

Three things, in increasing order of importance:

1. Per-thread areas fail locally. Shared areas fail globally.
2. **The thread that reports a shared-area failure is not necessarily the thread that
   caused it.** Every subsequent topic in this phase depends on you not being misled by
   this.
3. You now have a routine: symptom → which area → which command → evidence. You will
   use exactly this routine in Topics 71, 79, 80 and 82, with different areas plugged
   in.

---

## Measurement

### The instrument, and the claim it makes falsifiable

The claim this topic makes is: **the JVM's memory is in seven named places, and you can
attribute the process footprint across them.** The instrument that makes it falsifiable
is Native Memory Tracking plus `jcmd`, checked against the operating system's view.

```bash
# Start with NMT on.
java -XX:NativeMemoryTracking=summary -Xmx1g -jar orderflow.jar

# The JVM's own accounting.
jcmd $PID VM.native_memory summary

# The operating system's accounting, for the same process.
ps -o pid,rss,vsz -p $PID           # RSS in kilobytes on Linux/macOS
cat /proc/$PID/status | grep -i vmrss   # Linux only, same number
```

### The arithmetic — apply it to your own numbers

Do not look for a worked example with invented numbers here; there will not be one. Do
the arithmetic on **your** service.

```
observed_rss                                     (from ps / VmRSS)
  - committed_java_heap                          (NMT "Java Heap" committed)
  - committed_class_metaspace                    (NMT "Class" committed)
  - committed_thread_stacks                      (NMT "Thread" committed)
  - committed_code                               (NMT "Code" committed)
  - committed_gc_structures                      (NMT "GC" committed)
  - committed_compiler                           (NMT "Compiler" committed)
  - committed_symbol_and_internal                (NMT "Symbol" + "Internal" + "Other")
  = unattributed
```

**How to read `unattributed`:**

| What you see | What it means |
|---|---|
| A small remainder relative to RSS | Normal. NMT does not track memory allocated by native libraries or the allocator's own overhead. |
| A large and *growing* remainder | Something outside the JVM's accounting is allocating. Suspects: a JDBC driver's native parts, a compression or crypto library, glibc arena fragmentation, or direct buffers if you are on a JDK where they land outside your reading of the categories. Topic 80. |
| Committed heap much lower than `-Xmx` | Normal — the JVM commits lazily and gives memory back. Your **container limit must still accommodate `-Xmx`**, because the heap can grow to it at any moment. |
| RSS *lower* than committed heap | Also normal. Committed but untouched pages are not resident. This is why "the container is using less than the limit" is not evidence that the limit is safe. |

### The second measurement: warm-up cost

The claim: **a cold JVM is meaningfully slower than a warm one, and the difference is
class loading plus JIT.** Make it falsifiable:

```bash
# Run 1: record throughput and p99 in 10-second buckets for the first 5 minutes.
k6 run --summary-trend-stats='p(50),p(95),p(99),p(99.9)' baseline.js

# Simultaneously, count class loading over time.
java -Xlog:class+load=info:file=classload.log:time,uptime -jar orderflow.jar
```

Then plot two series against uptime: requests-per-second from k6, and cumulative lines
in `classload.log`.

| What you see | What it means |
|---|---|
| Class loading flattens at roughly the same moment throughput stabilises | Class loading and JIT warm-up dominate your cold-start penalty. AppCDS / the AOT cache (Topic 83) will help. |
| Throughput keeps improving long after class loading flattens | The remaining warm-up is C2 compilation, the connection pool filling, and caches warming. `-XX:+PrintCompilation` volume over time will separate the JIT's share. |
| Throughput never stabilises | You are not warming up; something is degrading. Check the GC log for a rising live set (Topic 70) and the code cache (Trap in Example 2). |
| Throughput is flat from the first second | Either your warm-up is genuinely negligible for this workload, or your load generator is the bottleneck and you are measuring *it*. Check its own CPU usage. Coordinated omission (Topic 65) makes this mistake easy. |

### Why `System.nanoTime()` is the wrong tool here — say it once per phase

```java
long t0 = System.nanoTime();
orderService.place(request);
long elapsed = System.nanoTime() - t0;   // this number is not what you think
```

That measurement is wrong for at least five reasons, and they compound:

1. **It includes JIT state.** The first thousand calls run in the interpreter or tier-3
   profiled code. The number changes as the method gets compiled, and you have no way to
   know which state you sampled.
2. **It can measure dead code.** If the result is unused, C2 can eliminate the work
   entirely, and you time nothing.
3. **It can include a GC pause** that had nothing to do with your call, and exclude one
   your call caused later.
4. **It can include time to safepoint** — the interval where your thread was stopped
   waiting for an unrelated operation. Topic 73.
5. **A single fork hides profile pollution.** Two implementations timed in the same JVM
   contaminate each other's call-site profiles. Topic 74.

The correct instruments are JMH for microbenchmarks (Topic 77), the load generator's
own percentiles for end-to-end latency (Topic 65), and a sampling profiler for
attribution (Topic 78). Never `console.time`'s Java cousin.

---

## Practice exercises

Write real files, run them, and keep the output. Where an exercise asks for a number,
the number must come from a tool, not from arithmetic in your head.

### 1 — Easy: the area census

Write down, without looking at this document, the seven runtime data areas from the
table. For each one, produce:

- per-thread or shared,
- the exact exhaustion message,
- the flag that bounds it,
- the `jcmd` subcommand that shows it.

Then verify **every row** against your running `orderflow` process by executing the
command and pasting a one-line summary of what it told you about *your* service. Not
what it told you in general — what it told you about your service, at your baseline
load.

Finally, answer: which two of the seven areas were you previously unaware existed, and
what symptom would each have produced that you would have misdiagnosed as a heap
problem?

### 2 — Medium: the audit (combines Topics 01, 12, 17, 19, 31–32, 39, 54, 65)

Below is a fragment from an `orderflow` reporting service. It contains **six** distinct
defects, each of which is explained by something in Topics 01–66. Find them all. For
each, state the **exact symptom the on-call engineer sees** — not "it's bad practice" —
and which runtime data area it stresses.

```java
package com.orderflow.reporting;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class DailySalesReporter {

    // "cache so we don't recompute"
    private static final Map<String, List<OrderLine>> REPORT_CACHE = new HashMap<>();

    private long reportsGenerated;

    public List<String> generate(String day) {

        List<OrderLine> lines = REPORT_CACHE.get(day);
        if (lines == null) {
            lines = loadAllLinesFor(day);          // 5M rows exist; a day is ~15k
            REPORT_CACHE.put(day, lines);
        }

        Map<Long, Long> unitsByProduct = new HashMap<>();
        for (OrderLine line : lines) {
            Long current = unitsByProduct.get(line.productId());
            unitsByProduct.put(line.productId(),
                    current == null ? line.quantity() : current + line.quantity());
        }

        String summary = "";
        for (Map.Entry<Long, Long> e : unitsByProduct.entrySet()) {
            summary = summary + e.getKey() + "=" + e.getValue() + ";";
        }

        reportsGenerated++;

        List<String> out = new ArrayList<>();
        out.add(summary);
        return out;
    }

    @Transactional
    private List<OrderLine> loadAllLinesFor(String day) {
        return repository.findAllByDay(day);       // returns entities, not projections
    }
}
```

For each defect, also answer: **would raising `-Xmx` help, hurt, or do nothing?** That
question is the point of the exercise.

### 3 — Hard: production simulation on the `orderflow` baseline

You are going to build the diagnostic runbook you will use for the rest of Phase 8.

**Part A — instrument the baseline.** Re-run your Topic 65 baseline with a full
observability flag set. Record everything to files, not to stdout:

```bash
java -XX:NativeMemoryTracking=summary \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
     -Xlog:gc*,gc+heap=info,class+load=info,safepoint:file=/logs/jvm.log:time,uptime,level,tags:filecount=10,filesize=50M \
     -XX:+UnlockDiagnosticVMOptions \
     -Xmx<your baseline value> \
     -jar target/orderflow.jar
```

Confirm the baseline still lands within ±10% of your recorded p50/p95/p99. **If the
flags themselves moved your numbers by more than 10%, that is a finding** — write down
which flag did it and how you determined that. (`-Xlog` to a file is cheap;
`class+load` is not free; NMT `summary` is usually modest. Prove it rather than
assuming.)

**Part B — build the area snapshot script.** Write `snapshot.sh` that captures, in one
run, timestamped into a directory: `VM.flags`, `VM.version`, `GC.heap_info`,
`VM.metaspace`, `Compiler.codecache`, `VM.native_memory summary`, `Thread.print`,
`VM.classloader_stats`. You will run this at the start and end of every drill for the
next seventeen topics.

**Part C — take a baseline snapshot and a loaded snapshot.** One with the service idle,
one at your recorded baseline load. Diff them. Produce a table with one row per area and
three columns: idle, loaded, delta.

**Part D — answer these, in writing, from your own numbers:**

1. What fraction of your process RSS is *not* Java heap? Is that fraction what you
   expected before you measured?
2. How many threads does `orderflow` run at baseline load, and what is
   `threads × -Xss`? Compare that to the `Thread` category in NMT. Do they agree? If
   not, why might committed differ from reserved?
3. Is your code cache growing under sustained steady load, or has it stabilised? If it
   is still growing after ten minutes at constant load, what could be generating code?
4. Given your measured non-heap footprint, what is the largest `-Xmx` you could safely
   set for your container limit? Show the arithmetic. Then note that this is a
   *provisional* answer, because you have not yet measured your live set — that is
   Topic 70, and you will revise this number there.

**Part E — the honest write-up.** Commit the snapshots and your answers next to your
Topic 65 baselines. If any measurement surprised you, say so explicitly. "I expected X,
measured Y, and here is why I now think Y is right" is the most valuable sentence you
will write in this phase.

---

## Interview questions

### Q1 — "Walk me through what happens between `java -jar app.jar` and your first line of code running."

**Mid-level answer:** "The JVM starts, loads the main class, and calls `main`."

**Senior answer:** "The launcher creates the VM, which sets up the heap according to
`-Xmx`/`-Xms` or the container-aware defaults, initialises the collector, starts the
compiler threads and the GC threads. Then the bootstrap loader brings in `java.base`,
the platform and application loaders are created, and the application loader is asked
for my main class. That class is loaded, **verified** against the bytecode type rules,
prepared — static fields get default values — and its symbolic references are resolved.
Then it is **initialised**: static field initialisers and static blocks run, in textual
order, after its superclass has been initialised. Only then does `main` execute, and it
executes **interpreted**, because nothing has been compiled yet. Every class my code
touches goes through the same sequence, lazily, the first time it is actively used —
which is why the first request to any endpoint is slower than the second one forever."

**What separates them:** naming *linking* as distinct from *loading* and *initialisation*,
knowing that the first execution is interpreted, and connecting laziness to a
production symptom rather than reciting the phases.

**Follow-up:** "Where do static field *values* live — heap or metaspace?" They are
checking whether you know the post-Java-8 answer. Metadata is in native metaspace;
static field storage is on the heap in the `java.lang.Class` mirror. A senior candidate
also says "which is why a static collection shows up rooted at a `Class` in a heap
dump."

---

### Q2 — "Our container has a 2 GB memory limit and we set `-Xmx2g`. It keeps getting OOMKilled but there's no `OutOfMemoryError` in the logs. What's going on?"

**Mid-level answer:** "The heap filled up. We should increase the memory limit or reduce
the heap."

**Senior answer:** "The absence of an `OutOfMemoryError` is the key detail — that tells
me the **kernel** killed the process, not the JVM. The JVM never got the chance to
notice, so this is not a heap-exhaustion problem at all. Heap is only one part of the
footprint: metaspace, the code cache, thread stacks at `-Xss` each, direct byte buffers,
GC structures like card tables and remembered sets, and the JVM's own native
allocations all sit outside `-Xmx`. Setting `-Xmx` equal to the limit guarantees the
process footprint exceeds the limit as soon as the heap approaches its maximum.

I'd turn on `-XX:NativeMemoryTracking=summary`, take `jcmd VM.native_memory summary` at
load, and attribute the non-heap portion. Then I'd size the heap from the measured live
set plus headroom rather than from the container limit, and use `MaxRAMPercentage` so
it scales if the limit changes. If NMT shows the `Thread` category is large, the fix is
thread count or `-Xss`, not heap. If `Internal` is large and growing, I'd look at direct
buffers and set `-XX:MaxDirectMemorySize`."

**What separates them:** treating "no `OutOfMemoryError`" as evidence rather than
noise, and being able to enumerate the non-heap consumers with the command that
attributes them.

**Follow-up:** "How would you decide the right `-Xmx`?" They want to hear: measure the
live set from heap occupancy after a full GC, then add headroom — not a rule of thumb.
That is Topic 70, and saying "I'd measure the live set first" is the answer.

---

### Q3 — "How is the JVM different from V8? You've come from Node."

This is a trap dressed as a friendly question. They are testing whether you overclaim in
either direction.

**Mid-level answer:** "The JVM is multi-threaded and compiled, V8 is single-threaded and
interpreted. Java is faster."

**Senior answer:** "Architecturally they're more similar than people expect. Both have a
bytecode interpreter, a tiered compiler with deoptimisation — Ignition/Sparkplug/Maglev/
TurboFan against the template interpreter, C1 and C2 — and both have a generational
collector with a copying young generation. Inline caches, hidden classes versus Klass
metadata, speculative optimisation: those all transfer.

Three things genuinely differ. First, the JVM's heap is shared across preemptively
scheduled OS threads, where a V8 isolate is single-threaded with run-to-completion
semantics — that one difference is why Java needs a memory model and JavaScript
essentially doesn't. Second, the JVM verifies bytecode from untrusted sources, because
it accepts class files it didn't compile. Third, and the one that changes how I work:
the collector is pluggable and everything is observable. In Node I could take a heap
snapshot and set a max old space size. Here I have GC logs with causes, per-generation
occupancy, heap dumps I can query, native memory tracking, and a choice of collector on
the command line. That's a capability I didn't have, and it means 'I don't know why it
OOMed' isn't an acceptable answer any more."

**What separates them:** refusing the false dichotomy, naming V8's actual tier names,
and framing observability as an *obligation* rather than a feature. The last sentence is
what makes an interviewer believe you have actually made the transition.

**Follow-up:** "So which is faster?" The correct answer is a refusal to answer as posed:
"for what workload, at what percentile, after how long a warm-up?" A senior candidate
adds that the JVM's peak throughput advantage is paid for with a warm-up cost that
matters enormously for short-lived processes and not at all for a service running for
weeks.

---

### Q4 — "Throughput on one of our services drops by a large factor several hours into every run. GC is normal, heap is normal, no code changes. Where do you look?"

**Mid-level answer:** "I'd take a heap dump and look for a leak. Or check if the
database got slower."

**Senior answer:** "GC normal and heap normal rules out the heap, so I'd go through the
*other* shared areas — because a gradual, global degradation with no error points at a
shared resource, and there are only a few. The one that produces exactly this signature
is the **code cache**. When it fills, HotSpot prints `CodeCache is full. Compiler has
been disabled.` and stops compiling. Already-compiled methods keep running, but anything
that deoptimises never gets recompiled, so throughput decays rather than dropping
cleanly. It's a warning, not an exception, so it never reaches your error dashboards.

I'd run `jcmd <pid> Compiler.codecache` and check used against reserved, and grep the
JVM log for that message. If confirmed, the immediate fix is a larger
`-XX:ReservedCodeCacheSize` and checking that code cache flushing is enabled, but the
real question is what is generating so much code — an APM agent instrumenting
everything, runtime proxy generation, a huge number of lambda call sites, or a
scripting engine.

If the code cache is fine, my next candidates are metaspace growth from a class-loader
leak, and then time-to-safepoint, because a long TTSP shows up as latency without
showing up in GC pause time."

**What separates them:** having a *list* of shared resources to eliminate, knowing that
the code cache failure is a warning rather than an error, and knowing the decay is
gradual because of deoptimisation.

**Follow-up:** "How would you have caught it before it hurt?" They want a metric on the
code cache. `jcmd Compiler.codecache` on a schedule, or the JMX
`MemoryPoolMXBean` for the code cache segments, exported to your dashboards. Topic 118.

---

### Q5 — "What is thread-safe by construction in Java, and why?"

**Mid-level answer:** "Immutable objects and local variables. And anything with
`synchronized`."

**Senior answer:** "The mechanical answer is: anything that lives only in a **per-thread**
runtime data area. Local variables, method parameters, the operand stack and the program
counter are per-thread, so no other thread can observe them, full stop. That's why
'make it a local variable' fixes so many concurrency bugs — it moves state out of the
shared column.

The subtlety is that a local variable holding a *reference* is per-thread, but the
**object it points at is on the shared heap**. So a local reference is safe only while
nothing else can reach the object. The moment you store it in a field, add it to a
collection, or hand it to an executor, it's shared and it's a correctness question.
Primitives in locals are unconditionally safe; object references in locals are
conditionally safe.

Beyond that: immutable objects with `final` fields are safely publishable because of the
final-field freeze in the memory model, and that's a memory-model guarantee, not just a
convention."

**What separates them:** deriving safety from the *architecture* rather than listing
safe things, and catching the reference-versus-object distinction — which is the exact
place engineers coming from a single-threaded runtime get it wrong.

**Follow-up:** "So is a method with only local variables and primitives always
thread-safe?" Yes — and the interesting follow-on is *why that's useful*: it is the
justification for the functional-core / imperative-shell design, and it is why escape
analysis (Topic 75) can eliminate the allocation entirely.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The heap is shared and stacks are per-thread. Name a JVM feature that only exists
   because of that split — something that would be unnecessary in a single-threaded
   runtime — and explain the causal chain from the split to the feature.

2. Static field *storage* is on the heap, but class *metadata* is in native metaspace.
   Construct a leak that grows metaspace but barely touches the heap, and a leak that
   grows the heap through a static field but leaves metaspace flat. What does each look
   like in `jcmd` output?

3. The code cache filling produces a *warning* and degraded throughput rather than an
   error. Argue that this was the right design decision. Then make the strongest case
   against it.

4. V8 has a generational GC, a tiered compiler and deoptimisation, just like HotSpot.
   Given that, why does Java have a formal memory model and JavaScript essentially
   doesn't? Answer in terms of runtime data areas, not in terms of language design.

5. `-Xss` is per-thread and `-Xmx` is per-process. If you were designing a JVM from
   scratch today, would you make the stack a per-thread reservation or something
   growable on the heap? What would you gain, and what would you break? (Consider what
   virtual threads do — Topic 101 — before you answer.)

6. You have a service where p99 is dominated by a 300 ms downstream HTTP call. Somebody
   proposes switching collectors to reduce GC pauses from 40 ms to 0.5 ms. Using only
   what is on this page, explain why that might change nothing at all — and construct a
   case where it would actually make things *worse*.

7. Class loading is lazy. Name one production advantage and one production disadvantage
   of that, and then describe a deployment strategy that keeps the advantage while
   neutralising the disadvantage.

---

## Quick reference card

### The seven areas

| Area | Scope | Exhaustion message | Bounding flag |
|---|---|---|---|
| JVM stack | per-thread | `StackOverflowError` | `-Xss` |
| PC register | per-thread | — | — |
| Native method stack | per-thread | `StackOverflowError` / `hs_err_pid` crash | OS |
| Heap | **shared** | `OutOfMemoryError: Java heap space` | `-Xmx`, `-XX:MaxRAMPercentage` |
| Metaspace | **shared** | `OutOfMemoryError: Metaspace` | `-XX:MaxMetaspaceSize` |
| Code cache | **shared** | `CodeCache is full. Compiler has been disabled.` (a warning) | `-XX:ReservedCodeCacheSize` |
| String/symbol table | **shared** | shows as heap or native growth | `-XX:StringTableSize` |

### JVM flag table

| Flag | What it does | When you reach for it |
|---|---|---|
| `-Xms` / `-Xmx` | initial / maximum Java heap | always set both explicitly in production; equal values avoid resize pauses |
| `-XX:MaxRAMPercentage=<n>` | heap as a percentage of the container limit | containers, so heap scales with the limit |
| `-Xss<size>` | thread stack size | rarely, and only after counting threads |
| `-XX:MaxMetaspaceSize=<size>` | bounds class metadata | to turn a slow memory leak into a fast, diagnosable failure |
| `-XX:ReservedCodeCacheSize=<size>` | bounds JIT output | agents, heavy proxying, or after seeing the "compiler disabled" warning |
| `-XX:NativeMemoryTracking=summary` | enables NMT | any RSS-versus-heap investigation |
| `-XX:+HeapDumpOnOutOfMemoryError` | writes a dump on heap OOM | **always on in production**, with a writable `-XX:HeapDumpPath` |
| `-XX:HeapDumpPath=<dir>` | where the dump goes | must be writable and have space for a full heap |
| `-XX:+PrintFlagsFinal` | prints every flag's final value | settling any "what is the default?" question |
| `-Xint` | interpreter only, no JIT | as an A/B control to measure what the JIT is worth |
| `-XX:+UnlockDiagnosticVMOptions` | enables diagnostic flags | prerequisite for several flags in Topics 74–75 |
| `-Xlog:<selectors>:file=...` | unified logging | the single most useful flag family in the JVM |

### Unified logging selectors worth knowing now

| Selector | Shows |
|---|---|
| `-Xlog:gc*` | every collection, cause, before/after occupancy — Topics 68, 70, 71 |
| `-Xlog:class+load=info` | every class loaded, and by which loader — Topic 67 |
| `-Xlog:class+unload=info` | classes being unloaded — class-loader leak diagnosis |
| `-Xlog:safepoint` | safepoint operations and time-to-safepoint — Topic 73 |
| `-Xlog:codecache+sweep=info` | code cache flushing activity |
| `-Xlog:jit+compilation=debug` | structured compilation events — Topic 74 |
| `-Xlog:all=debug:file=jvm.log` | everything; useful once, overwhelming twice |

Decorators worth adding to every selector: `:time,uptime,level,tags`, plus
`:filecount=10,filesize=50M` so logs rotate instead of filling a disk.

### Diagnostic command table

| Command | Answers |
|---|---|
| `jcmd -l` | which JVMs are running and their pids |
| `jcmd <pid> VM.version` | exact build — put this in every bug report |
| `jcmd <pid> VM.flags` | **non-default** flags only; start every investigation here |
| `jcmd <pid> VM.flags -all` | every flag with its final value |
| `jcmd <pid> VM.info` | a broad dump: threads, memory, flags, environment |
| `jcmd <pid> VM.system_properties` | system properties as the JVM sees them |
| `jcmd <pid> GC.heap_info` | current heap and generation occupancy |
| `jcmd <pid> GC.heap_dump <file>` | a full heap dump — Topic 79 |
| `jcmd <pid> GC.run` | request a full GC (diagnostics only, never in production paths) |
| `jcmd <pid> VM.metaspace` | class metadata, with a per-loader breakdown |
| `jcmd <pid> VM.classloader_stats` | loader count and classes per loader |
| `jcmd <pid> Compiler.codecache` | code cache usage per segment |
| `jcmd <pid> Thread.print` | every thread's stack, plus a deadlock section |
| `jcmd <pid> VM.native_memory summary` | footprint attribution (needs NMT on) |
| `jcmd <pid> VM.native_memory baseline` / `summary.diff` | what grew since a marker |
| `jcmd <pid> VM.stringtable` | interned string table size |
| `jcmd <pid> JFR.start` / `JFR.dump` | flight recording — Topic 78 |
| `jcmd <pid> help` | the full subcommand list on your JDK |

### Gotchas checklist

- [ ] Heap is not the process footprint. Never set `-Xmx` to the container limit.
- [ ] `-Xmx` does nothing for `OutOfMemoryError: Metaspace`.
- [ ] `-Xss` is per thread. Multiply by your thread count before you raise it.
- [ ] Code cache exhaustion is a **warning**, not an error, and it kills throughput.
- [ ] The thread that reports an OOM is usually not the thread that caused it.
- [ ] Exit code 137 with no `OutOfMemoryError` means the **kernel** killed you.
- [ ] The first request to any code path is slower, forever. Class loading is lazy.
- [ ] Never assert a JVM default. Print it with `-XX:+PrintFlagsFinal -version`.
- [ ] `System.nanoTime()` around a call measures cold JIT, GC and safepoints too.
- [ ] `-XX:+HeapDumpOnOutOfMemoryError` should be on in production, with disk space.
- [ ] Locals are per-thread and safe. The **objects** they reference are not.

---

## When would I use this at work?

**1. Triaging any memory-shaped incident, in the first ninety seconds.**
The pod restarted. Somebody says "we're leaking memory". Before opening any dashboard,
you ask: was there an `OutOfMemoryError` in the log, and which one? No error and exit
137 means the kernel and a footprint problem — a completely different investigation from
`Java heap space`, which is a retention problem, which is different again from
`Metaspace`, which is a class-loader problem. That one question routes the incident
correctly and saves hours of the wrong people looking at the wrong graph.

**2. Reviewing the JVM flags in a Dockerfile or a Helm chart.**
Almost every Java service in the industry has flags that were copied from somewhere and
never justified. You can now read them and ask real questions: why is `-Xmx` set to the
limit? Why is `-Xss` 8 MB, and how many threads do we run? Is
`-XX:+HeapDumpOnOutOfMemoryError` on, and is the path writable inside the container?
Is `MaxMetaspaceSize` unbounded on purpose? This review costs ten minutes and prevents
the class of incident in point 1.

**3. Deciding whether a performance problem is even a JVM problem.**
A team is about to spend a sprint on GC tuning. You run the load test once with `-Xint`
and once normally, and once with the GC log on, and you can say: the JIT is worth a
large factor here so the JVM is doing real work, but GC accounts for a small fraction of
wall time, so tuning the collector has a small ceiling — the time is going to Postgres,
and here is the flame graph. Redirecting a sprint away from work that could not have
paid off is worth more than any individual optimisation, and it is a judgement you can
only make with the map on this page.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and boxing**: why `List<Integer>` costs 4–5× `int[]` — the objects
  are on the shared heap with headers, and the array holds references to them. Topic 69
  gives you the exact bytes.
- **12 — HashMap internals**: every `Node` is a heap object. A map with a million
  entries is a million objects for the collector to trace, which is why Topic 70's live
  set is the number that matters.
- **17 — Immutability and safe publication**: `final` fields have memory-model meaning.
  That meaning only exists because the heap is shared, which is this page.
- **19 — Serialization**: deserialization *creates classes and objects* from bytes,
  which is why it touches the class loader, the verifier and the heap all at once.
- **31–32 — Maven and the classpath**: the classpath is the input to the application
  class loader. `NoClassDefFoundError` at runtime with a clean compile is a
  build-and-classpath problem expressed as a JVM error. Topic 67 makes this precise.
- **65 — The load-test gate**: everything in this phase is measured against those
  baselines. Without them you have no control group.

**This unlocks:**
- **67 — Class loading**: the delegation chain, laziness, and why an initialisation
  failure poisons a class for the JVM's lifetime. Read it next; it is the direct
  continuation.
- **68 — Heap generations and TLABs**: the internal structure of the shared heap, and
  why allocation is a pointer bump rather than a free-list search.
- **69 — Object layout**: exactly how many bytes an object occupies, and the ~32 GB
  compressed-oops cliff.
- **70 — GC fundamentals**: GC roots include every thread's stack — which is a direct
  consequence of the per-thread/shared split defined here.
- **71 — G1 in depth** and **72 — ZGC/Shenandoah**: the pluggable part of "pluggable
  collector".
- **73 — Safepoints**: why the JVM has to stop every thread at all, and why the reported
  GC pause is not the stop-the-world duration.
- **74–75 — JIT**: tiered compilation, deoptimisation, inlining and escape analysis, in
  full.
- **76 — Bytecode**: what is actually in a `.class` file and what the operand stack does.
- **77 — JMH**: why the `System.nanoTime()` warning above is not pedantry.
- **78 — Profiling**: attributing wall time and CPU across the areas defined here.
- **79 — Heap dumps**: the shared-heap column, made queryable.
- **80 — Off-heap**: the footprint outside every area on this page.
- **81 — Instrumentation agents**: how an agent rewrites bytecode between loading and
  verification, and why that inflates the code cache.
- **82 — Containers**: cgroup limits, `availableProcessors()`, and the OOMKill in Trap 2.
- **84–96 — Concurrency**: every one of those topics is a rule about the shared column.
- **90 — Thread stacks**: the per-thread column's footprint, at scale.
- **101 — Virtual threads**: continuations that put a thread's stack **on the heap**,
  which is a genuinely startling idea until you have this page's split in your head.

---

*Java baseline 21, running on JDK 25. Two things in this document are deliberately
hedged rather than asserted, and both have a command that settles them on your machine
in under a minute: the **default collector and its mode** — G1 remains the default and
generational ZGC became the default ZGC mode in JDK 23+, but selection also depends on
available memory and CPU count, so run `jcmd <pid> VM.flags` or
`java -XX:+PrintFlagsFinal -version` on the actual JVM rather than believing a
document — and the exact behaviour of `-Xverify:none` on your build, which has moved
from supported to deprecated to ignored across recent releases. Everything else here —
the seven areas, the per-thread/shared split, the tier structure, and the diagnostic
command table — has been stable across JDK 11, 17, 21 and 25, and will still be true
when you next read a thread dump at 2am.*
