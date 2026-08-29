# 32 — Dependency Resolution: Nearest-Wins vs npm's Nested Tree

## Phase: 3 — Build & Supply Chain
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: The `orderflow` parent POM gains a real `<dependencyManagement>` block with the Spring Boot BOM imported and enforcer rules that fail the build on a version conflict. Every module from Topic 35 onward inherits it.

---

## ELI5 anchor

A dinner party.

**npm's way:** every guest brings their own chair. Alice brings a red chair,
Bob brings a blue chair. Both sit down. Both are happy. The room is crowded but
nothing goes wrong.

**Maven's way:** there is exactly **one chair per seat at the table**. Alice
wants a red chair. Bob wants a blue chair. The host looks at who is closest to
the head of the table, gives that person's chair to the seat, and **tells nobody
that a decision was made.**

Bob sits down expecting blue. He gets red. Dinner proceeds. Nothing is said.

Three hours later Bob reaches for the armrest that only the blue chair had, and
falls on the floor.

That fall is `NoSuchMethodError`. It happens at the moment of use, not at the
moment of seating. That is why it appears in production, at 3am, in a code path
that compiled cleanly six weeks earlier.

---

## The bridge from what you know

### This is the one. Read this section twice.

> **`node_modules` nesting versus Maven's flat classpath is NO ANALOGUE, and it
> is the most important thing in Phase 3.**

Everything else in this phase you can look up. This one installs a wrong model if
you assume your npm intuition transfers, and the wrong model produces silent
production bugs.

### What npm does

npm can install **two versions of the same package** and both work:

```
node_modules/
├── jackson-ish@2.0.0/
├── billing-lib/
│   └── node_modules/
│       └── jackson-ish@1.0.0/       <- its own private copy
└── reporting-lib/
    └── node_modules/
        └── jackson-ish@3.0.0/       <- another private copy
```

Three versions. Three copies on disk. Each `require('jackson-ish')` resolves by
walking up the directory tree, so each library gets the version it asked for.
This is why npm conflicts are usually a *disk space* problem and not a
*correctness* problem.

The mechanism that makes this possible is that **module identity in Node is a
resolved file path.** Two files at two paths are two different modules, even if
their contents are identical.

### What Maven does

Maven produces a **flat classpath**. One list of jars, no nesting:

```
target/classes:
  ~/.m2/repository/com/example/billing-lib/1.4.0/billing-lib-1.4.0.jar:
  ~/.m2/repository/com/example/reporting-lib/2.1.0/reporting-lib-2.1.0.jar:
  ~/.m2/repository/com/example/jackson-ish/2.0.0/jackson-ish-2.0.0.jar
```

Notice what is missing: there is exactly **one** `jackson-ish`. Maven picked one
version and the other two are simply not there.

The mechanism is that **class identity on the JVM is (classloader, fully
qualified class name).** With one application classloader and one classpath,
`com.example.TaxCalculator` can only mean one class. There is no path to
disambiguate on. Two jars containing that class is not "two versions coexisting"
— it is an ambiguity resolved by whichever jar the classloader reaches first,
which is a much worse situation.

### The translation table, ruthlessly

| npm reality | Maven reality | Verdict |
|---|---|---|
| Two versions of a package coexist | Exactly one version of an artifact | **NO ANALOGUE** |
| A version conflict is a warning at install time | A version conflict is silent, resolved, and never mentioned | **NO ANALOGUE** |
| `package-lock.json` pins the entire graph, transitively | A BOM pins versions you named; the rest of the graph is still mediated | **PARTIAL** |
| `npm ls <pkg>` shows every copy | `mvn dependency:tree -Dincludes=<g>:<a>` shows the one that won plus what lost | **PARTIAL** |
| `npm audit` | OWASP dependency-check / Snyk | **HONEST ANALOGUE** (Topic 34) |
| Peer dependencies | Nothing. Closest is `provided` scope plus documentation. | **NO ANALOGUE** |
| `overrides` / `resolutions` in `package.json` | `<dependencyManagement>` | **HONEST ANALOGUE** — this one genuinely transfers |

### The lockfile question, answered honestly

You will ask "where is `package-lock.json`?" The answer is: **Maven does not have
one by default, and a BOM is not a substitute.**

| | `package-lock.json` | Maven BOM / `dependencyManagement` |
|---|---|---|
| Pins versions you declared | yes | yes |
| Pins versions you did **not** declare (deep transitives) | yes | **no** |
| Records integrity hashes | yes | no |
| Fails if the graph changed | yes | no |
| Reproducible across time | yes, if the registry is honest | only for what is managed |

A Boot BOM manages roughly six hundred artifacts, so in practice it covers most
of what a Spring app pulls in. But a library outside the BOM, brought in
transitively by a library outside the BOM, is unmanaged and can move under you.

**Gradle's dependency locking is genuinely closer to a lockfile** — it writes a
`gradle.lockfile` per configuration with every resolved version, and fails the
build if resolution produces anything different. That is Topic 33. If lockfile
guarantees matter enormously to your team, that is a real point in Gradle's
favour, and I would rather you know that than pretend Maven has parity.

What Maven gives you instead is: BOMs, plus the enforcer plugin failing the build
on convergence violations, plus a repository manager that mirrors what you have
already resolved. That is a weaker guarantee achieved with more moving parts.
Say so out loud in an interview and you will sound like someone who has run this
in production.

---

## What is this?

**Dependency mediation** is the algorithm Maven uses to decide which single
version of an artifact goes on the classpath when the graph asks for several.

The rule has two parts:

1. **Nearest wins.** The version at the shallowest depth from your project's POM
   wins. Depth 1 (you declared it) beats depth 2 (a dependency declared it) beats
   depth 3.
2. **Ties break by declaration order.** At equal depth, the version encountered
   first in a depth-first walk of your POM's `<dependencies>` wins.

And one rule that overrides both:

3. **`<dependencyManagement>` in your own POM beats mediation entirely.** If you
   manage a version, that version is used regardless of depth. This is not a
   preference — it is a decision, and it is why BOMs work.

**A BOM** ("Bill of Materials") is a POM with `<packaging>pom</packaging>` whose
entire content is a `<dependencyManagement>` block. It is a version list,
published as an artifact. You import it:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Importing a BOM says: "for any of these several hundred artifacts, if it appears
anywhere in my graph at any depth, use *this* version."

**Note carefully what mediation does NOT do.** It does not warn. It does not
fail. It does not print anything at normal log level. It picks a version and
moves on. This is the entire reason this topic exists.

---

## Why does it matter?

### Because a clean compile is not evidence

Java linking is **lazy**. `javac` checks that the method you call exists in the
class you compiled against, and writes a symbolic reference into the bytecode:

```
invokevirtual com/orderflow/tax/TaxCalculator.applyTax:(JLjava/lang/String;)J
```

That reference is resolved **the first time that instruction executes**, against
whatever class is actually on the runtime classpath. If the runtime class has no
matching method, the JVM throws at that instant.

So the failure mode is:

- Compile: green.
- Unit tests: green, if they do not exercise that path.
- Deploy: fine.
- Startup: fine.
- First request that hits that code path, possibly hours later: `NoSuchMethodError`.

The four errors in this family, and what each one tells you:

| Error | What it means | Typical cause |
|---|---|---|
| `NoSuchMethodError` | The class is there; the method signature is not | A *lower* version won mediation than you compiled against |
| `NoClassDefFoundError` | The class was present at compile time and is absent now | An over-eager `<exclusion>`, or a `provided` scope (Topic 31) |
| `ClassNotFoundException` | Something asked for a class by name and it is not on the classpath | Reflection, service loading, or the cause underneath the above |
| `AbstractMethodError` | A class implements an interface that has since grown a method | Interface and implementation from two different versions |
| `IncompatibleClassChangeError` | The class changed shape — class became interface, field became method | A major-version jump won mediation |

**All five of these are dependency resolution problems until proven otherwise.**
If you see one, your first move is `mvn dependency:tree -Dincludes=`, not opening
the source file in the stack trace.

### Because a transitive bump changes behaviour with no signal

You add one new dependency for an unrelated feature. That dependency brings a
newer version of a library at depth 2. Your existing library was at depth 3.
Nearest wins, the new one takes the slot, and a JSON serialiser somewhere starts
emitting dates in a different format. Nothing failed. Nothing was logged. A
downstream consumer breaks next week and nobody connects the two events.

This is the day-to-day cost, and it is worse than the loud crash.

---

## Syntax breakdown

### `<dependencyManagement>` — the override

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-tax-lib</artifactId>
      <version>2.0.0</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Read as: *"if `orderflow-tax-lib` appears anywhere in the graph at any depth, it
is 2.0.0. I am not asking."*

It adds nothing to the classpath by itself. A module must still declare the
dependency (without a `<version>`) to actually get it.

### `<scope>import</scope>` — how a BOM is consumed

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-dependencies</artifactId>
  <version>${spring-boot.version}</version>
  <type>pom</type>          <!-- REQUIRED. A BOM is a pom, not a jar. -->
  <scope>import</scope>     <!-- Only legal inside dependencyManagement. -->
</dependency>
```

| Element | Why it is there |
|---|---|
| `<type>pom</type>` | Without it Maven looks for a jar that does not exist and fails with `Could not find artifact ... :jar:` |
| `<scope>import</scope>` | Not a classpath scope. It means "splice this BOM's dependencyManagement into mine, here." |

**Import is a splice, not a link.** The BOM's entries are copied in at the point
of the import. That has two consequences people trip over:

1. **Order matters between imported BOMs.** If you import BOM A then BOM B and
   both manage `jackson-databind`, A's version wins, because first declaration
   wins within `dependencyManagement`.
2. **Your own explicit entry always beats any imported BOM**, regardless of
   position, because a directly-declared entry in your `dependencyManagement`
   takes precedence over one spliced in from an import.

So the reliable way to override a Boot-managed version is not "import my BOM
after Boot's". It is:

```xml
<dependencyManagement>
  <dependencies>

    <!-- Your override, stated directly. This wins. -->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>${jackson.override.version}</version>
    </dependency>

    <!-- The Boot BOM, spliced in below. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>

  </dependencies>
</dependencyManagement>
```

> Overriding a Boot-managed version is a decision with consequences: Boot's BOM
> represents a *tested combination*. Overriding one entry takes you off the
> tested path. Do it when you must (a CVE fix — Topic 34), record why, and plan
> to drop the override at the next Boot upgrade.

### `<exclusions>` — the sharp tool

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>reporting-lib</artifactId>
  <exclusions>
    <exclusion>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <!-- no version: exclusions are version-independent -->
    </exclusion>
  </exclusions>
</dependency>
```

An exclusion **removes an artifact from the graph entirely**, along with
everything it would have brought.

**Exclusions are the wrong tool for a version conflict.** They are the right tool
for "I have two competing implementations of the same API and I want exactly one"
— the classic case being logging bridges, where you exclude `commons-logging` so
that `jcl-over-slf4j` can take its place.

Using an exclusion when you meant "use version 2.0.0" gives you no version at
all, and turns a `NoSuchMethodError` into a `NoClassDefFoundError`. Trap 3.

### `<optional>true</optional>`

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <optional>true</optional>
</dependency>
```

"I compile against this, but it does not propagate to anyone who depends on me."
It is how a library offers pluggable integrations without forcing every consumer
to take every one. From the consumer's side, an optional dependency of a
dependency simply is not there — you must declare it yourself if you want it.

### The enforcer plugin — turning silence into failure

This is the single highest-leverage thing in this document.

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-enforcer-plugin</artifactId>
  <executions>
    <execution>
      <id>enforce-dependency-hygiene</id>
      <phase>validate</phase>
      <goals><goal>enforce</goal></goals>
      <configuration>
        <rules>
          <!-- Fail if a transitive dependency asks for a HIGHER version
               than the one mediation selected. This is the exact shape of
               the NoSuchMethodError bug. -->
          <requireUpperBoundDeps/>

          <!-- Ban version ranges and SNAPSHOTs in a release build:
               both make the build non-reproducible. -->
          <banDynamicVersions/>

          <requireMavenVersion>
            <version>[3.9,)</version>
          </requireMavenVersion>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

`requireUpperBoundDeps` is the one that matters. It says: *if anything in the
graph wanted a newer version than the one that won, fail the build.* That
converts the silent downgrade into a `validate`-phase error, before a single
class is compiled.

There is a stricter rule, `dependencyConvergence`, which fails if the graph
requests *any* two different versions of an artifact even when the higher one
won. On a real Spring project it fires constantly and teams end up disabling it.
`requireUpperBoundDeps` catches the dangerous half and is livable. Start there.

---

## Example 1 — minimal

Two dependencies, one shared transitive, one silent decision.

```xml
<dependencies>

  <!-- billing-lib internally depends on tax-lib 1.0.0 -->
  <dependency>
    <groupId>com.orderflow</groupId>
    <artifactId>orderflow-billing-lib</artifactId>
    <version>1.0.0</version>
  </dependency>

  <!-- reporting-lib internally depends on tax-lib 2.0.0 -->
  <dependency>
    <groupId>com.orderflow</groupId>
    <artifactId>orderflow-reporting-lib</artifactId>
    <version>1.0.0</version>
  </dependency>

</dependencies>
```

The graph:

```
your-app
├── orderflow-billing-lib:1.0.0        (depth 1)
│   └── orderflow-tax-lib:1.0.0        (depth 2)
└── orderflow-reporting-lib:1.0.0      (depth 1)
    └── orderflow-tax-lib:2.0.0        (depth 2)
```

Both candidates are at depth 2. **Tie. Declaration order decides.**
`orderflow-billing-lib` is listed first, so **`tax-lib:1.0.0` wins.**

Swap the two `<dependency>` blocks in the POM and `tax-lib:2.0.0` wins instead.

Sit with that for a second. **Reordering two lines in an XML file changes which
version of a third library runs in production, with no error and no warning.**
There is nothing in your npm experience that behaves like this.

The fix, always:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-tax-lib</artifactId>
      <version>2.0.0</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Now the answer is 2.0.0, it is written down, it is in version control, and
reordering the dependency list changes nothing.

---

## Example 2 — production scenario: `orderflow` and a Jackson downgrade

The setup, which is entirely ordinary:

`orderflow-api` is a Spring Boot 4.1 service. Boot's BOM manages Jackson. In
Boot 4, **Jackson 3 is the standard** and Jackson 2 support is deprecated — so
the BOM's managed Jackson artifacts are what every Spring component compiles and
runs against.

Someone adds a dependency for a new feature: a third-party PDF-invoice library,
`com.vendor:invoice-renderer`. It is not in Boot's BOM. It internally depends on
an older Jackson.

```xml
<!-- orderflow-api/pom.xml -->
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- new, for the invoice PDF feature -->
  <dependency>
    <groupId>com.vendor</groupId>
    <artifactId>invoice-renderer</artifactId>
    <version>3.2.0</version>
  </dependency>
</dependencies>
```

### Why this is safe here, and exactly when it stops being safe

Jackson's artifacts **are** managed by the Boot BOM. So `dependencyManagement`
wins over mediation, the BOM's version is used, and the vendor's older Jackson
never gets a look in. The build is fine. This is the BOM earning its keep,
silently, on a Tuesday.

Now change one thing: the vendor library depends on a library that is **not** in
the BOM — say a validation helper, or a PDF layout engine, or a date library —
and `orderflow-persistence` also depends on a newer version of that same library
at greater depth.

```
orderflow-api
├── orderflow-persistence:1.0.0-SNAPSHOT          (depth 1)
│   └── com.vendor:layout-engine:4.1.0            (depth 2)
└── com.vendor:invoice-renderer:3.2.0             (depth 1)
    └── com.vendor:layout-engine:3.0.0            (depth 2)
```

Tie at depth 2. `orderflow-persistence` is declared first, so **4.1.0 wins** —
this time by luck. Six months later somebody alphabetises the dependency block,
`invoice-renderer` moves above `orderflow-persistence`, **3.0.0 wins**, and the
persistence module throws `NoSuchMethodError` on a code path that has not been
touched in half a year.

### The `orderflow` parent that prevents all of this

```xml
<!-- orderflow/pom.xml (parent) -->
<properties>
  <!-- Set to the current Spring Boot 4.1.x release.
       Find it with: mvn versions:display-property-updates
       I am not writing a patch number from memory. -->
  <spring-boot.version>SET-ME</spring-boot.version>
</properties>

<dependencyManagement>
  <dependencies>

    <!-- 1. Our own explicit pins come FIRST. These beat every import. -->
    <dependency>
      <groupId>com.vendor</groupId>
      <artifactId>layout-engine</artifactId>
      <version>4.1.0</version>
    </dependency>

    <!-- 2. Our own modules. -->
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-domain</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-persistence</artifactId>
      <version>${project.version}</version>
    </dependency>

    <!-- 3. The Boot BOM, spliced in last. Manages ~600 artifacts. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>

  </dependencies>
</dependencyManagement>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-enforcer-plugin</artifactId>
      <executions>
        <execution>
          <id>enforce-dependency-hygiene</id>
          <phase>validate</phase>
          <goals><goal>enforce</goal></goals>
          <configuration>
            <rules>
              <requireUpperBoundDeps/>
              <banDynamicVersions/>
            </rules>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Three properties this gives you, and each is worth stating explicitly:

1. **The Boot BOM covers the framework's world**, so no Spring, Jackson,
   Hibernate, Micrometer or Tomcat version can be moved by a transitive.
2. **Our explicit pin covers the vendor's world**, the part the BOM cannot know
   about, and it is written where a reviewer will see it.
3. **The enforcer makes a future violation loud.** The next person who adds a
   library wanting a higher `layout-engine` gets a build failure at `validate`,
   not a `NoSuchMethodError` in production.

> **[BOOT 3.x DELTA]**
> Two things change if you are on a 3.x codebase.
> **First**, Boot 4 modularised into many smaller jars, so an artifact that
> carried a feature in 3.x may not exist under the same coordinates in 4.x. That
> means a `<dependencyManagement>` override you copied from a 3.x project can
> silently manage an artifact that is no longer on your classpath — it will not
> error, it will just do nothing. Verify every override with
> `mvn dependency:tree -Dincludes=<groupId>:<artifactId>` and confirm the
> artifact actually appears.
> **Second**, Boot 3.5 left OSS support in June 2026. Its BOM is frozen. Any CVE
> in a managed artifact must be fixed by *you*, with an explicit override in your
> own `dependencyManagement` — which is exactly the off-the-tested-path situation
> described above, now as a permanent condition rather than a temporary one. That
> is the real cost of staying on an EOL line, and it is Topic 34's subject.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — assuming a conflict will be reported

**Wrong:** treating a green `mvn clean verify` as evidence that the dependency
graph is consistent.

**Exact symptom:** none at build time. That is the trap. Then, at some later
point:

```
java.lang.NoSuchMethodError: 'long com.orderflow.tax.TaxCalculator.applyTax(long, java.lang.String)'
	at com.orderflow.pricing.PriceEngine.priceOrder(PriceEngine.java:64)
	at com.orderflow.api.OrderController.placeOrder(OrderController.java:41)
```

(That is the *shape* of the message on a modern JVM — it names the full signature
it looked for. It is an illustration, not captured output.)

**Root cause:** mediation resolved the conflict at build time, chose a version,
logged nothing. `javac` compiled `PriceEngine` against the version *it* saw at
compile time. The classpath at runtime carries a different one.

**Fix, in order:**

```bash
# 1. Who is on the classpath, and who lost?
mvn dependency:tree -Dverbose -Dincludes=com.orderflow:orderflow-tax-lib

# 2. Pin it.
#    <dependencyManagement> entry with the version you actually want.

# 3. Make the next one loud.
#    maven-enforcer-plugin with <requireUpperBoundDeps/>.
```

---

### Trap 2 — pinning versions module by module instead of importing a BOM

**Wrong:**

```xml
<!-- in orderflow-api -->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-web</artifactId>
  <version>7.0.1</version>
</dependency>

<!-- in orderflow-persistence, six months later, different developer -->
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-tx</artifactId>
  <version>7.0.4</version>
</dependency>
```

Nobody did anything obviously wrong. Each developer pinned the version they
needed, in their own module.

**Exact symptom:** the application context fails to start, and the stack trace is
entirely inside framework code, mentioning nothing you wrote:

```
Caused by: java.lang.NoSuchMethodError:
  'void org.springframework.core.SomeInternalClass.<init>(...)'
	at org.springframework.context.support.AbstractApplicationContext.refresh(...)
```

Or subtler: it starts fine and a specific feature misbehaves.

**Root cause:** Spring's modules are released as a **set** and are not
independently version-compatible. `spring-web` 7.0.1 calls internal methods of
`spring-core` that may only exist at 7.0.1's matching version. Individual pins
guarantee drift, because there is no mechanism forcing them to move together.

**Fix:** delete every individual version. Import `spring-boot-dependencies` once
in the parent. Declare dependencies with **no `<version>` element at all** in
every module. Then the whole set moves together when you bump one property.

**The reviewable rule:** a `<version>` element inside a `<dependencies>` block
(not `dependencyManagement`) in a module POM is a code smell. There are legitimate
cases — a library genuinely outside every BOM — but each one should be a
conscious decision, not a default.

---

### Trap 3 — using `<exclusions>` to solve a version conflict

**Wrong:**

```xml
<dependency>
  <groupId>com.vendor</groupId>
  <artifactId>invoice-renderer</artifactId>
  <version>3.2.0</version>
  <exclusions>
    <exclusion>
      <groupId>com.vendor</groupId>
      <artifactId>layout-engine</artifactId>   <!-- "the old one is causing problems" -->
    </exclusion>
  </exclusions>
</dependency>
```

**Exact symptom:** the `NoSuchMethodError` is gone. It has been replaced by:

```
java.lang.NoClassDefFoundError: com/vendor/layout/PageLayout
	at com.vendor.invoice.InvoiceRenderer.render(InvoiceRenderer.java:88)
Caused by: java.lang.ClassNotFoundException: com.vendor.layout.PageLayout
```

If `orderflow-persistence` happened to also pull `layout-engine` in, the
exclusion only removed it from *that one path* and the class is still present —
so it works, until someone refactors the persistence module and it stops.

**Root cause:** an exclusion removes the artifact from the graph. It does not
select a version. You asked for "not the old one" and got "not any of them".

**Fix:**

```xml
<!-- Say which version you want. Do not say which one you do not want. -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.vendor</groupId>
      <artifactId>layout-engine</artifactId>
      <version>4.1.0</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Exclusions are correct** when you want an artifact genuinely gone: competing
logging bridges (exclude `commons-logging` so `jcl-over-slf4j` can take the API),
an embedded server you are replacing (exclude `spring-boot-starter-tomcat` when
using Undertow or Jetty), a transitively-pulled JSON library you have no use for.

---

### Trap 4 — relying on a transitive dependency you never declared

**Wrong:** your code does `import org.apache.commons.lang3.StringUtils;` and your
POM never mentions `commons-lang3`. It compiles, because something else brought
it in.

**Exact symptom:** the build breaks on a day you changed nothing relevant:

```
[ERROR] .../OrderNormaliser.java:[12,38] package org.apache.commons.lang3 does not exist
```

The commit that broke it upgraded an unrelated library, which happened to stop
depending on `commons-lang3`.

**Root cause:** you took a compile-time dependency on something you never
declared. Its presence was an accident of somebody else's transitive graph, and
that graph is not yours to control.

**Fix:** run `mvn dependency:analyze` and declare everything under
`Used undeclared dependencies`. Consider running it as a build gate:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <executions>
    <execution>
      <id>analyze</id>
      <goals><goal>analyze-only</goal></goals>
      <configuration>
        <failOnWarning>true</failOnWarning>
        <ignoredUnusedDeclaredDependencies>
          <!-- runtime-only artifacts legitimately look "unused" -->
          <dep>org.postgresql:postgresql</dep>
        </ignoredUnusedDeclaredDependencies>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Introduce it on a new module first. Retrofitting it to an old codebase produces a
long list and a bad afternoon.

---

### Trap 5 — expecting a later BOM import to override an earlier one

**Wrong:**

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type><scope>import</scope>
    </dependency>

    <!-- "our overrides, applied afterwards" -->
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-platform-bom</artifactId>
      <version>${orderflow-platform.version}</version>
      <type>pom</type><scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Exact symptom:** you bump a version in `orderflow-platform-bom`, rebuild, and
`mvn dependency:tree` shows the old version still winning. No error. No warning.
The change appears to have done nothing, and people start doubting the build
cache.

**Root cause:** BOM imports are spliced in **in order**, and within
`dependencyManagement` the **first** declaration of a given artifact wins. Boot's
BOM was imported first, so for every artifact both BOMs manage, Boot's version is
already established by the time yours is spliced in.

**Fix — two options, and prefer the first:**

1. **State the override directly** in your own `dependencyManagement`, above the
   imports. A directly-declared entry beats any import, at any position. This is
   unambiguous and a reviewer can see it.
2. Move your BOM import **above** Boot's. This works, but it is positional magic
   — the next person to tidy the block will break it and get no error.

**How to prove which you are getting:** `mvn help:effective-pom` and read the
resolved `dependencyManagement` block. It shows the flattened, decided list.

---

## Hands-on proof

Every command below is one **you** run. I have no Maven and no JVM here, so
nothing below is captured output. Where I show the shape of a line, it is labelled
as an illustration.

### Proof 1 — read the tree

```bash
mvn dependency:tree
```

**What to look for:** an indented tree of `groupId:artifactId:packaging:version:scope`.
The indentation is depth. Depth is what mediation uses.

**This is an illustration of the format, not captured output:**

```
[INFO] com.orderflow:orderflow-api:jar:1.0.0-SNAPSHOT
[INFO] +- com.orderflow:orderflow-persistence:jar:1.0.0-SNAPSHOT:compile
[INFO] |  \- com.vendor:layout-engine:jar:4.1.0:compile
[INFO] \- com.vendor:invoice-renderer:jar:3.2.0:compile
```

Read it as: `+-` and `\-` are siblings at the same depth; `|` continues a branch.
The scope is the last field, and it is the **effective** scope after
transitivity (Topic 31's second table), not what the other project declared.

| What you see | What it means |
|---|---|
| An artifact appears once | Normal. Mediation already ran; you are seeing the winner. |
| An artifact you never declared, at depth 3 | Transitive. Something two levels up wants it. |
| Scope `runtime` on something you declared `compile` | Transitivity demoted it, or a `dependencyManagement` entry set the scope. |
| The tree is enormous | Normal for Spring Boot. Filter it — see Proof 3. |

### Proof 2 — see what LOST

```bash
mvn dependency:tree -Dverbose
```

**What to look for:** lines containing `omitted for conflict with` and
`omitted for duplicate`. These are the losers of mediation, and they are the only
place Maven ever tells you a decision was made.

**Illustration of the shape of a conflict line — not captured output:**

```
[INFO] |  \- (com.vendor:layout-engine:jar:3.0.0:compile - omitted for conflict with 4.1.0)
```

Read it as: *"this branch asked for 3.0.0; 4.1.0 is what is actually on the
classpath."*

| What you see | What it means | What to do |
|---|---|---|
| `omitted for conflict with <higher version>` | A lower version lost. Usually harmless. | Note it; no action unless the winner is unexpected. |
| `omitted for conflict with <lower version>` | **A higher version lost.** Something in your graph compiled against APIs that are not on the classpath. | This is the dangerous one. Pin the higher version. |
| `omitted for duplicate` | The same version reached via two paths. | Nothing. Cosmetic. |
| `omitted for cycle` | A dependency cycle in someone else's published POMs. | Nothing you can fix; note it. |

> **Honest caveat about `-Dverbose`.** Depending on your `maven-dependency-plugin`
> version, verbose mode may print a warning that the verbose tree is computed with
> a **different algorithm** than the one that produces the real classpath, and so
> may not match resolution exactly. Treat the verbose tree as a strong diagnostic
> hint, and confirm the actual winner with the non-verbose tree or
> `mvn dependency:list`. If your plugin prints such a warning, believe it.

### Proof 3 — filter to one artifact

```bash
mvn dependency:tree -Dverbose -Dincludes=com.vendor:layout-engine
```

**What to look for:** only the paths that reach that artifact, with every
requesting branch shown. This is the command you run when you have a
`NoSuchMethodError` and a class name.

| What you see | What it means |
|---|---|
| Nothing at all | The artifact is not in this module's graph. You are looking at the wrong module — try `-pl <module>` or run from the module directory. |
| One path, one version | No conflict here. Your problem is elsewhere (scope, packaging, or a genuinely missing dependency). |
| Two or more paths, one `omitted for conflict` | You have found it. |

`-Dincludes` accepts wildcards: `-Dincludes=com.fasterxml.jackson.core:*` or
`-Dincludes=:*jackson*`.

### Proof 4 — the flat truth

```bash
mvn dependency:list -DincludeScope=runtime | sort
```

**What to look for:** one line per artifact, one version each. This is closer to
"what is actually on the classpath" than the tree, because the tree shows
structure and this shows the result.

| What you see | What it means |
|---|---|
| One entry per `groupId:artifactId` | Correct. Maven guarantees this. |
| Two entries with the same artifactId but different groupIds | **Not a Maven conflict — a relocation.** An artifact moved namespaces and you now have both copies with the same classes inside. Maven will not catch this; see Proof 7. |

### Proof 5 — where did this version come from?

```bash
mvn help:effective-pom -Doutput=effective.xml
grep -A3 -B3 'layout-engine' effective.xml
```

**What to look for:** whether the artifact appears in the resolved
`<dependencyManagement>` and with which version.

| What you see | What it means |
|---|---|
| Present in `dependencyManagement` with your version | Your pin is active. Mediation is not deciding this. |
| Present with a version you did not write | It came from a BOM. Find which one — check import order (Trap 5). |
| Absent from `dependencyManagement` entirely | Mediation is deciding. If this matters, pin it. |

### Proof 6 — turn silence into a build failure

Add `requireUpperBoundDeps` to the enforcer as shown above, then:

```bash
mvn -q validate
```

| What you see | What it means |
|---|---|
| Build succeeds | No artifact lost mediation to a lower version. This is a real, checkable property. |
| `Failed while enforcing RequireUpperBoundDeps` with a list of paths | Each listed path wanted a higher version than won. Read the paths; each is a latent `NoSuchMethodError`. |
| Dozens of violations on an existing project | Normal for an old codebase. Fix them one at a time, or start with the rule on new modules only. |

### Proof 7 — duplicate classes across different artifacts

Maven guarantees one version per artifact. It does **not** guarantee one
definition per class. Two *different* artifacts can contain the same fully
qualified class (relocations, shaded jars, `javax`/`jakarta` overlaps).

```bash
mvn dependency:build-classpath -Dmdep.outputFile=cp.txt -DincludeScope=runtime
tr ':' '\n' < cp.txt | while read -r j; do
  [ -f "$j" ] && unzip -l "$j" 2>/dev/null | awk -v J="$j" '/\.class$/ {print $4, J}'
done | sort | awk '{ if ($1 == prev) print "DUPLICATE: " $1; prev = $1 }' | sort -u | head -40
```

**What to look for:** any class name appearing in two different jars.

| What you see | What it means |
|---|---|
| No output | Good. No duplicate class definitions on the runtime classpath. |
| A handful of `module-info.class` or `META-INF` entries | Ignore; those are per-jar by design. |
| Real application/library classes duplicated | **Which one wins is classpath order, and classpath order is not something you should be relying on.** Exclude one of the artifacts. |

For a maintained version of this check, the `extra-enforcer-rules` project
provides a `banDuplicateClasses` rule and there is a `duplicate-finder` plugin.
Check current coordinates before adding either; I am not quoting them from
memory.

---

## Failure drill  `[BONUS]`

**Goal:** deliberately force a diamond version conflict, produce a
`NoSuchMethodError` at runtime from a completely clean compile, then fix it with
`<dependencyManagement>` and prove the fix.

This drill is assigned in the master plan's failure-drill map. Do not skip it.
Reading about silent mediation and *watching your own code break because of it*
are different levels of knowing.

Budget: about 45 minutes.

### Setup — three tiny projects

```bash
mkdir -p ~/java-lab/32/{tax-lib,pricing-lib,app} && cd ~/java-lab/32
```

**1. `tax-lib` version 1.0.0** — the old API.

`tax-lib/pom.xml`:
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-tax-lib</artifactId>
  <version>1.0.0</version>
  <properties><maven.compiler.release>21</maven.compiler.release></properties>
</project>
```

`tax-lib/src/main/java/com/orderflow/tax/TaxCalculator.java`:
```java
package com.orderflow.tax;

public final class TaxCalculator {
    /** v1 API: one rate for everyone. Amounts are minor units (pence). */
    public long applyTax(long amountMinorUnits) {
        return amountMinorUnits + (amountMinorUnits * 20 / 100);
    }
}
```

```bash
cd tax-lib && mvn -q clean install && cd ..
```

**2. `tax-lib` version 2.0.0** — the *breaking* change.

Edit `tax-lib/pom.xml`: change `<version>1.0.0</version>` to `2.0.0`.
Edit `TaxCalculator.java` — **replace** the method, do not overload it:

```java
package com.orderflow.tax;

public final class TaxCalculator {
    /** v2 API: rate depends on the destination country. */
    public long applyTax(long amountMinorUnits, String countryCode) {
        int rate = "GB".equals(countryCode) ? 20 : 0;
        return amountMinorUnits + (amountMinorUnits * rate / 100);
    }
}
```

```bash
cd tax-lib && mvn -q clean install && cd ..
```

You now have **both versions** in `~/.m2/repository/com/orderflow/orderflow-tax-lib/`.
Confirm it:

```bash
ls ~/.m2/repository/com/orderflow/orderflow-tax-lib/
```

> This is Topic 31's Trap 1 working *for* you: `install` put both in your local
> repo. In a real org these would come from a shared repository manager. The
> mechanics are identical.

**3. `pricing-lib`** — compiles against **v2**.

`pricing-lib/pom.xml`:
```xml
  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-pricing-lib</artifactId>
  <version>1.0.0</version>
  <properties><maven.compiler.release>21</maven.compiler.release></properties>
  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-tax-lib</artifactId>
      <version>2.0.0</version>
    </dependency>
  </dependencies>
```

`pricing-lib/src/main/java/com/orderflow/pricing/PriceEngine.java`:
```java
package com.orderflow.pricing;

import com.orderflow.tax.TaxCalculator;

public final class PriceEngine {
    private final TaxCalculator tax = new TaxCalculator();

    public long priceOrder(long subtotalMinorUnits, String countryCode) {
        return tax.applyTax(subtotalMinorUnits, countryCode);   // v2 signature
    }
}
```

```bash
cd pricing-lib && mvn -q clean install && cd ..
```

**4. `app`** — declares `tax-lib` **1.0.0 directly**, at depth 1.

`app/pom.xml`:
```xml
  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-drill-app</artifactId>
  <version>1.0.0</version>
  <properties><maven.compiler.release>21</maven.compiler.release></properties>
  <dependencies>

    <!-- depth 1. Nearest wins. This is the whole trap. -->
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-tax-lib</artifactId>
      <version>1.0.0</version>
    </dependency>

    <!-- depth 1, and it brings tax-lib 2.0.0 at depth 2 -->
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-pricing-lib</artifactId>
      <version>1.0.0</version>
    </dependency>

  </dependencies>
```

`app/src/main/java/com/orderflow/app/Main.java`:
```java
package com.orderflow.app;

import com.orderflow.pricing.PriceEngine;

public class Main {
    public static void main(String[] args) {
        System.out.println("app started, classpath linked lazily");
        long total = new PriceEngine().priceOrder(1999L, "GB");
        System.out.println("total = " + total);
    }
}
```

Notice: `Main` never mentions `TaxCalculator`. It compiles against `PriceEngine`
only. There is nothing in `app`'s source that could possibly warn you.

### Step 1 — build it and watch it succeed

```bash
cd app && mvn -q clean package
```

**What to capture:** the fact that this succeeds. Write down "compile: clean, no
warnings". You are about to need that as evidence.

### Step 2 — see the decision Maven made and never mentioned

```bash
mvn dependency:tree -Dverbose -Dincludes=com.orderflow:orderflow-tax-lib
```

**What to capture:** the exact line containing `omitted for conflict with`.

**What you are looking for (illustration of the shape, not captured output):**

```
[INFO] +- com.orderflow:orderflow-tax-lib:jar:1.0.0:compile
[INFO] \- com.orderflow:orderflow-pricing-lib:jar:1.0.0:compile
[INFO]    \- (com.orderflow:orderflow-tax-lib:jar:2.0.0:compile - omitted for conflict with 1.0.0)
```

**How to read it:**

| What you see | What it means |
|---|---|
| `omitted for conflict with 1.0.0` on the 2.0.0 line | The drill is set up correctly. The **higher** version lost, which is the dangerous direction. |
| `omitted for conflict with 2.0.0` on the 1.0.0 line | Your depths are the wrong way round. Check that `tax-lib` is declared directly in `app/pom.xml`. |
| Only one `tax-lib` line, no omission | Either `pricing-lib` did not install with the 2.0.0 dependency, or you have a stale `~/.m2` entry. Re-run `mvn -U clean install` in `pricing-lib`. |
| A plugin warning about verbose mode using a different algorithm | Expected on some plugin versions. Cross-check with `mvn dependency:list \| grep tax-lib` — it must show **1.0.0**. |

Also confirm the flat truth:

```bash
mvn dependency:list | grep tax-lib
```

It must show exactly one line, `1.0.0`. One artifact. One version. Flat classpath.

### Step 3 — run it and get the error

```bash
mvn -q dependency:build-classpath -Dmdep.outputFile=cp.txt
java -cp "target/classes:$(cat cp.txt)" com.orderflow.app.Main
```

**What to capture:** the complete stack trace, verbatim, into a file.

**What you should see** — note that `app started...` prints **first**:

| What you see | What it means |
|---|---|
| `app started`, then a `NoSuchMethodError` naming `applyTax(long, java.lang.String)` | The drill worked. This is the whole lesson. |
| The error names `com.orderflow.tax.TaxCalculator.applyTax` | Confirms the resolution is the cause, not your code. |
| It fails at `PriceEngine.priceOrder`, not in `Main` | Linking happened at the point of *use*, inside a library you did not write. |
| No error, and a total printed | v2 is on the classpath. Re-check Step 2 — something pinned 2.0.0. |
| `NoClassDefFoundError` instead | You excluded `tax-lib` somewhere instead of letting mediation pick 1.0.0. |

**The single most important observation:** the JVM printed a line of your output
*before* it failed. Class linking is lazy. The bad jar was on the classpath from
the first millisecond and the JVM said nothing until the instruction that needed
the missing method actually executed. In a web service, that instruction might
first execute on a code path exercised once a month.

Write that sentence in your own words before continuing.

### Step 4 — fix it with `<dependencyManagement>`

Add to `app/pom.xml`, **above** `<dependencies>`:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-tax-lib</artifactId>
      <version>2.0.0</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**Leave the `<dependencies>` block exactly as it is** — `tax-lib` is still
declared there at depth 1 with `<version>1.0.0</version>`.

Wait. That is the interesting part. Remove the `<version>1.0.0</version>` from
the `<dependencies>` entry so the managed version applies, and re-run:

```bash
mvn -q clean package
mvn dependency:tree -Dverbose -Dincludes=com.orderflow:orderflow-tax-lib
mvn -q dependency:build-classpath -Dmdep.outputFile=cp.txt
java -cp "target/classes:$(cat cp.txt)" com.orderflow.app.Main
```

| What you see | What it means |
|---|---|
| Tree now shows `2.0.0` winning, `1.0.0` omitted for conflict | `dependencyManagement` overrode the resolution. |
| The program prints a total | Fixed. |
| Still 1.0.0 | You left an explicit `<version>1.0.0</version>` on the direct dependency. A version written *directly on a dependency* is the strongest statement of all — it beats management. Remove it. |

**Then do the experiment that teaches the most:** put
`<version>1.0.0</version>` back on the direct `<dependency>` while leaving the
`dependencyManagement` entry at 2.0.0. Run the tree again.

**What you have just proved:** the precedence chain, from strongest to weakest,
is:

```
1. An explicit <version> on the dependency in THIS pom
2. <dependencyManagement> in THIS pom (including inherited from a parent)
3. Mediation: nearest wins, then declaration order
```

### Step 5 — make it impossible to happen again

Add the enforcer with `requireUpperBoundDeps` to `app/pom.xml`. Remove the
`dependencyManagement` fix. Run:

```bash
mvn -q validate
```

**What to capture:** the enforcer failure message, and note the phase — it is
`validate`, which runs **before compilation**.

| What you see | What it means |
|---|---|
| `RequireUpperBoundDeps failed` listing the tax-lib path | The rule catches exactly this bug class, at the earliest possible moment, with the dependency path printed for you. |
| Build passes | Either the rule is not configured, or the fix is still in place. Check both. |

### What the fix proves

Write these down in your own words. This is the retained part.

1. **Mediation is silent by design.** The build tool made a decision that changed
   which code runs, and reported nothing at default verbosity. The only record of
   it is `-Dverbose`, which you have to know to ask for.
2. **A clean compile is evidence about the compile classpath, and nothing else.**
   `javac` verified `PriceEngine` against tax-lib 2.0.0 and was completely
   correct to do so. The runtime classpath is a different set.
3. **The error surfaces at the point of use, not at startup.** Lazy linking means
   the blast radius is "whenever that code path first runs", which in a service
   is unbounded.
4. **`dependencyManagement` is a decision, not a suggestion.** It sits above
   mediation in the precedence chain, and that is precisely why a BOM works at
   the scale of six hundred artifacts.
5. **`requireUpperBoundDeps` converts an unbounded runtime risk into a
   `validate`-phase build failure.** That trade — a slightly noisier build for
   a class of production bug you can no longer ship — is the reason it belongs in
   every parent POM you own.

---

## Practice exercises

### 1 — Easy: read a real tree

Take the `orderflow-api` module you built in Topic 31.

1. `mvn dependency:tree` and count the total artifacts. Write the number down.
2. `mvn dependency:tree -Dverbose | grep -c "omitted for conflict"` — how many
   silent decisions did Maven make in a project you thought you controlled?
3. Pick three of those conflicts. For each, use `-Dincludes` to see the full
   paths and answer: did the higher or the lower version win?
4. Run `mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-databind`.
   Which version won, and which POM decided it? Prove your answer with
   `mvn help:effective-pom`, not by guessing.

### 2 — Medium: combining earlier topics

Build a module that uses material from Phases 1 and 2, then break its
dependencies.

1. Write an `OrderSummary` **record** (Topic 27) containing a `List<OrderLine>`,
   and a `sealed interface PaymentResult` (Topic 28). Serialise `OrderSummary` to
   JSON with Jackson and assert the round-trip in a JUnit test.
2. Note that your POM declares **no Jackson version** — the Boot BOM manages it.
   Confirm with `mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:*`.
3. Now add an explicit `<dependency>` on `jackson-databind` with a version
   **two minor versions older** than the managed one. Rebuild.
   - Does it compile? Record the answer.
   - Do the tests pass? Record the answer.
   - Does the Spring context start (write a trivial
     `@SpringBootTest`)? Record the exact error if not.
4. Explain, referring to Topic 19 (serialization) and Topic 27 (records
   deserialise through the canonical constructor), why a Jackson version mismatch
   is more likely to surface as a *runtime* failure than a compile failure.
5. Remove the explicit version. Add `requireUpperBoundDeps`. Re-add the old
   version. Record the difference in *when* you found out.

### 3 — Hard: production simulation on `orderflow`

Take the four-module skeleton from Topic 31 and make its dependency graph
defensible.

**Part A — establish the baseline.** For each of the four modules, record:
- total artifact count from `dependency:tree`
- count of `omitted for conflict` lines from `-Dverbose`
- the output of `dependency:analyze`

**Part B — introduce a real conflict.** Add to `orderflow-persistence` a
dependency on a small third-party library that is **not** managed by the Boot BOM
and that itself depends on something Boot *does* manage (a JSON library, a
logging bridge, or a validation helper are all common). Then:
- Show with `-Dverbose` which version won.
- Predict, before running anything, whether the Boot-managed version or the
  library's version is on the classpath. Then verify. If you were wrong, work out
  which precedence rule you misapplied.

**Part C — build the guard rails.** In the parent POM:
- Add `maven-enforcer-plugin` with `requireUpperBoundDeps` and
  `banDynamicVersions`.
- Add `maven-dependency-plugin`'s `analyze-only` with `failOnWarning`, with an
  explicit ignore list for genuinely runtime-only artifacts.
- Get all four modules green. Record every violation you had to fix and how.

**Part D — the ordering experiment.** In `orderflow-api`, create a genuine
depth-2 tie by depending on two libraries that both pull the same transitive.
Build, record the winner. Swap the two `<dependency>` blocks. Build again, record
the winner. Then add a `<dependencyManagement>` entry and repeat both orderings.

Write four sentences: what changed, what did not, why, and what you would say in
a code review to a colleague who says "the dependency order in this POM is just
style".

**Part E — the honest lockfile answer.** Maven has no lockfile. Design the
closest thing you can with what you have: BOM coverage, enforcer rules, a
committed `mvn dependency:list` snapshot with a CI diff, a repository manager
mirror, and banned dynamic versions. Then write one paragraph on what your scheme
still does **not** guarantee that `package-lock.json` does. Be specific. You will
reuse this answer in Topic 33 and again in Topic 34.

---

## Interview questions

### Q1 — "You get a `NoSuchMethodError` in production but the code compiles fine. Walk me through it."

**Mid-level answer:** "There is a version mismatch between compile time and
runtime. I would check the dependency versions."

**Senior answer:** "That error means the class is present and the *signature* is
not, so it is a classpath problem, not a code problem — and I would not open the
source file first. Java links symbolically and lazily: `javac` writes a method
reference into the bytecode and the JVM resolves it the first time that
instruction executes. So the failure surfaces at the point of use, which is why
it can be hours after startup on a code path unit tests never hit.

My sequence is: pull the class name from the trace, run
`mvn dependency:tree -Dverbose -Dincludes=<groupId>:<artifactId>`, and look for
`omitted for conflict with`. In Maven the classpath is flat with exactly one
version per artifact, resolved by nearest-wins and then declaration order, and
nothing is printed when that decision is made — so a transitive that shortened
the path to a lower version silently downgraded us. The fix is a
`<dependencyManagement>` entry pinning the version, or importing the BOM if the
artifact belongs to a family. Then I add `requireUpperBoundDeps` to the enforcer
so the next occurrence fails at `validate` rather than in production.

Worth adding: if it were `NoClassDefFoundError` instead, I would suspect an
`<exclusion>` or a `provided` scope rather than mediation."

**What separates them:** naming *lazy symbolic linking* as the reason for the
delayed failure, giving an ordered diagnostic procedure with real commands, and
distinguishing `NoSuchMethodError` from `NoClassDefFoundError` by cause.

**Follow-up:** "Why did the tests not catch it?" Looking for: unit tests use the
same classpath, so they *would* catch it if they exercised that path — the honest
answer is coverage of that specific path, plus the fact that a test-scoped
dependency can pull a different version onto the test classpath than the runtime
one.

---

### Q2 — "How does Maven resolve two different versions of the same dependency? How is that different from npm?"

**Mid-level answer:** "Maven picks the nearest one. npm installs both."

**Senior answer:** "Structurally different, and it is the thing that surprises
people moving from Node. npm can nest, so `node_modules/a/node_modules/x@1` and
`node_modules/b/node_modules/x@2` coexist and each library gets what it asked
for — module identity in Node is a resolved file path. On the JVM, class identity
is (classloader, fully qualified name), so with one application classloader
`com.example.X` can only mean one class. Maven therefore flattens: exactly one
version per artifact, chosen by nearest depth from the root POM, ties broken by
declaration order.

The consequence is that a version conflict in npm is mostly a disk-space
problem, and in Maven it is a silent correctness problem — mediation reports
nothing at default verbosity. `<dependencyManagement>` sits above mediation in
the precedence chain, which is why BOMs exist and why importing
`spring-boot-dependencies` is not optional on a Spring project.

I would also be honest that Maven has no lockfile. A BOM manages versions you
named; it does not freeze the whole graph the way `package-lock.json` does.
Gradle's dependency locking is genuinely closer. What I do in Maven instead is
BOM coverage plus `requireUpperBoundDeps` plus banning dynamic versions plus a
repository manager — a weaker guarantee assembled from several pieces."

**What separates them:** explaining *why* the JVM cannot nest (classloader
identity), and volunteering the lockfile gap honestly instead of claiming parity.

**Follow-up:** "Could you make two versions coexist on the JVM?" Yes — separate
classloaders, which is what OSGi and application servers do, and which is why
those systems have their own class-loading pain. Also shading with relocation.
Both have real costs.

---

### Q3 — "What is a BOM and why not just pin versions?"

**Mid-level answer:** "A BOM is a POM with a list of versions so you do not have
to specify them yourself."

**Senior answer:** "A BOM is a `pom`-packaged artifact containing only
`dependencyManagement`, imported with `<type>pom</type><scope>import</scope>`.
Importing it says 'for any of these artifacts, at any depth, use this version' —
and because `dependencyManagement` outranks mediation, it settles conflicts
before they happen.

Individual pinning fails for a specific reason: families like Spring, Jackson and
Hibernate are released as tested *sets*, and their modules call each other's
internals. Pin `spring-web` in one module and `spring-tx` in another and they
drift apart on different weeks, and the failure is a `NoSuchMethodError` inside
framework code during context refresh — a stack trace with none of your classes
in it. A BOM moves the whole set together from one property.

Two details I would want a reviewer to know: within `dependencyManagement`, first
declaration wins, so BOM import order matters — and a directly declared entry
beats any import, which is the reliable way to override a BOM entry. And
overriding a Boot-managed version takes you off Boot's tested combination, so it
should be a documented, dated exception, usually because of a CVE."

**What separates them:** the release-as-a-set argument, the import-order rule,
and treating an override as an exception with an expiry rather than a routine
edit.

**Follow-up:** "You need to fix a CVE in a Boot-managed artifact. Exactly what do
you change?" That is Topic 34: a direct entry in your own `dependencyManagement`
above the import, with a comment naming the CVE and a removal condition.

---

### Q4 — "You reorder two `<dependency>` blocks and a test starts failing. Explain."

**Mid-level answer:** "That should not matter. Maybe a flaky test."

**Senior answer:** "It absolutely can matter. Nearest-wins breaks ties by
declaration order, so if two dependencies bring the same transitive at the same
depth, the one listed first decides the version. Moving a block moves the winner.

That is genuinely a design wart — dependency order in a POM should be
semantically neutral and it is not. The practical response is to make it neutral
by construction: pin anything that appears at multiple versions in
`dependencyManagement`, and add `requireUpperBoundDeps` so a future reorder that
downgrades something fails the build instead of changing behaviour. Then
alphabetising the POM really is just style.

I would also check `mvn dependency:tree -Dverbose` before and after the reorder
and diff them, because that turns 'a test got flaky' into a specific artifact and
a specific version delta."

**What separates them:** not dismissing it as flakiness, and turning "order
matters" into "make order not matter" as an engineering action.

**Follow-up:** "How would you catch this in code review?" A CI job that diffs
`mvn dependency:list` between the base branch and the PR, and comments the delta.
Cheap, and it makes an invisible change visible.

---

### Q5 — "Maven has no lockfile. Is that a problem?"

**Mid-level answer:** "Not really, you pin versions in the POM."

**Senior answer:** "It is a real gap and I would not pretend otherwise. Pinning
covers artifacts I named. It does not cover a transitive-of-a-transitive that is
outside every BOM I import, and it records no integrity hashes, so I cannot prove
the jar I built with today is byte-identical to the one I built with last month.

What I actually do is assemble an approximation: import BOMs so the framework's
several hundred artifacts are managed; `banDynamicVersions` so no ranges or
SNAPSHOTs reach a release; `requireUpperBoundDeps` so a downgrade fails the
build; and a repository manager that mirrors and caches everything so the upstream
disappearing does not break me. Some teams also commit a
`mvn dependency:list` snapshot and diff it in CI, which is a poor man's lockfile
but does make graph changes visible in a pull request.

If lockfile guarantees were a hard requirement — a regulated environment, or a
team that has been burned — that is one of the genuinely good arguments for
Gradle, which has real dependency locking with a `gradle.lockfile` per
configuration and fails on any deviation. I would rather make that trade
explicitly than claim Maven has parity."

**What separates them:** naming precisely what is not covered, describing a
composite mitigation, and being willing to say a competing tool is better at one
specific thing.

**Follow-up:** "How does this connect to reproducible builds and SBOMs?"
Straight into Topic 34 — an SBOM is only meaningful if the graph it describes is
the graph that shipped.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Maven picks the **nearest** version; Gradle picks the **highest**. Argue that
   each is the more defensible default, then say which you would choose if you
   were designing a build tool in 2026 and why.

2. Mediation reports nothing at default verbosity. Give the strongest argument
   that this is correct behaviour — what would break if every conflict printed a
   warning? Then say whether you are convinced.

3. A `NoSuchMethodError` can only happen because the JVM links lazily. What would
   Java gain, and what would it lose, if all symbolic references were resolved at
   class-load time instead?

4. `<dependencyManagement>` beats mediation, and a direct `<version>` beats
   `<dependencyManagement>`. Explain why that ordering — and not the reverse — is
   the right design.

5. Two different artifacts containing the same fully qualified class is worse
   than two versions of one artifact. Why? What does the JVM actually do, and why
   can Maven not detect it?

6. Your team imports the Boot BOM and overrides two entries for CVE fixes. Six
   months later you upgrade Boot by one minor version. What is the specific
   failure mode of leaving those overrides in place, and what mechanism would
   force you to revisit them?

7. npm's nesting means a library's dependency choices are private to that
   library. Maven's flat classpath makes them everybody's business. Which
   property produces better software over ten years, and does your answer change
   if the codebase has 5 dependencies versus 500?

---

## Quick reference card

### The precedence chain

```
1. <version> written directly on the <dependency> in THIS pom     <- strongest
2. <dependencyManagement> in THIS pom (or inherited from parent)
3. An imported BOM (first import wins between BOMs)
4. Mediation: nearest depth wins; ties break by declaration order  <- weakest
```

### The diagnostic commands

```bash
mvn dependency:tree                                    # the graph
mvn dependency:tree -Dverbose                          # + what LOST
mvn dependency:tree -Dverbose -Dincludes=g:a           # filtered, the one you'll use
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:*   # wildcards work
mvn dependency:list -DincludeScope=runtime | sort      # the flat result
mvn dependency:analyze                                 # declared vs used
mvn dependency:resolve                                 # force-download everything
mvn help:effective-pom                                 # where a version came from
mvn versions:display-dependency-updates                # what could be bumped
mvn -X ...                                             # full debug, when desperate
```

### The error decoder

| Error | First command to run |
|---|---|
| `NoSuchMethodError` | `dependency:tree -Dverbose -Dincludes=<the class's artifact>` |
| `NoClassDefFoundError` | check for `<exclusions>` and `provided` scope first |
| `ClassNotFoundException` | check for reflection / service loading, then classpath |
| `AbstractMethodError` | interface and impl from different versions — same command |
| `IncompatibleClassChangeError` | a major version won mediation — same command |

### Guard rails to put in every parent POM

```xml
<requireUpperBoundDeps/>     <!-- fails if a HIGHER version lost. The important one. -->
<banDynamicVersions/>        <!-- no ranges, no SNAPSHOTs in a release -->
<dependencyConvergence/>     <!-- stricter; expect noise on Spring projects -->
```

### Gotchas checklist

- [ ] Exactly one version of an artifact reaches the classpath. Always.
- [ ] Mediation is silent. `-Dverbose` is the only way to see it.
- [ ] `omitted for conflict with <LOWER version>` is the dangerous direction.
- [ ] `<exclusions>` remove; they do not select. Use `dependencyManagement` for versions.
- [ ] BOM imports: first one wins. A direct entry beats every import.
- [ ] A `<version>` in a module's `<dependencies>` is a smell — justify each one.
- [ ] Declare what you import. `dependency:analyze` finds the ones you did not.
- [ ] Maven has no lockfile. Say so honestly; do not claim BOMs are one.
- [ ] A clean compile is evidence about the compile classpath only.

---

## When would I use this at work?

**1. The first ten minutes of a production incident.**
The trace says `NoSuchMethodError` or `AbstractMethodError`. You do not read
application code, you do not ask who deployed what. You run
`mvn dependency:tree -Dverbose -Dincludes=` on the artifact from the trace, find
the `omitted for conflict` line, and you have the cause. Engineers who do not
know this spend the first hour reading the wrong file.

**2. Reviewing a pull request that adds one dependency.**
A colleague adds a single library for a small feature. You ask for the
`mvn dependency:tree` diff. It turns out that one line brings forty transitive
artifacts and downgrades two things the framework depends on. That review comment
costs you two minutes and saves an incident. This is the highest-value habit in
the whole phase.

**3. A framework upgrade — Boot 3.5 to 4.1, or a Java version bump.**
Boot 4's modularisation means artifacts moved. Half of an upgrade is not code, it
is discovering which of your explicit versions and overrides are now managing
artifacts that no longer exist, and which transitives shifted depth. You do that
with `dependency:tree` diffs before and after, and with `requireUpperBoundDeps`
telling you which paths regressed. Without those tools an upgrade is a series of
runtime surprises; with them it is a checklist.

---

## Connected topics

**Prerequisites:**
- **31 — Maven fundamentals.** Scopes, `<dependencyManagement>` vs
  `<dependencies>`, the reactor, and the `~/.m2` local repository that this
  topic's drill depends on.
- **06 — Type erasure** and **19 — Serialization**, lightly: both explain why a
  library version change can alter *behaviour* without altering any signature you
  can see.

**This unlocks:**
- **33 — Gradle.** Different mediation (highest wins), real dependency locking,
  and `platform()` for consuming exactly the BOMs you learned here.
- **34 — Supply chain.** A CVE triage begins with "which version is actually on
  the classpath", which is the command you just learned. An SBOM is a serialised
  form of the resolved graph.
- **35 — `ApplicationContext`.** The bean graph is built from classes that got
  onto the classpath through this process.
- **42 — Auto-configuration mechanics.** Boot scans every jar on the classpath
  for `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
  Which auto-configurations are *candidates* is therefore a direct function of
  the resolved dependency graph — mediation choosing a different artifact version
  can add or remove auto-configuration candidates, which is one of the strangest
  Spring bugs to diagnose without this topic.
- **61 — Testcontainers.** Test-scoped dependencies bring their own transitive
  graph. A test-only Jackson or Netty version conflicting with the runtime one is
  a common and confusing failure.
- **83 — GraalVM native image.** Closed-world analysis operates on the resolved
  classpath. Two jars with duplicate classes, or a mediated-away version, produce
  build-time analysis failures instead of runtime ones — which is arguably
  better, but only if you can read a dependency tree.
- **122 — Layered Docker jars.** The dependency layer is exactly the set of jars
  mediation selected. A single transitive change invalidates that Docker layer
  for every consumer.
- **127 — Migration planning.** Boot 2→3→4 is largely a dependency-graph exercise
  with a `javax`→`jakarta` rename on top.

---

*Java baseline 21, running on JDK 25. Target Spring Boot 4.1 / Framework 7.0 /
Jakarta EE 11. The mediation algorithm described here has been stable for the
entire life of Maven 3 and is not going to change under you. Every version number
in this document that refers to a third-party library is either a placeholder in
a drill you build yourself, or a `SET-ME` you resolve with a command.*
