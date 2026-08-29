# 59 — Mockito — Stubbing vs Verification, and When Mocking Is the Wrong Tool

## Phase: 6 — Testing
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: test doubles for `orderflow`'s genuinely external collaborator — the `PaymentGateway` from Topic 39 — and a deliberate demonstration of why mocking `OrderRepository` produces a test that passes while production breaks.

---

## Mastery line (from the master plan)

> You can articulate when a mock encodes an implementation detail and makes
> refactoring impossible, and you prefer a real object or a fake for anything you own.

## Mid → Senior (from the master plan)

> "We mock the repository" → "stubbing controls inputs; verification asserts on
> interactions and couples the test to the implementation. Every `verify` is a design
> assertion. I mock what I don't own and can't run — and use Testcontainers for the
> database rather than mocking a repository whose real behaviour (flush timing,
> constraint violations) is exactly what I need to test."

---

## ELI5 anchor

You are filming a scene where the hero jumps off a building.

You do not throw the actor off a building. You use a **stunt double** — someone who
looks close enough from the camera's angle and will do the dangerous thing on cue.

A **mock** is a stunt double for an object.

But notice the two very different things a director can do with a stunt double:

1. **"When the scene calls for a jump, do this."** — you are controlling what the
   double *gives back*. That is **stubbing**.
2. **"Afterwards, confirm the double jumped exactly once, from the third floor."** —
   you are checking what *happened to* the double. That is **verification**.

Stubbing sets up the world your code lives in. Verification asserts on your code's
behaviour toward the world. They feel similar. They have opposite consequences for
how hard your code is to change later, and confusing them is the most expensive
mistake in this topic.

And here is the part that has no film equivalent: **in Java you cannot swap the
stunt double in secretly.** There is no "replace the actor at the door" mechanism.
The scene has to have been written so that whoever plays the hero is *handed in* at
the start. That is dependency injection, and it is why this topic and Topic 39 are
really the same topic.

---

## The bridge from what you know

### The one that transfers: `jest.fn()` and `jest.spyOn`

```ts
// TypeScript
const gateway = { authorize: jest.fn().mockResolvedValue({ status: 'AUTHORIZED' }) };
const service = new OrderService(gateway, repo);

await service.place(command);

expect(gateway.authorize).toHaveBeenCalledWith(expect.objectContaining({ amount: 4797 }));
```

```java
// Java
PaymentGateway gateway = mock(PaymentGateway.class);
when(gateway.authorize(any())).thenReturn(new AuthorizationResult(AUTHORIZED, "ref-1"));

OrderService service = new OrderService(gateway, repo);
service.place(command);

verify(gateway).authorize(argThat(req -> req.amountMinor() == 4797L));
```

**Verdict: PARTIAL, and honestly quite close.** `jest.fn()` → `mock(...)`.
`mockResolvedValue` → `thenReturn`. `toHaveBeenCalledWith` → `verify`.
`jest.spyOn(realObject, 'method')` → `spy(realObject)` plus `doReturn(...).when(...)`.

If you stop reading here you will write working Mockito. You will also write the
tests that Trap 1 and Trap 2 describe, because the framework similarity hides a
structural difference.

### THE BIG ONE: there is no module mocking. None.

This is the centrepiece of the topic. Read it twice.

```ts
// TypeScript. This works, and you have relied on it for years.
jest.mock('./paymentGateway');
import { authorize } from './paymentGateway';

(authorize as jest.Mock).mockResolvedValue({ status: 'AUTHORIZED' });
```

What happened there: Jest sits between your module and the module registry. When
your code under test does `import { authorize } from './paymentGateway'`, Jest hands
it a fake module instead of the real one. **Your production code did not need to be
written any particular way.** It can `import` whatever it likes, at the top of the
file, hard-coded, and Jest can still intercept it.

**Java has no equivalent. There is no mechanism to intercept an `import`.**

An `import` in Java is not a runtime operation. It is a *compile-time name
shortcut*. By the time your code runs, `import com.orderflow.payments.StripeGateway;`
has left no trace — the bytecode contains a direct reference to the class, resolved
by the class loader (Topic 67). There is no registry to swap, no module cache to
poison, no interception point.

So this production code:

```java
public class OrderService {

    public Order place(PlaceOrderCommand command) {
        StripeGateway gateway = new StripeGateway(apiKey);   // <-- hard-coded
        AuthorizationResult result = gateway.authorize(...);
        ...
    }
}
```

is, for practical purposes, **untestable**. Not "hard to test". Untestable. There is
no flag, no framework, no clever trick that lets a unit test stop that `new
StripeGateway(...)` from making a real HTTPS call to Stripe. (Mockito's
`mockConstruction` exists and can intercept it — it is a last-resort tool for legacy
code you cannot change, and reaching for it in new code is an admission that the
design is wrong.)

### The consequence: testability is a design property, enforced by injection

Because you cannot intercept the import, the **only** way to substitute a
collaborator is for the collaborator to arrive from outside. Which means:

```java
public class OrderService {

    private final PaymentGateway gateway;      // an INTERFACE
    private final OrderRepository orders;

    public OrderService(PaymentGateway gateway, OrderRepository orders) {   // handed in
        this.gateway = gateway;
        this.orders = orders;
    }
}
```

Now a test can write `new OrderService(mock(PaymentGateway.class), repo)` and there
is nothing left to intercept, because the substitution point is a parameter.

**This is why Topic 39 said constructor injection is not a style preference.** In
TypeScript, constructor injection is one option among several, and you can always
fall back on `jest.mock`. In Java it is the mechanism. There is no fallback.

Restate it as a rule you can use in code review:

> **In Java, "is this testable?" is answered by reading the constructor, not by
> reading the test framework docs.**

A class with a hard-coded `new` of anything that touches the network, the clock, the
filesystem or randomness has just declared itself untestable, and no amount of
Mockito skill recovers it.

### The full comparison table

| Jest | Mockito | Verdict |
|---|---|---|
| `jest.fn()` | `mock(Interface.class)` | **PARTIAL — close** |
| `jest.fn().mockReturnValue(x)` | `when(m.call()).thenReturn(x)` | **PARTIAL — close** |
| `jest.fn().mockRejectedValue(e)` | `when(m.call()).thenThrow(e)` | **PARTIAL — close** |
| `jest.spyOn(obj, 'm')` | `spy(realObject)` + `doReturn(x).when(s).m()` | **PARTIAL** — Java's spy wraps a real object; call-through is the default |
| `expect(fn).toHaveBeenCalledWith(...)` | `verify(m).call(...)` | **HONEST ANALOGUE** |
| `expect(fn).toHaveBeenCalledTimes(2)` | `verify(m, times(2)).call(...)` | **HONEST ANALOGUE** |
| `mockClear` / `mockReset` | `reset(m)` — and *avoid it*; JUnit's new instance per method already resets everything | **PARTIAL** |
| **`jest.mock('./module')`** | **nothing** | **NO ANALOGUE — the whole point of this topic** |
| `jest.mock` of a Node builtin (`fs`, `http`) | nothing; you inject an abstraction instead | **NO ANALOGUE** |
| `jest.useFakeTimers()` | inject a `java.time.Clock` | **NO ANALOGUE** — again solved by injection, not by the framework |

That last row generalises. Every place Jest solves a problem by intercepting the
module system, Java solves it by making the dependency explicit. Time is the
clearest example: there is no `useFakeTimers`, so you inject `Clock.fixed(...)` and
your production code calls `clock.instant()` rather than `Instant.now()`.

---

## What is this?

**Mockito** creates, at run time, an object that implements an interface (or extends
a class) where every method does nothing useful until you tell it otherwise.

Mechanically: Mockito uses **ByteBuddy** to generate a subclass or a proxy class in
memory, records every call made to it, and consults a list of stubs to decide what to
return. That is it. It is not magic and it is not instrumentation of your code — it
is a generated object that you must arrange to be *used*.

Two crucial consequences of "it is a generated subclass":

1. **A mock is only used if someone hands it to the code under test.** See the
   bridge section. This is not a Mockito limitation; it is Java.
2. **Mocking works by overriding methods.** `final` classes, `final` methods,
   `static` methods and constructors are not overridable by ordinary subclassing.
   Mockito's *inline mock maker* (which uses instrumentation rather than
   subclassing) removes most of that restriction — but the fact that this needed a
   different mechanism tells you it is unusual territory.

> **Which mock maker are you on?** Since Mockito 5 the inline mock maker is the
> default in `mockito-core`, which means `final` classes and methods are mockable
> without an extra dependency, and the separate `mockito-inline` artifact is no
> longer needed. Do not take the version from me — check what your Boot BOM manages:
> ```bash
> mvn dependency:tree -Dincludes=org.mockito:*
> ```
> If you see `mockito-inline` listed explicitly in a `pom.xml`, that is a leftover
> from an older setup and can usually be removed.

### The two halves of Mockito, and why you must keep them separate

This distinction is the intellectual content of the topic. Everything else is
syntax.

| | **Stubbing** | **Verification** |
|---|---|---|
| Syntax | `when(x).thenReturn(y)` | `verify(mock).method(args)` |
| What it does | Controls what the collaborator *gives* your code | Asserts what your code *did to* the collaborator |
| Kind of statement | "the world looks like this" | "my code must call this, exactly this way" |
| If the implementation is refactored | usually survives | **usually breaks** |
| Analogous to | test setup / arrange | an assertion / assert |
| Cost when overused | slow tests, unrealistic scenarios | **a suite that blocks refactoring** |

**Stubbing is arrangement. Verification is assertion.**

The rule that follows:

> **Prefer asserting on the outcome. Verify only when the interaction *is* the
> outcome.**

If your code's job is to compute an order total, assert on the total. If your code's
job is *to send a payment authorization to the gateway*, then the call to the gateway
is the observable outcome and `verify` is correct — there is nothing else to look at.

A useful test: ask "if I rewrote the body of this method entirely but the system
behaved identically, would this assertion still pass?" If not, you have written a
test of the implementation, not of the behaviour.

---

## Why does it matter?

**1. Over-mocked suites are the most common cause of "we can't refactor this".**
Not architecture, not language. A 3,000-test suite where half the tests `verify`
internal calls means every structural change breaks hundreds of green tests, so the
team stops making structural changes. The suite that was supposed to enable change
prevents it.

**2. Mocking the thing whose behaviour you needed produces confident wrong tests.**
`when(orderRepository.save(any())).thenReturn(order)` says "saving always works and
returns what I gave it". Reality: the save might violate a unique constraint, might
be deferred to flush time, might assign an ID your code then depends on, might
deadlock. Your mock asserts that none of those things exist. Topic 61 exists because
of this exact failure.

**3. Java's lack of module mocking makes this a design conversation, not a testing
one.** In TypeScript you can defer the design question forever, because `jest.mock`
bails you out. In Java, an untestable class stays untestable. So "how do I test
this?" and "is this designed well?" become the same question — which is a gift, if
you notice it.

---

## Syntax breakdown

### Wiring: `MockitoExtension`, `@Mock`, `@InjectMocks`, `@Spy`, `@Captor`

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock  PaymentGateway paymentGateway;
    @Mock  InventoryService inventoryService;

    @InjectMocks OrderService orderService;
}
```

| Bit | What it means |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | The JUnit extension from Topic 58. It creates the `@Mock` fields before each test and validates stub usage after. Without it, `@Mock` fields stay `null`. |
| `@Mock` | Field is replaced with a generated mock. Fresh per test method — because JUnit gives a fresh instance per method (Topic 58). |
| `@InjectMocks` | Mockito constructs the object and pushes the mocks into it by reflection. **Read the warning below before using this.** |
| `@Spy` | Wraps a *real* object. Calls go through to the real implementation unless stubbed. |
| `@Captor ArgumentCaptor<X> captor` | Declares a captor without the generics boilerplate. |

**The `@InjectMocks` warning — a genuine senior/mid divider.**

`@InjectMocks` picks a constructor by reflection and matches mocks to parameters by
type, then by name. If it cannot find a match, **it injects `null` and says
nothing.** You then get a `NullPointerException` from deep inside your service and
spend twenty minutes wondering why.

It also *hides* the constructor. When someone adds a seventh dependency to
`OrderService`, the test does not change and nothing draws attention to the fact
that the class now has seven dependencies.

**Prefer the plain constructor call:**

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock PaymentGateway paymentGateway;
    @Mock InventoryService inventoryService;
    @Mock WalletService walletService;

    OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(paymentGateway, inventoryService, walletService);
    }
}
```

Three lines longer, and now a new dependency is a compile error in the test — which
is exactly the feedback Topic 39 promised constructor injection would give you. It
also means the test documents the real construction of the object.

### Stubbing

```java
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;

// value
when(paymentGateway.authorize(any(AuthorizationRequest.class)))
    .thenReturn(new AuthorizationResult(AUTHORIZED, "auth-1001"));

// exception
when(paymentGateway.authorize(any()))
    .thenThrow(new GatewayTimeoutException("upstream timeout"));

// different result on consecutive calls — for retry tests
when(paymentGateway.authorize(any()))
    .thenThrow(new GatewayTimeoutException("first attempt"))
    .thenReturn(new AuthorizationResult(AUTHORIZED, "auth-1001"));

// compute from the argument
when(walletService.debit(any(), anyLong()))
    .thenAnswer(inv -> new WalletSnapshot(inv.getArgument(0), 50_000L - inv.getArgument(1, Long.class)));

// void methods, and anything on a @Spy: doXxx().when() form
doThrow(new InsufficientStockException("SKU-1001", 5, 2))
    .when(inventoryService).reserve("SKU-1001", 5);

doNothing().when(auditLog).record(any());
```

| Bit | What it means |
|---|---|
| `when(x.call()).thenReturn(y)` | The normal form. Note it **calls the method** to record which invocation you are stubbing. |
| `doReturn(y).when(x).call()` | The alternative form that does **not** call the real method. Required on a `@Spy` and on `void` methods. Trap 5. |
| `any()`, `anyLong()`, `eq("SKU-1001")` | **Argument matchers.** If you use one matcher in a call, **every** argument must be a matcher — mixing a raw value with a matcher throws `InvalidUseOfMatchersException`. Wrap the raw value in `eq(...)`. |
| `thenAnswer(inv -> ...)` | Compute a response from the arguments. Powerful and easily abused: an `Answer` with logic in it is a hand-written fake wearing a mock costume. If you need one, write a real fake class instead. |
| `thenThrow(...).thenReturn(...)` | Consecutive stubbing. The idiomatic way to test retry logic. |

### Verification

```java
verify(paymentGateway).authorize(any());                      // exactly once (the default)
verify(paymentGateway, times(2)).authorize(any());
verify(paymentGateway, never()).refund(any());
verify(paymentGateway, atLeastOnce()).authorize(any());
verify(paymentGateway, timeout(500)).authorize(any());        // async; use sparingly

verifyNoInteractions(refundService);
verifyNoMoreInteractions(paymentGateway);                     // see Trap 1 before using

InOrder inOrder = inOrder(inventoryService, walletService, paymentGateway);
inOrder.verify(inventoryService).reserve("SKU-1001", 2);
inOrder.verify(walletService).debit(CUSTOMER_ID, 4797L);
inOrder.verify(paymentGateway).authorize(any());
```

| Bit | What it means |
|---|---|
| `verify(m).call()` | Asserts the call happened exactly once with those arguments. |
| `never()` | Asserts absence. Often the *most* valuable verification — "we must not charge twice". |
| `verifyNoMoreInteractions` | Fails if anything at all was called that you did not verify. Extremely brittle; see Trap 1. |
| `InOrder` | Asserts ordering across mocks. Legitimate when ordering is a real requirement (reserve stock *before* debiting the wallet). |

### `ArgumentCaptor` — inspecting what was passed

```java
@Captor ArgumentCaptor<AuthorizationRequest> requestCaptor;

@Test
void authorizesTheDiscountedTotal() {
    orderService.place(command);

    verify(paymentGateway).authorize(requestCaptor.capture());

    AuthorizationRequest sent = requestCaptor.getValue();
    assertThat(sent.amountMinor()).isEqualTo(4797L);
    assertThat(sent.currency()).isEqualTo("GBP");
    assertThat(sent.idempotencyKey()).isNotBlank();
}
```

| Bit | What it means |
|---|---|
| `captor.capture()` | Used *inside* `verify` as an argument matcher. It records the value rather than matching it. |
| `captor.getValue()` | The last captured value. `getAllValues()` returns every call's argument, in order. |
| Why not `argThat(...)`? | `argThat` gives you "no matching invocation" on failure, which tells you nothing about what *was* passed. A captor lets you assert on the object with AssertJ and get a real diff. **Prefer the captor when the assertion is non-trivial.** |

### Strictness

```java
@ExtendWith(MockitoExtension.class)                       // STRICT_STUBS by default
@MockitoSettings(strictness = Strictness.LENIENT)         // do not do this
class SomeTest { }

lenient().when(clock.instant()).thenReturn(FIXED_INSTANT);   // per-stub escape hatch
```

`MockitoExtension` defaults to `STRICT_STUBS`, which fails the test if you declared a
stub that was never used, and fails immediately on an argument mismatch instead of
silently returning `null`. **This is a feature.** See Trap 3.

---

## Example 1 — minimal

`orderflow`'s `PaymentGateway` is the one collaborator that is genuinely external:
it makes an HTTPS call to a third party, costs money, and is not something you can
run in your test. It is the canonical thing to mock.

```java
package com.orderflow.payments;

/** Topic 39: an interface with two implementations chosen by @Qualifier. */
public interface PaymentGateway {
    AuthorizationResult authorize(AuthorizationRequest request);
    void refund(String authorizationRef, long amountMinor);
}
```

```java
package com.orderflow.payments;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class PaymentAttemptTest {

    @Mock PaymentGateway gateway;

    @Test
    void returnsTheAuthorizationReferenceOnSuccess() {
        when(gateway.authorize(any()))
            .thenReturn(new AuthorizationResult(Status.AUTHORIZED, "auth-1001"));

        PaymentAttempt attempt = new PaymentAttempt(gateway);

        assertThat(attempt.charge(4797L, "GBP")).isEqualTo("auth-1001");
    }

    @Test
    void translatesAGatewayTimeoutIntoADomainException() {
        when(gateway.authorize(any()))
            .thenThrow(new GatewayTimeoutException("upstream timeout"));

        PaymentAttempt attempt = new PaymentAttempt(gateway);

        assertThatThrownBy(() -> attempt.charge(4797L, "GBP"))
            .isInstanceOf(PaymentUnavailableException.class);
    }
}
```

**Why this is a good use of a mock, in one sentence:** the gateway is something you
do not own, cannot run, and whose failure modes you need to simulate — and the second
test simulates a failure that is genuinely hard to produce any other way.

Note what is *not* here: no `verify`. The first test asserts on the returned
reference; the second asserts on the thrown type. Both assert on the outcome. There
is nothing an implementation refactor could break except the behaviour itself.

---

## Example 2 — production scenario (on the project spine)

`orderflow`'s order placement, from Topic 54. It must: reserve inventory, debit the
wallet, authorize the payment, persist the order, and publish an event — atomically.

### First, the version that ships and then lets a bug through

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepository;     // <-- we own this. Red flag.
    @Mock InventoryService inventoryService;   // <-- we own this too.
    @Mock WalletService walletService;         // <-- and this.
    @Mock PaymentGateway paymentGateway;       // <-- external. Legitimate.

    @InjectMocks OrderService orderService;

    @Test
    void placesAnOrder() {
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(paymentGateway.authorize(any()))
            .thenReturn(new AuthorizationResult(AUTHORIZED, "auth-1001"));

        orderService.place(command);

        verify(inventoryService).reserve("SKU-1001", 2);
        verify(walletService).debit(CUSTOMER_ID, 4797L);
        verify(paymentGateway).authorize(any());
        verify(orderRepository).save(any(Order.class));
        verifyNoMoreInteractions(orderRepository, inventoryService, walletService);
    }
}
```

This test is green, fast, and reads well in a review. It is also close to worthless,
and it will actively obstruct you. Here is everything wrong with it:

- `when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0))` asserts a
  falsehood. The real `save` returns a **managed** entity with a **generated ID**,
  and under Hibernate the INSERT may not even have run yet (Topic 48: dirty checking
  and flush timing). The mock says "save is an identity function". Production says
  otherwise.
- The unique constraint on `(customer_id, idempotency_key)` that prevents double
  charging does not exist in the mock. A test that mocks the repository can never
  discover a constraint violation.
- Every `verify` is a statement about *how* `place` is implemented. Change the order
  of operations, extract a collaborator, batch the two writes — the behaviour is
  identical, the test goes red.
- `verifyNoMoreInteractions` means adding any call at all — an audit log entry, a
  metric — breaks the test. That is not a safety net; it is a tripwire in a corridor.
- It proves nothing about the transaction. `@Transactional` is a proxy (Topic 40);
  the proxy does not exist here because there is no Spring context. The test exercises
  the method *without* the transactional semantics that are the whole point of it.

### The version that is worth having

Split it into the two tests it was always trying to be.

**Test A — the decision logic, with a fake, no Spring, microseconds.**

```java
package com.orderflow.orders;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("OrderService — placement decisions")
class OrderPlacementDecisionTest {

    /** A real, in-memory implementation. Not a mock. We own the interface,
     *  so we can write an honest implementation of it. */
    private final InMemoryOrderRepository orders = new InMemoryOrderRepository();
    private final InMemoryInventory inventory = new InMemoryInventory();
    private final InMemoryWallets wallets = new InMemoryWallets();

    @Mock PaymentGateway paymentGateway;          // the only true external
    @Captor ArgumentCaptor<AuthorizationRequest> authRequest;

    private OrderService orderService;

    @BeforeEach
    void setUp() {
        inventory.stock("SKU-1001", 12);
        wallets.credit(CUSTOMER_ID, 50_000L);
        orderService = new OrderService(orders, inventory, wallets, paymentGateway,
                                        Clock.fixed(FIXED_INSTANT, ZoneOffset.UTC));
    }

    @Test
    @DisplayName("authorizes the discounted total, not the gross total")
    void authorizesDiscountedTotal() {
        when(paymentGateway.authorize(any()))
            .thenReturn(new AuthorizationResult(AUTHORIZED, "auth-1001"));

        orderService.place(command(CUSTOMER_ID, "SKU-1001", 10));   // 10 units -> 5% off

        // The interaction with the gateway IS the outcome here: the amount we
        // send to a third party is the observable behaviour. Verify is correct.
        verify(paymentGateway).authorize(authRequest.capture());
        assertThat(authRequest.getValue().amountMinor()).isEqualTo(18_991L);
        assertThat(authRequest.getValue().idempotencyKey()).isNotBlank();
    }

    @Test
    @DisplayName("does not touch the wallet or the gateway when stock is short")
    void shortStockStopsBeforeAnyCharge() {
        assertThatThrownBy(() -> orderService.place(command(CUSTOMER_ID, "SKU-1001", 99)))
            .isInstanceOf(InsufficientStockException.class);

        // 'never' is the highest-value verification in the codebase:
        // it encodes 'we must not charge for an order we cannot fulfil'.
        verify(paymentGateway, never()).authorize(any());
        assertThat(wallets.balanceOf(CUSTOMER_ID)).isEqualTo(50_000L);
        assertThat(orders.findAll()).isEmpty();
    }

    @Test
    @DisplayName("leaves the wallet untouched when the gateway declines")
    void gatewayDeclineLeavesWalletUntouched() {
        when(paymentGateway.authorize(any()))
            .thenReturn(new AuthorizationResult(DECLINED, null));

        assertThatThrownBy(() -> orderService.place(command(CUSTOMER_ID, "SKU-1001", 2)))
            .isInstanceOf(PaymentDeclinedException.class);

        assertThat(wallets.balanceOf(CUSTOMER_ID)).isEqualTo(50_000L);
        assertThat(inventory.availableOf("SKU-1001")).isEqualTo(12);
    }
}
```

What changed, and why each change matters:

| Change | Why |
|---|---|
| Repositories are **fakes**, not mocks | We own them. A fake is a real implementation, so assertions are on *state* (`wallets.balanceOf(...)`) rather than on interactions. State assertions survive refactoring; interaction assertions do not. |
| Only `PaymentGateway` is mocked | It is external, costs money, and its failure modes need simulating. |
| `Clock` is injected | Java's answer to `jest.useFakeTimers()`. There is no other answer. |
| `verify` appears only twice | Once where the interaction *is* the outcome (what amount we send to a third party), once as `never()` (we must not charge). Both are behaviour, not implementation. |
| `verifyNoMoreInteractions` is gone | Adding a metric or an audit log no longer breaks the test. |
| `@InjectMocks` is gone | The constructor is visible; a new dependency is a compile error here. |

**Test B — the persistence and transaction behaviour, against a real Postgres.**

Everything the mocks were lying about goes here, in Topic 61's territory:

```java
// Sketch — the full version is Topic 61. Named here so you can see the split.
@SpringBootTest
@Testcontainers
class OrderPlacementPersistenceIT {

    @Test
    void aDuplicateIdempotencyKeyIsRejectedByTheDatabase() { ... }   // real unique constraint

    @Test
    void aConcurrentSecondOrderCannotOversellTheLastUnit() { ... }   // real row locking

    @Test
    void aFailedPaymentRollsBackTheInventoryReservation() { ... }    // real transaction

    @Test
    void placingAnOrderIssuesExactlyThreeQueries() { ... }           // Topic 50's N+1 fix
}
```

**The split is the lesson.** Test A runs in microseconds and covers decisions.
Test B runs in seconds and covers everything only a real database can tell you.
Neither one can do the other's job, and mocking the repository is an attempt to
make Test A do Test B's job — which is why it fails silently.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — over-verification: `verify(repo).save(any())`

**Wrong:**

```java
@Test
void placesTheOrder() {
    orderService.place(command);

    verify(orderRepository).save(any(Order.class));
    verify(orderLineRepository, times(3)).save(any(OrderLine.class));
    verify(auditLog).record(any());
    verifyNoMoreInteractions(orderRepository, orderLineRepository, auditLog);
}
```

**Exact symptom:** eight months later, a colleague replaces three individual
`orderLineRepository.save()` calls with one `saveAll()` — a genuine improvement that
turns three round trips into one batch (Topic 53). The system behaves identically.
The database ends up in exactly the same state. **Fourteen tests go red.** The PR
sits for three days. The reviewer eventually approves "just update the mocks", and
in the bulk update one test that was checking something real — `verify(paymentGateway,
never()).authorize(any())` on the insufficient-stock path — gets deleted along with
the noise. Six weeks later `orderflow` charges customers for orders it cannot fill.

**Root cause:** every `verify` is an assertion about *how* the code is written, not
about what it does. `saveAll` versus three `save` calls is an implementation detail;
the test made it a contract. `verifyNoMoreInteractions` amplifies this: it asserts
that the implementation does *nothing else*, forever.

**Fix — three rules:**

1. **Assert on state or on the return value wherever you can.** With a fake
   repository: `assertThat(orders.findById(id)).isPresent()`. With a real database:
   query it.
2. **Verify only when the interaction is the observable outcome.** Sending a request
   to a third party. Publishing an event. Not calling something (`never()`).
3. **Delete `verifyNoMoreInteractions` unless you can name the specific extra call
   that would be a bug.** "Something else got called" is not a defect.

A code-review question that catches this instantly: *"if I rewrote the body of this
method and the observable behaviour was identical, would this test still pass?"*

---

### Trap 2 — mocking a repository whose real behaviour is the point

**Wrong:**

```java
@Test
void aRetriedOrderIsNotChargedTwice() {
    when(orderRepository.findByIdempotencyKey("key-1"))
        .thenReturn(Optional.empty())          // first call: not seen before
        .thenReturn(Optional.of(existingOrder)); // second call: seen

    orderService.place(command);
    orderService.place(command);               // the retry

    verify(paymentGateway, times(1)).authorize(any());
}
```

**Exact symptom:** green forever. In production, two concurrent retries from the same
customer both run `findByIdempotencyKey`, both get empty, both proceed, and the
customer is charged twice. The incident report says "race condition in idempotency
check". The test suite had a test named `aRetriedOrderIsNotChargedTwice`.

**Root cause:** the mock simulated a *sequence* (first empty, then present). The real
system has *concurrency*, and the actual protection is a database unique constraint
on `(customer_id, idempotency_key)` that throws
`DataIntegrityViolationException` on the second insert. **The mock replaced the very
mechanism the test claimed to be testing.**

This generalises. Everything on this list is invisible to a mocked repository:

| Real repository behaviour | What the mock says instead |
|---|---|
| Unique / foreign-key / check constraint violations | never happens |
| Generated IDs (`IDENTITY` / `SEQUENCE`, Topic 53) | whatever you stubbed |
| Flush timing — the INSERT happens at commit, not at `save()` (Topic 48) | happens immediately |
| Dirty checking writing an UPDATE with no `save()` call at all (Topic 48) | no writes happen |
| `LazyInitializationException` outside the session (Topic 49) | lazy loading always works |
| N+1 query explosion (Topic 50) | one call is one call |
| Optimistic lock failure on `@Version` (Topic 52) | never happens |
| Row locking, `SKIP LOCKED`, deadlocks | no such thing as concurrency |
| Rollback on a `RuntimeException`, commit on a checked one (Topic 54) | no transaction exists |

**Fix:** move the test to a real database. That is Topic 61. Concretely, the
idempotency test becomes:

```java
@Test
void aConcurrentRetryIsRejectedByTheUniqueConstraint() {
    // two threads, one real Postgres, one unique index. The database is the mechanism.
    ...
    assertThat(paymentAttempts).hasSize(1);
}
```

**The rule:** *mock what you do not own and cannot run. Use the real thing for what
you own.* Your repository is both owned and runnable — Docker makes Postgres a
five-second dependency.

---

### Trap 3 — `UnnecessaryStubbingException`, and why the strictness is a gift

**Symptom:** you add a stub, the test passes, someone else's refactor lands, and now
your test fails with:

```
org.mockito.exceptions.misusing.UnnecessaryStubbingException:
Unnecessary stubbings detected.
Clean & maintainable test code requires zero unnecessary code.
  1. -> at com.orderflow.orders.OrderServiceTest.setUp(OrderServiceTest.java:41)
```

*(shape of the Mockito strict-stubs failure, shown as an illustration of the format —
not captured output)*

The first instinct is to reach for `@MockitoSettings(strictness = Strictness.LENIENT)`
and move on. **Do not.**

**Root cause:** you told Mockito "when the code calls `inventoryService.available()`,
return 12". The code no longer calls it. One of two things is true:

- **The stub is dead** — the code path changed and the setup was left behind. Delete
  the stub. Free cleanup.
- **The code path changed and you did not notice** — someone removed the availability
  check. Your test still passes because the assertion did not depend on it. Mockito
  is telling you your test is now testing less than it used to.

That second case is the one that matters, and there is no other tool that catches
it. A lenient mock returns `null` (or `0`, or an empty collection) for anything
unstubbed and the test sails on. Strictness turns a silent semantic drift into a
build failure.

The related strictness win: with `STRICT_STUBS`, if your code calls
`gateway.authorize(requestWithAmount(4797))` but you stubbed
`gateway.authorize(requestWithAmount(4800))`, you get **"Argument mismatch"** naming
both, instead of a `null` return and a `NullPointerException` eighteen frames deeper.

**Fix, in order:**

1. Delete the stub if it is genuinely dead.
2. If the stub is needed by *some* tests in the class but not all, move it out of
   `@BeforeEach` and into the tests that need it. This is Topic 58's Trap 5 wearing
   a different hat.
3. Only if the stub is legitimately shared and legitimately unused in one case, use
   `lenient().when(...)` on **that one stub**. Never class-wide leniency.

---

### Trap 4 — mocking a third party, then discovering the real one behaves differently

**Wrong:**

```java
when(stripeGateway.authorize(any()))
    .thenReturn(new AuthorizationResult(AUTHORIZED, "auth-1001"));
```

**Exact symptom:** every test passes. In production, the real gateway returns HTTP
200 with a body containing `{"status": "requires_action"}` for 3-D Secure — a third
state your enum does not have. Your code takes the `else` branch and marks the order
`PAID`. You have shipped goods for money you never captured. The finance
reconciliation finds it three weeks later.

**Root cause:** a mock encodes *your belief* about what the collaborator does. If
your belief is wrong, the mock is wrong in exactly the same way, and it is wrong
consistently across every test — so the suite is maximally confident about a
falsehood. Mocking an external system converts "I might be wrong about their API"
into "my tests agree with me that I am right".

**Fix — three layers, in increasing cost:**

1. **A contract test (Topic 62).** The mock and the provider's verification become
   the same artefact, so a provider change fails a build rather than production.
   This is the only real answer.
2. **A stubbed HTTP server rather than a mocked interface.** WireMock or Spring's
   `MockRestServiceServer` replays *recorded real responses*, so at least your JSON
   parsing, error mapping and timeout handling are exercised. A mocked interface
   skips all three.
3. **A sandbox/staging integration test**, tagged and run nightly rather than on
   every commit.

At minimum, when you must mock an external system, make the mock's responses come
from **recorded real payloads** checked into the repository, not from your memory of
the docs.

---

### Trap 5 — `when()` on a spy calls the real method

**Wrong:**

```java
@Spy OrderRepository repository = new JdbcOrderRepository(dataSource);

@Test
void ... {
    when(repository.findById(42L)).thenReturn(Optional.of(order));   // <-- runs for real
}
```

**Exact symptom:** the test fails during *setup*, before any assertion, with a
`NullPointerException` from inside `JdbcOrderRepository` — or worse, it succeeds and
silently hits a real database, and the test is now slow and non-deterministic.

**Root cause:** `when(x.method())` works by **actually calling the method** and
letting Mockito intercept the recorded invocation. On a plain mock the method body is
empty, so calling it is harmless. On a **spy**, the method body is the real
implementation, so `when(...)` executes it.

**Fix:** use the `doReturn(...).when(spy).method()` form, which never invokes the
real method:

```java
doReturn(Optional.of(order)).when(repository).findById(42L);
```

The same form is required for `void` methods on any mock, because `when(x.voidCall())`
does not compile:

```java
doThrow(new InsufficientStockException("SKU-1001", 5, 2))
    .when(inventoryService).reserve("SKU-1001", 5);
```

**The deeper point:** a `@Spy` on your own class is usually a design smell. It means
"I want most of this object real but one method fake", which is almost always
"this class does two things and one of them should be a separate collaborator". Spies
are a legitimate tool for legacy code you cannot restructure yet. They are rarely
the right answer in code you are writing today.

---

## Hands-on proof

Every command is one **you** run. I have no JVM and no test runner; nothing below is
captured output.

### Proof 1 — a mock is a generated class, and only used if injected

`src/test/java/com/orderflow/MockNatureProofTest.java`:

```java
package com.orderflow;

import com.orderflow.payments.PaymentGateway;
import org.junit.jupiter.api.Test;

import static org.mockito.Mockito.mock;

class MockNatureProofTest {

    @Test
    void aMockIsAGeneratedClass() {
        PaymentGateway gateway = mock(PaymentGateway.class);
        System.out.println("class       : " + gateway.getClass().getName());
        System.out.println("interfaces  : " + java.util.Arrays.toString(
                                gateway.getClass().getInterfaces()));
        System.out.println("unstubbed   : " + gateway.authorize(null));
    }
}
```

```bash
mvn -Dtest=MockNatureProofTest test -Dsurefire.useFile=false
```

**What to look for:** the generated class name, and what an unstubbed method returns.

| What you see | What it means |
|---|---|
| A class name containing `$MockitoMock$` or `ByteBuddy` | Confirmed: Mockito generated a class at run time. It is a real object, and it had to be *handed* to your code to be used. |
| `unstubbed : null` | The default answer for an object-returning method is `null`. That is why an unstubbed call under lenient settings produces an NPE far from the cause — and why strict stubs are better. |
| A `MockitoException` about the mock maker | Rare; usually a JDK/agent restriction. Note the message and check the Mockito version with `mvn dependency:tree -Dincludes=org.mockito:*`. |

### Proof 2 — prove there is no module mocking

This is the most important five minutes in the topic. Write a class that hard-codes
its collaborator:

```java
package com.orderflow.payments;

public class HardCodedCharger {
    public String charge(long amountMinor) {
        StripeGateway gateway = new StripeGateway("sk_test_key");  // hard-coded
        return gateway.authorize(new AuthorizationRequest(amountMinor, "GBP", "k")).reference();
    }
}
```

Now try to write a test that makes `charge` return `"auth-1001"` without touching
`HardCodedCharger` or `StripeGateway`.

**What to look for:** you cannot. There is no `jest.mock`. Work through the options
and note why each fails:

| What you try | What happens |
|---|---|
| `mock(StripeGateway.class)` | Creates a mock. `HardCodedCharger` never sees it — it constructs its own. |
| `@Mock StripeGateway` + `@InjectMocks HardCodedCharger` | Injects nothing; there is no field or constructor parameter to inject into. |
| Change the classpath to a fake `StripeGateway` | Works, and is the closest Java gets. Requires two source trees and a build hack; nobody does this. |
| `mockConstruction(StripeGateway.class)` | Actually works — Mockito intercepts the constructor via instrumentation. Note that you needed a bytecode-instrumentation escape hatch to do what Jest does with one line, and that it only works inside a `try` block scoped around the call. |

**How to read the result:** the fact that only the last option works, and that it
requires instrumentation, *is* the proof. Then apply the fix — add a constructor
parameter — and watch the test become three trivial lines. That contrast is the
entire argument for constructor injection, and you have now produced it yourself
rather than been told it.

### Proof 3 — strict stubs catching a dead stub

Add an unused stub to a passing test:

```java
when(inventoryService.availableOf("SKU-9999")).thenReturn(0);   // nothing calls this
```

```bash
mvn -Dtest=OrderPlacementDecisionTest test
```

**What to look for:** the build outcome and the exception type.

| What you see | What it means |
|---|---|
| `UnnecessaryStubbingException` naming the file and line | Strict stubs are on. Working as designed. Delete the stub. |
| Test passes | Strictness has been lowered somewhere: a class-level `@MockitoSettings(strictness = LENIENT)`, a `lenient()` on the stub, or `MockitoAnnotations.openMocks(this)` instead of the extension. Find it and remove it. |
| `PotentialStubbingProblem` | You stubbed a call with different arguments than the code actually makes. The message names both — read it; it is usually the whole diagnosis. |

### Proof 4 — see the mismatch message that strictness buys you

Deliberately stub the wrong argument:

```java
when(paymentGateway.authorize(argThat(r -> r.amountMinor() == 9999L)))
    .thenReturn(new AuthorizationResult(AUTHORIZED, "auth-1001"));
```

when the code will actually authorize 4797.

**What to look for:** whether the failure names the argument.

| What you see | What it means |
|---|---|
| `PotentialStubbingProblem` naming the stubbed args and the actual args | Strict stubs. The failure is a diagnosis, not a mystery. |
| A `NullPointerException` deep inside `OrderService` | You are running lenient. The mock returned `null` and the NPE happened later. This is the exact experience strictness exists to remove. |

### Proof 5 — count how much of your suite is mock-driven

```bash
grep -rc "Mockito.verify\|verify(" src/test/java | sort -t: -k2 -rn | head -20
grep -rl "@Mock .*Repository" src/test/java | wc -l
```

**What to look for:** which test classes verify most, and how many mock a repository.

| What you see | What it means |
|---|---|
| A handful of classes with many `verify` calls | Look at those first when refactoring feels blocked. They are the ones asserting on implementation. |
| Many files matching `@Mock .*Repository` | Trap 2 at scale. Each is a candidate to become either a fake-based decision test or a Testcontainers test (Topic 61). |
| Almost no `verify` at all | Good sign, usually — a suite asserting on outcomes. Confirm it is not because there are no assertions at all (Topic 58, Trap 4). |

This is a five-second audit you can run on any Java codebase on your first day, and
it tells you a great deal about how the team thinks.

---

## Practice exercises

### 1 — Easy: stub versus verify, on the same behaviour

`orderflow` has a `RefundService.refund(orderId, amountMinor)` that:

- loads the payment for the order
- calls `paymentGateway.refund(authRef, amount)`
- marks the payment `REFUNDED`
- must refuse to refund more than was captured

**Part A.** Write a test using **only stubbing and outcome assertions** — no
`verify` anywhere — that proves an over-refund is rejected.

**Part B.** Write a test using `verify` that proves the gateway is called with the
correct reference and amount. Justify in a comment why `verify` is correct *here*
and was not needed in Part A.

**Part C.** Write a `verify(gateway, never())` test for the over-refund path.
Explain in one sentence why this is the single most valuable assertion in the file.

**Part D.** Now break it: change `RefundService` to call `gateway.refund` twice by
mistake. Which of your three tests catches it? If none do, add the one that does.

### 2 — Medium: replace mocks with a fake, combining earlier topics

You are given this test class (write it first if you like, it is deliberately bad):

```java
@ExtendWith(MockitoExtension.class)
class WalletServiceTest {
    @Mock WalletRepository walletRepository;
    @InjectMocks WalletService walletService;

    @Test
    void debitsTheWallet() {
        Wallet wallet = new Wallet(CUSTOMER_ID, 50_000L);
        when(walletRepository.findById(CUSTOMER_ID)).thenReturn(Optional.of(wallet));
        when(walletRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        walletService.debit(CUSTOMER_ID, 4797L);

        verify(walletRepository).save(argThat(w -> w.balanceMinor() == 45_203L));
    }
}
```

**Part A.** Write `InMemoryWalletRepository` — a real implementation of the
`WalletRepository` interface backed by a `Map`. Use Topic 13's rules: the key is a
`CustomerId` record, so `equals`/`hashCode` must be correct or lookups will silently
fail. Deliberately break `hashCode` first, watch a lookup miss, then fix it.

**Part B.** Rewrite the test against the fake, asserting on state
(`repository.findById(...).get().balanceMinor()`) rather than on `save`. Remove
`@InjectMocks`.

**Part C.** Add three tests the mocked version could not express: a debit that
exactly empties the wallet, two sequential debits, and a debit of a wallet that does
not exist (Topic 26 — does your API return `Optional` or throw? Justify it against
Topic 09).

**Part D.** Write down the list of things your fake **still** cannot tell you about
`WalletService` in production. There are at least four. Keep the list — Topic 61
answers every item on it.

### 3 — Hard: production simulation — the double-charge that mocks hid

This exercise reproduces Trap 2 end to end, and it advances `orderflow`.

**Part A.** Implement idempotent order placement: `PlaceOrderCommand` carries an
`idempotencyKey`, and placing the same key twice must produce one order and one
payment. Implement it *only in application code* first — a `findByIdempotencyKey`
check before proceeding.

**Part B.** Write the Mockito test from Trap 2 that "proves" it works
(`thenReturn(Optional.empty()).thenReturn(Optional.of(order))`). Confirm it is green.

**Part C.** Write a test that runs two placements **concurrently** using
`ExecutorService` and a `CountDownLatch`, still against mocks. Report honestly what
happens. Does it fail? Does it pass for the wrong reason? Can it detect the race at
all with a mocked repository? Write two sentences on why.

**Part D.** Add the real mechanism: a unique index on
`(customer_id, idempotency_key)` via Flyway, and catch
`DataIntegrityViolationException` to return the existing order. Now write the
concurrent test against a real Postgres (Topic 61 — read ahead if you must; a
`@Container PostgreSQLContainer` and `@ServiceConnection` is four lines). Assert that
exactly one `Payment` row exists.

**Part E.** The write-up. Answer in your own words:

- Which of your four tests would have caught the production double-charge?
- Which of them gave you *confidence* without giving you *coverage*?
- State a rule you would put in your team's testing guidelines that would have
  prevented writing Part B's test in the first place.

---

## Interview questions

### Q1 — "When is mocking the wrong tool?"

**Mid-level answer:** "When you're testing something simple, or when the mock setup
is longer than the test. Then you should use the real object."

**Senior answer:** "Three cases. First, when I own the thing and can run it — a
repository, a domain service, a value object. A fake or the real thing gives me state
assertions, which survive refactoring, instead of interaction assertions, which
don't. Second, when the collaborator's *real behaviour* is what the test is about:
mocking a repository hides constraint violations, flush timing, generated IDs and
row locking, which is usually exactly the class of bug I'm trying to catch. Third,
when the mock is encoding my belief about a third party — that just makes the suite
confidently agree with me; a contract test or a recorded-response HTTP stub is
honest about the assumption. The rule I use is: mock what I don't own and can't run.
`orderflow`'s payment gateway qualifies. Its own repositories do not, because
Postgres in Docker starts in a couple of seconds."

**What separates them:** the mid answer is about test aesthetics. The senior answer
names *classes of bug that mocking makes invisible*, and gives a rule with a
threshold ("can't run it") rather than a preference.

**Interviewer's follow-up:** "You said mocking a repository hides constraint
violations. Give me a concrete example where that reached production." They want the
idempotency/double-charge story or the equivalent. Have one.

---

### Q2 — "What's the difference between stubbing and verification?"

**Mid-level answer:** "`when(...).thenReturn(...)` sets up what a mock returns.
`verify(...)` checks it was called. You use both."

**Senior answer:** "Stubbing is arrangement — it controls the world my code runs in.
Verification is an assertion about my code's behaviour toward that world. The
practical difference is what happens under refactoring: a stub usually survives
because it describes the collaborator, while a `verify` usually breaks because it
describes my implementation. So every `verify` is a design assertion — I'm declaring
that *this specific call, in this shape,* is part of the contract. That's sometimes
true: sending a request to a payment gateway is genuinely the observable behaviour,
and `verify(gateway, never()).authorize(any())` on a failure path is one of the most
valuable assertions in the codebase. But `verify(repository).save(any())` is
asserting on plumbing, and it's what makes a suite block refactoring instead of
enabling it."

**What separates them:** framing verification as a *coupling decision*, and giving
both a case where it is right and a case where it is wrong.

**Interviewer's follow-up:** "How do you decide, in a specific test?" Good answer:
"If I rewrote the method body and the system behaved identically, would this
assertion still pass? If no, it's testing the implementation."

---

### Q3 — "There is no `jest.mock` in Java. What follows from that?"

**Mid-level answer:** "You use Mockito instead and inject the mock in the
constructor."

**Senior answer:** "It follows that testability is a property of the *design*, not of
the test framework. An `import` in Java is a compile-time name resolution — there's
nothing at run time to intercept — so the only substitution point is a parameter. A
class that does `new StripeGateway(key)` inside a method is untestable, full stop;
no framework recovers it. That's why constructor injection in Java is a hard
requirement rather than a style preference, and it's why 'can I test this?' and 'is
this designed well?' collapse into the same question. It also applies to things you
wouldn't think of as dependencies: there's no `useFakeTimers`, so time has to be an
injected `Clock`; there's no `jest.mock('fs')`, so file access has to be behind an
interface you own. Mockito does have `mockConstruction` and `mockStatic` as
instrumentation-based escape hatches, and I treat reaching for either in new code as
a signal that the design is wrong rather than as a solution."

**What separates them:** knowing *why* there is no equivalent (compile-time
resolution, not a runtime registry), and generalising it to time and I/O rather than
stopping at "inject your services".

**Interviewer's follow-up:** "How would you test code that calls `Instant.now()` in
three places?" They want `Clock` injected, and they are checking whether you say
"and I'd make that a constructor parameter" without being prompted.

---

### Q4 — "Your build fails with `UnnecessaryStubbingException`. What do you do?"

**Mid-level answer:** "Remove the stub, or set the strictness to lenient if lots of
tests are failing."

**Senior answer:** "First I read it as information rather than an obstacle. It means
a stub I declared is no longer exercised, which is one of two things: dead setup
left behind by a refactor, which I delete, or a code path that quietly disappeared,
which is a real regression my assertions didn't catch. The second case is the
valuable one and there is no other tool that finds it. If it's genuinely a
shared-setup problem — one stub needed by three of five tests — the fix is to move
the stub into the tests that need it, not to lower strictness. I'd only use
`lenient()` on that one individual stub, never class-wide. Turning off strictness
globally also loses the argument-mismatch diagnostic, which is the difference between
'PotentialStubbingProblem: you stubbed 4800, the code called 4797' and a null
pointer eighteen frames deeper."

**What separates them:** treating the exception as a *finding*, naming the
second-order loss (argument-mismatch diagnostics), and scoping the escape hatch to
one stub.

**Interviewer's follow-up:** "Someone's PR adds `@MockitoSettings(strictness =
LENIENT)` to a class. What's your review comment?" They want you to ask what problem
it solved and propose the narrower fix.

---

### Q5 — "Walk me through testing `orderflow`'s order placement."

**Mid-level answer:** "I'd mock the repositories, the inventory service, the wallet
service and the payment gateway, then verify each one was called and assert on the
returned order."

**Senior answer:** "I'd split it in two, because it's two different questions.
The decision logic — which discount tier applies, that we don't charge when stock is
short, that a declined payment leaves the wallet untouched — is a fast test with the
payment gateway mocked, a fixed `Clock` injected, and in-memory fakes for the things
I own so I can assert on state rather than on interactions. That runs in
microseconds and I'd have thirty of them.

Then everything the mocks would have lied about goes into an integration test
against a real Postgres via Testcontainers: the unique constraint that makes
idempotency work, the rollback on failure, the actual query count so the N+1 fix
can't regress, and the row locking under concurrent placement. Those are seconds
each and I'd have five.

The mistake I'd specifically avoid is a single mocked test that verifies
`repository.save` was called — that proves nothing about persistence and blocks
every future refactor. And it's worth saying that the mocked test can't exercise
`@Transactional` at all, because that's a Spring proxy that doesn't exist without a
context."

**What separates them:** the *split*, naming what each half can and cannot prove,
and volunteering the `@Transactional`-proxy point unprompted.

**Interviewer's follow-up:** "How many of those integration tests would you have, and
why not more?" They are checking whether you understand the cost curve — which is
Topic 60 and Topic 61.

---

## Mental model checkpoint

Reason these out without looking anything up.

1. Jest can intercept an import; Java cannot. Explain the underlying reason in terms
   of *when* the name is resolved in each language. Then say what would have to be
   true about the JVM for `jest.mock` to be possible.

2. Every `verify` is a design assertion. Construct a case where verification is the
   *only* correct assertion — where asserting on state is impossible in principle,
   not merely inconvenient.

3. A colleague says "fakes are just mocks you have to maintain". Steelman that
   position, then give the strongest counterargument. Which one do you actually
   believe for `orderflow`'s `WalletRepository`, and does your answer change for a
   third-party SDK?

4. Strict stubs fail on an unused stub. Argue that this is *too* strict — construct
   the case where it produces noise — and then say what you would change about the
   test rather than about the strictness.

5. You cannot mock `Instant.now()`. Java's answer is to inject a `Clock`. What does
   that tell you about the relationship between "hard to test" and "hidden
   dependency"? State it as a general principle and then apply it to three things
   other than time.

6. `@InjectMocks` silently injects `null` when it cannot match a parameter. Why did
   Mockito make that choice rather than failing loudly? Would you make the same
   choice designing the API today?

7. Mockito's `mockConstruction` can do what `jest.mock` does. If that is true, why is
   the argument "Java has no module mocking, therefore inject your dependencies"
   still correct? Answer without appealing to convention.

---

## Quick reference card

### Wiring

```java
@ExtendWith(MockitoExtension.class)   // strict stubs by default
@Mock   PaymentGateway gateway;       // a generated stand-in
@Spy    OrderMapper mapper = new OrderMapper();   // wraps a real object
@Captor ArgumentCaptor<AuthorizationRequest> captor;

// Prefer this over @InjectMocks:
service = new OrderService(gateway, inventory, wallets, Clock.fixed(T, UTC));
```

### Stubbing

```java
when(m.call(arg)).thenReturn(value);
when(m.call(any())).thenThrow(new GatewayTimeoutException("x"));
when(m.call(any())).thenThrow(FIRST).thenReturn(SECOND);   // consecutive
when(m.call(any())).thenAnswer(inv -> inv.getArgument(0));

doReturn(value).when(spy).call();        // spies: does NOT run the real method
doThrow(ex).when(m).voidMethod(arg);     // void methods
doNothing().when(m).voidMethod(any());
lenient().when(m.call()).thenReturn(v);  // exempt ONE stub from strictness
```

### Matchers — all or nothing

```java
verify(m).call(eq("SKU-1001"), anyInt());   // correct
verify(m).call("SKU-1001", anyInt());       // InvalidUseOfMatchersException
any(), anyLong(), anyString(), anyList(), isNull(), isNotNull()
eq(x), same(x), argThat(p -> ...)
```

### Verification

```java
verify(m).call(arg);                 // exactly once
verify(m, times(2)).call(arg);
verify(m, never()).call(any());      // the highest-value verification
verify(m, atLeastOnce()).call(any());
verifyNoInteractions(m);
verifyNoMoreInteractions(m);         // brittle; justify it or delete it

InOrder o = inOrder(a, b);
o.verify(a).first();
o.verify(b).second();
```

### Capturing

```java
verify(gateway).authorize(captor.capture());
assertThat(captor.getValue().amountMinor()).isEqualTo(4797L);
assertThat(captor.getAllValues()).hasSize(2);
```

### Decision table — what kind of double?

| The collaborator is… | Use |
|---|---|
| A third-party HTTP API you cannot run | **Mock** — plus a contract test (62) or a recorded-response stub |
| A third-party SDK with a hostile API | **Mock** the thin interface *you* wrote over it, not the SDK |
| Your own repository | **Real database** (Testcontainers, 61) or an in-memory **fake** |
| Your own domain service | **The real one** |
| A value object / record / pure function | **The real one.** Never mock these. |
| The clock | Injected `Clock.fixed(...)` |
| Randomness | Injected `RandomGenerator` with a fixed seed |
| A message broker | Testcontainers Kafka (61), or a fake for decision tests |

### Gotchas checklist

- [ ] Mock what you do not own **and cannot run**. Nothing else.
- [ ] Assert on state or return values. `verify` only when the interaction is the outcome.
- [ ] `verify(..., never())` is usually your best verification.
- [ ] Delete `verifyNoMoreInteractions` unless you can name the specific bad call.
- [ ] Never mock a record, a value object, or a pure function.
- [ ] `@InjectMocks` injects `null` silently. Call the constructor instead.
- [ ] `when()` on a `@Spy` runs the real method. Use `doReturn().when()`.
- [ ] `void` methods need `doThrow`/`doNothing`, not `when`.
- [ ] Mixing raw args and matchers throws. Wrap the raw one in `eq()`.
- [ ] `UnnecessaryStubbingException` is a finding. Do not silence it class-wide.
- [ ] There is no `jest.mock`. If a class hard-codes `new`, fix the class.

---

## [BOOT 3.x DELTA]

1. **`mockito-inline` is no longer a separate dependency.** From Mockito 5 the inline
   mock maker is the default in `mockito-core`, so `final` classes and methods are
   mockable out of the box. An older Boot 3.x codebase may still declare
   `mockito-inline` explicitly; it is usually removable. Verify what your BOM manages
   with `mvn dependency:tree -Dincludes=org.mockito:*` rather than assuming.

2. **`MockitoAnnotations.initMocks(this)` is long deprecated** in favour of
   `openMocks(this)` — and both are superseded by `@ExtendWith(MockitoExtension.class)`,
   which additionally gives you strict stubs. If you find `initMocks` in a codebase,
   you are looking at JUnit 4-era test code, and it is silently running *lenient*.

3. **`@MockBean` versus `@MockitoBean`** belongs to Topic 60 rather than here, because
   it is a Spring test-context concern, not a Mockito one. The short version:
   `@MockitoBean` (`org.springframework.test.context.bean.override.mockito`) arrived
   in Spring Framework 6.2 and is the current annotation for the target stack;
   Boot's older `@MockBean` (`org.springframework.boot.test.mock.mockito`) was
   deprecated in favour of it. **I am not certain whether Boot 4.1 still ships
   `@MockBean` at all or has removed it** — do not take my word either way. Check:
   ```bash
   mvn dependency:tree -Dincludes=org.springframework.boot:spring-boot-test
   javap -cp $(find ~/.m2 -name 'spring-boot-test-*.jar' | head -1) \
     org.springframework.boot.test.mock.mockito.MockBean 2>&1 | head -5
   ```
   If `javap` cannot find the class, it has been removed on your version. Topic 60
   covers what changes in your test code either way.

4. **Plain Mockito usage — `mock`, `when`, `verify`, `ArgumentCaptor`, strictness —
   is identical across Boot 3.x and 4.x.** Mockito is not a Spring library; the Boot
   BOM only manages its version.

---

## When would I use this at work?

**1. Reviewing a PR that adds `@Mock OrderRepository`.**
Your comment is not "don't use mocks" — it is a question: "what does this test prove
that a Testcontainers test wouldn't, and what does the mock assume about `save` that
Hibernate doesn't guarantee?" Half the time the author realises the test belongs at a
different level. That one question, asked consistently, is most of what separates a
suite that enables refactoring from one that blocks it.

**2. Joining a team whose refactors keep stalling.**
Run `grep -rc "verify(" src/test/java | sort -t: -k2 -rn | head`. The top five files
are almost certainly the reason. You now have a concrete, evidence-backed proposal
on day two rather than an opinion.

**3. Deciding whether a new class is well designed, before writing any test.**
You look at the constructor. If it takes its collaborators as interfaces and takes a
`Clock` instead of calling `Instant.now()`, the test is going to be three lines. If
it constructs an HTTP client inside a method, no amount of Mockito skill will save
you, and the right move is to change the class. **In Java, the constructor is the
test plan.**

---

## Connected topics

**Prerequisites:**
- **04 — Interfaces**: a mock is a generated implementation of an interface. Mocking
  a concrete class works but is a weaker design.
- **09 — Exception design**: `thenThrow` is only useful if your domain exceptions
  carry structured data. Topic 58's Trap 2 applies here too.
- **17 — Immutability**: never mock a value object. Construct the real one.
- **27 — Records**: fixtures and command objects. Records make fakes trivial.
- **39 — Constructor injection**: **the load-bearing prerequisite.** Without it,
  nothing in this topic is usable. Re-read it if any of this felt abstract.
- **40 — Proxying**: a Mockito mock is a generated class, exactly like a CGLIB proxy.
  Also: a `@Transactional` method under test is only transactional when called
  *through* the Spring proxy — which a plain Mockito test does not create.
- **58 — JUnit 5**: `MockitoExtension` is an `@ExtendWith`, and per-method test
  instances are why mocks are fresh without `reset()`.

**This unlocks:**
- **60 — Spring test slices**: `@MockitoBean` puts a Mockito mock *into the Spring
  context* — and doing so forks the context cache. That is where the cost lives.
- **61 — Testcontainers**: the answer to Trap 2. Every line of the "what the mock
  lies about" table is something a real Postgres tells you truthfully.
- **62 — Contract testing**: the answer to Trap 4. The mock and the provider's
  verification become the same artefact.
- **63 — Mutation testing**: reveals mock-heavy tests that execute code without
  asserting anything about it.
- **64 — Property-based testing**: needs real implementations to run against;
  properties over mocks are meaningless.
- **99 — jcstress**: the honest tool for concurrency correctness. Trap 2's race
  condition is not detectable by any amount of mocking.

---

*Java baseline 21, running on JDK 25. Mockito's API shown here has been stable across
its 4.x and 5.x lines; the notable change was the inline mock maker becoming the
default in 5, which removed the need for the separate `mockito-inline` artifact.
Verify the managed version against your own Boot BOM rather than against this
document.*
