# 25 — File Compression & Archiving

## ELI5 — The Simple Analogy

You're moving house.

- **Archiving** is putting all your loose stuff into **one big box** so you can carry it in a single trip instead of juggling 400 individual items. The box doesn't make anything *smaller* — a box of books weighs exactly as much as the books. It just makes them **one thing**. That's `tar`.
- **Compression** is **vacuum-sealing** a bag of clothes so it takes up a third of the space. It doesn't bundle multiple things — you seal *one* bag. That's `gzip` / `xz` / `zstd`.

These are **two completely separate jobs**, done by two completely separate tools. The reason a `.tar.gz` file exists is that you first put everything in a box (`tar`), and *then* you vacuum-sealed the whole box (`gzip`). Box first, then seal. That order — and the fact that they're separate — is the single idea that makes the entire `tar` command make sense.

And here's the twist Linux people care about: if you vacuum-seal the *whole box at once*, similar clothes squish against each other and you save more space than sealing each shirt in its own little bag. That's why Linux prefers `.tar.gz` (seal the whole box) over `.zip` (seal each item separately).

---

## Where This Lives in the Linux Stack

```
Hardware (disk blocks; the network wire for streamed archives)
  └── KERNEL
       │    • read()/write() move bytes; pipe() connects two processes
       │    • the page cache (compressing re-reads come from RAM, not disk)
       │
       └── System Calls
            │    openat(), read(), write(), close()   ← tar walking the tree
            │    lstat()  ← tar reading each file's metadata into the archive header
            │    pipe(), dup2()  ← how `tar | gzip | ssh` is wired (topic 12)
            │
            └── C Library (zlib for gzip; liblzma for xz; libzstd)
                 │
                 └── ARCHIVERS & COMPRESSORS  ◀◀◀ THIS TOPIC (userspace programs)
                      │    • tar   — bundles only (calls a compressor for you via -z/-j/-J)
                      │    • gzip / bzip2 / xz / zstd / lz4 — compress a single stream
                      │    • zip / unzip — do BOTH at once, per-file (for interop)
                      │    • zcat / zless / zgrep — read compressed WITHOUT decompressing to disk
                      │
                      └── Shell — pipes them together: pg_dump | gzip | ssh ...
```

**Key insight:** `tar` and the compressors are ordinary filter programs. Most of them read `stdin` and write `stdout`, which is *the whole reason* you can chain them into pipelines that never touch a temp file (topic 12). Hold onto that — the streaming section is the most valuable part of this doc.

---

## What Is This?

**Archiving** bundles many files (with their names, permissions, ownership, timestamps, symlinks) into **one stream/file** — without shrinking anything. `tar` does this.

**Compression** re-encodes bytes to take less space, by finding and eliminating redundancy. `gzip`, `bzip2`, `xz`, `zstd` do this — classically to a **single** stream.

`.tar.gz` (= `.tgz`) is the composition: a `tar` archive that has then been run through `gzip`.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| tar bundles, gzip compresses | `tar` flags will feel like random letters instead of two jobs glued together |
| Flag order / `-f` last | `tar cvf` vs `tar -czf out.tgz dir` — you'll hang tar on the tape drive or clobber a file |
| The tarbomb | You extract an archive and 400 loose files vomit into your home dir, mixed with your own |
| `--strip-components` | You can't cleanly extract `myapp-1.2.3/...` into `current/` on deploy |
| xz's memory cost | Your 256MB container OOMs while *decompressing* a package |
| Streaming pipes | You try to `pg_dump` 40GB to a disk with 30GB free, fill it, and take the DB down |
| `rm` vs streaming when disk is full | You can't back anything up because you have no room for the temp file |
| zip loses the +x bit | Your deploy's `entrypoint.sh` arrives non-executable and the container won't start |

---

## The Physical Reality — What a `.tar` Actually Looks Like

A tar file is **not** a directory. It's a flat concatenation of **512-byte blocks**: a header block, then the file's data (padded up to a 512-byte boundary), then the next header, and so on. Two zero-filled blocks mark the end.

```
┌─────────────────────────────────────────────────────────────────────┐
│  file1.txt.tar  (a plain, UNcompressed archive)                     │
├─────────────────────────────────────────────────────────────────────┤
│ [512B HEADER]   name="app/server.js"  mode=0644  uid=1001 gid=1001  │
│                 size=4021  mtime=...  typeflag='0'(regular)  chksum  │
│ [DATA blocks]   the actual bytes of server.js, padded to a 512 mult │
│ ─────────────────────────────────────────────────────────────────  │
│ [512B HEADER]   name="app/package.json"  mode=0644 ...              │
│ [DATA blocks]   ...                                                  │
│ ─────────────────────────────────────────────────────────────────  │
│ [512B HEADER]   name="app/bin/start"  mode=0755 ...  ◀◀ the +x bit  │
│ [DATA blocks]   ...                                     lives HERE   │
│ ─────────────────────────────────────────────────────────────────  │
│ [512B zero] [512B zero]   ← end-of-archive marker                   │
└─────────────────────────────────────────────────────────────────────┘
```

**This header is why tar preserves Unix things zip can't** — modes (the `0755` executable bit), owner/group, mtimes, and symlink targets are all in that header. `.zip` has a weaker model and drops most of it on Linux.

Now compress the *whole thing*:

```
     tar (bundle)                gzip (seal)
  app/ ───────────► app.tar ───────────────► app.tar.gz
  400 files          one           one gzip stream over the ENTIRE
  one 512B header    stream        concatenation → redundancy ACROSS
  each                              files gets exploited. Big win on
                                    e.g. node_modules (thousands of
                                    near-identical files).
```

---

## The Real Difference: `.tar.gz` vs `.zip`

This is the honest trade-off, and it's the one thing to actually understand.

```
  .tar.gz  — "solid" archive: bundle everything, THEN compress the whole stream
  ─────────────────────────────────────────────────────────────────────────────
   node_modules/  ┐
     a/index.js   │  concatenate       ┌──────────────┐   ONE gzip stream
     b/index.js   ├──► into one tar ──►│ gzip sees the│──► sees that a/index.js and
     c/index.js   │    stream          │ WHOLE stream │    b/index.js are 95% identical
     ...5000 more ┘                    └──────────────┘    → compresses the 2nd to ~nothing

   ✅ Best ratio on many similar files (node_modules, logs, source trees)
   ❌ NO random access: to read ONE file you must gunzip everything up to it.
      `tar -xzf big.tgz just/one/file` still decompresses the whole prefix.


  .zip  — compress each file INDEPENDENTLY, then bundle, with a central directory
  ─────────────────────────────────────────────────────────────────────────────
     a/index.js ──►[gzip'd separately]─┐
     b/index.js ──►[gzip'd separately]─┤──► one .zip + a CENTRAL DIRECTORY (index)
     c/index.js ──►[gzip'd separately]─┘         at the end listing every entry+offset

   ✅ RANDOM ACCESS: jump straight to any file via the central directory.
   ✅ Universal on Windows/macOS; what AWS Lambda wants.
   ❌ Can't exploit redundancy BETWEEN files → worse ratio on many-similar-files.
```

| | `.tar.gz` | `.zip` |
|---|---|---|
| Model | Solid: bundle → compress whole | Per-file: compress each → bundle |
| Ratio on many similar files | **Better** (cross-file redundancy) | Worse |
| Extract one file | Must decompress everything up to it | **Instant** (central directory) |
| Preserves Unix perms/owner/symlinks | **Yes** | Poorly / not reliably |
| Windows/macOS interop | Meh | **Native** |
| AWS Lambda deploy package | No | **Yes (zip required)** |

**Rule of thumb:** Linux-to-Linux, many files, backups, deploys → `.tar.gz` (or `.tar.zst`). Interop with Windows/macOS, or a Lambda artifact → `.zip`.

---

## How It Works — Step by Step

### Trace: `tar -czf app.tar.gz app/`

```
COMMAND:     tar -czf app.tar.gz app/

TAR DOES:    • lstat("app/") → it's a dir, so recurse (topics 03/04 — walking inodes)
             • for each entry: lstat() to read mode/uid/gid/mtime/size/symlink-target
             • build a 512-byte header block from that metadata
             • open() the file, read() its bytes
             • because of -z, tar does NOT write to disk directly. It pipes the
               tar stream into an in-process gzip (zlib) — or forks the `gzip`
               binary — and writes the COMPRESSED result to app.tar.gz (the -f target)

GZIP DOES:   • DEFLATE = LZ77 (replace repeated byte sequences with back-references)
               + Huffman coding (short codes for common bytes)
             • emits the compressed stream

KERNEL:      write(fd_of_app.tar.gz, compressed_bytes, n)

RESULT:      app.tar.gz — a gzip-wrapped tar archive
```

### Trace: the streaming pipeline `tar -czf - app/ | ssh host 'tar -xzf - -C /srv'`

```
  tar -czf -              |            ssh host           |    tar -xzf - -C /srv
  ─────────────           │            ──────────         │    ────────────────────
  bundle app/, gzip it,   │  the       encrypt the        │  read the stream from
  write to STDOUT ('-')   ├─pipe──────►byte stream,       ├──►STDIN ('-'), gunzip it,
  (NO temp file created)  │  (topic    send to host,      │   recreate the tree
                          │   12)      decrypt on arrival  │   under /srv
                          ▼                                ▼
              NO 2GB temp file ever exists.        NO temp file on the far side.
              Disk usage on BOTH ends: ~0 extra.   This is the killer feature.
```

`-` means "use stdout" (for `-f -` on create) or "use stdin" (for `-f -` on extract). This is the hinge that makes tar composable.

---

## `tar` Flags — Annotated Hard

The mnemonic matters because tar's flag *clustering* is where people get burned.

```
tar -c z v f  out.tar.gz  dir/
│   │ │ │ │    │          │
│   │ │ │ │    │          └── what to archive (the source paths)
│   │ │ │ │    └── THE ARCHIVE FILE. Must be the arg RIGHT AFTER -f.
│   │ │ │ └── -f FILE: operate on this file. ◀◀ -f MUST BE LAST in the cluster,
│   │ │ │            because it CONSUMES THE NEXT ARGUMENT as the filename.
│   │ │ │            Forget -f entirely → tar tries the default TAPE DRIVE
│   │ │ │            (/dev/st0) or stdin and just HANGS or errors. A classic.
│   │ │ └── -v: verbose — list each file as it's processed
│   │ └── -z: filter through GZIP  (-j bzip2, -J xz, --zstd, -a = auto by extension)
│   └── -c: CREATE a new archive  (-x extract, -t list, -r append, -u update)
└── tar
```

| Flag | Long form | What it does | Trap |
|---|---|---|---|
| `-c` | `--create` | Create an archive | |
| `-x` | `--extract` | Extract | Extracts into the **current dir** unless `-C` given |
| `-t` | `--list` | **List contents WITHOUT extracting** | **`tar -tzf x.tgz \| head` an untrusted archive FIRST** |
| `-f FILE` | `--file` | The archive file to use | **Must be last in the cluster / take the next arg. Forgetting it = hang.** |
| `-v` | `--verbose` | List files as processed | Skip it in scripts; it's slow on huge trees |
| `-z` | `--gzip` | Filter through gzip | |
| `-j` | `--bzip2` | Filter through bzip2 | |
| `-J` | `--xz` | Filter through xz | **Capital J.** Memory-hungry to decompress |
| `--zstd` | | Filter through zstd | Needs a modern tar (1.31+); missing on old boxes |
| `-a` | `--auto-compress` | **Pick the compressor from the OUTPUT extension** | `tar -caf x.tar.zst dir` → uses zstd automatically |
| `-C DIR` | `--directory` | `cd` to DIR **before** acting | **The right way to control where things land — don't `cd` around** |
| `--exclude=PAT` | | Skip matching paths | Put it **before** the source paths; it's a glob, quote it |
| `-p` | `--preserve-permissions` | Restore exact modes | Default when root extracts; use it as non-root to keep modes |
| `--strip-components=N` | | Drop the first N path components on extract | **How you extract `myapp-1.2.3/*` into the CWD** |
| `--same-owner` / `--no-same-owner` | | Restore original uid/gid or not | **Root defaults to `--same-owner` → files owned by whatever uid the archive recorded** |
| `-r` / `-u` | `--append` / `--update` | Add/refresh files in an **uncompressed** tar | Can't append to a `.gz` — it's a single compressed stream |
| `--one-file-system` | | Don't cross into other mounts | Stops a `/` backup wandering into `/proc`, `/sys`, network mounts |
| `-P` | `--absolute-names` | **Keep leading `/`** | **DANGEROUS — normally tar strips it for safety. Don't use it.** |

### The mnemonics people actually remember

```
tar -czvf out.tar.gz dir/     "Create Ze Vucking File"   (compress)
tar -xzvf in.tar.gz           "eXtract Ze Vucking File"  (extract)
tar -tzf  in.tar.gz | head    "look inside first"        (list — DO THIS on untrusted)
```

Modern GNU tar can also **auto-detect** the compressor on extract, so `tar -xf archive.tar.zst` works without `--zstd`. On create you still choose (`-z`/`-J`/`--zstd`/`-a`).

---

## Compressors Compared — The Honest Table

| Tool | Algorithm | Compress speed | Ratio | Decompress speed | Decompress RAM | Use when |
|---|---|---|---|---|---|---|
| **gzip** | DEFLATE (LZ77+Huffman) | Fast | Baseline | Fast | Tiny | **The safe default.** Always installed, everywhere. |
| **bzip2** | Burrows–Wheeler | Slow | ~10–15% better than gzip | Slow | Moderate | Rarely worth it now — superseded. You'll still *receive* `.bz2`. |
| **xz** | LZMA2 | **Very slow** | **Best (or near)** | Medium | **HIGH — hundreds of MB** | Distro packages, release tarballs you compress **once** and download **many** times. |
| **zstd** | LZ77 + FSE/entropy | **Fast (gzip-class)** | **xz-class at high levels** | **Blazing** | Low | **The modern winner. Use it if available.** Backups, deploys, anything. |
| **lz4** | LZ77 (speed-tuned) | **Insanely fast** | Low | **Insanely fast** | Tiny | Real-time / throughput-bound (e.g. ZFS, transient streams). |

**The two things that actually bite in production:**

1. **`xz` decompression is memory-hungry.** Decompressing an `xz -9` file can need **several hundred MB of RAM**. On a 256MB container or a tiny VPS this **OOM-kills** the decompressor (topic 01/34). zstd needs a fraction of that. If you distribute artifacts that get unpacked in small containers, don't use `xz -9`.
2. **`-9` is usually not worth it.** The level knob is `-1` (fast, bigger) to `-9` (slow, smaller). Going from `-6` (the gzip default) to `-9` typically buys **1–3% smaller** for **much** more CPU. Use `-6` for gzip; for zstd, `-19` is the high end and `--long` helps on big inputs; `zstd -T0` uses all cores.

```
gzip -1 file      # fast, larger
gzip -9 file      # slow, marginally smaller — usually NOT worth it vs -6
zstd -19 -T0 file # near-xz ratio, all CPU cores, still fast to DEcompress
xz  -9 file       # smallest, slowest, heavy to decompress — for "compress once, ship often"
```

### ⚠️ Compressors REPLACE the original by default

```bash
gzip app.log
ls                     # app.log is GONE. There is now app.log.gz.
                       # This surprises everyone the first time.
```

To keep the original:

```bash
gzip -k app.log        # -k = --keep → app.log AND app.log.gz
gzip -c app.log > app.log.gz   # or redirect stdout, leaving the original alone
```

Same for `xz`, `bzip2`, `zstd` (all support `-k`). Decompression (`gunzip`/`gzip -d`) *also* deletes the `.gz` and restores the original — use `-k` there too.

### Reading compressed files WITHOUT decompressing to disk

```bash
zcat  app.log.2.gz              # cat, decompressing on the fly to stdout
zless app.log.2.gz             # page through it (topic 10)
zgrep "ECONNREFUSED" /var/log/app.log.*.gz   # grep rotated logs (topic 11/23) ◀◀ constant use
zdiff old.gz new.gz            # diff two compressed files
gzip -t archive.gz             # TEST integrity (decompress to /dev/null, report errors). No output = OK.
gzip -l archive.gz             # List: compressed size, uncompressed size, ratio
zstd -t file.zst               # zstd's integrity test
```

`zgrep` on a mix of `app.log app.log.1 app.log.2.gz` transparently handles both plain and gzipped members — which is exactly what you want when searching rotated logs.

---

## `zip` / `unzip` — For Interop

Use zip when you **must** cross to Windows/macOS, or build an **AWS Lambda** package.

```
zip -r release.zip dist/ -x '*.git*' '*node_modules*'
│   │  │           │     │
│   │  │           │     └── -x: exclude patterns (quote them)
│   │  │           └── what to add
│   │  └── output archive
│   └── -r: recurse into directories (WITHOUT -r, zip adds the dir entry but not contents!)
└── zip
```

```bash
unzip -l release.zip           # LIST contents (like tar -t) — do this first
unzip release.zip -d /srv/app  # -d: extract INTO this dir (creates it)
unzip -o release.zip           # -o: overwrite without prompting
```

### ⚠️ zip does NOT preserve Unix permissions or symlinks reliably

The classic breakage:

```bash
chmod +x deploy/entrypoint.sh
zip -r release.zip deploy/
# ...ship to a Linux host, unzip...
ls -l deploy/entrypoint.sh
# -rw-r--r--   ◀◀ the +x bit is GONE. Container: "exec: entrypoint.sh: permission denied"
```

`zip`/`unzip` *can* store Unix modes (Info-ZIP does), but it's unreliable across implementations, and a file that round-trips through Windows loses them entirely. **For a Linux deploy artifact, use `tar.gz`** — the executable bit lives safely in the tar header. Only use zip when the other end genuinely requires it (Lambda, a Windows user).

---

## ⭐ Streaming / Pipes — The Most Valuable Section

Because `tar` and the compressors read stdin and write stdout, you can compress **and** move data **without ever writing a temp file**. This is not a party trick — it's how you back up and migrate when **the disk is nearly full** or **the payload is bigger than your free space** (topic 12 — pipes).

```bash
# Copy a whole directory tree across the network in ONE shot — no temp file, either end
tar -czf - /srv/app | ssh deploy@web-2 'tar -xzf - -C /srv'
#        │  │                          │        │  └── -C: land it under /srv on web-2
#        │  │                          │        └── read the archive from STDIN
#        │  └── source                 └── the pipe carries the stream (encrypted by ssh)
#        └── write archive to STDOUT

# Dump a 40GB database — compressed, NEVER landing 40GB of plaintext on disk
pg_dump mydb | gzip > backup.sql.gz
#            │       └── only the COMPRESSED result touches disk (~4GB, not 40GB)
#            └── stream straight into gzip

# Dump + compress + ship to a backup host, all in-flight, zero local temp
mysqldump --single-transaction mydb | gzip | ssh backup@nas 'cat > /backups/mydb-$(date +%F).sql.gz'

# Move a Docker image between hosts with no registry
docker save myapp:1.4.2 | gzip | ssh web-2 'gunzip | docker load'

# Restore: decompress on the fly straight into psql
zcat backup.sql.gz | psql mydb
# or:  gunzip -c backup.sql.gz | psql mydb

# Size an archive WITHOUT creating it (how big WOULD the backup be?)
tar -czf - /srv/app | wc -c
# 41288302   ← ~41 MB, and not a single byte was written to disk

# Add a progress bar with `pv` (pipe viewer) — see throughput/ETA on a long stream
tar -czf - /srv/app | pv -s $(du -sb /srv/app | cut -f1) | ssh web-2 'tar -xzf - -C /srv'
#                     │  └── -s: total size, so pv can show a % and ETA
#                     └── pv passes bytes through unchanged and draws a bar on stderr
```

**Why this matters at 2am (the disk-full case):** your `/` is at 96%. You need a backup before you touch anything. `pg_dump mydb > backup.sql` would need 40GB of free space you don't have and would push you to 100% (topic 23 — that takes the server down). `pg_dump mydb | gzip | ssh backup 'cat > /b/mydb.sql.gz'` needs **essentially zero local disk** — the bytes flow out over the network as they're produced.

---

## Exact Syntax Breakdown

```
tar -xzf release.tar.gz --strip-components=1 -C /srv/app/current
│   ││ │  │              │                    │  │
│   ││ │  │              │                    │  └── extract INTO here
│   ││ │  │              │                    └── -C: change to this dir first
│   ││ │  │              └── drop the FIRST path component:
│   ││ │  │                    myapp-1.2.3/server.js  →  server.js
│   ││ │  │                    (so a wrapper dir doesn't nest under current/)
│   ││ │  └── the archive
│   ││ └── -f: this file
│   │└── -z: gunzip it (modern tar auto-detects, but be explicit)
│   └── -x: extract
└── tar
```

```
tar -tzf suspicious.tar.gz | head
│   ││ │  │                  │
│   ││ │  │                  └── just peek at the first few entries
│   ││ │  └── the archive
│   ││ └── -f
│   │└── -z
│   └── -t: LIST, don't extract. ◀◀ ALWAYS do this on an archive you didn't make,
│          to catch a tarbomb (loose files) or path traversal (../ , /etc/...).
└── tar
```

```
zstd -19 -T0 --long=27 -o backup.tar.zst backup.tar
│    │   │   │          │  │              │
│    │   │   │          │  │              └── input
│    │   │   │          │  └── output file
│    │   │   │          └── -o: output name (else it appends .zst and deletes input)
│    │   │   └── --long: bigger match window (better ratio on large inputs)
│    │   └── -T0: use ALL cpu cores
│    └── -19: high compression level (1..19; 20..22 need --ultra)
└── zstd
```

---

## Example 1 — Basic

```bash
# --- Make a test tree ---
mkdir -p demo/src demo/bin
echo "console.log('hi')" > demo/src/app.js
echo '#!/bin/sh'         > demo/bin/start
chmod +x demo/bin/start                 # ◀◀ note the executable bit — watch it survive

# --- ARCHIVE + COMPRESS with gzip ---
tar -czvf demo.tar.gz demo/
# demo/
# demo/src/app.js
# demo/bin/start

# --- LIST before trusting/extracting (habit!) ---
tar -tzf demo.tar.gz
# demo/
# demo/src/app.js
# demo/bin/start
#   ↑ every path is under demo/ → NOT a tarbomb. Safe to extract anywhere.

# --- Prove tar preserved the +x bit (this is the whole point vs zip) ---
rm -rf demo && tar -xzf demo.tar.gz
ls -l demo/bin/start
# -rwxr-xr-x 1 you you 10 Jul 12 14:20 demo/bin/start   ← +x SURVIVED ✅

# --- Compare compressors on the same input ---
tar -cf demo.tar demo/                              # uncompressed baseline
gzip -k -6 -c demo.tar | wc -c                      # gzip
xz   -k -9 -c demo.tar | wc -c                      # xz (smaller, slower)
zstd -k -19 -c demo.tar | wc -c                     # zstd (small AND fast)

# --- The "replaces original" surprise ---
cp demo.tar keepme.tar
gzip keepme.tar          # keepme.tar VANISHES; keepme.tar.gz appears
ls keepme*               # keepme.tar.gz
gzip -dk keepme.tar.gz   # -d decompress, -k keep the .gz too
ls keepme*               # keepme.tar   keepme.tar.gz

# --- Read a compressed file without unpacking it ---
gzip -c demo.tar > demo.tar.gz2
zcat demo.tar.gz2 | tar -t | head       # peek inside without ever writing a .tar

# cleanup
rm -rf demo demo.tar* keepme* 
```

---

## Example 2 — Production Scenario

**Two tasks on a Friday: (1) take a nightly Postgres backup that won't fill the disk, and (2) deploy a release tarball.**

```bash
# ─────────────────────────────────────────────────────────────────────────
# TASK 1 — nightly Postgres backup script (runs from cron — topic 20)
# Goal: compressed, dated, streamed (never 40GB of plaintext on disk),
#       uploaded to S3, and OLD backups pruned so /backups never fills.
# ─────────────────────────────────────────────────────────────────────────
$ cat /usr/local/bin/pg-backup.sh
#!/usr/bin/env bash
set -euo pipefail
DB=mydb
STAMP=$(date +%F-%H%M)                    # dateext-style: 2026-07-12-0300
OUT="/backups/${DB}-${STAMP}.sql.gz"

# STREAM: pg_dump → gzip → file. The 40GB plaintext NEVER exists on disk.
pg_dump --format=plain "$DB" | gzip -6 > "$OUT"

# integrity check — a truncated backup is worse than no backup
gzip -t "$OUT"                            # exits nonzero (and `set -e` aborts) if corrupt

# ship it off-box
aws s3 cp "$OUT" "s3://acme-backups/pg/"

# PRUNE local backups older than 30 days so /backups never fills (topic 08/20/23)
find /backups -name "${DB}-*.sql.gz" -mtime +30 -delete
#             │                      │           └── delete the matches
#             │                      └── modified MORE than 30 days ago
#             └── only our backup files, never anything else

$ sudo /usr/local/bin/pg-backup.sh
$ ls -lh /backups/ | tail -3
-rw-r--r-- 1 postgres postgres 3.9G Jul 12 03:00 mydb-2026-07-12-0300.sql.gz
#                              ^^^^ 3.9G compressed — the plaintext would've been ~38G

# Restore drill (you MUST test restores — an untested backup is a rumour):
$ zcat /backups/mydb-2026-07-12-0300.sql.gz | psql -h 127.0.0.1 mydb_restore_test

# ─────────────────────────────────────────────────────────────────────────
# TASK 2 — deploy a release tarball built by CI
# CI produced myapp-1.4.2.tar.gz whose TOP-LEVEL dir is "myapp-1.4.2/"
# ─────────────────────────────────────────────────────────────────────────
$ tar -tzf myapp-1.4.2.tar.gz | head -3        # ◀◀ ALWAYS look first
myapp-1.4.2/
myapp-1.4.2/package.json
myapp-1.4.2/server.js
#   ↑ everything is under one wrapper dir. If we extract naively into current/,
#     we get current/myapp-1.4.2/... — an extra level the app doesn't expect.

$ sudo mkdir -p /srv/app/releases/1.4.2

# --strip-components=1 drops "myapp-1.4.2/" so files land DIRECTLY in the target
$ sudo tar -xzf myapp-1.4.2.tar.gz --strip-components=1 -C /srv/app/releases/1.4.2
$ ls /srv/app/releases/1.4.2
package.json  server.js  node_modules  bin        ← clean, no wrapper dir ✅

# atomically flip the "current" symlink (topic 03/04 — a rename is atomic)
$ sudo ln -sfn /srv/app/releases/1.4.2 /srv/app/current
$ sudo systemctl restart myapp
$ curl -sf localhost:3000/healthz && echo " deployed 1.4.2 ✅"
{"status":"ok"} deployed 1.4.2 ✅

# ─────────────────────────────────────────────────────────────────────────
# BONUS — migrate /srv/app to a new box with NO intermediate file, NO registry
# ─────────────────────────────────────────────────────────────────────────
$ tar -czf - -C /srv app | pv | ssh web-2 'tar -xzf - -C /srv'
# 4.12GiB 0:00:41 [ 102MiB/s]      ← pv shows live throughput; nothing hit either disk
```

---

## Common Mistakes

### Mistake 1 — The TARBOMB

**Wrong:** `tar -xzf random-download.tar.gz` in your home directory.
**Root cause:** a well-behaved archive puts everything under **one top dir** (`pkg/...`). A tarbomb's members are at the **top level** (`file1`, `file2`, `Makefile`, ...). Extracting it scatters hundreds of loose files into your CWD, tangled with your own files — and there's no clean "undo."
**What the broken state looks like:**
```bash
$ ls
file1.c  file2.c  Makefile  README  config  my-actual-project/  notes.txt
#  ↑ which of these were here before?? Good luck.
```
**Fix / Prevent:** **list first**, and **extract into a fresh dir**:
```bash
tar -tzf random-download.tar.gz | head        # do the members share a top dir?
mkdir unpack && tar -xzf random-download.tar.gz -C unpack   # contain the blast radius
```

---

### Mistake 2 — Path traversal / absolute paths (a security issue)

**Wrong:** `sudo tar -xf untrusted.tar` as root, without inspecting it.
**Root cause:** by default tar **strips a leading `/`** ("Removing leading `/' from member names") and refuses `../` escapes — good. But `-P`/`--absolute-names` **disables** that, and a malicious archive containing `../../../etc/cron.d/pwn` or `/etc/passwd` can then write **outside** the extraction directory. As root, that's a full compromise.
**What you'd see (the safe default protecting you):**
```bash
$ tar -xf weird.tar
tar: Removing leading `/' from member names        ← tar defending you
tar: Removing leading `../' from member names
```
**Fix / Prevent:** never use `-P`. Never extract an untrusted archive as root. Inspect with `tar -tvf` first (look for names starting with `/` or `../`). Extract into a throwaway dir.

---

### Mistake 3 — Forgetting `-f` / wrong flag order

**Wrong:** `tar -czv backup.tar.gz src/`  (no `-f`)
**Root cause:** without `-f`, tar writes to its **default device** — historically the tape drive `/dev/st0`, or stdin/stdout. It either errors (`tar: can't open /dev/st0`) or **hangs waiting on stdin**, and `backup.tar.gz` gets misread as *a file to archive*.
**Also wrong:** `tar -cfz backup.tar.gz src/` — here `-f` consumes `z` as the filename, so it tries to create an archive literally named `z` and treats `backup.tar.gz` as input. Chaos.
**Right:** keep `-f` **last** in the cluster, immediately followed by the filename: `tar -czf backup.tar.gz src/`.
**Prevent:** internalise "**f is for file, and file comes last.**"

---

### Mistake 4 — `tar -czf backup.tar.gz /` (backing up a live root)

**Wrong:** `sudo tar -czf /backup/full.tar.gz /`
**Root cause:** `/` contains **virtual filesystems** — `/proc` (kernel state), `/sys`, `/dev` (device files), plus `/backup` itself. `/proc/kcore` alone appears as a **128TB** file. tar will try to read infinite/garbage data, may recurse into the archive it's writing, and fills the disk (topic 23).
**What you'd see:** the archive balloons past your total disk size; `tar: /proc/kcore: File shrank`; the disk hits 100%.
**Fix / Prevent:**
```bash
sudo tar -czf /backup/full.tar.gz \
  --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run \
  --exclude=/backup --one-file-system /
#                    └── --one-file-system: don't cross mount points (skips /proc, /sys,
#                        network mounts, and other disks automatically)
```

---

### Mistake 5 — Compressing already-compressed data

**Wrong:** `gzip photos.zip`, `tar -czf media.tar.gz *.mp4`, `xz already.gz`.
**Root cause:** `.jpg`, `.png`, `.mp4`, `.gz`, `.zst`, `.zip` are **already compressed** — their redundancy is gone. Running a compressor over them burns CPU for ~0% gain, and gzip's framing overhead can make the result **slightly larger** than the input.
**What you'd see:** `gzip -l` shows `-0.1%` or a bigger file; a huge CPU spike for nothing.
**Fix / Prevent:** archive them with **no** compression (`tar -cf media.tar *.mp4`, or `tar --zstd` at level 1 just for the container), or store as-is. Compress the *text* (source, JSON, SQL dumps, logs) where the wins are 5–20×.

---

### Mistake 6 — Using `zip` for a Linux deploy artifact

**Wrong:** `zip -r release.zip dist/` for a container/host deploy whose entrypoint needs `+x`.
**Root cause:** zip's Unix-mode support is unreliable and often lost; `unzip` on the target restores `entrypoint.sh` as `0644`.
**What breaks:** `exec /app/entrypoint.sh: permission denied` — the container crash-loops.
**Fix / Prevent:** use `tar -czf release.tar.gz dist/` for Linux deploys (the mode is in the tar header). If you're forced to use zip, `chmod +x` after `unzip`, or restore modes explicitly. Reserve zip for genuine interop (Lambda, Windows).

---

## Hands-On Proof

```bash
# PROVE IT: archiving and compressing are separate — see the two stages
mkdir -p t/a t/b; head -c 100000 /dev/urandom > t/a/f1; cp t/a/f1 t/b/f1
tar -cf t.tar t/              # STAGE 1: bundle only. No shrinkage.
ls -l t.tar                   # ~200KB+ (both copies, uncompressed)
gzip -k t.tar                 # STAGE 2: compress the bundle
ls -l t.tar t.tar.gz          # the .gz is a fraction — the two identical f1 files
                              # collapse because gzip sees the WHOLE stream
rm -rf t t.tar t.tar.gz

# PROVE IT: .tar.gz beats .zip on many-similar-files (cross-file redundancy)
mkdir m; for i in $(seq 1 200); do seq 1 500 > m/file$i.txt; done   # 200 identical files
tar -czf m.tgz m/ ; zip -qr m.zip m/
ls -l m.tgz m.zip             # m.tgz is MUCH smaller — solid compression wins
rm -rf m m.tgz m.zip

# PROVE IT: a tar file is 512-byte blocks with readable headers
tar -cf h.tar /etc/hostname
xxd h.tar | head -4           # you can literally READ "etc/hostname" in the header block
rm h.tar

# PROVE IT: tar strips the leading slash by default (the safety net)
tar -cf abs.tar /etc/hostname 2>&1        # tar: Removing leading `/' from member names
tar -tf abs.tar                           # etc/hostname   ← the slash is gone
rm abs.tar

# PROVE IT: --strip-components drops path levels
mkdir -p wrap/inner; echo hi > wrap/inner/f
tar -czf w.tgz wrap/
mkdir out && tar -xzf w.tgz --strip-components=1 -C out
ls out                        # inner/  ← "wrap/" was stripped
rm -rf wrap out w.tgz

# PROVE IT: streaming creates NO temp file, and you can size it in-flight
mkdir big; head -c 5000000 /dev/urandom > big/data
tar -czf - big/ | wc -c       # prints a byte count; NO archive file was written
ls | grep -c '\.tar'          # 0 — nothing landed on disk
rm -rf big

# PROVE IT: xz needs far more RAM to decompress than gzip (concept check)
head -c 5000000 /dev/zero > z.bin
gzip  -9 -c z.bin | xz  -9 -c | wc -c    # (just observe both run; xz is slower/heavier)
rm -f z.bin

# PROVE IT: integrity test catches corruption
echo hello | gzip > good.gz
gzip -t good.gz && echo "good.gz OK"
printf 'x' | dd of=good.gz bs=1 seek=5 conv=notrunc 2>/dev/null   # corrupt a byte
gzip -t good.gz || echo "good.gz is now CORRUPT (as expected)"
rm -f good.gz
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal:
1. Make a directory with 3 files, one of them `chmod +x`.
2. Create `demo.tar.gz` from it. **List** its contents with `tar -tzf` — confirm every path shares a top-level dir.
3. Delete the directory, extract the archive, and prove with `ls -l` that the executable bit survived.
4. Compress a copy of a plain text file with `gzip` and observe the original disappears. Recover it with `gunzip -k` and confirm both files exist.
5. Run `gzip -l` on your `.gz` and read off the compression ratio.

Paste your output.

---

### Exercise 2 — Medium

Compare the tools on real data (grab any large-ish text file — e.g. `journalctl > /tmp/j.txt`).

1. Produce `j.gz`, `j.bz2`, `j.xz`, `j.zst` (keep the original each time with `-k`).
2. Build a table: for each, the **file size** and the **wall-clock time** to compress (`time`).
3. Now time the **decompression** of each. Which is fastest to decompress? Which is smallest?
4. Compress the same file at `-1`, `-6`, and `-9` with gzip. How many bytes does `-9` actually save over `-6`, and how much longer did it take? Was it worth it?
5. Try to `gzip` an already-`.gz` file. Show with `gzip -l` (or `ls -l`) that it gained ~nothing (or grew).

Paste your table and commands.

---

### Exercise 3 — Hard (Production Simulation)

Do a full backup-and-migrate, entirely streamed, then handle two hostile archives.

1. Create `/tmp/srv/app` with a nested structure and a fake 20MB "database" (`head -c 20000000 /dev/urandom`).
2. **Stream** it to a second location with **no intermediate archive file**: `tar -czf - -C /tmp/srv app | tar -xzf - -C /tmp/dest` (simulate the two-host case locally). Verify the tree and permissions match on both sides.
3. Add `pv` to show a live progress bar during the stream.
4. **Simulate a DB backup that mustn't fill the disk:** `head -c 50000000 /dev/urandom | gzip > /tmp/db.sql.gz` and prove with `ls -l` that the on-disk result is far smaller than 50MB would be if it were text (use a text source to make the point: `yes "INSERT INTO t VALUES (1);" | head -c 50000000 | gzip | wc -c`).
5. **Tarbomb drill:** build an archive whose members are at the top level (`tar -czf bomb.tgz *.txt` from inside a dir of loose files). Prove with `tar -tzf` that it's a bomb, then extract it *safely* into a throwaway dir with `-C`.
6. **Strip-components drill:** build `app-9.9.9.tar.gz` with a wrapper dir, then extract it so the files land directly in `/tmp/release` with no wrapper.

Paste your full session.

---

## Mental Model Checkpoint

Answer from memory.

1. **What are the two separate jobs, which tool does each, and why does `.tar.gz` need both?**
2. **Why does `.tar.gz` usually compress a `node_modules` tree better than `.zip`, and what does `.zip` give you in return?**
3. **In `tar -czf out.tar.gz dir/`, why must `-f` come last, and what happens if you forget `-f` entirely?**
4. **Which compressor is the modern default choice and why? What specific production problem does `xz -9` cause on a small container?**
5. **You run `gzip app.log`. What happened to `app.log`? How do you keep the original?**
6. **Write the one-line pipeline that dumps a 40GB Postgres database, compressed, to a backup host, without ever writing a large temp file locally.**
7. **What is a tarbomb, and what two habits prevent it from wrecking your CWD?**
8. **What does `--strip-components=1` do, and when do you need it?**

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `tar -czf out.tgz dir/` | Create a gzip'd archive | `-c` create, `-z` gzip, `-j` bzip2, `-J` xz, `--zstd`, `-a` auto-by-ext, `-v` verbose, **`-f` last** |
| `tar -xzf in.tgz` | Extract | `-C DIR` (extract into), **`--strip-components=N`**, `-p` preserve perms, `--no-same-owner` |
| `tar -tzf in.tgz` | **List without extracting** | **Do this on any untrusted archive first** |
| `tar -czf - dir \| ...` | Stream to stdout | `-` = stdout/stdin — the pipe hinge |
| `gzip` / `gunzip` | Compress/decompress one stream (DEFLATE) | `-k` keep original, `-c` to stdout, `-d` decompress, `-1..-9`, `-t` test, `-l` list |
| `xz` | Best ratio, slow, **heavy to decompress** | `-k`, `-9`, `-T0` (threads), `-d` |
| `zstd` | **Modern default: fast + great ratio** | `-k`, `-1..-19` (`--ultra` for 20–22), `-T0`, `--long`, `-o out` |
| `bzip2` | Legacy middle ground | `-k`, `-d` |
| `zcat`/`zless`/`zgrep`/`zdiff` | Read/search compressed WITHOUT unpacking | Work on `.gz` directly (rotated logs — topic 23) |
| `zip -r out.zip dir/` | Cross-platform archive (per-file) | `-r` **required for dirs**, `-x` exclude, `-9` |
| `unzip` | Extract a zip | `-l` list, `-d DIR`, `-o` overwrite |
| `pv` | Progress bar for a pipe | `-s SIZE` for % + ETA |

---

## When Would I Use This at Work?

### Scenario 1: Nightly database backups from cron
Your cron job (topic 20) runs `pg_dump mydb | gzip > /backups/mydb-$(date +%F).sql.gz`, then `find /backups -name 'mydb-*.sql.gz' -mtime +30 -delete` to prune. Streaming means a 40GB database becomes a ~4GB file **without** the 40GB plaintext ever touching disk — so the backup itself can't fill the volume and take the DB down (topic 23).

### Scenario 2: Deploying a release tarball
CI hands you `myapp-1.4.2.tar.gz` with a wrapper directory. You `tar -tzf` to look, then `tar -xzf ... --strip-components=1 -C /srv/app/releases/1.4.2`, flip the `current` symlink atomically, and restart. The executable bits on your scripts survive because they're in the tar header — something a `.zip` would have silently dropped.

### Scenario 3: Migrating a service to a new host with no registry
No time to stand up a registry or an S3 bucket. `tar -czf - -C /srv app | ssh web-2 'tar -xzf - -C /srv'` (topic 24) moves the entire tree — permissions, symlinks, and all — over an encrypted pipe, with zero temp files on either machine. Add `pv` to watch it go.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 03 — Everything Is a File | tar headers store symlink targets, device nodes, file types |
| **Builds on** | 04 — Inodes | tar reads mode/uid/gid/mtime from each inode; the +x bit |
| **Builds on** | 09 — Working with Files | `du`/`df` to know if an archive will fit; `find -mtime` to prune |
| **Builds on** | 11 — Text Processing | `zgrep` rotated logs; `awk` over decompressed streams |
| **Builds on** | 12 — Pipes and Redirection | The entire streaming section is `|` and `-` (stdout/stdin) |
| **Builds on** | 23 — Logs | Rotated logs are `.gz`; `zgrep`; why you compress to save disk |
| **Builds on** | 24 — SSH | `tar | ssh | tar`; SSH is the transport for streamed archives |
| **Next** | 26 — Network Interfaces | Phase 5 — what actually carries your streamed archives |
| **Used by** | 31 — Disk Management | Compression is how you survive a full disk; streaming avoids temp files |
| **Used by** | 33 — Node in Production | Building and shipping deploy artifacts (tar.gz, not zip) |
