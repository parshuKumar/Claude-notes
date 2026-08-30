# 95 — Atomics, CAS at the Instruction Level, ABA, and `LongAdder`

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: `orderflow`'s request counter and its in-memory inventory decrement — the two places where the service updates one number from many threads at once, and the place where adding CPUs makes it slower.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> `AtomicLong.incrementAndGet()` does **not** compile to a CAS retry loop on x86-64.
> The JIT intrinsifies it to a **single `lock xadd`** instruction — an unconditional
> atomic add that cannot fail and never retries.
>
> `compareAndSet(expected, next)` compiles to **`lock cmpxchg`**, which *can* fail.
> `getAndUpdate(fn)` / `updateAndGet(fn)` / `accumulateAndGet(...)` wrap `lock cmpxchg`
> in a **genuine Java-level retry loop**, because an arbitrary function cannot be
> expressed as a single instruction.
>
> **All three are equally expensive under contention, and the reason has nothing to do
> with retrying.** The `lock` prefix requires the executing core to take the cache line
> holding that variable into **exclusive ownership**. Every other core's copy is
> invalidated. So N cores updating one counter must pass one 64-byte cache line between
> them, one at a time, at a cost measured in tens to hundreds of nanoseconds per
> transfer.
>
> That is why a contended atomic counter's **throughput decreases as you add cores**.
> More cores means more line transfers, and the line is the bottleneck.
>
> `LongAdder` solves it by not sharing: each contending thread hashes to its own
> `Cell`, each `Cell` sits alone in its own cache line, and the total is computed only
> when someone calls `sum()`.

The one-sentence version to say in an interview: **"lock-free means no thread is
blocked; it does not mean it scales, because the cache line is still a serial
resource."**

---

## The bridge from what you know

### `NO TYPESCRIPT ANALOGUE.` — with one real exception

The general case first. A single-threaded runtime has no CAS contention, because it has
nothing to contend with. `count++` in Node is already atomic in every way that matters:
nothing can run between the read and the write. There is no `AtomicInteger`, no
`LongAdder`, no ABA problem, and no scaling curve, because there is nothing to scale.

Everything in this document about **contention** — cache-line ownership, retry storms,
throughput falling as cores are added — has no counterpart in your experience. Do not
try to map it onto `worker_threads` either: workers have **isolated heaps** and
communicate by copying, so they cannot contend on a variable at all.

### The exception: `Atomics.compareExchange` on a `SharedArrayBuffer` is a genuine CAS

This is real, it is the same primitive, and most JavaScript engineers have never used
it — so it is worth knowing precisely, both because it is the honest bridge and because
mentioning it in an interview is unusual enough to be memorable.

```ts
// main.ts
const sab = new SharedArrayBuffer(8);
const counter = new Int32Array(sab);

const worker = new Worker('./worker.js', { workerData: sab });
```

```ts
// worker.js -- runs in a different thread, on the SAME memory
const counter = new Int32Array(workerData);

// An unconditional atomic add. This is `lock xadd`.
Atomics.add(counter, 0, 1);

// A compare-and-exchange. This is `lock cmpxchg`.
const previous = Atomics.compareExchange(counter, 0, /* expected */ 5, /* next */ 6);
if (previous === 5) {
  // we won the race
}

// The CAS retry loop, written by hand -- this is exactly AtomicInteger.updateAndGet
let observed;
do {
  observed = Atomics.load(counter, 0);
} while (Atomics.compareExchange(counter, 0, observed, observed * 2) !== observed);
```

**What transfers exactly:**

| Fact | JS `Atomics` on `SharedArrayBuffer` | Java `java.util.concurrent.atomic` |
|---|---|---|
| Two threads share the same physical memory | yes — that is what `SharedArrayBuffer` means | yes — the whole heap is shared |
| `add` is one atomic instruction | `Atomics.add` → `lock xadd` | `getAndAdd` → `lock xadd` |
| Compare-and-exchange exists as a primitive | `Atomics.compareExchange` → `lock cmpxchg` | `compareAndSet` → `lock cmpxchg` |
| The retry loop must be written by the programmer for arbitrary updates | yes, the `do/while` above | no — `updateAndGet` writes it for you |
| There is a real memory model with `seq_cst` ordering | yes — ECMAScript has one, for this and only this | yes — the JMM (Topics 86–87) |
| Blocking wait primitive | `Atomics.wait` / `Atomics.notify` | `LockSupport.park` / `unpark` |

**Verdict: HONEST ANALOGUE for the primitive. NO ANALOGUE for everything that
follows from contention.** The instruction is the same. What you have never had is
sixty-four threads hitting it at once, and that is the entire subject of this document.

### What has no counterpart at all

| Concept here | Why JS cannot have it |
|---|---|
| `LongAdder`'s striping | You cannot have enough contending threads for it to matter, and there is no equivalent in the language or in any library |
| **ABA** | `Atomics` in JS operate on **integers in a typed array**, not on object references. ABA is fundamentally about a *pointer* whose target was recycled. You cannot CAS a pointer in JS. |
| Throughput falling as cores are added | You do not have shared mutable state across cores to begin with |
| `@Contended` / cache-line padding | You cannot observe or control memory layout from JS at all |

That third row deserves a moment. Your entire performance intuition says **more cores
is more throughput, or at worst the same**. This document contains a case where adding
cores makes the program measurably slower, permanently, with no bug in the code. That
result is genuinely counter-intuitive and it is the thing to internalise.

---

## What is this?

### Compare-and-swap, in one paragraph

**Compare-and-swap (CAS)** is a single machine instruction that does three things
indivisibly: read a memory location, compare it to a value you supply, and — only if
they match — write a new value. It reports whether it succeeded.

That indivisibility is the entire point. Nothing can happen between the compare and the
swap, not on this core and not on any other. It is the hardware primitive on which
every lock, every atomic and every lock-free algorithm in the JDK is built. AQS's
`compareAndSetState` (Topic 94) is a CAS. `synchronized`'s fast path (Topic 85) is a
CAS on the mark word.

### The atomic classes

`java.util.concurrent.atomic` gives you objects whose updates are atomic without a
lock:

| Class | What it holds | Notable operations |
|---|---|---|
| `AtomicInteger`, `AtomicLong` | one number | `incrementAndGet`, `getAndAdd`, `compareAndSet`, `updateAndGet`, `accumulateAndGet` |
| `AtomicBoolean` | one flag | `compareAndSet` — the standard "run once" guard |
| `AtomicReference<T>` | one reference | `compareAndSet`, `updateAndGet` — the basis of lock-free data structures |
| `AtomicStampedReference<T>` | reference **+ an int stamp** | the ABA fix |
| `AtomicMarkableReference<T>` | reference **+ a boolean** | logical deletion in lock-free lists |
| `AtomicIntegerArray`, `AtomicLongArray`, `AtomicReferenceArray` | per-element atomics | avoids one lock per array |
| `LongAdder`, `DoubleAdder` | a **striped** sum | `increment`, `add`, `sum` — **no CAS, no read-modify-write** |
| `LongAccumulator`, `DoubleAccumulator` | striped fold with a custom function | for max, min, or any associative operation |
| `VarHandle` (Topic 87) | a typed handle to any field | `compareAndSet` plus **weaker memory modes** |

Two entries deserve immediate attention because their absence from the list is what
people get wrong:

- **`LongAdder` has no `compareAndSet` and no `get`-then-`set`.** You can `add` and you
  can `sum`. That is deliberate: the value is not stored in one place, so there is no
  single word to compare and swap. If you need CAS semantics, `LongAdder` is the wrong
  class.
- **There is no double-word CAS in Java.** You cannot atomically change two fields.
  `AtomicStampedReference` fakes it by boxing the pair into a single immutable object
  and CASing the reference to that object — which allocates.

### `LongAdder` in one picture

```
AtomicLong:            LongAdder:

  [ value ]              [ base ]        <- used only when uncontended
     ^  ^  ^  ^          [ cells[0] ]    <- thread probe 0 hashes here
     |  |  |  |          [ cells[1] ]    <- thread probe 1 hashes here
    T1 T2 T3 T4          [ cells[2] ]
                         [ cells[3] ]
  ONE cache line.        Each Cell is @Contended -> its own cache line.
  Every core must        Threads write to DIFFERENT lines. No transfers.
  own it exclusively.
                         sum() = base + cells[0] + cells[1] + ...
```

The trade is stated in one line: **`LongAdder` makes writes scale and makes reads
worse.** A `sum()` walks the whole array and is not atomic. `AtomicLong.get()` is one
load.

---

## Why does it matter?

**1. It is the one case where your scaling intuition is inverted.**

Everything else you have learned says "add capacity, get throughput". A contended
atomic counter gets *slower* per core as cores are added, and the aggregate throughput
can be lower at 64 threads than at 4. If you cannot explain that, you will misdiagnose
it as "the JVM doesn't scale" or "we need a bigger instance", and both of those cost
real money for no benefit.

**2. Metrics counters are the most contended objects in most services.**

Every request increments something: a request counter, a per-endpoint timer, an error
count. In `orderflow` at the Topic 65 baseline, that is one shared write per request
per metric, from every request thread simultaneously. It is the single most likely
place in the service for this pathology to appear, and it is instrumentation code that
nobody profiles because "it's just a counter".

**3. "Lock-free" is a correctness term that people read as a performance term.**

Lock-free means *some* thread always makes progress — no thread can be blocked by
another thread being descheduled. It says nothing about throughput. An interviewer
asking "atomics are lock-free so they scale, right?" is checking whether you know the
difference between a progress guarantee and a performance property.

**4. ABA is the bug that only appears when you optimise.**

Java's garbage collector makes ABA rare, because a node you hold a reference to cannot
be freed and reused underneath you. ABA reappears the moment you introduce **object
pooling or recycling** — which is exactly what people do when they are trying to reduce
allocation rate after Topic 68. The optimisation reintroduces the bug the GC was
protecting you from.

---

## Machine-level reality

### What `lock` means on x86-64

The `lock` prefix on an x86 instruction makes that instruction's read-modify-write
sequence atomic with respect to every other core. On any processor made this century it
does **not** assert a physical bus lock; it works through the **cache coherence
protocol** instead. That distinction is the whole story.

The three instructions you care about:

| Instruction | Java operation | Can it fail? |
|---|---|---|
| `lock xadd` | `getAndAdd`, `incrementAndGet`, `getAndIncrement` | **no** — unconditional; returns the old value |
| `lock cmpxchg` | `compareAndSet`, `weakCompareAndSet` | **yes** — sets the zero flag on success |
| `lock xchg` | `getAndSet` | no — unconditional swap (`xchg` with memory is implicitly locked) |

`lock cmpxchg` in words: compare `RAX` with the destination operand. If equal, store the
source operand into the destination and set `ZF`. If not equal, load the destination
into `RAX` and clear `ZF`. Either way you learn the current value, which is what lets a
retry loop make progress.

**On x86, every `lock`-prefixed instruction is also a full memory barrier.** That is
why `AtomicInteger`'s methods have `volatile` read and write semantics for free
(Topic 87) — the barrier comes with the atomicity whether you wanted it or not.

### Apple Silicon and ARM are different, and you should say so

On ARMv8 there is no `lock` prefix. Atomics are built one of two ways:

- **LL/SC** — `LDXR` (load-exclusive) then `STXR` (store-exclusive), which fails if the
  line was touched in between. This is a hardware-level retry loop, and it is why
  "CAS is a retry loop" is more literally true on ARM than on x86.
- **LSE atomics** (ARMv8.1+, which includes all Apple Silicon) — single instructions
  like `LDADD`, `SWP`, `CASAL` that do the whole thing without an explicit loop.

The consequence for you: **the contention behaviour is the same in shape but not
identical in detail**, and ARM's weaker memory model means the JVM must emit explicit
barrier instructions (`DMB`) where x86 got them for free. If you develop on an M-series
Mac and deploy to x86-64 Linux, your microbenchmark numbers will not transfer. Your
*conclusions about scaling* generally will.

### Cache coherence: why a contended atomic is slow

Modern CPUs keep each 64-byte cache line in one of a few states per core. The classic
model is MESI:

| State | Meaning |
|---|---|
| **M**odified | This core has the only copy, and it is dirty |
| **E**xclusive | This core has the only copy, and it is clean |
| **S**hared | Several cores have read-only copies |
| **I**nvalid | This core's copy is stale and must not be used |

**To execute any atomic read-modify-write, a core must hold the line in Modified or
Exclusive state.** That means it must issue a **read-for-ownership (RFO)**: broadcast
to every other core "invalidate your copy of this line, I am taking it".

Now walk through what happens with four cores incrementing one `AtomicLong`:

```
Core 0 owns the line (M).             Cores 1,2,3: Invalid.
Core 1 wants to increment.
  -> RFO: "invalidate line X"
  -> Core 0 writes back / transfers the line
  -> Core 1 now owns it (M).          Cores 0,2,3: Invalid.
Core 2 wants to increment.
  -> RFO ... the line moves again.
Core 3 wants to increment.
  -> RFO ... the line moves again.
Core 0 wants to increment again.
  -> RFO ... the line moves back.
```

The line is a **serial resource**. Every increment requires one transfer. The
transfers cost, in rough orders of magnitude:

| Where the line comes from | Order of magnitude |
|---|---|
| This core's L1 (uncontended) | ~1 ns, a few cycles |
| Another core's cache, same socket | tens of nanoseconds |
| Another socket (NUMA) | **hundreds of nanoseconds** |

So an uncontended `incrementAndGet` costs a handful of nanoseconds and a heavily
contended one can cost two orders of magnitude more. **Do not memorise those numbers —
memorise that the ratio is large and that it grows with core count and socket count.**

### Why a *failed* CAS still costs a full round trip

This is the detail that makes the retry-storm behaviour make sense, and it is a
frequent interview follow-up.

You might imagine a failed CAS is cheap: it did not write anything, so surely it was
just a read? No. **To attempt the compare-and-swap at all, the core must already own
the line exclusively.** The hardware cannot know whether the comparison will succeed
until it has the line. So the sequence for a failed CAS is:

1. Issue the RFO. Invalidate every other core's copy. Wait for the line.
2. Perform the compare. It fails, because another core changed the value.
3. Return failure. You now hold a line you are about to lose again.
4. Retry: read the new value, attempt the CAS again — **and every other core has just
   been invalidated by you, so they must RFO it back.**

**A failed CAS costs approximately what a successful one costs, and produces no work.**
That is the mechanism behind a retry storm: at N contending threads, roughly N−1 of
every N attempts fail, so you are paying N cache-line transfers to accomplish one
increment. The useful work is O(1) and the coherence traffic is O(N).

This is also exactly why `LongAdder` works. It does not make CAS cheaper. It arranges
for threads to CAS **different lines**, so there is no transfer at all.

### What `incrementAndGet` actually compiles to

Read the JDK source and you will see a CAS-shaped thing:

```java
// java.util.concurrent.atomic.AtomicLong
public final long incrementAndGet() {
    return U.getAndAddLong(this, VALUE, 1L) + 1L;
}
```

and `Unsafe.getAndAddLong` is written in Java as a CAS retry loop. **But it is
annotated `@IntrinsicCandidate`.** C2 replaces the entire call with a single machine
instruction. On x86-64 that is `lock xadd`. There is no loop in the compiled code.

That matters for two reasons:

1. **Do not describe `incrementAndGet` as "a CAS loop" in an interview** without the
   caveat. Say: "the Java source is a CAS loop, but it is an intrinsic — C2 emits a
   single `lock xadd` on x86. The retry loop is real for `updateAndGet`, where the
   function is arbitrary."
2. **It explains why `incrementAndGet` and `updateAndGet(x -> x + 1)` perform
   differently** despite computing the same thing. The first is one instruction; the
   second is a Java loop around `lock cmpxchg` that also allocates or captures a lambda.

You will verify this yourself with `-XX:+PrintAssembly` in the Hands-on section. That
requires `hsdis`, and the instructions are there.

### `LongAdder` internals: `Striped64`

`LongAdder extends Striped64`. The relevant fields:

```java
// java.util.concurrent.atomic.Striped64
transient volatile Cell[] cells;   // null until contention is detected
transient volatile long base;      // used while uncontended
transient volatile int cellsBusy;  // a spinlock guarding cells resize

@jdk.internal.vm.annotation.Contended
static final class Cell {
    volatile long value;
    // ... CAS helpers
}
```

The algorithm, in order:

1. `add(x)`: if `cells == null`, try `casBase(base, base + x)`. **If it succeeds, done
   — this is exactly `AtomicLong`, no worse.** This is why an uncontended `LongAdder`
   is roughly as fast as an `AtomicLong`.
2. If the base CAS **fails**, contention has been detected. Allocate the `cells` array
   and go to step 3 from now on.
3. Pick a cell using the thread's **probe** — `ThreadLocalRandom.getProbe()`, a
   per-thread pseudo-random int stored in the `Thread` object. Index is
   `probe & (cells.length - 1)`.
4. CAS that cell's `value`. If it succeeds, done.
5. If that CAS *also* fails, two threads hashed to the same cell. Rehash the probe
   (`advanceProbe`) and/or **double the table**, up to a maximum of the next power of
   two at or above the number of CPUs. Beyond that, growing further would not help,
   because there are not enough cores to have more simultaneous writers.

And the read:

```java
public long sum() {
    Cell[] cs = cells;
    long sum = base;
    if (cs != null) {
        for (Cell c : cs) {
            if (c != null) sum += c.value;
        }
    }
    return sum;
}
```

Three properties of `sum()` you must know:

- It is **O(number of cells)**, which is bounded by the core count. Not free, but not
  large.
- It is **not atomic**. It reads cells one at a time. A concurrent `add` to a cell you
  have already passed is not included. The result is a value that was *plausible* at
  some point during the call, not a snapshot at any instant.
- Therefore **`sum()` must never be used in a compare-and-act**. `if (adder.sum() <
  limit) adder.increment()` is a check-then-act race (Topic 92) with an inaccurate check
  on top.

For metrics — a monotonically increasing counter that a scraper reads once every 15
seconds — every one of those properties is fine. For a rate limiter, none of them are.

### `@Contended`: what it does, and why you probably cannot use it

`Cell` is annotated `@jdk.internal.vm.annotation.Contended`. The JVM responds by
padding the object so that its fields do not share a cache line with anything else.
That is what makes the striping actually work: without padding, `cells[0]` and
`cells[1]` could land in one 64-byte line and you would be back to one contended line
(Topic 96).

**State this plainly, because it is a common source of wasted afternoons:**

- The annotation lives in `jdk.internal.vm.annotation`, which is **not exported** to
  application code. To compile against it you need
  `--add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED`, and you need the
  same flag at runtime.
- Even then, the JVM **ignores** `@Contended` on non-JDK classes unless you also pass
  **`-XX:-RestrictContended`**.
- There is no supported public equivalent. `sun.misc.Contended` from Java 8 is gone.

So for application code, **manual padding is the portable alternative**, and it is what
you should reach for. Topic 96 covers exactly how to write it, how to keep the JIT from
eliminating the padding fields, and how to verify the layout with JOL.

`[JAVA 25]` Compact object headers (`-XX:+UseCompactObjectHeaders`) shrink the object
header, which **changes where your padding fields land relative to the cache-line
boundary**. Any hand-computed padding must be re-verified with JOL under the flag you
actually run with. Do not carry a padding constant across a JDK upgrade without
re-measuring.

### ABA: what CAS actually compares

CAS compares a **value**, not a history. It answers "is this word still what I last
saw?" — not "has this word been unchanged since I last saw it". Those are different
questions, and the gap between them is ABA.

The name is the sequence: the value is **A**, you read it; another thread changes it to
**B** and then back to **A**; your CAS compares against **A**, succeeds, and you
proceed on an assumption that is no longer true.

For a plain counter, ABA is harmless — 5 is 5 regardless of how it got there. For a
**reference**, it is a correctness disaster, because "the same address" does not mean
"the same object in the same state" once memory can be recycled.

**Why Java is usually safe, and exactly when it stops being safe:**

Java's garbage collector will not free an object while you hold a reference to it. So
in a normal lock-free stack, a node you popped cannot be reallocated as a different node
while your local variable still points at it. The classic C++ ABA — free the node,
`malloc` returns the same address for a new node — cannot happen.

ABA returns the moment **you** recycle objects:

- an object pool (a `ReservationSlot` pool, a `ByteBuffer` pool, a Netty `ByteBuf`);
- a ring buffer of reused slots;
- a free-list you manage by hand;
- any "reduce allocation rate" optimisation that reuses instances.

That is the shape in `orderflow`, and it is the trace below.

**The fix: `AtomicStampedReference`.** It pairs the reference with an `int` stamp that
you increment on every change. CAS then compares *both*, so a value that went A → B → A
has a different stamp and the CAS fails correctly.

```java
AtomicStampedReference<Node> head = new AtomicStampedReference<>(initial, 0);

int[] stampHolder = new int[1];
Node observed = head.get(stampHolder);
int stamp = stampHolder[0];
// ... decide what to do ...
boolean won = head.compareAndSet(observed, next, stamp, stamp + 1);
```

The cost is real: internally it holds an immutable `Pair` object, so **every successful
update allocates**. You have traded an allocation for correctness. That is usually the
right trade, and it is also an argument for asking whether you needed the object pool
at all.

---

## Concurrency trace

Two traces, as the brief demands: **ABA**, then a **CAS retry storm**. Both before any
correct code.

### Trace 1 — ABA on `orderflow`'s recycled reservation slots

**The setup.** After Topic 68, someone reduced allocation rate by pooling
`ReservationSlot` objects — small mutable holders for an in-flight order's stock
reservation. The pool is a lock-free Treiber stack: an `AtomicReference<Slot> head`,
where each `Slot` has a `next` field. Pop is a CAS on `head`.

```java
// The pooled pop, written the standard lock-free way. It has an ABA hole.
Slot acquire() {
    Slot observed;
    do {
        observed = head.get();
        if (observed == null) return new Slot();
    } while (!head.compareAndSet(observed, observed.next));   // <-- the hole
    return observed;
}
```

Three slots are in the pool: `X -> Y -> Z`.

| Step | Thread A — `http-exec-17` (order 8812, SKU-1001) | Thread B — `http-exec-42` (order 8813, SKU-1001) | Pool state / outcome |
|---|---|---|---|
| 1 | `observed = head.get()` → **X**. Reads `X.next` → **Y**. | — | head → X → Y → Z |
| 2 | **descheduled by the OS, mid-method** | — | A holds `observed = X`, intends `CAS(head, X, Y)` |
| 3 | — | `acquire()` → CAS(head, X, Y) **succeeds**. Gets slot **X**. | head → Y → Z |
| 4 | — | `acquire()` → CAS(head, Y, Z) **succeeds**. Gets slot **Y**. | head → Z |
| 5 | — | Finishes with X. `release(X)`: sets `X.next = Z`, CAS(head, Z, X) **succeeds**. | head → X → Z. **X is back at the head — the second "A".** |
| 6 | — | Still holds **Y**, actively using it for order 8813. | Y is **not** in the pool |
| 7 | **rescheduled.** Executes `CAS(head, X, Y)`. `head` **is** X. **The CAS succeeds.** | — | **head → Y.** Y is in the pool *and* owned by B. |
| 8 | Returns slot **X** to the caller. Order 8812 writes `slot.quantity = 3`. | — | — |
| 9 | — | Order 8813 writes `slot.quantity = 1` **into Y**. | — |
| 10 | Order 8814 arrives on a third thread, calls `acquire()`, pops **Y** from the head. | — | **Y is now shared by order 8813 and order 8814.** |
| 11 | Order 8814 writes `slot.quantity = 7` into Y. | Order 8813 reads `slot.quantity` → **7**, not 1. | Corrupt |

**Outcome, in business terms.** Order 8813 reserved one unit of SKU-1001. At commit
time it reads its slot and finds a quantity of 7, so it decrements stock by 7. Six
units of a hot product become unsellable with no order behind them. The reverse
interleaving is worse: order 8814's 7-unit reservation is recorded as 1, and you
**oversell six units** of a product you do not have.

No exception is thrown. No log line appears. The only evidence is a nightly stock
reconciliation that does not balance, by a small amount, on high-traffic SKUs only.
It will be blamed on the warehouse for at least one quarter.

**Note precisely why the CAS succeeded at step 7.** It compared `head == X` and that
was true. It could not compare "and `X.next` is still `Y`", because CAS compares one
word. The stack's *shape* changed while the *head pointer's value* returned to what it
was. That is ABA, exactly.

### Trace 2 — the CAS retry storm on the shared request counter

**The setup.** `orderflow` counts requests per endpoint with a shared counter updated
through `updateAndGet`, because someone wanted to also cap it:

```java
private final AtomicLong requestCount = new AtomicLong();

void onRequest() {
    requestCount.updateAndGet(current -> current + 1);   // a real CAS retry loop
}
```

Four request threads on four cores, all arriving at once. The counter is at 100.

| Step | Core 0 — `http-exec-1` | Core 1 — `http-exec-2` | Cores 2, 3 | Cache line state / outcome |
|---|---|---|---|---|
| 1 | reads 100 | reads 100 | both read 100 | line **S**hared on all four cores |
| 2 | `CAS(100, 101)`: RFO — invalidates cores 1, 2, 3 | — | — | line **M** on core 0 |
| 3 | CAS **succeeds** → 101 | `CAS(100, 101)`: RFO — invalidates core 0, pulls the line over | — | line **M** on core 1 |
| 4 | — | CAS **fails** (sees 101, expected 100). Reads 101. Retries. | `CAS(100,101)`: RFO — line moves again | line **M** on core 2 |
| 5 | — | `CAS(101, 102)`: RFO — line moves back to core 1 | CAS **fails**. Reads. Retries. | line **M** on core 1 |
| 6 | — | CAS **succeeds** → 102 | `CAS(101,102)` **fails** — the value moved on again | line ping-ponging |
| 7 | next request: `CAS(102,103)` — RFO again | — | still retrying | — |
| 8 | — | — | core 3 has now failed 3 times, having paid 3 full RFOs | **3 line transfers, 0 increments of useful work** |

**The arithmetic.** With N threads contending, roughly one CAS in N succeeds. Each
attempt — successful or not — costs one cache-line transfer. So the coherence traffic
per successful increment is **O(N)**, while the useful work is **O(1)**.

**Outcome, in business terms.** At 4 threads the counter keeps up. At 16 it costs
measurable CPU. At 64 threads — the Topic 65 load profile with all request threads
active — the aggregate increment throughput is **lower than it was at 8 threads**, and
each request thread spends a meaningful slice of its time inside a metrics counter it
does not care about.

The observable symptom is the confusing one: you scale the pod from 4 CPUs to 16, and
**p99 gets worse**. The flame graph (Topic 78) shows time in `AtomicLong.updateAndGet`
— a method nobody suspects, in instrumentation code nobody profiles. The natural
conclusion, "the JVM does not scale", is wrong, and the correct conclusion is that one
64-byte line is a serial resource shared by sixteen cores.

**And note the second-order failure.** A CAS retry loop is `RUNNABLE` in a thread dump
and it burns CPU. If the arrival rate is high enough that a thread's CAS essentially
never succeeds, you have a **livelock**: full CPU, no progress, no blocked threads, and
nothing in a thread dump that looks wrong. That is Topic 98, and this is where it comes
from.

---

## Example 1 — minimal

Four ways to count. Three of them are wrong for a specific, nameable reason.

```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAdder;

public final class Counters {

    // 1. BROKEN. Not atomic. Recap of Topic 87.
    private volatile long plainVolatile = 0;
    public void bumpVolatile() {
        plainVolatile++;          // read, add, write -- three operations, interleavable
    }

    // 2. Correct, and the right choice at LOW contention or when you need the value often.
    private final AtomicLong atomic = new AtomicLong();
    public void bumpAtomic() {
        atomic.incrementAndGet(); // one `lock xadd` on x86-64. No retry loop.
    }
    public long readAtomic() {
        return atomic.get();      // one load. Exact. Instantaneous.
    }

    // 3. Correct, and MORE EXPENSIVE than (2) for the same result. Avoid.
    public void bumpAtomicTheSlowWay() {
        atomic.updateAndGet(v -> v + 1);   // a real CAS retry loop around `lock cmpxchg`
    }

    // 4. Correct, and the right choice at HIGH write contention.
    private final LongAdder adder = new LongAdder();
    public void bumpAdder() {
        adder.increment();        // CAS the base while uncontended; a per-thread Cell after
    }
    public long readAdder() {
        return adder.sum();       // walks the cells. NOT atomic. NOT instantaneous.
    }
}
```

**Why (1) is broken**, restated because it is the foundation: `volatile` gives you
visibility and ordering, not atomicity (Topic 87). `plainVolatile++` is a read, an add
and a write. Two threads can both read 100, both compute 101, and both write 101. One
increment is lost. `volatile` guarantees they both *see* 100 promptly; it does nothing
to stop them both writing 101.

**Why (3) is worse than (2)** despite looking equivalent: `incrementAndGet` is an
intrinsic that becomes one unconditional instruction. `updateAndGet` cannot be, because
the function is arbitrary — so it is a genuine Java loop around `lock cmpxchg`, it can
retry many times under contention, and it may allocate or capture. Reach for the
specific method when one exists.

**Why (4) is not simply better than (2):**

| | `AtomicLong` | `LongAdder` |
|---|---|---|
| Uncontended write | one `lock xadd` | one `lock cmpxchg` on `base` — comparable |
| Contended write | degrades sharply with cores | scales roughly linearly |
| Read | one load, exact | walks up-to-cores cells, **not atomic** |
| Memory | 16 bytes | 16 bytes **plus** a padded `Cell` per contending thread — each `Cell` occupies a full cache line, so ~64–128 bytes each |
| Supports CAS | yes | **no** |
| Correct for a rate limiter | yes | **no** — `sum()` cannot be used in a check-then-act |

**The rule:** `LongAdder` when writes are frequent, contended, and reads are rare
(metrics). `AtomicLong` when you read often, need an exact instantaneous value, or need
`compareAndSet`.

### The CAS retry loop, written correctly

You will need to write one for any update that is not a plain add. Here is the shape,
with the two things people get wrong marked.

```java
private final AtomicLong stock = new AtomicLong(initialStock);

/** Decrement, but never below zero. Returns false if there was not enough. */
public boolean tryReserve(long quantity) {
    long observed, next;
    do {
        observed = stock.get();
        if (observed < quantity) {
            return false;                    // no CAS attempted -- correct
        }
        next = observed - quantity;
    } while (!stock.compareAndSet(observed, next));
    return true;
}
```

- **The loop body must be pure.** It re-executes on every retry. Anything with a side
  effect — a log line, a metric increment, an allocation you count, sending an event —
  will happen more than once. This is Trap 4 below and it is the most common CAS bug in
  real code.
- **The `return false` is inside the loop but before the CAS.** You must re-read and
  re-check on every attempt; the stock may have been replenished or drained since your
  last read.

The same thing with `updateAndGet` is shorter and has the same purity requirement:

```java
long result = stock.updateAndGet(v -> v >= quantity ? v - quantity : v);
// then compare `result` to what you expected, because updateAndGet gives you no
// success/failure signal of its own.
```

---

## Example 2 — production scenario (on the project spine)

### The constraints

From the Topic 65 baseline:

- 100k products, 1M orders, 5M order lines; a few hot SKUs take a large share.
- k6 open-model arrival, 70/20/10 read/read/place mix.
- 200 Tomcat request threads; the container is sized at 4 CPUs today, and there is a
  proposal to move to 16 to improve p99.
- Recorded p50/p95/p99 committed to `/docs/java/baselines/`.

Two shared counters exist in the request path, and they are different problems:

1. **`requestCount`** — incremented on every request, read once every 15 seconds by the
   Prometheus scrape. Write-heavy, read-rare.
2. **`availableStock` per hot SKU** — an in-memory reservation counter, read and
   conditionally decremented on every order placement. Read-and-write, and the write
   must be conditional.

They need different answers, and the reason is the whole lesson.

### The code that ships and gets slower when you add CPUs

```java
package com.orderflow.metrics;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class RequestMetrics {

    // One AtomicLong per endpoint. Looks perfectly reasonable.
    private final Map<String, AtomicLong> counts = new ConcurrentHashMap<>();
    private final AtomicLong totalRequests = new AtomicLong();     // <-- the hot one
    private final AtomicLong totalErrors   = new AtomicLong();     // <-- adjacent field!

    public void record(String endpoint, boolean error) {
        totalRequests.incrementAndGet();
        if (error) {
            totalErrors.incrementAndGet();
        }
        counts.computeIfAbsent(endpoint, k -> new AtomicLong()).incrementAndGet();
    }

    public long total() {
        return totalRequests.get();
    }
}
```

```java
package com.orderflow.inventory;

@Component
public class InventoryCache {

    private final Map<String, AtomicLong> stock = new ConcurrentHashMap<>();

    /** Reserve stock atomically, or fail. This part is actually correct. */
    public boolean reserve(String sku, long quantity) {
        AtomicLong cell = stock.get(sku);
        if (cell == null) return false;

        long observed, next;
        do {
            observed = cell.get();
            if (observed < quantity) return false;
            next = observed - quantity;
        } while (!cell.compareAndSet(observed, next));
        return true;
    }
}
```

Nothing here is a bug in the ordinary sense. `RequestMetrics.record` is correct.
`InventoryCache.reserve` is a textbook CAS loop and it is correct. Both would pass any
review.

### What actually happens under the Topic 65 baseline

**At 4 CPUs, 200 request threads.** Fine. `totalRequests` is contended, but with four
cores the line transfers are cheap and infrequent enough to disappear into the noise.
p99 matches the recorded baseline.

**Move to 16 CPUs to improve p99. p99 gets worse.**

Three separate effects, and you should be able to separate them:

**Effect 1 — `totalRequests` contention scales with cores, not with load.** Every one
of 16 cores now wants the same 64-byte line for every single request. Coherence traffic
per increment is O(cores). At 4 cores this was invisible; at 16 it is a measurable slice
of every request.

**Effect 2 — `totalRequests` and `totalErrors` are almost certainly in the same cache
line.** They are two adjacent reference fields in one object; the `AtomicLong` objects
they point at were allocated one after another, so they are very likely adjacent in
eden. Two 16-byte objects fit comfortably inside one 64-byte line. Every request
increments `totalRequests` and thereby **invalidates `totalErrors` for every other
core**, even though errors are rare and logically independent. That is **false
sharing**, and it is Topic 96 — this document is where you first meet it.

**Effect 3 — the inventory CAS loop retries more.** On the hot SKU, more cores means
more simultaneous `compareAndSet` attempts, so a higher failure rate per attempt, so
more iterations of the loop, so more RFOs. The CPU time shows up as `RUNNABLE` threads
in `InventoryCache.reserve` and looks like "the inventory code is slow".

The flame graph shows time in `AtomicLong` methods. The natural conclusion is "the JVM
does not scale". The correct conclusion is that three 64-byte cache lines have become
the bottleneck for a sixteen-core machine.

### The fix, counter by counter

**`totalRequests` and `totalErrors` → `LongAdder`.** Write-heavy, read every 15
seconds by a scrape. This is precisely the workload `LongAdder` was designed for.

```java
package com.orderflow.metrics;

import java.util.concurrent.atomic.LongAdder;

@Component
public class RequestMetrics {

    private final Map<String, LongAdder> counts = new ConcurrentHashMap<>();
    private final LongAdder totalRequests = new LongAdder();
    private final LongAdder totalErrors   = new LongAdder();

    public void record(String endpoint, boolean error) {
        totalRequests.increment();
        if (error) {
            totalErrors.increment();
        }
        counts.computeIfAbsent(endpoint, k -> new LongAdder()).increment();
    }

    /**
     * Read by the Prometheus scrape, every 15 seconds.
     *
     * sum() is O(cells) and NOT atomic: a concurrent increment to a cell already
     * visited is not included. For a monotonic counter sampled on an interval that
     * is exactly right -- the missed increment appears in the next scrape.
     *
     * It would be WRONG for a rate limiter, a quota check, or anything that
     * compares the value and then acts on it.
     */
    public long total() {
        return totalRequests.sum();
    }
}
```

The doc comment is not decoration. **Every `LongAdder.sum()` in a codebase should carry
one sentence explaining why an inexact, non-atomic read is acceptable there.** It is the
single most effective way to stop the next engineer using it in a quota check.

**Better still: do not write this class at all.** Micrometer's `Counter` is already
backed by a striped adder. In production `orderflow` you register a
`Counter.builder("orderflow.requests").register(registry)` and the concurrency problem
is someone else's. Say that in an interview — "and then I would check whether the
metrics library already solved it, because Micrometer's counters are striped" — because
it shows you know when *not* to write concurrency code.

**`availableStock` → keep the `AtomicLong`, but bound the retries and reconsider the
design.**

`LongAdder` is the **wrong** answer here, and knowing why is the point of this example.
The inventory update is **conditional** — "decrement only if there is enough". That
requires compare-and-set semantics on a single authoritative value. `LongAdder` has no
`compareAndSet`, and `sum()` is not atomic, so `if (adder.sum() >= q) adder.add(-q)` is
a check-then-act race that will oversell.

What to do instead, in order:

```java
@Component
public class InventoryCache {

    private final Map<String, AtomicLong> stock = new ConcurrentHashMap<>();
    private final LongAdder casRetries = new LongAdder();   // instrument the retries

    private static final int MAX_ATTEMPTS = 64;

    public boolean reserve(String sku, long quantity) {
        AtomicLong cell = stock.get(sku);
        if (cell == null) return false;

        for (int attempt = 0; attempt < MAX_ATTEMPTS; attempt++) {
            long observed = cell.get();
            if (observed < quantity) {
                return false;
            }
            if (cell.compareAndSet(observed, observed - quantity)) {
                return true;
            }
            casRetries.increment();
            Thread.onSpinWait();          // <-- see below
        }
        // Bounded. We refuse rather than spin forever. This is the anti-livelock guard.
        throw new ContentionTimeoutException(sku, MAX_ATTEMPTS);
    }
}
```

Three deliberate changes:

1. **The loop is bounded.** An unbounded CAS loop under sustained contention is a
   livelock waiting to happen: a thread that always loses burns CPU forever and the
   thread dump shows it `RUNNABLE`, which looks healthy. Bounding it converts an
   invisible hang into a visible, retryable error — the same principle as Topic 94's
   `tryLock(timeout)`.
2. **`Thread.onSpinWait()`** compiles to the x86 `PAUSE` instruction (and `YIELD` on
   ARM). It tells the CPU "I am in a spin loop", which reduces power and, more
   importantly, avoids a memory-order violation pipeline flush when the loop exits. It
   costs nothing and it is the correct thing to put in any spin.
3. **The retry count is instrumented** with a `LongAdder` — which is itself the correct
   use of `LongAdder`, and a nice illustration that the two classes solve different
   problems in the same method. Graph `casRetries` per second. **A rising retry rate is
   your early warning for this entire class of problem**, and almost nobody instruments
   it.

**And the design question you should raise before any of this.** The durable inventory
invariant is enforced in the database (Topic 52: an atomic conditional `UPDATE ... WHERE
available >= ?`). This in-memory counter is a *fast rejection path*, not the source of
truth. If it is contended enough to matter, the honest options are:

- **Shard the hot SKU** into K sub-counters and reserve from a randomly chosen one,
  falling back to the database when a shard is empty. This is `LongAdder`'s idea applied
  to a conditional decrement, and it is what a flash-sale system actually does.
- **Remove it** and rely on the database's conditional update, accepting a round trip
  per placement. At 10% of 400 rps that is 40 rps of conditional updates, which Postgres
  will not notice.

Say the second one out loud in an interview. "The best fix for a contended counter is
often to establish that you did not need the counter" is the answer that separates a
senior engineer from someone who knows the API surface.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a contended `AtomicLong` whose throughput **drops** as cores are added

**Wrong:** one shared `AtomicLong` incremented on every request, in a service you then
scale up vertically.

**Exact symptom:** you move the pod from 4 CPUs to 16 to improve p99. **p99 gets
worse.** Throughput per core falls. A flame graph (Topic 78) shows a meaningful and
growing slice in `AtomicLong.incrementAndGet` or `Unsafe.getAndAddLong`. On Linux,
`perf stat` shows `cache-misses` rising super-linearly with core count while the
request rate is flat. In JMH, the `@Threads` scaling curve is not flat — it **descends**
past a few threads.

**Root cause:** the `lock`-prefixed instruction requires exclusive ownership of the
cache line holding the counter. N cores means N read-for-ownership transactions per N
increments, and the line is a serial resource. The coherence traffic scales with core
count while the useful work does not. **Adding cores adds contenders for a resource
that cannot be parallelised.**

**Fix:** `LongAdder`, if the value is read rarely. It gives each contending thread its
own `@Contended` cell, so threads write to different lines and there is no transfer at
all. Verify with the JMH harness in the Measurement section; the fix is proven by the
scaling curve turning from descending to rising.

**When `LongAdder` is not the fix:** when you need `compareAndSet`, an exact
instantaneous read, or a conditional update. Then the fix is to reduce the number of
threads touching that value — shard it by key, or move the state off the hot path
entirely.

---

### Trap 2 — ABA on a CAS'd reference to a recycled object

**Wrong:** the lock-free object pool from Trace 1 — `AtomicReference<Slot> head`, with
slots returned to the pool for reuse.

**Exact symptom:** the pool occasionally hands the same `Slot` instance to two threads,
or loses slots entirely (a slow leak in pool size). In `orderflow` this shows as a
nightly stock reconciliation that is off by a small number of units, only on high-traffic
SKUs, never reproducible, and never in a test. No exception. No log. The bug survives
for months because every individual line of the pool code is correct.

**Root cause:** CAS compares one word. It answers "is `head` still `X`?" and cannot
answer "has `head` been continuously `X`?". Another thread popped `X`, popped `Y`, and
pushed `X` back, so `head` is `X` again with a different `next`. The stale CAS succeeds
against a stack whose shape changed underneath it.

Java's GC normally prevents this by keeping a referenced object alive, so a popped node
cannot be reallocated as a different node. **Your object pool deliberately defeats that
protection** — that is what a pool is. The bug was introduced by an allocation-rate
optimisation, not by the concurrency code.

**Fix, in order of preference:**

1. **Delete the pool.** Ask what it bought. Measure the allocation rate with and without
   (Topic 68). Young-generation allocation in HotSpot is a pointer bump and collection
   of dead young objects is nearly free; pools frequently cost more in complexity and
   bugs than they save in GC. This is the right answer more often than people expect.
2. **`AtomicStampedReference`** — CAS the reference and an `int` stamp together, so
   A → B → A has a different stamp and the CAS correctly fails. Cost: an internal `Pair`
   allocation per successful update, which partly defeats the pool's purpose. Say that
   trade out loud.
3. **Use a JDK structure that already solves it.** `ConcurrentLinkedQueue` and
   `ConcurrentLinkedDeque` handle their internal ABA concerns for you. If you need a
   pool, back it with one of those rather than hand-writing a Treiber stack.

**How to catch it in review:** any `compareAndSet` on an `AtomicReference` whose target
objects can be **reused** is an ABA candidate. Ask one question: "can this exact object
instance be removed from the structure and put back?" If yes, you need a stamp.

---

### Trap 3 — a non-pure function passed to `updateAndGet` / `accumulateAndGet`

**Wrong:**

```java
inventoryCounter.updateAndGet(current -> {
    log.info("decrementing stock for {}", sku);       // <-- side effect
    metrics.increment("stock.decrement");             // <-- side effect
    auditQueue.offer(new StockEvent(sku, current));   // <-- side effect
    return current - 1;
});
```

**Exact symptom:** under load, duplicate audit events and inflated metrics — the
`stock.decrement` counter reads higher than the number of decrements that actually
happened, sometimes by 30% or more, and the discrepancy grows with core count.
Duplicate log lines with identical content and identical timestamps to the millisecond.
Downstream consumers of `auditQueue` process the same event several times.

**Root cause:** the lambda passed to `updateAndGet` is the body of a **CAS retry loop**.
The Javadoc states the function must be side-effect-free, because it **is re-applied on
every failed attempt**. Under contention it may run many times per successful update.
This is not a bug in `updateAndGet`; it is the documented contract, and it is the single
most common misuse of the atomic API.

**Fix:** the function computes and returns a value, and nothing else. Do the side
effects **after** the loop, once, using the returned result.

```java
long after = inventoryCounter.updateAndGet(current -> current - 1);   // pure
log.info("decremented stock for {} to {}", sku, after);
metrics.increment("stock.decrement");
auditQueue.offer(new StockEvent(sku, after));
```

**The general rule, which also applies to `ConcurrentHashMap.compute`/`merge` from
Topic 92:** any function you hand to a concurrent collection or an atomic may be invoked
more than once, on any thread, with the lock or the line held. Keep it short, pure, and
free of anything that can block.

---

### Trap 4 — `LongAdder.sum()` used as if it were exact

**Wrong:**

```java
private final LongAdder inFlight = new LongAdder();

public void handle(Request r) {
    if (inFlight.sum() >= MAX_IN_FLIGHT) {        // <-- check
        throw new TooManyRequestsException();
    }
    inFlight.increment();                          // <-- act
    try { process(r); } finally { inFlight.decrement(); }
}
```

**Exact symptom:** the in-flight limit is exceeded, sometimes substantially. You set
`MAX_IN_FLIGHT` to 100 and observe 140 concurrent requests in a trace. The excess grows
with core count and with arrival rate — exactly when the limit matters most. The service
falls over at the load the limiter was installed to prevent.

**Root cause — two independent defects, and you should name both:**

1. **Check-then-act.** Even with a perfectly exact counter, `sum()` and `increment()`
   are two operations. Twenty threads can all read 99 and all proceed. This is Topic 92's
   race, unchanged.
2. **`sum()` is not atomic.** It walks the cells in order, adding as it goes. An
   increment to a cell it has already passed is not counted. So the value is not even a
   correct snapshot of any single instant — it is a plausible-looking number that was
   never true.

**Fix:** use the primitive designed for the job. A concurrency limit is a **permit
count**, which is a `Semaphore` (Topic 97), and it is the bulkhead pattern (Topic 111).

```java
private final Semaphore permits = new Semaphore(MAX_IN_FLIGHT);

public void handle(Request r) {
    if (!permits.tryAcquire()) {
        throw new TooManyRequestsException();
    }
    try { process(r); } finally { permits.release(); }   // release ALWAYS in finally
}
```

`tryAcquire` is a single atomic operation that both checks and takes. There is no
window. If you must use a counter, use `AtomicLong` with a bounded `compareAndSet` loop
so the check and the act are one instruction.

**The rule:** `LongAdder` is for values you **accumulate and report**. The moment a
value is used to make a decision, you need `AtomicLong` or a `Semaphore`.

---

### Trap 5 — an unbounded CAS retry loop

**Wrong:**

```java
while (!cell.compareAndSet(observed, next)) {
    observed = cell.get();
    next = compute(observed);
}
// no bound, no backoff, no onSpinWait
```

**Exact symptom:** CPU pegged at 100% on all cores. Throughput near zero. **A thread
dump shows every thread `RUNNABLE`, no `BLOCKED`, no `WAITING`, and no
`Found one Java-level deadlock`.** Every diagnostic you reach for says the service is
healthy and busy. Restarting fixes it until the load returns.

**Root cause:** this is a **livelock** (Topic 98). Threads are making attempts and no
attempt completes. Under enough contention an unlucky thread can lose repeatedly for an
unbounded time — the CAS is lock-*free* (some thread progresses) but not **wait-free**
(no guarantee *this* thread progresses). The progress guarantee you have is weaker than
the one you assumed.

It is worse than a deadlock in one specific way: **it is invisible to every automatic
detector.** `findDeadlockedThreads()` returns `null`. There is no cycle. The threads are
genuinely running.

**Fix, all three together:**

1. **Bound the attempts.** After N tries, fail with a retryable error. N of 32–128 is a
   reasonable starting point; measure the real distribution and set it above the
   99.9th percentile.
2. **`Thread.onSpinWait()`** inside the loop — the `PAUSE`/`YIELD` hint. Free, and it
   both reduces power and avoids a pipeline flush on loop exit.
3. **Back off with jitter** if you retry the whole operation at a higher level, or you
   will have built a retry storm (Topic 111).

And then ask the design question: if the loop needs a bound, the contention is high
enough that the shared value is the wrong design. Shard it.

---

### Trap 6 — reaching for `Atomic*` when the invariant spans two fields

**Wrong:**

```java
private final AtomicLong available = new AtomicLong(100);
private final AtomicLong reserved  = new AtomicLong(0);

public void reserve(long q) {
    available.addAndGet(-q);     // atomic
    reserved.addAndGet(q);       // also atomic
}                                // TOGETHER: not atomic
```

**Exact symptom:** a reader calling `available.get() + reserved.get()` occasionally sees
a total that is not 100 — short by exactly one order's quantity. In `orderflow` this
surfaces as a stock-summary endpoint that intermittently reports a total that does not
match the sum of its parts, which support tickets describe as "the dashboard is wrong
sometimes".

**Root cause:** each field's update is atomic; the **pair** is not. There is no
double-word CAS in Java. Between the two `addAndGet` calls the object is in a state that
violates its own invariant, and any concurrent reader can observe it. This is exactly
the torn read that `StampedLock`'s optimistic mode guards against (Topic 94).

**Fix, in order of preference:**

1. **Make the pair a single immutable object and CAS the reference.**

   ```java
   record Stock(long available, long reserved) { }
   private final AtomicReference<Stock> state = new AtomicReference<>(new Stock(100, 0));

   public void reserve(long q) {
       state.updateAndGet(s -> new Stock(s.available() - q, s.reserved() + q));
   }
   ```

   One CAS, one invariant, always consistent. Cost: one allocation per update — usually
   fine, since it is a short-lived young object (Topic 68), and it is exactly what
   `AtomicStampedReference` does internally.

2. **Pack both counters into one `long`** if the ranges allow it — high 32 bits
   available, low 32 bits reserved — and use a single `AtomicLong` with bit arithmetic.
   Zero allocation, no tearing. This is precisely what
   `ReentrantReadWriteLock` does with its 16/16 split (Topic 94). Cost: readability, and
   a hard maximum you must document.

3. **Use a lock.** If the invariant is complex or spans more than two words, a
   `ReentrantLock` around both updates is correct, obvious, and probably fast enough.
   Not every concurrency problem should be solved lock-free.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print
output and call it real. What I give you precisely is what to look for and what each
possible result means.

### Setup

```bash
mkdir -p ~/java-lab/95 && cd ~/java-lab/95
java --version
nproc 2>/dev/null || sysctl -n hw.ncpu      # how many cores you actually have
```

Write that core count down. Every scaling curve in this document is relative to it, and
a curve that flattens at exactly your core count is telling you something different from
one that flattens at four.

### Proof 1 — `volatile` is not atomic (the baseline you are building on)

```java
public class VolatileIsNotAtomic {
    static volatile long counter = 0;

    public static void main(String[] args) throws Exception {
        Thread[] ts = new Thread[8];
        for (int i = 0; i < ts.length; i++) {
            ts[i] = new Thread(() -> {
                for (int j = 0; j < 1_000_000; j++) counter++;
            });
            ts[i].start();
        }
        for (Thread t : ts) t.join();
        System.out.println("expected 8000000, got " + counter);
    }
}
```

**What to look for:**

| What you see | What it means |
|---|---|
| A number well below 8,000,000 | Expected. Each lost increment is one interleaving of read-add-write. |
| Exactly 8,000,000 | Possible on a very fast machine with short threads that barely overlap. Raise the iteration count and the thread count until it fails. **The absence of a failure is not evidence of correctness** — that is Topic 99's whole argument. |
| A different number every run | Correct and expected. This is what non-determinism looks like. |

Now change `volatile long counter` to `AtomicLong counter` with `incrementAndGet()` and
re-run. It must be exact, every time.

### Proof 2 — see `lock xadd` with your own eyes

This is the one that settles the mechanical statement. It needs `hsdis`, the HotSpot
disassembler plugin.

```bash
# Check whether your JDK build already has it:
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -version 2>&1 | head -5
```

**What to look for:**

| What you see | What it means |
|---|---|
| Assembly, or a normal version banner | `hsdis` is present. Proceed. |
| `Could not load hsdis-<arch>.so` / `.dylib` | Not present. You must build or obtain it — see below. |

Recent JDK builds can build it from source: `make build-hsdis` in an OpenJDK checkout
produces the library, which you place in `$JAVA_HOME/lib/`. Some vendor builds ship it.
**If you cannot obtain it, skip to Proof 3 — do not fabricate the result.** The claim
is independently checkable in the OpenJDK source (`macroAssembler_x86.cpp`, the
`atomic_add` and `cmpxchg` paths) and in the intrinsic list, and "I could not run
PrintAssembly on this machine" is a perfectly good answer.

With `hsdis` present:

```java
import java.util.concurrent.atomic.AtomicLong;

public class ShowAsm {
    static final AtomicLong c = new AtomicLong();
    static long sink;

    public static void main(String[] args) {
        for (int i = 0; i < 200_000; i++) {     // force C2 compilation
            sink = hot();
        }
        System.out.println(sink);
    }

    static long hot() { return c.incrementAndGet(); }
}
```

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,ShowAsm::hot \
     ShowAsm.java 2>&1 | grep -iE 'lock|xadd|cmpxchg' | head -20
```

**What to look for:**

| What you see | What it means |
|---|---|
| `lock xaddq` (or `lock xadd`) | Confirms the mechanical statement. `incrementAndGet` is one instruction, not a loop. |
| `lock cmpxchg` inside a small loop | You are on a JDK/architecture where the add is not intrinsified this way, **or** you compiled `updateAndGet` by mistake. Check which method you printed. |
| `ldaddal` / `ldxr` + `stxr` | You are on ARM (Apple Silicon). `ldaddal` is the LSE atomic add; `ldxr`/`stxr` is the LL/SC form. Both are correct answers for that architecture. |

Now change `hot()` to `return c.updateAndGet(v -> v + 1);` and re-run. **You should see
a `cmpxchg` inside a loop.** That contrast — one instruction versus a loop — is the
proof that these two methods are not the same thing, and it is worth the effort of
getting `hsdis` working once in your life.

### Proof 3 — `LongAdder` really is striped

```java
import java.util.concurrent.atomic.LongAdder;
import java.lang.reflect.Field;

public class AdderInternals {
    static final LongAdder adder = new LongAdder();

    public static void main(String[] args) throws Exception {
        System.out.println("before contention: " + describe());

        Thread[] ts = new Thread[Runtime.getRuntime().availableProcessors() * 2];
        for (int i = 0; i < ts.length; i++) {
            ts[i] = new Thread(() -> { for (int j = 0; j < 2_000_000; j++) adder.increment(); });
            ts[i].start();
        }
        for (Thread t : ts) t.join();

        System.out.println("after contention:  " + describe());
        System.out.println("sum = " + adder.sum());
    }

    static String describe() throws Exception {
        Class<?> striped64 = Class.forName("java.util.concurrent.atomic.Striped64");
        Field cellsField = striped64.getDeclaredField("cells");
        cellsField.setAccessible(true);
        Object[] cells = (Object[]) cellsField.get(adder);
        if (cells == null) return "cells = null (uncontended, using base only)";
        long nonNull = java.util.Arrays.stream(cells).filter(java.util.Objects::nonNull).count();
        return "cells.length = " + cells.length + ", non-null = " + nonNull;
    }
}
```

```bash
java --add-opens java.base/java.util.concurrent.atomic=ALL-UNNAMED AdderInternals.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `cells = null` before, a sized array after | **The whole design, observed.** `LongAdder` starts as an `AtomicLong` on `base` and only stripes once a CAS actually fails. |
| `cells.length` a power of two, at or below the next power of two above your core count | Correct. The table stops growing there because more cells cannot help with fewer cores. |
| `sum` exactly equal to threads × iterations | Correct — `sum()` after all writers have joined is exact. It is only inexact **concurrently**. |
| An `InaccessibleObjectException` | You omitted `--add-opens`. Note that this is reflection into JDK internals purely for teaching; never do it in production code. |

Also confirm the `@Contended` annotation is really there:

```bash
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/concurrent/atomic/Striped64.java' -d ~/jdk-src
grep -n "Contended" ~/jdk-src/java.base/java/util/concurrent/atomic/Striped64.java
```

**What to look for:** a line reading `@jdk.internal.vm.annotation.Contended` immediately
above `static final class Cell`. That single annotation is why the striping works
rather than just moving the contention around, and it is Topic 96's whole subject.

### Proof 4 — reproduce ABA deliberately

```java
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.AtomicStampedReference;

public class AbaDemo {

    static final AtomicReference<String> ref = new AtomicReference<>("A");
    static final AtomicStampedReference<String> stamped = new AtomicStampedReference<>("A", 0);

    public static void main(String[] args) throws Exception {

        // --- unstamped: the CAS succeeds even though the value went A -> B -> A ---
        String observed = ref.get();                     // sees "A"
        Thread other = new Thread(() -> {
            ref.compareAndSet("A", "B");
            ref.compareAndSet("B", "A");                 // back to "A"
        });
        other.start();
        other.join();
        System.out.println("plain CAS succeeded: " + ref.compareAndSet(observed, "C"));

        // --- stamped: the same sequence, and the CAS correctly FAILS ---
        int[] holder = new int[1];
        String observed2 = stamped.get(holder);
        int stamp = holder[0];
        Thread other2 = new Thread(() -> {
            int[] h = new int[1];
            String v = stamped.get(h);
            stamped.compareAndSet(v, "B", h[0], h[0] + 1);
            stamped.compareAndSet("B", "A", h[0] + 1, h[0] + 2);
        });
        other2.start();
        other2.join();
        System.out.println("stamped CAS succeeded: "
                + stamped.compareAndSet(observed2, "C", stamp, stamp + 1));
    }
}
```

**What to look for:**

| What you see | What it means |
|---|---|
| `plain CAS succeeded: true` | **ABA, demonstrated.** The value returned to `A`, so the CAS could not tell that anything happened. |
| `stamped CAS succeeded: false` | The fix, demonstrated. The stamp advanced twice, so the CAS correctly refused. |
| Both true | Check that the second thread actually ran and that you joined it. The demo is deterministic by design; if it is not behaving, the sequencing is wrong, not the JDK. |

This demo is deliberately deterministic — `join()` everywhere — because the point is to
**show the mechanism**, not to catch a race. The realistic version is Trace 1's pool,
which you will build in the medium exercise.

### Proof 5 — measure the scaling curve

Do **not** use `System.nanoTime()`. The correct harness is in the Measurement section
below, and running it is the Failure drill. Do that before forming any opinion about
which counter is faster.

---

## Failure drill

**Assignment (Topic 95, from the master plan's drill map):** `AtomicLong` versus
`LongAdder` as `orderflow`'s request counter under 64 threads, measured with JMH
`@Threads`. Explain the crossover point.

Budget two hours. JMH runs are slow; start it and go and do something else.

### Part A — the harness

`CounterBench.java`:

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAdder;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)                 // ONE shared instance -- this is the whole point
@Fork(3)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Threads({1, 2, 4, 8, 16, 32, 64})      // the CURVE is the result
public class CounterBench {

    private AtomicLong atomic;
    private LongAdder adder;
    private volatile long plain;         // deliberately racy: the "how fast could it be" ceiling

    @Setup(Level.Iteration)
    public void setup() {
        atomic = new AtomicLong();
        adder  = new LongAdder();
        plain  = 0;
    }

    @Benchmark
    public void atomicIncrement() {
        atomic.incrementAndGet();
    }

    @Benchmark
    public void adderIncrement() {
        adder.increment();
    }

    /** The ceiling: correct-value-not-required, to show what the coherence traffic costs. */
    @Benchmark
    public void racyIncrement() {
        plain++;                         // WRONG on purpose. Never ship this.
    }

    // --- the read side, which is where LongAdder pays ---

    @Benchmark
    public long atomicRead() {
        return atomic.get();
    }

    @Benchmark
    public long adderRead() {
        return adder.sum();
    }

    /** A realistic mixed workload: many writers, one reader every so often. */
    @Benchmark
    public void atomicMixed(Blackhole bh) {
        atomic.incrementAndGet();
        bh.consume(atomic.get());
    }

    @Benchmark
    public void adderMixed(Blackhole bh) {
        adder.increment();
        bh.consume(adder.sum());
    }
}
```

Every annotation is doing specific work. Be able to say what each is for:

| Annotation | Why it is there |
|---|---|
| `@State(Scope.Benchmark)` | **One shared instance across all threads.** With `Scope.Thread` each thread gets its own counter, there is no contention, and the benchmark measures nothing at all. This single line is the difference between a contention benchmark and a fast-path benchmark. |
| `@Threads({1,2,4,8,16,32,64})` | JMH reruns the whole benchmark at each thread count. The scaling curve is the deliverable. |
| `@BenchmarkMode(Throughput)` | Operations per second. For a scaling question, throughput is the right mode; average latency per op hides the aggregate collapse. |
| `@Fork(3)` | Three separate JVMs. Disagreement between forks means JIT profile pollution (Topic 74) and you would never see it in one fork. |
| `@Warmup(5)` | Lets C2 compile the benchmark body. Without it you measure the interpreter. |
| `@Setup(Level.Iteration)` | Fresh counters each iteration, so a `LongAdder` that has already grown its cell table does not carry that advantage into the next measurement. |

```bash
mvn -q archetype:generate -DinteractiveMode=false \
    -DarchetypeGroupId=org.openjdk.jmh -DarchetypeArtifactId=jmh-java-benchmark-archetype \
    -DgroupId=com.orderflow -DartifactId=bench -Dversion=1.0
# put CounterBench.java in src/main/java/com/orderflow/bench/
mvn -q clean package
java -jar target/benchmarks.jar CounterBench -rf json -rff results.json
```

### Part B — write down your predictions before you run it

Do this. It is the part that makes the exercise teach you something.

Predict, for your specific core count:

1. Which is faster at **1 thread**, and by roughly what factor?
2. At what thread count does the other one overtake — the **crossover**?
3. Does `atomicIncrement` throughput at 64 threads exceed its throughput at 8?
4. How much slower is `adderRead` than `atomicRead`, and does that gap depend on thread
   count? Why?
5. What does `racyIncrement` do at 64 threads, and what does that tell you about how
   much of the atomic's cost is coherence versus the instruction itself?

### Part C — what to expect, and how to explain the crossover

I do not have a JVM and will not invent your numbers. What I can give you is the shape
to expect and the reasoning for each part of it. **If your measurements disagree, trust
your measurements** and work out why — different core counts, socket counts, JDK builds
and CPU architectures all move these curves, and Apple Silicon in particular behaves
differently from x86-64.

**The shape to expect:**

| Threads | `AtomicLong` | `LongAdder` | Why |
|---|---|---|---|
| 1 | fastest of the two | **slower** | `LongAdder` costs a null check on `cells` plus a `casBase`. `AtomicLong` is one `lock xadd`. There is no contention to amortise the extra work. |
| 2–4 | still competitive, starting to flatten | catching up, then ahead | Contention begins. `LongAdder` allocates its cell table on the first failed base CAS and stops sharing a line. |
| **the crossover** | — | — | Wherever the cost of cache-line transfers exceeds `LongAdder`'s extra indirection. Typically low — often between 2 and 8 threads on a normal machine. **The crossover point is the answer the drill wants.** |
| 8–64 | **flat, then descending** | rising, then flattening near core count | The atomic's line is a serial resource; more contenders is more transfers. The adder's cells are separate lines, so threads genuinely run in parallel until it runs out of cores. |
| beyond core count | continues to degrade | flattens — cannot exceed the machine | Cell table growth stops at the core count, because more cells cannot help with fewer cores. |

**Why the crossover exists, in one paragraph you should be able to say out loud:**

> At one thread there is no contention, so the cache line is always in this core's L1
> and the atomic instruction costs a few cycles. `LongAdder` does strictly more work for
> the same result — check whether cells exist, CAS the base — so it loses. As threads
> are added, the atomic's cost stops being the instruction and starts being the cache
> line transfer, which grows with the number of contenders. `LongAdder`'s cost does not
> grow, because after the first contention it stops sharing a line at all: each thread
> hashes to its own `@Contended` cell. The crossover is the thread count at which the
> transfer cost exceeds `LongAdder`'s fixed extra work — and because a cross-core
> transfer is one to two orders of magnitude more expensive than an L1 hit, that
> crossover comes early.

**Then the reversal on the read side.** `atomicRead` is one load and should be
essentially free and flat. `adderRead` walks the cell array, so it costs O(cells) —
bounded by the core count — and it gets **relatively worse** as the table grows. That is
the trade, stated as a measurement rather than as an opinion, and it is why the
`*Mixed` benchmarks exist: at a high enough read:write ratio, `AtomicLong` wins again.

**Find your own read:write crossover.** Vary the mixed benchmarks so the read happens
every Nth operation, for N in 1, 10, 100, 1000. Report the N at which `LongAdder`
overtakes `AtomicLong` at 64 threads. **That number is your actual decision rule**, and
you now have it for your hardware rather than from a blog.

### Part D — the same experiment on `orderflow` under the Topic 65 baseline

A microbenchmark is not a service. Prove it end to end.

1. Confirm the recorded baseline reproduces within ±10%.
2. Deploy `RequestMetrics` with `AtomicLong`. Run k6. Record p50/p95/p99, CPU, and a
   flame graph (Topic 78).
3. **Scale the container from 4 CPUs to 16** with no other change. Re-run. Record the
   same things. Expect p99 to improve less than you would predict, and possibly to get
   worse.
4. Swap to `LongAdder`. Re-run at both 4 and 16 CPUs.
5. Produce a four-cell table: {AtomicLong, LongAdder} × {4 CPU, 16 CPU}, with p99 and
   the flame-graph share attributed to counter methods.

**The finding you are looking for** is not "LongAdder is faster". It is: **the benefit is
zero at 4 CPUs and material at 16.** The fix's value is a function of core count, which
means "should we use `LongAdder`" has no context-free answer — and being able to say
that, with your own numbers, is the senior answer.

### Part E — the honest negative result

Very plausibly, on `orderflow` at 400 rps, **neither counter is on the critical path at
all** and the difference is unmeasurable end to end.

**Write that up if it is what you find.** "I measured it, the microbenchmark shows a
large difference at 64 threads, and it does not move p99 in the real service because the
request path is dominated by the database round trip" is a *better* deliverable than a
change you cannot justify. It is also the answer that stops the team from spending a
sprint on micro-optimisation.

Then answer the follow-up: **at what request rate or core count would this start to
matter?** Extrapolate from your curve. That number is the trigger you put in the ticket
you close as "not now".

---

## Measurement

### The instrument for each claim

| Claim | Instrument that makes it falsifiable |
|---|---|
| "`incrementAndGet` is one `lock xadd`" | `-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=print,...` with `hsdis` |
| "`updateAndGet` is a real retry loop" | The same, comparing the two disassemblies |
| "Throughput falls as cores are added" | JMH `@Threads({1,2,4,8,16,32,64})` with `@State(Scope.Benchmark)` |
| "It is cache-line transfers, not the instruction" | `perf stat -e cache-misses,LLC-load-misses` on **Linux**; the `racyIncrement` arm as a proxy elsewhere |
| "`LongAdder` really striped" | Reflection on `Striped64.cells` (Proof 3) |
| "The retry rate is the early warning" | A `LongAdder` counting failed CAS attempts, exported as a metric |
| "ABA is possible here" | A deterministic reproduction (Proof 4), then jcstress (Topic 99) for the real structure |
| "The fix helped the service" | **The recorded Topic 65 p50/p95/p99, re-run identically** |

### The standing rule: a naive `System.nanoTime()` loop is wrong

You will want to settle "`AtomicLong` versus `LongAdder`" like this. Every number it
produces is meaningless.

```java
// DO NOT DO THIS.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    counter.incrementAndGet();
}
System.out.println((System.nanoTime() - start) / 10_000_000 + " ns/op");
```

Five reasons, and you cannot tell which one is lying:

1. **It is single-threaded, so there is no contention.** You have measured the L1-hit
   fast path, which is the one case where the two classes are closest and the one case
   that never happens in production. **This is the reason that matters most here** — the
   entire subject of this document is invisible to this benchmark.
2. **Dead-code elimination.** If the counter is never read afterwards, C2 may prove the
   loop has no observable effect and delete it.
3. **Loop optimisations.** C2 can unroll, and for `LongAdder`'s uncontended path it may
   hoist the `cells == null` check out of the loop, changing the ratio between the arms.
4. **Cold JIT and on-stack replacement.** Your average blends interpreted, C1 and C2
   execution in a ratio set by the loop count you happened to choose.
5. **The two arms are optimised by different amounts**, so the comparison is not
   comparing what you think it is. This is the same failure shape as Topic 80's
   direct-versus-heap buffer benchmark.

**Topic 77 is the full treatment.** The correct harness is in the Failure drill.

### `perf` — and the honest note about macOS

On Linux you can watch the coherence traffic that this whole document is about:

```bash
perf stat -e cache-misses,cache-references,LLC-load-misses,LLC-store-misses \
  -- java -jar target/benchmarks.jar CounterBench.atomicIncrement -t 64

# The really informative one, if your CPU supports it:
perf c2c record -- java -jar target/benchmarks.jar CounterBench.atomicIncrement -t 64
perf c2c report
```

`perf c2c` — "cache to cache" — is purpose-built for exactly this: it identifies the
specific cache lines suffering **HITM** (hit-modified) events, meaning one core read a
line that another core had modified. That is the direct measurement of the pathology in
this document, and if you ever have a Linux box, it is worth an hour.

**On macOS, `perf` does not exist, and there is no equivalent that exposes these
counters to a JVM.** Your options:

- Run it in a Linux container. Note that PMU access usually needs `--privileged`, and on
  Apple Silicon under virtualisation the hardware counters may be **unavailable
  entirely** — you can get a container and still not get counters.
- `xctrace record --template 'Time Profiler'` gives CPU-time attribution, which will
  show you that time is going into `AtomicLong` methods. It will **not** tell you that
  the cause is cache-line transfers.
- Accept the JMH scaling curve as your measurement, and treat cache coherence as the
  *explanation* for the curve rather than something you have independently observed.

**Say this out loud rather than pretending.** "The scaling curve is consistent with
cache-line contention; I would confirm with `perf c2c` on Linux, which I cannot run on
this machine" is a strictly better answer than a confident claim about HITM rates you
have never measured.

### What to graph in production, permanently

| Metric | Why |
|---|---|
| CAS retry count per second (a `LongAdder`) | **The early warning for this entire class of problem.** Almost nobody instruments it. A rising retry rate precedes the throughput collapse. |
| Throughput per core (rps ÷ CPU limit) | If this **falls** when you scale up, you have a shared-line bottleneck somewhere. It is the single cheapest detector. |
| CPU utilisation alongside throughput | 100% CPU with flat throughput is livelock or coherence saturation — not "we need more CPU". |
| p99 before and after any vertical scale change | Make "we added CPUs and it got worse" a thing you find in a dashboard rather than in a postmortem. |

---

## Practice exercises

### 1 — Easy: find your own crossover point

Run the `CounterBench` harness from the Failure drill. Produce two charts from
`results.json`:

- **Chart A:** throughput (ops/sec) against thread count, one line per benchmark.
- **Chart B:** throughput **per thread** against thread count. This one is more
  revealing: a horizontal line means perfect scaling, and a descending line means the
  shared resource is saturated.

Then answer in writing:

1. Your crossover thread count for `atomicIncrement` versus `adderIncrement`.
2. Whether `atomicIncrement`'s **aggregate** throughput at 64 threads is higher or lower
   than at 8, and by how much.
3. Where `racyIncrement` sits relative to both, and what the gap between `racyIncrement`
   and `atomicIncrement` tells you about how much of the cost is the atomicity versus
   the sharing.
4. The read:write ratio at which `AtomicLong` wins the `*Mixed` benchmarks back.

Do not hardcode 64 or your core count in the analysis — express the answers as
"at N threads" and "at N × cores".

### 2 — Medium: the audit (combines Topics 01, 13, 68, 87, 90, 92, 94)

The fragment below contains **eight** distinct defects. Find them all, state the
**exact symptom each produces in production** — what an on-call engineer sees on a
dashboard or in a log, not "it's bad practice" — and rewrite it correctly.

```java
@Component
public class ReservationPool {

    private final AtomicReference<Slot> head = new AtomicReference<>();
    private final AtomicLong poolSize = new AtomicLong();
    private final AtomicLong totalAcquired = new AtomicLong();
    private final Map<Long, AtomicLong> perUserCount = new HashMap<>();
    private static final int MAX_POOL = 1000;

    static class Slot {
        Slot next;
        long quantity;
        String sku;
    }

    public Slot acquire(Long userId) {
        Slot observed;
        do {
            observed = head.get();
            if (observed == null) {
                return new Slot();
            }
        } while (!head.compareAndSet(observed, observed.next));

        totalAcquired.updateAndGet(n -> {
            log.debug("acquired slot, total now {}", n + 1);
            return n + 1;
        });

        AtomicLong userCount = perUserCount.get(userId);
        if (userCount == null) {
            userCount = new AtomicLong();
            perUserCount.put(userId, userCount);
        }
        userCount.incrementAndGet();

        poolSize.decrementAndGet();
        return observed;
    }

    public void release(Slot s) {
        if (poolSize.get() >= MAX_POOL) {
            return;
        }
        s.next = head.get();
        head.set(s);
        poolSize.incrementAndGet();
    }

    public boolean sameUser(Long a, Long b) {
        return a == b;
    }
}
```

Where to look, in no particular order: what `release` does that `acquire` was careful
not to do; what happens when `acquire` and `release` interleave on the same `Slot`
(Trace 1); whether `poolSize` can be trusted given how it is used; what the lambda
inside `updateAndGet` does on a retry; how `perUserCount` behaves at 400 rps; what
`a == b` returns for user 4,829,113 (Topic 01); whether `Slot`'s fields are safely
published to the thread that receives it (Topic 88); and whether `perUserCount` ever
shrinks (Topic 79).

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A.** Confirm the Topic 65 baseline reproduces within ±10%.

**Part B — build the ABA bug on purpose.** Implement `ReservationPool` as a Treiber
stack with reuse. Write a test that reproduces the double-hand-out from Trace 1
deterministically (use latches to force the interleaving — this is a rehearsal for
Topic 99). Prove that stock reconciliation drifts.

**Part C — fix it three ways and compare.** (1) `AtomicStampedReference`.
(2) `ConcurrentLinkedQueue` as the pool. (3) Delete the pool entirely. For each, measure
under the k6 load: p99, allocation rate (Topic 68), and young-GC count (Topic 71).
**State which you would ship and why**, using your own numbers.

**Part D — the counter experiment at two container sizes.** As in Failure drill Part D:
{AtomicLong, LongAdder} × {4 CPU, 16 CPU}, with p99 and flame-graph attribution.

**Part E — instrument the early warning.** Ship the CAS-retry-rate metric. Drive load up
until the retry rate becomes non-trivial. Determine the request rate at which retries
begin to climb, and set an alert threshold below it. **This artefact — an alert with a
number derived from a measurement — is the deliverable.**

**Part F — argue the other side.** Make the strongest case that all of this is premature
optimisation for a service doing 400 rps against a database. Then make the strongest case
against yourself. State the condition under which your answer flips: a request rate, a
core count, or a change in what the counter is used for.

---

## Interview questions

### Q1 — "Atomics are lock-free, so they scale. Right?"

**Mid-level answer:** "Yes — they avoid locks, so there is no blocking and no context
switching, which makes them faster than `synchronized` under contention."

**Senior answer:** "No, and the two halves of that sentence are about different things.
**Lock-free is a progress guarantee, not a performance property.** It means no thread
can be blocked indefinitely by another thread being descheduled — if a thread holding a
lock is preempted, everyone waits; with a CAS, someone always makes progress. That is a
liveness property and it says nothing about throughput.

Under contention, atomics do not scale, and the mechanism is specific. A
`lock`-prefixed instruction requires the executing core to take the cache line into
exclusive ownership, which invalidates every other core's copy. So N cores updating one
counter must pass one 64-byte line between them serially, at tens to hundreds of
nanoseconds per transfer. Coherence traffic is O(N) per successful update while the
useful work is O(1). Throughput per core falls, and aggregate throughput can *decrease*
as you add cores.

There is a second-order point too: a failed CAS is not cheap. To attempt the compare you
must already own the line, so a failure costs a full round trip and produces no work.
That is why retry storms are expensive rather than merely wasteful.

The fix, when the value is written often and read rarely, is `LongAdder` — it gives each
contending thread its own `@Contended` cell so the threads write to different lines and
there is no transfer. It costs you an inexact, non-atomic read and some memory. I would
prove both with a JMH harness at `@Threads({1,2,4,8,16,32,64})` and
`@State(Scope.Benchmark)`, because the crossover point depends on the hardware."

**What separates them:** distinguishing a progress guarantee from a performance
property; naming the cache line as the serial resource; knowing a **failed** CAS costs a
full round trip; and offering the measurement rather than the assertion.

**Follow-up:** "So should every counter be a `LongAdder`?" No — it has no
`compareAndSet`, and `sum()` is not atomic, so anything that reads the value to make a
decision needs `AtomicLong` or a `Semaphore`.

---

### Q2 — "Walk me through what `AtomicInteger.incrementAndGet()` compiles to."

**Mid-level answer:** "It's a compare-and-swap loop — it reads the value, adds one, and
retries the CAS until it succeeds."

**Senior answer:** "That is what the **Java source** looks like — `Unsafe.getAndAddInt`
is written as a CAS loop. But it is annotated `@IntrinsicCandidate`, so C2 replaces the
whole call. On x86-64 it becomes a **single `lock xadd`** — an unconditional atomic add
that cannot fail and never retries. On ARMv8.1, including Apple Silicon, it becomes an
LSE atomic like `ldadd`; on older ARM it is an `ldxr`/`stxr` load-exclusive/store-exclusive
pair, which genuinely is a hardware retry loop.

The retry loop is real for `updateAndGet`, `getAndUpdate` and `accumulateAndGet`,
because an arbitrary function cannot be one instruction — those are a Java loop around
`lock cmpxchg`. Which is a practical difference: `updateAndGet(v -> v + 1)` is
measurably worse than `incrementAndGet()` for the same result, and its lambda must be
side-effect-free because it re-executes on every failed attempt.

None of that changes the contention story, though. All of them require exclusive
ownership of the cache line, so all of them serialise on the line. The intrinsic saves
you the loop, not the coherence traffic.

I'd verify with `-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=print,...`
and `hsdis` — `javap -c` won't show it, because the substitution happens in the JIT, not
in the bytecode."

**What separates them:** knowing it is an **intrinsic** and therefore not a loop;
knowing which methods genuinely *are* loops and why; the ARM difference; the
side-effect-free requirement; and naming the tool that settles it, including that
`javap` is the wrong tool.

**Follow-up:** "Why is a failed CAS expensive?" The RFO — you must own the line before
you can compare, so failure costs the same transfer as success.

---

### Q3 — "What is the ABA problem, and does it matter in Java?"

**Mid-level answer:** "A value changes from A to B and back to A, so a CAS succeeds when
it shouldn't. It's mostly a C++ problem; Java's GC handles it."

**Senior answer:** "The mechanism first: CAS compares a **value**, not a history. It can
answer 'is this word still what I last saw' but not 'has it been unchanged since I last
saw it'. If a reference goes A → B → A, a stale CAS succeeds against a structure whose
shape changed underneath it.

Java is usually protected, and the reason is precise: the GC will not free an object
while you hold a reference to it, so a node you popped cannot be reallocated as a
different node. The classic C++ failure — free the node, `malloc` returns the same
address — cannot happen.

But that protection disappears the moment **you** recycle objects: an object pool, a
ring buffer, a hand-rolled free list, a Netty `ByteBuf`. And the irony is that people
add object pools as an *allocation-rate optimisation*, so the performance fix
reintroduces the correctness bug the GC was preventing.

Concretely, in a pooled lock-free stack: thread A reads `head == X` and intends
`CAS(head, X, X.next)`. Thread B pops X, pops Y, and pushes X back. `head` is X again,
but `X.next` is now something else. A's CAS succeeds and the stack is corrupted — two
callers end up holding the same object.

The fix is `AtomicStampedReference`: CAS the reference and a monotonically increasing
`int` together, so A → B → A has a different stamp and the CAS correctly fails. It costs
an internal `Pair` allocation per update, which partly defeats the pool's purpose — and
that is usually the signal to ask whether the pool earns its keep at all. My first
question would be what the pool saved, measured, versus plain allocation with a
generational collector."

**What separates them:** the value-versus-history framing; explaining *why* the GC
protects you rather than just asserting it; identifying object pooling as the specific
thing that removes the protection; and noticing the irony that the performance
optimisation caused the correctness bug.

**Follow-up:** "How would you test for it?" Deterministically with latches for a demo;
jcstress (Topic 99) for the real structure — and being honest that a passing stress run
means "not observed", not "impossible".

---

### Q4 — "When would you use `LongAdder` over `AtomicLong`, and what does it cost?"

**Mid-level answer:** "`LongAdder` is faster under high contention, so use it for
counters."

**Senior answer:** "The rule is write-heavy and read-rare. Metrics counters are the
canonical case: incremented on every request from every thread, read once every fifteen
seconds by a scrape.

Mechanically, `LongAdder` starts as an `AtomicLong` — it CASes a `base` field. On the
first *failed* base CAS it allocates a `Cell` array and from then on each thread hashes
to its own cell using its `ThreadLocalRandom` probe. `Cell` is annotated
`@jdk.internal.vm.annotation.Contended`, so each one occupies its own cache line, and
that padding is what makes the striping actually work rather than just relocating the
contention. The table grows to at most the next power of two above the core count,
because more cells cannot help than you have cores.

Three costs. First, **`sum()` is O(cells) and not atomic** — it walks the array, so a
concurrent increment to a cell it has already passed is missed. The value is plausible,
not a snapshot. Second, **there is no `compareAndSet`** — the value is not in one place,
so there is nothing to compare. Third, **memory**: a padded cell per contending thread,
so effectively a cache line each.

Those costs decide it. For a monotonic counter scraped on an interval, all three are
fine. For a **rate limiter or a quota check** they are all disqualifying — `if
(adder.sum() < limit)` is a check-then-act race with an inaccurate check on top, and
the right primitive there is a `Semaphore`, whose `tryAcquire` checks and takes in one
atomic operation.

And at low thread counts `AtomicLong` is faster, because `LongAdder` does strictly more
work for the same result. The crossover is usually low — a handful of threads — but it
is hardware-dependent, so I'd measure it rather than quote it. In practice I'd also
check whether the metrics library already solved this: Micrometer's `Counter` is striped
already, so the best version of this change is often deleting the hand-rolled counter."

**What separates them:** the mechanism (base → cells → probe → `@Contended`); naming all
three costs; the specific disqualifying use case with the correct alternative; knowing
`AtomicLong` wins at low thread counts; and reaching for the library rather than writing
concurrency code.

**Follow-up:** "Why is `Cell` `@Contended` rather than just a plain field?" Because
without padding, adjacent cells share a line and you are back to one contended line —
which is Topic 96.

---

### Q5 — "We scaled the service from 4 CPUs to 16 and p99 got worse. Where do you look?"

**Mid-level answer:** "I'd check GC — more heap and more threads usually means more GC —
and look at whether the thread pool is sized wrong for the new CPU count."

**Senior answer:** "Both of those are worth checking, and I'd check them, but the
symptom 'more cores made it worse' has a short list of causes and I'd work through it in
order.

1. **`availableProcessors()`-derived sizing changed under me.** The common
   `ForkJoinPool` is `cores − 1`, GC thread counts scale with cores, and Tomcat and
   Hikari defaults may derive from it. Sixteen cores means a different, possibly worse,
   configuration for the same code. This is Topic 82 and it is the cheapest thing to
   rule out.
2. **A shared cache line.** More cores contending on one atomic counter or one adjacent
   pair of fields means more read-for-ownership transactions for the same amount of
   work. This is the one that surprises people, because the code did not change and the
   load did not change — only the number of contenders did. Metrics counters are the
   usual culprit, because they are on every request path and nobody profiles
   instrumentation.
3. **A lock whose critical section is now the bottleneck.** More threads arriving at the
   same `synchronized` block means more of them park rather than spin.
4. **NUMA**, if the 16-CPU instance spans two sockets and the 4-CPU one did not. A
   cross-socket line transfer is far more expensive than a cross-core one, so the same
   contention gets dramatically worse.

To distinguish them: a flame graph (Topic 78) at both sizes, differenced. If time has
moved into `AtomicLong`/`Unsafe` methods, it is (2). If into `park`/`unpark`, it is (3).
If GC threads, it is (1). On Linux I'd confirm (2) with `perf c2c`, which names the
specific contended cache lines — though on a Mac I cannot run that and would fall back to
a JMH scaling curve reproducing the effect in isolation.

The fix for (2) is `LongAdder` if the counter is read rarely, or padding if it is false
sharing between two independent fields — but I'd measure before padding anything,
because false sharing is over-diagnosed."

**What separates them:** having an **ordered list** rather than one guess; including the
`availableProcessors` cascade, which is the most common real answer; naming the
diagnostic that distinguishes each case; the NUMA consideration; and the honesty about
what they can and cannot measure on their own machine.

**Follow-up:** "How would you have caught it before the change?" Throughput **per core**
as a standing dashboard panel — if it falls when you scale up, you have a shared-resource
bottleneck, and that is one panel and no extra instrumentation.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A failed CAS costs roughly what a successful one costs, because you must own the
   cache line before you can compare. Given that, explain why an *optimistic* algorithm
   is still usually a good idea — and state the contention level at which the argument
   reverses.

2. `LongAdder` stops growing its cell table at the number of CPUs. Why is that the right
   bound rather than, say, the number of threads? What would go wrong if it grew with
   thread count on a service with 200 request threads and 4 cores?

3. Java's GC protects you from most ABA because a referenced object cannot be recycled.
   Name two things you might legitimately do for performance that remove that protection,
   and explain why the protection was invisible to you until you removed it.

4. `AtomicLong.get()` is one load and is exact. `LongAdder.sum()` walks the cells and is
   not atomic. Is there any way to make a striped counter's read exact without
   serialising the writers? Argue for your answer, then make the strongest case against
   yourself.

5. Topic 94's `ReentrantLock` and this topic's `AtomicLong` both use `lock cmpxchg` on
   their fast path. Under heavy contention, which degrades more gracefully, and why? Does
   your answer change if the critical section is long rather than a single increment?

6. `Thread.onSpinWait()` compiles to `PAUSE` on x86 and costs essentially nothing. Why is
   an instruction whose entire purpose is to do nothing more useful than an empty loop
   body? What is the CPU actually being told?

7. You have measured that replacing an `AtomicLong` with a `LongAdder` improves a
   microbenchmark by a large factor at 64 threads and does not move the service's p99 at
   all. Write the two-sentence recommendation you would put in the pull request — and
   state what would have to change for you to revisit it.

---

## Quick reference card

### Instruction mapping (x86-64)

| Java | Instruction | Can fail? |
|---|---|---|
| `getAndAdd`, `incrementAndGet`, `getAndIncrement` | `lock xadd` | no |
| `compareAndSet`, `weakCompareAndSet` | `lock cmpxchg` | **yes** |
| `getAndSet` | `xchg` (implicitly locked) | no |
| `updateAndGet`, `getAndUpdate`, `accumulateAndGet` | Java loop around `lock cmpxchg` | retries |
| `LongAdder.increment` | `lock cmpxchg` on `base`, then on a per-thread `Cell` | retries once, then stripes |

On ARMv8.1 / Apple Silicon: LSE atomics (`ldadd`, `swp`, `casal`) or LL/SC
(`ldxr`/`stxr`). Same semantics, different code, different microbenchmark numbers.

### Choosing a counter

| Need | Use |
|---|---|
| Write-heavy, read-rare (metrics) | **`LongAdder`** — or your metrics library's counter, which already is one |
| Exact instantaneous read | `AtomicLong` |
| Conditional update ("decrement if ≥ n") | `AtomicLong` + a **bounded** CAS loop |
| Concurrency limit / permit count | **`Semaphore`** — never a counter plus a check |
| Two fields that must change together | `AtomicReference` to an immutable record, or pack into one `long` |
| A reference to a **recyclable** object | **`AtomicStampedReference`** — or don't recycle |
| Max / min / custom associative fold | `LongAccumulator` |
| Weaker memory ordering for a hot read | `VarHandle` `getOpaque` / `getAcquire` (Topic 87) |

### The CAS loop idiom

```java
long observed, next;
int attempts = 0;
do {
    if (++attempts > MAX_ATTEMPTS) throw new ContentionTimeoutException();
    observed = cell.get();
    if (!precondition(observed)) return false;   // re-check EVERY iteration
    next = compute(observed);                    // MUST be pure -- it re-runs
    Thread.onSpinWait();                         // PAUSE / YIELD hint
} while (!cell.compareAndSet(observed, next));
```

### Gotchas checklist

- [ ] `volatile x++` is not atomic. It never was. (Topic 87)
- [ ] The lambda in `updateAndGet` / `compute` / `merge` **re-runs**. Keep it pure.
- [ ] Prefer `incrementAndGet()` to `updateAndGet(v -> v + 1)` — one is an intrinsic.
- [ ] `LongAdder` has **no** `compareAndSet` and **no** exact read.
- [ ] `LongAdder.sum()` is not atomic. Never use it in a check-then-act.
- [ ] Bound every CAS retry loop, or you have built a livelock. (Topic 98)
- [ ] Put `Thread.onSpinWait()` in every spin loop. It is free.
- [ ] CAS on a reference to a **reusable** object needs a stamp. (ABA)
- [ ] There is no double-word CAS. Two atomics are not one atomic.
- [ ] `@Contended` needs `-XX:-RestrictContended` **and** `--add-exports`. Use manual
      padding instead. (Topic 96)
- [ ] Adding cores can reduce throughput. Graph throughput **per core**.
- [ ] Never benchmark this single-threaded. `@State(Scope.Benchmark)` + `@Threads`.
- [ ] Check whether Micrometer already solved it before writing a counter.

### Commands

```bash
nproc / sysctl -n hw.ncpu                     # core count -- every curve is relative to it
sysctl -n hw.cachelinesize                    # macOS: 128 on Apple Silicon, 64 on Intel
getconf LEVEL1_DCACHE_LINESIZE                # Linux

java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly \
     -XX:CompileCommand=print,Class::method   # needs hsdis

java -jar target/benchmarks.jar CounterBench -rf json -rff results.json

perf stat -e cache-misses,LLC-load-misses -- java ...   # Linux only
perf c2c record / perf c2c report                       # Linux only; the direct measurement

unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/concurrent/atomic/*' -d ~/jdk-src
grep -n Contended ~/jdk-src/java.base/java/util/concurrent/atomic/Striped64.java
```

---

## When would I use this at work?

**1. The vertical-scaling ticket that does not pay off.**

Someone proposes doubling the pod's CPU limit to fix p99. You ask for one graph first:
**throughput per core at the current size**. If it is already falling as concurrency
rises, more cores will make it worse, not better, and you have saved the team a sprint
and the company a cloud bill. Then you go and find the shared line — usually a metrics
counter — with a flame graph.

**2. Reviewing a pull request that adds an object pool.**

Someone pools `ByteBuffer`s or domain objects to reduce allocation rate after reading
Topic 68. You ask two questions: "what did you measure before and after?" and "can the
same instance be removed from the pool and put back?" The second question is the ABA
check, and asking it takes ten seconds. The bug it prevents takes a quarter to find,
because it manifests as a reconciliation discrepancy that gets blamed on a different
team.

**3. Choosing the primitive for a new rate limiter.**

Product wants a per-tenant concurrency cap. Someone proposes a `LongAdder` and a check.
You explain in two sentences that `sum()` is not atomic and that check-then-act races,
and you propose `Semaphore.tryAcquire` — one atomic operation that checks and takes —
with the release in a `finally`. This is the cheapest possible intervention: it costs a
code-review comment and prevents an outage at exactly the load the limiter existed to
survive.

---

## Connected topics

**Prerequisites:**

- **01 — primitives and boxing.** `AtomicLong` is a mutable box. `Long` is an immutable
  one. Confusing them — `Long counter` incremented in a loop — allocates per increment
  and is not atomic either.
- **69 — object layout and alignment.** Why an `AtomicLong` is ~16 bytes, why two of them
  allocated together land in one cache line, and how to compute where a padding field
  falls. This is the arithmetic Topic 96 uses.
- **75 — escape analysis and lock elision.** The reason a naive benchmark of atomics
  lies: C2 can delete work whose result is never observed.
- **85 — `synchronized` and lock inflation.** The fast path is the same `lock cmpxchg` on
  the mark word. Atomics are that fast path, exposed as an API.
- **87 — `volatile` and memory barriers.** `volatile x++` is the bug this topic fixes,
  and on x86 the `lock` prefix is *also* the full barrier — which is why every atomic
  operation carries volatile semantics whether you wanted them or not.
- **92 — `ConcurrentHashMap`.** CHM's `size()`/`mappingCount()` uses `CounterCell`, which
  is the **same `Striped64` machinery** as `LongAdder`. That is not a coincidence and it
  is not an analogy — it is literally the same base class, which is why CHM's size is
  documented as an estimate.
- **94 — explicit locks.** AQS's `compareAndSetState` is this topic's instruction. Topic
  94 is what happens when the CAS fails and you park; this topic is what happens when you
  refuse to park and retry instead. The `tryLock`-livelock from 94's drill and the
  unbounded-CAS livelock here are the same failure.

**This unlocks:**

- **96 — false sharing.** The direct sequel, and it is already in this document twice:
  `Cell` being `@Contended`, and `totalRequests`/`totalErrors` sharing a line. Topic 96
  gives you the cache-line arithmetic, the padding technique, and the measurement.
- **97 — coordination primitives.** `Semaphore` is the correct answer to Trap 4, and its
  `state` is a permit count manipulated by exactly the CAS in this document.
- **98 — the bug taxonomy.** The unbounded CAS retry loop is the canonical **livelock**:
  100% CPU, all threads `RUNNABLE`, no deadlock reported, no progress. Topic 98's
  decision table tells it apart from the other three failure classes.
- **99 — jcstress.** How you would actually test the ABA fix, rather than failing to
  reproduce the bug and calling it fixed.
- **100 — `ForkJoinPool`.** Work-stealing deques are CAS-based lock-free structures; this
  is the machinery underneath them.
- **101 — virtual threads.** A CAS retry loop does not yield, so a spinning virtual
  thread does **not** unmount and can hold its carrier. Spin loops and virtual threads
  interact badly, and `Thread.onSpinWait()` does not fix it.
- **109 — the HikariCP pool.** Its internal concurrent bag is built from atomics and
  `ThreadLocal`s, and its metrics use the same striped-counter idea.
- **118 — metrics and cardinality.** Micrometer's `Counter` is striped for exactly the
  reason in this document. The other half of that topic — that a high-cardinality tag
  creates a new counter per value — interacts badly with striping, because each counter
  brings its own cells.

---

*Java baseline 21, running on JDK 25. Four things in this document are deliberately
hedged rather than asserted: the exact instruction your JDK emits for `incrementAndGet`
on your architecture (Proof 2 settles it, if you can obtain `hsdis`; the ARM answer
genuinely differs from x86), the crossover thread count between `AtomicLong` and
`LongAdder` on your hardware (the Failure drill measures it, and it moves with core
count, socket count and JDK build), whether cache-miss counters are available to you at
all (they are not on macOS, and pretending otherwise would be the exact fabrication this
curriculum refuses), and whether any of this is measurable end-to-end on `orderflow` at
its current load — which Failure drill Part E is deliberately designed to let you answer
"no" to. Everything else — that CAS compares a value and not a history, that a failed
CAS costs a full cache-line round trip, that `Cell` is `@Contended`, and that adding
cores to a contended counter reduces throughput — follows from cache coherence and will
outlive every version number in this file.*
