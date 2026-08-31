# 118 — Micrometer Metrics, RED/USE, and Cardinality Traps

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: RED metrics (rate, errors, duration) on every `orderflow` endpoint, and USE metrics (utilisation, saturation, errors) on the HikariCP pool from Topic 109 and every executor from Topic 90 — with a written cardinality budget that says, per meter, how many time series it is allowed to create and what enforces that limit.

---

## Mechanical statement

**A Prometheus time series is created per unique label set. One tag whose value is an
order ID creates one series per order — the scrape target's memory grows without bound
and the monitoring system dies before the application does.**

Read that as three separate mechanical facts, because each one is load-bearing:

1. **Identity is name + tags, not name.** `http_server_requests_seconds_count{uri="/orders",status="200"}`
   and `http_server_requests_seconds_count{uri="/orders",status="500"}` are two different
   series that happen to share a name. Every distinct combination of tag values is a
   separate object in the application's meter registry, a separate line in the scrape
   payload, and a separate append-only chunk in Prometheus's TSDB.
2. **Nothing in the pipeline bounds that count for you.** Not Micrometer, not Actuator,
   not Prometheus. The number of series is a property of the *values* your code puts in
   tags, and your code is what decides.
3. **The blast radius is the monitoring system.** The application usually survives longer
   than Prometheus does, because Prometheus holds every active series of every target in
   memory and your application holds only its own. So the failure looks like "monitoring
   is down" — and you lose the ability to see the incident you are having, which is the
   worst possible time to lose it.

---

## The bridge from what you know

### `prom-client` is a genuine analogue — say the mapping once and move on

You have instrumented Node services with `prom-client`. The concepts transfer cleanly:

| `prom-client` (Node) | Micrometer (Java) | Same? |
|---|---|---|
| `new Counter({ name, help, labelNames })` | `Counter` from `MeterRegistry.counter(name, tags)` | Yes |
| `new Gauge({...})` | `Gauge.builder(name, obj, fn).register(registry)` | Mostly — see the weak-reference trap |
| `new Histogram({ buckets: [...] })` | `Timer` with `publishPercentileHistogram()` or explicit `serviceLevelObjectives(...)` | Yes, different spelling |
| `new Summary({ percentiles: [...] })` | `Timer` with `publishPercentiles(0.5, 0.95, 0.99)` | Yes — **and it has the same fatal aggregation flaw in both languages** |
| `register.metrics()` on `/metrics` | Actuator's `/actuator/prometheus` | Yes |
| `collectDefaultMetrics()` | Boot auto-registers JVM, system, Tomcat, Hikari, Kafka binders | Java's set is much larger |
| label cardinality kills Prometheus | label cardinality kills Prometheus | **Identical.** The trap is not a Java trap |

**HONEST ANALOGUE.** If you know why you must never put a user ID in a `prom-client`
label, you already know 118's headline lesson. This document exists for the parts that
are *not* the same.

### The three parts that are genuinely different in Java

**1. The `MeterRegistry` is a Spring bean, and things register themselves into it.**

In Node you called `new Counter(...)` and got an object. In Spring, `MeterRegistry` is a
single injected bean, and a large amount of instrumentation registers into it *without
you writing a line*: HTTP server timings, HikariCP pool gauges, JVM memory and GC, Tomcat
threads, Kafka client metrics, Spring Data repository timings, cache statistics. These
arrive because a dependency is on the classpath (Topic 42's mechanism, again).

The consequence is the same shape as Topic 121's health indicators: **you did not choose
what is in there.** The first honest thing to do on any Spring service is print the meter
list and read it, rather than assume it matches what you would have written.

**2. `Timer` is not `Histogram` — it is a *choice* between two export shapes, and one of
them cannot be aggregated.**

Micrometer's `Timer` can publish three different things, and the difference decides
whether your dashboard's p99 is a real number or a nonsense number:

- **client-side percentiles** (`publishPercentiles`): the JVM computes p50/p95/p99 from a
  local sketch and exports `quantile="0.99"` series. **These cannot be combined across
  instances.**
- **a bucket histogram** (`publishPercentileHistogram` or `serviceLevelObjectives`):
  the JVM exports `_bucket{le="..."}` counters. **These can be summed across instances and
  the quantile computed afterwards.**
- **count and sum only** (the default): enough for a rate and an average, and nothing else.

**3. Meter names are dotted; Prometheus names are snake-cased for you.**

You write `orderflow.orders.placed`. The Prometheus registry exports
`orderflow_orders_placed_total`. Micrometer's naming convention layer does the
translation, adds `_total` to counters, and appends a base-unit suffix (`_seconds`,
`_bytes`) when the meter declares one. This means **the name you grep for in the
exposition output is not the name in your source**, which trips up every Java engineer
exactly once.

### The one sentence that separates you from a mid-level engineer

**You cannot average p99s across instances. You must aggregate the buckets.**

This is not a Java fact, it is an arithmetic fact, but it shows up in Java as a specific
misconfiguration (`publishPercentiles` instead of `publishPercentileHistogram`) plus a
specific wrong PromQL query (`avg(...{quantile="0.99"})`). Say it out loud now; there is
a whole section on it below and an interview question about it at the end.

---

## What is this?

**Micrometer** is a vendor-neutral metrics *facade*. It is to metrics what SLF4J
(Topic 120) is to logging: your code depends on an API, and a *registry* implementation
decides where the data goes — Prometheus, OTLP, CloudWatch, Datadog, or several at once
through a `CompositeMeterRegistry`.

There are five meter types you will actually use.

| Meter | What it is | Reset on scrape? | Use it for |
|---|---|---|---|
| `Counter` | monotonically increasing `double` | No — it only ever goes up (and resets to 0 on restart) | orders placed, payments declined, messages consumed |
| `Gauge` | an instantaneous reading, sampled at scrape time | N/A | pool connections in use, queue depth, cache size |
| `Timer` | count + total time + optional distribution of *short* operations | No | request duration, gateway call duration |
| `DistributionSummary` | same as `Timer` but for a non-time quantity | No | order value, payload size, lines per order |
| `LongTaskTimer` | how long the *currently running* invocations have been running | N/A | outbox relay batch, nightly reconciliation, anything that can hang |

The `Timer` / `LongTaskTimer` distinction is the one people miss, so state it plainly:

> **A `Timer` records nothing until the operation finishes.** An operation that has been
> stuck for ten minutes contributes zero observations to a `Timer`. Its *absence* is the
> only signal, and absence is a terrible signal. A `LongTaskTimer` reports the duration of
> in-flight invocations at scrape time, so a stuck operation is visible *while it is
> stuck*.

**RED** and **USE** are the two instrumentation checklists this topic makes you apply.

- **RED — for every request-serving endpoint:** **R**ate (requests per second),
  **E**rrors (failed requests per second), **D**uration (a *distribution*, not a mean).
  RED answers "what is the user experiencing".
- **USE — for every finite resource:** **U**tilisation (fraction of the resource busy),
  **S**aturation (how much work is *queued* for it), **E**rrors (rejections, timeouts).
  USE answers "which resource is the constraint".

For `orderflow`, RED applies to `GET /products/{id}`, `GET /orders`, `POST /orders`,
`POST /payments`. USE applies to the HikariCP pool (Topic 109), every
`ThreadPoolTaskExecutor` (Topic 90), the Tomcat thread pool, and the Kafka consumer
(Topic 113, where saturation is *lag*).

You already know both frameworks from the Node side. What is new is which Java object
publishes which half, and that is the content of Example 2.

---

## Why does it matter?

Four reasons, in descending order of how often they bite.

**1. Because the average hides exactly what the SLO is about.**

Your Topic 65 baseline recorded p50, p95, p99 and p999. Topic 130 will write an SLO
against one of them. An average response time cannot reconstruct any of those numbers,
and it moves in the *wrong direction* under the failure modes you care about: when 1% of
requests take ten seconds and 99% take five milliseconds, the mean barely twitches. Every
distributed-systems failure you drilled in Phases 8–11 — a GC pause (Topic 71), a
safepoint stall (Topic 73), pool exhaustion (Topic 109), a rebalance (Topic 113) — is a
*tail* event. The average is structurally blind to all of them.

**2. Because metrics are how you know a fix worked.**

Topic 50's rule was "only a query counter proves an N+1 fix." This is that rule at
service scale. Every change you make from here — pool sizing, GC choice, a cache, an
outbox — is a claim about a number. Without the number, the claim is a belief.

**3. Because the cardinality failure mode takes down the thing you use to diagnose
outages, and it does it during a deploy.**

The sequence is always the same: someone adds a helpful tag, the change passes review
because it is one line, it is deployed, series count climbs with traffic, Prometheus's
memory climbs with series count, and Prometheus is OOMKilled in the middle of the
resulting incident. Now you are debugging blind. This is why cardinality gets a failure
drill of its own and a written budget.

**4. Because "we have metrics" and "we have the right metrics" are different claims, and
only one of them survives an incident.**

Boot gives you hundreds of meters for free. Almost none of them are your SLIs. The
work in this topic is deciding what you would *want to be already recorded* at 3am, and
recording it now.

---

## Machine-level reality

### The registry is a concurrent map keyed by a sorted tag list

A `Meter.Id` is `(name, Tags, baseUnit, description, type)`. `Tags` is an **immutable,
sorted, de-duplicated** list of `Tag` (key/value string pairs). Sorting matters: it means
`Tags.of("uri", "/orders", "method", "POST")` and `Tags.of("method", "POST", "uri", "/orders")`
produce an *equal* id and therefore the *same* meter. It also means constructing an id
costs a sort and a series of string comparisons.

Registration goes through a `ConcurrentHashMap<Meter.Id, Meter>` in
`MeterRegistry`. So:

```java
// This is a map lookup with a Tags construction and sort on EVERY call.
registry.counter("orderflow.orders.placed", "channel", channel).increment();

// This resolves the meter once. The hot path is then a single adder increment.
private final Counter webOrders =
    Counter.builder("orderflow.orders.placed").tag("channel", "web").register(registry);
// ... later, in the hot path:
webOrders.increment();
```

The difference is small per call and completely invisible until you are doing it inside a
loop over five million order lines. The rule is: **resolve meters in the constructor when
the tag values are known at construction time; look them up per call only when a tag value
is genuinely per-request** (and if it is per-request, ask yourself the cardinality
question before you finish typing).

Under the covers a `Counter` in the Prometheus registry is backed by a striped adder of
the `LongAdder` family (Topic 95). It is designed for many writers and one reader, which
is exactly the scrape pattern: N request threads increment, one scrape thread reads. That
is why a counter increment is not a contention problem even at high request rates —
whereas a naively shared `AtomicLong` would be.

**The registry retains every meter it has ever created for the life of the process.**
That is a `ConcurrentHashMap` that only grows. This is the same shape as Topic 79's
unbounded static map: a high-cardinality tag is *simultaneously* a Prometheus problem and
a Java heap leak. Prometheus usually falls over first because it holds every series from
every target, but do not let anyone tell you the application is unaffected — you can watch
its heap climb in the same drill.

### Histogram buckets: what is actually exported, and why aggregation needs them

A Micrometer `Timer` maintains, per meter id:

- a count and a total-time adder (always),
- a `TimeWindowMax` for the max over a rolling window (always),
- **optionally** a `TimeWindowPercentileHistogram` — an array of counters, one per bucket
  boundary, each holding "how many observations were ≤ this boundary". These are
  cumulative ("less-than-or-equal"), which is the Prometheus convention.

When you enable `publishPercentileHistogram()`, Micrometer emits a *fixed set* of bucket
boundaries chosen to cover a wide dynamic range with bounded error, clamped by
`minimumExpectedValue` / `maximumExpectedValue`. When you enable
`serviceLevelObjectives(...)`, it emits a bucket at exactly the boundaries you name. You
can use both; the SLO boundaries are merged in.

Here is the shape of the exposition output — *illustration of the format, not captured
output*:

```
# HELP http_server_requests_seconds
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds_bucket{method="POST",uri="/orders",status="200",le="0.05"} <n>
http_server_requests_seconds_bucket{method="POST",uri="/orders",status="200",le="0.1"} <n>
http_server_requests_seconds_bucket{method="POST",uri="/orders",status="200",le="0.3"} <n>
http_server_requests_seconds_bucket{method="POST",uri="/orders",status="200",le="+Inf"} <n>
http_server_requests_seconds_count{method="POST",uri="/orders",status="200"} <n>
http_server_requests_seconds_sum{method="POST",uri="/orders",status="200"} <n>
```

Now the arithmetic that everything else in this section depends on.

**A bucket count is a counter, and counters add.** If pod A saw 900 requests ≤ 300 ms and
pod B saw 700 requests ≤ 300 ms, the fleet saw 1600 requests ≤ 300 ms. That is simply
true. So you can sum the `_bucket` series across pods and *then* compute a quantile from
the summed histogram:

```promql
histogram_quantile(
  0.99,
  sum by (le, uri, method) (rate(http_server_requests_seconds_bucket[5m]))
)
```

**A percentile is not a counter, and percentiles do not add.** If pod A's p99 is 300 ms
and pod B's p99 is 900 ms, the fleet's p99 is not 600 ms. It is not any function of those
two numbers alone. To see why in one line: suppose pod A served 100,000 requests and pod
B served 100. The fleet p99 is essentially pod A's p99; the average of the two is wildly
wrong. Now suppose the reverse traffic split and the answer moves somewhere else entirely.
**The same two inputs give different correct answers depending on information the inputs
do not contain.** That is the definition of "not aggregatable".

This is why `publishPercentiles` is a trap in a multi-instance deployment. It produces
series like:

```
# illustration of the format, not captured output
http_server_requests_seconds{method="POST",uri="/orders",quantile="0.99"} <n>
```

…which are perfectly correct *for that one JVM* and which every dashboard author on earth
will then wrap in `avg()`.

**Client-side percentiles are still useful in exactly one place:** a single-instance
context where you want an accurate high quantile without choosing bucket boundaries — for
example a local JMH-adjacent experiment, or a per-pod debug panel that is explicitly
labelled per-pod. In a fleet dashboard they are wrong.

There is a second, subtler cost. `publishPercentileHistogram()` on a `Timer` emits **one
series per bucket boundary per tag combination**. The bucket count is not small. That
makes the histogram itself a cardinality multiplier, and it is the reason the cardinality
budget later in this document has a "× buckets" column.

### `[JAVA 25]` / Prometheus native histograms — what I am not sure of

Prometheus has a "native histogram" format that stores a sparse, exponentially-bucketed
distribution in a single series instead of one series per bucket, which would change the
cardinality arithmetic above substantially. Micrometer and the Prometheus Java client have
been moving toward supporting it.

**I am not going to state what is on by default in your Micrometer version, or what the
enabling property is called, because I do not know it reliably for the Boot 4.1 line.**
Settle it locally rather than trusting me:

```bash
# 1. What Prometheus client library version is actually resolved?
./mvnw dependency:tree -Dincludes=io.prometheus:*,io.micrometer:*

# 2. Does your scrape output contain classic _bucket series, native histograms, or both?
curl -s localhost:8080/actuator/prometheus | grep -c '_bucket{'
```

Everything else in this document is written for classic bucket histograms, which are
supported everywhere and are what you should ship until you have a specific reason not to.

### How the scrape actually happens

1. Prometheus issues `GET /actuator/prometheus` on your management port, on its
   `scrape_interval`.
2. Actuator's endpoint asks the `PrometheusMeterRegistry` to render every meter it holds
   into the text exposition format, into a response buffer.
3. Rendering walks the whole meter map, allocating strings. **Cost is proportional to
   series count.** This is a real, measurable slice of your service's CPU and allocation
   rate once series count is large — and it is *allocation on the scrape thread*, which is
   Topic 68's story, not a free operation.
4. Prometheus parses the payload and appends one sample per series to its head block. Its
   memory is roughly proportional to the number of *active* series across all targets.

Two consequences worth internalising:

- **A scrape is a request your service serves.** If it takes longer than Prometheus's
  `scrape_timeout`, the scrape fails and you get a gap in *all* metrics for that target —
  including the ones you would use to diagnose why.
- **Series that stop being written still cost.** Prometheus keeps a stale series in its
  head block until the staleness window passes. A burst of 200,000 one-off series does not
  become free the instant your traffic stops.

### Where the built-in `http.server.requests` tags come from

Boot's web instrumentation records a `Timer` per request through Micrometer's
`Observation` API. The default tags are `method`, `uri`, `status`, `outcome`, `exception`
(and `error` in some versions). The important one is `uri`: it is the **matched handler
template**, not the raw path. A request to `/orders/9f2a1c04` records `uri="/orders/{id}"`.

That single design decision is the only thing standing between you and unbounded
cardinality on your busiest meter, and it has two well-known holes:

- **404s and unmatched paths.** There is no template to report. Boot maps these to
  `uri="NOT_FOUND"` (and `uri="REDIRECTION"` for 3xx) precisely so that a scanner hitting
  ten thousand random URLs does not create ten thousand series. Verify this on your
  version rather than assuming; the drill below shows you how.
- **Anything you instrument by hand.** A `Timer` you build yourself gets exactly the tags
  you type. If you type the raw path, you have just re-created the problem the framework
  carefully avoided.

### `MeterFilter` — the enforcement mechanism

A `MeterFilter` is a bean that intercepts every meter registration and can accept, deny,
rename, re-tag, or attach distribution configuration. This is the **only** mechanism that
enforces a cardinality budget in code rather than in a wiki page.

```java
@Bean
MeterFilter orderflowCardinalityGuard() {
    return MeterFilter.maximumAllowableTags(
        "orderflow.orders.placed",  // meter name prefix
        "channel",                  // tag key to bound
        20,                         // how many distinct values are allowed
        MeterFilter.deny()          // what to do with the 21st: drop the meter
    );
}
```

There is also `MeterFilter.maximumAllowableMetrics(n)` (a global cap on meter count),
`MeterFilter.deny(id -> ...)` (predicate-based rejection), `MeterFilter.replaceTagValues(...)`
(map long-tail values to `"other"`), and `MeterFilter.ignoreTags("...")` (strip a tag
entirely).

**`replaceTagValues` is usually the right answer**, because it preserves the total while
bounding the series count: the top-20 payment providers keep their own series and
everything else lands in `other`, so your sum is still correct.

The mechanical point: a filter runs **at registration time**, which is the first time a
given tag combination is seen. It cannot un-create series that were created before you
deployed the filter — those live in Prometheus until they age out.

---

## Example 1 — minimal

A single counter and a single timer, with nothing else in the way, so you can see the
name transformation and the exposition output for yourself.

**Dependencies** (Maven):

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <scope>runtime</scope>
</dependency>
```

**Configuration:**

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when_authorized
```

> **Uncertainty, flagged in one line:** the property that toggles the Prometheus *exporter*
> itself has been renamed across Boot majors — `management.metrics.export.prometheus.enabled`
> on Boot 2.x, `management.prometheus.metrics.export.enabled` on Boot 3.x, and I am not
> certain it is unchanged on 4.1. You almost never need to set it (adding the registry jar
> is enough). If you do need it, settle the name with
> `curl -s localhost:8080/actuator/configprops | grep -i prometheus` or by reading your
> IDE's property completion, which is driven by the version's own metadata — do not trust
> a blog post or me.

**The code:**

```java
package com.orderflow.catalog;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

@Service
public class ProductLookupService {

    private final Counter lookups;
    private final Timer lookupTimer;
    private final ProductRepository repository;

    ProductLookupService(MeterRegistry registry, ProductRepository repository) {
        this.repository = repository;

        // Resolved once, in the constructor. No per-call tag construction.
        this.lookups = Counter.builder("orderflow.catalog.lookups")
                .description("Product detail lookups served")
                .register(registry);

        this.lookupTimer = Timer.builder("orderflow.catalog.lookup.duration")
                .description("Time to serve a product detail lookup")
                .publishPercentileHistogram()          // buckets — aggregatable
                .minimumExpectedValue(Duration.ofMillis(1))
                .maximumExpectedValue(Duration.ofSeconds(2))
                .register(registry);
    }

    public Product findBySku(String sku) {
        lookups.increment();
        return lookupTimer.record(() -> repository.findBySku(sku));
    }
}
```

**What to run:**

```bash
curl -s localhost:8080/api/products/SKU-1001 > /dev/null
curl -s localhost:8080/actuator/prometheus | grep orderflow_catalog
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `orderflow_catalog_lookups_total <n>` | The counter registered, and Micrometer snake-cased the dotted name and appended `_total` |
| `orderflow_catalog_lookup_duration_seconds_bucket{le="..."} <n>` — many lines | `publishPercentileHistogram()` is working; these are the aggregatable series |
| `_count` and `_sum` present, no `_bucket` lines | You forgot `publishPercentileHistogram()`. You have a rate and an average and no percentiles |
| A `quantile="0.99"` label instead of `le="..."` | You used `publishPercentiles`. This is the non-aggregatable shape |
| Nothing at all | The meter has never been registered — the constructor ran but nothing called the method, **or** the endpoint is not exposed. Check `/actuator` first |

**The one thing to notice:** the meter appeared under a *different name* than the one you
typed. `orderflow.catalog.lookup.duration` became
`orderflow_catalog_lookup_duration_seconds`. The `_seconds` suffix comes from the `Timer`'s
base unit. Get used to this now; it is the reason your first PromQL query returns nothing.

**Now do the wrong thing on purpose, briefly:**

```java
// Do NOT ship this. Added here so you can see the shape of the disaster.
lookupTimer = Timer.builder("orderflow.catalog.lookup.duration")
        .tag("sku", sku)              // 100,000 products in the Topic 65 dataset
        .publishPercentileHistogram() // × the number of buckets
        .register(registry);
```

Do not run this against the seeded catalogue yet. That is the failure drill, and it is
better done deliberately with the instruments in place.

---

## Example 2 — production scenario (on the project spine)

### The constraints

This is `orderflow` at the Topic 65 baseline, and the constraints are the real ones:

- **Dataset:** 100,000 products, 1,000,000 orders, 5,000,000 order lines.
- **Load:** the Topic 65 k6 mix — 70% catalogue read, 20% order read, 10% order placement,
  as a constant-arrival-rate open model, skewed toward hot products.
- **Deployment:** multiple `orderflow` pods behind a Kubernetes Service. **This is the
  fact that makes percentile aggregation a correctness issue rather than a style issue.**
- **Resources:** the pods run under the CPU and memory limits you set in Topic 82. Scrape
  rendering competes with request handling for those.
- **Existing instrumentation:** Hikari (Topic 109), the `ThreadPoolTaskExecutor`s from
  Topic 90, the Kafka consumer (Topic 113), the outbox relay (Topic 115), and the
  correlation/trace fields from Topics 120 and 119.
- **Prometheus is shared** with other teams' services. Your cardinality is not only your
  problem.

### Step 1 — common tags, applied once

Every series from this service should carry the labels that let you slice by deployment
without adding them at each call site.

```java
package com.orderflow.observability;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.config.MeterFilter;
import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class MetricsConfiguration {

    @Bean
    MeterRegistryCustomizer<MeterRegistry> commonTags(
            @Value("${spring.application.name}") String application,
            @Value("${ORDERFLOW_ENV:local}") String environment,
            @Value("${ORDERFLOW_VERSION:dev}") String version) {

        return registry -> registry.config().commonTags(
                "application", application,
                "env", environment,
                "version", version);
    }
}
```

**The judgement call, stated explicitly:** `version` is a bounded-but-churning tag. Every
deploy creates a fresh set of series for every meter, and the old ones linger until they
age out. That is a **deliberate** trade: it is what lets you say "p99 got worse at the
version boundary", which is the single most useful thing a metric can tell you during a
rollout. It is worth the churn. `pod` or `instance` is **not** added here — Prometheus
adds an `instance` label at scrape time, and duplicating it doubles nothing but confusion.

**What must never go in `commonTags`:** anything per-request. `commonTags` multiplies
*every meter in the process*, including the several hundred you did not write.

### Step 2 — RED on every endpoint, without writing per-endpoint code

Here is the part that surprises people coming from hand-rolled Node instrumentation: for
RED you write **no per-endpoint code at all**. `http.server.requests` already carries
rate, errors and duration, tagged by `uri`, `method`, `status` and `outcome`. What you
must do is make sure it publishes *buckets*, and that the buckets bracket your SLO.

```properties
# Buckets for the aggregatable percentiles.
management.metrics.distribution.percentiles-histogram.http.server.requests=true

# Clamp the bucket range so you do not pay for buckets covering durations you will
# never see. Pick these from YOUR Topic 65 baseline, not from this document.
management.metrics.distribution.minimum-expected-value.http.server.requests=5ms
management.metrics.distribution.maximum-expected-value.http.server.requests=10s

# Explicit SLO boundaries: guarantees a bucket edge exactly at the number the SLO
# is written against, so the "fraction under target" query is exact rather than
# interpolated.
management.metrics.distribution.slo.http.server.requests=100ms,300ms,500ms,1s,3s

# Do NOT do this in a multi-pod deployment:
# management.metrics.distribution.percentiles.http.server.requests=0.5,0.95,0.99
```

> **`[BOOT 3.x DELTA]`** These `management.metrics.distribution.*` property paths are the
> Boot 2.x/3.x spelling and, as far as I know, are unchanged on 4.1 — but I have not
> verified every one of them against 4.1's metadata. The programmatic equivalent below is
> version-independent and is what I would ship in a codebase that has to survive an
> upgrade. **Verify with `/actuator/configprops` on the version you are running**, and
> prefer the bean if there is any doubt.

The version-independent form, which also lets you apply different rules to different
meters:

```java
@Bean
MeterFilter httpDistributionConfig() {
    return new MeterFilter() {
        @Override
        public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
            if (!id.getName().equals("http.server.requests")) {
                return config;
            }
            return DistributionStatisticConfig.builder()
                    .percentilesHistogram(true)
                    .serviceLevelObjectives(
                            Duration.ofMillis(100).toNanos(),
                            Duration.ofMillis(300).toNanos(),
                            Duration.ofMillis(500).toNanos(),
                            Duration.ofSeconds(1).toNanos(),
                            Duration.ofSeconds(3).toNanos())
                    .minimumExpectedValue(Duration.ofMillis(5).toNanos())
                    .maximumExpectedValue(Duration.ofSeconds(10).toNanos())
                    .build()
                    .merge(config);
        }
    };
}
```

**The RED queries this makes possible** (these are the three panels every `orderflow`
endpoint gets):

```promql
# R — request rate per endpoint
sum by (uri, method) (rate(http_server_requests_seconds_count{application="orderflow"}[5m]))

# E — error rate as a FRACTION, which is what an SLO is written against
  sum by (uri) (rate(http_server_requests_seconds_count{application="orderflow",outcome=~"SERVER_ERROR"}[5m]))
/ sum by (uri) (rate(http_server_requests_seconds_count{application="orderflow"}[5m]))

# D — p99 across the whole fleet, computed from summed buckets
histogram_quantile(0.99,
  sum by (le, uri, method) (rate(http_server_requests_seconds_bucket{application="orderflow"}[5m])))

# D' — the SLO form: what fraction of order placements were under 300 ms?
  sum(rate(http_server_requests_seconds_bucket{uri="/orders",method="POST",le="0.3"}[5m]))
/ sum(rate(http_server_requests_seconds_count{uri="/orders",method="POST"}[5m]))
```

That last query is the one Topic 130 builds error budgets on. Note it needs a bucket edge
at exactly `0.3` — which is why `serviceLevelObjectives` exists and why you set it in
step 2 rather than hoping the default bucket set happens to have an edge there.

### Step 3 — the business meters that RED does not give you

RED tells you the HTTP layer is healthy. It does not tell you that orders stopped being
placed because every one of them is being rejected for insufficient stock with a
perfectly cheerful `200 OK` and a rejection body.

```java
package com.orderflow.orders;

@Service
public class OrderPlacementService {

    private final Counter placed;
    private final Counter rejectedOutOfStock;
    private final Counter rejectedInsufficientFunds;
    private final Counter rejectedIdempotentReplay;
    private final DistributionSummary linesPerOrder;
    private final Timer inventoryReservation;

    OrderPlacementService(MeterRegistry registry, /* ... */) {
        this.placed = Counter.builder("orderflow.orders.placed")
                .description("Orders successfully placed")
                .register(registry);

        // One meter NAME, one bounded 'reason' tag — not three names. This lets a
        // dashboard show the rejection breakdown with one query, and the tag has a
        // fixed, enumerable domain: it comes from a Java enum, not from user input.
        this.rejectedOutOfStock         = rejection(registry, "OUT_OF_STOCK");
        this.rejectedInsufficientFunds  = rejection(registry, "INSUFFICIENT_FUNDS");
        this.rejectedIdempotentReplay   = rejection(registry, "IDEMPOTENT_REPLAY");

        this.linesPerOrder = DistributionSummary.builder("orderflow.orders.lines")
                .description("Lines per placed order")
                .publishPercentileHistogram()
                .register(registry);

        this.inventoryReservation = Timer.builder("orderflow.inventory.reservation.duration")
                .description("Time to reserve inventory for one order")
                .publishPercentileHistogram()
                .register(registry);
    }

    private static Counter rejection(MeterRegistry registry, String reason) {
        return Counter.builder("orderflow.orders.rejected")
                .tag("reason", reason)     // bounded: one value per enum constant
                .register(registry);
    }
}
```

**The rule this demonstrates:** a tag value is safe when its domain is *declared in your
code* — an enum, a status code, an HTTP method, a boolean. It is unsafe when its domain
comes from *data* — an ID, an email, a SKU, a free-text error message, a URL path
segment, a customer name, a `Throwable`'s message.

That is a rule you can apply at code-review speed, and it is worth adopting as a team
standard (Topic 132's kind of artefact).

### Step 4 — USE on HikariCP

Boot registers Hikari's metrics automatically when a `DataSource` and a `MeterRegistry`
are both present. **You write no code — you write the dashboard and the alert.** The
meters (Prometheus names) are:

| Meter | USE role | What it tells you |
|---|---|---|
| `hikaricp_connections_active` | **Utilisation** | connections currently checked out |
| `hikaricp_connections_max` | denominator | pool size, so utilisation is `active / max` |
| `hikaricp_connections_pending` | **Saturation** | threads *blocked waiting* for a connection. **The most important pool metric you have** |
| `hikaricp_connections_timeout_total` | **Errors** | acquisition timeouts — requests that failed to get a connection at all |
| `hikaricp_connections_acquire_seconds` | Saturation (distribution) | how long acquisition takes — the leading indicator |
| `hikaricp_connections_usage_seconds` | diagnosis | how long a connection is *held*. Topic 55's HTTP-call-in-a-transaction shows up here first |

```promql
# Utilisation
  sum by (pool) (hikaricp_connections_active{application="orderflow"})
/ sum by (pool) (hikaricp_connections_max{application="orderflow"})

# Saturation — anything sustained above zero is a queue forming
sum by (pool) (hikaricp_connections_pending{application="orderflow"})

# Errors
sum by (pool) (rate(hikaricp_connections_timeout_total{application="orderflow"}[5m]))
```

**Why `pending` beats `active`:** `active` saturates at `max` and then tells you nothing
more. It looks identical whether one thread is waiting or two hundred are. `pending` is
the queue depth, and queue depth is the number that predicts latency (Topic 90's lesson,
in a different pool). If you only get one Hikari panel, make it `pending`.

**Connect it to Topic 109:** the pool-vs-thread-pool deadlock shows up here as `active ==
max` and `pending` climbing monotonically and never draining — because the threads holding
connections are themselves blocked waiting for connections. `usage_seconds` going to
"forever" at the same moment is the confirming signal. Two panels turn a thread-dump
investigation into a five-second diagnosis.

### Step 5 — USE on the Topic 90 executors

This one **does** need code, or at least a decision, because a bare `ExecutorService` has
no metrics at all.

```java
package com.orderflow.observability;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics;

@Configuration
class ExecutorMetricsConfiguration {

    /**
     * The Topic 90 executor: bounded queue, explicit rejection policy, named threads.
     * Wrapping it in ExecutorServiceMetrics is what turns it from an invisible
     * resource into a USE-instrumented one.
     */
    @Bean
    ExecutorService notificationExecutor(MeterRegistry registry) {
        ThreadPoolExecutor delegate = new ThreadPoolExecutor(
                8, 8,
                60L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(500),                 // BOUNDED (Topic 90)
                new CustomizableThreadFactory("orderflow-notify-"),
                new ThreadPoolExecutor.AbortPolicy());         // explicit rejection

        return ExecutorServiceMetrics.monitor(
                registry, delegate, "orderflow-notify",
                Tags.of("purpose", "notification"));
    }
}
```

`ExecutorServiceMetrics.monitor` returns a *wrapped* executor. **You must use the returned
instance.** Submitting to `delegate` after wrapping records nothing — a mistake that
produces a perfectly flat, perfectly wrong dashboard.

The meters it registers, mapped to USE:

| Meter | USE role | Notes |
|---|---|---|
| `executor_active_threads` | **Utilisation** | against `executor_pool_size_threads` |
| `executor_queued_tasks` | **Saturation** | the number that matters |
| `executor_queue_remaining_tasks` | Saturation (inverse) | how much headroom before rejection |
| `executor_seconds_count` / `_sum` / `_bucket` | throughput + duration | a `Timer` per executed task |
| `executor_completed_tasks_total` | throughput | monotonic |

**What is missing, and it is important:** there is no built-in counter for *rejections*.
`AbortPolicy` throws `RejectedExecutionException` and Micrometer does not see it. Rejection
is the "E" in USE for a thread pool and it is the single most actionable signal a bounded
pool produces. Wire it yourself:

```java
class MeteredAbortPolicy implements RejectedExecutionHandler {
    private final Counter rejections;

    MeteredAbortPolicy(MeterRegistry registry, String poolName) {
        this.rejections = Counter.builder("orderflow.executor.rejected")
                .tag("pool", poolName)     // bounded: one value per pool
                .description("Tasks rejected because the bounded queue was full")
                .register(registry);
    }

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        rejections.increment();
        throw new RejectedExecutionException("orderflow pool saturated");
    }
}
```

**`[JAVA 25]` / virtual threads (Topic 101):** if a path runs on a virtual-thread executor,
**there is no pool to instrument.** `executor_active_threads` and `executor_queued_tasks`
are meaningless or absent: the whole point of virtual threads is that there is no bounded
worker set and no queue. Saturation moves somewhere else — to the *semaphore or rate
limiter you put in front of the downstream call*, and to the downstream resource itself
(the Hikari pool, the gateway's concurrency limit). So:

> **Rule:** every unbounded concurrency mechanism needs an explicit, instrumented bound
> somewhere, or you have no saturation signal at all. On virtual threads, instrument the
> `Semaphore` permits in use — that is your new USE numerator.

### Step 6 — the meters that only exist because of Phases 10–11

These are the ones a generic "add Micrometer" ticket will never produce, and they are the
ones that make an `orderflow` incident diagnosable.

```java
// Topic 115 — the outbox relay. A Timer records NOTHING while the relay is stuck,
// so the stuck case needs a LongTaskTimer.
LongTaskTimer relayBatch = LongTaskTimer.builder("orderflow.outbox.relay.batch")
        .description("Duration of the in-flight outbox relay batch")
        .register(registry);

// Outbox depth: the single best "are we falling behind" signal in the whole service.
Gauge.builder("orderflow.outbox.pending", outboxRepository, OutboxRepository::countPending)
        .description("Unpublished rows in the outbox table")
        .strongReference(true)     // see Trap 4 — do not skip this
        .register(registry);

// Topic 111 — the circuit breaker state, as a number you can alert on.
// resilience4j-micrometer registers these for you; the point is to KNOW they exist.
//   resilience4j_circuitbreaker_state{name="payment-gateway",state="open"} 1
//   resilience4j_circuitbreaker_calls_seconds_count{kind="failed"} <n>
```

And the two USE-shaped signals that are not pools:

| Signal | Meter | Why it belongs in USE |
|---|---|---|
| Kafka consumer lag (Topic 113) | `kafka_consumer_fetch_manager_records_lag_max` | Lag **is** the saturation of a consumer group |
| Rebalance rate (Topic 113) | `kafka_consumer_coordinator_rebalance_rate_per_hour` (name varies by client version — check yours) | A rebalance storm is invisible in RED and obvious here |

### Step 7 — the cardinality budget, written down

This is the artefact. It goes in the repository next to the Topic 65 baseline, and Topic
124's gate review references it.

**Fill in — cardinality budget (blank template — every cell is yours to compute or
measure):**

| Meter name | Tag keys | Distinct values per key (measured) | Buckets (if histogram) | Series = product × buckets | Bounded by what? |
|---|---|---|---|---|---|
| `http.server.requests` | method, uri, status, outcome | | | | URI templating + `MeterFilter` cap |
| `orderflow.orders.placed` | (common only) | | | | nothing needed |
| `orderflow.orders.rejected` | reason | | | | Java enum |
| `orderflow.orders.lines` | (common only) | | | | |
| `orderflow.inventory.reservation.duration` | (common only) | | | | |
| `hikaricp.connections.*` | pool | | | | one pool |
| `executor.*` | name | | | | number of declared executors |
| `orderflow.executor.rejected` | pool | | | | number of declared executors |
| `orderflow.outbox.pending` | (common only) | | | | |
| **Common tags multiplier** | application, env, version | | | ×  | version churns per deploy |
| **TOTAL for this service** | | | | | |

The last row is the number you take to whoever owns Prometheus. If you cannot fill it in,
you do not know what you are asking them to store.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a high-cardinality tag: the order ID

**Wrong approach**

```java
// Reviewed, approved, and shipped, because "it's just one tag and it'll be so useful
// for debugging."
Timer.builder("orderflow.orders.placement.duration")
        .tag("orderId", order.id().toString())
        .tag("customerId", order.customerId().toString())
        .publishPercentileHistogram()
        .register(registry)
        .record(elapsed);
```

**Exact symptom** — in the order you actually meet it:

1. Nothing at all in staging. Twelve orders were placed; twelve series is nothing.
2. In production, the `/actuator/prometheus` response body grows steadily. Nobody looks at
   a response body.
3. Prometheus's scrape of this target starts taking longer, then exceeds `scrape_timeout`.
   `up{job="orderflow"}` flips to 0 **for the whole target**, so *every* metric from the
   service has gaps — including the ones you would use to notice.
4. Prometheus's own resident memory climbs. Queries across other teams' services get slow.
   Eventually Prometheus is OOMKilled and restarts, losing its head block.
5. Meanwhile the application's heap is also growing, because the meter registry retains
   every one of those `Timer` objects with their bucket arrays. If nobody notices for long
   enough, you get a Topic 79 heap dump with `ConcurrentHashMap` nodes holding `Meter.Id`s
   at the top of the dominator tree.

**Root cause**

Time-series identity is `(name, label set)`. `orderId` has a distinct value per order —
1,000,000 in the seeded dataset alone, unbounded in reality. Multiply by the histogram
bucket count, then by `customerId`, and the series count is the product. Nothing in the
pipeline bounds it, because a monitoring system cannot know that "order ID" is different
from "HTTP status".

**Fix**

1. **Delete the tag.** The per-order duration is not a metric; it is a *trace* (Topic 119)
   or a *log line* (Topic 120). This is the load-bearing distinction:

   | Question | Right tool |
   |---|---|
   | "How is the p99 of order placement trending?" | metric |
   | "Why was *this* order slow?" | trace |
   | "What exactly happened during order `9f2a…`?" | log, joined by correlation ID |

   High-cardinality identity belongs on the *unaggregated* signals, where the storage
   model is per-event, not per-series. Putting it on the aggregated signal is a category
   error.

2. **Make the mistake un-shippable**, because a rule that lives only in review will be
   broken by the next person:

```java
@Bean
MeterFilter denyHighCardinalityTags() {
    Set<String> forbidden = Set.of(
            "orderId", "order_id", "customerId", "customer_id",
            "userId", "user_id", "sku", "email", "paymentId", "idempotencyKey");

    return MeterFilter.denyUnless(id ->
            id.getTags().stream().noneMatch(t -> forbidden.contains(t.getKey())));
}

@Bean
MeterFilter globalMeterCeiling() {
    // A blunt instrument, deliberately. If a deploy pushes the process past this,
    // new meters are dropped rather than the monitoring system dying.
    return MeterFilter.maximumAllowableMetrics(3_000);
}
```

3. **Add a test**, so the budget is enforced by CI and not by memory:

```java
@SpringBootTest
class CardinalityBudgetTest {

    @Autowired MeterRegistry registry;
    @Autowired MockMvc mvc;

    @Test
    void hittingDistinctOrderIdsDoesNotCreateDistinctSeries() throws Exception {
        int before = registry.getMeters().size();

        for (int i = 0; i < 200; i++) {
            mvc.perform(get("/api/orders/" + UUID.randomUUID()));
        }

        int after = registry.getMeters().size();
        assertThat(after - before)
                .as("200 distinct order IDs must not create 200 meters")
                .isLessThan(10);
    }
}
```

**The sentence to remember:** *this trap kills the monitoring system, not the application.*
The application keeps serving orders while you lose the ability to see it doing so.

---

### Trap 2 — tracking average response time

**Wrong approach**

A dashboard panel, or an alert, built on:

```promql
  rate(http_server_requests_seconds_sum[5m])
/ rate(http_server_requests_seconds_count[5m])
```

…labelled "avg response time", with a threshold alert at some round number. Sometimes it
is worse: a hand-rolled `Gauge` holding a running mean the application computes itself.

**Exact symptom**

- The graph is beautifully flat during an incident where a meaningful slice of your users
  are timing out.
- Support tickets and the graph disagree, and the graph wins the argument, so the
  investigation starts in the wrong place and stays there.
- After the incident, the postmortem timeline shows the alert firing many minutes after
  users noticed — or not firing at all.
- The specific `orderflow` version: a GC pause (Topic 71) or a safepoint stall (Topic 73)
  hits a small fraction of requests very hard. The mean moves by a rounding error.

**Root cause**

The mean is a single number summarising a distribution, and it is dominated by the bulk.
Under the tail-latency failure modes that make up essentially every production incident,
the bulk is *fine*. That is what makes the failure a tail failure. Alerting on the mean is
alerting on the part of the distribution that is not broken.

There is also a mechanical version of the same problem: `sum/count` cannot answer "what
fraction of requests were under 300 ms", which is the only shape an SLO can take.

**Fix**

Publish buckets and alert on a quantile or on an SLO fraction:

```promql
# Alert on the tail, per endpoint.
histogram_quantile(0.99,
  sum by (le, uri) (rate(http_server_requests_seconds_bucket{application="orderflow"}[5m])))
> 0.3

# Or, better, alert on SLO burn (Topic 130 develops this properly).
1 - (
    sum(rate(http_server_requests_seconds_bucket{uri="/orders",method="POST",le="0.3"}[5m]))
  / sum(rate(http_server_requests_seconds_count{uri="/orders",method="POST"}[5m]))
) > 0.01
```

**Keep the average for exactly one thing:** it is a cheap, always-available sanity check
and it is the correct input to a *capacity* calculation (Topic 129: total service time =
mean × rate is Little's Law's numerator). Averages are for capacity. Percentiles are for
experience. Do not let one do the other's job.

---

### Trap 3 — averaging p99 across pods

**Wrong approach**

Client-side percentiles are enabled…

```properties
management.metrics.distribution.percentiles.http.server.requests=0.5,0.95,0.99
```

…and the dashboard does the obvious thing with the resulting series:

```promql
avg(http_server_requests_seconds{quantile="0.99", uri="/orders"})
```

**Exact symptom**

- The dashboard p99 and the k6 client-side p99 from your Topic 65 run **disagree**, and
  the dashboard is consistently the optimistic one.
- The disagreement gets *worse* as you scale out: adding pods makes the graph look better
  while user experience is unchanged.
- During a partial failure — one pod degraded by a GC pause, a noisy neighbour, or a slow
  disk — the fleet dashboard barely moves, because five healthy pods dilute one sick one.
- You cannot produce a "p99 for the last 24 hours" number, because `avg_over_time` on a
  quantile series is a second layer of the same error.

**Root cause**

A quantile is a rank statistic. It is not a sum, so it is not additive; averaging it is
arithmetic that has no meaning. The correct fleet p99 requires the *merged distribution*,
and a per-pod `quantile="0.99"` sample has thrown the distribution away — it is one number
per pod per scrape, with the request counts and the shape discarded.

The scale-out effect has a simple explanation: `avg()` weights every pod equally
regardless of how many requests it served, so a low-traffic pod's fast p99 gets the same
vote as a high-traffic pod's slow one.

**Fix**

```properties
# Off: not aggregatable.
# management.metrics.distribution.percentiles.http.server.requests=0.5,0.95,0.99

# On: aggregatable.
management.metrics.distribution.percentiles-histogram.http.server.requests=true
```

```promql
histogram_quantile(0.99,
  sum by (le, uri, method) (rate(http_server_requests_seconds_bucket[5m])))
```

Read the query inside-out, because the order is the whole point:

1. `rate(..._bucket[5m])` — per-second increase of each bucket counter, per pod.
2. `sum by (le, ...)` — **merge the pods**, bucket by bucket. This is legal because bucket
   counts add. The `instance` label disappears here; that is intentional.
3. `histogram_quantile(0.99, ...)` — compute the quantile from the merged histogram.

**Aggregate first, quantile last.** Reverse those two steps and the number is wrong.

**And know the honest cost:** `histogram_quantile` linearly interpolates *within* the
bucket that contains the quantile. Your p99 is therefore accurate to the width of that
bucket. This is why `serviceLevelObjectives` matters — a bucket edge at exactly your SLO
target makes the "fraction under target" query exact even though the interpolated
percentile is approximate. Someone will ask you about this in an interview; the correct
answer is "approximate, bounded by bucket width, and that is a fine trade for
aggregatability."

---

### Trap 4 — the gauge that quietly disappears

**Wrong approach**

```java
// Looks right. Compiles. Works in the test. Vanishes in production.
public void registerOutboxDepth(MeterRegistry registry) {
    OutboxStats stats = new OutboxStats(dataSource);
    Gauge.builder("orderflow.outbox.pending", stats, OutboxStats::countPending)
            .register(registry);
    // `stats` goes out of scope here. Nothing else references it.
}
```

**Exact symptom**

- The series exists after startup, then reports `NaN` — or disappears from the scrape
  output entirely — some minutes or hours later.
- It reliably comes back after a restart and disappears again. It disappears *sooner*
  under memory pressure.
- The Grafana panel shows a line that stops. Nobody notices, because a missing line looks
  like "nothing to report" and your alert is `> 1000`, which never fires on `NaN`.
- **It is the outbox-depth gauge that vanishes, which is the one signal that would have
  told you the relay stopped.**

**Root cause**

`Gauge` holds a **weak reference** to the object it samples. This is deliberate: a gauge
must not be the reason an object stays alive, or every gauge would be a memory leak by
construction. But it means that once the last strong reference is gone, the object is
collectable, and after the next GC the gauge has nothing to sample.

The reason it works in the test is that a short test never GCs. The reason it fails under
memory pressure is that memory pressure is exactly when GC runs. This is a genuinely
Java-specific trap with no `prom-client` analogue whatsoever — in Node the closure keeps
the object alive.

**Fix**

Keep a strong reference — and prefer to make it obvious rather than incidental:

```java
@Component
class OutboxMetrics {

    private final OutboxRepository repository;   // strong reference: this bean is a singleton

    OutboxMetrics(MeterRegistry registry, OutboxRepository repository) {
        this.repository = repository;
        Gauge.builder("orderflow.outbox.pending", repository, OutboxRepository::countPending)
                .description("Unpublished rows in the outbox table")
                .strongReference(true)   // explicit, so nobody 'tidies up' the field later
                .register(registry);
    }
}
```

**Second half of the fix, and it is the more important half:** a gauge whose lambda hits
the database is executed **on the scrape thread, on every scrape**. `SELECT count(*)` over
a large outbox table, every fifteen seconds, forever, is a query you did not intend to
write. Sample it on a schedule into an `AtomicLong` and gauge the field:

```java
private final AtomicLong pending = new AtomicLong();

OutboxMetrics(MeterRegistry registry, OutboxRepository repository) {
    this.repository = repository;
    Gauge.builder("orderflow.outbox.pending", pending, AtomicLong::get).register(registry);
}

@Scheduled(fixedDelay = 15_000)
void sample() {
    pending.set(repository.countPending());   // bounded, cheap, off the scrape path
}
```

**General rule: a gauge lambda must be cheap and non-blocking.** It runs while Prometheus
is waiting for your response, and if it is slow, it makes the scrape slow, and a slow
scrape times out and takes *all* your metrics with it.

---

### Trap 5 — hand-rolled instrumentation that puts the raw path in the tag

**Wrong approach**

```java
// A filter someone added to "measure our downstream calls properly".
@Override
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    long start = System.nanoTime();
    try {
        chain.doFilter(req, res);
    } finally {
        registry.timer("orderflow.http.duration",
                       "path", ((HttpServletRequest) req).getRequestURI())   // RAW PATH
              .record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
    }
}
```

The same bug in its other common disguise, on the client side:

```java
// RestClient / WebClient with the URI already interpolated: the metric tag is the
// full concrete URL, so every product ID is a new series.
restClient.get()
        .uri("https://pricing.internal/api/products/" + sku)   // no template
        .retrieve();
```

**Exact symptom**

- Series count grows in proportion to *distinct paths hit*, which under the Topic 65 load
  means in proportion to distinct product SKUs — up to 100,000.
- A vulnerability scanner or a crawler probing random URLs produces a series spike with no
  corresponding traffic spike, which is a strange and memorable shape.
- `promtool check metrics` on your scrape output flags nothing; this is not a *format*
  problem. Only counting series finds it.
- Two duration metrics now exist for the same requests (`http_server_requests_seconds` and
  `orderflow_http_duration_seconds`) and they disagree, because they measure at different
  points in the filter chain. Two disagreeing numbers is worse than one number.

**Root cause**

The framework's `uri` tag is the *matched route template* precisely because the raw path
is unbounded. Hand-rolled instrumentation gets exactly the tags you type, and the raw path
is the easiest thing to type. On the client side, `RestClient`/`WebClient` metrics tag by
`uri` too — but they can only report a *template* if you gave them one.

**Fix**

1. Delete the filter. `http.server.requests` already exists and is already templated.
2. If you genuinely need a custom server-side tag, add it through the `Observation` API's
   convention mechanism rather than a parallel meter — one meter, extra tag, still
   templated, still bounded.
3. On the client side, **always pass a URI template with variables**, so the metric can
   record the template and substitute only for the actual request:

```java
restClient.get()
        .uri("https://pricing.internal/api/products/{sku}", sku)   // template preserved
        .retrieve();
```

4. Add a scrape-output assertion to CI so a future filter cannot reintroduce it:

```bash
# Fails the build if any label value looks like a raw identifier.
curl -s localhost:8080/actuator/prometheus \
  | grep -E '(uri|path)="[^"]*/[0-9a-f]{8}-' \
  && { echo "FAIL: raw identifier in a metric label"; exit 1; }
```

---

## Hands-on proof

Every step here is a command you run against your own service. No numbers are supplied;
every number in the tables is one you produce.

### Setup

```bash
# The Topic 65 stack, running, seeded, and warm.
docker compose -f load/docker-compose.yml up -d --wait
docker compose -f load/docker-compose.yml ps        # every service (healthy)

# Confirm the endpoint exists at all before anything else.
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/actuator/prometheus
```

### Proof 1 — what is actually being measured, before you add anything

```bash
# Every meter NAME the registry currently holds (Micrometer's dotted names).
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | sort

# How many, and how many are yours?
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | wc -l
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | grep -c '^orderflow\.'
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Dozens of `jvm.*`, `system.*`, `tomcat.*`, `hikaricp.*` names | Boot's binders are active. You did not write these and you are paying to export them |
| `http.server.requests` present | RED's raw material exists. Confirm it has buckets in Proof 2 |
| No `hikaricp.*` | Metrics were registered before the `MeterRegistry` bean existed, or the pool is not Hikari. Check bean ordering |
| Very few `orderflow.*` names | You have infrastructure metrics and no *service* metrics. That is the gap this topic closes |

### Proof 2 — the drill-down endpoint, and what the tags are

```bash
# Aggregate across all tag combinations.
curl -s localhost:8080/actuator/metrics/http.server.requests | jq

# Available tag keys and their values — this is your cardinality reconnaissance.
curl -s localhost:8080/actuator/metrics/http.server.requests | jq '.availableTags'

# Slice by one endpoint.
curl -s 'localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/orders' | jq
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `availableTags` lists `uri` with a short, template-shaped list (`/api/orders/{id}`) | Templating is working. Cardinality is bounded by route count |
| `uri` values containing concrete IDs | Something is bypassing templating — a filter, a `Handler` without a mapping, or a manually recorded timer. Find it now |
| `uri` value `NOT_FOUND` present | Good — unmatched requests are being collapsed into one series as intended |
| `measurements` shows only `COUNT`, `TOTAL_TIME`, `MAX` | You have no distribution. Only an average is computable from this |

### Proof 3 — confirm the histogram shape

```bash
# Aggregatable form: le= labels.
curl -s localhost:8080/actuator/prometheus \
  | grep 'http_server_requests_seconds_bucket' | head -20

# Non-aggregatable form: quantile= labels. If this returns lines, read Trap 3 again.
curl -s localhost:8080/actuator/prometheus \
  | grep 'http_server_requests_seconds{' | grep quantile

# Is there a bucket edge at your SLO target?
curl -s localhost:8080/actuator/prometheus \
  | grep 'http_server_requests_seconds_bucket' | grep 'le="0.3"'
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| Many `le="..."` lines per endpoint | Buckets are on. Fleet-wide `histogram_quantile` will be correct |
| `quantile="0.99"` lines | Client-side percentiles are on. Correct per pod, wrong when averaged. Decide deliberately |
| Both shapes present | You are paying for both. Usually a leftover from a migration — pick one |
| No bucket at your SLO boundary | Add `serviceLevelObjectives`; your "fraction under target" query is currently interpolated |

### Proof 4 — count your series, and time your scrape

This is the measurement that makes cardinality real rather than theoretical.

```bash
# Total series exposed (every non-comment line is one series).
curl -s localhost:8080/actuator/prometheus | grep -vc '^#'

# The top meter families by series count — your cardinality hot list.
curl -s localhost:8080/actuator/prometheus \
  | grep -v '^#' \
  | sed 's/[{ ].*//' \
  | sort | uniq -c | sort -rn | head -20

# Payload size.
curl -s localhost:8080/actuator/prometheus | wc -c

# How long the scrape takes — this is a request YOUR service serves.
curl -s -o /dev/null -w 'time_total=%{time_total}s size=%{size_download}B\n' \
  localhost:8080/actuator/prometheus

# Lint the exposition format (naming, HELP/TYPE consistency, unit suffixes).
curl -s localhost:8080/actuator/prometheus | promtool check metrics
```

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| One meter family dominating the `uniq -c` list | That is where a cardinality problem will start. Usually a histogram × a tag |
| Series count that **rises while you run load** | A tag value is coming from request data. This is the bug, caught early |
| Series count flat under load | Cardinality is bounded by code, not by traffic. This is the state you want |
| `promtool` complaining about counters lacking `_total` | A naming-convention deviation — usually a hand-registered meter |
| Scrape `time_total` a meaningful fraction of your `scrape_timeout` | You are close to the cliff. Reduce series or raise the timeout *and* understand why |

### Proof 5 — see the meter registry from inside the JVM

```bash
# Heap histogram: how much is the registry itself holding?
jcmd $(pgrep -f orderflow) GC.class_histogram | head -30

# Meter count as a metric, if you register one (recommended):
#   Gauge.builder("orderflow.meters.count", registry, r -> r.getMeters().size())
curl -s localhost:8080/actuator/metrics/orderflow.meters.count | jq
```

Registering a gauge for your own meter count is a small, permanent, extremely cheap
insurance policy. It turns "our monitoring broke" into an alertable number.

**WHAT TO LOOK FOR**

| What you see | What it means |
|---|---|
| `io.micrometer.core.instrument.Meter$Id` instance count climbing over a load run | Meters are being created from request data. Confirm which with Proof 4's family list |
| `Meter$Id` count flat | Registration happens at startup and stays put. Correct |
| Meter count near your `maximumAllowableMetrics` ceiling | The ceiling is about to start silently dropping meters. Investigate before it does |

### Proof 6 — prove the aggregation rule to yourself

Run two `orderflow` instances against the Topic 65 load, with deliberately different
conditions — for example, constrain one with `--cpus` (Topic 82) so its latency is worse.

```promql
# The wrong number.
avg(http_server_requests_seconds{quantile="0.99", uri="/api/orders"})

# The right number.
histogram_quantile(0.99,
  sum by (le) (rate(http_server_requests_seconds_bucket{uri="/api/orders"}[5m])))
```

Compare both against k6's own client-side p99 for the same window, which is measured
across all instances at once and is therefore the ground truth here.

**Fill in — aggregation check (blank template):**

| Source | p99 for `POST /api/orders` |
|---|---|
| k6 client-side (ground truth) | |
| `histogram_quantile` over summed buckets | |
| `avg()` of per-pod `quantile="0.99"` series | |
| `max()` of per-pod `quantile="0.99"` series | |

**How to read your own table:** the bucket-derived value should land close to k6's, within
bucket-width interpolation error plus the difference between server-measured and
client-measured time (network and queueing). The `avg()` value should be visibly
optimistic. `max()` is not correct either — it is the *worst pod's* p99, not the fleet's —
but it fails safe, which is why some teams use it as a stopgap. Say that out loud in an
interview and you will sound like someone who has actually run this.

---

## Failure drill

**Assigned drill:** tag a metric with the order ID, run the Topic 65 load, watch series
count and scrape memory explode.

This is a deliberate-breakage exercise. Run it in your local compose stack, never against
a shared Prometheus. **Read the whole drill before starting it**, because the measurements
you take *before* breaking anything are the entire point.

### The scenario

`orderflow` is at the Topic 65 baseline. A well-meaning change adds an order ID tag to the
order-placement timer, "so we can find slow orders". The change is one line and passes
review. You are going to ship it on purpose and watch what it does.

### Part A — record the healthy state first

You cannot see an explosion without a "before". Take these readings while the service is
warm and under the Topic 65 load, and write them down.

```bash
# 1. Series count
curl -s localhost:8080/actuator/prometheus | grep -vc '^#'

# 2. Scrape payload size and duration
curl -s -o /dev/null -w 'time=%{time_total}s size=%{size_download}\n' \
  localhost:8080/actuator/prometheus

# 3. Meter count inside the registry
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | wc -l

# 4. Application heap used
curl -s 'localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap' | jq

# 5. Prometheus's own head series and memory (from Prometheus's /metrics)
curl -s localhost:9090/metrics | grep -E '^prometheus_tsdb_head_series |^process_resident_memory_bytes '

# 6. Prometheus's view of this target
#    (run in the Prometheus expression browser)
#    scrape_duration_seconds{job="orderflow"}
#    scrape_samples_scraped{job="orderflow"}
#    up{job="orderflow"}
```

**Fill in — before (blank template):**

| Reading | Value |
|---|---|
| Series exposed by `/actuator/prometheus` | |
| Scrape payload size (bytes) | |
| Scrape duration (`curl` `time_total`) | |
| `scrape_duration_seconds{job="orderflow"}` | |
| `scrape_samples_scraped{job="orderflow"}` | |
| Meter names in the registry | |
| Application heap used | |
| `prometheus_tsdb_head_series` | |
| Prometheus `process_resident_memory_bytes` | |
| Configured `scrape_interval` / `scrape_timeout` | |

### Part B — break it

```java
// com.orderflow.orders.OrderPlacementService — the deliberate defect.
public OrderResult place(PlaceOrderCommand command) {
    Timer.Sample sample = Timer.start(registry);
    try {
        return doPlace(command);
    } finally {
        sample.stop(Timer.builder("orderflow.orders.placement.duration")
                .tag("orderId", command.orderId().toString())   // <-- THE DEFECT
                .publishPercentileHistogram()                   // <-- the multiplier
                .register(registry));
    }
}
```

Make sure your cardinality guard filters from Trap 1 are **not** present for this run —
the drill is about seeing the failure, and you will re-enable them as the fix.

Deploy, then run the Topic 65 load at its normal rate. Sample every reading from Part A
**every 60 seconds** while the load runs. Do not stop at the first sign of trouble; let it
run long enough to see the curve's shape.

```bash
# A sampler loop. Redirect to a file and graph it afterwards.
while true; do
  ts=$(date +%s)
  series=$(curl -s localhost:8080/actuator/prometheus | grep -vc '^#')
  timing=$(curl -s -o /dev/null -w '%{time_total} %{size_download}' localhost:8080/actuator/prometheus)
  echo "$ts $series $timing"
  sleep 60
done
```

### Part C — what to capture

| Capture | Command / source |
|---|---|
| Series count over time | the sampler loop above |
| Scrape duration over time | the sampler loop above |
| Prometheus head series | `prometheus_tsdb_head_series` |
| Prometheus RSS | `process_resident_memory_bytes` in Prometheus's own `/metrics` |
| Scrape failures | `up{job="orderflow"}` and `scrape_duration_seconds` |
| Application heap | `jvm.memory.used{area="heap"}` |
| Registry contents | `curl -s localhost:8080/actuator/prometheus \| grep -c 'orderflow_orders_placement_duration'` |
| Prometheus logs | `docker compose logs prometheus` — look for sample-limit or OOM messages |

**Fill in — during and after (blank template):**

| Reading | t+0 | t+5 min | t+15 min | t+30 min |
|---|---|---|---|---|
| Series exposed | | | | |
| Series for the defective meter only | | | | |
| Scrape duration | | | | |
| Scrape payload size | | | | |
| `scrape_samples_scraped` | | | | |
| `up{job="orderflow"}` (1 or 0) | | | | |
| `prometheus_tsdb_head_series` | | | | |
| Prometheus RSS | | | | |
| Application heap used | | | | |
| Application p99 for `POST /api/orders` (k6) | | | | |

### Part D — how to read what you captured

| What you see | What it means |
|---|---|
| Series count rising roughly linearly with orders placed | Confirmed: one series per order. Multiply by the bucket count to explain the slope |
| Scrape duration rising with series count | Rendering is O(series). Your service is now spending real CPU serializing metrics |
| `up` flipping to 0 while the app serves traffic normally | **The headline.** Monitoring failed; the application did not. Every metric for this target now has gaps |
| `scrape_samples_scraped` jumping then going to 0 | Prometheus hit a sample limit or the scrape timed out |
| Prometheus RSS rising faster than the app's heap | Prometheus holds all targets' series; you hold only yours. This is why it dies first |
| App heap also rising, with `Meter$Id` prominent in `GC.class_histogram` | The registry leak — Topic 79's shape. Confirms the app is not unharmed, just slower to fail |
| Application p99 unchanged for most of the run | The user-visible service is fine. That is precisely what makes this failure mode so dangerous |
| Prometheus restarting and coming back with a low head-series count | It was OOMKilled and lost its head block. Data before the last checkpoint may be gone |

### Part E — fix it and re-prove

1. Remove the `orderId` tag.
2. Restore the `MeterFilter` guards from Trap 1 (`denyUnless` + `maximumAllowableMetrics`).
3. **Restart the application** — a filter cannot delete meters that already exist.
4. Restart Prometheus, or wait out the staleness window, so the old series are gone.
5. Re-run the identical load and re-take every Part A reading.

**Fill in — after the fix (blank template):**

| Reading | Before defect | With defect (peak) | After fix |
|---|---|---|---|
| Series exposed | | | |
| Scrape duration | | | |
| `prometheus_tsdb_head_series` | | | |
| Prometheus RSS | | | |
| Application heap used | | | |
| `up{job="orderflow"}` stability | | | |

### Part F — three variations worth running

1. **Bounded-but-large.** Tag with `sku` instead of `orderId`. Now cardinality is bounded
   at 100,000 rather than unbounded. Does that make it safe? Compute the series count with
   buckets and common tags multiplied in *before* you run it, then check your prediction.
   The lesson: "bounded" is not the test; "bounded *small enough*" is.
2. **Cardinality without a histogram.** Same `orderId` tag on a plain `Counter`. The slope
   changes by exactly the bucket multiplier. This isolates how much of the damage the
   histogram contributes.
3. **The filter as a safety net.** Re-introduce the defect *with* `maximumAllowableTags`
   set on that meter. Watch series count rise to the cap and stop. Then answer the harder
   question: is a silently truncated meter better or worse than a loud failure? Write down
   your answer — Topic 124 will ask you to defend it.

### Part G — write it up

Three sentences, in the form Topic 133 will demand:

- **Contributing factors** (not "root cause"): what in the *system* allowed a one-line
  change to take down monitoring? At minimum: no cardinality guard, no series-count alert,
  no review checklist item, no staging load that would reveal it.
- **Detection:** what fired, how long after the deploy, and what would have fired sooner?
- **Prevention that scales:** a `MeterFilter` in the shared configuration, a CI assertion,
  and an alert on `prometheus_tsdb_head_series` growth rate — three mechanisms, each
  catching it at a different stage.

---

## Measurement

### The instrument for each claim

Never state one of these claims without the instrument beside it.

| Claim | Instrument | Command / query |
|---|---|---|
| "Our p99 for order placement is X" | Prometheus bucket aggregation | `histogram_quantile(0.99, sum by (le) (rate(http_server_requests_seconds_bucket{uri="/api/orders",method="POST"}[5m])))` |
| "X% of requests meet the latency target" | SLO bucket ratio | `sum(rate(..._bucket{le="0.3"}[5m])) / sum(rate(..._count[5m]))` |
| "The pool is the bottleneck" | Hikari saturation | `hikaricp_connections_pending` sustained above zero |
| "The executor is saturated" | queue depth + rejections | `executor_queued_tasks`, `orderflow_executor_rejected_total` |
| "We are not falling behind on events" | outbox depth + consumer lag | `orderflow_outbox_pending`, `kafka_consumer_fetch_manager_records_lag_max` |
| "The relay is not stuck" | `LongTaskTimer` | `orderflow_outbox_relay_batch_seconds_max` (in-flight duration) |
| "Metrics cost us little" | scrape timing + series count | `curl -w '%{time_total}'` on `/actuator/prometheus`; `grep -vc '^#'` |
| "Our cardinality is bounded" | series count under load | series count flat while request count climbs |
| "Prometheus can hold us" | head series + RSS | `prometheus_tsdb_head_series`, `process_resident_memory_bytes` |

### The metrics-about-metrics you should always have

These four cost almost nothing and are the difference between noticing cardinality growth
in a code review and noticing it in an outage:

```promql
# 1. Series per target — the leading indicator, per service.
sum by (job) (scrape_samples_scraped)

# 2. Growth rate — the alert. A sustained positive slope is a bug, not traffic.
deriv(scrape_samples_scraped{job="orderflow"}[30m]) > 0

# 3. Scrape duration against timeout — the cliff.
scrape_duration_seconds{job="orderflow"}

# 4. Prometheus's own head — the shared blast radius.
prometheus_tsdb_head_series
```

**Alert on number 2.** Series count *has a slope of zero* in a healthy service. Traffic
volume does not create series; distinct tag values do. A positive slope over half an hour
is a cardinality bug in flight, and catching it there is the difference between a code
revert and an incident.

### The cost model, measured rather than guessed

Fill this in for `orderflow` from your own readings. Every cell is a measurement or an
arithmetic result of measurements.

**Fill in — metrics cost (blank template):**

| Quantity | How you get it | Your value |
|---|---|---|
| Series exposed per pod | `grep -vc '^#'` on the scrape | |
| Pods at the Topic 65 baseline | your deployment | |
| Total series for `orderflow` | series × pods | |
| Scrape payload per pod | `%{size_download}` | |
| Scrape interval | Prometheus config | |
| Scrape bytes/second for this service | payload × pods ÷ interval | |
| Scrape CPU cost per pod | app CPU with scraping on vs off, at fixed load | |
| Prometheus RSS attributable to this service | RSS delta when the job is removed | |
| Retention window | Prometheus config | |

The last two are the ones a platform team will ask you for when you request a cardinality
increase, and "I don't know" is the answer that gets the request denied.

### Measure the observer, not just the observed

Two experiments worth running once, so you have a real answer instead of a hunch:

1. **Scrape cost.** Run the Topic 65 load with Prometheus scraping normally. Then run it
   again with the scrape job disabled. Compare application CPU and allocation rate
   (Topic 68's instruments). The delta is what observability costs you.
2. **Instrumentation cost.** Time the same hot path with and without your custom timers,
   using JMH (Topic 77) — *not* a `System.nanoTime()` loop, for exactly the reasons Topic
   77 gives. Expect the answer to be "negligible next to a database round trip", but
   *know* it rather than assume it. The one place it is not negligible is a meter resolved
   by name inside a tight loop, which is why Machine-level reality made you hoist it.

---

## Practice exercises

### 1 — Easy: inventory what you already export, and read it

**Goal:** replace "we have metrics" with a list.

1. Dump every meter name: `curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | sort`.
2. Classify each into one of: JVM/runtime, framework/infrastructure, `orderflow` business.
3. For each of the three RED signals and each USE resource in Example 2, write down which
   meter supplies it — or write **MISSING**.
4. Count series per meter family with the `uniq -c` pipeline from Proof 4, and list the
   top five.
5. Time a scrape and record it.

**Deliverable:** a one-page table. **Acceptance:** every RED and USE row either names a
meter or says MISSING, and the top-five list has a one-line justification for why each is
allowed to be that large.

### 2 — Medium: complete RED and USE on `orderflow` (combines Topics 55, 90, 109, 113, 115)

**Goal:** every request path has RED; every finite resource has USE; nothing is
unbounded.

1. Turn on `percentiles-histogram` for `http.server.requests` with
   `serviceLevelObjectives` at the boundaries your Topic 65 baseline suggests.
2. Add the business counters from Example 2 step 3, with a bounded `reason` tag.
3. Instrument every executor from Topic 90 with `ExecutorServiceMetrics.monitor`,
   **including a rejection counter** — the one Micrometer does not give you.
4. Build the Hikari USE panel set (utilisation, `pending`, timeouts, `usage_seconds`).
5. Add outbox depth (Topic 115) as a scheduled-sample gauge with a strong reference, and a
   `LongTaskTimer` on the relay batch.
6. Add Kafka consumer lag (Topic 113) as the consumer's saturation signal.
7. Write the cardinality budget table from Example 2 step 7 and fill in every cell by
   measurement.
8. Add the CI test from Trap 1 and the `MeterFilter` guards.

**Acceptance:**
- [ ] The fleet p99 query uses `histogram_quantile` over summed buckets. No `avg()` of a
      quantile anywhere in the dashboard JSON.
- [ ] Series count is flat across a full Topic 65 load run (record before and after).
- [ ] Every executor reports queue depth **and** rejections.
- [ ] The budget table has no blank cells.
- [ ] `promtool check metrics` passes on the scrape output.

### 3 — Hard: production simulation — detect three incidents from metrics alone

**Goal:** prove your instrumentation is sufficient by using nothing else.

Have someone else (or a script you write and then forget the contents of) inject **one**
of the following into a running `orderflow` under Topic 65 load, without telling you
which:

- **A.** Topic 55's defect: a 2-second HTTP call inside a `@Transactional` method on the
  order-placement path.
- **B.** Topic 113's defect: a consumer made slower than `max.poll.interval.ms`, causing a
  rebalance loop.
- **C.** Topic 115's defect: the outbox relay stopped (scale its scheduler to zero, or make
  its query error out silently).
- **D.** Topic 90's defect: the notification executor's queue bound removed and the
  downstream slowed, so the queue grows without bound.

**Rules:** you may look at dashboards and PromQL. You may **not** read application logs,
attach a debugger, take a thread dump, or look at the injected diff.

**Deliverable, per injected incident:**

| Question | Your answer |
|---|---|
| Which metric moved first? | |
| How long between injection and the first metric that moved? | |
| Which metric identified the *resource*, not just the symptom? | |
| Would an existing alert have fired? Which one, and after how long? | |
| Which metric was missing and would have made this a 30-second diagnosis? | |

**Acceptance:** you identify the injected fault from metrics alone, for at least three of
the four. For any you cannot, the deliverable is the *new meter or alert* that would have
made it possible — implemented, not described. That gap list is exactly what Topic 124's
"observability coverage gaps" section wants, so keep it.

---

## Interview questions

### Q1 — "We track average response time. Is that enough?"

**MID-LEVEL ANSWER**

"Not really, we should also track p95 and p99 so we can see the slow requests. Averages
hide outliers."

**SENIOR ANSWER**

"No, and the reason is structural rather than a preference. Every failure mode that
actually causes incidents is a tail event — a GC pause, a safepoint stall, connection-pool
saturation, a rebalance. Those hit a small fraction of requests very hard, and a mean is
dominated by the bulk, which is healthy by definition during a tail failure. So the mean
is systematically blind to exactly the thing I need to see.

Mechanically, `sum/count` also cannot answer the only question an SLO can be written
against: what *fraction* of requests were under the target. That requires a distribution.

So I publish a bucket histogram — in Micrometer, `publishPercentileHistogram` plus explicit
`serviceLevelObjectives` at the SLO boundary, so there is a bucket edge exactly where the
SLO is, which makes the fraction query exact rather than interpolated. Then percentiles are
computed in Prometheus with `histogram_quantile` over summed buckets.

I do keep the average, for one job: capacity modelling. Little's Law wants a mean service
time. Averages are for capacity, percentiles are for experience, and I would not let either
do the other's work."

**WHAT SEPARATES THEM**

The mid-level answer knows averages are bad. The senior answer knows *why* they are bad in
a way that predicts which incidents they will miss, knows that an SLO needs a fraction and
not a percentile, names the exact Micrometer configuration, and — the tell — keeps the
average for the one job it is genuinely correct for instead of performing outrage at it.

**FOLLOW-UP:** *"Your histogram gives you an approximate p99. Isn't an exact client-side
percentile better?"* — More accurate per JVM and useless across a fleet, because you cannot
combine percentiles. I would rather have a number that is approximate to bucket width and
correct for the whole service than a number that is exact for one pod and wrong when
aggregated. And I would put a bucket edge at the SLO target so the number that actually
gates the release is not interpolated at all.

---

### Q2 — "You run six pods. How do you compute the fleet p99?"

**MID-LEVEL ANSWER**

"I'd average the p99 across the pods, or take the max if I want to be conservative."

**SENIOR ANSWER**

"Neither works, because a percentile is a rank statistic and rank statistics do not add.
Concretely: if one pod served a hundred thousand requests and another served a hundred, the
fleet p99 is essentially the busy pod's, and the average of the two is wildly wrong — and
the correct answer changes with traffic split, which is information the two percentile
values do not contain. So the same two inputs have different right answers, which is the
definition of not aggregatable. `max()` at least fails safe, but it reports the worst pod
rather than the fleet.

The correct approach is to export bucket counters and aggregate those. Bucket counts *are*
counters, so summing them across pods is arithmetically valid, and I compute the quantile
from the merged histogram:

`histogram_quantile(0.99, sum by (le, uri) (rate(http_server_requests_seconds_bucket[5m])))`

Order matters: sum first, quantile last. Reversed, it is the same error in PromQL clothing.

In Micrometer that means `publishPercentileHistogram`, not `publishPercentiles` — the
latter emits `quantile=` labelled series which are correct per JVM and an invitation to
average them.

The cost is interpolation error within the bucket containing the quantile, which I bound by
placing an explicit SLO boundary at the number the SLO is written against."

**WHAT SEPARATES THEM**

The senior answer explains *why* aggregation fails with a concrete counterexample, names the
right query with the sum inside, names the Micrometer setting that produces the right export
shape, and states the residual error honestly instead of pretending buckets are exact.

**FOLLOW-UP:** *"When would you ever want client-side percentiles?"* — Single-instance
contexts, or a deliberately per-pod debug panel that is labelled as such. Also when the
registry backend is a system that does its own distribution merging. Never on a fleet
dashboard where someone will wrap it in `avg()` at 3am.

---

### Q3 — "Prometheus fell over an hour after our deploy. Walk me through it."

**MID-LEVEL ANSWER**

"Probably too many metrics. I'd check what changed in the deploy and roll back."

**SENIOR ANSWER**

"Rolling back is right, and I would do it while investigating rather than after. But the
diagnosis has a specific shape.

The signature of a cardinality incident is: the application is healthy — RED metrics
normal, error rate normal, users unaffected — while the monitoring system is not. That
asymmetry is the tell, and it distinguishes this from an application incident immediately.

I would look at three things in order. First `scrape_samples_scraped` per job, to find
which target's series count jumped — that names the service in seconds. Second
`scrape_duration_seconds` and `up` for that target, because a growing payload makes scrapes
slow, then time out, then the target goes down and *all* its metrics get gaps. Third
`prometheus_tsdb_head_series` and Prometheus's RSS, which is the mechanism: Prometheus holds
every active series from every target in memory.

Then I find the offending meter by counting series per family in the scrape output —
strip everything from the first brace and `sort | uniq -c | sort -rn`. The top line is
almost always a meter with a tag whose values come from data: an ID, a SKU, an email, a
raw URL path, or an exception message.

Prevention has three layers, because one is not enough. A `MeterFilter` that denies known
identifier tag keys and caps total meters, so the mistake cannot ship. A CI test that hits
an endpoint with N distinct IDs and asserts the meter count barely moves. And an alert on
the *slope* of `scrape_samples_scraped` — series count in a healthy service is flat, so any
sustained positive slope is a bug in flight.

One nuance worth saying: this also leaks in the JVM. The meter registry retains every meter
it ever created, so the application's heap grows too. Prometheus just dies first, because it
carries everyone's series and we carry only our own."

**WHAT SEPARATES THEM**

The senior answer identifies the asymmetric signature, gives an ordered diagnostic path
with named metrics, produces the exact command that finds the offending meter, gives layered
prevention rather than "be careful", and knows the application is also affected rather than
repeating the slogan.

**FOLLOW-UP:** *"You add the filter and redeploy. Does the problem go away?"* — Not
immediately, and this is the part people miss. A filter only affects registration, so the
JVM must restart to drop the existing meters, and Prometheus keeps the existing series until
they age past the staleness window and are compacted out of the head block. So the fix is:
deploy the filter, restart the pods, and then wait — or, if Prometheus is still unhealthy,
drop that job's data explicitly. Expect a recovery window, and say so in the incident channel
rather than declaring victory at deploy time.

---

### Q4 — "How do you know a thread pool is the bottleneck?"

**MID-LEVEL ANSWER**

"I'd check CPU usage and the number of active threads, and increase the pool size if
they're high."

**SENIOR ANSWER**

"Active threads is utilisation and it saturates uselessly: once the pool is full, `active`
equals `max` whether one task is queued or ten thousand. The number that carries information
past that point is **saturation** — queue depth. That is `executor_queued_tasks` for a
`ThreadPoolTaskExecutor` and `hikaricp_connections_pending` for the connection pool, and it
is the metric that predicts latency, because queueing time is where the latency goes.

I also want the E in USE, which Micrometer does not give you for free: rejections. A bounded
queue with `AbortPolicy` throws `RejectedExecutionException` and nothing counts it, so I
wrap the policy in a counter. Rejections are the most actionable pool signal there is —
they mean load shedding is happening and the system is at its designed limit rather than
silently growing an unbounded queue, which is Topic 90's failure.

On whether to increase the size: usually no, and this is where people go wrong. A bigger
Hikari pool typically makes latency worse, because Postgres runs a process per connection
and the contention just moves into the database. What sustained queue depth usually means is
that tasks are holding the resource too long — Topic 55's shape, an HTTP call inside a
transaction. `hikaricp_connections_usage_seconds` distinguishes those two hypotheses
directly: high usage time means the holder is slow, not that the pool is small.

One caveat for the modern stack: on virtual threads there is no pool and no queue, so this
whole USE set is empty. Saturation moves to whatever explicit bound you put in front of the
downstream — usually a semaphore — and if you did not put one there, you have no saturation
signal at all, which is its own finding."

**WHAT SEPARATES THEM**

Utilisation versus saturation as distinct concepts; naming the exact meters; knowing
rejections are missing by default and wiring them; refusing the reflex of enlarging the pool
and naming the metric that discriminates between the two causes; and knowing the virtual-thread
case breaks the model.

**FOLLOW-UP:** *"Queue depth is zero but latency is bad. Now what?"* — Then the pool is not
the constraint and I stop looking at it. Zero queue with bad latency means the work itself is
slow: a slow downstream (`Timer` on the client call), a slow query (Hibernate statistics,
Topic 50), GC pauses (Topic 71's logs), or safepoint stalls (Topic 73). RED tells me *where*;
USE tells me *which resource*; when USE says "no resource is saturated", the answer is that
the service time itself grew, and that is a different investigation.

---

### Q5 — "Your outbox relay silently stopped. Which metric tells you?"

**MID-LEVEL ANSWER**

"We'd see the timer for the relay stop reporting, or we'd notice events weren't arriving
downstream."

**SENIOR ANSWER**

"'The timer stopped reporting' is the trap in this question. A `Timer` records on
*completion*, so a relay that is hung contributes zero observations, and zero observations
look exactly like 'no work to do'. Absence is not a signal you can alert on reliably,
because absence is also the normal quiet-period state.

Three metrics answer this properly, and they answer different failure modes.

First, outbox depth as a gauge — unpublished rows in the table. That is the direct business
signal: it rises whether the relay is hung, crashed, scaled to zero, or just too slow. I
alert on the depth *and* on its slope, because a rising slope catches 'too slow' before the
absolute number is alarming.

Second, a `LongTaskTimer` on the relay batch. It reports the duration of *in-flight*
invocations at scrape time, so a batch stuck for ten minutes is visible right now rather
than after it finishes — which for a hang is never.

Third, relay iteration count as a counter, so I can alert on `rate() == 0` during a period
when depth is non-zero. That specific conjunction — work waiting and no iterations happening
— is unambiguous.

The gauge has two implementation details worth stating. It must hold a strong reference or
Micrometer's weak reference lets the sampled object be collected and the series turns into
NaN — which is the same failure as being blind, but sneakier. And it must not run
`SELECT count(*)` on the scrape thread; I sample it on a schedule into an `AtomicLong` and
gauge that, because a slow gauge makes the scrape slow and a slow scrape takes every metric
for that target down with it."

**WHAT SEPARATES THEM**

Recognising that absence of data is a bad alert; naming `LongTaskTimer` and knowing exactly
what problem it solves; alerting on the conjunction rather than a single signal; and knowing
both gauge implementation traps — the weak reference and the scrape-thread query — which are
the two things that make this gauge fail in production after working in a test.

**FOLLOW-UP:** *"How would you alert on it without paging someone every quiet night?"* —
Alert on the conjunction and on duration, not on an instantaneous value: depth above
threshold **for** a sustained window **and** iteration rate zero. And derive the threshold
from the Topic 65 baseline's normal depth distribution rather than picking a round number.
Topic 130 turns that into an error-budget-based alert, which is the version that survives
contact with an on-call rotation.

---

## Mental model checkpoint

Answer from memory. Write your answers down before checking them against the document —
retrieval, not recognition, is what makes this stick.

1. **A time series is identified by what, exactly?** And what follows from that about a tag
   whose value is a customer's email address?

2. **Why can bucket counters be summed across pods when percentiles cannot?** Give the
   one-sentence reason, and then give a concrete two-pod counterexample that shows averaging
   percentiles produces a number with no meaning.

3. **You have `_count` and `_sum` but no `_bucket` series.** Name every question you can
   still answer and every question you now cannot.

4. **A `Timer` and a `LongTaskTimer` on the same operation report different things.** What
   does each one show while the operation is *currently running*, and which failure mode
   does that difference matter for?

5. **Your outbox-depth gauge worked in the test and reports NaN in production after an
   hour.** What is the mechanism, and what are the *two* independent fixes this gauge needs?

6. **Hikari reports `active == max`.** What can you conclude, what can you not conclude, and
   which single additional metric resolves the ambiguity?

7. **You deploy a `MeterFilter` that denies the offending tag.** Why is Prometheus still
   unhealthy ten minutes later, and what are the two waits involved?

---

## Quick reference card

### The four rules

1. Tag values come from **code** (enums, statuses, methods), never from **data** (IDs, SKUs,
   emails, paths, exception messages).
2. Duration is a **histogram**, never an average, never a client-side percentile in a fleet.
3. **Aggregate buckets first, compute the quantile last.**
4. Every finite resource gets **saturation**, not just utilisation — queue depth, pending
   count, lag.

### Meter selection

| Need | Meter |
|---|---|
| A thing that happened | `Counter` |
| A current level | `Gauge` (strong reference; cheap lambda) |
| Duration of completed short operations | `Timer` with `publishPercentileHistogram()` |
| Size/value distribution | `DistributionSummary` |
| Duration of an operation that might hang | `LongTaskTimer` |

### Micrometer configuration

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.metrics.distribution.percentiles-histogram.http.server.requests=true
management.metrics.distribution.slo.http.server.requests=100ms,300ms,500ms,1s,3s
management.metrics.distribution.minimum-expected-value.http.server.requests=5ms
management.metrics.distribution.maximum-expected-value.http.server.requests=10s
# NOT this in a multi-pod deployment:
# management.metrics.distribution.percentiles.http.server.requests=0.5,0.95,0.99
```

### `MeterFilter` recipes

```java
MeterFilter.maximumAllowableTags(prefix, tagKey, n, MeterFilter.deny());
MeterFilter.maximumAllowableMetrics(3000);
MeterFilter.replaceTagValues("provider", v -> KNOWN.contains(v) ? v : "other");
MeterFilter.ignoreTags("instance");
MeterFilter.denyUnless(id -> id.getTags().stream().noneMatch(t -> FORBIDDEN.contains(t.getKey())));
MeterFilter.deny(id -> id.getName().startsWith("jvm.gc.pause") && lowValue(id));
```

### PromQL — the queries you will type most

```promql
# Rate
sum by (uri) (rate(http_server_requests_seconds_count{application="orderflow"}[5m]))

# Error fraction
  sum by (uri) (rate(http_server_requests_seconds_count{outcome="SERVER_ERROR"}[5m]))
/ sum by (uri) (rate(http_server_requests_seconds_count[5m]))

# Fleet p99
histogram_quantile(0.99, sum by (le, uri) (rate(http_server_requests_seconds_bucket[5m])))

# SLO attainment
  sum(rate(http_server_requests_seconds_bucket{uri="/api/orders",le="0.3"}[5m]))
/ sum(rate(http_server_requests_seconds_count{uri="/api/orders"}[5m]))

# Cardinality watch
deriv(scrape_samples_scraped{job="orderflow"}[30m])
```

### Diagnostic commands

```bash
curl -s localhost:8080/actuator/metrics | jq -r '.names[]' | sort
curl -s localhost:8080/actuator/metrics/http.server.requests | jq '.availableTags'
curl -s localhost:8080/actuator/prometheus | grep -vc '^#'                       # series count
curl -s localhost:8080/actuator/prometheus | grep -v '^#' | sed 's/[{ ].*//' \
  | sort | uniq -c | sort -rn | head                                             # top families
curl -s -o /dev/null -w '%{time_total} %{size_download}\n' localhost:8080/actuator/prometheus
curl -s localhost:8080/actuator/prometheus | promtool check metrics
```

### Reading the evidence

| Symptom | First metric to look at |
|---|---|
| Users report slowness, dashboard flat | Are you graphing an average or a quantile? |
| p99 disagrees with the load generator | `avg()` of a `quantile=` series somewhere |
| All metrics for one service have gaps | `up` and `scrape_duration_seconds` for that job |
| Prometheus memory climbing | `prometheus_tsdb_head_series`, then per-job `scrape_samples_scraped` |
| A gauge line just stops | Weak-reference gauge; the sampled object was collected |
| Latency bad, no queue anywhere | Not a resource problem — service time grew |

### Gotchas checklist

- [ ] Dotted meter names become snake_case with a unit suffix — grep for the exported name.
- [ ] `Counter` gets `_total` appended in Prometheus.
- [ ] `Gauge` holds a weak reference — keep a strong one.
- [ ] Gauge lambdas run on the scrape thread — keep them cheap.
- [ ] `ExecutorServiceMetrics.monitor` returns a wrapper — use the returned instance.
- [ ] Rejection counting is not built in — wire it yourself.
- [ ] `MeterFilter` applies at registration — a restart is needed to drop existing meters.
- [ ] Client-side percentiles cannot be aggregated.
- [ ] `Timer` records nothing while an operation is still running.
- [ ] Virtual-thread paths have no pool metrics — instrument the semaphore instead.

---

## When would I use this at work?

**1. The first week on any Java service you inherit.**
Before changing anything, dump the meter list, count series per family, and map RED and USE
onto what exists. You will find the same three gaps almost every time: no bucket histograms
(so no real percentiles), no rejection counters on bounded pools, and no business counters
(so a 100%-rejection outage looks perfectly healthy). That inventory is a credible
first-week deliverable and it makes every later investigation faster.

**2. When someone proposes adding a tag in code review.**
"What is the domain of that value, and how many distinct values will it have in
production?" is a fifteen-second question that prevents an outage. If the answer is "it
comes from the request", the answer is no — and the right response is not just no but
"put it on the trace and the log line instead", which gives the person what they actually
wanted. Pair it with the `MeterFilter` so the rule does not depend on you being in the
review.

**3. When you need to prove a change worked.**
Every optimisation from Phases 8–11 — a pool resize, a collector change, a cache, an
outbox, a batching fix — is a claim about a number over a window. Bucket histograms plus the
Topic 65 load give you a before-and-after you can put in a pull request description and
defend. This is also exactly the evidence Topic 124's gate review demands: a claim with a
named measurement that would falsify it.

---

## Connected topics

**Backwards:**

- **65 — the load-testing gate.** Your baseline p50/p95/p99 is the client-side ground truth
  that your server-side histogram must agree with. Proof 6 is the reconciliation.
- **68 / 82 — heap, allocation and container limits.** The scrape path allocates in
  proportion to series count, and the JVM metrics you export come from the same subsystem
  Topic 82 tunes. A cardinality problem shows up in Topic 68's allocation profile.
- **77 — JMH.** The only honest way to answer "does instrumentation cost anything on this
  path" is a benchmark, not a `nanoTime` loop.
- **79 — `ThreadLocal` and unbounded-map leaks.** The meter registry is an unbounded map
  keyed by tag set. High cardinality is Topic 79's leak wearing a monitoring costume.
- **90 — executors and pool sizing.** Supplies the pools that need USE, and the bounded
  queue whose rejections you must count yourself.
- **95 — atomics and `LongAdder`.** Explains why a counter increment is cheap under many
  concurrent writers.
- **101 — virtual threads.** Breaks the pool-metrics model: no pool, no queue, so saturation
  must be instrumented at an explicit bound.
- **109 — HikariCP.** `pending` and `usage_seconds` are the USE metrics that turn the
  pool-vs-thread-pool deadlock into a two-panel diagnosis.
- **111 — Resilience4j.** Breaker state and rejection counts are metrics; without them an
  open breaker is invisible.
- **113 / 115 — Kafka and the outbox.** Lag is consumer saturation; outbox depth plus a
  `LongTaskTimer` is how a stopped relay becomes visible.
- **120 — MDC and structured logging.** The exact mirror image: high-cardinality identity
  belongs in logs and traces, never in metric tags. One document per signal type, one rule
  each.
- **121 — Actuator and probes.** Same endpoint infrastructure, same management-port
  decisions, same "the framework assembled this, not you" hazard.

**Forwards:**

- **119 — tracing.** The third signal. When a metric says "p99 is bad", a trace says *which
  span*. The order-ID tag you were forbidden here is a first-class trace attribute there.
- **124 — the production-readiness gate.** Your cardinality budget, your RED/USE coverage
  table, and your Exercise 3 gap list are direct inputs to the review document.
- **129 — capacity and cost.** Little's Law needs the mean and the rate from these meters;
  the cost model needs your measured series count and scrape volume.
- **130 — SLOs and error budgets.** An SLO is a bucket-ratio query. The `serviceLevelObjectives`
  boundary you set here is what makes that query exact.
- **133 — postmortems.** "We had no metric for that" is the most common contributing factor
  in a Java postmortem, and Exercise 3's gap list is the pre-emptive version of it.
