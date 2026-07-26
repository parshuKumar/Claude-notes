# 11 — npm in depth

## What is this?

`npm` (Node Package Manager) is the command-line tool that comes bundled with Node.js for installing, managing, and running third-party code packages. Think of it like a hardware store for your project — instead of building every tool (a logger, a password hasher, an HTTP framework) from scratch, you "buy" pre-built, tested ones off the shelf and npm handles fetching them, tracking their versions, and keeping them organized in your project. Every backend project you will ever touch — a solo script or a company's production API — depends on npm to manage its dependencies.

## Why does it matter for backend development?

A real backend app might depend on 20-plus packages (Express, a database driver, a validator, a logger, dotenv...), and each of those depends on more packages underneath it. Without npm, you would have to manually download, version, and wire up hundreds of files. Backend developers use npm every single day to: install packages (`npm install express`), run project scripts (`npm run dev`, `npm test`), scaffold new projects (`npm init`), execute one-off tools without installing them globally (`npx`), and audit dependencies for security vulnerabilities before shipping to production (`npm audit`). Getting npm wrong — committing `node_modules`, ignoring `package-lock.json`, installing packages globally that should be local — causes real production incidents ("it works on my machine" bugs).

---

## Syntax / API

```bash
# ── Installing packages ──────────────────────────────────────────────────────
npm install express              # install as a regular dependency (runtime needed)
npm install nodemon --save-dev   # install as a devDependency (only needed while developing)
npm install                      # (no package name) install everything listed in package.json
npm install express@4.18.2       # install an exact version
npm uninstall express            # remove a package and update package.json

# ── Running scripts ───────────────────────────────────────────────────────────
npm run dev                      # run the "dev" script defined in package.json
npm start                        # shorthand — runs the "start" script (no "run" needed)
npm test                         # shorthand — runs the "test" script (no "run" needed)

# ── Scaffolding a new project ────────────────────────────────────────────────
npm init                         # interactive prompts to generate package.json
npm init -y                      # skip prompts, accept all defaults

# ── npx: run a package's binary WITHOUT installing it globally ──────────────
npx create-react-app my-app      # downloads temporarily, runs it, then discards it
npx nodemon server.js            # runs a locally-installed binary without a global install

# ── Global vs local packages ─────────────────────────────────────────────────
npm install -g pm2               # install globally — available as a command anywhere
npm list -g --depth=0            # list all globally installed packages

# ── Maintenance commands ──────────────────────────────────────────────────────
npm outdated                     # show which installed packages have newer versions available
npm audit                        # scan dependencies for known security vulnerabilities
npm audit fix                    # automatically upgrade packages to patch vulnerabilities
npm prune                        # remove packages NOT listed in package.json from node_modules
```

## How it works — line by line

- `npm install <pkg>` reads your `package.json`, downloads the package (and everything IT depends on) from the npm registry, drops the actual files into a `node_modules/` folder, adds an entry to `package.json`'s `dependencies`, and locks the exact resolved versions in `package-lock.json`.
- `--save-dev` (or `-D`) puts the package under `devDependencies` instead — meaning "only needed to build or test the project, not to run it in production." Tools like `nodemon`, `jest`, and `eslint` belong here.
- `npm run <script>` looks up the `scripts` object in `package.json` and executes the shell command mapped to that name. `start` and `test` are special-cased so you can type `npm start` instead of `npm run start`.
- `npm init` walks you through creating a `package.json` from scratch by asking questions (name, version, entry point, etc.); `npm init -y` fills in sensible defaults instantly with zero prompts — the one every backend dev actually uses when starting a new project.
- `npx <command>` looks for the binary first in your project's local `node_modules/.bin`, and if it's not there, temporarily downloads the package, runs it once, and does not permanently install it (unless you explicitly ask it to). This avoids polluting your global system and avoids "works on my machine because I have it installed globally" bugs.
- Global installs (`npm install -g`) put a package somewhere on your system PATH so it's usable as a command from any directory (e.g. `pm2`, `nodemon` as CLI tools) — but they are NOT tracked in any project's `package.json`, so a teammate cloning your repo will not automatically get them.
- `package-lock.json` records the **exact** version of every package and sub-dependency that was actually installed, down to the specific commit-like resolved URL and integrity hash — so `npm install` produces byte-for-byte identical `node_modules` on every machine and every CI run.
- `node_modules/` is the actual folder where all downloaded package code physically lives — it can contain thousands of files and is always excluded from git via `.gitignore` because it is fully reproducible from `package.json` + `package-lock.json`.
- `npm audit` cross-references every installed package version against a public vulnerability database and reports known security issues by severity; `npm audit fix` attempts to auto-upgrade affected packages to patched versions without breaking your semver ranges.
- `npm outdated` compares what's installed against what's the latest available on the registry, showing three columns: `Current` (what you have), `Wanted` (the highest version matching your `package.json` semver range), and `Latest` (the newest version published, which may require a `package.json` change to adopt).
- `npm prune` removes anything sitting in `node_modules` that is no longer listed in `package.json` — useful after manually deleting a dependency line, or when switching branches with different dependency sets.

---

## Example 1 — basic

```js
// Terminal commands run in an empty project folder — not JS code, but the workflow every backend dev repeats

// 1. Scaffold package.json with defaults (name, version 1.0.0, entry index.js, license ISC)
// $ npm init -y

// 2. Install Express as a runtime dependency — needed when the app actually runs
// $ npm install express

// 3. Install nodemon as a dev-only dependency — only needed while coding, not in production
// $ npm install nodemon --save-dev

// 4. Check what got installed, without descending into sub-dependencies
// $ npm list --depth=0
// └── express@4.18.2
// └── nodemon@3.0.1

// 5. Add custom scripts by hand-editing package.json (shown below), then run them
// $ npm run dev      →  runs nodemon, auto-restarts server.js on file changes
// $ npm start        →  runs "node server.js" directly, no auto-restart
```

```json
// File: package.json (relevant excerpt after the steps above)
{
  "name": "my-backend-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

---

## Example 2 — real world backend use case

```bash
# File: setup-and-audit.sh
# A realistic onboarding + maintenance workflow a backend dev runs when
# joining an existing project and before shipping a release.

# ── Step 1: Fresh clone — install EXACTLY what package-lock.json specifies ──
# npm ci (not npm install!) is stricter and faster for reproducible installs:
# it deletes node_modules first and installs the locked versions verbatim,
# failing loudly if package.json and package-lock.json are out of sync.
npm ci

# ── Step 2: Run a one-off code generator without adding it as a dependency ──
# npx fetches @nestjs/cli temporarily, scaffolds a module, then discards it
npx @nestjs/cli generate module auth

# ── Step 3: Check for outdated packages before starting new feature work ────
npm outdated
# Package     Current  Wanted  Latest
# express     4.18.2   4.19.0  5.0.0     ← Wanted is safe to take, Latest may break things

# ── Step 4: Update to the "Wanted" version safely (respects semver ranges) ──
npm update express

# ── Step 5: Security audit before a production deploy ───────────────────────
npm audit --production
# found 2 vulnerabilities (1 moderate, 1 high) in 340 scanned packages

# Auto-fix what's safely fixable without breaking the API surface
npm audit fix

# ── Step 6: Clean up node_modules after someone removed a dependency line ────
# (e.g. a teammate deleted "moment" from package.json in their last commit)
npm prune
# removes moment/ and any of its unused sub-dependencies from node_modules

# ── Step 7: Install a process manager globally — a system-wide dev tool, ────
# not a project dependency, so it does NOT go in package.json
npm install -g pm2
pm2 start server.js --name api-server
```

```js
// File: scripts/checkAuthToken.js
// A real backend script that only runs occasionally — a perfect npx candidate
// if it were published as a package, or a devDependency-driven npm script otherwise.

const jwt = require('jsonwebtoken');   // installed via: npm install jsonwebtoken

const authToken = process.argv[2];     // token passed as a CLI argument
const apiKey    = process.env.JWT_SECRET; // secret loaded from environment (Topic 12)

try {
  const decoded = jwt.verify(authToken, apiKey); // verify signature + expiry
  console.log('Valid token for userId:', decoded.userId);
} catch (err) {
  console.error('Invalid or expired token:', err.message);
  process.exit(1); // non-zero exit code signals failure to shell scripts / CI
}

// Wired up in package.json as:
// "scripts": { "check-token": "node scripts/checkAuthToken.js" }
// Run with: npm run check-token -- <the-jwt-string>
```

---

## Common mistakes

### Mistake 1 — Committing node_modules to git

```bash
# ❌ WRONG — node_modules can contain 50,000+ files and bloats the repo massively
git add node_modules
git commit -m "add dependencies"
# Every clone becomes gigabytes in size; merge conflicts inside vendored code

# ✅ CORRECT — ignore node_modules, commit package.json + package-lock.json instead
echo "node_modules" >> .gitignore
git add package.json package-lock.json
git commit -m "add express and jsonwebtoken dependencies"
# Anyone can rebuild node_modules exactly with: npm ci
```

### Mistake 2 — Using npm install in CI/CD instead of npm ci

```bash
# ❌ WRONG — npm install can silently update package-lock.json and install
# slightly different versions than what was tested, causing "works locally,
# breaks in production" bugs during a deploy pipeline
npm install
npm run build

# ✅ CORRECT — npm ci installs the EXACT locked versions, is faster, and
# fails immediately if package.json and package-lock.json disagree
npm ci
npm run build
```

### Mistake 3 — Installing a project tool globally instead of as a devDependency

```bash
# ❌ WRONG — installing nodemon only globally means teammates and CI servers
# without that global install can't run "npm run dev" at all
npm install -g nodemon
# package.json has no record of this — it's invisible to anyone else on the team

# ✅ CORRECT — install it locally as a devDependency so it's tracked and
# reproducible; npm automatically makes it available to "npm run" scripts
npm install nodemon --save-dev
# package.json now lists it under devDependencies — "npm ci" on any machine
// or CI server reproduces the exact same dev environment
```

---

## Practice exercises

### Exercise 1 — easy

In an empty folder, do the following using only npm commands (no manual file editing):
1. Initialize a new `package.json` with all defaults accepted automatically.
2. Install `express` as a regular dependency.
3. Install `nodemon` as a dev dependency.
4. Add a `"dev"` script to `package.json` that runs `nodemon index.js`, and a `"start"` script that runs `node index.js`.
5. Run `npm list --depth=0` and confirm both packages show up correctly, one under dependencies and one under devDependencies.

```js
// Write your code here
```

---

### Exercise 2 — medium

Set up a small project that demonstrates the difference between `npm install` and `npm ci`:
1. Create a project, install two or three packages of your choice (e.g. `express`, `dotenv`, `uuid`).
2. Manually delete the `node_modules` folder (but keep `package.json` and `package-lock.json`).
3. Run `npm ci` and observe how it rebuilds `node_modules` from the lock file.
4. Now manually edit `package.json` to add one more dependency by hand (without running `npm install` for it) so it's out of sync with `package-lock.json`.
5. Run `npm ci` again and note what error message appears — write down why `npm ci` behaves this way and why it is preferred over `npm install` in CI/CD pipelines.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small CLI maintenance script, `scripts/depsReport.js`, that a backend team could run before every release:
1. It should use Node's `child_process` module (a preview — covered fully in Topic 27) to run `npm outdated --json` and `npm audit --json` as subprocesses and capture their output.
2. Parse the JSON output of both commands.
3. Print a clean summary report to the console: how many packages are outdated (grouped by current vs wanted vs latest), and how many vulnerabilities exist grouped by severity (low, moderate, high, critical).
4. If any `critical` or `high` vulnerabilities exist, `process.exit(1)` so the script can be wired into a CI pipeline as a release gate; otherwise `process.exit(0)`.
5. Wire it into `package.json` as `"scripts": { "deps-report": "node scripts/depsReport.js" }` and run it with `npm run deps-report`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
INSTALLING
  npm install <pkg>              → add as a dependency (runtime)
  npm install <pkg> --save-dev   → add as a devDependency (build/test only)
  npm install                    → install everything from package.json
  npm ci                         → install EXACT locked versions (CI/prod — deletes node_modules first)
  npm uninstall <pkg>            → remove a package + its package.json entry

RUNNING
  npm run <script>               → run a custom script from package.json
  npm start / npm test           → shorthand for "npm run start" / "npm run test"
  npx <cli-tool>                 → run a binary once without a permanent global install

INIT
  npm init                       → interactive package.json creation
  npm init -y                    → instant package.json with defaults

GLOBAL vs LOCAL
  npm install -g <pkg>           → system-wide CLI tool, NOT tracked in package.json
  npm install <pkg>              → project-local, tracked, reproducible via package-lock.json
  Rule of thumb: if your code "require()"s it → local. If it's a CLI tool you run
  from anywhere → global (or better, npx).

FILES
  package.json        → declares WHAT you depend on (semver ranges like ^4.18.2)
  package-lock.json    → records EXACTLY what was installed (pinned versions + hashes)
  node_modules/        → the actual downloaded code — never commit to git

MAINTENANCE
  npm outdated                   → Current / Wanted / Latest versions comparison
  npm audit                      → scan for known security vulnerabilities
  npm audit fix                  → auto-upgrade to patched versions where possible
  npm prune                      → remove packages not listed in package.json from node_modules
  npm update <pkg>                → upgrade to latest version allowed by semver range (Wanted)

NEVER DO
  Commit node_modules to git             → always .gitignore it
  Use "npm install" in CI/CD pipelines   → use "npm ci" for reproducibility
  Install project tools only globally    → teammates/CI won't have them; use --save-dev
```

---

## Connected topics

- **10 — package.json in depth** — the file npm reads and writes; understanding `dependencies`, `scripts`, and semver ranges (`^`, `~`) makes every npm command here make sense
- **12 — Environment variables** — `.env` files and secrets are often loaded via an npm-installed package (`dotenv`), tying directly into the install workflow covered here
- **81 — Dependency auditing** — goes deeper into `npm audit`, Snyk, and CVE tracking as a full production security practice, building on the basic `npm audit` usage introduced here
