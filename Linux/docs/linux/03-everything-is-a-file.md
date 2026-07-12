# 03 — Everything Is a File

## ELI5 — The Simple Analogy

Imagine a giant office building where **every single thing you interact with is a mail slot in a wall**.

- Want to send a letter to a coworker? Push it through their mail slot.
- Want to talk to the printer? Push your document through the printer's mail slot.
- Want to throw something away forever? There's a slot labelled "SHREDDER" — push paper in, it's gone, and if you ever reach *into* it you get nothing back.
- Want an endless supply of blank paper? There's a slot labelled "BLANK" — reach in and pull out as much empty paper as you want, forever.
- Want to talk to the person in the next room over a private line? There's a slot for that too.

Every slot looks **identical from the outside**. You interact with all of them the exact same way: **push stuff in (write), reach in and pull stuff out (read).** You don't need to know a special ritual for the printer versus the shredder versus your coworker. One motion — push or pull — works on all of them.

Behind each slot, something completely different is happening. Behind the shredder is a shredder. Behind the printer is a printer. Behind your coworker's slot is a person. But **you don't care**, because the *interface* is always the same slot in the wall.

That is "everything is a file." In Linux, a hard disk, a network connection, your terminal, a running process's memory, `/dev/null`, and a text document are all **slots in the wall**. You open them, read from them, write to them, and close them — with the same four motions. The kernel figures out what's actually behind each slot.

---

## Where This Lives in the Linux Stack

```
Hardware (disks, NICs, keyboards, RAM, RNG chips)
  └── KERNEL
       │    ┌──────────────────────────────────────────────────────┐
       │    │  VFS — Virtual File System                            │
       │    │  Defines ONE interface: open/read/write/close/lseek   │
       │    │  Behind it, DIFFERENT DRIVERS implement that interface│ ◀◀◀ THIS TOPIC
       │    │   • ext4 driver      → regular files & dirs           │
       │    │   • tty driver       → your terminal                  │
       │    │   • null driver      → /dev/null (discard everything) │
       │    │   • socket layer     → /var/run/docker.sock           │
       │    │   • pipe/fifo code   → named pipes                    │
       │    └──────────────────────────────────────────────────────┘
       │
       └── System Calls: open() read() write() close() — the SAME 4 for everything
            │
            └── C Library (glibc — read(2), write(2) thin wrappers)
                 │
                 └── Shell (redirection: >  <  |  all just wire up file descriptors)
                      │
                      └── Commands (cat, ls, ln, stat, lsof, mkfifo, mknod)
                           ◀◀◀ the tools that reveal the 7 file types
```

**The core idea:** the VFS defines a uniform interface. Wildly different things — a spinning disk, a TCP socket, a random-number generator — all hide behind `read()` and `write()`. That uniformity IS the Unix abstraction.

---

## What Is This?

"Everything is a file" means Linux exposes almost every resource — real files, directories, hardware devices, network sockets, inter-process pipes, even a process's own memory and status — through **one uniform interface**: you `open()` it to get a **file descriptor** (a small integer), then `read()` and `write()` that descriptor, then `close()` it.

It does NOT mean everything is *literally a document on a disk*. It means everything **speaks the file protocol**. A more accurate slogan is "everything is a **file descriptor**" — a uniform handle you perform the same operations on, regardless of what's behind it.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `>` `<` `\|` are just fd rewiring | Redirection and pipes will feel like magic instead of a mechanism you control (Topic 12) |
| Sockets are files | You won't understand why Docker mounts `/var/run/docker.sock`, or how your app talks to Postgres over a unix socket |
| Symlink vs hard link | You'll break a deploy with a dangling symlink, or be baffled that `rm original.log` didn't free disk space |
| A directory is a file of name→inode maps | Topic 04 (inodes) and the whole permission model won't click |
| `/dev/null` is a real device driver | You'll write `2>/dev/null` without knowing you're sending stderr to a discard driver |
| Deleted-but-open files | You'll `rm` a huge log and be mystified that the disk is still full (the #1 production trap) |

---

## The 7 File Types

Every entry in the filesystem is exactly one of seven types. The **first character** of `ls -l` output tells you which:

```
$ ls -l
crw-rw-rw-  1 root root 1, 3 Jul 12 /dev/null
brw-rw----  1 root disk 8, 0 Jul 12 /dev/sda
srw-rw----  1 root docker 0  Jul 12 /var/run/docker.sock
prw-r--r--  1 app  app  0    Jul 12 /tmp/mypipe
lrwxrwxrwx  1 root root 7    Jul 12 /bin -> usr/bin
drwxr-xr-x  2 root root 4096 Jul 12 /etc
-rw-r--r--  1 app  app  512  Jul 12 server.js
^
│
└── THIS FIRST CHARACTER is the file type
```

| Char | Type | What it is | Example |
|---|---|---|---|
| `-` | **Regular file** | Ordinary data: text, binaries, images | `server.js`, `/usr/bin/node` |
| `d` | **Directory** | A file containing name→inode mappings | `/etc`, `/home/deploy` |
| `l` | **Symbolic link** | A tiny file whose *data* is a path string | `/bin → usr/bin` |
| `c` | **Character device** | Unbuffered, byte-stream hardware | `/dev/null`, `/dev/tty`, `/dev/urandom` |
| `b` | **Block device** | Buffered, block-addressable storage | `/dev/sda`, `/dev/nvme0n1` |
| `s` | **Socket** | A bidirectional communication endpoint | `/var/run/docker.sock` |
| `p` | **Named pipe (FIFO)** | A one-way byte channel between processes | created by `mkfifo` |

Mnemonic for the exotic four: **c**haracter, **b**lock, **s**ocket, **p**ipe → "**c**an **b**e **s**een **p**lainly."

```bash
# PROVE IT: see multiple types at once
ls -l /dev/null /dev/sda /bin /etc/hostname 2>/dev/null
# crw-rw-rw- ... /dev/null      ← c: character device
# brw-rw---- ... /dev/sda       ← b: block device
# lrwxrwxrwx ... /bin -> usr/bin ← l: symlink
# -rw-r--r-- ... /etc/hostname  ← -: regular file
```

The `file` command names the type too, but reads deeper (magic bytes for regular files):

```bash
$ file /dev/null /etc /bin server.js
/dev/null:  character special (1/3)
/etc:       directory
/bin:       symbolic link to usr/bin
server.js:  ASCII text
```

---

## What a Directory ACTUALLY Is

A directory is **not** a container that "holds" files. It is a **special file whose contents are a table mapping names to inode numbers.**

```
Directory /home/deploy   →  its data blocks contain literally this table:

     ┌─────────────┬────────────┐
     │  name       │  inode #   │
     ├─────────────┼────────────┤
     │  .          │  262145    │  ← itself
     │  ..         │  131074    │  ← its parent
     │  server.js  │  262180    │
     │  package.json│ 262181    │
     │  node_modules│ 262200    │  ← inode of a directory
     └─────────────┴────────────┘
```

The **filename lives in the directory, not in the file.** The file itself (its inode) has no idea what it's called. This one fact explains everything that follows:

- **Hard links** — the same inode number appearing under two different names in the table.
- **Renaming** — just editing the name column; the inode and data never move. That's why `mv` within a filesystem is instant even for a 50GB file.
- **`rm`** — deleting a row from the table (an `unlink()` call), NOT erasing data.
- **Permission to list a dir** needs `r`; permission to *access files through it* needs `x`. (Topic 05.)

```bash
# PROVE IT: a directory has a size — its size is the size of the name table
stat -c '%s %n' /etc
# 12288 /etc     ← 12KB of name→inode mappings

# PROVE IT: read the raw name→inode table of the current directory
ls -lai .
#   ^ -i shows inode numbers, -a shows . and ..
# 262145 drwxr-xr-x  .
# 131074 drwxr-xr-x  ..
# 262180 -rw-r--r--  server.js     ← name in THIS dir, inode is the file's identity
```

Topic 04 — Inodes in Depth takes this all the way down to the on-disk structures. For now: **directory = a file of name→inode rows.**

---

## Symlinks vs Hard Links — In Depth

This is the single most-confused topic in all of Linux. Slow down here.

### Hard link — another name for the SAME inode

A hard link is a **second row in a directory table pointing at an inode that already exists.** There is no "original" and no "copy" — there are two equal names for one identical file.

```
BEFORE:  ln original.txt  hardlink.txt

  Directory table:                    Inode 500  (link count: 2)
  ┌──────────────┬────────┐          ┌────────────────────────┐
  │ original.txt │  500   │─────────▶│ owner, perms, size,    │
  │ hardlink.txt │  500   │─────────▶│ timestamps,            │
  └──────────────┴────────┘          │ → data blocks on disk  │
         both names point to         └────────────────────────┘
         the SAME inode number
```

- **Link count** = how many names point to this inode. `ls -l` shows it. When it hits **0**, the kernel frees the data blocks.
- **Cannot cross filesystems.** An inode number is only meaningful within one filesystem. A hard link on `/var` cannot point to an inode on `/home` — they're different inode tables. Attempt it: `Invalid cross-device link`.
- **Cannot link directories** (by default). Doing so would let you create loops in the tree that would break `find`, `pwd`, and every tree-walker. Only root, only with special tools, and you shouldn't.

```bash
# PROVE IT: hard links share one inode and a link count
echo "data" > original.txt
ln original.txt hardlink.txt          # NOTE: no -s → HARD link
ls -li original.txt hardlink.txt
# 262180 -rw-r--r-- 2 you you 5 ... original.txt    ← link count is 2
# 262180 -rw-r--r-- 2 you you 5 ... hardlink.txt    ← SAME inode 262180, count 2
#      ^ same number       ^ count

# Delete the "original" — nothing is lost:
rm original.txt
cat hardlink.txt
# data                                  ← still here! inode 262180 survives
ls -li hardlink.txt
# 262180 -rw-r--r-- 1 you you 5 ... hardlink.txt    ← link count dropped to 1
```

**What just happened at the kernel level:** `rm original.txt` called `unlink("original.txt")`. The kernel removed that ONE row from the directory table and decremented inode 262180's link count from 2 to 1. Because the count is still ≥ 1, the inode and its data blocks are untouched. The word "original" was never special — it was just one of two equal names.

### Symbolic link — a tiny file that CONTAINS a path

A symlink is its **own separate file, with its own inode**, whose entire data content is a **string: the path to something else.** It's a signpost, not a second name.

```
  ln -s /var/www/app/current  latest

  latest is a NEW file, inode 900, type 'l':
  ┌─────────────────────────────────────┐
  │ Inode 900 (type: symlink)           │
  │ data = "/var/www/app/current"       │  ← just a text path, 20 bytes
  └─────────────────────────────────────┘
                    │
                    │ the kernel FOLLOWS this string when you open `latest`
                    ▼
  /var/www/app/current  (a totally separate inode, maybe on another disk)
```

- **Can cross filesystems.** It stores a *path*, and paths are universal. This is why symlinks are used everywhere in deployments.
- **Can point to directories.** No loop danger, because tools know to treat symlinks specially.
- **Can dangle.** If the target is deleted or never existed, the symlink still exists but points to nothing → "broken"/"dangling" link. Opening it gives `ENOENT`.
- Has its **own permissions display** (`lrwxrwxrwx`) but they're ignored — the target's permissions are what matter.

```bash
# PROVE IT: a symlink is a separate file whose DATA is a path
ln -s /etc/hostname  hostlink
ls -li /etc/hostname hostlink
# 131 -rw-r--r-- 1 root root  9 ... /etc/hostname
# 902 lrwxrwxrwx 1 you  you 12 ... hostlink -> /etc/hostname
#  ^ DIFFERENT inode   ^ its size is 12 = length of the string "/etc/hostname"

readlink hostlink
# /etc/hostname                         ← the symlink's data IS this path string

# PROVE IT: a symlink can dangle; a hard link never can
ln -s /does/not/exist  broken
cat broken
# cat: broken: No such file or directory     ← the target is gone, link remains
ls -l broken
# lrwxrwxrwx 1 you you 15 ... broken -> /does/not/exist   ← still here, pointing at nothing
```

### Side-by-side

| | Hard link | Symlink (`ln -s`) |
|---|---|---|
| What it is | Another name for the same inode | A separate file containing a path string |
| Own inode? | No — shares the target's | Yes — its own |
| Crosses filesystems? | **No** (`EXDEV` error) | **Yes** |
| Links directories? | No (would create loops) | Yes |
| Can dangle? | No — target always exists while it does | Yes — target can vanish |
| Delete the "target" | Nothing lost; data survives while count > 0 | Symlink now dangles (broken) |
| `ls -l` first char | `-` (looks like a normal file) | `l` |
| Link count effect | Increments the shared inode's count | Target's count unchanged |
| Command | `ln a b` | `ln -s a b` |

**Which do deployments use?** Almost always **symlinks**. The classic zero-downtime deploy:

```bash
/var/www/releases/2026-07-12-a3f9c/     ← the new release
/var/www/current -> releases/2026-07-12-a3f9c    ← symlink flipped atomically
```
nginx and systemd point at `/var/www/current`. To deploy, you extract a new release dir, then atomically repoint the `current` symlink. Instant, reversible, no half-copied files. Symlinks (cross-directory, repointable) are perfect; hard links couldn't do this.

---

## Device Files — hardware as files

A device file has no data blocks. Its inode instead stores **two numbers** that tell the kernel which driver to call:

- **Major number** = which driver (e.g. 8 = SCSI/SATA disk driver, 1 = memory devices).
- **Minor number** = which specific device that driver manages (which disk, which partition).

```bash
$ ls -l /dev/null /dev/sda /dev/sda1
crw-rw-rw- 1 root root    1,   3 Jul 12 /dev/null
brw-rw---- 1 root disk    8,   0 Jul 12 /dev/sda
brw-rw---- 1 root disk    8,   1 Jul 12 /dev/sda1
#                         │    │
#                         │    └── MINOR: 0 = whole disk, 1 = first partition
#                         └── MAJOR: 8 = the sd (SCSI/SATA) block driver
# ↑ where a regular file shows a SIZE, a device shows "major, minor"
```

### Character vs block

| | Character device (`c`) | Block device (`b`) |
|---|---|---|
| Access | Byte stream, unbuffered, sequential-ish | Fixed-size blocks, buffered, random-access |
| Analogy | A pipe of bytes flowing past | An array of numbered storage boxes |
| Examples | `/dev/null`, `/dev/tty`, `/dev/urandom` | `/dev/sda`, `/dev/nvme0n1`, `/dev/loop0` |
| You mount | Never | Yes — filesystems live on block devices |

### The essential device files

```bash
# /dev/null — the bit bucket. Writes vanish, reads give instant EOF.
echo "goodbye" > /dev/null        # discarded. This is the null DRIVER eating bytes.
cat /dev/null                     # returns nothing, immediately (EOF)

# /dev/zero — infinite zero bytes. For allocating/zeroing.
head -c 10 /dev/zero | xxd
# 00000000: 0000 0000 0000 0000 0000              ← ten NUL bytes

# /dev/urandom — cryptographic randomness (what Node's crypto.randomBytes uses)
head -c 16 /dev/urandom | xxd
# 00000000: a3f9 1c7e 44b0 ...                     ← 16 random bytes

# /dev/sda vs /dev/sda1 — whole disk vs a partition ON that disk
lsblk
# NAME   MAJ:MIN SIZE TYPE MOUNTPOINT
# sda      8:0   500G disk
# ├─sda1   8:1   512M part /boot
# └─sda2   8:2 499.5G part /
```

`/dev/null` vs `/dev/sda` is the whole abstraction in one line: both are files you `write()` to. One driver throws the bytes away; the other lays them on spinning platters. **Same syscall, different driver.**

---

## Unix Domain Sockets — processes talking over a "file"

A socket file (`s`) is an endpoint for **two processes on the same machine to talk**, using the same socket API as network sockets — but with **no network stack, no TCP/IP overhead**. The "address" is a path in the filesystem.

```bash
$ ls -l /var/run/docker.sock
srw-rw---- 1 root docker 0 Jul 12 09:00 /var/run/docker.sock
#^ 's' = socket. Size 0 — it's an endpoint, not stored data.
```

**Why Docker uses it, and why you mount it:** The Docker CLI (`docker ps`) doesn't do the work — it sends a request to the Docker **daemon** (`dockerd`). They communicate over `/var/run/docker.sock`. When you run a container that needs to control Docker (a CI runner, Portainer, Traefik), you do:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

This mounts the daemon's socket *into* the container, so the process inside can speak to the host's Docker daemon. **Security warning:** this is effectively giving that container root on the host — anything that can talk to the socket can launch a privileged container that mounts `/`. (Topic 35 — Security.)

**Postgres does the same:**
```bash
$ ls -l /var/run/postgresql/
srwxrwxrwx 1 postgres postgres 0 Jul 12 .s.PGSQL.5432
#^ socket. Local connections use THIS instead of TCP 127.0.0.1:5432.
```
When your Node app connects with host `/var/run/postgresql` (or no host), it uses this unix socket — faster and more secure than TCP, because it never touches the network stack and is protected by filesystem permissions. Connecting with host `127.0.0.1` uses a real TCP socket on port 5432 instead. Same `pg` library, different kind of "file" behind the connection.

---

## Named Pipes (FIFOs) — a file you pour bytes through

A regular pipe (`ls | grep`) is anonymous and vanishes when the command ends. A **named pipe** is a pipe with a **name in the filesystem** (`p` type), so two *unrelated* processes — started separately — can connect to it.

Key property: it has **no storage**. A write blocks until someone reads. It's a rendezvous point, not a buffer.

```bash
# PROVE IT: a working named-pipe demo (run in TWO terminals)

# --- Terminal 1: create the pipe and read from it ---
mkfifo /tmp/mypipe
ls -l /tmp/mypipe
# prw-r--r-- 1 you you 0 Jul 12 /tmp/mypipe    ← 'p', size 0
cat /tmp/mypipe          # BLOCKS here, waiting for a writer...

# --- Terminal 2: write into the pipe ---
echo "message through a file" > /tmp/mypipe

# --- Back in Terminal 1, instantly appears: ---
# message through a file
# (and `cat` exits, because the writer closed → EOF)

rm /tmp/mypipe
```

**Real backend use:** streaming a huge database dump to a compressor without a temp file, or feeding a process that only reads from a file path:

```bash
# Backup Postgres straight into gzip via a FIFO — no giant intermediate .sql file
mkfifo /tmp/dump.pipe
gzip < /tmp/dump.pipe > backup.sql.gz &     # reader waits
pg_dump mydb > /tmp/dump.pipe               # writer feeds it
rm /tmp/dump.pipe
```

---

## /proc and /dev — proof the abstraction is universal

The strongest evidence that "everything is a file" is real: you can `cat` a running process's internal state and the kernel's own memory statistics, using the *same command* you'd use on a text file.

```bash
# PROVE IT: read a process's live status as if it were a text file
cat /proc/self/status | head -8
# Name:   cat            ← the process doing the reading
# State:  R (running)
# Tgid:   30012
# Pid:    30012
# PPid:   28733
# VmRSS:  1216 kB        ← this process's real memory use, RIGHT NOW
# Threads: 1

# There is NO status.txt on any disk. The kernel MANUFACTURES this text
# the instant you read(), then hands it back through the same read() syscall
# you'd use on server.js. The file interface is a universal facade.
```

---

## How It Works — The Punchline: Same Syscall, Different Driver

```
COMMAND A: echo "x" > file.txt
COMMAND B: echo "x" > /dev/null

SHELL DOES (both):   opens the target with O_WRONLY|O_CREAT|O_TRUNC,
                     dup2()s that fd onto fd 1 (stdout), then execs echo.
ECHO DOES (both):    write(1, "x\n", 2)      ← IDENTICAL system call
                                                echo has NO idea what fd 1 is.

KERNEL DOES:
  For file.txt   → VFS routes write() to the EXT4 DRIVER
                 → allocates/updates data blocks, marks inode dirty, schedules disk I/O
  For /dev/null  → VFS routes write() to the NULL DRIVER
                 → the null driver's write function is literally: "return count; do nothing"

RESULT:  Byte-for-byte identical userspace behavior. The ONLY difference is which
         driver function the VFS dispatched to behind file descriptor 1.
```

You can watch this with `strace` and see the two `write()` calls are indistinguishable:

```bash
strace -e write sh -c 'echo x > /tmp/f'   2>&1 | grep 'write(1'
# write(1, "x\n", 2) = 2
strace -e write sh -c 'echo x > /dev/null' 2>&1 | grep 'write(1'
# write(1, "x\n", 2) = 2      ← the exact same call
```

This is the entire reason Unix pipelines, redirection, and tooling compose so beautifully: no program needs to know whether its output is a file, a terminal, a pipe, a socket, or the void. It just calls `write()`.

---

## Exact Syntax Breakdown

```
ls -li /var/log
│  ││  │
│  ││  └── directory to list
│  │└── -i = print each file's INODE NUMBER in the first column
│  └── -l = long format (type char, perms, links, owner, size, mtime, name)
└── ls = list. The type character is the FIRST char of the -l output.
```

```
ln  original.txt  hardlink.txt
│   │             │
│   │             └── the new NAME (a new directory-table row → same inode)
│   └── an existing file whose inode we point a second name at
└── ln = "link". With NO -s → HARD link. Increments the inode's link count.

ln -s /var/www/current  latest
│  │  │                 │
│  │  │                 └── the new SYMLINK file to create
│  │  └── the TARGET path (stored verbatim as the symlink's data — may not exist!)
│  └── -s = SYMBOLIC. Creates a separate inode of type 'l'.
└── ln
```

```
readlink -f  /bin/sh
│        │   │
│        │   └── the symlink (or path) to resolve
│        └── -f = follow the ENTIRE chain of symlinks to the final real path
└── readlink — prints a symlink's target. Bare readlink = one hop; -f = all hops.
```

```
mkfifo /tmp/mypipe
│      │
│      └── path of the named pipe to create
└── mkfifo — creates a FIFO special file (type 'p') via the mknod() syscall.
```

```
mknod /dev/mynull  c  1  3
│     │           │  │  │
│     │           │  │  └── minor number (3 = the null device)
│     │           │  └── major number (1 = memory-device driver)
│     │           └── c = character device (b = block, p = fifo)
│     └── path of the device node to create
└── mknod — "make node". Creates device files. Needs root. Rare today (udev
    auto-creates /dev entries), but proves a device file is just an inode + 2 numbers.
```

```
stat -c '%F %i %h %n'  hardlink.txt
│    │   │  │  │  │     │
│    │   │  │  │  │     └── the file
│    │   │  │  │  └── %n = name
│    │   │  │  └── %h = number of hard links (the link count)
│    │   │  └── %i = inode number
│    │   └── %F = human file type ("regular file", "symbolic link", "directory"...)
│    └── -c = custom output format string
└── stat — dumps inode metadata. The precise, scriptable version of `ls -l`.
```

```
lsof +L1
│    │
│    └── +L1 = list files whose link count is < 1 (deleted, but still OPEN)
└── lsof = "list open files". THE tool for the deleted-but-open disk-space mystery.
    Also: lsof -p PID (one process), lsof -i :3000 (network), lsof /path (who has it open)
```

---

## Example 1 — Basic: See All Seven Types and Prove Links

```bash
# 1. Make a workspace
mkdir /tmp/filetypes && cd /tmp/filetypes

# 2. A regular file (type -)
echo "hello" > regular.txt

# 3. A directory (type d)
mkdir subdir

# 4. A hard link (still type -, but shares regular.txt's inode)
ln regular.txt hard.txt

# 5. A symlink (type l)
ln -s regular.txt soft.txt

# 6. A named pipe (type p)
mkfifo mypipe

# 7. Look at them all with inode numbers
ls -li
# 262180 -rw-r--r-- 2 you you 6 ... regular.txt   ← inode 262180, link count 2
# 262180 -rw-r--r-- 2 you you 6 ... hard.txt      ← SAME inode, that's why count=2
# 262185 lrwxrwxrwx 1 you you 11 ... soft.txt -> regular.txt  ← own inode, type l
# 262186 prw-r--r-- 1 you you 0 ... mypipe        ← type p, size 0
# 262184 drwxr-xr-x 2 you you 4096 ... subdir     ← type d

# 8. Character & block devices already exist in /dev — just observe them
ls -l /dev/null /dev/sda 2>/dev/null
# crw-rw-rw- 1 root root 1, 3 ... /dev/null       ← c, major 1 minor 3
# brw-rw---- 1 root disk 8, 0 ... /dev/sda        ← b, major 8 minor 0

# 9. Prove the hard link IS the file: delete the "original"
rm regular.txt
cat hard.txt
# hello                                            ← data survives! (inode still linked)

# 10. Prove the symlink now DANGLES (it pointed at a NAME, and the name is gone)
cat soft.txt
# cat: soft.txt: No such file or directory        ← broken symlink

# cleanup
cd / && rm -rf /tmp/filetypes
```

---

## Example 2 — Production Scenario

**Situation:** 02:40. Alert: `/` filesystem at 100%. Your Node API is throwing `ENOSPC`. You SSH in, and you *did* delete the giant log an hour ago — but the disk is still full.

```bash
# Step 1 — Confirm the disk really is full
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        49G   49G     0 100% /

# Step 2 — But du says the files only add up to 20G. Contradiction!
$ sudo du -sh /var /home /opt /usr 2>/dev/null
14G     /var
2.1G    /home
3.0G    /opt
4.8G    /usr
# Total ~24G accounted for, on a 49G disk that's 100% full. Where are the other 25G?

# Step 3 — The answer: a file with NO NAME, held open by a process.
#          Earlier you ran `rm /var/log/api/app.log` while node had it open.
#          rm removed the NAME (link count → 0) but node still holds the fd,
#          so the inode and its 25GB of blocks are NOT freed.
$ sudo lsof +L1
COMMAND  PID   USER   FD   TYPE DEVICE   SIZE/OFF NLINK    NODE NAME
node    1847 deploy   7w   REG  253,2 26843545600     0 2621999 /var/log/api/app.log (deleted)
#                                                       ^^^^^ NLINK 0 = no name left
#                                                             but 25GB still allocated
#                                       ^ the process still WRITING to a ghost file

# du can't see it (no directory entry to walk to), but df counts the blocks (still allocated).
# THIS is why df and du disagree — the classic interview question, live in production.

# Step 4 — Recover WITHOUT killing node (zero downtime). Truncate the ghost via /proc:
$ sudo truncate -s 0 /proc/1847/fd/7
#                     ^^^^^^^^^^^^^^^ /proc/PID/fd/N is a SYMLINK to the open file —
#                     even a deleted one. "Everything is a file" lets you reach a
#                     nameless file THROUGH the process that holds it open.

$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        49G   23G   26G  47% /          ← 25GB freed instantly, node never died

# Step 5 — Real fix: don't rm open logs. Configure logrotate with copytruncate,
#          and have the app reopen its log file on SIGHUP. (Topics 17 & 23.)
```

**What made this solvable:** you understood that a filename is just a directory-table row (Topic 04's territory), that `rm` = `unlink()` decrements a link count, that a file survives with link count 0 while a process holds it open, and that `/proc/PID/fd/N` exposes that open file as a path you can act on. Pure "everything is a file."

---

## Common Mistakes

### Mistake 1 — Thinking `cp` on a symlink copies the link

**Wrong:**
```bash
$ ln -s /etc/nginx/nginx.conf  conf
$ cp conf /backup/                  # you think you copied a symlink
```
`cp` **follows** the symlink by default and copies the *target's contents*, giving you a regular file, not a link. To copy the link itself:
```bash
$ cp -P conf /backup/               # -P = never follow symlinks (copy link as-is)
$ cp -a /var/www /backup/           # -a = archive: preserves symlinks, perms, times
```
**Root cause:** almost every tool follows symlinks by default because a symlink is *meant* to be transparent. You must opt out (`-P`, `-a`, `--no-dereference`) to operate on the link itself.

---

### Mistake 2 — Hard-linking across filesystems and being confused by the error

**Wrong:**
```bash
$ ln /var/log/app.log  /home/deploy/app.log
ln: failed to create hard link '/home/deploy/app.log' => '/var/log/app.log':
    Invalid cross-device link
```
**Root cause (kernel level):** `/var` and `/home` are **different filesystems** (different `st_dev`, Topic 02). A hard link is a directory row pointing to an **inode number**, and inode numbers are only unique *within one filesystem*. Inode 500 on `/var` and inode 500 on `/home` are different files. The kernel refuses (`EXDEV`) because the reference would be meaningless.
**Fix:** use a **symlink** (`ln -s`) — it stores a path, which crosses filesystems fine.
**Prevent:** default to `ln -s` unless you specifically need same-inode semantics on one filesystem.

---

### Mistake 3 — `rm` a big open logfile, disk stays full

**Wrong:** `rm /var/log/api/huge.log` while the app writes to it; `df` unchanged.
**Right:** `truncate -s 0 /var/log/api/huge.log` (keeps the name and fd valid, empties the data).
**Root cause:** covered in Example 2 — `unlink()` drops the name and link count, but data blocks are freed only when link count is 0 **and** no fd holds the inode open. This is not a bug; it's what lets a program keep a private temp file that auto-deletes when it exits.
**Diagnose:** `sudo lsof +L1`. **Prevent:** never `rm` a file a running process writes to; truncate it, and use logrotate `copytruncate`.

---

### Mistake 4 — `> /dev/null` when you meant `2>/dev/null` (or both)

**Wrong:**
```bash
$ noisy-command > /dev/null        # you still see the errors!
error: could not connect...        # stderr (fd 2) is NOT redirected
```
**Root cause:** `>` redirects only **fd 1 (stdout)**. Errors go to **fd 2 (stderr)**, a *different* file descriptor. `/dev/null` here is just a normal target file; redirection is fd rewiring (Topic 12).
**Right:**
```bash
$ noisy-command > /dev/null 2>&1   # fd 1 → /dev/null, then fd 2 → wherever fd 1 goes
$ noisy-command &> /dev/null       # bash shorthand for the same
```
**Prevent:** remember stdout and stderr are *separate slots*; silencing one doesn't touch the other.

---

### Mistake 5 — Editing the target of a "current" deploy symlink instead of flipping it

**Wrong:** `rm -rf /var/www/current && cp -r new-release /var/www/current` — a window where `current` doesn't exist and nginx 502s; a slow copy that serves half-written files.
**Right:** extract to a *new* release dir, then atomically repoint the symlink:
```bash
$ ln -sfn /var/www/releases/2026-07-12-b/  /var/www/current
#     ││└── n = treat existing 'current' as a normal file, don't descend into it
#     │└── f = force (replace the existing symlink)
#     └── s = symbolic
```
**Root cause:** replacing a symlink with `ln -sfn` is a single `rename()` on most systems → **atomic**. No moment where `current` is missing or partial. This is exactly how Capistrano and most deploy tools achieve zero-downtime.
**Prevent:** deploy by flipping a symlink, never by mutating the live directory.

---

## Hands-On Proof

```bash
# PROVE IT: the 7 type characters are real — survey your system
ls -l /dev/null /dev/sda /bin /etc /etc/hostname 2>/dev/null | cut -c1
# c   ← /dev/null  (character device)
# b   ← /dev/sda   (block device)
# l   ← /bin       (symlink)
# d   ← /etc       (directory)
# -   ← hostname   (regular file)

# PROVE IT: a directory's "size" is the size of its name→inode table
stat -c '%s bytes: %n' /etc /usr/bin
# 12288 bytes: /etc         ← bigger dir = bigger name table

# PROVE IT: hard link = shared inode + link count; symlink = own inode
echo hi > f ; ln f h ; ln -s f s
stat -c '%i %h %F %n' f h s
# 262180 2 regular file f    ← inode 262180, 2 links
# 262180 2 regular file h    ← SAME inode, that's why count is 2
# 262190 1 symbolic link s   ← different inode, its own thing
rm f h s

# PROVE IT: /dev/null discards, /dev/zero is infinite, /dev/urandom is random
echo "vanish" > /dev/null && echo "written to void, saw nothing"
head -c 8 /dev/zero    | xxd    # eight 00 bytes
head -c 8 /dev/urandom | xxd    # eight random bytes

# PROVE IT: write to a regular file and to /dev/null are the SAME syscall
strace -e trace=write sh -c 'echo hi > /tmp/x'      2>&1 | grep 'write(1'
strace -e trace=write sh -c 'echo hi > /dev/null'   2>&1 | grep 'write(1'
# Both:  write(1, "hi\n", 3) = 3     ← identical; only the driver behind fd 1 differs

# PROVE IT: a socket is a file type, and Docker/Postgres use them
ls -l /var/run/docker.sock /var/run/postgresql/.s.PGSQL.* 2>/dev/null
# srw-rw---- ... docker.sock          ← 's' = socket

# PROVE IT: you can read a live process's memory stats as a "file"
cat /proc/self/status | grep -E 'Name|VmRSS'
# Name:  cat
# VmRSS: 1180 kB          ← kernel-generated, no disk file exists

# PROVE IT: a named pipe blocks until both ends connect (one-liner demo)
mkfifo /tmp/p ; ( echo "through the pipe" > /tmp/p & ) ; cat /tmp/p ; rm /tmp/p
# through the pipe
```

---

## Practice Exercises

### Exercise 1 — Easy

Create one of every creatable file type and identify each by its `ls -l` type character:

```bash
mkdir /tmp/ex1 && cd /tmp/ex1
echo data > reg          # regular
mkdir dir                # directory
ln reg hardlink          # hard link
ln -s reg symlink        # symlink
mkfifo fifo              # named pipe
ls -li
file *
```

**Questions:** Which two entries share an inode number, and why? Which entry has a size equal to the length of a path string? What is the link count on `reg`, and what will it become if you `rm hardlink`?

---

### Exercise 2 — Medium

Demonstrate the difference between deleting a hard-linked file and a symlinked file:

```bash
cd /tmp/ex1
# For the HARD link:
rm reg              # remove the "original"
cat hardlink        # ??? — does the data survive?
stat -c '%h' hardlink   # what is the link count now?

# For the SYMLINK (recreate first):
echo data > reg
ln -s reg sym2
rm reg              # remove the target
cat sym2            # ??? — what happens now?
readlink sym2       # what does the symlink still contain?
```

**Deliverable:** Explain in one paragraph, using the words *inode*, *link count*, *directory entry*, and *path string*, why one survived deletion of the "original" and the other broke.

---

### Exercise 3 — Hard (Production Simulation)

Reproduce and correctly fix the deleted-but-open disk-space bug:

```bash
# 1. Start a writer that holds a log open and grows it
( while true; do echo "$(date) log line padding padding padding" ; sleep 0.005; done > /tmp/growing.log ) &
WRITER=$!

# 2. Let it grow, confirm it has size
sleep 5 ; ls -lh /tmp/growing.log

# 3. Delete it the WRONG way while the writer runs
rm /tmp/growing.log
ls -lh /tmp/growing.log     # gone from the listing...
du -sh /tmp/growing.log 2>/dev/null   # du can't find it

# 4. But the space is NOT freed. Prove it:
sudo lsof +L1 | grep growing        # find the deleted-but-open file & its PID/fd
ls -l /proc/$WRITER/fd/              # find which fd points at "(deleted)"

# 5. Reclaim the space WITHOUT killing the writer, via /proc:
sudo truncate -s 0 /proc/$WRITER/fd/1   # (use the correct fd number you found)

# 6. Then stop the writer
kill $WRITER
```

**Deliverable:** Explain what `lsof +L1` showed, why `du` and `df` would have disagreed here, and how `/proc/PID/fd/N` — itself a symlink — let you reach a file that no longer has a name. Tie it back to "everything is a file."

---

## Mental Model Checkpoint

Answer from memory:

1. **What does "everything is a file" actually mean?** (Hint: it's really about a uniform *interface*, not about disks.)
2. **Name the 7 file types and the `ls -l` character for each.**
3. **What, physically, is a directory?** What does it contain, and where does a filename actually live?
4. **State the three hard differences between a hard link and a symlink.** Which can cross filesystems, and why can't the other?
5. **You `rm` a 40GB file the app is writing to and `df` doesn't change. Explain why, and give the one command that frees the space without a restart.**
6. **What do the two numbers `1, 3` in `ls -l /dev/null` mean?** How do character and block devices differ?
7. **Why does Docker mount `/var/run/docker.sock` into some containers, and what is the security risk?**
8. **Why is `echo x > file.txt` and `echo x > /dev/null` the same system call from the program's point of view?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `ls -l` | Show file type (1st char), perms, link count | `-i` (inode #), `-a` (hidden), `-L` (follow symlinks), `-S` (by size) |
| `file` | Identify true type via magic bytes | `-b` (no filename), `-L` (follow links) |
| `stat` | Dump inode metadata | `-c '%F %i %h %n'` (type, inode, links, name), `-f` (fs stats) |
| `ln` | Create a **hard** link (shared inode) | (no flag = hard link) |
| `ln -s` | Create a **symbolic** link (path string) | `-f` (replace existing), `-n` (don't follow existing dir link), `-sfn` combo for deploys |
| `readlink` | Print a symlink's target | `-f` (resolve full chain to real path) |
| `mkfifo` | Create a named pipe (FIFO) | `-m MODE` (set permissions) |
| `mknod` | Create a device/fifo node | `c`/`b`/`p` + major minor (needs root) |
| `lsof` | List open files | `+L1` (deleted-but-open), `-p PID`, `-i :PORT`, `+D DIR` |
| `truncate` | Set a file's size (0 to empty it in place) | `-s 0` (empty), `-s 10M` (grow/shrink) |
| `lsblk` | Tree of block devices & mount points | `-f` (filesystems), `-o` (custom cols) |

---

## When Would I Use This at Work?

### Scenario 1: Disk full, but `du` says there's plenty of space
`df` reports 100% while `du -sh /*` adds up to half the disk. You immediately suspect a deleted-but-open file, run `sudo lsof +L1`, find a `(deleted)` log still held by your Node process, and `truncate -s 0 /proc/PID/fd/N` to reclaim the space with zero downtime. Without the "filename is just a link to an inode" model, this bug is unsolvable and you'd reboot the box in a panic.

### Scenario 2: Setting up zero-downtime deploys
You design releases as timestamped directories under `/var/www/releases/` with a `current` symlink that nginx points at. Deploys become an atomic `ln -sfn` symlink flip; rollbacks are re-flipping to the previous release. You understand exactly why this is safe (a symlink stores a path, repointing is a single atomic `rename()`) and why hard links couldn't do it (can't span directories cleanly, can't dangle-and-repoint).

### Scenario 3: Wiring a container to the Docker daemon (and knowing the risk)
Your CI runner or Traefik needs to manage containers, so you mount `/var/run/docker.sock` into it. Because you know a socket is a file and this socket is a control channel to a root daemon, you understand you've just granted that container host-root-equivalent power — and you gate it accordingly (read-only where possible, a socket proxy, or a rootless daemon). This is the difference between using the socket and getting owned by it.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | The VFS layer and the syscall boundary are what make one interface span every resource |
| **Builds on** | 02 — Filesystem Hierarchy | You met `/dev`, `/proc`, sockets, and symlinks there; here you learn what they *are* |
| **Next** | 04 — Inodes in Depth | "A directory maps names to inodes" and "a hard link is a shared inode" both demand you now understand the inode itself, in full |
| **Used by** | 05 — File Permissions | Permission bits, the type char, and setuid/sticky all live in the inode's mode field |
| **Used by** | 12 — Pipes and Redirection | `>`, `<`, `\|`, `2>&1`, `/dev/null` are all just file-descriptor rewiring — this topic is the foundation |
| **Used by** | 19 — File Descriptors | `/proc/PID/fd`, `lsof`, and the deleted-but-open trick are the everyday payoff of this abstraction |
| **Used by** | 28 — Ports and Sockets | Unix domain sockets vs TCP sockets — same API, one is a file path, one is an address:port |
