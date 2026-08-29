# 58 — JUnit 5 vs Jest — Lifecycle, Parameterized Tests, Extensions

## Phase: 6 — Testing
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: the first tests over `orderflow`'s pure domain logic — order totals, volume-discount tiers, and the domain exception hierarchy from Topic 09. No Spring context yet. This is the layer that must be fast, because everything in Topics 60–65 is slower.

---

## Mastery line (from the master plan)

> You use `@ParameterizedTest` with a `@MethodSource` instead of copy-pasted cases,
> and know why JUnit creates a new test instance per method by default (and what
> `@TestInstance(PER_CLASS)` changes).

## Mid → Senior (from the master plan)

> "JUnit is Java's Jest" → "structurally similar, but there's no module mocking —
> you can't intercept an import, so testability is a *design* property enforced by
> injection. The extension model is also the integration point Spring, Testcontainers
> and Mockito all hook into."

---

## ELI5 anchor

Imagine a workshop with a bench.

In **Jest**, you have one bench. Every test walks up to the same bench, uses it, and
walks away. If a test leaves a screwdriver on it, the next test finds the
screwdriver. You clean the bench yourself in `beforeEach`. If you forget, tests
start depending on the order they ran in.

In **JUnit**, by default, you get **a brand new bench for every single test**. The
workshop builds one, you use it, it is thrown away, and the next test gets a fresh
one. Nothing you leave on the bench survives.

That is the single biggest mechanical difference, and it flips a habit you have
built over years. Your Jest instinct says "state leaks unless I clean it". Java's
default says "state cannot leak unless you go out of your way".

There is a switch — `@TestInstance(PER_CLASS)` — that gives you Jest's single bench
back. People flip it for convenience and then spend a week debugging tests that
only fail on CI. That is Trap 1.

---

## The bridge from what you know

### The vocabulary map — mostly honest

| Jest | JUnit 5 (Jupiter) | Verdict |
|---|---|---|
| `describe('...', () => {})` | `@Nested class` + `@DisplayName` | **HONEST ANALOGUE** — grouping and readable names |
| `test(...)` / `it(...)` | `@Test` on a method | **HONEST ANALOGUE** |
| `beforeEach` | `@BeforeEach` | **HONEST ANALOGUE** |
| `afterEach` | `@AfterEach` | **HONEST ANALOGUE** |
| `beforeAll` | `@BeforeAll` (must be `static` by default) | **PARTIAL** — the `static` requirement is a consequence of per-method instances |
| `test.each([...])` | `@ParameterizedTest` + `@CsvSource`/`@MethodSource` | **PARTIAL** — richer, more verbose, statically typed |
| `expect(x).toBe(y)` | AssertJ `assertThat(x).isEqualTo(y)` | **PARTIAL** — different library, better failure output |
| `test.skip` | `@Disabled("reason")` | **HONEST ANALOGUE** |
| `test.only` | *no equivalent* | **NO ANALOGUE** — you use `mvn -Dtest=ClassName#methodName` or the IDE |
| `jest.setTimeout` | `@Timeout(5)` | **PARTIAL** |
| custom environment / setupFiles | `@ExtendWith(SomeExtension.class)` | **PARTIAL** — the extension model is far more powerful |
| **`jest.mock('./module')`** | **nothing. At all.** | **NO ANALOGUE** — Topic 59 is entirely about what you do instead |

### The three differences that actually change your behaviour

**1. A new test instance per test method.**

```java
class OrderTotalsTest {

    private List<OrderLine> lines = new ArrayList<>();   // fresh in EVERY test

    @Test
    void firstTest() {
        lines.add(line("SKU-1001", 2, 1999L));
        assertThat(lines).hasSize(1);
    }

    @Test
    void secondTest() {
        assertThat(lines).isEmpty();   // passes. JUnit built a NEW OrderTotalsTest.
    }
}
```

In Jest, the module-level equivalent of `lines` would still hold that entry. In
JUnit, `secondTest` runs on a different object. `lines` was constructed again from
scratch.

**Why JUnit does this:** test isolation by construction rather than by discipline.
The JUnit team decided that the cheapest way to stop tests leaking into each other
was to make leaking structurally impossible. Object allocation is nearly free
(Topic 68 will show you why), so the cost is negligible.

**The consequence you must remember:** `@BeforeAll` and `@AfterAll` run once for the
whole class, so they cannot belong to any single instance — hence they must be
`static`. Same for `@MethodSource` factory methods. If you have ever wondered "why
does JUnit demand `static` here", this is the entire reason.

**2. There is no `test.only`.**

Jest lets you focus a test by editing the file. JUnit does not, deliberately —
a focused test is easy to commit by accident. You select tests from the command line
or the IDE instead:

```bash
mvn -Dtest=OrderTotalsTest test
mvn -Dtest=OrderTotalsTest#appliesFivePercentAtTenUnits test
mvn -Dtest='Order*Test' test
```

**3. Assertions live in a separate library, and which one you pick matters a lot.**

JUnit ships `org.junit.jupiter.api.Assertions` (`assertEquals`, `assertTrue`,
`assertThrows`). It is adequate. **AssertJ** (`assertThat`) ships in
`spring-boot-starter-test` and produces dramatically better failure messages. You
will use AssertJ for almost everything and JUnit's own assertions only for
`assertThrows` and `assertAll`. Trap 3 shows exactly what the difference looks like
when a test fails at 2am.

### What genuinely does not transfer

Jest is a *runner plus a mocking framework plus an assertion library plus a coverage
tool* in one package. JUnit is only the runner and lifecycle. In Java you assemble:

| Job | Jest | Java |
|---|---|---|
| Find and run tests | Jest | **JUnit Platform** |
| Write tests | Jest | **JUnit Jupiter** (the `@Test` API) |
| Assert | `expect` | **AssertJ** (or Jupiter's `Assertions`) |
| Mock | `jest.fn`, `jest.mock` | **Mockito** — Topic 59 |
| Spring wiring | n/a | **`SpringExtension`** — Topic 60 |
| Real infrastructure | n/a (usually) | **Testcontainers** — Topic 61 |
| Coverage | built in | **JaCoCo** |
| Build integration | `npm test` | **Maven Surefire / Failsafe** — Topic 31 |

That looks like more moving parts, and it is. The payoff is that each piece is
replaceable and each one hooks in through the same documented extension point.

---

## What is this?

**JUnit 5** is three separate things with three separate jobs. Knowing the split is
what lets you read a stack trace or a Maven error later.

| Piece | What it is | You interact with it… |
|---|---|---|
| **JUnit Platform** | The engine API and launcher. Maven Surefire, Gradle and your IDE talk to *this*. | Almost never directly. It appears in `junit-platform.properties` and in Surefire output. |
| **JUnit Jupiter** | The programming model: `@Test`, `@BeforeEach`, `@ParameterizedTest`, assertions, and the **extension model**. | Constantly. This is "JUnit 5" in everyday speech. |
| **JUnit Vintage** | An engine that runs old JUnit 4 tests on the Platform. | Only when migrating a legacy codebase. Boot's starter excludes it by default. |

The Platform/Jupiter split exists so that other test styles (Spock, Cucumber,
jqwik from Topic 64) can run on the same launcher and appear in the same report.
That is not trivia — it is why your property-based tests in Topic 64 will show up
in the same `mvn test` run as everything else.

### The extension model — the part that matters most long term

`@ExtendWith(X.class)` registers an object that JUnit calls back at defined points
in a test's lifecycle. Extensions can:

- run code before/after each test or the whole class (`BeforeEachCallback`, `AfterAllCallback`, …)
- **supply arguments to your test methods** (`ParameterResolver`)
- intercept and transform thrown exceptions (`TestExecutionExceptionHandler`)
- decide whether a test runs at all (`ExecutionCondition`)

This one interface family is how every other tool in Phase 6 plugs in:

- `@ExtendWith(SpringExtension.class)` — starts and caches a Spring `ApplicationContext` (Topic 60)
- `@ExtendWith(MockitoExtension.class)` — creates `@Mock` fields and verifies strict stubs (Topic 59)
- `@Testcontainers` — a meta-annotation over an extension that starts/stops containers (Topic 61)

So when the master plan says "the extension model is the integration point Spring,
Testcontainers and Mockito all hook into", it means it literally. You are not
learning three unrelated tools; you are learning one callback interface three times.

> **Which JUnit version does Spring Boot 4.1 give you?**
> Do not take a number from me. JUnit 6.0 exists and raised the Java baseline; the
> Jupiter programming model, the annotations and the `org.junit.jupiter.api` package
> names are the same across that boundary, so everything in this document holds
> either way. But the exact managed version comes from the Boot BOM, not from
> memory. Check it:
> ```bash
> mvn dependency:tree -Dincludes=org.junit.jupiter:*,org.junit.platform:*
> mvn help:evaluate -Dexpression=junit-jupiter.version -q -DforceStdout
> ```
> If those disagree with anything written here, your build is the authority.

---

## Why does it matter?

Three concrete reasons, in the order you will feel them.

**1. Your Jest instincts about shared state are now backwards.**
You will write a `@BeforeEach` that resets a field which never needed resetting, and
one day you will add `@TestInstance(PER_CLASS)` to avoid a `static` keyword and
silently reintroduce every state-leak bug Jest ever gave you — but only on CI, where
the method order differs.

**2. Copy-pasted test cases are how boundary bugs survive.**
`orderflow`'s discount tiers change at 10, 50 and 100 units. Six copy-pasted test
methods will test 5, 25, 75 and 200 and never test 9, 10, 49 or 50. A
`@ParameterizedTest` with a `@CsvSource` makes the boundary list explicit and
reviewable. Topic 63 (mutation testing) will prove that this is not a style
preference: the surviving mutants in a suite are overwhelmingly boundary mutants.

**3. Every slow thing in Phase 6 is an extension.**
Spring contexts, containers, Mockito. If you do not understand that `@ExtendWith` is
the hook, you cannot reason about *why* one test class takes 4 seconds and another
takes 40 milliseconds. Topic 60 is entirely about that difference.

---

## Syntax breakdown

Only constructs that are genuinely new to you.

### `@Test`

```java
import org.junit.jupiter.api.Test;

class OrderTotalsTest {
    @Test
    void sumsLineTotals() { ... }
}
```

| Bit | What it means |
|---|---|
| `org.junit.jupiter.api.Test` | **Jupiter's** `@Test`. If you import `org.junit.Test` you have imported JUnit 4 and the method will silently not run. Check this import first when "my test didn't execute". |
| no `public` | JUnit 5 does not require `public` on test classes or methods. Package-private is the convention (Topic 03 — this is a real use of package-private as a boundary). |
| returns `void` | A test that returns a value is an error in Jupiter. There are no `async` tests; blocking is normal in Java tests. |

### `@DisplayName` and `@Nested`

```java
@DisplayName("Order totals")
class OrderTotalsTest {

    @Nested
    @DisplayName("when the customer qualifies for a volume discount")
    class VolumeDiscounts {

        @Test
        @DisplayName("applies 5% from 10 units")
        void appliesFivePercentAtTenUnits() { ... }
    }
}
```

| Bit | What it means |
|---|---|
| `@DisplayName` | The label in reports and IDE output. Java method names cannot contain spaces; this is how you get a readable sentence. |
| `@Nested` | An **inner class** (non-static) that JUnit treats as a test group — your `describe`. |
| non-static inner class | Required. A `@Nested` class holds a reference to the outer instance, so the outer `@BeforeEach` runs before the inner one. That is what makes nested setup layering work. |

Outer `@BeforeEach` methods run before inner ones, outermost first — exactly like
nested `describe` blocks in Jest. This is the one place your Jest intuition
transfers perfectly.

### Lifecycle annotations

```java
@BeforeAll  static void loadFixtures() { }   // once per class. static by default.
@BeforeEach void setUp()             { }     // before every @Test
@AfterEach  void tearDown()          { }     // after every @Test, even on failure
@AfterAll   static void cleanUp()    { }     // once per class
```

The `static` on `@BeforeAll`/`@AfterAll` is not a style rule. With per-method
instances there *is* no single instance for a class-level method to belong to.

### `@ParameterizedTest` — your `test.each`

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.MethodSource;
import org.junit.jupiter.params.provider.ValueSource;

@ParameterizedTest(name = "{0} units -> {1} basis points off")
@CsvSource({
    "1,    0",
    "9,    0",
    "10, 500",
    "49, 500",
    "50, 1000",
    "99, 1000",
    "100, 1500"
})
void discountBasisPointsByQuantity(int quantity, int expectedBasisPoints) {
    assertThat(DiscountPolicy.basisPointsFor(quantity))
        .isEqualTo(expectedBasisPoints);
}
```

| Bit | What it means |
|---|---|
| `@ParameterizedTest` | Replaces `@Test`. Do not put both on one method. |
| `name = "{0} units -> {1}"` | Report label per invocation. `{0}`, `{1}` are the arguments. Without it your report says "[1]", "[2]" and a failure tells you nothing. |
| `@ValueSource(ints = {...})` | One argument per invocation. Simplest source. |
| `@CsvSource({"10, 500"})` | Multiple arguments, parsed from strings. Good up to ~3 columns of simple types. |
| `@EnumSource(OrderStatus.class)` | One invocation per enum constant. Excellent for exhaustiveness over `orderflow`'s statuses. |
| `@NullSource` / `@EmptySource` | Adds a `null` or empty invocation. Cheap null-handling coverage. |

**`@MethodSource` — the one you will use most.** When arguments are real domain
objects rather than strings:

```java
@ParameterizedTest(name = "{0}")
@MethodSource("insufficientStockScenarios")
void rejectsWhenStockIsShort(String description, Inventory inventory, int requested) {
    assertThatThrownBy(() -> InventoryPolicy.reserve(inventory, requested))
        .isInstanceOf(InsufficientStockException.class);
}

static Stream<Arguments> insufficientStockScenarios() {     // MUST be static
    return Stream.of(
        Arguments.of("no stock at all",       inventory(0, 0),   1),
        Arguments.of("all stock reserved",    inventory(0, 12),  1),
        Arguments.of("one short",             inventory(4, 0),   5)
    );
}
```

| Bit | What it means |
|---|---|
| `static Stream<Arguments>` | The factory. **Must be `static`** — same per-method-instance reason as `@BeforeAll`. Under `@TestInstance(PER_CLASS)` it may be non-static. |
| `Arguments.of(...)` | One invocation's argument tuple. |
| method name as a `String` | Resolved by reflection at run time. A typo is a runtime failure, not a compile error. This is one of the few places JUnit gives up type safety. |
| Returning `Stream` | Also allowed: `Iterable`, `Iterator`, arrays. `Stream` is idiomatic (Topic 23). |

### Assertions — AssertJ vs Jupiter

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertAll;

// AssertJ — your default. Fluent, and the failure message shows a real diff.
assertThat(order.totalMinor()).isEqualTo(4_797L);
assertThat(order.lines()).hasSize(3)
                         .extracting(OrderLine::sku)
                         .containsExactly("SKU-1001", "SKU-1002", "SKU-1003");
assertThat(order.status()).isEqualTo(OrderStatus.PENDING);

// Exceptions, AssertJ style
assertThatThrownBy(() -> orderService.place(command))
    .isInstanceOf(InsufficientStockException.class)
    .extracting("sku").isEqualTo("SKU-1001");

// Exceptions, Jupiter style — returns the exception so you can inspect it
InsufficientStockException ex =
    assertThrows(InsufficientStockException.class, () -> orderService.place(command));
assertThat(ex.sku()).isEqualTo("SKU-1001");

// assertAll — report EVERY failure, not just the first
assertAll(
    () -> assertThat(order.status()).isEqualTo(OrderStatus.PAID),
    () -> assertThat(order.totalMinor()).isEqualTo(4_797L),
    () -> assertThat(order.paidAt()).isNotNull()
);
```

| Bit | What it means |
|---|---|
| `static` import of `assertThat` | Java has no top-level functions. Every helper is a static method on a class, and `import static` is how you call it without the class name. |
| `assertThatThrownBy(() -> ...)` | Takes a lambda. Nothing is thrown until AssertJ invokes it. |
| `assertAll` | Runs every assertion and aggregates failures. Without it, the first failure hides the rest — and you burn a CI cycle per bug. |
| `.extracting(...)` | AssertJ's projection. Asserts on a field of each element without a loop. |

### `@ExtendWith` and `@TestInstance`

```java
@ExtendWith(MockitoExtension.class)      // Topic 59
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class SomethingTest { }
```

| Bit | What it means |
|---|---|
| `@ExtendWith(X.class)` | Register extension `X`. Repeatable; several extensions can stack. |
| `@TestInstance(PER_METHOD)` | **The default.** New instance per test method. |
| `@TestInstance(PER_CLASS)` | One instance for the whole class. `@BeforeAll` and `@MethodSource` may now be non-static. **And your fields now leak between tests.** See Trap 1. |

---

## Example 1 — minimal

`orderflow`'s volume-discount policy. Pure function, no Spring, no database.

```java
package com.orderflow.catalog;

/** Volume discount expressed in basis points (1/100th of a percent) to avoid
 *  floating point in money maths — see Topic 01. */
public final class DiscountPolicy {

    private DiscountPolicy() { }

    public static int basisPointsFor(int quantity) {
        if (quantity < 0)   throw new IllegalArgumentException("quantity must not be negative");
        if (quantity >= 100) return 1500;   // 15%
        if (quantity >= 50)  return 1000;   // 10%
        if (quantity >= 10)  return  500;   //  5%
        return 0;
    }
}
```

The test:

```java
package com.orderflow.catalog;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@DisplayName("DiscountPolicy")
class DiscountPolicyTest {

    @ParameterizedTest(name = "{0} units -> {1} bps")
    @DisplayName("applies the tier for the quantity, on both sides of every boundary")
    @CsvSource({
        "0,     0",
        "1,     0",
        "9,     0",
        "10,  500",
        "49,  500",
        "50, 1000",
        "99, 1000",
        "100,1500",
        "5000,1500"
    })
    void basisPointsByQuantity(int quantity, int expectedBps) {
        assertThat(DiscountPolicy.basisPointsFor(quantity)).isEqualTo(expectedBps);
    }

    @Test
    @DisplayName("rejects a negative quantity")
    void rejectsNegativeQuantity() {
        assertThatThrownBy(() -> DiscountPolicy.basisPointsFor(-1))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

**Read the `@CsvSource` list, not the code.** Every tier boundary appears twice —
the last value below it and the first value at it. `9, 10` and `49, 50` and
`99, 100`. That pairing is the whole point: an off-by-one in the policy
(`quantity > 10` instead of `>= 10`) fails the `10` row and only the `10` row.

Nine copy-pasted `@Test` methods would express the same thing in about sixty lines,
and reviewers would skim them. Nine rows in a table get read.

---

## Example 2 — production scenario (on the project spine)

The real thing: `orderflow`'s order-total calculation, which has to combine line
items, volume discounts, a wallet credit and rounding — and must never produce a
total that disagrees with the sum of its parts, because Finance reconciles it.

### The code under test

```java
package com.orderflow.orders;

import java.util.List;

/** All money in minor units (pence/cents) as a long — Topic 01. No doubles anywhere. */
public record OrderLine(String sku, int quantity, long unitPriceMinor) {

    public OrderLine {
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive, was " + quantity);
        }
        if (unitPriceMinor < 0) {
            throw new IllegalArgumentException("unitPriceMinor must not be negative");
        }
    }

    public long grossMinor() {
        return Math.multiplyExact((long) quantity, unitPriceMinor);   // overflow throws, Topic 01
    }
}
```

```java
package com.orderflow.orders;

import com.orderflow.catalog.DiscountPolicy;
import java.util.List;

public final class OrderTotals {

    private OrderTotals() { }

    /**
     * Discount applies per line, based on that line's quantity.
     * Rounding is HALF_UP at the line level, then lines are summed —
     * summing first and rounding once gives a different answer, and Finance
     * reconciles against the line-level figure.
     */
    public static long netTotalMinor(List<OrderLine> lines) {
        long total = 0;
        for (OrderLine line : lines) {
            long gross = line.grossMinor();
            int bps = DiscountPolicy.basisPointsFor(line.quantity());
            long discount = roundHalfUp(gross * bps, 10_000);
            total = Math.addExact(total, gross - discount);
        }
        return total;
    }

    private static long roundHalfUp(long numerator, long denominator) {
        return (numerator + denominator / 2) / denominator;
    }
}
```

### The test class

```java
package com.orderflow.orders;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

import java.util.List;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.junit.jupiter.api.Assertions.assertAll;

@DisplayName("OrderTotals")
class OrderTotalsTest {

    // Fresh for every test method. No @BeforeEach needed to reset it.
    private final List<OrderLine> singleCheapLine =
        List.of(new OrderLine("SKU-1001", 1, 1999L));

    @Nested
    @DisplayName("with no discount tier reached")
    class BelowFirstTier {

        @Test
        @DisplayName("net total equals gross total")
        void netEqualsGross() {
            assertThat(OrderTotals.netTotalMinor(singleCheapLine)).isEqualTo(1999L);
        }

        @Test
        @DisplayName("sums multiple lines exactly")
        void sumsMultipleLines() {
            List<OrderLine> lines = List.of(
                new OrderLine("SKU-1001", 1, 1999L),
                new OrderLine("SKU-1002", 2,  850L),
                new OrderLine("SKU-1003", 3,  333L)
            );
            // 1999 + 1700 + 999
            assertThat(OrderTotals.netTotalMinor(lines)).isEqualTo(4698L);
        }
    }

    @Nested
    @DisplayName("with volume discounts")
    class VolumeDiscounts {

        @ParameterizedTest(name = "{0}")
        @MethodSource("com.orderflow.orders.OrderTotalsTest#discountScenarios")
        void appliesTheCorrectTier(String description, OrderLine line, long expectedNetMinor) {
            assertThat(OrderTotals.netTotalMinor(List.of(line)))
                .as("%s", description)
                .isEqualTo(expectedNetMinor);
        }

        @Test
        @DisplayName("discounts each line by its own quantity, not the order's total quantity")
        void discountIsPerLineNotPerOrder() {
            // Two lines of 6 units each: 12 units in the order, but neither line
            // reaches the 10-unit tier. This is the rule Finance signed off on,
            // and it is the one a refactor is most likely to break.
            List<OrderLine> lines = List.of(
                new OrderLine("SKU-1001", 6, 1000L),
                new OrderLine("SKU-1002", 6, 1000L)
            );
            assertThat(OrderTotals.netTotalMinor(lines)).isEqualTo(12_000L);
        }
    }

    @Nested
    @DisplayName("rounding")
    class Rounding {

        @Test
        @DisplayName("rounds half up at the line level and never loses a penny across lines")
        void roundsHalfUpPerLine() {
            // 10 units @ 1005 = 10050 gross; 5% = 502.5 -> 503 half up; net 9547
            OrderLine line = new OrderLine("SKU-2001", 10, 1005L);
            assertThat(OrderTotals.netTotalMinor(List.of(line))).isEqualTo(9547L);
        }

        @Test
        @DisplayName("reports every rounding property, not just the first that fails")
        void totalIsConsistentWithLines() {
            List<OrderLine> lines = List.of(
                new OrderLine("SKU-2001", 10, 1005L),
                new OrderLine("SKU-2002", 50,  333L)
            );
            long net = OrderTotals.netTotalMinor(lines);
            long gross = lines.stream().mapToLong(OrderLine::grossMinor).sum();

            assertAll(
                () -> assertThat(net).as("net is never above gross").isLessThanOrEqualTo(gross),
                () -> assertThat(net).as("net is never negative").isNotNegative(),
                () -> assertThat(net).as("exact expected value").isEqualTo(9547L + 14_985L)
            );
        }
    }

    @Nested
    @DisplayName("invalid input")
    class InvalidInput {

        @Test
        @DisplayName("a zero-quantity line is rejected at construction, not at totalling")
        void zeroQuantityRejectedAtConstruction() {
            assertThatThrownBy(() -> new OrderLine("SKU-1001", 0, 1999L))
                .isInstanceOf(IllegalArgumentException.class);
        }

        @Test
        @DisplayName("an absurd quantity overflows loudly rather than silently wrapping")
        void overflowThrows() {
            OrderLine line = new OrderLine("SKU-1001", Integer.MAX_VALUE, Long.MAX_VALUE / 2);
            assertThatThrownBy(() -> OrderTotals.netTotalMinor(List.of(line)))
                .isInstanceOf(ArithmeticException.class);
        }
    }

    static Stream<Arguments> discountScenarios() {
        return Stream.of(
            Arguments.of("9 units: no tier",   new OrderLine("SKU-1001",   9, 1000L),   9_000L),
            Arguments.of("10 units: 5%",       new OrderLine("SKU-1001",  10, 1000L),   9_500L),
            Arguments.of("49 units: 5%",       new OrderLine("SKU-1001",  49, 1000L),  46_550L),
            Arguments.of("50 units: 10%",      new OrderLine("SKU-1001",  50, 1000L),  45_000L),
            Arguments.of("99 units: 10%",      new OrderLine("SKU-1001",  99, 1000L),  89_100L),
            Arguments.of("100 units: 15%",     new OrderLine("SKU-1001", 100, 1000L),  85_000L)
        );
    }
}
```

### What to notice about this test class

1. **`singleCheapLine` is an instance field with no `@BeforeEach` resetting it.**
   That is correct and safe *because of per-method instances*. In Jest you would
   need a `beforeEach` or a factory function. Here the constructor is the
   `beforeEach`.

2. **`@Nested` groups by *situation*, not by method name.** Reading the report gives
   you a sentence: "OrderTotals > with volume discounts > discounts each line by its
   own quantity, not the order's total quantity". That sentence is documentation
   that cannot go stale, because it fails when the behaviour changes.

3. **`discountIsPerLineNotPerOrder` is the most valuable test in the class.** It
   encodes a business rule that is not obvious from the code and that a well-meaning
   refactor would break. Notice the comment explains *why*, not *what*.

4. **`@MethodSource` uses a fully-qualified reference** (`com.orderflow.orders.OrderTotalsTest#discountScenarios`)
   because the `@ParameterizedTest` lives in a `@Nested` inner class and the factory
   lives in the outer one. A bare `"discountScenarios"` would fail at run time with
   a "could not find factory method" error. This is the single most common
   `@MethodSource` mistake.

5. **Overflow is asserted, not assumed.** `Math.multiplyExact` from Topic 01 throws
   instead of wrapping. The test locks that in.

6. **No mocks anywhere.** Everything here is a pure function over records. This is
   the fastest and most valuable tier of your suite, and Topic 59 will argue that
   most people mock things that should have lived here instead.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a shared mutable field plus `@TestInstance(PER_CLASS)`

You added `@TestInstance(PER_CLASS)` because you wanted a non-static
`@BeforeAll` (or a non-static `@MethodSource`). It compiled. Everything was green.

**Wrong:**

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderRepositoryContractTest {

    private final List<Order> placedOrders = new ArrayList<>();   // now SHARED

    @BeforeAll
    void loadCatalogue() { ... }        // non-static, which is why PER_CLASS was added

    @Test
    void placingAnOrderRecordsIt() {
        placedOrders.add(order("SKU-1001", 2));
        assertThat(placedOrders).hasSize(1);
    }

    @Test
    void anEmptyBasketPlacesNothing() {
        assertThat(placedOrders).isEmpty();       // depends on running FIRST
    }
}
```

**Exact symptom:** the suite passes on your machine and fails on CI with

```
Expecting empty but was: [Order[sku=SKU-1001, quantity=2]]
```

*(shape of an AssertJ failure message, shown as an illustration of the format)*

— from a test you did not touch, in a build where the only change was in another
module. Re-running the job sometimes goes green. Someone labels it "flaky" and adds
a retry.

**Root cause:** `PER_CLASS` gives you exactly Jest's model — one instance, shared
fields — and JUnit's method execution order is deterministic but deliberately
non-obvious, and *may differ across JDK versions or class-file layouts*. Test B only
passes when it runs before test A.

**Fix (in order of preference):**

1. **Delete `@TestInstance(PER_CLASS)`.** Make `@BeforeAll` and `@MethodSource`
   static. That is the default for a reason.
2. If you genuinely need `PER_CLASS` (a `@Container` field you want non-static, or
   an expensive fixture), add `@BeforeEach void reset() { placedOrders.clear(); }`
   and treat every mutable field as a hazard you are now personally responsible for.
3. Prefer making the field effectively immutable — build it in the test method, not
   in a field.

**Prove it to yourself:** force random ordering and watch it fail on demand.

```java
@TestMethodOrder(MethodOrderer.Random.class)
```

or globally, in `src/test/resources/junit-platform.properties`:

```properties
junit.jupiter.testmethod.order.default = org.junit.jupiter.api.MethodOrderer$Random
```

If any test fails under random ordering, you have order dependence. That is a
one-line permanent smoke alarm and it is worth turning on in CI.

---

### Trap 2 — asserting on an exception's message string

**Wrong:**

```java
@Test
void rejectsOversell() {
    assertThatThrownBy(() -> inventoryService.reserve("SKU-1001", 5))
        .hasMessage("Insufficient stock for SKU-1001: requested 5, available 2");
}
```

**Exact symptom:** six months later someone improves the error copy for the customer-
facing `ProblemDetail` response (Topic 46) — "requested 5, available 2" becomes
"only 2 left". The behaviour is identical. Eleven tests across four modules go red.
The PR that fixes a real bug is now blocked on updating message strings, so the
author bulk-updates them without reading them, and one genuine assertion is lost in
the noise.

**Root cause:** an exception's message is a **human-readable diagnostic**, not part
of the contract. You asserted on the least stable thing in the whole exception.

**Fix:** put the machine-readable data on the exception as fields, and assert on
those. This is exactly why Topic 09 told you to design a domain exception hierarchy.

```java
package com.orderflow.inventory;

public class InsufficientStockException extends OrderflowException {

    private final String sku;
    private final int requested;
    private final int available;

    public InsufficientStockException(String sku, int requested, int available) {
        super("Insufficient stock for %s: requested %d, available %d"
                .formatted(sku, requested, available));
        this.sku = sku;
        this.requested = requested;
        this.available = available;
    }

    public String sku()      { return sku; }
    public int requested()   { return requested; }
    public int available()   { return available; }
}
```

```java
@Test
void rejectsOversell() {
    assertThatThrownBy(() -> inventoryService.reserve("SKU-1001", 5))
        .isInstanceOf(InsufficientStockException.class)
        .asInstanceOf(InstanceOfAssertFactories.type(InsufficientStockException.class))
        .satisfies(ex -> {
            assertThat(ex.sku()).isEqualTo("SKU-1001");
            assertThat(ex.requested()).isEqualTo(5);
            assertThat(ex.available()).isEqualTo(2);
        });
}
```

Now the copy can change freely and the test still checks what it meant to check.

> **The exception to the exception:** if the message *is* the contract — an error
> code in a public API response body — then assert on it, and put it behind a
> constant so both the producer and the test reference the same symbol.

---

### Trap 3 — `assertTrue(a.equals(b))` instead of `assertThat(a).isEqualTo(b)`

**Wrong:**

```java
@Test
void buildsTheExpectedOrder() {
    Order actual = orderFactory.from(command);
    assertTrue(actual.equals(expectedOrder));
}
```

**Exact symptom:** the build fails with

```
org.opentest4j.AssertionFailedError: expected: <true> but was: <false>
```

*(shape of a JUnit `assertTrue` failure, shown as an illustration of the format)*

That is the entire message. It tells you nothing. You now reproduce it locally, add
print statements, and discover the difference was one field — `currency` was `"GBP"`
in one and `null` in the other.

**Root cause:** `assertTrue` receives a `boolean`. By the time the assertion runs,
both objects have been reduced to `true` or `false` and the values are gone. The
assertion library has nothing left to report.

**Fix:** give the assertion the objects, not the comparison result.

```java
assertThat(actual).isEqualTo(expectedOrder);
```

AssertJ then prints both objects and, for records and collections, a field-by-field
diff. `usingRecursiveComparison()` goes further and reports the exact path:

```java
assertThat(actual)
    .usingRecursiveComparison()
    .ignoringFields("id", "createdAt")
    .isEqualTo(expectedOrder);
```

*(this reports which field paths differ, so you fix it from the CI log without
reproducing locally)*

> **Connects to Topic 13.** `isEqualTo` uses `equals`. If `Order` is an entity with
> a generated ID and a hand-written `equals`, you are now testing your `equals`
> implementation as much as your factory. `usingRecursiveComparison` sidesteps
> `equals` entirely and compares fields — which is usually what you actually
> wanted for a JPA entity, whose `equals` has to satisfy constraints that have
> nothing to do with your test.

---

### Trap 4 — a test that cannot fail

**Wrong:**

```java
@Test
void walletDebitFailsWhenBalanceIsTooLow() {
    try {
        walletService.debit(walletId, 10_000L);
    } catch (InsufficientFundsException expected) {
        // expected
    }
}
```

**Exact symptom:** none. The test is green. It has been green for two years. It was
green the entire time the debit method silently returned without throwing, because
someone removed the balance check during a refactor. You find out when a customer
goes to a negative balance in production.

**Root cause:** if the method does *not* throw, the `try` block completes normally
and the test passes. The test asserts nothing. Topic 63 (mutation testing) exists
precisely to find these — this test executes the line, so line coverage says 100%,
and the mutant survives.

**Fix:**

```java
@Test
void walletDebitFailsWhenBalanceIsTooLow() {
    assertThatThrownBy(() -> walletService.debit(walletId, 10_000L))
        .isInstanceOf(InsufficientFundsException.class);
}
```

`assertThatThrownBy` fails if nothing is thrown. So does `assertThrows`. **Never
hand-write a try/catch in a test.** If you find one in review, it is a defect
regardless of what it looks like it is doing.

---

### Trap 5 — `@BeforeEach` doing work that belongs in the test

**Wrong:**

```java
@BeforeEach
void setUp() {
    product   = new Product("SKU-1001", "Ceramic mug", 1999L, true);
    inventory = new Inventory(product.id(), 12, 0);
    wallet    = new Wallet(customerId, 50_000L);
    payment   = new Payment(orderId, 1999L, PaymentStatus.AUTHORIZED);
    command   = new PlaceOrderCommand(customerId, List.of(new LineItem("SKU-1001", 2)));
    // ...forty more lines
}
```

**Exact symptom:** a test fails and you cannot tell what the inputs were without
scrolling up sixty lines and mentally executing setup that half the tests do not
use. Worse: someone changes `inventory` from 12 to 3 units to make *their* test
work, and four unrelated tests break.

**Root cause:** shared setup is coupling. Every field in `@BeforeEach` is a
dependency that every test in the class now has, whether it needs it or not.

**Fix:** use **object mother / builder methods** and construct in the test.

```java
private static Order orderWith(int quantity) {
    return Order.draft(CUSTOMER_ID)
                .withLine("SKU-1001", quantity, 1999L)
                .build();
}

@Test
void appliesVolumeDiscountAtTenUnits() {
    Order order = orderWith(10);      // the input is visible right here
    assertThat(OrderTotals.netTotalMinor(order.lines())).isEqualTo(18_991L);
}
```

Keep `@BeforeEach` for genuinely universal, cheap wiring (constructing the
service-under-test with its collaborators). Everything scenario-specific goes in the
test.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and no test runner, so I am
not going to print output and pretend I captured it. What follows is the exact
command, what to look for, and how to read each possible outcome.

### Setup

```bash
cd ~/projects/orderflow
mvn -v          # confirm Maven and the JDK it is using
java --version  # should be 21 or 25
mvn dependency:tree -Dincludes=org.junit.jupiter:*,org.assertj:*,org.mockito:*
```

**What to look for in that last command:** the JUnit Jupiter artifacts, AssertJ, and
Mockito, all arriving transitively through `spring-boot-starter-test`.

| What you see | What it means |
|---|---|
| `spring-boot-starter-test` with `junit-jupiter`, `assertj-core`, `mockito-core` beneath it | Normal. The BOM is managing all three versions; do not pin them yourself. |
| `junit-vintage-engine` present | You have a JUnit 4 dependency pulling it in, or someone added it deliberately for legacy tests. Two engines run, and JUnit 4 annotations will silently work — which is how a `org.junit.Test` import mistake goes unnoticed. |
| No JUnit at all | `spring-boot-starter-test` is missing or is not in `<scope>test</scope>`. |

### Proof 1 — a new test instance per method (the headline claim)

Create `src/test/java/com/orderflow/InstanceLifecycleProofTest.java`:

```java
package com.orderflow;

import org.junit.jupiter.api.Test;

class InstanceLifecycleProofTest {

    private int counter = 0;

    @Test
    void first() {
        counter++;
        System.out.println("first  : instance=" + System.identityHashCode(this)
                           + " counter=" + counter);
    }

    @Test
    void second() {
        counter++;
        System.out.println("second : instance=" + System.identityHashCode(this)
                           + " counter=" + counter);
    }
}
```

```bash
mvn -Dtest=InstanceLifecycleProofTest test
```

**What to look for:** the two `instance=` numbers, and the two `counter=` values.

| What you see | What it means |
|---|---|
| Two **different** `instance=` values, `counter=1` on both lines | The default `PER_METHOD` lifecycle, confirmed. Your Jest intuition about shared module state does not apply. |
| The **same** `instance=` value, `counter=1` then `counter=2` | Someone has `@TestInstance(PER_CLASS)` on the class or set `junit.jupiter.testinstance.lifecycle.default` in `junit-platform.properties`. Find it. |
| No `System.out` at all | Surefire may be redirecting output to a file. Look in `target/surefire-reports/com.orderflow.InstanceLifecycleProofTest-output.txt`, or run with `-Dsurefire.useFile=false`. |

Now add `@TestInstance(TestInstance.Lifecycle.PER_CLASS)` to the class and re-run.
The `instance=` values become equal and `counter` reaches 2. **You have just
reproduced Trap 1 deliberately, in ten seconds.** That is the whole mechanism.

### Proof 2 — order dependence, on demand

Add to `src/test/resources/junit-platform.properties`:

```properties
junit.jupiter.testmethod.order.default = org.junit.jupiter.api.MethodOrderer$Random
```

```bash
mvn test
mvn test
mvn test
```

**What to look for:** whether the same set of tests passes every time.

| What you see | What it means |
|---|---|
| Green all three runs | No order dependence detected in this sample. Not a proof of absence — run it in CI for a fortnight. |
| A different test fails each run | You have shared mutable state. Look for `@TestInstance(PER_CLASS)`, `static` fields, or a `@BeforeAll` that mutates something. |
| The **same** test fails every run | Not order dependence — a genuine bug you happened to expose. Fix that first. |
| A `NoSuchMethodError` or "unknown MethodOrderer" | The property name or the class name is wrong. Note the `$` in `MethodOrderer$Random` — it is a nested class, and the `$` is the JVM's binary name separator (Topic 67). |

### Proof 3 — see what actually ran, and in what order

```bash
mvn surefire:test -Dsurefire.reportFormat=plain
ls target/surefire-reports/
cat target/surefire-reports/com.orderflow.orders.OrderTotalsTest.txt
```

**What to look for:** the per-class report with test names and elapsed time per
test, and the `-output.txt` sibling file holding anything the test printed.

| What you see | What it means |
|---|---|
| A `.txt` per test class with `Tests run: N, Failures: 0, Errors: 0, Skipped: 0` | Normal. The `Time elapsed` per class is your first input for Topic 60's suite-time investigation. |
| `Tests run: 0` for a class you expected to run | Either the class name does not match Surefire's include patterns (`*Test`, `Test*`, `*Tests`, `*TestCase`), or you imported `org.junit.Test` instead of `org.junit.jupiter.api.Test`. |
| `Skipped: N` | `@Disabled`, or a failed `assumeTrue(...)`. Assumptions skip; assertions fail. Know which one you wrote. |

### Proof 4 — the parameterized report

```bash
mvn -Dtest=DiscountPolicyTest test
cat target/surefire-reports/com.orderflow.catalog.DiscountPolicyTest.txt
```

**What to look for:** one entry per `@CsvSource` row, labelled by your `name`
pattern.

| What you see | What it means |
|---|---|
| Nine separate invocations named `9 units -> 0 bps`, `10 units -> 500 bps`, … | The `name` attribute is doing its job. A failure now tells you *which row* failed with no further investigation. |
| Invocations labelled `[1]`, `[2]`, `[3]` | You omitted `name = "..."` on `@ParameterizedTest`. Add it — this is free diagnostic value. |
| `Tests run: 1` | You left `@Test` on the method as well as `@ParameterizedTest`, or the source annotation is missing. |
| `ParameterResolutionException` | The `@MethodSource` factory could not be found (typo, or non-static, or in a different class than the test — see Example 2, note 4). |

### Proof 5 — run one test method

```bash
mvn -Dtest=OrderTotalsTest#discountIsPerLineNotPerOrder test
```

**What to look for:** `Tests run: 1`. This is your `test.only` replacement, and it
does not risk being committed.

| What you see | What it means |
|---|---|
| `Tests run: 1` | Correct. |
| `No tests were executed!` | With `@Nested` classes, the outer class name plus `#method` may not resolve. Use the nested class: `-Dtest='OrderTotalsTest$VolumeDiscounts#discountIsPerLineNotPerOrder'` (quote it — `$` is shell-expanded otherwise). |
| Build fails with "No tests matching pattern" | Add `-DfailIfNoSpecifiedTests=false` while exploring. |

---

## Practice exercises

### 1 — Easy: prove the lifecycle, then break it on purpose

Write `LifecycleOrderTest` in `orderflow` with:

- a `@BeforeAll`, `@BeforeEach`, two `@Test` methods, `@AfterEach`, `@AfterAll`,
  each printing one line naming itself
- one `@Nested` class containing its own `@BeforeEach` and one `@Test`

Run it and write down the exact execution order you observe. Then:

**Part A.** Predict, before running, what the order will be. Compare with reality.
Where you were wrong, say *why* your Jest intuition misled you.

**Part B.** Add an instance field `int callCount` incremented in `@BeforeEach`, and
assert `callCount == 1` in both tests. Confirm it passes.

**Part C.** Add `@TestInstance(PER_CLASS)`. Re-run. Which test fails and why? Now
make it pass again **without** removing `PER_CLASS`, and explain what you had to
take on responsibility for.

### 2 — Medium: parameterize a real policy, combining earlier topics

`orderflow` needs a `SkuValidator`. A valid SKU:

- matches `SKU-` followed by exactly four digits
- is never `null` or blank
- is compared case-insensitively for equality but stored uppercase

Implement `Sku` as a **record** (Topic 27) with validation in the compact
constructor (Topic 27), throwing a domain exception from your Topic 09 hierarchy —
not `IllegalArgumentException`. Give it a correct `equals`/`hashCode` (Topic 13:
case-insensitive equality means `hashCode` must be computed on the uppercase form,
or you get the silent `HashSet` corruption from that topic).

Then write `SkuTest` with:

- a `@ParameterizedTest` + `@ValueSource(strings = {...})` for the valid cases
- a `@ParameterizedTest` + `@NullAndEmptySource` + `@ValueSource` for the invalid ones
- a `@ParameterizedTest` + `@MethodSource` returning `Arguments` pairs proving that
  `Sku` instances differing only in case are `equals`, have the same `hashCode`, and
  **collapse to one element in a `HashSet`**
- one `@Test` using `assertAll` to check three properties of the same `Sku` at once

Requirement: no test method may be a copy of another with one literal changed. If
you find yourself doing that, the case belongs in a source annotation.

### 3 — Hard: production simulation — lock in the wallet-debit contract

`orderflow`'s `WalletService.debit(walletId, amountMinor)` is the highest-risk pure
logic in the system: getting it wrong takes money from the wrong person.

**Part A.** Write `WalletBalancePolicy` as a pure function over a `Wallet` record —
no Spring, no repository, no database:

```java
static long debit(long balanceMinor, long amountMinor)   // returns new balance
```

Rules: amount must be positive; the resulting balance must never be negative;
overflow must throw rather than wrap; a debit of exactly the full balance is legal
and leaves zero.

**Part B.** Write the test class with `@Nested` groups for *legal debits*,
*rejected debits* and *arithmetic edges*. Every boundary must be tested from both
sides. Use `@MethodSource` for anything with more than two columns.

**Part C.** Introduce three deliberate bugs, one at a time, and record which of your
tests catches each:

1. change `amountMinor <= 0` to `amountMinor < 0`
2. change `balance - amount < 0` to `balance - amount < -1`
3. replace `Math.subtractExact` with plain `-`

If any bug is caught by **zero** tests, your suite has a hole. Write the test that
closes it before moving on. Write down, for each bug, the exact assertion message
that caught it.

**Part D.** Now the design argument. Your `WalletService` in `orderflow` currently
does the balance check inline inside a `@Transactional` method that also loads the
wallet from a repository. Write two paragraphs answering:

- Which part of `WalletService` is testable at this speed (microseconds, no
  container, no Spring), and which part fundamentally is not?
- Which of these is the concurrency correctness question (two debits at once) and
  can a JUnit test of this kind *ever* prove it? Name the topic that can (99), and
  the topic that gets you closest before that (61).

This is the question Topics 59, 60 and 61 spend their entire length answering. Write
your answer now, before reading them, and compare afterwards.

---

## Interview questions

### Q1 — "How is JUnit 5 different from Jest, beyond the syntax?"

**Mid-level answer:** "They're pretty similar — `describe` becomes `@Nested`,
`beforeEach` becomes `@BeforeEach`, `test.each` becomes `@ParameterizedTest`. JUnit
uses annotations instead of callbacks."

**Senior answer:** "Three real differences. First, JUnit constructs a new test
instance per test method by default, so instance fields cannot leak between tests —
the opposite of Jest, where module-level state persists and you clean it manually.
`@TestInstance(PER_CLASS)` opts back into Jest's model and reintroduces order
dependence, which is why I only use it when I have a specific reason. Second, JUnit
is only the runner and lifecycle — assertions are AssertJ, mocking is Mockito,
coverage is JaCoCo. That is more assembly but each piece is replaceable. Third and
biggest: **there is no module mocking.** `jest.mock('./paymentGateway')` has no
equivalent, because Java has no import interception. That means testability is a
design property enforced by dependency injection rather than something the test
framework can retrofit — which is exactly why constructor injection is a testing
requirement in Java and not a style preference."

**What separates them:** the mid answer maps vocabulary. The senior answer names the
*consequence for how you design production code*, which is the actual reason the
difference matters.

**Interviewer's follow-up:** "So how would you test a class that calls a payment
gateway?" They want to hear "I inject it as an interface and pass a test double in
the constructor" — and they are checking whether you reach for a mock reflexively or
consider a fake first. That is Topic 59.

---

### Q2 — "Why does `@BeforeAll` have to be static?"

**Mid-level answer:** "It's a JUnit rule. It runs once for the class so it can't be
an instance method."

**Senior answer:** "It follows from the per-method instance lifecycle. If JUnit
creates a new instance for every test method, there is no single instance for a
class-level hook to belong to — so it must be static. The same constraint applies to
`@MethodSource` factory methods for the same reason. `@TestInstance(PER_CLASS)`
removes the constraint because it removes the premise: now there is exactly one
instance, so `@BeforeAll` and `@MethodSource` may be instance methods. That is the
legitimate use of `PER_CLASS` — and it comes with the cost that your fields are now
shared across tests, which is a real hazard, not a theoretical one."

**What separates them:** deriving the rule from the lifecycle rather than
memorising it, and naming what `PER_CLASS` costs.

**Interviewer's follow-up:** "Have you ever been bitten by `PER_CLASS`?" A good
answer describes an order-dependent failure that only appeared on CI, and what
diagnostic they used — random method ordering — rather than "we added a retry".

---

### Q3 — "When would you write a `@ParameterizedTest` and when would you not?"

**Mid-level answer:** "When you have the same test with different inputs. It saves
duplication."

**Senior answer:** "When the *assertion* is the same and only the data varies — a
policy table, boundary values, an enum's constants. The value isn't reduced
duplication, it's that the cases become a reviewable table: for a discount policy
with tiers at 10, 50 and 100, a `@CsvSource` makes it obvious whether both sides of
every boundary are covered, which nine copy-pasted methods never do. I would *not*
use it when the cases need different assertions or different setup — then you get a
single test with a branch inside it, and a failure tells you which row failed but
not what it was checking. And I use `@MethodSource` rather than `@CsvSource` as soon
as the arguments are domain objects rather than primitives, because string parsing
of a domain object in an annotation is a maintenance trap."

**What separates them:** framing it as *reviewability of the case list*, and stating
the case where it is the wrong tool.

**Interviewer's follow-up:** "Your parameterized test has a `switch` on the input
inside it. What is that telling you?" The answer: two tests wearing a trench coat.
Split it.

---

### Q4 — "This test is green. What's wrong with it?"

```java
@Test
void rejectsOversell() {
    try {
        inventoryService.reserve("SKU-1001", 5);
    } catch (InsufficientStockException e) {
        // expected
    }
}
```

**Mid-level answer:** "It should use `assertThrows`."

**Senior answer:** "It asserts nothing. If `reserve` returns normally, the try block
completes and the test passes — so this test stays green even if the stock check is
deleted entirely. It also gives 100% line coverage on the method, which is why
coverage as a metric is misleading here; a mutation test would show the mutant
surviving. The fix is `assertThatThrownBy(...).isInstanceOf(...)`, and then I'd
assert on a *field* of the exception — the SKU and the available quantity — rather
than on the message string, so improving the error copy doesn't break the build. As
a rule I treat any hand-written try/catch in a test as a defect on sight."

**What separates them:** connecting it to coverage being a liar (Topic 63), and
volunteering the message-string rule unprompted.

**Interviewer's follow-up:** "How would you find every test like this in a 4,000-test
codebase?" Good answers: a grep for `catch` in `src/test`, an ArchUnit rule, or
running PIT and looking at surviving mutants on well-covered lines.

---

### Q5 — "A test fails on CI with 'expected: true but was: false'. Walk me through your next hour."

**Mid-level answer:** "I'd check out the branch and run it locally with a debugger."

**Senior answer:** "First I'd note that the message is the actual problem: the test
used `assertTrue(a.equals(b))`, so the assertion library was handed a boolean and
has nothing left to report. Before debugging I'd change it to
`assertThat(a).isEqualTo(b)` — or `usingRecursiveComparison()` if it's an entity
with a hand-written `equals` — push, and let CI tell me which field differs. That
usually ends the investigation in one cycle instead of an hour of local
reproduction. Separately, if it passes locally and fails on CI, my next hypothesis
is order dependence or environment: I'd check for `@TestInstance(PER_CLASS)`, static
state, a default locale or timezone difference, and I'd turn on random method
ordering to see whether the failure is reproducible on demand."

**What separates them:** treating the *poor failure message* as the first bug to
fix, and having a named list of local-vs-CI hypotheses rather than reaching for a
debugger.

**Interviewer's follow-up:** "What if it only fails when the full suite runs, never
in isolation?" That is order dependence or shared static state, and the answer is
random ordering plus finding the shared mutable thing — not a retry.

---

## Mental model checkpoint

Reason these out without looking anything up.

1. JUnit creates a new test instance per method to make state leakage structurally
   impossible. Jest does the opposite and expects discipline. Argue for Jest's
   choice — what does the single-instance model make *easier*, and is that a
   sufficient reason?

2. `@BeforeAll` must be static, and so must a `@MethodSource` factory. Both
   constraints disappear under `@TestInstance(PER_CLASS)`. State the single
   underlying rule that explains all four facts in one sentence.

3. You have a test that fails only when the whole suite runs. List, in the order you
   would check them, four different causes — and for each, the one command or code
   change that would confirm or eliminate it.

4. `assertTrue(a.equals(b))` and `assertThat(a).isEqualTo(b)` test the same thing and
   differ only in the failure message. Yet one of them costs an engineer an hour.
   Generalise: what property should you look for in *any* assertion API to predict
   how expensive its failures will be?

5. There is no `test.only` in JUnit, and that is deliberate. Give the argument for
   removing it, then the strongest argument for having it, then say which you'd pick
   for a 4,000-test codebase and why.

6. `@ExtendWith` is a single mechanism that Spring, Mockito and Testcontainers all
   use. Before reading Topics 59–61: predict what each of those three extensions
   must do at `beforeAll` and at `beforeEach`, and which of the three has the most
   expensive `beforeAll`.

7. A colleague proposes a rule: "every test class must have a `@BeforeEach` that
   resets all state." Given the per-method lifecycle, what is that rule actually
   protecting against — and what is it likely to cost?

---

## Quick reference card

### Annotations

```java
@Test                              // a test method (org.junit.jupiter.api.Test !)
@DisplayName("readable name")      // report label; classes and methods
@Nested                            // inner (non-static) class = describe block
@BeforeEach / @AfterEach           // per test method
@BeforeAll  / @AfterAll            // per class; static unless PER_CLASS
@Disabled("reason")                // skip; the reason is mandatory in review
@Tag("slow")                       // select with -Dgroups=slow / -DexcludedGroups=slow
@Timeout(5)                        // fail after 5 seconds
@TestInstance(Lifecycle.PER_CLASS) // one instance for the class. Read Trap 1 first.
@TestMethodOrder(MethodOrderer.Random.class)
@ExtendWith(SomeExtension.class)   // the hook Spring/Mockito/Testcontainers use
```

### Parameterized sources

```java
@ParameterizedTest(name = "{0} -> {1}")
@ValueSource(ints = {1, 9, 10})              // one primitive arg
@ValueSource(strings = {"SKU-1001"})
@NullSource @EmptySource @NullAndEmptySource // added invocations
@CsvSource({"10, 500", "50, 1000"})          // multi-arg from strings
@CsvFileSource(resources = "/tiers.csv")     // from a test resource
@EnumSource(OrderStatus.class)               // one per enum constant
@MethodSource("scenarios")                   // static Stream<Arguments>; FQN if @Nested
```

### Assertions you will actually use

```java
import static org.assertj.core.api.Assertions.*;

assertThat(actual).isEqualTo(expected);
assertThat(actual).isNotNull().isInstanceOf(Order.class);
assertThat(total).isEqualTo(4_797L);
assertThat(list).hasSize(3).containsExactly(a, b, c);
assertThat(list).extracting(OrderLine::sku).containsExactlyInAnyOrder("A", "B");
assertThat(map).containsEntry("SKU-1001", 12);
assertThat(optional).contains(order);
assertThat(actual).usingRecursiveComparison().ignoringFields("id").isEqualTo(expected);
assertThatThrownBy(() -> svc.call()).isInstanceOf(InsufficientStockException.class);
assertThatNoException().isThrownBy(() -> svc.call());

import static org.junit.jupiter.api.Assertions.*;
var ex = assertThrows(InsufficientStockException.class, () -> svc.call());
assertAll(() -> assertThat(a).isEqualTo(1), () -> assertThat(b).isEqualTo(2));
```

### Commands

```bash
mvn test                                   # everything
mvn -Dtest=OrderTotalsTest test            # one class
mvn -Dtest=OrderTotalsTest#methodName test # one method  (your test.only)
mvn -Dtest='Order*Test' test               # a pattern
mvn -Dgroups=slow test                     # by @Tag
mvn -DexcludedGroups=slow test
mvn surefire:test -Dsurefire.reportFormat=plain
mvn test -Dsurefire.useFile=false          # print output to the console
mvn dependency:tree -Dincludes=org.junit.jupiter:*
```

### Gotchas checklist

- [ ] Import `org.junit.jupiter.api.Test`, never `org.junit.Test`.
- [ ] `@BeforeAll` / `@AfterAll` / `@MethodSource` are `static` by default.
- [ ] `@Nested` classes are non-static inner classes.
- [ ] `@MethodSource` inside a `@Nested` class needs the fully-qualified reference.
- [ ] `@ParameterizedTest` replaces `@Test`; never both.
- [ ] Always give `@ParameterizedTest` a `name = "..."` pattern.
- [ ] Never hand-write try/catch in a test — use `assertThatThrownBy`.
- [ ] Never assert on an exception's message string; assert on its fields.
- [ ] Never `assertTrue(a.equals(b))` — `assertThat(a).isEqualTo(b)`.
- [ ] `@TestInstance(PER_CLASS)` means your fields are now shared. Reset them.
- [ ] `assumeTrue` skips; `assertTrue` fails. Know which you meant.

---

## [BOOT 3.x DELTA]

Almost nothing in this topic changes between Boot 3.x and Boot 4.1 — JUnit Jupiter's
programming model is stable and the annotations are identical. Three practical
differences worth knowing when you land in a 3.x codebase:

1. **The managed JUnit version differs.** Boot 3.x and Boot 4.x manage different
   JUnit lines through their respective BOMs. The annotations in this document work
   on both. Never pin `junit-jupiter` yourself in a Boot project — let the BOM do it,
   and check with `mvn dependency:tree -Dincludes=org.junit.jupiter:*`.

2. **`junit-vintage-engine` is more likely to be present in an older codebase**, and
   with it, a JUnit 4 `org.junit.Test` import runs silently. If tests behave
   inconsistently, check for two engines on the classpath.

3. **Everything about the *lifecycle* is unchanged.** Per-method instances,
   `@TestInstance(PER_CLASS)`, `@ParameterizedTest`, the extension model — all
   identical across 3.x and 4.x. What changes between the lines is Spring's *test
   support* (Topic 60), not JUnit itself.

---

## When would I use this at work?

**1. Reviewing a pull request with six nearly-identical test methods.**
You ask for a `@CsvSource` — not to save lines, but because the table makes it
immediately visible that the boundary at 50 units is tested from one side only. That
review comment finds a real off-by-one about one time in four.

**2. Diagnosing a "flaky" test on the first day of a new job.**
Everyone else has been retrying it for months. You check for
`@TestInstance(PER_CLASS)` and static fields, turn on random method ordering, and
make it fail reliably in one afternoon. Reliable failure is 90% of the fix.

**3. Deciding where a piece of logic should live.**
You want to test `orderflow`'s discount tiers. If that logic sits inside a
`@Transactional` service method that also loads entities, the only way to test it is
a slow integration test. If it is a static method over a record, it is a
microsecond-fast parameterized test. **The testability of a design is visible before
you write the test**, and noticing that is the habit this topic is really teaching.

---

## Connected topics

**Prerequisites:**
- **03 — Access modifiers**: why test classes and methods are package-private.
- **08 / 09 — Exceptions**: `assertThatThrownBy` is only useful if your exception
  hierarchy carries structured data. Trap 2 is a direct consequence of Topic 09.
- **13 — equals/hashCode**: `isEqualTo` calls `equals`. Testing an entity means
  testing its `equals` unless you use `usingRecursiveComparison`.
- **23 — Streams**: `@MethodSource` returns a `Stream<Arguments>`.
- **27 — Records**: the ideal shape for test fixtures and for the code under test.
- **31 — Maven**: Surefire runs `*Test`; Failsafe runs `*IT`. That split becomes
  important in Topic 61.

**This unlocks:**
- **59 — Mockito**: `MockitoExtension` is an `@ExtendWith`. The whole "no module
  mocking" consequence starts here and is resolved there.
- **60 — Spring test slices**: `SpringExtension` is an `@ExtendWith` with a very
  expensive `beforeAll`, and understanding the extension model is what lets you
  reason about context caching.
- **61 — Testcontainers**: `@Testcontainers` is an `@ExtendWith`. Static vs instance
  `@Container` fields is *exactly* the `PER_METHOD` vs `PER_CLASS` distinction from
  this topic, with a Docker container attached to the cost.
- **62 — Contract testing**: generated contract tests run on the JUnit Platform.
- **63 — Mutation testing**: Trap 4's green-but-useless test is precisely what PIT
  detects. Read Trap 4 again before that topic.
- **64 — Property-based testing**: jqwik is a *different engine* on the same JUnit
  Platform — the Platform/Jupiter split from "What is this?" is why that works.
- **99 — jcstress**: the honest answer to "can a JUnit test prove my concurrency is
  correct?" is no, and jcstress is what can.

---

*Java baseline 21, running on JDK 25. The JUnit Jupiter programming model shown here
is stable across the 5.x and 6.x lines and across Boot 3.x and 4.x — the annotations,
the lifecycle and the extension model have not changed. The one thing to verify
against your own build rather than against this document is the exact managed
version, via `mvn dependency:tree`.*
