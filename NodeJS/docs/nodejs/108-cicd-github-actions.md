# 108 — CI/CD basics with GitHub Actions

## What is this?

CI/CD (Continuous Integration / Continuous Deployment) is the practice of automatically testing and shipping your code every time you push it, instead of doing it by hand. GitHub Actions is GitHub's built-in automation engine — you write a YAML "workflow file" describing steps (install dependencies, run tests, build a Docker image, push it), and GitHub runs those steps on its own servers whenever something happens in your repo (a push, a pull request, a tag). Think of it as a tireless intern who checks out your code the instant you push, runs your entire test suite, and only lets the change move forward if everything passes — no human has to remember to do it.

## Why does it matter for backend development?

Backend teams push code many times a day, and a broken `main` branch means the API stops working for everyone downstream. CI/CD catches bugs before they merge (tests fail loudly on the PR, not silently in production), enforces consistency (every commit gets linted and tested the same way, regardless of whose laptop it came from), and automates the boring, error-prone parts of shipping — building a Docker image by hand and pushing it to a registry is exactly the kind of task humans forget steps in. Every hire-ready backend developer is expected to read and write a basic GitHub Actions workflow; it is usually the first thing a new engineer touches when joining a team's repo.

---

## Syntax / API

```yaml
# File: .github/workflows/ci.yml
# GitHub looks for workflow files in this exact folder — no other location works

name: CI                       # Name shown in the GitHub Actions tab

# ── Triggers: WHEN this workflow runs ───────────────────────────────────────
on:
  push:
    branches: [main]            # run on every push to main
  pull_request:
    branches: [main]            # run on every PR targeting main

# ── Jobs: WHAT runs, and on what machine ────────────────────────────────────
jobs:
  test:                         # job id — can be anything, referenced by other jobs
    runs-on: ubuntu-latest      # the OS image GitHub spins up for this job

    steps:                      # steps run top to bottom, in order, in the same VM
      - name: Checkout code
        uses: actions/checkout@v4   # pulls your repo's code into the runner

      - name: Set up Node.js
        uses: actions/setup-node@v4 # installs Node.js on the runner
        with:
          node-version: '20'        # pin the exact Node version your app uses
          cache: 'npm'               # cache node_modules between runs — faster builds

      - name: Install dependencies
        run: npm ci                 # 'ci' = clean install from package-lock.json (not 'npm install')

      - name: Run tests
        run: npm test                # runs the "test" script from package.json
```

---

## How it works — line by line

- `name: CI` — a label. It shows up as the workflow's name in GitHub's UI so you can tell workflows apart.
- `on:` — this block says which events wake the workflow up. Here it wakes up on a push to `main` and on any pull request aimed at `main`.
- `jobs:` — a workflow is made of one or more jobs. Each job gets its **own fresh virtual machine** — nothing from one job carries over to another unless you explicitly share it.
- `test:` — this is the job's ID. You'll see this name in the GitHub Actions tab as a green checkmark or red X.
- `runs-on: ubuntu-latest` — tells GitHub which operating system image to boot up to run this job's steps. `ubuntu-latest` is the cheapest and most common choice.
- `steps:` — the actual list of commands, run one after another, top to bottom, inside that single VM.
- `uses: actions/checkout@v4` — this step doesn't run a shell command, it runs a pre-built "Action" (a reusable plugin) that clones your repository into the runner's file system. Without this step there is no code to test.
- `uses: actions/setup-node@v4` with `node-version: '20'` — installs the exact Node.js version you tell it to, so your CI environment matches your local/production environment.
- `cache: 'npm'` — tells the setup-node action to cache your `node_modules` between runs, so future runs install dependencies faster instead of downloading everything from scratch every time.
- `run: npm ci` — runs a real shell command inside the VM. `npm ci` is used instead of `npm install` because it installs the *exact* versions locked in `package-lock.json` — deterministic, reproducible builds, no surprises from a dependency that updated overnight.
- `run: npm test` — runs whatever script is defined under `"test"` in `package.json` (usually `jest` or `mocha`). If any test fails, this step exits with a non-zero code, and the whole job is marked as failed — GitHub shows a red X on the commit/PR.

---

## Example 1 — basic

```yaml
# File: .github/workflows/test-on-push.yml
# Minimal workflow: run the test suite every time someone pushes code.

name: Run Tests                     # label shown in GitHub's Actions tab

on:
  push:                              # trigger on push events
    branches: ['**']                 # '**' matches every branch, not just main

jobs:
  run-tests:                         # job id
    runs-on: ubuntu-latest           # use GitHub's Ubuntu runner

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4    # get the pushed commit's code onto the runner

      - name: Install Node.js
        uses: actions/setup-node@v4  # sets up a Node.js environment
        with:
          node-version: '20'         # match the version used in production

      - name: Install dependencies
        run: npm ci                  # clean, lockfile-exact install

      - name: Run test suite
        run: npm test                # fails the job (red X) if any test throws

      - name: Run linter
        run: npm run lint            # catches style/syntax problems, separate from tests
```

---

## Example 2 — real world backend use case

```yaml
# File: .github/workflows/build-and-push-docker.yml
# Real pipeline: test the API, then build its Docker image and push it to
# Docker Hub — only after tests pass, and only on pushes to main.

name: Build and Push Docker Image

on:
  push:
    branches: [main]                 # only deploy-track builds trigger this

jobs:
  test:                              # first job: must pass before the build job runs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4               # get the code
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci                              # install exact dependency versions
      - run: npm test                            # run the backend's test suite

  build-and-push:                    # second job: only runs if 'test' succeeds
    needs: test                       # <- this is the dependency link between jobs
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4               # get the code again (fresh VM, fresh checkout)

      - name: Log in to Docker Hub
        uses: docker/login-action@v3             # authenticates docker CLI on the runner
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}   # pulled from GitHub repo secrets
          password: ${{ secrets.DOCKERHUB_TOKEN }}       # never hardcode credentials in YAML

      - name: Build Docker image
        run: |
          docker build -t myorg/user-api:${{ github.sha }} .
          # tag with the commit SHA so every image is traceable to an exact commit

      - name: Tag image as latest
        run: docker tag myorg/user-api:${{ github.sha }} myorg/user-api:latest
        # also tag 'latest' so deploy scripts can always pull the newest build

      - name: Push image to Docker Hub
        run: |
          docker push myorg/user-api:${{ github.sha }}   # push the SHA-tagged image
          docker push myorg/user-api:latest              # push the 'latest' tag too
```

```js
// File: src/health.js
// A tiny health-check endpoint the test suite (and CI) verifies before any
// Docker image is allowed to be built and pushed — a common CI gate pattern.

const express = require('express');
const router = express.Router();

// GET /health — used by CI smoke tests, load balancers, and container orchestrators
router.get('/health', (req, res) => {
  const dbConnection = req.app.get('dbConnection');   // pulled from app-level state

  // Report unhealthy if the DB connection isn't ready — CI/CD pipelines and
  // production monitors both rely on this endpoint responding correctly
  if (!dbConnection || !dbConnection.isConnected()) {
    return res.status(503).json({ status: 'unhealthy', reason: 'no db connection' });
  }

  res.status(200).json({ status: 'ok', uptime: process.uptime() });
});

module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Committing secrets directly into the workflow file

```yaml
# ❌ WRONG — hardcoded credentials committed to git history forever,
# visible to anyone with repo access (or anyone who finds the leaked commit)
- name: Log in to Docker Hub
  run: docker login -u myuser -p SuperSecret123 docker.io
```

```yaml
# ✅ CORRECT — store secrets in GitHub repo Settings → Secrets and variables → Actions
# then reference them; they are encrypted and masked in logs
- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### Mistake 2 — Using `npm install` instead of `npm ci` in CI

```yaml
# ❌ WRONG — 'npm install' can update package-lock.json and pull newer
# minor/patch versions than what's actually tested and deployed locally
- name: Install dependencies
  run: npm install
```

```yaml
# ✅ CORRECT — 'npm ci' installs the EXACT versions from package-lock.json,
# deletes node_modules first, and fails if the lockfile is out of sync
- name: Install dependencies
  run: npm ci
```

### Mistake 3 — Pushing the Docker image even when tests fail

```yaml
# ❌ WRONG — build and push job has no dependency on the test job,
# so a broken build can still ship a bad image to production
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build-and-push:
    runs-on: ubuntu-latest        # runs in parallel with 'test', doesn't wait for it!
    steps:
      - run: docker build -t myorg/user-api .
      - run: docker push myorg/user-api
```

```yaml
# ✅ CORRECT — 'needs: test' forces build-and-push to wait, and to SKIP
# entirely if the test job fails
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build-and-push:
    needs: test                    # will not start until 'test' succeeds
    runs-on: ubuntu-latest
    steps:
      - run: docker build -t myorg/user-api .
      - run: docker push myorg/user-api
```

---

## Practice exercises

### Exercise 1 — easy

Create a workflow file `.github/workflows/ci.yml` for a simple Node.js backend project. It should:
1. Trigger on every `push` to any branch and on every `pull_request` targeting `main`
2. Run on `ubuntu-latest`
3. Check out the repository code
4. Set up Node.js version 20 with npm caching enabled
5. Install dependencies using the lockfile-exact command
6. Run the project's test script

```yaml
# Write your code here
```

---

### Exercise 2 — medium

Extend the workflow from Exercise 1 into two separate jobs:
1. A `lint-and-test` job that installs dependencies, runs a linter (`npm run lint`), and runs tests (`npm test`)
2. A `build` job that depends on `lint-and-test` succeeding, checks out the code again, and builds a Docker image tagged with the current git commit SHA (`${{ github.sha }}`) — do not push it anywhere yet, just build it locally on the runner
3. Make sure the `build` job is genuinely blocked from running if `lint-and-test` fails

```yaml
# Write your code here
```

---

### Exercise 3 — hard

Design a full workflow `.github/workflows/deploy.yml` for a production backend service that:
1. Triggers only on pushes to `main`
2. Has a `test` job that runs the full test suite against a real PostgreSQL service container (use the `services:` key to spin up a `postgres` container available to the test job, with environment variables for the connection)
3. Has a `build-and-push` job that depends on `test`, logs into Docker Hub using repo secrets, builds an image tagged both with the commit SHA and with `latest`, and pushes both tags
4. Has a `notify` job that depends on `build-and-push` and runs a step that prints a success message including the commit SHA and the pushed image tag (simulating a Slack/Discord notification step)
5. Uses `needs:` correctly so each job only runs after its dependency succeeds, and the whole pipeline fails fast if any earlier job fails

```yaml
# Write your code here
```

---

## Quick reference cheat sheet

```
FILE LOCATION
  .github/workflows/*.yml   → the ONLY place GitHub looks for workflow files
  One repo can have many workflow files, each independent

TOP-LEVEL KEYS
  name:   → label shown in the Actions tab
  on:     → events that trigger the workflow (push, pull_request, schedule, workflow_dispatch)
  jobs:   → one or more jobs, each gets its own fresh VM

TRIGGERS
  on: push                         → any push
  on: push: branches: [main]       → push only to main
  on: pull_request: branches: [main] → PRs targeting main
  on: workflow_dispatch            → manual "Run workflow" button in GitHub UI
  on: schedule: - cron: '0 0 * * *' → runs on a cron schedule

JOB STRUCTURE
  runs-on: ubuntu-latest   → the VM image
  steps:                   → ordered list of actions/commands
  needs: <job-id>          → wait for another job to succeed first (build order)

STEP TYPES
  uses: owner/action@version   → run a reusable, pre-built Action
  run: <shell command>          → run a raw shell command in the VM

COMMON ACTIONS
  actions/checkout@v4        → clone the repo into the runner
  actions/setup-node@v4      → install a specific Node.js version
  docker/login-action@v3     → authenticate the Docker CLI
  docker/build-push-action@v5 → build + push an image in one step

SECRETS
  Settings → Secrets and variables → Actions → New repository secret
  Reference with: ${{ secrets.SECRET_NAME }}
  NEVER hardcode passwords/tokens directly in the YAML file

npm ci  vs  npm install
  npm ci      → exact versions from package-lock.json, deletes node_modules first — USE IN CI
  npm install → can bump versions, slower, less reproducible — for local dev only

USEFUL CONTEXT VARIABLES
  ${{ github.sha }}      → the commit hash that triggered the workflow
  ${{ github.ref }}      → the branch/tag ref, e.g. refs/heads/main
  ${{ secrets.X }}       → an encrypted repo secret

DEBUGGING
  Actions tab on GitHub  → see every run, every step's logs, red X or green check
  Re-run failed jobs     → button available directly on a failed run
```

---

## Connected topics

- **105 — Dockerizing a Node.js app** — the `Dockerfile` that the `docker build` step in this workflow actually builds from.
- **107 — Environment management** — CI/CD pipelines need to inject the right `NODE_ENV` and secrets for the environment being built (test, staging, prod).
- **86 — Testing Express routes** — the `npm test` step in every workflow here is running the supertest/Jest suites covered in that topic.
