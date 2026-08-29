# 31 — Maven: POM, Scopes, Lifecycle Phases, Multi-Module

## Phase: 3 — Build & Supply Chain
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: This is where the `orderflow` build is born. Topic 35 starts writing the service; this topic builds the four-module skeleton (`orderflow-domain`, `orderflow-persistence`, `orderflow-api`, `orderflow-worker`) that every topic from 35 onward compiles into.

---

## ELI5 anchor

Think of a factory assembly line with numbered stations.

Station 1 checks the paperwork. Station 5 cuts the metal. Station 9 bolts it
together. Station 12 tests it. Station 15 boxes it.

**You cannot add a station.** You cannot renumber them. You cannot say "run
station 12 before station 5".

What you *can* do is bolt a tool onto a station. "At station 9, also run the
label printer." "At station 12, also run the leak test."

And here is the rule that makes the whole thing work: **if you ask for station
12, stations 1 through 12 all run first, in order.** You never ask for a station
in the middle and get only that one.

npm is not this. In npm you write `"scripts": { "whatever": "..." }` and invent
your own stations with your own names. Maven's stations were decided in 2004 and
are the same in every Java project on earth. That rigidity is the point: any Java
engineer can walk up to any Maven project and already know what `mvn verify`
does.

---

## The bridge from what you know

### `package.json` scripts → Maven lifecycle phases: **PARTIAL**

This is the analogue you will reach for first, and it is half right.

```json
{
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "lint": "eslint .",
    "ship-it": "npm run lint && npm run build && npm run test && docker build ."
  }
}
```

You invented four names. `ship-it` exists because you typed it. There is no
ordering model — `&&` is the ordering model.

Maven has no equivalent of "invent a script name". It has a **fixed, ordered list
of phases** that exists before you write a single line of build config, and you
attach work to those phases.

| You do this in npm | You do this in Maven | Verdict |
|---|---|---|
| `npm run build` | `mvn compile` | **PARTIAL** — the name is fixed and so is everything that runs before it |
| `npm test` | `mvn test` | **PARTIAL** — `mvn test` also compiles first, always |
| Invent `npm run ship-it` | You cannot invent a phase | **NO ANALOGUE** |
| `pre`/`post` script hooks | Bind a plugin goal to an earlier/later phase | **PARTIAL** |
| Scripts run in whatever order you chained them | Phases run in a spec-defined order, always | **NO ANALOGUE** |

### `node_modules` → `~/.m2/repository`: **PARTIAL, and the difference matters**

npm downloads dependencies **into your project**. Every project gets its own
copy. That is why `node_modules` is enormous and why deleting it is a routine
act.

Maven downloads dependencies into **one shared cache for your whole machine**:
`~/.m2/repository`. Every project on your laptop reads from the same cache. Your
project directory contains no dependencies at all.

Consequences you will feel on day one:

- There is nothing to `rm -rf` inside your project to "fix" a dependency problem.
  The equivalent nuclear option is deleting a subtree of `~/.m2/repository`, and
  it affects every project you own.
- Two projects on your machine that both use library X share the exact same jar
  file on disk.
- A build can succeed on your machine and fail in CI because *your* cache has an
  artifact that was never published anywhere. This is the single most common
  "works on my machine" in Java, and Trap 1 below is exactly it.

### `devDependencies` → `<scope>test</scope>`: **PARTIAL**

`devDependencies` is a two-state flag: shipped, or not shipped.

Maven scopes are a **classpath-membership decision across three or four different
classpaths**, and one of them (`provided`) has no npm equivalent at all. Full
table in the Syntax breakdown.

### What has NO analogue at all

| npm reality | Maven reality |
|---|---|
| Two versions of the same package can coexist in one install | Exactly one version of an artifact on a flat classpath |
| `package-lock.json` locks the whole graph | Maven has no lockfile by default |
| Scripts are arbitrary shell | Phases are fixed; plugins are Java, configured in XML |
| `node_modules` is per-project | `~/.m2` is per-machine |

The first row is Topic 32 and it is the most important thing in this whole phase.
Everything else here is learnable in an afternoon. That one will bite you in
production.

---

## What is this?

**Maven is a build tool built on three ideas:**

1. **Convention over configuration.** Source goes in `src/main/java`. Tests go in
   `src/test/java`. Resources go in `src/main/resources`. Output goes in
   `target/`. You do not configure any of this. If you fight it, you lose.

2. **A declarative project model.** The `pom.xml` ("Project Object Model")
   describes *what the project is* — its identity, its dependencies, its
   packaging — not *how to build it*. There is no build script.

3. **A fixed lifecycle.** Building is a sequence of named phases. Plugins bind
   goals to phases. Asking for a phase runs every phase before it.

**A POM** is an XML file. **Coordinates** identify every artifact in the Java
world uniquely:

```
groupId : artifactId : version
com.orderflow : orderflow-domain : 1.0.0-SNAPSHOT
```

That triple is the equivalent of a package name in npm, except the `groupId` is
usually a reversed domain you control, which is why Java has far less
typosquatting than npm (Topic 34 returns to this).

---

## Why does it matter?

Four reasons, in order of how soon they will hurt you:

1. **You cannot run a Spring app you cannot build.** Phases 4 and 5 assume you
   can add a dependency, understand why it landed on your classpath, and produce
   a runnable jar. Every one of those is this topic.

2. **`mvn clean install` in CI hides broken releases.** It is the command
   everyone types and it is wrong for CI. Trap 1.

3. **Scope errors do not fail the build.** A `provided` dependency compiles
   perfectly and then throws `NoClassDefFoundError` at deploy time, in an
   environment you cannot attach a debugger to. Trap 2.

4. **Multi-module is how real Java services are laid out**, and the reactor's
   build order is a dependency graph, not the order you wrote in `<modules>`.
   People who do not know this write POMs that only build by luck.

---

## Syntax breakdown

### The minimum viable POM

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <modelVersion>4.0.0</modelVersion>

  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-domain</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>jar</packaging>

</project>
```

| Element | What it means |
|---|---|
| `<modelVersion>4.0.0</modelVersion>` | The POM *schema* version. It has been `4.0.0` for twenty years. It is not your project's version. Just type it. |
| `<groupId>` | Your namespace. Reversed domain by convention. Maps to a directory path in the repository: `com/orderflow/`. |
| `<artifactId>` | The module name. Becomes the jar filename. |
| `<version>` | `1.0.0-SNAPSHOT` — the `-SNAPSHOT` suffix means "in development, may change". Maven re-checks remote repositories for updated SNAPSHOTs; release versions are cached forever and are supposed to be immutable. |
| `<packaging>` | `jar` (default), `pom` (a parent or a BOM — produces no jar), `war`, `maven-plugin`. **Packaging chooses which plugin goals are bound to which phases.** |

### The three lifecycles

Maven has three independent lifecycles. You mostly use two.

| Lifecycle | Phases | What it does |
|---|---|---|
| `clean` | `pre-clean`, `clean`, `post-clean` | Deletes `target/` |
| `default` | 23 phases (below) | Builds, tests, packages, installs, deploys |
| `site` | `pre-site`, `site`, `post-site`, `site-deploy` | Generates project documentation. Rarely used now. |

`mvn clean package` is you invoking **two lifecycles** in one command: the whole
`clean` lifecycle, then the `default` lifecycle up to and including `package`.

### The default lifecycle, in order

These are the 23 phases. The ones in bold are the ones you will actually type.

```
validate
initialize
generate-sources
process-sources
generate-resources
process-resources
compile                  <-- **
process-classes
generate-test-sources
process-test-sources
generate-test-resources
process-test-resources
test-compile
process-test-classes
test                     <-- **
prepare-package
package                  <-- **
pre-integration-test
integration-test
post-integration-test
verify                   <-- **
install                  <-- **
deploy                   <-- **
```

**The rule again, because it is the whole model:** `mvn verify` runs *all
nineteen phases before `verify`*, then `verify`. `mvn install` runs everything up
to and including `install`. There is no "just this one phase" command.

### Phases do nothing. Goals do the work.

A phase is an empty slot with a name and a position. What actually executes is a
**plugin goal** bound to that phase.

The syntax for a goal is `plugin:goal`, e.g. `compiler:compile`,
`surefire:test`, `jar:jar`.

For `<packaging>jar</packaging>`, Maven binds these by default:

| Phase | Goal that runs |
|---|---|
| `process-resources` | `maven-resources-plugin:resources` |
| `compile` | `maven-compiler-plugin:compile` |
| `process-test-resources` | `maven-resources-plugin:testResources` |
| `test-compile` | `maven-compiler-plugin:testCompile` |
| `test` | `maven-surefire-plugin:test` |
| `package` | `maven-jar-plugin:jar` |
| `install` | `maven-install-plugin:install` |
| `deploy` | `maven-deploy-plugin:deploy` |

So the answer to "what does `mvn package` actually run, in order?" is:
resources → compile → testResources → testCompile → surefire:test → jar:jar,
with the empty phases passing through silently. Memorise that; it is an interview
question and it is Q1 below.

**You can also run a goal directly, skipping the lifecycle entirely:**

```bash
mvn dependency:tree            # runs one goal, no phases at all
mvn help:effective-pom         # same
```

That is why `mvn dependency:tree` is instant and `mvn test` is not.

### Binding your own plugin to a phase

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <executions>
        <execution>
          <id>integration-tests</id>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Read it as: "when the build reaches the `integration-test` phase, run failsafe's
`integration-test` goal; when it reaches `verify`, run failsafe's `verify` goal."
Failsafe declares those default phase bindings itself, which is why no `<phase>`
element is needed here. When a goal has no default binding, you add
`<phase>package</phase>` inside the `<execution>`.

This is the closest thing Maven has to `npm run something-custom`, and note how
different it feels: you are not naming a command, you are choosing a *slot*.

### Dependency scopes — the real table

```xml
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```

There are six scopes.

| Scope | On compile classpath? | On test classpath? | On runtime classpath? | Packaged into the app? |
|---|---|---|---|---|
| `compile` (default) | yes | yes | yes | yes |
| `provided` | yes | yes | **no** | **no** |
| `runtime` | **no** | yes | yes | yes |
| `test` | **no** | yes | **no** | **no** |
| `system` | yes | yes | no | no |
| `import` | — | — | — | — |

Notes that matter:

- **`compile`** is the default and what you get if you write no `<scope>`.
- **`provided`** means "something else will supply this at runtime — a servlet
  container, an application server, an agent, the platform." It compiles, it
  tests, and then it is *absent* when you deploy. **There is no npm equivalent.
  None.** This is Trap 2, and it is the single most common deploy-time surprise
  for people arriving from Node.
- **`runtime`** means "I never `import` this type in my code, but it must be
  present when the app runs." JDBC drivers are the canonical case: your code
  talks to `java.sql.*`, and Postgres's driver is discovered by service loading.
  Marking it `runtime` means a developer *cannot* accidentally import a
  Postgres-specific class.
- **`test`** is your `devDependencies`.
- **`system`** is deprecated. It points at an absolute path on disk. If you see
  it in a codebase, it is technical debt. Do not add new ones.
- **`import`** is not a classpath scope at all. It is only legal inside
  `<dependencyManagement>` on a `<type>pom</type>` dependency, and it means "pull
  this BOM's version list into mine". That is Topic 32.

### Scope transitivity — the second table nobody memorises but should

When your dependency has its own dependencies, the scope you get is a function of
both. Read as: *your dependency's scope* (row) combined with *its dependency's
scope* (column).

| ↓ yours \ theirs → | `compile` | `provided` | `runtime` | `test` |
|---|---|---|---|---|
| `compile` | `compile` | — | `runtime` | — |
| `provided` | `provided` | — | `provided` | — |
| `runtime` | `runtime` | — | `runtime` | — |
| `test` | `test` | — | `test` | — |

`—` means **omitted entirely**. Two things fall out of this:

1. `provided` and `test` dependencies are **never transitive**. If a library you
   depend on marks something `provided`, you do not get it, and you must declare
   it yourself. This is by design.
2. A `compile` dependency of your `test` dependency arrives as `test`. So a
   test-only library cannot leak into your production jar through the back door.

### `<dependencyManagement>` vs `<dependencies>`

This distinction causes more bad POMs than anything else.

```xml
<!-- In the PARENT pom -->

<dependencyManagement>          <!-- "IF a module uses this, use THIS version" -->
  <dependencies>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <version>${postgresql.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>                  <!-- "EVERY module gets this, no opt-out" -->
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
  </dependency>
</dependencies>
```

| | `<dependencyManagement>` | `<dependencies>` |
|---|---|---|
| Effect on children | declares a version, adds nothing | **adds the dependency to every child** |
| Child must re-declare? | yes, without a `<version>` | no |
| Right place for | versions, exclusions, scopes | the two or three things genuinely universal (a logging facade, maybe an annotations jar) |

**Rule of thumb: almost everything belongs in `<dependencyManagement>`.** Putting
a web framework in the parent's `<dependencies>` means your pure-domain module
now has a servlet API on its classpath, and in a Spring Boot project that can
trigger auto-configuration you never asked for. Trap 4.

### `<parent>` and `<modules>`

```xml
<!-- child module -->
<parent>
  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <relativePath>../pom.xml</relativePath>
</parent>

<artifactId>orderflow-persistence</artifactId>
<!-- no groupId, no version: both inherited from the parent -->
```

```xml
<!-- parent pom -->
<packaging>pom</packaging>
<modules>
  <module>orderflow-domain</module>
  <module>orderflow-persistence</module>
  <module>orderflow-api</module>
  <module>orderflow-worker</module>
</modules>
```

Two separate mechanisms that people conflate:

- **`<parent>` is inheritance.** Properties, dependencyManagement,
  pluginManagement, repositories flow down. A parent does not need to list you as
  a module.
- **`<modules>` is aggregation.** It tells the *reactor* — Maven's multi-module
  build engine — which projects to build together. An aggregator does not need to
  be your parent.

In practice one POM is usually both. But the important consequence is:

> **The reactor does NOT build modules in the order listed in `<modules>`.** It
> topologically sorts them by their inter-module dependencies. If
> `orderflow-api` depends on `orderflow-domain`, domain builds first regardless
> of listing order. If two modules depend on each other, you get
> `The projects in the reactor contain a cyclic reference` and the build refuses
> to start.

### The wrapper

```
.mvn/wrapper/maven-wrapper.properties
mvnw
mvnw.cmd
```

Commit these. `./mvnw` downloads and uses a pinned Maven version, so everyone —
including CI — runs the same build tool. It is the same instinct as pinning your
Node version in `.nvmrc`, except here the tool version genuinely changes build
behaviour.

> **Maven 4:** a Maven 4 line exists and changes some multi-module and POM-model
> details (a trimmed "consumer POM", better reactor handling). I am not going to
> state its release status or version number from memory — run `mvn --version`
> and check the Apache Maven site. Everything in this document is Maven 3
> semantics, which Maven 4 preserves.

---

## Example 1 — minimal

One module. One dependency. Java 21 source level.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-scratch</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>SET-ME</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

Three things to notice:

1. **`maven.compiler.release` = 21, not `source`/`target`.** `release` also tells
   the compiler which *API* is available, so you cannot accidentally call a
   JDK 25 method and ship a jar that explodes on a JDK 21 runtime. `source` and
   `target` do not do that. Always use `release`.
2. **`<scope>test</scope>` on JUnit.** It is your `devDependencies` line.
3. **`SET-ME` is deliberate.** I am not writing a JUnit version from memory. Find
   the current one with `mvn versions:display-dependency-updates`, or generate a
   project from `start.spring.io` and read it out of the generated POM. Guessing
   version numbers is how you spend an afternoon on a dependency that does not
   exist.

Run it:

```bash
mvn -q clean verify
```

---

## Example 2 — production scenario: the `orderflow` multi-module build

This is the skeleton Phase 4 onward compiles into. Four modules, one parent, the
Spring Boot BOM imported rather than inherited.

```
orderflow/
├── pom.xml                     <- parent + aggregator, packaging=pom
├── orderflow-domain/           <- pure Java. Records, sealed types, no Spring.
│   └── pom.xml
├── orderflow-persistence/      <- JPA entities + repositories. Depends on domain.
│   └── pom.xml
├── orderflow-api/              <- Spring Boot web app. The runnable service.
│   └── pom.xml
└── orderflow-worker/           <- Spring Boot consumer/batch. Also runnable.
    └── pom.xml
```

### Parent POM

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>
  <name>orderflow</name>

  <modules>
    <module>orderflow-domain</module>
    <module>orderflow-persistence</module>
    <module>orderflow-api</module>
    <module>orderflow-worker</module>
  </modules>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <!-- Set to the current Spring Boot 4.1.x release.
         Find it with: mvn versions:display-property-updates
         or from start.spring.io. I am not guessing a patch number. -->
    <spring-boot.version>SET-ME</spring-boot.version>
  </properties>

  <dependencyManagement>
    <dependencies>

      <!-- The Spring Boot BOM. This manages the versions of ~600 artifacts
           that Boot has tested together. Topic 32 explains why this is
           not optional. -->
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>

      <!-- Our own modules, so children can depend on each other
           without repeating ${project.version} everywhere. -->
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

    </dependencies>
  </dependencyManagement>

  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-maven-plugin</artifactId>
          <version>${spring-boot.version}</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```

**Why import the BOM instead of using `spring-boot-starter-parent` as our
`<parent>`?** Because we want our own parent, for our own properties and plugin
config, and a POM can only have one parent. Importing the BOM gives you Boot's
managed versions without giving up the parent slot. This is the standard
multi-module layout and you should default to it.

### `orderflow-domain/pom.xml`

```xml
<project ...>
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>com.orderflow</groupId>
    <artifactId>orderflow-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>orderflow-domain</artifactId>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
      <!-- no <version>: managed by the Boot BOM -->
    </dependency>
  </dependencies>
</project>
```

**This module has zero Spring dependencies and that is deliberate.** Your
`Order`, `OrderLine`, `Money`, `PaymentResult` types (Topics 27 and 28) belong
here. If you can compile the domain without Spring on the classpath, your domain
does not secretly depend on the framework. That is enforceable architecture, and
it costs you one module.

### `orderflow-persistence/pom.xml`

```xml
  <artifactId>orderflow-persistence</artifactId>

  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-domain</artifactId>
      <!-- version from the parent's dependencyManagement -->
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- runtime: our code never imports a Postgres class.
         Marking it runtime makes that a compile-time guarantee. -->
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- Topic 61 will replace H2-shaped testing with real Postgres. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
```

### `orderflow-api/pom.xml`

```xml
  <artifactId>orderflow-api</artifactId>

  <dependencies>
    <dependency>
      <groupId>com.orderflow</groupId>
      <artifactId>orderflow-persistence</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- ONLY on the runnable modules. This plugin's `repackage` goal
           rewrites the jar into an executable fat jar. Putting it on a
           library module produces a jar nothing else can depend on. -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <executions>
          <execution>
            <goals><goal>repackage</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

`orderflow-worker` looks the same minus `starter-web`, plus whatever messaging
starter Topic 113 introduces.

### Reactor order

You listed domain, persistence, api, worker. Maven does not care about that list.
It reads the dependencies and computes:

```
orderflow-parent
  → orderflow-domain
      → orderflow-persistence
          → orderflow-api
          → orderflow-worker      (api and worker are independent of each other)
```

`api` and `worker` have no dependency between them, so with `mvn -T 1C verify`
Maven can build them **in parallel**. That is a free speedup you get from a
correct module graph and cannot get from a wrong one.

> **[BOOT 3.x DELTA]**
> Boot 4 split the codebase into many smaller, more focused jars. Starters still
> exist and still work the same way, but **the artifact that carries a given
> feature may have moved** — the master plan's example is MongoDB health
> indicators relocating from `spring-boot-data-mongodb` to `spring-boot-mongodb`,
> with package renames. Practical consequence for this topic: **do not copy a
> dependency list out of a Boot 3.x project and expect it to resolve.** Generate
> a fresh POM from `start.spring.io` for your target Boot version, then diff.
> Separately: Boot 3.5 left OSS support in June 2026, so a 3.5 project receives
> no free CVE patches — that is Topic 34's problem, not this one, but it is why
> the target here is 4.1.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `mvn clean install` in CI

**Wrong:**
```yaml
# .github/workflows/ci.yml
- run: mvn clean install
```

**Exact symptom:** CI is green on every commit. Then a *different* team, or a
fresh CI runner with an empty cache, runs a build that depends on your artifact
and gets:

```
[ERROR] Failed to execute goal on project their-service:
Could not resolve dependencies for project com.them:their-service:jar:2.1.0
Could not find artifact com.orderflow:orderflow-domain:jar:1.0.0-SNAPSHOT
```

Or worse — no error at all, and they silently build against a **stale** version
of your module that has been sitting in that machine's `~/.m2` for six weeks.

**Root cause:** `install` copies your built artifacts into the **local
repository** on that machine. From that moment, every subsequent resolution on
that machine finds them there. So the build proves your artifacts resolve *from
a cache you just populated*. It proves nothing about whether they are published
anywhere, or whether the versions you declared actually exist remotely.

The multi-module version of this is nastier: a module that is missing from
`<modules>` still builds, because the reactor cannot find it but `~/.m2` can.

**Fix:**
```bash
# CI: build and test, do NOT write to the local repo
mvn -B -ntp clean verify

# Release job only, separately, on a tag:
mvn -B -ntp deploy
```

`verify` runs everything `install` does *except* the local-repo write —
compilation, unit tests, packaging, integration tests, any checks bound to
`verify`. It is strictly the right CI command.

Extra credit: run CI with `-Dmaven.repo.local=.m2-ci` on a clean checkout
occasionally, so a genuinely empty cache is exercised.

---

### Trap 2 — `provided` scope, which compiles perfectly

**Wrong:**
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
  <scope>provided</scope>
</dependency>
```

Someone added `provided` because they saw it elsewhere, or because a build tool
error message mentioned it, or because they were "trimming the jar size".

**Exact symptom:** `mvn clean verify` is green. All tests pass — `provided` **is**
on the test classpath, which is exactly why tests do not catch this. You deploy,
and the first request that touches a validated DTO produces:

```
java.lang.NoClassDefFoundError: jakarta/validation/Valid
	at com.orderflow.api.OrderController.<clinit>(OrderController.java)
Caused by: java.lang.ClassNotFoundException: jakarta.validation.Valid
```

The `Caused by: ClassNotFoundException` under a `NoClassDefFoundError` is the
signature. Learn to read it as "this class was there at compile time and is not
there now".

**Root cause:** `provided` is a promise that *the runtime environment supplies
this jar*. That promise is true for a servlet API inside a real application
server, and for a JVM agent. It is false for a Spring Boot fat jar, which is its
own runtime environment and supplies only what was packaged.

**Fix:** delete `<scope>provided</scope>`. For a Boot fat jar the correct scope
for almost everything is the default, `compile`.

**When `provided` IS right:** you are building a `war` for an external Tomcat
(then `spring-boot-starter-tomcat` is `provided`); you are writing a library and
you want the consumer to choose the implementation; you depend on an API that a
JVM agent injects.

---

### Trap 3 — `-Dmaven.test.skip=true`

**Wrong:**
```bash
mvn clean package -Dmaven.test.skip=true
```

Someone put this in a Dockerfile to make the image build faster.

**Exact symptom:** for weeks, nothing. Then someone runs the real test suite and
gets *dozens* of compilation errors in `src/test/java` — tests referencing
methods that were renamed months ago. The tests have not compiled since the flag
was added, and nobody noticed because they never ran.

**Root cause:** two different flags that look the same.

| Flag | Compiles tests? | Runs tests? |
|---|---|---|
| `-DskipTests` | **yes** | no |
| `-Dmaven.test.skip=true` | **no** | no |

`maven.test.skip` skips `test-compile` as well, so test code silently rots.

**Fix:** use `-DskipTests` when you need to skip execution. Better: do not skip
at all in the pipeline that produces your release artifact — build once with
tests, then reuse the artifact. Rebuilding without tests to make a Docker image
means the image contains a jar nothing verified.

---

### Trap 4 — dependencies in the parent's `<dependencies>`

**Wrong:**
```xml
<!-- parent pom -->
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

"Every module needs Spring anyway."

**Exact symptom:** several, and they arrive separately:

- `orderflow-domain`, which was supposed to be pure Java, now has Tomcat,
  Jackson and the whole web stack on its classpath. `mvn dependency:tree` on the
  domain module shows a hundred artifacts where you expected three.
- Someone in the domain module writes `@RestController` because the annotation
  is available, and your architectural boundary is gone with no build failure.
- In a Spring Boot app, a starter's mere presence on the classpath triggers
  auto-configuration (Topic 42). A test module that accidentally inherits
  `starter-web` starts a web server during unit tests and your test suite gets
  slower for no visible reason.

**Root cause:** `<dependencies>` in a parent is **inherited unconditionally by
every child**. There is no opt-out. You cannot "un-inherit" it.

**Fix:** move it to `<dependencyManagement>` and declare it, without a version,
in the modules that genuinely need it. Reserve the parent's `<dependencies>` for
things that truly are universal — at most a logging facade or an annotations
jar, and honestly often nothing at all.

---

### Trap 5 — assuming `<modules>` controls build order

**Wrong:** a reviewer says "move `orderflow-domain` to the top of `<modules>` so
it builds first."

**Exact symptom (variant A):** you reorder the list, nothing changes, and the
reviewer concludes Maven is broken.

**Exact symptom (variant B):** you add a dependency from `orderflow-domain` back
to `orderflow-persistence` (perhaps a test utility) and the build stops before
compiling anything:

```
[ERROR] The projects in the reactor contain a cyclic reference:
Edge between 'Vertex{label='com.orderflow:orderflow-domain:1.0.0-SNAPSHOT'}'
and 'Vertex{label='com.orderflow:orderflow-persistence:1.0.0-SNAPSHOT'}'
introduces to cycle
```

**Root cause:** `<modules>` is a *set*, not a sequence. The reactor builds a
directed graph from inter-module dependencies and topologically sorts it. Listing
order is only a tiebreaker between modules with no relationship.

**Fix:** express the order you want as a *dependency*, because that is the only
thing the reactor reads. And treat a reactor cycle as what it is: a design
problem in your module boundaries, not a build configuration problem. Extract the
shared piece into a fifth module rather than breaking the cycle with an
exclusion.

---

## Hands-on proof

Everything here is a command **you** run. I have no Maven and no JVM in this
session, so I will not print output and call it a result. What follows is: the
exact command, what to look for, and how to read each outcome.

### Setup

```bash
mkdir -p ~/java-lab/31 && cd ~/java-lab/31
mvn --version      # note the Maven version AND the "Java version:" line
java --version     # should be 21 or 25
```

**How to read `mvn --version`:** it prints the Maven version, the Maven home, and
crucially the **JDK that Maven itself is running on**. If that JDK is 17 and you
set `maven.compiler.release` to 21, the build fails with
`invalid target release: 21`. That mismatch is the first thing to check on any
"it compiles for me" disagreement.

### Proof 1 — watch the phase/goal sequence

```bash
mvn clean package
```

**What to look for:** the lines that start with `---`. Maven prints one banner
per goal execution, in execution order.

**This is an illustration of the format, not captured output:**

```
[INFO] --- resources:3.x.x:resources (default-resources) @ orderflow-domain ---
[INFO] --- compiler:3.x.x:compile (default-compile) @ orderflow-domain ---
[INFO] --- resources:3.x.x:testResources (default-testResources) @ orderflow-domain ---
[INFO] --- compiler:3.x.x:testCompile (default-testCompile) @ orderflow-domain ---
[INFO] --- surefire:3.x.x:test (default-test) @ orderflow-domain ---
[INFO] --- jar:3.x.x:jar (default-jar) @ orderflow-domain ---
```

Read one banner as: `plugin : version : goal (execution-id) @ module`.

| What you see | What it means |
|---|---|
| Six banners in that order for a `jar` module | The default bindings, exactly as in the table above. This *is* the answer to "what does `mvn package` run". |
| A `surefire:test` banner but `Tests run: 0` | Surefire ran but found no test classes. Check your naming — surefire matches `*Test`, `Test*`, `*Tests`, `*TestCase` by default. |
| No `surefire:test` banner at all | Tests were skipped. Look for `-DskipTests`, `<skipTests>` in the POM, or `maven.test.skip` in `~/.m2/settings.xml`. |
| An `install:install` banner when you typed `package` | You are not running the command you think you are. Check for an alias or a `.mvn/maven.config` file. |
| Banners for a module you did not expect | `<modules>` includes something you forgot about. |

### Proof 2 — ask Maven what a phase is bound to

```bash
mvn help:describe -Dcmd=package
```

**What to look for:** a listing of every phase in the default lifecycle with the
goal(s) bound to it, for *your* project's packaging.

| What you see | What it means |
|---|---|
| Phases listed with `plugin:goal` next to some of them | Correct. The empty phases are real phases with nothing bound. |
| A goal you did not add | Something you inherited — a parent POM, or a plugin that self-binds. Follow it up with `help:effective-pom`. |
| The command errors with "cmd" unknown | Older/newer help-plugin argument; try `mvn help:describe -Dcmd=deploy` or read `mvn help:describe -Dplugin=org.apache.maven.plugins:maven-help-plugin -Ddetail`. |

### Proof 3 — the effective POM

```bash
mvn help:effective-pom -Doutput=effective-pom.xml
less effective-pom.xml
```

**What to look for:** this is the POM Maven actually uses — yours, plus
everything inherited from parents, plus every default. It is usually ten times
longer than your file.

| What you see | What it means |
|---|---|
| A `<version>` on a dependency where you wrote none | The version came from `<dependencyManagement>` or an imported BOM. This is how you find out *where* a version came from. |
| A plugin you never declared, with a version | A default lifecycle plugin. Note the version — if it is not pinned anywhere, your build is not reproducible (Topic 34). |
| Properties resolved to literal values | Confirms property inheritance actually reached this module. |
| A dependency at a scope you did not choose | Scope transitivity (the second table above) applied. |

Run it in a **child module directory**, not the parent, to see what that module
really has.

### Proof 4 — see the actual classpath, per scope

```bash
mvn dependency:build-classpath -Dmdep.outputFile=cp-runtime.txt \
    -DincludeScope=runtime
wc -l cp-runtime.txt
tr ':' '\n' < cp-runtime.txt | wc -l
```

Then the compile classpath:

```bash
mvn dependency:build-classpath -Dmdep.outputFile=cp-compile.txt \
    -DincludeScope=compile
```

**What to look for:** diff the two.

| What you see | What it means |
|---|---|
| The Postgres driver in `runtime` but not `compile` | Your `<scope>runtime</scope>` is working. Your code cannot import a Postgres class even by accident. |
| JUnit in neither | Correct — `test` scope is in neither of these two. |
| A `provided` dependency in `compile` but not `runtime` | The exact hazard in Trap 2, made visible before you deploy. |

### Proof 5 — prove the local repository is real

```bash
mvn -o clean verify
```

`-o` is offline mode: Maven refuses to contact any remote repository.

| What you see | What it means |
|---|---|
| Build succeeds | Every artifact you need is already in `~/.m2/repository`. |
| `The repository system is offline but the artifact ... has not been downloaded` | You have proved that this dependency is not cached locally — and if a *teammate's* build passes offline while yours does not, you have found a cache-only artifact. |

Then look at where things actually live:

```bash
ls ~/.m2/repository/com/orderflow/orderflow-domain/1.0.0-SNAPSHOT/
```

**How to read it:** after `mvn install`, your own jar is sitting in there. That
directory is the reason Trap 1 works. Delete it and re-run a build that depends
on it, and you will see the difference between "resolves" and "resolves from a
cache I populated".

### Proof 6 — declared vs used

```bash
mvn dependency:analyze
```

**What to look for:** two sections.

| Section | What it means | What to do |
|---|---|---|
| `Used undeclared dependencies` | You `import` classes from a jar you never declared — you are getting it transitively. It will vanish the day the middle library drops it. | Declare it explicitly. |
| `Unused declared dependencies` | You declared it and never import from it. | Often correct to remove — but **be careful**: runtime-only things (drivers, logging backends, annotation processors) legitimately show up here. Never bulk-delete this list. |

This goal reads bytecode, so it cannot see reflection, service loading, or Spring
component scanning. Treat it as a strong hint, not a verdict.

---

## Practice exercises

### 1 — Easy: make the lifecycle visible

Create a single-module project by hand (no archetype, no Initializr — type the
POM).

1. Run `mvn clean`, then `mvn compile`, then `mvn test`, then `mvn package`,
   each as a separate command from a clean state, and record the `---` banner
   lines for each.
2. Answer from your own output: how many goals did `mvn test` run that
   `mvn compile` did not?
3. Now bind `maven-enforcer-plugin`'s `enforce` goal to the `validate` phase with
   a `requireMavenVersion` rule. Run `mvn compile` and confirm the enforcer
   banner appears **first**, before the compiler.
4. Break the rule deliberately (require a Maven version you do not have) and
   record the exact failure message and which phase it failed in.

Write down, in one sentence, why the enforcer ran during `mvn compile` when you
bound it to `validate`.

### 2 — Medium: a real module, using Phase 1 and 2 material

Build a module `orderflow-domain` containing code you already know how to write:

- A `Money` type storing minor units as `long` (Topic 01 — never `double`).
- A `record Product(String sku, String name, Money price)` (Topic 27).
- A `sealed interface PaymentResult` with records `Approved`, `Declined`,
  `Failed` (Topic 28).
- A method using a `switch` pattern match over `PaymentResult` (Topic 29).
- A `Comparator<Product>` built with `comparing().thenComparing()` (Topic 14).
- JUnit 5 tests, at `<scope>test</scope>`.

Then:

1. Set `maven.compiler.release` to `17` and run `mvn verify`. Record the exact
   compiler error you get from the sealed-type or pattern-matching code, and
   explain which language feature failed and why.
2. Set it back to `21`. Confirm the build passes.
3. Run `mvn dependency:build-classpath -DincludeScope=runtime` and prove JUnit
   is **not** on it.
4. Unpack the built jar: `unzip -l target/orderflow-domain-1.0.0-SNAPSHOT.jar`.
   Confirm no test classes are inside. Explain in one sentence which mechanism
   guaranteed that — the scope, or the source directory convention, or both.

### 3 — Hard: build the real `orderflow` skeleton

This is the artefact Phase 4 starts from. Do it properly; you will live in it for
months.

**Part A — structure.** Create the parent plus all four modules exactly as in
Example 2. `orderflow-domain` has no Spring dependency at all. Import the Spring
Boot BOM; find the current 4.1.x version yourself rather than copying a number
from anywhere.

**Part B — prove the graph.**
- Run `mvn -B clean verify` from the root and record the reactor summary order.
- Shuffle the `<modules>` list into reverse order. Re-run. Record whether the
  reactor order changed, and explain why.
- Run `mvn -T 1C clean verify` and record whether `orderflow-api` and
  `orderflow-worker` built concurrently.

**Part C — prove the scopes.** For each module, run
`mvn dependency:build-classpath -DincludeScope=runtime` and answer:
- Is `orderflow-domain`'s runtime classpath free of Spring? If not, find what
  dragged it in with `mvn dependency:tree`.
- Is the Postgres driver present in `orderflow-api`'s runtime classpath even
  though `orderflow-api` never declares it? Explain the path.

**Part D — break it deliberately.** Add
`<dependency>spring-boot-starter-web</dependency>` to the **parent's**
`<dependencies>` (not dependencyManagement). Re-run Part C on
`orderflow-domain`. Record how many artifacts the domain module gained. Then
revert, and write two sentences on why `<dependencyManagement>` is the default
answer.

**Part E — CI honesty.** Write a CI script that uses `verify`, not `install`.
Then run it twice: once normally, and once with
`-Dmaven.repo.local=$(mktemp -d)` to simulate a cold cache. Record how long each
takes and whether both pass. If the cold one fails, you have found something real
— write down what.

---

## Interview questions

### Q1 — "What does `mvn package` actually run, in order?"

**Mid-level answer:** "It compiles the code, runs the tests, and builds the jar."

**Senior answer:** "`package` is a phase in the default lifecycle, so Maven runs
every phase up to and including it. For `jar` packaging the goals that actually
execute are: `resources:resources`, `compiler:compile`,
`resources:testResources`, `compiler:testCompile`, `surefire:test`, then
`jar:jar`. The phases in between — `validate`, `initialize`, `process-classes`,
`prepare-package` and so on — exist but have nothing bound by default. Which
goals are bound is decided by the `<packaging>` element, so a `war` or a
`maven-plugin` project has a different set. And note `package` stops before
`verify`, so failsafe integration tests do **not** run — that is why CI should
say `verify`."

**What separates them:** naming the goals rather than describing the outcome,
knowing that packaging chooses the bindings, and volunteering the
`package`-vs-`verify` distinction unprompted.

**Follow-up:** "What is the difference between `verify` and `install`?" They are
checking whether you know `install` writes to the local repository and why that
makes it wrong for CI.

---

### Q2 — "Explain `provided` versus `runtime` versus `test` scope."

**Mid-level answer:** "`provided` is supplied by the container, `runtime` is
needed at runtime but not compile time, `test` is for tests."

**Senior answer:** "They are classpath-membership decisions across three
classpaths. `provided` is on compile and test but **not** runtime, and is not
packaged — it is a promise that the deployment environment supplies the jar. That
promise holds for a servlet API in an application server; it does not hold for a
Spring Boot fat jar, which supplies only what it packaged. Getting that wrong
gives you a green build and a `NoClassDefFoundError` at deploy, and unit tests
will not catch it because `provided` *is* on the test classpath. `runtime` is the
mirror image — absent at compile, present at run — which I use for JDBC drivers
so nobody can accidentally import a vendor class. `test` is the closest thing to
npm's `devDependencies`. Also worth knowing: `provided` and `test` are not
transitive, so if a library marks something `provided` you must declare it
yourself."

**What separates them:** framing scopes as classpath membership rather than
labels, naming the observable failure, and knowing the transitivity rule.

**Follow-up:** "You see `<scope>system</scope>` in a POM. What do you do?"
Correct answer: it is deprecated, points at an absolute filesystem path, and
should be replaced with a properly installed artifact.

---

### Q3 — "Why should CI run `verify` instead of `clean install`?"

**Mid-level answer:** "`install` is slower, and you do not need the artifact in
the local repo on CI."

**Senior answer:** "It is a correctness issue, not a speed one. `install` writes
your built artifacts into `~/.m2/repository` on the build machine. Every
resolution after that point finds them there, so the build is validating against
a cache it just populated. Two real failures follow: a module missing from the
reactor still resolves from the local repo, so a broken aggregation looks fine;
and on a persistent CI runner you can build successfully against a stale
snapshot from weeks ago. `verify` runs the same compilation, unit tests,
packaging and integration tests without that write. I'd have CI run
`mvn -B -ntp clean verify`, publish from a separate release job on a tag, and
periodically run a build with a fresh `-Dmaven.repo.local` to make sure a cold
cache actually works."

**What separates them:** identifying it as a *false-green* problem and naming the
specific ways the false green occurs.

**Follow-up:** "How would you catch the stale-snapshot case?" Looking for: clean
cache runs, or `-U` to force snapshot updates, or better, not using SNAPSHOTs
across team boundaries at all.

---

### Q4 — "In a multi-module build, how does Maven decide the build order?"

**Mid-level answer:** "It builds them in the order listed in `<modules>`."

**Senior answer:** "No — it topologically sorts the reactor by inter-module
dependencies. `<modules>` is a set of projects to include, not a sequence.
Listing order only breaks ties between modules with no dependency relationship.
Two consequences: a cycle between modules fails the build immediately with
'the projects in the reactor contain a cyclic reference', which is a module
boundary problem rather than a config problem; and modules with no dependency
between them can be built concurrently with `-T`. Also worth separating: the
`<parent>` relationship is inheritance of configuration, while `<modules>` is
aggregation of a build. One POM usually does both, but they are independent
mechanisms."

**What separates them:** knowing it is a graph, separating inheritance from
aggregation, and connecting the graph to parallel builds.

**Follow-up:** "You have a genuine cycle between two modules. What do you do?"
The answer is extract the shared abstraction into a third module — not an
exclusion, not a profile.

---

### Q5 — "Why would you import the Spring Boot BOM rather than use `spring-boot-starter-parent` as your parent?"

**Mid-level answer:** "Both work; importing is more flexible."

**Senior answer:** "A POM has exactly one parent, and in a multi-module repo I
want that slot for my own parent — my properties, my plugin management, my
enforcer rules, my module list. Importing `spring-boot-dependencies` with
`<type>pom</type><scope>import</scope>` inside my `<dependencyManagement>` gives
me Boot's tested version set without spending the parent slot. What I give up is
the small set of conveniences `starter-parent` adds on top of the BOM —
sensible plugin configuration and resource filtering defaults — and I have to
configure the `spring-boot-maven-plugin` version myself, which I'd do in
`<pluginManagement>`. For a single-module demo, `starter-parent` is fine. For
anything with more than one module, importing is the default."

**What separates them:** knowing that `starter-parent` is a superset of the BOM
rather than an alternative to it, and naming exactly what you give up.

**Follow-up:** "What happens if you import two BOMs that manage the same
artifact?" That is Topic 32 — first import wins, and your own
`<dependencyManagement>` entry beats both.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Maven's phases have been frozen since 2004 and you cannot add one. Argue that
   this is a *feature* rather than a limitation — then make the strongest case
   against yourself, using a build need you have actually had in Node.

2. `provided` scope is on the test classpath. Given that, describe a test you
   could write that would catch a wrong `provided` scope before deployment. Then
   say honestly whether you think anyone would write it.

3. `~/.m2` is shared across every project on your machine, while `node_modules`
   is per-project. Name one thing that gets *better* because of Maven's choice
   and one thing that gets *worse*, and say which you would pick if you were
   designing a build tool today.

4. Your parent POM declares `<dependencies>` containing `slf4j-api`. A colleague
   argues this is fine because "literally every module logs". Give the strongest
   argument for their position and the strongest against, and state your call.

5. `mvn install` on a developer machine and `mvn deploy` in a release job do
   nearly the same thing to different repositories. Why is it important that
   developers cannot easily run `deploy`?

6. The reactor refuses to build a cyclic module graph. Why is a cycle between
   *modules* fatal, when a cycle between *classes* inside one module compiles
   fine?

7. You inherit a project where `<packaging>` is `jar` but someone has manually
   bound `war:war` to the `package` phase. What would you expect to happen, and
   what does that tell you about the relationship between packaging and
   bindings?

---

## Quick reference card

### Commands you will actually type

```bash
mvn -B -ntp clean verify              # the CI command. -B batch, -ntp no transfer progress
mvn -T 1C clean verify                # parallel: one thread per CPU core
mvn clean install                     # local dev only, and know why
mvn -pl orderflow-api -am verify      # build one module (-pl) plus what it needs (-am)
mvn -o verify                         # offline: prove nothing needs downloading
mvn -U verify                         # force a re-check of SNAPSHOT dependencies
mvn -X verify                         # full debug output. Very long. Very useful.

mvn help:effective-pom                # the POM Maven actually uses
mvn help:describe -Dcmd=package       # what is bound to each phase
mvn dependency:tree                   # Topic 32
mvn dependency:analyze                # declared-vs-used
mvn dependency:build-classpath -DincludeScope=runtime
mvn dependency:resolve                # download everything, resolve nothing else
mvn versions:display-dependency-updates
```

### Scope cheat sheet

| Scope | compile cp | test cp | runtime cp | packaged | transitive |
|---|---|---|---|---|---|
| `compile` | yes | yes | yes | yes | yes (as compile) |
| `provided` | yes | yes | no | no | **no** |
| `runtime` | no | yes | yes | yes | yes (as runtime) |
| `test` | no | yes | no | no | **no** |
| `system` | yes | yes | no | no | no — deprecated |
| `import` | only valid inside `<dependencyManagement>` on a `pom` type | | | | |

### Phase landmarks

```
validate → compile → test → package → verify → install → deploy
                                        ^        ^         ^
                                        |        |         └ writes to the REMOTE repo
                                        |        └ writes to ~/.m2 (LOCAL)
                                        └ the CI stop point
```

### Gotchas checklist

- [ ] CI runs `verify`, never `install`.
- [ ] `maven.compiler.release`, not `source`/`target`.
- [ ] `-DskipTests` skips running; `-Dmaven.test.skip=true` skips *compiling* too.
- [ ] Parent `<dependencies>` is inherited by everyone with no opt-out. Use
      `<dependencyManagement>`.
- [ ] `provided` is not packaged. Fat jars do not "provide" anything.
- [ ] `<modules>` order is not build order.
- [ ] `spring-boot-maven-plugin` only on runnable modules.
- [ ] Commit `mvnw` and the wrapper properties.
- [ ] `mvn --version` shows which JDK Maven runs on — check it first in any
      "works on my machine" argument.

---

## When would I use this at work?

**1. Your first week on a Java team, reading an unfamiliar repo.**
You open the root `pom.xml`, read `<modules>` and each module's dependencies, and
in ten minutes you have the architecture: which module is pure domain, which
talks to the database, which is deployable. In a Node monorepo you would have to
read the code. The POM is a design document that cannot go stale, because the
build enforces it.

**2. Diagnosing a deploy that worked in staging and failed in production.**
The stack trace says `NoClassDefFoundError`. You do not read the application
code. You run `mvn dependency:build-classpath -DincludeScope=runtime`, compare it
against what is actually in the deployed jar, and find the `provided` scope
somebody added last sprint. Fifteen minutes instead of a day.

**3. Splitting a service that has grown too big.**
Someone proposes extracting the pricing engine. You do not argue about it in the
abstract — you create `orderflow-pricing`, move the classes, and let the reactor
tell you the truth. If it builds, the boundary was real. If you get a cyclic
reference, the boundary was imaginary and you now know exactly which two classes
made it imaginary. The build tool is doing architecture review for you.

---

## Connected topics

**Prerequisites:**
- **01–30** generally — you need something worth compiling. Specifically
  **20 (JPMS)**, because a Maven module and a JPMS module are *different things*
  and conflating them is a common beginner error: Maven modules are a build-time
  aggregation, JPMS modules are a runtime encapsulation boundary.
- **27, 28, 29** — the records, sealed types and pattern matching you will put in
  `orderflow-domain` in Exercise 2.

**This unlocks:**
- **32 — Dependency resolution.** This topic told you where dependencies are
  declared. That one tells you which *version* actually wins, and why the answer
  can silently change.
- **33 — Gradle.** You cannot evaluate the alternative until you know exactly
  what Maven's rigidity buys.
- **34 — Supply chain.** SBOMs and CVE triage operate on the dependency graph you
  just learned to read.
- **35 — `ApplicationContext`.** The spine starts. It compiles into the modules
  you built in Exercise 3.
- **42 — Auto-configuration mechanics.** Boot finds
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
  **on the classpath** — which means the set of active auto-configurations is a
  direct function of your POM's dependencies and their scopes. Trap 4 is a
  Topic 42 bug that starts life as a Maven mistake.
- **61 — Testcontainers.** A `test`-scoped dependency that starts real
  infrastructure. Scope correctness is what keeps Docker out of your production
  jar.
- **83 — GraalVM native image.** The native-image build is a Maven plugin bound
  to a phase, and closed-world analysis makes your exact classpath — not your
  source — the input.
- **122 — Layered Docker jars.** `spring-boot-maven-plugin`'s layering splits the
  fat jar by change frequency. It can only do that because the POM already
  distinguishes your code from your dependencies.

---

*Java baseline 21, running on JDK 25. Target Spring Boot 4.1 / Framework 7.0 /
Jakarta EE 11. Maven's lifecycle model has not changed in twenty years and will
not change under you; the version numbers of every plugin and library mentioned
here will. That asymmetry is the reason this document contains no third-party
version numbers — every one of them is a `SET-ME` you resolve with a command.*
