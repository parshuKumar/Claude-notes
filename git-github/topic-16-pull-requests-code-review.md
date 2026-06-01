# Topic 16 — Pull Requests & Code Review Culture

> **Curriculum**: Git & GitHub Deep Mastery — Zero → Principal Engineer  
> **Phase 5**: Team & Company-Level Git (Topics 15–19)  
> **Prerequisites**: Topic 09 (Merging), Topic 10 (Remote Repositories), Topic 11 (Rebasing), Topic 15 (Git Workflows)

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The Peer-Review Analogy](#1-eli5-anchor)
2. [What Is a Pull Request, Really?](#2-what-is-a-pull-request-really)
3. [Physical Architecture — Where PRs Live](#3-physical-architecture)
4. [PR Anatomy — Every Field Explained](#4-pr-anatomy)
5. [Draft PRs — Early Feedback Without Merge Risk](#5-draft-prs)
6. [The PR Lifecycle — Step-by-Step Data Flow](#6-the-pr-lifecycle)
7. [Review Types — Approve / Request Changes / Comment](#7-review-types)
8. [CODEOWNERS — Mandatory Review Routing](#8-codeowners)
9. [The Merge Button — Three Options Decoded](#9-the-merge-button)
10. [PR Best Practices — Size, Description, Self-Review](#10-pr-best-practices)
11. [Giving Code Review Feedback — The Senior Approach](#11-giving-code-review-feedback)
12. [Receiving Code Review — The Professional Mindset](#12-receiving-code-review)
13. [What to Look for in a Review — The Checklist](#13-review-checklist)
14. [CI Integration — How GitHub Checks Gate the Merge](#14-ci-integration)
15. [PR Templates](#15-pr-templates)
16. [Production Scenarios](#16-production-scenarios)
17. [Mistakes → Root Cause → Fix → Prevention](#17-mistakes)
18. [Byte-Level: What GitHub Actually Stores for a PR](#18-byte-level-internals)
19. [Mental Model Checkpoint](#19-mental-model-checkpoint)
20. [Connect Everything Backwards](#20-connect-everything-backwards)
21. [Quick Reference Card](#21-quick-reference-card)
22. [When Would I Use This At Work?](#22-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The Academic Paper Submission Analogy

Imagine you are writing a research paper for a scientific journal.

You do not just walk up to the journal editor and say "print this." Instead:

1. You **write a draft** (your feature branch with commits)
2. You **submit it for peer review** (open a Pull Request)
3. **Reviewers read it** and leave margin notes — "this section is unclear," "this citation is wrong," "this is excellent"
4. You **revise** the paper based on feedback (push new commits to the same branch)
5. The reviewers re-read the revised version
6. When everyone is satisfied, the **editor approves and publishes** (merge into main)
7. The paper is now **part of the permanent record** (commit in main's history)

A Pull Request is exactly that process applied to code.

The word "Pull" comes from the original Git model: you are **requesting** that the maintainer **pull** your changes into their branch. On GitHub the destination pulls from the source — hence "pull request." On GitLab they call it a "Merge Request" which is arguably clearer.

**Why does this exist?**

Without PRs, developers push directly to `main`. This means:
- No one reviewed the change before it affected production
- There is no documented record of *why* the change was made
- There is no gate for automated tests to run before the code lands
- Mistakes are only discovered after the fact — sometimes at 3am

PRs are a **quality gate, a documentation artifact, and a knowledge-sharing mechanism** all in one.

---

## 2. What Is a Pull Request, Really?

A Pull Request is **not a Git concept**. Git itself knows nothing about PRs.

A Pull Request is a **GitHub (or GitLab / Bitbucket) application-layer concept** built on top of Git. It is a database record that says:

> "I want to merge commits from branch `feature/login` into branch `main`. Please review them."

### What GitHub Stores for a PR

```
Pull Request Record (GitHub database):
├── id: 42
├── number: 42                    ← PR #42 (repo-scoped, displayed in UI)
├── title: "Add login endpoint"
├── body: "(markdown description)"
├── state: open | closed | merged
├── head: { branch: "feature/login", sha: "a3f2c1d...", repo: "owner/repo" }
├── base: { branch: "main",          sha: "d8e7b2c...", repo: "owner/repo" }
├── author: { login: "alice" }
├── reviewers: [{ login: "bob", state: "approved" }]
├── labels: ["backend", "auth"]
├── milestone: "v2.1"
├── linked_issues: [15, 22]
├── checks: [{ name: "tests", state: "success" }, ...]
├── commits: [sha1, sha2, sha3]   ← the commits being proposed
├── diff: (computed diff of head vs merge-base)
├── created_at: "2026-05-28T10:00:00Z"
├── merged_at: null
└── merge_commit_sha: null        ← filled when merged
```

The key insight: **GitHub computes the diff dynamically** between `head.sha` and the **merge base** of `head` and `base`. This is exactly the 3-way merge ancestor from Topic 09.

```
MERGE BASE COMPUTATION (same algorithm as git merge-base):

        A---B---C  ← feature/login (HEAD = C)
       /
  D---E---F---G    ← main (HEAD = G)
       ↑
       E is the merge base

PR diff = changes from E to C
          (not "all commits on main" — only what diverged)
```

---

## 3. Physical Architecture

### Where PRs Live — What GitHub Stores vs What Git Stores

PRs are stored in **GitHub's database**, not in your `.git/` directory. But there are Git-level traces.

#### Your Local `.git/` (no knowledge of the PR)

```
.git/
├── HEAD                         ← "ref: refs/heads/feature/login"
├── refs/
│   ├── heads/
│   │   ├── main                 ← local main pointer
│   │   └── feature/login        ← your feature branch pointer
│   └── remotes/
│       └── origin/
│           ├── main             ← remote-tracking: origin/main
│           └── feature/login    ← remote-tracking: origin/feature/login
├── objects/                     ← all commit/blob/tree objects (local)
│   ├── a3/
│   │   └── f2c1d8...            ← one of your feature commits
│   └── ...
└── config
    └── [remote "origin"]
        url = https://github.com/alice/myapp.git
        fetch = +refs/heads/*:refs/remotes/origin/*
```

#### GitHub's Hidden Refs — `refs/pull/*/head`

GitHub creates a **special read-only ref** for every PR so you can fetch any PR branch locally even if you don't have write access to the fork:

```
GitHub-side refs (not in your local .git/ until you fetch them):

refs/pull/42/head    ← tip of the PR branch (same as feature/login SHA)
refs/pull/42/merge   ← a test-merge commit GitHub auto-creates
                        (GitHub tests if the merge is clean)
```

You can fetch a PR by number:

```bash
# Fetch PR #42's branch locally:
git fetch origin pull/42/head:pr-42
git checkout pr-42

# Or one-liner:
git fetch origin pull/42/head && git checkout FETCH_HEAD
```

#### After Merge — The Merge Commit in `.git/`

Once a PR is merged, a **real Git commit** lands in `main`. The PR record in GitHub's database is updated with `merge_commit_sha`. The actual history is now in `.git/`:

```
.git/
├── refs/
│   └── remotes/
│       └── origin/
│           └── main             ← now points to the merge commit SHA
└── objects/
    └── 8f/
        └── 3a1b9c...            ← the merge commit (or squash commit)
```

---

## 4. PR Anatomy — Every Field Explained

```
┌─────────────────────────────────────────────────────────┐
│  [title]  Add OAuth2 login endpoint                     │
│  ─────────────────────────────────────────────────────  │
│  base: main  ←  head: feature/auth-oauth               │
│  author: @alice  │  3 commits  │  +247 -18 lines        │
│                                                         │
│  [Labels]   backend  auth  review-needed                │
│  [Milestone] v2.1                                       │
│  [Reviewers] @bob (pending)  @carol (approved)          │
│  [Assignees] @alice                                     │
│  [Linked Issues] Closes #15, Fixes #22                  │
│                                                         │
│  [Description / Body]                                   │
│  ## What this PR does                                   │
│  Implements OAuth2 login using Google provider.         │
│                                                         │
│  ## Why                                                 │
│  Closes #15 (users requested Google login)             │
│                                                         │
│  ## Testing                                             │
│  - [ ] Unit tests added                                 │
│  - [ ] Integration test against staging                 │
│  - [ ] Manual test with real Google account             │
│                                                         │
│  [Checks]   ✅ tests   ✅ lint   ⏳ deploy-preview      │
│  ─────────────────────────────────────────────────────  │
│  [Files Changed tab]  [Commits tab]  [Checks tab]       │
└─────────────────────────────────────────────────────────┘
```

### Field-by-Field Breakdown

| Field | What it means | Pro tip |
|-------|--------------|---------|
| **Title** | One-line summary of what changes | Use imperative: "Add login" not "Adding login" |
| **Base branch** | Where the commits will land | Almost always `main` or `develop` |
| **Head branch** | Where your commits are now | Your feature branch |
| **Description** | Markdown-formatted body | Most important field — explain WHY not WHAT |
| **Labels** | Categorization tags | Used for filtering, automation, and routing |
| **Milestone** | Release version grouping | Links PRs to a release |
| **Reviewers** | Who must review | CODEOWNERS auto-adds based on files changed |
| **Assignees** | Who is responsible | Usually the author |
| **Linked Issues** | `Closes #N` / `Fixes #N` | Auto-closes issue when PR merges |
| **Checks** | CI status from GitHub Actions | Can be required before merge |
| **Draft** | Work-in-progress flag | Blocks the Merge button |

### The "Closes #N" Magic

When your PR description contains:
```
Closes #15
Fixes #22
Resolves #30
```

GitHub does two things:
1. **Shows the link** in the UI (bidirectional: the issue shows the PR, the PR shows the issue)
2. **Auto-closes the issue** when the PR is merged into the default branch

This is just text parsing by GitHub — no Git magic involved. But it is powerful for project management.

---

## 5. Draft PRs — Early Feedback Without Merge Risk

### What Is a Draft PR?

A **Draft PR** is a PR in "not ready to merge" state. The Merge button is disabled. GitHub shows a grey "Draft" badge.

```
┌──────────────────────────────────────┐
│  [Draft] Add payment processing       │
│  This pull request is still a draft  │
│  ──────────────────────────────────  │
│  [Ready for review] button           │
│                                      │
│  Merge pull request  (GREYED OUT)    │
└──────────────────────────────────────┘
```

### When to Use Draft PRs

```
Timeline of a feature:

Day 1:  git push -u origin feature/payments
        → Open DRAFT PR immediately
        → CI starts running on every push
        → Team can see you're working on this
        → Early architectural questions can be raised

Day 3:  "Hey, before I go further, does this approach look right?"
        → Tag reviewer in a comment, they look at the design
        → No pressure — it's a draft

Day 5:  Feature complete, all tests green
        → Click "Ready for review"
        → Reviewers are notified, formal review begins

Day 6:  Reviewer requests changes
        → Push fix commit to same branch
        → PR auto-updates

Day 7:  Approved → Merge
```

### How to Create a Draft PR

```bash
# Via GitHub CLI:
gh pr create --draft --title "Add payment processing" --body "WIP"

# Via GitHub UI:
# New Pull Request → dropdown arrow next to "Create pull request"
# → "Create draft pull request"
```

### Draft PR vs Regular PR — Behavior Differences

| Behavior | Draft PR | Regular PR |
|----------|---------|------------|
| Merge button | Disabled | Enabled (if approved + checks pass) |
| CI runs | ✅ Yes | ✅ Yes |
| Review requests | ✅ Can request | ✅ Can request |
| Auto-assigned via CODEOWNERS | ❌ No (on GitHub) | ✅ Yes |
| Listed in "Needs review" filter | ❌ No | ✅ Yes |
| Signals intent | "WIP" | "Please review" |

---

## 6. The PR Lifecycle — Step-by-Step Data Flow

### Full Lifecycle Trace

```
STEP 1: Create feature branch locally
────────────────────────────────────────
COMMAND: git checkout -b feature/login main

READS:   .git/refs/heads/main → "d8e7b2c..."
WRITES:  .git/refs/heads/feature/login → "d8e7b2c..." (same SHA as main)
MODIFIES: .git/HEAD → "ref: refs/heads/feature/login"


STEP 2: Make commits
────────────────────────────────────────
(multiple git add / git commit cycles — see Topic 06)
After 3 commits:
.git/refs/heads/feature/login → "a3f2c1d..." (latest commit)


STEP 3: Push to remote
────────────────────────────────────────
COMMAND: git push -u origin feature/login

SENDS:   objects for commits, trees, blobs not on remote
WRITES:  origin's refs/heads/feature/login → "a3f2c1d..."
MODIFIES: .git/refs/remotes/origin/feature/login → "a3f2c1d..."
          .git/config (adds tracking: branch.feature/login.remote=origin)


STEP 4: Open PR on GitHub
────────────────────────────────────────
ACTION:  GitHub UI or `gh pr create`

GITHUB DOES:
1. Creates PR record in database (id=42, head=a3f2c1d, base=main@d8e7b2c)
2. Computes merge base: git merge-base a3f2c1d d8e7b2c → "d8e7b2c"
3. Computes diff: what changed from merge-base to head
4. Creates ref: refs/pull/42/head → "a3f2c1d..."
5. Runs test merge: creates refs/pull/42/merge (temp commit)
6. Triggers CI: sends webhook to GitHub Actions, etc.
7. Notifies CODEOWNERS / assigned reviewers


STEP 5: Reviewer leaves feedback
────────────────────────────────────────
ACTION:  Reviewer reads diff, clicks "+" to add inline comment

GITHUB DOES:
1. Stores comment in database: { pr_id:42, file:"auth.js", line:37, body:"..." }
2. Sends email/notification to PR author
3. PR state: "Changes requested" (if formal review submitted)


STEP 6: Author pushes fix
────────────────────────────────────────
COMMAND: (edit file) → git add → git commit → git push origin feature/login

GITHUB DOES:
1. Updates PR head SHA: "a3f2c1d..." → "b4e3d2c..."
2. Re-runs CI
3. Re-computes diff (now includes the fix commit)
4. Old "Changes requested" review is dismissed (configurable)
5. Notifies reviewer to re-review


STEP 7: Approval
────────────────────────────────────────
ACTION:  Reviewer clicks "Approve" in GitHub UI

GITHUB DOES:
1. Stores review: { reviewer: "bob", state: "APPROVED", pr_id: 42 }
2. If branch protection requires N approvals and N are met:
   → Merge button becomes active
3. If all required checks are green:
   → Merge button is fully enabled


STEP 8: Merge
────────────────────────────────────────
ACTION:  Author (or maintainer) clicks "Merge pull request"

GITHUB DOES:
1. Runs `git merge --no-ff feature/login` (for merge commit strategy)
   OR `git merge --squash feature/login` (squash strategy)
   OR `git rebase main feature/login; git merge --ff-only` (rebase strategy)
2. Creates a commit (or moves pointer) on main
3. Updates: refs/heads/main → new merge commit SHA
4. Updates PR record: state=merged, merge_commit_sha="8f3a1b9c..."
5. Deletes refs/pull/42/merge (cleanup)
6. If "Closes #15" was in description → closes Issue #15
7. Optionally deletes the feature branch (if "auto-delete" enabled)


STEP 9: Sync locally
────────────────────────────────────────
COMMAND: git checkout main && git pull origin main

READS:   origin/main's new HEAD
WRITES:  .git/refs/heads/main → "8f3a1b9c..." (the merge commit)
MODIFIES: .git/refs/remotes/origin/main → "8f3a1b9c..."
```

---

## 7. Review Types — Approve / Request Changes / Comment

GitHub has three formal review outcomes. This is not just UI — they have mechanical effects.

### Review State Machine

```
                        ┌─────────────┐
                        │   PENDING   │ ← review started but not submitted
                        └──────┬──────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        ┌──────────┐   ┌──────────────┐  ┌──────────┐
        │ APPROVED │   │   CHANGES    │  │ COMMENTED│
        │          │   │  REQUESTED   │  │ (no vote)│
        └──────────┘   └──────────────┘  └──────────┘
               │               │               │
               │         new push commits      │
               │               │               │
               │         ┌─────▼─────┐         │
               │         │ DISMISSED │         │
               │         │(auto or   │         │
               │         │ manually) │         │
               └─────────┴─────┬─────┴─────────┘
                               │
                        ┌──────▼──────┐
                        │   MERGED    │
                        └─────────────┘
```

### The Three Review Actions

#### 1. Approve
```
Effect on merge button:
  - Counts toward required approvals
  - If N required approvals reached AND checks pass → merge enabled

When to use:
  - You have read every changed line
  - You are satisfied with correctness, readability, security, and tests
  - You would be comfortable owning this code
  
What "Approve" means to a senior engineer:
  "I have reviewed this change and take shared responsibility for it."
```

#### 2. Request Changes
```
Effect on merge button:
  - Blocks merge (even if N approvals are met from others)
  - The blocker's approval is required to unblock (unless overridden by maintainer)

When to use:
  - There is a bug, security issue, or correctness problem
  - The approach is wrong (not just stylistically different)
  - Tests are missing for non-trivial logic
  - The change will cause future pain that the author hasn't considered

What "Request Changes" does NOT mean:
  - "I don't like your variable names" (use Comment instead)
  - "I would have done this differently" (use Comment + suggestion)
  
It means: "I believe this code should not merge as-is."
```

#### 3. Comment (No Vote)
```
Effect on merge button:
  - None — purely informational
  - Does NOT block merge

When to use:
  - You have questions but they aren't blockers
  - Nit-picks (style, naming preferences)
  - Suggestions for future improvement ("Out of scope for this PR, but...")
  - Praising good work
  - Asking for clarification to understand the code
```

### Review Comment Types

| Type | Syntax | Effect |
|------|--------|--------|
| **Inline comment** | Click `+` on a line | Attached to specific file+line |
| **Suggested change** | ` ```suggestion ` block | Author can apply with one click |
| **Resolved thread** | "Resolve conversation" button | Thread collapses, shows as resolved |
| **Review summary** | General comment on submission | Overall thoughts |

#### The `suggestion` Block

This is a power feature that reduces friction:

```markdown
In your review, instead of writing:

"Line 47: the variable name `x` is unclear, should be `userSessionToken`"

Write a suggestion:
```suggestion
const userSessionToken = req.headers.authorization;
```
```

The author sees a diff preview and can click "Commit suggestion" to apply it directly from the GitHub UI without going back to their editor. Multiple suggestions can be batched into one commit.

---

## 8. CODEOWNERS — Mandatory Review Routing

### What Is CODEOWNERS?

A `.github/CODEOWNERS` file maps **file paths** to **required reviewers**. When a PR touches those paths, GitHub automatically:
1. Adds the owner as a required reviewer
2. Their approval is **required** (not optional) before merge

### File Location

```
.github/CODEOWNERS       ← recommended
CODEOWNERS               ← also works (root)
docs/CODEOWNERS          ← also works
```

### Syntax

```
# CODEOWNERS syntax
# Pattern    Owner(s)

# Default owner for everything not matched below
*                     @alice

# Backend API — requires backend team review
src/api/**            @alice @bob

# Frontend — requires frontend team
src/frontend/**       @carol @dave

# Database migrations — requires senior + DBA
db/migrations/**      @alice @eve

# Secrets / infrastructure — requires specific person
.github/workflows/**  @alice
terraform/**          @alice

# Specific file — the auth module needs security review
src/auth/             @security-team

# Negation — exclude a path from a previous rule
# (not supported natively — use order: last matching rule wins)
```

### How CODEOWNERS Interacts with Branch Protection

```
Branch Protection Rule for `main`:
├── Require pull request reviews before merging
│   ├── Required approving reviews: 1
│   └── Require review from Code Owners: ✅ ENABLED
│
└── Effect:
    - PR touching src/api/ needs @alice OR @bob to approve
    - A non-owner's approval does NOT count toward CODEOWNERS requirement
    - Regular required count (1) and CODEOWNERS count are separate
```

### CODEOWNERS Example — Repo Structure

```
myapp/
├── .github/
│   └── CODEOWNERS
│       ────────────────────────────────
│       *                @alice
│       src/api/**       @backend-team
│       src/ui/**        @frontend-team  
│       db/              @alice @dba-team
│       ────────────────────────────────
├── src/
│   ├── api/
│   │   └── auth.js      ← PR touching this requires @backend-team
│   └── ui/
│       └── login.jsx    ← PR touching this requires @frontend-team
└── db/
    └── migrations/      ← PR touching this requires @alice AND @dba-team
```

A PR that touches both `src/api/auth.js` AND `src/ui/login.jsx` requires approval from **both** `@backend-team` **and** `@frontend-team`.

---

## 9. The Merge Button — Three Options Decoded

When a PR is approved and checks pass, the author clicks the merge button. GitHub offers three strategies. This connects directly to Topic 15 (Workflows) and Topic 09 (Merging).

```
┌──────────────────────────────────────────────┐
│  Merge pull request  ▼                       │
│  ──────────────────────────────────────────  │
│  ◉ Create a merge commit                     │
│    Merge with a merge commit. All commits    │
│    from this branch will be added to the     │
│    base branch via a merge commit.           │
│                                              │
│  ○ Squash and merge                          │
│    Combine all commits from the head branch  │
│    into a single commit.                     │
│                                              │
│  ○ Rebase and merge                          │
│    Rebase all commits from the head branch   │
│    onto the base branch individually.        │
└──────────────────────────────────────────────┘
```

### Option 1: Create a Merge Commit

```
Before merge:
  A---B---C  feature/login
 /
D---E---F    main (HEAD=F)

After merge:
  A---B---C  feature/login
 /         \
D---E---F---G  main (HEAD=G, G is merge commit)

G has two parents: F and C
```

**What this creates in `.git/objects/`:**
```
G (merge commit):
  tree   <new tree SHA>
  parent F-SHA        ← first parent: was HEAD of main
  parent C-SHA        ← second parent: tip of feature branch
  author alice
  committer alice
  message "Merge pull request #42 from feature/login"
```

**Properties:**
- Preserves full history of feature branch
- Non-linear history (visible fork + merge in `git log --graph`)
- `git revert` of a merge commit is possible but annoying (needs `-m 1`)
- Best for: long-lived feature branches, regulatory environments needing audit trail

### Option 2: Squash and Merge

```
Before squash:
  A---B---C  feature/login (3 commits)
 /
D---E---F    main (HEAD=F)

After squash:
D---E---F---S  main (HEAD=S, S is one new commit)

feature/login branch: NOT moved. Still at C.
The branch can now be deleted.
```

**What S contains:**
```
S (squash commit):
  tree   <final state of feature/login>
  parent F-SHA        ← only one parent (main's old HEAD)
  author alice
  committer alice
  message "Add login endpoint (#42)"
                      ← GitHub auto-generates: original title + PR number
  
  (Optionally the body lists the squashed commits)
```

**Properties:**
- Clean linear main history — every commit on main = one PR
- Individual "wip", "fix typo", "fix review feedback" commits disappear
- `git log main` reads like a changelog
- `git bisect` is much easier (every commit is a shippable unit)
- Loss: you cannot `git log --follow` into the individual changes after merge
- Best for: GitHub Flow teams that value clean main history

### Option 3: Rebase and Merge

```
Before rebase:
  A---B---C  feature/login (3 commits, branched from E)
 /
D---E---F    main (HEAD=F)

After rebase and merge:
D---E---F---A'---B'---C'  main (HEAD=C')

A', B', C' are NEW commits (new SHAs) replayed on top of F
```

**What this creates:**
- Three new commit objects: A', B', C' each with updated `parent` and `committer` fields
- New SHAs because parent chain changed (see Topic 11)
- Linear history, no merge commit, all original commits preserved (with rewrites)

**Properties:**
- Linear history like squash
- But individual commits are visible (unlike squash)
- The rewrites mean the feature branch's SHAs no longer match main's history
- Best for: teams that want clean linear history AND individual commit granularity

### Side-by-Side Comparison

```
Strategy         | Merge commit | Squash | Rebase
─────────────────|─────────────|--------|-------
History shape    | Non-linear  | Linear | Linear
Commits on main  | All + merge | One    | All (rewritten)
Original SHAs    | Preserved   | Lost   | Lost (new SHAs)
git bisect ease  | Hard        | Easy   | Medium
Revert a PR      | git revert -m 1 | git revert SHA | Multiple reverts
Author preserved | ✅ Yes      | ✅ Yes | ✅ Yes
Committer        | GitHub bot  | GitHub bot | GitHub bot
```

---

## 10. PR Best Practices — Size, Description, Self-Review

### The Right Size for a PR

The single biggest factor in PR quality is **PR size**. Not content, not writing — SIZE.

```
PR Review Time vs PR Size (empirical data from industry studies):

Lines changed │ Average review time │ Defect detection rate
──────────────┼────────────────────┼──────────────────────
< 50          │ 15 minutes          │ ~70%
50–200        │ 30-45 minutes       │ ~50%
200–500       │ 1-2 hours           │ ~35%
500–1000      │ 2-4 hours           │ ~20%
> 1000        │ "Will review later" │ ~5%

The reviewer's brain can hold roughly 400 lines in working memory.
Beyond that, review quality degrades rapidly.
```

**The rule of thumb**: A PR should do **one thing**. Not "add feature X and refactor Y and bump dependency Z." Three things = three PRs.

**Stacked PRs (advanced)**: If feature B depends on feature A, open PR A against `main` and PR B against `feature/a`. When A merges, B's base is now just the incremental diff. This is called "stacked PRs" or "PR stacking."

### The Perfect PR Description Template

```markdown
## What this PR does
<!-- One paragraph. What change is being made? -->
Adds Google OAuth2 login using the `passport-google-oauth20` library.
Users can now sign in with their Google account instead of username/password.

## Why
<!-- Why is this change needed? Link to the issue, user story, or decision. -->
Closes #15. Users have repeatedly requested social login. 
This is the first of three planned providers (Google, GitHub, Microsoft).

## How (if non-obvious)
<!-- Architecture decisions, trade-offs, things that aren't obvious from the diff -->
Using the existing `sessions` table with a new `provider` column rather than 
a separate OAuth table, because we expect less than 5% of users to use multiple 
providers. This keeps queries simple.

## Testing
- [x] Unit tests for token validation
- [x] Integration test with mock Google OAuth server
- [ ] Manual test with real Google account (will do in staging)

## Screenshots / output (if UI change)
<!-- Before/after screenshots or test output -->

## Checklist
- [x] No secrets in code
- [x] Database migration included
- [x] CHANGELOG updated
- [x] Docs updated (if API change)
```

### The Self-Review Rule

**Before requesting review, review your own PR.**

```bash
# On GitHub: click "Files changed" on your own PR
# Pretend you are a reviewer who knows nothing about this

# Or locally:
git diff main...feature/login

# Or with a tool like delta:
git diff main...feature/login | delta
```

The self-review catches:
- Debug code left in (`console.log`, `print`, `TODO: remove`)
- Accidentally committed secrets (API keys, tokens)
- Wrong files included (node_modules, .env, build artifacts)
- Obvious bugs you'd be embarrassed to have reviewed
- Description doesn't match the actual change

Studies show self-review reduces review round-trips by 30-50%.

---

## 11. Giving Code Review Feedback — The Senior Approach

### The Spectrum of Review Comments

Not all review comments are equal. Senior engineers label their feedback so the author knows what must change vs what is optional.

```
Comment Prefixes (informal convention, widely adopted):

nit:      Nitpick — style, naming, minor preference. Author's discretion.
          "nit: could rename `x` to `userCount` for clarity"

q:        Question — I don't understand this. Not necessarily wrong.
          "q: why do we need the try-catch here? Can this throw?"

suggest:  Suggestion — optional improvement. Won't block.
          "suggest: this could use Array.flatMap() instead of map+flat"

issue:    Actual problem — must be addressed before merge.
          "issue: this can panic if `user` is nil, no null check"

blocker:  Critical — security, data loss, correctness. Requires immediate attention.
          "blocker: this stores passwords in plaintext"

praise:   Positive feedback — normalize giving positive comments too.
          "praise: this error handling is really clean, nice pattern"
```

### Tone and Framing

**Bad review comment:**
```
"This is wrong. Why would you do this?"
```

**Good review comment:**
```
"issue: The `getUserById` call here doesn't handle the case where the user 
was deleted between authentication and this endpoint call. This could cause 
a 500 error. Could we add a `user || throw new NotFoundError()` check?"
```

The difference:
1. Label tells author the severity
2. Explains **what** the problem is
3. Explains **why** it matters
4. Suggests a concrete fix
5. Uses "we"/"could" language instead of "you/should"

### The Code Review Mental Checklist (What Seniors Look For)

```
CORRECTNESS (most important)
  □ Does the code do what the description says?
  □ Are edge cases handled? (null, empty, zero, MAX_INT, network failure)
  □ Are concurrent access / race conditions possible?
  □ Is error handling present and correct?
  □ Could this cause data loss or corruption?

SECURITY (never skip)
  □ User input is validated and sanitized before use
  □ No SQL injection (parameterized queries only)
  □ No hardcoded secrets, API keys, or credentials
  □ Authentication checks on every protected endpoint
  □ Authorization: can a user access other users' data?
  □ Dependencies: are new libraries trustworthy/maintained?

PERFORMANCE (for hot paths)
  □ N+1 queries (querying in a loop instead of a JOIN)
  □ Missing database indexes for new query patterns
  □ Large allocations in tight loops
  □ Blocking I/O on critical paths

MAINTAINABILITY
  □ Would a new team member understand this in 6 months?
  □ Is there duplicated logic that should be extracted?
  □ Are variable/function names clear?
  □ Is complexity hidden in "clever" one-liners?

TESTS
  □ Are tests present for non-trivial logic?
  □ Do tests test behavior, not implementation details?
  □ Are failure cases tested (not just the happy path)?
  □ Are tests deterministic? (no random, no time.now() without mocking)

CONTRACTS / API DESIGN (for interface changes)
  □ Is this a breaking change? Is it versioned correctly?
  □ Are error messages useful for API consumers?
  □ Is the endpoint consistent with existing patterns?
```

---

## 12. Receiving Code Review — The Professional Mindset

### The Golden Rules

**1. The review is about the code, not you.**

"This function is confusing" ≠ "You are a bad programmer."  
Separate your identity from your code. Even the best engineers get review feedback.

**2. Assume positive intent.**

A harsh-sounding comment is almost always someone who is busy, not someone who dislikes you. If a comment is unclear or feels disrespectful, ask for clarification before reacting.

**3. Don't argue — ask clarifying questions.**

Instead of:
```
"That's not necessary, the code is fine as-is."
```

Try:
```
"Can you help me understand the specific scenario you're worried about?
I want to make sure I address the right concern."
```

**4. Respond to every comment.**

Even if your response is "Good catch, fixed!" or "I kept it as-is because [reason]." Unresponded comments make reviewers feel ignored and slow down the process.

```
Good responses:
  ✅ "Fixed in latest commit"
  ✅ "Good point, I've added a null check"
  ✅ "Intentional — [explanation]. Added a comment in the code to clarify"
  ✅ "I disagree because [reason], but happy to change if you feel strongly"
  ❌ "K" (too terse)
  ❌ No response at all
```

**5. If you disagree, have the discussion — once.**

It is legitimate to push back on review feedback. State your reasoning clearly. If the reviewer maintains their position, either accept it (code review is part of team ownership) or escalate to a team decision (don't let it block forever).

---

## 13. Review Checklist — What to Look for Systematically

This is an enhanced version of the mental checklist from Section 11, formatted as an actual checklist template:

```markdown
## Code Review Checklist

### Functionality
- [ ] Does the code implement the described behavior?
- [ ] Are all acceptance criteria from the linked issue met?
- [ ] Are edge cases (empty, null, boundary values) handled?
- [ ] Are error states handled gracefully?

### Security
- [ ] No credentials, tokens, or secrets in code or comments
- [ ] All user inputs validated/sanitized
- [ ] Authorization checks present (not just authentication)
- [ ] No eval(), exec(), or equivalent dynamic code execution on user input
- [ ] Dependencies checked for known vulnerabilities

### Code Quality
- [ ] Logic is understandable without needing to read the PR description
- [ ] No dead code or commented-out code blocks
- [ ] No console.log / print / debug statements
- [ ] Functions do one thing (single responsibility)
- [ ] No magic numbers/strings (use named constants)

### Tests
- [ ] Tests cover the new/changed behavior
- [ ] Tests cover failure paths, not just success
- [ ] Tests are readable and self-documenting

### Deployment / Operations
- [ ] Database migrations are backward compatible (or deployment plan documented)
- [ ] Environment variables documented (if new ones added)
- [ ] Logging added for observable behavior
- [ ] No performance regression in hot paths

### Documentation
- [ ] Public API changes reflected in docs
- [ ] Complex logic has inline comments explaining WHY (not what)
- [ ] CHANGELOG updated if applicable
```

---

## 14. CI Integration — How GitHub Checks Gate the Merge

### What Are GitHub Checks?

Any external service (GitHub Actions, CircleCI, Jenkins, Codecov, etc.) can report a **status** on a commit via GitHub's Checks API. The PR shows these as required or optional gates.

```
PR Checks Dashboard:

✅ build (2m 14s)           ← GitHub Actions: compiled successfully
✅ test (4m 08s)            ← GitHub Actions: 127 tests passed
✅ lint (0m 45s)            ← ESLint: no violations
⏳ deploy-preview (waiting)  ← Vercel: building preview deployment
✅ codecov/patch (87% coverage)
❌ security-scan             ← Snyk: 1 high severity vulnerability found

Merge blocked: security-scan must pass
```

### How Checks Work (Data Flow)

```
STEP 1: You push a commit to feature branch
         ↓
STEP 2: GitHub sends a webhook to GitHub Actions (or external CI)
         Payload: { "sha": "a3f2c1d...", "ref": "feature/login", ... }
         ↓
STEP 3: CI clones repo at that commit
         git clone --depth=1 https://github.com/... 
         git checkout a3f2c1d...
         ↓
STEP 4: CI runs your pipeline (test, lint, build)
         ↓
STEP 5: CI calls GitHub Checks API:
         POST /repos/owner/repo/check-runs
         { "name": "tests", "status": "completed", 
           "conclusion": "success", "head_sha": "a3f2c1d..." }
         ↓
STEP 6: GitHub updates PR UI with check status
         ↓
STEP 7: If this check is "required" in branch protection:
         merge button enabled/disabled accordingly
```

### Requiring Checks in Branch Protection

```
Branch Protection for `main`:
├── Require status checks to pass before merging
│   ├── build  ← REQUIRED
│   ├── test   ← REQUIRED
│   ├── lint   ← REQUIRED
│   └── security-scan ← REQUIRED
│
├── Require branches to be up to date before merging
│   └── (PR must be on top of latest main, not behind by any commits)
│
└── Do not allow bypassing the above settings
    └── (even admins cannot force-merge — useful for SOC2 compliance)
```

The "require branches to be up to date" rule means if `main` got a new commit after you branched, you must `git merge main` (or `git rebase main`) into your branch before you can merge. This prevents "works in isolation but conflicts at merge time" problems.

---

## 15. PR Templates

### What Is a PR Template?

A markdown file in your repository that GitHub **auto-fills** into the PR description box when anyone opens a new PR. Authors edit/complete it rather than starting from a blank box.

### File Location

```
.github/PULL_REQUEST_TEMPLATE.md         ← single template (most common)
.github/PULL_REQUEST_TEMPLATE/            ← multiple templates (dropdown)
    feature.md
    bugfix.md
    hotfix.md
```

### Minimal Production Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Summary
<!-- What does this PR do? One or two sentences. -->

## Motivation
<!-- Why is this change needed? Link to issue: Closes #N -->

## Changes
<!-- Bullet list of what changed -->
- 

## Testing
- [ ] Unit tests pass locally
- [ ] Added tests for new behavior
- [ ] Tested manually in dev environment

## Checklist
- [ ] No debug code or console.log left in
- [ ] No secrets or credentials in code
- [ ] Database migration included (if schema change)
- [ ] Documentation updated (if public API change)

## Screenshots (if UI change)
```

### Multiple Templates with Query Params

When you have multiple templates, GitHub shows a dropdown. You can also link directly to a template:

```
https://github.com/owner/repo/compare/main...feature/login
  ?template=feature.md
  &title=Add+login+endpoint
  &labels=backend
```

---

## 16. Production Scenarios

### Scenario 1: Hotfix PR (Urgent, Must Be Fast)

**Context**: It's 11pm. A critical bug is causing payment failures in production. You need to get a fix in and deployed ASAP. The normal review process takes 24 hours.

```bash
# Step 1: Branch from main (not develop — hotfix goes to production immediately)
git checkout main
git pull origin main
git checkout -b hotfix/payment-null-pointer

# Step 2: Make the minimal fix — do NOT refactor anything else
# Fix only the line causing the issue

# Step 3: Commit with clear message
git add src/payments/processor.js
git commit -m "fix: null pointer in payment processor when card_token is nil

Fixes #500. The token can be nil when the card vault returns a 503 error.
Added a nil check and returns a RetryableError instead of panicking.

This is a hotfix — full test coverage will be added in follow-up PR #501."

# Step 4: Push and open PR
git push -u origin hotfix/payment-null-pointer
gh pr create \
  --title "fix: hotfix for payment processor null pointer (#500)" \
  --body "Closes #500. Hotfix — see commit message. Needs expedited review." \
  --label "hotfix" "critical" \
  --reviewer alice \
  --base main

# Step 5: Ping reviewer directly (Slack/phone) — don't rely on email notification
# Step 6: Reviewer fast-tracks the review focusing on: is this fix correct? Does it break anything?
# Step 7: Merge immediately when approved (squash to keep main clean)
# Step 8: Deploy
# Step 9: Open follow-up PR for proper tests
```

**What makes this different from a normal PR:**
- Branch from main, not develop
- Minimal change (don't fix anything else while you're there)
- Expedited review with direct communication
- Deploy immediately on merge
- Follow-up PR for tests/cleanup

### Scenario 2: Large Feature PR (1000+ lines)

**Context**: You spent 3 weeks implementing a new reporting module. It's 1400 lines changed across 23 files.

**The wrong approach**: Open one massive PR and ask for review.

**The right approach**: Split into reviewable units.

```
Original scope: "New Reporting Module"

Split into:
  PR 1: Database schema + migrations (50 lines)
         → Reviewer can focus: is the schema correct? Any index issues?
         → Merge first

  PR 2: Data access layer (200 lines, depends on PR 1's schema)
         → Base: feature/reporting-schema (after it merges into main,
                  rebase onto main)
         → Reviewer can focus: query correctness, N+1 issues

  PR 3: Business logic (300 lines, depends on PR 2)
         → Reviewer can focus: algorithm correctness, edge cases

  PR 4: API endpoints (150 lines, depends on PR 3)
         → Reviewer can focus: HTTP semantics, auth, input validation

  PR 5: Frontend components (400 lines, depends on PR 4)
         → Reviewer can focus: UI/UX, accessibility

  PR 6: Integration + E2E tests (300 lines)
         → Can be reviewed by QA engineer

Each PR: ~200 lines, reviewable in 30 minutes.
Total: 6 fast reviews instead of 1 multi-day nightmare.
```

### Scenario 3: Reviewing a Dependency Update PR

**Context**: Dependabot (or Renovate) has opened 15 PRs to bump your dependencies.

```
What to check for EACH dependency update PR:

1. Changelog review:
   - Click the "Release notes" link in the PR body
   - Read the changelog for the version bump range
   - Look for: breaking changes, security fixes, behavior changes

2. Diff inspection:
   - It should only touch package.json and lockfile
   - If it touches src/ files, something is wrong (reject)

3. CI checks:
   - All tests must pass — the CI is your safety net here
   - If CI is green and changelog shows no breaking changes → approve

4. Security-motivated bumps:
   - If the PR title says "security" or links to a CVE → prioritize
   - Read the CVE, understand if your usage is affected
   - If affected: merge same day

5. Major version bumps:
   - Read the migration guide
   - May require code changes (dependabot only bumps, doesn't fix code)
   - Create a separate "fix: migrate to library v3" PR for any needed changes
```

---

## 17. Mistakes → Root Cause → Fix → Prevention

### Mistake 1: Opening a 2000-Line PR

**Symptom**: "Nobody has reviewed my PR in 5 days."

**Root cause**: The PR is too large to review in a reasonable session. Reviewers mentally defer large PRs because they can't finish in one sitting.

**Fix (damage control for an existing large PR)**:
```bash
# You can't retroactively split commits easily once the branch is old.
# But you can partially:
git checkout -b review/part-1 feature/big-feature
git rebase -i main  # squash all into one, then manually split (hard but possible)

# Easier: add thorough description and schedule a review walkthrough call
# Walk the reviewer through the changes live — this substitutes for line-by-line review
```

**Prevention**: Before writing code, plan the PR structure. Open draft PRs early. Keep PRs under 300 lines as a team policy.

### Mistake 2: Merging Without Waiting for CI

**Symptom**: "I forgot to wait for the tests to finish — the build was probably fine."

**Root cause**: Human impatience, lack of branch protection enforcement.

**Consequence**: Broken `main`. Every developer who pulls gets broken code. CI for subsequent PRs starts failing. Root cause becomes hard to identify.

**Fix**:
```bash
# After breaking main:
git checkout main
git log --oneline -5  # find the bad merge commit SHA

# Option A: revert the merge
git revert -m 1 <merge-commit-SHA>
git push origin main

# Option B: if very recent, coordinate with team then:
git reset --hard HEAD~1
git push --force-with-lease origin main
# DANGER: communicate clearly, everyone must re-pull
```

**Prevention**: **Require status checks in branch protection.** This is the only reliable prevention. Human discipline alone fails.

### Mistake 3: Approving Without Reading

**Symptom**: "LGTM" from a reviewer who spent 30 seconds on a 500-line PR.

**Root cause**: Social pressure to approve quickly ("I don't want to slow the team down"), unclear team norms, fear of conflict.

**Consequence**: Bugs, security vulnerabilities, and unmaintainable code reach production. When something breaks, no one takes ownership.

**Fix**: None after the fact — the bad code is in main.

**Prevention**: 
- Establish team norms: "LGTM" is a professional statement of accountability
- Set review time expectations: "Reviews take 1-4 hours for a quality job"
- Praise thorough reviews in team retros
- Track escaped defects back to the review that missed them (without blame — as learning)

### Mistake 4: Rebasing a Shared PR Branch

**Symptom**: "I rebased my PR branch and now my teammate who checked out the branch has conflicts."

**Root cause**: Rebase rewrites SHAs. If anyone else has pulled the old SHAs, their local history diverges.

```
Before rebase:
  remote: A---B---C  feature/login (team pulled this)
  local:  A---B---C  feature/login (your rebased version — DIFFERENT SHAs)

After you `git push --force`:
  remote: A'--B'--C'  feature/login (new SHAs)
  teammate: A---B---C  (still has old SHAs — now diverged)
```

**Fix**:
```bash
# Tell teammate:
git fetch origin
git checkout feature/login
git reset --hard origin/feature/login  # discard their local, take remote
# They lose any local changes they haven't pushed to that branch
```

**Prevention**: 
- Only rebase PR branches that you own alone
- If multiple people work a branch, use merge commits to integrate, not rebase
- Coordinate before force-pushing ("I'm about to rebase feature/login, heads up")

### Mistake 5: Not Linking the PR to an Issue

**Symptom**: "Why was this code added?" (months later, no answer in git log)

**Root cause**: Developer rushed, thought the PR title was enough context.

**Consequence**: Future developers can't understand the WHY behind code decisions. Debugging becomes archaeology. "Chesterton's Fence" — you remove code you think is unnecessary, but it was there for a reason nobody documented.

**Fix**: None retroactively (can't change git history of PR metadata).

**Prevention**: 
- Every PR has a "Why" section in the template
- Make the template required (can't be deleted, just filled)
- Add `linked_issue` as a required label in automation

### Mistake 6: Force-Pushing After Reviews

**Symptom**: Reviewer says "I approved based on commit A but then you force-pushed and now commit A doesn't exist — I need to re-review."

**Root cause**: Author ran `git push --force` to clean up "wip" commits after getting approval.

**Consequence**: The approval is based on code that no longer exists. If the force-push introduced a regression, it goes unreviewed.

**Fix**: GitHub marks existing approvals as "stale" if branch protection has "Dismiss stale pull request approvals when new commits are pushed" enabled — but force-pushes can bypass this depending on configuration.

**Prevention**:
- Never force-push after approval
- Do your `git rebase -i` cleanup BEFORE requesting review
- If you must clean up after reviews, communicate: "I've done a rebase to squash the wip commits — please re-review the final state"

---

## 18. Byte-Level Internals — What GitHub Actually Stores for a PR

### The PR Object Model (Simplified)

GitHub's backend is not open source, but from the API documentation and behavior, we can infer the data model:

```
pull_requests table (conceptual):
┌──────────────────────────────────────────────────────────────────┐
│ id          │ BIGINT PRIMARY KEY                                 │
│ number      │ INT (repo-scoped, sequential)                      │
│ repo_id     │ BIGINT FK → repositories                          │
│ state       │ ENUM('open', 'closed', 'merged')                   │
│ title       │ TEXT                                               │
│ body        │ TEXT (markdown)                                    │
│ head_ref    │ VARCHAR(255)  ← branch name                        │
│ head_sha    │ CHAR(40)      ← current tip commit SHA             │
│ base_ref    │ VARCHAR(255)  ← target branch name                 │
│ base_sha    │ CHAR(40)      ← base commit SHA at time of PR open │
│ merge_base_sha │ CHAR(40)  ← computed merge base (3-way)         │
│ mergeable   │ ENUM('mergeable', 'conflicting', 'unknown')        │
│ merged_at   │ TIMESTAMP NULL                                     │
│ merge_commit_sha │ CHAR(40) NULL                                 │
│ created_by  │ BIGINT FK → users                                  │
│ created_at  │ TIMESTAMP                                          │
│ updated_at  │ TIMESTAMP                                          │
└──────────────────────────────────────────────────────────────────┘

reviews table:
┌──────────────────────────────────────────────────────────────────┐
│ id          │ BIGINT PRIMARY KEY                                 │
│ pr_id       │ BIGINT FK → pull_requests                         │
│ reviewer_id │ BIGINT FK → users                                  │
│ state       │ ENUM('approved','changes_requested','commented',   │
│             │      'dismissed','pending')                        │
│ body        │ TEXT (overall review comment)                      │
│ submitted_at│ TIMESTAMP                                          │
│ commit_id   │ CHAR(40) ← which commit was reviewed              │
└──────────────────────────────────────────────────────────────────┘

review_comments table:
┌──────────────────────────────────────────────────────────────────┐
│ id          │ BIGINT PRIMARY KEY                                 │
│ review_id   │ BIGINT FK → reviews                               │
│ pr_id       │ BIGINT FK → pull_requests                         │
│ path        │ TEXT ← file path                                   │
│ position    │ INT ← line position in the diff                    │
│ original_position │ INT ← position in original diff             │
│ commit_id   │ CHAR(40) ← commit this comment is on              │
│ diff_hunk   │ TEXT ← the diff context captured at review time   │
│ body        │ TEXT (markdown)                                    │
│ created_at  │ TIMESTAMP                                          │
└──────────────────────────────────────────────────────────────────┘
```

### The `refs/pull/*/head` Refs (Git-Level)

On GitHub's side, they maintain a special Git namespace for PR refs:

```
# These refs exist on the REMOTE (GitHub), not locally:
refs/pull/42/head    → "a3f2c1d..."  ← tip of feature/login at PR open
refs/pull/42/merge   → "9b2e3f1..."  ← GitHub's test merge commit

# The test merge commit looks like:
tree   <merged tree SHA>
parent <main HEAD>    ← first parent (base)
parent <feature HEAD> ← second parent (head)
author GitHub <noreply@github.com>
committer GitHub <noreply@github.com>

Merge pull request #42 from feature/login
```

GitHub regenerates `refs/pull/42/merge` every time either `main` or `feature/login` changes. This is how GitHub determines if a PR is "mergeable" or "has conflicts" — it tries the merge and checks if it succeeds.

### Fetch Any PR Locally

```bash
# Fetch PR #42 by its ref:
git fetch origin refs/pull/42/head:refs/pr/42
git checkout refs/pr/42

# Or use GitHub CLI:
gh pr checkout 42

# What gh pr checkout does internally:
# 1. gh pr view 42 --json headRefName → "feature/login"
# 2. git fetch origin feature/login
# 3. git checkout -b feature/login --track origin/feature/login
```

---

## 19. Mental Model Checkpoint

Test yourself — can you answer these from memory?

**1.** What is the difference between a "Comment" review and a "Request Changes" review? What effect does each have on the merge button?

**2.** You push a new commit to a PR branch after a reviewer has approved it. What happens to the approval? What setting controls this?

**3.** A PR has 4 commits: "add login", "fix test", "fix typo", "address review comments". You choose "Squash and merge." What does the history on `main` look like after? What SHA does the feature branch still point to?

**4.** How does GitHub determine what diff to show in a PR? What is the starting point of the diff? (Hint: it's not the tip of `main`.)

**5.** What is `refs/pull/42/merge` and how does GitHub use it?

**6.** A CODEOWNERS file has `src/api/**  @backend-team`. A PR touches both `src/api/auth.js` and `src/ui/Button.jsx`. No CODEOWNERS rule covers `src/ui/`. What happens?

**7.** You want to review a contributor's PR from a fork locally without adding their remote. What command lets you do this by PR number?

**8.** What is a Draft PR? List 3 ways it behaves differently from a regular PR.

**9.** Your team uses "Rebase and merge." A PR has 3 commits with SHAs `A`, `B`, `C` branched from `main` at commit `E`. After merge, `main` contains new commits `A'`, `B'`, `C'`. Why are the SHAs different?

**10.** `Closes #15` is in a PR description. The PR is merged into `main`. Into what branch must `main` be the default branch for the auto-close to trigger? What if the PR is merged into `develop`?

<details>
<summary>Answers</summary>

**1.** "Comment" has no effect on the merge button. "Request Changes" blocks merge even if other reviewers have approved — the requester's own approval (or dismissal by a maintainer) is required to unblock.

**2.** If "Dismiss stale pull request approvals when new commits are pushed" is enabled in branch protection, the approval is dismissed and the reviewer must re-approve. If disabled, the approval remains (less safe).

**3.** `main` has one new commit — the squash commit containing the combined diff of all 4 commits. The feature branch still points to its original tip (the "address review comments" commit SHA). Squash doesn't move or delete the source branch pointer.

**4.** GitHub diffs from the **merge base** (git merge-base of head and base) to the head commit. Not from the tip of `main`. This means only commits that diverged from the common ancestor are shown.

**5.** `refs/pull/42/merge` is a test-merge commit that GitHub creates by merging `feature/login` into `main`. It's used to: (a) determine if the merge is clean (no conflicts), and (b) run CI against the merged state, not just the feature branch.

**6.** `src/api/auth.js` requires `@backend-team` approval. `src/ui/Button.jsx` falls through to the `*` catch-all (if one exists) or has no required owner. Both rules apply independently.

**7.** `git fetch origin pull/42/head:pr-42` then `git checkout pr-42`. Or `gh pr checkout 42`.

**8.** Draft PR: (1) Merge button is disabled, (2) CODEOWNERS are not auto-assigned, (3) Not shown in "Needs review" filter for reviewers. CI still runs, comments can still be left.

**9.** Rebase replays commits on top of new parent. The SHA of a commit is a hash of its content + parent SHA. Since the parent of `A'` is now `E` (main's tip) instead of the original parent, the SHA changes. Similarly B' depends on A', C' depends on B'.

**10.** Auto-close only triggers when the PR is merged into the **repository's default branch** (usually `main`). If merged into `develop`, the issue remains open. This is a common Gitflow pitfall — use `develop` as default branch if using Gitflow, or manually close issues.

</details>

---

## 20. Connect Everything Backwards

| This Topic's Concept | Connects To |
|---------------------|-------------|
| PR diff is computed from **merge base** | Topic 09 — 3-way merge uses the same merge base computation. The PR diff is literally `git diff <merge-base>..<head>`. |
| PR branches are **remote tracking branches** | Topic 10 — `refs/remotes/origin/feature/login` is updated when you push. The PR tracks `origin/feature/login`. |
| **Rebase before PR** to clean history | Topic 11 — Interactive rebase to squash "wip" and "fix review" commits before the PR is reviewed. The `--autosquash` flag was invented exactly for this workflow. |
| **The three merge options** in the GitHub UI | Topic 09 (merge commit), Topic 11 (rebase), Topic 09 (squash = merge with `--squash`). The same Git operations, just triggered from the UI. |
| **Squash and merge** creates a linear history | Topic 15 — This is the technical implementation of "GitHub Flow" where every commit on main = one PR. |
| **Branch protection** requires checks pass | Topic 08 — Branch protection rules wrap around the ref update mechanism. Git allows any force-push; branch protection is GitHub's enforcement layer on top. |
| **CODEOWNERS** routes reviews by file path | Topic 01 — Files in `.git/` (working directory) have paths; CODEOWNERS maps those paths to owners using glob patterns. |
| `refs/pull/42/head` special refs | Topic 03 — Refs are just files containing SHAs. GitHub creates these refs in their server-side `.git/refs/pull/` namespace. You can fetch them the same way you fetch any ref. |
| **`git fetch origin pull/42/head`** | Topic 10 — This is a refspec fetch: `git fetch <remote> <src>:<dst>`. The same mechanism as `git fetch origin main:main`. |

---

## 21. Quick Reference Card

### PR Operations

| Action | Command / Location |
|--------|-------------------|
| Create PR from CLI | `gh pr create --title "..." --body "..."` |
| Create draft PR | `gh pr create --draft` |
| List open PRs | `gh pr list` |
| View PR details | `gh pr view 42` |
| Checkout PR branch | `gh pr checkout 42` |
| Fetch PR branch by number | `git fetch origin pull/42/head:pr-42` |
| Mark PR ready for review | `gh pr ready 42` |
| Approve a PR | `gh pr review 42 --approve` |
| Request changes | `gh pr review 42 --request-changes --body "..."` |
| Merge PR (merge commit) | `gh pr merge 42 --merge` |
| Merge PR (squash) | `gh pr merge 42 --squash` |
| Merge PR (rebase) | `gh pr merge 42 --rebase` |
| Close PR without merging | `gh pr close 42` |
| Reopen PR | `gh pr reopen 42` |

### Review Commands

| Action | Command |
|--------|---------|
| Add inline comment | GitHub UI only (click `+` on line in diff) |
| Add suggestion | ` ```suggestion ` block in comment |
| Resolve a thread | "Resolve conversation" button |
| Re-request review | `gh pr review 42 --request-changes` or UI |
| Dismiss a review | `gh api repos/:owner/:repo/pulls/42/reviews/:review_id/dismissals -X PUT` |

### Git Commands Useful in PR Context

| Command | What it does |
|---------|-------------|
| `git diff main...feature/login` | Show what the PR would show (diff from merge base) |
| `git log main..feature/login` | Commits in PR branch not in main |
| `git log --oneline main..feature/login` | Compact list of PR commits |
| `git rebase -i main` | Interactively clean up commits before review |
| `git fetch origin` | Sync remote-tracking branches (incl. review updates) |
| `git push --force-with-lease` | Force-push only if remote hasn't been updated |

### Branch Protection Key Settings

| Setting | What it enforces |
|---------|-----------------|
| Require pull request reviews | Cannot push directly to protected branch |
| Required approving reviews: N | N approvals needed before merge |
| Require review from Code Owners | CODEOWNERS approval required |
| Dismiss stale approvals | New push → existing approvals invalidated |
| Require status checks to pass | Named CI checks must be green |
| Require branches to be up to date | Must be current with base before merge |
| Do not allow bypassing | Applies even to admins |

---

## 22. When Would I Use This At Work?

### Monday Morning: Starting a New Feature

```
1. Pick up task from Jira/Linear: "AUTH-204: Add 2FA via TOTP"
2. git checkout main && git pull origin main
3. git checkout -b feature/auth-204-totp-2fa
4. Immediately: gh pr create --draft --title "feat(auth): add TOTP 2FA (AUTH-204)"
   ↑ Opens a draft PR immediately — CI starts running on every push
   ↑ Your teammates can see you're working on this
5. Work on the feature over 2-3 days, pushing commits daily
6. When ready for review:
   - Self-review: gh pr view --web, click Files Changed, review your own diff
   - Clean up commits: git rebase -i main (squash the "wip" commits)
   - Fill in the PR template completely
   - gh pr ready  ← converts draft to ready for review
```

### Tuesday: Giving a Code Review

```
1. GitHub notifies you: "Review requested: feature/payments-stripe-v2"
2. gh pr checkout 89  ← get the code locally
3. Read the PR description first — understand the WHY
4. Run the code locally with the feature: npm test, manual exploration
5. Go through Files Changed systematically using the checklist from Section 13
6. Leave labeled comments: "nit:", "issue:", "q:", "blocker:"
7. Submit review:
   - If issues found: Request Changes with summary
   - If only nits: Approve with comment "Approved — nits optional"
   - If serious issue: Request Changes, ping in Slack for urgency
```

### Wednesday: Your PR Gets Feedback

```
1. Review notification: "Bob requested changes on your PR"
2. Read every comment — don't respond immediately if frustrated
3. For each comment:
   - Agree: make the change, git commit --fixup (see Topic 07)
   - Disagree: ask a clarifying question in the PR comment thread
   - Nit: decide whether to apply (it's your call, but be gracious)
4. When all addressed:
   git rebase -i --autosquash main  ← fold fixup commits into their parents
   git push --force-with-lease origin feature/auth-204-totp-2fa
5. Respond to each comment: "Fixed in latest push" or "Kept as-is because X"
6. Re-request review: gh pr review --approve (no — request re-review from reviewer)
   → click "Re-request review" button in UI
```

### Thursday: Hotfix at 2pm

```
1. Alert: 503 errors on /api/orders in production
2. git checkout main && git pull origin main
3. git checkout -b hotfix/fix-orders-503
4. Diagnose: git log --oneline -20, git bisect (Topic 21)
5. Minimal fix, commit with "fix:" prefix
6. gh pr create --title "fix: [HOTFIX] orders 503 due to nil pointer" \
   --label hotfix,critical --reviewer alice
7. Slack: "@alice hotfix PR ready, need immediate review"
8. Alice reviews in 10 minutes (it's 8 lines changed)
9. Merge (squash), deploy, monitor
10. Open follow-up PR: "test: add coverage for nil pointer in orders handler"
```

### Friday: Merging the Week's Work

```
Your feature PR is approved and CI is green.
Before merging:
1. Is main ahead of my branch?
   git fetch origin
   git log HEAD..origin/main --oneline
   → If yes: git rebase origin/main (or GitHub "Update branch" button)
2. Check that CI is green on the latest commit (after the rebase push)
3. Choose merge strategy per team policy:
   - Team uses squash → gh pr merge --squash
   - Team uses merge commits → gh pr merge --merge
4. After merge: gh pr delete (cleanup if enabled)
5. Delete local branch: git branch -d feature/auth-204-totp-2fa
6. Verify: git log main --oneline -5 (see your PR commit)
```

---

*End of Topic 16 — Pull Requests & Code Review Culture*

**Next**: Topic 17 — Merge Strategies at Scale (squash vs merge commit vs rebase-merge, when each breaks down, octopus merges, rerere)
