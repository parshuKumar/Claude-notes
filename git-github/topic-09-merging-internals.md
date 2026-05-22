# Topic 09 — Merging — Fast-Forward, 3-Way, Recursive, What Happens in `.git/`
### The Complete Internal Mechanics of Every Type of Git Merge

---

## Connection to Previous Topics

- **Topic 02** showed you that a commit object has a `parent` field. A merge commit has
  TWO `parent` fields — that's the only structural difference between a regular commit
  and a merge commit. Today you'll watch that two-parent commit get created.
- **Topic 03** defined the DAG. Merging is what JOINS two diverged branches of the DAG
  back together. The DAG can diverge (branch) and converge (merge) — today is convergence.
- **Topic 05** showed the Three Trees. Merging changes all three: HEAD moves, Index is
  rebuilt from the merged tree, Working Directory is updated to match.
- **Topic 08** showed that branches are just 41-byte pointer files. A merge moves one of
  those pointer files (the current branch) to point at the new merge commit.

---

## RULE 1 — ELI5 Anchor

### The River Confluence Analogy

Imagine two rivers: the **Main River** and the **Feature River**. The Feature River
branched off from the Main River at some point. Both rivers have been flowing (commits
being added). Now you want to combine them back.

**Fast-Forward** = The Feature River has been flowing, but the Main River hasn't moved
at all since they separated. The confluence is trivial — you just extend the Main River
to follow the Feature River's path. No mixing required. The Main River's pointer just
jumps to where the Feature River currently ends.

**3-Way Merge** = Both rivers have been flowing since they separated. To combine them,
you must mix the waters at the junction point. The "3-way" part comes from the three
reference points:
1. **The fork point** (where they split — the "base")
2. **The Main River's current end** (your current commit — "ours")
3. **The Feature River's current end** (the branch being merged — "theirs")

Git compares each file against all three versions to figure out which changes came from
which river, and whether any changes conflict.

**Merge Conflict** = Both rivers brought something to the same spot (the same lines of
the same file). Git can't automatically decide which to keep. It stops and asks you.

---

## RULE 2 — Physical Architecture First

### `.git/` During and After a Merge

**Before any merge:**
```
.git/
├── HEAD                    ← "ref: refs/heads/main\n"
├── index                   ← reflects main's tree
├── objects/
│   ├── (all existing objects)
└── refs/
    ├── heads/
    │   ├── main            ← "C3_sha\n"
    │   └── feature/auth    ← "F2_sha\n"
```

**During a conflicted merge (in-progress state):**
```
.git/
├── HEAD                    ← "ref: refs/heads/main\n"  (unchanged)
├── MERGE_HEAD              ← "F2_sha\n"  ← SHA of the branch being merged
├── MERGE_MSG               ← "Merge branch 'feature/auth'\n"  ← default commit msg
├── MERGE_MODE              ← "" (empty for normal merge)
├── index                   ← STAGE NUMBERS are active!
│                              stage 1 = common ancestor (base) version
│                              stage 2 = ours (main) version
│                              stage 3 = theirs (feature/auth) version
│                              stage 0 = resolved files (no conflict)
├── objects/
│   ├── (existing objects unchanged)
```

**After a SUCCESSFUL merge (new merge commit created):**
```
.git/
├── HEAD                    ← "ref: refs/heads/main\n"  (unchanged)
│                              (MERGE_HEAD, MERGE_MSG, MERGE_MODE: DELETED)
├── index                   ← reflects merged tree (all stage 0)
├── objects/
│   ├── XX/
│   │   └── YY...           ← NEW: merged root tree object
│   ├── AA/
│   │   └── BB...           ← NEW: merge commit object (TWO parent lines)
└── refs/
    ├── heads/
    │   ├── main            ← UPDATED: "AA_BB_sha\n"  ← now points to merge commit
    │   └── feature/auth    ← UNCHANGED: still "F2_sha\n"
```

```bash
# PROVE IT: See merge state files during a conflict
# (After starting a conflicting merge):
ls .git/MERGE_HEAD .git/MERGE_MSG .git/MERGE_MODE
cat .git/MERGE_HEAD   # Shows the SHA of the branch being merged
cat .git/MERGE_MSG    # Shows the default merge commit message

# See the index stage numbers during conflict
git ls-files --stage
# Shows: stage 1, 2, 3 entries for conflicted files
```

---

## TYPE 1: Fast-Forward Merge

### What Fast-Forward Is

A fast-forward merge happens when:
- The branch being merged (`feature/auth`) is a **direct linear descendant** of the
  current branch (`main`)
- In other words: ALL of main's commits are already in feature/auth's history
- There is nothing to "combine" — the feature branch just extended main

```
BEFORE:
  C1 ──► C2 ──► C3    ← main
                  └──► F1 ──► F2    ← feature/auth

  Note: C3 is the direct parent of F1. main is an ancestor of feature/auth.
```

### Exact Data Flow: Fast-Forward Merge

```
COMMAND: git merge feature/auth  (on main, which is at C3)

STEP 1 — Find merge base
  Git computes merge-base(main, feature/auth)
  Traverses commit graph backward from both tips
  Finds: C3 (the common ancestor)
  Discovers: C3 IS main's current tip → main is directly behind feature/auth
  Conclusion: FAST-FORWARD is possible

STEP 2 — Update main's pointer
  READS:    .git/refs/heads/feature/auth → "F2_sha\n"
  WRITES:   .git/refs/heads/main → "F2_sha\n"
  (The branch file is simply overwritten with F2's SHA)

STEP 3 — Update HEAD (since HEAD → main, HEAD "moves" too)
  .git/HEAD still says "ref: refs/heads/main"
  But now main points to F2 — so HEAD effectively moved to F2

STEP 4 — Update Index and Working Directory
  READS:    F2's tree (from the commit object)
  UPDATES:  .git/index to reflect F2's tree
  UPDATES:  Working directory to match F2's files

NO NEW COMMIT OBJECT IS CREATED.
NO NEW TREE OBJECT IS CREATED (F2's tree is reused directly).
```

**The `.git/` state AFTER fast-forward:**
```
BEFORE:
  main         → C3
  feature/auth → F2

AFTER git merge feature/auth:
  main         → F2   ← pointer moved forward
  feature/auth → F2   ← unchanged
```

```bash
# PROVE IT: Perform a fast-forward merge and verify
git log --oneline --graph main feature/auth
# * F2 (feature/auth) latest feature commit
# * F1 first feature commit
# * C3 (main) main's latest

git merge feature/auth
# Updating d8329fc..a4b5c6d
# Fast-forward            ← Git tells you it was fast-forward
#  auth.js | 50 +++...

# Now both branches point to the same commit
git rev-parse main
git rev-parse feature/auth
# Same SHA!

git log --oneline
# F2 (HEAD → main, feature/auth)
# F1
# C3
# (No new merge commit appeared in history)
```

---

## RULE 4 — ASCII Diagram: Fast-Forward

```
BEFORE merge:
  C1 ──► C2 ──► C3                          Objects in .git/objects/:
                  │   ← main (C3 sha)          C1, C2, C3, F1, F2 blobs/trees
                  └──► F1 ──► F2
                                │ ← feature/auth (F2 sha)
                                │
                              HEAD (via main)

AFTER git merge feature/auth (fast-forward):
  C1 ──► C2 ──► C3 ──► F1 ──► F2
                                │ ← main (F2 sha)    ← MOVED
                                │ ← feature/auth (F2 sha)  ← unchanged
                              HEAD (via main)

Changes to .git/:
  BEFORE: .git/refs/heads/main = "C3_sha"
  AFTER:  .git/refs/heads/main = "F2_sha"
  NO new objects created. Just a 41-byte file overwrite.
```

---

## TYPE 2: True Merge (3-Way Merge)

### When 3-Way Merge Happens

A true merge is needed when:
- BOTH branches have new commits since they diverged
- main has commits that feature/auth doesn't have
- feature/auth has commits that main doesn't have
- The branch being merged is NOT a direct linear ancestor of the current branch

```
BEFORE:
  C1 ──► C2 ──► C3 ──► C4    ← main (has C4 that feature doesn't)
                  │
                  └──► F1 ──► F2    ← feature/auth (has F1,F2 that main doesn't)

Merge base: C3 (the common ancestor — where the paths diverged)
```

### The 3-Way Merge Algorithm

Git uses THREE snapshots to compute the merge result:

```
BASE (C3):   The last common ancestor. What the files looked like at the split point.
OURS (C4):   The current branch (main). Changes made since the split on main.
THEIRS (F2): The branch being merged. Changes made since the split on feature/auth.
```

**For each file, Git applies this logic:**

| BASE | OURS | THEIRS | Result |
|------|------|--------|--------|
| Same | Same | Same | Unchanged |
| X | X | Y | Take THEIRS (only theirs changed it) |
| X | Y | X | Take OURS (only ours changed it) |
| X | Y | Y | Take either (both made same change) |
| X | Y | Z | **CONFLICT** — both changed, differently |
| exists | exists | deleted | **CONFLICT** (usually) or THEIRS deleted it |
| exists | deleted | exists | **CONFLICT** (usually) or OURS deleted it |
| not exist | added | not exist | Take OURS addition |
| not exist | not exist | added | Take THEIRS addition |

The key rule: **if only ONE side changed a file from the base, that change is taken
automatically. If BOTH sides changed the same part of a file, that's a conflict.**

### Exact Data Flow: 3-Way Merge

```
COMMAND: git merge feature/auth  (on main, which is at C4)

STEP 1 — Find merge base
  READS:    Commit graph (traverses backwards from C4 and F2)
  FINDS:    C3 = merge-base(C4, F2) — the last common ancestor
  COMPUTES: C3's tree = base_tree
            C4's tree = our_tree
            F2's tree = their_tree

STEP 2 — 3-way diff all files
  For EACH file in base_tree ∪ our_tree ∪ their_tree:
    Compare file version in base, ours, theirs
    Determine: auto-resolve or conflict

STEP 3A — Auto-resolvable files
  WRITES:   Merged content to working directory
  UPDATES:  .git/index entry (stage 0) for resolved files
  CREATES:  New blob objects if content changed

STEP 3B — Conflicted files
  WRITES:   Conflict markers to working directory file:
              <<<<<<< HEAD
              (our version of the conflicting lines)
              =======
              (their version of the conflicting lines)
              >>>>>>> feature/auth
  WRITES:   .git/index entries at stages 1, 2, 3:
              stage 1: base blob SHA
              stage 2: our blob SHA
              stage 3: their blob SHA

STEP 4 — If NO conflicts: Finalize the merge
  COMPUTES: Merged tree (new tree objects, bottom-up)
  WRITES:   New tree objects to .git/objects/

STEP 5 — Create merge commit object
  Content:
    tree <merged_root_tree_sha>
    parent <C4_sha>              ← first parent: HEAD (ours)
    parent <F2_sha>              ← second parent: THEIRS (the merged branch)
    author ...
    committer ...

    Merge branch 'feature/auth'
  WRITES:   New commit object to .git/objects/

STEP 6 — Update main pointer
  WRITES:   .git/refs/heads/main → <merge_commit_sha>

STEP 7 — Clean up merge state files
  DELETES:  .git/MERGE_HEAD
  DELETES:  .git/MERGE_MSG
  DELETES:  .git/MERGE_MODE
```

---

## RULE 4 — ASCII Diagram: 3-Way Merge

```
BEFORE merge:
  C1 ──► C2 ──► C3 ──► C4    ← main (HEAD)
                  │
                  └──► F1 ──► F2    ← feature/auth

  Merge base = C3

3-WAY DIFF:
                BASE (C3)      OURS (C4)      THEIRS (F2)
  server.js:   version A   →  version B   vs  version C
  auth.js:     not exist   →  not exist   vs  version X    ← take X (only theirs added)
  styles.css:  version P   →  version Q   vs  version P    ← take Q (only ours changed)
  config.js:   version M   →  version N   vs  version O    ← CONFLICT

AFTER git merge feature/auth (no conflicts):

  C1 ──► C2 ──► C3 ──► C4 ──────────────────► M1    ← main (HEAD)
                  │              │               ▲
                  │              │         merge commit
                  └──► F1 ──► F2 ──────────────┘
                                │ ← feature/auth (unchanged)

  M1 = merge commit with:
    parent1: C4 (ours)
    parent2: F2 (theirs)
    tree: merged result

.git/ changes:
  CREATED: new blob(s) for merged file content
  CREATED: new tree objects (merged directory structure)
  CREATED: merge commit M1 (two-parent commit)
  UPDATED: .git/refs/heads/main → M1_sha
  DELETED: .git/MERGE_HEAD, MERGE_MSG, MERGE_MODE
```

```bash
# PROVE IT: Inspect the merge commit
git cat-file -p HEAD
# tree <sha>
# parent <C4_sha>       ← first parent (was HEAD before merge)
# parent <F2_sha>       ← second parent (the merged branch)
# author ...
# committer ...
#
# Merge branch 'feature/auth'

# Verify two parents
git log --oneline --graph
# *   abc123 (HEAD → main) Merge branch 'feature/auth'
# |\                         ← the "fork" shows two parents
# | * def456 (feature/auth) F2 - latest feature commit
# | * ghi789 F1 - first feature commit
# * jkl012 C4 - latest main commit
# * mno345 C3 - common ancestor

# Navigate parents
git cat-file -p HEAD^1   # First parent (was main before merge)
git cat-file -p HEAD^2   # Second parent (was feature/auth)
```

---

## RULE 9 — Byte-Level: The Merge Commit Object

A merge commit object is identical in format to a regular commit, except it has TWO
(or more) `parent` lines:

```
commit 412\0
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
parent a4b5c6d7e8f9a0b1c2d3e4f5g6h7i8j9k0l1m2n3
parent 1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0
author Parsh Khandelwal <parsh@example.com> 1716371200 +0530
committer Parsh Khandelwal <parsh@example.com> 1716371200 +0530

Merge branch 'feature/auth'
```

The format rules:
- `parent` lines appear in the order: first parent = what HEAD was, second parent =
  what was merged in
- For octopus merges (3+ branches), more `parent` lines are added
- The commit's `tree` is the fully merged tree (built from the 3-way merge result)

```bash
# PROVE IT: Count parent lines in a merge commit
git cat-file commit HEAD | grep "^parent"
# parent <sha1>
# parent <sha2>   ← two lines for a regular 2-way merge

# Navigate: HEAD^1 = first parent, HEAD^2 = second parent
git log --oneline HEAD^1 | head -3    # history via first parent (main's history)
git log --oneline HEAD^2 | head -3    # history via second parent (feature branch)

# Show only "mainline" history (first parents only)
git log --oneline --first-parent
# Shows only commits that were on main — skips commits from merged branches
```

---

## Deep Dive: Conflict Markers and the Index During Conflict

### The Conflict Marker Format

When a conflict occurs, Git writes the conflicted file to the working directory with
special markers:

```
<<<<<<< HEAD
function getUserById(id) {
  if (!id) throw new Error('ID required');
  return db.users.find(id);
}
=======
function getUserById(id) {
  const user = db.users.find(id);
  if (!user) return null;
  return user;
}
>>>>>>> feature/auth
```

**What each section means:**

```
<<<<<<< HEAD                   ← Start of conflict; HEAD = current branch (main)
(our version — what main has)
=======                        ← Divider between versions
(their version — what feature/auth has)
>>>>>>> feature/auth           ← End of conflict; branch name being merged
```

With `merge.conflictStyle=diff3` (configured in Topic 04):
```
<<<<<<< HEAD
(our version)
||||||| merged common ancestors
(BASE version — what the common ancestor had)
=======
(their version)
>>>>>>> feature/auth
```

The `diff3` style adds the BASE version in the middle. This shows you:
- What the file looked like at the split point (base)
- What main changed it to (ours)
- What feature/auth changed it to (theirs)

This is extremely helpful for understanding WHO changed WHAT and WHY.

### The Index Stage Numbers During Conflict

When you run `git ls-files --stage` during a conflict:

```
100644 aaa111... 0    .gitignore         ← stage 0: resolved (no conflict)
100644 bbb222... 1    src/config.js      ← stage 1: BASE version
100644 ccc333... 2    src/config.js      ← stage 2: OUR version (HEAD/main)
100644 ddd444... 3    src/config.js      ← stage 3: THEIR version (feature/auth)
100644 eee555... 0    src/server.js      ← stage 0: resolved (no conflict)
```

**Stage meanings:**
```
Stage 0: Normal (not in conflict). The file's final committed version.
Stage 1: The BASE version — what the common ancestor had.
Stage 2: OUR version — what the current branch (HEAD) has.
Stage 3: THEIR version — what the branch being merged has.
```

When ALL three stages (1, 2, 3) exist for a file, that file is in CONFLICT.
When you resolve a conflict (`git add <file>`), stages 1, 2, 3 are removed and
replaced with a single stage 0 entry (the resolved content).

```bash
# PROVE IT: See stage numbers during a conflict
git merge feature/auth   # (trigger a conflict)

# See the conflicted index
git ls-files --stage | grep "config.js"
# 100644 aaa... 1    src/config.js
# 100644 bbb... 2    src/config.js
# 100644 ccc... 3    src/config.js

# Read each version independently:
git show :1:src/config.js   # base (ancestor) version
git show :2:src/config.js   # our (main) version
git show :3:src/config.js   # their (feature/auth) version

# Check which files are conflicted:
git diff --name-only --diff-filter=U
# (U = Unmerged)

# Or:
git status
# both modified:   src/config.js
```

---

## Resolving a Merge Conflict — Step by Step

### The Manual Resolution Workflow

```bash
# Step 1: Start the merge (conflict occurs)
git merge feature/auth
# CONFLICT (content): Merge conflict in src/config.js
# Automatic merge failed; fix conflicts and then commit the result.

# Step 2: See which files have conflicts
git status
# On branch main
# You have unmerged paths.
#   (fix conflicts and run "git commit")
# Unmerged paths:
#   (use "git add <file>..." to mark resolution)
#         both modified:   src/config.js

# Step 3: Open the conflicted file
# Edit src/config.js — remove conflict markers, choose the right content

# Option A: Keep ours
git checkout --ours src/config.js

# Option B: Keep theirs
git checkout --theirs src/config.js

# Option C: Manual edit — combine both intelligently
vim src/config.js
# Remove <<<<<<, =======, >>>>>> markers
# Write the correct merged version

# Step 4: Mark as resolved
git add src/config.js
# This removes stages 1, 2, 3 from index and adds stage 0

# Step 5: Verify no more conflicts
git status
# All conflicts fixed but you are still merging.
#   (use "git commit" to conclude merge)

# Step 6: Commit the merge
git commit
# Editor opens pre-filled with .git/MERGE_MSG content:
# "Merge branch 'feature/auth'"
# You can keep it or enhance it

# Step 7: The merge commit is created, MERGE_HEAD/MSG/MODE deleted
```

### Aborting a Merge

```bash
# At any point during a conflicted merge, you can abort
git merge --abort

# This restores:
#   - Working directory to pre-merge state
#   - Index to pre-merge state
#   - Deletes MERGE_HEAD, MERGE_MSG, MERGE_MODE
# As if git merge was never run

# PROVE IT: State is fully restored
git status
# On branch main
# nothing to commit, working tree clean
```

### The `--continue` Flag

```bash
# After resolving ALL conflicts and staging resolved files:
git merge --continue
# Creates the merge commit using .git/MERGE_MSG as the message
# Equivalent to: git commit (during an in-progress merge)
```

---

## Deep Dive: Merge Strategies

### Strategy 1: `ort` (Default since Git 2.34)

The **ort** strategy (Ostensibly Recursive's Twin) is the current default merge strategy.
It replaced "recursive" as the default in Git 2.34.

```bash
git merge feature/auth                    # uses ort by default
git merge -s ort feature/auth            # explicit
```

**How ort works:**
- Finds the best common ancestor using a virtual merge of ancestors
- Handles criss-cross merges (where two branches have each merged from each other)
- More accurate rename detection
- Faster than recursive for large repos

### Strategy 2: `recursive` (Default before Git 2.34)

```bash
git merge -s recursive feature/auth
```

The recursive strategy handles criss-cross merges by recursively merging the common
ancestors together to produce a virtual base. This is what Git used by default for
over 15 years.

**Criss-cross merge example:**

```
  A ──► B ──► C ──► (merge1)    ← branch-x
        │     │         ▲
        │     └─────────┤
        │               │
        └──► D ──► E ──► (merge2)    ← branch-y

When branch-y tries to merge branch-x, there are TWO common ancestors: B and D.
Recursive strategy: merge B and D together to create a "virtual base",
then 3-way merge using that virtual base.
```

### Strategy 3: `ours`

```bash
git merge -s ours feature/auth
```

Ignores ALL changes from the merged branch. Creates a merge commit with two parents
but the tree is IDENTICAL to the current branch's tree. The feature/auth commits
are "absorbed" into history without any of their content being applied.

**Use case:** You want to merge a branch into history (so `--merged` shows it as merged)
but you've already cherry-picked specific changes manually.

### Strategy 4: `octopus`

```bash
git merge branch-a branch-b branch-c   # automatically uses octopus
```

Merges more than 2 branches simultaneously. Creates a merge commit with 3+ parents.

**Limitation:** Cannot handle conflicts. If any conflict exists, octopus refuses.
Used in repositories that want to merge many topic branches simultaneously without
conflicts (common in Linux kernel development).

### Strategy Options (`-X` flags)

These work with `ort` and `recursive` strategies:

```bash
git merge -X ours feature/auth
# In case of conflict: automatically take OUR version
# Not the same as -s ours (which ignores ALL their changes)
# Only applies to CONFLICTED hunks; non-conflicting changes are merged normally

git merge -X theirs feature/auth
# In case of conflict: automatically take THEIR version

git merge -X ignore-space-change feature/auth
# Ignores whitespace changes when comparing lines

git merge -X ignore-all-space feature/auth
# Ignores all whitespace (tabs, spaces, etc.)

git merge -X patience feature/auth
# Uses "patience" diff algorithm for finding differences
# Better at handling large blocks of code being moved

git merge -X diff-algorithm=histogram feature/auth
# Uses histogram diff algorithm (generally best quality)

git merge -X find-renames=85 feature/auth
# Sets rename detection threshold to 85% similarity
```

---

## `git merge --no-ff` — Forcing a Merge Commit

### Why This Matters

Fast-forward merges are "silent" — they leave no trace that a feature branch existed.
The history looks like work was done directly on main:

```
After fast-forward merge:
  C1 ──► C2 ──► C3 ──► F1 ──► F2
                                │ ← main (HEAD)

There's NO indication that F1 and F2 came from a separate branch.
```

With `--no-ff`, even on a fast-forwardable branch, Git creates a merge commit:

```
After git merge --no-ff feature/auth:
  C1 ──► C2 ──► C3 ──────────────────► M1
                  │                     ▲
                  └──► F1 ──► F2 ──────┘
                               │ ← feature/auth (unchanged)

M1 = merge commit preserving the feature branch boundary
```

```bash
git merge --no-ff feature/auth
# Creates a merge commit even if fast-forward was possible
# Editor opens for the merge commit message
```

**Data flow for `--no-ff`:**
```
STEP 1 — Fast-forward check: possible, but IGNORED due to --no-ff
STEP 2 — Create MERGE_HEAD (same as regular 3-way merge flow)
STEP 3 — Build merged tree (in this case, identical to F2's tree)
STEP 4 — Create merge commit:
           tree: F2's tree (unchanged — same content as feature/auth)
           parent1: C3 (main's current commit)
           parent2: F2 (feature/auth's tip)
STEP 5 — Update main pointer to M1
```

**When to use `--no-ff`:**
- Team wants to preserve the fact that each PR/feature was developed on its own branch
- Want `git log --first-parent main` to only show "integration" commits (merges)
- Using a Gitflow-style workflow where branch structure is important history

---

## `git merge --ff-only` — Reject if Not Fast-Forward

```bash
git merge --ff-only feature/auth
```

If the merge CANNOT be done as a fast-forward (because main has diverged), Git refuses:
```
fatal: Not possible to fast-forward, aborting.
```

**Use case:** Enforcing a "rebase before merge" policy. Your CI script uses `--ff-only`
to ensure all feature branches are rebased onto current main before merging.

---

## `git merge --squash` — Squash All Feature Commits into One

```bash
git merge --squash feature/auth
```

**What this does:**
1. Computes the merge result (same 3-way merge algorithm)
2. Updates Working Directory and Index with the merged result
3. Does **NOT** create a merge commit
4. Does **NOT** set MERGE_HEAD
5. The result is left STAGED but uncommitted

```
BEFORE:
  C1 ──► C2 ──► C3    ← main
                  └──► F1 ──► F2 ──► F3    ← feature/auth (3 commits)

After git merge --squash feature/auth:
  C1 ──► C2 ──► C3    ← main (unchanged! no new commit yet)
  Working dir + Index = merged state (includes all F1+F2+F3 changes)
  
After git commit -m "feat: add authentication module":
  C1 ──► C2 ──► C3 ──► S1    ← main (S1 = squash commit)
                         │  S1 has ONE parent (C3), NOT F3

  feature/auth's history is COLLAPSED into S1.
  F1, F2, F3 are orphaned from main's perspective.
```

**Key difference from `--no-ff`:**

| Approach | Main history | Feature history visible? | Revert granularity |
|----------|-------------|--------------------------|-------------------|
| Fast-forward | C1→C2→C3→F1→F2→F3 | No (merged in) | Per original commit |
| --no-ff | C1→C2→C3→M1 (with F1,F2,F3 as side chain) | Yes (parallel chain) | Per feature commit |
| --squash | C1→C2→C3→S1 | No (squashed) | Only the whole feature |

---

## RULE 5 — Hands-On Proof: Full Merge Walkthrough

### Setup: Create a Diverged History

```bash
mkdir merge-demo && cd merge-demo
git init
git config user.name "Parsh" && git config user.email "parsh@example.com"

# Create initial commit on main
echo "const app = require('./app');" > index.js
echo "module.exports = { version: '1.0' };" > app.js
git add . && git commit -m "chore: initial project setup"

# Create feature branch (shared base: this commit)
git switch -c feature/auth

# Feature branch: add auth
echo "const jwt = require('jsonwebtoken');" > auth.js
echo "// Route: POST /login" >> app.js   # modifies app.js too
git add . && git commit -m "feat: add auth module"
echo "module.exports = { verifyToken: () => {} };" >> auth.js
git add . && git commit -m "feat: add verifyToken to auth module"

# Go back to main and add a SEPARATE change
git switch main
echo "const db = require('./db');" >> index.js    # modifies index.js
echo "module.exports.dbVersion = '2.0';" >> app.js  # modifies app.js too!
git add . && git commit -m "feat: add database module connection"

# Now we have a diverged history:
git log --oneline --graph --all
# * abc123 (HEAD → main) feat: add database module connection
# | * def456 (feature/auth) feat: add verifyToken to auth module
# | * ghi789 feat: add auth module
# |/
# * jkl012 chore: initial project setup
```

### Perform the 3-Way Merge

```bash
git merge feature/auth
# Auto-merging app.js
# CONFLICT (content): Merge conflict in app.js
# Automatic merge failed; fix conflicts and then commit the result.

# See what Git resolved automatically:
git status
#   both modified:   app.js          ← conflict
#   new file:        auth.js         ← auto-resolved (only theirs added it)

# See the conflict in app.js:
cat app.js
# module.exports = { version: '1.0' };
# <<<<<<< HEAD
# module.exports.dbVersion = '2.0';
# =======
# // Route: POST /login
# >>>>>>> feature/auth

# See stage numbers in index:
git ls-files --stage | grep app.js
# 100644 sha1... 1    app.js   (base version)
# 100644 sha2... 2    app.js   (our main version)
# 100644 sha3... 3    app.js   (their feature version)

# Read each version:
git show :1:app.js    # base
git show :2:app.js    # ours (main)
git show :3:app.js    # theirs (feature/auth)

# Resolve: keep BOTH changes (manual edit)
cat > app.js << 'EOF'
module.exports = { version: '1.0' };
module.exports.dbVersion = '2.0';
// Route: POST /login
EOF

git add app.js

# Check we're clean:
git status
# All conflicts fixed but you are still merging.

# Commit the merge:
git commit
# Opens editor with "Merge branch 'feature/auth'"

# Inspect the result:
git log --oneline --graph
# *   abc (HEAD → main) Merge branch 'feature/auth'
# |\
# | * def feat: add verifyToken to auth module
# | * ghi feat: add auth module
# * | jkl feat: add database module connection
# |/
# * mno chore: initial project setup

# Verify merge commit has two parents:
git cat-file -p HEAD | grep "^parent"
# parent <main-before-merge-sha>
# parent <feature/auth-sha>
```

---

## RULE 6 — Deep Syntax Breakdown: `git merge`

```
git merge  [<branch>]  [--no-ff]  [--ff-only]  [--squash]
│    │       │          │          │             │
│    │       │          │          │             └─ Squash: stage result, no commit
│    │       │          │          └─ --ff-only: fail if not fast-forwardable
│    │       │          └─ --no-ff: always create merge commit
│    │       └─ The branch (or SHA) to merge into current branch
│    └─ subcommand: combine branches
└─ git binary

STRATEGY FLAGS:
  -s <strategy>          ort (default), recursive, resolve, octopus, ours, subtree
  -X <option>            Strategy-specific options (ours, theirs, ignore-space-change, etc.)

COMMIT FLAGS:
  --commit               Force commit (default; only meaningful alongside --no-edit)
  --no-commit            Stop after merge, before committing (inspect before finalizing)
  --edit / -e            Open editor for merge message (default)
  --no-edit              Accept default merge message without opening editor

MESSAGE FLAGS:
  -m <message>           Use this message instead of default "Merge branch 'X'"
  --log[=n]              Include one-line summaries of merged commits in the message
  --no-log               Don't include summaries (default)

CONFLICT HANDLING:
  --abort                Abort and restore pre-merge state
  --continue             Continue after resolving conflicts
  --quit                 Forget the in-progress merge (leave working dir as-is)

MERGE COMMIT LINEAGE:
  --allow-unrelated-histories   Allow merging repos with no common commit
                                 (needed when combining two independent repos)

STAT FLAGS:
  --stat / -stat         Show a diffstat after merging (default)
  --no-stat              Don't show diffstat
  --summary              Same as --stat

PROGRESS:
  --progress             Show progress
  --verbose / -v         Verbose output

VERIFICATION:
  --verify-signatures    Verify GPG signatures on tip of merged branch
                         Refuse to merge if signature is invalid
```

---

## `git merge --no-commit` — Preview Before Finalizing

```bash
git merge --no-commit feature/auth
# Merge is performed (files updated, index updated)
# But STOPS before creating the merge commit
# MERGE_HEAD is still set, merge state is active

# You can now:
git diff --cached          # review what will be committed
git diff HEAD              # see all changes from before merge to now
git log --oneline HEAD..MERGE_HEAD  # see commits being merged in

# If happy:
git commit -m "feat: merge authentication module"

# If not happy (e.g., you want to make manual adjustments):
# Edit files, git add, then commit
# OR abort:
git merge --abort
```

---

## RULE 7 — Beginner Example

### Merging a Feature Branch into Main (Clean Merge)

```bash
mkdir todo-app && cd todo-app
git init

# Initial commit on main
echo "<h1>Todo App</h1>" > index.html
git add . && git commit -m "feat: create initial todo app"

# Branch for adding a dark mode feature
git switch -c feature/dark-mode
echo "body { background: #222; color: #eee; }" > dark.css
git add . && git commit -m "feat: add dark mode stylesheet"
echo "document.body.classList.toggle('dark');" > toggle.js
git add . && git commit -m "feat: add dark mode toggle script"

# Go back to main (no new commits on main — fast-forward possible)
git switch main

# Merge
git merge feature/dark-mode
# Updating abc123..def456
# Fast-forward
#  dark.css   | 1 +
#  toggle.js  | 1 +

# The feature is now in main
ls
# dark.css  index.html  toggle.js

git log --oneline
# def456 (HEAD → main, feature/dark-mode) feat: add dark mode toggle script
# abc123 feat: add dark mode stylesheet
# ghi789 feat: create initial todo app

# Clean up: delete the feature branch
git branch -d feature/dark-mode
```

---

## RULE 7 — Production Example

### The Weekly Integration Merge

You're on a backend team of 6. Sprint ends Friday. Four feature branches must be merged
into `develop` (your integration branch):

```
develop ──► DB1 ──► DB2
              │
              ├──► feature/PROJ-231-user-auth   (3 commits, team A)
              ├──► feature/PROJ-244-notifications (5 commits, team B)
              ├──► feature/PROJ-258-payment-v2  (8 commits, team C)
              └──► feature/PROJ-261-search      (2 commits, team D)
```

```bash
git switch develop
git pull origin develop

# Merge 1: User auth (clean — different files)
git merge --no-ff feature/PROJ-231-user-auth -m "feat: merge user authentication module (#231)"
# Fast-forward possible but --no-ff preserves branch history
git push origin develop

# Merge 2: Notifications (conflict in config.js!)
git merge --no-ff feature/PROJ-244-notifications -m "feat: merge notifications module (#244)"
# CONFLICT in src/config.js
# Both auth and notifications added new config keys

# Use diff3 style to understand the conflict:
cat src/config.js
# <<<<<<< HEAD
# auth: { jwtSecret: process.env.JWT_SECRET }
# ||||||| merged common ancestors  (with diff3)
# (empty — neither existed in base)
# =======
# notifications: { slack: process.env.SLACK_TOKEN }
# >>>>>>> feature/PROJ-244-notifications

# Resolution: keep BOTH (they're different keys)
cat > src/config.js << 'EOF'
auth: { jwtSecret: process.env.JWT_SECRET },
notifications: { slack: process.env.SLACK_TOKEN }
EOF
git add src/config.js
git commit -m "feat: merge notifications module (#244)

Resolved conflict: auth and notifications both added config keys.
Kept both as they are independent configuration sections."
git push origin develop

# Continue with remaining merges...

# After all 4 merges: develop has a visible branch history
git log --oneline --graph develop | head -20
# Shows all 4 feature branches converging into develop
```

---

## RULE 8 — Mistakes, Root Causes, and Fixes

### Mistake 1: Accidentally Merged to the Wrong Branch

**Symptom:** You ran `git merge feature/auth` while on `feature/notifications` instead of
`develop`. Now `feature/notifications` has auth commits it shouldn't.

```bash
# Check what happened
git log --oneline -5
# abc (HEAD → feature/notifications) Merge branch 'feature/auth'
# def feat: add notification endpoint
# ghi (feature/auth) feat: add JWT validation
# ...

# The merge commit is the last one
```

**Fix (if not pushed):**
```bash
# Reset to before the merge (the commit before the merge commit)
git reset --hard HEAD~1
# OR: git reset --hard ORIG_HEAD
# (ORIG_HEAD is set by Git before merges/resets — Topic 12)

# Verify: auth's commits are gone from this branch
git log --oneline -3
# def feat: add notification endpoint (clean)
```

**Fix (if pushed):**
```bash
# Revert the merge commit (safe for shared branches)
git revert -m 1 HEAD
# -m 1: "mainline 1" — tells git revert which parent is the "main" parent
# When reverting a merge commit, you MUST specify which parent to revert to
# 1 = first parent (the branch you were on)
# 2 = second parent (the branch that was merged in)
```

---

### Mistake 2: Merge Conflict Markers Left in Code

**Symptom:** You resolved the conflict but accidentally left `<<<<<<<` or `=======` markers
in the file. Code is committed and potentially deployed.

**Root cause:** Edited the file but didn't remove ALL markers. `git add` doesn't check
for conflict markers.

**Prevention:**
```bash
# Before git commit during merge resolution:
git diff --check
# Reports conflict markers still present in tracked files:
# src/config.js:5: leftover conflict marker

# Or configure Git to always check:
git config --global core.whitespace conflict-marker-size=7
```

**Fix (if committed):**
```bash
# Find files with conflict markers
grep -r "<<<<<<" . --include="*.js" --include="*.ts"

# Fix the files
vim src/config.js

git add src/config.js
git commit -m "fix: remove conflict markers accidentally left in config"
```

---

### Mistake 3: `--squash` Merge, Then Trying to Merge the Same Branch Again

**Symptom:**
```bash
git merge --squash feature/auth   # Squashed and committed as S1
# ... later, feature/auth gets more commits ...
git merge --squash feature/auth   # Merge again
# Results in conflicts or double-application of old changes
```

**Root cause:** `--squash` does NOT record that feature/auth was merged. Git has no record
linking main's S1 to feature/auth. When you merge again, Git tries to merge the ENTIRE
feature/auth history (including the already-squashed parts) again.

**Prevention:**
- After a squash merge, DELETE the source branch immediately
- OR use `--no-ff` instead if you want to merge again later

**Fix:**
```bash
# If already in a mess:
git merge --abort
# Start fresh: delete and recreate the feature branch from the last new commit point
```

---

### Mistake 4: "Refusing to merge unrelated histories"

**Symptom:**
```bash
git merge origin/main
# fatal: refusing to merge unrelated histories
```

**Root cause:** The two repos have no common ancestor. This happens when:
- You ran `git init` locally then tried to merge a remote that was also `git init`-ed
  separately (both started with "initial commit" but they're unrelated DAGs)
- You cloned and the remote was force-pushed to a completely rewritten history

**Fix:**
```bash
git merge --allow-unrelated-histories origin/main
# Creates a merge commit even though the repos have no common ancestor
# Use with caution — you'll need to manually resolve many conflicts
```

---

### Mistake 5: Merge Commit Message Says Nothing Useful

**Symptom:**
```
abc123 Merge branch 'feature/auth'  ← tells you nothing about what was merged
```

**Prevention:**
```bash
# Use -m to provide a meaningful message:
git merge --no-ff feature/PROJ-142-user-auth \
  -m "feat: merge user authentication module

Includes JWT-based auth, login/logout endpoints, and session management.
PR #142. Reviewed by: Priya, Rahul."

# Or use --log to include commit summaries automatically:
git merge --no-ff --log feature/auth
# The merge message will include one-line summaries of merged commits
```

---

## `rerere` — Reuse Recorded Resolution

### What It Is

`rerere` (Reuse Recorded Resolution) teaches Git to remember how you resolved a specific
conflict and automatically apply that resolution the next time the same conflict appears.

This is invaluable when:
- You frequently rebase a long-lived feature branch onto main (each rebase re-encounters
  the same conflicts)
- You're testing multiple merge strategies

```bash
# Enable rerere globally
git config --global rerere.enabled true

# When you resolve a conflict manually, rerere records the resolution in:
# .git/rr-cache/<sha>/
#   preimage  ← the conflicted file with markers
#   postimage ← your resolved version

# Next time the same conflict appears (same base, same ours, same theirs):
# Git auto-applies your resolution!
git status
# (rerere applied existing resolution)
# You still need to git add and git commit

# List recorded resolutions:
git rerere status

# Forget a resolution:
git rerere forget <path>

# Diff between pre-image and resolution:
git rerere diff
```

---

## RULE 10 — Mental Model Checkpoint

**1.** What are the three conditions that determine whether Git can fast-forward instead
of creating a merge commit? Name the exact check Git performs.

**2.** In a 3-way merge, what are the three "ways"? Why are THREE versions needed to
auto-resolve changes — why not just two?

**3.** You merge `feature/auth` into `main`. The merge creates a new commit M1. How many
`parent` lines does M1's commit object have? What SHA does each parent contain?

**4.** During a conflict, you run `git ls-files --stage` and see entries at stages 1, 2,
and 3 for `src/config.js`. What does each stage contain? How do you resolve the conflict?
What does the index look like AFTER you run `git add src/config.js`?

**5.** What is the difference between `git merge --no-ff`, `git merge --ff-only`, and
`git merge --squash`? For each one, describe the resulting commit graph shape.

**6.** After a successful merge commit, which temporary files in `.git/` are deleted?
What were they used for?

**7.** You accidentally committed conflict markers (`<<<<<<<`) into a file. How do you
PREVENT this in the future? Name the specific Git config and command.

**8.** What does `git merge -s ours` do? How is it different from `git merge -X ours`?
Give a real use case where `-s ours` is the right tool.

**9.** What does `git merge --allow-unrelated-histories` do? When would you need it?

**10.** What is `rerere`? Where does it store its data? What problem does it solve?

---

## RULE 11 — Connections to Previous Topics

**From Topic 02** — Git Objects:
- A merge commit is a commit object with TWO `parent` lines — same format, more parents
- The merged tree is a new tree object built from the 3-way merge result
- No objects are ever modified — new ones are always created

**From Topic 03** — The DAG:
- A merge is literally where two branches of the DAG converge into one node
- The merge commit node has in-degree 2 (two parent edges pointing in)
- `git log --first-parent` follows only the first parent edge — used to see "mainline" history
- `HEAD^1` (first parent) and `HEAD^2` (second parent) navigate the DAG around merge commits

**From Topic 05** — The Three Trees:
- After a successful merge: HEAD moves to merge commit, Index rebuilt from merged tree,
  Working Directory updated to match
- During a conflicted merge: Index has stages 1/2/3; WD has conflict markers; HEAD is frozen

**From Topic 08** — Branching:
- After a fast-forward merge: the current branch's 41-byte file is updated to the merged
  branch's SHA — that's ALL that changes
- After a true merge: a new merge commit is created and the current branch's file is
  updated to point to it; the merged branch is UNCHANGED

---

## RULE 13 — When Would I Use This at Work?

### Scenario A: Sprint Merge Day

Every two weeks, your team merges all feature branches into `develop`. You're the senior
engineer coordinating the integration:

```bash
# Preview what each merge will bring:
git log --oneline develop..feature/PROJ-231
# Shows commits that feature/PROJ-231 has but develop doesn't

# Check for likely conflicts before merging:
git merge-tree $(git merge-base develop feature/PROJ-231) develop feature/PROJ-231
# Shows what the merge WOULD produce (without actually doing it)
# If output contains conflict markers: you'll need manual resolution

# Merge all with --no-ff to preserve branch history:
for branch in feature/PROJ-231 feature/PROJ-244 feature/PROJ-258; do
  git merge --no-ff $branch -m "feat: merge $branch"
  if [ $? -ne 0 ]; then
    echo "Conflict in $branch — resolve manually"
    break
  fi
done
```

---

### Scenario B: Long-Running Feature Branch Keeping Up with Main

Your feature branch has been active for 2 weeks. Main has moved forward by 40 commits.
Before opening a PR, you must integrate those changes.

```bash
# Option A: Merge main into feature/auth
git switch feature/auth
git merge main
# Creates a "merge commit from main" in your feature branch history
# Keeps your commits intact but adds noise to the branch history

# Option B (cleaner): Rebase feature/auth onto main (Topic 11)
git rebase main
# Rewrites your feature commits on top of latest main
# Result: linear history, easier to review

# At work: teams typically use ONE of these consistently (enforced by PR tools)
```

---

### Scenario C: Understanding `git log --first-parent`

After months of merges, `git log` looks noisy:

```bash
git log --oneline | head -20
# (mix of feature commits and merge commits)

# See ONLY what was merged into main (one line per feature):
git log --oneline --first-parent main | head -10
# abc Merge branch 'feature/PROJ-261-search' (#261)
# def Merge branch 'feature/PROJ-258-payment-v2' (#258)
# ghi Merge branch 'feature/PROJ-244-notifications' (#244)
# jkl Merge branch 'feature/PROJ-231-user-auth' (#231)
# mno chore: initial project setup

# Much cleaner — shows the "release-level" history of what features were added
```

---

### Scenario D: Emergency Revert of a Bad Merge

A merge introduced a bug. You need to undo the merge immediately on a shared branch.

```bash
# Find the merge commit:
git log --oneline --merges | head -5
# abc123 Merge branch 'feature/payment-v2' (HEAD → main)

# Revert the merge commit (NOT git reset — this is a shared branch):
git revert -m 1 abc123
# -m 1 = "keep the first parent's side" (main before the merge)
# Creates a new commit that undoes all changes from feature/payment-v2

git push origin main

# The bad feature is undone without rewriting history
# IMPORTANT: feature/payment-v2 is now "merged" per Git, but its changes are reverted.
# If you later want to re-merge after fixing the bug:
# You must REVERT the revert commit first, then merge again.
# (Or rebase feature/payment-v2 and merge fresh)
```

---

## RULE 12 — Quick Reference Card

### Merge Commands

| Command | What it does |
|---------|-------------|
| `git merge <branch>` | Merge branch into current branch |
| `git merge --no-ff <branch>` | Force merge commit (even if FF possible) |
| `git merge --ff-only <branch>` | Refuse if not fast-forwardable |
| `git merge --squash <branch>` | Squash to staged changes (no commit) |
| `git merge --no-commit <branch>` | Perform merge but stop before committing |
| `git merge --abort` | Abort in-progress merge, restore pre-merge state |
| `git merge --continue` | Finalize merge after resolving conflicts |
| `git merge -m "msg" <branch>` | Specify merge commit message |
| `git merge --log <branch>` | Include commit summaries in merge message |
| `git merge --allow-unrelated-histories <branch>` | Merge repos with no common ancestor |

### Strategy and Options

| Command | What it does |
|---------|-------------|
| `git merge -s ours <branch>` | Merge (absorb history) but keep only our content |
| `git merge -s octopus a b c` | Merge multiple branches at once (no conflicts allowed) |
| `git merge -X ours <branch>` | Auto-resolve conflicts by taking ours |
| `git merge -X theirs <branch>` | Auto-resolve conflicts by taking theirs |
| `git merge -X ignore-space-change <branch>` | Ignore whitespace in conflict detection |

### During Conflict Resolution

| Command | What it does |
|---------|-------------|
| `git status` | Show which files are conflicted |
| `git diff` | Show conflict markers in unstaged files |
| `git diff --name-only --diff-filter=U` | List only conflicted files |
| `git show :1:<file>` | Show BASE (ancestor) version of conflicted file |
| `git show :2:<file>` | Show OUR (current branch) version |
| `git show :3:<file>` | Show THEIR (merging branch) version |
| `git checkout --ours <file>` | Accept our version entirely |
| `git checkout --theirs <file>` | Accept their version entirely |
| `git add <file>` | Mark conflict as resolved |
| `git diff --check` | Check for leftover conflict markers |

### Inspecting Merges

| Command | What it does |
|---------|-------------|
| `git log --merges` | Show only merge commits |
| `git log --no-merges` | Show only non-merge commits |
| `git log --first-parent` | Follow only first parent (mainline) |
| `git log --graph --all` | Visual DAG with all branches |
| `git cat-file -p HEAD` | Inspect merge commit (see two parent lines) |
| `git merge-base <a> <b>` | Find common ancestor of two branches |
| `git merge-tree <base> <a> <b>` | Preview merge without doing it |

### The `.git/` Files During a Merge

| File | Contents | Exists when |
|------|----------|-------------|
| `.git/MERGE_HEAD` | SHA of the branch being merged | During in-progress merge |
| `.git/MERGE_MSG` | Default merge commit message | During in-progress merge |
| `.git/MERGE_MODE` | Merge mode flags | During in-progress merge |
| `.git/ORIG_HEAD` | SHA of HEAD before the merge | After merge completes (for undo) |

---

## Summary

Git merge is the operation that joins the DAG back together after branching. There are
two fundamentally different cases:

1. **Fast-forward** (when your current branch is an ancestor of the merged branch):
   Just moves the branch pointer forward. Zero new objects created. Instant.

2. **True merge** (when both branches have diverged): Finds the common ancestor (merge
   base), performs a 3-way diff of all files, auto-resolves non-conflicting changes,
   requires manual resolution for conflicts, creates a new merge commit with two parents.

The merge commit is proof that Git's object model (from Topic 02) is beautifully simple:
a merge commit is identical to a regular commit except it has two `parent` lines instead
of one. That's the entire difference, structurally.

Understanding `MERGE_HEAD`, index stage numbers 1/2/3, and the 3-way merge algorithm
is what lets you confidently resolve ANY conflict — because you know exactly what Git
is showing you and what it needs from you.

*Last Updated: Topic 09 — Merging — ✅ Completed.*
