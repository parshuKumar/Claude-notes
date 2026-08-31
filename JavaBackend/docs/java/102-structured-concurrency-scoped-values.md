# 102 — Structured Concurrency and Scoped Values

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 (preview) / 25 — **both APIs in this document are PREVIEW features whose shape CHANGED across preview rounds. Every code block here is a shape, not a signature. Compile it and let `javac` correct you.**
## Project spine: rewrite the Topic 91 order-detail fan-out (product + inventory + payment status) as a structured scope, and propagate the `orderflow` tenant context through it with a `ScopedValue` instead of the `ThreadLocal` that Topic 101 just multiplied by fifty thousand.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **A `StructuredTaskScope` binds subtask lifetimes to a lexical block. The scope cannot
> exit until every fork has completed or been cancelled — so there are no orphan tasks,
> ever, by construction rather than by discipline.**
>
> **Errors propagate to the parent and siblings are cancelled. The relationship between
> the forking code and the forked work is a real parent-child relationship the runtime
> knows about, which is why it can appear in a thread dump and in a stack trace.**
>
> **A `ScopedValue` is an immutable, lexically-scoped replacement for `ThreadLocal`. The
> binding exists only for the duration of one call, is popped when that call returns, and
> cannot be mutated by anything inside it. That is what makes it leak-free on a pooled or
> virtual thread — there is nothing to `remove()` and nothing that outlives the block.**

The single word that unifies both halves is **lexical**. Both APIs take a property that
used to be *temporal and thread-scoped* — "this task is running somewhere", "this thread
currently has a tenant" — and make it *syntactic and block-scoped*. The closing brace is
the guarantee.

---

## The bridge from what you know

### Part 1 — structured concurrency: NO ANALOGUE, and the gap is real

Node has nothing that does this. Be clear about what is missing, because the missing
thing is the whole point.

```typescript
// TypeScript. What happens to the other two when one rejects?
const [product, stock, payment] = await Promise.all([
  productClient.get(id),
  inventoryClient.get(id),
  paymentClient.status(id),
]);
```

`Promise.all` rejects as soon as the first promise rejects. **The other two keep
running.** They hold sockets, they consume the downstream's rate-limit budget, and if they
have side effects — a cache write, a metric, a retry — those side effects still happen,
for a request that already returned a 502 to the caller.

You have almost certainly never thought about this, because in Node the leaked work is
cheap: a pending promise and an open socket. Nobody notices. The pattern is so normal that
`AbortController` — the closest thing to a fix — is opt-in, manual, and has to be threaded
through every function by hand.

`CompletableFuture.allOf` in Java has **exactly the same hole**, and this is Topic 91's
fan-out:

```java
CompletableFuture.allOf(product, stock, payment).join();
```

One branch fails, `join()` throws, and the other two branches continue running on a thread
pool you no longer have a reference to. **`cancel(true)` on a `CompletableFuture` does not
interrupt the thread running the supplier** — it completes the future exceptionally and
that is all. There is no parent-child relationship anywhere in the API. Three independent
futures happen to be mentioned in one method.

Structured concurrency closes that hole at the language-shape level:

```java
try (var scope = /* a structured scope */) {
    var product = scope.fork(() -> productClient.get(id));
    var stock   = scope.fork(() -> inventoryClient.get(id));
    var payment = scope.fork(() -> paymentClient.status(id));
    scope.join();              // waits for ALL, or for the policy to short-circuit
    // ...
}                              // <-- close() GUARANTEES nothing survives this brace
```

**The closing brace is the contract.** You cannot leak a subtask past it, in the same way
you cannot leak a local variable past it. That is the analogy the JDK authors intended:
structured concurrency is to threads what structured programming (blocks, no `goto`) was
to control flow.

**Verdict: NO TYPESCRIPT ANALOGUE.** The nearest thing in the JS world is a `TaskGroup`
in Swift or a nursery in Trio (Python) — not anything in Node. If you want an intuition
that transfers, use this one: *`Promise.all` is `goto` for concurrency; a structured scope
is a block.*

### Part 2 — `ScopedValue`: `AsyncLocalStorage` is a GENUINE PARTIAL analogue

This one you do have, and it is a good analogue as far as it goes.

```typescript
import { AsyncLocalStorage } from 'node:async_hooks';

const tenantStore = new AsyncLocalStorage<Tenant>();

// At the request boundary:
tenantStore.run(tenant, async () => {
  await handleRequest(req, res);          // everything in here sees the tenant
});

// Deep inside, with no parameter threading:
const tenant = tenantStore.getStore();
```

```java
// Java 21 preview -- SHAPE, not a guaranteed signature. See the version box below.
private static final ScopedValue<Tenant> TENANT = ScopedValue.newInstance();

// At the request boundary:
ScopedValue.where(TENANT, tenant).run(() -> handleRequest(req, res));

// Deep inside:
Tenant tenant = TENANT.get();
```

**What genuinely transfers:**

- A value bound at a boundary and visible to everything called from inside, with no
  parameter threading.
- The binding is scoped to a *call*, not to a *thread*, so it survives the asynchronous
  hops that a naive thread-local would not.
- It is read-only from the perspective of the code inside. `AsyncLocalStorage.run` gives
  the callback a store; you are not supposed to reassign it either.
- Both exist because passing a request context as a parameter through forty layers is
  unbearable, and both are a deliberate, bounded exception to that discipline.

**Where the analogy runs out — and this is the part to be exact about:**

| | `AsyncLocalStorage` | `ScopedValue` |
|---|---|---|
| Propagation mechanism | The async-hooks machinery tracks async resource creation across the whole event loop | The binding is on the current thread's stack; forks inherit it explicitly through a structured scope |
| Mutability | `enterWith` can rebind mid-flight; the stored object is freely mutable | The **binding** is immutable. `where(...).run(...)` creates a new nested binding; nothing can rebind the outer one. |
| Cost | Non-trivial; async-hooks has a measured runtime cost and has historically been a performance concern | Designed to be a cheap read — see Machine-level reality |
| Cleanup | Handled by the runtime when the async context ends | Automatic when `run()` returns. **There is no `remove()`, because there is nothing to remove.** |
| Failure mode when misused | Context silently missing (a lost async boundary) | `NoSuchElementException` on `get()` — **loud, not silent** |
| Scope shape | Async-tree-scoped | **Lexically scoped**, and the closing brace is enforceable by reading the code |

### Now be exact: why immutability plus lexical scope is what makes it leak-free

This is the sentence that separates a senior answer from a repeated headline.

**`ThreadLocal` is mutable and thread-scoped.** `set()` writes a value into a map hanging
off the `Thread` object. That value stays there until someone calls `remove()` or the
thread dies. Two failure modes follow directly:

1. **On a pooled platform thread** the thread does not die. The request ends, the thread
   returns to the pool, the value stays. Next request on that thread sees the previous
   request's data, or the value is simply retained forever. **This is Topic 79's leak, and
   it is the subtlest leak shape in Java.**
2. **On a virtual thread** the thread does die, so nothing leaks — but there are now
   50,000 of them, so a per-thread cost got multiplied by 50,000. **This is Topic 101's
   Trap 4.**

**`ScopedValue` is immutable and lexically scoped.** There is no `set()`. The only way to
establish a value is `where(KEY, value).run(block)`, and the binding exists exactly for
the duration of `block`. When `run` returns — normally or by throwing — the binding is
popped. Both failure modes are structurally impossible:

1. There is no state left on the thread when the block exits, so a pooled thread carries
   nothing into the next request. **Nothing to `remove()`, so nothing to forget to
   `remove()`.**
2. There is no long-lived per-thread map to allocate, so the cost is not multiplied by
   thread count in the same way.

And a third property you get for free: **because the value is immutable, sharing it with a
forked subtask is safe without any synchronisation.** That is precisely why structured
concurrency and scoped values shipped together — the fork inherits a binding that cannot
change underneath it, so there is no publication problem (Topic 17, Topic 88).

**Verdict: PARTIAL ANALOGUE for `ScopedValue` — `AsyncLocalStorage` gives you the right
intuition for "context without parameter threading" and the wrong intuition for
mutability, cost and cleanup. NO ANALOGUE for structured concurrency.**

---

## The version situation — read this before any code

> ### These are PREVIEW APIs and the shapes CHANGED between rounds.
>
> Structured concurrency and scoped values have both been through **multiple preview
> rounds**, and the API shape changed between them. Specifically, and this is the change
> that will bite you:
>
> - **Java 21's preview** constructs a scope with a **subclass**:
>   `new StructuredTaskScope.ShutdownOnFailure()` or
>   `new StructuredTaskScope.ShutdownOnSuccess<T>()`, with `scope.join()` and
>   `scope.throwIfFailed(...)`.
> - **`[JAVA 25]` Later preview rounds moved to a static factory plus a policy object** —
>   an `open()`-style entry point taking a **`Joiner`** that expresses the completion
>   policy, replacing the two named subclasses. Method names and result-reading
>   accessors moved with it.
>
> **I am deliberately not asserting the exact current signatures.** They have changed
> before and may change again before final. What I show below is the **Java 21 preview
> shape**, because that is this curriculum's baseline, with `[JAVA 25]` notes marking
> where I know a change happened.
>
> **What you must do, every time:**
>
> ```bash
> java --version                    # write it down
> javac --release 21 --enable-preview -d out src/main/java/com/orderflow/**/*.java
> java  --enable-preview -cp out com.orderflow.Application
>
> # Single-file mode, which is what most of this document's labs use:
> java --source 21 --enable-preview OrderDetailScope.java
> ```
>
> `--enable-preview` only works when the `--release`/`--source` you pass **matches the JDK
> you are running**. Compiling with `--release 21 --enable-preview` on a JDK 25 install is
> rejected: preview features are tied to their own release. On JDK 25 you compile with
> `--release 25 --enable-preview` and you get **JDK 25's version of the API**, which is not
> necessarily the one below.
>
> **Then open the javadoc for `java.util.concurrent.StructuredTaskScope` and
> `java.lang.ScopedValue` in YOUR JDK and read the actual method list.** Do not trust a
> method name from this document, from a blog post, or from an LLM. The *concepts* below
> are stable and worth learning. The *spelling* is not.

Preview features also carry a practical consequence worth stating: `--enable-preview`
classfiles are **version-locked**. A class compiled with preview on JDK 21 will not load
on JDK 22. That is intentional, and it is a real argument against putting these APIs on a
production critical path today — a point Topic 107 will make again.

---

## What is this?

### Structured concurrency

A **structured task scope** is an object that owns a set of concurrently-running subtasks
and guarantees that none of them outlive the block that created it.

```java
// Java 21 preview shape.
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

    Subtask<Product> product = scope.fork(() -> productClient.get(productId));
    Subtask<Stock>   stock   = scope.fork(() -> inventoryClient.get(productId));

    scope.join();                      // wait until all done, or policy short-circuits
    scope.throwIfFailed();             // rethrow the first failure, if any

    return new View(product.get(), stock.get());
}
```

Four rules, and they are the whole model:

1. **`fork` creates a new thread** — a virtual thread by default. Not a pool. Not a queue.
2. **`join` waits.** Until then, no subtask result may be read.
3. **The policy decides what happens on the first failure or first success.** Shutdown on
   failure cancels the siblings; shutdown on success cancels the losers.
4. **`close()` — the closing brace of the try-with-resources — cannot complete while any
   subtask is still running.** It waits, and it interrupts.

Rule 4 is the one that makes the other three trustworthy. Everything else is a policy
detail; the lifetime guarantee is structural.

> `[JAVA 25]` In later previews, `fork`'s return type and result accessor changed name and
> the policy moved from a subclass into a `Joiner` argument. The four rules above are
> unchanged; the spelling is not. Read your javadoc.

### `ScopedValue`

```java
// Java 21 preview shape.
public final class TenantContext {
    public static final ScopedValue<Tenant> TENANT = ScopedValue.newInstance();
}

// Bind, at a boundary:
ScopedValue.where(TenantContext.TENANT, tenant)
           .run(() -> chain.doFilter(request, response));

// Read, anywhere inside that call:
Tenant t = TenantContext.TENANT.get();          // throws if not bound
boolean bound = TenantContext.TENANT.isBound(); // check first if it is optional
```

Three rules:

1. **The key is a static final constant.** Like a `ThreadLocal`, it names a slot, not a
   value.
2. **The only way to bind is to wrap a call.** `where(...).run(...)` for `void`,
   `where(...).call(...)` for a value. There is no `set()`. **That absence is the feature.**
3. **`get()` throws if there is no binding.** Missing context is a loud error, not a
   silent `null`. This alone catches a whole class of bug that `ThreadLocal` hides.

Nested bindings shadow outer ones for the duration of the inner call, exactly like a local
variable shadowing a field. When the inner call returns, the outer binding is visible again.

> `[JAVA 25]` The static helpers for binding have moved around across preview rounds —
> chained `where(...).where(...)`, and static `runWhere`/`callWhere` forms have appeared
> and been reshaped. Check your javadoc. The three rules above are the stable part.

### The two together

The reason they shipped as a pair:

```java
ScopedValue.where(TENANT, tenant).run(() -> {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        scope.fork(() -> {
            Tenant t = TENANT.get();       // <-- VISIBLE in the forked thread
            return productClient.get(t, id);
        });
        scope.join();
        scope.throwIfFailed();
    }
});
```

A forked subtask **inherits the scoped-value bindings in effect at the point of the
fork**. This is safe precisely because of the two properties: the value is immutable, so
there is no data race, and the scope guarantees the child ends before the binding is
popped, so the child can never read a dead binding.

`ThreadLocal` has `InheritableThreadLocal` for the same purpose, and it is a much weaker
construct: it copies the whole map at thread creation, has no lifetime guarantee tying the
child to the parent, and is a well-known retention hazard on pooled threads.

---

## Why does it matter?

**1. It fixes a real bug you already have.** The Topic 91 fan-out leaks work on every
failure. You have not noticed because the leak is invisible in normal operation and shows
up only as "downstream call volume that does not match served requests" during an incident.
The Concurrency trace below makes it concrete, with numbers.

**2. It is the direct answer to Topic 101's `ThreadLocal` problem.** Virtual threads made
`ThreadLocal` a memory-multiplier. `ScopedValue` is the JDK's own answer, designed
alongside virtual threads for exactly that reason. The two topics are one story.

**3. It turns cancellation from discipline into structure.** Every codebase has a comment
somewhere saying "remember to cancel the other future if this one fails". That comment is
a bug report. Structured concurrency makes the requirement unstatable-as-a-bug: you
literally cannot write the leaking version.

**4. It makes concurrency visible in a thread dump.** `jcmd Thread.dump_to_file
-format=json` groups virtual threads **by their structured scope**, so the dump shows you
the tree of who forked whom. With `CompletableFuture`, a dump shows you a pool of threads
and no relationships at all. This is a genuine operational improvement and it is
underrated.

**5. It is a strong interview signal — with a caveat you should also give.** Knowing this
API says you read JEPs. Knowing it is *still preview*, that the shape has changed, and
that you would therefore not put it on a critical path yet says you have judgment. **Give
both halves.** The second half is worth more.

---

## Machine-level reality

### How a `ThreadLocal` lookup actually works

```
Thread object
  └── threadLocals : ThreadLocalMap
        └── Entry[]   -- open-addressed hash table
              Entry = WeakReference<ThreadLocal<?>> key + strong Object value
```

- `get()` hashes the `ThreadLocal` instance, probes the table, follows the probe sequence
  on collision, returns the value.
- `set()` **writes into that table**, which lives on the `Thread` and outlives the call.
- The key is a weak reference; the **value is strong**. If the `ThreadLocal` object becomes
  unreachable but the thread lives on, you get a stale entry whose key is null and whose
  value is retained — the classic leak, cleaned only opportunistically on subsequent map
  operations. **This is why Topic 79 calls it the subtlest leak shape in Java.**
- Per thread, you pay: one map object, one `Entry[]`, one `Entry` per distinct
  `ThreadLocal`, plus whatever the values retain. At 50,000 virtual threads, multiply
  everything by 50,000.

### How a `ScopedValue` binding differs

The design is a **stack discipline**, not a map.

```
Thread's current binding chain (conceptually):

    [ TENANT -> t1 ] --> [ CORRELATION_ID -> c1 ] --> null
         ^ innermost                    ^ outer
```

- `where(KEY, value).run(block)` **pushes** a small immutable binding node, invokes
  `block`, and **pops** it in a `finally`. The push and pop are tied to a call frame.
- `get()` walks that chain looking for the key. Because bindings are typically few and
  shallow, the walk is short — and the implementation keeps a small per-thread cache of
  recently-read bindings so the common case is closer to an array probe than a search.
- **There is no long-lived table.** When the block returns there is nothing left. That is
  the leak-freeness, stated mechanically: *cleanup is a `finally`, not a convention.*
- **The value is immutable and there is no mutating operation**, so a binding can be shared
  with a forked thread by reference with no copying, no synchronisation and no safe-
  publication concern (Topics 17, 88).
- **Inheritance is by snapshot, not by copy.** A structured `fork` captures a reference to
  the parent's current binding chain. Contrast `InheritableThreadLocal`, which copies the
  whole map into the child at construction — an O(entries) cost per thread, which at
  virtual-thread counts is exactly the wrong shape.

> **Flagged uncertainty:** the exact caching strategy and node representation are JDK
> implementation details that have changed during preview and are not part of the spec. I
> am describing the design intent and the observable consequences, which are stable. If
> you need the specifics for a performance argument, read the source in your JDK
> (Topic 125) rather than quoting this paragraph.

**The consequence you can reason about without the implementation:** a `ScopedValue` read
is cheap and a binding is cheap, but **a deep chain of many distinct bindings is not
free**. Bind few values, near the boundary. If you find yourself binding eight scoped
values, you have reinvented a parameter object with worse ergonomics — and a plain
parameter would have been better.

### What `fork` actually does

`scope.fork(task)`:

1. Creates a **new virtual thread** (by default) — see Topic 101 for what that costs.
2. Records it in the scope's own set of children, so `close()` knows about it.
3. Captures the current scoped-value bindings for inheritance.
4. Starts it.

There is no pool, no queue, no reuse. **The scope is not an `ExecutorService`, and reading
it as one is the source of half the misunderstandings.** It is a lifetime boundary with a
completion policy.

### What `close()` actually guarantees

The closing brace of the try-with-resources runs `close()`, which:

1. Ensures the scope is shut down (cancelling any still-running subtasks by interrupting
   their threads).
2. **Waits for every child thread to terminate.**
3. Throws if you never called `join()` — which turns "I forgot to join" from a silent race
   into a loud error.

Point 2 is the whole guarantee, and note its cost: **cancellation is cooperative**. A
subtask that ignores interruption — a tight CPU loop with no interruption check, or a
blocking call that does not respond to interrupt — will make `close()` wait for it.
Structured concurrency does not give you preemption; it gives you a place where the wait
is guaranteed to happen and cannot be forgotten. That is a much better position than
`allOf`, and it is not magic.

### Scope confinement

A scope is **confined to the thread that created it**. Only the owner may `fork`, `join`
and `close`. You cannot store a scope in a field and fork into it from elsewhere; the API
rejects it.

This is deliberate and it is what makes the tree of scopes well-formed, which is in turn
what lets `jcmd Thread.dump_to_file -format=json` render the parent-child structure. It is
also why "let me make the scope a Spring bean" is a category error — a scope is a local
variable with a lifetime, like a `try-with-resources` on a file handle.

### Where the threads come from

Forked subtasks run on **virtual threads** by default, which means everything from
Topic 101 applies verbatim: they unmount on blocking calls, they pin on native frames, and
a `synchronized` block around a slow call inside a subtask can starve the carriers for the
whole JVM. **Structured concurrency does not protect you from pinning.** The two topics
compose; they do not substitute.

---

## Concurrency trace

Before any correct code: the failure, step by step. This is the Topic 91 fan-out, and it
is the bug structured concurrency exists to make unwritable.

### The scenario

`orderflow`'s order-detail endpoint, `GET /orders/{id}`, fans out to three downstreams:

```java
// OrderDetailService.java -- the version that ships
public OrderDetail detail(OrderId id) {
    var product = CompletableFuture.supplyAsync(() -> productClient.get(id),   pool);
    var stock   = CompletableFuture.supplyAsync(() -> inventoryClient.get(id), pool);
    var payment = CompletableFuture.supplyAsync(() -> paymentClient.status(id), pool);

    CompletableFuture.allOf(product, stock, payment).join();   // <-- the hole

    return new OrderDetail(product.join(), stock.join(), payment.join());
}
```

Right now:

- The **payment gateway** is returning HTTP 500 in about 20 ms — it is failing fast.
- The **product service** is degraded and taking 30 seconds per call before timing out.
- The **inventory service** is healthy at about 15 ms.
- `GET /orders/{id}` is 20% of the Topic 65 traffic mix.

Two columns: the request thread that called `detail()`, and the product-branch thread.
The payment branch's single event appears inline because it lasts 20 ms and then is gone.

### The interleaving

| Step | Request thread (serving `GET /orders/9001`) | Product-branch thread (`pool-1-thread-7`) |
|---|---|---|
| 1 | Calls `supplyAsync` three times. Three tasks are queued on `pool`. Returns immediately with three `CompletableFuture` handles. | Picks up the product task. Opens a connection to the product service. |
| 2 | Calls `allOf(product, stock, payment).join()`. Blocks. | Sent the request. Waiting on the socket. |
| 3 | Blocked. | Waiting. |
| 4 | *(Inventory branch completes at ~15 ms and its thread returns to the pool.)* | Waiting. |
| 5 | *(Payment branch fails at ~20 ms with a 500. `payment` completes exceptionally.)* | Waiting. |
| 6 | `allOf` completes exceptionally. **`join()` throws `CompletionException` at ~20 ms.** The request returns HTTP 502 to the caller. | **Still waiting.** Nothing told it anything. |
| 7 | The request thread is released. The servlet container logs a 502. The `OrderDetailService` stack frame is gone; **the local variables `product`, `stock` and `payment` are now unreachable.** | Still waiting. Still holding a connection to the product service. Still occupying a pool thread. |
| 8 | The next request arrives and does the same thing. It forks three more tasks. | Still waiting. |
| 9 | ... 200 requests later, at the Topic 65 arrival rate for this endpoint, roughly 200 product-branch tasks have been forked. | 200 threads (or 200 queued tasks) waiting on a service that answers in 30 seconds. |
| 10 | Requests now start to slow down for a **new** reason: `pool` has a fixed size and every thread is occupied by an orphaned product call. `supplyAsync` starts queueing. | Still waiting. |
| 11 | The inventory and payment forks now sit in the pool's queue behind the orphans, so even the fast branches get slow. | At t=30 s, the first product call finally times out. The branch completes exceptionally. **Nobody is listening.** `CompletableFuture` swallows an exceptional completion that nobody joins — no log, no metric, no trace. |
| 12 | Endpoint p99 has gone from its Topic 65 baseline to the pool-queue delay. **The catalogue endpoint, which shares the same `pool`, degrades too.** | Thread returns to the pool. Another orphan takes its place. |

### Outcome, in business terms

The endpoint failed at step 6, correctly and quickly. **Everything after step 6 is work
done for a request that no longer exists.**

- **The product service receives roughly 3x its expected call volume from `orderflow`** —
  one call per request served, plus one per request that failed at the payment branch,
  plus retries. Its own load shedding will eventually fire, which spreads the incident to
  every other consumer of the product service. **You have amplified someone else's outage
  into your own, and then exported it.**
- **The shared pool fills with orphans.** The catalogue endpoint, which has nothing to do
  with order detail and calls no failing service, degrades. Same class of blast-radius
  surprise as Topic 100's common-pool blocking and Topic 101's carrier starvation, arriving
  by a third route.
- **The failure is silent.** An exceptional completion that nobody joins is discarded.
  There is no log line and no metric for the orphaned failures, so your error rate
  *understates* the number of downstream calls that failed.
- **The metric that reveals it** is the ratio of downstream calls issued to requests
  served. Above 1.0 during an error burst is orphaned work, and almost nobody graphs it.
- **On the Topic 65 dashboard** you see: order-detail p99 spiking (explained by the
  product outage), catalogue p99 spiking (**not** explained by anything), and a product-
  service call rate that does not match any endpoint's throughput.

**The one-line diagnosis:** `allOf` is a *completion* combinator, not a *lifetime* scope.
It tells you when things finish. It has no opinion about what should stop.

### The same trace under a structured scope

For contrast, the identical scenario with `ShutdownOnFailure`:

| Step | Request thread | Product-branch virtual thread |
|---|---|---|
| 1 | Enters the `try (var scope = ...)`. Forks three subtasks; each gets a new virtual thread. | Starts. Sends the request. Blocks on the socket — **and unmounts**, so it holds no carrier (Topic 101). |
| 2 | Calls `scope.join()`. | Waiting. |
| 3 | *(Payment subtask fails at ~20 ms.)* The `ShutdownOnFailure` policy fires: the scope shuts down. | **Interrupted.** The blocking socket read throws `InterruptedException`. |
| 4 | `join()` returns. `throwIfFailed()` rethrows the payment failure. | Unwinds, closes its connection, terminates. |
| 5 | The exception propagates out of the try-with-resources. **`close()` runs and waits for all three threads to have terminated.** They have. | Gone. |
| 6 | Returns 502. **Zero orphans. Zero extra downstream load. The product service sees one fewer call, not one more.** | — |

Step 3 is the entire difference, and it is not something you remembered to write. It is
what the policy does.

---

## Example 1 — minimal

The smallest scope that shows the three properties: parallel execution, failure
propagation, and sibling cancellation.

```java
// ScopeBasics.java
// Java 21 preview SHAPE. Run with:  java --source 21 --enable-preview ScopeBasics.java
package com.orderflow.lab;

import java.util.concurrent.StructuredTaskScope;
import java.util.concurrent.StructuredTaskScope.Subtask;

public class ScopeBasics {

    static String slowOk(String name, long ms) throws InterruptedException {
        System.out.printf("%-10s START  on %s%n", name, Thread.currentThread());
        try {
            Thread.sleep(ms);
            System.out.printf("%-10s DONE%n", name);
            return name + "-ok";
        } catch (InterruptedException e) {
            System.out.printf("%-10s INTERRUPTED after being cancelled%n", name);
            throw e;                       // propagate; do NOT swallow
        }
    }

    static String failFast(String name) {
        System.out.printf("%-10s START  on %s%n", name, Thread.currentThread());
        throw new IllegalStateException(name + " failed");
    }

    public static void main(String[] args) throws Exception {
        long t0 = System.currentTimeMillis();

        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            Subtask<String> slow    = scope.fork(() -> slowOk("slow", 3_000));
            Subtask<String> quick   = scope.fork(() -> slowOk("quick", 100));
            Subtask<String> broken  = scope.fork(() -> failFast("broken"));

            scope.join();                          // wait, or short-circuit on failure
            scope.throwIfFailed();                 // rethrow the first failure

            System.out.println(slow.get() + " " + quick.get() + " " + broken.get());

        } catch (Exception e) {
            System.out.printf("CAUGHT %s: %s%n",
                    e.getClass().getSimpleName(), e.getMessage());
        }

        System.out.printf("TOTAL_ELAPSED_MS=%d%n", System.currentTimeMillis() - t0);
    }
}
```

```bash
java --version                                        # record it
java --source 21 --enable-preview ScopeBasics.java
```

**What to look for:** the total elapsed time, and whether `slow` prints `INTERRUPTED`.

| What you see | What it means |
|---|---|
| `TOTAL_ELAPSED_MS` far below 3000, and `slow` prints `INTERRUPTED` | **Structured concurrency, working.** The failure of `broken` cancelled `slow` before it finished. On `allOf` the same program would have taken the full 3 seconds *and* left `slow` running afterwards. |
| Each subtask prints a different `VirtualThread[...]` | Forks are new virtual threads, one per fork, not pool threads. |
| `TOTAL_ELAPSED_MS` close to 3000 | Your `slow` task swallowed the interrupt, or the sleep is not responding to it. **Cancellation is cooperative** — a task that ignores interruption delays `close()`. |
| Compile error naming `Subtask`, `fork`, `join` or the scope constructor | **Expected on a different JDK.** The preview API changed shape. Open your javadoc, find the current equivalents, and rewrite. That rewrite is part of the exercise. |
| `preview features are disabled` | You forgot `--enable-preview`, or your `--source`/`--release` does not match your JDK. |
| It runs but prints a preview warning on stderr | Normal and expected. Preview features always warn. |

**Now do the contrast, because the contrast is the lesson.** Rewrite the same program with
`CompletableFuture.allOf` and three `supplyAsync` calls on a fixed pool.

**What to look for:** total elapsed time, and whether `slow` prints `DONE` **after** the
exception was caught and printed.

| What you see | What it means |
|---|---|
| `CAUGHT ...` prints early, then `slow DONE` prints afterwards, and total elapsed is ~3000 ms | **You have reproduced the orphan.** The program had already handled the failure, and the sibling kept running. In production that sibling is a downstream call and a connection. |
| The JVM does not exit promptly | Non-daemon pool threads. Another symptom of the same missing lifetime relationship. |

### The minimal `ScopedValue`

```java
// ScopedValueBasics.java
// Java 21 preview SHAPE. java --source 21 --enable-preview ScopedValueBasics.java
package com.orderflow.lab;

public class ScopedValueBasics {

    record Tenant(String id, String region) { }

    static final ScopedValue<Tenant> TENANT = ScopedValue.newInstance();

    static void deepInsideTheCallStack() {
        System.out.println("  bound? " + TENANT.isBound());
        System.out.println("  tenant = " + TENANT.get());
    }

    public static void main(String[] args) {
        System.out.println("outside any binding:");
        System.out.println("  bound? " + TENANT.isBound());
        try {
            TENANT.get();
        } catch (Exception e) {
            System.out.println("  get() threw " + e.getClass().getSimpleName());
        }

        System.out.println("inside where(...).run(...):");
        ScopedValue.where(TENANT, new Tenant("acme", "eu-west-1"))
                   .run(ScopedValueBasics::deepInsideTheCallStack);

        System.out.println("nested rebinding:");
        ScopedValue.where(TENANT, new Tenant("acme", "eu-west-1")).run(() -> {
            deepInsideTheCallStack();
            ScopedValue.where(TENANT, new Tenant("globex", "us-east-1")).run(() -> {
                System.out.println("  inner:");
                deepInsideTheCallStack();
            });
            System.out.println("  back outside the inner binding:");
            deepInsideTheCallStack();
        });

        System.out.println("after the block:");
        System.out.println("  bound? " + TENANT.isBound());
    }
}
```

**What to look for:** four things, in this order.

| What you see | What it means |
|---|---|
| `get()` throws outside a binding | **Missing context is loud.** A `ThreadLocal` would have returned `null` and the NPE would have surfaced three layers away. This is a real ergonomic win. |
| The inner binding shows `globex`, and after it the outer shows `acme` again | Lexical shadowing, exactly like a local variable. The outer binding was never mutated — it was shadowed. |
| `bound? false` after the block | **The cleanup you did not write.** Compare with `ThreadLocal`, where forgetting `remove()` in a `finally` is Topic 79's leak. |
| A compile error on `where` / `run` / `newInstance` | Expected on a different JDK. The binding helpers moved between preview rounds. Read your javadoc. |

---

## Example 2 — production scenario (on the project spine)

### The constraints

Rewrite `orderflow`'s order-detail fan-out (Topic 91) as a structured scope, and carry the
tenant context through it with a `ScopedValue` instead of the `ThreadLocal` that
Topic 101's audit flagged.

Real requirements, not simplified ones:

1. `GET /orders/{id}` needs product data, inventory, and payment status.
2. If **payment status** fails, the response is still useful — show the order with the
   payment status as "unknown". **A failure of this branch must not fail the request.**
3. If **product** or **inventory** fails, the response is not useful. Fail the request.
4. The whole fan-out has a **deadline**: 400 ms. Past that, fail rather than hold the
   caller.
5. Every downstream call must carry the tenant id, and it must be impossible to forget.
6. It must work under `spring.threads.virtual.enabled=true` (Topic 101).
7. The endpoint is 20% of the Topic 65 traffic mix and has a recorded baseline p99.

Requirement 2 is the interesting one, because it does not match `ShutdownOnFailure`. This
is where people reach for the wrong policy and end up back at `allOf`.

### Step 1 — bind the tenant at the boundary

```java
package com.orderflow.tenancy;

/**
 * The tenant context for the whole request. Replaces the ThreadLocal version that
 * Topic 101's audit flagged as multiplied-by-thread-count.
 *
 * PREVIEW API -- shape shown for Java 21. Verify against your JDK's javadoc.
 */
public final class TenantContext {

    public static final ScopedValue<TenantId> TENANT = ScopedValue.newInstance();

    private TenantContext() { }

    /** The single read point. Loud failure if unbound, with a message that helps. */
    public static TenantId require() {
        if (!TENANT.isBound()) {
            throw new IllegalStateException(
                "No tenant bound. This code path did not go through TenantFilter, "
              + "or it forked a thread outside a structured scope.");
        }
        return TENANT.get();
    }
}
```

```java
package com.orderflow.tenancy;

/**
 * Binds the tenant for the duration of the filter chain. Everything downstream --
 * controller, service, repository, and every structured fork -- sees it.
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 20)     // after auth (Topic 56), before everything else
public class TenantFilter extends OncePerRequestFilter {

    private final TenantResolver resolver;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        TenantId tenant = resolver.resolve(request);      // from JWT claim or header

        try {
            ScopedValue.where(TenantContext.TENANT, tenant)
                       .run(() -> {
                           try {
                               chain.doFilter(request, response);
                           } catch (IOException | ServletException e) {
                               throw new FilterChainException(e);   // ScopedValue.run
                           }                                        // takes a Runnable
                       });
        } catch (FilterChainException e) {
            throw (Exception) e.getCause();
        }
    }
}
```

**Read the ugly part, because it is real.** `ScopedValue.where(...).run(...)` takes a
`Runnable`, which cannot throw checked exceptions, and the servlet `FilterChain` throws
two of them. So you wrap, rethrow, and unwrap. That is genuine friction, it exists in the
Java 21 preview shape, and pretending it does not is how documents lose your trust.

Two honest notes on it:

- `where(...).call(...)` takes a `Callable` and **can** throw checked exceptions — use it
  where you have a return value, and the wrapping disappears. The `Runnable` case is the
  awkward one.
- `[JAVA 25]` The exact set of binding helpers and their functional interfaces changed
  between preview rounds. If your JDK offers a form that throws, use it and delete the
  wrapper. **Check the javadoc before copying this.**

The `TenantId` must be **immutable** — a record wrapping a `String`. Not a `Tenant` object
with a mutable settings map. See Trap 3 for why this is not a style preference.

### Step 2 — the fan-out, with a policy that matches the requirement

Requirement 2 says a payment failure must not fail the request. `ShutdownOnFailure` would
cancel everything when payment fails. `ShutdownOnSuccess` is for racing alternatives. So:
**two scopes, nested, each with the right policy.**

```java
package com.orderflow.orders;

@Service
public class OrderDetailService {

    private static final Duration FANOUT_DEADLINE = Duration.ofMillis(400);

    private final ProductClient   productClient;
    private final InventoryClient inventoryClient;
    private final PaymentClient   paymentClient;
    private final OrderRepository orders;

    /**
     * PREVIEW API SHAPE (Java 21). The policy classes and the deadline method changed
     * name in later previews -- see the version box at the top of this document.
     */
    public OrderDetail detail(OrderId id) throws InterruptedException {

        Order order = orders.findRequired(id);           // cheap, local, before the fan-out
        TenantId tenant = TenantContext.require();       // fail fast if unbound
        Instant deadline = Instant.now().plus(FANOUT_DEADLINE);

        // --- Scope 1: the branches that MUST succeed -----------------------------
        try (var required = new StructuredTaskScope.ShutdownOnFailure()) {

            Subtask<Product> product =
                required.fork(() -> productClient.get(order.productId()));   // inherits TENANT
            Subtask<Stock> stock =
                required.fork(() -> inventoryClient.get(order.productId()));

            // --- Scope 2: the branch that MAY fail -------------------------------
            // A separate scope, because its failure policy is different. Nesting is
            // the idiom: scopes compose, policies do not.
            PaymentStatus payment = paymentStatusOrUnknown(id, deadline);

            required.joinUntil(deadline);      // deadline, not an unbounded wait
            required.throwIfFailed(OrderDetailUnavailable::new);

            return new OrderDetail(order, product.get(), stock.get(), payment);

        } catch (TimeoutException e) {
            throw new OrderDetailUnavailable("fan-out exceeded " + FANOUT_DEADLINE, e);
        }
    }

    /**
     * The optional branch. Its own scope, so its failure is contained here and
     * cannot cancel the required branches.
     */
    private PaymentStatus paymentStatusOrUnknown(OrderId id, Instant deadline) {
        try (var optional = new StructuredTaskScope.ShutdownOnFailure()) {
            Subtask<PaymentStatus> status = optional.fork(() -> paymentClient.status(id));
            optional.joinUntil(deadline);
            optional.throwIfFailed();
            return status.get();
        } catch (Exception e) {
            // Degraded, not failed. Record it so the dashboard can see the degradation.
            paymentStatusFallbacks.increment();
            return PaymentStatus.UNKNOWN;
        }
    }
}
```

**What each line buys, in order of importance:**

1. **`joinUntil(deadline)` rather than `join()`.** An unbounded `join()` means the request
   is held for as long as the slowest downstream chooses. The deadline is a product
   decision — 400 ms — expressed once, in code.
2. **Two scopes with two policies.** The requirement "payment may fail, product may not"
   is a *policy* difference, and the API's answer is to nest scopes. This is more
   verbose than `allOf` and it is more verbose *because it forces you to state something
   `allOf` let you leave unstated*. That is the trade, and it is worth it.
3. **`throwIfFailed(OrderDetailUnavailable::new)`** maps the downstream failure onto your
   own exception hierarchy (Topic 09) at the boundary, so the controller advice
   (Topic 46) produces a `ProblemDetail` and not a leaked `CompletionException`.
4. **The tenant is inherited by every fork.** `productClient.get` calls
   `TenantContext.require()` internally and it works, because the fork captured the
   binding. Nobody had to pass it, and nobody can forget it.
5. **The `orders.findRequired(id)` call is outside the fan-out** because it is a
   prerequisite, not a parallel branch. Forking work that everything else depends on is a
   common and pointless complication.
6. **When any of this throws, `close()` has already waited for every subtask to
   terminate.** There is no path out of this method that leaves a thread running.

> **`[JAVA 25]` Version caveat, repeated because it matters here specifically.**
> `joinUntil(Instant)`, `throwIfFailed(Function)` and the `ShutdownOnFailure` /
> `ShutdownOnSuccess` classes are the **Java 21 preview** shape. Later preview rounds
> replaced the policy subclasses with a `Joiner` passed to a static factory, and the
> deadline moved into scope configuration. **I am not going to guess the current
> spelling.** The *structure* — two scopes, two policies, a deadline, mapped exceptions —
> is what you are learning, and it survives the rename. Open your javadoc and translate.

### Step 3 — what to delete

The migration is not complete until these are gone:

| Delete | Why | Topic |
|---|---|---|
| The shared `ExecutorService` the fan-out used | The scope creates its own virtual threads. A leftover pool is a leftover concurrency cap. | 90, 101 |
| The `ThreadLocal<TenantId>` and its `remove()` in a `finally` | Replaced by the `ScopedValue`. **The `finally` block is the thing you are deleting** — that is the leak-freeness, made concrete. | 79, 101 |
| `InheritableThreadLocal` anywhere near the fan-out | Copies the map per thread; retention hazard; no lifetime relationship. Scoped-value inheritance replaces it. | 79 |
| Any `.orTimeout(...)` / cancellation-token scaffolding | The scope's deadline replaces it. | 91 |
| The comment saying "remember to cancel the others if one fails" | It was a bug report. It is now unwritable. | — |

### Step 4 — measure it against the Topic 65 baseline

The claim to test is **not** "structured concurrency is faster". At matched arrival rates
with all downstreams healthy, it will be roughly the same — same number of concurrent
calls, same latency, plus or minus virtual-thread overhead versus pool overhead.

**The claim to test is what happens during a failure.** Method:

1. Run the Topic 65 mix at the recorded baseline arrival rate, all downstreams healthy.
   Record p50/p95/p99 for `GET /orders/{id}` and for `GET /products`. Both versions.
2. Now inject a fault: make the **product** service take 30 seconds, and make the
   **payment** service return 500 in 20 ms. Hold that for five minutes.
3. Record, for both versions: order-detail p99, **catalogue p99** (the endpoint that should
   be unaffected), error rate, and **the number of calls the product service received**.

**What to look for:** the ratio of product-service calls to order-detail requests served,
and the catalogue endpoint's p99.

**How to read it:**

| Measurement | `allOf` version | Structured version | What it means |
|---|---|---|---|
| Product calls / requests served | Expect **above 1.0** | Expect **at or near 1.0** | The gap is the orphaned work. This is the number that proves the whole topic. |
| Catalogue p99 during the fault | Expect degradation | Expect no change | The `allOf` version's orphans filled a shared pool. The structured version has no shared pool to fill. |
| Order-detail p99 during the fault | Bounded only by the product timeout (~30 s) | Bounded by your deadline (400 ms) | The deadline is the difference, and it is a product decision you can now state. |
| Order-detail p99 when healthy | Baseline | Baseline, roughly | **Expect no improvement here, and say so.** Structured concurrency is a correctness and blast-radius feature, not a throughput feature. |

That last row is the honest one, and it is the row people leave out.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `CompletableFuture.allOf` leaking a still-running sibling

**Wrong approach.** The Topic 91 fan-out, verbatim:

```java
CompletableFuture.allOf(product, stock, payment).join();
return new OrderDetail(product.join(), stock.join(), payment.join());
```

**Exact symptom.** Four observable things, and they look like four unrelated problems:

1. **Downstream call volume exceeds served-request volume** during any error burst. Graph
   `downstream.calls / http.server.requests` per endpoint and it goes above 1.0.
2. **A shared pool fills with orphans**, degrading unrelated endpoints that use the same
   pool — the same blast-radius surprise as Topic 100 and Topic 101, by a third route.
3. **The request's latency is the maximum of all branches, not the minimum failure time**,
   whenever the failing branch is the fast one. `allOf` waits for everything.
4. **The orphan's eventual failure is silent.** An exceptional completion nobody joins is
   discarded — no log, no metric, no trace span closed. Your error rate under-reports.

**Root cause.** `CompletableFuture` has no lifetime relationships. `allOf` is a completion
combinator; it observes, it does not own. `cancel(true)` does not interrupt the thread
executing the supplier — read the javadoc, it says so — it only completes the future
exceptionally. There is no parent, no child, and no scope. Three independent objects
happen to be referenced from one stack frame, and when that frame goes away, the objects
do not.

**Fix.**

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var product = scope.fork(() -> productClient.get(id));
    var stock   = scope.fork(() -> inventoryClient.get(id));
    var payment = scope.fork(() -> paymentClient.status(id));
    scope.joinUntil(deadline);
    scope.throwIfFailed(OrderDetailUnavailable::new);
    return new OrderDetail(product.get(), stock.get(), payment.get());
}   // <-- nothing survives this brace
```

**The interim fix if you cannot use a preview API in production** — and this is a real
constraint, so have the answer ready:

```java
// Not as good, but it closes most of the hole with stable APIs.
var product = CompletableFuture.supplyAsync(...).orTimeout(400, MILLISECONDS);
var stock   = CompletableFuture.supplyAsync(...).orTimeout(400, MILLISECONDS);
var payment = CompletableFuture.supplyAsync(...).orTimeout(400, MILLISECONDS);
// plus a shared cancellation flag the suppliers CHECK, because orTimeout does not
// stop the supplier -- it only completes the future.
```

Say the residual out loud: `orTimeout` bounds the *waiting*, not the *work*. The supplier
keeps going. To actually stop it you need a cooperative cancellation token checked inside
each supplier, threaded through by hand. **That boilerplate is exactly what structured
concurrency removes, and describing it accurately is a better interview answer than
pretending `orTimeout` solves it.**

**Observable:** the downstream-calls-to-requests-served ratio during an injected fault.
Graph it (Topic 118); it is the single most useful metric in this document.

### Trap 2 — `ThreadLocal` request context on virtual threads

**Wrong approach.**

```java
public class TenantContext {
    private static final ThreadLocal<Tenant> CURRENT = new ThreadLocal<>();
    public static void set(Tenant t) { CURRENT.set(t); }
    public static Tenant get() { return CURRENT.get(); }
    public static void clear() { CURRENT.remove(); }
}
```

Plus a filter that calls `set` and `clear` in a `finally`.

**Exact symptom.** Two different symptoms depending on which thread model you are on, and
you should be able to distinguish them:

- **On a pooled platform thread with a missing or skipped `clear()`:** retention grows to
  pool-size × context-size and then plateaus. People look at the plateau and say "not a
  leak". It is a leak — it is just a bounded one. **Worse: the next request on that thread
  may read the previous request's tenant.** Cross-tenant data exposure is a security
  incident, not a memory issue.
- **On virtual threads with `clear()` working perfectly:** nothing leaks, and heap use
  still climbs linearly with concurrent request count, because you multiplied a per-thread
  cost by 50,000. A class histogram shows tens of thousands of context objects.

**Root cause.** `ThreadLocal` is mutable, thread-scoped state whose lifetime is the
thread's, not the request's. Both symptoms follow from that one sentence.

**Fix.** `ScopedValue`, bound once at the filter boundary. Concretely, what changes:

```java
// BEFORE -- the finally block is the whole problem
try {
    TenantContext.set(tenant);
    chain.doFilter(req, res);
} finally {
    TenantContext.clear();          // forget this, and you have both bugs
}

// AFTER -- there is no finally block, because there is nothing to clean up
ScopedValue.where(TenantContext.TENANT, tenant).run(() -> chain.doFilter(req, res));
```

**Do not skip the interaction with Spring Security.** `SecurityContextHolder` is
`ThreadLocal`-backed (Topic 56) and you do not control its implementation. Your own
contexts can move to `ScopedValue`; the framework's cannot, until the framework moves it.
Measure what remains rather than assuming you fixed everything:

```bash
jcmd <pid> GC.class_histogram | grep -iE 'SecurityContext|Authentication|TenantId'
```

**Observable:** class-histogram counts at matched concurrency, before and after; plus the
absence of a `finally` block in the diff.

### Trap 3 — a mutable object inside a `ScopedValue`

**Wrong approach.**

```java
static final ScopedValue<RequestContext> CTX = ScopedValue.newInstance();

class RequestContext {
    private final String tenantId;
    private final Map<String, String> attributes = new HashMap<>();   // <-- mutable
    void put(String k, String v) { attributes.put(k, v); }            // <-- and mutated
}
```

Then, deep in a service: `CTX.get().put("enrichmentStage", "products-fetched");`

**Exact symptom.** Intermittent, load-dependent, unreproducible-in-tests. Missing map
entries. `ConcurrentModificationException` from a `HashMap` in a stack trace that contains
no obvious concurrency. Occasionally a corrupted `HashMap` that loops forever on `get` —
the classic unsynchronised-resize failure. All of it appears only when the fan-out has
more than one branch writing.

**Root cause.** `ScopedValue`'s immutability guarantee applies to the **binding**, not to
the object bound. The binding cannot be changed; the object can. When a structured scope
forks three subtasks, all three inherit a reference to **the same** `RequestContext`
instance, and all three write into the same `HashMap` from three different threads.

**You have accidentally shared mutable state between threads with no synchronisation** —
Topics 86, 87 and 88 in one line — and you did it while using the API specifically designed
to make context propagation safe. That is the trap: the API's name suggests a safety it
does not provide for your payload.

**Fix.** Bind only immutable values.

```java
record TenantId(String value) { }                       // immutable
record RequestContext(TenantId tenant, String correlationId, Instant startedAt) { }
```

If you genuinely need to accumulate data across branches, **return it from the subtasks**
and combine it in the parent after `join()`. That is what the scope's result accessors are
for, and it is the whole point of having a join point. If you need shared mutable state
across forks, use a concurrent structure deliberately and document why — but ask first
whether you actually need it, because in a fan-out you almost never do.

**Careful with records:** `record Ctx(List<String> tags)` is only *shallowly* immutable
(Topic 27). Copy defensively, or use `List.copyOf`.

**Observable:** a code review rule you can automate — the type parameter of every
`ScopedValue` must be a record, an enum, a `String`, or a boxed primitive. Anything else
needs a written justification.

### Trap 4 — reading a subtask result before `join()`, or letting the scope escape

**Wrong approach.**

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var product = scope.fork(() -> productClient.get(id));
    return new OrderDetail(product.get());        // <-- no join()
}
```

Or the subtler version:

```java
Subtask<Product> escaped;
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    escaped = scope.fork(() -> productClient.get(id));
    scope.join();
}
return escaped.get();                             // <-- read AFTER close()
```

Or the architectural version:

```java
@Bean
StructuredTaskScope orderScope() { return new StructuredTaskScope.ShutdownOnFailure(); }
```

**Exact symptom.** `IllegalStateException` — loudly, immediately, deterministically, at
the first call in a unit test. Not a race, not a heisenbug. The message names the problem.

**Root cause.** The API enforces its own protocol. A subtask result is readable only
between a successful `join()` and `close()`. A scope is confined to its creating thread and
cannot be shared, stored in a field, or made a bean.

**Fix.** Treat the scope exactly like a `try-with-resources` on a file handle: create it,
use it, extract what you need **inside** the block, and let it close. Return a value, never
a `Subtask`.

**Say why this trap is good news.** Every one of these mistakes is a compile-time or
immediate-runtime error. The equivalent `CompletableFuture` mistakes — reading a future
that has not completed, storing one past its useful life, sharing a pool across unrelated
concerns — are all silent, and they fail under load in production. **A restrictive API that
fails loudly on the first test run is a feature.** That argument is worth having ready,
because "structured concurrency is too rigid" is the standard objection.

**Observable:** the exception, in a unit test, on the first run.

### Trap 5 — treating the scope as a thread pool

**Wrong approach.**

```java
// "It creates threads, so it's a pool, so I should reuse it and size it."
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    for (OrderLine line : order.lines()) {       // 5,000 lines on a large order
        scope.fork(() -> priceLine(line));       // 5,000 forks, 5,000 virtual threads
    }
    scope.join();
}
```

**Exact symptom.** For an `orderflow` order with a normal number of lines: fine. For the
bulk-repricing job over 5M order lines: the downstream pricing service receives 5,000
simultaneous requests from one order, its own rate limiter fires, everything gets a 429,
and the retry storm (Topic 111) takes it down. Heap climbs from thousands of live
continuations. If `priceLine` is CPU-bound rather than I/O-bound, you also get Topic 101's
Trap 3 — no gain, extra overhead.

**Root cause.** A scope is a **lifetime boundary**, not a **concurrency limiter**. It has
no queue, no size, and no admission control. `fork` starts a thread immediately, every
time. And — this is the through-line of the whole phase — **neither virtual threads nor
structured concurrency give you backpressure. Topic 105.**

**Fix.** Bound concurrency explicitly, as a separate concern from lifetime:

```java
private final Semaphore pricingPermits = new Semaphore(32);   // Topic 97 bulkhead

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    for (OrderLine line : order.lines()) {
        scope.fork(() -> {
            pricingPermits.acquire();        // AQS -> unmounts cleanly (Topic 101)
            try { return priceLine(line); }
            finally { pricingPermits.release(); }
        });
    }
    scope.joinUntil(deadline);
    scope.throwIfFailed();
}
```

Better still for a large batch: chunk it, so you are not holding 5,000 live continuations
just to run 32 at a time.

**Note what the semaphore does and does not do.** It limits *in-flight downstream calls*.
It does not limit *live virtual threads* — all 5,000 still exist, just mostly parked. If
the thread count itself is the problem, chunking is the fix, not the semaphore. Being able
to separate those two is the point.

**Observable:** concurrent downstream requests per order (Topic 118), and live virtual
thread count from `jcmd Thread.dump_to_file -format=json`.

### Trap 6 — assuming a preview API is safe to put on a critical path

**Wrong approach.** Shipping `--enable-preview` to production because the API is nicer.

**Exact symptom.** The upgrade to the next JDK does not compile. Classfiles compiled with
preview enabled **refuse to load** on a different JDK version, so you cannot do a rolling
upgrade, and you cannot roll back to an older JDK either. Meanwhile your `Joiner`-based
code does not exist on 21 and your `ShutdownOnFailure`-based code does not exist on 25.

**Root cause.** That is exactly what `--enable-preview` means, by design. It is not a
warning to be suppressed; it is a contract: *this API may change, and the platform will
prevent you from pretending it did not.*

**Fix.** A judgment, not a technique:

- Use it in `orderflow`, in labs, in internal tools, and in anything where a JDK upgrade
  is a coordinated event you control.
- On a production critical path, wrap it. Put the structured fan-out behind **your own
  interface** so the preview API appears in exactly one class. When the shape changes, you
  edit one file.
- **Say this in the interview.** "I'd use it, behind an interface, and I'd expect to
  rewrite that one class at the next LTS" is a senior answer. "It's preview so we can't use
  it" is timid, and "we use it everywhere" is careless.

**Observable:** try it. Compile a preview class on JDK 21 and run it on JDK 25. Read the
error. That is ten minutes and you will never forget it.

---

## Hands-on proof

### Setup

```bash
java --version                      # RECORD IT. Everything below depends on it.
```

Then, before anything else, **open the javadoc for `StructuredTaskScope` and `ScopedValue`
in your JDK and list the actual public methods.** Write that list down. Diff it against
the shapes in this document. That diff is Proof 1, and it is the most valuable ten minutes
in this topic.

### Proof 1 — establish what API you actually have

```bash
# The shipped javadoc for your exact JDK. Do not use a search engine for this.
open "$JAVA_HOME/docs/api/java.base/java/lang/ScopedValue.html"     # if docs are installed

# Otherwise, from the class files themselves:
javap --enable-preview -release 21 java.util.concurrent.StructuredTaskScope
javap --enable-preview -release 21 java.lang.ScopedValue
```

| What you see | What it means |
|---|---|
| `ShutdownOnFailure` / `ShutdownOnSuccess` nested classes, `fork` returning `Subtask` | You are on the Java 21 preview shape. The code in this document should compile with minor adjustments. |
| A static `open(...)` factory and a `Joiner` type, no policy subclasses | **`[JAVA 25]` You are on a later preview.** Translate every scope in this document: the policy moves from the constructor into a `Joiner` argument. The four rules are unchanged. |
| `javap` reports the class does not exist | Preview classes need `--enable-preview`; check the flag and that your `-release` matches your JDK. |
| Method names that match neither | A round I have not described. **Trust the javadoc, not this document.** Write down what you found. |

**This proof is not optional and it is not busywork.** Every other proof depends on it, and
the habit — check the API in your JDK before trusting any source, including this one — is
the actual lesson of the version situation.

### Proof 2 — prove cancellation happens

Run `ScopeBasics` from Example 1, then instrument it to prove that the sibling was
*interrupted* rather than merely ignored:

```java
static String slowOk(String name, long ms) throws InterruptedException {
    long t0 = System.currentTimeMillis();
    try {
        Thread.sleep(ms);
        return name + "-ok";
    } catch (InterruptedException e) {
        System.out.printf("%s INTERRUPTED after %d ms (asked for %d)%n",
                name, System.currentTimeMillis() - t0, ms);
        throw e;
    }
}
```

**What to look for:** the elapsed time in the `INTERRUPTED` line versus the requested sleep.

| What you see | What it means |
|---|---|
| `INTERRUPTED after ~0 ms (asked for 3000)` | Cancellation is real and prompt. The scope interrupted the thread within milliseconds of the sibling's failure. |
| No `INTERRUPTED` line, program takes 3 s | The policy did not fire, or your task swallows interrupts. Check that the failing fork actually threw. |
| `INTERRUPTED after ~3000 ms` | Something is not responding to interruption. If this is a real blocking call, that call is not interruptible — a genuinely important thing to discover about a downstream client. |

### Proof 3 — see the scope in a thread dump

This is the operational payoff, and almost nobody knows it exists.

```java
// Add to ScopeBasics, inside the scope, before join():
System.out.println("pid=" + ProcessHandle.current().pid());
Thread.sleep(20_000);      // hold the scope open so you can dump it
```

```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/scope.json

# The virtual threads, and the scope structure that owns them:
grep -c '"isVirtual": true' /tmp/scope.json
python3 -m json.tool /tmp/scope.json | head -100
```

**What to look for:** whether the JSON groups the forked virtual threads under a container
representing the scope, and whether that container names the owner thread.

| What you see | What it means |
|---|---|
| Forked threads appear grouped under a scope container with a parent | **The structure is visible to operations.** In an incident you can see who forked whom. With `CompletableFuture` the dump shows a flat pool and no relationships whatsoever. |
| Only a flat list of virtual threads | Your JDK's dump format may differ, or the scope had already closed. Re-check while the scope is open. |
| Nothing under `isVirtual` at all | You dumped after the program exited, or the forks had completed. |
| `jstack` shows almost nothing | Expected. `jstack` shows carriers. **The JSON dump is the tool now** (Topic 101). |

### Proof 4 — prove the `ScopedValue` binding is gone after the block

```java
static final ScopedValue<String> K = ScopedValue.newInstance();

public static void main(String[] args) {
    System.out.println("before: bound=" + K.isBound());
    ScopedValue.where(K, "value").run(() ->
        System.out.println("inside: bound=" + K.isBound() + " value=" + K.get()));
    System.out.println("after:  bound=" + K.isBound());

    // And prove it holds when the block THROWS:
    try {
        ScopedValue.where(K, "value").run(() -> { throw new RuntimeException("boom"); });
    } catch (RuntimeException ignored) { }
    System.out.println("after throw: bound=" + K.isBound());
}
```

**What to look for:** `bound=false` in both "after" lines.

| What you see | What it means |
|---|---|
| `after: bound=false` and `after throw: bound=false` | **The cleanup you did not write, including on the exceptional path.** This is the exact behaviour a `ThreadLocal` gives you only if every caller remembers a `finally`. |
| `after throw: bound=true` | Would be a JDK bug. You will not see this. |

Now do the same experiment with a `ThreadLocal` and *deliberately omit* the `finally`,
running the block twice on a pooled thread:

**What to look for:** whether the second run sees the first run's value.

| What you see | What it means |
|---|---|
| Second run reads the first run's value | **Topic 79's leak, and a cross-request data exposure.** On a real service where the value is a tenant or a `SecurityContext`, this is a security incident. |

### Proof 5 — the orphan, measured

The most important measurement in this document. Instrument the downstream client:

```java
// A counter on every outbound call, and one on every served request (Topic 118).
Counter downstreamCalls = registry.counter("orderflow.downstream.calls", "target", "product");
```

Then run the fault-injection scenario from Example 2 step 4 against both versions.

```
ratio = rate(orderflow_downstream_calls_total{target="product"})
      / rate(http_server_requests_seconds_count{uri="/orders/{id}"})
```

| What you see | What it means |
|---|---|
| Ratio at or near 1.0 in both versions when healthy | Correct baseline. Both versions make one call per request. |
| Ratio **above 1.0** for the `allOf` version during the fault | **The orphans, quantified.** Every 0.1 above 1.0 is work done for requests that already returned an error. |
| Ratio staying at 1.0 for the structured version during the fault | Cancellation is doing its job at the system level, not just in a unit test. |
| Ratio above 1.0 in the structured version too | Something in a subtask is not interruptible, or there is a retry inside the client you forgot about. Both worth finding. |

**Graph this ratio permanently.** It is one of the few metrics that detects a class of bug
rather than a symptom, and almost no team has it.

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the failure and written
down what you saw.

### The scenario

Reproduce the orphaned-sibling leak on `orderflow`'s order-detail fan-out, prove it with a
number rather than with reasoning, then fix it with a structured scope and prove the number
changed.

### Part A — the standalone version, to learn the shape

`src/main/java/com/orderflow/lab/OrphanDrill.java`:

```java
package com.orderflow.lab;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class OrphanDrill {

    /** Counts work that actually EXECUTED, regardless of whether anyone wanted it. */
    static final AtomicInteger downstreamCallsCompleted = new AtomicInteger();
    static final AtomicInteger downstreamCallsStarted   = new AtomicInteger();

    static String slowDownstream(String name, long ms) throws InterruptedException {
        downstreamCallsStarted.incrementAndGet();
        try {
            Thread.sleep(ms);                              // the degraded product service
            downstreamCallsCompleted.incrementAndGet();    // <-- WASTED if nobody wants it
            return name;
        } catch (InterruptedException e) {
            System.out.printf("  %s cancelled%n", name);
            throw e;
        }
    }

    static String failingDownstream() {
        downstreamCallsStarted.incrementAndGet();
        throw new IllegalStateException("payment gateway 500");
    }

    // ---- version 1: CompletableFuture.allOf -------------------------------------
    static void withAllOf(ExecutorService pool) {
        var slow = CompletableFuture.supplyAsync(() -> {
            try { return slowDownstream("product", 2_000); }
            catch (InterruptedException e) { throw new CompletionException(e); }
        }, pool);
        var fail = CompletableFuture.supplyAsync(OrphanDrill::failingDownstream, pool);

        try {
            CompletableFuture.allOf(slow, fail).join();
        } catch (CompletionException e) {
            System.out.println("  request FAILED (allOf): " + e.getCause().getMessage());
        }
    }

    // ---- version 2: structured scope --------------------------------------------
    // Java 21 preview SHAPE. Translate if your JDK differs -- Proof 1.
    static void withScope() throws InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            scope.fork(() -> slowDownstream("product", 2_000));
            scope.fork(OrphanDrill::failingDownstream);
            scope.join();
            scope.throwIfFailed();
        } catch (ExecutionException e) {
            System.out.println("  request FAILED (scope): " + e.getCause().getMessage());
        }
    }

    public static void main(String[] args) throws Exception {
        String mode = args.length > 0 ? args[0] : "allof";
        int requests = 20;

        ExecutorService pool = Executors.newFixedThreadPool(64);
        long t0 = System.currentTimeMillis();

        for (int i = 0; i < requests; i++) {
            System.out.printf("request %d%n", i);
            if (mode.equals("allof")) withAllOf(pool); else withScope();
        }

        long served = System.currentTimeMillis() - t0;
        System.out.printf("%nAll %d requests RETURNED after %d ms%n", requests, served);
        System.out.printf("downstream started=%d completed=%d%n",
                downstreamCallsStarted.get(), downstreamCallsCompleted.get());

        System.out.println("Sleeping 3 s to see what is STILL RUNNING...");
        Thread.sleep(3_000);
        System.out.printf("AFTER THE WAIT: started=%d completed=%d%n",
                downstreamCallsStarted.get(), downstreamCallsCompleted.get());

        pool.shutdownNow();
    }
}
```

### Commands

```bash
java --version

java --source 21 --enable-preview OrphanDrill.java allof  2>&1 | tee run-allof.log
java --source 21 --enable-preview OrphanDrill.java scope  2>&1 | tee run-scope.log
```

### What to capture, before reading on

For both runs, write down:

1. The elapsed time before "All 20 requests RETURNED".
2. `downstream started` and `completed` **immediately** after the requests returned.
3. `downstream started` and `completed` **after the 3-second wait**.
4. Whether any `product cancelled` lines appeared, and how many.
5. Whether the JVM exited promptly.

**Now write one sentence predicting the difference in `completed` between the two runs.**
Then read on.

### How to read it

| What you see | What it means |
|---|---|
| **allOf:** all 20 requests return quickly, `completed` is near 0 immediately after | Correct so far — every request failed fast on the payment branch. This part looks fine and is why the bug hides. |
| **allOf:** after the 3-second wait, `completed` has climbed to about 20 | **The orphans, counted.** Twenty product calls ran to completion, each for a request that had already returned an error two seconds earlier. In production those are twenty real HTTP calls to a service that is already struggling. |
| **allOf:** no `product cancelled` lines at all | Nothing was cancelled, because nothing could be. `allOf` has no cancellation. |
| **scope:** all 20 requests return quickly, and 20 `product cancelled` lines appear | **The fix, working.** Each failure cancelled its sibling within milliseconds. |
| **scope:** after the wait, `completed` is still near 0 | **The number that proves it.** No wasted downstream work. This single comparison is the whole topic. |
| **scope:** requests take noticeably longer than `allOf` | `close()` waits for cancelled subtasks to actually terminate. That wait is the guarantee you are buying. If it is long, something is not responding to interruption — worth investigating. |
| Both runs identical | Check that the failing fork really throws, and that you ran the mode you think you ran. |
| Compile error in `withScope` | **Expected on a JDK with a different preview shape.** Go back to Proof 1, read your javadoc, and translate. That translation is part of the drill. |

### Part B — on `orderflow`, under the Topic 65 baseline

1. Point `orderflow`'s `ProductClient` at a stub that sleeps 30 seconds (Topic 61's
   Testcontainers or a WireMock container).
2. Point `PaymentClient` at a stub returning HTTP 500 in 20 ms.
3. Add the `orderflow.downstream.calls` counter to every client (Topic 118).
4. Run the Topic 65 mix at the recorded baseline arrival rate for five minutes against the
   `allOf` version.
5. Record: order-detail p99, **catalogue p99**, error rate, and the
   downstream-calls-to-requests-served ratio.
6. Deploy the structured version. Repeat identically.

**What to look for:** the ratio, and the catalogue endpoint's p99.

**How to read it:** the catalogue endpoint calls neither the product service nor the
payment service. If its p99 degrades in step 4 and does not in step 6, you have
demonstrated that orphaned work in one endpoint degrades an unrelated endpoint through a
shared pool — the third distinct route to the same blast-radius failure you met in
Topics 100 and 101. If the ratio is above 1.0 in step 4 and at 1.0 in step 6, you have
quantified the orphaned work.

### Part C — argue against yourself

Before you write the fix up, write down the strongest case **against** the structured
version. It should include, at minimum:

- It is a preview API. The classfiles are version-locked. The shape has already changed
  once and may change again.
- It is more verbose. The two-scope version of "payment may fail, product may not" is
  noticeably longer than `allOf` plus an `exceptionally`.
- The cancellation is cooperative, so a downstream client that ignores interruption gives
  you the guarantee's cost without its benefit.
- At the healthy baseline it makes nothing faster.

Then write the rebuttal. **A recommendation that has not stated its own downside is not a
recommendation, it is advocacy** — and that distinction is the entire content of
Topics 107 and 131.

### What the drill proves

1. **The bug is invisible until you count.** Both versions "work". Both return the right
   status code at the right time. The only difference is work you never see, in a service
   you do not own, during an incident you did not cause. **The metric is the proof and
   the reasoning is not.**
2. **Lifetime is a structural property, not a discipline.** You did not write cancellation
   code in the structured version. You could not have forgotten it.
3. **The blast radius pattern is now familiar.** Blocked FJP workers (Topic 100), pinned
   carriers (Topic 101), orphaned futures (here) — three mechanisms, one shape: work that
   holds a shared resource degrades something that has no relationship to it. Recognising
   that shape quickly is most of production debugging.

---

## Measurement

### The standing rule

A naive `System.nanoTime()` loop is the **wrong** way to measure JVM performance — it
measures JIT warm-up, dead-code elimination, on-stack replacement, and ambient noise.
**Topic 77 (JMH)** is where you learn to do it properly.

For this topic specifically there is a sharper version of the rule: **do not benchmark
structured concurrency for throughput.** It is not a throughput feature. At the healthy
baseline it will match `allOf` within noise, because both make the same number of
concurrent calls. Benchmarking it and reporting "no improvement" is measuring the wrong
thing and will lead you to the wrong conclusion.

**Measure it on failure behaviour**: orphaned work, blast radius, and bounded latency under
a downstream fault. Those are counters and percentiles from the Topic 65 harness, not
microbenchmarks.

### The instrument for each claim

| Claim | Instrument | What invalidates it |
|---|---|---|
| "No orphaned work" | Downstream-calls-to-requests-served ratio under an injected fault | Measuring when everything is healthy (the ratio is 1.0 either way) |
| "Siblings are cancelled promptly" | Elapsed time in the `INTERRUPTED` log line inside the subtask | A client that swallows `InterruptedException` |
| "The fan-out is bounded by our deadline" | Endpoint p99 during an injected 30-second downstream stall | No fault injected |
| "The tenant reaches every fork" | `TenantContext.require()` throwing loudly, plus a test that forks and asserts | Only testing the happy path with one tenant |
| "Context memory dropped" | `jcmd GC.class_histogram` at matched concurrency, before and after | Measuring at idle; forgetting `SecurityContextHolder` is still `ThreadLocal` |
| "No thread outlives the scope" | `jcmd Thread.dump_to_file -format=json` after the request completes | Dumping while the request is in flight |
| "Nothing pins" | JFR `jdk.VirtualThreadPinned` (Topic 101) — **forks are virtual threads** | Assuming structured concurrency protects you from pinning. It does not. |

### The dashboard

```java
@Bean
MeterBinder fanOutMetrics(MeterRegistry registry) {
    return r -> { /* register the counters below at their call sites */ };
}

// At every outbound client call:
registry.counter("orderflow.downstream.calls", "target", "product").increment();

// At every fan-out completion, tagged by outcome:
registry.counter("orderflow.fanout.result", "endpoint", "order-detail",
                 "outcome", "ok|failed|deadline|degraded").increment();

// Timer on the whole fan-out, so the deadline is visible as a cliff:
registry.timer("orderflow.fanout.duration", "endpoint", "order-detail")
        .record(elapsed);
```

**The two alerts worth having:**

1. **`downstream.calls` rate divided by served-request rate above 1.1 for five minutes.**
   That is orphaned work or an undocumented retry. Almost nobody has this alert and it
   detects a whole class of bug.
2. **`fanout.result{outcome="deadline"}` above zero.** Your deadline is firing, which means
   a downstream is slower than your product decision allows. That is information, not
   necessarily a fault.

### Against the Topic 65 baseline — the rules

1. Same dataset, same JVM flags, same container limits, same seed.
2. **Open-model arrival rate.** A closed-loop VU test cannot show you a blast-radius
   effect, because the load generator throttles itself when the service slows.
3. **Two configurations of the world, not one:** healthy, and with an injected downstream
   fault. **The healthy comparison should show no difference. Report that explicitly** —
   an honest "no change here" makes the "large change there" credible.
4. Three repetitions of each, reporting the spread.
5. Watch the **unaffected** endpoint. The catalogue p99 during an order-detail fault is the
   blast-radius measurement and it is the one people forget to record.

---

## Practice exercises

### 1 — easy: establish the ground truth for your JDK

Produce a one-page reference for **your** environment, every line traceable to a command
you ran:

1. `java --version`.
2. The full public method list of `StructuredTaskScope` and `ScopedValue` on your JDK,
   from `javap` or the shipped javadoc.
3. A diff between that list and the shapes used in this document, with a note on each
   difference.
4. The exact `javac` and `java` command lines that compile and run a preview class in your
   setup.
5. What happens when you run a preview-compiled class on a different JDK. Paste the error.
6. Whether `ScopedValue.get()` throws or returns null when unbound, proven by running it.

**The deliverable is a file you keep**, because every other exercise in this topic depends
on it and it goes stale with every JDK upgrade.

### 2 — medium: the audit (combines Topics 01–101)

Audit `orderflow` for structured-concurrency and context-propagation readiness. For each
finding: file, line, the topic it comes from, the symptom, the fix, and the risk of the
fix.

Find at least:

1. Every `CompletableFuture.allOf` / `anyOf` (Topic 91). For each: what leaks when one
   branch fails, and does that branch have a side effect?
2. Every `supplyAsync` with no executor argument (Topics 91, 100) — the common pool.
3. Every `ThreadLocal` and `InheritableThreadLocal` on the request path (Topics 56, 79,
   120). For each: is it yours to change, or the framework's?
4. Every `finally { xxx.remove(); }` — each one is a place where `ScopedValue` deletes the
   `finally`.
5. Every fan-out with **no deadline** (Topic 111 territory). What bounds the request's
   latency today? If the answer is "the downstream's timeout", who chose that number?
6. Every place a fan-out branch has a **different failure policy** from its siblings.
   Those are the places that need nested scopes, and they are where `allOf` plus
   `exceptionally` hides a policy nobody wrote down.
7. Every `synchronized` inside code that a fork would execute (Topic 101). Forks are
   virtual threads; pinning applies.

**Deliverable:** the audit table, plus **one** rewritten fan-out with before/after code and
the measurement plan. Not all of them — one, done properly.

### 3 — hard: the tenant-context migration, end to end

Migrate `orderflow`'s tenant propagation from `ThreadLocal` to `ScopedValue`, and the
order-detail fan-out from `allOf` to a structured scope. Then prove both.

**Method:**

1. Write the ground-truth file from exercise 1 first. Everything depends on which API you
   actually have.
2. Put the preview API behind **your own interface** — `FanOut` with a method that takes
   suppliers and a deadline — implemented once by the structured version. Trap 6's fix.
   **This is a design requirement, not a suggestion.**
3. Migrate the tenant context to `ScopedValue`. Delete every `finally { clear(); }`.
4. Write a test that **fails on the old code**: fork a subtask and assert the tenant is
   visible inside it. On the `ThreadLocal` version with a plain executor, it will not be.
5. Migrate the fan-out, with the nested-scope structure for the optional payment branch and
   a 400 ms deadline.
6. Measure, per the Measurement section: healthy baseline (expect no change), and under an
   injected fault (expect a large change in the ratio and in catalogue p99).
7. Run a JFR recording during the fault and check `jdk.VirtualThreadPinned` — forks are
   virtual threads and the pinning rules from Topic 101 still apply.
8. Then do the thing most people skip: **run your preview-compiled artifact on a different
   JDK** and document exactly what happens. Include it in the writeup as a named risk with
   a mitigation.

**Deliverable — at most two pages:**

- The before/after code for one fan-out and one context binding.
- The measurement table: healthy and faulted, both versions, with the downstream ratio and
  the unaffected endpoint's p99.
- A "risks and mitigations" section naming the preview-API lock-in, the cooperative-
  cancellation caveat, and the verbosity cost.
- A recommendation with a rollback trigger.

**The grading criterion is whether someone who disagrees with you could act on this
document.** That is the Phase 12 standard, and this is a rehearsal for it — as is
Topic 107, which comes next.

---

## Interview questions

### Q1 — "What does `StructuredTaskScope` give you that `CompletableFuture.allOf` doesn't?"

**MID-LEVEL.** "It's a cleaner API for running tasks in parallel and waiting for them. It
uses virtual threads, so it scales better, and the code reads more like normal sequential
code."

**SENIOR.** "Three things, and the first is the one that matters.

**Lifetime.** A scope's subtasks cannot outlive the block. `close()` waits for and
interrupts every fork, so there is no path out of that method — including a thrown
exception — that leaves work running. `allOf` is a completion combinator: it observes
completion and has no opinion about what should stop. If one branch fails, `join()` throws
and the other branches keep running, holding connections and consuming a downstream's
budget for a request that already returned a 502. And `cancel(true)` on a
`CompletableFuture` does not interrupt the supplier — it only completes the future
exceptionally.

**Error propagation with sibling cancellation**, as a policy rather than as code you
remembered to write.

**Observability.** A structured scope appears in `jcmd Thread.dump_to_file -format=json`
as a tree, so during an incident you can see who forked whom. A `CompletableFuture`
fan-out shows a flat pool and no relationships at all.

I'd add the caveat, because it is part of the answer: it is still a preview API and the
shape changed between preview rounds. I'd use it behind my own interface so the migration
is one file."

**What separates them.** The mid answer describes the ergonomics. The senior answer names
the leak, names the specific `cancel(true)` semantics, gives the observability argument
that almost nobody mentions, and volunteers the preview caveat unprompted.

**Follow-up:** *"How would you prove the leak exists in our codebase?"* → Counter on every
outbound call, counter on served requests, inject a fault in one branch, and graph the
ratio. Above 1.0 is orphaned work. It takes an afternoon and it produces a number instead
of an argument.

### Q2 — "Why not just use `ThreadLocal`? It's been fine for twenty years."

**MID-LEVEL.** "`ThreadLocal` can leak if you forget to call `remove()`, and virtual
threads make it use more memory. `ScopedValue` is the newer replacement."

**SENIOR.** "It was fine because we had 200 pooled threads and a `finally` block. Two
things changed.

On a **pooled** thread, the thread outlives the request, so a missing `remove()` retains
the value forever and — worse — the next request on that thread can read the previous
request's data. If that value is a tenant id or a `SecurityContext`, that's a
cross-tenant exposure, not a memory issue.

On **virtual** threads, the `remove()` problem disappears because the thread dies, but
you've multiplied a per-thread cost by fifty thousand. Nothing leaks and the heap still
climbs linearly with concurrency.

`ScopedValue` removes both by changing two properties: it's immutable, and it's lexically
scoped. There is no `set()` — the only way to bind is to wrap a call — so the binding is
popped when that call returns, on the normal path and on the exceptional path. There is
nothing to remove, so there is nothing to forget. And because the value is immutable,
sharing it with a forked subtask needs no synchronisation and has no safe-publication
problem.

The read path is also different: `ThreadLocal.get()` probes a hash map hanging off the
`Thread` that outlives the call; a `ScopedValue` read walks a short binding chain on the
stack with a small cache.

Practical caveats: it's preview, and `SecurityContextHolder` is still `ThreadLocal`-backed,
so I can migrate my own contexts but not the framework's — I'd measure what's left rather
than claim I'd fixed it."

**What separates them.** Distinguishing the pooled-thread failure from the virtual-thread
failure precisely; naming the cross-tenant exposure rather than only the memory; and
knowing the framework constraint.

**Follow-up:** *"What could still go wrong with `ScopedValue`?"* → Binding a mutable
object. The immutability guarantee is on the binding, not the payload, so a
`ScopedValue<RequestContext>` holding a mutable `HashMap` gets shared across every fork and
you have an unsynchronised data race in the API designed to prevent context bugs. My review
rule is that the type parameter must be a record, an enum, a `String` or a boxed primitive.

### Q3 — "Would you use this in production today?"

**MID-LEVEL.** "It's preview, so no." Or: "Yes, it's the modern way to do concurrency."

**SENIOR.** "Qualified yes, behind an interface.

`--enable-preview` is a real contract: classfiles are version-locked, so a class compiled
with preview on 21 will not load on 22. That rules out a rolling JDK upgrade for any
artifact containing one, which is a genuine operational constraint and not a formality.

And the shape has already changed — the Java 21 preview used `ShutdownOnFailure` and
`ShutdownOnSuccess` subclasses; a later round moved the policy into a `Joiner` passed to a
static factory. I wouldn't quote current signatures from memory; I'd read the javadoc for
the JDK in front of me.

So: I'd put the fan-out behind my own interface, implemented in one class. Then the preview
API appears in exactly one file, the migration at the next LTS is one file, and I get the
correctness benefit now. For a service where a JDK upgrade is a coordinated event we
control, that's a good trade.

What I would not do is scatter `StructuredTaskScope` through forty service classes."

**What separates them.** Neither reflexively refusing nor reflexively adopting. Knowing the
specific mechanism of the constraint (version-locked classfiles), knowing that the API has
already changed, and proposing a concrete containment strategy.

**Follow-up:** *"What if we can't use preview at all?"* → Then bound the waiting with
`orTimeout` per branch and add a cooperative cancellation flag the suppliers check. Be
honest that `orTimeout` bounds the wait and not the work — the supplier keeps running until
it checks the flag. That's the boilerplate structured concurrency removes, and writing it
out is a good argument for adopting the API later.

### Q4 — "Your fan-out has three branches. Two must succeed; one is optional. How?"

**MID-LEVEL.** "Use `ShutdownOnFailure` and catch the exception from the optional one."

**SENIOR.** "That doesn't work, and the reason is worth stating: `ShutdownOnFailure` fires
on *any* subtask failure, so the optional branch failing would cancel the two required
ones. The policy is per-scope, not per-fork.

The idiom is **nested scopes**: an inner scope for the optional branch, whose failure is
caught and converted to a fallback value, inside an outer scope with shutdown-on-failure
for the required branches. Scopes compose; policies don't.

I'd add a deadline on both — `joinUntil` rather than `join` — because otherwise the
request's latency is whatever the slowest downstream chooses, and that number should be a
product decision. And I'd count the fallbacks as a metric, because a degraded response that
nobody can see is indistinguishable from a healthy one on a dashboard.

Worth noticing what happened there: `allOf` let me leave the policy unstated, and the
structured version forced me to write it down. The extra verbosity is the requirement
becoming visible."

**What separates them.** Knowing the policy is per-scope; reaching for nesting; adding the
deadline and the fallback metric unprompted; and reframing the verbosity as the requirement
becoming explicit.

**Follow-up:** *"What if two branches race and you want the first answer?"* → That's the
shutdown-on-success policy — fork both, take the first success, and the scope cancels the
loser. Which is a genuinely useful hedging pattern, and note that with `allOf` you would
have had to cancel the loser yourself, which as established you cannot actually do.

### Q5 — "How do you get a request's tenant into a forked subtask?"

**MID-LEVEL.** "Pass it as a parameter into the lambda, or use an
`InheritableThreadLocal`."

**SENIOR.** "Parameters work and are honestly fine for one or two values — I'd rather
thread a parameter than reach for ambient state, and I'd say that first.

Where it stops working is a context that's needed by code you don't control: a client
interceptor adding a header, a repository choosing a schema, an audit aspect. Threading a
parameter through those means changing signatures you don't own.

`InheritableThreadLocal` is the old answer and it's poor: it copies the whole map into the
child at thread creation, which is the wrong cost shape at virtual-thread counts, and it
has no lifetime relationship — the child can outlive the parent and retain the value.

`ScopedValue` is the designed answer. Bind at the request boundary; a structured `fork`
inherits the bindings in effect at the fork point. It's safe because the value is immutable
and the scope guarantees the child terminates before the binding is popped, so a child can
never read a dead binding.

Two things I'd insist on. The value must be genuinely immutable — a record, not an object
with a mutable map — because the immutability guarantee is on the binding, not the payload.
And I'd have exactly one read point that throws a message naming the likely cause, so
'tenant missing' points at the filter instead of surfacing as an NPE three layers away."

**What separates them.** Preferring parameters first and saying why ambient state is the
exception; knowing precisely why `InheritableThreadLocal` is worse; and naming the
immutability requirement on the payload, which is the trap most people fall into.

**Follow-up:** *"What about a plain `ExecutorService` fork — does the binding cross?"* →
No. Inheritance is a property of the structured fork, not of thread creation generally. If
you submit to an executor from inside a binding, the task runs unbound and `get()` throws.
That's a loud failure rather than a silent wrong answer, which is the right behaviour — and
it's a good test to write.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `Promise.all` and `CompletableFuture.allOf` have the same hole. Explain why the hole is
   nearly harmless in Node and genuinely expensive in a JVM service. Name at least two
   properties of the runtimes that make the difference.

2. Structured concurrency is described as "structured programming for concurrency". Push
   the analogy until it breaks. What is the `goto` here, what is the block, and what is the
   concurrency equivalent of a `break` out of a nested loop — does the API have one?

3. `close()` waits for every subtask. Construct a situation where that wait is unbounded,
   and say what you would do about it. Does adding a deadline to `join` help? Why or why
   not?

4. A `ScopedValue` binding is popped when the block returns. Design an alternative
   implementation that allows a `set()` and explain, mechanically, what guarantee you would
   lose. Then say whether any real use case needs it.

5. Forks inherit scoped-value bindings by reference, not by copy, and this is safe. State
   the two properties that make it safe, and construct a change to either one that would
   make it unsafe.

6. You have a fan-out where each branch takes 50 ms and there are 12 branches. Predict the
   difference in wall-clock latency between `allOf` and a structured scope when everything
   succeeds, and justify the prediction from the mechanism. Now predict it when branch 3
   fails at 5 ms.

7. Both structured concurrency and virtual threads make concurrency cheaper to write
   correctly. Neither gives you backpressure. Explain, precisely, why that is a different
   kind of problem that neither could have solved — and what that implies for Topic 107's
   decision.

---

## Quick reference card

### Structured concurrency, in five lines

```
fork()  -> starts a NEW virtual thread; records it as a child of this scope.
join()  -> wait for all, or until the policy short-circuits. Must be called.
policy  -> shutdown-on-failure cancels siblings; shutdown-on-success cancels losers.
close() -> the closing brace. Interrupts, then WAITS for every child. Cannot be skipped.
scope   -> confined to its creating thread. Not a bean. Not a pool. A local variable.
```

### Java 21 preview shape — VERIFY AGAINST YOUR JAVADOC

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<A> a = scope.fork(() -> callA());
    Subtask<B> b = scope.fork(() -> callB());
    scope.joinUntil(Instant.now().plusMillis(400));   // deadline, not join()
    scope.throwIfFailed(MyException::new);
    return combine(a.get(), b.get());                 // read ONLY between join and close
}
```

> `[JAVA 25]` Later previews replaced the policy subclasses with a static `open(...)`
> taking a `Joiner`, and moved the deadline into scope configuration. **Read your javadoc.
> Do not trust this card across a JDK upgrade.**

### `ScopedValue`, Java 21 preview shape

```java
static final ScopedValue<TenantId> TENANT = ScopedValue.newInstance();

ScopedValue.where(TENANT, tenant).run(() -> ...);     // void
var result = ScopedValue.where(TENANT, tenant).call(() -> ...);   // value, can throw

TENANT.get();        // throws if unbound -- LOUD, which is the point
TENANT.isBound();    // check when the binding is genuinely optional
// There is NO set(). That absence is the feature.
```

### Preview build commands

```bash
java --version                                          # ALWAYS FIRST
javac --release 21 --enable-preview -d out src/**/*.java
java  --enable-preview -cp out com.orderflow.Application
java  --source 21 --enable-preview SingleFile.java

# --release/--source MUST match the JDK you are running.
# Preview classfiles will NOT load on a different JDK version. By design.
```

### `ThreadLocal` versus `ScopedValue`

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Scope | The thread's lifetime | One call — lexical |
| Mutation | `set()` any time | None. Bind by wrapping a call. |
| Cleanup | `remove()` in a `finally`, by convention | Automatic, including on throw |
| Leak on pooled thread | **Yes** (Topic 79) | No |
| Cost at 50k threads | 50k maps (Topic 101) | Short binding chain per call |
| Missing value | `null`, silently | Throws, loudly |
| Inheritance | `InheritableThreadLocal` copies the map | Structured fork inherits by snapshot |

### Diagnostics

```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/t.json   # scopes appear as a tree
grep -c '"isVirtual": true' /tmp/t.json
jfr print --events jdk.VirtualThreadPinned rec.jfr        # forks are virtual threads
javap --enable-preview -release 21 java.util.concurrent.StructuredTaskScope
```

```
# The metric that proves the whole topic:
rate(downstream_calls_total) / rate(requests_served_total)   > 1.0  ==  orphaned work
```

### Gotchas checklist

- [ ] Bind only immutable values. Record, enum, `String`, boxed primitive. Nothing else.
- [ ] `joinUntil(deadline)`, never a bare `join()`, on anything serving a request.
- [ ] Different failure policies mean nested scopes, not one scope plus a `catch`.
- [ ] Read subtask results only between `join()` and `close()`. Never let one escape.
- [ ] A scope is a local variable. Not a field, not a bean, not shared across threads.
- [ ] A scope is not a concurrency limiter. Add a `Semaphore` if you need one (T97).
- [ ] Forks are virtual threads — pinning rules from Topic 101 still apply.
- [ ] Put the preview API behind your own interface. One file to migrate.
- [ ] Delete the `finally { remove(); }` — that deletion is the `ScopedValue` win.
- [ ] Verify every signature against your JDK's javadoc. Including the ones here.

---

## When would I use this at work?

**1. Any fan-out where a branch has a side effect.**

Reviewing a PR with `allOf` over three service calls, the question is: if one fails, what
do the others do that we would not want done? Write a cache entry? Charge something?
Consume a rate-limit budget from a service that is already struggling? If the answer to any
of those is yes, the leak is a real defect and not a theoretical one — and the structured
version makes it unwritable rather than merely fixed.

**2. Migrating request context off `ThreadLocal` as part of a virtual-thread rollout.**

This is the natural second half of Topic 101's migration and it has a security dimension
people miss. The pooled-thread `ThreadLocal` with a missing `remove()` is a cross-tenant
data exposure, not a memory problem. Framing it that way changes how quickly it gets
prioritised, and it is an accurate framing rather than a rhetorical one.

**3. Making concurrency legible during an incident.**

`jcmd Thread.dump_to_file -format=json` on a structured service shows the tree of who
forked whom, at 3am, with no prior knowledge of the codebase. The same dump on a
`CompletableFuture` codebase shows a pool of anonymous threads. That difference is worth
more at 3am than any amount of throughput, and it is the argument that usually persuades an
operations-minded reviewer when the correctness argument has not.

---

## Connected topics

**Prerequisites:**

- **17 — Immutability and safe publication:** why an immutable binding can be shared with a
  forked thread with no synchronisation. Trap 3 is what happens when the payload is not
  actually immutable.
- **27 — Records:** the right type for a `ScopedValue` payload, with the shallow-
  immutability caveat that Trap 3 turns on.
- **56 — Spring Security:** `SecurityContextHolder` is `ThreadLocal`-backed and is not
  yours to change. Measure what remains after your migration.
- **79 — Memory leaks:** the `ThreadLocal`-on-a-pooled-thread leak, which `ScopedValue`
  makes structurally impossible.
- **90 — Executors and pool sizing:** the model a scope replaces for fan-out — and the
  pool you must remember to delete.
- **91 — `CompletableFuture`:** the fan-out this topic rewrites, and the `allOf` hole this
  topic closes. Re-read its drill before starting this one.
- **97 — Coordination primitives:** `Semaphore` as the concurrency limiter a scope does
  not provide.
- **100 — `ForkJoinPool`:** the scheduling substrate underneath the forked virtual threads.
- **101 — Virtual threads:** forks are virtual threads. Pinning, carrier starvation and
  the `ThreadLocal` multiplication all carry over unchanged. **Read 101 first.**

**Also relevant:**

- **09 — Exception API design:** `throwIfFailed(MyException::new)` is where downstream
  failures enter your own hierarchy.
- **46 — `ProblemDetail`:** where that mapped exception becomes an HTTP response.
- **61 — Testcontainers:** how you inject the downstream fault the drill needs.
- **65 — The load baseline:** the numbers you compare against, healthy and faulted.
- **111 — Retries and timeouts:** the deadline on the scope is one half of a timeout
  budget; the other half is the client's own configuration, and they must agree.

**This unlocks:**

- **105 — Backpressure:** the thing neither virtual threads nor structured concurrency
  give you, and therefore the reason Topic 107 is not a foregone conclusion.
- **107 — The Loom-vs-reactive decision:** consumes this topic's cancellation and
  composition story directly. Structured concurrency is Loom's answer to "but reactive has
  composition" — a partial answer, and you should be able to say which part.
- **118 — Metrics:** the downstream-calls-to-requests-served ratio belongs on a dashboard.
- **119 — Tracing:** trace context across forks has exactly the shape of the tenant
  context here, and the same solution.
- **120 — MDC:** the logging half of the same context-propagation problem, and the one
  reactive solves differently (Topic 108).
- **131 — Design-doc authorship:** the hard exercise's deliverable is a rehearsal for it.

---

*Java baseline 21. **Both APIs in this document are preview features.** Structured
concurrency and scoped values have been through multiple preview rounds and the API shape
changed between them: the Java 21 preview constructs scopes with `ShutdownOnFailure` /
`ShutdownOnSuccess` subclasses, and a later round `[JAVA 25]` moved to a static factory
taking a `Joiner` policy object, with corresponding changes to result accessors and
deadline handling. Every code block here is the **Java 21 shape** and should be treated as
a structure to translate, not a signature to copy. Run `java --version`, read the javadoc
shipped with your JDK, and let `javac` correct you. The concepts — lexical lifetime
binding, sibling cancellation, immutable lexically-scoped context — are stable and are what
you are actually learning. Preview classfiles are version-locked and will not load on a
different JDK; that is by design and it is a real constraint on production adoption.*
