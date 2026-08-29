# 63 — Mutation Testing with PIT: The Honest Measure of a Suite

## Phase: 6 — Testing
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow` gets a mutation-tested pricing and inventory core. You will find a real surviving boundary mutant in the order-total calculation and write the test that kills it — that test is the deliverable, not the score.

---

## Mechanical statement

> **PIT modifies your compiled bytecode — flips a conditional, removes a call,
> changes a return value — and re-runs only the tests that cover the mutated line.
> A mutant that SURVIVES is a line your suite executes but does not verify.**

Everything else in this document is detail on that sentence. Read it twice.

Note what it does *not* say. It does not say "a survived mutant is a bug". It does
not say "your code is wrong". It says something narrower and much more useful:
**that line ran, and nothing noticed when its behaviour changed.**

---

## The bridge from what you know

### The bridge that actually matters

You already distrust coverage numbers. Everybody who has shipped code does. You have
sat in a meeting where somebody said "we're at 87%" and felt the specific irritation
of knowing that the number is not evidence of anything, while not being able to say
precisely why in a way that wins the argument.

Here is why, precisely: **line coverage measures execution. It does not measure
assertion.**

```java
@Test
void calculatesOrderTotal() {
    calculator.total(order);          // 100% line coverage of total()
}                                     // zero assertions
```

That test covers every line of `total()`. Delete the body of `total()` and replace it
with `return 0;` and the test still passes. Coverage tools report a green bar.

Mutation testing is the instrument that turns your intuition into a number you can
put in a pull request. That is the whole bridge. You do not need a new belief; you
need a measurement for one you already hold.

### The JS analogue

**Stryker Mutator** is the direct equivalent, and if you have used it, PIT will feel
familiar — same concept, same report shape, same "why is this so slow" experience.
Assume you have not used it; almost nobody has, because Stryker on a large JS
codebase is punishing and teams give up.

| | Stryker (JS) | PIT (JVM) |
|---|---|---|
| What it mutates | Source, then re-transpiles | **Compiled bytecode**, in memory |
| Cost of producing one mutant | A transpile step | An ASM byte-array rewrite — microseconds |
| Test selection | Improving, historically coarse | **Coverage-driven from the start**: only tests that touch the mutated line |
| Practical verdict | Often too slow to keep in CI | Fast enough to run on selected modules every build |

The bytecode point is the reason PIT is usable and Stryker frequently is not. Java
compiles to a stable, well-specified binary format that a library (ASM) can rewrite
in place. There is no re-compilation per mutant. This is one of the genuine
ergonomic wins of a bytecode platform, and it is worth knowing *why* your Java tool
is better than your JS tool here rather than just noticing that it is.

### What does not transfer

- **You cannot mutate what you cannot see.** PIT mutates *your* classes, from your
  `target/classes`. It will not mutate Spring, Hibernate or the JDK, and you would
  not want it to.
- **Java has no module mocking** (Topic 58), so Java suites lean harder on real
  objects — which, pleasantly, makes them mutation-test better than a JS suite full
  of `jest.mock` where half the assertions are about calls that never happened.

---

## What is this?

A **mutant** is a copy of your program with one small, deliberate change.

PIT generates thousands of them, runs your tests against each, and records what
happened:

| Status | Meaning | What you should feel |
|---|---|---|
| **KILLED** | At least one test failed against the mutant | Good. The suite noticed. |
| **SURVIVED** | Every covering test still passed | **This is the finding.** A line runs unverified. |
| **NO_COVERAGE** | No test executes that line at all | A coverage gap. Different problem, cheaper to fix. |
| **TIMED_OUT** | The mutant made a test hang; PIT killed it | Counts as killed. Usually a mutated loop condition. |
| **NON_VIABLE** | The mutated bytecode failed verification | PIT discards it. Not your problem. |
| **MEMORY_ERROR** / **RUN_ERROR** | The mutant blew up the JVM or the harness | Investigate if there are many; usually a config issue. |

**Mutation score** (PIT calls it *mutation coverage*):

```
mutation score = KILLED / (all generated mutants)
```

**Test strength** — the more diagnostic number:

```
test strength = KILLED / (KILLED + SURVIVED)          [ignores NO_COVERAGE]
```

Mutation score conflates two different failures: "we never test this" and "we test
this badly". Test strength isolates the second. When you have 40% line coverage and
90% test strength, the tests you *have* are excellent and you simply have not written
enough of them — a completely different remediation than 95% coverage and 45% test
strength, which means your assertions are decorative.

---

## Why does it matter?

**1. It finds the tests that assert nothing.**
Every codebase has them. The "characterisation test" someone wrote to hit a coverage
gate. The test whose assertions were commented out during a flaky patch and never
restored. The test that asserts `assertThat(result).isNotNull()`. Coverage rewards
all of these. PIT reports every one as a wall of surviving mutants.

**2. It finds boundaries nobody tested.**
`>` versus `>=` is the most common off-by-one in commercial software, and it is
almost always in pricing, quota or eligibility logic. It is invisible to coverage
(both branches ran) and invisible to review (the code looks right). PIT's
`CONDITIONALS_BOUNDARY` mutator finds it by construction: it changes `>=` to `>` and
asks whether anybody notices.

**3. It changes how you write tests, which is why it is here and not in Phase 11.**
Once you have watched a mutant survive, you stop writing `assertThat(total).isNotNull()`
forever. You start testing at boundaries by reflex. The master plan puts this topic
in Phase 6 deliberately: every test you write in Phases 8 through 11 is better
because you did this now.

**4. It is a differentiator in interviews.**
"We have 85% coverage" is a mid-level sentence. "Coverage measures execution, not
assertion; we run PIT on pricing, inventory and wallet because that is where a silent
bug costs money, and we gate on test strength rather than mutation score" is a senior
sentence, and it takes about ten seconds to say.

---

## Machine-level reality

### How a mutant is made

PIT loads your compiled `.class` files and rewrites them with **ASM**, a bytecode
manipulation library. No source is read. No recompilation happens. One mutant is one
byte-array transformation, produced in microseconds and handed to a classloader.

Take this source:

```java
boolean qualifiesForBulkDiscount(int units) {
    return units >= 10;
}
```

`javac` produces roughly this (Topic 76 teaches you to read it properly):

```
  iload_1              // push units
  bipush 10            // push 10
  if_icmplt L1         // if units < 10, jump to "return false"
  iconst_1
  ireturn              // return true
L1:
  iconst_0
  ireturn              // return false
```

The `CONDITIONALS_BOUNDARY` mutator changes exactly one opcode:

```
  if_icmplt L1    ->   if_icmple L1
```

`if_icmplt` is "jump if less than". `if_icmple` is "jump if less than **or
equal**". One byte. The method now behaves as `units > 10`. That mutant is loaded
into a fresh classloader, your covering tests run, and PIT records whether any of
them failed.

This is why mutation testing on the JVM is practical: the unit of work is an opcode
swap, not a build.

### The default mutation operators

These are the ones PIT enables by default. Learn what each one *asks*.

| Mutator | What it does | The question it asks |
|---|---|---|
| `CONDITIONALS_BOUNDARY` | `<` ↔ `<=`, `>` ↔ `>=` | "Did you test the boundary, or just either side of it?" |
| `NEGATE_CONDITIONALS` | `==` → `!=`, `<` → `>=`, etc. | "Does this branch matter at all?" |
| `MATH` | `+` → `-`, `*` → `/`, `%` → `*` … | "Is the arithmetic asserted, or just executed?" |
| `INCREMENTS` | `i++` → `i--` | "Does anything observe this counter?" |
| `INVERT_NEGS` | `-x` → `x` | "Is the sign checked?" |
| `VOID_METHOD_CALLS` | Deletes a call to a `void` method | "Does anyone verify this side effect happened?" |
| `EMPTY_RETURNS` | Returns `""`, `0`, empty collection, `Optional.empty()` | "Is the return value inspected?" |
| `FALSE_RETURNS` / `TRUE_RETURNS` | `return false` / `return true` | Same, for booleans |
| `NULL_RETURNS` | `return null` | "Would a null here be noticed?" |
| `PRIMITIVE_RETURNS` | Returns `0` for a numeric return | Same, for numbers |

Additional groups you can opt into with `<mutators>`:

- `STRONGER` — default plus `REMOVE_CONDITIONALS` (make a branch always taken) and
  a few others. A reasonable step up for a small, high-value module.
- `ALL` — everything, including experimental operators. Produces a lot of noise and
  a lot of equivalent mutants. Do not start here.
- Named groups: `DEFAULTS`, `RETURNS`, `OLD_DEFAULTS`. You can also list individual
  mutators.

**The judgement:** start with `DEFAULTS`. Move a specific module to `STRONGER` when
its default mutation score is already high and you want more pressure. `ALL` is a
research setting, not a CI setting.

### Why PIT is slow, and the two things that make it fast

Naively, the cost is `mutants × tests`. For a real module that is millions of test
executions. PIT does not do that.

**1. Coverage-driven test selection.**
PIT first runs your entire suite once under a coverage agent, building a map of
`line → tests that execute it`. For each mutant it then runs **only** the tests that
touch the mutated line, ordered fastest-first, and **stops at the first failure**.
Most mutants die on the first or second test. This turns a quadratic problem into
something close to linear in the number of mutants.

The consequence you must internalise: **a test that touches a line is what gives PIT
the chance to kill a mutant there.** If your only coverage of `OrderTotalCalculator`
comes from a slow `@SpringBootTest`, PIT will run that slow test once per mutant.
This is the single biggest cause of "PIT takes forty minutes".

**2. Forked JVMs ("minions") and timeouts.**
PIT forks worker JVMs to run mutants in isolation, because a mutant can corrupt
static state or hang. Hangs are handled by a timeout computed as:

```
timeout = timeoutConstant + (timeoutFactor × normal test duration)
```

Defaults are around 4000 ms and 1.25. A mutated loop condition producing an infinite
loop is caught by this and recorded as `TIMED_OUT`, which counts as killed — which is
correct, because a test that would hang forever is a test that noticed.

If you see many `TIMED_OUT` results on tests that are legitimately slow, raise
`timeoutConstant`. Do not raise it reflexively: a wall of timeouts on a fast module
usually means a mutated loop, which is a real kill.

### Scoping — the three settings that decide whether this is usable

```xml
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <!-- version: let a BOM or dependencyManagement own it; check the current
       release with: mvn versions:display-plugin-updates -->
  <dependencies>
    <dependency>
      <groupId>org.pitest</groupId>
      <artifactId>pitest-junit5-plugin</artifactId>
    </dependency>
  </dependencies>
  <configuration>
    <targetClasses>
      <param>com.orderflow.pricing.*</param>
      <param>com.orderflow.inventory.*</param>
      <param>com.orderflow.wallet.*</param>
    </targetClasses>
    <targetTests>
      <param>com.orderflow.pricing.*Test</param>
      <param>com.orderflow.inventory.*Test</param>
      <param>com.orderflow.wallet.*Test</param>
    </targetTests>
    <excludedClasses>
      <param>com.orderflow.*.*Dto</param>
      <param>com.orderflow.*.*Config</param>
    </excludedClasses>
    <excludedTestClasses>
      <param>com.orderflow.**.*IntegrationTest</param>
    </excludedTestClasses>
    <avoidCallsTo>
      <param>org.slf4j</param>
      <param>java.util.logging</param>
    </avoidCallsTo>
    <threads>4</threads>
    <timeoutConstant>6000</timeoutConstant>
    <timestampedReports>false</timestampedReports>
    <outputFormats>
      <param>HTML</param>
      <param>XML</param>
    </outputFormats>
    <mutationThreshold>80</mutationThreshold>
    <coverageThreshold>70</coverageThreshold>
    <withHistory>true</withHistory>
  </configuration>
</plugin>
```

| Setting | Why it exists |
|---|---|
| `targetClasses` | **The most important line in the file.** Mutating everything is how PIT gets abandoned. Mutate where a silent bug costs money. |
| `targetTests` | Restricts which tests PIT even considers. Leave this wide and PIT will pull your Testcontainers suite into every mutant run. |
| `excludedTestClasses` | Explicitly keep integration tests out. A Postgres-backed test that takes 3 s is a catastrophe as a mutation-killing test. |
| `avoidCallsTo` | PIT will not mutate calls to these packages. Logging is the canonical case — nobody asserts on log lines, so every logging mutant survives and pollutes the report. |
| `threads` | Parallel minions. Set it to your core count minus one. Free speed. |
| `timestampedReports` | `false` writes to a stable `target/pit-reports/` path instead of a timestamped subdirectory. Set it false so CI can archive a predictable path. |
| `mutationThreshold` | Fail the build below this mutation score. **Only set this once a module is already above it** — see Measurement. |
| `withHistory` | Incremental analysis. |

### Incremental analysis, honestly

`withHistory=true` makes PIT write a history file (default under `target/`) recording
each mutant's status plus hashes of the class and its covering tests. On the next
run, mutants whose class and covering tests are both unchanged are **not re-run**;
their previous status is reused.

This is a genuine order-of-magnitude improvement for local development. Two caveats
you must know:

- **The history file must persist between runs.** In CI that means caching it as a
  build artefact keyed on the branch. `mvn clean` deletes it. If you `clean` every
  build, `withHistory` does nothing and you will believe incremental analysis
  "doesn't work".
- **It is a cache, so it can be wrong.** A change in a dependency that is not part of
  the hash (a properties file, a transitive library version) can leave a stale
  status. For a release build, run without history.

Set the paths explicitly if you want CI control:

```xml
<historyInputFile>${project.basedir}/.pit-history</historyInputFile>
<historyOutputFile>${project.basedir}/.pit-history</historyOutputFile>
```

> **Changed-code-only runs.** The obvious next want is "only mutate lines changed in
> this PR". That capability exists through **arcmutate**, a commercial add-on from
> PIT's author, not in the open-source plugin. Say that plainly in a design
> discussion rather than promising a feature that is not there. The open-source
> approximation is `withHistory` plus a narrow `targetClasses`.

### Equivalent mutants — why 100% is not the goal

Some mutants are **semantically identical** to the original. No test can kill them,
because no observable behaviour differs.

```java
for (int i = 0; i < items.size(); i++) { ... }
```

If the loop body is order-independent and the collection is a set of unique ids, some
mutations of the index arithmetic produce a program that is different in structure
and identical in behaviour. Similarly:

```java
if (cache.size() > MAX_ENTRIES) { evictOldest(); }
```

Mutating `>` to `>=` changes when eviction happens by exactly one entry. If nothing
observable depends on that, the mutant is equivalent-in-practice and killing it
requires a test that asserts an implementation detail.

**Deciding whether a mutant is equivalent is undecidable in general.** That is a
theorem, not a tooling gap. The practical consequence:

> Never set a mutation threshold at 100. A team chasing the last 3% writes tests that
> assert implementation details, and those tests make refactoring impossible — which
> is a bigger cost than the mutants.

---

## Example 1 — minimal

A single method with a single boundary.

```java
package com.orderflow.pricing;

public final class BulkDiscountPolicy {

    private static final int BULK_THRESHOLD = 10;
    private static final int BULK_PERCENT   = 15;

    /** Returns the discount percentage to apply for a given line quantity. */
    public int discountPercentFor(int units) {
        if (units >= BULK_THRESHOLD) {
            return BULK_PERCENT;
        }
        return 0;
    }
}
```

The test somebody wrote:

```java
class BulkDiscountPolicyTest {

    private final BulkDiscountPolicy policy = new BulkDiscountPolicy();

    @Test
    void appliesDiscountForLargeOrders() {
        assertThat(policy.discountPercentFor(25)).isEqualTo(15);
    }

    @Test
    void noDiscountForSmallOrders() {
        assertThat(policy.discountPercentFor(3)).isEqualTo(0);
    }
}
```

**Line coverage: 100%.** Both branches execute. Both tests assert real values. A
reviewer would approve this without a second thought.

Now run PIT. `CONDITIONALS_BOUNDARY` changes `units >= 10` to `units > 10`.

- `discountPercentFor(25)` → still 15. Test passes.
- `discountPercentFor(3)` → still 0. Test passes.

**The mutant survives.** The line `if (units >= BULK_THRESHOLD)` is executed by two
tests and verified by neither, *at the point where it actually decides anything*.

In production, an order of exactly 10 units gets no discount. Customers who order a
box of twelve are fine. Customers who order exactly ten — which is a very common
quantity, because humans order round numbers — are quietly overcharged. Nobody
notices for months, and then finance notices.

The test that kills it:

```java
    @Test
    void appliesDiscountAtExactlyTheThreshold() {
        assertThat(policy.discountPercentFor(10)).isEqualTo(15);   // kills >= to >
    }

    @Test
    void noDiscountJustBelowTheThreshold() {
        assertThat(policy.discountPercentFor(9)).isEqualTo(0);     // kills >= to >=-1 style shifts
    }
```

Note what changed. Not more tests of the same kind — **tests at the boundary**. That
reflex is the actual deliverable of this topic.

---

## Example 2 — production scenario on the project spine

`orderflow`'s order total. This is the class where a silent defect costs real money,
and it is the class the failure drill uses.

```java
package com.orderflow.pricing;

import java.util.List;

public final class OrderTotalCalculator {

    private static final int  BULK_THRESHOLD_UNITS  = 10;
    private static final int  BULK_DISCOUNT_PERCENT = 15;
    private static final long FREE_SHIPPING_MINOR   = 5_000L;   // GBP 50.00
    private static final long SHIPPING_FLAT_MINOR   =   499L;
    private static final long MAX_WALLET_MINOR      = 20_000L;

    private final VatRates vatRates;

    public OrderTotalCalculator(VatRates vatRates) {
        this.vatRates = vatRates;
    }

    public OrderTotal calculate(List<OrderLine> lines, Country shipTo, long walletBalanceMinor) {

        long subtotalMinor = 0L;
        for (OrderLine line : lines) {
            long lineMinor = line.unitPriceMinor() * line.quantity();
            if (line.quantity() >= BULK_THRESHOLD_UNITS) {                 // (A)
                lineMinor -= (lineMinor * BULK_DISCOUNT_PERCENT) / 100;    // (B)
            }
            subtotalMinor += lineMinor;                                    // (C)
        }

        long shippingMinor = subtotalMinor >= FREE_SHIPPING_MINOR          // (D)
                ? 0L
                : SHIPPING_FLAT_MINOR;

        long vatMinor = (subtotalMinor * vatRates.percentFor(shipTo)) / 100;  // (E)

        long grandTotalMinor = subtotalMinor + shippingMinor + vatMinor;   // (F)

        long walletAppliedMinor = Math.min(                                // (G)
                Math.min(walletBalanceMinor, MAX_WALLET_MINOR),
                grandTotalMinor);

        long payableMinor = grandTotalMinor - walletAppliedMinor;          // (H)

        auditLog.recordPricing(subtotalMinor, vatMinor, payableMinor);     // (I) void call

        return new OrderTotal(subtotalMinor, shippingMinor, vatMinor,
                              walletAppliedMinor, payableMinor);
    }
}
```

Money is `long` minor units throughout, per Topic 01. Good. Now count the places PIT
will attack:

| Line | Mutators that fire | The defect it would ship |
|---|---|---|
| (A) | `CONDITIONALS_BOUNDARY`, `NEGATE_CONDITIONALS` | Bulk discount off by one unit, or inverted entirely |
| (B) | `MATH` (`-`→`+`, `*`→`/`, `/`→`*`) | Discount *added* instead of subtracted |
| (C) | `MATH` | Subtotal wrong for multi-line orders only |
| (D) | `CONDITIONALS_BOUNDARY` | An order at exactly GBP 50.00 charged shipping |
| (E) | `MATH` | VAT wrong by a factor |
| (F) | `MATH` | Shipping or VAT silently dropped |
| (G) | `MATH`, `PRIMITIVE_RETURNS` on `Math.min` inlining | Wallet over-applied; customer pays nothing |
| (H) | `MATH` | Payable = total + wallet |
| (I) | `VOID_METHOD_CALLS` | Audit record never written |

The typical existing test suite for this class:

```java
class OrderTotalCalculatorTest {

    @Test
    void calculatesASimpleOrder() {
        var total = calculator.calculate(
                List.of(line(2_999L, 1)), Country.GB, 0L);
        assertThat(total.payableMinor()).isEqualTo(4_098L);
    }

    @Test
    void appliesBulkDiscount() {
        var total = calculator.calculate(
                List.of(line(2_999L, 20)), Country.GB, 0L);
        assertThat(total.payableMinor()).isGreaterThan(0L);      // <-- weak
    }

    @Test
    void appliesWalletBalance() {
        var total = calculator.calculate(
                List.of(line(2_999L, 1)), Country.GB, 1_000L);
        assertThat(total.payableMinor()).isNotNull();            // <-- worthless
    }
}
```

Line coverage of `calculate` is **100%**. Every line executes. And:

- `appliesBulkDiscount` asserts `isGreaterThan(0)`, which is true for the original,
  true if the discount is doubled, true if it is added instead of subtracted, and
  true if it is not applied at all. Every mutant on lines (A) and (B) survives it.
- `appliesWalletBalance` asserts `isNotNull()` on a primitive-backed accessor.
  It cannot fail. Every mutant on (G) and (H) survives.
- No test uses exactly 10 units, so (A)'s boundary mutant survives.
- No test has a subtotal of exactly 5000, so (D)'s boundary mutant survives.
- No test verifies the audit call, so (I)'s `VOID_METHOD_CALLS` mutant survives.
- Only one test has more than one line, so some `MATH` mutants on (C) may survive.

**Coverage says 100%. The suite verifies perhaps half of what this method decides.**
That gap — between a green coverage bar and the actual set of decisions your tests
constrain — is the thing this topic exists to make visible.

The rewritten suite, driven by the surviving mutants:

```java
class OrderTotalCalculatorTest {

    private final AuditLog audit = mock(AuditLog.class);
    private final OrderTotalCalculator calculator =
            new OrderTotalCalculator(new FixedVatRates(20), audit);

    @Test
    void bulkDiscountAppliesAtExactlyTenUnits() {                 // kills (A) boundary
        var total = calculator.calculate(List.of(line(1_000L, 10)), Country.GB, 0L);
        assertThat(total.subtotalMinor()).isEqualTo(8_500L);      // 10_000 - 15%
    }

    @Test
    void bulkDiscountDoesNotApplyAtNineUnits() {                  // kills (A) negation
        var total = calculator.calculate(List.of(line(1_000L, 9)), Country.GB, 0L);
        assertThat(total.subtotalMinor()).isEqualTo(9_000L);
    }

    @Test
    void bulkDiscountReducesRatherThanIncreases() {               // kills (B) MATH
        var nine = calculator.calculate(List.of(line(1_000L, 9)),  Country.GB, 0L);
        var ten  = calculator.calculate(List.of(line(1_000L, 10)), Country.GB, 0L);
        assertThat(ten.subtotalMinor()).isLessThan(nine.subtotalMinor() + 1_000L);
    }

    @Test
    void shippingIsFreeAtExactlyTheThreshold() {                  // kills (D) boundary
        var total = calculator.calculate(List.of(line(5_000L, 1)), Country.GB, 0L);
        assertThat(total.shippingMinor()).isZero();
    }

    @Test
    void shippingIsChargedOnePennyBelowTheThreshold() {           // kills (D) negation
        var total = calculator.calculate(List.of(line(4_999L, 1)), Country.GB, 0L);
        assertThat(total.shippingMinor()).isEqualTo(499L);
    }

    @Test
    void subtotalIsTheSumOfAllLines() {                           // kills (C) MATH
        var total = calculator.calculate(
                List.of(line(1_000L, 1), line(2_500L, 2), line(300L, 3)), Country.GB, 0L);
        assertThat(total.subtotalMinor()).isEqualTo(1_000L + 5_000L + 900L);
    }

    @Test
    void walletIsCappedAtTheMaximum() {                           // kills (G)
        var total = calculator.calculate(List.of(line(30_000L, 1)), Country.GB, 100_000L);
        assertThat(total.walletAppliedMinor()).isEqualTo(20_000L);
    }

    @Test
    void walletNeverExceedsTheGrandTotal() {                      // kills (G), (H)
        var total = calculator.calculate(List.of(line(1_000L, 1)), Country.GB, 100_000L);
        assertThat(total.payableMinor()).isZero();
        assertThat(total.walletAppliedMinor()).isEqualTo(total.grandTotalMinor());
    }

    @Test
    void pricingIsAudited() {                                     // kills (I) VOID_METHOD_CALLS
        calculator.calculate(List.of(line(1_000L, 1)), Country.GB, 0L);
        verify(audit).recordPricing(anyLong(), anyLong(), anyLong());
    }
}
```

Nine focused tests instead of three vague ones. Note that the *last* one is the only
`verify` in the file — Topic 59's rule still applies, and PIT's `VOID_METHOD_CALLS`
mutator is one of the few places where an interaction assertion is genuinely the
right tool, because a side effect with no return value has no other observable.

> **Note for the drill:** `walletNeverExceedsTheGrandTotal` is the kind of statement
> that wants to be a *property*, not an example — "for any balance and any set of
> lines, `walletApplied <= grandTotal`". That is Topic 64, and mutation testing is
> how you find out which properties are worth stating.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — 85% line coverage with the critical branch never asserted

**Wrong:** the team gates CI on JaCoCo at 80% line coverage and nothing else. The
pricing module reports 91%.

**Exact symptom:** a developer changes `subtotalMinor >= FREE_SHIPPING_MINOR` to
`>` while refactoring a constant, and the full test suite passes. It ships. Two weeks
later, a support ticket: "I was told shipping is free over £50, I spent exactly £50
and was charged £4.99." Nobody can reproduce it because everyone tests with £60. The
JaCoCo report for that line is bright green in the commit that broke it.

**Root cause:** JaCoCo instruments for **execution**. The line executed; therefore it
is covered; therefore the metric is satisfied. Coverage cannot distinguish a test
that asserts the exact total from a test that asserts the total is not null. Those
two tests produce identical coverage reports and radically different safety.

**Fix:** run PIT on the module and read the surviving mutants. Then, importantly,
**change what you gate on**:

```xml
<mutationThreshold>80</mutationThreshold>
```

applied to `com.orderflow.pricing.*` only. Keep JaCoCo if you like — it is fast and
finds untested files — but stop treating it as evidence of quality. The sentence to
use in the design review is: *coverage is a lower bound on how bad the suite could
be; mutation score is a measurement of how good it actually is.*

---

### Trap 2 — PIT on the whole codebase, then abandoned

**Wrong:**

```xml
<targetClasses><param>com.orderflow.*</param></targetClasses>
<targetTests><param>com.orderflow.*</param></targetTests>
```

**Exact symptom:** the first run takes somewhere between forty minutes and "we killed
it after two hours". The report contains around 20,000 mutants, most of them in
generated DTOs, Spring configuration classes, `toString()` methods and logging
statements. Nobody reads it. Within two sprints the PIT profile is behind a
`-Ppitest` flag that nobody activates, and by the quarter it is deleted.

**Root cause:** two compounding errors. First, mutating everything means mutating
code where a survived mutant carries no information — nobody has ever been paged
because a `toString()` was wrong. Second, `targetTests` wide open means PIT includes
your `@SpringBootTest` and Testcontainers classes as covering tests, so each mutant
run pays a Spring context startup or a Postgres round trip. That is where the forty
minutes goes.

**Fix:** scope aggressively, then expand only if it stays fast.

```xml
<targetClasses>
  <param>com.orderflow.pricing.*</param>
  <param>com.orderflow.inventory.*</param>
  <param>com.orderflow.wallet.*</param>
</targetClasses>
<targetTests>
  <param>com.orderflow.pricing.*Test</param>
  <param>com.orderflow.inventory.*Test</param>
  <param>com.orderflow.wallet.*Test</param>
</targetTests>
<excludedTestClasses>
  <param>com.orderflow.**.*IntegrationTest</param>
  <param>com.orderflow.**.*ContractTest</param>
</excludedTestClasses>
<avoidCallsTo><param>org.slf4j</param></avoidCallsTo>
```

**The rule:** mutate the modules where a silent defect costs money — pricing,
inventory, wallet, tax, fraud thresholds. Do not mutate controllers, mappers, DTOs,
config classes, or anything whose bugs are loud. A five-minute PIT run that people
read beats a forty-minute one they do not.

---

### Trap 3 — the assertion-free test, discovered by a wall of survivors

**Wrong:**

```java
@Test
void placesAnOrder() {
    OrderResult result = orderService.place(command);
    assertThat(result).isNotNull();
}
```

Written to close a coverage gap during a release crunch. Perfectly ordinary. Present
in every codebase.

**Exact symptom:** PIT's report for that class shows a solid block of `SURVIVED`
mutants, including `NULL_RETURNS` and `EMPTY_RETURNS` on methods this test is the only
coverage for. Test strength for the class is near zero while line coverage is near
100%. The visual in the HTML report is unmistakable once you have seen it: red
markers on every line the test touches.

**Root cause:** `isNotNull()` on a non-null-returning method is a tautology. So is
`assertThat(list).isNotEmpty()` when the method always returns at least one element,
and `assertDoesNotThrow(...)` around a method that does not throw. These are
assertions in syntax only.

**Fix:** assert the value, not its existence.

```java
@Test
void placesAnOrderWithTheCorrectTotalAndStatus() {
    OrderResult result = orderService.place(command);

    assertThat(result.status()).isEqualTo(OrderStatus.CONFIRMED);
    assertThat(result.payableMinor()).isEqualTo(8_997L);
    assertThat(result.lines()).hasSize(1);
}
```

**Use PIT as a review tool for this specifically.** When a PR adds tests, a
survived-mutant count that did not go down means the new tests execute code without
constraining it. That is a much more actionable review comment than "please add more
assertions".

---

### Trap 4 — chasing 100% and killing equivalent mutants

**Wrong:** the mutation score on `InventoryReservationService` is 94%. Someone sets
`<mutationThreshold>100</mutationThreshold>` and spends two days on the remaining six.

**Exact symptom:** the new tests look like this:

```java
@Test
void evictsWhenCacheIsExactlyAtCapacity() {
    fill(cache, MAX_ENTRIES);
    cache.put(extraKey, value);
    assertThat(cache.size()).isEqualTo(MAX_ENTRIES);      // asserting an internal
}
```

Three weeks later someone changes the cache from a hand-rolled map to Caffeine
(Topic 15), the behaviour is identical from every caller's point of view, and eleven
tests fail. The refactor is abandoned. The team concludes "our tests are too
brittle", which is true, and blames mutation testing, which is fair.

**Root cause:** **equivalent mutants**. Some mutations produce a program with
identical observable behaviour. Killing them requires asserting on something that is
not observable behaviour — that is, an implementation detail. Detecting equivalence
automatically is undecidable, so the tool cannot filter them for you.

**Fix:**

- Set thresholds at a level you have already achieved, and raise them by small
  increments only when a real improvement earns it. 80 is a defensible number for a
  pricing module. 100 is not a defensible number for anything.
- When you decide a specific mutant is equivalent, **record that decision** —
  a comment in the test class or a line in the module's README naming the mutant and
  why. Otherwise the next person re-litigates it.
- `avoidCallsTo` for logging removes a whole class of pseudo-equivalent noise up
  front.

---

### Trap 5 — mutation score computed over code with no tests at all

**Wrong:** a new module reports a mutation score of 12% and the team panics about
test quality.

**Exact symptom:** the report shows a huge `NO_COVERAGE` count and a small
`SURVIVED` count. Meanwhile **test strength** is 88%.

**Root cause:** mutation score = `KILLED / all mutants`, and `NO_COVERAGE` mutants
are in the denominator. A module with excellent tests covering 20% of its classes
scores terribly on mutation coverage and superbly on test strength.

**Fix:** read both numbers, and know what each is telling you.

| Line coverage | Test strength | Diagnosis | Action |
|---|---|---|---|
| High | High | Genuinely well tested | Move on |
| High | **Low** | **Tests execute but do not verify** | Fix assertions. This is the dangerous quadrant — it looks safest and is not. |
| Low | High | Too few tests, but the ones you have are real | Write more tests of the same kind |
| Low | Low | Untested | Start with coverage, then come back |

The high-coverage/low-strength quadrant is the one that produces incidents, precisely
because the coverage number reassures everybody.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM, so I will not print a
mutation score and claim it is yours. What I can give you exactly is the command, the
file to open, and how to read every possible outcome.

### Setup

```bash
cd ~/code/orderflow
java --version        # 21 or 25
mvn -q -DskipTests package
mvn versions:display-plugin-updates | grep -i pitest    # find the current PIT version
```

Add the plugin block from *Machine-level reality* to the module's `pom.xml`.

> **The one setup failure everybody hits:** without the `pitest-junit5-plugin`
> dependency **inside the plugin's `<dependencies>` block**, PIT uses its JUnit 4
> runner, finds zero tests, and reports every mutant as `NO_COVERAGE`. If your first
> run says "no tests found" or shows 100% NO_COVERAGE, that is the cause. It is not
> a subtle failure and it is not your code.

### Proof 1 — the first run

```bash
mvn -q org.pitest:pitest-maven:mutationCoverage
open target/pit-reports/index.html          # macOS; use xdg-open on Linux
```

**What to look for**, in this order:

1. The console tail: PIT prints a summary of generated mutants and the run time.
2. `index.html`: a table with one row per package — **Line Coverage**, **Mutation
   Coverage**, **Test Strength**.
3. Click into the package, then into `OrderTotalCalculator`.

| What you see | What it means |
|---|---|
| A table with a per-class breakdown | Working. Drill into your lowest **Test Strength** class first, not your lowest coverage. |
| "no tests found" / everything `NO_COVERAGE` | The `pitest-junit5-plugin` dependency is missing. See above. |
| Run takes >10 minutes | `targetClasses` or `targetTests` is too wide. Narrow it before doing anything else. |
| Many `TIMED_OUT` | Either a mutated loop (a real kill — fine) or genuinely slow tests. Raise `timeoutConstant` only after checking which. |
| `target/pit-reports/<timestamp>/` instead of a flat path | `timestampedReports` defaults to true. Set it false. |

### Proof 2 — read the per-class report properly

Open the class page. It shows your source with line numbers, and each line that
generated mutants carries a marker. Below the source is a numbered list of every
mutant with its status.

**What to look for:** entries reading like

```
14   1. changed conditional boundary → SURVIVED
```

**How to read it:**

- The number in front is which mutant on that line.
- The description names the **mutator** — `changed conditional boundary` is
  `CONDITIONALS_BOUNDARY`, `negated conditional` is `NEGATE_CONDITIONALS`,
  `Replaced long subtraction with addition` is `MATH`, `removed call to ...` is
  `VOID_METHOD_CALLS`.
- The status is the finding.

Read the report **by status, not by line**. Filter your attention to `SURVIVED`. Then
for each one, ask the mutator's question from the table in *Machine-level reality*:
"did you test the boundary?", "does anyone verify this side effect?".

### Proof 3 — prove PIT is really running your tests

Deliberately weaken one test and watch the score move.

```bash
# 1. Baseline: record the numbers for OrderTotalCalculator in your own table below.
mvn -q org.pitest:pitest-maven:mutationCoverage

# 2. Change one assertion from isEqualTo(8_500L) to isGreaterThan(0L).
# 3. Re-run.
mvn -q org.pitest:pitest-maven:mutationCoverage
```

| What you see | What it means |
|---|---|
| Test strength drops, new `SURVIVED` mutants on that line | Correct. You have just demonstrated the whole thesis: the test still passes, coverage is unchanged, and the suite got measurably worse. |
| Nothing changes | Another test was already killing those mutants. Find it — that is useful information about redundancy. |
| The build fails on `mutationThreshold` | Also correct, and the reason to set a threshold at all. |

Record it here. **These are your numbers to fill in; I have not put any in.**

| Run | Line coverage | Mutation coverage | Test strength | Survived count |
|---|---|---|---|---|
| Baseline | | | | |
| Assertion weakened | | | | |
| After you strengthen it back | | | | |

### Proof 4 — measure the cost of scoping

```bash
# wide
time mvn -q org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses='com.orderflow.*' -DtargetTests='com.orderflow.*'

# narrow
time mvn -q org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses='com.orderflow.pricing.*' -DtargetTests='com.orderflow.pricing.*Test'
```

| Run | Wall time | Mutants generated | Survived | Useful findings |
|---|---|---|---|---|
| Wide | | | | |
| Narrow | | | | |

Fill both rows in. Then answer honestly: **did the wide run surface a single finding
the narrow run missed that you would actually act on?** If not, you have your
configuration.

### Proof 5 — incremental analysis

```bash
mvn -q org.pitest:pitest-maven:mutationCoverage -DwithHistory=true   # cold
mvn -q org.pitest:pitest-maven:mutationCoverage -DwithHistory=true   # warm, nothing changed
```

| What you see | What it means |
|---|---|
| Second run is dramatically faster | History file is being read and written. Working as designed. |
| Second run is the same speed | Either you ran `mvn clean` between them (which deletes the history file under `target/`) or the paths are not configured. Set `historyInputFile`/`historyOutputFile` outside `target/`. |
| Second run is faster but reports differ | Stale cache. For a release build, run without history. |

Time both and record:

| Run | Wall time |
|---|---|
| Cold (no history) | |
| Warm (history present, no source change) | |
| Warm (one class changed) | |

---

## Failure drill

**This drill is mandatory.** The master plan assigns it explicitly:
*run PIT on the order-total calculation, find a surviving boundary mutant, and write
the test that kills it.* Do not read past this section without doing it.

### Setup

```bash
cd ~/code/orderflow
git checkout -b drill/pit-order-total
```

1. Ensure `com.orderflow.pricing.OrderTotalCalculator` exists with at least two
   boundary conditions in it — the bulk-discount threshold and the free-shipping
   threshold from Example 2 are the ones to use.
2. Ensure `OrderTotalCalculatorTest` exists in its **weak** form — the three-test
   version from Example 2, with `isGreaterThan(0L)` and `isNotNull()`. If your real
   suite is already strong, temporarily weaken it on this branch. You need to *see*
   a survivor, and a strong suite will not give you one.
3. Confirm line coverage is high before you start, so you can quote the contrast:

```bash
mvn -q test jacoco:report
open target/site/jacoco/index.html
```

**Record the line coverage of `OrderTotalCalculator` here:** ____ %

### Step 1 — run PIT, scoped to one class

```bash
mvn -q org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses='com.orderflow.pricing.OrderTotalCalculator' \
  -DtargetTests='com.orderflow.pricing.OrderTotalCalculatorTest' \
  -DtimestampedReports=false \
  -DoutputFormats=HTML,XML
```

Scoping to one class makes this run in seconds and makes the report readable.

### Step 2 — capture the artefacts

Three things to keep. **Save them; they are the drill's output.**

```bash
open target/pit-reports/index.html
```

**(a) The summary numbers.** Fill this in from the report — blank on purpose:

| Metric | Value |
|---|---|
| Line coverage | |
| Mutation coverage | |
| Test strength | |
| Mutants generated | |
| Mutants KILLED | |
| Mutants SURVIVED | |
| Mutants NO_COVERAGE | |

**(b) A screenshot or saved copy of the class page** showing the surviving mutant
markers against your source. `target/pit-reports/` is a self-contained directory —
commit it to the drill branch, or zip it. You want to be able to look at this again
after you have fixed it.

**(c) The exact text of one surviving boundary mutant.** Copy it verbatim from the
mutation list at the bottom of the class page. It will look structurally like:

```
<line number>   <n>. changed conditional boundary → SURVIVED
```

You can also pull every survivor out of the XML mechanically, which is worth knowing:

```bash
grep -o '<mutation detected="false"[^>]*>' target/pit-reports/mutations.xml | head -20
```

or, more readably:

```bash
python3 - <<'PY'
import xml.etree.ElementTree as ET
t = ET.parse('target/pit-reports/mutations.xml')
for m in t.getroot():
    if m.get('detected') == 'false':
        print(m.get('status'),
              m.findtext('mutatedClass'),
              m.findtext('mutatedMethod'),
              'line', m.findtext('lineNumber'),
              '|', m.findtext('description'))
PY
```

**What to look for:** at minimum one `SURVIVED` entry whose description mentions
`changed conditional boundary`, on the line holding
`line.quantity() >= BULK_THRESHOLD_UNITS` or
`subtotalMinor >= FREE_SHIPPING_MINOR`.

| What you see | What it means | What to do |
|---|---|---|
| A `changed conditional boundary → SURVIVED` on a threshold line | **The drill target.** Proceed to step 3. | Continue |
| Everything KILLED | Your suite already tests both boundaries. Genuinely good — but you still need to see a survivor. Weaken one boundary test (`10` → `25`) and re-run. | Weaken and re-run |
| Everything NO_COVERAGE | The `pitest-junit5-plugin` dependency is missing, or `targetTests` does not match your test class name. | Fix config |
| A `SURVIVED` you believe is equivalent | Possible, but not on a threshold comparison. Boundary mutants on a business threshold are essentially never equivalent — the behaviour differs for exactly one input, and that input is a real order. | Look again |

### Step 3 — read the mutant, then predict the production defect

Before writing any code, write down, in one sentence, **the customer-visible bug this
mutant would ship.** Do this first. It is the part of the drill that builds the
instinct.

For `line.quantity() >= 10` → `> 10`:

> "A customer ordering exactly ten units receives no bulk discount and is overcharged
> by 15% of that line."

For `subtotalMinor >= 5000` → `> 5000`:

> "An order totalling exactly £50.00 is charged £4.99 shipping despite the site
> saying free shipping over £50."

Notice both are **quiet**. No exception, no 500, no alert. The only detection channel
is a customer complaint or a finance reconciliation. That is what a surviving mutant
on a threshold means, and it is why this class is where PIT earns its runtime.

### Step 4 — write the test that kills it

One test. At the boundary. Asserting an exact value.

```java
@Test
void bulkDiscountAppliesAtExactlyTheThresholdQuantity() {
    // 10 units × 1000 minor = 10_000; 15% discount => 8_500
    var total = calculator.calculate(
            List.of(new OrderLine(PRODUCT_ID, 1_000L, 10)), Country.GB, 0L);

    assertThat(total.subtotalMinor())
        .as("exactly %d units must qualify for the bulk discount", 10)
        .isEqualTo(8_500L);
}
```

Two things to be deliberate about:

- **Exact value, not a range.** `isGreaterThan` is what let the mutant live in the
  first place.
- **The `as(...)` description.** When this test fails in two years, the failure
  message should say what business rule broke, not `expected 8500 but was 10000`.

### Step 5 — re-run and prove the kill

```bash
mvn -q org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses='com.orderflow.pricing.OrderTotalCalculator' \
  -DtargetTests='com.orderflow.pricing.OrderTotalCalculatorTest' \
  -DtimestampedReports=false
```

| Metric | Before | After |
|---|---|---|
| Line coverage | | |
| Mutation coverage | | |
| Test strength | | |
| Mutants SURVIVED | | |

**What to look for:** the specific mutant you captured in step 2(c) now reads
`KILLED`. And — the point of the whole drill — **line coverage is unchanged**.

| What you see | What it means |
|---|---|
| That mutant now KILLED, line coverage identical | The drill succeeded. One test made the suite measurably stronger with zero coverage movement. This is the sentence you take into an interview. |
| That mutant still SURVIVED | Your test does not actually exercise the boundary. Check the arithmetic: is your quantity exactly the threshold? Is your assertion on a field the mutation affects? |
| A *different* mutant now survives | Fine and normal — you have just made the next finding visible. Repeat from step 3. |
| Line coverage went **up** | Your original suite did not fully cover the method. Note it, but it is not the point here. |

### Step 6 — what the fix proves

Write this down in the drill branch's commit message or a note. Three claims, and you
now have evidence for each:

1. **Coverage did not move.** Any metric that cannot distinguish the before-suite
   from the after-suite is not measuring test quality. You have a concrete instance,
   not an opinion.
2. **A boundary mutant on a business threshold is a shippable defect.** You wrote the
   customer-visible symptom in step 3 before you knew the fix. That is the same
   reasoning you will apply to every surviving mutant from now on.
3. **The cost was one test.** Not a process, not a coverage mandate. The remediation
   for a mutation finding is almost always a single focused test, which is why this
   technique survives contact with a delivery deadline and coverage mandates do not.

Then, finally: run PIT across the whole `com.orderflow.pricing` package and count the
remaining survivors. Fix the ones on thresholds and arithmetic. Leave the ones on
logging and `toString`. Set `mutationThreshold` to just below the score you now have,
and commit it. The threshold's job is to stop regression, not to force improvement.

---

## Measurement

### Mutation score versus line coverage, stated precisely

| | Line coverage | Mutation score | Test strength |
|---|---|---|---|
| Measures | Was the line executed? | Would the suite notice a change here? | Of the code we test, how well? |
| Cost to run | Milliseconds | Minutes | Same run |
| Can be gamed by | A test with no assertions | Almost nothing cheap | Almost nothing cheap |
| False sense of safety | **High** | Low | Low |
| Useful for | Finding files nobody tested | Finding tests that verify nothing | Judging the tests you have |

Use both. Coverage is a cheap smoke alarm — it finds *files* nobody has touched.
Mutation score is the actual inspection, and you cannot afford to run it everywhere,
which is the next section.

**The reporting sentence for a design doc:**

> Line coverage is an upper bound on how good the suite could be. Mutation score is a
> measurement of how good it is. A high coverage number with a low mutation score is
> the most dangerous state a codebase can be in, because it looks like the safest.

### Where to run it — a decision rule

Score each package on one question: **if a defect here were silent, what would it
cost?**

| Module in `orderflow` | Silent-defect cost | Run PIT? | Threshold |
|---|---|---|---|
| `pricing` (totals, discounts, VAT) | Money, directly. Undetectable without reconciliation. | **Yes** | High (80+) |
| `inventory` (reservation, decrement) | Oversell, refunds, angry customers. Topic 52's contended write. | **Yes** | High |
| `wallet` (debit, balance, cap) | Money and a regulatory story | **Yes** | High |
| `orders` (state machine) | Stuck orders, double fulfilment | Yes | Medium |
| `payments` (gateway adapter) | Partly; the real risk is the contract (Topic 62) | Selectively | Medium |
| `web` (controllers, DTOs, mappers) | Loud failures — a 500, a missing field, caught by contract tests | **No** | — |
| `config`, `*Configuration` | Fails at startup | **No** | — |

That table is the deliverable of thinking about this properly, and it is what a
senior engineer produces instead of "let's turn on mutation testing".

### How to run it in CI without it being hated

1. **On the critical modules only**, per the table.
2. **`withHistory=true`** with the history file cached between builds on the same
   branch, and the cache key including the branch name.
3. **`mutationThreshold` set at your current score, minus a small margin.** Its job is
   to catch regression. Raising it is a deliberate act, done in its own PR, with the
   tests that justify it.
4. **`coverageThreshold`** as well, so a module cannot game the mutation score by
   deleting tests until only easy code is covered.
5. **Archive `target/pit-reports/`** as a build artefact. A failing threshold with no
   report is an unactionable failure and people will disable it.
6. **Never gate at 100.** Equivalent mutants exist; a threshold of 100 is a threshold
   you will eventually turn off, and turning it off is worse than never having it.

### What a mutation score is actually worth

Honest calibration, because the number is meaningless without it:

- **Below 50% on business logic** — the suite executes the code and does not verify
  it. Treat the module as untested regardless of what coverage says.
- **60–80%** — normal for a well-tested module with some equivalent mutants and some
  genuinely unimportant survivors. A reasonable place to gate.
- **Above 90%** — either excellent, or the module is simple, or someone has been
  writing tests that assert implementation details. Look at the tests before
  celebrating.
- **100%** — check for `avoidCallsTo` hiding things, or a very small class. It is
  achievable on a pure function with three branches, and essentially never on real
  business logic.

The number is a **conversation starter and a regression alarm**, not a KPI. The moment
it becomes a KPI on a dashboard someone is measured against, it stops measuring test
quality and starts measuring willingness to write tautological tests.

---

## Practice exercises

### 1 — Easy: make a mutant survive on purpose

Take `BulkDiscountPolicy` from Example 1.

1. Write the two-test suite that gives 100% line coverage and lets the boundary
   mutant survive.
2. Run PIT scoped to that one class. Record line coverage, mutation coverage and test
   strength in a table.
3. Add the two boundary tests.
4. Re-run. Record the same three numbers.
5. In two sentences: which number moved, which did not, and what that proves.

### 2 — Medium: combine it with what you already built

Uses Topics 13 (equals/hashCode), 26 (Optional), 52 (locking) and 59 (Mockito).

Run PIT over `com.orderflow.inventory`, which contains the reservation logic Topic 52
made concurrency-safe.

1. Scope `targetClasses` and `targetTests` correctly. Exclude integration tests.
   Record the wall time.
2. Find at least **three** distinct surviving mutants of **different mutator types** —
   one `CONDITIONALS_BOUNDARY`, one `MATH` or `NEGATE_CONDITIONALS`, and one
   `VOID_METHOD_CALLS` or a `*_RETURNS`.
3. For each, write down the production symptom *before* writing the test.
4. Kill all three. Re-run. Record the before/after table.
5. Then the interesting part: find one surviving mutant you decide **not** to kill.
   Write a paragraph justifying that decision, naming whether it is equivalent, or
   whether killing it would require asserting an implementation detail. This is the
   judgement half of the topic and it is what gets tested in interviews.

### 3 — Hard: production simulation

You have inherited `orderflow` with a 91% line-coverage badge and a suspicion.

**Part A — the audit.** Run PIT across `pricing`, `inventory` and `wallet`. Produce a
table with one row per class:

| Class | Line coverage | Mutation coverage | Test strength | Survivors | Worst survivor (mutator + line) |
|---|---|---|---|---|---|
| | | | | | |

Sort it by **test strength ascending**. That ordering is your work queue, and
producing it is most of the skill.

**Part B — the honest report.** Write the paragraph you would put in a team channel.
It must contain: the coverage number, the mutation number, one concrete example of a
survivor with its customer-visible symptom, and a proposal. It must not contain the
phrase "we should improve our test coverage".

**Part C — the fix and the gate.** Kill the top ten survivors by business impact. Set
`mutationThreshold` per module at your achieved score minus 3. Wire it into CI with
`withHistory` and report archiving. Record the added build time.

**Part D — argue against yourself.** Name a module in `orderflow` where you would
*not* run PIT even though it contains real logic, and say exactly what makes the cost
exceed the benefit. Then name the condition under which your answer flips.

**Part E — the forward link.** Pick one surviving mutant that a *property* would kill
more convincingly than an example — the wallet cap is the obvious candidate. Write
the property in English. You will implement it in Topic 64.

---

## Interview questions

### Q1 — "Your team has 85% line coverage. Is that good?"

**Mid-level answer:** "It's decent. Ideally we'd get it higher, maybe 90%, and cover
the critical paths."

**Senior answer:** "It tells me 15% of lines never execute in a test, which is useful.
It tells me nothing about the other 85%, because coverage measures execution, not
assertion — a test with no assertions at all produces 100% coverage of everything it
touches. The number I'd actually want is mutation score, or better, test strength.
PIT flips a conditional or changes a return in the bytecode and re-runs the covering
tests; a surviving mutant is a line the suite runs but doesn't verify. In practice I
run it on pricing, inventory and wallet — where a silent bug costs money — and not on
controllers and mappers, where bugs are loud and contract tests catch them. And I'd
be more worried about a module at 95% coverage with 40% test strength than one at 60%
coverage with 90% strength, because the first one *looks* safe."

**What separates them:** the mid answer optimises the metric. The senior answer
rejects the metric, names the replacement, scopes where the replacement is worth its
cost, and identifies the dangerous quadrant.

**Follow-up:** "How long does PIT take on your codebase?" They are checking whether
you have actually run it. Real answers involve `targetClasses`, `withHistory`, and a
number of minutes.

---

### Q2 — "What is a surviving mutant, and is it always a bug?"

**Mid-level answer:** "It means a test didn't catch the change, so we need another
test."

**Senior answer:** "It means that line executed under test and no assertion
constrained its behaviour. It is not necessarily a bug in the production code — the
code may be perfectly correct — it's a gap in what the suite guarantees. And it isn't
always actionable either: some mutants are **equivalent**, meaning the mutated
program has identical observable behaviour, so no test can kill them. Deciding
equivalence is undecidable in general, which is exactly why gating at 100% is a
mistake — you end up writing tests that assert implementation details and then you
can't refactor. My rule is to triage survivors by the customer-visible symptom they'd
ship. A boundary mutant on a discount threshold means a customer ordering exactly ten
units is overcharged, and that's essentially never equivalent. A survivor on a
`toString` is noise, and I'd have excluded logging with `avoidCallsTo` before I even
saw it."

**What separates them:** knowing the equivalent-mutant problem is a theoretical limit,
having a triage rule based on business impact, and configuring the noise away in
advance.

**Follow-up:** "Give me an example of an equivalent mutant." A cache eviction
threshold off by one where nothing observable depends on the exact size is the
cleanest.

---

### Q3 — "How does PIT avoid taking forever?"

**Mid-level answer:** "It runs in parallel and you can limit which classes it
mutates."

**Senior answer:** "Three things. First, it mutates **bytecode** with ASM, so
producing a mutant is a byte-array rewrite rather than a compile — that's the
structural advantage the JVM has over Stryker on JS. Second, and most importantly,
**coverage-driven test selection**: it runs the suite once under a coverage agent to
build a line-to-tests map, then for each mutant it runs only the tests touching that
line, fastest first, stopping at the first failure. So most mutants die on one test
rather than the whole suite. Third, forked minion JVMs with a timeout, so a mutated
loop that hangs gets killed and counted rather than blocking the run. In practice the
thing that actually determines runtime is my configuration: if `targetTests` pulls in
`@SpringBootTest` or Testcontainers classes, every mutant pays a context startup and
the run goes from five minutes to forty. Narrow `targetClasses`, exclude integration
tests, `withHistory` for incremental runs locally."

**What separates them:** naming coverage-driven selection specifically, and knowing
that the practical bottleneck is slow covering tests rather than mutant count.

**Follow-up:** "Why not run PIT on the whole codebase every build?" Cost against
information: mutating DTOs and controllers produces survivors nobody will ever act
on, and a run people don't read is worse than no run.

---

### Q4 — "Walk me through a real mutation-testing finding you acted on."

**Mid-level answer:** "We ran it and found some places we needed more tests, so we
added them and the score went up."

**Senior answer:** "On our order-total calculator, PIT reported a surviving
`CONDITIONALS_BOUNDARY` mutant on the free-shipping check —
`subtotal >= 5000` mutated to `> 5000` and every test still passed. Line coverage on
that method was 100%. The production symptom would have been: an order of exactly
£50.00 gets charged shipping, on a site that advertises free shipping over £50 —
completely silent, detectable only by a customer complaint. Our tests all used £60 and
£20, so they exercised both branches without ever touching the boundary. One test at
exactly 5000 asserting shipping is zero killed it. Coverage did not change by a single
line, which is the part I quote when someone brings a coverage number to a review.
The broader change was that we stopped writing `isGreaterThan(0)` assertions — that
one habit was responsible for most of our survivors."

**What separates them:** a specific mutator, a specific line, the customer-visible
symptom, the observation that coverage did not move, and a habit change rather than a
one-off fix.

**Follow-up:** "What did you do about the mutants you didn't kill?" Looking for
`avoidCallsTo` for logging, a written record of the equivalence judgements, and a
threshold set below the achieved score rather than at 100.

---

### Q5 — "Where would you not use mutation testing?"

**Mid-level answer:** "Anywhere it's too slow, or on generated code."

**Senior answer:** "The rule I apply is: mutate where a defect would be **silent**.
Controllers, mappers and DTOs fail loudly — a wrong field is a 500 or a contract test
failure, and I'd rather spend the build minutes on Topic 62's contract suite there.
Configuration classes fail at startup. Generated code is somebody else's problem.
Integration-heavy code is a hard no for a different reason: if the only tests covering
a class are Testcontainers tests, PIT pays a Postgres round trip per mutant and the
run becomes unusable — that's a signal the logic should be extracted into a pure class
I can test fast, which is a design improvement independent of PIT. The places it earns
its keep in our system are pricing, inventory and wallet: money, quantities and
thresholds, where a wrong answer looks exactly like a right answer until someone
reconciles the books."

**What separates them:** the silent-versus-loud framing, the observation that
"PIT is slow here" is a design signal about testability rather than a tooling
complaint, and naming which layer covers the excluded code instead.

**Follow-up:** "So what covers the controllers?" Contract tests (Topic 62) plus slice
tests (Topic 60) — and being able to say which layer owns which risk is the whole
answer.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. PIT mutates bytecode, not source. Name two consequences of that choice — one that
   makes PIT better, and one that makes a mutant report harder to read than it would
   be if it mutated source.

2. A mutant is `TIMED_OUT` and PIT counts it as killed. Argue that this is correct.
   Then construct a case where counting it as killed is misleading.

3. Coverage-driven test selection means PIT only runs tests that touch the mutated
   line. What does that imply about a class whose only coverage comes from a
   `@SpringBootTest`? Give both the performance consequence and the design
   consequence.

4. `avoidCallsTo` for `org.slf4j` removes logging mutants. But an audit log that a
   regulator reads is also written through slf4j. How do you distinguish "logging" from
   "an observable side effect that must be verified", and what does that tell you about
   how audit trails should be written?

5. Your mutation score goes from 72% to 79% in one PR. Give three different things
   that could have caused that, only one of which is "the tests got better".

6. Equivalent mutants cannot be killed and cannot be automatically identified. Given
   that, is a mutation score a meaningful number to compare between two different
   codebases? Between two commits of the same codebase? Justify the difference.

7. You could gate CI on mutation score, or you could put the PIT report in the PR as
   information and gate on nothing. Make the strongest case for each. Which would you
   pick for a team of four, and does your answer change at forty?

---

## Quick reference card

### Commands

```bash
# Run it (config from the pom)
mvn org.pitest:pitest-maven:mutationCoverage

# Run it, scoped from the command line
mvn org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses='com.orderflow.pricing.*' \
  -DtargetTests='com.orderflow.pricing.*Test' \
  -DtimestampedReports=false

# The report
open target/pit-reports/index.html

# Every survivor, mechanically
grep -o '<mutation detected="false"[^>]*>' target/pit-reports/mutations.xml
```

### Statuses

| Status | Killed? | Meaning |
|---|---|---|
| `KILLED` | yes | A test failed. Good. |
| `SURVIVED` | **no** | **The finding.** Executed, unverified. |
| `NO_COVERAGE` | no | No test touches the line at all. |
| `TIMED_OUT` | yes | Mutant hung; usually a mutated loop. |
| `NON_VIABLE` | n/a | Bytecode failed verification. Discarded. |
| `MEMORY_ERROR` / `RUN_ERROR` | n/a | Harness problem. Investigate if frequent. |

### Metrics

```
mutation coverage = KILLED / all mutants
test strength     = KILLED / (KILLED + SURVIVED)      <- the diagnostic one
```

### Default mutators worth knowing by name

| Mutator | Change | Question |
|---|---|---|
| `CONDITIONALS_BOUNDARY` | `>=` ↔ `>` | Did you test the boundary? |
| `NEGATE_CONDITIONALS` | `==` ↔ `!=` | Does the branch matter? |
| `MATH` | `+` ↔ `-`, `*` ↔ `/` | Is the arithmetic asserted? |
| `VOID_METHOD_CALLS` | Delete a void call | Does anyone verify the side effect? |
| `EMPTY_RETURNS` / `NULL_RETURNS` / `TRUE_RETURNS` / `FALSE_RETURNS` / `PRIMITIVE_RETURNS` | Replace the return | Is the return value inspected? |
| `INCREMENTS` | `i++` → `i--` | Does anything observe the counter? |

### Key configuration

```xml
<targetClasses>        <!-- mutate only where a silent bug costs money -->
<targetTests>          <!-- fast unit tests only -->
<excludedTestClasses>  <!-- keep Testcontainers/integration tests out -->
<avoidCallsTo>         <!-- org.slf4j -->
<threads>              <!-- cores - 1 -->
<timeoutConstant>      <!-- raise only after checking why -->
<timestampedReports>   <!-- false, for a stable CI path -->
<withHistory>          <!-- incremental; cache the file outside target/ -->
<mutationThreshold>    <!-- set at achieved score, never 100 -->
<coverageThreshold>
```

Plus, mandatory for JUnit 5:

```xml
<dependencies>
  <dependency>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-junit5-plugin</artifactId>
  </dependency>
</dependencies>
```

### Gotchas checklist

- [ ] `pitest-junit5-plugin` inside the plugin's `<dependencies>`, or nothing is found.
- [ ] `targetClasses` scoped to business logic. Never `com.orderflow.*`.
- [ ] Integration and contract tests excluded from `targetTests`.
- [ ] `avoidCallsTo` logging.
- [ ] `timestampedReports=false` for a stable CI artefact path.
- [ ] History file lives outside `target/`, or `mvn clean` deletes it.
- [ ] Threshold set at achieved score. **Never 100.**
- [ ] Report archived on failure, or the gate gets disabled.
- [ ] Triage survivors by customer-visible symptom, not by count.
- [ ] Read **test strength**, not just mutation coverage.

---

## When would I use this at work?

**1. Inheriting a codebase with a coverage badge and no trust.**
You join a team, the README says 88%, and nobody can tell you whether the pricing
logic is safe to change. One scoped PIT run over the money-handling packages gives you
a defensible answer in an afternoon, plus a prioritised work queue sorted by test
strength. This is one of the fastest ways to establish credibility in a new codebase,
because you arrive with evidence rather than opinions.

**2. Before a risky refactor.**
You are about to restructure the discount engine. The question that matters is not
"are there tests?" but "will the tests notice if I break it?" PIT answers exactly
that, on exactly the classes you are about to touch, in minutes. Run it, kill the
survivors on the code you are changing, *then* refactor. You now have a safety net you
have actually tested.

**3. As a code-review instrument for test quality.**
"Please add assertions" is an unpersuasive review comment. "This PR adds 40 lines of
covered code and the survivor count went up by 12" is not arguable. Used this way — on
the diff, on high-value modules — mutation testing changes how the team writes tests
without anybody having to win a philosophical argument about it.

---

## Connected topics

**Prerequisites:**
- **58 — JUnit 5**: PIT needs the JUnit 5 plugin to discover your tests at all.
- **59 — Mockito**: `VOID_METHOD_CALLS` survivors are exactly where a `verify` is the
  right tool — one of the few places Topic 59's caution about interaction testing is
  overridden.
- **60 — Spring test slices**: if a class's only coverage is a `@SpringBootTest`, PIT
  becomes unusable on it. That is a design signal, not a tooling limit.
- **61 — Testcontainers**: exclude these from `targetTests`. Always.
- **62 — Contract testing**: the layer that covers the code you deliberately do *not*
  mutate. Contracts verify the wire; PIT verifies the logic behind it.
- **01 — Primitives and money**: half of PIT's `MATH` findings in `orderflow` are on
  `long` minor-unit arithmetic, which exists because of Topic 01's Trap 5.
- **76 — Reading bytecode** (forward, but relevant): the `if_icmplt` → `if_icmple`
  swap in *Machine-level reality* is exactly what `javap -c` shows you.

**This unlocks:**
- **64 — Property-based testing**: PIT tells you *which* invariants your examples fail
  to pin down; jqwik is how you state them once instead of enumerating cases. The
  wallet-cap survivor from Example 2 is the natural first property.
- **65 — GATE, load testing**: correctness first, then performance. A load test on
  logic you have not verified measures how fast you produce wrong answers.
- **52 — Locking** (backwards, exercised here): the inventory reservation logic is one
  of the three modules worth mutating, because oversell is a silent defect.
- **77 — JMH**: the same discipline applied to performance claims — a benchmark that
  cannot fail is as useless as a test that cannot fail.
- **118 — Metrics**: the production counterpart. PIT asks "would the suite notice?";
  a business metric asks "would production notice?" A silent pricing defect needs
  both.
- **125–135 — Principal track**: "we gate on mutation score in the modules where a
  silent defect costs money, and on nothing else" is a *policy* — the kind of artefact
  Phase 12 is about, and it fits in one sentence.

---

*Java baseline 21; runtime JDK 25. PIT operates on compiled bytecode, so the language
level of your source is nearly irrelevant to it — what matters is that PIT's ASM
version understands your class-file version. If you upgrade the JDK and PIT reports
`NON_VIABLE` mutants or fails to parse classes, that is the cause, and the fix is a
PIT upgrade. Check the current version with
`mvn versions:display-plugin-updates | grep -i pitest` rather than trusting any
number written here.*
