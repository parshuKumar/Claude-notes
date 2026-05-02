# Topic 01 — How Git Actually Works Internally
### Snapshots, Not Diffs | The DAG | Objects & SHA-1

> **Phase 1 — The Foundation**
> Prerequisite: None | Next Topic: [Topic 02 — Git Setup & Configuration](./topic-02-git-setup-configuration.md)

---

## 1. ELI5 — Plain English First

Imagine you have a scrapbook. Every time you finish a chapter of your life, you take a **full photograph** of your entire room exactly as it looks right now — furniture, decorations, everything. You paste that photo into the scrapbook and write the date on it.

That's Git.

Most people *think* Git works like a list of changes — "I moved the couch, added a lamp, painted the wall blue." That would be a **diff-based** system (like older tools such as SVN or CVS).

**Git does NOT work that way.**

Instead, Git takes a **complete snapshot** of every file in your project every time you commit. If a file didn't change, Git is smart enough to just point to the previous snapshot of that file instead of storing a duplicate — but **conceptually, every commit is a full picture of your entire project at that moment in time.**

This one insight unlocks everything else about Git.

---

## 2. The Mental Model — What Lives Inside Git

### 2.1 The Four Object Types

Git is, at its core, a **content-addressable key-value database**. You put data in, and Git gives you back a 40-character key (a SHA-1 hash) to retrieve it later. Everything in Git is stored as one of four object types:

```
┌─────────────────────────────────────────────────────────┐
│                   GIT OBJECT DATABASE                   │
│                    (.git/objects/)                       │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   BLOB   │  │   TREE   │  │  COMMIT  │  │  TAG   │ │
│  │          │  │          │  │          │  │        │ │
│  │ Raw file │  │Directory │  │Snapshot  │  │Named   │ │
│  │ contents │  │listing   │  │metadata  │  │pointer │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
└─────────────────────────────────────────────────────────┘
```

Let's build this up from the ground floor:

---

### 2.2 BLOB — The File Content

A **blob** (Binary Large OBject) stores the raw contents of a single file. Nothing else — no filename, no permissions, just the bytes.

```
File: server.js
Content: "const express = require('express');"

           SHA-1 hash of content
                    │
                    ▼
Git stores ──► blob: a3f2c1d8... ──► "const express = require('express');"
```

💡 **Key insight:** Two files with identical contents share the same blob object. Git deduplicates automatically.

---

### 2.3 TREE — The Directory Snapshot

A **tree** object is like a directory listing. It maps filenames and permissions to either blobs (files) or other trees (subdirectories).

```
Project structure:
my-api/
├── server.js
├── package.json
└── routes/
    └── users.js

Git Tree for my-api/:
┌──────────────────────────────────────────────┐
│  TREE: d9f3a2...                             │
│                                              │
│  100644 blob a3f2c1... server.js             │
│  100644 blob b7e9f0... package.json          │
│  040000 tree c4d1e8... routes/               │
└──────────────────────────────────────────────┘
         │                         │
         ▼                         ▼
  blob: a3f2c1...           TREE: c4d1e8...
  (server.js contents)      ┌─────────────────┐
                             │ 100644 blob     │
                             │ f2a9b3... users.js│
                             └─────────────────┘
```

---

### 2.4 COMMIT — The Snapshot with Context

A **commit** object wraps a tree (the project snapshot) with metadata: who made it, when, why, and what came before it.

```
┌──────────────────────────────────────────────┐
│  COMMIT: 9f2d14...                           │
│                                              │
│  tree     d9f3a2...  ← root tree (snapshot)  │
│  parent   7c3e8f...  ← previous commit       │
│  author   Parsh <p@dev.io> 1746134400 +0530  │
│  committer Parsh <p@dev.io> 1746134400 +0530 │
│                                              │
│  feat: add user authentication endpoint      │
└──────────────────────────────────────────────┘
         │
         ▼
     TREE: d9f3a2...  (full snapshot of all files)
```

---

### 2.5 The DAG — Directed Acyclic Graph

Every commit points to its **parent commit(s)**. This chain of commits forms a **DAG — Directed Acyclic Graph**. "Directed" means pointers only go one way (child → parent). "Acyclic" means you can never loop back to yourself.

```
TIME ──────────────────────────────────────────────────►

  [C1] ──► [C2] ──► [C3] ──► [C4] ──► [C5]
   │                           │
   └── Initial commit          └── You are here (HEAD)


Each arrow = "my parent is..."
C5 knows about C4. C4 knows about C3. Etc.
You can always walk backward through history.
```

With branches, it looks like this:

```
        main
          │
  C1──C2──C3──C6──C7   ← main branch
            \
             C4──C5    ← feature/auth branch

C6 and C7 are a "3-way merge" (more on this in Topic 05)
```

This is the entire data model of Git. Everything else — branches, tags, HEAD — is just a pointer to a node in this graph.

---

### 2.6 Branches, HEAD — Just Pointers

A **branch** is nothing magical. It is a plain text file containing a 40-character SHA-1 hash pointing to a commit.

```
.git/refs/heads/main      ← contains: "9f2d14..."
.git/refs/heads/feature   ← contains: "c4a8b2..."

.git/HEAD                 ← contains: "ref: refs/heads/main"
                                        (which commit you're on)
```

```
           HEAD
            │
            ▼
           main
            │
            ▼
  C1──C2──C3──C4──[C5]
                    │
                 feature ← branch pointer moves
                           forward on each new commit
```

💡 **When you make a new commit, Git just moves the branch pointer forward.** That's it. Branches are incredibly cheap in Git — they're just 41-byte files.

---

## 3. Syntax Breakdown — The Commands That Expose Internals

You won't type these daily, but running them once will make Git permanently click.

### Inspect any Git object:

```bash
git cat-file -t <sha>
git cat-file -p <sha>
```

| Part | Meaning |
|---|---|
| `git cat-file` | Git's internal object inspector |
| `-t` | Show the **type** of the object (blob/tree/commit/tag) |
| `-p` | **Pretty-print** the object's contents |
| `<sha>` | The 40-char (or abbreviated) SHA-1 hash of the object |

### Hash any content manually:

```bash
git hash-object <file>
git hash-object -w <file>
```

| Part | Meaning |
|---|---|
| `git hash-object` | Compute the SHA-1 hash Git would use for this file |
| `-w` | Actually **write** the object to `.git/objects/` |
| `<file>` | The file to hash |

### List a tree object:

```bash
git ls-tree <tree-sha>
git ls-tree HEAD
git ls-tree HEAD --recursive
```

| Part | Meaning |
|---|---|
| `git ls-tree` | Show the contents of a tree object |
| `HEAD` | Use the tree from the current commit |
| `--recursive` | Recurse into subdirectories |

### See the commit graph visually:

```bash
git log --oneline --graph --all --decorate
```

| Part | Meaning |
|---|---|
| `--oneline` | Show each commit as a single line |
| `--graph` | Draw ASCII art of the branch/merge graph |
| `--all` | Show ALL branches, not just current |
| `--decorate` | Show branch and tag names next to commits |

---

## 4. Beginner Example → Real-World Example

### Beginner Example — Watching Git Build the Object Database

Let's start from nothing and watch Git create objects in real time.

```bash
# Start a fresh repo
mkdir my-project && cd my-project
git init

# Create a file
echo "hello world" > hello.txt

# Stage it (this creates a BLOB object)
git add hello.txt

# Look inside .git/objects — a new blob appeared!
find .git/objects -type f

# Commit it (this creates a TREE and a COMMIT object)
git commit -m "first commit"

# Inspect the commit
git cat-file -p HEAD

# Output:
# tree d8329f...
# author Parsh <p@dev.io> 1746134400 +0530
# committer Parsh <p@dev.io> 1746134400 +0530
#
# first commit

# Inspect the tree
git cat-file -p d8329f...

# Output:
# 100644 blob 3b18e5... hello.txt

# Inspect the blob
git cat-file -p 3b18e5...

# Output:
# hello world
```

You just walked the entire Git object chain: **commit → tree → blob**.

---

### Real-World Example — Backend Developer Scenario

> You're a backend developer on a Node.js REST API team. Your team lead asks:
> *"Why did our `git log` suddenly show a commit from 3 weeks ago as the tip of main? Something looks corrupted."*

Because you understand that branches are just pointers, you know exactly what to check:

```bash
# See where all branch pointers currently point
cat .git/refs/heads/main

# See the full DAG — maybe a force-push moved the pointer
git log --oneline --graph --all --decorate

# See what HEAD is actually pointing to
cat .git/HEAD

# If a branch pointer was moved incorrectly, use reflog to find
# the correct commit SHA and reset (Topic 16 covers this deeply)
git reflog show main
```

You calmly explain: *"The main branch pointer was moved. It's not corruption — a branch is just a text file with a SHA. We can restore it in 10 seconds."* 

That's the difference between someone who memorized commands and someone who understands the model.

---

## 5. Common Mistakes Beginners Make

### ❌ Mistake 1: Thinking Git stores diffs

**The mistake:** "Git is like Track Changes in Word — it saves what changed."

**The truth:** Git stores complete snapshots. It does compute diffs for display (like `git diff`), but the storage model is snapshots. This is why operations like `git checkout branch` are fast — Git just reads the tree snapshot for that branch, it doesn't replay a series of patches.

**Why it matters:** If you think in diffs, rebasing and cherry-picking will confuse you forever. If you think in snapshots, they make perfect sense.

---

### ❌ Mistake 2: Thinking branches are "copies" of code

**The mistake:** Avoiding branches because "it makes a whole copy of my project, that's wasteful."

**The truth:** A branch is a 41-byte text file. Creating a branch is instantaneous regardless of repo size.

```bash
git branch my-new-feature   # Creates a 41-byte file. Done.
```

---

### ❌ Mistake 3: Thinking deleting a branch deletes commits

**The mistake:** Panicking after `git branch -d feature/auth` thinking the work is gone.

**The truth:** Deleting a branch only deletes the pointer (the 41-byte file). The commit objects still exist in the object database. You can recover them via `git reflog` (Topic 16).

```
Before delete:          After delete:
  feature ──► C5          (pointer gone, but C5 still exists)
              │            You can still reach C5 by its SHA
              ▼
             C4
```

---

### ❌ Mistake 4: Thinking the SHA-1 hash is arbitrary

**The mistake:** Treating commit hashes like random IDs.

**The truth:** The SHA-1 hash is a cryptographic hash of the **content** of the object. If you change anything — even one character in a commit message — you get a completely different hash. This is why Git history is **tamper-evident**.

---

### ❌ Mistake 5: Confusing HEAD with a branch

**The mistake:** "HEAD is always main."

**The truth:** HEAD is a pointer to whatever you currently have checked out. Normally it points to a branch (which points to a commit). But it can also point directly to a commit SHA — this is called **detached HEAD** state, and it's a common source of panic for beginners.

```
Normal state:                  Detached HEAD state:
HEAD → main → C5               HEAD → C3
                                (not pointing to any branch)
```

---

## 6. What Actually Happens — Git Internals

When you run `git commit`, here is the exact sequence of events:

```
Step 1: git add hello.txt
────────────────────────────────────────────────────────
Git reads hello.txt contents
Git computes SHA-1: hash("blob " + filesize + "\0" + contents)
Git writes compressed object to .git/objects/XX/YYYYYY...
Git writes the path + SHA to .git/index (the staging area)

Step 2: git commit -m "message"
────────────────────────────────────────────────────────
Git reads .git/index (all staged files)
Git creates TREE objects recursively for each directory
Git creates a COMMIT object:
    - tree = SHA of root tree
    - parent = SHA in .git/refs/heads/main (current branch)
    - author/committer = from .git/config
    - message = your message
Git writes the commit object → gets SHA e.g. "9f2d14..."
Git updates .git/refs/heads/main to contain "9f2d14..."
Done.
```

Visual summary:

```
BEFORE commit:                    AFTER commit:
                                  
  .git/index                        COMMIT 9f2d14
  ┌──────────┐                      ├── tree d9f3a2
  │hello.txt │──blob──► a3f2c1      ├── parent 7c3e8f
  │server.js │──blob──► b7e9f0      └── message: "feat: add auth"
  └──────────┘                           │
                                         ▼
  main: 7c3e8f                      TREE d9f3a2
  (old commit)                      ├── blob a3f2c1 hello.txt
                                    └── blob b7e9f0 server.js
  
                                  main: 9f2d14  ← UPDATED
```

The entire operation is just: create objects, update one pointer. That's why commits are instantaneous.

---

## 7. When Would I Use This At Work?

```
╔══════════════════════════════════════════════════════════════╗
║            WHEN WOULD I USE THIS AT WORK?                   ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  SCENARIO 1: DEBUGGING A CORRUPTED REPO                     ║
║  A CI/CD pipeline fails with "bad object" errors.           ║
║  You use git cat-file to inspect specific objects and        ║
║  identify exactly which blob or tree is missing/corrupt.     ║
║                                                              ║
║  SCENARIO 2: EXPLAINING A FORCE-PUSH INCIDENT               ║
║  A teammate force-pushed and "broke" main. You calmly        ║
║  explain: the commits aren't gone, the branch pointer        ║
║  was just moved. You use git reflog to restore it.           ║
║                                                              ║
║  SCENARIO 3: UNDERSTANDING WHY REBASE CHANGES HASHES        ║
║  A code review comment says "never rebase shared branches."  ║
║  You understand WHY: rebase creates NEW commit objects        ║
║  (new SHA hashes), making the old commits orphaned. Other    ║
║  team members' local copies still have the old SHAs.         ║
║                                                              ║
║  SCENARIO 4: OPTIMIZING LARGE REPOS                         ║
║  You're asked why `git clone` is slow for your monorepo.     ║
║  Knowing that Git stores all history as objects, you         ║
║  recommend shallow clones (--depth=1) for CI pipelines.      ║
║  (Covered in depth in Topic 22)                              ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 8. Quick Reference Card

| Command | What It Does |
|---|---|
| `git log --oneline --graph --all --decorate` | Visualize the entire commit DAG |
| `git cat-file -t <sha>` | Show the type of a Git object (blob/tree/commit/tag) |
| `git cat-file -p <sha>` | Pretty-print the contents of a Git object |
| `git cat-file -p HEAD` | Inspect the current commit object |
| `git ls-tree HEAD` | List the tree (files) of the current commit |
| `git ls-tree HEAD --recursive` | List all files recursively in current commit |
| `git hash-object <file>` | Compute what SHA Git would assign to a file |
| `git hash-object -w <file>` | Compute SHA and write the object to the database |
| `find .git/objects -type f` | See all raw object files in the database |
| `cat .git/HEAD` | See what HEAD is currently pointing to |
| `cat .git/refs/heads/main` | See the SHA that the main branch points to |

---

## 9. Key Takeaways — What to Remember Forever

```
┌─────────────────────────────────────────────────────────────┐
│                  THE 5 LAWS OF GIT INTERNALS                │
│                                                             │
│  1. Git stores SNAPSHOTS, not diffs.                        │
│                                                             │
│  2. Everything in Git is an object identified by           │
│     its SHA-1 hash. (blob, tree, commit, tag)               │
│                                                             │
│  3. A branch is just a 41-byte text file               │
│     pointing to a commit SHA.                               │
│                                                             │
│  4. HEAD tells Git where you currently are.                 │
│     It usually points to a branch,                          │
│     which points to a commit.                               │
│                                                             │
│  5. Commits form a DAG — each pointing to its              │
│     parent. This chain IS your project history.             │
└─────────────────────────────────────────────────────────────┘
```

---

> **Up Next:** [Topic 02 — Git Setup & Configuration](./topic-02-git-setup-configuration.md)
> `.gitconfig`, global vs local config, useful aliases, and setting up Git the way senior engineers do it.
