# Topic 21 — git bisect: Debugging with Git
## "Binary Search Through History — Find the Exact Commit That Introduced Any Bug"

> **Phase 6 — Advanced & Principal-Level** | Builds on Topics 02, 03, 09, 11, 20

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The Encyclopedia Search](#1-eli5-anchor)
2. [Physical Architecture — `.git/` State During a Bisect](#2-physical-architecture)
3. [The Binary Search Algorithm — Why O(log n)](#3-the-binary-search-algorithm)
4. [Exact Data Flow — Step-by-Step Through a Bisect Session](#4-exact-data-flow)
5. [Core Commands Deep Dive](#5-core-commands-deep-dive)
6. [git bisect run — Fully Automated Bisect](#6-git-bisect-run)
7. [git bisect skip — Handling Untestable Commits](#7-git-bisect-skip)
8. [git bisect log and git bisect replay](#8-git-bisect-log-and-replay)
9. [git bisect visualize — Seeing the Search Range](#9-git-bisect-visualize)
10. [Bisecting with Merge Commits](#10-bisecting-with-merge-commits)
11. [Custom Terms — Beyond "good" and "bad"](#11-custom-terms)
12. [Byte-Level Internals — What Git Writes During Bisect](#12-byte-level-internals)
13. [Beginner → Production Examples](#13-beginner-to-production-examples)
14. [Mistakes → Root Cause → Fix → Prevention](#14-mistakes)
15. [Mental Model Checkpoint — 10 Questions](#15-mental-model-checkpoint)
16. [Connect Everything Backwards](#16-connect-everything-backwards)
17. [Quick Reference Card](#17-quick-reference-card)
18. [When Would I Use This At Work?](#18-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The Encyclopedia Search

You're in a library. You want to find the page where the word "photosynthesis" first 
appears. The encyclopedia has 1,000 pages. You could read from page 1 until you find it 
— that's up to 1,000 reads.

OR: Open to page 500. Is "photosynthesis" on this page or before? No — it's after 500. 
Now open to page 750. Still no? Open page 625. Yes! Now open to page 562. No. Page 593. 
Yes! Page 578. No. Page 585. Found it!

That's **binary search** — you've found it in ~10 steps instead of 1,000. Each step 
HALVES the remaining search space.

**`git bisect` applies this exact algorithm to your commit history:**

```
You tell Git: "This bug doesn't exist in commit A (good). It exists in commit Z (bad)."
Git says: "Here's the commit exactly in the middle — test it."
You say: "The bug is here too" → Git eliminates the second half.
Git says: "Here's the new middle" → You test again.
Repeat until Git says: "The first bad commit is X."
```

**The power:** 10,000 commits between "good" and "bad"? Found in 13 steps.
$\lceil \log_2(10000) \rceil = 14$ steps maximum.

**Real scenario:** Your app was working in `v2.0.0` (tagged 3 months ago). It's broken 
today. There are 847 commits between them. You run `git bisect` — you'll find the exact 
breaking commit in at most $\lceil \log_2(847) \rceil = 10$ steps. 10 test runs vs 
potentially 847 manual investigations.

---

## 2. Physical Architecture

### `.git/` State During an Active Bisect Session

**Before bisect starts — normal state:**
```
.git/
├── HEAD                    → "ref: refs/heads/main"
├── ORIG_HEAD               (may or may not exist)
├── refs/
│   ├── heads/
│   │   └── main            → current branch SHA
│   └── ...
└── logs/
    └── HEAD                → normal reflog
```

**After `git bisect start` — bisect state:**
```
.git/
├── HEAD                    → a specific SHA (detached HEAD — bisect checks out midpoints)
├── BISECT_HEAD             ← NEW: the commit currently being tested
├── BISECT_LOG              ← NEW: full log of all bisect commands (replay-able)
├── BISECT_NAMES            ← NEW: space-separated list of paths to bisect within
├── BISECT_TERMS            ← NEW: "good\nbad\n" (or custom terms)
├── BISECT_START            ← NEW: the branch you were on when bisect started
├── refs/
│   ├── heads/
│   │   └── main            → unchanged (your branch is safe)
│   └── bisect/
│       ├── bad             ← NEW: SHA of the known-bad commit
│       └── good-<sha>      ← NEW: one file per known-good SHA
└── logs/
    └── HEAD                → reflog grows with each checkout during bisect
```

**After `git bisect reset` — cleanup:**
```
.git/
├── HEAD                    → "ref: refs/heads/main" (restored to original branch)
├── BISECT_HEAD             ← DELETED
├── BISECT_LOG              ← DELETED  
├── BISECT_NAMES            ← DELETED
├── BISECT_TERMS            ← DELETED
├── BISECT_START            ← DELETED
└── refs/
    └── bisect/             ← DELETED (entire directory)
```

**PROVE IT — Watch the files appear:**

```bash
# Start a repo and make some commits
git init bisect-demo && cd bisect-demo
for i in 1 2 3 4 5 6 7 8; do
  echo "version $i" > app.txt
  git add app.txt
  git commit -m "commit $i"
done

# See the current .git state
ls .git/

# Start bisect
git bisect start

# NOW check what appeared
ls .git/BISECT*
# .git/BISECT_LOG  .git/BISECT_NAMES  .git/BISECT_TERMS  .git/BISECT_START

cat .git/BISECT_TERMS
# good
# bad

cat .git/BISECT_START
# main

cat .git/BISECT_LOG
# git bisect start
```

---

## 3. The Binary Search Algorithm

### Why `O(log₂ n)` Steps

Binary search works by maintaining a **range** `[low, high]` and repeatedly testing the 
midpoint. Each test eliminates half the remaining range.

```
If you have N commits between good and bad:

Steps needed = ⌈log₂(N)⌉

N = 2        → 1 step
N = 4        → 2 steps
N = 8        → 3 steps
N = 100      → 7 steps
N = 1,000    → 10 steps
N = 10,000   → 14 steps
N = 100,000  → 17 steps
N = 1,000,000→ 20 steps
```

**Contrast with linear search:**
- Linear: test every commit from `bad` backwards until you find where it was `good`
- Worst case: N tests (test all N commits)
- Average case: N/2 tests

```
VISUALIZING THE ALGORITHM:

Commits:  A - B - C - D - E - F - G - H - I - J
           ↑                                   ↑
          good                                bad

Step 1: Test E (middle of A..J)
  If E is good:  range becomes F..J, next test: H
  If E is bad:   range becomes A..E, next test: C

Step 1 result: E is GOOD
  Range: F..J (5 commits)

Step 2: Test H (middle of F..J)
  If H is good:  range becomes I..J, next test: I or J
  If H is bad:   range becomes F..H, next test: G

Step 2 result: H is BAD
  Range: F..H (3 commits)

Step 3: Test G (middle of F..H)
  If G is good:  ANSWER: H is the first bad commit
  If G is bad:   range becomes F..G (2 commits)

Step 3 result: G is BAD
  Range: F..G (2 commits)

Step 4: Test F (middle of F..G)
  If F is good:  ANSWER: G is the first bad commit
  If F is bad:   ANSWER: F is the first bad commit

DONE in 4 steps for 10 commits. ⌈log₂(10)⌉ = 4. ✓
```

### How Git Selects the Midpoint

Git doesn't just pick the commit exactly halfway through the linear sequence. It uses 
a smarter algorithm based on the **commit graph** (the DAG):

1. Git collects all commits reachable from `bad` but NOT from `good` (the "bisect range")
2. For each candidate commit, Git computes: how many commits would be eliminated if 
   this commit is marked `good`? How many if marked `bad`?
3. Git picks the commit that **minimizes the worst case** — the commit that makes both 
   halves as equal as possible

This is why bisect works correctly even with branching/merging histories — it's not a 
simple linear midpoint but a graph-aware midpoint.

```bash
# PROVE IT: See Git's reasoning about which commit to test next
git bisect visualize --oneline
# Shows the remaining commits Git is narrowing between

# Git's internal midpoint selection can be observed by watching
# which commits it picks — it's always minimizing the larger half
```

---

## 4. Exact Data Flow — Step-by-Step Through a Bisect Session

### Complete Walkthrough of a Bisect Session

**Setup:** You have a repo with 8 commits (C1 through C8). The bug was introduced at C5.

```
History: C1 - C2 - C3 - C4 - C5 - C6 - C7 - C8
                                   ↑
                         bug introduced here

You know: C1 is good, C8 is bad.
```

---

#### COMMAND: `git bisect start`

```
READS:    .git/HEAD → "ref: refs/heads/main"
WRITES:   .git/BISECT_TERMS → "good\nbad\n"
WRITES:   .git/BISECT_START → "main\n"
WRITES:   .git/BISECT_LOG → "git bisect start\n"
WRITES:   .git/BISECT_NAMES → "\n" (empty — no path filter yet)
NO CHECKOUT yet — HEAD doesn't move.

.git/refs/bisect/ does not exist yet.
```

---

#### COMMAND: `git bisect bad C8` (or `git bisect bad` if HEAD is already at C8)

```
READS:    .git/HEAD (to resolve "HEAD" if no SHA given)
RESOLVES: C8 = <sha-of-C8>
WRITES:   .git/refs/bisect/bad → "<sha-of-C8>\n"
APPENDS:  .git/BISECT_LOG → "git bisect bad <sha-of-C8>\n# bad: [<sha-of-C8>] commit 8\n"

.git/refs/bisect/bad now exists with C8's SHA.
HEAD has not moved yet.
```

---

#### COMMAND: `git bisect good C1`

```
READS:    resolves C1 to <sha-of-C1>
WRITES:   .git/refs/bisect/good-<sha-of-C1> → "<sha-of-C1>\n"
          (one file per good commit — naming: "good-" + the SHA)
APPENDS:  .git/BISECT_LOG → "git bisect good <sha-of-C1>\n# good: [<sha-of-C1>] commit 1\n"

NOW Git has both endpoints. It computes the midpoint:
  Range: C2, C3, C4, C5, C6, C7 (6 commits between C1 and C8, exclusive)
  Midpoint: C4 or C5 (whichever balances the halves — in a linear history, C4)

WRITES:   .git/BISECT_HEAD → "<sha-of-C4>\n"
PERFORMS: git checkout <sha-of-C4> (detached HEAD)
MODIFIES: .git/HEAD → "<sha-of-C4>\n" (detached HEAD state)
MODIFIES: working directory to match C4's tree
MODIFIES: .git/index to match C4's tree
APPENDS:  .git/logs/HEAD → reflog entry for the checkout

OUTPUT:
  Bisecting: 3 revisions left to test after this (roughly 2 steps)
  [<sha-of-C4>] commit 4
```

---

#### You test: the bug is NOT present at C4. COMMAND: `git bisect good`

```
READS:    .git/BISECT_HEAD → "<sha-of-C4>"
          (BISECT_HEAD is what's currently being tested)
WRITES:   .git/refs/bisect/good-<sha-of-C4> → "<sha-of-C4>\n"
APPENDS:  .git/BISECT_LOG → "git bisect good\n# good: [<sha-of-C4>] commit 4\n"

Range updated: C5, C6, C7 (3 commits between C4 and C8)
New midpoint: C6

WRITES:   .git/BISECT_HEAD → "<sha-of-C6>\n"  
PERFORMS: git checkout <sha-of-C6>
OUTPUT:
  Bisecting: 1 revision left to test after this (roughly 1 step)
  [<sha-of-C6>] commit 6
```

---

#### You test: bug IS present at C6. COMMAND: `git bisect bad`

```
READS:    .git/BISECT_HEAD → "<sha-of-C6>"
WRITES:   .git/refs/bisect/bad → "<sha-of-C6>\n" (OVERWRITES previous bad — only 1 bad at a time)
APPENDS:  .git/BISECT_LOG

Range: C5 (only C5 between C4 good and C6 bad)
New midpoint: C5 (the only one)

WRITES:   .git/BISECT_HEAD → "<sha-of-C5>\n"
PERFORMS: git checkout <sha-of-C5>
OUTPUT:
  Bisecting: 0 revisions left to test after this (roughly 0 steps)
  [<sha-of-C5>] commit 5
```

---

#### You test: bug IS present at C5. COMMAND: `git bisect bad`

```
READS:    .git/BISECT_HEAD → "<sha-of-C5>"
WRITES:   .git/refs/bisect/bad → "<sha-of-C5>\n"
APPENDS:  .git/BISECT_LOG

Range: empty — no commits between C4 (good) and C5 (bad)
C5 is now the FIRST commit where the bug is present.

PRINTS THE RESULT:
  <sha-of-C5> is the first bad commit
  commit <full-sha-of-C5>
  Author: ...
  Date:   ...

      commit 5

  :100644 100644 <old-tree-sha> <new-tree-sha> M    app.txt

Git does NOT automatically checkout anything yet — you're still at C5.
```

---

#### COMMAND: `git bisect reset`

```
READS:    .git/BISECT_START → "main\n"
PERFORMS: git checkout main (restores original branch)
DELETES:  .git/BISECT_HEAD
DELETES:  .git/BISECT_LOG
DELETES:  .git/BISECT_NAMES
DELETES:  .git/BISECT_TERMS
DELETES:  .git/BISECT_START
DELETES:  .git/refs/bisect/ (entire directory, all good-* and bad files)
MODIFIES: .git/HEAD → "ref: refs/heads/main\n" (back to original branch)
MODIFIES: working directory to match main's tip
OUTPUT:
  Previous HEAD position was <sha-of-C5> commit 5
  Switched to branch 'main'
```

**The full `.git/` state is restored to exactly what it was before `git bisect start`.**

---

## 5. Core Commands Deep Dive

### `git bisect start`

```
git bisect start [<bad> [<good>...]] [--] [<path>...]
│   │       │     │       │               │
│   │       │     │       │               └─ Limit bisect to changes in these paths
│   │       │     │       │                  Only commits touching these files are tested
│   │       │     └─ Optionally specify good commits immediately (can be multiple)
│   │       └─ Optionally specify bad commit immediately (saves a separate 'git bisect bad' call)
│   └─ Subcommand: begins a bisect session
└─ git binary
```

**Three ways to start:**

```bash
# Way 1: Two-step start (most common)
git bisect start
git bisect bad HEAD        # mark current state as bad
git bisect good v2.0.0     # mark last known good state

# Way 2: One-command start (convenient shorthand)
git bisect start HEAD v2.0.0
# Equivalent to the three commands above

# Way 3: Scoped to a subdirectory (only test commits that touch src/)
git bisect start HEAD v2.0.0 -- src/
# Git will skip commits that don't touch files in src/
```

---

### `git bisect bad` / `git bisect good`

```
git bisect bad [<commit>]
               │
               └─ If omitted, marks BISECT_HEAD (current checkout) as bad
                  If provided, marks that specific commit as bad
                  Only ONE bad commit is tracked at a time

git bisect good [<commit>...]
                │
                └─ If omitted, marks BISECT_HEAD as good
                   Can provide multiple commits: git bisect good a1b2 c3d4 e5f6
                   Multiple good commits allowed (narrow from multiple angles)
```

**Important:** After marking the first commit, Git immediately checks out the midpoint. 
After each subsequent `git bisect good` or `git bisect bad`, it checks out the next 
midpoint. You don't have to do any manual checkout — bisect handles it.

```bash
# Typical session rhythm:
git bisect start
git bisect bad                    # current HEAD is broken
git bisect good v1.8.0            # this version was fine

# Git checks out midpoint automatically. Now YOU:
#   1. Run your tests / reproduce the bug
#   2. Mark result:
git bisect good   # or: git bisect bad

# Repeat until Git announces the first bad commit
```

---

### `git bisect reset`

```
git bisect reset [<commit>]
                  │
                  └─ If omitted: checkout BISECT_START (the original branch)
                     If provided: checkout this specific commit after resetting bisect state
                     Use case: git bisect reset v2.0.0 — ends bisect AND checks out v2.0.0
```

**Always run `git bisect reset` when done.** If you forget:
- You're in detached HEAD state
- `.git/refs/bisect/` still exists
- Running `git bisect start` again will error: "You need to start by 'git bisect start'"

```bash
# If you forgot bisect reset and want to clean up manually:
git bisect reset

# Verify cleanup:
ls .git/BISECT* 2>/dev/null && echo "still active" || echo "cleaned up"
```

---

### `git bisect terms`

```bash
# See what the current terms are (good/bad or custom)
git bisect terms
# Output:
# Your current terms are good for the old state
# and bad for the new state.
```

---

## 6. `git bisect run` — Fully Automated Bisect

### Why Automation Changes Everything

Manual bisect requires you to test each midpoint. If you can write a test that returns:
- Exit code **0** = commit is "good" (bug not present)  
- Exit code **1-127** (non-zero, non-125) = commit is "bad" (bug present)
- Exit code **125** = skip this commit (can't test it)

Then `git bisect run` will run the entire bisect automatically with zero human interaction.

### Syntax

```
git bisect run <command> [args...]
│   │       │   │         │
│   │       │   │         └─ Arguments passed to the command at each step
│   │       │   └─ Script or command to run. Can be: bash script, Python, Node, make, etc.
│   │       └─ Subcommand: run automated bisect
│   └─ bisect namespace
└─ git binary

Exit code contract:
  0          → good (bug not present at this commit)
  1-124      → bad (bug present at this commit)
  125        → skip (this commit can't be tested)
  126, 127   → abort bisect (command not found or permission denied)
  128+       → abort bisect (Git signal values — treated as errors)
```

### Example 1 — Automating a Unit Test

```bash
# You have a failing test: npm test -- --grep "user authentication"
# When the test passes: exit 0 (good)
# When the test fails: exit 1 (bad)

git bisect start
git bisect bad HEAD
git bisect good v2.0.0

# Automate it:
git bisect run npm test -- --grep "user authentication"

# Git will:
# 1. Checkout the midpoint commit
# 2. Run: npm test -- --grep "user authentication"
# 3. Read the exit code → mark good or bad
# 4. Checkout the next midpoint
# 5. Repeat until the first bad commit is found
# 6. Announce the result
# 7. Return to BISECT_HEAD at the first bad commit (you still need bisect reset)
```

### Example 2 — Automating with a Shell Script

```bash
# Create a test script: test-bug.sh
cat > /tmp/test-bug.sh << 'EOF'
#!/bin/bash
# Test: does the login endpoint return 200?
# Exit 0 = works (good)
# Exit 1 = broken (bad)
# Exit 125 = skip (build fails, can't test)

# Build the project
npm run build 2>/dev/null
if [ $? -ne 0 ]; then
    exit 125  # Can't build — skip this commit
fi

# Run the specific test
npm test -- --testPathPattern="auth.test.js" --silent
if [ $? -eq 0 ]; then
    exit 0   # Test passed — good commit
else
    exit 1   # Test failed — bad commit  
fi
EOF
chmod +x /tmp/test-bug.sh

# Run automated bisect
git bisect start HEAD v2.0.0
git bisect run /tmp/test-bug.sh
```

### Example 3 — Single-Line Command

```bash
# If you can express the test as a one-liner:
git bisect run sh -c "grep -q 'initializeDB()' src/server.js"
# exit 0 if the line exists (good — maybe the bug is ABSENCE of this line)
# exit 1 if the line doesn't exist (bad)

# Check if a function exists in source:
git bisect run grep -q "handleRateLimit" src/middleware.js

# Run a build and check if binary exists:
git bisect run sh -c "make 2>/dev/null && ./bin/server --check-config"
```

### Example 4 — Python Test Script

```python
#!/usr/bin/env python3
# test_regression.py
import subprocess
import sys
import os

def build():
    result = subprocess.run(["pip", "install", "-e", "."], 
                           capture_output=True)
    return result.returncode == 0

def test_bug():
    """Returns True if bug is present (bad commit), False if not (good commit)"""
    result = subprocess.run(
        ["python", "-m", "pytest", "tests/test_auth.py::test_login", "-x", "--tb=no", "-q"],
        capture_output=True
    )
    return result.returncode != 0  # True = test failed = bug present

if not build():
    sys.exit(125)  # Can't build — skip

if test_bug():
    sys.exit(1)    # Bug present — bad commit
else:
    sys.exit(0)    # No bug — good commit
```

```bash
git bisect run python3 test_regression.py
```

### What `bisect run` Does Step by Step

```
LOOP:
  1. Check if bisect range is exhausted
     YES → print "first bad commit is X" → EXIT LOOP
     NO → continue

  2. Compute midpoint commit
  
  3. Checkout midpoint (same as manual bisect)
  
  4. Run: <your command>
  
  5. Read exit code:
     0   → git bisect good (internally)
     1-124 → git bisect bad (internally)
     125 → git bisect skip (internally)
     126,127 → ERROR: "bisect run failed" (script itself is broken)
     128+ → ERROR: abort
  
  6. GOTO 1

EXIT: Print the first bad commit SHA + full commit details
      NOTE: bisect run does NOT call bisect reset — YOU must do that
```

**After `bisect run` completes:**
```bash
# bisect run leaves you at the first bad commit in detached HEAD
# You must still reset:
git bisect reset
```

---

## 7. `git bisect skip` — Handling Untestable Commits

### When You Can't Test a Commit

Sometimes the midpoint commit:
- Doesn't compile (dependency not installed, build system changed)
- Is broken for an unrelated reason
- Is a merge commit that doesn't represent a real change
- Is a commit that partially implements a feature (always broken)

```bash
git bisect skip
# OR skip a specific commit:
git bisect skip <commit>
# OR skip a range:
git bisect skip <older>..<newer>  # skip all commits in this range
```

### What Skip Does to the Algorithm

```
BEFORE SKIP:
  Range: C4 (good) ... C8 (bad)
  Midpoint: C6 (current)

git bisect skip  (C6 is untestable)

AFTER SKIP:
  Skip list: {C6}
  Git picks a different midpoint near C6 but not C6 itself
  Maybe: C5 or C7 (whichever is further from both endpoints)

IMPORTANT: If ALL remaining commits are skipped, bisect cannot determine
the first bad commit precisely. It will say:
"There are only 'skip'ped commits left to test.
The first bad commit could be any of:
<sha1> <sha2> <sha3>
We cannot bisect more!"
```

### The Skip File on Disk

```bash
cat .git/BISECT_LOG
# After skip, you'll see:
# git bisect skip
# # skip: [<sha>] commit message
```

Git also writes to `.git/refs/bisect/skip-<sha>` for each skipped commit.

---

## 8. `git bisect log` and `git bisect replay`

### Saving Your Bisect Progress

```bash
# Print the full log of all bisect operations so far
git bisect log

# Output:
# git bisect start
# # bad: [<sha-of-C8>] commit 8
# git bisect bad <sha-of-C8>
# # good: [<sha-of-C1>] commit 1
# git bisect good <sha-of-C1>
# # good: [<sha-of-C4>] commit 4
# git bisect good
# # bad: [<sha-of-C6>] commit 6
# git bisect bad
# # bad: [<sha-of-C5>] commit 5
# git bisect bad
# # first bad commit: [<sha-of-C5>] commit 5

# SAVE THE LOG TO A FILE
git bisect log > bisect.log
```

### Replaying a Bisect Session

```bash
# If you need to reproduce the exact same bisect session
# (e.g., to share with a colleague, or to redo from scratch):
git bisect start
git bisect replay bisect.log

# Git will replay all the good/bad marks from the log file
# and arrive at the same result
```

### Why This Matters at Work

**Scenario:** You're bisecting a complex bug. You've done 6 steps and found the first bad 
commit. You want to share your investigation with your team lead:

```bash
git bisect log > investigation-2026-05-28.log
# Send investigation-2026-05-28.log to the team
# Any colleague can replay it: git bisect start && git bisect replay investigation-...log
```

**Scenario:** Your bisect got interrupted (you had to switch to a hotfix). Save progress, 
do the hotfix, then resume:

```bash
# Before switching:
git bisect log > /tmp/bisect-in-progress.log
git bisect reset

# Do the hotfix...

# Resume bisect from where you left off:
git bisect start
git bisect replay /tmp/bisect-in-progress.log
# Git re-marks all the good/bad commits and checks out the next midpoint
```

---

## 9. `git bisect visualize` — Seeing the Search Range

```bash
# Show the remaining commit range as a git log
git bisect visualize
# Opens gitk GUI (if available) showing remaining commits

# More useful: command-line visualization
git bisect visualize --oneline
# Output: all commits in the current bisect range
# dc4e5f6 add payment retry logic
# 3b4a2e1 refactor auth middleware
# a9f8e7d update JWT expiry
# ...

# With graph:
git bisect visualize --oneline --graph

# As a stat view:
git bisect visualize --stat
# Shows which files changed in each remaining candidate commit
```

This is extremely useful because it tells you:
1. How many commits are left to test
2. What those commits changed (helps you decide if manual testing is worth it)
3. You can often spot the culprit visually: "Oh, the only commit that touches auth.js is X"

---

## 10. Bisecting with Merge Commits

### The Challenge

In repos using merge commits (not squash merge), the DAG is non-linear. A merge commit 
has two parents:

```
     main: A - B - C - M - N - O
                  ↗
feature:  D - E - F

M is a merge commit with parents C and F.
```

**How does bisect handle this?**

Git bisect works on the **DAG**, not a linear sequence. It uses `git log --ancestry-path` 
logic to enumerate all commits in the range. Merge commits ARE included in the bisect range 
unless you exclude them.

### Bisecting a Merge Commit

When Git checks out a merge commit `M` as the bisect midpoint:
- The working tree reflects the merge result (both branches' changes combined)
- Testing at `M` tests the merged state (useful!)
- If you mark `M` as bad, Git knows the bug is either in `M`'s merge logic OR in D/E/F

### Using `--first-parent` to Skip Merges

```bash
# Bisect only along the mainline (first-parent path), ignoring feature branch commits
git log --oneline --first-parent  # see the mainline commits only

# To bisect only mainline commits:
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
# Then when Git checks out a merge commit, skip it:
git bisect skip  # skip the merge commit itself
```

### The Better Approach: Squash Merges

Repos that use squash merges (where each PR becomes a single commit on main) have a 
**perfectly linear history** on main. Bisect on squash-merge repos is:
1. Cleaner (no merge commits to deal with)
2. More actionable (each bisect step = one PR's worth of changes)
3. Easier to identify the responsible developer and PR

```
Squash merge history (easy to bisect):
  A - B - C - D - E - F - G  (each commit = one entire PR)
  ↑                       ↑
 good                    bad

Merge commit history (harder to bisect):
  A - B - C - M1 - D - M2 - E - M3  (harder — M1, M2, M3 are merges with histories)
```

---

## 11. Custom Terms — Beyond "good" and "bad"

### Why Custom Terms Help Communication

The default `good`/`bad` terminology assumes you're looking for a regression. But bisect 
can find ANY first commit where a property changed:

- "When did this feature get added?" → the first commit where the feature exists
- "When did this file exceed 500 lines?" → the first commit where `wc -l file.py` > 500
- "When did this commit author appear?" → using `--author`
- "Performance regression" → `fast`/`slow` is clearer than `good`/`bad`

### Setting Custom Terms

```bash
# Set custom terms when starting bisect
git bisect start --term-new=<term-for-new-state> --term-old=<term-for-old-state>

# Example: finding when a feature was introduced
git bisect start --term-new=has-feature --term-old=no-feature
git bisect has-feature HEAD   # current state: feature exists
git bisect no-feature v1.0.0  # old state: feature didn't exist
# Git checks out midpoints, you test and mark with:
git bisect has-feature  # or: git bisect no-feature

# Example: performance regression
git bisect start --term-old=fast --term-new=slow
git bisect slow HEAD
git bisect fast v3.1.0
# You mark each midpoint:
git bisect slow  # this version takes >2s
git bisect fast  # this version takes <500ms
```

### Checking Current Terms

```bash
cat .git/BISECT_TERMS
# has-feature
# no-feature
```

```bash
git bisect terms
# Your current terms are no-feature for the old state
# and has-feature for the new state.
```

---

## 12. Byte-Level Internals — What Git Writes During Bisect

### `.git/BISECT_LOG` — Full Session Record

Every bisect operation appends to this file. It's a valid input for `git bisect replay`.

```
# Format of BISECT_LOG:
git bisect start
# bad: [<full-sha>] <commit message>
git bisect bad <full-sha>
# good: [<full-sha>] <commit message>
git bisect good <full-sha>
# good: [<full-sha>] <commit message>
git bisect good
# bad: [<full-sha>] <commit message>
git bisect bad
# first bad commit: [<full-sha>] <commit message>
```

Notes:
- Lines starting with `#` are comments (explanatory, not executed on replay)
- Lines starting with `git bisect` are the executable commands (run on replay)
- The `bad`/`good` entries without SHAs use the current BISECT_HEAD

### `.git/refs/bisect/bad` — Single File, Current Bad Boundary

```bash
cat .git/refs/bisect/bad
# <40-char SHA>\n
# This is just like a branch pointer — a plain text file with one SHA.
# Only ONE bad commit at a time. Each `git bisect bad` OVERWRITES this file.
```

### `.git/refs/bisect/good-<sha>` — One File Per Good Commit

```bash
ls .git/refs/bisect/
# bad
# good-a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5
# good-7c3e8f0d2a1b4c5e6f7d8e9a0b1c2d3e4f5a6b7c

# Each file contains the SHA it represents
cat .git/refs/bisect/good-a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5
# a3f2c1d8e4b6f9c2a1d3e5f7b8c9d0e1f2a3b4c5
```

Multiple good commits are allowed because you might know several safe starting points.
The bisect range is: all commits reachable from `bad` but NOT reachable from ANY `good-*`.

### `.git/BISECT_HEAD` — Current Checkout Target

```bash
cat .git/BISECT_HEAD
# <40-char SHA>\n
# The commit currently checked out for testing.
# This is what HEAD@{0} resolves to during bisect.
# Note: .git/HEAD itself also points to this SHA (detached HEAD state).
```

### `.git/BISECT_TERMS` — The Vocabulary File

```bash
cat .git/BISECT_TERMS
# good
# bad
# Two lines: first line = term for "old/working state", second = "new/broken state"
# With custom terms:
# fast
# slow
```

### `.git/BISECT_START` — Return Address

```bash
cat .git/BISECT_START
# main
# The branch name (or SHA if detached HEAD) to return to on `git bisect reset`
```

### How Git Computes the Next Midpoint (Internal Algorithm)

```
1. Collect the "bisect range":
   R = {all commits reachable from refs/bisect/bad}
     - {all commits reachable from any refs/bisect/good-*}
     - {all commits in refs/bisect/skip-*}

2. For each commit C in R:
   - count_good_if_C_is_good = |{commits reachable from C} ∩ R|
   - count_bad_if_C_is_bad = |R| - count_good_if_C_is_good
   - worst_case(C) = max(count_good_if_C_is_good, count_bad_if_C_is_bad)

3. Choose the commit C* that minimizes worst_case(C*)
   This is the "most equal split" — the true binary search midpoint in the DAG

4. Checkout C* and set BISECT_HEAD = C*

5. Print: "Bisecting: N revisions left to test after this (roughly M steps)"
   where N = |R| - 1 (remaining to test after this one)
   and M = ⌈log₂(N)⌉
```

---

## 13. Beginner → Production Examples

### Beginner Example

**Setup:** Simple Node.js project. A function `calculateTotal()` was working but now 
returns wrong values. You know it worked 2 weeks ago.

```bash
# Find the last known good tag
git tag --list
# v1.0.0  v1.1.0  v1.2.0

# Test v1.2.0 manually: works. Test HEAD: broken.
git bisect start
git bisect bad HEAD
git bisect good v1.2.0

# Git checks out a midpoint, say: "Bisecting: 7 revisions left to test..."
# [abc1234] add discount calculation

# Test: run node -e "const {calculateTotal} = require('./'); console.log(calculateTotal([10, 20]))"
# Output: 31 (wrong — should be 30)
git bisect bad

# Git checks out next midpoint: "Bisecting: 3 revisions left..."
# [def5678] refactor order model

# Test: output is 30 (correct!)
git bisect good

# ... (2 more steps) ...

# Git announces:
# e7f8a9b is the first bad commit
# commit e7f8a9b...
# Author: Junior Dev <junior@co.io>
# Date: ...
#     add tax calculation feature

# Now inspect the commit:
git show e7f8a9b
# Diff shows: calculateTotal now adds taxRate but the default was set to 0.01 instead of 0

git bisect reset
# You know exactly which commit to fix or revert!
```

---

### Production Example

**Context:** You're the lead backend engineer at a fintech company. A performance 
regression was reported: the `/api/transactions` endpoint now takes 4-6 seconds instead 
of 200ms. It was fast in the `v4.2.0` release (3 months ago). There are **312 commits** 
between `v4.2.0` and `HEAD`. The team can't find it manually.

```bash
# STEP 1: Write the test script
cat > /tmp/perf-test.sh << 'SCRIPT'
#!/bin/bash
# Build the project
cd /repo
npm run build:fast 2>/dev/null
if [ $? -ne 0 ]; then
    echo "BUILD FAILED — skipping"
    exit 125
fi

# Start server in background (with timeout)
timeout 30 npm start &
SERVER_PID=$!
sleep 3  # wait for startup

# Check if server started
if ! kill -0 $SERVER_PID 2>/dev/null; then
    echo "SERVER FAILED TO START — skipping"
    exit 125
fi

# Time the endpoint
DURATION=$(curl -o /dev/null -s -w "%{time_total}" \
  -H "Authorization: Bearer $TEST_TOKEN" \
  "http://localhost:3000/api/transactions?limit=50")

# Kill the server
kill $SERVER_PID 2>/dev/null
wait $SERVER_PID 2>/dev/null

# If response time > 1 second: bad commit
# Using awk for float comparison (bash can't do float comparison)
IS_SLOW=$(echo "$DURATION" | awk '{print ($1 > 1.0) ? "1" : "0"}')

if [ "$IS_SLOW" = "1" ]; then
    echo "SLOW: ${DURATION}s — BAD"
    exit 1
else
    echo "FAST: ${DURATION}s — GOOD"
    exit 0
fi
SCRIPT
chmod +x /tmp/perf-test.sh

# STEP 2: Run automated bisect
export TEST_TOKEN="your-test-token"

git bisect start
git bisect bad HEAD
git bisect good v4.2.0

git bisect run /tmp/perf-test.sh
```

**Expected output after ~8-9 automated test runs:**
```
running /tmp/perf-test.sh
FAST: 0.18s — GOOD
running /tmp/perf-test.sh
SLOW: 4.23s — BAD
...
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 is the first bad commit
commit a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
Author: Backend Dev <dev@fintech.io>
Date:   Thu Apr 10 11:23:45 2026 +0530

    optimize transactions query with eager loading

:100644 100644 ... M    src/models/transaction.js
```

**What you do with the result:**
```bash
# Look at the "optimization" that actually caused the regression
git show a1b2c3d4
# See the diff: the "eager loading" is loading ALL related records instead of paginating
# → You've found the root cause in ~9 automated test runs instead of 312 manual investigations
# → Time saved: potentially days of debugging

git bisect reset
```

**Business impact:** Identified a 4-second performance regression's root cause in 
~20 minutes of automated testing. Without bisect: could take days of manual investigation 
across 312 commits.

---

## 14. Mistakes → Root Cause → Fix → Prevention

---

### Mistake 1: Forgetting `git bisect reset` After the Session

**Scenario:** You found the bad commit. You `git show` it, make a note, and then 
start coding. Later, `git status` shows you're in detached HEAD. Your commits aren't 
going anywhere. Your branch pointer hasn't moved.

**Root Cause:** After bisect finds the first bad commit, you're still in detached HEAD 
state at that commit. The `.git/refs/bisect/` directory still exists.

**Broken state:**
```bash
git branch
# * (HEAD detached at a1b2c3d)
#   main

git commit -m "fix the bug"
# [detached HEAD b2c3d4e] fix the bug
# ← commit was made but it's on no branch! It will be lost.
```

**Fix:**
```bash
# If you've already made commits in detached HEAD:
git log --oneline -5  # note the SHA of your fix
git bisect reset      # returns to main
git cherry-pick b2c3d4e  # apply your fix to main
```

**Prevention:** Make `git bisect reset` the very first thing you do after bisect reports 
the result — before you read the commit, before you inspect the diff.

---

### Mistake 2: Marking a Commit Wrong (Bad Instead of Good)

**Scenario:** You're testing commit X. The test is slow or ambiguous. You mark it bad. 
Two steps later you realize you tested the wrong thing and X should have been good. Now 
the bisect result is wrong.

**Root Cause:** Bisect has no "undo last mark" command.

**Fix:** Use the log and replay:
```bash
# Save current log
git bisect log > bisect-progress.log

# Edit bisect-progress.log: find the wrong mark and fix it
# Change: "git bisect bad" to "git bisect good" for the wrong commit

# Restart and replay from the corrected log:
git bisect reset
git bisect start
git bisect replay bisect-progress.log
# Bisect will proceed from your corrected log
```

**Prevention:** Before marking, always verify you're testing what you think you're testing:
```bash
git log --oneline -1  # confirm which commit is checked out
# THEN run your test, THEN mark
```

---

### Mistake 3: Bisecting Without a Reproducible Test

**Scenario:** The bug is "sometimes the UI flickers." You run bisect manually, testing 
by eye. After 8 steps you mark a commit bad. But was the flicker actually there? You're 
not sure. The result might be noise.

**Root Cause:** Non-deterministic testing produces unreliable bisect results. Bisect 
requires consistent, reproducible test outcomes.

**Fix:** Before running bisect, create a deterministic test case. If you can't make the 
bug deterministic, bisect is not the right tool.

**For flaky tests:**
```bash
# Use git bisect skip to mark commits you couldn't test definitively
git bisect skip

# OR run the test multiple times and use a confidence threshold:
# In your bisect run script:
FAIL_COUNT=0
for i in 1 2 3; do
  ./run-test.sh && ((FAIL_COUNT++))
done
if [ $FAIL_COUNT -ge 2 ]; then
  exit 1  # bad (failed 2/3 times)
else
  exit 0  # good (passed 2/3 times)
fi
```

**Prevention:** Write the test script FIRST, verify it reliably produces exit 0 on a 
known-good commit and exit 1 on a known-bad commit, THEN start bisect.

---

### Mistake 4: Bisect Finding the Wrong Commit Due to Merge Commits

**Scenario:** Your repo uses merge commits. Bisect identifies commit M (a merge commit) 
as the first bad commit. But M is just a merge — the actual bug is in one of the commits 
on the merged branch. You look at M's diff and it shows only the merge resolution, not 
the bug.

**Root Cause:** When bisect identifies a merge commit, the "bug" is in the merge's 
history — somewhere in the commits that were merged in. You need to bisect further within 
the merged range.

**Fix:**
```bash
# Bisect found M (merge commit) as "first bad"
git show M  # See the merge commit
git log --oneline M^1..M  # See what was merged: parent1 to M
git log --oneline M^2..M  # Or the other parent

# Start a NEW bisect within the merged range:
git bisect reset
git bisect start
git bisect bad M
git bisect good M^1   # first parent (main before the merge)
# This bisects within the feature branch commits
```

**Prevention:** Use `git bisect visualize` before marking to see if the current commit 
is a merge commit:
```bash
git bisect visualize --oneline | head -5
# If the midpoint shows "Merge branch '...'", consider skipping it:
git bisect skip
```

---

### Mistake 5: Running `git bisect run` with a Script That Has Side Effects

**Scenario:** Your bisect run script modifies the database, sends emails, or creates 
external resources. It runs 10 times during bisect. You've now sent 10 test emails to 
customers or corrupted your dev database.

**Root Cause:** The bisect run script runs in the repo's actual environment at each 
checkout. If the script has side effects, those happen at every step.

**Fix:** For the current session, stop the bisect and clean up the side effects manually.

**Prevention:**
```bash
# Use a clean test database, mock external services, or use integration test flags:
git bisect run sh -c "NODE_ENV=test DATABASE_URL=postgres://localhost/test_bisect npm test"
# The test environment should be isolated from production
```

Also: always test your bisect script ONCE manually before running automated bisect:
```bash
# Test the script at the known-bad HEAD:
/tmp/your-test-script.sh
echo "Exit code: $?"  # should be 1 (bad)

# Test at the known-good tag:
git checkout v2.0.0
/tmp/your-test-script.sh
echo "Exit code: $?"  # should be 0 (good)

git checkout main
# NOW start bisect run with confidence
```

---

## 15. Mental Model Checkpoint — 10 Questions

Try to answer these from memory before checking the answers.

**Q1.** What is the maximum number of test steps needed to find a bug among 10,000 commits 
using `git bisect`? Show the formula.

**Q2.** Name all the files that Git creates in `.git/` when you run `git bisect start`, 
`git bisect bad`, and `git bisect good`. What is the purpose of each?

**Q3.** What is the exit code contract for `git bisect run`? What happens if your script 
returns exit code 125? What about 127?

**Q4.** You're mid-bisect at step 5 of 10. Your laptop dies and you lose the terminal. 
How do you resume the bisect session? Name the file that makes this possible and the 
command to use it.

**Q5.** What is the difference between `git bisect bad` and `git bisect bad HEAD~2`? 
When would you use the second form?

**Q6.** You ran `git bisect start HEAD v2.0.0` but after 3 steps you realize you got 
the direction wrong — the current HEAD is actually good and v2.0.0 was bad. What do you do?

**Q7.** Git announces that "commit M is the first bad commit" but M is a merge commit. 
What does this mean and how do you find the actual buggy commit within the merged history?

**Q8.** Describe the `git bisect visualize --oneline` command. What does it show and 
when is it most useful?

**Q9.** What happens to `.git/` when you run `git bisect reset`? List every file/directory 
that is deleted.

**Q10.** What is the difference between `git bisect skip` and `git bisect bad` when 
your build fails at the current commit? Which should you use and why?

---

**Answers:**

**A1.** $\lceil \log_2(10000) \rceil = 14$ steps maximum. Formula: $\lceil \log_2(N) \rceil$ where N is the number of commits in the bisect range.

**A2.** Files created:
- `.git/BISECT_TERMS` — stores the term names ("good\nbad\n" or custom)
- `.git/BISECT_START` — stores the branch name to return to on `bisect reset`
- `.git/BISECT_LOG` — full append-only log of all bisect commands (for replay)
- `.git/BISECT_NAMES` — path filter (empty if no path specified to `bisect start`)
- `.git/BISECT_HEAD` — SHA of current commit being tested (after first midpoint checkout)
- `.git/refs/bisect/bad` — SHA of the known-bad boundary commit
- `.git/refs/bisect/good-<sha>` — one file per known-good commit (named by SHA)

**A3.** Exit code contract: `0` = good (bug not present); `1-124` = bad (bug present); `125` = skip this commit (can't test); `126` or `127` = abort bisect (command not found / permission denied); `128+` = abort bisect (fatal Git signal error). Exit 125 causes `git bisect skip` to be called internally — Git picks a different midpoint.

**A4.** Use `git bisect log > saved.log` BEFORE the laptop dies (this should be standard practice). If you already saved it: `git bisect start && git bisect replay saved.log`. The file is `.git/BISECT_LOG`. Even if you didn't save it manually, the file itself is at `.git/BISECT_LOG` and survived the session — `cat .git/BISECT_LOG` then `git bisect replay .git/BISECT_LOG` (but first do `git bisect reset` then `git bisect start`).

**A5.** `git bisect bad` marks the currently checked-out commit (BISECT_HEAD) as bad. `git bisect bad HEAD~2` marks the commit 2 parents back from current HEAD as bad. Use the second form when you want to mark a specific commit without first checking it out — e.g., at the start of a session: `git bisect start; git bisect bad HEAD; git bisect good v1.0.0` vs `git bisect start HEAD v1.0.0` (the shorthand).

**A6.** Run `git bisect reset` to abort the current session. Then start over: `git bisect start; git bisect bad v2.0.0; git bisect good HEAD`. The terms "good" and "bad" don't have to correspond to "old" and "new" — they correspond to "state where the property is ABSENT" (good) and "state where the property is PRESENT" (bad). If you're looking for when something was ADDED, your "good" is the commit WITHOUT the thing and "bad" is the commit WITH it.

**A7.** A merge commit being "first bad" means the property changed in the MERGE itself or in the commits merged in. Since merge commits often don't show individual changes clearly, start a new bisect within the merged range: `git bisect reset; git bisect start; git bisect bad M; git bisect good M^1` (where M^1 is the main-branch parent of the merge). This bisects within the feature branch commits that were merged in.

**A8.** `git bisect visualize --oneline` shows all commits currently in the bisect range (between the current good and bad boundaries) in one-line format. Most useful when: (a) you want to see how many commits remain, (b) you want to visually scan if any commit message obviously describes the bug, (c) you want to see if the next midpoint is a merge commit (so you can decide to skip it).

**A9.** `git bisect reset` deletes: `.git/BISECT_HEAD`, `.git/BISECT_LOG`, `.git/BISECT_NAMES`, `.git/BISECT_TERMS`, `.git/BISECT_START`, and the entire `.git/refs/bisect/` directory (including `bad`, `good-*`, and `skip-*` files). It also runs `git checkout <BISECT_START>` to restore the original branch.

**A10.** Use `git bisect skip`, not `git bisect bad`. If your build fails, you DON'T KNOW if the bug is present — the commit is untestable. `git bisect bad` would incorrectly claim "the bug is here" when you actually have no information. `git bisect skip` correctly tells Git "I can't test this commit — pick a different one." Git will select a nearby commit that avoids the broken one and continue the bisect.

---

## 16. Connect Everything Backwards

### Topic 02 — Git Objects

`git bisect` operates on the commit object graph. When Git computes the "bisect range," 
it's doing a DAG traversal of commit objects — following parent pointers in `.git/objects/`. 
The `refs/bisect/bad` and `refs/bisect/good-*` files are just like branch pointers (Topic 08) 
— they hold SHAs of commit objects. Bisect's "first bad commit" result is a commit object 
SHA, which you then inspect with `git show` or `git cat-file`.

```
Topic 02: commit objects have parent pointers, stored in .git/objects/
Topic 21: bisect traverses these parent pointers to compute the range and midpoint
```

### Topic 03 — The DAG and Refs

Bisect's range is computed entirely on the DAG: "all commits reachable from bad but not 
from any good." This is the same reachability concept introduced in Topic 03. The 
`refs/bisect/` directory is a new category of ref alongside `refs/heads/` and `refs/tags/` — 
it uses the same plain-text SHA-file format.

```
Topic 03: refs are SHA pointer files, reachability determines commit visibility
Topic 21: bisect creates its own ref namespace (refs/bisect/) using the same pointer format
```

### Topic 09 — Merging

Topic 09 introduced merge commits with two parents. This is why bisect on merge-heavy repos 
can identify a merge commit as the "first bad" — the DAG traversal includes merge commits 
as regular nodes. The `M^1` (first parent) and `M^2` (second parent) syntax from Topic 09 
is how you scope a follow-up bisect within a merged branch.

```
Topic 09: merge commits have two parents (M^1 and M^2)
Topic 21: when bisect finds a merge commit as "first bad," use M^1..M to bisect the merged range
```

### Topic 11 — Rebasing

A rebased branch has a clean linear history — each commit is a logical unit of change. 
This makes bisect on rebased histories extremely effective (each step = one logical change, 
easy to understand). Squash-merged PRs (a form of rebase onto main) are the gold standard 
for bisect-friendly history.

```
Topic 11: rebase creates linear history with clean, atomic commits
Topic 21: linear history = each bisect step identifies one logical change = easier debugging
```

### Topic 20 — Reflog

After bisect finds the first bad commit, you'll often want to look at its context. The 
reflog (Topic 20) records every checkout that bisect performed — so you can trace back 
through every commit Git tested during the bisect session. Also: if you forget `bisect reset` 
and make commits in detached HEAD, use `git reflog` to find those commits before they're 
lost.

```
Topic 20: reflog records every HEAD movement
Topic 21: bisect generates many HEAD movements (checkouts); reflog tracks all of them
```

---

## 17. Quick Reference Card

### Session Management

| Command | What It Does |
|---------|-------------|
| `git bisect start` | Begin a new bisect session |
| `git bisect start HEAD v1.0.0` | Start with bad=HEAD, good=v1.0.0 (shorthand) |
| `git bisect start HEAD v1.0.0 -- src/` | Start with path filter (only changes to src/) |
| `git bisect bad` | Mark current checkout (BISECT_HEAD) as bad |
| `git bisect bad <commit>` | Mark a specific commit as bad |
| `git bisect good` | Mark current checkout as good |
| `git bisect good <commit>` | Mark a specific commit as good |
| `git bisect skip` | Skip current commit (untestable) |
| `git bisect skip <commit>` | Skip a specific commit |
| `git bisect skip <old>..<new>` | Skip a range of commits |
| `git bisect reset` | End bisect session, restore original branch |
| `git bisect reset <commit>` | End bisect and checkout a specific commit |

### Inspection

| Command | What It Does |
|---------|-------------|
| `git bisect log` | Print full log of all bisect operations |
| `git bisect log > file.log` | Save log to file (for replay) |
| `git bisect replay file.log` | Replay saved bisect session |
| `git bisect visualize` | Open gitk showing remaining commit range |
| `git bisect visualize --oneline` | Show remaining range as compact log |
| `git bisect visualize --stat` | Show remaining range with file change stats |
| `git bisect terms` | Show current good/bad term names |

### Automation

| Command | What It Does |
|---------|-------------|
| `git bisect run <script>` | Run script automatically at each step |
| `git bisect run npm test` | Automate with a npm test command |
| `git bisect run sh -c "grep -q 'fn()' src/app.js"` | Inline command as test |

### Custom Terms

| Command | What It Does |
|---------|-------------|
| `git bisect start --term-new=slow --term-old=fast` | Custom terms for new/old states |
| `git bisect slow` | Mark current as slow (= bad equivalent) |
| `git bisect fast` | Mark current as fast (= good equivalent) |

### Exit Codes for `bisect run` Scripts

| Exit Code | Bisect Result |
|-----------|---------------|
| `0` | good (bug absent) |
| `1-124` | bad (bug present) |
| `125` | skip (untestable) |
| `126, 127` | abort (script error) |
| `128+` | abort (fatal error) |

### Files Created During Bisect

| File | Contents |
|------|----------|
| `.git/BISECT_START` | Branch to return to on reset |
| `.git/BISECT_TERMS` | good/bad term names (2 lines) |
| `.git/BISECT_LOG` | Full session replay log |
| `.git/BISECT_HEAD` | SHA of current test commit |
| `.git/BISECT_NAMES` | Path filter (if specified) |
| `.git/refs/bisect/bad` | SHA of known-bad boundary |
| `.git/refs/bisect/good-<sha>` | SHA of each known-good commit |

### Steps Formula

$$\text{Max steps} = \lceil \log_2(N) \rceil \text{ where } N = \text{commits in range}$$

| Commits in Range | Max Steps |
|-----------------|-----------|
| 10 | 4 |
| 100 | 7 |
| 1,000 | 10 |
| 10,000 | 14 |
| 100,000 | 17 |

---

## 18. When Would I Use This At Work?

### Scenario 1 — The Mystery Performance Regression

**Context:** Your API's P99 latency went from 150ms to 2,400ms. Monitoring shows it 
happened sometime in the last 3 weeks (220 commits ago). No one remembers making any 
performance-related change.

**Action:**
```bash
# Write a load test script that times the endpoint:
cat > /tmp/perf-check.sh << 'EOF'
#!/bin/bash
docker-compose up -d --quiet 2>/dev/null
sleep 5
LATENCY=$(ab -n 100 -c 10 http://localhost:3000/api/v1/users 2>/dev/null | 
          grep "99%" | awk '{print $2}')
docker-compose down --quiet 2>/dev/null
[ "$LATENCY" -gt "500" ] && exit 1 || exit 0
EOF

git bisect start HEAD v3.0.0
git bisect run /tmp/perf-check.sh
# → Found in ~8 automated test runs: "add N+1 query to user serializer"
```

---

### Scenario 2 — The Non-Deterministic Test Failure in CI

**Context:** Tests pass locally but fail in CI ~30% of the time. It started 3 sprints ago. 
You need to find when the flakiness was introduced.

**Action:**
```bash
# CI environment variable that makes the flakiness deterministic:
git bisect start HEAD sprint-22-start
git bisect run sh -c "CI=true FORCE_SEED=12345 npm test 2>&1 | grep -q 'PASS' && exit 0 || exit 1"
# The seed makes the previously random behavior deterministic
# Bisect finds the commit that introduced a race condition in the test setup
```

---

### Scenario 3 — Finding When a Feature Was Added (Not a Bug)

**Context:** A compliance audit asks: "When was this cryptographic algorithm added to 
the codebase?" You need the exact commit with author, date, and code review link.

```bash
# Custom terms: "absent" / "present" instead of good/bad
git bisect start --term-old=absent --term-new=present

git bisect present HEAD      # it's in current code
git bisect absent v1.0.0     # it's not in the oldest version

git bisect run grep -rq "AES-256-GCM" src/crypto/
# exit 0 when found = "present"
# exit 1 when not found = "absent"

# Result: exact commit + author + date for the compliance report
git bisect reset
```

---

### Scenario 4 — Post-Deploy Incident Response

**Context:** Production deployment broke at 2am. You deployed 40 commits from the last 
sprint. On-call engineer needs to identify the root cause FAST to decide: 
rollback or hotfix?

```bash
# On-call workflow: bisect takes 6 steps for 40 commits
git bisect start HEAD v4.8.2  # last known good deploy

# Run the production smoke test suite against each commit
git bisect run npm run test:smoke -- --bail
# ~6 minutes of automated testing to find the exact commit
# Decision: one commit changed a DB migration — rollback is safe

git bisect reset
# Deploy v4.8.2, communicate RCA to stakeholders
```

---

### Scenario 5 — Cross-Team Bug Assignment

**Context:** The mobile team reports that the iOS app broke when it parses the 
`/api/v2/profile` response. The API team and mobile team are blaming each other. 
You need objective evidence of which deploy broke it.

```bash
git bisect start
git bisect bad HEAD
git bisect good $(git log --oneline | grep "before iOS team reported" | head -1 | awk '{print $1}')

git bisect run sh -c "
  curl -s http://localhost:3000/api/v2/profile | 
  python3 -c 'import sys,json; d=json.load(sys.stdin); assert \"avatarUrl\" in d' && 
  exit 0 || exit 1
"
# Finds exact commit: "change profile response format: rename avatar_url to profileImage"
# → API team made a breaking change → clear ownership for the fix
```

---

*Topic 21 Complete — You can now find any bug in any codebase in O(log n) steps.*

**Next: Topic 22 — git blame & git log Mastery: Codebase Archaeology**
*"Answer 'who wrote this?', 'why does this line exist?', and 'when did this behavior change?' — permanently."*
