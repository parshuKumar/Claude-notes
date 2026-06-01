# Topic 22 — git blame & git log Mastery: Codebase Archaeology
## "Answer 'Who Wrote This?', 'Why Does This Line Exist?', and 'When Did This Behavior Change?' — Permanently"

> **Phase 6 — Advanced & Principal-Level** | Builds on Topics 02, 03, 06, 07, 11

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The Forensic Detective](#1-eli5-anchor)
2. [Physical Architecture — How Blame and Log Read the Object Store](#2-physical-architecture)
3. [git blame — Deep Dive](#3-git-blame-deep-dive)
4. [git blame — Exact Data Flow](#4-git-blame-exact-data-flow)
5. [git blame — Flags That Change Everything](#5-git-blame-flags)
6. [git log — The Full Power](#6-git-log-the-full-power)
7. [git log Filtering — Finding Exactly What You Need](#7-git-log-filtering)
8. [The Pickaxe — `-S` and `-G` Flags](#8-the-pickaxe)
9. [`--follow` — Tracking Files Across Renames](#9-follow-flag)
10. [git log Formatting — Custom Output](#10-git-log-formatting)
11. [git log with Diff — `-p`, `--stat`, `--name-only`](#11-git-log-with-diff)
12. [git shortlog — Contribution Summaries](#12-git-shortlog)
13. [Combining blame + log — The Full Investigation Workflow](#13-full-investigation-workflow)
14. [Byte-Level Internals — How blame Reconstructs Line History](#14-byte-level-internals)
15. [Beginner → Production Examples](#15-beginner-to-production-examples)
16. [Mistakes → Root Cause → Fix → Prevention](#16-mistakes)
17. [Mental Model Checkpoint — 10 Questions](#17-mental-model-checkpoint)
18. [Connect Everything Backwards](#18-connect-everything-backwards)
19. [Quick Reference Card](#19-quick-reference-card)
20. [When Would I Use This At Work?](#20-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The Forensic Detective

Imagine you pick up an old book from a library. On page 237, there's a paragraph that 
makes no sense. You want to know: **Who wrote that paragraph? When? And why?**

A detective doesn't just look at the paragraph — they look at the author's notes, the 
drafts, the edit history, all previous versions of that chapter. They trace the evidence 
back to the source.

`git blame` and `git log` are your forensic detective tools:

**`git blame file.js`:**
- Shows every single line of a file
- Next to each line: who last changed it, when, and in which commit
- Like having a "margin annotation" on every line of a book showing who wrote it

**`git log`:**
- The full audit trail of every change ever made
- Filterable, searchable, formattable
- Can search for when a specific word/function appeared or disappeared (the "pickaxe")
- Can follow a file even after it was renamed

**Together they answer the most important engineering questions:**
- "Who wrote this?" → `git blame`
- "Why does this line exist?" → `git blame` → commit SHA → `git show`
- "When did this behavior change?" → `git log -S "function_name"`
- "What has changed in the last 2 weeks?" → `git log --since="2 weeks ago"`
- "What did Alice change last month?" → `git log --author="Alice" --since="last month"`
- "Show me all commits that touched this file, even after it was renamed" → `git log --follow`

---

## 2. Physical Architecture

### How `git blame` and `git log` Navigate the Object Store

Neither `git blame` nor `git log` modifies anything on disk. They are **read-only 
traversal tools** that walk the object database in `.git/objects/`.

```
.git/
├── HEAD                    ← starting point for log (current branch tip)
├── refs/
│   ├── heads/
│   │   └── main            ← git log starts here: reads this SHA
│   └── tags/               ← git log --tags includes these
├── objects/
│   ├── pack/
│   │   ├── pack-abc123.idx ← index for fast SHA lookup
│   │   └── pack-abc123.pack← compressed pack of many objects
│   └── ab/
│       └── cdef1234...     ← loose object (blob/tree/commit)
└── logs/
    └── HEAD                ← git log -g reads this (reflog)
```

**The traversal pattern for `git log`:**

```
START: Read .git/HEAD → "ref: refs/heads/main"
       Read .git/refs/heads/main → SHA of latest commit: C8

WALK:
  Read commit object C8 from .git/objects/
  → Extract: tree SHA, parent SHA (C7), author, committer, message
  → Print C8 info (filtered by --author, --grep, etc.)

  Read commit object C7 (parent of C8)
  → Extract: tree SHA, parent SHA (C6), author, committer, message
  → Print C7 info

  ... continue until:
    - No more parents (initial commit)
    - OR --max-count limit reached
    - OR --since/--until date limit reached
    - OR --ancestry-path filter stops it
```

**The traversal pattern for `git blame`:**

```
START: Identify the target file + revision (default: HEAD)
       Find the blob SHA for the file in HEAD's tree

WALK BACKWARDS through commit history:
  For each commit that touched this file:
    Compare the blob at this commit with the blob at the parent commit
    For each line in the current blob:
      If this line first appeared at commit X → attribute it to X
      Otherwise → continue looking at X's parent

  Continue until all lines have been attributed to a commit.

OUTPUT: Each line of the file, annotated with:
  - Commit SHA (abbreviated)
  - Author name
  - Timestamp
  - Line number
  - Line content
```

**PROVE IT — See the underlying traversal:**

```bash
# Watch git log walk the commit graph manually
git log --pretty=format:"%H %P" | head -10
# Each line: <commit-sha> <parent-sha(s)>
# For merge commits: two parent SHAs separated by space

# See the tree of each commit
git log --pretty=format:"%H %T" | head -5
# <commit-sha> <tree-sha>
# You can then inspect: git ls-tree <tree-sha>
```

---

## 3. git blame — Deep Dive

### What `git blame` Actually Shows

```bash
git blame src/auth/middleware.js
```

Sample output:

```
^a3f2c1d (Alice Johnson  2026-01-15 09:32:17 +0530  1) const jwt = require('jsonwebtoken');
^a3f2c1d (Alice Johnson  2026-01-15 09:32:17 +0530  2) const { promisify } = require('util');
7c3e8f0d (Bob Smith      2026-02-20 14:45:00 +0530  3) const rateLimit = require('express-rate-limit');
7c3e8f0d (Bob Smith      2026-02-20 14:45:00 +0530  4)
d1c2b3a4 (Alice Johnson  2026-03-10 11:20:33 +0530  5) const verifyToken = promisify(jwt.verify);
d1c2b3a4 (Alice Johnson  2026-03-10 11:20:33 +0530  6)
e5f6d7c8 (Carol Wei      2026-04-02 16:00:00 +0530  7) const RATE_LIMIT_WINDOW = 15 * 60 * 1000;
^a3f2c1d (Alice Johnson  2026-01-15 09:32:17 +0530  8) const MAX_REQUESTS = 100;
```

**Column breakdown:**

```
^a3f2c1d (Alice Johnson  2026-01-15 09:32:17 +0530  1) const jwt = require('jsonwebtoken');
│         │              │                      │    │  │
│         │              │                      │    │  └─ Line content (with leading spaces preserved)
│         │              │                      │    └─ Line number in the CURRENT file
│         │              │                      └─ Timezone offset
│         │              └─ Timestamp (when the commit was authored)
│         └─ Author name (from the commit's author field, not committer)
└─ Abbreviated commit SHA (8 chars by default)
   ^ prefix = this commit is a boundary commit (e.g., the initial commit or the --since boundary)
```

**The `^` prefix on SHAs:**

The `^` prefix marks a "boundary commit" — a commit at the edge of the blame range. By 
default, boundary = the initial commit. With `--since` or `--reverse`, boundary = the 
start of the requested range. Boundary commits are visually distinguished because you 
can't go further back to attribute the line.

**Boundary commits also appear when using `git blame -C` or `-M`** to indicate lines 
that were copied from another file or moved within the file.

### The Commit SHA Is Your Gateway

The most important thing in blame output is the **commit SHA**. Every line's SHA is 
your direct link to:

```bash
# Get the full context of why a line was written
git show a3f2c1d
# Shows: full diff of that commit, commit message (the WHY), author, date

# See just the commit message:
git log --format="%B" -1 a3f2c1d
# Output: the full commit message body

# See which PR/ticket introduced this change:
git log --format="%B" -1 a3f2c1d | grep -E "JIRA|TICKET|PR|#[0-9]+"
# Often commit messages reference the ticket that motivated the change
```

### PROVE IT — Trace a line from blame to context

```bash
# Step 1: Find a suspicious line
git blame src/api/users.js | grep "hardcoded\|TODO\|FIXME\|magic"

# Step 2: Get the commit SHA from that line (first column)
# Say it shows: 3b4c5d6e

# Step 3: See the full story
git show 3b4c5d6e
# Full diff + message: "temporary fix for prod incident — see INCIDENT-2847"

# Step 4: See all files changed in that commit
git show --stat 3b4c5d6e

# Step 5: See who else was involved
git log --format="%an <%ae>" 3b4c5d6e^..3b4c5d6e
```

---

## 4. git blame — Exact Data Flow

### What Happens When You Run `git blame src/auth/middleware.js`

```
COMMAND: git blame src/auth/middleware.js

STEP 1: RESOLVE the target revision
  Reads: .git/HEAD → "ref: refs/heads/main"
  Reads: .git/refs/heads/main → "d1c2b3a4..."
  Starting commit: d1c2b3a4 (HEAD of main)

STEP 2: FIND the blob for this file at HEAD
  Reads: commit object d1c2b3a4 → tree SHA: "e5f6d7c8..."
  Reads: tree object e5f6d7c8 → finds entry: "100644 blob f9a0b1c2 src/auth/middleware.js"
  File blob at HEAD: f9a0b1c2

STEP 3: INITIALIZE the blame map
  Reads: blob f9a0b1c2 → file content (N lines)
  Creates: blame_map[1..N] = {commit: d1c2b3a4, line: original_line_content}
  All lines start attributed to the HEAD commit (default attribution)

STEP 4: WALK backwards through commit history
  For each commit C in the walk (newest to oldest):
    
    a. Find the parent P of C
    b. Find the blob for this file at P
    c. Run a diff: blob-at-P vs blob-at-C
    d. For each line in the diff:
       If a line EXISTS in C but NOT in P → this line was ADDED in commit C
       → Update blame_map[line_number].commit = C
    e. If all lines are attributed → STOP
    f. Otherwise → continue to P

STEP 5: FORMAT and print output
  For each line 1..N:
    Read: blame_map[line].commit → commit SHA
    Read: that commit object → author name, author timestamp
    Print: "<short-sha> (<author> <timestamp> <lineno>) <line content>"
```

**The diff algorithm used internally:**

Git blame uses a **patience diff** (or xdiff) algorithm to compare blobs between commits. 
This is the same diff algorithm that `git diff` uses, but blame applies it in reverse — 
walking backwards from HEAD to find the earliest commit where each line appeared.

**Performance note:** For large files with long histories, `git blame` can be slow because 
it has to read and diff potentially thousands of commit pairs. Git caches intermediate 
results using delta compression in packfiles (covered in Topic 25).

---

## 5. git blame — Flags That Change Everything

### `-L` — Blame Only a Line Range

```
git blame -L <start>,<end> <file>
git blame -L <start>,+<count> <file>
git blame -L /<regex>/ <file>
           │
           └─ -L flag with line range selector

Forms:
  -L 10,20          → lines 10 through 20 (inclusive)
  -L 10,+5          → lines 10 through 14 (start + count)
  -L /function/     → from the line matching /function/ to end of next function
  -L /start/,/end/  → from /start/ match to /end/ match
```

```bash
# Blame only the function starting at line 45
git blame -L 45,80 src/payment.js

# Blame from a function signature to the next function
git blame -L '/^function processPayment/','/^function/' src/payment.js

# Blame the last 20 lines
git blame -L -20,+20 src/payment.js  # doesn't work directly; use:
TOTAL=$(wc -l < src/payment.js)
git blame -L $((TOTAL-20)),$TOTAL src/payment.js
```

---

### `-w` — Ignore Whitespace Changes

```bash
git blame -w src/api.js
# Ignores commits that only changed whitespace (indentation, blank lines)
# Useful when a reformatting commit would otherwise claim authorship of many lines
```

**Why this matters:** If someone ran a Prettier/ESLint auto-format over the whole file, 
every line would show that reformatting commit without `-w`. Use `-w` to see through 
formatting-only changes to the substantive author.

---

### `-M` — Detect Moved Lines Within the File

```bash
git blame -M src/utils.js
# If lines were moved around WITHIN the same file (e.g., a function was moved to the
# top from the bottom), -M attributes those lines to their ORIGINAL commit, not the
# move commit.

# -M[N] — threshold: N% of lines must match to be considered a move (default: 20)
git blame -M40 src/utils.js  # require 40% similarity to count as a move
```

---

### `-C` — Detect Lines Copied FROM Other Files

```bash
git blame -C src/auth.js
# If lines in auth.js were copied from another file (e.g., utils.js),
# -C attributes those lines to the ORIGINAL file and commit, not the copy commit.

# -CC — more aggressive: also detect copies from files modified in the same commit
git blame -CC src/auth.js

# -CCC — most aggressive: detect copies from any file in any commit
git blame -CCC src/auth.js
```

**The `-C` flag is very powerful for understanding "this code was originally written 
in X and copied here — the original context is over there."**

---

### `--since` / `--ignore-rev` / `--ignore-revs-file`

```bash
# Only show blame since a certain date (don't go further back)
git blame --since=6.months.ago src/server.js

# Ignore a specific commit when doing blame (e.g., a mass reformatting)
git blame --ignore-rev a3f2c1d src/server.js
# Lines changed by a3f2c1d will be attributed to the PREVIOUS author of those lines

# Ignore multiple commits via a file (the standard approach for large teams)
# Create .git-blame-ignore-revs:
echo "a3f2c1d # Prettier reformat 2026-01-10" >> .git-blame-ignore-revs
echo "7c3e8f0 # ESLint auto-fix 2026-03-15" >> .git-blame-ignore-revs

git blame --ignore-revs-file .git-blame-ignore-revs src/server.js

# Configure this file globally for the repo (so all developers skip these commits):
git config blame.ignoreRevsFile .git-blame-ignore-revs
# Now git blame automatically skips these commits for everyone
```

**The `.git-blame-ignore-revs` file is one of the most important team conventions** for 
repos that have had mass reformatting commits. Commit it to the repo root so everyone 
benefits.

---

### Blaming a Specific Revision

```bash
# Blame the file at a specific commit (not HEAD)
git blame a3f2c1d -- src/auth.js
# Shows who wrote each line as of commit a3f2c1d

# Blame the file at a tag
git blame v2.0.0 -- src/auth.js

# Blame the file as it was 3 months ago
git blame "HEAD@{3 months ago}" -- src/auth.js

# Blame an older version to understand pre-refactor code
git blame main~50 -- src/legacy/parser.js
```

---

### `-e` — Show Email Instead of Name

```bash
git blame -e src/auth.js
# Output:
# a3f2c1d (<alice@company.io> 2026-01-15 ...) const jwt = ...
```

---

### `-n` / `-s` — Line Number and Short Format

```bash
# Show original line number (before file was edited)
git blame -n src/auth.js

# Short format: no timestamp, no author (just SHA + line content)
git blame -s src/auth.js
# Useful for quick SHA lookup without visual clutter
```

---

## 6. git log — The Full Power

### Understanding What `git log` Does

`git log` walks the commit graph from a starting point (default: HEAD) backwards through 
parent pointers. It prints commit metadata. Every flag is a filter or a format modifier.

### Basic Syntax

```
git log [<options>] [<revision range>] [[--] <path>...]
        │            │                  │
        │            │                  └─ Only show commits that touched these paths
        │            └─ A range of commits to show:
        │               <since>..<until>   = commits reachable from until but not since
        │               <commit>           = start from this commit
        │               HEAD~10            = last 10 commits
        │               v1.0..v2.0         = between two tags
        └─ Format, filter, and display flags
```

### Output Formats

```bash
# Default: full commit info (SHA, author, date, message)
git log

# One line per commit (short SHA + first line of message)
git log --oneline

# One line + graph for branches/merges
git log --oneline --graph

# One line + graph + all refs (branches, tags, remotes)
git log --oneline --graph --all

# Full diff for each commit (most verbose)
git log -p

# Summary of files changed per commit
git log --stat

# Just the file names changed per commit
git log --name-only

# File names with change type (M=modified, A=added, D=deleted, R=renamed)
git log --name-status
```

### The `--graph` Flag — Visualizing Branch Structure

```bash
git log --oneline --graph --all --decorate
```

Output:
```
*   d1c2b3a (HEAD -> main, origin/main) Merge pull request #142 from feature/auth
|\
| * 7c3e8f0 (origin/feature/auth) implement JWT refresh tokens
| * 2b4d6e8 add JWT middleware
|/
* a3f2c1d refactor user model
* 8f9a0b1 (tag: v2.1.0) release v2.1.0 preparation
* e5d4c3b fix payment timeout issue
```

This ASCII graph is constructed by reading the parent pointers from commit objects and 
building a dependency tree for display purposes.

---

## 7. git log Filtering — Finding Exactly What You Need

### By Author

```
git log --author=<pattern>
                 │
                 └─ Matches against "Author: Name <email>" line of each commit
                    Pattern is a regex (case-insensitive by default)
                    Can match partial name or email
```

```bash
# All commits by Alice
git log --author="Alice"

# All commits by anyone with a @company.io email
git log --author="@company.io"

# All commits by Alice OR Bob (regex alternation)
git log --author="Alice\|Bob"

# All commits by Alice in the last 30 days
git log --author="Alice" --since="30 days ago"
```

---

### By Date

```bash
git log --since="2026-01-01"              # since a date
git log --after="2026-01-01"              # same as --since
git log --until="2026-03-31"              # before a date
git log --before="2026-03-31"             # same as --until
git log --since="2 weeks ago"             # relative
git log --since="yesterday"               # relative
git log --since="2026-01-01" --until="2026-03-31"  # date range (Q1 2026)
```

---

### By Commit Message

```
git log --grep=<pattern>
               │
               └─ Regex match against commit message (subject + body)
                  Case-insensitive by default
                  --fixed-strings: treat as literal string (no regex)
                  --all-match: commit must match ALL --grep patterns
                  --invert-grep: commits NOT matching the pattern
```

```bash
# All commits mentioning JIRA ticket AUTH-123
git log --grep="AUTH-123"

# All commits with "fix" in the message (matches fixup, bugfix, etc.)
git log --grep="fix" --oneline

# Commits with "refactor" AND "auth" (both must appear)
git log --grep="refactor" --grep="auth" --all-match --oneline

# Commits NOT mentioning "WIP" or "temp"
git log --grep="WIP\|temp" --invert-grep --oneline

# Commits with a specific conventional commit type
git log --grep="^feat\|^fix\|^refactor" --oneline
```

---

### By Changed Content (The Pickaxe — Section 8)

```bash
git log -S "processPayment"       # when this string appeared/disappeared
git log -G "processPayment.*rate" # when this regex matched changed lines
```

(Full coverage in Section 8)

---

### By Path

```bash
# All commits that touched a specific file
git log -- src/auth/middleware.js

# All commits that touched any file in a directory
git log -- src/auth/

# All commits that touched either file
git log -- src/auth/middleware.js src/auth/tokens.js

# Combine with author and date filters
git log --author="Alice" --since="2 months ago" -- src/api/
```

---

### By Revision Range

```bash
# Commits on feature/auth that aren't on main
git log main..feature/auth

# Commits reachable from either branch but not both (symmetric difference)
git log main...feature/auth

# Last 10 commits
git log -10

# Commits from v1.0.0 to v2.0.0
git log v1.0.0..v2.0.0

# Commits from v1.0.0 to v2.0.0 that touched src/
git log v1.0.0..v2.0.0 -- src/

# All commits since a tag
git log v2.1.0..HEAD

# Commits reachable from HEAD but not from any remote
git log @{upstream}..HEAD
# Shows "what I have locally that's not pushed yet"
```

---

### Combining Filters

```bash
# Complex query: Alice's commits since last month that touched auth files
# AND contained "fix" in the message
git log \
  --author="Alice" \
  --since="1 month ago" \
  --grep="fix" \
  -- src/auth/ \
  --oneline

# Commits that added/removed the word "deprecated" in src/ since v2.0
git log -S "deprecated" --since-tag=v2.0 -- src/
```

---

## 8. The Pickaxe — `-S` and `-G` Flags

### The Most Powerful Forensic Tool in Git

The "pickaxe" is Git's ability to search through commit history for changes to specific 
content — not the commit message, but the ACTUAL CODE. It answers:

- "When was the function `processPayment()` first added?"
- "When was the API key removed from the config file?"
- "Which commit changed the rate limit from 100 to 200?"

### `-S` — The String Pickaxe

```
git log -S <string> [options]
           │
           └─ Find commits where the NUMBER OF OCCURRENCES of <string> changed.
              A commit is selected if:
                count of <string> in the file BEFORE the commit ≠
                count of <string> in the file AFTER the commit

              This catches:
                - First time the string was ADDED (0 → N occurrences)
                - Last time the string was REMOVED (N → 0 occurrences)
                - NOT commits that just moved the string around (N → N occurrences)
```

**Critical distinction:** `-S` only fires when the COUNT changes. If a commit moves the 
string from line 10 to line 20 within the same file, `-S` does NOT show that commit 
(count is still 1 before and after). This is a feature — it finds true introductions and 
deletions, not just edits.

```bash
# When was processPayment() first added to the codebase?
git log -S "processPayment" --oneline
# Output (oldest first if you reverse):
# a3f2c1d add payment processing module
# → This is the commit that first introduced the string

# When was the old API key removed?
git log -S "sk_live_old_key_value" --oneline
# Output:
# 7c3e8f0 remove hardcoded API keys (SECURITY-1234)
# → This is the commit that removed it

# See the actual diff with the context:
git log -S "processPayment" -p --oneline
# Shows the commit + the full diff showing the addition/removal
```

**`-S` with `--pickaxe-all`:**

```bash
# By default, -S only shows commits where the count changed in at least one file.
# With --pickaxe-all, only show commits where the count changed in ALL files.
# (Rarely needed but useful for exact string tracking across file moves.)
git log -S "processPayment" --pickaxe-all
```

**`-S` with `--pickaxe-regex`:**

```bash
# Treat the -S string as a regex
git log -S "processPayment|handlePayment" --pickaxe-regex --oneline
# Finds commits that changed the count of either string
```

### `-G` — The Regex Pickaxe

```
git log -G <regex> [options]
           │
           └─ Find commits where ANY LINE in the diff matches <regex>.
              A commit is selected if:
                the patch (diff) contains at least one added or removed line
                that matches the regex.

              Unlike -S:
                - Matches changed LINES, not occurrence count
                - Will fire even if the string moved (if the line changed)
                - Matches anywhere in the line (not just equal count)
```

```bash
# Find commits where any changed line references a rate limit value
git log -G "rateLimit\s*=\s*[0-9]+" --oneline

# Find commits that changed any import statement in auth files
git log -G "^import.*from.*auth" -- src/ --oneline

# Find commits that changed a specific function signature
git log -G "function authenticate\(" --oneline

# See full diffs of those commits
git log -G "function authenticate\(" -p --oneline
```

### `-S` vs `-G` — When to Use Which

| Situation | Use |
|-----------|-----|
| "When was this exact string first added?" | `-S "exact_string"` |
| "When was this exact string removed?" | `-S "exact_string"` |
| "Find commits that touched a pattern in code" | `-G "regex"` |
| "This function was renamed — find the rename commit" | `-G "old_name|new_name"` |
| "Audit: when was this API key in the code?" | `-S "api_key_value"` |
| "Find all commits modifying SQL queries" | `-G "SELECT.*FROM.*users"` |

**Practical example — security audit:**

```bash
# Find every commit that ever contained an AWS access key pattern
git log -S "AKIA" --all --oneline
# --all = search ALL branches and tags (not just HEAD)

# Show the actual context (the line that had it):
git log -S "AKIA" --all -p | grep -A 3 -B 3 "AKIA"
```

**Practical example — API evolution:**

```bash
# Track the lifecycle of a deprecated function
git log -S "legacyProcessOrder" --oneline --reverse
# First output line: when it was ADDED
# Last output line: when it was REMOVED (if it has been)

# With dates:
git log -S "legacyProcessOrder" --format="%as %h %s" --reverse
# 2024-03-10 a3f2c1d add legacy order processor
# 2025-11-22 7c3e8f0 remove deprecated legacyProcessOrder
```

---

## 9. `--follow` — Tracking Files Across Renames

### The Problem Without `--follow`

```bash
# Normal git log on a renamed file shows only changes AFTER the rename
git log -- src/api/users.js
# Output: only commits since users.js was created (or last renamed to this name)
# If users.js used to be called controllers/userController.js, you miss all that history!
```

### How `--follow` Works

```bash
git log --follow -- src/api/users.js
# Follows the file backwards through renames
# If at some point users.js was renamed from userController.js,
# --follow continues looking at userController.js history before the rename
```

**What Git uses to detect renames:** When `--follow` reaches a commit where the file 
appeared from nowhere (was "added"), Git checks if any other file was "deleted" in the 
same commit. If the two files are sufficiently similar (similarity score > 50% by default, 
configurable with `-M<N>`), Git concludes it was a rename.

```bash
# Follow with a higher rename similarity threshold
git log --follow -M90% -- src/api/users.js
# Only follow the rename if the files are >90% similar (more conservative)

# See the log with diffs showing the rename
git log --follow --stat -- src/api/users.js
# Output will include: "controllers/userController.js → src/api/users.js" at the rename commit
```

### Combining `--follow` with Other Flags

```bash
# Full history of a file including renames, with diffs
git log --follow -p -- src/api/users.js

# Full history with just stats
git log --follow --stat -- src/api/users.js

# Who touched this file at any point in history (across renames)
git log --follow --format="%an" -- src/api/users.js | sort | uniq -c | sort -rn
```

### PROVE IT — See the Rename in History

```bash
# Create a file, commit it, rename it, commit again, then trace history
mkdir -p test-follow && cd test-follow
git init
echo "original content" > oldname.js
git add . && git commit -m "add oldname.js"
echo "more content" >> oldname.js
git add . && git commit -m "update oldname.js"
git mv oldname.js newname.js
git commit -m "rename oldname.js to newname.js"
echo "after rename content" >> newname.js
git add . && git commit -m "update after rename"

# Without --follow: only sees commits after the rename
git log --oneline -- newname.js
# a3f2c1d update after rename
# 7c3e8f0 rename oldname.js to newname.js

# With --follow: sees the full history through the rename
git log --follow --oneline -- newname.js
# a3f2c1d update after rename
# 7c3e8f0 rename oldname.js to newname.js
# 2b4d6e8 update oldname.js      ← found through the rename!
# d1c2b3a add oldname.js         ← original creation!
```

---

## 10. git log Formatting — Custom Output

### `--format` and `--pretty`

`--pretty=format:<string>` lets you build any output format. This is essential for 
scripting and automation.

```bash
git log --pretty=format:"<placeholders>"
```

**The most useful placeholders:**

| Placeholder | Meaning |
|-------------|---------|
| `%H` | Full commit SHA (40 chars) |
| `%h` | Abbreviated commit SHA (7 chars) |
| `%T` | Full tree SHA |
| `%P` | Full parent SHA(s) |
| `%p` | Abbreviated parent SHA(s) |
| `%an` | Author name |
| `%ae` | Author email |
| `%ad` | Author date (format configurable with `--date=`) |
| `%ar` | Author date, relative ("2 weeks ago") |
| `%as` | Author date, short format (YYYY-MM-DD) |
| `%cn` | Committer name |
| `%ce` | Committer email |
| `%cd` | Committer date |
| `%s` | Subject (first line of commit message) |
| `%b` | Body (rest of commit message, after subject) |
| `%B` | Full commit message (subject + blank line + body) |
| `%D` | Ref names (branch/tag decorations) |
| `%n` | Newline |
| `%x00` | Null byte (for NUL-delimited scripting) |

```bash
# CSV-style output for spreadsheet import
git log --pretty=format:"%as,%h,%an,%s" --since="2026-01-01"
# 2026-05-20,d1c2b3a,Alice Johnson,feat: add two-factor authentication
# 2026-05-19,7c3e8f0,Bob Smith,fix: resolve rate limit race condition

# Tab-separated for awk/cut processing
git log --pretty=format:"%H%x09%ae%x09%as%x09%s"

# JSON-style (useful for piping to jq)
git log --pretty=format:'{"sha":"%h","author":"%an","date":"%as","msg":"%s"}'

# Just author emails (for mailing list)
git log --pretty=format:"%ae" --since="1 month ago" | sort -u

# Commit count per author in current month
git log --pretty=format:"%an" --since="2026-05-01" | sort | uniq -c | sort -rn
```

### `--date` Flag Options

```bash
git log --date=iso          # 2026-05-20 14:30:00 +0530
git log --date=iso-strict   # 2026-05-20T14:30:00+05:30
git log --date=rfc           # Wed, 20 May 2026 14:30:00 +0530
git log --date=short         # 2026-05-20
git log --date=relative      # 2 days ago
git log --date=unix          # 1748390400 (Unix epoch)
git log --date=format:"%Y-%m" # Custom strftime: "2026-05"
```

### Built-in `--pretty` Presets

```bash
git log --pretty=oneline    # full SHA + subject
git log --pretty=short      # SHA + author + subject
git log --pretty=medium     # SHA + author + date + subject + body (default)
git log --pretty=full       # SHA + author + committer + subject + body
git log --pretty=fuller     # SHA + both dates + both names + subject + body
git log --pretty=email      # RFC 2822 email format (for git format-patch)
git log --pretty=raw        # raw commit object format
```

### `--format` vs `--pretty=format`

These are identical: `--format="..."` is an alias for `--pretty=format:"..."`.

---

## 11. git log with Diff — `-p`, `--stat`, `--name-only`

### Showing What Actually Changed

```bash
# Full patch (diff) for every commit
git log -p

# Limit to specific file + full diff
git log -p -- src/auth/middleware.js

# Stats: file name, insertions, deletions
git log --stat
# Output:
# commit d1c2b3a...
# Author: Alice...
# Date: ...
#     feat: add JWT refresh tokens
#
#  src/auth/middleware.js | 45 ++++++++++++++++---
#  src/auth/tokens.js     | 32 ++++++++++++
#  2 files changed, 77 insertions(+), 3 deletions(-)

# File names only (no line counts)
git log --name-only

# File names + change type (M/A/D/R/C)
git log --name-status
# Output:
# M    src/auth/middleware.js
# A    src/auth/tokens.js

# Diff stat as a single percentage bar per file
git log --stat --compact-summary

# Combine: oneline + stat
git log --oneline --stat
```

### Limiting Diff Output

```bash
# Show only files with more than 50 lines changed
git log --stat --diff-filter=M  # M=modified, A=added, D=deleted, R=renamed

# Only show added files
git log --diff-filter=A --name-only

# Only show deleted files
git log --diff-filter=D --name-only

# Show renames specifically
git log --diff-filter=R --name-status

# Multiple filters: show added OR modified files
git log --diff-filter=AM --name-only
```

### `-p` with Context

```bash
# Show diff with 5 lines of context (default is 3)
git log -p -U5

# Show diff with NO context (just changed lines)
git log -p -U0

# Show diff with function context (show which function the change is in)
git log -p --function-context
```

---

## 12. git shortlog — Contribution Summaries

### Aggregating Commit Activity

```bash
# Contribution summary: author + commit count + list of messages
git shortlog
# Output:
# Alice Johnson (34):
#       feat: add JWT authentication
#       feat: implement refresh tokens
#       fix: resolve token expiry race condition
#       ...
#
# Bob Smith (12):
#       feat: add rate limiting middleware
#       fix: correct rate limit window calculation
#       ...

# Just the counts, sorted by commit count (descending)
git shortlog -sn
# Output:
#    34  Alice Johnson
#    12  Bob Smith
#     8  Carol Wei

# Count by email instead of name
git shortlog -se

# Limit to recent commits
git shortlog -sn --since="3 months ago"

# Limit to specific branch range
git shortlog -sn v1.0.0..v2.0.0

# Limit to a path
git shortlog -sn -- src/auth/
```

### Using shortlog for Release Notes

```bash
# Generate formatted release notes between two tags
git shortlog v2.0.0..v2.1.0
# Groups commits by author
# Useful starting point for CHANGELOG.md entries

# Or with custom format for a release PR:
git log v2.0.0..v2.1.0 --pretty=format:"- %s (%an)" | head -20
```

---

## 13. Combining blame + log — The Full Investigation Workflow

### The Standard Forensic Sequence

When you encounter a mysterious line of code, the investigation always follows the same 
steps. Master this sequence and you will never be confused about why code exists.

```
STEP 1: Find the suspicious code
  You're reading src/payments/processor.js and see:
  Line 147: const RETRY_DELAY = 8500;  // magic number — why 8500?

STEP 2: git blame to find the commit
  git blame -L 147,147 src/payments/processor.js
  # Output:
  # 3b4c5d6e (Dev Name 2025-11-03 16:22:00 +0530 147) const RETRY_DELAY = 8500;

STEP 3: git show to see the full context
  git show 3b4c5d6e
  # Output includes commit message:
  # "fix: increase payment retry delay to avoid Stripe rate limit (INCIDENT-2893)"
  # Diff shows: was 3000, changed to 8500
  # AH — the magic number is a Stripe API rate limit workaround from an incident!

STEP 4: Find related changes (what else changed in that incident?)
  git show --stat 3b4c5d6e
  # src/payments/processor.js   | 3 +--
  # src/payments/retry.js       | 15 +++++++++-----
  # docs/incidents/2893.md      | 45 +++++++++++++++++++++++++++

STEP 5: Track the full history of that value
  git log -S "RETRY_DELAY" --oneline --reverse
  # a9b8c7d add payment retry logic (2025-06-10)  ← first added
  # 3b4c5d6e fix: increase payment retry delay (2025-11-03)  ← changed to 8500
  # Now you know the full lifetime of this constant

STEP 6: See if the incident doc explains it
  git show 3b4c5d6e -- docs/incidents/2893.md | head -50
  # Full incident postmortem!
```

### The "Why Was This Removed?" Workflow

```bash
# You notice a function is GONE from the code. Where did it go?

# Step 1: Search all history for when it existed
git log -S "function validateLegacyToken" --oneline
# Output:
# d4e5f6a add legacy token validation (2024-01-15)
# 8c9d0e1 remove deprecated legacy token validation (2025-09-20)

# Step 2: See the removal context
git show 8c9d0e1
# Commit message: "remove deprecated validateLegacyToken after migration complete"
# Diff: shows the function being deleted

# Step 3: See the last state of the function before removal
git show 8c9d0e1^:src/auth/tokens.js | grep -A 20 "function validateLegacyToken"
# Shows the full function as it existed just before it was deleted
```

### The "When Did This File Change So Much?" Workflow

```bash
# You inherited a file and want to understand its full evolution
git log --follow --oneline -- src/api/v2/routes.js
# Shows all commits, following through any renames

# See the most impactful commits (most lines changed)
git log --follow --stat --oneline -- src/api/v2/routes.js | 
  grep -E "^[a-f0-9]+ |changed" | 
  paste - - | 
  sort -t',' -k2 -rn | 
  head -10
# Shows the commits with the most insertions/deletions

# See who has owned this file historically
git log --follow --format="%an" -- src/api/v2/routes.js | sort | uniq -c | sort -rn
# Shows the top contributors to this file across its entire life
```

### The "Audit Trail for a Feature" Workflow

```bash
# Stakeholder asks: "Walk me through everything that was done for the payment feature"

# Step 1: Find all relevant commits
git log --grep="payment\|stripe\|checkout" --oneline --reverse

# Step 2: Filter to a time range (when the feature was built)
git log --grep="payment\|stripe\|checkout" \
  --since="2025-10-01" --until="2026-01-31" \
  --oneline --reverse

# Step 3: See what files are most affected
git log --grep="payment" --name-only --since="2025-10-01" | 
  grep -v "^commit\|^Author\|^Date\|^$\|^    " | 
  sort | uniq -c | sort -rn | head -20
# Shows which files were touched most in payment-related commits

# Step 4: Generate a formatted timeline
git log --grep="payment" --since="2025-10-01" \
  --format="%as | %h | %an | %s" --reverse
# 2025-10-05 | a3f2c1d | Alice Johnson | feat: add Stripe integration
# 2025-10-12 | 7c3e8f0 | Bob Smith | feat: add payment webhook handler
# ...
```

---

## 14. Byte-Level Internals — How blame Reconstructs Line History

### The Diff Algorithm at the Core of Blame

`git blame` repeatedly runs a diff between file versions at consecutive commits. The 
diff algorithm used is **xdiff** (an extended Myers diff), operating on blob objects.

```
For each consecutive pair of commits (C, C_parent):

  blob_at_C = tree[C]["src/file.js"]        = SHA abc...
  blob_at_Cp = tree[C_parent]["src/file.js"] = SHA def...
  
  diff(blob_at_Cp, blob_at_C) = list of hunks:
    @@ -10,5 +10,7 @@          ← hunk header
     unchanged line 10
     unchanged line 11
    -deleted line 12
    +added line 12a
    +added line 12b
     unchanged line 13
     unchanged line 14
  
  Lines in the +added hunks = first appeared in C
  → blame_map[these line numbers] = C
```

### Incremental Blame and Delta Encoding

Git doesn't store the full content of every version of every file. Packfiles (Topic 25) 
store **deltas** — the difference between versions. `git blame` reconstructs full file 
content by applying delta chains:

```
Full snapshot (base object):   blob f9a0b1c2
   ↓ delta (15 bytes)
Version after 5 commits:       blob abc1def2
   ↓ delta (8 bytes)  
Current version (HEAD):        blob 7e8f9a0b

git blame must "unpack" these deltas to get the full content at each point,
then diff adjacent versions.
```

This is why `git blame` on files with long histories (1000+ commits) can be slow on 
cold disk reads — it has to reconstruct many intermediate versions.

### The Blame "Origin" Concept

Internally, Git blame tracks not just which commit introduced each line, but which 
**"origin"** — a (commit, line_number_in_that_commit) pair. This is what makes `-M` 
and `-C` work: even when lines move, their origin can be traced to the original commit.

```
blame_map[line 42] = {
  commit: "a3f2c1d",
  origin_lineno: 15,    ← line 42 in current file was line 15 in a3f2c1d
  content: "const jwt = require('jsonwebtoken');"
}
```

### The `git log` Commit Walk Algorithm

`git log` uses a **topological sort** of the commit DAG combined with time-based 
ordering:

```
Algorithm (simplified):
1. Initialize a priority queue with the starting commit(s)
2. Pop the highest-priority commit (by author date, or topology order)
3. Apply filters (--author, --grep, --since, etc.)
4. If commit passes filters: print it
5. Add all parents to the priority queue (if not already visited)
6. Repeat until queue is empty or limit reached

For --graph: additionally track which "lanes" in the graph each commit occupies
and render the ASCII art accordingly.
```

**The `--topo-order` flag:**
```bash
git log --topo-order
# Ensures children are always shown before parents
# Prevents interleaving of branch commits in the output
# Default for --graph
```

---

## 15. Beginner → Production Examples

### Beginner Example — Who Wrote That?

```bash
# Simple repo, 3 files, 10 commits. You want to understand a specific function.

# Step 1: Find the line
grep -n "calculateDiscount" src/cart.js
# 23: const discount = calculateDiscount(user, cart);

# Step 2: Blame that line
git blame -L 23,23 src/cart.js
# 4a5b6c7d (Junior Dev 2026-04-15 10:30:00 +0530 23) const discount = calculateDiscount(user, cart);

# Step 3: See why it was added
git show 4a5b6c7d
# "feat: add discount calculation for premium users"
# Diff shows: entire discount feature was added in this commit

# Step 4: See when the function itself was defined
git log -S "function calculateDiscount" --oneline --reverse
# 2b3c4d5e add discount utility (first addition)
# 4a5b6c7d feat: add discount calculation (usage)
```

---

### Production Example — Security Audit

**Context:** Your company is doing a SOC2 audit. The auditor asks: "Give us a complete 
history of every change to your authentication code, who made it, and when. We also need 
to know if any secrets were ever committed to the repository."

```bash
# TASK 1: Full history of auth-related files
git log --follow --format="%as | %h | %an <%ae> | %s" -- src/auth/ | 
  sort > auth-change-log.txt
# Produces: date | SHA | author email | message

# TASK 2: Find every developer who touched auth code
git log --follow --format="%an <%ae>" -- src/auth/ | 
  sort -u > auth-contributors.txt

# TASK 3: Find every commit that changed authentication logic
git log --grep="auth\|login\|password\|jwt\|token\|session" \
  --format="%as | %h | %an | %s" --reverse -- src/ > auth-commits.txt

# TASK 4: Secret scanning — was any API key or secret ever in the code?
# AWS keys (AKIA prefix)
git log -S "AKIA" --all --format="%as | %h | %an | %s" > potential-secrets.txt

# Stripe keys
git log -S "sk_live_" --all --format="%as | %h | %an | %s" >> potential-secrets.txt

# Generic patterns (password=, secret=, private_key=)
git log -G "password\s*=\s*['\"][^'\"]{8,}" --all --format="%as | %h | %an | %s" \
  >> potential-secrets.txt

# TASK 5: For each suspicious commit in potential-secrets.txt, show the actual line:
git show <sha> | grep -E "password|secret|key|AKIA"

# DELIVER: auth-change-log.txt, auth-contributors.txt, auth-commits.txt, potential-secrets.txt
```

**Business impact:** A full audit trail generated in minutes. Without these tools: 
months of manual code review.

---

### Production Example — Onboarding Investigation

**Context:** You join a new team. Your first task is to understand the `RateLimiter` 
class. You have no documentation. The original author left.

```bash
# Step 1: Find the file and its blame
git blame src/middleware/rateLimiter.js | head -30
# Lines 1-30 show 4 different authors and 6 different commits

# Step 2: Find when this class was originally written
git log --follow -S "class RateLimiter" --oneline --reverse -- src/
# d1c2b3a initial implementation of rate limiter (2023-08-10, original author)

# Step 3: See the original PR/commit context
git show d1c2b3a
# Message: "feat: implement distributed rate limiting with Redis"
# Diff: 200+ lines showing the original implementation

# Step 4: See the entire evolution
git log --follow --oneline -- src/middleware/rateLimiter.js
# 15 commits over 2 years — see every person who touched it

# Step 5: Find the most impactful changes
git log --follow --stat --oneline -- src/middleware/rateLimiter.js |
  grep -E "^[a-f0-9]|changed"
# Identify which commits had the most lines changed

# Step 6: Read each impactful commit message for architectural context
git log --follow --format="%H %s%n%b%n---" -- src/middleware/rateLimiter.js |
  head -100
# Full messages for all commits, separated by ---
# This is your documentation about why design decisions were made

# You now understand the class better than most people who still work there.
```

---

## 16. Mistakes → Root Cause → Fix → Prevention

---

### Mistake 1: Blaming Without `-w` in Reformatted Repos

**Scenario:** You run `git blame src/api.js` and every line shows the same author 
("ESLint bot") with the same date (last week). The reformatting commit touched every 
single line.

**Root Cause:** A linting/formatting commit changed every line (whitespace, semicolons, 
quotes). `git blame` correctly attributes each line to the last commit that changed it — 
which is the formatter.

**Fix:**
```bash
# Option A: Use -w to ignore whitespace
git blame -w src/api.js

# Option B: Use --ignore-rev to skip the formatting commit
git blame --ignore-rev <formatter-commit-sha> src/api.js

# Option C: Use the team's .git-blame-ignore-revs file
git blame --ignore-revs-file .git-blame-ignore-revs src/api.js
```

**Prevention:**
```bash
# When you do a formatting commit, add it to .git-blame-ignore-revs:
git commit -m "chore: run prettier on entire codebase"
echo "$(git rev-parse HEAD) # prettier reformat $(date +%Y-%m-%d)" >> .git-blame-ignore-revs
git add .git-blame-ignore-revs
git commit -m "chore: add prettier commit to blame ignore list"

# And configure the file for all devs:
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

---

### Mistake 2: Using `git log <file>` Without `--` When a Branch Has the Same Name

**Scenario:** You run `git log auth` and get unexpected output. A branch called `auth` 
exists AND a file called `auth` exists. Git interprets `auth` as a branch name.

**Root Cause:** Without `--`, Git tries to resolve the argument as both a revision and 
a file path. Branch names win over file paths.

**Broken state:**
```bash
git log auth
# Shows commits on 'auth' branch, NOT commits that touched a file called 'auth'
```

**Fix:**
```bash
# Always use -- to separate revision from path
git log -- auth        # commits that touched file 'auth'
git log auth --        # commits reachable from branch 'auth'
git log -- src/auth.js # always unambiguous
```

**Prevention:** Make `--` a habit whenever specifying paths in git commands.

---

### Mistake 3: `-S` Missing the Commit Because the Count Didn't Change

**Scenario:** You run `git log -S "processPayment"` to find a commit that moved the 
function to a new file. The output is empty even though the commit definitely exists.

**Root Cause:** The function was moved: removed from `payments.js` AND added to 
`processor.js` in the SAME commit. The total count of "processPayment" in the entire 
codebase didn't change (-1 in one file, +1 in another). `-S` looks at count change 
and sees: count before commit = 1, count after commit = 1. No change → commit not shown.

**Fix:**
```bash
# Use -G instead: searches for changed LINES matching the pattern
git log -G "processPayment" --oneline
# This finds the commit because lines were both added and removed containing the string

# Or search for the rename event explicitly
git log --diff-filter=R --name-status | grep "processPayment\|processor"
```

**Prevention:** Know the fundamental difference: `-S` = count change, `-G` = line change. 
For moves (same count, different files), always use `-G`.

---

### Mistake 4: `--follow` Not Following Through All Renames

**Scenario:** `git log --follow` stops in the middle of history — it doesn't trace back 
far enough through multiple renames.

**Root Cause:** `--follow` can only follow ONE rename at a time. If a file was renamed 
multiple times (A→B→C), `--follow` starting at C might only follow back to B, not all 
the way to A.

**Technical detail:** `--follow` also only works with a single file (no wildcards, no 
directories).

**Fix:**
```bash
# Lower the similarity threshold (might pick up more rename detections)
git log --follow -M10% -- src/current/name.js

# Manual multi-hop following:
git log --diff-filter=R --name-status | grep "name.js"
# Find each rename in history, then trace back through each old name manually:
git log --follow -- src/old/name.js
```

**Prevention:** If your repo has complex rename chains, document them in a MIGRATIONS.md 
or use a more persistent file name convention (UUID-based file names or never rename files).

---

### Mistake 5: Slow `git blame` on Large Files with Long History

**Scenario:** `git blame` on a 3000-line file with 5 years of history takes 30+ seconds.

**Root Cause:** Git must reconstruct the file content at every commit in the path from 
HEAD to the initial commit (potentially thousands of commits), applying delta chains from 
packfiles. For large files with many small changes, this is extremely I/O intensive.

**Fix:**
```bash
# Option A: Blame only the specific lines you care about
git blame -L 100,200 src/large-file.js
# Much faster — only needs to track 100 lines

# Option B: Limit history depth
git blame --since="1 year ago" src/large-file.js
# Only traces back 1 year

# Option C: Use a specific commit as the root
git blame <recent-sha> -- src/large-file.js
# Blame from a recent commit instead of going all the way back

# Option D: Re-pack the repository (improves delta chains)
git repack -a -d --depth=50 --window=250
# Better pack organization = faster object access = faster blame
```

**Prevention:** For files that are critically important and frequently accessed with 
blame, consider periodically running `git gc` to ensure optimal packfile structure.

---

## 17. Mental Model Checkpoint — 10 Questions

**Q1.** What does `git blame -w` do differently from regular `git blame`? Give a concrete 
example of when you must use it.

**Q2.** What is the fundamental difference between `git log -S "string"` and 
`git log -G "string"`? Give a scenario where `-S` would show nothing but `-G` would 
show the correct commit.

**Q3.** You want to find every commit that ever touched any file in the `src/payments/` 
directory, authored by Alice, since the `v3.0.0` tag. Write the exact command.

**Q4.** What does `--follow` do in `git log --follow -- src/api/users.js`? What is its 
limitation? What does Git use to detect renames?

**Q5.** What does the `^` (caret) prefix mean on a commit SHA in `git blame` output?

**Q6.** You run `git blame src/server.js` and find that line 45 was last changed in 
commit `abc1234`. You want to know: (a) the full commit message, (b) all files changed 
in that commit, (c) the content of line 45 three commits BEFORE `abc1234`. Write a command 
for each.

**Q7.** What does `git shortlog -sn` show and when would you use it?

**Q8.** You need to find when a function called `legacyAuth()` was removed from the 
codebase. Write the command and explain what each part does.

**Q9.** What is `.git-blame-ignore-revs` and how do you configure Git to use it 
automatically for all developers on the team?

**Q10.** Write a `git log` command that outputs CSV format: date, short-SHA, author name, 
commit subject — for all commits in the last 3 months that touched `src/auth/`.

---

**Answers:**

**A1.** `-w` makes `git blame` ignore whitespace-only changes (indentation, blank lines, 
trailing spaces). Without it, a reformatting commit (Prettier, ESLint) claims authorship 
of every line it touched even though the logic is unchanged. Use `-w` whenever the repo 
has had mass-formatting commits that would otherwise pollute blame output.

**A2.** `-S "string"` fires when the **count** of the exact string changes (added or removed from the codebase). `-G "pattern"` fires when any **changed line** in the diff matches the regex. Scenario: a function `processPayment` is moved from `payments.js` to `processor.js` in a single commit. `-S` sees: count before=1, count after=1 → no change → commit NOT shown. `-G` sees: the line `function processPayment()` was removed in one file AND added in another → commit IS shown.

**A3.** `git log --author="Alice" --since="v3.0.0" -- src/payments/`
(Note: `--since` can take a tag name as the ref point in some Git versions; for portability: `git log --author="Alice" v3.0.0..HEAD -- src/payments/`)

**A4.** `--follow` makes `git log` continue tracking a file's history even when the file was renamed. Without it, history stops at the rename commit. Limitation: only works with single files (not directories, not patterns), and only follows one rename hop at a time (multiple renames may not be fully traced). Git detects renames by finding commits where a file was "deleted" and a similar file was "added" in the same commit — similarity is measured by content (default threshold: 50%, configurable with `-M<percent>`).

**A5.** The `^` prefix marks a **boundary commit** — the oldest commit in the blame range that Git stopped at. By default, boundary = the initial commit (can't go further back). With `--since` or when specifying a range, boundary = the first commit in the requested range. It's a visual indicator that Git cannot attribute the line to anything earlier.

**A6.** 
- (a) Full commit message: `git log --format="%B" -1 abc1234`
- (b) All files changed: `git show --stat abc1234`  
- (c) Content of line 45 three commits before: `git show abc1234~3:src/server.js | sed -n '45p'`

**A7.** `git shortlog -sn` shows a sorted list of commit counts per author, with the most-prolific author first. `-s` = summary (no commit messages, just counts), `-n` = sort by number (descending). Use it for: quick contribution stats, onboarding (who are the main committers?), release notes (who contributed to this release?).

**A8.** `git log -S "legacyAuth()" --oneline`
- `git log` — walk commit history
- `-S "legacyAuth()"` — find commits where the count of this exact string changed
- `--oneline` — compact output
The last commit shown (if multiple) is the removal commit. If you add `--reverse`, the first is when it was added and the last is when it was removed.

**A9.** `.git-blame-ignore-revs` is a plain text file (typically at repo root) listing commit SHAs that `git blame` should skip — attributing lines instead to the previous authors of those lines. It's used to skip mass-formatting commits. To configure for all developers: `git config blame.ignoreRevsFile .git-blame-ignore-revs` — this goes in the repo's local `.git/config`, but since it's per-repo not per-user, you also need each developer to run this command or add it to repo onboarding scripts. Note: the file path is relative to the repo root.

**A10.** `git log --since="3 months ago" --format="%as,%h,%an,%s" -- src/auth/`

---

## 18. Connect Everything Backwards

### Topic 02 — Git Objects

`git blame` and `git log` are both read-only traversals of the object store from Topic 02. 
`git log` walks commit objects following parent pointers. `git blame` reads blob objects 
(the file content at each commit) and diffs them. The SHA shown in `git blame` output is 
the commit object SHA. The `--stat` in `git log` counts insertions/deletions by diffing 
tree objects between parent and child commits.

```
Topic 02: commit objects have tree + parent pointers; blob objects store file content
Topic 22: git log walks commit→parent chains; git blame diffs blob objects at each commit
```

### Topic 03 — The DAG and Refs

`git log v1.0.0..HEAD` uses the reachability concept from Topic 03 — it shows commits 
reachable from HEAD but not from `v1.0.0`. The `..` and `...` range operators are pure 
DAG operations. `git log --all` starts from ALL refs (all branch tips and tags), not 
just HEAD.

```
Topic 03: reachability determines which commits are "visible" from a ref
Topic 22: git log revision ranges (v1..v2, A...B) are DAG reachability queries
```

### Topic 06 — git add & git commit

The commit message (Topic 07) is what appears in `git log --oneline` and `git log --grep`. 
The quality of commit messages written in Topic 07 directly determines how useful 
`git log --grep` is. A commit with message "stuff" cannot be found by `git log --grep`. 
A commit with "feat(auth): implement JWT refresh token rotation (AUTH-234)" can be found 
by grep on the feature, the component, the conventional type, or the ticket number.

```
Topic 07: commit messages are the "why" of every change
Topic 22: git log --grep, --format="%s", --format="%B" all depend on message quality
```

### Topic 11 — Rebasing

Rebased histories have clean, linear commit sequences — every `git log` output is 
unambiguous and every `git blame` result points to a meaningful, atomic commit. 
Squash-merged PRs mean `git log -S "function"` reliably finds the PR that introduced 
the feature (not an intermediate "WIP" commit). Non-rebased, merge-heavy histories 
complicate `git log --graph` output and make `git blame` results point to merge commits.

```
Topic 11: rebase creates clean linear history with atomic commits
Topic 22: git log and git blame are much more useful on rebased histories
```

### Topic 20 — Reflog

`git log -g` (walk reflogs) is the bridge between Topics 20 and 22. It uses the same 
`git log` format/filter system but walks the reflog instead of the commit DAG. This 
means all `--format`, `--grep`, `-S`, `-G` flags work on reflog entries too.

```
Topic 20: reflog tracks HEAD movements
Topic 22: git log -g applies all git log flags to the reflog instead of the DAG
```

---

## 19. Quick Reference Card

### git blame

| Command | What It Does |
|---------|-------------|
| `git blame <file>` | Show line-by-line authorship of a file |
| `git blame -L 10,20 <file>` | Blame only lines 10-20 |
| `git blame -L /regex/ <file>` | Blame from first line matching regex |
| `git blame -w <file>` | Ignore whitespace changes |
| `git blame -M <file>` | Detect lines moved within the same file |
| `git blame -C <file>` | Detect lines copied from other files |
| `git blame -e <file>` | Show emails instead of names |
| `git blame --since=6.months.ago <file>` | Only show blame since date |
| `git blame --ignore-rev <sha> <file>` | Skip one commit when blaming |
| `git blame --ignore-revs-file .git-blame-ignore-revs <file>` | Skip all reformatting commits |
| `git blame <rev> -- <file>` | Blame file as it was at a specific revision |

### git log — Output Format

| Command | What It Does |
|---------|-------------|
| `git log --oneline` | One line per commit |
| `git log --oneline --graph --all` | Graph view of all branches |
| `git log --stat` | Files changed + insertions/deletions per commit |
| `git log --name-only` | File names changed per commit |
| `git log --name-status` | File names + change type (M/A/D/R) |
| `git log -p` | Full diff for every commit |
| `git log -p -U5` | Full diff with 5 lines of context |
| `git log --pretty=format:"%as,%h,%an,%s"` | CSV output |
| `git log --date=short` | YYYY-MM-DD dates |
| `git log --date=relative` | "2 weeks ago" dates |

### git log — Filtering

| Command | What It Does |
|---------|-------------|
| `git log --author="Alice"` | Commits by Alice |
| `git log --since="2 weeks ago"` | Commits from last 2 weeks |
| `git log --until="2026-01-01"` | Commits before a date |
| `git log --grep="pattern"` | Commits with message matching pattern |
| `git log --grep="A" --grep="B" --all-match` | Message matches both A AND B |
| `git log -S "string"` | Commits where count of string changed |
| `git log -G "regex"` | Commits where changed lines match regex |
| `git log --follow -- <file>` | History of file across renames |
| `git log -- <path>` | Commits that touched this path |
| `git log v1.0..v2.0` | Commits between two refs |
| `git log main..feature` | Commits on feature not on main |
| `git log @{upstream}..HEAD` | Unpushed commits |
| `git log --diff-filter=A --name-only` | Only show added files |

### git shortlog

| Command | What It Does |
|---------|-------------|
| `git shortlog` | Commits grouped by author with messages |
| `git shortlog -sn` | Commit count per author, sorted |
| `git shortlog -sn --since="3 months ago"` | Recent commit counts |
| `git shortlog -sn v1.0..v2.0` | Contributions per release |

### Investigation Workflow

```bash
# 1. Find suspicious line
git blame -w src/file.js | grep "something"

# 2. Show full context of that commit
git show <sha>

# 3. Track the string's lifecycle
git log -S "string" --oneline --reverse

# 4. Find who touched a file across renames
git log --follow --format="%an" -- src/file.js | sort | uniq -c | sort -rn

# 5. Security audit: was a secret ever in the code?
git log -S "SECRET_PATTERN" --all --oneline
```

---

## 20. When Would I Use This At Work?

### Scenario 1 — The Mysterious Bug: "This Never Worked"

**Context:** A customer reports that the discount system has been applying 10% too much 
for 6 months. You're the new engineer on the team. You need to trace the root cause.

```bash
# Find the discount calculation code
grep -rn "applyDiscount\|calculateDiscount" src/

# Blame the core discount function
git blame -w src/pricing/discounts.js | grep "discountRate\|0.10\|10%"
# Shows: line 67 last changed by "dev@company.io" 6 months ago

# Get the full context
git show <sha>
# Commit message: "fix: apply loyalty discount on top of seasonal discount"
# Diff shows: rate was changed from 0.0 to 0.10 in the multiplication
# AH: the fix accidentally added a permanent 10% instead of conditional

# Confirm the timeline
git log -S "discountRate = 0.10" --oneline --reverse
# Exact commit and date confirmed: 6 months ago
```

---

### Scenario 2 — Sprint Retrospective: "Who's Been Working on What?"

**Context:** You're the tech lead preparing for the quarterly review. You need to 
show what each engineer contributed.

```bash
# Summary for the quarter
git shortlog -sn --since="2026-02-01" --until="2026-05-01"
#   47  Alice Johnson
#   32  Bob Smith
#   28  Carol Wei
#   15  David Lee

# Drill down: what did each person work on?
git log --author="Alice Johnson" --since="2026-02-01" \
  --format="- %s" --reverse

# Per-module breakdown:
git log --author="Alice" --since="2026-02-01" --name-only --format="" |
  grep "^src/" | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn
# Shows which directories Alice touched most
```

---

### Scenario 3 — Compliance: "Show Me Every Change to Production Config"

**Context:** An auditor needs every change to deployment and config files in the last year.

```bash
git log --since="2025-05-28" \
  --format="%as | %h | %an <%ae> | %s" \
  -- config/ deploy/ .env.example docker-compose.yml kubernetes/ \
  > config-changes-audit.txt

# With full diffs for the auditor:
git log --since="2025-05-28" -p \
  -- config/ deploy/ \
  > config-changes-with-diffs.txt

# Count of changes per person:
git shortlog -sn --since="2025-05-28" -- config/ deploy/
```

---

### Scenario 4 — Codebase Migration Planning

**Context:** You need to migrate the authentication system. You want to know exactly 
which files you'll need to change and who has the deepest knowledge of each.

```bash
# Find all auth-related files
git log --name-only --grep="auth\|login\|token\|session" --since="2024-01-01" |
  grep "^src/" | sort | uniq -c | sort -rn | head -20
# Shows the most frequently changed auth files

# For each critical file, find its primary owner
git log --follow --format="%an" -- src/auth/middleware.js |
  sort | uniq -c | sort -rn | head -3
# Alice Johnson (67) Bob Smith (23) Carol Wei (8)
# → Alice is the expert; loop her into the migration planning

# Find files that are "orphaned" (primary author left the company)
git log --follow --format="%ae" -- src/legacy/oldAuth.js |
  head -1
# former-dev@company.io — check if this person still works here
```

---

### Scenario 5 — Incident Post-Mortem: "Build the Timeline"

**Context:** A production incident at 11pm. You need to build a complete timeline 
for the post-mortem document.

```bash
# All commits in the last 48 hours (when the incident window started)
git log --since="48 hours ago" --format="%as %at | %h | %an | %s" --reverse

# All commits to the affected service
git log --since="48 hours ago" --reverse \
  --format="%as %H | %an | %s" \
  -- src/payments/ > incident-timeline.txt

# Show the full diff of the deploy that caused it
# (assuming you know the commit SHA from the incident alert)
git show --stat abc1234
git show abc1234

# See who reviewed and merged the PR
git log --format="%an <%ae>%n  Committer: %cn <%ce>" abc1234^..abc1234
# Author vs Committer distinction shows who wrote vs who merged
```

---

*Topic 22 Complete — You can now investigate any codebase like a forensic expert.*

**Next: Topic 23 — Submodules & Monorepos: Large-Scale Project Organization**
*"When one repo isn't enough — how Git manages dependencies between repositories."*
