# Topic 06 — `git add` & `git commit` — Exact Internal Mechanics
### The Complete Step-by-Step of What Happens in `.git/` Every Time

---

## Connection to Previous Topics

Before we begin, ground yourself:

- **Topic 01** showed you every file inside `.git/` — you know `objects/`, `refs/heads/`,
  `index`, `HEAD`. Today those files move.
- **Topic 02** taught you the 4 object types — blob, tree, commit, tag. Today you will watch
  blobs and trees and commits get physically created, one by one.
- **Topic 03** showed you that a branch is just a 41-byte file holding a SHA. Today you will
  watch that 41-byte file get overwritten to point to the brand new commit.
- **Topic 05** defined the Three Trees (Working Directory → Index → HEAD). Today is the
  **mechanical deep-dive** into how each tree transition actually works at the file level.

If any of that feels fuzzy, re-read those topics first. This topic is all mechanics — no
analogies can replace a solid foundation.

---

## RULE 1 — ELI5 Anchor

### The Photographer Analogy

Imagine you are a photographer cataloguing a museum's art collection.

**Your notebook** = the working directory. You walk around, make new sketches, write notes,
cross things out. It's messy and in-progress.

**The official submission form** = the staging area (Index). When you decide a sketch is
ready, you fill it into the form — you write the item's ID code (the SHA), its category
(file mode), and its name. You can stage multiple items at once on the same form. The form
represents exactly "what the catalogue will contain."

**The published catalogue entry** = the commit. When you hand in the form, the archive
officer:
1. Looks at your form
2. Physically photographs every item you listed (creates objects)
3. Groups photos into sections (tree objects per directory)
4. Writes a cover page for this submission: "Submitted by X, on date Y, following entry Z"
   (that's the commit object)
5. Stamps the cover page with a unique ID and updates the master index (moves the branch
   pointer to point at this new cover page)

`git add` = filling in the submission form  
`git commit` = handing the form to the archive officer who does steps 1–5

---

## RULE 2 — Physical Architecture First

### The Baseline: A New Repository

Let's start from absolute zero — a brand-new, freshly initialized repository with no commits.

```
$ mkdir demo && cd demo
$ git init

.git/
├── HEAD                   ← contains: "ref: refs/heads/main\n"
├── config                 ← local repo config
├── description            ← used by GitWeb, ignore for now
├── info/
│   └── exclude            ← local .gitignore (not tracked)
├── hooks/                 ← sample hook scripts (all disabled)
│   ├── pre-commit.sample
│   └── ...
├── objects/
│   ├── info/              ← empty
│   └── pack/              ← empty
└── refs/
    ├── heads/             ← EMPTY — no branches yet
    └── tags/              ← empty
```

**Key observations about this state:**
- `objects/` has NO files in it. The database is empty.
- `refs/heads/` is EMPTY. There is no `main` file yet.
- `HEAD` points to `refs/heads/main` but that file doesn't exist yet.
- There is NO `index` file. It gets created on the first `git add`.

```bash
# PROVE IT: Verify all of this right now
ls -la .git/
ls -la .git/objects/
ls -la .git/refs/heads/
cat .git/HEAD
# Expected: ref: refs/heads/main
```

---

### State After Creating a File (Before `git add`)

```
$ echo "console.log('hello');" > server.js
```

The `.git/` directory is **completely unchanged**. Git knows nothing about `server.js` yet.

```bash
# PROVE IT: Nothing changed in .git/
find .git -type f | sort
# Shows same list as before — no new objects, no index file
```

---

## RULE 3 — Exact Data Flow: `git add server.js`

### The Full Step-by-Step Trace

```
COMMAND: git add server.js

STEP 1 — Read the file from working directory
  READS:    server.js
  CONTENT:  "console.log('hello');\n"   (22 bytes)

STEP 2 — Compute the blob SHA-1
  COMPUTES: header = "blob " + "22" + "\0"   = "blob 22\0"
            full   = header + file_contents
                   = "blob 22\0console.log('hello');\n"
            SHA-1  = sha1(full)
                   = "b6fc4c620b67d95f953a5c1c1230aaab5db5a1b0"
            (Your SHA will differ — it's based on exact bytes of your file)

STEP 3 — Check if object already exists
  CHECKS:   Does .git/objects/b6/fc4c620b67d95f953a5c1c1230aaab5db5a1b0 exist?
  IF YES:   Skip writing (deduplication! From Topic 02 — blobs are content-addressed)
  IF NO:    Proceed to write

STEP 4 — Write the blob object
  COMPUTES: zlib_compress("blob 22\0console.log('hello');\n")
  CREATES:  Directory .git/objects/b6/
  WRITES:   .git/objects/b6/fc4c620b67d95f953a5c1c1230aaab5db5a1b0
            (binary, zlib-compressed)

STEP 5 — Update the Index
  READS:    .git/index  (if it exists; creates it if first add)
  STAT:     Reads filesystem metadata for server.js:
              - ctime (inode change time)
              - mtime (modification time)
              - dev   (device number)
              - ino   (inode number)
              - mode  (100644 for regular file, 100755 for executable)
              - uid   (user ID)
              - gid   (group ID)
              - size  (22 bytes)
  WRITES:   .git/index  with new/updated entry:
              100644 b6fc4c620b67d95f953a5c1c1230aaab5db5a1b0 0\tserver.js
  (The "0" is the stage number — 0 means no conflict, normal state)
```

**The `.git/` directory AFTER `git add server.js`:**

```
.git/
├── HEAD
├── config
├── description
├── index                   ← CREATED (was not here before)
├── info/
│   └── exclude
├── hooks/ ...
├── objects/
│   ├── b6/
│   │   └── fc4c620b67d95f953a5c1c1230aaab5db5a1b0  ← NEW blob object
│   ├── info/
│   └── pack/
└── refs/
    ├── heads/              ← still EMPTY (no commit = no branch file)
    └── tags/
```

```bash
# PROVE IT: Verify the blob was created
git ls-files --stage
# Expected: 100644 b6fc4c... 0    server.js

# Read the blob content
git cat-file -p b6fc4c
# Expected: console.log('hello');

# Confirm the type
git cat-file -t b6fc4c
# Expected: blob

# See the index in human-readable form
git ls-files --stage --debug
# Shows full stat data per entry
```

---

### What Happens When You `git add` a SECOND File

```
$ echo "body { margin: 0; }" > styles.css
$ git add styles.css
```

```
STEP 1 — Read styles.css (20 bytes)
STEP 2 — Compute SHA: "blob 20\0body { margin: 0; }\n" → different SHA
STEP 3 — Write blob .git/objects/XX/YY...
STEP 4 — Update index: ADD entry for styles.css
         Index now has TWO entries:
           100644 <sha-of-server.js>  0    server.js
           100644 <sha-of-styles.css> 0    styles.css
         (Index is always sorted by filename path)
```

---

### What Happens When You `git add` a MODIFIED File

```
$ echo "console.log('world');" > server.js   # Modify the file
$ git add server.js
```

```
STEP 1 — Read server.js (new content, 22 bytes)
STEP 2 — Compute NEW SHA for new content → completely different hash
STEP 3 — Write NEW blob object
STEP 4 — UPDATE existing index entry for server.js:
         OLD entry: 100644 <old-sha>  0    server.js
         NEW entry: 100644 <new-sha>  0    server.js

CRITICAL: The OLD blob is NOT deleted!
  .git/objects/ now has BOTH blobs:
    - old blob (b6fc4c...) → still there (unreferenced for now)
    - new blob (new-sha...) → the staged version
```

This is why `git stash`, `git reset`, and `git reflog` can recover "lost" work — old blobs
persist in the object database until garbage collection runs.

---

### What Happens When You `git add` a DELETED File

```
$ rm server.js
$ git add server.js
```

```
READS:    server.js — file does NOT exist on disk
COMPUTES: Nothing (no content to hash)
MODIFIES: .git/index — REMOVES the entry for server.js
          The blob object in .git/objects/ is NOT deleted
          (it stays until GC — see Topic 25)
```

Alternatively: `git rm server.js` = delete from disk AND stage that deletion in one step.

---

## RULE 4 — ASCII Architecture Diagram: `git add` Flow

```
Working Directory                 Index (.git/index)            Object DB (.git/objects/)
─────────────────                 ──────────────────            ──────────────────────────

server.js                         (empty before add)            (empty before add)
"console.log('hello');"
        │
        │  git add server.js
        │
        ▼
1. Read bytes of server.js ──────────────────────────────────► 3. Write blob object
   "console.log('hello');\n"                                      .git/objects/b6/fc4c...
                                                                    ▲
2. Compute SHA-1 of                                                 │ zlib-compressed
   "blob 22\0" + content                                            │ ("blob 22\0content")
   → b6fc4c620b...                                                  │
        │                                                           │
        └──────────────► 4. Write index entry ◄──────────────────┘
                            100644 b6fc4c... 0  server.js
                            (mode, sha, stage, path)
```

---

## RULE 3 (continued) — Exact Data Flow: `git commit -m "Init"`

### Pre-conditions

Index currently contains:
```
100644 <sha-styles> 0    styles.css
100644 <sha-server> 0    server.js
```

### Full Step-by-Step Trace

```
COMMAND: git commit -m "Init"

STEP 1 — Read the index
  READS:    .git/index
  FINDS:    2 entries: styles.css (sha-S), server.js (sha-V)

STEP 2 — Build tree object(s)
  (This is what `git write-tree` does internally)

  All files in this repo are in the ROOT directory.
  Git builds ONE tree object representing the root:

  TREE content (raw, before compression):
    "tree 72\0"
    "100644 server.js\0" + <20 raw bytes of sha-V>
    "100644 styles.css\0" + <20 raw bytes of sha-S>

  Wait — how does "72" get computed?
    Each entry = mode + space + filename + null-byte + 20-byte-SHA
    "100644 server.js\0" = 6+1+9+1 = 17 bytes, plus 20 SHA bytes = 37 bytes
    "100644 styles.css\0" = 6+1+10+1 = 18 bytes, plus 20 SHA bytes = 38 bytes
    Total content = 37 + 38 = 75 bytes... (exact number depends on your filenames)

  COMPUTES: SHA-1 of the tree content → tree_sha = "abc123..."
  WRITES:   .git/objects/ab/c123...  (the root tree object)

STEP 3 — Build commit object
  Git reads:
    - tree_sha (just computed)
    - parent SHA (NONE — this is the first commit!)
    - Author from config (user.name, user.email)
    - Committer from config (same for now)
    - Timestamps (current time, timezone)
    - Message: "Init"

  Commit content (raw, human-readable form):
    "tree abc123...\n"
    "author John Doe <john@example.com> 1715894400 +0530\n"
    "committer John Doe <john@example.com> 1715894400 +0530\n"
    "\n"
    "Init\n"

  COMPUTES: header = "commit " + content_length + "\0"
            full = header + content
            SHA-1 of full → commit_sha = "def456..."

  WRITES:   .git/objects/de/f456...

STEP 4 — Write COMMIT_EDITMSG
  WRITES:   .git/COMMIT_EDITMSG   ← contains the commit message text
            Content: "Init\n"
            (This file persists — it's the "last commit message" used for --reuse-message)

STEP 5 — Update the branch pointer
  READS:    .git/HEAD             → "ref: refs/heads/main\n"
  READS:    .git/refs/heads/main  → does NOT EXIST (first commit)
  WRITES:   .git/refs/heads/main  → "def456...\n"  (the commit SHA, 40 chars + newline)

STEP 6 — Update index TREE cache extension (optional optimization)
  MODIFIES: .git/index — updates the cached tree SHA for root directory
            This caches the fact that root tree = abc123...
            So next `git status` doesn't need to recompute tree SHA
```

**The `.git/` directory AFTER `git commit`:**

```
.git/
├── HEAD                           ← unchanged: "ref: refs/heads/main\n"
├── COMMIT_EDITMSG                 ← NEW: "Init\n"
├── config
├── description
├── index                          ← MODIFIED: TREE cache extension updated
├── info/
│   └── exclude
├── hooks/ ...
├── objects/
│   ├── ab/
│   │   └── c123...                ← NEW: root tree object
│   ├── b6/
│   │   └── fc4c...                ← server.js blob (from git add)
│   ├── XX/
│   │   └── YY...                  ← styles.css blob (from git add)
│   ├── de/
│   │   └── f456...                ← NEW: commit object
│   ├── info/
│   └── pack/
└── refs/
    ├── heads/
    │   └── main                   ← NEW: "def456...\n"
    └── tags/
```

```bash
# PROVE IT: Verify the commit structure
git log --oneline
# Shows: def456 Init

# Read the commit object
git cat-file -p HEAD
# Shows: tree, author, committer, message

# Read the tree object
git cat-file -p HEAD^{tree}
# Shows: blob entries for server.js and styles.css

# Verify the branch file
cat .git/refs/heads/main
# Shows the commit SHA

# Verify HEAD still points to main (not the SHA directly)
cat .git/HEAD
# Shows: ref: refs/heads/main
```

---

## RULE 4 — ASCII Architecture: Full `git add` + `git commit` Pipeline

```
                     git add                          git commit
                  ──────────►                       ──────────►
Working Dir         Index               Tree Objects    Commit Object   Branch Ref
───────────         ─────               ────────────    ─────────────   ──────────
server.js ──sha──► [100644              root tree ◄─┐   tree: abc123    main
styles.css──sha──►  sha-V  server.js]    ├─server.js │   parent: none   ↓
                   [100644              └─styles.css │   author: ...    def456
                    sha-S  styles.css]               │   message: Init
                                                     │        │
                                  abc123 ────────────┘        │
                                                     def456 ◄─┘
                                                        │
                                                        ▼
                                               .git/refs/heads/main
                                               (41-byte file: "def456...\n")
```

**Reading the diagram:**
- `git add` moves content from Working Dir → Index (creating blob objects as a side effect)
- `git commit` reads the Index → builds tree objects → creates commit object → updates branch ref
- HEAD still points at the branch NAME, not the commit SHA directly

---

## RULE 2 — Multi-Level Directory Trees

Now let's see what happens when your files are in subdirectories. This is where trees
get interesting.

```
project/
├── src/
│   ├── server.js
│   └── utils.js
├── test/
│   └── server.test.js
└── README.md
```

After `git add -A` and `git commit -m "First"`:

```
STEP 2 — Build trees (bottom-up, leaves first)

  Git builds tree for src/:
    Tree content: blob sha-S server.js + blob sha-U utils.js
    Writes: .git/objects/XX/YY...  (src tree)

  Git builds tree for test/:
    Tree content: blob sha-T server.test.js
    Writes: .git/objects/AA/BB...  (test tree)

  Git builds tree for root/:
    Tree content:
      040000 src\0  + <20 bytes of src tree SHA>
      040000 test\0 + <20 bytes of test tree SHA>
      100644 README.md\0 + <20 bytes of README blob SHA>
    Writes: .git/objects/CC/DD...  (root tree)

TOTAL OBJECTS CREATED by this commit:
  3 blobs (server.js, utils.js, server.test.js, README.md) = 4 blobs
  3 trees (src/, test/, root) = 3 trees
  1 commit
  Total: 8 new objects in .git/objects/
```

**Critical insight — TREE REUSE:**

If on the NEXT commit you only modify `src/server.js`:

```
STEP 2 — Build trees for second commit

  Git builds NEW tree for src/ (content changed)
  Git REUSES test/ tree (nothing in test/ changed — same SHA)
  Git builds NEW root tree (points to new src/, same test/ SHA)

TOTAL OBJECTS FOR SECOND COMMIT:
  1 new blob (new version of server.js)
  1 new src/ tree
  1 new root tree
  1 commit
  Total: 4 new objects — the test/ tree is SHARED with the first commit
```

This is the **structural sharing** from Topic 02 — trees that didn't change are NOT
duplicated. This is how Git achieves efficient storage.

```bash
# PROVE IT: See tree reuse
git cat-file -p HEAD^{tree}
# Note the SHA of src/ tree → call it SHA-SRC1

# Make a change only to README.md
echo "Updated" >> README.md
git add README.md
git commit -m "Update README"

git cat-file -p HEAD^{tree}
# Note the SHA of src/ tree → SHA-SRC2
# SHA-SRC1 == SHA-SRC2  ← same tree, reused, not duplicated
```

---

## RULE 6 — Deep Syntax Breakdown

### `git add`

```
git add [<pathspec>...]  [--verbose]  [-n]  [-f]  [-i]  [-p]  [-e]
│   │    │               │            │    │     │    │    │
│   │    │               │            │    │     │    │    └─ --edit: open patch
│   │    │               │            │    │     │    │        in editor
│   │    │               │            │    │     │    └─ --patch / -p:
│   │    │               │            │    │     │        interactive hunk staging
│   │    │               │            │    │     └─ --interactive / -i:
│   │    │               │            │    │        full interactive menu
│   │    │               │            │    └─ --force / -f:
│   │    │               │            │        add ignored files (bypasses .gitignore)
│   │    │               │            └─ --dry-run / -n:
│   │    │               │                show what WOULD be staged, don't do it
│   │    │               └─ --verbose / -v:
│   │    │                   print one line per file added
│   │    └─ the file(s) or glob or directory to add
│   └─ subcommand: add to the index
└─ git binary
```

**Pathspec variants:**
```bash
git add server.js              # exactly one file
git add src/                   # entire directory (recursively)
git add "*.js"                 # all .js files (quote to prevent shell expansion)
git add .                      # everything under current directory
                               # (includes new + modified + deleted if in CWD)
git add -A                     # all changes everywhere in the repo
                               # (includes deleted files anywhere)
git add -u                     # only TRACKED files (modified + deleted)
                               # does NOT add untracked new files
```

**The critical difference between `.`, `-A`, and `-u`:**

| Variant | New files | Modified files | Deleted files | Scope |
|---------|-----------|----------------|---------------|-------|
| `git add .` | ✅ | ✅ | ✅ (Git 2.x) | Current directory + subdirs |
| `git add -A` | ✅ | ✅ | ✅ | Entire repository |
| `git add -u` | ❌ | ✅ | ✅ | Entire repository |
| `git add <file>` | ✅ | ✅ | ✅ | Exact file only |

> **Historical note**: In Git 1.x, `git add .` did NOT stage deletions. Git 2.x fixed this.
> If you're reading old Stack Overflow answers, this is the source of confusion.

---

### `git commit`

```
git commit  [-m <msg>]  [-a]  [--amend]  [--no-edit]  [-v]  [--allow-empty]
│    │       │           │    │           │            │    │
│    │       │           │    │           │            │    └─ --allow-empty:
│    │       │           │    │           │            │        commit even if
│    │       │           │    │           │            │        index == HEAD tree
│    │       │           │    │           │            └─ -v / --verbose:
│    │       │           │    │           │                include diff of staged
│    │       │           │    │           │                changes in editor
│    │       │           │    │           └─ --no-edit:
│    │       │           │    │               use current commit message as-is
│    │       │           │    │               (used with --amend to only update
│    │       │           │    │                commit content, not message)
│    │       │           │    └─ --amend:
│    │       │           │        replace the most recent commit
│    │       │           │        (see full explanation below)
│    │       │           └─ -a / --all:
│    │       │               automatically stage all TRACKED modified files
│    │       │               before committing (skip explicit git add)
│    │       │               Does NOT add untracked files
│    │       └─ -m "message": provide message inline
│    │           If omitted, Git opens your configured editor
│    └─ subcommand
└─ git binary
```

**Other important flags:**
```bash
git commit --fixup=<commit>    # create a "fixup!" commit (for autosquash rebase)
git commit --squash=<commit>   # create a "squash!" commit
git commit --no-verify         # SKIP pre-commit and commit-msg hooks
                               # (dangerous — bypasses quality gates)
git commit --date="2026-01-01 12:00" --date  # override author date
git commit --author="Name <email>"            # override author
git commit --reset-author      # use current config's author on --amend
git commit -C <commit>         # reuse another commit's message + author
git commit -c <commit>         # like -C but open editor to edit message
```

---

## RULE 9 — Byte-Level Internals

### The Index Binary Format (Revisited — Deep Dive)

From Topic 05, you know the Index is a binary file at `.git/index`. Let's go deeper.

**Header (12 bytes):**
```
Bytes 0–3:  "DIRC"        ← Magic bytes (stands for "directory cache")
Bytes 4–7:  0x00000002    ← Version number (2 or 3 or 4)
Bytes 8–11: 0x00000002    ← Number of index entries (e.g., 2 for 2 files)
```

**Each Entry (variable length, minimum 62 bytes):**
```
Bytes 0–3:   ctime seconds  (32-bit big-endian int)
Bytes 4–7:   ctime nanoseconds
Bytes 8–11:  mtime seconds
Bytes 12–15: mtime nanoseconds
Bytes 16–19: dev (device number)
Bytes 20–23: ino (inode number)
Bytes 24–27: mode (e.g., 0x000081A4 = 100644 octal)
Bytes 28–31: uid
Bytes 32–35: gid
Bytes 36–39: file size in bytes
Bytes 40–59: SHA-1 (20 raw bytes — NOT hex, the actual binary SHA)
Bytes 60–61: flags (16 bits)
  Bit 15:    assume-valid flag (git update-index --assume-unchanged)
  Bit 14:    extended flag (if set, 2 more flag bytes follow — version 3+)
  Bits 12–13: stage (00=normal, 01=base, 10=ours, 11=theirs — for merge conflicts)
  Bits 0–11: name length (capped at 0xFFF for long names)
Bytes 62+:   null-terminated file path
Padding:     zero bytes to align to 8-byte boundary

Total entry size = 62 + path_length + null_byte, rounded up to multiple of 8
```

**Trailing checksum (20 bytes):**
```
Last 20 bytes of the index file: SHA-1 of all previous bytes
If this checksum doesn't match, Git refuses to use the index → "index file corrupt"
```

**Extensions (between entries and checksum):**
```
TREE extension ("TREE"):
  Caches computed tree SHAs per directory.
  If you haven't changed src/, Git doesn't recompute the src/ tree SHA — reads from cache.
  Format: path, entry count (ascii), space, subtree count, newline, SHA (20 bytes)

REUC extension ("REUC" — Resolve Undo Conflict):
  Saved when you resolve a merge conflict.
  Stores the 3 versions (base, ours, theirs) so git checkout -- can restore.

untracked extension ("UNTR"):
  Cached list of untracked files for faster `git status`.
```

```bash
# PROVE IT: Look at the raw index
xxd .git/index | head -20
# First 4 bytes: 44 49 52 43 = "DIRC"
# Next 4 bytes: 00 00 00 02 = version 2
# Next 4 bytes: 00 00 00 02 = 2 entries (if you have 2 files staged)
```

---

### The Commit Object Format (Deep)

From Topic 02 you know commits are text objects. Let's see the exact format end to end.

**A typical commit object (RAW, before header):**
```
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
parent 0155eb4229851634a0f03eb265b69f5a2d56f341
author Parsh Khandelwal <parsh@example.com> 1715894400 +0530
committer Parsh Khandelwal <parsh@example.com> 1715894400 +0530

Add user authentication module

Implements JWT-based auth using RS256 algorithm.
Closes #142.
```

**The FULL object as stored on disk (with header):**
```
"commit 281\0"          ← "commit " + content_length + null byte
"tree d8329f...\n"
"parent 015...\n"
"author ...\n"
"committer ...\n"
"\n"
"Add user authentication module\n"
"\n"
"Implements JWT-based auth...\n"
"Closes #142.\n"
```

Then this entire thing is zlib-compressed and written to `.git/objects/`.

**Timestamp format:**
```
1715894400 +0530
│           │
│           └─ UTC offset: +5 hours 30 minutes (India Standard Time)
└─ Unix timestamp: seconds since 1970-01-01 00:00:00 UTC
```

```bash
# PROVE IT: See the raw commit
git cat-file commit HEAD
# Shows the raw commit content (without the header)

# See the header too
printf "commit " && git cat-file commit HEAD | wc -c && echo
# That tells you the content length in the header
```

---

## Deep Dive: `git commit --amend`

### What `--amend` Actually Does

```
COMMAND: git commit --amend -m "New message"

STEP 1 — Read current HEAD commit SHA
  READS:    .git/HEAD → .git/refs/heads/main → "abc123..."

STEP 2 — Read current HEAD commit object
  READS:    .git/objects/ab/c123...
  EXTRACTS: tree SHA (let's say "tree123")
            parent SHA (the commit BEFORE this one)
            author info

STEP 3 — Read the current index
  READS:    .git/index
  (If you staged new changes before --amend, those are included)

STEP 4 — Build new tree (if index changed) OR reuse tree123

STEP 5 — Build NEW commit object
  COMPUTES: new commit with:
              tree: tree123 (or new tree if index changed)
              parent: SAME parent as original commit (grandfather, not the old commit)
              author: SAME as original (preserved!)
              committer: CURRENT time (updated!)
              message: "New message"
  WRITES:   .git/objects/de/f789...   (completely new commit object)

STEP 6 — Update branch pointer
  OVERWRITES: .git/refs/heads/main → "def789...\n"

RESULT:
  OLD commit "abc123" → orphaned (no refs pointing to it)
              BUT still exists in .git/objects/ until GC runs
  NEW commit "def789" → branch now points here
```

**Key insight: `--amend` does NOT edit the old commit — it creates a BRAND NEW commit
and moves the branch pointer. The old commit still exists until garbage collection.**

```bash
# PROVE IT: The old commit still exists after amend
git log --oneline
# Shows new commit SHA

# Read the old commit directly by SHA (get it from reflog)
git reflog
# Find the old SHA (HEAD@{1} or similar)

git cat-file -p <old-sha>
# Git can still read it! It's an orphan, not deleted.
```

**When SHOULD you use `--amend`?**
- ✅ Fix a typo in the commit message
- ✅ Add a file you forgot to include before pushing
- ✅ Fix a small bug in the last commit before sharing
- ❌ NEVER on commits that have already been pushed to a shared branch
  (it rewrites history, causing divergence for all collaborators)

---

## Deep Dive: `git commit -a`

### The "Skip Staging" Shortcut

```
COMMAND: git commit -a -m "Quick fix"

What this ACTUALLY does internally:
  1. Runs git add -u  (stage all modifications and deletions to tracked files)
  2. Then runs git commit with the updated index

It does NOT:
  - Add new untracked files (those still need explicit git add)
  - Respect your staged partial hunks (nukes them — replaces index with -u add)
```

**The danger of `git commit -a`:**

```
Scenario:
  You carefully staged only HALF of server.js using git add --patch.
  You run git commit -a without thinking.

  RESULT: git add -u runs first, staging ALL of server.js.
          Your careful partial staging is LOST.
          The commit now contains everything, not just the half you wanted.
```

```bash
# Example showing the danger
echo "line A" >> server.js
echo "line B" >> server.js
git add -p server.js   # stage only line A

git diff --cached      # staged: only line A
git diff               # unstaged: only line B

git commit -a -m "Oops"
# Both lines A and B end up in the commit — your selective staging was overridden
```

---

## Deep Dive: `git rm` and `git mv`

### `git rm` — Remove a File

```
COMMAND: git rm server.js

STEP 1 — Remove from working directory (delete the actual file)
  DELETES:  server.js  from filesystem

STEP 2 — Remove from index
  MODIFIES: .git/index  — removes the entry for server.js

STEP 3 — On next commit, a tree will be built WITHOUT server.js
          The blob object is NOT deleted (lives in .git/objects/ until GC)
```

**Variants:**
```bash
git rm server.js          # delete file + stage deletion
git rm --cached server.js # remove from index ONLY (keep file on disk)
                          # Use case: accidentally committed a file that should be
                          # in .gitignore — remove it from Git without deleting it
git rm -r dist/           # recursively remove entire directory
git rm -f server.js       # force — remove even if file has staged changes
```

**`git rm --cached` is incredibly important:**
```bash
# You committed node_modules/ by mistake
# (forgot .gitignore before first commit)

# 1. Add to .gitignore
echo "node_modules/" >> .gitignore

# 2. Remove from index (without deleting from disk)
git rm -r --cached node_modules/

# 3. Commit the removal
git commit -m "Remove node_modules from tracking"
# Now node_modules/ is untracked (ignored by .gitignore)
# All those files are STILL on your disk
```

### `git mv` — Rename / Move a File

```
COMMAND: git mv server.js app.js

INTERNALLY, this is equivalent to:
  1. mv server.js app.js           (rename on filesystem)
  2. git rm server.js              (remove old name from index)
  3. git add app.js                (add new name to index)

The resulting index state:
  OLD: 100644 <sha> 0  server.js
  NEW: 100644 <sha> 0  app.js
       ↑ SAME SHA! The blob didn't change — only the name did.
```

**Why Git can detect renames:**
Even if you use plain `mv` instead of `git mv`, Git detects the rename on `git status`
and `git commit` by similarity — if a deleted file and a new file share the same blob
SHA (same content), Git marks it as a rename. This is the basis of `git diff --find-renames`.

```bash
# PROVE IT: Manual mv vs git mv lead to same result
mv README.md NOTES.md
git status
# Shows: renamed: README.md -> NOTES.md   ← Git detected it automatically

# But the internal representation:
git ls-files --stage
# Shows the SHA for NOTES.md is same as README.md's old SHA
```

---

## Deep Dive: How Git Opens the Editor for Commit Messages

When you run `git commit` without `-m`, Git needs to open an editor. Here's the exact
sequence:

```
STEP 1 — Write COMMIT_EDITMSG
  WRITES:   .git/COMMIT_EDITMSG with:
              "\n"                    ← blank line (where you type message)
              "# Please enter the commit message...\n"
              "# On branch main\n"
              "# Changes to be committed:\n"
              "#   modified:   server.js\n"
              "#\n"
              "# Untracked files:\n"
              "#   .env\n"
              "#\n"

STEP 2 — Open editor
  Git runs: $GIT_EDITOR or $VISUAL or $EDITOR or core.editor from config
  Common: "vim .git/COMMIT_EDITMSG" or "code --wait .git/COMMIT_EDITMSG"
  The "--wait" flag for VS Code is critical — without it Git reads the file
  immediately while the editor is still opening (finds empty message → aborts)

STEP 3 — Wait for editor to close
  Git pauses here, waiting for the editor process to exit.

STEP 4 — Read the file back
  READS:    .git/COMMIT_EDITMSG
  Git strips lines starting with "#" (comments)
  Git trims leading/trailing blank lines
  If the result is empty → abort commit ("Aborting commit due to empty message")
  If non-empty → proceed with commit

STEP 5 — COMMIT_EDITMSG stays on disk
  After commit, the file stays as a record of the last message.
  Used by: git commit -C HEAD, git commit --reuse-message
```

**Configuring your editor properly:**
```bash
git config --global core.editor "code --wait"     # VS Code
git config --global core.editor "vim"              # Vim
git config --global core.editor "nano"             # Nano (easiest for beginners)
git config --global core.editor "notepad"          # Windows Notepad

# Verify it works:
git commit   # Should open the editor you configured
```

---

## Deep Dive: The `-v` (verbose) Flag for Commits

```bash
git commit -v
```

When you run this, the COMMIT_EDITMSG file contains not just the template but also the
**full diff** of what you're about to commit:

```
# Please enter the commit message...
# On branch feature/auth
# Changes to be committed:
#   modified:   src/auth.js
#
# ------------------------ >8 ------------------------
# Do not modify or remove the line above.
# Everything below it will be ignored.
diff --git a/src/auth.js b/src/auth.js
index a3b4c5d..e6f7g8h 100644
--- a/src/auth.js
+++ b/src/auth.js
@@ -10,6 +10,9 @@ function login(user) {
   ...
```

The `# >8 ------------------------` line is a scissors line. Everything below it is
stripped from the commit message. This lets you see exactly what you're committing
while writing the message — no more "wait, what did I change?" moments.

---

## Deep Dive: Empty Commits

```bash
git commit --allow-empty -m "chore: trigger CI pipeline"
```

Normally Git refuses to commit if the index matches the current HEAD tree:
```
On branch main
nothing to commit, working tree clean
```

With `--allow-empty`, Git creates a commit where `tree` SHA is **identical** to the parent
commit's `tree` SHA. The commit object is new (different timestamp/message) but it points
to the same tree.

**When is this useful?**
- Manually triggering CI/CD pipelines that watch for new commits
- Creating milestone markers in history
- Preserving a commit that would otherwise become empty after a rebase --interactive

```bash
# PROVE IT: Empty commit has same tree as parent
git commit --allow-empty -m "Trigger CI"
git cat-file -p HEAD
# tree: abc123...

git cat-file -p HEAD^
# tree: abc123...   ← SAME SHA
```

---

## The Plumbing Commands Behind the Porcelain

Every porcelain command (`git add`, `git commit`) is built on plumbing:

### `git hash-object` — Compute / Create a Blob

```bash
# Compute SHA without writing
git hash-object server.js
# Shows SHA that WOULD be used (doesn't write anything)

# Compute AND write to object database
git hash-object -w server.js
# Creates .git/objects/<2>/<38 chars>

# Hash from stdin
echo "hello" | git hash-object --stdin
# ce013625030ba8dba906f756967f9e9ca394464a
```

### `git update-index` — Directly Manipulate the Index

```bash
# Stage a specific blob SHA for a path (low-level git add)
git update-index --add --cacheinfo 100644,<sha>,server.js

# Remove from index without touching disk
git update-index --remove server.js

# Set assume-unchanged flag
git update-index --assume-unchanged server.js

# Set skip-worktree flag (see Topic 05)
git update-index --skip-worktree .env.local
```

### `git write-tree` — Build Tree from Index

```bash
# Build a tree from the current index and print its SHA
git write-tree
# abc123...  ← root tree SHA
# Object is written to .git/objects/
```

### `git commit-tree` — Create a Commit from a Tree

```bash
# Create a commit manually (no branch update)
git commit-tree <tree-sha> -m "Message" [-p <parent-sha>]

# Examples:
git commit-tree abc123 -m "First commit"
# → writes commit object, prints SHA; does NOT move any refs

git commit-tree abc123 -m "Second" -p <first-commit-sha>
# → commit with one parent

git commit-tree abc123 -m "Merge" -p <parent1-sha> -p <parent2-sha>
# → merge commit with two parents
```

**Why these matter:**

These plumbing commands are used by:
- `git filter-branch` — rewrites history using write-tree + commit-tree
- `git fast-import` — bulk import of commits
- Custom CI scripts that create commits programmatically
- `git stash` — creates a stash commit using commit-tree
- Git server-side operations

```bash
# PROVE IT: Create a commit entirely from plumbing
echo "hello plumbing" > plumbing-test.txt
BLOB=$(git hash-object -w plumbing-test.txt)
git update-index --add --cacheinfo 100644,$BLOB,plumbing-test.txt
TREE=$(git write-tree)
COMMIT=$(git commit-tree $TREE -m "Manual commit via plumbing")
echo $COMMIT
git cat-file -p $COMMIT
# This commit exists in .git/objects/ but is an orphan — nothing points to it
# You could point a branch at it:
# git branch my-plumbing-branch $COMMIT
```

---

## RULE 7 — Beginner Example

### Scenario: Your First Real Commit

You're building a todo app. You've just written your first HTML file.

```bash
# Start fresh
mkdir todo-app && cd todo-app
git init

# Create your file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Todo App</title></head>
<body>
  <h1>My Todos</h1>
</body>
</html>
EOF

# STEP 1: Check status — see what Git sees
git status
# On branch main (or master)
# No commits yet
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#         index.html

# STEP 2: Stage the file
git add index.html

# STEP 3: Verify what's staged
git status
# Changes to be committed:
#   new file:   index.html

git diff --cached
# Shows the diff between HEAD (empty) and staged index.html

# STEP 4: Commit
git commit -m "feat: add initial HTML structure for todo app"

# STEP 5: Verify the commit was created
git log --oneline
# abc1234 feat: add initial HTML structure for todo app

git show HEAD
# Full commit: tree, author, diff

# STEP 6: Look at .git/ to see what was created
find .git/objects -type f | sort
# Shows: 2 objects (blob + commit) and 1 tree
# (or 3 objects total: blob, tree, commit)

ls .git/refs/heads/
# main (or master) — the branch file was just created!
cat .git/refs/heads/main
# abc1234...   (the commit SHA)
```

---

## RULE 7 — Production Example

### Scenario: Atomic Commits on a Feature Branch

You're a backend engineer on a payment processing team. Your ticket is:
"Implement Stripe webhook handling." You've been coding for 3 hours and now have:

- Modified: `src/webhooks/stripe.js` — the webhook handler
- Modified: `src/config/stripe.js` — added webhook secret to config
- Modified: `src/routes/index.js` — registered the new route
- New file: `test/webhooks/stripe.test.js` — tests for the handler
- Modified: `package.json` — added Stripe SDK dependency

**You want THREE atomic commits:**
1. "chore: add Stripe SDK dependency" (just package.json)
2. "feat: implement Stripe webhook handler" (stripe.js + config + route)
3. "test: add unit tests for Stripe webhooks" (test file)

```bash
# COMMIT 1: Only package.json
git add package.json
git commit -m "chore: add Stripe SDK dependency"

# COMMIT 2: Handler, config, route
git add src/webhooks/stripe.js src/config/stripe.js src/routes/index.js
git commit -m "feat: implement Stripe webhook handler

Handles checkout.session.completed and payment_intent.failed events.
Verifies webhook signature using STRIPE_WEBHOOK_SECRET.
Logs all events to audit table via EventService.

Closes #284."

# COMMIT 3: Tests
git add test/webhooks/stripe.test.js
git commit -m "test: add unit tests for Stripe webhook handler

Tests: happy path, invalid signature, missing event type,
database failure simulation using test doubles."

# Verify the result
git log --oneline
# 3 clean, atomic, reviewable commits
# A reviewer can look at commit 2 in isolation and review the logic
# without test noise; commit 3 can be reviewed knowing the logic is correct
```

**Why atomic commits matter at work:**
- `git bisect` can binary-search to "feat: implement Stripe webhook handler"
  and find bugs introduced there (Topic 21)
- `git revert` can undo exactly one commit if the webhook breaks production
  without touching the others (Topic 12)
- Code reviewers can focus on one concern per commit
- CI can show which exact commit introduced a failure

---

## RULE 7 — Production Scenario: The `--patch` Workflow

### Scenario: You Made Too Many Changes in One Session

You spent an afternoon on a feature. The file `api/users.js` has:
- New endpoint `/users/:id/profile` (should be commit A)
- Fixed a null-pointer bug in `getUserById` (should be separate commit B — maybe a hotfix)

Both changes are in the same file. You need to split them.

```bash
git add --patch api/users.js
```

Git shows you each "hunk" (contiguous changed region) one at a time:

```
@@ -45,6 +45,18 @@ function getUserById(id) {
+  // Fix: check for null user before accessing properties
+  if (!user) return null;
   return user.profile;
 }
Stage this hunk [y,n,q,a,d,s,e,?]?
```

**The `--patch` hunk commands:**

| Key | Action |
|-----|--------|
| `y` | Yes — stage this hunk |
| `n` | No — don't stage this hunk |
| `q` | Quit — don't stage this or any remaining hunks |
| `a` | All — stage this AND all remaining hunks in the file |
| `d` | Don't — skip this AND all remaining hunks in the file |
| `s` | Split — split this hunk into smaller hunks (if possible) |
| `e` | Edit — open the hunk in an editor to manually edit what gets staged |
| `?` | Help — show all commands |

```bash
# Stage only the null fix (hunk 1: y)
# Skip the new endpoint (hunk 2: n)
git commit -m "fix: prevent null pointer in getUserById"

# Now stage the new endpoint
git add api/users.js
git commit -m "feat: add user profile endpoint GET /users/:id/profile"

# Result: Two clean commits from one messy work session
```

---

## RULE 8 — Mistakes, Root Causes, and Fixes

### Mistake 1: Committed to the Wrong Branch

**Situation**: You meant to be on `feature/auth` but committed to `main`.

```bash
git log --oneline -3
# abc123 your wrong commit (HEAD → main)
# def456 previous main commit
```

**Root cause**: You forgot to create/switch to your feature branch first.

**Fix:**
```bash
# 1. Save the commit SHA
git log --oneline -1   # note abc123

# 2. Move main back one commit (soft reset — keeps changes staged)
git reset --soft HEAD~1
# HEAD now at def456, abc123 is orphaned, changes still staged

# 3. Create and switch to the right branch
git checkout -b feature/auth

# 4. Commit on the right branch
git commit -m "feat: add auth module"
```

**Prevention**: Always `git status` to check your branch before committing.
Use your shell prompt to show the current branch (oh-my-zsh, Starship, etc.).

---

### Mistake 2: Staged Everything Including Debug Code

**Situation**: You ran `git add .` and accidentally staged `console.log` statements,
commented-out blocks, and `.env` file.

**Root cause**: Used a bulk-add command without checking what would be staged first.

**Fix (before committing):**
```bash
# Unstage a specific file
git restore --staged .env

# Unstage specific lines using patch
git restore --staged --patch api/users.js
# This is the REVERSE of git add --patch — lets you unstage specific hunks

# Check what's still staged
git diff --cached
```

**Prevention**:
1. Always run `git diff --cached` before `git commit` to review staged changes
2. Add sensitive files to `.gitignore` BEFORE first `git add`
3. Use `git add -p` instead of `git add .` for more control

---

### Mistake 3: Committed a File That Should Be Ignored

**Situation**: You committed `node_modules/` or `.env` before adding them to `.gitignore`.

**Root cause**: `.gitignore` only works for UNTRACKED files. Once a file is tracked (has
been committed), Git tracks it forever regardless of `.gitignore`.

**Fix:**
```bash
# Remove from tracking without deleting from disk
git rm -r --cached node_modules/
git rm --cached .env

# Add to .gitignore
echo "node_modules/" >> .gitignore
echo ".env" >> .gitignore

# Commit the removal
git commit -m "chore: remove tracked files that should be ignored

node_modules/ and .env should not be in version control.
Adding to .gitignore and removing from index."
```

**IMPORTANT**: This fixes it going FORWARD. The history still contains the committed files.
If `.env` contained secrets, you need to rotate those secrets AND use `git filter-repo`
to scrub the history (Topic 25 covers this).

---

### Mistake 4: Wrote the Commit Message Wrong

**Situation**: You committed with `-m "wip"` and now the branch has a useless message.

**If the commit was your LAST commit and NOT yet pushed:**
```bash
git commit --amend -m "feat: implement user registration with email verification"
# Creates a new commit, old "wip" commit becomes orphan
```

**If the commit was NOT your last (multiple commits back):**
```bash
# Use interactive rebase (Topic 11)
git rebase -i HEAD~3  # go back 3 commits
# Change "pick" to "reword" on the line with "wip"
# Git opens editor for that commit's message
```

**If already pushed:**
- Do NOT amend/rebase — it rewrites history for everyone
- Instead: `git commit --fixup <sha>` and coordinate with your team
- Or accept the bad message and fix it on the NEXT commit

---

### Mistake 5: `git add .` Staged a Huge Binary File

**Situation**: You ran `git add .` and staged a 500MB video file or compiled binary.

**Before committing (easy fix):**
```bash
git restore --staged large-file.mp4
echo "*.mp4" >> .gitignore
git add .gitignore
```

**After committing:**
```bash
# Amend to remove the file (if last commit, not pushed)
git rm --cached large-file.mp4
git commit --amend --no-edit
# Now the commit doesn't contain the file
# But the BLOB OBJECT is still in .git/objects/ — it'll be removed by GC
```

**After pushing (serious situation):**
- You cannot simply remove it — the push already put it on the remote
- You need `git filter-repo --path large-file.mp4 --invert-paths` (Topic 25)
- Everyone who cloned after your push needs to reclone
- This is very painful — the lesson is to **add `.gitignore` first, always**

---

### Mistake 6: Empty Commit Message (Accidental Abort)

**Situation**: You opened the editor for a commit, deleted everything by accident, and
saved. Git said "Aborting commit due to empty commit message."

**The `.git/COMMIT_EDITMSG` file still has your old partial message:**
```bash
cat .git/COMMIT_EDITMSG
# You can see what you had written
```

**Re-open the editor with your partial message:**
```bash
git commit -e -F .git/COMMIT_EDITMSG
# -F reads the message from a file
# -e opens the editor so you can edit it
```

---

## RULE 5 — Hands-On Proof: Complete Walkthrough

### Build and Verify a Full Commit from Scratch

```bash
# Setup
mkdir git-internals-demo && cd git-internals-demo
git init

# Step 1: Create files
echo "const greet = name => console.log('Hello ' + name);" > greet.js
mkdir utils
echo "module.exports = { greet };" > utils/index.js

# Step 2: Check .git/ state — NO index, NO objects
find .git/objects -type f
# (empty)

ls .git/refs/heads/
# (empty)

# Step 3: Add greet.js
git add greet.js

# See the blob that was created
find .git/objects -type f
# .git/objects/XX/YYYYYYY...  (one blob)

git ls-files --stage
# 100644 <sha> 0    greet.js

# Step 4: Add utils/index.js
git add utils/

git ls-files --stage
# 100644 <sha1> 0    greet.js
# 100644 <sha2> 0    utils/index.js

# Step 5: Examine the index binary header
xxd .git/index | head -4
# 00000000: 4449 5243 0000 0002 0000 0002 ...  DIRC....
#           ^ D I R C   ^ ver 2   ^ 2 entries

# Step 6: Build the tree manually (to see what commit will use)
git write-tree
# <root-tree-sha>

git cat-file -p <root-tree-sha>
# 100644 blob <sha1>   greet.js
# 040000 tree <sha2>   utils

git cat-file -p <sha2-of-utils-tree>
# 100644 blob <sha3>   index.js

# Step 7: Create the actual commit
git commit -m "feat: add greeting module with utils"

# Step 8: Read the commit
git cat-file -p HEAD
# tree <root-tree-sha>
# author ...
# committer ...
# 
# feat: add greeting module with utils

# Step 9: Verify branch was created
cat .git/refs/heads/main
# <commit-sha>

cat .git/HEAD
# ref: refs/heads/main

# Step 10: Count all objects created
find .git/objects -type f | wc -l
# 4: (1 commit + 1 root tree + 1 utils tree + 2 blobs = 5... check yours)

# Step 11: Draw the full object graph
git cat-file --batch-all-objects --batch-check
# Lists all objects: sha type size
```

---

## RULE 10 — Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

**1.** You run `git add server.js`. Name ALL the files in `.git/` that are created or
modified as a result. What is in each one?

**2.** You run `git commit -m "Init"` on a repo with 0 previous commits and 2 staged files
in the root directory. How many new objects are written to `.git/objects/`? List each one
and its type.

**3.** What is the difference between `git add .`, `git add -A`, and `git add -u`?
Give a scenario where they produce different results.

**4.** You did `git add --patch server.js` and staged only 2 of the 5 hunks. You then
run `git commit -a -m "Save"`. What happens to the 3 unstaged hunks? Why?

**5.** After `git commit --amend`, the old commit SHA is different from the new commit SHA.
The old commit "vanishes" from `git log`. Does it still exist in `.git/objects/`? How would
you access it?

**6.** Why does Git sort index entries by filename path? What performance benefit does this
give?

**7.** You run `git rm --cached config/secrets.json` followed by `git commit`. The file is
still in `git log --all` history. What does this mean for secrets stored in that file?
What would you need to do to truly remove them?

**8.** What is the COMMIT_EDITMSG file? Where is it? When is it written? When does its
content get used automatically by Git?

**9.** A colleague says "I'll just use `git commit -a` for everything — no need for
`git add`." Describe a realistic scenario where this causes a bug in production.

**10.** You manually run `git write-tree`. The output SHA is identical to the tree SHA in
the previous commit. What does this tell you? Is there any reason to call `git commit` in
this state?

---

## RULE 11 — Connections to Previous Topics

**From Topic 01** — The `.git/` directory:
- Today you saw the `index` file get CREATED (first `git add`) and MODIFIED (subsequent adds)
- Today you saw `refs/heads/main` get CREATED (first commit)
- Today you saw `COMMIT_EDITMSG` appear and persist
- Today you saw `objects/` fill up with real content

**From Topic 02** — Git objects:
- Today you watched blobs get created by `git add` in real time
- Today you watched tree objects get built, bottom-up, by `git commit`
- Today you watched a commit object get assembled with all its fields
- The structural sharing optimization (unchanged trees are reused) became concrete

**From Topic 03** — The DAG and refs:
- Today you saw `refs/heads/main` get CREATED on first commit (before: didn't exist)
- Today you saw `refs/heads/main` get UPDATED on every subsequent commit
- `--amend` now makes complete sense: it creates a new commit and moves the pointer;
  the old commit becomes an orphan (no refs → not reachable from any branch)

**From Topic 05** — The Three Trees:
- Today is the MECHANICS of every tree transition:
  - `git add` = Working Dir → Index (plus blob creation as a side effect)
  - `git commit` = Index → HEAD (plus tree and commit object creation)
  - The `-a` flag = auto-stage all tracked files (bypasses explicit WD→Index step)

---

## RULE 13 — When Would I Use This at Work?

### Scenario A: The Pre-Commit Audit

Before every push to a shared branch, a senior engineer runs:
```bash
git diff --cached          # "What am I about to commit?"
git diff HEAD              # "What's different from last commit overall?"
git log --oneline origin/main..HEAD  # "What am I about to push?"
```

This habit prevents accidents. Take 30 seconds to review staged changes before committing —
this is especially valuable when using `git add -A` or `git commit -a`.

---

### Scenario B: Hotfix on Production

It's 2 AM. A payment bug is in production. Your working directory has 2 hours of
unfinished feature work.

```bash
# Stash your feature work (Topic 13)
git stash

# Create hotfix branch from main
git checkout main
git pull
git checkout -b hotfix/payment-null-ref

# Make the ONE-LINE fix
vim src/payments/processor.js

# Stage ONLY the fix (even if you edited multiple things)
git add --patch src/payments/processor.js

# Verify exactly what's staged
git diff --cached

# Commit atomically
git commit -m "fix: prevent null reference in payment processor

Checks for null paymentMethod before accessing .type property.
Fixes production incident #892."

# Push and create PR
git push origin hotfix/payment-null-ref
```

The atomic commit with detailed message means:
- Reviewers can approve in 2 minutes (one tiny change, clear explanation)
- `git revert` can undo it cleanly if the fix introduces its own bug
- The incident is fully traceable in `git log` forever

---

### Scenario C: Catching What You're Committing with `-v`

Before an important commit (especially one that will be in a release):
```bash
git commit -v
```

The editor shows you the full diff. You're reviewing your own work one last time before
it's permanent. This is where you catch the `console.log` you forgot to remove, the
commented-out debug block, the TODO you left in a critical path. This 30-second habit
saves embarrassing code review comments.

---

### Scenario D: Telling the Story Through Atomic Commits

When your PR is ready, your commit history should read like a story:

```
git log --oneline feature/user-auth
abc1234 test: add integration tests for JWT refresh flow
def5678 feat: implement JWT token refresh endpoint
ghi9012 feat: add Redis session storage for auth tokens
jkl3456 feat: implement JWT generation and validation
mno7890 chore: add jsonwebtoken and ioredis dependencies
```

Each commit is ONE logical unit. A reviewer can click through them one by one and understand
what was built, in what order, and why. A future `git bisect` can isolate bugs to a single
purposeful change.

This doesn't happen by accident — it requires intentional `git add --patch` and multiple
`git commit` calls. That's the professional Git workflow.

---

## RULE 12 — Quick Reference Card

### `git add` Commands

| Command | What it does |
|---------|-------------|
| `git add <file>` | Stage a specific file |
| `git add <dir>/` | Stage all files in a directory |
| `git add .` | Stage all changes in current directory (new + modified + deleted) |
| `git add -A` | Stage ALL changes in entire repository |
| `git add -u` | Stage modifications and deletions only (NOT new files) |
| `git add -p` | Interactive hunk-by-hunk staging |
| `git add -i` | Full interactive staging menu |
| `git add -f <file>` | Force-add an ignored file |
| `git add -n .` | Dry run — show what WOULD be staged |
| `git add -v .` | Verbose — print each file as it's staged |

### `git commit` Commands

| Command | What it does |
|---------|-------------|
| `git commit -m "msg"` | Commit with inline message |
| `git commit` | Open editor for message |
| `git commit -v` | Open editor with diff shown |
| `git commit -a -m "msg"` | Auto-stage tracked files + commit |
| `git commit --amend` | Replace last commit (opens editor) |
| `git commit --amend -m "msg"` | Replace last commit with new message |
| `git commit --amend --no-edit` | Replace last commit, keep message |
| `git commit --allow-empty -m "msg"` | Commit even if nothing changed |
| `git commit --no-verify -m "msg"` | Skip pre-commit and commit-msg hooks |
| `git commit -C <sha>` | Reuse another commit's message + author |

### `git rm` / `git mv`

| Command | What it does |
|---------|-------------|
| `git rm <file>` | Delete from disk AND stage the deletion |
| `git rm --cached <file>` | Remove from index only (keep on disk) |
| `git rm -r --cached <dir>` | Remove entire directory from tracking |
| `git mv <old> <new>` | Rename/move a file and stage the change |

### Plumbing Commands

| Command | What it does |
|---------|-------------|
| `git hash-object <file>` | Compute SHA without writing |
| `git hash-object -w <file>` | Compute SHA and write blob |
| `git update-index --add --cacheinfo <mode>,<sha>,<path>` | Add entry to index directly |
| `git write-tree` | Build tree from index, print SHA |
| `git commit-tree <tree> -m "msg" [-p <parent>]` | Create commit object manually |
| `git ls-files --stage` | List index entries with SHAs |
| `git ls-files --others` | List untracked files |
| `git ls-files --deleted` | List deleted-but-tracked files |

### Verify What's Happening

| Command | What it shows |
|---------|--------------|
| `git status` | Summary: staged vs unstaged vs untracked |
| `git diff` | Unstaged changes (WD vs Index) |
| `git diff --cached` | Staged changes (Index vs HEAD) |
| `git diff HEAD` | All changes (WD vs HEAD) |
| `cat .git/refs/heads/main` | Current branch's commit SHA |
| `cat .git/HEAD` | What HEAD points to |
| `git cat-file -p HEAD` | Read the current commit |
| `git cat-file -p HEAD^{tree}` | Read the current tree |
| `find .git/objects -type f \| sort` | All objects in database |

---

## Summary

`git add` and `git commit` are the two commands you'll run hundreds of times per week.
You now understand what happens at every level:

1. **`git add`** = compute blob SHA → write blob to object DB → update binary index file
2. **`git commit`** = read index → build tree objects (bottom-up) → build commit object →
   write COMMIT_EDITMSG → move branch pointer

The branch pointer movement is what makes a commit "real" — it's what makes your new
commit reachable from `main` (or whatever branch you're on). Everything else (objects,
trees) could be built silently — it's the branch pointer that connects it to history.

Key insight to remember: **Git's staging area is not a "temporary holding zone." It is
the precise specification of what the NEXT commit will look like.** Every entry in the
index is a SHA + mode + path. When you commit, that specification becomes permanent.

*Last Updated: Topic 06 — git add & git commit — Exact Internal Mechanics*
