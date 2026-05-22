# Topic 11: Rebasing — The Power Tool
### (git rebase, interactive rebase, --onto, golden rule, rebase vs merge)

---

## CONNECTION TO PREVIOUS TOPICS (Rule 11)

Before we start: this topic builds on EVERYTHING you've learned so far.

- **Topic 01–02**: You know that a commit is a physical object in `.git/objects/` — a
  zlib-compressed blob with a SHA-1 hash. Rebasing **creates brand-new commit objects**
  with new SHAs. The old commits don't disappear; they just become unreachable.

- **Topic 03**: You know the DAG — a Directed Acyclic Graph where each commit points to its
  parent. Rebasing **rewires the parent pointer**, grafting commits onto a different base.

- **Topic 05**: You know the three trees (working dir, index, HEAD). During a rebase, Git
  temporarily moves through each commit, updating all three trees on each step.

- **Topic 08**: You know a branch is a 41-byte file in `.git/refs/heads/`. After rebase, that
  file is updated to point to the new tip commit with the new SHA.

- **Topic 09**: You understand fast-forward merge. After a rebase, a merge becomes a
  **pure fast-forward** — no diverged history, no merge commit. This is why teams use rebase.

- **Topic 10**: You know about remote-tracking branches and the push safety check. The golden
  rule of rebasing (`never rebase shared/pushed branches`) is rooted in exactly this: 
  rebasing changes SHAs, which breaks the fast-forward ancestry check on the remote.

---

## RULE 1: ELI5 ANCHOR — The Transplant Surgeon

Imagine you wrote 3 chapters of a book while your editor was working on a separate prologue.
Now the editor finishes and says "put your 3 chapters AFTER my prologue."

You can't just tape them on — your chapters start with "As I mentioned in the intro..." but
there IS no intro in your draft. You have to rewrite each chapter slightly so it reads
coherently after the new prologue.

That's **rebase**: take your commits (chapters), cut them out, replay them one by one
on top of the new base (the prologue), creating new versions of each commit that have the
same changes but a different starting point (and therefore a different SHA-1 hash).

The **original** chapters still exist (in reflog) — you just don't use them anymore.

```
BEFORE rebase:
                    A---B---C   ← your feature branch (feature)
                   /
    D---E---F---G               ← main (E is where feature branched)
                    
AFTER rebase:
                            A'--B'--C'   ← feature (same changes, new SHAs)
                           /
    D---E---F---G           ← main

(A' has same diff as A but parent is G, not E)
(Old A, B, C still exist in .git/objects — just unreachable)
```

---

## RULE 2: PHYSICAL ARCHITECTURE FIRST

### What `.git/` looks like BEFORE rebase starts

```
.git/
├── HEAD                        ← "ref: refs/heads/feature"
├── ORIG_HEAD                   ← (may not exist yet)
├── config
├── objects/
│   ├── e5/                     ← commit E (base of feature branch)
│   │   └── 7a2b...             
│   ├── f6/                     ← commit F
│   │   └── 8c3d...
│   ├── a1/                     ← commit A (first feature commit)
│   │   └── 2f4e...
│   ├── b2/                     ← commit B (second feature commit)
│   │   └── 3a5c...
│   └── c3/                     ← commit C (third feature commit)
│       └── 4d6b...
├── refs/
│   ├── heads/
│   │   ├── main               ← contains SHA of G (most recent main commit)
│   │   └── feature            ← contains SHA of C (tip of feature)
│   └── remotes/
│       └── origin/
│           └── main           ← contains SHA of G
└── index                      ← staging area
```

### What `.git/` looks like WHILE rebase is in progress

Git uses different directories depending on the rebase type:

**For `git rebase` (non-interactive) — uses `rebase-apply/` directory:**
```
.git/
├── HEAD                        ← detached HEAD — points directly to a commit SHA
│                                  (NOT "ref: refs/heads/feature" anymore!)
├── ORIG_HEAD                   ← SHA of C (where feature branch was before rebase)
├── rebase-apply/               ← ← ← THIS IS THE REBASE CONTROL CENTER
│   ├── patch                   ← the current patch being applied (diff format)
│   ├── original-commit         ← SHA of the original commit being replayed
│   ├── head-name               ← "refs/heads/feature" (which branch we're rebasing)
│   ├── onto                    ← SHA of the base we're rebasing onto
│   ├── last                    ← total number of patches (e.g., "3")
│   ├── next                    ← which patch we're currently on (e.g., "2")
│   └── msg                     ← commit message of current patch
├── objects/                    ← both old AND new commit objects coexist here
│   ├── a1/2f4e...              ← old commit A (still here, will become unreachable)
│   ├── b2/3a5c...              ← old commit B
│   ├── c3/4d6b...              ← old commit C
│   ├── [new SHA]/              ← new commit A' (already written if past step 1)
│   └── [new SHA]/              ← new commit B' (already written if past step 2)
└── refs/
    └── heads/
        └── feature             ← still points to C (old tip) — updated LAST
```

**For `git rebase -i` (interactive) — uses `rebase-merge/` directory:**
```
.git/
├── HEAD                        ← detached HEAD
├── ORIG_HEAD                   ← SHA of original tip
├── REBASE_HEAD                 ← SHA of the commit currently being applied
├── rebase-merge/               ← ← ← INTERACTIVE REBASE CONTROL CENTER
│   ├── git-rebase-todo         ← THE TODO FILE (what you edited with pick/squash/etc)
│   ├── done                    ← lines already processed from todo
│   ├── stopped-sha             ← SHA of commit that caused a stop (conflict or 'edit')
│   ├── head-name               ← "refs/heads/feature"
│   ├── onto                    ← SHA of base commit
│   ├── orig-head               ← SHA of original branch tip
│   ├── message                 ← commit message of current commit
│   ├── author-script           ← GIT_AUTHOR_NAME/EMAIL/DATE for current commit
│   └── current-fixups          ← accumulates squash/fixup messages
└── ...
```

### What `.git/` looks like AFTER rebase completes

```
.git/
├── HEAD                        ← "ref: refs/heads/feature" (restored to symbolic)
├── ORIG_HEAD                   ← SHA of C (the old tip — your safety net!)
├── objects/
│   ├── [A' SHA]/               ← new commit A' (different SHA than A)
│   ├── [B' SHA]/               ← new commit B' 
│   ├── [C' SHA]/               ← new commit C'
│   ├── a1/2f4e...              ← old commit A (STILL HERE — unreachable but present)
│   ├── b2/3a5c...              ← old commit B (still here)
│   └── c3/4d6b...              ← old commit C (still here — GC will clean up later)
└── refs/
    └── heads/
        └── feature             ← NOW points to C' (the new tip)
```

### KEY INSIGHT: Old commits still exist until GC runs
After rebase, the original A, B, C commits are **orphaned** (no branch points to them)
but they're still in `.git/objects/`. You can recover them via `git reflog` for 30 days
(default) before `git gc` prunes them. This is your safety net.

---

## RULE 3: EXACT DATA FLOW — How `git rebase main` Works Internally

### Setup:
```
main:    D---E---F---G          (G is the tip of main)
feature:         E---A---B---C  (E is where feature diverged from main)
```

### Command:
```bash
git switch feature
git rebase main
```

### Step-by-step trace:

```
COMMAND: git rebase main

STEP 1: FIND THE DIVERGENCE POINT
  READS:   .git/refs/heads/main  → SHA of G
           .git/refs/heads/feature → SHA of C
  COMPUTES: Common ancestor of G and C = E  (using merge-base algorithm)
  RESULT:   Commits to replay = A, B, C (everything on feature after E)

STEP 2: SAVE STATE
  WRITES:  .git/ORIG_HEAD ← SHA of C (current feature tip — undo anchor)
  WRITES:  .git/rebase-merge/head-name ← "refs/heads/feature"
  WRITES:  .git/rebase-merge/onto ← SHA of G
  WRITES:  .git/rebase-merge/orig-head ← SHA of C

STEP 3: DETACH HEAD
  MODIFIES: .git/HEAD ← SHA of G (detached, pointing to main's tip)
  RESULT:   Working directory now matches commit G

STEP 4: REPLAY COMMIT A onto G
  READS:   Commit A's patch (diff between E's tree and A's tree)
  APPLIES: That diff on top of G's tree (in working directory + index)
  COMPUTES: New commit A' with:
              tree   = new tree SHA (G's tree + A's changes)
              parent = SHA of G       ← DIFFERENT from A (A had parent=E)
              author = same as A
              committer = current time (can differ from A)
              message = same as A
  WRITES:  .git/objects/[A' SHA]  ← new commit object
  MODIFIES: .git/HEAD ← now points to A'

STEP 5: REPLAY COMMIT B onto A'
  (same process — B's diff applied on top of A')
  WRITES:  .git/objects/[B' SHA]
  MODIFIES: .git/HEAD ← now points to B'

STEP 6: REPLAY COMMIT C onto B'
  (same process)
  WRITES:  .git/objects/[C' SHA]
  MODIFIES: .git/HEAD ← now points to C'

STEP 7: UPDATE BRANCH POINTER
  MODIFIES: .git/refs/heads/feature ← SHA of C' (new tip)
  MODIFIES: .git/HEAD ← "ref: refs/heads/feature" (re-attached to branch)

STEP 8: CLEANUP
  REMOVES: .git/rebase-merge/ directory (rebase complete)

FINAL STATE:
  feature branch points to C'
  main branch still points to G (UNCHANGED)
  Old A, B, C still in .git/objects (unreachable, pending GC)
```

### Byte-level commit object change (Rule 9):

The ONLY difference between commit A and commit A' is the `parent` line:

```
ORIGINAL COMMIT A:
tree   a1b2c3d4e5...        ← tree built from E + A's changes
parent e7f8a9b0c1...        ← SHA of E  ← THIS CHANGES IN A'
author Alice <a@dev.com> 1748000000 +0000
committer Alice <a@dev.com> 1748000000 +0000

Add user authentication endpoint

REWRITTEN COMMIT A':
tree   d2e3f4a5b6...        ← tree built from G + A's changes (DIFFERENT tree!)
parent g9h0i1j2k3...        ← SHA of G  ← different parent
author Alice <a@dev.com> 1748000000 +0000
committer Alice <a@dev.com> 1748600000 +0000  ← committer time may differ!

Add user authentication endpoint

Since the CONTENT of the commit object changes, SHA-1 produces a completely different hash.
A and A' have the same diff, same message, same author — but different SHA.
```

---

## RULE 4: ASCII ARCHITECTURE DIAGRAMS

### Diagram 1: Before vs After Rebase (the classic view)

```
BEFORE:
                    A---B---C   feature
                   /
    D---E---F---G               main

AFTER git rebase main (run while on feature):
                            A'--B'--C'  feature
                           /
    D---E---F---G                       main

What happened to A, B, C?
They're still in .git/objects/ as orphaned commits:
    D---E---F---G---A'--B'--C'   (feature, linear history)
             \
              A---B---C           (orphaned, accessible via reflog only)
```

### Diagram 2: The Commit Replay Process

```
REPLAY STEP BY STEP:

Round 1: Apply A's diff onto G
  G ──── apply diff(E→A) ────► A'
  (A' parent = G, A' has new SHA)

Round 2: Apply B's diff onto A'
  A' ──── apply diff(A→B) ────► B'
  (B' parent = A', B' has new SHA)

Round 3: Apply C's diff onto B'
  B' ──── apply diff(B→C) ────► C'
  (C' parent = B', C' has new SHA)

feature pointer: C ──────────────────────────► C'
                 (old)                         (new)
```

### Diagram 3: Rebase enables clean fast-forward merge

```
AFTER REBASE:
    D---E---F---G---A'--B'--C'   feature (ahead of main by 3 commits)
                ^
                G = tip of main

NOW MERGE:
    git switch main
    git merge feature        ← FAST-FORWARD because C' descends from G

Result:
    D---E---F---G---A'--B'--C'   both main and feature point here
    
No merge commit created. History is perfectly linear.
```

### Diagram 4: Rebase vs Merge — history comparison

```
MERGE APPROACH:
    A---B---C--------M   main (M = merge commit)
             \      /
              D---E     feature

git log --graph shows:
*   M  Merge branch 'feature'
|\
| * E  Feature commit 2
| * D  Feature commit 1
* | C  Main commit 3
* | B  Main commit 2
|/
* A  Base

REBASE APPROACH:
    A---B---C---D'---E'  main (after merge feature)

git log --graph shows:
* E'  Feature commit 2
* D'  Feature commit 1
* C   Main commit 3
* B   Main commit 2
* A   Base

Linear. No parallel lines. Like it was always meant to be this way.
```

### Diagram 5: What's Inside `.git/rebase-merge/` During Interactive Rebase

```
.git/rebase-merge/
│
├── git-rebase-todo   ← THE INSTRUCTION SHEET
│   Contents:
│   pick a1b2c3 Add user login endpoint
│   pick b2c3d4 Add password hashing
│   pick c3d4e5 Fix typo in login error message
│   pick d4e5f6 Add logout endpoint
│   (you edit this file, then save and close editor)
│
├── done              ← COMPLETED STEPS (Git moves lines here as it processes them)
│   Contents (after first 2 done):
│   pick a1b2c3 Add user login endpoint
│   pick b2c3d4 Add password hashing
│
├── onto              ← WHERE WE'RE REBASING ONTO
│   Contents: g9h0i1j2k3...  (SHA of base commit)
│
└── head-name         ← WHAT BRANCH WE'RE ON
    Contents: refs/heads/feature
```

---

## RULE 5: HANDS-ON PROOF COMMANDS

### Setup: Create a repo to experiment with

```bash
# Create fresh repo
mkdir rebase-lab && cd rebase-lab
git init
git config user.email "you@test.com"
git config user.name "Test"

# Create base commit on main
echo "# App" > README.md
git add README.md
git commit -m "Initial commit"

# Create two more commits on main
echo "main line 2" >> README.md && git commit -am "Main: add line 2"
echo "main line 3" >> README.md && git commit -am "Main: add line 3"

# PROVE IT: See main's history
git log --oneline
# Expected output (newest first):
# abc1234 Main: add line 3
# def5678 Main: add line 2
# ghi9012 Initial commit

# Save main's current SHA
MAIN_TIP=$(git rev-parse HEAD)
echo "Main tip: $MAIN_TIP"
```

### Create the feature branch and diverge

```bash
# Go back to initial commit and create feature branch
git checkout -b feature HEAD~2

# Create 3 feature commits
echo "feature line A" > feature.txt && git add feature.txt
git commit -m "Feature: add feature.txt"

echo "feature line B" >> feature.txt
git commit -am "Feature: add line B"

echo "feature line C" >> feature.txt
git commit -am "Feature: add line C"

# PROVE IT: See diverged history
git log --oneline --graph --all
# Expected:
# * abc1234 Feature: add line C      ← feature (HEAD)
# * def5678 Feature: add line B
# * ghi9012 Feature: add feature.txt
# | * jkl3456 Main: add line 3       ← main
# | * mno7890 Main: add line 2
# |/
# * pqr1234 Initial commit

# PROVE IT: See what .git/HEAD contains
cat .git/HEAD
# Expected: ref: refs/heads/feature

# PROVE IT: See feature's ref file
cat .git/refs/heads/feature
# Expected: the SHA of "Feature: add line C"
```

### Perform the rebase and watch `.git/` change

```bash
# PROVE IT: Before rebase — feature's parent is Initial Commit
git log --oneline feature
# Feature: add line C
# Feature: add line B
# Feature: add feature.txt
# Initial commit

# Now rebase feature onto main
git rebase main

# PROVE IT: After rebase — feature's parent is now main's tip
git log --oneline feature
# Feature: add line C    ← C' (new SHA!)
# Feature: add line B    ← B' (new SHA!)
# Feature: add feature.txt ← A' (new SHA!)
# Main: add line 3       ← now in history!
# Main: add line 2
# Initial commit

# PROVE IT: ORIG_HEAD was written
cat .git/ORIG_HEAD
# Expected: SHA of old "Feature: add line C" (the original C, not C')

# PROVE IT: rebase-merge is GONE (cleanup happened)
ls .git/rebase-merge 2>/dev/null || echo "rebase-merge dir is gone — rebase complete"

# PROVE IT: The old commits still exist in objects!
# Get the old SHA from ORIG_HEAD
OLD_C=$(cat .git/ORIG_HEAD)
git cat-file -t $OLD_C       # Expected: commit (still there!)
git cat-file -p $OLD_C       # Expected: shows old commit C with old parent

# PROVE IT: The new commit C' has main's tip as ancestor
git log --oneline --graph
# * [new C' SHA] Feature: add line C
# * [new B' SHA] Feature: add line B
# * [new A' SHA] Feature: add feature.txt
# * [main tip SHA] Main: add line 3
# * [SHA] Main: add line 2
# * [SHA] Initial commit
```

### Prove fast-forward is now possible

```bash
# PROVE IT: Merge is now a pure fast-forward
git switch main
git merge feature --ff-only
# Expected: Updating abc1234..xyz9876   (no merge commit message!)

git log --oneline
# All commits in linear order — main now includes all feature commits
# No "Merge branch 'feature'" commit anywhere
```

### Prove reflog has the old commits

```bash
# PROVE IT: reflog shows where feature WAS before rebase
git reflog show feature | head -5
# feature@{0}: rebase finished: refs/heads/feature onto [main tip]
# feature@{1}: rebase: Feature: add line C       ← during replay
# feature@{2}: rebase: Feature: add line B
# feature@{3}: rebase: Feature: add feature.txt
# feature@{4}: rebase (start): checkout [main tip]
# feature@{5}: commit: Feature: add line C       ← the ORIGINAL commits
# feature@{6}: commit: Feature: add line B
# feature@{7}: commit: Feature: add feature.txt

# PROVE IT: You can recover to BEFORE the rebase
git reset --hard ORIG_HEAD
# HEAD is now at [old C SHA] Feature: add line C
# The branch is RESTORED to before the rebase!

# PROVE IT: Redo the rebase (just to show you can)
git rebase main
```

---

## THE INTERACTIVE REBASE — `git rebase -i`

### What it is

Interactive rebase opens an editor showing all commits to be replayed.
You can **reorder, squash, reword, drop, split, or pause** at any commit.

```bash
git rebase -i <base>
# Opens editor with a todo list like:
pick a1b2c3 Add user login endpoint
pick b2c3d4 Add password hashing
pick c3d4e5 Fix typo in login error message
pick d4e5f6 Add logout endpoint

# Commands:
# p, pick   <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit   <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup  <commit> = like "squash", but discard this commit's log message
# x, exec   <command> = run command (the rest of the line) using shell
# b, break            = stop here (continue rebase later with 'git rebase --continue')
# d, drop   <commit> = remove commit
# l, label  <label> = label current HEAD with a name
# t, reset  <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
```

### RULE 6: SYNTAX BREAKDOWN — `git rebase -i`

```
git rebase -i HEAD~4
│   │       │  │
│   │       │  └─ The BASE: everything AFTER this commit will be in the todo list
│   │       │     HEAD~4 means: the 4 commits before HEAD
│   │       │     Can also be: a SHA, a branch name, a tag
│   │       │     The base commit itself is NOT included in the list
│   │       └─ --interactive: open the todo list editor before replaying
│   └─ rebase subcommand
└─ git binary

git rebase -i --autosquash HEAD~8
│                │
│                └─ Automatically rearrange commits prefixed with fixup! or squash!
│                   so they fall after the matching commit and use fixup/squash action
└─ (rest same as above)

git rebase --onto <newbase> <oldbase> <branch>
│          │       │         │         │
│          │       │         │         └─ The branch to move (defaults to HEAD)
│          │       │         └─ The old base: commits AFTER this get moved
│          │       └─ Where to put them: the new parent
│          └─ --onto: use three-argument form
└─ git rebase

git rebase --continue
│          │
│          └─ After resolving conflicts or editing ('edit' command), resume the rebase
└─ git rebase (still in rebase session — .git/rebase-merge/ still exists)

git rebase --abort
│          │
│          └─ Cancel the entire rebase. Restores branch to ORIG_HEAD.
│             Removes .git/rebase-merge/ or .git/rebase-apply/
└─ git rebase

git rebase --skip
│          │
│          └─ Skip the current commit entirely (don't apply it)
│             Use when a commit's changes are already in the base
└─ git rebase
```

### The 8 Interactive Rebase Commands — Deep Explanation

```
pick a1b2c3 Add user login endpoint
```
**PICK**: Use the commit as-is. Replay it exactly. No changes to message or content.
This is the default for every commit — change it to use the other commands.

---

```
reword a1b2c3 Add user login endpoint
```
**REWORD**: Replay the commit's changes unchanged, but pause and open an editor to
let you change the commit message. Good for fixing typos in commit messages without
changing the content.

Data flow: commit is applied → Git pauses → opens `$EDITOR` with current message →
you save → Git creates new commit with same tree but new message → continues

---

```
edit a1b2c3 Add user login endpoint
```
**EDIT**: Replay the commit, then **pause rebase at that commit** with HEAD pointing
to that commit. You can:
- Add more files: `git add newfile.txt && git commit --amend`
- Split the commit: `git reset HEAD~1` (unstage everything), then create multiple commits
- Fix a bug introduced in that specific commit

When done: `git rebase --continue`

---

```
squash a1b2c3 Fix typo in login error message
```
**SQUASH**: Combine this commit with the commit ABOVE it in the todo list (the previous
commit). Opens editor showing BOTH commit messages combined — you edit the merged message.

```
BEFORE squash (todo list):
pick  d4e5f6 Add logout endpoint
squash a1b2c3 Fix typo in logout error message

AFTER squash:
One new commit containing changes from BOTH d4e5f6 and a1b2c3
You write the combined message (Git shows both originals for reference)
```

---

```
fixup a1b2c3 Fix typo in login error message
```
**FIXUP**: Like squash, but **silently discards this commit's message** and keeps
only the previous commit's message. No editor opens. Perfect for:
- Fixing a typo you made in the previous commit
- Small corrections that shouldn't pollute history

```bash
# WORKFLOW: Make a fix for a previous commit, label it for fixup
git commit -m "fixup! Add logout endpoint"
# Later during rebase -i --autosquash, this automatically becomes:
fixup [SHA] fixup! Add logout endpoint
# placed right under the "Add logout endpoint" commit
```

---

```
exec npm test
```
**EXEC**: Run a shell command after the preceding commit is applied. If the command
fails (non-zero exit code), rebase pauses. Used for:
- Running tests after each commit during rebase to verify nothing breaks
- Building the project to verify each commit compiles

```bash
git rebase -i HEAD~5 --exec "npm test"
# Automatically inserts:
pick a1b2c3 First commit
exec npm test
pick b2c3d4 Second commit
exec npm test
# ... (after every pick)
```

---

```
drop a1b2c3 Remove this debug log commit
```
**DROP**: Remove the commit entirely. Its changes are thrown away. Like the commit
never happened. Equivalent to deleting the line from the todo file.

---

```
break
```
**BREAK**: Stop here and wait. Same as `edit` but no commit to replay — just pause.
After doing whatever you need: `git rebase --continue`

---

### HANDS-ON PROOF: Interactive Rebase

```bash
# Setup: 5 messy commits to clean up
git init interactive-lab && cd interactive-lab
git config user.email "test@test.com" && git config user.name "Test"
echo "base" > file.txt && git add . && git commit -m "Initial commit"

echo "feature A" >> file.txt && git commit -am "Add feature A"
echo "typo" >> file.txt && git commit -am "Add fetaure B"   # typo in message
echo "fix" >> file.txt && git commit -am "fixup! Add fetaure B"  # fixup commit
echo "feature C" >> file.txt && git commit -am "WIP feature C"  # WIP commit
echo "done C" >> file.txt && git commit -am "Finish feature C"

# PROVE IT: See messy history
git log --oneline
# [sha] Finish feature C
# [sha] WIP feature C
# [sha] fixup! Add fetaure B
# [sha] Add fetaure B
# [sha] Add feature A
# [sha] Initial commit

# INTERACTIVE REBASE to clean up (last 5 commits, starting after Initial commit)
git rebase -i HEAD~5 --autosquash
```

**The editor opens showing:**
```
pick [sha1] Add feature A
pick [sha2] Add fetaure B      ← typo — we'll use 'reword'
fixup [sha3] fixup! Add fetaure B  ← autosquash moved this here automatically!
squash [sha4] WIP feature C    ← we'll squash WIP into Finish
pick [sha5] Finish feature C
```

**Edit to:**
```
pick [sha1] Add feature A
reword [sha2] Add fetaure B    ← reword to fix typo
fixup [sha3] fixup! Add fetaure B  ← automatically folded in
squash [sha4] WIP feature C
pick [sha5] Finish feature C
```

**After saving, Git will:**
1. Replay sha1 as-is
2. Replay sha2 + open editor (you fix: "Add feature B")
3. Replay sha3 as fixup (silently combined into sha2's new commit)
4. Replay sha4 + sha5 squashed together (editor opens for combined message)

```bash
# After interactive rebase completes:
git log --oneline
# [new sha] Add feature B + Finish feature C   ← squashed C
# [new sha] Add feature B                      ← reworded, typo fixed
# [new sha] Add feature A
# [sha] Initial commit

# PROVE IT: The fixup commit is GONE (merged in, message discarded)
git log --oneline | grep "fixup"
# (no output — it was consumed by fixup)
```

---

## THE `--onto` FLAG: Advanced Commit Transplanting

### The problem `--onto` solves

```bash
# Scenario: you accidentally branched feature2 from feature1 instead of main
# 
#                         feature1
#                         E---F
#                        /
#     A---B---C---D      main
#              \
#               E---F---G---H   feature2  (accidentally based on feature1!)
#
# You want feature2 to start from main's tip (D), NOT from feature1's base
```

### `git rebase --onto` syntax

```
git rebase --onto <newbase> <oldbase> [<branch>]
                  │          │         │
                  │          │         └─ Branch to move (defaults to HEAD)
                  │          └─ OLD starting point — commits AFTER this get moved
                  └─ NEW starting point — where the commits get replanted
```

### Example: Detach feature2 from feature1

```bash
# Before:
# A---B---C---D    main
#          \
#           E---F          feature1 (C is where it diverged)
#            \
#             G---H        feature2 (E is where it diverged from feature1)

# We want G and H (just the feature2-specific commits) onto D (main tip)

git rebase --onto main feature1 feature2
#                  │       │       │
#                  │       │       └─ move the feature2 branch
#                  │       └─ "the commits AFTER feature1's tip" (after F)
#                  └─ "put them after main's tip" (after D)

# After:
# A---B---C---D---G'--H'  feature2 (rebased onto main, feature1 commits excluded)
#          \
#           E---F          feature1 (unchanged)
```

### HANDS-ON PROOF: `--onto`

```bash
mkdir onto-lab && cd onto-lab
git init
git config user.email "t@t.com" && git config user.name "T"

# main: 3 commits
echo "A" > f.txt && git add . && git commit -m "A"
echo "B" >> f.txt && git commit -am "B"
echo "C" >> f.txt && git commit -am "C"

# feature1 branching from B (HEAD~1)
git checkout -b feature1 HEAD~1
echo "E" >> f.txt && git commit -am "E"
echo "F" >> f.txt && git commit -am "F"

# feature2 accidentally branching from feature1 at E
git checkout -b feature2 HEAD~1
echo "G" > g.txt && git add . && git commit -m "G"
echo "H" >> g.txt && git commit -am "H"

# PROVE IT: See the messy history
git log --oneline --graph --all
# * [sha] H                  ← feature2 (HEAD)
# * [sha] G
# * [sha] E                  ← feature1 starts here
# | * [sha] F
# |/
# * [sha] B                  ← feature2 and feature1 share B..E
# * [sha] A
# (C is on a different track, let's pretend main is at A-B-C)

# NOW: transplant G and H onto main's tip (C), excluding feature1's commits
git rebase --onto main feature1 feature2

# PROVE IT: feature2 now starts from main
git log --oneline --graph feature2 main
# * [sha] H   feature2
# * [sha] G
# * [sha] C   main
# * [sha] B
# * [sha] A
```

---

## THE GOLDEN RULE: Never Rebase Shared/Pushed Branches

### What "shared" means

A branch is **shared** if another person has:
- pulled it, OR
- based their own work on it, OR
- you've pushed it to a remote

### Why rebase breaks shared branches (the physics)

```
SCENARIO:
You and colleague Alice are both working on 'feature'.

Day 1: You both have:
    main: A---B
    feature: A---B---C---D   (you and Alice both have this)

Day 2: You decide to rebase feature onto main's new commits:
    git rebase main   # main now has E and F
    
Your local feature is now:
    main: A---B---E---F
    feature: A---B---E---F---C'---D'   (new SHAs!)

Day 3: Alice does: git pull origin feature
Git sees:
    Alice's feature:           A---B---C---D        (old commits, old SHAs)
    Origin's feature (yours):  A---B---E---F---C'---D'  (new commits, new SHAs)

Git says: "these have diverged" (C and C' have different SHAs, they look like separate commits)
Git tries to merge them → creates a merge commit COMBINING C AND C' (same changes, twice!)

Alice's history becomes:
    A---B---E---F---C'---D'---M    (M = merge commit)
                \           /
                 C---D------

Alice now has the same changes applied TWICE in her history. 
When she pushes this disaster, everyone gets it.
```

### The SHA-change cascade

```
ORIGINAL:                 AFTER REBASE:
A (parent: none)          A (unchanged — pre-divergence)
B (parent: A)             B (unchanged)
C (parent: B) ─────────► C' (parent: F) ← SHA changed (different parent)
D (parent: C) ─────────► D' (parent: C') ← SHA changed (parent C' ≠ C)

Every commit after the divergence point changes SHA.
If Alice has any of these commits (C, D), she now has conflicting history.
```

### The rule, stated precisely

```
SAFE to rebase:     Local-only branches that no one else has
SAFE to rebase:     Your own feature branches before pushing for the first time
SAFE to rebase:     After git push, if you are the ONLY person using the branch
                    AND you immediately force-push (with coordination)

NEVER rebase:       main, master, develop, release — any branch others pull from
NEVER rebase:       Any branch another person has checked out or based work on
NEVER rebase:       Any branch you've pushed that others may have pulled
```

### What to do if you MUST rebase a pushed branch

```bash
# If you rebased a pushed branch and need to force push:
git push --force-with-lease origin feature
# --force-with-lease checks that the remote hasn't been updated by someone else
# since your last fetch. Safer than bare --force.

# NEVER use bare --force on shared branches:
git push --force origin feature  # DANGEROUS: silently overwrites others' work
```

---

## REBASE CONFLICTS: What Happens and How to Resolve

### When conflicts occur

Conflicts happen when the diff being applied (a commit's changes) clashes with
changes already in the base. This can happen more than once — once per commit being replayed.

### The `.git/` state during a conflict

```
.git/
├── HEAD                    ← detached HEAD pointing to the partially-applied commit
├── MERGE_MSG               ← (may exist) message showing conflict
├── REBASE_HEAD             ← SHA of the commit that caused the conflict
├── rebase-merge/
│   ├── git-rebase-todo     ← remaining steps (conflict is on current step)
│   ├── done                ← completed steps so far
│   ├── stopped-sha         ← SHA of the conflicting commit
│   ├── message             ← commit message of the conflicting commit
│   └── author-script       ← author info for the conflicting commit
└── index                   ← CONTAINS CONFLICT MARKERS (stage 1/2/3 entries)
```

### Conflict markers in the file

```
<<<<<<< HEAD
code from the base (what you're rebasing onto)
=======
code from the commit being applied
>>>>>>> [SHA] commit message of the conflicting commit
```

Note: unlike merge conflicts, the "HEAD" side is the BASE, not your current branch.
The ">>>>>" side is the commit being applied.

### Resolution process

```bash
# 1. Git stops, shows conflict
# REBASE ERROR:
# CONFLICT (content): Merge conflict in server.js
# error: could not apply a1b2c3... Add user login endpoint
# hint: Resolve all conflicts manually, mark them as resolved with
# hint: "git add/rm <conflicted_files>", then run "git rebase --continue".

# 2. Open conflicting file, resolve manually
vim server.js   # or use VS Code's merge editor

# 3. Mark resolved
git add server.js

# 4. Continue rebase (DON'T use git commit — use rebase --continue)
git rebase --continue
# Git opens editor for commit message (in case you want to adjust it)
# Save and close → next commit is replayed

# 5. If more conflicts, repeat steps 2-4

# ALTERNATIVE: Skip this commit entirely
git rebase --skip
# The commit's changes are thrown away — use only if you're sure the commit
# is redundant (its changes are already in the base)

# ALTERNATIVE: Abort the entire rebase
git rebase --abort
# Returns branch to exactly where it was before rebase started (using ORIG_HEAD)
```

### PROVE IT: Trigger and resolve a rebase conflict

```bash
mkdir conflict-lab && cd conflict-lab
git init && git config user.email "t@t.com" && git config user.name "T"

# Both branches modify the same line
echo "line 1" > file.txt && git add . && git commit -m "Initial"
git checkout -b feature

# Feature changes line 1
echo "feature version" > file.txt && git commit -am "Feature changes line 1"

# Main also changes line 1 (conflict!)
git checkout main
echo "main version" > file.txt && git commit -am "Main changes line 1"

# Try to rebase feature onto main
git checkout feature
git rebase main

# CONFLICT! Git stopped.
# PROVE IT: See the conflict markers
cat file.txt
# <<<<<<< HEAD
# main version
# =======
# feature version
# >>>>>>> [sha] Feature changes line 1

# PROVE IT: rebase-merge exists and has control files
ls .git/rebase-merge/
# git-rebase-todo  done  stopped-sha  head-name  onto  orig-head  message  author-script

# PROVE IT: HEAD is detached
cat .git/HEAD
# [SHA of "Main changes line 1"] — NOT "ref: refs/heads/feature"

# Resolve: keep feature version
echo "feature version" > file.txt
git add file.txt
git rebase --continue
# Editor opens for commit message — save as-is

# PROVE IT: rebase completed, rebase-merge is gone
ls .git/rebase-merge 2>/dev/null || echo "Clean — rebase complete"
cat .git/HEAD
# ref: refs/heads/feature  ← symbolic ref restored
```

---

## `git rebase --autosquash`

### The fixup/squash workflow

```bash
# Day 1: Make a feature commit
git commit -m "Add payment processing module"

# Day 1 (later): Realize you forgot to handle edge case
git add payment.js
git commit -m "fixup! Add payment processing module"
# The "fixup! " prefix is the magic: it must EXACTLY match the target commit message

# Day 1 (even later): Add a test, want it squashed into the payment commit
git add payment.test.js
git commit -m "squash! Add payment processing module"

# Cleanup before PR: interactive rebase with autosquash
git rebase -i --autosquash main

# Git AUTOMATICALLY rearranges the todo list:
# BEFORE autosquash (normal order):
# pick a1b2c3 Add payment processing module
# pick b2c3d4 Other unrelated commit
# pick c3d4e5 fixup! Add payment processing module
# pick d4e5f6 squash! Add payment processing module

# AFTER autosquash (automatically reordered):
# pick a1b2c3 Add payment processing module
# fixup c3d4e5 fixup! Add payment processing module
# squash d4e5f6 squash! Add payment processing module
# pick b2c3d4 Other unrelated commit
```

### Make autosquash the default

```bash
# In ~/.gitconfig:
git config --global rebase.autoSquash true
# Now 'git rebase -i' always uses autosquash
```

### The `git commit --fixup` shortcut

```bash
# Instead of typing "fixup! Add payment processing module" manually:
git commit --fixup a1b2c3    # a1b2c3 = SHA of "Add payment processing module"
# Git automatically creates: "fixup! Add payment processing module"

git commit --squash a1b2c3   # Creates: "squash! Add payment processing module"
```

---

## REBASE vs MERGE: THE DECISION FRAMEWORK

### The fundamental tradeoff

```
MERGE preserves TRUTH: what actually happened (parallel work, diverged, rejoined)
REBASE creates CLARITY: the most readable history (linear, logical sequence)
```

### Decision matrix

| Situation | Use | Reason |
|-----------|-----|--------|
| Updating feature branch with latest main | `rebase` | Keeps feature history clean |
| Integrating feature into main | `merge --no-ff` | Preserves feature grouping |
| Cleaning up commits before PR | `rebase -i` | Makes PR review easier |
| Merging a hotfix | `cherry-pick` or `merge` | Depends on scope |
| Shared branch that others use | `merge` ONLY | Rebase would break teammates |
| Your local-only work | `rebase` freely | No one else has the commits |
| Preserving exact history for audit | `merge` | Don't rewrite anything |

### The principle: "Rebase locally, merge publicly"

```
LOCAL (before push):    Rebase freely — clean up, squash, reorder
REMOTE (after push):    Merge only — never rewrite published history
```

### When merge produces better history

```
# Scenario: A 3-month feature with 40 commits
# If you rebase and fast-forward merge:
git log --oneline main
# 40 commits interleaved with other features — hard to see "what was feature X?"

# If you merge with --no-ff:
git log --oneline --graph main
# *   Merge branch 'feature/payment'   ← clear boundary
# |\
# | * 40 commits for payment feature   ← all grouped together
# |/
# * Other work continues               ← cleanly separated
```

---

## RULE 7: BEGINNER → PRODUCTION EXAMPLE

### Beginner example: Clean up before pushing

```bash
# You've been working and made messy commits
git log --oneline
# 7a8b9c0 Fix lint error
# 5e6f7a8 WIP: add user model
# 3c4d5e6 Actually fix the thing
# 1a2b3c4 Add user model

# Clean up: squash into one meaningful commit
git rebase -i HEAD~4

# Todo list:
pick 1a2b3c4 Add user model
squash 3c4d5e6 Actually fix the thing
squash 5e6f7a8 WIP: add user model
squash 7a8b9c0 Fix lint error

# Result: one clean commit ready to push
git log --oneline
# [new sha] Add user model with validation
```

### Production example: Feature branch workflow at a 6-person team

**Scenario**: You're a backend engineer on a 6-person team. You've been working on the
`feature/payment-v2` branch for 2 weeks. main has had 15 commits since you branched.
Your PR reviewer said "rebase onto main before I review."

```bash
# Day 1: You branched from main at commit A
git checkout -b feature/payment-v2

# 2 weeks of work: 23 commits (some messy WIP commits)
git log --oneline feature/payment-v2 | head -10
# [sha] Fix failing test in payment_service_test.go
# [sha] WIP: debugging stripe webhook
# [sha] TEMP: add debug logging (remove before PR!)
# [sha] Add retry logic for payment failures
# [sha] Fix lint
# [sha] Add stripe webhook handler
# ...

# Step 1: Fetch latest main
git fetch origin
git log --oneline origin/main | head -5   # 15 new commits

# Step 2: Interactive rebase to clean up YOUR commits first (before rebasing onto main)
git rebase -i HEAD~23
# In the editor:
# - drop the "TEMP: add debug logging" commit
# - squash "Fix lint" into the previous payment commit
# - squash "WIP: debugging stripe webhook" into the webhook handler commit
# - reword any unclear messages
# Result: 23 commits → 8 clean, meaningful commits

# Step 3: Rebase cleaned branch onto latest main
git rebase origin/main

# Step 4: If conflicts, resolve them one by one
# Each conflict is in context — you know EXACTLY which commit caused it

# Step 5: Push to your remote feature branch
git push --force-with-lease origin feature/payment-v2
# --force-with-lease because you rewrote history
# "with-lease" checks no one else pushed to the remote since your last fetch

# Step 6: Open PR — reviewer sees:
git log --oneline feature/payment-v2 --not origin/main
# 8 clean, logical commits
# No "Fix lint", no "WIP", no "TEMP" commits
# Easy to review, easy to bisect if something breaks later
```

---

## RULE 8: COMMON MISTAKES → ROOT CAUSE → FIX → PREVENTION

### Mistake 1: Rebasing main/develop

**What happened**: You ran `git rebase feature` while on main, or pushed a rebased main.

**Root cause**: Misunderstanding which branch you're on, or thinking "rebase just cleans up
history" without realizing it creates new SHAs.

**How to diagnose**:
```bash
git log --oneline origin/main..main   # if this shows commits, you've rebased main
git push origin main                  # rejected: non-fast-forward
```

**Fix**:
```bash
git reset --hard origin/main   # DANGEROUS: loses your local commits
# OR (safer):
git checkout main
git pull origin main --rebase=false  # fetch and merge (no rewrite)
```

**Prevention**: Set branch protection on `main` on GitHub/GitLab to prevent force-push.

---

### Mistake 2: Running `git rebase` instead of `git rebase -i` and losing squash opportunity

**What happened**: You ran `git rebase main` but wanted to clean up commits too.

**Root cause**: Forgot that non-interactive rebase doesn't let you squash/drop.

**Fix**: After a successful rebase, immediately run `git rebase -i main` to clean up.
This is safe — you're just rewriting your already-rebased local commits.

---

### Mistake 3: Forgetting to add files before `--continue`

**What happened**: You resolved a conflict, forgot `git add`, ran `git rebase --continue`.

**Error message**:
```
error: you need to resolve your current index first
```

**Fix**:
```bash
git status              # shows what's still conflicted
git add <resolved-files>
git rebase --continue
```

**Prevention**: Always run `git status` after resolving conflicts to verify index is clean.

---

### Mistake 4: Using `squash` on the first commit in the todo list

**What happened**: The first item in the todo list was `squash [sha] something`.

**Error**: "Cannot squash without a previous commit."

**Root cause**: `squash` and `fixup` always combine with the PREVIOUS commit. There's
no previous commit for the first item.

**Fix**: Change the first `squash` to `pick`. You can only squash commits 2+ into commit 1,
never the other way.

---

### Mistake 5: `git rebase --continue` after editing — forgetting to amend

**What happened**: Used `edit` command in interactive rebase, made changes, but ran
`git rebase --continue` without committing them.

**Root cause**: `edit` pauses at a commit to let you amend it. If you have unstaged changes
and run `--continue`, those changes get left behind (they go into the NEXT commit).

**Fix**:
```bash
# If you forgot to amend:
git rebase --abort
# Start again, this time:
# 1. Stop at 'edit' commit
# 2. Make your changes
git add .
git commit --amend   # amend the stopped commit
git rebase --continue  # now continue
```

---

### Mistake 6: Rebase a branch that's already in a PR

**What happened**: PR is open for `feature/x`. You rebased it and force-pushed.
PR shows "This branch has been force-pushed." GitHub may lose review comments tied to
the old commits.

**Root cause**: Force-pushing rewrites history on the remote branch. GitHub links review
comments to specific commit SHAs — those SHAs are now gone.

**Prevention**: 
- Squash and clean up BEFORE opening the PR
- If you must rebase after PR is open, give reviewers a heads up
- Some teams use "rebase-merge" or "squash-merge" in the GitHub PR settings
  so reviewers never need to wait for you to rebase

---

## RULE 9: BYTE-LEVEL INTERNALS

### What Git actually does internally per-commit during rebase

```
For each commit being replayed, Git runs the equivalent of:
  git diff <original-parent> <commit>   → produces a patch
  git apply <that-patch>                → applies to current HEAD

If the patch applies cleanly:
  git write-tree                        → new tree SHA
  git commit-tree <tree> -p HEAD -m <msg> → new commit SHA
  git update-ref HEAD <new-commit>      → advance HEAD

If the patch fails to apply:
  → CONFLICT: stop, let user resolve manually
```

### How Git handles the `rebase-merge/` format

```
The git-rebase-todo file format:
<action> <sha> <message>

Valid actions and their internal behavior:
  pick    → cherry-pick the commit (git cherry-pick --allow-empty-message)
  reword  → cherry-pick then invoke editor for new message
  edit    → cherry-pick then stop (exec: false, amend: true)
  squash  → cherry-pick with --no-commit, then combine message
  fixup   → cherry-pick with --no-commit, discard message
  exec    → run shell command (exec: true)
  break   → stop (amend: false, exec: false)
  drop    → skip (no cherry-pick)
  label   → create temp ref
  reset   → reset to temp ref
  merge   → run git merge (for rebasing merges)
```

### The `cherry-pick` connection

Rebase uses `git cherry-pick` internally for each `pick` action. If you understand
`cherry-pick` (Topic 12 will cover this fully), you understand rebase at the commit level.

```bash
# You can manually simulate a rebase with cherry-pick:
git checkout -b manual-rebase main  # start from main's tip
git cherry-pick A B C               # replay feature commits one by one
# This is EXACTLY what git rebase does automatically
```

### `rebase.instructionFormat` — customizing the todo list

```bash
git config rebase.instructionFormat "%s (%an, %ar)"
# Now the todo list shows:
# pick a1b2c3 Add payment module (Alice Smith, 2 days ago)
# pick b2c3d4 Fix webhook handler (Bob Jones, 1 day ago)
# More context for each line — easier to decide what to squash
```

---

## RULE 10: MENTAL MODEL CHECKPOINT

Can you answer these from memory? If not, re-read the relevant section.

**1.** You run `git rebase main` on your feature branch. Name ALL the files Git writes or
modifies inside `.git/` during this operation, in order.

**2.** After rebase completes, you realize it was a mistake. You run `git reset --hard ORIG_HEAD`.
What does `ORIG_HEAD` contain, and how was it written?

**3.** What is the difference between `squash` and `fixup` in an interactive rebase todo list?
When would you choose one over the other?

**4.** Your teammate pushed a commit to `feature/auth` this morning. You also have `feature/auth`
checked out. If you run `git rebase main` right now, what problem could arise and why?

**5.** You used `edit` in an interactive rebase to stop at commit B. You made changes to 3 files.
What is the exact sequence of commands to properly continue the rebase with those changes
included in commit B?

**6.** What does `git rebase --onto main feature1 feature2` do? Draw the before/after DAG.

**7.** During a rebase, a conflict occurs on the 3rd commit out of 5. You run `git rebase --skip`.
What happens to that 3rd commit? What happens to commits 4 and 5?

**8.** What is the `rebase-merge/git-rebase-todo` file and when does it exist? What would you
find in `rebase-merge/done` after the first 2 commits are successfully replayed?

**9.** Explain why commit A and commit A' (same diff, same message, same author) have
different SHA-1 hashes after a rebase.

**10.** You ran `git rebase -i HEAD~5 --autosquash`. Git found 2 commits with the prefix
`fixup!`. What did it do to those commits in the todo list?

---

## RULE 12: QUICK REFERENCE CARD

| Command | What It Does |
|---------|-------------|
| `git rebase <branch>` | Replay current branch's commits on top of `<branch>` |
| `git rebase -i HEAD~N` | Interactive rebase: edit, squash, reorder last N commits |
| `git rebase -i <sha>` | Interactive rebase starting after a specific commit |
| `git rebase --onto <new> <old> [branch]` | Transplant commits from `<old>` to `<new>` base |
| `git rebase --continue` | Resume after resolving a conflict or editing a commit |
| `git rebase --abort` | Cancel rebase entirely, return to ORIG_HEAD |
| `git rebase --skip` | Discard the current conflicting commit and continue |
| `git rebase --autosquash` | Automatically reorder `fixup!`/`squash!` commits |
| `git rebase --exec "cmd"` | Run shell command after each replayed commit |
| `git commit --fixup <sha>` | Create a `fixup!` commit targeting `<sha>` |
| `git commit --squash <sha>` | Create a `squash!` commit targeting `<sha>` |
| `git push --force-with-lease` | Force-push with safety check (abort if remote updated) |

| Interactive Rebase Command | Effect |
|---------------------------|--------|
| `pick` (p) | Use commit as-is |
| `reword` (r) | Use commit, edit its message |
| `edit` (e) | Use commit, then pause to amend |
| `squash` (s) | Combine into previous commit, edit combined message |
| `fixup` (f) | Combine into previous commit, discard this message |
| `exec` (x) | Run a shell command |
| `break` (b) | Pause here (no commit to apply) |
| `drop` (d) | Delete this commit entirely |

| Config Setting | Meaning |
|---------------|---------|
| `rebase.autoSquash = true` | Always use autosquash in `git rebase -i` |
| `rebase.autoStash = true` | Auto stash/unstash around rebase |
| `pull.rebase = true` | `git pull` rebases instead of merges by default |
| `rebase.instructionFormat = "%s (%an)"` | Add author info to todo list lines |

| Safety File | When It Exists | Content |
|-------------|---------------|---------|
| `.git/ORIG_HEAD` | After rebase starts | Old branch tip — undo anchor |
| `.git/REBASE_HEAD` | During interactive rebase | SHA of current commit being applied |
| `.git/rebase-merge/` | During interactive rebase | Todo list and control files |
| `.git/rebase-apply/` | During non-interactive rebase | Patch files and control files |

---

## RULE 13: WHEN WOULD I USE THIS AT WORK?

### Scenario 1: Daily feature branch hygiene

```
You: Working on feature/user-onboarding for 3 days
Team: Merging into main 4-5 times per day
Problem: Your branch is 20 commits behind main, conflicts everywhere

Solution: Every morning, run:
  git fetch origin
  git rebase origin/main

Why: Your feature branch stays up-to-date. Conflicts are caught and fixed in small
increments (per commit, not all at once at PR time). When you open the PR, the
diff is clean — just your changes relative to the CURRENT main.
```

### Scenario 2: PR cleanup before review

```
You: 15 commits including "fix", "WIP", "oops", "debug logging"
Tech lead: "I won't review this. Clean it up."

Solution:
  git rebase -i origin/main

In the editor: squash all related commits, drop debug commits, reword unclear messages.
Result: 4 meaningful commits that tell a story. Code review takes 20 minutes instead of
90 minutes. PR is merged same day.
```

### Scenario 3: Recovering from branching off wrong branch

```
You: Created feature/reports by accident from feature/auth instead of main
feature/auth has 8 commits you don't want in feature/reports

Solution:
  git rebase --onto main feature/auth feature/reports

In seconds, feature/reports is transplanted onto main with only its own commits.
```

### Scenario 4: CI verification during rebase

```
Team policy: "Every commit on main must pass CI"
Problem: After squashing, you're not sure all intermediate commits compile

Solution:
  git rebase -i main --exec "go build ./... && go test ./..."

Git runs the build and tests after EACH commit. If any fails, rebase stops — you
fix the commit right then. Guarantees every commit in history is buildable.
```

### Scenario 5: Keeping a long-lived fork in sync

```
You: Maintainer of an internal fork of an open-source library
Upstream: Active development, major releases monthly

Workflow:
  git remote add upstream https://github.com/original/library.git
  git fetch upstream
  git rebase upstream/main

Your custom patches are always applied on top of the latest upstream code.
No merge commits polluting your fork's history. Clean diff to show your company's
customizations when auditing.
```

### Scenario 6: The `fixup!` + autosquash workflow on a team

```
Team convention: "When you fix a mistake in a previous commit (same PR),
use `git commit --fixup <sha>`. We autosquash before merging."

You: Added a typo in commit abc1234 (which added the payment model)
Fix: git commit --fixup abc1234

Before PR merge:
  git rebase -i --autosquash main
  → fixup automatically placed after abc1234
  → squashed silently
  → PR has clean history, no "Fix typo" commits
```

---

## SUMMARY: The Mental Model in One Paragraph

Rebase is a commit replay machine. It takes the commits on your branch that are NOT
on the target branch, extracts each one as a patch (diff), checks out the target branch's
tip, and applies each patch sequentially — creating brand-new commit objects with new SHA
hashes (because they have a different parent). The original commits become orphaned objects
in `.git/objects/` and are accessible via reflog for 30 days. During the replay, if a
patch conflicts with the base, rebase stops and asks you to resolve it manually. This
is Git saying: "I don't know how to combine these two changes automatically."

The golden rule flows directly from the SHA change: if you rebase commits that someone
else has in their history, their history and yours diverge at a SHA level — they look
like independent parallel work even though they're the same logical changes. Git will
try to "merge" them together and create duplicate commits. Never rebase shared branches.

---

*Topic 11 of 26 — Next: Topic 12: Undoing Things (reset soft/mixed/hard, revert, restore, clean)*
