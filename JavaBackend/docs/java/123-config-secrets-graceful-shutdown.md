# 123 — Configuration, Secrets, Graceful Shutdown, and Rolling Deploys

## Phase: 11 — Distributed Systems & Production
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: a rolling deploy of `orderflow` that drops **zero requests**, proven by running the deploy underneath the Topic 65 load generator and showing an error count of exactly zero — not "low", zero — with every in-flight order committed, every Kafka message acknowledged, and every outbox batch finished.

---

## ELI5 anchor

A shop is closing for the night. There is a right order to do things in, and it is not the
obvious one.

**The wrong way.** At 9pm sharp the manager switches off the lights and locks the front
door. Three customers are still at the till with full baskets. Two more are walking through
the door as it closes on them. Everyone is angry, the till has a half-finished transaction
in it, and tomorrow morning somebody has to work out whether that customer was charged.

**The right way**, in strict order:

1. **Take the "OPEN" sign out of the window.** Nobody new walks in. This costs nothing and
   it happens *first*.
2. **Wait for the people already inside to finish.** They are mid-purchase; you serve them.
   You do not start new queues, but you do not throw out the person holding a basket.
3. **Only when the shop is empty, cash up and lock the doors.** Close the till, count the
   money, turn off the lights.

Step 1 is the one everybody skips, and skipping it is what ruins the evening. If you start
cashing up while the sign still says OPEN, people keep arriving to a shop that cannot serve
them.

Now, three details that make this the actual topic.

**The sign takes time to be believed.** You took it out of the window, but somebody across
the street who looked thirty seconds ago is already walking over. There is a gap between
"I am no longer open" and "everyone knows I am no longer open", and you must wait out that
gap before you start closing anything.

**Somebody is going to come and physically remove you from the building.** In Kubernetes,
after a fixed grace period, the platform stops asking and kills the process. If your longest
customer takes ninety seconds and the platform gives you thirty, you lose that customer no
matter how politely you were shutting down.

**It is not just the front door.** The shop also has a delivery bay (Kafka), a stockroom
worker mid-task (your thread pools), and a courier who has picked up a parcel but not yet
handed it over (the outbox relay). Closing the front door correctly and forgetting the
delivery bay still loses work.

The whole of this document is: **sign first, then drain, then close** — applied to every
door your JVM has.

---

## The bridge from what you know

### Graceful shutdown is conceptually familiar — the Java-specific part is the ORDER

You have done this in Node:

```js
process.on('SIGTERM', async () => {
  server.close();                 // stop accepting new connections
  await drainInFlight();
  await pool.end();               // close the database pool
  process.exit(0);
});
```

You know the shape. You know `SIGTERM` versus `SIGKILL`. You know
`terminationGracePeriodSeconds` and `preStop` hooks. **None of that is re-taught here.**

Four things are genuinely different in Java, and they are the reasons this topic exists.

**1. The ordering is enforced by a lifecycle framework, not by your code.**
In Node you wrote the sequence yourself, in one function, so the order was visible. In Spring
the sequence is assembled from `SmartLifecycle` beans, `@PreDestroy` methods and the web
server's own graceful mode, each with a *phase number*, and shutdown runs them in descending
phase order. **You cannot see the order by reading one file.** It is emergent, and getting it
wrong is the topic's main failure mode.

**2. Readiness is a separate signal from "the server is accepting connections".**
Node's `server.close()` does both at once — it stops accepting and it is the same object the
load balancer probes. In Spring, readiness (Topic 121) is an Actuator endpoint reflecting an
`AvailabilityState`, and the web server's graceful drain is a different mechanism entirely.
**They have to be sequenced deliberately.** Doing them in the wrong order is silent request
loss.

**3. There are more doors.** A Node service is usually an HTTP server and a database pool. By
Topic 123, `orderflow` also has a Kafka consumer (Topic 113), several thread pools (Topic 90),
a scheduled outbox relay (Topic 115), a metrics registry, and an OTel span exporter with a
buffered queue (Topic 119). Each needs draining, and each has its own API for it.

**4. Configuration has a defined ~15-level precedence chain instead of `process.env` plus a
config file.** You met this in Topic 43. What is new here is the *container* half: how
Kubernetes ConfigMaps and Secrets reach a Spring `Environment`, and why the mechanism you
choose determines whether the secret shows up in `kubectl describe`.

### Secrets: the analogue is exact, the exposure surface is bigger

`process.env.DB_PASSWORD` and `${DB_PASSWORD}` in `application.yml` are the same idea. What is
different is that a Spring application has **more places that will happily print your
configuration back to you**: `/actuator/env`, `/actuator/configprops`, the `--debug`
condition-evaluation report, and any framework that logs its own configuration at DEBUG.
Boot sanitises the well-known key patterns, and "well-known" is doing a lot of work in that
sentence.

**Verdict: PARTIAL ANALOGUE.** Same concepts, larger blast radius, and one Java-specific
hazard — the image layer — that Topic 122 set up.

### The one sentence to carry

**Readiness false → wait for that to propagate → drain in-flight work → close pools → exit.**
Every failure in this document is that sequence performed out of order, or truncated by a
grace period that was too short.

---

## What is this?

Three subjects that are one subject, because they meet in the deploy.

### 1. Configuration in a container

Where an `orderflow` pod's settings come from, in the order Spring resolves them, and why the
environment variable always wins. Topic 43 gave you the precedence chain. Here we care about
which *mechanism* to use for which kind of value, and the fact that a container's
configuration is fixed at start: **there is no reload.** Changing a ConfigMap does not change
a running JVM's `Environment` unless you have built something that makes it so. In practice,
in Kubernetes, the way you change configuration is a rolling deploy — which is why
configuration and rolling deploys are the same topic.

### 2. Secrets

A secret is a configuration value whose disclosure is an incident. That single property
changes everything about how it may be stored, transported, logged and displayed. The work is
mostly negative: enumerating every place a value can leak and closing each one.

### 3. Graceful shutdown

The sequence a JVM must perform between receiving `SIGTERM` and exiting, such that no unit of
work is lost. In Kubernetes:

```
kubectl delete / rolling update
   ├──> the pod is removed from Service endpoints   ) these two happen
   └──> SIGTERM is sent to PID 1 in the container   ) CONCURRENTLY
              ...
        terminationGracePeriodSeconds elapses
   └──> SIGKILL. Not negotiable. Nothing runs after this.
```

**The word "concurrently" is the single most important thing on this page.** Endpoint removal
is not instantaneous or ordered relative to the signal: the kubelet gets the signal to your
container while, in parallel, the endpoints controller updates the Service and every kube-proxy
or ingress or mesh sidecar reconciles that change. For a short window — commonly a second or
several — **traffic is still being routed to a pod that has already begun shutting down**.

If your application reacts to `SIGTERM` by immediately refusing connections, every request
routed during that window fails. That is the source of the 502s people blame on Kubernetes.

---

## Why does it matter?

**1. Because a rolling deploy is the most frequent planned outage you have.**

You deploy `orderflow` many times a month. Each deploy replaces every pod. If each pod
replacement drops even a handful of requests, you are manufacturing errors on a schedule,
against your own error budget (Topic 130), and the errors are correlated in time so they look
like an incident.

**2. Because the requests you drop are the expensive ones.**

Shutdown truncates whatever is longest-running. For `orderflow` that is `POST /orders`: the
transaction that reserves inventory, debits a wallet, calls a payment gateway, and writes an
outbox row. Killing that mid-flight is not a failed page view — it is a customer whose wallet
may be debited for an order that does not exist. Topic 115 gave you the outbox so a *crash*
does not lose an event; graceful shutdown is what makes the *planned* case not need it.

**3. Because a leaked secret is not recoverable by rolling back.**

A dropped request is retried. A secret printed at DEBUG into a log aggregator with ninety-day
retention, or baked into an image layer sitting in a registry, has to be rotated, and every
consumer of it has to be updated. The cost is entirely front-loaded onto prevention.

**4. Because the fix is cheap and almost nobody does it.**

Correct shutdown is roughly four configuration properties, one `preStop` hook, one lifecycle
bean, and a grace period derived from a number you already measured in Topic 65. It is
perhaps two hours of work. The reason it is rare is not difficulty; it is that nobody
measures deploy-time errors, so nobody knows they have the problem.

---

## Syntax breakdown

### `server.shutdown=graceful`

```properties
# Turn on the web server's drain phase. Default is `immediate`.
server.shutdown=graceful

# How long each shutdown PHASE may take before Spring stops waiting. Default 30s.
spring.lifecycle.timeout-per-shutdown-phase=25s
```

What `graceful` actually does, mechanically:

1. On `ContextClosedEvent`, the embedded web server (Tomcat, Jetty, Undertow) **stops
   accepting new connections**. The listening socket stops handing out new work.
2. It waits for **in-flight requests** to complete, up to
   `spring.lifecycle.timeout-per-shutdown-phase`.
3. If the timeout elapses with requests still running, remaining requests are terminated and
   shutdown proceeds anyway.

Three things it explicitly does **not** do, and all three are why this document is long:

- It does not set readiness false. That is a separate signal (Topic 121).
- It does not wait for the load balancer to notice anything.
- It does not drain Kafka consumers, thread pools, or scheduled tasks. Those are separate
  lifecycle participants.

> **`[BOOT 3.x DELTA]`** `server.shutdown=graceful` and
> `spring.lifecycle.timeout-per-shutdown-phase` are unchanged from Boot 2.3 onward, including
> 3.x and 4.x. What did change is the surrounding availability machinery: Boot 2.3 introduced
> `AvailabilityState` and the `livenessState`/`readinessState` health groups, and Boot
> 3.x/4.x refined the Actuator property names around them (Topic 121). If you are reading a
> pre-2.3 codebase, none of this exists and shutdown is a hand-rolled
> `Runtime.addShutdownHook`.

### `@PreDestroy` — and its three siblings

Four mechanisms run code on shutdown. They are **not** interchangeable, and choosing wrongly
is how ordering bugs happen.

```java
import jakarta.annotation.PreDestroy;

@Component
class PaymentGatewayClient {

    @PreDestroy
    void close() {
        // Runs during bean destruction, near the END of shutdown.
        // No ordering guarantee relative to other beans except via dependencies:
        // a bean is destroyed BEFORE the beans it depends on.
        httpClient.close();
    }
}
```

| Mechanism | When it runs | Ordering control | Use it for |
|---|---|---|---|
| `@PreDestroy` | bean destruction, late in shutdown | only by dependency graph | releasing a resource this bean owns |
| `DisposableBean.destroy()` | same point | same | the same thing, but couples you to Spring — prefer `@PreDestroy` |
| `@Bean(destroyMethod = "...")` | same point | same | third-party classes you cannot annotate |
| **`SmartLifecycle.stop()`** | **the `stop()` phase, BEFORE bean destruction, in descending `getPhase()` order** | **explicit numeric phase** | **anything where order matters** |

**This distinction is the load-bearing part of the section.** `@PreDestroy` is for releasing a
resource. `SmartLifecycle` is for *sequencing*. If you need "stop accepting work, then finish
work, then close the pool", you need phases, and `@PreDestroy` cannot express it.

```java
/**
 * A lifecycle participant with an explicit phase.
 * Spring STARTS in ascending phase order and STOPS in DESCENDING phase order.
 * So a HIGH phase number means "stopped early" -- which is what you want for
 * anything that ACCEPTS work.
 */
@Component
class OutboxRelayLifecycle implements SmartLifecycle {

    private volatile boolean running;

    @Override public int getPhase() { return Integer.MAX_VALUE - 100; }  // stops early
    @Override public boolean isAutoStartup() { return true; }
    @Override public boolean isRunning() { return running; }

    @Override
    public void start() { running = true; }

    @Override
    public void stop() {
        running = false;                 // the relay loop checks this and stops claiming rows
        relay.awaitCurrentBatch(Duration.ofSeconds(10));   // finish what is in flight
    }
}
```

**The phase rule, stated once so you can apply it:**

> **Higher phase = stops EARLIER.** Give a high phase to anything that *accepts* new work
> (listeners, schedulers, relays). Give a low phase to anything that *serves* work already
> accepted (pools, clients, connections). Spring's web server graceful shutdown runs at
> `SmartLifecycle.DEFAULT_PHASE` — so your acceptors should be numerically **above** it, and
> your resource closers **below** it or in `@PreDestroy`.

### Configuration precedence — the parts that matter in a container

Topic 43 has the full chain. Here is the operative subset, **highest priority first**:

| # | Source | In a pod, this is |
|---|---|---|
| 1 | Command-line arguments | `args:` in the container spec |
| 2 | `SPRING_APPLICATION_JSON` | a single env var holding JSON |
| 3 | OS environment variables | `env:` and `envFrom:` — **ConfigMaps and Secrets injected as env** |
| 4 | `spring.config.import` sources (including `configtree:`) | **mounted Secret/ConfigMap volumes** |
| 5 | Profile-specific `application-{profile}.yml` outside the jar | a mounted ConfigMap file |
| 6 | Profile-specific `application-{profile}.yml` inside the jar | baked into the image |
| 7 | `application.yml` outside the jar | mounted |
| 8 | `application.yml` inside the jar | baked in — **defaults only** |

**Relaxed binding** is what makes environment variables usable at all. Spring maps:

```
spring.datasource.url          ->  SPRING_DATASOURCE_URL
orderflow.payment.api-key      ->  ORDERFLOW_PAYMENT_API_KEY
orderflow.limits[0].max        ->  ORDERFLOW_LIMITS_0_MAX
```

Uppercase, dots and dashes to underscores, index brackets to underscores. This is why an env
var you set "just to test something" silently overrides a carefully reviewed YAML file, and
why "it works locally, not in the cluster" is so often an env var nobody remembers setting.

```bash
# The definitive answer to "where did this value come from", from the running app:
curl -s localhost:8081/actuator/env/spring.datasource.url | jq
# It lists EVERY property source that defines the key, in precedence order, with the winner
# reported as the resolved value.
```

### Secret sources, ranked

```properties
# 4. WORST: literal in a file inside the image. Never.
spring.datasource.password=hunter2

# 3. BAD: env var placeholder from a Kubernetes Secret injected with envFrom.
#    Works, and the value is visible in `kubectl describe pod`, in /proc/<pid>/environ,
#    in a crash dump, and to any process in the container.
spring.datasource.password=${DB_PASSWORD}

# 2. GOOD: a mounted Secret volume read as a config tree.
#    Each key is a FILE. Not in the environment, not in `describe`, file permissions apply,
#    and the kubelet can update the file contents in place.
spring.config.import=optional:configtree:/run/secrets/orderflow/

# 1. BEST for high-value credentials: a short-lived credential fetched at startup from a
#    secret manager using the pod's workload identity, with rotation. No static secret exists.
```

**`configtree:` is the mechanism worth learning**, because it is the one most Java engineers
have never seen. Given a mounted Secret at `/run/secrets/orderflow/` containing files:

```
/run/secrets/orderflow/
├── spring.datasource.password
├── orderflow.payment.api-key
└── orderflow.jwt.signing-key
```

…each *file name* becomes a property name and each file's *contents* becomes the value. Deeper
directories map to dots. So the Kubernetes Secret's keys are your Spring property names, with
no glue code and no env vars.

```yaml
# The pod side.
volumes:
  - name: orderflow-secrets
    secret:
      secretName: orderflow-secrets
      defaultMode: 0400
containers:
  - name: orderflow
    volumeMounts:
      - name: orderflow-secrets
        mountPath: /run/secrets/orderflow
        readOnly: true
```

**The `optional:` prefix matters.** Without it, a missing directory is a startup failure —
which is correct in production and infuriating on a laptop. With it, local development works
and production still gets the values, provided you also *validate* that the required
properties are present (below).

### Making a missing secret a startup failure, not a 3am mystery

```java
@ConfigurationProperties("orderflow.payment")
@Validated
public record PaymentProperties(
        @NotBlank String apiKey,
        @NotNull URI gatewayUrl,
        @Positive int timeoutMillis) {

    // toString is generated by the record and WOULD print the API key.
    // Override it. This one line prevents a whole class of leak.
    @Override
    public String toString() {
        return "PaymentProperties[apiKey=***, gatewayUrl=%s, timeoutMillis=%d]"
                .formatted(gatewayUrl, timeoutMillis);
    }
}
```

`@Validated` on `@ConfigurationProperties` turns a missing or blank secret into a **context
startup failure with the property name in the message**. That is a fast, loud, unambiguous
failure at deploy time instead of a `401 Unauthorized` from the payment gateway at peak hours.

### Actuator's sanitisation, and its limits

```properties
# Boot 3.x/4.x: values are masked by default. NEVER means never show them.
management.endpoint.env.show-values=never
management.endpoint.configprops.show-values=never
management.endpoint.health.show-details=when-authorized

# Do not expose these publicly at all -- put the management port behind a separate
# listener that only the cluster can reach (Topic 121).
management.server.port=8081
management.endpoints.web.exposure.include=health,info,metrics,prometheus,startup
```

> **`[BOOT 3.x DELTA]`** Boot 2.x used `management.endpoint.env.keys-to-sanitize`, a list of
> key patterns. Boot 3.x replaced it with the `show-values` enum
> (`never` / `always` / `when-authorized`) which defaults to masking. If you inherit a
> codebase with `keys-to-sanitize` in it, that property is doing nothing on a modern Boot and
> the sanitisation you are relying on may be different from what you think.
>
> **Flagged uncertainty:** I am confident `show-values` is the Boot 3.x mechanism and expect
> it unchanged on 4.1, but verify with `curl -s localhost:8081/actuator/configprops | jq` on
> your build and confirm secrets appear as `******`. Do not take my word for the default —
> check it, once, in your own service.

**The limit of sanitisation:** it protects the *endpoint*. It does nothing about a log line
you wrote, a `toString()` you did not override, an exception message that includes a
connection URL with credentials, or an image layer.

---

## Example 1 — minimal

The smallest correct shutdown, plus the property that proves it is working.

```properties
# application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=20s
logging.level.org.springframework.boot.web.embedded=DEBUG
```

```java
@RestController
class SlowController {

    private static final Logger log = LoggerFactory.getLogger(SlowController.class);

    @GetMapping("/api/slow")
    public String slow() throws InterruptedException {
        log.info("request started");
        Thread.sleep(10_000);          // stands in for a real long request
        log.info("request finished");
        return "done";
    }
}
```

**Prove it:**

```bash
# Terminal 1 — start the app.
java -jar target/orderflow.jar

# Terminal 2 — start a 10-second request, then immediately SIGTERM the JVM.
curl -s -w '\nHTTP %{http_code} in %{time_total}s\n' localhost:8080/api/slow &
sleep 1
kill -TERM $(pgrep -f orderflow.jar)

# Terminal 3 — while it is shutting down, try a NEW request.
sleep 1
curl -s -o /dev/null -w 'new request: HTTP %{http_code}\n' localhost:8080/api/slow
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| The first request returns `HTTP 200` after ~10s, **after** the SIGTERM | Graceful shutdown is working: in-flight work was allowed to finish |
| The first request fails immediately with a connection reset | `server.shutdown` is `immediate` (the default), or the property did not apply |
| The new request gets connection refused / no route | Correct — the listener stopped accepting. **This is also the problem readiness solves**, since a load balancer that has not noticed will keep sending these |
| Log line "Commencing graceful shutdown. Waiting for active requests to complete" | Boot's own confirmation. Grep for it in the drill |
| Log line "Graceful shutdown complete" | The drain finished within the timeout |
| The JVM exits before 10s with the request unfinished | `timeout-per-shutdown-phase` is shorter than the request, or something sent SIGKILL |

Now set the timeout to 5 seconds and repeat. The request is cut off at 5 seconds. **That is
the trap-1 mechanism, reproduced in thirty seconds on a laptop**, and it is exactly what
happens in production when your grace period is shorter than your p99.

---

## Example 2 — production scenario (on the project spine)

### The constraints

- `orderflow` under the Topic 65 load: 70% catalogue read, 20% order read, 10% order
  placement, constant arrival rate.
- **`POST /orders` is the long pole.** Its p99 from your baseline is the number every timeout
  in this section is derived from. You measured it in Topic 65; go and read it now, because
  the numbers below are formulas over *your* number, not constants.
- Multiple pods behind a Kubernetes Service. A rolling deploy replaces them one or two at a
  time.
- **Other doors:** a Kafka consumer (Topic 113), two `ThreadPoolTaskExecutor`s (Topic 90), a
  `@Scheduled` outbox relay (Topic 115), a HikariCP pool (Topic 109), and an OTel span
  exporter with a buffered queue (Topic 119).
- Secrets: database password, payment gateway API key, JWT signing key.
- **Acceptance:** k6 reports zero failed requests across a full rolling deploy.

### Step 1 — the timing budget, derived rather than guessed

Every number below is computed from your measured p99. Write the arithmetic down; it is the
part a reviewer will check.

**Fill in — shutdown budget (blank template):**

| Quantity | Source | Your value |
|---|---|---|
| `POST /orders` p99 (from Topic 65 baseline) | measured | |
| `POST /orders` max observed | measured | |
| Longest Kafka message processing time | measured | |
| Longest outbox relay batch duration | measured (`LongTaskTimer`, Topic 118) | |
| **A** — endpoint propagation delay (`preStop` sleep) | measured: see Proof 3 | |
| **B** — HTTP drain (`timeout-per-shutdown-phase`) | ≥ `POST /orders` max, with margin | |
| **C** — Kafka + executor + relay drain | ≥ the longest of those, with margin | |
| **`terminationGracePeriodSeconds`** | **> A + B + C + margin** | |

**The inequality that matters:**

```
terminationGracePeriodSeconds  >  preStop sleep
                                + spring.lifecycle.timeout-per-shutdown-phase
                                + every other drain
                                + margin
```

Get this backwards and Kubernetes SIGKILLs you *during* a drain you configured carefully,
which is worse than not configuring it — you spent the effort and still lost the requests.

### Step 2 — the application properties

```properties
# ---- HTTP drain -------------------------------------------------------------
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=25s

# ---- Readiness must be a separate, controllable signal (Topic 121) ----------
management.endpoint.health.probes.enabled=true
management.endpoint.health.group.readiness.include=readinessState,db
management.endpoint.health.group.liveness.include=livenessState
management.server.port=8081

# ---- Secrets ---------------------------------------------------------------
spring.config.import=optional:configtree:/run/secrets/orderflow/
management.endpoint.env.show-values=never
management.endpoint.configprops.show-values=never

# ---- Kafka: stop the containers before the rest of shutdown ----------------
spring.kafka.listener.immediate-stop=false

# ---- Hikari: do not let a connection close hang the exit -------------------
spring.datasource.hikari.connection-timeout=3000
```

### Step 3 — sequencing the doors with `SmartLifecycle`

This is the code that makes the ordering explicit rather than emergent.

```java
package com.orderflow.lifecycle;

/**
 * PHASE ORDER FOR ORDERFLOW SHUTDOWN.
 * Spring stops in DESCENDING phase order, so higher number = stopped earlier.
 *
 *   MAX-100  ReadinessLifecycle      readiness -> false, then WAIT for propagation
 *   MAX-200  AcceptorsLifecycle      Kafka listeners + @Scheduled relay stop CLAIMING work
 *   DEFAULT  (Spring's web server graceful shutdown runs here)
 *   MIN+200  DrainLifecycle          executors + relay finish IN-FLIGHT work
 *   @PreDestroy / bean destruction   pools, clients, exporters close
 */
final class ShutdownPhases {
    static final int READINESS = Integer.MAX_VALUE - 100;
    static final int ACCEPTORS = Integer.MAX_VALUE - 200;
    static final int DRAIN     = Integer.MIN_VALUE + 200;
    private ShutdownPhases() {}
}
```

**Door 1 — readiness, first and with a wait:**

```java
@Component
class ReadinessLifecycle implements SmartLifecycle {

    private static final Logger log = LoggerFactory.getLogger(ReadinessLifecycle.class);
    private final ApplicationEventPublisher publisher;
    private final Duration propagationDelay;      // from configuration, MEASURED
    private volatile boolean running;

    @Override public int getPhase() { return ShutdownPhases.READINESS; }
    @Override public boolean isRunning() { return running; }
    @Override public void start() { running = true; }

    @Override
    public void stop() {
        running = false;

        // 1. Take the sign out of the window.
        AvailabilityChangeEvent.publish(publisher, this, ReadinessState.REFUSING_TRAFFIC);
        log.info("readiness set to REFUSING_TRAFFIC; waiting {} for endpoint propagation",
                 propagationDelay);

        // 2. Wait for everyone across the street to notice. Requests that arrive during
        //    this window are STILL SERVED NORMALLY -- that is the entire point.
        try {
            Thread.sleep(propagationDelay.toMillis());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        log.info("propagation wait complete; proceeding with shutdown");
    }
}
```

> **A note on belt and braces.** If you also use a Kubernetes `preStop` sleep (Step 4), this
> in-application wait is redundant for the HTTP path — `preStop` runs *before* `SIGTERM` is
> delivered, so it covers the window more cleanly. Keep the in-application version when you
> want the same behaviour outside Kubernetes (a bare container, a VM, a local run), or when
> your platform's `preStop` support is unreliable. **Do not configure both to their full
> value**, or you double the delay for no benefit. Pick the primary, and document which.

**Door 2 — stop accepting, before draining:**

```java
@Component
class AcceptorsLifecycle implements SmartLifecycle {

    private final KafkaListenerEndpointRegistry kafkaRegistry;
    private final OutboxRelay relay;
    private volatile boolean running;

    @Override public int getPhase() { return ShutdownPhases.ACCEPTORS; }
    @Override public boolean isRunning() { return running; }
    @Override public void start() { running = true; }

    @Override
    public void stop() {
        running = false;

        // Kafka: stop polling for NEW records. Records already polled are still processed,
        // and offsets for completed work are committed. This prevents the rebalance-storm
        // shape from Topic 113 during a rolling deploy.
        kafkaRegistry.getListenerContainers().forEach(MessageListenerContainer::stop);

        // Outbox relay: stop CLAIMING new batches. The batch in flight finishes in the
        // DRAIN phase below -- Topic 115's requirement.
        relay.stopClaiming();
    }
}
```

**Door 3 — drain in-flight background work:**

```java
@Component
class DrainLifecycle implements SmartLifecycle {

    private final List<ThreadPoolTaskExecutor> executors;   // from Topic 119's central factory
    private final OutboxRelay relay;
    private volatile boolean running;

    @Override public int getPhase() { return ShutdownPhases.DRAIN; }
    @Override public boolean isRunning() { return running; }
    @Override public void start() { running = true; }

    @Override
    public void stop() {
        running = false;

        // Topic 115: the relay must finish the batch it claimed, or those rows stay
        // locked until the claim expires and republish later -- correct but slow.
        relay.awaitCurrentBatch(Duration.ofSeconds(10));

        // Topic 90: executors were configured with waitForTasksToCompleteOnShutdown(true)
        // and an awaitTerminationSeconds; this just makes the wait explicit and ordered.
        for (ThreadPoolTaskExecutor executor : executors) {
            executor.shutdown();
        }
    }
}
```

**Door 4 — flush the span exporter, which nobody remembers:**

```java
@Component
class TelemetryFlushLifecycle implements SmartLifecycle {

    private final OpenTelemetrySdk sdk;

    @Override public int getPhase() { return ShutdownPhases.DRAIN - 1; }   // after drain

    @Override
    public void stop() {
        // Topic 119: the BatchSpanProcessor holds a queue. Without this flush, the last
        // batch is lost -- and that is exactly the batch covering the shutdown.
        sdk.getSdkTracerProvider().forceFlush().join(5, TimeUnit.SECONDS);
        sdk.getSdkTracerProvider().shutdown().join(5, TimeUnit.SECONDS);
    }
}
```

**Door 5 — the pools, last, by dependency-ordered destruction:**

HikariCP and the Hibernate `SessionFactory` are closed by Spring during bean destruction,
which happens **after** all `stop()` phases. That ordering is correct and free — as long as
nothing above holds a connection past its drain. Which is why draining comes first.

### Step 4 — the pod spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # never go below the current replica count
      maxSurge: 1              # add one new pod, then remove one old
  template:
    spec:
      # Must exceed preStop + HTTP drain + background drain + margin.
      # DERIVE THIS from the budget table, do not copy this number.
      terminationGracePeriodSeconds: 60
      containers:
        - name: orderflow
          image: registry.internal/orderflow@sha256:<digest>
          lifecycle:
            preStop:
              exec:
                # Runs BEFORE SIGTERM is delivered. This is the endpoint-propagation
                # window, and it is the single highest-value line in this file.
                command: ["sh", "-c", "sleep 8"]
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8081 }
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8081 }
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: 8081 }
            periodSeconds: 5
            failureThreshold: 30      # from Topic 122's measured cold start
          envFrom:
            - configMapRef: { name: orderflow-config }     # non-secret config
          volumeMounts:
            - name: orderflow-secrets
              mountPath: /run/secrets/orderflow
              readOnly: true
      volumes:
        - name: orderflow-secrets
          secret:
            secretName: orderflow-secrets
            defaultMode: 0400
```

**Why `preStop: sleep` and not something cleverer.** The pod is removed from Service endpoints
and sent `SIGTERM` concurrently. `preStop` runs *before* the signal, so a sleep there holds the
container fully alive and fully serving while the endpoint removal propagates through
kube-proxy, the ingress, and any mesh sidecars. It looks crude. It is the standard solution,
it is what the platform documentation recommends, and it is correct.

**How long?** Measure it (Proof 3). It is a property of your cluster's control plane and data
plane, not of your application, so it is the one number in this document you cannot derive
from `orderflow`.

**`maxUnavailable: 0`** means the rollout never dips below the current healthy count. Combined
with a `PodDisruptionBudget`, it also protects you during node drains — which are the
*unplanned* version of this whole exercise.

### Step 5 — the exec-form entry point, or none of the above runs

```dockerfile
# CORRECT — exec form. `java` becomes PID 1 and receives SIGTERM directly.
ENTRYPOINT ["java", "-XX:SharedArchiveFile=/app/orderflow.jsa", \
            "org.springframework.boot.loader.launch.JarLauncher"]

# WRONG — shell form. Docker runs `/bin/sh -c "java ..."`, so the SHELL is PID 1.
# Many shells do not forward SIGTERM to their child. Your JVM never sees the signal,
# runs happily until terminationGracePeriodSeconds elapses, and is then SIGKILLed
# with every in-flight order still in flight.
# ENTRYPOINT java -jar /app/app.jar
```

This is Topic 122's file and Topic 123's failure, which is why they are adjacent. **A perfect
`SmartLifecycle` sequence behind a shell-form `ENTRYPOINT` does nothing at all**, and the
symptom — requests dropped at every deploy despite correct configuration — sends people
looking everywhere except the Dockerfile.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the grace period is shorter than the p99

**Wrong approach**

```yaml
terminationGracePeriodSeconds: 30      # the Kubernetes default, never revisited
```

```properties
spring.lifecycle.timeout-per-shutdown-phase=30s   # equal to the grace period
```

…on a service whose `POST /orders` occasionally takes considerably longer than that, and whose
Kafka consumer processes a batch that can take tens of seconds.

**Exact symptom**

- A small number of `POST /orders` requests fail during every deploy, always the slowest ones.
- The client sees a connection reset, not an HTTP error code, so it looks like a network fault
  and gets blamed on the mesh.
- Application logs show `Commencing graceful shutdown` and then **stop mid-sentence**. There is
  no "Graceful shutdown complete" line.
- `kubectl describe pod` on the terminated pod shows the container's last state as terminated
  with a non-zero exit or a signal, not a clean exit.
- Some orders are in a half-applied state: inventory reserved, wallet debited, no payment row —
  the exact scenario the transaction was supposed to prevent, now caused by process death
  rather than by a bug.
- It correlates with deploys and with node drains, and with nothing else, which makes it
  maddening to reproduce on demand.

**Root cause**

Two separate mistakes that compound.

First, `terminationGracePeriodSeconds` is a **hard ceiling**. When it expires, Kubernetes sends
`SIGKILL`, which cannot be caught, blocked or delayed. Nothing after it runs — no `@PreDestroy`,
no transaction rollback, no span flush.

Second, the internal timeout equals the external one, leaving no room for anything else. Even
if the HTTP drain finishes at exactly 30 seconds, the Kafka drain, the relay batch, the span
flush and bean destruction all still need to happen, and there is zero time left for them.

**Fix**

Derive the numbers from the measurement, with the inequality strict:

```
terminationGracePeriodSeconds  >  preStop
                                + spring.lifecycle.timeout-per-shutdown-phase
                                + background drains
                                + margin
```

```yaml
terminationGracePeriodSeconds: 60          # from YOUR budget table
lifecycle:
  preStop: { exec: { command: ["sh","-c","sleep 8"] } }
```

```properties
spring.lifecycle.timeout-per-shutdown-phase=25s
```

**And bound the requests themselves**, because an unbounded request makes any grace period
wrong eventually:

```properties
# A request that cannot finish within the drain window should not exist.
spring.mvc.async.request-timeout=20s
spring.datasource.hikari.connection-timeout=3000
# Topic 111: every downstream call has a timeout, so no request can outlive the budget.
```

**Assert it in the deployment pipeline**, because these numbers drift apart the moment someone
edits one of the two files:

```bash
GRACE=$(kubectl get deploy orderflow -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}')
DRAIN=$(kubectl get cm orderflow-config -o jsonpath='{.data.SPRING_LIFECYCLE_TIMEOUT_PER_SHUTDOWN_PHASE}')
# Fail the pipeline if GRACE is not comfortably greater than preStop + DRAIN.
```

---

### Trap 2 — readiness does not go false before shutdown begins

**Wrong approach**

`server.shutdown=graceful` is set. There is no `preStop` hook and no readiness sequencing. The
assumption is that Kubernetes removes the pod from the Service *before* sending `SIGTERM`.

**Exact symptom**

- A burst of `502 Bad Gateway` or `connection refused` at the ingress **at the exact moment each
  pod terminates**, repeated once per pod, for the whole rollout.
- The count is small per pod and perfectly correlated with the deploy timeline.
- Application logs show the graceful drain running correctly and completing successfully.
  **The application did everything right.** The failing requests never reached it.
- k6 shows a short spike of failures per pod replacement — the shape is a sawtooth with one
  tooth per pod.
- Increasing `terminationGracePeriodSeconds` does not help at all, which is the diagnostic
  clue: this is not a drain-duration problem.

**Root cause**

Endpoint removal and `SIGTERM` are **concurrent**, not sequential. The moment your application
stops accepting connections, there is still a window during which kube-proxy rules, the
ingress controller's endpoint list, and any sidecar's cluster membership have not yet
converged. Requests routed during that window arrive at a socket that is no longer listening,
so they fail at the TCP level — which is why they appear as 502 or connection-refused rather
than as an HTTP error your application produced.

**Fix**

Insert a wait **before** the application reacts to the signal:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 8"]     # measured; see Proof 3
```

During that sleep the container is fully alive and serving normally. Endpoint removal
propagates. Only then does `SIGTERM` arrive and the drain begin.

Optionally add the in-application readiness flip (Example 2, door 1) so the behaviour also
holds outside Kubernetes.

**Measure the propagation delay rather than guessing:**

```bash
# Run the Topic 65 load. Delete one pod. Timestamp the first and last failed request,
# then compare with the pod's deletion timestamp.
kubectl delete pod <pod> --grace-period=30 &
kubectl get events --watch --field-selector involvedObject.name=<pod>
```

The interval between "pod deletion requested" and "no more requests arrive at that pod" is
your `preStop` duration. Add margin, and re-measure after any change to the ingress or mesh.

**The trap inside the trap:** an ingress or service mesh sidecar has its *own* shutdown
sequence. If the sidecar exits before the application, in-flight requests fail even though
your application is still draining perfectly. That is why the failures sometimes persist after
you have done everything in this section correctly, and why it is worth knowing whether your
mesh has a "hold the sidecar until the app exits" setting.

---

### Trap 3 — the shell-form `ENTRYPOINT`

**Wrong approach**

```dockerfile
ENTRYPOINT java -jar /app/app.jar
```

**Exact symptom**

- Requests are dropped on **every** deploy, despite `server.shutdown=graceful` being set and
  verified in the configuration.
- The application logs contain **no shutdown messages whatsoever**. Not "Commencing graceful
  shutdown", nothing. The logs simply stop.
- Pod termination always takes exactly `terminationGracePeriodSeconds` — never less, whatever
  the load. That perfectly constant duration is the giveaway.
- The same jar shuts down correctly when you run it directly and press Ctrl+C, which sends
  people looking for a Kubernetes bug.

**Root cause**

Shell form makes Docker run `/bin/sh -c "java -jar /app/app.jar"`. The shell is PID 1 and the
JVM is its child. `SIGTERM` goes to PID 1 — the shell — and many shells do not forward signals
to children, and do not exit while a foreground child is running. So the JVM never learns it is
supposed to stop. It runs happily until the grace period expires and `SIGKILL` arrives, which
kills everything instantly with no drain, no `@PreDestroy`, and no flush.

**Fix**

```dockerfile
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Exec form: no shell, `java` is PID 1, signals arrive directly.

**If you genuinely need a shell** (to expand an environment variable into a JVM flag, say),
either use `exec` explicitly, or better, avoid the shell entirely:

```dockerfile
# Option A: exec replaces the shell process, so java becomes PID 1.
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]

# Option B (preferred): JAVA_TOOL_OPTIONS is read by the JVM itself. No shell needed.
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70"
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Verify:**

```bash
# PID 1 must be java, not sh.
docker exec <container> ps -o pid,comm
# Then prove the signal lands.
docker stop -t 30 <container>          # sends SIGTERM, waits, then SIGKILL
docker logs <container> | tail -20     # you MUST see the shutdown log lines
```

---

### Trap 4 — secrets in the image, in the environment, or in the logs

**Wrong approach — four variants, all common**

```dockerfile
# 1. Baked into an image layer.
COPY application-prod.yml /app/config/          # contains the database password
```

```java
// 2. Logged, usually while debugging something else.
log.debug("connecting with config: {}", dataSourceProperties);   // toString() prints it
```

```yaml
# 3. In the pod spec, as a literal.
env:
  - name: SPRING_DATASOURCE_PASSWORD
    value: "hunter2"
```

```properties
# 4. Exposed by an Actuator endpoint on the main port with details on.
management.endpoints.web.exposure.include=*
management.endpoint.env.show-values=always
```

**Exact symptom**

Symptoms are the wrong frame here, because the failure is usually discovered by *someone else*.
The observable facts:

- Variant 1: `docker history` shows the layer, and **the secret survives even if a later
  `RUN rm` deletes the file** — layers are additive, so the file is still present in an earlier
  layer and extractable by anyone who can pull the image.
- Variant 2: the value is in your log aggregator, indexed and searchable, with whatever
  retention that system has. Everyone who can search logs can now read it.
- Variant 3: `kubectl describe pod` prints it in plain text to anyone with pod-read
  permission; it is also in `/proc/<pid>/environ`, so any process in the container can read it,
  and it appears in some crash dumps.
- Variant 4: an HTTP GET returns your configuration to anyone who can reach the port.
- The unifying symptom: **an audit, a scanner, or an incident finds it**, and the remediation
  is rotation across every consumer, not a code fix.

**Root cause**

Every one of these is a *default* that is convenient. Baking config into an image makes builds
simpler. `toString()` on a properties object is generated for you and prints everything.
`envFrom` is one line. Exposing all Actuator endpoints makes debugging easier. Nobody chooses
to leak a secret; they choose the convenient option and the leak is a consequence.

**Fix**

Layered, because no single control is sufficient:

1. **Never in the image.** `.dockerignore` excludes local config; CI scans layers.

```bash
# In CI: fail the build if a secret pattern appears in any layer.
docker save orderflow:${TAG} | tar -xO | grep -aE 'BEGIN (RSA )?PRIVATE KEY|password\s*[:=]' \
  && { echo "FAIL: secret material in image layers"; exit 1; }
```

2. **Prefer mounted files to environment variables.**

```properties
spring.config.import=optional:configtree:/run/secrets/orderflow/
```

A mounted Secret does not appear in `kubectl describe`, is not in `/proc/<pid>/environ`, and
carries file permissions.

3. **Override `toString()` on every properties class that holds a secret**, and use records
   with `@Validated` so a missing secret fails startup loudly (Syntax breakdown).

4. **Lock the endpoints down:** separate management port, `show-values=never`, explicit
   `exposure.include` rather than `*`.

5. **Add a logging guard**, because rule 3 depends on someone remembering:

```java
// A Logback filter (Topic 120) that drops any event whose message matches a secret shape.
// Blunt, and worth it: the cost of a false positive is a missing log line; the cost of a
// false negative is a rotation exercise.
public class SecretRedactingFilter extends Filter<ILoggingEvent> {
    private static final Pattern SECRET = Pattern.compile(
            "(?i)(password|secret|api[-_]?key|token|authorization)\\s*[=:]\\s*\\S+");

    @Override
    public FilterReply decide(ILoggingEvent event) {
        return SECRET.matcher(event.getFormattedMessage()).find()
                ? FilterReply.DENY : FilterReply.NEUTRAL;
    }
}
```

6. **Rotate on a schedule**, so the blast radius of an unknown leak is bounded by time rather
   than by your confidence that there is no leak.

---

### Trap 5 — graceful shutdown for HTTP only

**Wrong approach**

```properties
server.shutdown=graceful      # and nothing else
```

The team tests the deploy with a load generator hitting HTTP endpoints, sees zero errors, and
declares the problem solved. Meanwhile the service also consumes Kafka, runs a scheduled outbox
relay, and dispatches notification work to a thread pool.

**Exact symptom**

Four distinct symptoms, all appearing after deploys and none of them looking like a shutdown
problem:

- **Duplicate side effects.** A Kafka message was processed — inventory decremented,
  notification sent — but the offset was not committed before the process died. After the
  rebalance, another consumer processes it again. Idempotency (Topic 116) saves you if you
  built it; if you did not, this is a double decrement.
- **A rebalance storm at every deploy.** Consumers vanishing without a clean `stop()` are
  detected by session timeout rather than by leaving the group, so the group stalls for the
  timeout duration on each pod replacement (Topic 113).
- **Outbox rows stuck.** The relay claimed a batch with `SELECT ... FOR UPDATE SKIP LOCKED` and
  died mid-batch. Those rows are unpublished until the claim expires — correct eventually, but
  with a latency spike that shows up as a gap in downstream processing.
- **Notification work silently lost.** Tasks queued in a `ThreadPoolTaskExecutor` are discarded
  when the pool is shut down without waiting. No error; the notification simply never happens,
  and the customer's "your order is confirmed" email does not arrive.

None of these produce an HTTP error, so the load test that only checks HTTP passes.

**Root cause**

`server.shutdown=graceful` is scoped precisely to the embedded web server. Every other
component has its own lifecycle, and by default several of them stop abruptly. The mental model
"I enabled graceful shutdown" is doing damage here, because it is true and insufficient.

**Fix**

Enumerate every door and drain each explicitly (Example 2, step 3):

```java
// Kafka: stop containers before the rest of shutdown, so polling stops but in-flight
// records are completed and their offsets committed.
kafkaRegistry.getListenerContainers().forEach(MessageListenerContainer::stop);
```

```java
// Executors (Topic 90): configured once, in the central factory.
executor.setWaitForTasksToCompleteOnShutdown(true);
executor.setAwaitTerminationSeconds(20);
```

```java
// Scheduled tasks (Topic 115): stop claiming, then finish the claimed batch.
@Bean
TaskSchedulerCustomizer schedulerShutdown() {
    return scheduler -> {
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(15);
    };
}
```

```java
// Telemetry (Topic 119): flush the span queue or lose the shutdown traces.
sdk.getSdkTracerProvider().forceFlush().join(5, TimeUnit.SECONDS);
```

**And test the right thing.** A deploy test that only asserts on HTTP errors will keep passing
while all four of these fail. The acceptance criteria must include: zero duplicate side effects,
zero rebalances beyond the expected one per pod, outbox depth returning to baseline promptly,
and every dispatched notification accounted for.

---

## Hands-on proof

### Setup

```bash
docker compose -f load/docker-compose.yml up -d --wait
# Confirm the shutdown-relevant properties are actually in effect:
curl -s localhost:8081/actuator/env/server.shutdown | jq
curl -s localhost:8081/actuator/env/spring.lifecycle.timeout-per-shutdown-phase | jq
```

### Proof 1 — where did this configuration value come from?

```bash
# Every source that defines the key, in precedence order.
curl -s localhost:8081/actuator/env/spring.datasource.url | jq

# All active property sources, in order.
curl -s localhost:8081/actuator/env | jq -r '.propertySources[].name'

# Confirm secrets are masked.
curl -s localhost:8081/actuator/configprops | grep -i -E 'password|secret|key' | head
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| The winning value comes from `systemEnvironment` | An env var is overriding your file. Relaxed binding at work — usually intentional, sometimes not |
| The winning value comes from `configtree:...` | A mounted Secret is supplying it. This is the shape you want for credentials |
| Any secret shown in plain text | `show-values` is not `never`, or the key does not match Boot's sanitisation patterns. **Fix before anything else** |
| A property source you did not expect | Something is importing configuration you did not know about. Find it before you debug anything else |

### Proof 2 — the drain, on your own machine

```bash
APP_PID=$(pgrep -f orderflow.jar)

# Start a long request, then signal.
curl -s -w '\n%{http_code} %{time_total}s\n' localhost:8080/api/orders/slow-test &
sleep 1
kill -TERM $APP_PID

# Watch the sequence in the logs.
docker compose logs -f app | grep -iE 'shutdown|graceful|readiness|lifecycle|stopping'
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `Commencing graceful shutdown. Waiting for active requests to complete` | The drain started |
| `Graceful shutdown complete` | It finished within the timeout. This is the success line |
| The long request returns 200 after the signal | In-flight work was protected |
| No shutdown lines at all | The JVM never got the signal — shell-form `ENTRYPOINT` (Trap 3) |
| Your own lifecycle log lines in the expected order | The phase numbers are doing what you intended |
| Lifecycle lines in the *wrong* order | Re-check `getPhase()` — remember, **higher stops earlier** |

### Proof 3 — measure the endpoint propagation delay

This is the number you cannot derive; it is a property of your cluster.

```bash
# Terminal 1: continuous requests, one per 100ms, logging failures with timestamps.
while true; do
  ts=$(date +%s.%N)
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 2 http://orderflow.internal/api/products/SKU-1001)
  [ "$code" != "200" ] && echo "$ts FAIL $code"
  sleep 0.1
done

# Terminal 2: delete one pod and timestamp it.
date +%s.%N; kubectl delete pod <pod-name>
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Failures starting at deletion time and stopping after an interval | That interval **is** your propagation delay. Set `preStop` above it, with margin |
| No failures at all | Either you already have a `preStop`, or your ingress drains connections itself. Verify which — do not assume you are safe by luck |
| Failures continuing for the whole grace period | Not propagation: the application is refusing connections while still an endpoint. Check `preStop` is actually configured and running |
| Failures resuming later in the rollout | A second pod is being replaced. Expected; count them per pod |

### Proof 4 — the zero-dropped-request deploy, which is the spine deliverable

```bash
# Terminal 1: the Topic 65 load, at the baseline rate, with a strict threshold.
k6 run --out json=deploy-run.json load/k6/orderflow.js

# Terminal 2: roll the deployment while the load runs.
kubectl set image deployment/orderflow orderflow=registry.internal/orderflow@sha256:<new>
kubectl rollout status deployment/orderflow --timeout=300s

# Then: the only number that matters.
jq -s '[.[] | select(.type=="Point" and .metric=="http_req_failed") | .data.value] | add' deploy-run.json
```

Make the threshold explicit in the k6 script so the run *fails* rather than reporting:

```javascript
export const options = {
  thresholds: {
    'http_req_failed': ['count==0'],       // ZERO. Not a rate, not "low".
    'dropped_iterations': ['count==0'],
  },
};
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Zero failed requests across the whole rollout | The deliverable is met. Record the configuration that produced it |
| A small burst per pod replacement | Endpoint propagation — Trap 2. Add or lengthen `preStop` |
| Failures only on `POST /orders` | The drain window is shorter than that endpoint's tail — Trap 1 |
| Connection resets rather than HTTP errors | The socket closed under the client. Almost always propagation or the grace period |
| Zero HTTP failures but duplicate notifications or stuck outbox rows | Trap 5: HTTP is drained, background work is not |
| p99 rising during the rollout with no errors | Not a shutdown problem — fewer pods serving, or cold-JVM warm-up (Topic 122) |

### Proof 5 — the background doors, checked individually

```bash
# Kafka: a clean stop leaves the group without waiting for a session timeout.
docker compose logs app | grep -iE 'partitions revoked|partitions assigned|LeaveGroup'
kubectl exec -it kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 --describe --group inventory

# Outbox: depth must return to its baseline shortly after the rollout.
curl -s localhost:8081/actuator/metrics/orderflow.outbox.pending | jq

# Executors: queued tasks must be zero at the moment of shutdown, not discarded.
curl -s 'localhost:8081/actuator/metrics/executor.queued?tag=name:orderflow-notify' | jq

# Traces: the shutdown spans must be present, which proves the exporter flushed.
curl -s 'http://localhost:16686/api/traces?service=orderflow&limit=5' | jq '.data | length'
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `partitions revoked` logged before the pod exits | The consumer left the group cleanly. No session-timeout stall |
| Consumer lag spiking for the session-timeout duration at each pod replacement | Containers are **not** being stopped cleanly — Trap 5 |
| Outbox depth spiking and staying high after the rollout | The relay died mid-batch and the claim has not expired |
| Executor queue non-zero at shutdown | Queued work was discarded. Set `waitForTasksToCompleteOnShutdown(true)` |
| No traces covering the shutdown window | The span exporter was not flushed. Add the flush lifecycle |

---

## Practice exercises

### 1 — Easy: prove your shutdown is graceful, then prove it is not

1. Set `server.shutdown=graceful` and a 20-second phase timeout.
2. Add an endpoint that sleeps for 10 seconds.
3. Start a request, `kill -TERM` the JVM, and confirm the request completes.
4. Set the phase timeout to 5 seconds and repeat. Confirm the request is cut off.
5. Change the Dockerfile to shell-form `ENTRYPOINT`, run in a container, `docker stop`, and
   confirm **no shutdown log lines appear at all**.
6. Restore exec form and confirm they come back.

**Deliverable:** the four log excerpts and one sentence per case explaining the mechanism.
**Acceptance:** you can state from your own observation what each of the three settings changes,
and you recognise the shell-form signature (constant termination time, silent logs).

### 2 — Medium: audit every door (combines Topics 90, 109, 113, 115, 118, 119, 121)

**Goal:** a written inventory of everything in `orderflow` that must drain, with the mechanism
and the timeout for each.

**Fill in — shutdown door inventory (blank template):**

| Component | Accepts work? | Drain mechanism | Timeout | Phase | Verified how |
|---|---|---|---|---|---|
| HTTP server | yes | `server.shutdown=graceful` | | DEFAULT | k6 during rollout |
| Readiness signal | n/a | `preStop` + `AvailabilityChangeEvent` | | high | Proof 3 |
| Kafka listener containers | yes | `MessageListenerContainer.stop()` | | high | `partitions revoked` in logs |
| `@Scheduled` outbox relay | yes | stop claiming, then await batch | | high / low | outbox depth metric |
| Notification executor | no (queued) | `waitForTasksToCompleteOnShutdown` | | low | queue depth at shutdown |
| Reconciliation executor | no | same | | low | queue depth |
| HikariCP pool | no | bean destruction | n/a | destroy | no connection errors in logs |
| OTel span exporter | no | `forceFlush()` | | low | shutdown traces present |
| Metrics registry | no | final scrape may be missed | n/a | — | accept and document |

1. Complete every cell for your service.
2. Implement the missing drains.
3. Verify each row independently, with the command in the last column.
4. Compute the total worst-case shutdown duration and set
   `terminationGracePeriodSeconds` above it.

**Acceptance:**
- [ ] Every component that accepts or holds work appears in the table.
- [ ] Each has a mechanism, a timeout, and an independent verification.
- [ ] The arithmetic for `terminationGracePeriodSeconds` is written down and checked in CI.
- [ ] The `ENTRYPOINT` is exec form and `ps` confirms `java` is PID 1.

### 3 — Hard: production simulation — the zero-dropped-request deploy

**Goal:** the spine deliverable. A rolling deploy under the Topic 65 load with zero failures,
and evidence for every claim.

1. Start the Topic 65 load at your baseline rate with `http_req_failed: ['count==0']`.
2. Roll out a one-line change.
3. Record: total failed requests, failures per pod replacement, p50/p95/p99 during versus
   before the rollout, rollout wall-clock, Kafka rebalance count, outbox depth curve,
   duplicate-side-effect count.
4. If any number is non-zero, diagnose it with the trap table and fix it.
5. Re-run until failures are zero **twice consecutively** — once could be luck.
6. Then make it worse on purpose and confirm each diagnosis:
   - remove `preStop` → expect a burst per pod;
   - shorten `timeout-per-shutdown-phase` below the `POST /orders` p99 → expect failures on
     that endpoint only;
   - shell-form `ENTRYPOINT` → expect constant termination time and silent logs;
   - remove the Kafka container stop → expect rebalance stalls and duplicate processing.

**Fill in — rolling deploy results (blank template):**

| Measurement | Baseline (no deploy) | Deploy, before fixes | Deploy, after fixes |
|---|---|---|---|
| Total requests | | | |
| Failed requests | | | |
| Failures per pod replacement | | | |
| p50 / p95 / p99 | | | |
| Connection resets | | | |
| Kafka rebalances | | | |
| Duplicate side effects | | | |
| Outbox depth peak | | | |
| Rollout wall-clock | | | |

**Acceptance:** zero failed requests on two consecutive runs; every deliberate regression in
step 6 reproduces the predicted symptom; and you can name, for each symptom, the single
configuration line responsible.

---

## Interview questions

### Q1 — "How do you deploy a service without dropping requests?"

**MID-LEVEL ANSWER**

"Enable graceful shutdown so in-flight requests can finish, use a rolling update strategy, and
make sure the readiness probe is configured so traffic only goes to healthy pods."

**SENIOR ANSWER**

"The core of it is an ordering, and the order is the part people get wrong: readiness goes
false first, then you wait for that to propagate, then you drain in-flight work, then you close
pools, then you exit.

The step that is missing from most implementations is the wait. In Kubernetes, endpoint removal
and `SIGTERM` happen concurrently, not in sequence. So there is a window where kube-proxy, the
ingress and any sidecars have not converged yet and traffic is still being routed to a pod that
has already started shutting down. If the application reacts to `SIGTERM` by closing its
listener immediately, everything routed in that window fails at the TCP level — which is why
those errors show up as 502s or connection resets rather than as an HTTP status your app
produced. The fix is a `preStop` hook that just sleeps, because `preStop` runs *before* the
signal, so the container stays fully alive and serving while the endpoint removal propagates.
How long it sleeps is a measured property of the cluster, not a guess.

Then the drain. `server.shutdown=graceful` with
`spring.lifecycle.timeout-per-shutdown-phase` gives the HTTP drain, and that timeout must
exceed the p99 — really the observed max — of the longest endpoint. For us that is order
placement. And `terminationGracePeriodSeconds` has to be strictly greater than `preStop` plus
the HTTP drain plus every background drain plus margin, because it is a hard ceiling: when it
expires, `SIGKILL` arrives and nothing after it runs.

The part that is specific to a service like ours is that HTTP is not the only door. We also
have Kafka listener containers, thread pools, a scheduled outbox relay, and a span exporter
with a buffered queue. Each drains separately. I sequence them with `SmartLifecycle` phases —
higher phase stops earlier — so the things that *accept* work stop before the things that
*finish* work, and pools close last during bean destruction.

Two things I would check before believing any of it. The `ENTRYPOINT` must be exec form, or the
shell is PID 1 and the JVM never receives `SIGTERM` — the signature is termination taking
exactly the grace period every time with no shutdown lines in the log. And I would prove it
rather than assert it: run the deploy underneath the load generator with a threshold of exactly
zero failed requests, and treat anything above zero as a bug."

**WHAT SEPARATES THEM**

The mid-level answer names the right features. The senior answer knows the *sequence* and why
the propagation wait is the missing step, derives the timeouts from a measured p99 with the
inequality stated, knows the doors beyond HTTP, knows the PID 1 trap, and finishes with a
falsifiable acceptance criterion instead of a claim.

**FOLLOW-UP:** *"How do you pick the `preStop` duration?"* — Measure it: run continuous
requests, delete a pod, and record the interval between the deletion timestamp and the last
failed request. That is a property of the cluster's control plane and data plane, not of the
application, so it has to be re-measured after ingress or mesh changes. Then add margin. And
be aware that if the mesh sidecar shuts down before the application, you can still lose
requests with a perfect `preStop` — that is a separate setting on the sidecar.

---

### Q2 — "Where do your secrets come from, and how do you know they have not leaked?"

**MID-LEVEL ANSWER**

"They come from Kubernetes Secrets, injected as environment variables, and we reference them
in `application.yml` with `${...}`. We do not commit them to git."

**SENIOR ANSWER**

"Environment variables work, and they are the weakest of the workable options, so I would name
the trade rather than defend it. A secret injected with `envFrom` is visible in
`kubectl describe pod` to anyone with pod-read permission, it is in `/proc/<pid>/environ` so any
process in the container can read it, and it turns up in some crash dumps.

I prefer a mounted Secret volume read with `spring.config.import=configtree:/run/secrets/...`.
Each key becomes a file, the file name is the property name, and the contents are the value —
so the Kubernetes Secret's keys are your Spring property names with no glue code. It is not in
the environment, not in `describe`, and file permissions apply. For high-value credentials, a
short-lived credential fetched at startup with the pod's workload identity is better still,
because then no static secret exists to leak.

For 'how do I know it has not leaked' I would enumerate the exposure surfaces rather than
assert confidence, because in Java there are more of them than people expect. Image layers —
and layers are additive, so a later `RUN rm` does not remove a secret copied in an earlier
step; a CI scan over the saved layers catches that. Log lines — the usual cause is a
`toString()` on a `@ConfigurationProperties` class, which for a record is generated for you and
prints everything, so I override it. Actuator — `/env` and `/configprops` will happily return
configuration, so `show-values=never`, the management endpoints on a separate port, and an
explicit `exposure.include` rather than a wildcard. And exception messages, which sometimes
include a connection URL with credentials in it.

I would also make a missing secret a loud failure: `@ConfigurationProperties` as a record with
`@Validated` and `@NotBlank`, so an absent value is a startup failure naming the property
rather than a 401 from the payment gateway at peak.

And I would rotate on a schedule regardless, because rotation bounds the blast radius of a leak
I have not found, and 'I am confident there is no leak' is not a control."

**WHAT SEPARATES THEM**

Ranking the mechanisms with the reason for each ranking; knowing `configtree` exists; treating
"has it leaked" as an enumeration of surfaces rather than a feeling; knowing the additive-layer
and generated-`toString` traps specifically; making absence a loud failure; and treating
rotation as a control rather than a chore.

**FOLLOW-UP:** *"A secret was printed at DEBUG six months ago. What now?"* — Rotate first,
because the value must be assumed compromised and rotation is the only action that actually
helps. Then determine the exposure: who could read that log store, over what retention, and
whether it was exported anywhere. Then prevention that scales — the `toString` override, a
redacting log filter, and a CI check — rather than "we told the team to be careful", which
prevents nothing. And write it up per Topic 133 with contributing factors, because the
underlying factor is almost always that the properties class printed itself by default.

---

### Q3 — "Your pods terminate in exactly 30 seconds every time, regardless of load. Why?"

**MID-LEVEL ANSWER**

"Thirty seconds is the default `terminationGracePeriodSeconds`, so it sounds like shutdown is
taking too long and Kubernetes is killing it. I'd increase the grace period or speed up
shutdown."

**SENIOR ANSWER**

"The *exactly* is the diagnosis. A real drain varies with load — sometimes there are ten
in-flight requests, sometimes none. A duration that is constant to the second is not a drain
finishing; it is a timer expiring. So the application is doing nothing at all in response to
`SIGTERM`, and after the grace period it gets `SIGKILL`.

The most likely cause is a shell-form `ENTRYPOINT`. Docker runs `/bin/sh -c "java -jar ..."`,
so the shell is PID 1 and the JVM is its child. `SIGTERM` goes to the shell, many shells do not
forward it, and the JVM never learns it should stop. The confirming evidence is in the logs:
there will be no shutdown lines whatsoever — not 'Commencing graceful shutdown', nothing. Logs
that simply stop, rather than logs that show a shutdown attempt, is the signature.

I would confirm with `docker exec <container> ps -o pid,comm` — PID 1 must be `java`, not `sh`.
The fix is exec form, `ENTRYPOINT ["java", ...]`. If a shell is genuinely needed to expand an
environment variable, then `exec java ...` inside it so the shell is replaced, or better, use
`JAVA_TOOL_OPTIONS`, which the JVM reads itself and which needs no shell.

The other candidates, ruled out by the same evidence: if a signal handler were running and
overrunning, we would see shutdown log lines and a duration that varied. If the process were
ignoring `SIGTERM` deliberately, same thing. And if it were an application deadlock during
shutdown, a thread dump taken during the window would show it — which is worth doing anyway
before concluding, because it costs one `jcmd` and rules out a whole branch.

Raising the grace period, incidentally, would make it worse: termination would take exactly
sixty seconds instead of thirty, and every deploy would be slower with the same requests
dropped."

**WHAT SEPARATES THEM**

Reading "exactly" as the key evidence; distinguishing a timer from a drain; naming the PID 1
mechanism and its log signature; giving the confirming command; giving the correct fix
including the `exec` and `JAVA_TOOL_OPTIONS` variants; ruling out alternatives with evidence;
and noticing that the mid-level fix makes it worse.

**FOLLOW-UP:** *"You fix it and now termination takes two seconds. Is that good?"* — Not
necessarily. Two seconds means nothing was in flight, or the drain is not waiting for things it
should. I would check under load: with the Topic 65 generator running, termination should take
roughly as long as the longest in-flight unit of work. A shutdown that is instant under load is
a shutdown that is discarding something — most often a thread pool's queue, or a Kafka
container stopping without committing offsets.

---

### Q4 — "How does configuration get into a Spring Boot pod, and what wins?"

**MID-LEVEL ANSWER**

"From `application.yml` in the jar, overridden by profile-specific files, and environment
variables override those. We use ConfigMaps for config and Secrets for credentials."

**SENIOR ANSWER**

"There is a defined precedence chain, and the operative part in a container is: command-line
arguments beat `SPRING_APPLICATION_JSON`, which beats environment variables, which beat
`spring.config.import` sources like a config tree, which beat profile-specific files outside
the jar, which beat anything inside the jar. Files in the jar are defaults and nothing else.

Relaxed binding is what makes environment variables usable and is also what makes them
dangerous. `spring.datasource.url` maps to `SPRING_DATASOURCE_URL` — uppercase, dots to
underscores. That is why an env var somebody set once to test something silently overrides a
reviewed YAML file, and why 'works locally, not in the cluster' so often turns out to be an
`envFrom` on a ConfigMap nobody remembered.

For which mechanism to use where: non-secret configuration through a ConfigMap, either as env
vars or a mounted file. Secrets through a mounted Secret volume read as a config tree, because
that keeps them out of the environment and out of `kubectl describe`. And I would validate the
required ones with `@ConfigurationProperties` plus `@Validated`, so a missing value is a
startup failure naming the property rather than a runtime failure somewhere far away.

The operational point people miss is that **there is no reload**. A ConfigMap change does not
change a running JVM's `Environment` — even a mounted file that the kubelet updates is not
re-read, because `@ConfigurationProperties` binding happened at startup. So the way you change
configuration in production is a rolling deploy, which is why configuration and deploy
behaviour are genuinely one topic. Spring Cloud's `@RefreshScope` can do live refresh, but it
adds a bean-proxying layer with its own semantics, and I would want a specific reason before
introducing it.

And when something is not what I expect, I do not reason about it — `/actuator/env/<property>`
lists every source that defines the key in precedence order and tells me which one won."

**WHAT SEPARATES THEM**

Knowing the operative subset of the chain rather than reciting it; explaining relaxed binding as
the cause of a specific real bug; matching mechanism to value type with a reason; the "no
reload" point and its consequence; refusing `@RefreshScope` without justification; and having a
diagnostic command instead of an argument.

**FOLLOW-UP:** *"You need to change a feature flag without a deploy. Options?"* — Not
`@RefreshScope` by default. A flag that changes at runtime is not configuration; it is *state*,
and it belongs in a store built for it — a database row, Redis, or a feature-flag service — read
per evaluation with a cache and a bounded TTL. That gives an audit trail, per-environment
targeting and instant revert, none of which a ConfigMap gives you. The general rule: static
configuration is a deploy, dynamic behaviour is data.

---

### Q5 — "You enabled graceful shutdown and still see duplicate notifications after every deploy."

**MID-LEVEL ANSWER**

"Maybe the graceful shutdown timeout is too short, so the work is being cut off. I'd increase it
and check whether the notifications are idempotent."

**SENIOR ANSWER**

"Duplicates rather than losses is the informative part, because those point in opposite
directions. A loss means work was discarded. A duplicate means work completed but was not
*recorded* as completed — so something retried it. That immediately points at Kafka offsets
rather than at the HTTP drain.

`server.shutdown=graceful` is scoped precisely to the embedded web server. It does nothing for
Kafka listener containers, thread pools, scheduled tasks, or the span exporter. So the likely
sequence is: the consumer processed a record, sent the notification, and the process died before
the offset was committed. On the rebalance, another consumer in the group picks up from the last
committed offset and processes it again.

The fix has two halves and both are necessary. Mechanically, stop the listener containers
explicitly during shutdown, early, so polling stops while in-flight records complete and their
offsets commit. I would do that in a `SmartLifecycle` with a high phase number, since higher
phases stop earlier, and put the executor and relay drains at a low phase so in-flight work
finishes after acceptance has stopped.

Design-wise, at-least-once is the guarantee Kafka actually offers with an external side effect
— Topic 114's point that exactly-once covers the Kafka boundary and not an email — so the
consumer has to be idempotent regardless. A cleaner shutdown reduces duplicates; it cannot
eliminate them, because a `kill -9`, a node failure or an OOMKill will produce the same
situation with no shutdown at all. So idempotency is not the backup plan, it is the actual plan,
and graceful shutdown is the optimisation.

I would also check the two neighbours of this bug, because they share a root cause: are outbox
rows getting stuck because the relay died mid-batch, and is the notification executor's queue
being discarded on shutdown rather than drained? Both are silent, both correlate with deploys,
and neither shows up in an HTTP-only load test — which is probably why nobody noticed until the
duplicates did."

**WHAT SEPARATES THEM**

Reading duplicate-versus-loss as a diagnostic; knowing the precise scope of
`server.shutdown=graceful`; naming the offset-commit mechanism; giving phase-ordered fixes; and
correctly ranking idempotency as the guarantee and shutdown as the optimisation rather than the
reverse.

**FOLLOW-UP:** *"How would you prove the fix?"* — Run the Topic 65 load with the order-placement
scenario during a rolling deploy and count side effects, not requests: notifications sent versus
orders placed, with the idempotency key as the join. Zero duplicates *and* zero missing. Then
check the Kafka logs for `partitions revoked` at each pod termination — a clean leave rather
than a session timeout — and check that consumer lag does not spike for the session-timeout
duration at each replacement.

---

## Mental model checkpoint

1. **State the shutdown sequence in order**, and say what goes wrong if you swap the first two
   steps.

2. **Why does a `preStop` sleep fix 502s at deploy time**, when the application's graceful
   shutdown is already working correctly? What is happening during that sleep?

3. **Write the inequality** relating `terminationGracePeriodSeconds`, `preStop`,
   `spring.lifecycle.timeout-per-shutdown-phase`, and background drains. What happens when it
   is violated?

4. **`@PreDestroy` versus `SmartLifecycle.stop()`** — when does each run, which one can express
   ordering, and which do you use to stop a Kafka listener?

5. **Higher `getPhase()` means stopped earlier.** Given that, what phase should a Kafka listener
   container get relative to a connection pool, and why?

6. **`server.shutdown=graceful` is set and requests are still dropped at every deploy, with no
   shutdown lines in the log and a constant termination time.** Diagnose it in one sentence.

7. **Name four places a secret can leak in a Spring Boot service** that have no direct Node
   equivalent or a much smaller one, and the control for each.

---

## Quick reference card

### The sequence

```
1. readiness -> false        (preStop hook, before SIGTERM)
2. WAIT for propagation      (measured; the step everyone skips)
3. stop ACCEPTING            (web listener, Kafka containers, scheduler)
4. DRAIN in-flight           (HTTP requests, executor queues, outbox batch)
5. FLUSH                     (span exporter, log appenders)
6. CLOSE                     (Hikari, Hibernate, clients — bean destruction)
7. exit 0                    (before terminationGracePeriodSeconds)
```

### Properties

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=25s
management.endpoint.health.probes.enabled=true
management.endpoint.health.group.readiness.include=readinessState,db
management.endpoint.health.group.liveness.include=livenessState
management.server.port=8081
management.endpoint.env.show-values=never
management.endpoint.configprops.show-values=never
spring.config.import=optional:configtree:/run/secrets/orderflow/
```

### Pod spec essentials

```yaml
terminationGracePeriodSeconds: <preStop + drain + background + margin>
lifecycle:
  preStop: { exec: { command: ["sh","-c","sleep <measured>"] } }
strategy:
  rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
```

### Lifecycle mechanisms

| Need | Use |
|---|---|
| Release a resource this bean owns | `@PreDestroy` |
| Ordered stop | `SmartLifecycle` with an explicit `getPhase()` |
| Stop accepting work | **high** phase (stops early) |
| Finish in-flight work | **low** phase (stops late) |
| Close pools/clients | bean destruction (automatic, dependency-ordered) |

### Secret sources, best to worst

```
1. short-lived credential from a secret manager via workload identity
2. mounted Secret volume + spring.config.import=configtree:
3. environment variable from a Kubernetes Secret
4. literal in a file in the image        <- never
```

### Diagnostic commands

```bash
curl -s localhost:8081/actuator/env/<property> | jq        # where did this value come from
curl -s localhost:8081/actuator/configprops | grep -i key  # is anything unmasked
docker exec <c> ps -o pid,comm                             # PID 1 must be java
docker stop -t 30 <c> && docker logs <c> | tail -20        # do shutdown lines appear
kubectl describe pod <p> | sed -n '/Last State/,/Ready/p'
kubectl rollout status deployment/orderflow --timeout=300s
k6 run load/k6/orderflow.js       # with http_req_failed: ['count==0']
```

### Reading the evidence

| Symptom | Cause |
|---|---|
| 502s in a burst at each pod termination | No `preStop`; endpoint propagation window |
| Failures only on the slowest endpoint | Drain timeout < that endpoint's tail |
| Termination takes exactly the grace period, no shutdown logs | Shell-form `ENTRYPOINT`; JVM never got SIGTERM |
| Duplicate side effects after deploys | Kafka containers not stopped cleanly; offsets uncommitted |
| Outbox depth spikes at deploys | Relay died mid-batch; claim not released |
| Notifications silently missing | Executor queue discarded on shutdown |
| No traces covering the shutdown | Span exporter not flushed |
| Env var overriding your YAML | Relaxed binding; check `/actuator/env` |

### Gotchas checklist

- [ ] `ENTRYPOINT` is exec form; `ps` confirms `java` is PID 1.
- [ ] `preStop` sleep measured, not guessed.
- [ ] `terminationGracePeriodSeconds` strictly greater than the sum of every drain.
- [ ] Every accepting component has an explicit stop with a high phase.
- [ ] Executors have `waitForTasksToCompleteOnShutdown(true)` and a bounded await.
- [ ] Span exporter flushed before exit.
- [ ] Secrets from a config tree, not the environment; `show-values=never`.
- [ ] `toString()` overridden on every properties class holding a secret.
- [ ] `@Validated` on required properties, so absence fails startup.
- [ ] The deploy is tested under load with a threshold of exactly zero failures.

---

## When would I use this at work?

**1. The first time you look at deploy-time error rates and discover nobody has.**
Most teams have never plotted errors against deploy timestamps. Doing it once usually reveals a
sawtooth — one small burst per pod replacement — that has been quietly burning error budget for
months and being written off as noise. The fix is a `preStop` hook and two properties, and it is
one of the highest-value-per-hour changes available in this whole phase.

**2. When you add any new background component.**
A Kafka consumer, a scheduled job, an executor, a client with a connection pool — each one is a
new door, and the default behaviour of most of them at shutdown is "stop abruptly". The
reviewable question on such a pull request is: "what happens to this on `SIGTERM`, and where is
that written down?" If the answer is not in the door inventory, the pull request is not
finished.

**3. When you inherit a service and want a quick read on its operational maturity.**
Four checks, five minutes: is the `ENTRYPOINT` exec form; is there a `preStop`; is
`server.shutdown=graceful` set; and where do secrets come from. The answers tell you a great
deal about how the service will behave at 3am, and they are the first four rows of a Topic 124
readiness review.

---

## Connected topics

**Backwards:**

- **43 — configuration, profiles and properties.** The full precedence chain and
  `@ConfigurationProperties`. This topic is that knowledge applied to containers, plus the
  secrets dimension.
- **65 — the load baseline.** Supplies the p99 that every timeout here is derived from, and the
  load generator that proves the deploy drops nothing.
- **90 — executors and pool sizing.** `waitForTasksToCompleteOnShutdown` and
  `awaitTerminationSeconds` are shutdown settings that belong in the central factory.
- **109 — HikariCP.** The pool closes during bean destruction, which is correct only because
  everything holding connections drained first.
- **111 — Resilience4j.** Every downstream call needs a timeout, or a request can outlive any
  drain window you configure.
- **113 — Kafka consumer groups.** A clean container stop is what avoids a rebalance stall at
  every pod replacement.
- **114 — delivery semantics.** Why idempotency is the guarantee and clean shutdown is the
  optimisation, not the other way round.
- **115 — the outbox.** The relay must finish its claimed batch, or rows sit locked until the
  claim expires.
- **118 — metrics.** Executor queue depth and outbox depth are how you *verify* the drains
  rather than assume them.
- **119 — tracing.** The span exporter's queue must be flushed on shutdown, or you lose exactly
  the traces covering the shutdown.
- **120 — logging.** Async appenders have a queue too; a shutdown that does not flush loses the
  last log lines, which are the ones you would want.
- **121 — probes.** Readiness is the signal this topic sequences. Liveness must not be pointed
  at dependencies, or a shutdown-adjacent blip becomes a restart cascade.
- **122 — Docker and startup.** The exec-form `ENTRYPOINT` lives in that Dockerfile, and the
  new pod's startup time is the other half of a rolling deploy's duration.

**Forwards:**

- **124 — the production-readiness gate.** The door inventory, the timing budget, the
  zero-dropped-request evidence, and the secret-source table are all review rows.
- **129 — capacity and cost.** `maxSurge` means extra capacity during a rollout; deploy
  frequency times rollout duration is a real resource line.
- **130 — SLOs and error budgets.** Deploy-time errors consume the same budget as incident
  errors. A sawtooth at every deploy is a self-inflicted budget burn with a two-hour fix.
- **133 — postmortems.** "We dropped requests during the deploy" is one of the most common and
  most preventable contributing factors in a Java service.
