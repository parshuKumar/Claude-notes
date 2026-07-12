# 02 — The Filesystem Hierarchy

## ELI5 — The Simple Analogy

Imagine a **hospital**.

You walk in the front door. There is exactly ONE front door — the entrance (`/`). Everything in the building is reachable from that entrance. There is no "second hospital" you have to drive to.

Inside, every room has a **purpose you can guess from the sign on the door**:

- **Surgical Tools** (`/bin`, `/usr/bin`) — the instruments. Scalpels, clamps. Anyone on staff can pick one up and use it. They rarely change.
- **The Chief Surgeon's Locked Cabinet** (`/sbin`) — the tools only a senior doctor is allowed to touch. Defibrillators. Anaesthesia.
- **The Records Office** (`/etc`) — patient charts, staffing rosters, procedure manuals. All **paper**. All human-readable. Nobody stores a scalpel here.
- **The Wards** (`/var`) — where things **change constantly**. Patients come and go. Charts get updated hourly. This is the messy, growing, living part.
- **Staff Lockers** (`/home`) — each nurse gets their own locker. They can't open each other's.
- **The Chief's Office** (`/root`) — the boss's private office. Not in the staff locker room. Its own door.
- **The Bins** (`/tmp`) — emptied every night by the cleaners. Never leave anything valuable here.
- **The Window into the Operating Theatre** (`/proc`) — a live glass window showing exactly what the surgeons are doing RIGHT NOW. Nothing is stored behind the glass. It's a view, not a room.
- **The Machine Panel** (`/dev`) — every plug and socket in the building, exposed as labelled switches. The heart monitor. The oxygen line. The bin chute you throw things into and they vanish forever.

The building is one continuous space, but different **wings were built by different contractors** and physically bolted onto the main structure (mount points). You walk from one to the other without noticing.

Linux is that hospital. **One entrance. One tree. Every room has a reason.**

---

## Where This Lives in the Linux Stack

```
Hardware (Disk platters / SSD NAND / NVMe blocks)
  └── KERNEL
       │    • VFS (Virtual File System) layer ◀◀◀ THIS TOPIC LIVES HERE
       │      Presents ONE unified tree ("/") no matter how many
       │      physical disks, partitions, or fake filesystems exist.
       │    • Filesystem drivers: ext4, xfs, btrfs, tmpfs, procfs, sysfs, overlayfs
       │    • Mount table: which filesystem is glued at which directory
       │
       └── System Calls (open, openat, stat, getdents64, statfs, mount)
            │
            └── C Standard Library (glibc — opendir(), readdir(), fopen())
                 │
                 └── Shell (bash — resolves paths, expands globs, has a $PWD)
                      │
                      └── Commands (ls, df, du, findmnt, tree, stat, file)
                           ◀◀◀ AND HERE — the tools you use to explore it
```

**The key insight:** The directory tree is a **kernel-maintained illusion**. The VFS layer takes N different filesystems on M different devices and presents them as one seamless namespace starting at `/`.

---

## What Is This?

The **Filesystem Hierarchy Standard (FHS)** is the agreed-upon convention for what goes in which directory on a Linux system. It is maintained by the Linux Foundation and is why `/etc` means "config" on Ubuntu, Debian, RHEL, Alpine, and inside every Docker container.

Linux has **one single tree rooted at `/`**. There is no `C:` drive. A second hard disk doesn't become `D:` — it gets **mounted** into the existing tree at a directory like `/mnt/data`, and from then on it's just... part of the tree.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Where logs live (`/var/log`) | Your disk fills to 100%, your Node app can't write, and you have no idea what ate 40GB |
| `/etc` is config, never binaries | You'll put your app's `.env` in a random place and your systemd unit won't find it |
| `/tmp` is wiped on reboot | You'll store uploaded files there and lose customer data on the next reboot |
| `/proc` is virtual (not on disk) | You'll panic that `/proc/kcore` is "128 TB" and think your disk is broken |
| Mount points | You'll run `df -h` on `/` see 20% used, while `/var` is a separate 100%-full volume |
| `/dev/shm` and tmpfs | Your container OOMs because "disk" writes were secretly consuming RAM |
| `/usr/local/bin` vs `/usr/bin` | Your `node` upgrade "doesn't take effect" because PATH picks the wrong binary |

---

## The Physical Reality

The tree is NOT one disk. It is assembled at boot from many sources:

```
                              /  ◀── the root of everything
                              │      (mounted from /dev/sda2, filesystem: ext4)
        ┌────────────┬────────┼────────┬─────────┬──────────┬────────┐
        │            │        │        │         │          │        │
      /boot        /home    /etc     /var      /proc      /sys     /dev
        │            │        │        │         │          │        │
   REAL DISK    REAL DISK  (part of  REAL DISK  VIRTUAL   VIRTUAL  VIRTUAL
   /dev/sda1   /dev/sdb1    root fs) /dev/sda3  procfs    sysfs    devtmpfs
   (ext4)       (ext4)                (xfs)     (RAM)     (RAM)    (RAM)
    512MB        500GB                 100GB     0 bytes   0 bytes  0 bytes
                                                  on disk   on disk  on disk
```

**Three categories of filesystem live in this tree:**

| Category | Examples | Backed by | Survives reboot? |
|---|---|---|---|
| **Real / on-disk** | `/`, `/home`, `/var`, `/boot` | ext4/xfs on a block device | Yes |
| **Virtual / kernel-generated** | `/proc`, `/sys` | Nothing. Generated on read. | N/A — regenerated |
| **In-memory (tmpfs)** | `/run`, `/dev/shm`, often `/tmp` | RAM (+ swap) | **No** |

This is why:

```bash
$ df -h /proc
Filesystem      Size  Used Avail Use% Mounted on
proc               0     0     0    - /proc
#               ^^^^ zero. It has no size. It is not a thing on a disk.
```

**A file in `/proc` is a function call wearing a file costume.** When you `cat /proc/meminfo`, the kernel does not read a disk. It runs a function that formats memory stats into text and hands them to you through the `read()` syscall — the same syscall you'd use on a real file. (Topic 03 — Everything Is a File explains exactly how this works.)

---

## How It Works — Step by Step

### What happens when the kernel assembles the tree at boot

```
Step 1: BOOTLOADER (GRUB) reads /boot/grub/grub.cfg from the /boot partition
        Loads /boot/vmlinuz-5.15.0-72-generic  (the compressed kernel)
        Loads /boot/initrd.img-5.15.0-72-generic (a tiny temporary root fs in RAM)

Step 2: KERNEL unpacks the initrd into RAM. This becomes a TEMPORARY "/".
        Why? The kernel might need drivers (e.g. LVM, RAID, encrypted disk)
        just to READ the real root filesystem. Chicken-and-egg — initrd solves it.

Step 3: KERNEL (using initrd's tools) finds the real root device.
        Reads the kernel command line: root=UUID=8f3c...  rw
        Loads the ext4 driver, mounts /dev/sda2 at "/".

Step 4: KERNEL performs pivot_root — throws away the initrd, the real disk is now "/".

Step 5: KERNEL exec()s /sbin/init → which is a symlink to /lib/systemd/systemd.
        This becomes PID 1. (Remember from 01 — the kernel starts exactly one
        userspace process; everything else is forked from it.)

Step 6: systemd reads /etc/fstab and mounts everything else:
        /dev/sda1 → /boot
        /dev/sda3 → /var
        proc      → /proc   (type procfs)
        sysfs     → /sys    (type sysfs)
        tmpfs     → /run    (type tmpfs, in RAM)
        devtmpfs  → /dev

Step 7: The tree is now complete. Every path you type resolves through it.
```

### What happens when you type a path

```
COMMAND: cat /var/log/nginx/access.log

SHELL DOES:    forks, execs /usr/bin/cat with argv[1] = "/var/log/nginx/access.log"
KERNEL DOES:   Path resolution, one component at a time, starting from the ROOT INODE:
                 "/"      → root inode of the ext4 fs on /dev/sda2
                 "var"    → look up "var" in root's directory data → inode 131074
                            ...but wait: inode 131074 is a MOUNT POINT.
                            Kernel checks the mount table, finds /dev/sda3 (xfs)
                            mounted here, and JUMPS to the root inode of THAT fs.
                 "log"    → look up in /var's (xfs) directory data → inode 96
                 "nginx"  → look up → inode 4021
                 "access.log" → look up → inode 4099
               Checks execute (x) permission on every directory along the way.
               Checks read (r) permission on the final file.
SYSCALLS:      openat(AT_FDCWD, "/var/log/nginx/access.log", O_RDONLY) = 3
               fstat(3, ...) → learns file size, so it can pick a buffer size
               read(3, buf, 131072) → repeated until it returns 0
               write(1, buf, n)     → to stdout
               close(3)
OUTPUT:        the log text on your terminal
```

**That "JUMPS to another filesystem" step is the whole magic of a unified tree.** You typed one path; the kernel silently crossed a filesystem boundary mid-path and you never noticed.

---

## Every Top-Level Directory — What and Why

### The "programs" directories

| Dir | Contains | Why it exists |
|---|---|---|
| `/bin` | Essential user binaries: `ls`, `cp`, `cat`, `bash` | Historically: commands needed **before** `/usr` was mounted. Today: symlink to `/usr/bin`. |
| `/sbin` | Essential **system** binaries: `fdisk`, `ip`, `mount`, `iptables` | Root-oriented tools. The `s` = "system"/"superuser". Still executable by anyone, but usually useless without root. |
| `/usr/bin` | The bulk of userland: `node`, `python3`, `git`, `grep`, `awk` | Everything installed by the **package manager**. On Ubuntu, `apt install nodejs` → `/usr/bin/node`. |
| `/usr/sbin` | Non-essential system binaries: `nginx`, `sshd`, `useradd` | Daemons and admin tools installed by packages. |
| `/usr/local/bin` | Software **you** compiled/installed manually | The package manager NEVER touches this. `nvm`, `make install`, downloaded binaries. |
| `/usr/lib` | Shared libraries (`.so` files) for `/usr/bin` programs | `libssl.so`, `libc.so.6` — what `ldd` shows you. |
| `/usr/share` | Architecture-**independent** data: man pages, icons, timezone data, docs | Same on x86 and ARM. `/usr/share/man`, `/usr/share/zoneinfo`. |
| `/lib`, `/lib64` | Core shared libraries + kernel modules (`/lib/modules`) | Symlinks to `/usr/lib` on modern distros. |

**PATH order is why this matters:**

```bash
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
 ─────────────────────────────  ◀── searched FIRST
```

If you install Node via a tarball into `/usr/local/bin/node` (v20) while `apt` already put v18 in `/usr/bin/node`, **you get v20** — because `/usr/local/bin` comes first. This single fact explains 90% of "why is `node -v` showing the wrong version?"

```bash
# PROVE IT: which binary actually wins
$ which -a node
/usr/local/bin/node     ← this one runs
/usr/bin/node           ← this one is shadowed
```

### `/etc` — Configuration. Text. Never binaries.

The name is a historical accident ("et cetera" — the junk drawer of early Unix). Today the rule is absolute: **`/etc` holds system-wide configuration, in human-readable text, and nothing executable that isn't a script.**

```
/etc/
├── passwd              user accounts        (Topic 06)
├── shadow              password hashes      (Topic 06 — mode 640, root only)
├── group               groups               (Topic 06)
├── hostname            "prod-api-01"
├── hosts               static DNS overrides (Topic 27)
├── resolv.conf         DNS servers          (Topic 27)
├── fstab               what to mount where at boot ◀── THIS TOPIC
├── crontab             system-wide cron     (Topic 20)
├── os-release          distro identity      (you used this in Topic 01)
├── nginx/
│   ├── nginx.conf
│   └── sites-enabled/api.conf     ← your reverse proxy to localhost:3000
├── systemd/system/
│   └── my-api.service             ← YOUR Node app's unit file (Topic 22)
├── ssh/sshd_config     SSH server config    (Topic 24)
└── apt/sources.list    package repos        (Topic 21)
```

**Why text?** Because at 2am, over a flaky SSH connection, on a broken server, you can fix a text file with `vi`. You cannot fix a binary registry.

**Why no binaries?** Because `/etc` is what you back up, diff, and put in git. Config is state you *author*. Binaries are artifacts you *install*.

### `/var` — Data that CHANGES

`var` = "variable". If a file's size or content changes during normal operation, it belongs here.

| Path | Contains | Why you care |
|---|---|---|
| `/var/log` | **All logs.** `syslog`, `auth.log`, `nginx/access.log`, your app's logs | The #1 cause of a full disk. Topic 23. |
| `/var/lib` | Persistent state owned by services | `/var/lib/postgresql` (your DB!), `/var/lib/docker` (all images & containers), `/var/lib/apt` |
| `/var/cache` | Regenerable cache. Safe to delete. | `/var/cache/apt/archives` — downloaded `.deb` files. `apt clean` empties it. |
| `/var/tmp` | Temp files that **survive reboot** (unlike `/tmp`) | Long-running jobs use this. Cleaned after ~30 days, not on boot. |
| `/var/www` | Web content served by nginx/Apache | Classic location for your deployed app. |
| `/var/spool` | Queues: mail, print, cron jobs waiting to run | `/var/spool/cron/crontabs/deploy` — where your user crontab actually lives. |
| `/var/run` | **Legacy.** Now a symlink → `/run` | PID files, socket files. See below. |

```bash
# PROVE IT: /var/run is not a real directory anymore
$ ls -ld /var/run
lrwxrwxrwx 1 root root 4 Apr 18 2022 /var/run -> /run
#^ the leading "l" = symlink. (Topic 03 covers exactly what that means.)
```

**Why the move?** `/var` may live on a disk that isn't mounted early in boot. But daemons need somewhere to drop PID files immediately. `/run` is a **tmpfs mounted before anything else**, always available, always empty at boot. Stale PID files from a crash can't survive a reboot — which is exactly what you want.

### `/home` vs `/root`

```
/home/deploy/        ← your deploy user. UID 1001.
/home/parshuram/     ← you. UID 1000.
/root/               ← root's home. UID 0. NOT /home/root.
```

Why is root's home not in `/home`? Because `/home` is frequently a **separate partition or an NFS mount**. If it fails to mount, root must still be able to log in and fix the machine. Root's home is on the root filesystem, always available.

### `/tmp` — the bin that gets emptied

```bash
$ findmnt /tmp
TARGET SOURCE FSTYPE OPTIONS
/tmp   tmpfs  tmpfs  rw,nosuid,nodev,size=3987404k
#             ^^^^^ IN RAM. Not on your disk.
```

On most modern systems `/tmp` is **tmpfs** — it lives in RAM (and swap). Consequences:

1. **It is wiped on every reboot.** Never store anything you need.
2. **Writing there consumes RAM.** A 5GB upload to `/tmp` in a 2GB container = OOM kill.
3. **It's fast.** Genuinely the fastest scratch space you have.
4. It's **world-writable with a sticky bit** (`drwxrwxrwt`) — everyone can create files, but you can only delete your OWN. That trailing `t` is the sticky bit, covered in depth in Topic 05 — File Permissions.

```bash
$ ls -ld /tmp
drwxrwxrwt 15 root root 380 Jul 12 09:14 /tmp
#        ^ sticky bit — the reason a random user can't delete your temp files
```

### `/opt` — self-contained third-party software

`/opt/appname/` holds an entire application in one directory tree — its own `bin/`, `lib/`, `conf/`. Used by vendors who don't want to scatter files across FHS locations. `/opt/google/chrome`, `/opt/datadog-agent`. Many teams deploy their Node app to `/opt/myapp` for exactly this reason: **delete the directory, the app is gone.**

### `/proc` — the kernel exposing itself as files

Not on disk. Zero bytes. Generated on read.

```
/proc/
├── cpuinfo          every core, model, MHz, flags
├── meminfo          every byte of RAM accounted for
├── loadavg          the 3 load numbers you see in `uptime`
├── uptime           seconds since boot
├── version          kernel version string
├── mounts           the live mount table (what's mounted where)
├── cmdline          the kernel boot parameters
├── self/            → symlink to the PID of the process READING it (!)
├── 1/               PID 1 (systemd)
├── 1847/            PID 1847 — your node process
│   ├── cmdline      "node\0server.js\0"
│   ├── environ      ALL its environment variables (Topic 14)
│   ├── cwd  →       symlink to its current working directory
│   ├── exe  →       symlink to /usr/bin/node
│   ├── fd/          every open file descriptor (Topic 19)
│   ├── status       memory usage, UID, threads, state
│   └── limits       ulimits in effect
└── sys/             TUNABLE kernel parameters — you can WRITE here
    └── net/ipv4/ip_local_port_range
```

**`/proc/PID/` is the single most valuable debugging surface on Linux.** If a Node process is misbehaving and you can't attach a debugger, `/proc/<pid>/` tells you its memory, its open files, its env vars, and its working directory — with nothing but `cat`.

### `/sys` — sysfs, the device model

Where `/proc` is a historical grab-bag, `/sys` is the **structured, modern** view of the kernel's device tree: every bus, device, driver, and kernel object.

```
/sys/class/net/eth0/        ← your network interface (Topic 26)
│   ├── address             MAC address
│   ├── mtu                 1500
│   ├── operstate           "up"
│   └── statistics/rx_bytes total bytes received
/sys/block/sda/             ← the physical disk
│   ├── size                sectors
│   └── queue/rotational    1 = spinning disk, 0 = SSD
/sys/fs/cgroup/             ← cgroups: how Docker & systemd LIMIT resources
    └── memory.max          your container's RAM ceiling
```

```bash
# PROVE IT: is your server on an SSD or a spinning disk?
$ cat /sys/block/sda/queue/rotational
0    # 0 = SSD, 1 = HDD

# PROVE IT: your container's memory limit (cgroup v2, inside Docker)
$ cat /sys/fs/cgroup/memory.max
536870912    # 512MB. Exceed this and the kernel OOM-kills your node process.
```

### `/dev` — devices as files

Every piece of hardware, exposed as a file you can `open()`, `read()`, and `write()`.

| Device | What it is |
|---|---|
| `/dev/null` | The bit bucket. Writes vanish. Reads return EOF instantly. |
| `/dev/zero` | Infinite stream of `\0` bytes. Used to allocate/zero memory or files. |
| `/dev/random`, `/dev/urandom` | Cryptographic randomness from the kernel's entropy pool. `crypto.randomBytes()` in Node ultimately comes from here. |
| `/dev/sda` | The whole first SATA/SCSI disk. |
| `/dev/sda1` | The **first partition** of that disk. |
| `/dev/nvme0n1p1` | NVMe naming: controller 0, namespace 1, partition 1. |
| `/dev/tty`, `/dev/pts/0` | Your terminal. Yes — your terminal is a file. |
| `/dev/shm` | Shared-memory tmpfs. Chrome/Puppeteer in Docker crashes here (default 64MB). |

Topic 03 — Everything Is a File dissects these in full, including major/minor numbers.

### The rest

| Dir | Purpose |
|---|---|
| `/boot` | Kernel (`vmlinuz-*`), initramfs (`initrd.img-*`), bootloader config. Often a small separate partition. **Fills up** with old kernels — a classic Ubuntu failure. |
| `/mnt` | A place for **you** (the sysadmin) to temporarily mount something manually. |
| `/media` | Where the system **automatically** mounts removable media (USB sticks, CDs). Rare on servers. |
| `/srv` | Data **served** by this machine (`/srv/www`, `/srv/ftp`). FHS-blessed but rarely used; most people use `/var/www` or `/opt`. |
| `/run` | tmpfs. PID files, sockets, runtime state. Empty at every boot. |
| `/lost+found` | ext4 puts orphaned inodes here after `fsck` recovers a corrupted filesystem. One per filesystem. |

---

## The /bin → /usr/bin Merge (the "usr-merge")

On Ubuntu 20.04+, Debian 11+, Fedora, and most modern distros:

```bash
$ ls -ld /bin /sbin /lib
lrwxrwxrwx 1 root root 7 Apr 18  2022 /bin  -> usr/bin
lrwxrwxrwx 1 root root 8 Apr 18  2022 /sbin -> usr/sbin
lrwxrwxrwx 1 root root 7 Apr 18  2022 /lib  -> usr/lib
```

**They are all symlinks now.** `/bin/ls` and `/usr/bin/ls` are the *same file*, reached by two paths.

**Why the split ever existed:** In 1970s Unix, the root disk was tiny. `/usr` was a *second disk*. Boot-critical binaries had to be on the root disk (`/bin`), everything else went on the big disk (`/usr/bin`). That constraint died decades ago; the initramfs now handles early boot.

**Why you should care:** A `#!/bin/bash` shebang and a `#!/usr/bin/bash` shebang resolve to the same binary on Ubuntu — but **NOT on Alpine**, where `bash` may not exist at all (it's `/bin/sh` → BusyBox `ash`). This is the #1 reason a working script fails inside a slim Docker image.

---

## Where a Node.js App's Pieces Actually Live in Production

This is the map you will use every single day:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  THE RUNTIME                                                              │
│  /usr/bin/node                    the Node binary (apt/nodesource)        │
│  /usr/local/bin/node              if installed via nvm/tarball — WINS PATH│
│  /usr/lib/node_modules/           globally installed npm packages (pm2)   │
├──────────────────────────────────────────────────────────────────────────┤
│  THE APP                                                                  │
│  /var/www/api/                    your deployed code (server.js, package) │
│  /opt/api/                        ...or here, if you prefer self-contained│
│  /var/www/api/node_modules/       app dependencies                        │
├──────────────────────────────────────────────────────────────────────────┤
│  CONFIGURATION                                                            │
│  /etc/systemd/system/api.service  the systemd unit that starts your app   │
│  /etc/nginx/sites-enabled/api     reverse proxy → http://127.0.0.1:3000   │
│  /etc/api/api.env                 secrets/env file (mode 600, owner deploy)│
├──────────────────────────────────────────────────────────────────────────┤
│  RUNTIME STATE (changes constantly)                                       │
│  /var/log/nginx/access.log        every HTTP request                      │
│  /var/log/nginx/error.log         nginx's 502s when your app is down      │
│  /var/log/syslog                  systemd's view of your app's stdout     │
│  /run/api.pid                     the PID file (gone after reboot)        │
│  /run/postgresql/.s.PGSQL.5432    the DB's unix socket                    │
│  /var/lib/postgresql/14/main/     the actual database files               │
│  /var/lib/docker/                 every image layer & container           │
├──────────────────────────────────────────────────────────────────────────┤
│  SCRATCH                                                                  │
│  /tmp/                            tmpfs — WIPED ON REBOOT, EATS RAM       │
└──────────────────────────────────────────────────────────────────────────┘
```

**Read that table twice.** When someone says "the API is 502ing", you now know to look at `/var/log/nginx/error.log`, then `journalctl -u api`, then `/etc/api/api.env` — in that order — without thinking.

---

## Exact Syntax Breakdown

```
df -h /var
│  │  │
│  │  └── the path to report on (df reports the FILESYSTEM containing it)
│  └── -h = --human-readable — 1.0K, 234M, 2.1G instead of raw 1K-blocks
└── df = "disk free" — reports per-FILESYSTEM usage by asking statfs()

Output:
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        98G   91G  2.1G  98% /var
│                │     │    │     │    │
│                │     │    │     │    └── the mount point in the tree
│                │     │    │     └── percentage — >90% is a fire
│                │     │    └── space left (note: 5% is reserved for root by default)
│                │     └── space used
│                └── total capacity
└── the BLOCK DEVICE backing this part of the tree
```

```
du -sh /var/log
│  ││  │
│  ││  └── the directory to measure
│  │└── -h = human-readable
│  └── -s = --summarize — one total, don't list every subdirectory
└── du = "disk usage" — WALKS the tree and stat()s every file. Slow but exact.

    df asks the filesystem "how full are you?"   → instant
    du asks every file "how big are you?"        → slow, but tells you WHY
```

```
du -h --max-depth=1 /var | sort -rh | head -10
│  │   │             │    │  │  │     │
│  │   │             │    │  │  │     └── show only the top 10
│  │   │             │    │  │  └── -h = sort human-readable sizes correctly (2G > 900M)
│  │   │             │    │  └── -r = reverse — biggest first
│  │   │             │    └── pipe into sort (Topic 12)
│  │   │             └── the directory to break down
│  │   └── only descend ONE level — show me /var/log, /var/lib, not every leaf
│  └── human-readable
└── THIS IS THE "WHAT ATE MY DISK" COMMAND. Memorize it.
```

```
findmnt /var/log
│       │
│       └── optional: show only the filesystem containing this path
└── findmnt — pretty-prints the mount table as a TREE. Reads /proc/self/mountinfo.
              Strictly better than bare `mount` for humans.

Output:
TARGET   SOURCE    FSTYPE OPTIONS
/var     /dev/sda3 xfs    rw,relatime
│        │         │      │
│        │         │      └── mount options (rw, noexec, nosuid, nodev...)
│        │         └── the filesystem TYPE — tells you if it's real or virtual
│        └── what is mounted (device, or "tmpfs"/"proc" for virtual)
└── WHERE in the tree it's glued on
```

```
stat /var/log/syslog
│    │
│    └── file to inspect
└── stat — calls the stat() syscall and dumps the raw INODE metadata

Output:
  File: /var/log/syslog
  Size: 4823901    Blocks: 9424    IO Block: 4096   regular file
  Device: 803h/2051d  Inode: 262147  Links: 1
  │                   │              │
  │                   │              └── hard link count (Topic 03 & 04)
  │                   └── the inode number — the file's TRUE identity (Topic 04)
  └── which DEVICE (filesystem) it's on — proves whether you crossed a mount
Access: (0640/-rw-r-----)  Uid: (0/root)  Gid: (4/adm)
Modify: 2026-07-12 09:14:02.481920331 +0000
```

```
file /usr/bin/node
│    │
│    └── the file to identify
└── file — reads the first bytes ("magic number") and reports the REAL type,
           ignoring the extension entirely. Linux does not care about extensions.

Output: /usr/bin/node: ELF 64-bit LSB pie executable, x86-64, dynamically linked
```

```
man hier
│   │
│   └── "hierarchy" — the manual page describing the ENTIRE FHS
└── the manual. `man hier` is the authoritative, offline, always-available
    version of this whole document. Read it on any server you're ever confused on.
```

---

## Example 1 — Basic: Walk the Tree

```bash
# See the top level. -F appends / to dirs, @ to symlinks — instantly informative.
ls -F /
# bin@  boot/  dev/  etc/  home/  lib@  media/  mnt/  opt/  proc/  root/
# run/  sbin@  srv/  sys/  tmp/  usr/  var/
#  ^ note bin, lib, sbin have @ — they are SYMLINKS (usr-merge)

# Prove /bin is a symlink and where it points
ls -ld /bin
# lrwxrwxrwx 1 root root 7 Apr 18 2022 /bin -> usr/bin

readlink -f /bin/ls
# /usr/bin/ls        ← -f resolves ALL symlinks to the final real path

# Which filesystems make up your tree?
findmnt -t ext4,xfs,tmpfs,proc,sysfs
# TARGET  SOURCE     FSTYPE OPTIONS
# /       /dev/sda2  ext4   rw,relatime
# ├─/proc proc       proc   rw,nosuid,nodev,noexec
# ├─/sys  sysfs      sysfs  rw,nosuid,nodev,noexec
# ├─/run  tmpfs      tmpfs  rw,nosuid,nodev,size=798484k
# ├─/tmp  tmpfs      tmpfs  rw,nosuid,nodev
# └─/var  /dev/sda3  xfs    rw,relatime

# How full is each one?
df -h -x tmpfs -x devtmpfs      # -x excludes a fs type from the report
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda2        49G   12G   35G  26% /
# /dev/sda3        98G   14G   84G  15% /var
# /dev/sda1       511M   96M  415M  19% /boot

# What is /usr/bin/node, really?
file /usr/bin/node
# /usr/bin/node: ELF 64-bit LSB pie executable, x86-64, dynamically linked

# The kernel's own description of a running process
cat /proc/self/status | head -6
# Name:   cat            ← the process reading this file IS cat
# Umask:  0022
# State:  R (running)
# Tgid:   28431
# Pid:    28431
```

---

## Example 2 — Production Scenario

**Situation:** It's 02:14. PagerDuty fired. Your Node API at `api.example.com` is returning `502 Bad Gateway`. You SSH in.

```bash
# Step 1 — The classic 2am killer: is the disk full?
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        49G   11G   36G  24% /
/dev/sda3        98G   98G     0 100% /var        ◀◀◀ FOUND IT
tmpfs           798M  1.2M  797M   1% /run
/dev/sda1       511M   96M  415M  19% /boot

# NOTE what just happened: "/" looks FINE at 24%. If you had only looked at "/",
# you would have concluded the disk was healthy. /var is a SEPARATE FILESYSTEM.
# This is why you must understand mount points.

# Step 2 — What inside /var ate 98GB? One level at a time.
$ sudo du -h --max-depth=1 /var | sort -rh | head
98G     /var
71G     /var/log        ◀◀◀
24G     /var/lib
2.1G    /var/cache
180M    /var/www

# Step 3 — Drill into /var/log
$ sudo du -h --max-depth=1 /var/log | sort -rh | head -5
71G     /var/log
68G     /var/log/nginx  ◀◀◀
1.9G    /var/log/journal
612M    /var/log/syslog

$ sudo ls -lhS /var/log/nginx | head -4
total 68G
-rw-r----- 1 www-data adm  67G Jul 12 02:14 error.log     ◀◀◀ 67 GIGABYTES
-rw-r----- 1 www-data adm 1.1G Jul 12 02:14 access.log

# Step 4 — WHY is error.log 67GB? Look at the tail.
$ sudo tail -2 /var/log/nginx/error.log
2026/07/12 02:14:01 [error] 981#981: *44219 connect() failed (111: Connection
refused) while connecting to upstream, client: 10.0.1.7, server: api.example.com,
request: "GET /health HTTP/1.1", upstream: "http://127.0.0.1:3000/health"

# Connection refused to 127.0.0.1:3000 → NOTHING IS LISTENING ON PORT 3000.
# nginx has been logging this error for every request, thousands per second,
# until it filled the disk. The full disk is a SYMPTOM. The app dying is the CAUSE.

# Step 5 — Is the app running?
$ systemctl status api
● api.service - Node API
     Loaded: loaded (/etc/systemd/system/api.service; enabled)
     Active: failed (Result: exit-code) since Fri 2026-07-11 23:48:10 UTC
   Main PID: 1847 (code=exited, status=1/FAILURE)

$ journalctl -u api -n 3 --no-pager
Jul 11 23:48:10 prod-api-01 node[1847]: Error: ENOSPC: no space left on device,
                                        write '/var/www/api/uploads/tmp-9f2.bin'

# Step 6 — Recover, in the right order.
$ sudo truncate -s 0 /var/log/nginx/error.log
#      ^^^^^^^^ truncate, do NOT `rm`. nginx holds an open FILE DESCRIPTOR to
#      this inode. If you rm it, the disk space is NOT freed until nginx is
#      restarted, because the inode's link count hits 0 but its OPEN COUNT
#      doesn't. (Topic 03 & Topic 19 — this is the single most misunderstood
#      thing about deleting files on Linux.)

$ df -h /var
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        98G   31G   67G  32% /var        ← space back instantly

$ sudo systemctl start api
$ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/health
200

# Step 7 — Prevent the recurrence.
$ cat /etc/logrotate.d/nginx    # Topic 23 — this SHOULD have been rotating.
# It was. But rotation runs DAILY via cron. 67GB accumulated in 3 hours.
# Real fix: cap it by size, not just by day.
```

**What you just used:** mount points (`df` on `/` lied), `/var/log` as the volatile-data location, `du --max-depth` to bisect, and the inode/file-descriptor distinction to free space without a restart. Every one of those is filesystem-hierarchy knowledge.

---

## Common Mistakes

### Mistake 1 — Trusting `df -h /` and declaring the disk healthy

**Wrong:**
```bash
$ df -h /
/dev/sda2  49G  11G  36G  24% /      # "plenty of space!"
```
**Right:**
```bash
$ df -h          # NO argument — show EVERY filesystem
$ df -hT         # -T also shows the TYPE, so you can spot tmpfs/overlay
```

**Root cause:** `df <path>` reports on the **single filesystem containing that path**. `/var` is very often a separate block device. The kernel's mount table has multiple entries; you only asked about one.

**Diagnose:** `findmnt` to see the whole tree, `df -h` with no args.
**Prevent:** Alert on *every* filesystem's usage, not on `/`.

---

### Mistake 2 — `rm` a huge logfile and the disk stays full

**Wrong:**
```bash
$ sudo rm /var/log/nginx/error.log
$ df -h /var
/dev/sda3  98G  98G  0  100% /var     # ...still full?!
```
**Right:**
```bash
$ sudo truncate -s 0 /var/log/nginx/error.log
# or, if you must remove it:
$ sudo rm /var/log/nginx/error.log && sudo systemctl reload nginx
```

**Root cause (kernel level):** `rm` calls `unlink()`, which removes the **directory entry** (name → inode mapping) and decrements the inode's link count to 0. But the inode — and its data blocks — are only freed when link count is 0 **AND no process has it open**. nginx still holds fd 5 pointing at that inode. The file is now nameless but very much alive, and its 67GB are still allocated.

**Diagnose:**
```bash
$ sudo lsof +L1        # +L1 = show files with link count < 1 (deleted but open)
COMMAND  PID     USER   FD   TYPE DEVICE   SIZE/OFF NLINK     NODE NAME
nginx    981 www-data    5w   REG    8,3 71943208960     0  4194561 /var/log/nginx/error.log (deleted)
#                                                        ^ NLINK 0 — it's a ghost
```
**Prevent:** Always `truncate -s 0`, or use `logrotate` with `copytruncate`. Fully explained in Topic 03 and Topic 19.

---

### Mistake 3 — Putting persistent data in `/tmp`

**Wrong:** Your Node upload handler writes to `/tmp/uploads/` and you serve files from there.

**Root cause:** `/tmp` is **tmpfs** on Ubuntu 22.04, Debian 12, and most cloud images. It is RAM. Two failure modes:
1. **Reboot** → every file is gone. Not "maybe gone". Gone. It was never on a disk.
2. **Large writes consume RAM.** A 3GB file in `/tmp` in a 4GB container means the kernel's OOM killer will terminate your Node process. `free -h` will show the memory as "shared".

**Diagnose:**
```bash
$ findmnt /tmp
TARGET SOURCE FSTYPE OPTIONS
/tmp   tmpfs  tmpfs  rw,nosuid,nodev,size=3987404k    ← IN RAM, capped at ~4GB

$ df -h /tmp                    # it has a "size" but it's RAM
$ free -h                       # watch the "shared" column grow as you write to /tmp
```
**Fix:** Persistent uploads → `/var/lib/myapp/uploads` or `/var/www/api/uploads`, on a real disk, owned by the deploy user.
**Prevent:** Set Node's `os.tmpdir()` explicitly via the `TMPDIR` env var if you need disk-backed temp space.

---

### Mistake 4 — Installing a binary into `/bin` or `/usr/bin` by hand

**Wrong:**
```bash
$ sudo cp ./mytool /usr/bin/mytool
```
**Right:**
```bash
$ sudo cp ./mytool /usr/local/bin/mytool
```

**Root cause:** `/usr/bin` is **owned by the package manager** (`dpkg`). Every file there is tracked in `/var/lib/dpkg/info/*.list`. When you drop an untracked file in, or worse, overwrite a tracked one, the next `apt upgrade` will either silently overwrite your binary or refuse to upgrade with a file-conflict error. `/usr/local` exists precisely so that local software and packaged software never collide.

**Diagnose:**
```bash
$ dpkg -S /usr/bin/node
nodejs: /usr/bin/node          # dpkg owns it — DO NOT touch by hand

$ dpkg -S /usr/local/bin/mytool
dpkg-query: no path found matching pattern /usr/local/bin/mytool   # safe: unowned
```
**Prevent:** Rule of thumb — **if `dpkg -S` claims it, don't touch it.**

---

### Mistake 5 — Assuming `/bin/bash` exists in your Docker image

**Wrong (Dockerfile / CI script):**
```bash
#!/bin/bash
set -euo pipefail
[[ "$NODE_ENV" == "production" ]] && echo "prod"    # [[ ]] is a bash builtin
```
Run in `node:18-alpine`:
```
/entrypoint.sh: line 3: [[: not found
```

**Root cause:** Alpine has **no bash**. `/bin/sh` is a symlink to `/bin/busybox`, which implements `ash` — a POSIX shell without `[[ ]]`, without arrays, without `pipefail` in older versions. The shebang `#!/bin/bash` fails with `exec format error` or `no such file or directory` (which confusingly refers to the *interpreter*, not your script).

**Diagnose:**
```bash
$ docker run --rm node:18-alpine ls -l /bin/sh
lrwxrwxrwx 1 root root 12 /bin/sh -> /bin/busybox

$ docker run --rm node:18-alpine which bash
# (no output — exit code 1)
```
**Fix:** Either `RUN apk add --no-cache bash`, or write POSIX-compliant `#!/bin/sh` scripts.
**Prevent:** Lint scripts with `shellcheck -s sh` if they will run on Alpine.

---

## Hands-On Proof

```bash
# PROVE IT: the tree is assembled from MULTIPLE filesystems, not one disk
findmnt
# A tree diagram. Note how /proc, /sys, /run, /dev have SOURCE = proc/sysfs/tmpfs
# — not a block device. They are not on any disk.

# PROVE IT: /proc has zero size but infinite content
df -h /proc
# Size 0, Used 0, Avail 0. And yet:
cat /proc/meminfo | head -3
# MemTotal:  8039728 kB   ← generated by the kernel, THIS INSTANT

# PROVE IT: /proc/self is a symlink to whichever process is reading it
ls -l /proc/self
# lrwxrwxrwx 1 root root 0 Jul 12 09:20 /proc/self -> 28733
readlink /proc/self ; readlink /proc/self
# 28734
# 28735    ← a DIFFERENT number each time — because each `readlink` is a new PID!

# PROVE IT: /bin and /usr/bin are the same directory (usr-merge)
stat -c '%i %n' /bin/ls /usr/bin/ls
# 1310892 /bin/ls
# 1310892 /usr/bin/ls     ← SAME INODE NUMBER. One file, two paths. (Topic 04)

# PROVE IT: /tmp is in RAM, not on your disk
findmnt /tmp
# FSTYPE = tmpfs  → RAM-backed
free -h                              # note the "shared" column
dd if=/dev/zero of=/tmp/big bs=1M count=200 2>/dev/null
free -h                              # "shared" grew by ~200M. You just used RAM.
rm /tmp/big                          # and now it's back

# PROVE IT: crossing a mount point changes the device number
stat -c '%D %n' / /var /proc /tmp
# 802 /
# 803 /var        ← DIFFERENT device — you crossed a filesystem boundary
# 0   /proc       ← device 0 — it's virtual
# 2f  /tmp        ← tmpfs has its own device number too

# PROVE IT: /var/run is a symlink to /run
readlink -f /var/run
# /run

# PROVE IT: /etc contains no compiled binaries
file /etc/* 2>/dev/null | grep -i "ELF" | head
# (no output — nothing in /etc is a compiled executable)

# PROVE IT: your terminal is literally a file in /dev
tty
# /dev/pts/0
echo "hello from a file write" > /dev/pts/0
# It appears on your screen — because writing to your terminal IS a file write.

# PROVE IT: the FHS is documented on every Linux box, offline
man hier
```

---

## Practice Exercises

### Exercise 1 — Easy

Map your own machine's tree. Run each and note the answer:

```bash
ls -F /
findmnt -t ext4,xfs,btrfs,tmpfs,overlay
df -hT
ls -ld /bin /sbin /lib /var/run
readlink -f /bin/sh
```

**Questions to answer from your output:** How many separate filesystems make up your tree? Which top-level directories are symlinks? Is `/tmp` on disk or in RAM? What does `/bin/sh` actually point to?

---

### Exercise 2 — Medium

Find what is consuming the most space on your system, from the top down:

```bash
# Start at the root and bisect. Repeat, descending into the biggest hit each time.
sudo du -h --max-depth=1 / 2>/dev/null | sort -rh | head -10
# Then pick the biggest and repeat:
sudo du -h --max-depth=1 /var 2>/dev/null | sort -rh | head -10
```

Keep descending until you find the single largest directory on your machine.

**Then:** explain (a) why `du` and `df` can disagree, and (b) why `2>/dev/null` is necessary here (hint: `/proc`, permission denied — Topic 12 covers this redirect).

---

### Exercise 3 — Hard (Production Simulation)

Simulate the 2am disk-full incident and recover from it correctly.

```bash
# 1. Create a directory structure that mimics a deployed app
sudo mkdir -p /opt/fakeapp/logs

# 2. Start a background process that HOLDS AN OPEN FILE DESCRIPTOR to a log
#    and keeps appending to it
sudo sh -c 'while true; do echo "$(date) ERROR upstream refused" >> /opt/fakeapp/logs/error.log; sleep 0.01; done' &

# 3. Let it run ~30 seconds, then check its size
ls -lh /opt/fakeapp/logs/error.log

# 4. Now DELETE it the WRONG way and observe
sudo rm /opt/fakeapp/logs/error.log
df -h /                       # did the space come back?
sudo lsof +L1 | head          # what does this tell you?

# 5. Kill the writer, and check df again. Explain what happened.
```

**Deliverable:** Explain, in terms of inodes, link counts, and open file descriptors, why the space did not return in step 4 but did in step 5 — and what you should have run instead of `rm`.

---

## Mental Model Checkpoint

Answer from memory:

1. **Why does Linux have one tree rooted at `/` instead of drive letters like `C:` and `D:`?** What does a second disk look like in the tree?
2. **What is the difference between `/etc`, `/var`, and `/usr` — stated as a rule about how often the data changes and who changes it?**
3. **`df -h /` shows 20% used. Why might your server still be out of disk space?**
4. **`/proc/meminfo` shows 8GB of RAM info, but `df -h /proc` reports size 0. Explain both facts at once.**
5. **You `rm` a 60GB logfile and `df` doesn't change. Why? What should you have done?**
6. **Why is `/bin` a symlink to `/usr/bin` on Ubuntu 22.04 — and what problem did the original split solve?**
7. **You deploy a Node app. Name the exact path for: the binary, the code, the systemd unit, the nginx config, the logs, the PID file.**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `ls` | List directory contents | `-F` (type suffixes), `-l` (long), `-a` (hidden), `-d` (the dir itself, not contents), `-h` (human sizes), `-S` (sort by size) |
| `tree` | Recursive tree view | `-L 2` (depth limit), `-d` (dirs only), `-h` (sizes). Not installed by default: `apt install tree` |
| `df` | Per-filesystem usage (asks `statfs()` — instant) | `-h` (human), `-T` (show fs type), `-i` (inode usage!), `-x tmpfs` (exclude type) |
| `du` | Per-directory usage (walks & `stat()`s — slow, exact) | `-s` (summary), `-h`, `--max-depth=1`, `-x` (don't cross mount points) |
| `findmnt` | Pretty tree of the mount table | `-t <type>` (filter), `-D` (df-like), no args = whole tree |
| `mount` | Mount a filesystem; with no args, lists mounts | `-t <type>`, `-o <opts>`. Prefer `findmnt` for reading. |
| `stat` | Dump a file's inode metadata | `-c '%i %n'` (custom format), `-f` (filesystem stats instead) |
| `file` | Identify real file type from magic bytes | `-b` (brief, no filename) |
| `readlink` | Show what a symlink points to | `-f` (resolve the FULL chain to a real path) |
| `lsof` | List open files | `+L1` (deleted-but-open files), `-p PID`, `+D /path` |
| `man hier` | The authoritative FHS documentation, offline | — |

---

## When Would I Use This at Work?

### Scenario 1: "The server is out of disk" alert at 3am
You don't guess. You run `df -hT` (all filesystems, with types), spot that `/var` is at 100% while `/` is fine, then `du -h --max-depth=1 /var | sort -rh | head` to bisect down to the offender in three commands. You `truncate -s 0` the runaway log instead of `rm`-ing it, so the space returns immediately without restarting nginx. Total time: 90 seconds.

### Scenario 2: "It works on my machine but not in the container"
Your deploy script has `#!/bin/bash` and uses `[[ ]]`. It works on your Ubuntu build agent and dies in `node:18-alpine`. You know instantly: Alpine's `/bin/sh` is BusyBox `ash`, there is no bash, and `/bin` vs `/usr/bin` differences are distro-specific. You fix it in one line (`apk add bash`, or rewrite POSIX) instead of spending an hour on Stack Overflow.

### Scenario 3: Auditing a server you've never seen before
A colleague hands you a production box with no documentation. In four commands you have a complete picture: `findmnt` (what filesystems exist and how big), `ls /etc/systemd/system/` (what custom services run), `ls /var/www /opt` (where apps are deployed), `ls -lt /var/log | head` (what's actively logging). You now know more about the box than the person who set it up.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | The kernel's VFS layer is what creates this single unified tree; `/proc` and `/sys` are the kernel exposing itself |
| **Next** | 03 — Everything Is a File | You saw `/dev/null`, `/proc/self`, and symlinks here. Next you learn WHY they can all be read with the same `read()` syscall |
| **Sets up** | 04 — Inodes in Depth | `stat` showed you an inode number; the `rm`-doesn't-free-space mystery is an inode link-count story |
| **Sets up** | 05 — File Permissions | The sticky bit on `/tmp` (`drwxrwxrwt`) and mode `640` on `/etc/shadow` |
| **Used by** | 22 — systemd, 23 — Logs, 31 — Disk Management | `/etc/systemd/system`, `/var/log`, `/etc/fstab` and mount points are the raw material of all three |
| **Used by** | 33 — Node in Production | The "where a Node app's pieces live" map is the blueprint for a real deployment |
