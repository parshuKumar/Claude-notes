# 86 — The Java Memory Model I: happens-before, visibility, and reordering

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: this is the topic that explains why an `orderflow` pod goes into `Terminating` and never leaves it — a background inventory reconciler whose shutdown flag is set, observably set, provably set, and which keeps running anyway until Kubernetes `SIGKILL`s it. Nothing in the thread dump says `BLOCKED`. The thread is `RUNNABLE`, burning a core, reading a field that changed thirty seconds ago.

---

## Mechanical statement

Read this three times. Every section below is an elaboration of it.

> **The Java Memory Model is a contract about which writes a read is PERMITTED to
> observe.** It is not a description of caches. It is not a promise about timing. It is
> a set of rules that says, for a given read, which set of writes are legal answers.
>
> **If there is no happens-before edge between a write in one thread and a read in
> another, the read is permitted to return the old value.** Any old value. The default
> value. A value from ten minutes ago.
>
> **And it is permitted to keep returning it forever.** Not "for a while". Not "until
> the cache line is invalidated". **Forever.** There is no clause in the specification
> that says a read must eventually observe a write. "Eventually" is a word from your
> intuition, not from JLS Chapter 17.
>
> **Three independent agents may reorder your program**, and all three are legal:
>
> 1. **The compiler** (`javac`, marginally) and **the JIT** (C1 and C2, enormously) may
>    reorder, eliminate, duplicate and hoist memory operations. The JIT's licence to
>    **hoist a read out of a loop** — which you met as loop-invariant code motion in
>    Topics 74 and 75 — is the entire bug in this document's drill.
> 2. **The CPU** may execute instructions out of order and may buffer stores so that
>    they become visible to other cores later than they were issued.
> 3. **The memory system** may make one core's store visible to a second core before a
>    third, on a sufficiently weak architecture.
>
> **A happens-before edge is the only thing that constrains any of them.** Not a
> `sleep`. Not a retry. Not "surely it will notice". An edge, established by one of a
> short and closed list of language constructs.

Four consequences follow directly, and you should be able to derive each one. **A
correctly-behaving JVM may run your program forever in a loop whose exit condition is
true** — that is not a JVM bug, it is your program being wrong. **The same program
terminates under `-Xint` and hangs under the JIT**, and both behaviours are legal; the
gap between them is not an anomaly, **the gap IS the memory model**, made visible. **You
cannot test your way to correctness here**: a passing test says the permitted-but-unwanted
outcome did not happen on that machine, that JDK, that run; the proof is the
happens-before argument, and testing (Topics 87, 88, 99) catches the argument being
wrong. **And the fix is never "add a delay"** — it is always "establish the edge".

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

This is not a formality and it is not modesty. It is the single most important
paragraph in Phase 9 for you specifically, so read it slowly.

**A single-threaded, run-to-completion runtime cannot have a memory model, because a
memory model answers a question that runtime cannot ask.**

The question a memory model answers is: *given a read, which of the writes that exist
in the program are legal values for it to return?* In Node, that question has exactly
one answer, always, trivially: **the most recent write in program order.** There is one
thread. There is one program order. There is no second observer. A memory model with
one possible answer is not a model; it is an identity function.

Everything downstream of that follows. **There is no visibility question** — you have
never asked "will the other thread see this write", because there is no other thread that
can see anything. **There is no observable reordering question** — V8 reorders your code
constantly: it hoists, it inlines, it eliminates dead stores, it keeps values in registers
across statements, and you have never once cared, because **reordering is only observable
from a second thread** and you do not have one. **There is no staleness** — a value cannot
be stale when there is one reader. **There is no publication problem** — an object cannot
be observed half-constructed when the only observer is the constructor's own thread.

### Do not map this onto the event loop. Explicitly.

You will be tempted, at some point in the next hour, to reach for one of these. *"So it's
like a callback that hasn't been queued yet?"* No. *"Like reading a variable before an
`await` resolves?"* No. *"A race between two `setTimeout`s?"* No. *"So a `volatile` write
is like `queueMicrotask`, forcing the update to be seen?"* No, and this one is actively
dangerous.

**Every one of those maps a visibility problem onto a scheduling problem.** They are
different problems. Scheduling is about *when* work runs. Visibility is about *what a
read returns*, and the answer can be "the wrong thing, permanently, while both threads
are running at full speed on separate cores".

The reason this document forbids the analogy rather than merely discouraging it: **the
wrong model costs more to remove than the analogy saves.** An engineer who believes the
JMM is "about scheduling" writes fixes that look like scheduling fixes — a sleep, a
retry, a yield, a bigger delay before checking — and every one of those fixes appears
to work in testing, because they perturb the JIT's behaviour by accident. Then it
breaks in production, and the engineer's model has no room for the real cause. You will
meet this exact failure in Trap 3 below.

There is no bridge here. Build the model from the mechanism, which is what the rest of
this document does.

### The one honest exception: `SharedArrayBuffer` + `Atomics`

There is exactly one place where JavaScript has a real memory model, and it is worth
naming precisely because it proves the point rather than softening it.

When you allocate a `SharedArrayBuffer` and hand it to two `worker_threads`, you have
created the thing JavaScript otherwise refuses to give you: **two threads with genuine
shared mutable memory.** At that instant the ECMAScript specification acquires a memory
model — a real one, with sequentially-consistent atomic operations, unordered plain
accesses, a happens-before-like partial order, and the explicit statement that a plain
(non-`Atomics`) read of shared memory may observe a stale value.

| JavaScript | Java | Verdict |
|---|---|---|
| `SharedArrayBuffer` — shared memory, opt-in | The Java heap — shared memory, default | **HONEST ANALOGUE for the mechanism**, inverted for the default |
| `Atomics.store(ta, i, v)` / `Atomics.load(ta, i)` | `volatile` write / `volatile` read | **HONEST ANALOGUE** — sequentially consistent, ordering-establishing |
| A plain `ta[i] = v` on a shared buffer | A plain (non-`volatile`) field write | **HONEST ANALOGUE** — unordered, may be observed stale, may be reordered |
| `Atomics.wait` / `Atomics.notify` | `Object.wait` / `notify` (Topic 89) | **PARTIAL** — same role, different API and different spurious-wakeup rules |
| ECMAScript's "agent cluster" memory model | JLS Chapter 17 | **HONEST ANALOGUE** — both exist for the same reason: shared memory plus real parallelism |

**Most JavaScript engineers have never touched any of this.** `SharedArrayBuffer` is
gated behind cross-origin isolation headers in browsers, it is rare in Node
application code, and the vast majority of Node work never allocates one. If you are
among the majority, this table is a signpost, not a shortcut: it tells you that the
ideas in this document are not Java exotica, they are what *any* language must specify
once two threads touch the same memory. If you *have* used `Atomics`, then Topics 86
and 87 will feel like recognition, and the main new thing is that Java's default is the
unsafe one — a plain field is a plain shared-buffer access, and you get one by writing
nothing at all.

### What actually transfers

Almost nothing from the runtime. Two things from elsewhere, and they are both worth
having.

| You know | Here | Why it helps |
|---|---|---|
| Distributed-systems reasoning: "there is no global now", replicas observe writes at different times, you need a consistency model | Two cores are two replicas of memory, and the JMM is the consistency model | **This is the strongest transfer available to you.** If you have reasoned about eventual consistency, read-your-writes, or a stale read from a follower, you already have the shape. The difference: the JMM does not even promise eventual. |
| Reasoning about compiler optimisation from bundlers/minifiers: dead-code elimination, constant folding, hoisting | The JIT does all of that, aggressively, at runtime, with profile data | You already accept that "the code that runs is not the code you wrote". Now that fact becomes observable. |

---

## What is this?

### The definition, stated precisely

The Java Memory Model, specified in **JLS Chapter 17**, defines a partial order over
the actions of a program called **happens-before** (written `hb`). Its central rule is:

> **If action X happens-before action Y, then X's effects are visible to Y, and X
> appears to occur before Y.**
>
> **If there is no happens-before relationship between a write W and a read R of the
> same variable, then R is permitted to return either the value written by W, or any
> other value written to that variable by any thread, including the default value
> written at initialisation.**

Read the second half again. It does not say "R will probably return the stale value" or
"R will return the stale value for a short time". It says **R is permitted to return
any of them, and the specification places no bound on how long that continues.**

Two terms that get used loosely and must not be. **Visibility** — whether a read can
observe a particular write at all. **Ordering** — whether two operations appear to occur
in program order, from the point of view of some other thread. The JMM constrains both
with the same tool: a happens-before edge buys you both at once, and nothing else buys
you either.

### A data race, defined exactly

Java's specification gives "data race" a precise technical meaning, and it is narrower
than the everyday use of the word:

> Two accesses to the same variable form a **data race** if at least one of them is a
> write, they are performed by different threads, and they are **not ordered by
> happens-before**.

Three things follow that surprise people. **Two reads are never a race**, no matter how
many threads — nothing changes. **A race is not "two threads at the same time"**: two
threads a full second apart race if there is no edge between them, because timing is not
in the definition. And **a program with no data races is guaranteed sequentially
consistent** — the JMM's actual promise, and a strong one: *if you correctly synchronise
every shared access, you may reason about your program as if all threads' actions were
interleaved in a single global order.* Every hard part of this phase is the price of that
guarantee, and the guarantee is what makes correct concurrent Java tractable at all.

**That last point is the whole strategy of Phase 9.** You do not learn to reason about
reordering. You learn to establish enough edges that you never have to.

### The complete happens-before edge list

**This is the reference table. It is the thing you will come back to for years.** There
are not many edges. They are closed — this list is exhaustive for the language itself;
everything else in `java.util.concurrent` is documented in terms of these.

| # | Edge | The exact rule | Where you meet it |
|---|---|---|---|
| 1 | **Program order** | Within a **single** thread, each action happens-before every action that comes later in that thread's program order. | Everywhere. It is why single-threaded code works. Note carefully: this constrains what *that thread* observes, **not** what other threads observe. |
| 2 | **Monitor lock** | An **unlock** of a monitor happens-before every **subsequent lock** of *that same* monitor. | Topic 85. `synchronized` blocks and methods. Different monitors give you nothing. |
| 3 | **Volatile** | A **write** to a `volatile` field happens-before every **subsequent read** of *that same* field. | Topic 87. Also applies to `AtomicXxx` and `VarHandle` with the right access mode (Topic 95). |
| 4 | **Thread start** | `threadA.start()` happens-before **any action** in the started thread. | Everything the starting thread did before `start()` is visible inside the new thread. Free, and often overlooked. |
| 5 | **Thread join** | **Every action** in a thread happens-before another thread successfully returns from `join()` on it. | The other half of the pair. `start`/`join` bracket a thread's whole life with edges. |
| 6 | **Thread termination** | Every action in a thread happens-before any other thread detects that it has terminated — `join()` returning, or `isAlive()` returning `false`. | Same edge as 5 from the other side. |
| 7 | **Interruption** | A call to `threadB.interrupt()` happens-before thread B detects the interrupt (by `InterruptedException` or by `isInterrupted()`/`interrupted()` returning true). | Topic 84's cooperative cancellation. This is why interruption is a *correct* shutdown mechanism and a plain flag is not. |
| 8 | **Final field freeze** | The **end of a constructor** (the freeze action for its `final` fields) happens-before, for any thread, a read of any `final` field of that object — **provided the reference did not escape the constructor**. | **Topic 88.** This is the one edge you get without any synchronisation at all, and the one with a precondition you can violate. |
| 9 | **Default values** | The write of the default value (`0`, `false`, `null`) to every field happens-before the first action of every thread. | Why a read can return `0` from an object whose constructor set the field to `5` — the default write is legally visible. |
| 10 | **Transitivity** | If A `hb` B and B `hb` C, then A `hb` C. | **The rule that makes the others useful.** It is how a lock release/acquire pair carries *all* of a thread's prior writes, not just the ones inside the block. |

**Read edge 10 again, because it is the one that does the real work.** Consider:

```java
// Thread A
inventory.reserved = 5;          // plain write
inventory.lastAuditAt = now;     // plain write
synchronized (lock) { flag = true; }   // unlock

// Thread B
synchronized (lock) { seen = flag; }   // lock
int r = inventory.reserved;      // plain read
```

Thread B is guaranteed to see `reserved == 5`. Not because `reserved` is `volatile` —
it is not. The chain is: A's plain writes `hb` A's unlock (edge 1, program order); A's
unlock `hb` B's lock (edge 2); B's lock `hb` B's plain reads (edge 1). Transitivity
joins them. **The synchronised block carries everything the thread did before it.**

This is the mental move that separates people who "use `synchronized`" from people who
reason about the JMM: **an edge is not a property of the variable it is written on. It
is a barrier in time that carries everything across.**

### The edges you actually use, derived from the ten

Every `java.util.concurrent` guarantee is documented in its Javadoc as a happens-before
relationship, and every one is **built from the ten above** rather than being a new rule:
`submit` before the task starts; task completion before `Future.get()` returns;
`countDown()` before a returning `await()`; barrier arrival before the barrier action;
`Semaphore.release()` before a subsequent `acquire()`; `BlockingQueue.put(x)` before the
`take()` that returns `x`; a `ConcurrentHashMap` write to key K before a subsequent read
of K; `Lock.unlock()` before a subsequent `lock()` (Topic 94); `AtomicX.set` behaving as
a volatile write (Topic 95); static-initialiser completion before first use (Topic 67).

**Practical rule you can carry today:** if two threads communicate through a
`java.util.concurrent` structure, the edge is already there and you do not need to
think about it. If they communicate through a plain field, there is no edge and you
must make one.

### What the JMM does NOT say

An unusually valuable list, because most wrong beliefs are on it.

| People believe | The specification says |
|---|---|
| "The other thread will see it eventually" | Nothing. There is **no eventual-visibility clause.** An implementation is free to never propagate the write. |
| "Caches are coherent, so it'll be fine" | Cache coherence is a *hardware* property and it is real (see Machine-level reality). It does not help, because the write may never have left the store buffer, and the read may never have been executed at all — the JIT deleted it. |
| "It's only a problem if the writes are close together" | Timing is not in the definition of a data race. A one-second gap and a one-nanosecond gap are the same to the model. |
| "`synchronized` is only about mutual exclusion" | It is equally about the edge. A `synchronized` block with a single thread inside it still publishes. |
| "Reads are safe; only writes need synchronising" | A race requires only *one* write. The unsynchronised **read** is half the race and is equally the defect. |
| "It works on my machine, so the code is fine" | The model permits outcomes; an implementation need not produce all permitted outcomes on all runs. Your machine is one sample from a space you did not enumerate. |
| "64-bit values are atomic on 64-bit hardware" | JLS §17.7: `long` and `double` reads/writes are **not guaranteed atomic** unless `volatile`. In practice 64-bit HotSpot makes them atomic; that is an implementation fact, not a guarantee. |

---

## Why does it matter?

**1. Because the failure mode is a hang, and hangs are the worst incident shape.**

A crash gives you a stack trace. A slow request gives you a percentile. A hang gives you
a thread that is `RUNNABLE`, consuming CPU, with a stack pointing at a loop condition any
reader will tell you is false. Every diagnostic instinct you have — check the locks, the
database, the pool — returns clean, because nothing is blocked.

**2. Because the "fix" that appears to work is the most dangerous outcome available.**

Someone adds a `println` to debug it and the hang disappears. Someone adds
`Thread.sleep(10)` and it disappears. Someone adds a metrics counter and it disappears.
**All three are accidents**, and all three ship. The bug returns when the logging is
removed in a cleanup PR, or the metric is made lock-free, or a JDK upgrade changes an
inlining decision. You will meet all three in Trap 3.

**3. Because it is the foundation of every remaining topic in Phase 9.**

Topic 87 is one edge (volatile) in full mechanical detail. Topic 88 is one edge
(final-field freeze) and its precondition. Topic 92's `ConcurrentHashMap` reads are
lock-free precisely because they are volatile reads of `Node.val` — an edge, deliberately
placed. Topic 94's `ReentrantLock` and Topic 95's atomics are edges with different
ergonomics. Topic 99's jcstress exists to falsify a happens-before argument you got
wrong. **You cannot skip this and pick it up later. Everything after it is a special case
of it.**

**4. Because it changes what you consider a review comment.** "This field is written by
the scheduler thread and read by request threads with no synchronisation" becomes a
defect you can name in a pull request, with a citation, before anyone has run anything —
a materially different level of engineer from one who can only report bugs that
reproduced.

**5. Because on `orderflow` it is falsifiable in ten minutes.** You have a containerised
service, a load profile, a recorded baseline, and a `terminationGracePeriodSeconds`. You
can produce a pod that will not terminate, prove the mechanism with one JVM flag, and fix
it with one keyword. Very few important concepts here are that cheap to demonstrate.

---

## Machine-level reality

Everything in this section is about *why* the specification is so weak. The
specification is weak because the hardware and the compiler are strong, and Java chose
to expose the performance rather than pay for a guarantee most code does not need.

### Layer 1: the store buffer

A CPU core does not write directly to memory, or even to its own L1 cache, on the
critical path. It writes into a **store buffer**: a small, per-core, FIFO-ish queue of
pending writes. The core continues executing immediately. The buffer drains into the
cache hierarchy later.

```
   Core 0                         Core 1
+----------+                   +----------+
| pipeline |                   | pipeline |
+----------+                   +----------+
     |                              |
     v                              v
+--------------+              +--------------+
| STORE BUFFER |              | STORE BUFFER |     <-- per core, NOT coherent
|  stock <- 5  |              |              |
+--------------+              +--------------+
     |                              |
     v                              v
+----------------------------------------------+
|      L1 / L2 / L3 caches, kept COHERENT       |
|              stock = 10                       |
+----------------------------------------------+
```

**The critical fact: the store buffer is not part of the coherence protocol.** Core 0's
write to `stock` sits in Core 0's store buffer. Core 1 cannot see it. Core 1 reads
`stock` from the coherent cache and gets `10`, which is a perfectly correct read of a
perfectly coherent cache — the new value simply has not arrived yet.

The store buffer is also why the one reordering that x86 permits is exactly the one it
permits: a **later load** can complete before an **earlier store to a different
address** drains, because the load goes to cache and the store is still queued.

### Layer 2: cache coherence (MESI) — and why it does not save you

The caches themselves *are* kept coherent, by a protocol usually described as **MESI**:
every cache line in every core's cache is **M**odified (this core has the only copy and
changed it), **E**xclusive (only copy, unchanged), **S**hared (several clean copies), or
**I**nvalid (stale, must not be used).

To write a line, a core must first obtain it in `M` state, which requires **invalidating
every other core's copy** — a broadcast on the interconnect. That is why contended
writes to one variable from many cores are expensive (Topic 95's CAS ping-pong) and why
two unrelated variables in one 64-byte line contend (Topic 96's false sharing).

**Here is the point that trips people up, and it is worth stating flatly:**

> **Cache coherence guarantees that no two cores ever hold different values for the same
> address at the same time. It guarantees nothing about *when* your write reaches the
> cache, and nothing at all about the order in which two of your writes to two different
> addresses become visible.**

So "the caches are coherent" is true, and it is not an argument that your code is
correct. The staleness lives in the store buffer, which is before the coherent domain,
and the reordering lives in the pipeline and the compiler, which are before that.

### Layer 3: x86-TSO versus aarch64 — and which way it cuts on your Mac

You are on macOS, and almost certainly on Apple Silicon, which is **aarch64**. This
matters, and it matters in a specific direction that you must state correctly.

**x86-64 implements Total Store Order (TSO).** The hardware guarantees:

| Reordering | Allowed on x86-TSO? |
|---|---|
| Load then Load | **No** |
| Load then Store | **No** |
| Store then Store | **No** |
| Store then Load (different addresses) | **YES** — the one hole, and it comes from the store buffer |

**aarch64 is a weakly ordered architecture.** All four of those reorderings are
permitted unless you use explicit barriers (`dmb`) or the acquire/release load and store
instructions (`ldar`, `stlr`).

Two practical consequences, and they point in opposite directions:

> **Which way it cuts, in your favour:** hardware-level reordering bugs that x86 hides
> completely can become **observable** on Apple Silicon. For Topics 87 and 88, that makes
> your laptop a *better* instrument than a typical x86 CI runner. A jcstress run on your
> M-series Mac can surface an outcome that an x86 build server would never produce in a
> thousand years of running.

> **Which way it cuts, against you:** it means a jcstress result from your Mac and a
> jcstress result from x86 CI are **different experiments**, and neither is the whole
> answer. A test that is clean on x86 tells you almost nothing about aarch64. A test
> that is clean on aarch64 is stronger evidence but is still not proof.

> **And the honest limit, stated once and applied everywhere in this phase: I will never
> tell you a race "will" reproduce.** Whether a given reordering is observed on a given
> run depends on the JIT's output, which cores the OS chose (and on Apple Silicon,
> whether it chose performance cores or efficiency cores), what else is running, and
> luck. Every drill in Topics 86–88 has a "you saw nothing" row in its result table, and
> that row is a legitimate outcome, not a failed experiment.

**One crucial exception, specific to this document:**

> **The Topic 86 stop-flag bug is a COMPILER bug, not a hardware one.** The JIT hoists
> the field read out of the loop, so the read is not performed at all. There is no cache
> to be stale, no store buffer to drain, no barrier that would have helped. **This
> happens identically on x86 and on aarch64.** Architecture is irrelevant to this
> drill — which is exactly why it is the right first drill: it isolates the compiler from
> the hardware. Architecture becomes decisive at Topic 87 and Topic 88.

Say that distinction out loud once. Engineers who have half-learned this topic blame
"cache flushing" for the stop-flag hang, and the word "cache" does not appear anywhere in
the true explanation.

### Layer 4: the four barrier types

A **memory barrier** (or fence) is an instruction that constrains reordering across it.
Java's JMM is implemented, at the level below the JIT, using four conceptual barrier
types. HotSpot's own source uses exactly these names, and you should too.

| Barrier | Written as | What it prevents | Cost |
|---|---|---|---|
| **LoadLoad** | `Load1; LoadLoad; Load2` | `Load2` and any subsequent load being reordered before `Load1`. Guarantees `Load1`'s data arrives first. | Cheap or free on TSO; a real `dmb` on aarch64 |
| **StoreStore** | `Store1; StoreStore; Store2` | `Store2` becoming visible to other cores before `Store1`. | Free on TSO (stores are already ordered); a real barrier on aarch64 |
| **LoadStore** | `Load1; LoadStore; Store2` | `Store2` being reordered before `Load1` completes. | Cheap or free on TSO |
| **StoreLoad** | `Store1; StoreLoad; Load2` | `Load2` executing before `Store1` is visible to all cores. **This is the one that requires draining the store buffer.** | **The expensive one, on every architecture.** `mfence` or a `lock`-prefixed op on x86; `dmb ish` on aarch64 |

**`StoreLoad` is the expensive barrier and the important one.** It is the only barrier
x86 needs at all, because it is the only reordering x86 permits. It is also the barrier
that a `volatile` *write* must emit, which is exactly why volatile writes cost
meaningfully more than plain writes and volatile reads cost nearly nothing on x86. That
asymmetry is Topic 87's central mechanical fact; it is previewed here so the four names
are in place first.

### Layer 5: how the edges lower to barriers

The full treatment is Topic 87. The one-screen version, so you can see the shape:

| Java construct | Conceptual barriers | On x86-64 | On aarch64 |
|---|---|---|---|
| `volatile` **read** | `LoadLoad` and `LoadStore` **after** the read | An ordinary `mov`. TSO already forbids these reorderings, so **no instruction is needed** | `ldar` — a load-acquire instruction |
| `volatile` **write** | `StoreStore` **before**, `StoreLoad` **after** | A `mov` followed by a `lock addl $0, (%rsp)` (a cheap locked no-op used as a full fence) or `mfence` | `stlr` — a store-release — and typically a `dmb ish` to supply `StoreLoad` |
| `synchronized` **enter** | Acquire semantics — like a volatile read | The `lock cmpxchg` on the mark word is already a full barrier | `ldaxr`/`stlxr` or LSE atomics with acquire semantics |
| `synchronized` **exit** | Release semantics — like a volatile write | Ordered by the locked instruction | `stlr` and/or `dmb ish` |
| `final` field **freeze** at constructor end | `StoreStore` before the constructor returns | Free (TSO orders stores already) | A real `dmb ishst` |

Two things to take from that table. **First: on x86, a `volatile` read is free and a
`volatile` write is not.** The read compiles to the same instruction a plain read does;
the write needs the fence. That is why "make it volatile" is nearly costless for a flag
read constantly and written once, and a real cost for something written in a hot loop.
**Second: on aarch64 the JVM emits actual instructions for both** — `ldar` and `stlr`
exist for this. Reads are no longer free, which is one more reason a benchmark from an
x86 machine does not transfer to your Mac.

### Layer 6: the constructor freeze, stated here and developed at Topic 88

`final` fields are the one place the JMM gives you an edge with no synchronisation at
all, and the mechanism is worth naming now because it appears in edge 8 above.

> At the end of a constructor in which a `final` field is set, the JVM performs a
> **freeze action** on that field. Any thread that obtains a reference to that object —
> through a reference that was **not** published before the constructor finished — is
> guaranteed to see the correctly-initialised value of every `final` field, with no lock,
> no volatile, no synchronisation of any kind.
>
> **Mechanically this is a `StoreStore` barrier at the end of the constructor**, which
> prevents the store of the object's *reference* being reordered before the stores of
> its `final` *fields*.

Two limits, both Topic 88's material: **the precondition** — if `this` escapes the
constructor (a listener registration, a `Thread` started inside it, a `this` stored to a
static) the guarantee is void, because you published a reference before the freeze; and
**the freeze covers the field, not what it points at** — a `final List<OrderLine> lines`
guarantees you see the correct list *reference* and nothing about its *contents*, which
is the whole reason Topic 17 insisted on defensive copying.

### Layer 7: the JIT — the one that actually causes this document's bug

Everything above is the hardware. **On the drill in this document, the hardware is
innocent.** The culprit is C2, and it is doing something entirely reasonable.

Consider:

```java
private boolean running = true;      // plain field, no volatile

public void reconcile() {
    while (running) {                 // a read of `running` every iteration
        adjustOneSku();
    }
}
```

C2 performs **loop-invariant code motion (LICM)** — Topic 74's territory. It asks: is
`running` modified inside this loop? It analyses the loop body. Nothing in the loop
writes `running`. **Therefore, under the rules of the JMM, C2 is permitted to conclude
the read is loop-invariant and hoist it out**, producing, conceptually:

```java
// what C2 is PERMITTED to generate. Illustration of the transformation,
// not generated code.
boolean hoisted = running;            // read ONCE, before the loop
while (hoisted) {
    adjustOneSku();
}
```

Or even, since `hoisted` is then a loop-invariant `true`:

```java
while (true) {
    adjustOneSku();
}
```

**Is that legal?** Yes, unambiguously. The JMM constrains what a thread may observe of
*other* threads' writes only where a happens-before edge exists. There is no edge here.
Therefore the compiler is entitled to assume no other thread writes `running`, and to
optimise on that basis. **This is not a compiler bug. This is the compiler correctly
using the licence your code gave it.**

**And this is the exact licence you already know from Topics 74 and 75.** You have
already seen C2 hoist a read out of a loop and been told it was a good optimisation. It
still is. What changes here is that you now know the price: the licence to hoist is the
same licence that makes an unsynchronised flag useless.

Three further consequences worth having. **The field read may not merely be stale — it
may not happen**, so there is no load instruction to be stale, which is why "flush the
cache" is not a coherent proposed fix. **Under `-Xint` the loop is interpreted**, and the
interpreter re-reads the field on every `getfield` bytecode because it does not optimise,
so the loop terminates — **that difference is the whole drill.** **Under C1 only
(`-XX:TieredStopAtLevel=1`) the behaviour may differ again**, because C1 optimises far
less aggressively than C2; whether it hoists this particular read is a question for your
JDK, and the drill has you check it rather than believe me.

---

## Concurrency trace

**Read this before you read any correct code.** Every bug in this phase is a specific
interleaving. If you can write the interleaving, you understand the bug; if you cannot,
you are pattern-matching.

### The scenario

`orderflow` runs a background **inventory reconciler**: a daemon thread that walks the
hot-SKU set and corrects the in-memory reservation counters against Postgres. It is
started by a `@PostConstruct` and stopped by a `@PreDestroy` when the Spring context
closes — which is what happens when the container receives `SIGTERM` during a rolling
deploy.

The code under trace. It is fourteen lines and every reviewer on the team has approved
it:

```java
@Component
public class InventoryReconciler {

    private boolean running = true;              // <-- plain field. No volatile.
    private Thread worker;

    @PostConstruct
    public void start() {
        worker = new Thread(this::loop, "inventory-reconciler");
        worker.setDaemon(true);
        worker.start();
    }

    private void loop() {
        while (running) {                        // <-- the read
            reconcileNextSku();                  // touches the in-memory counters
        }
    }

    @PreDestroy
    public void stop() {
        running = false;                         // <-- the write
    }
}
```

Thread A is an `orderflow` **order-placement / lifecycle thread** — during the trace it
is the Spring container's shutdown thread, which is the same thread that has just
finished draining in-flight order placements. Thread B is the **inventory reconciler**.

### The interleaving

| Step | Thread A (order placement / shutdown) | Thread B (inventory reconciler) | Shared state / what is visible |
|---|---|---|---|
| 1 | Context starts. `start()` runs. `running` was set `true` in the field initialiser | — | `running = true` in memory. There **is** an edge here: `Thread.start()` (edge 4), so B is guaranteed to see `true` |
| 2 | Serving order placements normally | Enters `loop()`. Reads `running` → `true`. Calls `reconcileNextSku()` | `running = true` |
| 3 | — | Iterates. The loop runs a few thousand times, **interpreted**, re-reading `running` from memory each time | `running = true` |
| 4 | — | The loop's back-edge counter crosses the OSR threshold. **C2 compiles `loop()` and swaps the running frame over to compiled code (on-stack replacement).** During compilation it observes no write to `running` inside the loop and **hoists the read out** | `running = true`. From this instant, **B no longer reads the field at all.** The compiled loop tests a value held in a register |
| 5 | — | Continues reconciling, at full speed, on a performance core | `running = true` |
| 6 | Kubernetes begins a rolling deploy. The pod receives `SIGTERM`. Spring begins `close()` | still looping | `running = true` |
| 7 | Drains in-flight order placements. All HTTP work completes cleanly | still looping | `running = true` |
| 8 | `@PreDestroy` fires. Executes `running = false` | still looping | **A's write lands.** In A's store buffer first, then in the coherent cache. **`running` in memory is now `false`** |
| 9 | Returns from `stop()`. Logs "reconciler stopped". **The log line is true from A's point of view and false in fact** | still looping. **The compiled loop does not read the field. There is no load to be stale** | `running = false` in memory; B's register still holds `true` |
| 10 | Spring finishes closing the context. `main` returns | still looping | `running = false` |
| 11 | The JVM does not exit — or exits, and B is killed mid-write to a counter. Either branch is a defect | still looping | `running = false` |
| 12 | Kubernetes waits out `terminationGracePeriodSeconds` (30s by default) | **still looping.** 30 seconds of reconciliation nobody asked for | `running = false` |
| 13 | Kubernetes sends `SIGKILL`. The pod dies | Killed at an arbitrary point, possibly between reading a counter and writing it back | `running = false` |
| 14 | Meanwhile the **new** pod has been `Ready` since step 10 and started **its own** reconciler | Old pod's reconciler was also running for those 30 seconds | **Two reconcilers, on two pods, both adjusting the same hot-SKU counters against the same Postgres rows** |
| **15** | **OUTCOME, in business terms: for thirty seconds on every single deploy, two inventory reconcilers ran concurrently against the same hot SKUs. Each applied the same correction. Hot-SKU stock was decremented twice for the same reconciliation, so `orderflow` believed it had less stock than it did, refused orders it could have filled, and — on the SKUs where the correction went the other way — accepted orders it could not fill and oversold. Every deploy took thirty seconds longer than it should have, and the rollout dashboard showed pods stuck in `Terminating` with no error anywhere. Nobody connected the two facts for four months.** | | |

### What to take from this trace

**Step 4 is the entire topic.** Nothing "went wrong" at step 4. The JIT performed a
standard, valuable, decades-old optimisation, using an assumption your code explicitly
gave it permission to make. The bug was written at line 3 of the class, when `volatile`
was not typed.

**Step 9 is why this is expensive to find.** Thread A's write really did happen. If you
attach a debugger and inspect the object, `running` is `false`. If you add an actuator
endpoint that reports `running`, it reports `false`. The field is correct. The reader is
not reading it.

**Step 13 is why the data damage is invisible.** The reconciler is killed at an
arbitrary point. There is no exception, no log line, no failed health check. The
resulting counter drift looks exactly like ordinary reconciliation noise.

### The same bug without the JIT — the pure hardware variant

The trace above is the compiler variant, and it is the one your drill reproduces. There
is a second, weaker variant that needs no compiler at all, and you should be able to
write it too, because it is the shape that Topics 87 and 88 exploit:

| Step | Thread A (order placement) | Thread B (inventory reconciler) | Shared state / what is visible |
|---|---|---|---|
| 1 | — | Reads `running` → `true` from cache | `running = true` everywhere |
| 2 | Writes `running = false` | — | The write is in **A's store buffer**. Main memory and B's cache still say `true`. Nothing is incoherent — the write has not entered the coherent domain yet |
| 3 | Writes `lastShutdownAt = now` | — | Also buffered. On **aarch64** these two stores may drain **in either order**; on x86-TSO they drain in program order |
| 4 | — | Reads `running` → `true`. A perfectly correct read of a perfectly coherent cache | Still `true` from B's view |
| 5 | A's store buffer drains | — | `running = false` now visible |
| 6 | — | Next read returns `false`. Loop exits | Terminates — **this time** |
| **7** | **OUTCOME: the loop exited, which is worse than if it had hung. The bug is still present, it merely did not fire. This is the run that gets committed, passes CI on x86, and hangs in production. The specification permits step 4 to repeat indefinitely; on this run it did not.** | | |

Hold both variants. **The first says the read may not happen. The second says the read
may return a stale value.** Both are permitted by the same missing edge, and one keyword
fixes both.

---

## Example 1 — minimal

The smallest program that shows the gap. No Spring, no framework, no threads library.

```java
package com.orderflow.lab.jmm;

import java.util.concurrent.TimeUnit;

/**
 * The canonical non-volatile stop flag.
 *
 * Run it two ways:
 *   java -Xint  ... StopFlag     -> the loop is INTERPRETED; the read happens every
 *                                  iteration; the program is very likely to terminate.
 *   java        ... StopFlag     -> the loop is JIT-COMPILED; the read may be hoisted
 *                                  out of the loop; the program may never terminate.
 *
 * THE GAP BETWEEN THOSE TWO RUNS IS THE JAVA MEMORY MODEL.
 *
 * The word "may" is load-bearing in both sentences. Neither behaviour is promised by
 * anything. What IS promised: with no happens-before edge, the JVM is PERMITTED to
 * never show the write to the reader.
 */
public final class StopFlag {

    /** Plain field. Written by main, read by the worker. NO happens-before edge. */
    private static boolean running = true;

    /** Observed so the JIT cannot delete the loop body entirely (Topic 75). */
    private static long iterations = 0;

    public static void main(String[] args) throws Exception {

        Thread worker = new Thread(() -> {
            long local = 0;
            while (running) {          // <-- THE READ
                local++;
            }
            iterations = local;
            System.out.println("worker: observed running=false, exited after "
                               + local + " iterations");
        }, "flag-worker");

        worker.start();

        // Give the loop time to become hot enough for on-stack replacement.
        // This sleep is NOT a synchronisation mechanism. It exists only to let the
        // JIT compile the loop before the write happens. Do not read it as a fix.
        TimeUnit.MILLISECONDS.sleep(1000);

        System.out.println("main: writing running=false");
        running = false;               // <-- THE WRITE. No edge to the read above.
        System.out.println("main: write complete; joining worker with a 10s timeout");

        worker.join(10_000);

        if (worker.isAlive()) {
            System.out.println("RESULT: worker is STILL ALIVE 10s after the write.");
            System.out.println("        The write happened. The reader is not reading.");
            System.exit(1);            // non-zero so a script can detect the hang
        } else {
            System.out.println("RESULT: worker terminated. iterations=" + iterations);
        }
    }
}
```

Four deliberate choices in that file, each of which is a way the experiment can be
ruined:

| Choice | Why |
|---|---|
| The loop body is `local++` on a **local** variable | If the loop body called `System.out.println`, the call would be a synchronised method and would supply an accidental edge. See Trap 3. |
| `iterations` is written **after** the loop, not inside it | A write to a static inside the loop changes the loop's optimisation profile and could suppress the hoist. |
| `join(10_000)` with a timeout, and `System.exit(1)` | So the program **terminates and reports**, instead of hanging your terminal. You want a result, not a stuck shell. |
| The `sleep(1000)` before the write | The loop must be **hot** — compiled — before the write. Without the sleep the write may land while the loop is still interpreted, and you get the wrong answer for the wrong reason. |

Run it both ways:

```bash
mkdir -p ~/java-lab/86 && cd ~/java-lab/86
mkdir -p src/com/orderflow/lab/jmm out logs
# save the file to src/com/orderflow/lab/jmm/StopFlag.java
javac -d out src/com/orderflow/lab/jmm/StopFlag.java

echo "=== A: interpreter only (the CONTROL) ==="
java -Xint -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"

echo "=== B: default, JIT enabled (the EXPERIMENT) ==="
java -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"

echo "=== C: C1 only, no C2 ==="
java -XX:TieredStopAtLevel=1 -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"

echo "=== D: C2 only, no tiering ==="
java -XX:-TieredCompilation -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"

echo "=== E: JIT enabled, but no on-stack replacement ==="
java -XX:-UseOnStackReplacement -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"
```

**WHAT TO LOOK FOR:** the exit code and the final `RESULT:` line of each run. Write down
all five before reading the table.

| What you see | What it means |
|---|---|
| A (`-Xint`) terminates, B (default) hangs and exits 1 | **The drill has fired. This is the demonstration.** Same source, same JVM, same machine, opposite outcomes, and the only difference is whether the JIT was allowed to compile. **That gap is the memory model.** |
| A terminates, B terminates | The hoist did not occur on this run. **This is a legitimate outcome and not a failed experiment.** Try D (C2 only, no tiering), raise the sleep to 3000 ms so the loop is definitely compiled, and check `-XX:+PrintCompilation` output to confirm the loop method was actually compiled before the write. Then read the honesty rule below. |
| A hangs too | Something else is wrong. `-Xint` should re-read the field on every `getfield`. Check you really passed `-Xint` (`java -Xint -version` should print `interpreted mode`), and check you did not accidentally make the flag `volatile`. |
| C (C1 only) terminates but D (C2 only) hangs | Very informative: **the aggressive hoist is a C2 optimisation on your JDK.** Record this — it is the reason a bug can appear only under sustained load, when a method finally reaches the C2 threshold. |
| E (`-XX:-UseOnStackReplacement`) terminates but B hangs | Also very informative: the loop is compiled *while running* by OSR. Without OSR the already-running frame stays interpreted, so the read keeps happening. Topic 74's OSR, seen from the correctness side. |
| Different runs of B disagree with each other | **Expected, and the most important thing you can observe.** The outcome is not deterministic. That is precisely why "it passed" is not evidence. |

> **Honesty rule 1, stated here and repeated in every drill in Topics 86–88:**
> **A run in which the bug did not appear is NOT evidence that the code is correct.** It
> is evidence that this machine, this JDK, this JIT state and this run did not produce
> the permitted-but-unwanted outcome. The proof of correctness is the happens-before
> argument. Experiments catch a wrong argument; they never confirm a right one.

### The fix, and what makes it a fix

```java
private static volatile boolean running = true;
```

One keyword, buying three things precisely. **Edge 3 from the table**: the write
happens-before every subsequent read of that field — a specification-level guarantee, not
a hint. **The JIT may no longer hoist the read**: a `volatile` read is a memory operation
the compiler must not remove or move across other volatile operations, so LICM does not
apply. **The CPU is given the necessary barriers**: on x86 the read is a plain `mov` and
the *write* gets a fence; on aarch64 the read becomes `ldar` and the write `stlr`.

Run the five configurations again with `volatile` in place. **Every one should
terminate.** If any does not, stop and investigate — it would mean either a JVM bug or,
far more likely, that the flag is not the only thing wrong.

### The other correct fix, which is better in production

```java
// Interruption is edge 7, and it also unblocks a thread sitting in sleep(),
// wait(), a blocking queue take(), or an interruptible channel read.
while (!Thread.currentThread().isInterrupted()) { reconcileNextSku(); }
// ... and from the shutdown path:  worker.interrupt();
```

`Thread.interrupt()` establishes a happens-before edge with the interrupted thread's
detection of it (edge 7), **and** it works on a thread that is parked rather than
spinning. A `volatile boolean` cannot wake a thread blocked in `queue.take()`.
Interruption can. Topic 84 argued this on lifecycle grounds; the JMM gives you the
second, independent reason. **In real code you often want both** — interrupt for wakeup,
and a `volatile` flag so the loop's intent is explicit and survives an
`InterruptedException` that some library swallowed.

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
| Baseline artefacts | `/docs/java/baselines/` — p50/p95/p99/p999 per endpoint |
| Deployment | Kubernetes rolling update, `terminationGracePeriodSeconds: 30` |

### The code that ships

This is a realistic `orderflow` component. It is not a strawman; variants of it exist in
production systems everywhere.

```java
package com.orderflow.inventory;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

/**
 * Walks the hot-SKU set continuously, reconciling the in-memory reservation counters
 * against the authoritative Postgres rows. Reduces the number of obviously-impossible
 * orders that reach the database.
 *
 * THIS CLASS CONTAINS THE TOPIC 86 DEFECT. It is here to be diagnosed, not copied.
 */
@Component
public class InventoryReconciler {

    private static final Logger log = LoggerFactory.getLogger(InventoryReconciler.class);

    private final InventoryRepository repository;
    private final HotSkuRegistry hotSkus;
    private final ReservationCounters counters;

    /** DEFECT: written by the Spring shutdown thread, read by the worker thread,
     *  with no happens-before edge between them. */
    private boolean running = true;

    private Thread worker;

    public InventoryReconciler(InventoryRepository repository,
                               HotSkuRegistry hotSkus,
                               ReservationCounters counters) {
        this.repository = repository;
        this.hotSkus = hotSkus;
        this.counters = counters;
    }

    @PostConstruct
    public void start() {
        worker = new Thread(this::loop, "inventory-reconciler");
        worker.setDaemon(true);
        worker.start();
        log.info("inventory reconciler started");
    }

    private void loop() {
        while (running) {                       // <-- THE READ
            for (String sku : hotSkus.snapshot()) {
                long authoritative = repository.availableUnits(sku);
                counters.setBaseline(sku, authoritative);
            }
        }
        log.info("inventory reconciler loop exited");   // never printed
    }

    @PreDestroy
    public void stop() {
        log.info("stopping inventory reconciler");
        running = false;                        // <-- THE WRITE
        log.info("inventory reconciler stopped");       // printed, and misleading
    }
}
```

### What you observe, in the order you observe it

1. **Deploys are slow.** Every rolling update takes about thirty seconds longer per pod.
   Nobody escalates a slow deploy.
2. **Pods sit in `Terminating`** for the full grace period, then disappear. No error.
3. **The application log looks perfect.** `stopping inventory reconciler` appears.
   `inventory reconciler stopped` appears. What never appears is
   `inventory reconciler loop exited`, and nobody has that line memorised.
4. **CPU on the terminating pod stays high** for the whole grace period. Dismissed as
   "shutdown work".
5. **Hot-SKU stock counters drift** on deploy days — occasional oversells and occasional
   false "out of stock" on exactly the SKUs the reconciler touches. Attributed to the
   Topic 52 optimistic-locking retry logic, which is innocent.
6. **A thread dump shows nothing wrong.** This is the moment that matters.

### The thread dump, and why it is so misleading

```bash
kubectl exec -it <old-pod> -- jcmd 1 Thread.print > dump.txt
grep -A15 'inventory-reconciler' dump.txt
```

**WHAT TO LOOK FOR:** the thread's **state**, and the absence of any lock information.

| What you see | What it means |
|---|---|
| `"inventory-reconciler" #NN daemon prio=5 ... java.lang.Thread.State: RUNNABLE` with a stack inside `loop` or `reconcileNextSku` | **This is the fingerprint of the bug.** The thread is not blocked, not waiting, not parked. It holds no monitor and waits for none. It is *running*, at full speed, doing work it was told to stop doing thirty seconds ago. |
| No `- waiting to lock` and no `- locked` lines for this thread | Rules out Topic 85 contention and Topic 94 deadlock in one glance. A deadlocked thread is `BLOCKED`; this one is not. |
| The same stack in two dumps taken seconds apart, with the loop method on top | Consistent with a hot loop. Take a third dump to be sure it is not coincidence. |
| The thread state is `TIMED_WAITING` or `BLOCKED` instead | **Different bug.** You are not looking at a visibility problem. Go to Topic 85, 89 or 94. |
| No `inventory-reconciler` thread at all | It already exited, or the `@PostConstruct` did not run. Check the startup log. |

**Say the diagnostic sentence out loud, because it is the one that gets you to the
answer in production:**

> *A `RUNNABLE` thread that will not stop, holding no locks, with a loop condition that
> is demonstrably false, is a visibility bug until proven otherwise.*

Every other hang shape in this phase shows up as `BLOCKED` or `WAITING`. This one does
not, and that is what makes it distinctive rather than what makes it hard.

### Confirming it with the JIT log

```bash
# Restart the pod with compilation logging. Note the enormous volume; do this
# in the load environment, not in production, and to a file.
-XX:+UnlockDiagnosticVMOptions \
-Xlog:jit+compilation=debug:file=/var/log/orderflow/jit.log:time,uptime,tags \
-XX:+PrintCompilation
```

```bash
# Was the reconciler loop compiled, and at what tier, before the shutdown?
grep -i 'InventoryReconciler' /var/log/orderflow/jit.log

# The '%' marker in PrintCompilation output indicates on-stack replacement,
# which is how a loop that is ALREADY RUNNING gets compiled.
grep '%' /var/log/orderflow/jit.log | grep -i reconcil
```

**WHAT TO LOOK FOR:** a compilation entry for `InventoryReconciler::loop`, at tier 4 (C2)
or with the OSR marker, **timestamped before** the shutdown log line.

| What you see | What it means |
|---|---|
| `loop` compiled at tier 4, with an OSR marker, well before shutdown | **The mechanism is confirmed on your runtime.** The loop was running in C2-compiled code when the write landed. |
| `loop` compiled only at tier 3 (C1 with profiling) | The aggressive hoist is less likely, but the bug is still a bug. Under a longer-running pod it will reach tier 4. **Do not conclude you are safe.** |
| No compilation entry at all | The loop never got hot enough, which on a real deploy timeline is unlikely. Check your grep and your log file path. |
| Compilation entries with a `made not entrant` marker for `loop` | The method was deoptimised (Topic 74). Interesting, and worth reading — a deoptimisation would return the loop to the interpreter and could make the bug *stop* reproducing. That is exactly the kind of accident that makes this class of bug intermittent. |

### The fixes, in the order you should consider them

**Fix 1 — `volatile`, and understand that this is the *minimum*, not the answer.**

```java
private volatile boolean running = true;
```

Correct, one keyword, establishes edge 3. It is the fix that proves the diagnosis. It is
**not** where a production shutdown path should end, because it still busy-spins and
still cannot wake a parked thread.

**Fix 2 — interruption, which is the right shutdown mechanism.**

```java
private void loop() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            reconcileOnePass();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // restore the flag; Topic 84's Trap 3
            break;
        }
    }
    log.info("inventory reconciler loop exited");
}

@PreDestroy
public void stop() throws InterruptedException {
    worker.interrupt();                      // edge 7
    worker.join(5_000);                      // edge 5 — and a BOUND on the wait
    if (worker.isAlive()) {
        log.error("reconciler did not stop within 5s; investigate");
    }
}
```

Three improvements over Fix 1, all of them independently worth having:

- The edge comes from `interrupt()`, which is specified (edge 7).
- It **wakes a blocked thread**, which a flag cannot.
- `join(5_000)` means the shutdown path **verifies** the thread stopped, with a bound.
  Fix 1's `stop()` logged a lie; this one cannot.

**Fix 3 — do not own the thread at all.**

```java
private final ScheduledExecutorService scheduler =
    Executors.newSingleThreadScheduledExecutor(
        r -> Thread.ofPlatform().name("inventory-reconciler").daemon(true).unstarted(r));

@PostConstruct
public void start() {
    scheduler.scheduleWithFixedDelay(this::reconcileOnePass, 5, 5, TimeUnit.SECONDS);
}

@PreDestroy
public void stop() throws InterruptedException {
    scheduler.shutdown();                                   // no new tasks
    if (!scheduler.awaitTermination(5, TimeUnit.SECONDS)) { // bounded wait
        scheduler.shutdownNow();                            // interrupts running tasks
    }
}
```

**This is the fix to reach for in `orderflow`.** There is no flag, therefore no
visibility bug to have. `ExecutorService` submission and termination carry their own
happens-before guarantees (see the derived-edges table), the shutdown is bounded and
verified, and the "reconcile continuously" loop becomes "reconcile every five seconds",
which was almost certainly the requirement anyway. **The best fix for a memory-model bug
is frequently to delete the shared mutable state rather than to synchronise it.**

Topic 90 is the full treatment of executor shutdown; note the `shutdown` →
`awaitTermination` → `shutdownNow` sequence now.

**Fix 4 — while you are here, fix the Spring lifecycle too.**

`@PreDestroy` on a bean is not the same as a graceful HTTP shutdown. Set
`server.shutdown: graceful` and `spring.lifecycle.timeout-per-shutdown-phase: 20s`. That
is not a JMM concern, and it is the other half of why the pod took thirty seconds.
Fixing one without the other leaves you with half a slow deploy and a confusing result.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a non-`volatile` `boolean running` whose loop never exits

**Wrong:**

```java
private boolean running = true;                 // plain field

void loop()  { while (running) { work(); } }    // read on thread B
void stop()  { running = false; }               // write on thread A
```

**Exact symptom:**

- The thread does not stop. The process does not exit, or the container sits in
  `Terminating` for the full grace period and is `SIGKILL`ed.
- `jcmd <pid> Thread.print` shows the thread as **`RUNNABLE`**, not `BLOCKED`, not
  `WAITING`, holding no monitors and waiting on none.
- CPU for that thread stays pinned at roughly one core.
- The write **definitely happened**: the log line after it printed; a debugger inspecting
  the object shows `running == false`; an actuator endpoint reports `false`.
- **The program terminates correctly under `-Xint`.** This is the decisive observation.
- Whether it hangs varies between runs, between JDK builds, and with how long the loop
  ran before the write.

**Root cause:** there is no happens-before edge between the write in `stop()` and the
read in `loop()`. Given that licence, C2 performs loop-invariant code motion: it proves
nothing *inside the loop* writes `running`, hoists the read above the loop, and compiles
`while (register)` — or, once the register is a known constant, `while (true)`. **The
read is not stale. The read does not occur.** No cache flush, no barrier and no amount
of waiting can help, because there is no load instruction to make faster.

**Fix:** `volatile boolean running` establishes edge 3 and forbids the hoist — minimum
viable correctness. **Prefer `Thread.interrupt()`** (edge 7), which also wakes a blocked
thread, which a flag never can. **Better: own no thread** — an `ExecutorService` with
`shutdown` / `awaitTermination` / `shutdownNow` brings its own edges. **Verify the
shutdown** with `join(timeout)` or `awaitTermination(timeout)` and log an error if it did
not stop; the original code logged success without checking. And **prove the diagnosis
before fixing**, with the `-Xint` A/B below: "I added `volatile` and it stopped happening"
is weaker than you think, because so does adding a `println`.

**Architecture note:** this is a **compiler** effect. It reproduces the same way on x86
and on aarch64. Your Apple Silicon Mac has no advantage and no disadvantage here.

---

### Trap 2 — "it will see it eventually"

**Wrong:** the belief, usually unspoken, that a write propagates on its own within some
short unspecified time, so an unsynchronised read is a latency problem rather than a
correctness problem. It shows up as code like this:

```java
// "the reconciler will notice within a second or two, that's fine for a shutdown"
running = false;
Thread.sleep(2000);            // "give it time to notice"
```

or as a review comment: *"there's a tiny window here but the other thread will pick it
up on the next iteration."*

**Exact symptom:**

- Works in development, in unit tests, and in short-lived integration tests — because
  none of those run the loop long enough for it to be JIT-compiled.
- Fails under sustained load, in production, on long-lived instances. **The failure rate
  correlates with uptime**, which points investigations at memory leaks and connection
  pools instead.
- Increasing the sleep never fixes it. Someone tries 2 seconds, then 5, then 30. The
  behaviour does not improve, which should be the clue and usually is not.
- The team's working theory becomes "a race condition" in the vague sense, and the fix
  becomes "retry".

**Root cause:** **"eventually" is not in the specification.** JLS Chapter 17 defines
which writes a read is *permitted* to observe. It contains no clause requiring a write to
become visible within any bound, or at all. Implementations are free to never propagate
it — and a JIT that hoists the read out of the loop is precisely an implementation
exercising that freedom.

The intuition comes from somewhere reasonable: cache coherence *is* fast, store buffers
*do* drain in nanoseconds, and if the read were actually performed the value would
arrive almost immediately. **But the read is not performed.** The intuition models the
hardware and omits the compiler, and the compiler is the stronger of the two.

**Fix:** **replace the word "eventually" with a named edge**, or accept that there is no
guarantee — in a design discussion, "the reader will see it eventually" is not an
argument; "the reader sees it because there is a volatile write to the same field before
it" is. **Delete every `sleep` that exists to "give it time"**: a sleep is not a
synchronisation primitive, and if removing it changes correctness the code was wrong with
it too. **Establish the edge** (`volatile`, a lock, an interrupt, a `j.u.c` structure).
**Adopt the review phrasing**: for any field read by a thread that did not write it, ask
"which edge?" If nobody can name one from the ten-row table, it is a defect.

---

### Trap 3 — the accidental fix: adding a `println`, a `sleep`, or a metric makes it work

**Wrong:**

```java
while (running) {
    System.out.println("reconciling " + sku);   // added for debugging
    reconcileNextSku();
}
```

The hang disappears. The `println` is left in "because it's useful logging". Or:

```java
while (running) {
    reconcileNextSku();
    Thread.sleep(1);                            // "to be nice to the CPU"
}
```

The hang disappears. Or someone adds `meterRegistry.counter("reconciled").increment()`
and the hang disappears.

**Exact symptom:**

- The bug appears to be fixed, with a plausible-sounding explanation attached
  ("the sleep lets the other thread's write land").
- Months later, someone removes the debug logging in a cleanup PR, or swaps the counter
  for a `LongAdder`, or bumps the JDK — and the hang returns, **in a change that has
  nothing visibly to do with concurrency.**
- Nobody links the two commits. The bug is now "intermittent and unreproducible".

**Root cause — three distinct mechanisms, which is why the accident is so reliable:**

1. **`System.out.println` is synchronised.** `PrintStream`'s write methods synchronise on
   the stream: a monitor lock and unlock (edge 2) inside your loop, which forbids the
   hoist across it and supplies real barriers. **You accidentally synchronised.**
2. **`Thread.sleep` is a native call and a safepoint poll.** The loop is no longer a
   tight, call-free counted loop; the compiled code must reload state across the call.
   **You accidentally defeated the optimisation.**
3. **A metric increment usually touches an `Atomic*` or a `volatile`** — edge 3 in the
   middle of your loop, and transitivity (edge 10) carries the flag read along with it.

In all three cases the code is now *accidentally* correct, for a reason unrelated to the
author's intent and invisible at the call site. **This is strictly worse than the original
bug**, because the original bug was at least reliably present.

**Fix:** **never accept "adding a log line fixed it" as a diagnosis** — it is a symptom
of a memory-model bug, not a resolution of one, and it is a strong signal to go looking
for a missing edge. **Establish the edge deliberately**, then remove the accidental one
and confirm the fix still holds. **In review, treat any `sleep` or debug `println` inside
a loop whose exit condition is a shared field as load-bearing until proven otherwise**;
ask what happens when it is removed. And **write the test at the level that survives** —
a jcstress test (Topic 99), or at minimum an `-Xint`-versus-JIT A/B in a script, so the
property is checked rather than the incident being remembered.

---

### Trap 4 — synchronising the write but not the read (or vice versa)

**Wrong:**

```java
public class ReservationCounters {
    private final Map<String, Long> counters = new HashMap<>();

    // Writer: carefully synchronised.
    public synchronized void setBaseline(String sku, long units) {
        counters.put(sku, units);
    }

    // Reader: "it's just a read, reads are safe".
    public long baselineOf(String sku) {
        Long v = counters.get(sku);            // NOT synchronised
        return v == null ? 0L : v;
    }
}
```

**Exact symptom:**

- Readers observe stale values indefinitely, or observe `null` for a key that was
  definitely written.
- Occasionally a reader gets something worse than stale: a `HashMap` mid-resize can
  return a wrong value, or throw, or (on older JDKs) spin forever.
- Reviewers look at the class, see `synchronized`, and mark it thread-safe. The word
  appears in the file, so the box is ticked.
- The bug is much more visible on aarch64 than on x86, because the store-store and
  load-load reorderings that x86 forbids are permitted on your Mac.

**Root cause:** edge 2 says an unlock of a monitor happens-before a **subsequent lock of
that same monitor**. If the reader never locks, there is no subsequent lock, and
therefore no edge. **A happens-before edge requires participation from both sides.** One
thread synchronising is one thread taking a lock nobody else takes: mutual exclusion
against nobody, and an edge to nobody.

Additionally, `HashMap` is not merely un-published here; it is being *structurally
mutated* while another thread traverses it, which is a Topic 92 problem on top of the
Topic 86 one.

**Fix:** synchronise **both** sides on the same monitor; or use a `ConcurrentHashMap`,
whose per-key write happens-before a subsequent per-key read so the edge comes with the
data structure (Topic 92); or make the field `volatile` and publish an immutable
snapshot — `Map.copyOf(...)` assigned to a `volatile` field gives readers a consistent,
safely published view with no lock at all, which is the right shape for a read-mostly
registry like `HotSkuRegistry`. **Review rule:** the question is never "is this class
synchronised", it is **"is every access to this state ordered by an edge, on both
sides?"**

---

### Trap 5 — assuming a 64-bit `long` or `double` write is atomic

**Wrong:**

```java
public class WalletMetrics {
    private long totalDebitedMinor;                 // plain long

    public void record(long amountMinor) {
        totalDebitedMinor += amountMinor;           // read-modify-write, AND non-atomic write
    }

    public long total() { return totalDebitedMinor; }
}
```

**Exact symptom:**

- The reported total is wrong, and wrong in a way that arithmetic cannot explain — not
  just "missing some increments" but occasionally a value that is not the sum of any
  subset of the inputs.
- On 64-bit HotSpot you will very likely **never** observe the tearing part on your Mac.
  You will observe the lost-update part constantly.
- On a 32-bit VM, in an embedded JVM, or in some non-HotSpot implementations, the
  tearing is real.

**Root cause — two separate defects in one line, and you must be able to name both:**

1. **`+=` is a read-modify-write**, which is three operations with two gaps. Lost
   updates, exactly as in Topic 84's trace. This defect is present for `int` too, and
   `volatile` does **not** fix it (that is Topic 87's headline).
2. **JLS §17.7: reads and writes of `long` and `double` are not guaranteed atomic**
   unless the field is `volatile`. A 64-bit value may be written as two 32-bit halves,
   and a concurrent reader may observe one old half and one new half — a value **nobody
   ever wrote**. This is called **word tearing**.

The honest practical note, because the gap between spec and practice is exactly what
interviewers probe: **64-bit HotSpot implements `long` and `double` accesses atomically
in practice.** That is an implementation property of one JVM on one class of hardware. It
is not a specification guarantee, it has not always been true, and it is not true
everywhere. Relying on it is relying on an accident that currently holds.

**Fix:** for a **counter**, `LongAdder` (Topic 95) — striped, and the right tool when
writes dominate and reads are occasional. For atomic compound updates, `AtomicLong` with
`getAndAdd`/`compareAndSet`. If you only need **visibility of a value that is assigned,
never incremented**, `volatile long` is sufficient and removes the tearing question — but
it does **not** make `+=` atomic. **State both defects when you find one**; an engineer
who says only "that's not atomic" has seen half of it.

---

### Trap 6 — attributing the bug to the CPU when it was the compiler (or the reverse)

**Wrong:** the explanation, given confidently in a design review or an interview: *"the
other thread's CPU cache still had the old value, so we need `volatile` to flush the
cache."*

**Exact symptom:**

- The proposed fixes are hardware-shaped and wrong: "add a `sleep` so the cache
  synchronises", "pin the threads to the same core", "the cache line will be invalidated
  within microseconds so it's fine".
- The person is genuinely unable to explain why `-Xint` fixes it, because a cache
  explanation predicts `-Xint` would make no difference.
- Correct code sometimes results anyway, for the wrong reason, which entrenches the wrong
  model.

**Root cause:** two different mechanisms have been merged into one folk explanation.

| Mechanism | Where it lives | Does `-Xint` change it? | Architecture-dependent? |
|---|---|---|---|
| **The read is hoisted out of the loop** | The **JIT** (C2 LICM) | **Yes — decisively.** The interpreter re-reads on every `getfield` | **No.** Identical on x86 and aarch64 |
| **The read returns a stale value** | The **store buffer** and the memory system | No — the interpreter reads stale values too | **Yes.** aarch64 exposes more than x86-TSO |
| **Two writes become visible out of order** | The **CPU** (and the compiler) | Partially | **Yes** — Topics 87 and 88's territory |

`volatile` does not "flush the cache". Caches are coherent already; there is nothing to
flush. What `volatile` does is forbid the compiler from removing, hoisting or reordering
the access, **and** emit the barriers that force the store buffer to drain and the loads
to be ordered. A compiler constraint plus a hardware constraint, not a cache operation.

**Fix:**

1. **Use the three-row table to classify any visibility bug you meet.** The first
   question is always "does `-Xint` change it?" — that single experiment separates
   compiler from hardware in thirty seconds.
2. **Stop saying "flush the cache".** Say "establish a happens-before edge", and if you
   need the mechanism, say "forbid the hoist and emit the barrier".
3. In an interview, **volunteer** that the stop-flag bug is a compiler effect that
   reproduces identically on both architectures, while the Topic 87/88 reordering bugs
   are hardware effects that aarch64 exposes and x86 hides. That distinction is not
   common knowledge and it lands.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call
it captured. What follows is the exact command, what to look for, and how to read every
plausible result — including the ones that contradict the expected answer.

### Setup

```bash
mkdir -p ~/java-lab/86/{src/com/orderflow/lab/jmm,out,logs} && cd ~/java-lab/86
java --version                # record it; JIT behaviour is version-specific
uname -m                      # expect arm64 on Apple Silicon
sysctl -n hw.ncpu             # macOS core count, performance + efficiency
sysctl -n machdep.cpu.brand_string 2>/dev/null || true
```

Write all four down. Every result you record is a result *for that configuration*.

### Proof 1 — the `-Xint` A/B, which is the whole topic in two commands

```bash
javac -d out src/com/orderflow/lab/jmm/StopFlag.java

java -Xint -cp out com.orderflow.lab.jmm.StopFlag ; echo "interpreted exit=$?"
java       -cp out com.orderflow.lab.jmm.StopFlag ; echo "jit exit=$?"
```

**WHAT TO LOOK FOR:** two different exit codes from the same class file.

| What you see | What it means |
|---|---|
| `interpreted exit=0`, `jit exit=1` | **The demonstration.** Identical source, identical JVM, opposite behaviour. The only variable is whether the compiler was permitted to optimise. **That difference is the Java Memory Model.** |
| Both exit 0 | The hoist did not occur this run. Legitimate. Increase the pre-write sleep, try `-XX:-TieredCompilation`, and confirm with Proof 3 that the loop actually compiled. Then re-read Honesty rule 1. |
| Both exit 1 | Confirm `-Xint` took effect: `java -Xint -version` must say `interpreted mode`. If it does, and the interpreted run still hangs, the flag is not the only defect — look for something else holding the thread. |
| The JIT run hangs your terminal instead of exiting | You edited out the `join(10_000)` timeout. Put it back; you want a result, not a stuck shell. |

### Proof 2 — confirm the field read really is in the bytecode

`javap` shows what the *language* produced. The JIT's hoist happens later, so `javap`
will always show the read. That is the point: **the read is in the bytecode and is
absent from the machine code.**

```bash
javap -c -p -cp out com.orderflow.lab.jmm.StopFlag | sed -n '/lambda\$main/,/^$/p'
```

**WHAT TO LOOK FOR:** a `getstatic ... running` inside the loop body, with a
conditional branch back to it.

| What you see | What it means |
|---|---|
| `getstatic #x // Field running:Z` at the top of the loop, `ifeq`/`ifne` branching back | **Correct and expected.** The bytecode reads the field every iteration. Whatever eliminates that read is downstream of javac. |
| The read appears only once, before the loop | You are looking at a different method, or javac folded something. Re-check you dumped the lambda body, not `main`. |
| No `getstatic` at all | You made the field `final`, or you are reading the wrong class. `final` would let javac constant-fold it, which is a different (and legal) mechanism. |

**The lesson to write down:** `javap` proves the read exists in the bytecode; the hang
proves it does not exist in the executed machine code; the gap between those two is
`-Xint`'s A/B. Topic 76 taught you the first tool, Topic 74 the second.

### Proof 3 — catch the JIT compiling the loop

```bash
java -XX:+PrintCompilation -cp out com.orderflow.lab.jmm.StopFlag 2>&1 \
  | tee logs/compile.log | grep -i 'StopFlag'
```

**WHAT TO LOOK FOR:** a line for the lambda or the loop method with a **`%`** marker
(on-stack replacement) and a tier number, appearing **before** `main: writing
running=false`.

| What you see | What it means |
|---|---|
| A `%` line at tier 4 for the loop, before the write | **On-stack replacement into C2 happened while the loop was running.** This is step 4 of the concurrency trace, observed. |
| A `%` line at tier 3 only | Compiled by C1 with profiling. The bug may or may not manifest. Try `-XX:-TieredCompilation` to force C2 and re-run. |
| No line for the loop at all | The loop never got hot. Raise the sleep, or lower the OSR threshold with `-XX:Tier4BackEdgeThreshold` (diagnostic; confirm it exists on your JDK with `-XX:+PrintFlagsFinal`). |
| A `made not entrant` line for the loop method | It was deoptimised (Topic 74). If the hang stops after that point, you have watched the bug switch itself off — which is an excellent illustration of why this class of bug is intermittent. |

### Proof 4 — show that the accidental fixes are accidental

Make three copies of `StopFlag` with the loop body changed, and nothing else:

```java
// (a) accidental monitor: PrintStream.println synchronises on the stream
while (running) { local++; if (local % 100_000_000 == 0) System.out.println("."); }

// (b) accidental safepoint + native call
while (running) { local++; if (local % 100_000_000 == 0) Thread.onSpinWait(); }

// (c) accidental volatile: an AtomicLong increment inside the loop
while (running) { local++; counter.incrementAndGet(); }
```

Run each under the default JIT.

**WHAT TO LOOK FOR:** whether the hang disappears, and — crucially — whether you can
predict which ones will before you run them.

| What you see | What it means |
|---|---|
| (a) terminates | `println` took a monitor (edge 2) inside the loop. **You synchronised by accident.** This is Trap 3's mechanism 1, demonstrated. |
| (c) terminates | The atomic operation is edge 3 inside the loop, and transitivity carries the flag read. Trap 3's mechanism 3. |
| (b) terminates | The hint/call disrupted the loop's optimisation. Trap 3's mechanism 2. Note that `onSpinWait` is a *hint*, not a barrier — if this one terminates, it is the weakest of the three accidents and the most fragile. |
| One or more still hang | Also informative. It means that particular accident was not enough on your JDK. **The accidents are not reliable in either direction**, which is exactly the argument against relying on them. |
| You predicted the results correctly beforehand | You have the model. That is the actual pass condition for this proof. |

### Proof 5 — the same program, made correct, five ways

Build five variants of `StopFlag`, changing only the synchronisation, and run each under
the default JIT **and** under `-XX:-TieredCompilation`: (1) `volatile boolean`,
(2) `AtomicBoolean`, (3) `Thread.interrupt()` plus `isInterrupted()`, (4) `synchronized`
accessors on **both** sides, (5) a `VarHandle` with `setRelease`/`getAcquire`.

**WHAT TO LOOK FOR:** all five exit `0`, every time, in both configurations.

| What you see | What it means |
|---|---|
| All five exit 0 consistently | Each establishes a real edge. Now say **which numbered edge** each one uses — that is the exercise, not the exit code. (Answers: 3, 3, 7, 2, 3-equivalent.) |
| Case 4 hangs | You synchronised only one side. Trap 4. Both the read and the write must take the **same** monitor. |
| Case 5 hangs | Check the access mode. `getPlain`/`setPlain` establish nothing; you need `getAcquire`/`setRelease` or `getVolatile`/`setVolatile`. |
| Any case is flaky | Stop and find the reason. A fix that works four times in five is not a fix, and finding out why is worth more than the drill. |

### Proof 6 — reading the actual machine code, honestly

The ground truth is the disassembly, and it is the *expensive* path. The command and its
three honest caveats are in the Measurement section below; read them before you try it.
The short version: `-XX:+PrintAssembly` needs the `hsdis` plugin, which is **not bundled
with most JDKs**, it emits tens of thousands of lines, and what you are hunting is the
**absence** of an `ldr` inside the loop, which is harder to read than a presence. The
`-Xint` A/B answers the same question in two seconds with no tooling.

A cheaper middle option that needs no `hsdis`:

```bash
# Was it compiled, and what got inlined into it?
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining \
  -cp out com.orderflow.lab.jmm.StopFlag 2>&1 | grep -i stopflag
```

### Proof 7 — the thread dump fingerprint

```bash
# Terminal 1
java -cp out com.orderflow.lab.jmm.StopFlag

# Terminal 2, while it is hung
jcmd $(pgrep -f StopFlag) Thread.print > logs/dump1.txt
sleep 2
jcmd $(pgrep -f StopFlag) Thread.print > logs/dump2.txt

grep -A8 'flag-worker' logs/dump1.txt logs/dump2.txt
grep -c 'BLOCKED' logs/dump1.txt
grep -c 'waiting to lock' logs/dump1.txt
```

**WHAT TO LOOK FOR:** `flag-worker` in state `RUNNABLE` in both dumps, and **zero**
`BLOCKED` threads and **zero** `waiting to lock` lines.

| What you see | What it means |
|---|---|
| `RUNNABLE`, same stack in both dumps, no lock lines anywhere | **The fingerprint of a visibility bug.** Memorise this shape. Every other hang in Phase 9 looks different. |
| Any `BLOCKED` thread or `waiting to lock` line | A different problem. Topic 85 contention or Topic 94 deadlock. |
| The thread is missing from the dump | It exited between the hang and the dump. Re-run. |
| `jcmd` itself hangs | Unrelated and interesting: taking a thread dump requires a global safepoint (Topic 73), and a compiled counted loop with no poll may delay it. **Two topics colliding in one command.** |

---

## Failure drill

**Mandatory.** Do not read the "how to read it" tables until you have produced the result
yourself and written down what you saw.

### The assignment, restated from the master plan

> The classic non-`volatile` stop flag. Write it, run with `-Xint` (terminates) and then
> normally with a warmed loop (hangs). **That gap IS the memory model.**
> Capture: hang under JIT, termination under `-Xint`.

### Two honesty rules that govern this drill, and Topics 87 and 88

> **Honesty rule 1 — a non-observation is not a proof.** If the JIT run terminates, you
> have **not** shown the code is correct. You have shown that on this machine, this JDK,
> this JIT state and this run, the permitted-but-unwanted outcome did not occur.
> Correctness is proved by the happens-before argument. Experiments falsify a wrong
> argument; they never confirm a right one. **I will never tell you this drill "will"
> hang.**

> **Honesty rule 2 — architecture, and which way it cuts here.** x86 implements Total
> Store Order and hides a whole class of reordering bugs that aarch64 (your Apple
> Silicon Mac) exposes. **For Topics 87 and 88 that makes your Mac a better instrument
> than x86 CI.** **For THIS drill it makes no difference at all**, because the mechanism
> is the JIT hoisting a read out of a loop, and that is a compiler decision taken before
> any instruction is issued. If a colleague reports that this drill behaves differently
> on their Intel machine, the difference is JIT state or timing, not architecture.

### Step 0 — the control

```bash
java --version
uname -m
java -Xint -version                 # must print "interpreted mode"
java -XX:+PrintFlagsFinal -version | grep -iE "TieredCompilation|TieredStopAtLevel|UseOnStackReplacement"
```

Record all of it. Then run the `orderflow` Topic 65 baseline unchanged and confirm the
percentiles are within ±10% of `/docs/java/baselines/`. **If they are not, stop.** The
Topic 65 gate rule applies to every drill in this phase.

### Step 1 — standalone, before touching `orderflow`

Run `StopFlag` from Example 1 in all five configurations (A through E). Record the exit
code of each. **Do this first**, so you know the mechanism reproduces on your JDK before
you introduce it into a system with fifty other variables.

If configuration B (default JIT) does not hang: raise the pre-write sleep from 1000 ms to
3000 ms; add `-XX:-TieredCompilation` to force straight to C2; confirm with
`-XX:+PrintCompilation` that the loop method was compiled with an OSR marker before the
write; and make the loop body emptier by removing anything that could be a call.

If it still does not hang after all four, **that is a legitimate and recordable result.**
Write down "on JDK <version>, aarch64 macOS, this shape did not reproduce", and proceed
to Step 2 anyway: the `orderflow` version has more threads, more compilation pressure and
a much longer-lived loop, and often shows it when the standalone version does not.

### Step 2 — put the defect into `orderflow`

```java
package com.orderflow.lab;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.util.concurrent.atomic.AtomicLong;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

/**
 * DRILL CODE. Never merge this.
 *
 * The Topic 86 defect in the shape it really ships: a background reconciler with a
 * plain boolean stop flag, started in @PostConstruct and stopped in @PreDestroy.
 *
 * Guarded by a Spring profile so it cannot possibly reach production.
 */
@Component
@Profile("drill86")
public class StopFlagDrill {

    private static final Logger log = LoggerFactory.getLogger(StopFlagDrill.class);

    /** THE DEFECT. */
    private boolean running = true;

    private final AtomicLong passes = new AtomicLong();
    private Thread worker;

    @PostConstruct
    public void start() {
        worker = new Thread(this::loop, "drill86-reconciler");
        worker.setDaemon(true);
        worker.start();
        log.info("drill86 reconciler started");
    }

    private void loop() {
        long local = 0;
        while (running) {                      // THE READ
            local++;
            if ((local & 0xFFFFFFFL) == 0) {   // ~268M iterations between publishes
                passes.set(local);             // outside the hot path; see the note
            }
        }
        log.info("drill86 reconciler loop EXITED after {} iterations", local);
    }

    @PreDestroy
    public void stop() throws InterruptedException {
        log.info("drill86: writing running=false");
        running = false;                       // THE WRITE
        log.info("drill86: write complete; joining with 10s timeout");
        worker.join(10_000);
        log.error("drill86: worker alive after join? {}", worker.isAlive());
    }
}
```

> **Note the confound you must control.** The `passes.set(local)` is an
> `AtomicLong` write — edge 3 — inside the loop. It is behind a mask so it executes
> roughly once every 268 million iterations, which keeps it out of the hot path, **but
> it is still an edge and it may be enough to defeat the hoist on your JDK.** Run the
> drill **twice**: once as written, and once with that line deleted entirely. If
> deleting it changes the outcome, you have just reproduced Trap 3 inside your own drill
> code, which is a better lesson than the drill itself. Record both.

### Step 3 — run it under load, and shut it down

```bash
# 1. Start orderflow with the drill profile and compilation logging.
java -Xms1200m -Xmx1200m -XX:+UseG1GC \
  -Dspring.profiles.active=load,drill86 \
  -XX:+PrintCompilation \
  -Xlog:jit+compilation=debug:file=/var/log/orderflow/jit-drill.log:time,uptime,tags \
  -jar orderflow.jar 2>&1 | tee logs/stdout-drill86.log

# 2. Let it warm up under the UNCHANGED Topic 65 load profile for at least 3 minutes,
#    so the reconciler loop is definitely compiled.
k6 run --duration 5m load/baseline.js

# 3. While the load is still running, take a thread dump.
jcmd $(pgrep -f orderflow) Thread.print > logs/dump-during.txt

# 4. Now shut the service down the way Kubernetes would.
kill -TERM $(pgrep -f orderflow)

# 5. Immediately take another dump, and keep taking them.
for i in 1 2 3 4 5; do
  jcmd $(pgrep -f orderflow) Thread.print > logs/dump-shutdown-$i.txt 2>/dev/null || echo "process gone at $i"
  sleep 2
done
```

### Step 4 — what to capture

Write these down **before** reading any interpretation:

1. Did the process exit after `SIGTERM`, and how long did it take?
2. Did `drill86: write complete` appear in the log?
3. Did `drill86 reconciler loop EXITED` appear? **This is the decisive line.**
4. What did `drill86: worker alive after join?` report?
5. In `dump-shutdown-1.txt`, what is `drill86-reconciler`'s thread **state**?
6. How many `BLOCKED` threads and `waiting to lock` lines are in that dump?
7. In `jit-drill.log`, was `StopFlagDrill::loop` compiled, at what tier, with or without
   an OSR marker, and **before or after** the write?
8. CPU usage of the process between `SIGTERM` and exit.

```bash
grep -E 'drill86' logs/stdout-drill86.log
grep -A10 'drill86-reconciler' logs/dump-shutdown-1.txt
grep -c 'BLOCKED' logs/dump-shutdown-1.txt
grep -c 'waiting to lock' logs/dump-shutdown-1.txt
grep -i 'StopFlagDrill' /var/log/orderflow/jit-drill.log
```

### Step 5 — the A/B control, which is the actual proof

Everything above shows a hang. **A hang alone proves nothing** — plenty of things hang.
The proof is the control:

```bash
# Identical jar, identical load, identical shutdown. ONE flag different.
java -Xint -Xms1200m -Xmx1200m \
  -Dspring.profiles.active=load,drill86 \
  -jar orderflow.jar 2>&1 | tee logs/stdout-drill86-xint.log

# Warm up (much more slowly - the whole service is interpreted), then:
kill -TERM $(pgrep -f orderflow)
grep -E 'drill86' logs/stdout-drill86-xint.log
```

> **Expect `-Xint` to make the whole service dramatically slower.** The interpreter runs
> everything, not just your loop. Do **not** compare percentiles between the two runs;
> the only thing being compared is **whether the loop exited**. Use a much lighter load
> or none at all for the `-Xint` run — the loop only needs the write to arrive.

### Step 6 — how to read it

| What you see | What it means |
|---|---|
| JIT run: no `loop EXITED` line, worker alive after join, process hangs until you kill it. `-Xint` run: `loop EXITED` appears, clean shutdown | **The drill has fired.** Write the sentence: *"the same jar, the same load, the same shutdown signal; with the JIT the loop never observed the write, without the JIT it observed it immediately. The difference is not timing, it is the compiler's licence, and that licence is the Java Memory Model."* |
| Thread dump shows `drill86-reconciler` as `RUNNABLE` with zero `BLOCKED` threads and zero `waiting to lock` lines | **The fingerprint.** This is what separates a visibility hang from a lock hang at a glance, in production, at 3am. |
| `jit-drill.log` shows `loop` compiled at tier 4 with an OSR marker before the write | The mechanism is confirmed on your runtime. Step 4 of the concurrency trace, observed directly. |
| Both runs exit cleanly | The hoist did not occur. **Legitimate.** Try deleting the `passes.set(local)` line (see the confound note), force `-XX:-TieredCompilation`, and warm up longer. Then record the honest result: on this JDK and this shape, it did not reproduce. **That is a finding, and Honesty rule 1 says it is not an acquittal.** |
| Both runs hang | `-Xint` did not take effect, or something else is holding the JVM open. Check for a non-daemon thread: `grep -c 'daemon' dump-shutdown-1.txt` versus the total thread count. A single non-daemon thread keeps the JVM alive and would explain both. |
| The process exits promptly but `loop EXITED` never printed | The worker is a **daemon** thread, so the JVM killed it at exit rather than it exiting. **This is the sneaky case**: the bug is fully present and the symptom is masked, until someone marks the thread non-daemon or adds real shutdown work. Note it and move on to the fix. |
| It reproduces on some runs and not others | **The single most valuable observation available.** Run it ten times and record the ratio. Then explain to yourself why a test suite would never catch this. |
| Deleting `passes.set(local)` changes the result | You reproduced Trap 3 in your own drill harness. An atomic write inside the loop was supplying an accidental edge. Excellent. |

### Step 7 — fix it, three ways, and re-measure each

Apply in order, re-running Steps 3 and 5 after each. **Fix A — `volatile boolean
running`**: expect `loop EXITED` in **both** the JIT and `-Xint` runs, on every
repetition, and confirm it holds under `-XX:-TieredCompilation`. **Fix B —
interruption**: replace the flag with `Thread.interrupt()` plus `isInterrupted()` and
`join(5_000)` in `stop()`; confirm the shutdown is not just correct but **bounded and
verified**, which the original `stop()` never was. **Fix C — no owned thread**: replace
the class with a `ScheduledExecutorService` and `shutdown` → `awaitTermination` →
`shutdownNow`, then re-run the full Topic 65 baseline and confirm p50/p95/p99 are back
within ±10%.

Produce a table comparing: shutdown duration after `SIGTERM`, whether `loop EXITED`
appeared, thread state in the dump, lines of code changed, and whether the fix survives
`-XX:-TieredCompilation`.

### What the drill proves

Carry three sentences out of it:

> *The write happened. The reader never read. Those are different failures, and only one
> of them is fixable by waiting.*

> *`-Xint` versus the JIT, on one jar, is the cheapest A/B in the JVM, and it separates
> "the compiler optimised my read away" from every other hypothesis in two seconds.*

> *A `RUNNABLE` thread that will not stop, holding no locks, is a visibility bug until
> proven otherwise.*

---

## Measurement

### The instrument for this topic

**The `-Xint` A/B is the instrument.** That is unusual — most topics in this curriculum
are measured with a profiler or a log — and it is worth saying explicitly why.

This topic's question is **binary and causal**, not quantitative: *did the reader observe
the write?* There is no percentile to compute, no throughput to compare, no allocation
rate to trend. The measurement is: run the identical artefact with the JIT enabled and
with it disabled, and see whether the observable outcome changes.

| Instrument | What it answers here | Verdict |
|---|---|---|
| **`-Xint` vs default** | Is the compiler the cause? | **The primary instrument.** Two seconds, no tooling, decisive |
| `-XX:TieredStopAtLevel=1` vs `-XX:-TieredCompilation` | Is it C1 or C2? | Secondary. Narrows *which* compiler |
| `-XX:-UseOnStackReplacement` | Was the running loop compiled in flight? | Secondary, and often the clearest of the three |
| `-XX:+PrintCompilation` / `-Xlog:jit+compilation` | Was the method compiled, when, at what tier, with OSR? | **Corroboration.** Turns "it hung" into "it hung after this compilation event" |
| `jcmd <pid> Thread.print` | Is the thread `RUNNABLE` and lock-free? | **The production diagnostic.** The one you use when you cannot restart with flags |
| `-XX:+PrintAssembly` | Is there a load instruction inside the loop? | Ground truth, and **needs `hsdis`, which is not bundled.** Heavy. Know it exists |
| jcstress | Which outcomes are reachable? | **Topics 87, 88 and 99.** For this drill it is the wrong shape — the bug is a hang, not an outcome tuple, and jcstress handles hangs by timing out a test rather than reporting a result |
| A `System.nanoTime()` loop | Nothing useful. See below | **Wrong tool.** Always |

### The standing rule: a naive `System.nanoTime()` loop is WRONG

You will be tempted to write this at some point in this phase:

```java
// DO NOT DO THIS. It cannot measure what you think it measures.
long t0 = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    volatileFlag = true;                     // "how expensive is a volatile write?"
}
System.out.println((System.nanoTime() - t0) / 1_000_000 + " ms");
```

Four reasons the technique lies, all of which apply to every measurement in Phase 9.
**Dead-code elimination**: if the result is not observed, C2 may delete the loop entirely
and you measure the cost of nothing (Topic 75). **Constant folding and hoisting**: a
loop-invariant store of the same value can be sunk out of the loop and executed once.
**On-stack replacement**: the loop starts interpreted and is compiled mid-flight, so your
number blends interpreter, C1 and C2 in proportions set by the iteration count you
happened to pick (Topic 74). **Cold JIT**: the first iterations are interpreted, which is
exactly the state in which the behaviour you care about does not occur.

And one reason specific to concurrency, which is the decisive one: **a single-threaded
timing loop cannot produce the phenomenon at all.** Visibility, reordering and contention
are *multi-thread* properties. A benchmark with one thread measures a world in which the
bug cannot exist, and you would conclude the problem does not exist.

**Topic 77 is the full treatment of JMH.** Do not write a benchmark you intend to act on
until you have read it. For this phase specifically, the correct shapes are JMH with
`@Threads` (Topics 84, 85, 95) and jcstress (Topics 87, 88, 99) — and for *this* topic,
neither, because the question is causal rather than quantitative.

### `PrintAssembly`, stated honestly

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,com.orderflow.lab.jmm.StopFlag::* \
     -cp out com.orderflow.lab.jmm.StopFlag
```

**Available, and heavy.** It requires the `hsdis` disassembler plugin, which is not
shipped with most JDK builds; without it the JVM prints a "could not load hsdis" message
and no disassembly. Obtaining or building `hsdis` for aarch64 macOS is a real piece of
work. The output is tens of thousands of lines. And the thing you are looking for is an
*absence* — no `ldr` of the flag field inside the loop — which is harder to read than a
presence.

**It is the ground truth and you should know it exists.** It is not the tool for this
drill. `-Xint` answers the same question in two seconds.

### What to record in production, permanently

| Signal | Where from | Why |
|---|---|---|
| Shutdown duration per pod | Deployment logs, `kubectl` events | A pod that consistently takes the full grace period is a stuck thread until proven otherwise |
| Count of pods reaching `SIGKILL` | Kubernetes events | Should be zero. A non-zero steady state is a shutdown bug somewhere |
| A "loop exited" log line for **every** long-lived loop you own | Application logs | **The single cheapest defence against this entire bug class** |
| Thread count over time | JFR, `jcmd Thread.print` | A thread that never exits is also a thread leak (Topic 98) |
| CPU on terminating pods | Container metrics | High CPU during termination is the visible signature of a spinning loop |

**The third row is the one to actually implement.** A `log.info("... loop exited")` at
the bottom of every worker loop, plus an assertion in the shutdown path that it was
reached, converts a silent four-month bug into an obvious one on the first deploy.

---

## Practice exercises

### 1 — easy: build the happens-before fact sheet, then use it

**Part A.** Write out the ten-edge table from memory. Then check it against the Quick
reference card. For each edge you missed, write one sentence on what would break without
it.

**Part B.** For each of these snippets, state whether there is a happens-before edge
between the write and the read, **name the numbered edge if there is one**, and say what
the reader is permitted to observe:

```java
// (1)
int stock;
Thread t = new Thread(() -> System.out.println(stock));
stock = 5;
t.start();

// (2)
int stock;
stock = 5;
Thread t = new Thread(() -> System.out.println(stock));
t.start();

// (3)
int stock;
Thread t = new Thread(() -> stock = 5);
t.start();
System.out.println(stock);

// (4)
int stock;
Thread t = new Thread(() -> stock = 5);
t.start();
t.join();
System.out.println(stock);

// (5)
final Object lock = new Object();
int stock;
// thread A:  synchronized (lock) { stock = 5; }
// thread B:  synchronized (lock) { System.out.println(stock); }

// (6)
final Object lockA = new Object(), lockB = new Object();
int stock;
// thread A:  synchronized (lockA) { stock = 5; }
// thread B:  synchronized (lockB) { System.out.println(stock); }

// (7)
volatile boolean ready;
int stock;
// thread A:  stock = 5; ready = true;
// thread B:  if (ready) { System.out.println(stock); }
```

Snippets (1) and (2) differ by two lines and have different answers. Snippet (7) is the
most important one on the list — it is the shape Topic 87 is built on, and the answer
depends on transitivity. Snippet (6) is the one people get wrong.

**Part C.** Answer in one sentence each: why is `-Xint` a *control* rather than a *fix*?
Your colleague says "we added a log line and the hang went away, so it's fixed" — give
the two-sentence reply. A field is read by five threads and written by one: which
threads' code needs to change to establish an edge?

### 2 — medium: the audit (combines Topics 01–85)

This class is in the `orderflow` codebase. Find **seven** defects. Three are memory-model
defects from this topic; four are from earlier topics. For each: name the topic, state
the **observable** symptom (a log line, a thread state, a percentile, a wrong number),
and write the fix.

```java
package com.orderflow.inventory;

import java.util.*;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
public class HotSkuCache {

    private static final Map<String, Integer> RESERVED = new HashMap<>();

    private boolean refreshing = true;
    private long totalRefreshes;
    private Long lastRefreshMillis = 0L;
    private Thread refresher;

    private final InventoryRepository repository;

    public HotSkuCache(InventoryRepository repository) {
        this.repository = repository;
        this.refresher = new Thread(this::refreshLoop, "hot-sku-refresher");
        this.refresher.start();                       // started IN the constructor
    }

    @PostConstruct
    public void warmUp() {
        for (String sku : repository.hotSkus()) {
            RESERVED.put(sku, repository.availableUnits(sku));
        }
    }

    private void refreshLoop() {
        while (refreshing) {
            for (String sku : repository.hotSkus()) {
                RESERVED.put(sku, repository.availableUnits(sku));
            }
            totalRefreshes++;
            lastRefreshMillis = System.currentTimeMillis();
        }
    }

    public int reservedFor(String sku) {
        Integer v = RESERVED.get(sku);
        return v == null ? 0 : v;
    }

    public boolean isStale(String sku) {
        return lastRefreshMillis < System.currentTimeMillis() - 60_000;
    }

    @PreDestroy
    public void shutdown() {
        refreshing = false;
    }
}
```

Hints, in the order to think about them: one is this topic's stop flag; one is a
non-atomic 64-bit field read across threads; one is a plain `long` counter incremented
from one thread and read from another with no edge; one is a Topic 84 / Topic 88 problem
about starting a thread inside a constructor; one is a Topic 92 problem about a `HashMap`
mutated while other threads read it; one is a Topic 01 boxing problem that is also a
correctness problem (look hard at `isStale`); and one is a Topic 79 unbounded-static
retention shape.

(Yes, that is more than seven. Find them all, then **rank them by how long they would
survive in production before anyone noticed** — the ranking is the exercise, and the
answer is not the same as ranking by severity.)

### 3 — hard: production simulation under load

**Part A — reproduce and record.** Run the Topic 65 baseline with the `drill86` profile
active but the reconciler's flag `volatile` (i.e. correct). Confirm percentiles within
±10%. Record shutdown duration after `SIGTERM`, and confirm `loop EXITED` appears. Three
runs; state your noise floor.

**Part B — introduce the defect.** Remove `volatile`. Re-run identical load. Record:
shutdown duration, whether `loop EXITED` appeared, the reconciler's thread state in a
dump taken 5 seconds after `SIGTERM`, and CPU during termination. Run it **ten times** and
record how many of the ten hung.

**Part C — the control.** Re-run Part B with `-Xint`, with a much lighter load. Record
whether `loop EXITED` appeared. **This pair — B and C — is the deliverable.** Write the
one-paragraph explanation you would give a colleague who says "but the write definitely
happened, I can see it in the log".

**Part D — the accidental fixes.** Starting from the broken version, apply each of these
*one at a time* and record whether the hang disappears:

1. Add `log.debug("pass {}", local)` inside the loop.
2. Add `Thread.sleep(1)` inside the loop.
3. Add a Micrometer counter increment inside the loop.
4. Add `if (local % 1000 == 0) Thread.yield();` inside the loop.

For each one that "fixes" it, **name the mechanism** (which edge, or which optimisation
was defeated) and write the sentence you would put in a PR review to reject it.

**Part E — the blind test.** This is the exercise that actually teaches the topic.
Produce five artefacts taken from (i) the visibility hang, (ii) a Topic 85
lock-contention stall, (iii) a Topic 94 two-lock deadlock, (iv) a Topic 73
time-to-safepoint stall, and (v) a healthy shutdown. **Strip the labels.** Hand them to a
colleague, or to yourself in a week. For each, write down the interesting thread's state,
whether anything is `BLOCKED` or `waiting to lock`, your one-sentence diagnosis, and the
single next command you would run. Then check against your notes. **Any artefact you
cannot classify from the dump alone is a gap in your reading skill, not your JMM
knowledge**, and it is the more expensive gap of the two.

**Part F — argue against yourself.** You will conclude that `volatile` is the fix. Make
the strongest possible case that `volatile` is the **wrong** fix for this specific
`orderflow` component, and name what should replace it. Then state the circumstance in
which `volatile` really is the right answer, and be specific about what makes that
circumstance different.

---

## Interview questions

### Q1 — "Why does this loop never terminate?"

*(The interviewer writes the non-`volatile` stop flag on the whiteboard.)*

**MID-LEVEL answer:** "The `running` field isn't `volatile`, so the other thread might
be reading a cached copy. Making it `volatile` forces it to read from main memory and
the loop will exit."

**SENIOR answer:** "It never terminates because there is **no happens-before edge**
between the write in `stop()` and the read in the loop, and that missing edge gives the
JIT a licence I can name precisely.

**The dominant mechanism here is the compiler, not the cache.** C2 performs
loop-invariant code motion. It analyses the loop body, proves that nothing *inside* the
loop writes `running`, and — because the absence of an edge means it is entitled to
assume no other thread does either — hoists the read out of the loop. What executes is
`while (register)`, or once the value is a known constant, `while (true)`. **The read is
not stale. The read does not happen.** So 'flush the cache' is not a coherent proposal,
because there is no load instruction to be served by any cache.

**I'd prove that in two commands rather than assert it.** Run the same jar with `-Xint`.
It terminates, because the interpreter re-reads the field on every `getfield` bytecode.
Run it with the JIT and it hangs. Same source, same JVM, same machine, opposite outcome,
one variable. **That gap is the memory model, demonstrated.** I'd corroborate with
`-XX:+PrintCompilation` and look for the loop method compiled at tier 4 with an OSR
marker, timestamped before the write.

**And there is a second, weaker mechanism that also needs the edge**, which matters for
the next question: even interpreted, the write can sit in the writing core's store buffer
and the reader can legitimately read a stale value from a perfectly coherent cache. That
one is architecture-sensitive — aarch64 exposes more than x86-TSO — whereas the hoist is
identical on both. Two mechanisms, one missing edge, one fix.

**The fix I'd actually ship is not `volatile`.** `volatile` is correct and it is the
minimum. For a shutdown path I want `Thread.interrupt()`, because interruption is also a
happens-before edge and it additionally *wakes a thread that is parked* — a volatile flag
cannot wake a thread blocked in `queue.take()`. And better still, I would not own the
thread: a `ScheduledExecutorService` with `shutdown` / `awaitTermination` /
`shutdownNow` brings its own edges and gives me a bounded, *verified* shutdown. The
original `stop()` logged 'stopped' without ever checking, which is how this survived four
months.

**Diagnostically, in production**: the fingerprint is a thread that is `RUNNABLE`, not
`BLOCKED`, holding no monitors and waiting for none, with a loop condition that is
demonstrably false. Every other hang shape in this area is `BLOCKED` or `WAITING`. That
one glance at a thread dump narrows it enormously."

**What separates them:** naming the compiler rather than the cache as the dominant
mechanism; knowing LICM by name and why the absence of an edge licences it; offering the
`-Xint` A/B as a two-second falsifiable experiment; distinguishing the compiler mechanism
(architecture-independent) from the hardware mechanism (architecture-sensitive);
preferring interruption and executors over `volatile` for a *shutdown* specifically; and
giving the thread-dump fingerprint, which is what actually gets used at 3am.

**Interviewer's follow-up:** *"Someone adds a log line inside the loop and it stops
hanging. Is it fixed?"* — No, it is worse. `PrintStream.println` synchronises on the
stream, so there is now a monitor lock and unlock inside the loop: an accidental
happens-before edge that also defeats the hoist. The code is correct for a reason
invisible at the call site and unrelated to the author's intent. It will break when
someone removes the logging in a cleanup PR, and nobody will connect the two commits. I
would establish the edge deliberately, then remove the log line and confirm the fix still
holds.

---

### Q2 — "What does the Java Memory Model actually guarantee?"

**MID-LEVEL answer:** "It defines how threads see changes to shared variables. It says
you need `volatile` or `synchronized` for changes to be visible between threads,
otherwise a thread might see a stale value."

**SENIOR answer:** "I'd state it as a permission model rather than a visibility promise,
because the difference is where all the surprises live.

**The JMM defines a partial order called happens-before, and its central rule is about
what a read is PERMITTED to return.** If a write W and a read R of the same variable are
not ordered by happens-before, R is permitted to return the value from W, or the value
from any other write to that variable by any thread, **including the default value
written at initialisation.** The specification places no bound on how long that
continues. **'Eventually' does not appear anywhere in Chapter 17.** That is the single
most commonly-held wrong belief about the JMM, and it is why people write fixes involving
sleeps.

**The edges are a short, closed list**: program order within one thread; monitor unlock
to a subsequent lock of the same monitor; volatile write to a subsequent read of the same
field; `Thread.start` to everything in the started thread; everything in a thread to a
successful `join`; thread termination detection; `interrupt` to its detection;
final-field freeze at constructor end; the default-value write; and transitivity.

**Transitivity is the one that does the real work**, and it is the part people miss. A
lock release does not just publish the variables named in the block — it publishes
*everything the thread did before it*, because those writes happen-before the unlock by
program order, the unlock happens-before the next lock, and the next lock happens-before
the reader's subsequent reads. That is why a single `volatile` write at the end of an
initialisation sequence safely publishes an entire object graph, and it is the mechanism
behind safe publication in Topic 88.

**And the guarantee I'd actually lead with, because it is the useful one:** a program
with **no data races is sequentially consistent**. If every shared access is ordered by
an edge, you may reason about the program as a simple interleaving of thread actions and
forget everything about barriers and store buffers. That is the payoff. All of this
machinery exists so that correctly-synchronised code can be reasoned about simply, and
the price is that incorrectly-synchronised code has almost no guarantees at all.

**A data race also has a precise technical definition** worth stating: two accesses to
the same variable, at least one a write, from different threads, **not ordered by
happens-before**. Note what is absent from that definition — timing. Two accesses a full
second apart race if there is no edge. 'They can't collide, they're seconds apart' is not
an argument."

**What separates them:** framing it as permission rather than eventual visibility;
knowing "eventually" is not in the spec and naming that as the common error; reciting the
edges as a closed list; identifying transitivity as the load-bearing rule and connecting
it to safe publication; leading with the data-race-free sequential-consistency guarantee
as the *point* of the model; and giving the technical definition of a data race including
the absence of timing from it.

**Interviewer's follow-up:** *"Where in the JDK do you actually rely on this?"* —
Everywhere in `java.util.concurrent`, and it is documented. `BlockingQueue.put`
happens-before the matching `take`, so a producer can build an object with plain fields
and the consumer sees it fully formed. `CountDownLatch.countDown` happens-before a
returning `await`, which is how initialisation handoff works. `ConcurrentHashMap`'s
per-key write happens-before a subsequent per-key read. `Future.get` returns everything
the task did. Those guarantees are why using `j.u.c` correctly means you rarely think
about barriers — the edges are already placed, and they are placed using exactly the ten
rules above.

---

### Q3 — "Is assignment to a `long` atomic in Java?"

**MID-LEVEL answer:** "Yes, on a 64-bit JVM. `long` is 64 bits and the hardware writes it
in one instruction. It's only a problem on 32-bit systems, which nobody uses any more."

**SENIOR answer:** "**No, not by specification** — and the gap between the specification
and the practice is the whole reason this question gets asked.

**JLS §17.7 is explicit:** reads and writes of `long` and `double` are **not guaranteed
to be atomic** unless the field is `volatile`. The JVM is permitted to treat a 64-bit
access as two 32-bit accesses. A concurrent reader can then observe one half of an old
value and one half of a new one — a value **nobody ever wrote**. That is word tearing.

**In practice, 64-bit HotSpot implements `long` and `double` accesses atomically**, so on
my machine and yours you are unlikely to ever observe tearing. I would say that out loud
rather than hide it, because pretending otherwise is a worse answer. But it is an
implementation property of one JVM on one class of hardware — not a guarantee, not
historically universal, and not something I would rely on in code that other people
maintain.

**And here is why I think the question is usually a trap rather than a trivia
question:** atomicity is almost never the actual problem in the code where it is asked.
If someone shows me `totalDebited += amount` on a `long`, the tearing question is the
*second* defect. The first is that `+=` is a read-modify-write — load, add, store, three
operations with two gaps — so increments are lost regardless of width, regardless of
architecture, and regardless of `volatile`. **`volatile` fixes the tearing and does
nothing at all for the lost update.** An engineer who answers only the atomicity half has
seen half the bug.

**And there is a third thing in the same line, which is visibility**: a plain `long`
written by one thread and read by another has no happens-before edge, so the reader may
see a stale value indefinitely. Three defects in eleven characters.

**What I'd actually write**: `LongAdder` for a counter, because writes dominate and reads
are occasional and it is striped to avoid the cache-line ping-pong; `AtomicLong` when I
need compound atomic operations like `getAndAdd` or `compareAndSet`; and a plain
`volatile long` only when I need visibility of a value that is *assigned*, never
incremented.

**One more thing worth knowing:** references are always written atomically — you never
see half a pointer. That does **not** make them safe. You can see a fully-formed
reference to a half-initialised object, which is Topic 88's entire subject."

**What separates them:** giving the specification answer first and the practical answer
second, clearly separated; naming JLS §17.7; identifying that the question is usually
asked about code whose real defect is the read-modify-write; stating explicitly that
`volatile` fixes tearing and not atomicity; adding visibility as a third distinct defect;
naming the right tool for each of the three cases; and closing with the reference-write
point, which sets up safe publication.

**Interviewer's follow-up:** *"How would you demonstrate tearing?"* — Honestly, I would
say up front that I very likely **cannot** demonstrate it on 64-bit HotSpot on aarch64 or
x86, because the implementation makes those accesses atomic. The honest experiment is a
jcstress test with two actors writing `0L` and `-1L` to a plain `long` and an arbiter
reading it, declaring the torn values (`0x00000000FFFFFFFF` and `0xFFFFFFFF00000000`) as
`ACCEPTABLE_INTERESTING`. **And a run with zero observations of those outcomes means
"not observed on this machine, this JDK, this run" — not "proven impossible".** The
specification permits it; the implementation happens not to produce it. That distinction
is the whole point of the question.

---

### Q4 — "It works on my laptop and fails in production on ARM. Why?"

**MID-LEVEL answer:** "ARM and x86 are different architectures, so the code behaves
differently. Probably a timing issue — production has more load, so races are more likely
to hit."

**SENIOR answer:** "I would first refuse the framing, because 'more load, so more races'
is the answer that leads a team in the wrong direction for a week. If it is a memory-model
bug, it is not more likely under load; it is **permitted always** and merely observed
sometimes.

**Then I would split it into two mechanisms and test them separately, because they have
different fixes and different architecture sensitivity.**

**Mechanism one: the compiler.** The JIT reorders, hoists and eliminates memory
operations whenever there is no happens-before edge forbidding it. Loop-invariant code
motion on an unsynchronised flag is the classic. **This is architecture-independent** — it
happens identically on x86 and aarch64 — and the reason it 'works on my laptop' is
usually that the laptop run was short and the method never reached C2, while production
runs for days. **The test is `-Xint` versus the JIT**, and it takes two seconds.

**Mechanism two: the hardware.** **x86-64 implements Total Store Order.** The hardware
forbids load-load, store-store and load-store reordering; the only reordering it permits
is a later load moving ahead of an earlier store to a different address. **aarch64 is
weakly ordered and permits all four.** So an unsynchronised program can be *sequentially
consistent in practice* on x86 and visibly reordered on ARM. **x86 hides a whole class of
bugs that ARM exposes.** That is the specific answer to the question as asked: the code
was always wrong; x86's stronger hardware model was papering over it.

**And note this cuts both ways for a team**: if the laptops are Apple Silicon and CI is
x86, then the *laptops* are the better test environment and CI is the one giving false
confidence. That is worth saying out loud, because most teams assume CI is authoritative.

**What I would actually do**, in order:

1. **Stop treating the x86 result as evidence of correctness.** It is evidence of a
   stronger hardware model, nothing more.
2. **Find the unsynchronised shared state by reading**, not by reproducing. The
   happens-before argument is the proof; reproduction is only useful for convincing
   people.
3. **Write a jcstress test for the specific invariant** and run it on both
   architectures. Declare the bad outcome `FORBIDDEN`. And be explicit with the team
   that a `FORBIDDEN` outcome with zero observations means **'not observed on this
   machine, this JDK, this run'**, not 'proven impossible'. The proof is the argument;
   jcstress catches the argument being wrong.
4. **Fix by establishing the edge**, and then verify the argument rather than the
   absence of the symptom.

**The sentence I'd want the team to leave with:** 'it works on x86' and 'it is correct'
are different claims, and only one of them is about our code."

**What separates them:** refusing the "more load means more races" framing; splitting
compiler from hardware and giving each its own test; stating x86-TSO's guarantees
precisely (which three reorderings it forbids and which one it permits); knowing which
way the laptop/CI asymmetry cuts; treating the happens-before argument as the proof and
the test as a falsifier; and volunteering the "zero observations is not impossibility"
caveat unprompted.

**Interviewer's follow-up:** *"How confident would you be after a clean jcstress run on
both architectures?"* — More confident, and not certain, and I would be precise about
why. jcstress explores interleavings aggressively and reports observed frequencies; it
does not enumerate the state space. A `FORBIDDEN` outcome with zero samples means the
outcome was not produced under the conditions tested — one JDK, one microarchitecture,
one set of JIT decisions, one machine load. What makes me confident is the happens-before
argument being airtight; jcstress is what tells me the argument was wrong when I thought
it was right. I would treat a clean run as *failure to falsify*, which is a real result
and is not proof.

---

### Q5 — "Walk me through what `-Xint` proves, and what it doesn't."

**MID-LEVEL answer:** "`-Xint` turns off the JIT so everything runs in the interpreter.
If the bug goes away with `-Xint`, it's a JIT bug."

**SENIOR answer:** "It is the single cheapest experiment in the JVM and it is very
frequently misread, so I'd be careful about both halves.

**What it proves.** `-Xint` runs everything in the interpreter, and the interpreter does
not optimise: it executes each `getfield` bytecode as an actual load from memory, every
time. So if a program hangs under the JIT and terminates under `-Xint`, **the compiler's
optimisation is on the causal path.** For an unsynchronised stop flag, that is direct
evidence of loop-invariant code motion hoisting the read. It converts 'something is wrong
with concurrency' into 'the read was optimised away' — a completely different
investigation.

**What it does NOT prove**, and this is the part people get wrong. **First**, it is not
evidence of a JVM bug: the absence of an edge is a licence and C2 used it; our code is the
defect. **Second**, `-Xint` terminating does not make the interpreted version correct —
the interpreter reads through the same store buffers, so the hardware staleness mechanism
is fully present and on a weakly-ordered architecture an interpreted program can still
observe stale and reordered values. `-Xint` removes one of the two mechanisms, not both.
**Third**, it is neither a fix nor a workaround; nobody ships it. **Fourth**, a
*negative* result is weak: hanging under both means the compiler was not the mechanism,
or the loop was not hot, or something else is holding the JVM open — I'd check for a
non-daemon thread before concluding anything.

**The reason I like it as a first move** is that it is a genuine A/B: one flag, one
artefact, one variable, partitioning the hypothesis space cleanly into 'compiler' and
'not compiler'. Most concurrency diagnostics are not that clean. **The follow-ups**, in
increasing precision: `-XX:TieredStopAtLevel=1` (is C1 alone enough?),
`-XX:-TieredCompilation` (force C2), `-XX:-UseOnStackReplacement` (was the running loop
compiled in flight?), and `-XX:+PrintCompilation` to timestamp the compilation against
the write."

**What separates them:** stating both what the experiment proves and the four things it
does not; explicitly rejecting "it's a JIT bug"; knowing the interpreter does not remove
the hardware mechanism; treating a negative result as weak rather than exculpatory; and
having the escalation sequence of follow-up flags ready.

**Interviewer's follow-up:** *"What is the equivalent A/B for a hardware reordering bug,
where `-Xint` won't help?"* — There isn't a flag; you change the *architecture*. Run the
same jcstress test on x86 and on aarch64 and compare the observed outcome sets. x86-TSO
forbids three of the four reorderings, so an outcome that appears on ARM and never on x86
is strong evidence the mechanism is hardware reordering rather than the compiler. That is
also why, for Topics 87 and 88, an Apple Silicon Mac is a *better* instrument than an x86
CI runner — and why a clean CI run on x86 should never be quoted as evidence.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The JMM says a read is *permitted* to return a stale value with no bound on how long.
   Derive, from that single sentence, why adding `Thread.sleep(5000)` before a read is
   not a fix — and then explain why it nevertheless makes the bug disappear often enough
   that people ship it.

2. `-Xint` makes the stop-flag program terminate. Explain the mechanism in terms of what
   the interpreter does per bytecode. Then explain why `-Xint` terminating does **not**
   establish that the interpreted program is free of memory-model bugs.

3. Cache coherence guarantees no two cores hold different values for the same address.
   Given that, explain how a reader can still observe a stale value — and name the
   hardware structure responsible. Then explain why that structure is also the reason
   x86-TSO permits exactly one of the four reorderings.

4. Transitivity (edge 10) is what makes a lock publish more than the variables inside the
   block. Construct a concrete three-step chain showing a plain, non-`volatile` field
   being safely published across threads, and state which edge is used at each step.

5. Two threads, a plain field, and a one-second gap between the write and the read. Is
   that a data race? Answer from the definition, not from intuition, and then say what
   changes if the gap is one nanosecond.

6. The stop-flag bug is caused by the compiler and reproduces identically on x86 and
   aarch64. Construct a *different* memory-model bug that is caused by the hardware and
   would be far more likely to be observed on aarch64 than on x86. Say which reordering
   your example depends on.

7. You have a `volatile boolean shutdownRequested` and a worker thread blocked in
   `queue.take()`. The flag is set. Explain precisely why the worker does not stop, even
   though the memory model is now satisfied — and name the mechanism that would stop it.

---

## Quick reference card

### The happens-before edges — the complete list

**This is the table. Come back to it.**

| # | Edge | Rule |
|---|---|---|
| 1 | **Program order** | Within one thread, each action happens-before every later action in that thread's program order |
| 2 | **Monitor lock** | An unlock of a monitor happens-before every subsequent lock of **that same** monitor |
| 3 | **Volatile** | A write to a `volatile` field happens-before every subsequent read of **that same** field |
| 4 | **Thread start** | `Thread.start()` happens-before any action in the started thread |
| 5 | **Thread join** | Every action in a thread happens-before another thread returns from `join()` on it |
| 6 | **Thread termination** | Every action in a thread happens-before another thread detects it has terminated |
| 7 | **Interruption** | `interrupt()` happens-before the interrupted thread detects the interrupt |
| 8 | **Final field freeze** | The end of a constructor happens-before a read of any `final` field of that object — **provided `this` did not escape** (Topic 88) |
| 9 | **Default values** | The write of the default value (0 / false / null) happens-before the first action of every thread |
| 10 | **Transitivity** | A `hb` B and B `hb` C implies A `hb` C — **the rule that makes the others useful** |

**Derived (documented in `java.util.concurrent` Javadoc, built from the ten above):**
`submit` before task start · task end before `Future.get` · `countDown` before `await`
returns · `release` before `acquire` · `put` before the matching `take` · CHM write to K
before a read of K · `unlock` before `lock` · static-initialiser completion before first
use.

### The four barrier types

| Barrier | Prevents | x86-64 (TSO) | aarch64 |
|---|---|---|---|
| **LoadLoad** | A later load moving before an earlier load | Free — TSO forbids it already | Real barrier / `ldar` |
| **StoreStore** | A later store becoming visible before an earlier store | Free — TSO forbids it already | Real barrier / `stlr` |
| **LoadStore** | A store moving before an earlier load | Free — TSO forbids it already | Real barrier |
| **StoreLoad** | A load executing before an earlier store is globally visible | **The expensive one.** `mfence` or `lock addl $0,(%rsp)` | `dmb ish` |

**How the constructs lower:**

| Construct | Barriers | x86-64 | aarch64 |
|---|---|---|---|
| `volatile` read | LoadLoad + LoadStore after | Plain `mov` — **free** | `ldar` |
| `volatile` write | StoreStore before, **StoreLoad** after | `mov` + locked op / `mfence` — **the cost** | `stlr` (+ `dmb ish`) |
| `synchronized` enter | Acquire | `lock cmpxchg` is already a full barrier | `ldaxr`/LSE with acquire |
| `synchronized` exit | Release | Ordered by the locked op | `stlr` / `dmb ish` |
| `final` freeze | StoreStore at constructor end | Free | `dmb ishst` |

### Architecture, in one box

- **x86-64 = Total Store Order.** Forbids Load-Load, Store-Store, Load-Store. Permits
  **only** Store-then-Load to a different address.
- **aarch64 (your Mac) = weakly ordered.** Permits all four without barriers.
- **Consequence:** x86 hides bugs aarch64 exposes. Your Mac is a **better** instrument
  for Topics 87 and 88 than x86 CI. **It is not a guarantee of reproduction** — never say
  a race "will" reproduce.
- **Exception:** the Topic 86 stop-flag bug is a **compiler** effect. Identical on both
  architectures.

### Diagnostic commands

```bash
# THE instrument for this topic: the A/B that separates compiler from everything else.
java -Xint -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"
java       -cp out com.orderflow.lab.jmm.StopFlag ; echo "exit=$?"

# Narrow it: which compiler, and was the running loop compiled in flight?
java -XX:TieredStopAtLevel=1     -cp out ...     # C1 only
java -XX:-TieredCompilation      -cp out ...     # C2 only
java -XX:-UseOnStackReplacement  -cp out ...     # no in-flight loop compilation

# Corroborate: was the method compiled, when, at what tier, with OSR ('%' marker)?
java -XX:+PrintCompilation -cp out ... 2>&1 | grep -i <ClassName>
-Xlog:jit+compilation=debug:file=jit.log:time,uptime,tags

# The production diagnostic. Look for RUNNABLE with no lock lines.
jcmd <pid> Thread.print > dump.txt
grep -A10 '<thread-name>' dump.txt
grep -c 'BLOCKED' dump.txt              # expect 0 for a visibility hang
grep -c 'waiting to lock' dump.txt      # expect 0 for a visibility hang

# The bytecode still contains the read. Proves the elimination is downstream of javac.
javap -c -p -cp out com.orderflow.lab.jmm.StopFlag

# Ground truth, and heavy: requires hsdis, which is NOT bundled with most JDKs.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,com.orderflow.lab.jmm.StopFlag::* -cp out ...

# Never quote a default you did not print.
java -XX:+PrintFlagsFinal -version | grep -iE "Tiered|OnStackReplacement"
jcmd <pid> VM.flags -all
```

### `[JAVA 25]` one-line note

`-XX:+UseCompactObjectHeaders` (JDK 25) narrows the object header and changes the mark
word's field widths. **It changes nothing in this document** — no happens-before edge, no
barrier, no JIT optimisation depends on header width. It is flagged only so that when you
read a mark word with JOL in Topic 85 or Topic 88 and the layout differs from a blog
post, you know why. Settle it with
`java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders` and with JOL's
`ClassLayout.parseInstance(o).toPrintable()`.

### Gotchas checklist

- [ ] "Eventually" is not in the specification. A stale read may persist forever.
- [ ] The stop-flag bug is the **compiler**, not the cache. `-Xint` proves it.
- [ ] `volatile` does not "flush the cache". It forbids the hoist and emits barriers.
- [ ] An edge needs **both** sides. Synchronising only the writer buys nothing.
- [ ] A data race has no timing component. Seconds apart is still a race.
- [ ] Adding a `println` or a `sleep` that "fixes" it is an accident, and worse than the bug.
- [ ] `long` / `double` are not guaranteed atomic without `volatile` (JLS §17.7).
- [ ] `volatile` fixes tearing and visibility. It never fixes read-modify-write.
- [ ] Transitivity is what makes a lock publish everything before it, not just the block.
- [ ] `RUNNABLE` + no locks + won't stop = visibility bug until proven otherwise.
- [ ] x86 hides what aarch64 exposes. A clean x86 CI run is weak evidence.
- [ ] A test passing is failure-to-falsify, not proof. The proof is the edge.
- [ ] Prefer `interrupt()` over a flag for shutdown: it is an edge **and** it wakes a parked thread.
- [ ] Prefer owning no thread at all: executors bring their own edges.
- [ ] Log "loop exited" from every worker loop, and check for it on shutdown.

---

## When would I use this at work?

**1. Reviewing any pull request that adds a background thread, a cache, or a flag.**

You now have a one-question review checklist that takes ten seconds and catches a whole
bug class: *this field is written by thread X and read by thread Y — which numbered edge
orders them?* If nobody can name one from the ten-row table, it is a defect, and you can
say so before anything has been run. That is a different kind of review comment from "I
think there might be a race here" — it is a specific, citable, falsifiable claim. Over a
year this catches more production incidents than any amount of load testing, because
these bugs do not reproduce on demand and therefore do not get caught downstream.

**2. Diagnosing a hang that is not a deadlock.**

Every engineer reaches for the thread dump. Most of them can only read the `BLOCKED`
cases — the deadlock section, the `waiting to lock` lines. When the dump shows a
`RUNNABLE` thread that will not stop and no lock information anywhere, most
investigations stall, because the obvious hypotheses have all been eliminated. You have a
specific hypothesis for that exact shape, and a two-second `-Xint` A/B to confirm it. The
alternative is days of investigation into a database that was never involved.

**3. Deciding what a "flaky" test is telling you.**

A test that passes 98 times in 100 gets quarantined, retried, or given a bigger timeout.
After this topic you can distinguish three cases that look identical from the outside: a
genuine timing sensitivity in the test, an environment problem, and **a real
memory-model defect whose permitted-but-unwanted outcome is simply rare on this
hardware.** The third is the one that costs a production incident, and the tell is that
the failure rate changes when you change the JIT settings, the architecture, or the
machine load — none of which should affect a correct program's outcome. Being the person
who says "before we retry this, run it with `-XX:-TieredCompilation` and on the ARM
runner" is a materially different level of engineer.

---

## Connected topics

**Prerequisites:**

- **17 — Immutability, `final`, safe publication**: where you were first told `final` has
  memory-model meaning and not just "cannot be reassigned". Edge 8 is the formal version
  of that claim, and **Topic 88 is the promised payoff.**
- **27 — Records and shallow immutability**: a record's components are `final`, so they
  get edge 8 free — and a record holding a mutable `List` gets the guarantee on the
  reference and nothing on the contents. That trap starts here and lands at Topic 88.
- **66 — JVM architecture**: per-thread stacks versus the shared heap. Every bug in this
  document lives on the heap side; nothing on a stack can race.
- **69 — The mark word**: edge 2 is implemented by a CAS on the mark word, which is also
  a full barrier on x86. The header is where the edge physically lives.
- **73 — Safepoints and TTSP**: **the same compiler reasoning that elides a counted
  loop's safepoint poll is what hoists the flag read** — one licence, two symptoms. Also
  why `jcmd Thread.print` can itself hang on a JVM with this bug.
- **74 — JIT and tiered compilation**: tier thresholds, on-stack replacement,
  deoptimisation, `-XX:+PrintCompilation`. **This document's bug IS a JIT optimisation**,
  and you cannot diagnose it without 74's vocabulary.
- **75 — Escape analysis and inlining**: loop-invariant code motion and the licence to
  hoist a read out of a loop. **Topic 75 taught you to admire this optimisation; Topic 86
  shows you its price.**
- **76 — Bytecode and `javap`**: proves the read exists in the bytecode, which is what
  makes its absence from the machine code so striking.
- **84 — Threads versus the event loop**: preemption between any two bytecodes and the
  lost-update trace. This document adds the deeper problem: even a *correctly ordered*
  sequence of operations gives no visibility guarantee.
- **85 — `synchronized` and monitors**: edge 2, the first edge you used. Its Trap 4
  (double-checked locking without `volatile`) deferred its explanation to Topics 86 and
  88; this document supplies the first half.

**This unlocks:**

- **87 — `volatile` and memory barriers**: edge 3 in full mechanical detail — which
  barriers, where, what they cost on x86 versus aarch64, and why visibility is not
  atomicity.
- **88 — `final` fields and safe publication**: edge 8, its precondition, and the five
  safe-publication idioms. **The payoff Topic 17 promised.**
- **89 — `wait`/`notify`**: guarded blocks on edge 2, plus spurious wakeups. The
  `while`-not-`if` rule is a memory-model rule as much as a liveness one.
- **90 — Executors**: `submit`/`get`/`shutdown` carry edges so you do not have to build
  them. The right fix for this document's `orderflow` scenario.
- **92 — `ConcurrentHashMap`**: lock-free reads are volatile reads of `Node.val` — edge 3
  placed deliberately, which is why the per-key guarantee can be documented at all.
- **94 — Explicit locks**: `ReentrantLock` gives edge 2's guarantee through AQS's
  `volatile int state`. Same edge, different ergonomics.
- **95 — Atomics and CAS**: `AtomicX` operations are edge-3-equivalent, and `VarHandle`
  exposes access modes (`getAcquire`, `setRelease`, `getOpaque`) that let you pick a
  weaker edge deliberately.
- **99 — jcstress**: the tool for falsifying a happens-before argument. Its honesty rules
  are this document's: zero observations means "not observed here", never "impossible".
- **101 — Virtual threads**: mounting and unmounting a continuation carries happens-before
  edges, which is why a virtual thread's writes survive a carrier migration. Pinning is a
  scheduling problem, not a memory-model one, and telling those apart is this document's
  skill.
- **102 — Structured concurrency and `ScopedValue`**: a `StructuredTaskScope` gives edges
  from fork to join by construction, and `ScopedValue`'s immutability is the final-field
  guarantee applied to context propagation.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted: whether the JIT hoists the read on your specific JDK build
and loop shape, whether C1 alone is sufficient to produce it, and whether `long`/`double`
tearing is observable on your hardware (it very likely is not, on 64-bit HotSpot). Each
has a command in the Hands-on section that settles it on your machine in under a minute,
and the failure drill's result table explicitly covers the outcome where the bug does not
reproduce. **No hang, timing, percentile or throughput figure in this document was
measured — I have no JVM.** Two rules govern every experiment in Topics 86 through 88 and
are repeated deliberately: a permitted-but-unobserved outcome is "not observed on this
machine, this JDK, this run", never "proven impossible"; and x86's Total Store Order
hides reordering bugs that aarch64 exposes, which makes your Apple Silicon Mac a better
instrument than x86 CI for Topics 87 and 88 — and makes no difference at all to this
document's drill, because that one is the compiler. What has been stable since Java 5 and
will still be true at 2am: the JMM is a permission model, not a promise; "eventually" is
not in the specification; an edge requires both sides to participate; transitivity is what
makes an edge carry everything before it; and the fix is never a sleep.*
