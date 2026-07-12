# 12 — Pipes and Redirection

## ELI5 — The Simple Analogy

Imagine a factory assembly line.

- Every worker has **exactly three chutes**: an **IN chute** (parts arrive), an **OUT chute** (finished parts leave), and a **REJECT chute** (broken parts). The worker has no idea where any chute leads — they just grab from IN, drop into OUT, toss junk into REJECT.
- **Redirection** is the foreman re-plumbing a chute *before* the worker clocks in: "Your OUT chute now empties into that crate." The worker never notices — still just dropping parts into "OUT."
- **A pipe** is the foreman bolting worker 1's OUT chute onto worker 2's IN chute. They now work **at the same time** — worker 2 starts as soon as the first part slides down, not after worker 1's whole batch.
- The chute holds only so many parts. If worker 2 is slow, it fills and **worker 1 must stop and wait** (backpressure). If worker 2 walks off and seals the chute, worker 1's next part hits a wall and **worker 1 is killed on the spot** (SIGPIPE).

That is the entire Unix philosophy: small workers, three chutes each, a foreman who re-plumbs them.
IN = **fd 0 (stdin)** · OUT = **fd 1 (stdout)** · REJECT = **fd 2 (stderr)**

---

## Where This Lives in the Linux Stack

```
Hardware (Disk, Terminal, NIC)
  └── KERNEL
       │   • Per-process FILE DESCRIPTOR TABLE   ◀◀◀ THE FOUNDATION OF THIS TOPIC
       │   • The PIPE BUFFER (64 KB of kernel RAM, no disk, no inode on disk)
       │
       └── System Calls:  open()  dup2()  pipe()  close()  read()  write()   ◀◀◀ THIS TOPIC
            │
            └── C Standard Library (glibc — and its stdout BUFFERING)        ◀◀◀ THIS TOPIC
                 │
                 └── SHELL (bash) — parses  > >> < 2> 2>&1 |  and issues the
                      │            syscalls above BEFORE exec'ing your command  ◀◀◀ THIS TOPIC
                      └── Commands (grep, node, tee) — blindly read fd 0, write fd 1 and 2
```

**The critical insight:** redirection and pipes are implemented **entirely by the shell**, using kernel syscalls, **before your program starts running**. `grep` contains no code for `>`. `node` contains no code for `|`. They never see those characters — the shell eats them.

---

## What Is This?

Every Linux process is born with three open **file descriptors** — small integers (0, 1, 2) that index a table the kernel keeps per process. Programs read fd 0 and write fd 1 and 2 without ever knowing what's on the other end: a terminal, a file, a socket, or another process.

**Redirection** (`>`, `<`, `2>&1`) rewires those descriptors to point at files. **Pipes** (`|`) rewire them to an in-kernel buffer shared with another *concurrently running* process. Together they compose tiny single-purpose tools into powerful chains, with none of the tools knowing the others exist.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| fds 0/1/2 and the fd table | `2>&1`, `lsof`, `EMFILE: too many open files`, and `docker logs` all stay magic |
| Left-to-right redirection order | You write `cmd 2>&1 > log` in a cron job, lose **every** error message, debug blind for months |
| Why `sudo cmd > /root/f` fails | You fight "Permission denied" for 30 minutes *despite typing sudo* |
| Pipeline exit status | CI passes green while the build actually failed — `false \| tee log` exits **0** |
| stdio buffering | `node app.js \| grep ERROR` shows **nothing** for 10 min and you assume the app is dead |
| SIGPIPE / `>` vs `>>` | Node dies with code **141** for no clear reason; two `>` cron jobs truncate each other's logs |

---

## The Physical Reality

Three kernel structures. Learn these and the rest of this document is obvious.

```
  PER-PROCESS FD TABLE       SYSTEM-WIDE OPEN FILE TABLE     INODE TABLE (topic 04)
  (one per process)          (one entry per open() call)
  ┌────┬────────────┐        ┌────────────────────────┐      ┌──────────────────┐
  │ fd │ points to →│   ┌───▶│ offset: 4096           │─────▶│ inode 8419       │
  ├────┼────────────┤   │    │ flags: O_WRONLY|O_APPEND│     │ regular file     │
  │ 0  │ ───────────┼─┐ │    │ refcount: 1            │      │ /var/log/app.log │
  │ 1  │ ───────────┼─┼─┘    └────────────────────────┘      └──────────────────┘
  │ 2  │ ───────────┼─┤      ┌────────────────────────┐      ┌──────────────────┐
  │ 3  │ ───────────┼─┴─────▶│ offset: 0              │─────▶│ inode 1030       │
  └────┴────────────┘        │ flags: O_RDWR          │      │ CHAR DEVICE      │
                             │ refcount: 2  ◀─ shared │      │ /dev/pts/0 (TTY) │
                             └────────────────────────┘      └──────────────────┘

  DEFAULT STATE of every process launched from a terminal — INHERITED, not created:
      fd 0 (stdin)  ──┐
      fd 1 (stdout) ──┼──▶ all three point at the SAME thing: your terminal /dev/pts/0
      fd 2 (stderr) ──┘
```

That's why output *and* errors both hit your screen, and why typing feeds the program. `fork()` **copies the fd table**, so children inherit whatever the shell had.

```bash
# PROVE IT: fds 0,1,2 are symlinks to the terminal device
ls -l /proc/$$/fd
# lrwx------ 1 you you 64 Jul 12 14:03 0 -> /dev/pts/0     (1 and 2 point at the same /dev/pts/0)
```
(No `/proc` on macOS — use `lsof -p $$`. Another reason to run these in `docker run -it ubuntu:22.04 bash`.)

**A pipe is NOT a file.** No disk inode, no filename, no path. It is a **ring buffer in kernel memory**, 64 KB by default on Linux (16 pages × 4 KB):

```
   Writer                                              Reader
   ┌────────────┐      ╔══════════════════════╗      ┌────────────┐
   │ fd 1 ──────┼─────▶║ KERNEL PIPE BUFFER   ║◀─────┼─── fd 0    │
   │ write(1,…) │      ║ 64 KB ring buffer    ║      │ read(0,…)  │
   └────────────┘      ╚══════════════════════╝      └────────────┘
     FULL → writer BLOCKS (backpressure)   ·   EMPTY → reader BLOCKS
     write end closed → reader gets EOF    ·   read end closed → writer gets SIGPIPE, DIES
```

---

## How It Works — Step by Step

### Trace A — How the shell implements `ls > out.txt`

```
COMMAND: ls > out.txt

SHELL (parent):
  1. Parses the line and STRIPS "> out.txt" off the argument list.
     argv handed to ls is exactly ["ls"] — NOT ["ls", ">", "out.txt"]
  2. fork() → a child process (still bash) — topic 07

CHILD (still bash — the window between fork and exec is where ALL the magic happens):
  3. open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666)  → returns fd 3
  4. dup2(3, 1)   ← "make fd 1 a copy of fd 3". The kernel CLOSES the old fd 1 (the TTY)
                    and points fd 1 at the same open-file entry as fd 3.
  5. close(3)     ← tidy up. fd 1 alone now points at the file.
  6. execve("/usr/bin/ls", ["ls"], environ)  ← the ls binary REPLACES the child's memory.
       ⚠ execve DOES NOT RESET THE FD TABLE. fd 1 still points at out.txt.
  7. ls: write(1, "file1\nfile2\n", 12)  ← ls believes it writes to the screen. It does not.

SYSCALLS: fork() → open() → dup2() → close() → execve() → write(1,…) → exit_group()
OUTPUT:   nothing on the terminal; out.txt contains the listing
```

Step 6 is the whole game: **`execve()` preserves file descriptors.** Redirection is set up in the tiny window while the child is still bash and still obeys bash. (You'll `strace` this yourself in Hands-On Proof and watch `argv` arrive as just `["ls"]`.)

### Trace B — How the shell implements `ls | grep log`

```
1. pipe(pipefd) → kernel allocates the 64 KB buffer, returns TWO fds:
                  pipefd[0] = READ end  (say fd 3)
                  pipefd[1] = WRITE end (say fd 4)
2. fork() → CHILD 1:  dup2(4, 1)            # my stdout IS the pipe's write end
                      close(3); close(4)    # close the originals — CRITICAL
                      execve("/usr/bin/ls", …)
3. fork() → CHILD 2:  dup2(3, 0)            # my stdin IS the pipe's read end
                      close(3); close(4)
                      execve("/usr/bin/grep", ["grep","log"], …)
4. PARENT: close(3); close(4)   # if the parent keeps the write end open, grep NEVER sees EOF
5. PARENT: waitpid() on both children.  BOTH CHILDREN RUN SIMULTANEOUSLY.
```

```
      ┌────────────────┐                 ┌────────────────┐
      │ ls  (PID 5001) │                 │ grep (PID 5002)│
      │ fd 1 ──────────┼─▶ ╔══════╗      │ fd 0 ◀─────────┤
      │ fd 2 ──────────┼┐  ║ PIPE ║──────▶ fd 1 ──────────┼─▶ TTY
      └────────────────┘│  ╚══════╝      │ fd 2 ──────────┼─▶ TTY
                        └────────────────────────────────────▶ TTY
                         ⚠ ls's STDERR IS NOT PIPED
```

**`|` connects fd 1 only.** `ls /nope | grep x` still splatters the error on your terminal. To pipe errors too: `2>&1 |` (or bash's `|&`).

### Trace C — Concurrency and backpressure

`cmd1 | cmd2` does **not** mean "run cmd1, save the output, then run cmd2." Both start immediately.

```
t=0ms   ls and grep BOTH exec. grep calls read(0) → pipe empty → grep BLOCKS (sleeps in kernel).
t=1ms   ls writes 200 bytes into the pipe. Kernel WAKES grep. grep processes them
        WHILE ls is still running and still writing.
…       If ls floods faster than grep drains, the buffer hits 64 KB and ls's next
        write(1,…) BLOCKS. That is BACKPRESSURE — automatic flow control, for free.
```

This concurrency is exactly why the single most useful command in production works at all — if pipelines were sequential, this would print nothing, forever:

```bash
tail -f /var/log/app.log | grep ERROR    # tail NEVER exits; grep filters LIVE
```

### Trace D — SIGPIPE: why `yes | head -1` terminates

```
t=0    `yes` prints "y\n" as fast as the CPU allows, filling the 64 KB pipe.
t=0    `head -1` reads, prints one "y", then calls exit(0).
t=0+ε  head exits → kernel closes head's fd 0 → the pipe's READ END refcount hits 0.
t=0+ε  `yes` calls write(1, "y\n", 2) into a pipe with NO READER.
       KERNEL → sends SIGPIPE to `yes` → default action is TERMINATE.
       `yes` dies instantly. Exit status 141 = 128 + 13  (SIGPIPE is signal 13).
```

This is a **feature**: it's how a finite consumer tells an infinite producer to stop. Without SIGPIPE, `yes | head -1` would spin forever burning a core.

**Node relevance:** if something downstream goes away (`head`, a closed `docker logs` pipe), Node gets **`EPIPE`** on `process.stdout` instead of dying (libuv ignores SIGPIPE). Unhandled, it crashes your app — hence the classic guard `process.stdout.on('error', e => { if (e.code === 'EPIPE') process.exit(0); });`.

---

## THE ORDER-MATTERS TRAP — `> file 2>&1` vs `2>&1 > file`

The single most misunderstood thing in shell. Read it twice.

**The rule:** `2>&1` does **not** mean "send stderr to stdout forever." It means **"make fd 2 a copy of whatever fd 1 points to RIGHT NOW."** It is `dup2(1, 2)` — a one-time snapshot of a pointer. And **bash processes redirections strictly left to right.**

### Case A — `cmd > file 2>&1` — BOTH go to the file ✅

```
START (inherited)               fd 1 ──▶ /dev/pts/0 (TTY)
                                fd 2 ──▶ /dev/pts/0 (TTY)

STEP 1  "> file"  → open("file", O_WRONLY|O_CREAT|O_TRUNC)=3 ; dup2(3,1)
                                fd 1 ──▶ file          ◀── CHANGED
                                fd 2 ──▶ /dev/pts/0

STEP 2  "2>&1"    → dup2(1, 2)  "copy fd 1's CURRENT target into fd 2"
                                fd 1 currently points at ──▶ file
                                fd 1 ──▶ file
                                fd 2 ──▶ file          ◀── CHANGED

RESULT      fd 1 ──┐
                   ├──▶ file    ✅ stdout AND stderr land in the file
            fd 2 ──┘
```

### Case B — `cmd 2>&1 > file` — stderr goes to the TERMINAL ❌

```
START (inherited)               fd 1 ──▶ /dev/pts/0 (TTY)
                                fd 2 ──▶ /dev/pts/0 (TTY)

STEP 1  "2>&1"    → dup2(1, 2)  "copy fd 1's CURRENT target into fd 2"
                                fd 1 currently points at ──▶ THE TTY
                                fd 1 ──▶ /dev/pts/0
                                fd 2 ──▶ /dev/pts/0    ◀── "changed" to where it ALREADY was. NO-OP.

STEP 2  "> file"  → open("file",…)=3 ; dup2(3,1)
                                fd 1 ──▶ file          ◀── CHANGED — but fd 2 was NOT re-pointed!
                                fd 2 ──▶ /dev/pts/0    ◀── STILL THE TERMINAL

RESULT      fd 1 ──▶ file           stdout in the file
            fd 2 ──▶ /dev/pts/0     ❌ ERRORS ON YOUR SCREEN, NOT IN THE LOG
```

**fd 2 does not "follow" fd 1.** Changing fd 1 afterwards does not drag fd 2 along. They are independent slots in a table; `dup2` copies a value at one instant in time.

**Why this destroys you in production:** a cron job with `2>&1 > /var/log/job.log`. Cron has no terminal, so stderr goes to cron's mail spool — or nowhere. Your log has the happy output and **none of the errors**. The job has been failing silently for three months.

**Is Case B ever useful?** Exactly once — when you deliberately want to pipe *only stderr*: `./build.sh 2>&1 >build.log | grep -i warn` sends **only** stderr into grep while stdout goes to the log. If you didn't write that on purpose, it's a bug.

**Memory aid:** read it right-to-left as "and also." `> file 2>&1` = "stdout to file, **and also** stderr to wherever stdout is (= file)." Or just use `&> file`.

---

## /dev/null — The Bit Bucket

`/dev/null` is a **character device** (major 1, minor 3 — topic 03), not a file. Its kernel driver does two things: **writes always succeed and are discarded**, **reads return EOF immediately**. `ls -l /dev/null` shows `crw-rw-rw- 1 root root 1, 3` — the leading `c` is "character device," and `1, 3` are its major/minor numbers.

| Idiom | Effect |
|---|---|
| `cmd > /dev/null` | Discard stdout, **keep errors visible** |
| `cmd 2> /dev/null` | Discard stderr (silences `find`'s "Permission denied" noise) |
| `cmd > /dev/null 2>&1` | Discard everything (order matters!) |
| `cmd &> /dev/null` | Same, bash shorthand |
| `cmd < /dev/null` | Instant EOF on stdin — stops commands hanging for input in cron/CI |

**The danger.** `2>/dev/null` is a loaded gun:
```bash
npm ci 2>/dev/null || echo "install failed"   # ❌ you threw away the ONE message saying WHY
```
**Rule: discard stderr only for noise you have already read and understood.** Never as a first debugging move, and never in a deploy script.

---

## Buffering — Why Your Piped Logs Vanish

A **libc** behavior, not kernel or shell — and it burns everybody. glibc buffers stdout and picks the mode by calling `isatty(1)`:

| fd 1 points at | Buffering mode | Flushed when |
|---|---|---|
| A **TTY** | **Line-buffered** | every `\n` |
| A **pipe** or **file** | **Block-buffered (4–8 KB)** | buffer full, or clean exit |
| **stderr (fd 2)** | **Unbuffered — always** | immediately |

```bash
./noisy.sh                 # output appears instantly    (TTY  → line-buffered)
./noisy.sh | grep ERROR    # NOTHING for minutes         (pipe → 4KB block-buffered)
```

The program is fine. The data is sitting in a userspace buffer **inside the process**, never handed to `write()` yet. **And if the process is then SIGKILLed (hello, OOM killer), that buffer is never flushed — those lines are gone forever.** This is why "the logs just stop right before the crash."

**Fixes:**
```bash
stdbuf -oL ./cmd | grep ERROR   # force LINE buffering on the child's stdout (-o0 = unbuffered)
grep --line-buffered ERROR      # fix a stage in the MIDDLE of a pipeline (awk: fflush())
unbuffer ./cmd | …              # (expect pkg) runs cmd under a fake PTY → it thinks it's a TTY
python3 -u …    #  or  PYTHONUNBUFFERED=1
```

**Node specifics — you will hit this.** Node doesn't use glibc stdio, but has the same split: `process.stdout` to a **TTY** is **synchronous**; to a **pipe** it is **asynchronous** (queued on the event loop); to a **file** it is synchronous on Linux/macOS.

So `console.log(x); process.exit(0);` can **truncate output when stdout is a pipe** — the async write never drains. That's why `docker logs` sometimes misses your last lines. Don't call `process.exit()` right after logging; set `process.exitCode = 1` and let the loop drain. And note `docker run` gives Node a **pipe**, not a TTY, unless you pass `-t` — which is why adding `-t` "magically fixes" missing logs.

---

## Exact Syntax Breakdown

```
command > file 2>&1
│       │ │    │││
│       │ │    ││└── 1 = the TARGET fd. "&1" means "fd number 1" — NOT a file named "1"
│       │ │    │└─── & = "what follows is an FD NUMBER, not a filename"
│       │ │    └──── 2 = the fd being redirected (stderr)
│       │ └───────── the file to open: O_WRONLY|O_CREAT|O_TRUNC
│       └─────────── > = redirect fd 1 (stdout is the DEFAULT for a bare >)
└─────────────────── the program — knows NOTHING about any of this

  ⚠ dropping the & gives you "2 > 1" = stderr into a FILE LITERALLY NAMED "1"
```

```
./deploy.sh 2>&1 | tee -a deploy.log
│           │    │ │   │  └── the file to ALSO write to
│           │    │ │   └───── -a = APPEND. Without it, tee TRUNCATES the log!
│           │    │ └───────── tee: copy stdin → stdout AND → file
│           │    └─────────── the pipe (carries fd 1 ONLY — hence we merged 2 into 1 FIRST)
│           └──────────────── merge stderr into stdout BEFORE the pipe. Order is correct here.
└──────────────────────────── you SEE everything live AND it's all saved.

  In cmd1 | cmd2 | cmd3, all THREE run CONCURRENTLY, and $? = cmd3's status ONLY.
```

```
cat <<'EOF' > /etc/systemd/system/api.service
    │ │└─┘
    │ │ └── the delimiter. QUOTED ('EOF') → NO expansion; UNQUOTED (EOF) → $VAR/`cmd` DO expand
    │ └──── heredoc: feed the following lines to the command's stdin (fd 0)
    └────── cat just reads fd 0. It has no idea a heredoc exists.

  <<-EOF     strips leading TABS (not spaces) — lets you indent inside a function
  <<< "str"  HERESTRING: one string → stdin.  grep x <<< "$var"  beats  echo "$var" | grep x

  diff <(cmd1) <(cmd2)   PROCESS SUBSTITUTION: bash runs each cmd, wires its stdout to a pipe,
                         and substitutes the PATH /dev/fd/63 into argv. diff open()s two
                         "filenames" that are really pipes. No temp files, no cleanup.
```

---

## Example 1 — Basic

```bash
mkdir -p /tmp/redir && cd /tmp/redir

# A command that writes to BOTH streams:
sh -c 'echo "this is stdout"; echo "this is stderr" >&2'
# this is stdout       ← fd 1
# this is stderr       ← fd 2   (both hit the TTY, so they LOOK identical)

# Redirect ONLY stdout — stderr still hits your screen, PROVING they are separate:
sh -c 'echo "this is stdout"; echo "this is stderr" >&2' > out.txt
# this is stderr       ← still on screen!
cat out.txt            # this is stdout   ← the file got only fd 1

# Redirect ONLY stderr — now the OTHER one is left on screen:
sh -c 'echo "this is stdout"; echo "this is stderr" >&2' 2> err.txt
# this is stdout
cat err.txt            # this is stderr

echo one > log.txt ; echo two >  log.txt ; cat log.txt   # "two"        (>  = O_TRUNC, one GONE)
echo one > log.txt ; echo two >> log.txt ; cat log.txt   # "one" / "two" (>> = O_APPEND, kept)

# The empty-a-file idiom — a redirection with NO COMMAND AT ALL:
> log.txt                    # bash opens with O_TRUNC and execs nothing. Now 0 bytes.
wc -c < log.txt              # 0    ← and here stdin comes from a file

# PIPES ARE CONCURRENT — prove it with timestamps:
(echo A; sleep 2; echo B) | while read -r l; do echo "$(date +%T) got $l"; done
# 14:03:01 got A     ← printed IMMEDIATELY, not after 2 seconds
# 14:03:03 got B     ← the reader was consuming LIVE

# tee — see it AND save it, mid-pipeline:
ls -la | tee listing.txt | wc -l      # 7    (tee passed everything through to wc)
```

---

## Example 2 — Production Scenario

**Situation:** 02:14, PagerDuty. Your Node API on `api-prod-01` is 502'ing — nginx is up, the app is not. A cron-triggered deploy ran 20 minutes ago and **reported success**. You SSH in.

```bash
# 1. Is the app alive?
systemctl status api
# ● api.service - Node API
#      Active: activating (auto-restart) (Result: exit-code) since Sat 02:13:58 UTC
#     Process: 30112 ExecStart=/usr/bin/node /srv/api/current/server.js (code=exited, status=1/FAILURE)
#                                                                              ^^^^^^^^ crash-looping

# 2. The deploy "succeeded" — check its log.
tail -3 /var/log/deploy.log
# [02:12:48] Running npm ci...
# [02:13:55] Build complete.
# [02:13:56] Deploy finished OK          ← IT LIED. Why?

# 3. Look at the cron line that ran it.
crontab -l | grep deploy
# 0 2 * * * /srv/scripts/deploy.sh 2>&1 > /var/log/deploy.log
#                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^
#   THE TRAP. fd 2 was dup2'd to the TTY-that-doesn't-exist BEFORE fd 1 was pointed at
#   the file. Every error message went into the void. The log holds only the happy echoes.

# 4. So where DID the errors go? Nowhere — but the APP's own stderr is in journald:
journalctl -u api -n 3 --no-pager
# Jul 12 02:13:58 api-prod-01 node[30112]: Error: Cannot find module 'pino'
# Jul 12 02:13:58 api-prod-01 node[30112]:     at Module._resolveFilename (node:internal/modules/cjs/loader:1145:15)

# 5. So npm ci must have failed and the script never noticed.
ls /srv/api/current/node_modules/ | wc -l          # 0   ← EMPTY. npm ci definitely failed.
df -h /
# Filesystem  Size  Used Avail Use% Mounted on
# /dev/vda1    39G   39G     0  100% /
#                                ^^^^ DISK FULL. npm ci died with ENOSPC — on stderr — into the void.

# 6. Fix the outage.
sudo journalctl --vacuum-size=200M                 # Vacuuming done, freed 6.4G
cd /srv/api/current && npm ci --omit=dev
sudo systemctl restart api
curl -sf -o /dev/null -w '%{http_code}\n' http://localhost:3000/health    # 200 ✅

# 7. Fix the REAL bug — the redirection order.
crontab -e
# 0 2 * * * /srv/scripts/deploy.sh > /var/log/deploy.log 2>&1
#                                  ^^^^^^^^^^^^^^^^^^^^^ file FIRST, THEN merge.
```

And the second half of the bug, inside `deploy.sh` itself:

```bash
npm ci 2>&1 | tee -a "$LOG"      # WRONG: $? is TEE's status (0), NOT npm's!

set -o pipefail                  # RIGHT: pipeline status = rightmost NON-ZERO status
npm ci 2>&1 | tee -a "$LOG"      # now $? is npm's failure, and `set -e` aborts (topic 13)
```

**What you just did:** used the fd table and left-to-right redirection ordering to explain a three-month-old silent-failure bug — and used `pipefail` to make it impossible to recur.

---

## Common Mistakes

### Mistake 1 — `sudo echo "x" > /root/file` → Permission denied

**Wrong:** `sudo echo "deploy" > /root/notes.txt` → `bash: /root/notes.txt: Permission denied`. "But I typed sudo?!"

**Root cause — the exact sequence:**
```
1. bash (uid 1000, NOT root) parses the line, then forks a child — still uid 1000
2. child calls open("/root/notes.txt", O_WRONLY|O_CREAT|O_TRUNC)
     → KERNEL: /root is drwx------ root root. uid 1000 has no write bit. → -1 EACCES
3. bash prints "Permission denied" and ABORTS.
   ⚠ sudo IS NEVER EXECUTED. It never got the chance to elevate anything.
```
`sudo` elevates **the command**. The **redirection** is the *shell's* job — and the shell is still you.

**Fix — pipe into a privileged writer:**
```bash
echo "deploy" | sudo tee    /root/notes.txt > /dev/null   # tee runs as ROOT and calls open()
echo "deploy" | sudo tee -a /root/notes.txt > /dev/null   # append variant
```
tee's *stdin* is the pipe (opened as you — fine), but **tee itself is root**, so root does the `open()`. The trailing `> /dev/null` just stops tee echoing the content back at you. (The alternative `sudo sh -c 'echo deploy > /root/notes.txt'` also works — the redirecting shell is itself root — but quoting gets ugly; prefer `tee`.)

**Prevention:** memorize `| sudo tee`. With a quoted heredoc it is *the* idiom for writing any root-owned config: `cat <<'EOF' | sudo tee /etc/systemd/system/api.service > /dev/null`.

---

### Mistake 2 — `cmd 2>&1 > file`

**Wrong:** `./deploy.sh 2>&1 > deploy.log` — errors go to the terminal/void, not the log.
**Right:** `./deploy.sh > deploy.log 2>&1` — or unambiguously `./deploy.sh &> deploy.log`.

**Root cause:** `2>&1` is `dup2(1,2)` = "copy fd 1's *current* target." Redirections apply left→right; fd 2 does not track fd 1. **Diagnose:** if a log file has plenty of output but *never any errors*, check the redirection order first. **Prevention:** in bash just write `&>` / `&>>` — but note **`&>` is bash/zsh only, NOT POSIX**; in a `#!/bin/sh` script (which is **dash** on Debian/Ubuntu!) you must write `> file 2>&1`.

---

### Mistake 3 — Trusting a pipeline's exit code

**Wrong:** `npm run build | tee build.log` then `if [ $? -eq 0 ]; then echo "build OK"; fi` — prints "build OK" even when the build EXPLODED.

**Root cause:** POSIX says a pipeline's exit status is that of the **last** command. `tee` essentially always succeeds, so npm's failure is discarded. The most disturbing two lines in this document:
```bash
false | true;  echo $?      # 0   ← a pipeline containing `false` "SUCCEEDED"
true  | false; echo $?      # 1
```

**Fix:** `set -o pipefail` makes the pipeline exit with the rightmost NON-ZERO status, so npm's failure survives. Or inspect every stage with the bash array `${PIPESTATUS[@]}` — e.g. `"1 0"` means npm failed(1), tee ok(0). Read `PIPESTATUS` on the VERY NEXT LINE — any command overwrites it.

**Prevention:** every script and every CI job starts with `set -euo pipefail` (topic 13).

---

### Mistake 4 — `cmd > file` where `file` is also the input

**Wrong:** `sort access.log > access.log` — access.log is now **EMPTY**, the data GONE. **Root cause:** the shell performs the redirection **before the command runs**. `open(..., O_TRUNC)` truncates the file to **0 bytes**, releasing the inode's data blocks (topic 04); *then* `sort` starts and reads an empty file. There is no undo.

**Fix:**
```bash
sort access.log > access.sorted && mv access.sorted access.log
sed -i.bak 's/x/y/' app.log            # in-place tools do the temp-file dance for you
grep ERROR app.log | sponge app.log    # `sponge` (moreutils) soaks up ALL stdin, THEN opens the output
```

**Prevention:** `set -o noclobber` makes `>` refuse to overwrite an existing file (force with `>|`).

---

### Mistake 5 — Assuming `|` carries stderr, and that `>` is safe for shared logs

**Wrong:** `./flaky-job.sh | grep -i error` — finds NOTHING. The errors went to fd 2; the pipe carries fd 1.
**Right:** `./flaky-job.sh 2>&1 | grep -i error` (or bash 4+: `./flaky-job.sh |& grep -i error`)

**And the concurrent-writer half — `>` and `>>` are NOT interchangeable.** With `>` (O_TRUNC) each `write()` targets the fd's **own private offset**; two processes on the same file overwrite each other and lose lines. With `>>` (O_APPEND) the kernel **atomically** seeks-to-end-and-writes under the inode lock, so every write lands at the true current end. That is why **every** logger, cron line and `tee -a` uses append: two cron jobs with `>> /var/log/jobs.log` coexist; two with `> /var/log/jobs.log` destroy each other. (This O_APPEND atomicity holds for a **single** `write()` on a **local** fs — not reliably over NFS, and a huge write can still be split; for safe logs keep one line per write, under 4096 bytes.)

---

## Hands-On Proof

```bash
# PROVE IT: every process starts with exactly fds 0, 1, 2
ls -l /proc/self/fd
# 0 -> /dev/pts/0    1 -> /dev/pts/0    2 -> /dev/pts/0    3 -> (the `ls` itself)

# PROVE IT: redirection is open()+dup2() BEFORE execve — and argv is JUST ["ls"]
strace -f -e trace=openat,dup2,execve bash -c 'ls > /tmp/x' 2>&1 | grep -E 'tmp/x|dup2|execve.*ls'
# openat(AT_FDCWD, "/tmp/x", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
# dup2(3, 1)                       = 1
# execve("/usr/bin/ls", ["ls"], …) = 0     ← ls NEVER SEES THE ">"

# PROVE IT: >> uses a DIFFERENT open flag than >
strace -e trace=openat bash -c 'echo a >> /tmp/x' 2>&1 | grep tmp/x
# openat(AT_FDCWD, "/tmp/x", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3   ← O_APPEND, not O_TRUNC

# PROVE IT: a pipe is an anonymous kernel object with NO path
bash -c 'sleep 60 | cat' & sleep 0.3; ls -l /proc/$(pgrep -x sleep | head -1)/fd
# 1 -> 'pipe:[482913]'    ← not a file. An inode in the `pipefs` pseudo-filesystem.

# PROVE IT: the order trap, live (compare the two log files)
sh -c 'echo OUT; echo ERR >&2' >/tmp/a 2>&1 ; cat /tmp/a    # OUT and ERR
sh -c 'echo OUT; echo ERR >&2' 2>&1 >/tmp/b ; cat /tmp/b    # only OUT (ERR hit your screen)

# PROVE IT: pipeline status is the LAST command's; pipefail fixes it; SIGPIPE = 141
false | true ; echo "pipeline=$?  stages=${PIPESTATUS[*]}"   # pipeline=0  stages=1 0
( set -o pipefail; false | true ; echo "pipefail=$?" )       # pipefail=1
yes | head -1 >/dev/null; echo "${PIPESTATUS[0]}"            # 141 = 128 + 13 = SIGPIPE

# PROVE IT: process substitution is a REAL PATH under /dev/fd
echo <(ls)    # /dev/fd/63   ← bash made a pipe and handed you its path
cat  <(ls)    # ...and cat can open() that path and read it  (/dev/fd -> /proc/self/fd)

# PROVE IT: buffering changes with the destination (compare the two — one BATCHES, one is LIVE)
python3    -c 'import time;[print("tick",i) or time.sleep(1) for i in range(3)]' | cat  # BATCHED
python3 -u -c 'import time;[print("tick",i) or time.sleep(1) for i in range(3)]' | cat  # -u → live
```

---

## Practice Exercises

### Exercise 1 — Easy

Using the two-stream command `sh -c 'echo GOOD; echo BAD >&2'` in `/tmp/ex12`:

1. Capture **only stdout** into `good.txt` (BAD must still print to your screen).
2. Capture **only stderr** into `bad.txt` (GOOD must still print to your screen).
3. Capture **both** into `all.txt` — two different ways (`2>&1` and `&>`).
4. Discard **both** entirely — your terminal must stay completely silent.
5. **Append** a second run to `all.txt` without destroying the first.

**Question:** show `cat good.txt bad.txt all.txt` and say, in one sentence each, which fd landed where.

---

### Exercise 2 — Medium

```bash
# 1. Run:  sh -c 'echo O; echo E >&2' 2>&1 > /tmp/trap.log
#    Where did "E" go? Now cat /tmp/trap.log. Explain it using the fd table.
# 2. Run:  sh -c 'echo O; echo E >&2' > /tmp/ok.log 2>&1
#    Explain the difference in ONE sentence about dup2().
# 3. Run:  false | true ; echo $? ; echo "${PIPESTATUS[@]}"    then enable pipefail and re-run.
# 4. Use strace to capture the exact openat() flags for `>` versus `>>`.
# 5. Run `echo <(echo hi)` then `cat <(echo hi)`. Why does the first print a PATH?
```
**Question:** paste your output. What are the exact `openat()` flags for `>` and for `>>`?

---

### Exercise 3 — Hard (Production Simulation)

Create a fake `deploy.sh` that echoes 3 progress lines to stdout, one `WARN` line to stderr, then `exit 1`. Write a runner (max 6 lines) that:

1. Streams **both** stdout and stderr **live to your terminal** so you can watch the deploy.
2. **Simultaneously** appends every line to `/tmp/deploy.log`.
3. Exits with **deploy.sh's** real exit code — **not** tee's.
4. Then prints only the `WARN` lines from the log, with **no buffering delay**.

Finally, use process substitution — `diff <(env | sort) <(sudo env | sort)` — and explain three of the differences you see.

**Question:** paste the script and its output. Which single construct guaranteed requirement #3?

---

## Mental Model Checkpoint

1. **What are fds 0, 1, 2, where do they come from, and what kernel structure holds them?**
2. **Explain precisely why `cmd 2>&1 > file` leaves stderr on the terminal — in terms of `dup2()` and left-to-right processing.**
3. **Which two syscalls implement `> file`, and at what exact moment relative to `fork()` and `execve()`?**
4. **Is a pipe a file? Where does its data physically live, and how big is it by default?**
5. **Why does `yes | head -1` terminate instead of running forever? What is `yes`'s exit status and why that number?**
6. **`false | true` — what is `$?`, why, and what two mechanisms fix it? And why does `sudo echo x > /root/f` fail, with the one-line idiom that works?**
7. **Your healthy Node app logs every second, but `node app.js | grep INFO` shows nothing for 5 minutes. What is happening, and how do you fix it without touching the app?**

---

## Quick Reference Card

| Construct | What It Does | Notes / Gotchas |
|---|---|---|
| `> file` | stdout → file, **truncate** | `O_WRONLY\|O_CREAT\|O_TRUNC`. Destroys existing content. |
| `>> file` | stdout → file, **append** | `O_APPEND`. Kernel-atomic. Safe for concurrent writers. |
| `< file` | file → stdin (fd 0) | Command reads it as if you typed it |
| `2> file` / `2>> file` | stderr → file / append | The fd number goes *before* the `>` |
| `2>&1` | fd 2 → **wherever fd 1 points NOW** | `dup2(1,2)`. **ORDER MATTERS** — must come AFTER `> file` |
| `&> file` / `&>> file` | both streams → file | **bash/zsh only** — NOT POSIX `sh`/dash |
| `\|` | left's fd 1 → right's fd 0 | Concurrent. **Does NOT carry stderr.** |
| `\|&` | fd 1 **and** fd 2 → next stdin | bash 4+; identical to `2>&1 \|` |
| `/dev/null` | Discards writes; reads give EOF | Char device 1,3 |
| `< /dev/null` | Instant EOF on stdin | Stops hangs in cron/CI |
| `tee file` | stdin → stdout **and** → file | `-a` = append (**it TRUNCATES by default**) |
| `\| sudo tee f` | Write a root-owned file | The only sane way — `sudo cmd > f` fails |
| `<<EOF` | Heredoc → stdin | Body **is** expanded (`$VAR`, backticks) |
| `<<'EOF'` | Heredoc, quoted delimiter | Body is **literal** — no expansion. Use for configs. |
| `<<-EOF` | Heredoc, strips leading **TABS** | Tabs only, never spaces |
| `<<< "str"` | Herestring → stdin | `grep x <<< "$var"` |
| `<(cmd)` | Process substitution (read) | Becomes `/dev/fd/63`. `diff <(a) <(b)` |
| `>(cmd)` | Process substitution (write) | `tee >(gzip > a.gz) >(wc -l)` |
| `mkfifo p` | **Named** pipe — has a real path | Topic 03. Persists on disk; data still in kernel. |
| `$?` | Exit status of the last command | For a pipeline: the **last stage only** |
| `${PIPESTATUS[@]}` | **Array** of every stage's status | bash only. Read on the very next line. |
| `set -o pipefail` | Pipeline status = rightmost non-zero | **Put this in every script.** |
| `set -o noclobber` | `>` refuses to overwrite | Override with `>\|` |
| `stdbuf -oL cmd` | Force line-buffering on cmd's stdout | Fixes delayed piped logs |
| `grep --line-buffered` | Flush grep per line | Fixes delay MID-pipeline |

---

## When Would I Use This at Work?

### Scenario 1: Capturing a deploy log without going blind
You want the output on your screen **and** in a file **and** you want the script to fail honestly:
```bash
set -o pipefail
./deploy.sh 2>&1 | tee -a "/var/log/deploy-$(date +%F).log"
[[ ${PIPESTATUS[0]} -eq 0 ]] || { echo "DEPLOY FAILED"; exit 1; }
```
`2>&1` **before** the pipe (errors must go *into* it). `tee -a` so today's second deploy doesn't erase the first. `PIPESTATUS`/`pipefail` so tee's success can't mask the failure.

### Scenario 2: Silencing a noisy cron job — correctly
Cron mails you every line of stdout. A chatty backup mails 400 lines a night, so you learn to ignore the mail, so you miss the night it fails.
```bash
0 3 * * * /srv/backup.sh > /dev/null 2>&1     # WRONG — you now never learn about failures
0 3 * * * /srv/backup.sh > /dev/null          # RIGHT — drop the chatter, KEEP errors (cron mails only on failure)
# BEST — everything to a real log, no hanging on stdin, loud on failure:
0 3 * * * /srv/backup.sh < /dev/null >> /var/log/backup.log 2>&1 || echo "backup FAILED — see the log"
```

### Scenario 3: Diffing prod vs staging config at 2am, without touching disk
Prod is 500ing, staging is fine, same commit. What differs?
```bash
diff <(ssh prod  'sudo cat /etc/nginx/sites-enabled/api') \
     <(ssh stage 'sudo cat /etc/nginx/sites-enabled/api')
# 12c12
# <     proxy_pass http://127.0.0.1:3000;
# >     proxy_pass http://127.0.0.1:3001;
```
No temp files, nothing to clean up, works over SSH — the fastest config-drift check that exists. Same trick for package lists: `diff <(ssh prod 'npm ls --depth=0') <(ssh stage 'npm ls --depth=0')`.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | `open`, `dup2`, `pipe`, `write` are the syscall boundary in action |
| **Builds on** | 03 — Everything Is a File | `/dev/null` is a char device; a FIFO is a file type — that's why fds work uniformly |
| **Builds on** | 07 — How the Shell Works | fork→exec is the *window* in which redirection is set up; `execve` preserves fds |
| **Builds on** | 11 — Text Processing | grep/sed/awk/sort are the *workers*; pipes are the *conveyor belt* between them |
| **Next** | 13 — Shell Scripting Fundamentals | `set -o pipefail`, exit codes, `trap` — no safe script exists without this topic |
| **Forward-ref** | 19 — File Descriptors | The full fd table, `lsof`, `/proc/PID/fd`, `ulimit -n`, `EMFILE` |
| **Used by** | 17 — Process Management | SIGPIPE is one signal among many; exit code 128+N is defined here |
| **Used by** | 20 — Cron · 22 — systemd · 23 — Logs | Every cron line and every `.service` file is an exercise in redirection |
| **Used by** | 33 — Node in Production | stdout buffering to pipes vs TTYs, EPIPE handling, `docker logs` |
