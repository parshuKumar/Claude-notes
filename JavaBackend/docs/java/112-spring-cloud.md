# 112 — Spring Cloud: Discovery, Config, Gateway

## Phase: 11 — Distributed Systems & Production
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` is one service among several. This topic decides which Spring Cloud components it actually adopts and which it declines because Kubernetes already does the job. The concrete outputs are: a gateway in front of `orderflow` doing the one thing an ingress cannot express, externalised configuration with a documented startup-failure behaviour, and a written decision NOT to run a service registry.

---

## ELI5 anchor

Imagine an office building full of teams.

- **Service discovery** is the **reception desk**. You want the payments team; you
  ask reception which floor they are on. The alternative is a **sign on every
  door** that is always correct because the building management updates it when
  teams move. If the building already keeps the signs correct, a second reception
  desk with its own list is not extra help — it is a **second list that can
  disagree with the first**, and someone has to keep them in sync.
- **Config server** is the **staff handbook kept in one place**, so you do not
  print a copy for every desk and then discover forty versions. The trade: on your
  first day, if the handbook cupboard is locked, do you (a) refuse to start work,
  or (b) start work by guessing? Both are defensible. Not deciding is not.
- **A gateway** is the **front desk that does something before letting you
  through** — checks your appointment, stamps your form, decides which floor you
  actually need. A plain **ingress** is a door with a floor number on it: it can
  route, it cannot stamp. You need the front desk only when there is a stamp to
  apply.

The whole topic is: **your building already has some of these.** Do not install a
second one out of habit.

---

## The bridge from what you know

You know service discovery, centralised config and API gateways as system-design
concepts. You have made these decisions. So this section is not about what they
are — it is about the specific Java-ecosystem situation, which is unusual and worth
naming honestly.

### The honest framing: Spring Cloud is a suite from a different era

Spring Cloud was built around 2015, largely from Netflix's OSS stack (Eureka,
Ribbon, Hystrix, Zuul), for teams deploying JVM services onto **plain VMs** with no
orchestrator. In that world, nothing else knew where your instances were, nothing
else load-balanced between them, and nothing else restarted a dead one. Spring
Cloud filled a genuine void.

That void is now filled by Kubernetes for most teams. Here is the overlap, stated
flatly:

| Concern | Spring Cloud component | What Kubernetes already does | Verdict on k8s |
|---|---|---|---|
| Where are the instances of `payments`? | Eureka / Consul discovery | A `Service` gives a stable DNS name; `Endpoints`/`EndpointSlice` are updated by the control plane as pods become ready | **Decline.** Eureka is a second source of truth. |
| Load-balance across instances | Spring Cloud LoadBalancer | kube-proxy / a service mesh balances at L4; a mesh or ingress at L7 | **Usually decline.** Client-side LB is worth it only for specific L7 policies the platform cannot express. |
| Restart a failed instance | (nothing — Spring Cloud never did this) | Deployments, probes, restarts | n/a |
| Externalised configuration | Spring Cloud Config Server | `ConfigMap` and `Secret`, mounted or projected as env | **Depends.** See below — this is the genuinely contested one. |
| Push a config change without a restart | Spring Cloud Bus + `@RefreshScope` | Mounted `ConfigMap` files update in place; nothing reloads them for you | **Sometimes adopt.** This is a real gap. |
| Route external traffic | Spring Cloud Gateway | Ingress / Gateway API routes by host and path | **Adopt only for request-level logic the ingress cannot express.** |
| Circuit breaking | Spring Cloud Circuit Breaker (an abstraction over Resilience4j) | A service mesh can do outlier detection | **Use Resilience4j directly** — Topic 111. The abstraction buys you portability between breaker implementations, which is a problem you do not have. |

The recommendation the master plan states, and which this document argues for:
**adopt per-component, not as a suite.** "We use Spring Cloud" should never be a
sentence anyone says. "We use Spring Cloud Gateway because we need per-tenant rate
limiting at the edge, and we do not use Eureka because we are on Kubernetes" is the
sentence.

### What transfers directly from your Node experience

Almost all of it, conceptually. What differs:

**Client-side load balancing is a Java-culture default and is not a Node one.** In
Node you almost certainly called `http://payments-service/...` and let DNS, a
sidecar, or a load balancer sort it out. Spring Cloud's model is that *your process*
holds the instance list and picks one. That is a real architectural difference with
real consequences: your app now has a dependency on the registry, its own view of
health, and its own staleness window.

**`@LoadBalanced` is the marker for "this hostname is logical, not real".** It is a
qualifier annotation on a `RestClient.Builder` or `RestTemplate` bean that installs
an interceptor which rewrites `http://payments/...` into
`http://10.1.2.3:8080/...`. Without it, `payments` is passed to DNS and you get an
`UnknownHostException`. There is no Node equivalent because Node never had the
pattern.

**Boot 4 changes the client story.** Spring Boot 4 auto-configures **HTTP Service
Clients** — annotated Java interfaces as HTTP clients, the successor to Feign and
to hand-written `RestTemplate` wrappers. If you are starting fresh on Boot 4,
`spring-cloud-openfeign` is legacy. This matters because half the Spring Cloud
tutorials you will find are Feign tutorials.

### A version-honesty note before any code

**Spring Cloud versions are managed by a release-train BOM**, not by individual
artifact versions, and the trains are named by calendar year
(`2023.0.x`, `2024.0.x`, `2025.0.x`, and so on). Each train pins a compatible Spring
Boot range.

> **I am not going to state which release train is compatible with Boot 4.1.**
> Compatibility is published on the Spring Cloud project page and in the release
> notes, and an invented coordinate here would cost you an afternoon. Look it up,
> import the BOM, and let it manage every `spring-cloud-*` version.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>   <!-- from the compatibility table -->
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Two artifact ids I am explicitly flagging as things to verify rather than copy:

| Thing | Why | How to settle it |
|---|---|---|
| The **gateway** starter artifact id | Spring Cloud Gateway split into a WebFlux-based server and a Spring MVC-based server, and the artifact ids were reorganised during the 2024/2025 trains (names in the `spring-cloud-starter-gateway-server-*` family). I do not want you to copy a name that no longer resolves. | The Spring Cloud Gateway reference docs' "Getting Started" page for your train |
| Whether **Spring Cloud Config** ships a client that reads `ConfigMap`s directly | `spring-cloud-kubernetes` provides `ConfigMap`/`Secret` property sources; its module names have moved | The `spring-cloud-kubernetes` reference docs |

Everything else in this document is API and configuration I am confident about.

---

## What is this?

Spring Cloud is a family of independent projects sharing a release train. Three
matter for this topic.

### 1. Discovery — a registry plus a client

**The model.** Each service instance, at startup, **registers** itself with a
registry: "I am `orderflow`, I am at 10.1.2.3:8080, here is my metadata." It then
sends a **heartbeat** on an interval. Clients **fetch** the registry (and cache it
locally), then pick an instance themselves.

The three timing parameters that define its behaviour, and which are the whole story
of why it is a poor fit under an orchestrator:

| Parameter | Typical default | What it controls |
|---|---|---|
| Heartbeat interval | ~30s | How often an instance says "still here" |
| Lease expiry | ~90s | How long the registry keeps a silent instance |
| Client cache refresh | ~30s | How stale a client's copy of the registry can be |

Add them up: **an instance that dies can remain in a client's list for well over a
minute.** Eureka's designers knew this and it is deliberate — Eureka favours
availability over consistency, and it has **self-preservation mode**, which stops
evicting instances when it sees a large drop in heartbeats, on the theory that a
network partition is more likely than a mass death. That theory is correct on VMs
and exactly wrong during a Kubernetes rolling deploy, which *is* a mass death by
design.

**Spring Cloud LoadBalancer** is the client half: given a logical service name, pick
an instance. Round-robin by default, pluggable, with optional health checks,
zone-preference and caching.

### 2. Config Server — configuration as an HTTP resource

A Spring Boot application that serves configuration over HTTP, backed by a Git
repository (usually), Vault, or a filesystem.

```
GET /{application}/{profile}[/{label}]
GET /orderflow/production
GET /orderflow/production/main
```

The client contacts it **very early in startup** — during environment preparation,
before the `ApplicationContext` is built — so that the returned properties can
participate in normal property precedence (Topic 43).

**`@RefreshScope`** and the `/actuator/refresh` endpoint let a running application
re-fetch and rebuild selected beans without a restart. **Spring Cloud Bus** turns
that into a broadcast: publish a refresh event to a Kafka or RabbitMQ topic and
every instance refreshes.

The honest comparison with `ConfigMap`s:

| | Config Server | ConfigMap / Secret |
|---|---|---|
| Source of truth | Git — so you get history, review, blame, and a revert that is a `git revert` | The cluster; history depends on your GitOps tooling |
| Encryption | Built-in `{cipher}` values, or Vault | Secrets are base64, not encrypted, unless you add sealed-secrets/external-secrets |
| Shared config across services | First-class (`application.yml` in the repo applies to all) | Requires duplication or a tool |
| Availability at startup | **A network dependency.** See Trap 2. | Local file or env var — no network |
| Refresh without restart | `@RefreshScope` + Bus | Mounted files update; **nothing reloads them** |
| Operational cost | A service you run, monitor, patch and scale | Zero — the platform already runs it |

The genuine argument for Config Server on Kubernetes is **shared configuration
across many services with Git as the audited source of truth**, and **refresh
without restart**. If you have five services and restart to change config, use
`ConfigMap`s and skip it.

### 3. Gateway — routing plus request-level logic

Spring Cloud Gateway is a reverse proxy you configure with **routes**, each made of
**predicates** (does this request match?) and **filters** (what do we do to it?).

```
Route = id + uri + [predicates] + [filters]
```

Predicates: path, method, host, header, query parameter, cookie, remote address,
time before/after/between, weight.
Filters: add/remove/rewrite headers and paths, set status, rate-limit, circuit-break,
retry, cache, and anything you write.

**What an ingress already does:** route by host and path to a Service, terminate
TLS, and — depending on the controller — some rewriting, basic rate limiting and
header manipulation.

**What a gateway can do that a plain ingress usually cannot:**

- Compose or fan out — one inbound request to several backends (a mobile
  backend-for-frontend).
- Apply logic that requires understanding your domain — rate-limit per **tenant id
  extracted from a JWT claim**, not per IP.
- Route on the **body** or on a decoded token claim.
- Aggregate/translate protocols, or serve a legacy URL shape while the backend
  moves.
- Enforce a request-signing scheme or a partner API contract.

If your list of needs is "route `/api/*` to `orderflow` and terminate TLS", the
ingress does that and a gateway is an extra hop, an extra deployment, and an extra
thing that can be down.

---

## Why does it matter?

**1. Because the default is to adopt all of it, and that default is expensive.**
Spring Cloud is presented as a suite in most tutorials, and a team that follows the
tutorial ends up running Eureka, Config Server, a gateway and Spring Cloud Bus for a
system of four services on Kubernetes. Each is a deployment to operate, patch,
monitor, and be paged for. The failure modes multiply: now a Eureka outage, a Config
Server outage and a gateway outage are all `orderflow` outages.

**2. Because "two sources of truth" is a real, specific, observable bug.** It is
not an architectural nicety. On Kubernetes, `Endpoints` and Eureka disagree during
every deploy, and the disagreement window is tens of seconds long. Requests go to
pods that are already terminated. Topic 123's graceful shutdown work is undermined
by a registry that has not noticed.

**3. Because a gateway is a shared-fate component.** Every request to every service
goes through it. A bad route, a slow custom filter, or a memory leak in it takes
down everything at once. That is acceptable when it earns its place and unacceptable
when it exists because a tutorial had one.

**4. Because config changes cause outages.** More outages come from configuration
changes than from code deploys in most organisations, and configuration changes are
usually less reviewed, less tested, and applied faster. Whatever mechanism you pick,
the questions "how is this reviewed", "how is it rolled back", and "what happens if
the source is unreachable at startup" have to have answers.

---

## Syntax breakdown

### Discovery client and `@LoadBalanced`

```java
@Bean
@LoadBalanced                          // <-- the marker. Without it, nothing works.
RestClient.Builder loadBalancedRestClientBuilder() {
    return RestClient.builder();
}

@Service
class PaymentClient {

    private final RestClient client;

    PaymentClient(@LoadBalanced RestClient.Builder builder) {
        // "payments" is a LOGICAL name resolved by the load balancer,
        // not a hostname resolved by DNS.
        this.client = builder.baseUrl("http://payments").build();
    }

    PaymentResult charge(ChargeCommand cmd) {
        return client.post().uri("/v1/charges").body(cmd)
                     .retrieve().body(PaymentResult.class);
    }
}
```

What `@LoadBalanced` actually does: it is a `@Qualifier`. Spring Cloud
auto-configuration finds every builder bean carrying it and adds a request
interceptor. On each request the interceptor asks a `ReactiveLoadBalancer` for an
instance of the logical name in the URI's host position, and rewrites the URI.

> **If you inject a plain `RestClient.Builder` by mistake, `http://payments` goes
> straight to DNS.** The symptom is `UnknownHostException: payments`, which is
> unambiguous once you have seen it once. This is Trap 5.

Reading the registry directly, which is occasionally useful and always useful for
debugging:

```java
@RestController
class DiscoveryDebugController {

    private final DiscoveryClient discovery;

    @GetMapping("/debug/instances/{service}")
    List<Map<String, Object>> instances(@PathVariable String service) {
        return discovery.getInstances(service).stream()
            .map(i -> Map.<String, Object>of(
                 "instanceId", i.getInstanceId(),
                 "uri", i.getUri().toString(),
                 "metadata", i.getMetadata()))
            .toList();
    }

    @GetMapping("/debug/services")
    List<String> services() { return discovery.getServices(); }
}
```

That endpoint is how you prove Trap 1 — you compare its output to
`kubectl get endpointslices`.

### Config client

On the client, the connection details go in `application.yml` (they must be
available before any remote config is fetched):

```yaml
spring:
  application:
    name: orderflow            # becomes {application} in the Config Server URL
  config:
    import: "optional:configserver:http://config-server:8888"
  cloud:
    config:
      fail-fast: true          # DECIDE THIS DELIBERATELY -- see Trap 2
      retry:
        initial-interval: 1000
        max-attempts: 6
        multiplier: 1.5
        max-interval: 5000
```

Three things to read carefully.

**`spring.config.import` is the Boot 2.4+ mechanism** and replaces the old
`bootstrap.yml` / `spring-cloud-starter-bootstrap` arrangement. If a tutorial tells
you to create `bootstrap.yml`, it predates 2020.

**`optional:` is a loaded prefix.** With it, a Config Server that cannot be reached
is **not an error** — the application starts with whatever configuration it has
locally. Without it, an unreachable Config Server fails startup. These are opposite
behaviours selected by six characters, and this is Trap 2.

**`fail-fast: true` plus retry** is the combination that usually makes sense: retry
a few times to survive a restart-ordering race, then fail loudly rather than start
misconfigured. Note that `fail-fast` requires `spring-retry` on the classpath to do
the retrying.

Refreshable configuration:

```java
@Component
@RefreshScope                     // rebuilt on /actuator/refresh
class PricingPolicy {

    private final BigDecimal expressShippingSurcharge;

    PricingPolicy(@Value("${orderflow.pricing.express-surcharge}") BigDecimal surcharge) {
        this.expressShippingSurcharge = surcharge;
    }
}
```

```bash
curl -X POST localhost:8080/actuator/refresh
```

**WHAT TO LOOK FOR:** the response is a JSON array of the property **keys that
changed**. An empty array means nothing changed — either the source is unchanged or
the client did not re-fetch. That array is the only confirmation you get, and it is
the first thing to check when a refresh "does not work".

`@ConfigurationProperties` beans are refreshed by rebinding without needing
`@RefreshScope`, which is one more reason to prefer them over scattered `@Value`
(Topic 43).

### Gateway routes

Declaratively, in YAML:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orderflow-api
          uri: http://orderflow:8080          # or lb://orderflow with discovery
          predicates:
            - Path=/api/**
            - Method=GET,POST,PUT,DELETE
          filters:
            - RemoveRequestHeader=X-Internal-Trust
            - AddRequestHeader=X-Gateway, edge-1

        - id: legacy-orders
          uri: http://orderflow:8080
          predicates:
            - Path=/v1/orders/**
          filters:
            - RewritePath=/v1/orders/(?<segment>.*), /api/orders/${segment}
```

Or programmatically, which is better when routes have any conditional structure
because it is type-checked and testable:

```java
@Bean
RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("orderflow-api", r -> r
            .path("/api/**")
            .filters(f -> f.removeRequestHeader("X-Internal-Trust"))
            .uri("http://orderflow:8080"))
        .route("legacy-orders", r -> r
            .path("/v1/orders/**")
            .filters(f -> f.rewritePath("/v1/orders/(?<s>.*)", "/api/orders/${s}"))
            .uri("http://orderflow:8080"))
        .build();
}
```

**`RemoveRequestHeader=X-Internal-Trust` is not decoration.** If any internal
service trusts a header, the gateway must strip it from inbound external requests or
a client can forge it. Stripping trusted headers at the edge is one of the few
things a gateway is unambiguously the right place for.

A custom filter — the case where a gateway earns its keep:

```java
@Component
class TenantRateLimitFilter implements GlobalFilter, Ordered {

    private final RateLimiterRegistry limiters;   // Resilience4j, Topic 111

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String tenant = tenantFromJwt(exchange.getRequest());   // a CLAIM, not an IP
        if (!limiters.rateLimiter(tenant).acquirePermission()) {
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override public int getOrder() { return -1; }
}
```

**Note the return type.** The WebFlux-based gateway is **reactive** (Topics
103–108): filters return `Mono<Void>` and **blocking inside one blocks an event
loop thread**, degrading every concurrent request through the gateway, not just
yours. A JDBC call in a gateway filter is a production incident. This single fact
disqualifies a lot of "let's just look something up in the gateway" ideas, and it is
the main reason the Spring MVC variant of the gateway exists.

### Rate limiting at the gateway

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 100     # steady-state requests/second
      redis-rate-limiter.burstCapacity: 200     # token bucket depth
      redis-rate-limiter.requestedTokens: 1
      key-resolver: "#{@tenantKeyResolver}"     # a SpEL bean reference
```

```java
@Bean
KeyResolver tenantKeyResolver() {
    return exchange -> Mono.justOrEmpty(exchange.getRequest().getHeaders().getFirst("X-Tenant-Id"))
                           .defaultIfEmpty("anonymous");
}
```

The Redis-backed limiter is a token bucket **shared across gateway instances**,
which is the thing a per-instance Resilience4j `RateLimiter` cannot be. That
distinction — shared vs per-JVM state — is the same one from Topic 111's circuit
breakers, and it is why edge rate limiting belongs at the edge with a shared store.

`defaultIfEmpty("anonymous")` matters: if the key resolver returns empty, the
default behaviour is to **deny** the request. A missing header then becomes a 403
for every anonymous caller, which is a surprising way to break your public
catalogue.

### `[BOOT 3.x DELTA]`

| Concern | Boot 3.x era | Boot 4.1 | Note |
|---|---|---|---|
| Release train | A `2023.0.x` / `2024.0.x` train | A newer train — **check the compatibility table** | Never pin `spring-cloud-*` artifacts individually |
| Config client bootstrap | `spring.config.import` (since 2.4); legacy `bootstrap.yml` needs an explicit starter | Same, and the legacy path is further discouraged | `bootstrap.yml` in a snippet is a date stamp |
| HTTP clients | `RestTemplate`, `WebClient`, OpenFeign | **HTTP Service Clients auto-configured**; `RestClient` for synchronous | Feign is legacy for new code |
| Gateway artifacts | `spring-cloud-starter-gateway` | Reorganised into server-webflux / server-mvc starters | **Verify the id** |
| Circuit breaking | `spring-cloud-starter-circuitbreaker-resilience4j` | Same, but prefer Resilience4j directly | Topic 111 |
| Jackson | Jackson 2 | **Jackson 3 standard** | Custom gateway filters that touch JSON may need updating |

---

## Example 1 — minimal

The smallest system that lets you *see* the mechanics: two services, a registry, a
config server, and a gateway, all on one machine. Build it once so that the
arguments in the rest of this document are about something you have watched work.

### The config server

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: file://${user.home}/orderflow-config     # a local git repo for the lab
          default-label: main
```

The repo contains:

```
orderflow-config/
  application.yml          <- applies to EVERY service
  orderflow.yml            <- applies to the orderflow service, all profiles
  orderflow-production.yml <- orderflow, production profile only
```

Read it directly, because this is the whole API:

```bash
curl -s localhost:8888/orderflow/default | jq '.propertySources[].name'
curl -s localhost:8888/orderflow/production | jq '.propertySources[] | {name, source}'
```

**WHAT TO LOOK FOR:** `propertySources` is an **ordered array, most specific
first**. That order is the precedence order the client will apply.

| What you see | What it means |
|---|---|
| `orderflow-production.yml` before `orderflow.yml` before `application.yml` | Correct precedence: profile-specific beats application-specific beats global. |
| Only `application.yml` | Your `{application}` name does not match a file. Check `spring.application.name` on the client. |
| An empty `propertySources` array | The server reached the git repo and found nothing. Check `default-label` — a repo whose default branch is `master` while you asked for `main` returns empty, not an error. |
| HTTP 500 | The server could not reach or read the repo. This is what the client will see too. |

### The registry (for the lab only — see Example 2 for why not in production)

```java
@SpringBootApplication
@EnableEurekaServer
public class RegistryApplication { ... }
```

```yaml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false     # the registry does not register with itself
    fetch-registry: false
```

### `orderflow` as a client

```yaml
spring:
  application:
    name: orderflow
  config:
    import: "configserver:http://localhost:8888"     # NOT optional: -- fail loudly
  cloud:
    config:
      fail-fast: true
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10       # heartbeat  (default ~30)
    lease-expiration-duration-in-seconds: 30    # eviction   (default ~90)
```

The two lease settings are lowered from the defaults so that the lab actually shows
you eviction inside your attention span. Note what you just did: you traded registry
load for staleness. That is the only knob there is, and it is Trap 1's whole
argument in one place.

### The gateway

```yaml
server:
  port: 8080
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false          # do NOT auto-create a route per registered service
      routes:
        - id: orderflow
          uri: lb://orderflow     # lb:// = resolve via the load balancer
          predicates:
            - Path=/api/**
```

`discovery.locator.enabled: true` automatically creates a route for every service in
the registry. It is a demo feature. In production it means **registering a service
publishes it to the internet**, which is not a decision a service's own deployment
should be able to make unilaterally. Leave it off.

### Prove the wiring

```bash
# 1. Is orderflow registered?
curl -s -H 'Accept: application/json' localhost:8761/eureka/apps \
  | jq -r '.applications.application[].instance[] | "\(.app) \(.status) \(.ipAddr):\(.port."$")"'

# 2. Did orderflow load remote config?
curl -s localhost:8081/actuator/env | jq -r '.propertySources[].name' | head

# 3. Does the gateway route?
curl -i localhost:8080/api/products/SKU-1001

# 4. Kill orderflow with SIGKILL and watch how long it stays registered.
kill -9 $(pgrep -f orderflow) ; date
watch -n 2 "curl -s -H 'Accept: application/json' localhost:8761/eureka/apps \
  | jq -r '.applications.application[].instance[] | \"\(.app) \(.status)\"'"
```

**WHAT TO LOOK FOR** on step 4 — this is the observation the whole topic rests on:

| What you see | What it means |
|---|---|
| The dead instance still listed as `UP` for tens of seconds | **This is the staleness window.** Every request routed to it during that window fails. Time it and write the number down. |
| It eventually flips to `DOWN` or disappears | Lease expiry fired. The elapsed time should be roughly your `lease-expiration-duration-in-seconds`. |
| It **never** disappears | Self-preservation mode engaged — the registry decided too many heartbeats were missing and stopped evicting. In a lab with one instance, losing it is a 100% drop. |
| Requests through the gateway keep failing after eviction | The gateway's own client-side cache has not refreshed yet. Add its refresh interval to the staleness window. |

That last row is the important one: **the total staleness is the registry's eviction
delay plus the client's cache refresh interval**, and there are usually two caches
(the gateway's and each service's).

---

## Example 2 — production scenario (on the project spine)

### The situation

`orderflow` runs on Kubernetes: 8 pods, containerised, at the Topic 65 baseline
(70/20/10 mix, 100k products, 1M orders). It is one of five services —
`orderflow`, `catalog-search`, `notifications`, `reconciliation`, and a
`payments-adapter`. There is already an ingress controller terminating TLS and
routing by host. Topic 121's probes are wired and Topic 123's graceful shutdown
works.

A team member proposes "let's add Spring Cloud". The decision, component by
component, with reasons.

### Decision 1 — service discovery: DECLINE

**Do not run Eureka.** Use Kubernetes `Service` DNS.

```java
@Bean
RestClient paymentsAdapterClient(RestClient.Builder builder) {
    // A real DNS name backed by a Kubernetes Service. No @LoadBalanced.
    return builder.baseUrl("http://payments-adapter.orderflow.svc.cluster.local:8080")
                  .build();
}
```

The argument in four points, each of which is checkable:

1. **The platform already has the answer, and it is authoritative.** The kubelet
   marks a pod ready or not ready from the readiness probe; the endpoints
   controller adds and removes it from the `EndpointSlice` within a second or two.
   Eureka learns the same fact from a heartbeat, tens of seconds later, through a
   different path.
2. **Two sources of truth disagree during every deploy.** A rolling deploy
   terminates pods deliberately. Kubernetes removes them from the Service the moment
   readiness goes false — which is exactly what Topic 123's graceful shutdown makes
   happen *before* the process exits. Eureka keeps them for the lease duration.
   Requests routed by Eureka's list go to pods that are draining or gone.
3. **Eureka's self-preservation is actively wrong here.** It exists to survive a
   network partition by *not* evicting. A rolling deploy looks exactly like a
   partition to it. During a large deploy it can stop evicting at the precise moment
   eviction is most needed.
4. **It is another deployment to run.** Registry pods, their availability, their
   upgrades, their monitoring, and their own pages. For a capability the cluster
   already provides.

**When you would still adopt it:** you are not on an orchestrator; or you have a
genuine multi-cluster/multi-region topology the platform's DNS does not span and no
mesh; or you need instance metadata for routing (canary weights, shard ownership)
that Services cannot express. Those are real cases. None of them is "we are building
microservices".

Write the decision down (Topic 131's design-doc discipline):

> *We do not run a service registry. Service-to-service addressing uses Kubernetes
> Service DNS. Rationale: the platform's endpoint state is authoritative and is
> updated within seconds of a readiness change, whereas a registry adds a
> tens-of-seconds staleness window that conflicts with our graceful-shutdown
> behaviour. Revisit if we adopt a second cluster without a mesh.*

### Decision 2 — configuration: ADOPT PARTIALLY, with the failure mode chosen

The five services share a meaningful amount of configuration: the Kafka bootstrap
servers, the topic names, the JWT issuer URI, the standard timeout and retry
budgets, and the logging format. Duplicating that across five `ConfigMap`s is how it
drifts.

Adopt Config Server for **shared, non-secret, Git-audited** configuration. Keep
**secrets** in the platform's secret mechanism, not in Git — even encrypted.

```yaml
# orderflow/src/main/resources/application.yml  -- the only config that is baked in
spring:
  application:
    name: orderflow
  config:
    import: "configserver:${CONFIG_SERVER_URI}"     # NOT optional:
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.5
        max-interval: 5000
      label: ${CONFIG_LABEL:main}
```

**The startup-failure decision, made explicitly.** `fail-fast: true` means a Config
Server outage prevents `orderflow` from starting. That sounds bad until you consider
the alternative: starting with *partial* configuration and connecting to the wrong
database, or with the wrong Kafka cluster, or with an unset JWT issuer that makes
every request 401. A service that refuses to start is a clear, loud, obvious failure
that the deployment surfaces immediately. A service that starts misconfigured is an
incident with a confusing signature.

Two consequences to accept:

- **Already-running pods are unaffected** by a Config Server outage. Config is
  fetched at startup. So the blast radius is "we cannot deploy or scale up", not
  "we are down" — provided you are not also in a crash loop.
- **Therefore your restart behaviour matters.** If a pod is restarting for an
  unrelated reason while the Config Server is down, it cannot come back. Combine
  `fail-fast` with the retry above so a brief blip is survived, and make sure your
  Kubernetes `startupProbe` allows enough time for the retries (Topic 121).

**Refresh, and its honest limits.** `@RefreshScope` plus Spring Cloud Bus can push a
change to all 8 pods in seconds. Use it for things that are genuinely runtime
policy:

```java
@Component
@RefreshScope
class FraudThresholds {
    private final Money manualReviewAbove;
    FraudThresholds(@Value("${orderflow.fraud.manual-review-above}") Money threshold) {
        this.manualReviewAbove = threshold;
    }
}
```

Do **not** use it for anything that owns a resource — a `DataSource`, a Kafka
consumer, a thread pool. Rebuilding those beans mid-flight is where Trap 4 lives.

### Decision 3 — gateway: ADOPT, for exactly two reasons

The ingress already routes `/api/**` to `orderflow`. That alone does not justify a
gateway. Two requirements do:

**(a) Per-tenant rate limiting from a JWT claim.** Partner integrations have
contractual request quotas per tenant. The ingress can rate-limit by source IP,
which is the wrong key: one partner behind one NAT looks like one client, and a
partner using a cloud provider looks like thousands. The key must be the `tenant_id`
claim inside the JWT — the ingress cannot see it.

**(b) A legacy URL surface during a migration.** `/v1/orders/*` must keep working
for eighteen months while partners migrate to `/api/orders/*`. Putting that
rewriting in `orderflow` means the legacy shape is in the service's own routing
forever; putting it in the gateway makes it a deletable edge concern with a date on
it.

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        # Strip anything internal services trust. Non-negotiable.
        - RemoveRequestHeader=X-Internal-Trust
        - RemoveRequestHeader=X-Tenant-Id
        - RemoveRequestHeader=X-Actor-Id
      routes:
        - id: orderflow-api
          uri: http://orderflow.orderflow.svc.cluster.local:8080
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 200
                redis-rate-limiter.burstCapacity: 400
                key-resolver: "#{@tenantClaimKeyResolver}"

        - id: legacy-orders
          uri: http://orderflow.orderflow.svc.cluster.local:8080
          predicates:
            - Path=/v1/orders/**
          filters:
            - RewritePath=/v1/orders/(?<segment>.*), /api/orders/${segment}
            - AddResponseHeader=Deprecation, "true"
            - AddResponseHeader=Sunset, "Wed, 01 Jul 2026 00:00:00 GMT"
```

Note two things.

**`uri:` is a Kubernetes Service DNS name, not `lb://`.** The gateway does not need
client-side load balancing; the Service does it. This is the same decision as
Decision 1, applied consistently. Using `lb://` here would drag Eureka back in
through the side door.

**`Deprecation` and `Sunset` response headers.** If you are running a legacy surface
with an end date, say so in the protocol. Partners' monitoring can see it; a
migration email cannot.

**What the gateway must NOT do**, written as a rule for the team:

- No database access. The WebFlux gateway runs on event-loop threads; a blocking
  JDBC call there stalls every concurrent request through the gateway (Topics
  103–105).
- No business logic. If a rule needs to know what an order is, it belongs in
  `orderflow`.
- No authentication decisions beyond token *validation*. Authorization is
  `orderflow`'s (Topic 57), because only `orderflow` knows what a customer may do
  with a specific order. A gateway that authorizes is a gateway that must be
  redeployed whenever a permission changes.

### Decision 4 — client-side load balancing: DECLINE

Kubernetes Services balance at L4. That is sufficient for `orderflow`'s traffic
shape. Adopt Spring Cloud LoadBalancer only if you need an L7 policy the platform
does not offer — for instance sticky routing by a shard key, or a canary weighting
your ingress cannot express — and prefer a service mesh over putting that logic in
every application's process, because in-process load balancing means every service
must be redeployed to change the policy.

### Decision 5 — circuit breaking: use Resilience4j directly (Topic 111)

`spring-cloud-starter-circuitbreaker-resilience4j` gives you a portable
`CircuitBreakerFactory` abstraction over several implementations. That portability
solves a problem you do not have, and it hides the configuration surface — the
`slowCallRateThreshold` and `recordExceptions` settings from Topic 111 are exactly
the ones you need and exactly the ones an abstraction blurs. Use Resilience4j
directly.

### The resulting bill of materials

| Component | Adopted? | Why |
|---|---|---|
| Eureka / discovery | **No** | Kubernetes Services are authoritative and faster |
| Spring Cloud LoadBalancer | **No** | L4 balancing is sufficient |
| Spring Cloud Config Server | **Yes** | Shared config across 5 services, Git-audited; secrets stay in the platform |
| Spring Cloud Bus | **Yes**, narrowly | Push refresh for policy values only, never resource-owning beans |
| Spring Cloud Gateway | **Yes** | Per-tenant rate limiting from a JWT claim; legacy URL surface |
| Spring Cloud Circuit Breaker | **No** | Resilience4j directly (Topic 111) |
| Spring Cloud OpenFeign | **No** | Boot 4 HTTP Service Clients |
| Spring Cloud Stream | **No** | Spring for Apache Kafka directly (Topics 113–115) — you need partition and offset control the abstraction hides |

**Two of eight.** That is the shape of a good Spring Cloud adoption on Kubernetes,
and it is the deliverable of this topic.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — running a service registry on Kubernetes: two sources of truth

**Wrong approach**

Eureka is added because "microservices need service discovery", alongside a
Kubernetes cluster that already maintains `EndpointSlice`s for every Service.

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
  instance:
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
```

**Exact symptom**

During every rolling deploy, a fraction of inter-service calls fail with connection
refused or connection reset — to IP addresses that no longer host a pod. The error
rate spikes for roughly a minute per deploy and then recovers on its own, so it gets
written off as "deploy noise".

The observation that proves it — compare the two lists at the same moment:

```bash
# Source of truth A: the platform.
kubectl get endpointslices -l kubernetes.io/service-name=orderflow \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]} {.conditions.ready}{"\n"}{end}' | sort

# Source of truth B: the registry.
curl -s -H 'Accept: application/json' http://eureka:8761/eureka/apps/ORDERFLOW \
  | jq -r '.application.instance[] | "\(.ipAddr) \(.status)"' | sort
```

Run both during a `kubectl rollout restart deployment/orderflow`.

| What you see | What it means |
|---|---|
| The two lists are identical | Steady state. They agree when nothing is changing, which is why this is not caught in testing. |
| The registry lists IPs the EndpointSlice does not | **The disagreement window.** Any client using the registry is sending traffic to pods Kubernetes has already removed. Time it. |
| The registry's extra entries persist for tens of seconds | Lease expiry plus client cache refresh. This is the staleness you cannot configure away without hammering the registry. |
| The registry lists them as `UP` long after the pods are gone | Self-preservation mode. During a large rolling deploy the heartbeat drop looks like a partition and eviction stops entirely. |

Correlate with your error rate: the failure spike and the disagreement window are
the same interval. That correlation is the proof.

**Root cause**

Two independent systems are tracking the same fact — which pods can serve traffic —
by different mechanisms with different latencies.

- Kubernetes: **push**, driven by the readiness probe, propagated by the endpoints
  controller in a second or two.
- Eureka: **pull plus heartbeat**, with an eviction delay by design and a client
  cache on top.

They cannot be made to agree, because the disagreement is inherent in the eviction
delay. Shortening the leases reduces the window and increases registry load and
false evictions during a GC pause (Topic 71) or a network blip.

The aggravating factor is Topic 123: graceful shutdown deliberately makes readiness
false **before** the process exits, so Kubernetes stops sending traffic during the
drain. A registry-based client never learns that readiness changed and keeps sending
traffic to a draining pod for the whole lease duration — undoing the graceful
shutdown work entirely.

**Fix**

Remove the registry. Address services by their Kubernetes DNS name and delete the
`@LoadBalanced` builders.

```java
// Before
RestClient client = lbBuilder.baseUrl("http://payments-adapter").build();

// After
RestClient client = builder
    .baseUrl("http://payments-adapter.orderflow.svc.cluster.local:8080")
    .build();
```

Put the base URL in configuration, not in code, so environments differ by config
rather than by profile-specific beans.

**If you genuinely cannot remove it** — multi-cluster, no mesh, a migration in
progress — then at minimum: make it a **read-only mirror**, not the source of truth,
by running `spring-cloud-kubernetes` discovery backed by the API server rather than
Eureka's self-registration; and align the health signal so the registry's view of an
instance is driven by the same readiness state Kubernetes uses. That is much more
work than deleting it, which is itself an argument.

---

### Trap 2 — the Config Server as an unowned startup dependency

**Wrong approach**

```yaml
spring:
  config:
    import: "optional:configserver:http://config-server:8888"
```

Six characters — `optional:` — copied from a getting-started guide because without
them local development failed when the config server was not running.

**Exact symptom**

The config server is unreachable during a deploy. `orderflow` pods start
**successfully**. Readiness passes. They are put into the load balancer. And then:

- The JWT `issuer-uri` is unset, so every authenticated request returns 401 (Topic
  57).
- Or `spring.datasource.url` falls back to a Boot default and the app starts an
  embedded database, so writes go somewhere that is not Postgres and reads return
  nothing.
- Or the Kafka bootstrap servers are unset and the outbox relay silently fails to
  publish (Topic 115).

Which of these you get depends on which properties were remote. All of them look
like an application bug, not a configuration-delivery bug, and the on-call engineer
spends the first twenty minutes in the wrong place.

The diagnostic that settles it in one command:

```bash
curl -s localhost:8080/actuator/env | jq -r '.propertySources[].name'
```

| What you see | What it means |
|---|---|
| A source named like `configserver:...` or `configClient` near the top | Remote config loaded. Good. |
| **No** config-server source at all | The import was skipped. With `optional:` that is silent. This is the bug. |
| A config-server source present but nearly empty | The server answered but matched no files — check `spring.application.name`, the profile, and the `label`/branch. |

Also check on startup:

```bash
kubectl logs deploy/orderflow | grep -i 'config server\|configserver\|Fetching config'
```

**WHAT TO LOOK FOR:** a startup line naming the config server URI and the resolved
`{application}/{profile}/{label}`. Its absence, with `optional:`, is the only
evidence you get.

**Root cause**

`optional:` converts a hard dependency into a silent one. The application's
correctness depends on properties it did not receive, and nothing checks that.

The deeper cause is that **nobody decided**. `optional:` was added to fix a local
development annoyance and shipped to production, where it means something entirely
different.

**Fix**

Three parts, and you need all three.

**(a) Fail fast in production, with retry.**

```yaml
spring:
  config:
    import: "configserver:${CONFIG_SERVER_URI}"    # no optional:
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.5
        max-interval: 5000
```

`fail-fast` requires `spring-retry` on the classpath. Verify the retries actually
happen by watching the startup log with the config server stopped: you should see
repeated attempts, then a startup failure — not an immediate failure and not a
successful start.

**(b) Make local development work without weakening production.** Use a profile, not
`optional:`:

```yaml
# application-local.yml  -- used ONLY on a developer machine
spring:
  config:
    import: "optional:configserver:http://localhost:8888"
```

The production profile keeps the non-optional import. Now the two environments
differ in a way that is visible in a file name.

**(c) Assert the configuration you cannot start without.** Independent of the
import mechanism, validate at startup:

```java
@ConfigurationProperties("orderflow")
@Validated
public record OrderflowProperties(
        @NotBlank String jwtIssuerUri,
        @NotBlank String kafkaBootstrapServers,
        @NotNull @Positive Integer outboxRelayBatchSize) { }
```

A missing property now fails context startup with a message naming the property.
This is Topic 43's `@ConfigurationProperties` discipline doing security work: it
turns "started with the wrong config" into "did not start", which is the failure you
want.

**(d) Set the Kubernetes probe budget accordingly.** With retries, startup takes
longer in the failure case. `startupProbe.failureThreshold x periodSeconds` must
exceed the worst-case retry sequence, or Kubernetes kills the pod mid-retry and you
get a crash loop whose cause is invisible. Topic 121.

---

### Trap 3 — retries at the gateway AND in the client: multiplicative amplification

**Wrong approach**

The gateway gets a retry filter, because retries are good:

```yaml
filters:
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE,GATEWAY_TIMEOUT
      methods: GET,POST                 # note: POST
```

And `orderflow`'s own payment client already has Topic 111's retry:

```yaml
resilience4j.retry.instances.payments.maxAttempts: 3
```

And the ingress controller has its own default retry policy, which nobody looked at.

**Exact symptom**

The payment provider has a partial outage. Instead of the 3× multiplier you reasoned
about in Topic 111, the provider sees roughly **3 × 4 = 12×**, and if the ingress
also retries, more. The provider rate-limits you. Your error rate goes to 100% and
stays there long after their incident is over, because you are now generating enough
load to keep yourself throttled.

Measure it directly rather than reasoning about it:

```bash
# 1. Requests entering the gateway.
curl -s localhost:8080/actuator/metrics/spring.cloud.gateway.requests | jq .

# 2. Requests arriving at orderflow.
curl -s localhost:8081/actuator/metrics/http.server.requests | jq '.measurements'

# 3. Requests arriving at the payment stub (Topic 111's /control/arrivals).
curl -s localhost:9099/control/arrivals | jq 'add'
```

| What you see | What it means |
|---|---|
| (2) is a small multiple of (1) during an incident | The **gateway** is retrying. Every retry is a fresh request to `orderflow`. |
| (3) divided by the order-placement count is much larger than `maxAttempts` | **Layered retries.** The multiplier is the product, not the sum. |
| (3) climbs while (1) is flat | All the extra load is generated inside your own stack. |
| Provider returns 429 | You are throttling yourself. |

**Root cause**

Retries compose **multiplicatively** across layers, and each layer is configured by
a different person who can see only their own layer. Three retries at the ingress,
three at the gateway and three in the service is 4 × 4 × 4 = **64 attempts** for one
user request, in the worst case.

Compounding it: **the gateway retry filter includes POST in the example above.** A
gateway cannot know whether a POST is idempotent. Retrying `POST /api/orders` at the
edge creates duplicate orders — Topic 111's Trap 3, applied at a layer that has even
less information than the service does.

**Fix**

State a **retry budget** as an explicit, written architectural rule:

> **Retries happen at exactly one layer: the innermost caller that knows whether the
> operation is idempotent. Every other layer passes failures through.**

Concretely for `orderflow`:

```yaml
# GATEWAY: retry only safe methods, only on connection-level failures,
# and only once. The gateway does not know your semantics.
filters:
  - name: Retry
    args:
      retries: 1
      methods: GET                       # NEVER POST/PUT/PATCH/DELETE
      series: SERVER_ERROR
      exceptions:
        - java.io.IOException
      backoff:
        firstBackoff: 50ms
        maxBackoff: 200ms
        factor: 2
        basedOnPreviousValue: false
```

- **Ingress:** no application-level retries. Connection-establishment retries only.
- **Gateway:** at most one retry, `GET` only.
- **Service (Resilience4j):** the real retry policy, with jitter, a circuit breaker
  outside it, and a bulkhead — because this is the only layer that knows the call is
  idempotent and holds an idempotency key (Topic 116).

Then **verify the product**, do not assume the sum:

> `total_amplification = ingress_attempts x gateway_attempts x service_attempts`

Put that number on the capacity model and compare it to the provider's rate limit.
`1 × 2 × 3 = 6×` peak is a number you can defend. `64×` is not.

And add the alert: a counter of requests leaving your estate for the provider,
divided by the count of `POST /api/orders`. If that ratio exceeds your intended
multiplier, someone added a retry somewhere.

---

### Trap 4 — `@RefreshScope` does not refresh what you think it refreshes

**Wrong approach**

```java
@Configuration
@RefreshScope
class DataSourceConfig {
    @Bean
    DataSource dataSource(@Value("${spring.datasource.url}") String url,
                          @Value("${spring.datasource.hikari.maximum-pool-size}") int poolSize) {
        var config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setMaximumPoolSize(poolSize);
        return new HikariDataSource(config);
    }
}
```

"We can tune the pool size at runtime without a deploy." Also seen with Kafka
consumer factories, thread pools, and `RestClient` beans holding connection pools.

**Exact symptom**

You push a pool-size change and call `/actuator/refresh`. Then, over the following
minutes:

- Connection count at Postgres **grows** rather than changing — the old pool was
  never closed. `SELECT count(*) FROM pg_stat_activity WHERE application_name =
  'orderflow';` climbs past what any single pool should hold.
- In-flight requests holding a connection from the **old** pool complete against a
  `DataSource` bean nothing references any more, and their transactions behave
  unpredictably at commit.
- Heap grows and does not come back (Topic 79) — the old pool's threads and
  connections are still reachable.
- Eventually Postgres refuses connections and every service that shares that
  database is affected.

The observation:

```bash
curl -s -X POST localhost:8080/actuator/refresh | jq .
curl -s localhost:8080/actuator/metrics/hikaricp.connections | jq '.measurements'
psql -c "select application_name, count(*) from pg_stat_activity group by 1;"
```

| What you see | What it means |
|---|---|
| `/actuator/refresh` returns an array containing your property key | The property **did** change. The refresh mechanism works. |
| `/actuator/refresh` returns `[]` | Nothing changed — the client did not re-fetch, or the source is unchanged. A different problem. |
| Postgres connection count higher than `maximum-pool-size` and rising after each refresh | **The old pool leaked.** Each refresh creates a new pool and abandons the old one. |
| Hikari metrics look normal | They report the **current** bean. The leaked pool is invisible to them, which is why this is hard to see. |

**Root cause**

`@RefreshScope` creates a **proxy** whose target is discarded and rebuilt on the
next access after a refresh. It has no knowledge of what the bean owns.

For a value holder that is exactly right. For a resource owner it is wrong twice
over:

1. **The old instance is not closed** unless it is a bean whose destroy callback
   Spring invokes — and a discarded refresh-scope target does not reliably get the
   full singleton destruction treatment. Even when it does, closing a pool with
   in-flight transactions is not safe.
2. **The new instance is created lazily, on the next call**, potentially on a
   request thread, potentially in several threads at once.

There is a second, quieter version of this trap that catches everyone: **a plain
`@Value` injected into a plain singleton is never refreshed at all.** The field was
set at construction. `/actuator/refresh` updates the `Environment`, not your field.
The refresh reports the key as changed and your code keeps using the old value —
which is worse than nothing, because the report says it worked.

**Fix**

**(a) Never put `@RefreshScope` on anything that owns a resource.** Connection
pools, Kafka producers/consumers, executors, HTTP clients with connection pools,
caches, schedulers. These change by **restart**, and on Kubernetes a restart is a
rolling deploy, which is cheap and observable. "Change without restart" is not worth
a leaked connection pool.

**(b) Use `@ConfigurationProperties` for refreshable values.** They are rebound on
refresh without `@RefreshScope`, and reading the current value is a method call
rather than a captured field:

```java
@ConfigurationProperties("orderflow.fraud")
@Validated
public class FraudProperties {
    @NotNull private Money manualReviewAbove;
    // getters/setters -- rebound in place on refresh
}

@Service
class FraudService {
    private final FraudProperties props;      // the same object, rebound
    boolean needsReview(Money amount) {
        return amount.isGreaterThan(props.getManualReviewAbove());   // read per call
    }
}
```

**(c) Write down which properties are refreshable.** A short table in the README
listing every runtime-tunable property. Anything not in it requires a deploy. This
prevents the "we can change anything at runtime" belief that produces the
`DataSource` example.

**(d) Prove the refresh, every time.** The `/actuator/refresh` response array is
necessary but not sufficient. Expose the effective value and check it:

```java
@GetMapping("/actuator/effective-config")
Map<String, Object> effective() {
    return Map.of("manualReviewAbove", props.getManualReviewAbove().toString());
}
```

If the refresh array names the key but this endpoint shows the old value, you have
the `@Value`-in-a-singleton version of the bug.

---

### Trap 5 — the logical service name that goes to DNS

**Wrong approach**

```java
@Service
class PaymentClient {
    private final RestClient client;

    // A PLAIN builder, injected by type. No @LoadBalanced qualifier.
    PaymentClient(RestClient.Builder builder) {
        this.client = builder.baseUrl("http://payments").build();
    }
}
```

Or, equally common, the `@LoadBalanced` builder is defined but a second, plain
builder bean also exists and Spring injects that one.

**Exact symptom**

```
java.net.UnknownHostException: payments
```

on the first call, in production, at the moment a real user places an order — not at
startup, because the client is constructed lazily and nothing resolves the host until
a request is made.

The variant that is harder: it works in the integration test (where a
`@LoadBalanced` builder is the only one in the context) and fails in production
(where an auto-configured plain one also exists).

| What you see | What it means |
|---|---|
| `UnknownHostException: payments` | The logical name reached DNS. The load-balancer interceptor was not applied. |
| `NoSuchElementException` / "No instances available for payments" | The interceptor **was** applied and the registry has no instances. A different, better-diagnosed failure. |
| It works from one service and not another | Two builder beans exist; injection resolved differently by construction order or by `@Primary`. |

Confirm which builder you got:

```java
@Component
class ClientProbe {
    ClientProbe(RestClient.Builder builder) {
        System.out.println("builder class: " + builder.getClass().getName());
    }
}
```

**Root cause**

`@LoadBalanced` is a `@Qualifier`. Injection by type alone does not honour it, so
any plain builder bean satisfies the parameter. The URI's host is then treated as a
real hostname.

**Fix**

Qualify the injection point, always:

```java
PaymentClient(@LoadBalanced RestClient.Builder builder) { ... }
```

And add a startup assertion so the failure moves from "first user request in
production" to "context refresh in CI":

```java
@Component
class LoadBalancerWiringCheck {
    LoadBalancerWiringCheck(@LoadBalanced RestClient.Builder builder,
                            LoadBalancerClientFactory factory) {
        Assert.notNull(factory, "Load balancer factory missing");
    }
}
```

**Or — the fix this document actually recommends — do not use logical names at
all.** On Kubernetes, `http://payments-adapter.orderflow.svc.cluster.local:8080` is
a real hostname that real DNS resolves, `@LoadBalanced` is unnecessary, and this
entire trap cannot occur. Trap 5 is a class of bug that only exists because you
adopted Trap 1.

---

## Hands-on proof

### Proof 1 — measure the discovery staleness window

The number that decides Trap 1, measured in your own environment.

```bash
# Terminal 1: sample the platform's view every second, timestamped.
while true; do
  printf '%s PLATFORM ' "$(date +%s)"
  kubectl get endpointslices -l kubernetes.io/service-name=orderflow \
    -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]},{end}'
  echo
  sleep 1
done

# Terminal 2: sample the registry's view every second, timestamped.
while true; do
  printf '%s REGISTRY ' "$(date +%s)"
  curl -s -H 'Accept: application/json' http://eureka:8761/eureka/apps/ORDERFLOW \
    | jq -r '[.application.instance[] | select(.status=="UP") | .ipAddr] | join(",")'
  sleep 1
done

# Terminal 3: cause a change.
kubectl delete pod <one-orderflow-pod> ; date +%s
```

**WHAT TO LOOK FOR:** the timestamp at which each list stops containing the deleted
pod's IP. The difference is the staleness window.

| What you see | What it means |
|---|---|
| PLATFORM drops the IP within ~1–2 seconds | The endpoints controller working normally. |
| REGISTRY drops it tens of seconds later | The window. Any client using the registry sends traffic to a dead pod for that long. |
| REGISTRY never drops it | Self-preservation. Check the registry's own logs for the mode. |
| Both drop it at the same time | You are using `spring-cloud-kubernetes` discovery backed by the API server, not Eureka self-registration. That is a much better arrangement — confirm which you are running. |

Repeat during a `kubectl rollout restart deployment/orderflow` with all 8 pods
cycling. The window is worse under a rolling deploy, which is when you can least
afford it.

### Proof 2 — see exactly which property sources the app loaded

```bash
curl -s localhost:8080/actuator/env | jq -r '.propertySources[].name'
```

**WHAT TO LOOK FOR:** an **ordered** list; earlier entries win.

| What you see | What it means |
|---|---|
| A `configserver:` / `configClient` source present, above `applicationConfig:` | Remote config loaded and takes precedence. |
| No config-server source | It was skipped (`optional:`) or never configured. Trap 2. |
| `systemEnvironment` above the config-server source | An environment variable is overriding remote config. This is correct precedence (Topic 43) and is a frequent surprise. |

To find where one specific value came from:

```bash
curl -s 'localhost:8080/actuator/env/spring.datasource.url' | jq .
```

That returns the resolved value **and every source that offered a value**, in
precedence order. It is the single most useful configuration-debugging endpoint in
Spring Boot. Note that it exposes configuration and must never be publicly reachable
(Topic 57).

### Proof 3 — read the gateway's actual route table

```bash
curl -s localhost:8080/actuator/gateway/routes | jq -r '.[] | "\(.route_id)  \(.uri)  \(.predicate)"'
curl -s localhost:8080/actuator/gateway/globalfilters | jq .
curl -s localhost:8080/actuator/gateway/routefilters | jq .
```

```yaml
management:
  endpoint:
    gateway:
      access: read_only     # Boot 3.4+ replaced enabled:true with an access level
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,gateway
```

**WHAT TO LOOK FOR:** the routes, in the order the gateway will evaluate them, with
their compiled predicates.

| What you see | What it means |
|---|---|
| Your route is listed with the predicate you wrote | Configuration parsed as intended. |
| A route you did not define, one per registered service | `discovery.locator.enabled: true`. Turn it off — registering a service should not publish it. |
| The predicate string differs from your YAML | A parsing surprise, usually in a regex or an escaped comma. The compiled form is the truth. |
| Routes in an unexpected order | Add explicit `order:` values. Route evaluation order is not the YAML order in every case. |

Then prove a route end to end and see which one matched:

```bash
curl -i -H 'X-Tenant-Id: partner-a' localhost:8080/v1/orders/8213
```

**WHAT TO LOOK FOR:** the `Deprecation` and `Sunset` response headers from the
legacy route. Their presence proves the `legacy-orders` route matched rather than
`orderflow-api`.

### Proof 4 — prove a refresh actually took effect

```bash
# 1. Change the value in the git repo and commit.
# 2. Confirm the SERVER sees it.
curl -s localhost:8888/orderflow/production | jq '.propertySources[0].source'

# 3. Refresh ONE pod and read the returned key list.
curl -s -X POST localhost:8080/actuator/refresh | jq .

# 4. Read the EFFECTIVE value from the application, not from the environment.
curl -s localhost:8080/actuator/effective-config | jq .
```

| What you see | What it means |
|---|---|
| Step 2 shows the new value; step 3 lists the key; step 4 shows the new value | The refresh worked end to end. |
| Step 3 lists the key; step 4 shows the **old** value | The bean captured the value at construction. `@Value` in a plain singleton — Trap 4's quiet variant. |
| Step 3 returns `[]` | The client did not re-fetch, or the server is serving a cached/different label. Check `label` and the git branch. |
| Step 2 shows the old value | The config server has not picked up the commit. Check its clone refresh behaviour and the branch. |

Do this on **one** pod first. Refreshing all 8 simultaneously via Spring Cloud Bus is
how a bad value becomes a total outage in one second.

---

## Practice exercises

### Easy — read the three route tables

Stand up the Example 1 lab. Then, without changing anything, produce a one-page
answer to: for a request to `GET /api/products/SKU-1001` arriving at the gateway,
list every hop, and for each hop name the component that decided where it went next
and the endpoint that would let you verify that decision.

Then break exactly one thing — stop `orderflow` — and record which hop reports the
failure, what status code the caller sees, and how long it takes. Success criterion:
you can predict the failure signature of each hop before you break it.

### Medium — the adoption decision (combines Topics 43, 57, 111, 121, 123)

For `orderflow` as described in Example 2, write a two-page adoption decision
covering all eight components in the bill of materials. For each: adopt or decline,
one paragraph of reasoning, and — this is the part that makes it an exercise rather
than an essay — **the observation that would change your mind**.

Then implement the two you adopt, and demonstrate:

1. The config server with `fail-fast: true`; prove that stopping it prevents startup
   and that the retry sequence is visible in the logs, and that the Kubernetes
   `startupProbe` budget accommodates it (Topic 121).
2. The gateway with per-tenant rate limiting from a JWT claim, and a test showing
   two tenants have independent budgets while sharing an IP.
3. The `RemoveRequestHeader` default filter, with a test proving that a forged
   `X-Tenant-Id` from outside is stripped before reaching `orderflow` (Topic 57).
4. A `@ConfigurationProperties` bean whose value refreshes, and a `@Value` singleton
   whose value does not — side by side, with the `/actuator/refresh` output and the
   effective-value endpoint for each. Explain the difference in two sentences.

### Hard — production simulation on the spine

Run `orderflow` at the Topic 65 baseline behind the gateway and produce numbers.

1. **The gateway tax.** Run the 70/20/10 mix directly against `orderflow`, then
   through the gateway. Report the p50/p95/p99 delta and the extra CPU. State
   whether the two features the gateway provides are worth that number, in one
   sentence with the number in it.
2. **Layered retry amplification.** Configure retries at the gateway (3, including
   POST) and in the service (Topic 111's `maxAttempts: 3`). Stub the payment gateway
   down. Measure the requests arriving at the stub per user request. Then apply the
   retry-budget rule and re-measure. Report both multipliers.
3. **Duplicate orders from an edge retry.** With the gateway retrying POST, confirm
   whether one user request can produce two orders. Then add the `Idempotency-Key`
   handling from Topic 116 and show it cannot. This is the strongest argument
   against gateway-level POST retries and you should have the evidence.
4. **Discovery staleness, if you run the registry variant.** Execute Proof 1 during
   a rolling restart of all 8 pods and correlate the disagreement window with your
   5xx rate. Report the number of failed requests attributable to the window.
5. **Config-server outage during a deploy.** Take the config server down, then
   trigger a rolling deploy. Record: whether running pods are affected, whether new
   pods start, what the Kubernetes events say, and how long before the deployment is
   declared failed. Then repeat with `optional:` and record how much worse the
   diagnosis is.
6. **Write the runbook entry.** Ten lines: how to check which config a pod loaded,
   how to roll back a config change, how to see the gateway route table, and what
   the blast radius of a gateway outage is.

---

## Interview questions

### Q1 — "We use Eureka for service discovery. Thoughts?"

**MID-LEVEL:** "That's the standard Spring Cloud approach. It works well with
Ribbon or Spring Cloud LoadBalancer."

**SENIOR:** "It depends entirely on where you run. On plain VMs it's solving a real
problem. On Kubernetes it's a second source of truth for a fact the platform already
tracks, and the two will disagree.

The disagreement is specific and measurable. Kubernetes updates EndpointSlices from
the readiness probe within a second or two. Eureka works on heartbeats with a lease
expiry — tens of seconds — plus a client-side cache on top of that. So during every
rolling deploy there's a window where the registry lists pods Kubernetes has already
removed, and clients using the registry send traffic to them. I'd prove it by
sampling both lists during a `rollout restart` and correlating the disagreement
window with the 5xx spike.

It's worse than merely redundant, because Eureka's self-preservation mode stops
evicting when it sees a large drop in heartbeats — which is exactly what a rolling
deploy is. And it actively undoes graceful shutdown: we make readiness false before
the process exits so Kubernetes stops sending traffic, and a registry-based client
never learns that.

I'd address services by Service DNS and delete the registry. I'd keep it only for a
multi-cluster topology with no mesh, or if I needed instance metadata for routing
that Services can't express."

**What separates them:** the mid-level answer evaluates the technology in isolation.
The senior answer evaluates it against the platform it runs on, names the specific
mechanism of the disagreement, gives the measurement, and knows the graceful-shutdown
interaction — which is the second-order consequence most people miss.

**Follow-up:** *"What if we're mid-migration and can't remove it?"* — Make it a
read-only mirror rather than the source of truth: `spring-cloud-kubernetes`
discovery reads the API server, so there is one source of truth with a Spring
Cloud-shaped interface on top. And align the registry's health signal with the same
readiness state, so the two at least degrade together.

### Q2 — "Your config server is down. What happens?"

**MID-LEVEL:** "The app would fail to start, I think. Or use cached config."

**SENIOR:** "That's a decision I have to have made, and it's controlled by six
characters. `optional:configserver:...` means an unreachable config server is not an
error and the app starts with whatever it has locally. Without `optional:` plus
`fail-fast: true`, it refuses to start.

I choose to fail. Starting with partial configuration means connecting to the wrong
database, or an unset JWT issuer so every request 401s, or unset Kafka bootstrap
servers so the outbox relay silently stops publishing. Those look like application
bugs and waste the first twenty minutes of an incident. A pod that won't start is
loud and unambiguous, and the deployment surfaces it immediately.

The blast radius is worth being precise about: config is fetched at startup, so
already-running pods are unaffected. A config-server outage means 'we can't deploy or
scale', not 'we're down' — unless something is already crash-looping, in which case
it can't come back. So I pair `fail-fast` with a bounded retry to survive a blip, and
I make sure the Kubernetes startupProbe budget exceeds the retry sequence, or the
kubelet kills the pod mid-retry and the real cause is invisible.

Independently, I validate required properties with `@ConfigurationProperties` and
`@Validated`, so 'started with the wrong config' fails at context refresh regardless
of how the config arrived."

**What separates them:** the mid-level answer guesses. The senior answer knows it is
a configured choice, argues for one side, states the blast radius precisely
(deploys, not availability), and connects it to the probe budget — which is the part
that turns a good decision into a working one.

**Follow-up:** *"Why not `ConfigMap`s?"* — Usually yes. Config Server earns its
place when you have many services sharing configuration and want Git as the audited,
reviewable source of truth with `git revert` as the rollback, and when you need
refresh without restart. For five services that restart to change config, use
`ConfigMap`s and run one less thing.

### Q3 — "Where do retries belong: the gateway, the client, or both?"

**MID-LEVEL:** "Both is safer — defence in depth."

**SENIOR:** "One layer, and it's the innermost one that knows whether the operation
is idempotent. Retries compose multiplicatively, not additively: three at the
ingress, three at the gateway and three in the service is 4 × 4 × 4 = 64 attempts
for one user request in the worst case. Each layer is configured by a different
person who can only see their own layer, so nobody ever computes the product.

The gateway is the worst place for it, because it has the least information. It sees
a POST and cannot know whether it is idempotent — so a gateway retry on
`POST /api/orders` creates duplicate orders. I'd allow at most one gateway retry, on
GET only, for connection-level failures.

The real retry policy lives in the service with Resilience4j: jitter, a circuit
breaker outside the retry, a bulkhead, and an idempotency key on anything with a side
effect. That's the only layer that has all three pieces of information.

Then I'd put the multiplier on the dashboard — outbound requests to the provider
divided by `POST /api/orders` count — and alert if it exceeds what we designed for.
That catches the day someone adds a retry somewhere."

**What separates them:** "defence in depth" is exactly the wrong instinct here, and
the senior answer says why with arithmetic. It also identifies the idempotency
information asymmetry — the gateway cannot know — and proposes a measurable
invariant rather than a convention.

**Follow-up:** *"The ingress already retries by default. How would you find out?"* —
Read the ingress controller's configuration, then measure: compare request counts at
the ingress and at the service during an induced failure. Do not trust the
documentation of a component someone else configured.

### Q4 — "What does a gateway give you that an ingress doesn't?"

**MID-LEVEL:** "Routing, load balancing, authentication, rate limiting — it's the
entry point for microservices."

**SENIOR:** "The ingress already does most of that list, so I'd invert the question:
what do I need that the ingress *cannot express*? For us there were two things.

Per-tenant rate limiting keyed on a JWT claim. The ingress can rate-limit by source
IP, which is the wrong key — one partner behind a NAT looks like one client, and a
partner on a cloud provider looks like thousands. Getting the tenant means decoding
the token, and that is application-level.

And a legacy URL surface during an eighteen-month partner migration. Putting the
rewrite in the gateway makes it a deletable edge concern with a `Sunset` header on
it, rather than something that lives in the service's routing forever.

What I would not put there: business logic, database access, or authorization.
Database access in particular — the WebFlux gateway runs on event-loop threads, so a
blocking JDBC call stalls every concurrent request through the gateway, not just
that one. And authorization belongs in the service, because only the service knows
whether this customer may read this order; a gateway that authorizes has to be
redeployed whenever a permission changes.

The cost side is real: it's a shared-fate component. Every request to every service
goes through it, so a bad route or a slow custom filter is a total outage. It has to
earn that."

**What separates them:** the mid-level answer lists capabilities. The senior answer
starts from what the existing platform already provides, names the specific gap,
states what must not go there and why (the event-loop point especially), and prices
the shared-fate risk.

**Follow-up:** *"Your gateway is at 100% CPU and everything is slow. First three
things you check?"* — Custom filters for blocking calls; the Redis rate limiter's
latency, since every request touches it; and route count and predicate complexity,
since predicates are evaluated in order per request.

### Q5 — "We changed a config value and pushed it to all pods instantly. Good or bad?"

**MID-LEVEL:** "Good — that's the point of Spring Cloud Bus, no restart needed."

**SENIOR:** "Fast is only good if it is also reversible and staged. Pushing to all 8
pods in one second means a bad value is a total outage in one second, with no canary
and no gradual rollout — which is strictly worse than a rolling deploy, where the
platform stops the rollout when the new pods fail their probes.

So: refresh one pod first, verify the effective value from the application rather
than trusting the `/actuator/refresh` key list, then the rest.

I'd also be careful what is refreshable at all. `@RefreshScope` on anything that
owns a resource — a connection pool, a Kafka consumer, an executor — leaks the old
instance: the new bean is built lazily on next access and the old pool's connections
are never released. You watch `pg_stat_activity` climb past your configured pool
size after every refresh, and Hikari's own metrics do not show it because they
report the current bean.

And the quiet failure: a plain `@Value` in a singleton is never refreshed, but
`/actuator/refresh` still lists the key as changed. So the report says it worked and
the code uses the old value. `@ConfigurationProperties` is rebound properly and is
what I use.

I'd keep a written list of which properties are runtime-tunable. Everything else
changes by deploy."

**What separates them:** the mid-level answer treats speed as the goal. The senior
answer identifies that instant fleet-wide change removes the safety property a
rolling deploy provides, and knows both failure modes of `@RefreshScope` — the
resource leak and the silently-unrefreshed `@Value`.

**Follow-up:** *"How do you roll back a config change?"* — With Config Server on
Git, `git revert` plus a refresh, and the history tells you who changed what and
when. That auditability is the strongest argument for Config Server over
`ConfigMap`s, and it only holds if changes go through review rather than direct
pushes to the branch the server reads.

---

## Mental model checkpoint

1. On Kubernetes, name the two systems that both track "which pods can serve
   traffic", the latency of each, and the exact command that shows them disagreeing.
2. What do the six characters `optional:` change about a config-server outage? Which
   behaviour do you want in production, and why?
3. Three retry layers with 3 retries each. What is the worst-case number of requests
   reaching the dependency for one user request?
4. `/actuator/refresh` returns `["orderflow.fraud.threshold"]` but the behaviour did
   not change. Give two causes and the command that distinguishes them.
5. Why is `@RefreshScope` on a `DataSource` bean dangerous? Name the specific
   resource that leaks and the query that shows it.
6. Name two things a gateway can do that an ingress cannot, and two things a gateway
   must never do.
7. You see `UnknownHostException: payments`. What was missing, and what is the
   design change that makes this class of bug impossible?

---

## Quick reference card

**The adoption question, for every component**

> *What does the platform already do? What is left over? Is the left-over worth a
> deployment I must operate?*

**Component verdicts on Kubernetes**

| Component | Default verdict | Adopt when |
|---|---|---|
| Eureka / registry | **Decline** | No orchestrator; multi-cluster with no mesh; routing on instance metadata |
| Spring Cloud LoadBalancer | **Decline** | An L7 policy the platform cannot express — and prefer a mesh |
| Config Server | **Consider** | Many services sharing config; Git-audited changes; refresh without restart |
| Spring Cloud Bus | **Narrowly** | Policy values only. Never resource-owning beans. |
| Gateway | **Consider** | Request-level logic the ingress cannot express (claims, composition, legacy surfaces) |
| Circuit Breaker abstraction | **Decline** | Use Resilience4j directly (Topic 111) |
| OpenFeign | **Decline** | Boot 4 HTTP Service Clients |
| Spring Cloud Stream | **Usually decline** | You need partition/offset control it hides (Topics 113–115) |

**Key configuration**

```yaml
spring:
  config:
    import: "configserver:${CONFIG_SERVER_URI}"    # no optional: in production
  cloud:
    config:
      fail-fast: true
      retry: { max-attempts: 6, initial-interval: 1000, multiplier: 1.5 }
    gateway:
      discovery:
        locator:
          enabled: false       # never auto-publish services
      default-filters:
        - RemoveRequestHeader=X-Internal-Trust
```

**Diagnostics**

```bash
curl -s localhost:8080/actuator/env | jq -r '.propertySources[].name'
curl -s 'localhost:8080/actuator/env/spring.datasource.url' | jq .
curl -s localhost:8080/actuator/gateway/routes | jq -r '.[] | "\(.route_id) \(.uri)"'
curl -s -X POST localhost:8080/actuator/refresh | jq .
curl -s -H 'Accept: application/json' http://eureka:8761/eureka/apps | jq .
kubectl get endpointslices -l kubernetes.io/service-name=orderflow -o wide
```

**Gotchas checklist**

- [ ] No service registry duplicating Kubernetes Services.
- [ ] `optional:` is absent from the production config import.
- [ ] `fail-fast: true` with retry, and the `startupProbe` budget covers it.
- [ ] Required properties validated by `@ConfigurationProperties` + `@Validated`.
- [ ] Retries exist at exactly **one** layer; the multiplier is on a dashboard.
- [ ] The gateway never retries POST/PUT/PATCH/DELETE.
- [ ] `discovery.locator.enabled: false`.
- [ ] Trusted internal headers stripped at the edge by a default filter.
- [ ] No `@RefreshScope` on any resource-owning bean.
- [ ] The refreshable-property list is written down.
- [ ] Refresh is applied to one pod first, and verified from the application.
- [ ] Every `spring-cloud-*` version comes from the BOM.

---

## When would I use this at work?

**1. When someone proposes "let's add Spring Cloud".** This document is the reply.
Not "no", but "which components, and what does the platform already do for each".
The two-of-eight bill of materials is the artefact, and producing it takes an
afternoon and saves a year of operating things you did not need.

**2. When a rolling deploy has a reliable error spike nobody has explained.** The
staleness-window measurement in Proof 1 either finds it or rules it out in twenty
minutes, and it is the sort of "reliable noise" that teams normalise for years.

**3. When you need to change behaviour without a deploy — and must decide what
"behaviour" means.** Feature flags, fraud thresholds, rate limits: yes. Connection
pools, consumer configuration, thread pools: no, those change by restart. Drawing
that line explicitly, and writing down which properties are on which side, prevents
the leaked-connection-pool incident before it happens.

---

## Connected topics

**Backwards**

- **32 — dependency resolution and BOMs.** Spring Cloud versions come from a release
  train BOM. Pinning individual artifacts is how you get a `NoSuchMethodError`.
- **43 — configuration and property precedence.** Config Server participates in the
  same precedence chain; `/actuator/env` is the tool that shows it.
- **44 / 46 — controllers and error contracts.** A gateway changes what the client
  sees on failure; keep the `ProblemDetail` shape consistent across the hop.
- **56 / 57 — Spring Security.** The gateway validates tokens and strips trusted
  headers; `orderflow` authorizes. Never move authorization to the edge.
- **62 — contract testing.** A gateway rewriting paths is a contract change; test it.
- **65 — the baseline.** Re-measure through the gateway. The extra hop is a number.
- **103–108 — reactive.** The WebFlux gateway is an event loop. Blocking in a filter
  is a total gateway stall, not a local slowdown.
- **111 — Resilience4j.** Retries belong in the service, not the edge. Layered
  retries multiply.

**Forwards**

- **113–115 — Kafka.** Why Spring Cloud Stream is usually declined: you need explicit
  partition, offset and rebalance control.
- **116 — idempotency.** The only safe way to allow any retry of a POST anywhere in
  the stack.
- **118 — metrics.** `spring.cloud.gateway.requests` for edge RED metrics; watch the
  route-id tag cardinality.
- **119 — tracing.** The gateway must propagate `traceparent`, or every trace starts
  at the service and you lose the edge latency.
- **121 — Actuator and probes.** The `startupProbe` budget must cover the config
  retry sequence; the gateway needs its own probes.
- **123 — graceful shutdown.** The interaction that makes a registry actively
  harmful; also, the gateway must drain before the services behind it.
- **126 — build/buy/adopt.** This whole topic is a worked example of that decision
  framework.
- **131 — design docs.** The adoption decision is a document, and "the observation
  that would change my mind" is the part that makes it a good one.
