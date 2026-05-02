# Git & GitHub — Deep Mastery Curriculum
### From Zero → Principal Engineer Level (No Shortcuts, No Surface-Level Fluff)

---

## PHILOSOPHY

This is NOT a quick reference or cheat sheet. This is a **deep-dive engineering education** 
where you will understand Git the way its creators understand it — from the bytes on disk 
up to the team workflows built on top. Every topic is written so that after reading it, 
you could **explain it to someone else from memory, draw diagrams on a whiteboard, 
and debug any situation** related to that topic.

---

## TEACHING RULES (Applied EVERY Topic — Non-Negotiable)

### RULE 1: ELI5 Anchor
Start with a real-world analogy a 10-year-old could follow. This becomes the mental 
anchor that everything else hangs on.

### RULE 2: Physical Architecture First
Before explaining ANY concept, show what it looks like **on disk**. Show the actual 
folder structure, actual file contents, actual bytes. The reader should be able to 
open their file explorer, navigate to `.git/`, and SEE exactly what you're describing.

Format: Always include a full directory tree like:
```
.git/
├── HEAD              ← what this contains and WHY
├── config            ← what this contains and WHY
├── objects/          ← what lives here
│   ├── pack/         
│   ├── info/
│   └── ab/           ← first 2 chars of SHA
│       └── cdef1234...  ← remaining 38 chars (actual object file)
├── refs/
│   ├── heads/        ← branch pointers (one file per branch)
│   └── tags/         ← tag pointers
└── ...
```

### RULE 3: Show the EXACT Data Flow
For every git command, show a **step-by-step trace** of:
- What files Git reads
- What computation Git does
- What files Git writes/modifies
- The before-and-after state of `.git/`

Use this format:
```
COMMAND: git add server.js

READS:    server.js (working directory)
COMPUTES: SHA-1("blob " + byte_size + "\0" + file_contents) → "a3f2c1d8..."
WRITES:   .git/objects/a3/f2c1d8... (zlib-compressed blob)
MODIFIES: .git/index (adds entry: "100644 a3f2c1d8... 0 server.js")
```

### RULE 4: ASCII Architecture Diagrams
Every major concept gets a diagram showing relationships. Not just boxes — show 
the POINTERS (what points to what, stored where on disk, how you'd verify it).

### RULE 5: Hands-On Proof Commands
After every explanation, give the reader terminal commands they can run RIGHT NOW 
to prove to themselves that what you said is true. Format:

```bash
# PROVE IT: See the blob we just created
git cat-file -p a3f2c1d8
# Expected output: (exact content of server.js)
```

### RULE 6: Syntax Breakdown (Deep)
For every command, show:
```
git command <required-arg> [optional-arg] --flag=value
│   │        │              │               │
│   │        │              │               └─ what this flag does, 
│   │        │              │                  what changes without it
│   │        │              └─ when you'd use this, what happens if omitted
│   │        └─ what Git expects here, what happens with wrong input
│   └─ the subcommand (what category of operation)
└─ calling the git binary
```

### RULE 7: Beginner Example → Production Example
- **Beginner**: Isolated, 2-3 files, demonstrates one concept
- **Production**: Real team scenario with context like "You're on a 6-person 
  backend team, the PR has 12 commits, your team lead says..."

### RULE 8: Mistakes → Root Cause → Fix → Prevention
Not just "don't do X" — explain WHY it breaks (at the architecture level), 
show the broken state, show how to diagnose, show how to fix.

### RULE 9: "What Actually Happens" (Byte-Level When Relevant)
This section goes DEEP. Show the actual object format:
```
commit 234\0tree d8329f...\nparent 7c3e8f...\nauthor ...\n\nmessage
│          │                                           
│          └─ null byte separator                      
└─ header: type + space + content-length              
```

### RULE 10: Mental Model Checkpoint
End each topic with 5-7 "Can you answer these?" questions. If the reader can't 
answer them from memory, they need to re-read.

### RULE 11: Connect Everything Backwards
Every new topic explicitly references how it connects to previous topics. 
"Remember from Topic 01 that a branch is a 41-byte pointer file? That's 
why [this topic's concept] works the way it does."

### RULE 12: Quick Reference Card
Cheat sheet table at the end with ALL commands from that topic + one-line descriptions.

### RULE 13: "When Would I Use This At Work?"
Real scenarios from backend development: CI/CD pipelines, team code reviews, 
production hotfixes, monorepo management, release processes.

---

## PROGRESS TRACKER

| #  | Topic                                                                  | Status      | File |
|----|------------------------------------------------------------------------|-------------|------|
| 01 | The .git Directory — Complete Physical Architecture                    | ✅ Completed  | [topic-01](./topic-01-the-git-directory.md) |
| 02 | Git Objects Deep Dive — Blobs, Trees, Commits, Tags on Disk           | ✅ Completed  | [topic-02](./topic-02-git-objects-deep-dive.md) |
| 03 | The DAG & Refs — How History is Built and Navigated                    | ⬜ Not Started | — |
| 04 | Git Setup & Configuration (.gitconfig, global/local/system, aliases)   | ⬜ Not Started | — |
| 05 | The 3 Trees of Git (Working Directory, Index, HEAD) — Data Flow       | ⬜ Not Started | — |
| 06 | git add & git commit — Exact Internal Mechanics                        | ⬜ Not Started | — |
| 07 | Commits Done Right (atomic commits, Conventional Commits, amend)       | ⬜ Not Started | — |
| 08 | Branching — Internals, Creation, Switching, Deletion                   | ⬜ Not Started | — |
| 09 | Merging — Fast-Forward, 3-Way, Recursive, What Happens in .git        | ⬜ Not Started | — |
| 10 | Remote Repositories (fetch, pull, push, tracking branches, origin)     | ⬜ Not Started | — |
| 11 | Rebasing — The Power Tool (rebase, interactive rebase, golden rule)    | ⬜ Not Started | — |
| 12 | Undoing Things (reset soft/mixed/hard, revert, restore, clean)         | ⬜ Not Started | — |
| 13 | Stashing (stash internals, pop, apply, named stashes, untracked)       | ⬜ Not Started | — |
| 14 | Tags & Releases (lightweight vs annotated, SemVer, on disk)            | ⬜ Not Started | — |
| 15 | Git Workflows (Gitflow vs GitHub Flow vs Trunk-Based Development)      | ⬜ Not Started | — |
| 16 | Pull Requests & Code Review Culture (PR best practices, draft PRs)     | ⬜ Not Started | — |
| 17 | Merge Strategies at Scale (squash, merge commit, rebase-merge)         | ⬜ Not Started | — |
| 18 | Resolving Conflicts Like a Senior (3-way merge, ours vs theirs)        | ⬜ Not Started | — |
| 19 | Git Hooks (pre-commit, commit-msg, pre-push, husky, lint-staged)       | ⬜ Not Started | — |
| 20 | git reflog — The Safety Net (recover anything, HEAD movement log)      | ⬜ Not Started | — |
| 21 | git bisect — Debugging with Git (binary search through history)        | ⬜ Not Started | — |
| 22 | git blame & git log Mastery (codebase archaeology, --follow, -S, -G)   | ⬜ Not Started | — |
| 23 | Submodules & Monorepos (git submodule, subtree, monorepo strategies)   | ⬜ Not Started | — |
| 24 | GitHub Advanced (branch protection, CODEOWNERS, Actions, secrets)      | ⬜ Not Started | — |
| 25 | Packfiles, Garbage Collection & Storage Optimization                   | ⬜ Not Started | — |
| 26 | Performance & Large Repos (shallow clone, sparse checkout, git-lfs)    | ⬜ Not Started | — |

---

## PHASES AT A GLANCE

### PHASE 1 — The Physical Foundation (Topics 1–3)
> You will OPEN the .git folder and understand every single file inside it.
> After this phase, Git stops being a black box forever.

- Topic 01: The `.git/` directory — every file and folder explained, physically
- Topic 02: Git objects — how blobs, trees, commits, and tags are actually stored as files
- Topic 03: The DAG and refs — how commit history is a graph, how branches/tags are just files

### PHASE 2 — Setup & The Three Zones (Topics 4–6)
> The working model that every single git command operates on.

- Topic 04: Configuring Git properly for professional development
- Topic 05: The 3 trees (working dir, staging/index, repository) — full data flow
- Topic 06: `git add` and `git commit` — exact step-by-step of what happens in .git

### PHASE 3 — Core Operations (Topics 7–9)
> Commits, branches, merges — the daily bread of Git.

- Topic 07: Writing commits that tell a story (atomic, conventional, amend)
- Topic 08: Branching — what physically happens when you create/switch/delete branches
- Topic 09: Merging — fast-forward vs 3-way, the merge commit, what changes on disk

### PHASE 4 — Collaboration (Topics 10–14)
> Working with other humans: remotes, rebase, undo, stash, tags.

- Topic 10: Remote repos (origin, upstream, fetch vs pull, tracking branches)
- Topic 11: Rebasing — rewriting history, interactive rebase, when NOT to rebase
- Topic 12: Undoing things — reset, revert, restore, clean (and when to use which)
- Topic 13: Stashing — the temporary shelf for your in-progress work
- Topic 14: Tags & releases — marking versions, semantic versioning

### PHASE 5 — Team & Company-Level Git (Topics 15–19)
> What separates a junior from a senior on a real engineering team.

- Topic 15: Workflows (Gitflow, GitHub Flow, Trunk-Based) — pros/cons/when
- Topic 16: Pull Requests and code review — the social contract of Git
- Topic 17: Merge strategies at scale — squash vs merge commit vs rebase
- Topic 18: Conflict resolution — the 3-way merge algorithm, manual resolution
- Topic 19: Git hooks — automating quality gates

### PHASE 6 — Advanced & Principal-Level (Topics 20–26)
> What separates a senior from a principal/staff engineer.

- Topic 20: reflog — your undo history for Git itself
- Topic 21: bisect — binary search through history to find bugs
- Topic 22: blame & log mastery — forensic investigation of code history
- Topic 23: Submodules & monorepos — large-scale project organization
- Topic 24: GitHub power features — branch rules, CODEOWNERS, Actions, secrets
- Topic 25: Packfiles & GC — how Git optimizes storage under the hood
- Topic 26: Performance at scale — shallow clones, sparse checkout, LFS

---

## HOW TO USE THIS PLAN

| Command | What Happens |
|---|---|
| `Start Topic N` | Full deep-dive lesson for topic N, saved as a file |
| `Quiz me on Topic N` | 5 scenario-based questions (real situations, not MCQ) |
| `Summarize Phase N` | Combined cheat sheet for all topics in that phase |

---

## DEPTH GUARANTEE

Every topic file will be **2000-4000 lines** of content including:
- Full directory trees of `.git/` state before and after operations
- Exact data flow traces for every command
- Hands-on proof commands you can run immediately
- Multiple ASCII diagrams (not one — MULTIPLE, building on each other)
- The actual byte-format of Git objects where relevant
- Connection to previous topics explicitly stated
- Production-grade scenarios from real backend teams

If a topic feels too shallow, it gets rewritten. Period.

---

## STATUS KEY

| Symbol | Meaning |
|--------|---------|
| ⬜ | Not Started |
| 🟡 | In Progress |
| ✅ | Completed |

---

*Last Updated: Topic 02 — Git Objects Deep Dive — ✅ Completed.*
