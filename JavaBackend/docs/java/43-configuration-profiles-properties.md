# 43 — Configuration, Profiles, `@ConfigurationProperties`, and Property Precedence

## Phase: 5 — Spring Boot & Persistence
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: `orderflow` configuration for the `local`, `docker` and `load` profiles — the `load` profile is what Topic 65's gate runs against

---

## ELI5 anchor

Imagine a stack of transparent sheets on an overhead projector.

The bottom sheet has your defaults written on it: `timeout = 5 seconds`.

On top of that you lay a sheet for this environment: `timeout = 30 seconds`.

On top of *that* someone in ops lays a sheet with `timeout = 2 seconds`.

You look down through the stack. You see **2 seconds**. The other two values are still
there, still written down, still real — they are just covered up.

Now the bug that costs people an afternoon: you are editing the **bottom** sheet, saving,
restarting, and nothing changes. You are staring at `timeout = 5 seconds` in a file and
the application keeps behaving as if it were 2. Nothing is broken. You are just looking
at a sheet that something else is covering.

Spring's job here is twofold:

1. **Define the stacking order** so "which sheet is on top" is a rule, not a surprise.
2. **Give you a way to look down through the stack** and see which sheet a value came
   from — that is `/actuator/env`.

The whole topic is: know the stacking order, and know how to look.

---

## The bridge from what you know

### `@nestjs/config` ≈ `@ConfigurationProperties`: **PARTIAL**

You have done this before. In NestJS:

```ts
// nest: config module + a typed accessor
ConfigModule.forRoot({
  envFilePath: ['.env.local', '.env'],
  load: [paymentConfig],
  validationSchema: Joi.object({
    PAYMENT_TIMEOUT_MS: Joi.number().default(5000),
  }),
});

// then, somewhere
const timeout = this.config.get<number>('payment.timeoutMs');
```

Spring:

```yaml
orderflow:
  payment:
    timeout: 5s
```

```java
@ConfigurationProperties(prefix = "orderflow.payment")
public record PaymentProperties(Duration timeout) {}
```

```java
public OrderService(PaymentProperties payment) {  // injected, typed, already validated
    this.timeout = payment.timeout();
}
```

### What transfers

| Concept you have | Spring equivalent | Verdict |
|---|---|---|
| `.env` files | `application.yml` / `application.properties` | **HONEST ANALOGUE** |
| `.env.local` overriding `.env` | `application-local.yml` overriding `application.yml` | **HONEST ANALOGUE** |
| `NODE_ENV` | `spring.profiles.active` | **PARTIAL** — Spring supports *multiple* simultaneous profiles |
| Joi/Zod validation of config at boot | `@Validated` on `@ConfigurationProperties` | **HONEST ANALOGUE** |
| `configService.get<T>()` | injecting a typed properties record | **PARTIAL** — Spring's is a real object with real types, not a lookup |
| A registered config factory (`load: [...]`) | `@ConfigurationPropertiesScan` | **PARTIAL** |

### What does not transfer — and this is the payload

**1. A defined, multi-level precedence chain.**
`@nestjs/config` has a short, simple rule: `process.env` wins over loaded files, and files
earlier in `envFilePath` win over later ones. That is roughly three levels and you can hold
it in your head.

Spring Boot has **about seventeen**, in a fixed documented order, spanning command-line
arguments, `SPRING_APPLICATION_JSON`, JVM system properties, OS environment variables,
profile-specific files inside *and* outside the jar, `@PropertySource`, and defaults set in
code. There is no Nest equivalent to that depth, and "why did my env var win?" is the daily
consequence.

**2. Relaxed binding.**
This one has genuinely no Nest analogue.

```
ORDERFLOW_PAYMENT_TIMEOUT      (an OS environment variable)
      binds to
orderflow.payment.timeout      (a property name)
```

Spring will also accept `orderflow.payment-timeout`, `orderflow.paymentTimeout` and
`orderflow.payment_timeout` as the same property when binding to
`@ConfigurationProperties`. In Node you would write that mapping by hand in a factory
function. In Spring it is a binder feature, and it is the mechanism that makes a
Kubernetes `env:` block work without any translation layer (Topic 123).

**3. Typed conversion with real types.**
`config.get<number>('x')` in Nest is a cast over a string; nothing checks it. Spring binds
`30s` into a `java.time.Duration`, `10MB` into a `DataSize`, `SEQUENTIAL` into your enum,
and a malformed value **fails the application startup** with the offending property named.

**4. Profiles are additive and plural.**
`NODE_ENV` is one string. `spring.profiles.active=docker,load,debug` activates three at
once, and `spring.profiles.group` lets one name expand into several. That is more powerful
and more confusing, and you need the tooling in the Hands-on section to keep it honest.

---

## What is this?

Spring's configuration system has four layers. Learn them as four separate things, because
mixing them up is where the confusion lives.

### Layer 1 — `Environment` and `PropertySource`

`Environment` is a bean in the context. It holds an **ordered list** of `PropertySource`
objects. A `PropertySource` is just a named key/value lookup — one wraps
`System.getenv()`, one wraps `System.getProperties()`, one wraps the parsed
`application.yml`, and so on.

Resolving a property means walking that ordered list and returning the **first** hit.
That is the entire precedence mechanism. There is no merging, no scoring, no cleverness:
first source in the list that has the key wins.

### Layer 2 — config *files*

`application.properties` / `application.yml`, plus profile-specific variants
`application-{profile}.yml`. Boot searches, by default, these locations (later ones win):

```
classpath:/
classpath:/config/
file:./
file:./config/
file:./config/*/
```

That "file: beats classpath:" ordering is why a file next to your jar overrides the one
packaged inside it — the deployment-time override without a rebuild.

### Layer 3 — profiles

A profile is a named string. Beans and config documents can declare that they only apply
when a profile is active.

```java
@Profile("load")
@Bean
DataSource loadTestDataSource() { ... }
```

```yaml
# in application.yml, a second document
---
spring:
  config:
    activate:
      on-profile: load
orderflow:
  payment:
    timeout: 2s
```

Multiple profiles can be active at once. Later-listed active profiles win over
earlier-listed ones when both supply the same key.

### Layer 4 — binding

Getting values out. Two mechanisms, and they are not equivalent:

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Granularity | one property per field | a whole tree into one object |
| Type safety | SpEL string, converted late | bound and converted at startup |
| Validation | none | `@Validated` + Jakarta constraints |
| Relaxed binding | limited (see below) | full |
| Failure timing | **runtime**, or silently a default | **startup**, with the property named |
| IDE support | none | metadata file gives autocomplete |
| Testability | needs a context or reflection | it is a plain object; `new` it in a test |

The mastery line for this topic is: **you bind typed config with validation rather than
scattering `@Value`.**

---

## Why does it matter?

**1. "Why did my env var win?" is a weekly question.**
Someone changes `application.yml`, redeploys, and nothing changes. The value is coming
from a Kubernetes `env:` entry set eight months ago by a person who has left. Without the
precedence chain and `/actuator/env` this is an hour of guessing. With them it is thirty
seconds.

**2. Config errors should fail at startup, not at 3am.**
`@Value("${orderflow.payment.timeout}")` into a `String` will accept `"thirty seconds"`
and blow up the first time a payment is attempted — possibly weeks later, possibly only in
the region where someone typo'd it. `@ConfigurationProperties` with a `Duration` field
refuses to start. A pod that will not start is caught by your rolling deploy. A pod that
starts and then fails on real traffic is an incident.

**3. Topic 65's gate depends on it.**
The `load` profile is not a nicety. It is how `orderflow` runs against the 1M-order
dataset with a larger connection pool, SQL logging off, and caches configured for a
realistic warm state. If the load profile is not a clean, reproducible configuration, the
baseline numbers are not reproducible either, and the gate rule (`±10%` on re-run) fails.

**4. `@ConditionalOnProperty` from Topic 42 is only as good as this.**
An auto-configuration backing off because a property is "missing" — when in fact it is set
in a source you did not check — is exactly the debugging loop this topic closes.

**5. Secrets and 12-factor deployment (Topic 123) ride on relaxed binding.**
`ORDERFLOW_PAYMENT_GATEWAY_APIKEY` from a Kubernetes secret binds to
`orderflow.payment.gateway.api-key` with no glue code. That property only works if you
understand the mapping rules.

---

## Syntax breakdown

### `@ConfigurationProperties` with constructor binding (the default you should use)

```java
package com.orderflow.payment;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;
import org.springframework.validation.annotation.Validated;

import java.net.URI;
import java.time.Duration;
import java.util.List;

@Validated
@ConfigurationProperties(prefix = "orderflow.payment")
public record PaymentProperties(

        @NotNull
        URI gatewayUrl,

        @NotBlank
        String apiKey,

        @DefaultValue("5s")
        Duration timeout,

        @Min(0) @Max(5)
        @DefaultValue("2")
        int maxRetries,

        @DefaultValue
        Retry retry,

        List<String> enabledMethods
) {
    public record Retry(
            @DefaultValue("200ms") Duration initialBackoff,
            @DefaultValue("2.0")   double multiplier
    ) {}
}
```

Line by line, the parts that are new to you:

| Piece of syntax | What it means |
|---|---|
| `@ConfigurationProperties(prefix = "orderflow.payment")` | bind everything under that YAML subtree into this object. The prefix must be kebab-case or lowercase dotted. |
| a `record` | records get **constructor binding automatically**. There is no setter, so the object is immutable after binding. This is the modern default. |
| `@DefaultValue("5s")` | the default used when the property is absent. Note it is a *string* — it goes through the same conversion as a real value. |
| `@DefaultValue` with no argument on `Retry retry` | "if nothing under `orderflow.payment.retry` is set, still construct a `Retry` using *its* defaults". Without this you get `null` and an NPE later. |
| `Duration timeout` | accepts `5s`, `PT5S`, `500ms`, `2m`. A bare `5000` is interpreted using the default unit (milliseconds) unless `@DurationUnit` says otherwise. |
| `@Validated` on the class | switches on Jakarta Bean Validation **at bind time**. Failure = the application does not start. |
| `jakarta.validation.constraints` | **not** `javax.validation`. That package move happened in Boot 3 and is Topic 127's headline. |

### Registering it — pick exactly one

```java
// Option A: scan a package for @ConfigurationProperties classes (recommended for an app)
@SpringBootApplication
@ConfigurationPropertiesScan
public class OrderflowApplication { }
```

```java
// Option B: name them explicitly (recommended inside a library / auto-configuration)
@AutoConfiguration
@EnableConfigurationProperties(PaymentProperties.class)
public class PaymentAutoConfiguration { }
```

```java
// Option C: make it a component. Works, but only with setter binding, not records.
@Component
@ConfigurationProperties(prefix = "orderflow.payment")
public class PaymentProperties { /* getters and setters */ }
```

> **If you do none of these, the class is never bound and nothing tells you.** Trap 3.

### Injecting it

```java
@Service
public class PaymentService {

    private final PaymentProperties props;

    public PaymentService(PaymentProperties props) {   // constructor injection, Topic 39
        this.props = props;
    }
}
```

The properties object is an ordinary singleton bean. In a unit test you construct it
directly — no Spring context needed:

```java
var props = new PaymentProperties(
        URI.create("http://localhost:9999"), "test-key",
        Duration.ofSeconds(1), 0,
        new PaymentProperties.Retry(Duration.ofMillis(10), 1.0),
        List.of("CARD"));
var service = new PaymentService(props);
```

That is a large part of why typed properties beat `@Value`: the class under test has no
opinion about where its configuration came from.

### `@Value` — what it is, and its two legitimate uses

```java
@Value("${orderflow.instance-id}")           private String instanceId;
@Value("${orderflow.payment.timeout:5000}")  private long timeoutMs;   // :5000 is a default
@Value("#{systemEnvironment['HOSTNAME']}")   private String hostname;  // SpEL
```

`@Value` is resolved by `PropertySourcesPlaceholderConfigurer`, which is a
`BeanFactoryPostProcessor` (Topic 35) — it rewrites the placeholder before the bean is
constructed.

Use it for exactly two things:
- A single, unrelated scalar in a class that has no other configuration.
- Inside a `@Bean` method where you need one value and creating a properties type is
  genuinely disproportionate.

Everything else is `@ConfigurationProperties`. A class with four `@Value` fields is a
`@ConfigurationProperties` record that has not been written yet.

### Profiles

```yaml
# application.yml — shared defaults
spring:
  application:
    name: orderflow-api
orderflow:
  payment:
    timeout: 5s

---
spring:
  config:
    activate:
      on-profile: load
orderflow:
  payment:
    timeout: 2s
```

```bash
# activate at run time
java -jar orderflow.jar --spring.profiles.active=docker,load
SPRING_PROFILES_ACTIVE=docker,load java -jar orderflow.jar
```

```java
@Profile("!load")                      // NOT load
@Profile({"docker", "load"})           // docker OR load
@Profile("docker & !debug")            // expression form
@Bean
SqlLoggingListener sqlLogging() { ... }
```

Profile **groups** let one name pull in several:

```yaml
spring:
  profiles:
    group:
      load: [docker, metrics, no-sql-logging]
```

Now `--spring.profiles.active=load` activates four profiles. Useful, and a good way to
confuse yourself — always confirm with `/actuator/env` which profiles are actually active.

### Importing extra config

```yaml
spring:
  config:
    import:
      - optional:file:./orderflow-local.yml          # a dev override, may be absent
      - optional:configtree:/etc/orderflow/secrets/  # k8s secret volume, one file per key
```

`configtree:` reads a directory where each *filename* is a property name and each file's
*content* is the value. That is exactly the shape a Kubernetes secret volume mount has.
Topic 123 uses this.

---

## The property precedence chain — in order, highest wins

This is the section to memorise. When two sources define the same key, the one **higher in
this list** wins.

| # | Source | You meet it as |
|---|---|---|
| 1 | Devtools global settings in `$HOME/.config/spring-boot` | only when spring-boot-devtools is on the classpath |
| 2 | `@TestPropertySource` on a test class | test-only overrides (Topic 60) |
| 3 | `properties` attribute of `@SpringBootTest` | test-only overrides |
| 4 | **Command-line arguments** (`--server.port=8081`) | `java -jar app.jar --x=y`, k8s `args:` |
| 5 | `SPRING_APPLICATION_JSON` (inline JSON in an env var or system property) | rare, but it beats plain env vars — worth knowing it exists |
| 6 | `ServletConfig` init parameters | legacy WAR deployment |
| 7 | `ServletContext` init parameters | legacy WAR deployment |
| 8 | JNDI attributes from `java:comp/env` | legacy app-server deployment |
| 9 | **Java system properties** (`-Dserver.port=8081`) | JVM flags, `JAVA_TOOL_OPTIONS` |
| 10 | **OS environment variables** | `export`, Docker `-e`, k8s `env:` — **this is the one that surprises people** |
| 11 | `RandomValuePropertySource` (`random.*` only) | `${random.uuid}` |
| 12 | **Profile-specific files OUTSIDE the jar** (`application-load.yml` next to the jar) | deployment-time override |
| 13 | **Profile-specific files INSIDE the jar** (`classpath:application-load.yml`) | what you commit |
| 14 | **`application.yml` OUTSIDE the jar** | deployment-time override |
| 15 | **`application.yml` INSIDE the jar** | what you commit — **the file you are usually staring at** |
| 16 | `@PropertySource` on a `@Configuration` class | legacy pre-Boot style |
| 17 | Default properties (`SpringApplication.setDefaultProperties(...)`) | set in code, lowest of all |

**The four rows that account for almost every real incident:**

- **4 beats 9 beats 10 beats 12–15.** Command line beats `-D` beats env var beats any file.
- **10 beats 15.** An OS environment variable silently beats the `application.yml` you have
  open in your editor. This is Trap 1 and it is the single most common configuration
  confusion in Spring.
- **12/14 beat 13/15.** Outside the jar beats inside the jar. A file dropped next to the jar
  at deploy time wins over the packaged one, without a rebuild.
- **Profile-specific beats non-profile-specific at the same location.** `application-load.yml`
  beats `application.yml`.

**Within a group:** when several profiles are active and both define a key, the profile
listed **later** in `spring.profiles.active` wins.

> **Do not trust this table over your own machine.** It is stable and correct to the best
> of my knowledge for Boot 3.x/4.x, but the ordering has shifted historically (notably the
> Boot 2.4 config-data rework), and I am reproducing it rather than reading your version's
> docs. `/actuator/env` is the authority for the app in front of you. That is the point of
> the Hands-on section.

### Relaxed binding — the exact rules

For a property named `orderflow.payment.max-retries`, all of these bind:

| Form | Where you would write it |
|---|---|
| `orderflow.payment.max-retries` | YAML / properties — **the canonical form, always write this** |
| `orderflow.payment.maxRetries` | camelCase, tolerated |
| `orderflow.payment.max_retries` | underscore, tolerated |
| `ORDERFLOW_PAYMENT_MAXRETRIES` | environment variable |
| `ORDERFLOW_PAYMENT_MAX_RETRIES` | environment variable |

**The environment-variable rule:** uppercase everything, replace `.` and `-` with `_`, and
drop any other non-alphanumeric character.

For lists, the index becomes another underscore-separated segment:

```
orderflow.payment.gateways[0].name
      becomes
ORDERFLOW_PAYMENT_GATEWAYS_0_NAME
```

**The important caveat:** full relaxed binding is a feature of the **binder**, so it
applies to `@ConfigurationProperties`. `@Value` goes through simple placeholder resolution
instead. The plain dotted-lowercase form usually still resolves from an environment
variable because the system-environment property source performs its own limited name
aliasing — but the camelCase and kebab-case tolerance is **not** available to `@Value`.

> **Flagged as genuine uncertainty:** the exact set of aliases the system-environment
> property source tries has varied across versions. Do not design around it. The rule to
> follow is simple and version-proof: **write the canonical kebab-case name in YAML, use
> `@ConfigurationProperties` for anything that might come from an environment variable, and
> verify with `/actuator/env`.**

---

## Example 1 — minimal

A single typed property, bound and validated.

```yaml
# src/main/resources/application.yml
orderflow:
  catalog:
    page-size: 50
```

```java
package com.orderflow.catalog;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;
import org.springframework.validation.annotation.Validated;

@Validated
@ConfigurationProperties(prefix = "orderflow.catalog")
public record CatalogProperties(
        @Min(1) @Max(200)
        @DefaultValue("25")
        int pageSize
) {}
```

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class OrderflowApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderflowApplication.class, args);
    }
}
```

**Predict these four outcomes before running anything:**

| What you do | What happens |
|---|---|
| nothing — run as above | `pageSize` is 50, from `application.yml` |
| delete the `page-size` line | `pageSize` is 25, from `@DefaultValue` |
| set `page-size: 500` | **startup fails**, with a message naming `orderflow.catalog.page-size` and the `@Max(200)` violation |
| `ORDERFLOW_CATALOG_PAGESIZE=10 ./mvnw spring-boot:run` | `pageSize` is 10 — the env var beats the file (precedence row 10 vs row 15) |

The third row is the one that matters. A bad configuration value becomes a **build-time
shaped failure** instead of a runtime one. That is the entire argument for typed
properties in one line.

---

## Example 2 — production scenario: three profiles for `orderflow`

### The constraints, stated concretely

`orderflow-api` must run in three shapes, and they differ in ways that cannot be papered
over with one file:

| | `local` | `docker` | `load` |
|---|---|---|---|
| Purpose | a laptop, one developer | the compose stack, integration testing | Topic 65's gate |
| Postgres | `localhost:5432`, tiny dataset | `postgres:5432` in the compose network | `postgres:5432`, **1.1M orders, 120k products, 5M order lines** |
| Target throughput | none | none | **1,800 rps sustained, open arrival model** |
| Latency SLO | none | none | **p99 < 40 ms** on catalogue read |
| Connection pool | 5 | 10 | **sized deliberately — see below** |
| SQL logging | on, formatted | off | **off, non-negotiable** |
| Payment gateway | in-process stub | wiremock container | in-process stub with a fixed 12 ms delay |
| Flyway migrations | on, clean allowed | on | **on, but never `clean`** |

The `load` profile is the one with hard requirements. SQL logging alone costs enough
string formatting and I/O to move a p99 by tens of milliseconds at 1,800 rps — turning it
on "just to see" invalidates a baseline. That is why it is configuration, not a checkbox
someone remembers.

### `application.yml` — shared defaults only

```yaml
spring:
  application:
    name: orderflow-api
  jpa:
    open-in-view: false          # Topic 49 — never true; it holds a connection through rendering
  threads:
    virtual:
      enabled: false             # Topic 101 turns this on deliberately, with measurements

server:
  shutdown: graceful             # Topic 123
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info

orderflow:
  catalog:
    page-size: 50
  payment:
    timeout: 5s
    max-retries: 2
    retry:
      initial-backoff: 200ms
      multiplier: 2.0
  inventory:
    reservation-ttl: 15m
```

Note what is **not** here: no host names, no credentials, no pool sizes. Shared defaults
are the values that are correct everywhere. Anything environment-shaped belongs in a
profile file, and anything secret belongs nowhere in the repository at all (Topic 123).

### `application-local.yml`

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderflow
    username: orderflow
    password: orderflow          # local only, and the DB is not reachable off-box
    hikari:
      maximum-pool-size: 5
  jpa:
    properties:
      hibernate:
        format_sql: true
  flyway:
    clean-disabled: false        # a laptop may reset its database

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # shows bound parameters

management:
  endpoints:
    web:
      exposure:
        include: "*"             # everything, locally only

orderflow:
  payment:
    gateway-url: http://localhost:9999/stub
    api-key: local-dev-key
```

### `application-docker.yml`

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/orderflow
    hikari:
      maximum-pool-size: 10
  flyway:
    clean-disabled: true

logging:
  level:
    org.hibernate.SQL: INFO

orderflow:
  payment:
    gateway-url: http://wiremock:8080/payments
```

No `username`, no `password`, no `api-key`. Those arrive as environment variables from
compose or from a Kubernetes secret, and relaxed binding maps them:

```yaml
# docker-compose.yml (fragment)
environment:
  SPRING_PROFILES_ACTIVE: docker
  SPRING_DATASOURCE_USERNAME: orderflow
  SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
  ORDERFLOW_PAYMENT_API_KEY: ${PAYMENT_API_KEY}
```

`ORDERFLOW_PAYMENT_API_KEY` → `orderflow.payment.api-key`. No mapping code anywhere. That
is the relaxed-binding payoff, and it is why the properties class must be
`@ConfigurationProperties` and not four `@Value` fields.

### `application-load.yml` — the Topic 65 profile

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/orderflow
    hikari:
      maximum-pool-size: 24         # deliberate; see the reasoning below
      minimum-idle: 24              # pre-warmed: a cold pool distorts the first minute
      connection-timeout: 2000      # fail fast rather than queue silently — Topic 109
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50            # Topic 53
        order_inserts: true
        order_updates: true
        generate_statistics: false  # ON would itself cost measurable CPU
  flyway:
    clean-disabled: true            # loading the 1.1M-row dataset takes ~40 minutes

logging:
  level:
    root: WARN
    org.hibernate.SQL: WARN
    com.orderflow: INFO

server:
  tomcat:
    threads:
      max: 200
    accept-count: 100

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,configprops,env,conditions
  metrics:
    tags:
      profile: load                 # so the baseline is attributable — Topic 118

orderflow:
  catalog:
    page-size: 50                   # MUST match production; page size changes the query plan
  payment:
    gateway-url: http://localhost:9999/stub
    timeout: 2s
    max-retries: 0                  # retries would hide latency in the baseline
  inventory:
    reservation-ttl: 15m
```

**The reasoning behind the two numbers that matter:**

- `maximum-pool-size: 24`. Not a guess and not `100`. A larger pool does not create more
  database CPU; past a point it converts connection wait into database contention and
  makes p99 worse while throughput stays flat. 24 is a starting hypothesis to be
  **measured and revised** at the gate. Topic 109 does the sizing properly; the point here
  is that the number is a recorded, reproducible part of the profile rather than a
  default someone forgot about.
- `minimum-idle` equal to `maximum-pool-size`. A pool that grows during the ramp puts
  connection establishment inside the measurement window and produces a first-minute
  latency spike that is an artefact of the test, not of the service.

### The typed properties, all of them

```java
@Validated
@ConfigurationProperties(prefix = "orderflow.inventory")
public record InventoryProperties(
        @NotNull @DefaultValue("15m") Duration reservationTtl,
        @Min(1)  @DefaultValue("100") int maxLinesPerOrder
) {}
```

```java
@Validated
@ConfigurationProperties(prefix = "orderflow.catalog")
public record CatalogProperties(
        @Min(1) @Max(200) @DefaultValue("25") int pageSize
) {}
```

Plus `PaymentProperties` from the Syntax breakdown. Three records, registered once with
`@ConfigurationPropertiesScan`, injected by constructor wherever they are needed. Zero
`@Value` annotations in the codebase.

### What running it looks like

```bash
# local
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# docker
docker compose up          # SPRING_PROFILES_ACTIVE=docker set in the compose file

# load — the gate
docker compose -f compose.load.yml up -d
java -jar orderflow-api.jar --spring.profiles.active=load
```

Note the last one uses a **command-line argument** (precedence row 4), which beats
everything including any stray `SPRING_PROFILES_ACTIVE` left in the shell. For a
measurement run you want the highest-precedence mechanism available, precisely so that
nothing in the environment can quietly change what you are measuring.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the environment variable that beats the file you are staring at

**Wrong:** you change `application.yml`, redeploy, and the behaviour does not change.

```yaml
# application.yml — you have edited this three times
orderflow:
  payment:
    timeout: 30s
```

**Exact symptom:** payments still time out at 2 seconds. The log line you added prints
`timeout=PT2S`. `grep -r "2s" src/` finds nothing. You start to suspect a caching problem,
a stale image, or the wrong pod. Restarting does not help. The value is correct in every
file you can find.

**Root cause:** somewhere — a Kubernetes deployment manifest, a compose file, a CI
variable, a `.bashrc`, a base image `ENV` line — there is:

```
ORDERFLOW_PAYMENT_TIMEOUT=2s
```

OS environment variables are **precedence row 10**. `application.yml` inside the jar is
**row 15**. The env var wins, silently, correctly, by design.

**Fix — diagnose first, then act:**

```bash
curl -s localhost:8080/actuator/env/orderflow.payment.timeout | jq .
```

This returns every source that supplies the key, in order. The first is the winner and it
is named. Look for `systemEnvironment`.

Then find where it is set:

```bash
env | grep -i ORDERFLOW
kubectl get deploy orderflow-api -o yaml | grep -A3 -i "ORDERFLOW_PAYMENT"
docker inspect <container> | jq '.[0].Config.Env'
```

**Prevention, in order of value:**
1. Log the effective configuration at startup — for a properties record,
   `log.info("payment config: {}", props)` gives you a full line in every environment.
   Never log secrets; see the sanitisation note in Hands-on Proof 2.
2. Expose `/actuator/env` in every non-production environment.
3. In code review, treat "add an env var override" as a change that needs a comment saying
   why it is not in a profile file.

---

### Trap 2 — `@Value` binding a string where `@ConfigurationProperties` would have failed at startup

**Wrong:**

```java
@Service
public class PaymentService {
    @Value("${orderflow.payment.timeout}")
    private String timeout;                    // a String

    public void charge(Order order) {
        var t = Duration.parse(timeout);       // parsed here, on the request path
        ...
    }
}
```

with a typo in the YAML:

```yaml
orderflow:
  payment:
    timeout: 30 seconds     # not a valid Duration
```

**Exact symptom:** the application starts perfectly. Health checks pass. Readiness passes.
The pod goes into the load balancer. Then the **first payment attempt** throws:

```
java.time.format.DateTimeParseException: Text '30 seconds' could not be parsed at index 0
```

If payments are 8% of your traffic, the failure appears minutes after deploy, after your
canary has already been promoted, and it looks like a payment-gateway incident rather than
a configuration one.

**Root cause:** `@Value` resolves a placeholder into a `String`. Nothing validates it. The
conversion happens at use time, so the blast radius is a request rather than a startup.

**Fix:**

```java
@Validated
@ConfigurationProperties(prefix = "orderflow.payment")
public record PaymentProperties(@NotNull @DefaultValue("5s") Duration timeout) {}
```

Now the same typo produces, **at startup**:

```
Failed to bind properties under 'orderflow.payment.timeout' to java.time.Duration
```

and the pod never becomes ready, so the rolling deploy halts and the previous version keeps
serving. The bug is contained by the deployment mechanism instead of reaching customers.

**The worse variant of the same trap:**

```java
@Value("${orderflow.payment.timeuot:5000}")   // typo in the KEY, plus a default
private long timeoutMs;
```

This never fails at all. The placeholder is unresolvable, the default kicks in, and the
value you configured is ignored forever with no signal whatsoever. A `@Value` with a
default is a permanent silent fallback for any typo in its own key.

---

### Trap 3 — the `@ConfigurationProperties` class that was never registered

**Wrong:**

```java
@Validated
@ConfigurationProperties(prefix = "orderflow.payment")
public record PaymentProperties(Duration timeout, String apiKey) {}
```

and the application class has neither `@ConfigurationPropertiesScan` nor
`@EnableConfigurationProperties(PaymentProperties.class)`.

**Exact symptom, depending on how you consumed it:**

| What you wrote | What you see |
|---|---|
| constructor injection of `PaymentProperties` | `NoSuchBeanDefinitionException: No qualifying bean of type 'com.orderflow.payment.PaymentProperties'` at startup. **Loud — the good case.** |
| `new PaymentProperties(...)` somewhere, or `@Autowired(required = false)` | all fields at their Java defaults: `null`, `0`, `false`. No error. Your `@Validated` constraints never ran, because binding never happened. |
| the class is a `@Component` with setters | it *is* a bean, and `@ConfigurationProperties` still binds it — this combination happens to work, which is why people are confused about when registration is needed |

The middle row is the silent one, and it is worse than it looks: `@Validated` gives you a
false sense of safety. Validation is performed **by the binder**, so if nothing binds,
nothing validates.

**Root cause:** `@ConfigurationProperties` on its own is inert metadata. Something must
tell Spring to create and bind an instance.

**Fix:** pick one registration mechanism and apply it consistently across the codebase.

- **Application code:** `@ConfigurationPropertiesScan` on the main class, once. Then adding
  a new properties record requires no registration step, which is the failure mode you are
  designing against.
- **Library / auto-configuration code (Topic 42):**
  `@EnableConfigurationProperties(PaymentProperties.class)` on the auto-configuration, so
  the properties type is bound only when the auto-configuration applies.

**How to prove it worked:**

```bash
curl -s localhost:8080/actuator/configprops | jq 'paths(scalars) as $p | select($p | join(".") | test("orderflow"))'
```

If your prefix does not appear in `/actuator/configprops` at all, the class is not
registered. That endpoint lists exactly the bound properties objects, which makes it the
direct test for this trap.

---

### Trap 4 — trying to activate a profile from inside a profile-specific file

**Wrong:**

```yaml
# application-load.yml
spring:
  profiles:
    active: load,metrics       # trying to pull in another profile
```

or the same key inside a `spring.config.activate.on-profile` document in `application.yml`.

**Exact symptom:** startup fails with an exception naming the property and the document:

```
InactiveConfigDataAccessException: Inactive property source 'Config resource ...
application-load.yml' imported from ... cannot contain property 'spring.profiles.active'
```

Or, on some configurations, it starts and the extra profile is simply not active — you
find out because a `@Profile("metrics")` bean is missing and the condition evaluation
report (Topic 42) shows it as not matched.

**Root cause:** profiles must be fully determined *before* Spring knows which
profile-specific documents to load. Allowing a profile-specific file to activate more
profiles would be circular: the file's own activation depends on the set it is trying to
change. Boot forbids it rather than resolving it with an arbitrary rule.

**Fix:** use `spring.profiles.group` in the **non**-profile-specific `application.yml`:

```yaml
# application.yml
spring:
  profiles:
    group:
      load: [docker, metrics, no-sql-logging]
```

Now `--spring.profiles.active=load` activates `load`, `docker`, `metrics` and
`no-sql-logging`. Groups are resolved during profile determination, which is the phase
where this decision legitimately belongs.

Verify what actually happened:

```bash
curl -s localhost:8080/actuator/env | jq '.activeProfiles'
```

---

### Trap 5 — a mutable properties class that something writes to at runtime

**Wrong:**

```java
@Component
@ConfigurationProperties(prefix = "orderflow.payment")
public class PaymentProperties {
    private int maxRetries;
    public int getMaxRetries() { return maxRetries; }
    public void setMaxRetries(int v) { maxRetries = v; }   // public setter, singleton bean
}
```

Then, in a retry handler or a test helper or an admin endpoint someone adds later:

```java
paymentProperties.setMaxRetries(0);      // "just for this call"
```

**Exact symptom:** at 1,800 rps, retries stop happening globally from a moment that is not
correlated with any deploy. Or: one integration test class changes a value and a
*different* test class in the same JVM fails, but only when the suite runs in a particular
order, and never when you run that class alone. Reproducing it takes hours because the
cause is elsewhere in the run.

**Root cause:** a `@ConfigurationProperties` bean is a **singleton** (Topic 38). A public
setter on a singleton is shared mutable state across every request thread, with no
synchronisation and no memory-model guarantees (Topic 17 — a non-`final`, non-`volatile`
field written by one thread is not guaranteed visible to others, so the behaviour can even
differ per thread).

**Fix:** use a `record`. Constructor binding, no setters, `final` fields, safe publication
for free.

```java
@Validated
@ConfigurationProperties(prefix = "orderflow.payment")
public record PaymentProperties(@Min(0) @Max(5) @DefaultValue("2") int maxRetries) {}
```

If you genuinely need a value to change at runtime, that is a *different feature* — a
feature flag or a dynamic-config service with explicit semantics, versioning and an audit
trail. It is not a properties setter. Reconfiguration by mutation is untraceable: nothing
records who changed it, when, or back to what.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and no running `orderflow`, so
nothing here is captured output. What follows is the exact command, what to look for, and
what each possible result means.

### Setup — expose the endpoints you will need

```yaml
# application-local.yml
management:
  endpoints:
    web:
      exposure:
        include: env,configprops,conditions,health,info
  endpoint:
    env:
      show-values: ALWAYS       # local only — see the warning below
    configprops:
      show-values: ALWAYS       # local only
```

> **Two warnings, both real.**
> 1. Actuator endpoints beyond `health` are **not exposed by default**. If `/actuator/env`
>    returns 404, that is configuration, not a bug.
> 2. `show-values: ALWAYS` prints secrets in plain text. By default Boot sanitises values
>    whose key looks sensitive (password, secret, key, token, credentials and similar) to
>    `******`. Turning that off is a local debugging convenience and must never reach a
>    shared environment. Topic 123 covers the production posture.

### Proof 1 — watch an environment variable beat `application.yml`

This is Trap 1, made visible in four commands.

```bash
# 1. baseline: value comes from the file
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
curl -s localhost:8080/actuator/env/orderflow.catalog.page-size | jq .
```

```bash
# 2. now set an env var and restart
ORDERFLOW_CATALOG_PAGESIZE=7 ./mvnw spring-boot:run -Dspring-boot.run.profiles=local
curl -s localhost:8080/actuator/env/orderflow.catalog.page-size | jq .
```

The response shape from `/actuator/env/{name}` is a `property` object (the winner) plus a
`propertySources` array listing **every** source that has the key, in precedence order.

| What you see | What it means |
|---|---|
| `propertySources[0].name` is `systemEnvironment` and its value is `7` | the env var won. Precedence row 10 beat row 15. The `application.yml` entry is still listed further down the array — visible but overridden. |
| `propertySources[0].name` contains `applicationConfig` and a `.yml` path | the file won; no env var is set. Check your spelling — `ORDERFLOW_CATALOG_PAGESIZE`, uppercase, underscores. |
| `.property` is `null` but `propertySources` has entries | the key exists in a source that is not currently active (a profile you did not activate). |
| HTTP 404 | the endpoint is not exposed. Add `env` to `management.endpoints.web.exposure.include`. |
| the value shows `******` | sanitisation. Set `show-values: ALWAYS` **locally** or pick a non-sensitive key for this experiment. |

Now climb one more rung of the ladder:

```bash
# 3. command-line argument beats the env var
ORDERFLOW_CATALOG_PAGESIZE=7 ./mvnw spring-boot:run \
  -Dspring-boot.run.profiles=local \
  -Dspring-boot.run.arguments=--orderflow.catalog.page-size=99
```

**Expect:** the winner is now `commandLineArgs` with value `99` — row 4 beating row 10.
If it is not, your argument did not reach the application; check that it is inside
`-Dspring-boot.run.arguments` and not a bare `-D`.

```bash
# 4. and a JVM system property sits between them
./mvnw spring-boot:run -Dspring-boot.run.profiles=local \
  -Dspring-boot.run.jvmArguments="-Dorderflow.catalog.page-size=55"
```

**Expect:** winner `systemProperties`, value `55` — row 9, which beats env vars (row 10)
and loses to command-line arguments (row 4).

**Do this once, properly, and write down each winner.** Four runs is fifteen minutes and it
converts the precedence table from something you have read into something you have seen.

---

### Proof 2 — see everything that was bound, with `/actuator/configprops`

```bash
curl -s localhost:8080/actuator/configprops | jq 'keys'
curl -s localhost:8080/actuator/configprops \
  | jq '.contexts.application.beans | with_entries(select(.key | test("orderflow"; "i")))'
```

`/actuator/env` shows raw key/value strings from every source.
`/actuator/configprops` shows the **bound objects** — the actual `PaymentProperties`
instance after conversion and defaults. They answer different questions.

| What you see | What it means |
|---|---|
| an entry with `prefix: "orderflow.payment"` and a `properties` object with your values | bound correctly. This is the effective configuration your code will use. |
| your prefix is **absent entirely** | the class is not registered. Trap 3. Add `@ConfigurationPropertiesScan` or `@EnableConfigurationProperties`. |
| present, but a field is `null` where you set a value | prefix or field-name mismatch. Compare the YAML key against the record component in kebab-case. |
| present, values are `******` | sanitisation, working as intended. |
| `timeout` shows `PT5S` where you wrote `5s` | correct — that is `Duration.toString()`. Conversion happened. |

The third row is worth dwelling on: a silently-`null` field means the binder looked for a
key you did not write. `orderflow.payment.apiKey` in YAML binding to a record component
`apiKey` works via relaxed binding, but `orderflow.payment.api_key` under a *different*
prefix does not. `/actuator/configprops` shows you the result rather than making you
reason about it.

---

### Proof 3 — prove which profiles are actually active

```bash
curl -s localhost:8080/actuator/env | jq '{active: .activeProfiles, default: .defaultProfiles}'
```

| What you see | What it means |
|---|---|
| `["load","docker","metrics","no-sql-logging"]` when you only passed `load` | a `spring.profiles.group` expanded it. Working as designed — find the group definition in `application.yml`. |
| `[]` and `["default"]` | no profile is active. Any `application-local.yml` is being ignored entirely, which explains "my local config does nothing". |
| more profiles than you expected | check `SPRING_PROFILES_ACTIVE` in the environment (`env \| grep SPRING`), and any `spring.profiles.active` in a parent config. |
| the profile you passed is missing | you passed it wrong. `--spring.profiles.active=load` as an application argument, or `SPRING_PROFILES_ACTIVE=load` as an env var. `-Dspring.profiles.active` also works as a JVM system property. |

Then confirm which **files** were loaded — this is the fastest way to catch a filename typo
like `application-loaad.yml`:

```bash
curl -s localhost:8080/actuator/env | jq '.propertySources[].name' | grep -i applicationConfig
```

**How to read it:** you should see one entry per config file that was actually found and
parsed, each naming its path. A file you expected and do not see was never loaded — check
the filename, then check it is under `src/main/resources`.

---

### Proof 4 — override the config location entirely

```bash
# replace the search path completely
java -jar target/orderflow-api.jar \
  --spring.config.location=file:/etc/orderflow/

# or add a location while keeping the defaults
java -jar target/orderflow-api.jar \
  --spring.config.additional-location=file:/etc/orderflow/

# or import specific files, tolerating absence
java -jar target/orderflow-api.jar \
  --spring.config.import=optional:file:/etc/orderflow/overrides.yml
```

| What you see | What it means |
|---|---|
| the app starts and your external values are in `/actuator/env` under a source naming that path | working. Note this is precedence row 12/14 — outside the jar beats inside. |
| `ConfigDataLocationNotFoundException` | the location does not exist and was not marked `optional:`. Prefix it with `optional:` if absence is acceptable. |
| the app starts but the **packaged** values are used | you used `location` (which *replaces* the search path) when you meant `additional-location`, or vice versa. This distinction is the single most common mistake with this flag. |
| `location` used, and now `application.yml`'s shared defaults are all gone | expected. `spring.config.location` replaces the entire default search path. That is almost never what you want; reach for `additional-location` or `import`. |

Practise the difference deliberately. `location` vs `additional-location` is a question you
will be asked in an interview and will get wrong under pressure if you have not run both.

---

### Proof 5 — make validation fail at startup, on purpose

Edit `application-local.yml`:

```yaml
orderflow:
  catalog:
    page-size: 5000        # violates @Max(200)
```

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```

| What you see | What it means |
|---|---|
| startup fails, and the message names `orderflow.catalog.page-size`, the value `5000`, and the constraint | correct. Note the **property name**, not the field name — the binder reports in configuration terms, which is what the person reading the log needs. |
| startup **succeeds** | one of three things: `@Validated` is missing from the class; the class is not registered (Trap 3 — check `/actuator/configprops`); or there is no Bean Validation implementation on the classpath. Check the last one with `./mvnw dependency:tree \| grep -i hibernate-validator`. |
| a `ConstraintViolationException` at first use rather than at startup | you validated in the wrong place — this is method validation on a service (Topic 45), not configuration binding. |

Now do the type-conversion version, which is the more common real failure:

```yaml
orderflow:
  payment:
    timeout: 30 seconds     # not a valid Duration
```

**Expect** a bind failure naming `orderflow.payment.timeout` and the target type
`java.time.Duration`. Compare that message with what Trap 2's `@Value` version gives you —
a `DateTimeParseException` on the request path, hours later, with no property name in it.
Running both is the argument for typed properties, made once, permanently.

---

### Proof 6 — verify relaxed binding for a list from environment variables

```yaml
orderflow:
  payment:
    enabled-methods:
      - CARD
      - WALLET
```

```bash
ORDERFLOW_PAYMENT_ENABLEDMETHODS_0=CARD \
ORDERFLOW_PAYMENT_ENABLEDMETHODS_1=BANK_TRANSFER \
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

curl -s localhost:8080/actuator/configprops \
  | jq '.. | objects | select(.prefix? == "orderflow.payment") | .properties.enabledMethods'
```

| What you see | What it means |
|---|---|
| `["CARD","BANK_TRANSFER"]` | the indexed env-var form bound. Note it **replaced** the YAML list rather than merging with it — list binding is replace-not-merge, and a partially-overridden list is a classic surprise. |
| `["CARD","WALLET"]` | the env vars did not bind. Check the index form and that the property is on a `@ConfigurationProperties` type, not a `@Value` field. |
| an empty list | the env vars bound but the YAML did not — check your prefix. |

The "replaced, not merged" behaviour is worth internalising: setting index 0 of a
three-element list from the environment gives you a **one**-element list, not a list with
one element changed.

---

## Practice exercises

### 1 — Easy: climb the precedence ladder

Take the minimal `CatalogProperties` from Example 1.

Set `orderflow.catalog.page-size` to a **different value** in each of these five places at
once:

1. `application.yml` (inside the jar) → `10`
2. `application-local.yml` (inside the jar) → `20`
3. an `application.yml` file placed next to the built jar → `30`
4. an OS environment variable → `40`
5. a command-line argument → `50`

Run with `--spring.profiles.active=local` and record which value wins. Then remove the
winner and re-run. Repeat until you have observed all five, in order.

**Deliverable:** the five values in the order they won, plus for each run the
`propertySources[0].name` from `/actuator/env/orderflow.catalog.page-size`. Then state
which two rows of the precedence table you would have got wrong from memory.

---

### 2 — Medium: replace every `@Value` in `orderflow` (combines Topics 17, 26, 27, 35, 39)

You are given (or you write) an `orderflow` service class with this shape:

```java
@Service
public class PaymentService {
    @Value("${orderflow.payment.gateway-url}")            private String gatewayUrl;
    @Value("${orderflow.payment.api-key}")                private String apiKey;
    @Value("${orderflow.payment.timeout:5000}")           private long timeoutMs;
    @Value("${orderflow.payment.max-retries:3}")          private int maxRetries;
    @Value("${orderflow.payment.retry.multiplier:2.0}")   private double multiplier;
    @Value("${orderflow.payment.enabled-methods:CARD}")   private String enabledMethods;
}
```

Requirements:

1. **Topic 27:** replace all six with one `record`, using nested records where the shape is
   nested. Explain in a comment why a record is the right carrier here and what "shallowly
   immutable" means for the `enabledMethods` list.
2. **Topic 17:** the list field must not be mutable by a caller. Show what you did and say
   which of "final field", "defensive copy at construction" and "unmodifiable view" you
   used, and why the others were not enough on their own.
3. **Topic 39:** convert `PaymentService` to constructor injection with `final` fields.
   State the specific benefit here — not the general one.
4. **Topic 26:** `enabledMethods` may legitimately be empty. Decide whether that is an
   `Optional`, an empty `List`, or a validation failure. Justify the choice; there is a
   defensible answer for each and a bad reason for each.
5. **Topic 35:** explain, in two sentences, why `@Value` can be resolved before the bean is
   constructed at all. Name the post-processor involved.
6. Add `@Validated` with constraints such that **every one of these** fails at startup:
   a missing `api-key`, a `timeout` of `"30 seconds"`, a `max-retries` of `-1`, and a
   `gateway-url` of `"not a url"`.

**Deliverable:** the record, the refactored service, a unit test that constructs
`PaymentProperties` directly with no Spring context, and four startup failure messages —
one per invalid configuration — with the property name each one reports.

---

### 3 — Hard: build the `load` profile and prove it is reproducible

This exercise produces an artefact Topic 65 depends on. Do not skip it and do not fake it.

**Part A — build the three profiles.** Implement `local`, `docker` and `load` exactly as
in Example 2, including the `compose.load.yml` stack (`orderflow` + Postgres + Redis).
No secret, no credential and no host name may appear in any committed file.

**Part B — prove profile isolation.** For each profile, capture:

```bash
curl -s localhost:8080/actuator/env | jq '.activeProfiles'
curl -s localhost:8080/actuator/configprops \
  | jq '.contexts.application.beans | with_entries(select(.key | test("orderflow"; "i")))'
```

Diff the three `configprops` outputs. Every difference must be intentional and explained in
one line. **An unexplained difference between `docker` and `load` is a defect** — it means
your baseline measures a configuration you did not design.

**Part C — the reproducibility check.** The Topic 65 gate rule is that re-running the
baseline must land within ±10%. Configuration drift is one of the ways that fails.
Write a script that:

1. starts the app under the `load` profile,
2. dumps `/actuator/configprops` and `/actuator/env` to a file,
3. hashes the `orderflow.*` and `spring.datasource.*` subtrees,
4. fails if the hash differs from a committed baseline.

Commit the baseline hash. Now deliberately break it: set
`SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE=100` in the environment and re-run. Your script
must fail. **If it does not, your script is checking the file rather than the effective
configuration, which is the exact mistake this whole topic is about.**

**Part D — measure one configuration change.** With the compose stack loaded to at least
100k products, run a fixed 3-minute load against `GET /products` twice: once with
`logging.level.org.hibernate.SQL: DEBUG` and once with `WARN`. Record p50/p95/p99 and
throughput for both.

Report the delta honestly. If it is small, say so with the numbers — "SQL logging cost 4 ms
at p99, less than I expected" is a better answer than a confident wrong one. Then state
the rule you would put in a code review checklist as a result.

**Part E — argue against yourself.** Three profile files is already a maintenance surface.
Make the case for collapsing to **one** `application.yml` with everything supplied by
environment variables (the strict 12-factor position). Then make the case against. State
which you would ship for `orderflow` today and what would change your mind — for example,
at what number of environments or what team size does your answer flip?

---

## Interview questions

### Q1 — "Why did my environment variable override `application.yml`?"

**Mid-level answer:** "Environment variables have higher priority than the properties file.
That's how Spring Boot works — externalised config beats packaged config."

**Senior answer:** "Because Spring resolves properties by walking an ordered list of
`PropertySource` objects and returning the first hit, and OS environment variables sit
above config files in that order. Roughly: command-line arguments, then
`SPRING_APPLICATION_JSON`, then JVM system properties, then OS environment variables, then
profile-specific files outside the jar, then profile-specific inside, then plain
`application.yml` outside, then inside. So an env var beats every file, and a `-D` beats
the env var.

The design reason is 12-factor: the artifact is immutable and the environment
differentiates it, which is what makes the same jar promotable from staging to production.

What matters operationally is that this is diagnosable rather than guessable. I'd hit
`/actuator/env/orderflow.payment.timeout`, which returns every source holding that key in
precedence order — the first one is the winner and it's named. That turns 'why is my config
being ignored' into a thirty-second lookup.

And I'd want the effective config logged at startup, because the person hitting this at 3am
may not have an exposed Actuator."

**What separates them:** giving the actual order rather than "higher priority", naming the
resolution mechanism (ordered sources, first hit wins), naming the diagnostic endpoint, and
noting the *design intent* rather than treating it as an arbitrary rule.

**Interviewer's follow-up:** "How does `ORDERFLOW_PAYMENT_MAX_RETRIES` become
`orderflow.payment.max-retries`?" That is relaxed binding: uppercase, `.` and `-` become
`_`, other non-alphanumerics dropped. Volunteer the limitation — full relaxed binding
belongs to the binder, so it applies to `@ConfigurationProperties`; `@Value` goes through
plain placeholder resolution and does not get the same tolerance.

---

### Q2 — "`@Value` or `@ConfigurationProperties`? When would you use each?"

**Mid-level answer:** "`@ConfigurationProperties` for groups of related properties,
`@Value` for one-offs. `@ConfigurationProperties` is type-safe and gives you IDE
autocomplete."

**Senior answer:** "Default to `@ConfigurationProperties`; `@Value` is the exception, and I
can name the specific failure it causes.

`@Value` gives you a string resolved from a placeholder, converted at use time. So a typo
in the *value* fails when the code path first executes — which for a payment timeout might
be minutes after the pod passed its readiness check and entered the load balancer. And a
typo in the *key*, if there's a default, never fails at all: the placeholder is
unresolvable, the default silently applies, and your configured value is ignored forever
with no signal.

`@ConfigurationProperties` binds and converts at startup, so both of those become a
`Failed to bind properties under 'orderflow.payment.timeout'` and the application does not
start. That's not just an earlier error, it's a *contained* one — a pod that doesn't start
is caught by the rolling deploy and the previous version keeps serving. A pod that starts
and then throws on real traffic is an incident.

There are three secondary reasons. It's testable: a record is a plain object I construct in
a unit test with no Spring at all. It gets full relaxed binding, which is what makes a
Kubernetes `env:` block work with no translation code. And `@Validated` plus Jakarta
constraints means my invariants — `maxRetries` between 0 and 5 — are enforced by the
framework instead of by a comment.

I'd use `@Value` for a genuinely isolated scalar in a class with no other configuration.
Four `@Value` fields in one class is a properties record that hasn't been written yet."

**What separates them:** the specific *timing* of each failure and why timing determines
blast radius, the silent-default-on-key-typo case, and testability framed as a design
property rather than a convenience.

**Interviewer's follow-up:** "Records or a class with setters?" Records: constructor
binding is automatic, fields are `final`, and there is no public setter for someone to call
on a singleton at runtime. Setters on a `@ConfigurationProperties` bean are shared mutable
state across every request thread.

---

### Q3 — "How do profiles work, and how do you handle secrets across environments?"

**Mid-level answer:** "You create `application-dev.yml`, `application-prod.yml` and so on,
and set `spring.profiles.active`. Secrets go in the profile file for that environment, or
in environment variables."

**Senior answer:** "Profiles first. A profile activates config documents and beans. Several
can be active at once — `spring.profiles.active=docker,load` — and later ones win on a key
collision. `spring.profiles.group` lets one name expand into several, which is how I keep
`load` meaning one thing. And a profile-specific file can't activate another profile; that
would be circular, and Boot fails fast with an `InactiveConfigDataAccessException` rather
than resolving it arbitrarily.

Secrets: they don't go in profile files, because those are in Git. The layering I'd use is
non-secret environment shape in profile files — pool sizes, hostnames, log levels — and
secrets injected at runtime, above all files in the precedence chain.

Concretely, in Kubernetes I'd mount a secret as a volume and use
`spring.config.import: optional:configtree:/etc/orderflow/secrets/`. Configtree reads a
directory where each filename is a property name and each file's content is the value,
which is exactly the shape of a mounted secret. That's better than environment variables
for secrets because env vars leak into crash dumps, child processes and anything that
prints the environment.

I'd also make sure Actuator's sanitisation stays on in production — `/actuator/env`
redacts values whose key looks sensitive, and `show-values: ALWAYS` must never leave a
local machine. And the endpoint itself needs to be behind authentication."

**What separates them:** multiple simultaneous profiles and their ordering, the
profile-cannot-activate-a-profile rule, configtree over env vars for secrets *with the
reason*, and remembering that the diagnostic endpoint is itself a disclosure surface.

**Interviewer's follow-up:** "What if a secret rotates while the app is running?" Honest
answer: Boot's standard binding is startup-time, so the app keeps the old value.
`@RefreshScope` from Spring Cloud can rebind, at the cost of a proxy per refreshable bean.
The simpler and more common answer is that rotation triggers a rolling restart, which you
need to be able to do safely anyway (Topic 123).

---

### Q4 — "You set a property and the application behaves as if you hadn't. Debug it."

**Mid-level answer:** "I'd check the file is in the right place, check for typos, and make
sure the profile is active. Then add a log statement to print the value."

**Senior answer:** "There are four distinct causes and they have different tells, so I'd
identify which one it is before changing anything.

One: something higher in the precedence chain is winning.
`/actuator/env/{property}` lists every source holding the key in order — the first is the
winner and it's named. If it says `systemEnvironment`, I go looking in the deployment
manifest, not in the repo.

Two: the property is being read but never bound.
`/actuator/configprops` shows the bound objects. If my prefix isn't there at all, the
`@ConfigurationProperties` class was never registered — no `@ConfigurationPropertiesScan`,
no `@EnableConfigurationProperties`. That's silent, and `@Validated` gives false comfort
because validation runs at bind time, so if nothing binds, nothing validates.

Three: the file was never loaded.
`/actuator/env` lists the config sources by path. A filename typo like `application-loaad.yml`
produces no error whatsoever — the file simply isn't there as far as Boot is concerned.

Four: the profile isn't active. `/actuator/env` has `activeProfiles`. An empty list plus
`default` means every `application-{profile}.yml` was ignored.

Adding a log statement is my *last* step, not my first, because it only tells me the
value — these tell me where the value came from, which is the actual question."

**What separates them:** four named causes with a distinct diagnostic for each, and
recognising that "print the value" answers the wrong question.

**Interviewer's follow-up:** "Actuator isn't exposed in that environment. Now what?" Log
the bound properties object at startup — a record's `toString()` gives you the whole thing
in one line, with sensitive fields excluded. Failing that, `--debug` still prints the
condition evaluation report (Topic 42), which will at least tell you whether a
property-conditional configuration matched.

---

### Q5 — "Walk me through what happens between `SpringApplication.run` and your first bean being constructed, from a configuration point of view."

**Mid-level answer:** "Spring reads `application.yml`, works out the profiles, and then
creates the beans with the values injected."

**Senior answer:** "The key thing is that configuration is fully resolved *before* any of
my beans exist, and Topic 35's two-phase startup is why that's possible.

Roughly: `SpringApplication` creates the `Environment` and populates it with the
non-file sources — command-line arguments, system properties, OS environment. Then the
config-data machinery determines the active profiles from what's in the Environment so far,
and only then loads `application.yml` and the profile-specific documents, resolving
`spring.config.import` as it goes. The result is an ordered list of `PropertySource`s.

Then the context refreshes. Bean *definitions* are registered — nothing is constructed
yet. `BeanFactoryPostProcessor`s run against those definitions, and one of them is
`PropertySourcesPlaceholderConfigurer`, which rewrites every `${...}` placeholder in the
definitions. Auto-configuration conditions including `@ConditionalOnProperty` are evaluated
in this phase too, which is exactly why they can decide anything at all — they're asking
the Environment, which is already complete.

Only after all of that does singleton pre-instantiation start and my beans get constructed.
`@ConfigurationProperties` binding happens as those beans are created, via a
`BeanPostProcessor`, which is where `@Validated` constraints fire.

The practical consequence: a configuration error is a startup error by construction. There
is no window where a bean exists with a half-resolved configuration."

**What separates them:** ordering profiles-before-files, naming
`PropertySourcesPlaceholderConfigurer` as a `BeanFactoryPostProcessor`, connecting it to
why `@ConditionalOnProperty` works, and drawing the conclusion about failure timing rather
than just reciting steps.

**Interviewer's follow-up:** "So can a bean influence which profiles are active?" No — by
the time your beans exist, profiles are long settled. If you need that, it is an
`EnvironmentPostProcessor` registered in `spring.factories`, which runs before the context
exists at all. Ask hard whether you really need it; it is a powerful and rarely-justified
hook.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Command-line arguments sit at the very top of the precedence chain, above environment
   variables. Why that way round? Give a concrete operational situation where the reverse
   ordering would be actively harmful.

2. `@Value` with a default (`${a.b:5}`) silently swallows a typo in the key. Why does
   Spring not warn about an unresolvable placeholder that has a default? Argue that the
   current behaviour is right, then argue it is wrong.

3. Relaxed binding accepts four spellings of the same property. That is convenience. Name
   the cost — what becomes harder or less safe because `maxRetries`, `max-retries`,
   `max_retries` and `MAXRETRIES` are all the same key?

4. `@Validated` on a `@ConfigurationProperties` class is enforced by the **binder**, not by
   a proxy — unlike `@Validated` on a service class (Topic 45). Why does that distinction
   exist, and what would go wrong if configuration validation were proxy-based?

5. A profile-specific file is forbidden from setting `spring.profiles.active`. Construct
   the circular scenario that rule prevents, concretely, with two files.

6. `spring.config.location` replaces the default search path; `additional-location` extends
   it. If you were designing this API, would you have made `location` mean "extend" and
   required an explicit flag to replace? Argue both sides, then say which surprises fewer
   people.

7. Your Kubernetes deployment sets 40 environment variables. Your `application.yml` has 60
   properties. A new engineer asks "where is the timeout configured?" Describe the process
   you would teach them — and then say what would have to be true about your setup for that
   question to have an obvious answer without running anything.

---

## Quick reference card

### Precedence — highest wins

```
 1  devtools global settings (~/.config/spring-boot)
 2  @TestPropertySource
 3  @SpringBootTest(properties = ...)
 4  command-line arguments            --x=y
 5  SPRING_APPLICATION_JSON
 6  ServletConfig init params
 7  ServletContext init params
 8  JNDI (java:comp/env)
 9  JVM system properties             -Dx=y
10  OS environment variables          X_Y=z          <- beats every file
11  RandomValuePropertySource         random.*
12  profile-specific files OUTSIDE the jar
13  profile-specific files INSIDE the jar
14  application.yml OUTSIDE the jar
15  application.yml INSIDE the jar                   <- the file you are looking at
16  @PropertySource
17  SpringApplication.setDefaultProperties
```

Within active profiles: the one listed **later** wins.

### Relaxed binding

```
orderflow.payment.max-retries      <- canonical; always write this in YAML
orderflow.payment.maxRetries
orderflow.payment.max_retries
ORDERFLOW_PAYMENT_MAX_RETRIES      <- env var: uppercase, . and - become _
ORDERFLOW_PAYMENT_GATEWAYS_0_NAME  <- list index becomes a segment
```

Full relaxed binding is a `@ConfigurationProperties` feature. `@Value` does not get it.

### The annotations

```java
@ConfigurationProperties(prefix = "orderflow.payment")   // bind a subtree
@ConfigurationPropertiesScan                             // on the app class — find them all
@EnableConfigurationProperties(X.class)                  // in a library / auto-config
@Validated                                               // fail at startup on constraint violation
@DefaultValue("5s")                                      // default for a record component
@DefaultValue                                            // construct a nested record from its own defaults
@Value("${a.b:fallback}")                                // one scalar; the exception, not the default
@Profile("load") / @Profile("!load") / @Profile("a & !b")
```

### Profiles

```yaml
spring:
  profiles:
    active: docker,load
    group:
      load: [docker, metrics]
---
spring:
  config:
    activate:
      on-profile: load
```

```bash
--spring.profiles.active=load           # command line (highest)
SPRING_PROFILES_ACTIVE=load             # env var
-Dspring.profiles.active=load           # system property
```

A profile-specific file may **not** set `spring.profiles.active`. Use a group.

### Config location flags

```bash
--spring.config.location=file:/etc/orderflow/               # REPLACES the search path
--spring.config.additional-location=file:/etc/orderflow/    # ADDS to it
--spring.config.import=optional:file:./local.yml
--spring.config.import=optional:configtree:/etc/secrets/    # one file per property
```

### Diagnostics

```bash
curl -s :8080/actuator/env/orderflow.payment.timeout | jq .   # who won, and who else has it
curl -s :8080/actuator/env | jq '.activeProfiles'             # which profiles
curl -s :8080/actuator/env | jq '.propertySources[].name'     # which files loaded
curl -s :8080/actuator/configprops | jq .                     # the BOUND objects
```

Endpoints are not exposed by default:
```yaml
management.endpoints.web.exposure.include: env,configprops,conditions,health
```
Values are sanitised by default. `show-values: ALWAYS` is local-only.

### Converters you get free

```
5s  500ms  2m  PT30S       -> java.time.Duration
10MB  512KB  1GB           -> org.springframework.util.unit.DataSize
SEQUENTIAL                 -> your enum (case-insensitive, - and _ tolerated)
http://x/y                 -> java.net.URI / URL
a,b,c                      -> List<String>
```

### Gotchas checklist

- [ ] An env var beats `application.yml`. Check `/actuator/env` before editing a file twice.
- [ ] `@Value` with a default silently hides a typo in its own key, forever.
- [ ] `@ConfigurationProperties` does nothing unless registered. Verify in `/actuator/configprops`.
- [ ] `@Validated` runs at bind time — if binding never happens, validation never happens.
- [ ] `location` replaces, `additional-location` adds.
- [ ] A profile file cannot activate a profile. Use `spring.profiles.group`.
- [ ] Records, not setters. A setter on a singleton is shared mutable state.
- [ ] List binding replaces; it does not merge.
- [ ] `jakarta.validation`, never `javax.validation`.
- [ ] Never commit a secret to a profile file, even for `local`.

---

## `[BOOT 3.x DELTA]`

| Change | Boot 2.x | Boot 3.x / 4.x |
|---|---|---|
| Config file processing | rewritten in **2.4**; the pre-2.4 behaviour was available via `spring.config.use-legacy-processing` | legacy flag **removed**. The 2.4+ config-data model is the only one. |
| Multi-document profile key | `spring.profiles: load` | **`spring.config.activate.on-profile: load`**. The old key is gone. |
| `spring.profiles.include` | free-form | usable only in non-profile-specific documents; `spring.profiles.group` is the modern mechanism |
| Constructor binding | `@ConstructorBinding` was required, and lived in `org.springframework.boot.context.properties` | a **single-constructor** class (and every record) uses constructor binding automatically. `@ConstructorBinding` moved to `...context.properties.bind.ConstructorBinding` and is now only needed to pick between multiple constructors. |
| Validation package | `javax.validation.*` | **`jakarta.validation.*`**. Topic 127. |
| `/actuator/env` values | shown by default in older lines | **sanitised by default**; `management.endpoint.env.show-values` is `NEVER` / `ALWAYS` / `WHEN_AUTHORIZED` |

**Boot 4 notes specific to this topic:**

- Modularisation means the artifact providing a given piece of configuration support may
  have moved. Resolve coordinates from `./mvnw dependency:tree`; never type a Spring
  version number — the Boot BOM supplies it (Topic 32).
- Jackson 3 is standard with Jackson 2 deprecated, which matters here only via
  `spring.jackson.*` property names. **Flagged: verify those property names against your
  resolved Boot version's reference documentation rather than against memory or a blog
  post.**
- The IDE autocomplete for your own properties comes from the configuration metadata
  annotation processor. Its artifact coordinates are affected by Boot 4's modularisation —
  find it in the reference docs for your version rather than copying it from an older
  project.

---

## When would I use this at work?

**1. The deployment that "ignored" a config change.**
Someone edits `application.yml`, ships it, and nothing changes. You hit
`/actuator/env/{property}`, see `systemEnvironment` at the top, and find a stale
`ORDERFLOW_*` entry in a Helm values file from a migration two quarters ago. Five minutes
instead of an afternoon, and — more valuable — you can now explain *why* it happened, so it
does not happen again on the next property.

**2. Onboarding a service into a new environment.**
A new region, a new cluster, a compliance-isolated deployment. What changes is a profile
file plus a set of injected secrets; what does not change is the artifact. Getting that
boundary right — environment shape in profile files, secrets injected above all files —
is the difference between a two-hour rollout and a week of one-off patches. Topic 123
builds directly on this.

**3. Making a performance baseline trustworthy.**
Topic 65's gate rule is ±10% on re-run. Half the ways that rule gets violated are
configuration drift: SQL logging left on, a different pool size, a page size that changes
the query plan, a cache that was warm last time. A `load` profile that is committed,
diffable and asserted-on is what makes a performance number an engineering measurement
rather than an anecdote.

---

## Connected topics

**Prerequisites:**

- **17 — Immutability and safe publication**: why a `@ConfigurationProperties` record with
  `final` fields is safe to share across every request thread, and why a setter on a
  singleton is not.
- **26 — Optional**: whether an optional configuration value is an `Optional` field, a
  default, or a validation failure. (Spoiler: `Optional` as a field is the one option
  Topic 26 tells you to avoid.)
- **27 — Records**: constructor binding, the compact constructor, and shallow immutability
  when a property is a `List`.
- **31–32 — Maven and dependency resolution**: config files are classpath resources, so
  which jar wins for `application.yml` is a resolution question. Never type a Spring
  version; let the BOM supply it.
- **35 — `ApplicationContext` two-phase startup**: the `Environment` is complete before any
  bean is constructed. `PropertySourcesPlaceholderConfigurer` is a
  `BeanFactoryPostProcessor`. This is why configuration errors are startup errors.
- **37 — Bean lifecycle**: `@ConfigurationProperties` binding happens via a
  `BeanPostProcessor` during bean creation — which is when `@Validated` fires.
- **38 — Bean scopes**: a properties object is a singleton. That is the whole argument
  against setters.
- **39 — Injection styles**: inject the properties record by constructor into a `final`
  field.
- **42 — Auto-configuration**: `@ConditionalOnProperty` asks the `Environment` a question,
  so its answer is only as predictable as your grasp of precedence. Read 42 and 43 as a
  pair.

**This unlocks:**

- **44 — `@RestController`**: `server.*`, `spring.mvc.*` and content-negotiation properties
  are configured exactly this way.
- **45 — Bean Validation**: the same Jakarta constraints, applied to request DTOs instead
  of configuration. Note the difference in *when* they run — bind time versus method
  invocation.
- **46 — Error handling**: `server.error.*` controls what leaks into an error response.
- **55 / 109 — Transactions and HikariCP**: `spring.datasource.hikari.*` is where pool
  sizing lives, and the `load` profile is where you record the number you chose.
- **60 — Test slices and context caching**: `@TestPropertySource` and
  `@SpringBootTest(properties=...)` sit at the top of the precedence chain — **and every
  distinct property override creates a new cached context**, which is a large part of why a
  suite is slow.
- **61 — Testcontainers**: `@ServiceConnection` injects a `DynamicPropertySource`, which is
  another property source in the same ordered list.
- **65 — GATE**: the `load` profile is a deliverable of that gate. The reproducibility rule
  depends on the configuration being reproducible first.
- **118 — Metrics**: `management.metrics.tags.*` is how a baseline is attributed to a
  profile.
- **121 — Actuator**: exposure rules, health groups, and the security posture of
  `/actuator/env` and `/actuator/configprops`.
- **122 — Docker and startup**: environment variables in the image, and the
  `spring.config.location` decision at container start.
- **123 — Config, secrets and graceful shutdown**: `configtree:` for mounted secrets, and
  what happens to configuration during a rolling deploy. The direct sequel to this topic.
- **127 — Migration planning**: `javax` → `jakarta`, `spring.profiles` →
  `spring.config.activate.on-profile`, and the 2.4 config-data rework.

---

*Java baseline 21, runtime JDK 25. Spring Boot 4.1 / Framework 7.0 / Jakarta EE 11.
The precedence table and the relaxed-binding rules in this document are reproduced from the
externalised-configuration model that has been stable since Boot 2.4; I have flagged the
two places where I am reproducing rather than reading a live source. `/actuator/env` on the
application in front of you outranks anything written here — that is the point of the topic.
No Spring artifact version numbers appear anywhere in this document by design.*
