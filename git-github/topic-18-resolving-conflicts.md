# Topic 18 — Resolving Conflicts Like a Senior

> **Curriculum**: Git & GitHub Deep Mastery — Zero → Principal Engineer  
> **Phase 5**: Team & Company-Level Git (Topics 15–19)  
> **Prerequisites**: Topic 05 (Three Trees / Index), Topic 09 (Merging), Topic 11 (Rebasing), Topic 17 (Merge Strategies)

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The Document Collaboration Analogy](#1-eli5-anchor)
2. [The 3-Way Merge Algorithm — How Git Decides What Conflicts](#2-the-3-way-merge-algorithm)
3. [Physical Architecture — `.git/` During a Conflict](#3-physical-architecture)
4. [The Index Stages — The Deepest Layer](#4-the-index-stages)
5. [Conflict Markers Decoded — Every Byte Explained](#5-conflict-markers-decoded)
6. [The `diff3` Conflict Style — Seeing the Ancestor](#6-the-diff3-conflict-style)
7. [`ours` vs `theirs` — The Most Confusing Terms in Git](#7-ours-vs-theirs)
8. [Resolving Conflicts — The 4 Approaches](#8-resolving-conflicts)
9. [Merge Tools — vimdiff, VS Code, kdiff3](#9-merge-tools)
10. [Complex Conflict Types — Beyond Text Conflicts](#10-complex-conflict-types)
11. [Rebase Conflicts vs Merge Conflicts](#11-rebase-conflicts-vs-merge-conflicts)
12. [Cherry-Pick Conflicts](#12-cherry-pick-conflicts)
13. [Production Scenarios — Real Conflict Situations](#13-production-scenarios)
14. [Mistakes → Root Cause → Fix → Prevention](#14-mistakes)
15. [Byte-Level: The Index in Conflict State](#15-byte-level-internals)
16. [Mental Model Checkpoint](#16-mental-model-checkpoint)
17. [Connect Everything Backwards](#17-connect-everything-backwards)
18. [Quick Reference Card](#18-quick-reference-card)
19. [When Would I Use This At Work?](#19-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The Shared Google Doc Analogy

Imagine two people are editing the same Google Doc at the same time, but WITHOUT the live-collaboration feature — they each downloaded a copy.

- Alice downloaded version 5 (the original), changed paragraph 3 to say "The sky is red."
- Bob also downloaded version 5, changed paragraph 3 to say "The sky is purple."

When they try to combine their changes back into the shared document, there is a conflict: **both changed the same paragraph starting from the same original text, but made different choices.**

A human editor now must decide: use Alice's "red"? Use Bob's "purple"? Or blend them into "The sky is reddish-purple"?

**Git is the editor.** When Git cannot automatically determine the right result — because two people changed the SAME lines starting from the SAME ancestor — it stops and says: **"I need a human to make this decision."** That's a conflict.

The key insight: **Git only conflicts when both parties changed the same lines.** If Alice changed paragraph 3 and Bob changed paragraph 7, Git combines them automatically — no conflict. Conflicts are specifically the cases where Git lacks the information to choose.

**What Git actually does** (the 3-way merge):
1. Start with the original version (the common ancestor)
2. Compute what Alice changed from the original
3. Compute what Bob changed from the original
4. Apply BOTH sets of changes
5. If they don't overlap → automatic merge
6. If they overlap (same lines changed differently) → conflict, stop, ask human

---

## 2. The 3-Way Merge Algorithm

### Why "3-Way"?

A 2-way merge compares only Alice's version to Bob's version directly. It cannot tell if a line was "both changed it" or "only Bob changed it (Alice kept the original)."

A **3-way merge** compares:
1. The **base** (common ancestor — version both started from)
2. **Ours** (our current branch)
3. **Theirs** (the branch being merged in)

With three versions, Git can reason:

```
LINE IN BASE:   "color: blue"
LINE IN OURS:   "color: red"       ← we changed it
LINE IN THEIRS: "color: blue"      ← they left it unchanged

3-way conclusion: Take OURS ("color: red")
                  We changed it, they didn't. No conflict.
```

```
LINE IN BASE:   "color: blue"
LINE IN OURS:   "color: blue"      ← we left it unchanged
LINE IN THEIRS: "color: green"     ← they changed it

3-way conclusion: Take THEIRS ("color: green")
                  They changed it, we didn't. No conflict.
```

```
LINE IN BASE:   "color: blue"
LINE IN OURS:   "color: red"       ← we changed it
LINE IN THEIRS: "color: green"     ← they also changed it (differently)

3-way conclusion: CONFLICT
                  Both changed the same line differently.
                  Git cannot decide. Human required.
```

```
LINE IN BASE:   "color: blue"
LINE IN OURS:   "color: red"       ← we changed it
LINE IN THEIRS: "color: red"       ← they changed it THE SAME WAY

3-way conclusion: Take "color: red" (auto-resolved)
                  Both made identical changes. No human needed.
```

### The Algorithm in Detail

```
INPUT:
  base_blob  = blob SHA of the file at merge base commit
  ours_blob  = blob SHA of the file on our branch (HEAD)
  theirs_blob = blob SHA of the file on their branch (MERGE_HEAD)

STEP 1: Diff base → ours   = set_of_changes_A
STEP 2: Diff base → theirs = set_of_changes_B

STEP 3: For each change (line range + new content):
  Case A: Change only in set_A → apply it (no conflict)
  Case B: Change only in set_B → apply it (no conflict)
  Case C: Change in both, identical content → apply it once (no conflict)
  Case D: Change in both, different content, OVERLAPPING line ranges → CONFLICT
  Case E: Change in both, different content, NON-OVERLAPPING → apply both (no conflict)

STEP 4: Assemble result:
  - Auto-resolved lines: written to working tree
  - Conflicted regions: written with conflict markers
  - Index stages 1/2/3: populated (see Section 4)

OUTPUT:
  Working tree file: contains conflict markers for conflicts
  Index stages:      1=base, 2=ours, 3=theirs for conflicted paths
  .git/MERGE_HEAD:   SHA of theirs (persists until resolved)
```

### A Concrete Example

```
BASE file (config.js):
  1: module.exports = {
  2:   host: 'localhost',
  3:   port: 3000,
  4:   debug: false,
  5: };

OURS file (config.js) — we changed port and added timeout:
  1: module.exports = {
  2:   host: 'localhost',
  3:   port: 8080,         ← changed from 3000
  4:   debug: false,
  5:   timeout: 30000,     ← added
  6: };

THEIRS file (config.js) — they changed port and added retries:
  1: module.exports = {
  2:   host: 'localhost',
  3:   port: 4000,         ← changed from 3000 (differently!)
  4:   debug: false,
  5:   retries: 3,         ← added
  6: };

3-WAY MERGE ANALYSIS:
  Line 2 (host):    unchanged in both → take it as-is
  Line 3 (port):    changed in BOTH to different values → CONFLICT
  Line 4 (debug):   unchanged in both → take it as-is
  Line 5 (timeout): added in OURS only → take it
  Line 5 (retries): added in THEIRS only → take it (non-overlapping addition)

RESULT in working tree:
  1: module.exports = {
  2:   host: 'localhost',
  3: <<<<<<< HEAD
  4:   port: 8080,
  5: =======
  6:   port: 4000,
  7: >>>>>>> feature/config
  8:   debug: false,
  9:   timeout: 30000,     ← auto-merged (our addition)
 10:   retries: 3,         ← auto-merged (their addition)
 11: };
```

The additions (timeout and retries) were auto-merged because they were in non-overlapping line positions. Only the port conflict requires human resolution.

---

## 3. Physical Architecture — `.git/` During a Conflict

### Before Merge (clean state)

```
.git/
├── HEAD                      → "ref: refs/heads/main"
├── refs/
│   └── heads/
│       ├── main              → "G-SHA"
│       └── feature/config    → "C-SHA"
├── index                     ← clean (stage 0 only — all files)
└── objects/
    └── ...
```

### During Conflict (merge started, conflict encountered)

```
.git/
├── HEAD                      → "ref: refs/heads/main"  ← UNCHANGED
│
├── MERGE_HEAD                → "C-SHA"   ← NEW: SHA of their branch tip
│                                            Presence of this file = merge in progress
│
├── MERGE_MSG                 → "Merge branch 'feature/config' into main"
│                                ← NEW: auto-generated merge commit message
│                                   You can edit this before completing the merge
│
├── ORIG_HEAD                 → "G-SHA"   ← NEW: safety backup of where main was
│                                            Before the merge, main was at G-SHA
│                                            Lets you undo: git reset --hard ORIG_HEAD
│
├── refs/
│   └── heads/
│       ├── main              → "G-SHA"   ← UNCHANGED (merge not committed yet)
│       └── feature/config    → "C-SHA"   ← UNCHANGED
│
├── index                     ← MODIFIED: contains stage 1/2/3 for conflicted paths
│                                          stage 0 for non-conflicted paths
│
└── objects/
    └── ...                   ← unchanged (no new commits yet)
```

### What MERGE_HEAD Tells You

The presence of `.git/MERGE_HEAD` is Git's indicator that you are in the middle of a merge:

```bash
# These commands check for MERGE_HEAD:
git status
# "You have unmerged paths."
# "Fix conflicts and run 'git commit'."

git merge --abort
# Reads MERGE_HEAD to know what to undo
# Then deletes MERGE_HEAD, restores HEAD to G-SHA, restores index

cat .git/MERGE_HEAD
# "a3f2c1d8..."  ← the SHA of feature/config's tip (commit C)
```

### After Successful Resolution and Commit

```
.git/
├── HEAD                      → "ref: refs/heads/main"
├── refs/
│   └── heads/
│       └── main              → "M-SHA"   ← UPDATED to merge commit
│
├── MERGE_HEAD                ← DELETED
├── MERGE_MSG                 ← DELETED
├── ORIG_HEAD                 → "G-SHA"   ← KEPT (useful for post-merge undo)
│
├── index                     ← CLEAN (all stage 0, no 1/2/3 entries)
│
└── objects/
    └── 8f/3a1b9...           ← NEW: merge commit M
```

---

## 4. The Index Stages — The Deepest Layer

This is the most important internals concept for understanding conflicts. The Git index (staging area) can hold **up to 4 versions** of each file simultaneously during a conflict.

### Stage Numbers

```
Stage 0: Normal (no conflict) — the current version of the file
Stage 1: BASE   — the version at the common ancestor (merge base)
Stage 2: OURS   — the version on HEAD (our branch)
Stage 3: THEIRS — the version on MERGE_HEAD (their branch)
```

For non-conflicted files: only stage 0 exists.
For conflicted files: stages 1, 2, 3 exist; stage 0 does NOT exist.

The absence of stage 0 is how Git tracks "this file has an unresolved conflict."

### Viewing the Index Stages

```bash
# Show all index entries including conflict stages:
git ls-files --stage

# Output for a file with conflict:
100644 a3f2c1d8... 1 config.js   ← stage 1: BASE version
100644 b4e3d2c7... 2 config.js   ← stage 2: OURS version (HEAD)
100644 c5f4e3b6... 3 config.js   ← stage 3: THEIRS version (MERGE_HEAD)

# Output for a clean file:
100644 d6e5f4a3... 0 server.js   ← stage 0: clean (no conflict)

# Show only conflicted files:
git ls-files --unmerged
```

### Reading Each Stage's Content

```bash
# Get the SHA for a specific stage:
git ls-files --stage config.js
# 100644 a3f2c1d8... 1 config.js

# Read the content of each stage:
git cat-file -p :1:config.js    # BASE version
git cat-file -p :2:config.js    # OURS version
git cat-file -p :3:config.js    # THEIRS version

# Or using show:
git show :1:config.js    # BASE
git show :2:config.js    # OURS
git show :3:config.js    # THEIRS
```

### The Stage Transition — Resolving a File

```
CONFLICTED STATE:
  index: { stage1: base-SHA, stage2: ours-SHA, stage3: theirs-SHA }
  working tree: config.js with conflict markers

AFTER RESOLUTION (git add config.js):
  Git computes: SHA of the resolved file content
  Git writes:   .git/objects/<resolved-SHA> (new blob)
  Git updates index: removes stages 1, 2, 3
                     adds stage 0 with resolved-SHA
  index: { stage0: resolved-SHA }
  working tree: config.js (no conflict markers — your edited version)

COMMIT:
  All files now at stage 0 → merge commit created
```

### Hands-On Proof — Seeing the Stages

```bash
# Create conflict scenario:
git init stage-demo && cd stage-demo
echo "port: 3000" > config.txt
git add . && git commit -m "base"

git checkout -b feature
echo "port: 8080" > config.txt
git add . && git commit -m "feature: port 8080"

git checkout main
echo "port: 4000" > config.txt
git add . && git commit -m "main: port 4000"

# Trigger conflict:
git merge feature
# "CONFLICT (content): Merge conflict in config.txt"

# PROVE: See all three stages in the index:
git ls-files --stage config.txt
# Should show entries for stages 1, 2, 3

# PROVE: Read each stage:
echo "=== BASE (stage 1) ===" && git show :1:config.txt
echo "=== OURS (stage 2) ===" && git show :2:config.txt
echo "=== THEIRS (stage 3) ===" && git show :3:config.txt

# PROVE: stage 0 does NOT exist for conflicted file:
git ls-files --stage config.txt | grep "^100644.*0 config"
# Should return empty (no stage 0)
```

---

## 5. Conflict Markers Decoded — Every Byte Explained

When Git writes conflict markers into a file, the format is precise. Every part has meaning.

### The Conflict Block Structure

```
<<<<<<< HEAD
  port: 8080,
=======
  port: 4000,
>>>>>>> feature/config
```

```
<<<<<<< HEAD
│       │
│       └─ The label for "ours" side.
│          Usually "HEAD" (the branch you were on when you ran git merge).
│          During rebase: the commit SHA being applied.
│          Configurable via merge.conflictstyle.
│
└─ 7 less-than signs. Always 7. This is a fixed marker.


=======
└─ 7 equals signs. The divider between ours and theirs.


>>>>>>> feature/config
│       │
│       └─ The label for "theirs" side.
│          Usually the branch name being merged.
│          During rebase: the branch being rebased onto.
│
└─ 7 greater-than signs. Always 7.
```

### What Is Inside the Markers

```
<<<<<<< HEAD
  (EVERYTHING HERE is the content from stage 2 — our HEAD version)
  (Exactly what git show :2:config.js would output for this region)
=======
  (EVERYTHING HERE is the content from stage 3 — their version)
  (Exactly what git show :3:config.js would output for this region)
>>>>>>> feature/config
```

The content outside the markers has already been auto-resolved by Git (applies to both, or only one side changed it).

### Multi-Hunk Conflicts

A file can have multiple separate conflict regions:

```javascript
// config.js with two conflict regions:

module.exports = {
<<<<<<< HEAD
  host: 'api.production.com',
=======
  host: 'api.staging.com',
>>>>>>> feature/env-config
  
  port: 3000,                    // ← auto-merged (neither changed)
  
<<<<<<< HEAD
  timeout: 30000,
  retries: 5,
=======
  timeout: 10000,
  retries: 3,
>>>>>>> feature/env-config
};
```

Each `<<<<<<<` ... `>>>>>>>` block is an independent conflict to resolve. They can be resolved independently. The order matters: you must resolve top to bottom (or use a merge tool that shows them all).

### How to Count Unresolved Conflicts

```bash
# Count conflict markers in a file:
grep -c "^<<<<<<< " config.js
# Output: 2  (two conflicts remaining)

# List all files with unresolved conflicts:
grep -rl "^<<<<<<< " .  --include="*.js"
# Or:
git diff --name-only --diff-filter=U
# U = Unmerged (files with unresolved conflicts)
```

### The 7-Character Marker — Why 7?

The original C code in Git uses 7 as the default marker length. It's arbitrary but fixed. You can change it:

```bash
git config merge.conflictStyle diff3  # adds ancestor section (Section 6)
# Marker size stays 7
```

There is no benefit to changing it. The 7-character convention is universal and all merge tools recognize it.

---

## 6. The `diff3` Conflict Style — Seeing the Ancestor

By default, conflict markers show only "ours" and "theirs." This is often insufficient to make a good decision — you need to see **where both sides started from** (the base).

### Enable `diff3`

```bash
# Globally (recommended):
git config --global merge.conflictStyle diff3

# Or per-merge (not practical but possible):
git checkout -m config.js  # re-generate conflict markers
```

### What `diff3` Adds

```
Default style:
  <<<<<<< HEAD
    port: 8080,
  =======
    port: 4000,
  >>>>>>> feature/config

diff3 style:
  <<<<<<< HEAD
    port: 8080,
  ||||||| merged common ancestors
    port: 3000,
  =======
    port: 4000,
  >>>>>>> feature/config
  │
  └─ NEW SECTION: shows the BASE (stage 1) version
     "merged common ancestors" = the content at the merge base commit
```

### Why `diff3` Makes Conflicts Easier to Resolve

**Without diff3:**
- You see: we have `port: 8080`, they have `port: 4000`
- You think: "Which is right? I don't know what the original was."
- You must run `git show :1:config.js` to find the ancestor value

**With diff3:**
- You see: originally `port: 3000`, we changed to `8080`, they changed to `4000`
- You can reason: "We explicitly chose 8080 for production. They chose 4000 for their environment. Correct resolution is probably environment-specific. Talk to the team."
- The ancestor context makes the conflict REASON visible immediately.

### Real-World Example Where `diff3` Is Critical

```
DEFAULT STYLE (confusing):
  <<<<<<< HEAD
    if (user.role === 'admin' || user.role === 'superadmin') {
  =======
    if (user.role === 'admin') {
  >>>>>>> feature/simplify-auth

  Question: Which is correct? Did we ADD superadmin, or did they REMOVE it?

DIFF3 STYLE (clear):
  <<<<<<< HEAD
    if (user.role === 'admin' || user.role === 'superadmin') {
  ||||||| merged common ancestors
    if (user.role === 'admin') {
  =======
    if (user.role === 'admin') {
  >>>>>>> feature/simplify-auth

  Answer: The base is "admin" only.
          We ADDED superadmin (a feature we're building).
          They kept admin only (no change on their side).
          Correct resolution: TAKE OURS — keep the superadmin role.
          (They simply didn't touch this line; "theirs" = no change = base)

  Wait — but the base equals theirs! That means Git should have auto-merged this
  in our favor automatically. Why is there a conflict?

  → Because in the FULL file context, the function signature or surrounding
    lines changed in THEIRS, shifting line numbers, causing Git to see this
    region as "modified in both." diff3 makes this visible.
```

---

## 7. `ours` vs `theirs` — The Most Confusing Terms in Git

This is the most frequent source of confusion, especially because "ours" and "theirs" **swap meaning during rebase**.

### During `git merge`

```
git checkout main
git merge feature/login

"OURS"   = main     = HEAD         = stage 2 in index
"THEIRS" = feature  = MERGE_HEAD   = stage 3 in index

Intuition: "We" are on main (our branch), "they" are bringing in the feature.
```

### During `git rebase`

```
git checkout feature/login
git rebase main

"OURS"   = main           = the branch we're rebasing ONTO
"THEIRS" = feature/login  = the commits being replayed (our feature!)

Intuition: REVERSED from merge!
           During rebase, each commit from "our" feature branch is replayed
           ONTO "their" target branch (main).
           "Ours" = the base (main) we're rebasing onto.
           "Theirs" = our own feature commit being replayed.
```

**THIS IS COUNTERINTUITIVE.** During a rebase, your own code is "theirs."

```
MEMORY AID for rebase:
  "Ours" = what was there BEFORE the commit being applied
          = the growing history we're building onto
  "Theirs" = the INCOMING commit (your feature commit)

Concrete: You're on feature/login, running git rebase main.
  When conflict appears in commit B:
    "ours"   = main + commit A' (what's been built so far during rebase)
    "theirs" = commit B from feature/login (currently being applied)
  
  git checkout --ours config.js  = take main's version + prior rebase progress
  git checkout --theirs config.js = take your own feature/login's commit B version
```

### The Commands and What They Do

```bash
# During a conflict (merge OR rebase):

git checkout --ours <file>
  # Writes the stage 2 content to working tree
  # During merge: takes main's (HEAD's) version
  # During rebase: takes the ongoing-rebase branch's version (confusing!)

git checkout --theirs <file>
  # Writes the stage 3 content to working tree
  # During merge: takes the feature branch's version
  # During rebase: takes YOUR feature commit's version (what you wrote!)

# After checkout --ours or --theirs, you must still:
git add <file>  # stage the resolution (moves from 1/2/3 to stage 0)
```

### Side-by-Side Comparison

```
SCENARIO: Conflict in config.js
          OURS has: port: 8080
          THEIRS has: port: 4000

DURING git merge feature:
  git checkout --ours config.js    → config.js now says "port: 8080"  (main's version)
  git checkout --theirs config.js  → config.js now says "port: 4000"  (feature's version)

DURING git rebase main (from feature branch):
  git checkout --ours config.js    → config.js now says "port: 4000"  (main's version!)
  git checkout --theirs config.js  → config.js now says "port: 8080"  (your feature!)
  
  ← Note: the mapping is the OPPOSITE of merge!
```

### Use `git show :N:file` to Be Certain

When you're not sure which is which, skip `--ours`/`--theirs` entirely and use explicit stage access:

```bash
# Always unambiguous:
git show :1:config.js    # BASE (common ancestor) — always the ancestor
git show :2:config.js    # STAGE 2 — always HEAD's version at merge time
git show :3:config.js    # STAGE 3 — always MERGE_HEAD / incoming commit version

# To accept stage 2 explicitly:
git show :2:config.js > config.js
git add config.js

# To accept stage 3 explicitly:
git show :3:config.js > config.js
git add config.js
```

This approach is 100% unambiguous, even during rebase.

---

## 8. Resolving Conflicts — The 4 Approaches

### Approach 1: Manual Edit (Most Common)

Open the conflicted file in your editor. Find the markers. Decide what the resolved version should look like. Delete the markers and everything you don't want.

```bash
# Step 1: Find conflicted files
git status
# "both modified: config.js"
# "both modified: server.js"

# Step 2: Edit config.js — remove markers, write correct content
# Before (conflict markers):
# <<<<<<< HEAD
#   port: 8080,
# =======
#   port: 4000,
# >>>>>>> feature/config

# After (your decision: 8080 for production):
#   port: 8080,

# Step 3: Stage the resolved file
git add config.js

# Step 4: Verify no markers remain
git diff --staged config.js
# Should show no <<< === >>> characters

# Step 5: If all files resolved, commit
git status  # "All conflicts fixed but you are still merging."
git commit  # uses the pre-written MERGE_MSG as commit message
```

### Approach 2: Accept One Side Entirely

When you know that one entire side is correct for a file:

```bash
# Accept our version of the entire file (stage 2):
git checkout --ours config.js
git add config.js

# Accept their version of the entire file (stage 3):
git checkout --theirs config.js
git add config.js

# Accept ours for ALL conflicted files (nuclear option):
git checkout --ours .
git add .
# USE WITH CARE — reviews all conflicts automatically in your favor
```

### Approach 3: Use `git mergetool`

Launch a configured merge tool that shows all three versions (base, ours, theirs) side-by-side and lets you pick:

```bash
git mergetool
# Opens your configured tool for each conflicted file
# After saving the resolved version in the tool, the file is auto-staged

# Configure a specific tool:
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

git config --global merge.tool vimdiff
# (no cmd needed — vimdiff is built-in)

git config --global merge.tool kdiff3
git config --global mergetool.kdiff3.cmd 'kdiff3 $BASE $LOCAL $REMOTE -o $MERGED'
```

### Approach 4: Abort and Re-Think

Sometimes the right answer is to not resolve the conflict right now:

```bash
# Abort entirely — go back to pre-merge state:
git merge --abort
# READS:  .git/ORIG_HEAD (where main was before merge)
# RESETS: HEAD back to ORIG_HEAD
# CLEANS: index (removes stages 1/2/3)
# REMOVES: MERGE_HEAD, MERGE_MSG
# RESULT:  Clean state, as if git merge was never run

# When to abort:
# - The conflict is complex and you need to understand the feature branch better
# - You realize the merge strategy is wrong (should rebase instead)
# - You need input from the author of the conflicting code
```

---

## 9. Merge Tools — vimdiff, VS Code, kdiff3

### VS Code as Merge Tool

VS Code has a built-in merge editor that highlights conflicts and lets you click "Accept Current" / "Accept Incoming" / "Accept Both":

```bash
# Configure VS Code as default merge tool:
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# The $MERGED variable is filled with the conflicted file path by Git.
# --wait makes git wait until VS Code closes the file before moving to next conflict.
```

VS Code conflict UI:

```
┌─────────────────────────────────────────────────────────────────┐
│  Current Change (HEAD / ours)        │  Incoming Change (theirs) │
│  ─────────────────────────────────   │  ──────────────────────── │
│    port: 8080,                        │    port: 4000,             │
│                                       │                            │
│  [Accept Current] [Accept Incoming] [Accept Both] [Compare]       │
└─────────────────────────────────────────────────────────────────┘
```

"Accept Both" inserts BOTH lines — useful when both additions are needed.

### vimdiff as Merge Tool (Terminal)

```bash
git config --global merge.tool vimdiff
git config --global merge.conflictstyle diff3  # recommended with vimdiff
git mergetool
```

vimdiff opens in a 4-panel layout:
```
┌──────────┬──────────┬──────────┐
│  LOCAL   │  BASE    │  REMOTE  │
│ (ours)   │(ancestor)│ (theirs) │
├──────────┴──────────┴──────────┤
│          MERGED                 │
│  (this is what you edit)        │
└─────────────────────────────────┘
```

vimdiff keybindings for merge:
```
:diffget LOCAL   or  :diffget //2   → take ours (LOCAL) for current conflict
:diffget REMOTE  or  :diffget //3   → take theirs (REMOTE) for current conflict
:diffget BASE    or  :diffget //1   → take base for current conflict
[c                                  → jump to previous conflict
]c                                  → jump to next conflict
:wqa                                → save and quit all panels
```

### kdiff3 (Graphical, Cross-Platform)

kdiff3 shows 3 input panes + 1 output pane with a manual chooser per conflict:

```
A: LOCAL (ours)          B: BASE (ancestor)      C: REMOTE (theirs)
────────────────         ────────────────        ────────────────
  port: 8080               port: 3000              port: 4000

Output:
  [A] [B] [C] buttons → click to pick the version
  Or type directly in output pane for custom resolution
```

### Checking for Leftover Conflict Markers After Resolution

```bash
# ALWAYS run this before committing:
git diff --check
# Output: "config.js:3: leftover conflict marker"
# Git refuses to commit if conflict markers remain, but this shows them explicitly.

# Also:
grep -rn "^<<<<<<< \|^=======$\|^>>>>>>> " --include="*.js" .
# Finds any leftover markers in your source files
```

---

## 10. Complex Conflict Types — Beyond Text Conflicts

### Type 1: Delete/Modify Conflict

One side deleted a file, the other modified it.

```bash
# Scenario:
# main: deleted config.js (decided to use env vars instead)
# feature: modified config.js (added new settings)

git merge feature
# "CONFLICT (modify/delete): config.js deleted in HEAD and modified in feature"
# "Use 'git rm config.js' to remove or 'git add config.js' to keep"

git status
# "deleted by us: config.js"

# Decision options:
# Option A: Keep the file (take their modification):
git checkout --theirs config.js  # restore the modified version
git add config.js

# Option B: Delete the file (keep our deletion):
git rm config.js
# Both options require: git commit
```

**How to reason about it**: Did your team decide to DELETE the file? Then `git rm`. Did the feature add important settings that still need to be somewhere? Then `git checkout --theirs` and integrate the new settings into whatever replaced config.js.

### Type 2: Rename/Rename Conflict

Both sides renamed the same file to different names.

```bash
# main: renamed auth.js → authentication.js
# feature: renamed auth.js → login.js

git merge feature
# "CONFLICT (rename/rename): auth.js renamed to authentication.js in HEAD
#  and to login.js in feature"

git status
# "both modified: auth.js" (shown as the original name)

# The index contains:
# auth.js with stages 1/2/3 but also
# authentication.js at stage 2 and login.js at stage 3

# Decision: pick one name and consolidate
git mv authentication.js auth-module.js  # or just keep authentication.js
git rm login.js                           # remove the other
git add .
git commit
```

### Type 3: Rename/Modify Conflict

One side renamed a file, the other modified it.

```bash
# main: renamed handler.js → request-handler.js
# feature: modified handler.js (added new method)

git merge feature
# Git is usually smart enough to track the rename and apply the modification to
# the renamed file. But if both renamed AND modified the content:

git status
# "both modified: handler.js"
# You need to manually ensure request-handler.js has the new method
```

### Type 4: Binary File Conflict

Text diff is meaningless for binary files (images, compiled files, etc.)

```bash
git merge feature
# "CONFLICT (content): Binary files logo.png differ"

git status
# "both modified: logo.png" (binary)

# Binary files cannot have conflict markers — you must choose entirely one side:
git checkout --ours logo.png    # keep our version
git add logo.png
# OR:
git checkout --theirs logo.png  # keep their version
git add logo.png
```

### Type 5: Submodule Conflict

```bash
git merge feature
# "CONFLICT (submodule): Conflict in vendor/library"

git status
# "both modified: vendor/library (submodule)"

# Git doesn't know which submodule commit is correct
# You must decide manually:
git ls-files --stage vendor/library
# Shows the three SHA entries (but they're submodule commit SHAs, not blob SHAs)

# Accept ours:
git checkout --ours vendor/library
git add vendor/library

# Accept theirs:
git checkout --theirs vendor/library
git add vendor/library
```

### Type 6: The "Both Added" Conflict

Both sides added a file with the same name but different content. There was no ancestor version.

```bash
# main: added test/auth.test.js (with test suite A)
# feature: also added test/auth.test.js (with test suite B)

git merge feature
# "CONFLICT (add/add): Merge conflict in test/auth.test.js"

git ls-files --stage test/auth.test.js
# stage 2 has our version, stage 3 has their version
# stage 1 does NOT exist (no ancestor — file didn't exist)

# Resolution: manually combine both test suites (usually the right answer)
# Or take one side and incorporate the other's tests later
```

---

## 11. Rebase Conflicts vs Merge Conflicts

Rebase conflicts feel similar but behave differently in important ways.

### How Rebase Conflicts Work

During `git rebase`, each commit from your branch is applied **one at a time** to the target. Each commit can independently conflict.

```
REBASE: git rebase main  (from feature/login with commits A, B, C)

Step 1: Apply commit A onto main (HEAD=G)
  → If A conflicts with G's changes → CONFLICT
  → You resolve → git add → git rebase --continue
  
Step 2: Apply commit B (onto the just-resolved A' state)
  → If B conflicts → CONFLICT again
  → You resolve → git add → git rebase --continue

Step 3: Apply commit C
  → No conflict → auto-applied

Done: feature/login = A'--B'--C' on top of main
```

### Key Differences from Merge Conflicts

| | Merge Conflict | Rebase Conflict |
|--|--|--|
| How many at once | All conflicts in one merge | One commit at a time |
| `ours` label | HEAD (your branch) | The growing rebase result |
| `theirs` label | Incoming branch | Your own commit being applied |
| Resume command | `git commit` | `git rebase --continue` |
| Cancel command | `git merge --abort` | `git rebase --abort` |
| State file | `.git/MERGE_HEAD` | `.git/rebase-merge/` directory |
| Skip a commit | Not applicable | `git rebase --skip` |

### The `.git/rebase-merge/` Directory

```
.git/rebase-merge/
├── head-name        → "refs/heads/feature/login"  (branch being rebased)
├── onto             → "G-SHA"  (what we're rebasing onto)
├── orig-head        → "C-SHA"  (original tip before rebase)
├── msgnum           → "2"  (which commit we're currently on — B)
├── end              → "3"  (total commits to apply)
├── stopped-sha      → "B-SHA"  (SHA of commit that caused conflict)
├── message          → "B: add session handling"  (commit message of stopped commit)
└── git-rebase-todo  → remaining commits to apply
    ├── "pick C-SHA C: add logout"  ← still to do
    └── (A is done, B is current, C is remaining)
```

```bash
# See rebase state during conflict:
cat .git/rebase-merge/msgnum   # "2" = currently resolving 2nd commit
cat .git/rebase-merge/end      # "3" = 3 total commits in rebase
cat .git/rebase-merge/stopped-sha  # SHA of the commit that conflicted

# After resolving all conflicts in the current commit:
git add <resolved-files>
git rebase --continue    # applies the commit, moves to next

# Skip the entire current commit (rare — use if commit is no longer needed):
git rebase --skip
# WARNING: the commit is dropped from history

# Abort the entire rebase:
git rebase --abort
# Returns feature/login to its original state (C-SHA)
```

### When the Same Conflict Appears in Multiple Commits

```
SCENARIO: Commit A and commit B both modified config.js.
          Main also modified config.js.
          
Rebase applies A first:
  → Conflict with main's config.js changes
  → You resolve: port: 8080

Rebase applies B next:
  → Conflict AGAIN with the (now partially resolved) config.js
  → You resolve again: timeout: 30000

This is where rerere (Topic 17) shines:
  rerere recorded your first resolution.
  rerere auto-resolves the same conflict in B.
```

---

## 12. Cherry-Pick Conflicts

`git cherry-pick` also creates conflicts, using the same mechanism as rebase.

```bash
git cherry-pick <commit-SHA>
# If conflict:
# "CONFLICT (content): Merge conflict in auth.js"
# "error: could not apply <SHA>..."
# "hint: After resolving the conflicts, mark them with..."
# "hint: git add/rm <pathspec>..."
# "hint: and run 'git cherry-pick --continue'"

# State in .git/:
cat .git/CHERRY_PICK_HEAD   # SHA of the commit being cherry-picked
# (similar to MERGE_HEAD but for cherry-pick)

# Resume:
git add <resolved>
git cherry-pick --continue

# Abort:
git cherry-pick --abort

# ours/theirs during cherry-pick:
# "ours" = current branch (HEAD)
# "theirs" = the commit being cherry-picked
# Same orientation as merge (not the flipped rebase orientation)
```

---

## 13. Production Scenarios — Real Conflict Situations

### Scenario 1: Long-Running Feature Branch Rebase

**Context**: You've been working on `feature/payment-refactor` for 3 weeks. It's Friday. You finally need to merge, but `main` has had 40+ commits since you branched. Your branch has 8 commits.

```bash
# Step 1: Fetch latest
git fetch origin

# Step 2: See how far behind you are:
git log --oneline HEAD..origin/main | wc -l
# Output: 47 (47 commits on main since you branched)

git log --oneline origin/main..HEAD | wc -l
# Output: 8 (your 8 feature commits)

# Step 3: Enable rerere (if not already):
git config rerere.enabled true

# Step 4: Rebase (expect conflicts):
git rebase origin/main

# Step 5: First conflict appears:
# CONFLICT (content): Merge conflict in src/payments/processor.js

# Step 6: Use diff3 style to understand:
git diff  # shows conflict markers (diff3 style if configured)

# Step 7: Understand what happened:
git show :1:src/payments/processor.js  # what was there originally
git show :2:src/payments/processor.js  # what main has now
git show :3:src/payments/processor.js  # what this commit tried to do

# Which commit is this?
cat .git/rebase-merge/stopped-sha
git show $(cat .git/rebase-merge/stopped-sha) --stat
# Shows: "Commit 3 of 8: refactor: extract payment validation"

# Step 8: Resolve manually — edit file, then:
git add src/payments/processor.js
git rebase --continue

# Repeat for each conflicting commit.
# rerere remembers your resolutions across commits.

# Step 9: After rebase completes:
git log --oneline origin/main..HEAD  # your 8 commits, now rebased as A'..H'

# Step 10: Push (force-with-lease — safe force push):
git push --force-with-lease origin feature/payment-refactor
```

### Scenario 2: Hotfix Merged to Both `main` and `develop` in Gitflow

**Context**: You're using Gitflow. A hotfix was merged to `main` and tagged `v1.2.1`. You now need to merge it into `develop`. But `develop` has diverged significantly.

```bash
git checkout develop
git merge main  # merge the hotfix into develop

# Common conflict: version numbers
# main has version "1.2.1" (from the hotfix)
# develop has version "1.3.0-dev" (already set for next release)

# diff3 style conflict:
# <<<<<<< HEAD
#   "version": "1.3.0-dev",
# ||||||| merged common ancestors
#   "version": "1.2.0",
# =======
#   "version": "1.2.1",
# >>>>>>> main

# Clear resolution with diff3:
# Ancestor was 1.2.0.
# main bumped to 1.2.1 (hotfix).
# develop set to 1.3.0-dev (next release).
# Correct answer: keep develop's "1.3.0-dev" — that's intentional.

git checkout --ours package.json  # keep develop's "1.3.0-dev"
git add package.json

# Other conflicts may be legitimate: hotfix changed code that develop also changed.
# Resolve each carefully using diff3 to see what the hotfix actually did.
git commit  # "Merge hotfix v1.2.1 into develop"
```

### Scenario 3: Two Teams Conflict on the Same Service

**Context**: The auth team and the API team both modified `src/middleware/index.js`. Two PRs land at almost the same time. PR A merges first. PR B now has a conflict.

```bash
# PR B developer's perspective:
git fetch origin
git rebase origin/main  # or merge origin/main

# CONFLICT in src/middleware/index.js

# Understanding the conflict:
git log --oneline origin/main..HEAD  # your commits
git log --oneline HEAD..origin/main  # what landed while you were working

# Find which PR caused the conflict:
git log --oneline origin/main -10
# "8f3a1b9 feat(auth): add JWT validation middleware (#127)"
# ← This is PR A, the auth team's change

# Now look at both:
git show :1:src/middleware/index.js  # original
git show :2:src/middleware/index.js  # your changes (API team)  
git show :3:src/middleware/index.js  # what main has (auth team's PR)

# Best approach: contact the auth team
# "Hey, I'm resolving a conflict between my PR and yours (#127).
#  Can you review my resolution before I push?"

# Resolve, push, both teams verify the resolution in the updated PR
```

### Scenario 4: Resolving a Conflict in a Generated File

**Context**: Both your branch and main modified `package-lock.json`. This is a 10,000-line generated file.

```bash
git merge feature
# CONFLICT (content): Merge conflict in package-lock.json

# DON'T try to manually resolve a lockfile conflict.
# The right approach: regenerate it.

# Option A: Take ours and regenerate
git checkout --ours package.json      # take our package.json (human-authored)
git add package.json
# Then handle lockfile:
rm package-lock.json
npm install                            # regenerates a clean lockfile
git add package-lock.json
git commit

# Option B: Take theirs for package.json, then regenerate
git checkout --theirs package.json
git add package.json
rm package-lock.json
npm install
git add package-lock.json
git commit

# The lockfile must always be GENERATED, never hand-merged.
# Same applies to: yarn.lock, Gemfile.lock, Pipfile.lock, go.sum, etc.
```

---

## 14. Mistakes → Root Cause → Fix → Prevention

### Mistake 1: Committing Conflict Markers

**Symptom**: CI fails with a syntax error on a line containing `<<<<<<< HEAD`.

**Root cause**: Developer ran `git add .` after editing only some conflict files, without verifying the others. Or used `--no-verify` to skip the pre-commit hook.

```bash
# Diagnose:
git show HEAD | grep "^<<<<<<< \|^=======\|^>>>>>>> " --color
# Finds committed conflict markers

# Fix:
git log --oneline -3
# Find the bad commit SHA

git revert <bad-commit-SHA>
# Creates a revert commit that removes the markers
# OR if not yet pushed:
git reset HEAD~1  # undo the commit
# Fix the file, re-add, re-commit
```

**Prevention**:

```bash
# Method 1: git diff --check (built-in):
# Add this to your pre-commit hook:
# .git/hooks/pre-commit:
git diff --check --cached
# Returns non-zero if conflict markers found in staged files

# Method 2: Configure in .gitattributes:
# *.js diff  # ensures diff is applied and markers would be caught

# Method 3: GitHub runs conflict detection and blocks merging if markers found
```

### Mistake 2: Using `--ours` During Rebase When You Meant Your Feature Code

**Symptom**: After rebase, your feature changes are gone. `main`'s version is everywhere.

**Root cause**: During `git rebase main`, ran `git checkout --ours` thinking "ours = my feature." But during rebase, `--ours` = the base (main), not your feature commits.

**Fix**:

```bash
# Check what happened:
git log --oneline -5    # see commits after rebase
git show HEAD           # does it have your changes?

# If your changes are missing:
git reflog
# Find the SHA of your feature/login BEFORE the rebase
# e.g., "feature/login@{1}" = before the rebase

git rebase --abort      # if still in progress
# OR if rebase is done:
git reset --hard feature/login@{1}  # restore from reflog
# Then rebase again more carefully
```

**Prevention**: Use `git show :2:file` and `git show :3:file` explicitly instead of `--ours`/`--theirs` during rebase, until you've internalized the orientation.

### Mistake 3: Resolving a Conflict and Then Forgetting `git add`

**Symptom**: `git rebase --continue` gives: "No changes — did you forget to use 'git add'?"

**Root cause**: You edited the file to remove conflict markers but forgot to stage it. `git rebase --continue` checks for staged changes, not working tree changes.

```bash
# Diagnose:
git status
# "modified:  config.js" (but NOT "Changes to be committed")

# Fix:
git add config.js
git rebase --continue  # now it works
```

**Prevention**: After editing a conflicted file, immediately run `git add <file>` before moving to the next file. Use `git status` obsessively to verify staging state.

### Mistake 4: Accidentally Accepting All of "Ours" or "Theirs" on a Critical File

**Symptom**: After `git checkout --ours .` (accepted all ours), the teammate's important feature is gone.

**Fix**:

```bash
# The stages are still in the index if you haven't committed:
git ls-files --stage  # verify stages 1/2/3 still exist

# Restore the theirs version of the specific file:
git show :3:critical-file.js > critical-file.js
git add critical-file.js

# Now manually merge the two versions
```

**Prevention**: Never use `git checkout --ours .` (with a dot) on the whole directory. Always resolve file-by-file. Reserve `--ours .` for situations like "this entire directory is machine-generated — always take ours."

### Mistake 5: Re-Introducing Reverted Code During Merge

**Symptom**: Code that was explicitly reverted last week reappears after a merge.

**Root cause**: The revert commit is in `main`. The feature branch was branched from BEFORE the revert. The feature's copy of the reverted code is then treated as "new" during 3-way merge.

```
Timeline:
  main: ... → bad-feature → revert-bad-feature → ...
                                 ↑
  feature/new: branched from here (before revert)
  feature/new: happens to include the same pattern as bad-feature

  When feature/new merges: the revert undoes its effect for other code,
  but feature/new's copy of that code is treated as their new addition.
```

**Fix and Prevention**:

```bash
# After every merge, audit the diff carefully:
git diff main..HEAD  # what is your branch adding vs main?
# Look for anything that was explicitly reverted

# Or review the merge commit diff:
git show HEAD  # look for anything surprising

# If you find re-introduced reverted code:
git log --oneline --all -- <problematic-file>  # full history of the file
git show <revert-commit-sha> -- <problematic-file>  # what was reverted and why
# Then decide: is the reintroduction intentional (new implementation) or accidental?
```

---

## 15. Byte-Level Internals — The Index in Conflict State

### Index File Format During Conflict

The Git index (`.git/index`) is a binary file. Here's its structure conceptually (the format is documented in the Git source as `Documentation/technical/index-format.txt`):

```
.git/index binary layout:

[4 bytes] "DIRC"                  ← magic header "Directory Cache"
[4 bytes] version number          ← 2 or 3 or 4
[4 bytes] number of entries

For EACH entry:
  [4 bytes] ctime seconds
  [4 bytes] ctime nanoseconds
  [4 bytes] mtime seconds
  [4 bytes] mtime nanoseconds
  [4 bytes] dev
  [4 bytes] ino
  [4 bytes] mode (e.g., 0100644 = regular file, read/write for owner)
  [4 bytes] uid
  [4 bytes] gid
  [4 bytes] file size
  [20 bytes] SHA-1 of the blob    ← the object this entry points to
  [2 bytes] flags:
    bits 0-11: name length (if < 0xFFF) OR 0xFFF if longer
    bits 12-13: stage number (0, 1, 2, or 3)  ← THE CONFLICT STAGE
    bit 14: assume-valid flag
    bit 15: extended flag
  [variable] filename (null-terminated)
  [padding] to 8-byte boundary

[Extensions] (optional, e.g., TREE cache, rerere)
[20 bytes] SHA-1 of entire index  ← checksum
```

The **stage number** (bits 12-13 of the flags field) is what makes conflict tracking possible. These are the stage numbers we've been discussing:

```
00 (binary) = 0 → normal entry (no conflict)
01 (binary) = 1 → conflict: base version
10 (binary) = 2 → conflict: ours (HEAD)
11 (binary) = 3 → conflict: theirs (MERGE_HEAD)
```

### Reading the Raw Index

```bash
# Git provides a plumbing command to read index entries:
git ls-files --stage --debug config.js

# During conflict output:
100644 a3f2c1d8... 1 config.js       ← stage 1, base SHA
  ctime: ...
  mtime: ...
  dev: ...
100644 b4e3d2c7... 2 config.js       ← stage 2, ours SHA
  ctime: ...
  ...
100644 c5f4e3b6... 3 config.js       ← stage 3, theirs SHA
  ctime: ...
  ...

# You can see the actual SHA of each version's blob:
git cat-file -p a3f2c1d8  # prints the BASE version's file content
git cat-file -p b4e3d2c7  # prints OUR version
git cat-file -p c5f4e3b6  # prints THEIR version
```

### What `git add` Does to the Index During Conflict Resolution

```
BEFORE git add config.js (conflict state):
  index entries for config.js:
    [stage=1, SHA=base-SHA,  name=config.js]
    [stage=2, SHA=ours-SHA,  name=config.js]
    [stage=3, SHA=theirs-SHA, name=config.js]

DURING git add config.js:
  Git reads working tree config.js (your resolved version)
  Git computes: resolved-SHA = SHA-1("blob " + size + "\0" + resolved-content)
  Git writes:   .git/objects/resolved-SHA (new blob)
  Git removes:  all stage 1/2/3 entries for config.js from index
  Git adds:     [stage=0, SHA=resolved-SHA, name=config.js]

AFTER git add config.js (clean state):
  index entries for config.js:
    [stage=0, SHA=resolved-SHA, name=config.js]
```

This is why "git add" is the signal that "this conflict is resolved." The transition from stages 1/2/3 → stage 0 is what Git tracks as "resolved."

---

## 16. Mental Model Checkpoint

Test yourself from memory.

**1.** A merge conflict appears in `server.js`. You run `git ls-files --stage server.js` and see entries for stages 1, 2, and 3. What does stage 1 contain? What does the ABSENCE of stage 0 tell Git?

**2.** You are on `feature/login` and run `git rebase main`. A conflict appears. You run `git checkout --ours server.js`. Whose version do you get — your feature's version or main's version? Why?

**3.** What is the `diff3` conflict style and what section does it add? Give an example of a case where it makes the correct resolution obvious, but the default style would leave you guessing.

**4.** You're resolving a conflict and run `git add config.js`, then `git rebase --continue`. The rebase says "No changes — did you forget to use git add?" What happened?

**5.** List the four files/directories Git creates in `.git/` when a merge is in progress (not yet committed). What is the purpose of each?

**6.** A colleague deleted `utils.js` in their branch. You modified `utils.js` in yours. What type of conflict is this, and what are the two resolution options? What command implements each?

**7.** After resolving all conflicts and running `git commit`, you notice `git log --oneline main` shows the commit but `cat .git/ORIG_HEAD` still contains the old main SHA. Is this a problem? What is `ORIG_HEAD` for?

**8.** You run `grep -r "<<<<<<< " src/` and find 3 files with conflict markers. You already ran `git add .` and `git commit`. What do you do?

**9.** During `git merge`, you resolve a conflict and run `git add server.js`. But you realize you want to look at the base version again. Is it still available? How?

**10.** A file has TWO separate conflict regions (two `<<<<<<<...>>>>>>>` blocks). You manually resolve only the first and run `git add`. What happens?

<details>
<summary>Answers</summary>

**1.** Stage 1 contains the BASE version — the content of `server.js` at the common ancestor (merge base) commit. The absence of stage 0 tells Git that this file has an unresolved conflict. Git tracks conflict state by the presence of stages 1/2/3 without a stage 0. When you run `git commit`, Git checks for any files with non-zero stages — if found, it aborts the commit.

**2.** You get main's version. During `git rebase main`, "ours" = the target branch (main) you're rebasing onto, not your own feature. "Theirs" = your feature commit being applied. This is the opposite of `git merge` where "ours" = your branch. Use `git show :2:server.js` (stage 2) and `git show :3:server.js` (stage 3) for unambiguous identification during rebase.

**3.** `diff3` adds a `||||||| merged common ancestors` section showing the BASE (ancestor) version between the `<<<<<<<` (ours) and `=======` divider. Example: ours has `role: 'admin' || 'superadmin'`, default shows this vs theirs having `role: 'admin'`. Without diff3, you don't know if you ADDED superadmin or they REMOVED it. With diff3, if base is `role: 'admin'`, you can see that you added superadmin (theirs = base, so they didn't change it) — correct resolution: keep ours.

**4.** During `git rebase`, when a conflict occurs, Git leaves the conflicted state with stages 1/2/3. When you ran `git add config.js`, it may have been from a previous file resolution that you already committed — or the file you edited was NOT the file with the conflict. `git rebase --continue` checks for staged changes (index different from last applied commit). If you added a wrong file or nothing is staged, it fails. Check `git status` to see what is staged vs what still has conflict stages.

**5.** `MERGE_HEAD` — SHA of the branch being merged in (their commit); the presence of this file signals "merge in progress." `MERGE_MSG` — auto-generated commit message for the merge commit (you can edit this). `ORIG_HEAD` — the SHA of where HEAD (main) was before the merge started, providing a way to undo with `git reset --hard ORIG_HEAD`. `MERGE_MODE` — (optional, for special merge modes like `--no-ff` enforced). The index also has changes (stages 1/2/3) but is not a new file.

**6.** This is a delete/modify conflict (or "modify/delete" depending on which side deleted). Two options: (A) Keep the file — `git checkout --theirs utils.js && git add utils.js` (restores their modified version, though "theirs" in this context means the side that modified, not deleted). Actually more precisely: `git checkout HEAD -- utils.js && git add utils.js` to keep your modifications, or `git rm utils.js` to honor the deletion. The Git output message says "Use 'git rm' to remove or 'git add' to keep."

**7.** No, it is not a problem. `ORIG_HEAD` is kept after the merge completes — it is a safety backup in case you want to undo the merge after the fact with `git reset --hard ORIG_HEAD`. It stays until overwritten by the next operation that creates an ORIG_HEAD (another merge, rebase, etc.). It does not indicate an error.

**8.** You've committed conflict markers — a bug. The code will likely not compile/parse. Fix: `git revert HEAD` (creates a new commit reverting the bad commit) or `git reset HEAD~1` (if not yet pushed, undo the commit), fix the files, re-stage, re-commit. Prevention: run `git diff --check --cached` before committing to detect leftover markers.

**9.** Yes — but ONLY if you haven't committed yet. Once you run `git add`, the stages 1/2/3 are replaced with stage 0. However, the blob objects (base, ours, theirs) still exist in `.git/objects/` as dangling objects until garbage collection. If you saved their SHAs earlier (`git ls-files --stage`), you can still `git cat-file -p <SHA>` to read them. If you didn't save the SHAs, they're effectively gone from easy access (would need `git fsck --lost-found` to find dangling objects).

**10.** `git add` stages the ENTIRE file, not just one conflict region. If the file still contains conflict markers (the second `<<<<<<<...>>>>>>>` block), running `git add` stages a file with raw conflict markers in it. Git does NOT check for remaining markers at `git add` time. It will let you commit a file with conflict markers unless you have a pre-commit hook or run `git diff --check --cached`. Always verify with `grep "<<<<<<< " <file>` before running `git add`.

</details>

---

## 17. Connect Everything Backwards

| This Topic's Concept | Connects To |
|---------------------|-------------|
| **3-way merge uses a common ancestor** | Topic 09 — The merge base is computed by `git merge-base` (also used by `git diff main...feature`). The 3-way algorithm is the same algorithm described abstractly in Topic 09, now shown in full detail. |
| **Index stages 1/2/3** | Topic 05 — The index (staging area) is one of the "three trees." During conflicts it holds up to 3 versions simultaneously. The normal workflow uses only stage 0; conflict state activates stages 1-3. |
| **`MERGE_HEAD` file** | Topic 01 — `.git/` contains state files for in-progress operations. MERGE_HEAD is exactly like CHERRY_PICK_HEAD or REBASE_MERGE/ — a marker that an operation is in progress. |
| **`ORIG_HEAD` for recovery** | Topic 12 — `git reset --hard ORIG_HEAD` is the undo for a merge (before pushing), same mechanism as all the other resets. ORIG_HEAD was also written during rebase and reset operations. |
| **`ours`/`theirs` swap during rebase** | Topic 11 — Rebase replays your commits ONTO another branch. The "base" being built onto is "ours" in rebase terminology because it's what your commits are being planted into. This is why the orientation flips. |
| **`rerere` auto-resolves repeated conflicts** | Topic 17 — rerere was introduced in the merge strategies topic as the solution to the "rebase conflict snowball" problem. This topic shows the exact conflict-marker format that rerere records as a preimage. |
| **diff3 shows the merge base content** | Topic 09 — The merge base commit is the starting point of the 3-way algorithm. diff3 exposes this by showing stage 1 content inline in the conflict markers. |
| **`git checkout --ours/--theirs`** | Topic 05 — These commands write the content of a specific index stage to the working tree. Same as `git restore --source=:2 -- file` (the modern syntax). |
| **Binary file conflicts require full replacement** | Topic 02 — Binary files are stored as a single blob. There is no line-level diffing. The conflict is entirely "which blob do you want?" — stage 2 or stage 3. |
| **Generated file conflicts (lockfiles)** | Topic 06 — Blob content determines SHA. A regenerated lockfile will have a different SHA from either stage 2 or stage 3. It becomes a brand new blob that supersedes both conflict sides. |

---

## 18. Quick Reference Card

### Conflict State Commands

| Command | What it does |
|---------|-------------|
| `git status` | Lists conflicted files ("both modified", "deleted by us", etc.) |
| `git diff` | Shows conflict markers in working tree |
| `git diff --staged` | Shows what's staged so far (resolved files) |
| `git diff --check` | Reports leftover conflict markers (non-zero exit if found) |
| `git diff --check --cached` | Same but for staged files — use in pre-commit hook |
| `git ls-files --unmerged` | Lists all files with conflict stages |
| `git ls-files --stage <file>` | Shows stage 0/1/2/3 entries for a file |
| `git diff --cc HEAD` | Show combined diff of a merge commit |

### Reading the Three Versions

| Command | What it reads |
|---------|--------------|
| `git show :1:<file>` | BASE version (common ancestor — stage 1) |
| `git show :2:<file>` | OURS version (HEAD — stage 2) |
| `git show :3:<file>` | THEIRS version (incoming — stage 3) |
| `git cat-file -p :1:<file>` | Same as above (plumbing form) |
| `git diff :1:<file> :2:<file>` | Diff between base and ours |
| `git diff :1:<file> :3:<file>` | Diff between base and theirs |
| `git diff :2:<file> :3:<file>` | Diff between ours and theirs |

### Resolving Conflicts

| Command | What it does |
|---------|-------------|
| `git checkout --ours <file>` | Write stage 2 (HEAD) to working tree |
| `git checkout --theirs <file>` | Write stage 3 (MERGE_HEAD) to working tree |
| `git checkout --ours .` | Accept all of ours (entire working tree) |
| `git checkout --theirs .` | Accept all of theirs (entire working tree) |
| `git mergetool` | Open configured merge tool for each conflict |
| `git mergetool <file>` | Open merge tool for a specific file |
| `git add <file>` | Mark a file as resolved (stage 0) |
| `git rm <file>` | Resolve a delete/modify conflict by deleting |
| `git diff --check` | Verify no conflict markers remain |

### Continuing and Aborting

| Operation | Continue | Abort | Skip commit |
|-----------|----------|-------|-------------|
| `git merge` | `git commit` | `git merge --abort` | N/A |
| `git rebase` | `git rebase --continue` | `git rebase --abort` | `git rebase --skip` |
| `git cherry-pick` | `git cherry-pick --continue` | `git cherry-pick --abort` | `git cherry-pick --skip` |

### Configuration

| Config | Effect |
|--------|--------|
| `git config merge.conflictStyle diff3` | Show ancestor section in conflict markers |
| `git config merge.tool vscode` | Use VS Code as merge tool |
| `git config merge.tool vimdiff` | Use vimdiff as merge tool |
| `git config rerere.enabled true` | Enable conflict resolution recording |
| `git config rerere.autoupdate true` | Auto-stage rerere resolutions |
| `git config merge.ff no` | Always create merge commits (no fast-forward) |

### OURS vs THEIRS Orientation

| Operation | OURS (stage 2 / `--ours`) | THEIRS (stage 3 / `--theirs`) |
|-----------|--------------------------|-------------------------------|
| `git merge feature` | HEAD (your branch) | feature (incoming) |
| `git rebase main` | main (rebasing onto) | your feature commit (being applied) |
| `git cherry-pick <SHA>` | HEAD (your branch) | the cherry-picked commit |

---

## 19. When Would I Use This At Work?

### The Daily Conflict Workflow

**Every time you rebase or merge, have this mental checklist ready:**

```
1. BEFORE starting:
   git config --global merge.conflictStyle diff3  (set once, forget)
   git config --global rerere.enabled true        (set once, forget)

2. WHEN conflict appears:
   git status                   → which files are conflicted?
   git diff                     → see all conflict markers
   
3. FOR EACH conflicted file:
   git show :1:file.js          → understand: what was the original?
   git show :2:file.js          → understand: what did WE change?
   git show :3:file.js          → understand: what did THEY change?
   
   Think: "Does both sides' change make sense? Should both be preserved?
           Is one of these now wrong given the other's context?"
   
   Edit the file → remove all <<<< ==== >>>> markers → write correct code
   git diff --check file.js     → verify no markers remain
   git add file.js

4. VERIFY:
   git diff --staged            → review everything staged for commit
   git status                   → confirm no remaining conflicts

5. COMMIT/CONTINUE:
   git commit                   (merge) or git rebase --continue (rebase)
```

### Handling the "This Is Too Complex" Conflict

**Scenario**: You're rebasing and hit a conflict in 15 files at once. The conflict spans 500 lines. You've been staring at it for 2 hours.

```bash
# Option A: Take a break from rebasing — save your work
git stash              # stash in-progress rebase? No — can't stash during rebase
# Actually: the rebase-in-progress state is already saved in .git/rebase-merge/
# You can just close the terminal and come back
# The rebase state is fully persistent

# Option B: Bring in help
git show :1:complex-file.js > /tmp/base.js   # save base to temp file
git show :2:complex-file.js > /tmp/ours.js   # save ours to temp file
git show :3:complex-file.js > /tmp/theirs.js # save theirs to temp file
# Share these with the original author for consultation

# Option C: Accept entirely one side for now, fix later
git checkout --theirs complex-file.js  # accept their version entirely
git add complex-file.js
# Make a note: "TODO: reconcile our changes to complex-file.js"
# Open a separate PR for the reconciliation after rebase completes

# Option D: Abort, make smaller commits, try again
git rebase --abort
git rebase -i HEAD~8   # re-order/squash your commits
# Make each commit smaller so each rebase step has fewer conflicts
git rebase origin/main  # retry
```

### Using This Knowledge to Speed Up Team Onboarding

When a new developer joins and encounters their first conflict:

```
What to teach them in 5 minutes:
1. "git status shows which files are conflicted"
2. "Open the file — find <<<< and >>>>"
3. "The <<<< to ==== is ours, ==== to >>>> is theirs"
4. "Delete the markers, write the correct version"
5. "git add the file, then git commit or git rebase --continue"
6. "ALWAYS run git diff --check before committing"

What to teach them in 30 minutes (add):
7. diff3 conflict style — show the ancestor
8. ours vs theirs orientation during rebase
9. git show :1:/:2:/:3: for explicit stage reading
10. git merge --abort / git rebase --abort as escape hatches
```

---

*End of Topic 18 — Resolving Conflicts Like a Senior*

**Next**: Topic 19 — Git Hooks (pre-commit, commit-msg, pre-push, the `.git/hooks/` directory, husky, lint-staged, automating quality gates across teams)
