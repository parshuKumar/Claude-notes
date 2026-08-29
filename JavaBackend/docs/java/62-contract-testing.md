# 62 — Contract Testing: Spring Cloud Contract / Pact

## Phase: 6 — Testing
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` gains a machine-checked contract on both of its service boundaries — as a **provider** to the storefront BFF, and as a **consumer** of the external payment gateway. The stubs the consumer tests run against become the same artefact the provider's build verifies.

---

## ELI5 anchor

Two workshops build two halves of a plug.

Workshop A builds the plug. Workshop B builds the socket.

Today, each workshop tests its own half against a **cardboard model** of the other
half, which it carved itself. Both models were carved on the day the teams first
talked. Both halves pass their tests.

Then Workshop A widens one pin by two millimetres. Their cardboard socket is still
the old shape, so their tests still pass. Workshop B's cardboard plug is also still
the old shape, so their tests still pass too. Nobody's build goes red.

The plug and socket only meet in the customer's house. That is where the failure
happens.

A **contract test** replaces both pieces of cardboard with **one shared metal
template**. Workshop B, who has to plug things in, cuts the template — they say what
shape they need. Workshop A must now push their plug through that same template
before their build is allowed to go green.

Widen the pin now and **Workshop A's build fails, on Workshop A's machine, before
anything ships**. That is the whole idea. Everything below is mechanism.

---

## The bridge from what you know

### What transfers — and you have most of it already

You have almost certainly used **Pact** from the Node side, or at least seen it. If
you have, you already own the important half of this topic: the idea that the
consumer writes an expectation, the expectation is serialised to a file, and the
provider replays it.

You have also definitely used `nock`, `msw`, or a hand-rolled Express stub in a Jest
suite. Hold on to that, because it is the thing this topic is arguing against.

| What you do in Node | Java equivalent | Verdict |
|---|---|---|
| `nock('https://payments').post('/authorize').reply(200, {...})` | A hand-written `WireMock` stub or a Mockito-stubbed client | **DIRECT** analogue — and both have the same flaw |
| `msw` handlers shared between tests | A shared `@TestConfiguration` stub bean | **DIRECT** analogue, same flaw |
| Pact consumer test → `pacts/*.json` → Pact Broker → provider verification | Pact-JVM, identical model, same broker | **DIRECT** — the file format is the same |
| — | **Spring Cloud Contract**: contract file → generated provider test **and** generated WireMock stub jar | **NO ANALOGUE** in the Node ecosystem |

### The one sentence that matters

> An integration test proves your service works **against a stub you wrote**. The
> stub is your *assumption* about the provider. Your assumption is exactly the thing
> that breaks.

Read that again, because everything in this document is a consequence of it.

Your Jest integration suite with `nock` does not test the payment gateway. It tests
your code against your memory of a Confluence page from four months ago. The suite
is green. The memory is stale. Nothing in your build system knows.

A **consumer-driven contract** removes the second copy. There is one artefact. The
consumer's stub and the provider's verification are the *same file*. If the provider
changes a field name, the provider's own build turns red — because the provider's
build now replays every consumer's recorded expectation.

### What does not transfer

- **There is no module mocking in Java.** (Topic 58 covered this.) In Jest you can
  `jest.mock('./paymentClient')` and intercept the import. Java has no such hook —
  the class is resolved by the classloader (Topic 67) and there is no interception
  point. So Java teams stub at the **HTTP layer** far more often than Node teams do,
  which makes HTTP stubs even more load-bearing, which makes them going stale even
  more dangerous.
- **Spring Cloud Contract generates code.** You write one contract file and a Maven
  plugin *writes the provider's test class for you* and packages the stubs into a
  jar with a `stubs` classifier. Nothing in the Node world does this. It is the
  single biggest ergonomic difference and it is why Spring shops pick it over Pact.

---

## What is this?

A **contract** is a written-down, executable description of one request/response
interaction between two services.

A **contract test** is a test that checks one side of that contract in isolation,
without running the other side.

There are two of them, and both are needed:

1. **The consumer test.** Your service runs against a stub *generated from the
   contract*. It proves you send what the contract says and can parse what the
   contract promises. No provider process is running.
2. **The provider verification test.** The provider's build replays the contract's
   requests against the real provider (usually via MockMvc or a running port) and
   asserts the real responses match the contract. No consumer process is running.

**Consumer-driven** means: the *consumer* authors the contract, because the consumer
is the one who knows what it actually needs. The provider then has an obligation to
satisfy it. Contrast with **provider-driven**, where the provider publishes a schema
and consumers are told to keep up — which is just OpenAPI with extra steps, and does
not tell the provider whether removing a field will break anybody.

### Where it sits between the tests you already have

| Test kind | Runs | Proves | Blind to |
|---|---|---|---|
| Unit (Topics 58–59) | One class, mocks | Logic inside that class | Everything about the wire |
| Slice (Topic 60) | One Spring layer | Serialization, routing, validation | Whether anyone else agrees with your JSON |
| Integration + Testcontainers (Topic 61) | Your service + real Postgres | Your service against reality *you control* | Reality you do **not** control |
| **Contract** | One service + a generated stub | Both sides agree on the wire format, **checked in both builds** | Whether the whole flow is semantically right |
| End-to-end | Everything | The flow really works | Nothing — but it is slow, flaky, and tells you *that* something broke, never *what* |

Contract testing is the layer that buys you **most of the confidence of E2E at the
cost and stability of a unit test**. That is its entire economic argument.

---

## Why does it matter?

Three concrete things, in the order you will meet them.

**1. It moves the failure into the right build.**
Without contracts, a provider's breaking change fails in the *consumer's*
production. With contracts, it fails in the *provider's* CI, at the moment of the
change, with the consumer named in the failure message. The person who can fix it
cheapest is the person who finds out.

**2. It makes deleting things possible.**
Every provider team has fields nobody dares remove because nobody knows who reads
them. A contract broker turns "who uses `legacy_total_cents`?" from an
archaeology project into a query. Without that, your API only ever grows.

**3. It kills a whole category of 3am page.**
The "we deployed the payment service and orders started 500ing" incident is
structurally a contract failure. It is not caught by unit tests, not caught by
integration tests, and caught by E2E only if your E2E suite happens to exercise that
exact path and is not currently disabled for flakiness.

---

## Syntax breakdown

Two tools. Learn Spring Cloud Contract properly (it is what Spring shops use), and
know Pact well enough to work in a polyglot org.

### Part A — Spring Cloud Contract

#### A1. Where contracts live

```
payment-gateway/                      <- the PROVIDER's repository
  src/test/resources/contracts/
    payments/
      shouldAuthorizeAValidPayment.groovy
      shouldRejectInsufficientFunds.groovy
  src/test/java/com/orderflow/contract/
      PaymentBase.java                <- you write this, once
  pom.xml
```

Contracts live in the **provider's** repo, but they are authored by (or PR'd by) the
consumer. That "PR to the provider's repo" step is what makes it consumer-driven.
Pact instead ships the contract through a broker; same idea, different plumbing.

#### A2. The contract DSL, line by line

```groovy
package contracts.payments

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "authorises a payment when the wallet has sufficient funds"

    request {
        method POST()
        url "/payments/authorize"
        headers {
            contentType applicationJson()
        }
        body(
            orderId:      $(consumer(regex(uuid())), producer("8f14e45f-ceea-467a-9f0e-6b3c2a1f0a11")),
            amountMinor:  $(consumer(anyPositiveInt()), producer(4599)),
            currency:     "GBP",
            walletId:     $(consumer(regex(uuid())), producer("2b1c9d70-1111-4a2b-9c3d-0e5f6a7b8c9d"))
        )
    }

    response {
        status OK()
        headers {
            contentType applicationJson()
        }
        body(
            authorizationId: $(consumer("AUTH-7781"), producer(regex("AUTH-[0-9]{4,}"))),
            status:          "AUTHORIZED",
            amountMinor:     fromRequest().body('$.amountMinor'),
            currency:        "GBP"
        )
        bodyMatchers {
            jsonPath('$.authorizationId', byRegex("AUTH-[0-9]{4,}"))
            jsonPath('$.status',          byRegex("AUTHORIZED|DECLINED"))
            jsonPath('$.amountMinor',     byType())
        }
    }
}
```

| Bit of syntax | What it means |
|---|---|
| `Contract.make { ... }` | One interaction. One file can hold a list of them, but one-per-file reads better in review. |
| `description` | Free text. It becomes the generated test method's Javadoc and the WireMock stub's name. Write it as a sentence about behaviour, because this is what a future engineer greps. |
| `method POST()` / `url "..."` | Exactly what you expect. `urlPath("/x") { queryParameters { ... } }` when you need query params matched separately. |
| `$(consumer(...), producer(...))` | **The single most important construct.** It gives *two different values*: the one baked into the stub the consumer runs against, and the one used when replaying against the real provider. |
| `regex(uuid())` | A built-in matcher. `uuid()`, `iso8601WithOffset()`, `number()`, `nonEmpty()` exist; `regex("...")` takes anything. |
| `fromRequest().body('$.amountMinor')` | The stub echoes a request field back. Without this, a stub returns a constant and your consumer test cannot tell echo from coincidence. |
| `bodyMatchers { jsonPath(...) }` | Applies to the **provider verification** side: assert the *shape*, not the literal value. Crucial for ids and timestamps — see Trap 3. |
| `byType()` | "any value of the same JSON type". The loosest useful matcher. |

**The `consumer`/`producer` split, said plainly:** when the stub is generated for the
consumer, the `consumer(...)` value is used. When the test is generated for the
provider, the `producer(...)` value is used. On a **request** field you want a loose
consumer value (the stub should match anything shaped like a UUID) and a concrete
producer value (the generated provider test has to send something real). On a
**response** field it is the other way round: the consumer wants a concrete value it
can assert on, the provider only has to match the pattern.

#### A3. The provider's Maven wiring

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-contract-maven-plugin</artifactId>
      <!-- version comes from the spring-cloud-dependencies BOM; see the note below -->
      <extensions>true</extensions>
      <configuration>
        <testFramework>JUNIT5</testFramework>
        <baseClassForTests>com.orderflow.contract.PaymentBase</baseClassForTests>
      </configuration>
    </plugin>
  </plugins>
</build>
```

> **Do not hardcode a version here.** Import the Spring Cloud BOM in
> `dependencyManagement` and let it manage the plugin and the starter together.
> Confirm which Spring Cloud release train matches your Boot line with:
> ```bash
> mvn -q dependency:tree -Dincludes=org.springframework.cloud
> mvn -q help:evaluate -Dexpression=spring-cloud.version -DforceStdout
> ```
> Spring Cloud's release train is versioned independently of Boot and must be
> matched to it; the compatibility matrix on `spring.io/projects/spring-cloud` is
> the authority, not me.

`<extensions>true</extensions>` is what binds the plugin's goals into the lifecycle,
so `mvn verify` runs them. The goals it adds:

| Goal | Lifecycle phase | What it does |
|---|---|---|
| `convert` | `generate-test-resources` | Groovy/YAML contracts → WireMock JSON stub mappings |
| `generateTests` | `generate-test-sources` | Contracts → JUnit 5 test classes in `target/generated-test-sources/contracts` |
| `generateStubs` | `package` | Packages the stub mappings into `<artifact>-<version>-stubs.jar` |

#### A4. The base class you write

The generated test extends this. Its only job is to put the provider into a state
where the contract's request can be answered.

```java
package com.orderflow.contract;

import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
public abstract class PaymentBase {

    @Autowired MockMvc mockMvc;

    @MockitoBean WalletLedger walletLedger;   // the one collaborator we must control

    @BeforeEach
    void setUp() {
        // Put the provider into the state the contract assumes.
        Mockito.when(walletLedger.availableMinor("2b1c9d70-1111-4a2b-9c3d-0e5f6a7b8c9d"))
               .thenReturn(50_000L);

        RestAssuredMockMvc.mockMvc(mockMvc);
    }
}
```

Two rules for base classes, learned the hard way:

- **Stub as little as possible.** Every mock here is a piece of the provider you are
  *not* verifying. Stub the far edge (a downstream bank API), never your own service
  layer. If your base class mocks `PaymentService`, your contract test verifies
  Jackson and nothing else.
- **Prefer a Testcontainers Postgres** (Topic 61) and real seeded rows over mocks
  where you can afford it. The state setup then becomes an SQL insert, which is
  honest about what the provider actually needs.

> `@MockitoBean` is the current annotation. `@MockBean` is deprecated in the Boot
> 3.4+/4.x line in favour of the `spring-test` bean-override annotations
> (`@MockitoBean`, `@MockitoSpyBean`).
>
> **[BOOT 3.x DELTA]** On Boot 3.3 and earlier the annotation is
> `org.springframework.boot.test.mock.mockito.@MockBean`. Same semantics, same
> context-cache-forking cost from Topic 60.

#### A5. `@AutoConfigureStubRunner` — the consumer side

This is the annotation to memorise.

```java
package com.orderflow.payments;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@AutoConfigureStubRunner(
    ids        = "com.orderflow:payment-gateway:+:stubs:8090",
    stubsMode  = StubRunnerProperties.StubsMode.LOCAL
)
class PaymentGatewayClientContractTest {

    @Autowired PaymentGatewayClient client;

    @Test
    void authorizesAPaymentAndReadsBackTheAuthorizationId() {
        AuthorizationResult result = client.authorize(
            new AuthorizationRequest(
                "8f14e45f-ceea-467a-9f0e-6b3c2a1f0a11",
                4599L,
                "GBP",
                "2b1c9d70-1111-4a2b-9c3d-0e5f6a7b8c9d"));

        assertThat(result.status()).isEqualTo(AuthorizationStatus.AUTHORIZED);
        assertThat(result.authorizationId()).startsWith("AUTH-");
        assertThat(result.amountMinor()).isEqualTo(4599L);
    }
}
```

Decoding the `ids` string — it is five colon-separated fields and people get it
wrong constantly:

```
com.orderflow : payment-gateway : +      : stubs      : 8090
groupId       : artifactId      : version: classifier : port
```

- `+` means "latest version". Use it locally; **pin it in CI** so a stub bump is a
  visible commit rather than an invisible Tuesday.
- `stubs` is the classifier of the jar `generateStubs` produced.
- `8090` is the port WireMock binds to. Omit it and a random port is chosen; you then
  read it back with `@StubRunnerPort("payment-gateway") int port;` and point your
  client at it. Random ports are better in CI where 8090 may be taken.

`stubsMode` values:

| Mode | Where stubs come from | Use it when |
|---|---|---|
| `CLASSPATH` | A `-stubs` jar on the test classpath | Monorepo, or provider stubs added as a test-scoped dependency. Fastest, no network. |
| `LOCAL` | Your `~/.m2` local repository | You just ran `mvn install` on the provider locally. Great for the drill below. |
| `REMOTE` | A Maven repo / Artifactory / a Git repo | CI. Add `repositoryRoot = "https://artifacts.internal/libs-release"`. |

Wire the client's base URL to the stub:

```java
// application-test.yml, or a @DynamicPropertySource
orderflow.payments.base-url: http://localhost:8090
```

### Part B — Pact, briefly

You need to recognise it, not master it, unless your org is polyglot — in which case
Pact wins because it speaks to Node, Go and Python consumers too.

**Consumer side:**

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-gateway")
class PaymentGatewayPactTest {

    @Pact(consumer = "orderflow")
    public V4Pact authorizesPayment(PactDslWithProvider builder) {
        return builder
            .given("wallet 2b1c9d70 has at least 4599 minor units available")
            .uponReceiving("an authorization request for 4599 GBP")
                .path("/payments/authorize")
                .method("POST")
                .body(newJsonBody(b -> {
                    b.uuid("orderId");
                    b.numberType("amountMinor", 4599);
                    b.stringValue("currency", "GBP");
                }).build())
            .willRespondWith()
                .status(200)
                .body(newJsonBody(b -> {
                    b.stringMatcher("authorizationId", "AUTH-[0-9]{4,}", "AUTH-7781");
                    b.stringValue("status", "AUTHORIZED");
                }).build())
            .toPact(V4Pact.class);
    }

    @Test
    @PactTestFor(pactMethod = "authorizesPayment")
    void authorizes(MockServer mockServer) {
        var client = new PaymentGatewayClient(mockServer.getUrl());
        assertThat(client.authorize(sampleRequest()).status())
            .isEqualTo(AuthorizationStatus.AUTHORIZED);
    }
}
```

> `@Pact` methods return `V4Pact` on pact-jvm 4.6 and later; older versions return
> `RequestResponsePact`. Check which your version wants — the compile error names the
> expected type, so this is a 10-second problem, not a research project.

**Provider side:**

```java
@Provider("payment-gateway")
@PactBroker(url = "${PACT_BROKER_URL}")
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class PaymentGatewayPactVerificationTest {

    @LocalServerPort int port;

    @BeforeEach
    void target(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @State("wallet 2b1c9d70 has at least 4599 minor units available")
    void seedFundedWallet() {
        walletRepository.save(new Wallet("2b1c9d70-...", 50_000L));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verify(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

The `@State` / `.given(...)` pairing is Pact's answer to Spring Cloud Contract's base
class: a named hook the provider implements to set up the world.

**The two differences that actually matter:**

| | Spring Cloud Contract | Pact |
|---|---|---|
| Where the contract lives | Provider's repo, as a file | A **broker**, published by the consumer's build |
| Transport | Stub jar via Maven | HTTP to the broker |
| Codegen | Generates the provider's test class | You write the provider test |
| Polyglot | JVM-centric (though stubs are plain WireMock JSON) | First-class in ~10 languages |
| Deployment safety | Manual | `pact-broker can-i-deploy` — a real gate |

`can-i-deploy` is Pact's strongest single feature: a CLI that answers "is the version
of orderflow I am about to deploy verified against every provider version currently
in production?" and exits non-zero if not.

---

## Example 1 — minimal

The smallest complete loop: one endpoint, one contract, both sides.

**The provider.** `payment-gateway` exposes a health-ish lookup.

```java
@RestController
class AuthorizationLookupController {

    private final AuthorizationStore store;

    AuthorizationLookupController(AuthorizationStore store) { this.store = store; }

    @GetMapping("/payments/{id}")
    AuthorizationView find(@PathVariable String id) {
        return store.find(id)
                    .map(a -> new AuthorizationView(a.id(), a.status().name(), a.amountMinor()))
                    .orElseThrow(() -> new NotFoundException(id));
    }
}

record AuthorizationView(String authorizationId, String status, long amountMinor) {}
```

**The contract**, `src/test/resources/contracts/payments/shouldReturnAnAuthorization.groovy`:

```groovy
package contracts.payments

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "returns a known authorization by id"
    request {
        method GET()
        url "/payments/AUTH-7781"
    }
    response {
        status OK()
        headers { contentType applicationJson() }
        body(
            authorizationId: "AUTH-7781",
            status:          "AUTHORIZED",
            amountMinor:     4599
        )
    }
}
```

**The base class:**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
public abstract class PaymentBase {

    @Autowired MockMvc mockMvc;
    @MockitoBean AuthorizationStore store;

    @BeforeEach
    void setUp() {
        Mockito.when(store.find("AUTH-7781"))
               .thenReturn(Optional.of(new Authorization("AUTH-7781", Status.AUTHORIZED, 4599L)));
        RestAssuredMockMvc.mockMvc(mockMvc);
    }
}
```

Now `mvn verify` in the provider generates a test that GETs `/payments/AUTH-7781`
and asserts the three fields. And `mvn install` publishes
`payment-gateway-<version>-stubs.jar`, which the consumer boots on port 8090 with
`@AutoConfigureStubRunner`.

**One contract file produced two artefacts that cannot drift from each other.** That
is the entire mechanism.

---

## Example 2 — production scenario on the project spine

`orderflow` sits in the middle of two boundaries. Both need contracts, and they run
in opposite directions.

```
storefront-bff  --(consumer)-->  orderflow  --(consumer)-->  payment-gateway
                                (provider)                    (provider)
```

### Boundary 1 — `orderflow` as a PROVIDER to `storefront-bff`

The storefront renders an order-detail page. It needs `GET /orders/{id}` with lines,
product names and payment status — the exact endpoint Topic 50 fixed the N+1 on and
the exact endpoint Topic 65 is about to hammer with load.

The storefront team PRs this into `orderflow/src/test/resources/contracts/orders/`:

```groovy
package contracts.orders

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "returns an order with its lines, product names and payment status"

    request {
        method GET()
        urlPath("/orders/3f9b2c11-4a5d-4e6f-8a90-1b2c3d4e5f60")
        headers {
            header("Authorization", $(consumer(regex("Bearer .+")),
                                      producer("Bearer " + JwtFixtures.CUSTOMER_TOKEN)))
            accept applicationJson()
        }
    }

    response {
        status OK()
        headers { contentType applicationJson() }
        body(
            orderId:      "3f9b2c11-4a5d-4e6f-8a90-1b2c3d4e5f60",
            status:       $(consumer("CONFIRMED"), producer(regex("PENDING|CONFIRMED|CANCELLED"))),
            totalMinor:   8_997,
            currency:     "GBP",
            placedAt:     $(consumer("2026-07-14T09:31:04Z"), producer(regex(iso8601WithOffset()))),
            payment: [
                status:            $(consumer("AUTHORIZED"), producer(regex("PENDING|AUTHORIZED|DECLINED"))),
                authorizationId:   $(consumer("AUTH-7781"), producer(regex("AUTH-[0-9]{4,}")))
            ],
            lines: [
                [
                    productId:   "b81c4d22-0000-4111-8222-333344445555",
                    productName: $(consumer("Aeropress Go"), producer(regex(nonEmpty()))),
                    quantity:    3,
                    unitMinor:   2_999,
                    lineMinor:   8_997
                ]
            ]
        )
        bodyMatchers {
            jsonPath('$.orderId',                 byRegex(uuid()))
            jsonPath('$.totalMinor',              byType())
            jsonPath('$.placedAt',                byRegex(iso8601WithOffset()))
            jsonPath('$.lines',                   byType { minOccurrence(1) })
            jsonPath('$.lines[0].productName',    byRegex(nonEmpty()))
            jsonPath('$.lines[0].lineMinor',      byType())
        }
    }
}
```

Read the design decisions in that file:

- **`totalMinor`, `unitMinor`, `lineMinor`** — money as `long` minor units, per Topic
  01, Trap 5. The contract enforces that across the boundary too. If someone
  "helpfully" changes the API to decimal pounds, the storefront's stub stops matching
  and the provider's generated test fails.
- **`lines` is matched with `minOccurrence(1)`**, not by exact content. The
  storefront needs *at least one line with these fields*, not this specific coffee
  press. Over-specifying array contents is Trap 3.
- **`status` is a regex on the producer side.** The storefront's stub always returns
  `CONFIRMED` so its rendering test is deterministic; the provider is allowed to
  return any of the three. This is exactly the `consumer`/`producer` split earning
  its keep.
- **The `Authorization` header is in the contract.** Auth is part of the contract.
  A provider that starts requiring a scope the consumer does not send is a breaking
  change, and a contract that omits the header cannot catch it.

The base class seeds a real order into a Testcontainers Postgres rather than mocking:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
@Testcontainers
public abstract class OrderApiBase {

    @Container @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:17-alpine");   // pin whatever tag you standardised in Topic 61

    @Autowired MockMvc mockMvc;
    @Autowired OrderTestDataFactory data;

    @BeforeEach
    void seed() {
        data.deleteAll();
        data.anOrder()
            .withId("3f9b2c11-4a5d-4e6f-8a90-1b2c3d4e5f60")
            .withStatus(OrderStatus.CONFIRMED)
            .withLine("b81c4d22-0000-4111-8222-333344445555", "Aeropress Go", 3, 2_999L)
            .withAuthorizedPayment("AUTH-7781")
            .persist();

        RestAssuredMockMvc.mockMvc(mockMvc);
    }
}
```

**This is where the topic pays a second dividend.** Topic 50 gave you a
query-counting assertion so N+1 could not regress. Put that assertion in this base
class:

```java
    @AfterEach
    void assertNoNPlusOne() {
        // Hibernate Statistics were enabled for the test profile in Topic 50.
        assertThat(statistics.getPrepareStatementCount())
            .as("GET /orders/{id} must stay a bounded number of queries")
            .isLessThanOrEqualTo(3);
    }
```

Now the contract test for the storefront's most important read path also **locks the
query count**. A future developer who satisfies the contract by lazily loading each
line's product inside the serializer will pass the field assertions and fail the
counter. The contract enforces the wire; the counter enforces the cost.

### Boundary 2 — `orderflow` as a CONSUMER of `payment-gateway`

This is the direction that saves you at 3am. `orderflow`'s order-placement path
(Topic 54's transactional flow: reserve inventory → debit wallet → authorize payment
→ publish event) calls the gateway.

```java
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.orderflow:payment-gateway:+:stubs",   // random port
    stubsMode = StubRunnerProperties.StubsMode.LOCAL)
@ActiveProfiles("contract")
class PlaceOrderPaymentContractTest {

    @StubRunnerPort("payment-gateway") int gatewayPort;

    @Autowired PlaceOrderUseCase placeOrder;
    @Autowired PaymentGatewayProperties props;

    @BeforeEach
    void pointClientAtStub() {
        props.setBaseUrl("http://localhost:" + gatewayPort);
    }

    @Test
    void authorizedPaymentMovesTheOrderToConfirmed() {
        PlaceOrderResult result = placeOrder.handle(
            new PlaceOrderCommand(customerId, List.of(new LineCommand(productId, 3))));

        assertThat(result.status()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(result.payment().authorizationId()).startsWith("AUTH-");
    }

    @Test
    void declinedPaymentRollsBackTheInventoryReservation() {
        // matches the shouldRejectInsufficientFunds.groovy contract
        PlaceOrderResult result = placeOrder.handle(commandForBrokeCustomer());

        assertThat(result.status()).isEqualTo(OrderStatus.CANCELLED);
        assertThat(inventory.availableFor(productId)).isEqualTo(stockBefore);
    }
}
```

The second test is the one worth staring at. It exercises Topic 54's rollback path
against a *real declined response shape from the provider's own contract*, not
against a hand-rolled `Mockito.when(gateway.authorize(any())).thenThrow(...)`. If
the gateway team changes a decline from HTTP 200 + `status: DECLINED` to HTTP 402,
your rollback code has to cope, and this test finds out — because the contract file
changed and the stub changed with it.

**[BOOT 3.x DELTA]** Everything above is annotation-identical on Boot 3.x. Two real
differences: (1) `@MockBean` instead of `@MockitoBean`, as noted; (2) Boot 4 ships
`RestTestClient`, which you may prefer to `RestAssuredMockMvc` in the base class —
Spring Cloud Contract's generated tests still use RestAssured by default, so this
only affects code *you* write in the base class. Also note Jackson 3 in Boot 4: if
your contract asserts on a serialized shape that depended on a Jackson 2 default
(for example null inclusion), the contract will catch the change for you. That is a
feature.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the hand-written stub that everybody trusts

**Wrong:**

```java
@TestConfiguration
class StubPaymentGatewayConfig {
    @Bean PaymentGatewayClient paymentGatewayClient() {
        return request -> new AuthorizationResult(
            "AUTH-7781", AuthorizationStatus.AUTHORIZED, request.amountMinor());
    }
}
```

Someone wrote this in March. It is used by 40 tests. It is nobody's job.

In June the gateway team renames the response field `authorizationId` to `authId`
and ships it. Their unit tests pass. Their integration tests pass. Nothing in
`orderflow`'s repository changed, so `orderflow`'s CI does not even run.

**Exact symptom:** `orderflow`'s CI is fully green on `main`. In production,
`POST /orders` starts returning 500. The log shows:

```
com.fasterxml.jackson.databind.exc.MismatchedInputException:
  Missing required creator property 'authorizationId' (index 0)
   at [Source: (String)"{"authId":"AUTH-7781","status":"AUTHORIZED",...
```

Orders are created, inventory is reserved, the wallet is debited, and then
serialization of the *response* blows up — so you have half-applied side effects. The
error rate on the placement endpoint goes to 100% while the read endpoints stay
perfectly healthy, which is the signature that misleads people into blaming the
database.

**Root cause:** the stub is a private copy of an assumption. Two copies of a fact
exist and only one of them has an owner. Nothing in either build system connects
them.

**Fix:** delete the stub. Replace it with `@AutoConfigureStubRunner` against the
provider's published stub jar (or a Pact broker interaction). Now there is one copy.
The gateway team's rename fails **the gateway's own build**, in the generated
provider test, with a message naming `orderflow` as the consumer whose contract
broke.

**The organisational half of the fix, which matters more:** the provider's CI must
run consumer contract verification on every commit, and that job must be
non-optional. A contract test that lives in a nightly job somebody muted is a
hand-written stub with extra ceremony.

---

### Trap 2 — provider-written contracts, called "consumer-driven"

**Wrong:** the payment-gateway team writes all the contracts, in their own repo,
describing what their API currently does.

**Exact symptom:** the contract suite has never once gone red on a real change. When
the team removes `legacy_total_cents`, they simply edit the contract in the same
commit — one green build, one broken consumer. Six months later somebody notices the
contract suite has caught zero defects and proposes deleting it, and they are right.

**Root cause:** a contract written by the provider is a **snapshot of the
implementation**, not a statement of anybody's requirement. It regenerates itself
whenever the implementation changes, so it can never disagree with it. You have
built a very slow way of asserting that your code equals your code.

**Fix:** two structural changes.

1. **Consumers author contracts** — a PR to the provider's `contracts/` directory
   (Spring Cloud Contract), or a broker publish from the consumer's build (Pact).
   The `CODEOWNERS` file should require a consumer-team review on that directory.
2. **Removing or narrowing anything in a contract requires the consumer's approval.**
   Adding is fine unilaterally. This is the same additive-only compatibility rule you
   already apply to Kafka schemas (Topic 19) and it is the actual point of the
   exercise.

Pact's `can-i-deploy` enforces this mechanically, which is why polyglot orgs drift
toward Pact even in Spring-heavy stacks.

---

### Trap 3 — the over-specified contract that fails on Tuesdays

**Wrong:**

```groovy
response {
    status OK()
    body(
        orderId:   "3f9b2c11-4a5d-4e6f-8a90-1b2c3d4e5f60",
        placedAt:  "2026-07-14T09:31:04Z",
        totalMinor: 8997
    )
    // no bodyMatchers block
}
```

**Exact symptom:** the provider's generated verification test fails with

```
Parsed JSON [{"orderId":"7c1e...","placedAt":"2026-08-29T11:02:47Z",...}]
doesn't match the JSON path [$[?(@.placedAt == '2026-07-14T09:31:04Z')]]
```

on every single run, because the provider generates a fresh UUID and stamps
`Instant.now()`. Someone "fixes" it by freezing the clock in the base class, which
works, and then someone else adds a field, and eventually the team marks the whole
contract module `-DskipTests` in CI "until we have time to look at it".

**Root cause:** without `bodyMatchers`, every value in the response block is compared
**by equality** during provider verification. A contract asserts *the shape both
sides agreed on*. Identifiers and timestamps have no agreed value — only an agreed
format.

**Fix:** every non-deterministic field gets a matcher.

```groovy
bodyMatchers {
    jsonPath('$.orderId',  byRegex(uuid()))
    jsonPath('$.placedAt', byRegex(iso8601WithOffset()))
    jsonPath('$.totalMinor', byType())
}
```

**The judgement call:** match by type where the consumer only needs the field to
exist, by regex where the consumer parses it, and by exact value only where the value
is genuinely part of the agreement — enum names like `AUTHORIZED`, currency codes,
error codes the consumer branches on. Ask "would the consumer's code behave
differently if this value changed?" If no, do not assert on it.

---

### Trap 4 — stale stubs, silently

**Wrong:** `stubsMode = LOCAL` with `ids = "...:+:stubs"` in CI.

**Exact symptom:** the worst kind — **none**. The consumer build is green. It has
been running against a stub jar that was installed into `~/.m2` on a build agent
eleven weeks ago, because the version selector `+` resolved to the newest thing
already present locally and nothing forced a re-resolve. Meanwhile the provider has
shipped four breaking changes, and its own verification suite is green because it
verifies against the *current* contract files. Both builds are green. The contract
system is doing nothing whatsoever.

**Root cause:** `LOCAL` is a developer convenience. It reads `~/.m2`, which on a
long-lived build agent is a cache with no invalidation policy.

**Fix:**

```java
@AutoConfigureStubRunner(
    ids = "com.orderflow:payment-gateway:2026.8.14:stubs",   // pinned
    stubsMode = StubRunnerProperties.StubsMode.REMOTE,
    repositoryRoot = "https://artifacts.internal/libs-release")
```

- `REMOTE` in CI, always.
- **Pin the version.** A stub upgrade then arrives as a dependency-bump PR you can
  see, review and revert, exactly like any other dependency (Topic 32).
- Add a scheduled job that opens that bump PR automatically, so pinning does not mean
  freezing.
- With Pact, the equivalent hygiene is verifying against broker `latest` for the
  consumer's deployed environment tags, plus `can-i-deploy` in the deploy pipeline.

---

### Trap 5 — using contracts to test business logic

**Wrong:** 34 contract files for the order API, covering the free-shipping threshold,
the bulk-discount tiers, three tax jurisdictions and every rejection reason.

**Exact symptom:** the provider build takes 14 minutes, the base class is 400 lines
of state setup, and any change to pricing requires editing a dozen `.groovy` files in
a language nobody on the team writes. Adding a discount tier becomes a two-day task
and people start routing around the system.

**Root cause:** category error. A contract's subject is **the shape of the
conversation**: paths, methods, status codes, headers, field names, field types,
enum values, error format. Business rules are the *provider's* concern and belong in
the provider's unit tests — where they run in milliseconds and, after Topic 63, are
mutation-tested.

**Fix:** the rule of thumb.

> One contract per **distinct response shape**, not per business case.

For `orderflow`'s order API that is roughly four: a successful order, a validation
failure (RFC 9457 `ProblemDetail`, Topic 46), a not-found, and an insufficient-stock
conflict. Whether the discount is 10% or 15% is not a contract question. Whether
`totalMinor` is an integer and `problem.type` is a URI absolutely is.

---

## Hands-on proof

Everything here is a command **you** run. I have no JVM and no Maven repository, so I
will not print output and call it real. What I can give you exactly is the command,
the file to look at, and how to read every outcome.

### Setup

```bash
cd ~/code/orderflow
java --version          # expect 21 or 25
mvn --version
mvn -q dependency:tree -Dincludes=org.springframework.cloud | head -40
```

**What to look for:** a `spring-cloud-contract-*` artifact and the release-train
version. If nothing prints, the BOM is not imported yet — add
`spring-cloud-dependencies` to `dependencyManagement` first.

### Proof 1 — the plugin really generates a test class

```bash
mvn -pl payment-gateway clean test-compile
find payment-gateway/target/generated-test-sources/contracts -name '*.java' | sort
```

**What to look for:** one Java file per contracts sub-directory, named after it
(e.g. `PaymentsTest.java`), containing one `@Test` method per `Contract.make` block.
Open it.

| What you see | What it means |
|---|---|
| A generated class extending `PaymentBase` with RestAssured calls | Correct. You now have a provider test you did not write. |
| Directory does not exist | `generateTests` did not bind. Check `<extensions>true</extensions>` on the plugin. |
| Class exists but extends `Object` | `baseClassForTests` is unset or misspelled. The generated test will fail with "no MockMvc configured". |
| One test method, several contracts | Multiple `Contract.make` blocks in one file collapse into one class — expected. |

**How to read it:** the assertions in that file were written by the plugin from your
contract. That is the artefact that will fail on the provider's machine when the
provider breaks a consumer. Read it once, properly. After this you will trust the
mechanism instead of believing in it.

### Proof 2 — break the provider, watch the PROVIDER's build fail

This is the proof that justifies the whole topic. Do it.

```bash
# 1. Green baseline
mvn -pl payment-gateway clean verify
```

**Look for:** `BUILD SUCCESS` and the generated contract test in the surefire summary.

```bash
# 2. Break it — rename the response field in the controller's view record
#    AuthorizationView(String authorizationId, ...)  ->  (String authId, ...)
mvn -pl payment-gateway clean verify
```

| What you see | What it means |
|---|---|
| `BUILD FAILURE` in the generated contract test, message naming `$.authorizationId` | **The point of the exercise.** The provider's own build caught a change that would have 500'd a consumer in production. |
| `BUILD SUCCESS` | Something is not wired. Either the generated tests are excluded by a surefire `<includes>`, or you changed only the internal field name and Jackson still emits the old JSON name via `@JsonProperty`. Check the actual serialized output first. |
| Compile error in the generated sources | You renamed something the base class references. Fix the base class; the generated test is downstream of it. |

Then restore the field and re-run to confirm green.

### Proof 3 — inspect the stub jar

```bash
mvn -pl payment-gateway install
ls ~/.m2/repository/com/orderflow/payment-gateway/*/ | grep stubs
unzip -l ~/.m2/repository/com/orderflow/payment-gateway/*/payment-gateway-*-stubs.jar
```

**What to look for:** entries under `META-INF/.../mappings/` — plain WireMock JSON —
and under `.../contracts/` — your original `.groovy` files, shipped for traceability.

```bash
unzip -p ~/.m2/repository/com/orderflow/payment-gateway/*/payment-gateway-*-stubs.jar \
  'META-INF/*/mappings/*.json' | head -60
```

**How to read it:** this JSON is what your consumer test actually runs against. It is
ordinary WireMock, which means you can debug it with ordinary WireMock knowledge, and
it means a Node consumer could run the same stub. The "magic" is a code generator
plus a jar.

### Proof 4 — the consumer runs against the real stub

```bash
mvn -pl orderflow test -Dtest=PaymentGatewayClientContractTest
```

**What to look for** in the log during startup:

| What you see | What it means |
|---|---|
| `Started stub server for [com.orderflow:payment-gateway...] at port XXXX` | Stub Runner resolved and booted the jar. Good. |
| `Exception ... Unable to find stubs ...:+:stubs` | Not installed locally. Run `mvn -pl payment-gateway install`, or switch to `REMOTE` with a `repositoryRoot`. |
| `Connection refused` from your client | Your client's base URL is not pointing at the stub port. Use `@StubRunnerPort` and set the property. |
| Test fails on a JSON field | Genuine mismatch — either the client's DTO is wrong or the contract is. **This is a real finding, not a setup problem.** |

### Proof 5 — see the stub with `curl`

Run the stub without a test at all:

```bash
# Standalone Stub Runner Boot, if you have it; otherwise put a breakpoint / Thread.sleep
# in the test above and curl the port it printed.
curl -s -X POST http://localhost:8090/payments/authorize \
  -H 'Content-Type: application/json' \
  -d '{"orderId":"8f14e45f-ceea-467a-9f0e-6b3c2a1f0a11","amountMinor":4599,"currency":"GBP","walletId":"2b1c9d70-1111-4a2b-9c3d-0e5f6a7b8c9d"}' | jq .
```

| What you see | What it means |
|---|---|
| The contract's response body | Matched. Note `amountMinor` echoing your request — that is `fromRequest()` working. |
| HTTP 404 with an empty body | **No stub matched your request.** WireMock returns 404 for unmatched. Your request differs from the contract's request block — usually a missing header or a body field the matcher rejects. |
| Connection refused | Nothing is listening; the stub server is only up while the test is running. |

The 404-on-no-match behaviour is worth internalising. A consumer test failing with a
404 it never expected almost always means "your request does not match the contract",
not "the endpoint is missing".

### Proof 6 — prove the query-count assertion still holds

```bash
mvn -pl orderflow test -Dtest=OrderApiContractTest -Dlogging.level.org.hibernate.SQL=DEBUG
```

**What to look for:** the count in the `@AfterEach` assertion from Topic 50, and the
number of `select` lines in the log. If the assertion trips, the contract may still
be satisfied — the JSON is right, the cost is wrong. That is precisely the class of
regression Topic 65 is about to measure under load.

---

## Practice exercises

### 1 — Easy: one contract, both sides, end to end

Take `orderflow`'s `GET /products/{id}`.

1. Write one Groovy contract for a successful lookup, with `bodyMatchers` on the id
   and on `priceMinor`.
2. Write the base class using a Testcontainers Postgres with one seeded product.
3. `mvn verify` — open the generated test and read it.
4. Rename `priceMinor` to `price` in the response DTO and re-run. **Paste the exact
   failure message.**
5. Restore it and confirm green.

Deliverable: the contract file, the generated test file, and the failure message from
step 4.

### 2 — Medium: combine it with what you already built

This uses Topics 46 (ProblemDetail), 50 (query counting), 57 (JWT auth), 60 (slices)
and 61 (Testcontainers).

Write the **complete contract set** for `GET /orders/{id}` — four contracts, no more:

- 200 with lines, product names, and payment status
- 404 as an RFC 9457 `ProblemDetail` (assert `type`, `title`, `status`; matcher on
  `instance`)
- 401 when the `Authorization` header is absent
- 403 when the JWT belongs to a different customer

Requirements:

- The base class seeds real rows, mocks nothing you own.
- The base class carries the Topic 50 query-count assertion in `@AfterEach`.
- Every non-deterministic field has a matcher. Zero frozen clocks.
- Then answer in writing: **which of these four contracts would a provider-authored
  contract suite have been least likely to include, and why does that matter?**

### 3 — Hard: production simulation, both directions

Set up the full triangle: `storefront-bff` (consumer) → `orderflow` (provider and
consumer) → `payment-gateway` (provider). Two contract relationships.

**Part A.** Get all four builds green (two providers, two consumers) with `REMOTE`
stubs and pinned versions.

**Part B.** Now play out a real incident, in this order, recording what goes red at
each step:

1. `payment-gateway` changes a decline from `200 {"status":"DECLINED"}` to
   `402 ProblemDetail`. Which build fails? At what commit?
2. Fix `orderflow` to handle both, keeping the Topic 54 rollback behaviour. Prove
   the rollback with an assertion on inventory levels.
3. `orderflow` now wants to remove `legacy_total_cents` from its order response.
   Show, using the contract system alone, whether the storefront still reads it. If
   your tooling cannot answer that question, say so — and describe what you would add
   (a broker, a `can-i-deploy` gate) so that it could.

**Part C.** Write down the honest cost. How long does the contract stage add to each
build? How many contract files ended up existing? At what number of files would you
stop and say the suite has become a liability? Give a number and defend it.

**Part D.** Argue the other side. Name a service boundary in a real system where you
would **not** add contract tests, and say exactly what makes it not worth it.

---

## Interview questions

### Q1 — "What does a contract test prove that an integration test doesn't?"

**Mid-level answer:** "It checks that the API matches between two services, so you
catch breaking changes without running everything together."

**Senior answer:** "An integration test verifies my service against a stub I wrote —
so it verifies my *assumption* about the provider, which is exactly the thing that
goes stale. A consumer-driven contract makes the stub and the provider's verification
the same artefact. The consequence is where the failure lands: a provider's breaking
change fails the *provider's* build, at the commit that caused it, naming the
consumer it breaks. That is a different economic proposition entirely — the person
who can fix it cheapest finds out first. It also gives me something E2E doesn't:
E2E tells me something broke, a contract failure tells me which field, in which
direction, for which consumer."

**What separates them:** the mid answer describes the mechanism. The senior answer
identifies *whose build goes red* as the point, and names the failure mode
(assumption drift) rather than the feature.

**Follow-up:** "So do you delete your E2E suite?" The answer they want: no — keep a
small number of true end-to-end journeys for wiring, auth and infrastructure, but
stop using E2E as the compatibility mechanism, because it is slow, flaky, and only
runs after both things are deployed.

---

### Q2 — "Who owns the contract in a consumer-driven model, and what happens when they disagree?"

**Mid-level answer:** "The consumer writes it and the provider has to satisfy it."

**Senior answer:** "The consumer authors it, because only the consumer knows what it
actually reads. The provider owns satisfying it, and the provider's build is the
enforcement point. When they disagree, the practical rule is: additions are
unilateral, removals and narrowings require consumer sign-off — the same
additive-compatibility discipline as a Kafka schema registry. The failure mode I
watch for is a provider writing its own contracts: that is a snapshot of the
implementation, it regenerates whenever the implementation changes, so it can never
go red. A contract suite that has never caught a defect isn't proof of quality, it's
proof it isn't wired to anything. And at organisational scale you need a broker plus
a `can-i-deploy` gate, because five consumers mean five opinions and someone has to
be able to ask 'is this version safe to deploy against everything currently in
prod?'"

**What separates them:** naming the provider-written-contract anti-pattern
unprompted, and treating "has this ever gone red?" as the health metric of the suite.

**Follow-up:** "How do you deprecate a field then?" Looking for: add the replacement,
get every consumer's contract off the old field (visible in the broker), then remove
— with the contract system as the evidence, not a Slack thread.

---

### Q3 — "Spring Cloud Contract or Pact?"

**Mid-level answer:** "Spring Cloud Contract if you're on Spring, Pact if you're
not."

**Senior answer:** "Roughly, but the real axis is org shape. Spring Cloud Contract's
advantage is code generation — you write one file and it produces both the provider's
test class and a WireMock stub jar that ships through Maven, which is essentially
free in a JVM monorepo with an internal Artifactory. Pact's advantage is the broker
and `can-i-deploy`, plus first-class support in Node, Go and Python — which matters
the moment a consumer isn't a JVM service, and my consumers usually include a
TypeScript BFF. If I had a JVM-only estate with a good artifact repository I'd take
Spring Cloud Contract for the ergonomics. In a polyglot org I'd take Pact, and I'd
accept writing the provider test by hand as the price of the broker. Worth noting the
two aren't as far apart as they look — SCC's stubs are plain WireMock JSON, so a Node
consumer can consume them; you just lose the broker."

**What separates them:** naming `can-i-deploy` as the specific thing Pact has,
picking based on estate shape rather than framework loyalty, and knowing SCC's stubs
are portable.

**Follow-up:** "What does `can-i-deploy` actually check?" That the version you're
deploying has a verification result against every counterparty version currently
tagged as deployed in the target environment.

---

### Q4 — "Your contract test suite is green, and a provider change still broke production. How?"

**Mid-level answer:** "Maybe the change wasn't covered by a contract."

**Senior answer:** "Several ways, and I'd check them in cost order. First: was the
consumer running against a **stale stub**? `LOCAL` mode with a `+` version selector
on a long-lived build agent resolves to whatever's in `~/.m2`, which can be months
old — both builds green, nothing actually compared. Second: were the contracts
provider-authored, so they were edited in the same commit as the change? Third: was
the break **semantic rather than structural** — the field is still a string, still
present, but now means something else, like an amount switching from minor units to
decimal. A contract checks shape, not meaning, and that's a real limitation I'd cover
with a consumer-side assertion on a known value. Fourth: was the contract verification
job actually required, or is it a nightly somebody muted? I'd look at when that job
last failed. A suite that's never gone red is a suite that isn't connected."

**What separates them:** four concrete hypotheses with a triage order, and naming the
genuine limitation — contracts verify structure, not semantics.

**Follow-up:** "How would you catch the minor-units-to-decimal change?" Good answers:
a contract with an exact value on a known fixture, so the number itself is part of
the agreement; or a shared value-object library; or, honestly, an integration
environment — and admitting that some semantic drift is not a contract problem.

---

### Q5 — "How many contract tests should the `orderflow` order API have?"

**Mid-level answer:** "Enough to cover the main endpoints and error cases."

**Senior answer:** "One per distinct response shape, not one per business case.
For `GET /orders/{id}` that's about four: the success shape, a `ProblemDetail`
validation failure, a not-found, and an insufficient-stock conflict. Whether the bulk
discount kicks in at ten units or twelve is not a contract question — that's a
provider unit test that runs in a millisecond and gets mutation-tested. The
anti-pattern is contract-explosion: I've seen suites with thirty-plus contracts per
endpoint where the base class is four hundred lines of state setup and adding a
pricing tier is a two-day task. At that point people route around the system, which
is worse than not having it. Concretely, the test I'd write is: 'would the consumer's
code behave differently if this value changed?' If no, it doesn't belong in the
contract."

**What separates them:** a stated rule with a rationale, awareness that contract
suites have a maintenance cliff, and giving the discriminating question rather than a
number.

**Follow-up:** "Where do the business rules get tested then?" Provider unit tests —
and after Topic 63, mutation-tested provider unit tests, because that's the layer
where a silent pricing bug costs money.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A contract test never runs the provider and the consumer in the same process. Why
   is that a *feature* rather than a compromise? What would you lose by running both?

2. A provider adds a new required field to a **request** body. No contract mentions
   it. Every contract test on both sides stays green, and production breaks. Explain
   the asymmetry between adding to a request and adding to a response — and say what
   you would change so the contract system catches it.

3. `$(consumer(regex(uuid())), producer("8f14e45f-..."))` gives two different values.
   Explain, in one sentence each, why a *request* field wants a loose consumer value
   and a concrete producer value, and why a *response* field wants the opposite.

4. You have five consumers of `orderflow`'s order API. One of them is a batch job that
   runs monthly and has no contract. What does that do to the guarantee the other four
   contracts give you? What is the honest thing to tell your team about it?

5. Contract testing proves structural compatibility. Name a production incident it
   would *not* have caught, and explain why not. Then say which testing layer would
   have — and whether it is worth having.

6. Topic 50 gave you an assertion on query count. That assertion is invisible on the
   wire — a contract cannot express it. So why is the contract base class a *good*
   place to put it, and what does that say about what a contract test class really is?

7. Your team is deciding between one contract per endpoint and one per response shape.
   Both are defensible. Make the strongest case for each, then say which you would
   pick for `orderflow` and what would make you switch.

---

## Quick reference card

### Spring Cloud Contract cheat sheet

```
src/test/resources/contracts/<group>/<name>.groovy   contracts live here (provider repo)
mvn verify                                           generate + run provider tests
mvn install                                          also publishes <artifact>-<ver>-stubs.jar
target/generated-test-sources/contracts/             the generated provider tests
```

```groovy
Contract.make {
  description "..."
  request  { method POST(); url "/x"; headers { contentType applicationJson() }; body(...) }
  response { status OK(); headers { contentType applicationJson() }; body(...)
             bodyMatchers { jsonPath('$.id', byRegex(uuid())) } }
}
```

| Construct | Use |
|---|---|
| `$(consumer(X), producer(Y))` | Different value for stub vs provider verification |
| `value(consumer(X), producer(Y))` | Same thing, longer form |
| `fromRequest().body('$.field')` | Echo a request field into the response |
| `byRegex(...)`, `byType()`, `byType { minOccurrence(1) }` | Provider-side shape matchers |
| `regex(uuid())`, `iso8601WithOffset()`, `nonEmpty()`, `number()` | Built-in patterns |
| `urlPath("/x") { queryParameters { parameter 'page', 1 } }` | Match query params separately |

### `@AutoConfigureStubRunner`

```java
@AutoConfigureStubRunner(
    ids = "group:artifact:version:stubs:port",   // version `+` = latest; port optional
    stubsMode = StubsMode.LOCAL,                 // CLASSPATH | LOCAL | REMOTE
    repositoryRoot = "https://artifacts/...")    // REMOTE only
@StubRunnerPort("artifact") int port;            // read back a randomly assigned port
```

| Mode | Source | Use for |
|---|---|---|
| `CLASSPATH` | `-stubs` jar on test classpath | Monorepo |
| `LOCAL` | `~/.m2` | Local development |
| `REMOTE` | Artifact repository | **CI — always** |

### Pact cheat sheet

```java
// consumer
@ExtendWith(PactConsumerTestExt.class) @PactTestFor(providerName = "payment-gateway")
@Pact(consumer = "orderflow") V4Pact aPact(PactDslWithProvider b) { ... }

// provider
@Provider("payment-gateway") @PactBroker(url = "...")
@State("...") void setUpState() { ... }
@TestTemplate @ExtendWith(PactVerificationInvocationContextProvider.class)
void verify(PactVerificationContext ctx) { ctx.verifyInteraction(); }
```

```bash
pact-broker can-i-deploy --pacticipant orderflow --version $GIT_SHA --to-environment production
```

### Gotchas checklist

- [ ] Contracts authored by the **consumer**, not the provider.
- [ ] `REMOTE` + **pinned version** in CI. Never `LOCAL` + `+`.
- [ ] Every non-deterministic field has a `bodyMatchers` entry.
- [ ] Base class stubs only what you do not own; prefer real seeded data.
- [ ] Auth headers are part of the contract.
- [ ] One contract per response *shape*, not per business case.
- [ ] The provider verification job is **required**, not nightly.
- [ ] Money crosses the wire as integer minor units, and the contract says so.
- [ ] Ask of the suite: has it ever gone red? If not, it is not connected.

---

## When would I use this at work?

**1. The week you split a monolith.**
The moment one deployable becomes two, every method call that used to be
compile-checked becomes a JSON string nobody checks. Contract tests are how you get
the compile check back. If you do this at the split rather than after the first
incident, it costs a day; after, it costs the incident plus the day.

**2. Deleting a field nobody admits to using.**
Every mature API has three of these. With a broker you run one query and get an
answer. Without one you send an email, wait two weeks, remove it anyway, and find out
from a customer. This is the single most quotable benefit in a planning meeting,
because it converts a scary change into a routine one.

**3. Onboarding a new consumer team.**
A contract file is executable documentation that cannot be out of date. "Here is the
contracts directory, PR the interaction you need" is a better onboarding than any
wiki page, and the PR review is where the API design conversation actually happens.

---

## Connected topics

**Prerequisites:**
- **58 — JUnit 5**: the extension model that Stub Runner and Pact both hook into.
- **59 — Mockito**: what to stub in a base class, and why every mock there is a piece
  of the provider you are not verifying.
- **60 — Spring test slices**: contract base classes are `@SpringBootTest`, so they
  pay full context cost; a `@MockitoBean` in one forks the context cache.
- **61 — Testcontainers**: the right way to set up provider state is real seeded rows,
  not mocks.
- **46 — ProblemDetail**: your error contract is an API surface and belongs in a
  contract file.
- **50 — N+1 detection**: the query-count assertion this doc puts in the contract base
  class, so the wire *and* the cost are both locked in.
- **54 — `@Transactional`**: the rollback path the declined-payment contract exercises.

**This unlocks:**
- **63 — Mutation testing**: contract tests verify the wire; PIT verifies whether your
  provider unit tests actually assert anything about the logic behind it. The two
  answer different questions and you need both.
- **64 — Property-based testing**: a contract fixes one example of a shape; a property
  states an invariant across all of them.
- **65 — GATE, load testing**: the endpoints you just contracted (`GET /products`,
  `GET /orders/{id}`, `POST /orders`) are the exact three the load profile hits. A
  contract keeps them *correct* under change; the gate measures them under load.
- **113 — Kafka consumer groups**: message contracts are the same idea with a
  different transport; Spring Cloud Contract has a messaging DSL for exactly this.
- **116 — Idempotency**: the idempotency key is part of the request contract.
- **119 — Tracing**: propagation headers are part of the contract too, and are
  routinely forgotten in one.
- **127 — `javax`→`jakarta` migration**: contract tests are what make a large
  framework migration provable rather than hopeful.

---

*Java baseline 21; runtime JDK 25. Spring Boot 4.1 / Framework 7.0 with the matching
Spring Cloud release train. Nothing in this topic depends on a Java 21→25 language
difference. The one thing that does change with your stack is version alignment
between Boot and the Spring Cloud release train — check it with
`mvn dependency:tree -Dincludes=org.springframework.cloud` rather than trusting any
version number written in a document, including this one.*
