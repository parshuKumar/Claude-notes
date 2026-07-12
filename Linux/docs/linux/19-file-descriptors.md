# 19 — File Descriptors

## ELI5 — The Simple Analogy

Imagine a **hotel coat check**.

You walk in wearing a coat. You hand it to the attendant. The attendant hangs it on a hook and gives you a **little numbered ticket: 3**.

That ticket is a **file descriptor**.

Notice what the ticket is *not*:
- It is **not** your coat. It's a scrap of cardboard with a number on it.
- It **doesn't describe** your coat. There's no "brown wool, size M" written on it.
- It's **useless anywhere else.** Ticket #3 at *this* hotel gets you your coat. Ticket #3 at the hotel across the street gets you a stranger's umbrella. The number only means something *inside this hotel*.

When you want your coat, you don't describe it — you hand over ticket 3 and say "give me my coat." The attendant looks up hook 3 and does the work. **You never touch the coat rack.**

Now the three-level structure — and this is the part that explains *everything*:

- **Your ticket** (3) is an entry in **your personal** list of tickets. Every guest has their own list, and everyone's list starts at 1, 2, 3…
- **The hook** the ticket points to is a slot on the hotel's **shared coat rack**. The hook remembers something the ticket doesn't: *which pocket the attendant last reached into.*
- **The coat itself** hangs on the hook. But — and here's the trick — **two different tickets can point to the SAME hook.** Give your friend a photocopy of ticket 3 and you're both pointing at the same hook, sharing the same "which pocket did we last reach into" bookmark. Whereas if your friend checks in *their own identical coat separately*, they get a **different hook** with its **own** bookmark.

That's the entire kernel file-descriptor architecture. Your ticket is an `int`. Your ticket list is the **per-process fd table**. The coat rack is the **system-wide open file table** (and the "which pocket" bookmark is the **file offset**). The coats are **inodes**.

Everything in this document — why `cmd > f 2>&1` works, why two `open()`s of one file don't clobber each other, why fds survive `exec`, why you can recover a deleted log — falls out of that one picture.

---

## Where This Lives in the Linux Stack

```
Hardware (disk, NIC)
  │
  └── KERNEL
       │
       ├── Inode table (the actual files → topic 04)
       │      ▲
       ├── System-wide OPEN FILE TABLE  ◀◀◀ THIS TOPIC LIVES HERE
       │      ▲     (offset + flags + refcount, one entry per open())
       │      │
       ├── Per-process FILE DESCRIPTOR TABLE  ◀◀◀ AND HERE
       │      ▲     (an ARRAY: fd number → pointer into the open file table)
       │      │      lives in the process's task_struct → files_struct
       │      │
       └── System Calls ── open() returns an fd. read/write/close/dup2/fcntl TAKE an fd.
            │              ★ AN FD IS THE HANDLE FOR EVERY I/O SYSCALL IN LINUX. ★
            │
            └── C Library (glibc)
                 │   FILE* (stdio) is a thin wrapper AROUND an int fd
                 │
                 └── Shell (bash)
                      │   `>`, `>>`, `2>&1`, `|` — ALL of these are just fd
                      │   manipulation via dup2() before exec (→ topic 12)
                      │
                      └── Commands & your Node app
                           Every socket, every fs.createReadStream, every DB
                           connection, every HTTP request = ONE FD.
```

**Why it lives exactly there:** the kernel can't hand a userspace program a raw pointer into kernel memory — that would let any process corrupt the kernel. So it hands out an **integer index** instead. The process can only say "do X to number 3", and the kernel does the pointer chase itself, inside the safety of ring 0. **The file descriptor exists to be a safe, opaque, per-process capability handle.**

---

## What Is This?

A **file descriptor** is a small non-negative integer. That's it. `0`, `1`, `2`, `3`, `4`…

It is an **index into an array** — the kernel's per-process file descriptor table. When you `open()` a file, the kernel finds the **lowest available slot** in *your* process's table, points that slot at a new entry in the system-wide open file table, and returns the slot number.

Everything you can do I/O on gets one: regular files, directories, pipes, TCP sockets, Unix sockets, terminals, `epoll` instances, `eventfd`s, `timerfd`s, `inotify` watches, `/dev/null`. **Everything is a file (topic 03), and every open file is an fd.**

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| An fd is a per-process integer index | `/proc/PID/fd` will look like magic instead of the obvious directory listing it is |
| The three-level table structure | You won't understand why `2>&1` interleaves correctly but `> f 2> f` corrupts your log |
| fds are inherited across `fork` and survive `exec` | Your child process will mysteriously hold the port open after the parent dies, and `EADDRINUSE` will haunt your restarts |
| `RLIMIT_NOFILE` | **`EMFILE: too many open files`** — the single most common way a busy Node server falls over in production |
| fd leaks | Your server works for 6 hours, then dies. Every night. Restarting "fixes" it, so nobody finds the bug for a month |
| Deleted-but-open files | `df` says the disk is 100% full, `du` says it's 40% used, you `rm` the logs and **nothing happens** |
| `lsof -i :3000` | You'll waste an afternoon on `EADDRINUSE` instead of solving it in four seconds |

---

## The Physical Reality — The Three-Level Structure

**This diagram is the centrepiece of the entire document.** Learn to draw it from memory and you can derive every behaviour in this doc from first principles.

```
   PROCESS A (PID 8234, node)                       PROCESS B (PID 8299, node)
 ┌───────────────────────────────┐            ┌───────────────────────────────┐
 │  task_struct                  │            │  task_struct                  │
 │    └── files_struct           │            │    └── files_struct           │
 │         └── fd_array[]        │            │         └── fd_array[]        │
 │  ┌────┬──────────────────┐    │            │  ┌────┬──────────────────┐    │
 │  │ fd │  pointer         │    │            │  │ fd │  pointer         │    │
 │  ├────┼──────────────────┤    │            │  ├────┼──────────────────┤    │
 │  │ 0  │ ──────────────┐  │    │            │  │ 0  │ ─────────────┐   │    │
 │  │ 1  │ ────────────┐ │  │    │            │  │ 1  │ ───────────┐ │   │    │
 │  │ 2  │ ────────────┤ │  │    │            │  │ 2  │ ───────────┤ │   │    │
 │  │ 3  │ ──────┐     │ │  │    │            │  │ 3  │ ─────┐     │ │   │    │
 │  │ 4  │ ────┐ │     │ │  │    │            │  └────┴──────┼─────┼─┼───┘    │
 │  └────┴─────┼─┼─────┼─┼──┘    │            └──────────────┼─────┼─┼────────┘
 └─────────────┼─┼─────┼─┼───────┘                           │     │ │
   ┌───────────┘ │     │ │        ┌──────────────────────────┘     │ │
   │  ┌──────────┘     │ └────────┼─────────┐   ┌────────────────────┘ │
   │  │                └──────┐   │         │   │  ┌───────────────────┘
   ▼  ▼                       ▼   ▼         ▼   ▼  ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║           SYSTEM-WIDE OPEN FILE TABLE   (kernel: struct file)                 ║
║           ONE ENTRY PER open() CALL — not per file!                           ║
╠═══════════╤══════════════╤═════════════════════╤═════════════════════════════╣
║ entry     │  OFFSET      │  FLAGS / MODE       │  REFCOUNT │  → inode         ║
╠═══════════╪══════════════╪═════════════════════╪═══════════╪══════════════════╣
║ #101      │  4096        │  O_RDONLY           │     1     │  → inode 1179721 ║  A's fd 3
║ #102      │  0           │  O_WRONLY|O_APPEND  │     2     │  → inode 2621451 ║  A's fd 4
║           │              │                     │  ▲        │                  ║  (shared!)
║ #103      │  8192        │  O_RDONLY           │     1     │  → inode 1179721 ║  B's fd 3
║ #104      │  512         │  O_RDWR             │     4     │  → inode 5 (tty) ║  0,1,2 of both
╚═══════════╧══════════════╧═════════════════════╧═══════════╧══════════════════╝
                                                                    │
                                                                    ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║                     INODE TABLE  (→ TOPIC 04)                                 ║
╠══════════════════════════════════════════════════════════════════════════════╣
║ inode 1179721: size, mode 0644, uid, gid, mtime, LINK COUNT, → data blocks    ║
║ inode 2621451: size, mode 0640, uid, gid, mtime, LINK COUNT, → data blocks    ║
║ inode       5: character device (the terminal)                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
        ★ NOTE: NO FILENAME ANYWHERE. Filenames live in DIRECTORY ENTRIES,
          not in the inode (topic 04). The kernel resolves the path ONCE,
          at open() time, and then works purely with the inode. This is why
          you can delete a file that's open — see "The Deleted-File Trick".
```

### Read that diagram again. Now derive four facts from it:

**Fact 1 — Two separate `open()`s of the same file have INDEPENDENT offsets.**
A's fd 3 → entry #101 (offset 4096). B's fd 3 → entry #103 (offset 8192). **Different open-file entries, same inode.** Both read the same file, neither disturbs the other's read position. This is why two `fs.createReadStream()` calls on one file don't fight.

**Fact 2 — `dup2()` makes two fds SHARE ONE offset.** ★ THE BIG ONE ★
A's fd 4 → entry #102, **refcount 2**. Something else points at #102 too. `dup2(oldfd, newfd)` does **not** create a new open-file entry — it copies the *pointer* and bumps the refcount. Both fds now share one offset.

This is *exactly* how `2>&1` works (→ topic 12):

```
   cmd > /var/log/app.log 2>&1              ← RIGHT
   ─────────────────────────────
   1. open("/var/log/app.log", O_WRONLY|O_CREAT|O_TRUNC) → new entry #200, offset 0
   2. dup2(that_fd, 1)   → fd 1 points at #200
   3. dup2(1, 2)         → fd 2 points at THE SAME #200. refcount = 2.

   ┌────┐                    ┌──────────────────┐
   │ 1  │──────┐             │ #200  offset: 0  │
   │ 2  │──────┴────────────▶│ refcount: 2      │──▶ inode(app.log)
   └────┘                    └──────────────────┘
   ONE shared offset. stdout writes 100 bytes → offset becomes 100.
   stderr writes next → starts at 100. THEY INTERLEAVE PERFECTLY. ✓
```

```
   cmd > /var/log/app.log 2> /var/log/app.log     ← WRONG. LOOKS IDENTICAL. ISN'T.
   ─────────────────────────────────────────
   TWO separate open() calls → TWO separate entries → TWO INDEPENDENT OFFSETS.

   ┌────┐   ┌──────────────────┐
   │ 1  │──▶│ #200  offset: 0  │──┐
   └────┘   └──────────────────┘  │
   ┌────┐   ┌──────────────────┐  ├──▶ inode(app.log)   SAME FILE
   │ 2  │──▶│ #201  offset: 0  │──┘
   └────┘   └──────────────────┘
   stdout writes 100 bytes at offset 0 → its offset is now 100.
   stderr writes 40 bytes AT OFFSET 0  → IT OVERWRITES THE FIRST 40 BYTES. ✗
   Your log is CORRUPTED. Silently. This is why `2>&1` exists.
```

**Fact 3 — `fork()` copies the fd table, but NOT the open file entries.**
The child gets its own fd table whose pointers aim at **the parent's open file entries**, with every refcount bumped by 1. Parent and child therefore **share offsets**. This is exactly what makes a shell pipeline work, and it's why a child process can keep a listening socket alive after the parent exits — the refcount never dropped to zero.

**Fact 4 — fds SURVIVE `execve()` unless CLOEXEC is set.** (→ topic 16)
`execve()` obliterates the process's memory — code, heap, stack, all replaced by the new binary. But the `files_struct` is **preserved**. That is the *only* reason the shell's redirection scheme works at all:

```
bash:  fork()                      ← child is a copy of bash
       [in the child]
       open("/var/log/app.log")    ← set up fd 1 and fd 2 with dup2
       execve("/usr/bin/node", …)  ← child becomes node. MEMORY WIPED.
                                     ★ BUT FDS 0,1,2 SURVIVE ★
       node starts, writes to fd 1 — and lands in app.log,
       having never been told the file exists.
```

The opt-out is the **`O_CLOEXEC`** flag (or `fcntl(fd, F_SETFD, FD_CLOEXEC)`), which tells the kernel: *close this fd automatically at exec.* Well-behaved libraries set it on every fd they open — otherwise, every time your Node app spawns a child process, that child inherits your **database sockets** and your **listening port**. That's a security hole and a resource leak. Node sets `O_CLOEXEC` by default on fds it opens; be suspicious of anything that doesn't.

---

## How It Works — Step by Step

```
COMMAND: node -e 'const fs=require("fs"); const fd=fs.openSync("/tmp/a.txt","r")'

USERSPACE:   fs.openSync → libuv → glibc open() wrapper
SYSCALL:     openat(AT_FDCWD, "/tmp/a.txt", O_RDONLY|O_CLOEXEC) = 20

KERNEL DOES:
  1. Path resolution: walk "/" → "tmp" → "a.txt" through directory entries,
     checking the execute (search) bit on each directory (→ topic 05).
     Result: inode number 1179721.
  2. Permission check: does this process's UID/GID allow O_RDONLY on inode 1179721?
     If no → return -EACCES. (This is where Node's EACCES comes from — topic 01.)
  3. Allocate a `struct file` in the system-wide open file table:
        offset = 0,  flags = O_RDONLY,  refcount = 1,  → inode 1179721
  4. Scan THIS PROCESS's fd table for the LOWEST FREE INDEX.
     0,1,2 taken. 3..19 taken (Node opens epoll fds, eventfds, the module cache…).
     20 is free. → fd_array[20] = &that_struct_file
  5. Return the integer 20 to userspace.

OUTPUT:      the integer 20. That's all the process ever gets.

NOW:  read(20, buf, 4096)
      → kernel: fd_array[20] → struct file → offset 0 → inode → data blocks
      → copies 4096 bytes into buf
      → ★ ADVANCES struct file->offset TO 4096 ★  (the offset lives in the
        OPEN FILE ENTRY, not in the fd, and not in the inode)
```

**Prove every step of that:**
```bash
strace -e trace=openat,read,close node -e 'require("fs").readFileSync("/etc/hostname")' 2>&1 | tail -6
# openat(AT_FDCWD, "/etc/hostname", O_RDONLY|O_CLOEXEC) = 20
# read(20, "myserver\n", 65536)                          = 9
# read(20, "", 65536)                                    = 0     ← EOF
# close(20)                                              = 0
```

---

## 0, 1, 2 — "Standard" Only by CONVENTION

```
   fd 0  = stdin     ← where the program READS input from
   fd 1  = stdout    ← where the program WRITES normal output
   fd 2  = stderr    ← where the program WRITES errors
```

**The kernel does not know or care about any of this.** There is no code in Linux that says "fd 1 is special". The kernel's rule is exactly one line long:

> **`open()` returns the LOWEST-NUMBERED unused fd in this process's table.**

0, 1 and 2 are just *the first three numbers*, and the shell always sets them up (pointing at your terminal) before `exec`ing your program. Every program then *agrees by convention* to treat them as stdin/stdout/stderr. That's the whole story.

**Prove it — close fd 0, then open a file, and watch it become fd 0:**

```c
// /tmp/lowest.c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
int main(void) {
    close(0);                                  // free up slot 0
    int fd = open("/etc/hostname", O_RDONLY);  // kernel: lowest free slot is... 0
    dprintf(2, "open() returned fd = %d\n", fd);   // write to fd 2 (stderr)
    return 0;
}
```
```bash
gcc /tmp/lowest.c -o /tmp/lowest && /tmp/lowest
# open() returned fd = 0
#                      ▲ A REGULAR FILE IS NOW YOUR "STANDARD INPUT".
#                        The kernel did not object. It has no opinion.
```

And you can do the same thing from the shell — `exec 3< file` explicitly asks for fd 3:

```bash
exec 3< /etc/hostname     # open /etc/hostname for reading as fd 3 IN THE SHELL ITSELF
read -u 3 line            # read one line FROM fd 3
echo "$line"              # myserver
exec 3<&-                 # close fd 3
```

---

## `/proc/PID/fd/` — Every Open File, As Symlinks

The kernel exposes each process's fd table as a **directory of symlinks**. This is the most useful debugging surface in Linux.

```bash
ls -l /proc/8234/fd
```
```
total 0
lrwx------ 1 node node 64 Jul 12 02:14 0 -> /dev/null
lrwx------ 1 node node 64 Jul 12 02:14 1 -> 'pipe:[482913]'
lrwx------ 1 node node 64 Jul 12 02:14 2 -> 'pipe:[482914]'
lr-x------ 1 node node 64 Jul 12 02:14 3 -> /dev/urandom
lrwx------ 1 node node 64 Jul 12 02:14 4 -> 'anon_inode:[eventpoll]'
lrwx------ 1 node node 64 Jul 12 02:14 5 -> 'anon_inode:[eventfd]'
lrwx------ 1 node node 64 Jul 12 02:14 6 -> 'socket:[482920]'
l-wx------ 1 node node 64 Jul 12 02:14 7 -> /srv/app/logs/access.log
lrwx------ 1 node node 64 Jul 12 02:14 8 -> 'socket:[483102]'
lrwx------ 1 node node 64 Jul 12 02:14 9 -> 'socket:[483118]'
l-wx------ 1 node node 64 Jul 12 02:19 12 -> '/srv/app/logs/old.log (deleted)'
```

Line by line — **this is a complete picture of what a Node process is doing:**

| fd | Target | What it actually is |
|---|---|---|
| `0` | `/dev/null` | stdin — a daemon has no terminal, so systemd wired stdin to the bit bucket |
| `1`, `2` | `pipe:[482913]` | stdout/stderr are **pipes**, not the terminal — because systemd/pm2 is on the other end capturing them into the journal. The number is the pipe's *inode* |
| `3` | `/dev/urandom` | Node keeps this open for crypto |
| `4` | `anon_inode:[eventpoll]` | **The epoll instance — this IS the event loop.** libuv registered every socket here |
| `5` | `anon_inode:[eventfd]` | How the libuv threadpool wakes the main loop |
| `6` | `socket:[482920]` | **The listening socket on :3000** |
| `7` | `/srv/app/logs/access.log` | `l-wx` = write-only. An `fs.createWriteStream` |
| `8`, `9` | `socket:[483102]` | Live client connections. **Each concurrent HTTP request is one of these.** |
| `12` | `... (deleted)` | ⚠ A file that was `rm`'d **while this process still had it open.** Read on. |

**The symlink target of a socket is `socket:[INODE]`** — cross-reference that inode against `ss -tanp` or `/proc/net/tcp` to find the actual IP:port pair (→ topic 28).

### The DELETED-FILE TRICK — recovering an `rm`'d log

Look at fd 12 again: `-> '/srv/app/logs/old.log (deleted)'`.

**Remember from topic 04 — Inodes:** `rm` does not delete data. `rm` calls `unlink()`, which removes the **directory entry** (the name → inode mapping) and decrements the inode's **link count**. The kernel only frees the data blocks when **link count == 0 AND no process has it open.**

An open fd is a reference. So:

```
   BEFORE rm:                          AFTER rm:
   ┌──────────────┐                    ┌──────────────┐
   │ dir entry:   │                    │  (gone)      │
   │ "old.log"    │──┐                 │              │
   └──────────────┘  │                 └──────────────┘
                     ▼                                   the NAME is gone…
              ┌─────────────┐                     ┌─────────────┐
              │ inode 2621  │◀── fd 12            │ inode 2621  │◀── fd 12
              │ links: 1    │    (open file       │ links: 0    │    STILL OPEN
              │ 4.2 GB data │     entry holds     │ 4.2 GB data │    …so the DATA
              └─────────────┘     a reference)    └─────────────┘    IS STILL THERE
                                                     ★ AND STILL EATS 4.2 GB OF DISK ★
```

The data is fully intact and fully readable — through the fd. And `/proc/PID/fd/12` **is** that fd:

```bash
# THE SAVE: someone rm'd the log the app is still writing to.
ls -l /proc/8234/fd/12
# l-wx------ 1 node node 64 Jul 12 02:19 12 -> '/srv/app/logs/old.log (deleted)'

cp /proc/8234/fd/12 /srv/app/logs/recovered.log     # ★ FULL RECOVERY ★
ls -lh /srv/app/logs/recovered.log
# -rw-r--r-- 1 root root 4.2G Jul 12 02:20 recovered.log
```

You just recovered a "deleted" 4.2 GB file. **This works only while the process is still alive.** The moment it exits or closes fd 12, the refcount hits zero, the link count is already zero, and the kernel frees the blocks — instantly and irrecoverably. **Do not restart the app first.**

---

## `lsof` — LiSt Open Files

`lsof` walks `/proc/*/fd/` for every process and joins it against the inode/socket tables. It answers questions nothing else can.

```bash
lsof -i :3000 -P -n
#    │  │     │  │
#    │  │     │  └── -n = do NOT resolve IPs to hostnames (fast, and no DNS hang)
#    │  │     └── -P = do NOT resolve port numbers to names (show 3000, not "ppp")
#    │  └── :3000 = any INTERNET socket on port 3000
#    └── -i = Internet (network) files
```
```
COMMAND  PID USER  FD  TYPE DEVICE SIZE/OFF NODE NAME
node    8234 node   6u IPv4 482920      0t0  TCP *:3000 (LISTEN)
node    8234 node   8u IPv4 483102      0t0  TCP 10.0.1.5:3000->10.0.1.9:51204 (ESTABLISHED)
        │         │                                                              │
        │         └── the FD NUMBER. u = open read/write.                        │
        └── the PID holding the port ─────────────────────────────────────────────┘
```

**This is the #1 use of `lsof` in a backend developer's life.** `Error: listen EADDRINUSE :::3000` → `lsof -i :3000 -P -n` → there's the PID → `kill` it. Four seconds, not an afternoon.

### The `lsof` commands worth memorising

| Command | Answers |
|---|---|
| `lsof -i :3000 -P -n` | **What is holding port 3000?** (`EADDRINUSE`) |
| `lsof -i -P -n` | Every network connection on the box, with owning PID |
| `lsof -p 8234` | Everything PID 8234 has open |
| `lsof /var/log/app.log` | **Who is holding this file open?** |
| `lsof /mnt/data` | **Who is stopping me unmounting this?** (`device is busy`) |
| `lsof +D /var/log` | Everything open under this directory, **recursively** (slow — it stats every file) |
| `lsof -u deploy` | Everything the `deploy` user has open |
| `lsof +L1` | ★ **Deleted-but-still-open files** — the "disk is full but I deleted it" incident |
| `lsof -p 8234 -a -i` | AND the filters: only PID 8234's *network* fds (`-a` = AND, default is OR) |

### `lsof +L1` — the `du`/`df` disagreement

```bash
df -h /
# Filesystem   Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1  40G   40G     0 100% /       ◀ FULL

du -sh /                    # (takes a while)
# 16G   /                   ◀ …but only 16G of FILES exist?!  24 GB UNACCOUNTED FOR.
```

`du` walks **directory entries**. `df` asks the **filesystem** how many blocks are allocated. A deleted-but-open file has **no directory entry** (so `du` cannot see it) but its **blocks are still allocated** (so `df` counts it). The gap is your answer.

```bash
sudo lsof +L1
#     │
#     └── +L1 = show files whose LINK COUNT is less than 1, i.e. UNLINKED but still open
```
```
COMMAND  PID USER FD  TYPE DEVICE  SIZE/OFF NLINK    NODE NAME
node    8234 node 12w  REG  259,1 24696061952     0 2621451 /srv/app/logs/app.log (deleted)
                                            ▲       ▲
                                   24 GB ───┘       └── NLINK = 0. The name is gone.
                                                         The BLOCKS ARE NOT.
```

**The bug:** someone (or logrotate, misconfigured without `copytruncate`) ran `rm app.log`. Node's write stream still holds fd 12. Node keeps appending, at ever-increasing offsets, into a file **with no name**, forever. The disk fills. You `rm` more things. **Nothing frees up.** Because the space was never held by a name in the first place.

```bash
# ✗ WRONG — the file has no name. There is nothing left to rm.
rm /srv/app/logs/app.log        # "No such file or directory"

# ✓ RIGHT (instant, zero downtime) — truncate THROUGH the fd. The kernel frees
#   the blocks immediately; the process's write stream carries on unharmed.
sudo truncate -s 0 /proc/8234/fd/12
df -h /        # 24 GB back. Immediately.

# ✓ ALSO RIGHT (but causes downtime) — restart the process. On exit, the kernel
#   closes fd 12, the refcount drops to 0, link count is already 0 → blocks freed.
sudo systemctl restart api
```

**And for the general "the live log file is huge" case, the rule is the same:**
```bash
# ✗ WRONG: rm access.log  → creates EXACTLY the invisible-space bug above.
# ✓ RIGHT: truncate the file IN PLACE. The inode survives, the fd stays valid,
#          the app keeps writing (at offset 0 again, or its old offset if O_APPEND).
truncate -s 0 /srv/app/logs/access.log
# or:      : > /srv/app/logs/access.log
```
**Never `rm` a file a running process is writing to.** (This is exactly what `logrotate`'s `copytruncate` option exists for → topic 23.)

---

## ULIMITS and `EMFILE` — The Big Production Payoff

Every fd costs the kernel a `struct file` and a slot in your fd table. Unbounded, one runaway process could exhaust kernel memory. So the kernel enforces **`RLIMIT_NOFILE`**: the maximum fd number a process may be given, per process.

### Soft vs hard limits

```
   ┌───────────────────────── RLIMIT_NOFILE ────────────────────────┐
   │                                                                 │
   │   SOFT limit (1024)             HARD limit (1048576)            │
   │   ────────────────              ───────────────────             │
   │   The limit ACTUALLY            The ceiling on the soft limit.  │
   │   ENFORCED right now.           Raising THIS requires root      │
   │   open() returns EMFILE         (CAP_SYS_RESOURCE).             │
   │   the moment you exceed it.                                     │
   │                                                                 │
   │   ANY process may raise its own soft limit up to the hard limit │
   │   — no privileges needed. It may also LOWER the hard limit,     │
   │   and that is IRREVERSIBLE for that process.                    │
   └─────────────────────────────────────────────────────────────────┘
```

```bash
ulimit -n            # 1024      ← the SOFT limit (what bites you)
ulimit -Hn           # 1048576   ← the HARD limit (the ceiling)
ulimit -a            # everything
```
```
core file size          (blocks, -c) 0           ← ⚠ 0 = NO CORE DUMPS ARE WRITTEN
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 31163
max locked memory       (kbytes, -l) 8192
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024        ★★★ THE ONE THAT KILLS NODE APPS
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 31163       ← -u: hit this and fork() gives EAGAIN
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

### Why `EMFILE: too many open files` is THE canonical Node.js crash

**In Node, virtually everything is a file descriptor:**

```
   Your Node process at 1000 concurrent users
   ┌──────────────────────────────────────────────────────────────┐
   │  1 listening socket on :3000                        =    1 fd│
   │  1000 inbound HTTP keep-alive connections           = 1000 fd│
   │  a 20-connection Postgres pool                      =   20 fd│
   │  a 10-connection Redis pool                         =   10 fd│
   │  outbound axios/fetch calls to a 3rd-party API      =   ~50 fd│
   │  fs.createReadStream for each file upload            =   ~30 fd│
   │  the access log write stream                        =    1 fd│
   │  epoll + eventfd + urandom + stdio + module cache   =   ~15 fd│
   │                                                       ────────│
   │                                                TOTAL ≈ 1127 fd│
   └──────────────────────────────────────────────────────────────┘
                                             DEFAULT SOFT LIMIT = 1024
                                             ▶ open() → EMFILE
                                             ▶ Node throws:
                                               Error: EMFILE: too many open files
```

At 1024 open fds, **the next `accept()` fails.** Your server stops accepting connections. It doesn't crash cleanly — it just starts refusing everything, while `top` shows a perfectly calm process using 4% CPU.

**The two distinct causes — and you MUST tell them apart:**

| | **Legitimate load** | **An FD LEAK** |
|---|---|---|
| Shape | fd count rises with traffic, **plateaus**, falls at night | fd count only ever **rises**, never falls — even when idle |
| Fix | **Raise the limit.** 1024 is absurdly low for a server | **Raising the limit just delays the crash by a few hours.** Find the bug |
| Typical bug | — | Not closing `fs.createReadStream` on the error path; creating a **new DB connection per request** instead of pooling; an HTTP agent with `keepAlive` and no `maxSockets`; forgetting `res.destroy()` on abort |

### Diagnosing — the exact commands

```bash
PID=$(pgrep -f 'node /srv/app/server.js')

# 1. How many fds is it holding RIGHT NOW?
ls /proc/$PID/fd | wc -l
# 1021                                    ← 3 away from the limit. About to die.

# 2. What IS the limit for THAT process? (NOT `ulimit -n` in your shell —
#    that shows YOUR shell's limit, which may be totally different!)
cat /proc/$PID/limits | grep -i 'open files'
# Max open files            1024                 1048576              files
#                           ▲soft                ▲hard

# 3. IS IT A LEAK? Watch the count over time. If it climbs while traffic is FLAT
#    (or, damningly, while traffic is ZERO) → it is a leak.
watch -n5 "ls /proc/$PID/fd | wc -l"
# or log it:
while true; do echo "$(date +%T) $(ls /proc/$PID/fd | wc -l)"; sleep 60; done

# 4. WHAT is leaking? Group the open fds by TYPE. This names the bug.
lsof -p $PID | awk '{print $5}' | sort | uniq -c | sort -rn
#     982 IPv4      ← 982 sockets. It's a network leak.
#      21 REG       ← regular files
#       6 unknown
#       4 CHR
#       2 FIFO
#       1 DIR

# 5. Narrow it to the actual peer — WHICH sockets?
lsof -p $PID -a -i -P -n | awk '{print $9}' | sort | uniq -c | sort -rn | head
#     961 10.0.1.5:44xxx->10.0.4.11:5432 (ESTABLISHED)
#                                  ▲▲▲▲ port 5432 = POSTGRES.
#     ★ 961 open Postgres connections. YOU ARE CREATING A NEW `new Client()`
#       PER REQUEST AND NEVER CALLING .end(). THERE IS YOUR BUG.
```

**In under a minute you went from "the server randomly stops responding" to "line 47 opens a DB connection per request."** That is what this chapter is for.

### Fixing it PROPERLY — the full chain

**`ulimit -n 65535` in your shell is NOT a fix.** It affects only that shell and processes it forks. It doesn't survive logout. It doesn't apply to systemd services (which are *not* forked from your shell). And it cannot exceed the **hard** limit.

```bash
# TIER 0 — the shell (for testing ONLY; ephemeral, and capped by the HARD limit)
ulimit -n 65535
ulimit -Hn 65535     # requires root; raises the ceiling itself

# TIER 1 — PAM logins (SSH sessions, `su`, login shells).
sudo tee -a /etc/security/limits.conf <<'EOF'
deploy  soft  nofile  65535
deploy  hard  nofile  65535
*       soft  nofile  65535
*       hard  nofile  65535
EOF
#  ⚠⚠ THIS DOES NOT APPLY TO SYSTEMD SERVICES. ⚠⚠
#  limits.conf is enforced by pam_limits.so, which runs at LOGIN.
#  systemd starts services from PID 1 — no login, no PAM, no limits.conf.
#  This is the #1 reason people "raise the limit" and STILL get EMFILE.

# TIER 2 — ★ THE REAL FIX FOR A SERVICE ★  (→ topic 22)
sudo systemctl edit api        # creates a drop-in override
```
```ini
[Service]
LimitNOFILE=65535
# systemd-wide default, if you prefer:
#   /etc/systemd/system.conf → DefaultLimitNOFILE=65535
```
```bash
sudo systemctl daemon-reload && sudo systemctl restart api
# VERIFY — never trust, always check:
cat /proc/$(pgrep -f server.js)/limits | grep -i 'open files'
# Max open files            65535                65535                files   ✓

# TIER 3 — the SYSTEM-WIDE ceiling (all processes combined).
cat /proc/sys/fs/file-max          # 9223372036854775807 on modern kernels — fine
cat /proc/sys/fs/file-nr           # 5824  0  9223372036854775807
#                                    ▲allocated ▲free ▲max
sudo sysctl -w fs.file-max=2097152
echo 'fs.file-max = 2097152' | sudo tee /etc/sysctl.d/99-nofile.conf

# TIER 4 — Docker / Kubernetes
docker run --ulimit nofile=65535:65535 myapp
# docker-compose:
#   services:
#     api:
#       ulimits:
#         nofile: { soft: 65535, hard: 65535 }
```

---

## Exact Syntax Breakdown

```
lsof -i :3000 -P -n -sTCP:LISTEN
│    │  │     │  │   │
│    │  │     │  │   └── -sTCP:LISTEN = only sockets in the LISTEN state
│    │  │     │  └── -n = don't reverse-DNS the IPs (fast; avoids a DNS hang)
│    │  │     └── -P = don't translate port numbers to service names
│    │  └── :3000 = the port selector. Also: -i TCP@10.0.1.5:3000, -i 4 (IPv4 only)
│    └── -i = Internet sockets. (Default: ALL open files of ALL processes.)
└── lsof = LiSt Open Files. Walks /proc/*/fd for every process on the box.

⚠ Filters are ORed by default. `-a` makes them AND:
    lsof -u deploy -i          →  files owned by deploy  OR  any network socket
    lsof -u deploy -a -i       →  network sockets owned by deploy   ← usually what you want
```

```
lsof +L1
│    ││
│    │└── 1 = link count threshold. "+L1" = show files with NLINK < 1.
│    └── +L = display the link count column and filter on it
└── The ONE command that finds "disk full but du says otherwise".
```

```
ulimit -n 65535
│      │  │
│      │  └── the new value. `unlimited` is accepted for some resources, not for -n.
│      └── -n = max number of open file descriptors (RLIMIT_NOFILE)
│           -a = show all    -u = max user processes    -c = core file size
│           -S = act on the SOFT limit (default)
│           -H = act on the HARD limit (lowering is irreversible; raising needs root)
└── ulimit — a BASH BUILTIN, not a program. It calls setrlimit(2) on the SHELL,
    and every process the shell subsequently forks inherits it.
    ★ Therefore it CANNOT affect an already-running process, and it CANNOT
      affect a systemd service (which is not forked from your shell).
```

```
cat /proc/8234/limits
│   │     │    │
│   │     │    └── every rlimit for THIS process, soft and hard, as it ACTUALLY IS
│   │     └── the PID
│   └── procfs
└── ★ THE ONLY WAY to know a RUNNING process's real limit. `ulimit -n` in your
    shell tells you about YOUR SHELL. It says nothing about the daemon.
```

```
ls -l /proc/8234/fd/ | wc -l
│  │  │              │
│  │  │              └── count the lines (subtract 1 for the "total" line)
│  │  └── a directory of SYMLINKS, one per open fd
│  └── -l = long format, so you can SEE the symlink targets
└── Faster, no-lsof-needed fd count. Works in a bare Alpine container.
```

---

## Example 1 — Basic

```bash
# 1. Start a long-lived process we can poke at, holding a file open.
sleep 300 &
PID=$!

# 2. Look at its file descriptors. sleep is boring: just 0, 1, 2.
ls -l /proc/$PID/fd
# lrwx------ 1 you you 64 Jul 12 14:02 0 -> /dev/pts/0     ← stdin  = your terminal
# lrwx------ 1 you you 64 Jul 12 14:02 1 -> /dev/pts/0     ← stdout = your terminal
# lrwx------ 1 you you 64 Jul 12 14:02 2 -> /dev/pts/0     ← stderr = your terminal
#                                              ▲ ALL THREE POINT AT THE SAME TTY.

# 3. Now redirect stdout and watch fd 1 CHANGE TARGET:
kill $PID
sleep 300 > /tmp/out.txt &
PID=$!
ls -l /proc/$PID/fd/1
# l-wx------ 1 you you 64 Jul 12 14:03 1 -> /tmp/out.txt
#  ▲ l-wx = write-only. The shell did open() + dup2(fd,1) BEFORE exec.
#    `sleep` has no idea. It just writes to fd 1, like always.
kill $PID

# 4. Open a file as fd 3 in YOUR OWN SHELL and read through it:
exec 3< /etc/hostname
ls -l /proc/$$/fd/3          # $$ = this shell's PID
# lr-x------ 1 you you 64 Jul 12 14:05 3 -> /etc/hostname
read -u 3 line && echo "read from fd 3: $line"
exec 3<&-                    # close fd 3

# 5. Watch the kernel hand out the LOWEST FREE NUMBER:
exec 3< /etc/hostname
exec 4< /etc/hosts
exec 3<&-                    # free up slot 3
exec 5< /etc/passwd          # ← what fd will the NEXT open get?
ls /proc/$$/fd
# 0  1  2  4  5      ← hmm, 5? Because we ASKED for 5 explicitly with `exec 5<`.
#                      bash's `exec N<` requests a SPECIFIC number via dup2.
#                      A bare open() in C would have taken 3.
exec 4<&- 5<&-

# 6. Your shell's fd limit:
ulimit -n        # 1024 (typical)
ulimit -Hn       # 1048576 — the ceiling you're allowed to raise the soft limit to
```

---

## Example 2 — Production Scenario

**03:40 AM.** Alert: `api-prod-01 — 5xx rate 100%`. The process is running. CPU is 3%. Memory is fine. Nothing in `dmesg`.

```bash
$ sudo journalctl -u api -n 20 --no-pager
Jul 12 03:31:02 api-prod-01 node[8234]: Error: EMFILE: too many open files, accept
Jul 12 03:31:02 api-prod-01 node[8234]:     at Server._listen2 (node:net:1421:21)
Jul 12 03:31:03 api-prod-01 node[8234]: Error: EMFILE: too many open files, accept
Jul 12 03:31:04 api-prod-01 node[8234]: Error: EMFILE: too many open files, open '/srv/app/uploads/tmp-9f2.dat'
```

**`EMFILE`.** The process is alive but the kernel is refusing to give it any more fds. Every `accept()` fails → every connection is refused → 100% 5xx. The process never crashed, so systemd never restarted it, so nothing self-healed.

```bash
$ PID=$(pgrep -f 'node /srv/app/server.js')
$ ls /proc/$PID/fd | wc -l                          # 1024
$ cat /proc/$PID/limits | grep -i 'open files'
Max open files            1024                 1048576              files   # SOFT 1024, HARD 1M
```

**At its limit.** Load or leak? Traffic is 12 req/s — 1024 fds at that rate is absurd, so it's a **leak**. What kind?

```bash
$ sudo lsof -p $PID | awk '{print $5}' | sort | uniq -c | sort -rn | head -3
    981 IPv4          # 981 sockets
     19 REG
$ sudo lsof -p $PID -a -i -P -n | awk '{print $9}' | sed 's/.*->//' | sort | uniq -c | sort -rn | head -2
    958 10.0.4.11:5432 (ESTABLISHED)     # ★ 958 connections to Postgres. Pool max is 20.
     18 10.0.4.20:6379 (ESTABLISHED)     #   → something opens `new Client()` per request, never .end()s
$ grep -rn "new Client" /srv/app/src | head -1
src/routes/report.js:14:  const client = new Client({ connectionString: process.env.DB_URL });
#   /report is polled by a dashboard every 30s; 1024 ÷ 2/min ≈ 8h to death — hence the nightly crash. ★
```

**Mitigate** (restore service in 10s): `sudo systemctl restart api` — on exit the kernel closes all 1024 fds. It will die again in ~8h, so:

**Fix 1 — raise the limit (headroom, NOT a fix):** `sudo systemctl edit api` → `[Service]\nLimitNOFILE=65535`, `daemon-reload`, restart; verify with `cat /proc/PID/limits`. ⚠ We did **not** touch `/etc/security/limits.conf` — it does nothing for systemd services (no PAM). That trap eats an afternoon.

**Fix 2 — the bug:** a module-level `const pool = new Pool({..., max:20})` reused per request (`pool.query(...)` in a `try/catch`), instead of `new Client()` per request.

**Fix 3 — prevention:** export an fd gauge — `fs.readdirSync('/proc/'+process.pid+'/fd').length` divided by the limit parsed from `/proc/PID/limits` — and alert at 0.8.

**Page to root cause: ~4 minutes**, using only `journalctl`, `/proc/PID/fd`, `/proc/PID/limits`, and `lsof`.

---

## Common Mistakes

### Mistake 1: `ulimit -n 65535` in the shell, then wondering why the service still gets EMFILE

**Wrong:**
```bash
ssh prod
sudo ulimit -n 65535      # (this doesn't even work — ulimit is a shell builtin)
# "fixed it" ... EMFILE returns that night
```
**Right:** `LimitNOFILE=65535` in the systemd unit.

**Root cause:** `ulimit` calls `setrlimit(2)` **on the calling shell**. Limits are inherited only through `fork()`. Your systemd service was forked by **PID 1**, hours ago, and never saw your shell. Editing `/etc/security/limits.conf` fails for the same reason: it's enforced by `pam_limits.so`, which runs **at login** — and systemd doesn't log in.

**Diagnose:** `cat /proc/$(pgrep -f server.js)/limits` — always check the **running process**, never your shell.
**Prevent:** put `LimitNOFILE=` in the unit file (or `DefaultLimitNOFILE=` in `/etc/systemd/system.conf`) and verify with `/proc/PID/limits` after every deploy.

---

### Mistake 2: `rm`-ing a log file that a running process has open

**Wrong:** `rm /srv/app/logs/app.log` to free disk space while Node is running.
**Right:** `truncate -s 0 /srv/app/logs/app.log` (or `: > file`).

**Root cause (topic 04 + this doc):** `rm` → `unlink()` → removes the **directory entry** and decrements the inode's **link count** to 0. But the process's open file entry still **references the inode**, so the kernel cannot free the data blocks. The file now has **no name** (so `du` and `ls` can't see it) but **all its blocks** (so `df` still counts them). Node happily keeps writing to it forever. **Disk space is gone and there is nothing left to delete.**

**Diagnose:**
```bash
df -h /                              # 100% full
du -sh / 2>/dev/null                 # …but only 16G of files? 24G missing.
sudo lsof +L1                        # THERE IT IS: NLINK=0, 24GB, (deleted)
```
**Fix:** `sudo truncate -s 0 /proc/PID/fd/N` — frees the blocks instantly, no restart. Or restart the process.
**Prevent:** configure `logrotate` properly (`copytruncate`, or send `SIGHUP`/`USR1` so the app reopens its log) → topic 23.

---

### Mistake 3: `cmd > file 2> file` instead of `cmd > file 2>&1`

**Wrong:**
```bash
node server.js > /var/log/app.log 2> /var/log/app.log     # ✗ CORRUPTS THE LOG
```
**Right:**
```bash
node server.js > /var/log/app.log 2>&1                    # ✓
```
**Root cause — read it off the three-level diagram:** the wrong form calls `open()` **twice** → **two** open-file entries → **two independent offsets** → both start at 0 → stdout writes 100 bytes and its offset becomes 100; stderr then writes 40 bytes **at offset 0**, silently overwriting the first 40 bytes of stdout's output. The right form calls `dup2(1, 2)`, so fd 2 points at **the same open-file entry** as fd 1 → **one shared offset** → writes append cleanly and interleave in true chronological order.

**Diagnose:** you'll see mangled, half-overwritten lines in your log and assume it's a "logging library bug." It isn't.
**Prevent:** memorise `2>&1` (and that the **order matters**: `2>&1 > file` redirects stderr to the *old* stdout — the terminal — and *then* moves stdout to the file. Not what you want. → topic 12).

---

### Mistake 4: Assuming "raise the limit" fixes an EMFILE

**Wrong:** EMFILE at 1024 → set 65535 → ship it → forget it.
**Right:** First determine whether the fd count **plateaus** (real load → raising is correct) or **climbs forever** (a leak → raising just moves the crash from 8 hours to 3 weeks, which is *worse*, because now it fires during a holiday).

**Root cause:** fds are only released by an explicit `close()` (or process exit). A stream that is never destroyed, a DB client that is never `.end()`ed, an HTTP agent socket that is never released — the fd is held **for the lifetime of the process.** There is no GC for kernel resources. V8's garbage collector will happily collect the *JavaScript object*, and the underlying fd will **still be open**.

**Diagnose:**
```bash
while true; do echo "$(date +%T) $(ls /proc/$PID/fd | wc -l)"; sleep 60; done
# Rising monotonically while traffic is flat → LEAK. Not capacity.
lsof -p $PID | awk '{print $5}' | sort | uniq -c | sort -rn   # WHAT is leaking
```
**Prevent:** always `try/finally { stream.destroy() }`; always use a connection **pool**; export an fd-count metric and alert at 80% of `RLIMIT_NOFILE`.

---

### Mistake 5: "The socket is closed, so the port is free"

**Wrong:** You `kill`ed the Node process but still get `EADDRINUSE` on restart.
**Right:** A **child process inherited the listening socket's fd across `fork`+`exec`.** The parent died; the child is still holding the open-file entry; its **refcount never hit zero**; the socket is still bound to :3000.

**Root cause:** `fork()` duplicates the fd table. `execve()` preserves it (unless `O_CLOEXEC` is set). If your Node app `spawn`ed a child (an ffmpeg job, a shell script, a `npm run` wrapper) **before** setting `FD_CLOEXEC` on the listening socket, that child now owns a reference to your port.

**Diagnose:**
```bash
lsof -i :3000 -P -n
# COMMAND  PID USER FD  TYPE  DEVICE NODE NAME
# ffmpeg  9102 node 6u  IPv4  482920  TCP *:3000 (LISTEN)     ◀◀◀ THE ORPHANED CHILD
```
**Fix:** kill the child (`kill 9102`). **Prevent:** ensure every fd is opened with `O_CLOEXEC` (Node does this by default for its own fds); prefer `{ stdio: ['ignore','pipe','pipe'] }` in `child_process.spawn`; use `detached: false`; and in systemd, `KillMode=control-group` (the default) so the whole cgroup dies with the service, orphaned children included.

---

## Hands-On Proof

```bash
# PROVE IT: an fd is the lowest free integer; two open()s have INDEPENDENT offsets;
#           dup() SHARES the offset (the mechanism behind 2>&1)
python3 - <<'EOF'
import os
print(os.open("/etc/hostname", os.O_RDONLY))      # 3
print(os.open("/etc/hosts",    os.O_RDONLY))      # 4
os.close(3); print(os.open("/etc/passwd", os.O_RDONLY))   # 3 again — lowest free
a = os.open("/etc/hostname", os.O_RDONLY); b = os.open("/etc/hostname", os.O_RDONLY)
os.read(a, 3)
print(os.lseek(a,0,os.SEEK_CUR), os.lseek(b,0,os.SEEK_CUR))   # 3 0  → SEPARATE entries
c = os.dup(a); print(os.lseek(a,0,os.SEEK_CUR), os.lseek(c,0,os.SEEK_CUR))  # 3 3 → SHARED
EOF

# PROVE IT: the shell does dup2() before exec (this IS 2>&1)
strace -f -e trace=openat,dup2 bash -c 'echo hi > /tmp/x 2>&1' 2>&1 | grep -E '/tmp/x|dup2'
# openat(...,"/tmp/x",O_WRONLY|O_CREAT|O_TRUNC)=3 ; dup2(3,1)=1 ; dup2(1,2)=2  ← 2 shares 1's entry

# PROVE IT: fds survive exec
exec 3> /tmp/inh.txt; bash -c 'echo "child wrote via inherited fd" >&3'; cat /tmp/inh.txt; exec 3>&-

# PROVE IT: /proc/PID/fd is symlinks
sleep 60 &  ;  ls -l /proc/$!/fd  ;  kill %1

# PROVE IT: a deleted-but-open file still occupies disk AND is recoverable
python3 -c "import os,time; f=open('/tmp/victim.log','w'); f.write('X'*50_000_000); f.flush(); print(os.getpid()); time.sleep(300)" &
sleep 1; PID=$!
df -h /tmp | tail -1; rm /tmp/victim.log; df -h /tmp | tail -1   # ★ space NOT reclaimed
sudo lsof +L1 | grep victim                 # NLINK=0, 50MB, (deleted)
cp /proc/$PID/fd/3 /tmp/recovered.log        # ★ RECOVERED
truncate -s 0 /proc/$PID/fd/3; df -h /tmp | tail -1   # ★ space freed, no restart
kill $PID

# PROVE IT: the limit is per-process — /proc/PID/limits is the truth, not `ulimit`
ulimit -n; cat /proc/$$/limits | grep -i 'open files'; cat /proc/1/limits | grep -i 'open files'

# PROVE IT: hit EMFILE yourself (errno 24 == EMFILE, exactly what Node throws)
bash -c 'ulimit -n 20; python3 -c "import os
fds=[]
try:
 while True: fds.append(os.open(\"/etc/hostname\",os.O_RDONLY))
except OSError as e: print(len(fds),e)"'
# → 17 [Errno 24] Too many open files

sudo lsof -i :3000 -P -n     # what's on port 3000?
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
sleep 500 &
PID=$!
ls -l /proc/$PID/fd
cat /proc/$PID/limits | grep -i 'open files'
kill $PID

sleep 500 > /tmp/out.txt 2>/tmp/err.txt &
PID=$!
ls -l /proc/$PID/fd
kill $PID
```
**Questions:** In the first case, what do fds 0, 1 and 2 point to, and why do all three point to the *same* thing? In the second case, which fds changed and what does the `l-wx` vs `lrwx` permission string on the symlink tell you? What is the soft limit, and how does it compare to `ulimit -n` in your shell?

---

### Exercise 2 — Medium

**Prove the `2>&1` mechanism to yourself, empirically.**

1. Write a program that writes a long string to stdout and a long string to stderr, without buffering. Run it two ways:
   ```bash
   ./prog > /tmp/wrong.log 2> /tmp/wrong.log      # two open()s
   ./prog > /tmp/right.log 2>&1                   # one open() + dup2
   ```
2. `diff` the results. Explain the corruption in `wrong.log` using the three-level table diagram — specifically, name which *table* the two cases differ in.
3. Confirm your explanation with `strace -f -e trace=openat,dup2 ...` for both.
4. Now run the program under `strace` and, in another terminal, `ls -l /proc/PID/fd` — show that in the wrong case fd 1 and fd 2 are *different open-file entries*, and in the right case they are the *same* one. **Hint:** compare the inode numbers shown by `lsof -p PID` — but note both point to the same *inode*. What is the observable difference, and where do you find it? (`lsof`'s `SIZE/OFF` column.)

---

### Exercise 3 — Hard (Production Simulation)

Build the incident, then solve it.

1. Write a tiny Node HTTP server that **leaks an fd per request** (e.g. `fs.openSync('/etc/hostname','r')` and never close it). Run it under systemd with `LimitNOFILE=64` so it dies fast.
2. Hammer it (`for i in $(seq 100); do curl -s localhost:3000 >/dev/null; done`) until it throws `EMFILE`.
3. **Without reading your own source code**, diagnose it purely from the box:
   - Confirm the limit: `cat /proc/PID/limits`
   - Confirm the count: `ls /proc/PID/fd | wc -l`
   - Prove it is a **leak, not load**: log the count for 3 minutes with traffic stopped
   - Identify **what type** is leaking: `lsof -p PID | awk '{print $5}' | sort | uniq -c | sort -rn`
   - Identify **which exact file/socket**: `ls -l /proc/PID/fd | awk '{print $NF}' | sort | uniq -c | sort -rn`
4. Fix it in two places: raise `LimitNOFILE` in the unit **and** close the fd. Verify the new limit is live via `/proc/PID/limits` (not `ulimit`).
5. **Then reproduce the disk-space incident:** have the server write to `/tmp/app.log`, `rm` the log while it's running, and show:
   - `df` and `du` disagree — quantify the gap
   - `lsof +L1` finds the culprit
   - `truncate -s 0 /proc/PID/fd/N` reclaims the space **with the process still running**
   - `cp /proc/PID/fd/N /tmp/recovered.log` recovers the content

Write a 5-line incident report: symptom, root cause (at the kernel-table level), mitigation, permanent fix, prevention.

---

## Mental Model Checkpoint

Answer from memory.

1. **Draw the three-level structure.** Name each table and say exactly what each level stores. Which level holds the **file offset**? Which holds the **filename**? (Trick question — reread topic 04.)
2. **You `open()` the same file twice in one process. Do the two fds share a read offset? Why or why not — in terms of the tables?**
3. **You run `cmd > f 2>&1`. Which syscall makes fd 2 behave correctly, and what exactly does it do to the tables?** Why does `cmd > f 2> f` corrupt the file?
4. **Why do fds survive `execve()`, and what flag opts out?** Why does the shell depend on this behaviour absolutely?
5. **`df` says the disk is 100% full; `du` says 40% used. What is happening, which single command finds it, and how do you fix it WITHOUT restarting the process?**
6. **Your Node app throws `EMFILE`. Give the four commands, in order, that take you from "it's broken" to "line 47 of report.js opens a DB connection per request."**
7. **Why does `ulimit -n 65535` in your SSH session not fix a systemd service — and what is the correct fix?** Why does `/etc/security/limits.conf` also not fix it?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `ls -l /proc/PID/fd` | Every open fd as a symlink — files, pipes, sockets | Works with no tools installed (Alpine!) |
| `ls /proc/PID/fd \| wc -l` | **Current fd count** | Poll it to detect a leak |
| `cat /proc/PID/limits` | **The RUNNING process's real limits** | The only trustworthy source. Not `ulimit` |
| `ulimit` | Get/set the *shell's* limits (bash builtin) | `-n` open files, `-a` all, `-u` procs, `-c` core, `-H` hard, `-S` soft |
| `lsof -i :3000 -P -n` | **What's on my port?** (`EADDRINUSE`) | `-P` no port names, `-n` no DNS, `-sTCP:LISTEN` |
| `lsof -p PID` | Everything a process has open | `-a -i` = AND, network only |
| `lsof <file>` | **Who holds this file?** (`device is busy`) | — |
| `lsof +D /var/log` | Everything open under a dir, recursively | Slow — stats every file |
| `lsof +L1` | ★ **Deleted-but-open files** (the `df`/`du` gap) | — |
| `lsof -u deploy` | Everything a user has open | `-a` to AND with other filters |
| `truncate -s 0 <file>` | Empty a file **in place** — fd stays valid | ★ Use instead of `rm` on a live log |
| `truncate -s 0 /proc/PID/fd/N` | Reclaim space from a **deleted-but-open** file | No restart needed |
| `cp /proc/PID/fd/N /tmp/x` | **Recover** a deleted file from a live process | Only while the process lives |
| `exec 3< file` / `exec 3<&-` | Open/close an fd in the shell itself | `3>` write, `3<>` read-write |
| `cat /proc/sys/fs/file-nr` | System-wide fds: allocated / free / max | `fs.file-max` via sysctl |
| systemd `LimitNOFILE=65535` | ★ **The real fix for a service's fd limit** | In the `[Service]` section |
| `docker run --ulimit nofile=65535` | The fd limit for a container | — |

---

## When Would I Use This at Work?

### Scenario 1: `EADDRINUSE` on every deploy
Your deploy script does `systemctl restart api` and it fails: `listen EADDRINUSE :::3000`. `lsof -i :3000 -P -n` shows a stray `ffmpeg` child process — spawned by the app for thumbnail generation — holding the **listening socket** it inherited across `fork`+`exec`. The parent died; the child kept the open-file entry alive, so the refcount never dropped to zero and the kernel never released the port. **Fix:** kill the orphan, and make sure `KillMode=control-group` (systemd's default) tears down the whole cgroup so orphaned children die with the service.

### Scenario 2: "Disk is full but I deleted everything"
`df` says `/` is 100% full. You delete gigabytes of old logs. `df` doesn't budge. `du -sh /` reports half of what `df` does. You run `sudo lsof +L1` and find a 30 GB file with `NLINK 0` still held open by your Node process — logrotate `rm`'d it three weeks ago without telling the app, and Node has been writing into a nameless inode ever since. **Fix:** `truncate -s 0 /proc/PID/fd/N` reclaims 30 GB instantly with zero downtime. **Prevent:** logrotate `copytruncate`, or `SIGHUP` the app to reopen its log (→ topic 23).

### Scenario 3: The app dies at exactly 4am, every night
Restarting "fixes" it, so nobody dug in. You poll `ls /proc/$PID/fd | wc -l` every minute and graph it: a perfectly straight line, climbing 2 fds/minute, 24/7, **regardless of traffic**. `lsof -p $PID | awk '{print $5}' | sort | uniq -c` shows 900+ `IPv4`. `lsof -p $PID -a -i` shows they all point at port 5432. **A dashboard poller hits an endpoint that opens a new Postgres client per request and never closes it.** 1024 ÷ 2/min ≈ 8.5 hours — which is exactly the interval between crashes. **Fix:** a shared `Pool`. **Prevent:** an fd-count gauge with an alert at 80% of `RLIMIT_NOFILE`.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 03 — Everything Is a File | Sockets, pipes, devices and files all get fds because they're all "files" |
| **Builds on** | 04 — Inodes in Depth | The third level of the table is the inode. The deleted-file trick is pure link-count mechanics |
| **Builds on** | 12 — Pipes and Redirection | `>`, `2>&1`, `\|` are *literally* `open()` + `dup2()` on the fd table. This doc is the *why* behind that doc |
| **Builds on** | 16 — Processes in Depth | `fork()` copies the fd table; `execve()` preserves it. That's why redirection works at all |
| **Builds on** | 18 — System Resource Monitoring | fds are a resource that runs out — and it never shows up in `free` or `top` |
| **Next** | 20 — Cron and Scheduled Tasks | Cron jobs inherit a *bare* environment — and their output goes to fds you must redirect yourself |
| **Used by** | 22 — systemd and Services | `LimitNOFILE=` is the real fix for `EMFILE` |
| **Used by** | 23 — Logs and Log Management | `logrotate` + `copytruncate` exists *entirely* because of the deleted-but-open-file problem |
| **Used by** | 28 — Ports and Sockets | `lsof -i` and `ss` both resolve `socket:[inode]` back to an IP:port pair |
| **Used by** | 31 — Disk Management | The `df` vs `du` discrepancy is an fd problem, not a disk problem |
| **Used by** | 33 — Node.js in Production | Connection pooling, stream cleanup, and `LimitNOFILE` are the three pillars of a Node box that survives the night |
