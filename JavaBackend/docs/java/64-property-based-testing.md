# 64 — Property-Based Testing (jqwik)

## Phase: 6 — Testing
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow`'s money invariants become executable properties — the wallet balance can never go negative, an order total always equals the sum of its lines, and a refund applied twice equals a refund applied once. Topic 63 told you which examples your suite failed to pin down; this is how you stop enumerating them.

---

## Mechanical statement

> **The framework generates inputs from a declared domain, and on failure SHRINKS
> toward the smallest input that still fails. Shrinking is what turns a 400-element
> counterexample into a 2-element one you can actually read.**

Two halves, and the second one is the one people underrate.

Random test generation is old and, on its own, nearly useless: a failure on a
randomly generated list of 400 orders with 3,000 lines tells you *that* something is
wrong and gives you no chance whatsoever of working out *what*. Shrinking is the
part that makes it a debugging tool instead of a lottery. It takes the failing input
and mechanically searches for a smaller one that still fails, repeatedly, until it
cannot. What lands in your terminal is usually two or three elements and a single
number — small enough that you can see the bug by reading it.

---

## The bridge from what you know

### The direct analogue

**`fast-check`** is the JavaScript equivalent and it is genuinely excellent. If you
have used it, the model transfers wholesale:

| fast-check | jqwik |
|---|---|
| `fc.assert(fc.property(fc.integer(), n => ...))` | `@Property void p(@ForAll int n) { ... }` |
| `fc.integer({min: 1, max: 100})` | `Arbitraries.integers().between(1, 100)` |
| `fc.array(fc.integer(), {minLength: 1})` | `.list().ofMinSize(1)` |
| `fc.record({...})` | `Combinators.combine(a, b).as(Ctor::new)` |
| `numRuns: 1000` | `@Property(tries = 1000)` |
| Shrinking, built in | Shrinking, built in |
| Seed printed on failure for replay | Seed printed **and persisted** in `.jqwik-database` |

The concepts are identical because both descend from Haskell's QuickCheck. If you
know fast-check, you know property-based testing; what you need from this document is
the jqwik API, jqwik's shrinking behaviour, and the judgement about *which*
`orderflow` invariants are worth stating as properties.

Assume you have not used fast-check. Most Node engineers have not, for a boring
reason: it requires you to know an invariant, and most application code is written
without anyone ever writing the invariant down.

### The bridge that matters — from Topic 63

You have just spent Topic 63 watching mutants survive because your tests only checked
the examples you thought of. The obvious response is "write more examples". That
works, and it does not scale, because the input space is combinatorial: three
discount types × four countries × two wallet states × any number of lines is not a
list you can enumerate by hand.

> **You test the edge cases you thought of. A property states the invariant and lets
> the generator hunt for the edge cases you did not.**

Property-based testing is not a replacement for example tests. It is the tool for
the specific situation where the input space is too large to enumerate and there is
a rule that must hold across all of it. `orderflow`'s pricing engine is exactly that
situation.

### What does not transfer

- **Java's type system does most of the domain declaration for you.** `@ForAll int
  units` already constrains the domain to 32-bit integers; fast-check needs you to
  say `fc.integer()`. jqwik generates from the parameter type by default and you only
  reach for `Arbitraries` when the type is too loose.
- **Java has no structural typing** (Topic 02), so generating a "shape" means
  generating a named type through its constructor. `Combinators.combine(...).as(...)`
  is the workhorse, and it is more verbose than `fc.record({...})`.
- **jqwik persists failures to disk.** `.jqwik-database` in the project root records
  failing samples so the next run tries them first. fast-check reports a seed and
  leaves the persistence to you. This is a genuine ergonomic advantage and it also
  causes a specific confusion, covered in Trap 4.

---

## What is this?

An **example test** says: *for this input, expect this output.*

```java
assertThat(calculator.total(List.of(line(1_000L, 3)))).isEqualTo(3_000L);
```

A **property** says: *for every input in this domain, this relationship holds.*

```java
@Property
void totalIsAlwaysTheSumOfLines(@ForAll("orderLines") List<OrderLine> lines) {
    long expected = lines.stream().mapToLong(l -> l.unitPriceMinor() * l.quantity()).sum();
    assertThat(calculator.subtotal(lines)).isEqualTo(expected);
}
```

jqwik then runs that method — by default around 1,000 times — with generated lists,
including deliberately nasty ones: empty, single-element, huge quantities, zero
prices, `Long.MAX_VALUE`.

### The four shapes of property that actually get written

Nearly every useful property in commercial code is one of these four. Knowing the
list is most of the skill, because the hard part of this technique is never the API —
it is *thinking of the property*.

**1. Invariant** — something that is always true of the output.

> A wallet balance is never negative, after any sequence of valid operations.

**2. Round-trip** — encode then decode returns the original.

> `parse(format(order)) == order`. Serialization (Topic 19) is full of these, and
> they are the cheapest properties in existence.

**3. Oracle / model-based** — a slow, obviously-correct implementation agrees with the
fast one.

> The optimised bulk-discount calculation agrees with a naive per-unit loop.

**4. Metamorphic** — a relationship between two runs, when you cannot state the
absolute answer.

> Adding a line to an order never decreases the total. Applying a refund twice equals
> applying it once (idempotence, Topic 116). Sorting a list twice equals sorting it
> once.

Metamorphic properties are the ones people fail to think of, and they are the most
valuable in business logic, because business logic rarely has a closed-form correct
answer you can independently compute — but it is full of relationships that must
hold.

---

## Why does it matter?

**1. It finds the input you would never have written.**
Quantity zero. An empty order. A line with a price of `Long.MAX_VALUE` that overflows
the subtotal. A discount stack where two 60% discounts compose to a negative total.
Every one of these is obvious in hindsight and none of them is in your test file.

**2. It documents the rule instead of the case.**
`totalNeverExceedsSumOfLines` is a sentence a product manager can read and confirm.
`calculatesTotalFor3ItemsAt2999WithVat` is a sentence about an arithmetic fact. Six
months later, only the first one tells you what the system is supposed to do.

**3. It is the natural follow-on from mutation testing.**
Topic 63's surviving mutants tell you which decisions your suite fails to constrain.
Some of those want one more example test at a boundary. Others — the wallet cap, the
discount composition — want a property, because the failure is not at one boundary but
across a region.

**4. It is a differentiator, and cheaply so.**
Very few Java engineers have written a property. Being able to say "we express the
wallet invariants as jqwik properties with a model-based chain over the operation
sequence, and the shrunk counterexamples get committed as regression examples" is a
distinctive answer in an interview, and it is about four days of learning.

---

## Machine-level reality

### Generation

An **`Arbitrary<T>`** is jqwik's generator: an object that can produce values of `T`
from a source of randomness, *and* knows how to shrink them.

When jqwik runs a `@Property`, it does this:

1. Pick a **random seed** (or reuse a recorded one — see below).
2. For each of `tries` iterations, ask each parameter's `Arbitrary` for a value,
   using a **size parameter** that grows as the run progresses. Early tries get small
   values and short lists; later ones get bigger. This is deliberate: small inputs
   find bugs faster and shrink better.
3. **Inject edge cases deliberately.** jqwik does not generate purely at random. It
   maintains an edge-case set per arbitrary — for integers: `0`, `1`, `-1`,
   `MIN_VALUE`, `MAX_VALUE`; for lists: the empty list and a single-element list; for
   strings: `""`. These are mixed into the run rather than left to chance. This is
   why a property with 100 tries reliably tests zero and empty, which pure random
   generation would essentially never produce.
4. If the domain is small enough, jqwik may switch to **exhaustive generation** and
   enumerate every value instead of sampling. `@ForAll Boolean` or `@ForAll
   OrderStatus` over a five-constant enum is exhaustible. When this happens the report
   says so, and it is a stronger guarantee than any number of random tries.

The consequence: **`tries` is not simply "number of random samples"**. A meaningful
fraction of the run is spent on values chosen because they break things.

### Shrinking — the part that matters

When a try fails, jqwik does **not** report that input. It starts searching for a
smaller one that fails the same way.

Shrinking is a property of the `Arbitrary`, not a global algorithm. Each generator
knows its own notion of "smaller":

| Type | Shrinks toward |
|---|---|
| Integral numbers | Zero, or the nearest boundary of the declared range |
| Strings | Shorter, then toward earlier characters in the alphabet |
| Lists | **Fewer elements first**, then shrink each remaining element |
| Combined values (`Combinators`) | Each component in turn, holding others fixed |
| `map`ped values | Shrinks the *source*, then re-applies the mapping |
| `filter`ed values | Shrinks the source, discarding candidates the filter rejects |

The list rule is the one that produces the dramatic results. A failure on a
400-element list is first attacked by removing elements — halving, then removing
individually — until removing any more makes it pass. Then each survivor's fields are
shrunk. A 400-element counterexample routinely becomes two elements with values like
`0` and `1`.

The report distinguishes the two:

```
Original Sample
---------------
  lines: [OrderLine[...], OrderLine[...], ... 398 more ...]

Shrunk Sample (37 steps)
------------------------
  lines: [OrderLine[unitPriceMinor=0, quantity=1]]
```

**Read the shrunk sample. It is the bug.** In this case: a zero-priced line, which
means a free promotional item, which your code did not handle.

Shrinking modes, when you need them:

```java
@Property(shrinking = ShrinkingMode.FULL)     // keep going; can take a while
@Property(shrinking = ShrinkingMode.BOUNDED)  // default; gives up after a budget
@Property(shrinking = ShrinkingMode.OFF)      // debugging only
```

`BOUNDED` is the default and occasionally gives up mid-search, reporting a partly
shrunk sample plus a note that it stopped. When that happens and the counterexample is
still unreadable, switch that one property to `FULL` and re-run.

**The critical implication for custom generators:** if you build an arbitrary in a way
that destroys shrinkability, you get random testing without the debugging tool. This
is Trap 5, and it is the most common way people ruin jqwik for themselves.

### Reproducing a failure

Two mechanisms, and you should know both.

**1. The seed.** Every failure report includes the seed used:

```
tries = 1000
checks = 1000
seed = -4183872925632481001
```

Pin it to re-run the identical sequence:

```java
@Property(seed = "-4183872925632481001")
```

**2. `.jqwik-database`.** jqwik writes a file (by default `.jqwik-database` in the
project root) recording which properties failed and the sample that broke them. On the
next run it tries the recorded failing sample **first**, so a fix is confirmed
immediately rather than after 1,000 tries.

This is controlled by `afterFailure`:

```java
@Property(afterFailure = AfterFailureMode.PREVIOUS_SEED)  // default: replay the seed
@Property(afterFailure = AfterFailureMode.SAMPLE_ONLY)    // only the failing sample
@Property(afterFailure = AfterFailureMode.RANDOM_SEED)    // ignore history
```

**Add `.jqwik-database` to `.gitignore`.** It is a local cache. Committing it makes
CI replay another machine's failure history, which is confusing rather than helpful.

### The `tries` budget

```java
@Property(tries = 1000)
```

Every try runs the whole method body. That is the entire cost model, and it is why
`tries` and test speed are directly coupled:

| Property body | 1000 tries costs |
|---|---|
| Pure calculation, no I/O | Milliseconds. Raise `tries` freely. |
| Builds a Spring context per try | Do not do this. |
| Hits a Testcontainers Postgres | 1000 round trips. Lower `tries` hard, or restructure. |

Default is 1000. Set a project-wide default in
`src/test/resources/junit-platform.properties`:

```properties
jqwik.tries.default = 1000
jqwik.database = .jqwik-database
jqwik.reporting.onlyFailures = true
```

> Property names for jqwik's configuration have moved between major versions (older
> releases read a separate `jqwik.properties`). Check what your version reads —
> jqwik prints its effective configuration on startup when reporting is verbose, and
> the user guide for your exact version is the authority. Do not copy a properties
> file from a blog post without checking.

### Maven wiring

```xml
<dependency>
  <groupId>net.jqwik</groupId>
  <artifactId>jqwik</artifactId>
  <scope>test</scope>
  <!-- version: check the current release with
       mvn versions:display-dependency-updates | grep -i jqwik -->
</dependency>
```

jqwik is a **JUnit Platform test engine**, not a JUnit 5 extension. It runs alongside
`junit-jupiter` in the same module and the same Surefire execution. `@Property`
methods are discovered by the jqwik engine; `@Test` methods by Jupiter. You can put
both in the same class, and you frequently should — a property for the rule, a couple
of examples for readability.

---

## Example 1 — minimal

The smallest property worth writing, plus the smallest realistic bug it catches.

```java
package com.orderflow.pricing;

public final class LineTotals {
    public static long lineMinor(long unitPriceMinor, int quantity) {
        return unitPriceMinor * quantity;
    }
}
```

The example test:

```java
@Test
void multipliesPriceByQuantity() {
    assertThat(LineTotals.lineMinor(2_999L, 3)).isEqualTo(8_997L);
}
```

Passes. Tells you almost nothing.

The property:

```java
import net.jqwik.api.*;
import net.jqwik.api.constraints.*;
import static org.assertj.core.api.Assertions.*;

class LineTotalsProperties {

    @Property
    void lineTotalIsNeverNegativeForValidInputs(
            @ForAll @LongRange(min = 0, max = 10_000_000L) long unitPriceMinor,
            @ForAll @IntRange(min = 0, max = 10_000) int quantity) {

        assertThat(LineTotals.lineMinor(unitPriceMinor, quantity)).isGreaterThanOrEqualTo(0L);
    }

    @Property
    void lineTotalGrowsMonotonicallyWithQuantity(
            @ForAll @LongRange(min = 1, max = 1_000_000L) long unitPriceMinor,
            @ForAll @IntRange(min = 0, max = 9_999) int quantity) {

        assertThat(LineTotals.lineMinor(unitPriceMinor, quantity + 1))
            .isGreaterThan(LineTotals.lineMinor(unitPriceMinor, quantity));
    }
}
```

Line by line, the bits that are new:

| Syntax | Meaning |
|---|---|
| `@Property` | This is a property. jqwik runs it `tries` times (default 1000). |
| `@ForAll` | This parameter is generated. Every parameter of a `@Property` must have it. |
| `@LongRange(min=, max=)` | An annotation-level constraint on the built-in generator. Shorter than an `Arbitrary` when the constraint is just a range. |
| `@IntRange` | Same for `int`. Also `@Size`, `@AlphaChars`, `@NotBlank`, `@Positive`. |

The second property is **metamorphic**: it does not say what the total should be, only
that adding a unit must increase it. That is a rule you can state confidently without
recomputing the arithmetic — which is exactly the situation you are in with real
pricing code.

Now remove the range constraints and re-run. Without `@LongRange`, jqwik will
eventually generate values near `Long.MAX_VALUE`, `unitPriceMinor * quantity`
overflows, the product goes negative, and the first property fails with a shrunk
counterexample. That is not a contrived failure — it is precisely the
"someone typed an extra zero into the price field" defect, and no example test you
would have written finds it.

**Which teaches the real lesson about ranges:** a range constraint is a *claim about
your domain*. `@LongRange(max = 10_000_000L)` says "a unit price above £100,000 cannot
occur". If that claim is not enforced somewhere — a validation annotation (Topic 45),
a database check constraint — then your property is testing a domain your production
system does not have. Which is Trap 1.

---

## Example 2 — production scenario on the project spine

Three properties on `orderflow`, chosen because each is a rule the business would
recognise and none is expressible as a finite list of examples.

### Generators for the domain

First, build arbitraries that produce *realistic* `orderflow` values. This is where
most of the work is, and where most of the mistakes are.

```java
package com.orderflow.pricing;

import net.jqwik.api.*;
import java.util.List;

class OrderflowArbitraries {

    /** Product ids drawn from a small pool, so lines can collide on the same product. */
    @Provide
    static Arbitrary<String> productIds() {
        return Arbitraries.of(
            "b81c4d22-0000-4111-8222-333344445555",
            "c92d5e33-1111-4222-9333-444455556666",
            "d03e6f44-2222-4333-a444-555566667777");
    }

    /**
     * Prices spanning the whole realistic range INCLUDING zero (promotional items)
     * and including values around the free-shipping threshold of 5000.
     */
    @Provide
    static Arbitrary<Long> unitPricesMinor() {
        return Arbitraries.oneOf(
            Arbitraries.just(0L),                              // free promo item
            Arbitraries.longs().between(1L, 200L),             // pennies
            Arbitraries.longs().between(4_900L, 5_100L),       // AROUND the threshold
            Arbitraries.longs().between(201L, 500_000L));      // the bulk of real prices
    }

    /** Quantities spanning the bulk-discount threshold of 10 from both sides. */
    @Provide
    static Arbitrary<Integer> quantities() {
        return Arbitraries.oneOf(
            Arbitraries.integers().between(1, 3),
            Arbitraries.integers().between(8, 12),             // AROUND the threshold
            Arbitraries.integers().between(4, 200));
    }

    @Provide
    static Arbitrary<OrderLine> orderLines() {
        return Combinators.combine(productIds(), unitPricesMinor(), quantities())
                          .as(OrderLine::new);
    }

    @Provide
    static Arbitrary<List<OrderLine>> orderLineLists() {
        return orderLines().list().ofMinSize(1).ofMaxSize(25);
    }

    @Provide
    static Arbitrary<Long> walletBalancesMinor() {
        return Arbitraries.oneOf(
            Arbitraries.just(0L),
            Arbitraries.just(20_000L),                          // exactly the cap
            Arbitraries.longs().between(1L, 19_999L),
            Arbitraries.longs().between(20_001L, 5_000_000L));
    }
}
```

Read what those generators encode. They are not "random longs". Each one deliberately
concentrates probability mass **around the thresholds the code branches on** — 10
units, 5,000 minor units, the 20,000 wallet cap, and zero. That targeting is the
craft of this technique, and it is a direct consequence of Topic 63: the surviving
mutants told you exactly which boundaries matter, so the generators aim there.

`Arbitraries.oneOf(...)` picks one of the supplied arbitraries per generation, so the
mixture is roughly uniform across the four groups — meaning about a quarter of all
generated quantities land in 8..12, which is where the bulk-discount decision lives.
Pure `Arbitraries.integers().between(1, 200)` would land there about 2.5% of the time.

### Property 1 — the subtotal is the sum of the lines (invariant)

```java
class OrderTotalProperties {

    private final OrderTotalCalculator calculator =
            new OrderTotalCalculator(new FixedVatRates(20), new NoOpAuditLog());

    @Property(tries = 2000)
    void subtotalEqualsTheSumOfDiscountedLines(
            @ForAll("orderLineLists") List<OrderLine> lines) {

        long expected = lines.stream()
            .mapToLong(l -> {
                long raw = l.unitPriceMinor() * l.quantity();
                return l.quantity() >= 10 ? raw - (raw * 15) / 100 : raw;
            })
            .sum();

        assertThat(calculator.calculate(lines, Country.GB, 0L).subtotalMinor())
            .isEqualTo(expected);
    }
}
```

> **Careful.** This is an **oracle** property, and it is one line away from being a
> tautology — see Trap 2. It is legitimate here only because the oracle is a
> deliberately naive per-line loop while the production implementation may batch,
> reorder or short-circuit. If someone later "optimises" `OrderTotalCalculator` by
> collapsing lines that share a product id, this property catches the behaviour change
> immediately. If the two implementations are line-for-line identical, delete the
> property and write property 2 instead.

### Property 2 — adding a line never reduces the total (metamorphic)

This one is not a tautology, cannot be written as an example, and is a rule the
business would sign off on in one sentence.

```java
    @Property(tries = 2000)
    void addingALineNeverReducesTheTotal(
            @ForAll("orderLineLists") List<OrderLine> lines,
            @ForAll("orderLines")     OrderLine extra) {

        long before = calculator.calculate(lines, Country.GB, 0L).grandTotalMinor();

        List<OrderLine> more = new ArrayList<>(lines);
        more.add(extra);
        long after = calculator.calculate(more, Country.GB, 0L).grandTotalMinor();

        assertThat(after)
            .as("adding %s must not reduce the grand total", extra)
            .isGreaterThanOrEqualTo(before);
    }
```

**This property has a real chance of failing on the code in Topic 63, and it should.**
The free-shipping rule means adding a cheap line can push the subtotal over 5,000 and
remove a 499 shipping charge. If the added line is worth less than 499 minor units,
the grand total *decreases*.

That is arguably correct business behaviour. It is also arguably a bug that lets a
customer reduce their bill by adding an item. Either way, **the property forced
somebody to make the decision explicitly**, which no example test was ever going to
do. The resolution is one of:

- Accept it, and narrow the property to `subtotalMinor` rather than
  `grandTotalMinor`, with a comment explaining why.
- Change the rule so shipping is never refunded below the threshold crossing.
- Keep it as a failing property tagged `@Disabled` with a link to the product ticket.

The first is usually right. What matters is that you *know*.

### Property 3 — the wallet invariant, over any sequence of operations

The master plan's mastery line for this topic is: *wallet balance never negative;
order total equals the sum of lines after any sequence of valid mutations.* The
"sequence" half needs jqwik's stateful testing.

```java
    @Property(tries = 500)
    void walletBalanceIsNeverNegativeAfterAnySequenceOfOperations(
            @ForAll("walletOperationChains") Chain<Wallet> chain) {

        chain.start(() -> new Wallet(WALLET_ID, 10_000L))
             .forEach(wallet ->
                 assertThat(wallet.balanceMinor())
                     .as("balance must never go negative")
                     .isGreaterThanOrEqualTo(0L));
    }

    @Provide
    Arbitrary<Chain<Wallet>> walletOperationChains() {
        return Chain.startWith(() -> new Wallet(WALLET_ID, 10_000L))
                    .withTransformer(debits())
                    .withTransformer(credits())
                    .withTransformer(holds())
                    .withTransformer(releases())
                    .withMaxTransformations(30);
    }

    private Arbitrary<Transformer<Wallet>> debits() {
        return Arbitraries.longs().between(1L, 30_000L).map(amount ->
            Transformer.mutate("debit " + amount, wallet -> {
                try { wallet.debit(amount); }
                catch (InsufficientFundsException expected) { /* a valid refusal */ }
            }));
    }
```

> **API version warning, stated honestly.** jqwik has had two generations of stateful
> testing API: the older `net.jqwik.api.stateful.Action` / `ActionSequence` (now
> deprecated) and the newer `net.jqwik.api.state.Chain` / `Transformer`. Which one
> your version ships determines which of these compiles. Check your version's user
> guide; the compile error will name the expected type, so this costs you a minute,
> not an afternoon.

Two things to notice about this property:

- **It catches the refusal as success.** `InsufficientFundsException` is a *correct*
  outcome, not a failure. The invariant is about the balance, not about whether every
  operation succeeds. Getting this distinction right is the main design decision in a
  stateful property.
- **This is where the shrinking really earns its place.** A failure on a 30-operation
  sequence is unreadable. Shrinking removes operations until it finds the minimum
  sequence that still breaks the invariant — typically two: a hold and a debit that
  together exceed the balance because the hold was not counted. That two-step sequence
  is a bug report you can act on immediately, and it is the same class of defect Topic
  52 dealt with under concurrency.

### What to do with a counterexample

When one of these fails, **do not just fix the code**. Do both of these:

1. Fix the code.
2. **Commit the shrunk counterexample as a plain example test.** It runs in
   microseconds, it never regresses, and it documents the specific bug.

```java
    @Test   // regression: found by walletBalanceIsNeverNegative..., 2026-08-29
    void aHoldFollowedByAFullDebitCannotOverdrawTheWallet() {
        Wallet wallet = new Wallet(WALLET_ID, 10_000L);
        wallet.hold(6_000L);
        assertThatThrownBy(() -> wallet.debit(10_000L))
            .isInstanceOf(InsufficientFundsException.class);
        assertThat(wallet.balanceMinor()).isEqualTo(10_000L);
    }
```

Properties hunt. Examples pin. You need both, and the pipeline from one to the other
is the workflow this topic is really teaching.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the generator so narrow it never produces the interesting case

**Wrong:**

```java
@Provide
Arbitrary<Integer> quantities() {
    return Arbitraries.integers().between(1, 5);
}

@Property
void bulkDiscountIsNeverNegative(@ForAll("quantities") int quantity, ...) { ... }
```

**Exact symptom:** the property passes. It has passed 1,000 times a build, for eight
months. It has never once exercised the bulk-discount branch, because the branch
requires `quantity >= 10` and the generator produces 1 to 5. The property's *name*
mentions the bulk discount. The report says `tries = 1000, checks = 1000`. Everything
looks superb.

The failure surfaces as a production defect in the branch the property claims to
cover, and the person investigating loses an hour because a property test named
`bulkDiscount...` is green.

**Root cause:** a property is only as good as its generator's *support* — the region
of the input space it actually produces with non-negligible probability. A range that
excludes the branch condition means the property is vacuously true.

The subtler version of the same bug: `Arbitraries.integers().between(1, 200)` does
include 10, but lands in 8..12 about 2.5% of the time, and the boundary value exactly
10 about 0.5% of the time. With 1,000 tries you hit it, but with a two-parameter
property where both need to be in an interesting region simultaneously, the joint
probability collapses and you may go thousands of tries without a single meaningful
combination.

**Fix — and this is the most important technique in the document:** *measure your
generator*. jqwik has `Statistics` for exactly this.

```java
@Property(tries = 1000)
@StatisticsReport(format = Histogram.class)
void bulkDiscountIsCorrect(@ForAll("quantities") int quantity) {

    Statistics.label("quantity band")
              .collect(quantity < 9  ? "below"
                     : quantity == 9 ? "just below"
                     : quantity == 10 ? "AT THRESHOLD"
                     : quantity <= 12 ? "just above" : "well above");

    Statistics.label("quantity band")
              .coverage(c -> c.check("AT THRESHOLD").percentage(p -> p >= 2.0));

    // ... the actual assertions
}
```

`Statistics.collect` records a classification per try and prints a distribution.
`Statistics.coverage` **fails the property** if a classification does not occur often
enough. That second call turns "I hope my generator is good" into a build failure when
it is not.

Then build the generator to target the branches, as in Example 2:

```java
return Arbitraries.oneOf(
    Arbitraries.integers().between(1, 3),
    Arbitraries.integers().between(8, 12),      // deliberately concentrated
    Arbitraries.integers().between(4, 200));
```

**The rule:** every property that claims to test a branch must have a coverage check
proving the generator reaches that branch. Without it, a passing property is not
evidence.

---

### Trap 2 — the property that restates the implementation

**Wrong:**

```java
@Property
void totalIsCorrect(@ForAll("orderLineLists") List<OrderLine> lines) {
    long expected = 0;
    for (OrderLine l : lines) {
        long raw = l.unitPriceMinor() * l.quantity();
        if (l.quantity() >= 10) raw -= (raw * 15) / 100;
        expected += raw;
    }
    assertThat(calculator.subtotal(lines)).isEqualTo(expected);
}
```

Somebody wrote this by copying the body of `subtotal` into the test.

**Exact symptom:** the property can never fail on a logic error, because both sides
compute the same thing. Change the threshold from 10 to 12 in production code and the
property fails — because the test *also* uses 10 — so it looks like it works. Now
change the discount from 15% to 18% in **both** places (which is what happens when a
developer does a project-wide find-and-replace, or when the constant is shared) and
the property passes while the behaviour changed. Worse, this property will pass with
`>=` mutated to `>` if the test uses the same shared constant and the same comparison
— which means it does not even kill the mutant Topic 63 found.

**Root cause:** the oracle is a copy of the implementation. This is the property-based
version of `assertThat(actual).isEqualTo(actual)`.

**Fix:** the oracle must be *independently obviously correct*, ideally at a different
level of abstraction, or you must switch to a metamorphic property.

```java
// Oracle at a different abstraction: expand every line into individual units.
@Property
void subtotalMatchesAPerUnitExpansion(@ForAll("orderLineLists") List<OrderLine> lines) {
    long expected = lines.stream()
        .flatMap(l -> IntStream.range(0, l.quantity())
                               .mapToObj(i -> new Unit(l.unitPriceMinor(), l.quantity())))
        .mapToLong(u -> u.discountedPriceMinor())
        .sum();
    assertThat(calculator.subtotal(lines)).isEqualTo(expected);
}

// Or, better, avoid an oracle entirely:
@Property
void doublingEveryQuantityAtLeastDoublesTheSubtotal(
        @ForAll("orderLineLists") List<OrderLine> lines) {
    long single = calculator.subtotal(lines);
    long doubled = calculator.subtotal(lines.stream().map(OrderLine::doubleQuantity).toList());
    assertThat(doubled).isGreaterThanOrEqualTo(single * 2);   // discounts only help
}
```

**The test for whether a property is worth having:** if someone deleted the production
implementation and reimplemented it from the property alone, would they be forced to
write correct code? If instead they would have to write *this exact code*, the
property is a mirror, not a test.

---

### Trap 3 — over-filtering, and the discard explosion

**Wrong:**

```java
@Provide
Arbitrary<Order> ordersThatQualifyForFreeShipping() {
    return orders().filter(o -> o.subtotalMinor() >= 5_000L
                             && o.lines().size() >= 3
                             && o.shipTo() == Country.GB);
}
```

**Exact symptom:** the property fails to run at all, with something like

```
net.jqwik.api.TooManyFilterMissesException:
  Trying to fulfil a filter condition has failed 10000 times in a row
```

Or, on a looser filter, the property runs but takes thirty seconds and the statistics
report shows a tiny fraction of tries surviving.

**Root cause:** `filter` is **rejection sampling**. jqwik generates a value, tests the
predicate, and throws the value away if it fails. Three conjoined conditions each
satisfied 20% of the time means 0.8% of generated values survive, so 99.2% of the
generation budget is wasted, and jqwik gives up when the miss streak gets absurd.

**Fix — construct, do not filter.** Build the value so it satisfies the constraint by
construction:

```java
@Provide
Arbitrary<Order> ordersThatQualifyForFreeShipping() {
    Arbitrary<List<OrderLine>> lines =
        orderLines().list().ofMinSize(3).ofMaxSize(10);

    return lines.map(ls -> {
        // Force the subtotal over the threshold by construction, not by rejection.
        long subtotal = ls.stream().mapToLong(OrderLine::lineMinor).sum();
        if (subtotal < 5_000L) {
            ls = new ArrayList<>(ls);
            ls.add(new OrderLine(TOP_UP_PRODUCT, 5_000L - subtotal + 1, 1));
        }
        return new Order(ls, Country.GB);
    });
}
```

**Rules of thumb:**

- `filter` is fine when it rejects a *small* fraction — dropping the odd `0` from a
  divisor, say.
- Use `Assume.that(condition)` inside the property body for the same purpose; it
  discards the try rather than failing it. Same cost model, same caution.
- `flatMap` is the right tool when one generated value determines the domain of the
  next: generate the subtotal first, then generate lines that sum to it.
- If your filter rejects more than roughly half, restructure the generator.

---

### Trap 4 — the property that "passes on my machine"

**Wrong:** a property fails in CI. The developer runs it locally, it passes, they
re-run CI, it passes, and the ticket is closed as flaky.

**Exact symptom:** intermittent CI failures on a property, never reproducible on
demand, gradually leading to the property being annotated `@Disabled`. Or the
opposite and more insidious version: the property fails locally on Monday, the
developer runs it again on Tuesday and it passes, so they assume they fixed it — but
what actually happened is that `.jqwik-database` replayed the failing sample on
Monday's second run, they changed something unrelated, and on Tuesday jqwik moved on
to a fresh random seed.

**Root cause:** every run uses a different seed, so a property that fails for 0.3% of
inputs fails roughly one run in three. It is not flaky. It is a **real bug with a
low-probability trigger**, which is precisely what property testing exists to find,
being misdiagnosed as noise.

**Fix — a four-step discipline, and it must be a team norm:**

1. **Capture the seed and the shrunk sample from the failing CI log.** They are in
   the report. If your CI truncates test output, fix that first; a property failure
   whose report you cannot read is worthless.
2. **Reproduce deterministically** by pinning the seed:
   ```java
   @Property(seed = "-4183872925632481001")
   ```
3. **Fix the bug.**
4. **Commit the shrunk counterexample as a `@Test`**, then remove the pinned seed so
   the property goes back to hunting. The example is the regression guard; the property
   is the explorer. Leaving the seed pinned permanently converts your property into a
   single very slow example test.

Add `.jqwik-database` to `.gitignore`, and know that it exists — otherwise the
"it passed the second time" confusion above will cost somebody a day.

---

### Trap 5 — a generator that cannot shrink

**Wrong:**

```java
@Provide
Arbitrary<Order> orders() {
    return Arbitraries.randomValue(random -> {
        // build an order using the raw Random directly
        int lineCount = random.nextInt(50) + 1;
        List<OrderLine> lines = new ArrayList<>();
        for (int i = 0; i < lineCount; i++) {
            lines.add(new OrderLine(randomProductId(random),
                                    random.nextLong(1_000_000L),
                                    random.nextInt(500) + 1));
        }
        return new Order(lines, Country.GB);
    });
}
```

**Exact symptom:** the property fails and the report reads something like:

```
Original Sample
---------------
  order: Order[lines=[OrderLine[...], ...47 more...], shipTo=GB]

Shrunk Sample (0 steps)
-----------------------
  order: Order[lines=[OrderLine[...], ...47 more...], shipTo=GB]
```

**Zero shrink steps.** You are handed a 48-line order with random prices and told
something in there is wrong. You now spend an hour bisecting it by hand — exactly the
work the tool was supposed to do for you.

**Root cause:** `Arbitraries.randomValue` (and any generator built by consuming a raw
`Random`) produces an opaque value. jqwik has no structural knowledge of it, so there
is nothing to shrink. The same happens if you `map` from a single random `long` seed
into a complex object: shrinking the seed produces a *completely different* object, so
shrink candidates almost never still fail, and the search dies immediately.

**Fix:** build values by **composing arbitraries**, which is what makes them
shrinkable.

```java
@Provide
Arbitrary<Order> orders() {
    Arbitrary<OrderLine> lines =
        Combinators.combine(productIds(), unitPricesMinor(), quantities())
                   .as(OrderLine::new);

    return Combinators.combine(lines.list().ofMinSize(1).ofMaxSize(50),
                               Arbitraries.of(Country.class))
                      .as(Order::new);
}
```

Now every component knows how to shrink: the list drops elements, the prices shrink
toward zero, the quantities shrink toward their range minimum, the country shrinks
toward the first enum constant. A 48-line failure collapses to one or two lines.

**The general rule:** `Combinators.combine`, `.map`, `.flatMap`, `.list()`,
`.filter()` all preserve shrinkability because jqwik shrinks the *source* and
re-applies your function. Raw `Random` consumption destroys it. If you find yourself
reaching for `Arbitraries.randomValue`, you almost always want `Combinators` instead.

---

## Hands-on proof

Everything here is a command **you** run. I have no JVM, so I will not print a
counterexample and claim it came from your code. What follows is the command, the file
to read, and how to interpret every outcome.

### Setup

```bash
cd ~/code/orderflow
java --version                                          # 21 or 25
mvn versions:display-dependency-updates | grep -i jqwik  # find the current version
```

Add the `net.jqwik:jqwik` test dependency, then:

```bash
mvn -q test -Dtest='*Properties'
```

| What you see | What it means |
|---|---|
| Properties run and report `tries`/`checks` | jqwik's engine is registered. Good. |
| "No tests were executed" | Surefire is filtering them out. `@Property` methods are not `@Test` methods; use `-Dtest='*Properties'` matching the class name, or drop the filter. |
| `ClassNotFoundException: net.jqwik...` | Dependency missing or wrong scope. |

### Proof 1 — watch a property fail and shrink

Write a property you know is false, so you can read a real report:

```java
@Property
void everySubtotalIsUnderTenPounds(@ForAll("orderLineLists") List<OrderLine> lines) {
    assertThat(calculator.subtotal(lines)).isLessThan(1_000L);   // deliberately false
}
```

```bash
mvn -q test -Dtest='OrderTotalProperties'
```

**What to look for** in the failure report — four things, in this order:

1. `tries = N` and `checks = N` — how many were generated and how many actually ran
   (they differ when `Assume`/`filter` discards).
2. `seed = ...` — copy this.
3. **`Original Sample`** — the input that first failed.
4. **`Shrunk Sample (K steps)`** — the minimal input.

| What you see | What it means |
|---|---|
| Shrunk sample much smaller than original, K > 0 | Shrinking is working. **Read the shrunk sample; that is your bug.** |
| `Shrunk Sample (0 steps)` | Either the original was already minimal, or your generator is unshrinkable (Trap 5). Check whether you used `Arbitraries.randomValue`. |
| A note that shrinking stopped early | `ShrinkingMode.BOUNDED` hit its budget. Re-run that property with `shrinking = ShrinkingMode.FULL`. |
| `checks` far below `tries` | A filter or `Assume` is discarding most values (Trap 3). |

### Proof 2 — reproduce it deterministically

```java
@Property(seed = "<paste the seed>")
```

```bash
mvn -q test -Dtest='OrderTotalProperties'
```

**What to look for:** the identical counterexample. If you get a different one, either
the seed was not applied or your property depends on something outside the generated
inputs — a clock, a static counter, a shared mutable fixture. That is a finding in its
own right.

### Proof 3 — prove your generator hits the interesting region

This is the proof that separates a real property from a decorative one.

```java
@Property(tries = 1000)
@StatisticsReport(format = Histogram.class)
void quantityGeneratorReachesTheBulkThreshold(@ForAll("quantities") int quantity) {
    Statistics.label("band").collect(
        quantity < 9 ? "below" : quantity == 9 ? "9" :
        quantity == 10 ? "10 (THRESHOLD)" : quantity <= 12 ? "11-12" : "above");
}
```

```bash
mvn -q test -Dtest='GeneratorCoverageProperties'
```

**What to look for:** the printed histogram, and specifically the percentage against
`10 (THRESHOLD)`.

Record it. **Blank on purpose — these are your numbers:**

| Band | % of tries (naive `between(1,200)`) | % of tries (targeted `oneOf`) |
|---|---|---|
| below 9 | | |
| exactly 9 | | |
| exactly 10 (threshold) | | |
| 11–12 | | |
| above 12 | | |

Run it once with a naive uniform generator and once with the targeted `oneOf`
generator from Example 2, and fill both columns. The comparison is the entire argument
for writing generators deliberately, and having your own two columns is far more
convincing than any number I could write here.

Then make it enforceable:

```java
Statistics.label("band")
          .coverage(c -> c.check("10 (THRESHOLD)").percentage(p -> p >= 1.0));
```

| What you see | What it means |
|---|---|
| Property passes | The generator provably reaches the threshold at the required rate. |
| Property fails with a coverage error | **The most useful failure in this document.** Your generator does not exercise what the property claims to test. |

### Proof 4 — the `tries` budget against wall time

```bash
time mvn -q test -Dtest='OrderTotalProperties' -Djqwik.tries.default=100
time mvn -q test -Dtest='OrderTotalProperties' -Djqwik.tries.default=1000
time mvn -q test -Dtest='OrderTotalProperties' -Djqwik.tries.default=10000
```

| tries | Wall time | Failures found |
|---|---|---|
| 100 | | |
| 1,000 | | |
| 10,000 | | |

Fill it in for a pure-calculation property, then repeat for a property that touches a
Testcontainers Postgres. The second table is the one that will change your mind about
where properties belong.

### Proof 5 — the `.jqwik-database`

```bash
ls -la .jqwik-database
mvn -q test -Dtest='OrderTotalProperties'   # after a failure
```

**What to look for:** the second run reproducing the failure immediately rather than
after hundreds of tries.

| What you see | What it means |
|---|---|
| Instant reproduction | `afterFailure` replay working as designed. |
| File missing | Database disabled, or the working directory differs. |
| CI behaves differently from local | Expected — CI has no history. Add the file to `.gitignore` and stop trying to make them match. |

---

## Measurement

### How many tries actually buy confidence

The honest answer first: **`tries` is not a confidence level and cannot be converted
into one.** There is no "1,000 tries means 99.7% correct". What `tries` buys is
*probability of hitting a failing region*, and that probability depends entirely on
how large the failing region is under your generator's distribution.

The rough model. If a bug is triggered by a fraction `p` of the inputs your generator
produces, the chance of missing it in `n` independent tries is `(1 - p)^n`:

| Failing fraction `p` | Miss probability at 100 tries | at 1,000 | at 10,000 |
|---|---|---|---|
| 10% | ~0.003% | ~0 | ~0 |
| 1% | ~37% | ~0.004% | ~0 |
| 0.1% | ~90% | ~37% | ~0.005% |
| 0.01% | ~99% | ~90% | ~37% |

Read the diagonal. Each order of magnitude smaller in `p` needs an order of magnitude
more tries for the same chance of detection. Which gives the real conclusion:

> **Raising `tries` is a weak lever. Improving the generator is a strong one.**

Moving the boundary case from `p = 0.005` to `p = 0.25` by targeting the generator
(as in Example 2) is worth more than a 50× increase in `tries`, and it costs nothing
at runtime.

Two important corrections to the table:

- **jqwik injects edge cases deliberately**, so `p` for "zero", "empty list",
  `MAX_VALUE` and similar is effectively 1 at low try counts, not the tiny number
  pure randomness would give. The table applies to values in the interior of your
  domain, not to the boundaries jqwik already knows about.
- **When jqwik generates exhaustively**, `tries` is irrelevant — every value in the
  domain is tested and the guarantee is complete. Check the report: it says when this
  happened. For small enums and booleans, prefer designing the property so exhaustive
  generation is possible.

### A practical budget

| Property kind | `tries` | Why |
|---|---|---|
| Pure calculation (pricing, totals, VAT) | 1,000–10,000 | Microseconds each. Cheap. |
| Stateful chain, in-memory | 200–1,000 | Each try runs a whole sequence. |
| Round-trip serialization | 1,000 | Cheap and high value. |
| Anything touching a container or the network | **10–50, or move it** | 1,000 database round trips per property per build is not a trade anybody should make. |

For the last row, the better answer is almost always to **restructure**: extract the
pure logic into a class you can property-test at 10,000 tries, and keep a handful of
example integration tests against the database. If a property genuinely needs the
database — testing that a repository query matches an in-memory model over generated
data, say — run it as a separate, nightly Maven profile rather than on every build.

### What to actually measure

Do not report `tries`. Report these three:

1. **Coverage of the interesting regions**, from `Statistics.coverage`, as an
   enforced check. This is the number that says the property tests what it claims.
2. **`checks` versus `tries`**, which tells you how much of your budget filtering is
   throwing away. A large gap means Trap 3.
3. **Whether the property has ever failed.** A property that has never gone red in a
   year is either protecting something genuinely stable, or it is vacuous. Deleting
   the production code the property covers and confirming it goes red — a manual
   mutation test, essentially Topic 63 by hand — is a five-minute way to find out.

That third one is worth stating as a rule:

> **A property you have never seen fail is a property you have not yet verified.**
> Break the implementation on purpose once, watch it go red, then fix it. Do this at
> the moment you write it, not later, because later never arrives.

### The relationship to mutation score

They measure different things and compose well:

- **Mutation score (Topic 63)** answers: does the suite notice a change here?
- **A property** answers: does this rule hold across the whole domain?

Running PIT with your properties in `targetTests` is legitimate and informative — a
property that kills mutants an example test cannot is exactly what you hoped for. Two
cautions: properties are slower than examples, so they inflate PIT's runtime, and a
property that fails only for rare inputs may not kill a mutant within its `tries`
budget on a given run, making PIT results slightly non-deterministic. Pin the seed on
properties included in a PIT run.

---

## Practice exercises

### 1 — Easy: one property, one shrink, one regression test

Take `LineTotals.lineMinor` from Example 1.

1. Write the monotonicity property with **no** range constraints on `unitPriceMinor`.
2. Run it. It should fail on overflow. **Paste the full report**: `tries`, `seed`,
   original sample, shrunk sample.
3. Pin the seed and reproduce it.
4. Fix the code with `Math.multiplyExact` so overflow throws instead of wrapping, and
   adjust the property to expect that.
5. Commit the shrunk counterexample as a `@Test`, and remove the pinned seed.

Deliverable: the report from step 2, the fix, and the regression test.

### 2 — Medium: combine it with what you already built

Uses Topics 13 (equals/hashCode), 26 (Optional), 27 (records), 45 (Bean Validation)
and 63 (mutation testing).

Take the surviving mutant you identified in Topic 63's failure drill.

1. Write a **property** that kills it, not another example test.
2. Add a `Statistics.coverage` check proving your generator reaches the relevant
   branch at least 2% of the time. Show the histogram.
3. Run PIT with the property class in `targetTests`. Record before/after mutation
   coverage and test strength in a table.
4. Then write **one property of each of the four shapes** — invariant, round-trip,
   oracle, metamorphic — over `orderflow`'s pricing. The round-trip one is easiest
   over the order DTO's JSON serialization (Topic 19); the metamorphic one is the
   hardest and the most valuable.
5. For each, state in one sentence: what production defect would this catch that an
   example test would not?

### 3 — Hard: production simulation

**Part A — the wallet state machine.** Build the `Chain`/`Transformer` property from
Example 2 over `Wallet`'s full operation set: debit, credit, hold, release, expire a
hold, refund. Assert two invariants: balance never negative, and
`available + held == balance` after every transformation.

**Part B — break it on purpose.** Introduce a real bug: make `release` credit the
balance without decrementing `held`. Run the property. **Record the original sample
length and the shrunk sample length.** Report both numbers and the number of shrink
steps. This is the single best demonstration of why shrinking matters and it is worth
having your own figures for it.

| | Value |
|---|---|
| Operations in the original failing sequence | |
| Operations in the shrunk sequence | |
| Shrink steps | |
| Time to understand the shrunk counterexample | |

**Part C — the generator audit.** Add `Statistics` to every property you have written
and produce a table of which branches of the pricing and wallet code your generators
actually reach, with percentages. Identify the least-reached branch and fix its
generator. Show the before/after distribution.

**Part D — the cost.** Measure the build-time cost of the property suite at
`tries.default` of 100, 1,000 and 10,000. Then decide and defend a project-wide
default, with any per-property overrides named explicitly.

**Part E — argue against yourself.** Name three places in `orderflow` where a property
test would be **worse** than an example test, and say exactly why. Then name the one
place where you would run properties against a real Postgres despite the cost, and
justify the trade.

---

## Interview questions

### Q1 — "What is property-based testing and when would you reach for it?"

**Mid-level answer:** "It generates random inputs and checks that your function
behaves correctly for all of them, instead of you writing individual test cases."

**Senior answer:** "It replaces 'for this input, expect this output' with 'for every
input in this domain, this relationship holds', and the framework hunts for a
counterexample. The half people undersell is **shrinking**: on failure it searches for
the smallest input that still fails, so a 400-element counterexample becomes two
elements you can read. Without that it's just random testing, which produces
unactionable failures. I reach for it where the input space is combinatorial and
enumerating examples is hopeless — discount stacking, partial refunds, retry ordering,
anything with a state machine. I don't reach for it for a single business rule with
three cases; three example tests are clearer and faster. In practice the properties
that pay off are the metamorphic ones — 'adding a line never reduces the total',
'applying a refund twice equals applying it once' — because real business logic rarely
has an independently computable correct answer, but it's full of relationships that
must hold."

**What separates them:** naming shrinking as the load-bearing feature, giving a
selection criterion rather than "use it everywhere", and naming metamorphic properties
specifically.

**Follow-up:** "Give me a property for a shopping cart." Idempotence of adding the same
item twice, monotonicity of the total, or round-tripping through serialization — any
of these is a good answer; "the total is correct" is not.

---

### Q2 — "Why is shrinking important? Isn't the failing input enough?"

**Mid-level answer:** "It makes the failure easier to read."

**Senior answer:** "It's the difference between a bug report and a pile of data. A
failure on a 30-operation wallet sequence with random amounts tells me nothing — I'd
spend an hour bisecting it by hand, which is exactly the work the tool should do. The
shrinker mechanically removes operations and reduces values until it finds a minimal
failing case, and that case is usually so small the bug is visible by inspection: a
hold followed by a debit that overdraws because the hold wasn't counted. Two
operations. The practical consequence for how I write generators is that **shrinking
is a property of the generator, not a global feature** — if I build a value by
consuming a raw `Random`, jqwik has no structural knowledge of it and reports zero
shrink steps. Composing with `Combinators`, `map` and `list()` preserves it, because
jqwik shrinks the source and re-applies my function. That's the single most common way
people accidentally ruin property testing for themselves."

**What separates them:** the concrete before/after, and knowing that unshrinkable
generators are a real failure mode with an observable symptom.

**Follow-up:** "How would you know your generator wasn't shrinking?" `Shrunk Sample
(0 steps)` in the report, with a large original sample.

---

### Q3 — "How many tries should a property run?"

**Mid-level answer:** "The default is fine, maybe a thousand. More if it's an
important test."

**Senior answer:** "`tries` isn't a confidence level and can't be converted into one.
If a bug triggers on a fraction `p` of generated inputs, the miss probability is
`(1-p)^n` — so each order of magnitude smaller in `p` needs an order of magnitude more
tries. Which means raising `tries` is a weak lever and improving the generator is a
strong one. If my quantity generator produces the boundary value 10 half a percent of
the time, targeting the distribution so it lands there 25% of the time beats a 50×
increase in tries and costs nothing at runtime. I'd rather spend effort on
`Statistics.coverage`, which fails the build if the generator doesn't reach the branch
the property claims to test. Two caveats: jqwik injects edge cases deliberately, so
zero and empty and `MAX_VALUE` are effectively always tested regardless of `tries`;
and if the domain is small enough it generates exhaustively, at which point `tries` is
irrelevant and the guarantee is complete. Budget-wise: thousands for pure
calculations, hundreds for stateful chains, and tens if it touches a container — and
if it touches a container I'd normally extract the pure logic instead."

**What separates them:** the arithmetic, the redirection from `tries` to generator
quality, and knowing about edge-case injection and exhaustive generation.

**Follow-up:** "How do you know your generator reaches the interesting case?"
`Statistics.collect` plus an enforced `Statistics.coverage` check.

---

### Q4 — "Your property test fails in CI but passes locally. What do you do?"

**Mid-level answer:** "Re-run it, and if it passes I'd check whether it's flaky and
maybe pin the inputs."

**Senior answer:** "I'd start from the assumption that it is **not** flaky — a
property that fails one run in three is a real bug that triggers on a small fraction
of inputs, which is exactly what the tool exists to find. First I'd get the seed and
the shrunk sample out of the CI log; if CI truncates test output I'd fix that first,
because an unreadable property failure is worthless. Then pin the seed with
`@Property(seed = ...)` to reproduce deterministically, fix the bug, commit the shrunk
counterexample as a plain `@Test` so it can never regress, and **remove** the pinned
seed so the property goes back to hunting. One local trap worth knowing: jqwik
persists failing samples to `.jqwik-database` and replays them first, so 'it failed
then passed locally' often means the database replayed it once and then moved on — not
that you fixed anything. That file belongs in `.gitignore`. The team norm I'd want is:
a property failure is a bug ticket, never a re-run."

**What separates them:** rejecting the flaky diagnosis, the four-step discipline, and
knowing the `.jqwik-database` confusion by name.

**Follow-up:** "What if you genuinely can't reproduce it even with the seed?" Then the
property depends on something outside the generated inputs — a clock, static state, a
shared fixture — and that non-determinism is itself the finding.

---

### Q5 — "What is a bad property?"

**Mid-level answer:** "One that's too slow, or that doesn't actually test anything
useful."

**Senior answer:** "Three specific kinds. First, a **tautology** — the property
reimplements the production algorithm as its oracle, so both sides compute the same
thing and it can't fail on a logic error. The test I apply is: if someone deleted the
implementation and rewrote it from the property alone, would they be forced to write
correct code, or would they have to write *this exact code*? Second, a **vacuous**
property, where the generator never produces inputs that reach the branch the property
claims to cover — a property named `bulkDiscountIsCorrect` with a generator producing
quantities 1 to 5 has been green for eight months and has never once run the discount
branch. That's why I put an enforced `Statistics.coverage` check on any property that
claims to test a branch. Third, an **over-filtered** one, where `filter` rejects
ninety-plus percent of generated values, so the budget is spent on rejection sampling
and jqwik eventually throws `TooManyFilterMissesException` — the fix there is to
construct the constrained value rather than filter for it. And a general one: a
property I have never seen fail is a property I haven't verified. I break the
implementation on purpose once, watch it go red, then fix it."

**What separates them:** three named failure modes with their symptoms and fixes, plus
the "never seen it fail" discipline.

**Follow-up:** "How does this relate to mutation testing?" Both ask whether the suite
would notice a defect; PIT tells you *which* decisions are unconstrained, and a
property is often the right way to constrain a whole region rather than one boundary.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Shrinking is a property of the generator, not a global algorithm. Explain why that
   must be true — what would a generic shrinker have to know that it cannot?

2. jqwik deliberately injects edge cases (`0`, empty list, `MAX_VALUE`) rather than
   relying on randomness. Argue that this is correct. Then construct a case where it
   makes a property misleadingly green.

3. A property with 10,000 tries passes. A colleague says "so it's correct". Give the
   most precise refutation you can, in one sentence, that does not resort to "tests
   can't prove correctness".

4. `filter` and `Assume.that` both discard a try. Name a situation where one is
   clearly better than the other, and say why.

5. You have a property and an example test that both cover the same rule. Under what
   circumstances would you delete the example? Under what circumstances would you
   delete the property? They are not symmetric — say why.

6. Topic 63's mutation testing and this topic both attack "tests that don't verify
   enough". Give a defect that mutation testing would find and a property would not,
   and one that a property would find and mutation testing would not.

7. The metamorphic property "adding a line never reduces the total" fails on
   `orderflow` because of the free-shipping threshold. Three people give three
   different resolutions: narrow the property, change the business rule, or disable
   the property. Argue for each, then say which you would actually do and what
   information would change your mind.

---

## Quick reference card

### Core annotations

```java
@Property                                  // this method is a property (default 1000 tries)
@Property(tries = 5000)
@Property(seed = "-4183872925632481001")   // reproduce a specific failure
@Property(shrinking = ShrinkingMode.FULL)  // BOUNDED (default) | FULL | OFF
@Property(afterFailure = AfterFailureMode.PREVIOUS_SEED)  // | SAMPLE_ONLY | RANDOM_SEED

@ForAll                    // generate this parameter
@ForAll("methodName")      // generate it from a @Provide method
@Provide                   // marks a method returning Arbitrary<T>

@Example                   // a single non-generated case, jqwik's @Test
@Label("...")              // readable name in reports
@StatisticsReport(format = Histogram.class)
```

### Built-in constraints

```java
@IntRange(min = 1, max = 100)    @LongRange(min = 0, max = 1_000_000L)
@Size(min = 1, max = 25)         @NotEmpty   @NotBlank
@AlphaChars   @NumericChars      @Positive   @Negative
@UniqueElements
```

### Arbitraries

```java
Arbitraries.integers().between(1, 100)
Arbitraries.longs().between(0L, 1_000_000L)
Arbitraries.strings().alpha().ofMinLength(1).ofMaxLength(20)
Arbitraries.of("A", "B", "C")                  // fixed set
Arbitraries.of(OrderStatus.class)              // an enum
Arbitraries.just(0L)                           // a constant
Arbitraries.oneOf(a, b, c)                     // pick one arbitrary per value
Arbitraries.frequency(Tuple.of(3, a), Tuple.of(1, b))   // weighted
```

### Combining and transforming

```java
arb.list().ofMinSize(1).ofMaxSize(25)          // List<T>
arb.set() / .map(k, v) / .optional() / .array(T[].class)
arb.map(x -> f(x))                             // shrinkable
arb.filter(x -> p(x))                          // rejection sampling — use sparingly
arb.flatMap(x -> dependentArbitrary(x))        // when one value determines the next
arb.injectNull(0.05)                           // 5% nulls
arb.unique()

Combinators.combine(a, b, c).as(Ctor::new)     // build a record — shrinkable
```

### Statistics — the tool that proves your generator is good

```java
Statistics.label("band").collect(classify(value));
Statistics.label("band").coverage(c -> c.check("AT THRESHOLD").percentage(p -> p >= 2.0));
```

### Stateful

```java
Chain.startWith(() -> new Wallet(id, 10_000L))
     .withTransformer(Arbitraries.longs().between(1, 100)
         .map(n -> Transformer.mutate("debit " + n, w -> w.debit(n))))
     .withMaxTransformations(30);
```

*(Older jqwik versions use the deprecated `Action`/`ActionSequence` API instead —
check which your version ships.)*

### Reading a failure report

| Field | Meaning |
|---|---|
| `tries` | Values generated |
| `checks` | Values actually tested (lower means filtering/`Assume` discards) |
| `seed` | Pin this to reproduce |
| `Original Sample` | The first failing input |
| `Shrunk Sample (K steps)` | **The minimal failing input. Read this one.** |

### Gotchas checklist

- [ ] Every `@Property` parameter needs `@ForAll`.
- [ ] Generators built with `Combinators`/`map`/`list`, never raw `Random` — or
      shrinking dies.
- [ ] Generators deliberately concentrated around the thresholds the code branches on.
- [ ] `Statistics.coverage` on any property claiming to test a branch.
- [ ] `filter` only for cheap rejections; otherwise construct the value.
- [ ] Oracle must be independently correct, not a copy of the implementation.
- [ ] `.jqwik-database` in `.gitignore`.
- [ ] A CI property failure is a bug, not a flake. Capture the seed.
- [ ] Every shrunk counterexample becomes a committed `@Test`, and the pinned seed is
      then removed.
- [ ] Low `tries` for anything touching a container — or extract the pure logic.
- [ ] Break the implementation once and watch the property go red before you trust it.

---

## When would I use this at work?

**1. Any pricing, tax, discount or fee engine.**
These are the canonical case: combinatorial inputs, money on the line, defects that are
silent, and rules that a product owner can state in one sentence ("a discount can never
make the total negative"). One afternoon of properties on a discount stacker
routinely finds two or three cases nobody had considered — most commonly a
composition of two valid discounts producing an invalid result.

**2. Anything with a state machine and an ordering.**
Order status transitions, payment retries, hold/release/capture on a wallet,
idempotent consumers (Topic 116). The bug is almost never in one operation; it is in a
sequence. Stateful properties generate sequences you would not write, and shrinking
hands you the two-step sequence that breaks it.

**3. Serialization and API boundaries.**
`parse(format(x)) == x` is the cheapest high-value property in existence, and it
catches the entire class of "we changed the date format and the mobile client broke"
defects. Pair it with Topic 62's contracts: the contract pins the wire format between
services, the round-trip property pins that your own encode/decode agree with each
other across the whole domain.

---

## Connected topics

**Prerequisites:**
- **58 — JUnit 5**: jqwik is a JUnit Platform engine and runs alongside Jupiter in the
  same module and the same Surefire execution.
- **63 — Mutation testing**: tells you *which* decisions your suite fails to constrain.
  Some want a boundary example; some want a property. This document is the second half
  of that answer.
- **27 — Records**: `Combinators.combine(...).as(Record::new)` is the idiomatic way to
  generate domain values, and records make it a one-liner.
- **13 — equals/hashCode**: the contract itself is four properties (reflexive,
  symmetric, transitive, consistent) and is the classic first jqwik exercise.
- **19 — Serialization**: round-trip properties are the cheapest useful properties you
  will ever write.
- **26 — Optional**: `arb.optional()` and `injectNull` generate the absence cases your
  example tests forget.
- **45 — Bean Validation**: a generator's range constraint is a claim about your
  domain; if validation does not enforce that claim, the property tests a domain
  production does not have.

**This unlocks:**
- **65 — GATE, load testing**: correctness before performance. Properties and mutation
  testing establish that `orderflow` computes the right answer; the gate establishes
  how fast it computes it and under what conditions it stops.
- **52 — Locking** (backwards, extended here): the wallet and inventory invariants you
  state as properties are the same ones that break under concurrency. A single-threaded
  property proves the logic; jcstress (Topic 100) proves it under threads. Different
  tools, same invariant, and writing it down once here pays twice.
- **100 — jcstress**: the concurrency analogue — generate interleavings rather than
  inputs, with the same "state the invariant, let the tool hunt" philosophy.
- **116 — Idempotency**: "applying it twice equals applying it once" is a property, and
  it is the property, so write it here and reuse it there.
- **129 — Capacity and latency budgets**: properties over a capacity model catch
  nonsense parameter combinations before they reach a spreadsheet somebody presents to
  a director.

---

*Java baseline 21; runtime JDK 25. jqwik is a JUnit Platform engine and is
independent of the Spring version, so nothing here changes between Boot 3.x and 4.x.
The one thing that genuinely varies is jqwik's own API generation — the stateful
testing package moved from `net.jqwik.api.stateful` to `net.jqwik.api.state`, and the
configuration file it reads has changed between major versions. Check your version's
user guide and confirm with `mvn dependency:tree -Dincludes=net.jqwik` rather than
trusting any version-specific detail written in a document, including this one.*
