# Topic 14: Tags & Releases
### (Lightweight vs Annotated Tags, SemVer, Physical Storage, Pushing Tags, GitHub Releases)

---

## CONNECTION TO PREVIOUS TOPICS (Rule 11)

Tags build directly on everything you know about refs and objects.

- **Topic 01**: You know `.git/refs/tags/` is a directory in `.git/`. Tags live there — one
  file per tag, just like `.git/refs/heads/` holds one file per branch.

- **Topic 02**: You know Git has 4 object types: blob, tree, commit, tag. The fourth one —
  the **tag object** — has never appeared in action until now. Annotated tags create this
  fourth object type. Lightweight tags do NOT.

- **Topic 03**: You know a branch is a pointer that MOVES (it follows new commits). A tag
  is a pointer that is **intentionally immutable** — it should never move. This is the
  fundamental distinction: branches track the present; tags preserve the past.

- **Topic 08**: You know a branch file contains a 40-char SHA + newline. A lightweight tag
  file is IDENTICAL in format. The ONLY difference is where it lives (refs/tags vs
  refs/heads) and the convention that you don't move it.

- **Topic 10**: You know `git push` does NOT push tags by default. Tags require an explicit
  push. You'll understand why when you see how tags live in `refs/tags/` locally but are
  absent on the remote until you push them.

---

## RULE 1: ELI5 ANCHOR — The Snapshot Label

Imagine your software project is a long roll of film, with each frame being a commit.
Branches are like a "YOU ARE HERE" marker that slides forward every time you take a new photo.

A **tag** is a physical sticker you press onto one specific frame of the film and leave
there permanently. It says "This frame is version 1.4.2 — it will always be 1.4.2."
No matter how much more film you shoot, that sticker stays on that frame forever.

A **lightweight tag** is a Post-it note: tiny, just a name written on paper, stuck to
the frame. Cheap, no extra information.

An **annotated tag** is a proper metal embossed label: it has a name, the tagger's
signature, the date it was applied, and a message explaining why this frame matters.
It's tamper-evident — if anyone changes the commit it was applied to, the tag's hash
changes too.

```
FILM ROLL (commit history):
  frame1──frame2──frame3──frame4──frame5
                   │               │
                   └─ v1.0.0       └─ YOU ARE HERE (branch: main)
                      (tag: immovable)
```

---

## RULE 2: PHYSICAL ARCHITECTURE FIRST

### Lightweight tag — what it looks like on disk

```
.git/
└── refs/
    └── tags/
        └── v1.0.0          ← one file, 41 bytes
                               Contents: "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\n"
                               (SHA of the commit being tagged — no tag object involved)
```

```bash
# Verify:
cat .git/refs/tags/v1.0.0
# a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

git cat-file -t a1b2c3d4   # what type is this SHA?
# commit    ← points directly to a COMMIT object
```

### Annotated tag — what it looks like on disk

```
.git/
├── refs/
│   └── tags/
│       └── v1.0.0          ← one file, 41 bytes
│                              Contents: SHA of the TAG OBJECT (not the commit!)
└── objects/
    └── d4/                 ← first 2 chars of tag object SHA
        └── e5f6a7b8...     ← the TAG OBJECT (a 4th Git object type)
```

```bash
# The ref file points to a TAG OBJECT:
cat .git/refs/tags/v1.0.0
# d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3  ← SHA of a TAG object

git cat-file -t d4e5f6a7   # what type is this SHA?
# tag    ← points to a TAG object, NOT directly to the commit

git cat-file -p d4e5f6a7   # inspect the tag object
# object a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0   ← SHA of the COMMIT
# type commit
# tag v1.0.0
# tagger Alice Smith <alice@company.com> 1748600000 +0530
#
# Release v1.0.0: First stable production release
# 
# Changelog:
# - Add user authentication
# - Add payment processing
# - Fix login rate limiting
```

### The two-level indirection of annotated tags

```
.git/refs/tags/v1.0.0
       │
       ▼
  TAG OBJECT (d4e5f6...)
  ├── object: a1b2c3...   ← points to commit
  ├── type: commit
  ├── tag: v1.0.0
  ├── tagger: Alice Smith <alice@company.com> 1748600000 +0530
  └── message: "Release v1.0.0: First stable production release..."
       │
       ▼
  COMMIT OBJECT (a1b2c3...)
  ├── tree: ...
  ├── parent: ...
  └── message: "Merge branch 'release/1.0.0'"
```

### The full `.git/` picture with multiple tags

```
.git/
├── HEAD                              ← "ref: refs/heads/main"
├── refs/
│   ├── heads/
│   │   ├── main                      ← moves forward with new commits
│   │   └── feature/auth              ← moves forward with new commits
│   └── tags/
│       ├── v0.1.0                    ← lightweight: points to commit SHA directly
│       ├── v1.0.0                    ← annotated: points to tag object SHA
│       ├── v1.1.0                    ← annotated: points to tag object SHA
│       └── v2.0.0-beta.1             ← pre-release tag
├── packed-refs                       ← older tags may be packed here (same as branches)
│   # Contents example:
│   # ^a1b2c3... (commit SHA dereferenced from annotated tag)
│   # d4e5f6... refs/tags/v1.0.0    ← tag object SHA
│   # e5f6a7... refs/tags/v0.1.0    ← lightweight tag, points to commit directly
└── objects/
    ├── d4/e5f6...                    ← tag object for v1.0.0
    ├── e5/f6a7...                    ← tag object for v1.1.0
    └── [all commits, trees, blobs]
```

### `packed-refs` — where tags end up after `git gc` or `git pack-refs`

```bash
# Raw packed-refs file (after git pack-refs --all):
cat .git/packed-refs
# # pack-refs with: peeled fully-peeled sorted
# d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3 refs/tags/v1.0.0
# ^a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0    ← peeled: commit SHA the tag points to
# e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4 refs/tags/v0.1.0
# (no ^ line for lightweight tags — they point directly to commits)
```

The `^` line is called the "peeled" value — it's the commit the annotated tag ultimately
points to, pre-computed and stored for performance (so `git log` doesn't need to
dereference two levels each time it renders a tag).

---

## RULE 3: EXACT DATA FLOW — Every Tag Command

### `git tag v1.0.0` (lightweight)

```
COMMAND: git tag v1.0.0

READS:   .git/HEAD → current commit SHA (e.g., a1b2c3...)
WRITES:  .git/refs/tags/v1.0.0
         Contents: "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\n"

NO TAG OBJECT IS CREATED.
The ref file points DIRECTLY to the commit object.

NET EFFECT:
  One new 41-byte file in refs/tags/.
  No new object in .git/objects/.
  Tag is an alias to the commit SHA — nothing more.
```

### `git tag -a v1.0.0 -m "Release v1.0.0"` (annotated)

```
COMMAND: git tag -a v1.0.0 -m "Release v1.0.0"

STEP 1: COMPUTE TAG OBJECT CONTENT
  The tag object content is:
    "object a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\n"
    "type commit\n"
    "tag v1.0.0\n"
    "tagger Alice Smith <alice@co.com> 1748600000 +0530\n"
    "\n"
    "Release v1.0.0\n"

STEP 2: COMPUTE SHA-1 OF TAG OBJECT
  header = "tag " + str(len(content)) + "\0"
  SHA-1(header + content) → d4e5f6a7...

STEP 3: WRITE TAG OBJECT
  WRITES: .git/objects/d4/e5f6a7...
          (zlib-compressed, same format as any other object)

STEP 4: WRITE TAG REF
  WRITES: .git/refs/tags/v1.0.0
          Contents: "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3\n"
          (SHA of the TAG OBJECT, not the commit)

NET EFFECT:
  One new tag object in .git/objects/.
  One new 41-byte ref file in .git/refs/tags/.
  Total: 2 new things on disk.
```

### `git tag -s v1.0.0 -m "Release"` (GPG-signed annotated tag)

```
COMMAND: git tag -s v1.0.0 -m "Release v1.0.0"

SAME AS -a EXCEPT:
  STEP 2.5: After composing tag object content, Git calls GPG to sign it
            GPG produces a PGP signature block
            The signature is APPENDED to the tag message

  The signed tag object content:
    object a1b2c3...
    type commit
    tag v1.0.0
    tagger Alice Smith <alice@co.com> 1748600000 +0530

    Release v1.0.0
    -----BEGIN PGP SIGNATURE-----
    [base64-encoded GPG signature block]
    -----END PGP SIGNATURE-----

  WRITES: .git/objects/[sha of signed tag object]
          (the signature is part of the object content — it's hashed into the SHA)

VERIFICATION:
  git tag -v v1.0.0
  → Calls GPG to verify signature against tagger's public key
  → "Good signature from Alice Smith <alice@co.com>"
```

### `git push origin v1.0.0` (push a single tag)

```
COMMAND: git push origin v1.0.0

READS:   .git/refs/tags/v1.0.0 → SHA
         For annotated tags: .git/objects/[tag-sha] (the tag object itself)
SENDS:   The tag object + all objects it references to origin
MODIFIES: origin's refs/tags/v1.0.0 ← set to the tag SHA

NOTE: git push origin --tags  → pushes ALL tags not yet on remote
      git push origin --follow-tags → pushes only annotated tags reachable from pushed commits
      DEFAULT: git push alone pushes NO tags (by design)
```

### `git tag -d v1.0.0` (delete a local tag)

```
COMMAND: git tag -d v1.0.0

READS:   .git/refs/tags/v1.0.0
DELETES: .git/refs/tags/v1.0.0

NOTE: The tag OBJECT (if annotated) remains in .git/objects/ — unreachable until GC.
      The COMMIT it pointed to is unaffected.
      
DELETE FROM REMOTE:
  git push origin --delete v1.0.0
  OR: git push origin :refs/tags/v1.0.0   (empty source = delete)
```

---

## RULE 4: ASCII ARCHITECTURE DIAGRAMS

### Diagram 1: Branch vs Tag — the moving vs fixed pointer

```
BRANCHES (move forward):                TAGS (stay fixed):

  A──B──C──D──E   main                   A──B──C──D──E   main
                ^                                ^
                │                               HEAD
               HEAD
                                        v1.0.0 ────► B  (stays here forever)
  After new commit F:                   v2.0.0 ────► D  (stays here forever)
  A──B──C──D──E──F   main
                  ^
                  HEAD (MOVED)           v1.0.0 still ────► B
                                         v2.0.0 still ────► D
```

### Diagram 2: Lightweight vs Annotated tag — object graph comparison

```
LIGHTWEIGHT TAG:                          ANNOTATED TAG:
  
  refs/tags/v1.0.0                          refs/tags/v1.0.0
       │                                          │
       │ (points directly)                        │ (points to tag object)
       ▼                                          ▼
  COMMIT object                             TAG object
  ├── tree                                  ├── object ──────────────► COMMIT object
  ├── parent                                ├── type: commit           ├── tree
  ├── author                                ├── tag: v1.0.0            ├── parent
  └── message                               ├── tagger: Alice Smith    ├── author
                                            └── message: "Release..."  └── message

  No extra object                           1 extra TAG object
  in .git/objects/                          in .git/objects/
```

### Diagram 3: SemVer tag naming on the DAG

```
RELEASE HISTORY:
  
  A──B──C──D──E──F──G──H──I──J──K  main
        │     │        │        │
        │     │        │        └─ v2.0.0  (major: breaking change)
        │     │        └─────────  v1.2.0  (minor: new features)
        │     └──────────────────  v1.1.0  (minor: new features)
        └────────────────────────  v1.0.0  (first stable)
  
  Off the release branches:
  
       E──F──G  release/1.1
            │
            └─ v1.1.1  (patch: bug fix in v1.1.0)

  PATCH TAGS (bug fixes):
  v1.0.1, v1.0.2   ← small fixes to v1.0 series
  v1.1.0, v1.1.1   ← v1.1 series
  
  PRE-RELEASE TAGS:
  v2.0.0-alpha.1, v2.0.0-beta.1, v2.0.0-rc.1, v2.0.0  ← release candidate lifecycle
```

### Diagram 4: `git describe` — finding the nearest tag

```
HISTORY:
  A──B──C──D──E──F──G   main
        │
        v1.0.0

git describe HEAD (when HEAD is at G):
  v1.0.0-5-gabcdef1
  │       │  │
  │       │  └─ g + first 7 chars of HEAD SHA
  │       └─ 5 commits after the tag
  └─ most recent reachable tag

git describe HEAD --tags (includes lightweight tags)
git describe HEAD --abbrev=0 (just the tag name: v1.0.0)

USE IN CI: VERSION=$(git describe --tags --always) to embed version in build artifacts
```

### Diagram 5: The packed-refs file structure

```
.git/packed-refs (after git pack-refs --all):

# pack-refs with: peeled fully-peeled sorted
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 refs/heads/main
b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1 refs/heads/feature/auth
d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3 refs/tags/v1.0.0    ← tag object SHA
^a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0                    ← peeled: commit SHA
e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4 refs/tags/v0.1.0    ← lightweight: commit SHA directly
(no ^ line for lightweight tag)

LOOKUP ORDER for any ref:
1. Check .git/refs/<path>  (loose ref file — fastest for active refs)
2. Check .git/packed-refs  (packed file — for historical refs)
Loose file always wins if both exist.
```

---

## RULE 5: HANDS-ON PROOF COMMANDS

### Proof Lab 1: Lightweight vs annotated — the physical difference

```bash
mkdir tags-lab && cd tags-lab
git init
git config user.email "t@t.com" && git config user.name "Test"

# Create some commits
echo "v0 content" > app.txt && git add . && git commit -m "Initial commit"
COMMIT_1=$(git rev-parse HEAD)
echo "v1 content" >> app.txt && git commit -am "Add v1 feature"
COMMIT_2=$(git rev-parse HEAD)

# ── LIGHTWEIGHT TAG ──
git tag v0.1.0 $COMMIT_1    # tag the first commit

# PROVE IT: Tag is just a file in refs/tags/
cat .git/refs/tags/v0.1.0
# [SHA of COMMIT_1]  ← points directly to commit

# PROVE IT: The SHA in the tag file IS the commit SHA
echo "Tag SHA:    $(cat .git/refs/tags/v0.1.0)"
echo "Commit SHA: $COMMIT_1"
# They match!

# PROVE IT: No new object was created
git cat-file -t $(cat .git/refs/tags/v0.1.0)
# commit    ← NOT a tag object — it's directly a commit

# ── ANNOTATED TAG ──
git tag -a v1.0.0 $COMMIT_2 -m "First major release: stable auth and payments"

# PROVE IT: Tag ref points to a TAG OBJECT (not the commit)
TAG_SHA=$(cat .git/refs/tags/v1.0.0)
echo "Tag ref SHA: $TAG_SHA"

git cat-file -t $TAG_SHA
# tag    ← this IS a tag object!

# PROVE IT: Inspect the tag object
git cat-file -p $TAG_SHA
# object [COMMIT_2 SHA]     ← points to the commit
# type commit
# tag v1.0.0
# tagger Test <t@t.com> [timestamp]
#
# First major release: stable auth and payments

# PROVE IT: The tag object SHA ≠ the commit SHA
echo "Tag object SHA: $TAG_SHA"
echo "Commit SHA:     $COMMIT_2"
# They are DIFFERENT

# PROVE IT: Dereference the annotated tag to get the commit
git rev-parse v1.0.0^{}   # ^{} = dereference to the underlying object
# [COMMIT_2 SHA]

git rev-parse v1.0.0      # without ^{} — gives tag object SHA
# [TAG_SHA]

git rev-parse v0.1.0      # lightweight tag
# [COMMIT_1 SHA] — same as tag SHA because lightweight has no tag object
```

### Proof Lab 2: Listing and filtering tags

```bash
# Create several tags with SemVer naming
git tag v0.1.0 HEAD~1           # already created above
git tag -a v1.0.0 HEAD -m "Major release"  # already created
git tag -a v1.1.0-beta.1 HEAD -m "Beta"
git tag -a v1.1.0 HEAD -m "Minor release"
git tag v1.1.1 HEAD             # lightweight patch tag

# PROVE IT: List all tags
git tag
# v0.1.0
# v1.0.0
# v1.1.0
# v1.1.0-beta.1
# v1.1.1

# PROVE IT: List tags matching a pattern
git tag -l "v1.*"
# v1.0.0
# v1.1.0
# v1.1.0-beta.1
# v1.1.1

# PROVE IT: List tags with annotations
git tag -n1        # show first line of annotation
# v0.1.0
# v1.0.0       First major release: stable auth and payments
# v1.1.0       Minor release
# v1.1.0-beta.1 Beta
# v1.1.1

# PROVE IT: List tags sorted by version (not alphabetically)
git tag -l --sort=version:refname "v*"
# v0.1.0
# v1.0.0
# v1.1.0
# v1.1.0-beta.1  ← note: -beta comes before release in default sort
# v1.1.1

# Correct SemVer sort (pre-release before release):
git tag -l --sort=-version:refname   # descending
# v1.1.1
# v1.1.0-beta.1
# ...
```

### Proof Lab 3: Tags in git log

```bash
# PROVE IT: Tags appear in git log as decorations
git log --oneline --decorate
# [sha2] (HEAD -> main, tag: v1.1.1, tag: v1.1.0, tag: v1.1.0-beta.1, tag: v1.0.0) Add v1 feature
# [sha1] (tag: v0.1.0) Initial commit

# PROVE IT: git describe finds the nearest tag
git describe HEAD
# v1.1.1  (or v1.1.0 depending on which was created last on HEAD)

# PROVE IT: See tag info
git show v1.0.0
# tag v1.0.0
# Tagger: Test <t@t.com>
# Date:   [date]
# 
# First major release: stable auth and payments
# 
# commit [sha2] (HEAD -> main, tag: v1.1.1, ...)
# Author: Test <t@t.com>
# Date:   [date]
# 
#     Add v1 feature
# 
# diff --git a/app.txt b/app.txt
# ...

# For lightweight tag, git show skips the tag header and shows the commit directly:
git show v0.1.0
# commit [sha1] (tag: v0.1.0)
# Author: Test <t@t.com>
# ...
```

### Proof Lab 4: Pushing tags to remote (simulated with a second local repo)

```bash
# Create a "remote" (second local repo simulating origin)
cd /tmp && git init --bare tags-remote.git && cd -
git remote add origin /tmp/tags-remote.git

# PROVE IT: Normal push doesn't include tags
git push origin main
git ls-remote origin --tags
# (empty — no tags pushed)

# Push a specific tag
git push origin v1.0.0
git ls-remote origin --tags
# d4e5f6... refs/tags/v1.0.0
# ^a1b2c3... refs/tags/v1.0.0^{}   ← peeled ref (commit SHA)

# Push all tags
git push origin --tags
git ls-remote origin --tags
# All tags now on remote

# PROVE IT: Delete a tag from remote
git push origin --delete v0.1.0
git ls-remote origin --tags | grep v0.1.0
# (empty — deleted)
```

### Proof Lab 5: `git describe` in action

```bash
# Navigate to a commit between tags
git checkout HEAD~1   # detached HEAD — one commit before tip

git describe HEAD
# v0.1.0  (if HEAD~1 is at the v0.1.0 commit)
# or: v1.0.0-1-gabcdef1 (if 1 commit after v1.0.0)

# Use in a build script:
VERSION=$(git describe --tags --long --always)
echo $VERSION
# v1.0.0-0-gabcdef1  (0 commits after tag, g = git prefix, then SHA)
# If no tags at all: just the SHA (--always ensures output even with no tags)

git switch main   # back to branch
```

---

## SEMANTIC VERSIONING (SemVer) — THE STANDARD

### The SemVer spec decoded

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILDMETA]

  v  2  .  1  .  4  -  beta.1  +  build.2025
  │  │     │     │     │          │
  │  │     │     │     │          └─ Build metadata (ignored for version ordering)
  │  │     │     │     └─────────── Pre-release identifier (lower precedence than release)
  │  │     │     └───────────────── PATCH: Bug fixes (backwards compatible)
  │  │     └─────────────────────── MINOR: New features (backwards compatible)
  │  └───────────────────────────── MAJOR: Breaking changes (not backwards compatible)
  └──────────────────────────────── Optional 'v' prefix (Git convention, not SemVer spec)

RULES:
  1. Once a version is released, its contents must not change. Any change = new version.
  2. 0.y.z: initial development — anything may change. Public API not stable.
  3. 1.0.0: first stable public API.
  4. PATCH: increment for backwards-compatible bug fixes.
  5. MINOR: increment for backwards-compatible new functionality. Reset PATCH to 0.
  6. MAJOR: increment for incompatible API changes. Reset MINOR and PATCH to 0.
  7. Pre-release: lower precedence. v1.0.0-alpha < v1.0.0-beta < v1.0.0-rc.1 < v1.0.0
```

### SemVer precedence ordering

```
0.1.0 < 0.9.0 < 1.0.0-alpha < 1.0.0-alpha.1 < 1.0.0-beta < 1.0.0-beta.2 < 1.0.0-rc.1 < 1.0.0
      < 1.0.1 < 1.1.0 < 2.0.0-beta < 2.0.0 < 10.0.0

COMMON PRE-RELEASE IDENTIFIERS (in order from earliest to latest):
  alpha   → Early development, may be very unstable, internal
  beta    → Feature-complete but potentially buggy, external testing
  rc.1    → Release candidate — production-ready unless a critical bug found
  (none)  → Stable release
```

### What "breaking change" means in practice (backend API example)

```
PATCH examples (v1.0.0 → v1.0.1):
  - Fix a bug where GET /users returned incorrect pagination
  - Fix a memory leak
  - Fix incorrect error message text
  - Performance improvement with no interface change

MINOR examples (v1.0.1 → v1.1.0):
  - Add new endpoint POST /users/bulk
  - Add optional field 'metadata' to GET /users response
  - Add new optional query parameter to existing endpoint
  - Deprecate an old endpoint (but keep it working)

MAJOR examples (v1.1.0 → v2.0.0):
  - Remove endpoint GET /users/legacy
  - Change response format of GET /users (field renames, type changes)
  - Change authentication mechanism (API keys → OAuth)
  - Change required request body format
  - Remove previously-deprecated endpoint
```

### Conventional Commits → SemVer automation (connecting Topic 07)

```
COMMIT TYPE → VERSION BUMP:

  fix: ...           → PATCH (1.0.0 → 1.0.1)
  feat: ...          → MINOR (1.0.0 → 1.1.0)
  feat!: ...         → MAJOR (1.0.0 → 2.0.0)
  BREAKING CHANGE:   → MAJOR (in footer of any commit type)

  ci:, docs:, style:, refactor:, perf:, test:, chore: → no version bump

TOOLS THAT AUTOMATE THIS:
  semantic-release    → reads Conventional Commits, determines version, 
                        creates annotated tag, generates CHANGELOG, creates GitHub release
  
  standard-version    → similar, more configurable
  
  release-please      → Google's tool, creates PRs for releases
```

---

## RULE 6: SYNTAX BREAKDOWN (Deep)

### `git tag`

```
git tag [-a | -s | -u <key-id>] [-f] [-d] [-v]
        [-m <msg> | -F <file>]
        [-l [<pattern>]] [--sort=<key>] [-n[<num>]]
        [<tagname> [<commit> | <object>]]
│       │                                │           │
│       │                                │           └─ what to tag (default: HEAD)
│       │                                └─ the tag name (e.g., v1.0.0)
│       └─ mode flags:
│          -a  = annotated (creates a tag object, prompts for message)
│          -s  = GPG-signed (like -a but also signs with your GPG key)
│          -u <key-id> = sign with a specific GPG key
│          -f  = force: move tag even if it already exists (be careful on pushed tags!)
│          -d  = delete: git tag -d v1.0.0
│          -v  = verify: check GPG signature on a signed tag
└─ git binary

EXAMPLES:
  git tag v1.0.0                           # lightweight tag at HEAD
  git tag v1.0.0 a1b2c3d4                  # lightweight tag at specific commit
  git tag -a v1.0.0 -m "Release notes"     # annotated at HEAD
  git tag -a v1.0.0 a1b2c3d4 -m "Notes"   # annotated at specific commit
  git tag -s v1.0.0 -m "Signed release"   # GPG-signed annotated
  git tag -l "v1.*"                        # list tags matching pattern
  git tag -l --sort=-version:refname       # list sorted by SemVer, newest first
  git tag -n3                              # list with first 3 lines of annotation
  git tag -d v1.0.0                        # delete local tag
  git tag -f v1.0.0 HEAD                  # MOVE a tag (dangerous if pushed)
```

### `git push` (tag-related flags)

```
git push <remote> <tagname>                 # push a specific tag
git push <remote> --tags                    # push ALL tags not on remote
git push <remote> --follow-tags             # push tags reachable from pushed commits
git push <remote> --delete <tagname>        # delete tag from remote
git push <remote> :refs/tags/<tagname>      # alternative delete syntax

DIFFERENCE:
  --tags:         Pushes ALL local tags not on remote (including old ones)
  --follow-tags:  Pushes only annotated tags that are reachable from the commits
                  you're pushing. Preferred for release workflows.
```

### `git describe`

```
git describe [--tags] [--always] [--long] [--abbrev=<n>] [--match <pattern>] [<commit>]
│            │         │          │        │               │                   │
│            │         │          │        │               │                   └─ which commit (default HEAD)
│            │         │          │        │               └─ only match tags matching pattern
│            │         │          │        └─ length of the SHA suffix (default 7)
│            │         │          └─ always show long format: v1.0.0-0-gabcdef1 even if on tag
│            │         └─ always produce output (use commit SHA if no tags reachable)
│            └─ include lightweight tags (default: annotated tags only)
└─ git binary

FORMAT: <tag>-<N>-g<SHA>
  <tag>: most recent reachable annotated tag
  <N>:   number of commits since the tag (0 if HEAD is exactly on the tag)
  g<SHA>: "g" + 7-char abbreviated HEAD SHA

EXAMPLES:
  git describe HEAD                  # v1.0.0 (if HEAD is exactly on tag) or v1.0.0-3-gabc1234
  git describe HEAD --tags           # include lightweight tags
  git describe HEAD --abbrev=0       # just the tag name: v1.0.0
  git describe HEAD --long           # always show long form: v1.0.0-0-gabc1234
  git describe HEAD --always         # output even if no tag found (just SHA)
  git describe HEAD --match "v1.*"   # only match v1.x tags
```

---

## RULE 7: BEGINNER → PRODUCTION EXAMPLE

### Beginner: Create and push your first release tag

```bash
# You've finished your project and it's ready for v1.0.0
git log --oneline
# a1b2c3 Fix final bug
# b2c3d4 Add documentation
# c3d4e5 Implement core feature

# Create an annotated release tag at HEAD
git tag -a v1.0.0 -m "Version 1.0.0

Initial stable release.

Features:
- User authentication
- Basic CRUD operations
- Rate limiting"

# Verify the tag
git show v1.0.0

# Push to remote
git push origin v1.0.0

# PROVE IT: Tag is on GitHub now
# Go to GitHub repo → Tags → v1.0.0 is visible
```

### Production: Full release workflow on a 6-person backend team

**Scenario**: You're a tech lead. The team uses `feature/*` branches with a `release/*`
branch workflow. You're cutting version 2.3.0.

```bash
# ── PHASE 1: CREATE RELEASE BRANCH ──
git fetch origin
git checkout -b release/2.3.0 origin/main

# Final stabilization: only bug fixes on this branch
# Run full test suite, QA testing, etc.

# ── PHASE 2: VERSION BUMP ──
# Update version in code (package.json, pyproject.toml, build.gradle, etc.)
sed -i 's/"version": "2.2.0"/"version": "2.3.0"/' package.json
git commit -am "chore(release): bump version to 2.3.0"

# ── PHASE 3: TAG THE RELEASE ──
git tag -a v2.3.0 -m "Release v2.3.0

## What's New
- feat: Add webhook signature validation (#234)
- feat: Support bulk user operations endpoint (#241)

## Bug Fixes  
- fix: Correct pagination in GET /users (#238)
- fix: Handle empty request body in POST /events (#242)

## Breaking Changes
None.

Full changelog: https://github.com/company/api/compare/v2.2.0...v2.3.0"

# ── PHASE 4: MERGE BACK TO MAIN AND DEVELOP ──
git switch main
git merge release/2.3.0 --no-ff -m "Merge release/2.3.0 into main"
git push origin main

git switch develop
git merge release/2.3.0 --no-ff -m "Merge release/2.3.0 into develop"
git push origin develop

# ── PHASE 5: PUSH TAG ──
git push origin v2.3.0

# ── PHASE 6: CREATE GITHUB RELEASE (via CLI or UI) ──
gh release create v2.3.0 --title "v2.3.0" --notes-file CHANGELOG.md

# ── PHASE 7: CLEAN UP RELEASE BRANCH ──
git branch -d release/2.3.0
git push origin --delete release/2.3.0

# ── RESULT ──
# GitHub shows v2.3.0 release with changelog
# CI/CD pipeline triggered on tag creation → deploys to production
# Users of the API can see what changed in this version
```

### Production: Automated tagging with semantic-release

```yaml
# .releaserc.json — semantic-release configuration
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",      # reads Conventional Commits
    "@semantic-release/release-notes-generator", # generates CHANGELOG
    "@semantic-release/changelog",            # writes CHANGELOG.md
    "@semantic-release/npm",                  # bumps package.json version
    "@semantic-release/github",               # creates GitHub release + tag
    ["@semantic-release/git", {               # commits CHANGELOG.md + package.json
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}

# .github/workflows/release.yml
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # ← IMPORTANT: fetch all history for semantic-release
          persist-credentials: false
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

# What happens on every push to main:
# 1. semantic-release reads commits since last tag
# 2. Determines version bump (fix→PATCH, feat→MINOR, breaking→MAJOR)
# 3. Creates annotated git tag (e.g., v2.3.0)
# 4. Generates CHANGELOG.md entry
# 5. Pushes tag to GitHub
# 6. Creates GitHub Release with generated notes
# 7. Publishes to npm (if configured)
```

---

## RULE 8: MISTAKES → ROOT CAUSE → FIX → PREVENTION

### Mistake 1: Tagging the wrong commit

**What happened**: You ran `git tag v1.0.0` without specifying a commit.
HEAD was 2 commits ahead of where you wanted to tag.

**Root cause**: `git tag <name>` without a commit argument tags HEAD.

**Fix**:
```bash
# Delete the wrong tag locally
git tag -d v1.0.0

# Create the correct tag on the right commit
git log --oneline | head -5   # find the right SHA
git tag -a v1.0.0 <correct-sha> -m "Release notes"

# If you already pushed the wrong tag:
git push origin --delete v1.0.0   # delete from remote
git push origin v1.0.0            # push the corrected tag
# WARN teammates if any CI was triggered by the wrong tag
```

**Prevention**: Always check `git log --oneline` before tagging to confirm which commit is HEAD.

---

### Mistake 2: Moving a published tag with `git tag -f`

**What happened**: You moved `v1.0.0` to a different commit with `git tag -f v1.0.0 <new-sha>`
and force-pushed the tag. Now colleagues who already fetched `v1.0.0` have a different
commit than what the remote says is `v1.0.0`.

**Root cause**: Tags are meant to be immutable. Moving a tag after it's published
breaks the contract — the same tag name now refers to different code for different people.

**How it breaks**:
```bash
# Alice pulled yesterday: v1.0.0 → commit A
# You force-moved: v1.0.0 → commit B
# Bob pulls today: v1.0.0 → commit B

# Alice and Bob have different code for "v1.0.0"
# Their `git fetch` won't update Alice's tag because tags don't auto-update

# To see the problem:
git fetch origin
git rev-parse v1.0.0        # local (Alice): SHA of commit A
git rev-parse origin/v1.0.0 # remote (your change): SHA of commit B
```

**Fix**: Create a NEW tag (v1.0.1 or v1.0.0-hotfix) instead of moving the old one.
If the move was an error and no one else pulled yet, force-push and alert the team.

**Prevention**: NEVER move a published tag. Treat published tags as immutable as commits.

---

### Mistake 3: Lightweight tag when annotated was intended

**What happened**: Your CI system or tooling expects annotated tags (e.g., `git describe`
only shows annotated tags by default; some GitHub Release tools only recognize annotated tags).

**Diagnosis**:
```bash
git cat-file -t v1.0.0
# commit   ← lightweight (bad)
# tag      ← annotated (good)
```

**Fix**:
```bash
# Delete and recreate as annotated
git tag -d v1.0.0
git tag -a v1.0.0 <sha> -m "Release v1.0.0"
git push origin --delete v1.0.0
git push origin v1.0.0
```

**Prevention**: Use annotated tags for all releases. Use lightweight tags only for
temporary/personal bookmarks that are never pushed.

---

### Mistake 4: Forgetting to push tags after pushing commits

**What happened**: You pushed your commits to main and deployed. But you forgot
`git push origin v2.0.0`. CI didn't create a release. Monitoring tools showing "unknown version."

**Root cause**: `git push` does NOT push tags by default (by design — tags can be private,
test-only, temporary). This surprises many developers who expect push to push everything.

**Fix**:
```bash
git push origin v2.0.0    # push the specific tag
# OR
git push origin --tags     # push all unpushed tags
```

**Prevention**: Use `push.followTags = true` in your config:
```bash
git config --global push.followTags true
# Now: git push automatically includes annotated tags reachable from pushed commits
```

---

### Mistake 5: Tagging a merge commit vs tagging a feature commit

**What happened**: You tagged the MERGE COMMIT instead of the commit before it.
`git show v1.0.0` shows "Merge branch 'release/1.0.0'" as the tagged commit message.

**Impact**: Minor — but confusing for `git show v1.0.0` output and `git log` decoration.

**Best practice**: Tag the last "real" commit before the merge, or the merge commit is
fine if you're using `--no-ff` merge commits as release points.

Many teams specifically structure their release workflow so the tagged commit has a
meaningful message like "chore(release): 1.0.0" not "Merge branch 'release/1.0.0'".

---

### Mistake 6: Using tags in GitHub as branches (trying to push TO a tag)

**What happened**: You tried to commit work "onto a tag" — treating a tag like a branch.

**Root cause**: Misunderstanding the immutable nature of tags. Tags point to one fixed commit.
You can CHECK OUT a tag (detached HEAD), but you cannot push commits to update a tag.

**Fix**: If you need to do work based on a tagged release:
```bash
# Correct approach: create a branch FROM the tag
git checkout -b hotfix/v1.0.1 v1.0.0
# make fixes, commit
git tag -a v1.0.1 -m "Hotfix release"
git push origin hotfix/v1.0.1
git push origin v1.0.1
```

---

## RULE 9: BYTE-LEVEL INTERNALS

### Exact tag object format on disk

```
ANNOTATED TAG OBJECT — raw content (before header and zlib):

"object a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0\n"   ← 47 bytes
"type commit\n"                                           ← 12 bytes
"tag v1.0.0\n"                                           ← 11 bytes
"tagger Alice Smith <alice@co.com> 1748600000 +0530\n"  ← variable
"\n"                                                      ← 1 byte (empty line)
"Release v1.0.0\n"                                       ← variable

HEADER (prepended before hashing):
"tag " + str(total_content_length) + "\0"

SHA-1 computed over: header + content
This SHA becomes the tag object's identity.

STORAGE:
zlib_compress(header + content) → written to .git/objects/<first2>/<remaining38>

PROOF:
git cat-file tag v1.0.0        # shows raw tag object content
git cat-file -s v1.0.0         # shows size in bytes
git cat-file -t v1.0.0         # shows type: "tag"
```

### Tag dereferencing operators

```bash
# These all work the same way (applied to annotated tags):
git rev-parse v1.0.0       # SHA of the TAG OBJECT
git rev-parse v1.0.0^{}    # SHA of the COMMIT (dereferences the tag object)
git rev-parse v1.0.0^{commit}  # explicitly dereference to a commit
git rev-parse v1.0.0^{tag}     # explicitly get the tag object (fails on lightweight)

# For lightweight tags (no tag object):
git rev-parse v0.1.0       # SHA of the COMMIT (directly — no object to dereference)
git rev-parse v0.1.0^{}    # same SHA (^{} is a no-op for lightweight)

# PRACTICAL:
# When you want the commit a tag points to (regardless of lightweight or annotated):
git rev-parse <tagname>^{}
# This always gives you the commit SHA.
```

### How Git resolves `v1.0.0` in commands like `git checkout v1.0.0`

```
Git disambiguation order for "v1.0.0":

1. .git/refs/tags/v1.0.0  → check if file exists
2. .git/packed-refs         → search for "refs/tags/v1.0.0" in packed refs
3. Interpret as branch: .git/refs/heads/v1.0.0 (unlikely for SemVer names)
4. Interpret as a SHA (if 4+ hex chars)

For annotated tags:
  Found SHA "d4e5f6..." in refs/tags/v1.0.0
  → read .git/objects/d4/e5f6...
  → object type is "tag"
  → follow "object" field to commit SHA "a1b2c3..."
  → checkout the commit a1b2c3 (detached HEAD)
```

---

## TAGS vs BRANCHES — THE COMPLETE COMPARISON

| Property | Branch | Tag |
|----------|--------|-----|
| Location | `.git/refs/heads/` | `.git/refs/tags/` |
| File format | 41-byte SHA + newline | 41-byte SHA + newline (same!) |
| Points to | Commit (always) | Commit (lightweight) or Tag object → Commit (annotated) |
| Moves on commit? | YES — always advances | NO — immutable by convention |
| Can be pushed with `git push`? | YES (default) | NO (must be explicit) |
| Has metadata? | No | Annotated: yes (tagger, date, message, GPG sig) |
| Shown in `git log` | As HEAD decoration | As decorations next to commit |
| `git checkout <name>` | Attaches HEAD to branch | Detaches HEAD at the commit |
| Deleted with | `git branch -d` | `git tag -d` |
| Remote delete | `git push origin --delete <branch>` | `git push origin --delete <tag>` |

---

## GITHUB RELEASES — THE LAYER ON TOP OF TAGS

### What a GitHub Release is

A GitHub Release is a GitHub-layer object that:
1. **Wraps** a Git tag (a release must have an associated tag)
2. **Adds** a web UI with formatted release notes
3. **Attaches** binary assets (`.zip`, `.tar.gz`, binaries, `.deb`, `.rpm`, etc.)
4. **Sends** notifications to watchers
5. **Sets** a "latest release" marker used by shields.io badges and package managers

```
TAG (Git layer):                           GITHUB RELEASE (GitHub layer):
  .git/refs/tags/v1.0.0                    Title: "v1.0.0 — First Stable Release"
  → SHA of annotated tag object            Body: formatted release notes (Markdown)
  → SHA of commit                          Assets:
                                             - app-linux-amd64.tar.gz
                                             - app-darwin-arm64.zip
                                             - Source code (auto-generated by GitHub)
                                           Pre-release: false
                                           Latest: true
```

### Creating a GitHub Release from CLI

```bash
# Prerequisites: GitHub CLI installed (gh)
gh release create v1.0.0 \
  --title "v1.0.0 — First Stable Release" \
  --notes "## What's New..." \
  dist/app-linux-amd64.tar.gz \    # attach binary assets
  dist/app-darwin-arm64.zip

# From a file:
gh release create v1.0.0 --notes-file CHANGELOG.md

# Pre-release:
gh release create v1.1.0-beta.1 --prerelease --title "v1.1.0 beta 1"

# Draft (not published yet):
gh release create v1.1.0 --draft
```

### CI/CD triggered by tags

```yaml
# GitHub Actions: trigger deployment on any release tag
on:
  push:
    tags:
      - 'v*.*.*'      # matches v1.0.0, v2.3.4, etc.
      - '!v*-*'       # exclude pre-releases like v1.0.0-beta.1

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version from tag
        id: get_version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT
        # GITHUB_REF = refs/tags/v1.0.0
        # After stripping: VERSION=v1.0.0
      
      - name: Deploy
        run: ./scripts/deploy.sh ${{ steps.get_version.outputs.VERSION }}
```

---

## RULE 10: MENTAL MODEL CHECKPOINT

**1.** What is the physical difference between a lightweight tag and an annotated tag in
terms of what is written to `.git/`? How many new files and/or objects does each create?

**2.** `cat .git/refs/tags/v1.0.0` outputs a SHA. How do you determine whether this
tag is lightweight or annotated just from that SHA? What command tells you?

**3.** You run `git push origin main`. Are your tags pushed? Why or why not?
What are two different ways to push tags to origin?

**4.** A colleague says "I can't see the v2.0.0 tag — it's not in my `git tag` list."
They ran `git fetch` but their local `git tag` still doesn't show it. What's wrong
and what's the correct command?

**5.** Explain `git describe HEAD` output format. If HEAD is 3 commits after tag `v1.2.0`,
what does `git describe HEAD` output? What does `git describe HEAD --abbrev=0` output?

**6.** What does the `^` line in `packed-refs` mean? Example:
```
d4e5f6... refs/tags/v1.0.0
^a1b2c3...
```
What is `d4e5f6...` and what is `a1b2c3...`?

**7.** You're at `v1.3.2` and you need to deploy an urgent security patch. What is the
next correct SemVer version and what type of change does that signify?

**8.** What is the difference between `git push origin --tags` and
`git push origin --follow-tags`? Which is preferred for release workflows and why?

**9.** You have a signed tag (`-s`). How is the GPG signature stored — as a separate file,
as part of the tag object, or in a separate Git object? How do you verify it?

**10.** A GitHub Release and a Git tag — what is the relationship between them?
Can you have a GitHub Release without a Git tag? Can you have a Git tag without a GitHub Release?

---

## RULE 12: QUICK REFERENCE CARD

| Command | What It Does |
|---------|-------------|
| `git tag v1.0.0` | Create lightweight tag at HEAD |
| `git tag v1.0.0 <sha>` | Create lightweight tag at specific commit |
| `git tag -a v1.0.0 -m "msg"` | Create annotated tag at HEAD |
| `git tag -a v1.0.0 <sha> -m "msg"` | Create annotated tag at specific commit |
| `git tag -s v1.0.0 -m "msg"` | Create GPG-signed annotated tag |
| `git tag` | List all tags |
| `git tag -l "v1.*"` | List tags matching pattern |
| `git tag -l --sort=-version:refname` | List sorted by SemVer (newest first) |
| `git tag -n3` | List with first 3 lines of annotation |
| `git show v1.0.0` | Show tag details + tagged commit |
| `git tag -d v1.0.0` | Delete local tag |
| `git tag -f v1.0.0 <sha>` | Move tag to different commit (dangerous if pushed) |
| `git tag -v v1.0.0` | Verify GPG signature on signed tag |
| `git push origin v1.0.0` | Push a specific tag |
| `git push origin --tags` | Push ALL local tags to remote |
| `git push origin --follow-tags` | Push annotated tags reachable from pushed commits |
| `git push origin --delete v1.0.0` | Delete tag from remote |
| `git describe HEAD` | Show nearest tag + commits since + SHA |
| `git describe HEAD --abbrev=0` | Just the nearest tag name |
| `git describe HEAD --tags` | Include lightweight tags |
| `git describe HEAD --always` | Output SHA if no tag found |
| `git rev-parse v1.0.0^{}` | Get the commit SHA the tag points to |
| `git ls-remote origin --tags` | List all tags on remote |

| SemVer Rule | Meaning |
|-------------|---------|
| MAJOR bump | Breaking change — users must update code |
| MINOR bump | New features, backwards compatible |
| PATCH bump | Bug fixes, backwards compatible |
| `-alpha` | Early development, internal |
| `-beta` | Feature-complete, external testing |
| `-rc.N` | Release candidate — code freeze |

| Tag type | Best for |
|----------|---------|
| Lightweight | Local bookmarks, temporary, internal refs |
| Annotated | All releases — includes tagger, date, message |
| Signed | Releases requiring cryptographic authenticity proof |

---

## RULE 13: WHEN WOULD I USE THIS AT WORK?

### Scenario 1: Every production deploy is tagged

```
Company policy: "Every production deploy must have a corresponding Git tag.
The deploy system reads DEPLOY_VERSION from the tag name."

Your CI/CD pipeline:
  1. PR merged to main → CI runs tests
  2. On pass: CI creates annotated tag v{MAJOR}.{MINOR}.{PATCH}
  3. Tag push event triggers deployment pipeline
  4. Deploy script: VERSION=$(git describe --tags --exact-match HEAD)
  5. Kubernetes deployment manifest uses VERSION as image tag
  6. Rollback: git checkout v2.2.1 + re-deploy (one command)

Benefit: If prod is broken, you KNOW which commit is deployed,
and rolling back is: git push origin v2.2.1  → triggers deploy of that version.
```

### Scenario 2: Security audit requires signed tags

```
Compliance requirement: All production releases must be cryptographically signed
to prove the code was approved by an authorized person.

Workflow:
  1. Release engineer signs the release tag with their hardware security key:
     git tag -s v3.0.0 -m "Compliance-approved release v3.0.0"
  
  2. CI/CD pipeline verifies signature before deploying:
     git tag -v v3.0.0
     # Must show "Good signature from release-engineer@company.com"
     # If signature invalid: deployment refused
  
  3. Audit trail: every release has an irrefutable record of WHO approved it
     (the signature can only come from someone with the private key)
```

### Scenario 3: Library versioning with semantic-release

```
You maintain a shared internal Go library used by 12 services.
Every change must be versioned so downstream teams know what to update to.

Automated workflow with semantic-release:
  - Any PR with feat: → triggers v{N+1}.0.0 or v{N}.{M+1}.0
  - Any PR with fix:  → triggers v{N}.{M}.{P+1}
  - Any PR with BREAKING CHANGE: → triggers v{N+1}.0.0

Teams import your library:
  import "github.com/company/auth-lib v1.2.3"
  
When they see v2.0.0 in the changelog, they KNOW they have breaking changes to handle.
When they see v1.2.1, they know it's safe to update immediately.
```

### Scenario 4: Hotfix release workflow

```
v2.3.0 is in production.
v2.4.0 is in development (feature branch, half-done).
A critical security bug is found in v2.3.0.

Correct workflow (DO NOT patch v2.4.0 branch — it has half-finished work):

  1. git checkout -b hotfix/cve-2025-1234 v2.3.0
     # branches from the v2.3.0 TAG — not from current main

  2. Fix the bug
     git commit -am "fix: patch SQL injection in user search (CVE-2025-1234)"

  3. Tag the hotfix
     git tag -a v2.3.1 -m "Security: patch CVE-2025-1234"

  4. Merge back to main AND to the current development branch
     git switch main && git merge hotfix/cve-2025-1234
     git switch develop && git merge hotfix/cve-2025-1234

  5. Push everything
     git push origin main develop v2.3.1

  Result: v2.3.1 is deployed to production in minutes.
          v2.4.0 development continues unaffected.
          The fix is in both production AND the next release.
```

### Scenario 5: Reading git describe in build metadata

```bash
# Build script that embeds version into the binary:
VERSION=$(git describe --tags --long --always --dirty=-modified)
# Outputs like:
#   v2.3.0-0-gabcdef1             (exactly on a tag, clean working dir)
#   v2.3.0-3-g1234567             (3 commits after tag, clean)
#   v2.3.0-3-g1234567-modified    (3 commits after tag, uncommitted changes)
#   gabcdef1                      (no tag found, just SHA)

# Embed in Go binary:
go build -ldflags "-X main.Version=$VERSION" ./...

# The running binary can print:
./myapp --version
# v2.3.0-3-g1234567

# On-call engineer can immediately determine:
# - Which release it's based on (v2.3.0)
# - Whether it's an exact release or has extra commits (+3)
# - The exact commit SHA for git log investigation (g1234567)
```

---

## SUMMARY: The Mental Model in One Paragraph

A tag is a ref — just like a branch — but one that lives in `.git/refs/tags/` and is
never meant to move. A **lightweight tag** is identical to a branch in file format
(41-byte SHA + newline), pointing directly to a commit. An **annotated tag** adds one
indirection: the ref points to a **tag object** — Git's fourth object type — which stores
the tagger's identity, timestamp, and message before pointing to the underlying commit.
Annotated tags are preferred for releases because they carry metadata and are verifiable.
Tags are local by default and must be pushed explicitly with `git push origin <tagname>` or
`--tags`. The SemVer convention (MAJOR.MINOR.PATCH) gives tags meaning: MAJOR = breaking
change, MINOR = new features (backwards compatible), PATCH = bug fixes. The whole system
forms the foundation of professional release management: a tag is the immutable anchor
point that CI/CD systems use to build artifacts, deployment systems use to deploy specific
versions, and rollback procedures use to restore a known-good state.

---

*Topic 14 of 26 — Next: Topic 15: Git Workflows (Gitflow vs GitHub Flow vs Trunk-Based Development)*
