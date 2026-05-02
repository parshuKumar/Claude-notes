# Topic 02 — Git Objects Deep Dive
### Blobs, Trees, Commits & Tags — Exactly How They Live on Disk

> **Phase 1 — The Physical Foundation**
> Prerequisite: [Topic 01 — The .git/ Directory](./topic-01-the-git-directory.md)
> Next: [Topic 03 — The DAG & Refs](./topic-03-dag-and-refs.md)

---

## Connection to Topic 01

In Topic 01 you learned that `.git/objects/` is the database where ALL content lives.
You saw that after `git add`, a file appears in `.git/objects/XX/YYYYYY...`.

Now we answer: **what IS that file?** What's inside it? How is it structured?
How does one object point to another? How does Git navigate from a commit
all the way down to individual file bytes?

By the end of this topic you will be able to:
- Create blobs, trees, and commits BY HAND using Git's low-level commands
- Read any object file without the normal Git porcelain commands
- Draw the exact object graph for any commit from memory
- Know precisely what Git stores for each of the 4 object types

---

## 1. ELI5 — The Analogy

Think of Git's object database like a **warehouse full of sealed boxes**.

Every box has a label on the outside — a long number (the SHA-1 hash).
**The label is computed FROM the contents of the box.** If you change anything
inside, the label changes completely.

There are 4 types of boxes:
- **Blob box** — holds the raw text/bytes of ONE file
- **Tree box** — holds a directory listing (a list of labels pointing to other boxes)
- **Commit box** — holds a "this is the state of the warehouse at 3pm Tuesday,
  and here's who put it there and why"
- **Tag box** — holds a sticky note saying "call this particular commit 'version 1.0'"

The entire project history is just these 4 box types, pointing to each other.
Nothing else. There is no other database. There is no other format.

---

## 2. Object Types Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                   THE 4 GIT OBJECT TYPES                             │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  BLOB                                                           │ │
│  │  Stores: Raw content of ONE file (bytes only, no filename)      │ │
│  │  Points to: Nothing (leaf node)                                 │ │
│  │  Created by: git add <file>                                     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  TREE                                                           │ │
│  │  Stores: Directory listing (name + mode → SHA for each entry)   │ │
│  │  Points to: Blobs (files) and other Trees (subdirectories)      │ │
│  │  Created by: git commit (Git builds trees from the index)       │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  COMMIT                                                         │ │
│  │  Stores: Root tree SHA + parent commit(s) SHA + metadata        │ │
│  │  Points to: Exactly ONE root tree + 1 or more parent commits    │ │
│  │  Created by: git commit                                         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  TAG (annotated)                                                │ │
│  │  Stores: Target SHA + tagger + date + message                   │ │
│  │  Points to: Usually a commit (can point to any object type)     │ │
│  │  Created by: git tag -a                                         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Object Storage — The Universal Format

Before diving into each type, understand this: **ALL four object types are stored
the same way on disk.** The format is identical, only the content differs.

### 3.1 The Storage Format

Every object file on disk contains:

```
zlib_compress(  HEADER  +  CONTENT  )
               │              │
               ▼              ▼
   "<type> <size>\0"    raw content bytes


WHERE:
  <type>  = "blob", "tree", "commit", or "tag" (plain ASCII)
  <size>  = decimal digit string of content byte count
  \0      = null byte (0x00) — separator between header and content
  content = the actual data (varies by type)
```

### 3.2 The SHA-1 Hash (The Filename)

```
filename = SHA-1( HEADER + CONTENT )
                = SHA-1( "<type> <size>\0" + raw_content_bytes )

The first 2 hex chars → subdirectory name
The remaining 38 chars → filename inside that subdirectory
```

### 3.3 Why This Design?

```
PROPERTY               BENEFIT
─────────────────────────────────────────────────────────────────
Content-addressable    Same content always gets same SHA — dedup
                       is automatic and free

SHA in filename        Corruption detection — if file content
                       changes, SHA won't match filename

Immutable objects      You cannot "edit" a stored object.
                       A change = a new object with a new SHA.

Type in header         Git knows what it's reading without
                       needing extra metadata files

Compressed (zlib)      Saves disk space — text compresses very well
```

---

## 4. BLOB Objects — File Content Storage

### 4.1 What a Blob Contains

A blob stores ONLY the raw bytes of a file. **No filename. No permissions. Just bytes.**

```
BLOB OBJECT STRUCTURE:
─────────────────────

Header:   "blob 21\0"
           │     │  │
           │     │  └── null byte separator
           │     └── decimal byte count of content
           └── object type

Content:  "console.log('hello')\n"
           (raw file bytes, exactly as they are on disk)

Full stored string (before compression):
"blob 21\0console.log('hello')\n"
```

### 4.2 Creating a Blob — Step by Step

Let's build a repo from scratch and watch every object appear:

```bash
mkdir git-deep-dive && cd git-deep-dive
git init
```

```bash
# Create a file
echo "console.log('hello')" > app.js

# Stage it — this creates the blob
git add app.js

# Find the new object
find .git/objects -type f
# Output: .git/objects/0a/4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3
#                       ││
#                       │└─ remaining 38 chars
#                       └─ first 2 chars = subdirectory
```

### 4.3 Reading the Blob

```bash
# Get the type
git cat-file -t 0a4e14e8
# Output: blob

# Get the size (bytes)
git cat-file -s 0a4e14e8
# Output: 21

# Get the content
git cat-file -p 0a4e14e8
# Output: console.log('hello')
```

**Syntax breakdown:**
```
git cat-file  -t  0a4e14e8
│    │         │   │
│    │         │   └── SHA-1 hash (can be abbreviated, min 4 chars if unambiguous)
│    │         └── flag: -t = type, -s = size, -p = pretty-print content
│    └── subcommand: inspect the object database
└── git binary
```

### 4.4 Proving the SHA Formula Yourself

```bash
# The content of app.js (including trailing newline) is 21 bytes
echo "console.log('hello')" | wc -c
# Output: 21

# Git's SHA-1 formula: SHA-1("blob 21\0" + file_content)
# Verify manually on Linux/Mac:
printf "blob 21\0" > /tmp/header
cat /tmp/header app.js | sha1sum
# Output: 0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3

# Or more concisely:
(printf "blob 21\0"; cat app.js) | sha1sum
# Output: 0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3

# Matches what git hash-object gives:
git hash-object app.js
# Output: 0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3
```

💡 **You just computed a Git SHA-1 by hand.** Git is not magic — it's arithmetic.

### 4.5 git hash-object — The Plumbing Command

```bash
git hash-object <file>
git hash-object -w <file>
git hash-object --stdin
```

| Flag/Arg | Meaning | What happens without it |
|---|---|---|
| `<file>` | File to hash | Required (or use `--stdin`) |
| `-w` | Write the object to the database | Without it: only prints hash, doesn't store |
| `--stdin` | Read content from standard input | Allows hashing arbitrary content |

```bash
# Hash without storing
git hash-object app.js
# Output: 0a4e14e8...  (printed, not stored)

# Hash and store
git hash-object -w app.js
# Output: 0a4e14e8...  (printed AND stored in .git/objects/)

# Hash arbitrary content
echo "test content" | git hash-object --stdin
# Output: d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

### 4.6 The Deduplication Property

**Two different files with identical content = ONE blob object.**

```bash
# Create two files with same content
echo "same content" > file1.txt
echo "same content" > file2.txt

git add file1.txt file2.txt

# Check: how many objects were created?
find .git/objects -type f | wc -l
# Output: 1  ← only ONE blob, even though two files staged

# The index points both paths to the same blob:
git ls-files --stage
# Output:
# 100644 8f75892... 0  file1.txt
# 100644 8f75892... 0  file2.txt
#         ↑ same SHA!
```

This is why large repos don't grow proportionally with repeated file content.

### 4.7 Blob Diagram

```
Working Directory         .git/index              .git/objects/
─────────────────        ──────────────          ──────────────────────

app.js                   100644                  0a/4e14e8...
"console.log('hello')"   0a4e14e8... app.js  ──► BLOB
                                                  "console.log('hello')"
package.json             100644
"{ "name": "app" }"      b7e9f0a3... pkg.json ──► b7/e9f0a3...
                                                    BLOB
                                                    "{ "name": "app" }"
```

---

## 5. TREE Objects — Directory Structure

### 5.1 What a Tree Contains

A tree is a **sorted list of entries**. Each entry is:
- A **mode** (file type / permissions)
- A **name** (filename or subdirectory name)
- A **SHA-1** pointing to a blob (for files) or another tree (for subdirectories)

```
TREE OBJECT STRUCTURE:
──────────────────────

Header:  "tree <byte_count_of_content>\0"

Content: (binary, NOT text — each entry is packed bytes)
  <mode_bytes> <space> <name_bytes> <null_byte> <20_raw_sha_bytes>
  <mode_bytes> <space> <name_bytes> <null_byte> <20_raw_sha_bytes>
  ...

IMPORTANT: The SHA-1 in the content is stored as 20 RAW BYTES (binary),
           not as a 40-character hex string!
```

### 5.2 File Mode Values

```
Mode     │ Meaning
─────────┼────────────────────────────────────
100644   │ Regular file (not executable)
100755   │ Executable file (chmod +x)
120000   │ Symbolic link
040000   │ Subdirectory (tree)
160000   │ Git submodule (gitlink)
```

### 5.3 A Tree's Decoded View

When you run `git cat-file -p <tree-sha>`, Git decodes the binary and shows:

```
100644 blob 0a4e14e8...    app.js
100644 blob b7e9f0a3...    package.json
040000 tree c4d1e8a2...    routes
```

Breaking down one line:
```
100644  blob  0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3    app.js
│       │     │                                            │
│       │     └── SHA-1 of the object this entry points to│
│       └── object type (blob for file, tree for dir)     │
└── mode (unix permissions)                               └── name
```

### 5.4 Building Trees — What git commit Does Internally

Git creates tree objects from the index. Let's watch it happen:

```bash
# We have 2 files staged from before
# Add a subdirectory too
mkdir routes
echo "module.exports = {}" > routes/users.js
git add routes/users.js

# Check the index now
git ls-files --stage
# Output:
# 100644 0a4e14e8... 0  app.js
# 100644 b7e9f0a3... 0  package.json
# 100644 f2a9b3c7... 0  routes/users.js

# Now commit — Git builds tree objects automatically
git commit -m "initial commit"
```

**What Git built internally:**

```
Step 1: Build tree for routes/ subdirectory
  entries: [ (100644, "users.js", f2a9b3c7...) ]
  → Create tree object: SHA = c4d1e8a2...

Step 2: Build tree for root /
  entries: [ (100644, "app.js",      0a4e14e8...),
             (100644, "package.json", b7e9f0a3...),
             (040000, "routes",       c4d1e8a2...) ]   ← subdirectory tree
  → Create tree object: SHA = d9f3a2b1...

Step 3: Build commit object
  tree   = d9f3a2b1...   ← root tree
  parent = (none, first commit)
  author = ...
  message = "initial commit"
  → Create commit object: SHA = 9f2d14a3...
```

### 5.5 Reading the Tree Hierarchy

```bash
# See the commit object
git cat-file -p HEAD
# Output:
# tree d9f3a2b1e4c5f8a7b2d6e9c3a7f1b8d4e2c6a9f3
# author Parsh <p@dev.io> 1746134400 +0530
# committer Parsh <p@dev.io> 1746134400 +0530
#
# initial commit

# See the root tree
git cat-file -p d9f3a2b1
# Output:
# 100644 blob 0a4e14e8...    app.js
# 100644 blob b7e9f0a3...    package.json
# 040000 tree c4d1e8a2...    routes

# See the routes/ subtree
git cat-file -p c4d1e8a2
# Output:
# 100644 blob f2a9b3c7...    users.js

# See the actual file content
git cat-file -p f2a9b3c7
# Output:
# module.exports = {}
```

You just navigated the entire object graph manually:
**commit → root tree → subtree → blob**

### 5.6 Visual: Object Graph for One Commit

```
COMMIT 9f2d14a3
├── tree    ──────────────────► TREE d9f3a2b1  (root /)
├── parent  (none)              ├── app.js      ──► BLOB 0a4e14e8  "console.log('hello')"
├── author  Parsh               ├── package.json──► BLOB b7e9f0a3  "{ name: app }"
└── message "initial commit"    └── routes/     ──► TREE c4d1e8a2  (routes/)
                                                     └── users.js ──► BLOB f2a9b3c7  "module.exports={}"
```

### 5.7 What Changes When You Modify One File

This is critical for understanding how Git stores data efficiently.

```bash
# Modify just app.js
echo "console.log('world')" > app.js
git add app.js
git commit -m "update app.js"
```

**What new objects does Git create?**

```
NEW objects created:
  BLOB: new SHA for modified app.js          ← new content = new SHA
  TREE: new root tree (app.js SHA changed)   ← any child change = new tree
  COMMIT: new commit pointing to new tree    ← new commit always new object

REUSED objects (NOT duplicated):
  BLOB: package.json (unchanged)             ← same content = same SHA, reused
  BLOB: routes/users.js (unchanged)          ← same content = same SHA, reused
  TREE: routes/ tree (unchanged)             ← same entries = same SHA, reused
```

```
BEFORE (commit 1):                    AFTER (commit 2):

COMMIT 9f2d14 ──► TREE d9f3a2        COMMIT e7b4c8 ──► TREE a1c5f9
                  ├── app.js   ──► BLOB 0a4e14 (old)   ├── app.js   ──► BLOB 3d7e21 (NEW)
                  ├── pkg.json ──► BLOB b7e9f0          ├── pkg.json ──► BLOB b7e9f0 (REUSED)
                  └── routes/  ──► TREE c4d1e8          └── routes/  ──► TREE c4d1e8 (REUSED)
                                        └── users.js ──► BLOB f2a9b3          │
                                                                               └── users.js ──► BLOB f2a9b3 (REUSED)
```

**Only 3 new objects created (1 blob + 1 tree + 1 commit), even though the repo has 3 files.**
This is why Git is so storage-efficient.

### 5.8 git ls-tree — Inspecting Trees

```bash
git ls-tree <tree-ish> [path]
git ls-tree HEAD
git ls-tree HEAD routes/
git ls-tree -r HEAD
git ls-tree -r --name-only HEAD
```

| Flag/Arg | Meaning |
|---|---|
| `<tree-ish>` | Any tree-like ref: a tree SHA, a commit SHA, `HEAD`, branch name |
| `[path]` | Limit to a specific subdirectory |
| `-r` | Recurse into subtrees (shows ALL files, not just top-level) |
| `-l` | Show object size |
| `--name-only` | Show only filenames (no SHA/mode) |
| `-t` | Show tree entries too (not just leaf blobs) |

```bash
# Top-level only
git ls-tree HEAD
# Output:
# 100644 blob 0a4e14e8...    app.js
# 100644 blob b7e9f0a3...    package.json
# 040000 tree c4d1e8a2...    routes

# Recurse all files (like 'find' but for Git)
git ls-tree -r HEAD
# Output:
# 100644 blob 0a4e14e8...    app.js
# 100644 blob b7e9f0a3...    package.json
# 100644 blob f2a9b3c7...    routes/users.js

# Just filenames — useful for scripting
git ls-tree -r --name-only HEAD
# Output:
# app.js
# package.json
# routes/users.js
```

### 5.9 Creating a Tree Object By Hand (Plumbing)

Git has a plumbing command called `git write-tree` that converts the index to tree objects:

```bash
# Start fresh
git init manual-tree-demo && cd manual-tree-demo

# Create and stage files
echo "hello" > a.txt
echo "world" > b.txt
git add a.txt b.txt

# Write the tree manually (without committing)
git write-tree
# Output: (SHA of the root tree)
# e.g. 68aba62e560c0ebc3396e8ae9335232cd93a3f60

# Inspect it
git cat-file -p 68aba62e
# Output:
# 100644 blob ce013625...    a.txt
# 100644 blob cc628ccd...    b.txt
```

---

## 6. COMMIT Objects — Snapshots with Context

### 6.1 What a Commit Contains

```
COMMIT OBJECT STRUCTURE:
─────────────────────────

Header:  "commit <byte_count_of_content>\0"

Content (plain text, UTF-8):
  tree <root-tree-sha>
  parent <parent-commit-sha>        ← 0 lines if initial commit
  parent <another-parent-sha>       ← 2 lines for merge commits
  author <name> <email> <timestamp> <timezone>
  committer <name> <email> <timestamp> <timezone>
  gpgsig <PGP signature>            ← only if commit is signed
                                    ← blank line separator
  <commit message>
```

### 6.2 Author vs Committer — What's the Difference?

```
author    = The person who wrote the code
committer = The person who applied the commit to the repo

These are the SAME person in most cases.
They DIFFER when:
  - You apply someone else's patch (cherry-pick, format-patch/am)
  - You rebase (author stays the same, committer becomes you + new timestamp)
  - You amend a commit (author stays, committer updates to now)
```

### 6.3 The Timestamp Format

```
1746134400 +0530
│          │
│          └── timezone offset (UTC+5:30 = India Standard Time)
└── Unix timestamp (seconds since 1970-01-01 00:00:00 UTC)
```

### 6.4 Reading a Real Commit Object

```bash
# See a commit in human-readable form
git cat-file -p HEAD
# Output:
# tree d9f3a2b1e4c5f8a7b2d6e9c3a7f1b8d4e2c6a9f3
# author Parsh <parsh@dev.io> 1746134400 +0530
# committer Parsh <parsh@dev.io> 1746134400 +0530
#
# initial commit

# See the RAW bytes (decompressed but before git parses it)
git cat-file -t HEAD           # commit
git cat-file -s HEAD           # byte count of content

# Walk backward to the parent
git cat-file -p HEAD^          # inspect parent commit
git cat-file -p HEAD^^         # grandparent commit
git cat-file -p HEAD~3         # 3 commits back
```

### 6.5 The Merge Commit — Two Parents

A merge commit is simply a commit with **two `parent` lines** instead of one:

```bash
# After merging a branch
git cat-file -p <merge-commit-sha>
# Output:
# tree 4a7b9c2e...
# parent 9f2d14a3...   ← first parent (was HEAD before merge)
# parent e7b4c8f1...   ← second parent (branch being merged in)
# author Parsh <parsh@dev.io> 1746134460 +0530
# committer Parsh <parsh@dev.io> 1746134460 +0530
#
# Merge branch 'feature/auth'
```

### 6.6 Creating a Commit By Hand (Plumbing)

Git has `git commit-tree` — the raw commit creator:

```bash
git commit-tree <tree-sha> -m "message" [-p <parent-sha>]
```

| Arg/Flag | Meaning |
|---|---|
| `<tree-sha>` | The root tree object this commit will point to |
| `-m "message"` | The commit message |
| `-p <parent-sha>` | Parent commit SHA (omit for initial commit; use multiple `-p` for merge commits) |

```bash
# Build an entire commit chain BY HAND (no git add/commit):

# 1. Create a file and write blob manually
echo "first file" | git hash-object --stdin -w
# Output: f1d2d2f924e986ac86fdf7b36c94bcdf32beec15

# 2. Create a tree pointing to that blob
# (git mktree reads "mode type sha\tname" from stdin)
printf "100644 blob f1d2d2f9\treadme.txt\n" | git mktree
# Output: 7e4b9e5a...  (the tree SHA)

# 3. Create a commit pointing to that tree
git commit-tree 7e4b9e5a -m "first manual commit"
# Output: 3a8c7f1b...  (the commit SHA)

# 4. Point a branch to it
git update-ref refs/heads/manual 3a8c7f1b

# 5. Check it out
git checkout manual
git log --oneline
# Output: 3a8c7f1b first manual commit
```

**You just created a complete Git commit WITHOUT using `git add` or `git commit`.**
This is how Git works internally. The porcelain commands are just wrappers around these plumbing commands.

### 6.7 Commit Object Diagram

```
 COMMIT 9f2d14a3
┌───────────────────────────────────────┐
│ tree     d9f3a2b1 ──────────────────► TREE d9f3a2b1
│ parent   (none)                       ├── app.js    ──► BLOB 0a4e14
│ author   Parsh <p@d.io> 1746... +0530 ├── pkg.json  ──► BLOB b7e9f0
│ committer Parsh <p@d.io> 1746...+0530 └── routes/   ──► TREE c4d1e8
│                                                          └── users.js ──► BLOB f2a9b3
│ initial commit
└───────────────────────────────────────┘
 parent arrow points to previous commit:
         ▲
         │
 COMMIT 7c3e8f9b   (the commit before this one)
┌───────────────────────────────────────┐
│ tree     a1b2c3d4 ...                 │
│ parent   (even earlier commit)        │
│ ...                                   │
└───────────────────────────────────────┘
```

---

## 7. TAG Objects — Named, Immutable Bookmarks

### 7.1 Two Types of Tags

```
┌──────────────────────────────────────────────────────────────────┐
│  TYPE 1: LIGHTWEIGHT TAG                                         │
│                                                                  │
│  Just a file in .git/refs/tags/<name>                            │
│  Contains: A SHA pointing directly to a commit                   │
│  NO tag object created in .git/objects/                          │
│                                                                  │
│  Created with: git tag v1.0                                      │
│                                                                  │
│  Essentially: just a branch that never moves                     │
│  Does NOT store: tagger name, date, or message                   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TYPE 2: ANNOTATED TAG                                           │
│                                                                  │
│  Creates a TAG OBJECT in .git/objects/                           │
│  File in .git/refs/tags/<name> points to the TAG OBJECT          │
│  TAG OBJECT in turn points to a commit (or any object)           │
│                                                                  │
│  Created with: git tag -a v1.0 -m "Release version 1.0"         │
│                                                                  │
│  Stores: tagger name, email, date, message                       │
│  Can be: GPG-signed                                              │
└──────────────────────────────────────────────────────────────────┘
```

### 7.2 Lightweight Tag — What's On Disk

```bash
git tag v0.1-alpha

# What was created:
cat .git/refs/tags/v0.1-alpha
# Output: 9f2d14a3...  (directly the commit SHA)

# No new object in .git/objects/
# The tag IS just this reference file
```

### 7.3 Annotated Tag — What's On Disk

```bash
git tag -a v1.0.0 -m "First stable release"

# What was created:
# 1. A new TAG OBJECT in .git/objects/
# 2. A ref file pointing to that tag object

cat .git/refs/tags/v1.0.0
# Output: 5f3e8a9b...  (SHA of the TAG OBJECT — not the commit!)

git cat-file -t 5f3e8a9b
# Output: tag

git cat-file -p 5f3e8a9b
# Output:
# object 9f2d14a3...   ← SHA of the commit being tagged
# type commit           ← what type of object is being tagged
# tag v1.0.0            ← tag name
# tagger Parsh <parsh@dev.io> 1746134400 +0530
#
# First stable release
```

### 7.4 Annotated Tag Structure

```
ANNOTATED TAG OBJECT 5f3e8a9b
┌────────────────────────────────────┐
│ object  9f2d14a3 ────────────────► COMMIT 9f2d14a3
│ type    commit                     ├── tree ...
│ tag     v1.0.0                     ├── parent ...
│ tagger  Parsh <p@d.io> 1746...     └── message ...
│
│ First stable release               (the tagged commit's full
└────────────────────────────────────  history is accessible)

.git/refs/tags/v1.0.0 ──► 5f3e8a9b (TAG OBJECT)
                                │
                                └──► 9f2d14a3 (COMMIT)
```

### 7.5 The Indirection — Why Annotated Tags Use a Tag Object

```
LIGHTWEIGHT TAG:    refs/tags/v0.1 ──────────────────► COMMIT
                    (one hop)

ANNOTATED TAG:      refs/tags/v1.0 ──► TAG OBJECT ──► COMMIT
                    (two hops)

Why two hops?
  The TAG OBJECT stores metadata about WHO tagged it, WHEN, and WHY.
  This matters for:
  - Release notes (tag message)
  - GPG signing (who verified this release)
  - Knowing when a release was cut (separate from commit date)
```

### 7.6 git tag — Full Command Breakdown

```bash
git tag                            # list all tags
git tag v1.0                       # create lightweight tag at HEAD
git tag v1.0 <commit-sha>          # lightweight tag at specific commit
git tag -a v1.0 -m "message"       # create annotated tag
git tag -a v1.0 -m "msg" <sha>     # annotated tag at specific commit
git tag -d v1.0                    # delete tag locally
git tag -l "v1.*"                  # list tags matching pattern
git show v1.0                      # show tag details + tagged commit
```

| Flag | Meaning |
|---|---|
| `-a` | Create an **annotated** tag (makes a tag object) |
| `-m "msg"` | Tag message (requires `-a` or `-s`) |
| `-s` | GPG-sign the tag (implies `-a`) |
| `-d <name>` | Delete a tag |
| `-l <pattern>` | List tags matching glob pattern |
| `<commit>` | Tag a specific commit (default = HEAD) |

---

## 8. The Complete Object Graph — Full Picture

After several commits, here's the full graph of objects and how they relate:

```
                    .git/refs/heads/main
                           │
                           ▼
                    COMMIT e7b4c8           (second commit)
                    ├── tree   a1c5f9  ─────────────────────────────────────┐
                    ├── parent 9f2d14 ─────────────────────────────────┐   │
                    ├── author ...                                      │   │
                    └── message "update app.js"                         │   │
                                                                        │   │
                                                         ┌──────────────┘   │
                                                         ▼                  ▼
                                                  COMMIT 9f2d14        TREE a1c5f9
                                                  ├── tree d9f3a2 ─┐   ├── app.js  ──► BLOB 3d7e21 "console.log('world')"
                                                  ├── parent (none) │   ├── pkg.json──► BLOB b7e9f0 (reused)
                                                  └── msg "initial" │   └── routes/ ──► TREE c4d1e8 (reused)
                                                                    │
                                                                    ▼
                                                               TREE d9f3a2
                                                               ├── app.js  ──► BLOB 0a4e14 "console.log('hello')"
                                                               ├── pkg.json──► BLOB b7e9f0
                                                               └── routes/ ──► TREE c4d1e8
                                                                                └── users.js ──► BLOB f2a9b3


.git/refs/tags/v1.0.0 ──► TAG 5f3e8a ──► COMMIT e7b4c8 (same commit main points to)
```

**Notice what's shared:**
- `BLOB b7e9f0` (package.json) appears in BOTH commit trees — stored once
- `TREE c4d1e8` (routes/) appears in BOTH commit trees — stored once
- `BLOB f2a9b3` (users.js) appears in BOTH commit trees — stored once

---

## 9. Navigating Objects with Revision Syntax

Git has a powerful syntax for navigating the object graph without knowing SHA hashes.

### 9.1 Parent Navigation

```bash
HEAD        # current commit
HEAD^       # parent of HEAD  (same as HEAD~1)
HEAD^^      # grandparent     (same as HEAD~2)
HEAD~3      # 3 commits back
HEAD~10     # 10 commits back

# For merge commits (two parents):
HEAD^1      # first parent  (the branch you were on)
HEAD^2      # second parent (the branch you merged in)
```

### 9.2 Reaching Trees and Blobs

```bash
HEAD^{tree}           # the root tree of HEAD commit
HEAD~2^{tree}         # root tree of 2 commits ago
HEAD:app.js           # the blob for app.js in HEAD
HEAD~1:app.js         # the blob for app.js in the previous commit
main:routes/users.js  # blob for routes/users.js on main branch
```

**Prove it:**
```bash
# Get the content of app.js as it was in the PREVIOUS commit
git cat-file -p HEAD~1:app.js
# Output: console.log('hello')

# Get the content NOW
git cat-file -p HEAD:app.js
# Output: console.log('world')
```

### 9.3 git rev-parse — Convert Refs to SHAs

```bash
git rev-parse HEAD
# Output: e7b4c8f1a3d5e2b7c9f4a6d8e1b3c7f5a2d4e8b6

git rev-parse HEAD^
# Output: 9f2d14a3b7c8e1f5d6a2b9c4e8f3a7d1c5b6e2f4

git rev-parse HEAD^{tree}
# Output: a1c5f9b3d7e2a8f4c6b1d9e3a7f5c2d8e4b6a9f1

git rev-parse main
# same as HEAD if you're on main
```

`git rev-parse` is the "resolve this reference to its SHA" command. Many Git internals use it.

---

## 10. Beginner Example — Build a Repo One Object at a Time

Let's build a complete, valid Git repository using ONLY plumbing commands.
No `git add`. No `git commit`. This proves you understand the object model.

```bash
mkdir plumbing-demo && cd plumbing-demo
git init
```

**Step 1: Create blobs**
```bash
# Create blob for a README
echo "# My Project" | git hash-object --stdin -w
# Output: e28e4c89...

# Create blob for main.py
echo "print('hello world')" | git hash-object --stdin -w
# Output: 8a4aa4e0...

# Verify both exist
find .git/objects -type f
```

**Step 2: Create a tree**
```bash
# mktree reads entries from stdin: "<mode> <type> <sha>\t<name>"
printf "100644 blob e28e4c89\tREADME.md\n100644 blob 8a4aa4e0\tmain.py\n" | git mktree
# Output: 5c3b9a7f...   (the tree SHA)

# Verify
git cat-file -p 5c3b9a7f
# Output:
# 100644 blob e28e4c89...    README.md
# 100644 blob 8a4aa4e0...    main.py
```

**Step 3: Create a commit**
```bash
# Set author info for the commit
GIT_AUTHOR_NAME="Parsh" \
GIT_AUTHOR_EMAIL="p@dev.io" \
GIT_AUTHOR_DATE="2026-05-02T10:00:00+05:30" \
GIT_COMMITTER_NAME="Parsh" \
GIT_COMMITTER_EMAIL="p@dev.io" \
GIT_COMMITTER_DATE="2026-05-02T10:00:00+05:30" \
git commit-tree 5c3b9a7f -m "initial commit (built by hand)"
# Output: 3f7d2e1a...   (the commit SHA)
```

**Step 4: Create a branch pointing to the commit**
```bash
git update-ref refs/heads/main 3f7d2e1a
```

**Step 5: Update HEAD**
```bash
git symbolic-ref HEAD refs/heads/main
```

**Step 6: Check out the working directory**
```bash
git checkout main
ls
# Output: README.md  main.py

git log --oneline
# Output: 3f7d2e1a initial commit (built by hand)
```

**You built a complete Git repository without using ANY porcelain commands.**

---

## 11. Real-World Example — Production Debugging Scenario

> **Scenario:** Your team's CI/CD pipeline is failing with:
> ```
> error: object file .git/objects/4b/825dc642cb6eb9a060e54bf8d69288fbee4904 is empty
> ```
> The repo seems corrupted. What do you do?

```bash
# 1. Identify what object type is broken
git cat-file -t 4b825dc6
# Output: error  (broken object)

# 2. Check if this SHA has special meaning — 4b825dc6... is the SHA of an EMPTY TREE
#    It's a well-known SHA in Git (SHA of "tree 0\0")
git cat-file -p 4b825dc6
# Expected: (empty output — it's the empty tree)

# 3. The file exists but is empty (0 bytes) — disk write probably interrupted
ls -la .git/objects/4b/825dc642cb6eb9a060e54bf8d69288fbee4904
# Output: -rw-r--r-- 1 parsh 0 May 2 10:00 825dc6...

# 4. Re-create it — the empty tree is a known constant
# In Git, the empty tree SHA is always 4b825dc642cb6eb9a060e54bf8d69288fbee4904
git hash-object -w -t tree /dev/null
# or:
printf '' | git hash-object --stdin -w -t tree

# 5. Verify the repo is healthy
git fsck
# Should output: nothing (no errors)
```

**What you demonstrated:**
- Knowledge that object files can be corrupted/empty
- The ability to identify object types by SHA
- Knowledge that the empty tree is a constant in Git
- How to re-create a known object from scratch

---

## 12. git fsck — Verifying Object Integrity

```bash
git fsck
git fsck --full
git fsck --unreachable
git fsck --lost-found
```

| Flag | Meaning |
|---|---|
| (none) | Check all reachable objects for corruption |
| `--full` | Also check pack files and loose objects |
| `--unreachable` | Show objects that exist but aren't pointed to by any ref |
| `--lost-found` | Write dangling objects to `.git/lost-found/` for recovery |

```bash
git fsck --unreachable
# Output:
# unreachable blob a3f2c1...    ← file content not reachable from any branch
# unreachable commit 7d4e8f...  ← commit not reachable from any branch (maybe after reset)
```

These "unreachable" objects are safe — Git keeps them until garbage collection (`git gc`).
This is why `git reset --hard` is recoverable (Topic 12 and Topic 20).

---

## 13. Common Mistakes

### ❌ Mistake 1: Thinking you can edit a committed file by modifying the object

**The mistake:** "The blob file is in `.git/objects/`, so I can change the file to fix a typo."

**Why it breaks:** The filename IS the hash of the content. If you change the content,
the file has a different hash — but it's still stored under the OLD hash name.
Git will either find the mismatched hash (corruption) or just continue using the
old content because it still finds the old hash on disk.

**The fix:** Never touch files inside `.git/objects/` manually. Use proper Git commands.

---

### ❌ Mistake 2: Confusing "deleting a branch" with "deleting commits"

**The mistake:** `git branch -d feature` "deleted my work."

**Root cause:** Deleting a branch removes the **pointer file** in `.git/refs/heads/`.
The commit OBJECTS still exist in `.git/objects/`. They become "unreachable" but are
NOT immediately deleted. `git fsck --unreachable` will show them. `git reflog` will
reference them. `git gc` won't remove them for at least 2 weeks (default).

**The fix:**
```bash
# Find the SHA from reflog
git reflog
# Look for the last commit on the deleted branch
# e.g. output shows: a3f2c1d HEAD@{3}: checkout: moving from feature to main

# Re-create the branch
git branch feature a3f2c1d
```

---

### ❌ Mistake 3: Assuming file renames are stored as renames in blobs

**The mistake:** "Git tracks file renames, so there must be a 'rename' object type."

**The truth:** There is NO rename object. Git only stores blobs (content), trees (structure),
and commits (snapshots). When you rename a file:
- The blob (content) is UNCHANGED — same SHA, reused
- The tree CHANGES — new entry with new filename, pointing to same blob
- Git detects "renames" at diff time by looking for blobs that appear in new paths

```bash
git mv old-name.js new-name.js
git add -A
# Git creates a new tree with new-name.js → same blob SHA
# Blob is reused. No bytes wasted.
```

---

### ❌ Mistake 4: Thinking there's a "file history" stored per file

**The mistake:** "I want to see the history of just this one file, so Git must store
per-file history somewhere."

**The truth:** There's no per-file history object. All history is in the commit objects.
When you run `git log -- app.js`, Git walks the entire commit DAG and checks each commit's
tree to see if `app.js` changed. It's a graph traversal, not a lookup.

This is why `git log -- some/file` can be slow on large repos with lots of commits.

---

### ❌ Mistake 5: Not knowing the empty tree SHA

**The matter:** In CI scripts and scripts that do `git diff <sha>`, you sometimes need
to diff against "nothing" (e.g., comparing the first commit against empty).

```bash
# The empty tree SHA — constant in Git forever:
git hash-object -t tree --stdin < /dev/null
# Output: 4b825dc642cb6eb9a060e54bf8d69288fbee4904

# Use it to diff the first commit against "nothing":
git diff 4b825dc642cb6eb9a060e54bf8d69288fbee4904 HEAD
# Shows ALL files in HEAD as additions
```

---

## 14. What Actually Happens — Byte-Level Detail

### 14.1 Exact Binary Format of a Tree Entry

Each entry in a tree object's content is stored as BINARY (not text):

```
TREE ENTRY (binary):
┌─────────────────┬────┬──────────────┬────┬────────────────────┐
│  mode (ASCII)   │ \s │ name (ASCII) │ \0 │ SHA-1 (20 raw bytes) │
│  e.g. "100644"  │    │  "app.js"    │    │  (NOT hex string!)   │
└─────────────────┴────┴──────────────┴────┴────────────────────┘

IMPORTANT: The SHA-1 is stored as 20 BYTES, not 40 hex characters.
  40-char hex: "0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3"
  20-byte binary: 0x0a 0x4e 0x14 0xe8 ... (each pair of hex digits = 1 byte)

This halves the space needed for SHAs in trees.
```

### 14.2 Exact Text Format of a Commit Object

Commits are stored as PLAIN TEXT (not binary) after the header:

```
commit 241\0tree d9f3a2b1e4c5f8a7b2d6e9c3a7f1b8d4e2c6a9f3
author Parsh <parsh@dev.io> 1746134400 +0530
committer Parsh <parsh@dev.io> 1746134400 +0530

initial commit
│                          │
│                          └── blank line between headers and message
└── everything after the \0 (null) is the commit content
```

Byte count breakdown:
```
"tree d9f3a2b1...\n"       = 46 bytes  (5 + 1 + 40 + 1)
"author Parsh ...\n"       = ~55 bytes
"committer Parsh ...\n"    = ~58 bytes
"\n"                       = 1 byte    (blank line separator)
"initial commit\n"         = 15 bytes
───────────────────────────
TOTAL content ≈ 175 bytes

Header: "commit 175\0"     = 11 bytes
SHA-1(header + content) = the commit's SHA
```

### 14.3 Why Commits Are Immutable and Tamper-Evident

```
Parent commit hash is EMBEDDED in child commit content.
Therefore child's SHA depends on parent's SHA.
Therefore grandparent's SHA → parent's SHA → child's SHA → grandchild's SHA

If you change ANY commit in the chain:
  - That commit gets a new SHA
  - All commits AFTER it must be rewritten (they embed the old SHA)
  - All THEIR SHAs change too

This is why `git rebase` "rewrites history" — it literally creates new
commit objects with new SHAs, even if the changes are identical.
```

```
Original:         C1(sha:aa) ──► C2(sha:bb, parent:aa) ──► C3(sha:cc, parent:bb)

After rebasing C2 with modified message:
Rewritten:        C1(sha:aa) ──► C2'(sha:dd, parent:aa) ──► C3'(sha:ee, parent:dd)

C2' and C3' are COMPLETELY NEW OBJECTS with new SHAs.
The old C2 and C3 still exist in .git/objects/ until garbage collected.
```

---

## 15. Mental Model Checkpoint

**Can you answer these without looking?**

1. A blob object stores what exactly? What does it NOT store?
2. You have 5 branches. How many times is an unchanged `config.yaml` file stored in `.git/objects/`?
3. What is the exact format of the string that Git SHA-1 hashes to create a blob?
4. A tree entry stores a SHA-1 as hex string (40 chars) or raw bytes (20 bytes)?
5. What's the difference between a lightweight tag and an annotated tag — physically, on disk?
6. You make 2 commits. Commit 2 modifies only 1 file out of 10. How many new objects are created?
7. What does `git cat-file -p HEAD^{tree}` give you?
8. The commit stores the author and committer separately — when would these be different?
9. Why does rebasing change commit SHAs even if the file changes are identical?
10. If you delete a branch, do the commit objects in `.git/objects/` get deleted immediately?

---

## 16. Quick Reference Card

| Command | What It Does |
|---|---|
| `git cat-file -t <sha>` | Show type of object (blob/tree/commit/tag) |
| `git cat-file -s <sha>` | Show size of object in bytes |
| `git cat-file -p <sha>` | Pretty-print contents of any object |
| `git cat-file -p HEAD` | Inspect the current commit |
| `git cat-file -p HEAD^{tree}` | Inspect root tree of current commit |
| `git cat-file -p HEAD:path/file` | Inspect specific file blob from HEAD |
| `git hash-object <file>` | Compute SHA-1 Git would assign to a file |
| `git hash-object -w <file>` | Compute SHA-1 and write blob to database |
| `git ls-tree HEAD` | List root tree of current commit |
| `git ls-tree -r HEAD` | List ALL files in current commit (recursive) |
| `git ls-tree -r --name-only HEAD` | List all filenames only |
| `git write-tree` | Write current index as a tree object |
| `git commit-tree <tree> -m "msg"` | Create commit object from a tree |
| `git mktree` | Create tree object from stdin entries |
| `git rev-parse HEAD` | Resolve HEAD to its full SHA |
| `git rev-parse HEAD^{tree}` | Get tree SHA of current commit |
| `git fsck` | Check object database integrity |
| `git fsck --unreachable` | Show objects not pointed to by any ref |
| `git count-objects -v` | Count loose and packed objects |

---

## 17. When Would I Use This At Work?

```
╔══════════════════════════════════════════════════════════════════════╗
║              WHEN WOULD I USE THIS AT WORK?                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. RECOVERING A LOST FILE FROM ANY POINT IN HISTORY                ║
║     git cat-file -p <branch>:<path/to/file> > recovered.js          ║
║     No need to checkout the whole commit.                            ║
║                                                                      ║
║  2. DEBUGGING CI CORRUPTION                                          ║
║     "bad object" errors → git fsck to identify broken objects        ║
║     → re-fetch or reconstruct the specific object                    ║
║                                                                      ║
║  3. WRITING GIT AUTOMATION SCRIPTS                                   ║
║     Shell scripts that manipulate blobs directly, build custom       ║
║     commit pipelines (e.g., publishing docs from a build artifact    ║
║     to a gh-pages branch without a working directory checkout).      ║
║                                                                      ║
║  4. UNDERSTANDING WHY git rebase AFFECTS SHARED BRANCHES            ║
║     You can explain to juniors: "rebase creates new commit           ║
║     objects with new SHAs — the old SHAs their branches track        ║
║     become orphaned and force-push would be needed."                 ║
║                                                                      ║
║  5. PERFORMANCE ANALYSIS                                             ║
║     git count-objects -v → "in-pack: 500000 objects"                 ║
║     → recommend git gc --aggressive or shallow clone for CI.         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Up Next:** [Topic 03 — The DAG & Refs](./topic-03-dag-and-refs.md)
> We now know what objects ARE. Next: how they form a Directed Acyclic Graph,
> how branches and tags are NAVIGATED, and how `git log` traverses history.
