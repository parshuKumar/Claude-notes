# Topic 08 — Branching — Internals, Creation, Switching, Deletion
### What Physically Happens in `.git/` When You Branch

---

## Connection to Previous Topics

- **Topic 01** introduced `refs/heads/` — a directory where every branch is a plain text
  file containing a 40-char SHA plus newline. That single fact is the entire foundation
  of this topic.
- **Topic 03** showed you the DAG: branches are just named pointers into the commit graph.
  Branching is cheap because moving a pointer is cheap.
- **Topic 05** showed the Three Trees. Switching branches moves ALL THREE trees (HEAD,
  Index, Working Directory) to a new commit's snapshot.
- **Topic 06** showed that `git commit` moves the branch pointer forward. That's HOW
  branches grow — one pointer-move per commit.

If you understand that a branch is a 41-byte text file and commits are immutable objects,
branching in Git will never feel mysterious again.

---

## RULE 1 — ELI5 Anchor

### The Bookmark Analogy

Imagine a very long book of history (your commit graph). Every page is a snapshot of
your project at one moment in time (a commit). Pages are numbered with a code (SHA),
and each page says "the previous page was number X" (the parent pointer).

A **branch** is just a **sticky bookmark** you place on a page. The bookmark has a name
written on it ("main", "feature/auth", "hotfix/payment"). The bookmark doesn't change
the book — it just marks a specific page.

When you make a new commit, the bookmark slides to the new page automatically. That's all
branch advancement is: the bookmark slides forward.

When you "switch branches," you pick up a different bookmark and your desk (working
directory) rearranges itself to show the page THAT bookmark points to.

When you "create a branch," you put a new sticky bookmark on the current page. Now TWO
bookmarks point to the same page. Neither branch has "moved" yet — you just have two
names for the same snapshot.

The key insight: **creating a branch costs almost nothing.** Writing 41 bytes to a file.
That's it.

---

## RULE 2 — Physical Architecture First

### What a Branch Is on Disk

A branch is a **plain text file** inside `.git/refs/heads/`.

```
.git/
└── refs/
    └── heads/
        ├── main                ← 41 bytes: "d8329fc1cc938780ffdd9f94e0d364e0ea74f579\n"
        ├── feature/auth        ← 41 bytes: "a4b5c6d7e8f9a0b1c2d3e4f5g6h7i8j9k0l1m2n3\n"
        └── hotfix/payment      ← 41 bytes: "1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t\n"
```

That's it. A branch is:
- **File path**: `.git/refs/heads/<branch-name>`
- **File content**: 40-char SHA-1 hex string + `\n` = exactly 41 bytes

```bash
# PROVE IT: See a branch as a raw file
cat .git/refs/heads/main
# d8329fc1cc938780ffdd9f94e0d364e0ea74f579

# Count the bytes
wc -c .git/refs/heads/main
# 41 .git/refs/heads/main
# (40 hex chars + 1 newline = 41 bytes)

# The branch name is the filename (including any slash-separated path)
ls .git/refs/heads/
# main  feature/  hotfix/

ls .git/refs/heads/feature/
# auth

# The "full" branch name is: refs/heads/feature/auth
# The "short" name is: feature/auth
```

**Branch names with slashes create subdirectories:**
```
Branch: feature/auth
File:   .git/refs/heads/feature/auth
        └─ refs/heads/ ← always this prefix
                  └─ feature/ ← directory created automatically
                          └─ auth ← the file
```

---

### What HEAD Is on Disk (Normal vs Detached)

**Normal state (on a branch):**
```
.git/HEAD
contents: "ref: refs/heads/main\n"
```

HEAD contains a **symbolic reference** — a pointer to a pointer. It says "I am currently
on the branch named `main`, and `main` is at `.git/refs/heads/main`."

When you make a commit on `main`:
1. Git writes the new commit SHA to `.git/refs/heads/main`
2. `.git/HEAD` stays the same — it still says `ref: refs/heads/main`
3. HEAD "moves forward" only because the branch file changed

**Detached HEAD state:**
```
.git/HEAD
contents: "a4b5c6d7e8f9a0b1c2d3e4f5g6h7i8j9k0l1m2n3\n"
          ← A direct SHA, NOT a "ref: refs/heads/..." line
```

When HEAD contains a SHA directly, no branch moves when you commit. You're working in
"read-only history browsing mode" — commits made here become orphans unless you create
a branch pointing to them.

```bash
# PROVE IT: See what HEAD contains
cat .git/HEAD
# Normally: ref: refs/heads/main
# Detached:  a4b5c6d7e8f9...

# Which branch am I on?
git branch --show-current
# main (or empty string if detached)

# Check for detached HEAD status
git status | head -1
# On branch main
# OR: HEAD detached at a4b5c6d
```

---

### The Full `.git/` State with Multiple Branches

```
.git/
├── HEAD                        ← "ref: refs/heads/main\n"
├── ORIG_HEAD                   ← set during merges, resets — Topic 03/12
├── config                      ← branch tracking config goes here
├── index                       ← current branch's staged state
├── objects/
│   ├── d8/
│   │   └── 329fc1...           ← commit A (tip of main)
│   ├── a4/
│   │   └── b5c6d7...           ← commit B (tip of feature/auth, also parent of C)
│   ├── (all other objects)
│   └── pack/
└── refs/
    ├── heads/
    │   ├── main                ← "d8329fc1...\n"   ← points to commit A
    │   └── feature/
    │       └── auth            ← "a4b5c6d7...\n"   ← points to commit B
    ├── remotes/                ← remote-tracking refs (Topic 10)
    │   └── origin/
    │       └── main            ← "f0e9d8c7...\n"
    └── tags/                   ← tag refs (Topic 14)
```

---

## RULE 3 — Exact Data Flow: `git branch feature/auth`

### Creating a Branch (Without Switching)

```
COMMAND: git branch feature/auth

PRE-CONDITION: We're on main at commit d8329fc1...

READS:    .git/HEAD → "ref: refs/heads/main\n"
          .git/refs/heads/main → "d8329fc1cc938780ffdd9f94e0d364e0ea74f579\n"

COMPUTES: New branch should point to current HEAD commit:
          target_sha = "d8329fc1cc938780ffdd9f94e0d364e0ea74f579"

CREATES:  Directory .git/refs/heads/feature/ (if it doesn't exist)
WRITES:   .git/refs/heads/feature/auth
          Content: "d8329fc1cc938780ffdd9f94e0d364e0ea74f579\n"

DOES NOT MODIFY:
  - .git/HEAD         (still "ref: refs/heads/main")
  - .git/index        (unchanged)
  - Working directory (unchanged)

RESULT:
  Two branches now point to the SAME commit:
  main         → d8329fc1...
  feature/auth → d8329fc1...   (same SHA, independent pointer)
```

**The `.git/` directory AFTER `git branch feature/auth`:**

```
.git/refs/heads/
├── main           ← "d8329fc1...\n"   ← unchanged
└── feature/
    └── auth       ← "d8329fc1...\n"   ← NEW file, same SHA as main
```

```bash
# PROVE IT: Both branches point to the same commit
cat .git/refs/heads/main
cat .git/refs/heads/feature/auth
# Both show: d8329fc1cc938780ffdd9f94e0d364e0ea74f579

# Or with git:
git rev-parse main
git rev-parse feature/auth
# Both print the same SHA

# HEAD is still on main
cat .git/HEAD
# ref: refs/heads/main
```

---

## RULE 3 — Exact Data Flow: `git switch feature/auth`

### Switching to a Branch — The Full Internal Process

```
COMMAND: git switch feature/auth

PRE-CONDITION:
  Current branch: main (d8329fc1...)
  Target branch:  feature/auth (a4b5c6d7...)
  These are DIFFERENT commits with DIFFERENT trees.

STEP 1 — Safety check
  READS:    .git/index
  COMPARES: index vs working directory (any uncommitted modifications?)
  IF DIRTY: Git refuses the switch (by default) to protect your work
            Exception: if the modifications don't conflict with the target tree,
            Git MAY allow it (brings the changes with you) — see below

STEP 2 — Read target commit
  READS:    .git/refs/heads/feature/auth → a4b5c6d7...
  READS:    .git/objects/a4/b5c6d7... (commit object)
  EXTRACTS: tree SHA from commit → target_tree = "xyz789..."

STEP 3 — Read current commit's tree
  READS:    .git/HEAD → .git/refs/heads/main → d8329fc1...
  READS:    .git/objects/d8/329fc1... (commit object)
  EXTRACTS: tree SHA → current_tree = "abc123..."

STEP 4 — Compute diff between trees
  COMPARES: current_tree (abc123) vs target_tree (xyz789)
  FINDS:    Which files need to be added / modified / removed

STEP 5 — Update the working directory
  For each file that differs:
    - Files present in target but not current: CREATE in working directory
    - Files present in current but not target: DELETE from working directory
    - Files that differ between current/target: OVERWRITE in working directory
    - Files identical in both trees: LEAVE UNTOUCHED

STEP 6 — Update the Index
  WRITES:   .git/index with entries from target_tree
  The index now reflects the full state of feature/auth's tree

STEP 7 — Update HEAD
  WRITES:   .git/HEAD → "ref: refs/heads/feature/auth\n"
```

**The `.git/` state AFTER `git switch feature/auth`:**

```
.git/HEAD     ← NOW: "ref: refs/heads/feature/auth\n"
.git/index    ← NOW: entries matching feature/auth's tree
Working dir   ← NOW: files matching feature/auth's snapshot
.git/refs/heads/main        ← UNCHANGED: still "d8329fc1...\n"
.git/refs/heads/feature/auth ← UNCHANGED: still "a4b5c6d7...\n"
```

**Key insight: `git switch` moves the THREE TREES in unison:**

```
BEFORE git switch:
  HEAD   → main → d8329fc1 → tree-A
  Index  → matches tree-A
  WD     → matches tree-A

AFTER git switch feature/auth:
  HEAD   → feature/auth → a4b5c6d7 → tree-B
  Index  → matches tree-B
  WD     → matches tree-B
```

```bash
# PROVE IT: Watch HEAD change
cat .git/HEAD
# ref: refs/heads/main

git switch feature/auth

cat .git/HEAD
# ref: refs/heads/feature/auth

# Watch index change
git ls-files --stage | head -5
# Now shows the files from feature/auth's tree

# Confirm working directory changed
ls -la
# Shows files as they exist in feature/auth
```

---

## RULE 4 — ASCII Architecture Diagram: Before and After Switching

### Before: On main

```
Working Dir    Index         Repository (Objects + Refs)
───────────    ─────         ──────────────────────────

server.js ─── index entry ──► blob-V (server.js content v1)
styles.css ── index entry ──► blob-S (styles.css)
                │                         ▼
                └──────────────► tree-A (root) = blob-V + blob-S
                                         ▼
                                    commit-D ("Add styles")
                                         ▲
HEAD ──────────────────────────► refs/heads/main ───────┘
```

### After: git switch feature/auth

```
Working Dir    Index         Repository (Objects + Refs)
───────────    ─────         ──────────────────────────

server.js ─── index entry ──► blob-W (server.js content v2, auth added)
auth.js ────── index entry ──► blob-A (auth.js — new file)
                │                         ▼
                └──────────────► tree-B (root) = blob-W + blob-A
                                         ▼
                                    commit-X ("feat: add auth")
                                         ▲
HEAD ──────────────────────────► refs/heads/feature/auth ─────┘

Note: styles.css DOES NOT APPEAR in WD or index
      because feature/auth's tree doesn't include it.
      Git DELETED it from the working directory during the switch.
```

---

## RULE 4 — The Branch Creation + Switch Diagram

### Creating and Switching: What Moves

```
INITIAL STATE (3 commits on main):

  C1 ──► C2 ──► C3
                 ▲
                 │
              main ◄── HEAD

AFTER: git branch feature/login

  C1 ──► C2 ──► C3
                 ▲   ▲
                 │   │
              main   feature/login ◄── both point to C3
                 │
                HEAD (still on main)

AFTER: git switch feature/login

  C1 ──► C2 ──► C3
                 ▲   ▲
                 │   │
              main   feature/login ◄── HEAD (moved to feature/login)

AFTER: git commit (new commit C4 on feature/login)

  C1 ──► C2 ──► C3 ──► C4
                 ▲           ▲
                 │           │
              main        feature/login ◄── HEAD

  NOTICE: main did NOT move. C4 exists only on feature/login.
```

---

## All Branch-Related Commands — Deep Dive

### Creating Branches

#### Method 1: `git branch <name>` — Create without switching

```bash
git branch feature/auth
# Creates .git/refs/heads/feature/auth pointing to current HEAD
# Does NOT switch — HEAD is unchanged
```

#### Method 2: `git switch -c <name>` — Create AND switch (modern, recommended)

```bash
git switch -c feature/auth
# EQUIVALENT to:
#   git branch feature/auth    (create the ref file)
#   git switch feature/auth    (update HEAD, index, WD)
```

#### Method 3: `git checkout -b <name>` — Legacy create + switch

```bash
git checkout -b feature/auth
# Same as git switch -c, but uses the older porcelain command
# Works identically; git checkout is the historical command,
# git switch was introduced in Git 2.23 (2019) to be clearer
```

#### Create a branch from a specific point (not current HEAD)

```bash
# From a specific commit SHA
git branch feature/auth abc1234

# From another branch
git branch feature/auth main
git branch feature/auth origin/main

# From a tag
git branch hotfix/v1.2.1 v1.2.0

# Create AND switch to a branch from a specific point
git switch -c feature/auth abc1234
git checkout -b feature/auth abc1234
```

**Data flow for `git branch feature/auth abc1234`:**
```
READS:    abc1234 (resolve to full SHA via git rev-parse)
WRITES:   .git/refs/heads/feature/auth → "<full-sha-of-abc1234>\n"
```

---

### Switching Branches

#### `git switch` (modern, Git ≥ 2.23)

```bash
git switch main              # switch to existing branch
git switch -c new-branch     # create + switch
git switch -                 # switch to previous branch (like cd -)
git switch --detach abc1234  # detach HEAD to a commit
git switch --detach v1.2.0   # detach HEAD to a tag's commit
```

#### `git checkout` (legacy, still works)

```bash
git checkout main            # switch to existing branch
git checkout -b new-branch   # create + switch
git checkout -               # switch to previous branch
git checkout abc1234         # detach HEAD to a commit
git checkout HEAD~3          # detach HEAD to 3 commits back

# DANGER: git checkout has DUAL USE that git switch avoids:
git checkout server.js       # RESTORE FILE from HEAD (not branching at all!)
                             # This is confusing — git restore (Topic 12) is clearer
```

**Why `git switch` was introduced:**
`git checkout` does two completely different things:
1. Switch branches: `git checkout main`
2. Restore files: `git checkout -- server.js`

These two operations are unrelated. `git switch` handles branching.
`git restore` handles file restoration. The split was made in Git 2.23.

---

### Listing Branches

```bash
git branch
# Lists all LOCAL branches
# The * marks the current branch (HEAD)
# Example:
#   * feature/auth
#     hotfix/payment
#     main

git branch -v
# Verbose: shows last commit SHA and message
#   * feature/auth  a4b5c6d feat: implement JWT auth
#     hotfix/payment 1a2b3c4 fix: prevent double charge
#     main           d8329fc chore: initial setup

git branch -vv
# Very verbose: also shows upstream tracking (remote branch)
#   * feature/auth  a4b5c6d [origin/feature/auth: ahead 2] feat: implement JWT auth
#     main          d8329fc [origin/main] chore: initial setup
#     local-only    f1e2d3c (no upstream configured)

git branch -a
# ALL branches: local + remote-tracking
#   * feature/auth
#     main
#     remotes/origin/feature/auth
#     remotes/origin/main

git branch -r
# REMOTE-TRACKING branches only
#   origin/feature/auth
#     origin/main

git branch --merged
# Branches whose tip IS an ancestor of current HEAD
# (safe to delete — their work is already in current branch)

git branch --no-merged
# Branches with commits NOT in current HEAD
# (deleting these loses commits unless merged first)

git branch --merged main
# Branches already merged into main (regardless of current HEAD)

git branch --sort=-committerdate
# Sort by most recently committed to (useful for finding active branches)

git branch --contains <sha>
# Which branches contain a specific commit?
# Useful: "which branches have this hotfix?"
```

---

## RULE 3 — Exact Data Flow: `git branch -d feature/auth`

### Deleting a Branch — The Full Trace

```
COMMAND: git branch -d feature/auth

PRE-CONDITION: We're on 'main'. feature/auth has been merged into main.

STEP 1 — Safety check (for -d, not -D)
  READS:    .git/refs/heads/feature/auth → a4b5c6d7...
  CHECKS:   Is a4b5c6d7 reachable from current HEAD (main)?
  METHOD:   Is a4b5c6d7 an ancestor of HEAD?
            Uses git merge-base to check if it's in the ancestor chain.
  IF NOT:   Refuses deletion with error:
            "error: The branch 'feature/auth' is not fully merged.
             If you are sure you want to delete it, run 'git branch -D feature/auth'."
  IF YES:   Proceed

STEP 2 — Delete the ref file
  DELETES:  .git/refs/heads/feature/auth
  If feature/ directory is now empty:
  DELETES:  .git/refs/heads/feature/  (the directory)

STEP 3 — Append deletion to reflog
  APPENDS:  .git/logs/refs/heads/feature/auth
            Line: "a4b5c6d7... 0000000000000000000000000000000000000000 user <email> timestamp\tdeleted"
            (This means git reflog can recover a deleted branch — Topic 20)

DOES NOT MODIFY:
  - Any commit objects (a4b5c6d7... still exists in .git/objects/ until GC)
  - .git/HEAD (you can't delete the branch you're on)
  - Any other branch
```

**Critical:** Deleting a branch does NOT delete commits. The commit objects remain in
`.git/objects/` as orphans until `git gc` (garbage collection) runs. You can recover
a deleted branch by pointing a new branch at its old commit SHA (from reflog).

```bash
# PROVE IT: Recover a deleted branch
git branch feature/auth     # will use to get SHA
git log --oneline feature/auth | head -1
# a4b5c6d feat: implement JWT auth

git branch -d feature/auth

# Oops! Let's recover it:
git reflog | grep feature/auth
# a4b5c6d HEAD@{5}: checkout: moving from feature/auth to main
# a4b5c6d refs/heads/feature/auth@{0}: Created from ...

# Recreate the branch
git branch feature/auth a4b5c6d

# Or use reflog directly
git checkout -b feature/auth HEAD@{5}
```

---

## RULE 6 — Deep Syntax Breakdown

### `git branch`

```
git branch  [<branchname>]  [<start-point>]
│    │        │               │
│    │        │               └─ Where the new branch starts.
│    │        │                  Can be: SHA, branch name, tag, HEAD~3, etc.
│    │        │                  Default: current HEAD
│    │        └─ Name of the new branch to create
│    │           Must not contain spaces, ~, ^, :, ?, *, [, \
│    │           CAN contain / (creates subdirectory in refs/heads/)
│    └─ subcommand: manage branch pointers
└─ git binary

FLAGS:
  -d <branch>     Delete (only if fully merged)
  -D <branch>     Force delete (even if not merged)
  -m <old> <new>  Rename a branch
  -M <old> <new>  Force rename (even if new name exists)
  -c <old> <new>  Copy a branch (keeps original)
  -C <old> <new>  Force copy
  -v              Verbose: show last commit
  -vv             Very verbose: show tracking info
  -a              Show all (local + remote-tracking)
  -r              Show remote-tracking only
  --merged        Show branches merged into HEAD
  --no-merged     Show branches NOT merged into HEAD
  --contains <sha> Show branches that contain a commit
  --sort=<key>    Sort by: committerdate, authordate, refname
  --list <pattern> Filter branch names by pattern
  --set-upstream-to=<remote/branch>  Set tracking for current branch
  --unset-upstream  Remove tracking info
  -u <remote/branch>  Short for --set-upstream-to
  --move          Same as -m
  --copy          Same as -c
  --format=<fmt>  Custom output format
```

### `git switch`

```
git switch  [--create / -c]  [--force-create / -C]  [<branch>]
│    │        │                │                      │
│    │        │                │                      └─ branch name to switch to
│    │        │                │                         (must exist, unless -c/-C)
│    │        │                └─ Force-create: create even if branch exists
│    │        │                   (resets existing branch to new start point)
│    │        └─ Create: create new branch AND switch to it
│    │           Usage: git switch -c feature/name [start-point]
│    └─ subcommand: switch branches (Git ≥ 2.23)
└─ git binary

FLAGS:
  -c / --create <branch>         Create + switch
  -C / --force-create <branch>   Force-create + switch (reset if exists)
  --detach                       Switch to commit (detach HEAD)
  -                              Switch to previous branch
  --discard-changes              Discard local changes (like --hard)
  --merge                        Merge local changes with target (3-way)
  --orphan <branch>              Create an orphan branch (no parent)
  --guess / --no-guess           Auto-guess remote tracking branch
  --progress / --no-progress     Show progress
```

### `git checkout` (legacy, still widely used)

```
git checkout  [-b / -B]  [<branch>]  [<start-point>]
│    │          │          │           │
│    │          │          │           └─ Where to start (for -b)
│    │          │          └─ Branch to switch to (or create with -b)
│    │          └─ -b: create + switch; -B: force-create + switch
│    └─ subcommand: OVERLOADED (switch branches OR restore files)
└─ git binary

When used as branch switch:
  git checkout main                  ← switch to main
  git checkout -b feature/auth       ← create + switch
  git checkout -b feature/auth main  ← create from main + switch
  git checkout abc1234               ← detach HEAD at commit

When used as file restore (DIFFERENT operation):
  git checkout -- server.js          ← restore server.js from index
  git checkout HEAD -- server.js     ← restore server.js from HEAD commit
  (prefer git restore for this — cleaner, less confusing)
```

---

## RULE 9 — Byte-Level: The `packed-refs` File

### When Branches Don't Live in `refs/heads/`

For performance, Git can "pack" many refs into a single file: `.git/packed-refs`.

After `git pack-refs --all` or after cloning a large remote:

```
.git/packed-refs
# pack-refs with: peeled fully-peeled sorted
d8329fc1cc938780ffdd9f94e0d364e0ea74f579 refs/heads/main
a4b5c6d7e8f9a0b1c2d3e4f5g6h7i8j9k0l1m2n refs/heads/feature/auth
1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s refs/tags/v1.0.0
^d8329fc1cc938780ffdd9f94e0d364e0ea74f579   ← peeled tag (tag → commit)
```

**Format:**
```
<sha> <refname>
```

Lines starting with `^` are "peeled" refs — for annotated tags, the `^` line shows the
commit SHA that the tag ultimately points to (tag object → commit object).

**Precedence rule:**
If a ref exists as BOTH a loose file (`refs/heads/main`) AND in `packed-refs`, the
**loose file wins**. This is how branch updates work even when packed:
- `git commit` writes a NEW loose file for the branch
- The packed-refs entry becomes stale and is superseded

```bash
# PROVE IT: See packed refs
cat .git/packed-refs
# Shows all packed refs

# Force packing all refs
git pack-refs --all

# After packing, the loose files are removed:
ls .git/refs/heads/
# (empty! All branches are now in packed-refs)

# But git still works normally:
git branch
# Still shows all branches (reads from packed-refs)

# Making a new commit creates a NEW loose ref file:
echo "test" >> test.txt
git add test.txt
git commit -m "test"
ls .git/refs/heads/
# main  ← loose file reappeared for main (new commit)
```

---

## Deep Dive: Detached HEAD State

### What It Is

When HEAD contains a SHA directly (not `ref: refs/heads/...`), you're in detached HEAD:

```
.git/HEAD contents:
  NORMAL:   "ref: refs/heads/main\n"
  DETACHED: "a4b5c6d7e8f9a0b1c2d3e4f5g6h7i8j9k0l1m2n3\n"
```

### When Does Detached HEAD Happen?

```bash
git switch --detach abc1234    # explicitly detach to commit
git checkout abc1234           # legacy: check out a commit directly
git checkout HEAD~3            # go 3 commits back
git checkout v1.2.0            # check out a tag (unless it's a branch)
git bisect start               # bisect moves HEAD around
git rebase                     # during rebase, HEAD is temporarily detached
```

### What Happens If You Commit in Detached HEAD?

```bash
# You're detached at abc1234
# You make changes and commit:
git commit -m "experimental change"
# New commit: def5678 (parent: abc1234)
# HEAD now contains "def5678\n" (moved forward, still detached)
```

```
Before commit:
  C1──►C2──►C3──►C4    (main → C4)
              ▲
              HEAD (detached, pointing at C3)

After commit:
  C1──►C2──►C3──►C4    (main → C4, unchanged)
              │
              └──►C5    (def5678 — your experimental commit)
                   ▲
                   HEAD (detached, pointing at C5)
```

**The danger**: When you switch away from this state:
```bash
git switch main
# Warning: you are leaving 1 commit behind, not connected to any of your branches.
# 
#   def5678 experimental change
# 
# If you want to keep it, you need to create a new branch:
# 
#   git branch <new-branch-name> def5678
```

The orphan commit `def5678` will be garbage-collected in 30 days unless you save it.

### Escaping Detached HEAD

```bash
# Option 1: Create a new branch at current position (SAVE the work)
git switch -c my-experiment     # create branch at current detached HEAD

# Option 2: Attach to an existing branch at this position
git branch -f some-branch HEAD  # move some-branch here
git switch some-branch

# Option 3: Abandon the detached work and go back
git switch main                 # warns about orphan commit, but you say OK

# Option 4: Save just the commit SHA and branch later
git log -1 --format="%H"        # copy the SHA
git switch main
git branch experiment <saved-sha>
```

---

## Deep Dive: Orphan Branches

### What Is an Orphan Branch?

An orphan branch has NO parent commit — it starts a completely fresh history.

```bash
git switch --orphan documentation
# or
git checkout --orphan documentation
```

```
BEFORE (on main with history C1→C2→C3):
  HEAD → main → C3

AFTER git switch --orphan documentation:
  HEAD → documentation → (NOTHING — branch ref file not created yet)
  Index: CLEARED (same files as before but staged for first commit)
  WD:    UNCHANGED (your files are still there, but unlinked from old commits)
```

The orphan branch ref file only gets created when you make your FIRST commit on it.

**Practical use:**
- GitHub Pages (`gh-pages` branch): documentation/static site with its own history,
  completely separate from main source code history
- Project restart: same repo, fresh history for a complete rewrite
- Data branches: storing generated data files without polluting source history

```bash
# Create a clean documentation branch
git switch --orphan gh-pages

# Remove all files staged from previous branch
git rm -rf .

# Create your docs
echo "# Documentation" > index.html
git add index.html
git commit -m "docs: initial GitHub Pages setup"

# Now gh-pages has a 1-commit history, no connection to main
git log --oneline
# abc1234 docs: initial GitHub Pages setup
```

---

## RULE 3 — Exact Data Flow: Renaming a Branch

```
COMMAND: git branch -m feature/auth feature/authentication

READS:    .git/refs/heads/feature/auth → SHA

RENAMES:  .git/refs/heads/feature/auth
          → .git/refs/heads/feature/authentication

ALSO UPDATES:
  .git/logs/refs/heads/feature/auth
          → .git/logs/refs/heads/feature/authentication
  (The reflog file is renamed too, preserving history)

IF HEAD was on the renamed branch:
  UPDATES: .git/HEAD → "ref: refs/heads/feature/authentication\n"

IF branch had a remote tracking configuration in .git/config:
  DOES NOT update it — you must update manually:
    git branch -u origin/feature/authentication
```

```bash
# PROVE IT: Rename a branch
git branch -m feature/auth feature/authentication

# If you renamed the branch you're on:
cat .git/HEAD
# ref: refs/heads/feature/authentication  ← updated!

# Config tracking might need update:
git config branch.feature/authentication.remote
git config branch.feature/authentication.merge
# May still say "feature/auth" — update if needed
```

---

## Dirty Working Directory and Branch Switching

### The 3 Scenarios When Switching with Uncommitted Changes

**Scenario 1: Modifications that CONFLICT with the target branch**

```bash
# On main, you've modified server.js
# feature/auth has a DIFFERENT version of server.js

git switch feature/auth
# error: Your local changes to the following files would be overwritten by checkout:
#         server.js
# Please commit your changes or stash them before you switch branches.
# Aborting.
```

Git REFUSES to proceed — your uncommitted changes would be lost/overwritten.

**Fix:**
```bash
# Option A: Stash your changes (Topic 13)
git stash
git switch feature/auth
git stash pop   # reapply later

# Option B: Commit your changes first
git commit -m "wip: save work in progress"
git switch feature/auth

# Option C: Discard your changes
git switch --discard-changes feature/auth
# or: git checkout -f feature/auth
# WARNING: Your changes are gone permanently
```

**Scenario 2: Modifications that do NOT conflict (Git ALLOWS the switch)**

```bash
# On main, you've created a NEW file that doesn't exist in feature/auth
echo "notes.txt" > notes.txt

git switch feature/auth
# Switched to branch 'feature/auth'
# Your file notes.txt is STILL in the working directory
# (Git brought it along because it doesn't conflict with the target tree)
```

This behavior is sometimes surprising: the uncommitted file "follows you" to the other
branch. The file appears as "untracked" on the new branch.

**Scenario 3: Staged changes**

```bash
# On main, you've staged a new file
git add new-feature.js

git switch feature/auth
# Switched to branch 'feature/auth'
# new-feature.js is STILL STAGED on feature/auth
# (Staged changes travel with you if they don't conflict)
```

Again surprising — staged changes follow you across branch switches unless there's a
conflict. Always `git status` before switching to understand what you're bringing.

---

## Branching Conventions at Companies (Naming Standards)

### Common Naming Patterns

```
main / master          ← production-ready code (protected)
develop / dev          ← integration branch (Gitflow — Topic 15)
release/1.5.0          ← release candidate
feature/<ticket>-<description>    ← e.g., feature/JIRA-142-add-dark-mode
fix/<ticket>-<description>        ← e.g., fix/BUG-891-payment-null-ref
hotfix/<version>-<description>    ← e.g., hotfix/1.4.1-critical-auth-bypass
chore/<description>               ← e.g., chore/upgrade-node-22
experiment/<description>          ← experimental, may be deleted anytime
```

### Why Branch Naming Matters

- CI/CD pipelines often MATCH branch names with rules:
  ```yaml
  # GitHub Actions example
  on:
    push:
      branches:
        - main        # → deploy to production
        - release/**  # → deploy to staging
        - feature/**  # → run tests only
  ```
- Branch protection rules on GitHub use name patterns
- Pull request tooling can auto-link `feature/JIRA-142` to Jira ticket #142
- `git branch --sort=-committerdate` sorted by date + grep makes sense with consistent names

---

## RULE 7 — Beginner Example

### Building a Feature Safely Isolated from Main

```bash
mkdir blog && cd blog
git init

# Commit initial structure on main
echo "# My Blog" > index.html
git add index.html
git commit -m "feat: create initial blog homepage"

# Check initial state
git log --oneline
# abc123 feat: create initial blog homepage

cat .git/HEAD
# ref: refs/heads/main

# Create a branch for a new About page feature
git switch -c feature/about-page

# Verify the switch
cat .git/HEAD
# ref: refs/heads/feature/about-page

# Verify both branches point to the same commit
git rev-parse main
git rev-parse feature/about-page
# Same SHA!

# Do the work on the feature branch
echo "<h1>About Me</h1>" > about.html
git add about.html
git commit -m "feat: add about page"

# Now branches diverge
git rev-parse main             # abc123
git rev-parse feature/about-page  # def456 (new commit)

# Switch back to main — about.html DISAPPEARS from working directory
git switch main
ls
# index.html   ← only this file (about.html is on feature/about-page)

# about.html is NOT lost — it's in the feature/about-page tree
git switch feature/about-page
ls
# about.html  index.html   ← both files

# When satisfied: merge into main (Topic 09)
git switch main
git merge feature/about-page

# Clean up the branch after merge
git branch -d feature/about-page
```

---

## RULE 7 — Production Example

### The Emergency Hotfix While Mid-Feature

You're a backend engineer, currently working on `feature/stripe-v2` (half-complete,
multiple commits). A production incident occurs — payment processing is returning 500 errors.

```bash
# Step 1: Check current state
git status
# On branch feature/stripe-v2
# Changes not staged for commit:
#   modified: src/payments/processor.js
#   modified: src/payments/webhook.js

git log --oneline -3
# f1e2d3 feat: refactor payment processor for Stripe v2
# a4b5c6 feat: add webhook event types for Stripe v2
# d7e8f9 chore: upgrade stripe SDK

# Step 2: Save work in progress
git stash push -m "wip: stripe v2 integration (mid-refactor)"

# Step 3: Go to main (production baseline)
git switch main
git pull origin main   # ensure you have latest production code

# Step 4: Create hotfix branch FROM MAIN
git switch -c hotfix/INCIDENT-1087-payment-500

# Step 5: Fix the bug
vim src/payments/processor.js   # fix null ref issue

git add src/payments/processor.js
git commit -m "fix(payments): prevent null ref when order has no shipping address

Orders with digital products have no shippingAddress. Added null guard.
Fixes production incident INCIDENT-1087."

# Step 6: Push hotfix and create PR
git push origin hotfix/INCIDENT-1087-payment-500
# (Create PR on GitHub, get fast approval, merge to main)

# Step 7: Return to feature work
git switch feature/stripe-v2
git stash pop   # reapply your in-progress work

git log --oneline -4
# f1e2d3 feat: refactor payment processor for Stripe v2
# (etc.)

# Step 8: Rebase feature onto latest main (so hotfix is included)
git rebase main
# (Topic 11 covers rebase in depth)

git log --oneline -5
# Now feature/stripe-v2 includes the hotfix
```

**Why branching made this smooth:**
- `main` was NEVER touched during the feature work
- The hotfix was surgically applied to production-state code, not half-refactored code
- Returning to feature work was trivial (stash + switch)

---

## RULE 8 — Mistakes, Root Causes, and Fixes

### Mistake 1: Worked on Main Instead of a Feature Branch

**Symptom:** You made 4 commits directly to `main`. The team uses branch protection
and your push gets rejected. Or worse, you pushed to `main` and your team is angry.

**Root cause:** Forgot to create a feature branch at the start of the work session.

**Fix (commits not yet pushed):**
```bash
# Get the SHA where main SHOULD have stayed
git log --oneline main
# abc123 last valid main commit
# (your 4 commits are on top)

# Create a branch pointing to your current work
git switch -c feature/my-work
# Now feature/my-work has your 4 commits

# Reset main to where it should be
git switch main
git reset --hard abc123
# main is back to the correct state

# Your 4 commits are safe on feature/my-work
```

**Fix (commits already pushed to shared main — more serious):**
```bash
# Option A: Create a branch from the current state, then revert main
git switch -c rescue/my-accidental-work  # save the commits
git push origin rescue/my-accidental-work

# Reset local main and push (requires force push — coordinate with team!)
git switch main
git reset --hard origin/main~4  # or the correct SHA before your commits
git push --force-with-lease origin main

# This is disruptive — communicate with team FIRST
```

**Prevention:** Make branch creation the FIRST thing you do when starting any work.
Many developers set up a Git alias:
```bash
git config --global alias.start '!f() { git switch main && git pull && git switch -c "$1"; }; f'
git start feature/my-work   # pulls latest main, creates branch, switches
```

---

### Mistake 2: Deleted a Branch Before Merging

**Symptom:**
```bash
git branch -D feature/important-work
# Oops — had 10 commits not yet merged
```

**Root cause:** Used `-D` (force delete) or deleted with `-d` after somehow passing the
merge check.

**Fix:** The commits still exist as orphans in `.git/objects/` for 30 days. Find them via
reflog:

```bash
git reflog | grep "feature/important-work"
# abc1234 HEAD@{12}: checkout: moving from feature/important-work to main
# def5678 refs/heads/feature/important-work@{0}: ...

# Recreate the branch at the last known SHA
git branch feature/important-work abc1234

# Or directly:
git switch -c feature/important-work HEAD@{12}
```

If the reflog entry is gone (> 30 days), you can try `git fsck --unreachable | grep commit`
to find dangling commits, then inspect them manually.

**Prevention:** Use `-d` (safe delete) not `-D` unless you KNOW the branch is disposable.
`-d` refuses if the branch hasn't been merged.

---

### Mistake 3: Branch Name With a Space or Illegal Character

**Symptom:**
```bash
git switch -c feature/my new feature
# fatal: 'feature/my' is not a valid branch name
# (shell split the argument at the space)
```

**Root cause:** Shell word splitting. "feature/my new feature" becomes 3 arguments.

**Fix:**
```bash
git switch -c "feature/my-new-feature"   # Use hyphens, not spaces
git switch -c feature/my-new-feature     # No quotes needed for this
```

**Illegal characters in branch names:**
```
CANNOT use:  space, ~, ^, :, ?, *, [, \, .., @{, consecutive dots
CANNOT start with: /  or  -
CANNOT end with: .lock, /
CANNOT be: @  or  HEAD

CONVENTION: use hyphens for spaces, lowercase, no dots
  ✅ feature/user-authentication-v2
  ✅ fix/JIRA-142-null-pointer
  ❌ feature/user auth
  ❌ feature/v1.2.3   (dots technically work but confuse with tags)
  ❌ fix/.hidden
```

---

### Mistake 4: Switched Branches with Uncommitted Changes and Lost Track of Them

**Symptom:**
```bash
# You have changes on feature/auth
# You switch to main — Git LETS YOU (no conflict)
# You forget your changes came with you
# You commit on main, accidentally including your feature changes
```

**Root cause:** Git silently carries non-conflicting uncommitted changes across branch
switches. This is convenient but dangerous if you forget.

**Prevention:**
```bash
# ALWAYS check status before switching
git status
# If dirty, explicitly stash or commit BEFORE switching:
git stash    # saves changes, clean working directory
git switch main

# Or set Git to refuse any switch if dirty:
git config --global advice.detachedHead true  # not directly what you want
# No single config prevents this universally — discipline is the fix
```

---

### Mistake 5: Can't Switch Because of Staged Changes That Conflict

**Symptom:**
```bash
git switch feature/auth
# error: Your local changes to the following files would be overwritten by checkout:
#         src/config.js
# Please commit your changes or stash them before you switch branches.
```

**You need to switch RIGHT NOW but don't want to commit:**
```bash
# Option A: Stash (recommended)
git stash push -m "wip: config changes in progress"
git switch feature/auth
# Later: git stash pop (back on original branch)

# Option B: Stash only specific files
git stash push src/config.js -m "config changes"
git switch feature/auth

# Option C: Switch and BRING the changes with you (3-way merge)
git switch --merge feature/auth
# Git attempts to merge your local changes into the new branch's version
# May still result in conflicts — then you resolve them
```

---

## RULE 10 — Mental Model Checkpoint

**1.** What is a Git branch, physically? Where is it stored? What does it contain?
     How many bytes is it?

**2.** You run `git branch feature/auth`. What file is created? What is written into it?
     Does HEAD change? Does the working directory change?

**3.** You run `git switch feature/auth` and the terminal shows files appear and disappear
     in your working directory. What exact steps did Git perform internally? Which THREE
     things changed?

**4.** What is detached HEAD? What does `.git/HEAD` contain in this state vs normal?
     What happens to commits you make in detached HEAD state?

**5.** `git branch -d feature/auth` refuses with "not fully merged." What is Git actually
     checking? How does Git determine if a branch is "fully merged"?

**6.** You delete a branch with `git branch -D feature/work`. Are the commits gone?
     How would you recover them within 30 days? After 30 days?

**7.** What is the `packed-refs` file? Why does it exist? Which takes precedence:
     a loose ref file in `refs/heads/` or an entry in `packed-refs`?

**8.** What's the difference between `git switch` and `git checkout` for switching
     branches? Why was `git switch` introduced?

**9.** You have uncommitted changes on main. You run `git switch feature/auth`.
     Git lets you switch without error. Where are your changes now?

**10.** What is an orphan branch? How do you create one? Name a real use case for it.

---

## RULE 11 — Connections to Previous Topics

**From Topic 01** — `.git/` directory:
- A branch is a file in `.git/refs/heads/` — you can verify ANY branch by opening the
  file in a text editor. The entire branch concept is just file naming.
- `packed-refs` is another file in `.git/` that consolidates many refs for performance.

**From Topic 03** — The DAG and refs:
- Creating a branch = adding a new named pointer into the existing DAG
- The DAG is NOT duplicated — branches share all commit objects
- Branch deletion = removing a pointer. The DAG nodes (commits) stay until GC.
- `git log branch1..branch2` from Topic 03 uses branch names as DAG navigation anchors

**From Topic 05** — The Three Trees:
- `git switch` moves ALL THREE trees: HEAD changes to new branch's commit,
  Index changes to that commit's tree, Working directory changes to match
- The dirty-working-directory protection is exactly the Three Trees safety check:
  "would switching overwrite uncommitted changes in the WD or Index?"

**From Topic 06** — `git commit`:
- After each `git commit`, Git writes a new SHA to the CURRENT branch file
- This is HOW branches grow: one 41-byte file write per commit (overwriting the old SHA)
- HEAD never changes during a commit — it stays pointing at the same branch name;
  the branch file is what moves

---

## RULE 13 — When Would I Use This at Work?

### Scenario A: The "Branch from the Right Base" Rule

A common team rule: **always branch from the latest `main`** (or `develop`).

```bash
# Wrong way (branch from a stale local copy):
git switch -c feature/new-endpoint   # main is 3 days old locally

# Right way:
git switch main
git pull origin main           # update to latest
git switch -c feature/new-endpoint  # now branching from current production state
```

Why? If your feature branch is based on an old `main`, you'll encounter MORE merge
conflicts when the PR is opened. The older your branch base, the more divergence has
accumulated.

---

### Scenario B: Branch List as a Sprint Health Monitor

```bash
# See all branches sorted by last activity
git branch -v --sort=-committerdate | head -20

# Feature branches untouched for > 1 week?
git for-each-ref --sort=-committerdate refs/heads \
  --format="%(refname:short) %(committerdate:relative)" | head -20
# output:
# feature/payment-v2       3 hours ago
# feature/notifications    2 days ago
# feature/user-import      9 days ago   ← stale! needs attention
# feature/old-experiment   3 weeks ago  ← probably dead, delete it
```

---

### Scenario C: Branch Protection in CI/CD

Your team's GitHub config:
- `main` → requires PR + 2 approvals + all CI checks pass → deploys to production
- `release/**` → requires PR + 1 approval → deploys to staging
- `feature/**` → no push restrictions, only tests run

```bash
# Your feature work:
git switch -c feature/PROJ-234-add-dark-mode
# (work, commit, push)
git push origin feature/PROJ-234-add-dark-mode
# CI runs tests only — no deployment

# When ready, open PR to main
# GitHub enforces 2 approvals + all checks
# On merge: production deploys automatically
```

The branch naming convention (`feature/**`) is what the CI matches on.

---

### Scenario D: Maintaining Multiple Release Trains

```
main:       v3.x development (cutting edge)
release/2.x: LTS branch, receives only security patches
release/1.x: End-of-life, read-only

A critical CVE is found in all versions.
```

```bash
# Fix on main
git switch main
git switch -c fix/CVE-2026-1234-sql-injection
# ... fix ...
git commit -m "fix: prevent SQL injection in user search (CVE-2026-1234)"
# PR → main

# Backport to 2.x
git switch release/2.x
git switch -c fix/CVE-2026-1234-2.x
git cherry-pick <fix-commit-sha>  # cherry-pick the exact fix commit
# PR → release/2.x

# Branches keep the fix isolated per release line
```

This is why cherry-pick (Topic 12) and clean, atomic commits (Topic 07) are essential —
cherry-picking a monolithic commit is painful.

---

## RULE 12 — Quick Reference Card

### Create Branches

| Command | What it does |
|---------|-------------|
| `git branch <name>` | Create branch at current HEAD (no switch) |
| `git branch <name> <sha>` | Create branch at specific commit |
| `git branch <name> main` | Create branch at tip of main |
| `git switch -c <name>` | Create + switch to new branch |
| `git switch -c <name> <sha>` | Create from SHA + switch |
| `git checkout -b <name>` | Legacy: create + switch |
| `git switch --orphan <name>` | Create orphan branch (no history) |

### Switch Branches

| Command | What it does |
|---------|-------------|
| `git switch <branch>` | Switch to branch |
| `git switch -` | Switch to previous branch |
| `git switch --detach <sha>` | Detach HEAD at commit |
| `git checkout <branch>` | Legacy: switch to branch |
| `git checkout -` | Legacy: switch to previous |
| `git checkout abc1234` | Legacy: detach at commit |

### Inspect Branches

| Command | What it does |
|---------|-------------|
| `git branch` | List local branches |
| `git branch -v` | List with last commit |
| `git branch -vv` | List with tracking info |
| `git branch -a` | List local + remote-tracking |
| `git branch -r` | List remote-tracking only |
| `git branch --merged` | Branches merged into HEAD |
| `git branch --no-merged` | Branches NOT merged into HEAD |
| `git branch --contains <sha>` | Branches containing a commit |
| `git branch --sort=-committerdate` | Sort by most recent activity |
| `cat .git/HEAD` | See what HEAD points to |
| `cat .git/refs/heads/<name>` | See raw branch SHA |

### Delete and Rename

| Command | What it does |
|---------|-------------|
| `git branch -d <name>` | Delete (safe — only if merged) |
| `git branch -D <name>` | Force delete (unmerged OK) |
| `git branch -m <old> <new>` | Rename branch |
| `git branch -M <old> <new>` | Force rename |
| `git push origin --delete <name>` | Delete remote branch |

### Manage Tracking

| Command | What it does |
|---------|-------------|
| `git branch -u origin/<name>` | Set upstream for current branch |
| `git branch --unset-upstream` | Remove upstream tracking |
| `git branch --set-upstream-to=origin/<name>` | Same as -u, verbose form |

---

## Summary

A Git branch is one of the simplest concepts in software engineering: a 41-byte plain
text file containing a commit SHA. The entire power of branching comes from:

1. **Cheapness**: Creating a branch is writing 41 bytes. There is no overhead.
2. **Isolation**: All commits made on a branch stay on that branch until merged.
3. **The Three Trees guarantee**: Switching branches atomically updates HEAD, Index,
   and Working Directory to the target branch's snapshot — Git prevents data loss by
   refusing to overwrite uncommitted changes.

The key mental model: **branches don't contain commits. They point to commits.** The
commits exist in the object database independent of any branch. A branch is just a
name for "start traversing the DAG from here."

This is why branch creation is fast, branch deletion is non-destructive, and the same
commit can be reachable from multiple branches simultaneously. The DAG is the truth;
branches are just named entry points.

*Last Updated: Topic 08 — Branching — Internals, Creation, Switching, Deletion — ✅ Completed.*
