# Topic 05 — The 3 Trees of Git
### Working Directory, Index (Staging Area), HEAD — The Complete Data Flow

> **Phase 2 — Setup & The Three Zones**
> Prerequisite: [Topic 04 — Git Setup & Configuration](./topic-04-git-setup-configuration.md)
> Next: [Topic 06 — git add & git commit Internals](./topic-06-git-add-commit-internals.md)

---

## Connection to Previous Topics

In Topic 02 you learned what objects are (blobs, trees, commits).
In Topic 03 you learned how commits chain together into the DAG.
In Topic 01 you saw `.git/index` as a binary file that maps paths to blob SHAs.

Now we answer: **how does a file travel from your editor, through Git, and into a commit?**

This is THE mental model that unlocks ALL of Git. Every command — `add`, `commit`,
`reset`, `checkout`, `restore`, `stash`, `merge`, `rebase` — is just moving data
between these three zones. Once you see it clearly, nothing in Git surprises you again.

---

## 1. ELI5 — The Analogy

Imagine you're packing boxes to move to a new house. You have three areas:

1. **Your current room** — everything is scattered around, some things are packed,
   some aren't. This is your **Working Directory**. It's the mess of reality.

2. **A staging table** in the hallway — you've made deliberate decisions about
   what to put here. "These 3 things are going in Box #47. I've decided."
   This is the **Index (Staging Area)**. It's the curated list of what goes next.

3. **The moving truck** — boxes already loaded. Permanent, done, recorded.
   This is the **Repository (HEAD)**. It's committed history.

The workflow:
- You decide something is ready → move it from room to staging table (`git add`)
- You seal and load a box → move everything from staging table to truck (`git commit`)
- You want to unpack a box to check something → move from truck back to room (`git checkout`)

The beauty: **the staging table lets you be selective**. Your room might have 20 things
changed. You can choose to put only 3 on the staging table right now. The other 17 stay
in the room until you're ready.

---

## 2. The Three Trees — Defined

Git officially calls these the "Three Trees" (it's in Git's own documentation):

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        THE THREE TREES OF GIT                            │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │
│  │  WORKING         │  │  INDEX           │  │  REPOSITORY          │   │
│  │  DIRECTORY       │  │  (Staging Area)  │  │  (HEAD)              │   │
│  │                  │  │                  │  │                      │   │
│  │  Your actual     │  │  .git/index      │  │  .git/objects/       │   │
│  │  files on disk   │  │  (binary file)   │  │  (object database)   │   │
│  │                  │  │                  │  │                      │   │
│  │  Anything goes.  │  │  Intentional     │  │  Permanent.          │   │
│  │  Untracked,      │  │  snapshot-in-    │  │  Every commit is     │   │
│  │  modified,       │  │  progress.       │  │  immutable once      │   │
│  │  deleted...      │  │  "The next       │  │  written.            │   │
│  │                  │  │   commit will    │  │                      │   │
│  │                  │  │   look like      │  │                      │   │
│  │                  │  │   this."         │  │                      │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘   │
│           │                     │                       │               │
│           └─────── git add ────►│                       │               │
│                                 └──── git commit ──────►│               │
│           ◄──────────────────────────── git checkout ───┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

The critical insight: **at any moment, all three zones can contain DIFFERENT versions
of the same file.** That's what `git status` and `git diff` are actually comparing.

---

## 3. Tree 1: The Working Directory

### 3.1 What It Is

The Working Directory is your project folder — every file and subdirectory you can
see in your file explorer. It is a filesystem view of your project at the current
moment.

```
my-project/           ← This entire folder is your Working Directory
├── src/
│   ├── server.js     ← you've been editing this
│   └── auth.js       ← untouched since last commit
├── tests/
│   └── server.test.js← you created this new file (untracked)
├── package.json      ← untouched
└── .gitignore        ← untouched
```

### 3.2 What Git Tracks About Working Directory Files

Git classifies each file in your Working Directory into one of these states:

```
┌──────────────────────────────────────────────────────────────────────┐
│           FILE STATES IN THE WORKING DIRECTORY                       │
│                                                                      │
│  UNTRACKED    ← Git has NEVER seen this file. Not in index or HEAD.  │
│               ← git status shows: "Untracked files:"                │
│               ← Example: a new file you just created                 │
│                                                                      │
│  TRACKED — one of:                                                   │
│                                                                      │
│    UNMODIFIED ← Identical to what's in HEAD. Clean. No changes.      │
│               ← git status doesn't show this file at all             │
│                                                                      │
│    MODIFIED   ← Different from what's in HEAD (or index).            │
│               ← git status shows: "Changes not staged for commit:"   │
│                                                                      │
│    DELETED    ← File was in HEAD/index but you deleted it on disk.   │
│               ← git status shows: "deleted: filename"                │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.3 How Git Detects Modifications (Fast Path)

Checking if every file changed by re-reading and hashing it would be slow.
Git uses a faster approach — it reads the **stat data** from the filesystem:

```
For each tracked file, Git stores in .git/index:
  - ctime (inode change time)
  - mtime (content modification time)
  - file size
  - inode number
  - device number

If ALL of these match what's on disk → file is UNMODIFIED (skip hashing)
If ANY of these differ → file MIGHT be modified → read and hash to confirm
```

This is why `git status` is fast even in large repos — it only re-hashes files
whose filesystem metadata suggests they changed.

---

## 4. Tree 2: The Index (Staging Area)

### 4.1 What It Is Physically

The index is a single **binary file** at `.git/index`. It is NOT a directory.
It is NOT a tree object in the object database. It's a custom binary format.

```bash
# It appears after your first git add
ls -la .git/index
# Output: -rw-r--r-- 1 parsh parsh 256 May 16 10:00 .git/index

# You can't cat it (binary garbage):
cat .git/index

# But git can read it:
git ls-files --stage
# Output:
# 100644 0a4e14e8... 0    src/server.js
# 100644 b7e9f0a3... 0    src/auth.js
# 100644 f2a9b3c7... 0    package.json
# 100644 d8e4c1f9... 0    .gitignore
```

### 4.2 What Each Index Entry Contains

```
One entry per tracked file:

100644  0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3  0      src/server.js
│       │                                          │      │
│       │                                          │      └── file path (relative to repo root)
│       │                                          └── stage number:
│       │                                              0 = normal
│       │                                              1 = base (during merge conflict)
│       │                                              2 = ours (during merge conflict)
│       │                                              3 = theirs (during merge conflict)
│       └── SHA-1 of the blob object for this version of the file
└── file mode (100644 = regular, 100755 = executable, 120000 = symlink)

Plus (stored in binary):
  - ctime, mtime (for fast dirty checking)
  - file size
  - inode number, device number
  - flags (assume-unchanged, skip-worktree, etc.)
```

### 4.3 The Index IS the "Next Commit"

This is the most important thing to understand about the index:

```
THE INDEX ALWAYS REPRESENTS WHAT THE NEXT COMMIT WILL LOOK LIKE.

Right after a commit:
  Index == HEAD  (they match perfectly — everything is committed)

After git add:
  Index != HEAD  (index has newer content than HEAD)

After modifying a file but NOT git add:
  Working Directory != Index  (file changed but not staged)
```

```
Example timeline:

10:00 — after last commit
  HEAD:              src/server.js → blob "v1 content"
  Index:             src/server.js → blob "v1 content"
  Working Directory: src/server.js → "v1 content"
  STATUS: nothing to commit, working tree clean

10:30 — you edit server.js
  HEAD:              src/server.js → blob "v1 content"
  Index:             src/server.js → blob "v1 content"     ← SAME AS HEAD
  Working Directory: src/server.js → "v2 content"           ← DIFFERENT
  STATUS: "Changes not staged for commit: modified: src/server.js"

11:00 — you run git add src/server.js
  HEAD:              src/server.js → blob "v1 content"
  Index:             src/server.js → blob "v2 content"     ← UPDATED
  Working Directory: src/server.js → "v2 content"
  STATUS: "Changes to be committed: modified: src/server.js"

11:30 — you run git commit
  HEAD:              src/server.js → blob "v2 content"     ← UPDATED (new commit)
  Index:             src/server.js → blob "v2 content"
  Working Directory: src/server.js → "v2 content"
  STATUS: nothing to commit, working tree clean
```

### 4.4 The Index Can Be Ahead of BOTH HEAD and Working Directory

This happens during conflict resolution:

```
During a merge conflict, the index has 3 versions of the conflicted file:

git ls-files --stage src/auth.js
# Output:
# 100644 aaa111... 1    src/auth.js    ← stage 1: base (common ancestor)
# 100644 bbb222... 2    src/auth.js    ← stage 2: ours (HEAD version)
# 100644 ccc333... 3    src/auth.js    ← stage 3: theirs (incoming version)

# After you resolve and git add:
# 100644 ddd444... 0    src/auth.js    ← stage 0: resolved (the merged version)
```

Topic 18 (Conflict Resolution) covers this in full depth.

---

## 5. Tree 3: HEAD (The Repository)

### 5.1 What It Is

"HEAD" as the third tree refers to the **snapshot of the last commit** — specifically,
the tree object that the current commit points to.

```
HEAD → (the current commit) → (the root tree of that commit) → (all blobs)

This is the "Repository" tree. It represents:
"What is the officially committed state of the project?"
```

```bash
# See what HEAD's tree looks like (all files in the last commit)
git ls-tree -r HEAD --name-only
# Output:
# .gitignore
# package.json
# src/auth.js
# src/server.js

# Compare HEAD's tree to the index:
git diff --cached           # index vs HEAD  ("what's staged?")

# Compare working directory to the index:
git diff                    # working dir vs index  ("what's modified but not staged?")

# Compare working directory to HEAD (both diffs combined):
git diff HEAD               # working dir vs HEAD  ("total changes since last commit")
```

### 5.2 What HEAD Actually Points To

```
Normally:
  .git/HEAD → "ref: refs/heads/main"
  .git/refs/heads/main → "c7f3a9b2..."   (commit SHA)
  .git/objects/c7/f3a9b2... → commit object
    └── tree: d9f3a2b1...              ← THIS is the "HEAD tree"
        ├── src/server.js → blob "v2 content"
        ├── src/auth.js   → blob "v1 content"
        └── package.json  → blob "{...}"

When people say "Tree 3 is HEAD", they mean this entire tree snapshot.
```

---

## 6. The Full State Space — What git status Really Shows

`git status` compares all three trees simultaneously:

```
git status actually performs TWO comparisons:

COMPARISON 1: HEAD  vs  Index
─────────────────────────────
  Files that differ → "Changes to be committed"
  (these are STAGED changes — ready for the next commit)

COMPARISON 2: Index  vs  Working Directory
──────────────────────────────────────────
  Files that differ → "Changes not staged for commit"
  (these are UNSTAGED modifications)

Files in Working Directory not in Index at all → "Untracked files"
```

```
                  HEAD          INDEX         WORKING DIR
────────────────────────────────────────────────────────────────
server.js        "v1 content"  "v2 content"  "v2 content"
                 └─────────────┘             
                 STAGED (HEAD≠Index)         not unstaged (Index==WD)

auth.js          "v1 content"  "v1 content"  "v3 content"
                               └────────────────┘
                 not staged    UNSTAGED (Index≠WD)

package.json     "v1 content"  "v1 content"  "v1 content"
                 CLEAN (all three match)

new-file.js      (doesn't exist) (doesn't exist) "some content"
                                                 UNTRACKED
```

```bash
git status
# Output:
# On branch main
# Changes to be committed:
#   (use "git restore --staged <file>..." to unstage)
#         modified:   src/server.js
#
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#         modified:   src/auth.js
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#         new-file.js
```

---

## 7. What git diff Actually Compares

```
git diff                    ← Working Directory  vs  Index
git diff --cached           ← Index              vs  HEAD
git diff --staged           ← same as --cached (alias)
git diff HEAD               ← Working Directory  vs  HEAD  (both comparisons combined)
git diff <commit>           ← Working Directory  vs  that commit's tree
git diff <commit1> <commit2>← that commit's tree vs that commit's tree (no working dir involved)
git diff HEAD~1 HEAD        ← what changed in the last commit
```

```
VISUAL:

  HEAD ◄────── git diff --cached ────── INDEX ◄────── git diff ────── WORKING DIR
    │                                                                        │
    └──────────────────── git diff HEAD ────────────────────────────────────┘
```

---

## 8. Commands That Move Data Between the Trees

This is the master table. Every Git command that touches files is moving data
between these three zones:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    COMMANDS AND WHICH TREES THEY TOUCH                       │
│                                                                              │
│  COMMAND                    │  WORKING DIR  │  INDEX  │  HEAD (repo)         │
│  ───────────────────────────┼───────────────┼─────────┼───────────────────   │
│  git add <file>             │  reads        │  writes │  unchanged           │
│  git add --patch            │  reads (part) │  writes │  unchanged           │
│  git commit                 │  unchanged    │  reads  │  writes (new commit) │
│  git commit --amend         │  unchanged    │  reads  │  replaces last commit│
│  git rm <file>              │  deletes      │  removes│  unchanged           │
│  git mv <old> <new>         │  renames      │  updates│  unchanged           │
│                             │               │         │                      │
│  git restore <file>         │  writes       │  reads  │  unchanged           │
│    (unstaged changes)       │  (overwrites) │  (source)│                     │
│  git restore --staged <file>│  unchanged    │  writes │  reads (source)      │
│    (unstage)                │               │  (reset)│                      │
│  git restore --staged       │  unchanged    │  writes │  reads               │
│             --worktree      │  writes       │  writes │  reads (source)      │
│                             │               │         │                      │
│  git reset --soft HEAD~1    │  unchanged    │  unchanged│ moves HEAD back     │
│  git reset HEAD~1 (mixed)   │  unchanged    │  writes │  moves HEAD back     │
│  git reset --hard HEAD~1    │  writes       │  writes │  moves HEAD back     │
│                             │               │         │                      │
│  git checkout <branch>      │  writes       │  writes │  moves HEAD          │
│  git checkout <file>        │  writes       │  reads  │  unchanged           │
│  git switch <branch>        │  writes       │  writes │  moves HEAD          │
│                             │               │         │                      │
│  git stash                  │  cleans       │  saves  │  unchanged           │
│  git stash pop              │  writes       │  restores│ unchanged           │
│  git merge <branch>         │  writes       │  writes │  may write           │
│  git rebase                 │  writes       │  writes │  writes              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Deep Dive: Every Transition Explained

### 9.1 Working Directory → Index: `git add`

```
BEFORE:
  Working Dir: server.js contains "v2 content" (modified)
  Index:       server.js → blob pointing to "v1 content"
  HEAD:        server.js → blob pointing to "v1 content"

COMMAND: git add src/server.js

STEP 1: Read src/server.js from Working Directory
STEP 2: Compute SHA-1("blob <size>\0v2 content") → new blob SHA
STEP 3: Write zlib-compressed blob to .git/objects/XX/YYYYYY...
STEP 4: Update .git/index:
        Change entry for src/server.js to point to new blob SHA
        Update ctime, mtime, size in the index entry

AFTER:
  Working Dir: server.js contains "v2 content"
  Index:       server.js → blob pointing to "v2 content"    ← UPDATED
  HEAD:        server.js → blob pointing to "v1 content"
```

### 9.2 Index → Repository: `git commit`

```
BEFORE:
  Working Dir: server.js "v2 content", auth.js "v1 content"
  Index:       server.js → blob_v2, auth.js → blob_v1
  HEAD:        commit C4 (server.js → blob_v1, auth.js → blob_v1)

COMMAND: git commit -m "feat: update server"

STEP 1: Read the entire .git/index
STEP 2: Build TREE objects (recursively, bottom-up through directories)
        Create subtrees for each subdirectory
        Create root tree with all entries from index
STEP 3: Create COMMIT object:
        tree = <root tree SHA>
        parent = SHA of current HEAD commit (C4)
        author = from git config
        committer = from git config
        message = "feat: update server"
STEP 4: Write commit object to .git/objects/
STEP 5: Update .git/refs/heads/main to new commit SHA
        (or whatever branch HEAD points to)

AFTER:
  Working Dir: unchanged
  Index:       unchanged
  HEAD:        NEW commit C5 (server.js → blob_v2, auth.js → blob_v1)
```

### 9.3 Repository → Index: `git reset HEAD` (unstage)

```
BEFORE:
  Index:  server.js → blob_v2  (staged for commit)
  HEAD:   server.js → blob_v1  (last committed version)

COMMAND: git restore --staged src/server.js
         (old syntax: git reset HEAD src/server.js)

STEP 1: Read the blob SHA for server.js from HEAD's tree
STEP 2: Update .git/index: set server.js entry back to blob_v1

AFTER:
  Working Dir: server.js → "v2 content"   ← UNCHANGED (your edits survive)
  Index:       server.js → blob_v1        ← RESET to match HEAD
  HEAD:        server.js → blob_v1
```

Your edits in the working directory are NEVER touched. Only the index entry changed.

### 9.4 Index → Working Directory: `git restore`

```
BEFORE:
  Working Dir: server.js → "v2 content"  (modified on disk)
  Index:       server.js → blob_v1       (what was last staged)

COMMAND: git restore src/server.js

STEP 1: Read blob for server.js from the INDEX
STEP 2: Write that blob's content to src/server.js on disk

AFTER:
  Working Dir: server.js → "v1 content"   ← OVERWRITTEN (your v2 edits are GONE)
  Index:       server.js → blob_v1        ← unchanged
  HEAD:        server.js → blob_v1        ← unchanged

⚠️  WARNING: This discards working directory changes permanently.
    There is NO undo for this (unless you had staged v2 first).
```

### 9.5 HEAD → Index + Working Directory: `git checkout <branch>`

```
COMMAND: git checkout feature/auth

STEP 1: Read .git/refs/heads/feature/auth → get commit SHA
STEP 2: Load the tree of that commit
STEP 3: For each file in that tree:
          If different from current index: update index entry
          If different from current working dir: overwrite working dir file
STEP 4: Update .git/HEAD to "ref: refs/heads/feature/auth"
STEP 5: For files in current tree but NOT in feature/auth tree: delete them

RESULT: Working Directory + Index + HEAD all match feature/auth's snapshot
```

⚠️ **Git will REFUSE to checkout if you have uncommitted changes that would be
overwritten.** This is a safety feature. You must either commit, stash, or
discard your changes first.

### 9.6 The Three-Tree Reset — Soft, Mixed, Hard

`git reset` is the command that moves HEAD to a different commit AND optionally
updates the index and working directory to match:

```
git reset --soft HEAD~1
─────────────────────────────────────────────────────────────
Moves: HEAD (and branch pointer) back to parent commit
Touches: HEAD only
Index: UNCHANGED (still has your staged changes)
Working Dir: UNCHANGED

Use case: "I committed too soon. Let me re-stage things differently."

BEFORE:                          AFTER:
HEAD → C5                        HEAD → C4
Index: matches C5                Index: unchanged (looks like C5's staged changes)
WD: some edits                   WD: unchanged
```

```
git reset HEAD~1  (default = --mixed)
─────────────────────────────────────────────────────────────
Moves: HEAD (and branch pointer) back to parent commit
Touches: HEAD + Index
Index: RESET to match new HEAD (C4's tree)
Working Dir: UNCHANGED

Use case: "I committed and staged too much. Let me re-stage only what I want."

BEFORE:                          AFTER:
HEAD → C5                        HEAD → C4
Index: matches C5                Index: matches C4 (C5's changes show as "not staged")
WD: some edits                   WD: unchanged (your changes intact, just not staged)
```

```
git reset --hard HEAD~1
─────────────────────────────────────────────────────────────
Moves: HEAD (and branch pointer) back to parent commit
Touches: HEAD + Index + Working Directory
Index: RESET to match new HEAD (C4's tree)
Working Dir: RESET to match new HEAD (C4's tree)

Use case: "Completely undo the last commit AND all staged/unstaged changes."

⚠️  DESTRUCTIVE: Working directory changes are LOST.

BEFORE:                          AFTER:
HEAD → C5                        HEAD → C4
Index: matches C5                Index: matches C4
WD: some edits                   WD: matches C4  ← YOUR EDITS ARE GONE
```

Visual summary of reset modes:

```
                HEAD      INDEX     WORKING DIR
reset --soft    moves     -         -
reset --mixed   moves     resets    -
reset --hard    moves     resets    resets
```

---

## 10. git status — Reading It Like a Senior

```bash
git status
# Full output (annotated):

On branch main                          ← which branch HEAD points to
Your branch is up to date with 'origin/main'.   ← local vs remote-tracking ref

Changes to be committed:                ← INDEX differs from HEAD
  (use "git restore --staged <file>..." to unstage)
        new file:   tests/auth.test.js  ← new in index, not in HEAD
        modified:   src/server.js       ← blob in index ≠ blob in HEAD

Changes not staged for commit:          ← WORKING DIR differs from INDEX
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   src/auth.js         ← file on disk ≠ blob in index

Untracked files:                        ← in working dir but NOT in index
  (use "git add <file>..." to include in what will be committed)
        scratch-notes.txt               ← Git has never seen this file
```

```bash
# Short version (great for terminal prompt scripts):
git status --short
# Output:
# A  tests/auth.test.js    ← A = Added to index (new file staged)
# M  src/server.js         ← M = Modified in index (staged)
#  M src/auth.js           ← space-M = Modified in working dir (not staged)
# ?? scratch-notes.txt     ← ?? = Untracked

# Column positions in --short:
# XY filename
# X = index status (staged)
# Y = working dir status (unstaged)
```

### Status Code Reference

```
X or Y  │ Meaning
────────┼────────────────────────────────
(space) │ unmodified
M       │ modified
A       │ added (new file staged)
D       │ deleted
R       │ renamed
C       │ copied
U       │ unmerged (conflict)
?       │ untracked
!       │ ignored
```

---

## 11. The `.git/index` Binary Format — What's Inside

The index has a specific binary format. Here's its structure:

```
.git/index BINARY FORMAT:
─────────────────────────────────────────────────────────────────

HEADER (12 bytes):
  4 bytes: "DIRC"           ← magic number (DIRectory Cache)
  4 bytes: version (2, 3, or 4 as uint32)
  4 bytes: number of entries (as uint32)

ENTRIES (variable length, one per file, SORTED BY PATHNAME):
  Each entry:
    4 bytes: ctime seconds
    4 bytes: ctime nanoseconds
    4 bytes: mtime seconds
    4 bytes: mtime nanoseconds
    4 bytes: device number
    4 bytes: inode number
    4 bytes: mode (e.g. 0100644 in octal)
    4 bytes: uid
    4 bytes: gid
    4 bytes: file size
   20 bytes: SHA-1 hash (raw bytes)
    2 bytes: flags (name length + special flags)
  variable: filename (null-terminated)
  padding:  1-8 null bytes to align to 8-byte boundary

EXTENSIONS (optional, variable):
  TREE extension: cached tree object SHAs for fast commit
  REUC extension: resolve-undo data (for after conflict resolution)
  UNTR extension: untracked cache (for fast status)

CHECKSUM (20 bytes):
  SHA-1 of all above content (integrity check)
```

```bash
# See the index version
python3 -c "
import struct
with open('.git/index','rb') as f:
    magic = f.read(4)
    version = struct.unpack('>I', f.read(4))[0]
    count = struct.unpack('>I', f.read(4))[0]
print(f'Magic: {magic}, Version: {version}, Entries: {count}')
"
# Output: Magic: b'DIRC', Version: 2, Entries: 4
```

---

## 12. git add — All the Variants

```bash
git add <file>                # stage one specific file
git add <directory>           # stage all changes inside a directory
git add .                     # stage all changes in current directory (recursively)
git add -A                    # stage ALL changes: modified, deleted, AND new files
git add -u                    # stage modified + deleted (NOT new/untracked files)
git add --patch               # interactively choose which HUNKS (parts) to stage
git add --interactive         # full interactive mode (menu-driven)
git add -n                    # dry run: show what WOULD be staged without actually staging
```

### `git add --patch` — The Senior Engineer's Tool

This is one of the most powerful but underused features of Git. It lets you stage
**partial file changes** — specific paragraphs (hunks) of a modified file, not the
whole file.

```bash
git add --patch src/server.js
```

You'll see diffs one hunk at a time and choose what to do with each:

```
@@ -10,7 +10,12 @@ const express = require('express');
 const app = express();
 
+// New middleware added
+app.use(express.json());
+app.use(cors());
+
 app.get('/', (req, res) => {
   res.send('Hello World');
 });
+
+app.listen(3000);

Stage this hunk [y,n,q,a,d,/,j,J,g,e,?]?
```

| Key | Action |
|---|---|
| `y` | Stage this hunk |
| `n` | Don't stage this hunk |
| `q` | Quit (stage nothing more) |
| `a` | Stage this AND all remaining hunks in file |
| `d` | Don't stage this OR any remaining hunks in file |
| `s` | Split into smaller hunks (if possible) |
| `e` | Manually edit the hunk before staging |
| `?` | Show help |

**Why this matters at work:** You're working on a bug fix and accidentally also did
some refactoring. Your team uses atomic commits — one commit per concern. With
`git add --patch`, you can commit ONLY the bug fix lines now, and the refactoring
separately.

---

## 13. Inspecting the Three Trees at Any Time

```bash
# WHAT IS IN THE WORKING DIRECTORY? (all files, tracked or not)
find . -not -path './.git/*' -type f | sort

# WHAT IS IN THE INDEX?
git ls-files --stage        ← all indexed files with blob SHAs + stage numbers
git ls-files                ← just filenames
git ls-files -m             ← only modified (index ≠ working dir)
git ls-files -d             ← only deleted (in index but gone from working dir)
git ls-files -u             ← only unmerged (stage != 0, conflict state)
git ls-files -o             ← only untracked (in working dir but not in index)

# WHAT IS IN HEAD?
git ls-tree -r HEAD --name-only     ← all files in HEAD commit
git ls-tree -r HEAD                 ← all files with blob SHAs

# COMPARE THE THREE TREES:
git diff                    ← Working Dir vs Index
git diff --cached           ← Index vs HEAD
git diff HEAD               ← Working Dir vs HEAD
```

---

## 14. Assume Unchanged & Skip Worktree — Advanced Index Tricks

These are flags you can set on index entries to change how Git treats a file:

### `--assume-unchanged`

```bash
git update-index --assume-unchanged src/config.local.js
```

Tells Git: "Don't check if this file changed. Assume it's unchanged."
- Faster `git status` for large, rarely-changing generated files
- DOES NOT prevent the file from being overwritten by `git checkout`
- Use case: generated files, large binary assets you never commit

```bash
# Un-assume:
git update-index --no-assume-unchanged src/config.local.js

# List files marked as assume-unchanged:
git ls-files -v | grep '^h'   ← lowercase 'h' = assume-unchanged
```

### `--skip-worktree`

```bash
git update-index --skip-worktree src/config.local.js
```

Stronger than assume-unchanged:
- Git won't overwrite this file even during `git checkout` / `git pull`
- Use case: local config overrides you never want committed or overwritten

```bash
# Un-skip:
git update-index --no-skip-worktree src/config.local.js

# List files marked as skip-worktree:
git ls-files -v | grep '^S'   ← uppercase 'S' = skip-worktree
```

---

## 15. Beginner Example — Watching All Three Trees Change

```bash
mkdir three-trees-demo && cd three-trees-demo
git init
```

```bash
# CREATE INITIAL STATE
echo "version 1" > app.js
git add app.js
git commit -m "initial commit"

# Check all three trees:
echo "=== HEAD ===" && git ls-tree HEAD
# 100644 blob <sha1> app.js

echo "=== INDEX ===" && git ls-files --stage
# 100644 <sha1> 0    app.js

echo "=== WORKING DIR ===" && cat app.js
# version 1

# All three trees agree. Status: clean.
git status
# nothing to commit, working tree clean
```

```bash
# MODIFY THE FILE (Working Dir changes)
echo "version 2" > app.js

git diff              # Working Dir vs Index
# -version 1
# +version 2

git diff --cached     # Index vs HEAD
# (no output — index still matches HEAD)

git status
# Changes not staged for commit:
#   modified: app.js
```

```bash
# STAGE THE FILE (Index changes)
git add app.js

git diff              # Working Dir vs Index
# (no output — they now match)

git diff --cached     # Index vs HEAD
# -version 1
# +version 2

git status
# Changes to be committed:
#   modified: app.js
```

```bash
# COMMIT (HEAD changes)
git commit -m "update to version 2"

git diff              # Working Dir vs Index   → nothing
git diff --cached     # Index vs HEAD          → nothing
git diff HEAD         # Working Dir vs HEAD    → nothing

git status
# nothing to commit, working tree clean

# All three trees agree again.
```

---

## 16. Real-World Example — Production Scenario

> **Scenario:** You're a backend developer mid-way through implementing a new
> payment endpoint. You've modified 4 files. Your team lead just paged you:
> *"Critical bug in prod — authentication is broken. Fix it NOW."*
>
> You need to switch to a hotfix branch immediately, but you're not done with
> the payment feature. What do you do?

**Option A: Stash (Topic 13 covers this in depth)**
```bash
# Save all three trees' current state and clean the working dir
git stash push --include-untracked -m "WIP: payment endpoint"

# Now your working dir is clean, matching HEAD
git checkout -b hotfix/auth-bug

# Fix the bug, commit it
# ...

# Come back to payment work
git checkout feature/payment
git stash pop
```

**Option B: Commit as WIP (simple, always safe)**
```bash
git add -A
git commit -m "WIP: payment endpoint - do not merge"
git checkout -b hotfix/auth-bug

# Fix bug...
# Come back
git checkout feature/payment
git reset HEAD~1   # undo WIP commit, keep changes staged
```

**The underlying principle:** You always need your working directory CLEAN (matching HEAD)
before switching branches, OR you need Git's permission to bring those changes along
(which only works if they don't conflict with the target branch).

Git checks: "Would switching branches overwrite any file in your working directory or
index that differs from the new branch?" If yes → refuses. If no → switches freely.

---

## 17. Common Mistakes

### ❌ Mistake 1: Thinking `git add .` and `git add -A` are the same

**The difference:**
```
git add .    ← stages new + modified files in CURRENT DIRECTORY and below
             ← does NOT stage deletions of files ABOVE current directory

git add -A   ← stages new + modified + DELETED files across the ENTIRE REPO

Example:
  You're in src/ and deleted ../README.md
  git add .   → won't stage the deletion of README.md
  git add -A  → WILL stage the deletion of README.md
```

**Fix:** Use `git add -A` when you want to stage everything including deletions.

---

### ❌ Mistake 2: Editing staged files and expecting the edit to be staged

**The mistake:** You `git add server.js`, then edit `server.js` again, then `git commit`.
You expect to commit the latest version.

**What actually happens:**
- `git add` copies the file content into the object database AT THAT MOMENT
- Further edits to `server.js` change the WORKING DIRECTORY only
- The INDEX still has the old version from when you ran `git add`
- `git commit` uses the INDEX — not the working directory!

```bash
echo "v1" > file.js && git add file.js
echo "v2" > file.js  # edit AFTER add

git diff              # shows v1→v2 (working dir ≠ index)
git diff --cached     # shows nothing→v1 (index has v1, HEAD has nothing)

git commit -m "test"  # commits v1, NOT v2!
```

**Fix:** Run `git add` again after editing to update the index.

---

### ❌ Mistake 3: Confusing `git reset HEAD <file>` with `git restore <file>`

Both sound like "undo" but they do DIFFERENT things:

```
git restore --staged <file>   ← copies from HEAD → Index  (unstages)
                               ← Working Directory: UNCHANGED

git restore <file>            ← copies from Index → Working Directory  (discards edits)
                               ← Index: UNCHANGED

git restore --staged --worktree <file>  ← copies from HEAD → both Index + Working Dir
```

---

### ❌ Mistake 4: Using `git reset --hard` when `git reset --mixed` would be safer

```
"I want to undo my last commit" — what do you mean?

If you mean "undo commit but keep my changes":
  git reset HEAD~1        ← mixed, keeps working dir changes as unstaged

If you mean "undo commit and re-stage everything":
  git reset --soft HEAD~1 ← HEAD moves back, index stays, changes are staged

If you mean "obliterate everything to clean state":
  git reset --hard HEAD~1 ← nuclear option, DESTROYS working dir changes

Most of the time, --mixed is what you want.
--hard is the dangerous one. Use it only when you're certain.
```

---

### ❌ Mistake 5: Thinking `git status` shows you ALL changes

**The mistake:** "Status says 'nothing to commit' so there are no changes."

**The truth:** `git status` only compares working dir/index/HEAD. It does NOT show:
- Changes on other branches
- Stashed changes
- Commits that exist locally but not on remote (use `git log @{u}..HEAD` for that)

---

## 18. What Actually Happens — The Index Checksum

Every time Git writes `.git/index`, it appends a 20-byte SHA-1 checksum of the entire
index file content. This prevents undetected corruption.

```
.git/index file layout:
[ HEADER (12 bytes) ]
[ ENTRY 1 ]
[ ENTRY 2 ]
...
[ ENTRY N ]
[ OPTIONAL EXTENSIONS ]
[ SHA-1 CHECKSUM OF EVERYTHING ABOVE (20 bytes) ]
```

When Git reads the index:
1. Read all content
2. Compute SHA-1 of everything except the last 20 bytes
3. Compare to the last 20 bytes
4. If mismatch → `error: index file corrupt` → rebuild with `git read-tree HEAD`

---

## 19. Mental Model Checkpoint

**Can you answer these without looking?**

1. You run `git diff` and see output. Which two trees is Git comparing?
2. You run `git diff --cached` and see nothing. What does that tell you about the index vs HEAD?
3. You edited `server.js`, ran `git add server.js`, then edited `server.js` again. If you run `git commit` right now, which version of `server.js` gets committed?
4. What does `git reset --soft HEAD~1` do to the working directory?
5. What does `git reset --hard HEAD~1` do to the working directory?
6. You run `git restore src/auth.js` (without `--staged`). Which tree is the source, which is the destination?
7. What does stage number 2 mean in a `git ls-files --stage` entry?
8. What's the difference between `git add -A` and `git add .`?
9. After `git commit`, in what state are all three trees relative to each other?
10. What git command lets you stage only PART of a file's changes?

---

## 20. Quick Reference Card

| Command | Trees Affected | What It Does |
|---|---|---|
| `git add <file>` | WD → Index | Copy file content from WD into Index |
| `git add -A` | WD → Index | Stage all changes (new + modified + deleted) |
| `git add --patch` | WD → Index | Interactively stage hunks |
| `git commit` | Index → HEAD | Write staged snapshot as new commit |
| `git diff` | WD vs Index | Show unstaged changes |
| `git diff --cached` | Index vs HEAD | Show staged changes |
| `git diff HEAD` | WD vs HEAD | Show all changes since last commit |
| `git status` | All three | Show state of all three trees |
| `git status --short` | All three | Compact two-column view |
| `git restore <file>` | Index → WD | Discard working dir changes (⚠️ destructive) |
| `git restore --staged <file>` | HEAD → Index | Unstage a file |
| `git restore --staged --worktree <file>` | HEAD → Both | Reset file in index AND working dir |
| `git reset --soft HEAD~1` | HEAD only | Move branch back, keep index + WD |
| `git reset HEAD~1` | HEAD + Index | Move branch back, unstage changes |
| `git reset --hard HEAD~1` | All three | Move branch back, discard all changes |
| `git ls-files --stage` | Index | Show all files in index with SHAs |
| `git ls-tree -r HEAD` | HEAD | Show all files in HEAD commit |
| `git ls-files -m` | WD vs Index | Show modified-but-unstaged files |
| `git ls-files -o` | WD | Show untracked files |

---

## 21. When Would I Use This At Work?

```
╔══════════════════════════════════════════════════════════════════════╗
║              WHEN WOULD I USE THIS AT WORK?                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. ATOMIC COMMITS (git add --patch)                                ║
║     You fixed a bug AND refactored code in the same session.        ║
║     Team rule: one commit per concern.                               ║
║     git add --patch lets you select EXACTLY which lines to commit   ║
║     first, then the refactor in a second commit.                     ║
║                                                                      ║
║  2. EMERGENCY CONTEXT SWITCH                                        ║
║     Mid-feature, urgent bug. Use stash (or WIP commit + reset)      ║
║     to clean your working directory without losing work.             ║
║                                                                      ║
║  3. DEBUGGING "WHY IS THIS FILE SHOWING AS MODIFIED?"               ║
║     git diff shows a change. git diff --cached shows nothing.        ║
║     → Change is in working dir but NOT staged.                       ║
║     git ls-files -m confirms exactly which files are affected.       ║
║                                                                      ║
║  4. UNDOING STAGING MISTAKES                                        ║
║     You accidentally staged the wrong file.                          ║
║     git restore --staged <file> unstages it in one command,          ║
║     your working dir change is preserved.                            ║
║                                                                      ║
║  5. UNDERSTANDING MERGE CONFLICTS                                   ║
║     Why does git status show 'both modified'?                        ║
║     Because the INDEX has stage 1/2/3 entries — Git stored all       ║
║     three versions. Resolving a conflict = writing stage 0 and       ║
║     removing stages 1, 2, 3.                                         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Up Next:** [Topic 06 — git add & git commit Internals](./topic-06-git-add-commit-internals.md)
> We now know what the three trees ARE. Next: we trace every single step Git takes
> when you run `git add` and `git commit` — from file read to object write to
> ref update — with zero steps skipped.
