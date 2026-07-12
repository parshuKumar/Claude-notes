# 34 — Performance Investigation

## ELI5 — The Simple Analogy

Imagine a **hospital emergency room**. A patient is wheeled in. The junior doctor panics and starts ordering every test in the building: MRI, blood panel, X-ray, ultrasound. Two hours later they have a mountain of data and no diagnosis.

The senior doctor does something different. She does **triage** — a fixed, boring, 60-second checklist that she runs on *every* patient, in the same order, every time:

- **Airway** — can they breathe?
- **Breathing** — are they breathing?
- **Circulation** — is there a pulse?
- **Disability** — are they conscious?
- **Exposure** — is there an obvious wound nobody looked at?

She does this *before* she thinks. Because the checklist finds the bleeding artery in ten seconds, and the bleeding artery is the answer 80% of the time. Only when the checklist comes back clean does she start ordering the expensive tests.

**A slow server is a patient in the ER.** Tool-flailing — running `top`, then `htop`, then `strace`, then googling "node slow" — is the junior doctor. This doc teaches you the triage checklist and the diagnostic method. The tools are the MRI machine: powerful, expensive, and useless until you know what you're looking for.

There are only **four organs** in this patient: **CPU, Memory, Disk, Network.** One of them is bleeding. Find it.

---

## Where This Lives in the Linux Stack

Performance investigation is unusual: it does not live at one layer. It is the skill of **reading every layer at once** and knowing which one is lying to you.

```
Hardware (CPU cores, RAM chips, disk platters/NAND, NIC)
  │    ◀◀◀ perf hardware counters read HERE (cache misses, IPC)
  │
  └── KERNEL
       │   ◀◀◀ /proc, /sys, and kernel counters live HERE.
       │       vmstat, iostat, mpstat, free, sar all just READ /proc.
       │       eBPF/bpftrace runs INSIDE the kernel.
       │
       └── System Calls  ◀◀◀ strace intercepts HERE (via ptrace)
            │                 "what is this process ASKING the kernel to do?"
            │
            └── C Library (glibc)  ◀◀◀ ltrace intercepts HERE
                 │
                 └── Node.js / V8 runtime
                      │   ◀◀◀ --cpu-prof, heap snapshots, event-loop lag HERE
                      │       The kernel CANNOT see inside V8. It sees one
                      │       busy thread. Only V8 knows it's a regex.
                      │
                      └── Your JavaScript  ◀◀◀ the actual bug is usually HERE
```

**The single most important structural fact:** the kernel's tools tell you *which resource* is saturated. They cannot tell you *which line of your code* did it. You must hand off from kernel tools to runtime tools at exactly the right moment. Most failed investigations are people trying to `strace` their way to a JavaScript bug, or trying to read a flamegraph to find a full disk.

---

## What Is This?

Performance investigation is a **method** for going from a vague complaint ("the app is slow") to a specific, proven root cause ("a regex in `validateEmail()` backtracks exponentially on 40-character inputs and pegs one core, blocking the event loop for every request").

It is not a pile of commands. It is: characterize the symptom → find the saturated resource → drill into the offending process → prove the fix. The commands only exist to answer questions the method asks.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| The USE method (Utilization/Saturation/Errors) | You will run 20 tools, gather no evidence, and "fix" something at random |
| That load average ≠ CPU usage (topic 18) | You'll see "load 8" on a 16-core box and panic over nothing |
| That per-core CPU matters for Node | You'll see "12% CPU" on a 8-core box and conclude "CPU is fine" while core 3 is pegged at 100% and your single-threaded Node process is dying |
| `dmesg` | You'll spend 4 hours debugging an app that the kernel OOM-killed 4 hours ago |
| `st` (steal) in vmstat | You'll spend a week optimizing code on a box your hypervisor is throttling |
| The kernel OOM kill vs V8 heap OOM | You'll raise `--max-old-space-size` on a box that's already out of RAM and make it worse |
| How to read a flamegraph | You'll stare at a beautiful picture and learn nothing |
| That strace is 10–100x slowdown | You'll attach it to your prod API and cause the outage you're investigating |
| Event loop blocking | Your process will be "up", healthchecks green, and every single request will time out |

---

## The Physical Reality

**There are only four hardware resources.** Everything you can call "slow" is one of them saturated, or one of a short list of software-side limits. Memorize this table — it *is* the map of the territory.

```
┌──────────┬─────────────────────┬──────────────────────┬─────────────────────┐
│ RESOURCE │ UTILIZATION         │ SATURATION           │ ERRORS              │
│          │ (% busy)            │ (queued work)        │                     │
├──────────┼─────────────────────┼──────────────────────┼─────────────────────┤
│ CPU      │ mpstat -P ALL 1     │ vmstat 'r' column    │ dmesg (MCE, thermal)│
│          │ %usr + %sys         │ load avg > nproc     │                     │
│          │ PER CORE, NOT AVG   │ vmstat 'st' = steal  │                     │
├──────────┼─────────────────────┼──────────────────────┼─────────────────────┤
│ MEMORY   │ free -m: AVAILABLE  │ vmstat si/so (swap)  │ dmesg: OOM killer   │
│          │ NOT "free"          │ page scan rate       │ exit code 137       │
├──────────┼─────────────────────┼──────────────────────┼─────────────────────┤
│ DISK I/O │ iostat -xz 1: %util │ iostat: aqu-sz,await │ dmesg: I/O error,   │
│          │                     │ vmstat 'wa' + 'b'    │ remounted read-only │
├──────────┼─────────────────────┼──────────────────────┼─────────────────────┤
│ NETWORK  │ sar -n DEV 1        │ ss -tin: retrans     │ ip -s link: drops   │
│          │ rxkB/s vs NIC speed │ Recv-Q / Send-Q      │ sar -n EDEV         │
└──────────┴─────────────────────┴──────────────────────┴─────────────────────┘

PLUS the SOFTWARE-side resources (invisible to the four above):
┌──────────────────────┬────────────────────────────────────────────────┐
│ File descriptors     │ ls /proc/PID/fd | wc -l  vs  ulimit -n → EMFILE│
│ DB connection pool   │ app metrics / DB: pg_stat_activity             │
│ THE NODE EVENT LOOP  │ event-loop lag; Recv-Q on the LISTEN socket    │
└──────────────────────┴────────────────────────────────────────────────┘
```

**Utilization vs Saturation is the distinction everyone misses.** A disk at 100% utilization with 1 queued request is *fine* — it's working. A disk at 100% utilization with 60 queued requests and 400ms await is a **bottleneck**. Utilization says "busy". Saturation says "there is a queue, and the queue is where your latency is being manufactured."

---

## How It Works — Step by Step

### THE METHOD (do this in order, every single time)

```
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 1 — CHARACTERIZE THE SYMPTOM                                       │
│                                                                        │
│ "Slow" is NOT a symptom. Get precise:                                   │
│                                                                        │
│   □ LATENCY or THROUGHPUT?   (p99 up? or req/s down? These differ!)    │
│   □ ALL requests or a SUBSET? (one endpoint? one customer? one region?)│
│   □ CONSTANT or SPIKY?        (spiky → correlate with a clock)          │
│   □ SUDDEN or GRADUAL?                                                  │
│        SUDDEN  → a deploy, a config change, a traffic spike,           │
│                  a cron job, a full disk, a cert expiry                │
│        GRADUAL → a leak (memory, fd, connections), data growth,        │
│                  a table that outgrew its index                        │
│   □ WHEN did it start? (to the minute)                                 │
│                                                                        │
│ ►►► THE HIGHEST-VALUE QUESTION IN ALL OF OPERATIONS: "WHAT CHANGED?"   │
│     A deploy. A feature flag. A dependency bump. A DNS change.         │
│     A schema migration. A new customer onboarded. A cron job added.    │
│     Most incidents are SOLVED here, before you touch a single tool.    │
│     If the graph fell off a cliff at 14:32, ask what happened at 14:32.│
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 2 — FIND THE SATURATED RESOURCE                                    │
│                                                                        │
│ Run the 60-SECOND TRIAGE. Walk the USE table. There are only four      │
│ hardware resources plus three software ones. Eliminate them.           │
│                                                                        │
│ ►►► If NONE are saturated and the app is still slow, the app is        │
│     WAITING on something off-box (DB, upstream API, DNS, a lock)       │
│     — or its event loop is blocked. This is the MOST COMMON case.      │
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 3 — DRILL INTO THE OFFENDING PROCESS                               │
│                                                                        │
│ Kernel-level: pidstat, strace -c, /proc/PID/*, perf                    │
│ Runtime-level: --cpu-prof, heap snapshot, flamegraph                   │
│ Pick the RIGHT layer. A kernel tool cannot see a slow regex.           │
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│ STEP 4 — PROVE THE FIX                                                  │
│                                                                        │
│ Re-measure the SAME metric that was bad. Under the SAME load.          │
│ "It feels faster" is not evidence. p99 went 8000ms → 45ms is evidence. │
│ If you can't reproduce the bad state, you cannot prove you fixed it —  │
│ you have merely stopped observing it.                                  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## The 60-Second Triage (Brendan Gregg's Checklist)

Run these ten commands, in this order, on every slow box. It takes one minute and it ends most investigations. Do not skip lines because they "look fine" — you are building a picture.

```bash
uptime
dmesg | tail
vmstat 1 5
mpstat -P ALL 1 5
pidstat 1 5
iostat -xz 1 5
free -m
sar -n DEV 1 3
sar -n TCP,ETCP 1 3
top
```

*(On Ubuntu: `sudo apt install sysstat` for mpstat/pidstat/iostat/sar. `sar` historical collection needs `ENABLED="true"` in `/etc/default/sysstat`.)*

### Line by line — what it TELLS you and what BAD looks like

**1. `uptime`** — load average and how long the box has been up.

```
 14:23:07 up 42 days,  3:15,  2 users,  load average: 12.50, 8.30, 4.15
                                                      └1min─┘ └5m─┘ └15m┘
```
- Compare against `nproc`. Load 12.5 on 4 cores = **3x oversubscribed**.
- The three numbers are a **trend**: 12.5 / 8.3 / 4.15 means it is **getting worse right now**. Reversed (4.15 / 8.3 / 12.5) means it is recovering — you may have missed the event.
- **Linux load includes uninterruptible sleep (D state) — disk wait counts as load** (topic 18). High load with idle CPU = an I/O problem, not a CPU problem.
- **`up 4 min` when you expected 42 days = it rebooted. That is your incident.**

**2. `dmesg | tail`** — the kernel ring buffer. **RUN THIS FIRST. It ends investigations in five seconds.**

```
[8845123.4] Out of memory: Killed process 8234 (node) total-vm:2891244kB, anon-rss:2013124kB
[8845123.4] oom_reaper: reaped process 8234 (node), now anon-rss:0kB
[8912004.1] EXT4-fs error (device sda1): ext4_journal_check_start:83: Detected aborted journal
[8912004.1] EXT4-fs (sda1): Remounting filesystem read-only
[8990112.7] nf_conntrack: table full, dropping packet
```
You are looking for exactly three families:
- **OOM kills** — the kernel murdered a process (topic 17: this is exit code 137)
- **I/O errors / "Remounting filesystem read-only"** — the disk is dying, and every write in your app is now failing
- **Network/conntrack drops, TCP resource exhaustion**

If any of these appear, **stop triaging and go investigate that**. You have your answer.

**3. `vmstat 1`** — the single densest command in Linux.

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 9  0 512000  62144  18432 402112  148  312     0  1240 4021 8934 82 11  2  4  1
```
| Column | Meaning | BAD looks like |
|---|---|---|
| `r` | **runnable** threads waiting for CPU (excl. those in I/O wait) | `r` > `nproc` consistently = CPU saturation. This is the best CPU-saturation signal there is. |
| `b` | blocked in uninterruptible sleep (usually disk) | `b` > 0 persistently = disk bottleneck |
| `si`/`so` | **swap in / swap out, KB/s** | **ANY sustained non-zero = you are out of RAM and the kernel is paging to disk. Latency dies here. This is a 4-alarm fire.** |
| `wa` | % CPU time idle-but-waiting-on-I/O | high `wa` + low `us` = disk-bound, not CPU-bound |
| `st` | **STEAL — % of time your vCPU wanted to run but the hypervisor gave the physical core to someone else** | **Non-trivial `st` (>5–10%) means you are being throttled by your cloud provider / a noisy neighbour. NOTHING you do on this box will help. Resize the instance, move it, or complain. This single column has saved engineers entire weeks.** |
| `us`/`sy` | user vs system (kernel) CPU time | high `sy` = syscall storm, context-switch storm, or kernel work — go look at `strace -c` |

*Ignore the first line of vmstat output — it is an average since boot, not "now".*

**4. `mpstat -P ALL 1`** — **THE MOST IMPORTANT COMMAND IN THIS DOC FOR A NODE DEVELOPER.**

```
14:23:09  CPU   %usr  %nice  %sys  %iowait  %irq  %soft  %steal  %idle
14:23:09  all   12.51  0.00   1.25    0.25   0.00   0.12    0.00   85.87
14:23:09    0    2.01  0.00   1.00    0.00   0.00   0.00    0.00   96.99
14:23:09    1    1.98  0.00   0.99    0.99   0.00   0.00    0.00   96.04
14:23:09    2    3.00  0.00   1.00    0.00   0.00   0.00    0.00   96.00
14:23:09    3   98.02  0.00   1.98    0.00   0.00   0.00    0.00    0.00   ◀◀◀
14:23:09    4    1.00  0.00   1.00    0.00   0.00   0.00    0.00   98.00
```

**The `all` row says 12.5% CPU. The `all` row is a LIE.**

Core 3 is pegged at 100%. Everything else is idle. On an 8-core box, one fully-saturated core = 12.5% average. And **what single-threaded process are you running? Your Node app.** Its one JavaScript thread is nailed to one core, spinning, and while it spins, **the event loop is blocked and every user is being served nothing.**

> **If you learn one thing from this doc: never, ever trust the box-wide CPU average when debugging Node. Always go per-core. `top` then press `1` does the same thing.**

**5. `pidstat 1`** — per-process CPU, rolling, one line per interval.

```
14:23:10   UID  PID   %usr %system  %guest   %CPU  CPU  Command
14:23:11  1001  8234  97.00    3.00    0.00 100.00    3  node
14:23:11     0   912   0.99    0.00    0.00   0.99    1  nginx
```
Better than `top` for *spotting a short-lived spike*, because it prints a scrolling history you can read afterwards instead of a screen that repaints. Note the `CPU` column — that's the core it's on. It says `3`. Same core mpstat flagged. That's your process.

**6. `iostat -xz 1`** — extended per-device disk stats. (`-x` extended, `-z` omit idle devices.)

```
Device   r/s   w/s   rkB/s   wkB/s  rrqm/s  wrqm/s  r_await  w_await  aqu-sz  %util
sda     2.00 412.00   32.00 18244.00   0.00   14.00    1.20   418.50   58.10  99.80
```
| Column | Meaning |
|---|---|
| `r_await`/`w_await` | **average ms per I/O, INCLUDING queue time. This is what your app actually feels.** SSD: <1–5ms normal. >20ms = suffering. 418ms = the disk is drowning. |
| `aqu-sz` | average queue depth — **this is SATURATION**. 58 requests deep = a queue. |
| `%util` | % of time the device had at least one request in flight. **On SSD/NVMe, 100% util does NOT mean "full" — they handle parallel requests. Use `await` + `aqu-sz`, not `%util`, to judge an SSD.** |

**7. `free -m`** — memory. **Read the `available` column. Only the `available` column.** (Topic 18.)

```
              total     used     free   shared  buff/cache   available
Mem:           7963     6821      118      148        1024         642
Swap:          2047     1802      245
```
- `free` = 118MB is **not a problem**. Linux uses spare RAM for page cache on purpose. Unused RAM is wasted RAM.
- `available` = 642MB **is** the number: how much a new process could get without swapping. **This box is nearly out.**
- **Swap `used` = 1802 of 2047MB. Combined with `si`/`so` in vmstat, this box is actively thrashing.**

**8. `sar -n DEV 1`** — NIC throughput.
```
14:23:11  IFACE  rxpck/s  txpck/s   rxkB/s   txkB/s  %ifutil
14:23:11   eth0  8912.00  9104.00 108224.0 112480.0   92.10
```
`%ifutil` near 100, or rxkB/s approaching your NIC's line rate (a 1 Gbit NIC ≈ 125,000 kB/s) = you are **network-saturated**. Also `sar -n EDEV 1` for errors/drops.

**9. `sar -n TCP,ETCP 1`** — TCP health: `retrans/s` (retransmits — packet loss on the path) and `atmptf/s` (failed connection attempts).

**10. `top`** (or `htop`) — the human's overview. Press **`1`** to split by core. Press **`M`** to sort by memory, **`P`** by CPU. Press **`H`** to show individual **threads** — which is how you see Node's 4 libuv threadpool threads plus V8's GC threads separately from the main thread.

---

## The Decision Tree

```
                          ┌──────────────────────┐
                          │  "The app is slow"    │
                          │  Run the 60s triage   │
                          └──────────┬───────────┘
                                     │
        ┌──────────────┬─────────────┼──────────────┬────────────────┐
        ▼              ▼             ▼              ▼                ▼
  ┌───────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────┐
  │ HIGH CPU  │  │ HIGH MEM │  │ SLOW, but │  │ DISK     │  │ vmstat st>0  │
  │           │  │ / OOM    │  │ CPU IS LOW│  │ FULL/SLOW│  │              │
  └─────┬─────┘  └────┬─────┘  └─────┬─────┘  └────┬─────┘  └──────┬───────┘
        │             │              │             │               │
        │             │              │             │        STOP. You are being
        │             │              │             │        throttled by the
        │             │              │             │        hypervisor. Nothing
        │             │              │             │        on-box will help.
        │             │              │             │        Resize / migrate.
        │             │              │             │
        ▼             ▼              ▼             ▼
   [BRANCH A]    [BRANCH B]     [BRANCH C]    [BRANCH D]
```

### BRANCH A — HIGH CPU

```
top → is it %us (user) or %sy (system) time?
 │
 ├─ HIGH SYSTEM (sy) TIME ──► the process is hammering SYSCALLS.
 │      → strace -c -p PID  (summary! never full trace on a hot process)
 │      → classic finds: 40,000 stat() calls/sec on a missing file;
 │        a busy epoll_wait loop; excessive small write()s (no buffering);
 │        context-switch storm (vmstat 'cs' in the hundreds of thousands)
 │
 └─ HIGH USER (us) TIME ──► the process is computing.
        │
        mpstat -P ALL 1  → is it ONE core at 100% (single-threaded)
        │                  or ALL cores (multi-threaded / many processes)?
        │
        pidstat 1  → which PID?
        │
        └─ IS IT YOUR NODE PROCESS?  ──► almost always yes
              │
              ├─ Genuine CPU-bound handler in JS?
              │    • sync crypto (bcrypt/pbkdf2 with high cost, scrypt)
              │    • JSON.parse / JSON.stringify on a multi-MB payload
              │    • a big sort/map over 100k objects per request
              │    • image/PDF processing in-process
              │    • ►► CATASTROPHIC REGEX BACKTRACKING (ReDoS) ◀◀
              │         A regex like /^(\w+\s?)*$/ on a 40-char input can
              │         take 2^40 steps. It pegs one core, blocks the event
              │         loop, and therefore breaks the request for EVERY
              │         user — not just the one who sent the bad input.
              │         A single attacker with one curl can take you down.
              │
              └─ Or an accidental busy loop / runaway setInterval / a
                 while-loop retry with no backoff.
              │
              └──► PROFILE IT. Do not guess.
                     node --cpu-prof app.js       → .cpuprofile → Chrome DevTools
                     node --prof + --prof-process  → text tick summary
                     perf top -p PID               → live, kernel+native
                     perf record -F 99 -p PID -g -- sleep 30 → flamegraph
                     0x / clinic.js flame          → the easy Node path
```

**Reading a flamegraph correctly** — most people get this wrong:

```
        ┌──────────────────────────────────────────────────────┐
        │                    node::Start                        │  ← bottom = entry
        ├──────────────────────────────────────────────────────┤
        │                  processTicksAndRejections            │
        ├────────┬─────────────────────────────────────────────┤
        │ parse  │              validateEmail                   │  ← WIDE = hot
        ├────────┼─────────────────────────────────────────────┤
        │  ...   │        RegExp::Exec  ██████████████████      │  ◀ the plateau
        └────────┴─────────────────────────────────────────────┘

  ►► WIDTH = total time spent in that frame (samples). NOT importance-by-depth.
  ►► The x-axis is ALPHABETICAL (merged samples). It is NOT TIME. Left-to-right
     does NOT mean earlier-to-later. Do not read it as a timeline.
  ►► Height = call-stack depth. A tall narrow spike is a deep call chain that
     costs nothing. IGNORE IT.
  ►► LOOK FOR: the widest PLATEAU — the widest frame that has nothing (or
     little) above it. That is where the CPU actually is. That is your bug.
```

### BRANCH B — HIGH MEMORY / OOM

```
free -m  → available near zero? swap being used? (vmstat si/so)
 │
ps -eo pid,rss,vsz,comm --sort=-rss | head
 │      RSS = Resident Set Size = actual physical RAM held. THIS is the number.
 │      VSZ = virtual size — includes mapped-but-untouched memory. Often huge
 │            and meaningless. Do not panic about VSZ.
 │
 ├─ Is RSS growing steadily over HOURS on a flat request rate?  → A LEAK.
 └─ Or is it just BIG from the start?                           → Sized wrong.
 │
dmesg | grep -i "killed process"
 │
 └──► ►►► THE CRITICAL DISTINCTION — TWO DIFFERENT OOMs, OPPOSITE FIXES ◀◀◀
```

| | **Kernel OOM kill** | **V8 heap OOM** |
|---|---|---|
| **What you see** | `dmesg`: `Out of memory: Killed process 8234 (node)`. Process exit code **137** (128+9 = SIGKILL). systemd logs `Main process exited, code=killed, signal=KILL`. No JS stack trace. Container `docker inspect` → `"OOMKilled": true` | `FATAL ERROR: Ineffective mark-compacts near heap limit — JavaScript heap out of memory` printed to stderr, with a V8 stack. Exit code **134** (SIGABRT). |
| **Cause** | The **box/cgroup** ran out of physical RAM. The kernel picked a victim by `oom_score` (roughly: biggest RSS wins) and SIGKILLed it. **It may not even be the guilty process — it kills the biggest one.** | The **V8 heap limit** (default ~2GB old-space on 64-bit, or auto-sized from cgroup memory in modern Node) was hit. The box may have plenty of free RAM. |
| **The fix** | **LOWER** memory use, or add RAM, or set a cgroup limit on the *guilty* process (systemd `MemoryMax=`) so it dies instead of your API. **Raising `--max-old-space-size` here makes it strictly worse — you're telling Node to grab more of the RAM you don't have.** | **RAISE** `--max-old-space-size=4096` (if you truly need the heap and the box has it) **or fix the leak**. |

**Finding a Node memory leak:**
```
process.memoryUsage()  → { rss, heapTotal, heapUsed, external, arrayBuffers }
   heapUsed climbing across GCs = a JS leak
   rss climbing but heapUsed flat = a NATIVE leak (Buffers, a native addon)

node --inspect  (or send SIGUSR1 to a running node process to open the inspector)
   → chrome://inspect → Memory → Heap snapshot

►►► TAKE TWO SNAPSHOTS AND DIFF THEM. ◀◀◀
    Snapshot 1 → run traffic for 10 minutes → force GC → Snapshot 2.
    In DevTools, select snapshot 2 and choose "Comparison" against snapshot 1.
    The objects with a positive DELTA that were NOT collected are your leak.
    A single snapshot tells you what's big. A DIFF tells you what's GROWING.
    Only the diff is actionable.

The usual suspects (this list catches ~90% of Node leaks):
  1. An unbounded cache — a plain object or Map used as a cache with no TTL
     and no eviction. `cache[userId] = data` forever.
  2. Event listeners added PER REQUEST and never removed.
     → the tell: MaxListenersExceededWarning: Possible EventEmitter memory leak
       detected. 11 'data' listeners added.
  3. A closure captured by a module-level array. `allRequests.push(req)` —
     and `req` holds the socket, which holds the buffer, which holds...
  4. A growing set of timers — setInterval created per connection, never cleared.
  5. Global arrays used as queues that are pushed to faster than they're drained.
```

### BRANCH C — SLOW / HIGH LATENCY BUT CPU IS LOW

**This is the most common real-world case, and the most misdiagnosed.** The box looks *healthy*. CPU 3%. Memory fine. Disk fine. And every request takes 8 seconds. That means your app is not *working* — it is **WAITING**.

```
Is it waiting on DISK?
   iostat -xz 1  → high await?      → yes: see Branch D
   iotop -oPa                       → which process is doing the I/O?
   pidstat -d 1                     → per-process kB_rd/s, kB_wr/s

Is it waiting on the NETWORK / a DEPENDENCY?  (topic 30 callback)
   curl -w "@curl-format.txt" -o /dev/null -s https://upstream-api/health
   → break the request into phases and see WHICH phase is slow:
        dns:     0.004s   ← slow here? DNS problem (topic 27)
        connect: 0.031s   ← slow here? network/TCP/firewall
        tls:     0.089s   ← slow here? TLS handshake / cert chain
        ttfb:    3.842s   ◀◀ SLOW HERE = the UPSTREAM is slow, not you.
        total:   3.851s
   → Your app is fine. Go tell the upstream team. Add a timeout and a
     circuit breaker so their outage stops being your outage.

Is it waiting on the DATABASE?  ►►► THE #1 CAUSE ◀◀◀
   • A missing index → a query that was 2ms at 10k rows is 4s at 10M rows.
     This is the classic GRADUAL degradation. Nothing changed in your code.
     The DATA changed.  → EXPLAIN ANALYZE the query.
   • An N+1 query → one request fires 500 queries. Each is fast. The request
     is not.
   • ►► CONNECTION POOL EXHAUSTION ◀◀
     Your pool has 10 connections. 200 concurrent requests want one.
     190 requests sit in a JS-land queue, waiting. They time out.
     CPU: 2%. Memory: fine. Disk: fine. The box looks PERFECT.
     THIS IS WHAT "the app is slow but the box is idle" USUALLY IS.
     → Check: pool.waitingCount / pool.idleCount in pg-pool.
     → Check on the DB: SELECT state, count(*) FROM pg_stat_activity GROUP BY 1;
       (lots of 'idle in transaction' = you forgot to COMMIT/release somewhere —
        a leaked connection holds a pool slot forever)

Is the EVENT LOOP BLOCKED?  ►►► THE NODE-SPECIFIC KILLER ◀◀◀
   Symptom: the process is UP. `systemctl status` says active. It has a PID.
            And EVERY request times out, including /health.
   Because: one synchronous operation on the main thread means Node cannot
            call accept(), cannot read sockets, cannot run your callbacks.
            Node is not "handling requests slowly" — it is handling ZERO.

   ►► THE KERNEL-LEVEL TELL (topic 28 callback):
      ss -tlnp | grep 3000
      State   Recv-Q  Send-Q  Local Address:Port
      LISTEN  128     128     0.0.0.0:3000       ◀◀◀
      On a LISTENING socket, Recv-Q = the number of ESTABLISHED connections
      that have completed the TCP handshake and are sitting in the accept
      queue — because YOUR APP HAS NOT CALLED accept() ON THEM.
      Send-Q on a LISTEN socket = the backlog limit.
      Recv-Q pinned at the backlog = your app is not accepting. It is BLOCKED.
      (The kernel completed the handshake for you. That's why the client sees
       "connected" and then a timeout, not "connection refused".)

   ►► Then find the sync work: mpstat -P ALL (one core hot) → --cpu-prof.
   ►► Measure it continuously in prod:
      const { monitorEventLoopDelay } = require('perf_hooks');
      const h = monitorEventLoopDelay({ resolution: 20 }); h.enable();
      setInterval(() => console.log('p99 lag ms', h.percentile(99) / 1e6), 5000);
      Healthy: <10ms. Concerning: >50ms. Broken: >1000ms.

Is it FILE DESCRIPTOR exhaustion?  (topic 19 callback)
   Symptom: EMFILE: too many open files. New connections refused. Existing ok.
   ls /proc/PID/fd | wc -l        → current count
   cat /proc/PID/limits | grep files → the limit for THAT process
   ►► ulimit -n in your shell is NOT the limit of the running service.
      systemd services get LimitNOFILE= from their unit file (topic 22/33).
   Common cause: sockets or file handles never closed → an fd LEAK. Watch the
   count climb over an hour: watch -n5 'ls /proc/$(pgrep -f server.js)/fd | wc -l'

Is it DNS?  (topic 27 callback)
   Every outbound call doing a fresh, slow, or FAILING lookup adds 5s timeouts.
   dig +stats yourdb.internal   → query time
   getent hosts yourdb.internal → does the LIBC path (what Node uses) work?
   Classic: a bad entry in /etc/resolv.conf; a nameserver that's down and being
   waited on for 5s before falling through to the second one.
```

### BRANCH D — DISK FULL / SLOW

Full runbook is in **topic 31 — Disk Management**. The 20-second version:

```bash
df -h                    # is a filesystem at 100%? (writes now fail: ENOSPC)
df -i                    # INODES exhausted? "disk full" with 60% space free —
                         # millions of tiny session/cache files (topic 04)
du -xh --max-depth=1 / | sort -rh | head    # where did it go? (-x = one fs)
sudo lsof +L1            # DELETED-but-still-open files. You `rm`'d the 40GB log
                         # but the process still holds the fd, so the blocks are
                         # NOT freed. df says full, du says empty. Restart the
                         # process (or truncate via /proc/PID/fd/N). Topic 19.
iostat -xz 1             # slow, not full? → await + aqu-sz
```

---

## strace — Watching a Process Talk to the Kernel

`strace` uses the **`ptrace(2)`** syscall to attach to a process. The kernel then stops the target on *every single syscall entry and exit* and hands control to strace, which reads the registers, decodes the arguments, prints them, and resumes the target. (Topic 01: syscalls are the only door into the kernel — strace sits in the doorway with a clipboard.)

That "stops on every syscall" is both its superpower and its danger.

```
strace -f -T -tt -c -e trace=openat,read -s 200 -o out.txt -p 8234
│      │  │  │   │  │                     │      │          │
│      │  │  │   │  │                     │      │          └── attach to a
│      │  │  │   │  │                     │      │              RUNNING pid
│      │  │  │   │  │                     │      └── write to a file instead
│      │  │  │   │  │                     │          of stderr (which would
│      │  │  │   │  │                     │          otherwise pollute your pipe)
│      │  │  │   │  │                     └── print 200 bytes of string args
│      │  │  │   │  │                         (DEFAULT IS 32 — you WILL be
│      │  │  │   │  │                          reading truncated paths and
│      │  │  │   │  │                          truncated HTTP bodies otherwise)
│      │  │  │   │  └── only trace these calls. Also: -e trace=network,
│      │  │  │   │      -e trace=file, -e trace=%memory, -e trace=desc
│      │  │  │   └── ►► -c = SUMMARY HISTOGRAM. Count + time per syscall.
│      │  │  │          Prints NOTHING until you Ctrl-C. THE ONE YOU WANT
│      │  │  │          FIRST, ALWAYS. Cheapest and most informative.
│      │  │  └── -tt = wall-clock timestamp on each line (microseconds)
│      │  └── -T = show TIME SPENT IN EACH CALL, in <angle brackets> at EOL.
│      │         This is how you find the one read() that took 4 seconds.
│      └── ►► -f = FOLLOW forks and THREADS. For Node this is ESSENTIAL:
│              the libuv threadpool does your fs and dns work on OTHER threads.
│              Without -f you attach to the main thread, see an idle
│              epoll_wait() loop, and conclude "it's doing nothing". WRONG.
└── the strace binary
```

**Start with `-c`. Always.** It answers "what is this process actually doing?" in one screen:

```
$ sudo strace -c -f -p 8234
strace: Process 8234 attached with 7 threads
^C
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 71.02    4.812044          40    120301    120301 stat            ◀◀◀
 18.31    1.240418          31     39821           read
  6.44    0.436291          22     19832           epoll_wait
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    6.774112                183109    120301 total
```

**120,301 `stat()` calls in 30 seconds — and 120,301 of them ERRORED.** It is stat()-ing a file that does not exist, 4,000 times a second. That is almost always a module resolution loop, a missing config file being re-checked on every request, or a template engine with caching disabled. **You found it in one command, and you never had to read a single line of decoded syscall output.**

### THE KILLER USE CASE: a HUNG process

A process is at 0% CPU and doing nothing. No logs. No crash. It's just... stuck. You have no idea why. This is where strace is unbeatable — **attach and see the syscall it is blocked in *right now*.**

```bash
$ sudo strace -p 8234
strace: Process 8234 attached
read(23,                      # ← and then NOTHING. It just sits there.
```
It is blocked in `read()` on fd 23. What is fd 23? (Topic 19.)
```bash
$ sudo ls -l /proc/8234/fd/23
lrwx------ 1 app app 64 Jul 12 14:31 /proc/8234/fd/23 -> socket:[38472]
$ sudo ss -tnp | grep 38472
ESTAB  0  0   10.0.1.5:52344   10.0.4.9:5432   users:(("node",pid=8234,fd=23))
```
**fd 23 is a TCP socket to 10.0.4.9:5432 — that is Postgres.** Your app is not hung. Your app is patiently waiting for a database that is never going to answer. The bug is not in your app; it is a query with no timeout against a DB that is wedged (or a firewall that dropped the connection without an RST, so the socket will hang until the TCP keepalive timer fires — potentially *hours*).

**How to read what it's blocked in:**

| Blocked in... | Means |
|---|---|
| `read()` / `recvfrom()` on a socket | Waiting on a network dependency (DB, upstream API). Find the peer via /proc/PID/fd + ss. |
| `futex()` | **Lock contention.** Threads fighting over a mutex. (You'll see this in native addons and in V8's GC.) |
| `epoll_wait()` (returning quickly, over and over) | Normal! This is a healthy idle Node event loop. Not a bug. |
| `epoll_wait()` (blocked, forever) | Node is idle and waiting for work. If requests ARE arriving, the listener socket may be broken/not accepting. |
| `flock()` / `fcntl(F_SETLKW)` | Waiting on a file lock. Another process holds it. |
| `nanosleep()` | It's sleeping. Someone wrote a retry loop. |
| **NOTHING — strace prints no lines at all** | **It is NOT making syscalls. It is spinning in USERSPACE.** strace is the wrong tool. It is burning CPU inside V8 (a regex, an infinite loop). **Go profile instead: `perf top -p PID` or `--cpu-prof`.** |

Two more no-strace ways to see where a process is stuck (they cost nothing):
```bash
cat /proc/8234/wchan     # one word: the KERNEL function the process is sleeping in
                         # e.g. "ep_poll", "futex_wait_queue_me", "io_schedule"
sudo cat /proc/8234/stack  # the full kernel-side stack (needs root; kernel threads)
cat /proc/8234/status | grep State   # R (running) D (uninterruptible=disk!) S (sleep)
```
`State: D` (uninterruptible sleep) means it is stuck in the kernel on I/O and **cannot even be killed with SIGKILL**. That is a disk or NFS problem, not an app problem.

### ⚠️ THE WARNING YOU MUST NOT IGNORE

**strace is catastrophically expensive.** Every syscall the target makes now causes **two context switches into the tracer** and a stop/resume. A syscall that took 1μs now takes 100μs+.

- Overhead is commonly **10x–100x**. A busy Node process doing 50k syscalls/sec can effectively **stop responding** while traced.
- **Never leave strace attached to a hot production process.** You will turn a slow API into a dead API and you will have caused the outage you were investigating.
- Rules: (1) prefer `-c` over full tracing; (2) filter aggressively with `-e trace=`; (3) attach for **seconds**, not minutes; (4) `-o file` so decoding I/O doesn't add to the pain; (5) detach with Ctrl-C and **verify the process recovered**.
- If strace crashes or you kill it wrong, the target can be left in a stopped state. (`kill -CONT PID` to resume.)

**`ltrace`** does the same for *library* calls (glibc functions) rather than syscalls. Rarely useful for Node — V8 doesn't route through libc in an interesting way — but know it exists.

### The modern successor: eBPF

**eBPF** lets you attach small, verified programs *inside the kernel* to trace events, aggregating in kernel space and only passing summaries to userspace. Overhead is typically **<1–2%**, not 100x. **This is what you should reach for in production.** Learn the words, even if you never install them:

| Tool | What it shows |
|---|---|
| `bpftrace` | A one-liner tracing language. `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'` |
| `execsnoop` | Every process exec'd, system-wide, live. **Instantly answers "what is that cron job actually running?"** |
| `opensnoop` | Every file opened, by whom. The safe `strace -e trace=openat` for prod. |
| `biosnoop` | Every block-device I/O with latency. Which process caused that 400ms disk wait. |
| `tcpconnect` / `tcpretrans` | Every outbound TCP connection / every retransmit. |
| `bcc-tools` / `bpfcc-tools` (apt) | The whole collection. `apt install bpfcc-tools` on Ubuntu. |

---

## perf — The CPU Profiler

`perf` samples the CPU at a fixed frequency and records the call stack each time. It sees **everything** — kernel, libc, V8 internals, and (with help) your JavaScript.

```bash
sudo perf top -p 8234           # live "top for functions" — instant orientation
sudo perf record -F 99 -p 8234 -g -- sleep 30
                    │   │       │
                    │   │       └── -g = capture CALL GRAPHS (needed for flamegraphs)
                    │   └── attach to this PID
                    └── -F 99 = sample 99 times/sec. Deliberately NOT 100 —
                        an odd number avoids sampling in lockstep with periodic
                        timers, which would bias the sample.
sudo perf report                # interactive tree
sudo perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flame.svg
sudo perf stat -p 8234 -- sleep 10   # HARDWARE COUNTERS: instructions, cycles,
                                     # IPC, cache-misses, branch-misses.
                                     # Low IPC (<1.0) + high cache-miss = the code
                                     # is memory-bound, not compute-bound.
```

**Two things you MUST do or perf output is useless for Node:**

1. **`perf_event_paranoid`** — the kernel restricts perf by default.
   ```bash
   cat /proc/sys/kernel/perf_event_paranoid   # 4 or 3 = locked down
   sudo sysctl -w kernel.perf_event_paranoid=1
   ```
   (Or grant `CAP_PERFMON` to the binary. In Docker you need `--cap-add=SYS_ADMIN` or `--privileged` and a matching seccomp profile — this is why people profile in staging.)

2. **Make JS frames visible.** V8 JIT-compiles JavaScript to machine code at runtime. perf has no symbol table for it, so you get a wall of hex addresses.
   ```bash
   node --perf-basic-prof server.js
   # writes /tmp/perf-<PID>.map — a symbol map perf reads to resolve JIT addresses
   node --perf-basic-prof-only-functions server.js   # smaller map file
   ```
   Also build/run with frame pointers (`--call-graph fp` needs them; `--call-graph dwarf` works without but is much heavier). **Without this, your flamegraph is 90% `0x00007f2a...` and tells you nothing.**

**For most Node work, skip perf and use the built-in profiler** — it's zero-setup and it speaks JavaScript:

```bash
node --cpu-prof --cpu-prof-dir=/tmp/prof server.js
#   → on exit, writes CPU.<date>.<pid>.cpuprofile
#   → open chrome://inspect → or drag the file into Chrome DevTools → Performance
node --prof server.js && node --prof-process isolate-*.log > profile.txt
#   → a plain-text "ticks" summary. Read the [Summary] and [Bottom up] sections.
npx 0x -- node server.js      # generates an interactive flamegraph HTML. Easiest.
npx clinic doctor -- node server.js   # diagnoses: CPU? event loop? GC? I/O?
```

---

## Other Tools Worth Knowing

| Tool | Answers |
|---|---|
| `pidstat -d 1` | Which **process** is doing the disk I/O (iostat only tells you the device is busy) |
| `iotop -oPa` | Same, interactive. `-o` only active, `-P` processes not threads, `-a` accumulated |
| `nethogs` | Bandwidth **per process** — "who is saturating the NIC?" |
| `ss -tin` | Per-socket TCP internals: `rtt:`, `cwnd:`, `retrans:`, `bytes_retrans`. **A high retrans count is your PROOF of packet loss** — you can stop blaming the app. |
| `tcpdump -i eth0 -nn port 5432 -c 100` | The wire. Last resort, but definitive. |
| `/proc/PID/wchan` | One-word answer: what kernel function is it sleeping in |
| `/proc/PID/stack` | Full kernel-side stack (root) |
| `/proc/PID/status` | `State:`, `VmRSS:`, `Threads:`, `voluntary_ctxt_switches:` |
| `pstack PID` / `gdb -p PID` then `bt` | A **native** userspace stack. For a wedged native addon. |
| `sar -A` | **Historical** data — what did the box look like at 3am, before you woke up? |

---

## Exact Syntax Breakdown

```
vmstat 1 5
│      │ │
│      │ └── take 5 samples, then exit (omit → runs until Ctrl-C)
│      └── report every 1 second  ◀ the FIRST line is a since-boot AVERAGE. Ignore it.
└── "virtual memory statistics" — but really: CPU + memory + swap + io + system, in one line
```

```
mpstat -P ALL 1
│       │  │   │
│       │  │   └── interval, seconds
│       │  └── ALL = every core individually, PLUS an "all" summary row
│       └── -P = which Processor(s) to report
└── "multiprocessor statistics" — THE tool for spotting a pegged single core
```

```
iostat -xz 1
│       ││ │
│       ││ └── interval
│       │└── -z = omit devices with Zero activity (kills the noise from loop0..loop27)
│       └── -x = eXtended stats (await, aqu-sz, %util) — WITHOUT THIS IT IS USELESS
└── "input/output statistics"
```

```
ps -eo pid,ppid,rss,vsz,pcpu,etime,cmd --sort=-rss | head -10
│  │ │                                  │             │
│  │ │                                  │             └── top 10
│  │ │                                  └── sort DESCENDING by rss (the '-' = desc)
│  │ └── -o = pick EXACTLY these columns. etime = elapsed run time (great for
│  │        "how long has this been alive?" — a leak needs TIME)
│  └── -e = every process
└── ps
```

```
perf record -F 99 -p 8234 -g -- sleep 30
│    │       │     │       │     │
│    │       │     │       │     └── run for 30s. perf stops when this child exits.
│    │       │     │       └── capture call graphs (needed for a flamegraph)
│    │       │     └── attach to this PID (already running)
│    │       └── sample at 99 Hz (odd number, on purpose — avoids lockstep bias)
│    └── record to perf.data (then: perf report / perf script)
└── the perf binary (linux-tools-$(uname -r))
```

---

## Example 1 — Basic: Manufacture a Bottleneck and Watch It

```bash
# Terminal 1 — watch every resource at once
vmstat 1

# Terminal 2 — BURN ONE CORE (a busy loop in one shell)
yes > /dev/null &

# Back in terminal 1, watch:
#   'r' goes to 1, 'us' climbs. On a 4-core box, load creeps toward 1.00.
mpstat -P ALL 1 3
#   ►► ONE core shows ~100% %usr. The others idle. The 'all' row shows ~25%.
#      THIS IS EXACTLY WHAT A BLOCKED NODE EVENT LOOP LOOKS LIKE.

pidstat 1 3
#   ►► names the culprit: `yes`, ~100% CPU. Note the CPU column = which core.

kill %1        # stop it

# Now GENERATE DISK LOAD and watch a DIFFERENT resource light up
dd if=/dev/zero of=/tmp/bigfile bs=1M count=4096 oflag=direct &
iostat -xz 1 5
#   ►► w_await climbs, aqu-sz > 1, %util → 100. But %usr stays LOW.
#      In vmstat: 'wa' climbs and 'b' > 0, while 'us' stays low.
#      HIGH LOAD + LOW CPU = I/O. This is the pattern to burn into your brain.
rm /tmp/bigfile

# Now watch a HUNG process (SIGSTOP simulates "wedged")
sleep 300 & PID=$!
sudo strace -p $PID
#   ►► You see: nanosleep({tv_sec=300, ...}   — and nothing else. It's asleep.
#      Ctrl-C to detach.
cat /proc/$PID/wchan; echo
#   ►► hrtimer_nanosleep  — the kernel function it is parked in.
kill $PID
```

---

## Example 2 — Production Scenario

### INVESTIGATION A — "The API is timing out"

**14:31.** PagerDuty. p99 latency on `/api/orders` went from 80ms to 30s. Healthchecks are flapping. The Node process is *up*.

```bash
# ── STEP 1: CHARACTERIZE. What changed?
# Deploy at 14:12. Yes. That's suspicious. Note it, but keep going —
# "there was a deploy" is a lead, not a diagnosis.
# Symptom: LATENCY (not throughput). ALL endpoints (not one). SUDDEN.

# ── STEP 2: 60-SECOND TRIAGE
$ uptime
 14:33:02 up 61 days,  load average: 1.24, 1.19, 0.98
#   ►► LOAD IS 1.24 ON AN 8-CORE BOX. That is NOTHING. A junior engineer
#      stops here and says "the server is fine, it must be the database."
#      DO NOT STOP HERE.

$ dmesg | tail
[... nothing unusual — no OOM, no I/O errors ...]
#   ►► Good. Memory and disk are not the story.

$ vmstat 1 5
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 3891244  20488 981204    0    0     0    12  912 2104 13  1 86  0  0
#   ►► si/so = 0 (no swap). wa = 0 (no disk wait). st = 0 (no steal).
#      us = 13%. "The CPU is fine!"  ◀ THE AVERAGE IS LYING TO YOU.

$ mpstat -P ALL 1 3
14:33:20  CPU  %usr %nice %sys %iowait %irq %soft %steal %idle
14:33:21  all 12.94  0.00 0.87    0.00 0.00  0.12   0.00 86.07
14:33:21    0  1.00  0.00 1.00    0.00 0.00  0.00   0.00 98.00
14:33:21    1  2.02  0.00 0.00    0.00 0.00  0.00   0.00 97.98
14:33:21    2  0.99  0.00 0.99    0.00 0.00  0.00   0.00 98.02
14:33:21    3 99.01  0.00 0.99    0.00 0.00  0.00   0.00  0.00   ◀◀◀◀◀ THERE IT IS
14:33:21    4  1.00  0.00 0.00    0.00 0.00  0.00   0.00 99.00
14:33:21    5  0.00  0.00 1.01    0.00 0.00  0.00   0.00 98.99
14:33:21    6  2.00  0.00 0.00    0.00 0.00  0.00   0.00 98.00
14:33:21    7  1.01  0.00 1.01    0.00 0.00  0.00   0.00 97.98
#   ►► ONE CORE PEGGED. SEVEN IDLE. 8 cores → 1/8 = 12.5% ≈ the "all" row.
#      A single-threaded process is spinning. I have exactly one of those.

$ pidstat 1 3 | grep -v " 0.00 "
14:33:25   1001    9012   98.02    1.98    0.00  100.00    3  node
#   ►► PID 9012. node. 100% CPU. On core 3. Confirmed.
```

**Reasoning stated:** it's user-time (not system), it's one thread, it's Node. This is not a syscall storm — so **strace is the wrong tool**. Let me confirm the event loop is the victim, then profile.

```bash
# ── Is the event loop actually blocked? Ask the KERNEL. (topic 28)
$ ss -tlnp | grep 3000
State  Recv-Q Send-Q  Local Address:Port
LISTEN    511    511        0.0.0.0:3000  users:(("node",pid=9012,fd=20))
#   ►► Recv-Q = 511 = the FULL backlog. 511 clients have completed the TCP
#      handshake and are sitting in the kernel's accept queue because MY APP
#      HAS NOT CALLED accept(). It is not slow. It is NOT RUNNING THE LOOP.
#      This is why clients see a timeout, not a connection refused.

# ── STEP 3: DRILL IN. It's spinning in userspace → PROFILE, don't trace.
$ sudo strace -c -f -p 9012      # (30s) — sanity check, cheap
% time  seconds  usecs/call  calls  errors syscall
 89.11  0.004102        1      4021         epoll_wait
#   ►► Barely ANY syscalls at all. It is not talking to the kernel.
#      Confirmed: it's burning CPU inside V8. Detach immediately.

# Restart one instance with the profiler (pm2/systemd; do it on ONE node
# behind the LB, not all of them):
$ node --cpu-prof --cpu-prof-dir=/tmp/prof server.js
# ... let it take traffic for 60s, then SIGINT it ...
$ ls /tmp/prof
CPU.20260712.143512.9188.0.001.cpuprofile
# scp it down, drag into Chrome DevTools → Performance → load profile
```

**The flamegraph:** one enormous plateau, 94% of total width:

```
 processTicksAndRejections
   └─ handleRequest
       └─ validateOrderPayload      (orders/validate.js:42)
           └─ RegExp.prototype.test
               └─ RegExp::Exec  ████████████████████████████████  94%
```

Line 42, shipped in the 14:12 deploy:

```js
const SKU_RE = /^([A-Za-z0-9]+-?)+$/;      // ◀ CATASTROPHIC BACKTRACKING
if (!SKU_RE.test(req.body.sku)) return res.status(400)...
```

`([A-Za-z0-9]+-?)+` — nested quantifiers. On a 30-char non-matching input the regex engine explores ~2³⁰ paths. **One request with a bad SKU pegs a core for 40 seconds and blocks the event loop for EVERY user.** A scraper hitting the endpoint with junk SKUs is, accidentally, a DoS.

```js
// FIX — anchor it, no nested quantifiers, and bound the length FIRST.
const SKU_RE = /^[A-Za-z0-9]+(?:-[A-Za-z0-9]+)*$/;
if (req.body.sku.length > 64) return res.status(400).json({ error: 'sku too long' });
```

```bash
# ── STEP 4: PROVE THE FIX. Same metric. Same load.
$ mpstat -P ALL 1 3
14:58:11    3   4.02  0.00  0.99  0.00 0.00 0.00  0.00 94.99   ◀ core 3 is free
$ ss -tlnp | grep 3000
LISTEN    0    511    0.0.0.0:3000                              ◀ Recv-Q = 0
# p99: 30,000ms → 47ms.  ►► THAT is a proven fix.
```

---

### INVESTIGATION B — "The server dies every night at 3am"

The API is down every morning. It restarts itself (systemd `Restart=always`), so nobody noticed for a week. No JS stack trace in the logs — the process just **vanishes** mid-request.

```bash
# ── STEP 1: CHARACTERIZE. SPIKY + a CLOCK. "Every night at 3am" is the
#    single biggest clue in operations. Something is SCHEDULED.

# ── STEP 2: TRIAGE. dmesg FIRST.
$ sudo dmesg -T | grep -i "killed process"
[Sat Jul 12 03:04:17 2026] Out of memory: Killed process 9012 (node)
    total-vm:4194304kB, anon-rss:3801244kB, file-rss:0kB, shmem-rss:0kB,
    UID:1001 pgtables:7488kB oom_score_adj:0
#   ►► THE KERNEL OOM-KILLED MY API. Not a JS crash. Not a bug in my code.
#      (topic 17: this is why the exit code was 137 = 128 + SIGKILL(9).)

$ journalctl -u api.service --since "03:00" --until "03:10" | tail -3
Jul 12 03:04:17 web-1 systemd[1]: api.service: Main process exited,
    code=killed, status=9/KILL
Jul 12 03:04:17 web-1 systemd[1]: api.service: Failed with result 'signal'.
#   ►► systemd confirms: killed by signal 9. It didn't crash. It was EXECUTED.

# ── What ELSE runs at 3am? (topic 20)
$ sudo cat /etc/cron.d/reports
0 3 * * * app /usr/bin/node /srv/app/jobs/nightly-export.js >> /var/log/export.log 2>&1
#   ►► There it is. A nightly export job at exactly 03:00.

# ── Watch it happen. Reproduce on purpose:
$ sudo -u app node /srv/app/jobs/nightly-export.js &
$ watch -n2 'ps -eo pid,rss,etime,comm --sort=-rss | head -4'
  PID    RSS   ELAPSED COMMAND
 9012 412844      6-01 node          ← the API. steady ~400MB.
14882 2914112     03:41 node         ◀◀ the export job. 2.9 GB AND CLIMBING.
```

**The mechanism — and this is the part everyone gets wrong:**

```
The export job does:  const rows = await db.query('SELECT * FROM orders');
                      // 14 million rows. ALL of them. Into a JS array.
                      fs.writeFileSync('/tmp/export.csv', rows.map(toCsv).join('\n'));

  03:00  job starts.       RSS:  60MB.  Box: 8GB total, 4GB free.
  03:02  job at 1.5GB.     Box free: 2.5GB.  Page cache being evicted.
  03:04  job at 3.9GB.     Box free: ~0.   vmstat si/so start climbing.
  03:04  Kernel: no more reclaimable memory. INVOKE THE OOM KILLER.

  ►►► THE KERNEL DOES NOT KILL THE GUILTY PROCESS. ◀◀◀
      It kills the one with the highest oom_score — which is driven mostly by
      RSS. The export job is at 3.9GB. The API is at 400MB... but on THIS run
      the API had grown to 4.1GB of cache+heap, so it scored HIGHER.
      The kernel killed the INNOCENT process. It killed my API.
      Check the score yourself:  cat /proc/PID/oom_score
```

**The fix — two parts. Both are required.**

```js
// 1. FIX THE CAUSE — stream it. Never materialize a table in RAM.
const { pipeline } = require('stream/promises');
const copyTo = require('pg-copy-streams').to;
await pipeline(
  client.query(copyTo('COPY orders TO STDOUT WITH CSV')),
  fs.createWriteStream('/tmp/export.csv')
);
// RSS: 60MB, flat, regardless of table size.
```

```ini
# 2. BOUND THE BLAST RADIUS — make the job die instead of my API.
#    /etc/systemd/system/nightly-export.service   (topic 22/33; use a timer, not cron)
[Service]
Type=oneshot
User=app
ExecStart=/usr/bin/node /srv/app/jobs/nightly-export.js
MemoryMax=512M          # ◀ cgroup hard limit. If it exceeds this, the KERNEL
                        #   OOM-kills THIS CGROUP — not the whole machine.
                        #   The job dies. The API lives. You get an alert.
MemoryAccounting=yes
```
```bash
# Also protect the API explicitly (lower = less likely to be picked):
# /etc/systemd/system/api.service.d/oom.conf
[Service]
OOMScoreAdjust=-500
```
**Prove it:** run the job, watch `ps` — RSS flat at 60MB. Trigger the old version under the new unit — the *job* gets killed, the API keeps serving. Sleep through the night.

---

### INVESTIGATION C — "Everything is slow after the deploy"

Deployed at 16:00. By 18:30 *everything* — including static files served by nginx, including `ls` in your SSH session — is sluggish. It got **gradually** worse.

```bash
$ uptime
 18:41:22 up 9 days,  load average: 18.20, 14.80, 9.10     # 4-core box → 4.5x
$ vmstat 1 5
 r  b   swpd    free   buff  cache   si   so    bi     bo   in    cs us sy id wa st
 3  4 1998848   61240   3104  41208  892 1204  4012   1880 8102 21044 21 18  9 52  0
#         ▲▲▲▲                        ▲▲▲ ▲▲▲▲                            ▲▲
#   swpd: 2GB in swap.  si/so: ~1MB/s BOTH WAYS, sustained.  wa: 52%.
#   ►► THE BOX IS THRASHING. It is not doing work — it is moving memory
#      between RAM and disk, forever. The 52% iowait is swap I/O.
#      EVERYTHING on this box is slow, because everything must page in.
#      That's why even `ls` is slow. This is a WHOLE-MACHINE symptom.

$ free -m
              total   used   free  shared  buff/cache   available
Mem:           3934   3801     61     12          72         88      ◀ 88MB!!
Swap:          2047   1952     95

$ ps -eo pid,rss,etime,cmd --sort=-rss | head -3
  PID     RSS     ELAPSED CMD
 9012 3212844    02:41:09 node /srv/app/server.js     ◀ 3.2GB and it started 2h41m ago

# ── Is it GROWING? A leak needs TIME. Take two samples 10 minutes apart.
$ ps -o rss= -p 9012 ; sleep 600 ; ps -o rss= -p 9012
3212844
3488120      # ►► +275MB in 10 minutes. On a FLAT request rate. THAT IS A LEAK.
```

**The characterization was right:** GRADUAL degradation → a leak. And "what changed?" → the 16:00 deploy. The deploy added a new dependency, `@acme/geo-lookup`.

```bash
# ── STEP 3: DRILL IN. This is a JS-heap question, not a kernel question.
#    kill -USR1 opens the inspector on a RUNNING node process (default 127.0.0.1:9229)
$ kill -USR1 9012
$ journalctl -u api -n2
Debugger listening on ws://127.0.0.1:9229/...

# Tunnel it to your laptop (topic 24 — do NOT expose 9229 to the internet):
laptop$ ssh -L 9229:127.0.0.1:9229 deploy@web-1
laptop$ open chrome://inspect
```

In DevTools → **Memory** → **Heap snapshot**. Take one. Wait 10 minutes of traffic. Take another. **Select snapshot 2 → change the dropdown from "Summary" to "Comparison" → compare against snapshot 1.**

```
Constructor              # New   # Deleted  # Delta   Size Delta
─────────────────────────────────────────────────────────────────
(object)               418,204          12  418,192   +212 MB   ◀◀◀
Map                          1           0        1     +212 MB
string                 418,930         210  418,720    +64 MB
GeoResult              418,192           0  418,192   +148 MB
```

418,192 `GeoResult` objects created and **zero deleted**. Retained by a single `Map`. Click it → **Retainers** pane → `Map @123 ← _cache ← GeoLookup @456 ← module exports`.

```js
// node_modules/@acme/geo-lookup/index.js
const _cache = new Map();                  // ◀ module-level. Lives forever.
function lookup(ip) {
  if (_cache.has(ip)) return _cache.get(ip);
  const r = doLookup(ip);
  _cache.set(ip, r);                       // ◀ NEVER evicted. NEVER bounded.
  return r;                                //   One entry per UNIQUE IP, forever.
}
```
An unbounded cache keyed by client IP. Every new visitor permanently adds an entry. It grows exactly as fast as your unique-visitor count. **In dev, with one IP, it is invisible. In prod it is a time bomb with a 3-hour fuse.**

```js
// FIX — bound it. An unbounded cache is a memory leak with good PR.
const LRU = require('lru-cache');
const _cache = new LRU({ max: 10_000, ttl: 1000 * 60 * 60 });
```
```bash
# ── STEP 4: PROVE IT. Deploy, then watch RSS for an hour.
$ watch -n60 'ps -o rss=,etime= -p $(pgrep -f server.js)'
#   ►► RSS climbs to ~380MB and FLATTENS. vmstat si/so back to 0.
#      load average: 0.9. Even `ls` is fast again.
# ►► Add the guardrail so this can never take the box down again:
#    systemd: MemoryMax=1G  +  a Prometheus alert on process_resident_memory_bytes
```

---

## Common Mistakes

### Mistake 1 — Trusting the box-wide CPU average with a single-threaded runtime

**Wrong:** `top` says 12% CPU → "CPU isn't the problem." Move on to the database.
**Right:** `mpstat -P ALL 1`. One core at 100% on an 8-core box **is** 12.5% average.

**Root cause (kernel level):** the Linux scheduler runs your Node process's main JS thread on **one** core at a time. It cannot split a single thread across cores — that's not a Node limitation, it's what a thread *is*. The `all` row is `sum(cores)/nproc`. A fully saturated single thread is, by arithmetic, `100/nproc` percent. On a 32-core box that's **3%** — and it will look like the box is asleep while your API is dead.

**Prevention:** for Node, make `mpstat -P ALL 1` reflex #1 after `dmesg`. In `top`, press `1`.

---

### Mistake 2 — Reaching for strace before you know which resource is saturated

**Wrong:** app is slow → `sudo strace -f -p $(pgrep node)` on the prod API → a wall of `epoll_wait` → you learn nothing, and the API's p99 goes from 800ms to 40 seconds because you just added 100x syscall overhead. **You have now caused a worse outage than the one you were paged for.**
**Right:** triage first. strace only when you have a *specific* question ("what is it blocked on?" / "what syscall is it spamming?"). And then `-c`, filtered, for seconds.

**Root cause:** `ptrace` stops the tracee on **every** syscall entry and exit and context-switches to the tracer. A process doing 50,000 syscalls/sec now does 100,000 extra context switches/sec. That is not "some overhead." That is a different machine.

**Prevention:** `strace -c` (summary) or eBPF (`opensnoop`, `execsnoop`) in prod. Full strace in staging.

---

### Mistake 3 — Confusing the kernel OOM kill with V8's heap OOM (and applying the opposite fix)

**Wrong:** `node` died → "out of memory" → add `--max-old-space-size=8192` to the start command.
**Right:** find out **which** OOM. `dmesg | grep -i "killed process"`. Exit code 137 = the kernel killed you (SIGKILL, 128+9). Exit 134 + `FATAL ERROR: JavaScript heap out of memory` = V8 hit its own ceiling.

**Root cause:** these are opposite problems. If the *box* is out of RAM, raising V8's heap ceiling instructs Node to try to allocate **more** of the memory that does not exist — you will get OOM-killed **faster**, and the kernel may kill something else important (nginx, sshd, postgres) on the way. Raising the heap limit is only correct when the box genuinely has spare RAM and your working set genuinely needs it.

**Prevention:** always check `dmesg` and the exit code before you touch a flag. And in containers, set `--max-old-space-size` to roughly **75–80% of the container's memory limit**, so V8 starts GC'ing aggressively *before* the cgroup killer fires.

---

### Mistake 4 — "The box is idle, so it must be the database's fault" (and stopping there)

**Wrong:** CPU 2%, memory fine, disk fine, app slow → shrug → blame the DBA.
**Right:** an idle box with a slow app means the app is **WAITING**. That is a *finding*, not a dead end. Enumerate what it could be waiting on: DB query, DB **connection pool** (the pool is a queue and queues have latency!), an upstream HTTP call, DNS, a file lock, a mutex, or its own blocked event loop.

**Root cause:** you are measuring **utilization** and ignoring **saturation**. The connection pool is a saturated resource with 100% utilization and a 190-deep queue — and *no kernel tool in existence will show it to you*, because it is a JavaScript array. This is exactly why the USE method insists you check saturation for *software* resources too.

**Prevention:** instrument the pool (`pool.waitingCount`), instrument outbound call durations, and run `monitorEventLoopDelay`. The kernel cannot see your queues. You must export them.

---

### Mistake 5 — Not asking "what changed?" and not establishing a baseline

**Wrong:** two hours of heroic flamegraph analysis... on a performance profile that has looked exactly like that for six months. You "optimized" a function that was never the problem. The real cause was a config change at 14:12.
**Right:** ask what changed **first**. Deploys, feature flags, dependency bumps, schema migrations, DNS/cert changes, a new customer, a traffic spike, a cron job, a disk that quietly hit 100%.

**Root cause:** without a **baseline** you cannot recognize "abnormal." Is 40% CPU bad? *You have no idea* unless you know it's usually 8%. Every number in this doc is meaningless in isolation.

**Prevention:** keep historical data (`sar` is already collecting it; enable it). Keep a deploy log with timestamps and overlay it on your dashboards. **The first thing you should look at in any incident is a graph with a deploy marker on it.**

---

## Hands-On Proof

```bash
# PROVE IT: the box-wide CPU average hides a pegged core
yes > /dev/null &            # one busy loop
mpstat -P ALL 1 3            # ►► one core ~100%, 'all' row = 100/nproc
top                          # then press 1  ►► same story, per-core
kill %1

# PROVE IT: load average counts DISK WAIT, not just CPU (topic 18)
# In one terminal:
uptime                                    # note the load
dd if=/dev/urandom of=/tmp/x bs=1M count=2000 oflag=direct &
vmstat 1                                  # ►► 'b' > 0, 'wa' climbs, 'us' stays LOW
uptime                                    # ►► load rises with ~0% user CPU
rm /tmp/x

# PROVE IT: strace is ptrace, and ptrace stops the process on every syscall
strace -c ls /usr > /dev/null             # ►► a histogram of syscalls
time (for i in $(seq 200); do true; done) # baseline
time strace -f -o /dev/null bash -c 'for i in $(seq 200); do /bin/true; done'
#   ►► compare. The traced run is dramatically slower. That's the tax.

# PROVE IT: /proc tells you EXACTLY what a process is blocked on — free of charge
sleep 60 & PID=$!
cat /proc/$PID/wchan; echo                # ►► hrtimer_nanosleep
grep State /proc/$PID/status              # ►► S (interruptible sleep)
kill $PID

# PROVE IT: Recv-Q on a LISTEN socket = connections your app hasn't accept()ed
# (topic 28). Start a node server, block it with a sync loop, then hammer it:
node -e 'require("http").createServer((q,s)=>s.end("ok")).listen(3000, ()=>{
  setTimeout(()=>{const t=Date.now(); while(Date.now()-t<20000);}, 3000);})' &
sleep 4
for i in $(seq 60); do curl -s --max-time 1 localhost:3000 & done >/dev/null 2>&1
ss -tlnp | grep 3000
#   ►► Recv-Q climbs. The kernel completed the handshakes. Your app did not
#      accept them. The event loop is blocked. THIS is the fingerprint.

# PROVE IT: RSS is real memory, VSZ is a fantasy
ps -eo pid,rss,vsz,comm --sort=-rss | head -5
#   ►► VSZ is often 10-100x RSS. Node reserves huge virtual address ranges it
#      never touches. Panic about RSS. Ignore VSZ.

# PROVE IT: the kernel scores every process for OOM-kill eligibility
for p in $(pgrep -f node); do echo "$p $(cat /proc/$p/oom_score)"; done
#   ►► Higher = killed first. Mostly driven by RSS. This is how the kernel
#      chooses its victim — and why it kills your API instead of the batch job.
```

---

## Practice Exercises

### Exercise 1 — Easy

Run the full 60-second triage on any Linux box (a VM, a Docker container with `--pid=host`, or a cloud instance). For **each** of the ten commands, write down in one line: (a) what it told you, and (b) whether it was normal or abnormal — and *how you know*.

```bash
uptime; nproc; dmesg | tail; vmstat 1 5; mpstat -P ALL 1 3
pidstat 1 3; iostat -xz 1 3; free -m; sar -n DEV 1 3; top -b -n1 | head -15
```
Then: **what is this box's baseline?** Record it. You cannot spot abnormal without it.

---

### Exercise 2 — Medium

Write and run a Node server with a deliberately blocked event loop:

```js
// slow.js
const http = require('http');
function blockingHash(n){ const c=require('crypto'); let h='x';
  for(let i=0;i<n;i++) h=c.createHash('sha256').update(h).digest('hex'); return h; }
http.createServer((req,res)=>{ res.end(blockingHash(2_000_000)); }).listen(3000);
```
Now, from the terminal, **prove five separate things**:
1. It is CPU-bound on **one** core (`mpstat -P ALL 1`).
2. Which PID and which core (`pidstat 1`).
3. It is **not** doing syscalls (`strace -c -f -p PID` for 10s — near-zero calls).
4. The accept queue backs up under concurrency (`ss -tlnp | grep 3000` while running 50 concurrent `curl`s).
5. Which **function** is hot: run it with `node --cpu-prof slow.js`, load the `.cpuprofile` into Chrome DevTools, and find the widest plateau.

Then fix it (move the hashing to a `worker_threads` worker or use the async crypto API) and **prove** the fix by re-running steps 1 and 4.

---

### Exercise 3 — Hard (Production Simulation)

Reproduce **Investigation B** end to end, in Docker.

1. Write `leak.js`: an HTTP server that pushes a 1MB `Buffer` into a module-level array on every request.
2. Run it in a container with a hard memory limit: `docker run -m 512m --name leaky ...`
3. Drive load: `while true; do curl -s localhost:3000 > /dev/null; done`
4. In a second terminal, watch it die: `docker stats leaky` and `watch -n1 'docker exec leaky cat /sys/fs/cgroup/memory.current'`
5. When it dies, **prove what killed it**, three independent ways:
   - `docker inspect leaky --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` → expect `true 137`
   - `dmesg -T | grep -i "killed process"` on the **host** → the kernel's own record
   - the *absence* of a V8 `JavaScript heap out of memory` message → **prove it was the KERNEL, not V8**
6. Now make V8 die instead: rerun with `-m 2g --node-options="--max-old-space-size=256"`. You should now get exit **134** and a V8 fatal error. **Explain, in writing, why the same leaking code produced two completely different failure modes — and why the fixes are opposites.**
7. Finally: fix the leak (bound the array), and prove RSS flattens under sustained load.

---

## Mental Model Checkpoint

1. **What are the four hardware resources, and for each: what command shows Utilization, and what shows Saturation?**
2. **`top` says 12% CPU on an 8-core box and the app is dying. What is the very next command you run, and what do you expect to see?**
3. **`vmstat` shows `st` at 30%. What does that mean, and what is the correct action?**
4. **A process exits with code 137 and there is no JS stack trace. What killed it, how do you confirm it, and why is raising `--max-old-space-size` the WRONG fix?**
5. **The app is slow, the CPU is at 2%, memory is fine, and the disk is idle. Name four things it could be waiting on.**
6. **You attach `strace -p PID` and see literally nothing printed. What does that tell you, and what tool do you switch to?**
7. **In a flamegraph, what does the WIDTH of a frame mean? What does the x-axis position mean? Which frame do you actually look at?**
8. **What is `Recv-Q` on a LISTENING socket, and why does a non-zero value indict your event loop?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `uptime` | Load average (1/5/15 min) — compare to `nproc` | — |
| `dmesg -T \| tail` | Kernel ring buffer. **OOM kills, I/O errors. RUN FIRST.** | `-T` human timestamps, `-w` follow |
| `vmstat 1` | CPU+mem+swap+io+system in one line | `r`, `b`, `si/so`, `wa`, **`st`** |
| `mpstat -P ALL 1` | **Per-core** CPU. Finds the pegged single thread. | `-P ALL` |
| `pidstat 1` | Per-process CPU over time | `-d` disk I/O, `-r` memory, `-t` threads |
| `iostat -xz 1` | Per-device disk stats | `-x` extended (**await, aqu-sz**), `-z` skip idle |
| `free -m` | Memory — read the **available** column | `-m`, `-h`, `-s 1` repeat |
| `sar -n DEV 1` | NIC throughput; `sar -A` = **historical** | `-n DEV/EDEV/TCP,ETCP` |
| `top` / `htop` | Overview. **Press `1`** for per-core, `H` for threads | `-b -n1` batch (scriptable) |
| `ps -eo pid,rss,etime,cmd --sort=-rss` | Biggest processes by real memory | `--sort=-rss`, `-o` custom columns |
| `strace` | Syscall trace (ptrace). **10–100x slowdown!** | **`-c`** summary, `-f` threads, `-T` timing, `-e trace=`, `-s 200`, `-p PID` |
| `perf top -p PID` | Live CPU profile, kernel + native | `-p` |
| `perf record -F 99 -p PID -g` | Sampled profile → flamegraph | `-F` freq, `-g` call graphs |
| `node --cpu-prof` | V8 CPU profile → Chrome DevTools | `--cpu-prof-dir` |
| `node --inspect` / `kill -USR1 PID` | Open the inspector → **heap snapshots** | tunnel 9229 over SSH |
| `ss -tlnp` / `ss -tin` | Sockets. **Recv-Q on LISTEN = blocked loop.** `-i` = rtt/retrans | `-t -l -n -p -i` |
| `lsof +L1` | Deleted-but-open files eating disk | topic 31 |
| `iotop -oPa` / `nethogs` | Per-process disk / network usage | `-o` active only |
| `bpftrace`, `execsnoop`, `opensnoop`, `biosnoop` | **eBPF — low overhead, prod-safe tracing** | `apt install bpfcc-tools` |

---

## When Would I Use This at Work?

### Scenario 1: The 2am page — "checkout is timing out"
You SSH in. `dmesg | tail` — clean. `uptime` — load 1.1, looks fine. **You do not stop there.** `mpstat -P ALL 1` — core 5 at 100%. `pidstat 1` — it's node. `ss -tlnp` — Recv-Q is at the backlog limit: the event loop is blocked and the kernel is queueing customers who will never be served. `--cpu-prof` on one instance behind the LB names the function in 90 seconds. You roll back the deploy that introduced it, then fix it properly in the morning. **Total time: 12 minutes**, because you ran a checklist instead of guessing.

### Scenario 2: The gradual mystery — "it's been getting slower all quarter"
No incident, no page, just a p99 that has crept from 90ms to 700ms over three months. Nothing "changed." You check the four resources: all fine. So it's **waiting**. You `curl -w` the internal API — fast. You look at the DB: one query on `orders` went from 4ms to 600ms as the table grew past 8M rows. `EXPLAIN ANALYZE` → sequential scan → **a missing index**. Nothing changed in the *code*; the **data** changed. Gradual degradation with a healthy box is almost always data growth outrunning an index, or a slow leak.

### Scenario 3: The cost conversation — "can we downsize this instance?"
Your finance team wants to cut the 16-core box to 4 cores. You don't guess. You pull `sar -A` history for the last month: peak `r` (run queue) never exceeds 2, peak memory `available` never drops below 5GB, disk `%util` peaks at 4%. **But** `mpstat` history shows a nightly single-core pin during the export job. You downsize to 4 cores, move the export to a worker thread, and save 60% — and you have the data to defend it if it goes wrong. **This is the same method, run proactively instead of reactively.**

---

## How to Make the Next Investigation Easier

Everything above is *reactive*. Every hour you spend here is an hour you could have saved by having done these six things first:

| Do this | Why |
|---|---|
| **Establish a BASELINE** | You cannot recognize "abnormal" without knowing normal. Is 40% CPU bad? You have no idea. Write down your box's idle-state triage output *today*, while nothing is wrong. |
| **Keep HISTORICAL data** | `sar` is probably already installed (`sysstat`) — just enable collection. Better: `node_exporter` + Prometheus + Grafana. **The question "what did the box look like at 3am?" must be answerable at 9am.** Ad-hoc tools only show you *now*. |
| **STRUCTURED LOGS with request IDs and durations** | `{"reqId":"a3f","route":"/orders","ms":8421,"dbMs":8390}` — that one log line would have solved Investigation A without any of the tools in this doc. If your logs don't have durations, add them today. (Topic 23.) |
| **RED + USE dashboards** | **RED** for services: **R**ate, **E**rrors, **D**uration. **USE** for resources: **U**tilization, **S**aturation, **E**rrors. Two dashboards. Every incident starts on one of them. |
| **ALERT ON EVENT-LOOP LAG** | If you add exactly **one** Node metric, make it this. `monitorEventLoopDelay().percentile(99)`. It catches blocked loops, GC storms, and CPU-bound handlers — and it goes bad *before* your users notice. Alert at 100ms. |
| **A deploy log overlaid on your graphs** | "What changed?" should take **five seconds** to answer, not five minutes of asking in Slack. |

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 35 — Security Basics | The other thing that takes your box down at 3am. Same discipline: a checklist beats heroics. |
| **Builds on** | 16 — Processes in Depth | PIDs, /proc/PID, the process tree — every tool here reads them |
| **Builds on** | 17 — Process Management | Signals, exit codes (137 = SIGKILL = OOM), how to stop a runaway |
| **Builds on** | 18 — System Resources | load average, `free`'s `available` column, /proc/meminfo — the vocabulary of "busy" |
| **Builds on** | 19 — File Descriptors | fd exhaustion (EMFILE), `/proc/PID/fd`, `lsof` — how you identify the socket a process is hung on |
| **Builds on** | 28 — Ports and Sockets | `ss`, Recv-Q/Send-Q — the kernel-side proof that your event loop is blocked |
| **Builds on** | 31 — Disk Management | `df -i`, `lsof +L1` — the "disk full" branch of the tree |
| **Builds on** | 33 — Node in Production | systemd `MemoryMax`, `--max-old-space-size`, graceful shutdown, PM2 |
| **Builds on** | 20 — Cron and Scheduling | The 3am OOM kill was a cron job. Every "spiky at a fixed time" bug is a clock. |
| **Builds on** | 30 — curl and wget | `curl -w` timing breakdown — how you prove a slow dependency isn't your fault |
| **Builds on** | 01 — Linux Architecture | strace = ptrace = watching the syscall boundary you learned about in topic 01 |
| **Used by** | Every future incident you will ever work | This is the method. The tools change. The method does not. |
