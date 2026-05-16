# Topic 07 — Commits Done Right
### Atomic Commits, The Perfect Commit Message, Conventional Commits, and `--amend`

---

## Connection to Previous Topics

- **Topic 02** showed you that a commit object is a text file: `tree`, `parent`, `author`,
  `committer`, blank line, message. Today you fill in the "message" field like a professional.
- **Topic 03** showed you that commits form a DAG. Today you'll learn how the SHAPE of your
  commits (atomic vs monolithic) determines how useful that DAG actually is.
- **Topic 06** showed you the exact mechanics of `git commit`. Today is about what to commit,
  when, and how to write a message that makes that commit permanently useful.

This topic is less about bytes on disk and more about the **craft** of committing — but
every decision here has mechanical consequences at the team level (CI, bisect, revert,
blame). Those consequences are explained in full.

---

## RULE 1 — ELI5 Anchor

### The Surgeon's Notes Analogy

Imagine a surgeon who performs an operation and must write a medical record entry
afterward. There are two types of surgeons:

**Surgeon A** writes: *"Did stuff. Patient seems okay. —Dr. A, sometime Tuesday"*

**Surgeon B** writes:
```
Procedure: Removed 7mm gallstone from common bile duct
Technique: Laparoscopic cholecystectomy, 3 incisions
Reason: Patient presented with biliary colic; confirmed by ultrasound (Case #4521)
Result: Stone removed intact; bile duct irrigated; hemostasis achieved
Follow-up: Check liver enzymes at 48h; NPO for 8h post-op
—Dr. B, 2026-05-16 14:32, OR Suite 3
```

Six months later, when the patient comes back with a complication, which surgeon's notes
let the next doctor understand exactly what happened, why, and what to look for?

A Git commit is a medical record for your codebase. Every commit you write is a note
for:
- Future-you at 2 AM debugging a production incident
- A colleague reviewing your PR
- A new engineer onboarding in 6 months
- `git bisect` trying to find which exact change introduced a bug
- `git revert` trying to undo exactly one logical change cleanly

The cost of a bad commit message: zero, today.  
The cost of a bad commit message: potentially hours, six months from now.

---

## RULE 2 — Physical Architecture: What "Atomic" Means on Disk

### Monolithic vs. Atomic — Same Files, Different History Shape

Imagine you're building a user authentication system. You write:
- `src/auth/jwt.js` — JWT token generation
- `src/auth/middleware.js` — Express auth middleware
- `src/routes/auth.js` — Login/logout routes
- `test/auth.test.js` — Tests for the above
- `package.json` — Added `jsonwebtoken` dependency

**Approach A — One Big Commit:**
```
.git/objects/
  ├── aa/  ← commit object "Add entire auth system"
  ├── bb/  ← root tree
  ├── cc/  ← src/ tree
  ├── dd/  ← src/auth/ tree
  ├── ee/  ← src/routes/ tree
  ├── ff/  ← test/ tree
  ├── (blobs for all 5 files)
```

**Approach B — 3 Atomic Commits:**
```
Commit 1: "chore: add jsonwebtoken dependency" (package.json only)
Commit 2: "feat: implement JWT auth module and middleware" (jwt.js, middleware.js, routes/auth.js)
Commit 3: "test: add unit tests for auth module" (test/auth.test.js)

.git/objects/ has 3 commit objects, with proper parent chain:
  commit3 → commit2 → commit1
```

**The DAG shapes:**

Monolithic:
```
main ──► [auth system]──► [previous work]──► ...
```

Atomic:
```
main ──► [test: auth tests]──► [feat: auth module]──► [chore: jwt dep]──► ...
```

**Why the atomic shape is better — mechanical consequences:**

| Operation | Monolithic commit | Atomic commits |
|-----------|------------------|----------------|
| `git revert` | Reverts ALL 5 files — impossible if only the middleware is broken | Reverts exactly one logical unit |
| `git bisect` | Blame narrows to a 500-line commit | Blame narrows to a 50-line commit |
| Code review | Reviewer must context-switch between dep change, logic, and tests | Each commit has a single concern |
| `git cherry-pick` | Must take everything or nothing | Can cherry-pick just the middleware to another branch |
| CI failure analysis | "Which of the 5 files broke CI?" | CI log points to exactly one commit |

```bash
# PROVE IT: See commit sizes
git log --oneline --stat
# Shows files changed + insertions/deletions per commit
# Atomic commits: small, focused stat blocks
# Monolithic commit: large, mixed stat block

# Cherry-pick just one commit
git cherry-pick <commit-sha>  # Works cleanly with atomic commits
                              # Causes conflicts with monolithic ones
```

---

## The Anatomy of a Perfect Commit Message

### The Standard Format

```
<type>(<scope>): <subject>
                              ← blank line separator (REQUIRED)
<body>
                              ← blank line separator
<footer>
```

**The 50/72 Rule:**
- Subject line: **≤ 50 characters** (GitHub truncates at 72, `git log --oneline` at 80)
- Body lines: **hard-wrap at 72 characters** (terminal width, email clients, `git log`)

```bash
# PROVE IT: See how GitHub truncates long subjects
git log --pretty=format:"%h %<(50,trunc)%s" | head -10
# Subjects over 50 chars get truncated in this view
```

---

### The Subject Line — 5 Rules

**Rule 1: Capitalize the first letter**
```
✅  Add user authentication middleware
❌  add user authentication middleware
```

**Rule 2: Do NOT end with a period**
```
✅  Fix null pointer in payment processor
❌  Fix null pointer in payment processor.
```

**Rule 3: Use the IMPERATIVE mood ("Add", not "Added" or "Adds")**

Think of the subject line completing the sentence: *"If applied, this commit will ___"*

```
✅  "If applied, this commit will: Add user authentication middleware"
✅  "If applied, this commit will: Fix null pointer in payment processor"
✅  "If applied, this commit will: Refactor database connection pooling"

❌  "If applied, this commit will: Added user authentication middleware"  ← past tense
❌  "If applied, this commit will: Adding user authentication middleware" ← gerund
❌  "If applied, this commit will: Fixes null pointer..."                ← third person
```

Git itself uses imperative mood in system messages:
- "Merge branch 'feature/auth' into main"
- "Revert 'Add broken feature'"
- "Initial commit"

**Rule 4: Keep it under 50 characters**
```
✅  Add JWT authentication (30 chars)
✅  Fix null ref in payment processor (34 chars)
❌  Added the new user authentication system with JWT tokens and Redis sessions (74 chars)
    → Split into subject + body
```

**Rule 5: Don't write WHAT changed — Git already knows that from the diff. Write WHY.**

```
❌  Change constant from 100 to 250
✅  Increase connection pool limit to prevent timeout under peak load

❌  Update README
✅  Add production deployment instructions to README

❌  Fix bug
✅  Prevent duplicate invoice generation when webhook is retried
```

---

### The Commit Body — When and How

The body is optional but should be included when:
- The subject alone doesn't explain WHY
- There are important context details (performance implications, architecture decisions)
- You're referencing related issues, tickets, or discussions
- A workaround was needed and future maintainers should understand the constraint

**What to write in the body:**
- WHY the change was necessary (what problem does it solve?)
- HOW the problem was solved at a high level (not code details — the diff shows that)
- Any alternatives considered and why they were rejected
- Side effects or limitations of this approach
- Links to specifications, issue trackers, Stack Overflow threads

**Example of a body-worthy commit:**
```
fix: prevent duplicate invoice generation on webhook retry

Stripe webhooks can be delivered multiple times if our server returns
a non-2xx response. Previously, we were generating a new invoice on
every delivery of checkout.session.completed, causing duplicate charges.

Solution: Added an idempotency key using the Stripe event ID. The
invoice is only created if no invoice with that event ID already exists
in the database. Existing invoices return 200 without side effects.

Alternative considered: Redis-based deduplication lock. Rejected because
the database check is sufficient and avoids adding Redis as a dependency
for this one case.

Note: This does not fix past duplicate invoices — a separate migration
script (scripts/deduplicate-invoices.js) handles historical data.

Closes #891.
Co-authored-by: Priya Sharma <priya@company.com>
```

**Format rules for the body:**
- Separate from subject with ONE blank line (Git's parser requires this)
- Wrap lines at 72 characters
- Use bullet points sparingly — paragraphs read more naturally
- Don't just repeat the diff in English

---

### The Footer — References, Breaking Changes, Co-authors

```
Closes #123              ← GitHub/GitLab auto-close issue on merge
Fixes #123               ← Same as Closes
Refs #123                ← Reference without closing
Resolves #123            ← Same as Closes (GitHub supports all three)

BREAKING CHANGE: <description>  ← Conventional Commits: signals major version bump
                                   Must be in footer or as ! after type/scope

Co-authored-by: Name <email>    ← Attribution for pair-programming / AI assistance
Reviewed-by: Name <email>       ← Optional attribution
```

**Multi-issue reference:**
```
Closes #123, #124
Refs: https://docs.stripe.com/webhooks/signatures
```

---

## RULE 9 — The Commit Object: What Your Message Becomes on Disk

```
commit 378\0
tree 4b825dc642cb6eb9a060e54bf8d69288fbee4904
parent d8329fc1cc938780ffdd9f94e0d364e0ea74f579
author Parsh Khandelwal <parsh@example.com> 1715894400 +0530
committer Parsh Khandelwal <parsh@example.com> 1715894400 +0530

fix: prevent duplicate invoice generation on webhook retry

Stripe webhooks can be delivered multiple times...
[body text...]

Closes #891.
```

The **entire message** (subject + blank line + body + blank line + footer) is stored as
the last field of the commit object, separated from the header metadata by a blank line.

```bash
# PROVE IT: Read your commit message from the object
git cat-file commit HEAD
# The last section after the blank line is your full message

# See just the message
git log -1 --pretty=format:"%B"
# %B = raw body (subject + body + footer)

# See subject only
git log -1 --pretty=format:"%s"

# See body only (everything after subject)
git log -1 --pretty=format:"%b"
```

---

## Conventional Commits Specification

### What It Is and Why It Exists

Conventional Commits is a specification (conventionalcommits.org) that adds a
machine-readable layer to commit messages. The goal: tools can parse your commit history
to automatically:
- Generate a **CHANGELOG** (what changed between v1.2.0 and v1.3.0?)
- Determine the **next semantic version** (is this a patch, minor, or major bump?)
- Filter commits by type in `git log`
- Trigger different CI behaviors per commit type

Format:
```
<type>[optional scope][optional !]: <description>

[optional body]

[optional footer(s)]
```

---

### The Commit Types

| Type | Meaning | SemVer Impact | Example |
|------|---------|---------------|---------|
| `feat` | New feature for the user | MINOR (1.x.0) | `feat: add dark mode toggle` |
| `fix` | Bug fix for the user | PATCH (1.0.x) | `fix: prevent crash on empty user list` |
| `chore` | Build process, tooling, deps | none | `chore: upgrade eslint to v9` |
| `docs` | Documentation changes | none | `docs: add API rate limiting section` |
| `style` | Formatting, whitespace (no logic change) | none | `style: reformat auth module per ESLint` |
| `refactor` | Code restructuring (no bug fix, no feature) | none | `refactor: extract payment logic into service` |
| `perf` | Performance improvements | PATCH | `perf: add index on users.email column` |
| `test` | Adding or fixing tests | none | `test: add unit tests for JWT validation` |
| `build` | Changes to build system | none | `build: switch from webpack to vite` |
| `ci` | Changes to CI config | none | `ci: add staging deployment workflow` |
| `revert` | Reverts a previous commit | depends | `revert: feat: add dark mode toggle` |

---

### Scope: The Optional Context

```
feat(auth): add OAuth2 login with Google
fix(payments): prevent double-charge on retry
chore(deps): upgrade stripe SDK to v14
test(api): add integration tests for /users endpoint
```

The scope goes in parentheses after the type. It's a noun describing what AREA of the
codebase was affected. Common scope names: `auth`, `api`, `ui`, `db`, `payments`,
`cli`, `config`, `build`, `ci`, `docs`.

**When to use scope:**
- Medium+ sized projects with multiple subsystems
- Monorepos where changes affect specific packages
- When the type alone isn't specific enough

**When to skip scope:**
- Small projects with few subsystems
- When the subject line makes the scope obvious

---

### Breaking Changes

A **breaking change** is any change that requires users of your API/library to update
their code. In SemVer, breaking changes require a MAJOR version bump (x.0.0).

**Two ways to signal a breaking change:**

Method 1 — `!` after type/scope:
```
feat!: remove support for Node.js 16
refactor(auth)!: require JWT instead of session cookies
```

Method 2 — `BREAKING CHANGE:` in footer:
```
feat(auth): replace session-based auth with JWT

BREAKING CHANGE: All existing session tokens are invalidated.
Clients must re-authenticate and store the JWT in their
Authorization header instead of a session cookie.

Migration guide: https://docs.example.com/auth-migration
```

Both methods cause tools to flag this as a MAJOR version bump.

---

### Automated Changelog Example

Given these commits between v1.2.0 and v1.3.0:
```
feat(auth): add OAuth2 login with Google
feat(payments): add Apple Pay support
fix(invoices): prevent duplicate generation on webhook retry
fix(users): correct email validation regex for .museum TLDs
chore(deps): upgrade stripe SDK to v14
docs: update API authentication guide
perf(db): add composite index on orders table
```

A tool like `standard-version` or `semantic-release` produces:

```markdown
# Changelog

## [1.3.0] - 2026-05-16

### Features
- **auth:** add OAuth2 login with Google
- **payments:** add Apple Pay support

### Bug Fixes
- **invoices:** prevent duplicate generation on webhook retry
- **users:** correct email validation regex for .museum TLDs

### Performance Improvements
- **db:** add composite index on orders table
```

The `chore` and `docs` commits are excluded from the changelog automatically.

---

### SemVer Version Bump Logic

| Commits since last release | Next version (from 1.2.3) |
|---------------------------|--------------------------|
| Only `fix:` commits | 1.2.4 (PATCH) |
| At least one `feat:` commit | 1.3.0 (MINOR) |
| Any `BREAKING CHANGE` or `!` | 2.0.0 (MAJOR) |

```bash
# Tools that automate this:
npx standard-version           # Bump version, generate CHANGELOG, create tag
npx semantic-release           # CI-driven fully automated releases
npx conventional-changelog-cli # Just generate the CHANGELOG
npx commitlint                 # Validate commit messages in CI
```

---

## RULE 4 — ASCII Architecture Diagram: Commit Quality at Scale

```
Timeline: 6 months of commits

BAD history (monolithic, cryptic messages):
────────────────────────────────────────────
  [WIP]──►[fix]──►[stuff]──►[update]──►[final fix]──►[actually final]──►...
    │                │
    │                └─ 400 lines changed in 8 files. What was fixed? Unknown.
    └─ 1200 lines changed in 15 files. Where does auth end and routing begin?

GOOD history (atomic, conventional):
──────────────────────────────────────────────────────────────────────────
  [chore: add jsonwebtoken]
      ──►[feat(auth): add JWT generation and validation]
            ──►[feat(auth): add auth middleware for protected routes]
                  ──►[test(auth): add unit tests for JWT module]
                        ──►[fix(auth): handle expired token gracefully]
                              ──►[docs(auth): add authentication guide]

What you can do with good history:
  git log --grep="feat(auth)"     → See all auth features
  git bisect                      → Narrow bug to one 50-line commit
  git revert <sha>                → Undo exactly the expired token fix
  git log --oneline --follow auth → See auth's full history
  semantic-release                → Automatically bump version + CHANGELOG
```

---

## Deep Dive: `git commit --amend`

### What It Does (Recap from Topic 06, Expanded)

`--amend` replaces the most recent commit. It:
1. Creates a **brand new commit object** (different SHA)
2. Moves the branch pointer to the new commit
3. Leaves the old commit as an orphan (still in `.git/objects/` until GC)

**The 3 use cases:**

---

### Use Case 1: Fix the commit message only

```bash
git commit --amend
# Opens editor with the current message
# Edit and save → new commit SHA, same tree

# OR inline:
git commit --amend -m "fix: prevent null ref in payment processor"
```

**Data flow:**
```
BEFORE:               AFTER:
abc123 (HEAD/main)    def456 (HEAD/main)
  tree: xyz           →  tree: xyz (SAME tree)
  message: "oops"        message: "fix: prevent null ref..."
                         parent: same parent as abc123
                      abc123: now an orphan
```

---

### Use Case 2: Add a forgotten file to the last commit

```bash
# You committed, then realized you forgot to stage a file
git add forgotten-file.js
git commit --amend --no-edit
# --no-edit: keep the same message, no editor opened
```

**Data flow:**
```
BEFORE amend:          AFTER amend:
Index:                 Index:
  [original files]       [original files + forgotten-file.js]
  + forgotten-file.js    (same as before)
  
Old commit abc123:     New commit def456:
  tree: (missing file)   tree: (includes file)
  message: "Add module"  message: "Add module" (unchanged)
```

---

### Use Case 3: Fix a small mistake in the last commit's content

```bash
# You committed with a typo in server.js
# Fix the typo:
vim server.js

# Stage the fix:
git add server.js

# Amend the commit (keeps message, adds fix):
git commit --amend --no-edit
# Now the commit contains the corrected server.js
# It's as if the typo never happened
```

---

### The Golden Rule of `--amend`

```
NEVER amend a commit that has been pushed to a shared branch.
```

**Why?** Your local branch and the remote branch now have different commit SHAs for what
was "the same commit." When you push:

```
$ git push origin main
! [rejected]  main -> main (non-fast-forward)
  hint: Updates were rejected because the tip of your current branch is behind
  hint: its remote counterpart.
```

Git refuses the push because you've rewritten history. If you force push (`--force`):
- Everyone who pulled the original commit has it in their history
- Their `git pull` will fail with "refusing to merge unrelated histories"
- They must `git reset --hard origin/main` (discarding any local work built on top)
- Team chaos ensues

**The only exception**: Fixing a commit you pushed to YOUR OWN feature branch that
no one else has pulled yet. Even then, communicate with your team first.

```bash
# Safe: amend on a private feature branch you haven't shared
git push origin feature/my-work --force-with-lease
# --force-with-lease is safer than --force:
# it checks that no one else pushed to the remote branch since your last fetch
# if someone else pushed: it REFUSES instead of silently overwriting their work
```

---

## Deep Dive: `git revert` for "Safe Undo" in Shared History

When a commit HAS been pushed and shared, you can't amend it. Instead, use `git revert`:

```bash
git revert <commit-sha>
```

This creates a NEW commit that is the exact inverse of the target commit. The original
commit stays in history. This is "safe undo" — it doesn't rewrite history.

```
BEFORE revert:
  C──►D──►E  (HEAD/main)
  
  D introduced the bug.

git revert D

AFTER revert:
  C──►D──►E──►D'  (HEAD/main)
  
  D'  is the inverse of D.
  D is still in history (audit trail preserved).
  HEAD is clean (D's changes are undone).
```

**When to use `revert` vs `reset`:**

| Situation | Command | Safe? |
|-----------|---------|-------|
| Undo last commit (not pushed) | `git reset --soft HEAD~1` | ✅ Local only |
| Undo last commit (pushed, shared) | `git revert HEAD` | ✅ Safe for shared |
| Undo any past commit (not pushed) | `git reset --soft <sha>` | ✅ Local only |
| Undo any past commit (pushed) | `git revert <sha>` | ✅ Safe for shared |

---

## Deep Dive: Writing Great Commit Messages for Special Situations

### The Initial Commit

```
feat: initial project setup

Bootstrap Node.js Express API with:
- TypeScript configuration
- ESLint + Prettier
- Jest for testing
- GitHub Actions CI workflow
- Docker development environment

Project requirements: docs/requirements.md
Architecture overview: docs/architecture.md
```

---

### The Hotfix Commit

Short, precise, references the incident:
```
fix(payments): prevent null ref when order has no shipping address

getUserShippingCost() assumed order.shippingAddress was always defined.
Orders with digital-only products have no shipping address. Added null
guard before accessing address.country.

Fixes production incident #1042. 
Refs: https://incident-tracker.company.com/INC-1042
```

---

### The Refactor Commit

Explain that behavior is PRESERVED:
```
refactor(db): extract user queries into UserRepository class

No behavior change — moves SQL queries from UserService into a dedicated
repository class to improve separation of concerns.

UserService now depends on UserRepository (injected via constructor).
All existing tests pass without modification.
```

---

### The Performance Commit

Include numbers:
```
perf(api): add Redis cache for user profile endpoint

GET /api/users/:id was hitting the database on every request.
Added 60-second Redis cache keyed on user ID.

Benchmark results (1000 requests, p95 latency):
  Before: 280ms
  After:   12ms

Cache is invalidated on user profile update via cache.del() in
UserService.updateProfile().
```

---

### The Chore/Dependency Update

```
chore(deps): upgrade stripe SDK from v12 to v14

Changelog: https://github.com/stripe/stripe-node/blob/master/CHANGELOG.md
Breaking changes affecting us: None (we only use PaymentIntent API)
Tested: all payment integration tests pass
```

---

### The Revert Commit

```
revert: feat(payments): add Apple Pay support

Reverts commit a1b2c3d.

Reason: Apple Pay integration is causing 15% failure rate on iOS 17.
Rolling back while investigation is in progress.

Incident: #1087
Original PR: #234
```

---

## RULE 6 — Syntax Breakdown: Every `git commit` Flag Relevant to Craft

```
git commit  [-m <msg>]          ← message inline (skip editor)
            [--amend]           ← replace last commit
            [--no-edit]         ← with --amend: don't open editor
            [-v]                ← show diff in editor during message writing
            [-a]                ← auto-stage tracked changes (skip git add)
            [--fixup=<sha>]     ← create "fixup! original subject" commit
                                   (used with git rebase --autosquash)
            [--squash=<sha>]    ← create "squash! original subject" commit
            [--allow-empty]     ← commit even if nothing changed
            [--no-verify / -n]  ← skip pre-commit and commit-msg hooks
            [-C <sha>]          ← reuse message + author from another commit
            [-c <sha>]          ← reuse from another commit, but open editor
            [--reset-author]    ← with --amend: update author to current config
            [--author="N <e>"]  ← override the commit's author field
            [--date=<date>]     ← override the author date
            [--signoff / -s]    ← append "Signed-off-by: Name <email>" to message
                                   (required by some OSS projects — Linux kernel)
            [--gpg-sign / -S]   ← GPG sign the commit (for verified commits on GitHub)
            [--trailer <token:value>] ← add trailer line to message footer
```

---

## RULE 5 — Hands-On Proof: The Full Professional Workflow

### Setup: Simulate a real feature branch workflow

```bash
mkdir pro-commit-demo && cd pro-commit-demo
git init
git config user.name "Parsh Khandelwal"
git config user.email "parsh@example.com"

# Simulate pre-existing project
echo '{"name": "demo", "version": "1.0.0"}' > package.json
echo "# Demo Project" > README.md
git add .
git commit -m "chore: initial project structure"

# Create feature branch
git checkout -b feature/user-auth
```

### Step 1: Atomic commit for dependency

```bash
# Add dependency line to package.json
cat >> package.json << 'EOF'
{
  "dependencies": {
    "jsonwebtoken": "^9.0.0"
  }
}
EOF

git add package.json
git commit -m "chore(deps): add jsonwebtoken dependency for JWT auth"
```

### Step 2: Feature commit for core logic

```bash
mkdir -p src/auth
cat > src/auth/jwt.js << 'EOF'
const jwt = require('jsonwebtoken');

const SECRET = process.env.JWT_SECRET;

function generateToken(userId) {
  return jwt.sign({ userId }, SECRET, { expiresIn: '24h' });
}

function verifyToken(token) {
  return jwt.verify(token, SECRET);
}

module.exports = { generateToken, verifyToken };
EOF

git add src/auth/jwt.js
git commit -m "feat(auth): implement JWT token generation and validation

Uses RS256-equivalent security via a strong secret from JWT_SECRET env var.
Token TTL is 24 hours. verifyToken throws JsonWebTokenError on invalid/
expired tokens — callers should handle this in middleware.

Note: In production, JWT_SECRET must be at least 256 bits of entropy."
```

### Step 3: Notice a typo in the last commit

```bash
# You realize "RS256-equivalent" is misleading — it's actually HS256
git commit --amend -m "feat(auth): implement JWT token generation and validation

Uses HS256 algorithm with a symmetric secret from JWT_SECRET env var.
Token TTL is 24 hours. verifyToken throws JsonWebTokenError on invalid/
expired tokens — callers should handle this in middleware.

Note: In production, JWT_SECRET must be at least 256 bits of entropy."

# Verify the amend
git log -1 --pretty=format:"%B"
```

### Step 4: Tests

```bash
mkdir -p test
cat > test/jwt.test.js << 'EOF'
const { generateToken, verifyToken } = require('../src/auth/jwt');
process.env.JWT_SECRET = 'test-secret-min-32-chars-for-test';

test('generateToken returns a string', () => {
  expect(typeof generateToken('user123')).toBe('string');
});

test('verifyToken returns payload with userId', () => {
  const token = generateToken('user123');
  const payload = verifyToken(token);
  expect(payload.userId).toBe('user123');
});
EOF

git add test/jwt.test.js
git commit -m "test(auth): add unit tests for JWT generation and verification"
```

### Step 5: Review the commit history

```bash
git log --oneline
# 3 clean, atomic, purposeful commits

git log --stat
# Each commit shows only the files it logically touched

git log -1 --pretty=format:"%B"
# Shows the full message of the last commit

# Simulate what semantic-release would compute:
# chore + feat + test → MINOR version bump (feat is highest)
```

---

## RULE 7 — Beginner Example

### Your First Project: A Simple Calculator

```bash
mkdir calculator && cd calculator
git init

# File 1: core logic
cat > calc.js << 'EOF'
function add(a, b) { return a + b; }
function subtract(a, b) { return a - b; }
module.exports = { add, subtract };
EOF

# BAD approach (beginner mistake):
git add .
git commit -m "stuff"

# ---
# GOOD approach:
git add calc.js
git commit -m "feat: add basic calculator with add and subtract operations"

# Add multiplication later:
cat >> calc.js << 'EOF'
function multiply(a, b) { return a * b; }
EOF

git add calc.js
git commit -m "feat: add multiply operation to calculator"

# Fix a bug:
# (subtract was wrong — forgot to handle negative numbers properly)
git add calc.js
git commit -m "fix: ensure subtract handles negative number inputs correctly"
```

Result: A git log that reads like a feature list:
```
abc feat: add multiply operation to calculator
def fix: ensure subtract handles negative number inputs correctly
ghi feat: add basic calculator with add and subtract operations
```

---

## RULE 7 — Production Example

### Scenario: PR Review Feedback Drove You to `--amend`

You open a PR with 3 commits:
```
abc123 feat(checkout): add cart total calculation
def456 feat(checkout): add checkout form validation
ghi789 test(checkout): add tests for checkout flow
```

PR reviewers find two issues in commit `abc123`:
- Missing currency formatting function
- Typo in variable name `totlPrice` should be `totalPrice`

**Response A (bad):** Add a new commit "fix typos and missing function"
```
jkl012 fix typos and missing function   ← pollutes history; meaningless
```

**Response B (good):** Fix the issues and amend the original commit

```bash
# Make the fixes:
vim src/checkout/cart.js  # fix totalPrice and add formatCurrency()

# Stage only the cart.js fixes
git add src/checkout/cart.js

# Amend abc123 (which is NOT the latest commit — requires rebase)
# (Full coverage of interactive rebase in Topic 11)
git rebase -i HEAD~3
# Change abc123's "pick" to "edit"
# Git pauses at abc123
git add src/checkout/cart.js  # (already staged above conceptually)
git commit --amend --no-edit
git rebase --continue

# Force push the updated branch
git push origin feature/checkout --force-with-lease
```

Result: The PR history now shows `abc123` was always correct — the revision history is
clean and the commit tells the right story without noise.

---

## Tools That Enforce Commit Quality

### commitlint — Validate Messages in CI

```bash
npm install -D @commitlint/cli @commitlint/config-conventional

# .commitlintrc.json
{
  "extends": ["@commitlint/config-conventional"]
}

# GitHub Actions workflow:
# .github/workflows/lint-commits.yml
name: Lint commits
on: [pull_request]
jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: wagoid/commitlint-github-action@v5
```

### Husky + commitlint — Validate at commit time

```bash
npm install -D husky
npx husky init
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg

# Now git commit -m "bad message" fails immediately:
# ⧗   input: bad message
# ✖   subject may not be empty [subject-empty]
# ✖   type may not be empty [type-empty]
# ✖   found 2 problems, 0 warnings
```

### git log with format — Filter by conventional type

```bash
# See all features since v1.2.0 tag
git log v1.2.0..HEAD --oneline --grep="^feat"

# See all bug fixes since last month
git log --since="1 month ago" --oneline --grep="^fix"

# See all breaking changes ever
git log --all --oneline --grep="BREAKING CHANGE"
git log --all --oneline --grep="!:"
```

---

## RULE 8 — Mistakes, Root Causes, and Fixes

### Mistake 1: "WIP", "fixes", "update", "misc" Commit Messages

**The symptom:**
```
git log --oneline
e3a4b2c misc
d1f9a8b fixes
c7e2f1a update
b5d0e3a wip
a4c1d2e stuff
```

**Root cause:** Treating commits as saves in a video game (pressing Ctrl+S) rather than
as permanent documentation entries.

**The cost, realized 3 months later:**
```bash
git log --all --oneline --grep "fix" | head -5
# Nothing useful

git bisect start
git bisect bad HEAD
git bisect good v1.0.0
# Git navigates to "wip" commit — what broke? No idea. Time wasted.
```

**Fix (if not yet pushed — interactive rebase):**
```bash
git rebase -i HEAD~5
# Change each "pick" to "reword" on the WIP commits
# Git opens editor for each one — write real messages
```

**Prevention:**
- Set a personal rule: if you can't write a subject in 50 chars that describes the change,
  your change is too big or too vague
- Use `git commit -v` to see what you're committing before typing the message
- If you genuinely need a WIP save mid-session, use `git stash` instead (Topic 13)

---

### Mistake 2: One Giant "Add authentication system" Commit

**The symptom:**
```
git show abc123 --stat
  src/auth/jwt.js            |  120 insertions
  src/auth/middleware.js     |   85 insertions
  src/routes/auth.js         |   60 insertions
  src/models/session.js      |   45 insertions
  test/auth.test.js          |  200 insertions
  package.json               |    5 insertions
  package-lock.json          |  980 insertions
  6 files changed, 1495 insertions(+)
```

**Root cause:** Developing the whole feature, then committing everything at once. Common
when developers think of committing as "done" rather than as "checkpoint."

**The cost:** If the auth system has a bug:
- `git bisect` points to a 1495-line commit — you still have to manually find the bug
- `git revert abc123` reverts everything — you lose all the auth work
- Code reviewer sees 1495 lines at once — impossible to review meaningfully

**Fix (if not yet pushed):**
```bash
# Use git add -p to selectively stage and commit in pieces
git reset HEAD~1          # unstage everything from that commit (soft reset)
git add -p                # stage logically related hunks
git commit -m "chore(deps): add jsonwebtoken"
git add src/auth/jwt.js
git commit -m "feat(auth): implement JWT core module"
# etc.
```

**Prevention:**
- Commit as you work, not when you're "done"
- After implementing each logical unit, stop and commit it before moving on

---

### Mistake 3: Amended a Pushed Commit

**The symptom:**
```bash
git push origin main
# error: failed to push some refs to 'origin/github.com/...'
# hint: Updates were rejected because the tip of your current branch is behind
```

**Root cause:** `git commit --amend` creates a new SHA. Your local `main` diverged from
`origin/main` the moment you amended a commit that was already on the remote.

**Fix:**
```bash
# Check what's different
git log --oneline origin/main..HEAD
# Shows your amended commit

git log --oneline HEAD..origin/main
# Shows nothing (you're ahead, not behind — it's a divergence, not a lag)

# Option 1 (if you're SURE no one else pulled):
git push --force-with-lease origin main
# Safer than --force: checks remote hasn't changed since your last fetch

# Option 2 (if someone else may have pulled):
# DON'T rewrite history — create a new commit with the fix instead
git revert HEAD
git commit -m "fix: correct the mistake from previous commit"
```

**Prevention:**
- Only amend commits that exist ONLY on your local machine
- Remember: as soon as you `git push`, that commit is shared

---

### Mistake 4: Forgetting the Blank Line Between Subject and Body

**The symptom:**
```
git log --oneline
abc1234 Fix null pointer issue Stripe webhook retries cause duplicate invoices...
```

Git merged the subject and body because there was no blank line between them.

```
WRONG (no blank line):
fix: prevent null pointer in payment processor
Stripe webhooks can deliver events multiple times...

CORRECT (blank line):
fix: prevent null pointer in payment processor

Stripe webhooks can deliver events multiple times...
```

**Root cause:** The blank line is REQUIRED by Git's commit message parser. Without it,
the entire message is treated as the subject line. `git log --oneline` shows it all on
one line. `git log --pretty=format:"%b"` returns empty.

**Fix:**
```bash
git commit --amend
# Edit the message in the editor, add the blank line
```

**Prevention:** Always use `git commit` (without `-m`) to write multi-line messages.
Using `-m` for multi-line messages requires explicit `\n` escaping which is error-prone.

---

### Mistake 5: Using `git commit -a` on a Repo with .env Files

**The symptom:**
```bash
git commit -a -m "feat: add database configuration"
# Oops — .env was tracked (not in .gitignore) and got committed
# .env contains: DB_PASSWORD=secret123
```

**Root cause:** `-a` stages ALL tracked modified files. If `.env` was tracked (committed
before being added to `.gitignore`), it gets included.

**Fix (if not pushed):**
```bash
# Amend to remove .env from the commit
git rm --cached .env
git commit --amend --no-edit
# Now the commit doesn't contain .env

# Add to .gitignore
echo ".env" >> .gitignore
git add .gitignore
git commit -m "chore: add .env to .gitignore"
```

**Fix (if pushed — secrets are compromised):**
1. IMMEDIATELY rotate the exposed secret (change DB_PASSWORD, API key, etc.)
2. Remove from history using `git filter-repo` (Topic 25)
3. Force push the cleaned history
4. Require all collaborators to reclone

**Prevention:** Never use `git commit -a` on repos with sensitive files.
Always use `git diff --cached` to review staged content before committing.

---

## RULE 10 — Mental Model Checkpoint

**1.** What does "imperative mood" mean in the context of a commit subject line? Give
three examples of bad phrasing and their corrected imperative versions.

**2.** You have 3 changes in 3 files: a dependency addition in `package.json`, a new
feature in `src/feature.js`, and tests for that feature in `test/feature.test.js`.
How many commits should you make? What should each message say?

**3.** What is the 50/72 rule? What breaks if you violate each limit?

**4.** What is a Conventional Commit? If you have commits `feat:`, `fix:`, and `chore:`
since v1.2.3, what will the next version be and why?

**5.** You ran `git commit --amend` to fix a typo in your last commit message. You then
do `git push origin main` and get a rejection error. What happened? What are your two
options?

**6.** What is `BREAKING CHANGE:` in a commit footer? What version bump does it trigger?
Show what the commit message looks like for a breaking API change.

**7.** You're on a team that uses `commitlint`. Someone tries to push a commit with
message "updated stuff". What happens? Where does the validation run (locally vs CI)?

**8.** `git commit --fixup=<sha>` creates a commit with message `"fixup! original subject"`.
What is this used for? What other command consumes it?

**9.** Why does `git bisect` work better with atomic commits? Be specific — describe
exactly what happens with a monolithic vs atomic commit history when bisect narrows
to "the bad commit."

**10.** What information goes in the commit BODY vs the commit FOOTER? Give an example
where something could go in either place, and explain which is more appropriate and why.

---

## RULE 11 — Connections to Previous Topics

**From Topic 02** — Git Objects:
- The commit body (your message) is literally the last field of the commit object stored
  in `.git/objects/`
- A better message doesn't change the object format — only the content of that last field

**From Topic 03** — The DAG:
- Atomic commits shape the DAG into a useful, readable structure
- Every operation that traverses the DAG (`git bisect`, `git log --grep`, `git revert`)
  works better with atomic, well-named commits
- A branch's PR history should be a sequence of atomic commits that each advance the
  feature by one logical step

**From Topic 05** — The Three Trees:
- `git add --patch` is the mechanism that enables atomic commits from mixed work sessions
- Each `git commit` you make should represent one complete, intentional snapshot of the
  Index — the subject line should describe exactly what's different in this snapshot

**From Topic 06** — `git commit` mechanics:
- `--amend` physically creates a new commit object and moves the branch pointer
- The old commit becomes an orphan (reachable only via `git reflog` until GC)
- `COMMIT_EDITMSG` stores the last message — used by `-C HEAD` and `--reuse-message`

---

## RULE 13 — When Would I Use This at Work?

### Scenario A: The Release Engineering Review

Your company does weekly releases. The release engineer reads `git log --oneline main`
to understand what changed since last release and decide the version number.

With conventional commits:
```bash
git log v1.5.2..HEAD --oneline
# feat(api): add bulk user import endpoint
# feat(ui): add dark mode toggle
# fix(payments): prevent double charge on retry
# chore(deps): upgrade Node.js to 22 LTS
# perf(db): add index on orders.created_at
```

Release engineer immediately knows:
- Two new features → MINOR bump: 1.6.0
- CHANGELOG writes itself with `standard-version`
- Deploys with confidence

---

### Scenario B: The Production Incident Post-Mortem

At 3 AM, payments are failing in production. The on-call engineer runs:

```bash
git log --oneline --since="48 hours ago"
# a1b2c3d fix(payments): switch from sync to async processing
# d4e5f6g chore(deps): upgrade stripe SDK to v14
```

Only 2 commits in 48 hours. The engineer can quickly check BOTH in under 5 minutes.

If instead those 2 commits were one "update payment system" monolith with 800 lines,
the debugging session takes 2 hours.

---

### Scenario C: Code Review Culture

Your PR contains 8 atomic commits. Each commit:
- Has a clear subject line explaining what it does
- Has a body explaining WHY
- Only touches one logical concern

The reviewer can:
1. Review commit by commit — each one is reviewable in under 5 minutes
2. Leave comments on specific commits (GitHub shows per-commit diff)
3. Approve the PR with confidence because the changes are transparent

---

### Scenario D: New Engineer Onboarding

A new backend engineer joins your team. She's trying to understand why the payment
processor uses idempotency keys.

```bash
git log --all --oneline --follow src/payments/processor.js | head -10
# Shows the evolution of that file

git show <specific-commit>
# fix(payments): add idempotency key for Stripe webhook deduplication
# 
# Stripe webhooks are retried on our 5xx responses. Without idempotency
# keys, every retry created a duplicate invoice.
# 
# Solution: Hash the Stripe event ID and use as idempotency key.
# Database has UNIQUE constraint on invoice.stripe_event_id.
# 
# Closes #891.
```

The new engineer learns in 30 seconds what took the original developer days to discover.
This is the compounding return on investment of good commit messages.

---

## RULE 12 — Quick Reference Card

### The Commit Message Template

```
<type>[(<scope>)][!]: <subject in imperative, ≤ 50 chars>
                                    (blank line)
<body: explain WHY, wrap at 72 chars>
                                    (blank line)
<footer: Closes #N, BREAKING CHANGE:, Co-authored-by:>
```

### Conventional Commit Types

| Type | Use for | SemVer |
|------|---------|--------|
| `feat` | New user-facing feature | MINOR |
| `fix` | Bug fix | PATCH |
| `chore` | Tooling, dependencies, config | none |
| `docs` | Documentation | none |
| `style` | Formatting, whitespace | none |
| `refactor` | Code restructure (no behavior change) | none |
| `perf` | Performance improvement | PATCH |
| `test` | Add/fix tests | none |
| `build` | Build system changes | none |
| `ci` | CI configuration | none |
| `revert` | Revert a previous commit | depends |

### `git commit` Commands for Quality

| Command | Use for |
|---------|---------|
| `git commit` | Open editor — best for multi-line messages |
| `git commit -v` | Open editor WITH diff — review what you're committing |
| `git commit -m "type: subject"` | Quick one-liners |
| `git commit --amend` | Fix last commit (message or content) |
| `git commit --amend --no-edit` | Add to last commit, keep message |
| `git commit --amend -m "new"` | Replace last commit message |
| `git revert <sha>` | Safe undo of any pushed commit |
| `git commit --allow-empty -m "ci: trigger pipeline"` | Empty trigger commit |

### The "Atomic Commit" Checklist

Before every `git commit`, ask:
- [ ] Does this commit do ONE logical thing?
- [ ] If this commit were cherry-picked to another branch, would it still make sense?
- [ ] Could someone `git revert` this commit cleanly without affecting unrelated work?
- [ ] Does my subject line complete: "If applied, this commit will ___"?
- [ ] Have I explained WHY (not what — the diff shows that)?

---

## Summary

The mechanics of `git commit` (Topic 06) determine whether a commit CAN be made.
The craft of committing (this topic) determines whether that commit is USEFUL.

The two key practices that separate junior from senior Git users:

1. **Atomic commits**: each commit is one logical change, independently reversible,
   independently reviewable, independently deployable. Built through disciplined use of
   `git add -p` and multiple commits per work session.

2. **Meaningful messages**: follow the 50/72 rule, use imperative mood, explain WHY in
   the body, reference issues in the footer, use Conventional Commits if your team uses
   automated changelogs. Your message is the only documentation that survives forever
   alongside the code.

The future engineer reading your commits (which might be you, 6 months from now) will
either thank you or curse you. The choice is made at commit time.

*Last Updated: Topic 07 — Commits Done Right — ✅ Completed.*
