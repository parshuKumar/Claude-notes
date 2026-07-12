# 09 — Working with Files

## ELI5 — The Simple Analogy

Picture a giant **shipping warehouse**.

- **The warehouse floor** is the disk. Goods sit on it in pallets. Those pallets are the **data blocks** — the actual bytes of your file.
- **The record card** is the **inode**. One card per pallet-group. It says: how big, who owns it, when it was last touched, and exactly which floor slots hold the goods. The card does **not** have a name on it.
- **The index room** is a **directory**. It's a wall of shelves, and on each shelf is a **sticky label**: `"api.log" → card #1310725`. That label is the **filename**. A name is a *pointer to a card*, nothing more.

Now every command in this topic is a warehouse action:

| Command | What the warehouse worker does |
|---|---|
| `touch` | Prints a blank record card, sticks a label on a shelf — or, if the label already exists, just re-stamps today's date on the card |
| `mkdir` | Builds a new shelf in the index room |
| `cp` | Forklift. Physically duplicates every pallet, prints a **new** card. Slow. Costs floor space. |
| `mv` (same warehouse) | Peels the label off shelf A, sticks it on shelf B. **The goods never move.** Instant. |
| `mv` (to a warehouse across town) | Loads a truck, drives, unloads, then shreds the old label. Slow. If the truck crashes, half the goods are in limbo. |
| `rm` | Peels the label off and bins it. **The pallet is still sitting on the floor.** The floor manager only sends it to the crusher when the *last* label is gone **AND** no worker is standing at the pallet with a clipboard. |
| `du` | You, walking the floor with a notepad, adding up every pallet you can find a label for. |
| `df` | Asking the **floor manager** how much free floor space there is. |

And there, in the last two rows, is the single most-asked production question in this entire curriculum:

> **A pallet with no label, but a worker still standing at it, is invisible to your notepad and very much taking up floor space.**

That is why `du` says 8 GB and `df` says 100% full. Hold onto this image. We will cash it in at 2am.

---

## Where This Lives in the Linux Stack

```
Hardware (spinning platters / NAND cells — where the pallets physically are)
  └── KERNEL
       │   • Filesystem driver (ext4/xfs/overlayfs): owns the inode table,
       │     the block allocator, the free-space counters in the superblock
       │   • VFS layer: turns "unlink this path" into "decrement this inode"
       │
       └── System Calls  ◀◀◀ THIS TOPIC IS A THIN SKIN OVER ~10 OF THEM
            │   openat(O_CREAT)  utimensat()  mkdir()  rmdir()
            │   rename()/renameat2()  unlink()/unlinkat()  truncate()
            │   statx()/stat()  statfs()  copy_file_range()
            │
            └── C Standard Library (glibc wraps each one)
                 │
                 └── Shell (expands globs and $VARS *before* the command runs
                      — this is where `rm -rf $DIR/` becomes `rm -rf /`)
                      │
                      └── Commands  ◀◀◀ AND THIS TOPIC
                           touch mkdir rmdir cp mv rm file stat du df
                           truncate shred install ln rename
```

**The whole point of this topic:** these commands are *tiny*. `mv` is one syscall. `rm` is one syscall. All the danger and all the power lives in what the **kernel** does with that syscall — and in what the **shell** did to your arguments before the command ever started.

---

## What Is This?

The complete lifecycle of a file: create it, copy it, rename/move it, delete it, inspect it, and measure it. On Linux these are not ten unrelated tools — they are ten different ways of manipulating **two things**: a *directory entry* (a name → inode number pair) and an *inode* (the metadata + block pointers).

Once you internalise that split, `mv` being instant, `rm` not freeing space, and `du` disagreeing with `df` all stop being trivia and become obvious.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `mv` is `rename()` within a filesystem | You'll copy 4 GB of `node_modules` when you could have moved it in 40 microseconds — and you'll wonder why your Docker build is slow |
| `rename()` is **atomic** | Your config reload will read a half-written file, your app will crash on `JSON.parse`, and you'll blame Node |
| `rm` is `unlink()`, not "erase" | You'll `rm` a 27 GB log to free space, `df` will still say 100%, and you will lose an hour at 2am |
| `cp -r` ≠ `cp -a` | Your "backup" will silently reset every file's owner to you and every mtime to now — and your restore will break `logrotate`, `git status`, and `make` |
| The shell expands `$VAR` before `rm` sees it | `rm -rf $DIR/` with `DIR` unset. That is the whole sentence. |
| `du` walks names, `df` reads the superblock | You will not be able to answer "the disk is full but I can't find the files" |
| Linux ignores file extensions | You'll trust `.jpg` from an upload endpoint and serve a PHP payload |

---

## The Physical Reality

Here is `/srv/app` on disk. **Everything** in this topic is a mutation of this picture.

```
            DIRECTORY /srv/app                      INODE TABLE                DATA BLOCKS
            (itself just a file whose               (fixed-size records,       (4 KB each)
             contents are name→inode pairs)          allocated at mkfs time)
   ┌────────────────────────────────┐        ┌──────────────────────────┐   ┌──────────┐
   │ name          │ inode          │        │ inode 1310725            │   │ block    │
   ├───────────────┼────────────────┤        │  type: regular file      │   │ 82,001   │
   │ "."           │ 1310720        │        │  mode: 0644              │──▶│ {"level" │
   │ ".."          │ 1179649        │        │  uid: 1001 (deploy)      │   │ :"info"..│
   │ "server.js"   │ 1310724  ──────┼───┐    │  nlink: 1  ◀── LINK COUNT│   └──────────┘
   │ "api.log"     │ 1310725  ──────┼─┐ │    │  size: 4096              │   ┌──────────┐
   │ "api.log.1"   │ 1310725  ──────┼─┤ │    │  atime/mtime/ctime       │──▶│ block    │
   │                ▲               │ │ │    │  blocks: [82001, 82002]  │   │ 82,002   │
   └────────────────┼───────────────┘ │ │    └──────────────────────────┘   └──────────┘
                    │                 │ │
        TWO NAMES, ONE INODE ─────────┘ │    ┌──────────────────────────┐
        (a hard link — nlink would be 2)│    │ inode 1310724            │
                                        │    │  ... server.js metadata  │
                                        └───▶└──────────────────────────┘
```

**Remember from 04 — Inodes in Depth:** the filename is *not* in the inode. The inode does not know what it is called. It only knows how many names point at it (`nlink`).

Now watch what each command touches:

```
touch newfile   →  allocate an inode, add ONE directory entry, zero data blocks
touch oldfile   →  no new inode, no new entry — just rewrite atime/mtime IN the inode

mkdir d         →  allocate an inode (type=directory), add entry in parent,
                   write "." and ".." entries INSIDE it, parent's nlink++ (because
                   d/.. now points back at the parent — this is why an empty
                   directory has nlink=2, not 1)

cp a b          →  allocate a NEW inode, allocate NEW data blocks, copy every byte,
                   add a new directory entry.   Costs: time + disk space.

mv a b          →  rewrite ONE directory entry. Same inode. Same blocks. Same data.
   (same fs)       Costs: microseconds + zero disk space.

rm a            →  remove the directory entry, inode.nlink--.
                   Blocks are freed ONLY when nlink hits 0 AND no process holds
                   an open file descriptor for that inode.  ◀◀◀ THE BIG ONE
```

---

## How It Works — Step by Step

### `touch app.log` (file does not exist)

```
COMMAND:     touch app.log
SHELL DOES:  no globbing to do; fork(), exec /usr/bin/touch
KERNEL DOES: walks the path, checks WRITE+EXECUTE on the *directory* (topic 05),
             allocates a free inode from the inode bitmap, writes a directory
             entry "app.log" → inode, sets atime=mtime=ctime=now, allocates
             ZERO data blocks (size 0)
SYSCALLS:    openat(AT_FDCWD, "app.log", O_WRONLY|O_CREAT|O_NONBLOCK|O_NOCTTY, 0666) = 3
             utimensat(3, NULL, NULL, 0)                                             = 0
             close(3)
OUTPUT:      nothing. Silence is success.
```

Note the mode: `0666`, not `0644`. The **umask** (topic 05) subtracts `0022` in the kernel, and you land on `0644`. `touch` never asks for `0644`.

### `touch app.log` (file already exists)

Identical syscalls. `openat` finds the existing inode instead of creating one; `utimensat(fd, NULL, NULL, 0)` means "set both timestamps to *now*". **No data is written. The file's contents are untouched.** `touch` on a 27 GB log costs nothing.

### `cp big.log backup.log`

```
COMMAND:     cp big.log backup.log
SYSCALLS:    openat(AT_FDCWD, "big.log", O_RDONLY)                          = 3
             newfstatat(3, "", {st_size=1073741824, ...}, AT_EMPTY_PATH)    = 0
             openat(AT_FDCWD, "backup.log", O_WRONLY|O_CREAT|O_EXCL, 0600)  = 4
             copy_file_range(3, NULL, 4, NULL, 1073741824, 0)               = 1073741824
             fchmod(4, 0644)                    ← mode copied from source
             close(4); close(3)
KERNEL DOES: allocates a new inode + new blocks; copy_file_range() does the copy
             INSIDE the kernel (no bounce through userspace buffers), and on a
             CoW filesystem (btrfs/XFS) may not copy the bytes at all.
NOT DONE:    fchown() and futimens() are ABSENT. The new file is owned by YOU
             and its mtime is NOW.  ◀◀◀ this is the -a vs -r bug, in one line.
```

### `mv old.log new.log` (same filesystem)

```
SYSCALLS:    renameat2(AT_FDCWD, "old.log", AT_FDCWD, "new.log", 0) = 0
KERNEL DOES: rewrites ONE directory entry, under a lock, in one atomic step.
             Zero bytes of file data are read or written. Inode unchanged.
COST:        O(1). A 100 GB file moves as fast as an empty one.
```

### `mv /tmp/x.log /var/log/x.log` (**different** filesystem)

```
SYSCALLS:    renameat2(...) = -1 EXDEV (Invalid cross-device link)   ◀◀◀ kernel refuses
             → mv now does it the hard way, in userspace:
             openat("/tmp/x.log", O_RDONLY) ... read()/write() loop ...
             fchown(); fchmod(); futimens()      ← mv DOES try to preserve metadata
             unlink("/tmp/x.log")
COST:        O(size). NOT ATOMIC. Ctrl-C halfway = a truncated file at the destination.
             New inode. New device. `ls -i` proves it.
```

A directory entry can only ever point to an inode **on its own filesystem** — inode numbers are only unique per-filesystem. That is the entire reason `EXDEV` exists.

### `rm api.log`

```
SYSCALLS:    newfstatat(AT_FDCWD, "api.log", ...) = 0
             unlinkat(AT_FDCWD, "api.log", 0)     = 0
KERNEL DOES: 1. removes the directory entry
             2. inode.nlink--
             3. IF nlink == 0 AND the kernel's in-memory open-count for that
                inode is also 0:
                     → mark inode free, mark its blocks free in the block bitmap
             4. IF nlink == 0 BUT some process still has it open:
                     → the inode LIVES ON with no name. Blocks stay allocated.
                       Space is NOT returned. df will not move.  ◀◀◀ 2am, this one
NOT DONE:    the data blocks are NEVER overwritten. `rm` writes no zeroes.
```

---

## `touch` — One Command, Two Completely Different Jobs

**Job 1: create an empty file.** `openat(O_CREAT)`.
**Job 2: update timestamps on an existing file.** `utimensat()`.

The same command, and it picks based on whether the path exists.

```bash
touch newfile.txt              # Job 1 — creates it, size 0
touch newfile.txt              # Job 2 — same file, mtime is now "now"

touch -a f    # only atime
touch -m f    # only mtime
touch -c f    # --no-create: if f doesn't exist, do NOTHING, exit 0 silently
touch -r ref f  # copy ref's timestamps onto f
touch -d "2 hours ago" f        # free-form date string (GNU only)
touch -d "2026-07-12T02:14:00" f
touch -t 202607120214.00 f      # [[CC]YY]MMDDhhmm[.ss]
touch -h symlink                # touch the SYMLINK itself, not its target
```

**The thing nobody tells you: you cannot set `ctime`.** There is no flag. `ctime` is the *inode change time*, and the kernel bumps it whenever the inode is modified — including when you use `touch` to fake the mtime. Backdating a file always leaves a fresh `ctime` behind. That's a forensics fact, not a bug.

```bash
# PROVE IT: you can fake mtime but never ctime
touch -d "2020-01-01" evidence.txt
stat evidence.txt
#   Modify: 2020-01-01 00:00:00.000000000 +0000   ← you faked this
#   Change: 2026-07-12 02:14:59.882914003 +0000   ← the kernel told the truth
```

### The `touch` trick: testing write permission for real

Reading `ls -ld` permission bits tells you what the bits *say*. It does not tell you whether you can actually write, because a **read-only mount**, a **full disk**, a **`chattr +i` immutable flag**, or an **ACL** will all stop you while the bits look perfect.

```bash
# The only honest test — try it and clean up:
touch /var/log/app/.writetest && rm /var/log/app/.writetest && echo WRITABLE

# Real failures this catches that `ls -ld` does not:
# touch: cannot touch '/var/log/app/.writetest': Read-only filesystem
# touch: cannot touch '/var/log/app/.writetest': No space left on device
# touch: cannot touch '/var/log/app/.writetest': Operation not permitted   ← chattr +i
```

This is the standard first move in a container/deploy debug: *before* blaming Node's `EACCES`, prove whether the directory is writable at all, as the same user (`sudo -u deploy touch ...`).

---

## `mkdir` and `rmdir` — Directories Are Just Files

```bash
mkdir logs                     # mkdir("logs", 0777) & ~umask → 0755
mkdir -p /srv/app/logs/2026    # create EVERY missing parent; no error if it exists
mkdir -m 700 secrets           # set the mode at creation (dodges the umask)
mkdir -pv a/b/c                # print each directory as it's created
```

**`-p` is the single most important flag in shell scripting**, for one reason that has nothing to do with parents:

```bash
# WRONG — a rerun of your deploy script dies here:
mkdir /srv/app/releases
# mkdir: cannot create directory '/srv/app/releases': File exists
# exit code 1 → with `set -e`, your whole deploy aborts

# RIGHT — idempotent. Exit code 0 whether or not it already existed.
mkdir -p /srv/app/releases
```

Idempotence is why every Dockerfile and every deploy script on earth uses `mkdir -p`.

**The `-p` + `-m` trap:** `-m` applies **only to the final directory**. Intermediate parents are created with the default `0777 & ~umask`.

```bash
mkdir -p -m 700 /srv/secrets/keys
ls -ld /srv/secrets /srv/secrets/keys
# drwxr-xr-x  ... /srv/secrets        ← 755. NOT 700. Not what you meant.
# drwx------  ... /srv/secrets/keys   ← 700
```

**`rmdir` only removes *empty* directories** (`rmdir()` returns `ENOTEMPTY` otherwise). That is a feature: it's the safe, non-recursive delete. `rmdir -p a/b/c` removes `c`, then `b`, then `a`, stopping as soon as one is non-empty.

| | `rmdir dir` | `rm -r dir` |
|---|---|---|
| Syscall | `rmdir()` | `getdents64()` + `unlinkat()` per entry, then `unlinkat(AT_REMOVEDIR)` |
| Non-empty dir | **refuses** — `Directory not empty` | deletes everything inside, recursively |
| Risk | essentially zero | unbounded |
| Use it when | you want a guard rail | you actually mean it |

An empty directory has `nlink = 2` (its own `.` plus the parent's entry). Every subdirectory you add bumps it, because each child's `..` points back. So: **a directory's link count = 2 + number of subdirectories.** `ls -ld` gives you the subdirectory count for free.

---

## `cp` — Why `-r` Is Not Enough and `-a` Is

`cp` creates a **new inode**. That means every piece of metadata on the new inode has to be *deliberately re-applied* — and by default, most of it isn't.

| | plain `cp -r` | `cp -a` (archive) |
|---|---|---|
| File contents | copied | copied |
| Permission bits | copied | copied |
| **Owner / group** | **becomes YOU** | preserved (needs root) |
| **mtime / atime** | **becomes NOW** | preserved |
| **Symlinks** | preserved as symlinks (GNU) | preserved as symlinks |
| **Hard links** | **broken — 2 names become 2 separate inodes, 2× the disk** | preserved as links |
| **Extended attrs / ACLs / SELinux** | **dropped** | preserved |
| Sparse-ness | mostly preserved (`--sparse=auto`) | preserved |

`-a` is literally defined as `-dR --preserve=all`. **`cp -a` is the correct flag for a backup or a release copy. `cp -r` is for casual copies you don't care about.**

Why it bites, concretely:

```bash
# You "back up" the app before a risky deploy:
cp -r /srv/app /srv/app.bak

# Restore goes fine... and then:
#  • every file in /srv/app is now owned by root (you ran it with sudo)
#    → your Node process runs as `deploy` and gets EACCES on its own files
#  • every mtime is "now"
#    → logrotate thinks nothing is old, `make` rebuilds everything,
#      and your "which file changed?" forensics are destroyed forever
```

Other flags that earn their keep:

```bash
cp -p src dst      # preserve mode,ownership,timestamps (a subset of -a)
cp -i src dst      # interactive: prompt before overwriting
cp -n src dst      # never overwrite (no prompt, just skip)
cp -u src dst      # only copy if source is NEWER than dest (poor man's rsync)
cp -v src dst      # verbose: print 'src' -> 'dst'
cp --reflink=auto src dst   # CoW copy on btrfs/XFS: instant, zero extra space
```

### `-L` vs `-P` — dereference the symlink, or copy the symlink?

```bash
ln -s /etc/nginx/nginx.conf link.conf

cp -P link.conf out/     # copy the LINK: out/link.conf is a symlink → /etc/nginx/nginx.conf
cp -L link.conf out/     # DEREFERENCE: out/link.conf is a real file with nginx's contents
```

GNU `cp -r` defaults to `-P` (do not follow links inside the tree). `-a` implies `-P`. If you want a self-contained tarball-like copy with no dangling links, you want `cp -rL` — but beware: **`-L` on a tree with a symlink loop will run forever, and a symlink pointing at `/` will try to copy the whole disk.**

### The trailing-slash gotcha (it is *not* what you think)

Everyone imports this expectation from `rsync`. **`cp` does not work like `rsync`.**

```bash
# rsync: the trailing slash on the SOURCE is meaningful
rsync -a src/ dest/     # copies the CONTENTS of src into dest
rsync -a src  dest/     # creates dest/src/

# cp: the trailing slash on the source changes NOTHING
cp -a src  dest/        # if dest exists → creates dest/src/   |  if not → dest IS the copy
cp -a src/ dest/        # ...identical. Same result. The slash is decoration.
```

The thing that actually controls it in `cp` is **whether the destination already exists**, which makes `cp -a src dest` *non-idempotent*: run it once and you get `dest`; run it twice and you get `dest/src`. That's a classic deploy-script bug.

```bash
# To copy the CONTENTS of src (including dotfiles) into an existing dest:
cp -a src/. dest/       #        ^^ the "/." is the idiom. Learn it.

# To force dest to BE the copy and never be treated as a container:
cp -aT src dest         # -T = --no-target-directory. Idempotent. Use this in scripts.
```

---

## `mv` — One Syscall, or a Whole Copy

### Within one filesystem: `rename()`. Instant. Same inode.

```bash
# PROVE IT: mv does not move data — it moves a NAME
cd /tmp
dd if=/dev/zero of=big.bin bs=1M count=500 status=none   # a 500 MB file
ls -i big.bin
# 1310726 big.bin          ← remember this inode number

time mv big.bin renamed.bin
# real  0m0.001s           ← 500 MB "moved" in one millisecond. No data was copied.

ls -i renamed.bin
# 1310726 renamed.bin      ← SAME INODE. It is literally the same file.
#                             Only the label on the shelf changed.
```

**Remember from 04 — Inodes in Depth:** the name lives in the directory, the data lives under the inode. `mv` only ever edits the directory. That is the whole trick.

### Across filesystems: copy + unlink. Slow. New inode. Not atomic.

```bash
# /tmp is often tmpfs (RAM); / is your disk. Different filesystems.
df --output=source /tmp / | tail -n +2
# tmpfs
# /dev/nvme0n1p1        ← two different devices

strace -e trace=renameat2,rename mv /tmp/big.bin /srv/big.bin
# renameat2(AT_FDCWD, "/tmp/big.bin", AT_FDCWD, "/srv/big.bin", 0)
#     = -1 EXDEV (Invalid cross-device link)     ◀◀◀ the kernel says NO
# → mv silently falls back to read()/write()/unlink(). 500 MB actually copied.
```

Consequences you must feel in your bones:

- **Not atomic.** A reader can see a half-written destination. A `kill -9` leaves a truncated file behind at the target *and* the original still present at the source.
- **New inode** → any hard links to the original are now pointing at a different file than the moved one. Any process holding the file open keeps holding the *old* inode.
- **Slow, and it needs free space for both copies at once.**
- This is why `mv` out of a Docker container's overlay layer, or between `/tmp` (tmpfs) and `/var`, is not free — even though `mv` "feels" free.

### THE deployment pattern: atomic rename

`rename()` is atomic **by kernel guarantee**: any process opening the target path sees either the complete old file or the complete new file. **Never a partial one. There is no window.**

This one property is the foundation of essentially every safe-write pattern in production.

```bash
# WRONG — there is a window of milliseconds where config.json is empty/partial.
# If nginx/your app reloads in that window, it reads garbage and dies.
generate-config > /etc/app/config.json

# RIGHT — write elsewhere, then swap in one atomic step.
generate-config > /etc/app/config.json.tmp   # readers still see the OLD file
mv /etc/app/config.json.tmp /etc/app/config.json   # rename() — instant swap
#  ^ the temp file MUST be on the same filesystem, or you lose atomicity to EXDEV.
#    Same directory is the safe habit.
```

Node does exactly this internally in every "write-file-atomic" package, and it is why `fs.rename()` is the last line of every safe config writer:

```js
// The Node version of the same idea
fs.writeFileSync('/etc/app/config.json.tmp', json);
fs.renameSync('/etc/app/config.json.tmp', '/etc/app/config.json'); // atomic
```

**Durability footnote:** atomic ≠ durable. `rename()` guarantees *other processes* never see a half file. It does **not** guarantee the bytes survive a power cut. For that you need `fsync(fd)` on the temp file *before* the rename, and `fsync()` on the *directory* after.

### The zero-downtime symlink deploy — and why `ln -sfn` is a lie

The classic release layout:

```
/srv/releases/2026-07-12-a1b2c3/
/srv/releases/2026-07-11-9f8e7d/
/srv/current  →  symlink to one of them   (this is what systemd/pm2 points at)
```

```bash
# WRONG — ln -sfn does unlink("current") THEN symlink(). Between those two
#         syscalls, /srv/current DOES NOT EXIST. A request landing there 500s.
ln -sfn /srv/releases/2026-07-12-a1b2c3 /srv/current

# RIGHT — build the new link under a temp name, then rename() OVER the old one.
ln -s /srv/releases/2026-07-12-a1b2c3 /srv/current.tmp
mv -T /srv/current.tmp /srv/current
#  ^^ -T (--no-target-directory) is MANDATORY. Without it, mv sees that
#     /srv/current is a symlink-to-a-directory and moves your temp link
#     INSIDE the old release directory. Silent, baffling, very common.
```

`mv -T` on two symlinks is a straight `rename()` — atomic. The old release directory is untouched, so rollback is another one-line atomic swap.

---

## `rm` — What Deleting Actually Deletes

`rm` calls **`unlink()`**. Read the name of that syscall again. It does not "delete", it does not "erase", it does not "wipe". It **removes a link**.

```
                              rm api.log
                                  │
                                  ▼
          ┌──────────────────────────────────────────────┐
          │ 1. Remove "api.log" from the directory       │
          │ 2. inode.nlink--                             │
          └───────────────────┬──────────────────────────┘
                              ▼
                    ┌─────────────────┐
                    │ nlink == 0 now? │
                    └────┬───────┬────┘
                     NO  │       │  YES
          ┌──────────────┘       └──────────────┐
          ▼                                     ▼
 Another name still                  ┌──────────────────────────┐
 points here (hard link).            │ Any process still holding │
 NOTHING is freed.                   │ an open fd on this inode? │
 The file is fully intact            └────┬────────────┬────────┘
 under its other name.                YES │            │ NO
                              ┌───────────┘            └───────────┐
                              ▼                                    ▼
                  ┌───────────────────────────┐     ┌──────────────────────────┐
                  │ ORPHANED INODE:           │     │ Inode marked free.       │
                  │  • no name anywhere       │     │ Blocks marked free in    │
                  │  • invisible to ls/du/find│     │ the block bitmap.        │
                  │  • blocks STILL ALLOCATED │     │ df finally moves.        │
                  │  • df does NOT move       │     │                          │
                  │  • freed only when the    │     │ (The bytes are still     │
                  │    LAST fd is closed —    │     │  physically on disk. rm  │
                  │    i.e. the process exits │     │  writes no zeroes ever.) │
                  └───────────────────────────┘     └──────────────────────────┘
```

Two enormous consequences:

**1. `rm` on an open log file does NOT free disk space.** Your Node process has fd 9 open on `/var/log/app/api-error.log`. You `rm` it. `ls` shows it gone. `du` no longer counts it. **`df` does not move one byte.** The inode is orphaned and the blocks stay pinned until the process closes the fd — i.e. until you restart your app. This is the #1 cause of "I deleted 27 GB and the disk is still full."

**2. That same fact is a recovery superpower.** As long as the process is still alive, the data is still reachable through `/proc/PID/fd/`:

```bash
# Someone rm'd the live log. Don't panic — the fd is still open.
lsof -p 8234 | grep deleted
# node  8234 deploy  9w  REG  259,1  28991029248  0  1310725 /var/log/app/api.log (deleted)
#                     ^                                    ^ nlink = 0

cp /proc/8234/fd/9 /var/log/app/api.log.recovered    # FULL RECOVERY. It's all there.
```

Once the last fd closes, that's it. `rm` does not overwrite the blocks, so the bytes are technically still on the platter — but on a live, journaling, extent-based filesystem with a busy allocator, recovery from userspace is a research project, not a runbook step. **Treat `rm` as final.**

**Remember from 05 — File Permissions:** to `rm` a file you need **write + execute on the containing directory**, *not* on the file. You can delete a file you cannot read. That's why the sticky bit exists on `/tmp`.

```bash
rm -r dir      # recursive: getdents64() the dir, unlinkat() each child, repeat
rm -f x        # force: no prompt, no error if missing, exit 0 (used in Makefiles)
rm -i x        # prompt for EVERY file
rm -I x*       # prompt ONCE if >3 files or if recursive — the sane middle ground
rm -v x        # print each removal
rm -d emptydir # rmdir, basically
rm --one-file-system -rf /mnt/x   # NEVER cross into another mount. Use this on servers.
```

---

## `rm -rf` Disasters — and the Five Guards

Every one of these is a real outage that happened to a real company. **In every single case, the bug is in the shell, not in `rm`.** The shell had already expanded the arguments before `rm` was even `exec`'d (topic 07).

### Disaster 1 — the unset variable

```bash
#!/bin/bash
# DIR was supposed to be set by CI. It wasn't. Typo in the var name.
rm -rf $DIR/*
#      ^^^^^^ the shell expands this to:   rm -rf /*
#             which is EVERY top-level directory, one by one.
```

`rm` ships with `--preserve-root` **on by default**, which refuses an argument that resolves to `/`. But `/*` is not `/` — the shell hands `rm` a list of `/bin /boot /etc /home /usr /var ...`, and **none of those is `/`**. The guard does not fire. The machine dies. *(This is Steam's famous 2015 bug, verbatim.)*

### Disaster 2 — the stray space

```bash
rm -rf /usr /lib/nvidia-current/xorg/xorg     # ← a space that should not be there
#      ^^^^ deletes /usr. The rest of the line was supposed to be one path.
```
*(This is the Bumblebee bug. It shipped.)* The same class: `rm -rf ~/tmp` vs `rm -rf ~ /tmp`.

### Disaster 3 — the filename that starts with a dash

```bash
rm -rf -recovered-data/     # rm parses "-recovered-data/" as OPTIONS, not a path
# rm: invalid option -- 'e'
```
Harmless here — but the inverse (`rm *` in a directory containing a file called `-rf`) is not.

### The Five Guards — put these in every script you write

```bash
# GUARD 1 — set -u: abort on any unset variable. Costs nothing. Add it today.
set -euo pipefail

# GUARD 2 — ${VAR:?} : refuse to expand, with a message, if unset OR empty.
#           THIS IS THE BEST ONE. It turns a catastrophe into an error message.
rm -rf "${DEPLOY_DIR:?DEPLOY_DIR is not set — refusing to run}"/*
# bash: DEPLOY_DIR: DEPLOY_DIR is not set — refusing to run     ← exits 1. Nothing deleted.

# GUARD 3 — always quote, and always use `--` to end option parsing.
rm -rf -- "$dir"
#       ^^ everything after this is a PATH, never an option. Kills Disaster 3.

# GUARD 4 — an explicit sanity check before anything destructive.
[[ -n "$dir" && "$dir" != "/" && -d "$dir" ]] || { echo "bad dir: '$dir'" >&2; exit 1; }

# GUARD 5 — don't rm at all. Move it aside; delete it tomorrow.
mv -- "$dir" "/var/tmp/trash/$(basename -- "$dir").$(date +%s)"
#  ^ instant (rename(), same fs), fully reversible, and you sleep at night.
```

Plus, interactively: **`rm -I`** (prompt once for bulk/recursive deletes) is a good default. **Do not `alias rm='rm -i'`** — it teaches your fingers that `rm` asks first, and then one day you're in a script or on a server without your dotfiles and it doesn't.

Belt and braces for a file that must never die: `sudo chattr +i /etc/app/config.json` makes the inode immutable — **even root cannot `rm` it** until `chattr -i`.

---

## Filenames That Fight Back

A Linux filename may contain **any byte except `/` and NUL**. That includes spaces, newlines, escape codes, and leading dashes. Uploads and untrusted input will find these.

```bash
# Leading dash — every tool reads it as a flag
rm -- -weird-file           # -- ends option parsing        ← the idiomatic fix
rm ./-weird-file            # or make it a path, not a word ← also works everywhere

# Spaces — the shell word-splits on them, so ALWAYS quote
rm "my report.pdf"          # RIGHT
rm my report.pdf            # WRONG: tries to delete "my", "report.pdf"

# Newlines / anything at all — never parse `ls` or `find` output with a pipe.
# Use NUL as the separator; it's the ONE byte a filename cannot contain.
find /uploads -name '*.tmp' -print0 | xargs -0 rm -v --
#                           ^^^^^^^          ^^

# Or skip the pipe entirely — find does its own deleting:
find /uploads -name '*.tmp' -delete
```

**Remember from 07 — How the Shell Works:** unquoted `$var` is word-split and glob-expanded *before the command runs*. Every filename bug in this section is that one rule biting you.

---

## `file` and `stat` — Interrogating a File

### `file` reads MAGIC BYTES. Linux does not care about extensions.

Windows decides what a file *is* from its name. **Linux never does.** `file` opens the file, reads the first few bytes, and compares them against a compiled database of signatures (`/usr/share/misc/magic.mgc`, via libmagic).

```bash
# PROVE IT: rename a PNG to .txt and watch Linux not care
cp logo.png totally-a-text-file.txt
file totally-a-text-file.txt
# totally-a-text-file.txt: PNG image data, 512 x 512, 8-bit/color RGBA, non-interlaced
#                          ^^^ the extension is a LIE and file knows it

od -An -tx1 -N8 totally-a-text-file.txt
#  89 50 4e 47 0d 0a 1a 0a
#     ^^ ^^ ^^  = "P" "N" "G" in ASCII. THAT is what file actually read.
```

| Bytes | Type |
|---|---|
| `89 50 4E 47 0D 0A 1A 0A` | PNG |
| `7F 45 4C 46` (`\x7fELF`) | Linux executable / shared object |
| `1F 8B` | gzip |
| `50 4B 03 04` | zip / jar / docx |
| `23 21` (`#!`) | script — the kernel itself reads these two bytes in `execve()` |
| `FF D8 FF` | JPEG |

This is not trivia. **The kernel does the exact same thing.** When you `execve()` a file, the kernel looks at the first bytes: `\x7fELF` → load it as a binary; `#!` → read the interpreter path after it and run *that*. The `.sh` on the end of your script is decoration for humans.

**Security consequence:** if your Node upload endpoint trusts `req.file.originalname.endsWith('.png')`, you are trusting a string an attacker typed. Check the magic bytes.

```bash
file -b app.js                # brief: just the type, no filename
file -i upload.dat            # MIME type: e.g. "image/png; charset=binary"
file -L link                  # follow the symlink and report the TARGET's type
file -s /dev/nvme0n1p1        # read special files (filesystem type on a raw partition)
file -z archive.gz            # look INSIDE the compression
```
*(Docker note: `file` is **not** in `alpine`, `debian:slim`, or `node:*-slim` images. `apk add file` / `apt install file`.)*

### `stat` — the inode, printed

```bash
stat /var/log/app/api.log
```
```
  File: /var/log/app/api.log
  Size: 27395866624   Blocks: 53507552   IO Block: 4096   regular file
Device: 802h/2050d    Inode: 1310725     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1001/  deploy)   Gid: ( 1001/  deploy)
Access: 2026-07-12 02:14:03.118273911 +0000
Modify: 2026-07-12 02:14:59.882914003 +0000
Change: 2026-07-12 02:14:59.882914003 +0000
 Birth: 2026-06-30 09:12:44.000000000 +0000
```

Every field, tied to the inode (topic 04):

| Field | What it really is | Why you care |
|---|---|---|
| `Size` | `st_size` — **apparent** length in bytes | A sparse file lies here. Compare with `Blocks`. |
| `Blocks` | `st_blocks` — **512-byte** units actually allocated | `53507552 × 512 = 27.4 GB` really on disk. This is what `du` sums. |
| `IO Block` | preferred I/O chunk size (`st_blksize`) | Usually 4096. Not the allocation unit. |
| `Device` | major/minor of the filesystem | **If two files differ here, `mv` between them is `EXDEV`.** |
| `Inode` | the inode number | The file's real identity. Names are just labels. |
| `Links` | `nlink` — how many names point here | `0` in `lsof` = the deleted-but-open case. |
| `Access:(0644/...)` | mode bits, octal + symbolic | Topic 05. |
| `Uid/Gid` | numeric owner + group | Topic 06. The kernel only knows the numbers. |
| `Access` (atime) | last read | Often lazily updated (`relatime`). Don't build logic on it. |
| `Modify` (mtime) | last time the **contents** changed | What `make`, `rsync`, `logrotate`, and `find -mtime` all use. |
| `Change` (ctime) | last time the **inode** changed | Bumped by `chmod`, `chown`, `mv`, *and* by writes. **Cannot be faked.** |
| `Birth` | creation time (`crtime`) | Often `-`: not every fs/kernel exposes it. |

```bash
stat -c '%s %n' *.log       # scriptable: just size and name
stat -c '%U:%G %a %n' f     # owner:group, octal mode, name
stat -f /var                # --file-system: statfs() — this is df's data source
```
*(macOS/BSD `stat` is a **different program** with different fields and a `-f` format string that means something else entirely. `brew install coreutils` → `gstat`.)*

---

## `du` vs `df` — Two Different Questions

They are not two views of the same number. They ask **two different kernel subsystems two different questions.**

```
   du -sh /var                              df -h /var
        │                                        │
        │ walks the directory tree               │ one statfs() call on the mount point
        │ stat()s every file it can NAME         │
        │ sums st_blocks                         │ reads the filesystem SUPERBLOCK counters
        ▼                                        ▼
 "how much space do the files                "how much space does THE FILESYSTEM
  I can find labels for use?"                  say is allocated, in total?"
        │                                        │
        │ CANNOT see:                            │ COUNTS:
        │  • deleted-but-open files              │  • deleted-but-open files
        │  • files under a mount point           │  • filesystem metadata + journal
        │  • dirs it lacks permission to read    │  • the 5% root reserve
        └────────────────────────────────────────┴─── ...and that is the whole mystery.
```

```bash
du -sh /var/log            # -s summarize (one total), -h human-readable
du -h --max-depth=1 /var   # one level down — THE triage command. (BusyBox/macOS: -d 1)
du -sh /var/* | sort -rh | head   # biggest children first
du -x -sh /                # -x: stay on ONE filesystem. Do not wander into /proc,
                           #     /sys, a mounted NFS/EFS share, or a Docker volume.
du -sh --apparent-size f   # report st_size (the "logical" size) instead of blocks
du -ah /srv | sort -rh | head -20   # -a: include individual FILES, not just dirs
```

```bash
df -h                      # human-readable, per mounted filesystem
df -h /srv/app             # just the filesystem that path lives on
df -i                      # INODES, not bytes  ◀◀◀ the forgotten half of "disk full"
df -T                      # show the filesystem type (ext4/xfs/overlay/tmpfs)
df -h --output=source,size,used,avail,pcent,target
```

**`df -i` is not optional.** A filesystem has a **fixed number of inodes**, decided at `mkfs` time. Millions of tiny files (an npm cache, session files, an unrotated per-request log) can exhaust the inode table while gigabytes of *bytes* remain free. The error is identical:

```
ENOSPC: no space left on device
```
```
df -h  → /dev/nvme0n1p1  40G  9.1G  29G  24% /     ← "plenty of space!"
df -i  → /dev/nvme0n1p1  2.5M  2.5M   0  100% /     ← THERE it is. Zero inodes left.
```

### Sparse files — where `ls`, `du`, and `df` all disagree with each other

A file can have **holes**: ranges that were never written and have no blocks allocated. The filesystem records "bytes 0–1GB are zeroes" instead of storing a gigabyte of zeroes.

```bash
truncate -s 1G sparse.img       # a 1 GB file that occupies 0 bytes

ls -lh sparse.img
# -rw-r--r-- 1 deploy deploy 1.0G Jul 12 02:20 sparse.img    ← ls shows st_size

du -h sparse.img
# 0       sparse.img                                          ← du sums st_blocks: ZERO

du -h --apparent-size sparse.img
# 1.0G    sparse.img                                          ← now du shows st_size too

stat -c 'size=%s blocks=%b' sparse.img
# size=1073741824 blocks=0        ← the whole story in one line
```

Docker images, VM disks, and Postgres files are all commonly sparse. `cp` without `--sparse=always` and `tar` without `--sparse` will happily **inflate a 0-byte sparse file into a real 1 GB of blocks** on the destination. That is a genuine way to fill a disk during a backup.

---

## WHY DO `du` AND `df` DISAGREE?

The interview question. The 2am question. There are exactly five real answers.

### 1. Deleted-but-still-open files ← *this is the one, 80% of the time*

A process holds an fd on a file you `rm`'d. `nlink == 0`, so the file has **no name** — `du` walks names, so `du` cannot see it. But the blocks are still allocated, so the **superblock** still counts them, so `df` sees it. The gap is exactly the size of the orphaned inodes.

```bash
# Find them. +L1 = "list files whose link count is LESS THAN 1"
sudo lsof +L1
# COMMAND  PID   USER  FD  TYPE DEVICE   SIZE/OFF NLINK    NODE NAME
# node    8234 deploy   9w  REG  259,1 28991029248    0 1310725 /var/log/app/api-error.log (deleted)
#                                      ^^^^^^^^^^^     ^
#                                      27 GB pinned    nlink=0 — no name. Invisible to du.

# No lsof in the container? /proc tells you anyway:
ls -l /proc/*/fd 2>/dev/null | grep '(deleted)'
```

**Fix:** you must make the process close the fd.
- Best: `systemctl restart myapp` / `pm2 reload api` — space is returned the instant the fd closes.
- No restart allowed? Truncate through `/proc`: `sudo truncate -s 0 /proc/8234/fd/9` — this zeroes the *orphaned inode itself* through the process's own fd, returning the blocks with the process still running.

### 2. A mount point with files hidden underneath it

Someone wrote 20 GB to `/data` **before** the disk was mounted there. Then the disk got mounted on top. Those 20 GB are still on the **root** filesystem, sitting under the mount point, unreachable — `du /data` now walks the *mounted* disk and never sees them, but `df /` still counts them.

```bash
# Look UNDER the mount, with a bind mount:
sudo mkdir -p /mnt/underneath
sudo mount --bind / /mnt/underneath
du -sh /mnt/underneath/data      # ← the hidden files, revealed
sudo umount /mnt/underneath
```

### 3. `du` couldn't read some directories (permissions)

Run `du` as a normal user and it silently skips what it can't `getdents64()`. `du` **undercounts**. Always run disk triage with `sudo`, and don't ignore `du: cannot read directory ...: Permission denied` on stderr.

### 4. The 5% root reserve (`df` *looks* wrong, isn't)

ext4 reserves 5% of the blocks for root by default so a full disk doesn't lock root out. `df` counts those blocks as part of `Size` but **not** as part of `Avail`, so `Used + Avail < Size` and `Use%` can hit 100% while there are still gigabytes physically free. Your `deploy` user gets `ENOSPC`; `root` doesn't. On a 500 GB data volume that reserve is 25 GB of waste — `sudo tune2fs -m 1 /dev/nvme0n1p1` reclaims it.

### 5. Filesystem metadata and the journal

`df` counts the inode table, the journal, and the block-group bookkeeping. `du` counts only file contents. On a fresh 40 GB filesystem, `df` shows ~1.5 GB used with zero files on it. That's normal.

---

## Exact Syntax Breakdown

```
touch -c -m -d "2 hours ago" /var/log/app/api.log
│     │  │  │   │             │
│     │  │  │   │             └── the file (must already exist, thanks to -c)
│     │  │  │   └── free-form date string — GNU parses this. Not on BSD/macOS.
│     │  │  └── --date : set the timestamp to THIS instead of now
│     │  └── only the MODIFY time (leave atime alone)
│     └── --no-create : if it doesn't exist, do nothing. Exit 0 anyway.
└── touch — either openat(O_CREAT), or utimensat(). Never both.
```

```
cp -a --no-clobber -v /srv/current/. /srv/backup/
│  │  │             │  │              │
│  │  │             │  │              └── destination (existing directory)
│  │  │             │  └── the "/." idiom: copy the CONTENTS of current, dotfiles included
│  │  │             └── print 'src' -> 'dst' for each file
│  │  └── = -n : never overwrite an existing destination file, and don't prompt
│  └── --archive = -dR --preserve=all : recursive + keep owner, mode, timestamps,
│                   symlinks, hard links, xattrs.  THE flag for backups.
└── cp — new inode, new blocks, real byte copying (via copy_file_range()).
```

```
mv -T -v /srv/current.tmp /srv/current
│  │  │  │                 │
│  │  │  │                 └── target: an EXISTING symlink we want to replace
│  │  │  └── source: the new symlink we just built
│  │  └── verbose
│  └── --no-target-directory : treat the target as a plain name to overwrite.
│       WITHOUT THIS, mv would move the source INSIDE the directory the
│       target symlink points at. This flag is what makes the deploy atomic.
└── mv — one renameat2() syscall. Atomic. Same-filesystem only.
```

```
rm -rf --one-file-system -- "${RELEASE_DIR:?not set}"
│  ││  │                  │  │
│  ││  │                  │  └── quoted + :? guard — refuses to expand if unset OR empty
│  ││  │                  └── end of options: anything after is a PATH, even if it starts with -
│  ││  └── refuse to cross into another mounted filesystem (won't eat your NFS share)
│  │└── --force : no prompts, no error on missing files, exit 0
│  └── --recursive : getdents64() + unlinkat() the whole tree, depth-first
└── rm — unlink(). Removes a NAME. Frees blocks only if nlink hits 0 AND no fd is open.
```

```
du -x -h --max-depth=1 /var | sort -rh | head -10
│  │  │  │              │      │            │
│  │  │  │              │      │            └── top 10 only
│  │  │  │              │      └── -r reverse (biggest first), -h understands "4.8G" > "512M"
│  │  │  │              └── where to start walking
│  │  │  └── only descend ONE level — you want the guilty child, not 40,000 files
│  │  └── human-readable (4.8G, not 5033164)
│  └── stay on this filesystem — don't wander into /proc, /sys, or a mounted volume
└── du — walks names, sums st_blocks. Cannot see deleted-but-open files.
```

```
df -h -i --output=target,size,used,avail,pcent
│  │  │  │
│  │  │  └── pick exactly the columns you want (GNU only)
│  │  └── report INODES instead of bytes — the other way to run out of "space"
│  └── human-readable
└── df — one statfs() per mount. Reads the SUPERBLOCK. Sees everything, names or not.
```

---

## Example 1 — Basic

```bash
cd /tmp && rm -rf lab && mkdir -p lab/src && cd lab

# --- touch: create, then re-stamp ---
touch a.txt                     # creates it: openat(O_CREAT) → size 0
stat -c 'inode=%i size=%s mtime=%y' a.txt
# inode=1310730 size=0 mtime=2026-07-12 02:20:01.000000000 +0000

touch a.txt                     # SAME file — utimensat() only. No new inode.
stat -c 'inode=%i size=%s mtime=%y' a.txt
# inode=1310730 size=0 mtime=2026-07-12 02:20:44.000000000 +0000
#       ^^^^^^^ identical inode — touch did NOT recreate the file, only re-dated it

# --- mkdir -p is idempotent; plain mkdir is not ---
mkdir src                       # mkdir: cannot create directory 'src': File exists  → exit 1
mkdir -p src                    # silence. exit 0. Run it a hundred times.  ← use this in scripts

# --- cp makes a NEW inode. mv does not. ---
echo "v1" > src/config.json
ls -i src/config.json
# 1310731 src/config.json

cp src/config.json src/copy.json
ls -i src/copy.json
# 1310732 src/copy.json          ← DIFFERENT inode. Real bytes were copied.

mv src/copy.json src/renamed.json
ls -i src/renamed.json
# 1310732 src/renamed.json       ← SAME inode as copy.json was. Only the name moved.

# --- rm removes a NAME. Watch the link count. ---
ln src/config.json src/hardlink.json     # a second NAME for the same inode (topic 04)
stat -c 'links=%h' src/config.json
# links=2                                 ← one inode, two names

rm src/config.json                        # unlink() — removes ONE name
stat -c 'links=%h' src/hardlink.json
# links=1                                 ← the DATA is completely intact.
cat src/hardlink.json
# v1                                      ← nothing was deleted. Only a label was.

# --- file ignores extensions; it reads bytes ---
printf '\x89PNG\r\n\x1a\n' > definitely-not-an-image.txt
file definitely-not-an-image.txt
# definitely-not-an-image.txt: PNG image data     ← the .txt is a lie and file knows

# --- du vs df, sparse edition ---
truncate -s 100M sparse.img
ls -lh sparse.img | awk '{print $5}'      # 100M   ← what ls thinks (st_size)
du -h  sparse.img                         # 0      ← what the DISK thinks (st_blocks)
```

---

## Example 2 — Production Scenario

**02:11.** PagerDuty. Your Node API is returning 500s on every request. You SSH in.

Topic 08 taught you how to *find* the big files. This is what happens **after** you find them — and it is where most engineers lose an hour.

```bash
# ── STEP 1: bytes? ──────────────────────────────────────────────────────────
df -h /
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme0n1p1   40G   40G     0 100% /
#                                  ^^ ← zero bytes free. Node's write() is returning
#                                       ENOSPC. That is the 500.

# ── STEP 2: inodes? (NEVER skip this — same error, totally different fix) ───
df -i /
# Filesystem      Inodes  IUsed IFree IUse% Mounted on
# /dev/nvme0n1p1     2.5M   412K  2.1M   17% /
#                                        ^^^ ← inodes are fine. It really is bytes.

# ── STEP 3: where are the bytes? One level at a time. ──────────────────────
sudo du -x -h --max-depth=1 / 2>/dev/null | sort -rh | head -6
#  40G   /
#  31G   /var          ← follow the weight
# 4.8G   /opt
# 2.3G   /home
sudo du -x -h --max-depth=1 /var | sort -rh | head -4
#  31G   /var
#  30G   /var/log      ← logs, as always
sudo du -x -sh /var/log/* | sort -rh | head -3
# 2.1G  /var/log/syslog
# 900M  /var/log/nginx
# 1.2G  /var/log/journal
#
#  ...wait. 2.1 + 0.9 + 1.2 = 4.2 GB. But du said /var/log is 30 GB.
#  ◀◀◀ du DISAGREES WITH ITSELF. 26 GB is unaccounted for. Somebody already
#      "fixed" this before I got here.

# ── STEP 4: THE du/df GAP. Someone rm'd a live log. ────────────────────────
sudo lsof +L1
# COMMAND  PID   USER  FD  TYPE DEVICE    SIZE/OFF NLINK    NODE NAME
# node    8234 deploy   9w  REG  259,1 27395866624     0 1310725 /var/log/app/api-error.log (deleted)
#                                       ^^^^^^^^^^^     ^
#                                       25.5 GB         nlink=0 → NO NAME
#
# There it is. The on-call before me ran `rm /var/log/app/api-error.log` at 01:58,
# saw `ls` show it gone, and assumed the space came back. It did not.
# Node still holds fd 9. The inode is orphaned; its 25.5 GB of blocks are PINNED.
# `du` cannot see it (no name to walk). `df` counts it (superblock). Hence the gap.

# ── STEP 5: get the space back WITHOUT restarting Node ──────────────────────
sudo truncate -s 0 /proc/8234/fd/9      # zero the orphaned inode THROUGH the live fd
df -h /
# /dev/nvme0n1p1   40G   14G   26G  35% /
#                              ^^^ ← 26 GB back, instantly. Node never dropped a request.

# ── STEP 6: why was it 25 GB? The disk was the symptom. ────────────────────
sudo tail -2 /var/log/app/api-error.log.1
# {"level":"error","msg":"ECONNREFUSED 10.0.3.14:5432","ts":"2026-07-12T01:44:59Z"}
# {"level":"error","msg":"ECONNREFUSED 10.0.3.14:5432","ts":"2026-07-12T01:44:59Z"}
#   ← Postgres went away. The app retry-loops and logs every failure at ~40 MB/min.
#     THE DISK IS NOT THE BUG. The dead database is the bug.

ls /etc/logrotate.d/
# apt  dpkg  nginx  rsyslog
#   ← no entry for /var/log/app. Nothing has EVER rotated this file. Root cause #2.
```

**The two lessons, and they are the whole topic:**

1. **`rm` on an open log frees nothing.** Truncate instead. `: > file`, `truncate -s 0 file`, or — if it's already been `rm`'d — `truncate -s 0 /proc/PID/fd/N`.
2. The correct way to zero a live log, *before* anyone reaches for `rm`:

```bash
: > /var/log/app/api-error.log      # O_TRUNC. Same inode. Node's fd 9 stays valid.
truncate -s 0 /var/log/app/api-error.log   # identical effect, clearer intent
```

> ### The truncate trap you must know about
> Truncating is safe **only if the writer opened the file with `O_APPEND`** (Node: `fs.createWriteStream(path, {flags:'a'})`, which is what pm2 and every sane logger do). `O_APPEND` forces every `write()` to the current end of file, so after truncation the next write lands at offset 0.
>
> If the writer opened it **without** `O_APPEND`, it keeps its own file offset — say, 25.5 GB. Truncation doesn't reset that offset. The very next `write()` goes to byte 27,395,866,624, and the kernel creates a **sparse file**: `ls -lh` instantly shows 25G again, though `du` shows almost nothing. Confusing as hell at 3am. If you see that, restarting the process is the real fix.

---

## Common Mistakes

### Mistake 1 — `rm` the log to free space

**Wrong:** `rm /var/log/app/api-error.log` while Node has it open.
**What breaks, at the kernel level:** `unlink()` removes the *directory entry* and decrements `nlink` to 0. But the kernel keeps an in-memory reference count for every open file description. That count is 1 (Node's fd 9). Blocks are freed only when **both** counters reach zero. `df` does not move. You have now also destroyed your ability to `tail` the log while the incident is still happening.
**Diagnose:** `sudo lsof +L1` → `NLINK` column is `0`.
**Fix:** `sudo truncate -s 0 /proc/<PID>/fd/<N>`, or restart the process.
**Prevent:** never `rm` a live log. `truncate -s 0` (or `: >`) keeps the inode, keeps the fd valid, and returns the blocks immediately. Then fix `logrotate` so it never gets big again (topic 23).

### Mistake 2 — `cp -r` as your backup

**Wrong:** `sudo cp -r /srv/app /srv/app.bak`
**What breaks:** `cp` allocates a **new inode** per file. Owner defaults to the *calling* user (root, because you sudo'd), mtimes default to *now*, xattrs/ACLs are dropped, and hard links inside the tree become independent duplicate files (doubling their disk usage).
**The broken state:** you restore, Node starts as `deploy`, and immediately `EACCES` on files it "owns". Every mtime is identical so `find -mtime`, `rsync`, `make`, and your forensics are all worthless.
**Fix / Prevent:** **`cp -a`**. Always, for anything you might restore. `-a` = `-dR --preserve=all`.

### Mistake 3 — `mv` where you needed atomicity, or vice versa

**Wrong:** `generate-config > /etc/app/config.json` while the app watches that file.
**What breaks:** `>` opens with `O_TRUNC` and then writes in chunks. Between the truncate and the last write, the file is **partial**. Your file-watcher fires on the first write, reads `{"port": 30`, and `JSON.parse` throws. The app crash-loops.
**Fix:** write to `config.json.tmp` **in the same directory**, then `mv`. `rename()` is atomic — no reader can ever observe an intermediate state.
**Prevention gotcha:** if the temp file is in `/tmp` and the target is on `/`, the `rename()` fails with `EXDEV` and `mv` silently degrades to a **non-atomic** copy — you have reintroduced the exact bug you were fixing. **Same filesystem. Same directory. Every time.**

### Mistake 4 — `rm -rf $DIR/` with `DIR` unset

**Wrong:**
```bash
rm -rf $DIR/build          # DIR is empty → the shell hands rm the literal path "/build"
rm -rf $DIR/*              # DIR is empty → the shell hands rm  /bin /boot /etc /home ...
```
**Root cause:** this is not an `rm` bug. **The shell expanded the variable before `rm` was even executed** (topic 07). `rm` received a perfectly valid list of paths and did exactly what it was told. `--preserve-root` is on by default, but it only refuses an argument that *resolves to `/`* — and `/bin` is not `/`. The guard never fires.
**Fix / Prevent:** `set -euo pipefail` at the top of every script, and make the expansion itself refuse:
```bash
rm -rf -- "${DIR:?DIR is not set}"/build
```
This aborts with an error message and deletes nothing. It is one character (`?`). Use it.

### Mistake 5 — trusting the file extension

**Wrong:** your upload endpoint accepts `avatar.png` because it ends in `.png`.
**Root cause:** Linux has **no concept of a file extension.** The kernel dispatches on magic bytes (`\x7fELF`, `#!`), and so does every serious tool. The name is metadata for humans.
**Diagnose:** `file -b --mime-type /uploads/avatar.png` → `text/x-php`.
**Prevent:** validate content, not names. And never make an upload directory executable or servable.

### Mistake 6 — `du` says 8 GB, so the disk must be fine

**Wrong:** ignoring `df` because `du` looks reasonable.
**Root cause:** `du` walks **names**. Anything without a name (deleted-but-open) or hidden under a mount point is invisible to it. `df` reads the **superblock** and sees the truth.
**Rule:** **`df` is the authority on whether you are out of space. `du` is only a tool for finding out where the space went.** When they disagree, `df` is right and `du` is blind — go find what `du` cannot see (`lsof +L1`).

---

## Hands-On Proof

```bash
cd /tmp && rm -rf proof && mkdir proof && cd proof

# PROVE IT: touch's two personalities, in syscalls
strace -e trace=openat,utimensat touch new.txt 2>&1 | grep -E 'new.txt|utimensat'
# openat(AT_FDCWD, "new.txt", O_WRONLY|O_CREAT|O_NONBLOCK|O_NOCTTY, 0666) = 3
# utimensat(3, NULL, NULL, 0)                                            = 0
#   ← O_CREAT made the file; utimensat set the times. Two jobs, one command.

# PROVE IT: mv is rename() — the inode does not change
dd if=/dev/zero of=big.bin bs=1M count=300 status=none
ls -i big.bin                    # 1310801 big.bin
time mv big.bin moved.bin        # real 0m0.001s   ← 300 MB in 1 ms. No data moved.
ls -i moved.bin                  # 1310801 moved.bin   ← SAME INODE. Same file. New label.
strace -e trace=renameat2 mv moved.bin final.bin 2>&1 | head -1
# renameat2(AT_FDCWD, "moved.bin", AT_FDCWD, "final.bin", 0) = 0   ← ONE syscall.

# PROVE IT: cross-filesystem mv is NOT a rename — it's a copy + unlink
mkdir -p /dev/shm/x                        # /dev/shm is tmpfs — a DIFFERENT filesystem
strace -e trace=renameat2 mv final.bin /dev/shm/x/ 2>&1 | head -1
# renameat2(...) = -1 EXDEV (Invalid cross-device link)   ← the kernel REFUSES.
#   mv then falls back to read/write/unlink in userspace. New inode. Not atomic.
stat -c 'device=%d inode=%i' /dev/shm/x/final.bin      # different device AND inode
rm -rf /dev/shm/x

# PROVE IT: rm is unlink() — it removes a NAME, not data
echo "irreplaceable" > data.txt
ln data.txt backup.txt                     # second name, same inode
stat -c 'inode=%i links=%h' data.txt       # inode=1310805 links=2
rm data.txt                                # unlink() one name
cat backup.txt                             # irreplaceable   ← data is 100% intact
stat -c 'inode=%i links=%h' backup.txt     # inode=1310805 links=1  ← same inode!

# PROVE IT: **rm does not free space while a process holds the file open**
python3 -c "
import time
f = open('/tmp/proof/huge.log','w'); f.write('x'*200_000_000); f.flush()
print('holding fd open, PID', __import__('os').getpid()); time.sleep(120)
" &
sleep 2
df -h /tmp | tail -1                       # note the Used column
rm /tmp/proof/huge.log                     # "delete" 200 MB
ls /tmp/proof/huge.log                     # No such file or directory  ← the name is gone
du -sh /tmp/proof                          # tiny — du cannot see it (no name to walk)
df -h /tmp | tail -1                       # ◀◀◀ USED HAS NOT DROPPED. 200 MB still pinned.
sudo lsof +L1 | grep huge                  # NLINK column = 0. There is your ghost.
kill %1; sleep 1
df -h /tmp | tail -1                       # NOW the space is back. The fd closed.

# PROVE IT: cp -r destroys metadata; cp -a preserves it
touch -d "2020-01-01" old.conf
cp -r old.conf plain.conf   && stat -c '%y %n' plain.conf   # 2026-07-12 ...  ← mtime LOST
cp -a old.conf archive.conf && stat -c '%y %n' archive.conf # 2020-01-01 ...  ← preserved

# PROVE IT: file reads bytes, not names
printf '\x7fELF' > fake.txt && file fake.txt      # fake.txt: ELF ...  ← ignores ".txt"
od -An -tx1 -N4 fake.txt                          # 7f 45 4c 46

# PROVE IT: rename() is atomic — a reader NEVER sees a half file
( while :; do printf 'v%s\n' "$RANDOM" > t.tmp; mv t.tmp target.txt; done ) &
for i in $(seq 200); do wc -l < target.txt; done | sort -u
# 1        ← every single read saw exactly one complete line. Never 0. Never partial.
kill %1

# PROVE IT: a sparse file — ls, du and stat all tell different stories
truncate -s 500M hole.img
ls -lh hole.img | awk '{print "ls says:  " $5}'          # ls says:  500M
du -h  hole.img | awk '{print "du says:  " $1}'          # du says:  0
stat -c 'stat says: size=%s blocks=%b' hole.img          # size=524288000 blocks=0
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal:

1. `mkdir -p ~/lab/a/b/c` — then run it **again**. What is the exit code (`echo $?`) both times? Now try the same with plain `mkdir`. Explain the difference in one sentence.
2. Create `notes.txt`, note its inode with `ls -i`. `cp` it to `copy.txt` and `mv` it to `moved.txt`. Show all three inode numbers and explain which operation created a new one and why.
3. Use `touch -d` to set a file's mtime to `2015-06-01`. Then `stat` it. **Why is `Change:` still today?**

### Exercise 2 — Medium

Prove the `cp -r` metadata bug to yourself, end to end:

1. As root: `sudo mkdir -p /srv/x && sudo touch -d "2021-03-01" /srv/x/app.js && sudo chown 1001:1001 /srv/x/app.js`
2. Copy it two ways: `sudo cp -r /srv/x /srv/x-r` and `sudo cp -a /srv/x /srv/x-a`
3. Run `stat -c '%U %G %y %n' /srv/x/app.js /srv/x-r/app.js /srv/x-a/app.js`
4. Report exactly which fields differ, and explain **at the inode level** why `cp` had to re-apply them at all.
5. Now do the same with a hard-linked pair (`ln`), and use `du -sh` to show that `cp -r` **doubled the disk usage** while `cp -a` did not.

### Exercise 3 — Hard (Production Simulation)

Recreate the 2am incident and fix it without restarting the "app":

1. Start a fake app that holds a big log open and keeps writing to it (any language; make sure it opens with append mode).
2. Grow the log to at least 500 MB. Record `df -h` and `du -sh` for that filesystem.
3. **`rm` the log file** (yes, do the wrong thing on purpose).
4. Show that: `ls` says it's gone, `du` no longer counts it, but **`df` has not moved**. Explain the discrepancy in terms of `nlink` and open file descriptors.
5. Find the ghost with `lsof +L1`. Identify the PID and the fd number.
6. **Reclaim the space without killing the process.** (Hint: `/proc/PID/fd/N` is a real path to the orphaned inode.)
7. Verify with `df -h`.
8. Finally, write the *correct* 3-line runbook you should have followed instead — using `truncate` — and add the `logrotate` config that would have prevented all of it.

---

## Mental Model Checkpoint

1. `touch` on an existing 27 GB file takes ~0 seconds. **Why?** What syscall does it make, and what does it *not* touch?
2. You `mv` a 4 GB file within `/srv` and it's instant. You `mv` it from `/tmp` to `/srv` and it takes 20 seconds. **Explain both, naming the syscall and the errno.**
3. What exactly makes writing a temp file and then `mv`-ing it over the target **atomic**? What single condition must hold for that guarantee to survive?
4. You `rm` a 27 GB log. `ls` shows it gone. `df` still says 100% full. **What is the state of that inode, and what are the two counters the kernel is waiting on?**
5. Your teammate ran `sudo cp -r /srv/app /backup`. Name **three** things that are now wrong with `/backup` that `cp -a` would have preserved.
6. `du -sh /var` says 8 GB. `df -h` says `/var` is 100% full on a 40 GB disk. Name **three** possible explanations and the command that distinguishes them.
7. Why does `rm -rf $DIR/*` destroy a server when `--preserve-root` is enabled by default? Which layer is actually at fault, and what is the one-character fix?

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `touch` | Create empty file **or** update timestamps | `-c` no-create · `-a`/`-m` atime/mtime only · `-d`/`-t` set time · `-r` copy from ref · `-h` don't follow symlink |
| `mkdir` | Create directory (`mkdir()`) | **`-p`** parents + idempotent · `-m` mode · `-v` verbose |
| `rmdir` | Remove **empty** directory only | `-p` remove parents too |
| `cp` | Copy — **new inode, real bytes** | **`-a`** archive (backups!) · `-r` recursive · `-p` preserve · `-i`/`-n` don't clobber · `-u` newer only · `-L`/`-P` follow/don't follow symlinks · `-T` no-target-dir · `--reflink=auto` |
| `mv` | Rename/move — `rename()`, **atomic on one fs** | `-T` no-target-dir (**the deploy flag**) · `-i`/`-n` · `-f` · `-v` · `-b` backup |
| `rm` | `unlink()` — removes a **name** | `-r` recursive · `-f` force · `-i` prompt each · **`-I`** prompt once · `--one-file-system` · `--` end options |
| `file` | Identify type from **magic bytes** | `-b` brief · `-i` MIME · `-L` follow symlink · `-s` special files · `-z` inside compression |
| `stat` | Dump the **inode** | `-c` format · `-f` filesystem (statfs) · `-L` follow symlink |
| `du` | Disk **used by named files** (sums `st_blocks`) | `-s` summary · `-h` human · `--max-depth=N` (BusyBox/mac: `-d N`) · **`-x`** one fs · `-a` files too · `--apparent-size` |
| `df` | Filesystem **free space** (superblock) | `-h` human · **`-i` inodes** · `-T` type · `--output=` |
| `truncate` | Set a file's size **without changing the inode** | `-s 0` empty it · `-s 1G` extend (sparse) · `-s -100M` shrink by |
| `shred` | Overwrite contents, then optionally remove | `-n N` passes · `-z` final zero pass · `-u` unlink after |
| `install` | Copy **+ set mode/owner/group** in one step | `-m` mode · `-o`/`-g` owner/group · `-d` create dir · `-D` create parents |
| `ln` | Make a link (topic 04) | *(none)* hard link · `-s` symlink · `-f` force · `-n` treat symlink-dir as a file · `-r` relative · `-T` |
| `rename` | Bulk-rename by pattern | **Two different programs — check `rename --version`.** Perl: `rename 's/\.txt$/.md/' *.txt`. util-linux: `rename .txt .md *.txt` |

### `shred` — read this before you trust it

`shred` overwrites the file's **current blocks**. That assumption is false on: **ext4 with journaling** (metadata copies survive), **btrfs/ZFS/overlayfs and every CoW filesystem** (a write goes to *new* blocks — the old ones are untouched), **any SSD** (wear-levelling means the controller writes elsewhere; the original NAND cells are still there), **any snapshotted or backed-up volume**, and **every Docker container** (overlay2 is CoW). On a modern cloud server, `shred` is close to security theatre. **Real answer: full-disk encryption, and destroy the key.**

### The portability traps that will actually bite you

| | GNU (Ubuntu/Debian) | BusyBox (Alpine / Docker) | macOS / BSD (your laptop) |
|---|---|---|---|
| `cp -a` | ✅ | ✅ (approximate) | ❌ **no `-a`** — use `cp -Rp`, `rsync -a`, or `ditto` |
| `du --max-depth=N` | ✅ | ❌ use `-d N` | ❌ use `-d N` |
| `du --apparent-size` | ✅ | ❌ | ❌ |
| `truncate` | ✅ | ✅ | ❌ **not installed** — use `: > file`, or `brew install coreutils` → `gtruncate` |
| `file` | ✅ | ❌ **not installed** — `apk add file` | ✅ |
| `stat -c '%s'` | ✅ | ✅ | ❌ different program: `stat -f '%z'` |
| `lsof` | ✅ | ❌ — use `ls -l /proc/*/fd \| grep deleted` | ✅ |
| `touch -d "2 hours ago"` | ✅ | ❌ | ❌ |
| `shred` | ✅ | ❌ | ❌ (`rm -P`, and APFS ignores it) |

**Write your deploy scripts against GNU. Test them in the container, not on your Mac.**

---

## When Would I Use This at Work?

### Scenario 1: Zero-downtime config reload
Your Node service watches `/etc/app/config.json` with `fs.watch` and hot-reloads on change. Your config-generator writes straight to that path — and once a week the app crash-loops on a `JSON.parse` error nobody can reproduce. You now know exactly why: `>` truncates, then writes in chunks, and the watcher fires on the *first* chunk. Fix: generate to `config.json.tmp` **in the same directory**, then `mv` it over. `rename()` is atomic; the reader gets the whole old file or the whole new one. The bug never comes back.

### Scenario 2: The disk fills at 2am and someone already "fixed" it
`df` says 100%. `du` accounts for only 4 GB of a 30 GB gap. Instead of guessing, you run `sudo lsof +L1`, see `NLINK 0` on a 26 GB log that the previous on-call already `rm`'d, and reclaim the space with `truncate -s 0 /proc/8234/fd/9` — **without restarting the API**. Total time: 90 seconds. Then you add the `logrotate` rule that should have existed and file a bug on the retry loop that wrote 26 GB in an hour.

### Scenario 3: The release that "went out" but didn't
Your deploy does `ln -sfn /srv/releases/$SHA /srv/current` and health checks flap for a second on every deploy. `ln -sfn` is `unlink()` + `symlink()` — **two syscalls, with a gap where `/srv/current` does not exist**. Any request landing in that gap 500s. You switch to `ln -s /srv/releases/$SHA /srv/current.tmp && mv -T /srv/current.tmp /srv/current`, which is a single atomic `rename()`. The flapping stops. Rollback becomes another one-line atomic swap.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 04 — Inodes in Depth | Every command here is a mutation of a directory entry or an inode. `mv` = edit the entry. `rm` = decrement `nlink`. `cp` = allocate a new inode. |
| **Builds on** | 05 — File Permissions | You need **w+x on the directory** to create or delete a file — *not* on the file itself. The sticky bit on `/tmp` exists because of exactly this. |
| **Builds on** | 07 — How the Shell Works | The shell expands `$DIR` and `*` **before** `rm` runs. Every `rm -rf` disaster is a shell-expansion bug wearing an `rm` costume. |
| **Builds on** | 08 — Navigating the Filesystem | `find` locates the big files; this topic explains why deleting them didn't help. |
| **Next** | 10 — Viewing and Reading Files | `tail -f` follows an **fd**; `tail -F` follows a **name** — which is the difference between surviving a `truncate` and surviving a `logrotate`. |
| **Used by** | 12 — Pipes and Redirection | `: > file` and `> file` are `O_TRUNC` — the safe way to empty a live log, and the *unsafe* way to write a config. |
| **Used by** | 13 — Shell Scripting | `set -euo pipefail`, `"${VAR:?}"`, quoting, and `--` are the guards that stand between your script and `rm -rf /`. |
| **Used by** | 19 — File Descriptors | `lsof +L1`, `/proc/PID/fd/N`, and *why* an unlinked inode with an open fd keeps its blocks. |
| **Used by** | 23 — Logs and Log Management | `logrotate`'s `copytruncate` is literally `cp` + `truncate -s 0` — and it has the exact race this topic describes. |
| **Used by** | 31 — Disk Management | `df -i`, the 5% root reserve, mount-point shadowing, and what to do when the volume is genuinely full. |
