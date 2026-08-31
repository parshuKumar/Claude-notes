# 106 — WebFlux vs MVC: Threading Models and When Each Wins

## Phase: 10 — Reactive & Async at Scale
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21. Netty, Reactor and Tomcat versions come from the Spring Boot BOM — never pin them by hand, and never quote a version number from this document.
## Project spine: a WebFlux module for `orderflow`'s payment-callback ingestion, kept deliberately separate from the MVC core, so that both can be benchmarked against the Topic 65 baseline at matched arrival rates. The output is a three-way comparison table that Topic 107 consumes directly.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **Spring MVC assigns one thread to a request for the request's entire lifetime. That
> thread is occupied while the database is thinking, while the payment gateway is
> thinking, and while the response is being written. Concurrency is bounded by the thread
> pool size.**
>
> **Spring WebFlux serves requests on a small, fixed set of event-loop threads — typically
> one per core — each multiplexing many connections. A request occupies a loop only while
> its code is actually executing; between the request going out and the response coming
> back, the loop serves other connections.**
>
> **That saves thread stacks and context switches. And it means one blocking call on an
> event-loop thread stalls every connection assigned to that loop — which is the single
> most damaging mistake in reactive Java.**

The trade is stated in one sentence: **WebFlux buys thread economy and charges you
absolute discipline about blocking.** Everything else in this document is working out
whether that trade is worth it for `orderflow`, with numbers rather than opinions.

---

## The bridge from what you know

### The inversion: WebFlux is the model you already have. MVC is the one you don't.

Every other topic in this curriculum has taught you something Java does that Node does
not. This one is the other way round, and noticing that is worth a lot.

**Spring WebFlux on Netty is structurally libuv.** A small number of threads, each running
an `epoll_wait` loop over registered sockets, dispatching readiness events to handlers.
Topic 103 established this in detail and called it the closest honest analogue in the whole
curriculum. Everything you know applies:

- Never block the loop.
- A slow handler delays every other connection on that loop.
- I/O is registered and returns immediately; the continuation runs later.
- CPU-heavy work goes somewhere else.

**Spring MVC's thread-per-request model is the one with no Node analogue.** Node has never
had it and could not — there is one thread. The idea that a request gets its *own* thread,
which then sits in a kernel wait while Postgres thinks, is genuinely foreign to you, and
it is worth naming what it buys, because it is not nothing:

| | Thread-per-request (MVC) | Event loop (WebFlux, Node) |
|---|---|---|
| Where "what happens next" lives | **The thread's stack.** It is just the next line. | A callback, a promise, an operator |
| Stack traces | Complete. One trace, one logical request. | The subscription stack, not the assembly stack (Topic 108) |
| Debugger | Breakpoint, step, inspect locals | Steps through framework internals |
| `try`/`catch`/`finally` | Works over the whole request | Works within one operator |
| `ThreadLocal` context | Works (MDC, `SecurityContext`) | **Does not work.** Needs `Context` (Topic 108) |
| Cost of an in-flight request | An OS thread and its stack | A subscription and a few objects |
| Concurrency ceiling | The pool size | Memory |

**The single sentence:** *MVC keeps the continuation on a stack and pays for the stack.
WebFlux keeps the continuation on the heap and pays in programming model.*

And now say the thing that makes Topic 107 interesting: **virtual threads (Topic 101) keep
the continuation on a stack that lives on the heap, and try to pay neither.** That is the
whole argument, and this document produces the data for it.

### The one place your Node instincts will actively mislead you

In Node, every database driver is non-blocking. `pg`, `mysql2`, `mongodb` — all of them.
You have never had to ask whether a client library was safe to call from the loop, because
there was no other kind.

**In Java, the default is the opposite.** JDBC is a blocking API by specification —
`Statement.executeQuery` blocks the calling thread until the database answers, and no
amount of wrapping changes that. Hibernate is built on JDBC. Spring Data JPA is built on
Hibernate. **The entire mainstream Java persistence stack is blocking, and it is the part
of your service that does the most waiting.**

So "just use WebFlux" in Java is not the small change it would be in Node. It means:

- **R2DBC instead of JDBC.** A different driver, a different API, a different transaction
  abstraction.
- **No Hibernate.** No persistence context, no dirty checking, no lazy loading, no
  `@OneToMany` graph fetching. Spring Data R2DBC is a much thinner mapper; you write more
  SQL.
- **Everything from Topics 48–53 stops applying** to the reactive path. The N+1 fix, the
  batching configuration, the `@Version` optimistic locking — all of it was Hibernate.
- **`@Transactional` works differently.** Reactive transactions exist, but they are bound
  to the subscription rather than to a thread, which changes how they compose.

That is not a warning against WebFlux. It is the honest scope of the commitment, and it is
the single largest input to the decision. Anyone who says "WebFlux is faster, we should
switch" without mentioning R2DBC has not costed the change.

**Verdict: STRONG ANALOGUE for WebFlux — it is libuv with N loops, and your discipline
transfers whole. NO ANALOGUE for MVC's thread-per-request model. And a specifically
misleading instinct about driver availability, which is the biggest cost in the decision.**

---

## What is this?

Two complete, parallel web stacks that ship in the same framework and share a
programming-model surface.

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Starter | `spring-boot-starter-web` | `spring-boot-starter-webflux` |
| Specification | Servlet (Jakarta Servlet) | Reactive Streams |
| Default server | Tomcat | **Netty** (Jetty, Undertow, Tomcat also possible) |
| Front controller | `DispatcherServlet` | `DispatcherHandler` |
| Threading | One thread per request, from a pool | Event loops, one per core |
| Return types | `Order`, `ResponseEntity<Order>`, `List<Order>` | `Mono<Order>`, `Flux<Order>`, plus the blocking types |
| HTTP client | `RestClient` / `RestTemplate` | `WebClient` |
| Database | JDBC, Hibernate, Spring Data JPA | **R2DBC**, Spring Data R2DBC |
| Test client | `MockMvc`, `RestTestClient` | `WebTestClient` |
| Streaming | Servlet async, `SseEmitter` | Native: `Flux<T>` as SSE or NDJSON |
| Security | `SecurityContextHolder` (`ThreadLocal`) | Reactive security via `Context` |

**Annotated controllers work on both.** This surprises people and it matters:

```java
@RestController
public class ProductController {

    // Compiles and runs on MVC. Also compiles and runs on WebFlux.
    @GetMapping("/products/{id}")
    public Product byId(@PathVariable long id) { ... }

    // WebFlux only in spirit -- MVC also accepts a Mono return, awkwardly.
    @GetMapping("/products/{id}")
    public Mono<Product> byIdReactive(@PathVariable long id) { ... }
}
```

So the annotation model is not the difference. **The difference is the threading model
underneath and what your code is allowed to do on it.**

### The functional routing alternative

WebFlux also offers a functional style, which has no MVC equivalent in common use:

```java
@Bean
RouterFunction<ServerResponse> callbackRoutes(CallbackHandler handler) {
    return RouterFunctions.route()
            .POST("/internal/payments/callbacks", handler::receive)
            .GET("/internal/payments/health", handler::health)
            .build();
}
```

Use it when routing is the interesting part (composition, filters, dynamic routes). For a
handful of endpoints, annotations are fine and more familiar to reviewers. This is a style
choice, not a performance one.

### The thing Spring Boot does that catches everyone

**If both starters are on the classpath, Spring Boot builds a servlet application.** Not a
reactive one. The WebFlux annotated controllers still work — on the servlet stack, with
thread-per-request — and you get none of the thread economy while believing you do.

```properties
# Force it, and then VERIFY, because this is silently wrong otherwise:
spring.main.web-application-type=reactive
```

```java
// The verification. One line, at startup, and it has saved entire benchmarks:
log.info("web application type = {}", context.getClass().getSimpleName());
// ReactiveWebServerApplicationContext  -> WebFlux
// ServletWebServerApplicationContext   -> MVC
```

This is Hands-on Proof 1 and it is not optional. **A benchmark comparing MVC to a WebFlux
module that is secretly running on Tomcat produces a table of noise, and people publish
those tables.**

---

## Why does it matter?

**1. "WebFlux is faster than MVC" is false as usually stated, and being able to say why is
a senior signal.** At the same throughput, with the same downstream, it is not faster. It
uses fewer threads. That only converts into a win when thread count is the binding
constraint — which, for most services, it is not. Topic 101's baseline re-run probably
already told you that about `orderflow`.

**2. The failure mode is total, not gradual.** MVC under overload degrades: threads fill,
the accept queue fills, connections are refused, latency rises. Ugly, and it stays up.
WebFlux with one blocking call collapses: throughput drops to roughly
`loops / blocking-duration` requests per second and CPU sits idle. **The same mistake costs
20% in MVC and 99% in WebFlux.**

**3. It is an all-or-nothing architectural commitment, and it is easy to make by
accident.** R2DBC instead of Hibernate is a large, permanent decision about how your
persistence layer is written. Choosing it because a benchmark blog post claimed a number is
how teams end up two years into a rewrite they cannot justify.

**4. `orderflow` has exactly one endpoint where the argument is genuinely live.** Payment
callbacks: many concurrent connections, small payloads, a genuine push source (Topic 105),
and the only place in the system where demand propagation matters. Everything else —
catalogue, orders, placement — is request/response where the caller waiting *is* the
backpressure. **This document is about drawing that boundary with data.**

**5. It produces the table Topic 107 needs.** MVC on platform threads, MVC on virtual
threads, WebFlux — the same workload, the same arrival rates, the same dataset. That
three-way comparison is the deliverable, and Topic 107's recommendation is not writable
without it.

---

## Machine-level reality

### MVC thread accounting

```
Client connection
  -> Tomcat acceptor thread accepts, hands to the poller
  -> Poller assigns a WORKER thread from the pool
  -> Worker runs the ENTIRE request:
       filter chain (Topic 56)
       DispatcherServlet -> HandlerMapping -> HandlerAdapter
       your controller
       your service
       JDBC call  <-- THREAD PARKS IN THE KERNEL HERE, doing nothing, for 20 ms
       serialization
       write the response
  -> Worker returns to the pool
```

The numbers that decide everything:

| Setting | Boot default (verify yours) | What it bounds |
|---|---|---|
| `server.tomcat.threads.max` | 200 | **Maximum concurrent in-flight requests** |
| `server.tomcat.threads.min-spare` | 10 | Threads kept warm |
| `server.tomcat.accept-count` | 100 | OS accept-queue backlog once all threads are busy |
| `server.tomcat.max-connections` | 8192 | Connections the server will hold at all |

> **Print yours rather than trusting the table.** `GET /actuator/configprops` with the
> actuator enabled, or log the values at startup. Defaults change between Boot versions and
> your `application.yaml` may already override them.

**The arithmetic that matters.** With 200 worker threads and a 50 ms average request,
Little's Law gives you a ceiling of `200 / 0.050 = 4,000` requests per second — if nothing
else is the constraint. At 200 ms per request it is 1,000 per second. **The thread pool is
a hard concurrency cap, and every request above it waits in the accept queue.**

Two costs beyond the cap:

- **Stack memory.** Each platform thread reserves stack address space (commonly ~1 MB by
  default, committed lazily). 200 threads is a modest commitment; 20,000 would not be, which
  is why you cannot simply raise `threads.max` to 20,000 and call it solved.
- **Context switches.** Every park and unpark is a kernel transition. At high thread counts
  with short work units, the scheduler's own overhead becomes measurable.

**What MVC does well, and it is genuinely well:** the accept queue plus a bounded pool is
**backpressure at the door** (Topic 90, Topic 93). When you are overloaded, connections
queue and then get refused. That is a graceful, well-understood degradation and it protects
the database. Do not throw it away casually.

### WebFlux thread accounting

```
Client connection
  -> Netty acceptor accepts, ASSIGNS the connection to one event loop
     (and it stays on that loop for the connection's lifetime)
  -> That loop's epoll_wait reports the socket readable
  -> Loop thread runs your handler chain:
       decode, route, your handler
       WebClient call -> registers interest, RETURNS IMMEDIATELY
  -> Loop goes and serves OTHER connections   <-- the entire point
  -> Response arrives; the loop that owns that connection resumes the chain
  -> Encode and write
```

The numbers:

| Setting | Documented default (verify) | What it bounds |
|---|---|---|
| `reactor.netty.ioWorkerCount` | `max(4, availableProcessors())` | **The number of event loops. Your entire request-serving parallelism.** |
| `reactor.netty.ioSelectCount` | Platform-dependent | Selector threads, where separate |
| `reactor.netty.pool.maxConnections` | Per-host connection pool for `WebClient` | Downstream concurrency |
| `Schedulers.boundedElastic()` cap | Derived from `availableProcessors()`, large | Offloaded blocking work |

> **Print it. Do not quote it.**
>
> ```java
> log.info("ioWorkerCount={}", reactor.netty.resources.LoopResources.DEFAULT_IO_WORKER_COUNT);
> log.info("processors={}",    Runtime.getRuntime().availableProcessors());
> ```
>
> And check the thread names at runtime — `reactor-http-nio-1` through `-N`. Counting them
> is more reliable than any constant.

**The two facts that produce every WebFlux failure:**

1. **A connection is bound to one loop for its lifetime.** Not per-request — per
   *connection*. With HTTP keep-alive, that is many requests. Block that loop and you
   affect every connection assigned to it, not just the current one.
2. **There are very few loops.** On a `--cpus=2` container there may be four. Four
   blocking calls and the service is doing nothing at all.

**The blocking arithmetic, which you should be able to do in your head:**

```
throughput_ceiling = ioWorkerCount / blocking_call_duration

8 loops, 20 ms blocking JDBC call  ->    400 requests/second. Hard ceiling.
8 loops, 200 ms payment call       ->     40 requests/second.
4 loops, 200 ms payment call       ->     20 requests/second.
```

Compare with MVC's `200 / 0.200 = 1,000` per second for the same 200 ms call. **A blocking
call makes WebFlux an order of magnitude worse than the framework it was supposed to
improve on.** That is not a small regression; it is the drill.

### Container reality — Topic 82, again

Both models derive their sizing from `availableProcessors()`, and both are wrong in a
container nobody checked:

| Container | `availableProcessors()` | Event loops | Consequence |
|---|---|---|---|
| `--cpus=8` | 8 | 8 | Fine |
| `--cpus=2` | 2 | 4 (the floor) | Four loops for the whole service |
| `--cpus=0.5` | 1 | 4 (the floor) | Four loops on half a core |

The `max(4, ...)` floor is why WebFlux does not collapse to a single loop on a small
container — but four loops sharing half a core is a different problem, and one blocking
call still takes out a quarter of your serving capacity.

### Where the memory actually goes

| | MVC, 10,000 concurrent requests | WebFlux, 10,000 concurrent requests |
|---|---|---|
| Threads | Impossible — the pool caps at 200; 9,800 wait in the accept queue | 8 loops |
| Thread stacks | 200 × stack reservation | 8 × stack reservation |
| Per-request state | On 200 stacks | 10,000 subscription object graphs on the heap |
| Where the queue is | The OS accept queue and the pool queue | Nowhere — all 10,000 are in flight |
| Failure mode | Connections refused; latency rises | Heap pressure; and **no backpressure unless you added it** (Topic 105) |

**Read the last row carefully, because it is counter-intuitive.** MVC's thread pool is an
accidental admission-control mechanism. WebFlux removes it, which is the point, and in
doing so removes the protection it was providing to everything downstream. A WebFlux
service happily accepts 10,000 concurrent requests and forwards all of them to a database
that can serve 20 at a time. **You must add back deliberately what MVC gave you by
accident** — a `Semaphore`, a bounded connection pool with a fail-fast timeout, or an
explicit rate limiter.

### `boundedElastic` — the escape hatch and its true cost

```java
Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
    .subscribeOn(Schedulers.boundedElastic())
```

This is the sanctioned way to call blocking code from a reactive chain: the blocking work
runs on a separate elastic pool, and the event loop is never blocked. It is correct and you
should use it where you must.

**But be precise about what you have built.** That pool is threads. Blocking work on those
threads is thread-per-blocking-operation. **You have reinvented MVC, on a second thread
pool, behind a reactive API — with worse stack traces, no `ThreadLocal` context, and a
harder debugging story.** If most of your requests go through `boundedElastic`, you have
all of WebFlux's costs and none of its benefits, and the honest move is to go back to MVC.

The legitimate uses are narrow: a single legacy blocking client, a file read, a library
with no reactive equivalent. Not your entire persistence layer.

### The request path, side by side

Same endpoint, same downstream, 20 ms database call, 8 cores.

| Step | MVC | WebFlux |
|---|---|---|
| 1 | Tomcat worker `http-nio-8080-exec-14` picks up the request | Loop `reactor-http-nio-3` decodes the request |
| 2 | Filter chain runs on that thread; `SecurityContext` set in a `ThreadLocal` | Filter chain runs on the loop; security context is in the Reactor `Context` |
| 3 | Controller calls the service, which calls JDBC | Handler returns a `Mono` from R2DBC; the driver registers interest and returns |
| 4 | **The thread parks in the kernel for 20 ms.** It serves nobody. | **The loop moves to another connection.** It serves ~thousands of other requests in those 20 ms. |
| 5 | Database responds; the same thread continues | Database responds; the loop that owns this connection resumes the chain — **possibly a different logical continuation than it was running a moment ago** |
| 6 | Serialize and write on the same thread | Serialize and write on the loop |
| 7 | Thread returns to the pool | Nothing to return |
| Concurrency ceiling | 200 in flight | Memory |
| If step 3 blocks | One thread of 200 is busy. 0.5% capacity lost. | **One loop of 8 is dead. 12.5% capacity lost, and every connection on it stalls.** |

The last row is the whole document.

---

## Example 1 — minimal

### 1a — the same endpoint, both ways, with the thread named

```java
// MVC version -- spring-boot-starter-web
@RestController
class MvcProbeController {

    @GetMapping("/probe/mvc")
    public Map<String, String> probe() throws InterruptedException {
        String before = Thread.currentThread().getName();
        Thread.sleep(50);                    // stands in for a database call
        String after = Thread.currentThread().getName();
        return Map.of("before", before, "after", after,
                      "virtual", String.valueOf(Thread.currentThread().isVirtual()));
    }
}
```

```java
// WebFlux version -- spring-boot-starter-webflux
@RestController
class FluxProbeController {

    @GetMapping("/probe/flux")
    public Mono<Map<String, String>> probe() {
        String assembly = Thread.currentThread().getName();
        return Mono.delay(Duration.ofMillis(50))          // non-blocking
                   .map(tick -> Map.of(
                        "assembly", assembly,
                        "after",    Thread.currentThread().getName()));
    }
}
```

```bash
for i in $(seq 1 5); do curl -s localhost:8080/probe/mvc  | jq -c; done
for i in $(seq 1 5); do curl -s localhost:8080/probe/flux | jq -c; done
```

**What to look for:** the thread names, and whether they change across the delay.

| What you see | What it means |
|---|---|
| MVC: `before` and `after` are the **same** `http-nio-8080-exec-N` | Thread-per-request. One thread owned the whole request, including the 50 ms of doing nothing. |
| MVC: different `exec-N` across the five calls | The pool is round-robining. Normal. |
| WebFlux: `assembly` is `reactor-http-nio-N`, `after` is `parallel-N` | **Two different threads served one request.** `Mono.delay` schedules on the parallel scheduler, so the continuation resumes there. This is the thread-hop that breaks `ThreadLocal`, MDC and stack traces (Topic 108). |
| WebFlux: only a handful of distinct `reactor-http-nio-N` names across many calls | **Count them.** That is `ioWorkerCount`, and it is your entire request-serving parallelism. |
| MVC: `virtual=true` | You have `spring.threads.virtual.enabled=true` set (Topic 101). Note it — it changes the comparison completely and is the third column of the benchmark. |

### 1b — the destructive one-line change

Now break the WebFlux version exactly as someone will break yours:

```java
    @GetMapping("/probe/flux-blocking")
    public Mono<Map<String, String>> broken() throws InterruptedException {
        Thread.sleep(50);                    // <-- BLOCKING. On the event loop.
        return Mono.just(Map.of("thread", Thread.currentThread().getName()));
    }
```

```bash
# Compare the two under identical concurrent load. 50 concurrent, 10 seconds.
hey -z 10s -c 50 http://localhost:8080/probe/flux
hey -z 10s -c 50 http://localhost:8080/probe/flux-blocking
hey -z 10s -c 50 http://localhost:8080/probe/mvc
```

**What to look for:** requests per second in each of the three runs, and CPU utilisation.

| What you see | What it means |
|---|---|
| `/probe/flux` sustains high throughput with a handful of threads | The model working. 50 concurrent requests on 8 loops, none of them blocked. |
| `/probe/flux-blocking` throughput collapses to roughly `ioWorkerCount / 0.050` | **The arithmetic, observed.** With 8 loops and a 50 ms block: about 160 requests per second. Not a slowdown — a hard ceiling. |
| `/probe/mvc` sustains roughly `min(threads.max, concurrency) / 0.050` | Around 1,000 per second at 200 threads. **MVC is six times faster than the broken WebFlux endpoint.** |
| CPU near idle during the blocking WebFlux run | **The signature.** Same as Topic 100's blocked FJP workers and Topic 101's pinned carriers. Collapsed throughput plus idle CPU means threads are parked somewhere they cannot be replaced. |
| The blocking run's latency is bimodal | Requests landing on an unblocked loop are fast; the rest wait behind everything else queued on their loop. |

**This ten-minute experiment is the most valuable thing in this document.** It shows that
the *same mistake* costs a few percent in MVC and everything in WebFlux, which is the real
content of "WebFlux demands discipline."

---

## Example 2 — production scenario (on the project spine)

### The constraints

Build the payment-callback ingestion path as a WebFlux module, kept separate from the MVC
core, and set up so both can be benchmarked against the Topic 65 baseline.

Real requirements:

1. Callbacks arrive from the payment provider, spiky, with settlement bursts (Topic 105).
2. **No callback may be lost.** Money.
3. The MVC core — catalogue, orders, placement — stays on Hibernate and Spring Data JPA.
   Rewriting it on R2DBC is not on the table and would not be justified.
4. Ingestion must not degrade the MVC core when a burst arrives.
5. Both paths must be benchmarkable at matched arrival rates, so Topic 107's decision has
   data.
6. It must be operable by people who did not write it, at 3am.

### The decision that comes before any code

**You cannot get both stacks in one Spring Boot application** and have both threading
models. Both starters on the classpath means a servlet application; WebFlux controllers
then run on Tomcat's thread pool with no thread economy at all. So requirement 5 forces a
structural choice, and there are exactly three honest options:

| Option | What it is | Pro | Con |
|---|---|---|---|
| **A — separate deployable** | `orderflow-ingest`, its own image, its own pod, WebFlux + R2DBC | True isolation; independent scaling; a clean benchmark; a burst cannot touch the core's heap or GC | A second service to deploy, monitor and page for |
| **B — one app, MVC only, callbacks on virtual threads** | Keep the servlet stack; `spring.threads.virtual.enabled=true` | Simplest; one deployable; keeps Hibernate | No backpressure and no thread economy beyond what Loom gives; does not answer the WebFlux question |
| **C — one app, forced reactive** | `spring.main.web-application-type=reactive` for everything | One stack | **Forces the entire MVC core onto R2DBC.** Not on the table per requirement 3. |

**Choose A.** It is the only option that satisfies requirements 4 and 5, and separate
deployables give the isolation requirement 4 asks for anyway — a settlement burst that
fills a heap fills *that* heap, and the catalogue endpoint is in a different process.

Say the honest cost out loud: **you now have two services.** Two images, two dashboards,
two on-call runbooks, and a shared database that both write to. That is a real operational
tax and it is the main argument for option B. The reason to pay it here is that requirement
5 — being able to benchmark them independently — is itself the point of the exercise.

### The module

```
orderflow/
  orderflow-core/         MVC + Hibernate + Spring Data JPA. Unchanged.
  orderflow-ingest/       WebFlux + R2DBC. Payment callbacks ONLY.
  orderflow-domain/       Shared records, no framework dependencies.
```

```xml
<!-- orderflow-ingest/pom.xml -->
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
  </dependency>
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>
  <!-- NOTE: spring-boot-starter-web is ABSENT, deliberately. If it appears
       transitively, this module silently becomes a servlet app. See the
       enforcement below -- this is not a theoretical risk. -->
</dependencies>

<build><plugins>
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <executions><execution>
      <goals><goal>enforce</goal></goals>
      <configuration><rules><bannedDependencies><excludes>
        <exclude>org.springframework.boot:spring-boot-starter-web</exclude>
        <exclude>org.springframework.boot:spring-boot-starter-jdbc</exclude>
        <exclude>org.springframework.boot:spring-boot-starter-data-jpa</exclude>
      </excludes></bannedDependencies></rules></configuration>
    </execution></executions>
  </plugin>
</plugins></build>
```

**The enforcer rule is the important part of this file.** Topic 32 taught you that Maven
flattens to one version per artifact with nearest-wins, silently. A transitive
`spring-boot-starter-web` from some shared library turns this module into a servlet
application with no error anywhere. **Ban it at build time**, because you will not notice
it at runtime until a benchmark tells you something impossible.

### The startup assertion

```java
@Component
class ReactiveStackAssertion {

    @EventListener(ApplicationReadyEvent.class)
    void assertReactive(ApplicationReadyEvent event) {
        String contextType = event.getApplicationContext().getClass().getSimpleName();
        log.info("web application type = {}", contextType);
        log.info("availableProcessors  = {}", Runtime.getRuntime().availableProcessors());
        log.info("event loop threads   = {}", countEventLoopThreads());

        if (!contextType.contains("Reactive")) {
            throw new IllegalStateException(
                "orderflow-ingest is running on the SERVLET stack. A "
              + "spring-boot-starter-web dependency has leaked in. "
              + "Every benchmark from this process is invalid.");
        }
    }

    private long countEventLoopThreads() {
        return Thread.getAllStackTraces().keySet().stream()
                .filter(t -> t.getName().startsWith("reactor-http-nio"))
                .count();
    }
}
```

**Fail the startup.** A service that silently runs the wrong stack produces months of
wrong conclusions, and this is eight lines.

### The handler

```java
package com.orderflow.ingest;

@RestController
public class PaymentCallbackController {

    private final PaymentCallbackInbox inbox;         // R2DBC, non-blocking
    private final Counter accepted, rejected;

    /**
     * Persist first, acknowledge second (Topic 105). Nothing in this method blocks.
     * Nothing in this method may EVER block -- there are only a handful of loops.
     */
    @PostMapping("/internal/payments/callbacks")
    public Mono<ResponseEntity<Void>> receive(@RequestBody @Valid PaymentCallback callback) {
        return inbox.insertIfAbsent(callback)                 // Mono<Void>, R2DBC
                .doOnSuccess(v -> accepted.increment())
                .thenReturn(ResponseEntity.accepted().<Void>build())
                .onErrorResume(DataAccessResourceFailureException.class, e -> {
                    rejected.increment();
                    // 503 -> the provider RETRIES. Backpressure across the network.
                    return Mono.just(ResponseEntity
                            .status(HttpStatus.SERVICE_UNAVAILABLE)
                            .header(HttpHeaders.RETRY_AFTER, "30")
                            .<Void>build());
                });
    }
}
```

```java
package com.orderflow.ingest;

@Repository
public class PaymentCallbackInbox {

    private final DatabaseClient client;      // R2DBC. NOT JdbcTemplate.

    public Mono<Void> insertIfAbsent(PaymentCallback callback) {
        return client.sql("""
                INSERT INTO payment_callback_inbox
                    (provider, callback_id, order_id, payload, received_at, status)
                VALUES (:provider, :callbackId, :orderId, :payload, now(), 'PENDING')
                ON CONFLICT (provider, callback_id) DO NOTHING
                """)
            .bind("provider",   callback.provider())
            .bind("callbackId", callback.callbackId())
            .bind("orderId",    callback.orderId())
            .bind("payload",    callback.rawJson())
            .then();
    }
}
```

**Notice what is missing and why:**

- No `@Transactional` in the Hibernate sense. A single statement is atomic; the idempotency
  is the unique constraint, not a transaction. Simpler is correct here.
- No entity, no persistence context, no dirty checking. **All of Topics 48–53 are absent
  from this module by construction.** That is the R2DBC commitment, made concrete.
- No `JdbcTemplate` anywhere. One `jdbcTemplate.update(...)` on a loop thread and this
  module's throughput ceiling drops by an order of magnitude (the failure drill).

### The processing side stays on MVC

The callback still has to be *processed* — order lookup, inventory decrement, event publish
— and that work is Hibernate-shaped and belongs in the core:

```java
// In orderflow-core: MVC, Hibernate, virtual threads. A scheduled pump reads the
// durable inbox and processes at a rate the database sustains.
@Scheduled(fixedDelay = 200)
public void pump() {
    List<PaymentCallback> batch = inboxRepository.claimBatch(64);
    for (PaymentCallback callback : batch) {
        processor.process(callback);      // blocking, transactional, Hibernate
    }
}
```

**This is the shape of the whole recommendation, and it is worth stating explicitly:**
**WebFlux at the edge, where connection count is high and work is trivial. Blocking,
transactional, Hibernate code behind a durable buffer, where the work is complex and the
concurrency is bounded by the database anyway.** The boundary between them is a table, and
the table is also the backpressure mechanism (Topic 105) and the crash-safety mechanism
(Topic 115).

### The benchmark harness

Requirement 5 means the k6 scripts must be able to hit either implementation:

```javascript
// k6/callback-burst.js -- one script, three targets
const TARGET = __ENV.TARGET;     // 'mvc-platform' | 'mvc-virtual' | 'webflux'
const BASE   = __ENV.BASE_URL;

export const options = {
  scenarios: {
    // The Topic 65 baseline mix, unchanged.
    baseline: {
      executor: 'constant-arrival-rate',
      rate: __ENV.BASELINE_RATE, timeUnit: '1s',
      duration: '10m', preAllocatedVUs: 500, maxVUs: 5000,
      exec: 'baselineMix',
    },
    // The settlement burst, INDEPENDENTLY controlled. This must be an open model:
    // a closed loop throttles itself when the service slows, which hides the effect
    // you are trying to measure.
    burst: {
      executor: 'constant-arrival-rate',
      rate: __ENV.BURST_RATE, timeUnit: '1s',
      startTime: '3m', duration: '5m',
      preAllocatedVUs: 1000, maxVUs: 20000,
      exec: 'callbackBurst',
    },
  },
};
```

Run the identical script against all three configurations. **Everything else — dataset,
JVM flags, container limits, seed — held constant.** That is what makes the table
comparable, and it is the whole reason this module exists.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a blocking call on an event-loop thread

**The most damaging mistake in reactive Java.** It appears in Topics 103, 105, 106 and 108
because it is the one that turns a working service into an outage.

**Wrong approach.** It rarely looks like a mistake:

```java
@GetMapping("/internal/payments/callbacks/{id}")
public Mono<Callback> byId(@PathVariable String id) {
    Callback c = jdbcTemplate.queryForObject(...);      // blocking JDBC, on the loop
    return Mono.just(c);
}

// Or hidden one layer down, in a shared library:
return Mono.just(legacyAuditService.recordAccess(id));  // this does a blocking HTTP call

// Or the one nobody sees, inside an operator:
flux.map(callback -> orderRepository.findById(callback.orderId()))   // Spring Data JPA

// Or in a filter, which affects EVERY endpoint:
@Component
class AuditFilter implements WebFilter {
    public Mono<Void> filter(ServerWebExchange ex, WebFilterChain chain) {
        auditRepository.save(new AuditRecord(ex));      // blocking. On every request.
        return chain.filter(ex);
    }
}
```

**Exact symptom.** Throughput drops to roughly `ioWorkerCount / blocking-duration` requests
per second — a hard ceiling, not a slowdown. **CPU is near idle.** Latency is bimodal:
requests on an unblocked loop are fast; requests on a blocked loop wait for every other
request queued on that loop. Endpoints that do no database work degrade identically,
because they share the loops. As load rises, all loops are blocked and everything stops.
**MVC, running exactly the same code, would be five to ten times faster.**

**Root cause.** There are `ioWorkerCount` loops. A connection is bound to one for its
lifetime. A blocked loop serves none of its connections. This is Topic 100's blocked FJP
worker and Topic 101's pinned carrier, arriving by a third route — and the signature is the
same every time: **collapsed throughput, idle CPU.**

**Fix, in order of preference:**

1. **Use a non-blocking driver.** R2DBC for Postgres, `WebClient` for HTTP. This is the
   real answer and it is the commitment the module boundary above exists to contain.
2. **If you must call blocking code, get it off the loop:**
   ```java
   Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
       .subscribeOn(Schedulers.boundedElastic())
   ```
   And be honest that you have reinvented thread-per-request behind a reactive API.
3. **Prove it in CI with BlockHound**, so it cannot come back.

**Observable — BlockHound, and it is automatable:**

```xml
<dependency>
  <groupId>io.projectreactor.tools</groupId>
  <artifactId>blockhound</artifactId>
  <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
class NoBlockingOnEventLoopTest {

    @BeforeAll
    static void install() {
        BlockHound.builder()
            // Whitelist known-safe blocking, with a comment saying WHY:
            .allowBlockingCallsInside("com.orderflow.ingest.StartupWarmup", "warm")
            .install();
    }

    @Test
    void callback_ingestion_does_not_block_a_loop() {
        webTestClient.post().uri("/internal/payments/callbacks")
            .bodyValue(sampleCallback()).exchange()
            .expectStatus().isAccepted();
    }
}
```

| What you see | What it means |
|---|---|
| `BlockingOperationError` naming a JDBC or socket method | **Found it.** Read the thread name in the stack: `reactor-http-nio-*` is a loop. |
| Test passes | No *instrumented* blocking call ran on a non-blocking thread on this path. Not proof of absence — BlockHound knows JDK blocking primitives, not your CPU-heavy loop. |
| BlockHound will not install | Java agent / module access. **Flagged uncertainty: support for the newest JDKs sometimes lags — verify it installs on your JDK before relying on it in CI.** |
| False positives on your own code | `allowBlockingCallsInside(class, method)`, with a comment recording why it is safe. |

Run BlockHound in tests and staging, not production. Topic 103 Proof 4 has the fuller
setup.

### Trap 2 — `block()` inside a reactive chain

**Wrong approach.**

```java
@GetMapping("/internal/payments/callbacks/{id}")
public Callback byId(@PathVariable String id) {
    return inbox.findById(id).block();       // <-- on the event loop
}
```

**Exact symptom.** In recent Reactor versions, an immediate exception:

```
java.lang.IllegalStateException: block()/blockFirst()/blockLast() are blocking,
which is not supported in thread reactor-http-nio-3
```

*(Illustration of the format, not captured output.)*

On a thread Reactor does not recognise, you get no exception and the far worse outcome: a
silently stalled loop, exactly as in Trap 1.

**Root cause.** `block()` subscribes and parks the calling thread on a latch until the
`Mono` terminates. If the calling thread is the one that would deliver the completion, you
have deadlocked a loop against itself.

**Fix.** Do not unwrap. Return the `Mono` and let the framework subscribe:

```java
@GetMapping("/internal/payments/callbacks/{id}")
public Mono<Callback> byId(@PathVariable String id) {
    return inbox.findById(id);
}
```

**`block()` is legitimate in `main()`, in a test, and on an imperative boundary thread you
own that is not a loop. Nowhere else.**

**Observable:** the exception, BlockHound, and a grep for `.block()` in `orderflow-ingest/src/main`
that must return zero results — enforce it in the build if you like.

### Trap 3 — "WebFlux is faster", benchmarked into a table of noise

**Wrong approach.** Both starters on the classpath, or a benchmark run against localhost
with 10 concurrent users, or a closed-model load generator, or comparing WebFlux+R2DBC
against MVC+Hibernate and attributing the whole delta to the web stack.

**Exact symptom.** A results table that shows WebFlux 3x faster, or 3x slower, and which
nobody can reproduce. Six months later someone reruns it and gets the opposite answer.

**Root cause.** Four independent methodology errors, each sufficient on its own:

1. **The app was never reactive.** Both starters present means a servlet application. This
   is why the startup assertion above exists.
2. **Concurrency below the MVC thread pool size.** At 50 concurrent requests against a
   200-thread pool, MVC's constraint is not active. **WebFlux's advantage only exists above
   the point where thread count binds.** Benchmarking below it measures nothing.
3. **Closed-model load.** A fixed number of VUs looping throttles itself when the service
   slows, so you never observe the ceiling. Topic 65 already told you this and it is the
   error people repeat most.
4. **Two variables at once.** WebFlux+R2DBC versus MVC+Hibernate changes the persistence
   layer as well as the web layer, and the persistence delta is usually the larger one.

**Fix.**

- Assert the stack at startup and fail if it is wrong.
- **Sweep the arrival rate.** 0.5x, 1x, 2x, 4x, 8x of the Topic 65 baseline. The
  interesting result is where the curves diverge, and whether they diverge at all.
- **Open-model arrival rate.** k6 `constant-arrival-rate`, not a VU loop.
- **Change one thing at a time**, and if you cannot — because WebFlux forces R2DBC —
  **say so in the writeup** and attribute the delta honestly to "the whole stack" rather
  than to the web layer.
- Report throughput, p50/p95/p99/p999, error rate, thread count, heap, and CPU. A single
  requests-per-second number is not a result.

**Observable:** the three-way table from the Measurement section, with the arrival rate on
the x-axis rather than a single number.

### Trap 4 — `ThreadLocal` context silently lost

**Wrong approach.** Carrying MDC, `SecurityContextHolder`, or a tenant `ThreadLocal` into a
WebFlux handler, exactly as you would in MVC.

```java
@GetMapping("/internal/payments/callbacks")
public Flux<Callback> list() {
    String tenant = TenantContext.get();          // ThreadLocal. Works... sometimes.
    return inbox.findByTenant(tenant)
                .doOnNext(c -> log.info("found {}", c.id()));   // correlationId MISSING
}
```

**Exact symptom.** Log lines missing the correlation ID — **but only some of them**, and
only the ones after an operator that hopped threads. Traces broken at a `flatMap`
boundary. `SecurityContextHolder.getContext()` returning an empty context intermittently.
Everything works in a unit test with one request and fails in production under
concurrency, which is the worst possible failure schedule.

**Root cause.** A reactive chain hops threads between operators. `ThreadLocal` is bound to
a thread. The value set on `reactor-http-nio-3` is invisible on `parallel-2`. Worse: a
`ThreadLocal` left set on a loop thread is visible to the *next* request that lands on that
loop, which is a cross-request data exposure.

**Fix.** Reactor's `Context`, carried in the subscription rather than on the thread:

```java
return inbox.findByTenant(tenant)
        .contextWrite(ctx -> ctx.put("correlationId", correlationId));

// And read it where you need it:
Mono.deferContextual(ctx -> {
    String correlationId = ctx.get("correlationId");
    ...
});
```

Plus Micrometer's context-propagation library and `ContextSnapshot` to bridge to MDC at the
logging boundary. **Topic 108 is the full treatment and the drill.**

**Observable:** grep your logs for lines missing the correlation ID; count them as a
fraction. Topic 108's drill makes this concrete.

### Trap 5 — `boundedElastic` used as a general-purpose escape hatch

**Wrong approach.**

```java
// Every repository method in the "reactive" service:
public Mono<Order> findById(long id) {
    return Mono.fromCallable(() -> jpaRepository.findById(id).orElseThrow())
               .subscribeOn(Schedulers.boundedElastic());
}
```

**Exact symptom.** No blocking errors, BlockHound passes, and the service performs
**identically to or worse than** the MVC version it replaced. Thread count under load is
similar to MVC's. Stack traces are worse. Debugging is worse. Nobody can explain what the
migration bought.

**Root cause.** You moved thread-per-request onto a different pool. `boundedElastic` is
threads; blocking on them is one thread per blocking operation. **You have MVC with a
reactive API on top, plus a thread hop per call, plus the loss of `ThreadLocal` context.**

**Fix.** Two honest options and no third:

1. **Commit to R2DBC** for this module and get the actual benefit.
2. **Go back to MVC**, ideally on virtual threads (Topic 101), and get the imperative model
   back for free.

The useful diagnostic question, and it is worth asking in a review: **what fraction of
requests on this path touch `boundedElastic`?** If it is most of them, the reactive stack is
decoration. If it is one legacy client, `boundedElastic` is exactly right.

**Observable:** thread count and thread names under load. If `boundedElastic-N` threads
outnumber `reactor-http-nio-N` threads by a wide margin, you have this.

### Trap 6 — losing MVC's accidental admission control

**Wrong approach.** Migrating to WebFlux and keeping the same downstream configuration.

**Exact symptom.** The web tier now happily accepts 20,000 concurrent requests. Every one
of them tries to acquire a connection from a pool of 20, or calls a downstream rate-limited
to 100/second. You get pool-acquisition timeouts, downstream 429s, and a retry storm
(Topic 111) that takes out a service you do not own. **The web tier is healthy throughout,
which makes the diagnosis harder, not easier.**

**Root cause.** MVC's 200-thread pool plus a 100-deep accept queue was admission control
you never designed. It capped in-flight work at 200 and refused connections beyond that,
which protected everything behind it. WebFlux removes that cap deliberately — and you must
**add back on purpose what you were getting by accident.**

**Fix.**

```java
// A bulkhead at the ingress, sized from the downstream's real capacity (Topic 97):
private final Semaphore inFlight = new Semaphore(500);

// Or, reactively, cap the concurrency where the work happens:
callbacks.flatMap(this::process, 8)        // 8 = the R2DBC pool size, not a guess

// Plus a fail-fast connection acquisition timeout, so a saturated pool produces
// a fast 503 rather than a slow pile-up (Topic 109).
```

**Observable:** concurrent in-flight requests, R2DBC pool pending-acquire count, and
downstream 429 rate. Graph all three (Topic 118). The web tier's own metrics will look
fine, which is exactly why you need the others.

---

## Hands-on proof

### Setup

```bash
./mvnw -pl orderflow-ingest dependency:tree | grep -E 'starter-web|starter-webflux|netty|reactor'
```

### Proof 1 — which stack are you actually running?

**Do this before any measurement.** Everything else is invalid if this is wrong.

```java
@EventListener(ApplicationReadyEvent.class)
void reportStack(ApplicationReadyEvent event) {
    log.info("context      = {}", event.getApplicationContext().getClass().getSimpleName());
    log.info("processors   = {}", Runtime.getRuntime().availableProcessors());
    log.info("virtual VT   = {}", virtualThreadsEnabled);
    Thread.getAllStackTraces().keySet().stream()
          .map(Thread::getName)
          .filter(n -> n.startsWith("reactor-http-nio") || n.startsWith("http-nio"))
          .sorted().forEach(n -> log.info("server thread = {}", n));
}
```

| What you see | What it means |
|---|---|
| `ReactiveWebServerApplicationContext` and `reactor-http-nio-N` threads | **WebFlux on Netty.** Count the loop threads — that number is your entire request-serving parallelism. |
| `ServletWebServerApplicationContext` and `http-nio-8080-exec-N` threads | **MVC on Tomcat**, even if your controllers return `Mono`. If you expected WebFlux, a `spring-boot-starter-web` has leaked in. |
| `ReactiveWebServerApplicationContext` but Tomcat thread names | WebFlux on the Tomcat reactive adapter. Legal, unusual, and worth knowing before you interpret anything. |
| Loop count equals `availableProcessors()`, or 4 on a small container | The documented default is `max(4, availableProcessors())` — **and counting beats quoting.** |

### Proof 2 — count the threads under load

```bash
# Under sustained load, sample the thread picture:
jcmd <pid> Thread.print | grep -oE '"(reactor-http-nio|http-nio-8080-exec|boundedElastic|parallel)-[0-9]+"' \
  | sort | uniq -c | sort -rn

# And the totals:
jcmd <pid> Thread.print | grep -c '^"'
```

| What you see | What it means |
|---|---|
| MVC: `http-nio-8080-exec-N` count rising to `threads.max` and stopping | The pool is the ceiling. Requests above it are in the accept queue. |
| WebFlux: a small, constant `reactor-http-nio-N` count regardless of load | **The thread economy, observed.** This is the number that justifies the model. |
| WebFlux: a growing `boundedElastic-N` count | Blocking work is being offloaded. **How much?** If it dominates, see Trap 5. |
| WebFlux: total thread count comparable to MVC's | You have Trap 5. The reactive stack is a wrapper over a thread pool. |
| MVC on virtual threads: few platform threads, and `jcmd Thread.print` looks nearly empty | Expected — that tool shows carriers now (Topic 101). Use `jcmd Thread.dump_to_file -format=json`. |

### Proof 3 — the blocking arithmetic, verified

Run Example 1b's three endpoints under identical load and fill in this table with your own
numbers:

| Endpoint | Loops or threads | Blocking duration | **Predicted** ceiling | **Measured** rps |
|---|---|---|---|---|
| `/probe/flux` (non-blocking) | | 0 | not thread-bound | |
| `/probe/flux-blocking` | `ioWorkerCount` | 50 ms | `loops / 0.050` | |
| `/probe/mvc` | `threads.max` | 50 ms | `threads / 0.050` | |

**What to look for:** whether the measured numbers match the predictions.

| What you see | What it means |
|---|---|
| The blocking WebFlux number is close to `loops / 0.050` | **The model is confirmed.** You can now predict this failure's magnitude before it happens, which is what makes the review comment worth making. |
| The blocking WebFlux number is much higher than predicted | Some requests are being served by another scheduler, or keep-alive is distributing connections unevenly across loops. Investigate — the mechanism is more interesting than the number. |
| MVC is far below `threads / 0.050` | Something else is the constraint — the connection pool, the database, or the load generator itself. **Check the generator is not saturated** (Topic 65). |

### Proof 4 — BlockHound in CI

See Trap 1 for the full setup and the reading table. The point to internalise: **make it a
test, not a habit.** A discipline that depends on every reviewer remembering will fail; a
failing build will not.

### Proof 5 — where does a request actually run?

Print the thread at three points in a WebFlux chain:

```java
@GetMapping("/probe/threads")
public Mono<String> probe() {
    log.info("[1 handler]  {}", Thread.currentThread().getName());
    return inbox.count()
        .doOnNext(n -> log.info("[2 after-db] {}", Thread.currentThread().getName()))
        .publishOn(Schedulers.parallel())
        .map(n -> {
            log.info("[3 after-publishOn] {}", Thread.currentThread().getName());
            return "count=" + n;
        });
}
```

| What you see | What it means |
|---|---|
| [1] and [2] on the same `reactor-http-nio-N` | The R2DBC driver completed on the loop that owns the connection. Normal. |
| [3] on `parallel-N` | `publishOn` switched schedulers. **Every one of these hops is where a `ThreadLocal` dies** (Trap 4, Topic 108). |
| [2] on `boundedElastic-N` | Something on this path is offloading blocking work. Find out what. |
| All three on the same thread | Fusion (Topic 104) collapsed the hops, or you have no scheduler switch. Fine — but do not build a `ThreadLocal` on the assumption that it stays true. |

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the failure yourself and
written down what you saw.

### The scenario

Put one blocking JDBC call in the WebFlux ingestion handler, run the Topic 65 baseline
against it, and watch throughput collapse to a hard ceiling with CPU at idle. **Then run
the identical blocking code on MVC and observe that MVC is dramatically faster.** That
comparison is the point: the same mistake costs a few percent in one model and everything
in the other.

### Setup

Add a deliberately blocking variant to `orderflow-ingest`. You will need
`spring-boot-starter-jdbc` on the test classpath only — your enforcer rule bans it from the
main build, which is exactly right, so add it in a `drill` Maven profile and remove it
afterwards.

```java
package com.orderflow.ingest.drill;

@RestController
@Profile("drill")
public class BlockingCallbackController {

    private final JdbcTemplate jdbcTemplate;      // BLOCKING. Deliberately.
    private final PaymentCallbackInbox reactiveInbox;

    /** The defect: blocking JDBC directly on an event-loop thread. */
    @PostMapping("/drill/callbacks/blocking")
    public Mono<ResponseEntity<Void>> blocking(@RequestBody PaymentCallback c) {
        log.debug("handler thread = {}", Thread.currentThread().getName());
        jdbcTemplate.update(
            "INSERT INTO payment_callback_inbox (provider, callback_id, order_id, "
          + "payload, received_at, status) VALUES (?,?,?,?,now(),'PENDING') "
          + "ON CONFLICT DO NOTHING",
            c.provider(), c.callbackId(), c.orderId(), c.rawJson());
        return Mono.just(ResponseEntity.accepted().build());
    }

    /** Fix 1: same blocking call, offloaded. */
    @PostMapping("/drill/callbacks/offloaded")
    public Mono<ResponseEntity<Void>> offloaded(@RequestBody PaymentCallback c) {
        return Mono.fromRunnable(() -> jdbcTemplate.update(/* same SQL */))
                   .subscribeOn(Schedulers.boundedElastic())
                   .thenReturn(ResponseEntity.accepted().<Void>build());
    }

    /** Fix 2: the real answer. Non-blocking driver. */
    @PostMapping("/drill/callbacks/r2dbc")
    public Mono<ResponseEntity<Void>> reactive(@RequestBody PaymentCallback c) {
        return reactiveInbox.insertIfAbsent(c)
                   .thenReturn(ResponseEntity.accepted().<Void>build());
    }
}
```

And the MVC control, in `orderflow-core`:

```java
@RestController
@Profile("drill")
public class MvcCallbackController {

    /** The SAME blocking code, on the servlet stack. This is the control. */
    @PostMapping("/drill/callbacks/mvc")
    public ResponseEntity<Void> mvc(@RequestBody PaymentCallback c) {
        jdbcTemplate.update(/* same SQL */);
        return ResponseEntity.accepted().build();
    }
}
```

### Commands

```bash
# Record the environment first. All of it goes in the writeup.
java --version
docker inspect orderflow-ingest | grep -i -E 'cpu|memory'
curl -s localhost:8081/actuator/configprops | jq '.. | .ioWorkerCount? // empty'

# Four runs, identical open-model arrival rate, five minutes each.
k6 run -e TARGET=blocking  -e RATE=800 k6/callback-drill.js | tee run-blocking.log
k6 run -e TARGET=offloaded -e RATE=800 k6/callback-drill.js | tee run-offloaded.log
k6 run -e TARGET=r2dbc     -e RATE=800 k6/callback-drill.js | tee run-r2dbc.log
k6 run -e TARGET=mvc       -e RATE=800 k6/callback-drill.js | tee run-mvc.log

# During the blocking run, in another terminal:
jcmd <pid> Thread.print | grep -A8 'reactor-http-nio'
top -H -p <pid>                      # Linux; Activity Monitor on macOS
```

And the automated confirmation, which should have caught it before you deployed:

```bash
./mvnw -pl orderflow-ingest test -Dtest=NoBlockingOnEventLoopTest
```

### What to capture, before reading on

For each of the four runs:

1. Achieved throughput (requests/second actually completed, not offered).
2. p50 / p95 / p99 / p999.
3. Error rate and error *shape* (timeouts? refused? 503?).
4. **CPU utilisation.**
5. Thread counts by name prefix.
6. From the thread dump during the blocking run: how many `reactor-http-nio-N` threads,
   and what frame each is in.
7. `ioWorkerCount`, and the predicted ceiling `ioWorkerCount / blocking-duration`.

**Now write down your prediction for which of the four runs is fastest, and by how much.**
Most people get the ordering right and the magnitude badly wrong.

### How to read it

| What you see | What it means |
|---|---|
| **Blocking:** throughput pinned near `ioWorkerCount / blocking-duration`, far below the offered rate | **The drill has fired.** This is a hard ceiling set by loop count, not a gradual slowdown. Compute the prediction and check it matches. |
| **Blocking:** CPU near idle while throughput has collapsed | **The signature.** Third occurrence in this curriculum after Topics 100 and 101. Collapsed throughput plus idle CPU means threads parked where they cannot be replaced. You should recognise this instantly by now. |
| **Blocking:** thread dump shows every `reactor-http-nio-N` inside a JDBC driver frame | Confirmation, and the fastest possible diagnosis in a real incident. |
| **Blocking:** latency bimodal — some requests fast, most very slow | Connections are bound to loops. Land on a free loop and you are fast; land on a blocked one and you wait behind everything queued there. |
| **MVC:** throughput several times higher than the blocking WebFlux run, at the same code | **The lesson.** The same defect costs a few percent in MVC and an order of magnitude in WebFlux. Write the ratio down; it is the most persuasive number in the whole topic. |
| **Offloaded:** throughput recovers substantially; `boundedElastic-N` thread count grows | The escape hatch working — **and you have rebuilt thread-per-request on a second pool.** Compare its thread count with MVC's. They will be similar, which is the point. |
| **R2DBC:** highest throughput, lowest thread count, flat CPU headroom | The model working as designed. This is the number that justifies the module. |
| **R2DBC not fastest** | The database or the R2DBC pool is now the constraint, not threads. **A completely legitimate finding**, and it means the whole WebFlux argument does not apply to this workload. Say so. |
| BlockHound test passes but the drill still fires | The blocking call is not one BlockHound instruments — a native call, a busy CPU loop, or a whitelisted method. Read your whitelist. |

### Part B — the same drill, with virtual threads as a fourth column

Run the MVC control a second time with `spring.threads.virtual.enabled=true` (Topic 101).

**What to look for:** whether MVC-on-virtual-threads beats WebFlux-with-R2DBC at high
arrival rates.

**How to read it:**

| Result | What it means for Topic 107 |
|---|---|
| MVC-virtual close to WebFlux-R2DBC | **The Loom argument, made empirically.** You get comparable thread economy with the imperative model, Hibernate, whole stack traces and working `ThreadLocal`s. That is a strong case for not adopting WebFlux at all. |
| WebFlux-R2DBC clearly ahead at very high arrival rates | The remaining gap is real. Quantify it, and ask whether the arrival rates where it appears are ones you will ever see. |
| Both bounded by the database | **The most likely outcome, and the most useful finding.** Neither web stack is the constraint. Go to Topic 109 and stop arguing about the web layer. |

**This four-way table is the deliverable Topic 107 consumes. Do not skip Part B.**

### What the drill proves

1. **The same mistake has wildly different costs in the two models.** MVC degrades a few
   percent; WebFlux collapses by an order of magnitude. That asymmetry *is* the discipline
   requirement, expressed as a number instead of as advice.
2. **The ceiling is predictable.** `ioWorkerCount / blocking-duration` is arithmetic you
   can do in a code review, before the code ships, which is what makes the review comment
   worth making.
3. **The escape hatch works and costs you the reason you came.** `boundedElastic` recovers
   throughput by rebuilding a thread pool. If most of your requests use it, you have MVC
   with worse ergonomics.
4. **The comparison must include virtual threads or it is out of date.** Any WebFlux-vs-MVC
   benchmark that predates Java 21 is answering a question nobody is asking any more.

---

## Measurement

### The standing rule

A naive `System.nanoTime()` loop is the **wrong** way to measure JVM performance. It
measures JIT warm-up, dead-code elimination, on-stack replacement and ambient noise.
**Topic 77 (JMH)** is where you learn to do it properly.

For this topic there is a sharper version: **a microbenchmark cannot answer this question at
all.** The difference between the two models only exists at concurrency levels above where
the thread pool binds, over sustained periods, with a real network and a real database. The
instrument is the Topic 65 harness. JMH is the wrong tool here and using it is itself an
error.

### The instrument for each claim

| Claim | Instrument | What invalidates it |
|---|---|---|
| "We are running WebFlux" | Application context class + `reactor-http-nio` thread names at startup | Both starters on the classpath |
| "We have N event loops" | Count `reactor-http-nio-*` threads in a dump | Quoting `max(4, cores)` instead of counting |
| "Nothing blocks a loop" | BlockHound in CI, plus thread-name logging in operators | Assuming; BlockHound not installing on your JDK |
| "WebFlux uses fewer threads" | Thread count under sustained load, both stacks | Measuring at idle |
| "WebFlux handles more load" | Throughput and p99 **swept across arrival rates** | A single arrival rate; closed-model load; concurrency below `threads.max` |
| "The migration is worth it" | The three- or four-way table, plus the R2DBC rewrite cost in engineer-weeks | Only measuring the happy path |
| "Ingestion doesn't hurt the core" | Catalogue p99 during a callback burst | Only measuring the endpoint under test |
| "The web tier isn't the constraint" | R2DBC/Hikari pool pending-acquire counts (Topic 109) | Assuming the web tier because that is what you changed |

### The comparison table — the deliverable

Four configurations, identical everything else. This is what Topic 107 consumes.

| Config | Web stack | Threads | Persistence |
|---|---|---|---|
| **A** | MVC / Tomcat | Platform, `threads.max=200` | Hibernate + JDBC |
| **B** | MVC / Tomcat | **Virtual** (Topic 101) | Hibernate + JDBC |
| **C** | WebFlux / Netty | Event loops | **R2DBC** |
| **D** | WebFlux / Netty | Event loops | JDBC on `boundedElastic` |

For each config, at arrival rates 0.5x, 1x, 2x, 4x, 8x of the Topic 65 baseline:

| Metric | Why it is in the table |
|---|---|
| Throughput achieved vs offered | The gap is where the ceiling is |
| p50 / p95 / p99 / p999 | p50 rarely moves; the tail is the story |
| Error rate **and shape** | Refused vs timeout vs 503 tells you where the queue is |
| Peak thread count | The thread-economy claim, measured |
| Peak heap used | WebFlux's cost: in-flight state on the heap |
| CPU utilisation | Idle CPU plus low throughput means parked threads |
| DB pool pending-acquire | Usually the real constraint (Topic 109) |
| Catalogue p99 (unaffected endpoint) | Blast radius |

**Rules that make it comparable:**

1. Same dataset — 100k products, 1M orders, 5M order lines, same seed.
2. Same container CPU and memory limits. Record them.
3. Same JVM flags and collector. Record them.
4. **Open-model arrival rate.** Not a VU loop. This is the error that invalidates most
   published comparisons.
5. Same warm-up discard window.
6. Three repetitions; report the spread, not one number.
7. **Verify the stack at startup in every run** and paste the log line into the writeup.

**How to read it:**

- **All four curves identical up to 2x** — thread count is not your constraint at
  realistic load. The most likely outcome for `orderflow`, and a completely legitimate
  finding. Report it plainly.
- **A diverges first, then B holds, C holds** — thread count binds somewhere between,
  and virtual threads solved it without a rewrite. **That is Topic 107's answer, delivered
  by the data.**
- **C ahead of B only above 4x** — quantify the gap and then ask the only question that
  matters: will you ever run at 4x?
- **D indistinguishable from A** — confirms Trap 5. `boundedElastic` is thread-per-request
  wearing a costume.
- **Everything bounded by pool pending-acquire** — the web layer is irrelevant here. Close
  the comparison and go to Topic 109.

### What to graph permanently

```java
// Both stacks:
registry.gauge("orderflow.threads.total", threadMXBean, ThreadMXBean::getThreadCount);
registry.gauge("orderflow.http.inflight", inFlightCounter, AtomicInteger::get);

// WebFlux specifically:
registry.gauge("orderflow.netty.loops", loopCount);
registry.gauge("orderflow.scheduler.boundedElastic.active", boundedElasticActive);

// MVC specifically -- exposed by Boot's Tomcat metrics:
// tomcat.threads.busy, tomcat.threads.current, tomcat.threads.config.max
```

**The alert worth having on WebFlux:** `boundedElastic` active thread count above a small
threshold on a path that should be fully reactive. That fires on Trap 5 creeping back in,
which it will, one PR at a time.

---

## Practice exercises

### 1 — easy: establish the thread facts

Produce a one-page table for **your** environment, every row traceable to a command:

1. `availableProcessors()` on your laptop, at `--cpus=2`, and at `--cpus=0.5`.
2. The event-loop count in each, by counting `reactor-http-nio-*` threads — not by quoting
   a formula.
3. `server.tomcat.threads.max` and `accept-count` for `orderflow-core`, from
   `/actuator/configprops`.
4. The application context class for both modules, from the startup log.
5. For each of `/probe/mvc` and `/probe/flux`, the thread name before and after the delay.
6. The predicted blocking ceiling `ioWorkerCount / 0.050` and the measured one.

**Rows you inferred rather than measured do not count.**

### 2 — medium: the audit (combines Topics 01–105)

Audit `orderflow-ingest` for reactive-stack correctness. For each finding: file, line, the
topic it comes from, the symptom under load, the fix, and the risk of the fix.

Find at least:

1. Every blocking call reachable from a handler (Trap 1). **Prove it with BlockHound**, not
   by reading — the interesting ones are three layers down in a shared library.
2. Every `.block()` in `src/main` (Trap 2).
3. Every `subscribeOn(boundedElastic)` and what fraction of requests hit it (Trap 5).
4. Every `ThreadLocal`, MDC use, or `SecurityContextHolder` reference (Trap 4, Topic 108).
5. Every unbounded buffer or `Sinks` without a size (Topic 105).
6. Every `flatMap` with a default concurrency, and the number it should have (Topic 105,
   Topic 109).
7. Anything that would now be admitted without limit that MVC's pool used to cap (Trap 6).
8. The dependency tree: is `spring-boot-starter-web` present transitively (Topic 32)?

**Deliverable:** the audit table, plus the enforcer rule and the startup assertion actually
committed, plus one BlockHound test that fails on the current code before you fix it.

### 3 — hard: the four-way benchmark

**This is the spine deliverable and it feeds Topic 107 directly.**

Produce a decision-quality comparison of `orderflow` across configurations A, B, C and D
from the Measurement section.

**Method:**

1. Reproduce the Topic 65 baseline within ±10%. If you cannot, stop.
2. Build config C properly: `orderflow-ingest` as a separate deployable, WebFlux + R2DBC,
   enforcer rule, startup assertion. **Verify the stack in the startup log of every run and
   paste the line into the writeup.**
3. Record the full environment for every run: JDK version, container limits, JVM flags,
   collector, pool sizes, dataset seed, k6 script hash.
4. Sweep arrival rates 0.5x, 1x, 2x, 4x, 8x. Three repetitions. Open model.
5. Capture every metric in the Measurement table, including the **unaffected** catalogue
   endpoint.
6. Run BlockHound against C and D and include the result. **A config with a blocking call
   on a loop is not a valid data point** — fix it and rerun.
7. Then measure the thing nobody measures: **the engineering cost.** How many hours did the
   R2DBC ingestion path take compared with the JDBC one? How many lines of SQL replaced
   Spring Data derived queries? How long did the first production-shaped bug take to
   debug in each? These are estimates and you should label them as estimates — and they are
   frequently the deciding input.

**Deliverable — at most three pages:**

- The four-way table, filled in.
- Two graphs: throughput vs arrival rate, and p99 vs arrival rate. Four curves each.
- **A statement of what the constraint actually is**, defended by the data.
- The engineering-cost estimate, labelled as an estimate.
- An explicit "what we did not measure" section.
- A recommendation, with a named risk and a rollback trigger.

**The grading criterion is whether Topic 107's recommendation could be written from this
document alone, by someone who did not run the benchmark.** That is the standard, and it is
also the Phase 12 standard.

---

## Interview questions

### Q1 — "Is WebFlux faster than Spring MVC?"

**MID-LEVEL.** "Yes, it's non-blocking, so it handles more requests with fewer threads and
scales better under load."

**SENIOR.** "At the same throughput it isn't faster — it uses fewer threads. Those are
different claims and the second one only becomes a performance win when thread count is the
binding constraint.

MVC caps concurrency at the worker pool, typically 200 in Boot. Little's Law says that is
`200 / average-request-time` requests per second: 4,000 at 50 ms, 1,000 at 200 ms. Below
that ceiling, MVC is not constrained by threads and WebFlux has nothing to improve.

WebFlux's win shows up above it — tens of thousands of concurrent, mostly-idle connections,
where the thread stacks and context switches would dominate. Long-lived SSE or WebSocket
connections are the clearest case.

And it's all-or-nothing. One blocking call on an event loop and throughput drops to roughly
`ioWorkerCount / blocking-duration` — with eight loops and a 20 ms query, 400 per second,
against MVC's 4,000. **The same mistake makes WebFlux ten times worse than the framework it
replaced.**

In Java specifically it also means R2DBC instead of Hibernate, which is a much larger
commitment than the web layer itself — no persistence context, no lazy loading, no Spring
Data JPA. And since Java 21, virtual threads give you most of the thread economy without
any of that, so I'd benchmark MVC-on-virtual-threads as the third option before adopting
WebFlux at all."

**What separates them.** Separating "faster" from "fewer threads"; doing Little's Law out
loud; naming the blocking ceiling as arithmetic; naming R2DBC as the real cost; and bringing
virtual threads in unprompted.

**Follow-up:** *"Where would you use WebFlux today?"* → An API gateway, a fan-out
aggregator, a long-lived SSE or WebSocket endpoint, or anything with a genuine streaming
source needing backpressure (Topic 105). Not a CRUD service over a relational database.

### Q2 — "Someone put a JDBC call in a WebFlux handler. Walk me through what happens."

**MID-LEVEL.** "It blocks the event loop, which is bad for performance. You should use
`subscribeOn(boundedElastic)` or R2DBC."

**SENIOR.** "Netty has a small number of event loops — `max(4, availableProcessors())` by
default, so eight on an eight-core box — and each connection is bound to one loop for the
connection's lifetime, not per request.

A JDBC call parks that loop thread in the kernel. While it's parked it serves none of its
connections, including ones hitting endpoints that touch no database at all. Throughput
becomes `loops / blocking-duration`: eight loops, a 20 ms query, 400 per second, hard
ceiling. Latency goes bimodal — requests landing on a free loop are fast, the rest queue
behind everything on their loop.

The signature on a dashboard is collapsed throughput with **idle CPU**. That rules out
contention and GC immediately, and it's the same signature as a blocked ForkJoinPool worker
or a pinned virtual-thread carrier — threads parked where they can't be replaced.

To confirm: thread dump, and every `reactor-http-nio-N` is in a JDBC driver frame. Thirty
seconds.

To prevent it recurring: BlockHound in CI. It instruments the JDK and throws on a blocking
call from a non-blocking thread, so it becomes a failing test rather than a review habit.

The fix is R2DBC. `subscribeOn(boundedElastic)` works and rebuilds thread-per-request on a
second pool — fine for one legacy client, and if most requests go through it you've
recreated MVC with worse stack traces and should just use MVC."

**What separates them.** Doing the arithmetic; knowing connections bind to loops rather
than requests; naming the CPU-idle signature and connecting it to Topics 100 and 101;
proposing BlockHound as automation; and being honest about `boundedElastic`.

**Follow-up:** *"How would you find this in a codebase you've just joined?"* → BlockHound
against the integration test suite. It finds calls three layers down in shared libraries
that no amount of reading would surface, and it takes an afternoon to wire up.

### Q3 — "We're on MVC and hitting the thread pool ceiling. WebFlux or virtual threads?"

**MID-LEVEL.** "Virtual threads are simpler, so I'd start there. WebFlux is a bigger
rewrite."

**SENIOR.** "Virtual threads first, and I'd want to establish something before either.

First, is the thread pool actually the ceiling? A pool of 200 saturating usually means
requests are slow, and requests are usually slow because of the database or a downstream —
not because of threads. If it's a 20-connection Hikari pool, both options just move the
queue. So step one is a thread dump under load: what are the 200 threads actually doing? If
they're all in `getConnection`, neither web stack helps and the answer is Topic 109.

If thread count really is the ceiling: virtual threads are one property, `spring.threads.
virtual.enabled=true`, no rewrite, and you keep Hibernate, whole stack traces, working
debuggers and `ThreadLocal` context. The audit that goes with it is real work — `synchronized`
around blocking calls can pin, `ThreadLocal` cost multiplies by thread count — but it's an
audit, not a rewrite.

WebFlux is a rewrite of the persistence layer as well as the web layer, and you get
backpressure and stream composition, which virtual threads genuinely do not provide.

So my rule is: high-concurrency request/response, virtual threads. A genuine streaming path
with a producer that can outrun the consumer and cannot be told to slow down, reactive —
and I'd scope that to the specific module rather than the whole service.

And I'd measure all three at swept arrival rates before committing, because the most likely
outcome is that they're all bounded by the database and the whole argument is moot."

**What separates them.** Refusing the premise until it's established; the thread dump as
step one; scoping reactive to a module; and predicting that the answer is probably "neither".

**Follow-up:** *"What if the team already has a WebFlux service in production?"* → Then
migration cost is usually decisive and I'd leave it. A working reactive service is not a
bug. I'd want a measured reason, not an aesthetic one.

### Q4 — "How do you keep a WebFlux module and an MVC module in the same codebase?"

**MID-LEVEL.** "Add both starters and use `Mono` in the reactive controllers."

**SENIOR.** "That specific thing doesn't work, and it fails silently, which is what makes
it dangerous. If both starters are on the classpath, Spring Boot builds a **servlet**
application. Your WebFlux controllers still run — on Tomcat's thread pool, thread-per-request,
with no thread economy at all. You believe you're benchmarking WebFlux and you're
benchmarking MVC with extra allocation.

So: separate deployables. Separate Maven modules, separate images, separate pods. Which also
gives me the isolation I wanted anyway — an ingestion burst that fills a heap fills *that*
heap, and the catalogue endpoint is in a different process.

Two things I'd enforce mechanically, because the failure is silent. A Maven enforcer rule
banning `spring-boot-starter-web` from the reactive module — Maven's nearest-wins flattening
means a transitive dependency can leak it in with no error. And a startup assertion that
checks the application context class and **fails the boot** if it isn't reactive.

The honest cost is that you now run two services: two dashboards, two runbooks, a shared
database. That's a real operational tax and it's the main argument for just using virtual
threads on one MVC service instead. I'd pay it here specifically because being able to
benchmark them independently is the point."

**What separates them.** Knowing the silent-failure behaviour; proposing build-time and
startup-time enforcement rather than a convention; and volunteering the operational cost.

**Follow-up:** *"What if you can't run two services?"* → Then MVC on virtual threads for
everything, with a durable inbox for the callback path to get backpressure. You give up the
WebFlux comparison; you keep one deployable and one mental model, which is often worth
more.

### Q5 — "What do you lose by moving to WebFlux?"

**MID-LEVEL.** "It's harder to debug and you have to learn reactive programming."

**SENIOR.** "Five things, roughly in order of how much they hurt.

**Hibernate.** JDBC is blocking by specification, so a real WebFlux service needs R2DBC.
That means no persistence context, no dirty checking, no lazy loading, no Spring Data JPA
derived queries. You write SQL. Everything the team learned about N+1 fixes, batching and
optimistic locking is Hibernate knowledge and stops applying on that path.

**Stack traces.** A reactive trace shows the subscription stack, not the assembly stack —
where it ran, not where you wrote it. You get it back with `checkpoint()` at chosen points
or `Hooks.onOperatorDebug()` everywhere at real cost. That's Topic 108's material and it's a
permanent tax on every incident.

**`ThreadLocal` context.** MDC, `SecurityContextHolder`, tenant context — all gone, because
the chain hops threads. You use Reactor's `Context` and Micrometer's context propagation
instead, and the failure mode is a correlation ID that goes missing for *some* log lines
under concurrency, which is a horrible thing to debug.

**Blocking safety.** MVC forgives a blocking call; WebFlux does not. That has to become a
CI check, not a code-review habit.

**The team.** Everyone must understand cold publishers, subscription time, and demand. The
bugs are subtle — a `Mono` built and never subscribed is a silent no-op, not an error.

What you get for it is thread economy and real backpressure. Whether that's a good trade
depends entirely on whether either is currently a constraint — and since Java 21, virtual
threads give you the thread economy without any of the five costs, which means the honest
justification is now backpressure and streaming, not throughput."

**What separates them.** Naming the persistence commitment first rather than the
programming model; being specific about the observability tax; and landing on "the
justification narrowed to backpressure", which is exactly Topic 107's conclusion.

**Follow-up:** *"So is WebFlux dead?"* → No — it's narrower. Gateways, aggregators,
long-lived streaming connections, and anything needing demand propagation. What died is the
generic "use it for throughput" argument, and that was always weaker than it sounded.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. MVC caps concurrency at the thread pool; WebFlux does not cap it at all. Argue that
   MVC's cap is a **feature**. Then argue it is a bug. What would you have to know about a
   specific service to decide?

2. A blocking call costs MVC a few percent and WebFlux an order of magnitude. Derive both
   numbers from first principles, then construct a workload where the ratio is reversed.

3. A connection is bound to one event loop for its lifetime, not per request. Name two
   consequences of that which do not follow from "one loop per core", and say how you would
   observe each.

4. `boundedElastic` makes blocking safe in WebFlux. Explain, mechanically, why using it for
   everything gives you MVC with extra steps. Then construct the case where it is exactly
   the right tool.

5. WebFlux removes MVC's accidental admission control. Design the minimum you would add
   back for `orderflow-ingest`, and justify each piece from a specific downstream limit
   rather than from a general principle.

6. Virtual threads give MVC most of WebFlux's thread economy. Name the two capabilities that
   remain exclusively WebFlux's, and construct a service where each one is decisive.

7. You are told a benchmark shows WebFlux 3x faster than MVC. List, in priority order, the
   five things you would check about the methodology before believing it. For each, say what
   result would make you discard the benchmark entirely.

---

## Quick reference card

### The mechanism in five lines

```
MVC:     one thread per request, for the request's whole life. Cap = threads.max.
         Little's Law: ceiling = threads.max / avg_request_seconds.
WebFlux: N event loops (max(4, cores)); a CONNECTION is bound to one for its life.
         A request occupies a loop only while executing.
Blocking on a loop: ceiling = ioWorkerCount / blocking_seconds. Order-of-magnitude worse.
```

### The stacks

```
spring-boot-starter-web      -> Tomcat, DispatcherServlet, thread-per-request, JDBC
spring-boot-starter-webflux  -> Netty,  DispatcherHandler,  event loops,        R2DBC

BOTH on the classpath        -> SERVLET application. Silently. Verify at startup.
spring.main.web-application-type=reactive   # forces it -- then still verify
```

### Verify, always

```java
context.getClass().getSimpleName()
// ReactiveWebServerApplicationContext -> WebFlux
// ServletWebServerApplicationContext  -> MVC

Thread.getAllStackTraces().keySet().stream()
      .map(Thread::getName).filter(n -> n.startsWith("reactor-http-nio")).count();
```

### Thread-name prefixes worth recognising instantly

```
http-nio-8080-exec-N   Tomcat worker         -- MVC request thread
reactor-http-nio-N     Netty event loop      -- NEVER BLOCK THIS
parallel-N             Schedulers.parallel() -- CPU work, never blocking
boundedElastic-N       Schedulers.bounded... -- the sanctioned blocking offload
ForkJoinPool.commonPool-worker-N   parallel streams / default CF async (T100)
```

### Defaults — verify, do not quote

```
server.tomcat.threads.max        200      (Boot default; print /actuator/configprops)
server.tomcat.accept-count       100
reactor.netty.ioWorkerCount      max(4, availableProcessors())   -- COUNT the threads
```

### The blocking arithmetic

```
MVC ceiling      = threads.max      / request_seconds
WebFlux ceiling  = ioWorkerCount    / blocking_seconds     <-- if you block a loop

8 loops, 20 ms block   ->   400 rps      200 threads, 20 ms   -> 10,000 rps
8 loops, 200 ms block  ->    40 rps      200 threads, 200 ms  ->  1,000 rps
```

### Diagnostics

```bash
jcmd <pid> Thread.print | grep -A8 'reactor-http-nio'      # what are the loops doing?
jcmd <pid> Thread.print | grep -oE '"[a-z-]+-[0-9]+"' | sort | uniq -c | sort -rn
jcmd <pid> Thread.dump_to_file -format=json /tmp/t.json    # if on virtual threads
curl -s localhost:8080/actuator/configprops | jq
# CPU idle + throughput collapsed = threads parked where they cannot be replaced
```

```java
BlockHound.install();                              // in tests. Make it a failing build.
log.info("thread={}", Thread.currentThread().getName());   // in operators
```

### When each wins

```
MVC (+ virtual threads)   request/response, relational DB, Hibernate, transactions,
                          a team that must debug it at 3am. THE DEFAULT.
WebFlux                   API gateway, fan-out aggregator, long-lived SSE/WebSocket,
                          genuine streaming with a producer you cannot slow down (T105).
Neither                   when the database is the constraint. Go to Topic 109.
```

### Gotchas checklist

- [ ] Verify the application context type at startup. Fail the boot if it is wrong.
- [ ] Enforcer rule banning `spring-boot-starter-web` from the reactive module.
- [ ] BlockHound as a CI test, not a code-review habit.
- [ ] No `.block()` in reactive `src/main`. Enforce it.
- [ ] Count event-loop threads; never quote the formula.
- [ ] Add back the admission control MVC was giving you by accident.
- [ ] No `ThreadLocal`, MDC or `SecurityContextHolder` on a reactive path (T108).
- [ ] Measure what fraction of requests touch `boundedElastic`.
- [ ] Benchmark with an open-model arrival rate, swept, above the MVC thread ceiling.
- [ ] Always include MVC-on-virtual-threads as a column. Otherwise the table is out of date.

---

## When would I use this at work?

**1. Costing a proposed "let's move to WebFlux" migration.**

The question is never "is it faster". It is: what is currently the constraint, does it
bind at our real arrival rates, and what does R2DBC cost us in engineer-weeks and in lost
Hibernate capability? Producing that as a table — with virtual threads as a column, because
they usually win it — turns an architectural argument into a decision. That is the single
highest-value use of this topic.

**2. Reviewing a PR that touches a reactive handler.**

Two mechanical questions. Does anything on this path block, three layers down included?
And what fraction of this path now runs on `boundedElastic`? The first prevents an outage;
the second detects the slow slide back into thread-per-request that every reactive codebase
experiences one PR at a time.

**3. Diagnosing a reactive service with collapsed throughput and idle CPU.**

Thread dump, grep for `reactor-http-nio`, read the frames. Ninety seconds to either find a
blocking driver call or rule the whole class out. You have now met this signature three
times — blocked FJP workers, pinned carriers, blocked loops — and recognising it instantly
is one of the most transferable diagnostic skills in the curriculum.

---

## Connected topics

**Prerequisites:**

- **82 — JVM tuning in containers:** `availableProcessors()` under a cgroup quota sets your
  loop count and your Tomcat sizing. Both stacks get this wrong in a container nobody
  checked.
- **90 — Executors and pool sizing:** Little's Law, and why MVC's bounded pool plus accept
  queue is admission control you did not design.
- **93 — Bounded queues:** the accept queue is backpressure at the door, and WebFlux
  removes it.
- **101 — Virtual threads:** **the third and usually winning column of every comparison in
  this document.** Read it before drawing any conclusion here.
- **103 — NIO and Netty's event loop:** the machinery underneath WebFlux. This document is
  Topic 103 wearing a Spring shirt; that one has the epoll and `ByteBuf` detail.
- **104 — `Mono`/`Flux`:** the programming model you are committing to.
- **105 — Backpressure:** the capability that actually justifies WebFlux, and the reason
  the ingestion module exists at all.

**Also relevant:**

- **32 — Dependency resolution:** why a transitive `spring-boot-starter-web` silently turns
  your reactive module into a servlet app, and why the enforcer rule is not paranoia.
- **44 — `@RestController` and `DispatcherServlet`:** the MVC request path in detail.
- **48–53 — Hibernate:** everything you give up on an R2DBC path. Six topics of knowledge
  that stops applying.
- **56 — Spring Security:** `SecurityContextHolder` is `ThreadLocal`-backed; the reactive
  stack needs a different mechanism entirely.
- **65 — The load baseline:** the harness for every measurement here.
- **97 — Coordination primitives:** the `Semaphore` bulkhead that replaces MVC's accidental
  cap.
- **109 — HikariCP:** the constraint that is usually the real answer, whichever web stack
  you pick.

**This unlocks:**

- **107 — The Loom-vs-reactive decision:** **this document's benchmark table is a direct
  input.** Do not skip the hard exercise; Topic 107's deliverable is not writable without
  it.
- **108 — Reactive debugging:** the observability tax quantified — subscription stacks,
  `checkpoint()`, and the lost MDC that Trap 4 previews.
- **113–114 — Kafka:** the other push source, and where the ingestion module's durable
  buffer might live instead of Postgres.
- **118 — Metrics:** the thread and loop gauges from the Measurement section.
- **121 — Actuator and k8s probes:** two deployables means two readiness stories, and the
  ingestion module's readiness must reflect the inbox's health, not just the port.

---

*Java baseline 21. Netty, Reactor and Tomcat versions come from the Spring Boot BOM; do not
pin them by hand and do not trust a version number quoted in any document, including this
one. The defaults cited here — `server.tomcat.threads.max`, `accept-count`,
`reactor.netty.ioWorkerCount` — are the documented defaults at the time of writing and are
version-dependent: print them from `/actuator/configprops` and count the actual threads
rather than quoting a formula. The one structural fact worth committing to memory is that
both web starters on one classpath produces a **servlet** application, silently, and that
every benchmark run from such a process is invalid.*
