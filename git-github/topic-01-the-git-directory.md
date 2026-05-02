# Topic 01 — The `.git/` Directory: Complete Physical Architecture
### What's Actually Inside That Hidden Folder?

> **Phase 1 — The Physical Foundation**
> Prerequisite: None | Next: [Topic 02 — Git Objects Deep Dive](./topic-02-git-objects-deep-dive.md)

---

## 1. ELI5 — The Analogy

Imagine you walk into a library. You see shelves of books, a card catalog, a sign saying 
"You Are Here", and a logbook where the librarian records every time someone borrows or 
returns a book.

Now imagine someone says: *"The library IS the building, not the books."*

That's wrong. **The library is the system** — the catalog, the organization rules, the 
tracking of who borrowed what. The books are just the content being managed.

**Git is the same.** Your project files (your code) are the "books." But the **real magic** 
lives in one hidden folder called `.git/`. This folder IS Git. Without it, your project 
folder is just a regular folder. With it, you have version control, history, branches, 
everything.

When you run `git init`, Git creates this hidden `.git/` folder. That's ALL that happens. 
Your existing files aren't touched. Git just sets up its "library system" next to your files.

**Key insight:** If you delete `.git/`, you delete ALL history, ALL branches, ALL tracking. 
Your files remain, but Git is gone. If you copy `.git/` to another machine, you bring the 
entire repository with you — every commit ever made, every branch, everything.

---

## 2. Creating a Git Repository — What Physically Happens

Let's do this step by step. Open your terminal and follow along.

```bash
# Create a brand new empty folder
mkdir my-project
cd my-project

# Look inside — it's empty
ls -la
# Output:
# total 0
# drwxr-xr-x  2 parsh parsh  40 May  2 10:00 .
# drwxr-xr-x 15 parsh parsh 300 May  2 10:00 ..
```

Now, initialize Git:

```bash
git init
```

**What just happened?** Git created ONE folder: `.git/`. Nothing else. Let's see:

```bash
ls -la
# Output:
# total 4
# drwxr-xr-x  3 parsh parsh  60 May  2 10:00 .
# drwxr-xr-x 15 parsh parsh 300 May  2 10:00 ..
# drwxr-xr-x  7 parsh parsh 140 May  2 10:00 .git    ← THIS IS GIT
```

Your project folder now has exactly ONE new thing: the `.git/` directory. That's it. 
That IS your repository.

---

## 3. The Full `.git/` Directory Tree — Every File Explained

Let's look at everything Git created:

```bash
find .git -type f | sort
```

Here is the complete structure of a fresh `.git/` directory:

```
.git/                          ← THE REPOSITORY. Everything Git knows lives here.
│
├── HEAD                       ← "Where am I right now?" (current branch/commit)
│
├── config                     ← Repository-level configuration (local settings)
│
├── description                ← Used by GitWeb (you'll never touch this)
│
├── info/
│   └── exclude                ← Like .gitignore but not committed (personal ignores)
│
├── hooks/                     ← Scripts that run automatically on git events
│   ├── pre-commit.sample      
│   ├── pre-push.sample        
│   ├── commit-msg.sample      
│   └── ... (more .sample files)
│
├── objects/                   ← THE DATABASE. All file content, commits, trees live here.
│   ├── info/                  ← Metadata for the object database
│   └── pack/                  ← Compressed/packed objects (optimization)
│
└── refs/                      ← POINTERS. Branches and tags are just files here.
    ├── heads/                 ← Branch pointers (one file per branch)
    └── tags/                  ← Tag pointers (one file per tag)
```

That's 4 files + 3 important directories. Let's go through each one in detail.

---

## 4. HEAD — "Where Am I Right Now?"

### 4.1 What It Is

`HEAD` is a plain text file. Open it:

```bash
cat .git/HEAD
```

Output on a fresh repo:
```
ref: refs/heads/main
```

That's it. That's the entire file. One line of text.

### 4.2 What It Means

```
ref: refs/heads/main
│     │
│     └── "Go look at the file .git/refs/heads/main for the actual commit SHA"
└── "I'm pointing to a reference (a branch), not directly to a commit"
```

**HEAD answers the question: "Which commit is currently checked out?"**

It does this indirectly:
```
HEAD ──says──► "look at refs/heads/main"
                        │
                        ▼
              .git/refs/heads/main ──contains──► "a3f2c1d8e9..." (a commit SHA)
```

### 4.3 The Two States of HEAD

```
┌─────────────────────────────────────────────────────────────┐
│                      STATE 1: NORMAL                        │
│                   (attached to a branch)                     │
│                                                             │
│  .git/HEAD contains:                                        │
│  ref: refs/heads/main                                       │
│                                                             │
│  Meaning: "I'm on the 'main' branch. When I make a new     │
│  commit, move the 'main' pointer forward."                  │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                    STATE 2: DETACHED                         │
│                (pointing directly to a commit)               │
│                                                             │
│  .git/HEAD contains:                                        │
│  a3f2c1d8e9b4f7a2c8d1e5f6a9b3c7d2e8f4a1b6                  │
│                                                             │
│  Meaning: "I'm NOT on any branch. I'm looking at a         │
│  specific commit directly. New commits won't belong to      │
│  any branch unless I create one."                           │
│                                                             │
│  ⚠️  This is called 'detached HEAD state'                   │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 Prove It Yourself

```bash
# See HEAD right now
cat .git/HEAD
# Output: ref: refs/heads/main

# Switch to a specific commit (detached HEAD)
git checkout a3f2c1d8
cat .git/HEAD
# Output: a3f2c1d8e9b4f7a2c8d1e5f6a9b3c7d2e8f4a1b6

# Go back to a branch (attached HEAD)
git checkout main
cat .git/HEAD
# Output: ref: refs/heads/main
```

💡 **Pro tip:** Every time you run `git checkout <branch>` or `git switch <branch>`, 
Git literally just rewrites this one-line text file.

---

## 5. config — Local Repository Settings

### 5.1 What It Is

```bash
cat .git/config
```

Output on a fresh repo:
```ini
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
```

This is an INI-format configuration file. Let's decode each line:

```
[core]
    repositoryformatversion = 0   ← Format version (0 = standard, compatible)
    filemode = true               ← Track file permission changes (chmod)
    bare = false                  ← This repo has a working directory
                                    (bare = true means no working files, 
                                     like a server-side repo on GitHub)
    logallrefupdates = true       ← Keep a reflog (Topic 20 covers this)
```

### 5.2 What Gets Added Here Over Time

After you add a remote (like GitHub):
```ini
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git@github.com:parsh/my-project.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
```

After you set local configs:
```ini
[user]
	name = Parsh
	email = parsh@company.com
```

### 5.3 Config Hierarchy (Where Git Reads Settings From)

```
PRIORITY (highest to lowest):
─────────────────────────────────────────────────────────

1. .git/config                    ← LOCAL (this repo only)
   Set with: git config --local user.name "Parsh"

2. ~/.gitconfig  (or ~/.config/git/config)  ← GLOBAL (all your repos)
   Set with: git config --global user.name "Parsh"

3. /etc/gitconfig                 ← SYSTEM (all users on this machine)
   Set with: git config --system core.autocrlf true

─────────────────────────────────────────────────────────
If the same key exists at multiple levels, the HIGHEST priority wins.
```

### 5.4 Prove It Yourself

```bash
# See all config and WHERE each value comes from
git config --list --show-origin

# Output (example):
# file:/home/parsh/.gitconfig     user.name=Parsh
# file:/home/parsh/.gitconfig     user.email=parsh@dev.io
# file:.git/config                core.repositoryformatversion=0
# file:.git/config                core.filemode=true
```

---

## 6. objects/ — The Database (Where ALL Content Lives)

### 6.1 What It Is

This is the heart of Git. **Every file you've ever committed, every directory structure, 
every commit message** — they all end up as files inside `.git/objects/`.

On a fresh repo, it's nearly empty:

```bash
find .git/objects -type f
# (no output — empty database)

ls .git/objects/
# Output:
# info/  pack/
```

### 6.2 What Happens When You Add Your First File

```bash
# Create a file in your project
echo "console.log('hello')" > app.js

# Stage it
git add app.js

# NOW look at objects
find .git/objects -type f
# Output:
# .git/objects/0a/4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3
```

**What just happened:**

```
COMMAND: git add app.js

STEP 1 — READ:    Git reads the contents of app.js
                   Content = "console.log('hello')\n"

STEP 2 — COMPUTE: Git calculates SHA-1 hash:
                   SHA-1("blob 21\0console.log('hello')\n") 
                   │       │    │ │
                   │       │    │ └── null byte separator
                   │       │    └── content length in bytes (21)  
                   │       └── object type
                   │
                   Result = "0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3"

STEP 3 — WRITE:   Git creates the file:
                   Path: .git/objects/0a/4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3
                                      ││ │
                                      ││ └── remaining 38 characters of SHA
                                      │└── directory name = first 2 characters of SHA
                                      └── objects database
                   
                   Content: zlib-compressed version of "blob 21\0console.log('hello')\n"

STEP 4 — INDEX:   Git adds an entry to .git/index:
                   "100644 0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3 0\tapp.js"
```

### 6.3 Why the 2-Character Directory Split?

```
.git/objects/
├── 0a/
│   └── 4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3   ← our blob
├── 3b/
│   └── 18e512dba79e4c8300dd08aeb37f8e728b8dad
├── d8/
│   └── 329fc1cc938780ffdd9f94e0d364e0ea74f579
└── ...
```

**Why?** Filesystems slow down when a single directory has too many files. By using 
the first 2 hex characters as a subdirectory name, Git distributes objects across 
256 possible directories (00-ff). A repo with 100,000 objects averages ~390 files 
per directory instead of 100,000 in one folder.

### 6.4 What's Actually Inside an Object File?

The file is NOT plain text. It's **zlib-compressed**. You can't `cat` it directly:

```bash
# This won't work (binary garbage):
cat .git/objects/0a/4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3

# Use git's built-in decompressor:
git cat-file -p 0a4e14e8
# Output: console.log('hello')

git cat-file -t 0a4e14e8
# Output: blob

git cat-file -s 0a4e14e8
# Output: 21  (size in bytes)
```

### 6.5 The Four Types of Objects That Live Here

```
┌─────────────────────────────────────────────────────────────────┐
│              .git/objects/ stores 4 types:                       │
│                                                                  │
│  TYPE      │ WHAT IT STORES           │ ANALOGY                  │
│  ─────────────────────────────────────────────────────────────── │
│  blob      │ Raw file content          │ A single page of a book │
│  tree      │ Directory listing         │ Table of contents       │
│  commit    │ Snapshot + metadata       │ A dated scrapbook entry │
│  tag       │ Named pointer + message   │ A bookmark with a note  │
│                                                                  │
│  (Topic 02 covers each in exhaustive detail)                     │
└─────────────────────────────────────────────────────────────────┘
```

### 6.6 The pack/ Subdirectory

```
.git/objects/pack/
```

On a fresh repo, this is empty. Over time (or after `git gc`, or after `git clone`), 
Git **packs** many loose object files into a single compressed `.pack` file for efficiency.

```
Before packing:                    After packing:
.git/objects/                      .git/objects/
├── 0a/4e14e8...                   ├── info/
├── 3b/18e512...                   │   └── packs
├── d8/329fc1...                   └── pack/
├── 9f/2d14a3...                       ├── pack-abc123.idx  ← index (lookup table)
├── ...  (thousands of files)          └── pack-abc123.pack ← all objects compressed
└── (slow filesystem)                      (fast, single file)
```

💡 **When you `git clone`, you receive a pack file.** That's why clone downloads 
one big file rather than millions of small ones. Topic 25 covers this in full depth.

---

## 7. refs/ — Branch and Tag Pointers

### 7.1 What It Is

```
.git/refs/
├── heads/          ← ONE FILE PER BRANCH
│   ├── main        ← contains the SHA of the latest commit on 'main'
│   ├── feature/auth ← contains the SHA of the latest commit on 'feature/auth'
│   └── develop     ← contains the SHA of the latest commit on 'develop'
│
├── tags/           ← ONE FILE PER TAG
│   ├── v1.0.0      ← contains the SHA of the tagged commit (or tag object)
│   └── v2.0.0
│
└── remotes/        ← Remote-tracking branches (created after fetch/clone)
    └── origin/
        ├── main    ← last known position of origin's main branch
        └── develop ← last known position of origin's develop branch
```

### 7.2 A Branch is Literally a Text File

```bash
# After making a commit on 'main':
cat .git/refs/heads/main
# Output: 9f2d14a3b7c8e1f5d6a2b9c4e8f3a7d1c5b6e2f4

# That's it. That's a branch. 40 characters in a text file.
```

### 7.3 Creating a Branch — What Physically Happens

```bash
git branch feature/login
```

```
COMMAND: git branch feature/login

READS:    .git/HEAD → "ref: refs/heads/main"
          .git/refs/heads/main → "9f2d14a3..."

WRITES:   .git/refs/heads/feature/login
          Content: "9f2d14a3..." (same SHA as main — they point to the same commit)

TOTAL WORK: Created one 41-byte text file. Done.
```

**Prove it:**
```bash
# Create the branch
git branch feature/login

# See the new file
cat .git/refs/heads/feature/login
# Output: 9f2d14a3b7c8e1f5d6a2b9c4e8f3a7d1c5b6e2f4

# Compare to main — same SHA!
cat .git/refs/heads/main
# Output: 9f2d14a3b7c8e1f5d6a2b9c4e8f3a7d1c5b6e2f4
```

### 7.4 Switching Branches — What Physically Happens

```bash
git checkout feature/login
# (or: git switch feature/login)
```

```
COMMAND: git switch feature/login

READS:    .git/refs/heads/feature/login → "9f2d14a3..."
          .git/objects/... (the commit, then its tree, then its blobs)

WRITES:   .git/HEAD
          New content: "ref: refs/heads/feature/login"

ALSO:     Updates working directory files to match the tree of that commit
          Updates .git/index to match the tree of that commit

TOTAL WORK: Rewrote HEAD (one line), possibly updated working directory files.
```

**Prove it:**
```bash
git switch feature/login
cat .git/HEAD
# Output: ref: refs/heads/feature/login
```

### 7.5 Visual Summary of refs/

```
                    .git/HEAD
                    "ref: refs/heads/main"
                         │
                         ▼
            .git/refs/heads/main ──────► "9f2d14a3..." ──► COMMIT OBJECT
                                                               │
            .git/refs/heads/feature ───► "c4a8b2e1..." ──► COMMIT OBJECT
                                                               │
            .git/refs/tags/v1.0.0 ─────► "7c3e8f9d..." ──► COMMIT OBJECT
                                                               │
            .git/refs/remotes/origin/main► "9f2d14a3..."──► COMMIT OBJECT

All of these are just text files containing a 40-char SHA.
Branches, tags, remote tracking — all just pointer files.
```

---

## 8. index — The Staging Area (Binary File)

### 8.1 What It Is

```bash
ls -la .git/index
# This file appears AFTER your first 'git add'
```

The **index** (also called the "staging area" or "cache") is a **binary file** that 
contains a sorted list of:
- File paths
- Their corresponding blob SHA
- File metadata (permissions, timestamps, size)

### 8.2 What It Looks Like (Decoded)

You can't read it directly (it's binary), but you can inspect it:

```bash
git ls-files --stage
```

Output after staging some files:
```
100644 0a4e14e8d2c6b4f3a9e1b7c5d8f2a6e9c4b1d7f3 0	app.js
100644 b7e9f0a3c8d1e5f6a9b2c4d7e1f3a8b5c6d2e9f4 0	package.json
100644 f2a9b3c7d4e8f1a6b5c2d9e3f7a4b8c1d6e5f9a2 0	routes/users.js
```

Breakdown:
```
100644  0a4e14e8...  0       app.js
│       │            │       │
│       │            │       └── file path (relative to repo root)
│       │            └── stage number (0=normal, 1/2/3=during merge conflicts)
│       └── SHA of the blob object containing this file's content
└── file mode (100644 = regular file, 100755 = executable, 120000 = symlink)
```

### 8.3 How the Index Relates to Everything Else

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   WORKING DIRECTORY │     │      INDEX           │     │    REPOSITORY       │
│   (your actual      │     │   (.git/index)       │     │  (.git/objects/)    │
│    files on disk)   │     │                      │     │                     │
│                     │     │  Sorted list of:     │     │  Permanent store    │
│  app.js             │────►│  path → blob SHA     │────►│  of blob/tree/      │
│  package.json       │ add │  path → blob SHA     │commit│  commit objects    │
│  routes/users.js    │     │  path → blob SHA     │     │                     │
│                     │     │                      │     │                     │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
         │                            │                           │
         └── What you see and edit    └── "What will go in my     └── Permanent 
                                           next commit"               history
```

💡 **The index is the "staging area."** When you run `git add file.js`, Git:
1. Computes the blob for `file.js`
2. Stores it in `.git/objects/`
3. Updates `.git/index` to point to that blob

When you run `git commit`, Git:
1. Reads `.git/index`
2. Creates tree objects from it
3. Creates a commit object pointing to the root tree
4. Done — the index becomes the basis for the new commit

Topic 05 and 06 cover this flow in full detail.

---

## 9. hooks/ — Automation Scripts

### 9.1 What It Is

```bash
ls .git/hooks/
# Output:
# applypatch-msg.sample     pre-commit.sample
# commit-msg.sample         pre-merge-commit.sample
# fsmonitor-watchman.sample pre-push.sample
# post-update.sample        pre-rebase.sample
# pre-applypatch.sample     prepare-commit-msg.sample
# pre-commit.sample         update.sample
```

These are **sample scripts** that Git provides. They do NOTHING by default because 
they have the `.sample` extension. To activate one:

```bash
# Remove .sample extension and make it executable
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

Now, every time you run `git commit`, Git will execute `.git/hooks/pre-commit` FIRST. 
If the script exits with a non-zero code, the commit is **blocked**.

### 9.2 Common Use Cases

```
Hook Name         │ When It Runs                  │ Use Case
──────────────────┼───────────────────────────────┼──────────────────────
pre-commit        │ Before commit is created      │ Run linter, formatter
commit-msg        │ After you write the message   │ Enforce commit message format
pre-push          │ Before push to remote         │ Run tests
post-merge        │ After a merge completes       │ Install deps (npm install)
```

Topic 19 covers hooks in full depth. For now, just know: they're scripts in `.git/hooks/`.

---

## 10. info/exclude — Personal .gitignore

```bash
cat .git/info/exclude
```

Output:
```
# git ls-files --others --exclude-from=.git/info/exclude
# Lines that start with '#' are comments.
# For a project-specific exclude file, see .gitignore
```

This works exactly like `.gitignore` but:
- It is NOT committed (it's inside `.git/`, which is never committed)
- Only affects YOUR local copy
- Use it for personal editor files, scratch notes, etc. that only you have

Example — add your personal ignores:
```bash
echo ".my-scratch-notes" >> .git/info/exclude
echo ".idea/" >> .git/info/exclude
```

---

## 11. description — You Can Ignore This

```bash
cat .git/description
# Output: Unnamed repository; edit this file 'description' to name the repository.
```

This is only used by **GitWeb** (an old web interface for Git). You will never need 
to touch this file in modern development. GitHub, GitLab, etc. don't use it.

---

## 12. The Bigger Picture — How It All Connects

After a few commits and a branch, here's what your `.git/` looks like:

```
.git/
├── HEAD                          ← "ref: refs/heads/main"
├── config                        ← [core], [remote "origin"], [branch "main"]
├── description                   ← (unused)
├── index                         ← Binary: sorted list of staged file paths → blob SHAs
│
├── hooks/                        ← Automation scripts (inactive by default)
│   └── pre-commit.sample         
│
├── info/
│   └── exclude                   ← Personal ignore rules
│
├── logs/                         ← REFLOG (Topic 20) — history of all ref movements
│   ├── HEAD                      ← Every time HEAD moved (checkout, commit, reset...)
│   └── refs/
│       └── heads/
│           └── main              ← Every time 'main' pointer moved
│
├── objects/                      ← ALL CONTENT LIVES HERE
│   ├── 0a/4e14e8...             ← blob (file content)
│   ├── 3b/18e512...             ← blob (another file)
│   ├── c4/d1e8a2...             ← tree (directory listing)
│   ├── d9/f3a2b7...             ← tree (root directory)
│   ├── 9f/2d14a3...             ← commit (snapshot #1)
│   ├── e7/b4c8f1...             ← commit (snapshot #2)
│   ├── info/
│   └── pack/                    ← Packed objects (after gc or clone)
│
├── refs/                         ← POINTER FILES
│   ├── heads/
│   │   ├── main                 ← "9f2d14a3..." (latest commit on main)
│   │   └── feature/auth         ← "e7b4c8f1..." (latest commit on feature)
│   ├── tags/
│   │   └── v1.0.0               ← "9f2d14a3..." (tagged commit)
│   └── remotes/
│       └── origin/
│           └── main             ← "9f2d14a3..." (last known remote main)
│
├── COMMIT_EDITMSG               ← Your last commit message (for editor reuse)
├── FETCH_HEAD                   ← Result of last 'git fetch'
├── ORIG_HEAD                    ← HEAD before last "dangerous" operation (merge/reset)
└── packed-refs                  ← Packed version of refs/ (optimization for many refs)
```

---

## 13. Data Flow Diagram — How Commands Touch .git/

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  YOUR WORKING DIRECTORY              .git/ (THE REPOSITORY)             │
│  ┌─────────────────────┐            ┌──────────────────────────────┐   │
│  │                     │            │                              │   │
│  │  app.js             │  git add   │  objects/                    │   │
│  │  package.json       │──────────► │    ├── blobs (file content)  │   │
│  │  routes/users.js    │            │    ├── trees (dir listings)  │   │
│  │                     │            │    └── commits (snapshots)   │   │
│  └─────────────────────┘            │                              │   │
│           ▲                         │  index (staging area)        │   │
│           │                         │    path → blob SHA list      │   │
│           │                         │                              │   │
│           │         git checkout    │  refs/heads/                  │   │
│           │◄──────────────────────  │    main → commit SHA         │   │
│           │                         │    feature → commit SHA       │   │
│  (Git writes files to match        │                              │   │
│   the tree of the target commit)   │  HEAD                        │   │
│                                     │    → current branch or SHA    │   │
│                                     │                              │   │
│                                     └──────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

SUMMARY:
  git add       → Writes to objects/ + updates index
  git commit    → Reads index → writes tree+commit to objects/ → updates refs/heads/
  git checkout  → Reads refs/ + objects/ → updates HEAD + index + working directory
  git branch    → Writes one file to refs/heads/
  git tag       → Writes one file to refs/tags/
  git log       → Reads commits from objects/ by following parent pointers
```

---

## 14. Beginner Example — Walk Through a Full Cycle

Let's watch `.git/` change at every step:

```bash
# 1. INIT — Create the repo
mkdir demo && cd demo && git init

# State of .git/:
#   HEAD: "ref: refs/heads/main"
#   objects/: empty
#   refs/heads/: empty (no commits yet!)
#   index: does not exist yet
```

```bash
# 2. CREATE A FILE
echo "hello" > hello.txt

# State of .git/: UNCHANGED.
# Git doesn't know about hello.txt yet. It's just a file on disk.
```

```bash
# 3. STAGE THE FILE
git add hello.txt

# State of .git/:
#   objects/ce/013625030ba8dba906f756967f9e9ca394464a  ← NEW blob
#   index: NOW EXISTS — contains "100644 ce013625... 0 hello.txt"
#
# PROVE IT:
git cat-file -t ce0136    # Output: blob
git cat-file -p ce0136    # Output: hello
git ls-files --stage      # Output: 100644 ce013625... 0  hello.txt
```

```bash
# 4. COMMIT
git commit -m "first commit"

# State of .git/:
#   objects/ce/013625...   ← blob (still here)
#   objects/XX/YYYYYY...   ← NEW tree (contains: "blob ce013625 hello.txt")
#   objects/ZZ/WWWWWW...   ← NEW commit (tree=XX, parent=none, message="first commit")
#   refs/heads/main: NOW EXISTS — contains the commit SHA "ZZWWWWWW..."
#   HEAD: still "ref: refs/heads/main" (unchanged)
#
# PROVE IT:
cat .git/refs/heads/main           # Shows the commit SHA
git cat-file -p HEAD               # Shows: tree XX, author, message
git cat-file -p $(git cat-file -p HEAD | head -1 | cut -d' ' -f2)  # Shows tree contents
```

```bash
# 5. CREATE A BRANCH
git branch feature

# State of .git/:
#   refs/heads/feature: NOW EXISTS — contains same SHA as refs/heads/main
#
# PROVE IT:
cat .git/refs/heads/feature
cat .git/refs/heads/main
# They're the same!
```

```bash
# 6. SWITCH BRANCH
git switch feature

# State of .git/:
#   HEAD: changed from "ref: refs/heads/main" to "ref: refs/heads/feature"
#   Everything else: UNCHANGED (both branches point to same commit)
#
# PROVE IT:
cat .git/HEAD    # Output: ref: refs/heads/feature
```

```bash
# 7. MODIFY AND COMMIT ON FEATURE BRANCH
echo "world" >> hello.txt
git add hello.txt
git commit -m "add world"

# State of .git/:
#   objects/: 2 NEW objects (new blob for modified hello.txt, new tree, new commit)
#   refs/heads/feature: UPDATED to new commit SHA
#   refs/heads/main: UNCHANGED (still points to first commit)
#   HEAD: unchanged ("ref: refs/heads/feature")
#
# PROVE IT:
cat .git/refs/heads/feature    # New SHA
cat .git/refs/heads/main       # Old SHA — they diverged!
```

Visual after step 7:

```
         HEAD
          │
          ▼
        feature
          │
          ▼
  C1 ◄── C2
  │
  ▼
 main

.git/refs/heads/main    → SHA of C1
.git/refs/heads/feature → SHA of C2
.git/HEAD               → "ref: refs/heads/feature"
```

---

## 15. Real-World Example — Backend Developer Scenario

> **Scenario:** You join a new company. Day 1, you clone the company's Node.js 
> backend repo. You want to understand what you just downloaded.

```bash
git clone git@github.com:company/api-service.git
cd api-service
```

Now let's explore what you have:

```bash
# How many objects (files, commits, trees) in this repo's history?
git count-objects -v
# Output:
# count: 0          ← loose objects (0 because everything is packed after clone)
# size: 0
# in-pack: 24831    ← 24,831 objects in pack files!
# packs: 1
# size-pack: 8420   ← 8.4 MB total compressed history

# What branches exist?
ls .git/refs/heads/
# Output: main  (just main locally after clone)

# What remote branches were fetched?
ls .git/refs/remotes/origin/
# Output: HEAD  main  develop  feature/payments  release/v2.3

# Where does 'origin' point to?
git remote -v
# OR just read the config:
grep -A2 'remote "origin"' .git/config
# Output:
# [remote "origin"]
#     url = git@github.com:company/api-service.git
#     fetch = +refs/heads/*:refs/remotes/origin/*

# How many commits on main?
git rev-list --count main
# Output: 1847
```

**You now understand:** The clone gave you:
1. A `.git/` folder with all 24,831 objects (entire history) in a pack file
2. One local branch (`main`) tracking `origin/main`
3. Remote-tracking refs for all the remote's branches
4. A working directory with files matching the latest commit on `main`

---

## 16. Common Mistakes

### ❌ Mistake 1: Manually editing files inside .git/

**The mistake:** "I'll just edit `.git/refs/heads/main` to change what commit main points to."

**Why it's dangerous:** It works! But if you typo a SHA, or point to a non-existent 
object, you corrupt your repo. Use `git update-ref` or `git reset` instead.

**The fix:** Use proper Git commands:
```bash
# Instead of: echo "abc123" > .git/refs/heads/main
# Use:
git update-ref refs/heads/main abc123
```

---

### ❌ Mistake 2: Thinking `.git/` is just metadata

**The mistake:** "I'll delete `.git/` and re-init to fix a problem."

**What you lose:** EVERYTHING. All history, all branches, all stashes, all reflog. 
Your working directory files survive but all Git knowledge is gone.

**Better approach:** Almost any Git problem can be fixed without deleting `.git/`:
- Wrong branch? → `git checkout`
- Bad commits? → `git reset` or `git revert`
- Total disaster? → `git reflog` (Topic 20)

---

### ❌ Mistake 3: Committing the .git/ folder into another repo

**The mistake:** Copying a project into a subdirectory of another project, including its `.git/`.

**What happens:** Git gets confused by nested `.git/` directories. The inner folder 
won't be tracked properly.

**The fix:** If you need to include another repo, use `git submodule` (Topic 23) or 
simply delete the inner `.git/` before adding.

---

### ❌ Mistake 4: Not understanding that .git/ IS the repo

**The mistake:** Thinking the repo lives "on GitHub" and your local copy is just a checkout.

**The truth:** Your `.git/` folder IS a complete, independent repository. It has the 
FULL history. GitHub is just another copy. You can work completely offline — commit, 
branch, merge, log — because everything is in `.git/`.

---

## 17. What Actually Happens — Deep Internals

### 17.1 The SHA-1 Computation (Exact Formula)

Every Git object's filename is computed the same way:

```
SHA-1( "<type> <content-length-in-bytes>\0<content-bytes>" )
         │            │                 │        │
         │            │                 │        └── raw content
         │            │                 └── null byte (0x00) separator
         │            └── decimal string of content byte count
         └── "blob", "tree", "commit", or "tag"
```

**Example for a blob:**
```
File content: "hello\n" (6 bytes including newline)

Git computes: SHA-1("blob 6\0hello\n")
Result: ce013625030ba8dba906f756967f9e9ca394464a
```

**Prove it yourself (using shell tools):**
```bash
echo "hello" > hello.txt

# Method 1: Ask Git
git hash-object hello.txt
# Output: ce013625030ba8dba906f756967f9e9ca394464a

# Method 2: Compute manually (Linux/Mac)
printf "blob 6\0hello\n" | sha1sum
# Output: ce013625030ba8dba906f756967f9e9ca394464a  -

# They match! Git is just SHA-1 of "type size\0content"
```

### 17.2 Object Storage Format (On Disk)

```
What's in .git/objects/ce/013625030ba8dba906f756967f9e9ca394464a:

┌──────────────────────────────────────────────────────────┐
│  zlib_compress("blob 6\0hello\n")                        │
│                                                          │
│  Decompressed structure:                                 │
│  ┌─────────┬───┬───────────────┬────┬─────────────────┐ │
│  │  "blob" │" "│     "6"       │\0  │   "hello\n"     │ │
│  │  (type) │   │(content size) │null│  (raw content)  │ │
│  └─────────┴───┴───────────────┴────┴─────────────────┘ │
│                                                          │
│  File path determined by SHA:                            │
│  SHA = ce013625030ba8dba906f756967f9e9ca394464a           │
│         ││                                               │
│         │└── objects/ce/013625030ba8dba906f756967f...     │
│         └── directory                                    │
└──────────────────────────────────────────────────────────┘
```

### 17.3 Why Content-Addressable?

The filename IS a hash of the content. This gives you:

1. **Deduplication** — Same content = same hash = stored only once
2. **Integrity** — If a bit flips on disk, the hash won't match, Git detects corruption
3. **Immutability** — You can't change an object without changing its name (hash)
4. **Efficient comparison** — Two trees with same hash? They're identical, skip them.

```
Repo with 100 branches but same package-lock.json:
  → Only ONE blob stored for package-lock.json, shared by all branches

Repo with 10,000 commits but README never changed:
  → Only ONE blob for README across entire history
```

---

## 18. logs/ — The Reflog (Preview)

```
.git/logs/
├── HEAD                    ← Every movement of HEAD (checkout, commit, reset, merge...)
└── refs/
    └── heads/
        ├── main            ← Every movement of the 'main' pointer
        └── feature/auth    ← Every movement of this branch pointer
```

```bash
cat .git/logs/HEAD
# Output (one line per HEAD movement):
# 0000000 ce01362 Parsh <p@dev.io> 1746134400 +0530	commit (initial): first commit
# ce01362 a8f9b2c Parsh <p@dev.io> 1746134460 +0530	commit: add world
# a8f9b2c ce01362 Parsh <p@dev.io> 1746134520 +0530	checkout: moving from feature to main
```

Each line format:
```
<old-SHA> <new-SHA> <who> <when>    <what-happened>
```

This is your safety net — even if you delete a branch or reset to the wrong commit, 
the reflog remembers where everything was. Topic 20 covers this in full depth.

---

## 19. Mental Model Checkpoint

**Can you answer these from memory? If not, re-read the relevant section.**

1. What single folder makes a directory a "Git repository"?
2. What does the file `.git/HEAD` contain when you're on branch `develop`?
3. What does `.git/HEAD` contain in "detached HEAD" state?
4. Where does Git store the actual content of your files?
5. What is a branch, physically? (What file, where, containing what?)
6. When you run `git add file.js`, which parts of `.git/` change?
7. When you run `git commit`, which parts of `.git/` change?
8. Why are objects stored in subdirectories named with 2-character hex prefixes?
9. What's the difference between `.git/info/exclude` and `.gitignore`?
10. If you delete `.git/refs/heads/feature`, what happens to the commits on that branch?

---

## 20. Quick Reference Card

| What | Where in .git/ | Content |
|---|---|---|
| Current branch/commit | `HEAD` | `ref: refs/heads/<branch>` or raw SHA |
| Local config | `config` | INI format: remotes, branches, settings |
| Branch pointer | `refs/heads/<name>` | 40-char SHA of latest commit |
| Tag pointer | `refs/tags/<name>` | 40-char SHA of commit or tag object |
| Remote tracking | `refs/remotes/<remote>/<branch>` | 40-char SHA |
| Object (blob/tree/commit) | `objects/<first-2-chars>/<remaining-38>` | zlib-compressed content |
| Packed objects | `objects/pack/*.pack` + `.idx` | Many objects in one file |
| Staging area | `index` | Binary: sorted file-path → blob-SHA mappings |
| Reflog | `logs/HEAD` and `logs/refs/...` | History of pointer movements |
| Hooks | `hooks/<hook-name>` | Executable scripts (no `.sample` extension) |
| Personal ignores | `info/exclude` | Same syntax as .gitignore |

---

## 21. When Would I Use This At Work?

```
╔══════════════════════════════════════════════════════════════════════╗
║              WHEN WOULD I USE THIS AT WORK?                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. DEBUGGING "CORRUPTED" REPOS IN CI/CD                            ║
║     CI job fails with "fatal: bad object HEAD". You SSH in,          ║
║     cat .git/HEAD, see it points to a ref that doesn't exist.        ║
║     Fix: git update-ref or re-fetch.                                 ║
║                                                                      ║
║  2. UNDERSTANDING WHY `git clone` IS SLOW                            ║
║     You check: git count-objects -v → see 500MB in pack files.       ║
║     You know the entire history is in .git/objects/pack/.            ║
║     Solution: shallow clone (--depth=1) for CI.                      ║
║                                                                      ║
║  3. RECOVERING FROM "I DELETED MY BRANCH"                           ║
║     Teammate deleted a branch. You know: only the pointer file       ║
║     (refs/heads/X) was removed. Commits still exist in objects/.     ║
║     Recovery: check .git/logs/ (reflog) for the last SHA.            ║
║                                                                      ║
║  4. SETTING UP REPO-SPECIFIC CONFIG                                  ║
║     Company uses one email for work repos, personal for side projects.║
║     You set user.email in .git/config (local) per-repo.              ║
║     git config --local user.email "parsh@company.com"                ║
║                                                                      ║
║  5. UNDERSTANDING BARE REPOS (SERVERS)                               ║
║     GitHub/GitLab store repos as "bare" (no working directory).      ║
║     A bare repo IS just the .git/ folder contents directly.          ║
║     Now you know what the server has: objects + refs + config.        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Up Next:** [Topic 02 — Git Objects Deep Dive](./topic-02-git-objects-deep-dive.md)
> We'll create blobs, trees, commits, and tags BY HAND using plumbing commands,
> and see exactly how they're structured byte-by-byte on disk.
