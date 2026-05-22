# Topic 12: Undoing Things
### (git reset, git revert, git restore, git clean — Every Undo Operation Explained from Disk Up)

---

## CONNECTION TO PREVIOUS TOPICS (Rule 11)

This topic is the direct payoff of Topic 05 (The Three Trees).

- **Topic 05**: You know Git has three zones: Working Directory, Index (staging area), and HEAD
  (the repository / last commit). Every undo command works by moving one or more of these zones
  to match a target state.

- **Topic 02**: You know commits are immutable SHA-1 objects in `.git/objects/`. You CANNOT
  delete a commit — you can only make it unreachable. This is why `git reset --hard` feels
  "destructive" but is actually recoverable via reflog for 30 days.

- **Topic 03**: You know HEAD is a pointer. `git reset` moves this pointer. When you
  `git reset --hard HEAD~1`, you move the branch pointer back one commit — the old commit
  object is still on disk.

- **Topic 08**: You know a branch is a 41-byte file. `git reset` rewrites that file.
  It does NOT move HEAD to a different branch — it moves the branch HEAD is currently
  pointing to. This is a critical distinction.

- **Topic 11**: You know rebase creates new commits (new SHAs). `ORIG_HEAD` is your undo
  anchor for rebase. The same file is used by `reset`, `merge`, and `rebase` — it's always
  the "where you were before a dangerous operation."

- **Topic 09**: You know merge commits have two parents. `git revert -m 1` is how you
  undo a merge commit — you must tell Git which parent is the "mainline."

---

## RULE 1: ELI5 ANCHOR — The Four Kinds of "Undo"

Imagine you're writing a document with 3 drafts:
1. **Your notepad** (working directory) — messy, unfinished, live edits
2. **The photocopy tray** (index/staging) — what you told the printer to queue
3. **The filing cabinet** (repository/HEAD) — officially committed versions, numbered

Now, "undo" could mean four different things:

| You want to... | Git command | What it touches |
|---|---|---|
| Throw away notepad scribbles, revert to the photocopy | `git restore <file>` | Working directory only |
| Empty the photocopy tray but keep notepad | `git restore --staged <file>` | Index only |
| Go back to an older filing cabinet version (keep notepad & photocopy) | `git reset --soft HEAD~1` | HEAD/branch pointer only |
| Go back to older version, empty photocopy tray too (keep notepad) | `git reset --mixed HEAD~1` | HEAD + Index |
| Go back to older version, nuke notepad AND photocopy tray | `git reset --hard HEAD~1` | HEAD + Index + Working Dir |
| Create a NEW filing entry that cancels an old one (safe undo) | `git revert <sha>` | Creates new commit |
| Throw away every loose scrap of paper not in the tray or cabinet | `git clean -fd` | Untracked files only |

---

## RULE 2: PHYSICAL ARCHITECTURE FIRST

### The Three Trees — Quick Reminder

```
WORKING DIRECTORY          INDEX (.git/index)          HEAD (.git/refs/heads/<branch>)
                                                         │
   server.js ────────────► [staged version] ───────────►│──► commit C
   auth.js   ────────────► [staged version]              │    ├── tree
   README.md               [not staged]                  │    ├── parent B
                                                         │    └── ...
   ^                        ^                            ^
   On disk, uncompressed    Binary file with             Immutable object
   Git has no control       stat cache entries           in .git/objects/
   until you git add        indexed SHA-1s
```

### What `git reset HEAD~1` physically changes

```
BEFORE reset:
  .git/HEAD ────────────► ref: refs/heads/main
  .git/refs/heads/main ──► [SHA of commit C]   ← branch tip

AFTER git reset HEAD~1:
  .git/HEAD ────────────► ref: refs/heads/main   (unchanged — still a symbolic ref)
  .git/refs/heads/main ──► [SHA of commit B]   ← branch tip MOVED BACK

  .git/ORIG_HEAD ─────────► [SHA of commit C]   ← written as safety net
```

### `.git/` directory state map for undo operations

```
.git/
├── HEAD              ← "ref: refs/heads/main" (always stays symbolic for reset)
├── ORIG_HEAD         ← Written BEFORE: reset, merge, rebase (your undo anchor)
├── MERGE_HEAD        ← Exists ONLY during in-progress merge
├── CHERRY_PICK_HEAD  ← Exists ONLY during in-progress cherry-pick
├── REVERT_HEAD       ← Exists ONLY during in-progress revert (with conflicts)
├── refs/
│   └── heads/
│       └── main      ← THIS FILE changes when you do git reset
└── objects/
    └── [all commits still here — nothing deleted by reset]
```

### Before and after `git reset --hard HEAD~2` — full state change

```
BEFORE:
  Branch: main → C (HEAD)
  Index: matches C's tree
  Working dir: matches C's tree

  History: A──B──C  (main points here)

AFTER:
  Branch: main → A (moved back 2)
  Index: matches A's tree
  Working dir: matches A's tree

  History: A──B──C  (B and C still exist in .git/objects! Just unreachable)
           ^
           main (HEAD) now points here

  .git/ORIG_HEAD: SHA of C (can do: git reset --hard ORIG_HEAD to undo the undo)
```

---

## RULE 3: EXACT DATA FLOW — Every Undo Command

### `git reset --soft HEAD~1`

```
COMMAND: git reset --soft HEAD~1

TARGET: The commit one step before HEAD

READS:   .git/HEAD → .git/refs/heads/main → SHA of current commit C
         .git/objects/[C sha] → parent = SHA of B
WRITES:  .git/ORIG_HEAD ← SHA of C  (safety net written FIRST)
MODIFIES: .git/refs/heads/main ← SHA of B  (branch pointer moves back)

DOES NOT TOUCH:
  - .git/index         (staging area unchanged — still has C's staged state)
  - Working directory  (files unchanged — still have C's content)

NET EFFECT:
  You "uncommit" C but keep all its changes staged.
  git status shows: "Changes to be committed: ..."
  
USE CASE:
  You committed too early and want to add more files to the same commit.
  Or you want to squash the last N commits manually.
```

### `git reset --mixed HEAD~1` (THE DEFAULT)

```
COMMAND: git reset HEAD~1    (--mixed is the default)

TARGET: The commit one step before HEAD

READS:   Same as --soft
WRITES:  .git/ORIG_HEAD ← SHA of C
MODIFIES: .git/refs/heads/main ← SHA of B
          .git/index ← RESET to match B's tree (staging area cleared of C's changes)

DOES NOT TOUCH:
  - Working directory (files unchanged — C's content still there as modifications)

NET EFFECT:
  You "uncommit" AND "unstage" C's changes.
  git status shows: "Changes not staged for commit: ..."
  Your files still have C's content but you'd need to re-stage them.
  
USE CASE:
  You committed the wrong files and want to restructure what goes into the commit.
  Or you want to completely redo the last commit from scratch.
```

### `git reset --hard HEAD~1`

```
COMMAND: git reset --hard HEAD~1

TARGET: The commit one step before HEAD

READS:   Same as --soft
WRITES:  .git/ORIG_HEAD ← SHA of C
MODIFIES: .git/refs/heads/main ← SHA of B
          .git/index ← reset to match B's tree
          Working directory ← ALL FILES reset to match B's tree
                              (C's changes are GONE from disk — not staged, not anywhere)

NET EFFECT:
  Everything — branch pointer, index, working directory — all match commit B.
  C's changes are gone from disk.
  
  BUT: Commit C's OBJECT still exists in .git/objects/ until GC runs!
  You can still recover via: git reset --hard ORIG_HEAD  (or via reflog)
  
USE CASE:
  You want to completely abandon the last commit AND all its changes.
  Nuclear option for "this commit was a mistake, I don't want any of it."
```

### `git revert <sha>`

```
COMMAND: git revert a1b2c3

TARGET: The commit a1b2c3 — create a NEW commit that reverses its changes

READS:   .git/objects/a1b2c3 → the commit object
         parent of a1b2c3 → used to compute the diff
COMPUTES: diff between parent-of-a1b2c3 and a1b2c3 → produces a patch
          INVERTS that patch (additions become deletions, vice versa)
APPLIES: Inverted patch to current working directory
WRITES:  New blob objects for modified files
         New tree object
         New commit object with message: "Revert 'original message'"
MODIFIES: .git/refs/heads/main ← SHA of new revert commit

NET EFFECT:
  Creates a new commit D that undoes what a1b2c3 did.
  History becomes: A──B──a1b2c3──C──D
  The original a1b2c3 is still in history — just has a counterpart.
  
USE CASE:
  You need to undo a commit that has ALREADY been pushed to a shared branch.
  Safe undo: doesn't rewrite history, just adds a counterpart commit.
```

### `git restore <file>`

```
COMMAND: git restore server.js

DEFAULT SOURCE: Index (staging area)
  → Discards working directory changes to server.js
  → Restores the file to whatever is in the index (staged version)

READS:   .git/index → staged version of server.js
WRITES:  Working directory: server.js overwritten with staged version

DOES NOT TOUCH:
  - .git/index  (unchanged)
  - Any commits (unchanged)

NET EFFECT:
  Discards UNSTAGED changes only.
  
VARIANT: git restore --staged server.js
  SOURCE: HEAD (last commit)
  → Unstages server.js (removes from index)
  → Index reverts to HEAD's version of server.js
  → Working directory UNCHANGED

VARIANT: git restore --source=HEAD~2 server.js
  SOURCE: The commit 2 steps back
  → Overwrites working directory server.js with version from HEAD~2
  → Does NOT stage the change (use git add after if you want to commit)

VARIANT: git restore --staged --worktree server.js
  → Both unstages AND discards working directory changes
  → File goes back to HEAD version everywhere
```

### `git clean`

```
COMMAND: git clean -fd

READS:   .git/index → to know what's tracked
COMPUTES: Everything in working directory NOT in .git/index = untracked files
DELETES: All untracked files and directories

FLAGS:
  -f  → force (required by default — safety gate so you don't run clean accidentally)
  -d  → also remove untracked directories
  -n  → DRY RUN: show what WOULD be deleted without deleting anything
  -x  → also remove ignored files (files matching .gitignore patterns)
  -X  → ONLY remove ignored files (keep other untracked files)
  -i  → interactive mode: choose which files to remove

NET EFFECT:
  Working directory only. .git/ untouched. No recovery — these files are gone.
  There is NO ORIG_HEAD, NO reflog for untracked files.
  git clean is the ONE truly irreversible git command.
```

---

## RULE 4: ASCII ARCHITECTURE DIAGRAMS

### Diagram 1: The Three Trees and Which Command Moves Which

```
                  WORKING         INDEX           HEAD
                  DIRECTORY       (.git/index)    (commit)
                  
git restore f          ◄──────────────── reads index, overwrites WD
git restore --staged f              ◄───────────── reads HEAD, overwrites index  
git reset --soft       ─ unchanged ─ unchanged ──► moves branch pointer
git reset --mixed      ─ unchanged ──◄─────────────────────────────────
git reset --hard       ◄────────────◄──────────────────────────────────
git revert             ────── applies inverted patch, new commit ──────►
git clean              ◄── deletes untracked files (index untouched)
```

### Diagram 2: `git reset` mode comparison — the three-tree view

```
SCENARIO: Committed 3 files but want to undo
History: A──B──C  (HEAD/main at C)
                 
┌─────────────────┬────────────────┬──────────────────┬────────────────────────────┐
│                 │ Branch pointer │     Index        │    Working Directory       │
│                 │ (main)         │ (.git/index)     │    (disk)                  │
├─────────────────┼────────────────┼──────────────────┼────────────────────────────┤
│ git reset       │ B (moved back) │ C (unchanged)    │ C (unchanged)              │
│   --soft HEAD~1 │                │ C's changes      │ C's changes                │
│                 │                │ are STAGED       │ are on disk                │
├─────────────────┼────────────────┼──────────────────┼────────────────────────────┤
│ git reset       │ B (moved back) │ B (reset to B)   │ C (unchanged)              │
│  --mixed HEAD~1 │ (DEFAULT)      │ C's changes are  │ C's changes still          │
│                 │                │ UNSTAGED         │ on disk (modified)         │
├─────────────────┼────────────────┼──────────────────┼────────────────────────────┤
│ git reset       │ B (moved back) │ B (reset to B)   │ B (OVERWRITTEN)            │
│   --hard HEAD~1 │                │                  │ C's changes GONE from disk │
└─────────────────┴────────────────┴──────────────────┴────────────────────────────┘
```

### Diagram 3: `git revert` vs `git reset` — history impact

```
HISTORY REWRITING (reset):                HISTORY PRESERVING (revert):
  
  A──B──C                                   A──B──C──D
     ^                                               ^
  (git reset HEAD~1 — C disappears          D = "Revert C"
   from branch tip, still in objects)       C still in history, D cancels it
  
  Safe for: local-only branches             Safe for: pushed/shared branches
  Unsafe for: shared branches               Works on: any branch, any time
```

### Diagram 4: `git restore` source options

```
THREE POSSIBLE SOURCES for git restore:

  Working Dir ──── reads from ────► INDEX  (default: git restore file)
  Index       ──── reads from ────► HEAD   (git restore --staged file)
  Working Dir ──── reads from ────► HEAD   (git restore --staged --worktree file)
  Working Dir ──── reads from ────► <sha>  (git restore --source=<sha> file)
  Index       ──── reads from ────► <sha>  (git restore --staged --source=<sha> file)
  
Direction of arrows: target zone ◄──── data flows from source zone
```

### Diagram 5: The complete undo decision tree

```
Do you need to undo something?
│
├─► Is it ONLY untracked files (new files never git-added)?
│       └─► git clean -fd  (WARNING: irreversible)
│
├─► Is it only WORKING DIRECTORY changes (unstaged edits to tracked files)?
│       └─► git restore <file>  or  git restore .
│
├─► Is it only STAGED changes (files you git-added but didn't commit)?
│       └─► git restore --staged <file>
│
├─► Is it the LAST COMMIT and it was never pushed?
│       ├─► Keep changes staged?  → git reset --soft HEAD~1
│       ├─► Keep changes unstaged? → git reset --mixed HEAD~1  (default)
│       └─► Nuke all changes?      → git reset --hard HEAD~1
│
├─► Is it a commit that WAS pushed (shared with others)?
│       └─► git revert <sha>  (creates a new "undo" commit — safe for shared branches)
│
└─► Is it a merge commit?
        └─► git revert -m 1 <merge-sha>  (must specify the mainline parent)
```

---

## RULE 5: HANDS-ON PROOF COMMANDS

### Proof Lab 1: The Three reset modes in action

```bash
mkdir undo-lab && cd undo-lab
git init
git config user.email "t@t.com" && git config user.name "T"

# Create base commit
echo "version A" > app.txt && git add . && git commit -m "Commit A"

# Create commit B on top
echo "version B" > app.txt && git commit -am "Commit B"

# Create commit C on top
echo "version C" > app.txt && git commit -am "Commit C"

# PROVE IT: We have 3 commits
git log --oneline
# [sha3] Commit C
# [sha2] Commit B
# [sha1] Commit A

SHA_C=$(git rev-parse HEAD)
echo "SHA_C = $SHA_C"

# ─────────────────────────────────
# TEST: git reset --soft
# ─────────────────────────────────
git reset --soft HEAD~1

# PROVE IT: Branch moved back
git log --oneline
# [sha2] Commit B
# [sha1] Commit A

# PROVE IT: Working dir still has C's content
cat app.txt
# version C

# PROVE IT: Index still has C's changes (staged)
git status
# Changes to be committed:
#   modified: app.txt

# PROVE IT: ORIG_HEAD was written
cat .git/ORIG_HEAD
# [same as SHA_C above]

# PROVE IT: Commit C object still exists in .git/objects
git cat-file -t $SHA_C
# commit  ← still there!

# Restore to C for next test
git reset --hard $SHA_C

# ─────────────────────────────────
# TEST: git reset --mixed (default)
# ─────────────────────────────────
git reset HEAD~1   # --mixed is default

# PROVE IT: Branch moved back
git log --oneline
# [sha2] Commit B

# PROVE IT: Working dir has C's content
cat app.txt
# version C

# PROVE IT: Index was reset (changes are UNSTAGED now)
git status
# Changes not staged for commit:
#   modified: app.txt

git reset --hard $SHA_C  # restore for next test

# ─────────────────────────────────
# TEST: git reset --hard
# ─────────────────────────────────
git reset --hard HEAD~1

# PROVE IT: Branch moved back
git log --oneline
# [sha2] Commit B

# PROVE IT: Working dir was OVERWRITTEN (C's changes gone)
cat app.txt
# version B  ← back to B's content!

# PROVE IT: Index clean
git status
# nothing to commit, working tree clean

# PROVE IT: Can still recover C (it's in .git/objects until GC)
git cat-file -t $SHA_C
# commit  ← still there!

# PROVE IT: Recover with ORIG_HEAD
git reset --hard ORIG_HEAD
cat app.txt
# version C  ← back! ORIG_HEAD saved us.
```

### Proof Lab 2: `git revert` internals

```bash
cd undo-lab

# Add something to revert
echo "bad feature" >> app.txt
git add app.txt
git commit -m "Add bad feature"
BAD_SHA=$(git rev-parse HEAD)

echo "good feature" >> app.txt
git commit -am "Add good feature"

# PROVE IT: bad feature is in history
git log --oneline
# [sha] Add good feature
# [sha] Add bad feature    ← we want to undo this WITHOUT rewinding
# [sha] Commit C (or wherever we are)

# Revert the bad commit
git revert $BAD_SHA --no-edit

# PROVE IT: Revert commit was created
git log --oneline
# [sha] Revert "Add bad feature"  ← NEW commit
# [sha] Add good feature
# [sha] Add bad feature           ← STILL IN HISTORY

# PROVE IT: The bad feature's line is gone from the file
# (The good feature line is still there because revert only inverts $BAD_SHA's changes)
cat app.txt
# version C (from previous)
# good feature    ← preserved
# (no "bad feature" line — reverted)

# PROVE IT: Look at the revert commit object
git cat-file -p HEAD
# tree [sha]
# parent [sha of "Add good feature"]
# author ...
# committer ...
#
# Revert "Add bad feature"
#
# This reverts commit [BAD_SHA].
```

### Proof Lab 3: `git restore` operations

```bash
cd undo-lab

# Make some changes
echo "working dir change" >> app.txt
echo "new staged file" > staged.txt
git add staged.txt

git status
# Changes to be committed:
#   new file: staged.txt
# Changes not staged for commit:
#   modified: app.txt

# PROVE IT: Restore working dir change (unstaged)
git restore app.txt
cat app.txt
# (no "working dir change" line — discarded)

# PROVE IT: Index is unchanged (staged.txt still staged)
git status
# Changes to be committed:
#   new file: staged.txt

# PROVE IT: Unstage the staged file
git restore --staged staged.txt
git status
# Untracked files:
#   staged.txt   ← moved from staged back to untracked

# PROVE IT: Restore file to a specific commit's version
git log --oneline
SOME_SHA=$(git log --oneline | tail -3 | head -1 | awk '{print $1}')
git restore --source=$SOME_SHA app.txt
cat app.txt    # now matches that commit's version of app.txt
# (does NOT commit this change — you'd need git add && git commit)
git status
# modified: app.txt  ← change is in working dir but not staged
```

### Proof Lab 4: `git clean` — the irreversible one

```bash
cd undo-lab

# Create untracked files
echo "temp file" > temp.txt
mkdir build && echo "compiled" > build/output.bin
echo "ignored" > .env   # pretend this is gitignored

# PROVE IT: See what would be deleted WITHOUT deleting
git clean -fdn
# Would remove build/
# Would remove temp.txt

# PROVE IT: .env not shown (not in -x mode)
# (gitignored files not shown unless you add -x)

# Now really clean
git clean -fd
ls
# (temp.txt and build/ are gone — IRREVERSIBLE)

# PROVE IT: No ORIG_HEAD was written — no recovery mechanism
cat .git/ORIG_HEAD  # still the old sha from reset, clean doesn't update it
```

---

## RULE 6: SYNTAX BREAKDOWN (Deep)

### `git reset`

```
git reset [--soft | --mixed | --hard | --merge | --keep] [<commit>] [--] [<pathspec>...]
│         │                                               │           │    │
│         │                                               │           │    └─ specific files
│         │                                               │           └─ separator: treat 
│         │                                               │              next args as paths
│         │                                               └─ target: HEAD~1, SHA, branch name,
│         │                                                  tag, ORIG_HEAD. Default: HEAD
│         └─ mode flags:
│            --soft:  move branch pointer only
│            --mixed: move branch pointer + reset index (DEFAULT)
│            --hard:  move branch pointer + reset index + reset working dir
│            --merge: like --hard but preserves uncommitted changes that 
│                     would NOT be overwritten (useful when aborting a merge)
│            --keep:  like --mixed but aborts if it would overwrite 
│                     uncommitted local changes
└─ git binary

SPECIAL FORM: git reset <pathspec>
  → Unstages the specified file (same as git restore --staged <file>)
  → Equivalent to: git reset --mixed HEAD -- <file>
  → Does NOT move the branch pointer (no <commit> argument)
  
EXAMPLES:
  git reset HEAD~3          # --mixed: go back 3 commits, keep files modified
  git reset --soft HEAD~1   # undo last commit, keep changes staged
  git reset --hard a1b2c3   # go to specific commit, nuke working dir
  git reset HEAD app.js     # unstage app.js (equivalent to git restore --staged)
  git reset ORIG_HEAD       # undo a rebase, merge, or reset
```

### `git revert`

```
git revert [--no-edit] [--no-commit | -n] [-m <parent-number>] <commit>...
│          │            │                  │                    │
│          │            │                  │                    └─ one or more commits
│          │            │                  │                       to revert (in order)
│          │            │                  └─ REQUIRED for merge commits:
│          │            │                     -m 1 means "parent 1 is the mainline"
│          │            │                     (parent 1 = the branch you merged INTO)
│          │            └─ --no-commit (-n): apply the inverse patch but don't commit
│          │               lets you combine multiple reverts into one commit
│          └─ --no-edit: don't open the editor for the revert commit message
│                        uses the default "Revert 'original message'" format
└─ git binary

EXAMPLES:
  git revert HEAD              # revert the last commit
  git revert a1b2c3            # revert a specific commit
  git revert HEAD~3..HEAD      # revert a range (HEAD~2, HEAD~1, HEAD — three commits)
  git revert -m 1 M            # revert a merge commit M, keeping parent 1 as mainline
  git revert -n a1b2c3 b2c3d4  # apply both reverts without committing, then commit once
```

### `git restore`

```
git restore [--source=<tree>] [--staged] [--worktree] [--] <pathspec>...
│           │                  │          │             │    │
│           │                  │          │             │    └─ files to restore
│           │                  │          │             └─ separator
│           │                  │          └─ --worktree (-W): restore the working directory
│           │                  │             (default when --staged not specified)
│           │                  └─ --staged (-S): restore the index (staging area)
│           │                     Can be combined with --worktree
│           └─ --source=<tree> (-s): where to restore FROM
│              Default: index when restoring WD, HEAD when restoring index
│              Examples: --source=HEAD, --source=HEAD~2, --source=a1b2c3
└─ git binary

EXAMPLES:
  git restore app.js              # discard working dir changes (restores from index)
  git restore --staged app.js     # unstage (restores index from HEAD)
  git restore --staged --worktree app.js  # unstage AND discard WD changes (from HEAD)
  git restore --source=HEAD~2 app.js  # bring old version to WD (not staged)
  git restore .                   # discard ALL working dir changes (entire tree)
  git restore --staged .          # unstage everything
```

### `git clean`

```
git clean [-f | --force] [-d] [-n | --dry-run] [-i | --interactive] [-x | -X] [<path>...]
│         │               │    │                 │                    │     │    │
│         │               │    │                 │                    │     │    └─ limit to path
│         │               │    │                 │                    │     └─ -X: remove ONLY 
│         │               │    │                 │                    │          ignored files
│         │               │    │                 │                    └─ -x: also remove 
│         │               │    │                 │                         ignored files
│         │               │    │                 └─ interactive: choose what to remove
│         │               │    └─ --dry-run: show what WOULD be removed (ALWAYS run this first!)
│         │               └─ -d: also remove untracked directories
│         └─ --force: REQUIRED unless clean.requireForce=false in config
│            Safety gate so you don't accidentally nuke files
└─ git binary

WORKFLOW (always):
  git clean -fdn    # dry run first — see what would be deleted
  git clean -fd     # then actually clean if dry run looks right
```

---

## RULE 7: BEGINNER → PRODUCTION EXAMPLE

### Beginner: Undo a bad commit before pushing

```bash
# You committed a debug console.log and don't want it in your PR
git log --oneline
# a1b2c3 Add user auth  ← good
# b2c3d4 WIP debug      ← bad, want to undo

# Option 1: Keep the debug code but uncommit it (use it, just don't commit)
git reset --mixed HEAD~1
# Now: "WIP debug" changes are in working dir, unstaged
# You can remove the console.log, then re-commit properly

# Option 2: Nuke it completely
git reset --hard HEAD~1
# Now: back to "Add user auth" — debug commit gone
```

### Beginner: Accidentally staged the wrong file

```bash
git add .   # oops — staged everything including .env
git status
# Changes to be committed:
#   new file: .env          ← should NOT be committed
#   modified: server.js     ← this is fine

# Unstage only .env
git restore --staged .env
git status
# Changes to be committed:
#   modified: server.js
# Untracked files:
#   .env
```

### Production: Reverting a bad deploy on main

**Scenario**: You're on-call. A commit pushed to main 2 hours ago introduced a bug that's
affecting 30% of users. The commit has already been deployed and is being used by 3 other
engineers who pulled from main.

```bash
# NEVER use git reset on main — it would rewrite shared history
# Use git revert — safe, preserves history, creates an audit trail

# Find the bad commit
git log --oneline main | head -20
# a1b2c3 Update payment timeout config  ← THIS ONE broke things
# b2c3d4 Add retry logic to webhook
# ...

# Revert it
git revert a1b2c3 --no-edit
# Creates: "Revert 'Update payment timeout config'"

# Push immediately
git push origin main

# Everyone can now git pull and get the fix
# The audit trail shows:
# a1b2c3 Update payment timeout config   ← bad commit still visible
# x1y2z3 Revert "Update payment timeout config"  ← the fix
```

### Production: Reverting a merge commit

**Scenario**: Feature branch `feature/v2-checkout` was merged to main via a merge commit.
The feature is causing data corruption. You need to revert the ENTIRE feature.

```bash
# Find the merge commit
git log --oneline --graph main | head -10
# *   M5f3a Merge branch 'feature/v2-checkout'
# |\
# | * d4e5f6 Final checkout v2 commit
# | * c3d4e5 Add v2 cart logic
# | * ...
# * | b2c3d4 Other main work
# |/
# * a1b2c3 Base

# What does -m 1 mean?
git cat-file -p M5f3a
# tree [sha]
# parent b2c3d4   ← parent 1: main (the mainline you want to KEEP)
# parent d4e5f6   ← parent 2: feature/v2-checkout (the work you want to UNDO)
# ...

# Revert the merge, keeping parent 1 (main) as mainline
git revert -m 1 M5f3a
# Creates a commit that effectively removes all feature/v2-checkout changes
# while preserving other main commits

# The result:
git log --oneline main | head -5
# R7g8h Revert "Merge branch 'feature/v2-checkout'"
# M5f3a Merge branch 'feature/v2-checkout'
# ...
```

**Important caveat**: After reverting a merge commit with `-m 1`, if you later want to
re-merge that feature after fixing the bug, Git will think it's already merged. You must
`git revert` the revert commit first:

```bash
# Fix the bug in feature/v2-checkout, then:
git revert R7g8h          # "Revert the revert" — re-enables the feature
git merge feature/v2-checkout   # now merge again with the fix
```

---

## RULE 8: MISTAKES → ROOT CAUSE → FIX → PREVENTION

### Mistake 1: `git reset --hard` on a shared branch

**What happened**: You ran `git reset --hard HEAD~3` on `main` to undo 3 commits.
You pushed. Colleagues pull and see "your branch is 3 commits ahead of origin/main."

**Root cause**: `git reset --hard` on a pushed branch rewrites history. Everyone who
has those 3 commits in their local copy now has diverged history.

**Fix**:
```bash
# For each affected team member:
git fetch origin
git reset --hard origin/main  # or git pull

# For you — undo your reset (commits still in reflog):
git reflog
# HEAD@{0}: reset: moving to HEAD~3
# HEAD@{1}: commit: The commit you wanted to keep
# ...
git reset --hard HEAD@{1}  # go back to before your reset
git push --force-with-lease  # only if you haven't told others yet
```

**Prevention**: NEVER use `git reset` on branches others have pulled. Use `git revert`.

---

### Mistake 2: `git clean -fd` deleted a file you needed

**What happened**: You ran `git clean -fd` without dry-running first. Deleted your
manually created config file that wasn't tracked by Git.

**Root cause**: `git clean` permanently removes files from disk. No ORIG_HEAD, no reflog,
no recovery mechanism for untracked files.

**Fix**: Check your editor's "local history" feature. Or check system backup. 
Git itself **cannot** recover untracked files after clean.

**Prevention**:
```bash
# ALWAYS dry-run first
git clean -fdn   # shows what would be deleted
# Then decide if it's safe:
git clean -fd    # only if dry-run output looks right

# Or use interactive mode:
git clean -fdi   # review and select each file
```

---

### Mistake 3: `git revert` on wrong commit (not the one you wanted)

**What happened**: `git log` shows commits newest-first. You counted wrong.

**Fix**:
```bash
# Undo the revert itself (git revert is just a commit — use revert on it):
git revert HEAD   # "revert the revert"
# Now back to where you were. Try again with the correct SHA.
```

**Prevention**: Use `--dry-run` flag or inspect the SHA before reverting:
```bash
git show a1b2c3   # inspect the commit before reverting it
git revert a1b2c3
```

---

### Mistake 4: Forgetting `--no-edit` in scripts

**What happened**: You ran `git revert a1b2c3` in a CI script. Git paused waiting for
a commit message editor that can't open in CI.

**Fix** (add to script):
```bash
git revert a1b2c3 --no-edit
# OR use GIT_EDITOR=true to make editor a no-op
```

---

### Mistake 5: Using `git reset` on a file path without understanding the effect

**What happened**: You ran `git reset HEAD~1 -- app.js` thinking it would undo one commit's
changes to app.js from the working directory.

**Root cause**: `git reset <sha> -- <file>` only restores the INDEX (staging area) to
match `<sha>`'s version of the file. It does NOT touch the working directory.

**What actually happens**:
```bash
git reset HEAD~1 -- app.js
# Effect: index now has HEAD~1's version of app.js
# Working directory: UNCHANGED (still has your current content)
# git status shows: "Changes to be committed: app.js"
# This is CONFUSING — the staged version is now different from both WD and last commit
```

**The correct command** for restoring a file's working directory content:
```bash
git restore --source=HEAD~1 app.js    # restores WD to HEAD~1 version (not staged)
git restore --staged --source=HEAD~1 app.js  # restores both WD and index
```

---

### Mistake 6: `git revert` a merge commit without `-m`

**What happened**: You ran `git revert <merge-sha>` without `-m 1`.

**Error**:
```
error: commit [sha] is a merge but no -m option was given.
```

**Root cause**: A merge commit has 2+ parents. Git doesn't know which side is the
"mainline" (the work you want to keep) vs the side you want to reverse.

**Fix**:
```bash
# Find which parent is parent 1 (the mainline):
git cat-file -p <merge-sha>
# parent [sha1]   ← this is parent 1 (usually the branch you merged INTO)
# parent [sha2]   ← this is parent 2 (the branch you merged FROM)

# Then revert properly:
git revert -m 1 <merge-sha>   # -m 1 means "keep parent 1 as mainline"
```

---

## RULE 9: BYTE-LEVEL INTERNALS

### What `git reset --hard HEAD~1` actually changes at the byte level

```
BEFORE:
  File: .git/refs/heads/main
  Contents: "c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2\n"  (41 bytes: 40-char SHA + newline)

AFTER git reset --hard HEAD~1:
  File: .git/refs/heads/main
  Contents: "b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1\n"  (41 bytes: different SHA)
  
  File: .git/ORIG_HEAD  (WRITTEN by reset)
  Contents: "c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2\n"  (the OLD tip)

That's it. One 41-byte file overwrite. The commit objects are untouched.
The working directory is then updated by reading the new HEAD's tree and applying it.
```

### How `git revert` creates its commit object

```
ORIGINAL COMMIT (a1b2c3):
  tree   old_tree_sha
  parent parent_sha
  author Alice <a@example.com> 1748000000 +0000
  
  Add bad feature

REVERT COMMIT (created by git revert a1b2c3):
  tree   new_tree_sha       ← tree is built with the bad feature's changes REMOVED
  parent [current HEAD sha] ← parent is the CURRENT HEAD, not a1b2c3
  author Alice <a@example.com> 1748600000 +0000
  
  Revert "Add bad feature"
  
  This reverts commit a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1.
```

The revert commit is identical in format to any other commit — it's just that its
`tree` has been constructed by applying the INVERSE diff of the original commit.

### `git restore --source` internals

```bash
git restore --source=HEAD~2 app.js

# What Git does internally:
# 1. Reads .git/HEAD → .git/refs/heads/main → current commit SHA
# 2. Walks back 2 parent pointers → finds HEAD~2's commit SHA
# 3. Reads that commit's tree object
# 4. Finds the blob SHA for app.js in that tree
# 5. Reads the blob from .git/objects/
# 6. Decompresses and writes to working directory app.js
# 
# No index modification (unless --staged is also specified)
# No commit created
# No ORIG_HEAD written (non-destructive to history)
```

---

## THE ORIGIN OF `git restore` AND `git switch`

Before Git 2.23 (August 2019), you had to use `git checkout` for everything:

```bash
# OLD WAY (git checkout doing multiple unrelated things):
git checkout <branch>           # switch branch
git checkout -b <branch>        # create and switch branch  
git checkout -- <file>          # discard working dir changes
git checkout <sha> -- <file>    # restore file from commit

# NEW WAY (clear separation of concerns):
git switch <branch>             # switch branch (Topic 08)
git switch -c <branch>          # create and switch branch
git restore <file>              # discard working dir changes
git restore --source=<sha> <file>   # restore file from commit
```

`git checkout -- <file>` still works (backward compatibility), but it's deprecated 
in favor of `git restore`. On older systems, you'll still see `git checkout` used 
for file restoration.

---

## RECOVERY MATRIX: What Can Be Recovered and How

| Lost work | Recoverable? | How |
|-----------|-------------|-----|
| Committed then `reset --hard` | YES (30 days) | `git reflog` → `git reset --hard <sha>` |
| Stashed then lost | YES (30 days) | `git stash list` or `git fsck --unreachable` |
| Committed then `git gc` ran | MAYBE | Only if within expiry window, check reflog first |
| Untracked file deleted by `git clean` | NO | Not in Git at all — check OS backup |
| Staged change then `git checkout -- <file>` | NO (pre-2.23) | Staged → index → overwritten WD |
| Staged change then `git restore <file>` | YES | File is still in index! Use `git restore --staged <file>` first |
| Changes in working dir, never staged | NO | Git never knew about them |
| Commit on detached HEAD that you switched away from | YES (30 days) | `git reflog` → find the SHA |

### The safety hierarchy

```
MOST PROTECTED:
  Committed data              ← in .git/objects/, reflog holds for 30 days
  ↓
  Staged data (index)         ← in .git/index as blob references, somewhat protected
  ↓
  Unstaged tracked changes    ← only in working directory
  ↓
LEAST PROTECTED:
  Untracked files             ← only on disk, Git has no record of them
```

**Rule of thumb**: If you've run `git add`, Git knows about it. If you haven't, Git can't save you.

---

## RULE 10: MENTAL MODEL CHECKPOINT

**1.** You run `git reset --mixed HEAD~2`. Name every file in `.git/` that changes.
Name every zone (working dir, index, commits) that changes and which does NOT change.

**2.** Your colleague force-pushed to `main` and `git log` shows 5 commits you don't
recognize. `git log origin/main` shows 5 different commits. What happened, and what's
the safest recovery path for your local repo?

**3.** What is the difference between `git restore app.js` and `git restore --staged app.js`?
What is each one's SOURCE (where does the content come from)?

**4.** You need to undo a commit that was pushed to main 3 days ago and has 5 commits
built on top of it. Should you use `git reset` or `git revert`? Why? What's the exact
command?

**5.** Why does `git revert -m 1 <merge-sha>` require the `-m 1` flag? What does
"parent 1" mean, and how do you find out which commit is parent 1?

**6.** You ran `git reset --hard HEAD~1` and then realized you needed those changes.
What command recovers them, and what is the time limit on this recovery?

**7.** You have 3 untracked files and 2 modified tracked files. You run `git clean -fd`.
Which files are deleted? Which survive? Why?

**8.** What does `git reset HEAD app.js` do? How is it different from `git restore --staged app.js`?
(Hint: they do the same thing — but state why they're equivalent.)

**9.** `git revert HEAD~3..HEAD` — how many commits does this revert? In what order are
they reverted? Why does the order matter?

**10.** Describe the physical state of `.git/` immediately after running
`git revert a1b2c3 --no-commit` (with the `-n` flag). What file contains the
in-progress state? What is in the working directory and index?

---

## RULE 12: QUICK REFERENCE CARD

| Command | What It Does | Reversible? |
|---------|-------------|-------------|
| `git reset --soft HEAD~N` | Move branch back N commits; keep changes STAGED | YES (ORIG_HEAD) |
| `git reset --mixed HEAD~N` | Move branch back N commits; keep changes UNSTAGED | YES (ORIG_HEAD) |
| `git reset --hard HEAD~N` | Move branch back N commits; NUKE working dir | YES (reflog, 30 days) |
| `git reset ORIG_HEAD` | Undo the last reset/merge/rebase | YES (check reflog) |
| `git revert <sha>` | Create new commit that reverses `<sha>` | YES (revert the revert) |
| `git revert -m 1 <merge-sha>` | Revert a merge commit, keeping parent 1 | YES |
| `git revert -n <sha>` | Apply revert without committing | YES (git checkout .) |
| `git restore <file>` | Discard working dir changes (from index) | NO (changes gone) |
| `git restore --staged <file>` | Unstage file (from HEAD) | YES (git add again) |
| `git restore --source=<sha> <file>` | Bring old version to working dir | YES (restore again) |
| `git restore .` | Discard ALL working dir changes | NO |
| `git clean -fdn` | DRY RUN: show untracked files that would be deleted | N/A |
| `git clean -fd` | Delete all untracked files and dirs | NO (gone from disk) |
| `git clean -fdx` | Delete untracked + ignored files | NO |
| `git clean -fdi` | Interactive clean: choose each file | NO (once confirmed) |

| Safety File | Written By | Content |
|-------------|-----------|---------|
| `.git/ORIG_HEAD` | reset, merge, rebase | SHA of tip before the operation |
| `.git/REVERT_HEAD` | revert (when conflict stops it) | SHA of commit being reverted |
| `.git/MERGE_HEAD` | merge (when in progress) | SHA of commit being merged in |

| Scenario | Command |
|----------|---------|
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` |
| Undo last commit, keep changes unstaged | `git reset HEAD~1` |
| Undo last commit, discard changes | `git reset --hard HEAD~1` |
| Undo a push (public commit) | `git revert <sha>` |
| Unstage a file | `git restore --staged <file>` |
| Discard working dir changes | `git restore <file>` |
| Bring old version of file | `git restore --source=<sha> <file>` |
| Delete untracked files | `git clean -fd` (dry run with -n first) |
| Recover from bad reset | `git reset --hard ORIG_HEAD` |
| Recover any commit | `git reflog` → `git reset --hard <sha>` |

---

## RULE 13: WHEN WOULD I USE THIS AT WORK?

### Scenario 1: The "oops, wrong branch" commit

```
You: Made 2 commits on main instead of your feature branch
Problem: main is protected — you can't push directly

Solution:
  1. Note the SHAs of your commits: git log --oneline -2
  2. git reset --hard HEAD~2      (undo on main — safe, unpushed)
  3. git checkout -b feature/my-fix  (create the right branch)
  4. git cherry-pick <sha1> <sha2>   (apply your commits here)
  
Done in under 2 minutes. No one knows it happened.
```

### Scenario 2: Cleaning up before opening a PR

```
You: 12 commits including "WIP", "fix", "oops try again", "add console.log for debug"
Team: "Please clean up your branch before requesting review"

Solution:
  git rebase -i origin/main   (covered in Topic 11)
  
But ALSO useful:
  git reset --soft origin/main  
  # ← moves HEAD back to where main is, keeps ALL your changes staged
  # Now you can recommit as 3 clean, atomic commits
  # Simpler than interactive rebase for fully restructuring commits
```

### Scenario 3: Emergency rollback on prod

```
Production is broken. The culprit commit was pushed 30 min ago.
3 more commits have been added since then.

WRONG approach: git reset --hard (rewrites shared history)
RIGHT approach:
  git revert <bad-sha> --no-edit
  git push origin main
  
Takes < 1 minute. History preserved. Other engineers' work unaffected.
Post-incident review can see exactly what happened and when.
```

### Scenario 4: Removing a file committed by mistake

```
You: Committed .env file with API keys by accident
Problem: Keys are now in git history — rotating keys is mandatory

IMMEDIATE: Remove from current state
  git rm --cached .env
  echo ".env" >> .gitignore
  git commit -m "Remove .env from tracking"
  
THEN: Remove from history (because keys are still visible in old commits)
  git filter-repo --path .env --invert-paths  (Topic 25 covers this)
  OR use GitHub's secret scanning which auto-alerts and can auto-revoke for some providers
  
ALWAYS: Rotate the leaked keys — assume they are compromised
```

### Scenario 5: Recovering from "I didn't mean to rebase"

```
You: Ran git rebase main and it turned your 8 commits into chaos
Problem: Conflicts everywhere, history looks wrong

Solution:
  git rebase --abort      # if still in progress
  
  OR if rebase "completed" but looks wrong:
  git reset --hard ORIG_HEAD  # immediately after
  
  OR if you did other things since:
  git reflog show HEAD | head -5   # find the pre-rebase SHA
  git reset --hard HEAD@{N}        # restore to that SHA
```

### Scenario 6: Selectively undoing a specific file change from an old commit

```
You: A commit from 2 weeks ago introduced a regression in config.yaml
     You want to revert JUST that file, not the whole commit

Solution:
  git log --oneline --follow -- config.yaml    # find the bad commit
  
  # Option A: Restore the file to before that commit
  git restore --source=<bad-sha>~1 config.yaml
  git add config.yaml
  git commit -m "Revert config.yaml to pre-regression state"
  
  # Option B: Use revert but only for that file (no built-in command — use checkout)
  git show <bad-sha>:config.yaml > config.yaml   # extract old version
  git add config.yaml
  git commit -m "Restore config.yaml: revert regression from <bad-sha>"
```

---

## SUMMARY: The Mental Model in One Paragraph

Every undo in Git is about moving one or more of the three trees (working directory,
index, HEAD/branch pointer) backward to an earlier state. `git reset` is the most
powerful — it can move the branch pointer alone (`--soft`), the branch pointer and
index together (`--mixed`), or all three zones simultaneously (`--hard`). The key
insight is that commit objects are NEVER deleted by reset — they remain in `.git/objects/`
and are accessible via `git reflog` until garbage collection runs (30 days by default).
`git revert` is the "public-safe" undo: instead of erasing history, it creates a new
commit that exactly reverses the chosen commit's changes. `git restore` handles
working-directory and index restoration without touching commit history at all.
`git clean` is the one truly irreversible operation — it deletes untracked files that
Git has never seen, with no recovery path.

---

*Topic 12 of 26 — Next: Topic 13: Stashing (stash internals, pop, apply, named stashes, untracked)*
