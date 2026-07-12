# 16 — Processes in Depth

## ELI5 — The Simple Analogy

A **recipe card** in a drawer is a *program*. It is paper. Nothing is happening. That's `/usr/bin/node` on disk.

The moment a cook picks it up, claims a counter, lays out bowls and starts chopping — that's a **process**: the recipe *being executed*, by a specific cook, at a specific counter, with a specific pile of half-chopped onions.

- The **counter space** is the process's memory (its address space). One cook's counter is not another's. If cook A knocks over a bowl, cook B's soup is fine.
- The manager's **clipboard line** — *"Cook #4831, started 2pm, working from recipe 'node', reports to Cook #1, currently waiting on the oven, has three pans checked out"* — is the kernel's `task_struct`. That clipboard line **is** what a process is, from the kernel's point of view.
- Ten cooks can cook the same recipe at once: one recipe card, ten counters, ten clipboard lines. That's why `node` is one file but `ps aux | grep node` shows eight processes.
- A cook who finished and walked out, but whose clipboard line the manager hasn't crossed off, is a **zombie**. Nobody is standing there. The counter is clean. Only ink remains.

**Program = recipe. Process = cooking. `task_struct` = the clipboard line.**

---

## Where This Lives in the Linux Stack

```
Hardware (CPU cores, RAM, MMU/page tables)
  └── KERNEL
       │   ┌──────────────────────────────────────────────────┐
       │   │  PROCESS SUBSYSTEM  ◀◀◀ THIS TOPIC                │
       │   │   • task_struct (one per process/thread)          │
       │   │   • the scheduler (who runs next, for how long)   │
       │   │   • virtual address spaces (one per process)      │
       │   │   • the process tree (parent/child, reaping)      │
       │   └──────────────────────────────────────────────────┘
       └── System Calls  ◀◀◀ THIS TOPIC TOO
            │   fork() clone() execve() wait4() exit_group() getpid()
            └── C Library (glibc wraps them: fork(), execl(), system())
                 └── Shell (bash — forks + execs for EVERY external command)
                      └── Commands (node, nginx, ls — each becomes a process)
```

The process is the **unit the kernel schedules and the unit the kernel isolates**. File descriptors (19), signals (17), memory (18), systemd (22) are all just *things the kernel tracks per-process*.

---

## What Is This?

A **process** is an executing instance of a program: its code loaded into memory, plus its own private virtual address space, plus a block of kernel bookkeeping tracking who it is, what it owns, and what it is doing.

The program is a passive file on disk. The process is a live thing in RAM with a PID, a parent, open files, a memory map, and a state. Run the same program twice and you get two completely independent processes.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| fork + copy-on-write | You'll believe forking a 2 GB Node process copies 2 GB of RAM, and architect around a problem that does not exist |
| exec keeps the PID and open fds | You'll never understand why `node app.js > log.txt` works, or how pm2 hands your app a log file |
| Zombies | You'll `kill -9` a `<defunct>` process forever and wonder why it won't die (it is already dead) |
| Orphans / re-parenting | You'll panic that child processes "escaped" to PID 1 — which is normal and correct |
| **PID 1 in Docker** | `docker stop` SIGTERMs your app, your app **ignores it**, and 10s later it's SIGKILLed mid-request. Dropped connections on every deploy. |
| D state | You'll `kill -9` a stuck process, it won't die, and you'll blame the kernel — when your EBS volume or NFS mount is hung |
| `/proc/PID` | You'll have no way to answer "what is this thing actually doing, what files does it hold, what env did it get" on a box with no debugger |
| Threads are tasks | You'll see 11 `node` rows in `ps -eLf` and think you're leaking processes (that's libuv's threadpool) |

---

## The Physical Reality

### 1. In the kernel: `task_struct`

Per process, the kernel allocates one `struct task_struct` (kernel memory, ring 0 — you can never touch it directly). It holds everything the kernel knows:

```
┌──────────────── struct task_struct (kernel space) ────────────────┐
│  pid          4831           ← this process's ID                   │
│  ppid         1              ← who forked it                       │
│  state        S              ← R / S / D / T / Z                   │
│  exit_code    —              ← filled in when it exits             │
│  cred:  uid/euid 1001        ← real vs effective user (05, 06)     │
│         gid/egid 1001           setuid changes euid                │
│  fs:    cwd   /srv/app       ← current working directory           │
│         root  /              ← its "/"  (chroot / containers!)     │
│         umask 0022           ← default permission mask (05)        │
│  files: → FD TABLE           ← fd 0,1,2,3...          ◀ TOPIC 19   │
│           [0]→/dev/pts/0 [1]→/var/log/app.log [3]→socket:[…]      │
│  mm:    → ADDRESS SPACE      ← text/data/heap/mmap/stack + pgtables│
│  signal:→ HANDLER TABLE      ← what to do on SIGTERM  ◀ TOPIC 17   │
│           blocked mask, pending mask                               │
│  sched: prio 20, nice 0, utime/stime (CPU consumed)                │
│  thread_group → sibling tasks   ← THREADS live here                │
│  parent / children / sibling    ← THE PROCESS TREE                 │
└────────────────────────────────────────────────────────────────────┘
```

**Everything under `/proc/PID/` is the kernel formatting parts of this struct as text.** That is all `/proc` is.

### 2. In memory: the virtual address space

Each process gets a private 64-bit address space and *believes* it owns the machine. The MMU + page tables translate its fake addresses to real RAM pages.

```
   HIGH  0xffffffffffffffff
┌────────────────────────────────────────────────────────┐
│  KERNEL SPACE                                           │
│  Mapped into every process but marked ring-0 only.      │
│  Touch it from userspace → SIGSEGV. It's mapped here so │
│  a syscall doesn't have to swap page tables.            │
├──────────────── ~0x00007fff... ────────────────────────┤
│  [env + argv]   environment variables & command-line args│
├────────────────────────────────────────────────────────┤
│  STACK          function frames, locals, return addrs   │
│      │          GROWS DOWN ▼   (8 MB default: ulimit -s)│
│         ...unmapped hole (fault here = SIGSEGV)...      │
│      ▲          mmap region grows DOWN too              │
│  MEMORY-MAPPED REGIONS                                  │
│    libc.so.6 and every other shared library             │
│    mmap()'d files, and BIG malloc()s (>128 KB)          │
│    Node's V8 heap lives here (giant anonymous mmaps)    │
│         ...unmapped hole...                             │
│      ▲          GROWS UP (brk / sbrk / mmap)            │
│  HEAP           small malloc() lives here               │
├────────────────────────────────────────────────────────┤
│  BSS    uninitialized globals — zero-filled by the      │
│         kernel; takes ZERO bytes in the binary on disk  │
├────────────────────────────────────────────────────────┤
│  DATA   initialized globals (int x = 5;)          rw-   │
├────────────────────────────────────────────────────────┤
│  TEXT   the machine code.  r-x  READ-ONLY.  SHARED —    │
│         8 nginx workers share ONE physical copy.        │
└──────────────── 0x0000000000400000 ────────────────────┘
   LOW  (page 0 deliberately unmapped → NULL deref = SIGSEGV)
```

Two facts that pay off later:
- **TEXT is read-only and shared** — 8 `node` processes don't use 8× the binary's memory, and this is *why RSS double-counts* (topic 17).
- **Heap grows up, stack grows down.** Infinite recursion grows the stack down into a guard page → SIGSEGV (V8 usually catches it first: "Maximum call stack size exceeded").

---

## How It Works — Step by Step

### Trace: `node server.js` typed into bash

```
COMMAND:     node server.js
SHELL DOES:  fork() to clone itself, then execve() the node binary in the child
KERNEL DOES: allocate a new task_struct; COW-share the parent's pages; on exec,
             discard that address space and build a fresh one from the ELF file
SYSCALLS:    clone() → execve() → mmap() ×N → brk() → openat() → wait4()
OUTPUT:      a new process — new PID, same fds 0/1/2 as the shell
```

```
1. bash calls fork()  [really clone() with no sharing flags]
   ├── kernel allocates a NEW task_struct
   ├── COPIES the fd table  → the child inherits stdin/stdout/stderr
   ├── COPIES the page tables — but NOT the pages. Every writable page in BOTH
   │   parent and child is marked READ-ONLY and flagged copy-on-write.
   └── fork() RETURNS TWICE:  in the parent → the child's PID (e.g. 4831)
                              in the child  → 0

2. The child (still a copy of bash) sees fork() returned 0, so it knows it is the
   child, and immediately calls:  execve("/usr/bin/node", ["node","server.js"], environ)

3. Kernel's execve():
   ├── checks the file is executable by this euid (topic 05)
   ├── reads the ELF header, finds the interpreter /lib64/ld-linux-x86-64.so.2
   ├── DESTROYS the entire old address space (all that COW bookkeeping: discarded)
   ├── maps the new TEXT (r-x) and DATA (rw-), zeroes the BSS
   ├── builds a fresh stack holding argv + envp
   ├── KEEPS:   the PID, the PPID, open file descriptors, cwd, uid/gid, nice value
   └── RESETS:  signal HANDLERS back to default (the code that handled them is gone).
                Signals set to IGNORE stay ignored.
   → jumps to the dynamic linker, which mmap()s libc etc., then into node's main()

4. Meanwhile bash calls wait4(4831, &status, 0, NULL) and blocks in S state.

5. node eventually calls exit_group(0).
   ├── kernel tears down the address space (all memory freed NOW)
   ├── closes every fd
   ├── keeps the task_struct in state Z (zombie) holding ONLY the exit status
   └── sends SIGCHLD to bash

6. bash's wait4() returns the status. The kernel FREES the task_struct. The zombie
   is gone, the PID is reusable. bash sets $? and prints a prompt.
```

**That is the whole game: fork + exec + wait + exit.** Everything else is detail.

### Why fork() is cheap: copy-on-write

"fork copies the process" would mean forking a 2 GB Node process copies 2 GB. It does not.

```
AFTER fork(): nothing is copied. Page tables are cloned and every writable page
              in BOTH processes is marked READ-ONLY + flagged COW.

    parent ──┐                  ┌── child
             ▼                  ▼
          ┌──────────┐  refcount = 2
          │ page A   │ r-  (COW)      ONE physical copy of everything.
          │ page B   │ r-  (COW)
          │ page C   │ r-  (COW)
          └──────────┘

CHILD WRITES to page B:
   → MMU raises a page fault (write to a read-only page)
   → kernel sees the COW flag: not an error — this is the plan
   → kernel allocates ONE new 4 KB page, memcpy's B into it, maps it rw- into the
     child only, drops the refcount on the original
   → the write instruction is retried and succeeds
   → COST: 4096 bytes + one fault.  NOT 2 GB.
```

What you can do with that at work: forking a huge Node process to shell out to `git` is fine — only written pages get copied, and `execve` discards them all anyway. Redis's `BGSAVE` forks precisely to get a free, frozen snapshot of memory. The only real cost of forking a 2 GB process is copying **page tables** — a few MB of kernel structs, microseconds. Caveat: a GC'd runtime writes all over its heap, so a forked-and-*not*-exec'd Node child gradually un-shares pages as V8's GC touches them.

### clone(): threads are just processes that share memory

Linux has no separate "thread" object. There is only the **task**. `fork()` and `pthread_create()` both call the same syscall — `clone()` — with different flags.

| clone flag | `fork()` | `pthread_create()` | Meaning |
|---|---|---|---|
| `CLONE_VM` | no | **yes** | share the same address space |
| `CLONE_FILES` | no | **yes** | share the same fd table |
| `CLONE_FS` | no | **yes** | share cwd / root / umask |
| `CLONE_SIGHAND` | no | **yes** | share signal handlers |
| `CLONE_THREAD` | no | **yes** | same thread group → same reported PID |

So a **thread is a task with `CLONE_VM|CLONE_THREAD`**. It has its own **TID** but shares the **TGID** — and the TGID is what userspace calls "the PID." `getpid()` returns the TGID, so all threads report the same PID. Their TIDs live in `/proc/PID/task/`.

This is why plain `ps aux` shows **one** `node` line but `ps -eLf` shows ~11 — the main thread, 4 libuv threadpool threads (`UV_THREADPOOL_SIZE`, default 4, used by `fs.*`, DNS, crypto), and several V8 threads (GC, compiler). `ls /proc/$(pgrep -n node)/task/` lists one TID per thread; the main thread's TID equals the PID.

### wait() and the exit-status protocol

On exit the kernel does **not** delete the process. It keeps the corpse — the `task_struct` stripped of all resources — so the parent can learn *how* it died. The parent collects it with `wait()` / `waitpid()` / `waitid()`.

```
 The status is a packed int; macros unpack it:
   WIFEXITED(s)   → it called exit()  → WEXITSTATUS(s) = the 0–255 code
   WIFSIGNALED(s) → a signal killed it → WTERMSIG(s)   = the signal number

 The SHELL flattens this into $?  (topic 13):
   exited normally  → $? = the exit code               (0–125)
   killed by signal → $? = 128 + the signal number
       137 = 128 + 9  = SIGKILL   ◀ OOM killer / docker kill / k8s hard-stop
       143 = 128 + 15 = SIGTERM   ◀ docker stop / systemctl stop, unhandled
       139 = 128 + 11 = SIGSEGV
```

Memorise **137**. It is the most common "why did my container die" number in the industry, and in a container it almost always means the OOM killer.

---

## Zombies, Orphans, and PID 1

### Zombie processes (`Z`, `<defunct>`)

A **zombie** has already exited, but its parent has not yet called `wait()`.

```
 A zombie STILL has:                 A zombie does NOT have:
   • a process-table entry             • memory (freed at exit)
   • its PID                           • open file descriptors
   • its exit status                   • CPU time (never scheduled again)
   • a few hundred bytes of kernel     • a stack, heap, or code
                                       • ANY ability to run
```

```bash
$ ps aux | grep defunct
deploy 14992  0.0  0.0     0     0 ?   Z   03:11  0:00 [imagemin] <defunct>
#                          ^^^^^ ^^^^^     ^
#                          VSZ=0 RSS=0     state Z — it holds no memory at all
```

**You cannot kill a zombie.** It is already dead. `kill -9 14992` sends SIGKILL to something with no code to run and no state to destroy; the kernel drops it. There is nothing left to kill.

The fix is always the **parent**:
- **(a)** Fix the parent to reap — it calls `wait()`, the zombie is freed. The correct fix.
- **(b)** Send SIGCHLD to the parent — sometimes nudges a sloppy handler into reaping.
- **(c)** Kill the **parent** — the zombie is re-parented to PID 1, which reaps in a loop forever, and it vanishes instantly.

**When zombies actually matter.** One zombie is a few hundred bytes — harmless. Ten thousand is an outage: they consume PIDs, and when `/proc/sys/kernel/pid_max` is exhausted, **`fork()` starts returning EAGAIN and *nothing* can start** — not even a shell to diagnose it with.

The classic Node case:

```js
// ❌ fire-and-forget: nothing ever waits for this child
app.post('/thumbnail', (req, res) => {
  spawn('convert', [src, dst]);     // no 'exit' handler, no reference kept
  res.send('ok');
});
```

Node's `child_process` installs a SIGCHLD handler and reaps for you, so plain Node rarely leaks zombies. The leak appears when you `spawn` with `{detached: true}` and the intermediate parent dies, when a native addon forks raw — **or when your Node app is PID 1 in a container and inherits orphans it never reaps.** That last one is the real killer.

### Orphan processes

An **orphan** is a process whose parent died first. This is **not** a leak and **not** an error.

```
 BEFORE:                          AFTER the parent dies:
  PID 1 (systemd)                  PID 1 (systemd)
     └── 900 deploy-script            ├── 900  ✝ (gone)
            └── 1200 node             └── 1200 node  ← RE-PARENTED. PPID 900 → 1.
```

The kernel walks the dead process's children and sets each PPID to **1** — or to the nearest ancestor marked a *subreaper* (`prctl(PR_SET_CHILD_SUBREAPER)`), which is exactly how systemd keeps a service's grandchildren inside its own cgroup, and how `containerd-shim` supervises containers.

PID 1's job includes `while (wait(NULL) > 0);` — reap forever. So an orphan is reaped the instant it exits. **Orphans do not become permanent zombies — provided PID 1 does its job.**

```bash
# PROVE IT: watch a process get re-parented
bash -c 'sleep 300 & echo "child: $!"' & sleep 1
ps -eo pid,ppid,cmd | grep '[s]leep 300'
#  5210     1 sleep 300     ← PPID is now 1. Adopted, not killed. Working as designed.
```

### PID 1 in Docker — the one that *will* bite you

The kernel treats PID 1 specially, and Docker makes *your app* PID 1.

**The special rule:** for PID 1, the kernel **does not apply default signal actions**. Normally a process with no SIGTERM handler gets the default action (terminate). For PID 1, a signal with no *installed handler* is **silently discarded** — the kernel refuses to let a stray signal accidentally kill init and take down the machine.

```
docker stop my-api
   ├─ Docker sends SIGTERM to PID 1 inside the container
   │     └─▶ Your Node app IS PID 1. Did it call process.on('SIGTERM', …)?
   │            NO  → the kernel DROPS the signal. Nothing happens. It keeps serving.
   │            YES → your handler runs. 
   ├─ Docker waits 10 seconds (default; change with `docker stop -t 30`)
   └─ Docker sends SIGKILL. Uncatchable. Dead mid-request.
        → in-flight HTTP requests dropped · DB transactions uncommitted · exit code 137
```

**Second PID 1 problem:** any orphan inside the container is re-parented to PID 1 = **your Node app**, which has no reaping loop. Those orphans become permanent zombies and the container slowly fills its PID table.

**Third, and sneakiest — the Dockerfile shell form:**

```dockerfile
# ❌ WRONG — shell form
CMD node server.js
#  → Docker actually runs:  /bin/sh -c "node server.js"
#  → PID 1 is /bin/sh.  node is PID 7, a CHILD.
#  → `docker stop` SIGTERMs /bin/sh, which does NOT forward signals to children.
#  → node NEVER SEES SIGTERM. Everything gets SIGKILLed 10s later.
#  → Your beautiful process.on('SIGTERM') handler is NEVER CALLED.

# ✅ RIGHT — exec form (JSON array). No shell. node IS PID 1.
CMD ["node", "server.js"]
```

This is a top-tier real-world bug: teams write a perfect graceful-shutdown handler, verify it locally with Ctrl-C (which works — that's SIGINT to a *foreground* process, not PID 1), deploy, and it never fires again.

**The three fixes — use at least two:**

```dockerfile
CMD ["node", "server.js"]                     # 1. Exec form. Always. Non-negotiable.
```
```js
process.on('SIGTERM', () => server.close(() => process.exit(0)));   // 2. Handle it.
```
```bash
docker run --init my-api      # 3. Docker injects `tini` as PID 1: it forwards
                              #    signals to your app AND reaps every orphan.
# Or bake it in:  ENTRYPOINT ["/usr/bin/dumb-init", "--"]
#                 CMD ["node", "server.js"]
```

With `--init`, PID 1 is tini, node is PID ~7 (a normal process with normal signal defaults), tini forwards SIGTERM and reaps orphans. Kubernetes still sends SIGTERM to PID 1 of the container, so all three rules apply there too.

---

## Process States — what `ps` STAT is telling you

| STAT | Name | Means | Killable? |
|---|---|---|---|
| `R` | Running/Runnable | On a CPU *or* queued for one. `R` does **not** mean "burning CPU". | yes |
| `S` | Interruptible Sleep | Waiting, and a signal can wake it. **This is 95% of all processes.** An idle Node app parked in `epoll_wait()` is `S`. Healthy. | yes |
| `D` | **UNINTERRUPTIBLE Sleep** | Blocked inside the kernel on I/O. Will not accept **any** signal because the kernel is mid-operation holding locks/DMA buffers. | **NO — not even -9** |
| `T` | Stopped | SIGSTOP / SIGTSTP (Ctrl-Z), or under ptrace. | yes |
| `Z` | Zombie / `<defunct>` | Exited, not yet reaped. | already dead |
| `I` | Idle kernel thread | A kernel worker doing nothing. | n/a |

**D state is the most valuable diagnostic here.** A D-state process is stuck inside a syscall waiting on hardware. `kill -9` does nothing: the signal bit is set, but it can only be *delivered* when the syscall returns — which is exactly what isn't happening. A pile of D-state processes means **your storage is stuck**: a hung NFS mount, a dead EBS volume, a failing disk. The fix is not in the process table.

```bash
# PROVE IT: find every process stuck on I/O, and WHICH kernel function it's stuck in
ps -eo pid,ppid,stat,wchan:25,cmd | awk '$3 ~ /^D/'
#   PID  PPID STAT WCHAN                     CMD
#  8123     1 D    io_schedule               node /srv/app/server.js
#  9001   900 D+   nfs_wait_bit_killable     ls /mnt/shared
#                  ^^^^^^^^^^^^^^^^^^^^^^^^^ your NFS server is gone.
#                  That single word IS the diagnosis.
```

**STAT modifiers** append to the letter: `<` high priority (negative nice) · `N` low priority · `L` pages locked in RAM · `s` **session leader** (job control, topic 17) · `l` multi-threaded (Node always) · `+` in the **foreground process group** of a terminal.

So `Ssl+` on a node line = sleeping, session leader, multi-threaded, foreground. `Ssl` with **no `+`** = a daemon with no terminal. That one character tells you whether it survives your SSH session closing.

---

## /proc/PID — the entire process, as text files

`/proc` is not a disk. It is a **virtual filesystem**: the kernel generating text on demand from `task_struct`. Reading `/proc/4831/status` executes kernel code. It is the most powerful debugging surface on a Linux box, and it needs **zero tools installed** — critical in a slim Alpine container with no `htop`, no `lsof`, maybe not even `ps`.

```
/proc/4831/
├── cmdline    the exact argv, NUL-separated (not spaces!)
├── environ    the FULL environment the running process actually got
├── exe        symlink → the binary on disk
├── cwd        symlink → its working directory
├── root       symlink → its root dir (differs inside containers)
├── fd/        one symlink per open file descriptor   ◀ TOPIC 19
├── status     human-readable: state, PPid, UIDs, Threads, VmRSS, signal masks
├── stat       machine-readable, 52 fields (this is what `ps` parses)
├── maps       the ENTIRE virtual address space, line by line
├── smaps      maps + per-region RSS/PSS/dirty-page accounting
├── limits     soft/hard rlimits (max open files, stack size, core size)
├── task/      one subdirectory per THREAD (TID)
├── oom_score  0–1000: how attractive this process is to the OOM killer
├── oom_score_adj  −1000..+1000: your thumb on the scale
└── io         bytes this process has actually read/written
```

```bash
PID=$(pgrep -n node)

# cmdline — args are NUL-separated, so plain `cat` looks mangled. Translate:
tr '\0' ' ' < /proc/$PID/cmdline; echo
# node /srv/app/server.js --max-old-space-size=1024

# environ — THE production trick: what env did the RUNNING process actually get?
# (Did systemd really pass DATABASE_URL? This is the only way to know for certain.)
tr '\0' '\n' < /proc/$PID/environ | sort | head -4
# HOME=/home/deploy
# NODE_ENV=production
# PORT=3000
#  ⚠ needs root or the same uid. Secrets are plainly visible here — which is exactly
#    why "secrets in env vars" is convenient, not secure.

ls -l /proc/$PID/{exe,cwd,root}
# /proc/4831/cwd  -> /srv/app
# /proc/4831/exe  -> /usr/bin/node
# /proc/4831/root -> /
#  ⚠ exe still resolves after the binary is DELETED — it prints "(deleted)". That is
#    how you catch "we deployed over a running process and it's still on the old code."

grep -E '^(Name|State|Tgid|PPid|Uid|Threads|VmSize|VmRSS|SigCgt)' /proc/$PID/status
# Name:    node
# State:   S (sleeping)
# Tgid:    4831              ← thread-group id == the PID userspace sees
# PPid:    1                 ← started by systemd, or orphaned and re-parented
# Uid:     1001 1001 1001 1001     ← real, effective, saved, filesystem
# Threads: 11                ← main + 4 libuv + V8 workers. Normal.
# VmSize:  11284736 kB       ← VIRTUAL. ~11 GB. Terrifying. Meaningless. (topic 17)
# VmRSS:   84320 kB          ← RESIDENT. 84 MB. This is the real RAM.
# SigCgt:  0000000180014202  ← bitmask of the signals it has HANDLERS for

grep 'open files' /proc/$PID/limits     # the answer to EMFILE / "too many open files"
# Max open files   1024    4096    files
#                  ^soft (what applies)  ^hard (what you may raise it to)

cat /proc/$PID/io | head -2      # is it actually doing disk work?
cat /proc/$PID/oom_score         # 671 ← high. It's the biggest thing on the box. (17)
```

### `/proc/self/maps` — proving the address space is real

`/proc/self` is a magic symlink to *whichever process is reading it* — so `cat /proc/self/maps` shows `cat`'s own map.

```
5581b0e2c000-5581b0e31000 r-xp 00002000 08:01 1835023  /usr/bin/cat      ← TEXT (code)
5581b0e35000-5581b0e36000 rw-p 0000a000 08:01 1835023  /usr/bin/cat      ← DATA
5581b1f4c000-5581b1f6d000 rw-p 00000000 00:00 0        [heap]            ← HEAP (grows up)
7f2a4c028000-7f2a4c1b0000 r-xp 00028000 08:01 1573312  /usr/lib/.../libc.so.6
7f2a4c202000-7f2a4c20f000 rw-p 00000000 00:00 0                          ← anonymous (malloc)
7ffd3c1a1000-7ffd3c1c2000 rw-p 00000000 00:00 0        [stack]           ← STACK (grows down)
7ffd3c1f9000-7ffd3c1fb000 r-xp 00000000 00:00 0        [vdso]            ← kernel fast-path
```

**Every column, decoded:**

```
5581b0e2c000-5581b0e31000 r-xp 00002000 08:01 1835023   /usr/bin/cat
└────────┬────────────┘   └┬─┘ └───┬──┘  └┬─┘ └──┬──┘   └─────┬─────┘
         │                 │       │       │      │            │
         │                 │       │       │      │            └─ PATHNAME, or a pseudo-name
         │                 │       │       │      │               [heap] [stack] [vdso], or
         │                 │       │       │      │               BLANK = anonymous memory
         │                 │       │       │      │               (malloc'd / the V8 heap)
         │                 │       │       │      └─ INODE of the backing file (topic 04).
         │                 │       │       │         0 = no file behind it.
         │                 │       │       └─ DEVICE major:minor holding that file.
         │                 │       │          00:00 = not on a device (anonymous).
         │                 │       └─ OFFSET into that file where this mapping begins
         │                 └─ PERMISSIONS  r w x  and  p/s
         │                       r-xp = read+execute, PRIVATE (copy-on-write)
         │                       rw-p = read+write, private
         │                       r--s = SHARED — writes are visible to other processes
         │                              mapping the same file
         └─ ADDRESS RANGE in this process's VIRTUAL address space.
            0x5581b0e31000 − 0x5581b0e2c000 = 0x5000 = 20480 bytes = 5 pages.
```

This immediately explains three things: **your code is `r-xp`, never writable** (self-modifying code faults; JITs like V8 map their own regions, which is what those huge anonymous blocks are); **the addresses change every run** (ASLR — run it twice); and **Node's V8 heap is anonymous `rw-p` mappings**, which is why it appears as a giant blob with no filename.

---

## Exact Syntax Breakdown

```
ps -ef --forest
│  │ │  │
│  │ │  └── draw ASCII art showing the parent/child TREE
│  │ └── full format: UID PID PPID C STIME TTY TIME CMD
│  └── every process (all users, including daemons with no terminal)
└── "process status" — the UNIX-syntax form (flags take a dash)

pstree -paul 4831
│      ││││  └── root the tree at this PID (omit → root at PID 1)
│      │││└── -l : long lines, don't truncate
│      ││└── -u : show the username when it CHANGES (spot privilege drops)
│      │└── -a : show command-line ARGUMENTS
│      └── -p : show PIDs
└── render the process tree

ps -o pid,ppid,stat,wchan:20,etime,cmd -p 4831
│  │                    │      │
│  │                    │      └── ELAPSED wall-clock time since it started
│  │                    └── which KERNEL FUNCTION it is sleeping in, 20 chars wide
│  │                        (`:N` sets a column's width) ← the D-state diagnosis
│  └── -o : exactly the columns you want, in the order you want
└── -p : only this PID

strace -f -e trace=process -o /tmp/trace.log ./deploy.sh
│      │  │                 └── write to a file (stderr would collide with the script)
│      │  └── only process-lifecycle syscalls: clone, fork, execve, exit_group, wait4
│      └── -f : FOLLOW forks. Without this you trace the parent only and learn nothing.
└── print every syscall a process makes
```

---

## Example 1 — Basic

```bash
echo "my shell is PID $$"          # $$ = this shell's own PID  →  3092
ps -o pid,ppid,cmd -p $$           # who is my parent?
#  3092  3088 -bash
ps -o pid,ppid,cmd -p 3088         # ...and who is THAT? Walk the chain up.
#  3088     1 sshd: deploy@pts/0    ← its parent is PID 1. Everything roots at 1.

sleep 500 &                        # start a child;  $! = its PID  →  3120
ps -o pid,ppid,stat,cmd -p $!      # prove the parent/child link exists in the kernel
#  3120  3092 S    sleep 500        ← PPID 3092 == our shell. S = sleeping. Correct.
cat /proc/$!/maps | wc -l          # its private address space: 22 regions, for a
                                   # program that does nothing

kill 3120; ls /proc/3120/          # prove /proc is LIVE, not a snapshot
# ls: cannot access '/proc/3120/': No such file or directory
#   ← the task_struct is gone, so the directory is gone. /proc IS the kernel.
```

---

## Example 2 — Production Scenario

**02:40 AM.** PagerDuty: `api-prod-3` — health checks failing, memory alarm. You SSH in.

```bash
$ uptime
 02:41:07 up 18 days,  4:02,  1 user,  load average: 8.44, 6.10, 3.55
#                                                     ^^^^ 4-core box. 2× overloaded.

# Step 1 — What does the tree actually look like? Is anything duplicated?
$ ps -ef --forest | grep -A6 -E 'init|pm2|node' | head
UID      PID  PPID  C STIME TTY   TIME     CMD
root       1     0  0 Jun24 ?     00:04:12 /sbin/init
root     901     1  0 Jun24 ?     00:11:30  \_ /usr/bin/pm2 God Daemon
deploy  9142   901 92 02:12 ?     00:26:41  |   \_ node /srv/app/server.js
deploy 14201  9142  0 02:31 ?     00:00:00  |       \_ [sharp] <defunct>
deploy 14202  9142  0 02:31 ?     00:00:00  |       \_ [sharp] <defunct>
deploy 14203  9142  0 02:31 ?     00:00:00  |       \_ [sharp] <defunct>
#                                                       ^^^^^^^^^^^^^^^^ ZOMBIES

# Step 2 — How many? One is fine. Thousands is an outage.
$ ps -eo stat | grep -c '^Z'
3187

# Step 3 — Confirm the PID table is filling. THIS is the real danger.
$ cat /proc/sys/kernel/pid_max
32768
$ ls /proc | grep -c '^[0-9]'
3402
#  ← 3402/32768 PIDs consumed and climbing ~100/min. In ~5 hours fork() starts
#    failing with EAGAIN and NOTHING can start — not even a shell.

# Step 4 — Is the real worker healthy?
$ ps -p 9142 -o pid,ppid,stat,%cpu,rss,etime,cmd
    PID   PPID STAT %CPU     RSS   ELAPSED CMD
   9142    901 Rsl+ 92.4 1841236  00:29:12 node /srv/app/server.js
#                    ^^^^ pegged   ^^^^^^^ 1.8 GB resident and climbing

# Step 5 — Stuck on I/O, or genuinely running? (R vs D — the critical distinction)
$ ps -eo pid,stat,cmd | awk '$2 ~ /^D/'
# (nothing) ← no D-state. Not an I/O hang. It really is burning CPU and leaking children.

# Step 6 — What is it doing at the syscall level, right now?
$ sudo strace -c -f -p 9142 2>&1 | head -6
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 44.11    0.882012          21     41412           clone
 22.03    0.440503          10     41398           wait4
#  ← 41,412 clone() calls. It is forking like mad and not keeping up with reaping.

# Step 7 — What env is the RUNNING process actually using?
$ sudo tr '\0' '\n' < /proc/9142/environ | grep -E 'NODE_ENV|CONCURRENCY'
NODE_ENV=production
IMAGE_CONCURRENCY=64        ← 🔴 THERE IT IS.

$ grep IMAGE_CONCURRENCY /srv/app/.env
IMAGE_CONCURRENCY=4         ← the FILE says 4. The RUNNING PROCESS says 64.
#  ← a stale override in the pm2 env that was never reloaded. THIS is exactly why you
#    read /proc/PID/environ instead of trusting the .env file.
```

**Diagnosis:** a config override set image concurrency to 64. The app forks 64 `sharp` workers per job, its `'exit'` handling can't keep up, orphaned workers pile up as zombies, RSS climbs, CPU pegs.

```bash
# FIX 1 — stop the bleeding. Kill the PARENT. Its 3187 zombies re-parent to PID 1,
#         which reaps them instantly. All 3187 vanish in about a second.
$ sudo pm2 restart api          # graceful: SIGTERM → drain → restart
$ ps -eo stat | grep -c '^Z'
0
#  ← zombies gone. Not because you killed them — because their NEW parent reaped them.

# FIX 2 — reload with the correct env, and verify against the RUNNING process.
$ sudo pm2 delete api && sudo pm2 start /srv/app/server.js --name api
$ sudo tr '\0' '\n' < /proc/$(pgrep -n node)/environ | grep CONCURRENCY
IMAGE_CONCURRENCY=4   ✅
```

**The lesson:** you did not kill a single zombie. You fixed the parent. That is the only thing that ever works.

---

## Common Mistakes

### Mistake 1 — Trying to `kill -9` a zombie

**Wrong:** `sudo kill -9 14201` → run `ps` again → it's still there → repeat → rage.

**Root cause:** SIGKILL tells the kernel "destroy this process's execution context." A zombie *has* no execution context — no memory, no code, no stack, and it will never be scheduled again. All that's left is a row in the process table, and **only a `wait()` from the parent removes that row.** There is nothing for SIGKILL to act on.

**Right:**
```bash
ps -o ppid= -p 14201          # find the PARENT — that's who is failing to reap
#  9142
ps -p 9142 -o pid,cmd         # identify it before you touch it
sudo systemctl restart api    # fix/restart the parent; its zombies re-parent to PID 1
```

**Prevention:** in Node, never fire-and-forget a child:
```js
const child = spawn('convert', args);
child.on('exit',  (code, signal) => { /* reaped */ });
child.on('error', (err) => { /* spawn itself failed */ });
```

---

### Mistake 2 — Shell-form `CMD` in a Dockerfile

**Wrong: `CMD node server.js`**
```bash
$ docker exec api ps -ef
PID  PPID  CMD
  1     0  /bin/sh -c node server.js     ← PID 1 is a SHELL
  7     1  node server.js                ← your app is a CHILD
$ docker stop api                        # hangs the full 10s, then hard-kills
$ docker inspect api --format '{{.State.ExitCode}}'
137                                      ← 128+9 = SIGKILL. No graceful shutdown.
```

**Right: `CMD ["node", "server.js"]`**
```bash
$ docker exec api ps -ef
PID  PPID  CMD
  1     0  node server.js                ← your app IS PID 1
$ docker stop api                        # returns in 0.4s
$ docker inspect api --format '{{.State.ExitCode}}'
0                                        ← clean. server.close() ran. Pool drained.
```

**Root cause:** shell form wraps your command in `/bin/sh -c`, making `sh` PID 1. Docker signals PID 1. `sh` is not an init: it does not forward signals to children and it does not reap. Your node at PID 7 never receives SIGTERM.

**Prevention:** exec form always, **plus** a real `process.on('SIGTERM')` handler, **plus** `docker run --init`. Belt, braces, and a second pair of braces.

---

### Mistake 3 — "My Node app is using 11 GB of RAM!"

**Wrong:** reading `VSZ` (or `VmSize`) and panicking.
```bash
$ ps aux | grep node
deploy 4831 0.7 1.0 11284736  84320 ? Ssl Jul11 12:04 node server.js
#                    ^^^^^^^^  ^^^^^
#                    VSZ 11 GB  RSS 84 MB
```

**Right:** `VSZ` is *virtual address space* — every region ever mapped, including huge regions V8 reserves up front and never touches, plus every shared library and every guard page. It is not RAM. **RSS is RAM.** 84 MB is the real number. (Full treatment in topic 17.)

**Root cause:** on a 64-bit machine, *reserving* address space costs nothing but a page-table entry. V8 reserves enormous contiguous regions deliberately.

**Prevention:** `ps -eo pid,rss,cmd --sort=-rss | head`. Sort by RSS. Never by VSZ.

---

### Mistake 4 — Assuming a dead shell kills its children

**Wrong:** `./long-migration.sh &` over SSH, then `exit` and go home. Next morning: half-done and gone.

**Root cause:** two different mechanisms, and you must know which bit you. **(1)** When the controlling terminal disappears, the kernel sends **SIGHUP** to the foreground process group (topic 17), and bash HUPs its background jobs on exit. **(2)** Even if it survives, it becomes an **orphan** re-parented to PID 1 — which is fine and expected, not a bug.

**Right:**
```bash
nohup ./long-migration.sh > /var/log/migration.log 2>&1 &   # immune to SIGHUP
tmux new -s migration                                        # better, for interactive (15)
sudo systemd-run --unit=migration ./long-migration.sh        # best, for anything real (22)
```

**Prevention:** anything that must outlive your SSH session belongs to systemd or pm2, not to your shell.

---

### Mistake 5 — Confusing threads with processes

**Wrong:** `ps -eLf | grep -c node` → `11` → *"we're leaking Node processes!"*

**Right:**
```bash
$ ps -e | grep -c node
1                                          # ONE process.
$ ls /proc/$(pgrep -n node)/task | wc -l
11                                         # ELEVEN threads. -L = threads.
# 1 main + 4 libuv threadpool (UV_THREADPOOL_SIZE=4) + V8 GC/compiler workers. Normal.
```

**Root cause:** on Linux a thread *is* a task with its own TID and `task_struct`. Tools that list **tasks** (`ps -eLf`, `top -H`, `htop` with `H`) show every thread. Tools that list **thread groups** (`ps aux`) show one line.

**Prevention:** know which view you're in. `%CPU` in `top -H` is per-thread; in plain `top` it's summed across the process — which is why a busy Node process can legitimately show >100%.

---

## Hands-On Proof

```bash
# PROVE IT: the shell forks and execs for every external command
strace -f -e trace=clone,execve,wait4 -o /tmp/t.log bash -c 'ls > /dev/null'
grep -E 'clone|execve|wait4' /tmp/t.log | head -4
# execve("/bin/bash", ...)                ← bash itself starts
# clone(...) = 5501                       ← bash FORKS
# [pid 5501] execve("/usr/bin/ls", ...)   ← the CHILD becomes ls
# wait4(-1, ...) = 5501                   ← bash WAITS for it

# PROVE IT: exec REPLACES the process (same PID, new program) and KEEPS open fds —
#           which is precisely WHY redirection survives exec (topic 12)
bash -c 'exec > /tmp/proof.txt; echo "went to a file"; exec cat /etc/hostname'
cat /tmp/proof.txt
# went to a file
# myserver          ← `cat` inherited fd 1 pointing at the file, ACROSS the exec

# PROVE IT: make a zombie with your own hands, then watch PID 1 reap it
bash -c 'sleep 1 & sleep 30' &          # parent ignores its dead child for 30s
sleep 3; ps -eo pid,ppid,stat,cmd | grep -E 'Z +\[' 
#  6402  6401 Z    [sleep] <defunct>    ← a real zombie you made
kill 6401; sleep 1; ps -eo stat,cmd | grep defunct
# (nothing)   ← gone. You never touched the zombie. PID 1 inherited and reaped it.

# PROVE IT: your address space is real, and randomized (ASLR)
cat /proc/self/maps | grep -E '\[heap\]|\[stack\]'
cat /proc/self/maps | grep -E '\[heap\]|\[stack\]'
#   ← different addresses each run. That's ASLR.

# PROVE IT: memory is per-process and the kernel enforces it
cat /proc/1/mem
# cat: /proc/1/mem: Permission denied     ← process isolation, enforced in ring 0
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
echo $$                                  # your shell's PID
ps -o pid,ppid,stat,cmd -p $$            # its parent and state
pstree -p $$                             # the tree rooted at your shell
sleep 200 &
ps -o pid,ppid,stat,cmd -p $!
grep -E 'State|PPid|Threads|VmRSS' /proc/$!/status
kill $!; ps -p $!
```
**Answer from your output:** What is your shell's PPID, and what is that process? What STAT is `sleep` in, and why *that* state and not `R`? How many threads does `sleep` have? What is its VmRSS and why so small?

### Exercise 2 — Medium

Create a zombie deliberately, then make it disappear **without killing it**.

```bash
cat > /tmp/zombie.sh <<'EOF'
#!/bin/bash
sleep 1 &                 # this child dies after 1 second
echo "parent PID $$, child PID $!"
sleep 120                 # ...and I never call wait(). For two whole minutes.
EOF
chmod +x /tmp/zombie.sh && /tmp/zombie.sh &

sleep 3
ps -eo pid,ppid,stat,rss,vsz,cmd | grep -E 'Z|defunct'   # find it
kill -9 <ZOMBIE_PID>                                      # try to kill it
ps -eo pid,stat,cmd | grep defunct                        # still there?
kill <PARENT_PID>; sleep 1                                # kill the PARENT instead
ps -eo pid,stat,cmd | grep defunct                        # and now?
```
**Answer:** What are the zombie's RSS and VSZ, and what does that prove? Why did `kill -9` do nothing? Exactly *which process* reaped the zombie in the final step, and what syscall did it call to do it?

### Exercise 3 — Hard (Production Simulation)

Reproduce and fix the Docker PID 1 bug end to end.

```bash
mkdir -p /tmp/pid1 && cd /tmp/pid1
cat > server.js <<'EOF'
const http = require('http');
const server = http.createServer((_, res) => res.end('ok\n'));
server.listen(3000, () => console.log('listening, pid', process.pid));
process.on('SIGTERM', () => {
  console.log('SIGTERM received — draining');
  server.close(() => { console.log('closed cleanly'); process.exit(0); });
});
EOF
printf 'FROM node:20-alpine\nWORKDIR /app\nCOPY server.js .\nCMD node server.js\n' > Dockerfile
docker build -t pid1-broken . && docker run -d --name broken -p 3000:3000 pid1-broken

# INVESTIGATE BEFORE YOU FIX. Answer each with a command:
docker exec broken ps -ef            # what is PID 1? what PID is node?
time docker stop broken              # how long did it take?
docker inspect broken --format '{{.State.ExitCode}}'
docker logs broken                   # did "SIGTERM received" EVER print?

# Now: (a) fix it by changing ONE line in the Dockerfile, rebuild as pid1-fixed,
#      and re-run all four commands.
#      (b) Then run the ORIGINAL broken image with `docker run -d --init ...`
#      and re-run all four commands again.
```
**Deliverable:** a table with rows `broken` / `fixed` / `broken + --init` and columns *PID 1 is* · *node's PID* · *stop duration* · *exit code* · *did the handler run*. Then: one sentence on why `--init` rescues the broken image, and one reason you'd still fix the `CMD` anyway.

---

## Mental Model Checkpoint

1. **What exactly is the difference between a program and a process?** Name three things a process has that a file on disk does not.
2. **Draw a process's virtual address space from memory.** Which way does the heap grow? The stack? Where do shared libraries live, and why is the text segment read-only *and* shared?
3. **Why doesn't `fork()` on a 2 GB Node process copy 2 GB?** What exactly happens the first time the child writes to a page?
4. **After `execve()`, what is preserved and what is destroyed?** (Three of each.) Why does that make shell redirection possible?
5. **A process shows `Z` with RSS 0.** Why can't `kill -9` remove it, and what are the two things that actually will?
6. **Your Node app is PID 1 in a container, has a SIGTERM handler, and `docker stop` still takes 10s and exits 137.** Give two independent explanations and the fix for each.
7. **You see 40 processes in `D` state.** What does D mean, why won't `kill -9` clear them, and where do you look next?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `ps -ef --forest` | Full listing as a process tree | `--forest` tree, `-L` threads |
| `pstree -p` | Process tree with PIDs | `-p` PIDs, `-a` args, `-u` user changes |
| `pgrep -n node` | PID of the newest match | `-n` newest, `-f` full cmdline, `-l` show name |
| `ps -p PID -o pid,ppid,stat,rss,etime,cmd` | Custom-column view of one PID | `-o` columns, `:N` width |
| `ps -eLf` | Every **thread** (`LWP` = TID) | `-L` |
| `/proc/PID/cmdline` | Exact argv (NUL-separated) | `tr '\0' ' '` |
| `/proc/PID/environ` | The env the **running** process actually got | `tr '\0' '\n'`; needs root/same-uid |
| `/proc/PID/status` | State, PPid, UIDs, Threads, VmRSS, signal masks | grep it |
| `/proc/PID/maps` | The entire virtual address space | `/proc/self/maps` for your own |
| `ls -l /proc/PID/{exe,cwd,root}` | Binary, working dir, root dir | catches `(deleted)` |
| `ls /proc/PID/fd` | Every open file descriptor | topic 19 |
| `/proc/PID/limits` | rlimits — max open files, stack size | the EMFILE answer |
| `ps -eo pid,stat,wchan:25,cmd` | Which kernel function it's sleeping in | the D-state diagnosis |
| `strace -f -e trace=process CMD` | Trace fork/exec/exit only | `-f` follow children, `-c` summary, `-p` attach |
| `docker run --init` | Inject `tini` as PID 1 (forwards signals, reaps) | or `ENTRYPOINT ["dumb-init","--"]` |

---

## When Would I Use This at Work?

### Scenario 1: "We deployed, but the old code is still serving"

`systemctl status api` says active, yet responses are stale. Don't guess — ask the running process:
```bash
$ ls -l /proc/$(pgrep -n node)/cwd
lrwxrwxrwx ... /proc/9142/cwd -> /srv/releases/2024-06-30 (deleted)
#                                                         ^^^^^^^^^
```
`(deleted)` — its working directory is a release folder the deploy script already removed. It never restarted, and it's holding the old inode open (topic 04: the directory entry is gone; the inode survives while a process references it). `systemctl restart api`, then add a restart step to the deploy.

### Scenario 2: "Kubernetes keeps restarting my pod, exit code 137"

You know instantly: 137 = 128 + 9 = **SIGKILL**. Two candidates — the OOM killer (memory limit exceeded) or the kubelet hard-killing you after `terminationGracePeriodSeconds` expired. Check `kubectl describe pod` for `OOMKilled`. If it's absent, it's the grace period — which means your app isn't handling SIGTERM, which sends you straight to: *is my `CMD` exec form, and is `process.on('SIGTERM')` registered?*

### Scenario 3: "Load average is 40 but the CPU is idle"

Load counts **R *and* D** tasks. High load with idle CPU means a wall of processes stuck in uninterruptible I/O:
```bash
$ ps -eo pid,stat,wchan:30,cmd | awk '$2 ~ /^D/' | head -2
 4102 D    nfs_wait_bit_killable   node /srv/app/worker.js
 4103 D    nfs_wait_bit_killable   node /srv/app/worker.js
```
Every worker is blocked in the NFS client. `kill -9` cannot help — the signal can't be delivered until the syscall returns. The problem is the NFS server, not your app. Fix the mount and 40 processes come back to life at once.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 17 — Process Management | You know what a process *is*; next you learn to inspect and control it: `ps`, `top`, signals, `kill`, job control, the OOM killer |
| **Builds on** | 01 — What Linux Actually Is | Kernel vs userspace, ring 0/3, the syscall boundary — the process is the kernel's unit of isolation |
| **Builds on** | 07 — How the Shell Works | The fork-exec model was introduced there; here you learn what fork and exec *do to memory* |
| **Builds on** | 05/06 — Permissions, Users & Groups | uid/euid/gid live in `task_struct`; the kernel checks *the process's* credentials on every access |
| **Builds on** | 12 — Pipes and Redirection | Redirection survives `exec` because fds live in the process, not the program |
| **Builds on** | 13 — Shell Scripting | `$?` is the `wait()` status flattened. 128+N means signal N killed it. |
| **Used by** | 19 — File Descriptors | `/proc/PID/fd/` is the fd table from `task_struct`, rendered as symlinks |
| **Used by** | 18 — System Resources | RSS, VSZ, load average — per-process values from this doc, aggregated |
| **Used by** | 22 — systemd | systemd IS PID 1: it reaps orphans, tracks services as process trees in cgroups, sends SIGTERM then SIGKILL |
| **Used by** | 33 — Node in Production | Graceful shutdown, pm2 cluster mode, and the Docker PID 1 problem are direct applications |
| **Used by** | 34 — Performance Investigation | `strace -f`, `/proc/PID/*`, D-state hunting and OOM forensics all start here |
