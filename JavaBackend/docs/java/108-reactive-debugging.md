# 108 — Debugging Reactive: Subscription Stacks, Checkpoints, and Context Propagation

## Phase: 10 — Reactive & Async at Scale
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21. Reactor, `context-propagation` and Micrometer versions come from the Spring Boot BOM. Do not pin them by hand and do not quote a version number from this document.
## Project spine: make `orderflow`'s payment-callback ingestion debuggable — every log line on the reactive path carries the correlation ID, and every failure names the line of code that assembled the failing operator.

---

## Before anything else — what is and is not in this document

**I have no JVM, no running `orderflow`, no log aggregator and no Reactor on a classpath.
Nothing in this document is captured tool output.**

You will **not** find here:

- a real stack trace copied from a run,
- a real thread name, a real correlation ID, a real timestamp,
- a latency or overhead figure presented as measured,
- "`Hooks.onOperatorDebug()` costs 30% throughput" or anything shaped like a result,
- a claim that a specific line number appeared in a specific frame.

You get instead, everywhere:

- **the exact code, command or config**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see → what it means"** table covering the plausible outcomes, including the
  ones that mean your instrument is broken rather than your code.

### The labelled exceptions — and why they are unavoidable here

This topic is *about reading output*. You cannot learn to read the shape of a reactive stack
trace without seeing the shape. So in five places I show **structure only** — frame
ordering, section headers, the position of the assembly-trace block — with every identifier,
number and path replaced by a placeholder. Each carries the inline label:

> *illustration of the format, not captured output*

Placeholders are `<Class>`, `<method>`, `<n>`, `<file>`, `<corr-id>`, `xxx`. **If you find a
plausible-looking value anywhere in this document, treat it as a bug in the document.**

### Spec-level facts I state plainly, each with a confirming command

1. **A reactive stack trace shows the subscription path, not the assembly path.** Confirm:
   Proof 1 — assemble a chain in one method, subscribe in another, and observe that the
   assembling method does not appear.
2. **Reactor's `Context` is carried in the subscription and propagates from the subscriber
   *upstream*.** Confirm: Proof 4 — put `contextWrite` above a `deferContextual` and observe
   the value is missing; move it below and observe it appear.
3. **`ThreadLocal` cannot survive an operator thread hop.** Confirm: the failure drill —
   `publishOn` between an MDC put and an MDC read, and watch the field vanish.
4. **`checkpoint()` captures assembly information only at the points you place it;
   `Hooks.onOperatorDebug()` captures it at every operator in the JVM.** Confirm: Proof 3 —
   compare the two failure outputs from the same broken chain.

### THE RULE

> **If your own output disagrees with anything here, YOUR OUTPUT IS THE TRUTH.** Reactor's
> debug surface has moved across versions — automatic context propagation in particular. The
> mechanism is stable; the API names are not. Where I am uncertain about a current API shape
> I say so in one line and point at the Reactor reference documentation rather than
> guessing.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **A reactive stack trace shows the SUBSCRIPTION stack, not the ASSEMBLY stack. It tells
> you where the failing operator RAN, not where you WROTE it.**
>
> **`checkpoint("name")` inserts assembly information at chosen points, cheaply.**
> **`Hooks.onOperatorDebug()` inserts it everywhere, expensively.**
>
> **And the same root cause explains lost MDC: the chain hops threads between operators, so
> a `ThreadLocal` set on one thread is not visible on the next. Reactor therefore carries
> its own `Context` inside the subscription instead.**

Both halves of this topic — unreadable traces and vanished correlation IDs — are the same
fact wearing two hats:

> **In an imperative program, the thread's stack is both your call history and your ambient
> context store. Reactive throws the stack away between operators. So you lose the call
> history (bad traces) and you lose the ambient context (`ThreadLocal`/MDC) at the same
> moment, for the same reason.**

If you can say that sentence unprompted, you have the topic.

---

## The bridge from what you know

### You have met the trace problem, and Node fixed it for you

In early Node, an error thrown inside a callback produced a stack that started at the event
loop and told you nothing about who scheduled the work. You know the feeling:

> *illustration of the format, not captured output*
> ```
> Error: <message>
>     at <anonymous> (<file>:<n>)
>     at process.processTicksAndRejections (node:internal/process/task_queues:<n>)
> ```

Node then spent years fixing this *for you*, invisibly:

- `--async-stack-traces` (on by default for `async`/`await` since Node 12) stitches the
  awaiting frames back on;
- V8 tracks async frames through promises;
- `AsyncLocalStorage` gives you request-scoped ambient state that survives `await`.

**Java did not get any of that for free, and Reactor cannot get it for free**, because a
Reactor chain is not built from language-level `await` points the runtime can instrument. It
is built from library objects you composed. So Reactor gives you *opt-in* equivalents, and
this document is about using them deliberately.

| Node | Reactor | Free? |
|---|---|---|
| `--async-stack-traces` | `Hooks.onOperatorDebug()` or `ReactorDebugAgent` | **No.** Opt-in, with cost. |
| Stack frames across `await` | `checkpoint("name")` at chosen boundaries | **No.** You place them. |
| `AsyncLocalStorage` | Reactor `Context` / `ContextView` | **No.** You write and read it explicitly. |
| `AsyncLocalStorage` auto-propagating into `await` | `Hooks.enableAutomaticContextPropagation()` + `ThreadLocalAccessor` | Version-dependent; verify. |

### The one that will actually bite you: `AsyncLocalStorage` ≈ `Context`, but MDC ≠ either

You are used to `AsyncLocalStorage` making a request ID available anywhere downstream in an
`async` call tree, and to a Nest interceptor or pino child logger picking it up
automatically.

**Java's equivalent is MDC (Topic 120), and MDC is `ThreadLocal`-backed.** That is the crux:

- On Spring MVC (thread-per-request), MDC works perfectly. One thread, whole request. This
  is why every Java logging tutorial says "just use MDC" and never mentions a caveat.
- On virtual threads (Topic 101), MDC still works — a virtual thread is a `Thread` and has
  its own thread-locals. It costs more memory per thread, but semantically it is fine.
- **On a reactive chain, MDC breaks**, because between `map` and `flatMap` your code may
  move to a different `reactor-http-nio-*` or `boundedElastic-*` thread, and the MDC map
  lives on the thread you left.

**And the failure is silent.** MDC does not throw when a key is absent. `%X{correlationId}`
in a Logback pattern renders as empty. So the symptom is not an error — it is log lines that
are *missing a field*, which nobody notices until an incident when you try to grep for a
request ID and half the trail is gone.

### The RxJS instinct that transfers, and the one that misleads

**Transfers:** you already accept that operators can run on different schedulers, and you
already know `subscribeOn` affects the source while `observeOn`/`publishOn` affects
everything downstream of it. That mental model is correct in Reactor.

**Misleads:** in RxJS you do not have thread-locals, so you never developed the habit of
reaching for ambient context. In Java you have — through six phases of Spring, MDC and
`SecurityContextHolder` — and that habit is a trap here. **Anything you were getting from a
`ThreadLocal` needs an explicit plan on a reactive path**: correlation IDs (Topic 120),
`SecurityContextHolder` (Topic 56), tracing spans (Topic 119), and the tenant context from
Topic 38.

**Verdict: STRONG ANALOGUE for the problem — you have lived both halves in Node. NO
ANALOGUE for the fix being manual, because Node's runtime did it for you and Reactor's
cannot.**

---

## What is this?

Four distinct instruments, commonly confused. Know which question each one answers.

| Instrument | Answers | Cost | Where it lives |
|---|---|---|---|
| **`.log()`** | "What signals flow through this point, in what order, on which thread?" | Per-signal logging. Fine in dev, noisy in prod. | One operator position |
| **`.checkpoint("name")`** | "Which of my chains failed?" | Captures one lightweight assembly marker at that point. | Chosen boundaries |
| **`Hooks.onOperatorDebug()`** | "Which *line* assembled the failing operator, anywhere?" | **Captures a stack trace at every operator assembly, JVM-wide.** Heavy. | Global, at startup |
| **`ReactorDebugAgent`** (from `reactor-tools`) | Same as above | Bytecode instrumentation at class load; substantially cheaper than the `Hooks` version | Global, at startup |
| **`Context` / `ContextView`** | "What request-scoped data is available here?" | An immutable map carried in the subscription | The chain itself |
| **`ContextSnapshot`** (Micrometer `context-propagation`) | "How do I bridge `ThreadLocal` and `Context` at the boundary?" | Capture/restore at chosen points | Boundaries with non-reactive code |
| **BlockHound** | "Is anything blocking a non-blocking thread?" | Instrumentation agent; test/dev | JVM-wide, in tests |

### The two problems, named

**Problem 1 — the trace names Reactor, not you.**

When an operator fails, the exception propagates as `onError` down the subscriber chain. The
JVM stack at that moment is whatever thread was executing the operator — typically an
event-loop or scheduler thread that entered the chain at `subscribe()`. **Your assembly
code — the method where you wrote `.flatMap(...)` — ran earlier, on a different stack, and
that stack is long gone.**

**Problem 2 — the correlation ID vanishes.**

Between two operators the execution may move threads. MDC is a `ThreadLocal`. Therefore the
MDC contents do not move. Every log statement after the hop renders the correlation ID as
empty, and the failure is silent.

### `[BOOT 3.x DELTA]`

The mechanism is identical on Boot 3.x. Two practical differences:

- **Automatic context propagation** (`Hooks.enableAutomaticContextPropagation()`, and the
  Micrometer `context-propagation` library with `ThreadLocalAccessor`) arrived and then
  changed defaults across the Reactor 3.5 → 3.6 → later line. **Whether it is on by default
  in your build is a thing to verify, not remember.** Print it: register a
  `ThreadLocalAccessor`, run the drill, and see whether the value appears without you
  calling `ContextSnapshot` yourself.
- On older 3.x builds you are more likely to need the explicit
  `ContextSnapshotFactory`/`ContextSnapshot` calls at every boundary. The explicit form
  works everywhere and is what this document teaches, precisely because it does not depend
  on a version-dependent default.

---

## Why does it matter?

**1. It is the true operational cost of reactive, and Topic 107's decision needs it
priced.** "Reactive is harder to debug" is a vague complaint until you can say exactly
what breaks (assembly information, `ThreadLocal` context), exactly what it costs to fix
(checkpoints at boundaries, an explicit context plan, a debug agent in non-prod), and
exactly what residual cost remains (a debugger that steps into `FluxFlatMap`). That
pricing is an input to Topic 107's recommendation and to Topic 131's design docs.

**2. A missing correlation ID makes an incident unresolvable.** Not "harder" —
unresolvable. If a payment callback fails for one merchant and the log lines from the
failing path have no correlation ID, you cannot join them to the request, to the trace
(Topic 119), or to the order. You have a pile of unrelated lines. Every distributed-systems
skill you have depends on being able to follow one request through the system, and this is
the mechanism by which that ability is silently lost.

**3. `Hooks.onOperatorDebug()` in production is a classic self-inflicted outage.** Someone
enables it during an incident to get better traces, it works, and it never gets turned off.
Every operator assembly now captures a stack trace — an allocation and a stack walk per
operator per request. Throughput falls, and the cause is a debugging aid nobody remembers
enabling.

---

## Machine-level reality

### 1. Assembly time vs subscription time — the whole trace problem in one diagram

Topic 104 established this; here is why it determines what your stack trace can possibly
contain.

```java
// ---- ASSEMBLY TIME: this runs when the method is called. ----
// Each operator call CONSTRUCTS an object and returns it. Nothing executes.
Mono<Receipt> chain = callbacks.load(id)      // -> MonoFromPublisher
        .map(this::validate)                  // -> MonoMap wrapping the above
        .flatMap(gateway::confirm)            // -> MonoFlatMap wrapping the above
        .timeout(Duration.ofSeconds(2));      // -> MonoTimeout wrapping the above

// ---- SUBSCRIPTION TIME: this is when anything runs. ----
chain.subscribe(...);   // walks the object graph, builds a Subscriber chain, requests
```

At assembly time you have a **tree of operator objects**. The stack that built that tree —
the method with your `.flatMap(...)` in it — returns immediately and is discarded.

At subscription time, some thread calls `subscribe()`, which builds the subscriber chain
downward and then data flows. **When `validate` throws, the JVM stack contains:**

- your `validate` method,
- `MonoMap$MapSubscriber.onNext`,
- whatever called `onNext` — another operator's subscriber,
- ... down to the thing that started the emission: an event-loop read, a scheduler task, or
  `subscribe()` itself.

**Your assembling method is not on that stack, because it finished long ago.** This is not a
Reactor deficiency; it is arithmetic. The information genuinely does not exist at failure
time unless something captured it at assembly time.

### 2. The shape of an undecorated reactive stack trace

> *illustration of the format, not captured output*

```
<ExceptionType>: <message>
    at com.orderflow.payments.<Class>.<method>(<file>:<n>)          <-- YOUR failing code
    at reactor.core.publisher.<OperatorSubscriber>.onNext(<file>:<n>)
    at reactor.core.publisher.<OperatorSubscriber>.onNext(<file>:<n>)
    at reactor.core.publisher.<OperatorSubscriber>.onNext(<file>:<n>)
    at reactor.core.publisher.<Operator>.subscribe(<file>:<n>)
    at reactor.core.publisher.<Operator>.subscribe(<file>:<n>)
    at reactor.netty.<...>(<file>:<n>)
    at io.netty.<...>(<file>:<n>)
    at java.base/java.lang.Thread.run(Thread.java:<n>)
```

**Read the structure, not the values.** Three regions:

1. **The top frame is genuinely yours** — the lambda or method that threw. This is why
   people say the trace is "useless" and are overstating it: you do get the throw site.
2. **The middle is a wall of Reactor internals.** These frames tell you the operator
   *kinds* in the chain and nothing about which chain, in which service, assembled by which
   method.
3. **The bottom is the subscription origin** — Netty, a scheduler, or `block()`.

**What is missing and what you actually needed:** *which of the eleven places in this
codebase that build a similar chain is this one?* When `validate` is called from four
different pipelines, the top frame does not disambiguate. That is the practical loss.

### 3. What `checkpoint()` does, mechanically

`.checkpoint("description")` inserts an operator that, on error, attaches a **suppressed
exception** carrying the description and the assembly location of that checkpoint.

> *illustration of the format, not captured output*

```
<ExceptionType>: <message>
    at com.orderflow.payments.<Class>.<method>(<file>:<n>)
    ... reactor frames ...
    Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException:
    Error has been observed at the following site(s):
        *__checkpoint <-- <your description string>
    Original Stack Trace:
        ... (the same frames as above) ...
```

Two variants, and the difference matters:

```java
.checkpoint("payment-callback-confirm")        // description only: NO stack capture. Cheap.
.checkpoint("payment-callback-confirm", true)  // forceStackTrace: also captures assembly stack.
.checkpoint()                                  // no description: captures assembly stack. Avoid.
```

**The cheap and correct default is the description-only form.** It costs a string reference
per assembly and gives you the single piece of information you actually lacked: *which
chain*. The stack-capturing forms cost a `fillInStackTrace` at assembly.

**A checkpoint marks the chain from its position upward.** Multiple checkpoints accumulate,
oldest-upstream first, giving you a breadcrumb path — which is why placing them at
*boundaries* (per public method, per pipeline stage) is more useful than sprinkling them.

### 4. What `Hooks.onOperatorDebug()` does, and why it is expensive

```java
Hooks.onOperatorDebug();   // call once, at startup, before any chain is assembled
```

This installs a global assembly hook. **Every operator constructed anywhere in the JVM from
that point on captures its assembly stack trace.** On failure, Reactor reconstructs a
readable "assembly trace" naming the exact lines that built the chain.

The cost is where you would expect:

- **`fillInStackTrace()` per operator, per assembly.** For a cold `Mono` assembled per
  request with ten operators, that is ten stack walks per request. Stack-walking cost scales
  with stack depth, and a Spring request stack is deep.
- **Allocation per operator** to retain those traces, which is GC pressure proportional to
  request rate (Topics 68, 70).
- **It is global.** It does not respect package boundaries. Library-internal chains pay too.

**Therefore:** development and test, yes. Production, no — with one narrow exception noted
in Trap 3.

**The cheaper global option:** `reactor-tools`' `ReactorDebugAgent`.

```java
public static void main(String[] args) {
    ReactorDebugAgent.init();      // BEFORE the Spring context starts
    SpringApplication.run(OrderflowApplication.class, args);
}
```

It instruments Reactor classes at load time to record assembly locations without a runtime
`fillInStackTrace` per operator. It is materially cheaper than `Hooks.onOperatorDebug()`.
**Whether it is cheap enough for your production is a measurement, not a belief** — and the
Measurement section tells you how to take it. Note also that it must run before the classes
it instruments are loaded, which in practice means first thing in `main`.

### 5. The `Context` — carried in the subscription, not in a `ThreadLocal`

This is the structural fix, and the direction of propagation is the thing everyone gets
wrong.

**`Context` is an immutable key-value map attached to the `Subscription`.** When you call
`subscribe()`, the subscriber's context is passed *up* the chain as each operator subscribes
to its upstream. So:

```java
Mono.deferContextual(ctx -> {                     // (1) reads the context
        String corr = ctx.getOrDefault("correlationId", "none");
        return callbacks.load(id).map(c -> tag(c, corr));
    })
    .contextWrite(Context.of("correlationId", incomingId));   // (2) writes it
```

**(2) is written *after* (1) in the fluent chain, and that is correct.** The context flows
from the bottom of the chain upward at subscription time, so a `contextWrite` affects
everything **above** it (upstream of it), not below it.

Say it as a rule, because it is the single most common `Context` bug:

> **`contextWrite` must be placed DOWNSTREAM of every operator that needs to read the
> value. Writing the context at the top of the chain does nothing for the operators below
> it.**

Consequences:

| Property | `ThreadLocal` / MDC | Reactor `Context` |
|---|---|---|
| Where it lives | The thread | The subscription |
| Survives a thread hop | **No** | **Yes** |
| Direction of visibility | Whole thread, both directions | Upstream from the `contextWrite` |
| Mutability | Mutable in place | **Immutable**; each write produces a new context |
| Read by a logging framework automatically | Yes (`%X{key}`) | **No** — you must bridge it |
| Available inside a plain lambda | Yes | Only via `deferContextual` / `transformDeferredContextual` / `doOnEach` |

**That last row is the operational sting.** Your Logback pattern reads MDC. `Context` is not
MDC. So even with a perfect `Context`, your log lines are still missing the field until you
bridge the two — which is what the drill does.

### 6. `ContextSnapshot` — the bridge, and where it must sit

The Micrometer `context-propagation` library defines two things:

- **`ThreadLocalAccessor<T>`** — a registered adapter that knows how to get/set/clear one
  particular `ThreadLocal` (MDC entry, tracing span, security context) under a key.
- **`ContextSnapshot`** — a capture of all registered thread-locals, restorable elsewhere,
  and convertible to/from a Reactor `Context`.

The shape of use, at a boundary:

```java
// Capture on the way in (a thread that HAS the ThreadLocal set):
ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

// Restore on the way out (a thread that does NOT):
try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
    log.info("this line now has the MDC fields");
}
```

**API-shape caveat, stated once and honestly:** the exact factory/builder surface of
`context-propagation` and of `Hooks.enableAutomaticContextPropagation()` has changed across
versions, and Boot's BOM decides which you get. **Verify the current signatures against the
Reactor reference documentation and the `context-propagation` README for your BOM version
rather than trusting the shapes above.** The *mechanism* — a registered accessor plus a
capture/restore scope — is stable; the method names are the part that moves.

**Where the bridge must sit:** at every place where a reactive thread runs code that logs.
In practice that means `doOnEach` (which gives you the `ContextView` alongside the signal),
or automatic propagation if your version supports it. Putting it only at the entry point
does nothing, because the entry point is not where the thread hop happens.

---

## Example 1 — minimal

### 1a — prove the thread hop with your own eyes

The instrument for everything in this document is one line: print the thread name inside
operators.

```java
Mono.just("callback-1")
    .doOnNext(v -> log.info("A: {} on {}", v, Thread.currentThread().getName()))
    .publishOn(Schedulers.boundedElastic())
    .doOnNext(v -> log.info("B: {} on {}", v, Thread.currentThread().getName()))
    .flatMap(v -> Mono.just(v)
            .subscribeOn(Schedulers.parallel())
            .doOnNext(x -> log.info("C: {} on {}", x, Thread.currentThread().getName())))
    .doOnNext(v -> log.info("D: {} on {}", v, Thread.currentThread().getName()))
    .block();
```

**WHAT TO LOOK FOR:** whether A, B, C and D print the same thread name.

| What you see | What it means |
|---|---|
| A on the calling thread, B/D on `boundedElastic-*`, C on `parallel-*` | Normal. **Three different threads in one chain** — this is why `ThreadLocal` cannot work. |
| All four the same | You are on a chain with no scheduler hop *in this run*. Do not conclude there never is one — a `flatMap` over a network call will hop as soon as the response arrives on an I/O thread. |
| D on `parallel-*` rather than `boundedElastic-*` | Also normal: after a `flatMap`, downstream continues on whichever thread delivered the inner completion. **This is the detail that makes hops unpredictable and therefore un-plannable by hand.** |

**The point of the third row:** you cannot reason your way to where a value will be. That is
why the fix is structural (`Context`) rather than careful placement of MDC calls.

### 1b — the stack trace, undecorated and then checkpointed

```java
class MinimalTraceDemo {

    Mono<String> buildChain(String input) {          // ASSEMBLY happens here
        return Mono.just(input)
                .map(this::normalise)
                .map(this::mustNotBeBlank)           // throws for blank input
                .map(String::toUpperCase);
    }

    private String mustNotBeBlank(String s) {
        if (s.isBlank()) throw new IllegalArgumentException("blank callback reference");
        return s;
    }

    void run() {                                     // SUBSCRIPTION happens here
        buildChain("   ").block();
    }
}
```

**WHAT TO LOOK FOR in the resulting trace:** the presence of `mustNotBeBlank` (yes) and the
presence of `buildChain` (no).

Now add one operator:

```java
    Mono<String> buildChain(String input) {
        return Mono.just(input)
                .map(this::normalise)
                .map(this::mustNotBeBlank)
                .map(String::toUpperCase)
                .checkpoint("MinimalTraceDemo.buildChain");   // <-- description only
    }
```

| What you see | What it means |
|---|---|
| A `Suppressed: ...OnAssemblyException` block naming your description | Working. You now know *which* chain failed. |
| No suppressed block at all | The error did not pass through the checkpoint — it happened downstream of it, or the chain you checkpointed is not the one that failed. |
| The description appears but you still cannot find the line | Expected with the description-only form. That is the trade. Use `checkpoint("name", true)` on that one chain temporarily if you must have the line. |
| Several `*__checkpoint` lines | Multiple checkpoints on the path. Read them as a breadcrumb trail from the outermost inward. |

### 1c — lose the correlation ID in six lines

```java
MDC.put("correlationId", "corr-from-caller");
Mono.just("payload")
    .doOnNext(v -> log.info("before hop"))       // MDC present
    .publishOn(Schedulers.boundedElastic())
    .doOnNext(v -> log.info("after hop"))        // MDC EMPTY
    .block();
MDC.clear();
```

With a Logback pattern containing `%X{correlationId}`:

> *illustration of the format, not captured output*
> ```
> <ts> INFO  [<thread-a>] [<corr-id>] c.o.p.Demo - before hop
> <ts> INFO  [<thread-b>] []          c.o.p.Demo - after hop
> ```

**That empty pair of brackets is the entire bug**, and note that nothing threw. This is the
shape you are going to reproduce properly in the failure drill.

---

## Example 2 — production scenario (on the project spine)

### The situation

`orderflow`'s payment-callback ingestion (Topics 105–106) is a WebFlux module. A gateway
posts callbacks; the module validates, deduplicates against the inbox, and updates the
order. Under the Topic 65 load profile, two things are true and both are unacceptable:

1. When a callback fails, the log shows a stack trace whose only application frame is a
   validation method used by four different pipelines. Nobody can tell which pipeline.
2. Log lines after the first `flatMap` have no correlation ID, so a failing callback cannot
   be joined to the incoming request, the order, or the distributed trace (Topic 119).

### Step 1 — establish the correlation ID at the edge, into the `Context`

```java
@Component
class CorrelationWebFilter implements WebFilter {

    static final String CORRELATION_KEY = "correlationId";
    private static final String HEADER = "X-Correlation-Id";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders().getFirst(HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }
        exchange.getResponse().getHeaders().add(HEADER, correlationId);

        String finalId = correlationId;
        return chain.filter(exchange)
                // contextWrite is placed here, at the BOTTOM, so it is visible to the
                // entire chain above it -- i.e. the whole request handling. This
                // placement is the part people get wrong.
                .contextWrite(Context.of(CORRELATION_KEY, finalId));
    }
}
```

**`[BOOT 3.x DELTA]`** — identical on 3.x. `WebFilter` is the reactive analogue of a servlet
`Filter`; on the MVC side of `orderflow` the same job is done by a `OncePerRequestFilter`
with `MDC.put`/`MDC.remove` in a `finally` (Topic 120).

### Step 2 — make the logger see it

The `Context` is not MDC, so `%X{correlationId}` is still empty. Two ways to bridge, and you
should know both because the second is version-dependent.

**Option A — explicit, works on every version: log through `doOnEach`.**

```java
public final class ReactiveLogging {

    private ReactiveLogging() {}

    /** Logs on every next/error/complete signal WITH the context's MDC applied. */
    public static <T> Consumer<Signal<T>> logOnNext(Consumer<T> logStatement) {
        return signal -> {
            if (!signal.isOnNext()) return;
            String correlationId = signal.getContextView()
                    .getOrDefault(CorrelationWebFilter.CORRELATION_KEY, "none");
            try (MDC.MDCCloseable ignored = MDC.putCloseable("correlationId", correlationId)) {
                logStatement.accept(signal.get());
            }
        };
    }
}
```

Used at each place you log:

```java
.doOnEach(ReactiveLogging.logOnNext(cb ->
        log.info("validated callback ref={} merchant={}", cb.reference(), cb.merchantId())))
```

Verbose, explicit, and **it works regardless of Reactor version**, which is why it is the
form to learn. `doOnEach` is the only `doOn*` variant that hands you the `ContextView`, and
that is precisely why it exists.

**Option B — automatic propagation, if your version supports it.**

```java
@Configuration
class ContextPropagationConfig {

    @PostConstruct
    void enable() {
        // Registers an accessor so the MDC entry is captured/restored automatically
        // at operator boundaries.
        ContextRegistry.getInstance()
                .registerThreadLocalAccessor(new CorrelationIdThreadLocalAccessor());
        Hooks.enableAutomaticContextPropagation();
    }
}
```

> **Uncertainty, stated plainly:** the exact registration API, the interface shape of
> `ThreadLocalAccessor`, and whether `enableAutomaticContextPropagation()` is required or
> already the default all depend on the Reactor and `context-propagation` versions your Boot
> BOM selects. **Verify against the current Reactor reference documentation ("Context
> Propagation") and the `micrometer-context-propagation` README before writing this code.**
> Do not copy the shape above into production without checking it compiles against your BOM.
> The mechanism — register an accessor, then boundaries capture and restore — is stable.

**How to tell whether Option B is actually working:** delete your `doOnEach` bridging from
one endpoint, run the drill, and see whether the field survives. If it does, automatic
propagation is on and working. **If you cannot prove it, keep Option A.** A silent
correlation-ID loss is worse than verbose code.

### Step 3 — checkpoints at the boundaries that matter

```java
@Component
class CallbackIngestionHandler {

    Mono<ServerResponse> accept(ServerRequest request) {
        return request.bodyToMono(PaymentCallback.class)
                .switchIfEmpty(Mono.error(() -> new InvalidCallback("empty body")))
                .checkpoint("callback:parse")

                .flatMap(validator::validate)
                .checkpoint("callback:validate")

                .flatMap(inbox::recordIfNew)                  // Topic 116
                .checkpoint("callback:inbox-dedupe")

                .flatMap(orders::applyPaymentResult)
                .checkpoint("callback:apply-to-order")

                .doOnEach(ReactiveLogging.logOnNext(r ->
                        log.info("callback applied orderId={}", r.orderId())))

                .flatMap(r -> ServerResponse.accepted().bodyValue(r))
                .onErrorResume(InvalidCallback.class, this::badRequest)
                .checkpoint("callback:ingestion");            // outermost
    }
}
```

**The placement rule, stated as a rule:** one checkpoint per *stage boundary*, named after
the stage, description-only. Not one per operator. The goal is to answer "which stage of
which pipeline", and a stage is the unit at which you would act.

**Why the outermost checkpoint is last in the fluent chain:** it is the most downstream, so
every error passing through the whole chain crosses it. Reading the suppressed block from
the bottom up gives you the path.

### Step 4 — what a failure looks like after all of this

> *illustration of the format, not captured output*

```
<ts> ERROR [reactor-http-nio-<n>] [<corr-id>] c.o.p.CallbackIngestionHandler - ingestion failed
<ExceptionType>: <message>
    at com.orderflow.payments.<Class>.<method>(<file>:<n>)
    ... reactor frames ...
    Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException:
    Error has been observed at the following site(s):
        *__checkpoint <-- callback:validate
        *__checkpoint <-- callback:ingestion
    Original Stack Trace:
        ...
```

Read it as: **failed at the `validate` stage of the callback ingestion pipeline, on
correlation `<corr-id>`.** That is enough to grep the correlation ID across every service
and reconstruct the request. Compare that to the undecorated version, which told you a
validation method threw somewhere.

### Step 5 — the non-negotiable test

```java
@Test
void everyLogLineOnTheIngestionPathCarriesTheCorrelationId() {
    ListAppender<ILoggingEvent> appender = attachAppenderToIngestionLoggers();

    webTestClient.post().uri("/api/payments/callbacks")
            .header("X-Correlation-Id", "test-correlation")
            .bodyValue(sampleCallback())
            .exchange()
            .expectStatus().isAccepted();

    assertThat(appender.list)
            .isNotEmpty()
            .allSatisfy(event ->
                    assertThat(event.getMDCPropertyMap())
                            .containsEntry("correlationId", "test-correlation"));
}
```

**This test is the deliverable, not the code.** Correlation-ID propagation regresses on
every PR that adds an operator, and it regresses *silently*. A test that asserts every log
event on the path carries the field is the only thing that keeps it working. Add the same
test for the error path — that is the path you will actually need during an incident, and
it is the one people forget to cover.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a stack trace that names only Reactor internals

**Wrong approach.** Ship the reactive path with no checkpoints, no debug agent, and the
assumption that a stack trace is a stack trace.

**Exact symptom.** During an incident: an exception whose only application frame is a shared
helper (`validate`, `toDomain`, `mapRow`) used by several pipelines, followed by twenty-plus
`reactor.core.publisher.*` frames. You cannot tell which endpoint, which pipeline, or which
of the four call sites. Grepping the helper's name finds all four.

**Root cause.** The assembly stack was discarded at assembly time and nothing captured it.
The trace is the *subscription* path, which is structurally the same for every chain that
happens to include a `flatMap` and a `map`.

**Fix — three levels, in increasing cost:**

```java
// Level 1 -- permanent, production-safe. Description only, at stage boundaries.
.checkpoint("callback:validate")

// Level 2 -- dev/test, JVM-wide, cheaper than Hooks. First line of main().
ReactorDebugAgent.init();

// Level 3 -- dev only, JVM-wide, expensive. Full assembly stacks everywhere.
Hooks.onOperatorDebug();
```

**WHAT TO LOOK FOR after Level 1:**

| What you see | What it means |
|---|---|
| `*__checkpoint <-- <your name>` in a suppressed block | Working. The chain is identified. |
| No suppressed block | The error path bypasses your checkpoint. Check whether an `onErrorResume` upstream swallowed and re-threw a *new* exception — a new exception starts a new assembly trail. |
| Checkpoint names you do not recognise | Another team's chain, or a library's. Useful information. |
| Correct checkpoint but still ambiguous | Your stage granularity is too coarse. Split the stage, do not add per-operator checkpoints. |

**The rule to carry:** *checkpoint at boundaries you would act on, not at operators.*

### Trap 2 — MDC lost across a `flatMap` (this is the drill)

**Wrong approach.** `MDC.put("correlationId", id)` in a `WebFilter`, exactly as you would in
a servlet filter, and assume the pattern layout does the rest.

**Exact symptom.** Log lines from the same request split into two groups: early lines with
the field populated, later lines with it empty. **Nothing throws.** No error appears
anywhere. The first person to notice is whoever tries to trace a failed payment.

**Root cause.** MDC is a `ThreadLocal`. `publishOn`, `flatMap` over an async source, and any
scheduler boundary can move execution to a different thread. The MDC map stays behind. This
is exactly Topic 120's mechanism, and the same reason `SecurityContextHolder` (Topic 56) and
plain `ThreadLocal`-based tenant context (Topic 38) break on this path.

**Fix.** Move the value into the subscription, and bridge it back to MDC at each log point:

```java
// 1. Put it in the Context, at the bottom of the chain:
.contextWrite(Context.of("correlationId", correlationId))

// 2. Read it where you log, via doOnEach (the only doOn* that exposes ContextView):
.doOnEach(signal -> {
    if (!signal.isOnNext()) return;
    try (var ignored = MDC.putCloseable("correlationId",
            signal.getContextView().getOrDefault("correlationId", "none"))) {
        log.info("...");
    }
})
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Field present on every line, both sides of the hop | Fixed. |
| Field present before the hop, `"none"` after | The `contextWrite` is placed **upstream** of the reading operator. Move it further down the chain. |
| Field present but wrong value under concurrency | You captured the ID into a shared field or a non-final variable, not into the per-subscription `Context`. |
| Field present on `onNext` lines, missing on error lines | You only bridged the `isOnNext()` branch. Handle `isOnError()` too — **and that is the branch you will need during an incident.** |
| Field present in tests, missing in production | Different code path, or an operator added since. This is why the assertion test exists. |

### Trap 3 — `Hooks.onOperatorDebug()` left on in production

**Wrong approach.** Enable it during an incident to get readable traces. It works. It stays.

**Exact symptom.** Throughput drops at the next deploy with no obvious code change.
Allocation rate rises (visible in `-Xlog:gc*` and in a profiler, Topics 68, 78). A CPU flame
graph (Topic 78) shows a broad plateau in `fillInStackTrace` / `StackTraceElement` machinery
under `reactor.core.publisher.*` assembly frames. Nobody can attribute it to a commit,
because the change was a config flag.

**Root cause.** Every operator assembly now performs a stack walk and retains the result.
With a cold chain assembled per request, the cost multiplies by operator count and by
request rate.

**Fix.**

```java
// Make it impossible to enable accidentally in production.
@Configuration
@Profile("!prod")
class ReactorDebugConfig {
    @PostConstruct
    void enableOperatorDebug() {
        Hooks.onOperatorDebug();
        log.warn("Reactor operator debug mode ENABLED -- non-production only");
    }
}
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| That `WARN` line in a production log | It is on where it should not be. Turn it off and re-measure. |
| `fillInStackTrace` prominent in a production flame graph | Either this hook, or exceptions used for control flow. Both are worth fixing. |
| Throughput recovers after removing it | Confirmed cause. Record the delta — that number is your evidence for the "reactive costs more to operate" line in Topic 107. |
| Throughput does not recover | It was not the cause. Keep looking; do not leave the flag on because it "might help". |

**The narrow exception:** on a low-traffic internal service where debuggability matters more
than throughput, leaving `ReactorDebugAgent` (not `Hooks`) on can be a defensible, *measured*
choice. Measure it, write down the number, and revisit it. Never leave `Hooks.onOperatorDebug()`
on by default.

### Trap 4 — `contextWrite` placed at the top of the chain

**Wrong approach.** Reading the fluent chain top-to-bottom like imperative code, and writing
the context first:

```java
// WRONG -- nothing below can see this value.
Mono.just(request)
    .contextWrite(Context.of("correlationId", id))     // <-- too early
    .flatMap(this::validate)
    .doOnEach(logWithCorrelationId())                  // reads "none"
```

**Exact symptom.** `getOrDefault` always returns the default. The `Context` looks empty
everywhere. People conclude "`Context` doesn't work" and go back to MDC.

**Root cause.** Context propagates **upstream** during subscription. A `contextWrite` at
position N is visible to operators at positions 1..N−1 (upstream of it), not to N+1 onward.
It is the exact opposite of how the code reads.

**Fix.** Put `contextWrite` at the **bottom** of the chain — or, in a filter, on the result
of `chain.filter(exchange)`, as in Example 2 Step 1.

**WHAT TO LOOK FOR — Proof 4 below is exactly this experiment.** Move one line and watch the
value appear.

| What you see | What it means |
|---|---|
| Value appears after moving `contextWrite` down | Confirmed. Internalise the direction. |
| Still absent | Something between is subscribing to a *separate* chain — an inner `Mono` created with its own `subscribe()` does not inherit the outer context. |
| Value present in one branch of a `flatMap`, absent in another | The absent branch subscribes independently. Pass the value explicitly, or ensure the inner publisher is composed rather than separately subscribed. |

### Trap 5 — "fixing" the context problem with `block()` or a `ThreadLocal` copy

**Wrong approach.** Two variants of the same instinct:

```java
// Variant A: capture the ThreadLocal at assembly and close over it.
String corr = MDC.get("correlationId");                 // read on the ASSEMBLING thread
return chain.doOnNext(v -> log.info("{} {}", corr, v)); // "works" -- until it doesn't

// Variant B: block to get back onto a known thread.
return Mono.fromCallable(() -> service.blockingCall()).block();   // on an event loop
```

**Exact symptom.** Variant A: the correlation ID is present but **wrong** — it is whichever
request's ID happened to be on the assembling thread, so under concurrency you get another
customer's ID in your log lines, which is both a debugging failure and arguably a
data-leakage incident. Variant B: `IllegalStateException` from Reactor about blocking on a
non-blocking thread, or, worse, no exception and a stalled event loop (Topic 106 Trap 1).

**Root cause.** Both treat a structural problem as a plumbing problem. The value must travel
with the *subscription*, because that is the only thing that is per-request. The assembling
thread is not per-request in a reactive server (chains are frequently assembled once and
subscribed many times), and the executing thread is not stable.

**Fix.** `Context` for the value; `ContextSnapshot` at genuine boundaries with non-reactive
code; BlockHound to catch variant B in CI:

```java
@Test
void ingestionPathNeverBlocks() {
    BlockHound.install();
    webTestClient.post().uri("/api/payments/callbacks")
            .bodyValue(sampleCallback())
            .exchange()
            .expectStatus().isAccepted();
}
```

| What you see | What it means |
|---|---|
| `BlockingOperationError` naming a JDBC or filesystem frame | A blocking call on a non-blocking thread. Fix it or move it to `boundedElastic` deliberately. |
| Correlation IDs that belong to *other* requests | Variant A is live somewhere. Search for `MDC.get` outside a `doOnEach`. |
| BlockHound will not install on your JDK | It is instrumentation-based and can lag JDK releases. Say so honestly rather than claiming the path is verified. |

---

## Hands-on proof

### Setup

```java
// build: reactor-core comes via spring-boot-starter-webflux (BOM-managed).
// Add for the debug agent and BlockHound:
//   io.projectreactor:reactor-tools
//   io.projectreactor.tools:blockhound-junit-platform  (test scope)
```

### Proof 1 — the assembly method is genuinely absent from the trace

```java
class AssemblyProof {
    Mono<String> assembleHere(String s) {          // <-- look for this name
        return Mono.just(s).map(v -> { throw new IllegalStateException("boom"); });
    }
    void subscribeHere() { assembleHere("x").block(); }
    public static void main(String[] a) { new AssemblyProof().subscribeHere(); }
}
```

**WHAT TO LOOK FOR:** search the printed trace for `assembleHere`.

| What you see | What it means |
|---|---|
| `assembleHere` **absent**, `subscribeHere`/`block` present | The mechanical statement, demonstrated. The assembly stack is gone. |
| `assembleHere` present | You have `Hooks.onOperatorDebug()` or `ReactorDebugAgent` already active. Turn it off and re-run — you need to see the undecorated form first. |
| Only `main` and Reactor frames | Also correct; `block()` inlines. The absence of `assembleHere` is the finding either way. |

### Proof 2 — `.log()` shows the signal sequence and the thread

```java
Flux.range(1, 3)
    .log("stage.source")
    .publishOn(Schedulers.boundedElastic())
    .log("stage.after-hop")
    .blockLast();
```

**WHAT TO LOOK FOR:** `onSubscribe`, `request(...)`, `onNext(...)`, `onComplete`, each
prefixed with the category and the thread.

| What you see | What it means |
|---|---|
| `request(unbounded)` at the source | No backpressure is being applied here (Topic 105). |
| `request(<n>)` with a bounded n, repeatedly | Demand propagation is live. |
| Different thread names before and after the hop | The thread-hop fact, in your own output. |
| `onNext` after `onComplete` | A specification violation somewhere upstream — usually a hand-written `Publisher` or a misused `Sinks`. Rare and serious. |

**Use `.log()` to understand a chain; remove it before merging.** It logs every signal and
will dominate your log volume under load.

### Proof 3 — `checkpoint()` vs `Hooks.onOperatorDebug()`, same failure

Run the Proof 1 program three times: unmodified, with `.checkpoint("assembleHere")` added,
and with `Hooks.onOperatorDebug()` as the first line of `main`.

**WHAT TO LOOK FOR:** the presence and content of the suppressed `OnAssemblyException` block.

| Run | What you see | What it means |
|---|---|---|
| Unmodified | No suppressed block | Baseline. |
| `checkpoint("name")` | Suppressed block naming your description, **no assembly line number** | The cheap form. Identifies the chain, not the line. |
| `checkpoint("name", true)` | Description **and** an assembly stack | The expensive form of the cheap tool. |
| `Hooks.onOperatorDebug()` | Assembly information for **every** operator, not just yours | The expensive form. Note it also decorated operators inside libraries. |

**The comparison is the lesson:** you are choosing between targeted-and-cheap and
global-and-expensive, and the targeted one is what ships.

### Proof 4 — `contextWrite` direction

```java
// Run A: write ABOVE the read.
Mono.deferContextual(ctx -> Mono.just(ctx.getOrDefault("k", "MISSING")))
    .contextWrite(Context.of("k", "value"))
    .doOnNext(v -> System.out.println("A: " + v))
    .block();

// Run B: write BELOW the read -- i.e. deferContextual first in the fluent chain.
Mono.just("ignored")
    .flatMap(x -> Mono.deferContextual(ctx -> Mono.just(ctx.getOrDefault("k", "MISSING"))))
    .doOnNext(v -> System.out.println("B: " + v))
    .contextWrite(Context.of("k", "value"))
    .block();
```

**WHAT TO LOOK FOR:** which run prints `MISSING`.

| What you see | What it means |
|---|---|
| A prints the value, B prints the value | Both `contextWrite`s are downstream of their readers. Re-read the chains — this is the correct outcome for the code as written above, and the point is that *position in the fluent chain* is what decides it. |
| A reader prints `MISSING` | Its `contextWrite` is upstream of it. Move the write further down. |
| Both print `MISSING` | No `contextWrite` is on the subscribed path at all — check you are not subscribing to an inner publisher independently. |

**Do this experiment by hand until the direction is instinct.** It is the single most
common `Context` bug and the one you will diagnose for other people.

### Proof 5 — where a chain actually executes, end to end

```java
private static String here(String label) {
    return label + "@" + Thread.currentThread().getName();
}

return request.bodyToMono(PaymentCallback.class)
        .doOnNext(c -> log.info(here("parse")))
        .flatMap(validator::validate)
        .doOnNext(c -> log.info(here("validate")))
        .flatMap(inbox::recordIfNew)
        .doOnNext(c -> log.info(here("inbox")))
        .flatMap(orders::applyPaymentResult)
        .doOnNext(r -> log.info(here("apply")));
```

**WHAT TO LOOK FOR:** the thread name at each stage, under load rather than for one request.

| What you see | What it means |
|---|---|
| All stages on `reactor-http-nio-*` | Fully non-blocking path. Good. |
| A stage on `boundedElastic-*` | Something was wrapped for blocking. Deliberate or accidental — find out which (Topic 106 Trap 5). |
| A stage on a `http-nio-*` (Tomcat) name | You are not on the reactive stack at all. Two starters on the classpath (Topic 106). |
| Different threads for the same stage across requests | Normal and expected. Reinforces why `ThreadLocal` cannot work. |

---

## Failure drill

**The assigned drill: lose the correlation ID across a `flatMap`, prove it in the logs, then
restore it with `ContextView` and `ContextSnapshot`.**

This is not an illustration. Run it against `orderflow`'s ingestion module.

### The scenario

The gateway posts a callback with `X-Correlation-Id`. Ingestion validates, deduplicates, and
applies to the order. You want every log line from that request to carry the ID.

### Part A — break it (and this is what a naive implementation looks like)

**Step 1.** Set the correlation ID with MDC in a `WebFilter`, exactly as you would on MVC:

```java
@Component
class NaiveCorrelationFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String id = Optional.ofNullable(
                exchange.getRequest().getHeaders().getFirst("X-Correlation-Id"))
                .orElseGet(() -> UUID.randomUUID().toString());
        MDC.put("correlationId", id);                 // <-- the bug
        return chain.filter(exchange)
                .doFinally(s -> MDC.remove("correlationId"));
    }
}
```

**Step 2.** Ensure the pattern renders the field. In `logback-spring.xml`:

```xml
<pattern>%d{HH:mm:ss.SSS} %-5level [%thread] [%X{correlationId}] %logger{36} - %msg%n</pattern>
```

**Step 3.** Log at four points in the ingestion chain, and force a hop so the failure is
deterministic rather than timing-dependent:

```java
return request.bodyToMono(PaymentCallback.class)
        .doOnNext(c -> log.info("1-parsed ref={}", c.reference()))
        .publishOn(Schedulers.boundedElastic())          // deterministic hop
        .doOnNext(c -> log.info("2-after-hop ref={}", c.reference()))
        .flatMap(validator::validate)
        .doOnNext(c -> log.info("3-validated ref={}", c.reference()))
        .flatMap(inbox::recordIfNew)
        .doOnNext(c -> log.info("4-recorded ref={}", c.reference()));
```

**Step 4.** Send one request:

```bash
curl -i -X POST http://localhost:8081/api/payments/callbacks \
  -H 'Content-Type: application/json' \
  -H 'X-Correlation-Id: drill-108-a' \
  -d '{"reference":"cb-drill-1","orderId":"...","status":"CAPTURED","amount":"40.00"}'
```

**Step 5.** Grep:

```bash
grep -n "drill-108-a" logs/orderflow-ingestion.log
grep -nE "^.*(1-parsed|2-after-hop|3-validated|4-recorded)" logs/orderflow-ingestion.log
```

### What to capture, before reading on

Write down, on paper:

1. Which of lines 1–4 carry `drill-108-a`, and which show an empty `[]`.
2. The thread name on each of the four lines.
3. Whether **any** error or warning appeared. (It did not. Write that down explicitly — the
   silence is the point.)
4. Whether the ID survived if you remove the explicit `publishOn`. (Try it. Under load it
   will still be lost, intermittently — which is worse.)

### How to read it

| What you see | What it means |
|---|---|
| Line 1 has the ID; lines 2–4 empty | **Textbook.** The `publishOn` moved execution to `boundedElastic-*` and the MDC map stayed on the request thread. |
| All four empty | `MDC.put` ran on a different thread from line 1 as well — the `WebFilter` itself may have been invoked on a different thread than the body-decode. Even worse, and the same root cause. |
| All four have the ID, with no `publishOn` | This run had no hop. **Do not conclude it works.** Run it under the Topic 65 load profile and check the ratio of lines with and without the field. |
| The ID is present but *wrong* on some lines | MDC leaked between requests: a `remove` was missed, and a pooled thread carried a stale value into another request. This is Topic 79b's `ThreadLocal` leak shape, and it is a correctness problem, not just a debugging one. |
| Line 4 empty only under load | The intermittent form. The worst one, because it passes review and fails during an incident. |

### Part B — fix it with `ContextView`

**Step 1.** Replace `MDC.put` with `contextWrite`, at the bottom:

```java
@Component
class CorrelationWebFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String id = Optional.ofNullable(
                exchange.getRequest().getHeaders().getFirst("X-Correlation-Id"))
                .orElseGet(() -> UUID.randomUUID().toString());
        exchange.getResponse().getHeaders().add("X-Correlation-Id", id);
        return chain.filter(exchange)
                .contextWrite(Context.of("correlationId", id));   // BOTTOM. Deliberately.
    }
}
```

**Step 2.** Replace each `doOnNext(log...)` with the context-aware form:

```java
.doOnEach(signal -> {
    if (!signal.isOnNext() && !signal.isOnError()) return;    // handle BOTH
    String id = signal.getContextView().getOrDefault("correlationId", "none");
    try (var ignored = MDC.putCloseable("correlationId", id)) {
        if (signal.isOnNext()) {
            log.info("2-after-hop ref={}", signal.get().reference());
        } else {
            log.error("2-after-hop FAILED", signal.getThrowable());
        }
    }
})
```

**Step 3.** Re-run Step 4 of Part A with `X-Correlation-Id: drill-108-b` and re-grep.

| What you see | What it means |
|---|---|
| All four lines carry `drill-108-b`, on different threads | Fixed. The value travelled with the subscription. |
| Lines still empty | `contextWrite` is upstream of the readers, or the readers are on a separately-subscribed inner chain (Trap 4). |
| Field present on success, empty on failure | You did not handle `isOnError()`. Fix it now — the error path is the one you will need. |

### Part C — restore it across a boundary with `ContextSnapshot`

Part B covers the reactive chain. Part C covers the boundary: reactive code calling a
non-reactive component that logs, or handing work to an executor.

**Step 1.** Register a `ThreadLocalAccessor` for the MDC entry and enable propagation:

```java
ContextRegistry.getInstance()
        .registerThreadLocalAccessor(new CorrelationIdThreadLocalAccessor());
Hooks.enableAutomaticContextPropagation();
```

> **Verify this API against your BOM before writing it.** The `ThreadLocalAccessor`
> interface shape and the registration entry point have moved across `context-propagation`
> versions, and whether `enableAutomaticContextPropagation()` is needed depends on your
> Reactor version. The Reactor reference documentation's "Context Propagation" section is
> the authority; this document is not.

**Step 2.** At an explicit boundary — say, handing a batch to an executor for a non-reactive
settlement job:

```java
ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

executor.submit(() -> {
    try (ContextSnapshot.Scope scope = snapshot.setThreadLocals()) {
        settlementJob.run(batch);        // logs inside here now carry the field
    }
});
```

**Step 3.** Prove it: log inside `settlementJob.run` and grep for `drill-108-c`.

| What you see | What it means |
|---|---|
| The executor's log lines carry the ID | The snapshot bridged the boundary. |
| Empty inside the executor, present outside | The snapshot was captured on a thread that did not have the value, or the scope was not opened around the logging code. |
| The ID appears but is stale on the *next* task | The `Scope` was not closed. Use try-with-resources, always. **This is the `ThreadLocal` leak from Topic 79b, reintroduced.** |
| It works without any of Part C's code | Automatic propagation is active in your version. **Verify by removing it deliberately once**, and keep the explicit form if you cannot prove the automatic one. |

### What the drill proves

Three things you can now state from experience rather than from reading:

1. **`ThreadLocal` context loss is silent.** No exception, no warning, no failing test unless
   you wrote one.
2. **The value must live in the subscription, because the subscription is the only
   per-request thing.** Not the thread, not the assembling method.
3. **The bridge to `ThreadLocal` is needed anyway**, because your logging framework, your
   tracing library and your security context all read thread-locals. `Context` does not
   replace `ThreadLocal`; it is the transport, and `ContextSnapshot` is the adapter at each
   end.

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` loop is the WRONG way to measure the cost of any of these
> instruments.** It measures JIT warm-up, dead-code elimination, on-stack replacement and
> ambient noise. Topic 77 (JMH) is where you learn to do it properly.

For assembly-time costs specifically, the naive loop is even worse than usual: assembling a
chain in a tight loop with no subscription is precisely the shape the JIT can prove has no
effect and delete. **A `nanoTime` loop around `Mono.just(x).map(f).flatMap(g)` with no
subscribe may legitimately measure nothing at all.**

### Two different questions, two different instruments

| Question | Instrument | Why not the other one |
|---|---|---|
| "What does `Hooks.onOperatorDebug()` cost per assembly?" | **JMH** (Topic 77), with a `Blackhole` consuming the assembled chain | It is a small, repeatable, CPU-bound operation — exactly JMH's domain |
| "What does it cost `orderflow` at the baseline?" | **Topic 65's harness**, open-model, matched arrival rates | A microbenchmark cannot capture GC pressure, allocation rate and real request shape |

Run both. The JMH number tells you the per-operator cost; the harness number tells you
whether it matters at your rate. **Reporting only one of them is how people end up either
paranoid about a negligible cost or blind to a large one.**

### The overhead comparison to actually run

Four configurations of the *same* ingestion path, at matched arrival rates against the
Topic 65 baseline:

| Config | Setting |
|---|---|
| **1** | No debug instrumentation |
| **2** | `checkpoint("name")` at stage boundaries, description only |
| **3** | `ReactorDebugAgent.init()` |
| **4** | `Hooks.onOperatorDebug()` |

For each: throughput at a fixed offered rate, p99, allocation rate (`-Xlog:gc*` or a
profiler), and peak heap.

| What you see | What it means |
|---|---|
| 2 indistinguishable from 1 | Expected. Description-only checkpoints are close to free. **Ship them.** |
| 3 measurably below 1 but acceptable | The debug agent's cost is real. Decide with the number in front of you, per environment. |
| 4 substantially below 1, with allocation rate up | Expected. This is why it does not ship. **Record the number — it is evidence for Topic 107.** |
| 2 measurably below 1 | You used `checkpoint()` with no description, or `checkpoint(name, true)`. Both capture stacks. Check every call site. |
| No difference between any of them | Your load is not assembly-bound, or you never reached saturation. Check the load model (Topic 107 Trap 4) before concluding "it's free". |

### Correlation-ID coverage — the metric that should exist and usually does not

Do not measure this by looking at logs. Measure it as a number.

```java
// In the doOnEach bridge, count the misses:
if ("none".equals(id)) {
    missingCorrelationId.increment();       // Micrometer counter, Topic 118
}
```

```
orderflow.log.correlation_id.missing
```

| What you see | What it means |
|---|---|
| Zero at all arrival rates | Propagation is correct on this path. |
| Zero at 1x, nonzero at 4x | An intermittent hop that only manifests under concurrency. **The worst kind, and only a counter finds it.** |
| Nonzero constantly on one endpoint | A chain missing its `contextWrite`, or an inner publisher subscribed independently. |
| Nonzero only on the error path | The `isOnError()` branch is unbridged. |

**Pair it with an alert.** A correlation ID you cannot rely on is worse than none, because
you will trust it during an incident.

### BlockHound as a permanent instrument

BlockHound belongs in CI, not in a one-off investigation:

```java
// A test-scope extension via blockhound-junit-platform installs it for the whole suite.
```

| What you see | What it means |
|---|---|
| Build fails with `BlockingOperationError` naming a driver frame | A blocking call reached a non-blocking thread. This is the outage BlockHound exists to prevent (Topic 106 Trap 1). |
| Build fails naming a logging or classloading frame | Often benign. Allow-list it **explicitly, with a comment saying why**, rather than disabling BlockHound. |
| BlockHound does not install on your JDK | Record it as a gap. Do not claim the path is verified when your verifier is not running. |

---

## Practice exercises

### 1 — easy: read three traces

Produce three stack traces from the same deliberate failure in `orderflow`'s ingestion path:

1. undecorated,
2. with `.checkpoint("callback:validate")`,
3. with `Hooks.onOperatorDebug()`.

For each, write down: (a) how many application frames appear, (b) whether you can name the
assembling method, (c) whether you can name the *line* that assembled the failing operator,
and (d) how long it took you to identify the pipeline.

**Done when:** you can state, in one sentence each, what information each level buys, and
why (2) is the one that ships.

### 2 — medium: the propagation audit

Take every public entry point on the ingestion module. For each:

1. Write a test asserting that **every** log event on the success path carries the
   correlation ID.
2. Write the same test for the **error** path.
3. Run the suite and fix what fails.
4. Add the `orderflow.log.correlation_id.missing` counter and run the Topic 65 load profile.
5. Report the counter at 1x, 2x and 4x arrival rates.

**Done when:** the counter is zero at every rate, both tests exist for every entry point,
and you can name at least one place where the error-path test failed while the success-path
test passed. If none did, you did not test enough error paths.

### 3 — hard: the debuggability tax, priced

Produce a two-page memo, in the shape Topic 107 consumes, answering: **what does debugging
the reactive ingestion module cost us, per year, compared with the imperative core?**

Include:

1. **The overhead table** from the Measurement section, four configurations, real runs, with
   environment and repetition count recorded.
2. **The instrumentation you had to add** to reach parity with the imperative path:
   checkpoints, the `doOnEach` bridging, the accessor registration, the coverage tests.
   Count the lines. The imperative path needs a filter and a `finally` — count those too.
3. **The residual gap.** What is still worse after all that work? Be specific: the debugger
   steps into `FluxFlatMap`; a heap dump shows operator subscribers rather than request
   frames (Topic 79); a thread dump during an in-flight request shows no application frame
   (Topic 107 Proof 2).
4. **An incident-minutes estimate** derived from your own incident record, not invented. If
   you have no incident record, say so and state what you would need to collect.
5. **A recommendation**: is the ingestion module's reactive design still justified once this
   cost is on the table? Defend either answer — but defend it with the numbers you produced.

**Marking rubric:**

| Criterion | Fail | Pass | Strong |
|---|---|---|---|
| Overhead numbers | Quoted from a blog | Measured once | Measured with spread, environment recorded, invalid runs discarded |
| Instrumentation count | Hand-waved | Counted | Counted for both paths, so the delta is real |
| Residual gap | "It's harder" | Three specific mechanisms | Mechanisms plus how each one manifests during an incident |
| Incident cost | Invented | Derived from the record | Derived, with the record's limitations stated |
| Recommendation | Absent or hedged | Stated | Stated, with what would change it |

---

## Interview questions

### Q1 — "Why is a reactive stack trace useless, and what do you do about it?"

**MID-LEVEL ANSWER.** "The stack trace only shows Reactor internals, so you turn on
`Hooks.onOperatorDebug()`." Correct instrument, no mechanism, and no awareness of cost.

**SENIOR ANSWER.**

> "It is not useless — the top frame is genuinely your throw site. What is missing is
> *which chain*, and that is missing for a structural reason. A Reactor chain is built at
> assembly time and executed at subscription time. The stack that built the chain returned
> long before anything ran, so at failure time the JVM stack contains the operator
> subscribers and the subscription origin, not your assembling method. The information does
> not exist unless something captured it at assembly.
>
> So there are three levels. `checkpoint("description")` at stage boundaries is cheap —
> description-only captures no stack — and it answers the question I actually have, which is
> which pipeline and which stage. That ships to production. `ReactorDebugAgent` from
> `reactor-tools` instruments at class load and is much cheaper than the hook; I would run
> it in dev and test, and consider it in production only with a measured number.
> `Hooks.onOperatorDebug()` does a `fillInStackTrace` per operator assembly JVM-wide — great
> for a local investigation, an outage waiting to happen if it survives to production.
>
> And I would guard it: the enabling `@Configuration` gets `@Profile("!prod")` and logs a
> warning when it activates, so a stray flag is visible in the logs."

**WHAT SEPARATES THEM.** The mechanism (assembly vs subscription), the three-level cost
ladder rather than one tool, the recognition that the top frame *is* useful, and the
operational guard. The last item is what makes it sound like someone who has been on call.

**FOLLOW-UP: "Where exactly do you put checkpoints?"**

> "At stage boundaries I would act on, named after the stage — parse, validate, dedupe,
> apply — not one per operator. If a checkpoint fires and I still cannot tell what to do, my
> stages are too coarse and I split the stage. And description-only, always, unless I am
> actively debugging one specific chain."

### Q2 — "Our correlation IDs disappear halfway through a reactive request. Why?"

**MID-LEVEL ANSWER.** "MDC doesn't work with reactive, you have to use the Reactor
`Context`." True, and does not explain, so it will not survive a follow-up.

**SENIOR ANSWER.**

> "MDC is `ThreadLocal`-backed. A reactive chain can change threads between operators — a
> `publishOn`, or a `flatMap` whose inner publisher completes on an I/O thread. When it
> does, the MDC map stays on the thread you left, so every log statement after the hop
> renders the field as empty. And it fails *silently*: MDC does not throw on a missing key,
> so the only symptom is a field that is quietly absent.
>
> This is the same root cause as the stack-trace problem, which is worth saying out loud:
> the thread's stack was both the call history and the ambient context store, and reactive
> discards it between operators. So you lose both at the same moment.
>
> The fix is to move the value into Reactor's `Context`, which lives in the subscription and
> therefore travels with the request rather than with the thread. The direction catches
> people: context propagates upstream at subscription time, so `contextWrite` has to be
> *below* everything that reads it — at the bottom of the chain, or on the result of
> `chain.filter(exchange)` in a `WebFilter`.
>
> Then you still need a bridge, because Logback reads MDC, not `Context`. The portable way
> is `doOnEach`, which is the only `doOn*` that hands you the `ContextView`; open an
> `MDCCloseable` around the log statement. There is also Micrometer's `context-propagation`
> with `ThreadLocalAccessor` and automatic propagation, which does it at boundaries for you
> — but whether it is on by default depends on your Reactor version, so I verify it with a
> test rather than assuming it.
>
> And the actual deliverable is a test asserting every log event on the path has the field,
> plus a counter for misses under load, because this regresses on any PR that adds an
> operator and nobody notices until an incident."

**WHAT SEPARATES THEM.** Naming the silence as the defining property; connecting it to the
stack-trace problem as one root cause; getting the `contextWrite` direction right unprompted;
knowing `doOnEach` is the operator with `ContextView` access; and finishing with the test and
the counter rather than the code.

**FOLLOW-UP: "Does this affect anything besides logging?"**

> "Everything `ThreadLocal`-based. `SecurityContextHolder` — the reactive stack has a
> separate `ReactiveSecurityContextHolder` for exactly this reason. Tracing spans, which is
> why OpenTelemetry needs context propagation configured on a reactive path. And any
> home-grown tenant or request context. On virtual threads all of these keep working, which
> is one of the underrated arguments in the Loom-versus-reactive discussion."

### Q3 — "Someone enabled `Hooks.onOperatorDebug()` in production. What happens and how would you find it?"

**MID-LEVEL ANSWER.** "It slows things down." Directionally right, no mechanism, no method.

**SENIOR ANSWER.**

> "Every operator assembly captures a stack trace — `fillInStackTrace` plus retention. With
> a chain assembled per request, the cost is operator-count times request rate, and stack
> walking scales with stack depth, which in Spring is deep. So you get lower throughput,
> higher allocation rate, and more GC pressure.
>
> Finding it: a CPU flame graph would show a broad plateau in stack-trace machinery under
> Reactor assembly frames — that is a distinctive shape and it is what I would look for
> first. Allocation profiling would corroborate it. And I would grep the config and the
> startup logs, because this is usually a flag someone set during an incident and never
> removed, so there is no commit to blame.
>
> The prevention is structural: the configuration that enables it carries `@Profile("!prod")`
> and logs a warning on activation, so the flag is visible if it is ever on where it should
> not be. If we genuinely want assembly information in production, `ReactorDebugAgent` is
> the cheaper mechanism, and I would only enable it with a measured throughput cost written
> down and a date to revisit."

**WHAT SEPARATES THEM.** Naming the mechanism precisely; giving a *distinctive signature* to
look for rather than "profile it"; recognising the config-with-no-commit provenance problem;
and proposing a structural guard instead of a resolution to be careful.

### Q4 — "Is debugging reactive hard enough to affect your architecture choice?"

**MID-LEVEL ANSWER.** Either "no, you get used to it" or "yes, reactive is unmaintainable".
Both are positions rather than assessments.

**SENIOR ANSWER.**

> "It is a real cost and I would price it rather than assert it. Concretely, going reactive
> costs you: assembly information, which you buy back with checkpoints at stage boundaries;
> `ThreadLocal` context, which you buy back with `Context` plus a bridge and a coverage test;
> and a debugger that steps into operator internals, which you cannot buy back at all. A
> heap dump or a thread dump during an in-flight request shows operator subscribers rather
> than a request stack — that one is permanent too.
>
> The instrumentation to reach parity is countable: I would count the lines of checkpoints,
> bridging and tests on the reactive path versus a filter and a `finally` on the imperative
> path, and I would count incident minutes attributable to reactive-specific debugging from
> our own incident record.
>
> Then it goes into the same table as everything else. For a path with genuine backpressure
> requirements the capability is worth the tax. For a request/response path where virtual
> threads give the same thread economy with none of it, the tax is pure loss. That is Topic
> 107's decision, and this is one of its inputs — but it is an input, not the answer."

**WHAT SEPARATES THEM.** Converting a subjective complaint into countable line items and
incident minutes; distinguishing what you can buy back from what you cannot; and explicitly
subordinating it to the larger decision rather than treating it as decisive on its own.

### Q5 — "Walk me through diagnosing an intermittent failure on a reactive path with no useful logs."

**MID-LEVEL ANSWER.** "Add more logging and reproduce it." Will work eventually; no ordering,
no hypothesis.

**SENIOR ANSWER.**

> "Ordered by cost, cheapest first, and each step designed to eliminate a class of cause.
>
> First, check whether the correlation ID is present on *all* lines from the affected
> requests. If it is missing on some, I have two problems and the missing context is the one
> blocking me — fix that before anything else, because without it I cannot even assemble the
> evidence.
>
> Second, add description-only checkpoints at stage boundaries and redeploy. That is cheap
> and safe, and it tells me which stage is failing, which is usually 80% of the diagnosis.
>
> Third, if it is intermittent, I suspect concurrency or backpressure rather than logic.
> `.log()` on a canary instance shows me the signal sequence and the `request(n)` pattern —
> if demand is unbounded somewhere, a buffer is growing and the failure will correlate with
> load.
>
> Fourth, thread names inside operators. If a stage is running on `boundedElastic` when I
> expected `reactor-http-nio`, someone wrapped a blocking call, and that changes the failure
> model entirely.
>
> Fifth, BlockHound in the test suite against the same path — if something blocks, the
> failure will correlate with load and look intermittent when it is actually deterministic
> under contention.
>
> Only then `Hooks.onOperatorDebug()`, on a canary, with a plan to remove it. It is the
> most expensive and the most informative, and by that point I usually do not need it."

**WHAT SEPARATES THEM.** An explicit cost ordering; treating missing correlation IDs as a
blocker rather than an inconvenience; the hypothesis that intermittent means concurrency or
backpressure rather than logic; and the discipline of a canary plus a removal plan for the
expensive tool.

---

## Mental model checkpoint

Answer without scrolling up.

1. **Why does a reactive stack trace not name the method where you wrote the chain?**
   Because that method ran at assembly time and returned. At failure time the JVM stack is
   the subscription path. The information does not exist unless captured at assembly.

2. **State the relationship between the trace problem and the lost-MDC problem.**
   Same root cause. The thread's stack was both the call history and the ambient context
   store; reactive discards it between operators, so you lose both.

3. **`checkpoint("name")` versus `Hooks.onOperatorDebug()` — one sentence each.**
   `checkpoint` inserts assembly information at chosen points, cheaply, and
   description-only captures no stack. `Hooks.onOperatorDebug()` captures an assembly stack
   at every operator in the JVM, expensively.

4. **Which direction does Reactor's `Context` propagate, and what does that mean for where
   you put `contextWrite`?**
   Upstream, at subscription time. `contextWrite` must be placed **downstream** of every
   operator that reads the value — at the bottom of the chain.

5. **Why does a correct `Context` still not put the correlation ID in your log lines?**
   Logback reads MDC, which is `ThreadLocal`-backed. `Context` is not MDC. You must bridge —
   `doOnEach` plus `MDC.putCloseable`, or a registered `ThreadLocalAccessor` with automatic
   propagation.

6. **What is the observable symptom of MDC loss, and why is it dangerous?**
   An empty field in log lines. Nothing throws. It is dangerous because it is invisible until
   an incident, and because a correlation ID you cannot rely on is worse than none.

7. **Name one thing about reactive debugging you cannot buy back with instrumentation.**
   The debugger. Stepping goes into operator internals. Same for a thread dump or heap dump
   during an in-flight request: there is no request stack to read, only subscribers.

---

## Quick reference card

### The four instruments

```
.log("cat")                  what signals, what order, which thread   -- dev
.checkpoint("stage")         which chain/stage failed                 -- SHIPS
ReactorDebugAgent.init()     assembly lines, class-load instrumented  -- dev/test
Hooks.onOperatorDebug()      assembly lines, per-operator fillInStack -- dev ONLY
```

### Context, in five lines

```java
.contextWrite(Context.of("k", v))                    // WRITE -- at the BOTTOM
Mono.deferContextual(ctx -> ...)                     // READ  -- inside the chain
.doOnEach(sig -> sig.getContextView().get("k"))      // READ  -- while logging
.transformDeferredContextual((flux, ctx) -> ...)     // READ  -- transforming
ContextSnapshotFactory.builder().build().captureAll()// BRIDGE to ThreadLocal
```

### The direction rule

```
Context flows UPSTREAM at subscription time.
contextWrite affects everything ABOVE it in the fluent chain.
Therefore: write LAST, read EARLIER. It reads backwards. Get used to it.
```

### Diagnostic order for a reactive failure

```
1. Is the correlation ID on every line?      -> if not, fix that FIRST
2. checkpoint() at stage boundaries          -> which stage
3. .log() on a canary                        -> signal order, request(n)
4. Thread names inside operators             -> unexpected scheduler?
5. BlockHound in tests                       -> anything blocking?
6. Hooks.onOperatorDebug() on a canary       -> last resort, with a removal plan
```

### What breaks on a reactive path

```
MDC / %X{...}              -> Context + doOnEach bridge      (Topic 120)
SecurityContextHolder      -> ReactiveSecurityContextHolder  (Topic 56)
Tracing spans              -> context propagation configured (Topic 119)
Custom ThreadLocal context -> Context                        (Topic 38)
Debugger stepping          -> nothing. Permanent loss.
Thread/heap dump readability -> nothing. Permanent loss.     (Topics 79, 107)
```

### Gotchas checklist

```
[ ] checkpoint descriptions are STRINGS, not the no-arg or forceStackTrace form
[ ] contextWrite is at the BOTTOM of the chain
[ ] doOnEach handles isOnError(), not just isOnNext()
[ ] ContextSnapshot.Scope is always in try-with-resources
[ ] Hooks.onOperatorDebug() is @Profile("!prod") and logs a warning
[ ] A test asserts every log event on the path carries the correlation ID
[ ] A counter tracks missing correlation IDs under load, not just in tests
[ ] BlockHound runs in CI, and you know whether it installed
```

---

## When would I use this at work?

**1. The first hour on a reactive service you did not write.**

Add description-only checkpoints at every stage boundary and confirm the correlation ID
survives to the end of each path. Those two changes convert an opaque service into one where
a failure report tells you which pipeline and which stage, and where you can grep a single
request end to end. It is perhaps two hours of work and it changes every subsequent
investigation you will do on that codebase.

**2. Reviewing any PR that adds an operator to a reactive chain.**

Two mechanical questions. Does this add a scheduler boundary, and if so is anything
downstream reading a `ThreadLocal`? And is there a stage boundary here that deserves a
checkpoint? Correlation-ID propagation regresses one operator at a time and always silently,
so review plus the coverage test is the only thing that holds it.

**3. Pricing the reactive-versus-imperative decision honestly.**

When someone says "reactive is harder to debug", you can turn it into line items:
checkpoints, the bridging code, the coverage tests, the measured overhead of each debug
level, and the residual gap that no instrumentation closes. That converts a taste argument
into a cost that belongs in Topic 107's table — and being the person who can price a
subjective complaint is a large part of what senior means.

---

## Connected topics

**Prerequisites:**

- **103 — NIO and Netty's event loop:** why there is more than one thread, and why an
  operator can resume on a different one than it started on.
- **104 — Reactor `Mono`/`Flux`:** assembly time versus subscription time. Without that
  distinction, none of this topic's mechanics are derivable.
- **105 — Backpressure:** `.log()`'s `request(n)` lines, and why an intermittent failure
  under load is often a demand problem rather than a logic problem.
- **106 — WebFlux vs MVC:** Trap 4 there previews the `ThreadLocal` loss; this document is
  that trap's full treatment.
- **79 — Memory leaks and heap dumps:** the `ThreadLocal` retention shape you can recreate
  by forgetting to close a `ContextSnapshot.Scope`.

**Also relevant:**

- **56 — Spring Security:** `SecurityContextHolder` is `ThreadLocal`-backed, which is why
  the reactive stack has a separate holder. Same root cause, different subsystem.
- **38 — Bean scopes:** request-scoped beans and any home-grown `ThreadLocal` context have
  the same problem on a reactive path.
- **77 — JMH:** the correct instrument for per-assembly overhead, and the reason a
  `nanoTime` loop around an unsubscribed chain may measure literally nothing.
- **78 — Profiling and flame graphs:** how you find `Hooks.onOperatorDebug()` left on in
  production without a commit to blame.
- **91 — `CompletableFuture`:** the same context-propagation problem in the other async
  model, and the same fix shape.
- **101–102 — Virtual threads and `ScopedValue`:** where all of this stops being a problem,
  and the modern replacement for `ThreadLocal` in imperative code.

**This unlocks:**

- **107 — The Loom-vs-reactive decision:** this document is where the observability tax gets
  priced. The hard exercise here feeds that document's cost table directly.
- **118 — Metrics:** the missing-correlation-ID counter, and why a coverage number beats
  reading logs.
- **119 — Distributed tracing:** the same propagation problem for spans, with an extra
  cross-process hop. `ContextSnapshot` is the same bridge.
- **120 — Logging and MDC:** the canonical treatment of MDC, its `ThreadLocal` nature, and
  the leak shape when you forget to clear it. **This topic is Topic 120's hardest case.**
- **124 — The readiness gate:** a service whose logs cannot be correlated is not
  operationally ready, whatever its health endpoint says.

---

*Java baseline 21. Reactor, `context-propagation` and Micrometer versions come from the
Spring Boot BOM; do not pin them by hand and do not trust a version number quoted in any
document, including this one. Two API surfaces in this document are explicitly
version-dependent and must be checked against the current Reactor reference documentation
before use: `Hooks.enableAutomaticContextPropagation()` together with whether automatic
propagation is on by default, and the `ThreadLocalAccessor` / `ContextSnapshotFactory`
shapes from the `context-propagation` library. The mechanism behind both — an accessor
registered once, then capture and restore at boundaries — is stable; the method names are
not. The one structural fact worth committing to memory is the mechanical statement: the
trace shows the subscription stack, not the assembly stack, and `ThreadLocal` cannot survive
an operator thread hop, for exactly the same reason.*
