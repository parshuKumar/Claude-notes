# Topic 20 — git reflog: The Safety Net
## "Recover Anything, Understand Everything — Your Personal Time Machine for Git Itself"

> **Phase 6 — Advanced & Principal-Level** | Builds on Topics 01, 03, 08, 11, 12, 13

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The CCTV of Your Repository](#1-eli5-anchor)
2. [Physical Architecture — `.git/logs/` on Disk](#2-physical-architecture)
3. [Reflog Entry Format — Decoded Byte by Byte](#3-reflog-entry-format)
4. [git reflog Output — Every Column Explained](#4-git-reflog-output)
5. [The Exact Data Flow — How Reflog Entries Get Written](#5-exact-data-flow)
6. [Reflog Notation — @{N} and @{time}](#6-reflog-notation)
7. [Recovery Scenario 1 — Recovering a Deleted Branch](#7-recovery-deleted-branch)
8. [Recovery Scenario 2 — Undoing a Catastrophic reset --hard](#8-recovery-reset-hard)
9. [Recovery Scenario 3 — Finding Lost Commits After Rebase](#9-recovery-after-rebase)
10. [Recovery Scenario 4 — Stash Reflog](#10-stash-reflog)
11. [The Ultimate Fallback — git fsck --lost-found](#11-git-fsck-lost-found)
12. [Reflog Expiry — How Long Git Remembers](#12-reflog-expiry)
13. [Reflog on Remotes — The Critical Gap](#13-reflog-on-remotes)
14. [All Reflogs — Not Just HEAD](#14-all-reflogs)
15. [Byte-Level Internals — The Log File Format](#15-byte-level-internals)
16. [Mistakes → Root Cause → Fix → Prevention](#16-mistakes)
17. [Mental Model Checkpoint — 10 Questions](#17-mental-model-checkpoint)
18. [Connect Everything Backwards](#18-connect-everything-backwards)
19. [Quick Reference Card](#19-quick-reference-card)
20. [When Would I Use This At Work?](#20-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The CCTV Camera of Your Repository

Imagine you're working in a secured office building. Your work (Git commits) is stored in 
filing cabinets. Git's normal history (the DAG) is like the **filing system** — it tracks 
every document ever filed, but only if it was officially filed.

Now imagine there's also a **CCTV camera system** recording everything that happens in the 
building: every door opened, every desk moved, every cabinet accessed. Even if someone 
accidentally shreds a document, the CCTV footage shows you exactly when it happened and 
what it looked like.

That CCTV system is **git reflog**.

**Git's DAG (normal history):**
- Records: commits that exist right now, connected by parent pointers
- What it misses: "I was looking at this commit two hours ago but then reset away from it"

**git reflog:**
- Records: every single time a "pointer" (HEAD, branch, stash) moved to a different commit
- Captures: checkouts, resets, merges, rebases, pulls, stash operations — EVERYTHING
- Stored: locally in `.git/logs/`
- Lifespan: 90 days by default (configurable)

```
Normal Git DAG — what commits EXIST:

  A ← B ← C ← D  (main)
              ↑
             HEAD

Reflog — where HEAD HAS BEEN (in time order, newest first):

  HEAD@{0}: reset: moving to D        ← right now
  HEAD@{1}: commit: add feature       ← was at D before reset
  HEAD@{2}: checkout: moved to main   ← switched from feature branch
  HEAD@{3}: commit: fix bug on feature← was on feature branch
  HEAD@{4}: clone: from origin        ← initial state
```

Even if you delete a branch and lose a commit from the DAG, the reflog shows you the SHA 
of that commit so you can bring it back.

**The single most important thing to understand:** The reflog is your personal undo history 
for Git operations themselves. Git commands modify branches and HEAD. Reflog records every 
single modification. If you can remember it, you can recover from it.

---

## 2. Physical Architecture

### `.git/logs/` on Disk — The Complete Directory Tree

```
.git/
├── HEAD                    ← points to current branch (or detached SHA)
├── logs/                   ← WHERE REFLOG LIVES
│   ├── HEAD                ← every time HEAD pointer moved (THE main reflog)
│   └── refs/
│       ├── heads/
│       │   ├── main        ← every commit to main branch pointer
│       │   ├── feature/    
│       │   │   └── auth    ← every commit to refs/heads/feature/auth
│       │   └── hotfix-v2   ← commits to this branch
│       ├── remotes/
│       │   └── origin/
│       │       └── main    ← every time remote-tracking branch updated
│       └── stash           ← every stash push/pop
└── ...
```

**Each file is a plain text file.** Not binary. Not compressed. Open it right now:

```bash
# See the HEAD reflog file directly (raw)
cat .git/logs/HEAD

# See the main branch reflog
cat .git/logs/refs/heads/main

# Count how many reflog entries you have for HEAD
wc -l .git/logs/HEAD
```

**What you'll see in `.git/logs/HEAD`:**

```
0000000000000000000000000000000000000000 a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5 Alice <alice@co.io> 1748390400 +0530	clone: from https://github.com/team/api
a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5 7c3e8f0d2a1b4c5e6f7d8e9a0b1c2d3e4f5a6b7c Alice <alice@co.io> 1748394000 +0530	commit: add user authentication
7c3e8f0d2a1b4c5e6f7d8e9a0b1c2d3e4f5a6b7c 2b4d6e8f0a2c4e6f8d0b2d4f6a8c0e2d4f6a8c0e Alice <alice@co.io> 1748397600 +0530	checkout: moving from main to feature/auth
2b4d6e8f0a2c4e6f8d0b2d4f6a8c0e2d4f6a8c0e d1c2b3a4e5f6d7c8b9a0e1f2d3c4b5a6e7f8d9c0 Alice <alice@co.io> 1748401200 +0530	commit: implement JWT middleware
d1c2b3a4e5f6d7c8b9a0e1f2d3c4b5a6e7f8d9c0 a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5 Alice <alice@co.io> 1748404800 +0530	reset: moving to HEAD~2
```

Each line = one reflog entry. One line per pointer movement.

**File creation:** Git creates the log file for a ref the first time that ref is created. 
For HEAD it exists from `git init`. For a branch, `.git/logs/refs/heads/branchname` is 
created when you first run `git checkout -b branchname` or `git branch branchname`.

---

## 3. Reflog Entry Format — Decoded Byte by Byte

Every single line in a `.git/logs/` file follows this exact format:

```
<old-sha> <new-sha> <identity> <timestamp> <timezone>\t<action>: <description>
```

Let's decode a real entry:

```
a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5 7c3e8f0d2a1b4c5e6f7d8e9a0b1c2d3e4f5a6b7c Alice <alice@co.io> 1748394000 +0530	commit: add user authentication
│                                        │                                         │                     │           │    │    │
│                                        │                                         │                     │           │    │    └─ description: human-readable reason
│                                        │                                         │                     │           │    └─ action prefix (commit/checkout/reset/rebase/merge/etc)
│                                        │                                         │                     │           └─ TAB character (U+0009) separates coords from message
│                                        │                                         │                     └─ timezone offset (+0530 = India, -0700 = US Pacific)
│                                        │                                         └─ Unix timestamp (seconds since epoch)
│                                        └─ new SHA (where the pointer moved TO)
└─ old SHA (where the pointer was BEFORE this operation)
```

**Special case — first entry (nothing before it):**

```
0000000000000000000000000000000000000000 a3f2c1...  Alice ... 1748390400 +0530	clone: from https://...
│
└─ All zeros = "null SHA" = this ref didn't exist before (just initialized)
```

**The fields in detail:**

| Field | Size | Notes |
|-------|------|-------|
| old-sha | 40 hex chars | The commit SHA the ref was pointing to BEFORE this operation |
| space | 1 byte | Separator |
| new-sha | 40 hex chars | The commit SHA the ref is pointing to AFTER this operation |
| space | 1 byte | Separator |
| identity | variable | "Name <email>" format, from `user.name` + `user.email` config |
| space | 1 byte | Separator |
| timestamp | 10 digits | Unix epoch seconds (seconds since 1970-01-01 00:00:00 UTC) |
| space | 1 byte | Separator |
| timezone | 5 chars | `+HHMM` or `-HHMM` |
| TAB | 1 byte | U+0009 — hard tab separates metadata from message |
| action | variable | `commit`, `checkout`, `reset`, `merge`, `rebase`, `pull`, `push` |
| colon + space | 2 bytes | `: ` separator |
| description | variable | Human-readable description of why the pointer moved |

**Action prefixes you'll see and what triggered them:**

```
commit: add login page          ← git commit
commit (amend): fix typo        ← git commit --amend
commit (initial): initial commit← first-ever commit
checkout: moving from X to Y    ← git checkout / git switch
reset: moving to HEAD~2         ← git reset
reset: moving to <hash>         ← git reset <sha>
merge refs/heads/feature: ...   ← git merge
rebase (start): ...             ← git rebase began
rebase (pick): ...              ← individual cherry-pick in rebase
rebase (finish): ...            ← git rebase completed
pull: Fast-forward              ← git pull (fast-forward merge)
pull: Merge made by 'ort'       ← git pull (3-way merge)
Branch: renamed from X to Y     ← git branch -m
clone: from https://...         ← git clone
fetch: ...                      ← git fetch (remote tracking refs)
stash: WIP on main: <msg>       ← git stash push
```

---

## 4. git reflog Output — Every Column Explained

The `git reflog` command reads `.git/logs/HEAD` and formats it for human consumption:

```bash
git reflog
```

Output:

```
d1c2b3a (HEAD -> main) HEAD@{0}: reset: moving to HEAD~2
7c3e8f0 HEAD@{1}: commit: implement JWT middleware
2b4d6e8 HEAD@{2}: commit: add OAuth provider
a3f2c1d HEAD@{3}: checkout: moving from feature/auth to main
f8e7d6c HEAD@{4}: commit: add rate limiting
e5d4c3b HEAD@{5}: commit: initial user model
0a9b8c7 HEAD@{6}: clone: from https://github.com/team/api
```

**Column breakdown:**

```
d1c2b3a (HEAD -> main) HEAD@{0}: reset: moving to HEAD~2
│       │              │         │      │
│       │              │         │      └─ Description (what operation and details)
│       │              │         └─ Action prefix
│       │              └─ Reflog selector: HEAD@{0} = where HEAD is now
│       │                                  HEAD@{1} = where HEAD was 1 step ago
│       │                                  HEAD@{N} = N steps back in time
│       └─ Decorations: shows where branches/tags currently point (same as git log)
└─ Abbreviated SHA (7 chars by default, enough to be unique in most repos)
```

**Key insight about `HEAD@{0}` vs `HEAD@{1}` etc.:**

- `HEAD@{0}` = where HEAD is RIGHT NOW = same as `HEAD`
- `HEAD@{1}` = where HEAD was 1 operation ago
- `HEAD@{2}` = where HEAD was 2 operations ago
- `HEAD@{N}` = where HEAD was N operations ago
- `HEAD@{yesterday}` = where HEAD was at the end of yesterday
- `HEAD@{1 hour ago}` = where HEAD was 1 hour ago

The list is in reverse-chronological order (newest first). Entry 0 is the current state.

**To see reflog for a specific branch (not HEAD):**

```bash
git reflog show main
git reflog show feature/auth
git reflog show refs/stash
```

**To see reflog with full dates:**

```bash
git reflog --date=iso
# Output:
# d1c2b3a HEAD@{2026-05-28 14:30:00 +0530}: reset: moving to HEAD~2
# 7c3e8f0 HEAD@{2026-05-28 13:00:00 +0530}: commit: implement JWT middleware

git reflog --date=relative
# Output:
# d1c2b3a HEAD@{2 hours ago}: reset: moving to HEAD~2
# 7c3e8f0 HEAD@{3 hours ago}: commit: implement JWT middleware
```

**To see full SHAs instead of abbreviated:**

```bash
git reflog --format="%H %gd %gs"
# %H = full SHA, %gd = reflog selector (HEAD@{N}), %gs = reflog subject (action)
```

---

## 5. Exact Data Flow — How Reflog Entries Get Written

### Every time a pointer moves, Git runs this logic:

```
COMMAND: git reset --hard HEAD~2

STEP 1: READ current HEAD
        Reads: .git/HEAD → "ref: refs/heads/main"
        Reads: .git/refs/heads/main → "d1c2b3a4e5f6..."

STEP 2: RESOLVE target
        HEAD~2 = two commits back from current HEAD
        Walks DAG: d1c2b3a → parent → parent → 7c3e8f0d...

STEP 3: COMPUTE new state
        New commit for main: 7c3e8f0d...

STEP 4: WRITE the reflog entry (BEFORE modifying the ref)
        Opens: .git/logs/HEAD for APPEND
        Writes: "d1c2b3a4... 7c3e8f0d... Alice <alice@co.io> 1748404800 +0530\treset: moving to HEAD~2\n"
        Opens: .git/logs/refs/heads/main for APPEND
        Writes: (same content)

STEP 5: UPDATE the branch pointer
        Writes: .git/refs/heads/main → "7c3e8f0d...\n"

STEP 6: UPDATE the working directory
        Walks tree objects from 7c3e8f0d commit
        Writes changed files to disk
        Updates .git/index to match

STATE BEFORE:
  .git/refs/heads/main = "d1c2b3a4..."
  .git/logs/HEAD = [...older entries...]

STATE AFTER:
  .git/refs/heads/main = "7c3e8f0d..."
  .git/logs/HEAD = [...older entries...]
                   "d1c2b3a4... 7c3e8f0d... Alice ... 1748404800 +0530\treset: moving to HEAD~2\n"
  .git/logs/refs/heads/main = (same new entry appended)
```

### The crucial order: reflog is written BEFORE the ref is updated

This means: even if Git crashes mid-operation, the reflog has a partial record. You can 
always see what Git was trying to do.

### What does NOT get logged:

```
git status     ← read-only, no pointer movement
git log        ← read-only
git diff       ← read-only
git show       ← read-only
git cat-file   ← read-only
git ls-files   ← read-only
```

Only **pointer-moving operations** create reflog entries.

### PROVE IT — Watch reflog entries being created:

```bash
# Initial state
git reflog | head -3

# Make a commit
echo "test" > test.txt
git add test.txt
git commit -m "test commit"

# Reflog now has a new entry at HEAD@{0}
git reflog | head -3

# See the raw log file
tail -1 .git/logs/HEAD
```

---

## 6. Reflog Notation — @{N} and @{time}

### The `@{N}` Selector

`@{N}` is a **reflog qualifier** — it selects the Nth prior position of a ref.

```bash
HEAD@{0}   # Current HEAD (same as HEAD)
HEAD@{1}   # HEAD one operation ago
HEAD@{5}   # HEAD five operations ago
main@{0}   # Current tip of main
main@{1}   # Where main was one commit ago
main@{3}   # Where main was three commits ago
```

**Use these anywhere a commit SHA is valid:**

```bash
# Show the diff between HEAD now and HEAD 3 operations ago
git diff HEAD@{3}

# Show the commit that HEAD pointed to 2 operations ago
git show HEAD@{2}

# Create a branch at the position HEAD was 4 operations ago
git checkout -b recovery-branch HEAD@{4}

# Compare current main with where main was yesterday
git diff main@{yesterday}..main

# Log of commits between 1 week ago state and now
git log HEAD@{1 week ago}..HEAD
```

### The `@{time}` Selector

Instead of counting operations, reference by wall-clock time:

```bash
HEAD@{yesterday}          # where HEAD was at midnight yesterday
HEAD@{1 hour ago}         # where HEAD was 1 hour ago
HEAD@{2026-05-27}         # where HEAD was on this date
HEAD@{2026-05-27 14:00}   # where HEAD was at this exact time
main@{3 days ago}         # where main was 3 days ago
```

**Git resolves these by scanning the reflog backwards** until it finds the last entry 
with a timestamp ≤ the requested time.

**Syntax breakdown:**

```
HEAD@{1 week ago}
│    │
│    └─ Time selector — any of:
│         N seconds/minutes/hours/days/weeks/months ago
│         YYYY-MM-DD
│         YYYY-MM-DD HH:MM:SS
│         yesterday, noon, midnight, tea, etc.
└─ Ref name (HEAD, main, feature/auth, stash, etc.)
```

**PROVE IT — time-based reflog:**

```bash
# See where main was yesterday
git show main@{yesterday}

# If you have enough history:
git log --oneline main@{1 week ago}..main
# Shows every commit merged to main in the last week

# See what the working tree looked like at a time point
git checkout HEAD@{3 days ago}
# WARNING: this puts you in detached HEAD state
# Then:
git checkout main   # return to present
```

---

## 7. Recovery Scenario 1 — Recovering a Deleted Branch

### The Scenario

You're a backend engineer. You've been working on `feature/payment-gateway` for two days. 
Your colleague asks you to look at their branch. You type:

```bash
git checkout main
git branch -D feature/payment-gateway   # -D = force delete (no merge check)
```

Then you realize: that branch had uncommitted work you needed. The branch pointer is gone.

### Why This Works (The Architecture Reason)

`git branch -D` does two things:
1. Deletes `.git/refs/heads/feature/payment-gateway`
2. Deletes `.git/logs/refs/heads/feature/payment-gateway`

But it does NOT:
- Delete the commit objects from `.git/objects/`
- Delete the entry from `.git/logs/HEAD`

The commits still exist in `.git/objects/`. They're just **unreachable** — no branch or 
tag points to them anymore. And the reflog for HEAD still has the SHA from when HEAD 
was last pointing there.

### The Recovery Process

```
SITUATION:
  .git/refs/heads/feature/payment-gateway ← DELETED (file removed)
  .git/objects/d1c2b3a4.../              ← STILL EXISTS
  .git/logs/HEAD                         ← STILL HAS THE SHA

STEP 1: Find the SHA from reflog
```

```bash
# Option A: Look at HEAD reflog for when you were on that branch
git reflog
# Find the last entry that says "checkout: moving from feature/payment-gateway to main"
# The SHA BEFORE that checkout is the tip of the deleted branch

# Option B: Search reflog for the branch name directly  
git reflog | grep "payment-gateway"
# Output:
# a1b2c3d HEAD@{4}: commit: add Stripe webhook handler (was feature/payment-gateway)
# 7e8f9a0 HEAD@{5}: checkout: moving from main to feature/payment-gateway
# ...
```

```
STEP 2: Verify the SHA is still reachable in objects
```

```bash
# The commit at HEAD@{4} is a1b2c3d — let's verify it still exists
git cat-file -t a1b2c3d
# Expected: commit    ← the object still exists!

git show a1b2c3d
# Shows the full commit (code changes, message, author) — IT'S STILL THERE
```

```
STEP 3: Recreate the branch at that SHA
```

```bash
# Recreate the branch pointing to the lost commit
git checkout -b feature/payment-gateway a1b2c3d
# OR
git branch feature/payment-gateway a1b2c3d
git checkout feature/payment-gateway

# Verify it's back
git log --oneline -5
# Shows your lost commits!
```

### Diagram — Before, During, After Recovery

```
BEFORE DELETION:
  .git/refs/heads/feature/payment-gateway → a1b2c3d (tip of branch)
  .git/logs/HEAD: ... a1b2c3d HEAD@{4}: commit (was feature/payment-gateway) ...
  .git/objects/a1/b2c3d... (commit object exists)
  
AFTER DELETION (git branch -D):
  .git/refs/heads/feature/payment-gateway → GONE (file deleted)
  .git/logs/HEAD: ... a1b2c3d HEAD@{4}: commit (was feature/payment-gateway) ...  ← STILL HERE
  .git/objects/a1/b2c3d... ← STILL EXISTS (GC hasn't run yet)
  
AFTER RECOVERY (git checkout -b feature/payment-gateway a1b2c3d):
  .git/refs/heads/feature/payment-gateway → a1b2c3d (RECREATED)
  .git/logs/refs/heads/feature/payment-gateway (RECREATED, new log)
  .git/objects/a1/b2c3d... ← STILL EXISTS (now reachable again)
```

### Time Window — How Long Do You Have?

By default, Git's garbage collector (`git gc`) runs automatically approximately every 
100 `git commit` commands (configurable via `gc.auto`). When GC runs, it removes objects 
that are both:
1. Unreachable (no branch/tag/reflog points to them)
2. Older than 2 weeks (for unreferenced objects; 30 days for reflog-referenced objects)

So you have **at least 2 weeks** before the objects disappear from disk. But reflog 
entries themselves expire after **90 days** by default.

**Rule of thumb:** If you deleted the branch today, you can recover it. If you deleted 
it 3 months ago and GC ran since then, you likely cannot.

---

## 8. Recovery Scenario 2 — Undoing a Catastrophic `reset --hard`

### The Scenario

It's 4pm on a Friday. You're about to push your week's work. You want to reset one file 
back to main's state. You accidentally run:

```bash
git reset --hard origin/main
```

Instead of what you meant (`git restore server.js`). Now your local branch is at the same 
state as `origin/main`. Three days of work appear to be gone.

### Understanding What Happened Architecturally

```
BEFORE:
  .git/refs/heads/main = d1c2b3a  (your local, 3 days ahead of origin)
  Working directory = your 3 days of changes

COMMAND: git reset --hard origin/main

STEP 1: Resolved origin/main to SHA: 7c3e8f0 (3 days behind)
STEP 2: WROTE reflog: "d1c2b3a 7c3e8f0 Alice ... reset: moving to origin/main" to .git/logs/HEAD and .git/logs/refs/heads/main
STEP 3: WROTE: .git/refs/heads/main → "7c3e8f0"
STEP 4: RESET working directory to match 7c3e8f0's tree
STEP 5: RESET .git/index to match 7c3e8f0's tree

AFTER:
  .git/refs/heads/main = 7c3e8f0  (now matches origin)
  Working directory = origin/main state (your files appear gone)
  BUT: .git/logs/HEAD has "d1c2b3a" as the old-sha in the last entry
  AND: .git/objects/d1/c2b3a... still exists (you made commits, objects persist)
```

Your commits still exist as objects in `.git/objects/`. The branch pointer just moved.

### The Recovery Process

```bash
# STEP 1: See reflog — find your commits
git reflog
# Output (newest first):
# 7c3e8f0 (HEAD -> main, origin/main) HEAD@{0}: reset: moving to origin/main
# d1c2b3a HEAD@{1}: commit: finalize payment processing logic
# 8f7e6d5 HEAD@{2}: commit: add idempotency keys to payment API
# 3a2b1c0 HEAD@{3}: commit: implement refund flow
# ...

# STEP 2: Confirm HEAD@{1} is your latest commit (before the reset)
git show HEAD@{1}
# Shows: "finalize payment processing logic" — YES, that's it!

# STEP 3: Reset back to where you were
git reset --hard HEAD@{1}
# OR equivalently:
git reset --hard d1c2b3a
```

**That's it.** One command. Three days of work back in 30 seconds.

### Diagram — The Undo

```
REFLOG ENTRIES (newest first):
  HEAD@{0}: 7c3e8f0  reset: moving to origin/main   ← current position
  HEAD@{1}: d1c2b3a  commit: finalize payment processing logic
  HEAD@{2}: 8f7e6d5  commit: add idempotency keys
  HEAD@{3}: 3a2b1c0  commit: implement refund flow

git reset --hard HEAD@{1}
         ↓
  HEAD@{0}: d1c2b3a  reset: moving to HEAD@{1}    ← new current (your work back!)
  HEAD@{1}: 7c3e8f0  reset: moving to origin/main  ← the accident
  HEAD@{2}: d1c2b3a  commit: finalize payment processing
  ...
```

**Note:** The recovery itself adds a new reflog entry. The accident is still recorded in 
the reflog. The reflog is append-only — you cannot erase the mistake from history. 
But the RESULT is your branch is back where you want it.

### What About Uncommitted Changes?

`git reset --hard` also resets the **working directory** and **index** to match the commit. 
Uncommitted changes that were not staged are permanently gone — no reflog can help with 
those because they were never committed.

This is why the rule is: **commit early and often** in development. Staged changes can 
sometimes be recovered via `git fsck` (covered in Section 11), but working directory 
changes that were never staged: truly gone.

---

## 9. Recovery Scenario 3 — Finding Lost Commits After Rebase

### The Scenario (Connecting to Topic 11)

Recall from Topic 11: `git rebase` creates NEW commit objects with different SHAs and 
abandons the old ones. If something goes wrong during a rebase (or after a rebase you 
wanted to undo), you need to find the pre-rebase commits.

```
BEFORE REBASE:
  feature: A - B - C - D - E   (D and E are your commits)
                    ↑
                   main

AFTER REBASE:
  feature:     A - B - C - D' - E'  (D' and E' are NEW objects with different SHAs)

  D and E still exist in .git/objects/ but are now UNREACHABLE from any ref
```

### How Rebase Writes to Reflog

During a rebase, Git writes multiple reflog entries:

```
HEAD@{0}: rebase (finish): returning to refs/heads/feature
HEAD@{1}: rebase (pick): implement feature part 2     ← E'
HEAD@{2}: rebase (pick): implement feature part 1     ← D'
HEAD@{3}: rebase (start): checkout main               ← went to main tip
HEAD@{4}: commit: implement feature part 2            ← E (the original, now gone)
HEAD@{5}: commit: implement feature part 1            ← D (the original, now gone)
```

### Recovery — Undo an Entire Rebase

```bash
# Find the "rebase (start)" entry — that's where you were before the rebase began
git reflog | grep "rebase (start)"
# HEAD@{3}: rebase (start): checkout main

# The entry BEFORE the rebase start (HEAD@{4}) is where feature was
git show HEAD@{4}
# Shows your original D commit — before rebase

# Reset the branch to the pre-rebase state
git reset --hard HEAD@{4}
# OR find the exact SHA for E (the tip before rebase) — it's HEAD@{4}'s parent
```

**Actually, there's an easier way for recent rebases:**

During a rebase, Git saves `ORIG_HEAD` (remember from Topic 12?):

```bash
# ORIG_HEAD points to where the rebased branch was before rebase started
cat .git/ORIG_HEAD
# Shows: SHA of E (the pre-rebase tip)

# So immediately after a bad rebase:
git reset --hard ORIG_HEAD
```

But `ORIG_HEAD` gets overwritten by the next operation. If you did more commits after 
the rebase, ORIG_HEAD is gone. In that case, use the reflog method above.

### Recovery — Find a Specific Lost Commit After Rebase

```bash
# Maybe you rebased, and now the order is wrong and you can't find commit "D"
# Search reflog for the commit message
git log -g --oneline --grep="implement feature part 1"
# -g = walk reflog instead of ancestry, --grep searches commit messages

# Output:
# d1c2b3a HEAD@{5}: implement feature part 1  ← the original D before rebase!

# Cherry-pick it onto your current branch
git cherry-pick d1c2b3a
```

---

## 10. Stash Reflog

### Stash Has Its Own Reflog

From Topic 13, you know stash uses `refs/stash` to point to a special stash commit. 
Every stash operation is recorded in `.git/logs/refs/stash`:

```bash
# See the stash reflog
git reflog stash
# OR
git reflog refs/stash

# Output:
# a1b2c3d stash@{0}: WIP on main: 7c3e8f0 fix auth bug
# e5f6d7c stash@{1}: WIP on feature/auth: 2b4d6e8 add OAuth
# 8f9a0b1 stash@{2}: WIP on main: 3c4d5e6 initial setup
```

### Recovering a Dropped Stash

The most common stash disaster: `git stash drop` or `git stash pop` when you didn't 
mean to, or `git stash clear` (deletes ALL stashes).

```
PROBLEM: You ran `git stash drop stash@{0}` — the stash ref is gone
         BUT the stash commit object is still in .git/objects/
         AND .git/logs/refs/stash still has the SHA

STEP 1: Find the SHA in the stash reflog
git reflog stash
# Output (even after drop, the log file may still have the entry):
# stash@{0}: WIP on main: 7c3e8f0 fix auth bug  ← SHA is a1b2c3d

# If the reflog file was also cleaned, try fsck:
git fsck --unreachable | grep commit
# Lists unreachable commit objects — one of them is your stash
```

```bash
# STEP 2: Show the stash commit to verify it's the right one
git show a1b2c3d

# STEP 3: Apply it like a regular stash
git stash apply a1b2c3d
```

### After `git stash clear` — The Nuclear Recovery

`git stash clear` removes `refs/stash` and cleans `.git/logs/refs/stash`. Now the stash 
commits are fully unreachable. You have to go through `git fsck`:

```bash
# Find all dangling (unreachable) commits
git fsck --unreachable 2>/dev/null | grep "unreachable commit"
# Output:
# unreachable commit a1b2c3d4e5f6...
# unreachable commit e7f8a9b0c1d2...

# Look at each one to find your stash
git show a1b2c3d4e5f6
# If this looks like your stash work, apply it:
git stash apply a1b2c3d4e5f6
```

---

## 11. The Ultimate Fallback — `git fsck --lost-found`

### When Reflog Isn't Enough

The reflog only records pointer movements for **refs** (HEAD, branches, stash). It does 
NOT record:
- Objects created by `git add` but never committed (blob objects without a commit)
- Commits that were never referenced by any branch (created by `git commit-tree` directly)
- Any operation that didn't move a ref pointer

For those cases, `git fsck` is your final weapon.

### What `git fsck` Does

`git fsck` walks the entire object database in `.git/objects/` and finds any object that 
is not reachable from any ref (HEAD, branches, tags, reflog-referenced SHAs).

```bash
# Find all unreachable objects
git fsck --unreachable

# Output example:
# unreachable blob d3a4b5c6d7e8f9a0b1c2d3e4f5f6f7f8f9a0b1c2
# unreachable commit a1b2c3d4e5f6a7b8c9d0e1f2f3f4f5f6f7f8f9a0
# unreachable tree e5f6a7b8c9d0e1f2f3f4f5f6f7f8f9a0b1c2d3e4
# dangling commit 7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a
# dangling blob  2a3b4c5d6e7f8a9b0c1d2e3f4f5f6f7f8f9a0b1c
```

**The difference between `unreachable` and `dangling`:**
- `dangling` = top-level unreachable (not reachable even from another unreachable object)
- `unreachable` = unreachable but referenced by a dangling parent

For recovery, **focus on dangling commits** — those are the tips of lost commit chains.

### The `--lost-found` Flag

```bash
# The nuclear recovery option
git fsck --lost-found

# This does two things:
# 1. Prints all unreachable objects
# 2. Writes them to .git/lost-found/

ls .git/lost-found/
# commit/    ← unreachable commits
# other/     ← unreachable blobs and trees

ls .git/lost-found/commit/
# a1b2c3d4e5f6...  ← each file named after the commit SHA
# e7f8a9b0c1d2...
# ...

# Each file contains the commit info
cat .git/lost-found/commit/a1b2c3d4e5f6...
# Shows commit metadata

# Or use git show on the SHA:
git show a1b2c3d4e5f6
```

### Recovering Lost Blobs (File Content)

```bash
# Maybe you staged a file but never committed, then something wiped the index
# The blob object still exists in .git/objects/

# Find dangling blobs
git fsck --lost-found | grep "dangling blob"
# dangling blob 2a3b4c5d6e7f8a9b0c1d2e3f4f5f6f7f8f9a0b1c

# View the blob content
git cat-file -p 2a3b4c5d6e7f8a9b0c1d2e3f4f5f6f7f8f9a0b1c
# Output: the raw file content you thought was lost!

# Save it to a file
git cat-file -p 2a3b4c5d6e7f8a9b0c1d2e3f4f5f6f7f8f9a0b1c > recovered-file.js
```

### The Decision Tree for Recovery

```
                    Lost some work?
                         │
           ┌─────────────┴─────────────┐
     Was it committed?           Never committed?
           │                           │
           ▼                           ▼
    git reflog              Was it staged (git add)?
    to find SHA                        │
           │               ┌───────────┴──────────┐
           ▼            Staged                Never staged
    git checkout              │                    │
    git reset --hard          ▼                    ▼
    git cherry-pick     git fsck               PERMANENTLY
    to recover        --lost-found              LOST
                     (find the blob)
```

---

## 12. Reflog Expiry — How Long Git Remembers

### Default Expiry Policies

Git's garbage collector removes reflog entries (and then the now-unreachable objects) 
on a schedule. The defaults are:

| Config Key | Default | Meaning |
|------------|---------|---------|
| `gc.reflogExpire` | 90 days | Normal reflog entries expire after 90 days |
| `gc.reflogExpireUnreachable` | 30 days | Entries pointing to unreachable commits expire after 30 days |

**The distinction:** A reflog entry pointing to a commit that is also reachable from a 
current branch gets the full 90 days. An entry pointing to a commit that is NOT reachable 
from any current branch (like a deleted branch's tip) gets only 30 days.

### How Expiry Works

```bash
# Manually expire reflog entries (runs automatically with git gc)
git reflog expire --expire=90.days refs/heads/main
git reflog expire --expire=30.days --expire-unreachable=30.days --all

# See what would be expired without actually expiring
git reflog expire --expire=now --dry-run
```

When `git gc` runs automatically:
1. It calls `git reflog expire --expire=90.days --expire-unreachable=30.days --all`
2. This removes expired entries from all `.git/logs/` files
3. Then `git prune` removes objects that are no longer referenced by any reflog entry

```bash
# Check when gc will run automatically
git config gc.auto
# Default: 6700 (runs when loose objects exceed 6700)

# How git gc is automatically triggered
# Git calls pack-refs and prune after every 100th commit-creating command (gc.autoPackLimit)

# Manually run gc (with reflog expiry)
git gc

# More aggressive cleanup
git gc --aggressive --prune=now
# WARNING: --prune=now removes ALL unreachable objects immediately, no grace period
```

### Configuring Longer Reflog Retention

If your team works on long-running features and wants more recovery window:

```bash
# Extend the reflog to 180 days (6 months)
git config --global gc.reflogExpire 180.days
git config --global gc.reflogExpireUnreachable 90.days

# Per-repo override (e.g., for a critical production repo):
git config gc.reflogExpire "never"
# "never" means entries never expire (be careful — repo grows indefinitely)
```

### Checking What's In Your Reflog Right Now

```bash
# Count reflog entries for HEAD
git reflog | wc -l

# See oldest entry
git reflog | tail -1

# See how old your reflog history goes back (relative dates)
git reflog --date=relative | tail -5
```

---

## 13. Reflog on Remotes — The Critical Gap

### The Most Important Thing Most Developers Don't Know

**Remote repositories (GitHub, GitLab, Bitbucket) do NOT have reflog enabled by default.**

This means:
- If you `git push --force` and overwrite commits on the remote, those commits are gone 
  from the remote's perspective
- There is no equivalent of `git reflog` you can run against GitHub to find the old commits
- Branch protection rules and the `--force-with-lease` safeguard exist specifically because 
  of this gap

### Why Remotes Don't Have Reflog

A server-side reflog would mean:
1. Storing reflog data for every user who pushes (disk space × users × pushes)
2. Security implications (you could see when other users pushed)
3. The reflog is a LOCAL development tool, not a server-side auditing tool

```
LOCAL REPO (.git/logs/)              REMOTE REPO (GitHub)
  logs/HEAD ← ALL your moves          No logs/ directory
  logs/refs/heads/main ← all pushes   No reflog
  logs/refs/stash                     No stash

You push --force to origin/main:
  LOCAL: reflog records the push      REMOTE: branch pointer overwritten
                                              old commits unreachable
                                              NO RECORD of the old state
```

### What GitHub Does Provide Instead

GitHub has its own audit mechanisms (not called reflog):
- **Force push protection** via branch protection rules (blocks the push entirely)
- **Push event logs** in organization audit logs (shows who pushed when, not what was lost)
- **Pull Request history** (if the commits were in a PR, they're archived there)

### The Safety Net for Pushed Commits

If you force-pushed and need to recover:

```bash
# YOUR LOCAL REFLOG has the record of what you pushed
git reflog | grep push
# HEAD@{N}: update by push

# The SHA before the push is what was on the remote
# If your teammate also has a clone, THEIR local clone might have the old commits
# Contact them immediately after a bad force push
```

**Best practice:** If you're about to force-push and want a safety copy:

```bash
# Create a backup branch before force-pushing
git branch backup/before-force-push-$(date +%Y%m%d)
git push --force

# If something goes wrong:
git push origin backup/before-force-push-20260528:main --force
```

---

## 14. All Reflogs — Not Just HEAD

### The Reflog System Covers All Refs

```bash
# See ALL refs that have reflog data
ls -la .git/logs/refs/heads/    # all local branches
ls -la .git/logs/refs/remotes/  # remote tracking branches
ls -la .git/logs/              # HEAD + refs directory

# Show reflog for a specific branch
git reflog show main
git reflog show feature/auth
git reflog show refs/remotes/origin/main  # remote tracking branch

# Show ALL reflogs in one command
git log -g --oneline --all
# -g = --walk-reflogs, walks reflog of all refs
# --all = all branches and tags
# --oneline = compact format
```

### Remote Tracking Branch Reflog

Even though the **remote** doesn't have a reflog, **your local copy of the remote 
tracking branch** (`origin/main`) does:

```bash
# See what origin/main looked like before the last fetch/pull
git reflog refs/remotes/origin/main
# Output:
# 7c3e8f0 refs/remotes/origin/main@{0}: fetch: fast-forward
# a3f2c1d refs/remotes/origin/main@{1}: fetch: fast-forward
# 0b9a8c7 refs/remotes/origin/main@{2}: clone: from https://...

# This means: before the last fetch, origin/main was at a3f2c1d
# If a bad push went to remote, and you haven't re-fetched, you still have the old SHA:
git checkout -b recovery/origin-main-before-bad-push refs/remotes/origin/main@{1}
```

This is a powerful but little-known recovery path when someone force-pushed to origin.

### Using `-g` to Walk Reflogs in git log

```bash
# git log normally walks the DAG. -g makes it walk the reflog instead
git log -g
# Shows all commits that ever appeared in any reflog, in reflog order

# More useful: see all commits in reflog with one-line format
git log -g --oneline

# Search commit messages across ALL reflog entries
git log -g --oneline --grep="payment"
# Finds "payment" in any commit that's ever appeared in your reflog
# Even if that commit is no longer reachable from any branch!
```

---

## 15. Byte-Level Internals — The Log File Format

### Exact Format of `.git/logs/HEAD`

Each line in a reflog file is exactly this format (no exceptions):

```
<40-char SHA>\s<40-char SHA>\s<GIT-IDENT>\s<EPOCH>\s<TZ>\t<MESSAGE>\n
```

Where:
- `\s` = ASCII space (0x20)
- `\t` = ASCII horizontal tab (0x09)
- `\n` = ASCII newline (0x0A)
- GIT-IDENT = `Name <email>` (the committer identity at time of operation)
- EPOCH = Unix timestamp as decimal integer (not padded, 10 digits currently)
- TZ = timezone in `+HHMM` or `-HHMM` format (exactly 5 characters)

```
Full binary layout of one entry:

Bytes 0-39:  old SHA (40 ASCII hex characters: 0-9, a-f)
Byte 40:     0x20 (space)
Bytes 41-80: new SHA (40 ASCII hex characters)
Byte 81:     0x20 (space)
Bytes 82-N:  identity "Name <email>" (variable length)
Byte N+1:    0x20 (space)
Bytes N+2 to N+11: timestamp (10 ASCII decimal digits)
Byte N+12:   0x20 (space)
Bytes N+13 to N+17: timezone (5 characters: +0530, -0700, +0000, etc.)
Byte N+18:   0x09 (TAB — THE ONLY TAB IN THE FORMAT)
Bytes N+19+: message string (variable, NO newlines embedded)
Last byte:   0x0A (newline, terminates the entry)
```

### How Git Parses This Back

When you run `git reflog`, Git:
1. Opens `.git/logs/HEAD`
2. Reads line by line
3. For each line, splits on the TAB (0x09) to separate coords from message
4. Splits the first part on spaces to extract: old-sha, new-sha, identity, timestamp, tz
5. Formats the output based on `--format` flags

### Why TAB as the Separator?

Because commit messages (the description after `action: `) CAN contain spaces but 
should NOT contain tab characters. So TAB is used as an unambiguous delimiter between 
the metadata fields and the free-form message text.

```bash
# PROVE IT: Read a raw reflog entry in hex
# (Linux/Mac: xxd, Windows: use Git Bash with xxd or format-hex in PowerShell)
tail -1 .git/logs/HEAD | xxd | head -5
# You'll see the 0x09 (TAB) byte clearly in the output, separating metadata from message
```

### How New Entries Are Appended

When Git writes a new reflog entry, it:
1. Opens the log file with `O_WRONLY | O_APPEND | O_CREAT` flags
2. Constructs the line in memory
3. Writes it in a single `write()` syscall (atomic from the filesystem perspective)
4. Closes the file

This append-only behavior means:
- Reflog files grow monotonically (entries are never reordered)
- The most recent entry is always at the END of the file (but `git reflog` shows newest first)
- `git reflog expire` removes entries from the BEGINNING of the file (oldest first)

---

## 16. Mistakes → Root Cause → Fix → Prevention

---

### Mistake 1: "The reflog doesn't show my missing commits"

**Scenario:** You lost some commits but `git reflog` doesn't list them anywhere. You 
can't find the SHA.

**Root Cause (two possibilities):**

A) The commits were in a different repo or a different branch's reflog:
```bash
# You're checking HEAD reflog but the work was on a different branch
git reflog show feature/auth   # Check the branch-specific reflog
git log -g --all --oneline     # Check ALL reflogs at once
```

B) The reflog expired because GC ran and it's been >30 days:
```bash
# GC may have pruned it — check if objects still exist via fsck
git fsck --unreachable | head -20
```

**Fix:** Use `git fsck` as the fallback (see Section 11). If fsck also shows nothing, 
the objects have been garbage collected and the commits are permanently gone.

**Prevention:**
```bash
# Extend reflog retention time
git config --global gc.reflogExpireUnreachable 90.days
# Don't run git gc --prune=now unless you INTEND to permanently remove unreachable objects
```

---

### Mistake 2: Running `git gc --prune=now` Immediately After an Accident

**Scenario:** You reset --hard accidentally. You panic and run `git gc` to "clean up". 
Now the objects are gone.

**Root Cause:** `--prune=now` sets the grace period to zero. Objects that would normally 
survive for 2 weeks (the default grace period) are immediately deleted.

**The broken state:**
```bash
git reset --hard HEAD~5   # accidental reset
git gc --prune=now        # DISASTER: permanently removed HEAD~1 through HEAD~5
git reflog                # shows the SHAs but...
git cat-file -t <sha>     # fatal: Not a valid object name  ← gone forever
```

**Fix:** There is no fix. The objects are gone.

**Prevention:** Never run `git gc --prune=now` without intent. After an accident, 
immediately reach for `git reflog` BEFORE running any cleanup commands.

---

### Mistake 3: Expecting Reflog to Work on a Fresh Clone

**Scenario:** You clone a repo and run `git reflog`. It shows almost nothing — just the 
clone entry. You're confused because there's years of history.

**Root Cause:** Git history (the DAG) is shared when you clone. The reflog is NOT. The 
reflog records YOUR local operations on YOUR local copy. It doesn't transfer in a clone.

```bash
# Fresh clone:
git clone https://github.com/team/api.git
cd api
git reflog
# Output:
# a3f2c1d (HEAD -> main, origin/main) HEAD@{0}: clone: from https://github.com/team/api.git
# That's it. No history of what others did.
```

**Fix:** Not a problem — expected behavior. The reflog will build up as you work.

**Prevention:** Understand that reflog is personal (local only) history. Use `git log` 
to see shared project history. Use `git reflog` to see YOUR local HEAD movements.

---

### Mistake 4: Confusing `HEAD@{1}` (reflog) with `HEAD~1` (ancestry)

**Scenario:** You want to go back to the previous commit in history. You use `HEAD@{1}` 
but get the wrong commit — it jumps to the last checkout position, not the parent commit.

**Root Cause:** These two notations are COMPLETELY DIFFERENT:

```
HEAD~1     = parent of the current commit (navigate the DAG backwards by ancestry)
HEAD@{1}   = where HEAD was pointed 1 operation ago (navigate the reflog backwards by time)

Example: If you just checked out main (which is at commit D),
HEAD~1 = commit C (parent of D in the commit graph)
HEAD@{1} = the commit you were at BEFORE checking out main (could be anything — commit X on a totally different branch)
```

**Fix:** Use `HEAD~N` for "N commits back in history". Use `HEAD@{N}` for "N operations 
ago in the reflog".

**Prevention:** Burn this table into your memory:

| Notation | Type | Meaning |
|----------|------|---------|
| `HEAD~1` | Ancestry | Parent commit in the DAG |
| `HEAD~3` | Ancestry | Great-grandparent commit in the DAG |
| `HEAD^2` | Ancestry | Second parent of a merge commit |
| `HEAD@{1}` | Reflog | Where HEAD was 1 reflog operation ago |
| `HEAD@{3}` | Reflog | Where HEAD was 3 reflog operations ago |
| `HEAD@{1 hour ago}` | Reflog | Where HEAD was 1 hour ago (by timestamp) |

---

### Mistake 5: Not Using Reflog Before Force-Pushing

**Scenario:** Your colleague ran `git push --force` to main (bypassing branch protection 
somehow). Three other teammates had pulled those commits. Now origin/main is 50 commits 
behind, everyone's histories are diverged.

**Root Cause:** No one saved the pre-force-push SHA. The remote has no reflog. Everyone's 
local repos need to be checked.

**Fix:**

```bash
# Each teammate runs this to see if they have the old commits
git reflog | grep "HEAD@{0}"   # current HEAD
git log --oneline -10          # see their current history

# Whoever still has the commits on their local branch:
git push origin their-branch-name:main --force
# (after team agrees)
# OR create a backup branch and notify the team
git push origin <sha-of-old-tip>:refs/heads/main --force
```

**Prevention:**
```bash
# Before any force push, ALWAYS create a backup branch
git push origin main:backup/main-before-force-$(date +%Y%m%d-%H%M)
# Then force push
git push --force-with-lease origin main
```

---

## 17. Mental Model Checkpoint — 10 Questions

Try to answer these from memory before checking the answers below.

**Q1.** Where exactly is the reflog stored on disk? Name at least 3 specific file paths.

**Q2.** What is the exact format of a single reflog entry (the raw file format)?

**Q3.** What is the difference between `HEAD@{1}` and `HEAD~1`? Give a concrete example 
where they would return different commits.

**Q4.** You deleted a branch with `git branch -D feature/auth`. What is the step-by-step 
process to recover it? What's the time limit before the commits are gone forever?

**Q5.** You ran `git reset --hard origin/main` accidentally. Your three days of commits 
appear gone. What single command sequence do you run to recover them?

**Q6.** After `git rebase`, where in the reflog do you look to find the pre-rebase branch 
tip? What is the quick one-command recovery?

**Q7.** What is `git fsck --lost-found`? When would you use it instead of `git reflog`? 
What objects does it place in `.git/lost-found/`?

**Q8.** Does GitHub (or any remote) have a reflog? What are the implications for 
force-pushes? What is the workaround?

**Q9.** What are the default expiry times for reflog entries? What is the difference 
between `gc.reflogExpire` and `gc.reflogExpireUnreachable`?

**Q10.** What does `git log -g --oneline --grep="login"` do that `git log --oneline --grep="login"` does NOT do?

---

**Answers:**

**A1.** `.git/logs/HEAD` (HEAD movements), `.git/logs/refs/heads/main` (main branch), 
`.git/logs/refs/heads/feature/X` (feature branches), `.git/logs/refs/stash` (stash).

**A2.** `<old-sha> <new-sha> <Name> <<email>> <epoch-timestamp> <+HHMM>\t<action>: <description>\n`
Key: old-sha and new-sha are 40 hex chars, identity and timestamp separated by spaces, 
TAB (not space) separates the metadata from the message.

**A3.** `HEAD~1` = parent commit in the DAG (navigate commit history). `HEAD@{1}` = where 
HEAD pointed one operation ago (navigate reflog). Example: If HEAD is on main at commit D, 
`HEAD~1` = commit C (D's parent). If you just checked out main from feature/auth, 
`HEAD@{1}` = the commit you were at on feature/auth (could be commit X, completely unrelated to main).

**A4.** 
1. `git reflog | grep feature/auth` — find last SHA the branch was at
2. `git cat-file -t <sha>` — verify object still exists
3. `git checkout -b feature/auth <sha>` — recreate branch
Time limit: ~30 days for unreachable commits before GC purges them (gc.reflogExpireUnreachable default). 
The actual commit objects remain in `.git/objects/` until GC runs AND the reflog entry 
has expired (30 days for unreachable). So practically: recover within 30 days.

**A5.** 
1. `git reflog` — find the SHA at HEAD@{1} (just before the reset)
2. `git reset --hard HEAD@{1}` (or the specific SHA shown)
Commits still exist in `.git/objects/`. The reflog entry shows the SHA they were at.

**A6.** Look for `rebase (start)` entry in `git reflog`. Entry immediately before it 
(`HEAD@{N+1}`) is the pre-rebase branch tip. Or: `git reset --hard ORIG_HEAD` if no 
other operations have run since the rebase.

**A7.** `git fsck --lost-found` finds ALL objects in `.git/objects/` that are not 
reachable from any ref (including reflog). It places them in `.git/lost-found/commit/` 
and `.git/lost-found/other/`. Use it when: you dropped a stash with `git stash clear`, 
or `git add`ed files that were never committed. It's the last resort when reflog fails.

**A8.** No — remotes do NOT have a reflog. Implication: `git push --force` overwrites 
commits with no server-side record of the old state. Workaround: create a backup branch 
before force-pushing (`git push origin main:backup/main-before-force-20260528`), or use 
`--force-with-lease` to at least ensure you're not overwriting others' work.

**A9.** `gc.reflogExpire` = 90 days (entries pointing to commits that ARE reachable from 
current branches). `gc.reflogExpireUnreachable` = 30 days (entries pointing to commits 
that are NOT reachable from any current branch — e.g., deleted branch tips). The shorter 
time for unreachable entries is because those commits are "dead" and you want faster cleanup.

**A10.** `git log -g` walks the **reflog** instead of the commit ancestry graph. So 
`git log -g --grep="login"` finds commits matching "login" in their message that appear 
in ANY reflog entry — even if those commits are no longer reachable from any branch 
(lost/deleted). Regular `git log --grep="login"` only searches currently reachable commits.

---

## 18. Connect Everything Backwards

### Topic 01 — `.git/` Physical Architecture

In Topic 01, when you first opened `.git/`, the `logs/` directory was listed in the 
directory tree. We noted that HEAD, refs/heads/main, and refs/stash were subdirectories. 
Now you know exactly what every file in `.git/logs/` contains — every pointer movement 
ever made, one line per movement.

```
Topic 01 introduced: .git/logs/ exists
Topic 20 explains:   .git/logs/HEAD, .git/logs/refs/heads/*, format of each line
```

### Topic 03 — The DAG and Refs

Topic 03 explained that branches are pointer files (one file = one SHA). The reflog is 
the HISTORY of those pointer files — every previous value they've ever held, with a 
timestamp and reason. The DAG is the relationship graph of commit objects. The reflog is 
the timeline of how ref pointers navigated that graph (and all the dangling nodes they 
touched along the way).

```
Topic 03: branches are pointer files (.git/refs/heads/main contains one SHA)
Topic 20: .git/logs/refs/heads/main is the history of every value that file has ever had
```

### Topic 08 — Branching

Topic 08 explained that `git checkout -b newbranch` creates a new file in 
`.git/refs/heads/newbranch` and adds an entry to `.git/logs/HEAD`. Now you know 
that recovering a deleted branch is just `git checkout -b` with a SHA from the reflog — 
because the file can be recreated at any SHA.

```
Topic 08: creating and deleting branches modifies .git/refs/heads/ files
Topic 20: you can always recreate a deleted branch from the reflog SHA — the objects persist
```

### Topic 11 — Rebasing

Topic 11 explained that rebase creates NEW commit objects with different SHAs and abandons 
the old ones. The reflog is why you can always undo a rebase — the old SHAs are recorded 
in the reflog entries for each "rebase (pick)" step. `ORIG_HEAD` (from Topic 12) is the 
quick undo mechanism, but reflog is the deep recovery tool.

```
Topic 11: rebase abandons old commit objects (new SHAs, old ones become unreachable)
Topic 20: those abandoned SHAs are in the reflog under "rebase (pick)" entries
```

### Topic 12 — Undoing Things

Topic 12 showed `git reset --soft/--mixed/--hard` but acknowledged "what if you reset 
too far?" Topic 20 answers that question: `git reset --hard HEAD@{N}` to undo any reset. 
The reflog is the undo-of-undo. `ORIG_HEAD` (saved by reset) is the quick path; reflog 
is the comprehensive path.

```
Topic 12: git reset moves the branch pointer (potentially losing commits)
Topic 20: every git reset writes to the reflog, so you can always reverse it
```

### Topic 13 — Stashing

Topic 13 showed that stashes are stored at `refs/stash` as special commits. Now you know 
that `refs/stash` has its own reflog at `.git/logs/refs/stash`, and even after 
`git stash drop` or `git stash clear`, the stash commit objects persist in `.git/objects/` 
(until GC). The stash reflog + `git fsck --lost-found` is how you recover dropped stashes.

```
Topic 13: stash is a commit at refs/stash
Topic 20: .git/logs/refs/stash records every stash push/pop/drop; use it to recover dropped stashes
```

---

## 19. Quick Reference Card

### Core Commands

| Command | What It Does |
|---------|-------------|
| `git reflog` | Show HEAD reflog (newest first) |
| `git reflog show main` | Show reflog for `main` branch specifically |
| `git reflog --date=iso` | Show with ISO timestamps instead of @{N} |
| `git reflog --date=relative` | Show with relative timestamps ("2 hours ago") |
| `git reflog stash` | Show stash reflog |
| `git log -g` | Walk reflogs instead of commit ancestry |
| `git log -g --oneline` | Compact reflog walk |
| `git log -g --all --oneline` | Walk ALL reflogs (HEAD + all branches) |
| `git log -g --grep="keyword"` | Find commits in reflog matching keyword |

### Recovery Commands

| Scenario | Command |
|----------|---------|
| Deleted a branch | `git checkout -b <name> HEAD@{N}` where N shows the last commit on that branch |
| Bad `reset --hard` | `git reset --hard HEAD@{1}` (or `HEAD@{N}` for the right state) |
| Undo a rebase | `git reset --hard ORIG_HEAD` (if recent) or `git reset --hard HEAD@{N}` (from reflog) |
| Dropped a stash | `git stash apply <sha>` (SHA from `git reflog stash`) |
| Stash cleared | `git fsck --unreachable \| grep commit`, then `git stash apply <sha>` |
| Truly lost objects | `git fsck --lost-found` → inspect `.git/lost-found/` |

### Reflog Notation

| Notation | Meaning |
|----------|---------|
| `HEAD@{0}` | Current HEAD |
| `HEAD@{1}` | Where HEAD was 1 operation ago |
| `HEAD@{N}` | Where HEAD was N operations ago |
| `HEAD@{yesterday}` | Where HEAD was at end of yesterday |
| `HEAD@{1 hour ago}` | Where HEAD was 1 hour ago |
| `HEAD@{2026-05-27}` | Where HEAD was on that date |
| `main@{3}` | Where main branch was 3 pointer-movements ago |

### Reflog vs History Navigation

| Notation | Type | Direction |
|----------|------|-----------|
| `HEAD~N` | Ancestry | N commits back in the DAG |
| `HEAD^2` | Ancestry | Second parent of merge commit |
| `HEAD@{N}` | Reflog | N operations back in time |
| `HEAD@{time}` | Reflog | At a specific time point |

### Expiry Configuration

| Config | Default | Meaning |
|--------|---------|---------|
| `gc.reflogExpire` | 90 days | Expire entries pointing to reachable commits |
| `gc.reflogExpireUnreachable` | 30 days | Expire entries pointing to unreachable commits |
| `gc.auto` | 6700 | Run GC when loose objects exceed this count |

```bash
# Extend retention globally:
git config --global gc.reflogExpire 180.days
git config --global gc.reflogExpireUnreachable 90.days

# Never expire (for critical repos):
git config gc.reflogExpire never
git config gc.reflogExpireUnreachable never
```

---

## 20. When Would I Use This At Work?

### Scenario 1 — The Monday Morning Panic

**Context:** It's Monday morning. A junior developer messaged you at 8am:
"I ran `git reset --hard` on Friday to 'clean up' and now I can't find my Thursday and 
Friday commits. I pushed nothing. The PR isn't created yet."

```bash
# You walk them through it on a call:
git reflog --date=iso
# Find the last commit entries from Thursday/Friday
# HEAD@{12}: 2026-05-22 17:45:00 +0530: commit: add payment webhook handler
# HEAD@{13}: 2026-05-22 16:30:00 +0530: commit: implement Stripe API client

git reset --hard HEAD@{12}  # or the specific SHA

# Their week of work is back. They're relieved. You've become their hero.
```

**Business impact:** Saved 2 days of a developer's work. Took 2 minutes.

---

### Scenario 2 — Post-Incident Force-Push Recovery

**Context:** A deployment script had a bug. It ran `git push --force origin main` with 
the wrong SHA. Production broke. The PR for the fix hasn't been merged. The last 7 good 
commits on main are "gone" from origin.

```bash
# On your local machine (you pulled earlier today):
git log --oneline -15
# You can see the 7 commits that got overwritten!
# The SHA of the last good commit: abc1234

# Push the recovery:
git push origin abc1234:refs/heads/main --force
# (After team confirmation and incident channel notification)
```

**Business impact:** Restored production. Recovery time: under 5 minutes.

---

### Scenario 3 — Investigating When a Bug Was Introduced

**Context:** A bug appeared. You know it wasn't there last week. You want to see exactly 
what commits were made to main in the last week.

```bash
# See what main looked like a week ago
git show main@{1 week ago}

# See all commits merged to main since 1 week ago
git log --oneline main@{1 week ago}..main

# Then you can narrow down and use git bisect (Topic 21) to find the exact commit
```

---

### Scenario 4 — Recovering Work During a Failed Interactive Rebase

**Context:** You were doing an interactive rebase with 8 commits. You edited one commit 
message, then got confused and ran `git rebase --abort` — but now the branch is in a 
weird state and you can't remember what you intended.

```bash
# See the full trace of what happened during the rebase
git reflog | head -20
# rebase (finish): ...
# rebase (pick): add feature
# rebase (reword): fix typo in commit message  ← this is where you edited
# rebase (pick): add tests
# rebase (start): checkout main

# The SHA before "rebase (start)" is your original branch tip
# Alternatively:
git reset --hard ORIG_HEAD
# ORIG_HEAD was set when the rebase started, before any changes
```

---

### Scenario 5 — Auditing Yourself Before a Code Review

**Context:** You've been working on a feature for 3 days, making messy commits. Before 
opening a PR, you want to see exactly what you've done and when.

```bash
# See your personal history of this branch
git reflog show feature/payment-v2 --date=iso
# Shows every commit, every amend, every reset on this branch
# Helps you understand what intermediate states you went through
# Useful for writing a clean PR description: "I added X, then Y, then refactored Z"
```

---

### Scenario 6 — The "Where Did That Feature Go?" Investigation

**Context:** Someone deleted a feature branch that was in progress. The developer who 
was working on it is on vacation. You need to find the work.

```bash
# Walk ALL reflogs looking for commits mentioning the feature
git log -g --all --oneline --grep="email-notifications"
# Finds: "3a4b5c6 HEAD@{15}: commit: add email notification templates"

# That commit still exists in objects! Recover it:
git checkout -b recovered/email-notifications 3a4b5c6
git log --oneline -10  # All their commits are here, in order
```

---

*Topic 20 Complete — You are now the person your entire team calls when commits are "lost".*

**Next: Topic 21 — git bisect: Debugging with Git (Binary Search Through History)**
*"You have 10,000 commits and a bug. You'll find the exact commit that introduced it in 14 steps."*
