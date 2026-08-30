# 121 — Actuator: Health, Readiness, Liveness, and Kubernetes Semantics

## Phase: 11 — Distributed Systems & Production
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` exposes an honest readiness signal — one that goes false when the pod genuinely cannot serve an order, and stays true through every dependency blip it can survive. Plus a liveness signal that answers exactly one question: is this JVM wedged.

---

## ELI5 anchor

A restaurant has two different questions about a waiter, and confusing them ruins the
evening.

**Question 1: "Is this waiter ready to take new tables right now?"**
Maybe they are carrying six plates. Maybe the kitchen just told them the specials are
sold out. The answer might be no, temporarily. When it is no, the host stops seating
people at their section — and starts again the moment it flips back to yes. Nobody is
fired. This is **readiness**.

**Question 2: "Has this waiter passed out on the floor?"**
If yes, there is nothing to wait for. You cannot ask them to try harder. Someone has to
carry them out and bring in a fresh waiter. This is **liveness**, and the answer to a
failed liveness check is **replacement**.

Now here is the mistake that closes the restaurant.

The kitchen goes down for ninety seconds. Every waiter, honestly, answers "no" to
question 1 — they cannot serve food. Fine: the host pauses seating, the kitchen comes
back, everyone resumes.

But suppose you wired the *same* answer into question 2. Now the kitchen going down for
ninety seconds means **every waiter on the floor is declared dead and carried out at the
same instant**. Fresh waiters walk in, ask the kitchen, get the same answer, and are
carried out too. The kitchen recovers and there is nobody left standing to notice,
because everyone is stuck in the doorway putting their apron on.

That is the entire topic. A ninety-second database blip becomes a multi-minute total
outage, and it is caused by one line of YAML pointing at the wrong URL.

---

## The bridge from what you know

### What you already know, stated once so we can move past it

You know Kubernetes probes. You have written `livenessProbe`, `readinessProbe` and
`startupProbe` in Node deployments. You know:

- readiness failing removes the pod from the Service's endpoint list;
- liveness failing restarts the container;
- a startup probe suppresses the other two until the app has come up;
- `periodSeconds`, `failureThreshold`, `initialDelaySeconds`, `timeoutSeconds`.

**None of that changes in Java.** The platform is the platform. This document does not
re-teach it and will not show you a full Deployment manifest.

In Node you probably wrote the endpoints by hand:

```ts
// what you have today
app.get('/healthz', (_req, res) => res.status(200).send('ok'));
app.get('/readyz', async (_req, res) => {
  try { await pool.query('select 1'); res.status(200).send('ok'); }
  catch { res.status(503).send('not ready'); }
});
```

Two handlers, twelve lines, and complete control. That control is why the Node version
rarely goes wrong: you had to think about what went in `readyz` because you typed it.

### What is genuinely new in Spring: the endpoints are generated, not written

Spring Boot Actuator **discovers** every health check on your classpath and aggregates
them automatically. Add `spring-boot-starter-data-redis` and a Redis health check appears
in `/actuator/health` without you writing a line. Add Kafka, and one appears for Kafka.
Add a datasource, and one appears for the database.

This is the same auto-configuration mechanism as Topic 42, and it has the same two faces.
The convenient face: you get comprehensive health reporting for free. The dangerous face:
**you did not choose what is in there, and the default aggregate is "everything, ANDed
together".** One unhealthy contributor makes the whole endpoint return 503.

So the Node instinct — "point the probe at the health endpoint" — is safe when *you* wrote
the endpoint and hostile when the framework assembled it from whatever is on the
classpath. That single sentence is why this topic exists for a Spring engineer and not for
a Node one.

| | Node, hand-written | Spring Boot Actuator |
|---|---|---|
| What is checked | exactly what you typed | **every `HealthIndicator` on the classpath** |
| How it changes | when you edit the handler | **when someone adds a dependency** |
| Aggregation rule | yours | worst status wins, across all contributors |
| The failure shape | you forgot a check | **you got checks you did not want, in the probe that restarts pods** |

### The concept that has no Node equivalent: health GROUPS

Spring's answer to the problem it created is **health groups**. A group is a named subset
of the discovered contributors, exposed at its own URL with its own HTTP status mapping.

```properties
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.readiness.include=readinessState,db
```

Now `/actuator/health/liveness` reports on exactly one thing, `/actuator/health/readiness`
on two, and `/actuator/health` still shows you everything for a human. Three URLs, three
audiences, one set of underlying checks.

**Verdict: NO ANALOGUE.** In Node you got this by writing two handlers. In Spring you get
it by *subsetting an automatically assembled set*, which is a different mental operation
and the one place people most often reach for the default and lose.

### The second genuinely new thing: the JVM's startup profile

A Node process is serving requests in tens to low hundreds of milliseconds. A Spring Boot
JVM is not. It has to start a JVM, load and verify several thousand classes, scan the
classpath, evaluate auto-configuration conditions, instantiate beans, build a Hibernate
`SessionFactory`, and open a connection pool. Multiple seconds is normal and not a defect.

Two consequences you do not have in Node:

1. **A liveness probe with a short `initialDelaySeconds` can kill the pod before the
   application has ever started.** Kubernetes sees connection-refused, counts failures, and
   restarts — into the same race. That is a crash loop caused entirely by probe timing,
   with a perfectly healthy application inside. The fix is a startup probe, which you
   already know how to write; the part that is Java-specific is knowing that *seconds* is
   the normal scale and that Topic 122 is where you find out where those seconds go.

2. **A freshly started JVM is slow even after it is "ready".** The JIT has not compiled the
   hot paths yet (Topic 74). The first few hundred requests through a cold JVM run in the
   interpreter or at tier 1 and can be an order of magnitude slower than steady state.
   Caches are empty. The connection pool may be lazily filling. So a pod that just passed
   readiness and immediately receives a full share of production traffic will show a
   latency spike that is not a bug — it is warm-up. In Node you have nothing that
   corresponds to this, because V8's warm-up is comparatively cheap and your process
   footprint is smaller.

Hold that second point. It comes back in Topic 123's rolling deploys and in this
document's Trap 2.

---

## What is this?

**Spring Boot Actuator** is a module that exposes operational endpoints over HTTP (and
JMX): health, metrics, environment, loggers, thread dump, heap dump, and more. You add
one dependency:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

and it auto-configures. The pieces that matter for this topic:

**`HealthIndicator`** — one check. An interface with a single method returning a `Health`
object carrying a `Status` (`UP`, `DOWN`, `OUT_OF_SERVICE`, `UNKNOWN`, or a custom one) and
an optional detail map. Boot ships many: `DataSourceHealthIndicator`,
`RedisHealthIndicator`, `KafkaHealthIndicator` (via the admin client), `DiskSpaceHealthIndicator`,
`PingHealthIndicator`. Each is registered under a name (`db`, `redis`, `kafka`, `diskSpace`,
`ping`).

**`HealthEndpoint`** — the aggregator. It runs the contributors and combines their statuses
using a `StatusAggregator` (default: the worst status wins, in the order
`DOWN` > `OUT_OF_SERVICE` > `UP` > `UNKNOWN`). A `HttpCodeStatusMapper` then maps the
aggregate status to an HTTP code — by default `UP` → 200 and `DOWN`/`OUT_OF_SERVICE` → 503.

**Health groups** — named subsets, each with their own include/exclude list, their own
status aggregator, and their own HTTP mapping. Exposed at `/actuator/health/<group>`.

**`ApplicationAvailability`** — Boot's in-process model of the two Kubernetes questions,
independent of any indicator:

- `LivenessState` is `CORRECT` or `BROKEN`. Boot sets it to `CORRECT` when the application
  context has started. Nothing else in Boot ever sets it to `BROKEN` — **that is your job**,
  and it should almost never happen.
- `ReadinessState` is `ACCEPTING_TRAFFIC` or `REFUSING_TRAFFIC`. Boot sets it to
  `ACCEPTING_TRAFFIC` when the application is fully ready, and — this is the part that
  matters for Topic 123 — **flips it to `REFUSING_TRAFFIC` at the very start of a graceful
  shutdown**, before it stops accepting connections.

These two states surface as the `livenessState` and `readinessState` health contributors,
which is why the group definitions above include them by name.

### The two questions, precisely

Write these on the wall:

> **Liveness asks: "is this process in a state from which it can never recover on its own?"**
> The only correct remedy is a restart. If a restart would not help, the answer is not
> liveness.
>
> **Readiness asks: "can this instance successfully serve a request right now?"**
> The remedy is to stop sending it traffic and try again shortly.

Everything else in this document follows from those two sentences. A database being down
does not make your JVM unrecoverable — restarting the JVM does not fix the database, and
the JVM will be perfectly fine the moment the database returns. Therefore the database
belongs in readiness and **never** in liveness.

---

## Why does it matter?

**1. The default configuration is a loaded gun and it points at your whole fleet.**

`/actuator/health` includes every discovered indicator. Pointing a liveness probe at it
means any dependency's failure restarts every pod simultaneously. This is not a subtle
tail risk; it is the most common self-inflicted outage in Kubernetes-deployed Spring
services, and the amplification is severe: a dependency that was degraded for ninety
seconds is now an application that is down for as long as it takes six JVMs to restart,
warm up and reconnect — during which the recovering dependency gets hit by six
simultaneous cold starts opening pools and warming caches.

**2. A dishonest readiness signal is worse than no readiness signal.**

If readiness reports UP while the pod cannot actually serve — because the pool is
exhausted, or the circuit breaker to payments is open and every order fails — then the
load balancer keeps sending work to a pod that will fail it. Your error rate is now
spread evenly across the fleet instead of being concentrated where it can be shed. The
whole point of readiness is *load shedding at the right granularity*, and it only works if
the signal is true.

**3. It is the interface between your application and the platform's automation.**

Kubernetes will act on these endpoints without asking. Rolling deploys wait on readiness
(Topic 123). Horizontal autoscaling interacts with it. A `PodDisruptionBudget` is
evaluated against ready pods. You are, by configuring these three URLs, programming a
control loop that can restart your entire production estate. That deserves the same care
as any other production code path — and unlike most code paths, it has no test coverage
by default.

**4. It is a standard senior interview filter, and a fast one.**

"We point both probes at `/actuator/health`" is a sentence an interviewer can say in three
seconds that separates people who have operated a Java service from people who have
deployed one. There is no way to bluff the follow-up.

---

## Syntax breakdown

### Exposing endpoints — and the default that surprises people

```properties
# Only "health" is exposed over HTTP by default. Everything else must be listed.
management.endpoints.web.exposure.include=health,info,metrics,prometheus

# Exclusions win over inclusions.
management.endpoints.web.exposure.exclude=env,beans

# The base path. Default /actuator.
management.endpoints.web.base-path=/actuator
```

Reading that:

- `include` takes endpoint **IDs**, not paths: `health`, `info`, `metrics`, `prometheus`,
  `loggers`, `threaddump`, `heapdump`, `env`, `configprops`, `mappings`, `shutdown`.
- `*` includes everything. **Never write `*` on a service reachable from an ingress.**
  `heapdump` will hand a stranger your entire heap — every JWT, every card token in flight,
  every row you loaded. `env` and `configprops` print configuration including, depending on
  masking, credentials. `loggers` is a *write* endpoint. `shutdown` is exactly what it
  sounds like and is disabled by default for a reason.
- Being *exposed* over HTTP and being *enabled* are two different switches. An endpoint can
  be enabled (`management.endpoint.heapdump.enabled=true`, the default) but not exposed.
  Exposure is the one that decides reachability.

### Health details, and who is allowed to see them

```properties
# never (default) | when-authorized | always
management.endpoint.health.show-details=when-authorized
management.endpoint.health.show-components=when-authorized
management.endpoint.health.roles=ACTUATOR_ADMIN
```

The default `never` means `/actuator/health` returns only `{"status":"UP"}` to an
unauthenticated caller. That default is correct and people override it to `always` while
debugging and forget. Component details leak internal hostnames, database product versions,
disk paths and free-space figures — a free reconnaissance report.

### Enabling the Kubernetes probe endpoints

```properties
management.endpoint.health.probes.enabled=true
```

This creates two groups for you: `liveness` (containing `livenessState`) and `readiness`
(containing `readinessState`), served at:

```
GET /actuator/health/liveness
GET /actuator/health/readiness
```

**Boot enables this automatically when it detects it is running on Kubernetes** (it looks
for the environment variables Kubernetes injects into every pod). Relying on auto-detection
is a bad habit: it means the endpoints exist in production and not on your laptop, so you
cannot test the thing that will restart your pods. **Set the property explicitly** and get
the same endpoints everywhere.

Note what the default groups contain: **only the availability states**. Out of the box,
readiness does *not* check your database. That default is deliberately conservative and it
is a reasonable starting point — but it is probably not what you want for readiness, which
brings us to groups.

### Health groups — the core mechanism of this topic

```properties
# LIVENESS: the JVM's own state. Nothing external. Ever.
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.liveness.show-details=never

# READINESS: the availability state plus dependencies WITHOUT WHICH THIS POD
# CANNOT SERVE ITS CORE REQUEST, and whose failure is not fleet-wide.
management.endpoint.health.group.readiness.include=readinessState,db
management.endpoint.health.group.readiness.show-details=when-authorized

# A third group for humans and dashboards: everything, no probe attached.
management.endpoint.health.group.detailed.include=*
management.endpoint.health.group.detailed.show-details=always
```

Every knob a group has:

```properties
management.endpoint.health.group.<name>.include=a,b,c   # names of contributors, or *
management.endpoint.health.group.<name>.exclude=c        # exclusions win
management.endpoint.health.group.<name>.show-details=never|when-authorized|always
management.endpoint.health.group.<name>.show-components=never|when-authorized|always
management.endpoint.health.group.<name>.status.order=DOWN,OUT_OF_SERVICE,UP,UNKNOWN
management.endpoint.health.group.<name>.status.http-mapping.out-of-service=503
management.endpoint.health.group.<name>.additional-path=server:/healthz
```

Two of those deserve a sentence each.

**`status.http-mapping`** lets a group return a different HTTP code for a status. Useful
when you want a degraded-but-serving state that does *not* pull the pod out of rotation:
map `OUT_OF_SERVICE` to 200 in the readiness group while it stays 503 in the detailed
group.

**`additional-path`** exposes a group on the **main server port** as well as the management
port. This is the answer to a real problem: you want the management port private (see
below), but the kubelet probes the container's ports directly and your ingress does not
mediate it — so both work. Where it genuinely matters is any environment where a probe or a
load-balancer health check can only reach the application port. `server:/healthz` puts the
group at `http://<app-port>/healthz`.

### A separate management port

```properties
management.server.port=9090
management.endpoints.web.exposure.include=health,info,prometheus,metrics
```

Now Actuator listens on 9090 and your API on 8080. Your ingress only routes 8080. The
kubelet and your Prometheus scraper can reach 9090 inside the cluster. This is the
single highest-value hardening step in the document and it costs one line.

The catch to know: with a separate management port, some things that "just worked" change
— the servlet filters registered for the main port (including your correlation-ID filter
from Topic 120 and the Spring Security chain from Topic 56) apply to a different server.
Verify your management port is not accidentally *unsecured* just because it is
"internal only".

### Writing a custom `HealthIndicator`

```java
package com.orderflow.payments;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

/**
 * The bean name determines the contributor name in the health report.
 * "paymentGatewayHealthIndicator" -> reported as "paymentGateway".
 * Boot strips a trailing "HealthIndicator" from the bean name.
 */
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final CircuitBreakerRegistry breakers;

    public PaymentGatewayHealthIndicator(CircuitBreakerRegistry breakers) {
        this.breakers = breakers;
    }

    @Override
    public Health health() {
        var state = breakers.circuitBreaker("payment-gateway").getState();
        return switch (state) {
            case CLOSED, HALF_OPEN -> Health.up()
                    .withDetail("breaker", state.name())
                    .build();
            // OPEN means we are already failing fast. Report it, but see the
            // Traps section before you put this contributor in the readiness group.
            case OPEN, FORCED_OPEN -> Health.status("DEGRADED")
                    .withDetail("breaker", state.name())
                    .build();
            default -> Health.unknown().withDetail("breaker", state.name()).build();
        };
    }
}
```

Three rules for writing one, all learned the hard way:

1. **It must be fast and it must have a timeout.** The health endpoint is called every few
   seconds by the kubelet, per pod. An indicator that makes a 5-second network call will
   time out the probe and mark the pod unready for reasons unrelated to the pod.
2. **It must not take a resource the request path needs.** See Trap 4 — this is the one
   that surprises people.
3. **It must never throw.** A thrown exception is turned into `DOWN` with the exception
   details attached, which may leak internals and will definitely surprise you. Catch and
   report.

For an indicator that needs to do I/O, extend `AbstractHealthIndicator` and put the work in
`doHealthCheck(Health.Builder)`; it handles the exception-to-`DOWN` conversion for you.

### Controlling the auto-discovered indicators

```properties
# Turn OFF every auto-configured indicator, then opt back in explicitly.
management.health.defaults.enabled=false
management.health.db.enabled=true
management.health.diskspace.enabled=false
management.health.redis.enabled=false
```

`defaults.enabled=false` is worth considering for a production service. It converts
"whatever the classpath decided" into "the list I chose", which is the same argument as
explicit `@Bean` definitions over component scanning in Topic 36. The cost is that adding a
dependency no longer gives you its health check for free — which is precisely the point.

### Driving availability state from your own code

```java
package com.orderflow.availability;

import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.LivenessState;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

@Component
public class AvailabilityControl {

    private final ApplicationEventPublisher events;

    public AvailabilityControl(ApplicationEventPublisher events) {
        this.events = events;
    }

    /** Stop taking traffic without dying. Reversible. */
    public void refuseTraffic() {
        AvailabilityChangeEvent.publish(events, this, ReadinessState.REFUSING_TRAFFIC);
    }

    public void acceptTraffic() {
        AvailabilityChangeEvent.publish(events, this, ReadinessState.ACCEPTING_TRAFFIC);
    }

    /**
     * Declare the JVM unrecoverable. This asks Kubernetes to KILL this container.
     * There is almost never a good reason to call this. See the Traps section.
     */
    public void declareBroken() {
        AvailabilityChangeEvent.publish(events, this, LivenessState.BROKEN);
    }
}
```

And to react to the state rather than set it:

```java
@EventListener
public void onReadinessChange(AvailabilityChangeEvent<ReadinessState> event) {
    log.info("readiness now {}", event.getState());
}
```

That listener is genuinely useful: it gives you a log line, with a correlation-free but
timestamped record, at exactly the moment the pod left or rejoined rotation. During a
rolling deploy investigation (Topic 123) that line is the anchor for the timeline.

### The shape of a health response

*Illustration of the format, not captured output. `<n>` and `xxx` are placeholders.*

`GET /actuator/health` with `show-details=always`:

```json
{
  "status": "UP",
  "components": {
    "db":            { "status": "UP", "details": { "database": "PostgreSQL", "validationQuery": "isValid()" } },
    "diskSpace":     { "status": "UP", "details": { "total": <n>, "free": <n>, "threshold": <n>, "exists": true } },
    "livenessState": { "status": "UP" },
    "ping":          { "status": "UP" },
    "readinessState":{ "status": "UP" },
    "redis":         { "status": "UP", "details": { "version": "xxx" } }
  },
  "groups": [ "detailed", "liveness", "readiness" ]
}
```

`GET /actuator/health/liveness` — note how much smaller it is, and that HTTP 200 is the
signal that matters, not the body:

```json
{ "status": "UP" }
```

`GET /actuator/health/readiness` when the database is down, returned with **HTTP 503**:

```json
{
  "status": "DOWN",
  "components": {
    "db":             { "status": "DOWN", "details": { "error": "xxx" } },
    "readinessState": { "status": "UP" }
  }
}
```

Read that last one carefully, because it contains the whole lesson: `readinessState` is
`UP` — Boot thinks the application is fine — and the *group* is `DOWN` because one
contributor is. The group aggregation is what turns a dependency failure into a probe
failure. Which probe you attached that group to decides whether the outcome is "stop
sending traffic" or "kill the container".

---

## Example 1 — minimal

The smallest configuration that gets the two questions right. Four properties.

`src/main/resources/application.properties`:

```properties
# 1. Actuator on its own port, so the ingress never routes to it.
management.server.port=9090

# 2. Expose only what you need. health is exposed by default; be explicit anyway.
management.endpoints.web.exposure.include=health,info,prometheus

# 3. Create the probe endpoints EVERYWHERE, not just when Boot detects Kubernetes.
management.endpoint.health.probes.enabled=true

# 4. Define the two groups explicitly. This is the line that prevents the outage.
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.readiness.include=readinessState,db
```

Run it and interrogate the three endpoints by hand:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:9090/actuator/health/liveness
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:9090/actuator/health/readiness
curl -s http://localhost:9090/actuator/health | jq .
```

**What to look for:** the HTTP status codes, not the bodies. Kubernetes only reads the
status code. A 200 means pass; anything else means fail.

Now stop the database and repeat both `curl`s. Liveness must still return 200. Readiness
must return 503. **If liveness changes, your configuration is wrong and this is the
two-minute test that proves it.** Run this test on every service you own; it takes longer
to read this paragraph than to run it.

And the probe configuration this pairs with — shown once, minimally, because you already
know this part:

```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 9090 }
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
```

The only Java-specific commentary: the startup probe exists because a JVM takes seconds to
start, and its budget (`failureThreshold` times `periodSeconds`) must exceed your measured
worst-case startup — which you will measure properly in Topic 122, not guess at here.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline:

- **6 pods**, 400 order placements/second sustained, 100k products, 1M orders.
- Dependencies: **Postgres** (single primary — every pod talks to the same instance),
  **Redis** (product catalogue cache, Topic 110), **Kafka** (outbox relay publishes, Topic
  115), and an external **payment gateway** behind a Resilience4j circuit breaker
  (Topic 111).
- HikariCP pool of 20 per pod, tuned in Topic 109. Six pods times 20 is 120 connections
  against one Postgres — already near the sensible ceiling.
- The service has three request classes: order placement (needs Postgres and the gateway),
  catalogue reads (needs Postgres, prefers Redis), and order history reads (needs Postgres).

### The decision that has to be made, dependency by dependency

This is the actual work of the topic. For each dependency, three questions:

1. If it is down, can this pod serve *any* useful request?
2. Would removing this pod from rotation *help* — is there another pod that could serve?
3. Would restarting this pod help?

| Dependency | In liveness? | In readiness? | Why |
|---|---|---|---|
| **Postgres** | **never** | **yes** | Nothing works without it. But it is shared: if it is down, *all six* pods go unready simultaneously and the Service has no endpoints. That is honest — there is nothing to route to — and it is very different from all six restarting. |
| **Redis** | never | **no** | The cache is an optimisation. Without it, catalogue reads fall through to Postgres and get slower. A degraded pod is better than no pod. Putting Redis in readiness converts a cache outage into a total outage. |
| **Kafka** | never | **no** | Only the outbox relay publishes to it. If Kafka is down the relay retries and the outbox table grows — by design (Topic 115). HTTP requests are unaffected. Kafka in readiness means a Kafka blip stops order placement, which is precisely backwards. |
| **Payment gateway** | never | **no**, and see below | If the gateway is down, order placement fails but catalogue and history reads work. More importantly the gateway is *external and shared*: all six pods see it as down at once, so shedding traffic sheds it to nowhere. |
| **Disk space** | never | **no** | The default `diskSpace` indicator will mark you DOWN when free space drops below a threshold. On a container with an ephemeral filesystem this is usually noise. Decide deliberately; do not inherit it. |
| **The JVM itself** | **yes** — this is all liveness is | via `readinessState` | `livenessState` is `BROKEN` only if you publish it. |

The rule that generates that whole table:

> **Put a dependency in readiness only if (a) this pod genuinely cannot serve without it,
> and (b) some other pod plausibly could.** If every pod shares the dependency, marking them
> all unready does not shed load anywhere — it just replaces "some requests fail with a
> useful error" with "all requests fail with a 503 from the ingress and you have lost your
> logs' request context".

Postgres passes (a) and fails (b) — and is still included, because when Postgres is down
there is genuinely nothing to serve and the honest 503 is better than a pod accepting
orders it cannot persist. That is a judgement call and you should be able to argue it both
ways; the Mental Model Checkpoint asks you to.

### The configuration

```properties
# ---------- Actuator surface ----------
management.server.port=9090
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoint.health.show-details=when-authorized
management.endpoint.health.show-components=when-authorized

# ---------- turn off the buffet, order from the menu ----------
management.health.defaults.enabled=false
management.health.db.enabled=true
management.health.redis.enabled=true
management.health.diskspace.enabled=false

# ---------- probes ----------
management.endpoint.health.probes.enabled=true

# LIVENESS: the JVM only. If this is ever DOWN, a restart is the correct remedy.
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.liveness.show-details=never

# READINESS: can this pod serve an order right now?
management.endpoint.health.group.readiness.include=readinessState,db
management.endpoint.health.group.readiness.show-details=never

# DETAILED: for humans, dashboards and the on-call runbook. No probe points here.
management.endpoint.health.group.detailed.include=*
management.endpoint.health.group.detailed.show-details=always

# The db indicator takes a POOL CONNECTION. Cache the result so probe traffic
# does not compete with request traffic. See Trap 4.
management.endpoint.health.cache.time-to-live=5s
```

### The `orderflow`-specific indicator that is worth writing

The default `db` indicator answers "can I get a connection and validate it". It does not
answer the question that actually predicts failure at this baseline, which is: **is the
pool exhausted?** Topic 109 taught you that pool exhaustion is how this service dies. A
readiness signal that goes false when the pool has been fully saturated for a sustained
period is a genuinely useful load-shedding signal — and one that differs per pod, which
satisfies condition (b) above.

```java
package com.orderflow.availability;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.time.Instant;

/**
 * Reports DOWN only when the pool has been fully saturated with waiters
 * CONTINUOUSLY for longer than a grace window.
 *
 * Two design choices worth defending:
 *  - it reads Hikari's MXBean counters and takes NO connection, so it cannot
 *    make the problem it is measuring worse;
 *  - it requires SUSTAINED saturation, because a momentary spike is normal at
 *    400 rps and a probe that flaps is worse than no probe.
 */
@Component
public class ConnectionPoolHealthIndicator implements HealthIndicator {

    private static final Duration GRACE = Duration.ofSeconds(20);

    private final HikariDataSource dataSource;
    private volatile Instant saturatedSince;

    public ConnectionPoolHealthIndicator(HikariDataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        var pool = dataSource.getHikariPoolMXBean();
        if (pool == null) {
            return Health.unknown().withDetail("reason", "pool not started").build();
        }
        int idle = pool.getIdleConnections();
        int waiting = pool.getThreadsAwaitingConnection();
        boolean saturated = idle == 0 && waiting > 0;

        Instant since = saturatedSince;
        if (!saturated) {
            saturatedSince = null;
        } else if (since == null) {
            saturatedSince = Instant.now();
        }

        Instant startedAt = saturatedSince;
        boolean sustained = startedAt != null
                && Duration.between(startedAt, Instant.now()).compareTo(GRACE) > 0;

        var builder = sustained ? Health.down() : Health.up();
        return builder
                .withDetail("active", pool.getActiveConnections())
                .withDetail("idle", idle)
                .withDetail("waiting", waiting)
                .withDetail("total", pool.getTotalConnections())
                .build();
    }
}
```

Add it to readiness — and **only** readiness:

```properties
management.endpoint.health.group.readiness.include=readinessState,db,connectionPool
```

Note the honesty required here: if all six pods saturate simultaneously (because Postgres
itself is slow), all six go unready and you have an outage. This indicator earns its place
only if pod-level saturation can happen independently — which it can, because a single pod
can get a disproportionate share of expensive requests, or hit the Topic 55
transaction-holding-a-connection bug on one code path. Write down that reasoning in the
Topic 124 review; an indicator you cannot justify is an indicator that will cause an
incident.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — both probes point at `/actuator/health`

**This is the trap. Everything else in this section is secondary.**

**Wrong:**

```yaml
livenessProbe:
  httpGet: { path: /actuator/health, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /actuator/health, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
```

Nothing about that looks wrong. It is the configuration in a hundred blog posts and it is
what a well-meaning engineer writes on day one.

**Exact symptom:** Postgres has a thirty-second interruption — a failover, a `VACUUM FULL`,
a network partition, a connection storm. Then:

- **All six pods restart within a few seconds of each other**, because all six probes fail
  at the same time for the same reason. `kubectl get pods` shows `RESTARTS` incrementing in
  lockstep across the whole Deployment.
- `kubectl describe pod <name>` shows, in the Events section, `Liveness probe failed:
  HTTP probe failed with statuscode: 503` followed by `Killing container ... Container
  failed liveness probe, will be restarted`.
- The container's last state is `Terminated` with **exit code 137** — SIGKILL. Not an
  application error. Nothing in your application logs explains it, because from the JVM's
  point of view it was simply murdered. This is the detail that sends people looking for
  an OOM for an hour: 137 is also what an OOMKill looks like, and the way to tell them
  apart is `kubectl describe pod` showing `Reason: OOMKilled` versus the liveness event.
- Then it gets worse. The six restarting JVMs take seconds each to start, and when they
  come up they all open connection pools against the just-recovered Postgres at the same
  instant — 120 connections arriving simultaneously. That can push Postgres back over the
  edge, failing the probes again. **The restart loop sustains itself after the original
  cause is gone.** `kubectl get pods -w` shows the pods cycling `Running` →
  `CrashLoopBackOff`.
- Total outage duration is several minutes for a dependency blip of thirty seconds. Your
  incident timeline shows the database recovering long before the application does, which
  is the fingerprint of this bug.

**Root cause:** `/actuator/health` is the *aggregate* of every discovered contributor,
ANDed. When any contributor is DOWN, the endpoint returns 503. A liveness probe interprets
503 as "this process is unrecoverable, kill it". But a database outage does not make the
JVM unrecoverable — the JVM is fine, and it will work perfectly the moment the database
returns. **Restarting is not merely useless here; it is actively harmful**, because it
destroys warm JIT state (Topic 74), empties caches, and stampedes the recovering
dependency with pool initialisation.

**Fix:**

```properties
management.endpoint.health.probes.enabled=true
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.readiness.include=readinessState,db
```

```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 9090 }
```

**The test that proves the fix, and that should be a standing check:** stop the database,
`curl` both endpoints, assert liveness is 200 and readiness is 503. Twenty seconds of work.
Put it in your integration test suite with Testcontainers (Topic 59): start the app, stop
the Postgres container, assert the two status codes. Now a future engineer cannot
reintroduce this by editing a properties file.

**The generalised rule to carry forward:** *anything that a restart cannot fix must not be
in the liveness probe.* Apply it to every dependency, one at a time, and write down the
answer. That table is a deliverable in Topic 124.

---

### Trap 2 — liveness kills the pod before the JVM has finished starting

**Wrong:**

```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
# no startupProbe
```

The liveness group is correct. The timing is not.

**Exact symptom:** the pod never reaches `Running`+ready. `kubectl get pods` shows
`RESTARTS` climbing from the very first attempt and the pod entering `CrashLoopBackOff`.
`kubectl describe pod` Events shows repeated `Liveness probe failed: Get
"http://...:9090/actuator/health/liveness": dial tcp ...: connect: connection refused`.

The tell that distinguishes this from every other crash loop: **the application logs stop
partway through startup, at a different point each time**, and there is no exception. You
see the Spring banner, some auto-configuration, maybe the Hibernate dialect line — and then
nothing. That truncation is the JVM being SIGKILLed mid-startup. It looks like a hang or a
deadlock in initialisation, and engineers lose hours reading `HikariPool` startup code
before someone notices the probe budget.

It also appears *only under specific conditions*, which makes it worse: it works on a
laptop and in a quiet staging cluster, and fails in production where the node is busier,
the image layer cache is cold, or the database is slower to hand out the first connections.
It can also appear suddenly after an unrelated change that adds two seconds to startup —
one new dependency with a few hundred more classes to load.

**Root cause:** `initialDelaySeconds: 5` plus three failures at five seconds each gives the
JVM twenty seconds, total, to be answering HTTP on the management port. A Spring Boot
service with Hibernate, a connection pool, and a real classpath frequently needs more,
especially on a cold page cache. Kubernetes is not distinguishing "not started yet" from
"wedged"; it cannot, from connection-refused alone. That distinction is exactly what the
startup probe exists to provide.

**Fix, in two parts, and the second part matters more:**

Part one, the immediate fix:

```yaml
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  periodSeconds: 5
  failureThreshold: 24          # 24 x 5s = 120s budget, generously above measured worst case
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  periodSeconds: 10
  failureThreshold: 3           # only starts counting AFTER the startup probe passes
```

Part two, the part that separates a fix from a workaround: **do not pick 120 from
intuition.** Measure your startup time, decompose it, and set the budget from the measured
worst case with margin. That is Topic 122's entire subject, and this trap is the reason it
is a differentiator topic rather than a footnote. A team that "fixes" this by setting
`failureThreshold: 200` has hidden a startup regression rather than found one — and the
next incident is a deploy that takes fifteen minutes to roll because every pod is slow to
start and nobody noticed it creeping up.

**The related, sneakier version of this trap:** a *readiness* probe budget shorter than
startup does not crash-loop, it just makes rolling deploys stall or fail. You will meet
that in Topic 123.

---

### Trap 3 — a shared, non-sheddable dependency in the readiness group

**Wrong:**

```properties
management.endpoint.health.group.readiness.include=readinessState,db,redis,kafka,paymentGateway
```

This looks *more* thorough than the correct configuration, and thoroughness is the
instinct that causes it. "Readiness means we can serve requests, and we can't really serve
requests without Kafka, so include it."

**Exact symptom:** the Kafka broker has a rolling restart. `orderflow` HTTP traffic is
entirely unaffected at the application level — order placement writes to the outbox table
and returns, exactly as Topic 115 designed. But:

- All six pods report readiness 503 simultaneously.
- The Service's endpoint list empties. `kubectl get endpoints orderflow` shows no
  addresses.
- Every request now fails at the ingress with a 503 that never reaches your application —
  so it produces **no application log line, no metric, no trace**. Your dashboards show
  traffic simply stopping. Topic 118's RED metrics go to zero requests, which reads like a
  traffic drop rather than an outage, and you may not even alert on it.
- `kubectl describe pod` shows `Readiness probe failed: HTTP probe failed with statuscode:
  503`, and the pods are otherwise perfectly healthy and idle.

You have converted "a background publisher is retrying" into "the API is down", and you
have blinded your own observability while doing it.

The `paymentGateway` inclusion has its own flavour of the same bug: the gateway is external
and shared, so when its circuit breaker opens, it opens on all six pods within seconds
(Topic 111). All six go unready. But catalogue reads and order-history reads did not need
the gateway at all, and now they fail too.

**Root cause:** readiness is a *load-shedding* signal. Shedding only helps if there is
somewhere for the load to go. A dependency shared identically by every replica fails on
every replica at once, so marking them all unready sheds the load into a 503 at the
ingress. You have not protected anything; you have removed your own ability to serve the
requests that did not need the failed dependency, and removed your own telemetry.

**Fix:** apply the two-condition rule from Example 2. A dependency belongs in readiness only
if (a) this pod cannot serve its core request without it, **and** (b) another pod plausibly
could. Kafka fails (a) — HTTP works fine. Redis fails (a) — it degrades. The payment gateway
fails (b) and partially fails (a).

Report all of them in the `detailed` group so a human can see the state, and **alert on
them** (Topic 118) rather than probing on them. The distinction to internalise:

> **A probe is an instruction to the platform. A metric is information for a human.**
> "Kafka is down" is information. "Stop sending this pod traffic" is an instruction. Do not
> encode the first as the second.

**The partial-degradation question this raises**, and the honest answer: if order placement
is genuinely broken but reads work, the right response is not readiness. It is to fail the
order-placement endpoint with a clear error (Topic 46's `ProblemDetail`), let the circuit
breaker fail fast (Topic 111), and alert. Readiness has one bit; your failure modes have
more than two states. Do not try to encode a rich state in one bit.

---

### Trap 4 — the health check competes with request traffic for the connection pool

**Wrong:** the default `db` indicator in the readiness group, with no cache, on a service
whose failure mode is pool exhaustion.

```properties
management.endpoint.health.group.readiness.include=readinessState,db
# no management.endpoint.health.cache.time-to-live
```

**Exact symptom, and this one is genuinely nasty because it is a positive feedback loop:**

Load rises. The Hikari pool saturates — Topic 109's shape, or Topic 55's
HTTP-call-inside-a-transaction. Now `DataSourceHealthIndicator` runs, tries to borrow a
connection, and **queues behind the request traffic**. It waits up to
`spring.datasource.hikari.connection-timeout` (default 30 seconds) and the probe times out
first (`timeoutSeconds`, commonly 1–3).

The pod is marked unready and removed from the Service. Its share of traffic moves to the
five remaining pods. Their pools saturate sooner. Their probes time out. Pods drop out one
by one, each departure accelerating the next, until the Service has no endpoints — while
every JVM is alive, and every pod would have served *some* traffic successfully.

The observable fingerprint, in order: rising `hikaricp_connections_pending` (Topic 118's
USE metrics), then readiness failures, then a **staircase** in the ready-pod count going
down, with the interval between steps shrinking. That shrinking interval is the signature
of a feedback loop, and it is how you distinguish this from six pods failing independently.

`kubectl describe pod` says `Readiness probe failed: Get ...: context deadline exceeded
(Client.Timeout exceeded while awaiting headers)` — a **timeout**, not a 503. That
distinction is diagnostic: a 503 means the endpoint answered and said DOWN; a timeout means
the endpoint could not answer at all, which points at resource starvation rather than a
dependency being down.

**Root cause:** the health check is not free and it is not isolated. It consumes the exact
resource whose scarcity it is trying to report on. Under the conditions where the signal
matters most, the signal itself becomes unavailable — and its unavailability is
interpreted as failure.

**Fix, layered:**

1. **Cache the health result** so probe frequency does not drive pool usage:
   ```properties
   management.endpoint.health.cache.time-to-live=5s
   ```
   With a 5-second TTL and a 5-second probe period across three probe types, you go from
   several pool borrows per pod per period to roughly one.
2. **Prefer an indicator that reads counters over one that takes a resource.** The
   `ConnectionPoolHealthIndicator` in Example 2 reads Hikari's MXBean and borrows nothing.
   It cannot make the problem worse.
3. **Give the health check its own tiny pool** if you truly need to execute SQL for
   readiness: a second `DataSource` with `maximumPoolSize=1` or 2, used only by the
   indicator. It costs one connection per pod and makes the signal independent of request
   load. This is worth it on a service where pool exhaustion is the known failure mode.
4. **Set `timeoutSeconds` on the probe deliberately** and keep it well under
   `periodSeconds`, so a slow check fails fast rather than overlapping with the next probe.
5. **Require sustained failure, not instantaneous.** `failureThreshold: 3` on readiness
   means a single unlucky check does not remove the pod. Flapping readiness is its own
   incident class: it produces load-balancer churn and connection resets while everything
   is technically working.

**The general principle, worth stating because it recurs:** *a health check must be cheaper
than the work it is reporting on, and must not consume the resource it is measuring.* Same
family as Topic 118's rule that instrumentation must not be the bottleneck.

---

### Trap 5 — exposing the whole Actuator surface, or leaking details

**Wrong:**

```properties
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
# and no separate management port, so everything is on 8080 behind the ingress
```

Written during development so the dashboards "just work", and never revisited.

**Exact symptom:** none, in the sense that everything works perfectly. That is what makes
it a trap — there is no failure to notice. The symptoms are all discovered externally:

- A penetration test or a bug-bounty report showing `GET /actuator/heapdump` returns a
  multi-hundred-megabyte file. **That heap contains every JWT in flight, every session, every
  card token, every row currently loaded, and every string ever interned.** It is the single
  most damaging endpoint in the framework.
- `GET /actuator/env` and `/actuator/configprops` return configuration. Boot masks keys
  matching common secret-ish patterns, but the masking is name-based and heuristic. A
  property named `orderflow.gateway.credential` may or may not be masked; a
  `spring.datasource.url` containing a username certainly is not the kind of thing you want
  public.
- `GET /actuator/mappings` is a complete map of your API surface, including endpoints not
  in your public documentation.
- `POST /actuator/loggers/com.orderflow.payments` with `{"configuredLevel":"DEBUG"}` — a
  **write** endpoint reachable by anyone — turns on debug logging in production. Combined
  with Topic 120's Trap 4, an attacker can make your service log its own request bodies into
  a system with a 30-day retention.
- `GET /actuator/threaddump` gives stack traces revealing internal class names, library
  versions, and thread pool structure — free reconnaissance for choosing an exploit.

**Root cause:** `*` is a wildcard over a set that grows when you add dependencies. You did
not audit the set; you cannot, because it changes under you. And `show-details=always`
turns the health endpoint from a one-bit signal into a structured report about your
infrastructure.

**Fix:**

```properties
management.server.port=9090                                   # not routed by the ingress
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoint.health.show-details=when-authorized
management.endpoint.health.show-components=when-authorized
management.endpoint.health.roles=ACTUATOR_ADMIN
```

Then, in the Spring Security configuration (Topic 57), require authentication on the
management port for everything except the two probe groups — the kubelet cannot present
credentials, so those two specific paths stay open, and they return one bit each with
`show-details=never`.

**Verify rather than believe:**

```bash
curl -s http://localhost:9090/actuator | jq -r '._links | keys[]'
```

That lists every exposed endpoint. Read the list. If anything on it surprises you, that is
the finding. Do this on every service before it goes to production, and put the resulting
list in the Topic 124 review.

---

## Hands-on proof

Every command here is one **you** run. No output is reproduced. What follows is the exact
configuration, the exact command, what to look for, and how to read each possible result.

### Setup

```bash
mkdir -p ~/java-lab/121 && cd ~/java-lab/121
java --version

curl https://start.spring.io/starter.zip \
  -d dependencies=web,actuator,data-jpa,postgresql \
  -d javaVersion=21 \
  -d groupId=com.orderflow -d artifactId=probe-lab \
  -d type=maven-project -o probe-lab.zip && unzip probe-lab.zip -d probe-lab
cd probe-lab
```

Run a Postgres you can stop and start at will:

```bash
docker run -d --name orderflow-pg \
  -e POSTGRES_PASSWORD=orderflow -e POSTGRES_DB=orderflow \
  -p 5432:5432 postgres:16
```

`src/main/resources/application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/orderflow
spring.datasource.username=postgres
spring.datasource.password=orderflow
spring.jpa.hibernate.ddl-auto=none

management.server.port=9090
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always
management.endpoint.health.probes.enabled=true
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.readiness.include=readinessState,db
management.endpoint.health.group.detailed.include=*
```

### Proof 1 — see what is actually in your health endpoint

```bash
./mvnw spring-boot:run &
sleep 20    # or watch the log for "Started"
curl -s http://localhost:9090/actuator/health | jq -r '.components | keys[]'
curl -s http://localhost:9090/actuator/health | jq -r '.groups[]'
```

**What to look for:** the list of contributor names, and the list of groups.

| What you see | What it means |
|---|---|
| `db`, `diskSpace`, `livenessState`, `ping`, `readinessState` | The auto-discovered set for this classpath. **You did not choose any of these.** That is the point of the exercise. |
| More entries than you expected (e.g. `redis`, `kafka`, `mail`) | Auto-configuration found those dependencies. If any of them are in a group a probe points at, you have inherited a failure mode you never designed. |
| `groups` lists `detailed`, `liveness`, `readiness` | Your group definitions took effect. |
| `groups` is absent or empty | Your group properties are misspelled. The property path is long and easy to get wrong — check `management.endpoint.health.group.<name>.include`, singular `endpoint`, singular `group`. |
| `components` is absent, only `{"status":"UP"}` | `show-details` is `never`. That is the production-correct setting; set it to `always` temporarily for this exercise only. |

### Proof 2 — the two probes disagree, and that is correct

With the app running:

```bash
curl -s -o /dev/null -w 'liveness  %{http_code}\n' http://localhost:9090/actuator/health/liveness
curl -s -o /dev/null -w 'readiness %{http_code}\n' http://localhost:9090/actuator/health/readiness

docker stop orderflow-pg
sleep 15

curl -s -o /dev/null -w 'liveness  %{http_code}\n' http://localhost:9090/actuator/health/liveness
curl -s -o /dev/null -w 'readiness %{http_code}\n' http://localhost:9090/actuator/health/readiness
curl -s http://localhost:9090/actuator/health/readiness | jq .

docker start orderflow-pg
```

**What to look for:** four status codes.

| What you see | What it means |
|---|---|
| Before: both 200. After: liveness 200, readiness 503 | **Correct.** This is the whole configuration working. The JVM is fine; it just cannot serve. |
| After: both 503 | Your liveness group includes something other than `livenessState`. Print it: `curl -s localhost:9090/actuator/health/liveness \| jq .` with details on, and look at the components. |
| After: both 200 | The `db` contributor is not in your readiness group, or Hikari has not noticed the database is gone yet. Hikari does not probe idle connections continuously; give it longer, or make a request that needs the database first. |
| Readiness takes many seconds to return 503 | The indicator is waiting on `connection-timeout` (default 30s). This is Trap 4 in miniature. Lower `spring.datasource.hikari.connection-timeout` and observe the response time change. |
| Readiness body shows `readinessState: UP` and `db: DOWN` | The exact shape from the Syntax section. The **group** is DOWN; Boot's own readiness state is fine. Understanding that distinction is the point. |

### Proof 3 — the availability state is separate from the indicators

Add the `AvailabilityControl` bean and an endpoint that calls `refuseTraffic()`.

```bash
curl -X POST http://localhost:8080/admin/refuse-traffic
curl -s http://localhost:9090/actuator/health/readiness | jq .
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:9090/actuator/health/liveness
```

| What you see | What it means |
|---|---|
| Readiness 503 with `readinessState: OUT_OF_SERVICE`, liveness still 200 | You can take a pod out of rotation from inside the application, with the database perfectly healthy. This is the mechanism Boot itself uses at the start of graceful shutdown — Topic 123. |
| Readiness still 200 | The event did not reach the availability bean. Confirm you published `ReadinessState.REFUSING_TRAFFIC` via `AvailabilityChangeEvent.publish(publisher, source, state)`. |
| Liveness changed too | You published a `LivenessState` by mistake. Check the type parameter. |

### Proof 4 — measure the cost of the health check itself

The health endpoint is called every few seconds, per probe type, per pod, forever. It is
production traffic.

```bash
# With the Topic 65 load running against 8080, hammer the readiness endpoint:
for i in $(seq 1 200); do
  curl -s -o /dev/null -w '%{time_total}\n' http://localhost:9090/actuator/health/readiness
done | sort -n | awk '{a[NR]=$1} END {print "p50", a[int(NR*0.5)]; print "p99", a[int(NR*0.99)]}'
```

Then set `management.endpoint.health.cache.time-to-live=5s` and repeat.

| What you see | What it means |
|---|---|
| p99 in the low milliseconds, unchanged by caching | The check is cheap. Fine. |
| p99 in the hundreds of milliseconds without caching, much lower with it | The `db` indicator is doing real work per call and competing for the pool. Caching is earning its keep. |
| p99 rises sharply while the Topic 65 load is running, but is fine when idle | **Trap 4 confirmed on your own service.** The health check queues behind request traffic. Now compare against the counter-reading `ConnectionPoolHealthIndicator` and see the difference. |
| Occasional very slow responses among fast ones | The indicator is intermittently waiting for a connection. Under sustained load these become probe timeouts. |

Also check what Actuator itself costs at the JVM level: with the Topic 65 baseline running,
compare a run with the management port scraped every 5 seconds against one with scraping
off, and look at the p99 of your *application* endpoints. If probing measurably moves
application latency, your health checks are too expensive.

### Proof 5 — a regression test so this cannot come back

This is the highest-value artefact in the document, because a properties file is edited by
people who have not read it.

```java
package com.orderflow.availability;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = {
            "management.server.port=",                       // same port, simpler test
            "management.endpoint.health.probes.enabled=true",
            "management.endpoint.health.group.liveness.include=livenessState",
            "management.endpoint.health.group.readiness.include=readinessState,db"
        })
class ProbeSemanticsTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired TestRestTemplate rest;

    @Test
    void liveness_survives_a_database_outage_and_readiness_does_not() {
        assertThat(rest.getForEntity("/actuator/health/liveness", String.class).getStatusCode())
                .isEqualTo(HttpStatus.OK);
        assertThat(rest.getForEntity("/actuator/health/readiness", String.class).getStatusCode())
                .isEqualTo(HttpStatus.OK);

        postgres.stop();

        // Liveness MUST NOT change. This assertion is the entire point of the test.
        assertThat(rest.getForEntity("/actuator/health/liveness", String.class).getStatusCode())
                .isEqualTo(HttpStatus.OK);
        assertThat(rest.getForEntity("/actuator/health/readiness", String.class).getStatusCode())
                .isEqualTo(HttpStatus.SERVICE_UNAVAILABLE);
    }
}
```

| What you see | What it means |
|---|---|
| Test passes | Your probe semantics are correct **and now protected**. Anyone who adds `db` to the liveness group breaks the build. |
| Liveness assertion fails after `postgres.stop()` | Trap 1, caught in CI instead of in production. |
| Readiness assertion fails (still 200) | Hikari has not noticed yet. Add a repository call between the stop and the assertion, or add a short awaitility poll. Do not weaken the assertion. |
| The test hangs | The `db` indicator is waiting on the connection timeout. Set `spring.datasource.hikari.connection-timeout=2000` for the test. That hang is itself Trap 4 evidence. |

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the failure yourself and
written down what you saw. The value is not the knowledge — you already believe the
conclusion. The value is watching six pods die at once because of one line you wrote.

### The scenario

You point the liveness probe at a health group that includes the database. You take
Postgres down for sixty seconds. Every pod restarts, and keeps restarting after the
database comes back.

### Setup

Deploy `orderflow` as it stands, with **deliberately wrong** probe configuration.

`application.properties` — the wrong version:

```properties
management.server.port=9090
management.endpoints.web.exposure.include=health,info,prometheus
management.endpoint.health.probes.enabled=true

# WRONG ON PURPOSE: the database is in the liveness group.
management.endpoint.health.group.liveness.include=livenessState,db
management.endpoint.health.group.readiness.include=readinessState,db

management.endpoint.health.show-details=always
```

Deployment probes — the shape you already know, so only the relevant fields:

```yaml
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 9090 }
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  periodSeconds: 5
  failureThreshold: 3
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  periodSeconds: 5
  failureThreshold: 24
```

Scale to six replicas. Start the Topic 65 k6 load at the baseline rate, so you are watching
a *running system* fail rather than an idle one.

### Run the drill

Open three terminals and keep them all visible. Watching this happen in real time is the
point.

Terminal 1 — pod state:

```bash
kubectl get pods -l app=orderflow -w
```

Terminal 2 — events, which is where the reason is written:

```bash
kubectl get events --sort-by=.lastTimestamp -w | grep -iE 'probe|kill|restart|unhealthy'
```

Terminal 3 — the endpoint list, which is what your users experience:

```bash
while true; do
  printf '%s ready-endpoints=' "$(date +%T)"
  kubectl get endpoints orderflow -o jsonpath='{.subsets[*].addresses[*].ip}' | wc -w
  sleep 2
done
```

Now break it. Choose whichever matches your setup:

```bash
# If Postgres runs in the cluster:
kubectl scale statefulset orderflow-postgres --replicas=0

# If it runs in Docker beside the cluster:
docker stop orderflow-pg

# If you want a network partition instead of a stop (closer to the real failure):
kubectl patch networkpolicy orderflow-db-access --type=json \
  -p='[{"op":"replace","path":"/spec/ingress","value":[]}]'
```

Wait **60 seconds**. Then restore Postgres. Then keep watching for **five more minutes** —
the most important part of the drill happens after the dependency recovers.

### What to capture

Write these down before reading further. Actually write them.

1. Time from the database stopping to the first readiness failure event.
2. Time from the database stopping to the first `Killing container` event.
3. The number of pods with `RESTARTS` greater than zero, sixty seconds in.
4. The container's `Last State` reason and **exit code** from `kubectl describe pod`.
5. Where the application log stops on a killed pod (the last line before truncation).
6. The ready-endpoint count over time, sampled every two seconds.
7. **The time from the database recovering to the ready-endpoint count returning to six.**
8. Whether any pod entered `CrashLoopBackOff`, and what its backoff delay grew to.

### How to read it

| What you see | What it means |
|---|---|
| Readiness fails first, then liveness a few seconds later | Expected. Both groups contain `db`; readiness usually crosses its threshold first only because of probe scheduling jitter. The dangerous one is the second. |
| `Liveness probe failed: HTTP probe failed with statuscode: 503` in Events | **The drill has fired.** Kubernetes has decided your JVM is unrecoverable because a *different process* is down. |
| `Killing container ... Container failed liveness probe, will be restarted` | The kill decision, in writing. Screenshot this; it is the single most persuasive artefact when you argue this configuration in a design review. |
| Exit code **137**, `Reason: Error`, and no `OOMKilled` | SIGKILL from the kubelet, not memory. Learn to tell these apart: `OOMKilled` appears as the explicit reason when it is memory (Topic 82). |
| The application log stops mid-sentence with no exception | The JVM was killed. There is **no application-side evidence** of what happened. This is why people misdiagnose it for hours. |
| All six pods restart within a ~15-second window | The amplification. One dependency blip, six simultaneous cold starts. |
| Ready endpoints drop to zero | Total outage. Note that this would have happened anyway with correct probes — the difference is what happens next. |
| **After the database recovers**, ready endpoints take far longer to return than the JVMs take to boot | The real cost. The pods must restart, re-run startup, refill pools, and re-warm. Compare this number to the 60-second outage you injected. |
| Pods enter `CrashLoopBackOff` with a growing backoff | The self-sustaining phase: six JVMs opening 20 connections each against a just-recovered Postgres can knock it back over, failing the probes again. Kubernetes then *slows down* the restarts, which extends the outage further. |
| Recovery is fast and clean | Your Postgres tolerated the reconnection stampede. Good for you; note that this is a property of your database's headroom, not of your configuration. Re-run at a higher replica count and see if it holds. |

### Now fix it, and prove the fix with the same drill

```properties
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.readiness.include=readinessState,db
```

One word removed. Re-run the identical drill and capture the same eight measurements.

| What you should now see | Why |
|---|---|
| Readiness fails; ready endpoints go to zero | Unchanged, and correct. There is genuinely nothing to serve. |
| **Zero `Killing container` events. Zero restarts.** | The JVMs stay alive through the whole outage, holding their JIT-compiled code, their caches, and their thread pools. |
| Ready endpoints return within one or two probe periods of the database recovering | No restart, no startup, no cold JVM. Recovery time is now bounded by the probe period, not by JVM startup. |
| Recovery time drops by a large factor versus the broken run | This ratio is your drill's result. Record both numbers side by side; it is the most compelling line in your Topic 124 review. |

### Then push it further — three variations worth running

1. **A slow database rather than a dead one.** Instead of stopping Postgres, make it slow
   (`pg_sleep` in a loop from another client, or a CPU limit on its container). Does the
   `db` indicator time out? Does readiness flap on and off? Flapping is a distinct and
   nastier failure than clean failure, because it causes continuous load-balancer churn.
   Record how it differs.

2. **Trap 4, deliberately.** Remove `management.endpoint.health.cache.time-to-live`, set the
   Hikari pool to 5, and run the Topic 65 load. Watch whether pods drop out of rotation
   one at a time in an accelerating staircase, and whether the readiness failures are 503s
   or timeouts. Then add the cache and the counter-reading indicator and re-measure.

3. **The startup-probe budget.** Set `startupProbe.failureThreshold` to 2 (a 10-second
   budget) and deploy. Watch the crash loop from Trap 2 and note that the log truncation
   point differs on each attempt. Then measure your actual startup time and set the budget
   from evidence — which is Topic 122.

### Write it up

This drill produces a runbook entry. Write it now, while it is fresh, because Topic 124
asks for exactly this artefact:

- **Failure mode:** dependency failure amplified into fleet-wide restart loop.
- **Trigger:** any contributor other than `livenessState` present in the liveness group.
- **Detection:** simultaneous `RESTARTS` increments across the Deployment; exit code 137
  with no `OOMKilled`; application logs truncated mid-startup.
- **Mitigation:** the liveness group contains `livenessState` only.
- **Regression control:** the Testcontainers test from Proof 5, failing the build.
- **Measured blast radius:** *(your two recovery-time numbers, from this drill)*.

---

## Practice exercises

### 1 — Easy: audit what you actually expose

On any Spring Boot service you have (or the lab app from Hands-on):

1. List every exposed endpoint: `curl -s localhost:9090/actuator | jq -r '._links | keys[]'`.
2. List every health contributor and note, for each, **who put it there** — you, or a
   dependency you added for another reason.
3. For each contributor, answer the three questions from Example 2: can the pod serve
   without it, would shedding help, would restarting help.
4. Produce the resulting liveness/readiness membership table.
5. Set `management.health.defaults.enabled=false` and opt back in only to what your table
   says. Confirm the health endpoint now contains exactly what you chose and nothing else.

Deliverable: the table, plus one sentence per dependency justifying its placement. This is a
section of the Topic 124 review; write it as if someone else will read it.

### 2 — Medium: the audit (combines Topics 40–119)

The configuration and code below contain **six** defects. Four are from this topic; two are
from earlier topics. For each: name the topic, state the **observable** symptom in
production, and write the fix.

```properties
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.health.group.liveness.include=livenessState,db,redis
management.endpoint.health.group.readiness.include=readinessState,db,kafka,paymentGateway
```

```java
@Component
public class OrderflowHealthIndicator implements HealthIndicator {

    @Autowired
    private OrderRepository orders;

    private final RestTemplate rest = new RestTemplate();

    @Override
    @Transactional
    public Health health() {
        long count = orders.countAllOrders();                       // full table scan on 1M rows
        var res = rest.getForEntity("https://gateway.example.com/ping", String.class);
        return Health.up()
                .withDetail("orderCount", count)
                .withDetail("gateway", res.getStatusCode())
                .withDetail("datasourceUrl", System.getenv("SPRING_DATASOURCE_URL"))
                .build();
    }
}
```

Hints in the order to think about them: one defect makes the *entire cluster* restart on a
cache outage. One makes the health check slower than the probe timeout under load, and it
is not the one you think — read the `RestTemplate` line and ask what its default timeout is.
One is a Topic 39 injection-style issue that also makes the class untestable. One is a Topic
55 issue where the annotation should not be on this method at all. Two are disclosure
problems of different severities.

### 3 — Hard: production simulation — design the honest readiness signal

**Part A — the dependency matrix.** For `orderflow` as it stands after Topic 119, build the
full table: every dependency, every request class (order placement, catalogue read, order
history, payment callback), and for each cell, whether that request class can be served when
that dependency is down. Six dependencies by four request classes.

**Part B — derive the probe configuration from the matrix, not from instinct.** Write the
`management.endpoint.health.group.*` properties and justify each inclusion against the
two-condition rule. Where the matrix says "some request classes work and some do not",
state explicitly what you do instead of using readiness, and why one bit cannot express
that state.

**Part C — build the pod-local signal.** Implement the `ConnectionPoolHealthIndicator` from
Example 2. Then, under the Topic 65 load, deliberately induce single-pod pool exhaustion:
send a burst of requests that hit the Topic 55 path (an HTTP call inside a transaction) to
**one** pod only, using a direct pod IP rather than the Service. Confirm that (a) the
affected pod goes unready, (b) the other five stay ready, (c) overall error rate falls
compared to the same experiment without the indicator. That third measurement is the one
that justifies the indicator's existence — if error rate does not improve, delete it.

**Part D — the flapping question.** Reduce the indicator's `GRACE` window to zero so it
reports DOWN on instantaneous saturation. Re-run Part C. Measure readiness transitions per
minute and total error rate. Then explain, with your numbers, why a hysteresis window is
not optional. Choose a `GRACE` value and justify it against your measured p99.

**Part E — argue against yourself.** You included `db` in readiness. Every pod shares that
database, so when it fails all six go unready and the Service empties — which means every
request gets an ingress 503 with no application log line, no metric and no trace (Trap 3's
blindness problem). Make the strongest possible case for **excluding** `db` from readiness
and letting requests fail inside the application with a `ProblemDetail` instead. Then decide,
and write down what would change your mind.

---

## Interview questions

### Q1 — "We point both probes at `/actuator/health`. What's wrong with that?"

**Mid-level answer:** "You should use the separate liveness and readiness endpoints. Liveness
is for restarting and readiness is for load balancing, so they shouldn't be the same check."

**Senior answer:** "The mechanical problem is that `/actuator/health` is the aggregate of
every health contributor Actuator discovered on the classpath, ANDed together — and you did
not choose that set; auto-configuration did, based on which starters are present. So the
moment any dependency is DOWN, the endpoint returns 503.

Pointed at readiness, that is merely aggressive. Pointed at **liveness**, it means a
thirty-second Postgres blip kills every pod in the Deployment simultaneously, because all of
them see the same failure at the same time. And the amplification is the real cost: six JVMs
restart, each takes seconds to start, each comes up cold — no JIT-compiled code, empty
caches — and all six open their connection pools against the just-recovered database at the
same instant. That reconnection stampede can knock the database back over, which fails the
probes again, and Kubernetes then adds exponential backoff. A thirty-second dependency
outage becomes a multi-minute application outage, and the fingerprint in the timeline is
that the database recovers well before the application does.

The diagnostic detail I would look for: exit code 137 with no `OOMKilled` reason, and
application logs that stop mid-startup with no exception. That combination is the
signature, and it is why this gets misdiagnosed as an out-of-memory problem.

The fix is health groups: `liveness` contains `livenessState` and nothing else — Boot's own
'has the context started and is it not broken' state — and `readiness` contains
`readinessState` plus the dependencies without which this pod genuinely cannot serve. Then
I'd add a Testcontainers test that stops the database and asserts liveness stays 200 while
readiness goes 503, because this is a properties file and the next person to edit it will
not have read the design doc.

The rule I apply to every dependency: if a restart would not fix it, it does not belong in
liveness."

**What separates them:** the mid answer knows the vocabulary. The senior answer explains
**why the default is dangerous** (the set is assembled by auto-configuration, not chosen),
describes the **amplification and the self-sustaining loop**, gives the **specific
diagnostic fingerprint**, and proposes a **regression test** rather than only a fix. Naming
the reconnection stampede is what signals someone who has watched it happen.

**Follow-up:** "So what *should* ever make liveness fail?" — Almost nothing. A deadlock you
can detect, an unrecoverable initialisation state, an executor that has permanently
stopped consuming its queue. If you cannot name a concrete condition where a restart is the
only remedy, `livenessState` alone is the correct configuration. Many strong services never
fail liveness in their lifetime, and that is a healthy sign, not an unused feature.

---

### Q2 — "What should be in your readiness probe?"

**Mid-level answer:** "The dependencies your service needs — database, cache, message broker.
If any are down, the pod isn't ready."

**Senior answer:** "I apply two conditions, and a dependency has to pass both. First: can
this pod serve its core request without it? Second: would another pod plausibly be able to?

Readiness is a load-shedding signal, so it only helps if there is somewhere to shed to. If a
dependency is shared identically by every replica — Kafka, an external payment gateway,
usually the primary database — then when it fails, every pod fails the probe at the same
time, the Service's endpoint list empties, and every request gets a 503 at the ingress. That
is worse than the alternative in a specific way people miss: those requests never reach your
application, so they produce no log line, no metric and no trace. You have blinded your own
observability at the worst possible moment, and your RED dashboard shows traffic going to
zero, which reads like a traffic drop rather than an outage.

So for `orderflow`: Redis is out — the cache is an optimisation and a degraded pod beats no
pod. Kafka is out — only the outbox relay publishes, and if Kafka is down the outbox table
grows by design; HTTP order placement is unaffected, so putting Kafka in readiness would
stop order placement for a reason unrelated to order placement. The payment gateway is out —
it fails on all six pods at once, and catalogue and history reads do not need it at all.

What I *do* want in readiness is anything that can differ per pod. The database, arguably,
because without it nothing works. And a connection-pool saturation indicator with a
hysteresis window, because one pod can saturate independently — and that one is genuinely
useful load shedding, where the other five absorb the traffic.

For everything else, I report it in a detailed health group for humans and dashboards, and I
alert on it. The distinction I hold onto is that a probe is an instruction to the platform
and a metric is information for a human. 'Kafka is down' is information. 'Stop sending this
pod traffic' is an instruction. Encoding the first as the second is the mistake."

**What separates them:** the two-condition rule, and specifically condition (b). Most people
never think about whether shedding *helps*. The observability-blindness point — that ingress
503s produce no telemetry — is the detail that shows someone has debugged this rather than
read about it.

**Follow-up:** "Order placement is broken but reads work. How do you express that?" — Not
with readiness; it has one bit and this is a richer state. Fail the placement endpoint with
a clear `ProblemDetail`, let the circuit breaker fail fast, alert on the placement error
rate, and keep serving reads. Trying to encode partial degradation in a boolean is how you
get flapping.

---

### Q3 — "Your pods are in a crash loop right after a deploy. The application logs show no exception. Walk me through it."

**Mid-level answer:** "I'd check the logs and `kubectl describe pod` to see why it's
restarting. Maybe it's out of memory."

**Senior answer:** "'No exception' is the most informative part of that sentence — it means
the JVM did not decide to stop, something stopped it. So I go to `kubectl describe pod` and
look at two specific fields: the Last State reason and the exit code, and the Events list.

Exit code 137 is SIGKILL. Two things produce it and they are distinguishable: if the reason
says `OOMKilled`, the container exceeded its memory limit — and for a JVM that is usually
the Topic 82 story, where the heap is within `-Xmx` but metaspace, thread stacks, code cache
and direct buffers push the RSS over the container limit. If instead the Events show
`Liveness probe failed` followed by `Container failed liveness probe, will be restarted`,
it is the probe.

If it is the probe and it happens from the very first attempt, with the failure being
connection-refused rather than a 503, and the application log truncating at a *different
point on each attempt* — then the probe budget is shorter than JVM startup. A Spring Boot
service with Hibernate and a connection pool takes seconds to start, and Kubernetes cannot
tell 'still starting' from 'wedged' by connection-refused alone. That is what a startup
probe is for.

The fix is a startup probe with a budget derived from measured startup time, not guessed.
And I would treat 'we had to raise the budget' as a finding rather than a fix: if startup
grew, I want to know where the seconds go — JVM init, class loading, context refresh, or
bean instantiation — because a team that keeps raising the threshold has hidden a
regression. That decomposition is a separate piece of work and it changes what the right
optimisation is; reaching for native image before measuring is how people spend a month on
the wrong two hundred milliseconds.

One more thing I'd check: whether the liveness probe port is the management port and
whether the management server starts at the same time as the main server. If Actuator is on
a separate port that comes up later, you can get connection-refused on a perfectly healthy
application."

**What separates them:** distinguishing 137-with-OOMKilled from 137-from-liveness, and
naming **connection-refused versus 503** and **the varying log truncation point** as the
discriminating evidence. Then treating a raised threshold as a finding rather than a
resolution — that is the judgement an interviewer is actually testing.

**Follow-up:** "How would you measure startup time properly?" — Not with a stopwatch on the
log timestamps. Boot's `ApplicationStartup` instrumentation, `-Xlog:class+load` for the
class-loading slice, and comparing JVM init against context refresh before choosing an
optimisation. Topic 122.

---

### Q4 — "Your readiness probe times out under load, pods drop out one by one, and then everything is down. What happened?"

**Mid-level answer:** "The service got overloaded and couldn't respond to the probes in time.
I'd increase the probe timeout or scale up."

**Senior answer:** "That is a positive feedback loop, and the shape of it is diagnostic: pods
leaving rotation one at a time with the interval *shrinking* between departures. Independent
failures do not accelerate; a feedback loop does.

The usual mechanism in a Spring service is that the health check consumes the resource it is
measuring. `DataSourceHealthIndicator` borrows a connection from HikariCP. When the pool
saturates, the health check queues behind request traffic and waits up to the connection
timeout — thirty seconds by default — so the probe times out first. The pod is marked
unready, its traffic moves to the remaining pods, their pools saturate sooner, and it
cascades. Every JVM is alive the whole time, and every pod would have served some traffic
successfully.

The evidence that confirms it: the probe failure is a **timeout**, not a 503. A 503 means
the endpoint answered and reported DOWN; a timeout means it could not answer at all, which
points at resource starvation rather than a dependency being down. And in Micrometer I would
see `hikaricp_connections_pending` rising before the first readiness failure — the cause
precedes the symptom, which is what tells me which way the causality runs.

Fixes, in order of how much they help. First, make the indicator not consume the resource: I
read Hikari's MXBean counters instead of borrowing a connection, so the check cannot make
the problem worse. Second, cache the health result with a short TTL so probe frequency does
not drive pool usage. Third, if I genuinely need to execute SQL, give the indicator its own
one-connection datasource so it is isolated from request load. Fourth, hysteresis: require
sustained saturation before reporting DOWN, because a probe that flaps produces load-balancer
churn and connection resets while everything technically works.

Raising the probe timeout is not on that list, and scaling up is not either — scaling adds
pods that will saturate the same shared database, which is Topic 109's point that a bigger
pool moves the contention into Postgres rather than removing it. The underlying question is
why the pool is saturating, and at this baseline that is usually a transaction holding a
connection across a network call."

**What separates them:** recognising the **accelerating staircase** as a feedback signature,
knowing that the health check itself is the amplifier, and reading **timeout versus 503** as
evidence. Rejecting "raise the timeout and scale up" — the two most natural instincts — with
a reason is the senior move.

**Follow-up:** "How do you know the pool is the cause and not the effect?" — Ordering in the
metrics: pending-connection count rises before the first readiness failure. If readiness
failures come first, you are looking at something else.

---

### Q5 — "Is it ever right for liveness to fail? Give me a concrete example."

**Mid-level answer:** "If the application is deadlocked or has run out of memory, liveness
should fail so it gets restarted."

**Senior answer:** "Rarely, and I would want a named, detectable condition before I wired
anything into it — the default of `livenessState` alone is correct for the large majority of
services, and a service that never fails liveness in its lifetime is healthy, not
under-instrumented.

The conditions that actually qualify share one property: **a restart is the only remedy, and
the process cannot recover on its own.** A few real ones. A thread-pool executor whose
worker threads have all died from an `Error` and which is therefore permanently not draining
its queue — the queue grows, nothing processes, and no amount of waiting helps. A classic
deadlock between two locks, which `ThreadMXBean.findDeadlockedThreads()` can actually
detect. An initialisation path that left a singleton in a state it can never leave. On the
memory side I would be careful: the JVM being at 100% GC with almost nothing reclaimed is a
genuinely unrecoverable state, but I would not write that check by hand — the container
memory limit and OOMKill already handle the terminal case, and a hand-rolled heap-usage
check will fire during a normal pre-GC peak and restart a healthy pod.

The way I would wire it is not through a health indicator. I would publish
`AvailabilityChangeEvent` with `LivenessState.BROKEN` from the component that detected the
condition, because that keeps the detection where the knowledge is, and Boot's `livenessState`
contributor picks it up with no extra configuration.

And I would hold a very high bar for adding one, because the failure mode of a wrong liveness
check is fleet-wide restarts, which is strictly worse than the condition it was meant to
catch. The asymmetry matters: a missing liveness check costs you one wedged pod that stays
wedged until someone notices. A wrong one costs you every pod, at the worst possible moment.
I would rather have the first problem."

**What separates them:** the mid answer names the textbook cases without noticing that
Kubernetes already handles the memory one, and would happily add a heap check that fires
during normal GC. The senior answer states the **asymmetry of the failure modes**, insists on
a detectable condition, uses `AvailabilityChangeEvent` rather than an indicator, and is
comfortable concluding "almost never".

**Follow-up:** "You have a deadlock detector wired to liveness. What is the risk?" — False
positives from lock contention that is not a true deadlock, and the fact that
`findDeadlockedThreads` only finds monitor and `Lock` cycles — it will not see a pool-vs-pool
deadlock (Topic 109), where every thread is legitimately waiting on a connection. So the
check can miss the deadlock you most expect at this baseline while firing on something
benign.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Liveness failing means "restart". Derive, from that one sentence alone, the complete rule
   for what may appear in a liveness group — without recalling the rule from the text.

2. Readiness is a load-shedding signal. Explain why that makes a *shared* dependency a poor
   readiness input, and then explain why `db` is nonetheless usually included anyway. Which
   argument is stronger, and does your answer change if `orderflow` had a read replica?

3. Boot auto-discovers health indicators from the classpath. Name one thing that makes better
   and one thing that makes worse, and then decide whether you would set
   `management.health.defaults.enabled=false` on a production service. Justify it with a
   concrete scenario, not a preference.

4. A health check that consumes the resource it measures becomes unavailable exactly when it
   matters most. Name two other places in this curriculum where instrumentation has the same
   property, and say what the general mitigation is.

5. A cold JVM behind a load balancer is slow for the first few hundred requests (Topic 74).
   Readiness returning 200 immediately sends it a full share of traffic. Design a mechanism
   that ramps traffic to a newly ready pod, decide whether it belongs in the application or
   the platform, and say what evidence would tell you it was worth building.

6. Your liveness group contains only `livenessState`, which Boot sets to `CORRECT` once the
   context starts and never changes. So the liveness probe now only detects "the process is
   not answering HTTP at all". Is that a useful check, or have you configured a probe that
   can never usefully fail? Argue both sides.

7. Trap 3 says an ingress 503 produces no application log line, metric or trace. Given that,
   design one alert that fires for the "all pods unready" state and explain why your normal
   RED alerting (Topic 118) will not.

---

## Quick reference card

### The two questions

```
LIVENESS  = "is this process unrecoverable?"   -> remedy: RESTART
READINESS = "can this pod serve right now?"    -> remedy: STOP SENDING TRAFFIC

If a restart would not fix it, it does not belong in liveness. No exceptions.
```

### Minimum correct configuration

```properties
management.server.port=9090
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.endpoint.health.show-details=when-authorized
management.endpoint.health.probes.enabled=true
management.endpoint.health.group.liveness.include=livenessState
management.endpoint.health.group.readiness.include=readinessState,db
management.endpoint.health.group.detailed.include=*
management.endpoint.health.cache.time-to-live=5s
```

### Group knobs

```properties
management.endpoint.health.group.<n>.include=a,b        # or *
management.endpoint.health.group.<n>.exclude=b          # exclusions win
management.endpoint.health.group.<n>.show-details=never|when-authorized|always
management.endpoint.health.group.<n>.status.http-mapping.out-of-service=200
management.endpoint.health.group.<n>.additional-path=server:/healthz
```

### Turning the buffet into a menu

```properties
management.health.defaults.enabled=false
management.health.db.enabled=true
management.health.diskspace.enabled=false
```

### The availability API

```java
AvailabilityChangeEvent.publish(publisher, source, ReadinessState.REFUSING_TRAFFIC);
AvailabilityChangeEvent.publish(publisher, source, ReadinessState.ACCEPTING_TRAFFIC);
AvailabilityChangeEvent.publish(publisher, source, LivenessState.BROKEN);   // rarely correct

@EventListener void on(AvailabilityChangeEvent<ReadinessState> e) { ... }
```

### Diagnostic commands

```bash
curl -s -o /dev/null -w '%{http_code}\n' :9090/actuator/health/liveness
curl -s -o /dev/null -w '%{http_code}\n' :9090/actuator/health/readiness
curl -s :9090/actuator/health | jq -r '.components | keys[]'
curl -s :9090/actuator | jq -r '._links | keys[]'          # AUDIT what is exposed

kubectl describe pod <pod>          # Events + Last State + exit code
kubectl get endpoints <svc>         # who is actually in rotation
kubectl get events --sort-by=.lastTimestamp | grep -i probe
```

### Reading the evidence

| Evidence | Meaning |
|---|---|
| exit 137 + `Reason: OOMKilled` | container memory limit — Topic 82, not a probe problem |
| exit 137 + `Liveness probe failed` event | the kubelet killed it; check what is in the liveness group |
| probe failure = **503** | the endpoint answered and reported DOWN — a dependency is down |
| probe failure = **timeout** | the endpoint could not answer — resource starvation; suspect Trap 4 |
| probe failure = **connection refused**, from the first attempt | not started yet — startup-probe budget too small |
| log truncates mid-startup, different point each time | SIGKILL during startup, i.e. the same thing |
| ready-pod count falling in an *accelerating* staircase | positive feedback loop, not independent failures |

### Endpoints never to expose publicly

```
heapdump    -> every secret currently in memory
env         -> configuration, imperfectly masked
configprops -> the same, structured
loggers     -> a WRITE endpoint; turns on DEBUG in production
threaddump  -> internal class names and library versions
shutdown    -> exactly what it says
mappings    -> your full undocumented API surface
```

### Gotchas checklist

- [ ] Liveness group contains `livenessState` and nothing else.
- [ ] Readiness membership justified against the two-condition rule, in writing.
- [ ] `probes.enabled=true` set explicitly, not left to Kubernetes auto-detection.
- [ ] A startup probe exists, with a budget from a *measured* startup time.
- [ ] Health result cached; no indicator borrows a pool connection on every probe.
- [ ] Indicators have timeouts and never throw.
- [ ] Management on its own port; exposure list audited with `curl /actuator`.
- [ ] `show-details` is not `always` in production.
- [ ] A Testcontainers test asserts liveness survives a database outage.
- [ ] The drill's before/after recovery-time numbers are recorded.

---

## When would I use this at work?

**1. Reviewing the Deployment manifest of a service you are inheriting.**
Two `httpGet` paths tell you almost everything about whether the team has operated the
service under stress. Both pointing at `/actuator/health` means the next dependency blip is
a fleet-wide outage, and you can say so with the mechanism, the amplification, and a
twenty-second test that proves it. This is one of the highest-leverage code review comments
available in a Java shop, and it costs one line to fix.

**2. During an incident, distinguishing "the platform killed us" from "we crashed".**
Application logs with no exception plus exit code 137 is a specific, recognisable state, and
the difference between `OOMKilled` and a liveness event routes the investigation in two
completely different directions — Topic 82's container memory arithmetic versus this
document's probe configuration. Knowing to look at `kubectl describe pod` before reading
application code saves the first thirty minutes of the incident, which are the thirty
minutes that matter.

**3. Making the readiness decision for a new service, on purpose.**
Every new service gets this configuration, and it is almost always copied from the last one.
Being the person who asks "would shedding this help, and would restarting help" for each
dependency — and who writes the answers down — converts an inherited default into a
designed control. That written table is a Topic 124 review artefact and, later, a Topic 130
input: your availability SLO is only meaningful if readiness reports honestly.

---

## Connected topics

**Prerequisites:**
- **42 — Auto-configuration mechanics**: health indicators appear because a starter is on the
  classpath and a condition matched. The condition-evaluation report tells you why one
  exists.
- **56/57 — Spring Security**: the management port needs an authorization story, and the two
  probe paths need to be the exception.
- **65 — the load baseline**: every claim in this document about probe behaviour under load
  is only checkable because you have a load generator and recorded numbers.
- **109 — HikariCP**: the pool is the resource Trap 4 is about, and pool metrics are the
  leading indicator of the readiness cascade.
- **111 — Resilience4j**: the circuit-breaker state is health *information*, and the reason it
  must not be a readiness *instruction*.
- **118 — Micrometer**: `hikaricp_connections_pending` and the ready-pod count are the
  metrics that let you see a feedback loop while it is forming.

**This unlocks:**
- **122 — Docker and startup**: the startup-probe budget is only defensible if you have
  measured and decomposed startup time. Trap 2 is the reason that topic exists.
- **123 — Graceful shutdown and rolling deploys**: readiness must go **false before** the
  drain begins. Boot's `ReadinessState.REFUSING_TRAFFIC` transition at the start of graceful
  shutdown is the mechanism, and it only helps if your readiness group is the one the probe
  reads.
- **124 — Production-readiness gate**: the dependency/probe table and this document's drill
  results are two named sections of that review.
- **129 — Capacity**: the ready-pod count is what your capacity model is actually about, and
  a flapping readiness signal invalidates it.
- **130 — SLOs**: availability measured at the ingress is meaningless if readiness is
  dishonest — you will measure a service that reports itself healthy while failing.
- **133 — Postmortems**: this document's drill *is* a rehearsal incident. Write it up as one.

**Also related:**
- **74 — JIT warm-up**: why a newly ready pod is slow, and why restarting six pods is much
  more expensive than it looks.
- **82 — Container limits**: the other cause of exit code 137, and the one this is most often
  confused with.
- **113 — Kafka consumer groups**: consumer health is a dependency you will be tempted to put
  in readiness. Trap 3 is why you should not.
- **120 — Logging and MDC**: an ingress 503 produces no log line, which is why an unready
  fleet is also an unobservable one.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0. One thing in this
document is deliberately hedged: the exact set of environment variables Boot inspects to
auto-detect a Kubernetes environment for `management.endpoint.health.probes.enabled`. I have
not asserted the variable names because the detection has been refined across releases, and
because the correct practice is to set the property explicitly anyway — which removes the
question entirely and gives you the same endpoints on your laptop as in production. There is
no `[BOOT 3.x DELTA]` of substance here: health groups, the probe endpoints and the
availability state API have all been present and stable since Boot 2.3–2.6, and nothing in
this document behaves differently on 3.x. The mechanism — liveness means restart, readiness
means stop routing, and the aggregate endpoint is not either of them — will outlive every
version number on this page.*
