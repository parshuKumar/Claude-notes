# 07 — How the Shell Works

## ELI5 — The Simple Analogy

Imagine a **theatre stage manager** with a clipboard.

- You shout a request: *"Do the juggling act, with all the balls in the red box!"*
- The stage manager **does not juggle** — that is the whole point. He is a coordinator.
- First he **rewrites your request into a proper cue.** He opens the red box, sees three balls, and rewrites the cue as *"juggle ball1 ball2 ball3."* By the time it reaches the performer, **the vague words are gone** — the performer only ever sees the final expanded list. He has no idea a "red box" was mentioned.
- Then he **clones himself.** A perfect duplicate steps out. There are now two.
- The clone **walks into a costume booth and comes out as the juggler.** Same body, same employee badge number — but he *is* the act now. He is no longer a stage manager. There's no way back.
- Before stepping out, the clone can **rearrange props**: aim the spotlight, put a bucket under the stage. That's the window between "I'm a clone" and "I'm the juggler."
- The original manager **waits in the wings**, watching. When the act ends he notes whether it went well or the juggler dropped everything (the **exit code**), then turns for the next request.

That clone-then-become-someone-else move is `fork()` then `exec()`. It is the single most important mechanism in Unix, and it's how *every* program you have ever run got started.

---

## Where This Lives in the Linux Stack

```
Hardware (CPU, RAM, Disk)
  └── KERNEL   — implements clone, execve, wait4, dup2. Creates the process.
       │
       └── System Calls (clone, execve, wait4, dup2, open, close, pipe)
            │   ◀◀◀ THIS TOPIC leans HARD on these six.
            │
            └── C Library (glibc — fork(), execvp(), waitpid())
                 │
                 └── SHELL (bash/zsh)  ◀◀◀◀◀◀◀◀◀◀ THIS TOPIC IS *HERE*
                      │   The shell is JUST A PROGRAM — a userspace REPL. It has a
                      │   PID, shows up in `ps`, you can kill it. It is NOT the kernel
                      │   and NOT "Linux". It reads a line, mangles the text, and asks
                      │   the kernel to run things.
                      │
                      └── Commands (ls, node, grep — programs the shell EXECS)
                           They receive an argv[] the shell already finished chewing on.
                           They never see what you typed.
```

**One sentence:** the shell turns a line you typed into an `argv[]` array, then asks the kernel to run it. Everything else here is detail.

---

## What Is This?

The shell is an ordinary userspace program running a **REPL**: print a prompt, read a line, **expand** it, execute it, print the exit status, repeat. It has no special kernel privileges — `bash` runs with your uid, exactly like `ls` does (see **06 — Users and Groups**).

Its two real skills: (1) an aggressive text-expansion engine that rewrites your line *before* anything runs, and (2) the `fork()`/`exec()`/`wait()` dance that turns the rewritten line into a running process.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| The **shell** globs, not the command | `find . -name *.log` silently returns wrong results while `find . -name "*.log"` works, and you have no idea why |
| `fork()` then `exec()` | You can't read a `strace` of your own app, can't explain a `pm2` process tree, can't reason about zombies |
| Expansion order | `rm $FILE` where `FILE="my file.txt"` deletes `my` and `file.txt` — two files that aren't yours |
| Why `cd` is a builtin | Your deploy script `cd`s in a subshell and silently does work in the wrong directory |
| PATH + the hash table | You install a tool and get `command not found` from the *same shell* that just installed it |
| `$?` and exit codes | Your CI reports green while the deploy actually failed, because a pipe swallowed the status |
| Redirection is set up by the child, before exec | You can't explain `2>&1 >file` vs `>file 2>&1`, and a cron job silently loses all its errors |

---

## The Physical Reality

Your shell is a process. Nothing more.

```bash
$ echo $$
8207                     # $$ is the shell's OWN PID
$ ps -p $$ -o pid,ppid,uid,comm
    PID   PPID   UID COMMAND
   8207   8201  1001 bash
#            ▲     ▲ same uid as you — no special powers
#            └ its parent is sshd. Your shell was FORKED BY SSHD.
```

What sits in that process's memory — and why builtins have to exist:

```
┌── PROCESS: bash   PID 8207 ─────────────────────────────────────────────────┐
│  TEXT     the compiled bash binary, mmap'd from /usr/bin/bash                │
│  HEAP     • variable table:   PATH=/usr/bin:/bin   HOME=/home/deploy         │
│           • alias table:      ll → ls -alF                                   │
│           • the COMMAND HASH TABLE:   ls → /usr/bin/ls                       │
│           • the job table, readline history                                  │
│  FDs      0 → /dev/pts/0 (stdin)   1 → /dev/pts/0 (stdout)   2 → …(stderr)   │
│           ▲ these three are INHERITED by every child — that's why `ls` output │
│             hits your screen without ls knowing anything about terminals.     │
│  CREDS    uid=1001 gid=1001 groups=[1001,27,999]  ← from 06; inherited via fork│
└──────────────────────────────────────────────────────────────────────────────┘
```

Everything in that HEAP box is why builtins exist. `cd`, `export`, and `alias` mutate state *in this process's memory*. A child cannot reach in and change it — and that fact is the whole answer to "why is `cd` a builtin."

---

## How It Works — Step by Step

### The shell loop, honestly

```c
while (1) {
    print_prompt();                    // expand and print $PS1
    line = readline();                 // history, tab-completion, Ctrl-A… (EOF => exit; Ctrl-D)

    // ── PHASE 1: TEXT MANGLING (no process created yet — all string work) ──
    tokens = tokenize(line);           // split on unquoted metacharacters
    ast    = parse(tokens);            // build the command tree (pipes, &&, ;, subshells)
    words  = expand(ast);              // ◀◀◀ THE NINE EXPANSIONS (below). The big one.

    // ── PHASE 2: EXECUTION ──
    if (is_function(words[0])) { run_function(words); continue; }   // no fork
    if (is_builtin(words[0]))  { run_builtin(words);  continue; }   // no fork

    path = hash_lookup(words[0]) ?: search_PATH(words[0]);
    if (!path) { fprintf(stderr,"%s: command not found\n",words[0]); $?=127; continue; }

    pid = fork();                      // ◀◀◀ RETURNS TWICE
    if (pid == 0) {                    // ── CHILD (still bash!) — THE WINDOW ──
        setup_redirections();          // dup2() — this is where >, <, 2>&1, | happen
        execve(path, words, environ);  // ◀◀◀ NEVER RETURNS on success
        _exit(126);                    // only reached if exec FAILED
    } else {                           // ── PARENT ──
        waitpid(pid, &status, 0);      // block until child dies
        $? = WEXITSTATUS(status);      // harvest the exit code
    }
}
```

### `fork()` returns TWICE

Called **once**, returns **twice** — once in each of the two processes that now exist.

```
        BEFORE                    KERNEL: sys_clone()                 AFTER
  ┌──────────────────┐    ┌─────────────────────────────┐   ┌──────────────────────┐
  │ bash  PID 8207   │    │ • new task_struct, PID 8412 │   │ PARENT PID 8207      │
  │ fd 0,1,2 → tty   │    │ • COPY the fd table         │   │ pid = 8412 (child's) │
  │ uid 1001         │───▶│ • COPY the cred struct      │──▶│ → else branch        │
  │ cwd /home/deploy │    │ • COPY-ON-WRITE the memory  │   │ → waitpid(8412) BLOCK│
  │ pid = fork();    │    │   (page tables shared R/O;  │   └──────────────────────┘
  │      ◀ ONE call  │    │   nothing copied until a    │   ┌──────────────────────┐
  └──────────────────┘    │   WRITE traps the CPU)      │   │ CHILD  PID 8412      │
                          └─────────────────────────────┘   │ PPID = 8207          │
                                                            │ pid = 0  ◀ "you're   │
                                                            │           the kid"   │
                                                            │ → if branch          │
                                                            │ → dup2(), execve()   │
                                                            └──────────┬───────────┘
                                                                       ▼
                                              execve("/usr/bin/ls"): KERNEL tears down
                                              the child's memory, loads ls's ELF, resets
                                              the stack. KEEPS: PID 8412, the fd table,
                                              uid, cwd.
                                                                       ▼
                                              ┌──────────────────────────────────────┐
                                              │ ls  PID 8412  ← SAME PID.             │
                                              │ It WAS bash. Now it's ls. The process │
                                              │ stayed; the PROGRAM INSIDE was replaced│
                                              └──────────────────────────────────────┘
```

**Copy-on-write:** `fork()` copies the *page tables*, not the memory, marking every page read-only. The first write to a page traps and copies just that 4 KB. Since the child almost always immediately `execve()`s (throwing its whole address space away), essentially nothing gets copied. That's why forking a 2 GB Node process is fast.

**exec keeps the PID:** `execve()` is not "start a new process." It **replaces the program running inside this process.** PID, parent, file descriptors, uid, cwd all survive. Only code and memory are swapped. This is the mechanism that makes redirection possible.

### Why fork-then-exec instead of one `spawn()` call?

Windows has `CreateProcess()` — one call that takes a giant options struct because every customization must be a parameter up front. Unix's answer is more elegant: the **window** between fork and exec.

```
   fork() ───────── the child IS bash here ──────────▶ execve()
     │   It can run ARBITRARY C, call ANY syscall, and              │
     │   whatever it does to ITSELF SURVIVES the exec               │
     │   (exec keeps the fd table, cwd, uid, signals):              │
     │     close(1); open("out.txt"); dup2(fd,1);  ────────────────▶│  ls starts with fd 1
     │     setuid(1001); chdir("/var/www"); setrlimit(...) ────────▶│  ALREADY pointing at
     │                                                              │  the file. ls was never
     ▼                                                              ▼  told and cannot tell.
```

**`ls` has no idea it was redirected.** It writes to fd 1 like every Unix program does; the shell rearranged what fd 1 *points at* before `ls` started. This is the entire implementation of `>`, `>>`, `<`, `2>&1`, and `|` — with zero code in `ls`, `grep`, or `node`. That composability is what fork/exec buys, and it's why the pipeline is the defining Unix idea.

---

## The Nine Expansions — The Highest-Value Section in This Doc

When you press Enter, bash rewrites your line **before a single process is created.** The order is fixed and non-negotiable. Getting it wrong is the root cause of a huge fraction of all shell bugs.

```
  YOU TYPE:  echo ~/logs/{app,err}-$(date +%F).log

  0. TOKENIZE — split on unquoted metacharacters (| & ; ( ) < > space). Nothing
     substituted yet; still literal text. Build the command tree.
  1. BRACE {a,b} {1..5}   — PURELY TEXTUAL, happens FIRST, does NOT check the disk.
                            ~/logs/{app,err}-… → ~/logs/app-…  ~/logs/err-…
                            (`touch f{1,2}` CREATES two files.)
  2. TILDE ~ ~root ~+     — ~/logs → /home/deploy/logs. ONLY at word start, ONLY unquoted.
  3. PARAMETER $VAR ${V:-x} ${#V} ${V#pre} ${V/a/b}
  4. COMMAND SUBST $(cmd) `cmd`  — runs cmd in a SUBSHELL, captures stdout, strips
                            trailing newlines. $(date +%F) → 2026-07-12
  5. ARITHMETIC $(( 2+2 )) — integers only. $((10/3)) is 3.
        ▲ 3,4,5 run left-to-right in one pass and NEST.
  ─────────────────────────────────────────────────────────────────────────────
  6. WORD SPLITTING  ⚠⚠⚠ the RESULTS of 3,4,5 are split on $IFS (space/tab/newline).
                     THE #1 SOURCE OF SHELL BUGS. It is why you quote "$VAR".
                     A quoted "$VAR" is EXEMPT — quoting is how you opt out.
                     Literal text you typed is NOT re-split; only expansion RESULTS are.
  7. GLOB / PATHNAME  *  ?  [abc]  ⚠ THE SHELL HITS THE DISK HERE — readdir() + match.
                     Each word is checked against the filesystem. No match → the pattern
                     is left LITERAL (unless `shopt -s nullglob`).
  8. QUOTE REMOVAL   — strip the ' " \ that YOU typed. echo "hi" → argv[1] is `hi`.
  ─────────────────────────────────────────────────────────────────────────────
  9. REDIRECTION SETUP   (fork happens; child calls open() + dup2())
 10. EXECUTE            execve(path, argv[], envp[])

  THE COMMAND RECEIVES:  argv = ["echo",
                                 "/home/deploy/logs/app-2026-07-12.log",
                                 "/home/deploy/logs/err-2026-07-12.log"]
  It never saw a ~, a {, or a $(. By the time it runs, all of that is GONE.
```

### THE CRITICAL LESSON: the SHELL globs, not the command

Internalize this and you have internalized the shell. `ls` **has no glob support** — it never sees a `*`.

```
  YOU TYPE:  ls /var/log/*.log
  BASH (step 7): opendir("/var/log"), readdir() every entry, fnmatch() each name
                 against "*.log", sort the matches.
  BASH BUILDS:   argv = ["ls", "/var/log/auth.log", "/var/log/dpkg.log", "/var/log/kern.log"]
  BASH EXECS:    execve("/usr/bin/ls", argv, envp)
  ls RECEIVES:   three filenames. It thinks you typed them by hand. No idea a `*` existed.
```

**PROVE IT — the single best command in this doc:**
```bash
$ cd /var/log && echo *.log
alternatives.log auth.log dpkg.log kern.log   # `echo` can't glob either. Bash pre-expanded it.
```

Four consequences that explain a startling amount of shell behaviour:

**1. `rm *` is dangerous because bash hands `rm` an explicit kill list** — and the expansion happens before `rm` can reject anything. Worse, a file named `-rf` becomes a *flag*:
```bash
$ ls
-rf   important.txt
$ rm *            # bash expands to:  rm -rf important.txt   ← the "-rf" file became a flag
$ rm -- *         # DEFENCE: `--` means "no more flags after this"
```

**2. `find . -name *.log` is broken; `find . -name "*.log"` works.** Trips up every engineer once:
```bash
$ find . -name *.log     # bash expands *.log FIRST → find . -name app.log  → only finds
                         # files literally named app.log. With TWO .log files in cwd:
                         # find: paths must precede expression: `error.log'   ← the classic error
$ find . -name "*.log"   # quotes stop step 7 → find gets the LITERAL *.log and does its OWN
                         # recursive matching. bash's glob and find's -name are DIFFERENT engines.
```
The rule generalizes: **any command with its own pattern language — `find -name`, `grep`, `rsync --exclude`, `tar --wildcards` — needs the pattern quoted to keep bash's hands off it.**

**3. Quoting is not cosmetic — it is how you disable specific expansion steps** (table below).

**4. Empty globs are a footgun.** By default an unmatched `*.log` stays *literal* in argv:
```bash
$ cd /empty && ls *.log
ls: cannot access '*.log': No such file or directory     ← ls got a literal asterisk
$ for f in *.log; do echo "$f"; done   # runs ONCE with f='*.log' — not zero times!
$ shopt -s nullglob                    # FIX: unmatched globs expand to NOTHING
```

### Quoting — the exact rules

| Form | Blocks | Still expands | Use it when |
|---|---|---|---|
| `'single'` | **Everything.** Literal bytes. Can't even escape a `'` inside. | nothing | Exact bytes: regexes, `awk` programs, passwords with `$` |
| `"double"` | globbing (7), brace (1), tilde (2), **word splitting (6)** | `$VAR`, `$(cmd)`, `` `cmd` ``, `$((…))` | **The default. 95% of the time.** |
| `\x` | the single next character | — | one-off metacharacters: `mv my\ file.txt` |
| *(unquoted)* | nothing | everything, then splits and globs the result | only when you *deliberately* want splitting/globbing |

```bash
$ FILE="my report.txt"; touch "$FILE"
$ ls $FILE            # UNQUOTED → step 6 splits on the space → ls my report.txt (TWO args)
ls: cannot access 'my': No such file or directory
$ ls "$FILE"          # QUOTED → step 6 skipped → ONE argument.  ✅
my report.txt
$ echo "$HOME/*.log"  # /home/deploy/*.log   → $ expanded (3), glob blocked (7)
$ echo '$HOME/*.log'  # $HOME/*.log          → nothing expanded. Literal bytes.
```

**Burn this in: quote every variable expansion — `"$VAR"`, `"$@"`, `"$(cmd)"` — unless you have a specific, articulated reason not to.** (And it's `"$@"`, never `"$*"` and never bare `$@` — only `"$@"` preserves each argument as a separate word.)

---

## Builtins vs External Commands — and Why `cd` MUST Be a Builtin

An **external command** is a file on disk (`ls` is `/usr/bin/ls`, an ELF binary) — running it costs a fork + exec. A **builtin** is C code compiled *inside* bash — a function call, **no fork, no exec, no new process.**

```bash
$ type cd echo ls grep
cd is a shell builtin
echo is a shell builtin          # also exists at /usr/bin/echo — the builtin wins
ls is aliased to `ls --color=auto'
grep is /usr/bin/grep
```

Some builtins exist for **speed** (`echo`, `test`, `[`). A handful are builtins **out of necessity** — impossible to implement as external programs. `cd` is the canonical case. Its syscall is `chdir()`, and **`chdir()` changes the cwd of the calling process only.** There is no syscall to change another process's cwd.

```
   IF `cd` WERE /usr/bin/cd:
   bash (cwd /home/deploy) --fork--▶ child (cwd /home/deploy, inherited)
                                       execve("/usr/bin/cd", ["cd","/var/log"])
                                       chdir("/var/log")  → succeeds ✅
                                       exit(0)  💀 the process DIES, and its cwd —
                                                the ONLY thing it changed — dies with it
   bash (cwd STILL /home/deploy)  ← the parent was never touched.
```

The child changed its own cwd, then evaporated. **`cd` must run *inside* the shell process to have any effect** — not a design preference, a hard consequence of how `chdir()` works.

**PROVE IT:**
```bash
$ pwd; /usr/bin/cd /tmp; pwd     # /home/deploy … /home/deploy  ← external cd did NOTHING
$ pwd; cd /tmp; pwd              # /home/deploy … /tmp          ← builtin ran INSIDE bash
```

Same reasoning covers every builtin that mutates shell state: `export` (env table), `cd`/`pushd` (cwd), `exit`, `source`/`.` (runs a script *in* this shell), `exec` (replaces this shell), `alias`, `set`, `read`, `trap`, `ulimit`, `umask`. **A forked child cannot reach into its parent's memory. Ever.** That's kernel-enforced isolation (**01 — What Linux Actually Is**).

> This is why `./setup-env.sh` doesn't export your variables but `source setup-env.sh` does. `./script` forks a child whose `export` dies with it; `source` runs the lines *in your current shell*. Same file, opposite outcome. It's also why `nvm` is a shell *function*, not a binary — it must mutate your `PATH`, which a forked child never could. (More in **14 — Environment Variables**.)

---

## PATH Resolution and the Hash Table

When bash has an expanded `argv[0]` that isn't a function or builtin, it finds a file, in strict order:

```
  1. Contains a `/` ?  (./app.js, /usr/bin/node) → use it DIRECTLY. PATH ignored.
     (This is why you must type `./script.sh` — `.` is deliberately NOT on PATH, so a
      malicious `ls` in /tmp can't hijack your command.)
  2. ALIAS?    → substitute the alias text
  3. FUNCTION? → call it (no fork)
  4. BUILTIN?  → run it (no fork)
  5. HASH TABLE hit? → use the cached absolute path. NO disk search. ← the bug lives here
  6. WALK $PATH left→right, first match wins:
       PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/bin
       stat(dir+"/node") in each; first hit that's EXECUTABLE by my uid (05+06) STOPS.
       ⚠ ORDER MATTERS: if /usr/local/bin precedes /usr/bin, an nvm node SHADOWS the
         system node. This is why `node -v` differs between your shell and cron (which
         has a minimal PATH). The #1 cause of "works in my shell, fails in cron."
  7. Found → store in HASH TABLE → fork() → execve().  Not found → "command not found", $?=127
```

### The hash table — and the bug it causes

Walking PATH means a `stat()` per directory on *every* command, so bash caches results:

```bash
$ hash
hits   command
   4   /usr/bin/ls
   2   /usr/local/bin/node
```

The bug you will hit:
```bash
$ node -v
v16.20.0                 # bash caches: node → /usr/bin/node
$ nvm install 20         # installs to /usr/local/... which is EARLIER in PATH
$ node -v
v16.20.0                 # 😤 STILL OLD — bash used its CACHE, never re-walked PATH
$ hash -r                # clear the whole table  (or: hash -d node)
$ node -v
v20.11.0                 ✅
```
Mirror image: you `mv` a binary and bash keeps the stale path — the tell is an error with an **absolute path you never typed**: `bash: /usr/local/bin/oldtool: No such file or directory`. `hash -r` fixes it (a new shell does too — it starts with an empty table, which is why "just open a new terminal" works).

### Which lookup tool to use

| Command | What it does | Verdict |
|---|---|---|
| `command -v node` | **Builtin.** Reports what bash will actually run (alias, function, builtin, hash, PATH) | ✅ **Use in scripts.** POSIX, always right |
| `type -a node` | **Builtin.** Same knowledge; `-a` shows *every* match in priority order | ✅ **Use interactively** to see shadowing |
| `which node` | **External binary.** A separate process — can't see aliases, functions, builtins, or hash | ⚠️ Lies. `which cd` "finds" `/usr/bin/cd`, which never runs |

```bash
$ type -a echo
echo is a shell builtin        # ← what runs
echo is /usr/bin/echo          # ← exists but never reached
$ which echo
/usr/bin/echo                  # ← WRONG. `which` can't know about the builtin.
```

---

## Exit Codes and `$?`

Every process hands the kernel an integer when it dies; the parent collects it via `waitpid()`; bash stores it in `$?`.

```bash
$ ls /var/log >/dev/null; echo $?       # 0  — the ONLY success value
$ grep -q x /etc/hostname; echo $?      # 1  — grep: 1 means "no match", not an error
```

| Code | Meaning |
|---|---|
| `0` | Success (the only success) |
| `1` | Generic failure |
| `2` | Misuse of a builtin / bad usage |
| `126` | Found the file but **could not execute** it (not `+x`, or it's a directory) |
| `127` | **Command not found** (PATH walk failed) |
| `128+N` | Killed by signal N. `130`=SIGINT (Ctrl-C), `137`=SIGKILL — **an OOM-killed Node process** — `143`=SIGTERM (a graceful `docker stop`) |

`137` in container logs = the kernel's OOM killer got you (**01**, and **34 — Performance Investigation**), not a Node bug.

```bash
# ⚠ In a PIPELINE, $? is the LAST command's status only:
$ false | true; echo $?          # 0  😱  `false` failed and you'd never know. CI goes GREEN.
$ set -o pipefail; false|true; echo $?   # 1  ✅
# ALWAYS at the top of a deploy script:
set -euo pipefail
#    ││└ pipefail: a pipeline fails if ANY stage fails
#    │└─ nounset:  error on an undefined variable (catches a typo'd $PAHT)
#    └── errexit:  exit immediately on any non-zero status
```

---

## Interactive vs Non-Interactive, Login vs Non-Login

Four combinations. They matter because they decide **which startup files bash reads** — which sets your `PATH` — which decides whether cron can find `node`.

```
                │  LOGIN                              │  NON-LOGIN
────────────────┼─────────────────────────────────────┼────────────────────────────────
 INTERACTIVE    │ ssh deploy@host ; console ; bash -l │ typing `bash` ; new tmux pane ;
                │ READS /etc/profile, then FIRST of   │ new terminal tab
                │ ~/.bash_profile / ~/.bash_login /   │ READS ~/.bashrc only
                │ ~/.profile                          │
────────────────┼─────────────────────────────────────┼────────────────────────────────
 NON-           │ rare — `bash -l script.sh`          │ ./deploy.sh ; bash -c '…' ;
 INTERACTIVE    │                                     │ a CRON JOB ⚠ ; systemd ExecStart ⚠;
                │                                     │ a Dockerfile RUN ⚠
                │                                     │ READS ***NOTHING*** (only $BASH_ENV,
                │                                     │ if set — and it usually isn't)
```

**That bottom-right cell is where careers go to die.** Cron / systemd / Docker `RUN` read **neither `.bashrc` nor `.bash_profile`** — they get a bare minimal PATH (cron's is typically `/usr/bin:/bin`). Your nvm node lives in `~/.nvm/.../bin`, which is **not** on it. Result: `node: not found`, exit 127, in your cron log at 3 AM, every night.

**Fix: use absolute paths in cron and systemd. Always.** `ExecStart=/usr/local/bin/node …`. (Full treatment in **14** and **20 — Cron and Scheduled Tasks**.)

```bash
[[ $- == *i* ]] && echo interactive || echo non-interactive
shopt -q login_shell && echo login || echo non-login
```

---

## Exact Syntax Breakdown

```
type -a node
│    │  └── the name to resolve
│    └── -a = show ALL matches in priority order (not just the winner)
└── type — a BUILTIN. The TRUTH about what bash runs. `which` can't see aliases/
           functions/builtins/hash.
```

```
strace -f -e trace=clone,execve,wait4 bash -c 'ls'
│      │  │                            │    └── -c = read commands from this string
│      │  └── the syscalls to show. NOTE the Linux names:
│      │       • `clone` NOT `fork` — glibc's fork() is clone() on Linux; there is no
│      │         fork syscall on x86-64. Tracing `fork` shows NOTHING.
│      │       • `wait4` NOT `wait` — waitpid() → the wait4 syscall.
│      └── -f = FOLLOW forks. ◀◀◀ WITHOUT THIS YOU SEE NOTHING — everything interesting
│              happens in the CHILD, and strace won't follow it. -f is the whole point.
└── strace — trace syscalls
```

```
set -x    ($PS4-prefixed "+ " lines).  set +x turns it off.
└── xtrace — print each command AFTER expansion. ◀◀◀ THE #1 SHELL DEBUG TOOL: it shows
    the line as the shell FINALLY sees it — the actual argv[] handed to execve().
  $ set -x; ls *.log
  + ls --color=auto app.log error.log     ← alias substituted, glob ALREADY expanded
```

```
echo $$    $?    $!    $0     $#    "$@"
     │     │     │     │      │      └ ALL args, each a SEPARATE word. Always quote it.
     │     │     │     │      └ number of args
     │     │     │     └ name of the shell/script
     │     │     └ PID of the last BACKGROUND job (from `cmd &`)
     │     └ exit status of the last foreground command
     └ PID of the CURRENT shell. ⚠ In a SUBSHELL $$ still shows the PARENT's PID
       (inherited deliberately). Use $BASHPID for the real current one.
```

---

## The Full Annotated Trace: `ls -la /var/log > out.txt`

From keypress to output — the whole doc in one example.

```
PHASE 1 — TEXT MANGLING (ZERO processes yet)
  TOKENIZE: [ls][-la][/var/log][>][out.txt]   — `>` is a REDIRECTION OPERATOR, removed
            from the word list and attached to the command node (defaults to fd 1).
  EXPAND:   no brace/tilde/param/glob to do → argv = ["ls","-la","/var/log"]
            alias ls='ls --color=auto' fires → ["ls","--color=auto","-la","/var/log"]
  RESOLVE:  "ls" — no `/`, not a function/builtin. Hash hit → /usr/bin/ls.

PHASE 2 — FORK
  bash (8207) fork() [clone() syscall]. KERNEL: PID 8412, fd table COPIED (0,1,2→tty),
  memory COW, cred COPIED (uid 1001, from 06). Returns 8412 in parent, 0 in child.

PHASE 3 — THE CHILD SETS UP REDIRECTION  (still bash — this is the fork/exec window)
  fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666)  → 3
       O_TRUNC ◀◀◀ THIS is why `>` DESTROYS the file's contents, and it happens HERE,
               BEFORE ls runs. Even if ls fails, out.txt is already empty. (`>>` uses
               O_APPEND and does not truncate.)  mode 0666 & ~umask(022) → 0644 on disk (05).
  dup2(3, 1)  ◀◀◀◀◀◀ THE ENTIRE MECHANISM OF `>` IS THIS ONE SYSCALL: "make fd 1 a
              duplicate of fd 3." fd 1 no longer refers to the terminal.
              ── fd table now: 0→tty  1→out.txt  2→tty  (stderr STILL to the screen! —
                 that's why errors appear on your terminal with a plain `>`)
  close(3)    — scaffolding; fd 1 already holds the reference so the file stays open.

PHASE 4 — EXEC
  execve("/usr/bin/ls", ["ls","--color=auto","-la","/var/log"], environ)
    KERNEL: check x-bit vs euid 1001 (05+06); tear down the child's address space (bye
    bash); mmap ls's ELF; build a fresh stack with argv+envp; jump to entry.
    ▶ PID STILL 8412. fd table UNTOUCHED. ls inherits fd 1 → out.txt AND HAS NO IDEA.

PHASE 5 — ls RUNS
  openat("/var/log", O_RDONLY|O_DIRECTORY); getdents64() (name→inode, 04);
  newfstatat() ×N (mode/uid/gid/size, 04+05);
  openat("/etc/passwd") → uid 0 → "root";  openat("/etc/group") → gid 4 → "adm"
        ◀◀◀ THIS IS TOPIC 06, LIVE: the kernel gave numbers, ls looked up names in a FILE.
  write(1, "total 4520\ndrwxr-xr-x 12 root adm …", 8192)
        ◀◀◀ ls calls write() on fd 1 as always. The kernel follows fd 1 → out.txt → disk.
            NOT ONE LINE of ls's source knows about out.txt. THIS is the payoff of fork/exec.
  exit_group(0)

PHASE 6 — THE PARENT REAPS
  bash was BLOCKED in wait4(8412,…). Kernel wakes it, hands over the status → $?=0.
  The child's task_struct is freed (REAPED — had bash not called wait4() it would linger
  as a ZOMBIE → 16). bash prints the prompt. Back to the top of the loop.
```

**Run it and watch every step:**
```bash
$ strace -f -e trace=clone,execve,openat,dup2,close,write,wait4 \
    bash -c 'ls -la /var/log > /tmp/out.txt' 2>&1 | grep -vE 'ENOENT|\.so'
clone(...)                            = 8412                              ← FORK
[pid 8412] openat(…,"/tmp/out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
[pid 8412] dup2(3, 1)                 = 1        ◀◀◀ THE REDIRECTION. Right there.
[pid 8412] close(3)                   = 0
[pid 8412] execve("/usr/bin/ls", ["ls","-la","/var/log"], …) = 0         ← BECOMES ls
[pid 8412] write(1, "total 4520\ndrwxr-xr-x 12 root ad"..., 4096) = 4096
wait4(-1, [{WIFEXITED && WEXITSTATUS==0}], 0, NULL) = 8412                ← REAPED
```
`clone` → `openat` → `dup2` → `execve` → `write` → `wait4`. Six syscalls. That's a shell.

---

## Example 1 — Basic

```bash
echo $$                       # 8207 — your shell's PID
cd /var/log && echo *.log     # THE BIG ONE: `echo` can't glob; bash pre-expanded it
# alternatives.log auth.log dpkg.log kern.log

set -x; ls *.log; set +x      # watch the expansion, exactly as execve() sees it
# + ls --color=auto alternatives.log auth.log dpkg.log kern.log

echo "$HOME/*.log"            # /home/deploy/*.log  → $ expanded (3), glob blocked (7)
echo '$HOME/*.log'            # $HOME/*.log         → nothing expanded

F="my report.txt"; touch "$F"
ls $F                         # ls: cannot access 'my': …   ← step 6 split it into TWO args
ls "$F"                       # my report.txt               ← quoted → ONE arg  ✅

type cd echo ls grep          # cd/echo = builtin, ls = alias, grep = /usr/bin/grep
pwd; /usr/bin/cd /tmp; pwd    # /var/log … /var/log   ← external cd died; cwd unchanged
pwd; cd /tmp; pwd             # /var/log … /tmp       ← builtin ran INSIDE bash  ✅

true; echo $?                 # 0
false; echo $?                # 1
notacommand; echo $?          # bash: notacommand: command not found  … 127
```

---

## Example 2 — Production Scenario

**03:40.** The nightly log-archival cron job on `prod-api-03` has silently done nothing for six days. Nobody noticed until `/` hit 94%. The script works perfectly when you run it by hand.

```bash
$ cat /home/deploy/bin/archive-logs.sh
#!/bin/bash
cd /var/log/myapp
tar -czf /backups/logs-$(date +%F).tar.gz *.log
find . -name *.log -mtime +7 -delete
node /home/deploy/bin/notify-slack.js "archived"

$ tail -4 /var/log/archive.log
/home/deploy/bin/archive-logs.sh: line 5: node: command not found
find: paths must precede expression: `error.log'
tar: *.log: Cannot stat: No such file or directory
```

Four bugs, and **every one is a shell bug from this doc.**

```bash
# BUG 1  node: command not found (exit 127) — cron runs a NON-interactive NON-login shell
#        that reads NEITHER .bashrc NOR .bash_profile, so the nvm PATH line never runs.
$ env -i /bin/bash -c 'echo $PATH; command -v node; echo exit=$?'
/usr/bin:/bin                                   ← cron's minimal PATH
exit=1                                          ← node is NOT on it. CONFIRMED.

# BUG 2  find: … 'error.log' — the cwd has app.log AND error.log, so bash expands *.log FIRST:
$ cd /var/log/myapp && echo find . -name *.log -mtime +7 -delete
find . -name app.log error.log -mtime +7 -delete   ← the SHELL ate the pattern; find never saw *

# BUG 3  tar: *.log: Cannot stat — the `cd` on line 2 FAILED (dir was renamed to /var/log/api
#        last week), but with no `set -e` the script blindly continued in $HOME, where *.log
#        matched NOTHING, so bash left the LITERAL "*.log" in argv and tar tried to stat it.
$ ls /var/log/myapp
ls: cannot access '/var/log/myapp': No such file or directory    ← there it is.

# BUG 4 (the worst) — for SIX DAYS the script exited 0 (last command's status wins), so cron
#        saw success and nobody was paged. The disk filled up instead.

# ─── THE FIXED SCRIPT ───
$ cat > /home/deploy/bin/archive-logs.sh <<'EOF'
#!/bin/bash
set -euo pipefail          # a failed `cd` now ABORTS instead of running tar in the wrong dir
shopt -s nullglob          # an unmatched glob → NOTHING, not the literal "*.log"  (kills bug 3)
LOG_DIR=/var/log/api
NODE=/home/deploy/.nvm/versions/node/v20.11.0/bin/node   # ABSOLUTE  (kills bug 1)
cd "$LOG_DIR"              # with set -e a failure here EXITS
logs=( *.log )
(( ${#logs[@]} )) || { echo "no logs"; exit 0; }
tar -czf "/backups/logs-$(date +%F).tar.gz" -- "${logs[@]}"   # -- and quoted array: safe
find . -name "*.log" -mtime +7 -delete   # QUOTED → find does its OWN matching  (kills bug 2)
"$NODE" /home/deploy/bin/notify-slack.js "archived $(date +%F)"
EOF

# ─── VERIFY UNDER CRON'S ACTUAL ENVIRONMENT ───
$ env -i /bin/bash -x /home/deploy/bin/archive-logs.sh
+ cd /var/log/api
+ logs=(app.log error.log access.log)               ← the glob resolved correctly
+ tar -czf /backups/logs-2026-07-12.tar.gz -- app.log error.log access.log
+ find . -name '*.log' -mtime +7 -delete            ← the pattern SURVIVED to find  ✅
```

**What you just did:** every bug was shell semantics — not tar, find, or Node. `set -x` showed the post-expansion argv; `env -i` reproduced cron's stripped environment. That turned a six-day silent outage into a ten-minute fix.

---

## Common Mistakes

### Mistake 1 — `find . -name *.log`
**Wrong:** `find /var/log -name *.log` **Right:** `find /var/log -name "*.log"`

**Root cause:** globbing (step 7) runs in **bash**, against the **current directory**, before `find` executes. Three different failures from one bug: zero `.log` files in cwd → pattern stays literal → works *by accident*; **one** `.log` file → bash silently substitutes it → wrong results, **no error** (the dangerous case); two+ → find gets extra bare words → `paths must precede expression`.

**Fix / Prevention:** quote every pattern meant for a command with its own matcher — `find -name`, `grep`, `rsync --exclude`, `tar --wildcards`.

### Mistake 2 — Unquoted variables
**Wrong:** `rm $FILE` **Right:** `rm -- "$FILE"`

**Root cause:** word splitting (step 6). `$FILE`'s *result* is split on `$IFS`, then each piece is globbed.
```bash
$ FILE="my report.txt"; rm $FILE       # → rm my report.txt  (TWO files, neither yours)
$ DIR="/var/www /"; rm -rf $DIR        # → rm -rf /var/www /  ← you just ran rm -rf /
$ rm -rf "$DIR"                        # quoting SAVED YOU: "No such file or directory"
```
**Fix / Prevention:** quote **every** expansion — `"$VAR"`, `"$@"`, `"$(cmd)"`, `"${arr[@]}"`. Run `shellcheck` in CI; it catches this automatically (SC2086).

### Mistake 3 — Expecting `cd`/`export` in a script to affect your shell
**Wrong:** `./setup.sh` **Right:** `source setup.sh` (or `. setup.sh`)

**Root cause:** `./setup.sh` **forks a child bash**; the child's `chdir()`/`export` change the *child's* state, then the child exits and the kernel destroys it. **A child physically cannot modify its parent's memory** (kernel page tables, **01**). `source` doesn't fork — it runs the lines *in the current shell*.

**Prevention:** if a script's purpose is to change your shell's state (activate a venv, set env, cd), it must be sourced. Guard it:
```bash
[[ "${BASH_SOURCE[0]}" == "${0}" ]] && { echo "must be SOURCED: source ${0}" >&2; exit 1; }
```

### Mistake 4 — Redirection order: `2>&1 >file` vs `>file 2>&1`
**Wrong:** `node app.js 2>&1 > app.log` (errors still spray your terminal) **Right:** `node app.js > app.log 2>&1`

**Root cause:** redirections process **left to right**, and `2>&1` means "make fd 2 a copy of whatever fd 1 points at **right now**" — a snapshot, not a permanent link.
```
2>&1 >file :  2>&1 copies fd1(=tty) into fd2  → 2→tty ; then >file rewires fd1 → 1→file, 2→tty
              RESULT: stdout→file, stderr→SCREEN. The stack trace you needed is NOT in the log.
>file 2>&1 :  >file rewires fd1→file ; then 2>&1 copies fd1(=file) into fd2 → BOTH→file  ✅
```
**Fix:** use bash's order-proof shorthand `node app.js &> app.log` (⚠ not POSIX — in a `#!/bin/sh`/Alpine `ash` script use the explicit `> app.log 2>&1`).

### Mistake 5 — Trusting `which`, and the stale hash
**Wrong:** `which node` **Right:** `type -a node` / `command -v node`

**Root cause:** `which` is an **external binary** — a separate process with no access to bash's alias/function/builtin/hash tables (which live in bash's memory). It just re-walks PATH and guesses. `which cd` → `/usr/bin/cd` (a lie; the builtin runs). Companion bug: after installing a new version, `node -v` shows the old one because of a stale **hash** entry — `hash -r` clears it; an error with an absolute path you never typed is the tell.

**Prevention:** `command -v` in scripts, `type -a` interactively, `hash -r` when a version doesn't match what you just installed.

---

## Hands-On Proof

```bash
# PROVE IT: the SHELL globs, not the command. (The most important one.)
cd /var/log && echo *.log        # echo has NO glob support; bash pre-expanded the words

# PROVE IT: watch the expansion, exactly as execve() sees it.
set -x; ls *.log; set +x         # + ls --color=auto alternatives.log auth.log …

# PROVE IT: fork() returns TWICE and exec() keeps the SAME PID.
strace -f -e trace=clone,execve,wait4 bash -c 'ls' 2>&1 | grep -E 'clone|execve|wait4'
# clone(...) = 8412                       ← ONE call…
# [pid 8412] execve("/usr/bin/ls", …) = 0 ← SAME PID 8412 — it WAS bash, now it's ls
# wait4(-1, […]) = 8412                    ← the parent reaps it

# PROVE IT: the dup2() that IS `>`, and that O_TRUNC fires BEFORE the command runs.
echo "important" > /tmp/t; cat /tmp/t     # important
notacommand > /tmp/t; cat /tmp/t          # (EMPTY!) the CHILD truncated it, THEN exec failed
strace -f -e trace=openat,dup2 bash -c 'echo hi > /tmp/x' 2>&1 | grep -E 'dup2|/tmp/x'
# openat(…,"/tmp/x", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
# dup2(3, 1) = 1     ◀◀◀ THAT is `>`. One syscall.

# PROVE IT: cd MUST be a builtin.
pwd; /usr/bin/cd /tmp; pwd        # unchanged — the child chdir'd, then DIED
pwd; cd /tmp; pwd                 # changed  — the builtin ran INSIDE bash

# PROVE IT: `which` lies; `type` doesn't.
which cd        # /usr/bin/cd          ← NOT what runs
type -a echo    # builtin AND /usr/bin/echo — the builtin wins

# PROVE IT: bash caches command locations.
hash -r; ls >/dev/null; ls >/dev/null; hash   # 2 hits on /usr/bin/ls, ZERO extra PATH walks

# PROVE IT: exit codes and the pipeline trap.
false; echo $?                   # 1
notacommand; echo $?             # 127  — PATH walk failed
/etc/hostname; echo $?           # 126  — found it, not executable
false | true; echo $?            # 0  😱
set -o pipefail; false|true; echo $?   # 1  ✅

# PROVE IT: cron's environment has almost nothing in it.
env -i /bin/bash -c 'echo "PATH=$PATH"; command -v node || echo NO-NODE'
# PATH=/usr/bin:/bin   NO-NODE     ← why your cron job fails and your terminal doesn't
```

---

## Practice Exercises

### Exercise 1 — Easy
```bash
mkdir -p ~/shell-lab && cd ~/shell-lab && touch app.log error.log debug.txt
echo *.log           # WHY can `echo` do this? Which expansion step?
echo "*.log"         # which quote disabled which step?
set -x; ls *.log; set +x    # what is ls's REAL argv?
echo {a,b,c}-{1,2}   # did brace expansion touch the disk?
F="my file.txt"; touch "$F"; ls $F; ls "$F"   # how many args each time, and which step split them?
```
**Questions:** for each line, name the numbered expansion step (1–8) responsible. Paste your output.

### Exercise 2 — Medium
```bash
# 1. strace -f -e trace=clone,execve,wait4 bash -c 'ls /tmp' 2>&1 | grep -E 'clone|execve|wait4'
#    → Which PID does clone() return in the parent? Which PID owns the execve("/usr/bin/ls")
#      line? They're the SAME — explain in one sentence why exec does NOT create a new process.
#    → Drop -f and rerun. Almost everything vanishes. WHY?
# 2. strace -f -e trace=openat,dup2 bash -c 'echo hi > /tmp/e2' 2>&1 | grep -E 'e2|dup2'
#    → Which openat() flag destroys the file's existing contents? Say aloud what dup2(3,1) does.
#    → Does `echo` know it was redirected? How could it possibly find out?
# 3. echo PRECIOUS > /tmp/e2b; nosuchcmd > /tmp/e2b; cat /tmp/e2b
#    → What happened to PRECIOUS? WHICH process destroyed it, at which step of the trace?
# 4. bash -c 'ls /nope 2>&1 > /tmp/a'; bash -c 'ls /nope > /tmp/b 2>&1'; cat /tmp/a /tmp/b
#    → One captured the error, one didn't. Which, and why? Walk the fd table left-to-right.
```

### Exercise 3 — Hard (Production Simulation)
You inherit this deploy script; it "works on my machine" and fails in cron. Find every bug (≥ six) using only tools from this doc, then fix it.
```bash
#!/bin/bash
cd /var/www/api
git pull
BUILD_DIR=$(cat .buildconfig)     # .buildconfig contains the line:  dist build  (fat-fingered)
rm -rf $BUILD_DIR/*
find . -name *.map -delete
pm2 restart api                   # pm2 is at /home/deploy/.nvm/versions/node/v20.11.0/bin/pm2
echo "deployed" > /var/log/deploy.log 2>&1
```
```bash
# 1. WITHOUT running it, predict the post-expansion argv[] of `rm -rf $BUILD_DIR/*`. CONFIRM
#    safely:  BUILD_DIR=$(cat .buildconfig); set -x; echo rm -rf $BUILD_DIR/*; set +x
#    Which step created the extra argument, and how much of the filesystem is at risk?
# 2. Reproduce the `find . -name *.map` failure (make two .map files first). Which step ate it?
# 3. env -i /bin/bash -x deploy.sh  → which commands vanish, and why? (name the quadrant)
# 4. If `cd /var/www/api` fails, what does the script do? Which single `set` flag prevents it?
# 5. Fix logging so stdout AND stderr of the WHOLE script APPEND (not truncate) to the log.
#    Hint: `exec >> "$LOG" 2>&1` at the top redirects the SHELL'S OWN fds, which every child
#    inherits through fork(). One line — the payoff of everything in this doc.
# 6. Rewrite it: set -euo pipefail; shopt -s nullglob; quote every expansion; quote the find
#    pattern; ABSOLUTE paths for node/npm/pm2; `--` before user-controlled filenames.
# 7. `shellcheck deploy.sh` must report ZERO warnings.
```
**Deliverable:** the `set -x` output from step 1 (the two-argument disaster), the `find` error from step 2, the fixed script, and a clean `shellcheck` run.

---

## Mental Model Checkpoint

1. **`fork()` is called once. How many times does it return, in how many processes, and what different value does each get?**
2. **After `execve()`, does the process have a new PID? What survives, what is destroyed?**
3. **Why fork-then-exec instead of one `spawn()`? Name the capability the gap provides and two features built on it.**
4. **List the nine expansions in order. Why does #6 force you to write `"$VAR"`?**
5. **When you run `ls *.log`, what does `ls` receive in `argv[]`? Does `ls` contain any glob code?**
6. **Why does `find . -name *.log` fail while `find . -name "*.log"` works? Which program matches the pattern in each case?**
7. **Why is it *impossible* for `cd` to be an external binary? Name the syscall and its scope.**
8. **Trace `ls > out.txt`: the four syscalls the child makes, in order, between `fork()` and output.**
9. **You installed a newer `node` earlier in PATH but `node -v` shows the old one. Cause and one-word fix?**
10. **Why does cron say `node: command not found` when your shell finds it? Which startup file does cron NOT read?**

---

## Quick Reference Card

| Command | What It Does | Key Flags / Notes |
|---|---|---|
| `type -a <cmd>` | The truth about what bash runs — alias/function/builtin/hash/file | `-a` (all matches in priority order) |
| `command -v <cmd>` | POSIX; resolves like bash would; exit 0 if found | ✅ use in scripts, not `which` |
| `which <cmd>` | External binary; re-walks PATH; can't see aliases/functions/builtins/hash | ⚠️ lies |
| `hash` | bash's command-location cache | `-r` (clear ALL), `-d <cmd>` (forget one), `-t` (show path) |
| `set -x` | xtrace — print each command *after* expansion. #1 debug tool | `set +x` off; `bash -x script.sh` |
| `set -euo pipefail` | errexit + nounset + pipefail — top of every script | turns silent failures loud |
| `shopt -s nullglob` | unmatched glob → nothing, not the literal pattern | prevents `for f in *.log` running once with `'*.log'` |
| `echo $$` / `$BASHPID` | shell PID / real current PID (differ in a subshell) | — |
| `echo $?` | exit status of the last command | 126=not executable, 127=not found, 128+N=signal N |
| `strace -f -e trace=clone,execve,wait4 bash -c 'ls'` | watch fork/exec/wait live | **`-f` mandatory**; trace `clone`/`wait4` not `fork`/`wait` |
| `source f` / `. f` | run a file in the current shell — no fork; state persists | only way a script can `cd`/`export` for you |
| `exec <cmd>` | replace the current shell — no fork, same PID | `exec >> log 2>&1` redirects the shell's own fds |
| `env -i <cmd>` | run with an empty environment | reproduce a cron/systemd failure locally |
| `shellcheck script.sh` | static analysis; catches unquoted vars (SC2086) etc. | run it in CI |

---

## When Would I Use This at Work?

### Scenario 1: The deploy script that `rm -rf`'d the wrong thing
CI runs `rm -rf $BUILD_DIR/*` and `BUILD_DIR` (from a config file mangled by a bad merge) contains `dist build`. Word splitting (step 6) makes it `rm -rf dist/* build/*` — two targets, one nuked by surprise. Because you know the expansion order, your first move is `set -x` to see the post-expansion `argv[]`, not `rm`'s man page. Fix: `rm -rf -- "$BUILD_DIR"/*`, plus `shellcheck` in CI.

### Scenario 2: `node: command not found` at 3 AM, in cron only
Exit 127, but it runs fine when you type it. You know instantly: **cron runs a non-interactive non-login shell that reads neither `.bashrc` nor `.bash_profile`**, so the nvm PATH line never runs. Reproduce with `env -i /bin/bash -c 'command -v node'`, confirm, fix with an absolute path. You never suspected Node, cron, or the disk.

### Scenario 3: Debugging a mystery process tree with `strace`
A `pm2`-spawned child dies immediately with nothing in the logs. `strace -f -e trace=clone,execve,exit_group -p $(pgrep -f 'pm2 God')` shows `clone()` → `execve()` → `exit_group(127)`. 127 means the child's `execve` never found the binary; the `execve()` argument shows it tried `/usr/local/bin/ts-node`, which the last deploy removed. Because you understand fork/exec, a strace reads like a story.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | **01 — What Linux Actually Is** | 01 sketched the shell loop; this is the honest version. The kernel/userspace boundary is why a forked child can never modify its parent — the whole reason builtins exist. |
| **Builds on** | **06 — Users and Groups** | The child inherits the parent's uid/gid/groups through `fork()`; that inherited `cred` is what the kernel checks when your command opens a file. It's also why `ls -l` opens `/etc/passwd`. |
| **Builds on** | **05 — File Permissions** | The PATH walk checks the `x` bit against your euid (no `x` → exit 126, not 127); `>` creates its file with mode `0666 & ~umask`. |
| **Builds on** | **04 — Inodes in Depth** | Globbing is `readdir()` on a directory's entries — the shell reads the name→inode map to expand `*`. |
| **Next** | **08 — Navigating the Filesystem** | Now that you know `cd` is a builtin and `ls *.log` is pre-expanded, you can use them without superstition. |
| **Used by** | **12 — Pipes and Redirection** | This showed the `dup2()` behind `>`; 12 does the full `pipe()` + two `dup2()`s + two forks behind `\|`, plus `2>&1`, `tee`, `/dev/null`. |
| **Used by** | **13 — Shell Scripting** | `$?`, `set -euo pipefail`, `"$@"`, exit codes, and quoting are the raw materials of every script. |
| **Used by** | **14 — Environment Variables** | The login/non-login/interactive matrix decides which of `.bashrc`/`.bash_profile`/`.profile` runs — the #1 cause of "works in my shell, fails in cron." |
| **Used by** | **16 — Processes in Depth** | fork/exec/wait *is* the process lifecycle. A **zombie** is a child that exited whose parent never called `wait4()` — the call the shell makes at the end of every command here. |
| **Used by** | **19 — File Descriptors** | fd 0/1/2, the fd table, inheritance across fork, and `dup2()` — introduced here, dissected there. |
