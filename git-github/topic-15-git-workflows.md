# Topic 15: Git Workflows
### (Gitflow vs GitHub Flow vs Trunk-Based Development — When, Why, and What Lives in `.git/`)

---

## CONNECTION TO PREVIOUS TOPICS (Rule 11)

This topic is the synthesis of everything in Phase 3 and Phase 4. A "workflow" is just
a set of rules governing how you use the Git mechanics you already know.

- **Topic 08**: You know branches are 41-byte files. Every workflow is a **naming convention
  and policy** for those files — which branch names matter, what can be merged where, and
  who is allowed to push to what.

- **Topic 09**: You know fast-forward vs 3-way merges. Workflows define WHICH type of merge
  happens at each integration point. Gitflow uses `--no-ff` merge commits; GitHub Flow uses
  PRs which GitHub can squash, merge, or rebase-merge; TBD uses fast-forward-only to main.

- **Topic 11**: Rebasing is the tool that makes Trunk-Based Development fast. Feature flags
  replace long-lived branches; frequent rebases onto trunk keep divergence minimal.

- **Topic 14**: Tags and releases are the OUTPUT of every workflow. Where a tag gets created
  differs: Gitflow tags on `main` after a release branch merge; GitHub Flow tags on main
  directly; TBD tags every commit that passes CI.

- **Topic 10**: Remote branches (origin/main, origin/develop) mirror the workflow's branch
  structure. Gitflow requires `origin/develop`; GitHub Flow has only `origin/main`;
  TBD has only `origin/main` (or trunk).

- **Topic 07**: Conventional Commits are the commit standard that makes TBD and automated
  releases possible. Without meaningful commit messages, you can't auto-generate changelogs.

---

## RULE 1: ELI5 ANCHOR — Three Ways to Run a Restaurant Kitchen

Imagine a restaurant kitchen where commits are dishes and the main branch is the dining room.

**Gitflow (the Classic Restaurant)**:
Has a full brigade: a prep kitchen (develop), a pass (release branch), and the dining room
(main). Every dish goes through prep → pass → dining room in formal stages. Highly structured.
Hard to change once plated. Works great for scheduled tasting menus (versioned software releases).

**GitHub Flow (the Bistro)**:
Much simpler: prep happens on a separate station (feature branch), but it goes DIRECTLY
to the dining room (main) after the head chef approves (PR). No intermediate pass. Fast,
flexible. Works great for places that have a special every night (continuous deployment).

**Trunk-Based Development (the Open Kitchen)**:
Everyone works at the same counter (trunk/main). Dishes are plated continuously. Quality
gates (tests, feature flags) prevent half-cooked dishes from reaching guests. Works great
for very experienced kitchens where everyone is highly trusted and fast.

```
GITFLOW:         feature ──► develop ──► release ──► main ──► tag
GITHUB FLOW:     feature ──────────────────────────► main ──► tag (on deploy)
TBD:             trunk ─── short-lived branch (optional) ──► trunk (immediate)
```

---

## RULE 2: PHYSICAL ARCHITECTURE FIRST

### What `.git/` looks like under each workflow

**Gitflow — two long-lived integration branches:**
```
.git/
├── HEAD                       ← "ref: refs/heads/develop" (daily work branch)
├── refs/
│   ├── heads/
│   │   ├── main               ← ONLY receives merges from release/* or hotfix/*
│   │   ├── develop            ← main integration branch for features
│   │   ├── feature/user-auth  ← one per in-progress feature
│   │   ├── feature/payments
│   │   ├── release/2.3.0      ← SHORT-LIVED: opened when feature freeze starts
│   │   └── hotfix/cve-1234    ← SHORT-LIVED: emergency fix off main
│   └── tags/
│       └── v2.3.0             ← created when release/* merged into main
└── logs/
    └── refs/
        └── heads/
            ├── main           ← rarely written (only on merge commits)
            ├── develop        ← frequently written (daily integration point)
            └── feature/...   ← written constantly during development
```

**GitHub Flow — one long-lived branch:**
```
.git/
├── HEAD                       ← "ref: refs/heads/main" (or your feature branch)
├── refs/
│   ├── heads/
│   │   ├── main               ← the ONLY permanent branch
│   │   ├── feature/add-oauth  ← short-lived, deleted after merge
│   │   └── fix/rate-limit     ← short-lived, deleted after merge
│   └── tags/
│       └── v1.4.2             ← created on deploy (optional, some teams skip)
└── ...
```

**Trunk-Based Development — one branch plus very short-lived branches:**
```
.git/
├── HEAD                       ← "ref: refs/heads/main" (always on trunk)
├── refs/
│   ├── heads/
│   │   ├── main               ← the trunk — everyone integrates here daily
│   │   └── user/alice/fix-x   ← optional, max 1-2 days old, then deleted
│   └── tags/
│       └── v1.4.2             ← every merge to trunk gets tagged (automated)
└── ...
```

### The branch lifetime comparison

```
GITFLOW BRANCHES:
  main          ──────────────────────────────────────────────────────► (forever)
  develop       ──────────────────────────────────────────────────────► (forever)
  feature/x     ─────────────────────────[2 weeks]────────────────────X (deleted)
  release/2.3   ────────────────────[1-2 weeks]──────────────────────X (deleted)
  hotfix/y      ──[hours/days]──────────────────────────────────────X (deleted)

GITHUB FLOW BRANCHES:
  main          ──────────────────────────────────────────────────────► (forever)
  feature/x     ─────[1-3 days to 1 week]──────────────────────────X (deleted)
  fix/y         ─[hours to 1 day]──────────────────────────────────X (deleted)

TRUNK-BASED DEVELOPMENT:
  main (trunk)  ──────────────────────────────────────────────────────► (forever)
  user/alice/x  ─[hours to 2 days max]───────────────────────────────X (deleted)
  (most work directly on trunk or paired with feature flags)
```

---

## RULE 4: ASCII ARCHITECTURE DIAGRAMS (before Rule 3 — needed for context)

### Diagram 1: Gitflow — the complete branch model

```
TIME ─────────────────────────────────────────────────────────────────────►

       v1.0         v1.1        v2.0
main:  ●────────────●───────────●
       │↑           │↑          │↑
       ││ merge     ││          ││ merge from
       ││ from      ││          ││ release/2.0
  ─────┼┼───────────┼┼──────────┼┼──────────────────────────────────────
       ││           ││          ││
develop:●───────────●───────────●──────────────────────────────────────►
        │↑     │↑   │↑          ↑ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
        ││     ││   ││          │feature/user-auth (merged in)
  ──────┼┼─────┼┼───┼┼──────────┘
feature/auth:  │      feature/payments (merged in)
  ────────────►│      │
  (deleted)    │      ├────────────────►│
               │                       │ (deleted)
         release/1.1:                  release/2.0:
  ────────────────────►│         ─────────────────►│
         (bug fix only)│ (deleted)                 │ (deleted)
                                   hotfix/crit:
                                ─────►│ (deleted)
                                      ↑ branched from main
                                      ↓ merged back into main AND develop
```

### Diagram 2: GitHub Flow — simple and continuous

```
TIME ─────────────────────────────────────────────────────────────────────►

main:  ●──────●──────────────●────────────●───────────────────────────►
       │      ↑              ↑            ↑
       │      │              │            │
       │   PR merged      PR merged    PR merged
       │      │              │            │
       │  feature/oauth   fix/rate-   feature/api-v2
       │  ────────────►│  ─────►│     ────────────────────────────────►│
       │  (2 days)     │  (4h)  │     (1 week)                         │
       │               │        │                                       │
       │               X        X     PR opened ──► reviewed ──► merged X
       │          (deleted) (deleted)                             (deleted)
```

### Diagram 3: Trunk-Based Development — high-frequency integration

```
TIME ─────────────────────────────────────────────────────────────────────►

trunk: ●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●────────────►
         │ │ │ │ │ │ │ │ │ │ │
         │ │ │ │ │ │ │ │ │ │ │  ← commits by multiple engineers daily
         │ │ │ │ │ │ │ │ │ │ └─ all passing CI before reaching trunk
         │ │ │ │ │ │ │ │ │ └─── feature flag OFF = code deployed but not visible
         │ └─┤ └─┘ └─┘ └─┘     groups show short-lived branches (0-2 days)
         │
         └─ CI runs on every commit: tests + lint + security scan
            CD: auto-deploys to staging, manual approval for prod
            Feature flags control which features users see
```

### Diagram 4: The release cadence comparison

```
                    GITFLOW        GITHUB FLOW    TRUNK-BASED
                    
Release cadence:    Scheduled      On-demand      Continuous
                    (monthly,      (after each    (multiple
                    quarterly)     PR merge)      per day)

Deploy frequency:   Low            Medium-High    Very High

Branch complexity:  High           Low            Minimal

Hotfix ease:        Good           Easy           Trivial
                    (hotfix/*)     (PR to main)   (direct to trunk)

History clarity:    Medium         High           High
                    (many branches)(linear-ish)   (linear)

Best for:           Versioned      SaaS, web      High-trust
                    products,      apps, APIs     platform teams,
                    libraries,                    microservices
                    mobile apps
```

### Diagram 5: Gitflow hotfix — exact branch lifecycle

```
BEFORE HOTFIX:
  main:    A──B──C──D   ← v2.0.0 tag on D
  develop: A──B──C──D──E──F   ← feature work continuing

BUG FOUND IN PRODUCTION (v2.0.0 is broken)

1. Branch hotfix from main (NOT develop):
   hotfix/fix-null-ptr: D──H1   ← H1 = the fix

2. Tag and merge into main:
   main: A──B──C──D─────────H1'  ← v2.0.1 tagged here

3. ALSO merge into develop (so fix doesn't get lost):
   develop: A──B──C──D──E──F──H1''

KEY: The fix MUST go into BOTH main AND develop.
     Missing this step is the most common Gitflow mistake.
```

---

## RULE 3: EXACT DATA FLOW — The Gitflow Command Sequence

### Complete Gitflow: Feature Development

```
SETUP (once per repo):
  git switch -c develop main   # create develop from main
  git push -u origin develop   # set tracking

STARTING A FEATURE:
  git switch develop
  git pull origin develop       # ensure up-to-date
  git switch -c feature/user-auth develop   # branch from develop

  READS:   .git/refs/heads/develop → SHA of develop tip
  WRITES:  .git/refs/heads/feature/user-auth ← same SHA (new branch = new file)
  MODIFIES: .git/HEAD ← "ref: refs/heads/feature/user-auth"

DURING FEATURE DEVELOPMENT:
  [many commits on feature/user-auth]
  git push -u origin feature/user-auth   # push for backup/PR

FINISHING A FEATURE (merging into develop):
  git switch develop
  git merge --no-ff feature/user-auth    # MUST use --no-ff to preserve branch grouping
  
  READS:   .git/refs/heads/feature/user-auth → tip SHA
           .git/refs/heads/develop → current SHA
  COMPUTES: merge commit (3-way or FF — but --no-ff FORCES a merge commit even if FF possible)
  WRITES:  New merge commit object in .git/objects/
           .git/refs/heads/develop ← SHA of new merge commit
  
  git push origin develop
  git branch -d feature/user-auth        # delete local feature branch
  git push origin --delete feature/user-auth  # delete remote
```

### Complete Gitflow: Release Process

```
CREATING A RELEASE BRANCH:
  git switch -c release/2.3.0 develop    # branch from develop (feature freeze!)
  
  # Only bug fixes allowed here — no new features
  # Update version numbers, CHANGELOG

FINISHING A RELEASE:
  # Step 1: Merge to main
  git switch main
  git merge --no-ff release/2.3.0 -m "Release version 2.3.0"
  git tag -a v2.3.0 -m "Version 2.3.0"   # TAG ON MAIN
  
  WRITES:  New merge commit on main
           .git/refs/heads/main ← new SHA
           .git/refs/tags/v2.3.0 ← tag object SHA

  # Step 2: Merge BACK to develop (CRITICAL — do not skip!)
  git switch develop
  git merge --no-ff release/2.3.0 -m "Merge release/2.3.0 back into develop"
  
  WRITES:  New merge commit on develop
           .git/refs/heads/develop ← new SHA

  # Step 3: Clean up
  git branch -d release/2.3.0
  git push origin main develop v2.3.0
  git push origin --delete release/2.3.0
```

### Complete Gitflow: Hotfix Process

```
CREATING A HOTFIX:
  git switch -c hotfix/2.3.1 main        # branch from main (NOT develop!)
  
  READS:   .git/refs/heads/main → SHA of tagged v2.3.0 commit
  WRITES:  .git/refs/heads/hotfix/2.3.1 ← same SHA (branched from main)
  MODIFIES: .git/HEAD ← "ref: refs/heads/hotfix/2.3.1"
  
  # Make the fix
  git commit -am "fix: patch SQL injection in user search"

FINISHING A HOTFIX:
  # Step 1: Merge to main
  git switch main
  git merge --no-ff hotfix/2.3.1 -m "Hotfix: SQL injection patch"
  git tag -a v2.3.1 -m "Security hotfix"
  
  # Step 2: Merge to develop (so fix isn't lost in next release)
  git switch develop
  git merge --no-ff hotfix/2.3.1 -m "Merge hotfix/2.3.1 to develop"
  
  # Step 3: Clean up
  git branch -d hotfix/2.3.1
  git push origin main develop v2.3.1
  git push origin --delete hotfix/2.3.1
```

### Complete GitHub Flow: Feature → PR → Main

```
START WORK:
  git fetch origin
  git switch -c feature/add-oauth origin/main   # always branch from latest main

PUSH EARLY FOR COLLABORATION:
  git push -u origin feature/add-oauth   # as soon as first commit exists
  # Open a Draft PR on GitHub immediately (shows work-in-progress to team)

KEEP UP WITH MAIN (to avoid big conflict at PR time):
  git fetch origin
  git rebase origin/main   # keep feature branch on top of current main
  git push --force-with-lease   # force-push after rebase (only you use this branch)

READY FOR REVIEW:
  # Mark PR as Ready for Review (was Draft before)
  # CI must pass before review
  # At least 1 (or N) approvals required

MERGE (done via GitHub UI or CLI):
  gh pr merge --squash    # squash: all feature commits become one commit on main
  # OR:
  gh pr merge --merge     # merge commit: preserves branch history
  # OR:
  gh pr merge --rebase    # rebase: feature commits appear directly on main (linear)

AFTER MERGE:
  git switch main && git pull   # get the merged commit
  git branch -d feature/add-oauth
  git push origin --delete feature/add-oauth
```

### Complete Trunk-Based Development: Commit Directly or via Short Branch

```
OPTION A: Direct to trunk (only for trivial, highly-reviewed changes)
  git switch main
  git pull --rebase    # sync first
  [make small, complete change]
  git commit -am "fix: correct null check in user validator"
  git push origin main   # immediate integration

  READS:   .git/refs/heads/main → current SHA
  MODIFIES: .git/refs/heads/main ← new SHA after commit
  (CI runs within seconds — if it fails, fix immediately or revert)

OPTION B: Short-lived branch (for changes taking >1 hour)
  git switch -c alice/fix-rate-limiter main   # short branch with user prefix
  
  [max 1-2 days of work]
  [push at least daily for backup]
  
  # Before integrating:
  git fetch origin
  git rebase origin/main   # keep in sync with trunk
  
  # Integrate (via PR or direct):
  git switch main
  git merge --ff-only alice/fix-rate-limiter  # FF-only: keeps history linear
  git push origin main
  
  git branch -d alice/fix-rate-limiter

FEATURE FLAGS (the key TBD mechanism):
  # Code is deployed but hidden behind a flag
  if featureFlags.IsEnabled("new-payment-ui", userID) {
    return newPaymentHandler(w, r)
  }
  return legacyPaymentHandler(w, r)
  
  # Even large features can be merged to trunk incrementally
  # The flag stays OFF until the feature is complete and tested
  # Enable progressively: 1% → 10% → 100% of users
```

---

## GITFLOW — COMPLETE DEEP DIVE

### The original Gitflow paper (Vincent Driessen, 2010)

The model was published at nvie.com and became THE standard for large software projects
releasing discrete versions. It was designed for:
- Software with explicit version numbers (libraries, SDKs, desktop apps, mobile apps)
- Teams that can't deploy "continuously" (app stores, enterprise releases, etc.)
- Projects needing a "staging" period between feature completion and release

### Gitflow branch rules

```
BRANCH        BRANCHES FROM     MERGES INTO           NAMING
main          —                 —                     main / master
develop       main (once)       —                     develop
feature/*     develop           develop               feature/<ticket-or-name>
release/*     develop           main AND develop      release/<version>
hotfix/*      main              main AND develop      hotfix/<version>
```

### Using `git-flow` CLI extension

```bash
# Install (optional — automates the branch management)
brew install git-flow-avh   # macOS
apt-get install git-flow    # Debian/Ubuntu

# Initialize in an existing repo
git flow init
# Asks: Branch name for production releases: main
#       Branch name for development: develop
# (stores choices in .git/config)

# Feature workflow
git flow feature start user-auth       # creates feature/user-auth from develop
git flow feature finish user-auth      # merges into develop, deletes branch
git flow feature publish user-auth     # push to remote

# Release workflow
git flow release start 2.3.0           # creates release/2.3.0 from develop
git flow release finish 2.3.0          # merges to main AND develop, creates tag, deletes branch

# Hotfix workflow
git flow hotfix start 2.3.1            # creates hotfix/2.3.1 from main
git flow hotfix finish 2.3.1           # merges to main AND develop, creates tag, deletes branch
```

### Gitflow `.git/config` after `git flow init`

```ini
[gitflow "branch"]
    master = main
    develop = develop
[gitflow "prefix"]
    feature = feature/
    bugfix = bugfix/
    release = release/
    hotfix = hotfix/
    support = support/
    versiontag =
```

### Gitflow pros and cons

```
PROS:
  + Crystal-clear: every branch has a defined role and lifecycle
  + Multiple versions in production simultaneously (v1.x and v2.x branches)
  + Release process is structured and auditable
  + Good for compliance environments requiring formal review stages
  + Hotfixes are clean and isolated

CONS:
  - Complex: engineers must internalize 5 branch types and their rules
  - "Integrate late, discover late": long feature branches → big merge conflicts at end
  - develop branch is always potentially broken/unstable
  - Slow feedback: bugs in features not discovered until merged to develop
  - Poor fit for CD: you CAN'T deploy from develop or feature/* (only from main)
  - History is cluttered with merge commits ("Merge feature/x into develop")
  - Overhead for small teams
```

---

## GITHUB FLOW — COMPLETE DEEP DIVE

### GitHub Flow rules (the ENTIRE spec in 6 rules)

1. Anything in `main` is deployable
2. Create a branch for new work (from latest main)
3. Push your branch to origin regularly  
4. Open a Pull Request when ready for discussion (or even just when you want feedback)
5. Merge only after review approval + CI pass
6. Deploy immediately after merging to main

### The GitHub Flow branch lifecycle in `.git/`

```
.git/refs/heads/:
  main (permanent)
  feature/add-oauth (created) ──► CI ──► PR ──► review ──► CI ──► merged ──► DELETED

.git/refs/remotes/origin/:
  origin/main (tracking)
  origin/feature/add-oauth (created) ──────────────────────────────────────► DELETED

Timeline for feature/add-oauth:
  Day 0: git switch -c feature/add-oauth  → .git/refs/heads/feature/add-oauth written
  Day 0: git push -u origin              → .git/refs/remotes/origin/feature/add-oauth written
  Day 1-3: commits, pushes
  Day 3: CI pass, PR opened, review
  Day 4: approved, merged
  Day 4: git push origin --delete feature/add-oauth → remote ref deleted
  Day 4: git branch -d feature/add-oauth → local ref deleted
```

### GitHub Flow pros and cons

```
PROS:
  + Simple: one rule — "main is always deployable"
  + Fast feedback: short branches mean recent divergence, small conflicts
  + Supports continuous delivery naturally
  + Easy to understand for new team members
  + Works perfectly with GitHub's PR model
  + Feature branches are always recent (days, not weeks)

CONS:
  - No staging area: hard to manage multiple versions simultaneously
  - "main is always deployable" requires excellent test coverage and CI
  - Not suitable for managed release cycles (mobile apps, libraries)
  - Can be chaotic without strong PR culture (review process must be enforced)
  - No built-in hotfix procedure distinct from normal workflow
    (hotfixes are just regular PRs that get merged and deployed quickly)
```

---

## TRUNK-BASED DEVELOPMENT — COMPLETE DEEP DIVE

### The core philosophy

TBD is not just a workflow — it is a set of engineering practices:

1. **Small, frequent commits to trunk** — no branch lives longer than 2 days
2. **Feature flags** — incomplete code is deployed but hidden
3. **Comprehensive automated tests** — must be able to trust every trunk commit
4. **Pair programming / mob programming** — eliminates need for async code review
5. **Release trains** — trunk is ALWAYS releasable; release whenever you want

### Feature flags — the enabler of TBD

```python
# Example: Deploying a half-built payment feature to production safely
# The code is in production but disabled for all users

class FeatureFlag:
    def __init__(self, config_service):
        self.config = config_service

    def is_enabled(self, flag_name: str, user_id: str = None) -> bool:
        flag = self.config.get(f"feature.{flag_name}")
        if not flag:
            return False
        if flag.get("enabled_for_all"):
            return True
        if user_id and user_id in flag.get("enabled_for_users", []):
            return True
        rollout_pct = flag.get("rollout_percentage", 0)
        if rollout_pct > 0:
            # Deterministic: same user always gets same experience
            return hash(user_id + flag_name) % 100 < rollout_pct
        return False

# In route handler:
@app.route("/checkout", methods=["POST"])
def checkout():
    if feature_flags.is_enabled("new-checkout-v2", user_id=current_user.id):
        return new_checkout_handler()
    return legacy_checkout_handler()
```

```bash
# Feature flag store (LaunchDarkly, AWS AppConfig, custom):
# Flag: new-checkout-v2
# State: enabled_for_users: [alice@test.com, bob@test.com]  ← internal testing
# 
# One week later:
# State: rollout_percentage: 5  ← 5% of users
# 
# Another week:
# State: rollout_percentage: 100 ← full rollout
# 
# Remove the flag (code cleanup):
# git commit -am "chore: remove new-checkout-v2 feature flag"
# Main now has just new_checkout_handler() with no flag check
```

### Branch-by-abstraction (TBD alternative to feature branches)

```
SCENARIO: You need to replace your ORM (big refactor — weeks of work)

INSTEAD OF: A long feature branch (feature/replace-orm — lives for 3 weeks)

DO THIS:
  Step 1: Create an interface (abstraction layer)
    class DataRepository(Protocol):
        def get_user(self, id: str) -> User: ...
        def save_user(self, user: User) -> None: ...

  Step 2: Make existing ORM implement the interface
    class ORMUserRepository(DataRepository): ...
    # Commit to trunk: "refactor: introduce DataRepository interface"

  Step 3: Build new implementation alongside old one
    class RawSQLUserRepository(DataRepository): ...
    # Feature flag: use_raw_sql_repository
    # Commit incrementally to trunk over days/weeks

  Step 4: Test both implementations in prod (behind flag)
    repo = RawSQLUserRepository() if ff.is_enabled("use_raw_sql_repository") else ORMUserRepository()

  Step 5: Once new implementation proven, flip flag to 100%

  Step 6: Delete old implementation
    # git commit -am "chore: remove ORMUserRepository"
```

### TBD pros and cons

```
PROS:
  + Fastest possible integration — no "integration hell" because you integrate constantly
  + Works with CI/CD: every trunk commit can be automatically deployed
  + Forces good engineering: feature flags, abstractions, small commits, comprehensive tests
  + History is clean and linear
  + Zero "long-lived branch rot"
  + Encouraged by Google, Facebook, Microsoft for their core products

CONS:
  - Requires VERY high test coverage and CI quality (if CI is flaky, trunk is broken constantly)
  - Feature flags add complexity (need flag management infrastructure)
  - Not suitable for junior-heavy teams without strong engineering culture
  - Harder to do code review (no PR to review — must review at commit or pair)
  - Challenging for open-source (external contributors can't push to trunk)
  - Release of partially-complete features requires discipline with flags
```

---

## RULE 5: HANDS-ON PROOF COMMANDS

### Proof Lab 1: Simulate Gitflow in a local repo

```bash
mkdir gitflow-lab && cd gitflow-lab
git init
git config user.email "t@t.com" && git config user.name "Test"

# Initial commit on main
echo "# App v1.0" > README.md && git add . && git commit -m "Initial commit"
git tag -a v1.0.0 -m "First release"

# Create develop from main
git switch -c develop main
echo "develop setup" >> README.md && git commit -am "Setup develop branch"

# PROVE IT: Two branches exist
git branch
# * develop
#   main

# Create feature branch
git switch -c feature/user-auth develop
echo "auth code" > auth.js && git add . && git commit -m "feat: add OAuth integration"
echo "more auth" >> auth.js && git commit -am "feat: add session management"

# PROVE IT: feature branch has 2 commits ahead of develop
git log --oneline develop..feature/user-auth
# [sha] feat: add session management
# [sha] feat: add OAuth integration

# Finish feature: merge into develop with --no-ff
git switch develop
git merge --no-ff feature/user-auth -m "Merge feature/user-auth into develop"

# PROVE IT: develop has a merge commit (not FF)
git log --oneline --graph develop
# *   [sha] Merge feature/user-auth into develop  ← merge commit
# |\
# | * [sha] feat: add session management
# | * [sha] feat: add OAuth integration
# |/
# * [sha] Setup develop branch
# * [sha] Initial commit

# PROVE IT: The feature branch can now be deleted
git branch -d feature/user-auth
git branch
# * develop
#   main

# Create release branch
git switch -c release/2.0.0 develop
echo '"version": "2.0.0"' > package.json && git add . && git commit -m "chore(release): bump version to 2.0.0"

# Merge to main
git switch main
git merge --no-ff release/2.0.0 -m "Release version 2.0.0"
git tag -a v2.0.0 -m "Version 2.0.0"

# PROVE IT: main has the v2.0.0 tag
git log --oneline --decorate main
# [sha] (HEAD -> main, tag: v2.0.0) Release version 2.0.0

# Merge back to develop
git switch develop
git merge --no-ff release/2.0.0 -m "Merge release/2.0.0 back into develop"

# Clean up release branch
git branch -d release/2.0.0

# PROVE IT: Final state - two clean permanent branches
git log --oneline --graph --all
# Shows main and develop with proper merge commits
```

### Proof Lab 2: Simulate GitHub Flow

```bash
mkdir ghflow-lab && cd ghflow-lab
git init && git config user.email "t@t.com" && git config user.name "Test"

# main must always be deployable
echo "# Stable app" > app.js && git add . && git commit -m "Initial stable version"

# Feature branch from main
git switch -c feature/dark-mode main
echo "dark mode code" >> app.js && git commit -am "feat: add dark mode toggle"
echo "dark mode tests" > dark.test.js && git add . && git commit -m "test: add dark mode tests"

# Simulate CI pass (in real life: GitHub Actions would run here)
# CI passes → PR approved → merge to main

# Merge options (GitHub UI offers these as buttons):

# Option A: Squash merge (squashes feature commits to ONE on main)
git switch main
git merge --squash feature/dark-mode
git commit -m "feat: add dark mode toggle (#42)"   # one clean commit

# PROVE IT: main has ONE clean commit (not 2)
git log --oneline main
# [sha] feat: add dark mode toggle (#42)  ← one commit
# [sha] Initial stable version

# PROVE IT: feature/dark-mode's individual commits are NOT in main
git log --oneline feature/dark-mode
# [sha] test: add dark mode tests        ← feature branch still has both
# [sha] feat: add dark mode toggle
# [sha] Initial stable version

# Delete the feature branch
git branch -d feature/dark-mode

# PROVE IT: One linear, readable main history
git log --oneline main
# [sha] feat: add dark mode toggle (#42)
# [sha] Initial stable version
```

### Proof Lab 3: TBD with feature flag simulation

```bash
mkdir tbd-lab && cd tbd-lab
git init && git config user.email "t@t.com" && git config user.name "Test"

echo '{ "feature_new_checkout": false }' > flags.json
echo "app code" > app.js && git add . && git commit -m "Initial commit"

# Commit feature code to trunk (main) with flag disabled
cat > checkout.js << 'EOF'
const flags = require('./flags.json');

function checkout(cart) {
  if (flags.feature_new_checkout) {
    return newCheckout(cart);   // new code: not complete yet
  }
  return legacyCheckout(cart);  // safe fallback
}

function legacyCheckout(cart) { return "legacy: " + cart; }
function newCheckout(cart)     { return "new: " + cart; }   // WIP

module.exports = { checkout };
EOF

git add . && git commit -m "feat: add new checkout skeleton (flag: feature_new_checkout)"

# PROVE IT: Code is on main, flag is off — safe to deploy
git log --oneline
# [sha] feat: add new checkout skeleton (flag: feature_new_checkout)
# [sha] Initial commit

# Finish the feature (still on main)
sed -i 's/return "new: " + cart;/return "new v2: " + JSON.stringify(cart);/' checkout.js
git commit -am "feat: complete new checkout v2 logic"

# Enable flag for testing
sed -i 's/false/true/' flags.json
git commit -am "feat: enable new-checkout for internal testing"

# All still on trunk — no branch needed
git log --oneline --graph
# * [sha] feat: enable new-checkout for internal testing
# * [sha] feat: complete new checkout v2 logic
# * [sha] feat: add new checkout skeleton (flag: feature_new_checkout)
# * [sha] Initial commit

# Linear history. Everyone has the latest. No merge commits.
```

---

## RULE 6: SYNTAX BREAKDOWN — Workflow-Related Commands

### `git switch -c` (create and switch branch)

```
git switch -c <new-branch> [<start-point>]
│           │  │            │
│           │  │            └─ where to start the branch from (default: HEAD)
│           │  │               Can be: branch name, SHA, tag, remote tracking branch
│           │  └─ the new branch name
│           └─ -c: create (use -C to force-create, overwriting if exists)
└─ git switch (from Git 2.23+, preferred over git checkout)

GITFLOW EXAMPLES:
  git switch -c feature/auth develop          # feature from develop
  git switch -c release/2.0.0 develop         # release from develop
  git switch -c hotfix/2.0.1 main             # hotfix from main (NOT develop!)
  git switch -c feature/auth origin/develop   # feature from remote develop

GITHUB FLOW EXAMPLES:
  git switch -c feature/oauth origin/main     # always from origin/main
```

### `git merge --no-ff` vs `--ff-only` vs `--squash`

```
git merge [--no-ff | --ff-only | --squash] [-m <message>] <branch>
          │           │           │
          │           │           └─ --squash: apply all commits as ONE staged change
          │           │              Does NOT create a merge commit automatically
          │           │              You must: git commit -m "your message"
          │           │              Used by: GitHub's "Squash and merge" button
          │           └─ --ff-only: REFUSE to merge if not a fast-forward
          │              Fails with error if a merge commit would be needed
          │              Used by: TBD to enforce linear history
          └─ --no-ff: ALWAYS create a merge commit even if FF is possible
             Used by: Gitflow to preserve branch grouping in history

WHICH TO USE WHEN:
  Gitflow feature → develop:     --no-ff (preserve feature grouping)
  Gitflow release → main:        --no-ff (preserve release boundary)
  GitHub Flow PR merge:          depends on team policy (--squash is common)
  TBD integration:               --ff-only (enforce linear)
  Regular fast-forward update:   (default) or --ff-only
```

### `gh pr merge` (GitHub CLI)

```
gh pr merge [<number>] [--squash | --merge | --rebase] [--auto] [--delete-branch]
│           │           │          │         │          │        │
│           │           │          │         │          │        └─ automatically delete
│           │           │          │         │          │           the head branch after merge
│           │           │          │         │          └─ auto-merge: merge when CI passes
│           │           │          │         └─ --rebase: rebase onto base branch (linear)
│           │           │          └─ --merge: create a merge commit (preserves branch history)
│           │           └─ --squash: combine all commits into one
│           └─ PR number (omit to use current branch's PR)
└─ gh: GitHub CLI tool

EXAMPLES:
  gh pr merge --squash --delete-branch       # squash-merge, delete branch (GitHub Flow)
  gh pr merge --rebase --delete-branch       # rebase-merge, delete branch (TBD-like)
  gh pr merge --merge                        # traditional merge commit (Gitflow-like)
  gh pr merge --auto --squash --delete-branch # auto-merge when CI passes
```

---

## RULE 7: BEGINNER → PRODUCTION EXAMPLE

### Beginner: Choosing your first workflow

```
SCENARIO: You're starting a solo project — a command-line tool.

RECOMMENDATION: GitHub Flow (simplified even further)

RULES FOR SOLO DEVELOPER:
  - main is always stable
  - Create a branch for each feature or fix
  - Merge to main when done, with a clean commit message
  - Tag releases with SemVer

git switch -c feature/add-json-output main
[work]
git switch main
git merge feature/add-json-output   # FF is fine solo
git tag -a v1.1.0 -m "Add JSON output support"
git push origin main v1.1.0
git branch -d feature/add-json-output
```

### Production: Choosing a workflow for a 20-person SaaS company

**Context**: Backend API team (8 engineers), mobile app team (6 engineers), 
infra team (6 engineers). Deploying to cloud. Mobile apps released monthly to app stores.
API can deploy any time.

```
DECISION:
  Mobile app:   Gitflow  (because: app store releases are scheduled, monthly cadence,
                          can't deploy continuously, need release/* stabilization period,
                          need hotfix/* for critical app store updates)

  Backend API:  GitHub Flow  (because: can deploy any time, REST API is versioned
                              independently of app store releases, SaaS mindset,
                              daily deploys are possible and desired)

  Infrastructure: Trunk-Based  (because: Terraform/Helm changes are small and atomic,
                                 infra team is experienced, changes need fast feedback,
                                 no "feature" concept — changes are improvements)

IMPLEMENTATION:
  Mobile repo: git flow init, branch protection on main and develop
  API repo: branch protection on main (require PR + 1 approval + CI pass),
            auto-delete merged branches, squash-merge policy
  Infra repo: CODEOWNERS (infra-team required reviewers), direct-to-trunk with pair review
```

### Production: Migrating from Gitflow to Trunk-Based Development

**Context**: A 40-person engineering org. Currently on Gitflow, struggling with:
- Feature branches living 3+ weeks → massive merge conflicts
- Develop branch frequently broken → daily standup starts with "who broke develop?"
- Release branches delay fixes by 2 weeks (waiting for next release window)
- Team morale low from conflict resolution marathon

**Migration plan (6-month journey)**:

```
MONTH 1: GitHub Flow (intermediate step)
  - Collapse develop into main (one final merge)
  - Delete develop branch permanently
  - Enforce: feature branches max 1 week
  - Add CI: every PR requires passing tests
  - Add: branch protection (can't push to main directly)
  
  .git/config on remote: protect main, require 1 review
  git push origin --delete develop

MONTH 2-3: Shrink branch lifetimes
  - Target: feature branches max 3 days
  - Add: daily "branch age" reports in Slack
  - Introduce feature flags for large features
  - Weekly demo of feature-flag-hidden features to stakeholders

MONTH 4-5: Introduce TBD practices
  - Encourage direct-to-main for small changes
  - Implement branch-by-abstraction for one large refactor
  - Add pair code review for trunk commits (skip PR for pairs)

MONTH 6: Full TBD
  - Feature branches optional and short-lived (< 2 days)
  - Feature flags for all in-progress work
  - CI runs in < 10 minutes
  - Deploy to staging on every trunk commit
  - Deploy to prod on-demand (any trunk commit)
  
RESULT: 
  Before: Avg 18-day feature cycle from branch to prod
  After:  Avg 2-day feature cycle from branch to prod
  Before: Develop broken 40% of mornings
  After:  Trunk broken < 2% of time (CI catches it within minutes)
```

---

## RULE 8: MISTAKES → ROOT CAUSE → FIX → PREVENTION

### Mistake 1 (Gitflow): Forgetting to merge release/* back into develop

**What happened**: Release/2.3.0 was merged into main (and tagged v2.3.0), but nobody
merged it back into develop. A bug fix made during release stabilization is now ONLY on
main — when v2.4.0 ships, the bug will REAPPEAR.

**Root cause**: The merge-back step is easy to forget because you "feel done" after
tagging and deploying. Gitflow has two mandatory merge targets for both release/* and hotfix/*.

**Diagnosis**:
```bash
# Find commits in main but NOT in develop
git log develop..main --oneline
# [sha] Bug fix made on release/2.3.0 that never reached develop
```

**Fix**:
```bash
git switch develop
git merge --no-ff main -m "Backport release/2.3.0 fixes to develop"
git push origin develop
```

**Prevention**: Use `git-flow` CLI which automates both merges. Or add a CI check:
```bash
# CI check: any commit in main must be reachable from develop
git log develop..main --oneline | wc -l   # must be 0 after merge-back
```

---

### Mistake 2 (Gitflow): Branching hotfix from develop instead of main

**What happened**: Production has a critical bug in v2.3.0. You ran:
`git switch -c hotfix/fix-null main` — wait, you actually ran from develop:
`git switch -c hotfix/fix-null develop`

**Root cause**: You're usually on develop (daily work branch), so when you panic-create
a hotfix branch, muscle memory puts you on develop which has UNRELEASED features.

**Consequence**: Your hotfix now includes half-finished features from develop.
When you merge to main and deploy, you accidentally ship unfinished code.

**Diagnosis**:
```bash
# Commits in hotfix/fix-null not in main
git log main..hotfix/fix-null --oneline
# [sha] The actual fix
# [sha] feat: WIP user notifications    ← WRONG: unreleased feature from develop!
# [sha] feat: WIP payment refactor      ← WRONG
```

**Fix**:
```bash
# Extract just the fix commit via cherry-pick
git switch -c hotfix/fix-null-v2 main
git cherry-pick <sha-of-actual-fix>
git tag -a v2.3.1 ...
# Abandon the wrong hotfix branch
git branch -D hotfix/fix-null   # delete the bad one
```

**Prevention**: ALWAYS verify with `git status` and `git log` before creating a hotfix.
Explicitly specify the base: `git switch -c hotfix/fix-null main`

---

### Mistake 3 (GitHub Flow): "main is always deployable" violated

**What happened**: A PR was merged that passed CI, but CI didn't cover an integration
scenario. main is now broken. Users are affected.

**Root cause**: GitHub Flow's "main is always deployable" is only as strong as your
test coverage and CI quality. With 40% test coverage, "CI passes" ≠ "works in production."

**Fix**:
```bash
# Option A: Quick fix commit (if fix is obvious)
git switch -c fix/regression-from-pr-456 main
[fix code]
git commit -am "fix: correct regression from PR #456"
git push origin fix/regression-from-pr-456
# FAST PR: immediate merge

# Option B: Revert the offending PR
git revert <merge-commit-sha> --mainline 1   # revert merge commit
git push origin main    # deploys revert within minutes
```

**Prevention**: Expand test coverage. Add integration tests that run against a staging
environment. Use deployment smoke tests. Consider `--ff-only` or preview environments.

---

### Mistake 4 (TBD): Long-lived branch defeats the purpose

**What happened**: An engineer created `feature/new-auth` "for TBD" but it lived for
3 weeks because "it's complex." Now it's a long-lived feature branch and integration
hell follows.

**Root cause**: TBD requires specific techniques (feature flags, branch-by-abstraction)
to handle large features. Without these techniques, engineers default to long branches.

**Fix (retroactively)**:
```bash
# If the branch is 3 weeks old, you have two options:

# Option A: Merge NOW despite the pain (get it on trunk)
git fetch origin
git rebase origin/main   # resolve all conflicts
# CI must pass
git push origin main --no-ff...  # or via PR

# Option B: Restructure with feature flag
# Add a flag, split the work into mergeable pieces
# Merge the "framework" first (behind flag), then fill in incrementally
```

**Prevention**: Team agreement: if a branch is > 2 days old, it needs a plan.
Daily standup checkin: "which feature flag are your changes behind?"

---

### Mistake 5 (Any workflow): Not protecting main on the remote

**What happened**: An engineer accidentally pushed directly to main, bypassing review.
The commit broke CI and nobody noticed for 3 hours.

**Root cause**: No branch protection rules on the remote. GitHub, GitLab, Bitbucket all
support branch protection — but it must be configured explicitly.

**Fix** (GitHub):
```
Repository Settings → Branches → Add rule for: main
  ✅ Require a pull request before merging
  ✅ Require approvals: 1 (or 2 for critical repos)
  ✅ Require status checks to pass before merging: [your CI job name]
  ✅ Require branches to be up to date before merging
  ✅ Do not allow bypassing the above settings
  ✅ Restrict who can push to matching branches: [release-team role only]
```

```bash
# Verify protection is working:
git push origin main   # should be rejected if not in PR
# remote: error: GH006: Protected branch update failed for refs/heads/main.
```

---

### Mistake 6 (TBD): Trunk broken because CI was flaky, not code

**What happened**: CI randomly fails 20% of the time due to a flaky integration test.
Team ignores "CI is failing" alerts. A real break goes unnoticed.

**Root cause**: "CI is red" alarm fatigue. TBD collapses without a green trunk guarantee.

**Prevention**: 
- Track flakiness rate per test; quarantine consistently flaky tests
- Use test retry budgets (retry once, but fail after two consecutive failures)
- Measure and display "time since last broken trunk" — make it a team KPI
- Never merge when trunk is red (even for "unrelated" changes)

---

## RULE 9: BYTE-LEVEL INTERNALS — How Workflows Affect `.git/`

### The `develop` branch file lifecycle (Gitflow)

```bash
# When develop is created:
# WRITTEN: .git/refs/heads/develop (41 bytes, same SHA as main)
# WRITTEN: .git/logs/refs/heads/develop (one reflog line: "branch: Created from main")

# Every merge into develop:
# MODIFIED: .git/refs/heads/develop (new 41-byte SHA pointing to merge commit)
# APPENDED: .git/logs/refs/heads/develop (one new reflog line per merge)

# After 6 months of development (many features merged):
# .git/logs/refs/heads/develop has hundreds of lines
# .git/refs/heads/develop is still just 41 bytes (just the CURRENT SHA)

# The reflog is your audit trail:
git reflog show develop | head -10
# develop@{0}: merge feature/payments: Merge made by the 'ort' strategy.
# develop@{1}: merge feature/auth: Merge made by the 'ort' strategy.
# develop@{2}: merge feature/ui-redesign: Merge made by the 'ort' strategy.
# ...
```

### How `--no-ff` creates a different object graph than `--ff`

```bash
# Fast-forward (no merge commit):
BEFORE: develop → C, feature → E (E's parent is C)
AFTER:  develop → E (no new object — just moved the pointer)
.git/refs/heads/develop: SHA of E
No new objects in .git/objects/

# No-fast-forward (merge commit):
BEFORE: develop → C, feature → E (E's parent is C)
AFTER:  develop → M (M is a new merge commit)
M's parents: C (develop's old tip) and E (feature's tip)
.git/objects/<M's SHA>: new commit object with two parent lines
.git/refs/heads/develop: SHA of M

# DIFFERENCE:
# With --ff: git log --graph shows a linear history (can't see where branch was)
# With --no-ff: git log --graph shows diverging lines even for simple features
# Gitflow uses --no-ff because the merge commit shows the branch boundary clearly
```

### The "squash merge" object graph (GitHub Flow default)

```bash
# BEFORE: main → C, feature/auth → F (F is 4 commits from C)
#   feature/auth: C──D──E──F

git switch main
git merge --squash feature/auth
git commit -m "feat: add OAuth integration (#42)"

# AFTER: main → S (S is one new commit)
# S's parent: C (only ONE parent — not a merge commit!)
# S's tree: the same as F's tree (all 4 commits' changes combined)
# S has NO pointer to F or the feature branch

# In .git/:
# .git/objects/<S sha>: one commit object, parent = C
# .git/refs/heads/main: SHA of S
# .git/refs/heads/feature/auth: still points to F (until you delete it)

# RESULT: Clean main history, but feature branch's individual commits are "lost"
# (they're still in .git/objects/ until GC, but not reachable from main)
```

---

## WORKFLOW DECISION FLOWCHART

```
Do you need to maintain MULTIPLE versions in production simultaneously?
(e.g., v1.x and v2.x both need support, security patches to old versions)
│
├── YES → GITFLOW
│         (or a simplified variant: main + release/* + hotfix/*)
│
└── NO → Can you deploy to production AT ANY TIME (no app store, no quarterly release)?
         │
         ├── NO → GITFLOW or scaled-down Gitflow
         │        (if releases are scheduled, you need a release branch)
         │
         └── YES → Is your team highly experienced and do you have >80% test coverage?
                   │
                   ├── YES → TRUNK-BASED DEVELOPMENT (with feature flags)
                   │         Maximum speed, minimum overhead
                   │
                   └── NOT YET → GITHUB FLOW
                                 Simple, safe, teaches good habits
                                 Graduate to TBD as team matures
```

---

## RULE 10: MENTAL MODEL CHECKPOINT

**1.** In Gitflow, a hotfix must be merged into two branches. Which two? What happens
if you forget the second one? How do you detect and fix this?

**2.** GitHub Flow says "anything in main is deployable." What are the two technical
prerequisites that must be true for this rule to hold? What breaks if either is missing?

**3.** In TBD, how do you ship a large feature (e.g., a 3-month rewrite) without a
long-lived feature branch? Name both techniques and explain how they work.

**4.** What does `git merge --no-ff feature/auth` create in `.git/objects/` that
`git merge feature/auth` (fast-forward) does NOT? Why does Gitflow require `--no-ff`?

**5.** A Gitflow release branch is different from a Gitflow feature branch. List three
differences: (a) where it branches from, (b) what work is allowed on it, (c) where it merges into.

**6.** You're evaluating GitHub Flow vs Gitflow for a mobile app that has monthly
releases to the App Store. Which do you choose and why? Name two specific Gitflow
features that GitHub Flow lacks that matter for this scenario.

**7.** In TBD, a developer commits half-finished feature code directly to trunk.
The feature flag is OFF. A bug in this half-finished code (even the hidden path) causes
a crash when a config value is null. What should have prevented this, and what's the fix?

**8.** What does `git merge --squash` produce in `.git/objects/` compared to a regular
merge? How many parents does the resulting commit have? How does this affect `git log --graph`?

**9.** Describe the branch protection settings you would configure on GitHub for the
`main` branch of a team using GitHub Flow. List at least four settings and explain
why each one matters.

**10.** What is "branch-by-abstraction"? Give a concrete example with code structure.
How does it enable large refactors in TBD without long-lived branches?

---

## RULE 12: QUICK REFERENCE CARD

### Gitflow commands

| Command | What It Does |
|---------|-------------|
| `git flow init` | Set up Gitflow in a repo (prompts for branch names) |
| `git flow feature start <name>` | Create feature/<name> from develop |
| `git flow feature finish <name>` | Merge feature/<name> into develop, delete branch |
| `git flow feature publish <name>` | Push feature/<name> to origin |
| `git flow release start <ver>` | Create release/<ver> from develop |
| `git flow release finish <ver>` | Merge to main + develop, create tag, delete branch |
| `git flow hotfix start <ver>` | Create hotfix/<ver> from main |
| `git flow hotfix finish <ver>` | Merge to main + develop, create tag, delete branch |
| `git merge --no-ff <branch>` | Merge with forced merge commit (Gitflow standard) |

### GitHub Flow commands

| Command | What It Does |
|---------|-------------|
| `git switch -c feature/<name> origin/main` | Create feature branch from latest main |
| `git push -u origin feature/<name>` | Push and set tracking (to enable PRs) |
| `git fetch && git rebase origin/main` | Keep feature branch up to date |
| `git push --force-with-lease` | Force-push after rebase (safe) |
| `gh pr create` | Open a Pull Request from CLI |
| `gh pr merge --squash --delete-branch` | Squash-merge and delete branch |
| `git switch main && git pull` | Sync main after PR merge |

### TBD commands

| Command | What It Does |
|---------|-------------|
| `git pull --rebase` | Sync trunk before starting work |
| `git switch -c <user>/<name> main` | Short-lived branch (max 2 days) |
| `git rebase origin/main` | Keep short branch up-to-date |
| `git merge --ff-only <branch>` | Integrate to trunk, enforce linear history |
| `git push origin main` | Direct push (only if branch-protected properly) |

### Workflow comparison

| | Gitflow | GitHub Flow | TBD |
|--|---------|-------------|-----|
| Permanent branches | main + develop | main only | main only |
| Feature branch lifetime | Weeks | Days | Hours-2 days |
| Integration point | develop | main (via PR) | main (direct or PR) |
| Release process | release/* branch | Tag main directly | Automated on trunk |
| Hotfix process | hotfix/* from main | Fast PR to main | Fast direct commit |
| Requires CI | Recommended | Required | Essential |
| Requires feature flags | No | No | Yes (for large features) |
| Best for | Versioned releases | SaaS/web | High-trust teams |

---

## RULE 13: WHEN WOULD I USE THIS AT WORK?

### Scenario 1: Joining a new team — identify their workflow in 60 seconds

```bash
# What branches exist?
git branch -a | grep -v HEAD | sort

# If you see: remotes/origin/develop AND remotes/origin/main → Gitflow
# If you see: only remotes/origin/main + many feature/* → GitHub Flow
# If you see: only remotes/origin/main + short-lived user/* branches → TBD

# How long do branches live?
git for-each-ref --format='%(refname:short) %(creatordate:relative)' refs/remotes/origin \
  | grep -v HEAD | sort -k2

# If feature branches are weeks old → Gitflow or slow GitHub Flow
# If feature branches are days old → healthy GitHub Flow
# If barely any branches exist → TBD

# What does the log look like?
git log --graph --oneline --all | head -30
# Many "Merge branch" commits → Gitflow or --no-ff GitHub Flow
# Linear or near-linear → GitHub Flow (squash) or TBD
```

### Scenario 2: Choosing a workflow for a new microservice

```
Context: New payments microservice, team of 3 engineers.
Deploys to Kubernetes. Each PR triggers a staging deploy automatically.
Rest API consumed by mobile and web teams.

Decision:
  GitHub Flow with squash-merge policy.
  
  Rationale:
  - 3 engineers → Gitflow overhead is too high for this team size
  - Kubernetes staging deploy on PR → "main is deployable" is enforced by infra
  - REST API → can deploy any time (no app store friction)
  - Squash-merge → linear main history, easy to bisect if regression introduced

  Branch protection on main:
  - Require PR
  - Require 1 approval (for a 3-person team, 1 approval means a second set of eyes)
  - Require CI: unit tests + integration tests + staging deploy smoke test
  - Auto-delete merged branches

  Result: each feature = one commit on main, easy to trace bugs to PRs
```

### Scenario 3: Engineering manager reviewing branch hygiene

```bash
# Weekly branch audit: find branches older than 7 days (GitHub Flow — branches should be short)
git for-each-ref --format='%(refname:short) %(creatordate:iso)' refs/remotes/origin \
  | grep "feature/" \
  | while read branch date; do
    age=$(( ($(date +%s) - $(date -d "$date" +%s)) / 86400 ))
    if [ $age -gt 7 ]; then
      echo "OLD ($age days): $branch"
    fi
  done

# Expected output should be empty for healthy GitHub Flow team
# If you see many OLD branches → team is not following GitHub Flow correctly
# → Discussion: is the branch lifetime policy realistic? Do features need to be smaller?
```

### Scenario 4: Implementing release automation for Gitflow

```yaml
# .github/workflows/gitflow-release.yml
# Trigger: when release/* branch is merged to main

on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  release:
    if: |
      github.event.pull_request.merged == true &&
      startsWith(github.event.pull_request.head.ref, 'release/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract version from branch name
        id: version
        run: |
          BRANCH="${{ github.event.pull_request.head.ref }}"
          VERSION="${BRANCH#release/}"
          echo "VERSION=$VERSION" >> $GITHUB_OUTPUT

      - name: Create and push tag
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          git tag -a v${{ steps.version.outputs.VERSION }} -m "Release v${{ steps.version.outputs.VERSION }}"
          git push origin v${{ steps.version.outputs.VERSION }}

      - name: Create GitHub Release
        run: |
          gh release create v${{ steps.version.outputs.VERSION }} \
            --title "v${{ steps.version.outputs.VERSION }}" \
            --generate-notes
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Merge back into develop
        run: |
          git fetch origin develop
          git checkout develop
          git merge --no-ff origin/main -m "Merge release v${{ steps.version.outputs.VERSION }} back into develop"
          git push origin develop
```

---

## SUMMARY: The Mental Model in One Paragraph

A Git workflow is not a technical constraint — it is an agreement between engineers about
which branches are sacred, how code travels from your keyboard to production, and who can
do what. Gitflow uses two permanent integration branches (`main` and `develop`) plus
short-lived `feature/*`, `release/*`, and `hotfix/*` branches, making it robust for
versioned software with scheduled releases but heavy for teams wanting continuous delivery.
GitHub Flow collapses to one rule: `main` is always deployable; everything else is a
short-lived feature branch that merges via Pull Request, making it simple and effective
for web services that can ship anytime. Trunk-Based Development pushes further: everyone
integrates to trunk at least daily, feature flags replace long-lived branches, and the
CI pipeline is the quality gate instead of human gatekeeping on branches. The right
choice depends on three questions: Do you need multiple versions in production? Can you
deploy any time? Do you have the test coverage and engineering maturity to trust every
commit? Answer those honestly and the workflow choice follows naturally.

---

*Topic 15 of 26 — Next: Topic 16: Pull Requests & Code Review Culture (PR best practices, draft PRs, review workflows)*
