# 17 — Process Management

## ELI5 — The Simple Analogy

Picture a **hospital ward**.

- **`ps`** is the clipboard at the foot of a bed: a one-time snapshot. You read it and walk away; the moment you turn around it's slightly out of date.
- **`top` / `htop`** is the wall of monitors at the nurses' station: the same data, but *live*, refreshing every second, sorted so the sickest patient is on top.
- **Signals** are how you talk to a patient — a whole vocabulary of force:
  - **SIGTERM** is a nurse tapping your shoulder: *"We need this bed. Please pack, settle your bill, say goodbye, walk out."* You finish what you're doing. You can even say "one minute."
  - **SIGKILL** is two guards physically carrying you out mid-sentence. You cannot argue. Your bag is in the room, your bill unpaid, your IV still in your arm. **This is why `kill -9` is a last resort.**
  - **SIGHUP** originally meant *"the phone line to your room went dead."* Daemons repurposed it to mean *"re-read your chart."*
  - **SIGSTOP** is a freeze ray — paused mid-step, not dead. **SIGCONT** un-pauses you.
- The **OOM killer** is the administrator during a bed shortage: when the ward is full they remove the patient taking up the *most space*. Not the sickest. Not the newest. The **biggest** — which, on your server, is your Node app.

The whole discipline: *read the ward, then use the right level of force.*

---

## Where This Lives in the Linux Stack

```
Hardware
  └── KERNEL
       │   ┌────────────────────────────────────────────────────────┐
       │   │  • the scheduler (nice/priority decide who runs)        │
       │   │  • SIGNAL delivery  ◀◀◀ THIS TOPIC                      │
       │   │      pending mask · blocked mask · handler table        │
       │   │  • the OOM killer   ◀◀◀ THIS TOPIC                      │
       │   │  • /proc — the source of everything ps/top print        │
       │   └────────────────────────────────────────────────────────┘
       └── System Calls   kill(2) tgkill(2) rt_sigaction(2) setpriority(2)
            │             ◀◀◀ every command below is a thin wrapper on these
            └── C Library (glibc: kill(), sigaction(), nice())
                 └── Shell (job control: & Ctrl-Z jobs fg bg disown)  ◀◀◀ THIS TOPIC
                      └── Commands: ps top htop kill pkill killall nice renice
                                    ◀◀◀ they just READ /proc and CALL kill()
```

Nothing here is magic. `ps` formats `/proc/*/stat`. `kill -9 4831` calls `kill(4831, SIGKILL)`. `top` does both in a loop.

---

## What Is This?

Process management is **observing** what runs (`ps`, `top`, `htop`) and **influencing** it (`kill`, signals, `nice`, job control). Observation is reading `/proc`; influence is sending signals and adjusting scheduler priority.

A **signal** is an asynchronous notification the kernel delivers to a process. For each signal a process can install a **handler**, **ignore** it, **block** it, or take the **default action** (usually: die). Two signals — SIGKILL and SIGSTOP — bypass all of that; the kernel enforces them itself.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| SIGTERM vs SIGKILL | You'll `kill -9` on every deploy, dropping in-flight requests and leaving DB transactions and lock files behind |
| Graceful shutdown | Every deploy = a burst of 502s. Users see them. Your error budget burns for nothing. |
| VSZ vs RSS | You'll file a bug because "node is using 11 GB" when it's using 84 MB |
| `%Cpu(s)` `wa` / `st` | You'll optimize code on a box whose problem is a saturated disk (`wa`) or a noisy hypervisor neighbour (`st`) |
| The OOM killer | Exit code 137 stays a mystery instead of a one-second diagnosis |
| `killall` semantics | `killall node` on a shared box takes down three other teams' services |
| `pkill -f` | You'll match your own `pkill` command, or more than you meant, and kill the wrong thing |
| Job control vs a supervisor | You'll `nohup` a service, close SSH, and have no logs, no restart-on-crash, no clean stop |

---

## The Physical Reality

### What a signal actually is, in `task_struct`

A signal is not a message queue. It is **bits**.

```
┌──────────── task_struct (kernel, per process) ────────────┐
│  sighand → ACTION TABLE — one slot per signal (1..64)      │
│    [ 1 SIGHUP ] → SIG_DFL   (terminate)                    │
│    [ 2 SIGINT ] → 0x4012a0  (your handler function)        │
│    [ 9 SIGKILL] → cannot be changed — sigaction() → EINVAL │
│    [15 SIGTERM] → 0x401350  (your handler function)        │
│  blocked = ...0010000   ← bitmask of signals you've MASKED  │
│  pending = ...0000000   ← delivered but not yet acted on   │
└────────────────────────────────────────────────────────────┘
```

Delivery, step by step:

```
1. Someone calls kill(4831, SIGTERM) — the shell, systemd, Docker, another process.
2. Kernel permission check: same UID, or root? (else EPERM)
3. Kernel sets bit 15 in the target's `pending` mask.  ← that is ALL "sending" is.
4. The target is NOT interrupted mid-instruction. Nothing happens yet.
5. Next time the target is about to RETURN TO USERSPACE (after a syscall, an
   interrupt, or a scheduler tick), the kernel checks: any (pending & ~blocked) bits?
6. Bit 15 set and not blocked → consult the ACTION TABLE:
     • handler installed → push a frame, jump to your handler IN USERSPACE
                           (your JS SIGTERM callback runs here)
     • SIG_IGN           → clear the bit, do nothing
     • SIG_DFL           → default action (SIGTERM → terminate)
```

**Two huge consequences fall straight out of step 5:**

1. **A `D`-state process never reaches step 5** — it's blocked deep in a syscall and won't return to userspace. That's why you can't kill it, not even with -9. The bit is set; delivery never happens.
2. **SIGKILL and SIGSTOP skip the table.** The kernel handles them in the scheduler without ever running your code. No slot to override. Your cleanup does not run. Your buffers aren't flushed. Your `finally` does not execute.

```
   kill -15 (TERM) ──▶  can be caught / ignored / blocked  ──▶  your handler runs
   kill -9  (KILL) ──▶  KERNEL DESTROYS THE PROCESS         ──▶  nothing runs. ever.
```

### The signal table you must know cold

| # | Name | Default | Catchable? | What it means to you |
|---|---|---|---|---|
| 1 | **SIGHUP** | Terminate | yes | Historically "the modem hung up." Kernel sends it to a session's foreground group when the controlling terminal closes — **this is why your process dies when you close SSH.** By convention daemons repurpose it: **"reload config without restarting"** (`nginx -s reload`, logrotate postrotate, `systemctl reload`). |
| 2 | **SIGINT** | Terminate | yes | **Ctrl-C.** Polite interrupt. `process.on('SIGINT')` in Node. |
| 3 | SIGQUIT | Terminate + **core** | yes | **Ctrl-\\**. Like INT but dumps core. |
| 9 | **SIGKILL** | Terminate | **NO — never** | Kernel destroys the process immediately. **No cleanup, no flush, no close, no unlink of the PID/lock file.** Last resort, always. |
| 11 | SIGSEGV | Terminate + core | (unwise) | Invalid memory access — a native crash in an npm addon. |
| 13 | **SIGPIPE** | Terminate | yes | Wrote to a pipe/socket whose reader is gone (topic 12). Why `node app \| head -5` can silently kill your app. Node ignores it and gives you `EPIPE` instead. |
| 15 | **SIGTERM** | Terminate | yes | **The default of `kill`.** The polite "please shut down." **What `systemctl stop`, `docker stop`, and Kubernetes send.** Catch it. Always. |
| 17 | SIGCHLD | **Ignore** | yes | "A child changed state" — the nudge to call `wait()` (topic 16). |
| 18/19 | **SIGCONT / SIGSTOP** | Continue / Stop | CONT yes, **STOP no** | Resume / freeze. SIGSTOP is uncatchable, like KILL. |
| 20 | SIGTSTP | Stop | yes | **Ctrl-Z** — the *catchable* stop (`less` catches it to restore your terminal). |
| 10/12 | **SIGUSR1 / SIGUSR2** | Terminate | yes | App-defined. **Node uses SIGUSR1 to open the debugger port** — `kill -USR1 <pid>` starts the inspector. |

**The escalation ritual — burn this in:**

```
   kill PID              (SIGTERM — polite)
        ├── wait 5–30s. WATCH it. Let it drain requests.
        ├── gone?  ✅  clean exit, $? = 0.
        └── STILL there after the grace period?
                 └── kill -9 PID   (SIGKILL — the guillotine)
                          → exit 137. Accept that state may be dirty.
```

Never start at -9. Reaching for it first is the clearest tell of someone who doesn't understand signals — and on a database or a mid-write Node process it can corrupt data or leave a stale lock file that blocks the next start.

---

## How It Works — Step by Step

### Trace: `kill -TERM 9142` on a Node app that handles SIGTERM

```
COMMAND:     kill -TERM 9142
SHELL DOES:  `kill` is a bash BUILTIN (also /bin/kill). No fork, no exec.
SYSCALL:     kill(9142, 15)
KERNEL DOES: permission check → set bit 15 in 9142's pending mask → wake it if sleeping
OUTPUT:      nothing. `kill` prints nothing on success. Silence means it worked.

Inside the Node process, microseconds later:
  1. The main thread wakes out of epoll_wait().
  2. Kernel sees pending bit 15 + a handler installed (libuv's), jumps to it.
  3. libuv's C handler just writes ONE BYTE to its self-pipe — it does NOT run JS
     (you can't safely run a JS engine from an async signal context).
  4. Back in the event loop, that byte is read and libuv emits the 'SIGTERM' event.
  5. YOUR callback finally runs, on the NORMAL event loop:
         server.close(...)  → stop accepting new connections
         drain in-flight requests → db.end() → process.exit(0)
  6. Kernel tears down the address space, leaves a zombie, SIGCHLDs the parent (16).
  7. Parent (systemd/pm2/docker) wait4()s it, reads exit status 0. Clean.
```

Note steps 3–5: **your handler does not run instantly. It runs when the event loop next turns.** If a synchronous 30-second CPU-bound loop is blocking the event loop, your SIGTERM handler won't run until it finishes — and Docker SIGKILLs you at 10 seconds. **Blocking the event loop breaks graceful shutdown.**

### Graceful shutdown in Node — the real thing

```js
const server = http.createServer(app);
let shuttingDown = false;
server.listen(3000);

function shutdown(signal) {
  if (shuttingDown) return;             // idempotent — SIGTERM can arrive twice
  shuttingDown = true;
  console.log(`${signal} received — graceful shutdown`);

  // 1. STOP ACCEPTING new connections; existing ones keep being served. The
  //    listening socket closes, the LB's next health check fails, we leave rotation.
  server.close(async () => {
    // 2. All in-flight requests done. Release resources.
    try { await pool.end(); await redis.quit(); process.exit(0); }
    catch (err) { console.error(err); process.exit(1); }
  });

  // 3. BACKSTOP: if a websocket or long query hangs, don't hang forever — exit on
  //    our own terms with a log line before the supervisor SIGKILLs us.
  setTimeout(() => { console.error('drain timed out'); process.exit(1); }, 10_000).unref();
}
process.on('SIGTERM', () => shutdown('SIGTERM'));   // docker stop / systemctl stop / k8s
process.on('SIGINT',  () => shutdown('SIGINT'));    // Ctrl-C in dev
```

**Tie the timeouts together. The app must be the fastest to give up:**

```
  Supervisor sends SIGTERM ──▶ waits ──▶ sends SIGKILL
    docker stop            default grace 10s   (docker stop -t 30 to change)
    systemd                TimeoutStopSec=90s (default)
    Kubernetes             terminationGracePeriodSeconds: 30 (default)

  RULE:  your app's drain backstop  <  the supervisor's grace period.
         10s backstop under 30s k8s grace  → you always exit yourself, code 0.
         30s backstop under 10s docker     → you ALWAYS get SIGKILLed, code 137.
```

Exit **143** (128+15) = SIGTERMed and *didn't handle it* (took the default). Exit **137** (128+9) = ignored SIGTERM or ran out of grace. **0** is what you want.

---

## `ps` — the snapshot

### Two syntax worlds, and why `ps aux` has no dash

`ps` implements both BSD and UNIX flag conventions:

```
 BSD style (NO dash)          UNIX style (dash)
   ps aux                       ps -ef
      │││                          ││
      ││└ x : include procs        │└ f : full-format (adds PPID, STIME…)
      ││    with NO controlling    └ e : every process
      ││    tty (daemons: nginx,
      ││    node under systemd)
      │└ u : user format (adds USER %CPU %MEM VSZ RSS)
      └ a : all users' processes
```

`ps aux` and `ps -ef` show the same processes, different columns: `aux` gives memory/CPU, `-ef` gives PPID, `-ef --forest` gives the tree. You'll see both everywhere.

### Every column of `ps aux`, annotated

```
USER   PID  %CPU %MEM     VSZ    RSS TTY  STAT START   TIME COMMAND
deploy 9142  1.3  1.0 11284736  84320 ?   Ssl  Jul11  12:04 node /srv/app/server.js
│      │     │    │    │        │      │   │    │      │     │
│      │     │    │    │        │      │   │    │      │     └ command + args. [in brackets]
│      │     │    │    │        │      │   │    │      │       = a kernel thread, or argv gone
│      │     │    │    │        │      │   │    │      │       (zombies: [x] <defunct>)
│      │     │    │    │        │      │   │    │      └ TIME = total CPU TIME CONSUMED
│      │     │    │    │        │      │   │    │        (user+sys), NOT wall-clock age
│      │     │    │    │        │      │   │    └ START = when it began (time today / date)
│      │     │    │    │        │      │   └ STAT = R/S/D/T/Z + modifiers (topic 16).
│      │     │    │    │        │      │     Ssl = Sleeping, session leader, multi-threaded.
│      │     │    │    │        │      │     trailing `+` = FOREGROUND process group.
│      │     │    │    │        │      └ TTY. `?` = NONE → a daemon, survives SSH exit.
│      │     │    │    │        │        `pts/0` = your SSH pty → dies on SIGHUP.
│      │     │    │    │        └ RSS = Resident Set Size (KB). ◀ THE REAL NUMBER.
│      │     │    │    │          Pages actually in physical RAM now. 84320 KB = 84 MB.
│      │     │    │    └ VSZ = Virtual Size (KB). ◀ MOSTLY MEANINGLESS. 11 GB of address
│      │     │    │      space, not RAM. (see below)
│      │     │    └ %MEM = RSS as a percentage of total physical RAM.
│      │     └ %CPU = CPU time ÷ time alive, over the WHOLE LIFETIME. ⚠ a lifetime
│      │       AVERAGE, not "right now" — for that, use `top`. Can exceed 100% (N cores).
│      └ PID
└ effective username (from euid — topic 06)
```

### VSZ vs RSS — the one that ends the "Node is eating 11 GB" panic

```
 VSZ (VIRTUAL SIZE)                     RSS (RESIDENT SET SIZE)
 ──────────────────────────────        ────────────────────────────────────
 Everything MAPPED, in RAM or not:      Pages actually present in PHYSICAL RAM.
   • every shared library in full         • the pages you've actually touched
   • memory RESERVED but never touched    • includes shared-lib pages that are in RAM
     (V8 reserves huge regions up front)  • EXCLUDES anything swapped out
   • file mappings never read
                                        ⚠ RSS DOUBLE-COUNTS SHARED PAGES: 8 nginx
 Reserving virtual address space on a     workers each count the libc pages in their
 64-bit box costs one page-table entry.   RSS, so SUMMING RSS over-counts total RAM.
 It is NOT RAM.
 Node routinely shows 5–12 GB VSZ       84 MB. The number that matters. What the OOM
 while using 84 MB. NOT A LEAK.         killer looks at.
```

For a number that doesn't double-count, use **PSS** (shared pages ÷ number of sharers):

```bash
# PROVE IT: PSS is the honest per-process figure
sudo awk '/^Pss:/{s+=$2} END{print s" kB PSS"}' /proc/$(pgrep -n node)/smaps
# 71204 kB PSS      ← lower than RSS, because shared libc pages are divided up
```

### The invocations you'll actually type

```
ps -eo pid,ppid,user,%mem,rss,etime,stat,cmd --sort=-%mem | head
│  │ │                         │             │
│  │ │                         │             └── top 10 only
│  │ │                         └── ELAPSED wall-clock since start (unlike TIME)
│  │ └── -o : pick exact columns, in order.   --sort=-FIELD : `-` prefix = descending
│  └── -e : every process
└── ps
```
```bash
ps -eo pid,ppid,user,%mem,rss,etime,cmd --sort=-%mem | head -4   # #1 memory triage
#   PID  PPID USER   %MEM    RSS    ELAPSED CMD
#  9142   901 deploy 22.4 1841236 1-04:12:33 node /srv/app/server.js  ← usual OOM victim
ps -eo pid,%cpu,etime,cmd --sort=-%cpu | head        # top CPU consumers
ps -p 9142 -o pid,ppid,user,stat,%cpu,rss,etime,cmd  # one PID, exact columns
ps -ef --forest | grep -A5 pm2                        # the tree (topic 16)
ps -eLf | grep node                                   # THREADS (LWP = TID)
pgrep -f "node.*server.js"    # -f : match the FULL command line, not just the name
pgrep -n node                 # -n : newest match     pgrep -l : also print the name
pidof node                    # all node PIDs, space-separated, one line
```

---

## `top` — the live view

```
top - 03:14:07 up 18 days,  4:02,  1 user,  load average: 8.44, 6.10, 3.55
Tasks: 187 total,   2 running, 184 sleeping,   0 stopped,   1 zombie
%Cpu(s): 42.1 us,  8.3 sy,  0.0 ni, 12.4 id, 35.9 wa,  0.0 hi,  0.3 si,  1.0 st
MiB Mem :  7976.0 total,  182.4 free, 7104.2 used,  689.4 buff/cache
MiB Swap:     0.0 total,    0.0 free,    0.0 used.   471.1 avail Mem

    PID USER     PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
   9142 deploy   20   0   11.2g   1.7g  38.4m R 92.4 22.4  26:41.03 node
   2201 postgres 20   0    1.1g 664.1m 612.0m S  4.0  8.1   9:12.44 postgres
```

**The header, line by line:**

```
LINE 1  load average: 8.44, 6.10, 3.55   ← 1-min, 5-min, 15-min
  = avg number of tasks in R (runnable) OR D (uninterruptible I/O). NOT a percentage.
    Compare to CORE COUNT (nproc): 4 cores + load 8.44 = 2× capacity, work is waiting.
    Because D counts: HIGH LOAD + IDLE CPU = an I/O hang, not a CPU problem (topic 16).
    Trend 8.44 > 6.10 > 3.55 = rising fast, getting worse.

LINE 2  Tasks: ... 1 zombie   ← sleeping is the HEALTHY norm (16). Watch `zombie` climb.

LINE 3  %Cpu(s):   ◀◀◀ THE MOST INFORMATION-DENSE LINE ON THE BOX
   us  USER    your code (node, postgres, nginx).
   sy  SYSTEM  kernel on your behalf: syscalls, network stack, page faults.
               persistently high sy = syscall storm (tiny reads, fork loops).
   ni  NICE    user time by niced processes.
   id  IDLE    doing nothing. THE NUMBER YOU WANT BIG.
   wa  IOWAIT ◀ CPU idle but BLOCKED on disk/network I/O. HIGH wa = STORAGE is the
               bottleneck, NOT your code. Optimizing JS does nothing; look at iostat,
               your DB, your EBS IOPS.
   hi  hardware IRQ.   si  softirq (mostly packet processing; high on a busy box is normal).
   st  STEAL ◀◀ time your vCPU wanted but the HYPERVISOR gave the core to someone else.
               st > 0 on a cloud VM = a noisy neighbour, OR you've burned your BURST
               CREDITS (AWS t2/t3/t4g CPU credits). YOU CANNOT FIX THIS IN CODE.
               st climbing to 20–30% on a t3.micro is the classic "my server got slow
               after a week" — throttled to baseline. Fix: bigger/unlimited instance.

LINE 4  Mem: free being tiny is NOT a problem — Linux uses spare RAM as page cache
        (buff/cache) and returns it instantly on demand.
LINE 5  avail Mem 471.1 ◀ THE REAL "how much can I still allocate" number.
        Swap 0.0 = typical on cloud VMs and ALWAYS in containers → memory pressure
        goes STRAIGHT to the OOM killer with no warning shot.
```

**Process-table columns:** `PR` kernel priority (20 normal, `rt` realtime) · `NI` nice **−20 greediest .. +19 most generous** (`PR = 20 + NI`) · `VIRT` = VSZ (ignore) · **`RES` = RSS, the real RAM you watch** · `SHR` shareable part of RES (`RES − SHR` ≈ private memory) · `S` state · `%CPU` **since the last refresh** — unlike `ps`, this is "right now", can exceed 100% · `%MEM` · `TIME+` cumulative CPU.

**Interactive keys:** `M` sort by **M**emory (press first in an OOM incident) · `P` by CPU · `T` by cumulative time · `1` expand `%Cpu(s)` to **one row per core** (reveals a single pegged core — a single-threaded bottleneck, which Node is) · `H` toggle **thread** view · `c` toggle full command line · `u` filter by user · `k` kill (prompts PID then signal, default 15) · `r` renice · `z`/`x` colour/highlight · `W` save config to `~/.toprc` · `q` quit.

```bash
top -b -n 1 -o %MEM | head -15     # non-interactive, for scripts/cron/pipes
#    │  │    └── -o : sort field
#    │  └── -n 1 : one iteration then exit
#    └── -b : batch mode (no ANSI codes — safe to pipe/redirect)
```

### `htop` — use it when installed

`htop` is `top` with a UI you don't fight: colour per-core meters, and: `F5`/`t` **tree view** (top can't) · `F3` incremental search · `F4` **filter** (hides non-matches) · `F6` sort column · `F9` **kill with a signal MENU** (pick SIGTERM without memorizing 15) · `F7`/`F8` renice · `H` show/hide threads · `Space` tag multiple, then F9 kills all tagged. Install: `apt install htop` / `apk add htop`. Worth the 200 KB on every server.

---

## Exact Syntax Breakdown

```
kill -TERM 9142           All of these are IDENTICAL:
│    │     │                kill 9142         ← DEFAULT IS SIGTERM (15). Not 9.
│    │     └ target PID     kill -15 9142
│    └ the signal          kill -TERM 9142  /  kill -s TERM 9142
└ a bash BUILTIN (also /bin/kill)

Forms you must know:
  kill -l              list every signal name/number
  kill -l 137          → KILL       (decodes an exit code: 137 − 128 = 9)
  kill -0 9142         ◀ SEND NOTHING. Just the permission+existence check.
                         exit 0 = exists & you may signal it;  exit 1 = gone.
                         THE PID-FILE HEALTH CHECK:
                            kill -0 "$(cat /run/app.pid)" || restart_app
  kill -- -901         ◀ NEGATIVE PID = signal the whole PROCESS GROUP 901
                         (kill a script AND every child, instead of orphaning them)
  kill -9 9142         SIGKILL. Uncatchable. Last resort.

pkill -f "node.*worker"
│     │  └ an EXTENDED REGEX matched against...
│     └ -f : ...the FULL COMMAND LINE, not just the 15-char process NAME.
│          Without -f you can't tell server.js from worker.js. With -f you can.
└ pkill = pgrep + kill (same matching rules)
  ⚠ FOOTGUN: `ps aux | grep node` matches its OWN grep line — use the bracket trick:
             ps aux | grep '[n]ode'      ("[n]ode" matches "node", not "[n]ode")
     (pgrep/pkill exclude themselves, so they don't have this problem)
  ⚠ ALWAYS DRY-RUN:  pgrep -af "node.*worker"   ← LOOK first
                     pkill  -f "node.*worker"   ← only then kill

killall node
│       └ matched by EXACT process NAME (not a regex, not the cmdline)
└ kills EVERY process with that name.
  ⚠ On a shared box this kills your API, your worker, pm2's children, and other
    teams' services. All of them. Instantly.
  ⚠ On Solaris `killall` kills ALL processes on the system — dangerous muscle memory.
  Safer:  killall -i node   (ask each)    killall -u deploy node   (scope by user)

renice -n 10 -p 9142     change an ALREADY-RUNNING process's priority
nice   -n 10 ./backup.sh start a NEW process with that nice value
  ⚠ ONLY ROOT CAN LOWER a nice value (be greedier). A normal user can only raise it
    (be nicer) — `nice -n -5` as non-root → "Permission denied".
ionice -c 3 -p 9142      I/O priority; class 3 = "idle" (only when disk is free).
                         Perfect for backups / `find /` / log compaction on a live box.
```

---

## Example 1 — Basic

```bash
sleep 300 &
# [1] 7201        ← [1] = JOB NUMBER (a bash concept, only meaningful in THIS shell)
jobs -l
# [1]+  7201 Running   sleep 300 &

fg %1             # foreground it (now owns the terminal; Ctrl-C would hit it)
# ^Z              ...press Ctrl-Z: sends SIGTSTP(20)
# [1]+  Stopped   sleep 300
ps -o pid,stat,cmd -p 7201
#  7201 T    sleep 300     ← T = stopped. Frozen, not dead.

bg %1             # resume in the BACKGROUND (sends SIGCONT, detaches from terminal)
ps -o pid,stat,cmd -p 7201
#  7201 S    sleep 300     ← back to S. SIGCONT resumed it.

kill %1           # `kill` with no signal = SIGTERM. Use %1 (job spec) or the PID.
# [1]+  Terminated  sleep 300

kill -l 137       # decode a Docker exit code
# KILL            ← 137 − 128 = 9 = SIGKILL. The container was hard-killed.

sleep 100 & kill -0 $! && echo alive    # test existence WITHOUT signalling
# alive
```

---

## Example 2 — Production Scenario

**03:52 AM.** Deploy is red: *"Error: listen EADDRINUSE: address already in use :::3000."* The new container can't start. **Do not reflexively `kill -9`.** Triage.

```bash
# ── STEP 1: WHAT holds port 3000? ─────────────────────────────────────────
$ sudo ss -tulpn | grep :3000
tcp LISTEN 0 511 *:3000 *:* users:(("node",pid=9142,fd=23))
#   │                              │        │      └ it's fd 23 (topic 19)
#   │                              │        └ ◀ THE PID YOU NEED
#   └ LISTEN = a server socket, actively bound (topic 28)
#   ss flags: -t TCP  -u UDP  -l LISTENing  -p show PID  -n numeric (don't resolve)
#   No ss?  sudo lsof -i :3000   or   sudo netstat -tulpn | grep :3000

# ── STEP 2: WHAT IS THAT PROCESS? Never kill a PID you haven't identified. ──
$ ps -p 9142 -o pid,ppid,user,stat,%cpu,rss,etime,cmd
    PID   PPID USER   STAT %CPU    RSS   ELAPSED CMD
   9142      1 deploy Ssl   0.1 184320  02:14:07 node /srv/app/server.js
#           ^^^                          ^^^^^^^^ 2h14m old — the PREVIOUS release.
#           PPID 1 — ORPHANED. Its supervisor died and left it running.
$ ls -l /proc/9142/cwd
... /proc/9142/cwd -> /srv/releases/20240710-1442 (deleted)
#                                                  ^^^^^^^^^ its cwd was DELETED.
#   100% the stale old release (topic 04: dirent gone, inode alive while held open).

# ── STEP 3: Any live traffic we'd be cutting off? ─────────────────────────
$ sudo ss -tnp state established '( sport = :3000 )' | wc -l
2       ← two live connections. Give it a few seconds to drain.

# ── STEP 4: SIGTERM. THE POLITE ONE. ──────────────────────────────────────
$ kill 9142          # = kill -TERM 9142.  NOT -9.
$ sleep 3; ps -p 9142
# (blank — gone)  ✅  It handled SIGTERM, drained, exited cleanly.
$ sudo ss -tulpn | grep :3000
# (nothing) — port free.

# ── STEP 5: escalate ONLY if it had refused to die ────────────────────────
#   $ kill -9 9142        ← after a 10s grace, and only then. Exit 137, state maybe dirty.
#   If it STILL won't die, check the state:
#   $ ps -o pid,stat,wchan:25 -p 9142
#      9142 D nfs_wait_bit_killable   ← D STATE. SIGKILL can't be delivered. Storage
#                                        is the problem, not the process (topic 16).

# ── STEP 6: fix the ROOT CAUSE — why was it orphaned? ─────────────────────
$ sudo grep -E 'Type|ExecStart' /etc/systemd/system/api.service
Type=simple
ExecStart=/bin/sh -c 'node /srv/app/server.js'    ← 🔴 A SHELL WRAPPER. Same bug as
#   Docker's shell-form CMD (topic 16): systemd tracks the `sh`, kills the `sh`, and
#   `node` is orphaned and keeps the port.
# FIX:  ExecStart=/usr/bin/node /srv/app/server.js
#       KillMode=control-group   (default; kills the whole cgroup)
#       TimeoutStopSec=30
#       Restart=on-failure
$ sudo systemctl daemon-reload && sudo systemctl restart api
```

**What you did:** identified before you killed, escalated politely, verified, then fixed the actual bug — a shell wrapper that broke the supervisor's ability to signal the real process. A reboot would have "fixed" it for six hours.

---

## Common Mistakes

### Mistake 1 — Reaching for `kill -9` first

**Wrong:** `sudo kill -9 $(pgrep node)` — the reflex.

**Root cause:** SIGKILL is handled *by the kernel* and never reaches your code. So `server.close()` never runs (in-flight requests are severed, clients see resets); the DB pool is never closed (Postgres holds those backends until its own keepalive timeout); buffered writes are lost; and **the PID/lock file is not removed**, so the next start says "already running" and refuses to boot.

**Right:**
```bash
kill 9142
for i in $(seq 10); do kill -0 9142 2>/dev/null || break; sleep 1; done
kill -0 9142 2>/dev/null && kill -9 9142      # escalate ONLY if still alive
```
**Prevention:** make SIGTERM actually work in the app, so you never *need* -9.

### Mistake 2 — `killall node` on a shared server

**Wrong:** `killall node` — kills your API, your worker, pm2's children, and other teams' services at once.

**Root cause:** `killall` matches the **process name**, and every Node process on Earth is called `node`. No scoping.

**Right — be specific, dry-run first, or use the supervisor:**
```bash
pgrep -af "node.*server.js"          # LOOK before you leap
pkill -f "node.*server.js"           # or pkill -u deploy -f server.js
sudo systemctl stop api              # BEST — kills the whole cgroup cleanly
```
**Prevention:** every production process should be owned by a supervisor (22/33). If you're `pkill`-ing production, you have a supervision problem.

### Mistake 3 — Panicking at VSZ, or at `free` being low

**Wrong:** "VIRT is 11.2g and free is 182 MB — we're about to OOM!"

**Right:** two misreadings. **VIRT/VSZ is address space, not RAM** — `RES` (1.7 GB) is the real figure. And **low `free` is by design** — Linux uses spare RAM as page cache and returns it instantly; the number that matters is **`avail Mem`** / the `available` column of `free -h`.
```bash
$ free -h
              total  used  free  shared  buff/cache  available
Mem:          7.8Gi 6.9Gi 182Mi   12Mi       689Mi       471Mi
#                          ^IGNORE                        ^WATCH THIS
```
**Prevention:** `--sort=-rss` never `-vsz`; alert on `available` never `free`.

### Mistake 4 — Using `nohup` / `&` / `disown` for a production service

**Wrong:** `nohup node server.js & ; disown -h ; exit`. Works until it crashes at 4am with nothing to restart it, or the box reboots, or you need the logs (in a `nohup.out` somewhere), or you need to stop it cleanly.

**Root cause:** these are **shell job-control tools** whose entire job is *"survive SIGHUP when this terminal dies."* `&` backgrounds it; `disown -h` removes it from bash's job table; `nohup` ignores SIGHUP and redirects output to `nohup.out`; `setsid` starts it in a brand-new session with no controlling terminal. None give you restart-on-crash, start-on-boot, log rotation, resource limits, a clean stop, or a status command.

**Right:**
```bash
tmux new -s migrate                     # interactive long work you want to watch (15)
sudo systemctl enable --now api         # an actual SERVICE (22) — or pm2 (33)
```
**Prevention:** if it must be running tomorrow, it belongs to systemd. `nohup` is for one-off scripts.

### Mistake 5 — Not knowing the OOM killer took your app

**Wrong:** "Node just disappeared — no logs, no stack trace. Must be a Node bug."

**Root cause:** Node didn't crash — **the kernel executed it.** Out of memory with nothing reclaimable, the OOM killer SIGKILLs a victim. SIGKILL runs no code, so your app logged nothing. It scores every process ~proportional to **RSS**, adds `oom_score_adj`, and kills the **highest** — which on an app server is essentially always your Node process, because it's the **biggest**, not the buggiest.

**Diagnose:**
```bash
$ sudo dmesg -T | grep -iE 'out of memory|killed process'
[Fri Jul 12 03:11:04 2025] Out of memory: Killed process 9142 (node)
    total-vm:11284736kB, anon-rss:1841236kB, ... oom_score_adj:0
#                                     ^^^^^^^^ 1.8 GB resident — the fattest process died.
$ docker inspect api --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
137 true       # 128+9 = SIGKILL, and Docker confirms OOM.
```
**Influence & prevent:**
```bash
echo -500 | sudo tee /proc/9142/oom_score_adj   # protect the API (−1000 = never kill)
echo  800 | sudo tee /proc/$(pgrep -f batch-worker)/oom_score_adj   # sacrifice batch first
# systemd:  OOMScoreAdjust=-500
# Best prevention: cap V8 so NODE throws a catchable JS OOM (log/alert) BEFORE the
# kernel SIGKILLs it silently. Rule of thumb: ~75% of the container memory limit,
# leaving room for the C++ heap, buffers, and stacks (which live OUTSIDE the V8 heap):
node --max-old-space-size=1024 server.js        # ~1 GB heap in a 2 GB container
```

---

## Hands-On Proof

```bash
# PROVE IT: SIGTERM is catchable, SIGKILL is not
cat > /tmp/sig.js <<'EOF'
process.on('SIGTERM', () => { console.log('caught SIGTERM'); process.exit(0); });
setInterval(() => {}, 1000); console.log('pid', process.pid);
EOF
node /tmp/sig.js & sleep 1; kill -TERM $!    # caught SIGTERM   ← handler RAN
node /tmp/sig.js & sleep 1; kill -9 $!       # (nothing) ← SIGKILL ran no code, ever

# PROVE IT: exit code 128+N names the signal
bash -c 'sleep 60 & wait $!' & P=$!; sleep 1; kill -9 $P; wait $P; echo "exit=$?"
# exit=137          (128+9)      kill -l 137 → KILL

# PROVE IT: kill -0 sends nothing, only tests
sleep 100 & P=$!; kill -0 $P && echo "exists"; kill $P; sleep 1
kill -0 $P 2>/dev/null || echo "gone"

# PROVE IT: a process's signal disposition is visible in /proc
node /tmp/sig.js & sleep 1; grep -E 'SigIgn|SigCgt' /proc/$!/status; kill $!
# SigIgn: 0000000000001000   ← bit 13 = SIGPIPE ignored (Node does this)
# SigCgt: 0000000180014202   ← the CAUGHT mask (bits for handled signals)

# PROVE IT: %CPU in `ps` is a LIFETIME AVERAGE; in `top` it's INSTANTANEOUS
yes > /dev/null & Y=$!; sleep 5
ps -p $Y -o %cpu=              # ~99  (busy its whole short life)
kill -STOP $Y; sleep 20        # freeze it (state T, burns nothing)
ps -p $Y -o stat=,%cpu=        # T ~24   ← ps's average has DECAYED
top -b -n1 -p $Y | tail -1     #   0.0   ← top tells the truth: using nothing now
kill -CONT $Y; kill $Y

# PROVE IT: only root can be greedy (lower the nice value)
nice -n 5 sleep 1 &            # works: raising nice = being nicer = allowed
nice -n -5 sleep 1            # nice: cannot set niceness: Permission denied

# PROVE IT: SIGHUP is what kills your process when SSH closes
#   Terminal A:  sleep 500 & echo $!    then CLOSE terminal A (don't type exit)
#   Terminal B:  ps -p <PID>            → gone. bash SIGHUP'd it.
#   Repeat with: nohup sleep 500 &  (or  setsid sleep 500)  → SURVIVES.

# PROVE IT: `top` and `ps` are just reading /proc
sudo strace -c -e trace=openat top -b -n1 2>&1 | grep openat
#   ← thousands of openat() calls, every one into /proc/<pid>/. That's all it is.
```

---

## Practice Exercises

### Exercise 1 — Easy
```bash
sleep 400 &
jobs -l
ps -o pid,stat,tty,cmd -p $!      # note STAT and TTY
fg %1                             # then press Ctrl-Z
ps -o pid,stat,cmd -p <PID>       # what state now, and why?
bg %1
ps -o pid,stat,cmd -p <PID>       # and now?
kill %1                           # what signal did that send?
```
**Answer from output:** What STAT did Ctrl-Z produce and which signal did the terminal send? What did `bg` send to resume it? What signal number did `kill %1` use, and how do you *prove* it wasn't 9?

### Exercise 2 — Medium
```bash
cat > /tmp/drain.js <<'EOF'
console.log('pid', process.pid); setInterval(() => {}, 1000);
process.on('SIGTERM', () => { console.log('draining 3s...');
  setTimeout(() => { console.log('drained, exit 0'); process.exit(0); }, 3000); });
EOF
node /tmp/drain.js & P=$!; sleep 1; kill $P;      wait $P; echo "A: $?"   # graceful
node /tmp/drain.js & P=$!; sleep 1; kill -9 $P;   wait $P; echo "B: $?"   # hard
node /tmp/drain.js & P=$!; sleep 1; kill $P; sleep 1; kill -9 $P 2>/dev/null; wait $P; echo "C: $?"  # too little grace
```
**Answer:** Give all three exit codes and decode each with `kill -l <N-128>`. In case C, which log lines printed and which didn't, and what does that reveal about the relationship between your drain time and `docker stop -t` / `terminationGracePeriodSeconds`?

### Exercise 3 — Hard (Production Simulation)
```bash
# SETUP — create the mess, then FORGET the PID:
( setsid node -e "require('http').createServer((_,r)=>r.end('old')).listen(3000)" \
    >/dev/null 2>&1 & )
# You now have an orphaned, detached process holding port 3000. You don't know its PID.
```
**Your job, in order, using only these tools:** (1) prove port 3000 is occupied and find the PID (`ss`/`lsof`/`netstat`); (2) identify it BEFORE touching it — user, binary, cwd, uptime, PPID, RSS (`ps -o …`, `ls -l /proc/PID/{exe,cwd}`); (3) check for ESTABLISHED connections; (4) SIGTERM, poll with `kill -0` for up to 5s; (5) escalate to SIGKILL only if needed, and record which you needed; (6) verify the port is free; (7) wrap it all in **one reusable `port_kill()`** that takes a port, prints the offending process's details, SIGTERMs, waits 5s, SIGKILLs if needed, confirms the port is free — and **refuses to act if the port is free** and **never kills a PID it hasn't first printed**. Deliverable: the function plus your transcript, stating which signal was actually needed and why.

---

## Mental Model Checkpoint

1. **What are the only two uncatchable signals, and why does the kernel enforce that?** Name three things that *don't happen* when a process is SIGKILLed.
2. **`docker stop` on your Node container takes the full 10s and exits 137.** Give three independent root causes and the fix for each.
3. **VSZ vs RSS:** why is 11 GB VSZ *not* a bug, and why is summing RSS across processes still wrong?
4. **In `%Cpu(s)`, what are `wa` and `st`?** For each: what does a high value mean, and can you fix it in code?
5. **You have a PID — write the correct kill escalation from memory,** including how to test whether it's dead without signalling it.
6. **How does the OOM killer choose a victim, and how do you (a) prove it happened and (b) make it pick a different process?**
7. **Why is `nohup node server.js &` the wrong way to run a production service?** Name four things a supervisor gives you that it doesn't.

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `ps aux` | BSD snapshot: USER PID %CPU %MEM VSZ RSS TTY STAT START TIME CMD | `a` all, `u` user fmt, `x` no-tty |
| `ps -ef` | UNIX snapshot (has PPID) | `-e` every, `-f` full, `--forest` tree |
| `ps -eo pid,rss,etime,cmd --sort=-rss` | Custom columns, sorted by memory | `-o`, `--sort=-FIELD` |
| `ps -eLf` | Show **threads** (`LWP` = TID) | `-L` |
| `pgrep -af PATTERN` | Find PIDs; dry-run before pkill | `-f` full cmdline, `-a` show, `-n` newest, `-u` user |
| `pkill -f PATTERN` | Signal by pattern | `-f`, `-u`, `-9` (last resort) |
| `killall NAME` | Signal by **exact name** — ⚠ kills ALL | `-i` interactive, `-u USER` |
| `kill PID` | **Send SIGTERM** (the default) | `-9`/`-KILL`, `-HUP`, `-USR1`, `-s NAME` |
| `kill -l [N]` | List signals / decode exit code (`kill -l 137`→KILL) | — |
| `kill -0 PID` | **Test existence, send nothing** (PID-file health checks) | — |
| `kill -- -PGID` | Signal a whole **process group** | note the `--` and minus |
| `top` | Live view | `-b` batch, `-n1` once, `-o FIELD`, `-p PID` |
| top keys | `M`mem `P`cpu `T`time `1`per-core `H`threads `c`cmd `u`user `k`kill `r`renice `W`save `q`quit | |
| `htop` | Better top | `F5`tree `F3`search `F4`filter `F9`kill-menu `H`threads |
| `nice -n 10 CMD` / `renice -n 10 -p PID` | Start with / change priority | −20 greediest (root only) .. +19 |
| `ionice -c 3 -p PID` | Disk-I/O priority; class 3 = idle | backups on a live box |
| `jobs` / `fg %1` / `bg %1` | Shell job control (this shell only) | `jobs -l` shows PIDs |
| `nohup CMD &` / `setsid CMD` | Ignore SIGHUP / new session — NOT a service manager | output → `nohup.out` |
| `dmesg -T \| grep -i oom` | Find OOM kills | `-T` human timestamps |
| `/proc/PID/oom_score_adj` | −1000 (never) .. +1000 (kill first) | `OOMScoreAdjust=` in systemd |

---

## When Would I Use This at Work?

### Scenario 1: Every deploy drops requests
Your ALB shows a spike of 502s at each rollout. `docker inspect` shows exit code **137** — SIGKILL — so the container hit the grace period without exiting. Three checks in order: is the Dockerfile `CMD` in **exec form** (topic 16)? Is `process.on('SIGTERM')` registered? Is the drain backstop shorter than `terminationGracePeriodSeconds`? You fix exec form, add the handler with a 10s backstop under a 30s grace, and the exit code becomes **0**. The 502s stop — a real, user-facing win from one signal.

### Scenario 2: "The API got slow and nothing changed"
`top` shows `%Cpu(s): 4.1 us, 1.2 sy, 66.0 id, 0.4 wa, 28.3 st`. **`st` = 28.3** — your code is idle; the *hypervisor* is stealing 28% of your CPU. It's a `t3.small` that burned through its CPU credits and is now throttled to baseline. Profiling JavaScript won't help — move to an unburstable type or enable T-unlimited. Knowing what `st` means saved you a day.

### Scenario 3: Reload nginx without dropping a connection
You changed `/etc/nginx/nginx.conf`. A restart drops every in-flight connection; instead you use SIGHUP's daemon convention: `sudo nginx -t && sudo nginx -s reload` sends **SIGHUP** to the master, which re-reads the config, spawns new workers, and lets old workers finish their existing requests first. Zero dropped connections — the same mechanism `logrotate`'s `postrotate` uses to make a daemon reopen its log file (topic 23).

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 18 — System Resource Monitoring | `free`, `vmstat`, `iostat`, load average — the system-wide view behind the `wa`/`st`/`avail Mem` numbers you just learned to read |
| **Builds on** | 16 — Processes in Depth | task_struct holds the signal table; PID/PPID/STAT/zombies/D-state/PID 1 are prerequisites |
| **Builds on** | 12 — Pipes and Redirection | SIGPIPE is what happens when the reader of your pipe goes away |
| **Builds on** | 13 — Shell Scripting | `$?` = exit code; 128+N means signal N. `trap 'cleanup' TERM INT` is the shell's `process.on('SIGTERM')`. |
| **Builds on** | 15 — Shell Productivity | tmux is the right answer for long-running *interactive* work; `nohup` is not |
| **Used by** | 19 — File Descriptors | `ss -tulpn` gave you a PID *and* an fd; `lsof -p PID` is the next step |
| **Used by** | 22 — systemd and Services | `systemctl stop` = SIGTERM → `TimeoutStopSec` → SIGKILL. `KillMode`, `Restart=`, `OOMScoreAdjust=` are this doc, declared in a unit file. |
| **Used by** | 28 — Ports and Sockets | The `ss -tulpn` → PID → `ps -p` → `kill` chain is the canonical "port in use" triage |
| **Used by** | 33 — Node in Production | pm2, graceful shutdown, `--max-old-space-size`, zero-downtime reloads: all direct applications |
| **Used by** | 34 — Performance Investigation | High `wa` → `iostat`; high `st` → your cloud provider; OOM forensics → `dmesg` |
