# Topic 17 — Merge Strategies at Scale

> **Curriculum**: Git & GitHub Deep Mastery — Zero → Principal Engineer  
> **Phase 5**: Team & Company-Level Git (Topics 15–19)  
> **Prerequisites**: Topic 09 (Merging Internals), Topic 11 (Rebasing), Topic 15 (Workflows), Topic 16 (Pull Requests)

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The Filing Cabinet Analogy](#1-eli5-anchor)
2. [The Three Merge Strategies — Overview](#2-the-three-merge-strategies)
3. [Physical Architecture — What Each Strategy Writes to `.git/`](#3-physical-architecture)
4. [Strategy 1: Merge Commit (`--no-ff`)](#4-strategy-1-merge-commit)
5. [Strategy 2: Squash and Merge (`--squash`)](#5-strategy-2-squash-and-merge)
6. [Strategy 3: Rebase and Merge](#6-strategy-3-rebase-and-merge)
7. [The `recursive` vs `ort` Merge Algorithms](#7-the-recursive-vs-ort-merge-algorithms)
8. [The Octopus Merge — Merging Multiple Branches at Once](#8-the-octopus-merge)
9. [rerere — Reuse Recorded Resolution](#9-rerere)
10. [Merge Strategy Flags (`-s ours`, `-s theirs`, `-X` options)](#10-merge-strategy-flags)
11. [`git merge-base` — Finding the Common Ancestor](#11-git-merge-base)
12. [Choosing a Strategy for Your Team — Decision Framework](#12-choosing-a-strategy)
13. [What Breaks at Scale — 5 Real Failure Modes](#13-what-breaks-at-scale)
14. [Mistakes → Root Cause → Fix → Prevention](#14-mistakes)
15. [Byte-Level: The Merge Commit Object vs Squash Commit Object](#15-byte-level-internals)
16. [Mental Model Checkpoint](#16-mental-model-checkpoint)
17. [Connect Everything Backwards](#17-connect-everything-backwards)
18. [Quick Reference Card](#18-quick-reference-card)
19. [When Would I Use This At Work?](#19-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The Library Catalogue Analogy

Imagine a library that receives donated books from three different donors every week.

**Strategy 1 — Merge Commit (keep everything):**  
The librarian adds ALL the books to the shelves exactly as received, then files a special card in the catalogue that says: "On June 1st, we received a batch from Alice containing these 4 books." The catalogue preserves the full story of where every book came from and when.

**Strategy 2 — Squash:**  
The librarian takes all of Alice's books, labels them "Alice's June Donation," and files one single catalogue card that says "1 donation from Alice." Individual books lose their individual origin story, but the catalogue is clean and easy to browse.

**Strategy 3 — Rebase:**  
The librarian takes Alice's books and shelves them one by one, re-stamping each with today's date and "shelved by librarian." The books are all there individually, but their original "donated by Alice on June 1st" stamps are gone — replaced with the librarian's re-shelving date.

Each approach is valid. The question is: **what does your team need the catalogue to tell future visitors?**

---

## 2. The Three Merge Strategies — Overview

Before going deep, see the full picture:

```
STARTING STATE (same for all strategies):

        A---B---C   ← feature/login (3 commits)
       /
  D---E---F---G     ← main (HEAD=G)

  E is the merge base (common ancestor of C and G)
```

```
AFTER MERGE COMMIT (--no-ff):

        A---B---C   ← feature/login (still here, unchanged)
       /           \
  D---E---F---G---M  ← main (HEAD=M, M is merge commit with 2 parents)

  Objects created: 1 merge commit M
  Commits on main: F, G, A, B, C all visible via M's history
  History shape: non-linear (diamond)
```

```
AFTER SQUASH:

        A---B---C   ← feature/login (still here, unchanged, not moved)
       /
  D---E---F---G---S  ← main (HEAD=S, S is squash commit with 1 parent)

  Objects created: 1 new squash commit S
  Commits A, B, C: NOT on main's history (only their combined diff is in S)
  History shape: linear
```

```
AFTER REBASE AND MERGE:

        A---B---C   ← feature/login (still here, unchanged)
       
  D---E---F---G---A'---B'---C'  ← main (HEAD=C')

  Objects created: 3 new commit objects A', B', C' (same diffs as A, B, C but new SHAs)
  Original A, B, C: NOT on main (new SHAs A', B', C' replace them)
  History shape: linear
```

---

## 3. Physical Architecture — What Each Strategy Writes to `.git/`

### Before Any Merge

```
.git/
├── HEAD                       → "ref: refs/heads/main"
├── refs/
│   ├── heads/
│   │   ├── main               → "G-SHA"  (7c3e8fa...)
│   │   └── feature/login      → "C-SHA"  (a3f2c1d...)
│   └── remotes/
│       └── origin/
│           ├── main           → "G-SHA"
│           └── feature/login  → "C-SHA"
└── objects/
    ├── 7c/3e8fa...             (G commit object)
    ├── a3/f2c1d...             (C commit object)
    ├── ...                    (A, B, D, E, F commit objects + all trees + blobs)
```

### After Merge Commit (`git merge --no-ff feature/login`)

```
.git/ CHANGES:
├── refs/
│   └── heads/
│       └── main               → "M-SHA"  ← UPDATED (was G-SHA)
│                                           feature/login UNCHANGED at C-SHA
└── objects/
    └── 8f/3a1b9...             ← NEW: merge commit M object

MERGE_HEAD file: DELETED (was written during merge, now gone after success)
MERGE_MSG file:  DELETED (was the auto-generated merge message)

TOTAL NEW OBJECTS: 1 (just the merge commit)
```

### After Squash (`git merge --squash feature/login && git commit`)

```
.git/ CHANGES:
├── refs/
│   └── heads/
│       └── main               → "S-SHA"  ← UPDATED
│                                           feature/login UNCHANGED at C-SHA
└── objects/
    └── 4d/e8f2a...             ← NEW: squash commit S

SQUASH_HEAD file: DELETED (written by --squash, deleted after commit)

TOTAL NEW OBJECTS: 1 (the squash commit) + possibly 1 new tree if different
```

### After Rebase and Merge

```
.git/ CHANGES:
├── refs/
│   └── heads/
│       └── main               → "C'-SHA"  ← UPDATED
│                                            feature/login UNCHANGED at C-SHA
└── objects/
    ├── b1/c2d3e...             ← NEW: A' commit object
    ├── f4/a5b6c...             ← NEW: B' commit object
    └── 9e/8d7c6...             ← NEW: C' commit object

TOTAL NEW OBJECTS: 3 (one per original commit, all with new SHAs)
```

### Key Observation: Feature Branch is NEVER Modified

All three strategies leave `feature/login` pointing to `C` (original SHA). The feature branch is never touched. The merge only affects `main`'s ref pointer and adds new objects. The feature branch can be deleted afterward — the commits it pointed to are preserved in `main`'s history (for merge commit and rebase), or their combined diff is (for squash).

---

## 4. Strategy 1: Merge Commit (`--no-ff`)

### The Git Command

```
git merge --no-ff feature/login
           │
           └─ --no-ff: "no fast-forward"
                       Forces a merge commit EVEN IF fast-forward is possible.
                       Without this, if main hasn't diverged, Git fast-forwards
                       (just moves main pointer to C — no merge commit created).
                       
                       --no-ff guarantees:
                       • A merge commit is always created
                       • The feature branch existence is preserved in history
                       • You can always see "this was a feature branch"
```

### What Is a Merge Commit?

A merge commit is a commit with **two (or more) parent pointers**. This is the only difference from a regular commit.

```
Regular commit object format (from Topic 02):
  tree   <tree-SHA>
  parent <single-parent-SHA>
  author Name <email> timestamp
  committer Name <email> timestamp
  
  Commit message

Merge commit object format:
  tree   <merged-tree-SHA>
  parent <G-SHA>            ← first parent: was HEAD of main
  parent <C-SHA>            ← second parent: tip of feature branch
  author Name <email> timestamp
  committer Name <email> timestamp
  
  Merge branch 'feature/login' into main
```

The **first parent** is the branch you were ON when you ran `git merge`. This matters for `git log --first-parent` (see below).

### The Fast-Forward Special Case

```
SCENARIO: No new commits on main since feature branched

        A---B---C   ← feature/login
       /
  D---E             ← main (HEAD=E, same as merge base)

Fast-forward possible because main is a direct ancestor of C.

git merge feature/login  (WITHOUT --no-ff):
  Result: main pointer moves to C. No new object created.
  
  D---E---A---B---C  ← main (HEAD=C)
  
  After: no trace that A, B, C were on a "feature branch"
  After: no merge commit. Cannot revert "the feature" as one unit.

git merge --no-ff feature/login:
  Result: merge commit M created. main → M.
  
        A---B---C
       /         \
  D---E-----------M  ← main (HEAD=M)
  
  After: history shows "these commits came from a feature branch"
  After: can revert M to undo the entire feature atomically
```

### Prove It Hands-On

```bash
# Setup
git init merge-demo && cd merge-demo
git commit --allow-empty -m "initial"         # D
git checkout -b feature/login
git commit --allow-empty -m "A: add login"    # A
git commit --allow-empty -m "B: add session"  # B
git commit --allow-empty -m "C: add logout"   # C

# Back to main — no new commits (fast-forward scenario)
git checkout main

# Try WITHOUT --no-ff first:
git merge feature/login
git log --oneline --graph
# Expected: linear — no merge commit
# A, B, C appear directly on main

# Reset
git reset --hard HEAD~3  # undo the fast-forward

# Now WITH --no-ff:
git merge --no-ff feature/login -m "Merge feature/login"
git log --oneline --graph
# Expected: diamond shape with merge commit M

# PROVE the merge commit has two parents:
MERGE_SHA=$(git rev-parse HEAD)
git cat-file -p $MERGE_SHA
# Expected output shows TWO "parent" lines
```

### `--first-parent` — Following the Mainline

When you have merge commits, `git log` shows ALL commits from ALL branches merged in. This can be noisy. `--first-parent` shows only the "mainline" commits (the first parent chain):

```bash
git log --oneline --first-parent main
# Shows: M, G, F, E, D
# Does NOT show: A, B, C (they are second-parent history)

# This is how you answer: "what PRs landed on main and when?"
# Each merge commit M represents one PR.
```

### When to Use Merge Commits

```
USE MERGE COMMITS WHEN:
  ✅ Gitflow (merge feature/* → develop, develop → main)
  ✅ Regulatory environments needing full audit trail
  ✅ You need to be able to `git revert <merge-commit>` to undo a whole feature
  ✅ Large, long-lived features where individual commit context matters
  ✅ Open source projects where contributor commit authorship matters

AVOID MERGE COMMITS WHEN:
  ❌ Many small PRs per day (history becomes unreadable)
  ❌ Team uses GitHub Flow (clean linear history is the goal)
  ❌ Developers commit "wip", "fix", "asdf" — squash those first
```

---

## 5. Strategy 2: Squash and Merge (`--squash`)

### The Git Command

```
git merge --squash feature/login
           │
           └─ --squash: 
                Applies all the changes from feature/login to the working tree
                and staging area WITHOUT creating a commit.
                You then run `git commit` to create a single new commit.
                
                The result: one new commit containing the combined diff.
                The feature branch commits are NOT added to history.
                The merge commit object is NOT created (no two-parent object).
```

### Step-by-Step Execution

```
COMMAND SEQUENCE:
git merge --squash feature/login
git commit -m "feat(auth): add login endpoint (#42)"

STEP 1: git merge --squash feature/login
  READS:   merge base E, feature/login tip C
  COMPUTES: diff from E to C (the complete change set across A+B+C)
  WRITES:  working tree and staging area (as if you hand-applied all changes)
  WRITES:  .git/SQUASH_HEAD → "C-SHA"  (records what was squashed)
  DOES NOT: create any commit object
  DOES NOT: move main pointer

STEP 2: git commit -m "..."
  READS:   staging area
  COMPUTES: new commit object
  WRITES:  .git/objects/<new squash commit S>
  MODIFIES: .git/refs/heads/main → "S-SHA"
  DELETES:  .git/SQUASH_HEAD
```

### The SQUASH_HEAD File

```bash
# After git merge --squash, before git commit:
cat .git/SQUASH_HEAD
# Output: a3f2c1d8...  (the SHA of feature/login's tip)

# This is how Git knows which commits to reference in the squash commit message
# GitHub CLI uses this to auto-generate: "Squashed commits from feature/login"
# After git commit, this file is deleted
```

### What the Squash Commit Looks Like

```
S (squash commit):
  tree   <combined-tree-SHA>   ← final state: same as if you had merge committed
  parent <G-SHA>               ← ONLY ONE PARENT: main's previous HEAD
  author alice <alice@co.com> 1748433600 +0000
  committer alice <alice@co.com> 1748433600 +0000
  
  feat(auth): add login endpoint (#42)
  
  Squashed commits from feature/login:
  - A: add login route
  - B: add session handling  
  - C: add logout route
```

**Critical**: Only one `parent` line. This commit looks exactly like a regular commit — no indication it came from a feature branch at the Git level. The `(#42)` in the message is the only human-readable reference.

### Squash and `git log --follow`

```bash
# After squash merge, try to trace a file's history:
git log --follow -- src/auth/login.js

# You will see: S (the squash commit) and any commits before the feature branch.
# You will NOT see A, B, C.
# You cannot drill into "why was this line added in A?" because A is not in main's history.

# Compare to merge commit:
git log --follow -- src/auth/login.js  # (with merge commit)
# Shows: M, C, B, A — the full history including within the feature branch
```

### Prove It Hands-On

```bash
# After setup from Section 4 (merge-demo repo):
git reset --hard origin/main 2>/dev/null || git reset --hard HEAD~1  # reset main

# Squash merge:
git merge --squash feature/login
ls .git/SQUASH_HEAD             # PROVE: file exists before commit
cat .git/SQUASH_HEAD            # PROVE: contains feature branch tip SHA
git commit -m "feat: add login (#42)"
ls .git/SQUASH_HEAD             # PROVE: file is gone after commit

# Inspect the squash commit:
git cat-file -p HEAD
# PROVE: only ONE "parent" line (unlike merge commit which has two)

# Check history:
git log --oneline --graph
# PROVE: linear history, no diamond, no A/B/C commits visible
```

---

## 6. Strategy 3: Rebase and Merge

### What "Rebase and Merge" Actually Does

This is **not** a single `git rebase` command. It is two operations:

```
STEP 1: Rebase the feature branch onto main
git rebase main feature/login
  → Creates new commits A', B', C' on top of G

STEP 2: Fast-forward merge (since A'B'C' is now ahead of main)
git checkout main
git merge --ff-only feature/login
  → Moves main pointer to C'
```

GitHub's "Rebase and merge" button does both steps for you. But understanding both is critical.

### The Rebase Step — What Changes in Each Commit

```
BEFORE REBASE:
A's commit object:
  tree   <tree-A>
  parent <E-SHA>          ← branched from E
  author alice ...
  committer alice ...

AFTER REBASE (A' is a new object):
A' commit object:
  tree   <tree-A>         ← SAME tree (same file contents)
  parent <G-SHA>          ← NEW parent: G (tip of main)
  author alice ...        ← SAME author
  committer GitHub ...    ← DIFFERENT committer (GitHub bot, not alice)

SHA(A') ≠ SHA(A) because SHA is computed from ALL fields including parent.
```

### The `--ff-only` Fast-Forward

After rebase, `feature/login` (now pointing at C') is a direct descendant of `main` (pointing at G). So the merge is always a fast-forward — no merge commit created.

```bash
git checkout main
git merge --ff-only feature/login
# This ONLY succeeds if feature/login is ahead of main (fast-forward possible)
# If it's not possible, the command FAILS (safety net: can't accidentally create merge commit)
```

### Rebase and Merge vs. Local Interactive Rebase

These are different:

| | `git rebase -i main` (local) | GitHub "Rebase and merge" |
|--|--|--|
| Who rebases | You, interactively | GitHub's server, automatically |
| Can squash/edit | ✅ Yes (interactive) | ❌ No — all commits kept |
| Committer field | Your git config | `GitHub <noreply@github.com>` |
| When it happens | Before push / before PR | At merge time |
| Force push needed | Usually yes | No (GitHub handles it) |

### When Rebase and Merge Breaks

```
SCENARIO: Two developers both have PRs open targeting main.
          PR A and PR B were both branched from commit G.

PR A merges first (rebase and merge):
  main: G---A'---B'---C'

Now PR B tries to "Rebase and merge":
  PR B's head is still D---E---F (branched from G)
  Rebasing D/E/F onto C' works fine, creates D'---E'---F'
  main: G---A'---B'---C'---D'---E'---F'

PROBLEM: This rebase must be re-done every time main advances.
GitHub "Rebase and merge" re-runs the rebase fresh each time.
But if PR B has conflicts with PR A's changes, the rebase FAILS.
The author must manually resolve and re-push.
```

### Prove It Hands-On

```bash
# After setup from Section 4:
git log --oneline --graph  # see current state

# Rebase feature onto main (simulate what GitHub does):
git rebase main feature/login
# Creates A', B', C' on top of main

# Check new SHAs:
git log --oneline feature/login -3
# SHAs are DIFFERENT from original A, B, C

# Verify original A, B, C still exist (just unreferenced):
git reflog | grep "A\|B\|C"
# They're still in reflog (will be GC'd after 30 days)

# Fast-forward merge:
git checkout main
git merge --ff-only feature/login
git log --oneline --graph
# PROVE: linear history, commits A'B'C' appear directly on main, no merge commit
```

---

## 7. The `recursive` vs `ort` Merge Algorithms

When a merge has conflicts to resolve (not just pointer arithmetic), Git uses a **merge algorithm**. These are the `-s` strategy (lowercase s = merge strategy algorithm).

Do not confuse with "merge strategy" (squash/merge-commit/rebase) — those are workflow choices. These are the **algorithms** Git uses internally to combine content.

### The Three Algorithms

```
git merge -s resolve    ← oldest, O(n²), handles 1 merge base only
git merge -s recursive  ← default pre-Git 2.33, handles criss-cross merges
git merge -s ort        ← default since Git 2.33, "Ostensibly Recursive's Twin"
                           Same behavior as recursive but O(n) time complexity
git merge -s octopus    ← for merging 3+ branches simultaneously
git merge -s ours       ← "merge" that discards all changes from theirs
git merge -s subtree    ← for merging a repo into a subdirectory
```

### The Criss-Cross Merge Problem (Why `recursive` Exists)

```
SCENARIO: criss-cross history

   A---C
  / \ / \
 /   X   \
/   / \   \
B---D   ?

When merging the two branches, there are TWO possible merge bases: C and D.
"resolve" picks one arbitrarily (wrong).
"recursive" RECURSIVELY merges the two merge bases first to get a virtual base,
then uses that virtual base for the real merge.
```

```bash
# You almost never need to specify this directly.
# Git chooses 'ort' automatically since Git 2.33.
# To check your Git version:
git --version

# To force a specific algorithm (rarely needed):
git merge -s recursive feature/login    # old algorithm
git merge -s ort feature/login          # new algorithm (explicit)
```

### The `ort` Algorithm Performance Improvement

```
Benchmark: Merging a branch with 1000 file changes into main

Algorithm   | Time    | Memory | Notes
────────────|─────────|--------|─────────────────────────────
resolve     | 45s     | 800MB  | Fails on criss-cross
recursive   | 38s     | 650MB  | Correct but exponential worst-case
ort         | 2s      | 120MB  | O(n) — index-based, same result as recursive

Why ort is faster:
- Does not modify the working tree or index during merge computation
- Computes the merged result in-memory first
- Only touches disk when applying the final result
- No side effects if the merge is aborted mid-way (safer)
```

---

## 8. The Octopus Merge — Merging Multiple Branches at Once

### What Is an Octopus Merge?

A merge with **more than two parents**. Git supports this for "pulling together" multiple feature branches simultaneously.

```
SCENARIO: Release manager wants to land 3 features simultaneously

        A---B   ← feature/auth
       /
  D---E---F---G  ← main
       \
        H---I   ← feature/payments
         \
          J---K  ← feature/logging


git checkout main
git merge feature/auth feature/payments feature/logging

Result:
        A---B
       /       \
  D---E---F---G---M   ← main (HEAD=M, M has FOUR parents: G, B, I, K)
       \       /
        H---I /
         \ /
          J---K
```

### When Octopus Merges Are Used

In practice, octopus merges are rare in application code. They are used by:
- **The Linux kernel**: Linus Torvalds regularly creates "octopus" merges combining 8-10 branches from different subsystem maintainers into a release commit
- **Release automation**: Some CI systems merge all release PRs together
- **Testing integration**: "Does the combination of these 3 features work together?"

### Octopus Merge Limitations

```bash
# Octopus merge FAILS if there are conflicts between any of the branches.
# This is by design — the octopus algorithm cannot resolve conflicts.
# If conflict detected: Git aborts with:
#   "Merge requires file-level merging, which is not supported by octopus strategy"
# 
# Solution: resolve conflicts between branches pairwise first, then octopus.
```

### Prove It Hands-On

```bash
git init octopus-demo && cd octopus-demo
git commit --allow-empty -m "root"
git checkout -b branch-a && git commit --allow-empty -m "A1" && git commit --allow-empty -m "A2"
git checkout main
git checkout -b branch-b && git commit --allow-empty -m "B1"
git checkout main

git merge branch-a branch-b -m "Octopus merge: A + B"

# Inspect the merge commit:
git cat-file -p HEAD
# PROVE: THREE "parent" lines (root, tip of A, tip of B)

git log --oneline --graph
# PROVE: diamond with 3 arms converging
```

---

## 9. `rerere` — Reuse Recorded Resolution

### The Problem `rerere` Solves

```
SCENARIO: Long-running feature branch

You're on feature/big-refactor. You periodically rebase onto main.
Each time you rebase, you hit the SAME conflict in src/config.js.
Each time, you manually resolve it the same way.

After 15 rebases over 3 weeks, you've resolved the same conflict 15 times.
```

`rerere` = **Re**use **Re**corded **Re**solution. It records how you resolved a conflict and automatically applies that same resolution the next time the same conflict appears.

### How `rerere` Works Physically

```
ENABLE RERERE:
git config --global rerere.enabled true
git config --global rerere.autoupdate true  # also auto-stages resolved files

WHAT GETS WRITTEN TO .git/:

.git/rr-cache/
├── <conflict-SHA>/
│   ├── preimage    ← the conflicted state (both sides) before resolution
│   └── postimage   ← your resolution
```

The conflict SHA is derived from the conflict markers content (independent of the file name or commit SHA). Same conflict pattern → same rr-cache entry.

### `rerere` Lifecycle

```
FIRST TIME you hit a conflict:

COMMAND: git merge feature/big-refactor
  → Conflict in src/config.js
  → rerere records the preimage: .git/rr-cache/<sha>/preimage
  → You manually edit src/config.js to resolve
  
COMMAND: git add src/config.js
  → rerere records the postimage: .git/rr-cache/<sha>/postimage
  
COMMAND: git commit
  → rerere notes: "this conflict + this resolution = learned"


SECOND TIME you hit the same conflict (next rebase):

COMMAND: git rebase main
  → Same conflict appears in src/config.js
  → rerere recognizes the preimage matches .git/rr-cache/<sha>/preimage
  → rerere AUTOMATICALLY applies the postimage (your previous resolution)
  → If rerere.autoupdate=true: also stages the file
  → You just need: git rebase --continue (no manual conflict resolution!)
```

### Hands-On Proof

```bash
git config --global rerere.enabled true

# Create a conflict scenario:
git init rerere-demo && cd rerere-demo
echo "color: blue" > config.txt && git add . && git commit -m "initial"
git checkout -b feature-a
echo "color: red" > config.txt && git add . && git commit -m "feature-a: red"
git checkout main
echo "color: green" > config.txt && git add . && git commit -m "main: green"

# First merge (conflict):
git merge feature-a
# → Conflict in config.txt

# Check that rerere recorded the preimage:
ls .git/rr-cache/
# PROVE: a directory was created

# Resolve manually:
echo "color: purple" > config.txt
git add config.txt
git commit -m "Merge feature-a (purple)"
# rerere records postimage

# Undo and redo the merge to test rerere:
git reset --hard HEAD~1
git merge feature-a
# PROVE: conflict IS auto-resolved by rerere
cat config.txt  # Should already say "color: purple"
ls .git/rr-cache/*/postimage  # postimage file exists
```

---

## 10. Merge Strategy Flags (`-s ours`, `-s subtree`, `-X` options)

### The `-s ours` Strategy

```
git merge -s ours feature/old-feature
```

This creates a merge commit that **discards all changes from the other branch**. The resulting tree is identical to your current branch. But the merge commit records that the other branch was "merged."

**When would you use this?**

```
USE CASE: Retiring a branch without losing its history

SCENARIO: You realize feature/old-feature's approach was wrong.
          You've already reimplemented it in feature/new-feature.
          But you don't want feature/old-feature to stay "open" —
          it should be marked as merged to avoid confusion.

git checkout main
git merge -s ours feature/old-feature -m "Retire old-feature (replaced by new-feature)"
# Result: merge commit recorded, but main's content is unchanged
# feature/old-feature is now "merged" in the graph
```

```
USE CASE: Gitflow branch maintenance

After releasing v2.0:
  git checkout develop
  git merge -s ours main
  # Tells Git: "develop has seen all of main's history"
  # Prevents false conflicts in future merges
  # (develop already has the hotfix code through other means)
```

### The `-X` Extended Strategy Options

`-X` passes options to the merge algorithm (recursive/ort):

```bash
git merge -X ours feature/login
#          │
#          └─ -X ours: when there is a conflict on a line,
#                      automatically take OUR version
#                      (NOT the same as -s ours — only applies to conflicts)

git merge -X theirs feature/login
#          └─ -X theirs: when conflict, take THEIR version automatically

git merge -X patience feature/login
#          └─ patience: use the "patience" diff algorithm
#                       Better at detecting moved blocks, fewer false conflicts
#                       Slower but produces cleaner diffs on large refactors

git merge -X ignore-space-change feature/login
#          └─ ignore whitespace-only differences when comparing lines

git merge -X ignore-all-space feature/login
#          └─ ignore ALL whitespace (treats "x = 1" same as "x=1")

git merge -X diff-algorithm=histogram feature/login
#          └─ use histogram diff algorithm (best for most code — default in Git 2.x)
```

### `-s ours` vs `-X ours` — Critical Distinction

```
-s ours    = STRATEGY: the entire merge strategy is "take ours"
             Result: merge commit with main's tree unchanged
             Feature branch changes: ENTIRELY DISCARDED

-X ours    = EXTENDED OPTION to recursive/ort strategy
             Applies only when CONFLICT OCCURS line-by-line
             Non-conflicting changes from feature branch: STILL APPLIED
             Only conflicting lines: take our version
```

---

## 11. `git merge-base` — Finding the Common Ancestor

The merge base is the most important concept in understanding what a merge (or rebase, or diff) actually operates on. Let's go deep.

### Basic Usage

```bash
git merge-base main feature/login
# Output: e7a3f9b...  (the SHA of the common ancestor commit)

# Three-way merge base:
git merge-base --all main feature/login
# --all: show ALL merge bases (there can be multiple in criss-cross scenarios)
```

### What merge-base Computes

```
Algorithm (conceptual):

1. Walk back from HEAD of branch A: collect all ancestors into set A_ancestors
2. Walk back from HEAD of branch B: collect all ancestors into set B_ancestors
3. Intersection = all commits reachable from both branches
4. Among the intersection, find the MOST RECENT ones
   (commits in the intersection with no children also in the intersection)
5. Those are the merge bases

For simple cases: 1 merge base
For criss-cross: 2+ merge bases (recursive algorithm merges them first)
```

### Where merge-base is Used Implicitly

Every time you run any of these, Git internally calls merge-base first:

```bash
git diff main...feature/login    # diff from merge-base to feature/login
                                  # (3-dot notation = use merge-base as starting point)

git merge feature/login          # merge-base used for 3-way merge computation

git rebase main                  # commits SINCE merge-base are replayed

git log main..feature/login      # commits reachable from feature/login 
                                  # but NOT from main (since merge-base)

git cherry-pick A..C             # implicitly uses range, respects merge-base
```

### Hands-On Proof

```bash
# In our merge-demo repo:
git merge-base main feature/login
# Save this SHA

git log --oneline --graph --all
# Find the divergence point — it should match the merge-base SHA

# PROVE that git diff uses merge-base:
MERGE_BASE=$(git merge-base main feature/login)
git diff $MERGE_BASE feature/login    # This is exactly what git diff main...feature/login shows
git diff main...feature/login         # Same output
```

---

## 12. Choosing a Strategy for Your Team — Decision Framework

### The Decision Flowchart

```
START: What matters most for your team's main branch history?
       │
       ├─ "Ability to revert an entire feature as one atomic unit"
       │   → MERGE COMMIT (--no-ff)
       │   → You can: git revert -m 1 <merge-commit>
       │
       ├─ "Clean, linear, one-commit-per-PR history"
       │   → SQUASH AND MERGE
       │   → git log reads like a changelog
       │   → git bisect is fastest
       │   → Individual commit context lost after merge
       │
       └─ "Linear history AND individual commits preserved"
           → REBASE AND MERGE
           → git log shows all commits individually
           → But: SHAs change, can't reference original commit SHAs
           → Works best when feature branch commits are already clean
```

### Team Size and Velocity Matrix

```
Team Profile                                    Recommended Strategy
──────────────────────────────────────────────────────────────────────
Startup, 2-4 devs, 5-15 PRs/day                 Squash
  (want clean history, fast iteration)

Mid-size team, 10-20 devs, 3-8 PRs/day          Squash OR Merge Commit
  (depends on regulatory requirements)

Enterprise, audit requirements, 50+ devs         Merge Commit
  (need full traceability, can run git revert)

Open source project, external contributors       Merge Commit
  (author attribution matters, full history)

Library/framework, semantic versioning           Squash
  (changelog = git log, each PR = one entry)

Kernel/systems software, Gitflow                 Merge Commit
  (long release cycles, named branches matter)
```

### Changing Strategy Mid-Project

```
DANGER: Switching strategies on a live repo

Before switch:
  main history uses merge commits:
  A---B---M1---C---D---M2---E---M3
  (M1, M2, M3 are merge commits)

After switch to squash:
  main history uses squash commits:
  A---B---M1---C---D---M2---E---M3---S1---S2---S3
  
PROBLEM: git blame and git log --follow now see TWO different styles of history.
         git bisect works differently for the two eras.
         Future developers are confused by the inconsistency.

BEST PRACTICE:
  1. Pick a strategy when starting the project
  2. If you must switch, do it at a clean boundary (new major version, repo restructure)
  3. Document the change in CONTRIBUTING.md with the date and reason
  4. Update branch protection settings atomically with the team agreement
```

---

## 13. What Breaks at Scale — 5 Real Failure Modes

### Failure 1: The "Lost Revert" Problem with Squash

```
SCENARIO:
Day 1:  PR #100 squash-merged: "feat: add experimental payments"
        → Squash commit S100 on main

Day 3:  Payments is broken. Revert:
        git revert S100  → creates revert commit R100 on main

Day 10: Payments is fixed. Re-open PR #100:
        Original feature branch still points to old commits A/B/C
        git merge --squash feature/payments  → Git computes diff from merge-base
        
PROBLEM:
The revert commit R100 is now an ancestor of main.
When Git computes the merge base between feature/payments and main, 
it sees main includes R100 which "undoes" S100.
The squash merge will appear to have no diff (or a partial diff).

ACTUAL BEHAVIOR:
git merge --squash feature/payments
# Message: "Already up to date" OR a confusing partial merge

ROOT CAUSE:
Squash commits destroy the parent-chain relationship between 
the feature branch and main. When the revert+re-merge pattern is needed,
the loss of parent linkage causes Git to miscalculate what is "new."

FIX when re-opening a reverted squash PR:
1. Create a NEW branch from current main
2. Cherry-pick the feature commits:
   git cherry-pick A..C  (the original commits)
3. Open a new PR from the new branch
4. This gives Git a clean parent chain with no history confusion
```

### Failure 2: The `--first-parent` Break with Rebase and Merge

```
SCENARIO: Your team uses rebase-and-merge. You have tooling that uses
          git log --first-parent main to generate the changelog.

Before (merge commit strategy):
  git log --first-parent main --oneline
  8f3a1b9 Merge PR #50: Add payments
  7c3e8fa Merge PR #49: Fix auth
  Each line = one PR

After switching to rebase-and-merge:
  git log --first-parent main --oneline  
  9e8d7c6 C: address review feedback  ← individual commit from PR #50
  f4a5b6c B: add payment validation   ← individual commit from PR #50
  b1c2d3e A: add payment models       ← individual commit from PR #50
  7c3e8fa C': fix edge case in auth   ← individual commit from PR #49
  
PROBLEM: --first-parent now shows individual commits, not PR boundaries.
         Your changelog script breaks.
         You can no longer easily see "what PRs landed?"

PREVENTION: 
  If you need PR-level granularity in git log, use merge commits or squash.
  Rebase and merge destroys the PR-level grouping at the Git history level.
  (GitHub's PR #N reference in the squash commit message is your only marker)
```

### Failure 3: The Rebase Conflict Snowball

```
SCENARIO: Long-running feature branch, team uses rebase-and-merge.

Week 1:  You branch feature/big-auth from main at commit G
Week 1:  50 PRs merge into main (squash, so G+50 commits)
Week 3:  You try to rebase feature/big-auth onto main:
         git rebase main
         → 50 commits of main history to replay over
         → Multiple conflicts in each of your 20 commits
         → 20 × 5 conflicts = 100 conflict resolutions
         
ROOT CAUSE:
The longer you wait to rebase, the more drift accumulates.
Each of your commits must be verified against EACH of main's changes.
Time complexity: O(your_commits × main_changes)

PREVENTION:
  Rebase onto main DAILY (or at least weekly) for long-running branches.
  git fetch origin && git rebase origin/main
  Resolve conflicts while they are fresh and few.
  
MITIGATION if already far behind:
  Option A: rerere — auto-resolves conflicts you've seen before (Section 9)
  Option B: Create a new branch from main, squash your entire feature into one
            commit, apply it: git checkout -b feature/big-auth-v2 main
            git diff G feature/big-auth | git apply
            git add . && git commit -m "feat: big auth (combined)"
```

### Failure 4: The Squash Blame Blindspot

```
SCENARIO: Six months after a feature shipped, a bug is found in auth.js.
          The team uses squash-and-merge.

git blame src/auth/login.js
# Output: 
#   S100 (alice 2026-01-15) const token = req.headers.authorization;
#   S100 (alice 2026-01-15) if (!token) return res.status(401).send();
#   S100 (alice 2026-01-15) const user = await getUserFromToken(token);
#   S100 (alice 2026-01-15) // TODO: validate token expiry
#   ^S100 (alice 2026-01-15) ...

git show S100
# Shows: 247-line diff of "feat: add OAuth login (#100)"
# You know WHAT changed but not WHY each line was written.
# The "TODO: validate token expiry" was in commit B of the original PR,
# where the commit message was "B: temporary, will add expiry check in C"
# But commit C never happened — the developer forgot.
# With squash, that context is GONE.

IMPACT:
  Debugging becomes harder.
  You see the what (the squash commit) but not the incremental why.

MITIGATION:
  Write excellent commit messages in your squash commit body.
  Include the individual commits' summaries (GitHub does this automatically).
  For security-critical code, consider merge commits to preserve full history.
```

### Failure 5: The Octopus Merge Conflict Bomb

```
SCENARIO: Release manager tries to octopus-merge 5 features for a release.
          3 of the 5 features modified the same file.

git merge feature/a feature/b feature/c feature/d feature/e
# Error: "Merge requires file-level merging"
# Git aborts. The octopus strategy doesn't do conflict resolution.

FIX:
  1. Identify which features conflict:
     git merge-tree $(git merge-base feature/a feature/b) feature/a feature/b
     # Shows what would conflict between feature/a and feature/b
  
  2. Merge the conflicting branches pairwise first:
     git merge feature/a  # resolve conflict
     git merge feature/b  # resolve conflict (now with feature/a integrated)
  
  3. Then octopus the non-conflicting remainder:
     git merge feature/c feature/d feature/e  # these don't conflict
```

---

## 14. Mistakes → Root Cause → Fix → Prevention

### Mistake 1: Using `git merge` Without `--no-ff` in Gitflow

**Symptom**: `git log --graph` shows a straight line with no branch structure even though your team uses Gitflow.

**Root cause**: Git fast-forwarded the merge (no diverged commits on main) because you didn't pass `--no-ff`.

**Fix**:
```bash
# Already merged without --no-ff? The commits are there but the merge commit isn't.
# If you haven't pushed:
git reset --hard HEAD~3  # undo (assumes 3 commits from feature branch)
git merge --no-ff feature/login -m "Merge feature/login"

# If already pushed (dangerous — coordinate with team):
git revert HEAD~3..HEAD  # revert the feature commits
# Then re-merge properly... or just accept linear history for this PR
```

**Prevention**:
```bash
# Set no-ff as default for a repo:
git config merge.ff no

# This makes --no-ff the default for ALL merges in this repo
# Equivalent to always passing --no-ff
```

### Mistake 2: Accidentally Squashing a Merge Commit

**Symptom**: `git merge --squash` on a branch that itself has merge commits in it produces a confusing combined diff.

**Root cause**: `--squash` takes the diff from merge-base to HEAD. If the feature branch has its own merge commits (e.g., someone merged `main` into the feature branch to stay up to date), those merges are included in the squash diff too — making the squash commit larger than expected.

**Fix**:
```bash
# Clean up before squashing:
# Instead of merge-squashing a branch with merge commits in it:

# Option A: Rebase first (linearize), then squash
git checkout feature/login
git rebase main                    # linearize (resolves merge commits)
git checkout main
git merge --squash feature/login   # now squash a clean linear branch

# Option B: Use interactive rebase to squash before PR
git rebase -i main                 # squash all commits into one interactively
git checkout main
git merge --ff-only feature/login  # fast-forward since it's clean
```

**Prevention**: Keep feature branches rebased onto main. Do not use `git merge main` to update a feature branch (use `git rebase main` instead).

### Mistake 3: Using `-s ours` When You Meant `-X ours`

**Symptom**: After running `git merge -s ours feature/important`, none of the feature's changes are in the codebase. You thought it would resolve conflicts in your favor.

**Root cause**: `-s ours` is the merge **strategy** (algorithm) — it discards everything from the other branch. `-X ours` is a strategy **option** — it only overrides conflicting lines.

```bash
# WRONG (discards feature branch entirely):
git merge -s ours feature/login

# CORRECT (auto-resolves conflicts with our version, keeps non-conflicting changes):
git merge -X ours feature/login
```

**Fix**:
```bash
git reset --hard HEAD~1  # undo the wrong merge
git merge -X ours feature/login  # correct merge
```

**Prevention**: Always double-check: lowercase `-s` = strategy (algorithm), uppercase-adjacent `-X` = extended option. Mnemonic: `-s` for **s**trategy, `-X` for e**X**tended option.

### Mistake 4: Forgetting to Delete Feature Branch After Squash Merge

**Symptom**: Feature branch is "merged" but still appears in `git branch -a`. Six months later, someone accidentally checks it out and continues working on it from the old commit.

**Root cause**: Squash merge does NOT move or delete the feature branch pointer. `feature/login` still points to `C` (original commits). Git has no way to know those commits are "already merged" since they're not in main's ancestry (unlike merge commit or rebase).

```bash
# PROVE the problem:
git log --oneline main..feature/login
# After squash merge, this shows the original commits A, B, C
# Git thinks they're NOT in main (because the squash commit S100 is not their parent)

# After merge commit or rebase, same command shows nothing
# (commits are reachable from main)
```

**Fix and Prevention**:
```bash
# After squash merge, ALWAYS delete the branch:
git branch -d feature/login           # local (Git warns if "not merged" — ignore: use -D)
git push origin --delete feature/login # remote

# GitHub can auto-delete branches after merge — enable in repo settings:
# Settings → General → "Automatically delete head branches"
```

---

## 15. Byte-Level Internals — Merge Commit vs Squash Commit Object

### Merge Commit Object (Binary Layout)

From Topic 02, all Git objects are: `<type> <size>\0<content>` zlib-compressed.

```
RAW BYTES of a merge commit (before zlib):

commit 234\0
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579\n
parent 7c3e8fa1b13a6c2fa3d0fa1aff3f3d4b72e1234\n
parent a3f2c1d8e5b67890abcdef1234567890abcdef12\n
author Alice Smith <alice@example.com> 1748433600 +0000\n
committer Alice Smith <alice@example.com> 1748433600 +0000\n
\n
Merge branch 'feature/login' into main\n
\n
Reviewed by: @bob\n

│                    │
│                    └── \n = literal newline byte (0x0A)
└── \0 = null byte separator between header and content

KEY DIFFERENCE from regular commit:
  TWO "parent" lines vs ONE
  First parent = main's HEAD before merge
  Second parent = feature branch tip
  
  Size includes BOTH parent lines in the byte count.
```

### Squash Commit Object

```
RAW BYTES of a squash commit:

commit 198\0
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579\n
parent 7c3e8fa1b13a6c2fa3d0fa1aff3f3d4b72e1234\n
author Alice Smith <alice@example.com> 1748433600 +0000\n
committer Alice Smith <alice@example.com> 1748433600 +0000\n
\n
feat(auth): add login endpoint (#42)\n

KEY DIFFERENCES from merge commit:
  ONE "parent" line (not two)
  SMALLER size (198 vs 234 bytes)
  tree SHA might be SAME as merge commit would produce
       (same final file content → same tree SHA)
  
  The tree SHA is identical to what merge commit would compute,
  because both represent the same final merged state of the files.
  
  Only the parent linkage differs.
```

### Proof: Same Tree, Different Parent

```bash
# In our merge-demo repo, compute what BOTH strategies would produce:

# Strategy 1: Merge commit
git checkout main
git merge --no-ff feature/login -m "merge" --no-edit
MERGE_TREE=$(git cat-file -p HEAD | grep "^tree" | awk '{print $2}')
echo "Merge commit tree: $MERGE_TREE"
git reset --hard HEAD~1  # undo

# Strategy 2: Squash
git merge --squash feature/login
git commit -m "squash"
SQUASH_TREE=$(git cat-file -p HEAD | grep "^tree" | awk '{print $2}')
echo "Squash commit tree: $SQUASH_TREE"
git reset --hard HEAD~1  # undo

# Compare:
echo "Same tree? $([ '$MERGE_TREE' = '$SQUASH_TREE' ] && echo YES || echo NO)"
# PROVE: The tree SHA is IDENTICAL — same file content in both cases
# ONLY the parent list and commit metadata differ
```

---

## 16. Mental Model Checkpoint

Test yourself. Answer from memory.

**1.** After `git merge --squash feature/login`, you run `git log main..feature/login`. Do you see commits? Why or why not?

**2.** What is the difference between `-s ours` and `-X ours`? Give a scenario where each is appropriate.

**3.** A merge commit has parents `G` and `C`. `git log --first-parent main` shows only `G` before the merge commit. Why does `C` not appear in `--first-parent` output?

**4.** After "Rebase and merge" on GitHub, a colleague tries to `git cherry-pick` the original commit SHA from `feature/login`. It doesn't appear in `main`'s history. Why?

**5.** What file does `git merge --squash` write to `.git/` that a regular `git merge` does not? What happens to this file after `git commit`?

**6.** You run `git merge -s recursive -X patience feature/login`. What does each part (`-s`, `-X`, `patience`) mean?

**7.** Your team switches from merge commits to squash on an existing repo. A developer runs `git log --follow -- src/api.js`. What problem do they encounter?

**8.** What does `rerere` stand for? What two files does it write to `.git/rr-cache/<sha>/`?

**9.** An octopus merge of 4 branches creates a commit with how many parents? List them.

**10.** After a squash merge, you need to revert the feature. What command do you run? Why is reverting a squash easier than reverting a merge commit?

<details>
<summary>Answers</summary>

**1.** Yes, you still see the original commits A, B, C. Squash merge does NOT move or include the feature branch's commits in main's ancestry. The feature branch pointer is untouched. `git log main..feature/login` shows commits reachable from `feature/login` but not from `main` — and A, B, C are still not ancestors of `main` (only S, the squash commit, is — but S is not an ancestor of A/B/C). This is why you must manually delete the branch after squash.

**2.** `-s ours` is the entire merge STRATEGY: the result of the merge is identical to your current branch's tree, all changes from the other branch are discarded, and only a merge commit recording the event is created. `-X ours` is an extended OPTION: the merge proceeds normally (applying non-conflicting changes), but where a conflict exists line-by-line, your version wins automatically. Use `-s ours` to retire/archive a branch. Use `-X ours` when you want all non-conflicting changes but want auto-resolution for conflicting lines.

**3.** The first parent of a merge commit is the branch you were ON when you ran `git merge`. Since you were on `main`, `main`'s previous HEAD (G) is the first parent. The feature branch tip (C) is the second parent. `--first-parent` only follows first-parent pointers, so it walks G → F → E → D. C and the feature commits are only reachable via the second-parent link.

**4.** "Rebase and merge" creates NEW commit objects A', B', C' with new SHAs. The original A, B, C SHAs are not in `main`'s ancestry chain. They exist in the repo's objects as dangling objects (until garbage collected) but are not reachable from any ref. To cherry-pick, you'd need to cherry-pick A'/B'/C' (the new SHAs), or find the originals in the reflog.

**5.** `git merge --squash` writes `.git/SQUASH_HEAD` containing the SHA of the feature branch tip. This is used by Git to reference what was squashed when building the commit message. It is deleted when `git commit` is run (or `git reset`).

**6.** `-s recursive` = use the "recursive" merge algorithm (instead of ort/default). `-X patience` = pass the "patience" diff algorithm as an extended option to the recursive algorithm, which uses a more careful diff computation to reduce false conflicts. Together: "merge using the recursive algorithm, and within that algorithm use the patience diff algorithm."

**7.** For commits before the strategy switch, `--follow` traces through merge commits and finds the full history including individual commits on feature branches. For commits after the switch, each squash commit is atomic and contains no parent link to the original feature commits. `--follow` loses track at the first squash commit, unable to trace the file's history before the squash. The history appears to "start" at the squash commit.

**8.** `rerere` = **Re**use **Re**corded **Re**solution. It writes two files: `preimage` (the conflicted state — both sides of the conflict as they appeared) and `postimage` (your resolution after you manually resolved and staged it). On future encounters of the same conflict pattern, it reads the postimage and applies it automatically.

**9.** An octopus merge of 4 branches (run from main) creates a commit with 5 parents: main's HEAD + the tips of the 4 branches being merged. Generally: an octopus merge of N branches creates a commit with N+1 parents (current branch HEAD + N branch tips).

**10.** `git revert <squash-commit-SHA>`. Reverting a squash is easier because it has exactly ONE parent — `git revert` on a single-parent commit is straightforward. Reverting a merge commit requires `-m 1` or `-m 2` to specify which parent represents "mainline" (i.e., which side to revert to). `git revert -m 1 <merge-commit>` reverts to main's previous state, which is non-obvious and error-prone.

</details>

---

## 17. Connect Everything Backwards

| This Topic's Concept | Connects To |
|---------------------|-------------|
| **Merge commit has two parents** | Topic 02 — Git commit object format. A merge commit is just a commit object with two `parent` lines instead of one. The SHA is still computed the same way. |
| **Fast-forward merge** | Topic 09 — The simplest merge: no merge commit, just a pointer move. `--no-ff` forces a merge commit even when fast-forward is possible. |
| **Merge base computation** | Topic 09 — The common ancestor (3-way merge) is exactly what `git merge-base` finds. Topic 16 showed the same concept for PR diffs. |
| **Rebase and merge creates new SHAs** | Topic 11 — Rebase always creates new commit objects. The SHA changes because parent changed. "Rebase and merge" on GitHub is just a server-side rebase followed by fast-forward. |
| **SQUASH_HEAD file in `.git/`** | Topic 01 — `.git/` contains special state files (MERGE_HEAD, CHERRY_PICK_HEAD, SQUASH_HEAD). These are "in-progress operation" markers that Git uses across command invocations. |
| **`git log main..feature/login`** | Topic 03 — The DAG: this notation means "commits reachable from feature/login but not from main." After squash, feature commits are NOT reachable from main (squash commit S is a new object, not their child). |
| **`rerere` records to `.git/rr-cache/`** | Topic 01 — `.git/` is where ALL Git state lives, including conflict resolution cache. rr-cache is just another directory in `.git/`. |
| **`-s ours` for Gitflow branch maintenance** | Topic 15 — In Gitflow, after a hotfix merges to main, you need `develop` to see main's history. `-s ours` creates the merge record without duplicate conflicts. |
| **Choosing strategy based on bisect needs** | Topic 21 (upcoming) — `git bisect` works best when every commit on main is a shippable unit. Squash makes every commit a whole PR — ideal for bisect. Rebase keeps individual commits — harder for bisect if "wip" commits exist. |

---

## 18. Quick Reference Card

### Core Merge Commands

| Command | What it does |
|---------|-------------|
| `git merge feature` | Merge feature into current branch (fast-forward if possible) |
| `git merge --no-ff feature` | Force merge commit even if fast-forward possible |
| `git merge --ff-only feature` | Fast-forward only, FAIL if not possible |
| `git merge --squash feature` | Apply changes without commit (stages only) |
| `git merge --squash feature && git commit` | Full squash merge |
| `git merge --abort` | Abort an in-progress conflicted merge |
| `git merge -s ours feature` | Merge commit discarding feature's changes |
| `git merge -X ours feature` | Merge, auto-resolving conflicts with our version |
| `git merge -X theirs feature` | Merge, auto-resolving conflicts with their version |
| `git merge -X patience feature` | Merge using patience diff algorithm |
| `git merge branch-a branch-b branch-c` | Octopus merge (no conflicts allowed) |

### Merge-Base Commands

| Command | What it does |
|---------|-------------|
| `git merge-base main feature` | Show merge base SHA |
| `git merge-base --all main feature` | Show ALL merge bases (criss-cross) |
| `git diff main...feature` | Diff from merge-base to feature tip |
| `git log main..feature` | Commits in feature not in main |

### rerere Commands

| Command | What it does |
|---------|-------------|
| `git config rerere.enabled true` | Enable rerere globally |
| `git config rerere.autoupdate true` | Also auto-stage rerere resolutions |
| `git rerere` | Apply any known resolutions to current conflicts |
| `git rerere diff` | Show rerere's current pending resolution |
| `git rerere forget <path>` | Forget stored resolution for a file |
| `ls .git/rr-cache/` | See all stored resolutions |

### Algorithm Flags

| Flag | Algorithm | Notes |
|------|-----------|-------|
| `-s resolve` | Oldest, simple | Fails on criss-cross |
| `-s recursive` | Default pre-2.33 | Handles criss-cross |
| `-s ort` | Default since 2.33 | O(n) time, same results as recursive |
| `-s octopus` | Multi-branch | No conflict resolution |
| `-s ours` | Discard other | Keeps your tree entirely |
| `-X patience` | Diff algorithm | Better for large refactors |
| `-X histogram` | Diff algorithm | Best general-purpose (default in modern Git) |
| `-X ignore-space-change` | Whitespace | Ignore whitespace changes |

### Strategy Comparison

| | Merge Commit | Squash | Rebase+Merge |
|--|--|--|--|
| History shape | Non-linear | Linear | Linear |
| Original SHAs preserved | ✅ | ❌ | ❌ (new SHAs) |
| Feature commits on main | ✅ All | ❌ None | ✅ All (rewritten) |
| PR reversible as unit | ✅ `revert -m 1` | ✅ `revert SHA` | ❌ Multiple reverts |
| `--first-parent` shows PR | ✅ Merge commit | ✅ Squash commit | ❌ Individual commits |
| git bisect ease | Hard | Easy | Medium |
| git blame detail | High | Low | High |
| New objects created | 1 | 1 | N (one per commit) |

---

## 19. When Would I Use This At Work?

### Daily Operations

**Scenario: You're the release engineer for a SaaS platform using GitHub Flow (squash merges). A critical bug is found in production that was introduced 3 weeks ago across 3 different PRs.**

```bash
# Use git bisect (Topic 21) to find which squash commit introduced the bug.
# Each squash commit = one PR = one "unit" to test.
# Bisect on squash-commit history is MUCH cleaner than bisect on a merge-commit history
# because every commit is independently deployable.

# Once found, say squash commit S147:
git revert S147
git push origin main
# Clean single-line revert. Easy.

# Compare: if using merge commits, you'd need:
git revert -m 1 M147  # need the -m 1 flag, which is confusing
# And need to know that M147 is the merge commit for the PR, not a feature commit
```

**Scenario: Your monorepo has a "platform" team and an "application" team. Platform makes a major refactor touching 500 files. Application has 10 PRs in flight that all conflict with the platform changes.**

```bash
# Enable rerere for the entire team:
git config --global rerere.enabled true
git config --global rerere.autoupdate true

# Platform merges first.
# Each application PR, when rebasing:
git fetch origin && git rebase origin/main
# First PR to rebase: manual conflict resolution (rerere records it)
# Subsequent PRs: rerere auto-applies the same resolution
# Teams with rerere enabled resolve conflicts once, not N times
```

### Architecture Decisions

**Scenario: You're the tech lead starting a new microservice. Choosing merge strategy.**

```
Decision criteria for your specific context:

1. "Will we need to audit exactly which commit introduced a change?"
   → YES: Merge commit (regulatory, security-sensitive)
   → NO: Continue to #2

2. "Does our changelog generation read from git log?"
   → YES: Squash (one commit = one PR = one changelog entry)
   → NO: Continue to #3

3. "Do individual commits within a PR have meaningful standalone context?"
   → YES: Rebase and merge (preserve individual commits)
   → NO: Squash (individual commits are "wip" noise)

4. "Do we need to be able to revert an entire PR as one unit easily?"
   → YES: Squash or Merge commit (both easy to revert)
   → NO: Any strategy works

TYPICAL RESULT for a new microservice at a startup:
  → Squash and merge
  → Reason: clean history, easy bisect, easy revert, changelog = git log
  → git log main reads: "feat: X (#5)", "fix: Y (#4)", "feat: Z (#3)"
```

**Scenario: You inherit a codebase that uses all three strategies inconsistently. What do you do?**

```bash
# Step 1: Audit the current state
git log --oneline --graph main | head -50
# Look for: merge commits (diamond), squash commits (linear + #PR), 
#            rebased commits (linear, no #PR reference)

# Step 2: Decide the going-forward strategy (team decision, document in CONTRIBUTING.md)

# Step 3: Update branch protection:
# GitHub Settings → Branches → main → Edit rule:
#   □ Allow merge commits  ← uncheck if switching to squash
#   ✅ Allow squash merging ← check this
#   □ Allow rebase merging ← optional

# Step 4: Update local developer configs:
git config merge.ff only  # or "no" for merge-commit teams

# Step 5: Document the cutover date in CONTRIBUTING.md:
# "As of 2026-06-01, this repo uses squash-and-merge for all PRs.
#  History before this date uses merge commits — use git log --graph
#  to see the pre-2026 branch structure."
```

---

*End of Topic 17 — Merge Strategies at Scale*

**Next**: Topic 18 — Resolving Conflicts Like a Senior (the 3-way merge algorithm, ours vs theirs, merge tools, conflict markers decoded, complex conflict scenarios)
