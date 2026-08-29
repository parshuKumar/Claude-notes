# 34 — Supply Chain: Reproducible Builds, SBOM, CVE Triage

## Phase: 3 — Build & Supply Chain
## Category: CORE
## Java baseline: 21  |  Notes features from: 21 (runtime JDK 25)
## Project spine: `orderflow` gains a reproducible build, an SBOM generated on every build, a CVE scan wired into CI, and a documented triage procedure with expiring exceptions. Everything from Topic 35 onward inherits it, and Topic 122 ships it in a container.

---

## ELI5 anchor

Three questions a food factory has to answer, and your service has exactly the
same three.

**1. "What is in this box?"**
That is an **SBOM** — a Software Bill of Materials. An ingredients list for the
artifact you shipped. Not what you *think* is in it; what is actually in it.

**2. "The supplier just recalled a batch of ingredient X. Are we affected?"**
That is **CVE triage**. And the honest answer is almost never a simple yes or no.
"We have ingredient X in the warehouse" is not the same as "ingredient X is in
the recipe" is not the same as "the recalled batch is in the recipe" is not the
same as "customers ate it".

**3. "If we made this box again from the same recipe, would we get the same box?"**
That is a **reproducible build**. If the answer is no, then question 1's answer
is a guess, because you cannot prove the box you shipped came from the recipe you
have.

The whole topic is those three questions and the discipline of answering them
before someone asks you at 2am.

---

## The bridge from what you know

### `npm audit` → dependency scanners: **HONEST ANALOGUE**

This is one of the few places where your Node intuition transfers directly.

```bash
npm audit
npm audit --production
npm audit fix
```

```bash
mvn org.owasp:dependency-check-maven:check
./gradlew dependencyCheckAnalyze
trivy fs .
grype dir:.
```

Same job: match your dependency graph against a vulnerability database, report
what matches. Same strengths and — this matters — **the same fundamental
weakness**, which you have probably already felt in Node:

> **A scanner reports *presence*, not *exploitability*.**

You have almost certainly run `npm audit` and seen forty "high severity"
findings, most of them in a dev-only transitive dependency of a build tool, on a
code path nothing in your app ever calls. You learned to distrust the number. Keep
that instinct. It is exactly right, and it is the entire content of the triage
section below.

### What is *better* in Java than in npm

Two structural advantages worth knowing, because they change what you have to
worry about:

| | npm | Maven Central |
|---|---|---|
| Namespace ownership | anyone can publish `left-pad-2` | `groupId` requires proving control of the matching domain (or a verified namespace) |
| Typosquatting risk | high — package names are first-come | much lower, because the namespace is verified |
| Artifact immutability | a version can be unpublished (with limits) | released versions on Central are **immutable and never removed** |
| Signing | optional, rarely enforced | **PGP signature required** for publication to Central |
| Install-time code execution | `postinstall` scripts run arbitrary code | **no equivalent.** Adding a Maven dependency executes nothing. |

That last row is a big deal and worth saying out loud in an interview: **adding a
dependency to a POM does not run any of that dependency's code.** There is no
`postinstall`. The attack surface is "code you eventually call", not "code that
runs the moment you install". Maven *plugins* are a different story — those do
execute during your build — which is why a plugin from an unknown source is a
much bigger risk than a library from one.

### What is *worse* in Java than in npm

| | npm | Maven |
|---|---|---|
| Lockfile | `package-lock.json` with integrity hashes, by default | none by default (Topic 32) |
| "What actually shipped" | the lockfile records it | you need an SBOM, which you must set up |
| Dependency verification | integrity hashes checked on install | Maven has no built-in checksum-pinning; Gradle does (`verification-metadata.xml`) |
| Fat jars | not a thing | a Boot jar nests other jars inside it, and naive scanners miss them |

That last one bites people. Your deployed artifact is one file with several
hundred jars nested inside `BOOT-INF/lib/`. A scanner that only looks at
top-level files sees one jar and reports nothing. Trap 5.

### The translation table

| You know | Java | Verdict |
|---|---|---|
| `npm audit` | OWASP dependency-check / Snyk / Trivy / Grype | **HONEST ANALOGUE** |
| `npm audit fix` | `dependencyManagement` override, or a BOM bump | **PARTIAL** — no automatic fix; the override is a deliberate act |
| `package-lock.json` integrity hashes | Gradle dependency verification; nothing built-in for Maven | **PARTIAL** |
| `npm ls` to see what shipped | SBOM (CycloneDX / SPDX) | **PARTIAL** — an SBOM is a standard document, not a CLI listing |
| `postinstall` scripts | nothing for dependencies; Maven **plugins** do execute | **NO ANALOGUE** for libraries; plugins are the real analogue |
| Dependabot / Renovate | Dependabot / Renovate (both support Maven and Gradle) | **HONEST ANALOGUE** |

---

## What is this?

Three connected practices.

### 1. Reproducible builds

A build is **reproducible** if building the same source, with the same declared
toolchain, produces a **byte-identical** artifact.

Java builds are non-reproducible by default. The usual causes:

| Cause | Why it breaks byte-identity |
|---|---|
| Jar entry timestamps | The jar records the build time for every entry |
| Jar entry **order** | Filesystem iteration order differs between machines |
| `MANIFEST.MF` fields | `Build-Jdk`, `Built-By`, `Build-Time` embed the environment |
| Absolute paths | Debug info and generated files can embed `/home/alice/...` |
| Generated code with a date | Many code generators stamp a timestamp in a comment |
| Different JDK | A different `javac` can emit different bytecode |
| Locale / timezone / charset | Sorting and encoding of generated resources |

**Why anyone cares:** if you cannot rebuild the artifact bit for bit, you cannot
prove that the jar running in production was built from the commit you reviewed.
That matters for an incident ("is the running code the code we audited?"), for a
compromised build server (the only detection is a mismatch between an independent
rebuild and the published artifact), and increasingly for regulation.

### 2. SBOM

A **Software Bill of Materials** is a machine-readable list of every component in
an artifact, with versions and identifiers. Two standard formats:

- **CycloneDX** — designed for security use cases. The common choice on the JVM;
  there are good Maven and Gradle plugins, and Spring Boot can expose one at
  runtime.
- **SPDX** — originated in licence compliance, broader scope, also widely
  supported.

Each component carries a **purl** (package URL), a canonical identifier:

```
pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.17.2
```

That string is how a scanner matches your component against a vulnerability
database without guessing.

**What an SBOM is actually for**, in order of how often you will use it:

1. **Answering "are we affected?" in minutes rather than days.** When a big CVE
   lands, the org-wide question is "which of our 200 services contain this?" With
   SBOMs stored per release, that is a query. Without them, it is 200 engineers
   running `mvn dependency:tree`.
2. **Recording exceptions.** A **VEX** document (Vulnerability Exploitability
   eXchange) attaches "we are not affected, and here is why" to a specific CVE
   and a specific product version. That is the artefact your triage produces.
3. **Licence compliance**, which is usually somebody else's problem until it
   suddenly is yours.
4. **Customer and regulatory requirements.** Enterprise procurement increasingly
   asks for one.

### 3. CVE triage

A **CVE** (Common Vulnerabilities and Exposures) identifier names a specific
vulnerability. A scanner tells you a CVE's affected component is present in your
graph.

**That is all it tells you.** Turning that into a decision is triage, and it is
the part that separates a senior engineer from a ticket queue.

---

## Why does it matter?

**Because the naive response to a scanner is actively harmful.**

A team that treats scanner output as a work queue does this: 40 findings appear,
someone bumps 40 versions in an afternoon, three of the bumps are major versions
with breaking changes, one breaks in production, and the team learns that
"security work causes outages". Meanwhile the *one* finding that was genuinely
reachable from an unauthenticated endpoint got the same fifteen minutes as the
other 39.

**Because the opposite failure is worse.** A team that suppresses everything with
no expiry ships a genuinely exploitable, actively-exploited vulnerability for two
years, with a suppression file entry pointing at a closed ticket nobody
remembers.

**Because running an unsupported framework line is itself the risk.**

> **[BOOT 3.x DELTA] — read this one carefully, it is the most consequential**
> **[BOOT 3.x DELTA] in the entire curriculum.**
>
> Spring Boot 3.5 left OSS support in June 2026. That is not a style preference.
> It means: **when a CVE lands in a Boot-managed artifact, there is no free patch
> release.** Your options become (a) commercial support, (b) override the managed
> version yourself in your own `dependencyManagement` and accept that you are off
> Boot's tested combination permanently, or (c) upgrade the framework under
> incident pressure, which is the worst possible time.
>
> The second consequence: Boot 4 modularised into many smaller jars, so the
> artifact carrying a given feature may have **moved** between 3.x and 4.x. When
> you look up "which Boot version fixes CVE-X", the fix may be in an artifact
> whose coordinates changed. Always confirm the fixed artifact is actually on
> your classpath with `mvn dependency:tree -Dincludes=<groupId>:<artifactId>`
> after applying an override — a `dependencyManagement` entry for an artifact
> that is not in your graph does nothing at all, silently.
>
> When someone asks "why upgrade, it works fine", this is the answer: staying on
> an EOL line converts every future CVE from a version bump into a project.

---

## Syntax breakdown

### Reproducible builds — Maven

```xml
<properties>
  <!-- The standard reproducible-builds property. Maven's archiver plugins
       read it and use this fixed timestamp for every jar entry instead of
       "now". Any fixed ISO-8601 instant works; convention is to use the
       commit/release date. -->
  <project.build.outputTimestamp>2026-08-29T00:00:00Z</project.build.outputTimestamp>
</properties>
```

That single property does most of the work: it fixes entry timestamps and, in
recent plugin versions, normalises entry ordering. It is deliberately a
*property* rather than plugin config, so it applies to every archiver.

In CI, set it from the commit date rather than hardcoding:

```bash
mvn -B verify -Dproject.build.outputTimestamp="$(git log -1 --format=%cI)"
```

Also worth doing:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <configuration>
    <archive>
      <!-- Build-Jdk-Spec and Created-By still vary; strip what you can. -->
      <manifest>
        <addDefaultImplementationEntries>false</addDefaultImplementationEntries>
      </manifest>
    </archive>
  </configuration>
</plugin>
```

The `maven-artifact-plugin` provides goals to check and compare a build for
reproducibility (a build-plan check and a rebuild-and-compare). Check the current
goal names and plugin coordinates on the Apache Maven site before wiring it in —
I am not quoting them from memory.

### Reproducible builds — Gradle

```kotlin
tasks.withType<AbstractArchiveTask>().configureEach {
    isPreserveFileTimestamps = false
    isReproducibleFileOrder = true
}
```

Two properties, every archive task. This is genuinely simpler than the Maven
side.

### Generating an SBOM — Maven

```bash
# Aggregate across a multi-module build. Run from the root.
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom
```

Wired into the build:

```xml
<plugin>
  <groupId>org.cyclonedx</groupId>
  <artifactId>cyclonedx-maven-plugin</artifactId>
  <!-- Check the current version:
       mvn versions:display-plugin-updates
       I am not writing a plugin version from memory. -->
  <version>SET-ME</version>
  <executions>
    <execution>
      <id>sbom</id>
      <phase>package</phase>
      <goals><goal>makeAggregateBom</goal></goals>
    </execution>
  </executions>
  <configuration>
    <!-- Only what ships. test-scoped components are not in the artifact. -->
    <includeCompileScope>true</includeCompileScope>
    <includeRuntimeScope>true</includeRuntimeScope>
    <includeProvidedScope>false</includeProvidedScope>
    <includeTestScope>false</includeTestScope>
  </configuration>
</plugin>
```

**That scope configuration is the whole reason Topic 31 mattered.** An SBOM that
includes test-scoped components describes a thing you did not ship, and every
CVE in JUnit's transitive graph becomes a finding you have to explain. An SBOM
that excludes `provided` is right for a fat jar and *wrong* for a war deployed to
a container — because there, the container really does supply those jars and they
really are in the running system, just not in your artifact. Know which one you
are producing.

### Generating an SBOM — Gradle

```kotlin
plugins {
    id("org.cyclonedx.bom") version "SET-ME"
}
```

```bash
./gradlew cyclonedxBom
```

### Spring Boot's runtime SBOM

Boot can expose an SBOM through the actuator when one is on the classpath:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,sbom
```

```bash
curl -s localhost:8080/actuator/sbom
curl -s localhost:8080/actuator/sbom/application
```

**Why this is better than the file in your build output:** it is served by the
*running process*. It tells you what the thing actually in production contains,
not what the last build produced. If those two disagree, you have discovered
something important. Do not expose this endpoint publicly — it is an inventory of
your attack surface, handed to whoever asks.

### Scanning — Maven

```bash
mvn org.owasp:dependency-check-maven:check
```

Wired in:

```xml
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>SET-ME</version>
  <configuration>
    <failBuildOnCVSS>7</failBuildOnCVSS>
    <suppressionFiles>
      <suppressionFile>${project.basedir}/security/suppressions.xml</suppressionFile>
    </suppressionFiles>
    <skipTestScope>true</skipTestScope>
  </configuration>
</plugin>
```

**Two operational facts that will surprise you on first run:**

1. **The first run is slow** — it downloads a local copy of the vulnerability
   database. Cache that directory in CI or every build pays the cost.
2. **Recent versions require an NVD API key** to fetch data at a usable rate.
   Without one you will hit rate limits and see a very long, apparently stuck
   build. Get a key, put it in a CI secret. Check the plugin's current
   documentation for the exact parameter name.

Also true and worth expecting: dependency-check matches components to
vulnerabilities partly by heuristic name/version matching, so **it has a real
false-positive rate**. That is not a reason to distrust it; it is a reason to
triage rather than to auto-bump.

### The suppression file — and the `until` attribute

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">

  <suppress until="2026-11-30Z">
    <notes><![CDATA[
      CVE-2026-XXXXX in layout-engine 4.1.0.
      TRIAGE (2026-08-29, @parshuram, ticket SEC-412):
        - Vulnerable class com.vendor.layout.XmlTemplateLoader IS on the classpath.
        - It is only reached via LayoutEngine.loadFromXml(), which orderflow
          never calls: we use loadFromJson() exclusively (verified by grep +
          call-graph inspection).
        - No attacker-controlled input reaches any layout loader; templates are
          bundled resources.
      DECISION: not exploitable in orderflow. Suppressed until vendor 4.2.0 ships.
      REVIEW: this suppression EXPIRES 2026-11-30. Re-triage then.
    ]]></notes>
    <packageUrl regex="true">^pkg:maven/com\.vendor/layout-engine@.*$</packageUrl>
    <cve>CVE-2026-XXXXX</cve>
  </suppress>

</suppressions>
```

**`until` is the most important attribute in the file.** A suppression without an
expiry is a permanent, invisible decision made by someone who has probably left.
With an expiry, the finding comes back and someone re-decides with current
information.

The `<notes>` block is not documentation ceremony. It is the artefact that makes
the decision reviewable: *which* class, *which* call path, *whether* input is
attacker-controlled, *who* decided, *when*, and *what would change the answer*.
If your notes say "not applicable" and nothing else, you have suppressed a finding
without triaging it.

### Gradle dependency verification — checksums and signatures

Gradle has something Maven does not:

```bash
./gradlew --write-verification-metadata sha256,pgp build
```

This generates `gradle/verification-metadata.xml` containing a checksum and/or
trusted PGP key for every artifact. Subsequent builds **fail** if any artifact's
bytes differ from what was recorded. That is npm's integrity-hash guarantee,
applied to the JVM.

Maven has no built-in equivalent. There are third-party plugins that verify PGP
signatures during a build; check current coordinates rather than trusting a name
from memory. The more common Maven answer is a **repository manager**
(Nexus, Artifactory) that mirrors, caches and policy-checks everything, which
also solves the next problem.

### Dependency confusion — the attack, and the fix

The attack: your org uses an internal groupId, say `com.orderflow.internal`, on a
private repository. An attacker publishes an artifact with those exact
coordinates to Maven Central at a very high version number. A build configured to
consult both repositories may prefer the public one.

The fix is repository routing, not vigilance:

```xml
<!-- ~/.m2/settings.xml or the CI settings -->
<mirrors>
  <mirror>
    <id>corp-repo</id>
    <mirrorOf>*</mirrorOf>          <!-- EVERYTHING goes through the manager -->
    <url>https://nexus.internal.example.com/repository/maven-public/</url>
  </mirror>
</mirrors>
```

Then configure the repository manager so that `com.orderflow.*` is served only
from the internal hosted repository and is **never** proxied to Central. In
Gradle the equivalent is `exclusiveContent` / `content { includeGroup(...) }`
filters on repository declarations, plus `FAIL_ON_PROJECT_REPOS` (Topic 33) so no
module can add a repository of its own.

---

## Example 1 — minimal

The three commands, on any project, in the order you would run them the first
time you are asked "is our service affected?".

```bash
# 1. What is actually here, and at what version?
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-databind

# 2. Produce the ingredients list.
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom
ls -la target/bom.json target/bom.xml

# 3. Match it against the vulnerability database.
mvn org.owasp:dependency-check-maven:check
open target/dependency-check-report.html
```

Notice the order. **You establish which version is on the classpath before you
look at any scanner output**, because a scanner reporting a version that
mediation did not select is a common and very confusing false lead. Topic 32 is
step one of every supply-chain investigation.

---

## Example 2 — production scenario: a Friday afternoon CVE in `orderflow`

The situation, which is entirely typical. It is 4pm. Security posts:

> "Critical RCE in `layout-engine`, CVSS 9.8, CVE-2026-XXXXX. All services using
> it must patch before Monday."

`orderflow-api` uses `layout-engine` transitively through the invoice-renderer
library from Topic 32. There are four modules and two deployables.

The wrong response is to bump the version, run the tests, and deploy on a Friday.
Here is the triage, as a procedure.

### Step 1 — is it in the thing we actually ship?

```bash
mvn dependency:tree -Dverbose -Dincludes=com.vendor:layout-engine
mvn dependency:list -DincludeScope=runtime | grep layout-engine
```

| What you find | What it means | Next step |
|---|---|---|
| Not present at all | You are not affected. Record it and move on. | Done — but check the *other* modules too. |
| Present only at `test` scope | It is not in the shipped artifact. | Low priority. Fix at leisure; note that CI itself is still running it. |
| Present at `provided` scope | Not in your jar, but present in the deployment container. **You are still affected** — the platform team owns the fix. | Escalate to whoever owns the container. |
| Present at `compile`/`runtime` scope | It ships. Continue triage. | Step 2. |

**Also check the deployed artifact itself, not just the POM:**

```bash
unzip -l orderflow-api/target/orderflow-api-1.0.0-SNAPSHOT.jar | grep -i layout-engine
```

A Boot fat jar nests its dependencies under `BOOT-INF/lib/`. If the jar and the
POM disagree, believe the jar.

### Step 2 — which version does resolution actually select?

The advisory says "fixed in 4.2.0". Your POM says 3.0.0. But Topic 32 taught you
that the POM is not the classpath.

```bash
mvn dependency:tree -Dverbose -Dincludes=com.vendor:layout-engine
```

| What you see | What it means |
|---|---|
| `4.1.0` won, `3.0.0` omitted for conflict | You are on 4.1.0, not 3.0.0. **Re-read the advisory against 4.1.0** — it may already be fixed, or the affected range may be different. |
| `3.0.0` won, `4.1.0` omitted for conflict | The dangerous direction (Topic 32). You are on the vulnerable one, and something in your graph expects the newer API. |
| Two different `groupId`s carrying the same classes | A relocation. You may have two copies; fixing one is not enough. |

This step alone resolves a meaningful share of findings. Scanners frequently
report the version declared somewhere in the graph rather than the version
mediation selected.

### Step 3 — is the vulnerable code even present?

```bash
mvn dependency:copy-dependencies -DoutputDirectory=target/deps -DincludeScope=runtime
unzip -l target/deps/layout-engine-4.1.0.jar | grep -i 'XmlTemplateLoader'
```

| What you find | What it means |
|---|---|
| The vulnerable class is not in the jar | Not exploitable through this artifact. Many libraries ship the vulnerable code in a *different* artifact of the same family. |
| It is present | Continue to Step 4. |

### Step 4 — is it reachable from our entry points?

This is the step that most teams skip, and it is the one that turns a scanner
into an engineering tool.

```bash
# Do we call it directly anywhere?
grep -rn "XmlTemplateLoader\|loadFromXml" --include=*.java .

# What does OUR code actually depend on, statically?
jdeps --multi-release 21 -verbose:class \
      -cp "$(cat cp.txt)" orderflow-api/target/classes | grep -i layout
```

Questions to answer, in order:

1. Does `orderflow` call the vulnerable API directly? (grep)
2. If not, does a library we call reach it? (`jdeps`, or a reachability-aware
   commercial scanner)
3. Is the input to that path **attacker-controlled**? A deserialisation flaw
   reachable only from a hardcoded bundled resource is a very different risk from
   one reachable from an HTTP request body.
4. Is there a **configuration mitigation**? Many CVEs have one — disable a
   feature, set a system property, restrict a parser. That can buy you the
   weekend.

Be honest about the limits: `jdeps` sees static references. It does not see
reflection, service loading, dynamic proxies, or Spring's own wiring. Java is
full of all four. **Reachability analysis reduces uncertainty; it does not
eliminate it.** Say that out loud rather than claiming certainty you do not have.

### Step 5 — decide, and record the decision

Four possible outcomes. Each is legitimate; the mistake is having only one.

| Decision | When | How |
|---|---|---|
| **Bump** | The fix is a patch version, reachable or plausibly reachable | `<dependencyManagement>` entry in the **parent** POM, so all four modules move together |
| **Mitigate by config** | A documented mitigation exists and the upgrade is risky today | Apply it, ship it, schedule the upgrade |
| **Accept with expiry** | Demonstrably not reachable, or not attacker-reachable | Suppression with `until`, full triage notes, named owner |
| **Escalate** | It is in a `provided` dependency, a base image, or the framework itself | It is not your fix to make; make sure it has an owner |

The fix, in the parent POM, looks like this — and note that it goes **above** the
BOM import (Topic 32, Trap 5):

```xml
<dependencyManagement>
  <dependencies>

    <!-- SECURITY OVERRIDE
         CVE-2026-XXXXX (CVSS 9.8, RCE) in layout-engine < 4.2.0.
         Ticket: SEC-412. Applied 2026-08-29 by @parshuram.
         REMOVE THIS when invoice-renderer 3.3.0 (which depends on 4.2.0)
         is released - then this override becomes a no-op we should delete.
         Verify with: mvn dependency:tree -Dincludes=com.vendor:layout-engine -->
    <dependency>
      <groupId>com.vendor</groupId>
      <artifactId>layout-engine</artifactId>
      <version>4.2.0</version>
    </dependency>

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

### Step 6 — prove the fix, do not assume it

```bash
mvn dependency:tree -Dincludes=com.vendor:layout-engine     # 4.2.0 must WIN
mvn -q clean verify                                          # tests still pass
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom    # regenerate the SBOM
mvn org.owasp:dependency-check-maven:check                   # finding must be GONE
unzip -l orderflow-api/target/*.jar | grep layout-engine     # the JAR, not the POM
```

**The last command is the one people skip and it is the one that catches
mistakes.** A `dependencyManagement` entry for an artifact that is not in your
graph does nothing and reports nothing (the Boot 4 modularisation delta above
makes this a live risk). The jar listing is ground truth.

### Step 7 — the part that makes it not happen again

- The SBOM is generated on every release build and stored **with the release
  artifact**, not just in `target/`. Next time, the org-wide question is a query.
- CI runs the scanner with `failBuildOnCVSS` set, and a scheduled nightly job
  re-scans the **released** version — because a CVE published tomorrow affects
  the artifact you shipped today, and nothing in your PR pipeline will notice.
- Every suppression has an `until` date and a CI job that fails when one expires.
- Dependabot or Renovate raises small, routine bumps continuously, so you are
  never a year behind when an emergency arrives. **Being current is the cheapest
  security control you have.**

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — treating scanner output as a work queue

**Wrong:** the scanner reports 43 findings. A ticket is created per finding. An
engineer works down the list bumping versions.

**Exact symptoms:**

- A major-version bump lands among the patch bumps. The build stays green — the
  API is source-compatible — and a behaviour change ships. Two weeks later a
  downstream consumer reports malformed timestamps.
- Another bump pulls a newer transitive that wins mediation and produces a
  `NoSuchMethodError` in an unrelated feature (Topic 32).
- The backlog never empties, because the database grows faster than the team
  bumps.
- Six weeks later, someone notices the one finding that was reachable from an
  unauthenticated endpoint is still open, because it was the 31st ticket.

**Root cause:** the scanner answers "is this component present?" You need the
answer to "can an attacker exploit this in our system?" Those are different
questions and the first is a poor proxy for the second. Ordering work by CVSS
sorts by *severity if exploited*, not by *probability of being exploited here*.

**Fix — a triage funnel, applied in this order:**

1. **Is it in a shipped artifact?** (scope, and the jar listing) — removes
   test-only findings immediately.
2. **Which version actually resolved?** (`dependency:tree`) — removes stale and
   mismatched reports.
3. **Is the vulnerable class present in the jar?**
4. **Is the vulnerable method reachable from our entry points?**
5. **Is the input attacker-controlled?**
6. **Only now**, rank what survives — by reachability first, then by
   **CISA KEV** (is it *known to be actively exploited*?) and **EPSS** (estimated
   probability of exploitation), and only then by CVSS.

CVSS 9.8 on an unreachable code path is lower priority than CVSS 6.5 on the
request-handling path. Being able to say that sentence, and defend it, is the
senior skill in this topic.

---

### Trap 2 — a suppression with no expiry

**Wrong:**

```xml
<suppress>
  <notes>Not applicable to our usage. See JIRA-1183.</notes>
  <cve>CVE-2021-44228</cve>
</suppress>
```

**Exact symptom:** none, for years. That is the trap. Then:

- The vulnerable library gets a *new* CVE and the suppression's `packageUrl`
  regex is broad enough to hide that too.
- JIRA-1183 was closed in 2023 and nobody who wrote it still works there.
- The usage pattern changed — someone added an endpoint that does reach the
  vulnerable path — and the suppression is still there, still green.
- An external security audit asks "why is this suppressed?" and there is no
  answer, which is a much worse conversation than "here is the triage".

**Root cause:** a suppression is a **decision made against a snapshot of
reality**: this version, this call graph, this input source. All three change.
A decision with no review date silently outlives its evidence.

**Fix:**

- Every suppression gets `until="YYYY-MM-DDZ"`, no more than one or two quarters
  out.
- Every suppression gets triage notes with: the vulnerable class, the call path,
  whether input is attacker-controlled, who decided, when, and **what would change
  the answer**.
- Scope the suppression as narrowly as possible — a specific CVE on a specific
  purl, never a whole artifact and never a whole CVE across all artifacts.
- A CI job that fails when a suppression is within 30 days of expiry, so
  re-triage is scheduled rather than surprising.

---

### Trap 3 — fixing the version in one module instead of the parent

**Wrong:** the finding was reported against `orderflow-api`, so someone adds a
direct dependency there:

```xml
<!-- orderflow-api/pom.xml -->
<dependency>
  <groupId>com.vendor</groupId>
  <artifactId>layout-engine</artifactId>
  <version>4.2.0</version>
</dependency>
```

**Exact symptoms:**

- The next scan is green for `orderflow-api` and **still red for
  `orderflow-worker`**, which has the same transitive and was never mentioned in
  the original report.
- `mvn dependency:analyze` now reports `Unused declared dependency:
  com.vendor:layout-engine` on `orderflow-api`, because the module declares
  something it never imports. The POM now lies about what the module uses.
- Six months later someone "cleans up unused dependencies", removes it, and
  silently reintroduces the vulnerable version.

**Root cause:** a direct dependency is a statement about *what this module uses*.
It was used here as a lever to change a *version*. Those are different tools, and
using the wrong one leaves a POM whose meaning no longer matches its content.

**Fix:** put the override in the **parent's `<dependencyManagement>`**, above the
BOM import, with a comment naming the CVE, the ticket, the date and the removal
condition. It applies to every module, it does not add anything to any classpath,
and it reads as what it is: a version decision.

Then verify it took effect in **every** module, not just the one that was
reported:

```bash
for m in orderflow-domain orderflow-persistence orderflow-api orderflow-worker; do
  echo "== $m"
  mvn -q -pl "$m" dependency:tree -Dincludes=com.vendor:layout-engine
done
```

---

### Trap 4 — a non-reproducible build, discovered during an incident

**Wrong:** never having set `project.build.outputTimestamp`, and building release
artifacts with `mvn clean install` from whichever machine was free.

**Exact symptom:** during a security incident, you are asked: *"Is the jar
running in production built from commit `a3f91c2`?"*

You rebuild from that commit and compare:

```bash
sha256sum orderflow-api-1.4.2.jar production-orderflow-api-1.4.2.jar
# two different hashes
```

Now you cannot tell whether the difference is benign (a timestamp, a different
`Build-Jdk` in the manifest) or catastrophic (the build server was compromised
and injected a class). **You have no way to distinguish those two cases**, which
means you have to treat it as the second one, which means a much longer and more
expensive incident.

**Root cause:** jar entries carry build timestamps, entry order follows
filesystem iteration order, and the manifest embeds the building JDK and user.
None of that is deterministic across machines or across time.

**Fix:**

```bash
# 1. Set the property, from the commit date.
mvn -B clean verify -Dproject.build.outputTimestamp="$(git log -1 --format=%cI)"

# 2. Prove it. Build twice into different directories and compare.
mvn -B clean package -Dproject.build.outputTimestamp="$(git log -1 --format=%cI)"
cp orderflow-api/target/orderflow-api-*.jar /tmp/run1.jar
mvn -B clean package -Dproject.build.outputTimestamp="$(git log -1 --format=%cI)"
cp orderflow-api/target/orderflow-api-*.jar /tmp/run2.jar
sha256sum /tmp/run1.jar /tmp/run2.jar
```

| What you see | What it means | What to do |
|---|---|---|
| Identical hashes | Reproducible within one machine. Necessary, not sufficient. | Next: reproduce on a *different* machine with the same JDK. |
| Different hashes | Something is still non-deterministic. | `unzip -l` both and diff the listings — differing timestamps or entry order point straight at the cause. `diffoscope` if you have it, which explains jar differences properly. |
| Same listing, different bytes | Content differs — usually a generated file with a date, or an embedded absolute path. | Find the generator and make it deterministic. |

Pin the JDK too (a container image or a Gradle toolchain), because a different
`javac` can emit different bytecode.

---

### Trap 5 — scanning the repo and not the artifact

**Wrong:** CI runs a dependency scan against the source tree, reports green, and
everyone assumes the deployed thing is clean.

**Exact symptoms, and there are three distinct ones:**

- **The base image.** Your Dockerfile starts from a JRE image with its own OS
  packages — glibc, zlib, an SSL library. Those have CVEs and no Maven scan will
  ever see them. Trivy or Grype against the *image* will.
- **The fat jar.** A Boot jar nests hundreds of jars under `BOOT-INF/lib/`. A
  scanner that inspects top-level archive entries sees one jar and reports
  nothing. Confirm your tool actually unpacks nested jars — most modern ones do,
  but "assume yes" is not a control.
- **The time gap.** You scanned at build time. The artifact has been running for
  five months. CVEs published since then are invisible, because nothing re-scans
  what is already deployed.

**Root cause:** three different scopes — source graph, built artifact, running
container — and a scan of one says nothing about the others.

**Fix — layered scanning, all three:**

```bash
# 1. Source/dependency graph, on every PR
mvn org.owasp:dependency-check-maven:check

# 2. The built artifact, as part of the release
trivy fs --scanners vuln orderflow-api/target/orderflow-api-1.0.0.jar

# 3. The container image, including the base OS, before push
trivy image orderflow/api:1.0.0
```

Plus a **scheduled re-scan of the currently deployed version**, driven off the
stored SBOM. That job is what catches "a CVE was published today against
something you shipped in March", and it is the single highest-value piece of
supply-chain automation you can add.

---

## Hands-on proof

Every command below is one **you** run. I have no Maven, no Gradle, no JVM and no
network access to any vulnerability database in this session. Nothing below is
captured output; where a structure is shown it is labelled as an illustration.

### Proof 1 — generate an SBOM and read it

```bash
cd ~/java-lab/31/orderflow
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom
ls -la target/bom.json target/bom.xml
```

Then interrogate it:

```bash
jq '.components | length' target/bom.json
jq -r '.components[] | "\(.purl)  [\(.scope // "unspecified")]"' target/bom.json | sort | head -40
jq -r '.metadata.component.purl' target/bom.json
```

| What you see | What it means |
|---|---|
| A component count in the low hundreds for a Boot app | Normal. Take a moment with that number — it is your actual attack surface, and it is the answer to "how many third parties do we trust?" |
| Components with `"scope": "required"` vs `"optional"` | CycloneDX's scope field. `required` means it is genuinely needed at runtime. |
| JUnit, Mockito, Testcontainers present | Your scope configuration is including test dependencies. Fix it — the SBOM must describe what shipped. |
| A `dependencies` array separate from `components` | The graph structure, not just a flat list. This is what lets you answer "who pulls this in?" from the SBOM alone. |
| A `purl` you do not recognise | Good. That is the point of doing this exercise. Trace it with `mvn dependency:tree -Dincludes=`. |

### Proof 2 — compare the SBOM against the actual jar

The SBOM is generated from the resolved model. The jar is what you ship. Verify
they agree.

```bash
unzip -l orderflow-api/target/orderflow-api-*.jar \
  | awk '/BOOT-INF\/lib\// {print $4}' \
  | sed 's|BOOT-INF/lib/||' | sort > /tmp/jar-libs.txt
wc -l /tmp/jar-libs.txt

jq -r '.components[] | "\(.name)-\(.version).jar"' target/bom.json | sort > /tmp/sbom-libs.txt
diff /tmp/jar-libs.txt /tmp/sbom-libs.txt | head -40
```

| What you see | What it means |
|---|---|
| Empty diff | The SBOM describes the artifact. This is the property you want. |
| Entries only in the SBOM | Test or provided scope leaked into the SBOM configuration. |
| Entries only in the jar | Something is packaged that the model does not know about — a shaded jar, a manually added lib, or a plugin adding files. **Investigate this one.** |
| Version mismatches | The SBOM was generated from a different build than the jar. Bind SBOM generation to the `package` phase so they cannot diverge. |

### Proof 3 — prove reproducibility

```bash
COMMIT_TS="$(git log -1 --format=%cI)"

mvn -B -q clean package -Dproject.build.outputTimestamp="$COMMIT_TS"
cp orderflow-api/target/orderflow-api-*.jar /tmp/run1.jar

mvn -B -q clean package -Dproject.build.outputTimestamp="$COMMIT_TS"
cp orderflow-api/target/orderflow-api-*.jar /tmp/run2.jar

sha256sum /tmp/run1.jar /tmp/run2.jar
```

| What you see | What it means | Next |
|---|---|---|
| Identical hashes | Reproducible on this machine, this JDK. | Try a different machine, same JDK version. Then a different JDK — expect a difference, and understand why. |
| Different hashes | Something is non-deterministic. | Run the drill-down below. |

Drill down on a mismatch:

```bash
unzip -l /tmp/run1.jar > /tmp/l1.txt
unzip -l /tmp/run2.jar > /tmp/l2.txt
diff /tmp/l1.txt /tmp/l2.txt | head -30
```

| What the diff shows | Root cause |
|---|---|
| Same names, different dates | The timestamp property is not being applied — check it reached the archiver plugin with `mvn help:effective-pom`. |
| Same names, different order | Entry ordering is not normalised. Check plugin versions; older archivers do not sort. |
| Same listing entirely, but bytes differ | Content differs. Extract and diff `META-INF/MANIFEST.MF` first — `Build-Jdk-Spec` and `Created-By` are the usual culprits. Then look for generated sources with a date in them. |

If you have `diffoscope`, use it — it unpacks archives recursively and explains
differences in human terms, which saves a lot of guessing.

### Proof 4 — run a scanner and read the report properly

```bash
mvn org.owasp:dependency-check-maven:check
open target/dependency-check-report.html      # or xdg-open
```

**First-run expectations, so you do not think it has hung:** it downloads a
vulnerability database, which takes a long time. Recent versions need an NVD API
key to fetch at a usable rate; without one you may sit at very slow progress for
a long while. Check the plugin's documentation for the current parameter name and
set it from a CI secret.

**How to read the report — this is the part that matters:**

| Column / field | What it actually tells you |
|---|---|
| Dependency | The jar file. Check it against your runtime classpath — test-scoped entries are noise for a shipped-artifact question. |
| Identifiers (the purl / CPE) | How the tool matched your jar to the database. **A wrong CPE match is the main source of false positives** — if the identifier looks wrong, the finding probably is. |
| CVSS score | Severity *if exploited*. Not probability. Not reachability. Do not sort by this alone. |
| Description / references | Read the actual advisory. It names the affected classes and the conditions. This is where reachability triage starts. |
| Evidence / Confidence | How sure the tool is about the match. `LOW` confidence matches deserve scepticism before they deserve a ticket. |

| Overall result | What it means | What to do |
|---|---|---|
| Zero findings | Either you are current, or the scan did not cover what you think. Verify the dependency count in the report matches your runtime classpath size. |
| Dozens of findings, mostly in test dependencies | Set `skipTestScope`. Then re-run and see what is left. |
| A finding on an artifact not in `dependency:list -DincludeScope=runtime` | It is not in the shipped artifact. Confirm and record. |
| A handful of findings on runtime artifacts | This is the real work queue. Apply the funnel from Trap 1. |

### Proof 5 — is the vulnerable class even there?

```bash
mvn -q dependency:copy-dependencies -DoutputDirectory=target/deps -DincludeScope=runtime
ls target/deps | head

# Find which jar contains a specific class
for j in target/deps/*.jar; do
  if unzip -l "$j" 2>/dev/null | grep -q 'SomeVulnerableClass.class'; then
    echo "FOUND in $j"
  fi
done
```

| What you see | What it means |
|---|---|
| No jar contains it | Not exploitable via this path. Strong evidence for a documented suppression. |
| Exactly one jar contains it | Continue to reachability analysis. |
| **Two jars contain it** | Duplicate class definitions (Topic 32, Proof 7). Which one wins is classpath order. Bumping one may not fix anything. Resolve the duplication first. |

### Proof 6 — static reachability, with honest limits

```bash
mvn -q dependency:build-classpath -Dmdep.outputFile=cp.txt -DincludeScope=runtime
jdeps --multi-release 21 -verbose:class -cp "$(cat cp.txt)" \
      orderflow-api/target/classes | grep -i 'vendor.layout'
```

| What you see | What it means |
|---|---|
| No references | Your code does not statically reference it. **Necessary, not sufficient** — a library you call might. |
| References from your classes | You call it directly. Look at which methods, and whether the input is attacker-controlled. |
| Nothing at all, and you expected something | Check the classpath file is populated and that `target/classes` is built. |

**State the limits when you write this up.** `jdeps` sees static references. It
does not see reflection, `ServiceLoader`, dynamic proxies, or Spring's own
wiring — and Java production code is full of all four. Reachability analysis
narrows the question; it does not close it. Writing "not statically reachable;
reflection not ruled out" is a more credible triage note than "not affected".

### Proof 7 — the runtime SBOM from a running service

With the actuator on and the SBOM endpoint exposed:

```bash
curl -s localhost:8080/actuator/sbom | jq .
curl -s localhost:8080/actuator/sbom/application | jq '.components | length'
```

| What you see | What it means |
|---|---|
| A component list matching your build-time SBOM | The running process is what you built. |
| 404 | The endpoint is not exposed (`management.endpoints.web.exposure.include`) or no SBOM is on the classpath. |
| A list that **differs** from the build-time SBOM | Something was added at deploy — an agent, a sidecar-injected jar, an extra classpath entry. **This is a genuinely important finding.** Investigate. |

Do not expose this publicly. It is an inventory of your attack surface.

---

## Practice exercises

### 1 — Easy: know what you ship

On the `orderflow` build from Topic 31:

1. Generate an SBOM. Report the component count.
2. List the ten components with the deepest position in the dependency graph
   (use the SBOM's `dependencies` section, or `mvn dependency:tree`). For each,
   name the direct dependency that pulled it in.
3. Find one component you did not know was there. Trace it with
   `mvn dependency:tree -Dincludes=`. Write two sentences on what it does and why
   it is present.
4. Configure the plugin to exclude test scope, regenerate, and report the new
   count. Explain why that number is the one you would give to a security
   reviewer.

### 2 — Medium: triage a real finding, combining earlier topics

1. Run a scanner on `orderflow`. Take the **highest-CVSS finding on a runtime
   artifact**. If there are none, deliberately downgrade one dependency by a few
   versions with a `dependencyManagement` entry until you get one.
2. Work the full funnel and write it up as a triage note:
   - Scope and presence in the jar (Topic 31).
   - The version mediation actually selected (Topic 32 — use
     `dependency:tree -Dverbose`).
   - Whether the vulnerable class is in the jar.
   - Static reachability from your code (`jdeps` + grep).
   - Whether input is attacker-controlled.
3. If the CVE involves deserialisation, connect it explicitly to Topic 19 — why
   `readObject` on untrusted input is remote code execution rather than merely
   "unsafe", and whether a JDK serialization filter would mitigate it.
4. Make a decision (bump / mitigate / accept-with-expiry / escalate) and
   implement it. If you accept, write the suppression with an `until` date and
   complete notes.
5. Prove the outcome with **four** commands: the tree, the tests, the regenerated
   SBOM, and the jar listing.

### 3 — Hard: production simulation — make `orderflow` supply-chain defensible

**Part A — reproducibility.** Set `project.build.outputTimestamp` from the commit
date. Build twice and compare SHA-256 of every produced jar. Get them identical.
If they are not, diagnose with `unzip -l` and the manifest, and record every
source of nondeterminism you had to fix. Then build on a *different* machine (or
a container with a pinned JDK image) and compare again. Report honestly whether
you achieved cross-machine reproducibility, and what stopped you if not.

**Part B — SBOM in the build.** Bind CycloneDX to the `package` phase, configured
to describe the shipped artifact only. Write the diff check from Proof 2 as a
script and make it a build step that fails on a mismatch.

**Part C — scanning, layered.** Wire in:
- a dependency scan on every PR, with `failBuildOnCVSS` at a threshold you can
  defend in writing;
- an artifact scan of the built jar;
- an image scan of the container from Topic 122 (or a placeholder Dockerfile
  now).
Document what each layer catches that the others do not.

**Part D — the exception process.** Create `security/suppressions.xml` with at
least one real, fully triaged suppression carrying an `until` date. Write a CI
step that fails when any suppression is within 30 days of expiry. Test it by
setting a date in the past.

**Part E — the incident rehearsal.** Pick a real, well-documented historical CVE
in a library you actually depend on — Log4Shell (CVE-2021-44228) is the canonical
teaching example and is exhaustively documented. Assume it was announced this
morning. Produce, timed:
- the answer to "are we affected?" with evidence, in under 10 minutes;
- a triage note;
- a fix or a documented exception;
- proof the fix landed in every module.

Then write half a page: what was slow, what you would automate, and what you
would need in place *before* the next one to make this a 10-minute job rather than
an afternoon. This is a Phase 12 artefact in miniature — write it for a director.

---

## Interview questions

### Q1 — "A scanner reports a critical CVE in a transitive dependency. What do you do first?"

**Mid-level answer:** "Check which version we have and upgrade it to the patched
version."

**Senior answer:** "The first thing I do is *not* upgrade. A scanner reports
presence, not exploitability, and a reflexive bump on a Friday is how security
work causes outages.

My funnel is: is the artifact in something we actually ship — check the scope and
list the jar, because a test-scoped finding is not in production. Then which
version did resolution actually select, with `mvn dependency:tree -Dverbose`,
because scanners often report a version somewhere in the graph rather than the
one mediation picked and that alone resolves a lot of findings. Then is the
vulnerable class even in the jar — libraries frequently ship the vulnerable code
in a different artifact of the same family. Then is the affected method reachable
from our entry points, with grep and `jdeps`, and I'd say explicitly that static
analysis misses reflection and service loading so it narrows the question rather
than closing it. Then is the input attacker-controlled — a deserialisation flaw
reachable only from a bundled resource is a very different risk from one
reachable from a request body.

Then I decide: bump via a `dependencyManagement` override in the parent so all
modules move together, mitigate by configuration if there's a documented one and
the upgrade is risky today, or accept with a dated, fully-triaged suppression.
And I prioritise what survives the funnel by reachability first, then KEV and
EPSS — is it actually being exploited in the wild — and only then CVSS. A 9.8 on
an unreachable path is below a 6.5 on the request path.

Whatever I do, I verify against the built jar, not the POM."

**What separates them:** having an ordered procedure with real commands, knowing
that CVSS is severity-if-exploited rather than priority, naming KEV/EPSS, and
volunteering the limits of static reachability instead of overclaiming.

**Follow-up:** "What if it's in Spring Boot itself?" Then it is a framework
upgrade, and if you are on an EOL line like Boot 3.5 there is no free patch — you
override the managed version yourself and go off the tested combination, or you
upgrade under pressure. That is the argument for staying current.

---

### Q2 — "What is an SBOM and what would you actually do with one?"

**Mid-level answer:** "A list of the libraries in your application, for security
scanning and compliance."

**Senior answer:** "A machine-readable inventory of everything in an artifact —
CycloneDX or SPDX — with each component identified by a purl, plus the
dependency graph so you can see who pulled what in.

The reason I'd fight for one is the org-wide question. When a big CVE lands, the
question is 'which of our 200 services contain this, and at what version?' With
SBOMs stored alongside each release, that is a query someone answers in minutes.
Without them it is 200 engineers running dependency trees on a Saturday. That is
the whole value proposition and it is worth the setup cost on its own.

Two details I care about in practice. It has to describe **what shipped** — so
test and provided scope excluded for a fat jar — and I'd verify that by diffing
the SBOM against `BOOT-INF/lib/` in the actual jar, because an SBOM that
disagrees with the artifact is worse than none. And it only means anything if the
build is reproducible; otherwise the SBOM describes a build, not the artifact
running in production.

The other use is VEX — recording 'not affected, here is why' against a specific
CVE and product version, so the triage work becomes a durable artefact rather
than a Slack thread. Spring Boot can also expose an SBOM through the actuator,
which is nice because it tells you what the running process contains rather than
what the last build produced."

**What separates them:** the org-wide query as the actual value, insisting the
SBOM matches the artifact, the link to reproducibility, and knowing what VEX is.

**Follow-up:** "Where do you store them?" With the release artifact, in the
registry, immutably. An SBOM in `target/` is deleted by the next `mvn clean`.

---

### Q3 — "What makes a Java build non-reproducible, and why should I care?"

**Mid-level answer:** "Timestamps in the jar. It matters for security."

**Senior answer:** "The usual causes are jar entry timestamps, jar entry ordering
from filesystem iteration, manifest fields like `Build-Jdk` and `Built-By`,
absolute paths in debug info or generated sources, code generators that stamp a
date, and the JDK version itself, since a different `javac` can emit different
bytecode.

Why I care is a specific question I want to be able to answer during an incident:
'is the jar in production built from the commit we reviewed?' If the build isn't
reproducible, I rebuild, get a different hash, and cannot distinguish a benign
timestamp difference from an injected class. That forces me to treat it as a
compromise, which turns a two-hour investigation into a two-day one. Reproducible
builds are also the only practical detection for a compromised build server —
an independent rebuild that doesn't match the published artifact.

Mechanically it's cheap on the JVM: `project.build.outputTimestamp` in Maven,
`isPreserveFileTimestamps = false` and `isReproducibleFileOrder = true` in
Gradle, plus pinning the JDK with a container image or a toolchain. Then prove it
by building twice and comparing hashes, and again on a different machine — and I
would not claim reproducibility until I'd actually done the cross-machine
comparison."

**What separates them:** the incident question as the motivation, naming
compromised-build-server detection, and insisting on *proving* it rather than
configuring it and assuming.

**Follow-up:** "Your two builds differ. How do you find out why?" `unzip -l` both
and diff the listings, then the manifest, then `diffoscope` for the rest.

---

### Q4 — "How do you fix a vulnerable transitive dependency across a multi-module repo?"

**Mid-level answer:** "Add the fixed version as a direct dependency in the module
that reported it."

**Senior answer:** "Not as a direct dependency — that makes the POM say the
module *uses* something it doesn't import, `dependency:analyze` will flag it as
unused declared, and the next person cleaning up removes it and silently
reintroduces the vulnerable version. It also only fixes the module that happened
to be reported.

I put a `<dependencyManagement>` entry in the parent POM, above the BOM import
because a directly-declared entry beats an imported one, with a comment naming
the CVE, the ticket, the date, and the condition under which it should be removed
— usually 'when the upstream library ships a version that depends on the fixed
one'. It applies to every module, adds nothing to any classpath, and reads as
what it is: a version decision.

Then I verify. `mvn dependency:tree -Dincludes=` in **every** module, not just
the reported one, and then `unzip -l` on the built jar. That last check matters
because a `dependencyManagement` entry for an artifact that isn't in your graph
does absolutely nothing and reports nothing — and with Boot 4's modularisation,
artifacts moved between 3.x and 4.x, so an override copied from an old project
can silently manage a coordinate that no longer exists.

Long term, the real fix is Renovate or Dependabot keeping us close to current, so
an emergency is a patch bump rather than a major version jump under pressure."

**What separates them:** knowing why a direct dependency is the wrong lever,
knowing the precedence rule for BOM overrides, verifying in every module and
against the jar, and the silent-no-op failure mode.

**Follow-up:** "When would you use an exclusion instead?" Almost never for a CVE
— an exclusion removes rather than selects and turns `NoSuchMethodError` into
`NoClassDefFoundError`. Only when you genuinely want the artifact gone and
something else provides the API.

---

### Q5 — "How would you defend a Java organisation against dependency confusion?"

**Mid-level answer:** "Use a private repository for internal packages."

**Senior answer:** "Routing, not vigilance. Every build resolves through a single
repository manager, enforced by a mirror with `mirrorOf` set to `*` in the CI
settings so no build can bypass it. In the manager, internal groupIds — say
`com.orderflow.*` — are served only from the internal hosted repository and are
never proxied to Central, so a public artifact with those coordinates cannot be
served to a build even if someone publishes one. In Gradle the equivalent is
repository content filters plus `FAIL_ON_PROJECT_REPOS` so a subproject can't
declare its own repository.

Worth noting that Java's exposure here is genuinely lower than npm's. Maven
Central verifies namespace ownership against a domain, artifacts require a PGP
signature, released versions are immutable, and — importantly — there's no
`postinstall` equivalent, so adding a library dependency executes none of its
code. The real execution risk in a Maven build is **plugins**, which do run
during the build, so an unvetted plugin from an unknown groupId is a much bigger
deal than an unvetted library.

On top of routing I'd add Gradle's dependency verification where we use Gradle —
checksums and trusted PGP keys in `verification-metadata.xml`, which fails the
build if any artifact's bytes change. Maven has no built-in equivalent, which is
one of the honest gaps."

**What separates them:** routing enforced centrally rather than trusting each
build, knowing why Java's exposure differs from npm's, singling out plugins as
the real execution risk, and being straight about Maven's gap.

**Follow-up:** "What stops a developer adding a repository to their POM?" The
mirror with `mirrorOf=*` in the CI settings, plus a build-time check. In Gradle,
`FAIL_ON_PROJECT_REPOS` makes it a build error.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A scanner reports presence; you want exploitability. Why has nobody built a
   scanner that reports exploitability directly? Name the specific things about
   the JVM that make it hard, and say whether you think it is solvable.

2. Reproducible builds detect a compromised build server by comparing an
   independent rebuild. Who does the rebuild, and what stops *that* machine from
   being compromised too? At what point does the argument bottom out?

3. Maven Central has no `postinstall` equivalent, so a library dependency
   executes no code at install time. Does that make Java's supply chain safer
   than npm's, or does it just move the risk somewhere else? Where?

4. Your team suppresses a finding with an expiry of one quarter. In three months
   the same person re-triages and extends it. Repeat four times. At what point
   has the expiry mechanism stopped working, and what would you change?

5. An SBOM tells you what is in the artifact. What important supply-chain risks
   does it not describe at all? Name at least three.

6. CVSS scores severity if exploited. EPSS estimates probability of exploitation.
   KEV records known exploitation. If you could only have one of the three,
   which would you take, and does the answer change for a public API versus an
   internal batch job?

7. Staying on an EOL framework line means no free CVE patches. Construct the
   strongest business case for staying on Boot 3.5 anyway, then the rebuttal. Who
   in your organisation should make that call?

---

## Quick reference card

### The commands

```bash
# Inventory
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom
./gradlew cyclonedxBom
curl -s localhost:8080/actuator/sbom | jq .

# Scan
mvn org.owasp:dependency-check-maven:check      # first run is SLOW; needs an NVD API key
./gradlew dependencyCheckAnalyze
trivy fs target/app.jar
trivy image orderflow/api:1.0.0                 # includes base-image OS packages
grype dir:.

# Investigate
mvn dependency:tree -Dverbose -Dincludes=<g>:<a>
mvn dependency:list -DincludeScope=runtime
mvn dependency:copy-dependencies -DoutputDirectory=target/deps -DincludeScope=runtime
unzip -l target/app.jar | grep 'BOOT-INF/lib/'
jdeps --multi-release 21 -verbose:class -cp "$(cat cp.txt)" target/classes

# Reproducibility
mvn -B clean package -Dproject.build.outputTimestamp="$(git log -1 --format=%cI)"
sha256sum run1.jar run2.jar
diffoscope run1.jar run2.jar

# Currency
mvn versions:display-dependency-updates
mvn versions:display-property-updates
mvn versions:display-plugin-updates
./gradlew --write-verification-metadata sha256,pgp build     # Gradle only
```

### The triage funnel

```
1. Is the artifact in something we SHIP?        (scope + unzip the jar)
2. Which version did resolution SELECT?         (dependency:tree -Dverbose)
3. Is the vulnerable CLASS in the jar?          (unzip -l on the dep)
4. Is the method REACHABLE from our entry points? (grep + jdeps; limits apply)
5. Is the input ATTACKER-CONTROLLED?
6. Rank survivors: reachability > KEV/EPSS > CVSS
7. Decide: bump / mitigate by config / accept with expiry / escalate
8. VERIFY against the built jar, not the POM
```

### Where the fix goes

| Situation | Fix |
|---|---|
| Transitive, multi-module | `<dependencyManagement>` in the **parent**, above the BOM import |
| Managed by a BOM you import | Same — a direct entry beats an import |
| You genuinely want the artifact gone | `<exclusion>` — and only then |
| It is in a `provided` dep or base image | Not your fix. Escalate to the owner. |
| It is in the framework, and you are on an EOL line | Override yourself, or upgrade. There is no third option. |

### Gotchas checklist

- [ ] A scanner reports presence, not exploitability.
- [ ] Verify the *resolved* version before believing any finding.
- [ ] SBOM must describe what shipped — exclude test scope; verify against `BOOT-INF/lib/`.
- [ ] Every suppression needs `until` and real triage notes.
- [ ] Fix versions in the parent's `dependencyManagement`, never as a direct dependency.
- [ ] A `dependencyManagement` entry for an artifact not in your graph is a silent no-op.
- [ ] Scan three layers: dependency graph, built artifact, container image.
- [ ] Re-scan what is **already deployed**, on a schedule. New CVEs are published daily.
- [ ] Prove reproducibility by comparing hashes, on two machines. Do not assume.
- [ ] Boot 3.5 is out of OSS support since June 2026 — no free CVE patches.
- [ ] Maven plugins execute during your build. Libraries do not. Vet plugins harder.

---

## When would I use this at work?

**1. The 4pm Friday CVE announcement.**
A critical RCE is announced in something you might use. The team that has an SBOM
per release answers "are we affected, and where?" in ten minutes and goes home.
The team that does not spends the weekend running dependency trees across
seventeen repositories. Same vulnerability, entirely different week. This is the
scenario the whole topic exists for.

**2. Arguing for a framework upgrade with a director.**
"We should upgrade because 4.1 is newer" loses. "We are on a line that left OSS
support in June 2026, so the next CVE in a managed artifact is a project rather
than a version bump, and here are the four overrides we are already carrying
because of it" wins. This topic gives you the second sentence.

**3. Passing a customer security review.**
Enterprise procurement asks for your SBOM, your patch SLA, and your exception
process. Having all three already — generated automatically, with dated
exceptions and triage notes — turns a two-week fire drill into sending three
links. It is unglamorous and it is the kind of thing that gets a senior engineer
noticed.

---

## Connected topics

**Prerequisites:**
- **31 — Maven fundamentals.** Scopes decide what is in the shipped artifact,
  which decides what belongs in the SBOM and what a finding actually means.
- **32 — Dependency resolution.** Every triage starts with "which version is
  actually on the classpath", and every fix is a `dependencyManagement` override
  whose precedence you have to understand.
- **33 — Gradle.** Dependency locking, dependency verification, reproducible
  archives, and repository content filtering are all materially better on the
  Gradle side, and knowing that is part of the tool decision.
- **19 — Serialization.** The largest single family of JVM CVEs is untrusted
  deserialisation. That topic explains *why* they are remote code execution, which
  is what lets you judge whether a given one is reachable.
- **20 — JPMS.** Strong encapsulation is a genuine mitigation for some
  reflection-based gadget chains, and it is one honest argument for modularising a
  library.

**This unlocks:**
- **42 — Auto-configuration mechanics.** Boot discovers
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
  by scanning the classpath, so **removing a dependency for a CVE can silently
  remove behaviour** — an auto-configuration backs off and a feature quietly stops
  working with no error. Always check the condition-evaluation report after a
  supply-chain-driven dependency change.
- **61 — Testcontainers.** Test-scoped dependencies do not ship, which is why
  scope hygiene keeps your SBOM honest — and why a CVE in a Testcontainers
  transitive is a CI risk, not a production one.
- **83 — GraalVM native image.** Closed-world analysis produces a genuinely
  smaller attack surface, since unreachable code is not in the binary at all —
  and it makes reachability something the build can partly answer for you. That
  is a real, underrated security argument for native images.
- **122 — Layered Docker jars.** Your container's CVE surface is base image plus
  dependency layer plus application layer. Layering makes the dependency layer
  cacheable *and* makes it the unit you scan and patch.
- **127 — Migration planning.** Upgrading off an EOL framework line is a
  supply-chain project before it is a code project, and this topic is where you
  get the numbers to justify it.
- **Phase 12 (125–135).** The triage note, the exception register and the
  incident rehearsal from Exercise 3 are miniature versions of the artefacts that
  phase is entirely about.

---

*Java baseline 21, running on JDK 25. Target Spring Boot 4.1 / Framework 7.0 /
Jakarta EE 11. Deliberately, this document contains no third-party plugin or
library version numbers — every one is a `SET-ME` you resolve with
`mvn versions:display-plugin-updates` or from the plugin's own site. It also
contains no scanner output, no CVE counts and no dependency-tree captures,
because I have no build tool and no vulnerability database in this session. The
only real CVE identifier named here is CVE-2021-44228 (Log4Shell), used as a
historical teaching example; `CVE-2026-XXXXX` is an obvious placeholder and is
not a real identifier.*
