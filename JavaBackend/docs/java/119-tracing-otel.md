# 119 — Distributed Tracing with OpenTelemetry and Context Propagation

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: one `orderflow` order placement produces **one** trace that spans HTTP ingress → order service → Postgres → wallet debit → outbox insert → the relay → Kafka → the inventory consumer in a different process — with no orphaned traces at any of those boundaries, and with the trace ID present on every log line from Topic 120.

---

## Mechanical statement

**Trace context lives in a `ThreadLocal` and is injected into outbound headers as W3C
`traceparent`. Any thread hop that is not context-aware drops it silently — no error, just
an orphaned trace.**

Three separate facts, all load-bearing:

1. **The current span is thread state, not request state.** `Context.current()` reads a
   `ThreadLocal`. Nothing carries it for you when execution moves to a different thread.
2. **Crossing a process boundary is an explicit serialisation step.** The context becomes a
   `traceparent` HTTP header, or a Kafka record header. If nobody injects it, the receiving
   process starts a fresh trace with a fresh trace ID.
3. **Failure is silent and looks like success.** A dropped context does not throw. You get
   a valid span, in a valid trace, with a valid trace ID — a *different* one. Your tracing
   UI shows two small healthy traces instead of one big broken one, and nothing anywhere
   says "these are the same request".

That third point is why this topic is a `DIFFERENTIATOR`. Every other failure mode in
Phase 11 announces itself. This one produces plausible, well-formed, wrong data.

---

## The bridge from what you know

### You already know OpenTelemetry, so let us be precise about what transfers

From the Node side you know: traces, spans, parent/child relationships, span attributes and
events, the W3C `traceparent` header, exporters, the OTel Collector, sampling, and the
distinction between an SDK and auto-instrumentation. **All of that is identical in Java.**
It is the same specification and, in large part, the same wire protocol and the same
backends (Jaeger, Tempo, whatever your platform runs).

Two things transfer *especially* cleanly and are worth naming so you stop worrying about
them:

- **The `traceparent` header format is identical.** Same version byte, same 32-hex trace
  ID, same 16-hex span ID, same flags byte.
- **The Collector and your backend do not care what language produced the spans.** A Java
  service and a Node service in the same trace is the normal case, not an integration
  problem.

### What is genuinely different: how the context is carried inside the process

This is the whole topic, so here is the comparison in one table.

| | Node | Java |
|---|---|---|
| Where the active context lives | `AsyncLocalStorage` | `ThreadLocal` (via `ContextStorage`) |
| Who propagates it across async boundaries | the runtime, via async_hooks — `await`, `setTimeout`, promise chains all carry it | **nobody, unless you wrap the boundary yourself** |
| What "a thread hop" costs you | there are no thread hops | context loss |
| How loss manifests | rare, and usually from a broken third-party lib | **common, and from your own code** |
| Failure mode | silent | silent |

`AsyncLocalStorage` follows the *logical* flow of execution. `ThreadLocal` follows the
*physical* thread. In Node those are the same thing often enough that you can forget the
difference. In Java, the moment you hand work to another thread — an `ExecutorService`, an
`@Async` method, a `new Thread(...)`, a `CompletableFuture.supplyAsync`, a Reactor
operator, a Kafka listener container thread — the physical thread changes and the logical
flow does not.

**Verdict: PARTIAL ANALOGUE.** The model transfers. The mechanism does not, and the
mechanism is where the bugs live.

### You have met this exact failure shape three times already

This is not a new problem; it is the *same* problem you have already drilled, in a third
costume.

| Topic | The thing that lives in a `ThreadLocal` | How it is lost | How it is restored |
|---|---|---|---|
| 56 | `SecurityContext` (the authenticated principal) | any thread hop | `DelegatingSecurityContextExecutor` |
| 120 | MDC (the correlation ID on log lines) | any thread hop | copy the MDC map into the task |
| **119** | **OTel `Context` (the active span)** | **any thread hop** | **wrap the executor or the task** |
| 108 | Reactor's `Context` | operator thread hops | `ContextView` + `ContextSnapshot` |

If you did Topic 120's drill, you have already watched a correlation ID vanish across an
`@Async` boundary. **This is the same drill with a different payload**, and the fix has the
same shape: something must capture the value on the submitting thread and restore it on the
executing thread.

The reason it gets its own `DIFFERENTIATOR` topic rather than a footnote in 120 is that the
consequences are worse. A missing correlation ID makes one log line harder to find. A
missing trace context breaks the *causal graph* across services, which is the only artefact
that can tell you which of six services caused a latency spike.

### The one sentence to take into an interview

**"We added the OTel agent, so we have tracing" is false in a specific, demonstrable way:
you have tracing for the code paths the agent knows how to instrument, and every
hand-rolled thread boundary in your own code is a silent hole.**

---

## What is this?

### The vocabulary, defined once

- **Span** — one timed operation with a name, a start and end timestamp, a kind
  (`SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`, `INTERNAL`), a status, attributes
  (key/value), and events (timestamped log-like entries).
- **Trace** — the tree of spans that share a trace ID. One trace ≈ one logical request.
- **Trace ID** — 16 random bytes, rendered as 32 hex characters. Generated at the root.
- **Span ID** — 8 random bytes, 16 hex characters. Unique per span.
- **Context** — the in-process carrier of "which span is currently active", plus baggage.
- **Propagator** — the thing that serialises context into a carrier (HTTP headers, Kafka
  headers) and parses it back out. The default is W3C Trace Context.
- **Baggage** — arbitrary key/value pairs propagated *alongside* the trace context to every
  downstream service. Powerful and dangerous; see the traps.
- **Sampler** — decides, per trace, whether spans are recorded and exported.
- **Exporter** — ships finished spans somewhere, usually over OTLP to a Collector.
- **Collector** — a separate process that receives, batches, processes (including **tail
  sampling**) and forwards spans to a backend.

### The `traceparent` header

*Illustration of the format, not captured output:*

```
traceparent: 00-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx-01
             |  |                                |                |
             |  |                                |                +-- flags: 01 = sampled
             |  |                                +------------------- parent span id (16 hex)
             |  +---------------------------------------------------- trace id (32 hex)
             +------------------------------------------------------- version (00)

tracestate: vendor1=value,vendor2=value        # optional, vendor-specific
baggage: tenant=retail,channel=web             # optional, separate header, separate spec
```

Two details that matter more than they look:

- **The flags byte carries the sampling decision.** `01` means "this trace is sampled".
  Every downstream service reads that and, under the default `parentbased` sampler,
  respects it. This is what makes a trace complete rather than a scatter of fragments —
  and it is also why the *first* service in a call chain effectively decides for everyone.
- **`baggage` is a different header and a different specification from `traceparent`.**
  Losing one does not imply losing the other, and they can be propagated by different
  propagators.

### What tracing is *for*, said precisely

Metrics (Topic 118) tell you **that** p99 is bad, aggregated over everything. Logs
(Topic 120) tell you **what happened** in one process. A trace tells you **where the time
went across process boundaries, for one specific request**.

That is the gap it fills, and it is a gap nothing else fills:

| Question | Signal |
|---|---|
| "Is order placement meeting its latency target?" | metric |
| "Which of the six services in this flow is slow?" | **trace** |
| "Why did *this* order take four seconds?" | **trace**, then logs joined by trace ID |
| "What exactly did the wallet service log while handling it?" | log, filtered by trace ID |
| "How often does this happen?" | metric |

The cross-references matter: **a trace ID on every log line (Topic 120) plus an exemplar
linking a metric to a trace is what turns three signals into one investigation.**

---

## Why does it matter?

**1. Because `orderflow` is no longer one process.**

By Topic 119 the order-placement flow is: HTTP ingress → order service → Postgres → wallet
debit → payment gateway (Topic 111) → outbox insert (Topic 115) → relay → Kafka → inventory
consumer → notification consumer. That is at minimum two processes and, in the deployment
Topic 124 reviews, several. **No single log file and no single metric contains the whole
request.** The trace is the only artefact that does.

**2. Because the Java-specific failure is silent and produces convincing wrong data.**

If your Kafka consumer starts a new trace, you will still see traces. They will still look
fine. You will conclude that order placement takes 40 ms and consumption takes 15 ms, and
you will never see the two-second gap between them where the outbox relay was sitting on a
stuck batch. **A broken trace does not look broken.** It looks like a smaller trace.

**3. Because every async boundary you added in Phases 9 and 10 is a place this breaks.**

You have `@Async` methods, `ThreadPoolTaskExecutor`s (Topic 90), `CompletableFuture`
composition (Topic 91), possibly virtual threads (Topic 101), Reactor chains (Topics
104–108), Kafka listener containers (Topic 113), and a scheduled outbox relay (Topic 115).
**Every one of those is a thread hop.** The agent instruments some of them. It cannot
instrument the executor you wrote yourself.

**4. Because the sampling decision you make on day one determines whether you have data on
the day you need it.**

Head sampling at 1% is the default advice, and it means that for any given incident there is
a 99% chance the request the customer complained about was never recorded. This is not a
subtle trade-off; it is the difference between tracing being useful and being decorative.

---

## Machine-level reality

### `Context`, `ContextStorage`, and `Scope`

The OpenTelemetry Java API's context mechanism is small enough to hold in your head, and
holding it is what makes the failures obvious rather than mysterious.

```java
// io.opentelemetry.context.Context — an IMMUTABLE map from ContextKey to value.
Context current = Context.current();          // reads the ThreadLocal; never null
Context withSpan = current.with(span);        // returns a NEW Context; nothing mutated

// Making it current is an explicit, scoped operation.
try (Scope scope = withSpan.makeCurrent()) {  // sets the ThreadLocal, returns a restorer
    doWork();                                  // Context.current() inside here sees `span`
}                                              // scope.close() RESTORES the previous value
```

Four mechanical facts that explain nearly every bug in this topic:

**1. `Context` is immutable.** `with()` returns a new instance. There is no "add to the
current context" operation — you always replace what is current, and later restore it.

**2. `ContextStorage` is a `ThreadLocal`, and it is deliberately *not* inheritable.**
The default implementation stores the current `Context` in a plain `ThreadLocal<Context>`.
It is **not** an `InheritableThreadLocal`. That is a deliberate design decision, and it is
the direct cause of the drill: a thread you create yourself starts at `Context.root()`, not
at its creator's context.

> Why deliberate? An inheritable thread-local would attach the creating request's span to a
> pool thread *permanently*, because pool threads are created once and reused forever. You
> would get every subsequent request parented to the first request that happened to trigger
> thread creation — which is a worse bug than losing the context, because it produces a
> trace that is confidently wrong rather than merely absent. Topic 79's `ThreadLocal`-leak
> shape, applied to traces.

**3. `Scope` is a restorer, and closing it is not optional.** `makeCurrent()` captures the
*previous* context and `close()` puts it back. Skip the close and the thread keeps that
context forever — on a pooled thread, that means the next request runs inside the previous
request's span. **Always use try-with-resources.** This is Topic 08's construct doing real
work.

**4. `Span` and `Scope` are independent.** `span.end()` stops the timer. `scope.close()`
restores the thread's context. Doing one without the other is a bug in each direction:

```java
Span span = tracer.spanBuilder("orderflow.reserve-inventory").startSpan();
try (Scope scope = span.makeCurrent()) {
    reserveInventory(order);
} catch (RuntimeException e) {
    span.recordException(e);                    // attaches an exception event
    span.setStatus(StatusCode.ERROR, e.getMessage());
    throw e;
} finally {
    span.end();                                 // MUST be in finally, and MUST be outside
}                                               // the try-with-resources' own close
```

| Mistake | Consequence |
|---|---|
| `makeCurrent()` without closing the `Scope` | context leaks onto a pooled thread; the next request is parented to this span |
| `startSpan()` without `end()` | the span is never exported; the trace has a hole and the exporter's buffer grows |
| `end()` before the work finishes | duration is wrong; child spans may be orphaned |
| `end()` inside the `try` rather than `finally` | any exception path leaves the span unended |

### The propagation API: `inject` and `extract`

Crossing a process boundary is two explicit calls, and understanding them is what lets you
fix Kafka, or any transport the agent does not know about.

```java
// OUTBOUND — serialise the current context into a carrier.
TextMapPropagator propagator = openTelemetry.getPropagators().getTextMapPropagator();

propagator.inject(Context.current(), record.headers(),
        (headers, key, value) -> headers.add(key, value.getBytes(UTF_8)));

// INBOUND — parse the carrier back into a Context, then make it the parent.
Context extracted = propagator.extract(Context.current(), record.headers(), GETTER);

Span span = tracer.spanBuilder("orderflow.inventory.consume")
        .setSpanKind(SpanKind.CONSUMER)
        .setParent(extracted)                    // <-- the join
        .startSpan();
```

The `TextMapSetter` and `TextMapGetter` are the adapters between OTel and *your* carrier
type. For HTTP the instrumentation supplies them. For Kafka headers, a custom protocol, a
database row, or a message on a queue you invented, **you supply them** — and if you do not,
the context does not cross.

**`setParent(extracted)` versus `.makeCurrent()` on the extracted context:** both work; the
first is explicit and preferable at a boundary, because it does not require you to hold a
`Scope` open across span creation.

### Where the JVM's spans come from: agent, starter, or hand-written

Three mechanisms, and you should know which ones are active in your service — because
running two of them produces duplicate spans.

**1. The Java agent (`opentelemetry-javaagent.jar`).**
Attached with `-javaagent:`, it uses the instrumentation API from **Topic 81** to rewrite
bytecode at class-load time. It knows hundreds of libraries: servlet containers, JDBC,
Hibernate, HTTP clients, Kafka clients, Redis clients, executors. **It requires zero code
changes**, which is its entire value proposition and also the source of the false
confidence in the interview question below.

Note the important one for this topic: the agent *does* instrument `Executors`-created
executors and `ThreadPoolExecutor`, wrapping submitted tasks so context propagates. What it
cannot instrument is an executor implementation it does not recognise, a raw `new Thread()`,
or a task you hand off through a queue you wrote yourself.

**2. The Spring Boot starter.**
Boot 4 ships a dedicated **OpenTelemetry starter**, which wires the SDK, the OTLP exporter
and Spring's own instrumentation as beans rather than through bytecode rewriting. This is
the in-process route: no `-javaagent` flag, configuration through normal Spring properties,
and instrumentation that knows about Spring's abstractions.

> **`[BOOT 3.x DELTA]`** On Boot 3.x there was no first-party OTel starter. The standard
> wiring was **Micrometer Tracing** as the facade plus a bridge to an implementation:
> `io.micrometer:micrometer-tracing-bridge-otel` (or `-brave`) together with
> `io.opentelemetry:opentelemetry-exporter-otlp`, with sampling configured through
> `management.tracing.sampling.probability`. Spans were created through Micrometer's
> `Observation` API and translated to OTel spans by the bridge. Code written against
> `ObservationRegistry` is portable across both worlds; code written against
> `io.opentelemetry.api.trace.Tracer` is not, but is more direct.
>
> **Flagged uncertainty, one line:** I am not certain of the exact artifact coordinates and
> property names of the Boot 4.1 OpenTelemetry starter, and I will not guess them. Resolve
> them from `start.spring.io` for your exact Boot version, or from the dependency list in
> the Boot BOM (`./mvnw dependency:tree | grep -i opentelemetry`), and confirm the
> properties with `/actuator/configprops`.

**3. Hand-written spans**, using `Tracer` directly or Micrometer's `Observation` API.
Necessary for your own business operations — "reserve inventory", "debit wallet", "publish
outbox batch" — which no agent can name for you.

**The rule: pick one automatic mechanism, not both.** Agent *and* starter both instrumenting
the same servlet filter produces two SERVER spans per request. That is not a subtle
duplicate; it doubles your span volume and confuses every latency breakdown.

### Instrumented executor wrappers — the mechanism of the fix

The fix to a lost context is always the same shape: capture on submit, restore on execute,
restore-the-previous on completion. Here it is, unwrapped, so you can see there is no magic:

```java
// This is essentially what Context.taskWrapping(...) does for you.
static Runnable wrap(Runnable task) {
    Context captured = Context.current();                 // on the SUBMITTING thread
    return () -> {
        try (Scope scope = captured.makeCurrent()) {      // on the EXECUTING thread
            task.run();
        }                                                 // restore whatever was there
    };
}
```

Three levels at which you can apply it, in increasing order of how much you can forget
about it afterwards:

```java
// 1. Per task — precise, and easy to forget on the next call site.
executor.submit(Context.current().wrap(() -> reserveInventory(order)));

// 2. Per executor — every task submitted through it is wrapped. Prefer this.
ExecutorService traced = Context.taskWrapping(rawExecutor);

// 3. Per Spring executor — a TaskDecorator, so @Async and every
//    ThreadPoolTaskExecutor consumer gets it without call-site changes.
@Bean
ThreadPoolTaskExecutor orderflowExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(500);                       // Topic 90: bounded
    executor.setThreadNamePrefix("orderflow-async-");
    executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
    executor.initialize();
    return executor;
}
```

`ContextPropagatingTaskDecorator` (Spring Framework, `org.springframework.core.task.support`)
propagates *everything* registered with the `io.micrometer:context-propagation` library —
trace context, MDC (Topic 120), and `SecurityContext` (Topic 56) — in one decorator. That is
usually what you want, because losing any one of the three is a bug and fixing them
separately is three chances to forget one.

> **Flagged uncertainty:** `ContextPropagatingTaskDecorator` arrived in the Spring Framework
> 6.1 line. I believe it is present and unchanged in Framework 7.0, but verify by import
> rather than trusting me; if it is absent, the three-line hand-rolled `TaskDecorator` that
> calls `ContextSnapshotFactory` directly is a complete substitute.

**The Reactor case (Topic 108)** is different, because there is no executor to wrap — the
chain hops threads inside operators:

```java
// Once, at startup. Makes Reactor restore ThreadLocals (including OTel context and MDC)
// around operator execution, using the context-propagation library's registry.
Hooks.enableAutomaticContextPropagation();
```

**The virtual-thread case (Topic 101)** is the one people get wrong in both directions, so
state it precisely:

- A virtual thread **has its own `ThreadLocal`s**, and they work normally. `Context.current()`
  inside a virtual thread returns whatever was made current *on that virtual thread*.
- A virtual thread does **not** inherit the creating thread's OTel context, for exactly the
  same reason a platform thread does not: the storage is a non-inheritable `ThreadLocal`.
- So `Executors.newVirtualThreadPerTaskExecutor()` has **precisely the same context-loss
  problem** as a fixed platform pool. "We moved to virtual threads" fixes nothing here.
- The fix is identical: `Context.taskWrapping(...)` or a decorator.

`[JAVA 25]` **Scoped values** (Topic 102) are the JDK's answer to inheritance across
structured concurrency — a `ScopedValue` is visible to threads forked inside a
`StructuredTaskScope`. That is a much better fit for trace context than a `ThreadLocal`.
**I do not know whether the OTel Java SDK uses scoped values for context storage in the
version you are running, and I am not going to guess.** Check with:

```bash
./mvnw dependency:tree | grep opentelemetry
# then look at the ContextStorage implementation your version registers:
java -Dotel.javaagent.debug=true -jar app.jar 2>&1 | grep -i contextstorage
```

Until that changes, assume `ThreadLocal` semantics and wrap your boundaries.

### Sampling, mechanically

The sampler runs **once per trace, at the root**, and its decision is encoded in the
`traceparent` flags byte and honoured by every downstream service under the default
`parentbased` sampler.

| Sampler | Behaviour | When it is right |
|---|---|---|
| `always_on` | record everything | local development; very low traffic |
| `always_off` | record nothing | disabling without removing the dependency |
| `traceidratio` | record a fixed fraction, decided from the trace ID | almost never on its own — it ignores the parent's decision and fragments traces |
| `parentbased_always_on` | follow the parent; sample if root | default in many setups |
| `parentbased_traceidratio` | follow the parent; ratio-sample if root | **the standard head-sampling choice** |

```properties
# Environment-variable form (works for the agent and the SDK).
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.05
OTEL_SERVICE_NAME=orderflow
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_PROPAGATORS=tracecontext,baggage
```

**Head sampling** decides at the start of the trace, before anything has happened. It is
cheap and it is blind: the decision cannot depend on whether the request turned out to be
slow or to fail, because neither is known yet.

**Tail sampling** decides after the trace is complete, in the Collector, based on what
actually happened — keep everything with an error, everything over a latency threshold, and
a small random sample of the rest. It costs more (the Collector must buffer all spans of a
trace until it decides) and it gives you the traces you actually want.

The Collector-side shape, so you know what you are asking a platform team for:

```yaml
# otel-collector-config.yaml — the processor, illustrative of the shape
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 50000
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }
```

**The critical prerequisite, and the thing people miss:** for tail sampling to see complete
traces, the services must *record* the spans in the first place. So the application-side
sampler must be `always_on` (or a high ratio), and the Collector does the reduction. If you
head-sample at 1% and then tail-sample in the Collector, the Collector can only choose among
the 1% it received — you have combined the costs of both and the benefits of neither.

### The export path, and why it can lose spans

Spans do not go straight out. They go into a `BatchSpanProcessor`: a bounded in-memory queue
drained by a background thread that ships batches over OTLP.

```
span.end() → BatchSpanProcessor queue (bounded) → exporter thread → OTLP → Collector → backend
```

Consequences worth knowing before an incident:

- **The queue is bounded, and it drops when full.** Under a burst, or when the Collector is
  slow, spans are silently discarded. The SDK exposes counters for this; find and graph them
  (Topic 118's job).
- **Export is asynchronous**, so `span.end()` does not block your request. Good for latency,
  and it means a span can be lost after you thought it was recorded.
- **Shutdown must flush.** If the JVM exits without the processor draining, the last batch is
  lost — which is exactly the traces from the moment things went wrong. This is Topic 123's
  problem, and it is why the shutdown ordering there includes flushing the exporter.

---

## Example 1 — minimal

One HTTP endpoint, one manual child span, one downstream call, so you can see a trace ID
propagate and appear on your log lines.

**Configuration** (environment-variable form, which works identically for the agent and the
in-process SDK):

```bash
export OTEL_SERVICE_NAME=orderflow
export OTEL_TRACES_SAMPLER=always_on           # local only; never in production
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_PROPAGATORS=tracecontext,baggage
```

**The code:**

```java
package com.orderflow.catalog;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

@RestController
class ProductController {

    private static final Logger log = LoggerFactory.getLogger(ProductController.class);

    private final Tracer tracer;
    private final ProductRepository repository;

    ProductController(OpenTelemetry openTelemetry, ProductRepository repository) {
        this.tracer = openTelemetry.getTracer("com.orderflow.catalog");
        this.repository = repository;
    }

    @GetMapping("/api/products/{sku}")
    public ProductView get(@PathVariable String sku) {
        // The SERVER span already exists — the servlet instrumentation created it.
        // This is a CHILD span for the part we care about.
        Span span = tracer.spanBuilder("catalog.lookup")
                .setSpanKind(SpanKind.INTERNAL)
                .setAttribute("orderflow.sku", sku)      // fine on a span; forbidden on a metric
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            log.info("looking up product");              // this line now carries traceId
            return repository.findBySku(sku).map(ProductView::from)
                    .orElseThrow(() -> new ProductNotFoundException(sku));
        } catch (RuntimeException e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getClass().getSimpleName());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Note the contrast with Topic 118, stated explicitly because it is the most useful rule in
this pair of documents:**

> `sku` as a **metric tag** is a cardinality bomb — 100,000 time series.
> `sku` as a **span attribute** is exactly right — spans are stored per-event, not per-series,
> and the attribute is what lets you find *this* request.
>
> **High-cardinality identity belongs on traces and logs. Never on metrics.**

**What to run:**

```bash
# 1. Let the service originate the trace.
curl -s localhost:8080/api/products/SKU-1001 > /dev/null

# 2. Now supply your own trace context and prove it is honoured.
#    Any 32-hex / 16-hex pair works; these are placeholders you replace with hex digits.
curl -s -H 'traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01' \
     localhost:8080/api/products/SKU-1001 > /dev/null

# 3. Find the log line for that request.
docker compose logs app | grep 4bf92f3577b34da6a3ce929d0e0e4736
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| The log line contains the **same** trace ID you sent in the header | Extraction works; the SERVER span joined your trace rather than starting a new one |
| The log line contains a **different** trace ID | The propagator is not configured, the header name is wrong, or something upstream stripped it |
| No `traceId` field on the log line at all | The MDC bridge is not wired — see Topic 120's `logging.pattern.correlation` and the tracing bridge |
| A trace with one span in the UI | The `catalog.lookup` child span is missing — check that `end()` is being called |
| A trace with two spans, parented correctly | Correct. This is your working baseline for the drill |

*Illustration of the log line format, not captured output:*

```json
{"@timestamp":"<iso8601>","level":"INFO","logger":"com.orderflow.catalog.ProductController",
 "thread":"http-nio-8080-exec-<n>","service":"orderflow",
 "traceId":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx","spanId":"xxxxxxxxxxxxxxxx",
 "message":"looking up product"}
```

**The `traceId` on the log line is the single highest-value integration in this document.**
It is what turns "I have a slow trace" into "here are the 40 log lines that request
produced, across three services", and it costs one configuration line (Topic 120).

---

## Example 2 — production scenario (on the project spine)

### The constraints

- `orderflow` at the Topic 65 baseline: 100k products, 1M orders, 5M order lines, the
  70/20/10 k6 mix under a constant-arrival-rate open model.
- **Multiple processes.** The order service, the inventory consumer, the notification
  consumer, and the outbox relay. Some are separate deployments.
- **Every async boundary from Phases 9–11 is present:** a `ThreadPoolTaskExecutor` (Topic
  90), `CompletableFuture` composition on the payment path (Topic 91), a Reactor chain for
  the payment-callback fan-in (Topic 108), a Kafka listener container (Topic 113), and a
  scheduled outbox relay (Topic 115).
- **Trace volume is a real cost.** At the baseline request rate, `always_on` sampling with
  full export is a meaningful amount of network and storage. The sampling decision is a
  budget decision.
- **The gateway is external** (Topic 111) and does not participate in your trace.

### Step 1 — decide, and write down, where spans come from

Make this an explicit decision, because the failure mode of *not* deciding is duplicate
instrumentation.

| Layer | Who instruments it | Deliberate? |
|---|---|---|
| HTTP server (Tomcat/servlet) | agent **or** Boot starter — **exactly one** | yes |
| JDBC / Hibernate | agent, or the Boot starter's datasource instrumentation | yes |
| Kafka producer/consumer client | agent, or manual inject/extract | **decide per boundary** |
| HTTP client (`RestClient`/`WebClient`) | agent or starter | yes |
| Redis | agent | yes |
| **Business operations** (reserve inventory, debit wallet, publish outbox batch) | **you, by hand** | yes |
| **Your executors** (Topic 90) | **you, by decorator** | yes |

Verify the decision rather than assuming it:

```bash
# Is an agent attached?
jcmd $(pgrep -f orderflow) VM.command_line | tr ' ' '\n' | grep -i javaagent

# Is the in-process SDK also present?
./mvnw dependency:tree | grep -E 'opentelemetry|micrometer-tracing'
```

If both are present, expect duplicate spans; pick one and remove the other.

### Step 2 — the executor fix, applied once, centrally

Every executor in the service goes through one factory. This is the difference between
fixing the bug and fixing the *class* of bug (Topic 133's distinction).

```java
package com.orderflow.observability;

import org.springframework.core.task.support.ContextPropagatingTaskDecorator;

@Configuration
class TracedExecutorConfiguration {

    /**
     * Every ThreadPoolTaskExecutor in orderflow is built here.
     * The decorator propagates OTel context, MDC (Topic 120) and SecurityContext
     * (Topic 56) together — because losing any one of them is a bug, and fixing
     * them one at a time is three chances to forget one.
     */
    private static ThreadPoolTaskExecutor traced(
            String namePrefix, int size, int queueCapacity) {

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(size);
        executor.setMaxPoolSize(size);
        executor.setQueueCapacity(queueCapacity);              // BOUNDED — Topic 90
        executor.setThreadNamePrefix(namePrefix);
        executor.setTaskDecorator(new ContextPropagatingTaskDecorator());
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);    // Topic 123
        executor.setAwaitTerminationSeconds(20);               // Topic 123
        executor.initialize();
        return executor;
    }

    @Bean("notificationExecutor")
    ThreadPoolTaskExecutor notificationExecutor() {
        return traced("orderflow-notify-", 8, 500);
    }

    @Bean("reconciliationExecutor")
    ThreadPoolTaskExecutor reconciliationExecutor() {
        return traced("orderflow-recon-", 4, 100);
    }

    /**
     * For anything that needs a raw ExecutorService rather than Spring's abstraction.
     * Context.taskWrapping is the OTel-native equivalent and covers trace context only.
     */
    @Bean
    ExecutorService outboxRelayExecutor() {
        ThreadPoolExecutor raw = new ThreadPoolExecutor(
                2, 2, 0L, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<>(50),
                new CustomizableThreadFactory("orderflow-relay-"));
        return Context.taskWrapping(raw);
    }
}
```

**And the rule that makes it stick:** `new ThreadPoolExecutor(...)` and `new Thread(...)`
outside this class are a code-review rejection. If it is not reviewable, make it a static
analysis rule (Topic 132's kind of gate). A rule that depends on someone remembering is not
a fix.

### Step 3 — Kafka: the boundary the agent may or may not cover

The producer and consumer are **different processes**. There is no shared memory, no shared
`ThreadLocal`, nothing but the bytes of the record. The context must travel in the record
headers, which is a serialisation step someone has to perform.

Modern OTel Kafka instrumentation does this automatically, and the Spring Kafka integration
generally does too. **Verify it rather than assuming it**, and know how to do it by hand,
because the moment you write a custom relay (which Topic 115 made you do) you own the
boundary.

```java
package com.orderflow.outbox;

/**
 * The outbox relay (Topic 115). It reads rows written by an HTTP request that finished
 * some time ago, possibly on a different pod. There is NO ambient context here — the
 * originating request is long gone.
 *
 * So: the trace context is stored IN THE OUTBOX ROW at insert time, and restored here.
 * This is the only way to connect the publish to the request that caused it.
 */
@Component
class OutboxRelay {

    private final Tracer tracer;
    private final TextMapPropagator propagator;
    private final KafkaTemplate<String, byte[]> kafka;
    private final OutboxRepository repository;
    private final LongTaskTimer batchTimer;                   // Topic 118

    @Scheduled(fixedDelay = 500)
    public void publishBatch() {
        LongTaskTimer.Sample sample = batchTimer.start();
        try {
            List<OutboxRow> batch = repository.claimBatch(100);   // SELECT ... FOR UPDATE SKIP LOCKED
            for (OutboxRow row : batch) {
                publishOne(row);
            }
        } finally {
            sample.stop();
        }
    }

    private void publishOne(OutboxRow row) {
        // 1. Rebuild the originating request's context from the columns we stored.
        Context origin = propagator.extract(Context.root(), row.traceHeaders(), MAP_GETTER);

        // 2. Start a PRODUCER span parented to it. The trace now spans the request,
        //    the commit, the delay in the outbox, and the publish.
        Span span = tracer.spanBuilder("orderflow.outbox.publish")
                .setSpanKind(SpanKind.PRODUCER)
                .setParent(origin)
                .setAttribute("messaging.system", "kafka")
                .setAttribute("messaging.destination.name", row.topic())
                .setAttribute("orderflow.outbox.id", row.id().toString())
                .setAttribute("orderflow.outbox.age_ms", row.ageMillis())   // the useful one
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            ProducerRecord<String, byte[]> record =
                    new ProducerRecord<>(row.topic(), row.key(), row.payload());

            // 3. Inject the CURRENT context (this producer span) into the record headers,
            //    so the consumer in another process can become its child.
            propagator.inject(Context.current(), record.headers(),
                    (headers, key, value) -> headers.add(key, value.getBytes(UTF_8)));

            kafka.send(record).get(5, TimeUnit.SECONDS);
            repository.markPublished(row.id());
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "outbox publish failed");
            throw new OutboxPublishException(row.id(), e);
        } finally {
            span.end();
        }
    }
}
```

The insert side, which is the half people forget:

```java
// In the SAME transaction as the order write (Topic 115's whole point).
Map<String, String> carrier = new HashMap<>();
propagator.inject(Context.current(), carrier, Map::put);   // captures traceparent (+ baggage)

outboxRepository.insert(new OutboxRow(
        UUID.randomUUID(),
        "orderflow.order.placed",
        order.id().toString(),
        payload,
        carrier));                                          // stored as a jsonb column
```

**`orderflow.outbox.age_ms` is the attribute that earns its keep.** It is the gap between
the row being written and being published — the exact latency the outbox pattern
deliberately introduces, and the number that tells you whether the relay is keeping up. On
the trace it appears as a visible gap between the transaction span and the producer span,
which is far more legible than any metric.

The consumer side, in the other process:

```java
@KafkaListener(topics = "orderflow.order.placed", groupId = "inventory")
public void onOrderPlaced(ConsumerRecord<String, byte[]> record, Acknowledgment ack) {
    Context extracted = propagator.extract(Context.root(), record.headers(), HEADER_GETTER);

    Span span = tracer.spanBuilder("orderflow.inventory.apply")
            .setSpanKind(SpanKind.CONSUMER)
            .setParent(extracted)
            .setAttribute("messaging.system", "kafka")
            .setAttribute("messaging.kafka.partition", record.partition())
            .setAttribute("messaging.kafka.offset", record.offset())
            .startSpan();

    try (Scope scope = span.makeCurrent()) {
        inventoryService.apply(deserialize(record.value()));
        ack.acknowledge();
    } catch (RuntimeException e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getClass().getSimpleName());
        throw e;
    } finally {
        span.end();
    }
}
```

**A design note worth understanding, because an interviewer may push on it:** parenting the
consumer span to the producer span makes one long trace. That is right for a single-message
flow like this one. For a *batch* consumer that processes 500 messages from 500 different
traces in one poll, a parent relationship is wrong — you would be claiming one of the 500
traces caused all the work. The correct modelling there is a **span link**: the batch span
links to all 500 originating spans without claiming parentage. Know that links exist and
what they are for.

### Step 4 — sampling that gives you the incident trace

```properties
# APPLICATION: record everything. Let the Collector reduce.
OTEL_TRACES_SAMPLER=parentbased_always_on
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_BSP_MAX_QUEUE_SIZE=4096
OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512
```

Plus the Collector's `tail_sampling` policy set from Machine-level reality: keep errors, keep
anything slower than your SLO boundary, keep a small probabilistic baseline for
everything else.

**If you cannot run tail sampling** — no Collector, or a platform team that says no — the
honest fallback is *not* 1% head sampling. It is:

1. `parentbased_traceidratio` at the highest ratio your budget allows, **plus**
2. a `Sampler` override that forces sampling for requests you already know are interesting:
   an explicit debug header, a specific customer under investigation, or a route you are
   actively working on. This lets support say "reproduce it with this header" and get a
   guaranteed trace.

State the residual honestly in the Topic 124 review: with head sampling, the probability
that any *given* customer complaint has a trace is the sampling ratio. That is a known,
quantified gap, not an oversight.

### Step 5 — connect the three signals

```properties
# Topic 120: trace and span IDs on every log line.
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

And the third link, which is the one most teams never wire: **exemplars**. An exemplar
attaches a trace ID to a specific histogram bucket observation, so a Grafana user can click
the spike in the p99 panel and land on a trace *from that spike*.

> **Flagged uncertainty:** exemplar support depends on the Micrometer version, the
> Prometheus registry, and Prometheus itself being started with the exemplar-storage feature
> flag. I am not certain of the enabling configuration on the Boot 4.1 line and will not
> guess it. Confirm by checking whether your scrape output contains exemplar suffixes:
> `curl -s localhost:8080/actuator/prometheus | grep ' # {'` — the `# {trace_id="..."}`
> suffix after a bucket line is the exemplar syntax. If it is absent, the manual join
> (metric → time window → trace search by service and duration) still works and is what
> most teams actually do.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a plain `ExecutorService` in a traced request

**Wrong approach**

```java
// A perfectly ordinary-looking service. Reviewed, merged, shipped.
@Service
public class OrderPlacementService {

    private final ExecutorService notifications =
            Executors.newFixedThreadPool(8);        // created by hand, not injected

    public OrderResult place(PlaceOrderCommand command) {
        Order order = persistOrder(command);        // inside the request span
        notifications.submit(() -> notifier.send(order));   // <-- the boundary
        return OrderResult.accepted(order.id());
    }
}
```

**Exact symptom**

- In Jaeger/Tempo, searching for the order-placement trace returns a trace containing the
  HTTP server span, the JDBC spans, and **nothing about the notification**.
- Searching for the notification span returns a **separate, single-span trace** with a
  different trace ID and no parent.
- The log lines from the notification thread carry a **different `traceId`** than the log
  lines from the request — or none at all, if MDC was lost too (it was).
- No exception, no warning, no log message anywhere says this happened.
- The end-to-end latency in your trace is *understated*, because the notification's time is
  in a trace nobody is looking at.

**Root cause**

`Context.current()` reads a non-inheritable `ThreadLocal`. The task submitted to the pool
runs on a pool thread whose `ThreadLocal` was never set, so `Context.current()` returns
`Context.root()`. A span started from root has no parent, so the SDK generates a **new trace
ID**. From the SDK's point of view nothing is wrong: you asked for a root span and you got
one.

Note the aggravating factor specific to a hand-created pool: the OTel agent *does*
instrument executors it recognises, so this bug sometimes does not reproduce in a service
where the agent is attached — until someone uses an executor shape the agent does not cover,
or a `new Thread()`, or a hand-rolled work queue. **"It works with the agent" is not the
same as "the code is correct."**

**Fix**

```java
@Service
public class OrderPlacementService {

    // Injected, built by the central factory from Example 2 step 2, decorated.
    private final ThreadPoolTaskExecutor notifications;

    public OrderResult place(PlaceOrderCommand command) {
        Order order = persistOrder(command);
        notifications.execute(() -> notifier.send(order));   // context now propagates
        return OrderResult.accepted(order.id());
    }
}
```

Or, if you must keep a raw executor:

```java
private final ExecutorService notifications =
        Context.taskWrapping(Executors.newFixedThreadPool(8));
```

**And prove it, because this is the class of bug that comes back:**

```java
@SpringBootTest
class ContextPropagationTest {

    @Autowired ThreadPoolTaskExecutor notificationExecutor;

    @Test
    void executorPropagatesTraceContext() throws Exception {
        Span parent = tracer.spanBuilder("test-parent").startSpan();
        AtomicReference<String> childTraceId = new AtomicReference<>();

        try (Scope scope = parent.makeCurrent()) {
            notificationExecutor.submit(() ->
                    childTraceId.set(Span.current().getSpanContext().getTraceId())
            ).get(5, TimeUnit.SECONDS);
        } finally {
            parent.end();
        }

        assertThat(childTraceId.get())
                .as("the task must run inside the submitting thread's trace")
                .isEqualTo(parent.getSpanContext().getTraceId());
    }
}
```

---

### Trap 2 — `makeCurrent()` without closing the `Scope`

**Wrong approach**

```java
// Someone found try-with-resources noisy and "simplified" it.
Span span = tracer.spanBuilder("orderflow.reserve-inventory").startSpan();
span.makeCurrent();                        // <-- Scope created and thrown away
reserveInventory(order);
span.end();                                // span ends; the THREAD's context does not reset
```

**Exact symptom**

- Traces are not missing — they are **wrong**. Request B's spans appear as children of
  request A's span, in request A's trace.
- The corruption is *sticky per thread*: it affects every subsequent request that lands on
  the same Tomcat worker or pool thread, so it looks intermittent and correlates with thread
  count rather than with anything in the code.
- Traces grow monotonically. A trace that should have eight spans has hundreds, accumulating
  over minutes. Your tracing backend starts rejecting or truncating them.
- Log lines carry a **stale** `traceId` — one that belongs to a request that finished long
  ago.
- Under low load with a fresh JVM it does not reproduce, because each request tends to get a
  fresh thread.

**Root cause**

`makeCurrent()` sets the thread's `ThreadLocal` and returns a `Scope` that remembers the
*previous* value. `Scope.close()` is the only thing that restores it. Discard the `Scope` and
the thread is permanently pinned to that context. `span.end()` does not help — ending a span
and un-currenting it are different operations on different objects.

This is Topic 79's `ThreadLocal`-leak shape exactly: state written to a pooled thread and
never removed, inherited by the next unit of work.

**Fix**

Always try-with-resources, always `end()` in `finally`:

```java
Span span = tracer.spanBuilder("orderflow.reserve-inventory").startSpan();
try (Scope scope = span.makeCurrent()) {
    reserveInventory(order);
} finally {
    span.end();
}
```

**Better: stop writing this by hand at all.** For business operations, use an annotation or
Micrometer's `Observation` API so the scope discipline is the framework's problem:

```java
@WithSpan("orderflow.reserve-inventory")     // OTel annotation; agent or SDK instrumentation
public void reserveInventory(@SpanAttribute("orderflow.order.id") String orderId, Order order) {
    // scope handling is generated; there is no Scope to leak
}
```

**Detect it cheaply**, because manual spans will exist somewhere:

```java
// A filter at the very end of the chain. If the context did not reset, something leaked.
@Component
@Order(Ordered.LOWEST_PRECEDENCE)
class ContextLeakDetectionFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws Exception {
        try {
            chain.doFilter(req, res);
        } finally {
            if (Span.current().getSpanContext().isValid()) {
                // Enable in test/staging. In production, count it as a metric instead.
                logger.warn("trace context leaked onto thread {}",
                        Thread.currentThread().getName());
            }
        }
    }
}
```

---

### Trap 3 — head sampling at 1%

**Wrong approach**

```properties
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.01        # "1% is the standard, it keeps costs down"
```

**Exact symptom**

- Tracing works. Dashboards show traces. Everyone believes the service is instrumented.
- A customer reports a four-second order placement with an order ID and a timestamp. You
  search the tracing backend and **there is no trace for it**. There is a 99% chance there
  never was.
- The traces you *do* have are, by construction, a uniform random sample — which means they
  are overwhelmingly the fast, successful, boring ones. Errors are rare, so a uniform sample
  of a rare thing is almost nothing.
- Engineers stop opening the tracing UI, because it never has the request they are looking
  for. The tool is paid for and unused.

**Root cause**

The head sampler decides at the root of the trace, before any work has happened. It cannot
condition on outcome, because there is no outcome yet. A uniform random sample retains 1% of
the interesting traces and 1% of the boring ones, and since interesting traces are the rare
ones, you end up with essentially none.

**Fix**

Move the decision to where the outcome is known:

1. **Application: record everything.** `parentbased_always_on`.
2. **Collector: tail-sample.** Keep 100% of errors, 100% of anything over the SLO latency
   boundary, and a small probabilistic baseline of the rest.

That gives you every trace you would ever want to look at, plus enough baseline to compare
against, at a fraction of "keep everything" cost.

**If tail sampling is not available:**

```java
// A composite sampler: force-sample when a caller asks, ratio-sample otherwise.
final class DebugHeaderSampler implements Sampler {
    private final Sampler delegate;

    @Override
    public SamplingResult shouldSample(Context parent, String traceId, String name,
                                       SpanKind kind, Attributes attrs, List<LinkData> links) {
        if ("true".equals(Baggage.fromContext(parent).getEntryValue("orderflow.debug"))) {
            return SamplingResult.recordAndSample();
        }
        return delegate.shouldSample(parent, traceId, name, kind, attrs, links);
    }

    @Override public String getDescription() { return "orderflow-debug-or-" + delegate; }
}
```

Now support can say "retry with `baggage: orderflow.debug=true`" and get a guaranteed trace.

**Do the arithmetic honestly and write it in the Topic 124 review:** the probability that
any given complaint has a trace equals the sampling ratio. At 1%, in a hundred complaints
you can investigate one. Say the number; do not say "we have tracing".

---

### Trap 4 — "we added the OTel agent, so we have tracing"

**Wrong approach**

```yaml
# The whole tracing story, as far as the team is concerned.
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/otel/opentelemetry-javaagent.jar"
  - name: OTEL_SERVICE_NAME
    value: "orderflow"
```

…and no further thought, no spans in the application code, no verification.

**Exact symptom** — the honest inventory of what you have and have not got:

| Path | Traced? | Why |
|---|---|---|
| HTTP in, JDBC, Hibernate, Redis, `RestClient` | yes | the agent knows these libraries |
| Standard `Executors` pools | usually | the agent wraps recognised executors |
| **Your hand-rolled `ThreadPoolExecutor` subclass / custom `Executor`** | **no** | the agent does not recognise it |
| **`new Thread(...)` anywhere** | **no** | nothing to wrap |
| **Reactor operator hops** (Topic 108) | **partially** | needs `Hooks.enableAutomaticContextPropagation()` |
| **Virtual threads you create yourself** (Topic 101) | **no** | non-inheritable `ThreadLocal`, same as platform threads |
| **The outbox relay** (Topic 115) | **no** | the originating request is gone; context must come from the row |
| **Business operations** ("reserve inventory") | **no** | the agent cannot know what your domain calls things |
| A trace for the request that actually failed | **only if sampled** | orthogonal to the agent entirely |

The visible symptom is a trace that is technically present and diagnostically useless: a
SERVER span, a handful of JDBC spans, and a flat 300 ms of unexplained time where all the
interesting work happened.

**Root cause**

The agent instruments *libraries*, by matching known class and method signatures at
class-load time (Topic 81's mechanism). Your code is not a known library. Everything you
wrote yourself — every thread boundary, every domain operation, every custom transport — is
outside its knowledge by construction.

**Fix**

Treat the agent as the floor, not the ceiling, and produce a written coverage table:

1. Enumerate every async boundary in the service (search for `new Thread`, `Executors.`,
   `ThreadPoolExecutor`, `@Async`, `CompletableFuture.*Async`, `.subscribeOn`,
   `.publishOn`, `@KafkaListener`, `@Scheduled`).
2. For each, write down whether context propagates, and **verify by test**, not by reading.
3. Add spans for the business operations that appear in your own vocabulary — the ones you
   would name in an incident channel.
4. Ban un-decorated executors at review time and enforce it with the central factory.

That table is a required section in Topic 124's review, and building it is Exercise 2.

---

### Trap 5 — trusting an inbound `traceparent`, and unbounded baggage

**Wrong approach**

The default configuration, on a service whose ingress is reachable by the public internet or
by any client you do not control — plus a well-intentioned addition:

```java
// "It'd be so useful to have the customer's details available in every downstream service."
Baggage.current().toBuilder()
        .put("customer.email", order.customerEmail())
        .put("customer.name", order.customerName())
        .put("cart.contents", serialize(order.lines()))     // <-- oh no
        .build()
        .makeCurrent();
```

**Exact symptom**

Two distinct failures, both real:

*Trust:* a client sends `traceparent` with the sampled flag set on every request. Under
`parentbased`, your service honours it, so an attacker (or an over-enthusiastic load test)
can force 100% sampling and drive your trace volume and Collector cost arbitrarily high.
More subtly, a client sending a **fixed** trace ID collapses thousands of unrelated requests
into one enormous trace, which your backend will truncate or reject — destroying the traces
of legitimate users caught in it.

*Baggage:* every baggage entry is serialised into an HTTP header on **every** outbound call
in the entire downstream call graph. Symptoms in order of appearance: header size grows;
some proxy or server rejects the request with 431 or 400 once the header limit is exceeded;
and — the serious one — PII is now present in the headers of every service, every access log,
and every span in your tracing backend, which is very likely outside whatever data-handling
boundary your compliance story assumes.

**Root cause**

`traceparent` is an unauthenticated client-supplied header, and `parentbased` sampling means
you honour a decision made by someone else. Baggage is *designed* to propagate everywhere,
so anything you put in it goes everywhere, including places you have not thought about.

**Fix**

At the trust boundary — your public ingress:

```
# Strip inbound trace headers at the edge for untrusted clients, so the trace starts
# in a system you control. Keep them for internal service-to-service calls.
# (Ingress/proxy configuration — the exact syntax depends on your ingress.)
more_clear_input_headers "traceparent" "tracestate" "baggage";
```

Or, in the application, a sampler that ignores a remote parent's decision for externally
originated requests while still joining the trace, so you keep the causal link without
handing over the sampling budget.

For baggage, three rules:

1. **Identifiers only, never content.** `tenant`, `channel`, `experiment` — yes.
   `email`, `name`, cart contents — no.
2. **A fixed, reviewed allow-list of keys**, enforced in code, not in a wiki.
3. **A size budget**, checked in a test, because header limits fail at the proxy and the
   error message will not mention baggage.

And say the general rule out loud, because it is the same rule as Topic 118's and Topic 120's
with a different noun: **a span attribute is stored once, per span; a baggage entry is
transmitted on every request in the call graph, forever.** They cost completely different
amounts. Put things on the span.

---

## Hands-on proof

### Setup

```bash
docker compose -f load/docker-compose.yml up -d --wait
# Add an OTel Collector and a trace backend (Jaeger all-in-one or Tempo) to the stack.
docker compose -f load/docker-compose.yml ps
```

### Proof 1 — is anything actually instrumenting this JVM?

```bash
# Agent attached?
jcmd $(pgrep -f orderflow) VM.command_line | tr ' ' '\n' | grep -i javaagent

# In-process SDK / bridges on the classpath?
./mvnw dependency:tree | grep -E 'opentelemetry|micrometer-tracing'

# What did the agent decide at startup? (verbose; grep for your own packages)
java -Dotel.javaagent.debug=true -jar target/orderflow.jar 2>&1 | head -100
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| A `-javaagent:` entry **and** OTel SDK artifacts in the tree | Both mechanisms active — expect duplicate spans. Choose one |
| Neither | Nothing is producing spans. Everything below will produce empty results |
| Agent only | Library paths traced; your own code and thread boundaries are not |
| Boot OTel starter only, no agent | In-process instrumentation; check that JDBC/Kafka coverage matches your expectation |

### Proof 2 — the context crosses the process boundary

```bash
# Send a known trace ID. Replace the x's with hex digits of your choosing.
TP='00-11111111111111111111111111111111-2222222222222222-01'

curl -s -H "traceparent: $TP" localhost:8080/api/products/SKU-1001 > /dev/null

# Does the log line carry the SAME trace id?
docker compose logs app --since 30s | grep 11111111111111111111111111111111
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Log lines with `"traceId":"1111...1111"` | Extraction works end to end, and the MDC bridge is wired |
| Log lines with a different trace ID | The header was not extracted — check `OTEL_PROPAGATORS` includes `tracecontext`, and that no proxy stripped it |
| Log lines with no `traceId` field | Tracing may work while the **logging** bridge does not. Fix `logging.pattern.correlation` (Topic 120) |
| A 400 from the server | Malformed `traceparent` — it must be exactly `00-<32 hex>-<16 hex>-<2 hex>` |

### Proof 3 — the span reaches the backend

```bash
# Jaeger's HTTP API: find traces for the service in the last few minutes.
curl -s 'http://localhost:16686/api/traces?service=orderflow&limit=5' \
  | jq '.data[] | {traceID, spans: [.spans[] | {operationName, duration, references}]}'

# Look up the exact trace you created.
curl -s 'http://localhost:16686/api/traces/11111111111111111111111111111111' | jq '.data[0].spans | length'
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Your trace ID with several spans, correctly nested | The full path works: extract → record → export → store |
| Trace present with one span | Downstream spans are being lost or orphaned — go to Proof 4 |
| Trace absent, log line present | The span was created but not exported: check the sampler, the exporter endpoint, and the Collector's logs |
| Trace absent, log line absent | Extraction failed. Go back to Proof 2 |

### Proof 4 — find the orphans, which is the real skill

```bash
# Single-span traces are the signature of a dropped context.
curl -s 'http://localhost:16686/api/traces?service=orderflow&limit=200' \
  | jq '[.data[] | select((.spans | length) == 1) | .spans[0].operationName]
        | group_by(.) | map({op: .[0], count: length}) | sort_by(-.count)'
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Single-span traces named after a *background* operation (`notification.send`, `outbox.publish`, `inventory.apply`) | **The context was dropped at that boundary.** This is the money query |
| Single-span traces named after an HTTP endpoint | Normal — a health check, or a genuinely leaf request |
| No single-span traces at all | Either propagation is complete, or nothing is being sampled. Confirm which |

This query, run weekly, is the cheapest continuous audit of trace completeness you can have.
Consider making it a scheduled check.

### Proof 5 — the thread boundary, isolated

A standalone reproduction with no Spring and no Kafka, so you can see the mechanism naked.

```java
public static void main(String[] args) throws Exception {
    Tracer tracer = GlobalOpenTelemetry.getTracer("proof");
    ExecutorService raw = Executors.newSingleThreadExecutor();
    ExecutorService wrapped = Context.taskWrapping(Executors.newSingleThreadExecutor());

    Span parent = tracer.spanBuilder("parent").startSpan();
    try (Scope scope = parent.makeCurrent()) {
        String onCallingThread = Span.current().getSpanContext().getTraceId();
        String onRawPool  = raw.submit(() -> Span.current().getSpanContext().getTraceId()).get();
        String onWrapped  = wrapped.submit(() -> Span.current().getSpanContext().getTraceId()).get();

        System.out.println("calling thread : " + onCallingThread);
        System.out.println("raw pool       : " + onRawPool);
        System.out.println("wrapped pool   : " + onWrapped);

        // Same again across a virtual thread, to kill the myth that Loom fixes this.
        try (var vt = Executors.newVirtualThreadPerTaskExecutor()) {
            System.out.println("virtual thread : " +
                vt.submit(() -> Span.current().getSpanContext().getTraceId()).get());
        }
    } finally {
        parent.end();
    }
}
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `raw pool` shows all-zeroes (`00000000000000000000000000000000`) | The invalid/root trace ID. **This is the bug, in its purest form** |
| `raw pool` shows a *different* non-zero ID | An agent is attached and wrapped the executor for you — informative, but do not rely on it |
| `wrapped pool` equals `calling thread` | `taskWrapping` works. This is the fix |
| `virtual thread` shows all-zeroes | Confirms virtual threads do **not** inherit context. Note it and move on |
| `virtual thread` equals `calling thread` | Something is decorating it — find out what, because your production path may not have it |

---

## Failure drill

**Assigned drill:** submit work to a plain `ExecutorService` from a traced request; watch the
child span become a NEW TRACE. Fix with context propagation, then repeat across a
virtual-thread boundary.

### The scenario

An `orderflow` order placement fires a notification asynchronously. The executor was created
with `Executors.newFixedThreadPool(8)` inside the service class. Everything works. Traces
exist. The notification's time is nowhere in them.

### Part A — establish a correct baseline first

Before breaking anything, prove that a *synchronous* notification appears in the trace, so
you know your instrumentation is working and the drill is measuring what you think.

```java
// Temporarily synchronous.
public OrderResult place(PlaceOrderCommand command) {
    Order order = persistOrder(command);
    notifier.send(order);                     // same thread
    return OrderResult.accepted(order.id());
}
```

Place one order and record:

**Fill in — baseline (blank template):**

| Reading | Value |
|---|---|
| Trace ID of the placement request | |
| Number of spans in that trace | |
| Is `notification.send` in it? | |
| Total trace duration | |
| `traceId` on the notifier's log lines | |

### Part B — break it

```java
private final ExecutorService notifications = Executors.newFixedThreadPool(8);

public OrderResult place(PlaceOrderCommand command) {
    Order order = persistOrder(command);
    notifications.submit(() -> notifier.send(order));    // the boundary
    return OrderResult.accepted(order.id());
}
```

**If an OTel agent is attached, this may not reproduce** — the agent wraps recognised
executors. Force the uninstrumented shape so you can see the mechanism:

```java
// A hand-rolled Executor the agent has no reason to recognise.
private final BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(1000);
private final Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try { queue.take().run(); } catch (InterruptedException e) { break; }
    }
}, "orderflow-notify-hand-rolled");
// started in @PostConstruct; queue.offer(...) instead of submit(...)
```

Run the Topic 65 load's order-placement scenario for several minutes.

### Part C — capture

| Capture | How |
|---|---|
| The placement trace | Jaeger UI, or Proof 3's API call |
| Span count in that trace | `jq '.data[0].spans \| length'` |
| Single-span traces named `notification.send` | Proof 4's orphan query |
| `traceId` in the request's log lines | `docker compose logs app \| grep <traceId>` |
| `traceId` in the notifier's log lines | `docker compose logs app \| grep notification` — compare the IDs |
| Thread name on the notifier's log lines | Confirms which thread ran it |
| Any error, warning or exception | **There will be none. Note that explicitly** |

**Fill in — with the defect (blank template):**

| Reading | Value |
|---|---|
| Span count in the placement trace | |
| Is `notification.send` present in it? | |
| Trace ID on the request's log lines | |
| Trace ID on the notifier's log lines | |
| Are those two IDs the same? | |
| Count of single-span `notification.send` traces in 5 minutes | |
| Count of orders placed in the same 5 minutes | |
| Any error logged by OTel? | |
| MDC correlation ID on the notifier's log lines (Topic 120) | |

### Part D — how to read it

| What you see | What it means |
|---|---|
| Placement trace is missing the notification span | Confirmed context loss at the executor boundary |
| A separate single-span trace per notification | The task started at `Context.root()`, so the SDK minted a new trace ID |
| Orphan count ≈ order count | Every single one is lost. This is not intermittent |
| Notifier log lines have a different `traceId` | The MDC bridge reads `Context.current()`, so the log and the trace fail together |
| Notifier log lines have **no** correlation ID either | MDC was lost at the same boundary — one bug, two visible symptoms |
| Zero errors anywhere | **The headline.** Silent, well-formed, wrong |
| Placement p99 (Topic 118) unchanged | The application is fine. Only the observability is broken |

### Part E — fix and re-prove

```java
// Option 1 — OTel only, minimal change.
private final ExecutorService notifications =
        Context.taskWrapping(Executors.newFixedThreadPool(8));

// Option 2 — preferred: the injected, centrally decorated Spring executor
// (trace context + MDC + SecurityContext together).
private final ThreadPoolTaskExecutor notifications;   // constructor-injected
```

Re-run the identical load and re-take every Part C reading.

**Fill in — after the fix (blank template):**

| Reading | Broken | Fixed |
|---|---|---|
| Span count in the placement trace | | |
| `notification.send` present? | | |
| Trace IDs match across the boundary? | | |
| Orphan traces in 5 minutes | | |
| MDC correlation ID present on notifier lines | | |
| Placement p99 (Topic 118) | | |

That last row matters: **the fix must not cost measurable latency.** If it does, you have
wrapped something you should not have, or the decorator is doing more than propagation.

### Part F — now the virtual-thread boundary

Repeat the whole drill with the executor replaced:

```java
private final ExecutorService notifications =
        Executors.newVirtualThreadPerTaskExecutor();      // Topic 101
```

**Predict before you run.** Write down your prediction: does the context propagate?

**Fill in — virtual threads (blank template):**

| Reading | Value |
|---|---|
| Your prediction (propagates / does not) | |
| Trace ID observed on the virtual thread | |
| Was your prediction right? | |
| Trace ID after `Context.taskWrapping(...)` around the virtual-thread executor | |
| MDC correlation ID on the virtual thread, before and after | |

**How to read it:** the context does **not** propagate, and the reason is the same as for
platform threads — a non-inheritable `ThreadLocal`. Virtual threads change the *cost* of a
thread, not the *semantics* of thread-local state. If you predicted otherwise, that
misconception was worth finding here rather than in production.

The fix is identical. Note that `Context.taskWrapping` works on a virtual-thread executor
exactly as it does on a platform pool.

### Part G — one more boundary, if you did Phase 10

Repeat across a Reactor `flatMap` (Topic 108's drill, with trace context instead of the
correlation ID). Confirm the loss, then add `Hooks.enableAutomaticContextPropagation()` at
startup and re-confirm. **Note in your write-up that this fix is global and startup-time**,
not per-call-site — a different shape of fix from the executor decorator, and worth being
able to explain.

### Part H — write it up

In Topic 133's form:

- **Contributing factors:** an executor created outside the central factory; no test
  asserting propagation; no orphan-trace check; an agent that masked the problem in some
  paths and not others; MDC and trace context fixed separately so one could regress alone.
- **Detection:** what told you — and note honestly that in production, nothing would have.
  That absence of detection is itself a finding for Topic 124.
- **Prevention that scales:** one executor factory, one decorator covering all three
  contexts, a propagation test in CI, and a scheduled orphan-trace query.

---

## Measurement

### The instrument for each claim

| Claim | Instrument |
|---|---|
| "This request is traced end to end" | Fetch the trace by ID and count spans against the expected boundary list |
| "We do not drop context anywhere" | Proof 4's orphan query, run continuously; plus the CI propagation test |
| "Trace and logs are joined" | `grep` the trace ID in logs; every service in the trace must appear |
| "We have a trace for that incident" | Search by trace ID from the customer's log line; if absent, it is a sampling gap |
| "Tracing costs us X" | Topic 65 load with tracing on vs off; compare p99 and CPU |
| "We are not dropping spans" | The SDK's own exporter metrics (queue size, dropped span count) |
| "Sampling is what we think it is" | spans exported per second ÷ requests per second, over the same window |

### Measure trace completeness as a number

Do not assert completeness; count it.

```bash
# Expected span names for one order placement. Derive this list from YOUR architecture.
EXPECTED='http.server|orderflow.persist-order|orderflow.wallet.debit|orderflow.outbox.insert|orderflow.outbox.publish|orderflow.inventory.apply'

curl -s "http://localhost:16686/api/traces/$TRACE_ID" \
  | jq -r '.data[0].spans[].operationName' \
  | grep -cE "$EXPECTED"
```

**Fill in — trace completeness (blank template):**

| Boundary | Expected span | Present? | If absent, why |
|---|---|---|---|
| HTTP ingress | `http.server` / `GET /api/orders` | | |
| Order persistence | `orderflow.persist-order` | | |
| JDBC | JDBC span(s) | | |
| Wallet debit | `orderflow.wallet.debit` | | |
| Payment gateway call | client span | | |
| Outbox insert | `orderflow.outbox.insert` | | |
| Outbox publish (relay) | `orderflow.outbox.publish` | | |
| Kafka consumer (inventory) | `orderflow.inventory.apply` | | |
| Notification (async executor) | `orderflow.notification.send` | | |
| **Completeness = present ÷ expected** | | | |

That last cell is a percentage you can put in the Topic 124 review and re-measure after each
change. It is the observability equivalent of Topic 50's query counter: **only a count proves
it.**

### Measure the overhead honestly

Run the Topic 65 load three times, changing exactly one thing each time:

1. Tracing disabled entirely (`OTEL_TRACES_SAMPLER=always_off`, or the dependency removed).
2. Tracing on, head-sampled at your production ratio.
3. Tracing on, `always_on` (which is the tail-sampling configuration).

**Fill in — tracing overhead (blank template):**

| Configuration | p50 | p95 | p99 | CPU per pod | Allocation rate | Spans exported/sec |
|---|---|---|---|---|---|---|
| Tracing off | | | | | | |
| Head-sampled at production ratio | | | | | | |
| `always_on` (tail-sampling mode) | | | | | | |

**How to read your own table:** span creation itself is cheap; the costs that show up are
attribute allocation, exporter queue work, and network. If `always_on` costs materially more
than head sampling on the *application* side, look at attribute volume before concluding that
tail sampling is unaffordable — the Collector's cost is a separate budget line and belongs in
Topic 129's cost model, not in your latency budget.

### Watch the exporter, permanently

The SDK exposes its own metrics; find their exact names on your version and graph them.
These are the ones that matter:

| Signal | Why |
|---|---|
| Spans exported per second | Sanity: divide by request rate to get your **effective** sampling ratio |
| Spans **dropped** (queue full) | Silent data loss. Any non-zero value invalidates completeness claims |
| Exporter queue size | Leading indicator of the above |
| Export failures / latency | The Collector being unreachable looks exactly like "no traces" |

```bash
# Find them on your build; the names are version-dependent, so look rather than guess.
curl -s localhost:8080/actuator/prometheus | grep -iE 'otel|span|exporter|processor'
```

**The single most useful derived number:** spans exported per second ÷ requests per second.
Compare it with your configured sampling ratio. If they disagree, either the sampler is not
what you think it is, or you are dropping spans. Both are worth knowing before an incident
rather than during one.

---

## Practice exercises

### 1 — Easy: prove propagation across one boundary

1. Send a request with a known `traceparent` (Proof 2).
2. Confirm the same trace ID appears in the application logs.
3. Confirm the trace exists in the backend with more than one span.
4. Add a manual `@WithSpan` (or hand-built span) on one business method and confirm it
   appears as a child.
5. Then *remove* the try-with-resources around a manual span's `Scope`, run twenty requests,
   and observe request N appearing inside request N−1's trace.

**Deliverable:** the trace ID, the span count, and a one-paragraph description of what the
leaked-scope traces looked like. **Acceptance:** you can state from your own observation what
a leaked `Scope` does to a *subsequent* request, not just to the current one.

### 2 — Medium: the async-boundary audit (combines Topics 56, 90, 101, 108, 113, 115, 120)

**Goal:** a written, verified inventory of every thread boundary in `orderflow` and what
happens to context at each one.

1. Find every boundary:

```bash
grep -rn "new Thread(\|Executors\.\|ThreadPoolExecutor\|@Async\|CompletableFuture\.\|supplyAsync\|runAsync\|subscribeOn\|publishOn\|@KafkaListener\|@Scheduled" src/main/java
```

2. For each hit, fill in the table below.
3. **Verify each row with a test** like Trap 1's, not by reading code.
4. Fix every row that fails, using the central factory.
5. Re-run and re-verify.

**Fill in — async boundary audit (blank template):**

| Location (file:line) | Boundary type | Trace context? | MDC (T120)? | `SecurityContext` (T56)? | Fix applied |
|---|---|---|---|---|---|
| | | | | | |

**Acceptance:**
- [ ] Every boundary in the codebase appears in the table.
- [ ] Every row is verified by an automated test, not by inspection.
- [ ] All three context types propagate everywhere, or the row states why not.
- [ ] No `new Thread(` or bare `Executors.` outside the central factory.
- [ ] Proof 4's orphan query returns nothing from a full Topic 65 run.

### 3 — Hard: production simulation — find the latency with tracing alone

**Goal:** use the trace as the primary diagnostic instrument, the way you would in an
incident.

Have someone inject **one** of the following into `orderflow` under the Topic 65 load,
without telling you which:

- **A.** A 400 ms artificial delay inside the wallet-debit path.
- **B.** The outbox relay's poll interval raised so rows sit for seconds before publishing.
- **C.** A retry loop on the payment gateway with no jitter (Topic 111), so some requests
  take several multiples of the normal time.
- **D.** A context-propagation regression: one executor's decorator removed.

**Rules:** you may use the tracing UI and the trace API. You may **not** read the diff, look
at metrics dashboards, or read application logs *until* you have used the trace to decide
where to look.

**Deliverable:**

| Question | Your answer |
|---|---|
| Which span carried the unexplained time? | |
| Was the time *inside* a span or in a **gap between** spans? | |
| What does a gap mean, mechanically? | |
| How many traces did you need to look at before you were confident? | |
| Was the trace sampled? If not, how did you get one? | |
| Which span attribute would have made this a 10-second diagnosis? | |
| For case D specifically: how did you notice a *missing* span? | |

**Acceptance:** you identify three of the four from traces alone. For any you cannot, the
deliverable is the **span, attribute, or link you would add** — implemented, not described.

Case D is the important one and it is deliberately unfair: a missing span is an absence, and
absences are hard to see. The honest answer is Proof 4's orphan query plus the completeness
table from Measurement. If you reached for those, you passed.

---

## Interview questions

### Q1 — "We added the OpenTelemetry agent, so we have tracing. Agree?"

**MID-LEVEL ANSWER**

"Yes, the agent auto-instruments the common libraries, so we get traces for HTTP calls and
database queries without writing code. We might want to add a few custom spans for business
logic."

**SENIOR ANSWER**

"We have tracing for the paths the agent knows how to instrument, which is genuinely a lot —
servlets, JDBC, Hibernate, HTTP clients, Kafka clients, recognised executors. What we do not
have is anything the agent cannot recognise, and that is exactly the code we wrote
ourselves.

Concretely, four gaps. First, thread boundaries the agent does not cover: a custom
`Executor`, a `new Thread`, a hand-rolled work queue. Trace context is a non-inheritable
`ThreadLocal`, so those start at root and mint a *new trace ID*. It does not throw — you get
two well-formed traces instead of one, which is worse than an error because it looks fine.
Second, our own business operations: no agent can know that 'reserve inventory' is a thing
worth naming. Third, transports we invented — our outbox relay reads rows written by a
request that finished minutes ago, so there is no ambient context at all; the trace context
has to be stored in the outbox row and re-extracted. Fourth, and orthogonal to the agent
entirely: sampling. At 1% head sampling we have no trace for 99% of the incidents anyone
will ever ask about.

So the agent is the floor. What I would actually do is enumerate every async boundary in the
codebase, write down what happens to context at each, and *verify each one with a test* —
because 'it looks instrumented' and 'context propagates' are different claims. Then hunt
single-span traces in the backend as a continuous audit: an orphan trace named after a
background operation is a dropped context, every time."

**WHAT SEPARATES THEM**

The mid-level answer treats custom spans as a nice-to-have. The senior answer names the
mechanism (non-inheritable `ThreadLocal`), enumerates the four distinct gap categories,
knows the failure is silent and produces plausible wrong data, gives a *verification*
strategy rather than a coding strategy, and separates the sampling question out as
independent.

**FOLLOW-UP:** *"How would you find the dropped boundaries in a codebase you've just
inherited?"* — Two passes. Statically: grep for `new Thread`, `Executors.`, `@Async`,
`CompletableFuture.*Async`, `subscribeOn`, `publishOn`, `@KafkaListener`, `@Scheduled`, and
put each hit in a table. Dynamically: query the backend for single-span traces and group by
operation name — anything named after a background operation is a smoking gun. The dynamic
pass finds the boundaries the static pass missed, including ones inside dependencies.

---

### Q2 — "We sample 1% of traces to control cost. Reasonable?"

**MID-LEVEL ANSWER**

"That's pretty standard for high-traffic services. If we need more detail we can temporarily
increase the sampling rate when investigating an issue."

**SENIOR ANSWER**

"It controls cost and it defeats the purpose, and the arithmetic makes that concrete: the
probability that any given customer complaint has a trace is 1%. So in a hundred escalations
we can investigate one. 'Increase the rate when investigating' does not help, because by then
the request we care about has already happened.

The deeper problem is *when* the decision is made. A head sampler decides at the root, before
anything has happened, so it cannot condition on the outcome. It keeps 1% of the boring
traces and 1% of the interesting ones — and since errors and slow requests are rare by
definition, a uniform sample of a rare thing is nearly nothing.

What I want is tail sampling: the application records everything, and the Collector decides
after the trace is complete. Keep 100% of traces with an error, 100% of traces over the SLO
latency boundary, and a few percent of the rest as a baseline for comparison. That is a
strictly better trace set at a similar total cost, because the expensive part is storage and
we are storing the useful ones.

The prerequisite catches people out: the application sampler has to be `always_on`, because
the Collector can only choose among traces it actually receives. Head-sample at 1% *and* tail
sample and you have paid for both mechanisms and got the worse outcome.

If tail sampling genuinely is not available, I would not just accept 1%. I would run the
highest ratio the budget allows and add a force-sample path — a debug header or baggage flag —
so support can say 'reproduce with this header' and get a guaranteed trace. And I would write
the residual risk in the readiness review as a number rather than leaving it implied."

**WHAT SEPARATES THEM**

The senior answer converts the policy into the probability that matters, explains *why* head
sampling is structurally blind rather than just expensive, names the tail-sampling
prerequisite that most people get wrong, and has a concrete fallback plus a written statement
of residual risk.

**FOLLOW-UP:** *"What does tail sampling cost you?"* — The Collector must buffer every span
of a trace until the decision window closes, so it needs memory proportional to trace rate ×
span count × decision wait, and it must be trace-aware in its routing: all spans of one trace
have to reach the *same* Collector instance, which means consistent hashing on trace ID in
front of the Collector fleet. That routing requirement is the part teams discover late. Also,
a trace whose spans arrive after the decision window is judged on what arrived — so a very
long request can be sampled on incomplete information.

---

### Q3 — "Why did our trace break when we moved work onto a thread pool?"

**MID-LEVEL ANSWER**

"The thread pool runs the task on a different thread, so the trace context doesn't carry over.
We need to pass it manually."

**SENIOR ANSWER**

"Because the active span lives in a `ThreadLocal`, and the pool thread's `ThreadLocal` was
never set. `Context.current()` on that thread returns `Context.root()`, so the span you start
there has no parent — and a parentless span gets a brand new trace ID. Nothing throws.

Worth adding: the storage is deliberately a *non-inheritable* `ThreadLocal`. If it were
inheritable, a pool thread created during request one would carry request one's span forever,
and every subsequent request on that thread would be silently attached to it. That is worse
than losing the context, because the trace is then confidently wrong. So the design chose
'absent' over 'wrong', and made propagation an explicit act.

The fix is a wrapper that captures on submit and restores on execute:
`Context.taskWrapping(executor)` for a raw `ExecutorService`, or a
`ContextPropagatingTaskDecorator` on a Spring `ThreadPoolTaskExecutor` — which I prefer,
because it carries trace context, MDC and the `SecurityContext` together. Those three are lost
by the same mechanism, and fixing them one at a time is three chances to forget one.

I would fix it structurally: one factory that builds every executor in the service, decorator
applied there, and `new Thread` or a bare `Executors.` call outside it treated as a review
failure. And a test that asserts the trace ID inside a submitted task equals the submitting
thread's, so the regression cannot come back quietly.

One correction people expect me to get wrong: virtual threads do **not** fix this. A virtual
thread has its own `ThreadLocal`s and does not inherit the creator's context — same
non-inheritable storage, same loss. Loom changes what a thread costs, not what thread-local
state means."

**WHAT SEPARATES THEM**

Naming root context and the new-trace-ID consequence; explaining *why* non-inheritable is the
right design; fixing all three context types at once; making the fix structural and tested;
and pre-empting the virtual-thread misconception.

**FOLLOW-UP:** *"What about Reactor?"* — Different shape, because there is no executor to
wrap; the chain hops threads inside operators. Reactor carries its own `Context` in the
subscription, and `Hooks.enableAutomaticContextPropagation()` bridges it to `ThreadLocal`s at
operator boundaries using the context-propagation library. It is a global, startup-time
switch rather than a per-call-site fix, and that difference is worth knowing — it is Topic
108's lesson with a different payload.

---

### Q4 — "How do you trace across Kafka?"

**MID-LEVEL ANSWER**

"The OpenTelemetry Kafka instrumentation handles it — it puts the trace context in the
message headers automatically, and the consumer picks it up."

**SENIOR ANSWER**

"That is the right mechanism and it is worth saying *why* it has to be that mechanism: the
consumer is a different process. There is no shared memory and no `ThreadLocal` to inherit —
the only thing that crosses is the bytes of the record. So the context has to be serialised
into record headers by the producer and extracted by the consumer. Modern instrumentation
does it for you; I would still verify it rather than assume, because the moment a custom
producer or a hand-rolled relay appears, we own that boundary.

For `orderflow` there is a wrinkle that instrumentation cannot solve. We use a transactional
outbox, so the event is written as a row inside the order transaction and published later by
a relay — possibly minutes later, possibly on a different pod. At publish time there is no
ambient context at all; the originating request is long gone. So the trace context has to be
stored *in the outbox row* at insert time and re-extracted by the relay. That is a deliberate
schema decision, and it is what lets the trace show the outbox delay as a visible gap between
the transaction span and the producer span. I would also put the row's age on the publish span
as an attribute, because that single number tells you whether the relay is keeping up.

One modelling point: parenting the consumer span to the producer span is right for a
one-message flow. For a batch consumer processing 500 messages from 500 different traces in
one poll, parentage is wrong — you would be claiming one trace caused all of it. That is what
span **links** are for: the batch span links to all the originating spans without claiming to
be their child.

And I would validate the whole chain with the boring test: send a request with a known
`traceparent`, then assert that a span with that trace ID appears in the consumer's process."

**WHAT SEPARATES THEM**

Explaining why headers are the *only* possible mechanism; knowing the outbox breaks the
automatic case and how to solve it in the schema; naming the age attribute; knowing span
links and when parentage is wrong; and validating rather than trusting.

**FOLLOW-UP:** *"The consumer retries a message five times. What should the trace look
like?"* — Each attempt is its own span, all linked or parented to the same originating
producer span, with an attribute for the attempt number and the failure reason. What you must
not do is one long span covering all five attempts, because the duration then means nothing
and you cannot tell four fast failures from one slow one. And if the message ends up in a
dead-letter topic, that span should carry an attribute saying so, so a search for DLQ'd
messages is a trace query rather than a log grep.

---

### Q5 — "A customer says an order took eight seconds. Metrics show p99 at target. Go."

**MID-LEVEL ANSWER**

"I'd look at the logs for that customer's order and check whether there was an error or a
slow query around that time."

**SENIOR ANSWER**

"First, the two facts are not in conflict, and saying so out loud avoids an hour of arguing
with the dashboard: p99 means one in a hundred requests is *worse* than that, and this
customer is in that one percent. So I am not looking for the metric to be wrong; I am looking
for the specific request.

The path is: get the trace ID from the customer's request. If we return it in a response
header or an error payload, support already has it. If not, find it from the logs — order ID
to log line to trace ID, which works because Topic 120's structured logging puts the trace ID
on every line. Then fetch that trace.

In the trace I read two things. Where the time is *inside* a span — that names the slow
operation directly. And where the time is in a **gap between** spans, which is more
interesting: a gap means time in something not instrumented. Queueing for a connection
(HikariCP saturation), waiting for a thread from the pool, a GC pause, or a boundary where we
dropped the context and the work is in a different trace entirely.

Then I check the obvious `orderflow`-specific suspects against the shape: a gap before the
JDBC spans is pool acquisition — confirm with `hikaricp_connections_pending` for that window.
A long client span on the payment gateway is Topic 111's territory; check for retries. A long
gap between the transaction span and the outbox publish span is relay lag, and the publish
span's age attribute confirms it directly.

If there is no trace at all for that request, that is itself the finding: our sampling policy
means we cannot investigate individual complaints, and that goes in the readiness review as a
gap with a cost attached rather than being rediscovered on the next escalation."

**WHAT SEPARATES THEM**

Reconciling the metric and the complaint instead of doubting one; knowing the trace ID is the
join key across all three signals; reading gaps as evidence rather than only spans; mapping
gap shapes to specific `orderflow` subsystems and naming the confirming metric; and treating
"no trace exists" as a policy finding rather than a dead end.

**FOLLOW-UP:** *"There's a 3-second gap right at the start of the trace, before your server
span. What is it?"* — Something before the application: TLS handshake, ingress or load
balancer queueing, DNS, or client-side. Or — and this is the one worth checking first in a
JVM — the pod was still warming up: a cold JIT (Topic 74) or a container that passed
readiness before it was genuinely warm. The instrument that discriminates is the request's
timing at the ingress versus at the server span, plus whether the pod's age at that timestamp
was small. That is Topic 122's territory, and it is why readiness gating on warm-up matters.

---

## Mental model checkpoint

1. **`Context.current()` returns the root context inside a pool thread. Why is the storage
   deliberately not an `InheritableThreadLocal`,** and what worse bug would that choice have
   caused?

2. **You call `span.makeCurrent()` and never close the returned `Scope`.** Describe what
   happens to the *next* request that lands on that thread, and why this reproduces under load
   but not in a unit test.

3. **`span.end()` and `scope.close()` do different things.** State what each one does, and
   name the bug produced by doing each without the other.

4. **Head sampling at 1% versus tail sampling.** Where does each decision happen, what
   information does each have available, and what must the application-side sampler be set to
   for tail sampling to work at all?

5. **A trace shows a two-second gap between two spans, with no span covering it.** List four
   distinct things that gap could be, and name the instrument that discriminates between them.

6. **Your Kafka consumer's spans have their own trace IDs.** Give the mechanical reason, and
   say why the outbox relay makes this harder than an ordinary producer would.

7. **You put `sku` on a metric tag and on a span attribute.** One is a serious bug and one is
   correct. Which is which, and what is the structural difference between the two storage
   models that makes it so?

---

## Quick reference card

### The rule

**Trace context is thread state. Every thread hop and every process hop is an explicit
propagation step. Failure is silent.**

### The API, minimal

```java
Context.current()                       // read the ThreadLocal; never null
Context.root()                          // the empty context
context.with(span)                      // returns a NEW context
try (Scope s = context.makeCurrent()) { /* ... */ }   // set + restore
Span.current()                          // the active span, or an invalid one
tracer.spanBuilder(name).setParent(ctx).setSpanKind(kind).startSpan()
span.setAttribute(k, v); span.addEvent(name); span.recordException(e)
span.setStatus(StatusCode.ERROR, msg); span.end()
```

### Fixing a boundary

```java
Context.taskWrapping(executorService)                       // raw executor
Context.current().wrap(runnable)                            // one task
executor.setTaskDecorator(new ContextPropagatingTaskDecorator());  // Spring; all 3 contexts
Hooks.enableAutomaticContextPropagation();                  // Reactor, once at startup
propagator.inject(Context.current(), carrier, setter);      // outbound, custom transport
propagator.extract(Context.root(), carrier, getter);        // inbound, custom transport
```

### Configuration

```properties
OTEL_SERVICE_NAME=orderflow
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_PROPAGATORS=tracecontext,baggage
OTEL_TRACES_SAMPLER=parentbased_always_on     # with Collector tail sampling
# or: parentbased_traceidratio + OTEL_TRACES_SAMPLER_ARG=0.05
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

### Diagnostic commands

```bash
jcmd $(pgrep -f orderflow) VM.command_line | tr ' ' '\n' | grep javaagent
curl -H 'traceparent: 00-<32hex>-<16hex>-01' localhost:8080/api/orders/1
docker compose logs app | grep <traceId>
curl -s 'http://localhost:16686/api/traces?service=orderflow&limit=100' \
  | jq '[.data[] | select((.spans|length)==1) | .spans[0].operationName] | group_by(.) | map({op:.[0],n:length})'
curl -s localhost:8080/actuator/prometheus | grep -iE 'span|exporter'
```

### Reading the evidence

| Symptom | Cause |
|---|---|
| Single-span traces named after a background operation | Context dropped at that boundary |
| Request N's spans inside request N−1's trace | A `Scope` was not closed |
| Consumer spans in their own traces | Context not in the message headers |
| Trace exists, log lines have no trace ID | Logging bridge not wired (Topic 120) |
| No trace at all for a known request | Sampling, or the exporter queue dropped it |
| Gap between spans | Uninstrumented time: pool wait, GC, or a dropped boundary |
| Duplicate SERVER spans per request | Agent **and** in-process starter both active |

### Signal ownership

| Data | Metric tag | Span attribute | Log field |
|---|---|---|---|
| HTTP status, method, route template | yes | yes | yes |
| Order ID, customer ID, SKU | **never** | **yes** | yes |
| Free-text error message | **never** | yes (as an event) | yes |
| PII | never | **never** | never |

### Gotchas checklist

- [ ] `Scope` in try-with-resources; `span.end()` in `finally`.
- [ ] Context storage is **not** inheritable — new threads start at root.
- [ ] Virtual threads do **not** inherit context either.
- [ ] Agent and Boot starter together produce duplicate spans.
- [ ] Kafka needs the context in headers; the outbox needs it in the row.
- [ ] Tail sampling requires `always_on` in the application.
- [ ] Baggage is transmitted on every downstream call — identifiers only, never content.
- [ ] Strip inbound trace headers at an untrusted edge.
- [ ] The exporter queue is bounded and drops silently — graph it.
- [ ] Shutdown must flush the span processor (Topic 123).

---

## When would I use this at work?

**1. Whenever the answer to "which service is slow" is a guess.**
The moment a request crosses two processes, no log file and no metric contains the whole
story. A trace is the only artefact with the causal graph in it. The first hour of any
cross-service latency investigation is either "open the trace" or "argue about dashboards for
a day", and which one you get is decided months earlier by whether context propagates.

**2. Every time you introduce an async boundary — which is a code-review trigger.**
A pull request that adds an executor, a `@Async` method, a `CompletableFuture` chain, or a
new listener is a pull request that can silently break tracing, logging correlation, and
security context in one line. The reviewable question is: "which executor is that, and is it
decorated?" That question takes ten seconds and prevents a class of bug that is invisible in
production.

**3. When you are asked to justify observability cost.**
Someone will propose cutting sampling to save money. The useful contribution is not an
opinion but the arithmetic: at ratio R, the probability a given escalation has a trace is R;
tail sampling gets you 100% of errors and slow requests for a comparable spend because the
storage goes to the traces you actually open. That is a Topic 129-shaped argument, and having
the measured span rate and export volume ready is what makes it land.

---

## Connected topics

**Backwards:**

- **08 — try-with-resources.** `Scope` is an `AutoCloseable` whose `close()` restores thread
  state. The idiom is load-bearing here, not stylistic.
- **56 — Spring Security.** `SecurityContextHolder` is the same `ThreadLocal` problem with a
  different payload; the same decorator fixes both.
- **65 — the load baseline.** The load under which propagation gaps and overhead become
  measurable rather than theoretical.
- **74 — JIT warm-up.** Explains the long spans at the start of a pod's life, and why a cold
  JVM behind a load balancer produces traces that look pathological but are not.
- **79 — `ThreadLocal` leaks.** An unclosed `Scope` is exactly Topic 79's leak, and it corrupts
  traces rather than merely retaining memory.
- **81 — instrumentation agents.** The mechanism the OTel Java agent uses. Knowing Topic 81 is
  why you can reason about what the agent can and cannot see.
- **90 — executors.** Every pool is a boundary. The central factory that fixes context is the
  same factory that bounds queues.
- **91 — `CompletableFuture` and the common pool.** `*Async` without an explicit executor is
  both a pool-choice bug and a context bug.
- **101 — virtual threads.** Does not inherit context. Same fix, and worth proving to yourself
  in the drill.
- **102 — structured concurrency and scoped values.** The JDK's answer to inheritance, and the
  plausible future of context storage — but not something to assume today.
- **108 — reactive context.** Same failure, fixed globally with
  `Hooks.enableAutomaticContextPropagation()` rather than per executor.
- **111 — Resilience4j.** Retries multiply spans; each attempt should be its own span with an
  attempt attribute.
- **113 — Kafka consumer groups.** The consumer is a different process; headers are the only
  channel.
- **115 — the outbox.** Breaks automatic propagation entirely: the context must be persisted
  in the row and re-extracted by the relay.
- **118 — metrics.** The mirror rule. High-cardinality identity is forbidden on a metric tag
  and correct on a span attribute, because one storage model is per-series and the other is
  per-event.
- **120 — MDC and structured logging.** The trace ID on the log line is the join between the
  two signals, and the same boundary loses both.

**Forwards:**

- **122 — startup and containers.** Explains the gap at the front of a cold pod's traces.
- **123 — graceful shutdown.** The span processor must flush on shutdown, or you lose the last
  batch — which is the batch from the moment things went wrong.
- **124 — the production-readiness gate.** Your async-boundary audit table, your trace
  completeness percentage, and your sampling residual risk are direct inputs.
- **129 — capacity and cost.** Span volume, export bandwidth and Collector memory are line
  items; the measured span rate from this topic is the input.
- **130 — SLOs.** Tail sampling's latency threshold should be the SLO boundary, so every
  budget-burning request has a trace by construction.
- **133 — postmortems.** "We had no trace for the failing request" is a contributing factor
  you can prevent in advance, and the sampling section is how.
