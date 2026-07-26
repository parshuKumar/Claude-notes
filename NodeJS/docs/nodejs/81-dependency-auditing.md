# 81 — Dependency auditing

## What is this?

Dependency auditing is the process of scanning every package your project pulls in — direct and transitive — against a database of known security vulnerabilities (CVEs), then reporting or fixing what it finds. Node ships this built in as `npm audit`, and third-party services like Snyk go deeper with their own vulnerability databases and continuous monitoring. Think of it like a food safety recall system: you didn't grow every ingredient yourself, but before you serve the meal you check every supplier's batch number against the government's recall list — because a bad ingredient from a supplier three levels removed can still poison the whole dish.

## Why does it matter for backend development?

A typical Express or Fastify app depends on hundreds of transitive packages you never directly chose — your `express` dependency alone pulls in dozens of sub-dependencies. Attackers exploit this by publishing malicious updates to popular packages (supply-chain attacks) or by known bugs sitting unpatched in libraries your code never audits by hand. Backend servers are the highest-value target because they touch databases, auth tokens, and file systems — a single vulnerable `jsonwebtoken` or `lodash` version has been the entry point for real production breaches. Every backend team runs `npm audit` (or Snyk) on every install and in every CI pipeline before deploying, because shipping a server with a known, published CVE is an avoidable, embarrassing failure.

---

## Syntax / API

```bash
# Scan installed dependencies (from package-lock.json) against npm's advisory database
npm audit

# Same scan, but output machine-readable JSON — used by scripts and CI pipelines
npm audit --json

# Only report/fail on vulnerabilities at or above this severity (low | moderate | high | critical)
npm audit --audit-level=high

# Automatically upgrade vulnerable packages to the lowest safe version that still satisfies
# the semver ranges in package.json (no breaking major bumps)
npm audit fix

# Same as above, but ALSO allows semver-MAJOR bumps to fix deeper issues — risky, test after
npm audit fix --force

# List packages that have newer versions published than what you currently have installed
npm outdated

# Update packages to the newest version still allowed by the ranges in package.json (^, ~)
npm update
```

```bash
# ── Snyk: a third-party scanner with a larger, faster-updated vulnerability database ──

# Install the Snyk CLI as a global tool
npm install -g snyk

# Log in — opens a browser to link this machine to your Snyk account
snyk auth

# Scan the current project's dependencies for known vulnerabilities
snyk test

# Upload a snapshot of this project to Snyk's dashboard for ongoing, scheduled monitoring
snyk monitor
```

---

## How it works — line by line

`npm audit` reads your `package-lock.json` (the exact resolved version tree, not just `package.json`'s loose ranges) and sends the package names and versions to npm's registry, which checks them against the **GitHub Advisory Database** — a public, community- and vendor-maintained list of known CVEs mapped to affected package version ranges. For every match, npm reports the package name, the vulnerable version range, the severity (`low`, `moderate`, `high`, `critical`), and — if one exists — the version that fixes it.

`npm audit fix` takes that report and rewrites `package-lock.json`, bumping each vulnerable package to the lowest version that both fixes the CVE and still satisfies the semver range already declared in your `package.json` (so `^4.0.0` can become `4.2.1` but never `5.0.0`). `--force` removes that safety rail and will bump across major versions too, which can silently change or break APIs your code relies on.

Snyk works on the same principle — comparing your dependency tree against a vulnerability database — but it maintains its **own** database (often faster to add new advisories than the public one), goes deeper into transitive dependency chains, checks license compliance, and `snyk monitor` keeps watching your project on a schedule even after you stop running commands manually, alerting you the moment a CVE is published for something you already have installed.

---

## Example 1 — basic

```js
// File: scripts/audit-summary.js
// Runs `npm audit` programmatically and prints a clean severity breakdown.

const { execSync } = require('child_process');   // lets Node run shell commands and capture output

function getAuditReport() {
  try {
    // Run npm audit in JSON mode; execSync returns stdout as a Buffer/string
    const rawOutput = execSync('npm audit --json', { encoding: 'utf8' });
    return JSON.parse(rawOutput);                 // parse the JSON report into a JS object
  } catch (err) {
    // npm audit exits with a non-zero code when vulnerabilities are found —
    // execSync throws in that case, but err.stdout still holds the JSON report
    return JSON.parse(err.stdout);
  }
}

const report = getAuditReport();

// report.metadata.vulnerabilities looks like: { info, low, moderate, high, critical, total }
const counts = report.metadata.vulnerabilities;

console.log('Dependency audit summary');
console.log('-------------------------');
console.log(`Critical : ${counts.critical}`);
console.log(`High     : ${counts.high}`);
console.log(`Moderate : ${counts.moderate}`);
console.log(`Low      : ${counts.low}`);
console.log(`Total    : ${counts.total}`);

if (counts.critical > 0 || counts.high > 0) {
  console.log('\n⚠️  Action required: high or critical vulnerabilities found.');
} else {
  console.log('\n✅ No high or critical vulnerabilities found.');
}
```

---

## Example 2 — real world backend use case

```js
// File: scripts/security-gate.js
// A CI-friendly audit gate: fails the pipeline (non-zero exit code) if vulnerabilities
// at or above a configurable severity threshold are found. Run this before every deploy.

const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

// Severity threshold comes from an env var so different pipelines (staging vs prod)
// can enforce different strictness without changing code
const severityThreshold = process.env.AUDIT_SEVERITY || 'high';   // default: block on high/critical
const severityRank = { info: 0, low: 1, moderate: 2, high: 3, critical: 4 };

function runAudit() {
  try {
    const raw = execSync('npm audit --json', { encoding: 'utf8' });
    return JSON.parse(raw);
  } catch (err) {
    // Non-zero exit still gives us the JSON report on stdout — parse it either way
    return JSON.parse(err.stdout || '{}');
  }
}

function extractFindings(report) {
  const findings = [];

  // report.vulnerabilities is keyed by package name in modern npm audit output
  for (const [packageName, details] of Object.entries(report.vulnerabilities || {})) {
    findings.push({
      packageName,
      severity: details.severity,                     // e.g. "high"
      viaRange: details.range,                         // affected version range
      fixAvailable: details.fixAvailable,               // true, false, or an object with a version
    });
  }

  return findings;
}

const report = runAudit();
const findings = extractFindings(report);
const thresholdRank = severityRank[severityThreshold];

// Filter down to only findings that meet or exceed our configured threshold
const blockingFindings = findings.filter(
  (finding) => severityRank[finding.severity] >= thresholdRank
);

// Persist a timestamped report so security team can review historical audit runs
const reportPath = path.join(__dirname, '..', 'logs', `audit-${Date.now()}.json`);
fs.mkdirSync(path.dirname(reportPath), { recursive: true });
fs.writeFileSync(reportPath, JSON.stringify({ findings, threshold: severityThreshold }, null, 2));

if (blockingFindings.length > 0) {
  console.error(`❌ Deploy blocked — ${blockingFindings.length} finding(s) at or above "${severityThreshold}":`);
  blockingFindings.forEach((finding) => {
    const fixNote = finding.fixAvailable ? 'fix available' : 'no fix published yet';
    console.error(`  - ${finding.packageName} [${finding.severity}] (${fixNote})`);
  });
  process.exit(1);   // non-zero exit code fails the CI/CD job and stops the deploy
}

console.log(`✅ No findings at or above "${severityThreshold}". Safe to deploy.`);
process.exit(0);

// In a GitHub Actions workflow (see Topic 108):
//   - run: node scripts/security-gate.js
//     env:
//       AUDIT_SEVERITY: high
```

---

## Common mistakes

### Mistake 1 — Running `npm audit fix --force` blindly in CI

```js
// ❌ WRONG — a postinstall or CI script that force-fixes without human review
// This can silently bump a major version (e.g. express 4 → 5) and break the app,
// but the pipeline still reports "success" because the audit passed.
// package.json:
// "scripts": { "postinstall": "npm audit fix --force" }
```

```js
// ✅ CORRECT — run audit fix locally, review the diff, run the full test suite,
// then commit the updated package-lock.json as its own reviewed change.
// Terminal (developer machine, not CI):
//   npm audit fix          // safe, semver-range-respecting fixes only
//   npm test                // confirm nothing broke
//   git diff package-lock.json   // review exactly what changed
//   git add package-lock.json && git commit -m "fix: patch vulnerable deps"
```

### Mistake 2 — Not enforcing audit results in the pipeline

```js
// ❌ WRONG — CI installs dependencies but never checks the audit result,
// so a critical CVE ships to production silently
// .github/workflows/deploy.yml (conceptually):
//   - run: npm install
//   - run: npm run build
//   - run: npm run deploy        // vulnerabilities never blocked the deploy
```

```js
// ✅ CORRECT — use `npm ci` (installs exactly from the committed lockfile) and
// fail the build on high/critical findings before the deploy step ever runs
// .github/workflows/deploy.yml (conceptually):
//   - run: npm ci                              // reproducible install from lockfile
//   - run: npm audit --audit-level=high         // exits non-zero → fails this step
//   - run: npm run build
//   - run: npm run deploy                       // only reached if audit passed
```

### Mistake 3 — Treating a clean `npm audit` as proof the app is safe

```js
// ❌ WRONG — assuming zero reported vulnerabilities means "fully secure",
// and skipping any other dependency hygiene
const dependencies = require('./package.json').dependencies;
console.log('npm audit: 0 vulnerabilities — we are 100% safe, ship it');
// Reality: npm audit only flags CVEs that have ALREADY been publicly disclosed.
// It does not catch zero-days, typosquatted package names, or malicious code
// hidden in a brand-new package version that has no advisory yet.
```

```js
// ✅ CORRECT — treat npm audit as ONE layer, not the whole strategy:
// combine it with a second scanner, pin exact versions for critical packages,
// and review new dependencies before adding them.
// package.json — pin exact versions (no ^ or ~) for security-critical packages:
// "dependencies": { "jsonwebtoken": "9.0.2", "bcrypt": "5.1.1" }
//
// Terminal — run a second, independently-maintained scanner too:
//   snyk test                 // different vulnerability database, catches more
//   snyk monitor              // keeps watching after this scan, alerts on new CVEs
//
// Before adding any new package: check its weekly download count, last publish
// date, and open issues on its GitHub repo — a package updated 4 years ago with
// 12 downloads/week is a red flag regardless of what npm audit says right now.
```

---

## Practice exercises

### Exercise 1 — easy

In any existing Node.js project (or a fresh one with a few dependencies installed):
1. Run `npm audit` from the terminal and read the human-readable output.
2. Run `npm audit --json` and save the output to a file called `audit-report.json`.
3. Open that JSON file and identify: total vulnerability count, and the counts broken down by severity (`low`, `moderate`, `high`, `critical`).
4. Write a short plain-text summary (a few lines) to `audit-summary.txt` describing what you found, in your own words.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a script `checkAuditLevel.js` that:
1. Runs `npm audit --json` programmatically using `child_process` (handle the case where the command exits with a non-zero code but still returns valid JSON on stdout).
2. Parses the `metadata.vulnerabilities` object from the report.
3. If there are any `high` or `critical` vulnerabilities, logs each one's package name and severity, then exits the process with code `1`.
4. If there are none, logs a success message and exits with code `0`.
5. Test it against a project that has at least one known vulnerable dependency, and confirm the exit code behaves correctly with `echo $?` (Mac/Linux) after running it.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `DependencyWatcher` module that:
1. Exposes a function `runCheck(options)` where `options.severityThreshold` defaults to `'high'`.
2. Internally runs both `npm audit --json` and `npm outdated --json` and combines the results into one report object: `{ vulnerabilities: [...], outdatedPackages: [...] }`.
3. Filters the vulnerability list down to only entries at or above `options.severityThreshold`.
4. Writes the full combined report to a timestamped file inside a `reports/` directory (e.g. `reports/dependency-check-2026-07-27T10-30-00.json`), creating the directory if it doesn't exist.
5. If any blocking vulnerability was found, prints a formatted alert for each one (package name, severity, and the fixed version if available) and returns `false`. Otherwise returns `true`.
6. Supports being run directly from the command line with a `--severity=` flag (e.g. `node dependency-watcher.js --severity=critical`) that overrides the default threshold, and exits the process with code `1` if `runCheck` returns `false`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
NPM AUDIT — CORE COMMANDS
  npm audit                        → scan installed deps against advisory database
  npm audit --json                 → machine-readable output for scripts/CI
  npm audit --audit-level=high     → only report/fail on high or critical
  npm audit fix                    → auto-fix within existing semver ranges (safe)
  npm audit fix --force            → allows MAJOR version bumps (risky, test after)
  npm outdated                     → shows packages with newer versions available
  npm update                       → updates within package.json's allowed ranges

SEVERITY LEVELS (low → high)
  info  →  low  →  moderate  →  high  →  critical

SNYK — CORE COMMANDS
  npm install -g snyk   → install CLI
  snyk auth             → link CLI to your Snyk account
  snyk test             → one-time scan of current project
  snyk monitor          → uploads snapshot, watches project on a schedule

HOW AUDITING WORKS
  package-lock.json  →  compared against  →  GitHub Advisory Database (or Snyk's DB)
  Match found        →  severity + affected range + fixed version reported

CI/CD RULE OF THUMB
  npm ci                            → reproducible install from committed lockfile
  npm audit --audit-level=high      → non-zero exit fails the pipeline step
  Run BEFORE build/deploy steps, never after

GOTCHAS
  Clean audit ≠ fully secure        → only catches KNOWN, disclosed CVEs
  --force in automated scripts      → can silently break the app on major bumps
  Deleting package-lock.json        → makes every install non-reproducible
  Low-severity noise                → tune --audit-level per environment, don't ignore forever

GOOD HABITS
  Pin exact versions for security-critical packages (auth, crypto libs)
  Run audit on every CI build, not just locally
  Review new dependency's download count / last publish date before installing
  Use a second scanner (Snyk) alongside npm audit for broader coverage
  Schedule `snyk monitor` / recurring CI audits — new CVEs get published constantly
```

---

## Connected topics

- **11 — npm in depth** — `npm audit`, `npm outdated`, and `npm update` are core npm commands covered there in full, alongside `package-lock.json` mechanics.
- **74 — Security fundamentals in Node** — dependency security is one of the pillars of the Node security model introduced in that topic; this doc is the deep dive.
- **108 — CI/CD basics with GitHub Actions** — where the audit gate script from Example 2 actually gets wired into a real pipeline that blocks deploys automatically.
