# 87 — `volatile` and Memory Barriers

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: `orderflow`'s hot-SKU reservation counter and the wallet balance are both `volatile` and both still wrong. This is the topic where you learn to say precisely what `volatile` bought (visibility and ordering, on one field), what it did not buy (atomicity of anything compound), and how to prove both with jcstress rather than assert either.

---

## Mechanical statement

Read this three times. Every section below is an elaboration of it.

> **A `volatile` write emits a `StoreStore` barrier before it and a `StoreLoad` barrier
> after it. A `volatile` read emits `LoadLoad` and `LoadStore` barriers after it.**
>
> On **x86-64**, whose Total Store Order model already forbids three of the four
> reorderings, the volatile **read** compiles to an ordinary `mov` — it is **free** — and
> the volatile **write** compiles to a store followed by a locked instruction
> (`lock addl $0, (%rsp)`) or `mfence`, which is the only way to buy `StoreLoad`.
>
> On **aarch64** — your Apple Silicon Mac — the hardware provides none of it implicitly,
> so the read becomes `ldar` (load-acquire) and the write becomes `stlr` (store-release),
> typically with a `dmb ish` to supply `StoreLoad`. **Both cost instructions here.**
>
> **This buys you exactly two things: visibility and ordering.**
>
> - **Visibility**: a volatile write happens-before every subsequent volatile read of
>   that same field (edge 3 from Topic 86). The reader sees the value.
> - **Ordering**: everything the writer did *before* the volatile write is visible to a
>   reader that observes the write — by transitivity (edge 10). One volatile write
>   publishes an entire object graph.
>
> **It buys you NO atomicity for compound operations.** `i++` is a read, an add and a
> write: three operations, two gaps, and `volatile` makes each of the three individually
> visible while doing nothing whatsoever about the gaps between them.

Four consequences follow directly. **`volatile int i; i++;` loses increments** —
reliably, under contention, on every architecture; `volatile` is not a weak lock, it is
not a lock at all. **`volatile` is exactly right for a flag** (write once, read many)
**and exactly wrong for a counter** (read-modify-write). **The asymmetric cost is real**:
on x86 you may add `volatile` to a hot *read* path for free and pay meaningfully on the
*write* path, while on aarch64 you pay for both — which is why "just make everything
volatile" is a different decision on your Mac than on an x86 server, and why you must
measure on the target. And **the ordering half is the underrated half**: most engineers
learn "volatile means other threads see it", but the transitive publication — *and
everything I wrote before it* — is what makes safe publication (Topic 88),
double-checked locking, and `ConcurrentHashMap`'s lock-free reads work at all.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

Topic 86 established why a single-threaded, run-to-completion runtime cannot have a
memory model. `volatile` is a *tool for repairing* a memory model problem, so it inherits
the same verdict: **there is nothing in your Node model that `volatile` corresponds to,
because there is nothing in your Node model for it to fix.**

Be precise about the shape of the absence. **There is no ordering question in Node**,
because reordering is only observable from a second thread — V8 reorders your memory
operations constantly and you have never once been able to detect it. **There is no
visibility question**, because there is one reader. **And critically, there is no "this
is visible but not atomic" distinction** — the distinction this entire document is built
on. In Node, `count++` inside a callback is atomic *for free*, because run-to-completion
guarantees nothing runs between the read and the write. You have never had to separate
those two properties, because you have never had one without the other.

### Do not map this onto the event loop. Explicitly.

The specific wrong analogies to refuse, and why each one installs a model that costs more
to remove than it saves:

| The tempting analogy | Why it is wrong |
|---|---|
| "`volatile` is like `queueMicrotask` — it forces the update to be picked up" | Maps ordering onto **scheduling**. A volatile write does not schedule anything. Both threads are already running, at full speed, on separate cores. |
| "`volatile` is like `await` — it's a synchronisation point" | `await` yields control. `volatile` yields nothing and blocks nobody. Zero threads stop. |
| "A volatile read is like reading from a `SharedWorker`'s message" | Message passing copies. `volatile` shares. The whole problem is that nothing is copied. |
| "`volatile int i; i++` is like `count++` in a callback — obviously atomic" | **The single most dangerous one.** Run-to-completion made `count++` atomic for you. Java has no such property, and `volatile` does not restore it. |

The last row is the reason this block is mandatory. Your intuition says `i++` is one
thing. In Node it effectively is. **In Java it is three, and `volatile` fixes none of the
gaps.** That is the whole document.

### The one honest exception: `SharedArrayBuffer` + `Atomics`

The exception from Topic 86 is unusually informative here, because JavaScript's designers
faced the exact same problem and made the exact same split.

Once you have a `SharedArrayBuffer` shared between `worker_threads`, ECMAScript gives you
two distinct families of operation, and the split is `volatile`'s split:

| JavaScript | Java | What it gives you |
|---|---|---|
| `Atomics.store(ta, i, v)` | `volatile` write | Ordered, visible. **Not** a compound operation |
| `Atomics.load(ta, i)` | `volatile` read | Ordered, visible |
| `ta[i] = v` (plain, on a shared buffer) | plain field write | Unordered, may be observed stale or reordered |
| **`Atomics.add(ta, i, 1)`** | **`AtomicInteger.incrementAndGet()`** (Topic 95) | **Atomic read-modify-write.** A *different function*, because store/load cannot express it |

**Look at that last row and take the lesson from it.** JavaScript could not express an
atomic increment with `Atomics.store` and `Atomics.load` either. It needed a separate
primitive, `Atomics.add`, because *read-modify-write is a different operation from
ordered store*. That is precisely why Java has `AtomicInteger` alongside `volatile`. **The
languages agree, for the same reason, at the same layer.**

If you have used `Atomics`, Topics 87 and 95 will feel like recognition. **Most JavaScript
engineers have not** — `SharedArrayBuffer` is gated behind cross-origin isolation in
browsers and is rare in Node application code. If that is you, this table is a signpost,
not a shortcut: it tells you these are not Java eccentricities but what any language must
provide once two threads touch the same memory.

### What actually transfers

| You know | Here | Why it helps |
|---|---|---|
| A CDN or replica returns a stale read; you need a consistency model, and stronger consistency costs latency | A volatile read is a stronger consistency guarantee that costs instructions | **The strongest transfer available.** The trade is identical: stronger guarantee, higher cost, applied per access. |
| A compare-and-set in Redis (`WATCH`/`MULTI`, or `SET ... NX`) is a *different operation* from `GET` then `SET` | `compareAndSet` is a different operation from a volatile read then a volatile write | You already know check-then-act is not atomic in a distributed system. It is not atomic in one JVM either, and for the same reason. |

---

## What is this?

### The definition

`volatile` is a field modifier. It applies only to fields — never to a local variable,
never to a method, never to a class. It makes three guarantees and no others:

1. **Visibility.** A write to a `volatile` field happens-before every subsequent read of
   that same field by any thread (JLS §17.4.4). This is edge 3 from Topic 86's table.
2. **Ordering.** Volatile accesses are not reordered with respect to each other, and —
   crucially — the JIT and the CPU may not move ordinary memory operations across them in
   the directions the barriers forbid. Combined with transitivity, this publishes
   everything the writer did before the write.
3. **Atomicity of the access itself, including 64-bit.** A `volatile long` or
   `volatile double` read or write is guaranteed atomic (JLS §17.7). Without `volatile`,
   64-bit accesses are **not** guaranteed atomic — Topic 86's Trap 5.

And the explicit non-guarantee, which is the whole point of this document:

> **`volatile` provides no atomicity for any operation composed of more than one access.**
> `i++`, `i += n`, `i = i * 2`, `if (x == null) x = new Thing()`, `if (stock > 0)
> stock--` — every one of those is a sequence, and `volatile` orders the individual
> accesses without joining them.

### What `volatile` forbids the compiler from doing

This is the half that Topic 86's drill made concrete. A `volatile` field access is a
**memory operation the compiler must actually perform**, at the point where you wrote it.

| Optimisation | Legal on a plain field | Legal on a `volatile` field |
|---|---|---|
| Hoist the read out of a loop (LICM) | **Yes** — Topic 86's bug | **No** |
| Cache the value in a register across a call | Yes | No |
| Eliminate a redundant re-read | Yes | No |
| Reorder with respect to other memory ops | Yes, subject to program semantics within the thread | Only in the directions the barriers permit |
| Eliminate a write nobody reads in this thread | Yes | No |
| Merge two writes into one | Yes | No |

**That is why `volatile` fixes the stop flag** and why nothing else — not a sleep, not a
retry — does.

### The four barrier types, and where `volatile` puts them

You met the four types in Topic 86. Here is the placement, which is the thing to
memorise:

```
        ... ordinary reads and writes ...
        [ StoreStore barrier ]          <-- nothing before can sink below
   W:   volatile write to `ready`
        [ StoreLoad barrier  ]          <-- THE EXPENSIVE ONE
        ... ordinary reads and writes ...


        ... ordinary reads and writes ...
   R:   volatile read of `ready`
        [ LoadLoad barrier   ]          <-- nothing after can rise above
        [ LoadStore barrier  ]
        ... ordinary reads and writes ...
```

Read the two boxes together and the mechanism falls out. The **`StoreStore` before the
write** guarantees that every ordinary write the thread did earlier is visible to another
core **before** the volatile write is — that is what makes the flag a valid *signal*: if
you see `ready == true`, everything written before it is already there. The
**`LoadLoad`/`LoadStore` after the read** guarantees that no subsequent load is hoisted
above the volatile read — that is what makes reading the flag a valid *test*: if the flag
says the data is ready, your reads of the data cannot have happened before you checked.

**Those two halves together are the message-passing idiom**, and it is the single most
important pattern in the whole document:

```java
// Writer thread
data = computeExpensiveThing();     // ordinary write
ready = true;                        // VOLATILE write  -- publishes `data` too

// Reader thread
if (ready) {                         // VOLATILE read
    use(data);                       // guaranteed to see computeExpensiveThing()'s result
}
```

`data` is a plain field. It is safely published anyway, by transitivity: the plain write
happens-before the volatile write (program order), the volatile write happens-before the
volatile read (edge 3), the volatile read happens-before the plain read (program order).
**One volatile field publishes an unbounded amount of ordinary state.** This is called
*piggybacking on synchronisation*, and it is how `ConcurrentHashMap`, `CopyOnWriteArrayList`
and half of `java.util.concurrent` achieve lock-free reads.

### The `StoreLoad` barrier, and why the write is the expensive one

`StoreLoad` is the only barrier x86 needs, and it is expensive on every architecture, for
a physical reason worth understanding.

A `StoreLoad` barrier says: *no load after this point may execute until the store before
this point is visible to every other core.* The store is sitting in this core's **store
buffer** (Topic 86, Layer 1). To satisfy the barrier, the core must **drain the store
buffer** — push the pending write out into the coherent cache hierarchy, which may
require obtaining the line in `M` state and invalidating every other core's copy.

**That is a pipeline stall plus potentially an interconnect round-trip.** The other three
barriers are, on a strong-memory-model machine, essentially bookkeeping for the compiler.
`StoreLoad` is real work for the hardware.

Hence the asymmetry, which is the number one practical fact in this document:

| | x86-64 | aarch64 (your Mac) |
|---|---|---|
| **volatile read** | Ordinary `mov`. **Free** | `ldar`. Costs, but modestly |
| **volatile write** | `mov` + `lock addl $0,(%rsp)` (or `mfence`). **Expensive** | `stlr` (+ `dmb ish`). **Expensive** |

### What `volatile` is for, and what it is not for

| Use it for | Do not use it for |
|---|---|
| A **stop flag** or state flag — write once (or rarely), read constantly | A **counter** — `i++` is a read-modify-write |
| **Safe publication of an immutable object** — build it, then one volatile write of the reference | Any **check-then-act** — `if (a) { b(); }` where another thread can change `a` |
| The **`instance` field in double-checked locking** — this is *mandatory*, not optional | **Two related fields** that must change together — one volatile write cannot cover two fields |
| A value **assigned** from one place and read from many — a config snapshot, a feature flag, a cached reference | A value that is **incremented, accumulated, or conditionally updated** |
| **64-bit values** where you need the access itself to be atomic | Anything where you need **mutual exclusion** |

**The one-line test:** *is every operation on this field a single read or a single
write?* If yes, `volatile` is a candidate. If any operation reads the field and then
writes a value derived from it, `volatile` is the wrong tool and you want
`AtomicInteger`/`AtomicReference` (Topic 95), a lock (Topics 85, 94), or a redesign.

### `volatile` on a reference, and what it does not cover

This trips up more engineers than the counter case, so it gets stated flatly:

```java
private volatile List<String> hotSkus = new ArrayList<>();
```

**`volatile` applies to the field, not to the object the field points at.** You get a
guarantee that a reader sees the most recently written *reference*. You get **no**
guarantee about the state of the `ArrayList` it points to. `hotSkus.add("SKU-1")` is not
a write to the volatile field at all — the field is unchanged — so it emits no barrier
and establishes no edge.

The same applies to arrays:

```java
private volatile int[] counters = new int[100];
counters[7]++;    // NOT a volatile operation. The field `counters` was never written.
```

**The correct patterns:**

```java
// 1. Immutable snapshot, volatile reference. Readers are lock-free and consistent.
private volatile List<String> hotSkus = List.of();
public void refresh(Collection<String> fresh) {
    this.hotSkus = List.copyOf(fresh);        // ONE volatile write, publishes the whole list
}

// 2. For per-element atomicity on an array: AtomicIntegerArray (Topic 95)
private final AtomicIntegerArray counters = new AtomicIntegerArray(100);
counters.incrementAndGet(7);

// 3. For a mutable shared map: ConcurrentHashMap (Topic 92)
```

Pattern 1 is the shape you will use most in `orderflow`, and it is worth naming: **a
`volatile` reference to an immutable object is a complete, lock-free, correct
publication mechanism.** Topic 88 explains why the immutability half matters.

### `[JAVA 25]` one line on headers

`-XX:+UseCompactObjectHeaders` (JDK 25) narrows the object header and changes the mark
word's field widths. **It changes nothing about `volatile`** — no barrier, no edge, no
JIT rule depends on header width. It is flagged only so that a JOL layout that differs
from a blog post does not confuse you. Settle it with
`java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders`.

---

## Why does it matter?

**1. Because "volatile makes it thread-safe" is the most common wrong belief in Java, and
it produces silently-wrong money.** The field is `volatile`; it reads in review as
evidence of care; and the increment guarding a wallet balance or a stock level loses
updates under exactly the load you built at Topic 65. No exception, no log, a plausible
final number.

**2. Because the ordering half is what makes lock-free code possible, and almost nobody
learns it.** An engineer who knows only "volatile means visible" cannot explain why
double-checked locking needs it, why `ConcurrentHashMap`'s reads are lock-free, or how a
single volatile write publishes an entire object graph. The transitive publication is the
mechanism behind most of `java.util.concurrent`.

**3. Because the cost is asymmetric and architecture-dependent, and you are on aarch64.**
On x86 a volatile read is genuinely free, so "make the read path volatile" costs nothing
measurable. On your Mac it is not free. Any performance intuition imported from an x86
blog post about `volatile` is suspect; measure on the target with JMH (Topic 77).

**4. Because it is the first topic where you can actually *test* a concurrency claim.**
Topic 86 gave you an A/B. This topic gives you **jcstress**: a tool that runs a pair of
actors millions of times across JIT modes and reports the frequency of every observed
outcome against outcomes you declared. That is the difference between asserting your code
is thread-safe and demonstrating which outcomes are reachable.

**5. Because on `orderflow` both money-critical fields are candidates.** The hot-SKU
reservation counter and the wallet balance are read constantly and written under
contention. The `volatile` decision on each is a real design judgement with a measurable
throughput consequence, not a style preference.

---

## Machine-level reality

### The store buffer, restated for this topic

Topic 86 established: a core writes into a per-core **store buffer** and continues
immediately; the buffer drains into the coherent cache later; **the store buffer is not
part of the coherence protocol**, so a write sitting in it is invisible to every other
core, and no amount of coherence helps.

**`volatile`'s job is to put a bound on that.** The `StoreLoad` barrier after a volatile
write forces the buffer to drain before any subsequent load may execute. That is the
entire physical mechanism, and it is why the write is the expensive half.

### Cache coherence (MESI) — and the cost you are paying

Every cache line in every core's cache is **M**odified, **E**xclusive, **S**hared, or
**I**nvalid. To write a line, a core must hold it in `M`, which requires invalidating
every other core's copy — a broadcast on the interconnect.

**The consequence for `volatile`:** a `volatile` field written by many threads and read
by many threads is a **cache line that ping-pongs**. Each write invalidates every
reader's copy; each subsequent read must re-fetch it. Throughput can *decrease* as you
add cores. That is Topic 95's CAS story and Topic 96's false-sharing story, and it starts
here: **the cost of `volatile` is not the barrier instruction, it is the coherence
traffic the barrier makes unavoidable.**

This is also why the read/write ratio matters so much. A flag written once at shutdown
and read a billion times costs essentially nothing: the line settles into `S` state in
every reader's cache and stays there. A counter written by eight threads is a line in
permanent `M`-state migration.

### x86-TSO versus aarch64 — and which way it cuts on your Mac

**x86-64 implements Total Store Order:**

| Reordering | x86-TSO | aarch64 |
|---|---|---|
| Load then Load | **Forbidden** | Permitted |
| Load then Store | **Forbidden** | Permitted |
| Store then Store | **Forbidden** | Permitted |
| Store then Load (different addresses) | **Permitted** — the one hole | Permitted |

**Which way it cuts, in your favour:** three of the four reorderings that this
document's barriers exist to prevent are *already impossible* on x86. So a program that
forgets `volatile` can be accidentally correct on x86 and visibly broken on aarch64.
**Your Apple Silicon Mac will surface reordering outcomes that an x86 CI runner would not
produce in a thousand years.** For the jcstress tests below, that makes your laptop the
better instrument.

**Which way it cuts, against you:** a jcstress result from your Mac and one from x86 CI
are **different experiments**. A clean x86 run is close to worthless as evidence about
aarch64. A clean aarch64 run is stronger and is still not proof.

**And the honest limit, which governs every experiment in this document:**

> **I will never tell you a race "will" reproduce.** Whether a reordering is observed
> depends on the JIT's output, which cores the OS scheduled you on (and on Apple Silicon,
> whether they were performance or efficiency cores), machine load, and luck. Every
> result table below has a "you saw nothing" row, and that row is a legitimate outcome.

**And the distinction carried forward from Topic 86:** Topic 86's stop-flag bug was a
**compiler** effect, identical on both architectures. **This document's bugs are
different.** The lost increment is a scheduling/interleaving effect, visible everywhere.
The *reordering* bugs — the message-passing test, and Topic 88's publication test — are
**hardware** effects, and those are the ones x86 hides and aarch64 exposes. Know which
kind you are looking at before you interpret a result.

### How `volatile` lowers — the instruction level

**On x86-64:**

```
; volatile READ of `ready`  (a boolean field)
    movzbl  0x0c(%rsi), %eax          ; ...that is the whole thing.
                                      ; TSO already forbids LoadLoad and LoadStore
                                      ; reordering, so NO barrier instruction is needed.

; volatile WRITE of `ready = true`
    movb    $0x1, 0x0c(%rsi)          ; the store itself
    lock addl $0x0, (%rsp)            ; a locked no-op used as a full fence.
                                      ; Cheaper than `mfence` on many microarchitectures,
                                      ; and it supplies the StoreLoad barrier.
```

**On aarch64 (your Mac):**

```
; volatile READ of `ready`
    ldarb   w0, [x1]                  ; load-ACQUIRE. Orders this load against
                                      ; everything after it. A real instruction.

; volatile WRITE of `ready = true`
    stlrb   w0, [x1]                  ; store-RELEASE. Orders everything before it
                                      ; against this store.
    dmb     ish                        ; data memory barrier, inner shareable -
                                      ; supplies the StoreLoad half.
```

***Illustration of the lowering, not disassembly captured from a specific JVM.** The
exact instruction selection varies by JDK version, by microarchitecture, and by
surrounding code — HotSpot may, for example, use the `dmb`-free `ldar`/`stlr` pairing
where it can prove the `StoreLoad` is unnecessary. Confirm on your own runtime with
`-XX:+PrintAssembly`, which needs `hsdis` — see the Measurement section for the honest
caveats.*

**The single sentence to carry into an interview:** *on x86 a volatile read is a plain
`mov` and costs nothing; the write needs a locked instruction and costs. On aarch64 both
are real instructions — `ldar` and `stlr` — so the cost profile is different, and a
benchmark from one architecture does not transfer to the other.*

### The constructor freeze — where it sits relative to `volatile`

Stated here because it completes the barrier picture and because Topic 88 needs it:

> At the end of a constructor in which a `final` field is set, the JVM performs a
> **freeze action**: mechanically, a **`StoreStore` barrier** before the constructor
> returns. It prevents the store of the object's *reference* being reordered before the
> stores of its `final` *fields*.

Compare the three publication mechanisms as barriers, because the comparison is the
clearest way to see what each buys:

| Mechanism | Barrier | Covers | Requires the reader to do anything? |
|---|---|---|---|
| `final` field freeze | `StoreStore` at constructor end | Only the `final` fields, only if `this` did not escape | **No** — this is its remarkable property |
| `volatile` write of the reference | `StoreStore` before, `StoreLoad` after | **Everything** the writer did before the write | **Yes** — the reader must read the volatile field |
| `synchronized` block | Release on exit, acquire on enter | Everything before the unlock | **Yes** — the reader must take the same lock |

**`final` is the only one that asks nothing of the reader.** That is Topic 88's headline,
and the reason `final` is not merely a style preference.

---

## Concurrency trace

**Read this before you read any correct code.**

### The scenario

`orderflow` keeps an in-memory reservation counter per hot SKU, in front of Postgres, to
shed obviously-impossible orders. Someone reviewed it, noticed a visibility problem, and
fixed it the way most people do:

```java
@Component
public class ReservationCounter {

    /** Made volatile after a Topic 86-style incident. The reviewer approved it. */
    private volatile int reservedUnits = 0;

    public void reserveOne()  { reservedUnits++; }        // <-- THE BUG
    public void releaseOne()  { reservedUnits--; }
    public int  reserved()    { return reservedUnits; }   // this read IS correct
}
```

`SKU-4471` has 10 physical units. The counter says how many are reserved; available
stock is `10 - reservedUnits`.

Thread A is an `orderflow` **order-placement** request thread. Thread B is the
**inventory reconciler**, which also reserves units when it detects an in-flight
Postgres reservation the in-memory counter has not accounted for.

Both are executing `reservedUnits++`, which compiles to (Topic 76's tooling confirms it):

```
getfield  reservedUnits      // (1) READ   - a volatile read: ldar / mov
iconst_1
iadd                         // (2) COMPUTE - purely in a register, on the private stack
putfield  reservedUnits      // (3) WRITE  - a volatile write: stlr+dmb / mov+lock
```

**Three bytecodes. `volatile` makes step (1) and step (3) individually ordered and
visible. It does nothing at all about the gap between them.**

### The interleaving

| Step | Thread A (order placement) | Thread B (inventory reconciler) | Shared state / what is visible |
|---|---|---|---|
| 1 | Enters `reserveOne()` for order #90210 | — | `reservedUnits = 8`. Available = 2 |
| 2 | **(1) volatile READ** → `8`. The read is correct, current, and barriered. It is genuinely the freshest value in the machine | — | `reservedUnits = 8` |
| 3 | **(2) COMPUTE** `8 + 1 = 9`, in a register on A's **private stack**. Nothing in shared memory has changed | — | `reservedUnits = 8`. A's intention to write 9 exists nowhere any other thread can see |
| 4 | **Preempted here.** The OS takes the core. A is descheduled between the read and the write | — | `reservedUnits = 8`. **No barrier helps: there is nothing to publish yet** |
| 5 | — | Enters `reserveOne()` for a reconciled in-flight reservation | `reservedUnits = 8` |
| 6 | — | **(1) volatile READ** → `8`. **This read is also completely correct.** The field really does still hold 8 | `reservedUnits = 8` |
| 7 | — | **(2) COMPUTE** `8 + 1 = 9` | `reservedUnits = 8` |
| 8 | — | **(3) volatile WRITE** `9`. `stlr` + `dmb ish` on aarch64. Store buffer drained. **Globally visible immediately and correctly** | **`reservedUnits = 9`.** Available = 1 |
| 9 | — | Returns. B has correctly recorded its reservation | `reservedUnits = 9` |
| 10 | **A resumes.** It does **not** re-read. It already holds `9` in a register, computed at step 3 from a read that was valid then | — | `reservedUnits = 9` |
| 11 | **(3) volatile WRITE** `9`. Fully barriered, fully visible, and **it overwrites B's 9 with an identical 9** | — | `reservedUnits = 9`. **B's increment has been erased by a write that is indistinguishable from it** |
| 12 | Returns. Order #90210 is accepted | — | `reservedUnits = 9`, but **two** reservations were made |
| 13 | Eight more orders arrive over the next minute; each increments correctly | Reconciler continues | `reservedUnits` reaches `10`. `orderflow` believes 0 units are available and stops accepting |
| 14 | — | — | **But 11 reservations exist.** The counter is one short of the truth, permanently |
| **15** | **OUTCOME, in business terms: `orderflow` accepted 11 reservations against 10 physical units of `SKU-4471`. One customer completed checkout, was charged, received a confirmation email, and will not receive a product. The in-memory counter reads a perfectly plausible `10`, matching neither the number of orders nor the physical stock. Nothing in the data is malformed. The oversell surfaces days later at the warehouse, is attributed to a picking error, and recurs every time a SKU goes hot. Every field involved is `volatile`, and every reviewer saw that and moved on.** | | |

### What to take from this trace

**Steps 2 and 6 are the point.** Both reads were correct. Both were barriered. Both
returned the genuinely freshest value in the entire machine. **`volatile` did its job
perfectly and the program is still wrong**, because the defect is in the *gap between*
step 2 and step 11, and `volatile` has nothing to say about gaps.

**Step 11 is why it is invisible.** A wrote `9`. B had already written `9`. The final
value is identical to the correct-looking one. **The data carries no evidence.** You find
it from the business — orders exceeding units — exactly as in Topic 84's trace.

**Step 4 is architecture-independent.** This is a preemption/interleaving bug, not a
reordering bug. It occurs on x86 and on aarch64 equally, and it occurs under `-Xint`
too — unlike Topic 86's, which `-Xint` cures. Note that difference; it is a diagnostic.

### The second bug in the same class, which this trace does not show

`releaseOne()` and `reserveOne()` on the same counter give you a second, subtler defect:
even with a *correct* atomic counter, `if (reserved() < capacity) reserveOne();` is
check-then-act across two atomic operations and races the same way. **Making each
operation atomic does not make a sequence of them atomic.** That is Topic 92's headline,
and it is why the correct fix at the end of this document is not `AtomicInteger` alone.

---

## Example 1 — minimal

The smallest program that shows the increment loss, and the smallest that shows what
`volatile` *does* buy.

```java
package com.orderflow.lab.jmm;

import java.util.concurrent.CountDownLatch;

/**
 * Two threads, each incrementing a VOLATILE int N times.
 *
 * Expected final value if increments were atomic:  2 * N
 * Actual final value:                              <= 2 * N, usually much less
 *
 * The gap is lost updates. `volatile` guarantees every read and every write is ordered
 * and visible. It guarantees NOTHING about the gap between the read and the write, and
 * `i++` is exactly that gap.
 *
 * NOTE: the numbers this prints are an observation from YOUR run on YOUR machine. They
 * are not a benchmark and must never be quoted as one. What matters is only whether the
 * total is less than 2*N, not by how much.
 */
public final class VolatileCounterLoss {

    private static volatile int reservedUnits = 0;

    private static final int PER_THREAD = 1_000_000;

    public static void main(String[] args) throws Exception {
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done  = new CountDownLatch(2);

        Runnable task = () -> {
            try { start.await(); } catch (InterruptedException e) { return; }
            for (int i = 0; i < PER_THREAD; i++) {
                reservedUnits++;              // read, add, write. THREE operations.
            }
            done.countDown();
        };

        new Thread(task, "order-placement").start();
        new Thread(task, "inventory-reconciler").start();

        start.countDown();                    // release both at once: maximum overlap
        done.await();                         // edge 5-equivalent: countDown hb await

        int expected = 2 * PER_THREAD;
        System.out.println("expected = " + expected);
        System.out.println("actual   = " + reservedUnits);
        System.out.println("LOST     = " + (expected - reservedUnits));
        System.out.println(reservedUnits == expected
            ? "no loss observed on THIS run (see Honesty rule 1 - not a proof)"
            : "increments were LOST. volatile did not make i++ atomic.");
    }
}
```

```bash
mkdir -p ~/java-lab/87/{src/com/orderflow/lab/jmm,out} && cd ~/java-lab/87
javac -d out src/com/orderflow/lab/jmm/VolatileCounterLoss.java

for i in 1 2 3 4 5; do java -cp out com.orderflow.lab.jmm.VolatileCounterLoss; echo "---"; done
for i in 1 2 3; do java -Xint -cp out com.orderflow.lab.jmm.VolatileCounterLoss; echo "---"; done
```

**WHAT TO LOOK FOR:** whether `actual` is less than `expected`, and whether `-Xint`
changes the answer.

| What you see | What it means |
|---|---|
| `actual` < `expected` on most or all runs | **The demonstration.** `volatile` made every access visible and ordered, and increments were still lost. Visibility is not atomicity. |
| Loss appears under `-Xint` too | **Expected, and diagnostically important.** Unlike Topic 86's bug, this one is not a compiler effect — it is a preemption/interleaving effect. `-Xint` does not cure it, and that difference is how you tell the two apart. |
| `actual == expected` on a run | The threads did not overlap in the window on that run. **Not evidence the code is correct.** Raise `PER_THREAD`, remove the `start` latch so they overlap less predictably, or run it twenty times. Then read Honesty rule 1. |
| The loss is much larger than you expected | Normal. Under sustained contention on one cache line, the two threads spend most of their time reading the same value. There is no "expected" loss rate; it is not a meaningful number. |
| Removing `volatile` entirely changes very little | Also expected, and worth noticing: the field was already being written so often that staleness had little room to matter. **`volatile` was solving a problem this program did not have, while the problem it did have went unaddressed.** That is the trap in one sentence. |

### What `volatile` *does* buy — the message-passing pair

```java
package com.orderflow.lab.jmm;

/**
 * The idiom `volatile` exists for. `payload` is a PLAIN field and is safely published
 * anyway, by transitivity through the volatile `ready`.
 *
 *   plain write hb volatile write   (program order, edge 1)
 *   volatile write hb volatile read (edge 3)
 *   volatile read hb plain read     (program order, edge 1)
 *   => plain write hb plain read    (transitivity, edge 10)
 *
 * Remove `volatile` from `ready` and the argument collapses at step 2. On aarch64 the
 * failure becomes observable; on x86-TSO it very likely does not, which is exactly why
 * an x86 test is weak evidence.
 */
public final class SafePublishByVolatile {

    private int[] priceTable;                    // PLAIN. Deliberately.
    private volatile boolean ready;              // The publishing field.

    /** Called once, by the catalogue warm-up thread. */
    public void publish(int[] freshPrices) {
        this.priceTable = freshPrices;           // ordinary write
        this.ready = true;                       // VOLATILE write: StoreStore before it
    }

    /** Called by every request thread. */
    public int priceOf(int index) {
        if (!ready) {                            // VOLATILE read: LoadLoad/LoadStore after
            return -1;
        }
        return priceTable[index];                // guaranteed to see freshPrices, fully written
    }
}
```

**State the guarantee out loud, because this is the sentence that separates people who
know `volatile` from people who have heard of it:**

> *One `volatile` write publishes everything the writing thread did before it, to any
> thread that reads that `volatile` field and sees the new value.*

**And state its precondition, because it is where people get hurt:** the reader must
actually read the volatile field, and must read it *before* the plain reads. Reordering
`priceTable[index]` above the `if (!ready)` check would break it, which is exactly what
the `LoadLoad` barrier forbids.

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
| Hot SKUs | ~50 SKUs take ~40% of order volume — maximum contention on a few counters |
| Baseline | `/docs/java/baselines/` — p50/p95/p99/p999 per endpoint |

### The code that ships

```java
package com.orderflow.wallet;

import org.springframework.stereotype.Component;

/**
 * In-memory wallet balance cache, in front of Postgres, so the order-placement path can
 * reject obviously-insufficient balances without a database round trip.
 *
 * THIS CLASS CONTAINS THE TOPIC 87 DEFECTS. It is here to be diagnosed, not copied.
 */
@Component
public class WalletBalanceCache {

    /** Keyed by wallet id. Made volatile after a visibility incident. */
    private final Map<Long, VolatileBalance> balances = new ConcurrentHashMap<>();

    static final class VolatileBalance {
        volatile long minorUnits;              // volatile: 64-bit access IS atomic. Good.
        volatile long lastUpdatedEpochMs;      // volatile too. Also good, in isolation.
    }

    /** DEFECT 1: compound operation on a volatile field. */
    public void debit(long walletId, long amountMinor) {
        VolatileBalance b = balances.get(walletId);
        b.minorUnits -= amountMinor;                       // read-modify-write
        b.lastUpdatedEpochMs = System.currentTimeMillis(); // a SECOND field
    }

    /** DEFECT 2: check-then-act across two separate volatile accesses. */
    public boolean tryDebit(long walletId, long amountMinor) {
        VolatileBalance b = balances.get(walletId);
        if (b.minorUnits >= amountMinor) {                 // CHECK  (volatile read)
            b.minorUnits -= amountMinor;                   // ACT    (read-modify-write)
            return true;
        }
        return false;
    }

    /** DEFECT 3: two fields read separately; no instant at which both were true. */
    public BalanceSnapshot snapshot(long walletId) {
        VolatileBalance b = balances.get(walletId);
        return new BalanceSnapshot(b.minorUnits, b.lastUpdatedEpochMs);
    }
}
```

**Three distinct defects, and you must be able to name each separately:**

| # | Defect | Why `volatile` does not help | The correct tool |
|---|---|---|---|
| 1 | `-=` is a read-modify-write | `volatile` orders the read and the write; the gap between them is unordered | `AtomicLong.addAndGet` (Topic 95) or a lock |
| 2 | check-then-act across two accesses | Each access is atomic; the *sequence* is not. Another thread debits in the gap | `compareAndSet` retry loop, or `synchronized`, or a database conditional UPDATE |
| 3 | two `volatile` fields read separately | Each read is atomic. There may be **no instant** at which both values were simultaneously current | One `volatile` reference to an **immutable pair** (a record) |

**Defect 3 is the one senior candidates get wrong**, so look at it carefully. Two
volatile reads give you two individually-correct values from two different instants. The
snapshot describes a state the wallet was never actually in. That is the "inconsistent
read of related fields" problem, and its fix is structural:

```java
// The record is immutable, so publishing the REFERENCE publishes a consistent pair.
record Balance(long minorUnits, long lastUpdatedEpochMs) {}

static final class Wallet {
    private volatile Balance balance = new Balance(0, 0);   // ONE volatile field
    Balance snapshot() { return balance; }                  // ONE read: always consistent
}
```

**One volatile read of one reference to an immutable object always gives a consistent
view.** This is the pattern to reach for whenever two or more fields must agree, and it
is Topic 88's material arriving early because it is genuinely the right answer here.

### What you observe, in the order you observe it

1. **Occasional customer complaints** — a wallet debited twice, a purchase completed with
   insufficient funds. Individually indistinguishable from support noise.
2. **Postgres and the cached balance diverge** on hot wallets. Attributed to cache
   staleness and "fixed" by shortening the refresh interval. It does not help.
3. **Reconciliation drifts only for high-volume wallets** — the ones with real contention.
   Low-volume wallets are perfect, which makes it look like a data problem.
4. **A code review finds nothing.** Every field is `volatile`; the map is a
   `ConcurrentHashMap`. Every box is ticked.
5. **It does not reproduce in a unit test** — two threads, a thousand iterations, idle
   machine, passes every time.
6. **It reproduces immediately at the Topic 65 load level**, because the hot-wallet
   distribution is what generates the contention.

### The diagnosis, as commands

There is no log for this and no flag. The diagnosis is a **test**:

```bash
# 1. Confirm the shape in bytecode: is the operation one access or three?
javap -c -p target/classes/com/orderflow/wallet/WalletBalanceCache.class \
  | sed -n '/debit/,/^$/p'

# 2. Encode the invariant as a jcstress test and run it. This is the real diagnosis.
cd ~/java-lab/87/orderflow-jcstress
mvn clean verify
java -jar target/jcstress.jar -t VolatileWalletDebit

# 3. Confirm what you ran it on. This is part of the result.
java --version ; uname -m
```

**WHAT TO LOOK FOR** in step 1: `getfield`, arithmetic, `putfield` — three separate
instructions, with the `volatile` modifier changing how each is compiled and not how many
there are.

| What you see | What it means |
|---|---|
| `getfield` / `lsub` / `putfield` for `minorUnits` | **Three operations.** The bytecode does not become atomic because the field is `volatile`. `javap` shows the same instruction sequence either way. |
| An `invokevirtual` to `AtomicLong.addAndGet` after the fix | One operation, implemented by a `lock cmpxchg` retry loop at the machine level (Topic 95). |
| `getfield` twice for `minorUnits` in `tryDebit` | **Two reads.** The check and the act read the field independently. That is defect 2, visible in the bytecode. |

### The fixes, in the order you should consider them

**Fix 1 — `AtomicLong` for the counter.**

```java
private final AtomicLong minorUnits = new AtomicLong();
public void debit(long amountMinor) { minorUnits.addAndGet(-amountMinor); }
```

Fixes defect 1. Does **not** fix defect 2 — `if (get() >= x) addAndGet(-x)` is still
check-then-act.

**Fix 2 — a CAS retry loop for the conditional debit.**

```java
public boolean tryDebit(long amountMinor) {
    long current, next;
    do {
        current = minorUnits.get();
        if (current < amountMinor) return false;   // insufficient funds
        next = current - amountMinor;
    } while (!minorUnits.compareAndSet(current, next));   // ATOMIC check-and-act
    return true;
}
```

`compareAndSet` fails and retries if anything changed since the read. This is the correct
lock-free conditional update, and Topic 95 is its full treatment.

**Fix 3 — one `volatile` reference to an immutable record, for the multi-field case.**

```java
record Balance(long minorUnits, long lastUpdatedEpochMs) {}

private final AtomicReference<Balance> balance = new AtomicReference<>(new Balance(0, 0));

public boolean tryDebit(long amountMinor) {
    Balance current, next;
    do {
        current = balance.get();
        if (current.minorUnits() < amountMinor) return false;
        next = new Balance(current.minorUnits() - amountMinor, System.currentTimeMillis());
    } while (!balance.compareAndSet(current, next));
    return true;
}
```

**This is the fix to reach for.** It solves all three defects at once: the update is
atomic, the check-and-act is atomic, and a reader gets a consistent pair because it reads
one reference to one immutable object. The cost is an allocation per successful debit,
which the Topic 68 TLAB work tells you is cheap and which you should nonetheless measure
against the baseline.

**Fix 4 — and the one an experienced engineer proposes first: do not cache the balance at
all.**

The wallet balance is money. `orderflow` already has an authoritative, transactional,
correctly-locked copy in Postgres, with `@Version` optimistic locking from Topic 52. An
in-memory cache in front of it is a second source of truth for a value that must not have
two sources of truth. **The best fix for a memory-model bug is frequently to delete the
shared mutable state rather than synchronise it.** The correct implementation of "reject
insufficient balance without a round trip" is a conditional UPDATE with a `WHERE
balance >= ?` clause — one statement, atomic by the database, no JMM involved.

Keep the cache only if you have measured that the round trip matters, and then treat it
explicitly as a *hint* that may be wrong, never as a decision input for money.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `volatile int i; i++` losing increments

**Wrong:**

```java
private volatile int reservedUnits;
public void reserveOne() { reservedUnits++; }
```

**Exact symptom:**

- The final count is **lower** than the number of calls, and the shortfall grows with
  thread count and with contention.
- **No exception, no log, no malformed data.** The value is plausible.
- Does not reproduce in a unit test with two threads and a thousand iterations on an idle
  machine. Reproduces immediately at the Topic 65 load level.
- The discrepancy surfaces days later as a business-level inconsistency — orders
  exceeding units, a balance that does not match the ledger.
- **Reproduces under `-Xint` as well.** That is the diagnostic difference from Topic 86.

**Root cause:** `i++` is `getfield`, `iadd`, `putfield` — three bytecodes, two gaps.
`volatile` gives each of the three accesses ordering and visibility. It says nothing
about the gaps, and the OS scheduler may preempt in either one (Topic 84). Two threads
read the same value, both add one, both write the same result: one increment is erased by
a write indistinguishable from it.

**This is not a reordering bug**, so it is not architecture-sensitive: it occurs on x86
and aarch64 alike. It is an interleaving bug.

**Fix:** `AtomicInteger`/`AtomicLong` with `incrementAndGet` — one atomic
read-modify-write, a `lock xadd` or `lock cmpxchg` loop at the machine level (Topic 95).
Or `LongAdder` when writes dominate and reads are occasional, because it stripes across
cells and avoids the cache-line ping-pong. Or a lock, if more than one field must change
together. **And test it with jcstress rather than asserting it** — the drill below is
exactly this.

---

### Trap 2 — "we made it `volatile`, so it's thread-safe now"

**Wrong:** the belief, applied as a blanket policy. Every shared field in a class gets
`volatile`, the class is declared thread-safe in a Javadoc comment, and review moves on.

**Exact symptom:**

- Fields are `volatile` and the class is still wrong, in ways that look nothing like a
  visibility bug: lost updates, impossible states, decisions made on stale-by-one-step
  data.
- Reviewers see `volatile` as evidence of care and stop reading. **The keyword actively
  reduces scrutiny**, which is what makes this trap expensive.
- The class name often contains "Safe", "Concurrent" or "Atomic" and none of those are
  true.
- Throughput is worse than before, because every write now carries a `StoreLoad` barrier
  and a cache-line invalidation, for a correctness benefit that was never achieved.

**Root cause:** `volatile` guarantees **visibility and ordering of individual accesses**.
Thread safety requires **atomicity of operations** and **consistency of invariants across
fields**. Those are different properties, and `volatile` supplies exactly one of them.

The blanket-policy version has a second cost worth naming: **it is a real performance
regression bought for nothing.** Every volatile write costs a barrier and a coherence
round; on a field that is never read by another thread, that is pure loss.

**Fix:** apply the one-line test — *is every operation on this field a single read or a
single write?* If not, `volatile` is the wrong tool. Ask **what the invariant is**, not
what the field is: if the invariant spans two fields, no per-field mechanism can hold it,
and you need one lock or one immutable object. Reserve `volatile` for flags,
safely-published references, and the double-checked-locking `instance` field. **And
review it as a claim requiring evidence**, not as a marker of diligence: the reviewer's
question is "which operation is being made atomic here, and by what?"

---

### Trap 3 — double-checked locking without `volatile`

**Wrong:**

```java
public class PricingEngineHolder {
    private static PricingEngine instance;              // NOT volatile

    public static PricingEngine get() {
        if (instance == null) {                          // check 1, unlocked
            synchronized (PricingEngineHolder.class) {
                if (instance == null) {                  // check 2, locked
                    instance = new PricingEngine();      // <-- the bug
                }
            }
        }
        return instance;
    }
}
```

**Exact symptom:**

- Intermittently, a thread receives a `PricingEngine` whose fields are still `null` or
  `0`. The `NullPointerException` comes from *inside* the engine, on a field the
  constructor demonstrably sets.
- Every reviewer reads the constructor, sees the field being assigned, and concludes the
  NPE is impossible. It is dismissed as flaky.
- Concentrated at the **start of a run**, when many threads race to initialise for the
  first time — which is precisely when nobody is watching, because "it's just warming up".
- **Far more likely to be observed on aarch64 than on x86**, because it depends on a
  `StoreStore` reordering that x86-TSO forbids outright.

**Root cause:** `instance = new PricingEngine()` is three operations: allocate memory; run
the constructor, initialising fields; write the reference into `instance`. **The JIT and
the CPU may reorder the second and third**, because within the writing thread the
reordering is unobservable, and the JMM only constrains what is observable across threads
where an edge exists. A thread executing check 1 with **no lock held** can therefore
observe a non-null `instance` pointing at an object whose constructor has not finished.

Note carefully: the `synchronized` block is correct and does its job. The bug is entirely
in the **unlocked first check**, which is the whole point of the idiom.

**Fix — in order of preference:**

```java
// 1. BEST for a static singleton: the holder idiom. Class initialisation is guaranteed
//    thread-safe and lazy by the JVM itself (Topic 67). You write no concurrency code.
public class PricingEngineHolder {
    private static class Holder { static final PricingEngine INSTANCE = new PricingEngine(); }
    public static PricingEngine get() { return Holder.INSTANCE; }
}

// 2. If it must be an instance field: volatile. This is the ONLY thing that makes
//    double-checked locking correct, and has been since Java 5.
private volatile PricingEngine instance;

// 3. In Spring: declare it a @Bean and let the container do it once, eagerly.
```

**Why `volatile` fixes it, precisely:** the `StoreStore` barrier *before* the volatile
write forbids the reference store from being reordered before the constructor's field
stores, and the `LoadLoad`/`LoadStore` after the reading thread's volatile read forbids
its reads of the object's fields from being hoisted above the null check. **Both halves
are required, which is why both the write and the read must go through the same volatile
field.**

Topic 85 raised this trap and deferred the explanation here and to Topic 88; **Topic 88
adds the remarkable fact that making every field of `PricingEngine` `final` also fixes
it**, via the constructor freeze, with no `volatile` at all.

---

### Trap 4 — `volatile` on a reference, expecting it to cover the contents

**Wrong:**

```java
private volatile List<String> hotSkus = new ArrayList<>();

public void addHotSku(String sku) { hotSkus.add(sku); }     // no volatile write here
public List<String> current()     { return hotSkus; }        // hands out the live list
```

or the array form:

```java
private volatile int[] reserved = new int[100];
public void reserveOne(int idx) { reserved[idx]++; }        // not a volatile op at all
```

**Exact symptom:**

- Readers see a partially-updated list: an element missing, an element twice, or a size
  that does not match the contents.
- `ArrayList` mutated concurrently can throw `ArrayIndexOutOfBoundsException` from
  `add`, return `null` from `get`, or silently lose elements during a grow.
- `ConcurrentModificationException` from a reader iterating while a writer adds — and
  **that is the lucky case**, because it is loud.
- The array version loses increments exactly as Trap 1 does, and no one notices that
  `volatile` was never involved.

**Root cause:** **`volatile` is a property of the field, not of the object graph the
field reaches.** `hotSkus.add(...)` does not write the field `hotSkus` — the reference is
unchanged — so no barrier is emitted and no edge is established. `reserved[idx]++` is an
`aaload`/`iaload` on the array object; the field `reserved` is never touched.

**Fix:**

```java
// 1. Immutable snapshot + volatile reference. Readers are lock-free and consistent.
private volatile List<String> hotSkus = List.of();
public void refresh(Collection<String> fresh) { this.hotSkus = List.copyOf(fresh); }

// 2. Per-element atomicity on an array:
private final AtomicIntegerArray reserved = new AtomicIntegerArray(100);

// 3. A genuinely concurrent collection (Topic 92):
private final Map<String, Integer> reserved = new ConcurrentHashMap<>();
```

Option 1 is the right shape for a read-mostly registry: **one volatile write of one
immutable object**, which publishes the whole thing by transitivity. Note that
`List.copyOf` returns an immutable list, which matters — handing out a mutable list
reference re-opens the hole, which is Topic 88's Trap 1 and Topic 17's defensive copying.

---

### Trap 5 — using `volatile` where two fields must agree

**Wrong:**

```java
private volatile long balanceMinor;
private volatile long reservedMinor;

public long availableMinor() {
    return balanceMinor - reservedMinor;    // two separate volatile reads
}
```

**Exact symptom:**

- `availableMinor()` occasionally returns a value that was never true — a negative
  available balance, or an available balance larger than the total.
- Every individual read is correct. Every individual write is correct. **There is simply
  no instant at which both values were simultaneously current.**
- Reproduces only under real concurrency; a test that reads while nothing writes always
  passes.
- Downstream logic that treats a negative available balance as impossible throws, or
  worse, silently clamps.

**Root cause:** `volatile` provides **per-field** atomicity and ordering. It provides no
mechanism whatsoever for reading two fields as one operation. A writer can update
`balanceMinor` between the reader's two reads, and the reader combines a new value with
an old one. There is no barrier that fixes this, because nothing is being reordered — the
reads are correctly ordered and describe two different points in time.

**Fix:**

```java
// Combine the fields into ONE immutable object behind ONE volatile reference.
record Funds(long balanceMinor, long reservedMinor) {
    long availableMinor() { return balanceMinor - reservedMinor; }
}
private volatile Funds funds = new Funds(0, 0);

public long availableMinor() { return funds.availableMinor(); }   // ONE read
```

**The rule to carry:** *when N fields must be consistent with each other, the unit of
publication must be one thing, not N things.* Either one immutable object behind one
volatile reference or one `AtomicReference`, or one lock covering all N. This is the same
argument as Topic 52's atomic conditional UPDATE and Topic 92's `compute`: **atomicity is
a property of an operation, and you get to choose what the operation is.**

---

### Trap 6 — adding `volatile` everywhere and paying for it

**Wrong:** a blanket policy — every field in every shared class gets `volatile`, on the
principle that it cannot hurt.

**Exact symptom:**

- Throughput drops measurably against the Topic 65 baseline, with no change in
  correctness, because none of the bugs were visibility bugs.
- The regression is **worse on aarch64 than on x86**, because on x86 the volatile *read*
  was free and on your Mac it is not. A change that measured as neutral on an x86 CI
  benchmark shows a real cost on the developer laptop, or vice versa.
- Allocation rate and GC are unchanged, so the usual investigation finds nothing.
- Profiling shows time spread thinly across every write path rather than concentrated
  anywhere, so a flame graph does not point at it.

**Root cause:** every volatile write emits a `StoreLoad` barrier, which drains the store
buffer and stalls the pipeline, and forces the cache line into `M` state, invalidating
every other core's copy. On a field written frequently by multiple threads, that is a
line in permanent migration between cores — the same physics as Topic 95's contended CAS
and Topic 96's false sharing. **The cost is not the instruction, it is the coherence
traffic the instruction makes unavoidable.**

**Fix:** apply `volatile` **per field, for a stated reason**, and write the reason down.
Ask the read/write ratio: a flag written once and read constantly is nearly free (the
line settles into `S` state everywhere and stays); a field written by eight threads is
expensive. **Measure on the target architecture with JMH `@Threads`** (Topic 77) — an x86
number does not transfer to aarch64 and vice versa. And prefer a design where the shared
mutable state does not exist: an immutable snapshot published once beats a mutable object
synchronised carefully, every time.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call
it captured.

### Setup

```bash
mkdir -p ~/java-lab/87/{src/com/orderflow/lab/jmm,out} && cd ~/java-lab/87
java --version                # record it
uname -m                      # expect arm64 on Apple Silicon
sysctl -n hw.ncpu             # performance + efficiency cores
```

Write all three down. They are part of every result you record.

### Proof 1 — `volatile i++` is still three bytecodes

```bash
javac -d out src/com/orderflow/lab/jmm/VolatileCounterLoss.java
javap -c -p -cp out com.orderflow.lab.jmm.VolatileCounterLoss | sed -n '/lambda/,/^$/p'
javap -v -p -cp out com.orderflow.lab.jmm.VolatileCounterLoss | grep -A3 reservedUnits
```

**WHAT TO LOOK FOR:** `getstatic`, `iconst_1`, `iadd`, `putstatic` — four instructions
for one `++` — and the `ACC_VOLATILE` flag on the field declaration.

| What you see | What it means |
|---|---|
| `getstatic` / `iadd` / `putstatic`, and `ACC_VOLATILE` on the field | **The proof, in one screen.** The field is marked volatile *and* the operation is still three separate accesses. `volatile` changed how each is compiled, not how many there are. |
| Identical bytecode with and without `volatile` | **Correct and expected.** The difference is entirely in the field's access flags and in the machine code the JIT emits. javac does not change the instruction sequence. |
| An `invokevirtual AtomicInteger.incrementAndGet` after the fix | **One** operation. That is the structural difference, visible at the bytecode level. |

### Proof 2 — predict the cost asymmetry before you measure it

Before writing the JMH benchmark in the Measurement section, write down two predictions:
which of *volatile read* / *volatile write* costs more on aarch64, and whether your
answer would differ on x86. **Predicting first is the exercise**; the number is secondary
and is worthless without a confidence interval.

### Proof 3 — get a jcstress project

```bash
mkdir -p ~/java-lab/87 && cd ~/java-lab/87

mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jcstress \
  -DarchetypeArtifactId=jcstress-java-test-archetype \
  -DgroupId=com.orderflow \
  -DartifactId=orderflow-jcstress \
  -Dversion=1.0

cd orderflow-jcstress
grep -A2 jcstress pom.xml     # see the version the archetype chose
java --version
uname -m
```

> **Version note.** I have deliberately not pinned a jcstress version. The archetype and
> `jcstress-core` move independently of the JDK, and any version I quote will be stale.
> Take the current version from the jcstress project README
> (`github.com/openjdk/jcstress`) and pin it in your POM; if the archetype coordinates
> have changed, the README is also where that will be said. **Do not copy a version
> number out of a teaching document.**

Write down the jcstress version, the JDK version, and `uname -m`. **All three are part of
every result**, and a result without them is not reportable.

### Proof 4 — the lost-increment test (the R16 test for this topic)

`src/main/java/com/orderflow/VolatileReservationIncrement.java`:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.Actor;
import org.openjdk.jcstress.annotations.Arbiter;
import org.openjdk.jcstress.annotations.Description;
import org.openjdk.jcstress.annotations.JCStressTest;
import org.openjdk.jcstress.annotations.Outcome;
import org.openjdk.jcstress.annotations.State;
import org.openjdk.jcstress.infra.results.I_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE_INTERESTING;

/**
 * orderflow: two threads reserve one unit each of the same hot SKU.
 *
 * The counter is VOLATILE. Every read and every write is ordered and visible.
 * The operation is still a read-modify-write, so an increment can be LOST.
 *
 * Expected outcomes:
 *   2 - both reservations recorded. Correct.
 *   1 - ONE RESERVATION LOST. orderflow believes one unit is reserved when two are.
 *       This is how a hot SKU gets oversold.
 */
@JCStressTest
@Description("volatile int reservedUnits, two concurrent increments. "
           + "volatile gives visibility and ordering, not atomicity.")
@Outcome(id = "2", expect = ACCEPTABLE,
         desc = "Both reservations landed. The actors did not overlap in the window.")
@Outcome(id = "1", expect = ACCEPTABLE_INTERESTING,
         desc = "ONE RESERVATION WAS LOST. Both actors read the same value, both "
              + "computed the same result, both wrote it. orderflow oversells.")
@State
public class VolatileReservationIncrement {

    volatile int reservedUnits;

    @Actor
    public void orderPlacement() {
        reservedUnits++;                 // read, add, write -- THREE operations
    }

    @Actor
    public void inventoryReconciler() {
        reservedUnits++;
    }

    @Arbiter
    public void readFinal(I_Result r) {
        r.r1 = reservedUnits;            // runs after both actors, with edges from both
    }
}
```

And the fixed version, in the same project, so you can compare the outcome tables
side by side:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.I_Result;
import java.util.concurrent.atomic.AtomicInteger;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.FORBIDDEN;

/**
 * The same scenario with an ATOMIC read-modify-write instead of a volatile one.
 *
 * `1` is declared FORBIDDEN. Read the honesty rule below before you interpret a
 * FORBIDDEN row with zero samples.
 */
@JCStressTest
@Description("AtomicInteger.incrementAndGet: one atomic read-modify-write per actor.")
@Outcome(id = "2", expect = ACCEPTABLE,
         desc = "Both reservations landed. The only outcome the happens-before "
              + "argument permits.")
@Outcome(id = "1", expect = FORBIDDEN,
         desc = "A reservation was lost. incrementAndGet is atomic, so this outcome "
              + "would mean the happens-before argument is wrong.")
@State
public class AtomicReservationIncrement {

    final AtomicInteger reservedUnits = new AtomicInteger();

    @Actor
    public void orderPlacement()      { reservedUnits.incrementAndGet(); }

    @Actor
    public void inventoryReconciler() { reservedUnits.incrementAndGet(); }

    @Arbiter
    public void readFinal(I_Result r) { r.r1 = reservedUnits.get(); }
}
```

Read what each part is doing:

| Element | Why it is there |
|---|---|
| `@JCStressTest` | Marks the class for the annotation processor. Without `mvn verify` the processor does not run and nothing is generated. |
| `@State` on the class | The class **is** the state. jcstress allocates many thousands of instances and runs the actor pair against each. |
| `volatile int reservedUnits` | The subject. Deliberately volatile, to isolate "visibility is not atomicity" from Topic 86's visibility problem. |
| Two `@Actor` methods | Two threads. **jcstress decides the scheduling; you do not.** It also varies affinity and JIT mode across configurations. |
| `@Arbiter` taking `I_Result` | Runs **after** both actors, with happens-before edges from both, so it reads the settled value. Without it you would be racing the reader too. |
| `id = "1"` graded `ACCEPTABLE_INTERESTING` | It is legal — that is the complaint — but it is what you are hunting, so the report should shout about it. |
| `id = "1"` graded `FORBIDDEN` in the atomic version | Declares the correctness claim. If jcstress ever observes it, **the test fails and your argument was wrong.** |
| No `id = "0"` declared anywhere | The arbiter cannot see 0. If it somehow did, the run fails as `UNKNOWN` — correct behaviour: an outcome you did not think about must not pass silently. |

### Build and run

```bash
mvn clean verify                             # NOT `mvn compile` - the processor must run

java -jar target/jcstress.jar -h             # do this once; the CLI has changed across versions
java -jar target/jcstress.jar -l             # list the tests it found

java -jar target/jcstress.jar -t VolatileReservationIncrement
java -jar target/jcstress.jar -t AtomicReservationIncrement

# Useful, but CONFIRM against your version with -h first:
java -jar target/jcstress.jar -t Reservation -m quick      # shorter run
java -jar target/jcstress.jar -t Reservation -v            # verbose per-config output
java -jar target/jcstress.jar -t Reservation -r results/   # HTML report location
```

### Reading the outcome table

jcstress prints a table per test configuration. Here is its **column structure**:

```
RESULT      SAMPLES     FREQ       EXPECT  DESCRIPTION
     1          <n>      xxx%  INTERESTING  ONE RESERVATION WAS LOST.
     2          <n>      xxx%   ACCEPTABLE  Both reservations landed.
```

*illustration of the format, not captured output*

**I have no JVM, so `<n>` and `xxx%` are placeholders.** I am not going to invent sample
counts: a plausible-looking number would teach you to expect a particular frequency, and
the frequency is exactly the thing that varies by machine, JDK, core type and
architecture.

| Column | What it is |
|---|---|
| `RESULT` | The result tuple, formatted the same way as your `@Outcome` `id` |
| `SAMPLES` | How many times this exact outcome was observed across the whole run |
| `FREQ` | That count as a percentage of all observations. **Not a performance number and not a production probability** |
| `EXPECT` | The grade you assigned. `UNKNOWN` means you did not declare it, and the test fails |
| `DESCRIPTION` | Your `desc` text, echoed back. Write these for a reader who is not you |

### How to read what you get

| What you see | What it means |
|---|---|
| `VolatileReservationIncrement`: both `1` and `2` appear, `2` far more common | **The expected shape.** You have directly observed a lost update on a `volatile` field. Record it next to your JDK version, jcstress version and `uname -m`. |
| `VolatileReservationIncrement`: only `2`, zero samples of `1` | The window was not hit. **This is NOT evidence that `volatile i++` is atomic.** Try `-m stress`, close other applications, and re-read Honesty rule 1 below. |
| `AtomicReservationIncrement`: only `2`, `1` has zero samples | **The outcome you want, read correctly:** "not observed on this machine, this JDK, this run". It is **failure to falsify** the happens-before argument, which is a real result and is not proof. |
| `AtomicReservationIncrement`: `1` observed even once | **Stop.** Either the test is wrong, or your understanding of `incrementAndGet` is wrong, or you have found something genuinely remarkable. In that order of likelihood. Re-read the test first. |
| An outcome you did not declare, marked `UNKNOWN` | The test fails, correctly. Work out how that outcome is reachable **before** you declare it — the thinking is the point of the tool. |
| The build errors before any test runs | The annotation processor did not run. Use `mvn clean verify`, not `mvn compile`, and confirm `target/jcstress.jar` exists. |
| Wildly different results between two runs on the same machine | **Normal and important.** jcstress's outcome frequencies are not stable quantities. Never report one run's `FREQ` as a property of the code. |

### Proof 5 — the message-passing test, where architecture decides

This is the test where your Mac earns its keep. It probes a **reordering** rather than an
interleaving, so x86-TSO and aarch64 give genuinely different experiments.

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.II_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE_INTERESTING;

/**
 * orderflow catalogue warm-up publishing a price table with PLAIN fields.
 *
 * Writer:  priceTable = 42;  ready = 1;      (both PLAIN - no barrier between them)
 * Reader:  r1 = ready;       r2 = priceTable;
 *
 * The dangerous outcome is r1=1, r2=0: the reader saw the READY flag but NOT the data
 * it was supposed to signal. That requires either a StoreStore reordering by the writer
 * or a LoadLoad reordering by the reader.
 *
 * x86-TSO FORBIDS both of those in hardware, so on x86 this is very unlikely to be
 * observed and the only remaining source would be the compiler.
 * aarch64 PERMITS both. On Apple Silicon this test is a real experiment.
 *
 * Make `ready` volatile and the outcome becomes impossible by the happens-before
 * argument - which is the second half of this drill.
 */
@JCStressTest
@Description("Plain-field message passing. r1=1,r2=0 means the flag arrived without "
           + "the data. Architecture-sensitive: aarch64 exposes what x86-TSO hides.")
@Outcome(id = "0, 0", expect = ACCEPTABLE,
         desc = "Reader ran entirely before the writer. Nothing published yet.")
@Outcome(id = "0, 42", expect = ACCEPTABLE,
         desc = "Data seen, flag not yet. Harmless: the reader would not use the data.")
@Outcome(id = "1, 42", expect = ACCEPTABLE,
         desc = "Flag and data both seen. The intended case.")
@Outcome(id = "1, 0", expect = ACCEPTABLE_INTERESTING,
         desc = "FLAG WITHOUT DATA. orderflow serves a price of 0 for every product. "
              + "This is the reordering the volatile barriers exist to forbid.")
@State
public class PlainPricePublication {

    int priceTable;         // PLAIN. Deliberately.
    int ready;              // PLAIN. Deliberately.

    @Actor
    public void catalogueWarmUp() {
        priceTable = 42;    // ordinary store
        ready = 1;          // ordinary store - NOTHING orders these two
    }

    @Actor
    public void requestThread(II_Result r) {
        r.r1 = ready;       // ordinary load
        r.r2 = priceTable;  // ordinary load - NOTHING orders these two either
    }
}
```

Then the corrected version — identical except for one keyword — with the bad outcome
declared `FORBIDDEN`:

```java
@JCStressTest
@Description("Same publication with a VOLATILE flag. StoreStore before the write and "
           + "LoadLoad after the read make 1,0 unreachable by happens-before.")
@Outcome(id = "0, 0",  expect = ACCEPTABLE, desc = "Reader ran first.")
@Outcome(id = "0, 42", expect = ACCEPTABLE, desc = "Data seen, flag not yet. Harmless.")
@Outcome(id = "1, 42", expect = ACCEPTABLE, desc = "Flag and data both seen. Intended.")
@Outcome(id = "1, 0",  expect = FORBIDDEN,
         desc = "FLAG WITHOUT DATA. Forbidden by the volatile happens-before edge. "
              + "Observing this would mean the argument is wrong.")
@State
public class VolatilePricePublication {

    int priceTable;                 // still PLAIN - and safely published anyway
    volatile int ready;             // THE ONLY CHANGE

    @Actor
    public void catalogueWarmUp() {
        priceTable = 42;
        ready = 1;                  // volatile write: StoreStore before, StoreLoad after
    }

    @Actor
    public void requestThread(II_Result r) {
        r.r1 = ready;               // volatile read: LoadLoad/LoadStore after
        r.r2 = priceTable;
    }
}
```

```bash
java -jar target/jcstress.jar -t PlainPricePublication
java -jar target/jcstress.jar -t VolatilePricePublication
```

**WHAT TO LOOK FOR:** whether `1, 0` appears in the plain test, and whether it is absent
from the volatile test.

| What you see | What it means |
|---|---|
| Plain test: `1, 0` observed on your Mac | **You have directly observed hardware memory reordering.** This is the result that x86 would very likely not produce. Record it with `uname -m` — it is an aarch64 result. |
| Plain test: `1, 0` never observed | **Legitimate.** Try `-m stress`, close other applications, run it again. And note honestly: on macOS the scheduler may place the two actors on cores that do not expose the window. **This is not evidence the plain version is correct.** |
| Volatile test: `1, 0` has zero samples | **The result you want, read correctly.** It means "not observed on this machine, this JDK, this run" — **not** "proven impossible". The proof is the happens-before argument: plain write hb volatile write hb volatile read hb plain read. jcstress is what catches that argument being wrong. |
| Volatile test: `1, 0` observed | **Stop everything.** Re-read the test for a mistake first. If the test is right, you would be looking at a JVM bug, which is possible and vanishingly rare. |
| Both tests behave identically | Either the plain window was never hit, or your JDK is emitting barriers you did not ask for. Try more iterations before concluding anything. |
| A colleague on x86 reports different results | **Expected, and the lesson.** x86-TSO forbids three of the four reorderings. Their clean run says almost nothing about your architecture. |

---

## Failure drill

**Mandatory.** Do not read the "how to read it" tables until you have produced the result
yourself and written down what you saw.

### The assignment, restated from the master plan

> Write the interleaving trace for `volatile` `i++` losing an increment, then encode it
> as a jcstress test and get the actual observed-outcomes table.
> Capture: the jcstress outcome table. The fix proves: **visibility is not atomicity.**

### The two honesty rules that govern this drill

> **Honesty rule 1 — a `FORBIDDEN` outcome with zero observations means "not observed on
> this machine, this JDK, this run". It does NOT mean "proven impossible."** jcstress
> explores interleavings aggressively; it does not enumerate the state space. **The proof
> of correctness is the happens-before argument. jcstress is how you catch the argument
> being wrong.** A clean run is *failure to falsify*, which is a real and useful result,
> and is not a proof.

> **Honesty rule 2 — architecture, and which way it cuts for each test in this drill.**
> x86 implements Total Store Order and hides a whole class of reordering bugs that
> aarch64 (your Apple Silicon Mac) exposes.
> - For the **increment test** (`VolatileReservationIncrement`): this is an
>   **interleaving** bug, not a reordering bug. **Architecture makes little or no
>   difference**; it is observable on both, and under `-Xint` too.
> - For the **publication test** (`PlainPricePublication`): this is a **reordering** bug.
>   **Your Mac is the better instrument, and an x86 CI run is close to worthless as
>   evidence.**
> - **And I will never tell you either one "will" reproduce.** Every table below has a
>   "you saw nothing" row, and that row is a legitimate outcome.

### Step 0 — the control

```bash
java --version
uname -m
sysctl -n hw.ncpu
cd ~/java-lab/87/orderflow-jcstress && grep -A2 jcstress pom.xml
```

Record all four. Then run the `orderflow` Topic 65 baseline unchanged and confirm the
percentiles are within ±10% of `/docs/java/baselines/`. **If they are not, stop** — the
Topic 65 gate rule applies to every drill in this phase.

### Step 1 — write the trace before you write the test

**Do this on paper, before touching the keyboard.** Reproduce the two-column interleaving
from the Concurrency trace section, from memory, for `volatile int reservedUnits` with two
threads incrementing. Your trace must include:

- The three separate operations of `++`, labelled.
- The exact step at which thread A is preempted.
- The observation that **both reads are correct**.
- The observation that A's final write is **indistinguishable** from B's.
- A final row stating the outcome **in business terms**.

**If you cannot write this trace unaided, do not proceed to the test.** The test is a
falsifier for an argument; you must have the argument first.

### Step 2 — build and run both tests

Follow Proof 3 and Proof 4 above. Run:

```bash
mvn clean verify
java -jar target/jcstress.jar -t VolatileReservationIncrement | tee ../results-volatile.txt
java -jar target/jcstress.jar -t AtomicReservationIncrement   | tee ../results-atomic.txt
java -jar target/jcstress.jar -t PlainPricePublication        | tee ../results-plain-pub.txt
java -jar target/jcstress.jar -t VolatilePricePublication     | tee ../results-vol-pub.txt
```

### Step 3 — what to capture

Write these down **before** reading any interpretation:

1. For `VolatileReservationIncrement`: did `1` appear? With what `SAMPLES` and `FREQ`?
2. For `AtomicReservationIncrement`: did `1` appear at all? (It should not.)
3. For `PlainPricePublication`: did `1, 0` appear? **This is the aarch64 result.**
4. For `VolatilePricePublication`: is `1, 0` absent?
5. Your JDK version, jcstress version, and `uname -m`. **A result without these is not
   reportable.**
6. How long each run took, and how many configurations jcstress reported per test.
7. Run the increment test **three times** and record whether the `FREQ` for `1` is
   stable. (It will not be.)

### Step 4 — how to read it

| What you see | What it means |
|---|---|
| `VolatileReservationIncrement` shows both `1` and `2` | **The drill has fired.** Write the sentence: *"the field was `volatile`, every read and every write was ordered and visible, and an increment was still lost — because `volatile` orders accesses and `i++` is three of them."* |
| `AtomicReservationIncrement` shows only `2`, `1` at zero | **Read this correctly**: not observed here, not proven impossible. The **proof** is that `incrementAndGet` is a single atomic read-modify-write. jcstress would have caught you if that claim were false. |
| `PlainPricePublication` shows `1, 0` | **You have observed hardware memory reordering on your own machine.** Label the result `aarch64` and note that an x86 run would very likely not show it. |
| `VolatilePricePublication` shows no `1, 0` | The barriers are doing their job. Same honesty rule: failure to falsify, not proof. |
| `1` never appears in the increment test | **Legitimate.** Try `-m stress`, close everything else, run it ten more times. Then record the honest result and move on — Honesty rule 1 says this is not an acquittal. |
| `FREQ` differs substantially between three runs of the same test | **Expected, and the point.** `FREQ` is an outcome frequency under jcstress's own scheduling, not a production probability. **Never put a jcstress `FREQ` into a risk assessment.** |
| A `FORBIDDEN` outcome is observed | The test fails, correctly, and something you believed is wrong. Re-read your test first, then your argument, then (rarely) suspect the JVM. |
| An `UNKNOWN` outcome appears | You did not declare it. **Work out how it is reachable before declaring it.** That reasoning is the entire value of the tool. |
| The increment test also loses updates under `-Xint` | **Correct and diagnostically important.** Unlike Topic 86's bug, this is not a compiler effect. That difference is how you distinguish the two families in production. |

### Step 5 — the `orderflow` version

Take the real defect from Example 2 and encode it:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.ZZ_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE_INTERESTING;

/**
 * orderflow wallet: check-then-act on a volatile balance.
 *
 * A wallet holds 100 minor units. Two concurrent debits of 100 each.
 * At most ONE should succeed.
 *
 * Outcome true,true means BOTH succeeded: 200 minor units were debited from a wallet
 * holding 100. orderflow gave away money.
 */
@JCStressTest
@Description("volatile long balance, check-then-act debit. Two debits of the full "
           + "balance; at most one may succeed.")
@Outcome(id = "true, false", expect = ACCEPTABLE, desc = "Only the first debit succeeded.")
@Outcome(id = "false, true", expect = ACCEPTABLE, desc = "Only the second debit succeeded.")
@Outcome(id = "false, false", expect = ACCEPTABLE_INTERESTING,
         desc = "Neither succeeded. Possible only if the balance was already reduced; "
              + "declare it and work out whether it is reachable in YOUR version.")
@Outcome(id = "true, true", expect = ACCEPTABLE_INTERESTING,
         desc = "BOTH DEBITS SUCCEEDED. 200 minor units taken from a 100-unit wallet. "
              + "orderflow lost money. Check-then-act is not atomic.")
@State
public class VolatileWalletDebit {

    volatile long balanceMinor = 100;

    private boolean tryDebit(long amount) {
        if (balanceMinor >= amount) {       // CHECK  - volatile read
            balanceMinor -= amount;         // ACT    - volatile read-modify-write
            return true;
        }
        return false;
    }

    @Actor
    public void firstOrder(ZZ_Result r)  { r.r1 = tryDebit(100); }

    @Actor
    public void secondOrder(ZZ_Result r) { r.r2 = tryDebit(100); }
}
```

Then write the corrected version with an `AtomicLong` and a `compareAndSet` retry loop
(Fix 2 from Example 2), declare `"true, true"` **`FORBIDDEN`**, and run both.

**WHAT TO LOOK FOR:** `true, true` in the volatile version, and its absence from the CAS
version.

| What you see | What it means |
|---|---|
| `true, true` observed in the volatile version | **The business bug, reproduced under laboratory conditions.** Two orders, one wallet's worth of money, both accepted. You can now put a jcstress table in a bug report instead of a hypothesis. |
| `true, true` never observed | **Legitimate.** The window is narrow. `-m stress`, more runs. **Not an acquittal** — the code is still wrong by the happens-before and atomicity argument. |
| CAS version: `true, true` has zero samples | Failure to falsify. The proof is that `compareAndSet` fails and retries if anything changed since the read. |
| CAS version: `true, true` observed | Re-read your retry loop. The most common mistake is re-reading `get()` inside the loop *after* computing `next`, which reopens the window. |
| `false, false` appears | Work out whether it is reachable in your version before you accept it. In this exact code it should not be; if it appears, you have a bug in the test. **That reasoning is the exercise.** |

### Step 6 — fix it, and re-measure the cost

Apply the three fixes from Example 2 and, for each, run **both** the jcstress test and a
JMH `@Threads` benchmark (Measurement section) so you have correctness *and* cost:

| Version | jcstress: is the bad outcome observed? | JMH: throughput at 1 / 2 / 8 threads |
|---|---|---|
| `volatile long` + `-=` | | |
| `AtomicLong.addAndGet` | | |
| `AtomicLong` + CAS retry loop | | |
| `AtomicReference<Balance>` + CAS | | |
| `synchronized` method | | |
| **No cache at all** (conditional SQL UPDATE) | n/a — the database is the arbiter | Measure end-to-end against the Topic 65 baseline |

**The last row is the one to take seriously.** Fill the table honestly and then ask
whether any in-JVM option beats deleting the cache.

### What the drill proves

Carry three sentences out of it:

> *`volatile` made every access ordered and visible, and the program still lost money.
> Visibility and atomicity are different properties, and only one of them is a keyword.*

> *A `FORBIDDEN` outcome with zero samples means "not observed here", never "impossible".
> The proof is the happens-before argument; jcstress catches it being wrong.*

> *The increment bug is an interleaving and shows up everywhere; the publication bug is a
> reordering and shows up on aarch64. Knowing which kind you have tells you which
> machine to run the test on.*

---

## Measurement

### The instruments for this topic

| Instrument | What it answers | Verdict |
|---|---|---|
| **jcstress** | Which outcomes are reachable? Is my happens-before argument falsifiable? | **The primary instrument for correctness.** Nothing else answers this |
| **JMH with `@Threads`** | What does a volatile read/write cost, at N threads, on this architecture? | **The primary instrument for cost.** Topic 77 |
| `javap -c -p` / `javap -v` | Is this one operation or three? Is the field actually `ACC_VOLATILE`? | Cheap, decisive, structural |
| `-Xint` | Is this a compiler effect (Topic 86) or an interleaving effect (this topic)? | **The discriminator between the two topics.** An interleaving bug survives `-Xint` |
| `-XX:+PrintAssembly` | Which instructions does my volatile access actually lower to? | Ground truth, **needs `hsdis` which is not bundled**. Heavy |
| A `System.nanoTime()` loop | Nothing useful | **Wrong tool. Always** |

### The standing rule: a naive `System.nanoTime()` loop is WRONG

You will be tempted to write this:

```java
// DO NOT DO THIS. It cannot measure what you think it measures.
long t0 = System.nanoTime();
for (int i = 0; i < 100_000_000; i++) { volatileFlag = true; }
System.out.println((System.nanoTime() - t0) / 1_000_000 + " ms");
```

Five reasons it lies. **Dead-code elimination**: if nothing observes the result, C2 may
delete the loop and you measure nothing (Topic 75). **Store sinking**: a loop-invariant
store of the same value may be sunk out of the loop and executed once. **On-stack
replacement**: the loop starts interpreted and is compiled mid-flight, so your number
blends interpreter, C1 and C2 in proportions set by the iteration count you happened to
choose (Topic 74). **Cold JIT**: the first iterations are interpreted, which is exactly
the state you do not care about.

And the decisive one for **this** topic: **a single-threaded loop measures the wrong
thing entirely.** An uncontended volatile write has the barrier cost and none of the
coherence cost, because the cache line stays in `M` state in one core and never migrates.
**Almost the entire real cost of `volatile` is contention**, and a one-thread benchmark
has none. You would conclude `volatile` is nearly free and be badly wrong at eight
threads.

**Topic 77 is the full treatment of JMH.** Do not write a benchmark you intend to act on
until you have read it.

### JMH with `@Threads` — the correct shape

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.LongAdder;

/**
 * The question: what does each mechanism cost, per operation, AT N THREADS?
 *
 * Thread count is the independent variable. A single-threaded number for any of these
 * is close to meaningless, because the dominant cost is cache-line coherence traffic
 * and one thread generates none.
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)
public class VolatileCostBenchmark {

    int plain;
    volatile int vol;
    final AtomicInteger atomic = new AtomicInteger();
    final LongAdder adder = new LongAdder();

    @Benchmark public int  plainRead()      { return plain; }
    @Benchmark public int  volatileRead()   { return vol; }

    @Benchmark public void plainWrite()     { plain = 1; }
    @Benchmark public void volatileWrite()  { vol = 1; }

    @Benchmark public int  atomicIncrement(){ return atomic.incrementAndGet(); }
    @Benchmark public void adderIncrement() { adder.increment(); }
}
```

```bash
# The sweep. This is the measurement; a single number is not.
java -jar target/benchmarks.jar VolatileCostBenchmark -t 1  -rf json -rff t1.json
java -jar target/benchmarks.jar VolatileCostBenchmark -t 2  -rf json -rff t2.json
java -jar target/benchmarks.jar VolatileCostBenchmark -t 4  -rf json -rff t4.json
java -jar target/benchmarks.jar VolatileCostBenchmark -t 8  -rf json -rff t8.json

# Check nothing allocates unexpectedly.
java -jar target/benchmarks.jar VolatileCostBenchmark -t 8 -prof gc
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | **One shared instance across all threads** — which is the point. `Scope.Thread` would give each thread its own field and measure nothing. |
| Returning the value from a read benchmark | Defeats dead-code elimination without a `Blackhole`. |
| `@Warmup(5)` | Reaches steady-state compiled code. Without it you measure the interpreter and OSR. |
| `@Fork(3)` | Separate JVMs; exposes run-to-run variance and defeats profile pollution between the benchmark methods (Topic 74). |
| `-t N` on the command line | **Thread count is the independent variable and must vary across invocations.** |

**WHAT TO LOOK FOR:** the shape of each curve as `-t` rises, and the **confidence
intervals**.

| What you see | What it means |
|---|---|
| `plainRead` and `volatileRead` are close at `-t 1` | Consistent with the barrier being cheap uncontended. On x86 the read is a plain `mov` and they should be indistinguishable; **on aarch64 `ldar` is a real instruction**, so a gap here is expected and is an architecture fact, not an error. |
| `volatileWrite` is substantially worse than `plainWrite`, and the gap **widens** with `-t` | **The core result.** The barrier cost is fixed; the coherence cost scales with contention. This is the curve to remember. |
| `atomicIncrement` degrades sharply from `-t 4` upward | The contended CAS retry loop — Topic 95. Lock-free does not mean scalable. |
| `adderIncrement` stays much flatter as `-t` rises | `LongAdder` stripes across `@Contended` cells, so the cores stop fighting over one line. Topic 95 and Topic 96. |
| Confidence intervals overlap between two variants | **The difference is below your noise floor.** Report the intervals, never the point estimate, and do not act on an overlap. |
| Your numbers disagree with an x86 blog post | **Expected.** You are on aarch64. Say so when you report them. |

**And state the limit:** JMH measures **cost**. It says nothing at all about
**correctness**. jcstress measures which outcomes are reachable and says nothing about
cost. **Two instruments, two questions, and neither substitutes for the other.**

### `PrintAssembly`, stated honestly

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,com.orderflow.lab.jmm.VolatileCounterLoss::* \
     -cp out com.orderflow.lab.jmm.VolatileCounterLoss
```

**Available, and heavy.** It requires the `hsdis` disassembler plugin, which is **not
shipped with most JDK builds**; without it the JVM prints a "could not load hsdis"
message and no disassembly. Obtaining or building `hsdis` for aarch64 macOS is a real
piece of work. The output is tens of thousands of lines. And reading aarch64 assembly to
confirm an `ldar`/`stlr`/`dmb` pattern is a genuine skill.

**It is the ground truth and you should know it exists.** For this topic it is also the
*only* way to see the barrier instructions directly, which makes it more tempting here
than in Topic 86 — and the honest advice is still that jcstress and JMH answer the
questions you actually have.

### What to track in production

| Signal | Where from | Why |
|---|---|---|
| Business-invariant violations (orders > units, ledger != cached balance) | A reconciliation job comparing the in-memory value to the authoritative store | **The only production signal for a lost update.** There is no log, no exception, and no metric that fires on its own |
| Throughput at the Topic 65 baseline, before and after any `volatile` change | k6 + `/docs/java/baselines/` | A blanket-`volatile` change is a throughput regression until measured otherwise |
| JFR `jdk.JavaMonitorEnter` | Only if you replaced `volatile` with a lock | Topic 85's contention measurement. `volatile` itself emits no JFR event |

**The first row is the point.** A lost update produces no telemetry. If a value must be
right, something must periodically check that it is — and that reconciliation job is
cheap insurance against a bug class with no other observable signal.

---

## Practice exercises

### 1 — easy: classify twelve fields

For each field below, answer three questions: (a) does it need `volatile`? (b) if yes,
is `volatile` **sufficient**, or is it merely necessary? (c) if not sufficient, what is
the right tool and why?

```java
class OrderflowFields {
    boolean shutdownRequested;                    // set once at shutdown, read in a loop
    int     reservedUnits;                        // incremented per reservation
    long    walletBalanceMinor;                   // debited and credited
    PricingEngine instance;                       // double-checked locking target
    List<String> hotSkus;                         // replaced wholesale every 30s, read constantly
    int[]   perSkuReserved;                       // element-wise increments
    Config  activeConfig;                         // immutable record, swapped on refresh
    long    lastRefreshEpochMs;                   // written by one thread, read by many
    Map<String,Integer> counters;                 // read and written by many threads
    double  conversionRate;                       // assigned from a feed, read by many
    int     highWaterMark;                        // updated only if the new value is larger
    Instant startedAt;                            // set in the constructor, never changed
}
```

Two of these need nothing at all. Three need `volatile` and it is sufficient. Four need
an atomic or a lock. Two need a different data structure. One is a trap — `highWaterMark`
is the interesting one, so say precisely why `volatile` cannot express "update only if
larger" and name the exact method that can.

Then answer in one sentence each: why is `startedAt` safe with **no** modifier at all if
it is `final`? Why does making `perSkuReserved` `volatile` change nothing? Why is
`activeConfig` the easiest one on the list?

### 2 — medium: the audit (combines Topics 01–86)

This class is in the `orderflow` codebase. Find **eight** defects. Four are from this
topic; four are from earlier topics. For each: name the topic, state the **observable**
symptom (a wrong number, a thread state, an exception, a percentile), and write the fix.

```java
package com.orderflow.inventory;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class HotSkuReservations {

    private static volatile Map<String, Integer> RESERVED = new HashMap<>();

    private volatile int totalReserved;
    private volatile long lastReservationEpochMs;
    private volatile int[] perSkuCounts = new int[1000];
    private Long capacity = 1000L;

    public boolean tryReserve(String sku, int units) {
        Integer current = RESERVED.get(sku);
        int now = current == null ? 0 : current;
        if (totalReserved + units <= capacity) {
            RESERVED.put(sku, now + units);
            totalReserved += units;
            perSkuCounts[sku.hashCode() % 1000]++;
            lastReservationEpochMs = System.currentTimeMillis();
            return true;
        }
        return false;
    }

    public Snapshot snapshot() {
        return new Snapshot(totalReserved, lastReservationEpochMs, RESERVED);
    }

    public boolean isSaturated() {
        return totalReserved == capacity;
    }
}
```

Hints, in the order to think about them: one is `volatile` on a field whose *contents*
are mutated; one is a compound `+=` on a volatile int; one is check-then-act spanning
several fields; one is two volatile reads combined into an inconsistent snapshot; one is
a Topic 01 boxing comparison that is also a correctness bug (look hard at
`isSaturated()`); one is a Topic 12/92 `HashMap` mutated concurrently; one hands out a
live mutable map reference (Topic 17/88); and one is an array index that can be negative.

(Yes, that is more than eight. Find them all, then rank them by **how much money each one
could lose before anyone noticed** — the ranking is the exercise.)

### 3 — hard: production simulation under load

**Part A — reproduce and record.** Run the Topic 65 baseline with the wallet cache in its
`volatile`-only form. Confirm percentiles within ±10%. Then add a **reconciliation
assertion**: after each run, compare the in-memory wallet balances against Postgres and
count the divergences. Three runs; state your noise floor.

**Part B — quantify the loss.** At the Topic 65 hot-wallet distribution, record: number of
divergent wallets, total minor units of divergence, and the direction (did `orderflow`
gain or lose money?). **This number is the business case for the fix**, and producing it
is the exercise.

**Part C — encode it as jcstress.** Write `VolatileWalletDebit` from the drill, run it,
and put the outcome table next to the Part B numbers. Write the paragraph you would send
to a product manager: one sentence of business impact, one of mechanism, one of fix, one
of cost.

**Part D — the fix matrix.** Implement all five in-JVM fixes plus "no cache", and fill in
the table from drill Step 6 (jcstress outcome + JMH throughput at 1/2/8 threads +
end-to-end p99 against the baseline). **Then argue for one**, in writing, on the evidence.

**Part E — the architecture experiment.** Run `PlainPricePublication` and
`VolatilePricePublication` on your Mac. If you have access to any x86 machine — a CI
runner, a cloud VM, a colleague's laptop — run both there too. Produce a two-column
comparison and write one paragraph explaining, to a team that treats a green x86 CI run
as proof, why that run was weak evidence. **If you have no x86 access, say so and write
the paragraph anyway from the TSO table** — reasoning about it correctly is the skill,
and the honest limitation is part of the answer.

**Part F — argue against yourself.** You will conclude that `AtomicReference<Balance>` is
the right fix. Make the strongest possible case for `synchronized` instead, on
`orderflow` specifically. Then state what would have to be true about the contention
level, the number of fields, or the team's familiarity for that case to win — and say
which of those you could measure today.

---

## Interview questions

### Q1 — "What does `volatile` do?"

**MID-LEVEL answer:** "`volatile` makes a variable thread-safe. It forces reads and
writes to go to main memory instead of a CPU cache, so other threads see the latest
value."

**SENIOR answer:** "I'd lead with what it does **not** do, because that is where the bugs
are.

**`volatile` does not make anything atomic beyond a single read or a single write.**
`volatile int i; i++;` is a read-modify-write — three operations — and it loses
increments under contention on every architecture. If someone shows me a `volatile`
counter, the field being `volatile` is not the fix, it is the *evidence that somebody
thought about it and reached for the wrong tool.*

**What it does do is two things, and the second is the underrated one.**

**Visibility**: a volatile write happens-before every subsequent read of that same field.
That is a specification-level guarantee, and it is also what forbids the JIT from
hoisting the read out of a loop — which is the actual mechanism behind the classic
non-terminating stop-flag bug.

**Ordering**: and this is the half most people miss. Because of transitivity, everything
the writer did *before* the volatile write is visible to a reader that observes the new
value. **One volatile write publishes an arbitrary amount of ordinary state.** That is
how the message-passing idiom works, it is why double-checked locking needs `volatile`,
and it is why `ConcurrentHashMap`'s reads can be lock-free.

**Mechanically**: a volatile write emits a `StoreStore` barrier before it and a
`StoreLoad` barrier after it; a volatile read emits `LoadLoad` and `LoadStore` after it.
`StoreLoad` is the expensive one because it requires draining the store buffer. On x86,
whose TSO model already forbids the other three reorderings, **the read compiles to a
plain `mov` and is effectively free**, while the write needs a locked instruction. On
aarch64 — which is what I'm developing on — both are real instructions, `ldar` and
`stlr`, so the cost profile is genuinely different and an x86 benchmark does not
transfer.

**And I'd correct one thing in the common phrasing**: it is not about 'main memory versus
cache'. Caches are coherent; there is nothing to flush. The write may be sitting in the
core's **store buffer**, which is before the coherent domain, and the read may have been
**eliminated by the compiler** entirely. `volatile` constrains the compiler *and* emits
the barriers. Two mechanisms, neither of which is a cache operation.

**When I use it**: a stop flag, a safely-published reference to an immutable object, the
`instance` field in double-checked locking. **When I do not**: anything compound —
counters go to `AtomicInteger` or `LongAdder`, check-then-act goes to `compareAndSet` or
a lock, and anything spanning two fields goes to one immutable object behind one
reference."

**What separates them:** leading with the non-guarantee; naming the ordering/transitivity
half and connecting it to safe publication and DCL; giving the actual barrier names and
placement; knowing the x86/aarch64 asymmetry and that it changes the cost decision;
correcting the "main memory vs cache" folk model with store buffers and compiler
elimination; and finishing with a concrete use/don't-use list rather than a definition.

**Interviewer's follow-up:** *"So is `volatile` cheaper than `synchronized`?"* — For the
right shape, yes, and the comparison is not apples to apples. An uncontended
`synchronized` block is a CAS on the mark word, which on x86 is already a full barrier —
so the *barrier* cost is comparable. What `volatile` avoids is the possibility of
*blocking*: no thread ever waits on a volatile access, so there is no inflation, no
parking, no context switch. What it gives up is mutual exclusion, which means it cannot
protect a compound operation or an invariant across two fields. So the honest framing is:
they are not competing implementations of the same thing. `volatile` is cheaper because
it does less, and the question is whether the less is enough.

---

### Q2 — "Why does `volatile i++` still lose updates?"

**MID-LEVEL answer:** "Because `i++` isn't atomic — it's read, increment, write. Two
threads can read the same value. You need `AtomicInteger` or `synchronized`."

**SENIOR answer:** "That is the right conclusion, and I'd want to be able to show the
interleaving rather than assert it, because the interleaving is what makes it obvious
that `volatile` was never going to help.

**`i++` compiles to three bytecodes**: `getfield`, `iadd`, `putfield`. `javap -c` shows
it, and it shows the same three whether or not the field is `volatile` — the modifier
changes how each access is *compiled*, not how many there are.

**Here is the trace.** Thread A reads 8. That read is volatile: fully barriered, fully
ordered, genuinely the freshest value in the machine. A computes 9 in a register on its
**private stack** — nothing in shared memory has changed and there is nothing to publish.
The OS preempts A. Thread B reads 8, and **that read is also completely correct**. B
computes 9 and does a volatile write of 9: barriered, drained, globally visible,
perfect. A resumes, does not re-read because it already holds 9, and writes 9 — also
barriered, also perfect. **Two increments, one result, and every single memory operation
was correct.**

**The defect is in the gap between A's read and A's write**, and `volatile` has nothing
to say about gaps. It orders accesses; it does not join them.

**What makes it expensive in production** is that the final value is *indistinguishable*
from a correct one. A wrote 9; B had written 9. The data carries no evidence. You find it
from the business — orders exceeding units, a ledger that does not match a cache.

**Two diagnostic notes I'd add.** First, this is an **interleaving** bug, not a reordering
bug, so unlike the stop-flag problem it is **architecture-independent** and it **survives
`-Xint`**. That difference is how I tell the two families apart in production. Second, it
does not reproduce in a two-thread unit test on an idle laptop and reproduces immediately
under real load, which is exactly why I would encode it as a **jcstress test** rather than
a JUnit test — two actors, an arbiter, and outcomes `1` and `2` declared, and `1` graded
`ACCEPTABLE_INTERESTING` so the report shouts about it.

**The fixes, chosen by shape**: `AtomicInteger.incrementAndGet` for one counter, which is
a `lock xadd` or a CAS retry loop at the machine level; `LongAdder` when writes dominate,
because it stripes across cells and avoids the cache-line ping-pong that makes a contended
CAS *lose* throughput as you add cores; a lock if more than one field must change
together; and — the one I'd propose first for a wallet balance — **not keeping a second
copy of money in memory at all**, and doing a conditional UPDATE in the database that is
atomic by construction."

**What separates them:** producing the interleaving with the observation that *both reads
were correct*; distinguishing interleaving from reordering and knowing `-Xint` does not
cure this one; noting that the corrupt final value is indistinguishable from a correct
one; reaching for jcstress rather than a JUnit test; choosing between `AtomicInteger` and
`LongAdder` on a stated criterion; and proposing the design-level fix.

**Interviewer's follow-up:** *"Would `AtomicInteger` fix `if (count < limit) count++`?"* —
No, and this is the same mistake one level up. `get()` and `incrementAndGet()` are each
atomic; the **sequence** is not, so another thread can increment between the check and
the act. **Making each operation atomic does not make a sequence of them atomic** — that
is Topic 92's headline. The fix is a single atomic operation that includes the condition:
a `compareAndSet` retry loop that re-reads, re-checks and retries on failure, or
`updateAndGet` with the guard inside the lambda, or a lock.

---

### Q3 — "Why does double-checked locking need `volatile`?"

**MID-LEVEL answer:** "Because without it, another thread might see a partially
constructed object. The `volatile` makes sure the object is fully built before the
reference is visible."

**SENIOR answer:** "That is the right shape and I'd make it precise, because the
precision is what lets you fix the *next* one.

**`instance = new PricingEngine()` is three operations**: allocate memory, run the
constructor initialising the fields, and write the reference into `instance`. **Steps two
and three may be reordered** — by the JIT, or by the CPU — because within the writing
thread the reordering is unobservable, and the JMM only constrains what is observable
across threads where an edge exists.

**The bug is entirely in the unlocked first check.** The `synchronized` block is correct
and does its job. But a thread executing `if (instance == null)` with no lock held has no
happens-before edge to the writer, so it can observe a non-null reference to an object
whose constructor has not finished. The symptom is an NPE from *inside* the engine, on a
field the constructor demonstrably sets — which is why every reviewer declares it
impossible.

**`volatile` fixes it with both of its barriers, and both halves are required.** The
`StoreStore` before the volatile write forbids the reference store from being reordered
before the constructor's field stores. The `LoadLoad`/`LoadStore` after the reader's
volatile read forbids the reader's field loads from being hoisted above the null check.
**That is why both the write and the read must go through the same volatile field** — one
side is not enough.

**But I would not write double-checked locking.** For a static singleton the **holder
idiom** is strictly better: a private static nested class with a `static final` field.
The JVM guarantees class initialisation is thread-safe and lazy (JLS class-initialisation
locking), so you get laziness, thread safety, and zero synchronisation in your own code,
with no `volatile` and no way to get it subtly wrong. In Spring, the honest answer is
usually 'declare it a `@Bean`' — the container already solved this.

**And one genuinely surprising fact worth knowing**: if every field of `PricingEngine`
were `final`, the constructor's **freeze action** — a `StoreStore` barrier at constructor
end — would prevent the partially-constructed observation, with no `volatile` at all.
That does not make unsynchronised lazy init correct, because you could still see a stale
`null` and construct twice, but it removes the *partially-constructed* hazard for free.
That is the safe-publication guarantee, and it is one of the strongest arguments for
making fields `final` by default."

**What separates them:** naming the three operations and which two reorder; identifying
that the bug lives in the *unlocked* check, not in the lock; explaining both barriers and
why both sides must be volatile; preferring the holder idiom and knowing why the JVM's
class-initialisation guarantee is stronger than anything you would write; and the
final-field observation, which is uncommon knowledge and directly foreshadows safe
publication.

**Interviewer's follow-up:** *"How likely is this to actually happen?"* — On x86,
unlikely from hardware alone: TSO forbids the `StoreStore` reordering, so the remaining
source would be the JIT. On aarch64 — which is what modern Macs and a growing share of
cloud instances are — the hardware permits it, so it is a real possibility. **That is
exactly the class of bug where 'it works on my laptop' and 'it works in production' can
disagree**, and where a green x86 CI run is weak evidence. I would encode it as a
jcstress test and run it on both, and I would still treat a zero-observation `FORBIDDEN`
row as "not observed here" rather than "impossible" — the proof is the happens-before
argument, not the empty column.

---

### Q4 — "This works on my laptop and fails in production on ARM. Why?"

**MID-LEVEL answer:** "ARM and x86 behave differently. Probably a timing issue — there's
more load in production so races are more likely to hit."

**SENIOR answer:** "I'd refuse the 'more load, more races' framing first, because it sends
teams in the wrong direction for a week. A memory-model bug is not *more likely* under
load; it is **permitted always** and merely *observed* sometimes.

**Then I'd split it into two mechanisms, because they have different fixes and different
architecture sensitivity.**

**The compiler.** The JIT reorders, hoists and eliminates memory operations whenever no
happens-before edge forbids it. **This is architecture-independent** — identical on x86
and aarch64 — and the reason it 'works on my laptop' is usually that the laptop run was
short and the method never reached C2. **The test is `-Xint` versus the JIT**, two
seconds.

**The hardware.** **x86-64 implements Total Store Order**: it forbids load-load,
store-store and load-store reordering, permitting only a later load moving ahead of an
earlier store to a different address. **aarch64 permits all four.** So an unsynchronised
program can be accidentally sequentially consistent on x86 and visibly broken on ARM.
**x86 hides a whole class of bugs that ARM exposes.** That is the direct answer to the
question: the code was always wrong, and x86's stronger hardware model was papering over
it.

**And here is the part I'd want the team to hear**: if our laptops are Apple Silicon and
CI is x86, **the laptops are the better test environment and CI is the one giving false
confidence.** Most teams assume the opposite.

**What I'd actually do**, in order. Stop treating the x86 result as evidence of
correctness — it is evidence of a stronger hardware model, nothing more. Find the
unsynchronised shared state **by reading**, because the happens-before argument is the
proof and reproduction is only useful for convincing people. Write a **jcstress** test
for the specific invariant, declare the bad outcome `FORBIDDEN`, and run it on both
architectures — while being explicit that **a `FORBIDDEN` row with zero samples means
'not observed on this machine, this JDK, this run', not 'proven impossible'.** Then fix by
establishing the edge, and verify the argument rather than the absence of the symptom.

**The sentence I'd want them to leave with:** 'it works on x86' and 'it is correct' are
different claims, and only one of them is about our code."

**What separates them:** refusing the load framing; splitting compiler from hardware with
a test for each; stating TSO's guarantees precisely; knowing which way the laptop/CI
asymmetry cuts; treating the argument as the proof and the test as a falsifier; and
volunteering the zero-observations caveat unprompted.

**Interviewer's follow-up:** *"How would you decide whether a specific bug is the
compiler or the hardware?"* — `-Xint`. If the symptom disappears under the interpreter,
the compiler's optimisation is on the causal path — that is the stop-flag family. If it
survives `-Xint`, it is an interleaving or a hardware reordering, and then the
discriminator is architecture: run the same jcstress test on x86 and aarch64 and compare
the observed outcome *sets*. An outcome that appears on ARM and never on x86 is strong
evidence for hardware reordering. Two experiments, and between them they classify almost
every memory-model bug you will meet.

---

### Q5 — "How would you test that a class is thread-safe?"

**MID-LEVEL answer:** "Write a test that spins up a bunch of threads, hammers the class,
and asserts the final state is correct. Run it a few thousand iterations."

**SENIOR answer:** "I would push back on the premise slightly, because **you cannot test
a class into being thread-safe**, and being clear about that changes what the testing is
for.

**The proof of thread safety is an argument, not a test.** For every piece of shared
mutable state: which happens-before edge orders the accesses, and is every compound
operation atomic? If I can name the edge for each and the atomic operation for each,
the class is correct. If I cannot, no amount of passing tests makes it correct.

**Then testing has a specific and valuable job: falsifying the argument.** And a
loop-and-assert test is a poor falsifier, because it tests one machine's scheduling, one
JIT state and one afternoon's luck. It passes for wrong code routinely — which is worse
than useless, because it manufactures confidence.

**The right tool is jcstress.** It runs a pair of `@Actor` methods against many thousands
of freshly-allocated `@State` instances, across multiple JIT modes and affinity
configurations, and reports the **frequency of every observed outcome** against outcomes
I declared `ACCEPTABLE`, `ACCEPTABLE_INTERESTING` or `FORBIDDEN`. An `@Arbiter` reads the
settled state after both actors, with edges from both. Crucially, **an outcome I did not
declare fails the test as `UNKNOWN`** — so I am forced to enumerate what I think is
reachable, and being surprised is the point.

**And I'd state two honesty rules to whoever reads the report.** First: **a `FORBIDDEN`
outcome with zero samples means 'not observed on this machine, this JDK, this run' — not
'proven impossible'.** It is failure to falsify, which is a real result and is not proof.
Second: **the `FREQ` column is not a production probability.** It is an outcome frequency
under jcstress's own aggressive scheduling. It must never go into a risk assessment as a
likelihood.

**I'd also make the architecture point explicitly**, because it decides where the test
runs. x86-TSO forbids three of the four hardware reorderings, so a clean x86 run is weak
evidence about aarch64. For publication and reordering bugs, an Apple Silicon machine is
the better instrument, and I'd want the test on both.

**What I'd actually ship for a class like this:** the happens-before argument written in
the class Javadoc — literally, 'this field is guarded by X; this operation is atomic via
Y' — a jcstress test for each non-obvious invariant, a JMH `@Threads` benchmark for the
cost, and a production reconciliation job that periodically checks the business invariant,
because a lost update produces no telemetry and something has to notice."

**What separates them:** rejecting the premise that testing establishes thread safety;
naming the argument as the proof and the test as the falsifier; describing jcstress's
actual mechanism including the arbiter and the `UNKNOWN` failure; stating both honesty
rules unprompted; knowing the architecture decides where to run the test; and closing with
a shippable four-part practice including the production reconciliation job.

**Interviewer's follow-up:** *"What if jcstress is too heavyweight for our CI?"* — Then I
would not run it on every commit. jcstress runs are slow by design; that is the price of
hunting rare interleavings. I would run it on a nightly job, or on a per-package basis
when the concurrency-sensitive code changes, and I would treat the test as a permanent
artefact that documents the invariant rather than a gate on every push. **The Javadoc
argument is the thing that must be on every commit**, because it is what a reviewer reads
and it costs nothing. And I would keep the production reconciliation check regardless,
because it is the only signal that fires when the argument was wrong in a way nobody
tested for.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A volatile write emits `StoreStore` before and `StoreLoad` after. Derive, from the
   placement alone, why the message-passing idiom (`data = x; ready = true;` /
   `if (ready) use(data);`) is correct, and explain which barrier would have to be missing
   for it to break in each of the two possible ways.

2. On x86 a volatile read compiles to a plain `mov` and costs nothing. Explain why, in
   terms of exactly which reorderings TSO forbids — then explain why the same read on
   aarch64 needs `ldar`, and what that means for a benchmark you read on a blog.

3. `volatile int i; i++;` loses updates. Construct the interleaving from memory, and
   identify the precise step at which the loss becomes inevitable. Then say why **both**
   reads in your trace are correct.

4. A `volatile` reference to an `ArrayList` gives you a guarantee about the reference and
   none about the list. Explain the mechanism — what write, and therefore what barrier,
   does `list.add(x)` perform? Then give the two-line fix.

5. Two `volatile long` fields, read one after the other, can produce a pair of values that
   describes a state the system was never in. Explain why no barrier can fix this, and
   name the structural change that does.

6. Topic 86's stop-flag bug is cured by `-Xint`. This topic's lost-increment bug is not.
   Explain the difference in terms of which agent — compiler, scheduler, or CPU — is
   responsible for each, and say which of the three `-Xint` removes.

7. A jcstress run declares an outcome `FORBIDDEN` and reports zero samples for it. State
   precisely what you are entitled to conclude, and what you are not. Then say what would
   make you *more* confident, and what would make you *certain*.

---

## Quick reference card

### The happens-before edges — the complete list

| # | Edge | Rule |
|---|---|---|
| 1 | **Program order** | Within one thread, each action happens-before every later action in program order |
| 2 | **Monitor lock** | An unlock happens-before every subsequent lock of **that same** monitor |
| 3 | **Volatile** | A write to a `volatile` field happens-before every subsequent read of **that same** field ← **this topic** |
| 4 | **Thread start** | `Thread.start()` happens-before any action in the started thread |
| 5 | **Thread join** | Every action in a thread happens-before another thread returns from `join()` |
| 6 | **Thread termination** | Every action in a thread happens-before another detects it has terminated |
| 7 | **Interruption** | `interrupt()` happens-before the interrupted thread detects it |
| 8 | **Final field freeze** | Constructor end happens-before a read of a `final` field, if `this` did not escape — Topic 88 |
| 9 | **Default values** | The default write (0/false/null) happens-before the first action of every thread |
| 10 | **Transitivity** | A hb B and B hb C implies A hb C — **what makes edge 3 publish more than one field** |

### The four barriers, and where `volatile` puts them

| Barrier | Prevents | x86-64 (TSO) | aarch64 |
|---|---|---|---|
| **LoadLoad** | A later load moving before an earlier load | Free | Real barrier / `ldar` |
| **StoreStore** | A later store becoming visible before an earlier store | Free | Real barrier / `stlr` |
| **LoadStore** | A store moving before an earlier load | Free | Real barrier |
| **StoreLoad** | A load executing before an earlier store is globally visible | **`mfence` / `lock addl $0,(%rsp)` — the cost** | `dmb ish` |

```
   [ StoreStore ]                            volatile READ
   volatile WRITE                            [ LoadLoad  ]
   [ StoreLoad  ]   <- the expensive one     [ LoadStore ]
```

| Construct | x86-64 | aarch64 |
|---|---|---|
| volatile read | plain `mov` — **free** | `ldar` |
| volatile write | `mov` + `lock addl $0,(%rsp)` — **the cost** | `stlr` (+ `dmb ish`) |
| `synchronized` enter/exit | `lock cmpxchg` is already a full barrier | `ldaxr`/LSE acquire, `stlr`/`dmb` |
| `final` freeze | free | `dmb ishst` |

### `volatile`: yes / no

| Yes | No |
|---|---|
| A stop or state flag (write rarely, read constantly) | A counter — `i++` is three operations |
| A safely-published reference to an **immutable** object | Check-then-act — `if (a) b();` |
| The DCL `instance` field (**mandatory**) | Two fields that must agree |
| A value **assigned** from one place, read from many | A value that is **derived from its own previous value** |
| 64-bit values needing atomic access | Anything needing mutual exclusion |
| | The **contents** of a `volatile` array or collection |

**The one-line test:** *is every operation on this field a single read or a single write?*

### Architecture, in one box

- **x86-64 = Total Store Order.** Forbids Load-Load, Store-Store, Load-Store. Permits
  only Store-then-Load to a different address.
- **aarch64 (your Mac) = weakly ordered.** Permits all four.
- **x86 hides bugs aarch64 exposes.** Your Mac is a **better** instrument for the
  publication/reordering tests. **It is not a guarantee** — never say a race "will"
  reproduce.
- **Which bugs are architecture-sensitive:** reordering bugs (publication, DCL) — **yes**.
  Interleaving bugs (`i++`, check-then-act) — **no**, they occur everywhere and survive
  `-Xint`.

### jcstress

```bash
mvn archetype:generate -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jcstress \
  -DarchetypeArtifactId=jcstress-java-test-archetype \
  -DgroupId=com.orderflow -DartifactId=orderflow-jcstress -Dversion=1.0
# Pin the CURRENT version from github.com/openjdk/jcstress README. Do not copy one
# out of a teaching document.

mvn clean verify                        # NOT `mvn compile` - the processor must run
java -jar target/jcstress.jar -h        # the CLI changes across versions
java -jar target/jcstress.jar -l        # list discovered tests
java -jar target/jcstress.jar -t <Regex>
java -jar target/jcstress.jar -t <Regex> -m quick    # shorter
java -jar target/jcstress.jar -t <Regex> -v          # per-config detail
java -jar target/jcstress.jar -t <Regex> -r results/ # HTML report
```

```
RESULT      SAMPLES     FREQ       EXPECT  DESCRIPTION
     1          <n>      xxx%  INTERESTING  One increment was LOST.
     2          <n>      xxx%   ACCEPTABLE  Both increments landed.
```

*illustration of the format, not captured output*

| Grade | Meaning |
|---|---|
| `ACCEPTABLE` | Legal and expected |
| `ACCEPTABLE_INTERESTING` | Legal, and the thing you are hunting — the report highlights it |
| `FORBIDDEN` | Your correctness claim. **Observing it fails the test and means your argument is wrong** |
| `UNKNOWN` | You did not declare it. Fails the test, correctly |

> **The two rules to say out loud every time you show someone a jcstress table:**
> **(1)** A `FORBIDDEN` outcome with zero samples means **"not observed on this machine,
> this JDK, this run" — NOT "proven impossible".** The proof is the happens-before
> argument; jcstress catches the argument being wrong.
> **(2)** `FREQ` is an outcome frequency under jcstress's scheduling. It is **not** a
> production probability and must never enter a risk assessment as one.

### Diagnostic commands

```bash
# Is this one operation or three? Is the field really volatile?
javap -c -p -cp out <Class>
javap -v -p -cp out <Class> | grep -B2 -A2 ACC_VOLATILE

# Compiler effect (Topic 86) or interleaving effect (Topic 87)?
java -Xint -cp out <Class>          # interleaving bugs SURVIVE this

# Cost, at N threads, on THIS architecture. Topic 77.
java -jar target/benchmarks.jar <Bench> -t 1 -rf json -rff t1.json
java -jar target/benchmarks.jar <Bench> -t 8 -rf json -rff t8.json
java -jar target/benchmarks.jar <Bench> -t 8 -prof gc

# Ground truth on the barriers. Needs hsdis, which is NOT bundled with most JDKs.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,com.orderflow.<Class>::* -cp out <Class>

# Never quote a default you did not print.
java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders
uname -m ; java --version
```

### `[JAVA 25]` one line

`-XX:+UseCompactObjectHeaders` narrows the object header and changes mark-word field
widths. **It changes nothing in this document** — no barrier, no edge, no `volatile`
semantics depend on header width. Flagged only so a JOL layout differing from a blog post
does not confuse you. Settle it with `-XX:+PrintFlagsFinal -version` and JOL's
`ClassLayout.parseInstance(o).toPrintable()`.

### Gotchas checklist

- [ ] `volatile` buys visibility and ordering. It buys **no** atomicity for compound ops.
- [ ] `i++` is three operations. `volatile` orders each and joins none.
- [ ] Check-then-act is never atomic, no matter how volatile the field is.
- [ ] `volatile` applies to the **field**, not to the object or array it points at.
- [ ] Two volatile fields read separately can describe a state that never existed.
- [ ] Double-checked locking **requires** `volatile` on both the write and the read side.
- [ ] The holder idiom beats DCL for a static singleton, every time.
- [ ] On x86 the volatile **read** is free; the **write** is the cost. On aarch64 both cost.
- [ ] `volatile` does not "flush the cache". Store buffers and the compiler, not caches.
- [ ] One volatile write publishes everything written before it — that is transitivity.
- [ ] A blanket `volatile` policy is a measurable throughput regression bought for nothing.
- [ ] Interleaving bugs survive `-Xint`. Compiler bugs do not. That is your discriminator.
- [ ] x86 hides reordering bugs aarch64 exposes. A green x86 CI run is weak evidence.
- [ ] A `FORBIDDEN` row with zero samples means "not observed here", never "impossible".
- [ ] `FREQ` is not a production probability. Never put it in a risk assessment.
- [ ] A lost update produces no telemetry. Something must reconcile it in production.

---

## When would I use this at work?

**1. Reviewing any pull request that adds `volatile` to a field.**

`volatile` reads as diligence, which is exactly why it slips through. You have a
ten-second question that catches the whole bug class: *is every operation on this field a
single read or a single write?* If any operation reads the field and writes a value
derived from it, `volatile` is the wrong tool, and you can say which is the right one —
`AtomicX`, a CAS loop, a lock, or one immutable object behind one reference. That is a
specific, citable review comment rather than "I think there might be a race here", and
over a year it prevents more silent data corruption than any load test, because these
bugs do not reproduce on demand.

**2. Deciding whether an in-memory cache in front of a money-critical store is
acceptable.**

This is the `orderflow` wallet decision, and it recurs constantly in real systems. You can
now do it properly: name the invariant, name which operations must be atomic, encode the
bad outcome as a jcstress test, measure the fix's cost with JMH at realistic thread counts
on the target architecture, and put "delete the cache" in the comparison table as a real
option. **Most teams have this argument on intuition. You can have it on evidence**, and
having the evidence is usually what makes "don't cache money" win.

**3. Adjudicating a "works on my machine" dispute between a laptop and CI.**

Increasingly common as teams move to Apple Silicon while CI stays x86. You know that
x86-TSO forbids three of the four reorderings, so the two environments are running
different experiments, and — counterintuitively for most teams — **the ARM laptop is the
stronger evidence, not the weaker.** Being the person who says "run it on both, and treat
the green x86 result as weak" changes how the team interprets every concurrency result
they get, and it costs nothing to say.

---

## Connected topics

**Prerequisites:**

- **17 — Immutability, `final`, safe publication**: the immutable-snapshot-behind-a-
  volatile-reference pattern is this topic's best answer, and it only works because the
  snapshot is genuinely immutable. Topic 88 is the payoff.
- **27 — Records and shallow immutability**: a `record` is the right carrier for the
  "two fields must agree" fix in Trap 5 — and a record holding a mutable collection
  re-opens the hole.
- **69 — The mark word**: `synchronized`'s CAS is a full barrier on x86, which is why the
  barrier cost of an uncontended lock and a volatile write are comparable.
- **73 — Safepoints**: the compiled-code poll placement and the barrier placement are both
  decisions the JIT makes about your memory operations.
- **74 / 75 — JIT, LICM, inlining**: the compiler's licence to hoist and eliminate reads
  is precisely what `volatile` withdraws. This topic's Trap 3 depends on the JIT's freedom
  to reorder the constructor against the reference store.
- **76 — Bytecode and `javap`**: proves `i++` is three instructions with or without
  `volatile`. One command, one screen, whole argument.
- **84 — Threads versus the event loop**: preemption between any two bytecodes is what
  makes the gap in `i++` exploitable. This topic's trace is Topic 84's trace with barriers
  added and the bug unchanged.
- **85 — `synchronized` and monitors**: edge 2, the alternative to edge 3, and the
  double-checked-locking trap this document finally explains.
- **86 — The JMM I**: the ten edges, the four barriers, x86-TSO versus aarch64, and the
  compiler's licence. **This document is edge 3 in full detail** and assumes all of it.

**This unlocks:**

- **88 — `final` fields and safe publication**: the freeze action is a `StoreStore`
  barrier at constructor end, and it is the only publication mechanism that asks nothing
  of the reader. Compare it against this document's volatile write.
- **89 — `wait`/`notify`**: a guarded block is check-then-act made correct by holding the
  monitor across both halves — the structural answer to this topic's Trap 5.
- **92 — `ConcurrentHashMap`**: lock-free reads are volatile reads of `Node.val`, and the
  per-key happens-before guarantee is edge 3 applied deliberately. Its headline —
  operations are atomic, sequences are not — is this topic's Trap 5 one level up.
- **94 — Explicit locks**: AQS's `volatile int state` is how `ReentrantLock` gets its
  edge. Same mechanism, different ergonomics, plus `tryLock` and conditions.
- **95 — Atomics, CAS, `LongAdder`**: the correct tool for every case where `volatile` is
  insufficient. `compareAndSet` is the atomic check-then-act; `LongAdder` is the answer
  when contention on one cache line is the bottleneck; `VarHandle` exposes the access
  modes (`getAcquire`, `setRelease`, `getOpaque`) that let you buy a weaker edge on
  purpose.
- **96 — False sharing**: the coherence traffic that dominates `volatile`'s real cost,
  one level down. `LongAdder`'s `Cell` is `@Contended` for exactly this reason.
- **99 — jcstress**: the full treatment of the tool this document introduced, including
  its own statement of the two honesty rules.
- **101 — Virtual threads**: mount/unmount carries happens-before edges, so a virtual
  thread's writes survive carrier migration. `volatile` semantics are unchanged; what
  changes is how many threads can be contending on one line.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted: the exact instruction selection HotSpot uses for volatile
accesses on your aarch64 build (`stlr` alone versus `stlr` plus `dmb ish` depends on
surrounding code and JDK version), whether any given jcstress test will surface its
interesting outcome on your machine, and the relative cost of the mechanisms in the JMH
benchmark, which you must measure rather than read. Each has a command that settles it on
your machine. **No sample count, frequency, timing, percentile or throughput number in
this document was measured — I have no JVM.** The jcstress table shown is a column-format
illustration with `<n>` placeholders and is labelled as such. Two rules govern every
experiment in Topics 86 through 88: a `FORBIDDEN` outcome with zero observations means
"not observed on this machine, this JDK, this run", never "proven impossible" — the proof
is the happens-before argument, and jcstress is how you catch that argument being wrong;
and x86's Total Store Order hides reordering bugs that aarch64 exposes, which makes your
Apple Silicon Mac the better instrument for the publication tests here and in Topic 88 —
while making no difference at all to the lost-increment test, which is an interleaving
bug and occurs everywhere. What has been stable since Java 5 and will still be true at
2am: a volatile write emits StoreStore before and StoreLoad after; a volatile read emits
LoadLoad and LoadStore after; this buys visibility and ordering; and it buys no atomicity
for anything composed of more than one access.*
