# Topic 10 — Remote Repositories
### `fetch`, `pull`, `push`, Tracking Branches, `origin` — Complete Internals

---

## Connection to Previous Topics

- **Topic 01** showed `.git/config` and briefly mentioned `[remote "origin"]`. Today you
  dissect that section completely.
- **Topic 03** introduced remote-tracking refs (`refs/remotes/origin/main`). Today they
  become fully real — you'll watch them get created, updated, and used.
- **Topic 08** showed that a branch is a 41-byte file. A remote-tracking branch is
  ALSO a 41-byte file — in a different directory (`refs/remotes/`), updated ONLY by fetch.
- **Topic 09** showed how merges work. `git pull` is literally `git fetch` + `git merge`
  (or `git fetch` + `git rebase`). Today you'll see both halves.

---

## RULE 1 — ELI5 Anchor

### The Library Reservation System Analogy

Imagine there's a **Central Library** in your city (the remote repository — GitHub,
GitLab, Bitbucket). You have a **personal copy** of a book at home (your local repo).

**`git fetch`** = You call the library and ask "what's new? what pages have been added?"
They send you a **catalogue update** — a list of all changes since your last call, plus
the actual new pages. But you're reading an OLD edition at home — the new pages sit in a
separate shelf labeled "Library's version" and don't interfere with your current reading.

**`git pull`** = Same phone call, PLUS you immediately replace your current edition with
the merged result. (Sometimes this causes confusion if you had handwritten notes on pages
that the library also updated.)

**`git push`** = You send your handwritten additions BACK to the library. The librarian
checks if your copy is still current (based on their latest). If your copy is behind,
they say "your additions might conflict with what we have — update your copy first."

**`origin`** = Just the DEFAULT NAME for "the main library" — the remote you cloned from.
You could call it anything. Teams with multiple remotes often have `origin` (your fork)
and `upstream` (the original project).

**Tracking branch** = Your bookmark that says "my `main` branch corresponds to the
library's `main` branch." When you check your local main, Git can tell you "you're 3
pages ahead of the library and 1 page behind."

---

## RULE 2 — Physical Architecture First

### What a Remote Is on Disk

A remote is **configuration** stored in `.git/config`. No new object types. No magic.

```
.git/config (after git remote add origin <url>):

[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true

[remote "origin"]
    url = https://github.com/yourname/yourrepo.git
    fetch = +refs/heads/*:refs/remotes/origin/*

[branch "main"]
    remote = origin
    merge = refs/heads/main
```

**Dissecting `[remote "origin"]`:**
```
[remote "origin"]
│         │
│         └─ The name "origin" — just a label. Could be "github", "upstream", etc.
│

    url = https://github.com/yourname/yourrepo.git
    │
    └─ Where to fetch from / push to. Can be:
         HTTPS: https://github.com/user/repo.git
         SSH:   git@github.com:user/repo.git
         File:  /path/to/local/bare/repo.git
         SSH with custom host: git@myserver:repo.git

    fetch = +refs/heads/*:refs/remotes/origin/*
    │   │   │               │
    │   │   │               └─ LOCAL destination: store as refs/remotes/origin/*
    │   │   └─ REMOTE source: all branches (refs/heads/*)
    │   └─ The colon separates source:destination
    └─ + means "force update even if not fast-forward" (for remote-tracking refs)
       This is called a REFSPEC.
```

**Dissecting `[branch "main"]`:**
```
[branch "main"]
│         │
│         └─ The local branch name "main"
│

    remote = origin
    │
    └─ When pushing/pulling "main", use the "origin" remote

    merge = refs/heads/main
    │
    └─ The REMOTE branch to merge/rebase against.
       This is the FULL ref name on the REMOTE (not locally).
       refs/heads/main on the remote maps to refs/remotes/origin/main locally.
```

---

### The Full `.git/` Structure with a Remote Configured

```
.git/
├── HEAD                           ← "ref: refs/heads/main\n"
├── config                         ← contains [remote "origin"] and [branch "main"]
├── FETCH_HEAD                     ← written by every git fetch (see below)
├── index                          ← local staged state
├── objects/
│   ├── (local objects)
│   └── pack/                      ← downloaded objects may be here
├── refs/
│   ├── heads/
│   │   ├── main                   ← LOCAL branch (41 bytes)
│   │   └── feature/auth           ← another local branch
│   ├── remotes/
│   │   └── origin/
│   │       ├── main               ← REMOTE-TRACKING branch (41 bytes)
│   │       ├── feature/auth       ← remote-tracking of remote's feature/auth
│   │       └── HEAD               ← which branch is default on remote
│   └── tags/
└── logs/
    └── refs/
        ├── heads/
        │   └── main               ← local reflog for main
        └── remotes/
            └── origin/
                └── main           ← reflog for remote-tracking main
```

```bash
# PROVE IT: See all remote-tracking refs
cat .git/refs/remotes/origin/main
# a4b5c6d7e8f9a0b1c2d3e4f5g6h7i8j9k0l1m2n3

ls .git/refs/remotes/origin/
# HEAD  feature/auth  main

# Read the remote default branch
cat .git/refs/remotes/origin/HEAD
# ref: refs/remotes/origin/main   ← symbolic ref pointing to default branch
```

---

## The Refspec — The Core Concept

A **refspec** defines the mapping between remote refs and local refs:

```
+refs/heads/*:refs/remotes/origin/*
│              │             │
│              │             └─ LOCAL destination pattern
│              └─ REMOTE source pattern (on the GitHub server)
└─ Force flag: update even if not fast-forward
```

**Reading it:** "For every branch on the remote (`refs/heads/*`), store a local copy
at `refs/remotes/origin/<same-name>`. Force-update even if the remote rewrote history."

**Custom refspecs:**
```
# Fetch only the main branch:
git fetch origin refs/heads/main:refs/remotes/origin/main

# Fetch a specific PR from GitHub:
git fetch origin refs/pull/42/head:refs/heads/pr-42

# Push local main to remote staging:
git push origin refs/heads/main:refs/heads/staging
# Short form: git push origin main:staging
```

```bash
# PROVE IT: See your repo's fetch refspecs
git remote show origin
# or
git config --get-all remote.origin.fetch
# +refs/heads/*:refs/remotes/origin/*
```

---

## RULE 3 — Exact Data Flow: `git fetch`

### What `git fetch` Actually Does

```
COMMAND: git fetch origin

STEP 1 — Read remote configuration
  READS:    .git/config → [remote "origin"]
  GETS:     url = https://github.com/user/repo.git
            fetch refspec = +refs/heads/*:refs/remotes/origin/*

STEP 2 — Establish connection
  CONNECTS: Opens TCP/HTTP or SSH connection to GitHub

STEP 3 — Exchange capability advertisement
  REMOTE SENDS: "I have these refs and their SHAs:
                  refs/heads/main → abc123
                  refs/heads/feature/auth → def456
                  refs/heads/hotfix → ghi789"

STEP 4 — Determine what to download
  COMPARES: Remote's refs vs. local .git/objects/
  FINDS:    "I'm missing: abc123, def456, and their associated tree/blob objects"

STEP 5 — Download missing objects
  DOWNLOADS: All missing blobs, trees, commits (as a packfile for efficiency)
  WRITES:    .git/objects/pack/pack-XXXX.pack
             .git/objects/pack/pack-XXXX.idx

STEP 6 — Update remote-tracking refs
  WRITES:   .git/refs/remotes/origin/main      → "abc123\n"  (was behind or new)
  WRITES:   .git/refs/remotes/origin/feature/auth → "def456\n"
  WRITES:   .git/refs/remotes/origin/hotfix    → "ghi789\n"

STEP 7 — Update FETCH_HEAD
  WRITES:   .git/FETCH_HEAD with one line per fetched branch:
              abc123    branch 'main' of https://github.com/user/repo.git
              def456  not-for-merge  branch 'feature/auth' of ...
              ghi789  not-for-merge  branch 'hotfix' of ...
            (The "not-for-merge" means it's not the branch that git pull will merge)

STEP 8 — Update reflog
  APPENDS:  .git/logs/refs/remotes/origin/main with the update record

DOES NOT MODIFY:
  - .git/HEAD             (you're still on the same branch)
  - .git/refs/heads/main  (your local main is UNCHANGED)
  - .git/index            (staged state unchanged)
  - Working directory     (your files unchanged)
```

**This is the CRITICAL insight about `git fetch`: it NEVER touches your working files or
your local branch pointers. It is a completely safe, read-only operation from your local
work's perspective. It only updates `refs/remotes/`.**

```bash
# PROVE IT: Run fetch and watch what changes
# Before:
cat .git/refs/remotes/origin/main     # some old SHA
cat .git/refs/heads/main              # your local SHA

git fetch origin

# After:
cat .git/refs/remotes/origin/main     # UPDATED to remote's current SHA
cat .git/refs/heads/main              # UNCHANGED — your local is the same

# See FETCH_HEAD:
cat .git/FETCH_HEAD
# abc123...   branch 'main' of https://github.com/...
```

---

### What FETCH_HEAD Contains

```
.git/FETCH_HEAD after git fetch origin:

abc123def456abc123def456abc123def456abc123d         branch 'main' of https://github.com/user/repo
def456abc123def456abc123def456abc123def456  not-for-merge   branch 'feature/auth' of https://...
ghi789def456ghi789def456ghi789def456ghi789  not-for-merge   branch 'hotfix' of https://...
```

**The format:**
```
<sha>  [not-for-merge]  <description>
│       │                │
│       │                └─ Human-readable source: "branch 'main' of <url>"
│       └─ "not-for-merge": this branch won't be used if you run git merge FETCH_HEAD
│           (only the default/requested branch is for-merge)
└─ Full SHA of the fetched branch tip
```

**`FETCH_HEAD` is used by:**
- `git pull` — it reads FETCH_HEAD to know what to merge after fetching
- `git merge FETCH_HEAD` — merge whatever was just fetched
- Scripts that need to know "what did the last fetch bring?"

---

## RULE 4 — ASCII Architecture: Local vs Remote State

```
LOCAL (.git/)                      REMOTE (GitHub/GitLab server)
─────────────                      ────────────────────────────

refs/heads/main    (C5)            refs/heads/main    (C7)
refs/heads/feat    (F3)            refs/heads/feat    (F3)
                                   refs/heads/hotfix  (H1)  ← you don't have this yet

refs/remotes/origin/main  (C4)     ← STALE: was C4 when you last fetched
refs/remotes/origin/feat  (F3)     ← CURRENT: same as remote

After git fetch origin:

refs/heads/main    (C5)            ← UNCHANGED (your local work)
refs/heads/feat    (F3)            ← UNCHANGED

refs/remotes/origin/main  (C7)     ← UPDATED to match remote
refs/remotes/origin/feat  (F3)     ← unchanged (same)
refs/remotes/origin/hotfix (H1)    ← NEW: learned about remote's hotfix branch

FETCH_HEAD: C7 is the "for-merge" branch (main was requested)
```

---

## Remote-Tracking Branches — The Full Picture

### What They Are

Remote-tracking branches are **read-only snapshots** of the remote's branches at the time
of the last `git fetch`. They are files in `.git/refs/remotes/<remote>/`.

```
Name format: <remote>/<branch>
  origin/main
  origin/feature/auth
  upstream/main

File location: .git/refs/remotes/origin/main

Rules:
  - NEVER commit to a remote-tracking branch directly
  - NEVER `git switch` to a remote-tracking branch directly
    (doing so detaches HEAD or creates a local tracking branch automatically)
  - Updated ONLY by git fetch (or git pull, which does fetch first)
  - Read-only from your local work perspective
```

```bash
# PROVE IT: You cannot commit to a remote-tracking branch
git switch origin/main
# warning: you are in 'detached HEAD' state.
# HEAD is now at abc123 [origin/main]

# Git does NOT let you "be on" origin/main as a branch
git branch --show-current
# (empty — detached HEAD)
```

### Creating a Local Tracking Branch from a Remote-Tracking Branch

```bash
# The standard way to start working on a remote branch:
git switch -c feature/auth origin/feature/auth
# Creates local feature/auth tracking origin/feature/auth

# Short form (Git >= 2.23):
git switch feature/auth
# If no local branch named "feature/auth" exists but origin/feature/auth does:
# Git AUTOMATICALLY creates local feature/auth tracking origin/feature/auth
# (git switch's --guess behavior)

# Legacy:
git checkout -b feature/auth origin/feature/auth
git checkout --track origin/feature/auth
```

**What "tracking" means in `.git/config`:**
```
[branch "feature/auth"]
    remote = origin
    merge = refs/heads/feature/auth
```

This config means:
- `git pull` on `feature/auth` → fetch from `origin`, merge `refs/heads/feature/auth`
- `git push` on `feature/auth` → push to `origin`, update `refs/heads/feature/auth`
- `git status` shows: "Your branch is ahead of 'origin/feature/auth' by 2 commits"
- `git branch -vv` shows the upstream tracking relation

---

## RULE 3 — Exact Data Flow: `git pull`

### `git pull` Is Two Commands

```
git pull origin main

STEP 1 — git fetch origin main
  (Everything from the git fetch flow above, but limited to 'main' branch)
  RESULT: .git/refs/remotes/origin/main updated, FETCH_HEAD updated, objects downloaded

STEP 2 — git merge origin/main  (or git rebase origin/main if pull.rebase = true)
  READS:    .git/refs/remotes/origin/main → "abc123"
  READS:    current branch tip
  PERFORMS: 3-way merge (or fast-forward if possible)
  RESULT:   New merge commit (or fast-forward) on current branch

FULL DATA FLOW TRACE (pull = fetch + merge):

  FETCH:
    Downloads new objects → .git/objects/
    Updates .git/refs/remotes/origin/main → new SHA
    Writes .git/FETCH_HEAD

  MERGE (fast-forward case):
    Updates .git/refs/heads/main → new SHA
    Updates .git/index to match new tree
    Updates working directory

  MERGE (3-way case):
    Creates merge commit
    Updates .git/refs/heads/main → merge commit SHA
    Updates index + working directory
    Writes .git/ORIG_HEAD (your pre-pull state, for undo)
```

```bash
# PROVE IT: Do a pull step by step manually
# Step 1: Fetch only
git fetch origin main
cat .git/FETCH_HEAD
# Shows the remote's main SHA

cat .git/refs/heads/main       # your LOCAL main (unchanged by fetch)
cat .git/refs/remotes/origin/main  # remote-tracking (updated)

# Step 2: See the divergence
git log --oneline main..origin/main
# Shows commits on origin/main that your local main doesn't have

# Step 3: Merge manually
git merge origin/main

# This is EXACTLY what git pull does
```

### `pull.rebase` Config (Rebase Instead of Merge)

```bash
git config pull.rebase true  # globally: git pull = fetch + rebase
git config pull.rebase false # globally: git pull = fetch + merge (default)

# Per pull:
git pull --rebase origin main     # this pull uses rebase
git pull --no-rebase origin main  # this pull uses merge

# The difference in history shape:
# pull --no-rebase:  C1──C2──C3──(remote commits)──M1  (merge commit)
# pull --rebase:     C1──C2──(remote commits)──C3' (your commit rebased on top)
```

---

## RULE 3 — Exact Data Flow: `git push`

### The Full Push Trace

```
COMMAND: git push origin main

STEP 1 — Read push destination
  READS:    .git/config → remote.origin.url
  READS:    .git/refs/heads/main → "def456\n" (your local commit)

STEP 2 — Establish connection
  CONNECTS: Opens connection to origin

STEP 3 — Exchange refs
  REMOTE SAYS: "My refs/heads/main is currently at abc123"
  LOCAL HAS:   refs/heads/main = def456

STEP 4 — Compute objects to send
  DETERMINES: Which objects are on the path def456 → abc123 but NOT on abc123's side?
  FINDS:      The new commits + their trees + their blobs that the remote doesn't have

STEP 5 — Safety check (non-force push)
  CHECKS: Is def456 a DESCENDANT of abc123?
          (Is abc123 in def456's ancestor chain?)
  IF YES: Fast-forward update — remote can advance from abc123 to def456 safely
  IF NO:  REJECT: "Updates were rejected because the remote contains work that
                   you do not have locally. Hint: run git pull before push."

STEP 6 — Upload objects
  SENDS:    All missing objects (as packfile) to remote
  REMOTE:   Writes objects to its .git/objects/

STEP 7 — Update remote ref
  REMOTE:   Updates its refs/heads/main → def456
  LOCAL:    Updates .git/refs/remotes/origin/main → def456
            (remote-tracking ref stays in sync after successful push)

DOES NOT MODIFY:
  - .git/refs/heads/main (unchanged — you already had this)
  - .git/index
  - Working directory
```

**The push safety check visualized:**

```
SAFE (fast-forward):
  Remote: A──B──C  (abc123 = C)
  Local:  A──B──C──D──E  (def456 = E)

  E is a descendant of C → push accepted, remote advances to E

REJECTED (diverged):
  Remote: A──B──C──X──Y  (abc123 = Y, someone pushed X and Y)
  Local:  A──B──C──D──E  (def456 = E)

  E is NOT a descendant of Y → push rejected
  You must pull (fetch + merge/rebase) first to integrate X and Y
```

```bash
# PROVE IT: Watch the push flow
# Before push:
cat .git/refs/heads/main            # local SHA
cat .git/refs/remotes/origin/main   # remote-tracking SHA (older)

git push origin main
# Enumerating objects: 5, done.
# Counting objects: 100% (5/5), done.
# Writing objects: 100% (3/3), 312 bytes | 312.00 KiB/s, done.
# Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
# To https://github.com/user/repo.git
#    abc123..def456  main -> main

# After push:
cat .git/refs/remotes/origin/main   # NOW matches local main
```

---

## RULE 4 — ASCII Diagram: Push/Fetch/Pull Complete Picture

```
LOCAL MACHINE                          GITHUB SERVER
─────────────                          ─────────────

.git/refs/heads/main = C5              .git/refs/heads/main = C3
.git/refs/remotes/origin/main = C3     (remote-tracking is in sync)

                ┌──── git fetch ────────────────────►
                │                                    │
                │  Remote says: "I have C4, C5 now"  │
                │  (Someone else pushed C4, C5)       │
                │◄──── sends C4, C5 objects ─────────┘
                │
                │  .git/refs/remotes/origin/main → C5   (UPDATED)
                │  .git/refs/heads/main → C3             (UNCHANGED)

       ┌─── git merge origin/main ───────────────────┐
       │                                             │
       │  Merges C5 into C3 → creates M1             │
       │  .git/refs/heads/main → M1                  │
       └─────────────────────────────────────────────┘

                ┌──── git push ─────────────────────►
                │                                    │
                │  "I have M1, you have C5, here are new objects"
                │  M1 is descendant of C5 → ACCEPTED  │
                │◄──── "ok, refs/heads/main = M1" ───┘
                │
                │  .git/refs/remotes/origin/main → M1  (UPDATED)
```

---

## `git push --force` and `--force-with-lease`

### When Force Push Is Needed

Force push is needed when you've REWRITTEN history locally (amend, rebase, reset) and
want to update the remote to match your rewritten history.

```bash
# Scenario: You rebased feature/auth onto latest main
git switch feature/auth
git rebase main
# Your commits are now C1'  C2'  C3' (new SHAs, same content, new parent)

git push origin feature/auth
# REJECTED: non-fast-forward
# (Remote has the OLD commits, your new ones are NOT descendants)

# Force push:
git push --force origin feature/auth
# Remote's refs/heads/feature/auth = your new SHA
# The old commits are orphaned on the remote (until GC)
```

### `--force` vs `--force-with-lease`

```
--force:
  "Overwrite whatever is on the remote. I don't care what's there."
  DANGER: If someone pushed to this branch between your fetch and your push,
          their commits are SILENTLY OVERWRITTEN AND LOST.

--force-with-lease:
  "Overwrite the remote ONLY IF its current SHA matches the remote-tracking SHA
   I have locally. If the remote moved since my last fetch, REFUSE."
  Safe version: fails if someone else pushed since you last fetched.
  
--force-with-lease=<refname>:<sha>:
  Even more explicit: "Only force-push if remote's <refname> is currently <sha>"
```

```bash
# The dangerous case where --force is wrong:
# You fetched at 10:00 → origin/feature/auth = C3
# Colleague pushed D4 to remote at 10:05
# You rebase and force-push at 10:10

git push --force origin feature/auth
# OVERWRITES D4! Colleague's work is lost!

# The safe version:
git push --force-with-lease origin feature/auth
# Git checks: is remote's feature/auth still C3?
# No, it's D4 (colleague pushed!) → REFUSED
# You see: "stale info; abort or fetch and re-push"
# You fetch first, incorporate D4, then rebase again and push

# ALWAYS use --force-with-lease instead of --force
```

---

## `git push -u` — Setting the Upstream

```bash
git push -u origin feature/auth
# Short for: --set-upstream

# This push:
# 1. Pushes the branch to origin (creates it on remote if needed)
# 2. Sets the tracking relationship in .git/config:
#      [branch "feature/auth"]
#          remote = origin
#          merge = refs/heads/feature/auth

# After -u, you can just use:
git push    # no need to specify remote/branch anymore
git pull    # same — knows where to pull from
git status  # shows "Your branch is ahead of 'origin/feature/auth' by 2 commits"
```

---

## RULE 6 — Deep Syntax Breakdown

### `git remote`

```
git remote  [-v]  [add | remove | rename | set-url | get-url | prune | show]
│    │        │
│    │        └─ -v / --verbose: show URLs next to each remote name
│    └─ subcommand: manage remote connections
└─ git binary

SUBCOMMANDS:
  git remote add <name> <url>
    Creates [remote "<name>"] section in .git/config
    Example: git remote add origin https://github.com/user/repo.git

  git remote -v
    Lists all remotes with their fetch and push URLs
    Example output:
      origin  https://github.com/user/repo.git (fetch)
      origin  https://github.com/user/repo.git (push)
      upstream https://github.com/original/repo.git (fetch)
      upstream https://github.com/original/repo.git (push)

  git remote remove <name>
    Removes [remote "<name>"] from config
    Also removes all refs/remotes/<name>/* refs

  git remote rename <old> <new>
    Renames the remote and updates all tracking configs that reference it

  git remote set-url <name> <new-url>
    Change the URL of an existing remote (e.g., switch from HTTPS to SSH)

  git remote get-url <name>
    Print the URL for a remote

  git remote prune <name>
    Delete local remote-tracking refs for branches deleted on the remote
    Example: git remote prune origin
    (Same as: git fetch --prune origin)

  git remote show <name>
    Detailed info: URL, tracking branches, pull/push behavior
    Example: git remote show origin
    Output:
      * remote origin
        Fetch URL: https://github.com/user/repo.git
        Push  URL: https://github.com/user/repo.git
        HEAD branch: main
        Remote branches:
          feature/auth tracked
          main         tracked
          hotfix       stale (use 'git remote prune' to remove)
        Local branch configured for 'git pull':
          main merges with remote main
        Local ref configured for 'git push':
          main pushes to main (up to date)
```

### `git fetch`

```
git fetch  [<remote>]  [<refspec>]
│    │       │           │
│    │       │           └─ Specific branch(es) to fetch
│    │       │              Default: fetch all branches per remote's fetch config
│    │       └─ Which remote to fetch from
│    │          Default: origin (if only one remote) or tracked remote
│    └─ subcommand: download objects and update remote-tracking refs
└─ git binary

FLAGS:
  --all              Fetch from ALL remotes (not just origin)
  --prune / -p       Delete remote-tracking refs for remote-deleted branches
                     Example: if origin deleted feature/auth, your
                     origin/feature/auth ref is also removed
  --prune-tags       Also prune local tags no longer on remote
  --tags             Fetch all tags (even not pointed to by fetched branches)
  --no-tags          Don't fetch tags
  --depth=N          Shallow fetch: only N commits deep (for large repos)
  --unshallow        Convert shallow clone to full clone
  --verbose / -v     Show more detail about what's downloaded
  --dry-run          Show what WOULD be fetched without actually fetching
  --force / -f       Update remote-tracking refs even if not fast-forward
  -j / --jobs=N      Fetch multiple remotes in parallel (with --all)
```

### `git pull`

```
git pull  [<remote>]  [<branch>]  [--rebase]  [--no-rebase]
│    │      │           │           │
│    │      │           │           └─ Rebase local commits on top of fetched ones
│    │      │           │              Instead of merge (creates cleaner history)
│    │      │           └─ Which remote branch to pull
│    │      │              Default: tracked branch from .git/config
│    │      └─ Which remote to pull from
│    │         Default: tracked remote from .git/config
│    └─ subcommand: fetch + integrate (merge or rebase)
└─ git binary

KEY FLAGS:
  --rebase                Rebase instead of merge
  --no-rebase             Merge (override pull.rebase config)
  --ff-only               Only allow fast-forward (abort if merge commit needed)
  --no-ff                 Force merge commit even if fast-forward possible
  --autostash             Stash local changes before pull, unstash after
  --allow-unrelated-histories  Merge even if repos have no common ancestor
  --depth=N               Shallow fetch + update
  --no-commit             Fetch + merge but don't auto-commit result
  --squash                Fetch + squash into staged changes
  -v / --verbose          Show fetch details
  -4 / -6                 Force IPv4 or IPv6
```

### `git push`

```
git push  [<remote>]  [<refspec>]
│    │      │           │
│    │      │           └─ What to push and where:
│    │      │              <local-branch>:<remote-branch>  e.g. main:main
│    │      │              <local-branch>  (same name on remote)
│    │      │              :<remote-branch>  (delete the remote branch)
│    │      │              Default: push tracked branch to tracked remote
│    │      └─ Remote name. Default: origin
│    └─ subcommand: upload objects and update remote refs
└─ git binary

KEY FLAGS:
  -u / --set-upstream     Set tracking after push
  --all                   Push ALL local branches
  --tags                  Push all tags
  --follow-tags           Push all tags that annotate commits being pushed
  --force / -f            Force push (overwrite remote history)
  --force-with-lease      Safe force push (fail if remote changed since last fetch)
  --force-with-lease=<ref>:<sha>  Extra-specific safe force push
  --dry-run / -n          Show what WOULD be pushed without pushing
  --delete / -d           Delete a remote branch:
                           git push origin --delete feature/auth
                           or: git push origin :feature/auth
  --no-verify             Skip pre-push hook
  --mirror                Mirror all refs (ALL branches, tags, remote-tracking)
                           Dangerous: used for repo migrations
  --atomic                All updates succeed or all fail (server support required)
  --push-option=<str>     Send custom data to server hooks
  -v / --verbose          Show more detail
  -q / --quiet            Suppress output
  --prune                 Delete remote branches that don't have local counterparts
```

---

## Deep Dive: `git clone` — What It Sets Up

When you clone a repo, Git does all of this in one step:

```bash
git clone https://github.com/user/repo.git mydir

# Equivalent to:
mkdir mydir && cd mydir
git init
git remote add origin https://github.com/user/repo.git
git fetch origin
git checkout -b main --track origin/main
# (also sets up remote-tracking refs for all remote branches)
```

**The `.git/config` after clone:**
```
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true

[remote "origin"]
    url = https://github.com/user/repo.git
    fetch = +refs/heads/*:refs/remotes/origin/*

[branch "main"]
    remote = origin
    merge = refs/heads/main
```

**What `git clone` creates:**
```
.git/refs/heads/main                ← local branch pointing to latest commit
.git/refs/remotes/origin/main       ← remote-tracking (same SHA initially)
.git/refs/remotes/origin/HEAD       ← which branch is default on remote
.git/refs/remotes/origin/<all-remote-branches>
```

### Clone Flags Worth Knowing

```bash
git clone <url>                     # clone into dirname of url
git clone <url> mydir               # clone into mydir/
git clone <url> .                   # clone into current directory (must be empty)
git clone --depth=1 <url>           # shallow clone: only latest commit (fast!)
git clone --depth=1 --branch=v2.0 <url>  # shallow clone of specific tag
git clone --single-branch <url>     # only clone default branch (saves bandwidth)
git clone --mirror <url>            # bare clone of ALL refs (for backup/migration)
git clone --bare <url>              # bare clone (no working directory)
git clone -b feature/auth <url>     # start on specific branch
git clone --recurse-submodules <url>  # also clone submodules (Topic 23)
git clone --filter=blob:none <url>  # partial clone: no blob content (download on demand)
```

---

## `git ls-remote` — Inspect Remote Without Cloning

```bash
git ls-remote origin
# Lists ALL refs on the remote server (without downloading objects):
# abc123  HEAD
# abc123  refs/heads/main
# def456  refs/heads/feature/auth
# v1.0sha refs/tags/v1.0
# ^v1.0t  refs/tags/v1.0^{}   ← peeled tag (commit SHA)

# Fetch specific GitHub PR:
git fetch origin refs/pull/42/head:refs/heads/pr-42
# After this: local branch pr-42 contains the PR's code
```

---

## Multiple Remotes — `origin` vs `upstream`

### The Fork Workflow

```
ORIGINAL REPO:  https://github.com/facebook/react   ← "upstream"
YOUR FORK:      https://github.com/yourname/react    ← "origin"
YOUR LOCAL:     .git/ on your machine
```

```bash
# Setup:
git clone https://github.com/yourname/react.git
# (origin = your fork, automatically configured)

git remote add upstream https://github.com/facebook/react.git
# (upstream = the original project)

git remote -v
# origin    https://github.com/yourname/react.git (fetch)
# origin    https://github.com/yourname/react.git (push)
# upstream  https://github.com/facebook/react.git (fetch)
# upstream  https://github.com/facebook/react.git (push)
```

**The workflow:**
```bash
# Pull latest from upstream (the original project)
git fetch upstream
git switch main
git merge upstream/main     # or: git rebase upstream/main

# Push to YOUR fork (not upstream — you usually don't have write access)
git push origin main

# Create feature branch, work, push to YOUR fork:
git switch -c feature/my-change
# ... work ...
git push origin feature/my-change
# Open Pull Request from yourname/react to facebook/react
```

**Why not push directly to upstream?**
- Open source projects restrict who can push
- The PR review process is the gate for code quality
- `upstream` remote is often set as fetch-only:
  ```bash
  git remote set-url --push upstream NO_PUSH
  # Makes `git push upstream` fail with helpful error
  ```

---

## `git fetch --prune` — Cleaning Up Stale Tracking Refs

### The Problem

Your colleague deletes a remote branch (`feature/old-work`) on GitHub after merging.
Your local repo still has `origin/feature/old-work` in `refs/remotes/origin/`. It's
now a stale pointer to an orphaned commit on the remote.

```bash
# See stale remote-tracking branches
git remote show origin
# Remote branches:
#   feature/active      tracked
#   main               tracked
#   feature/old-work   stale (use 'git remote prune' to remove)

# Clean up automatically during fetch:
git fetch --prune origin
# From https://github.com/user/repo
#  - [deleted]         (none)     -> origin/feature/old-work

# Or configure Git to always prune on fetch:
git config --global fetch.prune true
# Now every git fetch automatically prunes stale refs
```

---

## RULE 5 — Hands-On Proof: The Full Remote Workflow

### Setup: Create a Local "Remote" for Testing

You don't need GitHub to practice remotes — you can use a local bare repo:

```bash
# Create a bare repo (the "remote" server)
mkdir /tmp/remote-repo.git
cd /tmp/remote-repo.git
git init --bare
# Bare repo: no working directory, only .git/ contents

# Create local working repo
mkdir ~/work/my-project && cd ~/work/my-project
git init
git remote add origin /tmp/remote-repo.git

# Verify remote is configured
cat .git/config
# [remote "origin"]
#     url = /tmp/remote-repo.git
#     fetch = +refs/heads/*:refs/remotes/origin/*

# First commit + push
echo "# Project" > README.md
git add README.md
git commit -m "chore: initial commit"
git push -u origin main
# (sets up tracking)

# Verify
cat .git/refs/remotes/origin/main    # same SHA as local main
cat .git/config | grep -A3 'branch "main"'
# [branch "main"]
#     remote = origin
#     merge = refs/heads/main

# Now simulate a colleague: clone the same "remote"
mkdir ~/work/colleague && cd ~/work/colleague
git clone /tmp/remote-repo.git .

# Colleague makes a commit
echo "// Colleague's change" >> README.md
git add README.md
git commit -m "feat: colleague adds to README"
git push origin main

# Back to your repo: your origin/main is now BEHIND
cd ~/work/my-project
cat .git/refs/heads/main              # your SHA (older)
cat .git/refs/remotes/origin/main     # stale (your last fetch)

git fetch origin
cat .git/refs/remotes/origin/main     # NOW updated (colleague's push)
cat .git/refs/heads/main              # STILL your SHA (unchanged by fetch)

# Merge the colleague's work
git merge origin/main
cat .git/refs/heads/main              # NOW updated (fast-forward or merge commit)
```

---

## RULE 7 — Beginner Example

### First Time Pushing to GitHub

```bash
# You have a local project with commits
git log --oneline
# abc123 feat: add user authentication
# def456 chore: initial project setup

# You created an empty repo on GitHub at:
# https://github.com/yourname/my-project.git

# Connect local to remote
git remote add origin https://github.com/yourname/my-project.git

# Verify
git remote -v
# origin  https://github.com/yourname/my-project.git (fetch)
# origin  https://github.com/yourname/my-project.git (push)

# Push local main to remote
git push -u origin main
# Enumerating objects: 6, done.
# Writing objects: 100%
# To https://github.com/yourname/my-project.git
#  * [new branch]      main -> main
# Branch 'main' set up to track remote branch 'main' from 'origin'.

# Now check what was created:
cat .git/refs/remotes/origin/main
# abc123... (same as your local main)

cat .git/config
# [branch "main"]
#     remote = origin
#     merge = refs/heads/main   ← -u set this up

# Future pushes: just git push (no remote/branch needed)
git push
```

---

## RULE 7 — Production Example

### Managing Multiple Remotes in a Fork-Based OSS Contribution

You want to contribute to `expressjs/express`. You've already forked it.

```bash
# Clone YOUR fork
git clone https://github.com/yourname/express.git
cd express

# Add the ORIGINAL project as "upstream"
git remote add upstream https://github.com/expressjs/express.git

# Verify
git remote -v
# origin    https://github.com/yourname/express.git (fetch/push)
# upstream  https://github.com/expressjs/express.git (fetch/push)

# Keep your fork's main in sync with upstream:
git fetch upstream
git switch main
git rebase upstream/main   # or: git merge upstream/main
git push origin main       # push updated main to your fork

# Create a feature branch for your contribution
git switch -c fix/memory-leak-in-router

# ... code, test, commit ...
git commit -m "fix(router): prevent memory leak when routes are added dynamically

Routes added after server.listen() were not being garbage collected.
Added cleanup logic in Router.prototype._dispose().

Fixes: expressjs/express#5142"

# Push to YOUR fork (not upstream)
git push -u origin fix/memory-leak-in-router

# Open PR on GitHub: yourname/express → expressjs/express

# Maintainer requests changes: add a test
vim test/router.test.js
git add test/router.test.js
git commit --amend --no-edit    # add test to existing commit (clean PR history)
git push --force-with-lease origin fix/memory-leak-in-router
```

---

## RULE 8 — Mistakes, Root Causes, and Fixes

### Mistake 1: Push Rejected ("non-fast-forward")

**Symptom:**
```
! [rejected]        main -> main (non-fast-forward)
error: failed to push some refs to 'origin'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. Integrate the remote changes before pushing.
```

**Root cause:** Someone else pushed commits to the remote after your last fetch.
Your local main is not a descendant of remote's main.

**Fix:**
```bash
# Option A: Fetch and merge (preserves both histories)
git fetch origin
git merge origin/main  # or: git pull origin main

# Option B: Fetch and rebase (cleaner linear history)
git fetch origin
git rebase origin/main  # or: git pull --rebase origin main

# Option C: If this is YOUR private branch and you rewrote history intentionally:
git push --force-with-lease origin main
# (Only if no one else has this branch)
```

---

### Mistake 2: Pushed to `main` Directly (Should Have Used a Branch)

**Symptom:** Your commit went to `origin/main` but it should have been on a feature branch
for PR review. The team has branch protection, but somehow the push went through.

**Fix (if no branch protection and you caught it fast):**
```bash
# Create a branch at the current commit
git branch feature/my-work HEAD

# Reset main to one commit before
git reset --hard HEAD~1

# Force push to restore origin/main
git push --force-with-lease origin main

# Push the feature branch
git push origin feature/my-work
# Open PR normally
```

**Prevention:** Enable branch protection on `main` (Topic 24 covers GitHub branch rules).

---

### Mistake 3: Deleted a Remote Branch and Can't Recover

**Symptom:**
```bash
git push origin --delete feature/important-work
# Deleted branch 'feature/important-work' on remote
# Oops
```

**Recovery:** If you or a colleague had the branch checked out or fetched recently:

```bash
# Do you still have the commits locally?
git branch -a | grep feature/important-work
# If yes: just push again
git push origin feature/important-work

# If the remote-tracking ref is gone but the commits exist:
git reflog | grep important-work
# Find the last SHA
git push origin <sha>:refs/heads/feature/important-work
```

If no one has the commits: GitHub keeps deleted branches recoverable for ~30 days
via the "Recently deleted branches" button in the repository's branch list.

---

### Mistake 4: Wrong URL for Remote (HTTPS vs SSH)

**Symptom:**
```bash
git push origin main
# git@github.com: Permission denied (publickey).
# OR
# fatal: Authentication failed for 'https://github.com/user/repo.git'
```

**Check current URL:**
```bash
git remote get-url origin
# https://github.com/user/repo.git   ← HTTPS
# OR
# git@github.com:user/repo.git       ← SSH
```

**Switch HTTPS ↔ SSH:**
```bash
# Switch to SSH (recommended — no repeated authentication):
git remote set-url origin git@github.com:user/repo.git

# Switch back to HTTPS:
git remote set-url origin https://github.com/user/repo.git
```

---

### Mistake 5: `git pull` Created an Ugly Merge Commit

**Symptom:**
```
* Merge branch 'main' of https://github.com/user/repo.git
|\
| * colleague's commit
* | my commit
|/
```

This happens when: you have local commits AND the remote has new commits. Default `git pull`
does a merge, creating a noisy merge commit.

**Prevention:**
```bash
git config --global pull.rebase true
# Now git pull always rebases → linear history

# Or per-pull:
git pull --rebase origin main
```

**Fix the existing ugly merge:**
```bash
# Reset to before the merge (ORIG_HEAD is set by git pull)
git reset --hard ORIG_HEAD

# Re-pull with rebase
git pull --rebase origin main
# Now history is linear: my commit rebased on top of colleague's
```

---

### Mistake 6: Force-Pushed Over a Colleague's Work

**Symptom:** Colleague complains that their commits disappeared from the remote branch.

**Root cause:** You ran `git push --force` (not `--force-with-lease`) after they pushed.
Their commits were silently overwritten.

**Recovery:**
```bash
# Colleague can find their commits via reflog (locally):
git reflog
# or via git log before they pull:
git log HEAD..origin/branch  # if they haven't pulled yet

# They create a branch at their last commit:
git branch rescue/colleagues-work <their-sha>

# You fetch their rescue branch, cherry-pick their commits onto yours, force-push again
git fetch origin rescue/colleagues-work
git cherry-pick <their-commits>
git push --force-with-lease origin branch
```

**Prevention:** ALWAYS use `--force-with-lease`. Never use bare `--force` except in
extreme emergency situations where you understand all consequences.

---

## RULE 10 — Mental Model Checkpoint

**1.** What is a remote? Where is it stored in `.git/`? What two pieces of information
does the `[remote "origin"]` config section store?

**2.** What is a refspec? Write out the default fetch refspec for origin and explain
each part: the `+`, the source pattern, the colon, and the destination pattern.

**3.** You run `git fetch origin`. List EVERY file in `.git/` that changes as a result.
List EVERY file that does NOT change. What is the key guarantee `git fetch` provides?

**4.** What is `.git/FETCH_HEAD`? When is it written? What does "not-for-merge" mean in
its content?

**5.** A remote-tracking branch like `origin/main` is stored at
`.git/refs/remotes/origin/main`. Can you commit to it directly? Can you `git switch` to it?
What IS it, mechanically?

**6.** `git pull` is two commands internally. Name them. What is the difference in
resulting history shape between `git pull --rebase` and `git pull --no-rebase`?

**7.** You ran `git push` and got "rejected: non-fast-forward." Explain exactly WHY
Git rejected it (what check failed) and name two ways to fix it.

**8.** What is the difference between `git push --force` and `git push --force-with-lease`?
Describe the exact scenario where `--force` silently destroys a colleague's work and
`--force-with-lease` would have prevented it.

**9.** What does `git push -u origin feature/auth` do beyond a regular push? What two
fields does it write in `.git/config`?

**10.** Your team has `origin` (your fork) and `upstream` (the original project). You
never push to `upstream`. How do you keep your fork's `main` in sync with `upstream/main`?
Write the exact sequence of commands.

---

## RULE 11 — Connections to Previous Topics

**From Topic 01** — `.git/` directory:
- Today you saw `refs/remotes/` become real — populated with remote-tracking refs
- `.git/FETCH_HEAD` first appeared in Topic 01; today you understand its exact format
- `.git/config`'s `[remote]` and `[branch]` sections are now fully dissected

**From Topic 03** — The DAG and refs:
- Remote-tracking refs (`refs/remotes/origin/main`) are refs just like branch refs —
  they're 41-byte files pointing into the shared DAG
- The push safety check is a DAG ancestry check: "is my new tip a descendant of yours?"

**From Topic 08** — Branching:
- Remote-tracking branches are NOT branches you work on — they're snapshots of the remote
- Creating a local tracking branch from a remote-tracking one is exactly `git switch -c
  feature/auth origin/feature/auth`

**From Topic 09** — Merging:
- `git pull` = `git fetch` + `git merge origin/main`
- The merge commit from a pull is a regular 2-parent merge commit
- `git pull --rebase` skips the merge entirely (Topic 11 covers rebase deeply)

---

## RULE 13 — When Would I Use This at Work?

### Scenario A: Daily Sync Ritual

Every morning before starting work:
```bash
git fetch --all --prune     # Download everything from all remotes, clean stale refs
git log --oneline main..origin/main   # What did teammates push to main overnight?
git switch feature/my-work
git rebase origin/main      # Bring my feature up to date with latest main
```

This avoids the "big merge surprise" when your PR is finally ready.

---

### Scenario B: CI/CD Push Guards

In many teams, `git push` to `main` is blocked by CI:

```yaml
# .github/workflows/push-guard.yml
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test
      - run: npm run build
```

If CI fails, the branch protection rule prevents the PR from merging. Understanding
`git push` lets you know WHY the push to `main` is being rejected by the remote
(the remote is enforcing rules, not just Git itself).

---

### Scenario C: Keeping a Release Branch Updated

```bash
git remote -v
# origin   https://github.com/yourco/api.git (fetch)

# release/2.x branch is maintained for LTS support
# It receives cherry-picked bug fixes from main

git fetch origin
git switch release/2.x
git cherry-pick <bugfix-sha-from-main>
git push origin release/2.x
```

Understanding that `git fetch` safely downloads without affecting your local work
is crucial here — you can fetch to inspect what's on main without disrupting your
release/2.x checkout.

---

### Scenario D: The Accidental Push of Secrets

You accidentally committed `.env` containing `DB_PASSWORD=supersecret` and pushed to GitHub.

**Immediate action:**
1. IMMEDIATELY rotate `DB_PASSWORD` — treat it as compromised
2. Remove from history (Topic 25 — `git filter-repo`)
3. Force push cleaned history
4. Tell GitHub to clear their cache (`git push --mirror` to a new repo)
5. Alert your security team

**Understanding the remote helps:**
```bash
# Verify what's currently on the remote
git ls-remote origin
# See what refs exist

# After cleaning history locally:
git push --force-with-lease origin main
# The remote now has cleaned history

# Verify remote no longer has the bad commit:
git fetch origin
git log --all --oneline | grep "bad commit message"
# (should be gone)
```

---

## RULE 12 — Quick Reference Card

### Remote Management

| Command | What it does |
|---------|-------------|
| `git remote add origin <url>` | Add a remote named origin |
| `git remote -v` | List remotes with URLs |
| `git remote remove origin` | Remove a remote |
| `git remote rename origin github` | Rename a remote |
| `git remote set-url origin <url>` | Change remote URL |
| `git remote prune origin` | Delete stale remote-tracking refs |
| `git remote show origin` | Detailed remote info |

### Fetch

| Command | What it does |
|---------|-------------|
| `git fetch origin` | Fetch all branches from origin |
| `git fetch origin main` | Fetch only main from origin |
| `git fetch --all` | Fetch from all configured remotes |
| `git fetch --prune` | Fetch + delete stale tracking refs |
| `git fetch --tags` | Also fetch all tags |
| `git ls-remote origin` | List remote refs without fetching |

### Pull

| Command | What it does |
|---------|-------------|
| `git pull` | Fetch tracked remote + merge |
| `git pull origin main` | Fetch + merge specific branch |
| `git pull --rebase` | Fetch + rebase (linear history) |
| `git pull --ff-only` | Only if fast-forward possible |
| `git pull --no-commit` | Fetch + merge but don't auto-commit |

### Push

| Command | What it does |
|---------|-------------|
| `git push origin main` | Push local main to origin |
| `git push -u origin main` | Push + set tracking |
| `git push` | Push to tracked remote (after -u) |
| `git push --force-with-lease` | Safe force push |
| `git push --force` | Unsafe force push (avoid!) |
| `git push origin --delete branch` | Delete remote branch |
| `git push origin :branch` | Delete remote branch (alternative) |
| `git push --tags` | Push all tags |
| `git push --follow-tags` | Push + annotated tags that annotate pushed commits |
| `git push --dry-run` | Show what WOULD be pushed |

### Tracking Branches

| Command | What it does |
|---------|-------------|
| `git branch -vv` | Show tracking info for all branches |
| `git branch -u origin/main` | Set upstream for current branch |
| `git branch --unset-upstream` | Remove upstream tracking |
| `git switch feature/auth` | Auto-create tracking branch if remote exists |
| `git switch -c feat origin/feat` | Explicitly create tracking branch |

### `.git/` Files for Remotes

| File | Contents |
|------|----------|
| `.git/config` | `[remote "origin"]` and `[branch "main"]` sections |
| `.git/FETCH_HEAD` | Result of last fetch (one line per fetched branch) |
| `.git/refs/remotes/origin/main` | Remote-tracking branch (41 bytes) |
| `.git/refs/remotes/origin/HEAD` | Default branch on remote |

---

## Summary

A remote is just a named URL in `.git/config`. A remote-tracking branch is just a
41-byte file in `.git/refs/remotes/`. There is no magic.

The three core operations:
- **`git fetch`** = safely downloads objects and updates `refs/remotes/` only.
  Your local branches and working directory are never touched.
- **`git pull`** = `git fetch` + `git merge` (or `git fetch` + `git rebase`).
  It's a convenience wrapper for the two-step process.
- **`git push`** = uploads objects and requests the remote to advance its branch pointer.
  Rejected if your new tip is not a descendant of the remote's current tip (non-fast-forward).

The golden rules:
- **Always `git fetch --prune`** instead of bare `git fetch` — keeps your remote-tracking
  refs in sync with reality
- **Always `--force-with-lease`** instead of `--force` — never silently destroy teammates' work
- **`git pull --rebase`** keeps history linear and readable — much better than the default
  merge behavior for most team workflows

*Last Updated: Topic 10 — Remote Repositories — ✅ Completed.*
