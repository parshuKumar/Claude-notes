# 61 — Testcontainers — Real Postgres, Kafka, Redis in Tests

## Phase: 6 — Testing
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: every `orderflow` persistence test runs against a real Postgres container — the same major version production runs, with the same Flyway migrations applied. Kafka containers arrive at Topic 113; Redis arrives with the cache work. The container fixture built here is also the thing Topic 65's gate stack is derived from.

---

## Mastery line (from the master plan)

> You run one shared container per suite (singleton or reuse), not one per class, and
> you can explain why an H2-based test suite gives you false confidence about
> Postgres.

## Mid → Senior (from the master plan)

> "We test against H2, it's faster" → "H2 has different SQL semantics, different
> locking, different constraint behaviour and no `SKIP LOCKED`. Every bug it misses
> is one that only appears in production. Testcontainers with a singleton container
> and `@ServiceConnection` costs seconds, not minutes."

---

## ELI5 anchor

You are learning to drive.

**Option A — the driving simulator.** It is instant. It never runs out of fuel. It
starts up in a second. It has a steering wheel, pedals, a road. You can practise for
a hundred hours and get very good at it.

Then you sit in a real car and discover the clutch bites at a different point, the
brakes need more pressure than you expected, and the car rolls backwards on a hill
in a way the simulator never modelled.

**Option B — a real car in an empty car park.** Slower to set up. You have to unlock
it, start the engine, wait a few seconds. But the clutch is the real clutch.

**H2 is the simulator. Testcontainers is the real car in the car park.**

Testcontainers starts a real Postgres — the actual database program, in a container,
on your laptop — runs your tests against it, and throws it away afterwards. The cost
is a few seconds of startup, paid **once per test run** if you set it up correctly.
The benefit is that every behaviour you test is the behaviour production will have,
because it is the same program.

That is the whole idea. Everything below is how to pay the few seconds once instead
of once per test class.

---

## The bridge from what you know

### The honest bit first: you probably do not have this habit

In most Node and NestJS teams, integration tests against a real database are done
one of three ways:

1. A `docker-compose.yml` that a human runs by hand before `npm test`, and that CI
   runs as a separate step.
2. A `beforeAll` that connects to a database URL from an environment variable, and
   everyone hopes the right database is up.
3. `sqlite`/an in-memory adapter, or mocking the repository layer entirely.

There are Node libraries that do what Testcontainers does — `testcontainers` on npm
is a real, maintained port — but **in most teams' actual practice it is not the
default**, and there is no single tool everyone reaches for. Say this plainly in an
interview if asked: "In Node I usually saw a compose file managed outside the test
run; Testcontainers moves that lifecycle inside the test process, which makes it
reproducible and makes CI identical to local."

**Verdict: NO COMMON ANALOGUE in practice.** This is genuinely a new habit, and it
is one of the few places where the Java ecosystem's default is clearly better than
the Node ecosystem's default.

### What transfers

| What you do in Node | Java equivalent | Verdict |
|---|---|---|
| `docker compose up -d postgres` before tests | Testcontainers starts it *from inside the test JVM* | **PARTIAL** — same containers, different owner of the lifecycle |
| `process.env.DATABASE_URL` pointing at that container | `@ServiceConnection` or `@DynamicPropertySource` wiring the container's random port into Spring | **PARTIAL** |
| `beforeAll` / `afterAll` to migrate and truncate | JUnit 5 lifecycle plus Flyway plus explicit truncation | **FULL** |
| `jest --runInBand` because the shared DB cannot take parallel writers | The same problem, same shape | **FULL** |
| "It works on my machine but not in CI" because the local DB drifted | Testcontainers removes this class of failure outright | **NO ANALOGUE** — this is the payoff |

### The cost, stated honestly

**Seconds, not minutes.** A Postgres container on a warm Docker daemon with the
image already pulled starts in a small number of seconds. If you use the singleton
pattern in this document, you pay that **once for the entire test run**, not once per
test class. With reuse enabled (also in this document) you pay it **once per week**,
because the container survives between runs.

If your Testcontainers suite takes minutes, you have made one of the mistakes in the
"Wrong approach" section. That is a configuration defect, not the cost of the tool.

I am not going to tell you how many seconds it takes on your machine, because I have
not run it on your machine. The hands-on section shows you exactly where the number
is printed.

---

## What is this?

**Testcontainers** is a Java library that starts Docker containers as part of your
test lifecycle and stops them afterwards.

That is it. There is no magic. It talks to your Docker daemon over the Docker API,
runs `docker run` equivalents, waits until the container is genuinely ready (not just
"started"), hands you the mapped host port, and tears down at the end.

Three parts you should be able to name:

| Part | Artifact | What it does |
|---|---|---|
| **Core** | `org.testcontainers:testcontainers` | The `GenericContainer` class, wait strategies, Ryuk. |
| **JUnit 5 integration** | `org.testcontainers:junit-jupiter` | The `@Testcontainers` extension and the `@Container` annotation. |
| **Modules** | `org.testcontainers:postgresql`, `:kafka`, and others | Pre-configured subclasses that know the image's ports, env vars and readiness signal. |

And on the Spring side:

| Part | Artifact | What it does |
|---|---|---|
| **Boot's Testcontainers support** | `spring-boot-testcontainers` (verify the exact artifact id for your Boot version — see below) | `@ServiceConnection`, and the ability to run the app itself against containers at development time. |

> **Version discipline (R8).** Do not pin Testcontainers versions by hand. Spring
> Boot's dependency management includes the Testcontainers BOM, so declaring the
> dependency without a `<version>` gets you the version Boot tested against. Confirm
> what you actually resolved:
> ```bash
> mvn dependency:tree -Dincludes=org.testcontainers
> ```
> I am not quoting a version number here, because any number I write is stale the day
> after it is written.

### Ryuk — the part people are confused by

When you first run a Testcontainers test you will see a second container appear with
a name containing `testcontainers-ryuk`. That is not a bug and not something you
started.

**Ryuk is the reaper.** Testcontainers labels every container it creates. Ryuk holds
a socket connection back to your JVM. If your JVM dies — you hit stop in the IDE, CI
kills the build, the process segfaults — the socket closes and Ryuk deletes every
labelled container. Without it, a killed test run would leave orphaned Postgres
containers on your machine forever.

You will meet Ryuk again in the reuse section, because reuse is precisely "opt this
container out of Ryuk".

### What "ready" means, and why it is not "started"

`docker run postgres` returns almost immediately. Postgres itself is not accepting
connections yet. If your test connects at that moment it fails with a connection
refused, intermittently, on slower machines only. This is the classic flaky
integration test.

Testcontainers solves this with **wait strategies**. `PostgreSQLContainer` ships with
one already: it waits for a specific log line to appear a specific number of times
(Postgres logs "ready to accept connections" twice — once for the temporary
initialisation server, once for the real one). You get this for free from the module.
For a `GenericContainer` you must supply it yourself.

This is the single biggest reason to prefer a module (`PostgreSQLContainer`) over a
hand-rolled `GenericContainer`: someone has already worked out the readiness signal.

---

## Why does it matter?

### 1. H2 is a different database, and the differences are exactly where your bugs are

This is the mastery line, so it gets the most space. Concretely, here is what H2 does
not model, restricted to things `orderflow` actually depends on:

| Postgres behaviour `orderflow` relies on | H2 | Where it bites |
|---|---|---|
| `SELECT ... FOR UPDATE SKIP LOCKED` | Not supported in the Postgres sense | Topic 52's inventory work-queue pattern is untestable |
| MVCC snapshot isolation for `REPEATABLE READ` | Different implementation, different anomalies | Topic 55's isolation reasoning is unverifiable |
| `INSERT ... ON CONFLICT (idempotency_key) DO NOTHING` | Different syntax (`MERGE`) | Your idempotency guard is either untested or written twice |
| `jsonb` with operators and GIN indexes | Partial `json` support only | Payment gateway payload storage |
| Sequence allocation with `allocationSize = 50` under concurrency | Different caching semantics | Topic 53's batching work |
| Constraint violation `SQLState` codes and the constraint name in the message | Different codes, different message text | Any `catch (DataIntegrityViolationException)` that inspects which constraint failed |
| Unquoted identifier case-folding to **lower**case | Folds to **upper**case by default | Native queries that work in one and not the other |
| `NULLS LAST` as the default for `ORDER BY ... DESC` | Different default | Pagination ordering silently differs |
| `generate_series` | Not present | The seeding strategy Topic 65 depends on |
| Real query plans via `EXPLAIN (ANALYZE, BUFFERS)` | Different planner entirely | Every index decision you make is unvalidated |

Read that table as a list of **production-only bugs**. Not "risks". Bugs that exist,
that your suite is green about, and that will be found by a customer.

The deeper point, and the one to say in an interview: **a test against a different
database is not a weaker test of the same thing — it is a strong test of a different
thing.** It tells you your code works against H2. Nobody is running H2.

### 2. The persistence-context behaviour from Topics 48–53 is only real against a real database

Topic 48 taught you dirty checking and flush ordering. Topic 50 taught you to assert
on **statement counts**. Topic 52 taught you optimistic locking and the oversell
reproduction. Topic 53 taught you JDBC batching.

Every one of those is a statement about what Hibernate emits and what the database
does with it. `@BatchSize` producing an `IN (?, ?, ?)` is a real behaviour. Whether
that `IN` list uses an index scan is a Postgres planner decision. Whether two
concurrent `UPDATE ... WHERE quantity_available >= ?` statements serialise correctly
is a Postgres MVCC fact.

**None of Phase 5 is verified by an H2 suite.** Testcontainers is what makes the last
sixteen topics testable rather than merely written down.

### 3. CI and local become the same thing

The compose file that a human forgets to start is gone. The environment variable that
points at a shared staging database that someone else is concurrently truncating is
gone. The test declares its own dependencies and starts them.

This matters more than it sounds. "Reproducible" is the property Topic 65's gate rule
depends on, and reproducibility starts here.

### 4. It is the prerequisite for the gate

Topic 65 needs a containerised Postgres with a realistic dataset. You will build that
stack with `docker compose` rather than Testcontainers — a load test runs against a
long-lived stack, not a per-test one — but the *image pinning*, the *migration
discipline* and the *readiness checks* are the same skills. Getting them right here
means the gate stack is a small step rather than a new project.

---

## Syntax breakdown

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

No `<version>` elements. The Boot parent POM or the `spring-boot-dependencies` BOM
manages all three. Topic 32 explains why that matters more in Maven than in npm: one
version wins on a flat classpath, so an unmanaged version is a resolution accident
waiting to happen.

> **Boot 4 modularisation caveat (R8).** Boot 4 split the codebase into more,
> smaller jars, and some artifact ids moved. `spring-boot-testcontainers` is the
> right one on Boot 3.1 through 3.5. **I am not certain the id is unchanged on Boot
> 4.1.** Do not take my word for it — check:
> ```bash
> mvn dependency:tree -Dincludes=org.springframework.boot
> ```
> and look for the artifact that contains
> `org.springframework.boot.testcontainers.service.connection.ServiceConnection`:
> ```bash
> javap -cp "$(mvn -q dependency:build-classpath -Dmdep.outputFile=/dev/stdout | tail -1)" \
>   org.springframework.boot.testcontainers.service.connection.ServiceConnection
> ```
> If `javap` prints the annotation, the artifact you have is the right one.

### `@Testcontainers` — the JUnit 5 extension

```java
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class OrderRepositoryIT { }
```

| Piece | Meaning |
|---|---|
| `@Testcontainers` | A meta-annotation for `@ExtendWith(TestcontainersExtension.class)`. It is the JUnit 5 extension (Topic 58) that finds `@Container` fields and manages their lifecycle. |
| Without it | `@Container` fields are inert. The container is never started. Your test fails with a connection error and the cause is not obvious. |
| `@Testcontainers(disabledWithoutDocker = true)` | Skip rather than fail when no Docker daemon is available. Useful for a repo where some contributors cannot run Docker. Use it deliberately, not reflexively — a silently skipped integration suite is worse than a failing one. |

### `@Container` — and the static/non-static distinction that decides your suite runtime

```java
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres =                 // STATIC
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

| Declaration | Lifecycle | Cost |
|---|---|---|
| `@Container static` field | Started once **before all tests in this class**, stopped after the last one. | One container start per test class. |
| `@Container` **instance** field | Started **before every test method**, stopped after each. | One container start per test method. This is almost always a mistake. |

**The static/non-static distinction is the single most expensive one-character
decision in a Java test suite.** A class with 30 test methods and a non-static
`@Container` starts 30 Postgres containers.

Note carefully: **even the static form is one container per test class.** Twelve
persistence test classes means twelve container starts. That is the problem the
singleton pattern below solves, and it is the mastery line for this topic.

### The container builder API

```java
static PostgreSQLContainer<?> postgres =
    new PostgreSQLContainer<>(DockerImageName.parse("postgres:16-alpine"))
        .withDatabaseName("orderflow")
        .withUsername("orderflow")
        .withPassword("orderflow")
        .withCommand("postgres", "-c", "fsync=off", "-c", "synchronous_commit=off")
        .withTmpFs(Map.of("/var/lib/postgresql/data", "rw"));
```

| Call | What it does | Why you might want it |
|---|---|---|
| `DockerImageName.parse("postgres:16-alpine")` | Pins the image. **Never use `latest`.** | Reproducibility. Match the *major* version production runs. |
| `.withDatabaseName/Username/Password` | Sets the `POSTGRES_*` env vars the official image reads. | Predictable credentials for `@DynamicPropertySource`. |
| `.withCommand("postgres", "-c", "fsync=off", ...)` | Disables durability. | This database is thrown away. Durability buys you nothing and costs disk syncs. **Never do this to a real database.** |
| `.withTmpFs(...)` | Puts the data directory in RAM. | Removes disk I/O from test time entirely. |
| `.withInitScript("db/test-init.sql")` | Runs one SQL file at startup. | Convenient, but see the migration trap — prefer Flyway. |
| `.withReuse(true)` | Opt this container out of Ryuk so it survives the JVM. | See the reuse section. |
| `.withLogConsumer(new Slf4jLogConsumer(log))` | Pipes container stdout into your test log. | Essential when a container will not become ready and you cannot see why. |
| `.waitingFor(Wait.forListeningPort())` | Custom readiness. | Only needed for `GenericContainer`; the modules set their own. |

Two accessors you will use constantly:

```java
postgres.getJdbcUrl();      // jdbc:postgresql://localhost:54321/orderflow  <- random host port
postgres.getMappedPort(5432);
postgres.getHost();
```

**The port is random every run.** Docker maps the container's 5432 to a free
ephemeral port on your host. That is why you cannot put a JDBC URL in
`application-test.yml` — you do not know the port until the container has started.
Which is exactly the problem the next two annotations solve.

### `@ServiceConnection` — the modern wiring, and the one to use

```java
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;

@SpringBootTest
@Testcontainers
class OrderPlacementIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
}
```

That is the whole configuration. No URL, no username, no password, no property names.

**What it actually does.** Boot registers a `ConnectionDetails` bean derived from the
container. `JdbcConnectionDetails` supplies the URL, user and password to the
`DataSource` auto-configuration, which prefers a `ConnectionDetails` bean over
`spring.datasource.*` properties. There is a `ConnectionDetails` type per supported
service — JDBC, R2DBC, Kafka, Redis, MongoDB, Elasticsearch and others.

| Container type | What `@ServiceConnection` needs |
|---|---|
| `PostgreSQLContainer` | Nothing. Recognised by type. |
| `KafkaContainer` | Nothing. Recognised by type. |
| `GenericContainer` (e.g. Redis) | The image name cannot be inferred, so you say it: `@ServiceConnection(name = "redis")`. The `name` is the *connection name* Boot matches against, not an arbitrary label. |

Redis in practice:

```java
@Container
@ServiceConnection(name = "redis")
static GenericContainer<?> redis =
    new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
        .withExposedPorts(6379);
```

> Redis has no first-party Testcontainers module in core the way Postgres does. You
> either use `GenericContainer` as above, or a community module. `GenericContainer`
> plus `@ServiceConnection(name = "redis")` is fewer moving parts and is what I would
> write.

### `@DynamicPropertySource` — the older mechanism, still needed sometimes

```java
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

@SpringBootTest
@Testcontainers
class OrderPlacementIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void datasourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

| Piece | Meaning |
|---|---|
| `static void` method | Must be static. Spring calls it before the context is created; no instance exists yet. |
| `DynamicPropertyRegistry` parameter | The only parameter allowed. |
| `registry.add(key, Supplier)` | Note it takes a **`Supplier`**, not a value. `postgres::getJdbcUrl` is a method reference (Topic 22), evaluated lazily. |
| Ordering | Runs **after** static `@Container` fields have started, **before** the `ApplicationContext` is refreshed. That ordering is the entire reason this works. |

**When you still need `@DynamicPropertySource` rather than `@ServiceConnection`:**

- The property is not part of a `ConnectionDetails` contract — for example, your own
  `orderflow.payment.gateway.base-url` pointing at a WireMock or MockServer container.
- A service Boot has no `ConnectionDetails` support for.
- You need to derive a value rather than pass it through.

Use `@ServiceConnection` for anything it covers, `@DynamicPropertySource` for the
rest. They coexist happily.

> **Boot 3.4+ / Boot 4 convenience.** `DynamicPropertyRegistry` can be injected as a
> parameter into an `@Bean` method, which lets a `@TestConfiguration` own both the
> container and its property wiring. Check whether your version supports it before
> using it; the `@DynamicPropertySource` form above works everywhere.

### The singleton container pattern — the thing the mastery line is about

The problem, restated: `@Container static` gives you one container **per test class**.
Ten persistence test classes, ten container starts.

The fix is to take the container out of JUnit's lifecycle entirely and put it in a
static initialiser on a shared base class. A static initialiser runs **once per class
loader** — which for a Maven Surefire fork is once per JVM, which is once per test
run.

```java
package com.orderflow.support;

import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

/**
 * Base class for every orderflow test that needs a database.
 *
 * There is NO @Testcontainers and NO @Container here, deliberately.
 * The container is started by the static initialiser below, exactly once
 * per JVM, and is never stopped -- Ryuk removes it when the JVM exits.
 */
public abstract class PostgresTestBase {

    protected static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>(DockerImageName.parse("postgres:16-alpine"))
            .withDatabaseName("orderflow")
            .withUsername("orderflow")
            .withPassword("orderflow")
            .withCommand("postgres", "-c", "fsync=off", "-c", "synchronous_commit=off")
            .withReuse(true);

    static {
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void postgresProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

Line by line, the parts that are load-bearing:

| Line | Why it is written that way |
|---|---|
| No `@Testcontainers` | The JUnit extension would take ownership of the lifecycle and stop the container after the class. We do not want that. |
| No `@Container` | Same reason. The field is a plain static field. |
| `static { POSTGRES.start(); }` | Runs on first class initialisation (Topic 67) — that is, when the first test class extending this base is loaded. Once per JVM. |
| No `stop()` anywhere | Intentional. Ryuk removes the container when the JVM exits. Calling `stop()` in a shutdown hook is redundant and can race. |
| `@DynamicPropertySource` on the **base** class | Inherited by every subclass, so every test class registers the *same* customizer. This matters for the context cache — see below. |

Then every persistence test extends it:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryIT extends PostgresTestBase { }
```

### Why the singleton pattern also protects the Topic 60 context cache

This connection is worth understanding properly, because it is where two topics meet.

Topic 60 taught you that the Spring test context is cached under a
`MergedContextConfiguration` key, and that **context customizers are part of that
key**. `@DynamicPropertySource` registers a customizer. So does `@ServiceConnection`.

The question that follows: does using containers fork your context cache?

**With the singleton base class: no.** Every test class inherits the *same*
`@DynamicPropertySource` method from the same base class, so every test class
contributes an equal customizer, so the cache key is unaffected and contexts are
shared normally.

**With a per-class container and a per-class `@DynamicPropertySource`: it depends on
how the customizer implements equality**, and you should not be relying on that. You
now have N containers and possibly N contexts, and your suite runtime is N full
Spring startups plus N Postgres startups.

So the singleton pattern buys you two things at once: one container instead of N, and
a context cache that behaves the way Topic 60 described. That is why it is the
mastery line.

You verify the second half exactly the way Topic 60 taught:

```properties
logging.level.org.springframework.test.context.cache=DEBUG
```

### Container reuse — `~/.testcontainers.properties`

Reuse makes the container survive **between JVM runs**. The second `mvn test` finds
the already-running container and attaches to it instead of starting a new one.

It requires **two** things, and people usually do only one:

**1. Opt in globally, on your machine:**

```bash
echo 'testcontainers.reuse.enable=true' >> ~/.testcontainers.properties
cat ~/.testcontainers.properties
```

**2. Opt in per container, in code:**

```java
new PostgreSQLContainer<>("postgres:16-alpine")
    .withReuse(true);
```

If either is missing, reuse silently does not happen. There is no error. You just
keep paying startup, which is why "I enabled reuse and nothing changed" is a common
complaint.

| Property of reuse | Consequence |
|---|---|
| Ryuk does not label the container for reaping | It survives your JVM exiting. It also survives you forgetting about it. `docker ps` is how you find it. |
| Reuse matches on a **hash of the container configuration** | Change the image tag, the command, or an env var and you get a *new* container. The old one keeps running. |
| **Data persists between runs** | This is the trap. Your tests must clean their own data. If test A leaves rows behind, test A in the next run sees them. |
| Deliberately **not** enabled in CI | `~/.testcontainers.properties` is a per-developer file and is not committed. CI gets fresh containers, which is correct — CI should not inherit anyone's state. |

**The rule: reuse is a local developer convenience, and it is only safe if your data
cleanup is already correct.** If enabling reuse breaks your tests, reuse did not
break them; it revealed that they depended on a fresh database. Fix the cleanup.

### Kafka and Redis, briefly

```java
// Kafka. Used from Topic 113 onward.
@Container
@ServiceConnection
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("apache/kafka:3.8.0"));
```

> **An honest uncertainty (R8).** Testcontainers has two Kafka container classes: an
> older `org.testcontainers.containers.KafkaContainer` built around the
> `confluentinc/cp-kafka` image, and a newer `org.testcontainers.kafka.KafkaContainer`
> built around the official `apache/kafka` KRaft image. Which one your resolved
> version ships, and which is deprecated, changes between releases. **Check rather
> than trust me:**
> ```bash
> unzip -l "$(find ~/.m2/repository/org/testcontainers/kafka -name 'kafka-*.jar' | head -1)" | grep -i kafkacontainer
> ```
> Both work with `@ServiceConnection`. Pick the one your version documents, and pin
> the image tag to the Kafka version your production cluster runs.

```java
// Redis. No core module; GenericContainer is fine.
@Container
@ServiceConnection(name = "redis")
static GenericContainer<?> redis =
    new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
        .withExposedPorts(6379);
```

---

## Example 1 — minimal

The smallest thing that proves the machinery works: a real Postgres, a real
`JdbcTemplate`, one query. No Spring Boot test slice, no entities.

```java
package com.orderflow.support;

import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
class PostgresIsRealIT {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Test
    void connectsToARealPostgresAndReportsItsVersion() throws Exception {
        try (Connection c = DriverManager.getConnection(
                     postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
             Statement s = c.createStatement();
             ResultSet rs = s.executeQuery("select version()")) {

            rs.next();
            String version = rs.getString(1);
            System.out.println("Connected to: " + version);

            assertThat(version).startsWith("PostgreSQL 16");
        }
    }

    @Test
    void supportsSkipLocked() throws Exception {
        // H2 cannot run this statement. That is the point of the test.
        try (Connection c = DriverManager.getConnection(
                     postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
             Statement s = c.createStatement()) {

            s.execute("create table pending_payment (id bigserial primary key, status text)");
            s.execute("insert into pending_payment (status) values ('NEW'), ('NEW')");

            try (ResultSet rs = s.executeQuery(
                    "select id from pending_payment where status = 'NEW' "
                    + "order by id limit 1 for update skip locked")) {
                assertThat(rs.next()).isTrue();
            }
        }
    }
}
```

Run it:

```bash
mvn -Dtest=PostgresIsRealIT test
```

Two things to notice in your own output:

1. A line printed by Testcontainers telling you the container started and how long it
   took. That is your real cost number. Write it down.
2. The `version()` string. It is a real Postgres banner because it is a real Postgres.

The second test is the argument in miniature. `for update skip locked` is a statement
your inventory work-queue depends on and that an H2 suite cannot execute. Delete this
test and swap in H2 and you have a green suite that proves nothing about the feature.

---

## Example 2 — production scenario (on the project spine)

Now the real thing: the `orderflow` persistence test fixture, built the way it should
be built, with the singleton container, real Flyway migrations, and explicit data
cleanup.

### Step 1 — the shared base class

```java
package com.orderflow.support;

import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

/**
 * One Postgres for the entire test JVM.
 *
 * Every orderflow test that touches the database extends this class.
 * That single fact is what keeps the suite in seconds rather than minutes:
 * one container start, and one Spring context configuration.
 */
public abstract class PostgresTestBase {

    /**
     * Pin the MAJOR version to whatever production runs. Never :latest.
     * -alpine is smaller to pull; the Postgres inside is the same Postgres.
     */
    private static final DockerImageName IMAGE = DockerImageName.parse("postgres:16-alpine");

    protected static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>(IMAGE)
            .withDatabaseName("orderflow")
            .withUsername("orderflow")
            .withPassword("orderflow")
            // Durability is pointless for a database we throw away, and it costs fsyncs.
            // NEVER set these on a real database.
            .withCommand("postgres",
                         "-c", "fsync=off",
                         "-c", "synchronous_commit=off",
                         "-c", "full_page_writes=off")
            .withReuse(true);

    static {
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void datasourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);

        // Hibernate must NOT create the schema. Flyway owns it. See Trap 4.
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
        registry.add("spring.flyway.enabled",         () -> "true");
    }
}
```

### Step 2 — deterministic cleanup between tests

Reuse means data survives. A rolled-back `@Transactional` test does not flush the way
production flushes. You need a cleaner that is explicit about both.

```java
package com.orderflow.support;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Component
public class DatabaseCleaner {

    private final JdbcTemplate jdbc;

    DatabaseCleaner(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    /**
     * TRUNCATE ... RESTART IDENTITY CASCADE in one statement.
     *
     * One statement matters: truncating tables one at a time in the wrong
     * order fails on foreign keys, and CASCADE on a single table would
     * quietly truncate tables you did not name.
     *
     * flyway_schema_history is excluded -- deleting it would make Flyway
     * re-run every migration on the next context.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void clean() {
        List<String> tables = jdbc.queryForList("""
                select tablename
                  from pg_tables
                 where schemaname = 'public'
                   and tablename <> 'flyway_schema_history'
                """, String.class);

        if (tables.isEmpty()) return;

        jdbc.execute("truncate table " + String.join(", ", tables)
                     + " restart identity cascade");
    }
}
```

```java
package com.orderflow.support;

import org.junit.jupiter.api.AfterEach;
import org.springframework.beans.factory.annotation.Autowired;

public abstract class DatabaseIntegrationTestBase extends PostgresTestBase {

    @Autowired protected DatabaseCleaner cleaner;

    @AfterEach
    void wipe() {
        cleaner.clean();
    }
}
```

> **Why `@AfterEach` truncation rather than `@Transactional` rollback?**
> Rollback is faster and is the right default for many tests. But a rolled-back test
> never commits, so it cannot observe anything that only happens at commit: deferred
> constraints, commit-time triggers, and — critically for Topic 52 — the behaviour of
> a *second* concurrent transaction. Any test that spawns a second thread or a second
> transaction **must not** be wrapped in a rollback-only test transaction, because
> the second transaction cannot see the first one's uncommitted writes.
> Truncation is the cleanup that works for both kinds of test. Use it as the default
> for `orderflow`, and reach for `@Transactional` rollback only in single-transaction
> repository tests where you have measured that it matters.

### Step 3 — the N+1 assertion from Topic 50, now against real Postgres

```java
package com.orderflow.orders;

import com.orderflow.support.DatabaseIntegrationTestBase;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.domain.PageRequest;

import jakarta.persistence.EntityManagerFactory;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class OrderQueryStatementCountIT extends DatabaseIntegrationTestBase {

    @Autowired OrderQueryService orderQueries;
    @Autowired OrderTestDataSeeder seeder;
    @Autowired EntityManagerFactory emf;

    private Statistics statistics;

    @BeforeEach
    void resetStatistics() {
        statistics = emf.unwrap(SessionFactory.class).getStatistics();
        statistics.setStatisticsEnabled(true);
        statistics.clear();
    }

    @Test
    @DisplayName("GET /orders page of 50 executes a bounded number of statements")
    void listOrdersDoesNotNPlusOne() {
        // 100 orders, 5 lines each, drawn from 40 products -- Topic 50's fixture.
        seeder.seed(100, 5, 40);
        statistics.clear();

        var page = orderQueries.listOrders(PageRequest.of(0, 50));

        assertThat(page.getContent()).hasSize(50);

        // The assertion that cannot regress silently.
        // Three queries: id page, header projection, line projection.
        assertThat(statistics.getPrepareStatementCount())
                .as("statement count for one page of orders")
                .isLessThanOrEqualTo(4);
    }
}
```

**Why this test is only meaningful against Postgres.** The statement *count* would be
the same on H2 — Hibernate emits what Hibernate emits. What would not be the same is
everything downstream of the count: whether the `IN (?, ?, ?)` list uses the
`order_line(order_id)` index, whether the sequence allocation batches, whether the
`ORDER BY placed_at DESC` produces the ordering the DTO assumes. Topic 50 told you
the count is the regression guard. Testcontainers is what makes the rest of the
behaviour real.

### Step 4 — the oversell test from Topic 52, which genuinely cannot run on H2

```java
package com.orderflow.inventory;

import com.orderflow.support.DatabaseIntegrationTestBase;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class ConcurrentInventoryDecrementIT extends DatabaseIntegrationTestBase {

    @Autowired InventoryService inventory;
    @Autowired InventoryTestFixtures fixtures;

    @Test
    @DisplayName("two concurrent placements against one unit of stock never oversell")
    void neverOversells() throws Exception {
        Long productId = fixtures.productWithStock(1);   // exactly one unit

        int threads = 2;
        var start = new CountDownLatch(1);
        var succeeded = new AtomicInteger();

        try (ExecutorService pool = Executors.newFixedThreadPool(threads)) {
            List<Future<?>> futures = java.util.stream.IntStream.range(0, threads)
                .mapToObj(i -> pool.submit(() -> {
                    start.await();
                    try {
                        inventory.reserve(productId, 1);
                        succeeded.incrementAndGet();
                    } catch (InsufficientStockException expected) {
                        // one of the two must land here
                    }
                    return null;
                }))
                .toList();

            start.countDown();
            for (Future<?> f : futures) f.get();
        }

        assertThat(succeeded.get())
                .as("exactly one reservation may succeed")
                .isEqualTo(1);
        assertThat(fixtures.availableStock(productId))
                .as("stock must never go negative")
                .isZero();
    }
}
```

**This is the test that makes the whole topic concrete.** It uses two real threads,
two real transactions, and a real database. Its correctness depends entirely on the
database's MVCC behaviour for `UPDATE inventory SET quantity_available =
quantity_available - ? WHERE product_id = ? AND quantity_available >= ?`.

Postgres serialises the two updates on the row lock and the second one matches zero
rows. That is the guarantee `orderflow` sells on. It is a **Postgres** guarantee. A
green version of this test against H2 would tell you nothing about whether Postgres
does the same thing, and — worse — a *failing* version against H2 would send you
hunting for a bug in code that is correct.

Note also that this test cannot use `@Transactional` rollback cleanup. Two
transactions, so the test itself must not hold one. That is why
`DatabaseIntegrationTestBase` truncates in `@AfterEach` instead.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — H2 in tests, Postgres in production

**Wrong:**

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:orderflow;MODE=PostgreSQL
  jpa:
    hibernate:
      ddl-auto: create-drop
```

The `MODE=PostgreSQL` is the part that makes people comfortable. It should not.
It is a compatibility *mode*, not a compatibility *guarantee*: it changes some
syntax handling and some function names. It does not change the storage engine, the
locking implementation, the planner, the isolation implementation, the error codes,
or the feature set.

**Exact symptom — and there are three distinct shapes, so learn all three:**

*Shape A — the green suite and the production incident.* Every test passes. Then in
production: a `PSQLException` with `ERROR: syntax error at or near "MERGE"`, or a
duplicate key on a constraint your test suite proved was enforced, or an oversell
that your concurrency test said was impossible.

*Shape B — the test that fails for the wrong reason.* You write the `SKIP LOCKED`
work-queue from Topic 52. The test fails with `Syntax error in SQL statement`. You
spend an afternoon assuming your JPQL is wrong. It is not; H2 simply does not have
the feature. You then rewrite correct production code to satisfy a database nobody
runs.

*Shape C — the silent semantic drift.* `ORDER BY placed_at DESC` returns rows with
`NULL` in a different position. Your pagination test asserts on the first row's id.
It passes on H2 and returns a different row on Postgres. Nobody notices until a
customer reports a missing order on page 1.

**Root cause:** H2 and Postgres are different programs. A test is an experiment, and
the experiment you ran used a different apparatus than the one in production. The
`MODE=PostgreSQL` flag makes the apparatus *look* similar, which makes the mistake
harder to see, which makes it worse rather than better.

**Fix:** delete the H2 dependency. Not "stop using it in new tests" — delete it, so
it cannot be reached by accident. Then:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryIT extends PostgresTestBase { }
```

The `replace = NONE` is not optional and is the detail people miss. Topic 60 covered
it: `@DataJpaTest` **replaces your `DataSource` with an embedded one by default**. If
H2 is on the classpath it will silently be used even though you configured
Testcontainers. That is how a team ends up with a Testcontainers setup that never
runs. Removing the H2 dependency makes the mistake impossible rather than merely
unlikely.

**The one legitimate use of H2 that remains:** a test that has nothing to do with SQL
and merely needs *a* database to exist so a context can start. Even then, a shared
singleton Postgres costs you nothing extra, so there is rarely a reason.

---

### Trap 2 — one container per test class instead of a shared singleton

**Wrong:**

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    // ... and the identical five lines copy-pasted into eleven other test classes
}
```

**Exact symptom:** the suite takes minutes and the time is not in your tests. Two
observations pin it down precisely:

1. Run `docker ps` in a second terminal **while the suite is running**, repeatedly.
   You see Postgres containers appearing and disappearing, one at a time, in step
   with the test classes.
2. In the build log, the Testcontainers "container started" line appears once per
   test class rather than once per run. Count them.

The log line has this shape — *illustration of the format, not captured output*:

```
INFO  o.t.c.PostgreSQLContainer - Container postgres:16-alpine started in PT<n>S
```

If you count `<n>` occurrences and get twelve, you have twelve container starts.

**Root cause:** `@Container` binds the container to JUnit's *class* lifecycle. Static
means "once per class", not "once per run". People read "static" and assume "global".

Frequently compounded: the copy-pasted `@DynamicPropertySource` in each class may fork
the Topic 60 context cache too, so you also pay N Spring startups. Now the suite is
N × (container start + context start).

**Fix:** the `PostgresTestBase` singleton from Example 2. One static initialiser, no
`@Testcontainers`, no `@Container`, one `@DynamicPropertySource` on the base class,
every test class extends it.

**Proof that the fix worked** — run both and compare your own two numbers:

```bash
logging.level.org.springframework.test.context.cache=DEBUG   # put this in src/test/resources/application.properties
mvn test 2>&1 | grep -c "started in PT"     # container starts. Should be 1.
mvn test 2>&1 | grep "Spring test ContextCache" | tail -1    # contexts. Should be small.
```

| What you see | What it means |
|---|---|
| `1` container start, and a context cache `size` of 1 or 2 | Correct. This is the target state. |
| N container starts matching your test-class count | The singleton is not being used. Some test class still declares its own `@Container`. |
| 1 container start but a context cache `size` climbing per class | Containers are fine; the context cache is forking for a different reason. That is Topic 60's problem — a `@MockitoBean` or a property override somewhere. |
| 0 container starts on a re-run | Reuse is working. |

---

### Trap 3 — a non-static `@Container`

**Wrong:**

```java
@Testcontainers
class WalletDebitIT {

    @Container
    PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    //  ^^ no 'static'

    @Test void debitsBalance() { }
    @Test void rejectsInsufficientFunds() { }
    @Test void isIdempotentOnRetry() { }
    // ... 27 more
}
```

**Exact symptom:** this one class alone takes longer than the rest of the suite
combined. `docker ps` during the run shows a container churning — created and
destroyed once per test method. On a laptop with limited Docker resources you may
also see failures under `docker stats` memory pressure, or intermittent "connection
refused" as the machine struggles to start containers fast enough.

`docker events` makes it unambiguous:

```bash
docker events --filter 'type=container' --filter 'event=start' --format '{{.Time}} {{.Actor.Attributes.image}}'
```

Count the `postgres` lines during one run of one test class.

**Root cause:** `@Container` on an **instance** field is bound to JUnit's *method*
lifecycle, because JUnit creates a new test instance per test method by default —
which is exactly the `@TestInstance` behaviour Topic 58 covered. One instance per
method means one container per method.

**Fix:** in the singleton pattern this cannot happen, because there is no `@Container`
at all. If you are not using the singleton pattern, add `static` and move on. Then
add a check to stop it recurring:

```bash
# Any non-static @Container in the repo is a defect. Grep for it in CI.
grep -rn -B1 '@Container' src/test/java | grep -v 'static' | grep -A1 '@Container'
```

---

### Trap 4 — `ddl-auto: create-drop` against the container instead of running your real migrations

**Wrong:**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop     # Hibernate builds the schema from the entities
  flyway:
    enabled: false
```

This looks harmless. You have a real Postgres now, so surely the schema is fine?

**Exact symptom:** the suite is green, and then a deployment fails, or worse
succeeds and behaves differently. Three concrete shapes:

- A Flyway migration fails in staging with `ERROR: column "reserved_quantity" of
  relation "inventory" already exists` — because the migration was never executed
  by any test and nobody noticed it was wrong.
- A query that is fast in tests is slow in production, because Hibernate's generated
  schema created different indexes (or none) than your migration scripts do.
- A `CHECK (quantity_available >= 0)` constraint that your migration adds — and that
  Topic 52 relies on as the last line of defence against oversell — does not exist in
  tests, so the test that "proves" the constraint fires proves nothing.

**Root cause:** you are testing against a schema **Hibernate invented from your
entity annotations**, not the schema your deployment actually produces. Those are two
different artefacts that happen to be similar. The migrations — the thing that will
run in production — are never exercised.

**Fix:** Flyway owns the schema. Hibernate validates it and creates nothing.

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate       # fail startup if entities and schema disagree
  flyway:
    enabled: true
    locations: classpath:db/migration     # the SAME scripts production runs
```

`validate` is doing real work here: it turns "an entity field has no matching column"
from a runtime `SQLGrammarException` under load into a **startup failure in the test
suite**, which is where you want to find it.

**Proof:**

```bash
psql "$(your container jdbc url)" -c "select version, description, success from flyway_schema_history order by installed_rank"
```

If that table is empty or missing, Flyway did not run and you are testing an invented
schema.

---

### Trap 5 — tests that depend on data from other tests

**Wrong:**

```java
@Test @Order(1)
void seedsTheCatalogue() {
    productRepository.save(new Product("SKU-4471", "Wireless keyboard", 4999L));
}

@Test @Order(2)
void findsTheProductBySku() {
    assertThat(productRepository.findBySku("SKU-4471")).isPresent();
}
```

**Exact symptom:** this is the classic flaky-test shape, and it has a very specific
tell. The suite passes when run whole and **fails when you run one test alone from
your IDE**, with an assertion failure rather than an error. It may also start failing
after an unrelated change, because JUnit's default method ordering is deterministic
but not alphabetical or declaration-based, and a class rename can change it.

With container **reuse** enabled it gets stranger: the second `mvn test` on your
machine passes tests that fail on a colleague's fresh machine and in CI, because your
reused container still holds last run's rows.

The definitive diagnosis:

```bash
# Passes as part of the suite, fails alone => shared state.
mvn -Dtest=ProductRepositoryIT#findsTheProductBySku test
```

**Root cause:** the database is shared across the whole run — that is the entire point
of the singleton — so it is *mutable global state*. `@Order` makes the coupling
explicit but does not make it correct; it just makes the failure appear later, when
someone inserts a test in the middle.

Reuse is not the cause. Reuse is the thing that *exposed* the cause.

**Fix:** every test creates the data it needs and the suite truncates after every
test. The `DatabaseCleaner` from Example 2, plus fixtures:

```java
@Test
void findsTheProductBySku() {
    Product saved = fixtures.product("SKU-4471", "Wireless keyboard", 4999L);

    assertThat(productRepository.findBySku("SKU-4471"))
            .isPresent()
            .get()
            .extracting(Product::getId)
            .isEqualTo(saved.getId());
}
```

Then prove independence, which is the only assertion that matters here:

```bash
mvn test -Dsurefire.runOrder=random     # different order every run
mvn test -Dsurefire.runOrder=reversealphabetical
```

| What you see | What it means |
|---|---|
| Green under both, and green for any single test run alone | Tests are independent. This is the bar. |
| Green in declaration order, red under `random` | Order dependence confirmed. Find it with the single-test run. |
| Red on the first run after enabling reuse, green on a fresh container | The suite depends on an empty database it never emptied itself. Add cleanup. |

---

## Hands-on proof

Everything here is a command you run. I have no Docker daemon and no JVM, so I will
not print output and call it real. What follows is the exact command, what to look
for, and what each possible result means.

### Setup

```bash
docker --version
docker info | grep -i 'server version'
docker ps
```

| What you see | What it means |
|---|---|
| A server version and an empty or short container list | Docker is up. Proceed. |
| `Cannot connect to the Docker daemon` | Testcontainers will fail with `IllegalStateException: Could not find a valid Docker environment`. Start Docker Desktop / Colima / your daemon. |
| Docker running but with a small memory allocation | Container startup may be slow or flaky. Check the daemon's resource settings before blaming Testcontainers. |

Pre-pull the image so the first test run does not include a download:

```bash
docker pull postgres:16-alpine
docker image inspect postgres:16-alpine --format '{{.Id}} {{.Created}}'
```

Record that image ID. It is part of what makes a run reproducible, and Topic 65 will
ask you for it again.

### Proof 1 — the container is a real Postgres and the JDBC port is random

Run `PostgresIsRealIT` from Example 1, and **while it is running**, in a second
terminal:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
```

**What to look for:**

| What you see | What it means |
|---|---|
| A container from `postgres:16-alpine` with a mapping like `0.0.0.0:<n>->5432/tcp` | Correct. `<n>` is the random host port `getMappedPort(5432)` returned. |
| A second container named `testcontainers-ryuk-...` | Correct and expected. That is the reaper. |
| The port `<n>` differs between two runs | Correct, and it is the reason `application-test.yml` cannot hold a static URL. |
| No containers at all while tests run | `@Testcontainers` is missing, or the singleton's static initialiser never ran because nothing extended the base class. |

### Proof 2 — measure your own container startup cost

```bash
mvn -Dtest=PostgresIsRealIT test 2>&1 | grep -i "started in"
```

The line has this shape — *illustration of the format, not captured output*:

```
INFO  o.t.c.PostgreSQLContainer - Container postgres:16-alpine started in PT<n>S
```

Fill this in for yourself:

| Measurement | Your value |
|---|---|
| Cold start, image not pulled | |
| Cold start, image already pulled | |
| Warm start, second run in the same session | |
| With `testcontainers.reuse.enable=true`, second run | |

**How to read it:** the row that matters for the "seconds not minutes" claim is the
second one. If it is large, look at your Docker daemon's CPU and memory allocation
before concluding the tool is slow.

### Proof 3 — count container starts across the whole suite

```bash
mvn test 2>&1 | tee suite.log
grep -c "started in PT" suite.log
```

| What you see | What it means |
|---|---|
| `1` | The singleton pattern is working. This is the target. |
| A number equal to your count of database test classes | Per-class containers. Trap 2. |
| A number larger than your test-class count | Per-method containers somewhere. Trap 3. Find it with the `grep` in Trap 3's fix. |
| `0`, and tests pass | Either reuse attached to an existing container (check `docker ps`), or — much worse — the tests are silently using H2. Check `spring.datasource.url` in the log. |

That last row is important enough to have its own check:

```bash
grep -i "jdbc:" suite.log | sort -u
```

If you see `jdbc:h2:` anywhere, something is replacing your datasource. Almost always
`@DataJpaTest` without `replace = NONE`, with H2 still on the classpath.

### Proof 4 — the context cache is not being forked by the containers

Put this in `src/test/resources/application.properties`:

```properties
logging.level.org.springframework.test.context.cache=DEBUG
```

```bash
mvn test 2>&1 | grep "Spring test ContextCache" | tail -5
```

The line has this shape — *illustration of the format, not captured output*:

```
DEBUG o.s.t.c.c.DefaultContextCache - Spring test ContextCache statistics: [size = <n>, maxSize = 32, parentContextCount = 0, hitCount = <n>, missCount = <n>]
```

| What you see | What it means |
|---|---|
| `size` small (1–3) and a high `hitCount` relative to `missCount` | Contexts are shared. The container wiring is not forking the cache. |
| `size` incrementing once per database test class | Each class has its own `@DynamicPropertySource` or `@ServiceConnection`. Move it to the shared base class. |
| `size` approaching `maxSize = 32` | Contexts are being **evicted and rebuilt**. Suite time is now unbounded. This is Topic 60's failure mode and it is worth fixing before anything else. |

### Proof 5 — reuse actually reuses

```bash
echo 'testcontainers.reuse.enable=true' >> ~/.testcontainers.properties
cat ~/.testcontainers.properties

mvn test                       # run 1: starts a container
docker ps                      # note the container ID and the mapped port
mvn test                       # run 2
docker ps                      # SAME container ID, SAME port
```

| What you see | What it means |
|---|---|
| Same container ID and port across runs; run 2 logs no "started in PT" | Reuse is working. |
| A new container each run | One of the two opt-ins is missing. Check both `~/.testcontainers.properties` **and** `.withReuse(true)` in code. |
| Reuse works, but tests that passed on run 1 fail on run 2 | Your cleanup is incomplete. Trap 5. Reuse revealed it; it did not cause it. |
| A growing pile of stopped Postgres containers | Reuse containers are not reaped. Clean up with the command below. |

```bash
# Reuse opts out of Ryuk, so you clean up by hand when you are done.
docker ps -a --filter 'label=org.testcontainers=true' --format '{{.ID}} {{.Image}} {{.Status}}'
docker rm -f $(docker ps -aq --filter 'label=org.testcontainers=true')
```

### Proof 6 — the migrations really ran

```bash
# While a test with a debugger breakpoint is paused, or against a reused container:
docker exec -it <container-name> psql -U orderflow -d orderflow \
  -c '\timing on' \
  -c 'select installed_rank, version, description, success from flyway_schema_history order by installed_rank'
```

| What you see | What it means |
|---|---|
| Your migration files, all `success = t` | Flyway owns the schema. Trap 4 avoided. |
| `relation "flyway_schema_history" does not exist` | Flyway did not run. `ddl-auto` built the schema instead. Fix per Trap 4. |
| Migrations present but fewer than your `db/migration` folder | Flyway `locations` is pointing somewhere else, or a migration is being filtered out. |

While you are in there, confirm the schema is the one you expect:

```sql
\timing on
\d+ inventory
select count(*) from information_schema.table_constraints
 where table_name = 'inventory' and constraint_type = 'CHECK';
```

### Proof 7 — prove the H2 difference to yourself, once

This is worth doing once so the argument is yours and not mine. Write the same test
twice, once against H2 and once against the container:

```java
@Test
void skipLockedIsSupported() throws Exception {
    try (Connection c = dataSource().getConnection();
         Statement s = c.createStatement()) {
        s.execute("create table pending_payment (id bigint primary key, status varchar(20))");
        s.execute("insert into pending_payment values (1, 'NEW')");
        s.executeQuery("select id from pending_payment where status='NEW' "
                       + "order by id limit 1 for update skip locked");
    }
}
```

| Against | What you see | What it means |
|---|---|---|
| Testcontainers Postgres | Passes | The feature exists. Your work-queue is testable. |
| H2 with `MODE=PostgreSQL` | A syntax or feature error, or silently different locking | The compatibility mode did not give you the feature. This is the whole argument in one line of output. |

Do the same for `insert ... on conflict do nothing` and for the default `NULLS`
ordering on a `DESC` sort. Three experiments is enough to make you never argue for H2
again.

---

## Practice exercises

Write real files, run them, and keep your numbers.

### 1 — Easy: convert one test class and measure both ends

Take one existing `orderflow` persistence test class that currently runs against an
in-memory database.

**Part A.** Record the current state before touching anything:

| Measurement | Before |
|---|---|
| Wall-clock time for that class alone | |
| `grep -c "started in PT"` for that class | |
| The `spring.datasource.url` actually used (from the log) | |

**Part B.** Convert it to a `@Container static PostgreSQLContainer` — deliberately
the per-class form, not the singleton yet. Add
`@AutoConfigureTestDatabase(replace = NONE)` if it is a `@DataJpaTest`.

**Part C.** Record the same three measurements after. Then answer in a sentence:
where did the extra time go, and is it per-test or per-class?

**Part D.** Remove the H2 dependency from the POM entirely and re-run. If anything
breaks, that thing was silently using H2 and you have just found a test that was
proving nothing.

### 2 — Medium: the singleton conversion, with the context cache as evidence

Convert **every** `orderflow` database test to the singleton `PostgresTestBase`.

**Part A.** Before: turn on
`logging.level.org.springframework.test.context.cache=DEBUG` and record:

| Measurement | Before | After |
|---|---|---|
| Total suite wall-clock time | | |
| Container starts (`grep -c "started in PT"`) | | |
| Context cache final `size` | | |
| Context cache `hitCount` / `missCount` | | |

**Part B.** Do the conversion. The rules: no `@Testcontainers` and no `@Container`
anywhere; exactly one `@DynamicPropertySource`, on the base class; every database test
extends it.

**Part C.** Fill in the "After" column. Then answer honestly: **how much of your
saving came from fewer containers, and how much from fewer Spring contexts?** They
are different causes and the numbers above let you separate them. If the context
count did not change, say so — that is a real finding, and it means your suite's cost
was containers alone.

**Part D.** Now enable reuse (`~/.testcontainers.properties` plus `.withReuse(true)`),
run twice, and add a third column. Then deliberately break it: comment out the
`@AfterEach` truncation, run twice, and describe the exact failure. That failure is
the price of reuse and you should be able to recognise it instantly.

### 3 — Hard: make a production-only bug reproducible

The goal is to build a test that **fails on H2 and passes on Postgres**, or vice
versa, for a real `orderflow` behaviour. This is the exercise that converts the
argument from something you read into something you have seen.

**Part A — pick the behaviour.** Choose one:

- The Topic 52 concurrent inventory decrement (two threads, one unit of stock).
- An idempotent order insert using `insert into orders (...) values (...) on conflict
  (idempotency_key) do nothing`, asserting that a retried request creates no second
  row.
- A `SELECT ... FOR UPDATE SKIP LOCKED` payment work-queue where two workers must
  never pick the same row.

**Part B — write it once, run it twice.** Parameterise the test over two datasources
so the *same* test body runs against both. A JUnit 5 `@ParameterizedTest` with a
`@MethodSource` supplying two `DataSource`s is the clean way (Topic 58).

**Part C — record what happened, precisely.** Not "H2 failed" — the exact exception
class, the exact message, and the exact line.

| Behaviour under test | Postgres result | H2 result | Difference is: syntax / semantics / concurrency |
|---|---|---|---|
| | | | |

**Part D — the judgment question.** Suppose a colleague argues: "Our CI has no Docker
daemon, so we must use H2." Write the three-paragraph reply you would actually send.
It must include: what you would do instead (there are at least two real options), what
you would give up if you genuinely could not run Docker in CI, and what you would put
in place to bound the risk in the meantime. "Just use Testcontainers" is not an
answer to a constraint.

**Part E — connect it forward.** Your singleton container starts one Postgres with an
empty schema. Topic 65 needs a Postgres with 100,000 products, 1,000,000 orders and
5,000,000 order lines. Write down, in half a page, why you would **not** use
Testcontainers for that, and what you would use instead. (You will check your answer
against Topic 65.)

---

## Interview questions

### Q1 — "Why not just use H2? It's faster."

**Mid-level answer:** "H2 is in-memory so it's faster, but it's not exactly the same
as Postgres, so there can be differences. We use `MODE=PostgreSQL` to make it
compatible."

**Senior answer:** "Because H2 is a different database, so an H2 test is a strong test
of the wrong thing. Concretely, for the service I work on: no `SELECT ... FOR UPDATE
SKIP LOCKED`, which is how our payment work-queue is built; a different `ON CONFLICT`
syntax, which is our idempotency guard; a different locking and MVCC implementation,
so the concurrency test that proves we never oversell proves nothing; different
`SQLState` codes, so the exception handling that distinguishes a duplicate key from a
foreign-key violation is unverified; and different default `NULLS` ordering, which
silently changes pagination. `MODE=PostgreSQL` is a syntax-compatibility mode, not a
behaviour guarantee — it makes the mistake harder to spot rather than smaller.

On speed: with a singleton container and reuse, we pay a few seconds once per run and
zero on subsequent local runs. That is not the reason a Java suite is slow. Spring
context forking is, and that is a separate fix.

Where I would still accept an in-memory database: a test that needs *a* database only
so a context can start and asserts nothing about SQL. And even then I would rather
reuse the same container than add a second database to the classpath, because having
H2 on the classpath is how `@DataJpaTest` silently replaces your datasource and your
Testcontainers setup stops running without anyone noticing."

**What separates them:** the mid answer treats the difference as a risk. The senior
answer names specific features their own service depends on, distinguishes syntax
from semantics from concurrency, addresses the speed claim with the real cause of slow
Java suites, and states the conditions under which the other choice is acceptable.

**Follow-up the interviewer asks:** "How would you find out whether your current
suite is actually running against Postgres or has silently fallen back?" They want
`grep 'jdbc:'` over the test log, and ideally the `@DataJpaTest` /
`replace = NONE` trap named unprompted.

---

### Q2 — "Your Testcontainers suite takes eleven minutes. Where is the time?"

**Mid-level answer:** "Containers are slow to start. We could try reuse, or run tests
in parallel."

**Senior answer:** "I would not guess; there are two independent costs and they need
separating. First, count container starts: `grep -c 'started in PT'` in the build log.
If that number equals my test-class count, the containers are per-class and the fix is
a singleton in a static initialiser on a shared base class — no `@Testcontainers`, no
`@Container`. If it is larger than the class count, someone has a non-static
`@Container` and is starting one per test method.

Second, turn on `logging.level.org.springframework.test.context.cache=DEBUG` and read
the cache `size` and `hitCount`. If `size` is climbing per class, the cost is Spring
contexts, not containers — usually a `@MockitoBean` or a property override, which is
Topic 60's problem, not this one. Those two measurements split eleven minutes into two
numbers, and the fixes are completely different.

There is a third possibility worth ruling out cheaply: if the image is not pre-pulled
in CI, part of the time is a registry download on every build. That is fixed with a
warm layer cache or a pull step, not with anything in the test code."

**What separates them:** the senior answer refuses to guess, names two specific
commands that produce two specific numbers, and knows that the two costs have
different fixes. It also connects to Topic 60 rather than treating the suite as one
undifferentiated blob.

**Follow-up:** "The count says one container and the cache says size one. Now where is
the time?" They want you to go looking at the tests themselves — a missing index on
the test schema, a fixture that inserts 50,000 rows per test, or `fsync` still on.

---

### Q3 — "What is the difference between `@ServiceConnection` and `@DynamicPropertySource`?"

**Mid-level answer:** "`@ServiceConnection` is the newer, simpler one. It configures
the datasource automatically so you don't have to write the properties."

**Senior answer:** "They solve the same problem at different levels.
`@DynamicPropertySource` registers property *suppliers* into the environment before
the context refreshes — it is generic, it works for anything, and I write the property
names myself. `@ServiceConnection` registers a `ConnectionDetails` bean instead, and
Boot's auto-configuration prefers a `ConnectionDetails` bean over
`spring.datasource.*` properties. So it is not just less typing; it bypasses the
property layer entirely, which means it also works when properties are being set from
somewhere else.

I use `@ServiceConnection` for anything Boot has a `ConnectionDetails` type for —
JDBC, R2DBC, Kafka, Redis, Mongo — and `@DynamicPropertySource` for everything else,
typically my own configuration properties pointing at a WireMock container. For a
`GenericContainer` like Redis, `@ServiceConnection` needs a `name` because the type
alone is not enough to infer which service it is.

The detail that matters operationally is that **both register a context customizer**,
so both participate in the Topic 60 context cache key. That is why I put the container
wiring on one shared base class rather than duplicating it per test class — otherwise
I risk forking contexts as well as starting containers."

**What separates them:** knowing the mechanism (`ConnectionDetails` bean versus
environment properties, and the precedence between them), giving a rule for choosing,
and volunteering the context-cache consequence unprompted.

**Follow-up:** "What happens if you set both for the same service?" They want to hear
that `ConnectionDetails` wins, and that you would remove the redundant one rather than
leave two sources of truth.

---

### Q4 — "How do you keep tests independent when they share one database?"

**Mid-level answer:** "We use `@Transactional` on the test so everything rolls back
after each test."

**Senior answer:** "Rollback is the cheap default and it is right for single-
transaction repository tests. But it has a real hole: a rolled-back test never
commits, so it cannot observe anything that only happens at commit — deferred
constraints, commit-time triggers — and critically it breaks any test that needs a
*second* transaction, because the second transaction cannot see the first's
uncommitted writes. Our concurrency tests are exactly that shape, so they must not be
wrapped in a test transaction at all.

So the default for our persistence suite is `TRUNCATE ... RESTART IDENTITY CASCADE`
over every table except `flyway_schema_history`, in a single statement, in
`@AfterEach`. One statement because truncating tables individually fails on foreign
keys, and excluding the Flyway table because deleting it would make Flyway re-run
every migration.

The way I prove independence is to run the suite with
`-Dsurefire.runOrder=random` and to run individual tests alone. If a test passes in
the suite and fails alone, it is reading someone else's data. Container reuse makes
this visible fast, because a reused container carries the previous run's rows — reuse
does not cause order dependence, it exposes it."

**What separates them:** knowing the specific limitation of rollback rather than
treating it as the answer, and having a *proof* of independence rather than an
intention.

**Follow-up:** "Truncate on every test is slow at scale. What then?" Good answers:
truncate only the tables the test touched, use a per-test schema, or restore from a
template database — and a note that you would measure before optimising.

---

### Q5 — "Would you use Testcontainers for load testing?"

**Mid-level answer:** "Yes, it starts the same containers, so it should work."

**Senior answer:** "No, and for a structural reason rather than a preference. A load
test needs a long-lived stack with a seeded, realistic dataset, fixed resource limits,
and a JVM whose flags I control and record — because the whole point is that the run
is reproducible to within a tolerance. Testcontainers is designed around the opposite
property: ephemeral containers whose lifecycle is owned by a test JVM and whose ports
are random every run. Seeding five million order lines inside a per-run container
would dominate the measurement, and the container's resource limits would be whatever
the Docker daemon happens to give it.

For load I use a committed `docker compose` stack with pinned image digests, explicit
CPU and memory limits, a seed step that runs once, and the JVM flags recorded
alongside the results. The load generator runs outside that stack so it does not
compete for CPU with the thing it is measuring.

What does carry over is the discipline: pin images rather than using `latest`, let
Flyway own the schema, and use real readiness checks rather than sleeping. Those are
the same habits in both places."

**What separates them:** recognising that ephemerality and reproducibility are in
tension, and being specific about what a load environment needs that a test
environment does not. This is the question that bridges into Topic 65.

**Follow-up:** "Is there anything you would still use Testcontainers for in a
performance context?" A good answer: micro-scale reproductions — the two-thread
oversell test, a pool-exhaustion reproduction — where the container's speed matters
more than its realism, and the measurement is a count rather than a latency.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The singleton container is never stopped anywhere in the code. Explain precisely
   what removes it, and what happens to it if the JVM is killed with `SIGKILL`. Then
   explain why enabling reuse changes that answer.

2. `@Container static` gives one container per class, but a static initialiser gives
   one per JVM. Both use the `static` keyword. What is the actual difference in
   mechanism? (Topic 67 is relevant.)

3. `@DynamicPropertySource` must be a static method taking a `DynamicPropertyRegistry`
   and returning `void`, and it registers `Supplier`s rather than values. Give the
   reason for the `Supplier` specifically — what would break if it took plain strings?

4. Your suite runs one container and one Spring context, and takes six minutes. The
   tests themselves do very little. Name three plausible causes that have nothing to
   do with Testcontainers, and the single command you would run first for each.

5. A colleague proposes running the whole test suite against a *shared, long-lived*
   Postgres in CI instead of Testcontainers, to save startup time. List what you gain
   and what you lose. Then state the one property that shared database can never have,
   and why it matters more than the seconds.

6. Testcontainers gives you the same *database*. It does not give you the same
   *data volume*, the same *hardware*, or the same *concurrency*. Which classes of
   production bug does it therefore still fail to catch? Be specific — you will need
   this list in Topic 65.

7. You have made your suite fast by reusing a container across runs. A new engineer
   joins, clones the repo, runs `mvn test`, and three tests fail. Their machine is
   fine and yours is fine. What is the most likely cause, and what does that tell you
   about which of the two machines is telling the truth?

---

## Quick reference card

### The shape you should be writing

```java
// src/test/java/com/orderflow/support/PostgresTestBase.java
public abstract class PostgresTestBase {

    protected static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>(DockerImageName.parse("postgres:16-alpine"))
            .withDatabaseName("orderflow")
            .withUsername("orderflow")
            .withPassword("orderflow")
            .withCommand("postgres", "-c", "fsync=off", "-c", "synchronous_commit=off")
            .withReuse(true);

    static { POSTGRES.start(); }          // once per JVM. No @Container. No @Testcontainers.

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        r.add("spring.datasource.username", POSTGRES::getUsername);
        r.add("spring.datasource.password", POSTGRES::getPassword);
        r.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
    }
}
```

### Annotations

| Annotation | Where it goes | What it does |
|---|---|---|
| `@Testcontainers` | test class | Enables the JUnit 5 extension that manages `@Container` fields. |
| `@Container` **static** | test class field | One container per test **class**. |
| `@Container` **instance** | test class field | One container per test **method**. Almost always wrong. |
| `@ServiceConnection` | on a container field | Registers a `ConnectionDetails` bean; Boot wires the service with no property names. |
| `@ServiceConnection(name = "redis")` | on a `GenericContainer` | Same, when the type alone cannot identify the service. |
| `@DynamicPropertySource` | static method, `DynamicPropertyRegistry` param | Registers property **suppliers** before context refresh. |
| `@AutoConfigureTestDatabase(replace = NONE)` | on `@DataJpaTest` | **Required.** Stops Boot swapping your datasource for an embedded one. |

### Lifecycle cheat sheet

| Pattern | Container starts | Use when |
|---|---|---|
| Non-static `@Container` | per test **method** | Never, unless a test genuinely needs a pristine daemon. |
| Static `@Container` | per test **class** | A one-off class with an unusual container config. |
| Static initialiser on a shared base | per **JVM** | **The default. This is the answer.** |
| Shared base + `withReuse(true)` + `~/.testcontainers.properties` | per **machine**, until config changes | Local development, once cleanup is provably correct. |

### Commands

```bash
# Is Docker up?
docker info | grep -i 'server version'

# Pre-pull so the first run is not a download
docker pull postgres:16-alpine

# Watch containers during a run
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
docker events --filter 'type=container' --filter 'event=start'

# Count container starts in a run
mvn test 2>&1 | grep -c "started in PT"

# Context cache behaviour (put the logging level in src/test/resources/application.properties)
logging.level.org.springframework.test.context.cache=DEBUG
mvn test 2>&1 | grep "Spring test ContextCache" | tail -1

# Did anything fall back to H2?
mvn test 2>&1 | grep -i 'jdbc:' | sort -u

# Enable reuse (local only; never commit this file)
echo 'testcontainers.reuse.enable=true' >> ~/.testcontainers.properties

# Find and remove leftover containers
docker ps -a --filter 'label=org.testcontainers=true'
docker rm -f $(docker ps -aq --filter 'label=org.testcontainers=true')

# Prove tests are order-independent
mvn test -Dsurefire.runOrder=random

# What version did I actually resolve?
mvn dependency:tree -Dincludes=org.testcontainers
```

### Gotchas checklist

- [ ] `@Container` on an instance field starts a container **per test method**.
- [ ] `@Container static` is still **per class**. Use a static initialiser for per JVM.
- [ ] `@DataJpaTest` replaces your `DataSource` unless you add `replace = NONE`.
- [ ] Delete the H2 dependency. Leaving it on the classpath makes the previous line a
      silent failure instead of a loud one.
- [ ] Never `:latest`. Pin the image tag; match production's **major** version.
- [ ] Flyway owns the schema. `ddl-auto: validate`, never `create-drop`.
- [ ] Reuse needs **both** `~/.testcontainers.properties` and `.withReuse(true)`.
- [ ] Reuse means data survives. Your cleanup must already be correct.
- [ ] `TRUNCATE ... RESTART IDENTITY CASCADE` in one statement, excluding
      `flyway_schema_history`.
- [ ] `@Transactional` rollback cleanup breaks any test that needs a second
      transaction. Concurrency tests must not use it.
- [ ] Put the container wiring on **one** shared base class, or you fork the Topic 60
      context cache as well as starting containers.
- [ ] `fsync=off` is for throwaway test databases only. Never anywhere else.

---

## When would I use this at work?

**1. The first week on a new Java codebase.**
You open the test suite and find `jdbc:h2:mem`. That single line tells you which
categories of bug the team has been shipping without knowing: locking, upserts,
constraint-error handling, ordering. It is one of the highest-signal things you can
look at in an unfamiliar repository, and converting one test class to Testcontainers
is a small, uncontroversial pull request that starts the conversation with evidence
rather than opinion.

**2. Reproducing a production incident.**
An oversell happened. The existing test says it cannot. With a real Postgres and two
threads you can reproduce it locally in twenty minutes, which converts a
week-long argument about whether the bug is real into a failing test. That failing
test is then the artefact everyone agrees on. Without a real database you cannot even
begin, because the behaviour under investigation is the database's.

**3. Making CI trustworthy.**
The most expensive failure mode in a test suite is not slowness — it is the suite
that is green and wrong, or red for environmental reasons. Testcontainers removes an
entire category of both: no shared staging database being truncated by someone else,
no "did you remember to start compose", no drift between the CI database version and
production's. When a build goes red you know it is the code, which is the property
that makes people actually act on a red build.

---

## Connected topics

**Prerequisites:**

- **58 — JUnit 5:** `@Testcontainers` is a JUnit extension, and the per-method test
  instance lifecycle is exactly why a non-static `@Container` starts a container per
  test.
- **59 — Mockito:** the counterpart argument. Mock what you do not own and cannot
  run; run the real thing when you can. Testcontainers is what makes "you can" true
  for the database.
- **60 — Spring test slices and context caching:** the container wiring registers a
  context customizer, so it participates in the cache key. Put it on one base class.
  Also the source of the `replace = NONE` requirement on `@DataJpaTest`.
- **43 — Configuration and profiles:** `@DynamicPropertySource` writes into the same
  environment whose precedence chain you learned there.
- **67 — Class loading:** the singleton pattern relies on a static initialiser running
  exactly once per class loader. That is the mechanism, and it is worth understanding
  rather than trusting.

**This makes real, retroactively:**

- **48–49 — Persistence context and lazy loading:** flush timing and
  `LazyInitializationException` are only observable against a database that behaves
  like production's.
- **50 — N+1:** the statement-count assertion is the regression guard; the container
  is what makes the query plans and index usage behind that count real.
- **51 — Caching:** the stale-price demonstration needs a real native `UPDATE` against
  a real database.
- **52 — Locking and the oversell scenario:** the single most important thing on this
  list. Two threads, one unit of stock, real MVCC. Untestable without this topic.
- **53 — Batching:** sequence allocation and JDBC batch behaviour are database
  behaviours.
- **54–55 — Transactions, isolation and the connection pool:** isolation levels mean
  what Postgres implements, not what the standard describes.

**This unlocks:**

- **62 — Contract testing:** provider verification runs against a real stack.
- **63 — Mutation testing:** PIT re-runs your tests thousands of times. A suite whose
  database is a per-class container becomes unusable at that multiplier; the singleton
  is what makes mutation testing feasible at all.
- **64 — Property-based testing:** jqwik generates many cases per property, with the
  same multiplier argument.
- **65 — THE GATE:** the containerised stack, pinned images, Flyway-owned schema and
  readiness discipline all carry forward. What does **not** carry forward is
  Testcontainers itself — the gate needs a long-lived `docker compose` stack with a
  seeded 5-million-row dataset and fixed resource limits, for the reasons in Q5.
- **113 — Kafka consumers:** `KafkaContainer` plus `@ServiceConnection`, exactly the
  same pattern as Postgres here.
- **109 — HikariCP:** pool exhaustion is reproducible locally against a container with
  a deliberately tiny pool.
- **121 — Idempotency:** `ON CONFLICT` behaviour, which is one of the things H2 cannot
  verify.

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0. Testcontainers
versions are managed by Boot's BOM — do not pin them yourself, and check what you
resolved with `mvn dependency:tree -Dincludes=org.testcontainers`. `@ServiceConnection`
has been available since Boot 3.1 and is the form to write. The one thing in this
document I would re-verify on your exact version is the Kafka container class name,
which has changed as Testcontainers moved from the Confluent image to the official
Apache KRaft image.*
