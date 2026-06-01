# Topic 26 — Performance & Large Repos
## Shallow Clone · Sparse Checkout · Partial Clone · git filter-repo · Git LFS

---

## RULE 1 — ELI5 Anchor

Imagine you're moving into a new apartment. The landlord hands you a master key ring 
with keys to every room that has existed in the building since 1985 — all 40,000 rooms,
most of which you'll never visit, many of which no longer exist.

That's what `git clone` does by default: it gives you the *entire history* of every file
that has ever existed in the repository. For a 10-year-old codebase with design assets,
video files, and old database dumps, that might mean downloading 30 GB just to fix a 
one-line bug.

Git's performance toolkit gives you four different ways to say "I only need SOME of that":

| Problem | Solution | What You Skip |
|---|---|---|
| "I only need recent history, not all 10 years" | **Shallow clone** (`--depth=N`) | Old commits (history truncated) |
| "I only need 3 of the 50 directories" | **Sparse checkout** | Most of the working tree files |
| "I only need commits, not all blobs/trees" | **Partial clone** (`--filter=`) | Tree and blob objects (fetched on demand) |
| "The large binary files are bloating git" | **Git LFS** | Large binary content (stored outside git) |
| "Someone committed giant files into history" | **git filter-repo** | Rewrites history to remove them |

These aren't hacks — they're first-class Git features used by Microsoft (Windows repo: 
300 GB, 450,000 files), Google, and any company running a serious monorepo.

---

## RULE 2 — Physical Architecture First

### Shallow Clone: What `.git/` Looks Like

```
FULL CLONE:                           SHALLOW CLONE (--depth=3):

.git/                                 .git/
├── objects/                          ├── objects/
│   └── pack/                         │   └── pack/
│       ├── pack-aaa.pack             │       ├── pack-bbb.pack  ← smaller pack!
│       └── pack-aaa.idx              │       └── pack-bbb.idx
│                                     ├── shallow             ← NEW FILE (doesn't exist in full clone)
│                                     │     Contains the SHA(s) of the "boundary" commits
│                                     │     These are treated as root commits (no parents fetched)
│                                     │     Example content:
│                                     │       "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\n"
│                                     │       "f1e2d3c4b5a6f7e8d9c0b1a2f3e4d5c6b7a8f9e0\n"
├── refs/                             ├── refs/
│   └── heads/main                    │   └── heads/main
├── FETCH_HEAD                        ├── FETCH_HEAD
└── config:                           └── config:
    [remote "origin"]                     [remote "origin"]
      url = ...                             url = ...
      fetch = +refs/heads/*:refs/...        fetch = +refs/heads/*:refs/...
                                            shallow = true         ← new entry

WHAT THE .git/shallow FILE MEANS:
  Each SHA listed is treated as if it has NO parents.
  This is a LIE to git — those commits DO have parents on the remote,
  but the local repo pretends they don't, preventing history traversal
  beyond that point.
```

### Sparse Checkout: Working Tree vs Index

```
FULL CHECKOUT:                         SPARSE CHECKOUT:

Working directory:                     Working directory:
├── apps/                              ├── apps/
│   ├── web/          (5,000 files)    │   └── api/           ← only this dir
│   ├── mobile/       (3,000 files)    │       └── src/
│   └── api/          (1,200 files)    │           └── ...    (1,200 files)
├── packages/                          ├── packages/
│   ├── ui/           (800 files)      │   └── (empty)
│   ├── utils/        (200 files)
│   └── config/       (50 files)
├── infra/            (400 files)
└── docs/             (600 files)
Total: ~11,250 files                   Total: ~1,200 files (89% smaller!)

.git/index:                            .git/index:
  ALL files registered                   ALL files registered (same as full!)
  (100644 sha path for every file)       BUT: skip-worktree bit set on excluded files
                                         (sparse entries don't appear in working dir,
                                          but git still "knows" about them)

.git/info/sparse-checkout:             .git/info/sparse-checkout:  (cone mode OFF)
  /apps/api/**                            /apps/api/
  /(nothing else)

.git/config addition:
  [core]
    sparseCheckout = true
    sparseCheckoutCone = true          ← cone mode: faster pattern matching
```

### Partial Clone: The Missing Objects Mechanism

```
PARTIAL CLONE (--filter=blob:none):
  Downloads: all commits + all tree objects + ZERO blob objects at clone time
  Blobs fetched: on demand, the first time you access a file

.git/
├── objects/
│   └── pack/
│       ├── pack-partial.pack    ← contains: commits + trees, NO blobs
│       └── pack-partial.idx
├── config:
    [remote "origin"]
      url = ...
      fetch = +refs/heads/*:refs/...
      partialclonefilter = blob:none   ← NEW: tells git what was excluded
      promisor = true                  ← NEW: remote "promises" to serve missing objects

WHAT HAPPENS WHEN YOU ACCESS A FILE:
  git checkout HEAD -- src/app.js
  │
  ├── Git looks up the blob SHA for src/app.js in the tree object
  ├── Blob SHA not in local objects → "promisor remote" lookup
  ├── git fetch origin <blob-sha>      ← lazy fetch of just this one blob
  └── File appears in working directory

This is called "lazy loading" or "on-demand object fetching"
```

### Git LFS: Pointer Files and the LFS Store

```
WITHOUT LFS:                           WITH LFS:

.git/objects/.../                      .git/objects/.../
  blob: 104857600 bytes (100 MB)         blob: 134 bytes (pointer file!)
  (the actual video content)             (a text file pointing to LFS)

Working directory:                     Working directory:
  video.mp4  (100 MB)                    video.mp4  (100 MB ← downloaded from LFS server)

Repository content of video.mp4:       Repository content of video.mp4:
  <binary video data>                    version https://git-lfs.github.com/spec/v1
                                         oid sha256:4d7a214614ab2...  (64 hex chars)
                                         size 104857600
                                         (134 bytes total — this IS the blob in git)

LFS OBJECT STORE (separate from git):
  ~/.git/lfs/objects/                  ← local LFS cache
    4d/7a/
      4d7a214614ab2935c943f9e0ff69d22eadbb8f...  ← actual 100 MB file

  OR on GitHub's servers:
    https://github.com/org/repo.git/info/lfs/objects/batch  ← LFS API endpoint

.gitattributes (committed to repo):
  *.mp4 filter=lfs diff=lfs merge=lfs -text
  *.psd filter=lfs diff=lfs merge=lfs -text
  (these attributes tell git to run the LFS filter for these files)
```

### git filter-repo: Before and After

```
BEFORE filter-repo (repo has video.mp4 in all history):

Commit A  →  Commit B  →  Commit C  →  Commit D (HEAD)
  tree            tree            tree            tree
   └─ video.mp4    └─ video.mp4    └─ video.mp4    └─ video.mp4
      (100 MB)        (102 MB)        (98 MB)         (105 MB)
      blob a          blob b          blob c          blob d

.git/ total: 405 MB (4 versions of a video)

AFTER git filter-repo --path video.mp4 --invert-paths:
  ALL commits rewritten with NEW SHAs (because tree changes → commit SHA changes)

Commit A'  →  Commit B'  →  Commit C'  →  Commit D' (HEAD)
  tree'           tree'           tree'           tree'
   (no video.mp4)  (no video.mp4)  (no video.mp4)  (no video.mp4)

.git/ total: 1.2 MB (video blobs are now unreachable → pruned)

IMPORTANT:
  Old SHAs (A, B, C, D) no longer exist.
  Anyone with a clone of the old repo has different history.
  A force push is REQUIRED. ALL clones must be re-done.
```

---

## RULE 3 — Exact Data Flow

### Data Flow 1: `git clone --depth=3`

```
COMMAND: git clone --depth=3 https://github.com/org/repo.git

─────────────────────────────────────────────────────────────────────
STEP 1: Client connects to remote, starts capability negotiation
        Client sends: "I support shallow clones (deepen, unshallow)"

STEP 2: Client sends WANT + SHALLOW request
        "want <HEAD-sha>"
        "deepen 3"          ← I want 3 commits deep

STEP 3: Server computes the shallow boundary
        Server walks: HEAD → parent → parent → parent
        At depth=3, notes the boundary commits
        Server sends back: "shallow <sha-of-commit-at-depth-3>"
        (This tells the client "this SHA has no parents in my response")

STEP 4: Server packs and sends objects
        Included: the 3 most recent commits + their trees + their blobs
        Excluded: all commits older than depth=3 AND their blobs/trees
        (blobs/trees shared between the 3 commits are sent only once)

STEP 5: Client receives the pack
        Writes: .git/objects/pack/pack-NNN.pack
                .git/objects/pack/pack-NNN.idx

STEP 6: Client writes .git/shallow
        Content: the SHA(s) of the boundary commits
        These commits are "grafted" — treated as root commits

STEP 7: Client checks out working directory
        Normal checkout (HEAD → tree → blobs) — all needed blobs exist

─────────────────────────────────────────────────────────────────────
RESULT:
  git log --oneline → shows exactly 3 commits
  git log --oneline -5 → still shows 3 (can't go deeper than the shallow boundary)
  .git/shallow exists with the boundary SHA
  git rev-parse --is-shallow-repository → "true"
```

### Data Flow 2: `git fetch --unshallow` (Deepening a Shallow Clone)

```
COMMAND: git fetch --unshallow

STEP 1: Client reads .git/shallow → knows which SHAs are shallow boundaries
STEP 2: Client sends HAVE list + requests to "unshallow" those SHAs
        "unshallow <sha1>"
        "unshallow <sha2>"
STEP 3: Server sends all missing commits, trees, blobs (back to the root)
STEP 4: Client receives and integrates the full history pack
STEP 5: Client DELETES .git/shallow (repo is now a full clone)

ALSO: git fetch --deepen=N  (add N more commits without going full)
      git fetch --deepen=50 → "I had 3 commits, now give me 53"
      git fetch --shallow-since=2024-01-01  → get all commits since a date
      git fetch --shallow-exclude=<tag>     → deepen up to (but not including) a tag
```

### Data Flow 3: `git sparse-checkout` (Cone Mode)

```
COMMAND: git sparse-checkout init --cone
         git sparse-checkout set apps/api packages/utils

─────────────────────────────────────────────────────────────────────
STEP 1: git sparse-checkout init --cone
        WRITES: .git/config:  [core] sparseCheckout = true
                               [core] sparseCheckoutCone = true
        WRITES: .git/info/sparse-checkout (initially just "/*\n!/*/*\n")
                (cone mode: include root files, exclude all subdirs)

STEP 2: git sparse-checkout set apps/api packages/utils
        WRITES: .git/info/sparse-checkout:
                  /*
                  !/*/
                  /apps/
                  !/apps/*/
                  /apps/api/
                  /packages/
                  !/packages/*/
                  /packages/utils/
                (cone mode generates this pattern automatically)

STEP 3: Git updates the working tree
        For each file in index:
          IF the file path matches cone patterns → ensure it exists on disk
          IF the file path doesn't match → SET skip-worktree bit in index
                                            DELETE file from working directory

STEP 4: Working tree now contains only apps/api/ and packages/utils/
        The index still has ALL entries — just most have skip-worktree bit set

─────────────────────────────────────────────────────────────────────
THE skip-worktree BIT:
  In .git/index, each entry has status flags. Bit 23 (CE_SKIP_WORKTREE):
    0 = normal: file should match index
    1 = sparse: git ignores the absence of this file on disk
                git won't show it as "deleted"
                git won't try to update it on checkout

  This is DIFFERENT from .gitignore (which ignores untracked files)
  and DIFFERENT from assume-unchanged (which is about performance, not sparse checkout)
```

### Data Flow 4: Git LFS — What Happens on `git push`

```
COMMAND: git add video.mp4
         git commit -m "add video"
         git push origin main

─────────────────────────────────────────────────────────────────────
STEP 1: git add video.mp4
        Git LFS INTERCEPTS via the "clean" filter (configured in .gitattributes)
        
        CLEAN FILTER runs:
          Input:  the actual video.mp4 binary content
          Computes: SHA256 of the raw file
          Writes: .git/lfs/objects/4d/7a/4d7a214614ab...  ← actual video stored in LFS cache
          Output: the 134-byte pointer file:
                    version https://git-lfs.github.com/spec/v1
                    oid sha256:4d7a214614ab2935c943f9e0...
                    size 104857600
        
        Git stores the POINTER (134 bytes) as the blob, NOT the 100 MB binary.

STEP 2: git commit -m "add video"
        Normal commit — the pointer file is committed.
        Blob SHA in the commit tree: SHA-1("blob 134\0version https://...") → some short SHA

STEP 3: git push origin main
        Git LFS intercept: BEFORE sending the git pack, LFS uploads large objects:
        
        LFS PRE-PUSH HOOK runs:
          Reads: commits being pushed
          Finds: all pointer files in new commits
          For each pointer:
            Calls: GitHub LFS API to check if object already exists
            If not: uploads the actual file to LFS storage
                    POST https://github.com/org/repo.git/info/lfs/objects/batch
                    → 301 redirect to actual upload URL (Azure Blob, S3, etc.)
                    → Uploads 100 MB binary
        
        THEN: normal git pack is sent (contains only the 134-byte pointer blob)

STEP 4: Remote receives the git pack (fast — 134 bytes for the video)
        Remote also now has the LFS object stored in LFS backend

─────────────────────────────────────────────────────────────────────
COMMAND: git clone / git checkout (with LFS)

STEP 1: git clone downloads only git objects (commits, trees, pointer blobs)
        Fast: the 100 MB video is NOT downloaded yet

STEP 2: During checkout, for each file with LFS filter:
        SMUDGE FILTER runs:
          Input:  the 134-byte pointer
          Action: calls LFS API to download the actual binary
                  GET https://github.com/org/repo.git/info/lfs/objects/batch
                  Downloads from CDN into ~/.git/lfs/objects/...
                  Copies to working directory
          Output: the actual 100 MB binary in video.mp4

STEP 3: Working directory shows the real video file (not the pointer)

GIT LFS SMUDGE SKIP (for CI that doesn't need large files):
  GIT_LFS_SKIP_SMUDGE=1 git clone ...
  → Clones fast, pointer files remain as pointers in working tree
  → Only download specific LFS files you actually need:
    git lfs pull --include="assets/logo.png"
```

---

## RULE 4 — ASCII Architecture Diagrams

### Diagram 1: Shallow Clone Boundary

```
FULL HISTORY (remote):

  A ← B ← C ← D ← E ← F ← G ← H  (HEAD)
  │   │   │   │   │   │   │   │
  2018 ────────────────────── 2026

SHALLOW CLONE (--depth=3):

                          F ← G ← H  (HEAD, local)
                          │
                    .git/shallow contains SHA of F
                    F is treated as a root commit (no parents locally)

  What git log shows:    F  G  H   (only 3 commits)
  What the remote has:   A  B  C  D  E  F  G  H  (full history)

  If you need to:
  ├── Rebase onto older commits → need to unshallow first
  ├── git blame a file that was last changed at commit C → need to unshallow
  ├── Run git bisect to find a bug in commit D → need to unshallow to reach D
  └── Just run tests / build the app → shallow is perfect (you have the latest code)

SHALLOW WITH TAGS:
  --shallow-exclude=v2.0  → "give me everything since the v2.0 tag was cut"
  
  A ← B ← C ← [v2.0] ← D ← E ← F ← G ← H
                         ▲
                    .git/shallow boundary here
                    Everything from D onward is present
```

### Diagram 2: Partial Clone Filter Types

```
PARTIAL CLONE FILTERS (--filter= option):

  git clone --filter=blob:none  <url>
  ┌────────────────────────────────────────────────────────────────┐
  │  Downloaded at clone time:  commits + trees (NO blobs)        │
  │  Blobs fetched: on-demand when you access a file              │
  │                                                               │
  │  Best for: developers who only work in certain directories    │
  │            CI that does static analysis (reads AST, not files)│
  │  Worst for: building/testing the whole codebase               │
  └────────────────────────────────────────────────────────────────┘

  git clone --filter=tree:0  <url>
  ┌────────────────────────────────────────────────────────────────┐
  │  Downloaded at clone time:  commits ONLY (no trees, no blobs) │
  │  Trees/blobs fetched: on-demand when you checkout files       │
  │                                                               │
  │  Best for: git log / git blame (read commit messages + SHAs)  │
  │            CI that only needs metadata                        │
  │  Worst for: anything that needs to read actual file content   │
  └────────────────────────────────────────────────────────────────┘

  git clone --filter=blob:limit=1m  <url>
  ┌────────────────────────────────────────────────────────────────┐
  │  Downloaded at clone time:  commits + trees + blobs < 1 MB   │
  │  Large blobs (>1MB) fetched: on-demand                        │
  │                                                               │
  │  Best for: repos with a few giant binary assets              │
  │            Normal source files available immediately          │
  └────────────────────────────────────────────────────────────────┘

  COMBINING FILTERS (Git 2.24+):
  git clone --filter=combine:blob:none+tree:0  <url>
  └─ Only commits downloaded (ultra-minimal)

PROMISOR REMOTE:
  When a partial clone fetches a missing object, it goes to the
  "promisor remote" — the remote that "promised" to serve these objects.
  
  .git/config:
  [remote "origin"]
    promisor = true
    partialclonefilter = blob:none
  
  On first access to a missing blob:
    git fetch origin <missing-blob-sha>
    (transparent to the user — happens automatically)
```

### Diagram 3: Sparse Checkout Cone Mode vs Non-Cone Mode

```
NON-CONE MODE (old way):
  .git/info/sparse-checkout contains gitignore-style patterns:
    /src/**
    !/src/legacy/**
    /docs/api.md
  
  Problem: Git must evaluate every pattern against every file path
           For 100,000 files × 50 patterns = 5,000,000 comparisons per operation
           SLOW.

CONE MODE (modern, fast):
  Patterns are restricted to full directory boundaries:
    "include this directory and all its contents"
    "include root-level files"
    
  .git/info/sparse-checkout (generated by cone mode):
    /*          ← include root files (README.md, package.json, etc.)
    !/*/        ← exclude all subdirs (then we re-add specific ones below)
    /src/       ← include src/ and everything inside it
    /docs/      ← include docs/ and everything inside it

  Fast because:
    Git only needs to check: "does this path start with /src/ or /docs/ or is it root?"
    Single string prefix match — O(1) per file instead of O(patterns) per file

CONE MODE TOPOLOGY:
  
  Repo tree:              Cone includes /src/ and /docs/:
  
  ├── README.md  ✅       ├── README.md  ✅  (root file → always included)
  ├── package.json ✅     ├── package.json ✅
  ├── src/               ├── src/
  │   ├── app.ts ✅       │   ├── app.ts ✅
  │   └── util.ts ✅      │   └── util.ts ✅
  ├── docs/              ├── docs/
  │   └── api.md ✅       │   └── api.md ✅
  ├── assets/            ├── assets/           (excluded, not in cone)
  │   └── logo.png ❌     │   (no files on disk, skip-worktree bit set)
  └── infra/             └── infra/
      └── k8s/ ❌             (no files on disk, skip-worktree bit set)
```

### Diagram 4: Git LFS Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    DEVELOPER'S MACHINE                           │
│                                                                  │
│  Working Dir              Git Repo (.git/)     LFS Cache         │
│  ─────────────           ──────────────────   ─────────────────  │
│  video.mp4 (100MB) ←──── pointer blob(134B)   4d7a....(100MB)   │
│  logo.png  (5KB)  ←──── blob(5KB) ←────────── (not in LFS)      │
│                                                │                 │
│                           git objects          │                 │
│                           (normal git)         │                 │
└──────────────────────────────────────────────────────────────────┘
                                │ git push             │ lfs push
                                ▼                      ▼
┌──────────────────────────────────────────────────────────────────┐
│                       GITHUB / REMOTE                            │
│                                                                  │
│  Git Repository                    LFS Object Store              │
│  ─────────────────────────────    ────────────────────────────   │
│  Commits, trees, pointer blobs    Actual large file content      │
│  (normal git pack protocol)       (separate HTTP API)            │
│  ~ 200 KB for this commit         ~ 100 MB for the video         │
│                                                                  │
│  GitHub LFS API:                                                 │
│  POST /org/repo.git/info/lfs/objects/batch                       │
│    → returns upload/download URLs (to GitHub's CDN or Azure)     │
└──────────────────────────────────────────────────────────────────┘

IMPORTANT: LFS storage is SEPARATE from git storage.
  - GitHub Free: 1 GB LFS storage, 1 GB/month bandwidth
  - GitHub Pro: 2 GB / 2 GB
  - GitHub Teams: pay-per-use after included limits
  - Self-hosted: run your own LFS server (Gitea, MinIO, etc.)
```

### Diagram 5: git filter-repo vs git filter-branch

```
                    THE HISTORY REWRITING TOOLS

git filter-branch (old, AVOID):
  ┌──────────────────────────────────────────────────────┐
  │ git filter-branch --tree-filter 'rm -f secrets.txt'  │
  │   ├── Checks out EVERY commit to a temp directory     │
  │   ├── Runs the shell command for each commit          │
  │   ├── Re-commits the result                           │
  │   └── For 10,000 commits: 10,000 checkout/commit ops │
  │                                                       │
  │ Speed: 1,000 commits/hour (approximately)            │
  │ A 100,000-commit repo: ~100 hours                    │
  │ Memory: requires full working tree per commit         │
  └──────────────────────────────────────────────────────┘
  ❌ Deprecated since Git 2.24. Do not use.

git filter-repo (new, USE THIS):
  ┌──────────────────────────────────────────────────────┐
  │ git filter-repo --path secrets.txt --invert-paths    │
  │   ├── Reads the pack file directly (no checkout!)    │
  │   ├── Processes objects in topological order         │
  │   ├── Writes new objects in a single pass            │
  │   └── For 10,000 commits: one streaming pass         │
  │                                                       │
  │ Speed: ~1,000,000 commits/hour (approximately)       │
  │ A 100,000-commit repo: ~6 minutes                    │
  │ Memory: streams objects, minimal RAM needed           │
  └──────────────────────────────────────────────────────┘
  ✅ BFG Repo Cleaner is also fast, but filter-repo is more flexible

WHAT filter-repo REWRITES:
  Original:  A  → B  → C  → D  (HEAD)
  Rewritten: A' → B' → C' → D' (HEAD)
  
  EVERY commit gets a new SHA because:
    commit SHA = hash(tree + parents + author + message)
    changing the tree (removing secrets.txt) → new tree SHA
    new tree SHA → new commit SHA
    new commit SHA → children's parent pointers change → their SHAs change
    cascade: ALL descendants get new SHAs
  
  This is why a force push is required and all clones must re-clone.
```

---

## RULE 5 — Hands-On Proof Commands

### Shallow Clone

```bash
# Clone with only the last 1 commit
git clone --depth=1 https://github.com/nodejs/node.git node-shallow
cd node-shallow

# Verify it's shallow
git rev-parse --is-shallow-repository
# Output: true

# See the shallow boundary
cat .git/shallow
# Output: a7b5c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

# Only one commit in log
git log --oneline | wc -l
# Output: 1

# Try to see more history — hits the boundary
git log --oneline -5
# Still shows just 1 commit

# Compare sizes: shallow vs full
du -sh .git/objects/
# ~50 MB vs ~1.2 GB for the full node.js repo

# Deepen by 10 more commits
git fetch --deepen=10
git log --oneline | wc -l
# Output: 11

# Fully unshallow (warning: downloads everything)
git fetch --unshallow
cat .git/shallow
# cat: .git/shallow: No such file or directory  ← file is gone!

# Shallow clone for CI (most common use case):
git clone --depth=1 --branch=main --single-branch <url> .
# --single-branch: only fetch the main branch (skip all other branches)
# This is the standard CI pattern: fast clone, just need current state
```

### Sparse Checkout

```bash
# Method 1: sparse checkout on an existing clone
git sparse-checkout init --cone
git sparse-checkout set src/ docs/

# Verify what's included
git sparse-checkout list
# Output:
# docs
# src

# See the generated patterns
cat .git/info/sparse-checkout
# /*
# !/*/
# /docs/
# /src/

# Add more directories
git sparse-checkout add tests/

# Check working tree (other dirs are absent)
ls
# docs/  package.json  README.md  src/  tests/
# (assets/, infra/, etc. are absent despite being in the index)

# Verify skip-worktree bit on an excluded file
git ls-files -v assets/logo.png
# Output: S assets/logo.png   ← "S" means skip-worktree

# Temporarily access an excluded file
git checkout HEAD -- assets/logo.png   # force-populate it
git sparse-checkout reapply            # remove it again

# Method 2: sparse partial clone (most efficient for monorepos)
git clone --filter=blob:none --sparse <url> my-repo
cd my-repo
git sparse-checkout init --cone
git sparse-checkout set apps/api
# Now: only apps/api blobs are downloaded; everything else is lazy-loaded
```

### Partial Clone

```bash
# Clone with no blobs
git clone --filter=blob:none https://github.com/torvalds/linux.git linux-partial
cd linux-partial

# Compare size to theoretical full clone
du -sh .git/objects/
# ~800 MB vs ~4 GB for full clone

# Verify it's a partial clone
git config remote.origin.partialclonefilter
# Output: blob:none

# Access a specific file (triggers on-demand blob fetch)
git checkout HEAD -- init/main.c
# Git silently fetches the blob for init/main.c from origin

# See how many objects were fetched lazily so far
git count-objects -v
# look at the "promisor" count

# Fetch all missing blobs for the current checkout
git checkout HEAD  # checks out all files, fetching missing blobs as needed

# Clone with tree filter (only commits, no trees or blobs)
git clone --filter=tree:0 <url>
git log --oneline  # works fine (log reads commit objects only)
git show HEAD:README.md  # triggers on-demand fetch of tree + blob
```

### Git LFS

```bash
# Install git-lfs (one-time per machine)
# macOS:   brew install git-lfs
# Windows: winget install GitHub.GitLFS
# Linux:   apt install git-lfs

# Initialize LFS in a repo (one-time per repo)
git lfs install

# Track file patterns
git lfs track "*.mp4"
git lfs track "*.psd"
git lfs track "*.zip" "*.tar.gz"

# The above edits .gitattributes — commit it!
git add .gitattributes
git commit -m "chore: configure Git LFS for large binary files"

# Add a large file (LFS intercepts transparently)
cp ~/Downloads/demo-video.mp4 ./assets/
git add assets/demo-video.mp4
git status
# new file: assets/demo-video.mp4  ← looks normal

# See what git is actually storing (the pointer)
git show HEAD:assets/demo-video.mp4
# version https://git-lfs.github.com/spec/v1
# oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f...
# size 104857600

# Push (LFS uploads the binary first, then the git pack)
git push origin main

# List all LFS objects in the repo
git lfs ls-files
# 4d7a214614 * assets/demo-video.mp4

# See LFS status
git lfs status

# Download LFS objects (e.g., after GIT_LFS_SKIP_SMUDGE=1 clone)
git lfs pull                          # all LFS files
git lfs pull --include="assets/*.mp4" # just videos

# Migrate existing repo (add LFS to files already committed)
git lfs migrate import --include="*.mp4" --everything
# Rewrites history, converting blob objects to LFS pointers
# WARNING: this rewrites history — coordinate with team!

# Verify LFS objects are intact
git lfs fsck
# Git LFS fsck OK
```

### git filter-repo

```bash
# Install (not bundled with git)
pip install git-filter-repo
# OR: brew install git-filter-repo
# OR: download the single Python script from github.com/newren/git-filter-repo

# ALWAYS work on a fresh clone for safety
git clone --mirror https://github.com/org/repo.git repo-clean.git
cd repo-clean.git

# Remove a specific file from ALL history
git filter-repo --path secrets/credentials.json --invert-paths

# Remove multiple paths
git filter-repo --path .env --path config/secrets.yaml --invert-paths

# Remove files matching a pattern
git filter-repo --path-glob "*.psd" --invert-paths
git filter-repo --path-regex ".*\.(mp4|mov|avi)$" --invert-paths

# Remove a directory
git filter-repo --path old-legacy-module/ --invert-paths

# Keep ONLY specific paths (inverse: remove everything EXCEPT these)
git filter-repo --path src/ --path docs/

# Replace sensitive strings in file contents
git filter-repo --replace-text <(cat <<'EOF'
password123==>***REMOVED***
sk_live_AbCdEfGh==>***REMOVED***
EOF
)

# Rename a file across all history
git filter-repo --path-rename old-name.ts:new-name.ts

# After filter-repo, the repo needs cleanup
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Push back (MUST use --force, all SHAs have changed)
# WARNING: Coordinate with team first — everyone must re-clone!
git push origin --force --all
git push origin --force --tags

# VERIFY: The file is gone from all history
git log --all --full-history -- secrets/credentials.json
# Should produce no output
```

### Combining Techniques: The Ultimate Fast CI Clone

```bash
# Maximum performance CI clone:
git clone \
  --depth=1 \             # only last commit
  --single-branch \       # only main branch, not all branches
  --branch=main \         # explicitly name the branch
  --filter=blob:none \    # no blobs at clone time (fetch on demand)
  --no-tags \             # skip all tags
  https://github.com/org/repo.git \
  .

# For a monorepo where CI only builds apps/api:
git sparse-checkout init --cone
git sparse-checkout set apps/api packages/shared

# Total: clone a 5 GB monorepo in under 30 seconds
# (vs 8+ minutes for a full clone)
```

---

## RULE 6 — Syntax Breakdown (Deep)

### `git clone` (performance flags)

```
git clone [--depth=<n>] [--shallow-since=<date>] [--shallow-exclude=<rev>]
          [--single-branch] [--branch=<name>] [--no-tags]
          [--filter=<spec>] [--sparse] [--mirror] [--bare]
          <url> [<directory>]
│            │               │                     │           │       │
│            │               │                     │           │       └─ local dir name
│            │               │                     │           └─ don't fetch tags (speeds up clone)
│            │               │                     └─ initialize sparse checkout (needs follow-up set)
│            │               └─ partial clone filter:
│            │                  blob:none     → skip all blobs
│            │                  tree:0        → skip all trees and blobs
│            │                  blob:limit=N  → skip blobs larger than N bytes
│            └─ truncate history at this date (e.g., --shallow-since=2024-01-01)
└─ truncate history to N commits deep (1 = only latest commit)

--single-branch: only fetch the tracking branch, not all remote branches
                 reduces fetch bandwidth by 60-90% for repos with many branches
--branch=<name>: which branch to clone (works with --single-branch)
--mirror:        bare clone that also mirrors all refs (for backup/fork servers)
--bare:          .git/ without working tree (for server-side bare repos)
```

### `git sparse-checkout`

```
git sparse-checkout <subcommand> [options]

  init [--cone | --no-cone]
  │     └─ cone mode: fast prefix-based matching (RECOMMENDED)
  │        no-cone: full gitignore-style patterns (slow but flexible)
  │
  set <dir>...
  │   └─ set the sparse checkout to EXACTLY these directories
  │      (replaces any existing configuration)
  │      In cone mode: argument is a directory path (no globs needed)
  │
  add <dir>...
  │   └─ ADD directories to the existing sparse checkout
  │
  list
  │   └─ show current sparse checkout paths
  │
  reapply
  │   └─ re-apply the current config to the working tree
  │      (useful after merge/rebase that added files outside the cone)
  │
  disable
      └─ turn off sparse checkout (re-populate full working tree)
```

### `git lfs` (key subcommands)

```
git lfs <subcommand>

  install
  └─ one-time setup: installs the LFS hooks in .git/hooks/
     (pre-push, post-checkout, post-commit, post-merge)
     AND sets up the global LFS filter config in ~/.gitconfig

  track <pattern>
  │  └─ adds entry to .gitattributes:
  │     "*.mp4 filter=lfs diff=lfs merge=lfs -text"
  │     The "-text" disables line-ending conversion for binary files
  │     The "filter=lfs" activates clean/smudge filters for this pattern
  │
  untrack <pattern>
  └─ removes entry from .gitattributes

  ls-files [-l] [-s]
  │  └─ list LFS tracked files in HEAD
  │     -l: show full OID
  │     -s: show size
  │
  pull [--include=<pattern>] [--exclude=<pattern>]
  └─ download LFS objects (useful after GIT_LFS_SKIP_SMUDGE=1 clone)

  push [--all] <remote>
  └─ push LFS objects to remote (normally done automatically on git push)
     --all: push ALL LFS objects (not just those in current commits)

  migrate import --include=<pattern> [--everything]
  │  └─ convert existing blobs to LFS pointers, rewriting history
  │     --everything: process ALL branches and tags
  │     WARNING: rewrites history — all SHAs change
  │
  fsck
  └─ verify LFS objects are not corrupted

  prune [--dry-run]
  └─ delete local LFS objects no longer referenced by current history
```

### `git filter-repo` (key options)

```
git filter-repo [options]

  --path <path>
  └─ KEEP only objects that touch this path (all others removed)
     Can be specified multiple times

  --path --invert-paths
  └─ REMOVE objects touching this path (keep everything else)
     (--invert-paths reverses the meaning of --path)

  --path-glob <glob>
  └─ like --path but with gitignore-style glob patterns

  --path-regex <regex>
  └─ like --path but with Python regular expressions

  --path-rename <old>:<new>
  └─ rename a path across all history

  --replace-text <expressions-file>
  └─ file containing "find==>replace" pairs
     replaces in FILE CONTENTS of all blobs

  --strip-blobs-bigger-than <size>
  └─ remove all blobs larger than <size> (e.g., "10M")

  --commit-callback <python-code>
  └─ Python function called for each commit — full flexibility

  --refs <refspec>
  └─ only process specified branches/tags (not all refs)

  --force
  └─ run even if the repo doesn't look like a fresh clone
     (filter-repo refuses to run on non-fresh clones by default as a safety measure)
```

---

## RULE 7 — Beginner → Production Example

### Beginner: Set Up LFS for a Design Asset Repository

```
SCENARIO: You and a friend are making an indie game. Your repo has 
sprite sheets, sound effects, and level design files.
```

```bash
# Step 1: Install LFS and initialize
git lfs install

# Step 2: Track game asset types
git lfs track "*.png" "*.wav" "*.ogg" "*.aseprite" "*.ldtk"

# Step 3: Commit the .gitattributes
git add .gitattributes
git commit -m "chore: configure LFS for game assets"

# Step 4: Add assets normally — LFS handles everything
cp ~/Downloads/hero-spritesheet.png assets/sprites/
git add assets/sprites/hero-spritesheet.png
git commit -m "art: add hero spritesheet"
git push

# Step 5: Your friend clones the repo
# git clone ... → fast (just pointer files)
# When they open the project in their game engine, the sprites download automatically

# What your friend sees when they clone (if engine reads the file):
# Git LFS: (1 of 1 files) 2.4 MB / 2.4 MB  ← downloads transparently
```

---

### Production: Migrating a Monorepo from Full Clone to Optimized CI

```
SCENARIO: 12-person engineering team. Monorepo with 8 services.
  - Full clone: 4.2 GB, takes 12 minutes on GitHub Actions
  - Most CI jobs only build ONE of the 8 services
  - Design team committed 800 MB of Figma exports directly into git (!)

PHASE 1: Emergency — Remove the Figma files from history
```

```bash
# Find the damage first
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob" && $3>5000000' \
  | sort -k3 -rn
# Output confirms: 847 MB of .fig and .png files in design/exports/

# Clone fresh for filter-repo (never run on a working clone)
git clone --mirror https://github.com/my-org/monorepo.git monorepo-clean.git
cd monorepo-clean.git

# Remove the design exports from ALL history
git filter-repo --path design/exports/ --invert-paths

# Verify removal
git log --all --full-history -- design/exports/ | wc -l
# Output: 0  ✅

# Expire and prune
git reflog expire --expire=now --all
git gc --prune=now --aggressive
du -sh .  # 4.2 GB → 1.1 GB

# Push (coordinate with team: everyone re-clones after this)
git push --force origin --all
git push --force origin --tags
```

```bash
# PHASE 2: Configure LFS for future design files
cd /path/to/fresh-monorepo-clone
git lfs install
git lfs track "*.fig" "*.sketch" "design/exports/**"
git add .gitattributes
git commit -m "chore: prevent future binary assets in git objects"
git push
```

```bash
# PHASE 3: Optimize CI for each service

# .github/workflows/api-service.yml
# Before (full clone):
- uses: actions/checkout@v4
# Clone time: 12 minutes for 4.2 GB

# After (optimized):
- uses: actions/checkout@v4
  with:
    fetch-depth: 1                     # shallow: only latest commit
    filter: tree:0                     # partial: no trees or blobs at clone time
    sparse-checkout: |                 # sparse: only relevant directories
      services/api/
      packages/shared/
      packages/config/
    sparse-checkout-cone-mode: true

# Additional speed: cache the LFS objects between runs
- uses: actions/cache@v4
  with:
    path: ~/.git/lfs
    key: lfs-${{ hashFiles('.lfsconfig') }}

# RESULT:
# Old CI time:  12 minutes (full clone 4.2 GB)
# New CI time:  45 seconds (1.1 GB repo, only 180 MB downloaded)
# Team savings: 8 services × 30 PRs/day × 11 min = 44 hours of CI time saved per day
```

```bash
# PHASE 4: Configure git maintenance on the server
# (On your self-hosted runner or in a pre-step on GitHub Actions)
git maintenance start
git config maintenance.strategy incremental
# Keeps the repo healthy without full gc on every run
```

---

## RULE 8 — Mistakes → Root Cause → Fix → Prevention

### MISTAKE 1: Shallow Clone Breaks `git describe` / Semantic Release

**The mistake:**
```yaml
# CI workflow
- uses: actions/checkout@v4
  with:
    fetch-depth: 1   # shallow clone for speed
    
- name: Get version
  run: npx semantic-release   # or: git describe --tags --abbrev=0
# Error: fatal: No names found, cannot describe anything.
# OR:    git describe outputs: "v1.2.0-3000-gabc123" (wrong patch count!)
```

**Root Cause:** `git describe` works by finding the nearest tag in history, then counting 
commits since that tag. With `--depth=1`, Git has one commit and no tags (tags reference commits;
with a shallow clone of depth 1, no ancestor commits exist to be tagged).

Semantic release tools need to read commit messages back to the last release tag to determine
the next version number. With a shallow clone, they can't see those commits.

**Fix:**
```yaml
# Option A: Full history for release jobs only
- uses: actions/checkout@v4
  with:
    fetch-depth: 0   # full history for release job
    # But only use this in the release job, not in build/test jobs

# Option B: Fetch just enough history to reach the last tag
- uses: actions/checkout@v4
  with:
    fetch-depth: 0   # semantic-release needs this; there's no partial alternative
```

**Prevention:**
- Use `fetch-depth: 1` for build, test, and lint jobs (the majority)
- Use `fetch-depth: 0` ONLY for jobs that need full history: semantic-release, git-describe, changelog generation, git-blame CI checks
- Consider fetching tags explicitly: `git fetch --tags --depth=1` (gets tags at current depth)

---

### MISTAKE 2: LFS Files Not Pushed Before the Git Push

**The mistake:**
```bash
git add large-model.bin   # tracked by LFS
git commit -m "add ML model"
git push                  # appears to succeed!

# Teammate runs:
git pull
git checkout main -- large-model.bin
# Error: Smudge error: Error downloading object:
#        large-model.bin (sha256:abc...): [403] Forbidden
```

**Root Cause:** This happens in two ways:
1. LFS credentials expired mid-push: The git pack was pushed but the LFS upload failed silently
2. Running in an environment without LFS credentials but with LFS installed:
   - CI runner has git but not the LFS token
   - The pointer is pushed to git; the binary never reaches LFS storage
   - The pointer is now "dangling" — references an LFS object that doesn't exist on the server

**Fix:**
```bash
# Check which LFS objects are missing on the remote:
git lfs ls-files --all | awk '{print $1}' > local-lfs-oids.txt
# Compare with what the remote has (requires LFS API access)

# Re-push all LFS objects:
git lfs push --all origin
# "--all" pushes ALL local LFS objects, not just those in current commits

# Verify:
git lfs fsck --pointers   # check pointers in git are valid
git lfs fsck --objects    # check LFS objects are present locally
```

**Prevention:**
```bash
# Add a pre-push hook to verify LFS objects before push:
# .git/hooks/pre-push (or via husky):
git lfs pre-push "$@"
# This is the hook that git lfs install sets up automatically.
# If LFS is installed, this is already there. If not:
git lfs install  # installs the hooks
```

---

### MISTAKE 3: git filter-repo on a Non-Mirror Clone Corrupts the Repo

**The mistake:**
```bash
# You run filter-repo on your working clone (NOT a mirror clone)
cd my-normal-repo-clone
git filter-repo --path secrets.env --invert-paths
# Warning: Refusing to destructively overwrite repo history since
#          this does not look like a fresh clone.
#          (Pass --force to override, or clone your repo and use that.)
```

If they pass `--force`:
```bash
git filter-repo --path secrets.env --invert-paths --force
# Now .git/config remote "origin" is DELETED by filter-repo
# filter-repo removes remotes to prevent accidental force push of rewritten history
# git push origin  →  fatal: 'origin' does not appear to be a git repository
```

**Root Cause:** `git filter-repo` deliberately:
1. Requires a fresh clone (by default) to prevent accidents
2. Removes all remotes from config after running, as a safety measure to prevent 
   accidental `git push origin main` of rewritten history on a repo others depend on

**Fix:**
```bash
# ALWAYS work on a --mirror clone:
git clone --mirror https://github.com/org/repo.git repo-mirror.git
cd repo-mirror.git
git filter-repo --path secrets.env --invert-paths
# Now push explicitly when you're sure:
git push --force https://github.com/org/repo.git refs/heads/*:refs/heads/*
git push --force https://github.com/org/repo.git refs/tags/*:refs/tags/*
```

**Prevention:**
- Make `--mirror` clone part of your documented "history cleanup" runbook
- Never run `--force` to skip filter-repo's safety check in production repos
- Test filter-repo on a personal fork first

---

### MISTAKE 4: Sparse Checkout Causes "Missing" Files During Merge Conflicts

**The mistake:**
```
Engineer A: Working in apps/api/ with sparse checkout
            Runs: git merge feature/cross-service-refactor
            Gets: CONFLICT (content): Merge conflict in apps/web/config.js
            But apps/web/ is NOT in their sparse checkout!
            File doesn't exist on their disk.
            Conflict resolution tools can't open the file.
```

**Root Cause:** Merge doesn't respect sparse checkout — it works on the INDEX, not the 
working tree. A merge conflict can occur in any file touched by the merge, regardless of 
whether it's in your sparse checkout cone. The skip-worktree bit prevents the file from 
appearing on disk, but the conflict marker IS in the index.

**Fix:**
```bash
# Temporarily populate the conflicted file:
git checkout MERGE_HEAD -- apps/web/config.js   # bring in THEIR version
git checkout HEAD -- apps/web/config.js          # bring in OUR version

# Actually, easier:
git sparse-checkout add apps/web/                # temporarily add to cone
# Resolve the conflict normally
# Then remove from cone:
git sparse-checkout set apps/api/                # back to original cone
git sparse-checkout reapply                      # remove apps/web/ from working dir again

# Or use merge --no-commit + manual index edit:
git checkout --theirs -- apps/web/config.js     # accept their version
git add apps/web/config.js
git merge --continue
```

**Prevention:**
- If your monorepo has frequent cross-service refactors, sparse checkout may not be the right tool
- Use `git sparse-checkout reapply` after any merge/rebase to clean up stale files
- Consider partial clone (`--filter=blob:none`) instead: blobs are lazy-loaded but all files appear in the working tree when accessed

---

### MISTAKE 5: `GIT_LFS_SKIP_SMUDGE=1` Left On Permanently in Git Config

**The mistake:**
```bash
# Someone adds to their .bashrc or CI environment to speed up clones:
export GIT_LFS_SKIP_SMUDGE=1

# They forget about it. Later:
cp logo.png assets/  
git add assets/logo.png  # LFS is installed, pattern matches
git commit -m "add logo"
git push
# Looks fine!

# ACTUAL PROBLEM:
# In their working dir: assets/logo.png is a 20-byte POINTER file, not the real image
# The real 180 KB image was never copied into LFS cache on their machine
# git lfs clean ran and wrote the pointer, but the source file was already a pointer
# Result: LFS pointer pushed that points to a SHA that doesn't exist in LFS storage
```

**Root Cause:** `GIT_LFS_SKIP_SMUDGE=1` skips the smudge filter (pointer → real file on checkout),
but the clean filter (real file → pointer on add) still runs. If the file you're adding is 
ALREADY a pointer (or looks like one), LFS will try to compute the SHA256 of the actual file 
content — but it can't find the actual file in its cache (because smudge was skipped previously),
leading to a dangling pointer being committed.

**Fix:**
```bash
# Check if you have dangling LFS pointers in current HEAD:
git lfs fsck --pointers

# If broken, fix by adding the real file:
cp /actual/source/logo.png assets/logo.png  # overwrite pointer with real file
git add assets/logo.png
git commit --amend  # or new commit
git push

# Verify LFS objects were actually uploaded:
git lfs push --all origin
```

**Prevention:**
- Never set `GIT_LFS_SKIP_SMUDGE=1` globally in shell profiles
- In CI: only use `GIT_LFS_SKIP_SMUDGE=1` if you KNOW you don't need the LFS files
  (e.g., a pure backend lint job that ignores assets)
- After `GIT_LFS_SKIP_SMUDGE=1` clone, run `git lfs pull` before doing any adds

---

## RULE 9 — Byte-Level Internals

### The `.git/shallow` File Format

```
.git/shallow is a plain text file with exactly one SHA-1 per line:

Content:
  a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\n
  f9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0\n

Each SHA is a "graft" — a commit whose parent list is overridden to be empty.
Git reads this file during any operation that traverses history.

HOW GIT USES IT:
  1. git log: when traversing parents, if the parent SHA is in .git/shallow → STOP
  2. git merge-base: stops at shallow boundary (may return wrong results!)
  3. git push: sends .git/shallow entries as "shallow" lines in the pack protocol
  4. git fetch --unshallow: deletes lines from this file as history is fetched

GRAFT MECHANISM (historical context):
  Before --shallow existed, developers used .git/info/grafts to fake parent relationships.
  Shallow uses the same concept but as a first-class protocol feature.
  .git/shallow = "these commits have no parents in THIS clone"
  The remote still has the full history; we just didn't download it.
```

### Git LFS Pointer File Format

```
LFS pointer files are UTF-8 text files stored as regular git blobs.
They MUST follow this exact format (order matters, whitespace exact):

version https://git-lfs.github.com/spec/v1\n
oid sha256:<64-hex-chars>\n
size <decimal-bytes>\n

Example (134 bytes exactly for this object):
  version https://git-lfs.github.com/spec/v1\n      ← 41 bytes + newline
  oid sha256:4d7a214614ab2935c943f9e0ff69d22e\n      ← 76 bytes + newline
              adbb8f01a1dee2db6bcacfd6f3c54d3e
  size 104857600\n                                   ← "size " + decimal + newline

VALIDATION RULES (from LFS spec):
  - First line MUST be "version https://git-lfs.github.com/spec/v1"
  - "oid" field: currently always "sha256:" prefix + 64 lowercase hex chars
  - "size" field: must be a non-negative integer (the exact byte count of the original)
  - File must NOT have any other content
  - Maximum pointer size: 200 bytes (if > 200 bytes, it's not a valid pointer)

THE SHA256:
  This is SHA-256 of the raw file content (NOT git's SHA-1).
  Git's SHA-1 is computed on the pointer file itself (134 bytes).
  LFS's SHA-256 is computed on the actual large binary content.
  They use different algorithms and have different purposes.

LFS BATCH API PROTOCOL:
  POST /org/repo.git/info/lfs/objects/batch
  {
    "operation": "download",    ← or "upload"
    "objects": [
      { "oid": "4d7a214...", "size": 104857600 }
    ],
    "transfers": ["basic", "multipart-basic"]
  }
  
  Response:
  {
    "objects": [{
      "oid": "4d7a214...",
      "size": 104857600,
      "actions": {
        "download": {
          "href": "https://storage.github.com/...",  ← pre-signed URL
          "expires_at": "2026-05-29T00:00:00Z",
          "header": { "Authorization": "Bearer ..." }
        }
      }
    }]
  }
  
  Then: GET the href URL → actual binary content
```

### Partial Clone: The Pack Protocol Extension

```
PARTIAL CLONE PROTOCOL (Git protocol v2):

NORMAL CLONE (fetch negotiation):
  client: "WANT abc123" (commit SHA)
  server: "ACK abc123"
  server: sends pack containing abc123 + all reachable objects

PARTIAL CLONE:
  client: "WANT abc123"
  client: "FILTER blob:none"   ← NEW: tells server what to omit
  server: "ACK abc123"
  server: sends pack containing abc123 + all trees + NO blobs
  server: adds "object-format" and "filter" capabilities to pack header

PROMISOR PACK:
  The pack file sent for a partial clone has a special marker:
  .git/objects/pack/pack-NNN.promisor   ← empty file, indicates this is a promisor pack
  
  This tells git: "it's OK that some objects referenced in this pack are missing.
                   They can be fetched on demand from the promisor remote."

ON-DEMAND FETCH:
  When git needs a missing blob:
  1. git detects: "I need blob SHA abc123 but it's not in local objects"
  2. Checks: is any remote marked as promisor? (promisor = true in config)
  3. Yes → runs: git fetch origin <abc123>
     This is a "lazy object fetch" — protocol v2 feature
  4. The blob is now local and cached in a new pack file

LAZY FETCH PERFORMANCE:
  First access to any file: +50-200ms network round-trip
  Subsequent accesses to the same file: 0ms (cached locally)
  After checkout: blobs are cached; next operation is as fast as a full clone
```

### The Sparse Checkout Index Bit (skip-worktree)

```
git index entry (62+ bytes per file):

Bytes  0- 3: ctime (last metadata change, seconds)
Bytes  4- 7: ctime nanoseconds
Bytes  8-11: mtime (last data change, seconds)
Bytes 12-15: mtime nanoseconds
Bytes 16-19: dev (device number)
Bytes 20-23: ino (inode number)
Bytes 24-27: mode (file permissions: 100644 or 100755)
Bytes 28-31: uid (user ID)
Bytes 32-35: gid (group ID)
Bytes 36-39: file size
Bytes 40-59: SHA-1 of the blob object (20 bytes)
Bytes 60-61: flags (2 bytes)
  bit 15: assume-valid flag (performance hint: skip stat() check)
  bit 14: extended flag (if 1, 2 more flag bytes follow)
  bits 12-13: merge stage (0=normal, 1/2/3=conflict stages)
  bits  0-11: filename length (or 0xFFF if length >= 0xFFF)
Bytes 62-63: extended flags (only if bit 14 of flags is 1)
  bit 15 (of extended): RESERVED
  bit 14: SKIP-WORKTREE ← THIS IS THE SPARSE CHECKOUT BIT
    0 = normal (file should be on disk)
    1 = skip-worktree (file intentionally absent from disk, don't report as deleted)
  bit 13: intent-to-add (git add -N; blob is empty, staged intent to add)
  bits 0-12: UNUSED

VERIFY the bit:
  git ls-files -v | grep ^S
  # Lines starting with "S" have skip-worktree set
  # Lines starting with "s" have assume-unchanged set (different thing)
  # Lines starting with "H" are normal
```

---

## RULE 10 — Mental Model Checkpoint

**1.** After `git clone --depth=1`, you run `git log --oneline` and see 1 commit. You then run `git fetch --deepen=5`. How many commits does `git log --oneline` show now? What file was modified by `git fetch --deepen=5`?

**2.** You have a sparse checkout with cone mode including only `apps/api/`. Your colleague pushes a commit that modifies `apps/api/server.ts` AND `apps/web/index.html`. You run `git pull`. Which files appear on your disk? What happens to `apps/web/index.html`?

**3.** A partial clone was created with `--filter=blob:none`. You run `git log --oneline` — does this cause any lazy fetches? What about `git show HEAD:README.md`?

**4.** What is the difference between the SHA-1 of an LFS pointer blob and the SHA-256 stored inside the pointer file?

**5.** You run `git filter-repo --path build/ --invert-paths` to remove a build directory from history. After this, do the original commit SHAs still exist on the remote? What must you do before teammates can use the cleaned history?

**6.** The `.git/shallow` file contains a SHA. Does this mean that commit object is missing from `.git/objects/`? Or is it present but treated differently?

**7.** You clone with `--filter=blob:none --sparse` and then run `git sparse-checkout set apps/api/`. Later you run `git checkout main -- apps/api/server.ts`. What objects are fetched from the remote?

**8.** Why does `git describe --tags` fail on a shallow clone with `--depth=1` but work on a shallow clone with `--depth=500`?

**9.** A developer runs `git lfs migrate import --include="*.psd" --everything`. What must all other developers do after this is pushed? Why?

**10.** What is a "promisor remote" and what contract does it establish with the local repository?

---

**Answers:**

1. `git log --oneline` shows **6 commits** (1 original + 5 more). `git fetch --deepen=5` modifies `.git/shallow` — the boundary SHA is updated to point 5 commits earlier (or removed if the repo is now fully fetched).

2. Both `apps/api/server.ts` AND `apps/web/index.html` are applied to the **index**. Only `apps/api/server.ts` appears on disk. `apps/web/index.html` gets the skip-worktree bit set in the index — it's "known" but absent from the working directory. No error occurs.

3. `git log --oneline` reads only **commit objects** — no lazy fetches (commits are present in all partial clones). `git show HEAD:README.md` reads the tree for HEAD (present) then reads the blob for README.md — **this triggers a lazy fetch** of that specific blob.

4. The SHA-1 is git's hash of the pointer file content (`"blob 134\0version https://..."`) — it identifies the pointer as a git object. The SHA-256 inside is the hash of the **actual large binary file** — it identifies the large file in the LFS object store. They hash different things with different algorithms.

5. Yes — the original commits (with the build/ directory) **still exist on the remote**. `git filter-repo` only rewrites your local repo. You must: (1) `git push --force --all --tags` to overwrite the remote with the rewritten history, and (2) have all teammates **re-clone** the repository (their local clones have diverged history that cannot be merged).

6. The commit object **is present** in `.git/objects/`. The `.git/shallow` file just tells git to treat that commit as if it has no parents — it's a "virtual root." The object itself is fully intact; only the parent traversal stops at that point.

7. First, the sparse checkout configuration is applied to `apps/api/`. Then `git checkout main -- apps/api/server.ts` needs to read the blob for that specific file. Since it's a partial clone with `blob:none`, the blob for `apps/api/server.ts` **triggers a lazy fetch from the promisor remote** to download just that blob.

8. `git describe` walks backwards from HEAD looking for a tag. With `--depth=1`, there's only 1 commit — if it's not directly tagged, there are no ancestors to search. With `--depth=500`, git can walk 500 commits and will likely find the most recent tag within that range.

9. All developers must **re-clone** the repository. `git lfs migrate import --everything` rewrites all commits that touched `.psd` files, giving them new SHAs. Every developer's local clone now has diverged history (different SHA for every commit in the rewritten branches). `git pull` will fail with conflicts or create duplicate commits. A fresh `git clone` gets the new, clean history.

10. A promisor remote is a remote marked with `promisor = true` in `.git/config` (set automatically during partial clone). The contract: the remote **promises** to serve any git object on demand via lazy fetch — even objects not yet downloaded to the local repo. When git encounters a missing object (from a partial clone filter), it contacts the promisor remote to fetch it, transparently to the user.

---

## RULE 11 — Connect Everything Backwards

**→ Topic 02 (Git Objects):** Everything in this topic is built on the object model from Topic 02. Partial clone skips fetching blob objects (the raw file content type). LFS replaces the blob content with a pointer, so the blob in git is 134 bytes instead of 100 MB. `git filter-repo` rewrites the tree objects (removing the path to the large blob), which cascades to new commit SHAs. The object types are always the same; these tools just change which objects exist locally and what they contain.

**→ Topic 25 (Packfiles & GC):** Partial clone creates a "promisor pack" — a special pack file that allows referenced-but-missing objects. This is an extension of the normal packfile format. After `git filter-repo`, you must run `git gc --prune=now` to actually reclaim disk space (the old large blobs become unreachable, but GC must delete them). `--filter=blob:limit=10m` in partial clone is the on-demand version of "don't store large blobs" that GC enforces retroactively.

**→ Topic 10 (Remotes):** The "promisor remote" concept (partial clone) is an extension of the remote tracking system from Topic 10. Instead of remotes only providing full history, promisor remotes also provide on-demand objects. The `partialclonefilter` config in `[remote "origin"]` is a new field in the same remote config block you learned about in Topic 10.

**→ Topic 19 (Git Hooks):** Git LFS works entirely through hooks. `git lfs install` sets up `pre-push`, `post-checkout`, `post-commit`, and `post-merge` hooks that intercept normal git operations. The clean filter (running during `git add`) and smudge filter (running during `git checkout`) are a different mechanism than hooks — they're filter drivers defined in `.gitattributes` — but they're the same principle: intercepting git operations at defined points.

**→ Topic 24 (GitHub Actions):** The optimized CI clone pattern from Rule 7 directly implements the `actions/checkout` configuration from Topic 24. The `fetch-depth`, `filter`, `sparse-checkout`, and `sparse-checkout-cone-mode` parameters in `actions/checkout@v4` are all the techniques from this topic expressed as YAML. The 12-minute CI clone → 45-second CI clone is one of the highest-impact performance improvements a team can make.

**→ Topic 23 (Submodules & Monorepos):** Sparse checkout is the Git-native solution to the monorepo scale problem that Topic 23 addressed with submodules. Submodules solve "I don't want other repos' code in mine." Sparse checkout solves "I have all the code in one repo but I only want to work with some of it." Microsoft uses both: a sparse checkout inside the Windows monorepo, plus submodules for third-party dependencies.

**→ Topic 11 (Rebasing):** `git filter-repo` is conceptually a rebase of the ENTIRE history — every commit is rewritten with a new parent and new tree. The "rebase golden rule" (Topic 11: never rebase public history) applies here too: after `git filter-repo` + force push, everyone who cloned from the old history has diverged — exactly the same situation as rebasing a public branch.

---

## RULE 12 — Quick Reference Card

### Shallow Clone

| Command | What It Does |
|---|---|
| `git clone --depth=1 <url>` | Clone with only the latest commit |
| `git clone --depth=N <url>` | Clone with last N commits |
| `git clone --shallow-since=<date> <url>` | Clone all commits after a date |
| `git clone --shallow-exclude=<tag> <url>` | Clone commits after a tag |
| `git clone --single-branch --branch=main <url>` | Clone one branch only |
| `git fetch --deepen=N` | Add N more commits to a shallow clone |
| `git fetch --unshallow` | Convert shallow clone to full clone |
| `git rev-parse --is-shallow-repository` | Check if current repo is shallow |

### Sparse Checkout

| Command | What It Does |
|---|---|
| `git sparse-checkout init --cone` | Enable cone mode sparse checkout |
| `git sparse-checkout set <dir>...` | Set active directories |
| `git sparse-checkout add <dir>` | Add a directory to active set |
| `git sparse-checkout list` | Show currently active paths |
| `git sparse-checkout reapply` | Re-sync working tree with config |
| `git sparse-checkout disable` | Turn off sparse checkout |
| `git ls-files -v \| grep ^S` | List files with skip-worktree bit |

### Partial Clone

| Command | What It Does |
|---|---|
| `git clone --filter=blob:none <url>` | Clone without any blobs |
| `git clone --filter=tree:0 <url>` | Clone commits only |
| `git clone --filter=blob:limit=10m <url>` | Clone, omit blobs > 10 MB |
| `git config remote.origin.partialclonefilter` | View current filter |

### Git LFS

| Command | What It Does |
|---|---|
| `git lfs install` | Set up LFS hooks in current repo |
| `git lfs track "*.ext"` | Track file pattern with LFS |
| `git lfs untrack "*.ext"` | Stop tracking a pattern |
| `git lfs ls-files` | List LFS tracked files in HEAD |
| `git lfs pull` | Download all LFS objects |
| `git lfs pull --include="path"` | Download specific LFS files |
| `git lfs push --all origin` | Push all LFS objects to remote |
| `git lfs migrate import --include="*.psd" --everything` | Migrate existing history to LFS |
| `git lfs fsck` | Verify LFS objects are intact |
| `git lfs prune` | Delete local LFS objects not in current history |
| `GIT_LFS_SKIP_SMUDGE=1 git clone <url>` | Clone without downloading LFS content |

### git filter-repo

| Command | What It Does |
|---|---|
| `git filter-repo --path <path> --invert-paths` | Remove path from all history |
| `git filter-repo --path-glob "*.psd" --invert-paths` | Remove by glob pattern |
| `git filter-repo --path-regex "pattern" --invert-paths` | Remove by regex |
| `git filter-repo --strip-blobs-bigger-than 10M` | Remove all blobs > 10 MB |
| `git filter-repo --replace-text <file>` | Replace strings in all blobs |
| `git filter-repo --path <path>` | Keep ONLY this path (remove all else) |
| `git filter-repo --path-rename old:new` | Rename path across all history |

### Environment Variables

| Variable | Effect |
|---|---|
| `GIT_LFS_SKIP_SMUDGE=1` | Skip LFS file downloads during checkout |
| `GIT_TRACE_PACKET=1` | Show pack protocol communication |

---

## RULE 13 — When Would I Use This At Work?

### Scenario 1: GitHub Actions Clone Is Your Biggest CI Cost

```
Problem: Every PR triggers 6 jobs. Each job clones a 3 GB repo in 10 minutes.
         GitHub charges for compute time. 10 engineers, 8 PRs/day:
         10 × 8 × 6 × 10 min = 4,800 minutes/day = 80 engineer-hours wasted on clones.

Solution:
  .github/workflows/ci.yml:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 1
      filter: blob:none
      sparse-checkout: |
        ${{ env.SERVICE_DIR }}
        packages/shared
      sparse-checkout-cone-mode: true

Result:
  Clone time: 10 min → 30 sec (20x speedup)
  Monthly CI minutes saved: ~100,000 min (significant cost savings)

When you'd use this: Any team running more than ~50 CI runs per day.
```

### Scenario 2: Onboarding a New Engineer to a Monorepo

```
Problem: "git clone takes 45 minutes" is the first thing new engineers experience.
         They give up or clone overnight. Bad first impression.

Solution: Create an onboarding script:
  #!/bin/bash
  echo "Cloning monorepo (optimized)..."
  git clone --filter=blob:none --sparse https://github.com/org/monorepo.git
  cd monorepo
  git sparse-checkout init --cone
  
  echo "Which service will you be working on?"
  read -r SERVICE
  
  git sparse-checkout set "services/$SERVICE" "packages/shared" "packages/types"
  
  echo "Setting up background maintenance..."
  git maintenance start
  
  echo "Done! Total clone time: $(date)"

Result: Full clone: 45 min. Optimized onboarding: 2 min.
        New engineer is actually writing code on day 1.
```

### Scenario 3: Discovering a Secret Was Committed 6 Months Ago

```
Situation: Security audit reveals AWS_SECRET_KEY was committed to history
           6 months ago. It was removed in a commit 5 months ago, but
           anyone who cloned between those dates had access to it.

Immediate response:
  1. Revoke the AWS key immediately (regardless of git history)
  2. Create a new key
  3. Update the secret in GitHub Secrets (Topic 24)
  4. Schedule history rewrite for next maintenance window

History rewrite (coordinated with team):
  # 1. Announce: "Git history rewrite at 10pm. Everyone commit and push your work."
  # 2. At 10pm, freeze pushes to main
  # 3. Run on a mirror clone:
  git clone --mirror https://github.com/org/repo.git repo-clean.git
  cd repo-clean.git
  git filter-repo --replace-text <(echo "AKIAIOSFODNN7EXAMPLE==>***REMOVED***")
  git push --force origin --all --tags
  # 4. Send to team: "Re-clone tonight. All local clones are now diverged."
  # 5. Next day: verify with security team that the key is gone from GitHub's UI

Post-mortem: Add a pre-commit hook (Topic 19) that scans for AWS key patterns.
```

### Scenario 4: Setting Up Git LFS for a Game Studio

```
Scenario: 20-person game studio. Artists commit 2 GB of textures every week.
          Repo is 200 GB and growing. Engineers spend 3 hours each Monday
          pulling the latest assets.

Setup:
  git lfs install
  git lfs track "*.png" "*.tga" "*.psd" "*.fbx" "*.wav" "*.mp3" "*.ogg"
  git add .gitattributes && git commit -m "chore: configure Git LFS for game assets"

Migrate existing history (off-hours, coordinated):
  git lfs migrate import --include="*.png,*.tga,*.psd,*.fbx,*.wav,*.mp3,*.ogg" \
    --everything
  git push --force origin --all --tags
  # Team re-clones. Repo: 200 GB → 1.5 GB (130x reduction)

Configure for CI (don't download textures for code builds):
  # In .github/workflows/build.yml:
  env:
    GIT_LFS_SKIP_SMUDGE: 1   # CI doesn't need textures to compile C++
  - uses: actions/checkout@v4
  # LFS pointers are in working tree, but actual binary files are not downloaded
  # CI build takes 2 min instead of 45 min
  
Result: Monday morning pull: 3 hours → 4 minutes.
        CI build: 45 min → 2 min.
        Repo size: 200 GB → 1.5 GB.
        Artists: "Git finally works."
```

### Scenario 5: Forensic Analysis of a Large Repo Without Cloning

```
Scenario: You need to find which commit introduced a performance regression
          in a 5-year-old repo, but you're on a laptop with 30 GB free disk.
          The full clone would need 25 GB and take 2 hours.

Solution: Treeless partial clone + lazy bisect

  # Clone commits only (metadata, no files)
  git clone --filter=tree:0 https://github.com/org/big-repo.git
  # Clone time: 3 minutes (vs 2 hours). Download: 200 MB (vs 25 GB)
  
  # Set up bisect — reads only commits (no blobs needed for most operations)
  git bisect start
  git bisect bad HEAD
  git bisect good v2.0.0
  
  # Write a test that fetches only what it needs:
  git bisect run bash -c '
    # Only checkout the specific file that controls the algorithm
    git checkout HEAD -- src/algorithm/sort.ts
    # Run the benchmark on just that file
    node benchmark.js src/algorithm/sort.ts
  '
  # Each bisect step: fetches only the trees and specific blobs needed
  # Bisect of 1,000 commits: ~10 iterations, each fetching <10 MB lazily
  
  # Total data downloaded: ~100 MB (vs 25 GB for full clone)
  # Total time: 15 minutes (vs 2+ hours)
```

---

## EPILOGUE: The Complete Picture

You have now reached the end of all 26 topics. Here is how they form a single coherent system:

```
FOUNDATION (Topics 1-3):
  Every file in .git/objects/ is an immutable, content-addressed blob, tree, 
  commit, or tag. The DAG connects them. Refs are pointers into the DAG.

OPERATIONS (Topics 4-14):
  Every git command is a transformation of the 3 trees (working dir, index, HEAD):
  add → index; commit → HEAD; checkout → working dir; 
  branch/tag/ref → pointers into the DAG.

TEAM WORKFLOWS (Topics 15-19):
  PR → branch protection → required CI → CODEOWNERS → merge strategy.
  Hooks enforce quality locally; GitHub enforces it server-side.

RECOVERY & DEBUGGING (Topics 20-22):
  Nothing is ever truly lost (reflog). Binary search finds any bug in O(log N).
  Blame and log give you the "why" of every line.

SCALE (Topics 23-26):
  When repos grow: submodules or monorepos for organization (23).
  GitHub Actions + secrets for automation at scale (24).
  Packfiles + GC keep storage efficient (25).
  Shallow/sparse/partial/LFS/filter-repo handle size limits (26).
```

You now understand Git the way its creators understand it — from the bytes on disk 
up to the enterprise-scale workflows built on top. You can draw the object graph on 
a whiteboard, explain why `git push --force` is dangerous, debug any conflict, 
recover any lost commit, set up a production CI pipeline, and optimize any large repo.

*That's the complete curriculum. 26/26 topics done.*

---

*Topic 26 complete — 26/26 topics done. Curriculum complete.*
