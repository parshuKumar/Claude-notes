# Topic 13: Stashing — The Temporary Shelf
### (git stash internals, pop, apply, named stashes, untracked files, partial stash)

---

## CONNECTION TO PREVIOUS TOPICS (Rule 11)

Stash is built entirely on the object model you already know.

- **Topic 02**: Every stash is stored as a **commit object** in `.git/objects/`. Specifically,
  stash creates a special commit with TWO or THREE parents — one for the index state and one
  for the working directory state. This is the most surprising fact about stash: it reuses
  Git's own commit infrastructure, not some special storage format.

- **Topic 05**: You know the three trees (working dir, index, HEAD). Stash saves the working
  directory and index trees, resets both zones back to HEAD, then can restore them later. It is
  a controlled dump-and-restore of up to two of the three trees.

- **Topic 08**: You know `refs/` contains pointers. Stash uses a special ref:
  `.git/refs/stash` — a file pointing to the most recent stash commit. Older stashes are
  in the reflog at `refs/stash`.

- **Topic 12**: You know `git reset --hard HEAD` resets working dir and index to HEAD.
  `git stash` essentially does this after saving your work — it's a safe "clean slate"
  without losing anything.

- **Topic 03**: You know the reflog. `git stash list` reads the reflog of `refs/stash`.
  Each stash entry is just a reflog entry with a commit object behind it.

---

## RULE 1: ELI5 ANCHOR — The Coat-Check Counter

Imagine you're working at your desk (working directory) with papers spread everywhere.
Your boss suddenly says "drop everything, I need you in a meeting NOW."

You can't commit to filing everything officially — the work is half-done.
You CAN hand all your papers to the coat-check counter at the door.
The coat-check gives you a ticket stub (the stash SHA).

Now you go into the meeting with a clean desk.
After the meeting, you show your ticket and get all your papers back — exactly as you left them.

That's stash: a temporary, structured save of your in-progress work so you can come back to it.

```
BEFORE STASH:                          AFTER STASH:
  Working dir: modified files    →      Working dir: clean (matches HEAD)
  Index: staged files            →      Index: clean (matches HEAD)
  HEAD: last commit              →      HEAD: unchanged (same last commit)
  
                                        .git/refs/stash: → WIP commit (your papers)
```

---

## RULE 2: PHYSICAL ARCHITECTURE FIRST

### What `.git/` looks like with stashes

```
.git/
├── HEAD                        ← "ref: refs/heads/feature" (unchanged by stash)
├── index                       ← reset to HEAD state after stash save
├── refs/
│   ├── heads/
│   │   ├── main
│   │   └── feature             ← unchanged — branch pointer untouched by stash
│   └── stash                   ← ← ← THE STASH POINTER
│                                      Contains SHA of most recent stash commit
│                                      Example: "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8\n"
├── logs/
│   └── refs/
│       └── stash               ← ← ← THE STASH LOG (list of all stashes)
│                                      Each line = one stash entry
│                                      This is what `git stash list` reads
└── objects/
    ├── a1/b2c3...              ← stash "WIP commit" (working directory tree)
    ├── b2/c3d4...              ← stash "index commit" (staged changes)
    └── c3/d4e5...              ← stash base commit (= HEAD at time of stash)
```

### The stash commit structure — the surprising truth

A stash is NOT a single commit. It is a **commit with special parents**:

```
STASH COMMIT STRUCTURE:

  [WIP commit] ← .git/refs/stash points here
       │
       ├── parent 1: HEAD commit at time of stash (the base)
       │             (what your branch was at when you stashed)
       │
       ├── parent 2: [index commit]
       │             (a commit whose tree = your staged files at stash time)
       │
       └── (optionally) parent 3: [untracked commit]
                         (only exists if you used --include-untracked or --all)
                         (a commit whose tree = your untracked files)

The WIP commit's OWN tree = your working directory state (all modified tracked files).
The index commit's tree = your staged state.
```

### `.git/logs/refs/stash` — the stash list file (raw format)

```
# Each line format:
# <old-sha> <new-sha> <author> <timestamp> <tz> <reflog-message>

0000000000000000000000000000000000000000 a1b2c3d4... Alice <a@dev.com> 1748600000 +0530 WIP on feature: abc1234 Add user login
a1b2c3d4... b2c3d4e5... Alice <a@dev.com> 1748610000 +0530 WIP on feature: abc1234 Add user login

# Line 1: first stash (stash@{1} in git stash list — oldest shown last)
# Line 2: second stash added later (stash@{0} — most recent is 0)
# The file is read REVERSED: newest at top of 'git stash list'
```

### Before and after `git stash push` — full `.git/` state change

```
BEFORE:
  .git/HEAD             → ref: refs/heads/feature
  .git/refs/heads/feature → SHA of commit C (last commit)
  .git/index            → contains staged changes (different from HEAD)
  Working directory     → contains modified tracked files

AFTER git stash push:
  .git/HEAD             → ref: refs/heads/feature (UNCHANGED)
  .git/refs/heads/feature → SHA of commit C (UNCHANGED — stash never touches branch)
  .git/refs/stash       → SHA of new WIP commit  (WRITTEN)
  .git/logs/refs/stash  → new line appended       (MODIFIED)
  .git/index            → reset to match commit C (MODIFIED — staged changes cleared)
  Working directory     → reset to match commit C (MODIFIED — working changes cleared)
  .git/objects/         → 2-3 new commit objects written
```

---

## RULE 3: EXACT DATA FLOW — Every Stash Command

### `git stash push` (same as legacy `git stash`)

```
COMMAND: git stash push -m "WIP: half-done auth"

STEP 1: CAPTURE INDEX STATE
  READS:   .git/index (current staged changes)
  COMPUTES: git write-tree → index tree SHA
  WRITES:  New tree object in .git/objects/
  WRITES:  New "index commit" object:
             tree = index tree SHA
             parent = SHA of HEAD
             author/committer = current user
             message = "index on feature: abc1234 Add user login"

STEP 2: CAPTURE WORKING DIRECTORY STATE
  READS:   All modified tracked files in working directory
  COMPUTES: git write-tree (after adding all tracked modified files conceptually)
  WRITES:  New tree object capturing WD state
  WRITES:  New "WIP commit" object:
             tree = WD tree SHA
             parent 1 = SHA of HEAD
             parent 2 = SHA of index commit (from Step 1)
             author/committer = current user  
             message = "WIP on feature: abc1234 Add user login"
             (or your -m message: "WIP: half-done auth")

STEP 3: UPDATE STASH REF
  READS:   .git/refs/stash (current stash pointer, may not exist yet)
  WRITES:  .git/refs/stash ← SHA of new WIP commit
  WRITES:  .git/logs/refs/stash ← new reflog line

STEP 4: RESET WORKING STATE
  MODIFIES: .git/index ← reset to match HEAD
  MODIFIES: Working directory ← reset to match HEAD
            (git reset --hard HEAD equivalent, but stash content is safely saved first)

NET EFFECT:
  Two new commit objects written to .git/objects/
  Working dir and index are clean (match HEAD)
  .git/refs/stash points to the WIP commit
```

### `git stash list`

```
COMMAND: git stash list

READS:   .git/logs/refs/stash (the reflog file)
COMPUTES: Formats each entry with its index (stash@{0}, stash@{1}, ...)
OUTPUT:
  stash@{0}: WIP on feature: abc1234 Add user login  ← most recent
  stash@{1}: On main: save before hotfix             ← older

NO WRITES — read-only operation
```

### `git stash pop`

```
COMMAND: git stash pop

READS:   .git/refs/stash → SHA of most recent WIP commit
         The WIP commit object → its tree (WD state)
         The WIP commit's parent 2 → index commit → its tree (index state)

STEP 1: APPLY INDEX STATE
  READS:   index commit's tree
  MODIFIES: .git/index ← restores staged files

STEP 2: APPLY WORKING DIRECTORY STATE
  READS:   WIP commit's tree
  MODIFIES: Working directory ← restores modified files

STEP 3: REMOVE STASH ENTRY (if apply succeeded without conflicts)
  MODIFIES: .git/refs/stash ← advances to stash@{1} (previous stash)
  MODIFIES: .git/logs/refs/stash ← drops the top entry
  (The WIP commit objects remain in .git/objects/ — just no ref points to them now)

IF CONFLICT:
  STOPS with conflict markers in working directory
  .git/refs/stash is NOT modified (stash entry preserved)
  You resolve conflicts manually, then: git stash drop stash@{0}
```

### `git stash apply` (like pop but doesn't remove the stash)

```
COMMAND: git stash apply [stash@{N}]

IDENTICAL to pop EXCEPT:
  - Does NOT modify .git/refs/stash
  - Does NOT modify .git/logs/refs/stash
  - Stash entry remains after apply (you must manually git stash drop)

USE CASE: Apply same stash to multiple branches (e.g., cherry-pick a WIP across branches)
```

### `git stash drop`

```
COMMAND: git stash drop [stash@{N}]

READS:   .git/logs/refs/stash → finds the entry for stash@{N}
MODIFIES: .git/refs/stash → if dropping stash@{0}, advances to stash@{1}
          .git/logs/refs/stash → removes the specified line

The commit objects for the dropped stash become unreachable.
They remain in .git/objects/ until git gc runs (30 days default).
RECOVERY: See below — use git fsck to find them.
```

### `git stash clear`

```
COMMAND: git stash clear

MODIFIES: .git/refs/stash → deleted entirely
          .git/logs/refs/stash → deleted entirely (or emptied)

ALL stash commit objects become unreachable (pending GC).
This is the most dangerous stash operation — clears everything.
RECOVERY: git fsck --unreachable | grep commit → find the WIP commit SHAs
```

---

## RULE 4: ASCII ARCHITECTURE DIAGRAMS

### Diagram 1: The stash commit structure in the DAG

```
BEFORE STASH:
    A──B──C     ← feature branch (HEAD)
    
AFTER git stash push:
    A──B──C     ← feature branch (HEAD — UNCHANGED)
              \
               I ── W     ← stash commits
               │     │
               │     └── WIP commit (stash@{0})
               │          tree = working dir state
               │          parent 1 = C
               │          parent 2 = I
               │
               └── Index commit (not in stash list — internal)
                    tree = staged state
                    parent = C

.git/refs/stash → W (the WIP commit)
```

### Diagram 2: Multiple stashes — reflog chain

```
STASH HISTORY (newest first):

stash@{0}: WIP commit W1  ←── .git/refs/stash
               parent 1: commit C (HEAD when stashed)
               parent 2: index commit I1

stash@{1}: WIP commit W0  ←── reflog entry
               parent 1: commit B (HEAD when stashed — different branch state)
               parent 2: index commit I0

stash@{2}: WIP commit W-1 ←── reflog entry
               ...

The stash list is a reflog — same mechanism as HEAD's reflog in .git/logs/HEAD
```

### Diagram 3: `git stash pop` — the three-zone restoration

```
STASH@{0} contains:
  WIP commit tree (working dir state)
  Index commit tree (staged state)

git stash pop:

  WIP commit tree ──────────────────────────────► Working Directory
  Index commit tree ────────────────────────────► Index (.git/index)
  
  .git/refs/stash ──► now points to stash@{1} (previous stash)
  Working directory: was clean, now has your WIP changes back
  Index: was clean, now has your staged changes back
```

### Diagram 4: `git stash push --include-untracked` — the third parent

```
NORMAL stash (tracked files only):
  WIP commit
  ├── parent 1: HEAD
  └── parent 2: index commit

STASH WITH --include-untracked:
  WIP commit
  ├── parent 1: HEAD
  ├── parent 2: index commit
  └── parent 3: untracked commit
                 tree = all untracked files (new files never git-added)

STASH WITH --all:
  Same structure, but also includes .gitignore-matched files in untracked commit
```

### Diagram 5: stash@{N} addressing

```
.git/logs/refs/stash (bottom of file = oldest):

  line 0: 0000000 W-2 ... stash@{2} (oldest)
  line 1: W-2     W-1 ... stash@{1}
  line 2: W-1     W0  ... stash@{0} (most recent) ← .git/refs/stash points here

git stash list output (newest first):
  stash@{0}: WIP on feature: ...    ← line 2 above
  stash@{1}: WIP on main: ...       ← line 1 above  
  stash@{2}: On hotfix: ...         ← line 0 above

git stash pop          = apply + drop stash@{0}
git stash pop stash@{1} = apply + drop stash@{1}
```

---

## RULE 5: HANDS-ON PROOF COMMANDS

### Proof Lab 1: The stash IS a commit

```bash
mkdir stash-lab && cd stash-lab
git init
git config user.email "t@t.com" && git config user.name "Test"

# Create base commit
echo "original" > app.txt && git add . && git commit -m "Initial commit"
BASE_SHA=$(git rev-parse HEAD)

# Make changes: one staged, one unstaged
echo "staged change" >> app.txt && git add app.txt
echo "unstaged change" > wip.txt

# PROVE IT: Current state
git status
# Changes to be committed:
#   modified: app.txt
# Untracked files:
#   wip.txt   ← untracked, won't be stashed by default

# Stash
git stash push -m "My WIP changes"

# PROVE IT: Working dir is clean
git status
# nothing to commit, working tree clean

# PROVE IT: .git/refs/stash was written
cat .git/refs/stash
# [some SHA]

STASH_SHA=$(cat .git/refs/stash)
echo "Stash WIP commit: $STASH_SHA"

# PROVE IT: The stash IS a commit object
git cat-file -t $STASH_SHA
# commit

# PROVE IT: Look inside the WIP commit — it has TWO parents
git cat-file -p $STASH_SHA
# tree [sha]
# parent [SHA of Initial commit]    ← parent 1 = HEAD at stash time
# parent [SHA of index commit]      ← parent 2 = staged state
# author Test <t@t.com> ...
# committer Test <t@t.com> ...
#
# My WIP changes

# PROVE IT: Inspect the index commit (parent 2)
INDEX_COMMIT=$(git cat-file -p $STASH_SHA | grep "^parent" | tail -1 | awk '{print $2}')
git cat-file -p $INDEX_COMMIT
# tree [sha]
# parent [SHA of Initial commit]
# author Test <t@t.com> ...
#
# index on feature: [sha] Initial commit

# PROVE IT: The stash log file
cat .git/logs/refs/stash
# 0000000...0 [STASH_SHA] Test <t@t.com> ... WIP on main: [sha] My WIP changes

# PROVE IT: wip.txt was NOT stashed (untracked by default)
ls
# app.txt  wip.txt    ← wip.txt is still here (stash didn't touch untracked files)
```

### Proof Lab 2: Multiple stashes and the reflog

```bash
# Still in stash-lab, let's add more stashes

# Make another change
echo "second WIP" >> app.txt
git stash push -m "Second WIP"

echo "third WIP" >> app.txt
git stash push -m "Third WIP"

# PROVE IT: List stashes
git stash list
# stash@{0}: On main: Third WIP
# stash@{1}: On main: Second WIP
# stash@{2}: On main: My WIP changes

# PROVE IT: stash log file has 3 entries
wc -l .git/logs/refs/stash
# 3

# PROVE IT: Apply a specific stash (not the most recent)
git stash apply stash@{1}

git status
# modified: app.txt  ← second WIP changes applied

# PROVE IT: stash@{1} is STILL in the list (apply doesn't remove)
git stash list
# stash@{0}: On main: Third WIP
# stash@{1}: On main: Second WIP   ← still here!
# stash@{2}: On main: My WIP changes

# Reset and use pop instead
git restore .
git stash pop
# Applied stash@{0} AND removed it

git stash list
# stash@{0}: On main: Second WIP   ← now stash@{1} became stash@{0}
# stash@{1}: On main: My WIP changes
```

### Proof Lab 3: Stashing untracked files

```bash
# Create a new untracked file
echo "brand new file" > newfile.txt

git status
# Untracked files:
#   newfile.txt

# Normal stash — untracked files NOT included
git stash push -m "Test without untracked"
ls newfile.txt    # STILL HERE — not stashed

git stash drop    # clean up

# Stash WITH untracked files
git stash push --include-untracked -m "Test with untracked"
ls newfile.txt 2>/dev/null || echo "newfile.txt was stashed (gone from WD)"
# newfile.txt was stashed (gone from WD)

# PROVE IT: WIP commit now has 3 parents
STASH_SHA=$(cat .git/refs/stash)
git cat-file -p $STASH_SHA | grep "^parent"
# parent [sha1]    ← HEAD
# parent [sha2]    ← index commit
# parent [sha3]    ← untracked commit   ← NEW!

# PROVE IT: untracked commit contains newfile.txt
UNTRACKED_COMMIT=$(git cat-file -p $STASH_SHA | grep "^parent" | tail -1 | awk '{print $2}')
git ls-tree $(git cat-file -p $UNTRACKED_COMMIT | grep "^tree" | awk '{print $2}')
# 100644 blob [sha] newfile.txt

# Pop restores it
git stash pop
ls newfile.txt
# newfile.txt   ← restored!
```

### Proof Lab 4: Recover a dropped stash

```bash
# Make a change and stash it
echo "important work" >> app.txt
git stash push -m "IMPORTANT: don't lose this"

STASH_SHA=$(cat .git/refs/stash)
echo "Stash SHA: $STASH_SHA"

# Accidentally drop it
git stash drop

# PROVE IT: The stash ref is gone
cat .git/refs/stash 2>/dev/null || echo ".git/refs/stash does not exist"

# PROVE IT: The commit OBJECT still exists!
git cat-file -t $STASH_SHA
# commit   ← still in .git/objects/! (just unreachable)

# METHOD 1: Recover using the known SHA
git stash apply $STASH_SHA
# Applied successfully!

# METHOD 2 (if you don't know the SHA): Use git fsck
git stash drop  # drop it again to demonstrate fsck recovery

git fsck --unreachable | grep commit
# unreachable commit [SHA1]
# unreachable commit [SHA2]  ← one of these is your stash

# Inspect each to find the WIP commit
git show [SHA1]   # look at the commit message and changes
# Once found:
git stash apply [SHA_of_WIP_commit]
```

### Proof Lab 5: Partial stash — stash only specific files

```bash
# Make changes to 3 files
echo "feature A" >> app.txt
echo "feature B" > feature_b.txt && git add feature_b.txt
echo "debug log" > debug.txt

git status
# Changes to be committed: feature_b.txt
# Changes not staged: app.txt
# Untracked: debug.txt

# Stash ONLY app.txt (not feature_b.txt which is staged, not debug.txt which is untracked)
git stash push app.txt -m "Stash only app.txt"

git status
# Changes to be committed: feature_b.txt   ← still staged
# Untracked: debug.txt                     ← still here

# app.txt is clean now
cat app.txt   # back to its committed state

git stash list
# stash@{0}: On main: Stash only app.txt
```

---

## RULE 6: SYNTAX BREAKDOWN (Deep)

### `git stash push`

```
git stash push [-u | --include-untracked] [-a | --all] [-p | --patch]
               [-S | --staged] [-k | --keep-index] [-q | --quiet]
               [-m <message>] [--] [<pathspec>...]
│              │                │    │               │    │
│              │                │    │               │    └─ specific files to stash
│              │                │    │               │       (only stash these, not everything)
│              │                │    │               └─ separator: treat next as paths
│              │                │    └─ -k/--keep-index: after stashing, keep the index
│              │                │       (don't reset index to HEAD)
│              │                │       Use when you want WD clean but keep staged files
│              │                └─ -p/--patch: interactively select hunks to stash
│              │                   (like git add --patch but for stashing)
│              └─ -a/--all: include untracked AND gitignored files
│                 (even more inclusive than --include-untracked)
└─ -u/--include-untracked: also stash untracked files (not tracked by git)
                            Creates a third parent commit for untracked content

EXAMPLES:
  git stash                           # same as git stash push (legacy syntax)
  git stash push -m "WIP auth"        # stash with a descriptive message
  git stash push -u                   # include untracked files
  git stash push -p                   # interactively choose what to stash
  git stash push -k -m "test"         # stash WD changes, keep staged
  git stash push -- src/auth.js       # stash only src/auth.js
```

### `git stash pop` and `git stash apply`

```
git stash pop [--index] [stash@{N}]
│             │          │
│             │          └─ which stash to apply (default: stash@{0})
│             └─ --index: also restore the staged state (index)
│                WITHOUT --index: all changes come back as unstaged
│                WITH --index: staged changes come back as staged
└─ git stash pop removes the stash after applying (git stash apply does not)

IMPORTANT: --index behavior
  Without --index (default):
    ALL changes (staged and unstaged) come back as UNSTAGED (modified not staged)
    
  With --index:
    Changes that were staged come back as STAGED
    Changes that were unstaged come back as UNSTAGED
    Preserves the original staged/unstaged distinction

EXAMPLES:
  git stash pop                    # apply most recent stash, remove it
  git stash pop --index            # apply and preserve staged/unstaged state
  git stash apply stash@{2}        # apply specific stash, keep it in list
  git stash apply --index stash@{1} # apply specific stash, restore staged state
```

### `git stash list`

```
git stash list [<log-options>]
│              │
│              └─ Any option accepted by git log works here!
│                 Examples: --oneline, --stat, -p (show diff), --since="1 week ago"
└─ reads .git/logs/refs/stash

EXAMPLES:
  git stash list                       # default format
  git stash list --stat                # show files changed in each stash
  git stash list -p                    # show full diff of each stash
  git stash list --since="2 days ago"  # stashes from last 2 days
```

### `git stash show`

```
git stash show [-p | --patch] [--stat] [stash@{N}]
│              │               │        │
│              │               │        └─ which stash (default: stash@{0})
│              │               └─ --stat: show which files changed and how much
│              └─ -p/--patch: show the full diff (like git diff)
└─ shows the DIFFERENCE between the stash and its base (parent 1)
   i.e., what changes the stash contains

EXAMPLES:
  git stash show              # summary of most recent stash (files changed)
  git stash show -p           # full diff of most recent stash
  git stash show stash@{1}    # show specific stash
  git stash show -p stash@{2} # full diff of specific stash
```

### `git stash branch`

```
git stash branch <branchname> [stash@{N}]
│                │              │
│                │              └─ which stash (default: stash@{0})
│                └─ name for the new branch
└─ Creates a new branch from the commit the stash was based on,
   applies the stash to it, and drops the stash if successful

WHAT IT DOES:
  1. Checks out the commit that was HEAD when the stash was created (detached)
  2. Creates <branchname> starting from that commit
  3. Applies the stash changes to the new branch
  4. Drops the stash entry

USE CASE: You stashed work, then someone else pushed to the branch.
  Now the stash conflicts with the current state.
  git stash branch creates a branch from the ORIGINAL base where the stash will apply cleanly.
```

---

## THE `--keep-index` FLAG: A Powerful but Obscure Feature

```
SCENARIO: You have staged changes that are READY to commit, plus unstaged changes
          that are NOT ready. You want to:
          1. Test that the staged changes work correctly in isolation
          2. NOT have the unstaged changes interfere with the test

WITHOUT --keep-index:
  git stash push → saves both staged AND unstaged, resets both → clean WD
  
WITH --keep-index:
  git stash push -k → saves both staged AND unstaged to stash
                    → resets only the UNSTAGED changes from WD
                    → keeps the staged changes in index AND working dir
  
  Result: WD has only the staged changes (unstaged noise is gone)
  You can run your tests → pass → git commit → then git stash pop
```

```bash
# Example:
echo "good feature" >> app.txt && git add app.txt   # ready to commit
echo "debug noise" >> debug.txt                      # NOT ready

git stash push -k -m "Save debug noise"

git status
# Changes to be committed:
#   modified: app.txt    ← still staged! --keep-index preserved it
# (debug.txt is gone — stashed away)

# Run tests with only the staged change in play:
# go test ./...   → pass!

git commit -m "Add good feature"     # commit the staged change
git stash pop                         # restore debug work
```

---

## PARTIAL STASH WITH `--patch`

```bash
# Interactive stash — choose which hunks to stash (like git add --patch)
git stash push -p

# Git shows each hunk and asks:
# Stash this hunk [y,n,q,a,d,/,e,?]?
#   y = yes, stash this hunk
#   n = no, keep this hunk in WD
#   q = quit (stash what's selected so far)
#   a = stash all remaining hunks in this file
#   d = do not stash any remaining hunks in this file
#   s = split this hunk into smaller hunks
#   e = manually edit the hunk

# USE CASE: One file has both "ready" and "not ready" changes
# You stash only the not-ready hunks, commit the ready ones
```

---

## RULE 7: BEGINNER → PRODUCTION EXAMPLE

### Beginner: Context switch without losing work

```bash
# You're in the middle of a feature
echo "half-done login" >> auth.js
git status
# modified: auth.js   (half-done, can't commit yet)

# Urgent bug report comes in — need to fix main immediately
git stash push -m "WIP: login half-done"
git switch main
git switch -c hotfix/null-pointer

# Fix the bug
echo "null check" >> api.js
git commit -am "Fix null pointer in API"

# Back to your feature
git switch feature/login
git stash pop
# auth.js restored — right where you left off
```

### Production: Managing multiple context switches

**Scenario**: You're a senior engineer on a 6-person backend team. In a single afternoon:
1. You're implementing `feature/payments` (complex, half-done)
2. A P1 bug in prod requires immediate attention
3. While fixing, your tech lead asks you to review code on `feature/notifications`
4. While reviewing, you spot something you want to try locally

```bash
# State: you're on feature/payments with 4 modified files

# ── P1 BUG ──
git stash push -m "payments: half-done Stripe integration"
git fetch origin && git checkout -b hotfix/p1-bug origin/main
# fix, commit, push
git push origin hotfix/p1-bug   # open PR

# ── CODE REVIEW ──
git switch feature/notifications   # or git checkout locally
# Make notes, run tests locally

# ── QUICK LOCAL TRY ──
echo "test hypothesis" >> notifications.js
git stash push -m "notifs: testing webhook delay idea"

# ── DONE WITH REVIEW ──
git stash drop    # or pop and discard — you tested your idea, doesn't need keeping
git switch feature/payments

git stash list
# stash@{0}: On feature/payments: payments: half-done Stripe integration
git stash pop
# Back to your exact payments state: 4 files modified, some staged
```

### Production: Using `git stash branch` to escape a conflict trap

**Scenario**: You stashed work 5 days ago. Since then, your colleague rewrote the
files you had modified. `git stash pop` now fails with conflicts.

```bash
git stash list
# stash@{0}: WIP on feature: abc1234 Add payment base — 5 days old

git stash pop
# CONFLICT in payment.js — colleague rewrote this file completely

# Option 1: Resolve manually (may be complex if the files diverged significantly)
# Option 2: Use git stash branch to apply to the ORIGINAL context
git stash drop                   # drop the conflicting pop
git stash branch feature/payments-stash-recovery stash@{0}
# Creates branch at abc1234 (5 days ago), applies stash cleanly there
# Your original changes are now on feature/payments-stash-recovery
# You can now selectively cherry-pick or review what you had
```

---

## RULE 8: MISTAKES → ROOT CAUSE → FIX → PREVENTION

### Mistake 1: `git stash pop` without `--index` loses staged state

**What happened**: You had 3 files staged and 2 unstaged. After stash pop,
ALL 5 show as "modified" (unstaged). Your carefully staged set is gone.

**Root cause**: `git stash pop` (without `--index`) applies all changes as unstaged.
The index state was saved (it's in the index commit), but restoring it requires `--index`.

**Fix**:
```bash
# You already popped and lost the staged state. Re-stage manually:
git diff HEAD --name-only   # see all changed files
git add file1.js file2.js file3.js   # re-stage the ones you had staged
```

**Prevention**:
```bash
git stash pop --index    # always use --index to restore staged/unstaged distinction
```

---

### Mistake 2: Stash was dropped accidentally with `git stash clear`

**What happened**: Ran `git stash clear` thinking it only had old stashes.
Lost 2 important stashes.

**Root cause**: `git stash clear` removes all entries and the reflog. The commit
objects still exist in `.git/objects/` until GC, but there's no easy list anymore.

**Recovery**:
```bash
# Find all unreachable commits (stash commits are unreachable after clear)
git fsck --unreachable | grep commit | awk '{print $3}' > /tmp/candidates.txt

# Inspect each one looking for your stash
while read sha; do
  msg=$(git cat-file -p $sha | grep -A1 "^$" | tail -1)
  echo "$sha: $msg"
done < /tmp/candidates.txt

# Once found:
git stash apply <sha>
```

**Prevention**: Before `git stash clear`, always run `git stash list` first.
Consider `git stash drop stash@{N}` to remove specific entries instead of clearing all.

---

### Mistake 3: Stashing with untracked files but forgetting `-u`

**What happened**: You stashed, switched branches, but new files you created weren't
in the stash. They were still in your working directory — and they CONFLICT with files
on the new branch.

**Root cause**: Default `git stash push` only stashes TRACKED files (files that have
been `git add`-ed at least once). New files are ignored.

**Fix**: The untracked files are still in your working directory. Move them manually
if needed, or stash them separately:
```bash
git stash push -u    # if you still want to stash (includes untracked)
# or:
git add newfile.txt && git stash push  # track first, then stash
```

**Prevention**: When switching context with new files, always use `git stash push -u`.

---

### Mistake 4: `git stash pop` fails with conflicts — you're now in a mess

**What happened**: `git stash pop` hit conflicts. You resolved them badly or
lost track of what was staged vs. unstaged.

**Root cause**: When stash pop hits a conflict, it does NOT drop the stash entry
(unlike a conflict-free pop). But if you abort by just checking out files, the
stash is still in the list AND your index is messy.

**Recovery**:
```bash
# 1. Abort the stash pop by resetting to HEAD
git reset --hard HEAD    # clears all conflict mess

# 2. The stash is still in the list
git stash list   # stash@{0} is still there

# 3. Re-apply carefully
git stash apply stash@{0}   # try again, resolve conflicts properly this time

# 4. Once applied cleanly:
git stash drop stash@{0}   # manually drop it
```

---

### Mistake 5: Wrong branch when you pop

**What happened**: You stashed on `feature/auth`, switched to `feature/payments`,
then ran `git stash pop`. Now auth changes are mixed into payments.

**Fix**:
```bash
# Undo the pop (the stash entry was removed — need to find via reflog)
git reset --hard HEAD   # clear the wrongly-applied changes

# Find the dropped stash SHA
git reflog refs/stash | head -3
# stash@{0}: On feature/auth: WIP auth changes  ← this was just dropped

# Apply to correct branch
git switch feature/auth
git stash apply <SHA_of_dropped_stash>
git stash drop   # manually clean up since apply doesn't remove
```

---

### Mistake 6: Assuming stash survives `git gc`

**What happened**: Your stash is 45 days old. `git gc` ran. The stash's orphaned
commit objects were pruned. `git fsck` finds nothing.

**Root cause**: Git's garbage collection (Topic 25) prunes objects unreachable from
any ref. But stash entries ARE referenced by `.git/refs/stash` and `.git/logs/refs/stash`.
Active stash entries survive GC indefinitely. **Dropped** stash entries become orphaned
and are pruned after the `gc.reflogExpireUnreachable` period (default: 30 days).

**Prevention**: Don't leave stashes for long. Either pop them or save the SHA somewhere.

```bash
# Check stash age
git reflog show refs/stash --date=relative
# stash@{0}: 45 days ago   ← still safe if not dropped
```

---

## RULE 9: BYTE-LEVEL INTERNALS

### What a stash commit looks like in raw bytes

```bash
# Inspect the WIP commit
git cat-file -p stash@{0}
# tree a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
# parent c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0   ← HEAD at stash time
# parent d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1   ← index commit
# author Test <t@t.com> 1748600000 +0530
# committer Test <t@t.com> 1748600000 +0530
#
# WIP on feature: abc1234 Add user login

# Inspect the index commit (parent 2):
git cat-file -p d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1
# tree b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2
# parent c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0   ← same HEAD parent
# author Test <t@t.com> 1748600000 +0530
# committer Test <t@t.com> 1748600000 +0530
#
# index on feature: abc1234 Add user login
```

### The reflog entry format for stash

```bash
# Raw content of .git/logs/refs/stash
cat .git/logs/refs/stash
# 0000000000000000000000000000000000000000 a1b2c3d4 Test <t@t.com> 1748600000 +0530	WIP on main: abc1234 Initial commit
# a1b2c3d4 b2c3d4e5 Test <t@t.com> 1748610000 +0530	WIP on main: def5678 Second commit

# Format per line:
# <old-sha> <new-sha> <author> <timestamp> <timezone> TAB <message>
# Old SHA is all zeros for the first stash (no previous entry)
```

### How `git stash show` computes its diff

```bash
# git stash show -p stash@{0}  is EQUIVALENT to:
git diff stash@{0}^1 stash@{0}
#         │           │
#         │           └─ the WIP commit (stash@{0})
#         └─ parent 1 of the WIP commit (HEAD at stash time)
# 
# This shows what's DIFFERENT between your base and your stash
# (same as git diff HEAD..stash@{0} conceptually)
```

---

## STASH vs ALTERNATIVES

| Need | Best Tool | Why |
|------|----------|-----|
| Quick context switch, restore later | `git stash` | Exactly designed for this |
| Long-term "save this WIP" | Commit a WIP commit on a branch | Stash can get lost; commits are durable |
| Save specific files only | `git stash push -- <file>` | Partial stash |
| Test staged changes in isolation | `git stash push -k` | --keep-index |
| Share WIP with a teammate | WIP commit + push | Stash is local-only |
| Unstage without losing work | `git restore --staged` | No need to stash for this |

### The "WIP commit" pattern as a stash alternative

```bash
# Instead of stash, some teams commit WIP explicitly:
git add .
git commit -m "WIP: half-done payment integration -- DO NOT MERGE"

# Switch context, do your work, come back:
git switch feature/payments
git reset --soft HEAD~1   # uncommit — changes back to staged
# (or git reset --mixed HEAD~1 to unstage too)

# Advantage: survives machine restarts, shows up in git log, can be pushed
# Disadvantage: pollutes history if you forget to squash before PR
```

---

## RULE 10: MENTAL MODEL CHECKPOINT

**1.** `git stash push` creates commit objects in `.git/objects/`. How many commit objects
are created for a normal stash (tracked files only, no untracked)? What are the parents
of the WIP commit?

**2.** What is the difference between `git stash pop` and `git stash apply`? When would
you use apply over pop?

**3.** You stashed with `git stash push` while you had 3 files staged and 2 unstaged.
You run `git stash pop` (without `--index`). How many files show as staged in `git status`?
How many show as modified but unstaged? Why?

**4.** Where does `git stash list` read its data from? What file in `.git/` backs it?
How is `stash@{0}` different from `stash@{1}` in terms of physical storage?

**5.** You dropped a stash by accident. How do you recover it? What is the time limit
on recovery? What git command lists the orphaned commit objects?

**6.** You have staged changes you want to commit, plus messy unstaged changes you
want to test-run without. What exact `git stash` command preserves your staged changes
while hiding the unstaged noise?

**7.** `git stash push -u` vs `git stash push -a` — what's the difference? Give an
example of a file type that `--include-untracked` would NOT stash but `--all` would.

**8.** Describe what `git stash branch feature/rescue stash@{0}` does step by step.
When would this be more useful than `git stash pop`?

**9.** You have 3 stashes. You want to apply `stash@{1}` to a DIFFERENT branch than
where it was created. What's the risk and how do you handle a conflict?

**10.** Why is `git stash` local-only? (Hint: think about where `.git/refs/stash`
and `.git/logs/refs/stash` live.) What's the consequence for remote backups?

---

## RULE 12: QUICK REFERENCE CARD

| Command | What It Does |
|---------|-------------|
| `git stash` | Save WD + index, reset to HEAD (legacy syntax for push) |
| `git stash push -m "msg"` | Save with descriptive message |
| `git stash push -u` | Include untracked files in stash |
| `git stash push -a` | Include untracked + gitignored files |
| `git stash push -k` | Stash but keep staged changes in index |
| `git stash push -p` | Interactively choose hunks to stash |
| `git stash push -- <file>` | Stash only specific file(s) |
| `git stash list` | Show all stashes (newest first) |
| `git stash list --stat` | Show stashes with file change summary |
| `git stash show` | Show diff summary of most recent stash |
| `git stash show -p` | Show full diff of most recent stash |
| `git stash show stash@{N}` | Show specific stash |
| `git stash pop` | Apply + remove most recent stash |
| `git stash pop --index` | Apply + remove, restoring staged/unstaged state |
| `git stash pop stash@{N}` | Apply + remove specific stash |
| `git stash apply` | Apply most recent stash (keep it in list) |
| `git stash apply stash@{N}` | Apply specific stash (keep it) |
| `git stash drop` | Remove most recent stash |
| `git stash drop stash@{N}` | Remove specific stash |
| `git stash clear` | Remove ALL stashes (dangerous!) |
| `git stash branch <name>` | Create branch from stash base, apply stash, drop stash |
| `git fsck --unreachable` | Find dropped/orphaned stash objects |

| Stash structure | Explanation |
|----------------|-------------|
| `stash@{0}` | Most recent stash — `.git/refs/stash` points here |
| `stash@{N}` | Nth most recent — stored in `.git/logs/refs/stash` reflog |
| WIP commit | Has 2 parents: HEAD and index commit (3 with `--include-untracked`) |
| Index commit | Commit whose tree = your staged state at stash time |
| `.git/refs/stash` | Points to most recent WIP commit SHA |
| `.git/logs/refs/stash` | Reflog of all stashes — what `git stash list` reads |

---

## RULE 13: WHEN WOULD I USE THIS AT WORK?

### Scenario 1: Emergency P1 in the middle of a feature

```
2:34 PM: You're 60% through implementing a payment webhook handler
2:35 PM: PagerDuty alert — production API returning 500 for 12% of users

Solution:
  git stash push -u -m "WIP: payment webhook 60% done"
  git checkout main && git pull
  git checkout -b hotfix/api-500

  [15 minutes later — fix found and deployed]

  git switch feature/payments
  git stash pop
  Continue exactly where you left off.
```

### Scenario 2: Testing a colleague's PR locally without committing your work

```
Your work: 4 modified files, partially staged
Need: Pull colleague's PR branch to test it locally

git stash push -u -m "My current WIP"
git fetch origin
git checkout feature/colleague-pr
[test their feature]
git switch -            # switch back to your branch
git stash pop
```

### Scenario 3: Isolating staged changes for testing

```
You staged: auth.js (critical fix — ready to commit)
You haven't staged: debug-helpers.js (noisy test helpers)

Before committing, you want to run the test suite with ONLY the auth.js change:

git stash push -k    # stash everything, keep index (staged auth.js stays)
npm test             # runs with only auth.js change applied
git stash pop        # restore debug-helpers.js
git commit auth.js -m "Fix auth token validation"
```

### Scenario 4: Stash as a "clipboard" for moving changes between branches

```
You started implementing a feature on main (forgot to create a branch):
  modified: feature.js (5 hours of work)

git stash push -m "Feature work done on wrong branch"
git checkout -b feature/new-payments
git stash pop
# Now your work is on the right branch
git push origin feature/new-payments
```

### Scenario 5: Pre-commit hook bypass with stash

```
Your team uses git hooks (Topic 19) that run tests on commit.
You want to commit a WIP that doesn't pass tests yet (as a WIP save).

Correct approach: don't stash, just commit with a WIP marker:
  git commit -m "WIP: in progress — DO NOT MERGE [skip ci]"

But if your hook blocks ALL commits and you need a checkpoint:
  git stash push -m "checkpoint save"
  git stash apply    # re-apply
  # Now commit with --no-verify (bypass hooks — only for WIP saves, never for real commits)
  git commit --no-verify -m "WIP checkpoint"
  git stash drop     # the stash is now redundant
```

---

## SUMMARY: The Mental Model in One Paragraph

Git stash is not a special storage system — it is two (or three) ordinary commit objects
stored in `.git/objects/`, pointed to by a special ref at `.git/refs/stash`, and tracked
via a reflog at `.git/logs/refs/stash`. The "WIP commit" has your working directory's
tree as its tree, and two parents: the current HEAD (your base) and an "index commit"
(your staged state). When you pop or apply, Git reads those trees back and restores them
to the working directory and index. Because stash entries are commit objects, dropped
stashes remain recoverable via `git fsck --unreachable` until garbage collection runs.
The critical flag to remember is `--index` on `git stash pop`: without it, all changes
come back unstaged regardless of how you had them; with it, staged changes come back staged.
And the golden rule: stash is local-only, is never pushed to remotes, and should not be
relied on for long-term storage — use a WIP commit on a branch for anything you care about
lasting beyond a few hours.

---

*Topic 13 of 26 — Next: Topic 14: Tags & Releases (lightweight vs annotated, SemVer, on disk)*
