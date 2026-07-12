# 18 — System Resource Monitoring

## ELI5 — The Simple Analogy

Imagine a **supermarket with 4 checkout lanes**.

- The **4 lanes** are your 4 CPU cores. Each can serve exactly one customer at a time.
- **Customers being served** are processes actually running on a core.
- **Customers standing in the queue** are processes that are *ready* to run but waiting for a free lane. They're not being served — they're just... waiting.
- Now here's the weird part, and it's the part everyone gets wrong: Linux ALSO counts **customers standing at the deli counter waiting for their ham to be sliced**. They're not in a checkout lane. They're not even queueing for one. They're blocked, waiting on a slow machine in the back. But they're still *in the store, demanding service*.

The **load average** is the average number of customers in the store who want something — being served, queued, or stuck at the deli.

So if someone tells you "load is 8!" the correct question is **not** "are the lanes busy?" It's **"is it the lanes, or is it the deli counter?"** Eight people in a 4-lane store could mean:
- 8 people fighting for 4 lanes (CPU saturation), **or**
- 0 people at the checkout and 8 people all stuck waiting for the deli slicer (disk I/O saturation).

**Same load number. Completely different problem. Completely different fix.**

Meanwhile, the **shelves** are your RAM. A supermarket manager who leaves shelves empty is a *bad manager*. Linux is a good manager: it fills every empty shelf with stock (page cache). When you look at `free` and panic that "free memory is only 200MB!", you're panicking that the shelves are full of goods. That's the shelves doing their job. What you actually want to know is: **how much could I clear out on demand?** That's the `available` column, and it's the only one that matters.

---

## Where This Lives in the Linux Stack

```
Hardware (CPU cores, RAM chips, disk controllers, hypervisor)
  │   ◀◀◀ These are the RESOURCES being measured
  │
  └── KERNEL
       │   • Scheduler maintains the run queue (runnable tasks)
       │   • MM subsystem tracks every page of RAM
       │   • Block layer tracks every I/O request and its latency
       │   • Kernel exports ALL of this as counters ◀◀◀ THIS TOPIC LIVES HERE
       │
       ├── /proc  (procfs — virtual filesystem, the kernel's public API)
       │     /proc/loadavg   /proc/meminfo   /proc/stat
       │     /proc/cpuinfo   /proc/diskstats /proc/vmstat
       │     /proc/PID/*     ◀◀◀ per-process counters
       │
       ├── /sys/fs/cgroup  (cgroupfs — the REAL limits inside a container)
       │
       └── System Calls (open/read on /proc — that's it, no magic)
            │
            └── C Library (glibc)
                 │
                 └── Shell (bash)
                      │
                      └── Commands: uptime, free, vmstat, iostat, mpstat, sar
                          ◀◀◀ These are just PRETTY PRINTERS FOR /proc
```

**The single most important structural fact in this doc:** `free`, `vmstat`, `top`, and `uptime` do not have secret kernel powers. They `open()` and `read()` text files in `/proc` and format the numbers. You can `cat` the same files yourself. Every tool here is a thin wrapper. Once you know that, you can debug a box that has no tools installed — a minimal Alpine container, say — because `cat /proc/meminfo` always works.

---

## What Is This?

System resource monitoring is reading the kernel's own counters to answer one question: **which of the four resources — CPU, memory, disk I/O, network — is the bottleneck right now?**

The kernel counts everything: how many tasks want a CPU, how many bytes of RAM are in which state, how long each disk request took, how many context switches happened. It publishes those counters in `/proc`. Every tool in this doc reads those counters, and most of them show you the *delta between two reads* — which is why `vmstat 1` is useful and `vmstat` on its own is nearly useless.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Load average includes D-state (disk) tasks | You'll see load 20 with 5% CPU, spend two hours "optimizing CPU", and never notice the disk is dying |
| `free`'s `available` column | You'll page the on-call at 3am because "free memory is 90MB" when the box has 6GB of reclaimable cache and is perfectly healthy |
| `st` (steal) in vmstat | You'll rewrite your hot path for a week when the real answer is "your noisy-neighbour VM is being starved by the hypervisor — resize the instance" |
| `mpstat -P ALL` (per-core) | `top` shows 12% CPU so you conclude "not CPU bound" — while your single-threaded Node event loop has one core pinned at 100% and 7 cores idle |
| Swap thrashing | Your DB box will go catatonic — responding to nothing, but never actually dying, so no health check fires and no alert triggers. Worse than a crash |
| cgroup limits vs `/proc` | Node reads 64GB of host RAM, sizes its heap for it, and gets OOM-killed at the 512MB container limit — with no error in your app logs |
| `sar` | Someone asks "what happened at 3am?" and you have literally no way to find out, because `top` only shows *now* |

---

## The Physical Reality

### Load average: what the kernel actually stores

The kernel keeps three fixed-point numbers. Every 5 seconds a timer fires, counts the tasks, and folds that count into an **exponentially damped moving average**:

```
        ┌──────────────── The kernel's run queue accounting ─────────────────┐
        │                                                                     │
        │  Every 5 seconds, count tasks whose state is:                      │
        │                                                                     │
        │    R  = TASK_RUNNING          (on a CPU, or queued waiting for one)│
        │    D  = TASK_UNINTERRUPTIBLE  (blocked in the kernel — usually     │
        │                                waiting on disk I/O or an NFS mount)│
        │                                                                     │
        │  NOT counted:                                                       │
        │    S  = interruptible sleep (waiting on a socket, a timer, input)  │
        │    Z  = zombie          T = stopped                                 │
        │                                                                     │
        │  nr_active = count(R) + count(D)                                    │
        │                                                                     │
        │  load1  = load1  * e^(-5/60)   + nr_active * (1 - e^(-5/60))       │
        │  load5  = load5  * e^(-5/300)  + nr_active * (1 - e^(-5/300))      │
        │  load15 = load15 * e^(-5/900)  + nr_active * (1 - e^(-5/900))      │
        │                                                                     │
        └─────────────────────────────────────────────────────────────────────┘
```

That `D` in there is the whole ballgame. **Linux is the only major Unix that includes uninterruptible-sleep tasks in load average.** On Solaris or BSD, load = CPU demand. On Linux, load = **demand for the machine**, CPU *and* disk.

Which means the sentence "load average is CPU usage" is **false on Linux**, and everyone who says it is about to misdiagnose an incident.

```
    LOAD = 8.0 on a 4-core box
    ────────────────────────────────────────────────────────────
    Interpretation A                Interpretation B
    ────────────────                ────────────────
      CPU: 100% busy                  CPU: 3% busy
      8 tasks in R state              0 tasks in R state
      0 tasks in D state              8 tasks in D state (stuck on disk)

      → CPU saturated                 → DISK saturated
      → scale out / optimise code     → the disk is your problem;
                                        code changes will do NOTHING

    Same number. Opposite root cause. You CANNOT tell them apart
    from `uptime` alone. That's why the next command is always `vmstat 1`.
```

### Memory: why "free" memory is a lie you tell yourself

```
┌──────────────────────── 8 GB of physical RAM ────────────────────────┐
│                                                                       │
│ ┌────────────┬────────────────────────────────┬─────────┬──────────┐ │
│ │ KERNEL     │  ANONYMOUS MEMORY              │  PAGE   │  free    │ │
│ │ (Slab,     │  (process heaps & stacks —     │  CACHE  │  (truly  │ │
│ │  page      │   your Node heap, Postgres     │  +      │  unused, │ │
│ │  tables)   │   shared_buffers…)             │  BUFFERS│  wasted) │ │
│ │            │                                │         │          │ │
│ │  NOT       │  NOT reclaimable without       │ MOSTLY  │          │ │
│ │  reclaim-  │  swapping                      │ RECLAIM-│          │ │
│ │  able      │                                │ ABLE ✓  │          │ │
│ └────────────┴────────────────────────────────┴─────────┴──────────┘ │
│                                                  └──────────┬───────┘ │
│                                                             │         │
│              MemAvailable ≈ free + (reclaimable cache/slab) ┘         │
│              ▲                                                        │
│              └── THE ONLY NUMBER THAT MATTERS                         │
└───────────────────────────────────────────────────────────────────────┘
```

When your Node app reads `/srv/app/dist/bundle.js`, the kernel copies the file's blocks into RAM **and keeps them there** in the page cache. Next read: no disk, memory speed, 1000x faster. The kernel will happily grow the page cache until "free" memory approaches zero — **on purpose**.

**Unused RAM is wasted RAM.** A long-running Linux box with lots of free memory is a box that isn't caching anything useful. A healthy busy server has *almost no free memory* and *lots of available memory*.

When a process then asks for memory and `free` is exhausted, the kernel simply **evicts clean page-cache pages** (they're already on disk — dropping them costs nothing) and hands the pages over. That eviction is invisible and fast. It is not an emergency. It's the design.

---

## How It Works — Step by Step

```
COMMAND: free -h
SHELL DOES:    fork(), execve("/usr/bin/free", ...)
KERNEL DOES:   procfs generates /proc/meminfo on the fly from MM subsystem counters
SYSCALLS:      openat("/proc/meminfo") → read() → close() → write(1, ...)
COMPUTATION:   free reads MemTotal/MemFree/Buffers/Cached/SReclaimable/MemAvailable,
               computes used = MemTotal - MemFree - Buffers - Cached - SReclaimable
OUTPUT:        the human-readable table to stdout (fd 1)
```

Prove it — `free` really is just reading a text file:

```bash
strace -e trace=openat free -h 2>&1 | grep proc
# openat(AT_FDCWD, "/proc/meminfo", O_RDONLY) = 3
```

```
COMMAND: vmstat 1
SHELL DOES:    fork(), execve("/usr/bin/vmstat", ["vmstat","1"])
VMSTAT DOES:   1. reads /proc/stat + /proc/meminfo + /proc/vmstat  → SNAPSHOT A
               2. prints the FIRST LINE — which is (counters ÷ seconds since boot),
                  i.e. THE AVERAGE SINCE BOOT. It is not "now". IGNORE IT.
               3. nanosleep(1 second)
               4. reads the same files                              → SNAPSHOT B
               5. prints (B - A) / 1s ← THIS is the real, current rate
               6. goto 3
SYSCALLS:      openat ×3, read (repeatedly), nanosleep, write
```

**This is why the first line of `vmstat` is always a lie.** It's the lifetime average of a box that's been up for 42 days. A box that was idle for 41 days and is on fire right now will report a beautifully calm first line. Look at the SECOND line and onward. This trips up literally everyone once.

---

## Load Average — Reading It Properly

```bash
uptime
# 14:23:07 up 42 days,  3:15,  2 users,  load average: 12.50, 8.30, 4.15
#                                                       │      │     │
#                                                       │      │     └── 15-min average
#                                                       │      └── 5-min average
#                                                       └── 1-min average
```

### Rule 1 — Always interpret against core count

```bash
nproc          # 4
```

| Load vs cores (on a 4-core box) | Meaning |
|---|---|
| `load 0.5` | Idle. ~12% demand. |
| `load 4.0` | **Exactly saturated.** Every core busy, nobody queueing. This is 100% utilisation with zero waiting — theoretically perfect. |
| `load 8.0` | 2x oversubscribed. On average 4 tasks are *waiting* at any moment. Latency is now doubled. |
| `load 40.0` | Something is very wrong. Likely a fork bomb, a thundering herd, or a hung disk parking hundreds of tasks in D state. |

### Rule 2 — The three numbers are a TREND, read them right-to-left

```
load average: 12.50, 8.30, 4.15      load average: 1.20, 6.80, 11.90
              ▲      ▲     ▲                       ▲     ▲     ▲
              now    5m    15m                     now   5m    15m

  15m=4 → 5m=8 → 1m=12                  15m=12 → 5m=7 → 1m=1
  RISING. It's getting worse.           FALLING. The incident is OVER.
  Something started ~15 min ago         You are looking at the aftermath.
  and is escalating. Act now.           Don't "fix" anything. Find out
                                        what happened (→ sar).
```

### Rule 3 — Load does NOT tell you CPU vs disk. `vmstat` does.

```bash
cat /proc/loadavg
# 12.50 8.30 4.15 6/1183 28934
#  │     │    │    │ │    │
#  │     │    │    │ │    └── PID of the most recently created process on the system
#  │     │    │    │ └── total number of tasks (processes + threads) that exist
#  │     │    │    └── number of tasks CURRENTLY runnable (R state) right this instant
#  │     │    └── 15-minute load average
#  │     └── 5-minute load average
#  └── 1-minute load average
```

That 4th field is gold: `6/1183` means **6 runnable** out of 1183 total tasks. If load is 12.5 but only 6 tasks are runnable, roughly 6 more are sitting in **D state on disk**. You just split the load in half with one `cat`.

---

## Memory — `free -h` and the Myth That Won't Die

```bash
free -h
#                total        used        free      shared  buff/cache   available
# Mem:           7.8Gi       3.1Gi       201Mi       128Mi       4.5Gi       4.3Gi
# Swap:          2.0Gi       512Mi       1.5Gi
```

```
  total       = MemTotal. Physical RAM the kernel can use (slightly less than the
                RAM you paid for — firmware reserves some).

  used        = total - free - buff/cache.  A DERIVED number, not a kernel counter.
                Roughly: memory held by processes (anonymous) + non-reclaimable kernel.

  free        = MemFree. Pages doing NOTHING AT ALL. You WANT this to be small.
                A large `free` on a busy long-lived server means the cache is cold.
                ⚠ THIS IS THE NUMBER PEOPLE PANIC ABOUT. IT IS THE LEAST USEFUL ONE.

  shared      = tmpfs / shared memory (/dev/shm, POSIX shm). Note: this lives INSIDE
                buff/cache, it is not a separate region.

  buff/cache  = Buffers + Cached + reclaimable Slab.
                • Buffers = raw block-device metadata (small)
                • Cached  = file contents cached in RAM (usually the bulk of it)
                This is the kernel using spare RAM productively. It is FREE MEMORY
                WEARING A HAT. The kernel will hand it back the instant you need it.

  available   = MemAvailable. The kernel's own ESTIMATE of how much memory you could
                allocate RIGHT NOW without pushing the system into swap.
                ≈ free + reclaimable(cache) + reclaimable(slab) − watermarks
                ★★★ THIS IS THE ONLY NUMBER THAT MATTERS. ★★★
                Alert on THIS. Graph THIS. Ignore `free`.
```

**In the output above:** free = 201Mi (scary!) but available = 4.3Gi (fine!). This box has **55% of its RAM available**. It is completely healthy. Anyone who alerts on `free` is going to page you every single night for a non-event.

### `/proc/meminfo` — the raw source

```bash
grep -E '^(MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty|Writeback|Slab)' /proc/meminfo
```

| Field | Meaning | Why you care |
|---|---|---|
| `MemTotal` | Usable physical RAM | Your denominator for everything |
| `MemFree` | Completely idle pages | Near-zero is NORMAL and GOOD |
| `MemAvailable` | Estimated allocatable without swapping | **The metric to alert on** |
| `Buffers` | Block-device metadata cache | Usually tiny; ignore |
| `Cached` | Page cache — file contents in RAM | Big = your working set is hot = fast |
| `SwapTotal` / `SwapFree` | Swap space size / unused | `SwapTotal 0` on most cloud/container hosts |
| `Dirty` | Modified pages **not yet written to disk** | A big, growing `Dirty` = your writes are outrunning the disk. A crash here loses data |
| `Writeback` | Dirty pages **currently being flushed** | Persistently non-zero = disk can't keep up |
| `Slab` | Kernel's own object cache (dentries, inodes) | A huge Slab can be an fd/dentry leak (→ topic 19) |

---

## Swap — And Why It's a Trap on a Latency-Sensitive Box

**Swap** is disk space the kernel uses as overflow RAM. When memory is tight, it picks cold anonymous pages (process heap that hasn't been touched in a while), writes them to disk, and frees the RAM.

```bash
cat /proc/sys/vm/swappiness
# 60      ← default. Higher = kernel more eager to swap anonymous memory out.
#            0 = only swap to avoid OOM. 100 = swap as readily as it drops cache.
sudo sysctl vm.swappiness=10     # runtime
# persist in /etc/sysctl.d/99-swap.conf → vm.swappiness=10
```

### The trap

Swapping a page back in is a **disk read on the critical path of a memory access**. Memory access: ~100 nanoseconds. Swap-in from SSD: ~100 *microseconds*. **A thousand times slower.** From spinning rust: 10ms — **a hundred thousand times slower.**

When your working set exceeds RAM, the kernel swaps a page out, the process immediately touches it again, so the kernel swaps it back in — evicting a different page that's *also* about to be needed. This is **thrashing**. The box now spends all its time moving pages instead of doing work.

```
   HEALTHY                          THRASHING
   ┌───────────────┐                ┌───────────────┐
   │ CPU: working  │                │ CPU: 2% us    │
   │ Disk: idle    │                │ Disk: 100% util│
   │ Requests: 5ms │                │ wa: 85%        │
   └───────────────┘                │ si/so: 40MB/s  │
                                    │ Requests: 45s  │
                                    │ SSH: 20s to    │
                                    │      echo a char│
                                    └───────────────┘
                                    Not down. Not up.
                                    UNDEAD.
```

**And undead is worse than dead.** An OOM kill is loud: the process dies, systemd restarts it, your health check fires, the load balancer pulls the node, you get an alert with a stack trace in `dmesg`. A thrashing box passes TCP health checks, keeps the connection open, answers *eventually*, and quietly poisons your entire request pipeline while every dashboard says "host: up". You will spend hours on it.

**Therefore:** on a database, a cache, or a latency-critical API box, most operators either disable swap entirely or set `vm.swappiness=1`. Take the loud, honest OOM kill over the silent catatonia.

**And note:** most cloud instances and **essentially all Docker containers have no swap at all.** In a container, there is no gentle degradation — you hit `memory.max` and the cgroup OOM killer fires **immediately**. Your Node process is gone. Exit code 137 (`128 + 9` = SIGKILL). Nothing in your app logs, because SIGKILL cannot be caught.

---

## `vmstat 1` — The Single Best "What Is This Box Doing" Command

If you only ever learn one command from this entire doc, learn this one. It shows CPU, memory, swap, I/O and context switching **in one line per second**.

```bash
vmstat 1
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 201234  84512 4512344    0    0    31    92  221  495  8  2 89  1  0   ← IGNORE (since boot)
 9  0      0 198112  84512 4512344    0    0     0     8 3021 8874 94  5  1  0  0
11  0      0 197004  84512 4511220    0    0     0     0 3155 9210 96  4  0  0  0
```

```
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │  │  │
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │  │  └─ st: STEAL. % of time your vCPU was READY to run but the
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │  │      HYPERVISOR gave the physical core to someone else.
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │  │      >0 sustained = a noisy neighbour is eating your CPU.
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │  │      YOU CANNOT FIX THIS IN CODE. Resize / move the instance.
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │  └──── wa: I/O WAIT. % of time CPUs were idle *with at least one
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  │         task blocked on disk I/O*. High wa = DISK IS THE BOTTLENECK.
 │  │     │     │      │      │     │    │     │     │    │    │   │  │  └─────── id: idle
 │  │     │     │      │      │     │    │     │     │    │    │   │  └────────── sy: kernel (system) time. Consistently high sy = syscall storm:
 │  │     │     │      │      │     │    │     │     │    │    │   │               too many small reads/writes, or context-switch churn.
 │  │     │     │      │      │     │    │     │     │    │    │   └───────────── us: user time. YOUR code (Node, Postgres) burning CPU.
 │  │     │     │      │      │     │    │     │     │    │    └───────────────── cs: CONTEXT SWITCHES per second. A jump from ~1k to ~100k
 │  │     │     │      │      │     │    │     │     │    │                        means too many threads/processes fighting for cores, or
 │  │     │     │      │      │     │    │     │     │    │                        lock contention. The CPU is busy *switching*, not working.
 │  │     │     │      │      │     │    │     │     │    └────────────────────── in: interrupts/sec (timer, NIC, disk). A NIC under heavy
 │  │     │     │      │      │     │    │     │     │                             packet load pushes this very high.
 │  │     │     │      │      │     │    │     │     └─────────────────────────── bo: blocks OUT to disk per second (writes)
 │  │     │     │      │      │     │    │     └───────────────────────────────── bi: blocks IN from disk per second (reads)
 │  │     │     │      │      │     │    └─────────────────────────────────────── so: pages SWAPPED OUT to disk /sec
 │  │     │     │      │      │     └──────────────────────────────────────────── si: pages SWAPPED IN from disk /sec
 │  │     │     │      │      │                                                    ★ ANY SUSTAINED si/so = YOU ARE THRASHING. Stop. Fix memory.
 │  │     │     │      │      └─────────────────────────────────────────────────── cache: page cache (KB) — file data in RAM. BIG IS GOOD.
 │  │     │     │      └────────────────────────────────────────────────────────── buff: buffer cache (KB)
 │  │     │     └───────────────────────────────────────────────────────────────── free: idle memory (KB). Small is fine. See `free -h` above.
 │  │     └─────────────────────────────────────────────────────────────────────── swpd: total virtual memory currently swapped OUT (KB).
 │  │                                                                                A non-zero swpd that is NOT CHANGING is harmless —
 │  │                                                                                it's cold pages parked on disk. Only si/so movement hurts.
 │  └───────────────────────────────────────────────────────────────────────────── b: tasks BLOCKED in uninterruptible sleep (D state — disk).
 │                                                                                   b > 0 sustained = I/O is the bottleneck.
 └──────────────────────────────────────────────────────────────────────────────── r: tasks RUNNABLE (running + waiting for a core).
                                                                                     ★ r > nproc means CPU QUEUEING. THIS is your CPU
                                                                                       saturation signal — far better than load average.
```

### The vmstat decision table — memorise this

| Signal | Bottleneck | What to do |
|---|---|---|
| `r` > cores, `us` high | **CPU (your code)** | Profile the app, scale out, add cores |
| `r` > cores, `sy` high | **CPU (kernel)** | Syscall storm / context-switch churn — batch I/O, reduce threads |
| `b` > 0, `wa` high | **Disk** | `iostat -xz 1` next. Faster disk, less I/O, more cache |
| `si`/`so` non-zero and moving | **Memory (thrashing)** | Free memory NOW, or add RAM. Nothing else matters |
| `st` > 0 sustained | **Hypervisor / noisy neighbour** | Not your fault. Resize/migrate the instance |
| `cs` exploded | **Too many threads / lock contention** | Reduce concurrency, check thread pools |
| Everything low, app still slow | **Not this box.** | It's a downstream DB, an external API, or DNS |

---

## `iostat -xz 1` — Is the Disk the Bottleneck?

From the `sysstat` package (`apt install sysstat` — it is **not** installed by default on Ubuntu).

```bash
iostat -xz 1
#        │  ││
#        │  │└── 1 = repeat every 1 second
#        │  └── -z = omit devices with ZERO activity (hides the noise)
#        └── -x = eXtended stats (the columns you actually need)
```

```
Device   r/s     w/s    rkB/s    wkB/s  rrqm/s  wrqm/s  r_await  w_await  aqu-sz  %util
nvme0n1  8.00  412.00   128.00 51200.00    0.00   12.00     0.42    38.10   15.72  99.60
         │      │        │       │                          │        │       │      │
         │      │        │       │                          │        │       │      └── %util: % of time the device had AT LEAST ONE
         │      │        │       │                          │        │       │           request in flight. ⚠ ON SSD/NVMe THIS IS
         │      │        │       │                          │        │       │           MISLEADING — those devices serve dozens of
         │      │        │       │                          │        │       │           requests in PARALLEL, so 100% util can still
         │      │        │       │                          │        │       │           mean "loafing". On a spinning disk (which
         │      │        │       │                          │        │       │           serves ONE request at a time) 100% really
         │      │        │       │                          │        │       │           does mean saturated. TRUST await, NOT %util.
         │      │        │       │                          │        │       └── aqu-sz: avg QUEUE LENGTH. >1 means requests are
         │      │        │       │                          │        │           piling up. 15 = deep queueing = real saturation.
         │      │        │       │                          │        └── w_await: avg ms a WRITE waited (queue + service).
         │      │        │       │                          │            ★ THE NUMBER THAT MATTERS.
         │      │        │       │                          │            NVMe healthy: <1ms. SSD: <5ms. Spinning: <20ms.
         │      │        │       │                          │            38ms on NVMe = the disk is DROWNING.
         │      │        │       │                          └── r_await: same, for reads
         │      │        │       └── wkB/s: KB written per second  (50 MB/s here)
         │      │        └── rkB/s: KB read per second
         │      └── w/s: write IOPS
         └── r/s: read IOPS
```

**The judgement call:** disk is the bottleneck when **`await` is high AND `aqu-sz` > 1**. Ignore `%util` on anything modern. A high `%util` with `await` of 0.3ms is a happy NVMe drive doing its job.

---

## `mpstat -P ALL 1` — The Per-Core View (Read This Twice, Node Devs)

```bash
mpstat -P ALL 1
#        │      │
#        │      └── 1 = interval, repeat every second
#        └── -P ALL = report EVERY core separately (not the average)
```

```
Linux 5.15.0-72-generic   07/12/2026   _x86_64_   (8 CPU)

14:23:08  CPU   %usr  %nice  %sys %iowait  %irq %soft %steal %guest  %idle
14:23:09  all   12.51   0.00  0.75    0.13  0.00  0.12   0.00   0.00  86.49
14:23:09    0    1.02   0.00  0.51    0.00  0.00  0.00   0.00   0.00  98.47
14:23:09    1   99.01   0.00  0.99    0.00  0.00  0.00   0.00   0.00   0.00   ◀◀◀ PEGGED
14:23:09    2    0.99   0.00  0.50    0.00  0.00  0.00   0.00   0.00  98.51
14:23:09    3    1.51   0.00  1.01    0.00  0.00  0.00   0.00   0.00  97.48
14:23:09    4    0.50   0.00  0.50    0.50  0.00  0.00   0.00   0.00  98.50
14:23:09    5    1.00   0.00  0.00    0.00  0.00  0.00   0.00   0.00  99.00
14:23:09    6    0.99   0.00  0.99    0.00  0.00  0.00   0.00   0.00  98.02
14:23:09    7    1.00   0.00  0.50    0.00  0.00  0.00   0.00   0.00  98.50
```

**Read the `all` row: 12.51% user. "The box is basically idle."**

**Read CPU 1: 99% user. Your application is completely, totally, 100% saturated.**

This is **exactly** what a CPU-bound Node.js process looks like, and it is one of the most valuable things in this entire curriculum for you specifically:

> **JavaScript executes on ONE thread.** The event loop runs on a single core. If you do heavy synchronous work — `JSON.parse` on a 40MB payload, a `bcrypt` round with a high cost factor, a regex with catastrophic backtracking, a big `.map().filter().reduce()` over 500k rows — **you saturate exactly one core.** The other 7 sit idle. The system-wide average is 100/8 = **12.5%**.
>
> So `top` says "12% CPU, plenty of headroom", your CloudWatch CPU alarm never fires, your autoscaler never scales — **and your API is timing out**, because the only thread that can run your JavaScript is completely blocked.

`mpstat -P ALL 1` is how you see it. One core at ~100%, the rest idle, `%idle` on the `all` row still looking comfortable. The instant you see that shape, you know: **the app is CPU-bound and single-threaded.** The fix is not a bigger instance — a bigger instance gives you more idle cores. The fix is `cluster`/PM2 with N workers, moving the hot work to a `worker_thread`, or making it async.

(Caveat: `%iowait` is per-core and somewhat arbitrary — the kernel attributes the wait to whichever core the task last ran on. Treat per-core iowait as a hint, and use `vmstat`'s `wa` / `iostat` for the real story.)

---

## `sar` — The Only Way to Answer "What Happened at 3am?"

`top` shows you *now*. At 9am, "now" is calm. The incident was at 3am and it's gone. Without historical data you are **blind**, permanently.

`sar` (System Activity Reporter, from `sysstat`) fixes this. Once enabled, a cron/timer job samples every 10 minutes and writes binary logs to `/var/log/sysstat/saDD` (one file per day of month). This is the **single highest-value thing you can install on a new server.**

```bash
sudo apt install sysstat
sudo sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat   # Debian/Ubuntu
sudo systemctl enable --now sysstat
```

```bash
sar -u                                  # CPU, today, every 10 min
sar -r                                  # memory
sar -q                                  # load average + run queue
sar -b                                  # I/O rates
sar -n DEV                              # network per interface
sar -S                                  # swap usage

sar -u -f /var/log/sysstat/sa11         # ← CPU on the 11th of this month
sar -r -s 03:00:00 -e 04:00:00 -f /var/log/sysstat/sa11
#      │  │                     │
#      │  │                     └── which day's file
#      │  └── -s / -e = start and end time window
#      └── -r = memory
```

```
$ sar -q -s 02:50:00 -e 03:20:00 -f /var/log/sysstat/sa11
02:50:01 AM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
03:00:01 AM         1       412      0.31      0.28      0.25         0
03:10:01 AM         2       415      0.44      0.35      0.29         0
03:20:01 AM        14      1893     41.20     22.15      9.80        38   ◀◀◀ THERE IT IS
                                                                      ▲
                                     plist-sz jumped 415 → 1893 ──────┘
                                     (a fork storm) AND blocked = 38 (disk)
                                     → your 03:15 backup cron forked hundreds of
                                       processes and pinned the disk. Case closed.
```

---

## The CONTAINER / cgroup Caveat — READ THIS TWICE

**Inside a Docker container, `free`, `top`, `nproc` and `htop` lie to you.**

Why? Because they read `/proc`, and **`/proc/meminfo` and `/proc/cpuinfo` are NOT namespaced.** Docker gives you a PID namespace, a network namespace, and a mount namespace — but the memory and CPU counters your container sees are **the host's**.

```
   ┌─────────────────────── HOST: 64 GB RAM, 32 cores ───────────────────────┐
   │                                                                          │
   │   ┌──────────── CONTAINER (docker run -m 512m --cpus=1) ─────────────┐  │
   │   │                                                                   │  │
   │   │   $ free -h                                                       │  │
   │   │   Mem:  62Gi  ...     ◀── THE HOST'S RAM. A LIE.                 │  │
   │   │   $ nproc                                                         │  │
   │   │   32                  ◀── THE HOST'S CORES. A LIE.               │  │
   │   │                                                                   │  │
   │   │   Node sees 62GB → sizes its old-space heap for a big machine     │  │
   │   │   libuv sees 32 cores → but UV_THREADPOOL_SIZE defaults to 4      │  │
   │   │   Some libs size worker pools from os.cpus().length → 32 workers  │  │
   │   │                                                                   │  │
   │   │   THE ACTUAL LIMIT (enforced by cgroups, invisible to /proc):     │  │
   │   │   memory.max = 536870912  (512 MB)                                │  │
   │   │                                                                   │  │
   │   │   → Heap grows past 512MB → cgroup OOM killer → SIGKILL           │  │
   │   │   → Container exits with code 137. NOTHING in your app logs.      │  │
   │   └───────────────────────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────────────────────────┘
```

### Read the REAL limits from cgroupfs

```bash
# --- cgroup v2 (Ubuntu 22.04+, Debian 12, modern Docker/K8s) ---
cat /sys/fs/cgroup/memory.max      # 536870912   ← your ACTUAL memory limit (or "max" = unlimited)
cat /sys/fs/cgroup/memory.current  # 481296384   ← how much you are using RIGHT NOW
cat /sys/fs/cgroup/cpu.max         # 100000 100000  ← quota period → 1.0 CPU
cat /sys/fs/cgroup/memory.stat     # detailed breakdown (anon, file, ...)

# --- cgroup v1 (older hosts) ---
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/memory.usage_in_bytes
cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/cpu.cfs_period_us

# --- Don't know which version? ---
stat -fc %T /sys/fs/cgroup   # "cgroup2fs" = v2, "tmpfs" = v1

# --- From OUTSIDE, on the host (the easy way) ---
docker stats --no-stream
# CONTAINER ID   NAME      CPU %    MEM USAGE / LIMIT     MEM %   NET I/O    BLOCK I/O
# a3f9c21b8e44   api       98.31%   478.2MiB / 512MiB     93.40%  1.2GB/88MB  0B/12MB
#                          ▲                    ▲          ▲
#                          │                    │          └── 93% of your cgroup limit.
#                          │                    │              You are ~30 seconds from OOM.
#                          │                    └── the REAL limit (docker reads cgroupfs)
#                          └── 98% of ONE cpu (--cpus=1). CPU-throttled.
```

### The Node.js fixes

```dockerfile
# Tell V8 the truth about how much heap it may use.
# Rule of thumb: ~75-80% of the container limit, leaving room for
# the Node binary, native buffers, and the C++ side of libuv.
ENV NODE_OPTIONS="--max-old-space-size=384"     # for a 512MB container

# libuv's threadpool (fs, dns, crypto, zlib) defaults to 4 threads regardless
# of the cgroup CPU limit. If you only get 1 CPU, 4 threads just thrash.
# If you get 8 CPUs and do lots of fs/crypto work, 4 is too few.
ENV UV_THREADPOOL_SIZE=4
```

Node ≥ 12 does honour cgroup memory limits when computing its *default* heap size in most cases — but **do not rely on it.** Set `--max-old-space-size` explicitly. It costs one line and it removes an entire class of 3am pages. And `os.cpus().length` still returns the **host's** core count, so any library that sizes a worker pool from it will over-provision massively.

---

## The 60-Second Triage Checklist (Runbook)

You've been paged. The Node API is timing out. You have SSH. **Run these nine commands in this order.** They take a minute and they will identify the bottleneck class in almost every incident. (This sequence is Brendan Gregg's; it has earned its reputation.)

```bash
# ── 1. How bad, and is it rising or recovering? ─────────────────────
uptime
#   load average: 24.10, 18.50, 6.20   → RISING FAST. Started ~10 min ago.

# ── 2. Did the KERNEL already tell us what's wrong? ─────────────────
dmesg | tail -20
#   Look for: "Out of memory: Killed process", "I/O error", "TCP: out of memory",
#   "nf_conntrack: table full", filesystem errors. The kernel often just tells you.

# ── 3. THE BIG ONE. CPU vs disk vs memory, all in one screen. ───────
vmstat 1 5
#   IGNORE LINE 1. Read r, b, si/so, wa, st.

# ── 4. Is it ONE core (single-threaded app) or ALL cores? ───────────
mpstat -P ALL 1 3
#   One core at 100%, rest idle = your Node event loop is blocked. ★

# ── 5. WHICH PROCESS is doing it? ───────────────────────────────────
pidstat 1 3
#   Per-process CPU, sampled — unlike `top`, this scrolls so you keep the history.

# ── 6. Is the disk the bottleneck? ──────────────────────────────────
iostat -xz 1 3
#   Read r_await / w_await / aqu-sz. Ignore %util on SSD.

# ── 7. Memory — and remember: read `available`, not `free`. ─────────
free -m
#   available near zero AND si/so moving = you are thrashing.

# ── 8. Is the NIC saturated? ────────────────────────────────────────
sar -n DEV 1 3
#   rxkB/s + txkB/s vs your instance's bandwidth cap. Also check sar -n TCP,ETCP 1
#   for retransmits (a sign of network trouble, not host trouble).

# ── 9. The overall picture / top offenders. ─────────────────────────
top
#   Now that you know WHAT the problem is, find WHO. Press `1` to split CPUs,
#   `M` to sort by memory, `P` to sort by CPU.
```

**INSIDE A CONTAINER, add:** `cat /sys/fs/cgroup/memory.current` and `/sys/fs/cgroup/memory.max` — because steps 3 and 7 are showing you the host, not you.

---

## Exact Syntax Breakdown

```
free -h -w -s 2
│    │  │  │
│    │  │  └── -s 2 = repeat every 2 SECONDS (continuous monitoring)
│    │  └── -w = WIDE — splits buff/cache into separate `buffers` and `cache` columns
│    └── -h = human-readable (Gi/Mi). Use -m for MB, -b for exact bytes.
└── free — reads /proc/meminfo, prints a table. That's the entire program.
```

```
vmstat 1 10
│      │ │
│      │ └── COUNT — print 10 samples then exit. (Omit = run forever.)
│      └── DELAY in seconds. ⚠ WITH A DELAY, THE FIRST LINE IS THE SINCE-BOOT
│          AVERAGE AND MUST BE IGNORED. Without a delay you get ONLY that
│          useless line — which is why bare `vmstat` is nearly worthless.
└── vmstat — Virtual Memory STATistics (it does far more than memory)

  Useful variants:
    vmstat -w 1     → wide output, columns don't collide on big-memory boxes
    vmstat -S M 1   → show memory in MB instead of KB
    vmstat -s       → one-shot dump of every counter, one per line
```

```
iostat -x -z -m 1
│      │  │  │  │
│      │  │  │  └── interval: 1 second
│      │  │  └── -m = report in MB/s instead of kB/s
│      │  └── -z = suppress devices with NO activity in this interval
│      └── -x = extended statistics (await, aqu-sz, %util — the ones that matter)
└── iostat (package: sysstat) — reads /proc/diskstats
```

```
mpstat -P ALL 1
│      │      │
│      │      └── interval
│      └── -P ALL = one row PER CPU. (-P 0 = just core 0.)
│                   Without -P you only get the average — which HIDES a single
│                   pegged core, i.e. it hides exactly the Node.js failure mode.
└── mpstat (package: sysstat) — reads /proc/stat per-CPU lines
```

```
sar -u -s 03:00:00 -e 04:00:00 -f /var/log/sysstat/sa11
│   │  │            │            │
│   │  │            │            └── -f = read from this SAVED file (sa + day-of-month)
│   │  │            └── -e = end time
│   │  └── -s = start time  (window the report)
│   └── -u = CPU. Others: -r memory, -q load, -b I/O, -n DEV network, -S swap, -A all
└── sar (package: sysstat) — replays historical binary logs. THE time machine.
```

---

## Example 1 — Basic

```bash
# 1. How many cores does the kernel think we have?
nproc
# 8

# 2. What's the current demand on the machine?
uptime
# 14:23:07 up 3 days, 2:11, 1 user, load average: 0.42, 0.51, 0.38
# → 0.42 on 8 cores ≈ 5% demand. Idle.

# 3. The raw kernel source of that number:
cat /proc/loadavg
# 0.42 0.51 0.38 1/412 28934
#                 ▲ only 1 task runnable right now, out of 412 total tasks

# 4. Memory. Look at `available`, not `free`.
free -h
#               total   used   free   shared  buff/cache   available
# Mem:          7.8Gi   1.2Gi  201Mi   128Mi       6.4Gi        6.2Gi
# → free is 201Mi. Do NOT panic. available is 6.2Gi. This box has 80% of its RAM
#   available. The 6.4Gi in buff/cache is the kernel caching files — a GOOD thing.

# 5. Now MAKE the page cache grow, and watch it happen:
free -m | grep Mem                      # note buff/cache
cat /var/log/syslog > /dev/null         # read a big file — kernel caches it
free -m | grep Mem                      # buff/cache went UP, free went DOWN,
                                        # available barely moved. THAT'S THE POINT.

# 6. Live system view. Ignore line 1.
vmstat 1 5

# 7. Per-core — is any single core pegged?
mpstat -P ALL 1 3
```

---

## Example 2 — Production Scenario

**2:14 AM.** PagerDuty: `api-prod-03 — p99 latency > 10s`. You SSH in.

```bash
$ uptime
 02:14:51 up 61 days, 4:02, 1 user, load average: 18.42, 11.30, 5.15
$ nproc
4
```

Load 18 on 4 cores, **rising** (15m=5 → 5m=11 → 1m=18) — started ~15 min ago, escalating. But CPU or disk? `uptime` can't say:

```bash
$ vmstat 1 5
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0  102400 512340  91232 2104488    2    5    41   118  310  620 22  3 74  1  0   ← (boot avg, ignore)
 0 14 1892352  48120  12004  184320 8912 11204 42300  9800 8200 41200  3  9  4 84  0
 0 16 1948160  44008  11800  176128 9420 12008 45120 10240 8410 43800  2 11  3 84  0
```

Read it: **`us`=3%** (CPU idle — discard the "optimise code" instinct), **`wa`=84%** (idle *waiting on disk*), **`b`=14-16** (that's where load 18 comes from), **`si`=9000/`so`=12000** (~36/48 MB/s of pure swap — **this box is THRASHING**), **`swpd` climbing 102M→1.9G**.

```bash
$ free -m
              total   used   free  shared  buff/cache  available
Mem:           3936   3612     44      12         280         98      # ★ available=98MB, not 6GB of cache
$ ps aux --sort=-rss | head -2
node 8234 4.1 71.2 4821004 2871240 ? Dl 01:58 0:44 node /srv/app/server.js   # RSS 2.8G/3.9G, STAT Dl
```

PID 8234 started **01:58** (matches the ramp), holds 2.8 GB, and its STAT `Dl` = **D-state, being swapped to death** — not "hung." `sar -r -s 01:30` confirms memory fell off a cliff 01:50→02:10, right after the 01:55 deploy. **A leak.**

```bash
$ sudo systemctl restart api      # mitigate; vmstat now shows si/so≈0 (swpd 1.9G is cold pages, harmless)
```

**Permanent fix:** run under systemd with a hard cap so a leak dies as a **loud, honest OOM kill** instead of silent multi-hour thrash (→ topic 22): `MemoryMax=2G`, `MemorySwapMax=0`, `Restart=always`, `Environment=NODE_OPTIONS=--max-old-space-size=1536`. Then chase the leak with a heap snapshot. **But you found the shape in 90 seconds with `vmstat`.**

---

## Common Mistakes

### Mistake 1: "Load average is 8, so the CPU is at 200%"

**Wrong:** Load = CPU utilisation.
**Right:** On Linux, load = runnable tasks **+ tasks in uninterruptible (D) sleep**, i.e. mostly disk I/O.

**Root cause at the kernel level:** `calc_load()` sums `nr_running` **and** `nr_uninterruptible` from every CPU's run queue. A task blocked in `read()` on a stalled NFS mount is in `TASK_UNINTERRUPTIBLE`, consuming zero CPU, and it still adds `1.0` to your load average. This is a deliberate Linux design choice (dating to a 1993 patch) that no other Unix copies.

**Diagnose:** `vmstat 1` — is it `r` (CPU) or `b` (blocked)? Or list the D-state tasks directly:
```bash
ps -eo state,pid,comm | awk '$1=="D"'
```
**Prevent:** never conclude anything from `uptime` alone. It is a *symptom detector*, not a diagnosis.

---

### Mistake 2: "free shows only 200MB free — we're out of RAM, add more!"

**Wrong:** Alerting on the `free` column.
**Right:** Alert on `available` (= `MemAvailable` in `/proc/meminfo`).

**Root cause:** The kernel deliberately consumes all otherwise-idle RAM as page cache, because a cached disk block is ~1000x faster to serve than a real disk read, and there is **zero downside** — clean cache pages are dropped instantly when a process needs the memory. `MemFree` trending to zero on a busy long-uptime server is the system working *correctly*.

**Diagnose:**
```bash
free -h                                     # compare `free` vs `available`
grep MemAvailable /proc/meminfo
# You can even force-prove it (⚠ do NOT do this in prod — it cold-starts every cache):
#   sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
# → `free` jumps up, `available` barely changes. Because it was ALWAYS available.
```
**Prevent:** in Prometheus/CloudWatch/Datadog, alert on `MemAvailable / MemTotal < 10%`. Never on `MemFree`.

---

### Mistake 3: Reading the FIRST line of `vmstat` / `iostat`

**Wrong:**
```bash
$ vmstat 1
 r  b   swpd   free  ...  us sy id wa st
 0  0      0 201234  ...   8  2 89  1  0    ← "Only 8% CPU, we're fine!"
```
**Right:** That line is **the average since boot.** On a box up for 61 days, an incident that started 10 minutes ago is diluted by 61 days of idle. It is meaningless. **Read from the second line down.**

**Root cause:** `/proc/stat` exposes *monotonic cumulative counters* (total jiffies spent in each state since boot), not rates. To produce a rate you need two samples. On its first pass `vmstat` has only one — so it divides the cumulative totals by uptime. Same for `iostat`.

**Prevent:** run `vmstat 1 5` and mentally strike out line 1. Some people alias `vmstat 1 | tail -n +4` (skip the two headers and the boot-average row).

---

### Mistake 4: "`top` shows 12% CPU, so my Node app isn't CPU-bound"

**Wrong:** Trusting the *aggregate* CPU figure for a *single-threaded* application.
**Right:** Node runs your JavaScript on **one thread, on one core.** One core pegged out of 8 = **12.5% system-wide** — which looks perfectly healthy on every dashboard you own.

**Root cause:** The kernel scheduler reports per-CPU busy time. `top`'s summary line averages across all cores. A saturated single-threaded process is *invisible* in that average.

**Diagnose:**
```bash
mpstat -P ALL 1               # one core ~100%, all others idle → confirmed
top                           # then press `1` to expand per-CPU lines
top -H -p $(pgrep -f server.js)   # per-THREAD view: one thread at 100% = the event loop
```
**Fix:** cluster mode / PM2 `-i max`, offload CPU-heavy work to `worker_threads`, or make the operation async. **A bigger instance will not help** — it just gives you more idle cores to look at.

---

### Mistake 5: "The container reports 64GB — plenty of headroom"

**Wrong:** Trusting `free`, `top`, `nproc` or `os.totalmem()` inside a container.
**Right:** `/proc/meminfo` and `/proc/cpuinfo` are **not namespaced**. You are reading the *host's* numbers. Your real limits live in **cgroupfs**.

**Root cause:** Docker isolates PIDs, network, and mounts via namespaces, but memory/CPU are limited via **cgroups**, which are an *accounting and enforcement* mechanism — not a *visibility* one. Nothing rewrites `/proc/meminfo` for you (unless you run `lxcfs`, which very few people do).

**Diagnose:**
```bash
cat /sys/fs/cgroup/memory.max        # v2: the truth
cat /sys/fs/cgroup/memory.current    # v2: usage right now
docker stats --no-stream             # from the host
docker inspect <c> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
# true 137   ← 137 = 128 + 9 (SIGKILL). The cgroup OOM killer got you.
```
**Prevent:** `ENV NODE_OPTIONS="--max-old-space-size=<~75% of limit>"`, set `UV_THREADPOOL_SIZE` explicitly, and never size a worker pool from `os.cpus().length` in a container.

---

### Mistake 6: "Swap is enabled, so we're protected from OOM"

**Wrong:** Swap as a safety net on a latency-sensitive box.
**Right:** Swap converts a **fast, loud failure** (OOM kill → restart → alert) into a **slow, silent, invisible one** (thrashing → 40-second responses → health checks still pass → nobody is paged).

**Root cause:** Every page fault on a swapped-out page is a synchronous disk read *inside a memory access*. When the working set exceeds RAM, page-in/page-out becomes a self-reinforcing loop and the box does nothing but shuffle pages.

**Diagnose:** `vmstat 1` — sustained non-zero `si`/`so`. (A large but *static* `swpd` with `si`/`so` at zero is harmless — those are cold pages, correctly parked.)
**Prevent:** `vm.swappiness=1` on DB/API boxes, or no swap at all, plus a `MemoryMax=` cgroup limit in the systemd unit so a leak dies loudly at a threshold *you* chose.

---

## Hands-On Proof

```bash
# PROVE IT: every tool here is just reading a text file in /proc
strace -e trace=openat free -h 2>&1 | grep proc
# openat(AT_FDCWD, "/proc/meminfo", O_RDONLY|O_CLOEXEC) = 3     ← that's the whole program

# PROVE IT: load average is 3 numbers the kernel literally writes into a file
cat /proc/loadavg
# 0.42 0.51 0.38 1/412 28934
# fields: 1m  5m  15m  runnable/total  last-PID

# PROVE IT: D-state (disk-blocked) tasks are counted in load, using zero CPU.
dd if=/dev/zero of=/tmp/big bs=1M count=4096 oflag=direct 2>/dev/null   # in one terminal
vmstat 1     # in another: `b` and `wa` climb while `us` stays ~0, yet LOAD RISES. The lesson.
rm /tmp/big

# PROVE IT: the kernel uses free RAM as page cache and gives it back on demand
free -m; dd if=/dev/zero of=/tmp/c bs=1M count=1024; sync; free -m
#   → buff/cache UP ~1G, free DOWN ~1G, `available` ≈ UNCHANGED. It was never lost. rm /tmp/c

# PROVE IT: JavaScript pegs exactly ONE core
node -e 'while(true){Math.sqrt(Math.random())}' &   ;   mpstat -P ALL 1 3   ;   kill %1
#   → ONE core ~100% %usr, all others idle; the `all` row still looks fine.

# PROVE IT: /proc/stat holds cumulative jiffies (why vmstat's first line is useless)
grep '^cpu ' /proc/stat; sleep 1; grep '^cpu ' /proc/stat   # numbers only go UP; rate = Δ/interval

# PROVE IT: inside a container /proc lies and cgroupfs tells the truth
docker run --rm -m 256m alpine sh -c 'free -m | head -2; cat /sys/fs/cgroup/memory.max'
#   → free shows the HOST's GBs; memory.max shows 268435456 (256MB). Trust the second.

# PROVE IT: cores the kernel sees vs nproc
grep -c ^processor /proc/cpuinfo; nproc; lscpu | grep -E '^(CPU\(s\)|Model name)'
```

---

## Practice Exercises

### Exercise 1 — Easy

Establish a baseline for a machine you own (or a container):

```bash
nproc
uptime
cat /proc/loadavg
free -h
grep -E 'MemTotal|MemFree|MemAvailable|Cached|SwapTotal' /proc/meminfo
vmstat 1 5
```

**Questions:** How many cores? What is your load as a *percentage of capacity* (load ÷ cores)? What is the gap between `free` and `available`, and where did that memory go? In your `vmstat 1 5` output, which line did you ignore and why?

Paste your terminal output.

---

### Exercise 2 — Medium

**Manufacture each bottleneck and identify it from `vmstat` alone.** Run `vmstat 1` in one terminal and each of these in another. For each one, write down what `r`, `b`, `us`, `sy`, `wa`, and `cs` did.

```bash
# A) CPU-bound, single-threaded (the Node shape):
node -e 'while(true){Math.sqrt(Math.random())}'
#    also run: mpstat -P ALL 1

# B) CPU-bound, all cores:
for i in $(seq $(nproc)); do (while :; do :; done) & done
#    kill them: kill $(jobs -p)

# C) I/O-bound (bypass the page cache with O_DIRECT so it really hits disk):
dd if=/dev/zero of=/tmp/io.test bs=1M count=4096 oflag=direct

# D) Context-switch storm:
for i in $(seq 200); do (while :; do sleep 0.001; done) & done
#    kill them: kill $(jobs -p)
```

**Question:** Cases A and B both produce high load. What in `vmstat` and `mpstat` distinguishes them? Case C produces high load too — what makes it unmistakably *different* from A and B?

---

### Exercise 3 — Hard (Production Simulation)

You have SSH on a 4-core box. The Node API is timing out. `uptime` shows `load average: 16.20, 9.80, 3.40`.

1. Run the full **60-second triage checklist**, in order, capturing output.
2. From that output alone, classify the bottleneck: **CPU / memory / disk / network / steal / not-this-box**. State the *specific columns* that justify your answer. Guessing is not allowed — cite the numbers.
3. Enable `sysstat`, wait 20 minutes, then use `sar -q -s <20 min ago>` to prove your finding *retroactively*, the way you'd have to at 9am the next morning.
4. Run the same triage **inside a memory-limited container** (`docker run -it --rm -m 256m ubuntu`). Which of the nine commands now give you **wrong** answers, and what do you replace them with?
5. Write a one-paragraph incident summary: root cause, evidence, immediate mitigation, permanent fix.

---

## Mental Model Checkpoint

Answer from memory.

1. **Load average counts tasks in two states. Which two — and which of them is uniquely a Linux thing?** Why does that make "load = CPU usage" false on Linux?
2. **A 4-core box shows load 9.0 and 2% CPU. What is the bottleneck, and which single command confirms it?**
3. **In `free -h`, which column should you alert on, and why is the `free` column nearly useless on a healthy server?**
4. **Why must you ignore the first line of `vmstat 1`?** What is that line actually reporting?
5. **What does the `st` column mean, and why can you never fix it in application code?**
6. **`top` says 12% CPU on an 8-core box, but your Node API is timing out. What is happening, and which command reveals it?**
7. **Inside a 512MB Docker container, `free -h` reports 62GB. Why? Where do you read the real limit?**
8. **Why is a thrashing server operationally worse than one that got OOM-killed?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `uptime` | Load average + uptime | — (read 1/5/15 as a *trend*) |
| `cat /proc/loadavg` | Raw load + runnable/total + last PID | — |
| `nproc` | Core count — your load *denominator* | ⚠ shows HOST cores in a container |
| `free` | Memory summary | `-h` human, `-m` MB, `-w` split buff/cache, `-s N` repeat |
| `cat /proc/meminfo` | Every memory counter, raw | `MemAvailable` is the one that matters |
| `vmstat` | **The #1 triage tool.** CPU+mem+swap+io+cs | `1` interval, `-w` wide, `-S M` in MB. **Ignore line 1** |
| `iostat` | Per-disk I/O stats (pkg: `sysstat`) | `-x` extended, `-z` skip idle devs, `-m` MB/s |
| `mpstat` | Per-CPU stats (pkg: `sysstat`) | `-P ALL` **per-core — finds the pegged Node core** |
| `pidstat` | Per-process stats, sampled (pkg: `sysstat`) | `1`, `-d` disk, `-r` memory, `-t` threads |
| `sar` | **Historical** replay (pkg: `sysstat`) | `-u` cpu `-r` mem `-q` load `-b` io `-n DEV` net `-f <file>` `-s/-e` window |
| `top` | Live process view | `1` per-CPU, `M` sort by mem, `P` sort by CPU, `-H` threads |
| `lscpu` | CPU topology, cache sizes, virtualization | — |
| `dmesg` | Kernel ring buffer — OOM kills, disk errors | `-T` human timestamps, `\| tail` |
| `docker stats` | Real container CPU/mem **vs the cgroup limit** | `--no-stream` |
| `cat /sys/fs/cgroup/memory.max` | The **actual** container memory limit (v2) | `memory.current` = usage now |

---

## When Would I Use This at Work?

### Scenario 1: The autoscaler never fires but the API is dying
Latency alarms are firing; the CloudWatch **CPU** alarm is flat at 13%, so the ASG never scales out. You run `mpstat -P ALL 1` and see core 3 at 99% while cores 0-7 idle. Your Node process is doing a synchronous `bcrypt` at cost factor 14 on the event loop. The aggregate metric was mathematically incapable of ever detecting this. **Fix:** PM2 cluster mode across all cores + move bcrypt to a `worker_thread`. **And** change the alarm to watch event-loop lag, not CPU.

### Scenario 2: "Add more RAM, the server only has 200MB free!"
A teammate escalates. You run `free -h`, see `available: 5.9Gi` out of 8Gi, and explain that the 6GB in `buff/cache` is the kernel doing its job — evictable in microseconds. You then confirm with `vmstat 1` that `si`/`so` are both zero: the box has never once needed to swap. **Nothing is wrong.** You saved a pointless instance upgrade and fixed the monitoring to alert on `MemAvailable` instead.

### Scenario 3: The container that dies with no logs
CI is green, the image builds, but in production the pod restarts every ~20 minutes with **exit code 137** and *nothing* in the app logs. 137 = 128+9 = **SIGKILL** — which cannot be caught, so no `process.on('SIGTERM')` handler ever ran and no error was logged. `kubectl describe pod` shows `OOMKilled`. Inside the container `free -h` said 64GB (the node's RAM) so V8 sized its heap for a giant machine, but `/sys/fs/cgroup/memory.max` says `536870912`. **Fix:** `NODE_OPTIONS=--max-old-space-size=384` and raise the memory limit to match the real working set.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | The kernel owns CPU/memory scheduling; `/proc` is how it reports on itself |
| **Builds on** | 02 — The Filesystem Hierarchy | `/proc` and `/sys` are virtual filesystems, not real files on disk |
| **Builds on** | 16 — Processes in Depth | Load average counts *task states* (R, D) — you need those from 16 |
| **Builds on** | 17 — Process Management | `top`/`ps` show *who*; this topic shows *what resource* |
| **Next** | 19 — File Descriptors | The other resource that runs out on a Node box — and it isn't in `free` |
| **Used by** | 20 — Cron and Scheduled Tasks | A misbehaving cron job is a classic cause of a 3am load spike (see the `sar` example) |
| **Used by** | 22 — systemd and Services | `MemoryMax=` / `CPUQuota=` set the cgroup limits this doc teaches you to read |
| **Used by** | 31 — Disk Management | `iostat` tells you the disk is slow; topic 31 tells you what to do about it |
| **Used by** | 33 — Node.js in Production | `--max-old-space-size`, `UV_THREADPOOL_SIZE`, cluster mode |
| **Used by** | 34 — Performance Investigation | This doc finds the bottleneck *class*; 34 goes deep with `strace` and `perf` |
