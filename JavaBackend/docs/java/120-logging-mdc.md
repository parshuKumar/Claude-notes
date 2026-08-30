# 120 — SLF4J/Logback, Structured Logging, and MDC

## Phase: 11 — Distributed Systems & Production
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: structured JSON logging with correlation IDs across every `orderflow` path — HTTP in, service layer, Hibernate, Kafka producer, Kafka consumer, the outbox relay, and the scheduled jobs. By the end of this topic one `curl` produces a set of log lines you can `grep` by a single ID across all six.

---

## ELI5 anchor

Imagine a hospital.

Every patient who walks in gets a **wristband** with a number on it. From that moment
on, every form, every test result, every note gets that number written at the top. At
the end of the day you can pull every piece of paper for patient 4471 out of a filing
cabinet holding ten thousand sheets.

That wristband is a **correlation ID**. The filing cabinet is your log aggregator.

Now three things can go wrong, and all three happen in real hospitals and real
services:

1. **Nobody writes the number on the form.** The test result exists. You just cannot
   connect it to the patient. This is a log line with no correlation ID.

2. **The form is a paragraph of handwriting instead of labelled boxes.** You can read
   it, one sheet at a time. You cannot ask the cabinet "show me every patient whose
   temperature was over 39". This is unstructured logging — a message string instead of
   named fields.

3. **A nurse forgets to take the wristband off, and the next patient gets it.**
   Now patient B's blood test is filed under patient A. This is worse than having no
   number, because the data is *confidently wrong*.

That third one is the whole reason this topic exists in Java specifically. Java runs
requests on **pooled threads**. The wristband is stored on the thread. If you do not
take it off when the request ends, the next request that lands on that thread inherits
it — and your logs now attribute one customer's failed payment to a different customer.

---

## The bridge from what you know

### SLF4J + Logback ≈ Pino / Winston — HONEST ANALOGUE

Say this once, believe it, and move on. There is nothing clever to learn here.

| Node | Java | Verdict |
|---|---|---|
| `pino` / `winston` — the logger you call | **Logback** — the implementation that writes | **HONEST** |
| the `Logger` interface your code imports | **SLF4J** — `org.slf4j.Logger` | **HONEST** |
| levels: `trace/debug/info/warn/error/fatal` | `TRACE/DEBUG/INFO/WARN/ERROR` (no `FATAL` in SLF4J) | **HONEST** |
| `pino-http` request logger | Spring's `CommonsRequestLoggingFilter` or your own filter | **HONEST** |
| transports / streams | **appenders** | **HONEST** |
| `pino.child({ orderId })` | **MDC** — but see below, this is where it stops being honest | **PARTIAL** |
| JSON output by default (pino) | JSON via an encoder you configure | **HONEST**, one config block |

**SLF4J is a facade.** It is the same idea as coding against an interface so you can
swap the implementation: your code imports `org.slf4j.Logger`, and at runtime exactly
one *binding* jar on the classpath decides who actually writes the bytes. Spring Boot
ships Logback as that binding. You could swap in Log4j2 by changing a dependency and
zero application code.

That is the entire mental model, and it maps one-to-one onto what you already do. If
you can write a Pino logger with a child context and a JSON transport, you can write
Logback. **Do not spend time here.**

### MDC vs `AsyncLocalStorage` — this is the part that is genuinely new

You already have this in Node:

```ts
import { AsyncLocalStorage } from 'node:async_hooks';

const requestContext = new AsyncLocalStorage<{ correlationId: string }>();

app.use((req, res, next) => {
  requestContext.run({ correlationId: req.header('x-correlation-id') ?? randomUUID() },
                     () => next());
});

// ...400 lines away, three awaits deep, inside a setTimeout, inside a Promise.all:
logger.info({ correlationId: requestContext.getStore()?.correlationId }, 'wallet debited');
```

**That correlation ID survives every `await`.** It survives `Promise.all`. It survives a
`setTimeout`. It survives a callback handed to a library you did not write. Node's
async-hooks machinery threads the context through the microtask queue for you. You have
never had to think about it, because it has always just worked.

Java's equivalent is the **MDC** — Mapped Diagnostic Context. It is a `Map<String,String>`
attached to the current thread:

```java
MDC.put("correlationId", correlationId);
log.info("wallet debited");        // the encoder pulls correlationId out of the MDC
```

And here is the delta, stated as plainly as it can be stated:

> **MDC is backed by a `ThreadLocal`. It follows the THREAD. It does not follow the
> work.**
>
> `AsyncLocalStorage` follows the *logical* asynchronous operation. MDC follows the
> *physical* thread. In Node those are effectively the same thing because there is one
> thread. In Java they are different things, and every place they diverge is a bug you
> have to fix by hand.

Two consequences fall straight out of that, and they are opposite failures:

**Consequence 1 — the context VANISHES when work moves to another thread.**

```java
@Async
public CompletableFuture<Void> notifyWarehouse(OrderId id) {
    log.info("notifying warehouse");   // correlationId is GONE. Different thread.
    return CompletableFuture.completedFuture(null);
}
```

No exception. No warning. The log line is written, it just has no correlation ID — or
worse, it has the correlation ID of whatever request last used that pool thread. The
same hole exists for a raw `ExecutorService`, a `CompletableFuture.supplyAsync` without
an executor, a Reactor operator that hops schedulers (Topic 108), a Kafka listener
thread (Topic 113), and the outbox relay's scheduled thread (Topic 115).

**Consequence 2 — the context PERSISTS when it should not.**

A thread in a pool is reused. If request A puts a correlation ID in the MDC and never
removes it, that thread carries A's ID forever. Request B lands on it, logs, and is
labelled with A's ID. If B never sets its own ID — say it takes an early-return path
before your filter runs, or it is a background task — every line B writes is a lie.

This is not merely a logging annoyance. It is **Topic 79's `ThreadLocal` leak shape**
wearing a different hat: a value pinned to a pooled thread that outlives the work that
put it there. On a fixed pool of 200 threads, 200 stale maps live forever. They are
small, so they usually do not OOM you — they just make your incident timeline wrong at
the exact moment you need it to be right.

| | `AsyncLocalStorage` (Node) | MDC (Java) |
|---|---|---|
| What it follows | the async operation | **the thread** |
| Crosses an `await` / thread hop | **yes, automatically** | **no** |
| Cleanup on exit | automatic — the store dies with the scope | **manual — you must `remove` or `clear`** |
| Failure when you forget | none, it is scoped | **the next request inherits your context** |
| Failure when work moves | none | **context silently disappears** |

**Verdict: PARTIAL analogue, and the delta is the entire topic.** Your instincts are
calibrated for a runtime where context propagation is free. In Java it is explicit
work, and forgetting it fails in two directions.

### Parameterised logging — a small thing with no direct Node equivalent

In Node you write:

```ts
logger.debug(`order ${orderId} reserved ${units} units`);
```

The template literal is evaluated *before* `logger.debug` is called. If the level is
`info`, you built that string for nothing. Pino mitigates this by being fast and by
encouraging `logger.debug({ orderId, units }, 'reserved')` — the object form.

Java's SLF4J has a dedicated mechanism:

```java
log.debug("order {} reserved {} units", orderId, units);   // DO THIS
log.debug("order " + orderId + " reserved " + units + " units");  // NOT THIS
```

The `{}` form passes the *arguments*, not a built string. If `DEBUG` is disabled,
Logback checks the level first and never formats anything. The concatenated form builds
the string on every call, at every level, forever — because the arguments to a method
are evaluated before the method runs. Same rule as Node; Java just gives you a first-class
way out of it.

**This matters more than it looks.** A `DEBUG` line inside `orderflow`'s order-line
loop runs 5 times per order at 400 orders/second. That is 2,000 string
concatenations per second producing garbage that is immediately discarded — Topic 68's
allocation rate, for log lines nobody reads. The parameterised form allocates only the
varargs array, and even that is elided for one or two arguments because SLF4J declares
overloads taking one and two `Object` parameters specifically to avoid it.

There is a second, bigger reason, and it is the one to remember: **the parameterised
form keeps the arguments separate from the message.** That is what makes structured
output possible at all. A structured encoder can emit `"orderId": 4829113` as a real
JSON field only if `orderId` arrived as an argument rather than being melted into a
string.

---

## What is this?

Three separate things that people say in one breath. Separate them.

**1. SLF4J — the API you write against.**
`org.slf4j.Logger` and `org.slf4j.LoggerFactory`. Your application code imports only
these. It never imports Logback. This is the same discipline as depending on an
interface rather than a class: it means a library that logs through SLF4J will end up in
*your* log file, in *your* format, without the library knowing anything about your
setup.

**2. Logback — the implementation that does the writing.**
It reads a configuration file, builds a tree of loggers, and routes each event to one or
more **appenders**. An appender knows where bytes go (console, file, socket) and holds an
**encoder** that knows what shape those bytes take (a pattern, or JSON).

**3. MDC — the per-thread key/value map that gets merged into every log event.**
`org.slf4j.MDC`. You put values in at the edge of a request; every log line written on
that thread, anywhere in the call stack, can include them without a single method
signature changing.

**Structured logging** is the practice of emitting each log event as a machine-parseable
object — in practice, one JSON document per line — with named fields, rather than as a
sentence. It is not a library. It is a choice about the encoder, plus a discipline about
what you put in MDC versus what you put in the message.

### Why JSON lines and not a pretty format

Your log aggregator (Loki, Elasticsearch, CloudWatch, Datadog — the mechanism does not
matter) indexes fields. Given:

```
2026-08-29T11:04:22.881Z ERROR [orderflow] o.f.p.PaymentService - payment declined for order 4829113 (customer 90210, gateway adyen, code 51)
```

you can grep for "declined". You cannot ask "count declines by gateway over the last
hour, split by response code" without writing a regular expression that will break the
next time someone rewords the message. Given the same event as fields, that question is
a query. This is exactly the argument you already make for `logger.info({...})` over
`logger.info(\`...\`)` in Pino. Same argument, same conclusion.

---

## Why does it matter?

**1. A distributed system with uncorrelated logs is not debuggable.**

`orderflow` at the Topic 65 baseline serves an order placement that touches: an HTTP
handler, `OrderService`, `InventoryService`, `WalletService`, a Resilience4j-wrapped
payment gateway call (Topic 111), an outbox insert (Topic 115), and — asynchronously,
seconds later, in a different process — a Kafka consumer that decrements a projection.
That is seven log-emitting components across at least two JVMs. Without one shared ID
you are reading seven unrelated streams and guessing at the joins by timestamp. With
400 orders per second, timestamps join nothing; there are hundreds of candidate orders
inside any millisecond.

**2. The failure mode is silent and it corrupts your evidence.**

A missing correlation ID is annoying. A *wrong* correlation ID is dangerous. You will
build an incident timeline out of it, conclude the wrong thing, and ship a fix for a bug
that does not exist. Uncleaned MDC on a pooled thread produces exactly this, and nothing
in the system will tell you it is happening.

**3. It is the substrate under Topics 118 and 119.**

Metrics tell you *that* the p99 moved. Traces tell you *where* the time went. Logs tell
you *why* — they carry the business context (which SKU, which customer, which gateway
response code) that metrics must not carry because of cardinality (Topic 118). Those
three only compose if a log line, a span and a metric exemplar can be joined, and they
are joined by the trace ID sitting in the MDC.

**4. Logging is a production hazard in its own right.**

A synchronous appender writing to a full disk blocks the request thread that called
`log.info`. A `DEBUG` line that serialises a whole request body writes card numbers into
a system that ships logs to a third party. A misconfigured async appender silently
discards the ERROR lines you needed. Logging is I/O on the request path, and it deserves
the same scrutiny as any other I/O on the request path.

---

## Syntax breakdown

### Getting a logger

```java
package com.orderflow.orders;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {

    // static: one per class, not one per instance.
    // final: it never changes.
    // The class literal names the logger, which is how you configure levels per package.
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
}
```

Breaking that line down:

- `LoggerFactory.getLogger(X.class)` returns a logger **named** `com.orderflow.orders.OrderService`.
- Logger names are hierarchical by dot, exactly like package names. Setting
  `com.orderflow` to `DEBUG` sets every logger under it, unless a more specific logger
  overrides.
- `static final` because a logger has no per-instance state and you do not want a field
  reference per object. This is convention everywhere in Java.

With Lombok — which `orderflow` may or may not use, your call — `@Slf4j` on the class
generates that exact field. It is the same thing; know what it generates.

### The five levels and what each one means as a contract

| Level | Contract | In `orderflow` |
|---|---|---|
| `ERROR` | something failed that a human may need to act on | payment gateway returned 500 after all retries; outbox relay could not publish |
| `WARN` | something unexpected that the system handled | circuit breaker opened; optimistic lock retry #2; Kafka rebalance observed |
| `INFO` | a business-significant event, low volume | order placed; order cancelled; service started |
| `DEBUG` | developer detail, off in production by default | which fetch strategy was chosen; cache hit/miss for a SKU |
| `TRACE` | firehose | per-row detail; usually only enabled on one logger for minutes |

The discipline that matters: **`ERROR` should mean "page someone or file a ticket".** If
your ERROR rate is 300/minute in steady state, ERROR has become INFO with a scarier
colour, and the one that mattered is invisible. Downgrade the expected ones to WARN.

### Parameterised logging — every form you need

```java
log.info("order placed");                                     // no arguments
log.info("order {} placed", orderId);                         // one
log.info("order {} placed for customer {}", orderId, custId); // two
log.info("order {} lines {} total {}", orderId, n, totalMinor);  // three+ -> varargs

// Exception ALWAYS goes last and is NOT given a {} placeholder:
log.error("payment authorisation failed for order {}", orderId, ex);

// Escaping a literal brace:
log.info("pattern is \\{} literally", ignored);

// Guard only when building the ARGUMENT is expensive:
if (log.isDebugEnabled()) {
    log.debug("order graph {}", expensiveGraphDump(order));   // dump() would run regardless
}
```

Three rules that cover ninety percent of mistakes:

1. **Never concatenate.** `log.debug("order " + id)` builds the string even when DEBUG is
   off.
2. **The `Throwable` goes last, with no placeholder.** SLF4J detects a trailing
   `Throwable` and prints the stack trace. `log.error("failed {}", ex)` — with a
   placeholder — prints `ex.toString()` and **loses the stack trace**. This is one of the
   most common real defects in Java logging.
3. **Guard with `isDebugEnabled()` only when the arguments themselves are expensive to
   compute.** The formatting is already lazy; the argument evaluation is not.

### MDC — the complete API surface

```java
import org.slf4j.MDC;

MDC.put("correlationId", correlationId);   // set one key on the current thread
MDC.get("correlationId");                  // read it back
MDC.remove("correlationId");               // remove one key
MDC.clear();                               // remove everything on this thread

// Try-with-resources form: removes the key when the block exits, even on exception.
try (MDC.MDCCloseable ignored = MDC.putCloseable("orderId", orderId.toString())) {
    orderService.place(cmd);
}   // "orderId" is removed here, guaranteed

// Copying context to another thread (this is how you fix @Async):
Map<String, String> parentContext = MDC.getCopyOfContextMap();   // may be null!
// ... on the other thread:
if (parentContext != null) {
    MDC.setContextMap(parentContext);
}
```

Two details that bite:

- **`getCopyOfContextMap()` can return `null`**, not an empty map, when nothing has been
  put. Guard it. Passing `null` to `setContextMap` throws.
- **`putCloseable` is the form to prefer** inside a method, because it cannot leak. The
  filter at the request edge still needs an explicit `finally { MDC.clear(); }`, because
  the scope there is the whole request.

### `logback-spring.xml` — the file, annotated

Put it at `src/main/resources/logback-spring.xml`. **Use the `-spring` suffix**, not
plain `logback.xml`: Spring Boot loads the `-spring` variant *after* the environment
exists, which is what enables `<springProfile>` and `<springProperty>`. Plain
`logback.xml` is loaded by Logback itself before Spring starts and cannot see any Spring
property.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="false">

  <!-- Pull a value out of the Spring Environment into a Logback variable. -->
  <springProperty scope="context" name="appName"
                  source="spring.application.name" defaultValue="orderflow"/>

  <!-- Boot ships sensible console defaults; including them keeps colours locally. -->
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <!-- ---------- local development: human-readable ---------- -->
  <springProfile name="local">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
        <pattern>%d{HH:mm:ss.SSS} %-5level [%X{correlationId:-no-cid}] %logger{36} - %msg%n</pattern>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>

  <!-- ---------- docker / load / production: JSON ---------- -->
  <springProfile name="docker,load,prod">
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>correlationId</includeMdcKeyName>
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <includeMdcKeyName>customerId</includeMdcKeyName>
        <customFields>{"service":"${appName}"}</customFields>
        <fieldNames>
          <timestamp>@timestamp</timestamp>
          <message>message</message>
          <levelValue>[ignore]</levelValue>
        </fieldNames>
      </encoder>
    </appender>

    <!-- Bound, non-blocking buffer in FRONT of the encoder. See the traps section. -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
      <appender-ref ref="JSON"/>
      <queueSize>8192</queueSize>
      <discardingThreshold>0</discardingThreshold>   <!-- 0 = never discard WARN/ERROR -->
      <neverBlock>false</neverBlock>                 <!-- block rather than lose events -->
    </appender>

    <root level="INFO">
      <appender-ref ref="ASYNC"/>
    </root>
  </springProfile>

  <!-- Per-package levels. Same in every profile. -->
  <logger name="com.orderflow" level="INFO"/>
  <logger name="org.hibernate.SQL" level="WARN"/>
  <logger name="org.springframework.web" level="INFO"/>

</configuration>
```

Line by line, the parts that are not obvious:

- `%X{correlationId:-no-cid}` — `%X` is the MDC conversion word. `{key:-default}` supplies
  a default when the key is absent. Seeing `no-cid` in your local console is a *feature*:
  it makes a missing correlation ID visible instead of printing an empty bracket.
- `<springProfile name="docker,load,prod">` — comma-separated OR. The `load` profile is
  the one Topic 65 runs under, so your load test exercises the same encoder as production.
  Do not let your load profile use the pretty console encoder; JSON encoding has a
  different cost and you want it in your baseline.
- `<discardingThreshold>0</discardingThreshold>` — the default is **20**, meaning: when
  the queue is 80% full, silently drop TRACE, DEBUG and INFO events. That default is
  reasonable for noise and disastrous if you did not know about it. Setting it to 0
  disables discarding entirely.
- `<neverBlock>false</neverBlock>` — the default. When the queue is full the *application
  thread* blocks until there is room. Setting it `true` drops events instead. This is a
  real trade and you must pick deliberately: block and slow the request, or drop and lose
  evidence.

### Spring Boot's own structured logging (Boot 3.4+)

Boot gained first-class structured logging, so for common formats you may not need the
`logstash-logback-encoder` dependency at all:

```properties
# Emit ECS-format JSON on the console. Other built-in values include logstash and gelf.
logging.structured.format.console=ecs
logging.structured.format.file=ecs
logging.structured.json.add.service.name=orderflow
```

`[BOOT 3.x DELTA]` — this feature arrived in **Boot 3.4**. On Boot 3.0–3.3 and on 2.x
there is no `logging.structured.*`; you add `net.logstash.logback:logstash-logback-encoder`
and configure the encoder in `logback-spring.xml` as shown above. The XML approach works
on **every** line including 4.1, which is why this document shows it as the primary path.

> **Uncertainty, flagged:** I am confident `logging.structured.format.console` exists from
> Boot 3.4 onward and is present in the 4.x line, and that ECS/Logstash/GELF are the
> built-in formats. I am **not** confident about the exact spelling of every
> `logging.structured.json.*` sub-property on 4.1 (the customisation keys for adding,
> renaming and excluding members). Do not take my spelling on faith: run
> `./mvnw dependency:tree` to confirm what encoder you have, then check your Boot version's
> reference documentation appendix for "Common Application Properties" and search for
> `logging.structured`. The XML encoder path in this document has no such uncertainty.

### Correlation fields Boot populates for you

Once Micrometer Tracing is on the classpath (Topic 119), Boot puts `traceId` and `spanId`
into the MDC automatically for instrumented paths, and defines a correlation pattern used
by the default console format. The property to know is:

```properties
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]
management.tracing.baggage.correlation.fields=correlationId,customerId
```

The second line matters: **baggage** is trace context that propagates across services.
Listing a baggage field in `correlation.fields` copies it into the MDC on every hop, so
your own `correlationId` survives the jump from `orderflow` to a downstream service and
appears in *its* logs too. That is the mechanism that makes Topic 119's trace and this
topic's logs join up.

---

## Example 1 — minimal

The smallest thing that demonstrates the whole topic: a correlation ID that survives a
call stack, and then does not survive a thread hop.

```java
package com.orderflow.lab;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

import java.util.UUID;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class MdcMinimal {

    private static final Logger log = LoggerFactory.getLogger(MdcMinimal.class);

    public static void main(String[] args) throws Exception {

        ExecutorService pool = Executors.newFixedThreadPool(1);   // ONE thread, reused

        // ---- "request" A ----
        MDC.put("correlationId", "A-" + UUID.randomUUID());
        log.info("A: on the calling thread");
        deepInTheCallStack();                       // still has it: same thread
        pool.submit(MdcMinimal::onAnotherThread);   // does NOT have it: different thread
        // NOTE: no MDC.clear() here. That omission is the second half of the lesson.

        TimeUnit.MILLISECONDS.sleep(200);

        // ---- "request" B, which forgets to set an ID at all ----
        MDC.clear();                                // B starts clean on the MAIN thread...
        log.info("B: on the calling thread");
        pool.submit(MdcMinimal::onAnotherThread);   // ...but the POOL thread still has A's

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }

    private static void deepInTheCallStack() {
        log.info("nested call, no parameter passed, still labelled");
    }

    private static void onAnotherThread() {
        log.info("submitted task");
    }
}
```

With the console pattern from the Syntax section (`%X{correlationId:-no-cid}`), run it and
read the five lines. **Do not skip to the explanation — predict first, write your
prediction down, then run it.**

What the run demonstrates, in order:

1. `A: on the calling thread` carries A's ID. Expected.
2. `nested call...` carries A's ID **without anyone passing it as a parameter**. That is
   the entire value of MDC: context without plumbing. This is the part that feels like
   `AsyncLocalStorage`.
3. The first `submitted task` line, on the pool thread, has **no** ID (or `no-cid`). The
   context did not follow the work. This is Consequence 1.
4. `B: on the calling thread` has no ID, correctly — main was cleared.
5. The second `submitted task` line is the interesting one. The pool thread was never
   cleaned. Whether it still shows A's ID depends on whether anything wrote to that
   thread's MDC in between, and on your Logback version's inheritance behaviour. **Run it
   and see.** If it shows A's ID while "request" B is executing, you have reproduced
   Consequence 2 in twelve lines.

> **Version uncertainty, flagged honestly:** Logback's MDC adapter has changed its
> thread-inheritance behaviour across major lines. Logback 1.2's `LogbackMDCAdapter` used
> an `InheritableThreadLocal`, so a **newly created** child thread inherited a copy of the
> parent's map; later lines changed the internal representation and the copy-on-inherit
> semantics. **I am not going to assert which behaviour your build has.** It does not
> change the lesson — a *pooled* thread was created long before your request and inherits
> nothing regardless — but it does change what step 3 prints for a freshly created thread.
> The Hands-on section gives you a two-line probe that answers it definitively on your
> machine. Do not reason about this from memory, including mine.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` under the Topic 65 baseline:

- **400 order placements/second** sustained; 100k products, 1M orders, 5M order lines.
- Deployed as **6 pods**, each a JVM on the thread-per-request model. Topic 101 moved the
  servlet path to virtual threads; the `@Async` notification path and the Kafka listener
  containers still use platform-thread pools.
- Order placement calls the payment gateway through a Resilience4j circuit breaker
  (Topic 111), writes an outbox row (Topic 115), and the relay publishes to Kafka on a
  scheduled thread. An inventory-projection consumer in a **separate process** handles
  that event.
- Support asks: *"Customer 90210 says their order failed at 14:03 and their wallet was
  debited anyway. What happened?"*

Answering that question in under five minutes is the requirement. Everything below exists
to serve it.

### Step 1 — establish the ID at the true edge

```java
package com.orderflow.observability;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

/**
 * Establishes the correlation ID for the whole request.
 *
 * HIGHEST_PRECEDENCE so it runs BEFORE the Spring Security filter chain (Topic 56)
 * and before the dispatcher servlet. An authentication failure logged by Security
 * must carry the correlation ID too, or the most interesting failures are the ones
 * you cannot trace.
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    public static final String MDC_KEY = "correlationId";
    private static final String HEADER = "X-Correlation-Id";
    private static final int MAX_LENGTH = 64;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String incoming = request.getHeader(HEADER);
        String correlationId = sanitise(incoming);

        MDC.put(MDC_KEY, correlationId);
        response.setHeader(HEADER, correlationId);   // give it back so the caller can quote it
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();          // <-- THE LINE THIS WHOLE TOPIC IS ABOUT
        }
    }

    /**
     * An inbound header is attacker-controlled. Never put it in the MDC unvalidated:
     * it lands in your log pipeline and can forge log lines or blow up field sizes.
     */
    private String sanitise(String incoming) {
        if (incoming == null || incoming.isBlank() || incoming.length() > MAX_LENGTH) {
            return UUID.randomUUID().toString();
        }
        for (int i = 0; i < incoming.length(); i++) {
            char c = incoming.charAt(i);
            boolean allowed = (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z')
                           || (c >= '0' && c <= '9') || c == '-' || c == '_';
            if (!allowed) {
                return UUID.randomUUID().toString();
            }
        }
        return incoming;
    }
}
```

Four decisions in there worth defending in a review:

- **`finally { MDC.clear(); }`, not `MDC.remove(MDC_KEY)`.** Anything downstream may have
  added keys (`orderId`, `customerId`). `clear()` is the only way to guarantee the thread
  goes back to the pool empty. If some component genuinely needs a key to survive the
  request, that is a design smell, not a reason to weaken this.
- **`OncePerRequestFilter`**, so a `FORWARD` or `ERROR` dispatch does not re-generate the
  ID mid-request.
- **Sanitising the inbound header.** This is a real attack: a header of
  `abc\n{"level":"INFO","message":"admin login ok"}` injects a forged line into a
  line-delimited log pipeline. The allowlist above makes that impossible. It also caps
  length so a 2 MB header cannot be copied onto every log line for the whole request.
- **Echoing the header back.** Support can now ask the customer for the ID from the error
  page, and you go straight to the logs. This one line removes most of the guesswork from
  the support question above.

### Step 2 — add business context where you know it, and remove it when you leave

```java
package com.orderflow.orders;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final InventoryService inventory;
    private final WalletService wallet;
    private final PaymentGateway payments;
    private final OrderRepository orders;
    private final OutboxWriter outbox;

    public OrderService(InventoryService inventory, WalletService wallet,
                        PaymentGateway payments, OrderRepository orders,
                        OutboxWriter outbox) {
        this.inventory = inventory;
        this.wallet = wallet;
        this.payments = payments;
        this.orders = orders;
        this.outbox = outbox;
    }

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {
        try (var c1 = MDC.putCloseable("customerId", String.valueOf(cmd.customerId()));
             var c2 = MDC.putCloseable("sku", cmd.sku())) {

            log.info("placing order units={} totalMinor={}", cmd.units(), cmd.totalMinorUnits());

            inventory.reserve(cmd.sku(), cmd.units());
            wallet.debit(cmd.customerId(), cmd.totalMinorUnits());

            var auth = payments.authorize(cmd);
            if (!auth.approved()) {
                // WARN, not ERROR: a decline is an expected business outcome.
                // gatewayCode is an ARGUMENT, so it becomes a queryable JSON field.
                log.warn("payment declined gateway={} code={}", auth.gateway(), auth.code());
                throw new PaymentDeclinedException(auth.code());
            }

            OrderId id = orders.insert(cmd, auth.paymentId());
            MDC.put("orderId", id.value());          // now that we HAVE one
            outbox.enqueueOrderPlaced(id, cmd);
            log.info("order placed");
            return id;
        } finally {
            MDC.remove("orderId");                   // putCloseable handled the other two
        }
    }
}
```

Note what is **not** in the MDC: the wallet balance, the card token, the full command
object. MDC values are copied onto every subsequent log line on that thread. A large or
sensitive value in MDC is a large or sensitive value repeated hundreds of times.

### Step 3 — the async hop, fixed once and reused everywhere

`orderflow` notifies the warehouse asynchronously. Without help, those lines have no
correlation ID. The fix is a `TaskDecorator`, which Spring applies around every task a
`ThreadPoolTaskExecutor` runs:

```java
package com.orderflow.observability;

import org.slf4j.MDC;
import org.springframework.core.task.TaskDecorator;

import java.util.Map;

/**
 * Captures the SUBMITTING thread's MDC and installs it on the EXECUTING thread.
 *
 * The finally block is not optional: without it the pool thread keeps the submitter's
 * context after the task finishes, and the next task inherits it. That is the same
 * ThreadLocal-on-a-pooled-thread shape as Topic 79.
 */
public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> captured = MDC.getCopyOfContextMap();   // may be null
        return () -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            try {
                if (captured != null) {
                    MDC.setContextMap(captured);
                } else {
                    MDC.clear();
                }
                runnable.run();
            } finally {
                if (previous != null) {
                    MDC.setContextMap(previous);
                } else {
                    MDC.clear();
                }
            }
        };
    }
}
```

```java
package com.orderflow.config;

import com.orderflow.observability.MdcTaskDecorator;
import org.springframework.boot.task.ThreadPoolTaskExecutorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;

@Configuration
public class AsyncConfig {

    @Bean("notificationExecutor")
    public TaskExecutor notificationExecutor(ThreadPoolTaskExecutorBuilder builder) {
        return builder
                .corePoolSize(8)
                .maxPoolSize(8)
                .queueCapacity(500)               // BOUNDED — Topic 90
                .threadNamePrefix("notify-")      // named threads make dumps readable
                .taskDecorator(new MdcTaskDecorator())
                .build();
    }
}
```

Then `@Async("notificationExecutor")`, never bare `@Async`. Bare `@Async` uses whatever
executor Boot auto-configures, which is not the one you decorated.

> **Restore, do not clear.** Notice the decorator saves `previous` and restores it rather
> than clearing. On a virtual-thread executor the "pool thread" is per-task and clearing
> would be fine; on a platform pool, restoring is strictly safer and costs nothing.

**If `io.micrometer:context-propagation` is on your classpath** (it is, if you did Topic
119), there is a ready-made `ContextPropagatingTaskDecorator` that propagates trace
context *and* anything registered with the `ContextRegistry`, MDC included. Prefer it
over hand-rolling if you have it — one mechanism for tracing and logging is better than
two. **I am not certain whether Boot 4.1 auto-applies it to auto-configured executors or
whether you must set it explicitly**; check by running the Hands-on Proof 3 probe with
*no* decorator configured. If the correlation ID survives, it is automatic. If it does
not, wire it explicitly. Two minutes, definitive answer, no guessing.

### Step 4 — the Kafka consumer, in a different process

The consumer is a separate JVM. The MDC cannot travel through memory; it must travel in
the message. The outbox row already carries the correlation ID, so the relay puts it in a
Kafka header, and the consumer reads it back into its own MDC:

```java
package com.orderflow.inventory;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.nio.charset.StandardCharsets;

@Component
public class OrderPlacedListener {

    private static final Logger log = LoggerFactory.getLogger(OrderPlacedListener.class);

    @KafkaListener(topics = "orderflow.order-placed", groupId = "inventory-projection")
    public void onOrderPlaced(ConsumerRecord<String, OrderPlacedEvent> record) {
        var header = record.headers().lastHeader("X-Correlation-Id");
        String correlationId = header == null
                ? "kafka-orphan"
                : new String(header.value(), StandardCharsets.UTF_8);

        MDC.put("correlationId", correlationId);
        MDC.put("topic", record.topic());
        MDC.put("partition", String.valueOf(record.partition()));
        try {
            log.info("projecting order-placed offset={}", record.offset());
            // ... apply the projection ...
        } finally {
            MDC.clear();    // container threads are LONG-LIVED and REUSED across messages
        }
    }
}
```

`kafka-orphan` as the fallback is deliberate. It is greppable. A spike in `kafka-orphan`
tells you a producer stopped setting the header — a real regression you would otherwise
never notice.

### What the log line looks like

*Illustration of the format, not captured output. `<n>` and `xxx` are placeholders.*

```json
{"@timestamp":"<iso8601>","level":"WARN","logger":"com.orderflow.orders.OrderService","thread":"http-nio-8080-exec-<n>","service":"orderflow","correlationId":"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx","traceId":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx","spanId":"xxxxxxxxxxxxxxxx","customerId":"<n>","sku":"SKU-<n>","message":"payment declined gateway=adyen code=<n>"}
```

And now the support question is a query, not an investigation:

1. Ask the customer for the ID from the error page (you echoed it in the response header).
2. `correlationId:"<that id>"` sorted by timestamp returns every line from all six
   components, across both processes, in order.
3. The `wallet debited` line and the `payment declined` line are both there, adjacent, and
   you can see whether the compensating credit (Topic 117's saga) ran.

**That is the deliverable of this topic.** Not "we have logs" — *one identifier answers
the question in three steps*.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — MDC never cleared on a pooled thread: request B logs request A's ID

**Wrong:**

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        MDC.put("correlationId", UUID.randomUUID().toString());
        chain.doFilter(req, res);
        MDC.clear();        // <-- NOT in a finally block
    }
}
```

**Exact symptom, and it is specific enough to recognise:** in your aggregator, a single
`correlationId` value appears on log lines spanning **minutes**, from many different
`customerId` values, on the same `thread` name. Meanwhile the request that *should* have
produced those lines has no lines at all. A concrete detection query: group by
`correlationId`, and look for any ID whose set of distinct `customerId` values has size
greater than one. In a healthy system that count is always exactly one.

The trigger is an exception. `chain.doFilter` throws (a validation failure, a
`PaymentDeclinedException`, anything), `MDC.clear()` is skipped, and that Tomcat worker
thread carries the ID until it happens to be overwritten. On a 200-thread pool at 400 rps,
a 1% error rate poisons a thread every quarter second.

**Root cause:** MDC is a `ThreadLocal` on a thread that outlives the request. Cleanup that
is not in a `finally` is cleanup that does not happen on the error path — and the error
path is exactly the path you will be reading logs for.

**Fix:** `try { chain.doFilter(...); } finally { MDC.clear(); }`. Inside a method body,
prefer `MDC.putCloseable`, which cannot be got wrong. Then add the detection query above
as a saved search so a future regression is found by the system rather than by an
incident.

**Related, same root cause:** this is Topic 79's `ThreadLocal`-on-a-fixed-pool leak. There
the damage was retained memory; here it is retained *wrong information*. The second is
worse, because memory you can see in a heap dump.

---

### Trap 2 — the correlation ID vanishes across `@Async` and every other thread hop

**Wrong:**

```java
@Service
public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

    @Async
    public void notifyWarehouse(OrderId id) {
        log.info("notifying warehouse for order {}", id);
    }
}
```

**Exact symptom:** searching by correlation ID returns the request lines and stops at the
async boundary. The warehouse-notification lines exist — you can find them by searching
the message text — but they carry no `correlationId` field, or carry the `no-cid` default,
or (worse, and intermittently) carry a *different* correlation ID belonging to whichever
request last used that pool thread. The line you needed is present and unfindable.

Under load you get a second, more confusing symptom: the correlation ID is present
*sometimes*. Whichever pool thread happens to have been dirtied with a matching ID looks
correct. This intermittency is why people conclude "the logging is flaky" and stop
investigating.

**Root cause:** `@Async` runs your method on a different thread. MDC lives on the calling
thread's `ThreadLocal`. Nothing copies it. There is no error because there is nothing to
error about — an empty map is a perfectly valid map.

**The same root cause, in five more places you have already built:**

| Boundary | Where it appears in `orderflow` | Topic |
|---|---|---|
| `@Async` | warehouse notification | this doc |
| raw `ExecutorService.submit` | any hand-rolled pool | 90 |
| `CompletableFuture.supplyAsync` with no executor | uses the common ForkJoinPool | 25, 91 |
| Reactor `publishOn` / `flatMap` | the payment-callback fan-in | 105, 108 |
| Kafka listener container thread | the inventory projection | 113 |
| `@Scheduled` | the outbox relay | 115 |

**Fix:** a `TaskDecorator` on every executor you own (Example 2, Step 3), Kafka headers
across processes (Step 4), and for Reactor, the Context — not MDC — as the carrier
(Topic 108). If `context-propagation` is available, use
`ContextPropagatingTaskDecorator` so one mechanism serves both tracing and logging.

**The discipline that prevents recurrence:** never create an executor with a bare
`Executors.newFixedThreadPool` in application code. Create them through one factory
method that applies the decorator, the bound queue (Topic 90), the thread-name prefix and
the metrics binder (Topic 118). Make the paved road the easy road.

---

### Trap 3 — string concatenation instead of parameters, and the `Throwable` placeholder

**Wrong:**

```java
for (OrderLine line : order.lines()) {
    log.debug("order " + order.id() + " line " + line.sku() + " qty " + line.units());
}
// ...
catch (PaymentException e) {
    log.error("payment failed for order {}: {}", order.id(), e);   // e HAS a placeholder
}
```

**Exact symptom, part one (the concatenation):** nothing visible in correctness. What you
see is in the *profile*. Take an allocation profile at the Topic 65 baseline (Topic 78)
and `StringBuilder`/`String` allocations attributed to `OrderService` frames appear in the
top allocation sites — **while `DEBUG` is switched off**. That is the tell: you are paying
for logging you are not doing. In an aggregated flame graph the frames sit under
`java.lang.StringConcatFactory` call sites inside your loop.

**Exact symptom, part two (the `Throwable` placeholder), and this one is worse:** the log
line reads `payment failed for order 4829113: com.orderflow.payments.PaymentException: declined`
and **there is no stack trace**. You know it failed. You do not know where. In JSON output
the `stack_trace` field is absent entirely. During an incident this converts a two-minute
diagnosis into an hour of code reading.

**Root cause, part one:** arguments are evaluated before the method is called. `log.debug`
cannot suppress work that already happened. Part two: SLF4J prints a stack trace only for
a trailing `Throwable` that has **no** corresponding placeholder. Giving it a `{}` makes it
an ordinary formatting argument, rendered with `toString()`.

**Fix:**

```java
if (log.isDebugEnabled()) {                     // guard only if arguments are expensive
    for (OrderLine line : order.lines()) {
        log.debug("line sku={} qty={}", line.sku(), line.units());
    }
}
catch (PaymentException e) {
    log.error("payment failed for order {}", order.id(), e);   // NO placeholder for e
}
```

**Enforce it mechanically.** This is a rule a human will break under deadline pressure.
Add SpotBugs with the `findsecbugs`/SLF4J plugin, or ErrorProne, to CI (Topic 132's
argument for gating new code only). A static check catches both halves of this trap
forever; a code-review guideline catches it until the reviewer is busy.

---

### Trap 4 — secrets and PII in logs, usually via a well-meaning DEBUG line

**Wrong:**

```java
log.debug("inbound payment request: {}", objectMapper.writeValueAsString(request));
log.info("authenticating with gateway key {}", gatewayApiKey);
log.debug("customer {} card {}", customer.email(), card.number());
```

**Exact symptom:** none, until it is a very bad symptom. The realistic discovery paths, in
order of how they actually happen: (a) a compliance scan of the log index flags a
credit-card-shaped pattern; (b) an engineer turns `com.orderflow.payments` to DEBUG for
twenty minutes to investigate an incident and ships several thousand card numbers into a
third-party SaaS log platform with 30-day retention; (c) an auditor asks who has access to
the log index and the answer is "the whole engineering org".

The reason this is a *trap* and not just a mistake: the DEBUG line is harmless in
development and stays harmless right up until someone raises the level in production
during an incident — which is precisely when nobody is thinking about data classification.

**Root cause:** log statements are not covered by whatever mechanism protects the field
elsewhere. Serialising a whole object into a log message bypasses every `@JsonIgnore`,
every DTO boundary, and every masking rule you built for the API layer (Topic 46).

**Fix, in layers, because one layer will fail:**

1. **Never log a whole request/response object.** Log named fields you chose. This is a
   flat rule with no exceptions.
2. **Make the type refuse to be printed.** A value type is the strongest control available
   because it survives careless callers:

```java
public record CardNumber(String value) {
    @Override
    public String toString() {
        return "CardNumber[****" + value.substring(value.length() - 4) + "]";
    }
}
```

   Now `log.debug("card {}", card)` is safe even when written by someone who has not read
   this document. Records generate a `toString` from components; overriding it is the
   whole defence.
3. **Never log a secret at any level.** Credentials come from the environment and go into
   a client. They are never an argument to a logger. Topic 123 covers the sources.
4. **A masking filter at the appender is a backstop, not a control.** A regular expression
   over the message can catch card-shaped strings, and it will miss the ones split across
   fields or base64-encoded. Use it as the last line, never as the plan.
5. **Log-level changes in production are a change.** If `management.endpoint.loggers` is
   exposed (Topic 121) anyone can raise a level over HTTP. Gate it behind authentication
   and make the change auditable.

---

### Trap 5 — the appender is on the request path, and the async wrapper silently drops events

**Wrong:**

```xml
<appender name="FILE" class="ch.qos.logback.core.FileAppender">
  <file>/var/log/orderflow/app.log</file>
  <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
<root level="INFO">
  <appender-ref ref="FILE"/>
</root>
```

...or the "fix" that looks safe:

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
  <appender-ref ref="FILE"/>
</appender>
```

**Exact symptom of the first version:** p99 latency on every `orderflow` endpoint degrades
together — including endpoints that touch no database — whenever the log volume rises or
the log destination slows. It correlates with error rate, because errors produce more log
lines, which makes the incident worse than the incident. A thread dump (`jcmd <pid>
Thread.print`) taken during the spike shows request threads in
`ch.qos.logback.core.OutputStreamAppender.writeBytes` or blocked on the appender's lock.
That stack is the proof. Nothing else in the system points at logging.

**Exact symptom of the second version (the async wrapper with defaults), and it is a nasty
one:** during your worst incident — the one where you most need the logs — INFO and DEBUG
lines stop appearing. Not all of them: the ones that fell during the burst. There is no
error about this. `AsyncAppender` defaults `discardingThreshold` to 20, meaning it drops
TRACE/DEBUG/INFO events once the queue is 80% full. You reconstruct the incident from a
log that has holes exactly where the load was highest, and you draw the wrong conclusion
about when things started.

A second symptom of the same configuration: **the stack traces are there but the
surrounding context is not**, because ERROR survived the discard and INFO did not.

**Root cause:** `log.info` is a synchronous, blocking method call by default. The
appender's write happens on your request thread while it holds an appender lock. Wrapping
it in `AsyncAppender` moves the write to a background thread, which is correct — but the
queue between them is bounded, and the default policy for a full queue is to silently drop
the low-severity events.

**Fix, stated as the trade rather than as a single answer:**

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
  <appender-ref ref="JSON"/>
  <queueSize>8192</queueSize>
  <discardingThreshold>0</discardingThreshold>   <!-- never discard -->
  <neverBlock>false</neverBlock>                 <!-- block the app rather than lose data -->
  <includeCallerData>false</includeCallerData>   <!-- see below -->
</appender>
```

- `discardingThreshold=0` plus `neverBlock=false` means: **never lose a line; slow the
  application if the sink cannot keep up.** That is the right default for a payment
  service, where the log is evidence.
- `neverBlock=true` means: **never slow the application; lose lines under pressure.** That
  is right for a high-volume, low-value stream. Choose it deliberately and write down the
  choice.
- `includeCallerData=false` matters more than it looks. Capturing the caller's file and
  line number requires walking the stack **per event**, and on an async appender it must
  happen on the *producing* thread. Turning it on for a high-volume logger is a
  measurable, self-inflicted cost. Leave it off; the logger name already tells you the
  class.
- **In a container, log to stdout, not to a file.** Kubernetes collects stdout. A file
  inside the container needs a sidecar, a volume, and rotation you now own. This is the
  one Kubernetes-adjacent point in this document and you already know it — the Java-specific
  part is only that `ConsoleAppender` is the appender that does it.

**How to confirm the fix rather than believe it:** re-run the Topic 65 baseline with the
log level at INFO and again at DEBUG for `com.orderflow`. If p99 moves materially between
those two runs, logging is on your critical path and the async configuration is not doing
its job. That comparison is a five-minute experiment and it settles the argument.

---

## Hands-on proof

Every command here is one **you** run. No output is reproduced in this document; what
follows is the exact source, the exact command, what to look for, and how to read every
result you might get.

### Setup

```bash
mkdir -p ~/java-lab/120 && cd ~/java-lab/120
java --version                        # expect 21 or 25

curl https://start.spring.io/starter.zip \
  -d dependencies=web,actuator \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=logging-lab \
  -d type=maven-project -o logging-lab.zip && unzip logging-lab.zip -d logging-lab
cd logging-lab
```

Add the encoder dependency to `pom.xml` (needed unless you use Boot's built-in
`logging.structured.*`):

```xml
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version><!-- check the latest on Maven Central; do not copy a version from a blog --></version>
</dependency>
```

> I am deliberately not writing a version number here. Version numbers in documents rot,
> and an invented one is worse than a blank. Run
> `./mvnw versions:display-dependency-updates` or look it up, once.

### Proof 1 — which SLF4J binding is actually active

Before anything else, confirm there is exactly one.

```bash
./mvnw dependency:tree | grep -iE "slf4j|logback|log4j|jul|jcl"
```

| What you see | What it means |
|---|---|
| `slf4j-api`, `logback-classic`, `logback-core`, plus `jul-to-slf4j` and `log4j-to-slf4j` | The healthy Boot default. The two `-to-slf4j` bridges redirect libraries using other APIs into Logback. |
| `logback-classic` **and** `slf4j-log4j12` or `slf4j-simple` | **Two bindings.** SLF4J picks one arbitrarily and warns at startup. Your configuration is being applied to a logger nobody is using. Exclude the extra one. |
| `log4j-slf4j-impl` **and** `log4j-to-slf4j` together | A routing loop. Exclude one; usually you keep `log4j-to-slf4j`. |
| No `logback-classic` at all | Something excluded `spring-boot-starter-logging`. Your `logback-spring.xml` is being ignored entirely — which is a common reason "my config does nothing". |

Also run the app once and read the first lines of startup output: SLF4J prints a
multi-binding warning naming each binding it found. **Absence of that warning is a
positive signal.**

### Proof 2 — the MDC is on the thread, and you can see it

`src/main/java/com/orderflow/lab/MdcProbe.java`:

```java
package com.orderflow.lab;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

@Component
public class MdcProbe implements CommandLineRunner {

    private static final Logger log = LoggerFactory.getLogger(MdcProbe.class);

    @Override
    public void run(String... args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(1);

        MDC.put("correlationId", "REQ-A");
        log.info("caller thread");

        // (a) a freshly CREATED thread -- tests inheritance
        Thread fresh = new Thread(() -> log.info("fresh thread"));
        fresh.start();
        fresh.join();

        // (b) a POOLED thread -- created before the MDC was set
        pool.submit(() -> log.info("pooled thread, first task")).get();

        MDC.clear();
        log.info("caller thread after clear");

        // (c) the same pooled thread again, with the caller now clean
        pool.submit(() -> log.info("pooled thread, second task")).get();

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

with `src/main/resources/logback-spring.xml` console pattern including
`[%thread] [%X{correlationId:-NONE}]`.

```bash
./mvnw spring-boot:run
```

**What to look for:** the value in the second bracket on each of the five lines.

| What you see | What it means |
|---|---|
| `caller thread` shows `REQ-A` | Baseline. MDC works on the setting thread. |
| `fresh thread` shows `REQ-A` | Your Logback version's MDC adapter copies the map to newly created child threads. Useful to know; **irrelevant to pools**, which is the case that matters. |
| `fresh thread` shows `NONE` | Your version does not inherit. Also fine. Either way you must propagate explicitly for pools. |
| `pooled thread, first task` shows `NONE` | **The core demonstration.** The pool thread existed before you set the MDC, so it inherited nothing. Every `@Async` in your codebase behaves like this. |
| `pooled thread, first task` shows `REQ-A` | You are running a decorator or a context-propagation library already. Confirm with `./mvnw dependency:tree | grep context-propagation`, and if so you have your answer to the open question in Example 2, Step 3. |
| `pooled thread, second task` shows `NONE` after a clean first task | Healthy — nothing dirtied the thread. |
| `pooled thread, second task` shows `REQ-A` while the caller is clear | **Trap 1 reproduced.** A previous task left the value behind. Now go add the `finally`. |

This probe answers the version question I flagged in Example 1 definitively, on your
machine, in under a minute. Prefer it to any statement in this document.

### Proof 3 — the correlation ID survives a real HTTP request end to end

Add the `CorrelationIdFilter` from Example 2 and a controller that logs, calls an
`@Async` method that logs, and returns.

```bash
./mvnw spring-boot:run &

# Send our own ID so it is easy to grep:
curl -i -H 'X-Correlation-Id: probe-0001' http://localhost:8080/orders/probe
```

**What to look for:**

| What you see | What it means |
|---|---|
| The response carries `X-Correlation-Id: probe-0001` | The filter ran and echoed it. Support can now quote it. |
| Response header shows a UUID instead of `probe-0001` | Your sanitiser rejected the input, or the filter did not run. Check the allowlist and the `@Order`. |
| Log lines from the controller and the service both show `probe-0001` | MDC is doing its job across the call stack. |
| The `@Async` line shows `probe-0001` | Your `TaskDecorator` is wired to the executor that `@Async` actually used. |
| The `@Async` line shows nothing or a different ID | The decorator is not applied to that executor. Almost always: `@Async` with no qualifier, using a different executor than the one you configured. Name the executor explicitly. |
| A Spring Security line (try a 401) shows no ID | Your filter is ordered after the security chain. Move it to `HIGHEST_PRECEDENCE`. |

Then send two requests concurrently with different IDs and confirm no line carries the
other's:

```bash
curl -H 'X-Correlation-Id: probe-AAAA' http://localhost:8080/orders/probe &
curl -H 'X-Correlation-Id: probe-BBBB' http://localhost:8080/orders/probe &
wait
```

### Proof 4 — the JSON is actually valid JSON, one object per line

A structured log that your aggregator cannot parse is worse than a pretty one, because you
believe you have structure.

```bash
SPRING_PROFILES_ACTIVE=docker ./mvnw spring-boot:run 2>/dev/null | while read -r line; do
  echo "$line" | jq -e . >/dev/null 2>&1 || echo "NOT JSON: $line"
done
```

**What to look for:**

| What you see | What it means |
|---|---|
| No `NOT JSON` lines | Every event is a parseable object. This is what you want. |
| `NOT JSON` on the Spring banner and a few startup lines | Expected before Logback is configured, and the banner is not JSON by design. Set `spring.main.banner-mode=off` for containers. |
| `NOT JSON` on multi-line stack traces | Your encoder is not embedding the stack trace as a field. A stack trace split across lines makes every frame a separate unparseable record. Fix the encoder configuration, not the aggregator. |
| `NOT JSON` on lines from a third-party library | That library is writing to `System.out` directly instead of through SLF4J. Find it and route it, or accept it and tell your aggregator. |

Then confirm the fields you care about are present:

```bash
SPRING_PROFILES_ACTIVE=docker ./mvnw spring-boot:run 2>/dev/null \
  | jq -r 'select(.correlationId != null) | [.level, .correlationId, .logger] | @tsv'
```

If that prints nothing while the app is serving traffic, your `includeMdcKeyName` list
does not include `correlationId`, or the filter is not populating it.

### Proof 5 — prove logging is (or is not) on your critical path

```bash
# Run the Topic 65 k6 script twice against the same build.
# Run 1: production level.
SPRING_PROFILES_ACTIVE=load LOGGING_LEVEL_COM_ORDERFLOW=INFO  <your run command>
# Run 2: everything on.
SPRING_PROFILES_ACTIVE=load LOGGING_LEVEL_COM_ORDERFLOW=DEBUG <your run command>
```

Record p50/p95/p99 and throughput for both, in your Topic 65 results table.

| What you see | What it means |
|---|---|
| p99 essentially unchanged between INFO and DEBUG | Logging is not on your critical path at this volume. Good. Note the numbers and move on. |
| p99 materially worse at DEBUG, throughput down | Expected to some degree — you are doing more work. The question is *how much*: if DEBUG costs more than a few percent of throughput, your appender is synchronous or your DEBUG lines are concatenating. |
| p99 worse at DEBUG **and** a thread dump shows request threads inside Logback frames | Trap 5. The appender is blocking request threads. |
| Error rate rises at DEBUG | You have crossed a saturation point. Whatever is being saturated (disk, the log sidecar, the socket appender) is now a dependency of your request path. |

You can also flip a level at runtime without a restart, which is how you would do this in
production:

```bash
curl -X POST http://localhost:8080/actuator/loggers/com.orderflow \
  -H 'Content-Type: application/json' \
  -d '{"configuredLevel":"DEBUG"}'

curl http://localhost:8080/actuator/loggers/com.orderflow    # read it back
```

(That endpoint is Topic 121's territory. Note now that it is a **write** endpoint and must
not be publicly exposed.)

---

## Practice exercises

### 1 — Easy: make the wristband real

Build a Boot app with one endpoint, `GET /orders/{id}`, that logs at INFO in the
controller and at DEBUG in a service method it calls.

1. Add a `CorrelationIdFilter` that accepts an inbound `X-Correlation-Id`, generates one
   when absent, echoes it in the response, and clears the MDC in a `finally`.
2. Configure two profiles: `local` with a human pattern that shows the ID, `docker` with a
   JSON encoder that emits it as a field.
3. Prove with `curl` that (a) a supplied ID is used, (b) an absent one is generated, and
   (c) the response header matches the log lines.
4. Now **delete the `finally`** and replace it with a plain `MDC.clear()` after
   `chain.doFilter`. Add an endpoint that throws. Hit the throwing endpoint once, then hit
   the normal endpoint with **no** correlation ID header several times, and find the log
   line that carries the thrown request's ID.

Write down the exact log line from step 4. That line is the reason for the `finally`.

### 2 — Medium: the audit (combines Topics 01–119)

The class below contains **seven** defects. Four are from this topic; three are from
earlier topics. For each: name the topic, state the **observable** symptom in production
(not "it is bad practice"), and write the fix.

```java
package com.orderflow.payments;

@Service
public class PaymentReconciliationService {

    private final Logger log = LoggerFactory.getLogger(PaymentReconciliationService.class);

    private static final Map<String, Payment> SEEN = new HashMap<>();

    private final ExecutorService pool = Executors.newFixedThreadPool(4);

    @Value("${orderflow.gateway.api-key}")
    private String apiKey;

    public void reconcile(List<Payment> payments) {
        log.info("reconciling with key " + apiKey + " for " + payments.size() + " payments");

        for (Payment p : payments) {
            SEEN.put(p.reference(), p);
            pool.submit(() -> {
                MDC.put("paymentRef", p.reference());
                try {
                    Integer amount = p.amountMinor();
                    long cents = amount;
                    gateway.settle(p.reference(), cents);
                    log.debug("settled " + p.reference() + " amount " + cents);
                } catch (SettlementException e) {
                    log.error("settlement failed for {}: {}", p.reference(), e);
                }
            });
        }
    }
}
```

Hints, in the order to think about them: one defect makes a *field* injection into a class
that also holds a pool — ask what happens on shutdown (Topic 90). One is a Topic 01
unboxing NPE with a specific trigger. One is a Topic 79 leak that also breaks Topic 118's
memory profile. Four are from this document, and two of those four are in the same two
lines.

### 3 — Hard: production simulation — correlation across every `orderflow` boundary

**Part A — instrument.** Take `orderflow` as it stands after Topic 119 and make one
`curl` to `POST /orders` produce correlated log lines from all of: the HTTP filter,
`OrderService`, `InventoryService`, `WalletService`, the Resilience4j-wrapped gateway call,
the outbox writer, the outbox relay (a `@Scheduled` thread), and the inventory-projection
Kafka consumer **in a separate process**. Use the correlation ID as the only search key.

Record how many of the eight components you got on the first attempt without changes. That
number is your honest starting position.

**Part B — break it deliberately, three ways.** For each, predict the symptom in writing
*before* running it, then run it and record what you actually saw:

1. Remove the `TaskDecorator` from the notification executor.
2. Remove the `finally { MDC.clear(); }` from the filter, and drive a 1% error rate under
   the Topic 65 load for two minutes.
3. Remove the Kafka header propagation from the outbox relay.

For (2), write the aggregator query that **detects** the problem automatically — the one
that finds a correlation ID associated with more than one `customerId`. That query is a
deliverable; it goes in your monitoring, and it goes in the Topic 124 review as an
observability control.

**Part C — measure the cost.** Run the Topic 65 baseline three times: JSON encoder at
INFO, JSON encoder at DEBUG for `com.orderflow`, and human-readable pattern at INFO.
Record p50/p95/p99 and throughput for each. Then answer: what does structured JSON encoding
cost you, and is that cost on the request thread or on the appender thread? Prove your
answer with a thread dump taken during the run, not by reasoning.

**Part D — the virtual-thread question.** Topic 101 moved the servlet path to virtual
threads. Virtual threads are not pooled — a new one is created per request and discarded.
Answer with evidence, not reasoning: does Trap 1 still exist on a virtual-thread executor?
Does Trap 2? Design a probe that answers each, run it, and write down what changed and what
did not. Then state what this implies about which of your `finally` blocks are still load-
bearing after the migration.

**Part E — argue against yourself.** You have put `customerId` in the MDC. Make the
strongest possible case that this is wrong (consider: PII in a log index, retention policy,
field cardinality in the aggregator, cost per indexed field). Then state what would have to
be true for your original choice to be correct, and decide.

---

## Interview questions

### Q1 — "How do you get a request ID onto every log line in a Java service?"

**Mid-level answer:** "Use MDC. Put the ID in a servlet filter at the start of the request
and the logging pattern picks it up with `%X`."

**Senior answer:** "MDC in a filter is the mechanism, but the interesting parts are the two
edges. First, cleanup: MDC is a `ThreadLocal` and request threads are pooled, so the
`MDC.clear()` has to be in a `finally` — if it is after `chain.doFilter` it gets skipped on
the exception path, and then that pooled thread carries the dead request's ID until
something overwrites it. That produces log lines confidently attributed to the wrong
customer, which is worse than no ID at all. I detect it with a saved query: any correlation
ID associated with more than one customer ID is a bug.

Second, propagation: MDC follows the thread, not the work. So it silently disappears across
`@Async`, any raw executor, Reactor's scheduler hops, and the Kafka listener threads. I fix
in-process hops with a `TaskDecorator` applied by a single executor factory — so there is
one place to get it right — and cross-process hops by putting the ID in a Kafka header and
restoring it in the consumer. If Micrometer's context-propagation is present I use
`ContextPropagatingTaskDecorator` so tracing and logging share one mechanism instead of two.

I also put the filter at `HIGHEST_PRECEDENCE` so it runs before the Spring Security chain —
an authentication failure is exactly the kind of event you want to be able to trace — and I
sanitise the inbound header, because an unvalidated header goes straight into a
line-delimited log stream and can be used to forge entries."

**What separates them:** the mid answer names the tool. The senior answer names the **two
failure directions** (leak and loss), gives the `finally` reason with the specific exception
path, proposes a **detection query** rather than only a fix, and treats the inbound header as
untrusted input. Filter ordering relative to Security is a detail almost nobody volunteers
and it signals real production experience.

**Follow-up:** "Your service moved to virtual threads. What changes?" — The leak shape
largely goes away because virtual threads are not reused across requests, but the
propagation problem does not: any explicit hand-off to another executor still loses it, and
the `finally` in shared library code is still correct. The right answer says which half
changes and which does not.

---

### Q2 — "Why `log.debug("order {}", id)` instead of `log.debug("order " + id)`?"

**Mid-level answer:** "It is faster because the string is only built if DEBUG is enabled."

**Senior answer:** "That is half of it and it is the less important half. The performance
point is real — arguments are evaluated before the call, so the concatenated form builds a
string on every invocation regardless of level, and in a per-order-line loop at 400 orders
a second that is a measurable allocation rate for output nobody reads. You can see it in an
allocation profile as `StringConcatFactory` sites under your service frames with DEBUG
switched off, which is a satisfying thing to point at in a review.

But the reason I would enforce it even if it were free is **structure**. The parameterised
form keeps the arguments separate from the message template. That is what lets a structured
encoder emit `orderId` as a real JSON field instead of melting it into a sentence. Once it
is a field, 'count declines by gateway and response code' is a query rather than a regular
expression that breaks when someone rewords the message.

The related mistake I look for in review is the exception placeholder:
`log.error("failed {}", ex)` renders `ex.toString()` and **loses the stack trace**, because
SLF4J only prints a trace for a trailing `Throwable` with no placeholder. The correct form
is `log.error("failed for order {}", orderId, ex)`. That one is worth a static-analysis rule,
because it is invisible until the incident where you need the trace."

**What separates them:** reframing performance as the secondary reason and **structure** as
the primary one. Then volunteering the `Throwable` placeholder bug, which is the same
category of defect and costs an hour during a real incident.

**Follow-up:** "When is `if (log.isDebugEnabled())` still worth writing?" — Only when
computing the *arguments* is expensive: a graph dump, a serialisation, a database call you
should not be making from a log statement anyway. The formatting is already lazy.

---

### Q3 — "Walk me through what happens to your logs when the log destination gets slow."

**Mid-level answer:** "Logging might slow down. We would use an async appender."

**Senior answer:** "By default, `log.info` is a synchronous call that writes on the calling
thread while holding the appender's lock. So a slow sink turns into request latency on
every endpoint, including ones that touch no database — and it correlates with error rate,
because errors log more, so the incident amplifies itself. The evidence is a thread dump
showing request threads inside `OutputStreamAppender` frames; nothing else in the system
points at logging, which is why this one gets misdiagnosed as a database problem.

`AsyncAppender` is the right shape, but the defaults are a trap. `discardingThreshold`
defaults to 20, which means once the queue is 80% full it silently drops TRACE, DEBUG and
INFO. So during your worst minute — precisely when you most need the timeline — the log has
holes, with the ERROR lines surviving and their surrounding context gone. There is no
warning about this.

So I set the policy explicitly and write down which trade I chose. For a payment service I
use `discardingThreshold=0` and `neverBlock=false`: never lose a line, accept that a
pathological sink will slow the application. For a high-volume, low-value stream the
opposite choice is defensible. I also set `includeCallerData=false`, because capturing file
and line numbers walks the stack per event on the producing thread.

And in a container I log to stdout rather than a file, so the platform owns collection and
rotation instead of me owning a sidecar and a volume."

**What separates them:** knowing the `discardingThreshold` default and what it costs you.
That single fact is the difference between someone who added `AsyncAppender` because a blog
said to and someone who has read the class. Framing block-versus-drop as a **deliberate,
written-down trade** is the senior move.

**Follow-up:** "How would you prove logging is on your critical path?" — Run the load
baseline at INFO and DEBUG and compare p99, and take a thread dump during the DEBUG run.
Reasoning about it is not proof.

---

### Q4 — "A support ticket says a customer's wallet was debited but the order failed. You have 10 minutes. Go."

**Mid-level answer:** "Search the logs for the customer ID around that time and read
through what happened."

**Senior answer:** "First I ask for the correlation ID, because our error page shows it and
we echo it in the `X-Correlation-Id` response header. With that, one query returns every
line from all six components across both processes, ordered — the HTTP filter, the order
service, the wallet debit, the gateway call, the outbox write, and the projection consumer.
I am looking for three things in order: did the wallet debit line commit, did the payment
line show a decline or a timeout, and did the saga's compensating credit run (Topic 117).

If they cannot give me the ID, I fall back to `customerId` plus a time window, and that is
worse but workable because `customerId` is an indexed MDC field rather than text inside a
message.

Two things I would check about the *logs themselves* before trusting them. One: is the
correlation ID present on every line, or does the trail stop at an async boundary — because
if the compensating credit runs on a `@Scheduled` thread with no propagation, the absence
of a credit line is not evidence the credit did not happen. Two: is that correlation ID
associated with exactly one customer ID? If it is associated with two, the MDC leaked and I
am reading someone else's request.

Then I would use the trace ID from the same line to pull the distributed trace (Topic 119)
for the timing, and check the payment gateway's own metrics for the same window (Topic 118).
Logs give me the *why*, the trace gives me the *where*, metrics tell me whether it was one
customer or the leading edge of an incident — which decides whether this is a ticket or a
page."

**What separates them:** having a **procedure** rather than an instinct; knowing the ID is
already in the customer's hands because you echoed it; and — the part that gets people
hired — **auditing the evidence before trusting it**. Someone who asks "could this
correlation ID be lying to me" has been burned by a leaked MDC and fixed it.

**Follow-up:** "What if the trail stops at the Kafka boundary?" — Then the outbox relay is
not writing the correlation header, and the consumer's fallback value (`kafka-orphan` in our
setup) is greppable, so a spike in it is an alertable regression.

---

### Q5 — "What should never appear in a log line, and how do you enforce it?"

**Mid-level answer:** "No passwords, no card numbers. We review for it and we can mask with
a regex."

**Senior answer:** "Secrets never, at any level, because a secret in a log is a secret in
whatever third-party platform ingests the log with whatever retention it has. PII only
where there is a stated reason and a retention policy — I would keep a customer ID because
it is a pseudonymous key I need for support, and keep the email out.

Enforcement in layers, because any single layer fails. The strongest is the **type**: give
`CardNumber` a `toString` that returns the last four digits, so a careless
`log.debug("card {}", card)` written by someone who has never read our guidelines is still
safe. That survives refactors and new hires in a way a review checklist does not — and it
matters that records generate a `toString` from their components, so you must override it
deliberately.

Second, a flat rule: never log a serialised request or response object. That one line —
`log.debug("request {}", mapper.writeValueAsString(req))` — bypasses every `@JsonIgnore`
and every DTO boundary we built.

Third, static analysis in CI so the rule is mechanical rather than social.

A masking regex at the appender is a backstop, not a control: it catches card-shaped
strings and misses anything base64-encoded or split across fields.

The operational half people forget: raising a log level in production is a change. If the
Actuator `loggers` endpoint is exposed, anyone can turn on DEBUG for the payments package
during an incident — which is exactly when nobody is thinking about data classification.
That endpoint needs authentication, and level changes should be audited."

**What separates them:** treating the **type system as the enforcement mechanism** rather
than relying on discipline, and identifying the runtime log-level change as the realistic
attack path. The mid answer describes intent; the senior answer describes controls that
work when people are tired.

**Follow-up:** "Someone needs the full gateway response to debug an integration. What do
you do?" — Time-boxed, a dedicated logger not enabled by default, redaction applied at
construction rather than at output, and a written expiry — not a permanent DEBUG line.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `AsyncLocalStorage` follows the async operation; MDC follows the thread. From that one
   difference, derive **both** MDC failure modes without recalling them from the text — and
   explain why one of them cannot happen in Node at all.

2. You add `MDC.clear()` in a `finally` in your servlet filter. Name one place in
   `orderflow` where that `finally` does **not** help, and say why.

3. A colleague proposes putting the full `PlaceOrderCommand` in the MDC "so we have it on
   every line". Give three separate reasons this is wrong, each from a different topic.

4. Your service runs on virtual threads (Topic 101). A virtual thread is created per task
   and discarded. Which of this document's five traps stop being possible, which remain
   exactly as dangerous, and which become *harder* to detect? Justify each.

5. `AsyncAppender` with `neverBlock=false` will block a request thread when the queue is
   full. Argue that this is the correct default for `orderflow`. Then argue it is wrong.
   Which side is stronger, and does your answer change for the inventory-projection consumer
   rather than the order API?

6. Structured JSON logging makes every field queryable. Your aggregator charges by indexed
   field. Derive the design rule this implies about what belongs in MDC versus what belongs
   in the message text — and check it against the `sku` field in Example 2.

7. You have metrics (118), traces (119) and logs (120). For each of these three questions,
   say which signal answers it and why the other two cannot: "is the error rate rising?",
   "where did the 900 ms go?", "why did *this* order fail?". Then name the field that joins
   all three, and what breaks if it is missing on one of them.

---

## Quick reference card

### The three things people conflate

| | What it is | You import it? |
|---|---|---|
| **SLF4J** | the API — `org.slf4j.Logger` | **yes, only this** |
| **Logback** | the implementation that writes bytes | no — configuration only |
| **MDC** | per-thread key/value map merged into events | `org.slf4j.MDC` |

### MDC API

```java
MDC.put(k, v);                 MDC.get(k);
MDC.remove(k);                 MDC.clear();
MDC.putCloseable(k, v)         // try-with-resources; cannot leak
MDC.getCopyOfContextMap()      // MAY RETURN NULL
MDC.setContextMap(map)         // throws on null
```

### Logging statement rules

```java
log.info("order {} placed", id);            // parameterised — DO
log.info("order " + id + " placed");        // concatenated — DO NOT
log.error("failed for order {}", id, ex);   // Throwable last, NO placeholder — DO
log.error("failed {}", ex);                 // loses the stack trace — DO NOT
if (log.isDebugEnabled()) { ... }           // only when the ARGUMENTS are expensive
```

### Where MDC is lost (fix each explicitly)

```
@Async                          -> TaskDecorator on the executor
raw ExecutorService             -> TaskDecorator, via one shared factory
CompletableFuture.supplyAsync   -> pass a decorated executor, never the default
Reactor publishOn / flatMap     -> Reactor Context, not MDC (Topic 108)
Kafka producer -> consumer      -> a message header, restored in the listener
@Scheduled (outbox relay)       -> set it at the top of the scheduled method
a new process                   -> a header, always
```

### Logback config essentials

```xml
logback-spring.xml              <!-- NOT logback.xml: -spring sees Spring properties -->
<springProfile name="local">    <!-- profile-scoped blocks -->
<springProperty .../>           <!-- pull a value from the Environment -->
%X{correlationId:-NONE}         <!-- MDC in a pattern, with a visible default -->
<discardingThreshold>0</...>    <!-- default is 20 = silently drop INFO at 80% full -->
<neverBlock>false</neverBlock>  <!-- block rather than lose; choose deliberately -->
<includeCallerData>false</...>  <!-- true walks the stack per event -->
```

### Boot properties worth knowing

```properties
logging.level.com.orderflow=DEBUG
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}]
management.tracing.baggage.correlation.fields=correlationId,customerId
logging.structured.format.console=ecs        # Boot 3.4+; see the uncertainty note
```

```bash
curl -X POST localhost:8080/actuator/loggers/com.orderflow \
  -H 'Content-Type: application/json' -d '{"configuredLevel":"DEBUG"}'
```

### Gotchas checklist

- [ ] `MDC.clear()` is in a `finally`, not after the call.
- [ ] Every executor you own has a `TaskDecorator`, applied by one shared factory.
- [ ] `@Async` names its executor explicitly.
- [ ] Kafka producers write the correlation header; consumers restore and then clear it.
- [ ] No concatenation in log statements. No placeholder for the `Throwable`.
- [ ] No secrets at any level. No serialised request/response objects.
- [ ] Sensitive value types override `toString`.
- [ ] `discardingThreshold` and `neverBlock` are set deliberately, not defaulted.
- [ ] Container logs go to stdout, not to a file.
- [ ] Exactly one SLF4J binding on the classpath.
- [ ] A saved query detects one correlation ID spanning multiple customer IDs.

---

## When would I use this at work?

**1. The five-minute support answer.**
"Customer says their wallet was debited but the order failed." With a correlation ID
echoed to the customer and propagated across six components and two processes, that is
three steps and five minutes. Without it, it is an engineer reading two log streams by
timestamp at 400 requests per second, and the honest answer to support is "we cannot tell".
This is the single highest-value thing in the document and it costs one filter and one
`TaskDecorator`.

**2. Catching a silent evidence-corruption bug in review.**
A pull request adds a new `ThreadPoolTaskExecutor` for a batch job and calls `MDC.put` at
the top of the task. You ask two questions: where is the decorator, and where is the
`finally`. Without this topic those lines look fine and the consequence is discovered
months later during an incident, when the timeline you build turns out to belong to a
different customer. Reviews that catch *silent* defects are where senior engineers earn
their difference.

**3. Deciding what logging costs, with numbers.**
Someone proposes DEBUG-level logging in production "for observability", or proposes a
socket appender to a log service. You run the Topic 65 baseline twice, take a thread dump
during the second run, and come back with a p99 delta and a stack showing whether the cost
is on the request thread or the appender thread. Then you can say "yes, at this
configuration, and here is the queue policy it needs" instead of having an opinion contest.
That is the same instinct as Topic 122's "measure before you optimise", applied to a system
everybody assumes is free.

---

## Connected topics

**Prerequisites:**
- **18 — Strings**: why concatenation in a loop allocates, and why the parameterised form is
  not merely stylistic.
- **38 — Bean scopes**: request-scoped context is the alternative to MDC, and knowing why
  MDC usually wins (it needs no injection point) is a design decision, not a habit.
- **56 — Spring Security filter chain**: your correlation filter must run before it, or the
  most interesting failures are the untraceable ones.
- **79 — `ThreadLocal` leaks**: MDC is a `ThreadLocal`. Trap 1 is Topic 79's leak shape with
  the damage being wrong data instead of retained memory.
- **90 — Executor shutdown and bounded queues**: every executor that needs a `TaskDecorator`
  is an executor that also needs a bound and a shutdown policy.

**This unlocks:**
- **118 — Micrometer metrics**: the cardinality rule is the mirror image of the MDC rule.
  High-cardinality values (order ID, customer ID) belong in log fields, never in metric tags.
  Knowing which signal carries which value is the actual skill.
- **119 — Distributed tracing**: `traceId` and `spanId` arrive in the MDC and become log
  fields. That is the join between the two signals, and both are lost at exactly the same
  thread boundaries — so one `TaskDecorator` fixes both.
- **121 — Actuator**: the `loggers` endpoint changes levels at runtime, which is powerful and
  is a write endpoint that must not be public.
- **123 — Config and secrets**: where secrets come from, and why a secret must never reach a
  logger regardless of level.
- **124 — Production-readiness gate**: "can we answer a customer question from logs in five
  minutes" is an observability-coverage line item in the review, and the correlation-ID
  detection query is one of the controls you list.
- **133 — Postmortems**: your timeline is built from these log lines. A leaked MDC produces a
  postmortem that blames the wrong change.

**Also related:**
- **101 — Virtual threads**: changes the leak shape without removing the propagation problem.
  Exercise 3, Part D is the experiment that tells you which of your `finally` blocks are
  still load-bearing.
- **108 — Reactive context**: MDC cannot survive operator thread hops. Reactor's Context is
  the carrier there, and the bridge between them is explicit work.
- **113 — Kafka consumer groups**: the listener container thread is long-lived and reused
  across messages, so it needs both `put` and `clear`, exactly like an HTTP thread.
- **115 — Outbox**: the outbox row is the natural place to carry the correlation ID across
  the process boundary, because it is written in the same transaction as the business change.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0. Two things in this
document are deliberately hedged rather than asserted: the exact set of
`logging.structured.json.*` customisation properties on Boot 4.1 (the feature exists from
3.4; the sub-property names are what I will not spell from memory), and whether Boot 4.1
auto-applies `ContextPropagatingTaskDecorator` to auto-configured executors. Both are
settled in under two minutes by the probes in the Hands-on section, and neither changes the
mechanism. The mechanism — MDC is a `ThreadLocal`, it does not follow the work, and it must
be cleared on any thread you did not create — has been true since SLF4J 1.x and will still
be true the next time it puts the wrong customer in your incident timeline.*
