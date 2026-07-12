# 04 — Inodes in Depth

## ELI5 — The Simple Analogy

Imagine a **hospital**.

- Every patient gets a **medical record** on admission. It has a **record number** (`#8842`) and contains everything the hospital knows: blood type, allergies, when the chart was last updated, and **which storage lockers hold their X-rays**.
- The record does **NOT** contain the patient's name. It is identified only by its number.
- The name lives in a **list at the front desk**: `"John Smith" → #8842`. That's the only place the string "John Smith" exists.
- The **filing cabinet** holding all records was built when the hospital opened, with a **fixed number of slots**. Fill them and you cannot admit another patient **even if the X-ray lockers are completely empty**. You ran out of *records*, not *space*.
- Two names can point at the **same record**. Crossing one name off the front-desk list does not shred the record. The record is destroyed only when the **last** name is crossed off **AND** no doctor is currently holding the chart in their hands.

That last sentence is the entire reason a "deleted" 40 GB log file can keep your production disk 100% full.

| Hospital | Linux |
|---|---|
| Medical record | **inode** |
| Record number | **inode number** |
| Filing cabinet | **inode table** (fixed size, set at `mkfs` time) |
| Front-desk name list | **directory entry** |
| X-ray lockers | **data blocks** |
| Doctor holding the chart | an open **file descriptor** |
| Crossing a name off the list | **`unlink()` — which is all `rm` actually does** |

---

## Where This Lives in the Linux Stack

```
Hardware (disk platters / NAND cells — raw sectors, knows nothing about files)
  │
  └── KERNEL
       ├── VFS (Virtual File System) — the generic in-memory `struct inode`
       │
       └── Filesystem driver (ext4, xfs, btrfs) ◀◀◀ THIS TOPIC
            │   The ON-DISK inode lives here. This driver is what turns
            │   "inode #1179657" into "byte offset 0x3A800000 on /dev/sda1".
            │
            └── System Calls (stat, statx, open, unlink, link, rename)
                 │
                 └── C Library (glibc: stat(), unlink(), rename())
                      │
                      └── Shell (bash — expands globs; never sees an inode)
                           │
                           └── Commands (ls -i, stat, df -i, lsof, find -inum)
```

**Why it lives there:** the kernel needs ONE handle for "a file" that is independent of any name a human gave it — because a file can have zero names (open-but-deleted), one name, or fifty. The **inode number is that handle**.

---

## What Is This?

An **inode** ("index node") is a small, fixed-size struct on disk that describes **one file**: its type, permission bits, owner, group, size, timestamps, link count, and the addresses of its data blocks.

It does **NOT** store the filename, and does **NOT** store the path. Those live in *directories* — which are themselves just files whose contents are a list of `(name → inode number)` pairs.

Every file, directory, symlink, socket, FIFO, and device node has exactly one inode.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Inodes are a **fixed, finite** resource | Your app returns `ENOSPC: no space left on device` while `df -h` shows 52 GB free. You will stare at it for an hour. |
| `rm` only decrements a **link count** | You `rm` the 40 GB log, disk usage doesn't drop, and you take an outage restarting things for the wrong reason. |
| The filename is **not** in the inode | Hard links, instant `mv`, and `find -samefile` will never make sense. |
| `mv` across filesystems is copy+delete | Your "atomic" deploy that `mv`s a build from `/tmp` to `/var/www` is **not atomic**, and users get half-written files. |
| `ctime` ≠ creation time | You'll trust a `touch`-ed timestamp in an incident review and reach the wrong conclusion. |
| Directories have link counts | `ls -ld` showing `4` on an empty-looking directory will look like nonsense forever. |

---

## The Physical Reality

At `mkfs` time an ext4 filesystem is carved into **block groups**. Each carries a slice of the **fixed-size** inode table.

```
/dev/sda1 (ext4)
┌────────────┬───────────────────┬──────────────┬──────────────┬─────┐
│ Superblock │ Group Descriptors │ Block Group 0│ Block Group 1│ ... │
└────────────┴───────────────────┴──────┬───────┴──────────────┴─────┘
                                        ▼
   ┌──────────┬──────────┬──────────────────────┬───────────────────┐
   │  Block   │  Inode   │    INODE TABLE       │   DATA BLOCKS     │
   │  Bitmap  │  Bitmap  │  ┌────────────────┐  │  ┌───┬───┬───┐    │
   │ 1 bit    │ 1 bit    │  │ inode #11      │  │  │blk│blk│blk│    │
   │ per      │ per      │  │ inode #12      │  │  └───┴───┴───┘    │
   │ block    │ inode    │  │ inode #13 ...  │  │   4 KiB each,     │
   │          │          │  └────────────────┘  │   allocated on    │
   │          │          │  256 bytes each      │   demand          │
   │          │          │  ⚠ FIXED COUNT       │                   │
   └──────────┴──────────┴──────────────────────┴───────────────────┘
      ▲ these two bitmaps are what `df` reads. `du` never touches them.
```

**The inode itself (ext4, 256 bytes):**

```
┌──────────────── inode #1179657 ────────────────────────────────────────┐
│ i_mode        0x81A4  16 bits: [type][setuid/setgid/sticky][rwxrwxrwx] │
│                       → 0100644 = regular file, mode 644  (topic 05)   │
│ i_uid         1001    owner's NUMERIC id — NOT the string "deploy"     │
│ i_gid         1001    group's numeric id                               │
│ i_size        1284    logical length in BYTES                          │
│ i_blocks      8       allocated size in 512-byte SECTORS               │
│ i_links_count 1       ◀── HOW MANY DIRECTORY ENTRIES POINT HERE        │
│ i_atime               last time the DATA was read                      │
│ i_mtime               last time the DATA was written                   │
│ i_ctime               last time the INODE changed (perms/owner/links)  │
│ i_crtime              creation — ext4 only, needs a 256-byte inode     │
│ i_block[60 bytes] ──► the DATA BLOCKS                                  │
│                       (ext2/3: block pointers. ext4: an extent tree.)  │
│                                                                        │
│ ❌ NO FILENAME      ❌ NO PATH                                          │
└────────────────────────────────────────────────────────────────────────┘
```

**A directory is just a file whose data blocks hold a lookup table:**

```
Directory /var/www/app  (its own inode: #1179650)
Its data block contains:
   ┌────────────────┬──────────────┐
   │ name           │ inode number │
   ├────────────────┼──────────────┤
   │ "."            │   1179650    │ ← itself
   │ ".."           │   1179600    │ ← parent (/var/www)
   │ "server.js"    │   1179657    │
   │ "package.json" │   1179658    │
   └────────────────┴──────────────┘
     ▲ the ONLY place the string "server.js" exists on this disk
```

**The full chain, name to bytes:**

```
"/var/www/app/server.js"     the kernel walks it ONE COMPONENT AT A TIME
        │
  / (inode 2) ──"var"──► /var ──"www"──► /var/www ──"app"──► /var/www/app
  (needs +x)             (+x)            (+x)                 (+x)  ← topic 05
                                                                │
                                             "server.js" → 1179657
                                                                │
                                                                ▼
                                    ┌───────────────────────────────────┐
                                    │  INODE TABLE                      │
                                    │  seek to: table_start             │
                                    │         + (1179657-1) × 256 bytes │
                                    └───────────────┬───────────────────┘
                                                    │ read i_block[] / extents
                                                    ▼
                              ┌──────┬──────┬──────┬──────┐
                              │ blk  │ blk  │ blk  │ blk  │ ◀ YOUR FILE'S BYTES
                              │ 8192 │ 8193 │ 8194 │ 8195 │
                              └──────┴──────┴──────┴──────┘
```

---

## How It Works — Step by Step

### Block pointers — the classic ext2/ext3 model

`i_block` is 60 bytes = **15 slots** of 4 bytes. With 4 KiB blocks, one indirect block holds **1024** pointers.

```
 i_block[0]  ──► data block   ┐
 ...                          │ 12 DIRECT pointers → 48 KiB, zero extra reads
 i_block[11] ──► data block   ┘

 i_block[12] ──► ┌──────────────┐
   SINGLE        │1024 pointers │──► 1024 blocks = 4 MiB more.  1 extra read.
   INDIRECT      └──────────────┘

 i_block[13] ──► ┌──────────────┐   ┌──────────────┐
   DOUBLE        │1024 pointers │──►│1024 pointers │──► data. 1024² blocks
   INDIRECT      └──────────────┘   └──────────────┘   = 4 GiB more. 2 extra reads.

 i_block[14] ──► TRIPLE INDIRECT → 1024³ blocks = 4 TiB more. 3 extra reads.
```

**The problem:** a 1 GiB file needs ~262,144 pointers (~1 MiB of indirect blocks), and reading a byte near the end costs 3 extra disk reads *before* touching data. ext4 threw this out.

### Extents — what ext4 actually does

ext4 reuses the same 60 bytes to store an **extent tree**. One 12-byte extent says *"logical blocks 0–32767 live at physical block 8192, contiguously"* — describing up to **128 MiB** in 12 bytes.

```
ext4 i_block (60 bytes):
┌───────────────────────────────────────────────────────────┐
│ ext4_extent_header (12B)  magic=0xF30A  entries=2 depth=0 │
├───────────────────────────────────────────────────────────┤
│ extent 0 (12B): logical 0     → physical 8192   len 32768 │
│ extent 1 (12B): logical 32768 → physical 65536  len 18432 │
│ extent 2 (12B): (unused)                                  │
│ extent 3 (12B): (unused)                                  │
└───────────────────────────────────────────────────────────┘
  4 extents fit INSIDE the inode. Need more? depth becomes > 0 and these
  turn into INDEX entries pointing at extent blocks — a B+tree, not a chain.
```

`filefrag -v FILE` prints them. A contiguous 200 MB file needs 2 extents; ext2 would have needed 51,200 pointers.

### What `rm` actually does

```
COMMAND:  rm /var/www/app/server.js
SHELL:    fork(), exec /usr/bin/rm
SYSCALLS: newfstatat("/var/www/app/server.js", ...)   ← is it there? is it a dir?
          unlinkat(AT_FDCWD, "/var/www/app/server.js", 0)

KERNEL (inside unlinkat):
  1. Resolve the PARENT directory /var/www/app → inode #1179650
  2. Permission check: do I have  w AND x  on the PARENT DIRECTORY?
     ⚠ It does NOT check permissions on server.js itself. (Topic 05, Trap B.)
  3. Remove the ("server.js" → 1179657) entry from the parent's data block
  4. Load inode #1179657:   i_links_count -= 1
  5. IF i_links_count == 0:
        AND no process has it open (in-kernel reference count == 0):
            → mark data blocks FREE in the block bitmap
            → mark the inode  FREE in the inode bitmap
            → SPACE IS ACTUALLY RECLAIMED
        BUT if a process still holds it open:
            → the NAME is gone; the BLOCKS ARE NOT FREED
            → the inode lives on, reachable by no path, until the last fd closes
RETURNS:  0
```

**Step 5 is the whole ballgame.** `rm` is not "delete." `rm` is "remove one name." The syscall is literally `unlink`.

### What `mv` actually does

```
mv /var/www/app/old.js /var/www/app/new.js          ← SAME filesystem
  SYSCALL: renameat2(AT_FDCWD,"old.js", AT_FDCWD,"new.js", 0)
  KERNEL:  edits ONE directory entry. Data blocks untouched. Inode UNCHANGED.
  COST:    microseconds — whether the file is 1 KB or 100 GB.
  ATOMIC:  yes.

mv /tmp/build.tar.gz /var/www/releases/             ← DIFFERENT fs (/tmp is often tmpfs!)
  SYSCALL: renameat2(...) → -EXDEV ("Invalid cross-device link")
  mv FALLS BACK TO: open(src) + open(dst,O_CREAT) + copy loop
                    + fchmod + fchown + utimensat + unlink(src)
  KERNEL:  allocates a BRAND NEW INODE on the destination filesystem.
  COST:    proportional to size. Seconds or minutes.
  ATOMIC:  ABSOLUTELY NOT — readers can see a partial file.
```

This is why the correct atomic deploy is: unpack the release **onto the same filesystem** as the live directory, then swap a **symlink** (`ln -sfn`, which is implemented as a `rename()` and *is* atomic). Never `mv` across a mount boundary and call it atomic.

---

## Reading `stat` — Field by Field

```bash
stat /var/www/app/server.js
```
```
  File: /var/www/app/server.js
  Size: 1284       Blocks: 8          IO Block: 4096   regular file
Device: 830h/2096d Inode: 1179657     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1001/  deploy)   Gid: ( 1001/  deploy)
Access: 2026-07-12 09:14:02.123456789 +0000
Modify: 2026-07-10 22:41:18.000000000 +0000
Change: 2026-07-10 22:41:18.000000000 +0000
 Birth: 2026-07-10 22:41:18.000000000 +0000
```

| Field | What it really is | Gotcha |
|---|---|---|
| `File:` | The path **you typed** — echoed back from argv | This string is **not** in the inode |
| `Size: 1284` | `i_size`, logical length in bytes | A sparse file reports a huge Size with almost no Blocks |
| `Blocks: 8` | Allocated **512-byte sectors** | 8 × 512 = 4096 = one 4 KiB block. Different unit from `IO Block`! |
| `IO Block: 4096` | Filesystem block size | The minimum allocation unit — a 214-byte file still burns 4 KiB |
| `regular file` | Decoded from the top 4 bits of `i_mode` | Or `directory`, `symbolic link`, `socket`, `fifo`, `character special file` |
| `Device: 830h` | Major/minor of the device holding this fs | True identity of a file is the pair **(device, inode)** — see Mistake 5 |
| `Inode: 1179657` | Index into the inode table | Unique **per filesystem**, not globally |
| `Links: 1` | `i_links_count` | `2` on a fresh directory. `>1` on a file means hard links exist. |
| `Access: (0644/…)` | The low 12 bits of `i_mode` | Topic 05 dissects this in full |
| `Uid: (1001/deploy)` | `i_uid`. The **number** is on disk | `stat` looks the name up in `/etc/passwd`. Delete the user → prints a bare `1001`. Inside a container, 1001 is a *different human*. |
| `Access:` (time) | **atime** — last read of the data | Under the default `relatime` mount option this is only refreshed if it was already older than mtime/ctime or older than 24h. Do not treat it as precise. |
| `Modify:` (time) | **mtime** — last write to the data | `touch -m -d "2019-01-01" f` will happily set this to any lie |
| `Change:` (time) | **ctime** — last change to the *inode* | Set by chmod, chown, rename, link/unlink, **and** any data write. **There is no syscall to set it arbitrarily.** The one timestamp you can trust. |
| `Birth:` (time) | **crtime/btime** — creation | ext4 *does* store it (in a 256-byte inode's extra fields), exposed only via `statx()` (kernel 4.11+). Prints `-` on ext3 / 128-byte inodes / older coreutils. Never settable. |

**The ctime trap, live:**
```bash
touch -d "2019-01-01" evidence.log        # attacker backdates the file
stat -c 'mtime=%y%nctime=%z' evidence.log
# mtime=2019-01-01 00:00:00.000000000 +0000   ← the lie
# ctime=2026-07-12 14:03:51.882110000 +0000   ← the truth: the INODE changed today
```

---

## Exact Syntax Breakdown

```
ls -li /var/www/app
│  ││ │
│  ││ └── directory to list
│  │└── -l = long format
│  └── -i = print the INODE NUMBER first
└── ls

1179657 -rw-r--r--  1 deploy deploy  1284 Jul 10 22:41 server.js
│       │           │ │      │       │    │            │
│       │           │ │      │       │    │            └── the NAME — lives in the
│       │           │ │      │       │    │                DIRECTORY, not the inode
│       │           │ │      │       │    └── mtime
│       │           │ │      │       └── i_size
│       │           │ │      └── i_gid  → name via /etc/group
│       │           │ └── i_uid  → name via /etc/passwd
│       │           └── i_links_count ◀◀◀ THE LINK COUNT
│       └── i_mode, decoded
└── i_ino ◀◀◀ THE INODE NUMBER
```
```
df -i /var
│  │  │
│  │  └── any path — df reports on the filesystem CONTAINING it
│  └── -i = report INODES instead of blocks
└── df ("disk free")

Filesystem       Inodes   IUsed  IFree IUse% Mounted on
/dev/sda1       6553600 6553600      0  100% /
                │                     │      │
                │                     │      └── ⚠ you cannot create ANY new file
                │                     └── zero left
                └── FIXED at mkfs. resize2fs grows BLOCKS, never the inode table.
```
```
lsof +L1
│    │
│    └── +L1 = open files whose LINK COUNT is < 1 — i.e. unlink()ed but STILL HELD OPEN
└── lsof ("list open files")

COMMAND  PID   USER  FD  TYPE DEVICE   SIZE/OFF NLINK    NODE NAME
node    8234 deploy  3w   REG    8,1 42949672960   0 1179912 /var/log/app.log (deleted)
│       │            │                │            │  │      │
│       │            │                │            │  │      └── the kernel appends "(deleted)"
│       │            │                │            │  └── the ORPHANED inode
│       │            │                │            └── NLINK 0 → NO name points here
│       │            │                └── 40 GiB still allocated on disk
│       │            └── fd 3, open for writing
│       └── the process holding it hostage
└── it's your Node app
```
```
find /var/www -samefile /var/www/app/server.js      # every hard link of that file
              └── -samefile matches (device AND inode) — the correct, safe form

find /var/www -xdev -inum 1179657                   # the old-school equivalent
                    └── -inum matches an inode NUMBER, which is only unique WITHIN
                        one filesystem — hence -xdev to stay on one.
```
```
tune2fs -l /dev/sda1 | grep -i inode
│       │  │
│       │  └── the BLOCK DEVICE (not a mount point!)
│       └── -l = dump the superblock
└── tune2fs — tune ext2/3/4 parameters

Inode count:   6553600   ← the FIXED total, decided at mkfs
Free inodes:   12        ← what's left
First inode:   11        ← 1–10 are reserved (inode 2 = "/", 11 = lost+found)
Inode size:    256       ← bytes. 128 on old ext2/3 — no room for crtime!
```

`debugfs -R "stat <1179657>" /dev/sda1` dumps the **raw on-disk inode**, extent tree and all. Read-only, safe, and the ground truth when you want to *see* the struct rather than trust a tool.

---

## Example 1 — Basic: Prove the Filename Is Not in the Inode

```bash
mkdir -p /tmp/inode-lab && cd /tmp/inode-lab
echo "hello inodes" > original.txt

ls -li original.txt
# 262145 -rw-r--r-- 1 deploy deploy 13 Jul 12 14:20 original.txt
#   │                │
#   │                └── Links = 1. ONE name points at this inode.
#   └── inode 262145

ln original.txt second-name.txt      # HARD link. No -s. A second NAME, same inode.

ls -li
# 262145 -rw-r--r-- 2 deploy deploy 13 Jul 12 14:20 original.txt
# 262145 -rw-r--r-- 2 deploy deploy 13 Jul 12 14:20 second-name.txt
#   │                │
#   │                └── link count jumped to 2. ONE inode, TWO directory entries.
#   └── SAME inode number. There is ONE file here, not a copy.

echo "written via second-name" >> second-name.txt
cat original.txt
# hello inodes
# written via second-name       ← the data appeared through the OTHER name

rm original.txt                  # "delete" one name
ls -li
# 262145 -rw-r--r-- 1 deploy deploy 37 Jul 12 14:22 second-name.txt
#                   │
#                   └── link count fell 2 → 1. THE DATA IS FINE. rm deleted nothing.

# --- Directory link counts ---
mkdir -p parent/childA parent/childB
ls -ld parent
# drwxr-xr-x 4 deploy deploy 4096 Jul 12 14:25 parent
#            │
#            └── 4 = parent's own "."  +  the "parent" entry in /tmp/inode-lab
#                  +  childA's ".."    +  childB's ".."
#            RULE: a directory's link count = 2 + (number of subdirectories)
rmdir parent/childB && ls -ld parent   # now 3
```

---

## Example 2 — Production Scenario

**2:14 AM.** Your Node API throws `ENOSPC: no space left on device, write` on every upload. You SSH in.

```bash
df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        97G   41G   52G  44% /
#                              ^^^^  ^^^  52 GB free. THE DISK IS NOT FULL.
```

Most engineers stop here and lose 40 minutes. You don't — because inodes are a **separate, fixed** pool, and the kernel returns the **same `ENOSPC`** when either one runs dry.

```bash
df -i /
# Filesystem       Inodes   IUsed  IFree IUse% Mounted on
# /dev/sda1       6553600 6553600      0  100% /
#                                      ^^      ^^^^  ZERO inodes left. THERE it is.

# Who ate 6.5 million inodes? (Each directory entry ≈ one inode.)
du -a --inodes -d 2 /var 2>/dev/null | sort -rn | head -4
# 6104882  /var
# 6093110  /var/lib/myapp
# 6093004  /var/lib/myapp/sessions     ◀◀◀
# 2140     /var/log

ls /var/lib/myapp/sessions | head -2      # sess_00a1f4c9b2e77d31.json
stat -c '%s bytes, inode %i' /var/lib/myapp/sessions/sess_00a1f4c9b2e77d31.json
# 214 bytes, inode 4194891
```

Six million 214-byte session files. Real data: ~1.3 GB — nothing. But **six million inodes**, and each file also burns a full 4 KiB block (the minimum allocation unit), so `du` reports ~24 GB while the byte sum is 1.3 GB. The session store had no TTL sweep.

```bash
# FIX. Do NOT run `rm -rf sessions/*` — the shell would expand six million names
# into one argv and die with "Argument list too long" (E2BIG).
find /var/lib/myapp/sessions -type f -mtime +1 -delete
#                                              │
#                                              └── find calls unlinkat() itself.
#                                                  No exec, no argv, no limit.

watch -n2 'df -i /'      # in a second session — watch IFree climb back
curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/health    # 200
```

**Prevention, same night:**
```bash
echo '17 * * * * find /var/lib/myapp/sessions -type f -mmin +120 -delete' \
  | sudo tee /etc/cron.d/session-sweep
df -i / --output=ipcent | tail -1 | tr -d ' %'   # ← feed THIS to your metrics agent.
                                                 #   Most monitoring only watches df -h.
```

**Six hours later, the second half of the same incident.** `df -h` says 98% again — but `du -sh /` only accounts for 41 GB. **`du` walks names. A file with no name is invisible to it.**

```bash
sudo lsof +L1
# COMMAND  PID   USER  FD  TYPE DEVICE   SIZE/OFF NLINK    NODE NAME
# node    8234 deploy  3w   REG    8,1 42949672960   0 1179912 /var/log/app.log (deleted)
#                                      ^^^^^^^^^^^   ^
#                                      40 GiB        NLINK = 0
```

Someone (you, at 2 AM) ran `rm /var/log/app.log`. But the Node process still holds fd 3 on it. The name is gone; the inode and its 40 GiB **are not, and cannot be**, until that fd closes. The kernel is contractually obliged to keep serving it.

```bash
ls -l /proc/8234/fd/3
# l-wx------ 1 deploy deploy 64 Jul 12 08:31 /proc/8234/fd/3 -> /var/log/app.log (deleted)

cp /proc/8234/fd/3 /var/log/app.log.recovered   # you can still READ it — the fd is live

sudo truncate -s 0 /proc/8234/fd/3    # ◀ THE FIX. Frees the blocks INSTANTLY.
                                      #   The process keeps writing to the same inode.
df -h /
# /dev/sda1  97G  1.4G  91G   2% /    ← 40 GiB back. No restart. No outage.
```

**The rule:** never `rm` a log a live process is writing to. **Truncate** it (`truncate -s 0 f` or `: > f`), or let `logrotate` do it with `copytruncate` (topic 23).

---

## Common Mistakes

### Mistake 1 — "The disk isn't full, so ENOSPC must be a bug in my code"

**Wrong:** `df -h` → 44% → "must be an app bug" → read Node source for an hour.
**Right:** `df -h && df -i` — **always both.** They are two independent pools.

**Root cause:** the inode table is allocated once, at `mkfs`, sized by the bytes-per-inode ratio (default `16384` in `/etc/mke2fs.conf` — one inode per 16 KiB of filesystem). A 100 GB disk therefore gets ~6.5M inodes. Six million tiny files exhaust that while using ~1% of the bytes. `ext4_new_inode()` fails and `open(O_CREAT)` returns `-ENOSPC` — the **same errno** as a full disk.

**Fix:** delete files (`find … -delete`), or move the churn to a filesystem sized for it.
**Prevention:** alert on `df -i`. If you *know* a volume will hold millions of tiny files, format it denser: `mkfs.ext4 -i 4096 /dev/sdb1` (4× the inodes). **You cannot add inodes to ext4 later** — `resize2fs` grows blocks, not the inode table. **XFS allocates inodes dynamically** and simply does not have this failure mode, which is exactly why it's the usual pick for mail/session/cache/Docker volumes.

---

### Mistake 2 — "`rm` deletes the file"

**Wrong:** `rm /var/log/app.log` to free space while node/nginx is writing to it.
**Right:** `truncate -s 0 /var/log/app.log` (or `: > /var/log/app.log`).

**Root cause:** `rm` calls `unlink()`, which removes a *directory entry* and decrements `i_links_count`. Blocks are freed only when **both** `i_links_count == 0` **and** the in-kernel open-reference count is 0. Your process still holds an fd → the inode is orphaned but alive, its blocks still marked used in the **block bitmap** (so `df` counts them) but reachable by no name (so `du`, which walks directory entries, **cannot see them**). That is the entire `df` vs `du` mystery.

**Diagnose:** `sudo lsof +L1`, or `sudo lsof -nP | grep '(deleted)'`.
**Fix:** `sudo truncate -s 0 /proc/<pid>/fd/<n>` — instant, no restart. Restarting the process also works (it closes the fd) but costs an outage.
**Prevention:** `logrotate` with `copytruncate`, or make the app reopen its log on `SIGHUP`.

---

### Mistake 3 — "`mv` is always instant and atomic"

**Wrong:**
```bash
mv /tmp/build /var/www/app-new      # /tmp is tmpfs (RAM) — a DIFFERENT filesystem
```
**Right:**
```bash
tar -xzf build.tar.gz -C /var/www/releases/v42     # same fs as the live dir
ln -sfn /var/www/releases/v42 /var/www/current     # THE SYMLINK SWAP IS ATOMIC
```

**Root cause:** `rename()` can only rewrite a directory entry **within one filesystem** — you cannot conjure an inode on a device that doesn't own the source's blocks. Across a mount boundary the kernel returns `EXDEV` and `mv` silently degrades to copy-then-unlink: new inode, new blocks, byte-by-byte copy, and a window where the destination is half-written.

**Diagnose:** `df /tmp /var/www` — a different `Filesystem` column means a different device means **not atomic**. Or `stat -c %i` before and after: a changed inode number proves a copy happened.
**Prevention:** stage releases on the target's own filesystem and flip a symlink.

---

### Mistake 4 — "ctime is the creation time"

**Wrong:** "ctime is last Tuesday, so it was created last Tuesday."
**Right:** ctime = **inode change** time. A `chmod` on Tuesday sets it. Creation in 2019 does not survive in it.

**Root cause:** POSIX defines three timestamps and **none of them is creation**. ext4 does store `i_crtime`, but only in the extra fields of a **256-byte** inode (ext3's 128-byte inode has literally nowhere to put it), and it is reachable only via `statx()` — which is why older `stat` builds print `Birth: -`.

| | Set by | Forgeable with `touch`? |
|---|---|---|
| **atime** | reading the data | yes (`touch -a`) |
| **mtime** | writing the data | yes (`touch -m -d`) |
| **ctime** | chmod, chown, rename, link/unlink, **and** any write | **no** |
| **btime** | creation, once | no — and not settable at all |

**Use it right:** `stat -c 'a=%x m=%y c=%z birth=%w' file`. In an incident or a security review, **ctime is the one you trust** — an attacker who backdates `mtime` leaves ctime pointing at the moment they touched it.

---

### Mistake 5 — "Same inode number means same file"

**Wrong:** `find / -inum 262145` and assuming every hit is the same file.
**Right:** inode numbers are unique **per filesystem**. True identity is the pair `(st_dev, st_ino)`.

```bash
stat -c '%d:%i  %n' /home/deploy/a.txt /var/lib/docker/b.txt
# 2049:262145  /home/deploy/a.txt
# 2065:262145  /var/lib/docker/b.txt   ← same inode number, DIFFERENT device. Unrelated files.
```
Use `find -samefile` (compares both) or `find -xdev -inum N` (stays on one filesystem).

**Bonus — hard link vs symlink, since they get confused here:**

| | Hard link (`ln a b`) | Symlink (`ln -s a b`) |
|---|---|---|
| What it is | A second **directory entry** → same inode | A **separate inode** whose data is the *string* `"a"` |
| Own inode? | No | Yes |
| Bumps the target's link count? | **Yes** | No |
| Survives deleting the original name? | **Yes** — data is fine | No — dangles |
| Crosses filesystems? | **No** (`EXDEV`) | Yes |
| Points at a directory? | **No** (would create loops) | Yes |

A hard link isn't a *link* at all — it's just another name. There is no "original"; nothing on disk records which came first.

---

## Hands-On Proof

```bash
# PROVE IT: the root of every ext filesystem is inode 2. Always.
stat -c '%i %n' /                          # 2 /

# PROVE IT: a directory's data IS a name→inode table.
mkdir /tmp/d && touch /tmp/d/alpha /tmp/d/beta && ls -ai /tmp/d
#  262200 .    262146 ..    262201 alpha    262202 beta
#  ^^^^^^      ^^^^^^       the directory's contents, literally

# PROVE IT: rm calls unlink(), not "delete".
strace -e trace=unlink,unlinkat rm /tmp/d/alpha
# unlinkat(AT_FDCWD, "/tmp/d/alpha", 0) = 0

# PROVE IT: mv within a filesystem is rename() — the inode never changes.
touch /tmp/x; stat -c %i /tmp/x            # 262210
mv /tmp/x /tmp/y; stat -c %i /tmp/y        # 262210  ← SAME inode
strace -e trace=renameat2 mv /tmp/y /tmp/z 2>&1 | grep rename
# renameat2(AT_FDCWD, "/tmp/y", AT_FDCWD, "/tmp/z", RENAME_NOREPLACE) = 0

# PROVE IT: deleted-but-open files still hold disk space. (THE BIG ONE.)
cd /tmp
dd if=/dev/zero of=big.bin bs=1M count=500 status=none
df -h /tmp | tail -1        # note "Used"
exec 9< big.bin             # YOUR SHELL now holds fd 9 open on it
rm big.bin                  # unlink the ONLY name
ls big.bin                  # No such file or directory
du -sh /tmp                 # does NOT see the 500 MB — no name left to walk
df -h /tmp | tail -1        # DOES — the blocks are still allocated!
sudo lsof +L1 | grep big    # NLINK 0, "(deleted)"
exec 9<&-                   # close the fd...
df -h /tmp | tail -1        # ...500 MB freed THE INSTANT the last fd closed.

# PROVE IT: the inode table is fixed and finite.
sudo tune2fs -l "$(df --output=source / | tail -1)" | grep -E 'Inode count|Free inodes|Inode size'
df -i /                     # the same numbers, different presentation

# PROVE IT: ext4 uses extents, not indirect chains.
dd if=/dev/zero of=/tmp/frag.bin bs=1M count=200 status=none
filefrag -v /tmp/frag.bin | head -6
#  ext:   logical_offset:   physical_offset: length:  expected: flags:
#    0:        0.. 32767:  1116160..1148927:  32768:
#    1:    32768.. 51199:  1148928..1167359:  18432:            last,eof
# TWO extents describe 200 MB. ext2 would have needed ~51,200 pointers.

# PROVE IT: ctime cannot be forged.
touch /tmp/forge && sleep 1 && touch -m -d "2019-01-01 00:00:00" /tmp/forge
stat -c 'mtime=%y%nctime=%z' /tmp/forge
# mtime=2019-01-01 ...   ← the lie
# ctime=2026-07-12 ...   ← today. Always.
```

**macOS trap (you're on macOS locally):** `ls -i`, `lsof +L1`, `find -inum` and `df -i` all work. But `stat` is BSD — it's `stat -f '%i %N'`, **not** `stat -c '%i %n'`. There is no `tune2fs`, `debugfs`, `filefrag`, or `truncate`. APFS has **no fixed inode table**, so you can never reproduce `df -i` exhaustion locally. Run these in Linux: `docker run --rm -it ubuntu:22.04 bash`.

**Alpine/BusyBox trap:** BusyBox `stat` accepts `-c` but supports fewer format specifiers. `lsof`, `strace`, `tune2fs` are not installed: `apk add lsof strace e2fsprogs`.

---

## Practice Exercises

### Exercise 1 — Easy

```bash
mkdir -p /tmp/ex1 && cd /tmp/ex1
echo "config v1" > config.json
ln config.json config.bak
mkdir logs cache
```
Using only `ls -li`, `ls -ld` and `stat`:
1. What is `config.json`'s inode number and link count?
2. What is `/tmp/ex1`'s own link count? Derive **exactly** where each link comes from.
3. `rm config.json`, then `cat config.bak`. Explain in one sentence why the data survived, using the words *directory entry* and *link count*.

Paste your terminal output.

### Exercise 2 — Medium

Reproduce and diagnose the deleted-but-open-file incident.
```bash
cd /tmp
# 1. Create a 200 MB file. Record `df -h /tmp` Used AND `du -sh /tmp`.
# 2. Hold it open:   tail -f bigfile.log &
# 3. rm bigfile.log
# 4. Compare `df -h /tmp` and `du -sh /tmp`. They now DISAGREE. By how much?
# 5. Find the culprit with `lsof +L1`. Note the PID and the fd number.
# 6. Free the space WITHOUT killing the process.
# 7. Prove df dropped.
```
**Question:** which of `df` and `du` reads the *block bitmap*, and which walks *directory entries*? Why does exactly one of them see the orphaned inode?

### Exercise 3 — Hard (Production Simulation)

Manufacture, diagnose, and fix inode exhaustion — safely, on a loopback filesystem.

```bash
dd if=/dev/zero of=/tmp/tiny.img bs=1M count=64 status=none
mkfs.ext4 -N 512 -F /tmp/tiny.img        # -N 512 = force only 512 inodes, total
sudo mkdir -p /mnt/tiny && sudo mount -o loop /tmp/tiny.img /mnt/tiny
sudo chown "$USER" /mnt/tiny

# YOUR JOB:
# 1. Record `df -h /mnt/tiny` AND `df -i /mnt/tiny` BEFORE.
# 2. Exhaust the inodes with a loop creating ~600 tiny files.
#    Capture the EXACT error message.
# 3. Prove the disk is nearly EMPTY (df -h) while creation fails (df -i).
# 4. Show you can still APPEND to an EXISTING file. Explain why that works when
#    creating a new one does not. (Which operation needs a NEW inode?)
# 5. Confirm the total with `tune2fs -l /tmp/tiny.img | grep -i inode`.
# 6. Free inodes, prove creation works again.
# 7. Write a one-line health check that alerts when IUse% > 80,
#    using `df -i --output=ipcent`.

sudo umount /mnt/tiny && rm /tmp/tiny.img   # tear down
```
**Deliverable:** the transcript, plus a 3-sentence incident summary as you'd post it in Slack: symptom, root cause, fix.

---

## Mental Model Checkpoint

1. **Name three things an inode stores and two things it definitively does NOT store.** Where does the filename actually live?
2. **Walk the chain** from the string `/var/www/app/server.js` to the bytes on disk. What is consulted at each step?
3. **`df -h` says 44% used but `open()` returns `ENOSPC`.** What command do you run next? What resource is exhausted, and why can't you add more of it to an ext4 volume?
4. **What syscall does `rm` actually call, and what are the TWO conditions that must BOTH hold before a file's data blocks are freed?**
5. **A fresh empty directory has link count 2 — where do those two links come from?** You `mkdir` three subdirectories inside it. What is the count now, and why?
6. **What is the difference between mtime and ctime, and which one can an attacker forge with `touch`?**
7. **Why is `mv` from `/tmp` to `/var/www` slow and non-atomic, while `mv` within `/var/www` is instant and atomic?** Name the syscall and the errno.

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `ls -i` | Show inode numbers | `-li` inode + long; `-ai` include dotfiles |
| `stat FILE` | Dump the whole inode, human-readable | `-c '%i %h %s %n'` (GNU); macOS/BSD uses `-f` |
| `stat -c` | Custom format | `%i` inode, `%h` links, `%s` size, `%a` octal mode, `%U`/`%G` owner/group, `%y` mtime, `%z` ctime, `%w` birth, `%d` device |
| `df -h` | Free **BLOCKS** (bytes) | `-h` human, `-T` show fs type |
| `df -i` | Free **INODES** ◀ the one everyone forgets | `--output=ipcent` for just the percent |
| `du -a --inodes -d 2 D` | Count *inodes* per subtree, not bytes | needs coreutils 8.28+ |
| `ln a b` | **Hard** link — second name, same inode | — |
| `ln -s a b` | **Sym**link — new inode holding a path string | `-f` force, `-n` don't follow an existing symlink-to-dir |
| `find -samefile F` | All hard links of F (matches device **and** inode) | safer than `-inum` |
| `find -inum N` | Match by inode number | pair with `-xdev` |
| `find D -type f -delete` | Delete millions of files with no argv explosion | `-mtime +N`, `-mmin +N` |
| `lsof +L1` | Open files with link count < 1 — **deleted but held** | `-nP` skips DNS/port lookups (much faster) |
| `truncate -s 0 F` | Free a file's blocks, keep the inode and fds valid | works on `/proc/PID/fd/N` |
| `tune2fs -l DEV` | Superblock: inode count, free inodes, inode size | ext only; takes a **device**, not a mountpoint |
| `debugfs -R "stat <N>" DEV` | Dump the raw on-disk inode, extents and all | read-only by default; ext only |
| `filefrag -v F` | Show a file's **extents** | ext4/xfs/btrfs |

---

## When Would I Use This at Work?

### Scenario 1: Docker eats your build server
CI fails with `no space left on device`, but `df -h /` shows 60 GB free. `df -i /` shows 100%. Every image layer and every `node_modules` (≈40,000 files *per project, per layer*) is thousands of inodes. `docker system df -v` confirms it; `docker system prune -af --volumes` frees millions. Long-term: put `/var/lib/docker` on its own **XFS** volume (dynamic inodes), or ext4 with `-i 4096`.

### Scenario 2: The 2 AM "I deleted the log but the disk is still full"
The nginx access log hits 60 GB. Someone `rm`s it. `df` doesn't budge. You run `sudo lsof -nP +L1`, see `nginx … (deleted) NLINK 0`, and `truncate -s 0 /proc/<pid>/fd/<n>`. Space back in under a second, zero downtime. Then you fix `logrotate` so it never recurs. This incident happens at every company, forever.

### Scenario 3: Your zero-downtime deploy isn't
Your deploy builds into `/tmp` (a **tmpfs — RAM, a different filesystem** on most systemd distros) and `mv`s the result into `/var/www/app`. Crossing the boundary makes it a `copy + unlink`, not a `rename`, so for 3 seconds `/var/www/app` is half-written and users get 500s. `df /tmp /var/www` shows two devices. You restructure to build into `/var/www/releases/<sha>` and flip a symlink with `ln -sfn` — same filesystem, one `rename()`, zero broken requests.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | The inode lives in the *kernel's* filesystem driver; `stat`/`unlink` are syscalls crossing the Ring 3 → Ring 0 boundary |
| **Builds on** | 02 — The Filesystem Hierarchy | Path resolution walks `/` → `/var` → `/var/www` — every step is a directory-entry lookup |
| **Builds on** | 03 — Everything Is a File | The top 4 bits of `i_mode` are what *make* something a directory, socket, FIFO, or device node |
| **Next** | 05 — File Permissions in Depth | Mode, UID, and GID **are inode fields**. Permissions are literally 12 bits of the struct you just met. |
| **Used by** | 09 — Working with Files | `cp` (new inode) vs `mv` (same inode) vs `ln` (extra name) finally make sense |
| **Used by** | 19 — File Descriptors | An fd is an in-kernel reference to an inode — precisely why an open fd keeps an `unlink()`ed file alive |
| **Used by** | 23 — Logs and Log Management | `logrotate`'s `copytruncate` exists *only* because `rm` on an open log frees nothing |
| **Used by** | 31 — Disk Management | `df -h` vs `df -i` is the first fork in every "disk full" decision tree |
