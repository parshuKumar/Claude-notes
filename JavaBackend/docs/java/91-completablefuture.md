# 91 — `CompletableFuture` vs JS Promises, and Exactly Where the Analogy Breaks

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: the `orderflow` order-detail endpoint — `GET /orders/{id}` needs the order header, its lines with product data, the current inventory position, and the payment status. Four dependent reads that today run one after another. This topic makes them run at the same time, and then shows you the two ways that destroys the service.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> A `CompletableFuture` carries **no implicit executor**. It is a value that is not
> there yet, plus a stack of callbacks to run when it arrives. Nothing in the object
> says where those callbacks run.
>
> **Which thread runs a continuation depends on two things: the `*Async` variant you
> chose, and the executor argument you passed.**
>
> - `thenApply(fn)` — **no async, no executor.** The `fn` runs on whichever thread
>   completes the future. And if the future is **already complete** when you attach
>   the continuation, `fn` runs **on the calling thread**, synchronously, before
>   `thenApply` returns.
> - `thenApplyAsync(fn)` — **async, no executor.** The `fn` is submitted to
>   `defaultExecutor()`, which for a plain `CompletableFuture` is
>   `ForkJoinPool.commonPool()` — a **JVM-wide, shared, `cores − 1`** pool designed for
>   short non-blocking CPU work (Topic 25, Topic 100).
> - `thenApplyAsync(fn, executor)` — **async, explicit executor.** The `fn` runs on
>   the executor you named. This is the only form whose threading you can reason about
>   without reading the rest of the file.
>
> There is **no `await`**. `join()` and `get()` **block a real operating-system
> thread** until the value arrives. Blocking is not free and it is not hidden.
>
> Exceptions are **wrapped**. A failure inside the pipeline reaches your handler as a
> `CompletionException` (or `ExecutionException` from `get()`) whose `getCause()` is
> your real exception. `instanceof` on the wrapper is always false.
>
> And a future that **nobody joins** and that has **no exception handler attached**
> discards its exception **silently**. No stack trace. No log line. Nothing.

Two consequences you will use for the rest of your career. **`CompletableFuture` is a
composition API, not a concurrency policy** — the policy is the executor, and if you did
not name one you inherited one that is shared with every other piece of code in the JVM.
And **every `join()` is a thread you have taken out of circulation**: in Node an `await`
costs nothing but a continuation, while in Java it costs a stack, a scheduler slot and —
if that thread came from a bounded pool — a unit of your service's total capacity.

---

## The bridge from what you know

### `PARTIAL ANALOGUE.` The shape matches. The threading does not.

This is the single most important bridge in Phase 9, because the API is *so* close to
`Promise` that you will read it fluently on day one and be wrong about it for a year.

Start with what genuinely transfers:

| TypeScript / Node | Java `CompletableFuture` | How close? |
|---|---|---|
| `Promise<T>` | `CompletableFuture<T>` | Same idea: a container for a value that arrives later, or an error. |
| `p.then(v => f(v))` | `cf.thenApply(v -> f(v))` | Close. Transform the value. |
| `p.then(v => g(v))` where `g` returns a promise | `cf.thenCompose(v -> g(v))` | Close. This is the flat-map. **Using `thenApply` here gives you `CompletableFuture<CompletableFuture<T>>`**, which the compiler accepts and which then never completes the way you expect. |
| `p.then(v => { sideEffect(v); })` | `cf.thenAccept(v -> sideEffect(v))` | Same. `thenRun` is the arity-zero form. |
| `p.catch(e => fallback)` | `cf.exceptionally(e -> fallback)` | Close, but see break #3 below. |
| `p.finally(fn)` | `cf.whenComplete((v, e) -> fn())` | Close. `whenComplete` sees both value and error and **passes both through unchanged**. |
| `Promise.all([a, b])` | `CompletableFuture.allOf(a, b)` | Shape only. `allOf` returns `CompletableFuture<Void>` — **it does not collect the results**. You join each future individually afterwards. |
| `Promise.race([a, b])` | `CompletableFuture.anyOf(a, b)` | Close. Returns `CompletableFuture<Object>`, untyped, because the inputs may differ. |
| `Promise.resolve(v)` | `CompletableFuture.completedFuture(v)` | Same. |
| `new Promise((res, rej) => ...)` | `new CompletableFuture<>()` + `complete(v)` / `completeExceptionally(e)` | Java's is **not** a constructor callback. You hand the raw future to the producer. |
| `async function f()` | — | **Nothing.** There is no `async` keyword and no `await`. |

If you stop reading here you will write code that compiles, passes tests on your
laptop, and takes down the service under load. The four breaks below are the document.

---

### Break 1 — WHICH THREAD runs a continuation is your decision, and you probably did not make it

In Node there is exactly one answer to "which thread runs this `.then`": the event
loop, in a microtask, after the current synchronous block finishes. You have never had
to ask.

In Java there are three different answers depending on one suffix and one argument.

```java
CompletableFuture<Order> f = loadOrder(id);

f.thenApply(this::toDto);                       // (a)
f.thenApplyAsync(this::toDto);                  // (b)
f.thenApplyAsync(this::toDto, orderIoExecutor); // (c)
```

- **(a)** runs `toDto` on **whichever thread completes `f`**. If `f` was **already
  complete** when line (a) executed — because `loadOrder` returned
  `completedFuture(...)` from a cache — then `toDto` runs **on the thread executing line
  (a)**, right now, before `thenApply` returns. That is a Tomcat request thread.
- **(b)** submits `toDto` to `ForkJoinPool.commonPool()`. JVM-wide, shared with every
  parallel stream in the process and every library that reached for the same default.
- **(c)** submits `toDto` to the executor you named and sized.

**The non-determinism in (a) is the part that hurts.** The same line runs on a different
thread depending on whether a cache was warm — so the `ThreadLocal`-backed
`SecurityContext` (Topic 56) is present on the cache-hit path and absent on the miss
path, the MDC correlation id (Topic 120) appears in some log lines and not others, and a
CPU-bound transform can end up on a database callback thread delaying the next query.
None of these fail a unit test.

### Break 2 — there is no `await`; `join()` blocks a real OS thread

```typescript
const [product, stock] = await Promise.all([getProduct(sku), getStock(sku)]);
```

That `await` suspends a *function*. The event loop carries on serving other requests.
Suspension is nearly free — a closure and a queue entry.

```java
CompletableFuture.allOf(product, stock).join();   // <-- blocks. A real thread. Right here.
```

That `join()` parks a platform thread. On a Tomcat request thread it is fine — that
thread was dedicated to this request anyway. **Inside a pool worker it is a capacity
decision**, and inside a `ForkJoinPool` worker it is a capacity decision with a surprise
attached (see Machine-level reality).

The mental correction to make now: **in Node, concurrency is free and parallelism is
impossible. In Java, parallelism is possible and concurrency costs a thread.**

### Break 3 — exceptions are wrapped in `CompletionException`

```typescript
try { await placeOrder(cmd); }
catch (e) {
  if (e instanceof InsufficientFundsError) { /* runs */ }
}
```

```java
orderFuture.exceptionally(e -> {
    if (e instanceof InsufficientFundsException) {   // <-- ALWAYS FALSE. Always.
        return OrderResult.declined();
    }
    throw new CompletionException(e);
});
```

The `e` handed to `exceptionally`, `handle` and `whenComplete` is a
**`CompletionException` wrapping your exception**. You must inspect `e.getCause()`.

There is one nasty asymmetry worth knowing before it bites you: the wrapping is not
uniform. If the exception was set by `completeExceptionally(new Foo())` and you read it
via `whenComplete` on that same future, you may see `Foo` unwrapped; if it propagated
through a `thenApply`, you see `CompletionException(Foo)`. **Never rely on the
distinction.** Write one unwrapping helper and route everything through it. There is one
in the Quick reference card.

### Break 4 — a future nobody joins discards its exception in silence

This is the break with no Node counterpart at all, and it is the one that costs money.

In Node, an unhandled promise rejection is **loud**: `UnhandledPromiseRejection` on
stderr, and since Node 15 the default is to **terminate the process**. The runtime
treats a dropped error as a bug.

In Java, this:

```java
CompletableFuture.runAsync(() -> auditLog.record(orderId), auditExecutor);
// no .join(), no .exceptionally(), no .whenComplete(). Nothing.
```

...will, if `auditLog.record` throws, complete the future exceptionally, store the
`Throwable` in the future's `result` field, and **that is all that happens**. No handler
is attached, so nothing consumes it; the future becomes garbage and the exception is
collected with it. No warning, no flag to turn one on, nothing in `-Xlog`. The only
evidence is a missing side effect, found later by reconciliation.

Topic 09 taught you that "log and continue" is a defect. **This is worse: it is
"continue", with the log removed.**

### What has no analogue at all

- **A shared, process-global default executor.** Node has no `commonPool` equivalent
  because it has no pool. `.thenApplyAsync(fn)` enqueues work onto a `cores − 1`
  resource other libraries are also using, without saying so.
- **Blocking a scheduler thread.** Node's API physically will not let you occupy the
  event loop waiting for I/O. In Java, JDBC on a common-pool worker is one legal line.
- **Parallel execution of two continuations.** Two `.then` callbacks in Node run one
  after the other on one thread. Two `thenApplyAsync` continuations in Java can run at
  the same instant on two cores, touching the same object.

---

## What is this?

`CompletableFuture<T>` implements two interfaces, and the split matters:

- **`Future<T>`** — the Java 5 interface. `get()`, `cancel()`, `isDone()`. Blocking and
  poll-shaped. This is the part you mostly do not use.
- **`CompletionStage<T>`** — the Java 8 interface. `thenApply`, `thenCompose`,
  `thenCombine`, `exceptionally`, `handle`, `whenComplete`. This is the promise-shaped
  part, and it is where you live.

`CompletableFuture` is also the *only* JDK implementation of `CompletionStage`, and it
is the only one that lets **you** complete it (`complete`, `completeExceptionally`).
Hence the name.

### The method families, and how to read any of them

Every combinator comes in three forms. Learn the pattern once and you know two hundred
method signatures:

```
thenApply     (fn)              -> runs on the completing thread, or the CALLER if already done
thenApplyAsync(fn)              -> runs on defaultExecutor()  == commonPool()
thenApplyAsync(fn, executor)    -> runs on YOUR executor
```

The families:

| Family | Signature shape | `Promise` equivalent | Notes |
|---|---|---|---|
| `thenApply` | `T -> U` | `.then(v => u)` | transform |
| `thenAccept` | `T -> void` | `.then(v => { ... })` | consume |
| `thenRun` | `() -> void` | `.then(() => { ... })` | ignore the value |
| `thenCompose` | `T -> CompletionStage<U>` | `.then(v => promise)` | **flat-map — use this for a dependent async call** |
| `thenCombine` | `(T, U) -> V` | `Promise.all([a,b]).then(([t,u]) => v)` | join two independent futures |
| `thenAcceptBoth` / `runAfterBoth` | — | — | same, without a return value |
| `applyToEither` / `acceptEither` | — | `Promise.race` | first one wins |
| `exceptionally` | `Throwable -> T` | `.catch` | **recovers**; returns a value |
| `handle` | `(T, Throwable) -> U` | `.then(v, e)` | sees both; **always runs** |
| `whenComplete` | `(T, Throwable) -> void` | `.finally` | sees both; **passes both through unchanged** |
| `allOf` | `CF... -> CF<Void>` | `Promise.all` | **does not collect results** |
| `anyOf` | `CF... -> CF<Object>` | `Promise.race` | untyped return |

### Creation and timeouts

```java
CompletableFuture.completedFuture(v)              // already done
CompletableFuture.failedFuture(t)                 // already failed  [Java 9+]
CompletableFuture.supplyAsync(sup)                // runs on commonPool  <-- the trap
CompletableFuture.supplyAsync(sup, executor)      // runs on YOUR executor
CompletableFuture.runAsync(runnable, executor)    // no value
new CompletableFuture<>()                         // you complete it yourself

cf.orTimeout(2, TimeUnit.SECONDS);                        // fails with TimeoutException
cf.completeOnTimeout(fallback, 2, TimeUnit.SECONDS);      // succeeds with a fallback
```

Both timeout methods use one shared internal single-threaded scheduler
(`CompletableFuture.Delayer`), a daemon thread named `CompletableFutureDelayScheduler`
that you will see in every thread dump. It is one thread for the whole JVM; put timeouts
on it, never work.

**Neither cancels the underlying work.** `orTimeout` completes *your* future
exceptionally; the slow JDBC call is still running and still holding its connection.
That gap is Topic 102's entire argument for structured concurrency.

---

## Why does it matter?

**1. The order-detail endpoint is four sequential reads, and latency adds up.**

`GET /orders/{id}` in `orderflow` currently costs 18 ms (header) + 24 ms (lines) + 31 ms
(products) + 15 ms (payment status) = **88 ms of pure serial database time.** Three of
those four are independent. Run them concurrently and the wall clock becomes roughly the
slowest one plus overhead, not the sum. **The endpoint's latency stops being the sum of
its dependencies and becomes the maximum.** That is the single most valuable thing
`CompletableFuture` does for a request-response service.

**2. It is the only composition tool in the JDK until Topic 102.**

Before `StructuredTaskScope`, this is how you express "do these three things at once and
combine the results" without hand-rolling latches (Topic 97). Every Java codebase
written since 2014 uses it, so you will read it whether or not you write it.

**3. The failure modes are invisible until they are catastrophic.**

Blocking on the common pool does not throw and does not log. It degrades *other,
unrelated* code in the same JVM, whose authors have no idea you exist. The symptom
appears in a subsystem you did not touch — which is why this is a drill, not a footnote.

**4. Fan-out multiplies your connection-pool demand.**

The constraint most people miss. Serial code holds **one** HikariCP connection at a
time; fan-out holds **three**. At 200 concurrent requests against a pool of 20, the
serial version queues and the parallel version queues three times harder — and can
deadlock outright (Topic 109). Parallelising reads without resizing the pool converts a
latency win into an availability incident.

---

## Machine-level reality

### A `CompletableFuture` is two fields

Read the JDK source; it is shorter than you expect.

```java
// java.util.concurrent.CompletableFuture
volatile Object result;       // either T, or null-if-incomplete, or an AltResult
volatile Completion stack;    // Treiber stack of dependent actions
```

That is the whole object. Everything else is method bodies.

**`result`** holds one of three things:

| `result` value | Meaning |
|---|---|
| `null` | not complete yet |
| a reference to your `T` | completed normally with that value |
| an `AltResult` instance | completed with `null`, **or** completed exceptionally |

`AltResult` is a tiny private class with one field, `Throwable ex`. A future completed
with a `null` value is represented as `AltResult(null)` — which is why `null` is a legal
completion value and why the implementation cannot use `null` for "incomplete" and for
"completed with null" at the same time.

**`stack`** is a **Treiber stack** — a lock-free LIFO built by CASing a head pointer.
Each node is a `Completion` object holding the dependent future, the source future(s),
the function, and the executor (which may be `null`, meaning "not async"). Attaching a
continuation is a CAS onto that stack (Topic 95's mechanism).

**Consequence you can observe:** because it is a **stack**, when several continuations
are attached to one future, they fire in **reverse attachment order**. Nothing in the
specification promises any order, so do not depend on it — but if you ever see two
`whenComplete` handlers running "backwards", that is why.

### What completion actually does

```
someFuture.complete(value):
  1. CAS result from null -> value.
     - If the CAS fails, someone else completed it first. Return false. Done.
  2. postComplete():
       while (stack != null):
         pop a Completion node (CAS the head)
         call node.tryFire(NESTED)
```

`tryFire` is where the "which thread" question is finally answered:

```
Completion.tryFire(mode):
  if (this.executor != null)          -> executor.execute(this);   // *Async form
  else                                -> run the function INLINE   // non-async form
```

So the non-async `thenApply` function is executed **inside `complete()`**, on the thread
that called `complete()` — a JDBC driver's thread if a driver completed it, one of your
executor's threads if your task did, or **your own thread** if the future was already
complete when you attached, because `thenApply` then sees a non-null `result` and calls
`tryFire` immediately, synchronously, before returning.

That last case is the mechanical statement's sting, and it is trivially demonstrable:

```java
CompletableFuture<String> done = CompletableFuture.completedFuture("x");
System.out.println(Thread.currentThread().getName());          // main
done.thenApply(s -> { System.out.println(Thread.currentThread().getName()); return s; });
                                                               // ALSO main
```

### `defaultExecutor()` — what `*Async` with no executor really picks

```java
// java.util.concurrent.CompletableFuture
private static final boolean USE_COMMON_POOL =
        (ForkJoinPool.getCommonPoolParallelism() > 1);

private static final Executor ASYNC_POOL = USE_COMMON_POOL
        ? ForkJoinPool.commonPool()
        : new ThreadPerTaskExecutor();

public Executor defaultExecutor() { return ASYNC_POOL; }
```

Two facts fall out, and both matter. **On a machine or container where
`availableProcessors()` returns 1**, common-pool parallelism is `0`, so
`USE_COMMON_POOL` is false and the JDK silently uses a **thread-per-task** executor:
one platform thread per `*Async` call. Under load that is Topic 98's thread leak with a
different cause, and it is a real container hazard — Topic 82's `--cpus=0.5` produces
it. And **`defaultExecutor()` is overridable**: subclass `CompletableFuture`, override
`defaultExecutor()` and `newIncompleteFuture()`, and every `*Async` with no executor
uses your pool. Legitimate for a library that must not touch the common pool, invisible
to the reader, so document it loudly.

### The common pool, precisely (this is Topic 25 and Topic 100, restated once)

```
parallelism  = Runtime.getRuntime().availableProcessors() - 1
threads      = named "ForkJoinPool.commonPool-worker-N"
lifetime     = JVM-wide, never shut down, daemon threads
designed for = short, CPU-bound, non-blocking, recursively-splitting tasks
```

On a 4-vCPU container that is **three** worker threads for the entire JVM. Three
blocking JDBC calls occupy all of them. Everything else that touches the common pool —
every `.parallelStream()`, every `Arrays.parallelSort`, every library that used
`supplyAsync` without an executor — now waits behind your database.

Overridable at startup, and worth knowing for a controlled experiment:

```bash
-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
```

Raising it is **not** the fix. It papers over the design error and hands you a bigger
shared resource to exhaust.

### `join()` from inside a ForkJoinPool worker: compensation

This is the detail that separates "I read the docs" from "I read the source", and it is
directly relevant to the drill.

`CompletableFuture.join()` on an incomplete future calls `waitingGet`, which builds a
`Signaller` — and `Signaller implements ForkJoinPool.ManagedBlocker`. If the calling
thread is a `ForkJoinWorkerThread`, `join()` goes through `ForkJoinPool.managedBlock`,
which tells the pool "I am about to block; if you need parallelism, **start a
compensation thread**".

So the pool can temporarily exceed its parallelism target to cover a blocked worker.
Two things follow:

- **`join()` inside a common-pool task is survivable**, because the pool is told about
  it and compensates.
- **A raw blocking call — `DriverManager.getConnection`, `Socket.read`,
  `Thread.sleep`, `HikariDataSource.getConnection` — is NOT.** The pool has no idea.
  No compensation thread is created. That worker is simply gone from the stealing set
  until the call returns.

**That asymmetry is the whole reason the drill uses JDBC and not `join()`.** Say it in
an interview and you will be the only candidate who does.

### What the failure looks like in a thread dump

*Illustration of the format, not captured output. `<n>`, `<tid>` and `0x...` are
placeholders; your own dump will have real values.*

A common-pool worker doing blocking JDBC — the shape you are hunting:

```
"ForkJoinPool.commonPool-worker-<n>" #<tid> daemon prio=5 os_prio=31 tid=0x... nid=0x... runnable
   java.lang.Thread.State: RUNNABLE
        at java.base/sun.nio.ch.Net.poll(Native Method)
        at java.base/sun.nio.ch.NioSocketImpl.park(...)
        at org.postgresql.core.PGStream.receiveChar(...)
        at org.postgresql.jdbc.PgStatement.executeQuery(...)
        at com.orderflow.orders.OrderDetailService.lambda$loadLines$1(OrderDetailService.java:<line>)
        at java.base/java.util.concurrent.CompletableFuture$AsyncSupply.run(...)
        at java.base/java.util.concurrent.ForkJoinTask.doExec(...)
```

Three things to read off it, in order. **The thread name begins
`ForkJoinPool.commonPool-worker-`** — that is the finding; application work has no
business there. **The state is `RUNNABLE`** and it is using **zero CPU**, because a
socket read is `RUNNABLE` — the JVM cannot see inside a native call (Topic 94's table),
so always corroborate with `top -H -p <pid>`. And **the frame
`CompletableFuture$AsyncSupply.run`** tells you it arrived via `supplyAsync`; `AsyncRun`
means `runAsync`, and `UniApplyAsync` and friends mean a `*Async` continuation.

And a thread parked in `join()`:

```
"http-nio-8080-exec-<n>" #<tid> daemon prio=5 tid=0x... nid=0x... waiting on condition
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@<version>/Native Method)
        at java.base/java.util.concurrent.CompletableFuture$Signaller.block(...)
        at java.base/java.util.concurrent.ForkJoinPool.unmanagedBlock(...)
        at java.base/java.util.concurrent.CompletableFuture.waitingGet(...)
        at java.base/java.util.concurrent.CompletableFuture.join(...)
        at com.orderflow.orders.OrderDetailController.get(OrderDetailController.java:<line>)
```

`WAITING (parking)` at `CompletableFuture$Signaller.block` is the signature of a
`join()` that will never return. `unmanagedBlock` in that stack means the caller was
**not** a FJ worker; `managedBlock` means it was. That one word tells you whether
compensation is in play.

### Thread states, for this topic specifically

Topic 94 gives the full table; the Quick reference card at the end of this document has
the three rows you need here. The practical rule they add up to: **`join()` in production
code without an `orTimeout` upstream of it is an unbounded wait you have chosen.** Topic
94 made the same point about `lock()` versus `tryLock(timeout)`; it is the same idea in
a different API. `TIMED_WAITING` is a promise that the thread will return; `WAITING` is
not.

---

## Concurrency trace

Two traces, both **before** any correct code. The first is the assigned drill. The
second is the exception that vanishes.

### Trace 1 — blocking JDBC on the common pool starves unrelated work

**The setup.** `orderflow` runs in a container with `--cpus=4`, so
`availableProcessors()` is 4 and the common pool has **3** workers. Two entirely
separate pieces of code use it:

- `OrderDetailService.load()` was "made faster" last sprint with
  `CompletableFuture.supplyAsync(...)` — **no executor argument**. Each call fans out
  three JDBC reads onto the common pool.
- `CatalogueIndexJob` runs every five minutes and rebuilds the search index with
  `products.parallelStream().map(this::toIndexDocument)...` — pure CPU, no I/O,
  textbook-correct use of a parallel stream (Topic 25).

Neither author has read the other's file. Neither file mentions a thread pool.

| Step | Thread A — `http-nio-8080-exec-14` (order detail, order 8812) | Thread B — `scheduling-1` (catalogue re-index, 100k products) | Common pool state / outcome |
|---|---|---|---|
| 1 | `supplyAsync(loadLines)` — no executor | — | task queued; `commonPool-worker-1` picks it up |
| 2 | `supplyAsync(loadProducts)` — no executor | — | `commonPool-worker-2` picks it up |
| 3 | `supplyAsync(loadPaymentStatus)` — no executor | — | `commonPool-worker-3` picks it up. **All 3 workers now busy.** |
| 4 | `allOf(...).join()` → parks | — | A is `WAITING (parking)` at `Signaller.block` |
| 5 | worker-1 calls `dataSource.getConnection()`; pool of 20 is busy; **blocks** | — | worker-1: `RUNNABLE`, 0% CPU, inside `HikariPool.getConnection` |
| 6 | worker-2 blocks in `PgStatement.executeQuery` — a 31 ms read against 5M order lines | — | worker-2: `RUNNABLE`, 0% CPU |
| 7 | worker-3 blocks the same way | `products.parallelStream()...` submits its root task to the **common pool** | worker-3: `RUNNABLE`, 0% CPU. **No worker is free to steal B's task.** |
| 8 | still blocked | `ForkJoinTask.invoke` → the calling thread `scheduling-1` helps by running the root task itself, but the split subtasks sit in the queue with no worker to steal them | Index job is now effectively **single-threaded** |
| 9 | 40 more requests arrive; each queues 3 more blocking tasks | still crawling | Common pool queue depth climbs into the hundreds |
| 10 | request 14 finally completes after 900 ms instead of 31 ms | 5-minute index job is still at 3% after 5 minutes | **Every parallel stream in the JVM is now serialised behind JDBC** |
| 11 | Tomcat threads accumulate in `join()`; the 200-thread pool fills | next scheduled index run fires and **overlaps** the previous one | Two index jobs now compete for zero workers |
| 12 | `GET /products` — which uses **no** `CompletableFuture` and **no** parallel stream — starts timing out because there are no Tomcat threads left | — | **Service-wide failure** |

**Outcome, in business terms.**

The order-detail endpoint that was "optimised" from 88 ms to a hoped-for 35 ms is now
serving at 900 ms and climbing. The search index is **stale by hours**, so customers
searching for a product that went on sale this morning are told it does not exist — lost
revenue with no error anywhere to attribute it to. Within minutes, `GET /products`, an
endpoint sharing no code, no pool and no data with either changed file, begins returning
503, because every Tomcat request thread is parked in `join()` waiting on a common pool
that is waiting on a database.

The p99 dashboard for order-detail does **not** show 900 ms. It shows fewer and fewer
data points, because requests that time out at the load balancer never record a latency.
**The graph gets quieter as the service gets worse.** And nobody looks at
`CatalogueIndexJob`, because nobody changed it.

### Trace 2 — the future nobody joins, and the exception that never existed

**The setup.** After a wallet debit commits, `orderflow` fires a payment-confirmation
notification asynchronously so the HTTP response is not delayed by an SMTP round trip.

```java
walletService.debit(userId, total);
CompletableFuture.runAsync(() -> notifier.sendConfirmation(orderId), notifyExecutor);
return OrderResponse.accepted(orderId);          // <-- returns immediately. Nothing joins.
```

| Step | Thread A — `http-nio-8080-exec-22` (order intake, order 9917) | Thread B — `orderflow-notify-2` (the async task) | Shared state / outcome |
|---|---|---|---|
| 1 | debits wallet £64.00, commits the transaction | — | balance −64.00, **durable** |
| 2 | `runAsync(...)` → returns a `CompletableFuture<Void>` that is dropped on the floor | — | future has **no** dependents, **no** handler |
| 3 | returns HTTP 202 to the customer | picks up the task | customer sees "order accepted" |
| 4 | serves the next request | `notifier.sendConfirmation` throws `MailAuthenticationException` — the SMTP credential rotated last night | — |
| 5 | — | `AsyncRun.run` catches the throwable and calls `completeExceptionally(ex)` | future's `result` = `AltResult(CompletionException(MailAuthenticationException))` |
| 6 | — | `postComplete()` walks the `stack` field. **`stack` is null.** Nothing to notify. | The `Throwable` is now referenced only by the future |
| 7 | — | task ends normally; the worker returns to the pool | Thread B logs **nothing**. It did not fail. |
| 8 | — | — | The future becomes unreachable and is collected. **The exception is garbage-collected with it.** |
| 9 | — | — | Zero log lines. Zero metrics. `notifier` has no error counter because it never saw a caller. |

**Outcome, in business terms.**

Every order placed after the credential rotation is charged correctly and confirmed to
nobody. Support tickets arrive tagged "did my order go through?", and the engineer who
checks sees a healthy order row, a healthy wallet ledger, HTTP 202 in the access log,
and **no error of any kind** in eleven hours of application logs. The bug is found four
days later by a reconciliation query — `payments` rows with no matching `notifications`
row — exactly the shape of evidence Topic 09 warned about: **a defect whose only symptom
is a missing side effect.**

In Node this would have been a red `UnhandledPromiseRejection` within milliseconds, and
on modern Node it would have crashed the process. Java gives you nothing. **You attach
the handler yourself, every time.**

---

## Example 1 — minimal

The smallest honest fan-out, and the one-word variations that break it.

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;

/** Two independent lookups, combined. Deliberately tiny. */
public final class StockAndPrice {

    private final ExecutorService io;        // injected, bounded, named -- Topic 90
    private final ProductRepository products;
    private final InventoryRepository inventory;

    public StockAndPrice(ExecutorService io, ProductRepository p, InventoryRepository i) {
        this.io = io; this.products = p; this.inventory = i;
    }

    public CompletableFuture<ProductView> load(String sku) {

        CompletableFuture<Product> product =
                CompletableFuture.supplyAsync(() -> products.bySku(sku), io);   // executor: NAMED

        CompletableFuture<Integer> stock =
                CompletableFuture.supplyAsync(() -> inventory.availableFor(sku), io);

        return product
                .thenCombine(stock, ProductView::new)      // runs when BOTH are done
                .orTimeout(2, java.util.concurrent.TimeUnit.SECONDS)
                .exceptionally(ex -> ProductView.unavailable(sku, rootCause(ex)));
    }

    /** The unwrapping helper. Write it once; use it everywhere. */
    static Throwable rootCause(Throwable t) {
        while ((t instanceof java.util.concurrent.CompletionException
             || t instanceof java.util.concurrent.ExecutionException)
                && t.getCause() != null) {
            t = t.getCause();
        }
        return t;
    }
}
```

Five things in those twenty lines are load-bearing.

**Every `supplyAsync` names an executor.** Delete `, io` from either call and that lookup
moves to the common pool. It still compiles, tests still pass, and you have shipped
Trace 1.

**`thenCombine`, not two `join()`s.** `thenCombine` attaches a continuation and blocks
nothing. `new ProductView(product.join(), stock.join())` parks the calling thread twice,
and the second `join` cannot begin until the first returns — sequencing, reintroduced at
the point of combination.

**The method returns a future; it does not join.** Push `join()` as far up the stack as
you can, ideally to the controller where the thread was already dedicated to this
request. Every library-level `join()` is a thread taken from a caller who did not
consent.

**`orTimeout` before `exceptionally`.** The timeout must be upstream so its
`TimeoutException` flows into the handler. Swap them and a timeout becomes an unhandled
exceptional completion — Trace 2.

**`rootCause` exists.** `exceptionally` receives `CompletionException(TimeoutException)`.
Every `instanceof` you write must be against the cause.

### The same code, four ways to break it

```java
CompletableFuture.supplyAsync(() -> products.bySku(sku));       // 1: no executor -> Trace 1

CompletableFuture<CompletableFuture<Price>> wrong =             // 2: thenApply where
        product.thenApply(p -> priceService.quote(p));          //    thenCompose is meant.
CompletableFuture<Price> right =                                //    Compiles fine.
        product.thenCompose(p -> priceService.quote(p));

future.exceptionally(e -> {                                     // 3: instanceof on the
    if (e instanceof ProductNotFoundException) return miss();   //    wrapper: never taken
    throw new CompletionException(e);
});

CompletableFuture.runAsync(() -> auditLog.record(sku), io);     // 4: dropped -> Trace 2
```

---

## Example 2 — production scenario (on the project spine)

### The constraints

From the Topic 65 baseline, the numbers you are actually working against:

- 100,000 products, 1,000,000 orders, 5,000,000 order lines.
- k6 open-model arrival, 400 rps steady: 70% catalogue read, 20% order read, 10% order
  placement. So `GET /orders/{id}` runs at roughly **80 rps**.
- Container: `--cpus=4`, `--memory=2g`. Therefore `availableProcessors() == 4` and the
  common pool has **3** workers.
- Tomcat request threads: 200.
- **HikariCP maximum pool size: 20.** This is the binding constraint and it is easy to
  forget.
- Recorded baseline for `GET /orders/{id}`: p50 88 ms, p95 141 ms, p99 190 ms, committed
  in `/docs/java/baselines/`.

The endpoint does four reads. Read 1 must happen first (it yields the order and its line
SKUs). Reads 2, 3 and 4 are independent of each other.

```
loadOrderHeader(id)            18 ms   <- must be first
   |
   +-- loadLinesWithProducts   31 ms   \
   +-- loadInventoryPositions  24 ms    >  independent of each other
   +-- loadPaymentStatus       15 ms   /
```

Serial total: 88 ms, which matches the recorded p50 exactly. Ideal parallel total:
18 + 31 = 49 ms plus overhead.

### The code that ships and takes down the service

```java
package com.orderflow.orders;

import java.util.concurrent.CompletableFuture;

@Service
public class OrderDetailService {

    private final OrderRepository orders;
    private final ProductRepository products;
    private final InventoryRepository inventory;
    private final PaymentRepository payments;

    // constructor injection -- Topic 39

    @Transactional(readOnly = true)                       // <-- and this is a second bug
    public OrderDetail load(long orderId) {

        OrderHeader header = orders.header(orderId);

        CompletableFuture<List<LineView>> lines =
                CompletableFuture.supplyAsync(() -> products.linesFor(orderId));     // no executor

        CompletableFuture<Map<String, Integer>> stock =
                CompletableFuture.supplyAsync(() -> inventory.positionsFor(orderId)); // no executor

        CompletableFuture<PaymentStatus> payment =
                CompletableFuture.supplyAsync(() -> payments.statusFor(orderId));     // no executor

        CompletableFuture.allOf(lines, stock, payment).join();                        // blocks here

        return new OrderDetail(header, lines.join(), stock.join(), payment.join());
    }
}
```

It reviews well. It is shorter than the serial version. The p50 on a developer laptop
with a warm cache and no load is genuinely better. **There are five defects in it and
four of them are invisible without load.**

### The five defects, named

**Defect 1 — three `supplyAsync` calls with no executor.** Trace 1, exactly. Three
blocking JDBC calls per request onto a three-worker JVM-wide pool.

**Defect 2 — `@Transactional(readOnly = true)` wrapping a fan-out.** Topic 55: a
transaction pins a connection for its whole lifetime and is bound to the **calling
thread** via a `ThreadLocal`. The three async tasks run on *different* threads, so they
are **outside** the transaction entirely. You now hold four connections — one pinned by
the transaction on a request thread doing nothing, three taken independently — and the
`readOnly` hint applies to none of the three. Any `LazyInitializationException` this
annotation was supposed to prevent (Topic 49) now fires inside a pool worker where the
stack trace is useless.

**Defect 3 — connection amplification.** Serial code holds 1 connection per in-flight
request; this holds up to 4. At 80 rps and ~90 ms service time, Little's Law puts ~7
requests in flight, times 4 connections is 28, against a pool of 20. **A comfortable
pool is now an exhausted one.** The failure shape is Topic 109's — `HikariPool-1 -
Connection is not available, request timed out` — on endpoints unrelated to this one.

**Defect 4 — `join()` with no timeout.** If the database hangs, the request thread parks
at `Signaller.block` forever. Tomcat has 200 of those to lose.

**Defect 5 — no exception path.** If `payments.statusFor` throws, `allOf(...).join()`
throws. Fine — but the *other two* futures keep running, keep holding connections, and
nothing cancels them (Topic 102). And any branch whose handler you never attached
carries an undelivered exception.

### The fix, defect by defect

#### Step 1 — a dedicated, bounded, named executor sized from Little's Law

Topic 90's method, applied. The tasks are I/O-bound with a service time of roughly
25 ms. The **real** constraint is not CPU; it is that every task takes a HikariCP
connection, and the pool has 20.

```
Target: 80 rps of order-detail requests, 3 parallel DB tasks each = 240 tasks/sec.
Task service time ~25 ms => in-flight tasks = 240 x 0.025 = 6.
Add headroom for bursts: 12 threads.
Hard ceiling: this pool must NEVER want more connections than Hikari can give,
              while leaving room for the other 320 rps of traffic.
              12 threads is 60% of a 20-connection pool. That is already aggressive.
```

The honest conclusion of that arithmetic, which you should reach before writing any
code: **the connection pool, not the thread pool, is the real limit.** Either raise
Hikari to match (and check the database's `max_connections`), or accept that fan-out
buys you less than the latency arithmetic suggests. Write the number down before you
choose.

```java
@Bean(destroyMethod = "shutdown")               // Topic 90: shutdown is not optional
public ExecutorService orderDetailExecutor(MeterRegistry registry) {

    ThreadFactory factory = Thread.ofPlatform()
            .name("orderflow-detail-", 0)       // NAMED. Topic 94: pool-1-thread-7 costs an hour.
            .factory();

    ThreadPoolExecutor executor = new ThreadPoolExecutor(
            12, 12,                                   // core == max: no growth games
            0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(48),             // BOUNDED. Topic 90 / 93.
            factory,
            new ThreadPoolExecutor.AbortPolicy());    // deliberate: shed, do not buffer forever

    // Topic 118 forward-reference: queue depth, active, rejected as first-class metrics.
    return ExecutorServiceMetrics.monitor(
            registry, executor, "orderDetail", Tags.of("service", "orderflow"));
}
```

Queue bound of 48 is derived, not guessed: at 12 threads and 25 ms per task, 48 queued
tasks is `48 / 12 * 25ms = 100 ms` of buffered work — inside the p99 budget of 190 ms
with room for the task itself. **The capacity of the queue is the latency you are
willing to add.** That is Topic 93's mechanical statement arriving early.

#### Step 2 — the service, rewritten

```java
package com.orderflow.orders;

import java.util.concurrent.*;
import static com.orderflow.util.Futures.rootCause;

@Service
public class OrderDetailService {

    private static final Duration BUDGET = Duration.ofMillis(250);

    private final ExecutorService detailExecutor;
    private final OrderRepository orders;
    private final ProductRepository products;
    private final InventoryRepository inventory;
    private final PaymentRepository payments;

    /**
     * NOT @Transactional. Each read below opens and closes its own short transaction
     * inside its own repository call, on its own thread, holding a connection only for
     * the duration of that query. Topic 55.
     */
    public OrderDetail load(long orderId) {

        // Step 1 is a genuine dependency: it yields the SKUs the other reads need.
        // It runs on the CALLING thread, which is a Tomcat request thread already
        // dedicated to this request. Do not spend a pool thread on it.
        OrderHeader header = orders.header(orderId);

        CompletableFuture<List<LineView>> lines =
                supply(() -> products.linesFor(orderId), "lines", List.of());

        CompletableFuture<Map<String, Integer>> stock =
                supply(() -> inventory.positionsFor(orderId), "stock", Map.of());

        CompletableFuture<PaymentStatus> payment =
                supply(() -> payments.statusFor(orderId), "payment", PaymentStatus.UNKNOWN);

        // join() here is correct and deliberate: this thread belongs to this request
        // and has nothing else to do. Every future already carries its own timeout and
        // its own fallback, so this join cannot hang and cannot throw.
        CompletableFuture.allOf(lines, stock, payment).join();

        return new OrderDetail(header, lines.join(), stock.join(), payment.join());
    }

    /**
     * One helper. It enforces, for every fan-out branch:
     *   - an explicit executor
     *   - a timeout
     *   - a terminal handler, so no exception can be silently discarded
     *   - a degraded value rather than a failed request
     */
    private <T> CompletableFuture<T> supply(Supplier<T> work, String name, T fallback) {
        return CompletableFuture
                .supplyAsync(work, detailExecutor)
                .orTimeout(BUDGET.toMillis(), TimeUnit.MILLISECONDS)
                .exceptionally(ex -> {
                    Throwable cause = rootCause(ex);
                    meterRegistry.counter("orderflow.orderdetail.branch.failed",
                                          "branch", name,                     // LOW cardinality
                                          "cause", cause.getClass().getSimpleName())
                                 .increment();
                    log.warn("order-detail branch {} degraded: {}", name, cause.toString());
                    return fallback;                       // partial result, not a 500
                });
    }
}
```

Six decisions in that method, each of which you should be able to defend. **The first
read is synchronous, on the request thread** — a true dependency, so making it async
buys nothing and costs a pool thread and a context switch. **`@Transactional` is gone**:
each repository call manages its own connection for one query, so nothing is pinned
across the fan-out. **Every branch has an executor, a timeout and a handler**, and the
`supply` helper makes it impossible to forget — better than a review convention (Topic
94's `OrderedLocks` argument again). **Failure degrades rather than fails**: a missing
payment status renders as "pending" rather than a 500, which is a product decision you
should get agreed rather than assume, but which the code should *express* either way.
**The metric tags are low-cardinality** — `branch` has three values, `cause` a handful;
tagging with `orderId` would create a million series and kill Prometheus (Topic 118).
And **`join()` appears exactly once, at the top of the stack**; everything below returns
futures.

#### Step 3 — the context that does not travel

`SecurityContextHolder` (Topic 56) and the MDC correlation id (Topic 120) are
`ThreadLocal`-backed. Pool threads are not the request thread. **Neither survives the
hop**, and neither fails loudly:

- The security context is simply absent inside the task. A `@PreAuthorize` check there
  fails as "not authenticated", which reads like a bug in the auth layer.
- The MDC is absent, so log lines from the fan-out have no correlation id and cannot be
  joined to the request that caused them.

The fix at this JDK level is explicit capture-and-restore:

```java
/** Capture on the request thread; restore on the pool thread; ALWAYS clear. */
public static Runnable wrap(Runnable task) {
    SecurityContext security = SecurityContextHolder.getContext();
    Map<String, String> mdc = MDC.getCopyOfContextMap();
    return () -> {
        SecurityContextHolder.setContext(security);
        if (mdc != null) MDC.setContextMap(mdc);
        try {
            task.run();
        } finally {
            MDC.clear();                           // Topic 79: a ThreadLocal you do not
            SecurityContextHolder.clearContext();  // clear on a POOLED thread is a leak
        }
    };
}
```

The `finally` is not politeness. A pooled thread lives for the life of the JVM, so a
`ThreadLocal` set on it and never removed is retained forever along with everything the
security context references — Topic 79's leak. It is *also* a security defect: the next
request served by that thread inherits the previous user's context if anything reads it
before Spring overwrites it.

Spring Security's `DelegatingSecurityContextExecutorService` does the security half for
you; keep the MDC half by hand. **Topic 102 replaces all of it with `ScopedValue`**,
which structured forks inherit by construction and which cannot leak because it is
scoped to a lexical block.

#### Step 4 — what to expect from the change, stated before measuring

Write your prediction down before you run the load test. Then compare.

| Metric | Serial baseline | Prediction after the fix | Why |
|---|---|---|---|
| p50 `GET /orders/{id}` | 88 ms | 50–60 ms | 18 ms serial + max(31, 24, 15) + handoff overhead |
| p99 | 190 ms | **may get worse** | fan-out adds queueing at the executor *and* at Hikari; the tail is now the max of three tails, not one |
| HikariCP `pending` | ~0 | **watch this** | up to 3x connection demand per request |
| `executor.queued` (orderDetail) | n/a | should stay near 0 at 80 rps | if it does not, your 12 threads are wrong |
| `executor.rejected` | n/a | 0 at baseline, non-zero under burst | non-zero is **correct** — it is backpressure |

**The p99 row is the one to take seriously.** Fan-out improves the median and often
degrades the tail, because the request now waits for the slowest of three dependencies
rather than the sum of three medians. If your SLO is written on p99 (Topic 130), this
"optimisation" can fail it while improving the average. Measure before you claim a win.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — blocking JDBC on `ForkJoinPool.commonPool()`

**Wrong:**

```java
CompletableFuture.supplyAsync(() -> orderRepository.linesFor(orderId));   // no executor
```

**Exact symptom:** an unrelated subsystem slows down. `jcmd <pid> Thread.print` shows
threads named `ForkJoinPool.commonPool-worker-N` in state `RUNNABLE` with
`PgStatement.executeQuery` or `HikariPool.getConnection` in the stack, while
`top -H -p <pid>` shows them at **0% CPU**. Any parallel stream elsewhere in the JVM
takes far longer than it should or appears single-threaded. And the number of such
workers is capped at `availableProcessors() − 1`, so on a 4-vCPU container you see
exactly **three** and no more — itself a diagnostic fingerprint.

**Root cause:** `supplyAsync` with no executor uses `defaultExecutor()` —
`ForkJoinPool.commonPool()`: JVM-wide, `cores − 1`, designed for short non-blocking CPU
work. A blocking call parks a worker with **no compensation**, because unlike
`CompletableFuture.join()` a JDBC call does not go through `ForkJoinPool.managedBlock`,
so the pool is never told and the worker leaves the stealing set for the whole I/O.

**Fix:** pass an executor. Every time. Then make it impossible to forget, with ArchUnit
rather than review:

```java
@ArchTest
static final ArchRule no_default_executor_async =
    noClasses().should().callMethod(CompletableFuture.class, "supplyAsync", Supplier.class)
      .because("supplyAsync(Supplier) uses the common pool; pass an explicit executor");
```

Add the same rule for `runAsync(Runnable)`, `thenApplyAsync(Function)`,
`thenComposeAsync(Function)`, `thenAcceptAsync(Consumer)`, `thenRunAsync(Runnable)`,
`thenCombineAsync(...)`, `handleAsync(BiFunction)` and `whenCompleteAsync(BiConsumer)`.
All of them: the single-argument overload of every `*Async` method is the same bug.

**Why raising the parallelism is not the fix:**
`-Djava.util.concurrent.ForkJoinPool.common.parallelism=32` makes the symptom rarer
without changing anything structural — still unrelated subsystems sharing one resource,
still no metrics, still no rejection policy, and now much harder to reproduce.

---

### Trap 2 — a future nobody joins swallows its exception

**Wrong:**

```java
CompletableFuture.runAsync(() -> notifier.sendConfirmation(orderId), notifyExecutor);
```

**Exact symptom:** **nothing.** That is the symptom, and it is why this trap deserves
five minutes. The side effect does not happen. There is no exception in any log, at any
level, from any logger. No metric increments — the task's own error counter never fires,
because the task never got that far. `executor.completed` in Micrometer **does**
increment, because from the executor's point of view the task ran to completion: it
threw *into a future*, not out of the `Runnable`. The only evidence is a reconciliation
query finding a missing row, days later.

**Root cause:** `AsyncRun.run` catches the `Throwable` and calls
`completeExceptionally`. The future stores it in `result` as an `AltResult`, then calls
`postComplete()` to notify dependents. The `stack` field is `null` — there are no
dependents. Nothing consumes the throwable. When the future becomes unreachable, the
exception is collected with it. **The JDK has no unhandled-completion hook and no
warning.** This is a deliberate design choice and it is the opposite of Node's.

**Fix — three levels, and you should apply all three.** First, a terminal handler on
every future you do not join; `whenComplete` is the right verb because it passes value
and throwable through unchanged:

```java
CompletableFuture.runAsync(() -> notifier.sendConfirmation(orderId), notifyExecutor)
    .whenComplete((ignored, ex) -> {
        if (ex != null) {
            Throwable cause = rootCause(ex);
            log.error("confirmation failed for order {}", orderId, cause);
            registry.counter("orderflow.notification.failed",
                             "cause", cause.getClass().getSimpleName()).increment();
        }
    });
```

Second, make it structural: one `fireAndForget(Runnable, String name)` helper that
attaches the handler, plus an ArchUnit rule forbidding bare `runAsync`. A convention
every author must remember is not a control. Third, reconcile — any fire-and-forget side
effect the business depends on needs an independent check that it happened. That is
Topic 115's outbox argument in miniature: record the intent durably and have a worker
drive it to completion, rather than firing and hoping.

**The interview line:** "Java will silently discard an exception from a future nobody
observes. Node warns and, since 15, exits. So in Java the handler is not optional
hygiene, it is the only thing standing between you and a defect whose sole symptom is a
missing row."

---

### Trap 3 — `instanceof` on the wrapper instead of the cause

**Wrong:**

```java
placeOrder(cmd).exceptionally(ex -> {
    if (ex instanceof InsufficientFundsException) {
        return OrderResult.declined("insufficient funds");     // <-- unreachable
    }
    if (ex instanceof OutOfStockException) {
        return OrderResult.declined("out of stock");           // <-- unreachable
    }
    throw new CompletionException(ex);
});
```

**Exact symptom:** every domain failure becomes a **500**, never a 4xx. The customer
sees "something went wrong" for an insufficient-balance decline. Your
`@ControllerAdvice` (Topic 46) never maps it, because what reaches it is a
`CompletionException`, not the domain type it has a handler for — so the `ProblemDetail`
response carries `type: about:blank` and a generic title. Unit tests miss it because
they usually call the synchronous path, where the exception has not travelled through a
future.

**Root cause:** exceptions propagating through a `CompletionStage` are wrapped.
`exceptionally`, `handle` and `whenComplete` all receive the wrapper — and the wrapping
is inconsistent enough that you must not reason about when it happens.
`completeExceptionally(x)` observed directly may hand you `x` unwrapped; the same `x`
after a `thenApply` is wrapped.

**Fix:** one unwrapping helper, used everywhere, plus a rule that no `instanceof` in a
future handler is written against the raw parameter.

```java
public static Throwable rootCause(Throwable t) {
    while ((t instanceof CompletionException || t instanceof ExecutionException)
            && t.getCause() != null && t.getCause() != t) {     // self-cause is legal
        t = t.getCause();
    }
    return t;
}
```

**In an interview**, say the second half too: "and when I rethrow from a handler I wrap
it in `CompletionException` deliberately, because a checked exception is not allowed
there and an unwrapped runtime exception gets re-wrapped anyway."

---

### Trap 4 — `thenApply` on an already-complete future runs on the caller

**Wrong:**

```java
@GetMapping("/orders/{id}")
public OrderDetail get(@PathVariable long id) {
    return cache.lookup(id)                          // returns a CompletableFuture
                .thenApply(this::enrich)             // "this runs in the background"
                .thenApply(this::redactForRole)
                .thenApply(this::toDetail)
                .join();
}
```

The author's mental model: "the three transforms run off the request thread; the request
thread only waits at `join()`."

**Exact symptom:** on a **cache hit**, all three transforms run on the Tomcat request
thread, synchronously, before `join()` is even reached — and the endpoint's CPU time
shows up entirely in `http-nio-8080-exec-*` threads in a flame graph (Topic 78). On a
**cache miss**, they run on whatever thread completed the future — possibly a JDBC
callback thread, possibly a common-pool worker. **The same three lines execute on two
different pools depending on cache state**, so a flame graph taken during a warm period
and one taken during a cold period disagree about where your CPU goes, and neither is
wrong.

Worse: if `redactForRole` reads `SecurityContextHolder`, it works on the cache-hit path
(request thread has the context) and returns "not authenticated" on the cache-miss path
(pool thread does not). **A security check that passes or fails based on cache warmth.**

**Root cause:** the mechanical statement. `thenApply` has no executor: if `result` is
already non-null when you attach, `tryFire` runs the function inline on the calling
thread; otherwise on the completing thread.

**Fix:** decide explicitly per continuation. Cheap and pure? `thenApply` is correct and
fastest — no handoff, no queue, no context switch — but say so in a comment. Expensive,
blocking or context-sensitive? `thenApplyAsync(fn, executor)`, never the one-arg form.

The rule to internalise: **`thenApply` is not "background". It is "wherever we happen to
be".** If you cannot name the thread, use the two-argument `*Async` form.

---

### Trap 5 — `allOf(...).join()` with no timeout and no cancellation

**Wrong:**

```java
CompletableFuture.allOf(lines, stock, payment).join();
```

**Exact symptom:** two separate failures, both under load.

*Symptom A — the hang.* One dependency stops responding. The request thread parks at
`CompletableFuture$Signaller.block` in `WAITING (parking)` **with no timeout**. Tomcat's
200 threads fill within seconds, `GET /products` starts returning 503, and
`jcmd Thread.print` shows 200 identical stacks — Topic 94's visual signature.

*Symptom B — the leak.* `payment` fails fast at 5 ms, `allOf` completes exceptionally,
`join()` throws, the controller returns 500. Meanwhile **`lines` and `stock` are still
running**, still holding HikariCP connections, for the full duration of their queries.
Nothing cancels them. Under a failure storm you are running work nobody will read, at
exactly the moment you have least capacity: `hikaricp.connections.usage` stays high
while throughput collapses.

**Root cause:** `allOf` composes completion and has **no cancellation semantics
whatsoever**. It does not cancel siblings on failure, does not propagate cancellation
downward, and `cancel(true)` does **not** interrupt the running thread despite the
parameter name — `CompletableFuture` ignores `mayInterruptIfRunning` and merely
completes the future with `CancellationException`. The task keeps running.

**Fix — three parts.** Every branch carries its own `orTimeout`, so no branch can hang
the join. Every branch carries its own terminal handler, so `allOf` never completes
exceptionally and `join()` cannot throw — you trade "all or nothing" for "partial
result", deliberately. And for *real* cancellation, use Topic 102's
`StructuredTaskScope`: a scope with a shutdown-on-failure policy cancels siblings when
one fork fails and cannot be exited while a fork is still running. That is a
language-level guarantee `allOf` cannot give you at any amount of care.

**Say this in an interview:** "`allOf` waits, it does not supervise. If one branch fails
the others keep running and keep holding resources. That gap is precisely why structured
concurrency exists, and it is the reason I would reach for `StructuredTaskScope` on
JDK 21+ for new fan-out code."

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM and will not print
output and call it real. What I can give you precisely is what to look for and what each
possible result means.

### Setup

```bash
mkdir -p ~/java-lab/91 && cd ~/java-lab/91
java --version                # expect 21 or 25
jcmd -l                       # you will use this constantly
```

Keep two terminals open: one runs the program, one runs `jcmd`.

### Proof 1 — how many common-pool workers does this machine actually have?

```java
public class PoolFacts {
    public static void main(String[] args) {
        System.out.println("availableProcessors   = " + Runtime.getRuntime().availableProcessors());
        System.out.println("commonPool parallelism= " + java.util.concurrent.ForkJoinPool.getCommonPoolParallelism());
        System.out.println("commonPool            = " + java.util.concurrent.ForkJoinPool.commonPool());
        System.out.println("CF defaultExecutor    = " + new java.util.concurrent.CompletableFuture<String>().defaultExecutor());
    }
}
```

```bash
java PoolFacts.java

# Now the container reality from Topic 82:
java -XX:ActiveProcessorCount=4 PoolFacts.java
java -XX:ActiveProcessorCount=1 PoolFacts.java
docker run --rm --cpus=0.5 -v "$PWD":/app -w /app eclipse-temurin:21 java PoolFacts.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `commonPool parallelism` = `availableProcessors - 1` | The normal case. On 4 vCPUs, **3** workers for the whole JVM. |
| `commonPool parallelism = 0` and `defaultExecutor` is **not** a `ForkJoinPool` | A 1-CPU machine or container. `*Async` with no executor now creates **one thread per task** — confirm the class name in the output. |
| `availableProcessors` = host cores despite `--cpus=0.5` | Your JVM is not container-aware (Topic 82). Fix the flags before concluding anything. |

The second row is the one to remember. **On a small container, `supplyAsync` without an
executor stops being "shared pool starvation" and becomes "unbounded thread creation".**
Two different failure modes from one line of code, decided by a number you did not set.

### Proof 2 — which thread runs the continuation

```java
import java.util.concurrent.*;

public class WhichThread {
    static String show(String label) {
        System.out.printf("%-34s %s%n", label, Thread.currentThread().getName());
        return "x";
    }
    static CompletableFuture<String> done() { return CompletableFuture.completedFuture("x"); }

    public static void main(String[] args) throws Exception {
        ExecutorService mine = Executors.newFixedThreadPool(2, r -> new Thread(r, "mine-pool"));
        show("main");

        done().thenApply(s -> show("A thenApply (already done)")).join();

        CompletableFuture<String> b = new CompletableFuture<>();     // B: completed later
        b.thenApply(s -> show("B thenApply (completed later)"));
        new Thread(() -> b.complete("x"), "completer").start();
        Thread.sleep(200);

        done().thenApplyAsync(s -> show("C thenApplyAsync (no executor)")).join();
        done().thenApplyAsync(s -> show("D thenApplyAsync (mine)"), mine).join();
        CompletableFuture.supplyAsync(() -> show("E supplyAsync (no executor)")).join();

        mine.shutdown();
    }
}
```

```bash
java WhichThread.java
```

**What to look for:**

| Line | Expected thread | What it proves |
|---|---|---|
| A | `main` | **The mechanical statement.** An already-complete future runs a non-async continuation on the *calling* thread, synchronously. |
| B | `completer` | A non-async continuation runs on whichever thread completes the future. |
| C | `ForkJoinPool.commonPool-worker-1` | `*Async` with no executor means the common pool. |
| D | `mine-pool` | The only form whose threading you controlled. |
| E | `ForkJoinPool.commonPool-worker-1` | Same for the creation methods. |

Run it twice and note that A and B do not change, but which *numbered* common-pool
worker serves C and E may. **Print this table and keep it.** Four lines of output settle
an argument that otherwise takes a whole code review.

### Proof 3 — the exception that disappears

```java
import java.util.concurrent.*;

public class SwallowedException {
    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(1, r -> new Thread(r, "worker"));

        // 1. Dropped. Nothing will ever be printed about this failure.
        CompletableFuture.runAsync(() -> { throw new IllegalStateException("DROPPED"); }, pool);

        // 2. Handled. This one is visible.
        CompletableFuture.runAsync(() -> { throw new IllegalStateException("HANDLED"); }, pool)
                .whenComplete((v, ex) -> System.out.println("whenComplete saw: " + ex));

        Thread.sleep(500); System.gc(); Thread.sleep(500);
        System.out.println("--- end of main; note what was NEVER printed ---");
        pool.shutdown();
    }
}
```

```bash
java SwallowedException.java
java -Xlog:exceptions=info SwallowedException.java     # even this does not surface it as an error
```

**What to look for:**

| What you see | What it means |
|---|---|
| `whenComplete saw: java.util.concurrent.CompletionException: java.lang.IllegalStateException: HANDLED` | Confirms the **wrapping**. Note it is `CompletionException`, not the raw exception. |
| **No mention of `DROPPED` anywhere** | Trace 2, demonstrated in eight lines. There is no flag, no logger and no JVM option that surfaces it. |
| With `-Xlog:exceptions`, `DROPPED` appears as a *thrown* event but never as an error | The JVM logs that an exception object was constructed. It has no concept of "nobody handled it." Do not mistake this for a safety net. |

Now add a `Thread.setDefaultUncaughtExceptionHandler` and confirm it does **not** fire
either — because the exception never escaped a thread; it was captured into a future.
That negative result is the memorable part.

### Proof 4 — common-pool starvation, in one file

```java
import java.util.concurrent.*;
import java.util.stream.IntStream;

public class CommonPoolStarvation {
    public static void main(String[] args) throws Exception {
        System.out.println("pid = " + ProcessHandle.current().pid());
        int workers = ForkJoinPool.getCommonPoolParallelism();
        System.out.println("parallelism = " + workers);

        for (int i = 0; i < workers; i++) {              // occupy every worker
            CompletableFuture.supplyAsync(() -> {        // NO executor -- the bug
                try { Thread.sleep(60_000); } catch (InterruptedException e) { }
                return 1;
            });
        }
        Thread.sleep(500);                               // let them all be picked up

        long start = System.currentTimeMillis();         // unrelated, pure CPU work
        long sum = IntStream.range(0, 20_000_000).parallel().mapToLong(i -> i % 7).sum();
        System.out.println("parallel sum took " + (System.currentTimeMillis() - start)
                           + " ms, result " + sum);
        Thread.sleep(600_000);
    }
}
```

```bash
java CommonPoolStarvation.java
# in the other terminal:
jcmd <pid> Thread.print | grep -A6 'commonPool-worker'
top -H -p <pid>
```

Then re-run with the fix — a dedicated executor for the sleeping tasks — and compare the
elapsed time of the parallel sum.

**What to look for:**

| What you see | What it means |
|---|---|
| All `commonPool-worker-*` in `TIMED_WAITING (sleeping)`, and the parallel sum far slower than a control run | **Starvation reproduced.** The sum had no workers to steal from. |
| `top -H` showing those workers at ~0% CPU while wall-clock time elapses | The distinction that matters: occupying slots, not consuming CPU. |
| The parallel sum still finishing fast | Parallelism high enough that the sleepers did not exhaust it, **or** the calling thread did the work itself (`ForkJoinTask.invoke` lets the submitter help). Retry with `-XX:ActiveProcessorCount=2`. |
| With a dedicated executor for the sleepers, the sum returns to control time | **The whole fix, proven.** |

Take the third row seriously: the common pool lets the *submitting* thread help, so
starvation is a spectrum rather than a binary. That is exactly why this bug is hard to
reproduce on a 10-core laptop and trivial to hit on a 2-vCPU container.

### Proof 5 — is `join()` compensated? Read the source

```bash
mkdir -p ~/jdk-src && cd ~/jdk-src
unzip -o "$JAVA_HOME/lib/src.zip" 'java.base/java/util/concurrent/CompletableFuture.java'
grep -n "ManagedBlocker\|managedBlock\|unmanagedBlock\|class Signaller" \
     java.base/java/util/concurrent/CompletableFuture.java
```

**What to look for:** `static final class Signaller ... implements ForkJoinPool.ManagedBlocker`,
and in `waitingGet`, a branch calling `ForkJoinPool.managedBlock(q)` when the current
thread is a `ForkJoinWorkerThread` and `unmanagedBlock(q)` otherwise. That one branch is
the entire "join is compensated, raw blocking is not" distinction, and reading it
yourself is worth more than taking my word for it.

### Proof 6 — measure the fan-out, correctly

Do **not** use a `System.nanoTime()` loop; the Measurement section lists five reasons it
lies. For an end-to-end latency question the right instrument is the **k6 load test from
Topic 65 re-run identically**, not a microbenchmark at all.

---

## Failure drill

**Assignment (Topic 91, from the master plan's drill map):** run blocking JDBC on the
common pool under load, observe unrelated parallel work stall, and fix it with a
dedicated bounded executor.

Budget ninety minutes. Part E is the part that teaches; do not stop at Part C.

### Part A — establish the ground truth

1. Bring up the Topic 65 docker-compose stack with the `load` profile.
2. Re-run the recorded k6 baseline; confirm p50/p95/p99 for `GET /orders/{id}` within
   ±10% of `/docs/java/baselines/`. **If it does not reproduce, stop and fix that
   first** — the Phase 7 gate rule keeps everything downstream falsifiable.
3. Record the container's CPU allocation, confirm `availableProcessors()` inside the JVM
   matches it (Topic 82), and note the common-pool parallelism.

### Part B — add a second, innocent consumer of the common pool

You need something unrelated to be damaged, or the drill proves nothing.

```java
@Component
public class CatalogueIndexJob {
    @Scheduled(fixedDelay = 60_000)
    public void reindex() {
        long start = System.nanoTime();
        List<Product> all = products.findAllForIndex();   // ~100k rows, loaded once
        List<IndexDocument> docs = all.parallelStream()   // COMMON POOL. CPU-bound. Correct code.
                .map(this::toIndexDocument).toList();
        index.replaceAll(docs);
        registry.timer("orderflow.catalogue.reindex")
                .record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
    }
}
```

Run the baseline load with this job active and **record the reindex duration**. That
number is your control. It must be stable across several runs before you proceed.

### Part C — break it

Deploy the broken `OrderDetailService` from Example 2 — the one with three
`supplyAsync` calls and no executor. Run the identical k6 script.

### Commands

```bash
# 1. The load, unchanged from Topic 65
k6 run --vus 200 --duration 15m load/orderflow-baseline.js

# 2. Find the JVM
PID=$(jcmd -l | grep -i orderflow | cut -d' ' -f1)

# 3. The single most important command in this drill: what are the common-pool
#    workers doing?
jcmd $PID Thread.print | grep -A12 'ForkJoinPool.commonPool-worker'

# 4. Are they burning CPU, or blocked? (RUNNABLE does not mean running.)
top -H -p $PID
printf '0x%x\n' <tid>     # convert a top thread id to the nid= value in the dump

# 5. Three dumps, ten seconds apart. One gives state; three tell you what is CHANGING.
for i in 1 2 3; do jcmd $PID Thread.print -l > dump-$i.txt; sleep 10; done
grep 'java.lang.Thread.State' dump-1.txt | sort | uniq -c | sort -rn

# 6. Connection pool pressure -- the amplification effect
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending
curl -s localhost:8080/actuator/metrics/hikaricp.connections.acquire

# 7. The victim
curl -s localhost:8080/actuator/metrics/orderflow.catalogue.reindex
```

### What to capture, before reading on

Write these down from your own run. Do not read ahead. (1) How many threads are named
`ForkJoinPool.commonPool-worker-*`, and what is their `Thread.State`? (2) What frame
appears immediately below `CompletableFuture$AsyncSupply.run` in their stacks? (3) What
is their CPU usage in `top -H`, and how do you reconcile that with their reported state?
(4) What happened to `orderflow.catalogue.reindex` — the job you did not touch? (5) What
happened to `hikaricp.connections.pending`? (6) What happened to p50 and p99 for
`GET /orders/{id}`, **separately** — they will not move in the same direction. (7) What
happened to p99 for `GET /products`, which uses none of this code?

### How to read it

| What you see | What it means |
|---|---|
| Exactly `availableProcessors − 1` common-pool workers, all `RUNNABLE`, all in JDBC frames, all at 0% CPU | The bug, fully reproduced. `RUNNABLE` plus zero CPU is the socket-read signature (Topic 94). |
| `orderflow.catalogue.reindex` duration up by a large multiple | **The finding.** An unrelated subsystem degraded because of a change in a file it does not import. |
| `hikaricp.connections.pending` above zero | Connection amplification: 3 concurrent queries per request against a pool of 20. Topic 109's territory. |
| p50 improved, p99 worse | Expected, and the honest result. Fan-out trades the median against the tail. |
| `GET /products` p99 degraded | Blast radius. Tomcat threads parked in `join()` are threads not serving the catalogue. |
| Common-pool workers idle and no starvation visible | Your container has enough CPUs that three blocked workers do not exhaust it. Re-run with `-XX:ActiveProcessorCount=2` to force the issue, and note that **you have just discovered why this bug does not reproduce on a laptop**. |

### Part D — fix it

Deploy the `OrderDetailExecutorConfig` and the rewritten `OrderDetailService` from
Example 2. Re-run the **identical** k6 script.

Then prove the fix rather than assuming it.
`jcmd $PID Thread.print | grep -c 'orderflow-detail-'` should show your 12 threads;
`grep -A8 'commonPool-worker'` should show those **idle**, parked in
`ForkJoinPool.awaitWork`; `executor.queued{name="orderDetail"}` should sit near zero at
baseline; and `orderflow.catalogue.reindex` must return to its control value. **That
last one is the fix, proven** — not the latency of the endpoint you changed.

### Part E — push it until the new pool is the constraint

This is where the drill stops being a demo and starts being an education.

1. Raise k6 to 2x baseline. Watch `executor.queued` climb, then `executor.rejected`
   become non-zero. **A non-zero rejection count is correct behaviour** — backpressure
   reaching the caller instead of an unbounded queue absorbing work (Topic 90's lesson,
   Topic 93's mechanical statement).
2. Now set the queue to `new LinkedBlockingQueue<>()` — unbounded, the default in
   `Executors.newFixedThreadPool`. Re-run at 2x. Watch queue depth and p99 climb
   together while throughput stays flat, and keep going until the heap is in trouble.
   **The queue is your OOM.**
3. Restore the bound and compare the two failure modes in one table: which one told you
   it was failing?

### Part F — the connection-pool interaction

Set Hikari's maximum pool size to 8 and re-run the *fixed* version at baseline: 12
threads now compete for 8 connections. Capture `hikaricp.connections.pending` and
`hikaricp.connections.acquire` (p99), then answer in writing: **where is the queue
now?** Bounding the executor did not remove the queue; it moved it to Hikari, where it
is less visible and where Topic 109 shows it can deadlock outright. That realisation —
bounding one queue relocates pressure rather than removing it — is the most transferable
idea in this drill and Topic 105's whole subject.

### What the fix proves

Write one paragraph on each. **Did the fan-out actually improve the endpoint?** Compare
p50 *and* p99 against the recorded baseline; if p99 got worse, answer "was it worth
shipping" with the SLO, not a preference. **What did the common pool cost, and who
paid?** Name the degraded subsystem and explain in one non-specialist sentence why a
change in `OrderDetailService` slowed down search indexing. **Where is the real
ceiling** — threads, connections or CPU? Support it with a number.

---

## Measurement

### The instrument for each claim

Every claim in this document maps to an instrument that could falsify it.

| Claim | Instrument that makes it falsifiable |
|---|---|
| "Blocking work is running on the common pool" | `jcmd <pid> Thread.print \| grep -A12 commonPool-worker` showing application or JDBC frames |
| "Those threads are blocked, not busy" | `top -H -p <pid>` at ~0% CPU on the matching `nid=` values |
| "The fan-out is queueing" | Micrometer `executor.queued{name="orderDetail"}` and `executor.active` |
| "We are shedding load correctly" | `executor.rejected` counter — **non-zero under overload is correct** |
| "Requests are parked in `join()`" | Dump count of `CompletableFuture$Signaller.block` frames |
| "The connection pool is the real limit" | `hikaricp.connections.pending`, `hikaricp.connections.acquire` p99 |
| "A branch is failing silently" | It is not measurable. **That is the point** — add the `whenComplete` handler and a counter, and it becomes measurable. |
| "The change did not cost p99" | **The recorded Topic 65 p50/p95/p99, re-run identically** |

That last row is the one people skip. A latency fix that costs you tail latency is a
trade you made without measuring.

### Micrometer — what to expose permanently (Topic 118 forward-reference)

```java
ExecutorServiceMetrics.monitor(registry, executor, "orderDetail",
        Tags.of("service", "orderflow"));
```

The five series that matter, and how to alert on each:

| Metric | What it tells you | Alert? |
|---|---|---|
| `executor.queued` (gauge) | Buffered work. **The leading indicator** — it moves minutes before latency does. | Yes, as `queued / capacity > 0.8`, never as an absolute |
| `executor.active` (gauge) | Busy threads. `active == poolSize` sustained means saturated. | As a saturation ratio |
| `executor.pool.size` (gauge) | Thread count. Should be flat at 12. **Rising monotonically is a leak** (Topic 98). | No — use for diagnosis |
| `executor.rejected` (counter) | Backpressure applied. Should be zero at baseline, non-zero under burst. | Yes, on a *sustained* rate |
| `orderflow.orderdetail.branch.failed` (counter, tagged `branch` + `cause`) | Which fan-out branch degraded and why. **This is the metric that makes Trap 2 impossible.** | Yes, on rate per branch |

*Illustration of the exposition FORMAT, not captured output. `<n>` are placeholders.*

```
# TYPE executor_queued_tasks gauge
executor_queued_tasks{name="orderDetail",service="orderflow",} <n>.0
# TYPE executor_rejected_tasks_total counter
executor_rejected_tasks_total{name="orderDetail",service="orderflow",} <n>.0
# TYPE orderflow_orderdetail_branch_failed_total counter
orderflow_orderdetail_branch_failed_total{branch="payment",cause="TimeoutException",} <n>.0
```

**Two standing rules for these series.** First, **never tag with an order id, SKU or
customer id** — 1M orders means 1M time series and a dead Prometheus. `branch` and
`cause` are bounded sets; `orderId` is not. That is Topic 118's cardinality drill.
Second, **the common pool has no Micrometer binder by default**, and that is a symptom
rather than an oversight: a resource with no owner has no metrics. If you want to graph
common-pool queue depth, the real answer is to stop using it.

You can reach it programmatically for a diagnostic dashboard —
`Gauge.builder("jvm.forkjoin.common.queued", ForkJoinPool.commonPool(), ForkJoinPool::getQueuedTaskCount)`,
plus `getActiveThreadCount` and `getStealCount`. Queued climbing while active is pinned
at parallelism is the starvation signature, expressed as two numbers.

### `jcmd` and JFR — the commands you will actually type

```bash
jcmd -l                                             # find the pid
jcmd $PID Thread.print -l > dump-1.txt              # full dump WITH ownable synchronizers

grep -A12 'ForkJoinPool.commonPool-worker' dump-1.txt   # blocking on the shared pool?
grep -c 'CompletableFuture$Signaller.block' dump-1.txt  # threads parked in join()

# Which *Async form got us here? AsyncSupply=supplyAsync, AsyncRun=runAsync,
# UniApplyAsync/UniComposeAsync/BiApplyAsync = a *Async continuation.
grep -oE 'CompletableFuture\$[A-Za-z]+' dump-1.txt | sort | uniq -c | sort -rn

grep 'java.lang.Thread.State' dump-1.txt | sort | uniq -c | sort -rn   # state histogram
```

**Three dumps, ten seconds apart, always.** One gives you state; three tell you whether
anything is moving. A thread stuck in `join()` appears identically in all three.

```bash
java -XX:StartFlightRecording=duration=300s,filename=cf.jfr,settings=profile -jar orderflow.jar
jfr summary cf.jfr
jfr print --events jdk.SocketRead cf.jfr | head -40
jfr print --events jdk.ThreadPark cf.jfr | head -60
jfr print --events jdk.JavaThreadStatistics cf.jfr | tail -20
```

| Event | What to read from it |
|---|---|
| `jdk.ThreadPark` with `parkedClass = CompletableFuture$Signaller` | Time spent in `join()`/`get()`. Aggregate by stack to find which endpoint is paying. |
| `jdk.SocketRead` / `jdk.SocketWrite` on a `commonPool-worker` stack | **Direct evidence of blocking I/O on the common pool.** This is the single cleanest proof for Trap 1. |
| `jdk.JavaThreadStatistics` | `activeCount` over time. Flat is healthy; monotonically rising is Topic 98's leak — and remember Proof 1's one-CPU case creates exactly that. |
| `jdk.ThreadStart` / `jdk.ThreadEnd` | High start rate with no matching ends on a 1-CPU container confirms the `ThreadPerTaskExecutor` fallback. |

The default `profile` settings apply a duration threshold to `jdk.ThreadPark`. If you
see nothing, lower it:

```bash
java -XX:StartFlightRecording=filename=cf.jfr,settings=profile,jdk.ThreadPark#threshold=1ms \
     -jar orderflow.jar
```

### The standing rule: a naive `System.nanoTime()` loop is wrong

You will be tempted to settle "is the fan-out faster" like this. Do not.

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 10_000; i++) {
    service.load(orderId);
}
System.out.println((System.nanoTime() - start) / 10_000 + " ns/op");
```

Five independent reasons it lies, and you cannot tell which one is lying:

1. **Cold JIT and on-stack replacement.** Your average blends interpreted, C1 and C2
   execution in a ratio set by the iteration count you happened to pick.
2. **Dead-code elimination.** If the result is never consumed, C2 may delete work.
3. **Single-threaded means uncontended.** The whole point of this topic is behaviour
   under concurrency. A serial loop exercises none of the pool, queue or
   connection-contention behaviour that decides the answer.
4. **The database caches.** After the first iteration you are measuring Postgres's
   buffer cache and your own L1 persistence context (Topic 48, Topic 51), not the work.
5. **It measures the wrong thing entirely.** The claim under test is "the endpoint's
   p99 under 400 rps of mixed traffic". A single-threaded loop cannot express that
   sentence.

**Topic 77 is the full treatment.** For a *component-level* question — "how much does the
handoff to another executor cost?" — the correct instrument is JMH:

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)                 // ONE shared executor -- the real situation
@Fork(3)                                // three JVMs: profile pollution shows as variance
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Threads({1, 2, 4, 8, 16, 32, 64})      // the scaling curve IS the result
public class ContinuationHandoff {

    private ExecutorService pool;

    @Setup   public void setup() { pool = Executors.newFixedThreadPool(8); }
    @TearDown public void down() { pool.shutdownNow(); }

    @Benchmark public Integer nonAsync() {          // no handoff: runs inline
        return CompletableFuture.completedFuture(1).thenApply(i -> i + 1).join();
    }
    @Benchmark public Integer asyncCommonPool() {   // handoff to the shared pool
        return CompletableFuture.completedFuture(1).thenApplyAsync(i -> i + 1).join();
    }
    @Benchmark public Integer asyncDedicated() {    // handoff to a dedicated pool
        return CompletableFuture.completedFuture(1).thenApplyAsync(i -> i + 1, pool).join();
    }
}
```

Four annotations doing specific work: **`@State(Scope.Benchmark)`** shares one executor
across all threads (`Scope.Thread` would give each thread its own and measure nothing);
**`@Threads({1,...,64})`** makes the *shape of the curve* the result rather than any
single number; **`@Fork(3)`** exposes profile pollution (Topic 74) that a single fork
would hide; **`@BenchmarkMode(AverageTime)`** asks for latency, because the question is
"what does one handoff cost".

**What to expect, stated honestly before you run it:** `nonAsync` should be by far the
cheapest — no queue, no handoff, no context switch. The two async forms should be close
at one thread and diverge as threads rise, with the shared common pool degrading first.
If your results contradict that, trust your machine and report the numbers; hardware,
JDK build and core count all move these curves.

### `perf` — and the honest note about macOS

Nothing in this topic requires hardware counters, but the standing rule applies. On
Linux:

```bash
perf stat -e context-switches,cpu-migrations -- java -jar orderflow.jar
```

Context-switch rate is the closest thing to a direct measure of handoff cost.

**On macOS, `perf` does not exist**, and there is no equivalent exposing the same
counters to a JVM process. Your options: run the experiment in a Linux container
(`--privileged` is usually needed for PMU access, and on Apple Silicon under
virtualisation the counters may be unavailable entirely), use
`xctrace record --template 'Time Profiler'` for CPU-time attribution without counters,
or accept the JMH curve as the measurement you can actually get. **Say which one you
did.** "I would confirm with `perf stat` on Linux; on my Mac I can only infer it from
the JMH curve" is a better answer than a confident claim about a number you cannot see.

---

## Practice exercises

### 1 — Easy: build your own threading reference card

Extend Proof 2 so it also demonstrates: `thenCompose` versus `thenApply` returning a
future (print the type with `getClass().getName()` and show the `thenApply` version is a
future whose *value* is another future); `whenComplete` versus `handle` versus
`exceptionally` on a **failing** future — which run, what each receives, and what the
downstream value becomes (does `whenComplete` change the outcome? does `handle`?);
`orTimeout` firing, and which thread delivers the exceptional completion; and
`cancel(true)` on a future backed by `Thread.sleep(30_000)`, printing whether the task's
thread is still alive five seconds later.

Produce one table: **method, whether it ran, what it received, what the downstream value
became, which thread it ran on.** Then answer: which row surprised you most, and what
wrong assumption did it correct? Think hardest about the `cancel(true)` row — write one
sentence explaining why a parameter named `mayInterruptIfRunning` does nothing.

### 2 — Medium: the audit (combines Topics 09, 25, 39, 46, 55, 56, 79, 90, 92, 94)

The fragment below contains **eight** distinct defects drawn from this topic and earlier
ones. Find them all, state the **exact symptom each produces in production** — not "it
is bad practice", but what the on-call engineer sees on a dashboard or in a log — and
rewrite it correctly.

```java
@Service
public class CheckoutService {

    private static final Map<String, CompletableFuture<Receipt>> inFlight = new HashMap<>();

    private final ExecutorService pool = Executors.newFixedThreadPool(200);

    @Transactional
    public Receipt checkout(CheckoutCommand cmd) {

        CompletableFuture<Reservation> reservation =
                CompletableFuture.supplyAsync(() -> inventory.reserve(cmd.sku(), cmd.qty()));

        CompletableFuture<DebitResult> debit =
                CompletableFuture.supplyAsync(() -> wallet.debit(cmd.userId(), cmd.total()));

        CompletableFuture<Receipt> receipt = reservation
                .thenCombine(debit, (r, d) -> new Receipt(r.id(), d.reference()))
                .exceptionally(ex -> {
                    if (ex instanceof InsufficientFundsException) {
                        return Receipt.declined();
                    }
                    throw new RuntimeException(ex);
                });

        if (!inFlight.containsKey(cmd.idempotencyKey())) {
            inFlight.put(cmd.idempotencyKey(), receipt);
        }

        CompletableFuture.runAsync(() -> auditLog.record(cmd), pool);

        gatewayClient.notifyAsync(cmd)
                     .thenRun(() -> log.info("notified user {}", cmd.userId()));

        return receipt.join();
    }
}
```

Hints, in no particular order: which pool each `supplyAsync` uses and what else shares
it; what `@Transactional` means for work on three other threads (Topic 55); what a
`HashMap` shared across request threads does at 400 rps (Topic 12), and what the
containsKey/put pair does even after you fix the map type (Topic 92 — the next topic's
headline bug, appearing here early); whether that `instanceof` can ever be true; what
happens to the audit future's exception; whether `notifyAsync(...).thenRun(...)` runs
where the author thinks; who shuts down that 200-thread pool (Topic 90); whether the MDC
correlation id reaches any of these log lines (Topic 120); and whether a `Receipt` from
`inFlight` is safe to hand to a second caller (Topic 116).

For each defect also state **which observable signal would first reveal it** — a metric,
a log line, a dump line, or a reconciliation query. Two of the eight have **no**
observable signal at all: identify which two, and say what you would add.

### 3 — Hard: production simulation on the `orderflow` baseline

**Part A — ground truth.** Bring up the Topic 65 stack. Re-run the recorded k6 baseline
for `GET /orders/{id}` and confirm p50/p95/p99 within ±10%. Record the container CPU
allocation, `availableProcessors()`, common-pool parallelism, Hikari maximum pool size,
and the `CatalogueIndexJob` duration from Part B of the drill. **Seven numbers. Write
them down.** If any of them is unknown, you cannot interpret anything that follows.

**Part B — implement three versions of the endpoint**, behind three URLs so you can
load-test them in one JVM run without deploy-to-deploy variance: (1) **serial**, the
original four reads in sequence; (2) **naive fan-out**, three `supplyAsync` calls with
no executor inside `@Transactional(readOnly = true)`; (3) **correct fan-out**, dedicated
bounded executor, per-branch timeouts and handlers, no surrounding transaction.

**Part C — measure all three at baseline load.** Produce one table: p50, p95, p99,
throughput, `hikaricp.connections.pending`, `executor.queued`, `executor.rejected`,
`CatalogueIndexJob` duration, and the common-pool thread-state histogram.

**Part D — measure all three at 3x load.** Same table. **The ranking will change**, and
explaining *why* it changed is the deliverable, not the table.

**Part E — break the database.** With version 3 deployed, add 3 seconds of latency to
one of the three branch queries (`pg_sleep`, `tc netem`, or Toxiproxy). Record what the
endpoint returns and how long it takes; whether the other two branches still complete;
whether they are **cancelled** (they are not — prove it by counting active queries in
`pg_stat_activity`); and `hikaricp.connections.pending` during the incident. Then do the
same with version 2 and compare the blast radius.

**Part F — the connection-pool ceiling.** Reduce Hikari to 8 and re-run version 3 at
baseline. Find the load at which the executor's queue is empty but Hikari's pending
count is not. **That is the point at which your thread pool stopped being the
constraint.** State the number and what you would change first.

**Part G — argue both sides, then design.** Make the strongest possible case for
reverting to the serial version, then the strongest case against yourself, and state the
exact condition under which your answer flips (hint: the ratio of p50 improvement to p99
regression, and which one your SLO is written against — Topic 130). Then sketch the same
endpoint with `StructuredTaskScope` (Topic 102): state precisely what it fixes that the
correct `CompletableFuture` version still gets wrong, and what it does **not** fix
(hint: it does not create database connections).

---

## Interview questions

### Q1 — "`CompletableFuture` is Java's Promise. Where does that break down?"

**Mid-level answer:** "They are pretty much the same. `thenApply` is `then`,
`exceptionally` is `catch`, `allOf` is `Promise.all`. The main difference is that Java
does not have `async`/`await`."

**Senior answer:** "The composition API maps almost one to one, so it reads fluently from
day one — which is the problem, because four things underneath are different and none of
them fail a unit test.

First, **which thread runs a continuation is a decision I have to make**. `thenApply`
runs on whichever thread completed the future, and if the future is already complete it
runs on my calling thread, synchronously. `thenApplyAsync` with no executor uses
`ForkJoinPool.commonPool()` — JVM-wide, `cores − 1`. Only the two-argument form is
predictable, so that is the only one I use.

Second, **there is no `await`**. `join()` blocks a real platform thread. In Node
suspension is a closure; in Java it is a stack and a scheduler slot, and if that thread
came from a bounded pool it is a unit of capacity.

Third, **exceptions are wrapped in `CompletionException`**, so `instanceof` against my
domain type in an `exceptionally` block is always false. Every handler unwraps.

Fourth — the one that costs money — **a future nobody joins discards its exception
silently**. Node prints an unhandled-rejection warning and, since 15, exits. Java stores
the throwable in the future's `result`, finds no dependents, and lets it be
garbage-collected: no log, no metric, no flag to enable one. So every fire-and-forget
future gets a terminal `whenComplete`, and anything the business depends on gets an
outbox instead."

**What separates them:** naming all four breaks; knowing the already-complete case for
non-async continuations; and knowing the silent-swallow behaviour *and* that Node's
default is the opposite. The last point is the one almost nobody has.

**Follow-up:** "What does `cancel(true)` do to a running `CompletableFuture` task?"
They want: nothing to the task. `CompletableFuture` ignores `mayInterruptIfRunning`; it
completes the future with `CancellationException` and the work continues.

---

### Q2 — "We added `supplyAsync` to parallelise three database calls and the nightly index job got ten times slower. Explain."

**Mid-level answer:** "The database is probably under more load now because of the
parallel queries."

**Senior answer:** "The database is a red herring. `supplyAsync` with no executor uses
`ForkJoinPool.commonPool()` — one JVM-wide pool sized `availableProcessors − 1`, so
three worker threads on a 4-vCPU container.

Three blocking JDBC calls per request occupy all three. The index job's parallel stream
submits to the *same* pool, so its tasks sit in the queue with no worker free to steal
them, and it degenerates to roughly single-threaded.

The crucial mechanical detail: a blocking JDBC call is **not** compensated.
`CompletableFuture.join()` from inside a FJ worker goes through
`ForkJoinPool.managedBlock`, so the pool knows and can start a compensation thread. A
raw socket read does not — the pool is never told, and that worker simply leaves the
stealing set.

One command confirms it: `jcmd <pid> Thread.print | grep -A12 commonPool-worker`. If
those threads are `RUNNABLE` inside a JDBC frame at zero CPU in `top -H`, that is the
whole diagnosis.

The fix is a dedicated bounded executor, sized from Little's Law and — importantly —
against the **connection pool**, not just CPU. Fan-out triples connection demand per
request, so the thread pool must never want more connections than Hikari can give, or I
have moved the queue rather than removed it. Raising
`ForkJoinPool.common.parallelism` is not a fix; it makes the failure rarer and harder to
reproduce."

**What separates them:** naming `cores − 1` and the shared-resource nature; the
`managedBlock` versus raw-blocking asymmetry; producing the one diagnostic command; and
connecting the fan-out to connection-pool amplification rather than stopping at threads.

**Follow-up:** "It does not reproduce on your laptop. Why?" Ten cores means nine
workers, and the submitting thread helps execute. `-XX:ActiveProcessorCount=2` forces it.

---

### Q3 — "The service is hung. Walk me through diagnosing it."

**Mid-level answer:** "Take a thread dump, look for deadlocks, restart the service to
restore availability."

**Senior answer:** "First: **do not restart.** A restart destroys the only evidence and
guarantees a second incident. If availability is critical, take the pod out of the load
balancer via readiness while leaving the process alive — that is the distinction Topic
121 exists for.

Then, in order:

1. **Three thread dumps, ten seconds apart**, to files. One gives state; three tell me
   whether anything is moving.
2. **A state histogram** of each — `grep 'Thread.State' | sort | uniq -c`. That one line
   puts me in one of four buckets: many `BLOCKED` on one monitor, many
   `WAITING (parking)` on one thing, many `RUNNABLE` at zero CPU, or a rising total.
3. **Search for `Found one Java-level deadlock`.** If present, the header names both
   threads and both resources and I am nearly done. **If absent, that does not mean
   there is no hang.** For this service I would look at threads parked at
   `CompletableFuture$Signaller.block` — `join()` with no timeout; what the
   `commonPool-worker` threads are doing — Trap 1; and anything in
   `HikariPool.getConnection`, which is Topic 109's pool deadlock, where the resource is
   a connection and `findDeadlockedThreads()` returns null.
4. **Corroborate with `top -H`.** `RUNNABLE` does not mean burning CPU — a socket read
   is `RUNNABLE`. Two hundred `RUNNABLE` threads at zero CPU points downstream.
5. **Then look at what is *not* hung.** If an endpoint sharing no code with the suspect
   path is also failing, the blast radius is a shared pool — request threads, the common
   pool, or the connection pool — and that tells me more than the stacks do."

**What separates them:** "do not restart" first; three dumps rather than one; knowing the
specific frames that indicate `CompletableFuture` problems; knowing the cases the
automatic deadlock detector misses; and reasoning from blast radius to shared resource.

**Follow-up:** "Two hundred threads are at `Signaller.block` and nothing is deadlocked.
Now what?" They want: find what is supposed to complete those futures, and note that
the fix is a timeout on every branch, not a bigger Tomcat pool.

---

### Q4 — "A notification is missing and there is nothing in the logs. Where do you look?"

**Mid-level answer:** "I would add more logging to the notification service and wait for
it to happen again."

**Senior answer:** "Missing side effect with zero log output is a specific signature, and
in a `CompletableFuture` codebase my first hypothesis is a **dropped future**. If nobody
calls `join()` and no handler is attached, the task's exception is stored in the
future's `result` field, `postComplete` finds an empty dependent stack, and the throwable
is garbage-collected with the future. No warning, no flag to enable one — unlike Node,
which prints an unhandled rejection and by default exits.

So I grep for `runAsync(` and `supplyAsync(` whose result is not assigned, not returned,
and not chained into a `whenComplete`, `handle` or `exceptionally`. That usually finds
it in under a minute.

The confirming evidence is a mismatch: `executor.completed` incremented — the executor
thinks the task finished — while the side effect did not happen. The executor is right;
the task completed *the future* exceptionally.

Three fixes, all of them. A terminal `whenComplete` that logs and increments a counter
tagged by exception class. A single `fireAndForget` helper plus an ArchUnit rule so a
bare `runAsync` cannot be merged. And for anything the business depends on — a payment
confirmation here — an **outbox** rather than a fire-and-forget future at all, because
'we sent it and hoped' is not a delivery guarantee. That is Topic 115."

**What separates them:** recognising "missing side effect, no logs" as a named
signature; explaining the mechanism at the level of `result` and `stack`; contrasting
with Node's behaviour; and escalating from a code fix to a durability design.

**Follow-up:** "Would a default uncaught-exception handler have caught it?" No — the
exception never escaped the `Runnable`; `AsyncRun.run` caught it and put it in the
future.

---

### Q5 — "When would you not use `CompletableFuture` at all?"

**Mid-level answer:** "When the code is simple enough to be synchronous."

**Senior answer:** "Four cases, and the last two are the interesting ones.

**When the calls are dependent.** Fan-out only helps when branches are independent.
Three dependent calls composed with `thenCompose` are the same wall clock as three
blocking calls, plus handoff overhead and a much harder stack trace.

**When the real constraint is downstream.** If the endpoint is limited by a
20-connection Hikari pool, parallelising three reads triples connection demand per
request and makes things worse. I check where the queue actually is first — otherwise I
have moved it, not removed it.

**On JDK 21+ for new fan-out code, I would reach for `StructuredTaskScope`.** `allOf`
waits but does not supervise: if one branch fails the others keep running and keep
holding connections, and `cancel(true)` does not interrupt them. A structured scope
binds subtask lifetime to a lexical block, so failure cancels siblings and the parent
stack trace stays intact. That is Topic 102, and it is a strictly better tool here.

**With virtual threads, much of this stops being necessary.** Three blocking calls on
three virtual threads cost almost nothing to create, and plain blocking code is easier
to read and debug than a future chain. `CompletableFuture` exists partly because
platform threads were expensive; Loom changes that premise. It does **not** change the
connection pool, which is usually the actual limit — so I still do the Little's Law
arithmetic first."

**What separates them:** knowing that fan-out is a latency tool with a resource cost;
naming the downstream constraint; and knowing that both `StructuredTaskScope` and
virtual threads change when `CompletableFuture` is the right answer — while being clear
that neither one creates database capacity.

**Follow-up:** "So should we delete all our `CompletableFuture` code?" No. It is the only
non-preview composition API across JDK versions, it is everywhere in libraries, and a
correct dedicated-executor version is perfectly good. Migrate at boundaries, not
wholesale.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `thenApply` on an **already-complete** future runs the function on the calling thread,
   synchronously. Why did the designers choose that instead of always scheduling? What
   would break — specifically, in terms of the `stack` field and `postComplete` — if it
   always scheduled?

2. `join()` from inside a `ForkJoinPool` worker triggers compensation via
   `ManagedBlocker`; a JDBC call from the same worker does not. Explain why the JDK can
   compensate for one and not the other, and what a `ManagedBlocker` wrapper around a
   JDBC call would and would not fix.

3. A future nobody joins discards its exception. Node warns and exits. Argue Java's
   choice is defensible, then make the strongest case against yourself. What would the
   JDK have to add to change it, and what would that cost?

4. `allOf` returns `CompletableFuture<Void>` rather than a future of the collected
   results. Given erasure (Topic 06) and the varargs signature, explain why it could not
   reasonably return anything else, and what a typed alternative would look like.

5. You parallelise three reads: p50 improves by 30 ms, p99 degrades by 40 ms. Explain
   the mechanism that makes the tail worse in terms of order statistics rather than
   Java. Then say which number you take to a product owner, and why.

6. On a one-CPU container, `supplyAsync` with no executor silently switches from the
   common pool to a thread-per-task executor. Was that the right design decision? What
   failure does it prevent, what failure does it create, and which would you rather
   debug?

7. `orTimeout` completes *your* future exceptionally but does not stop the underlying
   work. Name the resource still held after the timeout fires, and explain why timing out
   therefore makes a stressed service **worse** before it makes it better. Connect that
   to Topic 111's bulkhead argument.

---

## Quick reference card

### The three forms of every combinator

```java
cf.thenApply(fn)                 // completing thread; CALLER if already complete
cf.thenApplyAsync(fn)            // ForkJoinPool.commonPool() -- do not use
cf.thenApplyAsync(fn, executor)  // your executor -- the only predictable form
```

### The method map, TypeScript to Java

`.then(v => u)` → `thenApply` · `.then(v => promise)` → **`thenCompose`** (not
`thenApply`) · `.then(v => { side effect })` → `thenAccept` · `.catch` → `exceptionally`
· `.finally` → `whenComplete` · value-and-error transforming → `handle` ·
`Promise.all` → `allOf` (returns `Void`; join each future after) · `Promise.race` →
`anyOf` (returns `Object`) · combine exactly two → `thenCombine` · `Promise.resolve` /
`reject` → `completedFuture` / `failedFuture` · `await` → **nothing**; `join()` blocks a
thread.

### The unwrapping helper — write it once

```java
public static Throwable rootCause(Throwable t) {
    while ((t instanceof CompletionException || t instanceof ExecutionException)
            && t.getCause() != null && t.getCause() != t) {
        t = t.getCause();
    }
    return t;
}
```

### The fan-out idiom — the only correct shape

```java
private <T> CompletableFuture<T> branch(Supplier<T> work, String name, T fallback) {
    return CompletableFuture
            .supplyAsync(work, dedicatedExecutor)          // 1. explicit executor
            .orTimeout(250, TimeUnit.MILLISECONDS)         // 2. bounded wait
            .exceptionally(ex -> {                         // 3. terminal handler
                registry.counter("branch.failed", "branch", name,
                                 "cause", rootCause(ex).getClass().getSimpleName())
                        .increment();
                return fallback;
            });
}
```

### Thread states for this topic

| State | Frame | Meaning | Recovers? |
|---|---|---|---|
| `RUNNABLE` | `commonPool-worker` + socket/JDBC | **The bug.** Blocking on the shared pool. 0% CPU. | when the I/O returns |
| `WAITING (parking)` | `CompletableFuture$Signaller.block` | `join()`/`get()` with no timeout | only if something completes it |
| `TIMED_WAITING (parking)` | `Signaller.block` | `get(timeout, unit)` | **yes, guaranteed** |
| `WAITING (parking)` | `ForkJoinPool.awaitWork` | an **idle** common-pool worker | healthy — this is what idle looks like |

### Diagnostic commands

```bash
jcmd -l
jcmd $PID Thread.print -l > d1.txt
grep -A12 'ForkJoinPool.commonPool-worker' d1.txt        # blocking on the shared pool?
grep -c 'CompletableFuture$Signaller.block' d1.txt       # threads parked in join()
grep -oE 'CompletableFuture\$[A-Za-z]+' d1.txt | sort | uniq -c | sort -rn
grep 'java.lang.Thread.State' d1.txt | sort | uniq -c | sort -rn
top -H -p $PID                                           # RUNNABLE does not mean running

jfr print --events jdk.SocketRead cf.jfr | grep -B5 commonPool   # the cleanest Trap 1 proof
jfr print --events jdk.ThreadPark cf.jfr | head -60
```

### Gotchas checklist

- [ ] **Never** call the one-argument `*Async` form. Enforce with ArchUnit, not review.
- [ ] Every `supplyAsync` / `runAsync` names an executor.
- [ ] Every future you do not `join()` has a terminal `whenComplete` / `handle`.
- [ ] Every `instanceof` in a handler is against `rootCause(ex)`, never the parameter.
- [ ] `thenCompose`, not `thenApply`, when the function returns a future.
- [ ] `join()` appears once, at the top of the call stack, never in a library.
- [ ] Every branch has an `orTimeout`. `join()` without one is an unbounded wait.
- [ ] `allOf` does **not** cancel siblings on failure. `cancel(true)` does **not** interrupt.
- [ ] `thenApply` on an already-complete future runs on the **calling** thread.
- [ ] Fan-out multiplies connection-pool demand. Size the executor against Hikari.
- [ ] `@Transactional` does **not** cover work on other threads.
- [ ] `SecurityContext` and MDC do not cross the thread hop. Capture, restore, and clear.
- [ ] Always clear a `ThreadLocal` set on a pooled thread (Topic 79).
- [ ] On a 1-CPU container, no-executor `*Async` becomes thread-per-task. Check it.
- [ ] Do **not** restart a hung pod before taking three thread dumps.

---

## When would I use this at work?

**1. Making a read endpoint faster without touching the database.**

Product says the order-detail page is slow. You find four sequential queries summing to
88 ms, three of them independent. Fan-out with a dedicated executor takes the median to
roughly 50 ms with no schema change, no index and no cache. **But the deliverable is not
the code — it is the table showing p50, p99, `executor.queued` and
`hikaricp.connections.pending` before and after.** Half the value of this topic is
knowing that fan-out can improve p50 and degrade p99, and saying so first.

**2. Reviewing a pull request that adds `supplyAsync`.**

The diff is three lines and looks obviously good. You ask two questions: "which executor
does that use?" and "what happens to the other two futures if the third fails?" The
first catches the common pool; the second catches the missing cancellation. Then you
propose routing every fan-out through one `branch()` helper so the answer stops
depending on the next author's memory — Topic 94's `OrderedLocks` argument again.

**3. The incident where nothing is in the logs.**

A customer was charged and never confirmed. The order row is healthy, the wallet ledger
is healthy, the access log shows 202, and eleven hours of application logs contain no
error. You grep for `runAsync(`/`supplyAsync(` whose result is discarded and have the
root cause in two minutes. **The lasting value is the follow-up**: an ArchUnit rule so
it cannot recur, and a conversation about whether a business-critical side effect should
have been an outbox all along (Topic 115). That escalation — bug, to class of bug, to
design principle — is what gets you asked to review other teams' designs.

---

## Connected topics

**Prerequisites:**

- **21 / 22 — lambdas and method references.** Every combinator takes a functional
  interface; a capturing lambda allocates per evaluation, which matters in a hot fan-out.
- **25 — parallel streams.** The common pool, introduced. This topic's drill is Topic
  25's with JDBC instead of `sleep`, and the victim is your own endpoint.
- **55 — isolation and the connection pool.** A transaction pins a connection for its
  lifetime and is `ThreadLocal`-bound. Both facts are why `@Transactional` around a
  fan-out is wrong in two independent ways.
- **56 — the Spring Security filter chain.** `SecurityContextHolder` is `ThreadLocal`-
  backed, so it does not cross the thread hop. The first thing that silently breaks.
- **79 — `ThreadLocal` leaks.** A `ThreadLocal` set on a **pooled** thread and never
  cleared is retained for the life of the JVM. Always clear in a `finally`.
- **85–88 — monitors and the JMM.** Completion establishes a happens-before edge:
  everything the completing thread did before `complete()` is visible to the
  continuation. That is why fan-out results need no extra synchronisation — a guarantee
  you should be able to name rather than assume.
- **90 — executors and pool sizing.** The executor you pass *is* this topic's entire
  concurrency policy. Little's Law, bounded queues and rejection policies are
  prerequisites, not extras.
- **94 — explicit locks.** The thread-state table, `jcmd Thread.print -l`, and the rule
  that `RUNNABLE` does not mean running. Every diagnostic here builds on it.
- **100 — `ForkJoinPool` and work stealing.** Why a blocked worker cannot steal, and
  what `ManagedBlocker` compensates for. The mechanism underneath Trap 1.

**This unlocks:**

- **101 — virtual threads.** Changes the premise. When a thread costs almost nothing,
  plain blocking code beats a future chain for readability and debuggability. Pooling
  virtual threads is an anti-pattern precisely because the pool existed to ration a
  resource that is no longer scarce — but the connection pool still is.
- **102 — structured concurrency and scoped values.** The direct successor.
  `StructuredTaskScope` replaces `allOf` and fixes what it cannot: cancellation on
  failure, no orphaned tasks, and a stack trace that includes the parent. `ScopedValue`
  replaces the manual `SecurityContext`/MDC propagation in Example 2's Step 3.
- **104 / 105 — Reactor and backpressure.** Bounding the executor queue here is the same
  idea one layer down, and Part F of the drill — where bounding one queue relocates
  pressure to another — is Topic 105's central subject.
- **109 — the HikariCP pool deadlock.** Fan-out triples connection demand per request:
  the same four Coffman conditions as Topic 94 with connections as the resource,
  `findDeadlockedThreads()` returning null, and every thread parked in
  `HikariPool.getConnection`.
- **111 — bulkheads and resilience.** A dedicated executor per downstream **is** a
  bulkhead. `orTimeout` without retry-with-jitter is how you build a retry storm, and a
  timeout that does not cancel the work is why bulkheading by permits (Topic 97's
  `Semaphore`) often beats bulkheading by threads.
- **115 / 116 — outbox and idempotency.** The correct answer to Trace 2 is not a better
  handler; it is not firing and forgetting a business-critical side effect at all.
- **118 — metrics and cardinality.** `executor.queued`, `executor.rejected` and a
  branch-failure counter are this topic's permanent instrumentation. Tagging any of them
  with an order id kills the monitoring system.
- **120 — logging and MDC.** The correlation id that does not survive the thread hop.

---

*Java baseline 21, running on JDK 25. Two things here are deliberately hedged: the exact
`CompletionException` wrapping for an exception observed directly on the future you
completed versus one that propagated through a stage (it differs, it is not usefully
specified, and the correct response is to always unwrap rather than memorise cases); and
the precise internal class names in `CompletableFuture` stack frames, which are
implementation detail and have changed between releases — your own dump settles them.
Everything else — that `defaultExecutor()` is the common pool, that the common pool is
`cores − 1` and JVM-wide, that `join()` is compensated inside a FJ worker while raw
blocking is not, and that a future nobody observes discards its exception in silence —
is stable, checkable with the commands above, and will still be true the next time
somebody adds three lines to make an endpoint faster.*
