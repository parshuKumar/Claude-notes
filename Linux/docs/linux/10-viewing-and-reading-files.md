# 10 — Viewing and Reading Files

## ELI5 — The Simple Analogy

Imagine a **1,000-page book**, and four different people who want to "look at it."

- **`cat`** rips out every page, throws them all in the air at once, and shouts every word at you simultaneously. Fine for a 2-page pamphlet; insane for the 1,000-page book — and once he starts shouting you can't easily make him stop.
- **`less`** is a **librarian with a bookmark**. She shows you *one page at a time* and lets you flip forward, flip back, and jump to any chapter — never memorizing the whole book, just keeping a finger on your current page.
- **`head`** / **`tail`** read only the **first** or **last** page and hand the book back.
- **`tail -f`** **sits with the book open at the last page while the author is still writing it**, reading each new sentence aloud the moment the ink dries. That's a live log.

The topic is about knowing *which person to send*. Sending `cat` to read a 4 GB production log is how you lock up your SSH session at 2am.

---

## Where This Lives in the Linux Stack

```
Hardware (SSD/disk — the actual bytes)
  └── KERNEL
       │    • Page cache  — keeps recently-read file blocks in RAM
       │    • VFS         — path → inode → data blocks (topic 04)
       │    • inotify     — tells a process "this file changed"  ◀── tail -f uses this
       │    • TTY layer   — the terminal driver your output must squeeze through
       │
       └── System Calls
            │    openat() — get a file descriptor
            │    read()   — pull bytes from the fd into a buffer
            │    lseek()  — MOVE the read position WITHOUT reading  ◀◀◀ this is what makes `less` a pager
            │    fstat()  — "how big is this file now?"             ◀◀◀ this is how `tail -f` notices growth
            │    write()  — push bytes to fd 1 (stdout)
            │
            └── C Standard Library (glibc — fopen/fread/getline wrap the above)
                 │
                 └── Shell (bash — sets up stdin/stdout, builds pipelines)
                      │
                      └── COMMANDS: cat, less, head, tail, wc,   ◀◀◀ THIS TOPIC
                                    zcat, zgrep, strings, xxd, watch
```

**The single most important idea in this topic:** `cat` only knows `read()` + `write()` in a loop. `less` also knows **`lseek()`**. That one extra syscall is the entire difference between "dump 4 GB at my face" and "show me page 1 and wait."

---

## What Is This?

These are the commands that let you **look at the contents of a file** without opening an editor. Some dump everything to your screen (`cat`), some paginate and let you seek around (`less`), some show only the edges (`head`, `tail`), and one of them — `tail -f` — stays attached to a file while your Node app is actively writing to it.

Reading logs is roughly 60% of what you actually do on a production server. This topic is that 60%.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `cat` streams the whole file to the TTY | You `cat /var/log/nginx/access.log` (4 GB) at 2am, your SSH session floods, Ctrl-C doesn't feel like it works, and you sit there for 90 seconds watching garbage scroll while the site is down |
| `less` seeks instead of loading | You'll open logs in `vim` or `nano`, which DO try to load the file into memory, and you'll OOM the box you were trying to debug |
| `tail -f` follows the **inode**, not the name | logrotate rotates your log at midnight. Your `tail -f` silently keeps following the *rotated* file forever. You see zero new lines and conclude "the app is quiet" — while it is actually throwing 500s into the *new* file |
| `less +F` exists | You'll use `tail -f`, see a stack trace fly past, and have no way to scroll back to read it |
| `zgrep` / `zcat` exist | You'll `gunzip` a 2 GB rotated log on a box with 500 MB free and fill the disk — taking the app down *harder* |
| `xxd` / `od` exist | You'll spend three hours debugging a JSON parse error that is actually a UTF-8 BOM, or a bash script that fails with `$'\r': command not found` |

---

## The Physical Reality

### What a file actually is when you "view" it

From topics 03 and 04: a file is an **inode** (metadata + pointers to data blocks) plus the **data blocks**. A filename is just a directory entry pointing at an inode number. When you view a file, this is what exists in memory:

```
┌──────────────────────────────────────────────────────────┐
│                    YOUR PROCESS (less)                    │
│   fd 0 stdin  → TTY      fd 1 stdout → TTY                │
│   fd 3 /var/log/app.log ───────────────────────────┐      │
│   Buffer in RAM: [ 64 KB window ] ◀── ALL less holds│      │
└────────────────────────────────────────────────────┼──────┘
                                                     │
════════════════ SYSCALL BOUNDARY ═══════════════════╪═══════
                                                     ▼
┌──────────────────────────────────────────────────────────┐
│                         KERNEL                            │
│   Open File Description                                   │
│   ┌─────────────────────────────────────────────┐         │
│   │ inode:       2097241                         │         │
│   │ file offset: 4,194,304  ◀── lseek() MOVES    │         │
│   │ flags:       O_RDONLY       this number.     │         │
│   └─────────────────────────────────────────────┘         │
│              │  THAT is how a pager works.                │
│              ▼                                            │
│   Page Cache  [blk 512][blk 513][blk 514]  ← only the     │
│               pages actually touched                      │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
             DISK: 4 GB of data blocks
                   (99.9% of them never read)
```

### `cat` vs `less` — the actual difference in syscalls

```
cat bigfile.log                     less bigfile.log
──────────────────                  ─────────────────────────────────
open() → fd 3                       open() → fd 3;  fstat() → 4 GB
loop until EOF:                     read(3, buf, 65536)  → first 64 KB only
  read(3, buf, 131072)              write(1, ...)        → paint ONE screen, WAIT
  write(1, buf, 131072) ◀── to TTY  (user presses G)
                                    lseek(3, -65536, SEEK_END)  ◀◀◀ THE MAGIC
Result: all 4 GB pushed through     read(3, buf, 65536); write(1, ...)
the TTY. Minutes. Can't pause.      Result: ~128 KB read total. Instant.
```

**`less` never loads the file into memory** — it keeps a window and `lseek()`s around. That's why it opens a 40 GB file instantly and `vim` does not.

---

## How It Works — Step by Step

### Trace: `cat /etc/hostname`

```
COMMAND:     cat /etc/hostname
SHELL DOES:  fork() → child; execve("/usr/bin/cat", ["cat","/etc/hostname"])
KERNEL DOES: path → dentry → inode; checks r permission vs your UID; caches blocks
SYSCALLS:    openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
             fstat(3, {st_size=11, ...})
             read(3, "prod-api-01\n", 131072) = 12
             write(1, "prod-api-01\n", 12)    = 12
             read(3, "", 131072) = 0    ◀── 0 = EOF, the stop signal
             close(3)
OUTPUT:      prod-api-01
```

### Trace: `tail -f /var/log/app.log` — how "following" actually works

This is the one you must be able to draw on a whiteboard.

```
Step 1: openat("/var/log/app.log", O_RDONLY) = 3
        fd 3 is bound to an INODE (say 2097241) — NOT to the NAME. The name was
        used once, at open() time, to find the inode. (Topic 04 — Inodes.)

Step 2: fstat(3) → 8,000 bytes.  lseek(3, -N, SEEK_END) to reach the last 10 lines.
        read() + write() them to your screen.

Step 3: Now FOLLOW. Modern GNU tail uses inotify:
            inotify_add_watch(fd, "/var/log/app.log", IN_MODIFY|IN_MOVE_SELF|IN_DELETE_SELF)
            read(inotify_fd, ...)   ← BLOCKS. Your process sleeps. 0% CPU.
        (Fallback on NFS / BusyBox / `--disable-inotify`: a poll loop — fstat(3)
         every 1.0s, and if st_size grew, read() the delta.)

Step 4: Your Node app write()s to its log fd → kernel appends bytes to inode
        2097241 and fires IN_MODIFY.

Step 5: tail wakes, read(3, ...) from its saved offset → ONLY the new bytes →
        write(1, ...) to your screen. Back to sleep.
```

### And now — the trap. What logrotate does at midnight:

```
BEFORE ROTATION                     AFTER `logrotate` RUNS
─────────────────────────           ───────────────────────────────────────────
app.log ──▶ inode 2097241           app.log.1 ─▶ inode 2097241  ← old data
               ▲                    app.log   ─▶ inode 2097999  ← NEW, empty ◀─┐
tail -f fd 3 ──┘                    tail -f fd 3 ─▶ inode 2097241 STILL         │
                                    Your app now writes to 2097999 ────────────┘
                                    You watch the WRONG file. Screen shows
                                    NOTHING. You think "app is quiet." It's
                                    throwing 500s into inode 2097999.
```

`mv` (rename) does **not** change the inode — it only rewrites the directory entry. Your open fd doesn't care; it holds the *inode*. Same mechanic as topic 04's "deleting a file a process has open doesn't free the disk space."

**`tail -F` fixes this.** `-F` = `--follow=name --retry`: it watches the NAME, sees the `IN_MOVE_SELF`/`IN_DELETE_SELF` inotify events, prints `tail: '/var/log/app.log' has appeared;  following new file`, then `close(3)` and re-`openat()`s the path — landing on inode 2097999, where your app is actually writing.

> **RULE: On a production server, always use `tail -F`, never `tail -f`.**
> There is essentially no situation where you want `-f`'s behavior and can't tolerate `-F`'s.

---

## Exact Syntax Breakdown

```
cat -n -s file1.txt file2.txt
│   │  │  │
│   │  │  └── one or more files — concatenated in order into ONE stream
│   │  └── --squeeze-blank — collapse runs of blank lines into a single blank line
│   └── --number — prefix every output line with a line number
└── cat = "conCATenate" — its ACTUAL job is joining files, not paging them
```

```
less -N -S -R +F /var/log/nginx/access.log
│    │  │  │  │  │
│    │  │  │  │  └── the file
│    │  │  │  └── +F — start in FOLLOW mode (a better `tail -f`; Ctrl-C to stop, F to resume)
│    │  │  └── -R — pass ANSI color escape codes through RAW (colored logs render as color)
│    │  └── -S — chop long lines instead of wrapping (scroll right with → ) — ESSENTIAL for logs
│    └── -N — show line numbers in the left gutter
└── less — a pager that lseek()s. Does NOT load the file into RAM.
```

```
tail -F -n 100 /var/log/app.log
│    │  │  │   │
│    │  │  │   └── the file
│    │  │  └── 100 lines (default is 10)
│    │  └── --lines
│    └── -F  = --follow=name --retry  ◀── survives logrotate. USE THIS ONE.
│        -f  = --follow=descriptor    ◀── dies silently on logrotate. The trap.
└── tail — print the END of a file
```

```
head -n -5 file.txt          tail -n +5 file.txt
│    │  │                    │    │  │
│    │  └── NEGATIVE n:      │    │  └── PLUS n: start FROM line 5 (skip the
│    │      all lines EXCEPT │    │      first 4). Classic use: strip a CSV header.
│    │      the last 5       │    └── --lines
│    └── --lines             └── tail — print the END
└── head — print the START
```

```
wc -l < access.log
│  │  │
│  │  └── REDIRECT, not an argument. Makes wc read stdin, so it prints ONLY the
│  │      number — no filename column. (`wc -l access.log` → "5231 access.log")
│  └── --lines — counts NEWLINE characters
└── wc = "word count" — lines (-l), words (-w), bytes (-c), chars (-m)
```

```
zgrep -c "ERROR" /var/log/app.log.3.gz
│     │  │        │
│     │  │        └── a GZIPPED rotated log — never decompressed to disk
│     │  └── the pattern
│     └── -c — count matching lines (all normal grep flags work)
└── zgrep — decompresses the gzip stream IN MEMORY, pipes it to grep.
            Zero disk writes. Safe on a full disk. (z-family: zcat, zless, zdiff)
```

---

## Example 1 — Basic

```bash
# Make a small test file (seq prints numbers 1..100, one per line)
seq 100 > /tmp/nums.txt

# ── cat: dump the WHOLE thing ────────────────────────────────
cat /tmp/nums.txt          # all 100 lines scroll past. Fine — it's small.
cat -n /etc/hostname       # number the lines:  "     1  prod-api-01"

# cat's REAL job — concatenating multiple files into one stream:
cat part1.txt part2.txt part3.txt > whole.txt

# cat reveals invisible characters:
printf 'hello\tworld \n' | cat -A
# hello^Iworld $
#      ^^      ^── $ marks END OF LINE, proving a trailing space hides before it
#      └── ^I is a TAB character

# ── head / tail: just the edges ──────────────────────────────
head -n 3   /tmp/nums.txt  # 1 2 3
tail -n 3   /tmp/nums.txt  # 98 99 100
head -c 20  /tmp/nums.txt  # first 20 BYTES (not lines)
head -n -95 /tmp/nums.txt  # everything EXCEPT the last 95 lines → 1..5
tail -n +98 /tmp/nums.txt  # start FROM line 98 → 98 99 100

# ── wc: count ────────────────────────────────────────────────
wc -l /tmp/nums.txt        # 100 /tmp/nums.txt
wc    /tmp/nums.txt        # 100  100  292 /tmp/nums.txt
#                            │    │    └── bytes
#                            │    └── words
#                            └── lines
ls /etc | wc -l            # the classic "how many X?" idiom

# ── less: the workhorse ──────────────────────────────────────
less /tmp/nums.txt
#   SPACE/b → page fwd/back   g/G → first/last line (G is instant even on 40 GB)
#   /error  → search forward  ?error → search backward   n/N → next/prev match
#   F       → follow (like tail -f); Ctrl-C stops it AND YOU CAN SCROLL BACK
#   q       → quit
# (Full key list in the Quick Reference Card below.)
```

---

## Example 2 — Production Scenario

**Situation:** 02:14. PagerDuty fires. Your Node API on `prod-api-01` is returning 502s through nginx. You SSH in. It is 4 hours past midnight — **logrotate has already run**, which is going to matter twice.

```bash
# ── Step 1: Is the app even alive? ───────────────────────────
$ pm2 list
┌────┬──────────────┬─────────┬─────────┬──────────┬────────┬──────────┐
│ id │ name         │ mode    │ ↺       │ status   │ cpu    │ memory   │
├────┼──────────────┼─────────┼─────────┼──────────┼────────┼──────────┤
│ 0  │ api          │ fork    │ 47      │ online   │ 0%     │ 61.2mb   │
└────┴──────────────┴─────────┴─────────┴──────────┴────────┴──────────┘
#                               ↑
#                               47 RESTARTS. It's crash-looping.

# ── Step 2: Watch the log live. USE -F, NOT -f. ──────────────
$ tail -F /var/log/app/api.log
2026-07-12T02:14:31.882Z ERROR unhandledRejection: connect ETIMEDOUT 10.0.3.14:5432
2026-07-12T02:14:31.883Z ERROR   at Socket.<anonymous> (/srv/api/node_modules/pg/lib/connection.js:74:12)
2026-07-12T02:14:31.883Z ERROR   at Object.onceWrapper (node:events:632:26)
2026-07-12T02:14:33.101Z INFO  server listening on :3000
2026-07-12T02:14:41.220Z ERROR unhandledRejection: connect ETIMEDOUT 10.0.3.14:5432
^C
# Postgres at 10.0.3.14 is unreachable. Every ~10s the app dies and pm2 restarts it.
#
# NOTE: had you typed `tail -f` here, and had logrotate fired while you watched,
# your screen would have gone silent and you'd have thought the crashes stopped.

# ── Step 3: When did this START? Not in the current file — it rotated at 00:00.
$ ls -lh /var/log/app/
-rw-r--r-- 1 deploy deploy   2.1M Jul 12 02:14 api.log
-rw-r--r-- 1 deploy deploy   180M Jul 12 00:00 api.log.1
-rw-r--r-- 1 deploy deploy   9.4M Jul 11 00:00 api.log.2.gz     ◀── compressed
-rw-r--r-- 1 deploy deploy   9.1M Jul 10 00:00 api.log.3.gz
# DO NOT `cat api.log.1` — 180 MB will flood your SSH session.
# DO NOT `gunzip api.log.2.gz` — you don't need to, and the disk may be full.

# -m1 = stop at the FIRST match. grep doesn't scan the rest of the file.
$ grep -m1 ETIMEDOUT /var/log/app/api.log            # today's file
2026-07-12T02:14:31.882Z ERROR unhandledRejection: connect ETIMEDOUT 10.0.3.14:5432

$ zgrep -m1 ETIMEDOUT /var/log/app/api.log.2.gz      # 2 days ago — read IN MEMORY
# (no output — clean)

$ grep -m1 ETIMEDOUT /var/log/app/api.log.1          # the file spanning the incident
2026-07-12T01:58:04.512Z ERROR unhandledRejection: connect ETIMEDOUT 10.0.3.14:5432
#                    ↑ 01:58 — that's when it began.

# ── Step 4: How bad? Count, don't read. ──────────────────────
$ grep -c ETIMEDOUT /var/log/app/api.log.1
14028
$ zgrep -c ETIMEDOUT /var/log/app/api.log.3.gz
0
# 14,028 timeouts since 01:58. Zero two days ago. This is new.

# ── Step 5: Read the FULL stack trace with a pager (never load 180 MB). ──
$ less -N -S /var/log/app/api.log.1
#   G → jump to EOF INSTANTLY (less lseek()s; never reads the middle 179 MB)
#   /01:58:04 → find the first bad timestamp;  n → next match;  q → quit

# ── Step 6: Confirm the DB is the actual cause ───────────────
$ journalctl -u postgresql -n 20 --no-pager
Jul 12 01:57:58 db-01 systemd[1]: Stopping PostgreSQL RDBMS...
# ← Postgres was restarted at 01:57:58 and never came back.
#   Your Node app is a victim, not the culprit.
```

**What you just did:** You never once loaded a 180 MB file into RAM. You used `tail -F` so rotation couldn't lie to you, `grep -m1` to find the first occurrence without scanning the whole file, `zgrep` to search a compressed archive with zero disk writes on a box you didn't want to fill, and `less` to seek to the end of a huge file instantly.

---

## Common Mistakes

### Mistake 1 — `cat`ing a huge log on a production box

**Wrong:**
```bash
cat /var/log/nginx/access.log     # 4 GB
```

**What actually happens at the kernel level:**
`cat` sits in a tight `read(3, buf, 131072)` / `write(1, buf, 131072)` loop. Every one of those 4 billion bytes goes through `write()` to **fd 1 — which is your TTY**, not a file. The kernel's TTY layer, your SSH connection, and your terminal emulator must each process, transmit, and *render* every line. Your terminal emulator becomes the bottleneck and pins a CPU core on your laptop, while all 4 GB crosses the network. `cat` stops only at EOF — it does not care that you've looked away.

Ctrl-C sends `SIGINT` to the foreground process group and *will* kill `cat` — but megabytes are already buffered in the TTY and in the SSH connection, so output keeps vomiting for several more seconds. It **feels** broken. (`Ctrl-S` freezes terminal output; if you don't know `Ctrl-Q` unfreezes it, you'll think you killed your session.)

**Right:**
```bash
less /var/log/nginx/access.log              # reads ~64 KB, waits
tail -n 200 /var/log/nginx/access.log
grep -c ' 500 ' /var/log/nginx/access.log   # count, don't display
```

**The same mistake wearing a different hat:** `vim /var/log/app/api.log.1` (180 MB). An **editor must load the file into memory** — it needs arbitrary insertion, undo history, syntax highlighting. `vim` `read()`s the whole thing into a buffer plus a swap file: tens of seconds and possibly a gigabyte of RSS. On an already-degraded box this can trigger the **OOM killer** (topic 01) — which picks the biggest memory hog, now *you*. `less` on a 40 GB file opens in ~5 ms using ~2 MB, because it `lseek()`s. **You never *edit* a log.** Reading tools for reading; editors for editing.

**Prevention:** Before viewing any file, ask how big it is (`ls -lh file`). Over a few hundred KB → `less`/`grep`/`tail`, never `cat`, never `vim`.

---

### Mistake 2 — `tail -f` across a logrotate (THE silent killer)

**Wrong:**
```bash
tail -f /var/log/app/api.log
# ...you leave this open during an incident. Midnight passes. Silence.
# "Great, the errors stopped."  They did not.
```

**Root cause (kernel level):** `open()` resolves the path to an **inode** exactly once and returns an fd bound to that inode. `logrotate` calls `rename("api.log", "api.log.1")` — which only rewrites a **directory entry**; the inode number is unchanged (topic 04). It then creates a *brand new* `api.log` with a *new inode*. Your `tail -f`'s fd still points at the old inode, which is now `api.log.1` and which nothing will ever write to again. `tail -f` is now, correctly and permanently, following a dead file.

**Diagnose it live:**
```bash
# Which inode is my tail actually holding?
ls -i /var/log/app/api.log            # 2097999  ← the file on disk NOW
lsof -p $(pgrep -f 'tail -f') | grep api    # shows NODE (inode) 2097241 ← the OLD one
# Numbers differ → you are following a ghost.
```

**Right:**
```bash
tail -F /var/log/app/api.log
# tail: '/var/log/app/api.log' has become inaccessible: No such file or directory
# tail: '/var/log/app/api.log' has appeared;  following new file
# ← it TELLS you, reopens by name, and keeps working.
```

**Prevention:** Build the muscle memory: **capital F on servers.** (Same for `less +F`, which also handles rotation gracefully.) Note that `logrotate` with the `copytruncate` directive behaves differently — it copies then `truncate()`s the *same* inode, so `-f` survives, but you can lose lines written during the copy. You cannot rely on knowing which mode is configured. Just use `-F`.

---

### Mistake 3 — Useless Use of Cat

**Wrong:**
```bash
cat app.log | grep ERROR
cat app.log | wc -l
```

**Why it's wrong:** You spawned an extra process (`fork` + `execve` + a pipe with two more fds) purely to hand a file to a command **that already knows how to open files itself**. `grep` will `open()` the file directly. Worse: given a filename, `grep` knows the file's size and can `mmap()`/`lseek()` it; reading from a pipe it cannot, and you lose grep's filename prefix in multi-file mode.

**Right:** `grep ERROR app.log` — and `wc -l < app.log`.

**When `cat` IS the right answer — do not over-correct:**
```bash
cat a.log b.log c.log | grep ERROR         # genuinely concatenating MULTIPLE files ✅
cat file | tr 'a-z' 'A-Z'                  # tr ONLY reads stdin — takes no filename ✅
cat -A weird.txt                           # inspecting non-printing characters ✅
cat <<'EOF' > /etc/nginx/conf.d/api.conf   # heredoc — writing a file inline ✅
server { listen 80; location / { proxy_pass http://127.0.0.1:3000; } }
EOF
```

---

### Mistake 4 — Decompressing rotated logs to search them

**Wrong:**
```bash
gunzip /var/log/app/api.log.5.gz    # 9 MB gz → 400 MB plain text
grep ERROR /var/log/app/api.log.5
```

**Root cause:** Logs compress ~40:1. You just wrote 400 MB to a disk that is probably the reason you're here. And `gunzip` **removes the `.gz` and replaces it with the decompressed file** by default — you've now destroyed the archive, and the next logrotate run will be confused.

**Right:**
```bash
zgrep ERROR /var/log/app/api.log.5.gz         # decompresses to a PIPE, in memory
zcat /var/log/app/api.log.5.gz | wc -l        # zero bytes hit the disk
zless /var/log/app/api.log.5.gz               # page through it compressed
zgrep ERROR /var/log/app/api.log.*.gz         # search ALL rotated logs at once
```

**Prevention:** There is a `z`-prefixed version of nearly everything: `zcat`, `zgrep`, `zless`, `zdiff`, `zmore`. For `.bz2` use `bzgrep`/`bzcat`; for `.xz` use `xzgrep`/`xzcat`. (More in topic 25 — Compression & Archiving.)

---

### Mistake 5 — Trusting your eyes about invisible characters

**Wrong:** "`config.json` looks fine but `JSON.parse` throws `Unexpected token ﻿ in JSON at position 0`." / "My deploy script fails with `line 2: $'\r': command not found` but looks perfect."

**Root cause:** Your editor hides bytes from you. A Windows-authored file has `\r\n` (CRLF) line endings — bash treats the trailing `\r` as part of the command name. Some Windows editors prepend a **UTF-8 BOM** (`EF BB BF`) — an invisible 3-byte prefix `JSON.parse` chokes on.

**Diagnose — four tools that see BYTES, not characters:**
```bash
$ file deploy.sh
deploy.sh: Bourne-Again shell script, ASCII text, with CRLF line terminators  ◀── there it is

$ cat -A deploy.sh | head -2
#!/bin/bash^M$        ◀── ^M is the carriage return. $ marks end-of-line.
npm ci^M$

$ od -c deploy.sh | head -1
0000000   #   !   /   b   i   n   /   b   a   s   h  \r  \n   n   p   m
#                                                     ^^ proof, in one glance

$ xxd config.json | head -1
00000000: efbb bf7b 0a20 2022 706f 7274 223a 2033  ...{.  "port": 3
#          ^^^^^^^^ UTF-8 BOM — three invisible bytes sitting before the '{'
```

**Fix:** `tr -d '\r' < deploy.sh > fixed && mv fixed deploy.sh` (or `dos2unix deploy.sh`). Strip a BOM with `sed -i '1s/^\xEF\xBB\xBF//' config.json`. (Topic 11 covers `tr` and `sed` properly.)

**Prevention:** Add `* text=auto eol=lf` to `.gitattributes`. When any file "looks fine but isn't," reach for `file`/`cat -A`/`xxd` immediately.

---

## Hands-On Proof

```bash
# PROVE IT: less lseek()s instead of loading the file. cat does not.
truncate -s 1G /tmp/big.txt       # 1 GB sparse file — instant, uses no disk
strace -e trace=openat,read,lseek -o /tmp/l.trace less /tmp/big.txt   # press G, then q
grep -c '^read'  /tmp/l.trace     # a handful of reads — NOT 1 GB worth
grep    'lseek'  /tmp/l.trace | head   # lseek() jumping straight to EOF = the whole trick
strace -c cat /etc/services > /dev/null
# cat's summary table: many read()+write() calls, and ZERO lseek().

# PROVE IT: tail -f uses inotify — it SLEEPS, it does not busy-poll.
tail -f /var/log/syslog &
strace -p $! 2>&1 | head -3       # BLOCKED in read() on an inotify fd — 0% CPU
kill %1

# PROVE IT: tail -f follows the INODE, not the name. (The logrotate trap, live.)
echo "line one" > /tmp/rot.log
tail -f /tmp/rot.log &                 # start following
sleep 1
mv /tmp/rot.log /tmp/rot.log.1         # logrotate: rename → inode UNCHANGED
echo "NEW FILE LINE" > /tmp/rot.log    # app writes to a NEW inode
sleep 2
# ← Screen shows NOTHING. "NEW FILE LINE" never appeared. THAT is the bug.
kill %1
ls -i /tmp/rot.log /tmp/rot.log.1      # two DIFFERENT inode numbers — the proof

# Now the identical sequence with -F:
echo "line one" > /tmp/rot.log
tail -F /tmp/rot.log &
sleep 1; mv /tmp/rot.log /tmp/rot.log.1; echo "NEW FILE LINE" > /tmp/rot.log; sleep 2
# tail: '/tmp/rot.log' has appeared;  following new file
# NEW FILE LINE       ◀── it works.
kill %1

# PROVE IT: zgrep never writes to disk.
gzip -c /var/log/syslog > /tmp/s.gz
df -h /tmp | tail -1              # note "Used"
zgrep -c . /tmp/s.gz              # counts every line of the DECOMPRESSED content
df -h /tmp | tail -1              # "Used" UNCHANGED — nothing was written to disk

# PROVE IT: -c is BYTES, -n is LINES; and wc -c ≠ wc -m for UTF-8.
printf 'abcdefghij\nklmno\n' > /tmp/x
head -c 3 /tmp/x                  # abc          ← 3 BYTES
head -n 1 /tmp/x                  # abcdefghij   ← 1 LINE
printf 'héllo' > /tmp/u
wc -c /tmp/u                      # 6  ← é is TWO bytes in UTF-8
wc -m /tmp/u                      # 5  ← but only ONE character

# PROVE IT: strings pulls readable text out of a binary.
strings /bin/ls | head -5
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
# Generate a 500-line file where every 7th line contains "ERROR":
seq 500 | awk '{print ($1 % 7 == 0) ? "line "$1" ERROR failed" : "line "$1" INFO ok"}' > /tmp/ex1.log
```

In the terminal, produce each of the following. **Use no `cat`.**
1. The first 5 lines.
2. The last 5 lines.
3. Everything **except** the last 490 lines.
4. Everything **from** line 495 onward.
5. The total number of lines, printed as a bare number (no filename).
6. The total number of lines containing `ERROR`.
7. Open it in `less`, jump to the last line, search **backwards** for `ERROR`, then quit.

Paste your terminal output.

---

### Exercise 2 — Medium

Reproduce the logrotate trap and prove it with inode numbers. Three terminals:

```bash
# Terminal 1 — a fake "app" appending a line every second:
while true; do echo "$(date +%T) heartbeat" >> /tmp/app.log; sleep 1; done

# Terminal 2 — note the LOWERCASE -f:
tail -f /tmp/app.log

# Terminal 3 — you are logrotate:
ls -i /tmp/app.log                     # record the inode number
mv /tmp/app.log /tmp/app.log.1
touch /tmp/app.log
ls -i /tmp/app.log                     # record the NEW inode number
```

**Answer with commands, not prose:**
1. What happens to Terminal 2's output after the `mv`? Why?
2. Use `lsof` to show which inode Terminal 2's `tail` is *actually* holding open, and prove it differs from the inode of the file now named `/tmp/app.log`.
3. Redo it all with `tail -F`. Paste the message tail prints when it recovers.

---

### Exercise 3 — Hard (Production Simulation)

You are on-call. A customer reports that request ID `req-8f3a91c2` returned a 500 at roughly 23:50 last night — which means the evidence spans **two log files**: last night's (now rotated *and gzipped*) and today's.

```bash
# Build the scenario — LAST NIGHT's log, which then gets rotated + gzipped:
mkdir -p /tmp/logs
seq 20000 | awk '{printf "2026-07-11T23:%02d:%02d INFO req-%08x ok\n", int(rand()*60), int(rand()*60), $1*7919}' > /tmp/logs/api.log.1
cat >> /tmp/logs/api.log.1 <<'EOF'
2026-07-11T23:50:04 ERROR req-8f3a91c2 TypeError: Cannot read properties of undefined (reading "id")
2026-07-11T23:50:04 ERROR req-8f3a91c2     at handler (/srv/api/routes/order.js:44:19)
2026-07-11T23:50:04 ERROR req-8f3a91c2     at Layer.handle (/srv/api/node_modules/express/lib/router/layer.js:95:5)
EOF
gzip /tmp/logs/api.log.1        # → api.log.1.gz

# TODAY's log:
seq 5000 | awk '{printf "2026-07-12T00:%02d:%02d INFO req-%08x ok\n", int(rand()*60), int(rand()*60), $1*104729}' > /tmp/logs/api.log
echo '2026-07-12T00:03:11 ERROR req-8f3a91c2 retry failed, giving up' >> /tmp/logs/api.log
```

**Do all of this without ever decompressing a file to disk, and without `cat`ing a whole log:**

1. Find every line mentioning `req-8f3a91c2` across **both** logs, with the filename shown on each hit.
2. Print the 3-line stack trace **with 2 lines of surrounding context** from the gzipped file.
3. How many total lines does the gzipped file contain?
4. How many `ERROR` lines are in each of the two files?
5. Is `/tmp/logs/api.log` small enough to `cat` safely? Prove it with a command *before* you decide.
6. Open the gzipped file in a pager, jump to the end, search backwards for the request ID.

Paste every command and its output.

---

## Mental Model Checkpoint

1. **`less` opens a 40 GB file instantly; `vim` takes a minute and 8 GB of RAM. What is the one syscall that explains the difference?**
2. **You run `tail -f /var/log/app.log`. logrotate renames the file to `app.log.1` and creates a fresh `app.log`. Draw what your file descriptor is now pointing at, and explain — in terms of inodes and directory entries — why you stop seeing output. What flag fixes it?**
3. **Why is `cat file | grep x` worse than `grep x file`? Name two concrete costs.**
4. **Name three situations where `cat` genuinely IS the right tool.**
5. **What actually happens to your terminal, your SSH connection, and the TTY layer when you `cat` a 4 GB log — and why does Ctrl-C feel like it doesn't work?**
6. **`less +F` does the same thing as `tail -f`. Why would you ever prefer it?**
7. **Your bash script fails with `line 2: $'\r': command not found`. What is physically wrong with the bytes in the file, which two commands prove it, and how do you fix it?**
8. **What does `tail -n +5 file` do, and how is it different from `tail -n 5 file`?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---------|-------------|-----------|
| `cat` | Concatenate files to stdout | `-n` number lines, `-A` show non-printing chars (`^I`=tab, `$`=EOL, `^M`=CR), `-s` squeeze blank lines, `-b` number non-blank only |
| `less` | Pager — **seeks**, never loads whole file | `-N` line numbers, `-S` chop long lines, `-R` render ANSI color, `-F` quit if fits one screen, `+F` start in follow mode, `+G` start at end |
| `less` keys | Navigation inside less | `SPACE`/`b` page fwd/back, `d`/`u` half-page, `g`/`G` first/last line, `/pat` search fwd, `?pat` search back, `n`/`N` next/prev match, `F` follow, `Ctrl-C` stop following, `q` quit |
| `more` | Ancient pager — forward only | Use `less` instead. "less is more than more" |
| `head` | First N lines/bytes | `-n 20`, `-c 100` (bytes), `-n -5` (all BUT last 5) |
| `tail` | Last N lines/bytes | `-n 20`, `-c 100`, `-n +5` (start FROM line 5) |
| `tail -f` | Follow the file **descriptor/inode** | **DIES SILENTLY ON LOGROTATE** |
| `tail -F` | Follow the **name**, reopen on rotate | `= --follow=name --retry`. **Use this on servers.** |
| `tail -f a.log b.log` | Follow multiple files | Prints `==> a.log <==` headers when the source switches |
| `wc` | Count | `-l` lines, `-w` words, `-c` bytes, `-m` chars. Use `wc -l < f` for a bare number |
| `zcat` / `zless` / `zgrep` | Read `.gz` **without decompressing to disk** | All normal flags work: `zgrep -c ERROR app.log.3.gz` |
| `bzcat`/`bzgrep`, `xzcat`/`xzgrep` | Same, for `.bz2` and `.xz` | — |
| `watch` | Re-run a command every N seconds | `-n 1` interval, `-d` highlight differences |
| `strings` | Extract printable text from a binary/core dump | `-n 10` min length |
| `xxd` | Hex dump | `xxd file \| head`. Reveals BOMs, CRLF, null bytes |
| `od` | Octal/char dump | `od -c` (chars — shows `\r`, `\n`), `od -x` (hex) |
| `file` | Identify a file's type by its magic bytes | Reports "with CRLF line terminators" |
| `journalctl -f` | systemd's `tail -F` for service logs | `-u api -f -n 100`. See topic 23 |

---

## When Would I Use This at Work?

### Scenario 1: Watching a deploy go out

You push a release and want to watch your Node app come up and bind port 3000 before you tell the channel it's live.

```bash
less +F /var/log/app/api.log
#    ││
#    │└── F = start in follow mode
#    └── Ctrl-C stops following → SCROLL BACK to read the stack trace that just flew past
#        → press F to resume. `tail -f` CANNOT do this. That's why less +F wins.

# systemd service instead of a log file? (topic 22)
journalctl -u api -f -n 50
```

### Scenario 2: Grepping a stack trace out of a rotated `.gz`

A customer reports an error from three days ago — that log is `api.log.3.gz`. The box has 800 MB free; decompressed, the log is 400 MB. So you never decompress it.

```bash
zgrep -B2 -A8 'TypeError' /var/log/app/api.log.3.gz | less -S
#      │   └── 8 lines AFTER the match → the full stack trace
#      └── 2 lines BEFORE the match    → the request that caused it
```
Zero bytes written to disk. (`-A`/`-B`/`-C` are covered properly in topic 11.)

### Scenario 3: Tracing one request ID across the rotation boundary

An incident straddled midnight, so the evidence is split across today's plain log and last night's gzipped one. `zgrep` reads **both** — it passes plain files straight through:

```bash
zgrep -H "req-8f3a91c2" /var/log/app/api.log /var/log/app/api.log.*.gz
#     └── -H = print the filename on every hit, so you know WHICH file (and when)
```

---

## Connected Topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Builds on** | 03 — Everything Is a File | Reading a log and reading `/proc/meminfo` are the same syscalls. That's the point of the abstraction |
| **Builds on** | 04 — Inodes in Depth | The `tail -f` vs `-F` logrotate trap is *entirely* an inode-vs-filename story. `rename()` moves the directory entry; the inode (and your fd) don't move |
| **Builds on** | 05 — File Permissions | `less /var/log/auth.log` → "Permission denied" is the kernel checking your UID against the inode's mode bits. That's a `sudo` job |
| **Builds on** | 09 — Working with Files | `ls -lh` / `du -h` — ALWAYS check a file's size before you decide how to view it |
| **Next** | 11 — Text Processing Powertools | You've found the file. Now `grep`/`sed`/`awk` extract meaning from it |
| **Used by** | 12 — Pipes and Redirection | `head`, `tail`, and `wc` are pipeline citizens: `... \| sort \| uniq -c \| sort -rn \| head -10` |
| **Used by** | 19 — File Descriptors | `lsof` is how you prove which inode your zombie `tail -f` is stuck on |
| **Used by** | 23 — Logs and Log Management | logrotate is the thing that sets the `-f`/`-F` trap. `journalctl -f` is systemd's answer to `tail -F` |
| **Used by** | 25 — Compression & Archiving | The `z`-family (`zcat`/`zgrep`/`zless`) is why you never `gunzip` a rotated log on a full disk |
