# Topic 04 — Git Setup & Configuration
### `.gitconfig`, Global/Local/System Scopes, Aliases & Professional Settings

> **Phase 2 — Setup & The Three Zones**
> Prerequisite: [Topic 03 — The DAG & Refs](./topic-03-dag-and-refs.md)
> Next: [Topic 05 — The 3 Trees of Git](./topic-05-three-trees-of-git.md)

---

## Connection to Previous Topics

In Topic 01 you saw that `.git/config` is a file inside your repository that holds
local settings. You also saw that Git reads config from THREE locations, with different
scopes and priorities.

Now we go deep on that entire system — every setting that matters, why it matters,
and how professional engineers configure Git on day one of a new job.

---

## 1. ELI5 — The Analogy

Think of Git config like the settings on your phone.

Your phone has:
- **Factory defaults** (system-level — the manufacturer's baseline)
- **Your account settings** (global — applies to all apps you use)
- **Per-app settings** (local — only for that one app)

When an app needs a setting, it checks: does this app have its own setting? If yes, use it.
If not, fall back to your account settings. If not there, fall back to factory defaults.

Git works the same way. Three levels. Highest priority wins.

---

## 2. The Three Configuration Scopes — Physical Locations

### 2.1 System Config — Lowest Priority

```
Location (Linux/Mac): /etc/gitconfig
Location (Windows):   C:\Program Files\Git\etc\gitconfig
                      (or wherever Git is installed)

Scope: Every user, every repository on this machine
Set with: git config --system <key> <value>
Requires: Administrator / sudo privileges
```

```bash
# Read the system config
git config --system --list

# Set a system-level value (requires sudo/admin)
sudo git config --system core.autocrlf input
```

You rarely touch this as a developer. It's mostly set by:
- Your company's machine provisioning scripts
- Git for Windows installer defaults
- DevOps/platform teams enforcing baseline settings

### 2.2 Global Config — Middle Priority

```
Location (Linux/Mac): ~/.gitconfig
                      OR ~/.config/git/config  (XDG standard)
Location (Windows):   C:\Users\<username>\.gitconfig

Scope: Every repository for your user account
Set with: git config --global <key> <value>
```

This is YOUR personal Git identity and preferences. You set this once on a new machine
and it applies everywhere.

```bash
# Read the global config
cat ~/.gitconfig

# Or via git:
git config --global --list
```

### 2.3 Local Config — Highest Priority

```
Location: .git/config  (inside the repository)

Scope: Only this repository
Set with: git config --local <key> <value>
          (or just: git config <key> <value>  — local is the default)
```

This is where per-project overrides live. Examples:
- Different email for work repos vs personal repos
- Repository-specific push behavior
- Remote URLs and tracking branch config

```bash
# Read the local config
cat .git/config

# Or via git:
git config --local --list
```

### 2.4 The Priority Stack — Full Picture

```
READING a config value (highest priority wins):

┌────────────────────────────────────────────────────────────────┐
│  PRIORITY 1 (highest): .git/config (local)                     │
│  PRIORITY 2:           ~/.gitconfig (global)                   │
│  PRIORITY 3 (lowest):  /etc/gitconfig (system)                 │
└────────────────────────────────────────────────────────────────┘

Example: user.email is set in all three:
  system:  dev@company-default.com
  global:  parsh@personal.com
  local:   parsh@client-project.com   ← THIS ONE WINS for this repo

WRITING a config value:
  git config --system → writes to /etc/gitconfig
  git config --global → writes to ~/.gitconfig
  git config --local  → writes to .git/config  (default if no flag)
  git config          → writes to .git/config  (same as --local)
```

```bash
# See ALL config values AND where each comes from
git config --list --show-origin

# Output example:
# file:/etc/gitconfig                core.symlinks=false
# file:/etc/gitconfig                core.autocrlf=true
# file:/home/parsh/.gitconfig        user.name=Parsh
# file:/home/parsh/.gitconfig        user.email=parsh@personal.com
# file:/home/parsh/.gitconfig        core.editor=vim
# file:.git/config                   core.repositoryformatversion=0
# file:.git/config                   core.filemode=true
# file:.git/config                   user.email=parsh@company.com  ← overrides global
# file:.git/config                   remote.origin.url=git@github.com:...
```

---

## 3. The Config File Format — INI Syntax

Both `~/.gitconfig` and `.git/config` use the same INI-style format:

```ini
[section]
    key = value

[section "subsection"]
    key = value
```

Real example of a `~/.gitconfig`:
```ini
[user]
    name = Parsh
    email = parsh@personal.com
    signingkey = ABC123DEF456  ← GPG key ID for signed commits

[core]
    editor = code --wait       ← VS Code as default editor
    autocrlf = input           ← line ending handling
    excludesfile = ~/.gitignore_global

[color]
    ui = auto                  ← colored output in terminal

[push]
    default = current          ← push to same-named remote branch

[pull]
    rebase = false             ← pull = merge (not rebase)

[merge]
    tool = vimdiff             ← merge conflict tool

[diff]
    tool = vimdiff

[alias]
    st = status
    co = checkout
    br = branch
    lg = log --oneline --graph --all --decorate

[init]
    defaultBranch = main       ← new repos start on 'main' not 'master'

[credential]
    helper = store             ← save credentials (Linux)
    helper = osxkeychain       ← macOS Keychain
    helper = manager           ← Windows Credential Manager
```

---

## 4. First-Time Git Setup — What to Do on a New Machine

### 4.1 The Absolute Minimum

```bash
# Your identity — goes in every commit you make
git config --global user.name "Your Full Name"
git config --global user.email "you@example.com"

# Without these, git commit will fail or use wrong identity
```

### 4.2 The Professional Setup

Run these once on every new machine:

```bash
# ─── IDENTITY ────────────────────────────────────────────────────────
git config --global user.name "Parsh"
git config --global user.email "parsh@company.com"

# ─── DEFAULT BRANCH NAME ─────────────────────────────────────────────
# Modern default: 'main' instead of 'master'
git config --global init.defaultBranch main

# ─── EDITOR ──────────────────────────────────────────────────────────
# VS Code (waits for you to close the file before Git continues)
git config --global core.editor "code --wait"

# Alternatives:
# git config --global core.editor "vim"
# git config --global core.editor "nano"
# git config --global core.editor "notepad"  (Windows)

# ─── LINE ENDINGS ────────────────────────────────────────────────────
# Mac/Linux:
git config --global core.autocrlf input
# Windows:
git config --global core.autocrlf true

# ─── PUSH BEHAVIOR ───────────────────────────────────────────────────
# 'current' = push current branch to same-named remote branch
# (avoids the "no upstream tracking" error)
git config --global push.default current

# ─── PULL BEHAVIOR ───────────────────────────────────────────────────
# Explicit: do you want pull to merge or rebase?
git config --global pull.rebase false    # pull = merge (simpler, safer for beginners)
# OR:
git config --global pull.rebase true     # pull = rebase (cleaner history, for advanced users)

# ─── COLOR OUTPUT ────────────────────────────────────────────────────
git config --global color.ui auto

# ─── GLOBAL GITIGNORE ────────────────────────────────────────────────
# OS/editor files you NEVER want in any repo (no need to add to every .gitignore)
git config --global core.excludesfile ~/.gitignore_global

# Create the global ignore file:
cat >> ~/.gitignore_global << 'EOF'
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# VS Code
.vscode/
*.code-workspace

# JetBrains IDEs
.idea/
*.iml

# Node.js
node_modules/

# Python
__pycache__/
*.pyc
.env

# Logs
*.log
EOF

# ─── CREDENTIALS ─────────────────────────────────────────────────────
# macOS:
git config --global credential.helper osxkeychain
# Windows:
git config --global credential.helper manager
# Linux (save in plain text — not recommended for shared machines):
git config --global credential.helper store
# Linux (cache in memory for 1 hour):
git config --global credential.helper "cache --timeout=3600"

# ─── DIFF & MERGE TOOLS ──────────────────────────────────────────────
# Use VS Code for diffs:
git config --global diff.tool vscode
git config --global difftool.vscode.cmd "code --wait --diff \$LOCAL \$REMOTE"

# Use VS Code for merge conflicts:
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd "code --wait \$MERGED"
```

### 4.3 What Your `~/.gitconfig` Looks Like After All That

```ini
[user]
    name = Parsh
    email = parsh@company.com

[init]
    defaultBranch = main

[core]
    editor = code --wait
    autocrlf = input
    excludesfile = /home/parsh/.gitignore_global

[push]
    default = current

[pull]
    rebase = false

[color]
    ui = auto

[credential]
    helper = osxkeychain

[diff]
    tool = vscode

[difftool "vscode"]
    cmd = code --wait --diff $LOCAL $REMOTE

[merge]
    tool = vscode

[mergetool "vscode"]
    cmd = code --wait $MERGED
```

---

## 5. Per-Repository Overrides — The Work Profile Pattern

### 5.1 The Problem

You work at a company AND have personal/side projects. You need different emails:
- Personal repos: `parsh@personal.com`
- Work repos: `parsh@bigcorp.com`

### 5.2 Solution 1: Manual Per-Repo Config

```bash
cd ~/work/company-api
git config --local user.email "parsh@bigcorp.com"

# Verify:
git config user.email
# Output: parsh@bigcorp.com  (local wins over global)

cat .git/config
# Shows:
# [user]
#     email = parsh@bigcorp.com
```

### 5.3 Solution 2: Conditional Includes (The Professional Way)

In your `~/.gitconfig`, add:

```ini
[user]
    name = Parsh
    email = parsh@personal.com    ← default (personal)

# If the repo is inside ~/work/, use work identity
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

# If the repo is inside ~/clients/, use client identity
[includeIf "gitdir:~/clients/"]
    path = ~/.gitconfig-clients
```

Then create `~/.gitconfig-work`:
```ini
[user]
    email = parsh@bigcorp.com
    signingkey = WORKGPGKEY123
```

And `~/.gitconfig-clients`:
```ini
[user]
    email = parsh.contractor@gmail.com
```

**How it works:**
- When Git reads config for a repo inside `~/work/`, it also reads `~/.gitconfig-work`
- Those values OVERRIDE the global ones
- Completely automatic — no manual setup per repo

```bash
# Verify it works:
cd ~/work/company-api
git config user.email
# Output: parsh@bigcorp.com  ✅

cd ~/personal/side-project
git config user.email
# Output: parsh@personal.com  ✅
```

---

## 6. Core Settings — Deep Explanation

### 6.1 core.autocrlf — Line Ending Handling

```
Windows uses: CRLF (\r\n)  — "Carriage Return + Line Feed"
Mac/Linux use: LF (\n)     — "Line Feed" only

If a Windows developer commits CRLF line endings, Mac/Linux devs see ^M characters.
If a Mac developer commits LF, Windows developers may have issues in some editors.
```

```
core.autocrlf = true      (Windows)
  On checkout: LF → CRLF  (Git gives you CRLF for Windows editors)
  On commit:   CRLF → LF  (Git stores LF in the repository)
  Result: Windows devs are comfortable, repo is always LF

core.autocrlf = input     (Mac/Linux)
  On checkout: no conversion
  On commit:   CRLF → LF  (if you accidentally have CRLF, convert to LF on commit)
  Result: Repo stays LF, nothing injected on checkout

core.autocrlf = false     (not recommended for cross-platform teams)
  No conversion at all
  Result: Whatever line endings exist in files get stored as-is
```

💡 **Best practice for teams:** Add a `.gitattributes` file to the repo:
```
# .gitattributes
* text=auto eol=lf          ← normalize all text files to LF in repo
*.bat text eol=crlf          ← bat files need CRLF even on Mac/Linux
*.sh text eol=lf             ← shell scripts must be LF
*.png binary                 ← binary files — no conversion
*.jpg binary
```

### 6.2 core.fileMode — Permission Tracking

```ini
core.fileMode = true    ← track executable bit changes (chmod +x)
core.fileMode = false   ← ignore permission changes
```

On Windows, Git can't track Unix permissions, so Git for Windows sets this to `false`
automatically. On Mac/Linux it's `true`. If you're on Mac and pair with Windows devs,
you might want `false` to avoid spurious "file mode changed" diffs.

```bash
# Disable for a repo (if you're seeing 'mode change' diffs you don't care about)
git config --local core.fileMode false
```

### 6.3 push.default — What Branch to Push To

```
push.default = nothing    ← refuse to push unless you explicitly name the remote branch
push.default = current    ← push current branch to same-named branch on remote
push.default = upstream   ← push to the tracked upstream branch (may differ in name)
push.default = simple     ← like upstream, but errors if names differ (Git's default since 2.0)
push.default = matching   ← push ALL local branches that have a remote counterpart (old default)
```

Recommendation: `current` — it's predictable and avoids surprises:
```bash
git config --global push.default current
```

### 6.4 pull.rebase — What `git pull` Does

```
pull.rebase = false   ← git pull = git fetch + git merge     (creates a merge commit)
pull.rebase = true    ← git pull = git fetch + git rebase    (rewrites your local commits)
pull.rebase = merges  ← rebase, but preserve local merge commits
```

Neither is universally correct. The choice depends on your team's workflow.
Topic 11 covers this in depth. For now: `false` is the safer default for beginners.

### 6.5 merge.conflictStyle — How Conflict Markers Look

```ini
merge.conflictStyle = merge    ← default: shows ours/theirs
merge.conflictStyle = diff3    ← shows ours/base/theirs (much more useful!)
merge.conflictStyle = zdiff3   ← like diff3 but trims common lines (Git 2.35+)
```

```bash
git config --global merge.conflictStyle diff3
```

With `diff3`, conflict markers look like:
```
<<<<<<< HEAD
your version of the code
||||||| original (base)       ← diff3 adds this! Shows the original before either side changed it
the original before any change
=======
their version of the code
>>>>>>> feature/auth
```

This is MUCH easier to resolve because you can see WHAT EACH SIDE CHANGED from the base.

---

## 7. Aliases — The Productivity Multiplier

### 7.1 What Aliases Are

An alias is a short command that expands to a longer one. They live in `~/.gitconfig`
under `[alias]` and are invoked as `git <alias-name>`.

```ini
[alias]
    st = status
```

Now `git st` runs `git status`.

### 7.2 Aliases vs Shell Aliases

```bash
# Git alias (in ~/.gitconfig):
[alias]
    st = status

git st          ← invoked as git subcommand

# Shell alias (in ~/.bashrc or ~/.zshrc):
alias gs="git status"

gs              ← invoked directly (no "git" prefix)
```

Git aliases are portable (they follow your `.gitconfig` wherever it goes).
Shell aliases are shell-specific and not part of Git itself.

### 7.3 The Essential Aliases — What Senior Engineers Actually Use

```ini
[alias]
    # ── SHORTHAND ─────────────────────────────────────────────────────
    st   = status
    co   = checkout
    br   = branch
    cp   = cherry-pick
    rb   = rebase

    # ── THE LOG ALIAS EVERY DEVELOPER SHOULD HAVE ─────────────────────
    lg   = log --oneline --graph --all --decorate

    # More detailed version with dates and authors:
    lga  = log --graph --all --decorate --pretty=format:'%C(yellow)%h%Creset -%C(red)%d%Creset %s %C(green)(%cr) %C(bold blue)<%an>%Creset'

    # ── STAGING ────────────────────────────────────────────────────────
    # Stage all changes (modified + deleted + new)
    aa   = add --all
    # Stage parts of a file interactively
    ap   = add --patch

    # ── DIFF ───────────────────────────────────────────────────────────
    # What's staged but not committed?
    dc   = diff --cached
    # What's modified but not staged?
    d    = diff

    # ── UNDO ───────────────────────────────────────────────────────────
    # Unstage a file (keep changes in working directory)
    unstage = restore --staged
    # Undo last commit, keep changes staged
    undo = reset --soft HEAD~1
    # Undo last commit, keep changes unstaged
    uncommit = reset HEAD~1

    # ── BRANCH ─────────────────────────────────────────────────────────
    # List branches sorted by most recently committed
    brs  = branch --sort=-committerdate --format='%(refname:short) %(committerdate:relative)'
    # Delete a branch (safe — only if merged)
    brd  = branch -d
    # Delete a branch (force — even if not merged)
    brD  = branch -D

    # ── STASH ──────────────────────────────────────────────────────────
    # Stash everything including untracked files
    save = stash push --include-untracked
    # List stashes
    sl   = stash list

    # ── REMOTE ─────────────────────────────────────────────────────────
    # Fetch and prune deleted remote branches
    fp   = fetch --prune
    # Push current branch to origin (set upstream if needed)
    pub  = push --set-upstream origin HEAD

    # ── USEFUL INSPECTION ──────────────────────────────────────────────
    # Who wrote each line of a file?
    praise = blame
    # Show the files changed in the last commit
    last = log -1 HEAD --stat
    # Search the entire commit history for a string
    find = log -S
```

### 7.4 Shell Command Aliases (! prefix)

Aliases starting with `!` run a shell command instead of a Git subcommand:

```ini
[alias]
    # Run a shell command (! prefix)
    whoami = !git config user.name && git config user.email

    # Delete all branches that have been merged into main
    cleanup = !git branch --merged main | grep -v 'main' | xargs git branch -d

    # Show the root directory of the current repo
    root = !pwd

    # Create a branch and push it to origin in one step
    publish = !git checkout -b $1 && git push --set-upstream origin $1

    # Undo all changes to a file (get it from HEAD)
    discard = !git checkout HEAD --
```

⚠️ **Security note:** Be careful with shell aliases that include `$1` — they evaluate
shell arguments. Don't run aliases from untrusted sources.

### 7.5 Aliases with Arguments

Git aliases can accept arguments — they're appended after expansion:

```ini
[alias]
    co = checkout
```

`git co feature/auth` expands to `git checkout feature/auth` — the argument is appended.

For shell aliases with `!`, use `$1`, `$2`, etc.:

```ini
[alias]
    new = !git checkout -b $1 && git push --set-upstream origin $1
```

`git new feature/login` → creates branch and pushes it.

---

## 8. Important Config Settings Reference

### 8.1 [core] Section

```ini
[core]
    editor = code --wait          # default editor for commit messages, rebase etc
    pager = less                  # what to pipe long output through
    autocrlf = input              # line ending conversion (input/true/false)
    fileMode = true               # track chmod changes
    ignoreCase = false            # case-sensitive filenames
    whitespace = fix              # handle trailing whitespace
    excludesfile = ~/.gitignore_global   # global gitignore file
    hooksPath = .githooks/        # custom hooks directory (useful for teams)
    compression = 9               # zlib compression level (0-9)
    fsmonitor = true              # use filesystem monitor for faster status (Git 2.37+)
    untrackedCache = true         # cache untracked file results (faster status)
```

### 8.2 [user] Section

```ini
[user]
    name = Parsh
    email = parsh@company.com
    signingkey = ABC123DEF456     # GPG key fingerprint for signed commits
    useConfigOnly = true          # NEVER guess user.name from system — require explicit config
```

💡 `useConfigOnly = true` is a great security setting. Without it, Git might fall back to
your Unix username (`git config user.name` returns empty, Git uses system login name instead).
With it, Git requires you to explicitly set `user.name` — prevents accidentally committing
as "ubuntu" or "root" on a server.

### 8.3 [branch] Section

```ini
[branch]
    autoSetupMerge = always       # newly created branches track their remote counterpart
    autoSetupRebase = always      # pulled branches use rebase instead of merge
    sort = -committerdate         # git branch lists branches by most recently committed
```

### 8.4 [fetch] Section

```ini
[fetch]
    prune = true                  # automatically delete local remote-tracking refs
                                  # for branches deleted on remote (after git fetch)
    pruneTags = false             # don't auto-delete local tags (safer default)
    parallel = 0                  # 0 = use optimal number of parallel fetches
```

💡 `fetch.prune = true` is one of the best settings to enable. Without it, `origin/feature/old-branch`
sticks around in your local remote-tracking refs FOREVER after someone deletes it on GitHub.
With it, `git fetch` automatically cleans those up.

### 8.5 [rebase] Section

```ini
[rebase]
    autoStash = true              # auto stash/unstash when rebasing with dirty working dir
    autoSquash = true             # respect fixup!/squash! commit message prefixes
    updateRefs = true             # update dependent branches during rebase (Git 2.38+)
    missingCommitsCheck = warn    # warn if commits are dropped during interactive rebase
```

### 8.6 [diff] Section

```ini
[diff]
    algorithm = histogram         # better diff algorithm (less confusing output)
                                  # options: myers (default), patience, histogram, minimal
    colorMoved = default          # color code moved lines differently from added/removed
    colorMovedWS = allow-indentation-change  # ignore whitespace in moved detection
```

The `histogram` algorithm produces much more readable diffs for real code.

### 8.7 [log] Section

```ini
[log]
    date = relative               # show dates as "2 weeks ago" instead of timestamps
    follow = true                 # always follow file renames in git log
    abbrevCommit = true           # show short SHAs in log output
```

### 8.8 [status] Section

```ini
[status]
    short = false                 # show full status (not --short)
    branch = true                 # show branch info in status output
    showUntrackedFiles = all      # show all untracked files (not just dirs)
    submoduleSummary = true       # show submodule status changes
```

### 8.9 [commit] Section

```ini
[commit]
    gpgsign = false               # sign every commit with GPG (set true for security-conscious teams)
    template = ~/.gitmessage      # default commit message template
    verbose = true                # show diff in commit message editor
```

With `commit.verbose = true`, when your editor opens for a commit message, it shows
the full diff of staged changes below the message area. Extremely useful for writing
better commit messages.

### 8.10 [tag] Section

```ini
[tag]
    gpgSign = true                # sign all annotated tags
    sort = version:refname        # sort tags by version number (v1.10 after v1.9)
```

---

## 9. Commit Message Template

One underused professional feature: a commit message template.

```bash
# Create the template file
cat > ~/.gitmessage << 'EOF'
# <type>(<scope>): <short summary>
# |<---- Max 50 chars ---->|
#
# Why is this change necessary?
#
# How does this address the issue?
#
# What side effects does this change have?
#
# Refs: #<issue-number>
# ─────────────────────────────────────────
# Type: feat, fix, docs, style, refactor, test, chore, perf, ci
# Scope: auth, api, db, ui, config, etc.
# ─────────────────────────────────────────
EOF

# Tell Git to use it
git config --global commit.template ~/.gitmessage
```

Now every time you run `git commit` (without `-m`), this template appears in your editor.
It guides you toward better commit messages. Topic 07 covers Conventional Commits fully.

---

## 10. git config — Full Command Reference

```bash
git config [--system | --global | --local | --worktree] <key> [<value>]
```

| Usage | What It Does |
|---|---|
| `git config --global user.name "Parsh"` | Set a value in global config |
| `git config --local user.email "x@y.com"` | Set a value in local config |
| `git config user.name` | Read a value (highest-priority scope wins) |
| `git config --global user.name` | Read from global only |
| `git config --list` | List all config values (merged, with priority) |
| `git config --list --show-origin` | List all values with file they come from |
| `git config --list --show-scope` | List all values with scope label |
| `git config --global --list` | List only global config |
| `git config --global --edit` | Open global config in editor |
| `git config --local --edit` | Open local config in editor |
| `git config --global --unset user.name` | Remove a value from global config |
| `git config --global --remove-section alias` | Remove entire `[alias]` section |
| `git config --get-all remote.origin.fetch` | Get multi-value entries |
| `git config --global --add core.sshCommand "ssh -i ~/.ssh/work_key"` | Add without replacing |

---

## 11. SSH Configuration — For GitHub Access

### 11.1 Generate an SSH Key

```bash
# Generate a new SSH key pair
ssh-keygen -t ed25519 -C "parsh@company.com"

# Where it goes:
# ~/.ssh/id_ed25519       ← PRIVATE key (NEVER share this)
# ~/.ssh/id_ed25519.pub   ← PUBLIC key (upload to GitHub)
```

### 11.2 Multiple SSH Keys (Work + Personal GitHub)

```bash
# Generate two keys
ssh-keygen -t ed25519 -C "parsh@personal.com" -f ~/.ssh/id_personal
ssh-keygen -t ed25519 -C "parsh@company.com" -f ~/.ssh/id_work
```

Create `~/.ssh/config`:
```
# Personal GitHub account
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_personal

# Work GitHub account
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_work
```

Then use the host alias in your remote URLs:
```bash
# Personal repo: uses personal key
git remote add origin git@github-personal:parsh/my-side-project.git

# Work repo: uses work key
git remote add origin git@github-work:bigcorp/api-service.git
```

### 11.3 Tell Git to Use a Specific SSH Key

```bash
git config --local core.sshCommand "ssh -i ~/.ssh/id_work"
```

---

## 12. git config — Data Flow Trace

When you run `git config user.email`:

```
COMMAND: git config user.email

STEP 1 — DETERMINE SCOPE:
  No scope flag given → read from all scopes

STEP 2 — READ CONFIG FILES (in order, lowest to highest priority):
  Read /etc/gitconfig           → user.email not found
  Read ~/.gitconfig             → user.email = "parsh@personal.com"  ← candidate
  Read .git/config              → user.email = "parsh@company.com"   ← overrides

STEP 3 — OUTPUT:
  Print "parsh@company.com" (highest priority value wins)
```

When you run `git config --global user.email "new@email.com"`:

```
COMMAND: git config --global user.email "new@email.com"

STEP 1 — READ ~/.gitconfig
STEP 2 — FIND [user] section, find 'email' key
  If exists: update the value
  If not exists: create [user] section and add 'email = new@email.com'
STEP 3 — WRITE back to ~/.gitconfig
STEP 4 — Done
```

---

## 13. Beginner Example — First Day Setup

You just got a new laptop. Here's the complete setup sequence:

```bash
# 1. Verify Git is installed
git --version
# Output: git version 2.45.0

# 2. Set your identity
git config --global user.name "Parsh"
git config --global user.email "parsh@company.com"

# 3. Set the default branch name
git config --global init.defaultBranch main

# 4. Set your editor (VS Code)
git config --global core.editor "code --wait"

# 5. Better diff algorithm
git config --global diff.algorithm histogram

# 6. Auto-clean stale remote tracking refs
git config --global fetch.prune true

# 7. Better conflict display
git config --global merge.conflictStyle diff3

# 8. Commit verbose mode (see diff while writing message)
git config --global commit.verbose true

# 9. Essential aliases
git config --global alias.st "status"
git config --global alias.co "checkout"
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.dc "diff --cached"
git config --global alias.undo "reset --soft HEAD~1"

# 10. Verify everything
git config --global --list
```

---

## 14. Real-World Example — Production Scenario

> **Scenario:** You join a new company. On Day 1, the DevOps engineer hands you a
> laptop and says: *"We use GitHub, work repos are at `~/repos/work/`, personal
> allowed in `~/repos/personal/`. We sign all commits with GPG. Our CI enforces
> Conventional Commits. Don't commit as root."*

Here's what you set up:

```bash
# 1. Global identity (personal default)
git config --global user.name "Parsh Kumar"
git config --global user.email "parsh@personal.com"
git config --global user.useConfigOnly true    ← never fall back to system username

# 2. Conditional include for work repos
cat >> ~/.gitconfig << 'EOF'
[includeIf "gitdir:~/repos/work/"]
    path = ~/.gitconfig-work
EOF

# 3. Work profile
cat > ~/.gitconfig-work << 'EOF'
[user]
    email = parsh.kumar@bigcorp.com
    signingkey = ABCD1234EFGH5678

[commit]
    gpgsign = true
EOF

# 4. Commit message template (Conventional Commits format)
cat > ~/.gitmessage << 'EOF'
# feat|fix|docs|style|refactor|test|chore(<scope>): <short description>

# Body: What and why (not how)

# Footer: Refs #<issue>, BREAKING CHANGE: <description>
EOF

git config --global commit.template ~/.gitmessage

# 5. Safety settings
git config --global fetch.prune true
git config --global merge.conflictStyle diff3
git config --global diff.algorithm histogram

# 6. Verify work repo picks up work config
cd ~/repos/work/some-service
git config user.email
# Output: parsh.kumar@bigcorp.com  ✅

cd ~/repos/personal/side-project
git config user.email
# Output: parsh@personal.com  ✅
```

---

## 15. Common Mistakes

### ❌ Mistake 1: Committing with wrong identity

**The mistake:** You forget to set `user.email` locally on a work repo and push commits
with your personal email. The company's Git server rejects them, or worse — they get
merged and the git history shows the wrong author.

**Root cause:** Global config wins by default. Work repos need local override.

**Prevention:**
```bash
# Option A: Manual per-repo override
git config --local user.email "you@company.com"

# Option B: Conditional include (automatic, zero maintenance)
# (shown in section 5.3 above)

# Option C: Safety net — refuse to use system-guessed identity
git config --global user.useConfigOnly true
# Now if you clone a new work repo without the override,
# git commit will FAIL with a helpful error instead of silently using wrong email
```

---

### ❌ Mistake 2: Using the wrong push.default and overwriting wrong branch

**The mistake:** `push.default = matching` (old default) means `git push` pushes ALL
local branches that match a remote branch name. You intended to push `feature/auth`
but your local `main` also got pushed, overwriting remote main.

**Fix:**
```bash
git config --global push.default current
# Now git push only ever pushes the branch you're currently on
```

---

### ❌ Mistake 3: No global .gitignore — IDE files pollute every repo

**The mistake:** You add `.idea/` to a repo's `.gitignore`. Then you add it to
another repo. Then another. You forget one. A `git status` shows `.idea/` as
untracked noise in that repo.

**Fix:**
```bash
git config --global core.excludesfile ~/.gitignore_global
echo ".idea/" >> ~/.gitignore_global
echo ".vscode/" >> ~/.gitignore_global
echo ".DS_Store" >> ~/.gitignore_global
# Now these are ignored in EVERY repo without you having to add them anywhere
```

---

### ❌ Mistake 4: Not understanding that aliases execute in Git's context

**The mistake:** Writing `git alias.push = push --force` and accidentally force-pushing
when you meant to set a reminder alias.

**The reality:** Aliases replace git subcommands directly. They run in the full Git context.
Be deliberate about what aliases do, especially destructive ones.

---

### ❌ Mistake 5: Editing `.git/config` by hand when `git config` exists

**The mistake:** Opening `.git/config` in a text editor and manually changing values.

**Why it's risky:** INI parsing is whitespace-sensitive. A misplaced tab, a missing
newline, or a syntax error silently breaks Git for that repo. Use `git config` instead:

```bash
# WRONG: manually editing .git/config
vim .git/config

# RIGHT: using the proper command
git config --local remote.origin.url "git@github.com:parsh/new-name.git"
```

---

## 16. What Actually Happens — Config Parsing Internals

When Git starts up, it reads ALL config files before executing any command.
Here's the parse order:

```
1. /etc/gitconfig         ← system
2. /etc/gitconfig.d/*     ← system fragments (some Linux distros)
3. ~/.gitconfig           ← global
4. ~/.config/git/config   ← global (XDG spec fallback)
5. .git/config            ← local
6. .git/config.worktree   ← worktree-specific (rare)

For each file:
  Read line by line
  [section] → set current section
  key = value → store as section.key = value
  [section "subsection"] → store as section.subsection.key
  includeIf → evaluate condition, if true → recursively read included file
  include → unconditionally read included file

Result: a flat key-value map, with later values overriding earlier ones
```

The final result is a merged configuration dictionary. When you request `user.email`,
Git looks up that key in the merged dictionary and returns it.

**Why `includeIf` works:** After reading `~/.gitconfig` and finding the `includeIf`
directive, Git checks: is the current repository's `.git/` directory inside
`~/repos/work/`? If yes, it reads the included file and merges its values on top.

---

## 17. Mental Model Checkpoint

**Can you answer these without looking?**

1. You run `git config user.email`. What three files does Git potentially read, and in what order of priority?
2. You need a different email for work repos in `~/work/`. What two approaches can you use?
3. What's the difference between `git config --global --unset` and deleting the line from `~/.gitconfig` manually?
4. What does `pull.rebase = true` change about how `git pull` behaves?
5. Why is `user.useConfigOnly = true` a good security setting?
6. A teammate clones your repo and immediately sees `user.email` is wrong in their commits. What setting would have caught this before commit?
7. What file holds the config for the `[remote "origin"]` section?
8. What does `fetch.prune = true` do, and why is it useful?
9. You create an alias: `git config --global alias.undo "reset --soft HEAD~1"`. What does `git undo` do?
10. What does `merge.conflictStyle = diff3` add to conflict markers that the default doesn't have?

---

## 18. Quick Reference Card

| Command | What It Does |
|---|---|
| `git config --global user.name "Name"` | Set your name globally |
| `git config --global user.email "x@y.com"` | Set your email globally |
| `git config --global init.defaultBranch main` | New repos start on `main` |
| `git config --global core.editor "code --wait"` | Set VS Code as editor |
| `git config --global diff.algorithm histogram` | Better diff output |
| `git config --global fetch.prune true` | Auto-clean stale remote refs |
| `git config --global merge.conflictStyle diff3` | Show base in conflict markers |
| `git config --global push.default current` | Push current branch only |
| `git config --global alias.lg "log --oneline --graph --all --decorate"` | Add lg alias |
| `git config --list --show-origin` | See all config + which file each came from |
| `git config --global --edit` | Open global config in editor |
| `git config --local user.email "work@co.com"` | Override email for this repo only |
| `git config --global --unset core.editor` | Remove a setting |
| `git config user.email` | Read effective value (highest priority) |
| `git config --global core.excludesfile ~/.gitignore_global` | Set global gitignore |

---

## 19. When Would I Use This At Work?

```
╔══════════════════════════════════════════════════════════════════════╗
║              WHEN WOULD I USE THIS AT WORK?                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  1. DAY 1 AT A NEW JOB                                              ║
║     Full setup: identity, editor, aliases, conditional includes,     ║
║     GPG signing, commit template. Takes 10 minutes. Saves hours.     ║
║                                                                      ║
║  2. ONBOARDING NEW TEAM MEMBERS                                     ║
║     You write a "Git setup guide" for your team. You know EXACTLY   ║
║     which settings to recommend (fetch.prune, diff3, histogram)      ║
║     and WHY each one matters.                                        ║
║                                                                      ║
║  3. CI/CD PIPELINE CONFIGURATION                                    ║
║     CI runners need:                                                 ║
║       git config --global user.email "ci@company.com"               ║
║       git config --global user.name "CI Bot"                        ║
║     Without this, automated commits fail or have wrong author.       ║
║                                                                      ║
║  4. DEBUGGING "WHY IS GIT BEHAVING DIFFERENTLY HERE?"               ║
║     git config --list --show-origin immediately shows which file     ║
║     is overriding a value. Fastest diagnostic tool for config issues.║
║                                                                      ║
║  5. ENFORCING TEAM STANDARDS                                        ║
║     Set core.hooksPath = .githooks/ in project .git/config          ║
║     → everyone on the team gets the same hooks automatically          ║
║     → no Husky, no extra dependencies needed                         ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

> **Up Next:** [Topic 05 — The 3 Trees of Git](./topic-05-three-trees-of-git.md)
> With Git configured correctly, we now go deep on the fundamental mental model
> behind EVERY Git command: the Working Directory, the Index (staging area),
> and HEAD — and how data flows between all three.
