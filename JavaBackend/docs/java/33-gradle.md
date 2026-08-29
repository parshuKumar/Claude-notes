# 33 — Gradle, and When a Team Should Choose It

## Phase: 3 — Build & Supply Chain
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: `orderflow` stays on Maven — that is a deliberate choice this document justifies. You will build a parallel Gradle skeleton of the same four modules as an exercise, so that when you meet a Gradle codebase at work you are not starting from zero.

---

## ELI5 anchor

Maven is a **form you fill in.** The questions are printed. You write answers in
the boxes. Two people filling in the same form produce recognisably the same
thing, and a stranger can read either one.

Gradle is a **workshop with a lathe in it.** You write a program that produces
your build. That is enormously more powerful. It is also how you end up with a
build that only one person understands, that behaves differently on Tuesdays,
and that nobody dares touch after they leave.

Both statements are true at once. That is the whole topic. Anyone who tells you
"Gradle is better" or "Maven is better" without naming the team size, the repo
size, and who owns the build is selling you something.

The honest one-liner: **Gradle trades declarative rigidity for speed and
expressiveness, and the cost of that trade is paid in maintenance, by whoever is
still there in three years.**

---

## The bridge from what you know

### Gradle is much closer to what you already do than Maven is

This is the good news and also the trap.

```javascript
// Node: your build IS a program. You have always known this.
const isCI = process.env.CI === 'true';
module.exports = {
  mode: isCI ? 'production' : 'development',
  entry: glob.sync('./src/**/index.ts'),
};
```

```kotlin
// Gradle Kotlin DSL: also a program.
val isCI = System.getenv("CI") == "true"
tasks.test {
    maxParallelForks = if (isCI) 4 else 1
}
```

If you have written a `webpack.config.js` or a gnarly `package.json` scripts
block, Gradle will feel *familiar* in a way Maven never does. You will
immediately reach for conditionals, loops and shared functions.

**Resist that instinct for the first month.** Everything that makes Node builds
hard to maintain — env-conditional logic, implicit ordering, config that reads
the filesystem at load time — is available in Gradle and produces the same
outcome, in a language where the failure is slower to diagnose.

### The translation table

| You know | Gradle | Verdict |
|---|---|---|
| `package.json` scripts | Tasks | **PARTIAL** — tasks form a DAG with declared inputs and outputs, not a shell chain |
| `webpack.config.js` being a program | `build.gradle.kts` being a program | **HONEST ANALOGUE** — and it carries the same risks |
| `package-lock.json` | Gradle dependency locking (`gradle.lockfile`) | **HONEST ANALOGUE** — the closest thing in the JVM world |
| `npm ci` (locked, reproducible install) | `./gradlew build --write-locks` then locked builds | **PARTIAL** — opt-in, per configuration |
| Turborepo / Nx remote caching | Gradle build cache (local + remote) | **HONEST ANALOGUE** |
| `devDependencies` | `testImplementation` | **PARTIAL** — Gradle has more configurations than npm has categories |
| Nothing you have | `api` vs `implementation` | **NO ANALOGUE** — and it is Gradle's single best idea |
| `nvm` / `.nvmrc` | the Gradle wrapper (`gradlew`) | **HONEST ANALOGUE** |
| `npm ls` | `./gradlew dependencies` / `dependencyInsight` | **HONEST ANALOGUE** |

### The one big carry-over from Topic 32

Gradle also produces a **flat classpath with exactly one version per module**.
Everything you learned in Topic 32 about `NoSuchMethodError`, lazy linking and
one-version-wins still applies.

What changes is the **selection rule**:

| | Maven | Gradle |
|---|---|---|
| Default conflict resolution | **nearest wins** (shallowest depth), ties by declaration order | **highest wins** (newest version requested by anyone) |
| Reports the decision? | no | `dependencyInsight` explains it precisely, and `--scan` shows it |
| Override mechanism | `<dependencyManagement>` | constraints, `strictly` versions, `platform()`, resolution rules |

**This means the same project, built by both tools, can put different versions on
the classpath.** If you are migrating a build, this is not a detail — it is the
thing most likely to change runtime behaviour, and it is Trap 4.

---

## What is this?

**Gradle is a general-purpose build automation tool built on a task graph.**

Three ideas replace Maven's three:

1. **Tasks, not phases.** A task is a named unit of work with declared **inputs**
   and **outputs**. Tasks declare dependencies on other tasks, forming a directed
   acyclic graph. Gradle runs the minimal set needed for what you asked for.

2. **Incrementality is the design goal.** Because a task declares its inputs and
   outputs, Gradle can hash them and skip the task when nothing changed
   (`UP-TO-DATE`), or fetch its outputs from a cache (`FROM-CACHE`). This is
   where the speed comes from — not from being "written better".

3. **The build is code.** `build.gradle.kts` is a Kotlin script. `settings.gradle.kts`
   is a Kotlin script. Plugins are Java/Kotlin classes. There is no XML and no
   fixed lifecycle.

### The two build phases you must internalise

Gradle runs your build script in **two distinct phases**, and almost every
confusing Gradle error comes from conflating them.

| Phase | When | What happens | Rule |
|---|---|---|---|
| **Configuration** | every build, before any work | Your entire `build.gradle.kts` **executes**. Tasks are created and configured. | Runs even for tasks you did not ask for. Keep it cheap. |
| **Execution** | after configuration | The selected tasks' actions run. | Must not reach back into the project model. |

```kotlin
tasks.register("printSummary") {
    println("A: this runs at CONFIGURATION time, on every single build")
    doLast {
        println("B: this runs at EXECUTION time, only if the task runs")
    }
}
```

Run `./gradlew help` — a build that does nothing at all — and line A still
prints. Line B does not.

That surprises everyone once. It is also the root of Trap 3: the **configuration
cache** works by serialising the configured task graph so configuration can be
skipped entirely, which is only possible if no task action reaches back into the
project model at execution time.

---

## Why does it matter?

You are being taught Maven as the default. So why spend a topic on Gradle?

1. **You will meet it.** Android is Gradle-only. A large share of JVM monorepos
   are Gradle. Kotlin projects default to it. If you cannot read a
   `build.gradle.kts`, a chunk of the job market is closed.

2. **"When would you choose Gradle?" is a standard senior interview question**,
   and it is really a question about whether you can reason about engineering
   trade-offs or only about tools. Q3 below.

3. **Gradle solves two problems Maven genuinely has not**: real dependency
   locking (Topic 32's honest gap) and the `api`/`implementation` distinction,
   which Maven has no equivalent for at all.

4. **You may inherit a Gradle build that has rotted**, and the rot has a specific
   shape you can learn to recognise. Trap 1.

---

## Syntax breakdown

### The files

```
orderflow-gradle/
├── settings.gradle.kts             <- which projects exist. REQUIRED.
├── build.gradle.kts                <- root build
├── gradle/
│   ├── libs.versions.toml          <- version catalog: your dependency versions
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew  /  gradlew.bat         <- COMMIT THESE
├── build-logic/                    <- convention plugins (the discipline)
└── orderflow-domain/
    └── build.gradle.kts
```

| File | Maven equivalent | Note |
|---|---|---|
| `settings.gradle.kts` | `<modules>` in the parent | **Required.** Without it you have a single-project build. |
| `build.gradle.kts` | `pom.xml` | One per project, all optional except the root |
| `gradle/libs.versions.toml` | `<properties>` + `<dependencyManagement>` | The version catalog. Use it. |
| `gradlew` | `mvnw` | Same idea, same importance |
| `build-logic/` | the parent POM's `<pluginManagement>` | Shared build logic as real, testable plugins |

### `settings.gradle.kts`

```kotlin
rootProject.name = "orderflow"

include(
    "orderflow-domain",
    "orderflow-persistence",
    "orderflow-api",
    "orderflow-worker",
)

// Where plugins are resolved from
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

// Where dependencies are resolved from, for every project.
// FAIL_ON_PROJECT_REPOS forbids a subproject declaring its own repositories,
// which is the single best guard against a rogue module pulling from
// somewhere unaudited. See Topic 34.
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
    }
}
```

`include("orderflow-domain")` maps to the directory `orderflow-domain/`. Like
Maven's `<modules>`, **this is not build order** — Gradle derives order from the
task graph.

### `gradle/libs.versions.toml` — the version catalog

This is Gradle's answer to `<properties>` plus `<dependencyManagement>`, and it
is genuinely nicer than either.

```toml
[versions]
# Set these to real current releases. Find them with:
#   ./gradlew dependencyUpdates      (the ben-manes versions plugin)
# or from start.spring.io. I am not writing version numbers from memory.
springBoot = "SET-ME"
postgresql = "SET-ME"

[libraries]
spring-boot-starter-web  = { module = "org.springframework.boot:spring-boot-starter-web" }
spring-boot-starter-jpa  = { module = "org.springframework.boot:spring-boot-starter-data-jpa" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test" }
postgresql               = { module = "org.postgresql:postgresql", version.ref = "postgresql" }

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "springBoot" }

[bundles]
web = ["spring-boot-starter-web"]
```

Used from a build script as `libs.spring.boot.starter.web` — dashes become dots,
and your IDE autocompletes it. Note that the Spring starters have **no version**:
they are going to be managed by the Boot BOM via `platform()` below, exactly as
in Maven.

### `build.gradle.kts` — the configurations

```kotlin
plugins {
    `java-library`
    alias(libs.plugins.spring.boot)
}

java {
    toolchain {
        // Gradle DOWNLOADS this JDK if you don't have it. Maven has no equivalent.
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

dependencies {
    // Consume the Spring Boot BOM. platform() is Gradle's <scope>import</scope>.
    implementation(platform(libs.spring.boot.bom))

    api(project(":orderflow-domain"))          // leaks to MY consumers
    implementation(libs.spring.boot.starter.jpa) // does NOT leak
    runtimeOnly(libs.postgresql)
    compileOnly(libs.some.annotations)
    testImplementation(libs.spring.boot.starter.test)
    annotationProcessor(libs.lombok)
}
```

**The configurations, mapped to what you know:**

| Gradle | Maven scope | On MY compile cp | On MY runtime cp | On my CONSUMER's compile cp |
|---|---|---|---|---|
| `api` | `compile` | yes | yes | **yes** |
| `implementation` | `compile` | yes | yes | **no** |
| `compileOnly` | `provided` | yes | **no** | no |
| `runtimeOnly` | `runtime` | **no** | yes | (runtime only) |
| `testImplementation` | `test` | test only | test only | no |
| `annotationProcessor` | (a plugin config in Maven) | processor path only | no | no |
| `developmentOnly` | (Boot plugin) | yes | dev run only, not in the jar | no |

**`api` versus `implementation` is Gradle's single best idea and Maven has
nothing like it.**

In Maven, every `compile` dependency is visible to everyone who depends on you.
Your consumers can compile against your internal libraries by accident, and then
you cannot upgrade them without breaking people who were never supposed to know
they existed.

In Gradle:

- `api` = "this type appears in my **public signatures**, so my consumers need it
  to compile against me." Return types, parameter types, extended classes,
  thrown checked exceptions, public field types.
- `implementation` = "I use this **inside** my methods. Nobody else's business."

The payoff is not only cleaner APIs — it is **compile avoidance**. When you
change an `implementation` dependency, Gradle knows your consumers' compilation
cannot be affected and does not recompile them. On a 200-module repo that is the
difference between a 4-minute build and a 40-second one.

The rule: **if you can't name the public signature it appears in, it is
`implementation`.**

### Tasks

```kotlin
tasks.named<Test>("test") {
    useJUnitPlatform()
    maxParallelForks = (Runtime.getRuntime().availableProcessors() / 2).coerceAtLeast(1)
}

// A custom task, declared properly: inputs and outputs are DECLARED,
// which is what makes it cacheable and incremental.
abstract class GenerateBuildInfo : DefaultTask() {
    @get:Input abstract val version: Property<String>
    @get:OutputFile abstract val outputFile: RegularFileProperty

    @TaskAction
    fun generate() {
        outputFile.get().asFile.writeText("version=${version.get()}\n")
    }
}

tasks.register<GenerateBuildInfo>("generateBuildInfo") {
    version.set(project.version.toString())
    outputFile.set(layout.buildDirectory.file("generated/build-info.properties"))
}
```

| Element | What it means |
|---|---|
| `@get:Input` | Part of the cache key. Change it, the task re-runs. |
| `@get:OutputFile` | What the task produces. Gradle can restore it from cache. |
| `@TaskAction` | The execution-time body. Everything outside it is configuration time. |
| `tasks.register` | **Lazy** — the task is only configured if it is actually needed. |
| `tasks.create` | **Eager** — configured on every build. Avoid it in new code. |

The `@Input`/`@Output` annotations are not documentation. They **are** the
incrementality mechanism. A task with undeclared inputs will be wrongly marked
`UP-TO-DATE` and produce stale output, which is a genuinely nasty bug class that
Maven cannot have because Maven does not try to be incremental.

### Dependency locking — the real lockfile

```kotlin
dependencyLocking {
    lockAllConfigurations()
}
```

```bash
./gradlew dependencies --write-locks      # generate/update gradle.lockfile
./gradlew build                           # now FAILS if resolution differs
```

This produces a `gradle.lockfile` per project listing every resolved
version — including deep transitives you never named. Commit it. From then on,
any change in the resolved graph fails the build until someone regenerates the
lock deliberately.

**This is the thing Maven does not have.** Topic 32 was honest about the gap; this
is what closes it. If your team has been burned by a silent transitive change, or
you work somewhere regulated, this is a legitimate reason to choose Gradle on its
own.

### Constraints and strict versions — Gradle's `dependencyManagement`

```kotlin
dependencies {
    constraints {
        // "if this ends up in the graph, use at least this version"
        implementation("com.vendor:layout-engine:4.1.0") {
            because("4.0.x has CVE-2026-XXXXX; see SEC-412")
        }
    }

    implementation("com.fasterxml.jackson.core:jackson-databind") {
        version {
            strictly("2.17.2")   // FAILS the build if anything needs a different version
        }
    }
}
```

| Mechanism | Meaning |
|---|---|
| `require` (the default) | "at least this version" — a higher one may still win |
| `prefer` | "this one, unless something else has an opinion" |
| `strictly` | "exactly this range; **fail** if anything conflicts" |
| `reject` | "never these versions" |
| `constraints { }` | affects the version only if the artifact is in the graph |
| `platform(...)` | consume a Maven BOM as version recommendations |
| `enforcedPlatform(...)` | consume a BOM as **strict** versions. Powerful and dangerous — it overrides everything downstream and can force an incompatible combination. Prefer `platform`. |

Note `because("...")`. Gradle lets you attach a *reason* to a constraint, and
`dependencyInsight` prints it. That is a genuinely excellent feature — the
version override in your build tells the next engineer why it exists. Use it for
every CVE pin (Topic 34).

### Convention plugins — the discipline that keeps a Gradle build alive

The single most important structural practice in Gradle. Instead of `subprojects { }`
blocks in the root build (which configure other projects from outside and are the
main cause of Trap 1), you write a plugin.

`build-logic/settings.gradle.kts`:
```kotlin
rootProject.name = "build-logic"
include("conventions")
```

`build-logic/conventions/build.gradle.kts`:
```kotlin
plugins { `kotlin-dsl` }
```

`build-logic/conventions/src/main/kotlin/orderflow.java-conventions.gradle.kts`:
```kotlin
plugins {
    `java-library`
}

java {
    toolchain { languageVersion.set(JavaLanguageVersion.of(21)) }
}

tasks.withType<Test>().configureEach {
    useJUnitPlatform()
}

tasks.withType<JavaCompile>().configureEach {
    options.compilerArgs.addAll(listOf("-Xlint:all", "-Werror"))
}

// Reproducible archives - Topic 34.
tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false
    isReproducibleFileOrder = true
}
```

Then each module says:

```kotlin
plugins {
    id("orderflow.java-conventions")
}
```

Every module now opts *in* to shared configuration, by name, visibly. Compare
that with a root-level `subprojects { }` block, where a module's configuration is
determined by a file the module does not reference and where the only way to find
out what a module does is to read the whole root build.

---

## Example 1 — minimal

One project, Kotlin DSL, Java 21.

`settings.gradle.kts`:
```kotlin
rootProject.name = "orderflow-scratch"
```

`build.gradle.kts`:
```kotlin
plugins {
    `java-library`
}

repositories {
    mavenCentral()
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

dependencies {
    testImplementation(platform("org.junit:junit-bom:SET-ME"))
    testImplementation("org.junit.jupiter:junit-jupiter")
}

tasks.named<Test>("test") {
    useJUnitPlatform()
}
```

```bash
gradle wrapper           # generates gradlew + wrapper properties. Commit them.
./gradlew build
```

Four things to notice against the Maven equivalent:

1. **`toolchain` downloads a JDK if you do not have one.** Maven has no
   equivalent — it uses whatever JVM Maven itself is running on. This is a real
   Gradle advantage on a team with mixed local setups.
2. **`platform("...junit-bom...")` is `<scope>import</scope>`.** Same mechanism,
   less ceremony.
3. **`useJUnitPlatform()` is required.** Gradle's `Test` task defaults to JUnit 4
   discovery for historical reasons. Forget it and your tests silently do not run
   — you get `BUILD SUCCESSFUL` and zero tests, which is worse than a failure.
4. **`SET-ME` again.** I am not writing a JUnit BOM version from memory. Get it
   from Maven Central or `start.spring.io`.

---

## Example 2 — production scenario: `orderflow` as a Gradle build

The same four modules from Topic 31, built the way a team that intends to keep
the build maintainable would build them.

```
orderflow/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle/libs.versions.toml
├── build-logic/
│   ├── settings.gradle.kts
│   └── conventions/
│       ├── build.gradle.kts
│       └── src/main/kotlin/
│           ├── orderflow.java-conventions.gradle.kts
│           └── orderflow.boot-app-conventions.gradle.kts
├── orderflow-domain/build.gradle.kts
├── orderflow-persistence/build.gradle.kts
├── orderflow-api/build.gradle.kts
└── orderflow-worker/build.gradle.kts
```

### `settings.gradle.kts`

```kotlin
pluginManagement {
    includeBuild("build-logic")          // our convention plugins
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { mavenCentral() }
}

rootProject.name = "orderflow"

include(
    "orderflow-domain",
    "orderflow-persistence",
    "orderflow-api",
    "orderflow-worker",
)
```

### `build-logic/conventions/src/main/kotlin/orderflow.java-conventions.gradle.kts`

```kotlin
plugins { `java-library` }

group = "com.orderflow"
version = "1.0.0-SNAPSHOT"

java {
    toolchain { languageVersion.set(JavaLanguageVersion.of(21)) }
    withSourcesJar()
}

tasks.withType<Test>().configureEach {
    useJUnitPlatform()
    testLogging { events("failed", "skipped") }
}

tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false     // Topic 34: reproducible builds
    isReproducibleFileOrder = true
}

dependencyLocking { lockAllConfigurations() }   // the real lockfile
```

### `orderflow-domain/build.gradle.kts`

```kotlin
plugins { id("orderflow.java-conventions") }

dependencies {
    testImplementation(platform(libs.spring.boot.bom))
    testImplementation(libs.junit.jupiter)
}
```

Same rule as the Maven version: **no Spring in the domain module.** In Gradle you
can make that a *build failure* rather than a convention, which Maven cannot do
without a custom enforcer rule:

```kotlin
// in the java-conventions plugin, or just in this module
configurations.named("compileClasspath") {
    resolutionStrategy.eachDependency {
        if (requested.group.startsWith("org.springframework")) {
            throw GradleException(
                "orderflow-domain must not depend on Spring. Found: ${requested.group}:${requested.name}"
            )
        }
    }
}
```

This is a fair example of Gradle's power *and* its cost: five lines give you an
architectural guarantee Maven cannot express, and those five lines are now build
code somebody has to maintain and understand.

### `orderflow-persistence/build.gradle.kts`

```kotlin
plugins { id("orderflow.java-conventions") }

dependencies {
    implementation(platform(libs.spring.boot.bom))

    // api: OUR domain types appear in this module's public repository signatures,
    // so consumers need them to compile.
    api(project(":orderflow-domain"))

    // implementation: JPA is an internal concern of this module.
    // orderflow-api gets our repositories, NOT our Hibernate.
    implementation(libs.spring.boot.starter.jpa)

    runtimeOnly(libs.postgresql)

    testImplementation(libs.spring.boot.starter.test)
    testImplementation(libs.testcontainers.postgresql)   // Topic 61
}
```

**Read that `api`/`implementation` split carefully.** In the Maven version of
this project, `orderflow-api` could `import jakarta.persistence.EntityManager`
and compile, because Maven's `compile` scope is transitive. Here it cannot — the
build stops it. The persistence module's choice of ORM is genuinely encapsulated.
That is real architecture enforcement from the build tool, and it is the single
strongest technical argument for Gradle in a large codebase.

### `orderflow-api/build.gradle.kts`

```kotlin
plugins {
    id("orderflow.java-conventions")
    alias(libs.plugins.spring.boot)
}

dependencies {
    implementation(platform(libs.spring.boot.bom))

    implementation(project(":orderflow-persistence"))
    implementation(libs.spring.boot.starter.web)
    implementation(libs.spring.boot.starter.actuator)

    testImplementation(libs.spring.boot.starter.test)
}

tasks.named<org.springframework.boot.gradle.tasks.bundling.BootJar>("bootJar") {
    // Topic 122: layered jars for Docker layer caching
    layered { enabled.set(true) }
}
```

> **[BOOT 3.x DELTA]**
> Two Gradle-specific points.
> **First**, Boot 4 modularised into many smaller jars. A `libs.versions.toml`
> entry copied from a Boot 3.x project may name an artifact that no longer exists
> under those coordinates in 4.x — and because a version catalog entry is just a
> string, you will find out at resolution time, not at edit time. Regenerate a
> starter project against your target Boot version and diff the catalogs.
> **Second**, older Spring Boot Gradle setups used the separate
> `io.spring.dependency-management` plugin to emulate Maven's BOM behaviour,
> because Gradle had no native BOM support at the time. Gradle's native
> `platform()` has made that plugin unnecessary for most projects, and mixing the
> two produces confusing precedence. On a modern Boot version, use
> `implementation(platform(libs.spring.boot.bom))` and drop the plugin. If you
> inherit a build with both, that is a cleanup worth doing.
> **Third**, Boot 3.5 left OSS support in June 2026, so a 3.5 build receives no
> free CVE patches — Topic 34.

### The reactor equivalent

Gradle derives order from the task graph, exactly as Maven derives it from the
module graph. `orderflow-api` and `orderflow-worker` are independent, so Gradle
builds them in parallel by default when `org.gradle.parallel=true` in
`gradle.properties`:

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.jvmargs=-Xmx2g
```

Those four lines are most of Gradle's speed advantage. Turning them on is also
where a rotted build reveals itself — see Trap 3.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the build that became a program

**Wrong:** a root `build.gradle.kts` that has grown organically over three years.

```kotlin
allprojects {
    repositories { mavenCentral(); maven("https://internal.example.com/repo") }
}

subprojects {
    apply(plugin = "java")

    if (name.endsWith("-api") || name == "orderflow-worker") {
        apply(plugin = "org.springframework.boot")
    }

    if (System.getenv("CI") != null) {
        tasks.withType<Test> { maxParallelForks = 8 }
    } else {
        tasks.withType<Test> { maxParallelForks = 1 }
    }

    if (file("src/main/proto").exists()) {
        apply(plugin = "com.google.protobuf")
    }

    version = if (System.getenv("BUILD_NUMBER") != null)
        "1.0.${System.getenv("BUILD_NUMBER")}" else "1.0.0-SNAPSHOT"
}
```

Everything here was reasonable when it was added.

**Exact symptoms, and there are several:**

- Opening `orderflow-persistence/build.gradle.kts` tells you almost nothing about
  what that module does. Its real configuration is in a file it never mentions.
- A new module named `orderflow-reporting-api` unexpectedly becomes a Spring Boot
  application because of a string suffix check, and produces a fat jar nothing
  can depend on.
- Tests pass locally and fail on CI, or vice versa, because parallelism differs.
- Enabling the configuration cache fails with
  `configuration cache problems found: read system property 'CI'`.
- A local build and a CI build of the same commit produce different jars, because
  `version` differs.
- Deleting `src/main/proto` from a module silently changes its build.

**Root cause:** the build is a program whose behaviour depends on ambient state —
environment variables, filesystem contents, project names. It is not a
description of the project; it is a computation over the environment. Gradle will
happily let you do this, and there is no phase where anyone is forced to look at
the result.

**Fix, in order of value:**

1. Move shared configuration into **convention plugins** in `build-logic`. Each
   module then opts in by name: `plugins { id("orderflow.java-conventions") }`.
   Configuration becomes visible from the module you are reading.
2. Delete every `if (System.getenv(...))`. Environment differences belong in
   *invocation*, not in the build: `./gradlew test -PmaxForks=8` or a
   `gradle.properties` value CI overrides.
3. Turn on `org.gradle.configuration-cache=true`. It will fail loudly on exactly
   the patterns above. **Use it as a linter for build hygiene**, not just as a
   speed feature. This is the highest-leverage single action on a rotted build.
4. Replace `subprojects { }` and `allprojects { }` with plugins entirely. Treat
   their presence as a code smell.
5. Set `version` from a single declared source, not from an env var with a
   fallback.

**The general principle:** Maven's rigidity means a Maven build cannot rot in
this way. That is the *feature* you gave up. If you choose Gradle, you must
recreate the rigidity through discipline, and discipline needs an owner.

---

### Trap 2 — `api` where you meant `implementation`

**Wrong:** every dependency declared as `api` because "that is how Maven worked
and everything compiled".

```kotlin
dependencies {
    api(libs.spring.boot.starter.jpa)
    api(libs.jackson.databind)
    api(libs.commons.lang3)
}
```

**Exact symptoms:**

- Change one line in `orderflow-persistence` and Gradle recompiles
  `orderflow-api` and `orderflow-worker` too. On a large repo, incremental builds
  stop being incremental and Gradle's main advantage evaporates. You will
  literally see it in `./gradlew build --profile`.
- Someone writes `import jakarta.persistence.EntityManager` in
  `orderflow-api` — a module that was never supposed to know an ORM exists — and
  it compiles. Your layering is gone with no build failure.
- You try to swap Hibernate for something else and discover four modules compile
  against it.

**The mirror-image failure:** everything declared `implementation` on a genuine
*library*. A consumer calls a method whose return type they cannot see:

```
> Task :consumer:compileJava FAILED
error: cannot access com.orderflow.domain.Order
  class file for com.orderflow.domain.Order not found
```

**Root cause:** `api` and `implementation` encode a design decision — is this
type part of my public contract? — and defaulting either way skips the decision.

**Fix:** the mechanical rule. Look at every `public` and `protected` signature in
the module: return types, parameter types, superclasses, implemented interfaces,
thrown checked exceptions, public field types. A dependency appearing in any of
those is `api`. Everything else is `implementation`. If you cannot name the
signature, it is `implementation`.

Then verify: `./gradlew :orderflow-api:dependencies --configuration compileClasspath`
should be much shorter than `runtimeClasspath`. If they are the same length, you
have declared everything `api`.

---

### Trap 3 — configuration-cache failures that reveal the build is not reproducible

**Wrong:** turning on the configuration cache on an old build and assuming
failures are Gradle's fault.

```properties
org.gradle.configuration-cache=true
```

**Exact symptom:** a report file with entries like:

```
2 problems were found storing the configuration cache.
- Task `:orderflow-api:bootJar` of type `BootJar`: invocation of 'Task.project'
  at execution time is unsupported.
- Build file 'build.gradle.kts': read system property 'CI'
See the complete report at file:///.../build/reports/configuration-cache/.../configuration-cache-report.html
```

**Root cause — and this is the important reframe:** the configuration cache works
by serialising the *configured task graph* after the configuration phase and
reusing it. That is only sound if a task's behaviour is fully determined by its
declared inputs. A task that reads `project` at execution time, or a script that
reads an environment variable during configuration, has behaviour that depends on
state the cache cannot capture.

So a configuration-cache failure is not an incompatibility. **It is a report that
your build's output is not a pure function of its declared inputs** — which is
exactly the property you need for correct incremental builds, correct build
caching, and reproducible artifacts (Topic 34). The cache is telling you
something true.

**Fix:**

- Capture what a task needs at **configuration** time into a `Property<T>`, and
  read only that at execution time:

```kotlin
abstract class WriteVersion : DefaultTask() {
    @get:Input abstract val appVersion: Property<String>
    @get:OutputFile abstract val out: RegularFileProperty

    @TaskAction fun run() = out.get().asFile.writeText(appVersion.get())
}

tasks.register<WriteVersion>("writeVersion") {
    appVersion.set(project.version.toString())     // configuration time: fine
    out.set(layout.buildDirectory.file("version.txt"))
}
```

- Replace `System.getenv("X")` with `providers.environmentVariable("X")`, which
  is a tracked provider the cache understands.
- Replace `System.getProperty` with `providers.systemProperty`.
- Replace file reads at configuration time with `providers.fileContents(...)`.

**Do this even if you do not care about the speed**, because every one of these
fixes also makes your build more reproducible.

---

### Trap 4 — assuming Maven's resolution rules

**Wrong:** migrating a Maven build to Gradle module by module, verifying that
each module compiles, and shipping.

**Exact symptom:** the build is green, the tests pass, and in production a
library behaves differently — a different date format in JSON, a changed default
in an HTTP client, or a `NoSuchMethodError` in the *opposite* direction from the
one you are used to (something compiled against an older API meeting a newer
one, which surfaces as `AbstractMethodError` or
`IncompatibleClassChangeError`).

**Root cause:** **Maven picks nearest-wins; Gradle picks highest-wins.** For any
artifact where a deep transitive requested a newer version than a shallow one,
the two tools select *different versions*. Maven picked the shallow one; Gradle
picks the newest. On a Spring project with several hundred artifacts there will
be dozens of these.

**Fix:**

1. Before trusting a migration, diff the resolved graphs:

```bash
# Maven side
mvn dependency:list -DincludeScope=runtime -DoutputFile=maven-deps.txt

# Gradle side
./gradlew :orderflow-api:dependencies --configuration runtimeClasspath > gradle-deps.txt
```

   Normalise both to `group:artifact:version` lines, sort, and diff. **Every
   difference is a behaviour change you have not tested.**

2. For anything that differs and matters, add a constraint with a `because`.
3. Enable dependency locking and commit the lockfiles, so that the graph you
   verified is the graph that keeps shipping.

**Do not skip step 1 because the build is green.** A green build proves the
compile classpath is consistent. Topic 32 already taught you what that is worth.

---

### Trap 5 — the wrapper, dynamic versions, and "it worked yesterday"

**Wrong:** any of these, and they compound:

```kotlin
dependencies {
    implementation("com.vendor:layout-engine:4.+")          // dynamic version
    implementation("com.orderflow:shared-lib:1.0-SNAPSHOT") // changing module
}
```

plus a `.gitignore` that excludes `gradle/wrapper/gradle-wrapper.jar` (a
well-meant supply-chain policy that some organisations adopt), so different
machines use whatever Gradle they happen to have.

**Exact symptoms:**

- A build that succeeded yesterday fails today with no commits in between.
- A build succeeds today and produces a *different jar* from yesterday's, same
  commit.
- Two engineers on the same commit get different dependency trees.
- CI passes; a rebuild of the exact same tag two weeks later fails.
- Gradle prints `Deprecated Gradle features were used in this build` on one
  machine and not another.

**Root cause:** three separate sources of nondeterminism.

- `4.+` resolves against the repository's *current* contents, and Gradle caches
  that answer for only 24 hours by default.
- A `-SNAPSHOT` is a changing module: same coordinates, different bytes, cached
  for 24 hours.
- An uncommitted wrapper means the build tool version is ambient state.

**Fix:**

- Never use dynamic versions or `-SNAPSHOT` in anything you release. Fail the
  build on them:

```kotlin
configurations.configureEach {
    resolutionStrategy {
        failOnDynamicVersions()
        failOnChangingVersions()
    }
}
```

- Commit `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.properties`
  **and** `gradle-wrapper.jar`. If your organisation's policy forbids committing
  jars, use Gradle's wrapper checksum verification
  (`distributionSha256Sum` in the wrapper properties) rather than deleting the
  wrapper — a missing wrapper is a worse supply-chain outcome than a verified
  one.
- Enable dependency locking so the resolved graph cannot drift silently.

---

## Hands-on proof

Every command below is one **you** run. I have no Gradle and no JVM in this
session, so nothing below is captured output. Where a line's shape is shown, it
is labelled as an illustration.

### Setup

```bash
mkdir -p ~/java-lab/33 && cd ~/java-lab/33
gradle --version      # note Gradle version, JVM, and Kotlin DSL version
```

**How to read `gradle --version`:** it prints the Gradle version, the **JVM it is
running on**, and the launcher JVM args. If your project uses toolchains, the
JVM shown here is the one running Gradle, not necessarily the one compiling your
code — that distinction resolves a lot of confused arguments.

### Proof 1 — see the two phases

```kotlin
// build.gradle.kts
println("CONFIGURATION phase: this line runs on every invocation")

tasks.register("noop") {
    println("CONFIGURATION of task 'noop'")
    doLast { println("EXECUTION of task 'noop'") }
}
```

```bash
./gradlew help
./gradlew noop
```

| What you see | What it means |
|---|---|
| `CONFIGURATION phase` prints for `help` | Correct. Your build script executes on every invocation, even for tasks you did not ask for. |
| `CONFIGURATION of task 'noop'` prints for `help` too | You used `tasks.register` but Gradle still configured it — check whether something forced realisation. With pure lazy registration it should *not* print for `help`. |
| `EXECUTION of task 'noop'` only for `./gradlew noop` | Correct, and it is the whole distinction. |

Now switch `tasks.register` to `tasks.create` and re-run `./gradlew help`.

| What you see | What it means |
|---|---|
| The task's configuration block now prints for `help` | You have just demonstrated eager vs lazy task creation. On a 200-module build, eager creation of thousands of tasks is a measurable chunk of configuration time. |

### Proof 2 — the dependency graph, per configuration

```bash
./gradlew :orderflow-api:dependencies --configuration runtimeClasspath
./gradlew :orderflow-api:dependencies --configuration compileClasspath
```

**What to look for:** an indented tree with annotations after versions.

**Illustration of the annotation shapes, not captured output:**

```
+--- com.vendor:layout-engine:3.0.0 -> 4.1.0
+--- com.vendor:other-lib:2.0.0 (*)
\--- com.orderflow:orderflow-domain:1.0.0-SNAPSHOT (c)
```

| Annotation | What it means |
|---|---|
| `3.0.0 -> 4.1.0` | **Something requested 3.0.0 and 4.1.0 won.** This is Gradle's highest-wins in action, and unlike Maven it tells you in the default output. |
| `(*)` | This subtree was already printed above; omitted for brevity |
| `(c)` | Selected by a **constraint** rather than by a direct request |
| `(n)` | Not resolved (a configuration that is declarable but not resolvable) |
| `FAILED` | Resolution failed — read the message; usually a missing repository or a `strictly` conflict |

**The comparison to make:** run both configurations and count the lines.
`compileClasspath` should be substantially shorter than `runtimeClasspath` in a
well-declared build. If they are identical, you have declared everything `api`
(Trap 2).

### Proof 3 — why is *this* version here?

```bash
./gradlew :orderflow-api:dependencyInsight \
    --dependency jackson-databind \
    --configuration runtimeClasspath
```

**This is the command to learn.** It is strictly better than anything Maven
offers: it prints the selected version, every path that requested it, every
version that was requested, the reason for selection, and any `because` text you
attached to a constraint.

| What you see | What it means |
|---|---|
| `by conflict resolution: between versions X and Y` | Highest-wins fired. Both were requested; the newer won. |
| `by constraint` plus your `because` text | Your constraint decided it. This is why you write `because`. |
| `Was requested: didn't match version X` | Something asked for a version that was rejected — usually a `strictly` or a `reject`. |
| `Selected by rule` | A `resolutionStrategy` rule in your build changed it. Find that rule. |
| `FAILED` with a conflict message | Two `strictly` constraints disagree. Gradle refuses rather than guessing — which is exactly what you want and exactly what Maven will not do. |

Compare that last row with Topic 32's silent mediation. This is a genuine and
real advantage: **Gradle can be configured to fail on a conflict; Maven's default
is to decide quietly.**

### Proof 4 — measure incrementality

```bash
./gradlew clean build                      # cold
./gradlew build                            # warm, nothing changed
touch orderflow-domain/src/main/java/com/orderflow/domain/Order.java
./gradlew build                            # one file changed
```

**What to look for:** the task status suffixes in the output.

| Suffix | Meaning |
|---|---|
| (nothing) | The task executed |
| `UP-TO-DATE` | Inputs unchanged since last run; skipped |
| `FROM-CACHE` | Outputs restored from the build cache without executing |
| `NO-SOURCE` | Nothing to do — no input files |
| `SKIPPED` | Excluded by a predicate or `-x` |

| What you see on the second run | What it means |
|---|---|
| Almost everything `UP-TO-DATE` | Correct incrementality. |
| Tasks re-executing with no changes | A task has **undeclared or volatile inputs** — a timestamp, an absolute path, an env var. Run `./gradlew build --info` and look for the `is not up-to-date because` line naming the input. |

The third run is the interesting one:

| What you see after touching a domain file | What it means |
|---|---|
| Only `orderflow-domain` recompiles, downstream modules are `UP-TO-DATE` | Compile avoidance is working — the change did not affect any ABI those modules see. |
| Everything recompiles | Either the change was to a public signature, or you have declared too much as `api` (Trap 2). |

Then add `org.gradle.caching=true` to `gradle.properties`, run
`./gradlew clean build` twice, and look for `FROM-CACHE`.

### Proof 5 — the configuration cache as a build linter

```bash
./gradlew build --configuration-cache
```

| What you see | What it means | What to do |
|---|---|---|
| `Configuration cache entry stored` | Your build is a pure function of declared inputs. Genuinely good news. | Turn it on permanently. |
| `N problems were found storing the configuration cache` plus an HTML report path | Your build reads ambient state. **This is a hygiene report, not a Gradle bug.** | Open the HTML report; it names the file, line and offending API for each problem. |
| `invocation of 'Task.project' at execution time` | A task action reaches into the project model. | Capture what it needs into a `@Input` property at configuration time. |
| `read system property 'X'` / `read environment variable 'X'` | The build's behaviour depends on the environment. | Use `providers.systemProperty(...)` / `providers.environmentVariable(...)`. |

Run it twice; the second run should say `Reusing configuration cache`. Time both.

### Proof 6 — the task graph without running anything

```bash
./gradlew build --dry-run
```

**What to look for:** every task that *would* run, in order, each marked
`SKIPPED`. This is the closest Gradle analogue to Maven's fixed phase list — the
difference being that Maven's list is the same in every project and Gradle's is
computed per project.

| What you see | What it means |
|---|---|
| A task you do not recognise | A plugin added it. Find out which with `./gradlew help --task <name>`, which prints the task's type and the plugin that contributed it. |
| Tasks from a module you did not expect | Your `include(...)` list or a project dependency is wider than you thought. |

### Proof 7 — write and verify a lockfile

```bash
./gradlew dependencies --write-locks
git add "**/gradle.lockfile" && git status
```

Then change a version in `libs.versions.toml` and, **without** regenerating:

```bash
./gradlew build
```

| What you see | What it means |
|---|---|
| Build fails naming the artifact and both versions | Locking is working. This is the guarantee Maven does not have. |
| Build succeeds | Locking is not enabled for that configuration. Check `lockAllConfigurations()` and that a `gradle.lockfile` exists in that project. |

Regenerate with `--write-locks` and commit the diff. **Reviewing that diff is the
point** — it is a pull-request-visible record of every transitive change, which
is exactly what Topic 32's Exercise 3 Part E was trying to approximate in Maven.

---

## Practice exercises

### 1 — Easy: read a Gradle build without the docs open

Take any open-source Spring Boot project that uses Gradle (Spring Boot's own repo
is a fair, if intimidating, example; a smaller sample project is kinder).

1. From `settings.gradle.kts` alone, list every module and say which are
   applications and which are libraries. Justify each from evidence in the file
   or the module's `plugins { }` block.
2. Find one dependency declared `api` and one declared `implementation`. For
   each, look at the module's public classes and say whether the choice is
   correct. Show the signature that justifies the `api` one.
3. Run `./gradlew :<some-module>:dependencyInsight --dependency <something>
   --configuration runtimeClasspath` and translate the output into plain English:
   which version won, who asked for what, and why.
4. Run `./gradlew help --task test` and report what type the `test` task is and
   which plugin contributed it.

### 2 — Medium: combining earlier topics

Build a two-module Gradle project that exercises Phase 1 and 2 material and makes
the `api`/`implementation` distinction bite.

- `catalog-domain`: a `record Product` (Topic 27), a `sealed interface PriceRule`
  with record variants (Topic 28), and a `Comparator<Product>` chain (Topic 14).
- `catalog-service`: depends on `catalog-domain`, uses a `Stream` pipeline with a
  custom `Collector` (Topics 23, 24) to group products.

Then:

1. Declare `catalog-domain` as `implementation` in `catalog-service`, and expose
   a `public List<Product> topProducts()` method. Build. Record the exact error a
   *third* module gets when it tries to call `topProducts()`.
2. Change it to `api`. Build. Confirm the third module compiles.
3. Now measure compile avoidance. Change the **body** of a private method in
   `catalog-domain` and run `./gradlew build`. Which modules recompiled? Then
   change a **public method signature** and re-run. Which recompiled this time?
   Explain the difference in terms of what Gradle hashes.
4. Add a `strictly` constraint on some library and deliberately introduce a
   conflict. Record the failure message and compare it with what Maven would have
   done in the same situation (Topic 32 — mediation would have picked one
   silently).

### 3 — Hard: production simulation — `orderflow` in Gradle, and the decision memo

**Part A — build it.** Reproduce the four-module `orderflow` skeleton from
Topic 31 in Gradle, using: a version catalog, `platform()` for the Boot BOM,
convention plugins in `build-logic` (no `subprojects { }` anywhere),
`FAIL_ON_PROJECT_REPOS`, dependency locking, and `failOnDynamicVersions()`.

Get `./gradlew build --configuration-cache` passing with zero problems. That last
requirement is the hard part; do not skip it.

**Part B — enforce the architecture.** Make it a **build failure** for
`orderflow-domain` to depend on anything from `org.springframework`, and for
`orderflow-api` to compile against `jakarta.persistence`. Prove both by adding
the forbidden dependency and capturing the failure. Then write one paragraph on
how you would achieve the same thing in Maven, and be honest about how much
worse it is.

**Part C — diff the two builds.** You now have the same project in Maven
(Topic 31) and Gradle. Produce normalised, sorted runtime dependency lists from
both and diff them.

- How many artifacts differ in version?
- For the three biggest differences, use `dependencyInsight` and
  `mvn dependency:tree -Dverbose` to explain *why* each tool chose what it chose.
- Which tool's answer would you rather ship, and does your answer differ per
  artifact?

**Part D — measure, honestly.** With a warm cache and daemon on both sides:

- Clean build wall time, Maven vs Gradle, three runs each, report the median.
- No-op build (`build` twice in a row) wall time for both.
- Single-file-change incremental build for both.

Report the numbers even if they are unflattering to your expectations. Note the
machine, the core count, and whether the Gradle daemon was warm — a cold daemon
makes the first Gradle build look terrible and is not representative.

**Part E — the decision memo.** One page. You are the tech lead. Recommend Maven
or Gradle for `orderflow` as it exists today (one service, four modules, small
team), and separately for `orderflow` as a 40-module platform with six teams in
two years. Use your own numbers from Part D. Name the maintenance owner your
recommendation requires. State the condition under which you would revisit.

This memo is a Phase 12 artefact in miniature. Write it as if a director will
read it.

---

## Interview questions

### Q1 — "When would you choose Gradle over Maven?"

**Mid-level answer:** "Gradle is faster and more flexible. Maven is more
structured. It depends on the team."

**Senior answer:** "I'd frame it as what each tool costs, not which is better.

Gradle is faster for a specific mechanical reason: tasks declare inputs and
outputs, so Gradle can skip work that cannot have changed, restore outputs from a
local or remote build cache, and skip the configuration phase entirely with the
configuration cache. On a large multi-module repo with a shared remote cache
that is a very large win — CI times can drop by more than half. It also gives you
two things Maven simply does not have: `api` versus `implementation`, which
enforces module boundaries and enables compile avoidance, and real dependency
locking, which is the closest thing on the JVM to `package-lock.json`.

What you pay is that the build is a program. Programs drift. I've seen root
builds full of `subprojects { }` blocks conditioned on environment variables,
where a module's real configuration lives in a file the module never references
and CI and local produce different jars. Maven cannot rot that way, because there
is nowhere to put the rot.

So concretely: Gradle for a large monorepo where build time is a team-level cost
and there is someone who owns the build; Gradle if you're on Android, since
there's no choice; Gradle if lockfile guarantees are a hard requirement. Maven for
a single service, a small team, or anywhere the build should be boring and
readable by someone who has never seen it. For `orderflow` today — one service,
four modules — I'd stay on Maven, and I'd revisit at roughly twenty modules or
when CI time crosses about ten minutes."

**What separates them:** naming the *mechanism* behind the speed rather than
asserting it, naming the specific rot pattern, and giving a concrete threshold at
which the decision flips.

**Follow-up:** "Who owns a Gradle build?" They are checking whether you know that
"the team owns it" means nobody owns it, and that a programmable build needs a
named maintainer.

---

### Q2 — "What is the difference between `api` and `implementation`?"

**Mid-level answer:** "`api` exposes the dependency to consumers,
`implementation` hides it."

**Senior answer:** "`api` means the type appears in my module's public
signatures — return types, parameter types, superclasses, thrown checked
exceptions — so consumers need it on their compile classpath to compile against
me. `implementation` means I use it inside method bodies and nobody else needs to
know.

Two payoffs. The obvious one is encapsulation: my persistence module can use
Hibernate as `implementation`, and the API module physically cannot import
`jakarta.persistence` even by accident. Maven's `compile` scope is transitive, so
it cannot express that at all — you'd need a custom enforcer rule or an
ArchUnit test.

The less obvious payoff is compile avoidance, and it's the bigger one on a large
repo. When an `implementation` dependency changes, Gradle knows consumers'
compilation cannot be affected and does not recompile them. Declaring everything
`api` — which is what people do when they migrate from Maven without thinking —
throws that away, and the build gets slow in a way that looks like Gradle being
bad rather than the build being wrong.

The rule I use in review: if you can't point at the public signature the type
appears in, it's `implementation`."

**What separates them:** the compile-avoidance consequence, the observation that
Maven cannot express this, and having a mechanical review rule.

**Follow-up:** "What breaks if you declare something `implementation` that should
be `api`?" A consumer gets a compile error like `cannot access <Type>` when
calling a method whose signature mentions it. Good answers note that this is a
*loud* failure, which is why erring toward `implementation` is the safer default.

---

### Q3 — "Your team's Gradle build takes 12 minutes on CI. Where do you look?"

**Mid-level answer:** "Enable the build cache and parallel builds."

**Senior answer:** "Measure before changing anything. `--profile` gives a
per-task breakdown and separates configuration time from execution time; a build
scan gives the same plus cache hit rates and the critical path.

Then it splits by what the numbers say. If configuration time is large — say
more than fifteen or twenty percent — the causes are eager task creation with
`tasks.create` instead of `register`, and `subprojects { }` blocks that configure
everything on every invocation; the fix is convention plugins plus the
configuration cache. If execution time dominates, I look at cache hit rate
first: a remote build cache shared with CI turns most of a clean build into
`FROM-CACHE`, and if the hit rate is low the usual cause is tasks with volatile
inputs — absolute paths, timestamps, an env var — which `--info` will name for
you as 'is not up-to-date because'. Then the critical path: if everything waits
on one module, that's a module-graph problem, and often it's too much declared as
`api`, so a change in one place invalidates half the repo.

The order matters: caching a badly-structured build gives you a fast wrong
answer. I'd fix input hygiene, then turn on caching, then parallelism."

**What separates them:** measuring first, knowing which lever addresses which
symptom, and the point that caching a build with volatile inputs makes things
worse rather than better.

**Follow-up:** "How do you know the cache is actually being hit?" `--scan`, or
counting `FROM-CACHE` in the output, or the build cache's own hit-rate metrics.
Guessing does not count.

---

### Q4 — "You migrate a Maven build to Gradle. The build is green. Is it safe to ship?"

**Mid-level answer:** "If the tests pass, it should be fine."

**Senior answer:** "No, and the reason is dependency resolution. Maven resolves
conflicts by nearest-wins — shallowest depth, ties by declaration order. Gradle
resolves by highest-wins. So for any artifact where a deeper node requested a
newer version than a shallower one, the two tools select **different versions**,
and on a Spring project that will be dozens of artifacts.

A green build only tells me the compile classpath is internally consistent. It
tells me nothing about behavioural differences from a library version change —
a different JSON date format, a changed HTTP client default, a Hibernate dialect
tweak. And the loud failures show up as `AbstractMethodError` or
`IncompatibleClassChangeError` at runtime rather than at compile time.

What I'd do before shipping: dump `mvn dependency:list -DincludeScope=runtime`
and `./gradlew dependencies --configuration runtimeClasspath`, normalise both to
`group:artifact:version`, sort and diff. Every line in that diff is an untested
change. Then pin the ones that matter with constraints — with a `because` string
saying why — enable dependency locking, commit the lockfiles, and run the load
test from our baseline before and after. I'd also byte-compare the produced jars
if we have reproducible builds set up."

**What separates them:** knowing the two resolution rules differ, naming the
specific diff procedure, and treating a version delta as an untested change
rather than a detail.

**Follow-up:** "How would you make that diff a permanent CI check?" Dependency
locking with the lockfile in review, or a CI job that diffs the resolved graph
against the base branch and comments on the PR.

---

### Q5 — "What is the Gradle configuration cache and why should I care if my build is already fast enough?"

**Mid-level answer:** "It caches the configuration phase to make builds start
faster."

**Senior answer:** "Mechanically, yes — it serialises the configured task graph
after the configuration phase and reuses it, so subsequent invocations skip
running your build scripts entirely.

But the reason I'd turn it on even if I didn't need the speed is that it is the
best build-hygiene linter available. It can only work if a task's behaviour is
fully determined by its declared inputs. So it fails on exactly the patterns that
make a build unreliable: reading an environment variable at configuration time,
touching `Task.project` at execution time, capturing mutable state in a task
action. Those are the same patterns that break incremental builds, poison the
build cache with wrong `UP-TO-DATE` results, and stop your artifacts being
reproducible.

So I treat a configuration-cache failure as a true report about my build, not an
incompatibility to work around. The HTML report names the file, the line and the
API. Fixing them is the same work as making the build reproducible for
Topic-34-style supply-chain requirements, so it pays twice."

**What separates them:** reframing a performance feature as a correctness tool,
and connecting it to reproducibility and build-cache correctness.

**Follow-up:** "Give an example of a task with undeclared inputs and what goes
wrong." Something reading a file it did not declare: Gradle marks it
`UP-TO-DATE`, the file changed, and the output is stale — a wrong build that
looks like a successful one.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Maven's rigidity is what stops a Maven build rotting the way Trap 1 describes.
   Design a hypothetical build tool that has Gradle's speed and Maven's
   un-rottability. What would you have to forbid? Would anyone use it?

2. Gradle picks the highest version; Maven picks the nearest. Construct a
   concrete scenario where Maven's answer is clearly better, and one where
   Gradle's is. What does that tell you about "which default is right"?

3. `api` versus `implementation` is a design decision the build tool forces you
   to make. Name another design decision you would like a build tool to force,
   and say why nobody has built it.

4. The configuration cache fails on builds that read environment variables. Some
   real builds genuinely need to behave differently on CI. How do you reconcile
   those two facts without weakening the guarantee?

5. Gradle's toolchains can download a JDK. Maven uses whatever JVM it is running
   on. Argue that Gradle's behaviour is a supply-chain *risk* rather than a
   convenience — then argue the opposite.

6. Dependency locking gives you reproducible resolution. Does it give you a
   reproducible *build*? Name at least two things that can still differ. (This
   is the bridge into Topic 34.)

7. You inherit a 60-module Gradle build with a 900-line root `build.gradle.kts`
   and no convention plugins. You have two weeks. What do you do first, and what
   do you deliberately not do?

---

## Quick reference card

### Commands

```bash
./gradlew build                                  # compile + test + assemble
./gradlew build -x test                          # exclude a task
./gradlew :orderflow-api:build                   # one project
./gradlew clean build --no-build-cache           # honest cold timing

./gradlew dependencies --configuration runtimeClasspath
./gradlew dependencyInsight --dependency jackson-databind \
          --configuration runtimeClasspath       # THE debugging command
./gradlew dependencies --write-locks             # generate/update gradle.lockfile

./gradlew build --dry-run                        # task graph, nothing executed
./gradlew help --task test                       # what is this task, from which plugin
./gradlew --profile build                        # per-task timing report
./gradlew build --scan                           # full report (published externally - get consent)
./gradlew build --configuration-cache            # hygiene linter
./gradlew build --info                           # "is not up-to-date because ..."
./gradlew --status ; ./gradlew --stop            # daemon management
./gradlew wrapper --gradle-version <v>           # change the pinned Gradle version
```

### Configuration ↔ Maven scope

| Gradle | Maven | Leaks to consumers? |
|---|---|---|
| `api` | `compile` | **yes** |
| `implementation` | `compile` | **no** |
| `compileOnly` | `provided` | no |
| `runtimeOnly` | `runtime` | runtime only |
| `testImplementation` | `test` | no |
| `annotationProcessor` | (plugin config) | no |

### Task status suffixes

| Suffix | Meaning |
|---|---|
| *(none)* | executed |
| `UP-TO-DATE` | inputs unchanged, skipped |
| `FROM-CACHE` | outputs restored from the build cache |
| `NO-SOURCE` | no inputs to work on |
| `SKIPPED` | excluded by `-x` or a predicate |

### `gradle.properties` starter set

```properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=768m
```

### Gotchas checklist

- [ ] `useJUnitPlatform()` or your tests silently do not run.
- [ ] `tasks.register`, not `tasks.create`.
- [ ] Convention plugins in `build-logic`, never `subprojects { }`.
- [ ] `implementation` by default; `api` only for public signatures.
- [ ] Gradle resolves **highest**, Maven resolves **nearest**. Diff before trusting a migration.
- [ ] Commit the wrapper, including a verified `gradle-wrapper.jar` or a `distributionSha256Sum`.
- [ ] No dynamic versions, no SNAPSHOTs in a release. `failOnDynamicVersions()`.
- [ ] `FAIL_ON_PROJECT_REPOS` so no module can add its own repository.
- [ ] `because("...")` on every constraint — the next engineer needs the reason.
- [ ] Configuration-cache problems are a report about your build, not a Gradle bug.

---

## When would I use this at work?

**1. Joining a team that already uses Gradle.**
You will not get to choose. What you will do on week one is read a
`build.gradle.kts` and work out where a dependency came from. `dependencyInsight`
plus knowing the configuration-vs-execution split gets you productive in a day
instead of a fortnight.

**2. The build-time conversation, which happens at every growing company.**
CI takes fourteen minutes and everyone is annoyed. Somebody proposes migrating to
Gradle. The senior move is to ask what the fourteen minutes is actually spent on
before agreeing — often it is a serial test suite, or no dependency caching in
CI, or a single module everything waits on, none of which a tool migration fixes.
When Gradle genuinely is the answer, you say so with the numbers.

**3. Enforcing an architectural boundary that keeps getting violated.**
"The API module keeps importing Hibernate types" is a losing code-review battle.
In Gradle it is a two-line change to `implementation` and the compiler enforces it
forever. Being the person who converts a recurring argument into a build failure
is a disproportionately valued skill.

---

## Connected topics

**Prerequisites:**
- **31 — Maven fundamentals.** You cannot evaluate the alternative without
  knowing exactly what Maven's rigidity buys and costs.
- **32 — Dependency resolution.** The flat-classpath and one-version-wins facts
  carry over unchanged; only the selection rule differs. The `NoSuchMethodError`
  diagnosis is identical.
- **21–24 — lambdas, method references, streams.** The Kotlin DSL leans on
  lambdas-with-receiver, and that reads much more easily once configuring-by-lambda
  is a familiar shape.

**This unlocks:**
- **34 — Supply chain.** Reproducible archives
  (`isPreserveFileTimestamps = false`), dependency locking, dependency
  verification with `gradle/verification-metadata.xml`, and the CycloneDX Gradle
  plugin all live here. Gradle is meaningfully ahead of Maven on this axis and
  that is worth knowing before you pick a tool.
- **42 — Auto-configuration mechanics.** Which
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
  files Boot finds is a function of the runtime classpath — and in Gradle the
  runtime classpath and the compile classpath genuinely differ, thanks to
  `implementation`. Something can be auto-configured that you cannot compile
  against. That surprises people once.
- **61 — Testcontainers.** `testImplementation` keeps Docker-dependent code off
  every classpath that ships.
- **83 — GraalVM native image.** The native-build-tools Gradle plugin, and
  `graalvmNative { }` configuration; closed-world analysis over the resolved
  runtime classpath.
- **122 — Layered Docker jars.** `bootJar { layered { } }` is the Gradle side of
  Boot's layering, and Gradle's remote build cache is what makes rebuilding those
  layers on CI cheap.
- **127 — Migration planning.** A build-tool migration is a migration like any
  other: sequence it by risk, keep it independently revertible, and prove
  equivalence with a resolved-graph diff rather than a green build.

---

*Java baseline 21, running on JDK 25. Target Spring Boot 4.1 / Framework 7.0 /
Jakarta EE 11. Gradle's version numbers move quickly and its APIs deprecate on a
real schedule, so this document names no Gradle version and no library version —
run `gradle --version`, read your project's wrapper properties, and check
`libs.versions.toml` against a freshly generated starter project. The
configuration/execution split, the task graph, and the `api`/`implementation`
distinction are the parts that will still be true in ten years.*
