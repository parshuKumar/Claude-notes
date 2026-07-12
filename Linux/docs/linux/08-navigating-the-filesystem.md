# 08 — Navigating the Filesystem

## ELI5 — The Simple Analogy

Imagine a **giant office building** with no windows.

- You are standing in one room. That room is your **current working directory (CWD)**. You always have exactly one — you cannot be nowhere.
- Every room has a **door plaque** with its full address: `Floor 3 → East Wing → Room 12`. That's an **absolute path** — it works no matter where you start.
- You can also say "go up one floor, then two doors left." That's a **relative path** — it only makes sense from where you're standing right now.
- `cd` is you **physically walking** into another room. Your body moves. Everything you do afterwards happens from the new room.
- `ls` is you **switching on the light and looking around**: what's in here, who owns it, how big is it, when was it last touched?
- `find` is **hiring a search party**. You give them a starting room and a description ("any red folder touched in the last week, but don't bother searching the basement"), and they walk *every* room below that point and report back.
- The building's **directory index in the lobby** is `locate` — instant answers, but it was printed at 6am and it's now 4pm. Anything that moved since then is wrong.

The whole topic is: knowing where you are, seeing what's around you, and finding what you need without walking every room by hand.

---

## Where This Lives in the Linux Stack

```
Hardware (Disk platters / SSD cells holding directory blocks + inode tables)
  └── KERNEL
       │    • Owns the ONE authoritative copy of each process's CWD
       │      (a struct path in the process's fs_struct — see /proc/PID/cwd)
       │    • Walks path components, checks +x on every directory
       │    • Reads directory data blocks (name → inode number)
       │
       └── System Calls
            │    chdir()   — change the process's CWD          ◀◀◀
            │    getcwd()  — ask the kernel "where am I?"      ◀◀◀
            │    openat()  — open a directory as a file        ◀◀◀
            │    getdents64() — read directory entries         ◀◀◀
            │    statx()/lstat() — get an entry's inode data   ◀◀◀
            │
            └── C Library (glibc wraps chdir/getcwd/opendir/readdir)
                 │
                 └── SHELL (bash/zsh)
                      │    `cd` is a BUILTIN — it must be. See below. ◀◀◀ THIS TOPIC
                      │    `pwd` is a builtin too (with a twist)
                      │    Expands globs BEFORE the command sees them
                      │
                      └── Commands: ls, find, locate, tree, realpath ◀◀◀ THIS TOPIC
```

**This topic straddles two layers**: `cd`/`pwd` live *inside the shell* (topic 07), while `ls`/`find` are ordinary external programs that hammer the kernel's directory syscalls.

---

## What Is This?

Navigating the filesystem means three distinct skills: knowing **where your process currently is** (CWD), **listing what is there** with enough metadata to make a decision, and **searching a subtree** for files matching arbitrary criteria.

The tools are `cd`, `pwd`, `ls`, and `find`. `find` is by far the most important — it is the only one of the four that is a full query language, and 90% of real production filesystem work is a `find` expression.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `cd` is a shell builtin calling `chdir()` | You'll write a deploy script with `cd /app` in a subshell/pipeline, wonder why the parent script is still in the old directory, and deploy to the wrong path |
| `ls -l` column meanings | You'll stare at `-rw-r--r-- 1 root root` on a file your `deploy` user can't write and not know it's a permission problem |
| `ls -ltr` | You'll `ls /var/log` on a box with 400 log files and have no idea which one is the one that just broke |
| `find`'s `-mtime +7` semantics | Your log-cleanup cron will delete the wrong week of logs — or nothing at all |
| `-exec {} \;` vs `-exec {} +` | Your cleanup job will fork 200,000 processes and peg the box for 40 minutes instead of finishing in 3 seconds |
| Shell globbing vs `find` patterns | `find . -name *.log` will silently do the wrong thing or error out and you'll blame `find` |
| `-prune` | Every `find` in a Node repo will spend 90% of its time inside `node_modules` |
| `du`/`find` for big files | Disk hits 100%, Node throws `ENOSPC`, and you have no method for locating the culprit |

---

## The Physical Reality

### What a directory actually is

Remember from **04 — Inodes in Depth**: a directory is *just a file* whose data blocks contain a list of `(filename, inode number)` pairs. The filename lives **in the directory**, never in the inode.

```
Directory /var/www/app  (itself an inode, say inode 262145)
│
└── its DATA BLOCK contains:
    ┌──────────────────┬──────────────┬──────────┐
    │ name             │ inode number │ d_type   │
    ├──────────────────┼──────────────┼──────────┤
    │ "."              │ 262145       │ DT_DIR   │ ← itself
    │ ".."             │ 131073       │ DT_DIR   │ ← parent (/var/www)
    │ "server.js"      │ 262180       │ DT_REG   │
    │ "node_modules"   │ 262200       │ DT_DIR   │
    │ ".env"           │ 262181       │ DT_REG   │
    │ "current"        │ 262190       │ DT_LNK   │ ← symlink
    └──────────────────┴──────────────┴──────────┘
```

**`.` and `..` are real, physical directory entries on disk.** They are not shell magic. `..` is a hard link to the parent directory — this is why `ls -ld /` shows a link count of 2 or more, and why you cannot hard-link directories yourself (you'd create a loop the kernel couldn't unwind).

### Where the CWD actually lives

Your CWD is **kernel state, per-process**. It is not a variable in the shell. The kernel keeps it in the process's `fs_struct`, and exposes it as a **magic symlink** at `/proc/PID/cwd`:

```
Process 4821 (bash)
├── /proc/4821/cwd  → symlink → /var/www/app     ← THE authoritative CWD
├── /proc/4821/root → symlink → /                 ← its root (chroot changes this)
└── /proc/4821/exe  → symlink → /usr/bin/bash

The shell ALSO keeps a string in $PWD.
$PWD is a CONVENIENCE COPY. It can drift from the truth (see pwd -P vs -L).
```

Two paths, one file:

```
ABSOLUTE (starts with /)              RELATIVE (does not start with /)
/var/www/app/server.js                ../app/server.js
│                                     │
└─ kernel starts at the process's     └─ kernel starts at the process's
   ROOT (/proc/PID/root)                 CWD  (/proc/PID/cwd)

Then it walks components left to right, needing +x (search) on EVERY
directory along the way (topic 05 — File Permissions).
```

---

## How It Works — Step by Step

### `cd /var/log` — and why `cd` MUST be a builtin

```
COMMAND: cd /var/log

SHELL DOES:   Recognizes "cd" in its builtin table. Does NOT fork.
              Saves current $PWD into $OLDPWD.
              Calls chdir("/var/log") IN ITS OWN PROCESS.
              On success, updates its internal $PWD string.
KERNEL DOES:  Resolves "/var/log" from the process root.
              Checks +x on / and on /var.
              Checks the target is a directory.
              Replaces the process's fs_struct->pwd with the new path.
SYSCALLS:     chdir("/var/log") = 0
OUTPUT:       nothing (silence = success)
```

**Why it cannot be an external program:** From **07 — How the Shell Works**, you know that running an external command means `fork()` + `execve()`. The child gets a *copy* of the parent's process state. If `/bin/cd` existed and called `chdir()`, it would change the **child's** CWD — and then the child would exit, taking its CWD with it. The parent shell would be exactly where it started. A directory change is only meaningful if it happens in the shell's own process, so `cd` is compiled into bash.

> There *is* a `/usr/bin/cd` on some systems (POSIX requires it). It's a stub that does nothing useful. Run `type cd` — bash tells you `cd is a shell builtin`. The builtin always wins.

### The four navigation shorthands

```
.     → this directory        (a real on-disk entry)
..    → the parent directory  (a real on-disk entry)
~     → $HOME                 (SHELL expansion — the kernel has never heard of ~)
-     → $OLDPWD               (SHELL feature — `cd -` = "go back", and it PRINTS
                               the directory it landed in, which trips up scripts)
```

`~` and `-` are **shell-level**. `.` and `..` are **kernel/filesystem-level**. Prove it:

```bash
# The kernel does not know what ~ means:
ls '~'          # error: cannot access '~': No such file or directory
# Quoting stopped the shell from expanding it, so ls got a literal tilde.
```

### `ls -l /var/log` at the syscall level

```
COMMAND: ls -l /var/log

SHELL DOES:    fork() + execve("/usr/bin/ls", ["ls","-l","/var/log"], env)
KERNEL DOES:   opens the directory, reads its data blocks, then reads ONE
               INODE PER ENTRY to get perms/owner/size/mtime
SYSCALLS:      openat(AT_FDCWD, "/var/log", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
               getdents64(3, ...) = 1272     ← bulk-read the name→inode pairs
               getdents64(3, ...) = 0        ← 0 = no more entries
               statx(3, "syslog", ...)       ← ONE stat per entry (this is the slow part)
               statx(3, "auth.log", ...)
               ... (repeat N times)
               close(3)
               write(1, "total 4820\n-rw-r-----  1 syslog adm ...", 1843)
OUTPUT:        formatted listing to fd 1 (stdout)
```

**This is why `ls -l` on a directory with 50,000 files is slow but plain `ls` is fast.** Plain `ls` (no `-l`, no `--color`, output not a TTY) can get away with only `getdents64()`, because `getdents64` already returns a `d_type` byte telling it file-vs-directory. Add `-l` and it must `stat()` every single entry — N extra syscalls, N extra inode reads.

---

## The `ls -l` Long Format — Every Column, Tied to the Inode

```
-rw-r-----  1  syslog  adm   1048576  Jul 12 04:02  syslog
│└────────┘ │  └────┘  └─┘   └─────┘  └──────────┘  └────┘
│    │      │     │      │       │          │           │
│    │      │     │      │       │          │           └── (7) NAME
│    │      │     │      │       │          │               NOT in the inode!
│    │      │     │      │       │          │               Lives in the DIRECTORY's
│    │      │     │      │       │          │               data block. (topic 04)
│    │      │     │      │       │          │
│    │      │     │      │       │          └── (6) MTIME — inode field i_mtime
│    │      │     │      │       │              Last time the file's CONTENT changed.
│    │      │     │      │       │              (ls -l shows mtime by default,
│    │      │     │      │       │               ls -lu shows atime, ls -lc shows ctime)
│    │      │     │      │       │
│    │      │     │      │       └── (5) SIZE in bytes — inode field i_size
│    │      │     │      │           For a DIRECTORY this is the size of the
│    │      │     │      │           directory's own data block (often 4096),
│    │      │     │      │           NOT the size of its contents.
│    │      │     │      │
│    │      │     │      └── (4) GROUP — inode field i_gid (resolved via /etc/group)
│    │      │     │
│    │      │     └── (3) OWNER — inode field i_uid (resolved via /etc/passwd)
│    │      │         If the user was deleted you see the raw number instead.
│    │      │
│    │      └── (2) HARD LINK COUNT — inode field i_nlink
│    │          How many directory entries point at this inode. (topic 04)
│    │          For a directory: 2 + number of subdirectories
│    │          (itself via ".", its entry in the parent, and one ".." per child)
│    │
│    └── (1b) PERMISSION BITS — the low 9 bits of inode field i_mode (topic 05)
│         rw- r-- ---  =  owner rw, group r, other nothing  =  0640
│
└── (1a) FILE TYPE — the high bits of i_mode (topic 03)
     -  regular file      d  directory        l  symbolic link
     c  character device  b  block device     p  named pipe (FIFO)
     s  socket
```

**Columns 1–6 all come out of the inode. Column 7 does not.** That single sentence is the entire relationship between topic 04 and this one.

The `total 4820` line at the top is **not** bytes — it is the sum of the disk blocks allocated to the listed files, in 1 KiB units by default. It disagrees with the sum of the size column for sparse files and for files with tail-packing.

---

## `pwd` vs `$PWD` — the symlink trap

```bash
$ mkdir -p /var/www/releases/v2
$ ln -s /var/www/releases/v2 /var/www/current
$ cd /var/www/current

$ pwd                    # bash BUILTIN pwd — LOGICAL by default (-L)
/var/www/current         # ← the path you WALKED

$ pwd -P                 # PHYSICAL — resolve every symlink
/var/www/releases/v2     # ← the path the KERNEL actually has

$ echo "$PWD"
/var/www/current         # $PWD is the shell's string, it matches logical pwd

$ /bin/pwd               # the EXTERNAL pwd — it calls getcwd(), it CANNOT lie
/var/www/releases/v2     # ← physical, always
```

```
WHY THEY DIFFER:
  The kernel's CWD is an inode, not a string. getcwd() reconstructs the path by
  walking UP from that inode via ".." — a symlink is not on that chain, so you
  always get the physical path.
  The shell remembers the string you typed and shows you THAT instead, because
  it's usually what you meant.

PROVE IT:
  readlink /proc/$$/cwd     →  /var/www/releases/v2   (the truth)
  echo $PWD                 →  /var/www/current       (the shell's story)
```

**This bites in deploy scripts.** `cd /var/www/current && pwd` gives you a path that will point somewhere *else* after the next release swaps the symlink. Use `pwd -P` (or `realpath .`) when you need the path to stay pinned to the release you're actually in.

---

## `find` — The Expression Model

`find` is not a command with flags. It is a **tiny query language**. Learn the grammar and every flag falls out of it.

```
find  [starting-paths...]  [global-options]  [tests]  [actions]
│      │                    │                 │        │
│      │                    │                 │        └── what to DO with matches
│      │                    │                 │            -print (default), -print0,
│      │                    │                 │            -delete, -exec, -ls, -quit
│      │                    │                 │
│      │                    │                 └── QUESTIONS asked of each file.
│      │                    │                     Each returns true/false.
│      │                    │                     -name -type -mtime -size -user -perm
│      │                    │
│      │                    └── MUST COME FIRST: -maxdepth, -mindepth, -depth,
│      │                        -follow, -xdev. These change the WALK itself.
│      │
│      └── where to start walking (GNU defaults to "." — BSD/macOS REQUIRES it)
└── the binary
```

**The core mental model:** `find` walks the tree, and for every single entry it evaluates the expression left-to-right as a boolean. Tests are joined by an **implicit AND**. If the whole expression is true and you gave no action, `-print` is appended for you.

```
find /var/log -type f -name "*.log" -mtime +30 -delete
│    │        └──────────┬────────────────────┘  └──┬──┘
│    │                 TESTS (implicit AND)       ACTION
│    │                 "is a regular file"
│    │                 AND "name ends in .log"
│    │                 AND "mtime older than 30 days"
│    └── starting path
└── walk everything under here
```

Operators, when you need more than AND:

| Operator | Meaning | Example |
|---|---|---|
| (implicit) or `-a` | AND | `-type f -name "*.js"` |
| `-o` | OR | `\( -name "*.log" -o -name "*.gz" \)` |
| `!` or `-not` | NOT | `! -name "*.js"` |
| `\( ... \)` | grouping — **must be escaped**, `(` is a shell metacharacter | see above |

### The `-mtime` off-by-one — read this twice

This is the single most misunderstood thing in `find`. The rule:

```
age_in_days = floor( (now - file_mtime) / 86400 )     ← TRUNCATED, not rounded

-mtime  N   →  age_in_days == N     (exactly N: between N and N+1 days old)
-mtime +N   →  age_in_days >  N     (MORE than N days old)
-mtime -N   →  age_in_days <  N     (LESS than N days old — i.e. RECENT)
```

Work an example. It is 12:00 today.

| File last modified | Exact age | floor(age) | `-mtime 7`? | `-mtime +7`? | `-mtime -7`? |
|---|---|---|---|---|---|
| 3 hours ago | 0.125 d | 0 | no | no | **YES** |
| 6.5 days ago | 6.5 d | 6 | no | no | **YES** |
| 7.5 days ago | 7.5 d | 7 | **YES** | no | no |
| 8.5 days ago | 8.5 d | 8 | no | **YES** | no |
| 30 days ago | 30.0 d | 30 | no | **YES** | no |

**The trap:** because of the truncation, `-mtime +7` does **not** mean "older than 7 days." It means "at least 8 full days old." A file modified 7 days and 12 hours ago is skipped by *both* `+7` and `-7`. If you need true "older than 7 days," use `-mtime +6`, or better, sidestep the whole thing with GNU's `-newermt`:

```bash
# Unambiguous, no truncation games (GNU find; also works on macOS/BSD find):
find /var/log -type f ! -newermt '7 days ago' -print   # strictly older than 7 days
find /var/log -type f   -newermt '10 minutes ago'      # touched in last 10 min
```

The three time tests, and the one people get wrong:

| Test | Inode field | What it means |
|---|---|---|
| `-mtime` / `-mmin` | `i_mtime` | file **contents** last changed |
| `-atime` / `-amin` | `i_atime` | file last **read** (often unreliable — most servers mount with `relatime`, so atime only updates once a day) |
| `-ctime` / `-cmin` | `i_ctime` | **inode** last changed — chmod, chown, rename, link count change. **NOT "creation time."** Nothing in POSIX gives you creation time; ext4 stores a birth time but `find` cannot test it. |

### `-size` — the other unit trap

```
find /var -size +100M
             │  │   │
             │  │   └── UNIT: c=bytes  k=KiB  M=MiB  G=GiB
             │  │       If you OMIT the unit, the default is `b` = 512-BYTE BLOCKS.
             │  │       So `-size +100` means "more than 100 blocks" = 51,200 bytes.
             │  │       ALWAYS write the unit.
             │  └── +  = greater than    - = less than    (none) = exactly
             └── test on i_size

Also: GNU find ROUNDS UP to the unit. A 1-byte file is "-size 1M" true, because
1 byte rounds up to 1 MiB. This only surprises you with exact matches; +/- are fine.
```

### `-maxdepth` must come first

```bash
# WRONG — works, but prints a warning and behaves unexpectedly:
find . -name "*.js" -maxdepth 2
# find: warning: you have specified the global option -maxdepth after the
# argument -name, but global options are not positional...

# RIGHT:
find . -maxdepth 2 -name "*.js"
```

**Root cause:** `-maxdepth` is a *global option* — it configures the tree walk itself, which has already been decided by the time `find` reaches it in the expression. Putting it late makes the code read as if it were a test, which it isn't. `-mindepth 1` is how you exclude the starting directory itself from results.

### `-prune` — the one that skips `node_modules`

```bash
find . -name node_modules -prune -o -type f -name "*.js" -print
│      └──────────┬─────┘  └──┬─┘ │  └──────────┬──────┘ └──┬─┘
│                 │           │   │             │           │
│                 │           │   │             │           └── explicit -print is
│                 │           │   │             │               MANDATORY here
│                 │           │   │             └── the tests you actually want
│                 │           │   └── OR
│                 │           └── "do not descend into this directory".
│                 │               -prune returns TRUE, so the left side of -o
│                 │               succeeds and the right side is never evaluated
│                 │               → the pruned dir is not printed either.
│                 └── match the dirs to skip
└── start here
```

**Why the explicit `-print` is required:** if you leave it off, `find` appends `-print` to the *whole* expression, which becomes `\( ...prune... -o ...tests... \) -print` — and now the pruned directories get printed too. The rule: **the moment you use `-prune -o`, you must write `-print` yourself.**

---

## Exact Syntax Breakdown

```
ls -lath /var/log
│  ││││  │
│  ││││  └── path to list (omit = current directory)
│  ││││
│  │││└── -h  human-readable sizes (4.0K, 1.2M, 3.4G) — ONLY works with -l or -s
│  ││└── -t  sort by MTIME, newest FIRST. The most useful ls flag that exists.
│  │└── -a  show ALL entries including dotfiles, AND "." and ".."
│  └── -l  long format (the 7 columns above)
└── ls = "list"

Two idioms worth burning into muscle memory:

  ls -lath           "what changed here most recently?"  newest at TOP
  ls -ltr            "what changed here most recently?"  newest at BOTTOM
                     └── -r reverses the -t sort: oldest→newest.
                         The newest file ends up on the line RIGHT ABOVE YOUR
                         PROMPT — no scrolling. This is THE log-directory idiom.
```

```
ls -d /var/log
│  │
│  └── -d  list the DIRECTORY ITSELF, not its contents.
│          Without -d, `ls -l /var/log` lists what's INSIDE.
│          With    -d, `ls -ld /var/log` shows ONE line: /var/log's own
│          permissions, owner, and link count.
│          You need this EVERY time you debug "why can't my app write here"
│          — because the answer is in the directory's OWN +w and +x bits.
└── ls

  ls -ld /var/www/app          ← the ONE command for "who owns this directory"
  ls -ld */                    ← long-list every subdirectory, not their contents
```

```
find /var/www -maxdepth 3 -type f -name "*.log" -mtime +7 -exec gzip {} +
│    │        └────┬────┘ └──┬──┘ └──────┬────┘ └───┬───┘ └───────┬─────┘
│    │             │         │           │          │             │
│    │             │         │           │          │             └── ACTION: run gzip,
│    │             │         │           │          │                 BATCHING as many
│    │             │         │           │          │                 filenames as fit on
│    │             │         │           │          │                 one command line
│    │             │         │           │          └── TEST: modified 8+ days ago
│    │             │         │           └── TEST: name glob (QUOTE IT — see below)
│    │             │         └── TEST: regular file only (not dirs, not symlinks)
│    │             └── GLOBAL OPT: don't descend more than 3 levels. Must be first.
│    └── starting path
└── find

  -type f  regular file    -type d  directory    -type l  symlink
  -type s  socket          -type p  named pipe   -type b/c  block/char device
```

```
find . -type f -print0 | xargs -0 grep -l "DATABASE_URL"
│                │        │     │
│                │        │     └── -0: expect NUL-separated input
│                │        └── xargs: build command lines from stdin, batched
│                └── -print0: separate results with a NUL byte (\0) instead of \n
│                    NUL is the ONLY byte that cannot appear in a filename.
│                    Newlines CAN appear in filenames. So plain `find | xargs`
│                    is broken for any filename containing a space or newline.
└── find

RULE: if a pipeline touches filenames you did not create yourself,
      it is -print0 | xargs -0, or it is a bug waiting to happen.
```

### `-exec ... \;` vs `-exec ... +` — the performance difference that matters

```
-exec rm {} \;                          -exec rm {} +
│           │                           │          │
│           └── \; ends the command      │          └── + ends the command,
│               ONE EXEC PER FILE        │              BATCH as many {} as fit
└── {} = the current filename            └── {} must be the LAST argument

WHAT THE KERNEL SEES:

  \;  →  fork()+execve("/bin/rm", ["rm","a.log"])      ← 200,000 times
         fork()+execve("/bin/rm", ["rm","b.log"])         ~200,000 fork+exec pairs
         fork()+execve("/bin/rm", ["rm","c.log"])         ≈ 40 minutes, 100% CPU
         ...

  +   →  fork()+execve("/bin/rm", ["rm","a.log","b.log", ... ~5000 names])
         fork()+execve("/bin/rm", ["rm", ... next ~5000 names])
                                                         ← ~40 execs total
                                                            ≈ 3 seconds

The batch size is bounded by ARG_MAX (getconf ARG_MAX → usually 2097152 bytes).
find handles the chunking for you.

USE \; ONLY WHEN: the command takes exactly one file, or {} is not the last arg,
                  e.g.  -exec mv {} /backup/ \;
                  (here {} isn't last, so + is illegal)
```

### QUOTING THE PATTERN — why `find . -name *.log` breaks

This is a direct callback to **07 — How the Shell Works**: *the shell expands globs before the command ever runs.*

```
YOU TYPE:      find . -name *.log
                             │
SHELL DOES:    sees an unquoted glob. Expands it AGAINST THE CURRENT DIRECTORY.
               Whatever find was going to do is irrelevant — find has not started yet.

CASE A — zero matches in CWD (bash default, nullglob off):
   shell passes the literal *.log through untouched → find works by ACCIDENT.
   This is why it "works on my machine."

CASE B — exactly ONE match in CWD (say access.log):
   find . -name access.log
   → find silently searches for ONLY access.log. Your other .log files are
     never found. NO ERROR. This is the dangerous case.

CASE C — MULTIPLE matches in CWD (access.log, error.log):
   find . -name access.log error.log
   → find: paths must precede expression: 'error.log'

FIX: quote it, so the shell hands the pattern to find intact.
   find . -name '*.log'        ← single quotes: best
   find . -name "*.log"        ← double quotes: also fine
   find . -name \*.log         ← escaping: fine, ugly
```

---

## Example 1 — Basic

```bash
# Where am I? Ask the kernel, not the shell.
pwd
# /home/deploy

# PROVE the kernel is the source of truth:
readlink /proc/$$/cwd
# /home/deploy          ← $$ is your shell's PID. Same answer. Same fact.

# Make a playground
mkdir -p /tmp/nav/{app,logs,node_modules/express}
cd /tmp/nav

# Create files with different ages
touch app/server.js app/.env
touch logs/access.log logs/error.log
touch -d '10 days ago' logs/old.log          # backdate the mtime
touch node_modules/express/index.js

# List everything, long format, human sizes, newest first
ls -lath
# total 20K
# drwxrwxrwt 12 root   root   4.0K Jul 12 14:31 ..
# drwxr-xr-x  5 deploy deploy 4.0K Jul 12 14:31 .
# drwxr-xr-x  2 deploy deploy 4.0K Jul 12 14:31 app
# drwxr-xr-x  2 deploy deploy 4.0K Jul 12 14:31 logs
# drwxr-xr-x  3 deploy deploy 4.0K Jul 12 14:31 node_modules

# -a shows . and ..   |   -A shows dotfiles but HIDES . and ..
ls -a  app     # .  ..  .env  server.js
ls -A  app     # .env  server.js          ← -A is what you almost always want

# The directory ITSELF, not its contents:
ls -ld logs
# drwxr-xr-x 2 deploy deploy 4096 Jul 12 14:31 logs

# Inode numbers (topic 04):
ls -li app
# 1310725 -rw-r--r-- 1 deploy deploy 0 Jul 12 14:31 .env      ← -i prepends the inode
# 1310726 -rw-r--r-- 1 deploy deploy 0 Jul 12 14:31 server.js

# find: every .log file, skipping node_modules
find . -name node_modules -prune -o -type f -name '*.log' -print
# ./logs/access.log
# ./logs/error.log
# ./logs/old.log

# Only the OLD one (8+ full days):
find . -type f -name '*.log' -mtime +7
# ./logs/old.log        ← 10 days old. access.log/error.log are 0 days old.

# `cd -` bounces you back to where you were (and prints where it landed)
cd /var/log
cd -
# /tmp/nav
```

---

## Example 2 — Production Scenario

**Situation:** 02:14. PagerDuty fires. Your Node API on `api-prod-01` is returning 500s. The container logs say `ENOSPC: no space left on device, write`. You SSH in.

```bash
deploy@api-prod-01:~$ df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1   40G   40G     0 100% /
#                                  ^^ ← zero bytes free. Node cannot write. 500s.

# Step 1 — WHERE is the weight? Walk one level at a time, don't guess.
deploy@api-prod-01:~$ sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -5
# 1.1G    /usr
# 2.3G    /home
# 4.8G    /opt
# 31G     /var          ← there it is
# 40G     /

deploy@api-prod-01:~$ sudo du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail -3
# 512M    /var/lib
# 30G     /var/log      ← logs
# 31G     /var

# Step 2 — WHICH files, exactly? find with a size test + a batched exec.
deploy@api-prod-01:~$ sudo find /var/log -xdev -type f -size +500M -exec ls -lh {} +
# -rw-r----- 1 syslog  adm    2.1G Jul 12 02:14 /var/log/syslog
# -rw-r--r-- 1 deploy  deploy  27G Jul 12 02:14 /var/log/app/api-error.log
#                              ^^^ ← 27 GB. Found it in one command.

# Step 3 — WHY is it 27G? Look at what's being written, right now.
deploy@api-prod-01:~$ sudo tail -2 /var/log/app/api-error.log
# {"level":"error","msg":"ECONNREFUSED 10.0.3.14:5432","ts":"2026-07-12T02:14:59Z"}
# {"level":"error","msg":"ECONNREFUSED 10.0.3.14:5432","ts":"2026-07-12T02:14:59Z"}
# ← Postgres went away, the app is retry-looping and logging every failure.
#   30 GB of the same line. The disk is a symptom; the DB is the cause.

# Step 4 — Is logrotate even configured for this file? Look at recent changes.
deploy@api-prod-01:~$ ls -ltr /etc/logrotate.d/
# -rw-r--r-- 1 root root  120 Mar 04  2024 apt
# -rw-r--r-- 1 root root  173 Mar 04  2024 dpkg
# -rw-r--r-- 1 root root  501 Jan 09 11:02 nginx
# -rw-r--r-- 1 root root  235 Jun 30 16:44 rsyslog
#   ← NO entry for /var/log/app. Nothing has ever rotated it. Root cause #2.

# Step 5 — Buy back space NOW without restarting Node.
#   DO NOT `rm` an open log: unlink() removes the NAME, but Node still holds
#   the file descriptor, so the kernel will NOT free the blocks. df stays 100%.
#   (Full explanation in topic 09.) Truncate instead — same inode, zero bytes:
deploy@api-prod-01:~$ sudo truncate -s 0 /var/log/app/api-error.log

deploy@api-prod-01:~$ df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1   40G   13G   26G  34% /
#                                   ^^^ ← 26G back. API stops 500ing.

# Step 6 — Sweep the historical junk, safely. DRY RUN FIRST. Always.
deploy@api-prod-01:~$ sudo find /var/log -xdev -type f -name '*.gz' -mtime +30 -print | head
# /var/log/app/api-error.log.14.gz
# /var/log/nginx/access.log.22.gz
# ... (looks right — only rotated archives, nothing live)

deploy@api-prod-01:~$ sudo find /var/log -xdev -type f -name '*.gz' -mtime +30 -delete
#                                        ^^^^^ stay on THIS filesystem — never
#                                              wander into a mounted NFS/EFS share

# Step 7 — Which config files did someone touch during the change window?
deploy@api-prod-01:~$ sudo find /etc -xdev -type f -mmin -180 -ls
#  262401  4 -rw-r--r-- 1 root root  501 Jul 12 00:47 /etc/nginx/sites-enabled/api
#   ← someone edited the nginx config 90 minutes before the incident. Not a coincidence.
```

**What you just did:** used `du` to localize, `find -size` to identify, `ls -ltr` to spot the missing config, `truncate` to recover, and `find -mmin` to correlate the incident with a human change. That is the entire loop.

---

## Common Mistakes

### Mistake 1 — `find . -name *.log` (the unquoted glob)

**Wrong:** `find /var/log -name *.log -delete`

**Root cause (shell level, before find runs):** Per **07 — How the Shell Works**, bash performs pathname expansion on unquoted words. If your CWD happens to contain `deploy.log`, bash rewrites your command to `find /var/log -name deploy.log -delete`. `find` never sees a wildcard. It deletes `/var/log/deploy.log` and nothing else — **silently**. You believe you cleaned 40 files. You cleaned 1.

**What the broken state looks like:** no error, no output, disk still full.

**Diagnose:** put `echo` in front. `echo find /var/log -name *.log` shows you the *actual* command bash built.

**Fix:** `find /var/log -name '*.log' -delete`

**Prevention:** **Every `-name` argument gets single quotes. No exceptions.** Make it a reflex, not a decision.

---

### Mistake 2 — `-exec ... \;` on a large tree

**Wrong:** `find /var/log -type f -name '*.gz' -mtime +30 -exec rm {} \;`

**Root cause (kernel level):** `\;` means one `fork()` + `execve()` **per file**. Each pair costs a page-table copy, a new process slot, a dynamic-link of `/bin/rm`, and a scheduler round-trip. On 200,000 files that is 400,000 syscalls of pure process-creation overhead, and it will saturate a CPU core for tens of minutes — on a box that is already unhealthy.

**What the broken state looks like:** `top` shows load average climbing, `rm` flickering in and out of the process list, your cleanup "hanging."

**Fix:**
```bash
find /var/log -type f -name '*.gz' -mtime +30 -delete       # best: no exec at all
find /var/log -type f -name '*.gz' -mtime +30 -exec rm {} + # good: batched
```
`-delete` calls `unlink()` from inside `find` itself — **zero** forks.

**Prevention:** default to `+`. Only reach for `\;` when `{}` cannot be the last argument.

---

### Mistake 3 — `cd` inside a pipeline or `$( )` and expecting it to stick

**Wrong:**
```bash
cat deploy.conf | while read dir; do cd "$dir"; npm ci; done
cd /var/www && ls | ( cd /tmp; pwd )
pwd    # ← still /home/deploy. What happened?
```

**Root cause:** Every stage of a pipeline runs in a **subshell** — a `fork()`ed child. Per **07**, `cd` calls `chdir()`, which modifies **that child's** `fs_struct`. When the subshell exits, its CWD dies with it. The parent's `/proc/PID/cwd` was never touched.

**Diagnose:**
```bash
echo $$                                   # parent shell PID, e.g. 4821
( cd /tmp; echo "subshell $BASHPID: $(readlink /proc/$BASHPID/cwd)" )
readlink /proc/4821/cwd                   # parent never moved
```

**Fix:** don't `cd` in a subshell and expect it to persist. Either use absolute paths, or use the subshell *deliberately* as scoping:
```bash
( cd /var/www/app && npm ci )   # ← GOOD: the ( ) scopes the cd. Parent unaffected.
                                #   This is a feature, not a bug — use it on purpose.
```

**Prevention:** in scripts, prefer `cd /path || exit 1` at the top level, and always guard it — an unchecked `cd` that fails means every following command runs **in the wrong directory**.

---

### Mistake 4 — `which node` and trusting the answer

**Wrong:** debugging "wrong Node version" with `which node`.

**Root cause:** `which` is an **external program**. It gets a copy of your `PATH` and scans it for an executable. It has **no idea** your shell has an alias, a function, or a hashed path — because those live *inside* the shell process and are never exported to a child.

```bash
$ alias node='/opt/node20/bin/node'
$ which node
/usr/bin/node                # ← A LIE. This is not what runs.

$ type node
node is aliased to `/opt/node20/bin/node'   # ← THE TRUTH

$ command -v node
alias node='/opt/node20/bin/node'           # ← also the truth, and POSIX-portable
```

| Tool | Is it a shell builtin? | Sees aliases/functions/builtins? | Verdict |
|---|---|---|---|
| `which` | No (external `/usr/bin/which`) | **No** | Avoid. Lies to you. |
| `whereis` | No | No — searches *hard-coded* dirs, ignores `PATH` entirely. Finds binary + man page + source. | Fine for "where's the man page" |
| `type` | **Yes** (bash builtin) | **Yes** | **Use this interactively.** `type -a node` shows *every* match, in resolution order. |
| `command -v` | **Yes** (POSIX builtin) | **Yes** | **Use this in scripts.** Portable; exit code 0/1 makes it a clean `if`. |

**Prevention:** `if command -v pm2 >/dev/null 2>&1; then ...` — that is the correct "is this installed?" idiom in any script.

---

### Mistake 5 — Trusting `locate` during an incident

**Wrong:** `locate app.log` at 2am and concluding the file doesn't exist.

**Root cause:** `locate` never touches the filesystem. It queries a **pre-built database** (`/var/lib/plocate/plocate.db` on Ubuntu 22.04+, `/var/lib/mlocate/mlocate.db` on older) that is rebuilt by a **daily cron/systemd timer** running `updatedb`. A file created 10 minutes ago is invisible. A file deleted 10 minutes ago is still listed. Also, `updatedb.conf` has `PRUNEPATHS` — `/tmp`, `/var/spool`, and network mounts are excluded on purpose.

**What the broken state looks like:** `locate` returns a path; `ls` on it says "No such file or directory."

**Fix:** `sudo updatedb` to refresh (slow — it walks the whole tree), or just use `find`, which reads the live filesystem.

**Prevention:** `locate` is for "where is that config file I installed last month." `find` is for anything happening **now**. During an incident, always `find`.

---

### Mistake 6 — Forgetting `-print` after `-prune -o`

**Wrong:** `find . -name node_modules -prune -o -name '*.js'`

**Root cause:** with no action given, `find` wraps the **entire** expression in an implicit `-print`, which effectively yields `\( -name node_modules -prune -o -name '*.js' \) -print`. Since `-prune` returns *true*, the `node_modules` directories themselves match the left branch — and get printed.

**Right:** `find . -name node_modules -prune -o -name '*.js' -print`

The explicit `-print` binds only to the right-hand branch of the `-o`.

---

## Hands-On Proof

```bash
# PROVE IT: /proc/PID/cwd is a real symlink to your process's CWD
cd /var/log
ls -l /proc/$$/cwd
# lrwxrwxrwx 1 deploy deploy 0 Jul 12 14:40 /proc/4821/cwd -> /var/log
#  ^ the leading "l" — it IS a symlink, the kernel synthesizes it on read

# PROVE IT: the CWD is kernel state, and cd changes it
readlink /proc/$$/cwd      # /var/log
cd /tmp
readlink /proc/$$/cwd      # /tmp   ← the kernel's copy moved. This is chdir().

# PROVE IT: cd is a builtin, and it MUST be
type cd                    # cd is a shell builtin
type -a pwd                # pwd is a shell builtin / pwd is /usr/bin/pwd  ← BOTH exist

# PROVE IT: an external command cannot change YOUR directory
( cd /etc ) ; pwd          # still /tmp. The subshell's chdir() died with it.

# PROVE IT: pwd -L lies, pwd -P and getcwd() cannot
sudo mkdir -p /tmp/real && sudo ln -sfn /tmp/real /tmp/link
cd /tmp/link
pwd                        # /tmp/link      (logical — the shell's string)
pwd -P                     # /tmp/real      (physical — resolved)
/bin/pwd                   # /tmp/real      (external pwd calls getcwd(): can't lie)
readlink /proc/$$/cwd      # /tmp/real      (the kernel's truth)
echo "$PWD"                # /tmp/link      (just a shell variable)

# PROVE IT: ls -l issues one stat() per entry, plain ls does not
strace -c -e trace=openat,getdents64,statx,newfstatat ls /tmp    >/dev/null
strace -c -e trace=openat,getdents64,statx,newfstatat ls -l /tmp >/dev/null
# Compare the stat counts. -l explodes it. THAT is why -l is slow on huge dirs.

# PROVE IT: getdents64 is what actually reads a directory
strace -e trace=getdents64 ls /etc 2>&1 | head -3
# getdents64(3, 0x55d3..., 32768) = 2712
# getdents64(3, 0x55d3..., 32768) = 0        ← 0 means "no more entries"

# PROVE IT: "." and ".." are real on-disk entries, not shell magic
ls -ai /tmp | head -3
#  786433 .        ← the inode of /tmp itself
#       2 ..       ← the inode of / (inode 2 is the root of most ext4 filesystems)

# PROVE IT: the shell expands globs BEFORE find runs
cd /var/log && touch zz.log
echo find . -name *.log        # watch bash rewrite your command in front of you
echo find . -name '*.log'      # quoted — passed through intact
rm zz.log

# PROVE IT: -exec \; forks per file; -exec + does not
mkdir -p /tmp/forkdemo && touch /tmp/forkdemo/f{1..200}
strace -f -e trace=execve find /tmp/forkdemo -type f -exec true {} \; 2>&1 | grep -c execve
# ~201   ← one execve PER FILE
strace -f -e trace=execve find /tmp/forkdemo -type f -exec true {} +  2>&1 | grep -c execve
# ~2     ← ONE batched execve. This is the whole argument.

# PROVE IT: realpath / readlink -f resolve every symlink to the physical path
realpath /tmp/link          # /tmp/real
readlink -f /tmp/link       # /tmp/real
readlink   /tmp/link        # /tmp/real   (one level only — note the difference)
```

---

## Practice Exercises

### Exercise 1 — Easy

In your terminal:

```bash
mkdir -p /tmp/ex1/{a/b/c,logs}
cd /tmp/ex1/a/b/c
```

1. Print your CWD three ways: the shell builtin, the external binary, and by reading `/proc`.
2. Get back to `/tmp/ex1` using **only** `..` (a relative path).
3. Now `cd /var/log`, then return to `/tmp/ex1` with a **single two-character command**.
4. List `/tmp/ex1` showing inode numbers, dotfiles (but not `.` and `..`), and the directory's own line.

Paste your terminal output.

---

### Exercise 2 — Medium

```bash
mkdir -p /tmp/ex2/{src,node_modules/lodash,dist}
touch /tmp/ex2/src/{index.js,util.js} /tmp/ex2/node_modules/lodash/index.js
touch /tmp/ex2/dist/bundle.js /tmp/ex2/.env
touch -d '45 days ago' /tmp/ex2/dist/old-bundle.js
cd /tmp/ex2
```

Write a **single** `find` command for each:

1. Every `.js` file, **excluding** anything under `node_modules`.
2. Every file modified **more than 30 full days ago** — then prove your `-mtime` number is right by re-running with `-newermt`.
3. Every `.env` file anywhere under `/tmp/ex2`, no matter how deep.
4. Every **empty** directory.
5. Every regular file **not** named `*.js`, printed NUL-separated and piped to `xargs -0 ls -l`.

Then break it on purpose: run `find . -name *.js` (unquoted) from a directory containing exactly one `.js` file and explain, in one sentence, what bash handed to `find`.

---

### Exercise 3 — Hard (Production Simulation)

You are on-call. `/` is at 97%. Node is throwing `ENOSPC`. Build a **single triage script** at `/tmp/triage.sh` that prints, in order:

1. Disk usage of every mounted filesystem, human-readable.
2. The **top 10 largest directories** under `/var`, one level deep, sorted, staying on one filesystem.
3. The **top 20 largest files** anywhere under `/var`, with sizes — **batched exec only**, no `\;`.
4. Every file under `/var/log` **older than 30 days** whose name ends in `.gz` or `.1` — a **dry run**, printing only.
5. Every file **anywhere** modified in the **last 15 minutes**, excluding `/proc`, `/sys`, and `/run`.
6. Every `node_modules` directory under `/opt` and `/home`, **pruned** so `find` never descends into them.

Constraints:
- Every `find` must use `-xdev`.
- Every glob pattern must be quoted.
- Nothing in the script may delete anything.
- Step 3 must complete in under 5 seconds on a box with 2M files — prove it with `time`.

Run it. Paste the output and the `time` result.

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the section.

1. **Why can `cd` not be an external program?** What syscall does it call, and in whose process?
2. `ls -l` prints 7 columns. **Which six come from the inode, and which one does not — and where does that one live instead?**
3. You `cd /var/www/current` where `current` is a symlink to `/var/www/releases/v2`. **What does `pwd` print? What does `pwd -P` print? What does `readlink /proc/$$/cwd` print — and why is that last one the only one that cannot be wrong?**
4. A file was modified **7 days and 12 hours ago**. Does `-mtime +7` match it? Does `-mtime -7`? Does `-mtime 7`? **Why?**
5. **What is the difference between `-exec cmd {} \;` and `-exec cmd {} +` at the syscall level**, and roughly how much faster is `+` on 100,000 files?
6. You run `find . -name *.log` in a directory containing exactly one file, `app.log`. **What command does `find` actually receive, and why is this the most dangerous of the three possible outcomes?**
7. **Why does `which node` sometimes lie, and which two commands tell the truth** — one for interactive use, one for scripts?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `cd` | Change CWD (**builtin** → `chdir()`) | `cd -` previous dir, `cd` alone → `$HOME`, `cd ..` up one |
| `pwd` | Print CWD | `-L` logical/default (follows the path you walked), `-P` physical (resolves symlinks) |
| `ls` | List directory contents | `-l` long, `-a` all incl. `.`/`..`, `-A` all excl. `.`/`..`, `-h` human sizes, `-t` sort by mtime, `-r` reverse, `-S` sort by size, `-i` inode, `-d` **the dir itself**, `-R` recursive, `-F` type suffixes, `--color=auto` |
| | **Idioms:** | `ls -lath` (newest at top) · `ls -ltr` (newest at bottom, next to your prompt) · `ls -ld dir` (the dir's own perms) · `ls -lS` (biggest first) |
| `find` | Walk a tree, test each entry, act | **Tests:** `-name`/`-iname`/`-path`, `-type f\|d\|l`, `-mtime`/`-mmin`/`-ctime`/`-atime`, `-size +100M`, `-user`/`-group`/`-perm`, `-empty`, `-newer`/`-newermt` |
| | | **Global (first!):** `-maxdepth N`, `-mindepth N`, `-xdev` |
| | | **Control:** `-prune`, `-o`, `!`, `\( \)` |
| | | **Actions:** `-print`, `-print0`, `-delete`, `-ls`, `-exec cmd {} +` (batched), `-exec cmd {} \;` (per-file) |
| `locate` | Search a **prebuilt DB** — instant but **stale** | `-i` case-insensitive, `-e` only show files that still exist |
| `updatedb` | Rebuild the `locate` database | run as root; config in `/etc/updatedb.conf` (`PRUNEPATHS`) |
| `type` | **Builtin** — what will this name actually run? | `-a` show **all** matches in order. **The correct tool.** |
| `command -v` | **Builtin**, POSIX — same, script-friendly exit code | The correct tool **in scripts** |
| `which` | External — scans `PATH` only. **Blind to aliases/functions.** | Avoid |
| `whereis` | Binary + source + man page in **hard-coded** dirs | `-b` binary only |
| `tree` | Recursive tree view | `-L 2` depth, `-d` dirs only, `-a` all, `-I 'node_modules'` ignore |
| `realpath` | Canonical absolute path (all symlinks resolved) | `-s` don't resolve symlinks |
| `readlink -f` | Same as `realpath` | bare `readlink` = **one** level only |
| `basename` | Strip the directory part | `basename /a/b/c.js .js` → `c` |
| `dirname` | Strip the filename part | `dirname /a/b/c.js` → `/a/b` |
| `xargs` | Build command lines from stdin | `-0` NUL-separated (**always pair with `-print0`**), `-n1`, `-P4` parallel |

### macOS/BSD vs GNU (you develop on macOS, you deploy to Linux)

| Thing | GNU (Ubuntu/Debian) | BSD (macOS) | Alpine (BusyBox) |
|---|---|---|---|
| `find` with no path | `find -name x` works (defaults to `.`) | **Error** — path is required: `find . -name x` | works |
| `-printf '%s %p\n'` | ✅ | ❌ **does not exist** — use `-exec stat -f ...` or `-ls` | ❌ |
| `-delete`, `-print0`, `-exec +` | ✅ | ✅ | ✅ |
| `-regex` | Emacs regex by default | needs `find -E .` for extended regex | limited |
| `-mtime +N` truncation | same | same | same |
| `ls --color` | ✅ | ❌ use `ls -G` or `export CLICOLOR=1` | ✅ |
| `ls --full-time` | ✅ | ❌ use `ls -lT` | ❌ |
| `stat -c '%i'` | ✅ | ❌ use `stat -f '%i'` | ✅ |
| `locate` | `plocate`/`mlocate` | needs `sudo /usr/libexec/locate.updatedb` | not installed |

**The safe move:** `brew install findutils coreutils` on your Mac, then use `gfind`, `gls`, `gstat` locally so your muscle memory matches production.

---

## When Would I Use This at Work?

### Scenario 1: "Which config did the last deploy actually change?"
A release goes out, the app starts 502'ing. You need to know what moved. `find /etc/nginx /var/www/app -xdev -type f -mmin -30 -ls` gives you every file touched in the last 30 minutes, with owner, size, and timestamp — in one line. Then `ls -ltr /var/www/releases/` shows you the release directories in deploy order, newest sitting right above your prompt, so you know exactly which one to roll back to.

### Scenario 2: "The Docker image is 2.4 GB and it should be 300 MB."
You exec into the container: `docker exec -it api sh`. Then `find / -xdev -type f -size +20M -exec ls -lh {} + 2>/dev/null | sort -k5 -h | tail -20`. You discover a 900 MB `.npm` cache and a 600 MB `node_modules` full of dev dependencies that `npm ci --omit=dev` should have stripped. `-xdev` keeps you out of the mounted volumes; `-exec +` means it finishes in seconds instead of forking 40,000 times inside a CPU-limited container.

### Scenario 3: "Are there secrets checked into the deploy tree?"
Security asks you to audit. `find /var/www -name node_modules -prune -o -type f \( -name '.env*' -o -name '*.pem' -o -name 'id_rsa*' \) -print` — pruning `node_modules` cuts the walk from 4 minutes to 2 seconds, the grouped `-o` catches all three patterns in one pass, and the explicit `-print` (which you now know is mandatory after `-prune -o`) makes sure you get the files and not the pruned directories. Pipe it to `xargs -0 ls -l` with `-print0` if any path might contain a space.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 02 — The Filesystem Hierarchy | You must know what `/var`, `/etc`, `/proc` *are* before navigating them |
| **Builds on** | 03 — Everything Is a File | A directory is a file; `ls -F`/`-l` type characters (`d l s p`) come straight from here |
| **Builds on** | 04 — Inodes in Depth | Six of `ls -l`'s seven columns are literally inode fields; `ls -i` prints the inode number; `find -inum` searches by it |
| **Builds on** | 05 — File Permissions | You need `+x` on **every** directory in a path to traverse it — that is why `cd` and `find` fail with "Permission denied" |
| **Builds on** | 07 — How the Shell Works | `cd` is a builtin because of fork/exec; unquoted `*.log` breaks `find` because of glob expansion; `which` lies because aliases don't cross the fork |
| **Next** | 09 — Working with Files | Now that you can *find* things, learn to safely create, copy, move, and delete them |
| **Used by** | 11 — Text Processing | `find ... -print0 \| xargs -0 grep` is the standard "search the codebase" pipeline |
| **Used by** | 20 — Cron and Scheduled Tasks | Log-cleanup crons are almost always a single `find -mtime +N -delete` |
| **Used by** | 23 — Logs and Log Management | `ls -ltr /var/log` and `find /var/log -size +100M` are step one of every log incident |
| **Used by** | 31 — Disk Management | "Disk is full" always ends in `du` + `find -size` |
