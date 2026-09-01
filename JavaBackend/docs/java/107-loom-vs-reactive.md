# 107 — The Loom-vs-Reactive Decision

## Phase: 10 — Reactive & Async at Scale
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25. Reactor, Netty and Tomcat versions come from the Spring Boot BOM. Do not pin them by hand and do not quote a version number from this document.
## Project spine: write and defend a recommendation for `orderflow` — which web and concurrency model each of its two paths should use — backed by the measurements from Topics 101, 105 and 106.

---

## Before anything else — what is and is not in this document

**I have no JVM, no Postgres, no Kafka, no k6, and no `orderflow` build. Nothing in this
document is captured tool output.**

This matters more here than in almost any other topic, because this document's whole
subject is *a decision made from data*. If I hand you numbers, you will remember the
numbers instead of learning how to produce them, and you will quote them in a design review
where they are wrong. So:

You will **not** find here:

- a throughput figure, a p50/p95/p99, or any latency number presented as measured,
- a thread count, a heap figure, or a CPU percentage from a run,
- a stack trace or a thread name captured from a process,
- "WebFlux was 3.2x faster" or "virtual threads matched WebFlux to 8x load" or anything
  shaped like a result,
- an engineer-week estimate for a migration presented as fact rather than as a method for
  producing your own.

You get instead, everywhere:

- **the exact command or config**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see → what it means"** table that covers the plausible outcomes *including
  the ones that contradict the fashionable answer*.

### The labelled exceptions

Twice below I show the **shape** of something — a thread name, a log line, the skeleton of
the recommendation document. Every value is a placeholder. Each carries the inline label:

> *illustration of the format, not captured output*

### Spec-level facts I state plainly, each with a confirming command

1. **A virtual thread's stack lives on the heap as a continuation and is copied to and from
   a carrier thread on mount/unmount.** Confirm: Topic 101's continuation-cost harness, and
   `jcmd <pid> Thread.dump_to_file -format=json`.
2. **Reactive Streams' `request(n)` travels upstream from consumer to producer; virtual
   threads have no such signal.** Confirm: Topic 105's `doOnRequest` proof, and the absence
   of any equivalent API in `java.lang.Thread`.
3. **A blocking call on a virtual thread parks the virtual thread and releases the carrier;
   a blocking call on an event loop stalls every connection on that loop.** Confirm: Topic
   101's proof, Topic 106's Trap 1, and BlockHound in both.
4. **Both Spring MVC-on-virtual-threads and Spring WebFlux are supported, first-class,
   non-deprecated options in Spring Boot 4.1.** Confirm: `spring.threads.virtual.enabled`
   in `/actuator/configprops`, and the presence of both starters in the BOM.

### THE RULE

> **If your measurements disagree with anything in this document, YOUR MEASUREMENTS ARE
> THE TRUTH.** This topic's deliverable is a recommendation *for `orderflow`*, not a
> recommendation for Java. A recommendation that cites a document instead of a run is an
> opinion wearing a table.

### What this document assumes you have already done

This is the one topic in the curriculum that is **not writable from scratch**. It consumes:

- **Topic 101** — `orderflow` on `spring.threads.virtual.enabled=true`, re-benchmarked at
  the Topic 65 baseline, with the pinning drill run.
- **Topic 105** — the payment-callback ingestion path with a bounded, explicit overflow
  strategy, and the OOM drill run to completion.
- **Topic 106** — the four-configuration comparison table (A: MVC/platform, B:
  MVC/virtual, C: WebFlux/R2DBC, D: WebFlux/JDBC-on-`boundedElastic`) swept across arrival
  rates.

If you have not run those, **stop and run them.** Everything below is the reasoning layer
on top of that data, and the reasoning layer without the data is exactly the blog post this
document exists to replace.

---

## Mechanical statement

Read this twice. Everything else is elaboration.

> **Both Loom and reactive solve the same problem: the cost of thread-per-request.**
>
> **Loom keeps the imperative programming model and moves the cost to heap-allocated
> continuations. Your code still blocks; the thing that blocks became cheap.**
>
> **Reactive changes the programming model to an operator chain over callbacks, and gets
> composable backpressure as a genuine extra.**
>
> **Loom does NOT give you backpressure. That is the one thing reactive still uniquely
> provides, and it is the entire residual case for reactive in 2026.**

Three consequences, stated now so you can hold them through everything that follows:

1. **If your reason for choosing reactive was thread economy, virtual threads took that
   reason away.** They did not weaken it. They took it.
2. **If your reason for choosing reactive was demand propagation across a pipeline —
   `request(n)` flowing upstream to a producer that can slow down — Loom has no answer at
   all.** Not a worse answer. No answer.
3. **The technical comparison is usually not what decides a real migration.** An existing
   WebFlux codebase has sunk cost in R2DBC, in operator fluency, in `checkpoint()`
   discipline, in reactive security. Rewriting it to imperative-on-virtual-threads is a
   large project with a modest payoff. That arithmetic, not the benchmark, is normally the
   answer.

---

## The bridge from what you know

### The same problem, approached from opposite ends

You already understand the problem precisely, because Node exists as an answer to it.

**The problem:** a request that spends 95% of its life waiting for I/O should not hold an
expensive resource while it waits. An OS thread costs a stack reservation (commonly around
a megabyte of virtual address space, some of it touched), a kernel scheduling entity, and a
context switch every time it is resumed. Ten thousand concurrent slow requests means ten
thousand threads, and that is a real machine.

**Node's answer, which you know cold:** make the I/O non-blocking, keep one thread, and
represent "what happens next" as a callback on the heap. The thread never waits, so one
thread is enough. The cost you pay is that the continuation is no longer a stack — it is a
closure graph, and every debugging tool you own was built for stacks.

**Reactive Java's answer:** the same one, with more structure. Netty's event loops are
libuv with N loops (Topic 103). Reactor's operator chain is your `Observable` pipeline
(Topic 104). "What happens next" is a `Subscriber` in a chain, not a stack.

**Loom's answer, which has no Node analogue at all:** *keep the blocking call.* Make the
thread cheap enough that having ten thousand of them is fine. The continuation is still a
stack — it just lives on the heap between mounts, and gets copied back onto a real carrier
thread's stack when it runs.

Say the shape of that out loud, because it is the whole topic:

| | Node / WebFlux | Loom |
|---|---|---|
| The call that waits | Made **non-blocking**; returns immediately | Left **blocking**; the caller genuinely waits |
| The thing that waits | Nothing waits; a callback is registered | A **thread** waits — but a cheap one |
| Where "what happens next" lives | Heap, as a closure / operator chain | Heap, as a `Continuation` holding stack frames |
| Who is expensive | Nothing, by construction | The thread — until Loom made it not be |
| What the programmer writes | `.flatMap(...)`, `await` | `var order = repo.load(id);` |
| Debugger and stack traces | Framework internals; assembly info lost (Topic 108) | Work exactly as they always did |

**One sentence: Node made the I/O cheap and kept one thread. Loom kept the blocking I/O and
made the THREAD cheap.** Opposite routes to the same place.

### The thing your Node background will get wrong

In Node, "non-blocking" and "backpressure" arrived together, so they feel like one idea.
They are not, and this topic depends on separating them.

- **Non-blocking / thread economy** is about *not tying up a scheduling resource while
  waiting*. It is a resource-utilisation property.
- **Backpressure** is about *a slow consumer telling a fast producer to slow down*. It is a
  flow-control property.

Node conflates them because `stream.pipe()` gives you both and you never had to ask. Java
separates them completely, and the separation is exactly where Loom's limit sits.

**Virtual threads give you unlimited non-blocking-equivalent concurrency and zero
backpressure.** That combination is dangerous in a way that platform threads were not,
because the bounded platform pool was *accidentally* doing admission control for you
(Topic 106 Trap 6, Topic 93). Remove the bound and you remove the accident.

This is Topic 90's lesson arriving from a new direction: **an unbounded queue is not
backpressure, it is a slower OOM.** A virtual-thread executor with no limit is an unbounded
queue made of threads.

### The RxJS instinct that transfers, and the one that does not

**Transfers:** operator composition. `merge`, `groupBy`, `window`, `sample`,
`retryWhen`, `timeout`, `zip`. If your problem is genuinely a *stream algebra* problem —
"group callbacks by merchant, window them by 200 ms, batch-settle each window, retry the
batch with backoff, and never let more than 4 merchants settle concurrently" — that is four
lines of Reactor and a substantial hand-rolled component in imperative code. **This is the
second thing reactive still gives you, and it is under-argued.** It is not as sharp as
backpressure, because you *can* write it imperatively; it is a productivity and
correctness-density argument, not an impossibility argument.

**Does not transfer:** the RxJS assumption that observables are the default way to express
asynchrony. In Java, since 21, they are not. Imperative code on a virtual thread is the
default, and reactive is the specialist tool for streams with demand.

**Verdict: STRONG ANALOGUE for the problem. NO ANALOGUE for Loom's solution — Node could
never have taken this route, because Node has one thread by design. And a specifically
misleading instinct that non-blocking implies backpressure.**

---

## What is this?

This topic is a decision, not an API. Here are the four things being decided between,
stated precisely so the comparison is honest.

| Option | Web tier | What a request occupies while waiting | Persistence | Backpressure |
|---|---|---|---|---|
| **A — MVC on platform threads** | Tomcat, `threads.max=200` | An OS thread | Hibernate + JDBC | **Accidental**: the bounded pool plus accept queue |
| **B — MVC on virtual threads** | Tomcat, `spring.threads.virtual.enabled=true` | A continuation on the heap | Hibernate + JDBC | **None by default** — you must add it |
| **C — WebFlux + R2DBC** | Netty event loops | A subscription and its operator state | Spring Data R2DBC | **Real, composable** — `request(n)` end to end |
| **D — WebFlux + JDBC on `boundedElastic`** | Netty event loops | A `boundedElastic` platform thread | Hibernate + JDBC | Bounded by `boundedElastic`, which is thread-per-request wearing a costume |

Option D is in the table because teams reach for it, and it is important to be able to say
why it is not a fifth position but a worse version of A.

### What each one actually is, mechanically

**Virtual threads (Topic 101).** `Thread.ofVirtual()` produces a `Thread` whose stack is a
`jdk.internal.vm.Continuation` on the heap. A scheduler — a `ForkJoinPool` in FIFO mode —
mounts it on a carrier platform thread to run. When it hits a blocking operation that the
JDK has been retrofitted for (socket I/O, `Thread.sleep`, most `java.util.concurrent`
locks), the JDK *yields*: it copies the live frames off the carrier's stack onto the heap
and frees the carrier for another virtual thread. When the I/O completes, the continuation
is mounted again — possibly on a different carrier — and the frames are copied back.

**Reactor (Topics 104–105).** A `Flux` is a *recipe*. Calling operators builds an object
graph at assembly time; nothing runs. `subscribe()` walks that graph building a `Subscriber`
chain from the bottom up, and the bottom-most subscriber calls `request(n)` upward. Data
flows down as `onNext` calls, demand flows up as `request(n)` calls. Threading is a separate
axis controlled by `publishOn`/`subscribeOn`.

**The key structural difference in one line:** *Loom's continuation is a suspended stack;
Reactor's chain is a state machine you assembled by hand.* Both are "the rest of the
computation on the heap". Only one of them is still shaped like the code you wrote.

### `[BOOT 3.x DELTA]`

`spring.threads.virtual.enabled` exists from Boot **3.2** onward, so the virtual-thread
option is available on 3.x too — this is not a Boot 4 feature. Two differences that matter:

- On Boot 3.x you are more likely to be on **JDK 21–23**, where `synchronized` pins the
  carrier. JDK 24 changed that. If your benchmark of option B is run on 21 and your
  production target is 25, the pinning column of your table does not transfer. State the
  JDK in the recommendation.
- Boot 3.x's `spring.threads.virtual.enabled` covers Tomcat's executor and several task
  executors, but the exact set of components it switches has grown across versions. Do not
  assume; print `/actuator/configprops` and check which executors are virtual in *your*
  version, and log a thread name from each path.

---

## Why does it matter?

Three reasons, in ascending order of how much they will affect your career.

**1. It is the highest-frequency senior Java interview question of this decade.** "Do
virtual threads make reactive obsolete?" is asked in some form at nearly every senior loop
that touches concurrency. It is asked *because* it separates people cleanly: the mid answer
is a slogan in either direction, and the senior answer concedes the thread-economy point,
names the residual, and then says that migration cost usually decides it anyway.

**2. It is a decision with a five-year blast radius.** Choosing WebFlux commits you to
R2DBC, to giving up Hibernate's persistence context, dirty checking and fetch graphs
(Topics 48–53), to reactive security, and to the debugging tax of Topic 108. Choosing
virtual threads commits you to auditing for pinning and to *adding back* the admission
control the platform pool used to provide accidentally. Neither is reversible cheaply.

**3. It is the rehearsal for Phase 12.** The deliverable here is a written recommendation
that a staff engineer would sign. Topic 131 teaches design-doc authorship formally; this is
where you first produce one under real constraints, with real data you gathered, defending
a position that might be "change nothing" — which is the hardest recommendation to write
well and the most common correct one.

### The honest state of the argument in 2026

Say this plainly, because a lot of writing on this topic is stale:

- Reactive Java's **mainstream** justification from roughly 2017 to 2021 was thread
  economy: "we can serve 20k concurrent connections without 20k threads."
- **Virtual threads deliver that same outcome without changing the programming model.**
  For that motivation specifically, they win, and it is not close, because the programming
  model is where all the cost was.
- What survives is **backpressure** and, more weakly, **stream composition**.
- Therefore: **request/response services should default to imperative on virtual threads.
  Streaming pipelines with a genuinely slow consumer and a producer that can be told to
  slow down should be reactive.** `orderflow` has one of each, which is why it is a good
  spine for this topic.

Anyone who tells you "reactive is dead" is wrong about backpressure. Anyone who tells you
"virtual threads are just green threads, reactive is still the answer" is wrong about the
programming model. Your job is to be the third person in the room.

---

## Machine-level reality

This is the section that makes the interview answer defensible rather than memorised.
**Where does each model pay its cost?**

### 1. The continuation vs the operator chain — both are "the rest of the computation on the heap"

Take one order-placement request that does three sequential I/O waits: load the product,
debit the wallet, call the payment gateway.

**On a virtual thread**, while waiting for the gateway, what exists on the heap is:

- a `VirtualThread` object,
- a `Continuation` holding **the actual live stack frames** — `placeOrder`, `debitWallet`,
  `gatewayClient.charge`, and every frame below down to the socket read,
- the objects those frames reference (the `Order`, the DTOs, the Hibernate persistence
  context for the open transaction).

**On a reactive chain**, while waiting for the gateway, what exists on the heap is:

- one `Subscriber` object per operator in the chain, each holding its own state,
- the captured lambdas and whatever they closed over,
- the `Context` map (Topic 108) carried in the subscription,
- the Netty channel state and any partially-parsed response.

**Both are the same idea. The difference is the encoding.** The continuation is a
*generic* encoding — the JVM copies frames it does not understand. The operator chain is a
*hand-written* encoding — you decomposed the computation into steps yourself, and each step
is an object.

Consequences that follow directly and that you should be able to derive on the spot:

| Consequence | Why it follows |
|---|---|
| Virtual-thread memory scales with **stack depth**, not just request count | The continuation copies live frames. A deep Spring + Hibernate call stack is a lot of frames. |
| Reactive memory scales with **operator count and buffered elements** | Each operator is an object; buffers are explicit. |
| Virtual threads give you a **real stack trace** | Because there is a real stack. It is just parked on the heap. |
| Reactive loses assembly information | Because the "stack" at failure time is the subscription path, not the code you wrote (Topic 108). |
| Mounting/unmounting a virtual thread has a **copy cost** | Frames are memcpy'd between heap and carrier stack. Frequent short parks are the bad case. |
| An operator hop has a **scheduling cost** | `publishOn` hands the item to another worker; that is a queue and a wakeup. |

**Neither is free. They are differently shaped.** A person who says "virtual threads are
free" has not measured a service with a deep stack and 50k in-flight requests; a person who
says "reactive has no overhead" has not counted the allocation of a ten-operator chain per
request.

### 2. Where each one blocks, and what blocking costs

This is the asymmetry that matters most operationally.

| | Blocking JDBC call happens... |
|---|---|
| **Platform thread (A)** | Thread parks. One of 200 pool threads is gone until the DB answers. Ceiling: 200 concurrent. Degrades **gracefully** — requests queue in the accept queue. |
| **Virtual thread (B)** | Virtual thread parks, **carrier is released**. Cost is a continuation on the heap. Ceiling: memory, and the DB pool. Degrades by **piling up at the pool** (Topic 109). |
| **Event loop (C/D)** | The loop thread is stuck. **Every other connection on that loop stalls.** Ceiling: catastrophic. Degrades by **falling over**. |

**The single most important line in this table:** a mistake that costs you one thread on
model A, and costs you a continuation on model B, takes out a whole slice of your
connections on model C. That difference in the *cost of a mistake* is a legitimate,
first-order input to the decision, and it is usually left out of comparisons.

**The pinning caveat (Topic 101).** On virtual threads there is one case where blocking
does *not* release the carrier: when the frame that blocks is inside a `synchronized` block
(on JDK 21–23) or inside a native frame. Then the carrier is pinned, and you have a
platform thread blocked with a tiny default pool. This is the one way model B can degrade
as badly as model C. It is auditable — that is what Topic 101's drill is for — and on JDK
24+ the `synchronized` case was largely addressed. **Your recommendation must state the
JDK, because this row of the table changes with it.**

### 3. Where backpressure lives — and where it does not exist

Be precise about what "backpressure" means, because the word is used for three different
things and only one of them is what reactive uniquely provides.

| Name | Mechanism | Available on virtual threads? |
|---|---|---|
| **Admission control** | Refuse or queue work at the door. A `Semaphore`, a bounded queue, a rate limiter. | **Yes** — you write it. `Semaphore`, `ThreadPoolExecutor` with a bounded queue, Resilience4j bulkhead (Topic 111). |
| **Flow control at the transport** | TCP window; the OS stops reading if you stop consuming. | **Yes**, implicitly, for socket reads — if you actually stop reading. |
| **Demand propagation across a pipeline** | `request(n)` travels upstream through every operator to the *producer*, which then produces less. | **NO. There is no equivalent.** |

The third one is the residual, and here is the sharpest way to see why it has no imperative
equivalent:

> In a reactive chain, a slow database writer at the end of a five-stage pipeline causes
> the Kafka consumer at the *start* to poll less. Nothing between them had to know. The
> signal is compositional: it passes through `map`, `filter`, `groupBy`, `window` without
> you writing any code.
>
> Imperatively, you achieve the same effect with a bounded queue between each stage and a
> blocking put. That works — it is Topic 93, and it is a completely legitimate design.
> **But it is not compositional.** You wrote each queue. Adding a stage means adding a
> queue and thinking about its bound. There is no operator algebra; there is plumbing.

So the honest form of the claim is not "Loom has no backpressure at all". It is:

> **Loom has no *composable* backpressure. You can build bounded, blocking pipelines by
> hand with `BlockingQueue`, and for a three-stage pipeline that is perfectly good
> engineering. Reactive gives you demand propagation as a property of the type, for free,
> across arbitrary composition.**

That is the sentence to say in an interview. It concedes what should be conceded and keeps
what is actually true.

### 4. The cost of a mistake, as a design input

Summarise the machine-level section into the thing that actually drives the recommendation:

| Mistake | Cost on A | Cost on B | Cost on C |
|---|---|---|---|
| Blocking call added by a junior on a hot path | One pool thread; visible in Tomcat metrics | One continuation; almost invisible | **Loop stall; site-wide latency spike** |
| Unbounded concurrency to a downstream | Bounded at 200 by the pool | **Unbounded — you overwhelm the downstream** | Bounded by demand, if the chain is honest |
| Forgetting to subscribe | N/A | N/A | **Silent no-op** (Topic 104) |
| `ThreadLocal`-based context (MDC, security) | Works | Works | **Broken** (Topics 108, 120) |
| Deep stack under high concurrency | Thread stacks; hits thread ceiling first | **Heap growth from continuations** | Not applicable |

Read the third column and the second row together. **The two models fail in opposite
directions, and each one's failure mode is the other's strength.** Reactive's discipline
requirement is high but its overload behaviour is principled; Loom's discipline requirement
is low but it will happily let you DDoS your own payment gateway.

---

## Example 1 — minimal

Two implementations of the same trivial thing: fetch a product, fetch its inventory, return
both. One I/O-bound step each. This is the smallest honest illustration of the programming
model difference.

### 1a — imperative, on a virtual thread

```java
// application.yaml:  spring.threads.virtual.enabled: true
@RestController
class ProductViewController {

    private final ProductRepository products;
    private final InventoryClient inventory;

    ProductViewController(ProductRepository products, InventoryClient inventory) {
        this.products = products;
        this.inventory = inventory;
    }

    @GetMapping("/api/products/{sku}/view")
    ProductView view(@PathVariable String sku) {
        log.info("handling sku={} on thread={}", sku, Thread.currentThread());

        Product product = products.findBySku(sku)          // blocks; carrier released
                .orElseThrow(() -> new ProductNotFound(sku));
        StockLevel stock = inventory.stockFor(product.id()); // blocks; carrier released

        log.info("finished sku={} on thread={}", sku, Thread.currentThread());
        return new ProductView(product, stock);
    }
}
```

Note what is *absent*: no operators, no lambdas, no `Mono`. `try`/`catch` works over the
whole method. A breakpoint on line 3 shows you `sku` and a stack that names
`ProductViewController.view`. `ProductNotFound` propagates to `@ControllerAdvice`
(Topic 46) exactly as it always did.

**Print the thread deliberately.** `Thread.currentThread()`'s `toString()` on a virtual
thread includes the carrier — that is the instrument that proves which model you are on.

> *illustration of the format, not captured output*
> ```
> VirtualThread[#<id>]/runnable@ForkJoinPool-<n>-worker-<m>
> ```
> versus a platform Tomcat thread:
> ```
> Thread[#<id>,http-nio-8080-exec-<m>,5,main]
> ```

### 1b — reactive, the same endpoint

```java
@RestController
class ReactiveProductViewController {

    private final ReactiveProductRepository products;   // Spring Data R2DBC
    private final WebClient inventory;

    @GetMapping("/api/products/{sku}/view")
    Mono<ProductView> view(@PathVariable String sku) {
        return products.findBySku(sku)
                .switchIfEmpty(Mono.error(() -> new ProductNotFound(sku)))
                .flatMap(product ->
                        inventory.get()
                                .uri("/stock/{id}", product.id())
                                .retrieve()
                                .bodyToMono(StockLevel.class)
                                .map(stock -> new ProductView(product, stock)))
                .checkpoint("product-view");   // Topic 108: buy back assembly info
    }
}
```

Everything the imperative version got for free now costs something:

| Free imperatively | Costs you reactively |
|---|---|
| `orElseThrow` on an empty result | `switchIfEmpty(Mono.error(...))` — and note the `Supplier` form, so the exception is not allocated on every call |
| Nesting two dependent calls | `flatMap` with a closure capturing `product` |
| A stack trace naming your controller | `.checkpoint("product-view")` — Topic 108 |
| MDC / correlation ID | `Context` + `ContextSnapshot` — Topic 108 |
| A debugger that steps | Stepping into Reactor internals |

**And what did the reactive version buy on this endpoint?** For a single request/response
with one downstream call: *nothing*. There is no stream, no slow consumer, no demand to
propagate. This is the honest baseline for the whole topic — **on request/response,
reactive costs and returns nothing.**

### 1c — the endpoint where that flips

Now a genuine stream: replay every payment callback recorded today to a slow reconciliation
sink.

```java
// Reactive: demand propagates from the sink all the way back to the DB cursor.
@GetMapping(value = "/api/internal/replay", produces = MediaType.APPLICATION_NDJSON_VALUE)
Flux<ReconciliationResult> replay(@RequestParam LocalDate day) {
    return callbacks.streamByDay(day)      // R2DBC streaming cursor: honours request(n)
            .limitRate(256)                // prefetch bound, explicit
            .concatMap(reconciler::apply)  // ordered, one at a time
            .checkpoint("replay");
}
```

The imperative equivalent on a virtual thread is not hard, but notice what you must write
yourself:

```java
// Imperative: you build the flow control by hand.
void replay(LocalDate day, Consumer<ReconciliationResult> sink) {
    try (Stream<PaymentCallback> cursor = callbacks.streamByDay(day)) {  // JDBC fetch size set!
        cursor.forEach(callback -> sink.accept(reconciler.apply(callback)));
    }
}
```

That works, and the blocking `sink.accept` naturally slows the loop, which naturally slows
the JDBC cursor — **as long as you remembered to set the JDBC fetch size**, because
otherwise the driver materialises the whole result set first and your "stream" is a list
with extra steps. The reactive version made the bound explicit and typed (`limitRate(256)`);
the imperative one made it a driver property that a config change can silently break.

**That contrast — explicit typed demand vs an easily-lost driver setting — is the real,
narrow, defensible case for reactive. It is not "reactive is faster".**

---

## Example 2 — production scenario (on the project spine)

### The two paths of `orderflow`, stated as they actually are

| | **Core API** | **Callback ingestion** |
|---|---|---|
| Endpoints | `POST /api/orders`, `GET /api/orders`, catalogue, wallet | `POST /api/payments/callbacks` from the gateway |
| Shape | Request/response, client waits for the answer | Fire-and-forget push from a third party |
| Concurrency at baseline | Bounded by real users | Bursty; gateway retries amplify bursts |
| Persistence | Hibernate: order + lines + inventory + wallet in one transaction (Topics 48–55) | Append to an inbox, then process |
| Who can be told to slow down | Nobody — a user is waiting | **Nobody either — but the buffer can be bounded and the strategy chosen** (Topic 105) |
| Consequence of dropping work | Failed order; user retries | **Lost payment confirmation. Unacceptable.** |
| Current state after Topics 101/105/106 | Benchmarked as A and B | Built as a WebFlux module (C) with a bounded overflow strategy |

### The decision, path by path

I will state the recommendation now and then show the work, because a recommendation buried
at the end of an analysis is a bad recommendation document (Topic 131).

---

**Core API → Option B: Spring MVC on virtual threads. Hibernate stays.**

The reasoning, in the order a reviewer will attack it:

1. **Is thread count the constraint?** Look at your Topic 106 table. If curves A and B are
   indistinguishable up to 4x the baseline, thread count is *not* your constraint at
   realistic load, and the entire thread-economy argument — for both B and C — is moot.
   That is the most likely finding and you must be willing to report it.
2. **If it is the constraint**, B removes it without a rewrite. One property.
   `spring.threads.virtual.enabled=true`. Hibernate, `@Transactional`, `MockMvc`, MDC,
   `SecurityContextHolder`, stack traces, the debugger — all keep working.
3. **What does C cost here?** R2DBC. That means losing the persistence context, dirty
   checking, lazy loading and fetch graphs — six topics of accumulated capability (48–53) —
   and rewriting every repository and the transactional order-placement flow. For a service
   whose core operation is a multi-table transaction, this is the worst possible place to
   give up Hibernate.
4. **Is there backpressure to lose?** No. There is no demand to propagate in
   `POST /api/orders`. A user is waiting. The only flow control that makes sense is
   admission control, and that is available on B.
5. **What does B cost?** Two things, both real, both listed in the recommendation as work
   items:
   - a **pinning audit** (Topic 101's drill), because a `synchronized` block around the
     gateway call would pin carriers;
   - **restoring admission control**, because the 200-thread Tomcat pool was doing it by
     accident and virtual threads remove it. See below — this is the biggest single risk of
     the recommendation and it must be named as such.

---

**Callback ingestion → Option C: keep WebFlux, keep the bounded `Flux`.**

1. **Is there demand to propagate?** Yes. The processing side — inbox insert, dedupe,
   downstream order update — is slower than a burst of gateway callbacks. That is the
   textbook slow-consumer case.
2. **Is the producer tellable?** Partly. The gateway is an HTTP client, so the honest
   mechanism is: bounded buffer, and when full, respond with 429/503 so the gateway retries.
   That is a *product decision* encoded in the overflow strategy (Topic 105), and it is
   expressible in one operator.
3. **Would virtual threads work here?** Yes — with a bounded `ArrayBlockingQueue`, a
   fixed-size consumer group, and an explicit rejection path. Roughly Topic 93. **That is a
   legitimate alternative and your document must say so**, because pretending it is
   impossible is the kind of overclaim that loses a design review.
4. **Then why keep C?** Because it already exists, it is already bounded and measured, the
   demand-propagation is typed rather than plumbed, and the module is small and isolated.
   **The deciding factor is that the migration cost is nonzero and the benefit is zero.**
   Which is the same reasoning as the core API's, applied in the other direction — and
   applying a principle symmetrically is what makes a recommendation credible.

---

### The risk this recommendation creates, named explicitly

Moving the core API from A to B **removes an admission-control mechanism that nobody
designed and everybody depended on.**

On A, 200 Tomcat threads meant at most 200 concurrent order placements, therefore at most
200 concurrent gateway calls, therefore at most 200 concurrent Hikari acquisitions. The
system had a ceiling. When it hit it, requests queued in the accept queue and, past that,
were refused at the socket. Ugly, but bounded, and the downstream was protected.

On B, there is no ceiling until memory. Ten thousand concurrent order placements will
attempt ten thousand concurrent gateway calls. The gateway will rate-limit you or fall over;
Hikari will queue ten thousand pending acquisitions (Topic 109); latency will go
super-linear and every request will time out together.

**The mitigation is mandatory and goes in the same PR as the property change:**

```java
@Configuration
class GatewayBulkheadConfig {

    // The number is a capacity decision, not a default. Derive it from the gateway's
    // documented concurrency limit and Little's Law (Topic 90), and record the derivation.
    @Bean
    Semaphore paymentGatewayPermits(
            @Value("${orderflow.gateway.max-concurrent}") int permits) {
        return new Semaphore(permits);
    }
}

@Component
class BulkheadedPaymentGateway implements PaymentGateway {

    private final PaymentGateway delegate;
    private final Semaphore permits;
    private final Timer waitTimer;

    @Override
    public PaymentResult charge(ChargeRequest request) {
        long start = System.nanoTime();
        boolean acquired;
        try {
            acquired = permits.tryAcquire(
                    properties.acquireTimeout().toMillis(), TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new GatewayUnavailable("interrupted waiting for a gateway permit", e);
        }
        waitTimer.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);

        if (!acquired) {
            // Shed load HERE, deliberately, with a 503 and a Retry-After.
            // Not at the socket, and not by exhausting the DB pool.
            throw new GatewayOverloaded("no gateway permit within the acquire timeout");
        }
        try {
            return delegate.charge(request);
        } finally {
            permits.release();
        }
    }
}
```

Three things to notice, because they are the difference between a bulkhead and a
decoration:

- **`tryAcquire` with a timeout**, not `acquire()`. `acquire()` on a virtual thread is free
  to the JVM and lethal to your latency SLO — you would simply move the unbounded queue
  from the gateway into the semaphore.
- **The wait time is a metric** (Topic 118). Permit-wait time rising is your earliest
  warning that the ceiling is binding.
- **The rejection is a typed exception mapped to 503 + `Retry-After`** (Topic 46), not a
  timeout somewhere deep in a driver.

Resilience4j's bulkhead (Topic 111) gives you the same thing with metrics included; use it
if it is already in the build. The point is not which library — it is that **the ceiling
must be re-created deliberately, and this is the single highest-risk consequence of
adopting virtual threads in an existing service.**

---

### The recommendation document — the skeleton

> *illustration of the format, not captured output — every `<...>` is yours to fill from
> your own runs*

```
# orderflow — concurrency and web-stack recommendation
Author: <you>   Date: <date>   Status: PROPOSED   Reviewers: <names>

## 1. Recommendation (read this and stop if you are busy)
   Core API:   adopt Option B (MVC on virtual threads). Effort: <n> days.
   Ingestion:  keep Option C (WebFlux + bounded Flux). Effort: 0.
   Mandatory companion change: gateway bulkhead + Hikari pending-acquire alert.
   If you read nothing else: <one sentence naming the single deciding fact>.

## 2. What question we are answering, and what we are NOT
   In scope:  which web/concurrency model each orderflow path should use.
   Out of scope: <e.g. moving to Kafka-based ingestion; that is Topic 113's decision>.

## 3. Evidence
   3.1 Environment (JDK <ver>, Boot <ver>, container limits, collector, dataset,
       load model, warm-up window, repetitions)     <-- credibility lives here
   3.2 The four-configuration table from Topic 106, swept 0.5x-8x
   3.3 The virtual-thread before/after from Topic 101, including the pinning audit
   3.4 The backpressure evidence from Topic 105 (the OOM, and the bounded fix)
   3.5 What we could NOT measure, and why

## 4. Analysis
   4.1 Is thread count the constraint at our real arrival rates?   <yes/no + which run>
   4.2 What each option costs to adopt (engineer-weeks, capability lost)
   4.3 What each option costs to operate (debugging, on-call, hiring)
   4.4 The cost of a mistake under each model

## 5. Risks of the recommendation
   R1 Loss of accidental admission control      Mitigation: bulkhead (PR #<n>)
   R2 Pinning under <synchronized site>          Mitigation: <ReentrantLock / JDK 25>
   R3 Heap growth from continuations             Mitigation: alert on <gauge>
   R4 <the risk you least want to write down>

## 6. Alternatives considered and rejected
   A (status quo): rejected because <...>  -- or ACCEPTED, if that is the finding
   C for the core API: rejected because <...>
   D: rejected because it is thread-per-request with extra steps; evidence: <run>

## 7. Decision criteria for revisiting
   Revisit if: sustained arrival rate exceeds <n>/s, OR ingestion burst factor
   exceeds <n>, OR the gateway publishes a streaming API.

## 8. Rollout and rollback
   Property-flagged, one instance first, <n> hours at baseline, rollback = flip the
   property and restart. Success criteria: <specific metric thresholds>.
```

**Sections 5, 6 and 7 are what make it a staff-level document.** Anyone can write sections
1–4. A recommendation with no stated risks reads as advocacy; a recommendation with no
revisit criteria is a decision nobody can ever revisit without re-litigating it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — adopting reactive for thread economy, in 2026

**Wrong approach.** A team hits the Tomcat thread ceiling under a traffic spike. Someone
proposes WebFlux, citing the thread-economy argument. A six-month migration to R2DBC begins.

**Exact symptom.** Observable, in this order:

- Before: `tomcat.threads.busy` pinned at `tomcat.threads.config.max`, CPU well below
  saturation, latency climbing.
- After migration: thread count is far lower — **the migration delivered exactly what was
  promised** — and p99 is unchanged or worse, because the constraint was the Hikari pool or
  the database, not the web tier.
- Also after: every N+1 fix from Topic 50 has been re-solved by hand in SQL; MDC is gone
  from half the logs; the on-call runbook has a section called "reading Reactor stack
  traces".

**Root cause.** The thread-economy problem was real, and it had a one-property solution
that nobody evaluated. Virtual threads remove the same ceiling with none of the model
change. The decision was made from a 2019 mental model.

**Fix.** Before any reactive migration proposal is accepted, require a run of Option B in
the comparison. It costs one property and one benchmark run:

```bash
java -Dspring.threads.virtual.enabled=true -jar orderflow.jar
# then re-run the Topic 65 harness at matched arrival rates
```

**WHAT TO LOOK FOR:** whether the throughput ceiling moves. Then:

| What you see | What it means |
|---|---|
| Ceiling moves substantially; CPU now saturates | Thread count *was* the constraint, and B fixed it. The reactive proposal is now unjustified on thread-economy grounds. |
| Ceiling does not move; `hikaricp.connections.pending` high | The pool is the constraint (Topic 109). **Neither B nor C helps.** Both proposals were mis-aimed. |
| Ceiling does not move; DB CPU saturated | The database is the constraint. Go optimise queries. |
| Ceiling moves, but p99 gets *worse* | You removed the admission control (Trap 3). The bulkhead is missing. |
| Throughput collapses and CPU is idle | **Pinning.** Run the Topic 101 audit before concluding anything about virtual threads. |

**The generalisation worth keeping:** *never accept a proposal to change the programming
model until you have run the version that does not.*

### Trap 2 — migrating an existing WebFlux codebase on technical merit alone

**Wrong approach.** The mirror image, and increasingly the more common mistake now that the
Loom argument has become fashionable. A team on WebFlux reads that virtual threads make
reactive unnecessary and proposes rewriting to imperative.

**Exact symptom.**

- A migration plan with no line item for R2DBC → JDBC, no line item for reactive-security →
  servlet-security, and no line item for retraining.
- Mid-migration, a codebase in **both** styles, with `block()` calls at the seams — which
  is the worst of all four options and will show up as `boundedElastic` thread growth or,
  worse, `block()` on an event-loop thread throwing `IllegalStateException`.
- The stated benefit — "simpler code" — cannot be shown on any dashboard, so the project
  loses funding halfway.

**Root cause.** The technical comparison was treated as the decision. It is an *input*.
The decision is a cost/benefit, and the benefit side of "our existing working service
becomes easier to debug" is real but diffuse, while the cost side is concrete and large.

**Fix.** Force the arithmetic into the document before any code moves:

| Line item | How to estimate it, honestly |
|---|---|
| Repository rewrite | Count `ReactiveCrudRepository` methods and custom R2DBC queries. Time-box a rewrite of the three hardest; extrapolate. |
| Transaction semantics | Every place a reactive transaction spans multiple operations. These are the bug farms. |
| Security | Reactive `SecurityWebFilterChain` → servlet filter chain. Count the custom filters. |
| Tests | `WebTestClient` → `MockMvc`/`RestTestClient`, plus every `StepVerifier`. |
| Observability | Reverse of Topic 108: `Context` → MDC. Usually *easier*, and worth counting as a benefit. |
| Retraining | The team's reactive fluency is an asset you are writing off. Say so. |
| **Risk of a half-migrated codebase** | The largest item. Estimate the window and who is on call during it. |

**Then apply the rule:** migrate a working reactive service only when there is a *forcing
function* — a rewrite already happening for another reason, a persistent on-call cost you
can measure (Topic 108's debugging tax, quantified as incident minutes), or a hiring
constraint you can evidence. **"It would be simpler" is not a forcing function.**

**Observable proof that a half-migration is happening:** grep the build for both starters,
and alert on `boundedElastic` thread count.

```bash
mvn -q dependency:tree | grep -E "starter-web(flux)?"
# Two hits in one module = the seam exists. Verify which context actually started:
grep -m1 "Application" logs/app.log   # AnnotationConfigServletWebServerApplicationContext
                                      # vs AnnotationConfigReactiveWebServerApplicationContext
```

### Trap 3 — believing virtual threads gave you backpressure

**Wrong approach.** `spring.threads.virtual.enabled=true` ships. The team reasons: "we used
to be capped at 200 concurrent requests and now we are not, so we handle more load." No
bulkhead is added.

**Exact symptom.** Under a spike, in this order:

1. Request concurrency climbs far past the old 200.
2. `hikaricp.connections.pending` climbs; `hikaricp.connections.acquire` timer's p99 climbs.
3. The payment gateway starts returning 429s, or its latency triples.
4. **Every** request times out at once, rather than some requests being refused quickly.
5. Heap climbs — thousands of parked continuations, each pinning a Hibernate persistence
   context and a partially-built order.
6. On-call sees "high latency everywhere, CPU not saturated" and cannot name a cause.

**Root cause.** The bounded thread pool was performing admission control. Virtual threads
removed the bound and nothing replaced it. **You did not gain capacity; you gained the
ability to accept work you cannot do.** This is Topic 90's unbounded-queue lesson in a new
costume — the queue is now made of threads.

**Fix.** Re-create the ceiling deliberately, at each contended resource, as in Example 2:

- a `Semaphore` or Resilience4j bulkhead per downstream, sized from that downstream's
  documented capacity;
- `tryAcquire` with a timeout and a fast 503, never an unbounded `acquire()`;
- **alert on permit-wait time and on `hikaricp.connections.pending`, not on thread count** —
  thread count is no longer a meaningful saturation signal, and if your dashboards still
  treat it as one they are now lying to you.

**WHAT TO LOOK FOR after the fix:**

| What you see | What it means |
|---|---|
| Some requests get a fast 503; the rest keep normal latency | The bulkhead is working. This is what healthy overload looks like. |
| Permit-wait p99 near the acquire timeout | Ceiling is binding routinely. Either raise it with evidence or add capacity. |
| No 503s and pending-acquire still climbing | The bulkhead is on the wrong resource. Find the one that actually saturates. |
| 503s at a rate far above the spike | Ceiling set too low. Re-derive from Little's Law (Topic 90). |

### Trap 4 — benchmarking at the wrong arrival rate and concluding either way

**Wrong approach.** Run a closed-model load test — 50 virtual users in a loop — against A,
B and C. All three produce similar throughput. Conclude "there is no difference, the whole
debate is hype."

**Exact symptom.** A comparison table where all configurations are within noise of each
other at every point, and no configuration ever shows an error. This looks like a clean
result and is a broken experiment.

**Root cause.** Two errors, usually together:

1. **Closed model.** With a fixed number of looping users, offered load automatically drops
   when the system slows down. A closed-model test *cannot* show a saturation cliff,
   because it removes the load that would cause one. You must drive an **open-model arrival
   rate** — k6's `constant-arrival-rate` executor — so that offered load is independent of
   how the system is coping.
2. **Concurrency below the binding point.** The difference between A and B only exists once
   concurrency exceeds `server.tomcat.threads.max`. At 50 concurrent requests against a
   200-thread pool there is nothing to see, and a table showing that is a table showing
   nothing.

**Fix.** The sweep, explicitly:

```javascript
// k6 — open model. Offered rate does NOT depend on how the system is coping.
export const options = {
  scenarios: {
    sweep: {
      executor: 'ramping-arrival-rate',
      startRate: 0, timeUnit: '1s',
      preAllocatedVUs: 2000, maxVUs: 20000,   // must exceed what saturation needs
      stages: [
        { target: BASE * 0.5, duration: '3m' },
        { target: BASE * 1,   duration: '3m' },
        { target: BASE * 2,   duration: '3m' },
        { target: BASE * 4,   duration: '3m' },
        { target: BASE * 8,   duration: '3m' },
      ],
    },
  },
};
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| All configs identical at every rate, no errors, CPU low | You never reached saturation. Raise the top of the sweep or you have measured nothing. |
| `dropped_iterations` in the k6 summary | **k6 could not generate the offered rate.** The load generator saturated, not the service. Every number in that stage is invalid. Raise `maxVUs` and re-run. |
| A diverges at some rate; B and C track each other above it | Thread count was the constraint and B fixed it without a rewrite. **This is the finding that makes the recommendation.** |
| All configs diverge together at the same rate | A shared downstream is the constraint — the DB or the gateway. The web tier is irrelevant. |
| C ahead only above 4x baseline | Quantify the gap, then ask whether you will ever run at 4x. Usually the answer ends the debate. |

### Trap 5 — reading a pinning result as a verdict on virtual threads

**Wrong approach.** Option B is benchmarked. Throughput is *worse* than A and CPU is idle.
Conclusion: "virtual threads do not work for us."

**Exact symptom.** Low throughput, idle CPU, and a small number of `ForkJoinPool-*-worker-*`
carrier threads all blocked in the same frame.

**Root cause.** Almost always **pinning** (Topic 101): a `synchronized` block — often in a
third-party client, a legacy cache, or a `synchronized` singleton initialiser — held across
a blocking call. On JDK 21–23 the carrier cannot be released. The default carrier pool is
sized to the core count, so a handful of pinned carriers is a near-total stall.

**Fix.** Audit before concluding:

```bash
# JDK 21-23 only: this system property was REMOVED in JDK 24.
java -Djdk.tracePinnedThreads=full -jar orderflow.jar

# Any version, and the correct instrument going forward:
jcmd <pid> JFR.start name=pin settings=profile filename=pin.jfr
jcmd <pid> JFR.dump name=pin filename=pin.jfr
jfr print --events jdk.VirtualThreadPinned pin.jfr | head -80
```

| What you see | What it means |
|---|---|
| `jdk.VirtualThreadPinned` events naming a frame in your code | Confirmed pinning. Replace `synchronized` with `ReentrantLock`, or move the blocking call out of the block. |
| Events naming a third-party library frame | Confirmed, and harder. Options: wrap that client on a dedicated platform-thread executor, upgrade the library, or run on JDK 24+. |
| No events, on JDK 21–23 | Genuinely not pinning. Look at the pool (Topic 109) or the DB. |
| No output at all from `jdk.tracePinnedThreads` | **On JDK 24+ the property does not exist. Its silence is not evidence.** Use JFR. |
| Thread dump shows carriers blocked in `Object.wait`/monitor frames | Corroborates pinning. Cross-check with JFR. |

**And the version note that must appear in your recommendation:** JDK 24 changed the
`synchronized` pinning story substantially. **A pinning benchmark run on 21 does not
transfer to a 25 deployment.** If you measure on one and deploy on the other, say so in the
Evidence section, or your document is wrong in a way a reviewer will find.

---

## Hands-on proof

Every claim in this topic reduces to one of five observations. All five use instruments you
already have.

### Proof 1 — which model is a given request actually running on?

Do not infer it. Print it.

```java
@Component
class ThreadNameFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        Thread t = Thread.currentThread();
        log.info("uri={} virtual={} thread={}", req.getRequestURI(), t.isVirtual(), t);
        chain.doFilter(req, res);
    }
}
```

**WHAT TO LOOK FOR:** `virtual=true` and a `VirtualThread[#...]/runnable@ForkJoinPool-...`
name, versus `virtual=false` and `http-nio-8080-exec-N`.

| What you see | What it means |
|---|---|
| `virtual=false` with the property set to true | The property did not take effect for this component. Check `/actuator/configprops` and your Boot version — coverage differs across versions. |
| `virtual=true` on the request thread but `false` in an `@Async` method | Only some executors were switched. Configure the rest explicitly. |
| A `reactor-http-nio-*` name on an endpoint you thought was MVC | Both starters on the classpath, or you are on the wrong module. Topic 106. |

### Proof 2 — the continuation is a real stack

```bash
jcmd <pid> Thread.dump_to_file -format=json -overwrite dump.json
```

**WHAT TO LOOK FOR:** virtual threads appear in this dump with **full stack traces naming
your application frames**, mounted or not. This is the single most convincing demonstration
that Loom kept the imperative model: the parked continuations are still readable stacks.

Now do the equivalent for reactive: take a thread dump of the WebFlux module while a request
is in flight, and look for a frame naming your handler. You will not find one — the handler
returned long ago. **That contrast is the debugging cost of reactive, made visible in ninety
seconds.** Topic 108 is the whole story.

### Proof 3 — BlockHound, on both stacks

BlockHound instruments the JVM to throw when a blocking call happens on a thread marked
non-blocking.

```xml
<dependency>
  <groupId>io.projectreactor.tools</groupId>
  <artifactId>blockhound-junit-platform</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@Test
void nothingBlocksTheEventLoop() {
    BlockHound.install();
    webTestClient.post().uri("/api/payments/callbacks")
        .bodyValue(callback)
        .exchange()
        .expectStatus().isAccepted();
}
```

**WHAT TO LOOK FOR:** a `BlockingOperationError`, and specifically **which frame** it names.

| What you see | What it means |
|---|---|
| `BlockingOperationError` naming a JDBC frame | A blocking driver on an event loop. The bug Topic 106 Trap 1 is about. |
| Error naming a logging or filesystem frame | Often a false positive worth allow-listing deliberately — but read it before you allow-list it. |
| BlockHound fails to install | It uses instrumentation and can lag JDK releases. If it will not install on your JDK, say so in the recommendation rather than claiming the path is verified. |
| No error, but latency is bad | Blocking is not your problem. Look at the pool. |

**The asymmetry to note in your document:** BlockHound is *necessary* for option C, because
a single blocking call is catastrophic there. On option B it is merely *informative* — a
blocking call is the normal thing to do. **That difference in required discipline is a
staffing and code-review cost, and it belongs in the cost/benefit table.**

### Proof 4 — demand propagation exists in one model and not the other

```java
Flux.range(1, 1_000_000)
    .doOnRequest(n -> System.out.println("SOURCE saw request(" + n + ")"))
    .limitRate(16)
    .concatMap(this::slowStep)
    .blockLast();
```

**WHAT TO LOOK FOR:** repeated `SOURCE saw request(...)` lines interleaved with processing —
the source is being asked for small amounts, repeatedly, because the consumer is slow.

Now write the imperative equivalent and try to find the equivalent signal. There is none.
There is only "the loop is slow, so it iterates slowly", which works but is not a signal,
cannot be observed, and cannot be composed. **This proof is the mechanical basis for the
sentence "Loom does not give you backpressure", and running it once makes that sentence
yours rather than borrowed.**

### Proof 5 — the accidental ceiling, removed

```bash
# Option A: cap is visible in the metric.
curl -s localhost:8080/actuator/metrics/tomcat.threads.busy
curl -s localhost:8080/actuator/metrics/tomcat.threads.config.max
```

**WHAT TO LOOK FOR:** under load on A, `busy` sits at `config.max`. Switch to B and re-run:
that metric becomes meaningless (there is no bounded pool to be busy), and
`hikaricp.connections.pending` becomes the signal that matters.

| What you see | What it means |
|---|---|
| On A: `busy == config.max` sustained | The web tier is your admission control. Removing it changes behaviour — plan for it. |
| On B: pending-acquire climbing where `busy` used to sit at max | **The constraint moved from the web tier to the pool.** Exactly as predicted; now bulkhead it. |
| On B: no metric saturates but latency climbs | The constraint is downstream and now unprotected. Trap 3. |

---

## Failure drill

**For this topic, the drill is the recommendation itself.** That is deliberate. Every other
ELITE topic breaks a running system; this one breaks a *decision*, and the failure mode
being drilled is the one that actually damages careers: a confident architectural
recommendation that does not survive review.

### The drill

Write the recommendation from the Example 2 skeleton. Then run it through the adversarial
review below **before** showing it to anyone. Every question here is one a staff engineer
will ask, and each one has broken a real design document.

### Part A — the evidence attack

| Question | Fails if |
|---|---|
| "What JDK produced these numbers, and what JDK do we deploy?" | They differ and you did not say so. Pinning behaviour changed in JDK 24. |
| "Was the load model open or closed?" | Closed, or you do not know. Every conclusion is then unfounded (Trap 4). |
| "What were `dropped_iterations` in each stage?" | Nonzero and unreported — the generator saturated, not the service. |
| "How many repetitions, and what was the spread?" | One run. A single run cannot distinguish a 10% effect from noise. |
| "What was the constraint in each configuration?" | You report throughput without naming what limited it. Then you measured an outcome without a mechanism. |
| "Which numbers came from a different environment than the others?" | Any of them. Mixed environments invalidate the comparison silently. |

### Part B — the reasoning attack

| Question | The answer that survives |
|---|---|
| "You recommend virtual threads. What do we lose?" | The accidental admission control. Named as R1, with the bulkhead PR as mitigation. **If you did not name this, you fail the drill.** |
| "You keep WebFlux for ingestion. Could virtual threads do that job?" | **Yes** — bounded queue, fixed consumers, explicit rejection. It is rejected on migration cost, not on impossibility. Overclaiming here loses the room. |
| "Reactive is faster though, isn't it?" | At the same throughput it is not; it uses fewer threads. That only converts to a win when thread count is the constraint, and here is the run showing whether it is. |
| "What if we do nothing?" | Section 6 must contain a real, costed status-quo option. If your document cannot articulate why doing nothing is worse, it is advocacy. |
| "What would change your mind?" | Section 7's revisit criteria, with numbers. "More load" is not an answer; "sustained arrival rate above `<n>`/s" is. |
| "Who is on call for this and what does their runbook change to?" | A specific answer about which dashboards and alerts change. Thread count stops being a saturation signal — that is a runbook edit. |

### Part C — the two failure modes to catch in yourself

**Failure mode 1 — the recommendation that is really a preference.** Symptom: every risk
listed is small, every alternative is dismissed in one line, and the estimated effort for
your preferred option is suspiciously low. **Test:** write the strongest possible version of
the option you rejected, as if you believed it. If you cannot, you have not understood it
well enough to reject it.

**Failure mode 2 — the recommendation that refuses to recommend.** Symptom: a thorough
analysis ending in "it depends on the workload". **Test:** section 1 must contain an
imperative sentence. "Adopt B for the core API." A staff engineer is paid to be wrong
occasionally in public; a document that cannot be wrong cannot be right either.

### What the drill proves

That you can hold a position under pressure *and* concede the parts that should be conceded.
The interview question in this topic is the same drill in five minutes instead of five days,
and the same two failure modes lose it.

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` loop is the WRONG way to measure JVM performance.** It
> measures JIT warm-up, dead-code elimination, on-stack replacement and ambient noise, in
> unknown proportions. Topic 77 (JMH) is where you learn to do it properly.

**And a sharper rule specific to this topic: JMH is also the wrong tool here.** The
difference between these models does not exist in a microbenchmark. It exists only at
concurrency levels above where the bounded pool binds, over sustained periods, with a real
network and a real database. **The instrument is the Topic 65 harness, always.** Reaching
for JMH on this question is itself a diagnostic error, and an interviewer who hears you
suggest it will notice.

### The instrument for each claim

| Claim in your recommendation | Instrument | What invalidates it |
|---|---|---|
| "We are on virtual threads" | `Thread.isVirtual()` logged per request; `/actuator/configprops` | The property set but a component not covered by your Boot version |
| "Nothing pins a carrier" | JFR `jdk.VirtualThreadPinned` under load | Auditing at idle; using `jdk.tracePinnedThreads` on JDK 24+ where it no longer exists |
| "Thread count is/is not the constraint" | Sweep A vs B, open model, to saturation | Closed model; sweep top too low; `dropped_iterations` unreported |
| "Nothing blocks the event loop" | BlockHound in CI + thread names inside operators | BlockHound not installing; asserting instead of testing |
| "Backpressure works end to end" | `doOnRequest` logging at the source under a slow consumer | Any unbounded `Sinks.many()` or `onBackpressureBuffer()` in the middle (Topic 105) |
| "Duplicate work does not increase under load" | Duplicate counters (Topic 116) at each arrival rate | Measuring correctness only at 1x |
| "The migration costs N weeks" | Time-boxed spike on the three hardest repositories | An estimate with no spike behind it |
| "This is the constraint" | Pool pending-acquire, DB CPU, gateway latency, all graphed together | Naming the component you changed rather than the one that saturated |

### The comparison table — the deliverable's core

This is Topic 106's four-configuration table, consumed here. For each of A, B, C, D, at
arrival rates 0.5x / 1x / 2x / 4x / 8x of the Topic 65 baseline:

| Metric | Why it is in the table |
|---|---|
| Offered vs achieved throughput | The gap is where the ceiling is. Without offered rate, the table is meaningless. |
| p50 / p95 / p99 / p999 | p50 rarely moves. The tail is the entire story. |
| Error rate **and shape** | Refused-at-socket vs timeout vs 503 tells you *where* the queue is. |
| Peak thread count | The thread-economy claim, measured rather than asserted. |
| Peak heap used | Reactive's operator state, Loom's continuations. Both real. |
| CPU utilisation | Idle CPU with low throughput means parked threads or a blocked loop. |
| `hikaricp.connections.pending` | Usually the actual constraint (Topic 109). |
| Gateway concurrency and error rate | Whether you are DDoSing your own downstream (Trap 3). |
| Catalogue p99 (an endpoint under no test load) | Blast radius. The number that tells you whether one path can hurt another. |

**Rules that make it comparable — each of these has silently invalidated a real comparison:**

1. Same dataset: 100k products, 1M orders, 5M order lines, same seed.
2. Same container CPU and memory limits. Record them in the document.
3. Same JVM flags, same collector, same JDK. Record them.
4. **Open-model arrival rate.** Not a VU loop.
5. Same warm-up discard window, long enough for C2 (Topic 74).
6. Three repetitions minimum; report the spread, not one number.
7. Verify the stack at startup in **every** run and paste the log line into the writeup.
8. Report `dropped_iterations`. A stage with dropped iterations is a stage you did not
   measure.

### Duplicate-processing counts — the correctness axis nobody benchmarks

Throughput comparisons routinely ignore correctness, and this is the one place the models
genuinely differ operationally:

```java
// Instrument both paths identically (Topics 116, 118).
Counter duplicatesSuppressed = Counter.builder("orderflow.idempotency.duplicate")
        .tag("endpoint", "orders")
        .register(registry);
```

**WHAT TO LOOK FOR:** whether the duplicate rate changes with the concurrency model.

| What you see | What it means |
|---|---|
| Duplicate suppressions rise sharply on B but not A | Higher real concurrency is exposing check-then-act races that the 200-thread cap was hiding. **Your idempotency was never correct; the thread cap was masking it.** Topic 116. |
| Duplicates appear only above 4x | The race window is narrow but real. It will find you in production during an incident, which is exactly when concurrency spikes. |
| Duplicate rate identical across all configs | Idempotency is enforced by the database, not by luck. This is what correct looks like. |

**This row belongs in your recommendation.** "Model B increased real concurrency and
surfaced a latent correctness bug in idempotency" is a genuine finding and exactly the kind
of thing that makes a document credible.

### What to graph permanently, after the decision

```java
registry.gauge("orderflow.threads.total", threadMXBean, ThreadMXBean::getThreadCount);
registry.gauge("orderflow.http.inflight", inFlight, AtomicInteger::get);
registry.gauge("orderflow.gateway.permits.available", permits, Semaphore::availablePermits);
// plus, from Boot's own instrumentation:
//   hikaricp.connections.pending, hikaricp.connections.acquire
//   jvm.threads.states, executor.*   (Topic 118)
```

**The alert that matters most after adopting virtual threads:** permit-wait time, not thread
count. **Thread count stopped being a saturation signal the moment threads became cheap**,
and a dashboard that still leads with it will mislead your on-call engineer at 3am. That is
a runbook change, and it belongs in the rollout section.

---

## Practice exercises

### 1 — easy: establish the facts for your own environment

Produce a one-page table, every row traceable to a command you ran.

1. For `/api/orders` on `orderflow`, the thread name and `isVirtual()` with the property
   off and on.
2. `server.tomcat.threads.max` from `/actuator/configprops`.
3. The carrier pool parallelism, from a thread dump, counting `ForkJoinPool-*-worker-*`.
4. Whether BlockHound installs on your JDK. Yes or no, with the error if no.
5. The `jdk.VirtualThreadPinned` event count during a 60-second baseline run.
6. The three components in your Boot version that `spring.threads.virtual.enabled` does
   **not** switch, found by logging a thread name from each.

**Done when:** every row cites a command, and row 6 has three concrete names. Row 6 is the
one that separates people who read the docs from people who checked.

### 2 — medium: demonstrate the residual

Build the smallest program that shows the thing Loom cannot do.

1. A producer that emits as fast as it can.
2. A consumer that takes 50 ms per item.
3. Version A: reactive, with `limitRate(16)` and `doOnRequest` logging at the source.
4. Version B: imperative on virtual threads, producer and consumer in separate virtual
   threads with an **unbounded** `LinkedBlockingQueue` between them.
5. Version C: version B with an `ArrayBlockingQueue(16)`.

**Capture:** for A, the `request(n)` lines at the source. For B, the queue depth over time
(and if you let it run, the heap). For C, that the producer blocks on `put`.

**Write two paragraphs answering:**
- What exactly did C achieve that B did not, and is it the same thing A achieved?
- Now add a third stage between producer and consumer. How much code changed in A? In C?
  **That delta is the composability argument, quantified in lines of code by you.**

**Done when:** you can state, in your own words and without hedging, both that C solves the
flow-control problem and that it is not compositional.

### 3 — hard: the written recommendation

**This is the deliverable. It is reviewed as a decision document, not as an essay.**

Write the recommendation for `orderflow` using the Example 2 skeleton. Constraints:

- **Maximum four pages.** A recommendation that needs ten pages has not decided anything.
- Section 1 contains an imperative sentence naming an option per path.
- Every number cites a run: date, environment, config, repetition count.
- Section 3.5 ("what we could not measure") is not optional and must be non-empty.
- Section 5 lists at least three risks **of your own recommendation**, with mitigations that
  are PRs, not intentions.
- Section 6 states the rejected options fairly enough that an advocate would recognise them.
- Section 7's revisit criteria contain numbers.

**Self-review before submission — the rubric it will be marked against:**

| Criterion | Fail | Pass | Strong |
|---|---|---|---|
| Recommendation clarity | Buried or hedged | Stated in section 1 | Stated with the single deciding fact named |
| Evidence quality | Numbers with no provenance | Environment and config recorded | Spread reported; invalid runs identified and discarded |
| Load model | Closed or unstated | Open model stated | Open model, sweep to saturation, `dropped_iterations` reported |
| Alternatives | Dismissed | Costed | Steel-manned, then rejected with a reason |
| Risks | Absent or trivial | Named with mitigations | Includes the risk you least wanted to write down |
| Migration cost | Ignored | Estimated | Estimated from a time-boxed spike |
| Revisit criteria | Absent | Qualitative | Numeric thresholds tied to existing metrics |
| Length | Over four pages | Four | Two, and complete |

**Done when:** you would sign it, and you can defend every row of Part A and Part B of the
failure drill without saying "I'd have to check".

---

## Interview questions

### Q1 — "Virtual threads make reactive obsolete. Do you agree?" `[THE STAPLE]`

This is *the* question for this topic. It is asked in almost every senior Java loop that
touches concurrency, and it is asked because it separates cleanly.

**MID-LEVEL ANSWER.** Either "Yes, virtual threads are simpler and reactive was always
overkill" or "No, reactive is still better for high load." Both are slogans. The first
misses backpressure entirely; the second cannot say what "better" means or when.

**SENIOR ANSWER.** Structured concession:

> "Largely yes — for the motivation most teams actually had. The mainstream case for
> reactive was thread economy: serving high concurrency without a thread per request.
> Virtual threads deliver that outcome and give you back the imperative model — real stack
> traces, a working debugger, `try`/`catch` over a whole request, `ThreadLocal`-based MDC
> and security context. That is a large win, because the programming model was where all
> the cost was.
>
> What Loom does **not** give you is backpressure. Reactive Streams' `request(n)` is a
> demand signal that flows upstream from consumer to producer, through arbitrary operator
> composition. There is no equivalent in Loom. I can build a bounded, blocking pipeline
> with `ArrayBlockingQueue` and that is genuinely good engineering for three stages, but it
> is plumbing I wrote, not a property of the type — it does not compose. The second thing
> reactive keeps is the stream algebra: `groupBy`, `window`, `sample`, `retryWhen`. That is
> a productivity argument rather than an impossibility one, but it is real.
>
> So my default is: request/response services go imperative on virtual threads; streaming
> pipelines with a genuinely slow consumer go reactive.
>
> And for an *existing* WebFlux codebase, none of that is usually the deciding factor.
> Migration cost is. You would be rewriting R2DBC to JDBC, reactive security to servlet
> security, and every `StepVerifier` — for a benefit you cannot put on a dashboard. I would
> only do it with a forcing function, and 'it would be simpler' is not one."

**WHAT SEPARATES THEM.** Three things, and an interviewer is listening for all three:

1. **The concession comes first.** The senior answer gives away the thread-economy point
   immediately, because it is true. Defending it marks you as out of date.
2. **The residual is named precisely** — backpressure as *upstream demand propagation*, not
   "reactive handles load better". And the senior answer says what the imperative
   alternative is (bounded queues) rather than pretending there is none.
3. **Migration cost is treated as usually decisive.** This is the judgment signal. It moves
   the answer from a technology comparison to an engineering decision.

**FOLLOW-UP: "Give me a concrete case where you would still choose reactive today."**

> "A pipeline consuming a push source into a consumer that is slower than the source, where
> dropping data is unacceptable — payment callbacks in our case. The demand signal reaches
> the source and the buffer bound is typed and explicit, so the overflow strategy is a
> product decision I made once rather than a driver setting someone can change. I could
> build it imperatively with a bounded queue, and if it were three stages I might. Past
> that, the operator algebra earns its keep."

### Q2 — "We're on MVC and hitting the thread pool ceiling. WebFlux or virtual threads?"

**MID-LEVEL ANSWER.** "WebFlux, since it's designed for high concurrency." Jumps to a
six-month migration without establishing the constraint.

**SENIOR ANSWER.**

> "First I'd verify that the thread pool is actually the ceiling, because 'thread pool
> ceiling' is a symptom people assign to a cause. `tomcat.threads.busy` pinned at
> `config.max` with CPU well below saturation supports it; `hikaricp.connections.pending`
> climbing says the pool is the real constraint and neither option helps.
>
> If it is the web tier, virtual threads first. It is one property, it keeps Hibernate, and
> I can benchmark it this afternoon. WebFlux means R2DBC, which means losing the persistence
> context, dirty checking and fetch graphs — that is months, on the path that does our
> multi-table transactional writes, which is the worst place to give up Hibernate.
>
> Two things I would do in the same change as the property. Audit for pinning under load
> with JFR, because a `synchronized` block around a blocking call makes virtual threads
> perform *worse* than platform threads. And add a bulkhead on the payment gateway, because
> the 200-thread pool was doing admission control by accident and removing it means we can
> now accept more work than we can do."

**WHAT SEPARATES THEM.** The senior answer refuses to accept the stated diagnosis, orders
the options by cost-to-try rather than by sophistication, and — most tellingly — names the
side effect of the cheap option. Mid-level answers never mention that virtual threads remove
admission control.

**FOLLOW-UP: "You added a bulkhead. How do you size it?"**

> "From the downstream's capacity, not ours. If the gateway documents 100 concurrent
> requests, that is the ceiling. If it documents nothing, I measure where its latency starts
> climbing and set it below that. Then Little's Law from Topic 90 tells me what throughput
> that permits at our service time, and I check that against our arrival rate. And I graph
> permit-wait time, because that is the leading indicator when the assumption goes stale."

### Q3 — "Walk me through what happens if someone adds a blocking JDBC call in each of the three models."

**MID-LEVEL ANSWER.** "It blocks the thread" for all three — technically true and misses
that the consequence differs by orders of magnitude.

**SENIOR ANSWER.**

> "Platform threads: one of 200 pool threads parks. Throughput drops by 1/200th of capacity
> for that path. Visible in `tomcat.threads.busy`. Recoverable, and honestly not a bug worth
> reverting on its own.
>
> Virtual threads: the virtual thread parks and the carrier is released. Cost is a
> continuation on the heap. **This is the correct thing to do** — blocking is the
> programming model. The only failure is if it happens inside a `synchronized` block on JDK
> 21–23, which pins the carrier; then a handful of those stalls the whole service because
> the carrier pool is core-count sized.
>
> Event loop: the loop thread is stuck. Every connection assigned to that loop stalls, not
> just the one request. With one loop per core, one blocking call takes out roughly 1/N of
> all connections, and if it is on a hot path it is a site-wide latency event. This is why
> BlockHound is mandatory on a reactive path and merely informative on an imperative one.
>
> That asymmetry in the *cost of a mistake* is an input to the architecture decision, and
> it usually gets left out of comparisons. Reactive requires a level of discipline from
> every engineer, including the ones who join next year, and I count that as an ongoing
> operational cost."

**WHAT SEPARATES THEM.** Quantifying the blast radius per model, naming pinning as the one
case where B behaves like C, and then converting the technical fact into a hiring and
code-review cost. The last move is the staff-level one.

**FOLLOW-UP: "How would you actually catch it in review?"**

> "BlockHound in CI on the reactive module, failing the build. Plus an alert on
> `boundedElastic` active-thread count, because the usual way a blocking call gets 'fixed'
> is by moving it to `boundedElastic`, and that is thread-per-request with extra steps.
> Review alone does not catch it — the blocking call is usually three layers down in a
> library."

### Q4 — "Your benchmark shows no difference between all four configurations. What do you report?"

**MID-LEVEL ANSWER.** "That they're all the same, so we should pick the simplest." Right
conclusion, no scepticism about the experiment.

**SENIOR ANSWER.**

> "First I would suspect the experiment. All four identical with no errors and unsaturated
> CPU almost always means I never reached saturation. Two usual causes: a closed-model load
> test, where offered load drops when the system slows down so a cliff cannot appear; and a
> sweep whose top rate is below where the thread pool binds. I would also check
> `dropped_iterations` — if the load generator saturated, every number in that stage is
> invalid.
>
> If the experiment is sound and they really are identical to 8x baseline, that is a
> genuine and valuable finding, and I would report it exactly that way: *thread count is not
> our constraint at any load we will plausibly see, therefore both the WebFlux proposal and
> the virtual-threads proposal are unjustified on performance grounds.* Then I would say
> what the constraint actually is — usually the connection pool or the database — and
> redirect the work there.
>
> I might still adopt virtual threads, but for a different reason: it costs one property
> and it raises the ceiling for free. I would say that plainly rather than dressing it up
> with numbers that do not support it."

**WHAT SEPARATES THEM.** Suspecting your own instrument first; being willing to report a
null result; and separating "adopt because measured benefit" from "adopt because cheap and
harmless", which are different arguments that people routinely blur.

**FOLLOW-UP: "The team is disappointed. How do you present it?"**

> "As a saved quarter. We were about to spend months on a migration whose premise we just
> falsified, and we now know where the actual constraint is. That is the highest-value
> outcome the benchmark could have had. I would put the constraint evidence on the first
> page, because that is the actionable part."

### Q5 — "What would make you migrate a working WebFlux service to virtual threads?"

**MID-LEVEL ANSWER.** "Simpler code and better stack traces." True, and not a business case.

**SENIOR ANSWER.**

> "A forcing function, of which I can think of three.
>
> One: a rewrite already happening for another reason. If we are splitting that service
> anyway, the marginal cost of changing the model is small and I would take it.
>
> Two: a measurable on-call cost. If I can show that reactive-specific debugging —
> subscription stacks, lost MDC, `checkpoint()` archaeology — accounts for a real number of
> incident minutes per quarter, that is a benefit with a unit. Topic 108 gives me the
> mechanisms; the incident record gives me the number.
>
> Three: a hiring or team-composition constraint I can evidence, not assert. 'Reactive is
> hard' is not evidence. 'Two of our five engineers can safely modify this service, and the
> bus factor is a stated risk in our last two postmortems' is.
>
> What would *not* move me: that it would be simpler, that virtual threads are newer, or a
> benchmark showing parity. Parity is the expected result and it is an argument for doing
> nothing.
>
> And if we did migrate, I would keep the parts that use real backpressure reactive. It is
> not an all-or-nothing decision — the boundary is per-path, and the seams are exactly where
> I would put the effort."

**WHAT SEPARATES THEM.** Requiring a forcing function; converting a diffuse benefit into a
measurable one; explicitly naming what would *not* move them; and refusing the false binary
of migrating everything or nothing.

---

## Mental model checkpoint

Answer without scrolling up. If any answer takes more than two sentences, re-read the
Machine-level reality section.

1. **State the difference between Loom and reactive in one sentence about where the
   continuation lives.**
   Both put the rest of the computation on the heap; Loom puts it there as a suspended
   *stack* the JVM copies generically, reactive puts it there as an operator chain *you*
   decomposed by hand.

2. **What exactly does Loom not give you, and what is the imperative substitute?**
   Composable backpressure — `request(n)` propagating upstream through arbitrary
   composition. The substitute is bounded blocking queues between stages: correct, and not
   compositional.

3. **A blocking JDBC call lands on each of a platform thread, a virtual thread, and an
   event loop. Rank the consequences.**
   Event loop is catastrophic (all connections on that loop). Platform thread costs one of
   N pool threads. Virtual thread costs a continuation and is the *intended* behaviour —
   unless it is inside `synchronized` on JDK 21–23, which pins the carrier and makes it
   behave like the event-loop case.

4. **Why does switching to virtual threads make an unrelated downstream fall over?**
   The bounded thread pool was doing admission control by accident. Removing the bound
   removes the ceiling on concurrent downstream calls. You gained the ability to accept work
   you cannot do, not capacity.

5. **Your comparison shows all configurations identical. What is your first hypothesis?**
   The experiment is wrong: closed-model load, or a sweep that never reached saturation.
   Check `dropped_iterations` before believing any of it.

6. **What is the usually-decisive factor in whether to migrate an existing WebFlux service,
   and why is it not on the benchmark?**
   Migration cost — R2DBC, security, tests, retraining, and the risk of a half-migrated
   codebase. The benchmark measures the destination, not the journey.

7. **Which two metrics replace "thread count" as saturation signals after adopting virtual
   threads?**
   Bulkhead permit-wait time, and `hikaricp.connections.pending`. Thread count stops being
   meaningful the moment threads become cheap, and a dashboard that leads with it will
   mislead your on-call.

---

## Quick reference card

### The decision in five lines

```
Request/response, client waiting            -> imperative on virtual threads
Streaming, slow consumer, cannot drop        -> reactive
Existing working WebFlux service             -> keep it, unless a forcing function
Existing MVC hitting the thread ceiling      -> virtual threads first; one property
Anything at all                              -> establish the constraint BEFORE choosing
```

### What each model gives and takes

| | Virtual threads | Reactive |
|---|---|---|
| Programming model | Unchanged, imperative | Operator chains |
| Stack traces | Real | Subscription stack (Topic 108) |
| Debugger | Works | Framework internals |
| `ThreadLocal` / MDC | Works | Broken; needs `Context` |
| Hibernate | Works | Not available; R2DBC |
| Backpressure | **None** — build bounded queues | **Composable `request(n)`** |
| Stream algebra | Hand-rolled | `groupBy`/`window`/`sample`/`retryWhen` |
| Cost of a stray blocking call | A continuation | **A stalled event loop** |
| Cost of adoption in an MVC app | One property + a pinning audit + a bulkhead | Months, and Hibernate |

### Commands you will actually run

```bash
java -Dspring.threads.virtual.enabled=true -jar orderflow.jar
jcmd <pid> Thread.dump_to_file -format=json -overwrite dump.json
jcmd <pid> JFR.start name=pin settings=profile filename=pin.jfr
jfr print --events jdk.VirtualThreadPinned pin.jfr | head -80
java -Djdk.tracePinnedThreads=full -jar orderflow.jar     # JDK 21-23 ONLY
curl -s localhost:8080/actuator/metrics/tomcat.threads.busy
curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending
mvn -q dependency:tree | grep -E "starter-web(flux)?"
```

### The interview answer, compressed

```
Concede:  thread economy -- virtual threads win, and it is not close.
Keep:     backpressure (upstream request(n), composable) and stream algebra.
Default:  request/response -> virtual threads. Streaming with demand -> reactive.
Decide:   migration cost usually beats technical merit for an existing codebase.
Warn:     virtual threads remove the accidental admission control. Bulkhead it.
```

### Things that invalidate a comparison

```
closed-model load test          -> no saturation cliff can appear
sweep top below the pool bound  -> nothing to see between A and B
dropped_iterations > 0          -> the generator saturated, not the service
one repetition                  -> cannot distinguish a 10% effect from noise
different JDKs across configs   -> pinning behaviour changed in 24
both web starters on classpath  -> you benchmarked a servlet app either way
correctness not measured        -> higher concurrency may be exposing real bugs
```

### Gotchas checklist

```
[ ] Pinning audited under LOAD, with JFR, not at idle
[ ] Bulkhead added in the same PR as the virtual-thread property
[ ] tryAcquire with timeout, never unbounded acquire()
[ ] Dashboards updated: thread count is no longer a saturation signal
[ ] JDK of the benchmark == JDK of production, or the difference is stated
[ ] Duplicate-processing counters read at every arrival rate, not just 1x
[ ] The status-quo option is costed, not dismissed
[ ] Revisit criteria contain numbers
```

---

## When would I use this at work?

**1. Killing a reactive migration proposal — or funding one — with one afternoon of work.**

Someone proposes WebFlux to fix a thread ceiling. You set one property, re-run the load
harness, and either the ceiling moves (proposal unjustified; you saved a quarter) or it
does not (the constraint is elsewhere; the proposal was mis-aimed and you saved a quarter a
different way). This is the highest return-on-effort action in the entire curriculum, and
it works because virtually nobody runs the cheap option before proposing the expensive one.

**2. Reviewing the PR that turns on virtual threads.**

One mechanical question: **what used to bound concurrency, and what bounds it now?** If the
answer is "nothing", the PR is incomplete regardless of how well it benchmarks. Ask for the
bulkhead, the pinning audit under load, and the dashboard change. This single question will
make you the most useful reviewer on that PR, and it is the thing teams most reliably miss.

**3. Being the third person in the "is reactive dead" argument.**

There is a person saying reactive is obsolete and a person saying virtual threads are
hype. Both are partly right and neither can say where the line is. Being able to say —
calmly, in one minute — that thread economy went to Loom, that backpressure did not, that
`orderflow` has one path of each kind, and that migration cost usually decides it anyway, is
what senior technical judgment sounds like out loud. It is also, almost verbatim, the answer
to the interview question you will be asked.

---

## Connected topics

**Prerequisites — this document is not writable without these:**

- **101 — Virtual threads:** the continuation mechanism, the pinning drill, and the
  before/after baseline that is column B of every table here. **Run its drill before
  writing anything.**
- **105 — Backpressure:** the residual capability this entire decision turns on. The OOM
  drill is what makes "Loom has no backpressure" a fact you have seen rather than a claim
  you repeat.
- **106 — WebFlux vs MVC:** the four-configuration comparison table, consumed directly.
  Without it you have opinions.
- **90 — Executors and pool sizing:** Little's Law, and why a bounded pool is accidental
  admission control.
- **93 — Bounded queues:** the imperative substitute for backpressure, and the honest
  alternative you must steel-man in the recommendation.

**Also relevant:**

- **48–53 — Hibernate:** six topics of capability that a WebFlux/R2DBC path gives up. The
  largest single line item in the migration cost.
- **65 — The load baseline:** the only valid instrument for this question. JMH is wrong here.
- **74 — JIT and tiered compilation:** why the warm-up discard window has to be long enough
  to matter.
- **77 — JMH:** the standing measurement rule, and the reason a `nanoTime` loop proves
  nothing.
- **91 — `CompletableFuture`:** the async model that preceded both, and why "always pass an
  explicit executor" is the same lesson as "always bound your concurrency".
- **102 — Structured concurrency and `ScopedValue`:** where imperative concurrency goes next,
  and the `ThreadLocal` replacement that makes virtual threads cheaper per-thread.
- **103–104 — Netty and Reactor:** the machinery under option C.
- **109 — HikariCP:** the constraint that is usually the real answer, whichever model wins.
- **111 — Resilience4j:** the bulkhead and rate limiter that re-create the ceiling you
  removed.
- **116 — Idempotency:** the correctness axis of the comparison. Higher real concurrency
  surfaces check-then-act races the thread cap was hiding.

**This unlocks:**

- **108 — Reactive debugging:** the observability tax on option C, quantified. Read it
  before claiming reactive's operational cost is small.
- **118 — Metrics:** the gauges and alerts your recommendation's rollout section commits to.
- **124 — The readiness gate:** two models means two readiness stories, and a bulkhead that
  is saturated is a readiness question, not just a metric.
- **129 — Capacity and latency budgets:** where the arrival rates in your sweep should have
  come from in the first place.
- **131 — Design-doc authorship:** this document's hard exercise is the rehearsal. Topic 131
  is where the form is taught properly, and where you will rewrite this recommendation and
  find it two pages too long.

---

*Java baseline 21, deployed on JDK 25. Reactor, Netty and Tomcat versions come from the
Spring Boot BOM; do not pin them by hand and do not trust a version number quoted in any
document, including this one. Two version-dependent facts will change under you and must be
re-verified rather than remembered: the set of components `spring.threads.virtual.enabled`
actually switches, which has grown across Boot versions, and the `synchronized` pinning
behaviour, which changed materially in JDK 24 and makes any pinning measurement taken on
21–23 non-transferable. Where an operator name or API shape in this document is uncertain,
it is because Reactor's surface is large and moves; check the current Reactor reference
documentation rather than this file. The one structural fact worth committing to memory is
the mechanical statement: both models put the continuation on the heap, only reactive gets
backpressure for it, and migration cost usually decides the rest.*
