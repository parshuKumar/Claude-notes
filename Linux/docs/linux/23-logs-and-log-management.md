# 23 — Logs and Log Management

## ELI5 — The Simple Analogy

Imagine a hospital.

- **Every department keeps its own notebook.** Radiology scribbles in one book, the pharmacy in another, the front desk in a third. Nobody coordinates. That's **plain text files in `/var/log`** — each app writing its own file, its own format, whenever it feels like it.
- **Then someone installed an intercom system.** Any department can pick up the handset, say "this is PHARMACY, priority URGENT: we're out of insulin," and a **central clerk** writes it into the correct ledger based on who called and how urgent it was. That's **syslog** — the caller declares a *facility* (who am I) and a *severity* (how bad is it), and the clerk routes it.
- **Then the hospital went digital.** Every message is now a database row with typed fields: department, timestamp, doctor ID, patient ID, severity, message. You can *query* it. That's **systemd-journald** — a structured, indexed, binary journal.
- **And the janitor.** If nobody ever threw away old notebooks, the storage room would fill up and eventually the hospital would stop functioning. The janitor comes every night, moves last week's notebook to the archive, shrink-wraps it, and burns anything older than a month. That's **logrotate** — and if the janitor is missing, **the hospital shuts down**. Not metaphorically. A full disk takes your server down.

All three systems run *at the same time* on a modern Linux box. That is the single most confusing thing about logging, and it is why this doc exists.

---

## Where This Lives in the Linux Stack

```
Hardware (disk platters / SSD cells where the bytes actually land)
  └── KERNEL
       │    • printk() ring buffer  ◀◀◀ dmesg reads THIS (OOM kills live here)
       │    • /dev/kmsg exposes it to userspace
       │
       └── System Calls
            │    write(fd, "ERROR ...", n)   ← an app logging to a file
            │    sendto(unix_socket, msg)    ← an app logging to syslog/journald
            │    openat(), rename(), ftruncate()  ← what logrotate does
            │
            └── C Library (glibc: syslog(3), openlog(3))
                 │
                 └── LOG DAEMONS (userspace!)  ◀◀◀ THIS TOPIC
                      │    • rsyslogd  — listens on /dev/log, routes to files
                      │    • systemd-journald — listens on /run/systemd/journal/*
                      │    • logrotate — a cron/timer-driven janitor
                      │
                      └── Shell + Tools
                           tail -F, less +F, grep, awk, zgrep, journalctl, jq
```

**Key insight:** the log daemons are ordinary userspace processes. `rsyslogd` and `systemd-journald` show up in `ps aux` like anything else. They can die. They can be misconfigured. They can be filled up. There is no magic.

---

## What Is This?

A **log** is a line of text (or a structured record) that a program emits to describe something that happened. Linux gives you three overlapping ways to collect them: apps writing files directly, apps talking to the **syslog** daemon over a Unix socket, and apps writing to **journald** (which systemd wires up automatically for any service's stdout/stderr).

**Log management** is everything around that: reading logs efficiently, filtering them, and — critically — **rotating and deleting them so they don't fill your disk and take the server down.**

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Where logs actually live | You'll SSH in during an outage and stare at `/var/log` with no idea which of the 40 files matters |
| logrotate | A 40GB `app.log` fills `/`, Postgres can't write its WAL, nginx can't write its access log, and **the whole server dies** at 2am |
| The inode-level rotation problem | You'll "fix" the disk, but your app keeps writing to `app.log.1` forever and the new `app.log` stays 0 bytes — and you won't know why |
| `rm` vs `truncate -s 0` | You'll `rm` the huge log, `df` will still say 100% full, and you'll conclude the server is haunted |
| Docker's default log driver | One chatty container writes 200GB to `/var/lib/docker` and takes down **every other container on the host** |
| journalctl filtering | You'll `journalctl | grep` 4 million lines instead of `journalctl -u myapp -p err --since "10 min ago"` |
| Structured logging | Your logs are unparseable prose and you cannot answer "how many 5xx did we serve between 03:00 and 03:10" |
| "never log secrets" | A JWT lands in `/var/log/app.log`, gets shipped to your log vendor, and lives in their backups for 7 years |

---

## The Physical Reality

### The three systems, side by side

```
┌──────────────────────────────────────────────────────────────────────────┐
│  YOUR NODE APP  (pid 1247)                                               │
│                                                                          │
│   console.log("req done")   fs.appendFile("/var/log/app.log", ...)      │
│         │ fd 1 (stdout)              │ fd 5 (a real file)               │
└─────────┼────────────────────────────┼──────────────────────────────────┘
          │                            │
          │                            └──► write(5, ...) ──► ext4 ──► /var/log/app.log
          │                                                            (SYSTEM 1: plain file)
          │
          │  systemd started you, so fd 1 is a SOCKET to journald,
          │  not a terminal.  ss -x | grep journal proves it.
          ▼
   /run/systemd/journal/stdout  (AF_UNIX socket)
          │
          ▼
┌──────────────────────┐        ┌────────────────────────────────────────┐
│  systemd-journald    │        │  rsyslogd                              │
│  (SYSTEM 3)          │───────►│  (SYSTEM 2 — reads from journald       │
│                      │ forward│   via imjournal, or from /dev/log)     │
│  writes BINARY,      │        │                                        │
│  indexed records to  │        │  routes by FACILITY.SEVERITY into      │
│  /var/log/journal/   │        │  plain text: /var/log/syslog,          │
│    <machine-id>/     │        │  /var/log/auth.log, /var/log/kern.log  │
│      system.journal  │        └────────────────────────────────────────┘
└──────────────────────┘

  ┌──────────────────────────────────────┐
  │  KERNEL RING BUFFER (printk)         │  ← fixed-size circular buffer IN RAM
  │  OOM kills, disk errors, driver msgs │     Old messages are OVERWRITTEN.
  │  read it with:  dmesg                │     journald also copies it out.
  └──────────────────────────────────────┘
```

**The overlap is real and confusing.** On Ubuntu 22.04, a message from your systemd-managed Node app can end up in the journal **and** in `/var/log/syslog`, because rsyslog's `imjournal`/`imuxsock` module pulls from the journal and writes it back out as text. That is not a bug. That is the default.

### What a journal file actually is on disk

```bash
$ ls -la /var/log/journal/9f2c.../ 
-rw-r-----+ 1 root systemd-journal   8388608 Jul 12 14:02 system.journal
-rw-r-----+ 1 root systemd-journal 109051904 Jul 10 03:11 system@0005ff...journal
                                    ^^^^^^^^^ archived, rotated by journald ITSELF
```

Binary. `cat` gives you garbage. `strings` gives you fragments. You **must** use `journalctl`. (Topic 22 — systemd.)

---

## How It Works — Step by Step

### Trace: your Node app logs a line under systemd

```
COMMAND:    console.log("checkout failed for user 88")

NODE DOES:  process.stdout.write(str)  →  libuv  →  write(1, "checkout...\n", 28)

KERNEL:     fd 1 is NOT a file and NOT a tty. systemd replaced it with an
            AF_UNIX SOCKET connected to /run/systemd/journal/stdout before
            it exec()'d your process. So write() = sendmsg() to journald.

JOURNALD:   • receives the bytes
            • ATTACHES METADATA IT KNOWS FROM THE KERNEL (you cannot forge this):
                _PID=1247  _UID=1001  _COMM=node  _SYSTEMD_UNIT=myapp.service
                _BOOT_ID=...  _MACHINE_ID=...  __REALTIME_TIMESTAMP=...
            • appends a structured record to /var/log/journal/<id>/system.journal
            • updates the inline indexes (that's why journalctl -u is FAST)

RSYSLOG:    (if configured) reads the journal, formats it as a syslog line,
            appends text to /var/log/syslog

YOU:        journalctl -u myapp -f
            → journald mmap()s the journal files, seeks the _SYSTEMD_UNIT index,
              and streams matching records
```

### Trace: rsyslog routing by facility.severity

```
COMMAND:    logger -p auth.crit "intrusion detected"

logger DOES:   connect to AF_UNIX /dev/log, send "<34>Jul 12 14:02:01 host ..."
                                                   ^^^^ the PRI number
PRI MATH:      PRI = facility*8 + severity
               auth = facility 4,  crit = severity 2
               4*8 + 2 = 34    ← that's the <34>

RSYSLOGD:      parses <34>, decodes facility=auth severity=crit
               walks /etc/rsyslog.conf + /etc/rsyslog.d/*.conf rules top to bottom
               matches rule:  auth,authpriv.*   /var/log/auth.log
               open()s (or reuses fd for) /var/log/auth.log, write()s the line
```

---

## Syslog — Facilities and Severities

Syslog is a **protocol**, not a program. `rsyslogd` is the program (on Debian/Ubuntu). Every message carries two tags.

### Facilities — "who is speaking"

| # | Facility | Meaning |
|---|----------|---------|
| 0 | `kern` | Kernel messages |
| 1 | `user` | Generic user-level messages (the default for `logger`) |
| 2 | `mail` | Mail system |
| 3 | `daemon` | System daemons (**nginx, your app, most services**) |
| 4 | `auth` | Security/authorization (login, su) |
| 5 | `syslog` | Messages from the syslog daemon itself |
| 6 | `lpr` | Line printer |
| 7 | `news` | Usenet news (a fossil) |
| 8 | `uucp` | UUCP (a fossil) |
| 9 | `cron` | Cron daemon (**topic 20**) |
| 10 | `authpriv` | Security/auth — **private**, contains sensitive data → `/var/log/auth.log` mode 0640 |
| 11 | `ftp` | FTP daemon |
| 16–23 | `local0`–`local7` | **Free for you.** Use these for your own apps. |

### Severities — "how bad is it"

| # | Severity | Keyword | Meaning | Would you page someone? |
|---|----------|---------|---------|---|
| 0 | Emergency | `emerg` | System is unusable | Yes, everyone |
| 1 | Alert | `alert` | Act immediately | Yes |
| 2 | Critical | `crit` | Critical condition | Yes |
| 3 | Error | `err` | Error condition | Probably |
| 4 | Warning | `warning` | Warning condition | No, but look at it |
| 5 | Notice | `notice` | Normal but significant | No |
| 6 | Informational | `info` | Informational | No |
| 7 | Debug | `debug` | Debug-level | Never in prod |

**Lower number = more severe.** This trips everyone up. `-p err` in `journalctl` means "priority err **and more severe**" — i.e. 3, 2, 1, 0.

### Reading `/etc/rsyslog.conf` rules

```
auth,authpriv.*                 /var/log/auth.log
│    │        │                 │
│    │        │                 └── ACTION: append to this file
│    │        └── severity selector: * = all severities
│    └── second facility (comma = OR)
└── first facility

*.*;auth,authpriv.none          -/var/log/syslog
│ │ │                           │
│ │ │                           └── the leading "-" means: DON'T fsync after every
│ │ │                               line (async write — much faster, but you can
│ │ │                               lose the last few lines in a hard crash)
│ │ └── ...EXCEPT auth/authpriv (".none" = no severities) — so passwords/keys
│ │     don't get duplicated into world-readable syslog
│ └── all severities
└── all facilities

kern.*                          -/var/log/kern.log
mail.err                        /var/log/mail.err     ← err AND ABOVE (err,crit,alert,emerg)
local0.=debug                   /var/log/app-debug.log ← "=" means EXACTLY debug, nothing else
*.emerg                         :omusrmsg:*            ← write to every logged-in terminal
```

---

## Touring `/var/log`

```bash
$ ls -la /var/log
drwxr-xr-x  2 root      adm         nginx/
-rw-r-----  1 syslog    adm      124K auth.log        ◀◀ LOGINS, sudo, SSH  ← BREAK-IN ATTEMPTS
-rw-r-----  1 syslog    adm       12K auth.log.1
-rw-r-----  1 syslog    adm      2.1K auth.log.2.gz
-rw-r-----  1 syslog    adm      891K syslog          ◀◀ the firehose (everything)
-rw-r-----  1 syslog    adm       44K kern.log        ◀◀ kernel messages, persisted
-rw-rw-r--  1 root      utmp     292K wtmp            ◀◀ BINARY — use `last`, not cat
-rw-rw-r--  1 root      utmp        0 btmp            ◀◀ BINARY — FAILED logins, use `lastb`
-rw-r--r--  1 root      root     1.2M cloud-init.log  ◀◀ what your VM did on first boot
-rw-r--r--  1 root      root      33K dpkg.log        ◀◀ every package install (topic 21)
drwxr-sr-x+ 3 root      systemd-journal  journal/     ◀◀ the binary journal
```

| File | What it is | When you read it |
|---|---|---|
| `/var/log/syslog` (Debian/Ubuntu) / `/var/log/messages` (RHEL) | Catch-all, all facilities | "Something happened around 03:14, what was it?" |
| `/var/log/auth.log` (Debian) / `/var/log/secure` (RHEL) | **Logins, `sudo`, SSH, PAM** | **Break-in attempts. Who ran sudo. Failed SSH.** → topic 35 |
| `/var/log/kern.log` | Kernel messages, persisted to disk | Disk I/O errors, network driver flaps |
| `dmesg` (not a file — the kernel's in-RAM ring buffer) | Kernel ring buffer | **OOM kills.** `dmesg -T | grep -i "killed process"` |
| `/var/log/cloud-init.log` | Cloud VM first-boot provisioning | "Why didn't my user-data script run?" |
| `/var/log/nginx/access.log` | Every HTTP request | Traffic analysis, finding the 5xx spike |
| `/var/log/nginx/error.log` | nginx's own errors | **"502 Bad Gateway" → read this, it names the upstream** |
| `/var/log/dpkg.log`, `/var/log/apt/` | Package operations | "What got upgraded right before it broke?" |

**`dmesg` is special.** It is a **fixed-size circular buffer in RAM** (`printk`). Old messages are overwritten. It does not survive a reboot (unless journald persisted it, which it does — `journalctl -k -b -1`).

```bash
# PROVE IT: the OOM killer is recorded in the kernel ring buffer
sudo dmesg -T | grep -iE "out of memory|killed process"
# [Sat Jul 12 03:14:22 2026] Out of memory: Killed process 1247 (node)
#   total-vm:4194304kB, anon-rss:3900123kB, file-rss:0kB, oom_score_adj:0
#                                  ^^^^^^^^^^ your Node heap. Topic 01, topic 34.
```

---

## The nginx Access Log — Parse It Field by Field

nginx's default `combined` format. **You will `awk` this more than any other file in your career.**

```
203.0.113.7 - alice [12/Jul/2026:14:02:01 +0000] "POST /api/checkout HTTP/1.1" 500 154 "https://shop.example.com/cart" "Mozilla/5.0 (Macintosh...)"
│           │ │     │                            │                             │   │   │                                │
│           │ │     │                            │                             │   │   │                                └── $http_user_agent  ($9 in awk)
│           │ │     │                            │                             │   │   └── $http_referer               ($8)
│           │ │     │                            │                             │   └── $body_bytes_sent — bytes of BODY only, no headers  ($7)
│           │ │     │                            │                             └── $status — the HTTP code  ($6)  ◀◀ the money field
│           │ │     │                            └── "$request" = METHOD URI PROTO — ONE quoted field, but THREE awk fields ($5 $6 $7 if naive!)
│           │ │     └── $time_local — nginx's local time + tz offset  (note: [ and ] are part of the field)
│           │ └── $remote_user — from HTTP Basic auth; "-" if none
│           └── $remote_addr's ident field — always "-" in practice (RFC 1413, dead since 1993)
└── $remote_addr — client IP. ◀◀ IF NGINX IS BEHIND A LOAD BALANCER THIS IS THE LB's IP,
                                 not the user's. You want $http_x_forwarded_for.
```

**The awk trap:** the quoted `"$request"` contains spaces, so with default whitespace splitting the fields are:

```
$1=203.0.113.7  $2=-  $3=alice  $4=[12/Jul/2026:14:02:01  $5=+0000]
$6="POST  $7=/api/checkout  $8=HTTP/1.1"  $9=500  $10=154  ...
                                            ^^^ status is $9, NOT $6
```

```bash
# Top 10 URLs by request count  (topic 11 — awk/sort/uniq)
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Count responses by status code
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
#  184213 200
#    9821 304
#     412 404
#      88 500     ◀◀ there's your problem

# Which IPs are hammering you? (top talkers)
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -5

# Every 5xx, with the URL and the client
awk '$9 >= 500 {print $1, $7, $9}' /var/log/nginx/access.log

# 5xx per minute — is it a spike or a steady bleed?
awk '$9 >= 500 {print substr($4, 2, 17)}' /var/log/nginx/access.log | uniq -c
#   3 12/Jul/2026:14:01
#  47 12/Jul/2026:14:02   ◀◀ it started at 14:02
#  52 12/Jul/2026:14:03
```

---

## Reading Logs Effectively — Applying Topics 10, 11, 12

| Need | Command | Why |
|---|---|---|
| Watch a file live | `tail -F /var/log/nginx/error.log` | **`-F`, not `-f`.** `-f` follows the *fd*; if logrotate renames the file, `-f` keeps watching the old inode and goes silent forever. `-F` follows the *name* and re-opens. (Topic 04 — inodes.) |
| Watch + search + scroll | `less +F /var/log/syslog` | `Ctrl-C` to stop following and scroll/search with `/`, `F` to resume following. Best of both. |
| Context around a stack trace | `grep -C 20 "TypeError" app.log` | A Node stack trace is 15 lines *after* the error and the request context is a few lines *before*. `-B`/`-A`/`-C`. |
| Search rotated logs | `zgrep "checkout failed" /var/log/app.log.*.gz` | `zgrep` decompresses on the fly. Also `zcat`, `zless`. (Topic 25.) |
| Search *everything* incl. rotated | `zgrep -h ERROR /var/log/app.log*` | glob catches `app.log`, `app.log.1`, `app.log.2.gz` — zgrep handles both plain and gz. |
| Count occurrences | `grep -c "ECONNREFUSED" app.log` | |
| Extract + count fields | `awk '{print $9}' access.log \| sort \| uniq -c \| sort -rn` | The universal "top N" pipeline. |
| Follow a systemd service | `journalctl -u myapp -f` | The modern `tail -f`. |

---

## journalctl — The Log-Analysis Angle

Topic 22 taught you `journalctl -u myapp`. Here is what makes it a genuine **analysis** tool.

```bash
journalctl -u myapp.service -p err --since "2026-07-12 03:00" --until "03:30" -o short-iso
│          │                │      │                          │              │
│          │                │      │                          │              └── timestamp format: short-iso, json,
│          │                │      │                          │                  json-pretty, cat (message only), verbose
│          │                │      │                          └── end of window
│          │                │      └── start of window. Also accepts: "10 min ago", "yesterday",
│          │                │          "-1h", "today". THIS IS THE BIG ONE.
│          │                └── priority: err AND MORE SEVERE (3,2,1,0). Also -p warning..err (range).
│          └── filter to one systemd unit (uses the _SYSTEMD_UNIT index — fast)
└── the journal reader
```

| Flag | Does |
|---|---|
| `-u nginx.service` | One unit. Repeat `-u` for **multiple units in one timeline.** |
| `-f` | Follow (live tail) |
| `-n 200` | Last 200 lines |
| `-b` / `-b -1` | This boot / the **previous** boot (did it reboot? what did it say before it died?) |
| `-k` | Kernel messages only (= `dmesg`, but persisted across reboots) |
| `-p err` | Priority filter (uses the syslog severity numbers above) |
| `--since` / `--until` | Time window — accepts human strings |
| `--grep 'ECONNREFUSED'` | Regex match on MESSAGE (server-side, faster than piping to grep) |
| `-o json` | One JSON object per line — **pipe to `jq`** |
| `-o cat` | Message text only, no timestamps/prefix — for feeding to awk |
| `_PID=1247` / `_UID=1001` | Match on any trusted metadata field |
| `--disk-usage` | How big is the journal? |
| `--vacuum-size=500M` / `--vacuum-time=7d` | **Shrink it now.** Useful when the disk is full. |

### The move: correlate across units in ONE timeline

This is the single best debugging trick in this doc.

```bash
journalctl -u nginx -u myapp --since "10 min ago" -o short-iso
```

```
2026-07-12T14:02:01+0000 web-1 myapp[1247]: POST /api/checkout start req=a3f9
2026-07-12T14:02:01+0000 web-1 myapp[1247]: db query timeout after 30000ms req=a3f9
2026-07-12T14:02:31+0000 web-1 myapp[1247]: 500 POST /api/checkout req=a3f9
2026-07-12T14:02:31+0000 web-1 nginx[901]: upstream timed out (110: Connection timed out)
                                            while reading response header from upstream,
                                            client: 203.0.113.7, upstream: "http://127.0.0.1:3000/api/checkout"
```

**You just proved the 502 was the app's DB timeout, not nginx**, by seeing the proxy and the app interleaved on one clock. Doing this with two `tail -f` windows is guesswork; doing it with `journalctl -u A -u B` is evidence.

### Structured logs: `-o json` + `jq`

```bash
# All errors from myapp as JSON, extract just what you need
journalctl -u myapp -p err -o json --since today \
  | jq -r '[.__REALTIME_TIMESTAMP, .MESSAGE] | @tsv'

# If your app emits JSON to stdout, journald stores it as an opaque MESSAGE string.
# Parse it a second time:
journalctl -u myapp -o cat --since "1 hour ago" \
  | jq -c 'select(.level >= 50)'          # pino: 50 = error, 60 = fatal
```

---

## logrotate — In Real Depth

**An unrotated log WILL fill your disk and take the server down.** This section is not optional.

logrotate is not a daemon. It's a **program run once a day** by `logrotate.timer` (systemd) or `/etc/cron.daily/logrotate`. It reads `/etc/logrotate.conf` (global defaults) and every file in `/etc/logrotate.d/` (per-package configs), and it keeps its own state in `/var/lib/logrotate/status`.

```bash
$ cat /var/lib/logrotate/status
logrotate state -- version 2
"/var/log/syslog" 2026-7-12-6:25:1
"/var/log/nginx/access.log" 2026-7-12-6:25:1
#  ← this is how it knows a file was "already rotated today". Delete this and
#     logrotate will rotate everything again on the next run.
```

### A real config, annotated

```
/var/log/myapp/*.log {
    daily                       # rotate once a day (also: weekly, monthly, hourly)
    size 100M                   # ...OR when it exceeds 100M. With `daily` present,
                                #    logrotate rotates when EITHER condition holds.
                                #    (`maxsize` = time AND size; `minsize` = size, but
                                #     not before the time interval.)
    rotate 14                   # keep 14 old copies, then DELETE the oldest.
                                #    rotate 0 = delete immediately, keep nothing.
    compress                    # gzip the rotated files → app.log.2.gz  (topic 25)
    delaycompress               # DON'T gzip app.log.1 on this run — wait for the next.
                                #    ◀◀ WHY: the app may still hold an open fd on the
                                #       file we just renamed and still be writing to it.
                                #       Compressing it right now would truncate/corrupt.
    missingok                   # if the file doesn't exist, don't error (fresh deploys)
    notifempty                  # don't rotate a 0-byte file (avoids 14 empty .gz files)
    create 0640 www-data adm    # after renaming, CREATE a new empty log file with
                                #    THIS mode/owner/group.
                                #    ◀◀ GET THIS WRONG AND YOUR APP CANNOT WRITE TO ITS
                                #       OWN LOG. If it says `create 0640 root root` and
                                #       your app runs as www-data → EACCES on next write.
    dateext                     # name it app.log-20260712 instead of app.log.1
                                #    (much easier to reason about; changes the "roll the
                                #     numbers up" behaviour to "stamp the date")
    su root adm                 # run the rotation as this user:group (needed when the
                                #    log dir isn't root-owned; logrotate refuses otherwise)
    sharedscripts               # with a glob matching 5 files, run postrotate ONCE,
                                #    not 5 times. Almost always what you want.
    postrotate
        # tell the app to REOPEN its log file — see below, this is the whole ballgame
        /usr/bin/systemctl kill -s USR1 myapp.service 2>/dev/null || true
    endscript
}
```

| Directive | What it does | Gotcha |
|---|---|---|
| `daily`/`weekly`/`monthly` | Time-based rotation | Only checked when logrotate *runs* (once a day) |
| `size 100M` | Size-based | Ignores the time interval |
| `rotate N` | Keep N old files | `rotate 0` deletes immediately |
| `compress` | gzip the rotated file | Uses `compresscmd` (default gzip) |
| `delaycompress` | Postpone gzip by one cycle | **Needed whenever the app might still be writing to the renamed file** |
| `missingok` | Tolerate a missing file | Without it, logrotate errors and *may skip later entries* |
| `notifempty` | Skip empty files | |
| `create MODE USER GROUP` | Recreate the log with these perms | **The #1 cause of "app stopped logging after rotation"** |
| `copytruncate` | Copy-then-truncate instead of rename | See below — has a race window |
| `dateext` | Date-stamped names | |
| `sharedscripts` | Run scripts once per glob, not per file | |
| `prerotate`/`postrotate` | Shell hooks | `postrotate` runs **after** rename, **before** compress |
| `su USER GROUP` | Drop privileges for this rule | |
| `maxage 30` | Delete rotated logs older than 30 days | Belt-and-braces with `rotate N` |

---

## ⭐ THE FUNDAMENTAL ROTATION PROBLEM (the heart of this doc)

Remember from **topic 04 — Inodes** and **topic 19 — File Descriptors**: **a filename is just a directory entry pointing at an inode. An open file descriptor points at the INODE, not at the name.**

Now watch what logrotate's default (`create`) mode does.

```
BEFORE ROTATION
───────────────
  directory entry              inode 812              your Node process
  /var/log/app.log  ─────────► [ 2.1 GB of data ] ◄──── fd 5  (open since boot)
                                link count = 1


STEP 1: logrotate calls rename("/var/log/app.log", "/var/log/app.log.1")
───────────────────────────────────────────────────────────────────────
  ⚠ rename() only rewrites the DIRECTORY ENTRY. The inode is UNTOUCHED.

  /var/log/app.log.1 ────────► [ inode 812, 2.1 GB ] ◄──── fd 5   ← STILL POINTS HERE


STEP 2: logrotate calls open("/var/log/app.log", O_CREAT, 0640)
───────────────────────────────────────────────────────────────
  /var/log/app.log   ────────► [ inode 999, 0 bytes ]     ← a BRAND NEW inode
  /var/log/app.log.1 ────────► [ inode 812, 2.1 GB ] ◄──── fd 5   ← the app is
                                                                    STILL HERE


THE RESULT — three simultaneous disasters:
──────────────────────────────────────────
  ① Your app keeps write()-ing to fd 5 → the bytes land in app.log.1. FOREVER.
  ② The new app.log stays 0 bytes forever. `tail -f app.log` shows NOTHING.
     You conclude "the app stopped logging" and go looking for a bug in your code.
  ③ Next rotation renames app.log.1 → app.log.2, then app.log.14 gets UNLINKED.
     But your fd is still open on it! (Topic 09/19.) Link count → 0, but the
     kernel CANNOT free the blocks while an fd is open. The disk fills, and
     `du` shows NOTHING using the space. `df` says 100%. `du -sh /var/log` says 3GB.
     You will lose an hour to this.
```

```bash
# PROVE IT: find a deleted-but-still-open log eating your disk
sudo lsof +L1 | grep -i log
# COMMAND  PID   USER  FD  TYPE DEVICE   SIZE/OFF NLINK  NODE NAME
# node    1247 deploy   5w  REG  259,1 2147483648     0   812 /var/log/app.log.1 (deleted)
#                                                     ^ NLINK=0 → unlinked, but held open
```

### Fix (a) — `copytruncate`

```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate            ◀◀ instead of `create`
}
```

What it does at the syscall level:

```
  ① copy the contents:   open(app.log) → read() → write() → app.log.1
  ② truncate the original IN PLACE:  ftruncate(fd_of_app.log, 0)

  The INODE (812) never changes. The app's fd 5 remains valid and pointing at
  the same inode. It keeps writing — into the now-empty file. It works.
```

**The catch — a real race window:** between step ① (the copy finishes) and step ② (the truncate), the app may write more lines. Those lines are **silently lost**. Also, the app's fd has a file *offset* — say 2.1 GB. After truncate, the next `write()` at offset 2.1 GB creates a **sparse file** (a hole). `ls -l` will show a 2.1 GB file; `du` shows a few KB. Confusing but harmless — most apps that open with `O_APPEND` avoid it, because `O_APPEND` re-seeks to end-of-file on every write.

**Use `copytruncate` when:** you don't control the app and it can't reopen its log. It's the pragmatic default for third-party binaries.

### Fix (b) — `create` + `postrotate` signal to REOPEN

This is the **correct** fix. Rename the file, then tell the app: *close your fd and open the path again.*

```
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
        #                        ^^^^^^^^^^ nginx's documented "reopen logs" signal.
        #                        `nginx -s reopen` does exactly this.
    endscript
}
```

**Why a signal?** (Topic 17 — Process Management.) The app cannot detect a rename — the kernel gives it no notification. You must *tell* it. By convention:

| Signal | Meaning by convention | Who uses it |
|---|---|---|
| `SIGHUP` (1) | "Reload your config" — historically "the terminal hung up," repurposed because daemons have no terminal | rsyslog, sshd, most daemons |
| `SIGUSR1` (10) | App-defined. **nginx: reopen log files.** | nginx, many others |
| `SIGUSR2` (12) | App-defined. nginx: upgrade the binary. | |

**This is literally why "SIGHUP means reload" exists** — a daemon has no controlling terminal, so SIGHUP could never mean what it originally meant, so daemons repurposed it as "re-read your files."

For a Node app, wire it up yourself:

```js
// in your Node app — a hand-rolled reopen handler
process.on('SIGUSR1', () => {
  logStream.end();
  logStream = fs.createWriteStream('/var/log/myapp/app.log', { flags: 'a' });
});
```
> ⚠️ **Node caveat:** Node uses `SIGUSR1` internally to start the debugger. Prefer `SIGHUP` for your Node app, or better — see the next section and don't write files at all.

---

## The Modern Answer: Don't Write Log Files At All

The 12-factor rule: **a process should write its event stream, unbuffered, to `stdout`. It should not manage log files. It should not know what a log file is.**

```
OLD (bad)                            NEW (right)
─────────────────────────────────    ──────────────────────────────────────
node app.js                          node app.js
  └─► fs.createWriteStream(            └─► console.log() → fd 1
        '/var/log/app.log')                  │
        │                                    ▼
        ▼                              systemd captured fd 1 as a socket
   YOU own: rotation, perms,               to journald
   reopen-on-SIGHUP, disk-full,            │
   the create/copytruncate choice           ▼
                                       journald: rotation, size caps,
                                       retention, indexing, structured
                                       metadata — ALL FREE. Zero config.
                                       (Or: Docker captures it. Or: your
                                        PaaS captures it.)
```

In your systemd unit (topic 22), `StandardOutput=journal` is already the default. You get rotation for free via `/etc/systemd/journald.conf`:

```
[Journal]
Storage=persistent
SystemMaxUse=1G            # hard cap on total journal size — ◀◀ SET THIS
SystemMaxFileSize=128M
MaxRetentionSec=1month
```

---

## Docker Logging — And the Trap That Kills Hosts

```bash
docker logs -f --tail 100 --since 10m myapp
#           │  │           │
#           │  │           └── time window
#           │  └── don't dump the whole history
#           └── follow
```

`docker logs` reads a file the daemon wrote:

```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

Each line is a JSON object: `{"log":"listening on 3000\n","stream":"stdout","time":"2026-07-12T14:02:01.1Z"}`

### ⚠️ THE DEFAULT `json-file` DRIVER HAS **NO ROTATION**

A chatty container writes **forever**. It fills `/var/lib/docker`, which is on `/` on most hosts. When `/` hits 100%:

- every other container dies
- the Docker daemon itself can't write
- Postgres can't write its WAL and shuts down
- **you cannot even SSH in comfortably** because `sshd` can't write `auth.log`

**One noisy container takes down the entire host.** Fix it:

```bash
# per-container
docker run --log-opt max-size=10m --log-opt max-file=3 myapp

# in docker-compose.yml
services:
  myapp:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

# BEST — a host-wide default in /etc/docker/daemon.json (then: systemctl restart docker)
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

> Note: `daemon.json` defaults only apply to **newly created** containers. Existing ones keep their old settings — you must recreate them.

---

## Structured Logging for Node

```js
// printf logging — unparseable
console.log(`user ${id} checkout failed: ${err.message}`);
// → "user 88 checkout failed: timeout"
//   You cannot query this. You cannot aggregate it. It is prose.

// structured logging — with pino
const pino = require('pino');
const log = pino();
log.error({ userId: 88, reqId: 'a3f9', err, durationMs: 30011 }, 'checkout failed');
// → {"level":50,"time":1752328921,"pid":1247,"userId":88,"reqId":"a3f9",
//    "durationMs":30011,"msg":"checkout failed"}
```

Now it's **queryable**:

```bash
journalctl -u myapp -o cat --since "1 hour ago" \
  | jq -c 'select(.level >= 50)'                    # all errors+fatal

journalctl -u myapp -o cat --since today \
  | jq -r 'select(.durationMs > 1000) | [.reqId, .durationMs, .msg] | @tsv' \
  | sort -k2 -rn | head                             # slowest requests
```

### Log levels — have discipline

| pino level | Use for | In prod? |
|---|---|---|
| 60 `fatal` | Process is about to die | Yes — page |
| 50 `error` | A request failed, an exception was caught | Yes |
| 40 `warn` | Degraded but handled (retry succeeded, cache miss storm) | Yes |
| 30 `info` | Lifecycle: startup, shutdown, one line per request | Yes (default level) |
| 20 `debug` | Detailed flow | **No** — turn on temporarily |
| 10 `trace` | Every function call | Never |

Set the level from an env var (`LOG_LEVEL=debug`, topic 14) so you can raise it during an incident without a redeploy.

### Correlation / request IDs

Generate an ID at the edge (nginx `$request_id`, or a middleware), attach it to *every* log line for that request, and **pass it downstream** in a header (`X-Request-Id`). Then:

```bash
journalctl -u api -u worker -u payments --since "1h ago" -o cat | jq -c 'select(.reqId=="a3f9")'
```

One request's entire journey across three services, in order. This is the difference between debugging and guessing.

### 🚫 Never log secrets or PII

Passwords, tokens, JWTs, API keys, full card numbers, emails, addresses. Once logged they are:
- on disk, world-readable-ish
- in your rotated `.gz` archives
- shipped to your log vendor
- **in their backups, for years, in a jurisdiction you didn't choose**

Redact at the logger. pino: `pino({ redact: ['req.headers.authorization', 'password', '*.token'] })`.

### Centralized logging

Eventually you have 12 servers and 40 containers, and `ssh && grep` stops working. You ship logs to **ELK/OpenSearch**, **Grafana Loki**, **CloudWatch Logs**, or **Datadog** — an agent (`filebeat`, `promtail`, `vector`, `fluent-bit`) tails the journal or the files and pushes them to a central index you can query across every host at once. Structured JSON logs are what make this actually useful; printf logs just become a slow, expensive grep.

---

## Exact Syntax Breakdown

```
tail -F -n 200 /var/log/nginx/error.log
│    │  │      │
│    │  │      └── the file
│    │  └── start by showing the last 200 lines (default 10)
│    └── follow BY NAME: if the file is renamed/recreated (logrotate!), REOPEN it.
│        Lowercase -f follows the FD and goes silent after rotation. Use -F. Always.
└── tail
```

```
grep -C 20 -n "TypeError" /var/log/myapp/app.log
│    │     │  │           │
│    │     │  │           └── file
│    │     │  └── pattern
│    │     └── show line numbers
│    └── 20 lines of Context (before AND after) — a Node stack trace + the request
│        that caused it
└── grep
```

```
logrotate -d -f /etc/logrotate.d/myapp
│         │  │  │
│         │  │  └── the config to test
│         │  └── FORCE rotation even if not due (ignores the daily interval + state file)
│         └── DEBUG / dry-run: print exactly what it WOULD do, change NOTHING.
│             ◀◀ -d implies "don't do it". Never combine -d and -f expecting a rotation.
│                Run -d first to check, then run -f (WITHOUT -d) to actually do it.
└── logrotate
```

```
truncate -s 0 /var/log/myapp/app.log
│        │  │ │
│        │  │ └── file (must already exist; -s 0 on a missing file CREATES it)
│        │  └── new size: zero bytes
│        └── --size
└── truncate — calls ftruncate(). The INODE SURVIVES. The app's open fd stays valid.
              ◀◀ THIS IS WHY YOU USE IT INSTEAD OF rm ON A LIVE LOG.
```

---

## Example 1 — Basic

```bash
# 1. Send your own message into the syslog system, with a facility & severity
logger -p local0.warning -t mytest "hello from the shell"
#      │                  │         └── the message
#      │                  └── -t = tag (the program name that appears in the log)
#      └── -p = priority = facility.severity

# 2. Find it — it went to the journal, and (via rsyslog) to /var/log/syslog
journalctl -t mytest -n 5
# Jul 12 14:22:10 web-1 mytest[3311]: hello from the shell

grep mytest /var/log/syslog
# Jul 12 14:22:10 web-1 mytest[3311]: hello from the shell

# 3. Watch a log grow, live. Open a second terminal and run logger again.
tail -F /var/log/syslog

# 4. Look at your own login history — auth.log is where identity lives
sudo grep -E "Accepted|Failed" /var/log/auth.log | tail -5
# Jul 12 09:01:44 web-1 sshd[2201]: Accepted publickey for deploy from 203.0.113.7 port 51422 ssh2: ED25519 SHA256:x9d...
# Jul 12 09:02:10 web-1 sudo:   deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/systemctl restart myapp

# 5. Did the kernel kill anything?
sudo dmesg -T | grep -i "killed process" || echo "no OOM kills, good"

# 6. How big is the journal, and cap it
journalctl --disk-usage
# Archived and active journals take up 1.1G in the file system.
```

---

## Example 2 — Production Scenario

**02:47. PagerDuty. `myapp` is returning 502. You SSH in.**

```bash
$ ssh deploy@web-1
-bash: cannot create temp file for here-document: No space left on device
#      ◀◀ the shell can't even write. The disk is FULL. This is a disk-full outage.

$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        49G   49G     0 100% /            ◀◀ 100%. Zero bytes free.
tmpfs           3.9G     0  3.9G   0% /dev/shm

# WHO ate it? Walk down, biggest-first. (Topic 09 — du.)
$ sudo du -sh /var/* 2>/dev/null | sort -rh | head -5
44G     /var/log
3.1G    /var/lib
812M    /var/cache

$ sudo du -sh /var/log/* | sort -rh | head -5
41G     /var/log/myapp
2.1G    /var/log/journal
611M    /var/log/nginx

$ sudo ls -laSh /var/log/myapp/
-rw-r--r-- 1 deploy deploy  41G Jul 12 02:47 app.log     ◀◀ FORTY-ONE GIGABYTES
#                                                            No .1, no .gz — this file
#                                                            has NEVER been rotated.

# Confirm the app has it OPEN (this decides rm vs truncate!) — topic 19
$ sudo lsof /var/log/myapp/app.log
COMMAND   PID   USER   FD   TYPE DEVICE    SIZE/OFF    NODE NAME
node     1247 deploy    5w   REG  259,1 44023414784  262147 /var/log/myapp/app.log
#                       ^^ fd 5, open for writing.

# ❌ WRONG:  sudo rm /var/log/myapp/app.log
#    unlink() drops the directory entry, but the kernel CANNOT free the 41G of blocks
#    while node holds fd 5. df STILL says 100%. du shows nothing. You'd have to
#    restart node to reclaim it — an outage you didn't need. (Topic 09/19.)

# ✅ RIGHT: truncate in place. ftruncate() frees the blocks IMMEDIATELY,
#    the inode survives, node's fd 5 stays valid, no restart needed.
$ sudo truncate -s 0 /var/log/myapp/app.log

$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        49G   8.1G   39G  18% /          ◀◀ 41G back, instantly. App is alive.

# --- Now FIX THE ROOT CAUSE: there is no logrotate config for this app. ---
$ ls /etc/logrotate.d/ | grep -i myapp
# (nothing)

$ sudo tee /etc/logrotate.d/myapp >/dev/null <<'EOF'
/var/log/myapp/*.log {
    daily
    size 100M
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    su root adm
}
EOF

# TEST IT — -d is a dry run, it changes nothing
$ sudo logrotate -d /etc/logrotate.d/myapp
reading config file /etc/logrotate.d/myapp
Handling 1 logs
rotating pattern: /var/log/myapp/*.log  104857600 bytes (7 rotations)
rotating log /var/log/myapp/app.log, log->rotateCount is 7
renaming /var/log/myapp/app.log to /var/log/myapp/app.log.1  (rotatecount 7, logstart 1)
truncating /var/log/myapp/app.log
#  ↑ exactly what we want, and it made no changes.

# FORCE one real rotation now to confirm end-to-end
$ sudo logrotate -f /etc/logrotate.d/myapp
$ ls -la /var/log/myapp/
-rw-r--r-- 1 deploy deploy    2841 Jul 12 02:53 app.log      ← app is writing again ✅
-rw-r--r-- 1 deploy deploy 1284211 Jul 12 02:53 app.log.1    ← the rotated copy

# Confirm the app is STILL writing to the live file (the copytruncate check)
$ tail -F /var/log/myapp/app.log
{"level":30,"time":...,"msg":"GET /health 200"}     ✅ lines flowing

# Set an ALERT so this never pages you again
#   (a cron entry — topic 20)
$ echo '*/10 * * * * root [ $(df --output=pcent / | tail -1 | tr -dc 0-9) -gt 85 ] && logger -p daemon.crit "DISK / OVER 85%"' \
  | sudo tee /etc/cron.d/disk-alert
```

**Postmortem in one line:** the app wrote logs to a file, nobody wrote a logrotate config, the file grew unbounded for 8 months, the disk filled, and everything on the box stopped. **The permanent fix is to log to stdout and let journald handle it** — then this class of outage cannot happen.

---

## Common Mistakes

### Mistake 1 — `rm` on a live log file

**Wrong:** `sudo rm /var/log/myapp/app.log` when the disk is full.
**What actually happens (kernel level):** `unlink()` removes the *directory entry* and decrements the inode's link count to 0. But the inode also has a **reference count from open file descriptors**. Node holds fd 5. The kernel only frees the data blocks when **both** counts hit zero. So `df` still reports 100% full, and `du` — which walks directory entries — reports nothing using the space. (Topics 04, 09, 19.)
**Diagnose:** `sudo lsof +L1` → look for `(deleted)` with `NLINK 0`.
**Fix:** you must now restart the app (or reopen the fd) to reclaim the space. Which is exactly the outage you were trying to avoid.
**Right:** `sudo truncate -s 0 /var/log/myapp/app.log` — `ftruncate()` frees the blocks immediately and leaves the inode and every open fd intact.
**Prevent:** never `rm` a file a process has open. Check with `lsof` first.

---

### Mistake 2 — `create` in logrotate, but the app never reopens

**Wrong:**
```
/var/log/myapp/*.log { daily rotate 7 create 0644 deploy deploy }
```
with a Node app that opened the log once at boot and has no SIGHUP handler.
**Root cause:** `rename()` doesn't touch the inode. The app's fd still points at the old inode, now named `app.log.1`. It writes there forever. The new `app.log` never grows.
**What the broken state looks like:**
```bash
$ ls -la /var/log/myapp/
-rw-r--r-- 1 deploy deploy        0 Jul 12 03:00 app.log     ◀◀ 0 bytes, 3 days old
-rw-r--r-- 1 deploy deploy 8.2G     Jul 12 14:02 app.log.1   ◀◀ STILL GROWING
$ sudo lsof -p $(pgrep -f 'node.*app.js') | grep log
node 1247 deploy 5w REG 259,1 8804532224 812 /var/log/myapp/app.log.1   ← there it is
```
**Fix:** `copytruncate`, **or** `create` + a `postrotate` that signals the app to reopen.
**Prevent:** after adding any logrotate config, run `logrotate -f`, then `tail -F` the live file and confirm lines still appear.

---

### Mistake 3 — `create 0640 root root` when the app runs as `www-data`

**Root cause:** logrotate creates the new file with the mode/owner you told it to. The app, running as UID 33 (`www-data`), then calls `open("/var/log/app.log", O_WRONLY|O_APPEND)` and the **kernel** checks the permission bits: owner is root, mode 0640, `www-data` is not the owner and not in the group. → `EACCES`. (Topics 05, 06.)
**What you see:** the app logs nothing after 06:25 (when logrotate ran); some apps crash outright.
**Fix:**
```
create 0640 www-data adm
#           ^^^^^^^^ the user the APP runs as — check with `ps -o user= -p <pid>`
```
**Prevent:** copy the ownership of the *existing* log file: `stat -c '%U %G %a' /var/log/myapp/app.log`.

---

### Mistake 4 — `tail -f` a rotating log and thinking the app died

**Wrong:** `tail -f /var/log/nginx/access.log` left open across 06:25.
**Root cause:** `tail -f` follows the **file descriptor**, which follows the **inode**. After rotation your terminal is silently tailing `access.log.1`. Traffic looks like it stopped dead.
**Right:** `tail -F` — follows the **name**, notices the inode changed, reopens. (`-F` = `--follow=name --retry`.)
**macOS note:** BSD `tail` supports `-F` too, but not `--retry`. Fine either way.

---

### Mistake 5 — Docker's default log driver on a chatty container

**Wrong:** `docker run -d myapp` with `console.log` on every request, at 2000 rps.
**Root cause:** the `json-file` driver has **no size limit by default**. It writes to `/var/lib/docker/containers/<id>/<id>-json.log`, on `/`. logrotate does **not** manage it.
**What breaks:** `/` hits 100% → the Docker daemon, Postgres, nginx, and sshd all fail to write. The host is effectively dead, and **it wasn't even your app's container that got hurt first.**
**Diagnose:** `sudo du -sh /var/lib/docker/containers/* | sort -rh | head`
**Fix now:** `sudo truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log`
**Prevent:** set `log-opts` in `/etc/docker/daemon.json`, restart docker, **and recreate the containers** (the defaults don't retroactively apply).

---

### Mistake 6 — Logging secrets

**Wrong:** `console.log('incoming request', req.headers)` — which contains `authorization: Bearer eyJhbGci...`.
**Root cause:** logs are permanent by design. That token is now in `app.log`, in `app.log.3.gz`, in your S3 log archive, in Elasticsearch, and in your vendor's snapshot backups.
**Fix:** rotate the leaked credential *immediately* — you cannot un-log it.
**Prevent:** redaction in the logger config, plus a CI grep for `console.log(req.headers`.

---

## Hands-On Proof

```bash
# PROVE IT: rename() does NOT change the inode — the heart of the rotation problem
cd /tmp && echo "line1" > proof.log
ls -i proof.log                 # e.g. 262151 proof.log
# open an fd on it, in the background, and keep writing
( while true; do echo "tick $(date +%s)"; sleep 1; done ) >> proof.log &
BGPID=$!
mv proof.log proof.log.1        # ← this is what logrotate does
touch proof.log                 # ← this is `create`
ls -i proof.log proof.log.1     # DIFFERENT inode numbers!
sleep 3
wc -l proof.log                 # 0   ← the "new" log is EMPTY. Forever.
wc -l proof.log.1               # growing ← the writer is still on the OLD inode
kill $BGPID; rm -f proof.log proof.log.1

# PROVE IT: truncate frees space, rm on an open file does not
dd if=/dev/zero of=/tmp/big.log bs=1M count=500 2>/dev/null
exec 9< /tmp/big.log            # hold an fd open in THIS shell
df -h /tmp | tail -1
rm /tmp/big.log                 # unlink
df -h /tmp | tail -1            # space NOT returned — the fd holds the inode
sudo lsof +L1 2>/dev/null | grep big.log   # NLINK=0, "(deleted)"
exec 9<&-                       # close the fd → NOW the kernel frees it
df -h /tmp | tail -1            # space returned

# PROVE IT: syslog PRI = facility*8 + severity
logger -p local0.err "pri test"          # local0=16, err=3 → 16*8+3 = 131
journalctl -t logger -n 1 -o json | jq '{PRIORITY, SYSLOG_FACILITY, MESSAGE}'
# { "PRIORITY": "3", "SYSLOG_FACILITY": "16", "MESSAGE": "pri test" }

# PROVE IT: journald attaches metadata YOU CANNOT FORGE
journalctl -u ssh -n 1 -o verbose | grep -E '_PID|_UID|_COMM|_SYSTEMD_UNIT'
# fields with a LEADING UNDERSCORE are trusted — journald got them from the kernel,
# not from the message. An app cannot lie about its own _UID.

# PROVE IT: your systemd service's stdout is a SOCKET, not a terminal
sudo ls -la /proc/$(systemctl show -p MainPID --value nginx)/fd/1
# lrwx------ 1 root root 64 Jul 12 14:30 /proc/901/fd/1 -> 'socket:[28841]'
#                                                            ^^^^^^ journald

# PROVE IT: logrotate remembers what it already rotated
sudo cat /var/lib/logrotate/status | head

# PROVE IT: dmesg is a fixed-size RAM ring buffer
sudo dmesg | wc -l ; cat /proc/sys/kernel/printk_ratelimit
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal:
1. Emit three messages with different severities: `logger -p local0.info`, `-p local0.warning`, `-p local0.err`.
2. Find all three with `journalctl -t <your tag>`.
3. Now show **only** the error one, using `-p err`. Confirm the info and warning ones are hidden.
4. Show the same message as JSON (`-o json`) and pipe it to `jq` to print only `PRIORITY` and `MESSAGE`.
5. Run `journalctl --disk-usage`. How big is your journal?

Paste your terminal output.

---

### Exercise 2 — Medium

Parse an nginx access log with `awk` (create a fake one if you don't have nginx):

```bash
cat > /tmp/access.log <<'EOF'
203.0.113.7 - - [12/Jul/2026:14:02:01 +0000] "GET /api/users HTTP/1.1" 200 1234 "-" "curl/8.0"
198.51.100.2 - - [12/Jul/2026:14:02:03 +0000] "POST /api/checkout HTTP/1.1" 500 87 "-" "Mozilla/5.0"
203.0.113.7 - - [12/Jul/2026:14:02:05 +0000] "GET /api/users HTTP/1.1" 200 1234 "-" "curl/8.0"
198.51.100.2 - - [12/Jul/2026:14:02:09 +0000] "POST /api/checkout HTTP/1.1" 500 87 "-" "Mozilla/5.0"
10.0.0.4 - - [12/Jul/2026:14:03:01 +0000] "GET /healthz HTTP/1.1" 200 2 "-" "kube-probe/1.28"
EOF
```

Produce, with one pipeline each:
1. A count of requests per status code, sorted descending.
2. The single URL with the most 5xx responses.
3. The top talker IP.
4. Total bytes sent (sum of `$10`).

Paste your commands and output.

---

### Exercise 3 — Hard (Production Simulation)

Reproduce the rotation disaster, then fix it — entirely in the terminal.

```bash
sudo mkdir -p /var/log/fakeapp
```

1. Write a tiny "app" that appends a line every second to `/var/log/fakeapp/app.log` and **keeps its fd open** (a shell `while` loop with `>>` outside the loop is enough). Run it in the background. Record its PID and the log's inode.
2. Write `/etc/logrotate.d/fakeapp` using `create` (no `copytruncate`, no `postrotate`). Run `sudo logrotate -d` first (verify it does nothing), then `sudo logrotate -f`.
3. **Prove the bug:** show that `app.log` is 0 bytes and not growing, that `app.log.1` IS growing, and use `lsof -p <pid>` to show the process is holding the old inode.
4. Now fix the config with `copytruncate`. Force another rotation.
5. **Prove the fix:** `tail -F /var/log/fakeapp/app.log` and show lines appearing in the *live* file. Show with `ls -i` that the inode did NOT change this time.
6. Bonus: simulate the disk-full fix — `truncate -s 0` the file while the writer is running, and show with `lsof`/`stat` that the writer's fd survived.

Paste your full terminal session.

---

## Mental Model Checkpoint

Answer from memory.

1. **What are the three overlapping logging systems on an Ubuntu 22.04 server, and where does each store its data?**
2. **A syslog message is tagged `<34>`. What facility and severity is that, and what's the arithmetic?**
3. **Explain, at the inode level, why an app keeps writing to `app.log.1` after logrotate runs with `create`. What are the two fixes, and what is the downside of each?**
4. **The disk is 100% full because of a 40GB log a running process has open. Why is `rm` the wrong command, and what does the kernel do differently for `truncate -s 0`?**
5. **What is the difference between `tail -f` and `tail -F`, and which one breaks at 06:25 every morning?**
6. **Why does `SIGHUP` conventionally mean "reload"? What does nginx do with `SIGUSR1`?**
7. **You have a 502. Write the one `journalctl` command that shows nginx and your app interleaved on one timeline for the last 10 minutes.**
8. **What is the default rotation policy of Docker's `json-file` log driver, and what is the blast radius when it bites?**

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `tail` | Show/follow the end of a file | `-F` (follow by NAME — survives rotation), `-n 200` |
| `less` | Pager | `+F` (follow mode), `/` search, `Ctrl-C` to stop following |
| `grep` | Search text | `-C N` (context), `-i`, `-c` (count), `-E` (regex) |
| `zgrep` / `zcat` / `zless` | Same, on `.gz` files | Works on rotated logs directly |
| `awk` | Field extraction | `'{print $9}'`, `'$9>=500 {print $7}'` |
| `journalctl` | Read the systemd journal | `-u UNIT` (repeatable!), `-f`, `-p err`, `--since`, `--grep`, `-o json`, `-b -1`, `-k`, `--disk-usage`, `--vacuum-size=` |
| `logger` | Send a message into syslog | `-p facility.severity`, `-t TAG` |
| `dmesg` | Kernel ring buffer | `-T` (human timestamps), `-w` (follow), `-l err` |
| `logrotate` | Rotate logs | `-d` (dry run — SAFE), `-f` (force), `-v` (verbose) |
| `truncate` | Resize a file in place | `-s 0` (**empty a live log without breaking the app's fd**) |
| `lsof` | List open files | `+L1` (unlinked-but-open — the "disk full but du shows nothing" tool), `-p PID` |
| `df` / `du` | Free space / used space | `df -h`, `du -sh /var/log/* \| sort -rh` |
| `docker logs` | Container stdout/stderr | `-f`, `--tail 100`, `--since 10m` |
| `last` / `lastb` | Login history / FAILED logins | Read the binary `wtmp`/`btmp` |

---

## When Would I Use This at Work?

### Scenario 1: The 502 at 3am
nginx returns 502. You run `journalctl -u nginx -u myapp --since "15 min ago" -o short-iso` and see, on one timeline, your app logging `db query timeout` at 03:14:01 and nginx logging `upstream timed out` at 03:14:31. You now know it's the database, not nginx, not the network — in about 20 seconds. You go look at Postgres.

### Scenario 2: "The app stopped logging"
A teammate says logs went dead three days ago. You run `ls -la /var/log/myapp/` and see `app.log` at 0 bytes and `app.log.1` at 8GB and still growing. You immediately know: logrotate used `create`, the app never reopened its fd, and it's been writing to the rotated file since Tuesday. You switch to `copytruncate`, force a rotation, and confirm with `tail -F`.

### Scenario 3: Someone is trying to break in
Your monitoring flags CPU on the box. You check `sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head` and see 40,000 failed SSH attempts from one IP. You've just done the core loop of topic 35 — read `auth.log`, identify the attacker, and now you go install `fail2ban` and disable password auth (topic 24).

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 04 — Inodes in Depth | The rename/inode/fd mechanics ARE the rotation problem |
| **Builds on** | 19 — File Descriptors | Why `rm` doesn't free space; `lsof +L1` |
| **Builds on** | 10, 11, 12 — Viewing / Text Processing / Pipes | `tail -F`, `grep -C`, `awk`, the `sort | uniq -c` pipeline |
| **Builds on** | 17 — Process Management | `kill -HUP` / `-USR1` to make an app reopen its log |
| **Builds on** | 20 — Cron | logrotate is driven by a timer/cron job |
| **Builds on** | 22 — systemd | journald, `StandardOutput=journal`, unit metadata |
| **Next** | 24 — SSH in Depth | `auth.log` is the SSH log; hardening SSH is what you do after reading it |
| **Used by** | 31 — Disk Management | Logs are the #1 cause of a full disk |
| **Used by** | 33 — Node in Production | Log to stdout, structured JSON, log levels |
| **Used by** | 34 — Performance Investigation | Logs are your first evidence in any incident |
| **Used by** | 35 — Security Basics | `auth.log` → fail2ban → hardening |
