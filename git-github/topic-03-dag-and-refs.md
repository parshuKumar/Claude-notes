# Topic 03 — The DAG & Refs
### How History is Built, Stored, and Navigated

> **Phase 1 — The Physical Foundation**
> Prerequisite: [Topic 01 — The .git/ Directory](./topic-01-the-git-directory.md) | [Topic 02 — Git Objects](./topic-02-git-objects-deep-dive.md)
> Next: [Topic 04 — Git Setup & Configuration](./topic-04-git-setup-configuration.md)

---

## Connection to Previous Topics

In Topic 01 you learned that `.git/refs/` holds plain text pointer files — one per branch, one per tag.
In Topic 02 you learned that commit objects embed their parent's SHA directly in their content.

Now we put these two facts together:

- The **refs** tell you WHERE to start in the history
- The **parent pointers inside commits** tell you WHERE to go next
- Together they form the **DAG** — the complete history of your project

This topic answers: how does `git log` know what to show? How does Git know which commits
belong to which branch? What exactly IS a branch, a remote-tracking ref, and HEAD — and
how do all three interact?

---

## 1. ELI5 — The Analogy

Imagine a city where every building has a sign on the front that says:
**"This building was constructed right after building #A4F2."**

Every single building in the city points to its predecessor. If you stand at the
newest building and follow the signs, you can walk backward through the entire
construction history of the city, all the way to the very first building.

Now imagine you have a **map** (the refs) with pins stuck in it:
- A red pin labeled "main" stuck on the newest building
- A blue pin labeled "feature/auth" stuck on a different building
- A green pin labeled "v1.0" stuck on an older building

The **pins are the refs.** The **buildings are commits.** The **signs on each building
are the parent pointers.** Together, they form the DAG.

---

## 2. What is a DAG?

**DAG = Directed Acyclic Graph**

Let's break that down:

```
DIRECTED:   Edges (pointers) go in one direction only.
            Every commit points BACKWARD to its parent.
            You can walk from child → parent, never parent → child directly.

ACYCLIC:    No cycles. You can never follow parent pointers and end up
            back at a commit you already visited.
            (Mathematically impossible because a commit's SHA depends on
             its parent's SHA — a cycle would require a commit's SHA to
             depend on itself.)

GRAPH:      A set of nodes (commits) connected by edges (parent pointers).
```

```
VALID DAG (what Git looks like):
  C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5
                  \
                   C6 ◄── C7

IMPOSSIBLE (cycle — can never happen in Git):
  C1 ◄── C2 ◄── C3
   └──────────────┘  ← C3's SHA would have to include C1's SHA,
                       but C1's SHA includes C3's SHA — impossible
```

---

## 3. The Physical Structure — Refs and the DAG Together

### 3.1 What refs/ Looks Like After Real Work

Let's say you've been working on a project for a while. Here's what `.git/` looks like:

```
.git/
├── HEAD                         → "ref: refs/heads/main"
│
├── refs/
│   ├── heads/
│   │   ├── main                 → "c7f3a9b2..." (latest commit on main)
│   │   ├── develop              → "a1e4d8c5..." (latest commit on develop)
│   │   └── feature/
│   │       ├── auth             → "9b2f7e1d..." (latest commit on feature/auth)
│   │       └── payments         → "3d8c5a4f..." (latest commit on feature/payments)
│   │
│   ├── tags/
│   │   ├── v1.0.0               → "7e2b9c4a..." (commit or tag object)
│   │   └── v1.1.0               → "4f8d1e3c..."
│   │
│   └── remotes/
│       └── origin/
│           ├── HEAD             → "ref: refs/remotes/origin/main"
│           ├── main             → "c7f3a9b2..." (last known remote main)
│           └── develop          → "b9f3e2a7..." (last known remote develop)
│
└── objects/
    ├── (all commit, tree, blob objects)
    └── ...
```

Every file under `refs/` contains exactly ONE thing: a 40-character SHA.

```bash
# Verify this yourself
cat .git/refs/heads/main
# Output: c7f3a9b2e4d1f8a5b3c7e2f9a4d6b8c1e3f7a2d5

wc -c .git/refs/heads/main
# Output: 41  (40 hex chars + 1 newline = 41 bytes)
```

### 3.2 The Object Graph Those Refs Point Into

```
.git/refs/heads/main ──────────────────────────────────► COMMIT c7f3a9b2
                                                          ├── tree ...
.git/refs/heads/feature/auth ──────────────────────────► COMMIT 9b2f7e1d
                                                          │   ├── tree ...
                                                          │   └── parent ──► COMMIT a1e4d8c5
                                                          │                  │   ├── tree ...
.git/refs/heads/develop ───────────────────────────────► COMMIT a1e4d8c5   │   └── parent ──► COMMIT 7e2b9c4a
                                                                             │                  │   ├── tree ...
.git/refs/tags/v1.0.0 ─────────────────────────────────────────────────────┘                  └── parent (none - initial)

HEAD ──► "ref: refs/heads/main" ──► COMMIT c7f3a9b2
```

**Key insight:** Multiple refs can point to the SAME commit. That's perfectly fine.
When you first create a branch from main, both refs point to the same commit.

---

## 4. HEAD — The Full Story

### 4.1 What HEAD Is

HEAD is a pointer to a pointer. Normally:

```
HEAD → refs/heads/main → commit SHA
```

This is called a **symbolic ref** — HEAD stores the NAME of a branch, not the SHA directly.

```bash
cat .git/HEAD
# Normal output: ref: refs/heads/main
# (this is a symbolic ref)

git symbolic-ref HEAD
# Output: refs/heads/main
```

### 4.2 Detached HEAD — What It Is, Why It Happens, How to Get Out

When HEAD points DIRECTLY to a commit SHA (not a branch name):

```bash
cat .git/HEAD
# Detached HEAD output: c7f3a9b2e4d1f8a5b3c7e2f9a4d6b8c1e3f7a2d5
# (a raw SHA, not "ref: refs/heads/something")
```

**How you get into detached HEAD:**
```bash
git checkout c7f3a9b2          # checkout a specific commit
git checkout v1.0.0            # checkout a tag (tags don't move)
git checkout origin/main       # checkout a remote-tracking ref
git bisect start               # bisect detaches HEAD to test commits
```

**What's dangerous about it:**

```
DETACHED HEAD STATE:

HEAD ──► c7f3a9b2 (a specific commit, no branch)
         │
         └── if you make NEW commits here...

HEAD ──► NEW_COMMIT_1 ──► NEW_COMMIT_2 ──► c7f3a9b2
         │
         └── these new commits have NO BRANCH pointing to them!
             If you switch to another branch, HEAD moves away,
             and your new commits become ORPHANED (no ref points to them).
             They'll be garbage-collected eventually.
```

**How to get out safely:**

```bash
# Option 1: Create a branch RIGHT NOW to save your work
git checkout -b my-recovery-branch
# Now HEAD → refs/heads/my-recovery-branch → your new commits ✅

# Option 2: If you made no commits and just want to go back
git checkout main
# HEAD reattaches to main ✅

# Option 3: You made commits and want to merge them into main
git checkout -b temp-branch
git checkout main
git merge temp-branch
```

### 4.3 HEAD Navigation Syntax (Full Reference)

These all resolve from HEAD's current position:

```
Syntax     │ Meaning
───────────┼────────────────────────────────────────────────────
HEAD       │ Current commit
HEAD^      │ Parent of HEAD (1 generation back)
HEAD^^     │ Grandparent (2 generations back)  
HEAD~1     │ Same as HEAD^
HEAD~2     │ Same as HEAD^^
HEAD~N     │ N generations back (linear ancestor)
HEAD^1     │ First parent (relevant for merge commits)
HEAD^2     │ Second parent (the merged-in branch)
HEAD^{tree}│ The root tree object of HEAD's commit
HEAD:path  │ The blob at 'path' in HEAD's tree
@          │ Shorthand for HEAD in recent Git versions
@{1}       │ Previous position of HEAD (from reflog)
@{-1}      │ The branch you were on before the last checkout
```

```bash
# Prove the parent navigation:
git log --oneline -5
# Output:
# c7f3a9b (HEAD -> main) feat: add payment processing
# a1e4d8c feat: add user auth
# 7e2b9c4 feat: initial API structure
# 3d8c5a4 chore: project setup
# 9b2f7e1 init: first commit

git rev-parse HEAD~2
# Output: 7e2b9c4... (two commits back)

git cat-file -p HEAD~2
# Shows the commit "feat: initial API structure"
```

---

## 5. The DAG in Detail — All the Shapes History Can Take

### 5.1 Linear History (No Branches Yet)

```
C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5
                              │
                             HEAD, main

Objects in .git/objects/:
  5 commit objects
  N tree objects (one root tree per commit, subtrees where unchanged = reused)
  M blob objects (one per unique file version)

.git/refs/heads/main = SHA of C5
.git/HEAD = "ref: refs/heads/main"
```

### 5.2 Diverged History (Feature Branch Created)

```bash
git checkout -b feature/auth    # at C5
# ... make 2 commits on feature/auth
git checkout main
# ... make 1 commit on main
```

```
C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5 ◄── C6
                                \
                                 C7 ◄── C8

.git/refs/heads/main          = SHA of C6
.git/refs/heads/feature/auth  = SHA of C8
.git/HEAD                     = "ref: refs/heads/main"  (we checked out main)

C5 = the "common ancestor" (merge base) of main and feature/auth
C6, C7, C8 are "diverged" — they exist in parallel
```

**How git knows which commits belong to which branch:**
Git does NOT tag commits with branch names. Instead, a branch "owns" all commits
reachable by following parent pointers backward from the branch tip.

```
main OWNS:    C6 → C5 → C4 → C3 → C2 → C1
feature OWNS: C8 → C7 → C5 → C4 → C3 → C2 → C1

C5, C4, C3, C2, C1 are reachable from BOTH. They are "shared history."
```

### 5.3 After a Fast-Forward Merge

```bash
git checkout main
git merge feature/auth    # if main hasn't moved since C5, this is a fast-forward
```

```
BEFORE:
  main ──────────────────────────────► C6
  feature/auth ──────────────────────► C8 ◄── C7 ◄── C5 (main's old tip)

  Wait — this can't fast-forward! C6 exists.
  Fast-forward only works if the target branch hasn't diverged.

CORRECT example for fast-forward:
  If main was still at C5 (no C6):
  C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5 ◄── C7 ◄── C8
                                │
                               main (still here)
                                            │
                                      feature/auth

AFTER fast-forward merge:
  C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5 ◄── C7 ◄── C8
                                                │
                                            main, feature/auth
                                            (both point to C8)

  .git/refs/heads/main  = SHA of C8  ← just moved the pointer!
  NO new commit object created.
  This is called "fast-forward" because the branch pointer just "fast-forwarded"
  along the existing line of commits.
```

### 5.4 After a 3-Way Merge (Creates a Merge Commit)

```bash
# main is at C6, feature/auth is at C8 — they have DIVERGED
git checkout main
git merge feature/auth
```

```
BEFORE:
  C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5 ◄── C6       ← main
                                  \
                                   C7 ◄── C8    ← feature/auth

AFTER (merge commit M9 created):
  C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5 ◄── C6 ◄──── M9
                                  \             /
                                   C7 ◄── C8 ──┘

  M9 = merge commit, has TWO parents: C6 and C8

  .git/refs/heads/main = SHA of M9
  M9's content:
    tree   <merged tree SHA>
    parent <SHA of C6>    ← first parent (was HEAD)
    parent <SHA of C8>    ← second parent (merged in)
    author ...
    committer ...

    Merge branch 'feature/auth'
```

### 5.5 After a Rebase (Rewrites the DAG)

```bash
# feature/auth at C8, main at C6
git checkout feature/auth
git rebase main
```

```
BEFORE:
  C1 ◄─ C2 ◄─ C3 ◄─ C4 ◄─ C5 ◄─ C6       ← main
                               \
                                C7 ◄─ C8  ← feature/auth

AFTER REBASE:
  C1 ◄─ C2 ◄─ C3 ◄─ C4 ◄─ C5 ◄─ C6 ◄─ C7' ◄─ C8'  ← feature/auth
                                               │       │
                               NEW objects!    │       │
                               C7' ≠ C7       │       │
                               C8' ≠ C8       │       │
                               (new SHAs)     ↑       ↑
                               Old C7, C8 still exist in objects/
                               but no ref points to them (orphaned)

  .git/refs/heads/feature/auth = SHA of C8'
  main: UNCHANGED, still at C6
```

Why are C7' and C8' different objects from C7 and C8?
Their `parent` field now contains a different SHA (C6 instead of C5).
Any change to a commit's content = new SHA. Topic 11 covers this fully.

---

## 6. Refs in Detail — Every Type Explained

### 6.1 Local Branch Refs — `.git/refs/heads/`

```
.git/refs/heads/<branch-name>
```

- One file per local branch
- Contains the SHA of the tip commit of that branch
- Moves forward automatically when you commit on that branch
- Created with `git branch` or `git checkout -b`
- Deleted with `git branch -d`

```bash
# List all local branch refs
ls .git/refs/heads/
# Or with subdirectories (like feature/auth which has a slash in the name):
find .git/refs/heads -type f

# Read one
cat .git/refs/heads/main
# Output: c7f3a9b2e4d1f8a5b3c7e2f9a4d6b8c1e3f7a2d5
```

⚠️ **Branch names with slashes create actual subdirectories:**
```
Branch name: "feature/auth"
Physical location: .git/refs/heads/feature/auth
                   (a file named "auth" inside a directory named "feature")
```

### 6.2 Remote-Tracking Refs — `.git/refs/remotes/`

```
.git/refs/remotes/<remote-name>/<branch-name>
```

- Created/updated when you run `git fetch` or `git pull`
- They are your LOCAL COPY of the last known state of remote branches
- **You do not commit to these directly** — they update only via fetch
- Reading them = "what was origin/main last time I fetched?"

```bash
# After fetching from origin
cat .git/refs/remotes/origin/main
# Output: c7f3a9b2...  (what origin's main pointed to last time you fetched)

# This might be STALE — origin might have moved on since
# Use git fetch to update it
```

```
Timeline:
  10am: You clone. origin/main = C5. main = C5.
  11am: Colleague pushes C6, C7 to origin.
  11:30am: You are still working. Your origin/main is STILL C5 (stale).
  12pm: You run git fetch. Now origin/main = C7. Your main still = C5.
  12:05pm: You run git merge origin/main. Now your main = C7.
```

### 6.3 Tag Refs — `.git/refs/tags/`

```
.git/refs/tags/<tag-name>
```

- Unlike branches, tags **never move** (they're permanent markers)
- Lightweight tag → file contains a commit SHA directly
- Annotated tag → file contains a tag object SHA (which then points to commit)

```bash
cat .git/refs/tags/v1.0.0
# For lightweight tag:  c7f3a9b2...  (commit SHA)
# For annotated tag:    5f3e8a9b...  (tag object SHA)

# Distinguish them:
git cat-file -t $(cat .git/refs/tags/v1.0.0)
# Output: "commit" (lightweight) or "tag" (annotated)
```

### 6.4 packed-refs — When Refs Get Packed

With thousands of branches/tags, having thousands of tiny files is inefficient.
Git compacts them into a single file:

```bash
cat .git/packed-refs
# Output:
# # pack-refs with: peeled fully-peeled sorted
# c7f3a9b2e4d1f8a5b3c7e2f9a4d6b8c1e3f7a2d5 refs/heads/main
# a1e4d8c5f7b2e9d3a6c8f1b4d7e2a5c9f3b6d8e1 refs/heads/develop
# 5f3e8a9b2c4d7e1f3a8b6c9d2e5f7a4b1c3d8e6f refs/tags/v1.0.0
# ^c7f3a9b2e4d1f8a5b3c7e2f9a4d6b8c1e3f7a2d5  ← "peeled" = the commit an annotated tag points to
```

**Rules when both packed-refs and refs/ exist:**
- If a ref exists as BOTH a file in `refs/` AND a line in `packed-refs`, the FILE wins
- `git pack-refs --all` packs all refs into `packed-refs` and removes the loose files
- Creating or updating a branch writes a new loose file in `refs/` (overrides packed version)

---

## 7. How `git log` Traverses the DAG

### 7.1 The Algorithm

`git log` is a graph traversal. Here's exactly what it does:

```
ALGORITHM for git log:

1. Start with the ref(s) given (or HEAD if none given)
2. Add their SHAs to a priority queue (sorted by committer date, newest first)
3. Pop the next commit from the queue:
   a. Display this commit (if it passes filters)
   b. Read the commit's parent SHA(s)
   c. Push parent(s) onto the queue (if not already seen)
4. Repeat step 3 until queue is empty or --max-count reached
```

```bash
# Basic traversal from HEAD
git log

# Traversal from a specific ref
git log feature/auth

# Traversal from MULTIPLE starting points (union of reachable commits)
git log main feature/auth

# Show commits reachable from feature but NOT from main
# (i.e., commits unique to feature/auth)
git log main..feature/auth
# Same as:
git log ^main feature/auth
```

### 7.2 The `..` and `...` Operators

These are range operators — they filter the DAG:

```
A..B    = commits reachable from B, but NOT from A
          ("what's in B that's not in A?")
          Equivalent to: ^A B

A...B   = commits reachable from A OR B, but NOT BOTH
          ("what's different between A and B in either direction?")
          Symmetric difference
```

```
DAG:
  C1 ◄── C2 ◄── C3 ◄── C4 ◄── C5    ← main
                   \
                    C6 ◄── C7         ← feature/auth

git log main..feature/auth
→ Shows: C7, C6
  (reachable from feature, not from main)

git log feature/auth..main
→ Shows: C5, C4
  (reachable from main, not from feature)

git log main...feature/auth
→ Shows: C7, C6, C5, C4
  (reachable from either, but not both — the diverged parts)
```

### 7.3 git log — Key Flags

```bash
git log [options] [revision range] [-- path]
```

| Flag | What It Does |
|---|---|
| `--oneline` | Show `<short-sha> <message>` on one line |
| `--graph` | Draw ASCII art of the branch structure |
| `--all` | Start traversal from ALL refs (not just HEAD) |
| `--decorate` | Show ref names (branch, tag labels) beside commits |
| `--decorate=full` | Show full ref paths (`refs/heads/main` instead of `main`) |
| `-n <number>` | Show at most N commits |
| `--since="2 weeks ago"` | Only commits newer than this date |
| `--until="yesterday"` | Only commits older than this date |
| `--author="Parsh"` | Only commits by this author (regex) |
| `--grep="fix"` | Only commits whose message matches pattern |
| `--follow` | Follow file renames (use with `-- <path>`) |
| `-p` | Show the diff (patch) for each commit |
| `--stat` | Show file change stats per commit |
| `--name-only` | Show only filenames changed per commit |
| `-S "keyword"` | Show commits that added/removed "keyword" (pickaxe) |
| `-G "regex"` | Show commits where diff matches regex |
| `--merges` | Show only merge commits |
| `--no-merges` | Exclude merge commits |
| `--first-parent` | Only follow first parent (skip merged branches) |
| `--reverse` | Show oldest commits first |
| `-- <path>` | Only show commits that touched this file/directory |

```bash
# The "standard" team log view — use this constantly
git log --oneline --graph --all --decorate

# Who changed this file and when?
git log --oneline -- src/api/users.js

# What changes were made to this file in the last month?
git log --since="1 month ago" --stat -- src/api/users.js

# Find when a specific string was introduced
git log -S "getUserById" --oneline

# Show commits that are on feature but not on main
git log --oneline main..feature/auth
```

---

## 8. git reflog — Refs Have History Too

You now know that refs are files containing SHAs. But refs themselves also have
a **history** — every time a ref moves, Git records it.

This is called the **reflog** and it lives in `.git/logs/`:

```
.git/logs/
├── HEAD                     ← every movement of HEAD
└── refs/
    ├── heads/
    │   ├── main             ← every movement of main pointer
    │   └── feature/auth     ← every movement of feature/auth pointer
    └── remotes/
        └── origin/
            └── main         ← every fetch that updated origin/main
```

```bash
# See where HEAD has been
git reflog
# Output:
# c7f3a9b (HEAD -> main) HEAD@{0}: commit: feat: add payment
# a1e4d8c HEAD@{1}: checkout: moving from feature/auth to main
# 9b2f7e1 HEAD@{2}: commit: feat: add auth endpoints
# 7e2b9c4 HEAD@{3}: checkout: moving from main to feature/auth
# 7e2b9c4 HEAD@{4}: commit: chore: project setup
# ...

# See where a specific branch has been
git reflog show main
```

**Why this matters:** The reflog is your TIME MACHINE. Even after:
- `git reset --hard` (moved branch pointer backward)
- `git branch -d feature` (deleted the pointer file)
- `git rebase` (replaced commits with new SHAs)

...all old commit SHAs are still in the reflog. You can always recover.

Topic 20 covers this in full depth. For now, know it exists and why.

---

## 9. ORIG_HEAD, MERGE_HEAD, CHERRY_PICK_HEAD — Special Refs

Git creates temporary special ref files during multi-step operations:

```
.git/ORIG_HEAD         Created by: merge, rebase, reset
                       Contains: where HEAD was BEFORE the operation
                       Use: quick undo of a "dangerous" operation

.git/MERGE_HEAD        Created by: in-progress merge
                       Contains: SHA of the commit being merged in
                       Cleared: when merge completes or is aborted

.git/REBASE_HEAD       Created by: in-progress rebase
                       Contains: the commit currently being replayed

.git/CHERRY_PICK_HEAD  Created by: in-progress cherry-pick
                       Contains: the commit being cherry-picked

.git/BISECT_HEAD       Created by: git bisect
                       Contains: current commit being tested
```

```bash
# After a merge, ORIG_HEAD tells you where you were before
git merge feature/auth       # creates .git/MERGE_HEAD during, .git/ORIG_HEAD after

# Undo a merge that you regret (using ORIG_HEAD):
git reset --hard ORIG_HEAD
```

---

## 10. Commit Ranges — Full Reference

### 10.1 Single Commits

```bash
HEAD                     # current commit
main                     # tip of main branch  
c7f3a9b                  # specific commit by SHA
v1.0.0                   # tagged commit
HEAD@{2}                 # where HEAD was 2 moves ago (from reflog)
main@{yesterday}         # where main was yesterday
```

### 10.2 Ranges

```bash
main..feature            # commits in feature, not in main
feature..main            # commits in main, not in feature
main...feature           # commits in either but not both (symmetric diff)
^C3 C8                   # commits reachable from C8, excluding C3 and its ancestors
C3..C8                   # same thing (shorthand)
C1 C2 C3                 # union: all commits reachable from C1, C2, OR C3
```

### 10.3 Using Ranges with Other Commands (Not Just log)

```bash
# Count commits between two points
git rev-list --count main..feature/auth

# List commits between two points
git rev-list main..feature/auth

# Show diff of changes introduced by a range
git diff main...feature/auth

# Format patches for a range (for email-based workflows)
git format-patch main..feature/auth
```

---

## 11. git rev-list — The DAG Walker

`git rev-list` is the low-level command that `git log` uses internally. It outputs
commit SHAs in traversal order.

```bash
git rev-list HEAD
# Outputs ALL commit SHAs from HEAD backward, one per line

git rev-list --count HEAD
# Count all commits reachable from HEAD

git rev-list --count main..feature/auth
# Count commits on feature/auth not on main

git rev-list --oneline HEAD -10
# Last 10 commits in machine-readable format (SHA + message)

git rev-list --ancestry-path C1..C5
# Only commits on the direct path from C1 to C5
# (excludes commits that are ancestors of C5 but not descendants of C1)
```

---

## 12. Finding the Merge Base — The Common Ancestor

The **merge base** is the most recent common ancestor of two branches.
It's the commit where the two branches diverged.

```bash
git merge-base main feature/auth
# Output: 7e2b9c4a...  (the SHA of the divergence point)

# See it in context
git log --oneline --graph main feature/auth
```

Why you need this:
- `git merge` needs it to know what changed on EACH branch since they diverged
- `git rebase` needs it to know which commits to replay
- `git diff` with `...` uses it to show only the changes introduced by a branch

```bash
# Show what feature/auth changed since it diverged from main:
git diff $(git merge-base main feature/auth) feature/auth

# Shorthand (same result):
git diff main...feature/auth
```

---

## 13. Visualizing the DAG — Practical Commands

### 13.1 ASCII Graph in Terminal

```bash
# Standard view — use this all the time
git log --oneline --graph --all --decorate

# Example output:
# * c7f3a9b (HEAD -> main) feat: add payment processing
# | * 9b2f7e1 (feature/auth) feat: add JWT validation
# | * 7e2b9c4 feat: add login endpoint
# |/
# * a1e4d8c feat: initial API structure
# * 3d8c5a4 chore: project setup
# * 9b2f7e1 init: first commit
```

### 13.2 Compact Graph with Stats

```bash
git log --oneline --graph --all --decorate --stat

# Example:
# * c7f3a9b (HEAD -> main) feat: add payment processing
# |  src/api/payments.js | 45 +++++++++++++++++++++
# |  tests/payments.test.js | 30 +++++++++++++++
# |  2 files changed, 75 insertions(+)
```

### 13.3 Custom Pretty Format

```bash
git log --pretty=format:"%h %ad | %s%d [%an]" --date=short --graph --all

# %h = short SHA
# %ad = author date (formatted by --date=)
# %s = subject (first line of message)
# %d = ref names (HEAD, branches, tags)
# %an = author name
```

---

## 14. Beginner Example — Watch the DAG Build Step by Step

```bash
mkdir dag-demo && cd dag-demo
git init
```

```bash
# COMMIT 1
echo "# API" > README.md
git add README.md
git commit -m "init: project start"

# State:
# C1 ← HEAD, main
git log --oneline --graph --all
# * a1b2c3d (HEAD -> main) init: project start
```

```bash
# COMMIT 2
echo "console.log('server')" > server.js
git add server.js
git commit -m "feat: add server entry point"

# State:
# C1 ← C2 ← HEAD, main
git log --oneline --graph --all
# * b2c3d4e (HEAD -> main) feat: add server entry point
# * a1b2c3d init: project start
```

```bash
# CREATE AND SWITCH TO FEATURE BRANCH
git checkout -b feature/users

# COMMIT 3 (on feature/users)
echo "module.exports = {}" > users.js
git add users.js
git commit -m "feat: add users module"

# State:
# C1 ← C2 ← C3 ← HEAD, feature/users
#              ↑
#             main (still pointing here)
git log --oneline --graph --all
# * c3d4e5f (HEAD -> feature/users) feat: add users module
# * b2c3d4e (main) feat: add server entry point
# * a1b2c3d init: project start
```

```bash
# SWITCH BACK TO MAIN AND COMMIT SOMETHING
git checkout main
echo "PORT=3000" > .env
git add .env
git commit -m "chore: add env config"

# State:
# C1 ← C2 ← C4 ← HEAD, main
#              \
#               C3 ← feature/users
git log --oneline --graph --all
# * d4e5f6a (HEAD -> main) chore: add env config
# | * c3d4e5f (feature/users) feat: add users module
# |/
# * b2c3d4e feat: add server entry point
# * a1b2c3d init: project start
```

**You just created a diverged DAG.** Two branches, one common ancestor (C2).

```bash
# Verify with rev-list
git rev-list --count main..feature/users   # Output: 1 (C3)
git rev-list --count feature/users..main   # Output: 1 (C4)
git merge-base main feature/users          # Output: SHA of C2
```

---

## 15. Real-World Example — Production Debugging Scenario

> **Scenario:** You're a senior backend engineer. A junior says:
> *"I accidentally ran `git reset --hard HEAD~3` on the main branch. 
> Three important commits are gone. What do we do?"*

You know the DAG. Here's your response:

```bash
# Step 1: The commits are NOT gone. They're in the DAG, just unreferenced.
# Find them in the reflog.
git reflog show main

# Output includes:
# c7f3a9b main@{0}: reset: moving to HEAD~3     ← current bad state
# 9b2f7e1 main@{1}: commit: feat: add payments  ← this is what we want
# 7e2b9c4 main@{2}: commit: feat: add orders
# 3d8c5a4 main@{3}: commit: feat: add inventory

# The SHA we want is main@{1} = 9b2f7e1

# Step 2: Move main back to where it was
git reset --hard main@{1}
# or: git reset --hard 9b2f7e1

# Step 3: Verify
git log --oneline -5
# Should show the three commits are back
```

**Why this works:** `git reset --hard HEAD~3` only moved the POINTER (`.git/refs/heads/main`).
The commit OBJECTS (C_payments, C_orders, C_inventory) still exist in `.git/objects/`.
The reflog recorded the old pointer position. We just moved the pointer back.

Total recovery time: ~30 seconds.

---

## 16. Common Mistakes

### ❌ Mistake 1: Thinking "I'm on a branch" means commits are isolated to that branch

**The mistake:** "I committed to feature/auth so those commits are only on feature/auth."

**The truth:** Commits are SHARED when branches diverge from common ancestors. Every
commit on `main` is also "visible from" `feature/auth` (by following parent pointers
backward). Branches don't isolate commits — they only define the TIP from which you
traverse backward.

```bash
# feature/auth can see all of main's history before the branch point:
git log feature/auth --oneline
# Shows: feature commits + ALL commits that came before the branch point
```

---

### ❌ Mistake 2: Using `git log` without `--all` and missing branches

**The mistake:** Running `git log` and thinking you see all history.

**The truth:** `git log` by default only shows commits reachable from HEAD.
If HEAD is on `main` and `feature/auth` has diverged commits, you won't see them.

```bash
# WRONG: might miss commits on other branches
git log --oneline --graph

# CORRECT: see the full picture
git log --oneline --graph --all --decorate
```

---

### ❌ Mistake 3: Treating the `..` and `...` operators as the same thing

**The mistake:** Using `main..feature` and `main...feature` interchangeably.

```bash
git log main..feature   # commits in feature NOT in main (one direction)
git log main...feature  # commits in EITHER but not BOTH (symmetric difference)

# If main has 2 unique commits and feature has 3 unique commits:
# main..feature shows: 3 commits
# main...feature shows: 5 commits (3 + 2)
```

---

### ❌ Mistake 4: Assuming branch deletion destroys the commits

**The mistake:** `git branch -d feature/auth` → "I lost all my work!"

**The reality:** The commit objects in `.git/objects/` are untouched. Only the
pointer file `.git/refs/heads/feature/auth` was deleted. The commits are
"unreachable" (no ref points to them) but exist until garbage collection.

```bash
# Recover:
git reflog   # find the last SHA that was on feature/auth
git checkout -b feature/auth <that-sha>
```

---

### ❌ Mistake 5: Not understanding what `origin/main` actually is

**The mistake:** "origin/main and main are both on GitHub's server."

**The truth:**
- `main` (local branch) = YOUR branch, points to YOUR latest commit
- `origin/main` (remote-tracking ref) = YOUR LOCAL RECORD of what GitHub had last time you fetched
- Neither one directly IS GitHub — they're both files in YOUR `.git/` folder

```
YOUR MACHINE:                         GITHUB:
  .git/refs/heads/main → SHA_A          main → SHA_C  (3 commits ahead of you)
  .git/refs/remotes/origin/main → SHA_B (stale — last fetched yesterday)

To update origin/main: git fetch
To update your main: git merge origin/main (or git pull = both)
```

---

## 17. What Actually Happens — DAG Traversal Internals

When you run `git log --all`, here's the exact algorithm:

```
1. COLLECT STARTING POINTS:
   Read all files in:
   - .git/refs/heads/       (all local branches)
   - .git/refs/tags/        (all tags)
   - .git/refs/remotes/     (all remote-tracking refs)
   - .git/HEAD              (current position)
   - .git/packed-refs       (any packed refs)
   
   Resolve each to a commit SHA.
   Put all SHAs in a priority queue (sorted by committer timestamp, newest first).

2. TRAVERSAL LOOP:
   While queue is not empty:
     Pop the commit with the most recent timestamp
     
     If already visited: skip
     Mark as visited
     
     Apply filters (--author, --grep, --since, etc.)
     If passes filters: output this commit
     
     Read the commit object from .git/objects/
     For each parent SHA in the commit:
       If parent not yet visited: push to priority queue

3. STOP CONDITIONS:
   Queue empty, OR
   --max-count reached, OR
   --until date reached, OR
   An exclusion boundary hit (e.g. ^main)
```

This is a standard **BFS/priority-queue graph traversal** — exactly what you'd
implement in a coding interview for "traverse a DAG."

---

## 18. Mental Model Checkpoint

**Can you answer these without looking?**

1. What does "Directed Acyclic Graph" mean in terms of Git commits?
2. A branch ref file contains a commit SHA. When does that SHA change?
3. What's the difference between `HEAD` in normal state vs detached HEAD state?
4. You have branches `main` and `feature`. `feature` was branched from `main` at commit C3. `main` is now at C5. `feature` is at C7. How many commits does `git log main..feature` show?
5. If you `git branch -d feature`, can you recover the commits? How?
6. What does `git fetch` do to your `.git/refs/remotes/origin/` directory?
7. What is `ORIG_HEAD` and when does Git create it?
8. If a commit has two parent SHAs in its content, what kind of commit is it?
9. What command shows you the common ancestor of `main` and `feature`?
10. Why can `git reflog` show you commits that no branch currently points to?

---

## 19. Quick Reference Card

| Command | What It Does |
|---|---|
| `git log --oneline --graph --all --decorate` | Visualize full DAG with all refs |
| `git log A..B` | Commits reachable from B but NOT from A |
| `git log A...B` | Commits reachable from A or B but not both |
| `git log --first-parent` | Follow only first parent (ignore merged branches) |
| `git log -S "keyword"` | Commits that added or removed "keyword" |
| `git log -- <path>` | Commits that touched a specific file |
| `git rev-list HEAD` | All commit SHAs from HEAD backward |
| `git rev-list --count HEAD` | Count all commits |
| `git rev-list --count A..B` | Count commits in B not in A |
| `git rev-parse HEAD` | Resolve HEAD to full SHA |
| `git rev-parse HEAD^` | SHA of HEAD's parent |
| `git rev-parse HEAD~3` | SHA of 3 commits back |
| `git merge-base A B` | Find common ancestor of A and B |
| `git reflog` | Show movement history of HEAD |
| `git reflog show <branch>` | Show movement history of a specific branch |
| `git symbolic-ref HEAD` | Show which branch HEAD points to |
| `cat .git/refs/heads/<branch>` | Raw SHA a branch points to |
| `cat .git/HEAD` | Raw content of HEAD file |
| `git branch -v` | List branches with their tip commit SHAs |

---

## 20. When Would I Use This At Work?

```
╔══════════════════════════════════════════════════════════════════════╗
║              WHEN WOULD I USE THIS AT WORK?                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. CODE REVIEW: "HOW MANY COMMITS IS THIS PR?"                     ║
║     git rev-list --count main..feature/auth                          ║
║     → Quick answer without opening GitHub UI                         ║
║                                                                      ║
║  2. RELEASE MANAGEMENT: "WHAT CHANGED SINCE v2.1.0?"               ║
║     git log v2.1.0..HEAD --oneline                                   ║
║     → See exactly what's going into the next release                 ║
║                                                                      ║
║  3. RECOVERING LOST WORK                                             ║
║     git reflog → find old SHA → git checkout -b recovery <sha>       ║
║     → Works after accidental reset, deleted branch, bad rebase       ║
║                                                                      ║
║  4. DEBUGGING: "WHEN DID THIS FUNCTION APPEAR?"                     ║
║     git log -S "functionName" --oneline                              ║
║     → Binary search through history for when code was introduced     ║
║                                                                      ║
║  5. CI/CD: "ONLY RUN TESTS IF RELEVANT FILES CHANGED"               ║
║     git diff --name-only origin/main...HEAD                          ║
║     → Script checks if src/ files changed, skips if only docs        ║
║                                                                      ║
║  6. EXPLAINING TO TEAM WHY REBASE BREAKS SHARED BRANCHES            ║
║     "Rebase rewrites commit objects. Other developers' branches      ║
║     reference the OLD SHAs which no longer exist in the DAG they'll  ║
║     receive after a force-push. Their history has diverged."         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Up Next:** [Topic 04 — Git Setup & Configuration](./topic-04-git-setup-configuration.md)
> We now understand what Git stores and how it's navigated.
> Next: configure Git properly for professional use — `.gitconfig`,
> global/local/system scopes, useful aliases, and settings senior engineers use.
