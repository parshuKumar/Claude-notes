# Topic 19 — Git Hooks

> **Curriculum**: Git & GitHub Deep Mastery — Zero → Principal Engineer  
> **Phase 5**: Team & Company-Level Git (Topics 15–19)  
> **Prerequisites**: Topic 01 (`.git/` directory), Topic 06 (git add/commit internals), Topic 07 (Commits Done Right), Topic 10 (Remote Repositories)

---

## TABLE OF CONTENTS

1. [ELI5 Anchor — The Airport Security Checkpoint Analogy](#1-eli5-anchor)
2. [What Are Git Hooks?](#2-what-are-git-hooks)
3. [Physical Architecture — `.git/hooks/` on Disk](#3-physical-architecture)
4. [The Complete Hook Catalogue](#4-the-complete-hook-catalogue)
5. [Client-Side Hooks In Depth](#5-client-side-hooks-in-depth)
   - [pre-commit](#51-pre-commit)
   - [prepare-commit-msg](#52-prepare-commit-msg)
   - [commit-msg](#53-commit-msg)
   - [post-commit](#54-post-commit)
   - [pre-push](#55-pre-push)
6. [Server-Side Hooks](#6-server-side-hooks)
7. [Hook Execution Model — Exit Codes, stdin, Arguments](#7-hook-execution-model)
8. [Writing Hooks from Scratch — Language-Agnostic](#8-writing-hooks-from-scratch)
9. [Husky — Sharing Hooks Across a Team](#9-husky)
10. [lint-staged — Running Linters Only on Changed Files](#10-lint-staged)
11. [Husky + lint-staged — The Production Setup](#11-husky-lint-staged-production-setup)
12. [Bypassing Hooks — When and How](#12-bypassing-hooks)
13. [Server-Side Enforcement — The Real Gate](#13-server-side-enforcement)
14. [Production Scenarios](#14-production-scenarios)
15. [Mistakes → Root Cause → Fix → Prevention](#15-mistakes)
16. [Byte-Level: What Git Does When Running a Hook](#16-byte-level-internals)
17. [Mental Model Checkpoint](#17-mental-model-checkpoint)
18. [Connect Everything Backwards](#18-connect-everything-backwards)
19. [Quick Reference Card](#19-quick-reference-card)
20. [When Would I Use This At Work?](#20-when-would-i-use-this-at-work)

---

## 1. ELI5 Anchor

### The Airport Security Checkpoint Analogy

Imagine an airport. Before your flight (the commit/push), you must pass through several checkpoints:

1. **Check-in counter** (pre-commit hook): "Do you have your ID? Is your bag within weight limits?" — If you fail, you're turned away before you even queue.

2. **Security scanner** (commit-msg hook): "Is your boarding pass valid? Does it match your ID?" — Checks the commit message format.

3. **Gate boarding** (pre-push hook): "Are you on the passenger list? Does your ticket have the right destination?" — Final check before you actually leave.

4. **Airport control tower** (server-side hooks): Even if you got through all the passenger checks, the control tower can STILL refuse to let your plane land at the destination airport. This is the server enforcing rules even if the client tried to bypass.

The key insight: **checkpoints 1-3 are on the passenger side (your local machine)**. A determined traveller can bribe their way past them, or take a private charter (commit with `--no-verify`). But the **control tower** (server-side hooks) is completely out of their control. That's why client-side hooks are about developer **convenience and speed**, while server-side hooks are about **enforcement**.

---

## 2. What Are Git Hooks?

Git hooks are **shell scripts** (or any executable) that Git automatically invokes at specific points in its workflow. They live in `.git/hooks/` and have fixed names.

```
Hook lifecycle for git commit:

git commit -m "feat: add login"
      │
      ├─ 1. Run pre-commit hook
      │      ↓ (exit 0 = continue, exit non-zero = abort commit)
      │
      ├─ 2. Open editor / apply -m message
      │
      ├─ 3. Run prepare-commit-msg hook
      │
      ├─ 4. [User edits message in editor, if -e or no -m]
      │
      ├─ 5. Run commit-msg hook
      │      ↓ (exit 0 = continue, exit non-zero = abort commit)
      │
      ├─ 6. Create commit object (write to .git/objects/)
      │
      └─ 7. Run post-commit hook
             (exit code ignored — commit already made)
```

### What Hooks Can Do

```
PREVENT operations:    pre-commit, commit-msg, pre-push can exit non-zero → abort
MODIFY messages:       prepare-commit-msg can rewrite the commit message
NOTIFY/log:            post-commit, post-merge, post-receive can trigger notifications
ENFORCE standards:     check code style, run tests, validate message format
INTEGRATE tools:       update ticket trackers, send Slack messages, run CI locally
```

### What Hooks Cannot Do

```
CANNOT modify the commit content:   A hook can reject a commit but not change
                                    the staged files. (Modify files → re-add → re-commit)
CANNOT be pushed to a remote:       .git/hooks/ is NOT pushed by git push
                                    (it's inside .git/ which is not versioned)
CANNOT be enforced on clients:      Client hooks are bypassable with --no-verify
                                    Server hooks are the real enforcement layer
```

---

## 3. Physical Architecture — `.git/hooks/` on Disk

### Default State After `git init`

```
.git/
├── hooks/
│   ├── applypatch-msg.sample        ← SAMPLE files (not executable — .sample suffix)
│   ├── commit-msg.sample
│   ├── fsmonitor-watchman.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── pre-merge-commit.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── pre-receive.sample
│   ├── prepare-commit-msg.sample
│   ├── push-to-checkout.sample
│   └── update.sample
└── ...
```

The `.sample` suffix is the key: **Git ignores `.sample` files**. They are templates. To activate a hook, you must:
1. Remove the `.sample` suffix (rename the file)
2. Make it executable: `chmod +x .git/hooks/pre-commit`

### Activating a Hook

```bash
# Method 1: Copy sample and edit
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
# Edit the file to add your logic

# Method 2: Create from scratch
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "Running pre-commit hook..."
exit 0  # 0 = success, allow commit to proceed
EOF
chmod +x .git/hooks/pre-commit

# Method 3: Point to a script elsewhere
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
./scripts/pre-commit-checks.sh
EOF
chmod +x .git/hooks/pre-commit
```

### File Permissions — Why `chmod +x` Is Required

```bash
# Before chmod:
ls -la .git/hooks/pre-commit
# -rw-r--r-- 1 alice staff 452 May 28 10:00 pre-commit
# ↑ rw- = read/write, NOT executable

# After chmod +x:
ls -la .git/hooks/pre-commit
# -rwxr-xr-x 1 alice staff 452 May 28 10:00 pre-commit
# ↑ rwx = read/write/execute

# Git checks the executable bit before running a hook.
# If not executable, Git silently ignores the hook (no error, no run).
# This is the #1 cause of "my hook isn't running."
```

### Configuring a Custom Hook Directory

By default, hooks must live in `.git/hooks/`. But `.git/` is not versioned — hooks cannot be shared via `git push`. The solution is a custom hook directory that CAN be in your repo:

```bash
# Configure a repo-level hooks directory:
git config core.hooksPath .githooks
# or:
git config --global core.hooksPath ~/.git-hooks  # global for all repos

# Now put hooks in .githooks/ (tracked by git, pushable):
mkdir .githooks
cat > .githooks/pre-commit << 'EOF'
#!/bin/sh
npm test
EOF
chmod +x .githooks/pre-commit
git add .githooks/
git commit -m "chore: add pre-commit hook"
```

This is how `core.hooksPath` solves the sharing problem before husky existed. Husky (Section 9) does this more ergonomically.

```
.git/
├── hooks/                  ← default path (ignored when core.hooksPath is set)
│   └── pre-commit.sample   ← samples still here but not used
│
.githooks/                  ← custom path (tracked by git, in working tree)
├── pre-commit              ← ACTIVE (executable)
└── commit-msg              ← ACTIVE (executable)
```

---

## 4. The Complete Hook Catalogue

### Client-Side Hooks

| Hook | Triggered By | Abortable? | Typical Use |
|------|-------------|-----------|-------------|
| `pre-commit` | `git commit` (before editor) | ✅ Yes | Lint, tests, secret scanning |
| `prepare-commit-msg` | `git commit` (before editor opens) | ✅ Yes | Auto-populate commit message |
| `commit-msg` | `git commit` (after message written) | ✅ Yes | Validate commit message format |
| `post-commit` | `git commit` (after commit created) | ❌ No | Notify, log, open PR |
| `pre-merge-commit` | `git merge` (before merge commit) | ✅ Yes | Same as pre-commit but for merges |
| `pre-rebase` | `git rebase` | ✅ Yes | Prevent rebasing published branches |
| `post-rewrite` | `git commit --amend`, `git rebase` | ❌ No | Update external tools post-rewrite |
| `post-checkout` | `git checkout`, `git switch` | ❌ No | Rebuild deps, env setup |
| `post-merge` | `git merge` (after merge completes) | ❌ No | Rebuild deps, notify team |
| `pre-push` | `git push` | ✅ Yes | Run full test suite, prevent force-push to main |
| `pre-auto-gc` | `git gc --auto` | ✅ Yes | Prevent automatic GC at bad times |

### Server-Side Hooks

| Hook | Triggered By | Abortable? | Typical Use |
|------|-------------|-----------|-------------|
| `pre-receive` | `git push` arrives at server | ✅ Yes | Enforce branch protection rules |
| `update` | Per-ref update during push | ✅ Yes | Per-branch policies (force-push prevention) |
| `post-receive` | After push accepted | ❌ No | Trigger CI, send notifications, deploy |
| `post-update` | After all refs updated | ❌ No | Update server-side info files |

---

## 5. Client-Side Hooks In Depth

### 5.1 `pre-commit`

**When**: Called by `git commit` before the commit message editor opens.  
**Purpose**: Inspect the snapshot about to be committed.  
**Input**: None (read staged files from `git diff --cached`)  
**Abort**: Exit non-zero → commit aborted. Working tree and index unchanged.

#### Data Flow

```
COMMAND: git commit -m "feat: add login"

STEP 1: Git reads .git/hooks/pre-commit
STEP 2: Git checks: is it executable? (exit if not)
STEP 3: Git runs: .git/hooks/pre-commit
         Environment available to hook:
           GIT_DIR = .git
           GIT_INDEX_FILE = .git/index
           GIT_AUTHOR_NAME, GIT_AUTHOR_EMAIL (if set)
         
STEP 4a: Hook exits 0 → Git continues to commit message step
STEP 4b: Hook exits non-zero → Git prints "pre-commit hook failed"
                                and aborts. Nothing written to .git/objects/.
```

#### What the Hook Can Access

```bash
# The hook can inspect staged content:
git diff --cached --name-only          # list of staged files
git diff --cached                      # full staged diff
git diff --cached --name-only --diff-filter=A  # only added files
git stash list                         # stash list (rarely needed in hooks)

# The hook CANNOT reliably access:
# - Unstaged changes (git diff without --cached)
# - The commit message (not written yet)
# - The commit SHA (commit not created yet)
```

#### Example: Detect Hardcoded Secrets

```bash
#!/bin/sh
# .git/hooks/pre-commit
# Prevent committing hardcoded secrets

# Patterns that look like secrets
PATTERNS=(
  'password\s*=\s*["\x27][^"\x27]'
  'api_key\s*=\s*["\x27][^"\x27]'
  'secret\s*=\s*["\x27][^"\x27]'
  'AWS_SECRET_ACCESS_KEY'
  'PRIVATE KEY'
)

STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

if [ -z "$STAGED_FILES" ]; then
  exit 0
fi

FOUND=0
for pattern in "${PATTERNS[@]}"; do
  MATCHES=$(git diff --cached | grep -iE "$pattern")
  if [ -n "$MATCHES" ]; then
    echo "ERROR: Potential secret detected in staged changes:"
    echo "$MATCHES"
    FOUND=1
  fi
done

if [ $FOUND -ne 0 ]; then
  echo ""
  echo "Commit aborted. Remove secrets before committing."
  echo "If this is a false positive: git commit --no-verify"
  exit 1
fi

exit 0
```

#### Example: Run ESLint Only on Staged Files

```bash
#!/bin/sh
# .git/hooks/pre-commit
# Run ESLint on staged JS/TS files only

STAGED_JS=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(js|ts|jsx|tsx)$')

if [ -z "$STAGED_JS" ]; then
  exit 0  # No JS/TS files staged — skip
fi

echo "Running ESLint on staged files..."
echo "$STAGED_JS" | xargs npx eslint --max-warnings=0

if [ $? -ne 0 ]; then
  echo ""
  echo "ESLint failed. Fix errors before committing."
  exit 1
fi

exit 0
```

#### Hands-On Proof

```bash
# Setup:
git init hook-demo && cd hook-demo
echo "const x = 1;" > app.js
git add app.js

# Create a failing pre-commit hook:
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "Pre-commit hook running!"
exit 1  # always fail
EOF
chmod +x .git/hooks/pre-commit

# Try to commit:
git commit -m "test"
# Output: "Pre-commit hook running!"
#         "pre-commit hook: exit 1"
# PROVE: no commit created (git log shows nothing)
git log --oneline
# fatal: your current branch 'main' does not have any commits

# Fix the hook to succeed:
echo "exit 0" >> .git/hooks/pre-commit  # this won't work cleanly, recreate:
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "Pre-commit hook running!"
exit 0
EOF
chmod +x .git/hooks/pre-commit

git commit -m "test"
# PROVE: now commits successfully
git log --oneline
```

---

### 5.2 `prepare-commit-msg`

**When**: Before the commit message editor opens, after the default message is set.  
**Purpose**: Modify the commit message template — auto-populate from branch name, ticket number, etc.  
**Input**: 3 arguments: (1) path to file containing commit message, (2) type of commit (`message`/`template`/`merge`/`squash`/`commit`), (3) commit SHA (for amend)  
**Abort**: Exit non-zero → abort commit.

#### Auto-Populate Jira Ticket from Branch Name

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg
# Auto-prepend Jira ticket number from branch name

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

# Only for regular commits (not merges, squashes, amends):
if [ "$COMMIT_SOURCE" = "merge" ] || [ "$COMMIT_SOURCE" = "squash" ]; then
  exit 0
fi

# Extract ticket number from branch name (e.g., feature/AUTH-204-add-login → AUTH-204)
BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
TICKET=$(echo "$BRANCH" | grep -oE '[A-Z]+-[0-9]+' | head -1)

if [ -n "$TICKET" ]; then
  # Read existing message
  EXISTING=$(cat "$COMMIT_MSG_FILE")
  
  # Only prepend if message doesn't already have the ticket
  if ! echo "$EXISTING" | grep -q "$TICKET"; then
    # Prepend ticket: "[$TICKET] " before the message
    echo "[$TICKET] $EXISTING" > "$COMMIT_MSG_FILE"
  fi
fi

exit 0
```

Result: On branch `feature/AUTH-204-add-login`, `git commit -m "add endpoint"` becomes commit `[AUTH-204] add endpoint`.

---

### 5.3 `commit-msg`

**When**: After the developer writes the commit message.  
**Purpose**: Validate the commit message format.  
**Input**: 1 argument: path to the file containing the commit message.  
**Abort**: Exit non-zero → commit aborted. Message file is NOT deleted (message preserved for retry).

#### Validate Conventional Commits Format

```bash
#!/bin/sh
# .git/hooks/commit-msg
# Enforce Conventional Commits format: type(scope): description

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Skip merge commits:
if echo "$COMMIT_MSG" | grep -q "^Merge "; then
  exit 0
fi

# Skip revert commits:
if echo "$COMMIT_MSG" | grep -q "^Revert "; then
  exit 0
fi

# Pattern: type(optional-scope): description
# Types: feat, fix, docs, style, refactor, test, chore, perf, ci, build, revert
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\([a-z0-9-]+\))?: .{1,72}"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo ""
  echo "ERROR: Invalid commit message format."
  echo ""
  echo "Expected: <type>(<scope>): <description>"
  echo "Example:  feat(auth): add OAuth2 login endpoint"
  echo ""
  echo "Valid types: feat, fix, docs, style, refactor, test, chore, perf, ci, build, revert"
  echo ""
  echo "Your message was: $COMMIT_MSG"
  exit 1
fi

# Check description length (first line should be ≤ 72 chars)
FIRST_LINE=$(echo "$COMMIT_MSG" | head -1)
if [ ${#FIRST_LINE} -gt 72 ]; then
  echo "ERROR: Commit message first line is too long (${#FIRST_LINE} chars, max 72)."
  exit 1
fi

exit 0
```

#### Hands-On Proof

```bash
# Add commit-msg hook from above to hook-demo repo
chmod +x .git/hooks/commit-msg

# Try invalid format:
git commit --allow-empty -m "added login"
# ERROR: Invalid commit message format.

# Try valid format:
git commit --allow-empty -m "feat(auth): add login endpoint"
# PROVE: succeeds

# Try too-long message:
git commit --allow-empty -m "feat: this is a very long commit message that exceeds the 72 character limit for first lines"
# ERROR: too long
```

---

### 5.4 `post-commit`

**When**: After commit is fully created.  
**Purpose**: Notification, side effects. Cannot abort.  
**Input**: None.  
**Exit code**: Ignored (commit already made).

#### Common Uses

```bash
#!/bin/sh
# .git/hooks/post-commit
# Open GitHub PR creation page after commit on feature branch

BRANCH=$(git symbolic-ref --short HEAD)
REMOTE_URL=$(git config remote.origin.url)

# Only open for non-main branches
if [ "$BRANCH" != "main" ] && [ "$BRANCH" != "develop" ]; then
  # Extract owner/repo from remote URL:
  REPO=$(echo "$REMOTE_URL" | sed 's/.*github.com[:/]\(.*\)\.git/\1/')
  
  if [ -n "$REPO" ]; then
    PR_URL="https://github.com/$REPO/compare/$BRANCH?expand=1"
    echo "Branch: $BRANCH"
    echo "Open PR: $PR_URL"
    # On macOS: open "$PR_URL"
    # On Linux: xdg-open "$PR_URL" 2>/dev/null
  fi
fi
```

---

### 5.5 `pre-push`

**When**: Before `git push` sends data to the remote.  
**Purpose**: Run expensive checks (full test suite, integration tests) that are too slow for pre-commit.  
**Input**: Remote name and URL as arguments. Reads stdin: space-separated `<local-ref> <local-sha1> <remote-ref> <remote-sha1>` for each ref being pushed.  
**Abort**: Exit non-zero → push aborted. Nothing sent to remote.

#### Data Flow

```
COMMAND: git push origin main

STEP 1: Git determines what refs to push
STEP 2: Git runs .git/hooks/pre-push with:
         Argument 1: "origin"
         Argument 2: "https://github.com/alice/repo.git"
         stdin: "refs/heads/main a3f2c1d refs/heads/main d8e7b2c\n"
                 │               │         │               │
                 │               │         │               └─ remote's current SHA
                 │               │         └─ remote ref
                 │               └─ local SHA being pushed
                 └─ local ref

STEP 3a: Hook exits 0 → Git sends the push to remote
STEP 3b: Hook exits non-zero → Push aborted. Remote unchanged.
```

#### Example: Prevent Pushing Directly to `main`

```bash
#!/bin/sh
# .git/hooks/pre-push
# Block direct pushes to main/master (must use PRs)

REMOTE=$1
REMOTE_URL=$2

while read local_ref local_sha remote_ref remote_sha; do
  if [ "$remote_ref" = "refs/heads/main" ] || [ "$remote_ref" = "refs/heads/master" ]; then
    echo "ERROR: Direct push to $remote_ref is not allowed."
    echo "Please open a Pull Request instead."
    echo ""
    echo "If you are a release engineer performing a release merge:"
    echo "  git push --no-verify origin main"
    exit 1
  fi
done

exit 0
```

#### Example: Run Full Test Suite Before Push

```bash
#!/bin/sh
# .git/hooks/pre-push
# Run tests before pushing (too slow for pre-commit)

echo "Running test suite before push..."
npm test

if [ $? -ne 0 ]; then
  echo ""
  echo "Tests failed. Push aborted."
  echo "Fix tests or use --no-verify to bypass (not recommended)."
  exit 1
fi

echo "All tests passed. Pushing..."
exit 0
```

---

## 6. Server-Side Hooks

Server-side hooks run on the remote (GitHub, GitLab, your self-hosted Gitea, Bitbucket Server, etc.). They receive push data and can accept or reject it.

**Critical difference**: Server-side hooks CANNOT be bypassed by the developer. `git push --no-verify` only skips client-side hooks; it has no effect on server-side hooks.

### `pre-receive`

**When**: At the start of a push, before any refs are updated.  
**Input**: stdin — one line per ref: `<old-value> <new-value> <ref-name>`  
**Abort**: Exit non-zero → entire push rejected. No refs updated.

```bash
#!/bin/sh
# Server-side pre-receive hook
# Runs on the remote server, not the developer's machine

while read old_sha new_sha ref_name; do
  # Prevent force-push to main:
  if [ "$ref_name" = "refs/heads/main" ]; then
    # Check if this is a force push:
    # A force push means old_sha is NOT an ancestor of new_sha
    if ! git merge-base --is-ancestor "$old_sha" "$new_sha"; then
      echo "ERROR: Force push to main is not allowed."
      exit 1
    fi
  fi
  
  # Prevent pushing commits > 10MB:
  # (pseudocode — actual implementation uses git rev-list + cat-file)
done

exit 0
```

### `update`

**When**: Once per ref being updated (unlike pre-receive which runs once for all refs).  
**Purpose**: Per-branch policies — different rules for different branches.  
**Arguments**: (1) ref name, (2) old SHA, (3) new SHA.

```bash
#!/bin/sh
# Server-side update hook
# Arguments: $1=ref, $2=old_sha, $3=new_sha

REF=$1
OLD_SHA=$2
NEW_SHA=$3

# Protect tagged releases from deletion/modification:
if echo "$REF" | grep -q "^refs/tags/v"; then
  if [ "$OLD_SHA" != "0000000000000000000000000000000000000000" ]; then
    echo "ERROR: Release tags cannot be modified. Tag: $REF"
    exit 1
  fi
fi

exit 0
```

### `post-receive`

**When**: After all refs are updated. Cannot abort (push already accepted).  
**Purpose**: Trigger CI, send notifications, deploy.  
**Input**: Same as pre-receive (stdin).

```bash
#!/bin/sh
# Server-side post-receive hook
# Trigger deployment after push to main

while read old_sha new_sha ref_name; do
  if [ "$ref_name" = "refs/heads/main" ]; then
    echo "Deployment triggered for main..."
    curl -X POST https://ci.internal/deploy \
      -H "Content-Type: application/json" \
      -d "{\"ref\": \"main\", \"sha\": \"$new_sha\"}"
  fi
done

exit 0
```

### Server-Side Hooks on GitHub

GitHub does NOT allow arbitrary server-side hooks on its managed service. Instead, GitHub provides equivalent functionality through:

```
GitHub's equivalents to server-side hooks:

pre-receive  →  Branch Protection Rules
               - Require PR reviews
               - Require status checks
               - Prevent force-push
               - Require linear history

post-receive →  GitHub Actions (triggered by push/PR events)
               - Run CI on every push
               - Deploy on push to main
               - Send Slack notifications

update       →  Per-branch branch protection rules
               - Can set different rules per branch pattern
```

On **self-hosted Git servers** (Gitea, GitLab self-managed, Bitbucket Server, Gerrit), you have full access to write custom server-side hooks.

---

## 7. Hook Execution Model — Exit Codes, stdin, Arguments

### Exit Code Contract

```
Exit code 0:   Hook succeeded → Git continues with the operation
Exit code 1-N: Hook failed   → Git aborts the operation (if abortable)
               Git prints any stdout/stderr from the hook to the terminal.

EXCEPTION: post-* hooks (post-commit, post-merge, post-checkout, post-receive)
           Git ignores their exit code. The operation has already completed.
```

### Standard Input and Arguments

```
Hook           Arguments                         stdin
─────────────────────────────────────────────────────────────────────────
pre-commit     none                              /dev/null (empty)
prepare-commit-msg  <msg-file> [<commit-type> [<sha>]]   /dev/null
commit-msg     <msg-file>                        /dev/null
post-commit    none                              /dev/null
pre-push       <remote-name> <remote-url>        per-ref lines
pre-receive    none                              per-ref lines
update         <ref-name> <old-sha> <new-sha>    /dev/null
post-receive   none                              per-ref lines
```

### Environment Variables Available to Hooks

```bash
# Always available:
GIT_DIR              # path to .git/ (e.g., "/home/alice/myapp/.git")
GIT_INDEX_FILE       # path to index file

# Available during commit hooks:
GIT_AUTHOR_NAME      # from git config user.name
GIT_AUTHOR_EMAIL     # from git config user.email
GIT_AUTHOR_DATE      # timestamp (set for rebase/cherry-pick)
GIT_COMMITTER_NAME   
GIT_COMMITTER_EMAIL  
GIT_COMMITTER_DATE   

# Available during rebase hooks:
GIT_REBASE_HEAD      # HEAD being applied

# Check all environment:
printenv | grep GIT_
```

### Hook Language — Not Just Shell

Hooks are just executables. Any language works, as long as the file has a shebang line and is executable:

```bash
#!/usr/bin/env python3
# .git/hooks/commit-msg (Python hook)
import sys
import re

msg_file = sys.argv[1]
with open(msg_file) as f:
    msg = f.read().strip()

pattern = r'^(feat|fix|docs|chore|test|refactor)(\(.+\))?: .{1,72}'
if not re.match(pattern, msg):
    print("ERROR: Invalid commit message format")
    print(f"Got: {msg}")
    sys.exit(1)
sys.exit(0)
```

```javascript
#!/usr/bin/env node
// .git/hooks/pre-commit (Node.js hook)
const { execSync } = require('child_process');

try {
  execSync('npx eslint --max-warnings=0 $(git diff --cached --name-only --diff-filter=ACM | grep -E "\\.(js|ts)$")', { stdio: 'inherit' });
  process.exit(0);
} catch (e) {
  process.exit(1);
}
```

---

## 8. Writing Hooks from Scratch — Language-Agnostic

### The Template Pattern

Every hook follows this structure:

```bash
#!/bin/sh
# Hook: <name>
# Purpose: <what this does>
# Abortable: yes/no
# Author: <name>

# 1. Parse arguments / stdin
COMMIT_MSG_FILE=$1  # (or however the hook receives input)

# 2. Do your check
RESULT=$(your_check_command)

# 3. Evaluate and report
if [ condition_fails ]; then
  echo "ERROR: Description of what went wrong"
  echo "HOW TO FIX: Hint for the developer"
  echo "BYPASS: git commit --no-verify (use with care)"
  exit 1
fi

# 4. Success
exit 0
```

### Making Hooks Portable

```bash
# Check if a tool is available before using it:
if ! command -v npx &> /dev/null; then
  echo "WARN: npx not found, skipping eslint check"
  exit 0
fi

# Check Node.js version:
NODE_VERSION=$(node -e "process.exit(Number(process.versions.node.split('.')[0]))" 2>/dev/null; echo $?)

# Check if running in CI (skip local hooks in CI pipelines):
if [ -n "$CI" ] || [ -n "$GITHUB_ACTIONS" ]; then
  exit 0  # CI has its own checks
fi
```

### Testing a Hook Without Committing

```bash
# Run the hook directly:
.git/hooks/pre-commit
echo "Exit code: $?"

# Run with the hook's expected arguments:
echo "feat: test message" > /tmp/test-msg.txt
.git/hooks/commit-msg /tmp/test-msg.txt
echo "Exit code: $?"

# Run commit but show what hook does without actually committing:
# (No built-in Git way to do this — just run the hook script directly)
```

---

## 9. Husky — Sharing Hooks Across a Team

### The Problem Husky Solves

`.git/hooks/` is not versioned. If Alice writes a great `pre-commit` hook, Bob cannot get it by cloning or pulling. Every developer must manually install hooks. This means:
- Hooks are optional (people forget to install them)
- Hooks drift (different versions across team members)
- Onboarding new developers requires manual steps

**Husky** solves this by:
1. Storing hook scripts in the repo (e.g., `.husky/` directory)
2. Automatically pointing `core.hooksPath` to `.husky/` after `npm install`
3. Every developer who runs `npm install` gets the latest hooks automatically

### Installing Husky

```bash
# Install husky:
npm install --save-dev husky

# Initialize husky (creates .husky/ directory and configures package.json):
npx husky init
# Creates: .husky/pre-commit (with sample content)
# Adds to package.json: "prepare": "husky"
# The "prepare" script runs automatically after npm install
```

### What `npx husky init` Creates

```json
// package.json (after husky init):
{
  "scripts": {
    "prepare": "husky"    ← runs after npm install
  },
  "devDependencies": {
    "husky": "^9.0.0"
  }
}
```

```
.husky/
├── pre-commit       ← actual hook script (tracked by git)
└── _/
    ├── .gitignore   ← ignores the _/ directory from git
    └── husky.sh     ← husky's bootstrap script (runs before your hook)
```

```bash
# .husky/pre-commit (default content):
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"  # source husky's bootstrap

npm test  # your actual command
```

### How Husky Works Internally

```
Developer runs: npm install
  ↓
npm runs "prepare" script: npx husky
  ↓
husky sets: git config core.hooksPath .husky
  ↓
.git/config now contains:
  [core]
    hooksPath = .husky

Developer runs: git commit -m "feat: add login"
  ↓
Git looks for hooks in .husky/ (not .git/hooks/)
  ↓
Git runs .husky/pre-commit
  ↓
husky.sh bootstraps the environment
  ↓
Your command runs (npm test, eslint, etc.)
```

```bash
# Verify husky is configured:
git config core.hooksPath
# Output: .husky

# Add a hook:
echo "npm test" > .husky/pre-commit
# OR using husky's CLI:
echo "npm test" > .husky/pre-commit && chmod +x .husky/pre-commit
```

### Husky Hook Examples

```bash
# .husky/pre-commit — run lint-staged (Section 10):
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npx lint-staged

# .husky/commit-msg — validate Conventional Commits:
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npx commitlint --edit $1

# .husky/pre-push — run full test suite:
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npm run test:integration
```

### Husky in CI

CI environments should NOT run local hooks (CI has its own pipeline). Husky detects CI environments and skips installation:

```bash
# Husky auto-skips if CI environment variable is set:
# CI=true npm install  → husky does NOT set core.hooksPath

# Or manually skip in package.json:
{
  "scripts": {
    "prepare": "husky || true"  ← || true prevents CI failure if husky not installed
  }
}
```

---

## 10. lint-staged — Running Linters Only on Changed Files

### The Problem lint-staged Solves

Running your linter on the entire codebase in a pre-commit hook is slow. For a large repo with 10,000 files, running `eslint .` takes 30-60 seconds — every commit. Developers stop committing frequently, or start using `--no-verify`.

**lint-staged** runs linters **only on the files that are staged** (about to be committed). If you changed 3 files, only those 3 files are linted.

```
Without lint-staged:
  pre-commit: npx eslint .
  → Lints 8,432 files
  → Takes 47 seconds
  → Developer uses --no-verify after 3 days

With lint-staged:
  pre-commit: npx lint-staged
  → Lints only staged files (e.g., 2 .js files, 1 .ts file)
  → Takes 0.8 seconds
  → Developer never uses --no-verify
```

### Installing lint-staged

```bash
npm install --save-dev lint-staged
```

### Configuring lint-staged

In `package.json` or `.lintstagedrc`:

```json
// package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss}": [
      "stylelint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"
    ],
    "*.py": [
      "black",
      "flake8"
    ]
  }
}
```

```
CONFIGURATION EXPLAINED:

"*.{js,jsx,ts,tsx}": [...]
│
└─ glob pattern: matches any staged file ending in .js, .jsx, .ts, .tsx

["eslint --fix", "prettier --write"]
│                │
│                └─ runs prettier on the file, writes fixes back
└─ runs eslint, --fix applies auto-fixable violations

IMPORTANT: lint-staged passes the filename as the last argument automatically.
           "eslint --fix" becomes "eslint --fix src/auth.js" when run.
           
IMPORTANT: If a tool modifies the file (--fix, --write):
           lint-staged re-adds the modified file to the index automatically.
           This means: your commit includes the auto-fixed version.
```

### What lint-staged Does Internally

```
COMMAND: git commit -m "feat: add login"
  ↓
pre-commit hook: npx lint-staged
  ↓
lint-staged reads: git diff --cached --name-only
                   Output: ["src/auth.js", "src/login.test.js"]
  ↓
lint-staged matches patterns:
  src/auth.js       → matches "*.{js,...}" → run ["eslint --fix", "prettier --write"]
  src/login.test.js → matches "*.{js,...}" → run ["eslint --fix", "prettier --write"]
  ↓
For each file:
  1. Run eslint --fix src/auth.js
     → Auto-fixes violations, writes to disk
  2. Run prettier --write src/auth.js  
     → Formats, writes to disk
  3. git add src/auth.js  ← lint-staged re-stages the fixed version
  ↓
All tools exit 0 → lint-staged exits 0 → pre-commit hook exits 0 → commit proceeds
  ↓
Commit object created with the auto-fixed, properly formatted files
```

### lint-staged with Tests

```json
{
  "lint-staged": {
    "src/**/*.test.{js,ts}": "jest --bail --findRelatedTests"
  }
}
```

`--findRelatedTests` runs only the test files related to the staged source files — another scoped execution strategy.

---

## 11. Husky + lint-staged — The Production Setup

This is the standard setup for professional JavaScript/TypeScript teams.

### Full Setup from Scratch

```bash
# 1. Install tools:
npm install --save-dev husky lint-staged prettier eslint commitlint @commitlint/config-conventional

# 2. Initialize husky:
npx husky init

# 3. Add pre-commit hook:
echo "npx lint-staged" > .husky/pre-commit

# 4. Add commit-msg hook:
echo "npx commitlint --edit \$1" > .husky/commit-msg

# 5. Configure lint-staged in package.json:
# (add the "lint-staged" key as shown in Section 10)

# 6. Configure commitlint:
cat > .commitlintrc.json << 'EOF'
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "scope-case": [2, "always", "lower-case"],
    "subject-max-length": [2, "always", 72]
  }
}
EOF

# 7. Commit all config:
git add .husky/ .commitlintrc.json package.json package-lock.json
git commit -m "chore: add husky + lint-staged + commitlint"
```

### The Complete File Tree

```
myapp/
├── .husky/
│   ├── _/
│   │   ├── .gitignore      ← prevents _/ from being committed
│   │   └── husky.sh        ← husky bootstrap (committed)
│   ├── pre-commit          ← "npx lint-staged"
│   └── commit-msg          ← "npx commitlint --edit $1"
│
├── .commitlintrc.json      ← commitlint config
│
├── package.json
│   └── "lint-staged": { ... }
│   └── "scripts": { "prepare": "husky" }
│
└── .git/
    └── config
        └── [core]
            └── hooksPath = .husky   ← set by husky after npm install
```

### Developer Experience After Setup

```bash
# New developer joins, clones repo:
git clone https://github.com/company/myapp.git
cd myapp
npm install
# ↑ "prepare" script runs → husky sets core.hooksPath → hooks active

# Developer makes a change:
git add src/auth.js
git commit -m "adding login"
# ↑ lint-staged runs eslint + prettier on auth.js
# ↑ commitlint fails: "adding login" doesn't match Conventional Commits
# Output: "✖ subject may not be empty [subject-empty]"
#         "✖ type may not be empty [type-empty]"

# Developer retries with correct format:
git commit -m "feat(auth): add login endpoint"
# ↑ commitlint passes
# ↑ lint-staged passes (eslint, prettier)
# ↑ commit created
```

### commitlint — Full Configuration

```json
// .commitlintrc.json
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "type-enum": [2, "always", [
      "feat", "fix", "docs", "style", "refactor",
      "test", "chore", "perf", "ci", "build", "revert"
    ]],
    "scope-enum": [2, "always", [
      "auth", "api", "ui", "db", "ci", "deps"
    ]],
    "subject-max-length": [2, "always", 72],
    "body-max-line-length": [2, "always", 100],
    "header-max-length": [2, "always", 100]
  }
}
```

Rule format: `[severity, applicability, value]`
- Severity: 0=off, 1=warn, 2=error
- Applicability: "always" or "never"

---

## 12. Bypassing Hooks — When and How

### The `--no-verify` Flag

```bash
git commit --no-verify -m "wip: saving state before meeting"
#           │
#           └─ Skip pre-commit and commit-msg hooks
#              Does NOT skip post-commit (can't abort a commit that exists)
#              Shorthand: -n

git push --no-verify
#          └─ Skip pre-push hook
#             Does NOT affect server-side hooks (those are on the remote)
```

### When `--no-verify` Is Legitimate

```
LEGITIMATE USE CASES:
  ✅ "wip" commits that will be squashed before PR (temp checkpoint)
  ✅ Emergency hotfix at 3am when tests are slow (accept the risk, fix later)
  ✅ The hook is broken/outdated and blocking valid commits
  ✅ Release engineer doing admin operations (merging branches, tagging)
  ✅ Auto-generated commits from scripts (e.g., version bumps in CD pipeline)

ILLEGITIMATE USE CASES:
  ❌ "The tests are failing and I'll fix them later" (they won't)
  ❌ "The linter is annoying" (fix the code)
  ❌ "I don't agree with the commit message format" (team discussion needed)
  ❌ Routinely bypassing security scans (defeats the purpose)
```

### Detecting Hook Bypass in CI

Since `--no-verify` cannot be prevented client-side, enforce the same checks server-side:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx eslint .            # same as pre-commit hook
      - run: npx commitlint --from HEAD~1 --to HEAD  # same as commit-msg hook
```

This ensures that even if a developer used `--no-verify`, CI catches it. The PR cannot merge until CI is green.

### Removing Hooks Temporarily

```bash
# Method 1: Temporarily rename (easy to restore):
mv .git/hooks/pre-commit .git/hooks/pre-commit.bak
git commit -m "..."
mv .git/hooks/pre-commit.bak .git/hooks/pre-commit

# Method 2: Remove executable bit:
chmod -x .git/hooks/pre-commit
git commit -m "..."
chmod +x .git/hooks/pre-commit

# Method 3: One-time bypass:
git commit --no-verify -m "..."
```

---

## 13. Server-Side Enforcement — The Real Gate

Client-side hooks can always be bypassed. For real enforcement, use server-side mechanisms.

### GitHub's Enforcement Layer

```
Branch Protection Rules (GitHub Settings → Branches → Add rule):

┌─────────────────────────────────────────────────────────────┐
│  Branch name pattern: main                                   │
│                                                              │
│  ✅ Require a pull request before merging                    │
│     Required approving reviews: 2                            │
│     ✅ Dismiss stale pull request approvals                  │
│     ✅ Require review from Code Owners                       │
│                                                              │
│  ✅ Require status checks to pass before merging             │
│     ✅ Require branches to be up to date                     │
│     Status checks: build, test, lint, security-scan          │
│                                                              │
│  ✅ Require conversation resolution before merging           │
│                                                              │
│  ✅ Require signed commits                                   │
│                                                              │
│  ✅ Require linear history                                   │
│                                                              │
│  ✅ Do not allow bypassing the above settings                │
│     (applies even to admins)                                 │
│                                                              │
│  ✅ Restrict who can push to matching branches               │
│     Allow: @release-team                                     │
└─────────────────────────────────────────────────────────────┘
```

### GitHub Actions as `post-receive` Equivalent

```yaml
# .github/workflows/enforce.yml
# This runs on every push to any branch — cannot be bypassed

name: Enforce Standards
on:
  push:
    branches: ['**']
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx eslint . --max-warnings=0

  commit-format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - run: npm ci
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to HEAD

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gitleaks/gitleaks-action@v2   # dedicated secret scanning
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### The Full Enforcement Stack

```
ENFORCEMENT LAYERS (weakest to strongest):

Layer 1: pre-commit hook (client-side)
  → Bypassable with --no-verify
  → Good for: fast feedback, developer convenience
  → When it runs: every commit on developer's machine

Layer 2: commit-msg hook (client-side)
  → Bypassable with --no-verify
  → Good for: keeping team conventions visible during daily work
  → When it runs: every commit on developer's machine

Layer 3: pre-push hook (client-side)
  → Bypassable with --no-verify
  → Good for: expensive checks before sharing code
  → When it runs: every push from developer's machine

Layer 4: GitHub Actions (server-side equivalent)
  → NOT bypassable (runs on GitHub's infrastructure)
  → Good for: the actual enforcement layer
  → When it runs: every push to GitHub

Layer 5: Branch Protection (GitHub-enforced)
  → NOT bypassable (GitHub enforces, even for admins if configured)
  → Good for: merge gate enforcement
  → When it runs: every PR merge attempt

PRINCIPLE: Client-side hooks = developer experience + early feedback
           Server-side = enforcement + compliance
```

---

## 14. Production Scenarios

### Scenario 1: Setting Up Quality Gates for a New Team

**Context**: You're leading a team of 8 engineers building a Node.js API. The team is joining from different backgrounds with different coding styles. You need consistent code quality from day one.

```bash
# Day 1: Set up the toolchain
npm install --save-dev \
  husky \
  lint-staged \
  eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  prettier \
  commitlint @commitlint/config-conventional \
  jest

# Initialize:
npx husky init

# pre-commit: lint and format staged files
echo "npx lint-staged" > .husky/pre-commit

# commit-msg: enforce Conventional Commits
echo "npx commitlint --edit \$1" > .husky/commit-msg

# lint-staged config:
cat >> package.json << 'JSON'
// (add to package.json manually):
"lint-staged": {
  "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
  "*.{json,md}": ["prettier --write"]
}
JSON

# CI enforcement (same checks, server-side):
# .github/workflows/ci.yml
# → runs eslint, commitlint, jest on every push

# Document in CONTRIBUTING.md:
cat > CONTRIBUTING.md << 'EOF'
# Contributing

## Setup
```bash
npm install  # automatically installs git hooks via husky
```

## Commit Format
This project uses [Conventional Commits](https://conventionalcommits.org).

Valid: `feat(auth): add OAuth2 login`
Invalid: `added login`

## Code Style
Prettier and ESLint are enforced automatically on commit.
EOF

git add .husky/ .commitlintrc.json .eslintrc.json .prettierrc package.json CONTRIBUTING.md
git commit -m "chore: set up code quality toolchain (husky + lint-staged + commitlint)"
git push origin main
```

### Scenario 2: The Slow Hook Problem

**Context**: Your team's pre-push hook runs `npm test` which takes 4 minutes. Developers are using `--no-verify` constantly.

```bash
# DIAGNOSIS: what's slow?
time .git/hooks/pre-push  # measure

# SOLUTION OPTIONS:

# Option A: Move slow tests to CI only, keep fast unit tests in pre-push
cat > .husky/pre-push << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
# Only run unit tests (fast), not integration tests (slow)
# Integration tests run in CI (GitHub Actions)
npm run test:unit  # 15 seconds vs 4 minutes
EOF

# Option B: Use --passWithNoTests and cache:
cat > .husky/pre-push << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
# Only run tests if src/ files changed
CHANGED_SRC=$(git diff --cached --name-only origin/HEAD...HEAD | grep "^src/" | wc -l)
if [ "$CHANGED_SRC" -gt 0 ]; then
  npm run test:unit
fi
EOF

# Option C: Remove pre-push entirely, add pre-commit with fast checks
# Move the test coverage to CI, rely on branch protection to enforce
rm .husky/pre-push
# Add to CI: required status check "test" on PRs
```

### Scenario 3: Secret Accidentally Staged

**Context**: A developer stages a file with a real AWS secret key. The pre-commit secret detection hook should catch it.

```
SCENARIO TRACE:

Developer runs:
  git add .env.local     ← accidentally stages .env.local which has AWS_SECRET

pre-commit hook runs:
  grep -rE 'AWS_SECRET_ACCESS_KEY\s*=\s*["\x27]?[A-Za-z0-9+/]{40}' (staged diff)
  → MATCH FOUND: "AWS_SECRET_ACCESS_KEY=abc123def456..."

Hook output:
  "ERROR: Potential secret detected in staged changes:"
  "AWS_SECRET_ACCESS_KEY=abc123..."
  ""
  "Commit aborted. Remove secrets before committing."

Developer fixes:
  echo ".env.local" >> .gitignore     # ensure it won't be added again
  git rm --cached .env.local          # unstage
  # Rotate the secret (IMPORTANT — treat it as compromised even though not pushed)
  git add .gitignore
  git commit -m "chore: add .env.local to gitignore"
```

### Scenario 4: Branch-Specific Hook Behavior

**Context**: You want the pre-push hook to run the full test suite when pushing to `main` or `release/*`, but only unit tests when pushing feature branches.

```bash
#!/bin/sh
# .husky/pre-push

. "$(dirname -- "$0")/_/husky.sh"

REMOTE=$1
REMOTE_URL=$2

while read local_ref local_sha remote_ref remote_sha; do
  case "$remote_ref" in
    refs/heads/main|refs/heads/release/*)
      echo "Pushing to protected branch — running full test suite..."
      npm run test:all || exit 1
      ;;
    refs/heads/feature/*|refs/heads/fix/*)
      echo "Pushing feature branch — running unit tests only..."
      npm run test:unit || exit 1
      ;;
    *)
      echo "Pushing to $remote_ref — skipping tests"
      ;;
  esac
done

exit 0
```

---

## 15. Mistakes → Root Cause → Fix → Prevention

### Mistake 1: Hook Not Running (Forgot `chmod +x`)

**Symptom**: You created a pre-commit hook but it never runs. No output, no errors.

**Root cause**: The hook file is not executable. Git silently ignores non-executable hooks.

**Diagnose and Fix**:
```bash
ls -la .git/hooks/pre-commit
# -rw-r--r-- (missing x bits)

chmod +x .git/hooks/pre-commit
ls -la .git/hooks/pre-commit
# -rwxr-xr-x

# Test it manually:
.git/hooks/pre-commit
```

**Prevention**: Always run `chmod +x` immediately after creating a hook. Or use husky (handles permissions automatically).

### Mistake 2: Hook Breaks After `git clone`

**Symptom**: Hooks work on the original developer's machine but not after cloning.

**Root cause**: `.git/hooks/` is not cloned (it's inside `.git/` which is not tracked). The clone starts with only sample hooks.

**Fix**:
```bash
# Developers must manually install hooks, OR
# Use husky (npm install sets up hooks automatically), OR
# Use a Makefile setup target:
#   make setup → runs git config core.hooksPath .githooks
```

**Prevention**: Use husky. The `prepare` script in `package.json` runs automatically on `npm install`, making hook setup zero-friction.

### Mistake 3: Hook Fails on CI Because Tools Not Installed

**Symptom**: CI fails with "eslint: command not found" even though it works locally.

**Root cause**: The pre-commit hook runs `eslint` but CI runs the hook differently (or you have `npm run lint` in CI that works, but the raw `eslint` doesn't resolve without `npx`).

**Fix**:
```bash
# Always use npx to invoke locally-installed tools:
npx eslint --fix src/

# OR use the full path:
./node_modules/.bin/eslint --fix src/

# OR skip hooks in CI (CI has its own lint step):
# In CI: HUSKY=0 npm install (husky doesn't run)
# Or:    npm install --ignore-scripts
```

**Prevention**: Use `npx` in hooks. Set `HUSKY=0` in CI environment. Run linters directly in CI pipeline (not via hooks).

### Mistake 4: Hook Modifies Files But Changes Not Staged

**Symptom**: Prettier fixes a file in the hook but the committed version still has the formatting issue.

**Root cause**: The hook ran Prettier (which modified the working tree) but didn't re-add the file to the index. The commit captured the PRE-prettier version from the index, not the modified working tree.

```
WRONG:
  pre-commit hook:
    prettier --write src/auth.js  ← modifies working tree
    # forgot: git add src/auth.js
  
  Result: index has old version, working tree has prettier version
  Commit: captures index → old un-prettied version is committed

CORRECT:
  pre-commit hook:
    prettier --write src/auth.js
    git add src/auth.js           ← re-stage the prettied version
  
  OR use lint-staged (it handles re-staging automatically)
```

**Fix and Prevention**: Always `git add` after auto-fixing in a hook. OR use lint-staged which handles this automatically.

### Mistake 5: Infinite Loop in post-commit Hook

**Symptom**: `git commit` hangs or creates commits in an infinite loop.

**Root cause**: The `post-commit` hook itself runs a `git commit`, which triggers `post-commit` again, which triggers another `git commit`, etc.

```bash
# BAD hook:
#!/bin/sh
# post-commit: auto-commit a timestamp file
date > .last-commit-time
git add .last-commit-time
git commit -m "chore: update timestamp"  ← triggers post-commit again!
```

**Fix**:
```bash
# Option A: Use --no-verify to prevent hook re-triggering:
git commit --no-verify -m "chore: update timestamp"

# Option B: Check if the hook was triggered by itself:
if git log --format='%s' -1 | grep -q "chore: update timestamp"; then
  exit 0  # Skip if last commit was our auto-commit
fi
```

**Prevention**: Never run `git commit` inside a post-commit hook without `--no-verify` or an explicit loop guard.

---

## 16. Byte-Level Internals — What Git Does When Running a Hook

### The Hook Invocation in Git Source

When Git runs a hook, it internally executes the system call equivalent of:

```c
// Conceptual (not actual C code):
execve(".git/hooks/pre-commit", { NULL }, environment);
// OR if the hooksPath config is set:
execve(".husky/pre-commit", { NULL }, environment);
```

### What Git Checks Before Running

```
For each hook invocation, Git:

1. Resolves hook path:
   path = core.hooksPath (if configured) + "/" + hook_name
   OR
   path = git_dir + "/hooks/" + hook_name
   
   (In code: find_hook() in run-command.c)

2. Checks if file exists:
   stat(path, &st) → if ENOENT: hook doesn't exist → skip silently

3. Checks if executable:
   st.st_mode & S_IXUSR → if not executable: skip silently (no error)
   
   THIS IS WHY non-executable hooks silently don't run.

4. Forks and execs:
   pid = fork()
   if pid == 0:  // child process
     execve(path, args, env)
   waitpid(pid, &status)

5. Checks exit status:
   WIFEXITED(status) && WEXITSTATUS(status) != 0
   → if non-zero: return error to caller
   → caller decides: abort operation or ignore (post-* hooks)
```

### The Hook Runs in the Repo Root

```bash
# The working directory when the hook runs is ALWAYS the repo root
# (the directory containing .git/)

# PROOF:
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "CWD: $(pwd)"
echo "GIT_DIR: $GIT_DIR"
exit 0
EOF
chmod +x .git/hooks/pre-commit

# Change to a subdirectory:
mkdir -p src/auth && cd src/auth
git commit --allow-empty -m "test"
# Output:
# CWD: /home/alice/myapp     ← repo root, NOT src/auth/
# GIT_DIR: /home/alice/myapp/.git
```

This means you can use relative paths in hooks without worrying about the current directory.

### The `GIT_INDEX_FILE` Environment Variable

Git sets `GIT_INDEX_FILE` to point to the current index file. In a normal commit this is `.git/index`. During a stash or worktree operation, it may point elsewhere.

Hooks should use `git diff --cached` (which reads `GIT_INDEX_FILE`) rather than hardcoding `git diff --cached` — they're equivalent but `git diff --cached` is aware of the environment variable.

```bash
# In a hook, these are equivalent:
git diff --cached --name-only
git diff --index HEAD  # also respects GIT_INDEX_FILE
```

### Hook stdout vs stderr

```
stdout: Printed to terminal. Visible to developer.
stderr: Also printed to terminal.

Convention (not enforced by Git):
  - Use stdout for informational messages ("Running ESLint...")
  - Use stderr for error messages ("ERROR: lint failed")

Git itself uses this distinction internally but doesn't enforce it for hooks.
```

---

## 17. Mental Model Checkpoint

Test yourself from memory.

**1.** You create `.git/hooks/pre-commit` with correct shell script content, but it never runs. Name two possible reasons.

**2.** A developer runs `git commit --no-verify`. Which hooks are skipped? Which hooks (if any) still run?

**3.** What is the difference between `pre-receive` (server-side) and `pre-push` (client-side)? Which one is actually enforceable?

**4.** `lint-staged` runs ESLint with `--fix`, which auto-corrects the file on disk. But the committed version still has the unfixed code. What step is missing?

**5.** You configure husky. A colleague clones the repo and runs `npm install`. What happens automatically? What file in the repo makes this happen?

**6.** A `post-commit` hook exits with code 1. The commit has already been created. What happens?

**7.** You want hooks to be shared across all developers, versioned in the repo, and automatically installed. List the two mechanisms Git provides natively (without husky) for this.

**8.** During a `pre-push` hook, how do you determine which branches are being pushed? How do you read this information?

**9.** GitHub does not allow custom server-side `pre-receive` hooks. What GitHub feature is the functional equivalent for enforcing that no direct push to `main` is possible?

**10.** What does `GIT_DIR` contain when a hook runs? What does this tell you about where relative paths resolve?

<details>
<summary>Answers</summary>

**1.** Two possible reasons: (1) The file is not executable — forgot `chmod +x`. Git silently ignores non-executable hooks. (2) The `core.hooksPath` config is set to a different directory (e.g., `.husky/`) and Git is looking there instead of `.git/hooks/`. Also valid: the file has CRLF line endings on Linux/Mac causing the shebang to fail; or the hook file is named with a `.sample` extension.

**2.** `git commit --no-verify` skips `pre-commit` and `commit-msg`. `post-commit` still runs (it cannot be aborted — the commit already exists). Server-side hooks (`pre-receive`, `update`, `post-receive`) are not affected because they run on the remote server.

**3.** `pre-push` runs on the developer's machine before sending data to the remote. It can be bypassed with `git push --no-verify`. `pre-receive` runs on the remote server when it receives the push. It cannot be bypassed by the developer — they have no access to the server. For real enforcement, only server-side hooks (or GitHub's branch protection rules) are reliable.

**4.** After ESLint modifies the file on disk, the modified version is in the working tree but NOT in the index (staging area). The commit captures the index, not the working tree. The missing step is `git add <file>` to re-stage the auto-fixed version. lint-staged handles this automatically by re-staging files after running fix commands.

**5.** When the colleague runs `npm install`, npm executes the `"prepare"` lifecycle script defined in `package.json`. That script is `"husky"` (set by `npx husky init`). Husky then runs `git config core.hooksPath .husky` in the repository. From that point, all git commands look for hooks in `.husky/` instead of `.git/hooks/`. The `"prepare"` key in `package.json` is what makes this happen automatically.

**6.** Nothing. Git ignores the exit code of `post-commit`. The operation (commit creation) has already completed. The commit exists in `.git/objects/` and the branch pointer has already been updated. The only effect of a failing `post-commit` is that the developer sees the error output in their terminal.

**7.** (1) `core.hooksPath` configuration: set it to a tracked directory (e.g., `.githooks/`) and commit that directory. Developers must run `git config core.hooksPath .githooks` once, or it can be set in a repo's `.gitconfig`. (2) A `Makefile` or setup script that developers run once: `make setup` → `git config core.hooksPath .githooks && chmod +x .githooks/*`. Husky automates option 1 via the `prepare` npm lifecycle script.

**8.** The `pre-push` hook receives the remote name and URL as arguments `$1` and `$2`. For each ref being pushed, Git writes one line to the hook's stdin: `<local-ref> <local-sha> <remote-ref> <remote-sha>`. You read these with a `while read local_ref local_sha remote_ref remote_sha; do ... done` loop.

**9.** Branch Protection Rules in GitHub Settings → Branches → Add rule for `main`. Specifically: "Restrict who can push to matching branches" blocks direct pushes; "Require a pull request before merging" prevents bypassing PRs; "Do not allow bypassing the above settings" makes it apply even to repository admins.

**10.** `GIT_DIR` contains the absolute path to the `.git/` directory (e.g., `/home/alice/myapp/.git`). The working directory when the hook runs is always the repo root (the parent of `.git/`). This means relative paths in hooks resolve from the repo root, regardless of which subdirectory the developer was in when they ran the git command.

</details>

---

## 18. Connect Everything Backwards

| This Topic's Concept | Connects To |
|---------------------|-------------|
| **Hooks live in `.git/hooks/`** | Topic 01 — `.git/hooks/` is one of the directories listed in the complete `.git/` architecture. It was mentioned in Topic 01 as "scripts Git can run automatically." Now we see exactly what those scripts do. |
| **`pre-commit` reads staged content via `git diff --cached`** | Topic 05 — The index (staging area) is what `--cached` reads. `pre-commit` hooks operate on the index, not the working tree. Topic 06 showed exactly how `git add` writes to the index. |
| **`commit-msg` receives the message file path** | Topic 07 — Commits Done Right covered Conventional Commits format. The `commit-msg` hook is the technical enforcement mechanism for the format standards described in Topic 07. |
| **`prepare-commit-msg` auto-populates from branch name** | Topic 08 — Branches are 41-byte files in `.git/refs/heads/`. `git symbolic-ref --short HEAD` reads that file to get the branch name. The hook uses this to extract ticket numbers. |
| **`pre-push` receives stdin with old/new SHAs per ref** | Topic 03 — Refs are SHA pointers. The `pre-push` hook receives the exact same information — old ref SHA and new ref SHA — that you see in `git reflog`. |
| **Server-side `pre-receive` checks force-push** | Topic 11 — Rebase rewrites SHAs. The rebase golden rule from Topic 11 is enforced at the technical level by server-side `pre-receive` hooks checking `git merge-base --is-ancestor old new`. |
| **`core.hooksPath` points to a tracked directory** | Topic 04 — git configuration levels. `core.hooksPath` is a Git config key, set at the local (repo) level. Husky sets it automatically, which is why the `.git/config` file has `hooksPath = .husky` after `npm install`. |
| **lint-staged runs only on staged files** | Topic 05/06 — The three trees again. lint-staged operates on the index (staged files), not the working tree. It uses `git diff --cached --name-only` to enumerate exactly what `git add` placed in the index. |
| **`post-commit` triggers PR creation URL** | Topic 16 — The PR lifecycle. After `git push`, you open a PR. The `post-commit` hook shown in this topic automates that by opening the GitHub compare URL. The full PR lifecycle from Topic 16 starts with this step. |
| **GitHub Actions = server-side `post-receive`** | Topic 10 — Remote repositories. When you push to `origin`, GitHub receives the push, updates its refs, and then fires webhooks. GitHub Actions listens to those webhooks — functionally equivalent to a `post-receive` hook on a self-hosted server. |

---

## 19. Quick Reference Card

### Hook Locations and Invocation

| Hook | When runs | Abortable | Primary use |
|------|----------|-----------|-------------|
| `pre-commit` | Before `git commit` | ✅ | Lint, secret scan, format check |
| `prepare-commit-msg` | Before commit editor | ✅ | Auto-fill message from branch |
| `commit-msg` | After message written | ✅ | Validate message format |
| `post-commit` | After commit created | ❌ | Notify, open PR page |
| `pre-push` | Before `git push` | ✅ | Run test suite, block push to main |
| `post-checkout` | After `git checkout` | ❌ | Rebuild deps on branch switch |
| `post-merge` | After `git merge` | ❌ | Reinstall deps if lockfile changed |
| `pre-rebase` | Before `git rebase` | ✅ | Prevent rebase of published branches |
| `pre-receive` | Push received at server | ✅ | Enforce branch rules server-side |
| `update` | Per-ref at server | ✅ | Per-branch policies at server |
| `post-receive` | After push accepted | ❌ | Trigger CI, deploy, notify |

### Hook Management Commands

| Command | What it does |
|---------|-------------|
| `ls .git/hooks/` | List available hooks (+ samples) |
| `chmod +x .git/hooks/<hook>` | Make a hook executable |
| `git config core.hooksPath <dir>` | Use custom hook directory |
| `git config --global core.hooksPath ~/.git-hooks` | Global custom hook dir |
| `.git/hooks/<hook>` | Run hook directly for testing |

### Bypass Commands

| Command | Skips |
|---------|-------|
| `git commit --no-verify` | `pre-commit`, `commit-msg` |
| `git commit -n` | Same as `--no-verify` |
| `git push --no-verify` | `pre-push` |
| `HUSKY=0 npm install` | Prevents husky setup |
| `HUSKY=0 git commit` | Skips husky hooks |

### Husky Commands

| Command | What it does |
|---------|-------------|
| `npm install --save-dev husky` | Install husky |
| `npx husky init` | Initialize: creates `.husky/`, adds `prepare` script |
| `echo "npx lint-staged" > .husky/pre-commit` | Add pre-commit hook |
| `echo "npx commitlint --edit \$1" > .husky/commit-msg` | Add commit-msg hook |
| `cat .git/config \| grep hooksPath` | Verify husky is configured |

### lint-staged Config Pattern

```json
"lint-staged": {
  "*.{js,ts,jsx,tsx}": ["eslint --fix", "prettier --write"],
  "*.{css,scss}":       ["stylelint --fix", "prettier --write"],
  "*.{json,md,yml}":    ["prettier --write"],
  "*.py":               ["black", "flake8 --max-line-length=120"]
}
```

### Reading Hook Input

```bash
# pre-push: read refs being pushed
while read local_ref local_sha remote_ref remote_sha; do
  echo "Pushing $local_ref → $remote_ref"
done

# pre-receive / post-receive: read pushed refs on server
while read old_sha new_sha ref_name; do
  echo "$ref_name: $old_sha → $new_sha"
done

# commit-msg: read message
MSG=$(cat "$1")
```

### Environment Variables in Hooks

| Variable | Contains |
|----------|---------|
| `GIT_DIR` | Absolute path to `.git/` |
| `GIT_INDEX_FILE` | Path to index file |
| `GIT_AUTHOR_NAME` | Author name from config |
| `GIT_AUTHOR_EMAIL` | Author email from config |
| `GIT_COMMITTER_DATE` | Timestamp (for rebase/amend) |

---

## 20. When Would I Use This At Work?

### Starting a New Project — The Setup Checklist

```bash
# Week 1 of a new project: set up all quality gates at once

# 1. Install toolchain
npm install --save-dev husky lint-staged prettier eslint commitlint \
  @commitlint/config-conventional

# 2. Initialize husky
npx husky init

# 3. Create hooks
echo "npx lint-staged" > .husky/pre-commit
echo "npx commitlint --edit \$1" > .husky/commit-msg
cat > .husky/pre-push << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npm run test:unit
EOF

# 4. Configure lint-staged (in package.json)
# 5. Configure commitlint (.commitlintrc.json)
# 6. Configure ESLint (.eslintrc.json)
# 7. Configure Prettier (.prettierrc)

# 8. Add CI equivalent (can't be bypassed):
# .github/workflows/ci.yml → runs eslint, commitlint, test on every push

# 9. Commit and push setup
git add .
git commit -m "chore: set up code quality toolchain"
git push origin main
```

### Enforcing Ticket Numbers in Every Commit

```bash
# Scenario: Your company requires every commit to reference a Jira ticket.

# .husky/prepare-commit-msg (auto-prepend from branch name):
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2

[ "$COMMIT_SOURCE" = "merge" ] && exit 0
[ "$COMMIT_SOURCE" = "squash" ] && exit 0

BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)
TICKET=$(echo "$BRANCH" | grep -oE '[A-Z]+-[0-9]+' | head -1)
[ -z "$TICKET" ] && exit 0

EXISTING=$(cat "$COMMIT_MSG_FILE")
echo "[$TICKET] $EXISTING" > "$COMMIT_MSG_FILE"

# .husky/commit-msg (validate ticket present):
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

MSG=$(cat "$1")
if ! echo "$MSG" | grep -qE '^\[[A-Z]+-[0-9]+\]'; then
  echo "ERROR: Commit message must include a Jira ticket: [PROJ-123] description"
  exit 1
fi
```

### The Incident Post-Mortem Hook

**After a production incident caused by a secret being committed:**

```bash
# Action item from post-mortem: "Add pre-commit secret detection"

# Install gitleaks (dedicated secret scanner):
# Mac: brew install gitleaks
# Linux: apt install gitleaks OR download binary

cat > .husky/pre-commit << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Secret scanning (gitleaks)
if command -v gitleaks &> /dev/null; then
  gitleaks protect --staged --redact -q
  if [ $? -ne 0 ]; then
    echo "ERROR: Potential secrets detected. See output above."
    echo "If this is a false positive: git commit --no-verify"
    echo "But PLEASE rotate the credential first."
    exit 1
  fi
fi

# Lint staged files
npx lint-staged
EOF

# Also add to CI (gitleaks GitHub Action):
# Uses gitleaks/gitleaks-action@v2 on every push/PR
```

---

*End of Topic 19 — Git Hooks*

**Phase 5 Complete**: Topics 15–19 (Workflows → PRs → Merge Strategies → Conflicts → Hooks) ✅

**Next**: Topic 20 — git reflog: The Safety Net (Phase 6 begins — Advanced & Principal-Level Git)
