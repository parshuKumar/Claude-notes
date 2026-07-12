# 31 — Disk Management

## ELI5 — The Simple Analogy

Imagine a giant warehouse where you store all your stuff.

- **The physical building** is the disk — the actual hardware, the SSD or the spinning platter. It's just raw space.
- **The floor plan drawn on the blueprint** is the partition table. Before you can use the warehouse you draw lines: "this half is for furniture, this quarter is for documents." Those regions are partitions.
- **The shelving units, labels, and the index card catalog** you install inside a region — *that's* the filesystem (ext4, xfs). Raw space can't tell you where anything is. The filesystem is the organizing system that turns "byte number 4,812,556,032" into "the file `invoice.pdf` in the folder `2026`."
- **The door you walk through to get into that region** is the mount point. The warehouse has ONE front entrance (`/`), and every storage region is reached by walking down a hallway to a specific door — `/`, `/home`, `/var`, `/mnt/data`. You never think about which physical building you're in; you just walk to `/var/log`.

And here's the thing that trips everyone up: **the catalog (the filesystem) is separate from the shelf space (the data blocks).** You can run out of index cards while the shelves are half empty (**inode exhaustion**), and you can throw away a box's index card while a worker is still carrying the box, so the space stays occupied even though the catalog says it's gone (**deleted-but-open files**). Those two facts are why `df` and `du` disagree, and they are the heart of this whole topic.

---

## Where This Lives in the Linux Stack

```
Hardware: the physical SSD / NVMe / spinning disk
  │
  └── KERNEL
       │
       ├── Block Layer + device drivers
       │     exposes the disk as /dev/sda, /dev/nvme0n1, /dev/vda (topic 03 — everything is a file)
       │
       ├── Partitioning: /dev/sda1, /dev/nvme0n1p1  (a slice of the block device)
       │
       ├── [optional] Device Mapper / LVM: PV → VG → LV
       │
       ├── FILESYSTEM DRIVERS ◀◀◀ THIS TOPIC (ext4, xfs, tmpfs, overlayfs)
       │     turn block offsets ↔ inodes + data blocks (topic 04 — inodes)
       │
       ├── VFS (Virtual Filesystem Switch): the unified `/` tree (topic 02)
       │     mount points graft each filesystem onto the tree
       │
       └── System Calls: mount(), statfs(), open(), write(), unlink(), ftruncate()
            │
            └── C Library
                 │
                 └── Shell
                      │
                      └── lsblk, df, du, mount, mkfs, fdisk ◀◀◀ THIS TOPIC (the tools)
```

Disk management spans from the raw block device all the way up to the `/` tree you navigate every day. When you `df -h /var/log`, the kernel walks: which filesystem is mounted at the deepest ancestor of that path → which block device backs it → `statfs()` on its superblock → the numbers you see.

---

## What Is This?

Disk management is everything between a raw block device and the files your app reads and writes: partitioning the device, putting a filesystem on it, mounting it into the directory tree, making that mount survive reboots (`/etc/fstab`), and — the part you'll do most as a backend dev — **diagnosing why the disk is full and fixing it before it takes your service down.** A full disk is one of the top three causes of production outages, and unlike a memory leak it usually has an obvious, fast fix once you know where to look.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| The storage stack layers | You won't know whether to grow the volume, the partition, or the filesystem — and you'll grow the wrong one |
| `df` vs `du` | You'll `du -sh /*` for 20 minutes hunting space that `df` sees but `du` can't — the deleted-open-file trap |
| Inode exhaustion (`df -i`) | Your app will fail every write with "No space left on device" while `df -h` cheerfully shows 40 GB free, and the error will make zero sense |
| `/etc/fstab` typos | A one-character mistake will make the box **fail to boot** and drop to an emergency shell |
| Device names aren't stable | Your data volume will mount on the wrong path (or not at all) after a reboot re-enumerates the disks |
| Reserved blocks | Your `node` user gets ENOSPC at "95% full" while root can still write — and you won't know why |
| xfs can't shrink | You'll over-provision a volume and discover you can never take the space back |
| The deploy-host disk eaters | `/var/lib/docker` will quietly consume 60 GB of dead build cache and you won't think to look |

---

## The Storage Stack, Layer by Layer

This diagram is the spine of the entire topic. Learn to draw it top to bottom.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  PHYSICAL DEVICE      the actual NVMe SSD / SATA disk / cloud EBS vol │
  └───────────────────────────────┬──────────────────────────────────────┘
                                   │  kernel driver exposes it as...
  ┌────────────────────────────────▼─────────────────────────────────────┐
  │  BLOCK DEVICE         /dev/sda   /dev/nvme0n1   /dev/vda              │
  │  (topic 03)           a file in /dev you can read()/write() raw       │
  └───────────────────────────────┬──────────────────────────────────────┘
                                   │  a partition TABLE (MBR or GPT) carves it into...
  ┌────────────────────────────────▼─────────────────────────────────────┐
  │  PARTITION            /dev/sda1   /dev/nvme0n1p1                      │
  │                       a contiguous slice of the block device          │
  └───────────────────────────────┬──────────────────────────────────────┘
                                   │  ┌── OPTIONAL LVM layer ─────────────┐
                                   │  │ PV (physical volume) = the part   │
                                   │  │   └─ VG (volume group) = a pool   │
                                   │  │        └─ LV (logical volume)     │
                                   │  └───────────────────────────────────┘
                                   │  you put a filesystem on the partition (or the LV)...
  ┌────────────────────────────────▼─────────────────────────────────────┐
  │  FILESYSTEM           ext4 / xfs                                      │
  │  (topic 04)           superblock + inode table + data blocks          │
  │                       THIS is what knows about files and free space   │
  └───────────────────────────────┬──────────────────────────────────────┘
                                   │  mount() grafts it onto...
  ┌────────────────────────────────▼─────────────────────────────────────┐
  │  MOUNT POINT          /   or  /var  or  /mnt/data                     │
  │  (topic 02)           a directory in the ONE unified tree             │
  └───────────────────────────────┬──────────────────────────────────────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────────┐
  │  YOUR FILES           /var/log/app.log  →  node writes here           │
  └──────────────────────────────────────────────────────────────────────┘
```

**When something is "full" or "won't grow," you fix it at a specific layer:**
- Cloud console resizes the **physical device**.
- `growpart` resizes the **partition**.
- `lvextend` resizes the **logical volume**.
- `resize2fs` / `xfs_growfs` resizes the **filesystem**.
Get the layer wrong and nothing happens (or you corrupt something). This is why the mental model matters more than the commands.

---

## Block Devices and Naming

```bash
lsblk
# NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# nvme0n1     259:0    0  100G  0 disk                 ← the whole device
# ├─nvme0n1p1 259:1    0 99.9G  0 part /               ← partition 1, mounted at /
# └─nvme0n1p15 259:2   0  100M  0 part /boot/efi        ← the EFI partition
# sdb         8:16     0  500G  0 disk                 ← a second disk, no partition table yet
```

| Name | What it is | Where you see it |
|---|---|---|
| `/dev/sda`, `/dev/sdb` | SATA/SCSI/USB disks. Letters = disk order (`a`, `b`, `c`...) | Bare metal, older VMs |
| `/dev/sda1`, `/dev/sda2` | **Partitions** of `sda`. Number appended directly | — |
| `/dev/nvme0n1` | An NVMe SSD. `nvme0` = controller 0, `n1` = namespace 1 | Modern servers, most fast cloud instances |
| `/dev/nvme0n1p1` | **Partition** of an NVMe device. Note the **`p`** before the number | ⚠️ **This trips people up** — it's `p1`, not `1` |
| `/dev/vda`, `/dev/vdb` | **virtio** disks — paravirtualized | Most cloud VMs (AWS Nitro, KVM, DigitalOcean) |
| `/dev/xvda` | Xen virtual disk | Older AWS EC2 |
| `/dev/loop0` | A **loop device** — a file pretending to be a block device | Snap packages, mounted `.iso`/disk images |

**The NVMe naming trap:** on a SATA disk the first partition of `sda` is `sda1`. On NVMe, the first partition of `nvme0n1` is `nvme0n1**p1**` — the `p` disambiguates the partition number from the namespace number. Forget the `p` and your `mkfs`/`mount`/`fstab` command silently targets the wrong thing (or errors).

---

## Filesystems Compared, Honestly

| FS | Strengths | The catch you must know | Use it when |
|---|---|---|---|
| **ext4** | Mature, safe, well-understood, **can grow AND shrink** | Slightly slower than xfs on huge parallel workloads | The default. When in doubt, ext4 |
| **xfs** | Excellent for large files and high-parallelism I/O; RHEL's default | **CANNOT BE SHRUNK. Ever. Grow only.** Over-provision and you're stuck | Big data volumes, media, DB files where you'll only ever grow |
| **btrfs / zfs** | Copy-on-write, snapshots, checksums, compression | More complex; zfs isn't in the mainline kernel (licensing) | When you need snapshots/CoW and know what you're doing |
| **tmpfs** | **RAM-backed** — blistering fast | **VANISHES on reboot, and CONSUMES RAM** | `/tmp` scratch, build caches — see the warning |
| **overlayfs** | Layered, copy-on-write — the engine of Docker images | **Copy-up cost**: modifying a big file copies the *whole* file into the writable layer first | You don't choose it; containers use it for you |

**xfs can't shrink — a real production constraint.** You provision a 500 GB xfs data volume, realize you only need 100, and want the money back. You can't. `xfs_growfs` exists; there is no `xfs_shrink`. Your only path is: create a new smaller volume, copy the data, swap them. With ext4 you'd just `resize2fs /dev/sdb1 100G`. **Check the filesystem type before you provision generously.**

**tmpfs — fast, and a foot-gun.** `/dev/shm` and `/run` are tmpfs. It lives in RAM (and swap), so reads/writes are memory-speed — great for a build cache or a hot scratch dir. But two dangers:
```bash
mount | grep tmpfs
# tmpfs on /run type tmpfs (rw,nosuid,nodev,size=1583316k)
# tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,inode64)
```
1. Everything on it **disappears on reboot** — never put anything you need to keep there.
2. It **consumes RAM**. A runaway process that writes 8 GB to a tmpfs on an 8 GB box will **OOM the machine** (topic 01, 18) — the kernel counts tmpfs pages as memory pressure. A "disk-full" bug on tmpfs looks like an out-of-memory bug, because it *is* one.

**overlayfs — why the first write to a big file in a container is slow.** A container's root filesystem is a read-only image stack plus a thin writable layer. The first time you modify a file that lives in a lower (image) layer, overlayfs must **copy the entire file up** into the writable layer before changing a byte. Rewriting one line of a 2 GB file copies 2 GB. This is why "just `sed -i` a config in the running container" can be surprisingly slow, and why container writable layers balloon.

---

## Creating Filesystems and Mounting

### mkfs — the destroyer

```
mkfs.ext4 /dev/sdb1
│    │    │
│    │    └── the target. ⚠️⚠️ THIS ERASES EVERYTHING on /dev/sdb1. There is no undo.
│    │        Double-check with `lsblk -f` that it's the right device and it's UNMOUNTED
│    │        and has no data you care about. `mkfs` on the wrong device is a resume-generating event.
│    └── the filesystem type: mkfs.ext4 / mkfs.xfs / mkfs.vfat
└── "make filesystem"

mkfs.xfs /dev/sdb1        # xfs equivalent
mkfs.ext4 -L data /dev/sdb1   # -L sets a LABEL you can mount by (safer than device name)
```

### mount — in depth

```
mount -o noatime,nodev /dev/sdb1 /mnt/data
│     │  │              │         │
│     │  │              │         └── the MOUNT POINT (must be an existing empty directory)
│     │  │              └── the block device (or LABEL=data / UUID=...)
│     │  └── the mount OPTIONS (comma-separated) — see the table below
│     └── -o introduces options
└── graft this filesystem onto the tree at /mnt/data

mount -o remount,rw /            # change options on an already-mounted fs (e.g. read-only → read-write)
mount -a                         # mount everything in /etc/fstab — THE command that tests fstab safely
mount                            # with no args: list everything currently mounted
```

**Mount options that actually matter:**

| Option | What it does | Why you care |
|---|---|---|
| `defaults` | = `rw,suid,dev,exec,auto,nouser,async` | The baseline in most fstab lines |
| `ro` / `rw` | read-only / read-write | The kernel remounts a corrupted fs **`ro`** to protect it — see the dmesg section |
| `noatime` | **Don't update access time on reads** | A real perf win. By default, every *read* updates the inode's atime → a *write* (topic 04). On a read-heavy app that's a lot of pointless writes. `noatime` kills them |
| `relatime` | Update atime only if it's older than mtime (or >24h) | The modern *default* compromise — most of `noatime`'s benefit, keeps atime roughly useful |
| `nosuid` | Ignore setuid bits (topic 05) | Hardening: a setuid binary dropped here can't escalate |
| `nodev` | Ignore device files | Hardening: nobody smuggles a `/dev/sda` node onto a user-writable mount |
| `noexec` | **Refuse to execute binaries from this fs** | Hardening `/tmp` — **and the cause of a great war story** (below) |
| `nofail` | Don't fail the boot if this device is missing | **Put this on non-critical volumes** so a detached data disk can't wedge the boot |

**The `noexec` war story:** hardening guides tell you to mount `/tmp` with `noexec`. Good idea — it stops an attacker from dropping and running a binary there. Then one day `npm install` mysteriously fails with `Permission denied`, or a downloaded installer (`curl ... | sh`) dies. Root cause: many tools extract a helper binary or a native `.node` addon into `/tmp` (or `$TMPDIR`) and **exec it** — and `noexec` blocks the `execve()` at the kernel level. The fix is to point the tool's temp dir elsewhere (`TMPDIR=/var/tmp npm install`, if `/var/tmp` is exec-able) rather than removing the hardening. Knowing this saves an afternoon of "why can't npm run?"

### umount and "target is busy"

```bash
umount /mnt/data
# umount: /mnt/data: target is busy.
```
The kernel won't unmount a filesystem while any process has a file open on it or has `cd`'d into it. **Find the culprit** (topic 19 — file descriptors, and the same `lsof`/`fuser` tools):
```bash
lsof +D /mnt/data          # every open file UNDER /mnt/data (slow — walks the tree)
fuser -vm /mnt/data        # every process USING that mount — faster, shows PIDs + how
# USER  PID  ACCESS COMMAND
# node 1455  ..c..  node        ← 'c' = cwd is in there; the process has cd'd in
```
Kill or `cd` those processes out, then unmount. **Escape hatch:** `umount -l /mnt/data` (lazy) detaches it from the tree *now* and cleans up when the last handle closes — useful when you can't kill the process immediately, but understand you're deferring, not solving.

---

## /etc/fstab — Annotated, and the Booby Trap

`/etc/fstab` is the table the boot process reads to mount filesystems automatically. **A typo here can make the machine fail to boot.** Six fields per line:

```
UUID=8f3b2c1a-...  /mnt/data   ext4    defaults,noatime,nofail   0   2
│                  │           │       │                         │   │
│                  │           │       │                         │   └── (6) FSCK ORDER (pass):
│                  │           │       │                         │       0 = never check
│                  │           │       │                         │       1 = check FIRST (root / only)
│                  │           │       │                         │       2 = check after the pass-1 fs
│                  │           │       │                         └── (5) DUMP: legacy backup flag.
│                  │           │       │                             Almost always 0.
│                  │           │       └── (4) OPTIONS: the mount -o options.
│                  │           │           `nofail` here = boot even if this disk is gone.
│                  │           └── (3) FILESYSTEM TYPE: ext4 / xfs / vfat / swap
│                  └── (2) MOUNT POINT: where in the tree it attaches
└── (1) DEVICE: ⚠️ USE UUID= OR LABEL=, NOT /dev/sdb1 (see below)
```

**Always use `UUID=` (or `LABEL=`), never `/dev/sdb1`.** Device names are assigned in **enumeration order at boot** and are **not stable**. Add a disk, move a cable, resize on a cloud provider, or just get unlucky with driver probe timing, and yesterday's `/dev/sdb` becomes today's `/dev/sdc`. Your fstab line now mounts the *wrong disk* at `/mnt/data`, or fails. A UUID is baked into the filesystem's superblock and follows the data wherever the kernel enumerates it. Get UUIDs from `lsblk -f` or `blkid`.

**The boot booby-trap, and how to never get bitten:**
```bash
# You just added a line to /etc/fstab. DO NOT REBOOT YET.
# Test it live — this mounts everything in fstab that isn't already mounted:
sudo mount -a
# If it returns silently → the entry is valid, the reboot will be fine.
# If it errors:
#   mount: /mnt/data: wrong fs type, bad option, bad superblock...
# → FIX IT NOW, while you still have a shell. A bad fstab line + reboot =
#   the boot hangs and drops you to:
#   "You are in emergency mode. Give root password for maintenance
#    (or press Control-D to continue):"
#   ...which on a remote/headless cloud box means the serial console, or worse.
```
And for any **non-critical** volume, add `nofail` so a missing/detached disk degrades gracefully (the mount is skipped) instead of blocking the boot. Reserve the strict default for `/` itself.

---

## df vs du — The Definitive Treatment

This is the centerpiece. As a backend dev, "disk full at 2am" is a runbook you *will* execute, and it starts here.

```
   ┌─────────────────────────────────────┐        ┌─────────────────────────────────────┐
   │  df  — "how full is the FILESYSTEM" │        │  du  — "how big is this DIRECTORY"  │
   ├─────────────────────────────────────┤        ├─────────────────────────────────────┤
   │  Asks the SUPERBLOCK (one number    │        │  WALKS the directory tree, adds up  │
   │  the fs keeps: total/used/free      │        │  the size of every file it can see  │
   │  blocks + inodes).                  │        │  from a given path.                 │
   │                                     │        │                                     │
   │  INSTANT (O(1)).                    │        │  SLOW (O(files)) — can take minutes │
   │                                     │        │  on /var or /.                      │
   │  Sees deleted-but-open files        │        │  CANNOT see deleted-but-open files  │
   │  (they still hold blocks).          │        │  (no directory entry to walk).      │
   │  Sees files hidden under mounts.    │        │  Can't see under a mount if you     │
   │                                     │        │  walk from above it.                │
   │  "The disk is 100% full."           │        │  "THIS folder is why."              │
   └─────────────────────────────────────┘        └─────────────────────────────────────┘
        the WHOLE-DISK gauge                            the WHICH-DIRECTORY finder
```

### df — the filesystem gauge

```bash
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1   99G   94G  0.5G  99% /            ← the problem is on /
# tmpfs           3.9G     0  3.9G   0% /dev/shm
# /dev/nvme0n1p15  98M  6.1M   92M   7% /boot/efi
#      │           │    │     │     │   │
#      │           │    │     │     │   └── where it's mounted (what you `cd` to)
#      │           │    │     │     └── percent used (from the superblock — instant)
#      │           │    │     └── available to a NORMAL user (see reserved blocks below)
#      │           │    └── used
#      │           └── total size
#      └── the block device
```

### df -i — inode exhaustion, the error that makes no sense

```bash
df -i
# Filesystem      Inodes  IUsed IFree IUse% Mounted on
# /dev/nvme0n1p1  6.4M    6.4M   1.2K  100% /          ← 100% of INODES used
```
A filesystem has a **fixed number of inodes**, set at `mkfs` time (topic 04 — every file/dir needs exactly one inode). Millions of tiny files — a runaway session store, an npm cache, a mail queue, a log-per-request bug — can **exhaust the inode table while gigabytes of block space sit free**. The symptom is maddening:
```bash
touch /var/lib/app/newfile
# touch: cannot touch '/var/lib/app/newfile': No space left on device
df -h /var/lib/app
# ... Use% 41% ...      ← "but there's 40 GB free?!"
df -i /var/lib/app
# ... IUse% 100% ...    ← THERE it is. Out of inodes, not out of space.
```
**`No space left on device` (ENOSPC) means "out of blocks OR out of inodes."** Always check `df -i` when `df -h` looks fine. Find the offender:
```bash
# which directory has the most FILES (not the most bytes)?
sudo find /var -xdev -type f | cut -d/ -f1-4 | sort | uniq -c | sort -rn | head
```
The fix is deleting/consolidating the tiny files; you can't add inodes to an existing ext4 without recreating it.

### du — the directory finder

```bash
du -sh /var/log
# 4.2G    /var/log        ← -s = summary (one total), -h = human-readable

# The command you actually run in a disk-full incident:
du -h --max-depth=1 /var | sort -h
#       │            │
#       │            └── which directory to scan
#       └── only go ONE level deep — show each immediate child's total, not every file
# 12K     /var/tmp
# 340M    /var/cache
# 4.2G    /var/log
# 61G     /var/lib          ← follow this down
# 66G     /var
# ↑ `sort -h` puts the biggest last, so the culprit is right above your prompt.

# Then drill in:
du -h --max-depth=1 /var/lib | sort -h
# 58G     /var/lib/docker   ← there it is

du -xh --max-depth=1 /       # -x = stay on ONE filesystem; don't wander into /proc, /sys,
#                              or other mounts and give you nonsense totals
```

---

## Why du and df Disagree — The Two Classic Causes

You run `df -h` → `/` is 96% full. You run `du -sh /*` → it adds up to *half* that. Where did the space go? Two causes.

### Cause 1: The deleted-but-still-open file (the big one)

This is the topic 09 / topic 19 callback, and it is the #1 "df and du disagree" mystery.

```
  Your Node app opens /var/log/app.log and holds the file descriptor.
  Log rotation (or a panicked human) runs `rm /var/log/app.log`.

  ┌──────────────────┐           ┌──────────────────────────────────┐
  │ DIRECTORY ENTRY  │           │ INODE + DATA BLOCKS (20 GB)       │
  │ "app.log" → inode│  rm  ✗    │ link count now 0 from the dir...  │
  │  (topic 04)      │ ───────▶  │ ...BUT the node process still has │
  └──────────────────┘  DELETED  │ an OPEN fd → kernel keeps the     │
                                 │ inode and its 20 GB ALIVE.        │
   du walks the DIRECTORY tree → │                                   │
   sees no "app.log" → counts 0. │ df reads the SUPERBLOCK → the     │
                                 │ 20 GB of blocks are still         │
                                 │ allocated → counts them.          │
                                 └──────────────────────────────────┘
   RESULT: df says 96% full. du can't find the 20 GB. They disagree.
   The blocks are freed ONLY when the last fd closes (process exits or truncates).
```

**Find it and fix it — without a restart:**
```bash
sudo lsof +L1
#        │
#        └── list open files whose LINK COUNT is < 1, i.e. DELETED but still open
# COMMAND  PID  USER  FD   TYPE  DEVICE  SIZE/OFF   NLINK  NODE  NAME
# node    1455 deploy 20w  REG   259,1  21474836480    0  918273 /var/log/app.log (deleted)
#                                       ^^^^^^^^^^^ 20 GB       ^ NLINK 0 = deleted
#
# THE FIX — reclaim the space WITHOUT killing the process:
sudo truncate -s 0 "/proc/1455/fd/20"
#        │        │  └── the process's fd to the deleted file, via /proc (topic 16, 19)
#        │        └── set size to 0 → the kernel frees the blocks immediately
#        └── NOT `rm` — the directory entry is already gone; rm does nothing.
# df -h now shows the 20 GB back, and the app keeps running (it just writes past offset 0).
#
# The other fix: restart the process (systemctl restart app) → it closes the fd → blocks freed.
```
**The lesson: `rm` on a big log that a process still has open does NOT free space.** Restart the writer or truncate its fd. This is why proper log rotation uses `copytruncate` or signals the process to reopen (topic 23).

### Cause 2: Files hidden under a mount point

If a directory had files in it *before* you mounted a filesystem on top, those files still exist on the underlying filesystem but are **shadowed** — you can't see or reach them while the mount is active. `df` (measuring the underlying fs) counts them; `du` walking the mounted tree cannot. To find them, `umount` and look, or bind-mount the parent elsewhere.

---

## The "Disk Full at 2am" Runbook

Copy-pasteable triage. Run these in order.

```bash
# ── 1. HOW full, and WHICH filesystem? ──────────────────────────────────────
df -h
#   → note the mount that's at 95-100%. Everything below targets THAT mount.

# ── 2. Is it BLOCKS or INODES? (do this SECOND, always) ─────────────────────
df -i
#   → if IUse% is 100% but Use% is low, it's inodes. Hunt tiny files, skip to find below.

# ── 3. WHICH directory on the full mount? Walk down, biggest last. ──────────
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -15
#   → -x keeps you on the root fs. Repeat into the biggest child:
sudo du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail
sudo du -xh --max-depth=1 /var/lib 2>/dev/null | sort -h | tail

# ── 4. The usual suspects, in order of likelihood on a deploy host ──────────

#  (a) DOCKER — the #1 disk eater on a deploy host
docker system df
# TYPE            TOTAL   ACTIVE   SIZE     RECLAIMABLE
# Images          42      6        18.3GB   14.1GB (77%)
# Containers      9       3        1.2GB    0.9GB
# Local Volumes   15      4        22.4GB   19.8GB (88%)
# Build Cache     301     0        31.7GB   31.7GB (100%)   ← build cache is often the biggest
docker system prune -a --volumes      # ⚠️ removes ALL unused images, networks, build cache
#                    │  └── AND unused volumes — make SURE no volume holds data you need
#                    └── -a = also remove unused (not just dangling) images
docker builder prune -af              # just the build cache, if that's the culprit

#  (b) LOGS that never rotated (topic 23)
sudo du -xh --max-depth=1 /var/log | sort -h | tail
sudo find /var/log -type f -name '*.log' -size +500M -exec ls -lh {} \;

#  (c) The systemd JOURNAL
journalctl --disk-usage
# Archived and active journals take up 4.0G in the file system.
sudo journalctl --vacuum-size=200M     # keep only the most recent 200 MB
sudo journalctl --vacuum-time=7d       # or keep only the last 7 days

#  (d) OLD KERNELS piling up in /boot (can fill a small /boot partition)
sudo apt autoremove --purge

#  (e) The APT package cache
sudo du -sh /var/cache/apt
sudo apt clean                         # empties /var/cache/apt/archives

#  (f) /tmp full of junk
sudo du -xh --max-depth=1 /tmp | sort -h | tail

# ── 5. df says full but du can't find it? → the phantom (deleted-open file) ─
sudo lsof +L1 | sort -k7 -n | tail
#   → truncate the fd or restart the process (see above).

# ── 6. EMERGENCY: need space THIS SECOND to keep the service alive ──────────
sudo truncate -s 0 /var/log/nginx/access.log    # zero a huge log in place, keep the file/inode
#   ⚠️ NOT `rm` — rm on an open log doesn't free space AND can break the writer.
#      truncate keeps the inode; the process keeps writing to the same fd.
```

---

## Reserved Blocks — Why "100% Full" Still Lets Root Write

By default, **ext4 reserves 5% of the filesystem for the root user** (and for system daemons running as root). It's not a bug — it exists so that if a user process fills the disk, root and critical daemons can still function (log, run, let you *fix* it), and so the filesystem's block allocator doesn't get pathologically fragmented near 100%.

The confusing consequence:
```bash
# Your `node` user (UID 1001):
node -e "require('fs').writeFileSync('/data/x', Buffer.alloc(1e8))"
# Error: ENOSPC: no space left on device
df -h /data
# ... Use% 95% Avail 0 ...     ← "95%? and root could still write? what?"
```
The `Avail` column already **subtracts the reserved 5%** — that's why it can read 0 while `Use%` is 95%. Root ignores the reservation; your `node` user doesn't. On a big **data** volume (not `/`), 5% reserved for root is pure waste — reclaim it:
```bash
sudo tune2fs -m 1 /dev/nvme0n1p1     # reduce reserve from 5% to 1%
#            │  │
#            │  └── -m 1 = reserve 1 percent (use -m 0 on a pure data disk with no root daemons)
#            └── tune2fs adjusts ext4 parameters on an existing filesystem
# On a 500 GB volume, 5% → 1% hands you back ~20 GB instantly. No reformat, no downtime.
```
(Only meaningful for ext4; xfs doesn't reserve like this.)

---

## Growing a Disk on a Cloud VM (Online, No Downtime)

You resized the EBS/persistent volume in the AWS/GCP console from 100 GB to 200 GB. The VM still shows 100 GB — because you only grew the **physical device** layer. You must now grow the partition, then the filesystem. Full real sequence:

```bash
# 1. Confirm the kernel sees the bigger device
lsblk
# nvme0n1     259:0  0  200G  0 disk          ← device is now 200G...
# └─nvme0n1p1 259:1  0  100G  0 part /         ← ...but the partition is still 100G

# 2. Grow the PARTITION to fill the device (note the SPACE: device, then partition number)
sudo growpart /dev/nvme0n1 1
# CHANGED: partition=1 start=2048 old: size=... new: size=...

# 3. Grow the FILESYSTEM to fill the partition
#    ext4:
sudo resize2fs /dev/nvme0n1p1
#    xfs (note: xfs grows by MOUNT POINT, not device, and must be MOUNTED):
sudo xfs_growfs /

# 4. Verify
df -h /
# /dev/nvme0n1p1  197G  94G  103G  48% /        ← done, and the app never stopped
```
All of this is **online** — the filesystem stays mounted and the app keeps running throughout. This is one of the most common real cloud-ops tasks, and the #1 mistake is stopping after step 1 or growing the wrong layer.

---

## LVM Basics — Why It Exists

LVM inserts a flexible layer between partitions and filesystems so you can grow across disks painlessly:

```
  /dev/sdb  ─┐
             ├─▶ Physical Volumes (PV) ─▶ Volume Group (VG "data") ─▶ Logical Volume (LV "app")
  /dev/sdc  ─┘                                                              │
                                                                    filesystem on the LV
```
```bash
sudo pvcreate /dev/sdb /dev/sdc          # mark disks as PVs
sudo vgcreate data /dev/sdb /dev/sdc     # pool them into VG "data"
sudo lvcreate -L 400G -n app data        # carve out a 400G LV
sudo mkfs.ext4 /dev/data/app             # filesystem on it
# ...later, out of space? add /dev/sdd to the VG and grow the LV + fs in ONE command:
sudo lvextend -r -L +200G /dev/data/app
#            │  │
#            │  └── -L +200G = add 200 GB
#            └── -r = ALSO resize the filesystem (resize2fs/xfs_growfs) automatically
```
**The whole point of LVM is that `lvextend -r`** — grow a volume across multiple physical disks with zero downtime and no repartitioning. If you know a volume will grow unpredictably, put it on LVM from day one.

---

## Swap — The Overflow Disk

Swap is disk space the kernel uses as overflow when RAM is full (topic 18). Not a substitute for RAM — it's orders of magnitude slower — but it prevents instant OOM kills.

```bash
swapon --show
# NAME      TYPE  SIZE  USED PRIO
# /swapfile file  2G    412M  -2
cat /proc/swaps          # same info, the raw kernel view

# Create a 2G swapfile (modern approach — a file, not a partition):
sudo fallocate -l 2G /swapfile        # allocate 2 GB
sudo chmod 600 /swapfile              # only root may read it (it can contain secrets from RAM)
sudo mkswap /swapfile                 # format it as swap
sudo swapon /swapfile                 # activate it now
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # persist across reboots

# swappiness (0-100): how eagerly the kernel swaps. Default 60. For a DB/latency-sensitive
# app, lower it so the kernel prefers dropping cache over swapping app memory (topic 18):
cat /proc/sys/vm/swappiness           # 60
sudo sysctl vm.swappiness=10          # and add `vm.swappiness=10` to /etc/sysctl.conf to persist
```

---

## SMART and dmesg — When the Hardware Is Dying

Filesystems get corrupted for two reasons: a crash mid-write, or a **failing disk**. When the kernel detects a filesystem error, it protects your data by **remounting the filesystem read-only**:

```bash
dmesg -T | grep -iE 'error|I/O|read-only|remount|ext4|xfs' | tail
# [Sun Jul 12 02:11:04] blk_update_request: I/O error, dev nvme0n1, sector 4823910
# [Sun Jul 12 02:11:04] EXT4-fs error (device nvme0n1p1): ext4_find_entry:1463: inode #...
# [Sun Jul 12 02:11:04] EXT4-fs (nvme0n1p1): Remounting filesystem read-only
#                                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
**Symptom your app sees:** suddenly *every write* fails with `EROFS: read-only file system`, even as root, even though there's plenty of space. The app can read but not write. **This is not a permissions problem — the kernel took the whole filesystem read-only because it detected corruption or an I/O error.** What to do:
```bash
# 1. Check the disk's own health (SMART):
sudo smartctl -a /dev/nvme0n1 | grep -iE 'health|reallocated|pending|media|wear'
# SMART overall-health self-assessment test result: FAILED    ← the disk is dying. Replace it.
# Media and Data Integrity Errors: 47

# 2. To repair the FILESYSTEM you must run fsck on it UNMOUNTED (or from a rescue boot):
#    - for a data volume: umount it, then `sudo fsck -y /dev/sdb1`
#    - for the ROOT fs: you can't fsck a mounted root; reboot into recovery, or on cloud,
#      the box often auto-fscks on the next boot. `tune2fs -l | grep -i 'mount count'` shows
#      when the next automatic check will run.
```
If SMART says FAILED or the reallocated-sector count is climbing, `fsck` is a temporary patch — **the disk is failing and needs replacing.** On a cloud VM, detach and swap the volume.

## iostat / iotop — Is the Disk the Bottleneck? (topic 18 callback)

```bash
iostat -xz 2                 # extended stats, every 2s; -z hides idle devices
# Device  r/s   w/s   rkB/s   wkB/s  await  %util
# nvme0n1 12.0  840.0 480.0   210000  92.4   99.8    ← %util 99.8 = disk is SATURATED,
#                                                      await 92ms = every I/O waits 92ms
sudo iotop -o                # live, per-PROCESS I/O; -o shows only processes doing I/O now
# TID  PRIO USER  DISK READ  DISK WRITE  COMMAND
# 1455 be/4 deploy  0 B/s    198 M/s     node server.js    ← which process is hammering the disk
```
`%util` near 100% with high `await` means the disk (not the CPU) is your bottleneck — the classic signature of a "slow app" that's actually waiting on disk I/O.

---

## Common Mistakes

### Mistake 1: `du -sh /*` to find "the missing space" that df reports — but du can't see it

**Wrong:** `df` says `/` is 98% full. You run `du -sh /* | sort -h`, it sums to 45 GB on a 100 GB disk, and you conclude "df is wrong / the disk is lying."

**Root cause:** A process holds an fd to a large file that was `rm`'d. The directory entry is gone (so `du`, which walks directory entries, cannot see it), but the inode's data blocks are still allocated (so `df`, which reads the superblock, still counts them — topic 04, topic 19). `df` and `du` are measuring two different things and *both are correct*.

**Fix:** `sudo lsof +L1` reveals the deleted-but-open file and its size. `truncate -s 0 /proc/<pid>/fd/<n>` or restart the process to free the blocks. **`rm` will not help — there's nothing left to unlink.**

**Prevention:** rotate logs with `copytruncate` (or signal the app to reopen its files); never `rm` a log a running process still writes to.

---

### Mistake 2: "No space left on device" with gigabytes free — ignoring inodes

**Wrong:** `touch` fails with ENOSPC. `df -h` shows 40% used. You assume filesystem corruption and start planning a reformat.

**Root cause:** The **inode table is exhausted** (`df -i` = 100%). Every file needs an inode (topic 04); a runaway process created millions of tiny files (sessions, per-request logs, an unbounded cache), consuming all inodes long before the block space ran out. `ENOSPC` means "out of blocks *or* out of inodes."

**Fix:** `df -i` confirms it. Find the directory with the most *files* (`find /path -xdev -type f | wc -l`, or the `uniq -c` one-liner above) and delete/consolidate them. You cannot add inodes to an existing ext4 without recreating the filesystem — so the real fix is stopping whatever creates the tiny files.

**Prevention:** cap caches, rotate/clean session and temp files, and monitor `df -i` alongside `df -h`.

---

### Mistake 3: A typo in /etc/fstab and a reboot — the box won't boot

**Wrong:** You add `/dev/sdb1 /mnt/data ext4 defaults 0 2`, fat-finger `ext4` as `ext3` (or the device is now `sdc`), and reboot to "make sure it mounts on boot."

**Root cause:** systemd tries to mount every fstab entry during boot. A failing mount on a non-`nofail` entry blocks boot and drops the machine into **emergency mode** ("Give root password for maintenance"). On a headless cloud box you now need the serial/rescue console.

**Fix / the habit that prevents it:** **never reboot to test fstab.** After editing, run `sudo mount -a` — it applies the fstab live and errors immediately if the line is bad, while you still have a working shell. Add `nofail` to non-critical volumes so a missing disk can never wedge the boot. And use `UUID=` so a re-enumerated device doesn't silently point the line at the wrong disk.

---

### Mistake 4: `mkfs` on the wrong device — instant, total data loss

**Wrong:** You mean to format a new blank disk `sdc` but run `mkfs.ext4 /dev/sda1` (or `/dev/sdb1` with last month's data on it).

**Root cause:** `mkfs` writes a fresh superblock and inode table, orphaning every existing data block. There is no undo. The kernel happily does exactly what you asked — it doesn't know `sda1` was your production data.

**Fix:** there isn't a clean one — restore from backup. **Prevention:** before any `mkfs`, run `lsblk -f` and confirm the target has **no filesystem, no label, no mount point, and no data you care about**. Say the device name out loud. Unmount-check it. Treat `mkfs`, `dd`, and `wipefs` as loaded weapons.

---

### Mistake 5: Grew the cloud volume, but the filesystem still shows the old size

**Wrong:** You bumped the EBS volume to 200 GB in the console. `df -h` still shows 100 GB. You conclude the resize "didn't take."

**Root cause:** You only enlarged the **physical device** layer. The **partition** and the **filesystem** on top are unchanged — they still describe the old geometry. Each layer must be grown in turn (see the storage-stack diagram).

**Fix:** `lsblk` (device is big, partition is small) → `sudo growpart /dev/nvme0n1 1` (grow the partition) → `sudo resize2fs /dev/nvme0n1p1` (ext4) or `sudo xfs_growfs /` (xfs) → `df -h` to verify. All online, no downtime.

**Prevention:** remember the stack has four resizable layers (device → partition → [LV] → filesystem), and a console resize only touches the first.

---

## Hands-On Proof

```bash
# PROVE IT: the storage stack is real — walk it top to bottom
lsblk -f            # device → partition → FSTYPE → UUID → MOUNTPOINT, all in one view
findmnt /           # which device backs the / mount, and with what options
df -h /             # the fs gauge for that mount
stat -f /           # statfs() — block size, total/free blocks, total/free INODES

# PROVE IT: df reads the superblock (instant); du walks the tree (slow)
time df -h /                     # real 0m0.01s
time du -sh /usr 2>/dev/null     # real 0m3s+ — it visited every file under /usr

# PROVE IT: inodes are finite and separate from block space (topic 04)
df -h /var ; df -i /var          # two different "used" percentages for the same fs

# PROVE IT: the deleted-but-open file — df and du disagree, live
cd /tmp
# terminal A: hold a big deleted file open
python3 -c "import time; f=open('ghost','wb'); f.write(b'x'*500_000_000); import os; os.remove('ghost'); print('deleted but held open, PID',os.getpid()); time.sleep(300)" &
sleep 2
du -sh /tmp/ghost 2>/dev/null    # du: cannot find it (no directory entry)
df -h /tmp                        # df: still counts the ~500 MB
sudo lsof +L1 | grep ghost        # THERE it is: NLINK 0, size ~500M, "(deleted)"
kill %1 ; sleep 1 ; df -h /tmp    # space returns the instant the fd closes

# PROVE IT: tmpfs lives in RAM and vanishes — and counts as memory
mount | grep '/dev/shm'
free -h ; dd if=/dev/zero of=/dev/shm/blob bs=1M count=500 ; free -h   # watch 'used' RAM climb
rm /dev/shm/blob

# PROVE IT: reserved blocks — Avail already subtracts the 5% root reservation
sudo tune2fs -l /dev/nvme0n1p1 | grep -iE 'reserved block count|block count'
# Block count:            26214144
# Reserved block count:   1310707        ← ~5% held back for root

# PROVE IT: mount options are enforced by the kernel (noexec)
sudo mkdir /mnt/noexec-test
sudo mount -t tmpfs -o noexec tmpfs /mnt/noexec-test
cp /bin/echo /mnt/noexec-test/ && /mnt/noexec-test/echo hi
# bash: /mnt/noexec-test/echo: Permission denied   ← execve() blocked by the mount option
sudo umount /mnt/noexec-test

# PROVE IT: Docker's disk usage is invisible to a casual df
docker system df
sudo du -sh /var/lib/docker      # the number that df attributes to "/" but you never see in the app
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
# 1. Run `lsblk -f`. Draw the storage stack for YOUR root filesystem: name the block
#    device, the partition, the filesystem type, the UUID, and the mount point.
# 2. Run `df -h` and `df -i` for /. What percentage of BLOCKS and of INODES are used?
#    Are they close, or wildly different? What would it mean if inodes were at 100%
#    but blocks at 30%?
# 3. Run `du -h --max-depth=1 /var | sort -h`. Which single subdirectory of /var is
#    the largest? Now drill one level deeper into it.
```

### Exercise 2 — Medium

**Reproduce and fix the df/du disagreement.**

```bash
# 1. In one terminal, create a program that opens a 200 MB file, deletes it from the
#    directory, and then sleeps for 5 minutes holding the fd open.
# 2. In another terminal, show that `df -h /tmp` counts the 200 MB but `du -sh` of that
#    path finds nothing. Explain WHY, in terms of directory entries vs inodes vs the
#    superblock (topic 04, topic 19).
# 3. Use `lsof +L1` to locate the phantom file. Read off its size and its NLINK.
# 4. Reclaim the space TWO ways, on two separate runs: (a) `truncate -s 0 /proc/<pid>/fd/<n>`
#    while the process keeps running, and (b) by killing the process. Confirm with `df`
#    that the space returns each time.
```

### Exercise 3 — Hard (Production Simulation)

**The disk-full incident, end to end.**

```bash
# SETUP (simulate a full-ish disk on a throwaway VM/container):
#   - Fill /var/log with a fake unrotated 2 GB log:  fallocate -l 2G /var/log/huge.log
#   - Create an inode-heavy directory: mkdir /tmp/sess && cd /tmp/sess &&
#     for i in $(seq 1 200000); do : > "s$i"; done   (200k empty files)
#   - Start a process holding a deleted 1 GB file open (as in Exercise 2).
#
# THE INCIDENT: "disk full at 2am, the app can't write." Run the RUNBOOK in order and,
# for EACH of the three problems you planted, show the exact command that reveals it
# and the exact command that fixes it:
#
#   1. df -h            → which mount is full?
#   2. df -i            → is it blocks or inodes? (the /tmp/sess dir should light this up)
#   3. du -xh --max-depth=1 (walked down)  → find the 2 GB unrotated log
#   4. lsof +L1         → find the phantom deleted-open 1 GB file
#   5. Fix each:  truncate the huge log (NOT rm — explain why),  delete the session files
#      (and explain how inodes free up),  truncate-or-restart to reclaim the phantom.
#   6. If Docker is installed: run `docker system df`, and write the one command you'd
#      run to reclaim build cache and dangling images — and note the risk in `--volumes`.
#   7. Finally: check `tune2fs -m` on a data volume to see how much space the 5% root
#      reservation is holding, and reclaim it to 1%.
#
# Deliverable: a numbered log of command → output → interpretation for the whole triage.
```

---

## Mental Model Checkpoint

1. **Draw the storage stack** from physical device to a file your app writes. If a cloud volume resize "didn't take," which layers might you have forgotten to grow, and with which commands?
2. **`df` and `du` disagree** — `df` says 96% full, `du` finds only half. Name the two classic causes, and the one command that finds the most common one.
3. **`touch` fails with "No space left on device" but `df -h` shows 40% used.** What's wrong, what command confirms it, and why can't you just delete a few big files to fix it?
4. **You `rm` a 20 GB log that your Node app still has open. Did `df` change?** Explain using directory entries vs inodes vs the superblock (topic 04). What's the correct way to reclaim the space *without* restarting the app?
5. **Why do we always use `UUID=` in /etc/fstab, and why must you run `mount -a` before rebooting after editing it?** What happens if you skip that and the line is wrong?
6. **A "100% full" disk still lets root write but your `node` user gets ENOSPC.** Why? What single command reclaims gigabytes on a large ext4 *data* volume?
7. **Your app suddenly can't write anything — every write is EROFS — with plenty of free space.** What did the kernel do, why, and what does `dmesg` / `smartctl` tell you about the real cause?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `lsblk -f` | **Best orientation command** — device tree + FSTYPE + UUID + mountpoint | `-f` (filesystem info) |
| `blkid` | UUIDs and types of block devices | (for filling in fstab) |
| `df -h` | Filesystem usage (from the superblock — instant) | `-h` human, `-T` show fs type |
| `df -i` | **INODE** usage — check when `df -h` looks fine but writes fail | the ENOSPC-mystery solver |
| `du -sh DIR` | Total size of a directory (walks the tree — slow) | `-s` summary, `-h` human |
| `du -h --max-depth=1 DIR \| sort -h` | Find which subdirectory is huge | `-x` stay on one fs |
| `lsof +L1` | Find deleted-but-still-open files (df/du disagreement) | topic 19 |
| `truncate -s 0 FILE` | Empty a file in place, keep the inode | the emergency log-reclaim |
| `mount /dev/x /mnt/y` | Attach a filesystem to the tree | `-o noatime,noexec,nofail`, `-o remount,rw` |
| `mount -a` | Mount everything in fstab — **test fstab safely** | run this, never a reboot, to validate fstab |
| `umount /mnt/y` | Detach | `-l` lazy (busy escape hatch); `fuser -vm` / `lsof +D` to find the blocker |
| `mkfs.ext4 /dev/x` | **Create a filesystem — DESTROYS the device** | `-L label`; `mkfs.xfs` for xfs |
| `fdisk -l` / `parted -l` | List partitions | MBR vs GPT |
| `growpart /dev/x N` | Grow a partition (note the space before N) | after a cloud volume resize |
| `resize2fs /dev/xN` | Grow (or shrink) an **ext4** filesystem | xfs uses `xfs_growfs /mount` (grow only) |
| `tune2fs -m 1 /dev/xN` | Change ext4 reserved-block % | `-l` to inspect; reclaim space on data volumes |
| `swapon --show` | Show active swap | `fallocate`+`mkswap`+`swapon` to add |
| `journalctl --disk-usage` | Journal size | `--vacuum-size=200M`, `--vacuum-time=7d` |
| `docker system df` | Docker's disk usage | `docker system prune -a --volumes` to reclaim |
| `smartctl -a /dev/x` | Disk hardware health (SMART) | look for FAILED / reallocated sectors |
| `iostat -xz 2` | Per-disk I/O + `%util` (topic 18) | `iotop -o` for per-process I/O |
| `dmesg -T` | Kernel messages — I/O errors, read-only remounts | `\| grep -iE 'error\|read-only'` |

---

## When Would I Use This at Work?

### Scenario 1: The deploy host that filled up overnight

Your CI deploys to a single app server via Docker. One morning the API is 503-ing and `df -h` shows `/` at 100%. You run the runbook: `df -i` (inodes fine), `du -xh --max-depth=1 /var/lib | sort -h` points at `/var/lib/docker` at 71 GB. `docker system df` shows 34 GB of build cache and 22 GB of dangling images from months of deploys — nothing ever pruned them. `docker builder prune -af` plus `docker image prune -a` reclaims 50 GB in ten seconds and the service recovers. You then add `docker system prune -af --filter 'until=168h'` to a weekly cron (topic 20) so it never happens again. **Build cache and dangling images are the #1 disk eater on a deploy host** — now you know to look there first.

### Scenario 2: "No space left on device" that makes no sense

Your Node app throws `ENOSPC` on every write, but `df -h` shows the volume 45% full and everyone's baffled. You run `df -i` and see inodes at 100%: a bug shipped last week writes one session file per request into `/var/lib/app/sessions` and never cleans them — 6 million tiny files. `df -h` had plenty of *space*; you were out of *inodes*. You stop the app, bulk-delete the session dir (`find ... -delete`), and inodes drop to 3%. The permanent fix is a TTL on sessions. Without knowing about `df -i`, this is a multi-hour "the filesystem is corrupted" wild goose chase.

### Scenario 3: The log that "won't delete"

At 2am `/` hits 100% and the app is down. `du` says the biggest thing is a 40 GB `/var/log/app.log`, so you `rm` it — and `df` **doesn't budge**. The app (still running) holds the fd open, so the blocks stay allocated even though the name is gone (topic 04, topic 19). You've now *also* broken logging, because the app is writing to a file with no directory entry. The right move you should have made: `truncate -s 0 /var/log/app.log` (frees the space, keeps the file the app is writing to), or `lsof +L1` → truncate the fd → then fix log rotation with `copytruncate` so it never recurs. Understanding deleted-but-open files turns a confusing 30-minute outage into a 30-second fix.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 32 — Deploy User Permissions | Who is allowed to write where on these filesystems — the permission layer on top of the disk |
| **Builds on** | 02 — Filesystem Hierarchy | Mount points graft filesystems onto the single `/` tree; `/var`, `/tmp`, `/boot` are often separate filesystems |
| **Builds on** | 03 — Everything Is a File | `/dev/sda`, `/dev/nvme0n1`, loop devices — block devices are files you can read/write raw |
| **Builds on** | 04 — Inodes in Depth | Inode exhaustion (`df -i`), and *the entire* deleted-but-open-file / df-vs-du story is inodes vs directory entries vs data blocks |
| **Builds on** | 09 — Working with Files | `du`, `df`, `stat`, and the `rm`-doesn't-free-an-open-file behavior first met here |
| **Builds on** | 18 — System Resources | `iostat`/`iotop` for I/O bottlenecks; swap and swappiness; tmpfs counting as memory pressure |
| **Builds on** | 19 — File Descriptors | `lsof +L1`, `/proc/<pid>/fd/<n>`, and why an open fd keeps deleted blocks alive |
| **Builds on** | 23 — Logs | Unrotated logs are the classic disk filler; `copytruncate` vs the deleted-open-file trap; `journalctl --vacuum` |
| **Used by** | 33 — Node in Production | Where your app writes logs/uploads, the volume it runs on, and the disk-space monitoring that keeps it alive |
| **Used by** | 34 — Performance Investigation | Disk `%util` and `await` as a bottleneck; a full disk as a root cause of a "slow" or crashing app |
