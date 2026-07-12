# 06 — Users and Groups

## ELI5 — The Simple Analogy

Imagine an office building with electronic keycard locks.

- **You** have a name — "Priya from Accounting." But the *door* doesn't know your name. The lock reads a **number** off your card: `1001`. That's all the hardware understands. Numbers.
- **The lobby directory** on the wall maps names to numbers: "Priya = 1001, Raj = 1002." Humans read the directory. Doors don't. It is a *convenience for people*.
- **Every door** has a sign: "Owned by 1001. Team 2000. Everyone else: look, don't touch." The lock compares the number on your card against the numbers on the sign. It never thinks about names.
- **Groups** are team badges. One *primary* team is printed on your card; you can also be on a *supplementary* list of other teams. The lock checks them all.
- **The building manager** carries badge `0`. The lock software has one special rule at the top: *if the badge is 0, open — don't even read the sign.* That is the entire superpower of root.
- **The password vault** is a safe in the basement. The public directory just says "Priya's password: see the safe." The safe (`/etc/shadow`) is bolted shut.

Linux users are badge numbers. The names are for you, not for the kernel.

---

## Where This Lives in the Linux Stack

```
Hardware (CPU, RAM, Disk)
  └── KERNEL   ◀◀◀ THIS TOPIC (the ENFORCEMENT half)
       │       Stores uid/gid NUMBERS in every process's cred struct and every
       │       file's inode. Permission checks are integer compares. The kernel
       │       has NEVER heard the word "deploy" or "www-data".
       │
       └── System Calls (setuid, setgid, getuid, geteuid, setgroups, execve)
            │
            └── C Library (glibc — getpwnam(), getpwuid(), NSS)
                 │   ◀◀◀ THIS TOPIC (the TRANSLATION half)
                 │   Reads /etc/passwd + /etc/group. Turns "deploy" → 1001 and back.
                 │
                 └── Shell (bash — runs id, sudo, su)
                      │
                      └── Commands (id, useradd, passwd, sudo, getent)
                           Userspace programs that EDIT the text files glibc reads.
                           That is ALL they are.
```

**The core idea:** identity is split across TWO layers. The kernel enforces with **numbers**. Userspace names them in **text files**. Every confusing user/group bug comes from forgetting these are separate.

---

## What Is This?

A **user** is a numeric ID (**uid**) the kernel attaches to every process and every file. A **group** is a second numeric ID (**gid**) that grants the same access to several users at once.

Usernames like `root`, `deploy`, `www-data`, `postgres` exist only in userspace text files — chiefly `/etc/passwd` and `/etc/group`. They are a lookup table so humans don't memorize numbers. Delete `/etc/passwd` and your files still have owners; you just can't print their names.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| uid is a number, not a name | Your Docker container writes files as uid 0; on the host they appear owned by `root`, your CI can't delete them, and the deploy fails with `EACCES` |
| `usermod -aG` vs `usermod -G` | You "add" your deploy user to `docker` and silently remove it from `sudo` and `adm`. You lock yourself out |
| Why services run as `www-data`/`postgres` | You run Node as root "because it was easier"; an RCE in one of 1,400 npm deps becomes total server takeover |
| `/etc/shadow` vs `/etc/passwd` | You `chmod 644 /etc/shadow` to "fix a permission error" and hand every hash on the box to any local process |
| visudo exists for a reason | You hand-edit `/etc/sudoers`, fumble a colon, and nobody can sudo. With root SSH disabled, you have bricked admin access |
| Group changes need a fresh login | You add yourself to `docker`, `docker ps` still says permission denied, and you lose 40 minutes blaming Docker |

---

## The Physical Reality

Every process has a `task_struct` in kernel memory holding a `struct cred`:

```
┌── PROCESS: node server.js   PID 8412  (kernel's struct cred) ────────────────┐
│  uid  = 1001   REAL uid — "who launched me"                                   │
│  euid = 1001   EFFECTIVE uid — "who am I FOR PERMISSION CHECKS RIGHT NOW"     │
│  suid = 1001   SAVED uid — "who I'm allowed to switch back to" (setuid progs) │
│  gid/egid/sgid = 1001                    (same three-way split for groups)    │
│  groups = [ 27, 999, 1001 ]   SUPPLEMENTARY GROUPS (sudo=27, docker=999, …)   │
│                                                                               │
│  Not one string anywhere. No "deploy". Just integers.                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

And every file has an inode (from **04 — Inodes in Depth**: the inode holds metadata, NOT the filename):

```
┌── INODE 2621447  (file: /var/www/api/server.js) ─────────────────────────────┐
│  mode = 0100644 (regular file, rw-r--r--)   uid = 1001   gid = 1001   ← NUMBERS
└──────────────────────────────────────────────────────────────────────────────┘
```

### The permission check — the actual kernel algorithm

When your process calls `open()`, the kernel runs this (`generic_permission()` in `fs/namei.c`, simplified):

```
  euid == 0 ?  ──YES──▶ ALLOW. Skip EVERY check.  ◀◀◀ THIS IS ALL ROOT IS.
     │ NO
  euid == inode.uid ? ──YES──▶ use OWNER bits (rwx------) and STOP.
     │ NO                       (do NOT fall through to group/other)
  inode.gid in egid or the supplementary group array ? ──YES──▶ use GROUP bits (---rwx---)
     │ NO
  use OTHER bits (------rwx)
```

Three consequences that trip up almost everyone:

1. **Root is not magic.** Root is one `if (euid == 0) return 0;` at the top. That is literally the whole thing. (From **01 — What Linux Actually Is**: root skips *permission* checks, not physics — root still can't write to a read-only mount.)
2. **The branches are exclusive, not cumulative.** If you are the owner, the kernel uses the owner bits and **stops**. A file with mode `0077` (`----rwxrwx`) owned by you means *you cannot read it* but everyone else can. Being the owner can make you *less* privileged.
3. **`euid` is checked, not `uid`.** That distinction is the entire mechanism behind `sudo`.

---

## How It Works — Step by Step

### Trace: `id` — a number from the kernel, a name from a file

```
COMMAND: id
SHELL DOES:  fork() + execve("/usr/bin/id"); the child INHERITS the shell's cred struct
KERNEL DOES: hands back integers. That is the end of the kernel's involvement.
SYSCALLS:    getuid()   → 1001              ── the KERNEL's answer ──
             getgroups(...) → [27, 999, 1001]
             openat("/etc/passwd", O_RDONLY) ── the USERSPACE lookup begins ──
             read(3, "deploy:x:1001:1001:...", 4096)
             openat("/etc/group", O_RDONLY)
             read(4, "sudo:x:27:deploy\ndocker:x:999:deploy\n", 4096)
OUTPUT:      uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),27(sudo),999(docker)
                 │      │                             │
                 │      └─ from /etc/passwd           └─ from /etc/group
                 └─ from the getuid() SYSCALL
```

`mv /etc/passwd /etc/passwd.bak` then run `id`: you get `uid=1001 gid=1001` with **no names**, and nothing else breaks. That is the proof that names are decoration.

### Trace: what `sudo` actually does

```
COMMAND: sudo systemctl restart myapi        (you are uid 1001, "deploy")

1. bash forks; the child execve()s /usr/bin/sudo, whose inode mode is 104755:
        -rwsr-xr-x  1 root root  /usr/bin/sudo    ← the `s` is the SETUID bit (from 05)
   Because setuid is set, the kernel sets the new process's EUID to the FILE OWNER's
   uid (root = 0), leaving the REAL uid at 1001:
        uid = 1001   (still YOU — sudo uses this to know who you are + which rules apply)
        euid = 0     (root — this is what permission checks use)
2. sudo (euid 0) reads /etc/sudoers (mode 0440) — possible ONLY because euid == 0 —
   and checks whether uid 1001 or a group it's in may run this command.
3. sudo prompts for YOUR password, verifies against /etc/shadow (readable only b/c euid 0).
4. sudo calls setresuid(0,0,0): real, effective AND saved uid all become 0 — fully root.
5. sudo forks + execve()s /usr/bin/systemctl, which inherits uid 0. Kernel: euid 0 → allow.
```

**The insight:** sudo is not a kernel feature. It is an ordinary program that got a temporary euid boost from the setuid bit, read a config file, and decided to keep it. `chmod u-s /usr/bin/sudo` and sudo stops working entirely — it can no longer read `/etc/sudoers`.

---

## The Identity Files — Field by Field

### `/etc/passwd` — the public directory (mode `0644`, world-readable, and it MUST be)

Every program that prints a username (`ls -l`, `ps aux`, `top`) needs uid→name, and those run as ordinary users.

```
deploy:x:1001:1001:Deploy User,,,:/home/deploy:/bin/bash
   │   │  │    │         │              │           │
   │   │  │    │         │              │           └─ (7) SHELL — run at login.
   │   │  │    │         │              │              /usr/sbin/nologin for services.
   │   │  │    │         │              └─ (6) HOME. $HOME. NOT auto-created —
   │   │  │    │         │                 useradd needs -m or the dir won't exist.
   │   │  │    │         └─ (5) GECOS/comment — full name, office, phone. Historical.
   │   │  │    │            `chfn` edits it. Nobody cares.
   │   │  │    └─ (4) PRIMARY GID — stamped on every file this user creates. Exactly ONE.
   │   │  └─ (3) UID  ◀◀◀ what the kernel actually enforces
   │   └─ (2) PASSWORD placeholder. "x" = "the hash is in /etc/shadow, look there."
   │      An EMPTY field 2 = NO PASSWORD REQUIRED. Terrifying.
   └─ (1) USERNAME — the only field the kernel never sees
```

**Why the `x` exists:** originally the hash lived in field 2. But field 2 is in a world-readable file, so any user could copy every hash and brute-force offline. The fix was **shadowing** — move the hashes to `/etc/shadow` (mode `0640`, group `shadow`, unreadable by normal users, so it's FATAL if made public) and leave an `x` pointer behind in the world-readable `/etc/passwd`.

### `/etc/shadow` — the vault (mode `0640`, `root:shadow`) — 9 fields

```
deploy:$6$Xy9k$aB7c…:19700:0:99999:7:14:20089:
   │          │        │    │   │   │  │    │  └─ (9) reserved / unused
   │          │        │    │   │   │  │    └──── (8) EXPIRE — account dies entirely on
   │          │        │    │   │   │  │           this day (days since 1970-01-01)
   │          │        │    │   │   │  └───────── (7) INACTIVE — grace days after expiry
   │          │        │    │   │   └──────────── (6) WARN — days of "expires soon" notice
   │          │        │    │   └──────────────── (5) MAX age — must change every N days
   │          │        │    └──────────────────── (4) MIN age — can't re-change for N days
   │          │        └───────────────────────── (3) LAST CHANGE (days since epoch;
   │          │                                    19700 ≈ 2023-12-01)
   │          └─ (2) HASH — or a special value:
   │                ""    → NO password needed. Login with nothing.
   │                "*"   → nothing will ever match. Login disabled.
   │                "!" / "!!" → LOCKED (passwd -l prepends "!"). "!!" = never set.
   │                "!$6$…" → a real hash, but locked. Unlock = remove the "!".
   └─ (1) USERNAME — the join key back to /etc/passwd
```

### Decoding the hash: `$id$[params$]salt$hash`

```
$6$rounds=5000$Xy9kL2mQ$aB7cD8eF9gH0iJ1kL2mN3oP4…
 │      │          │         └─ the HASH itself
 │      │          └─ the SALT — random PER USER. Two users with password "hunter2"
 │      │             get totally different hashes. Salt is why one rainbow table
 │      │             can't crack every server. It is NOT secret — only UNIQUE.
 │      └─ optional PARAMS (cost factor). More rounds = slower = harder to brute-force.
 └─ ALGORITHM ID
```

| `$id$` | Algorithm | Verdict |
|---|---|---|
| `$1$` | MD5 | Ancient, broken. If you see this, rotate it. |
| `$5$` | SHA-256 | Acceptable |
| `$6$` | SHA-512 | **The Ubuntu 22.04 default.** What you'll see 95% of the time. |
| `$y$` | yescrypt | Debian 12 / Ubuntu 24.04+ default. Memory-hard, GPU-resistant. Better. |

### `/etc/group` (mode `0644`) and `/etc/gshadow` (mode `0640`)

```
docker:x:999:deploy,ci-runner
  │    │  │        └─ (4) MEMBER LIST — users for whom this is a SUPPLEMENTARY group.
  │    │  │           ⚠ Users whose PRIMARY gid is 999 are members too and do NOT
  │    │  │             appear here. This list is INCOMPLETE by design.
  │    │  └─ (3) GID — the number the kernel checks
  │    └─ (2) password placeholder → see /etc/gshadow
  └─ (1) group name
```

`/etc/gshadow` holds group passwords (used by `newgrp` to join a group you're not in) and the group's admin list. Group passwords are almost never used in practice; the file exists for symmetry with shadow.

**Primary vs supplementary:** your **primary** group is field 4 of `/etc/passwd` — exactly one, and it's the gid stamped onto every file you create. **Supplementary** groups are the extra ones listed in `/etc/group`; they grant access but never determine new-file ownership.

### UID ranges — an unwritten contract

```
  0        │ root. The one uid the kernel treats specially.
  1 – 999  │ SYSTEM/SERVICE accounts: daemon, www-data (33), postgres, sshd, systemd-*.
           │ Created by package installs. Nearly all have /usr/sbin/nologin as a shell.
           │ (Debian/Ubuntu: 1–99 static, 100–999 assigned by package scripts.)
  1000 +   │ REGULAR HUMANS. Your `deploy` user. The first human on a fresh box is 1000.
  65534    │ `nobody`/`nogroup` — the least-privileged catch-all (2^16 − 2, a 16-bit relic).
```

These are **policy, not kernel law** — defined in `/etc/login.defs` (`UID_MIN`, `SYS_UID_MAX`). The kernel would happily give uid 5 to a human. Every tool assumes you didn't.

### Why service accounts get `/usr/sbin/nologin`

```
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
                                        ▲ this is the whole defence
```

Field 7 is the program `login`, `su`, and `sshd` exec **after** authenticating. `/usr/sbin/nologin` is a real tiny binary that prints `This account is currently not available.` and `exit(1)`. So if an attacker gets code execution *as www-data* via an nginx/PHP hole and tries `su www-data` or SSHes in as `www-data`, they land in `nologin`, which immediately exits. No interactive shell. (`/bin/false` is the same idea, silent — no message.)

**This is defense in depth, not a wall.** A process *already running* as www-data can still do anything www-data can. That's why the second half matters: **www-data owns almost nothing.** It can't write `/etc`, can't read other users' files, can't read `/etc/shadow`. The blast radius of "attacker becomes www-data" is small **by construction** — and that is precisely why your Node app must not run as root.

---

## Exact Syntax Breakdown

```
id -nG deploy
│  ││ │   └── the user to inspect (omit = yourself)
│  ││ └── -G = print ALL group ids (primary + supplementary)
│  │└── -n = print NAMES instead of numbers
└── id — uid/gid from the kernel, names from /etc/passwd + /etc/group
Output: deploy sudo docker
```

```
useradd -m -s /bin/bash -G sudo,docker -c "Deploy Bot" deploy
│       │  │            │              │               └── the username to create
│       │  │            │              └── -c = GECOS comment (field 5)
│       │  │            └── -G = SUPPLEMENTARY groups (comma-separated, NO SPACES)
│       │  └── -s = login shell (field 7). Default is /bin/sh — rarely what you want.
│       └── -m = CREATE the home dir and copy /etc/skel into it.
│              WITHOUT -m the home dir is listed in /etc/passwd but DOES NOT EXIST.
└── useradd — the low-level, non-interactive, scriptable tool. Exists everywhere.

adduser deploy   — a Debian/Ubuntu INTERACTIVE Perl wrapper over useradd. Prompts for
                   a password, creates the home dir, picks a uid. Nice for humans at a
                   keyboard. NOT on RHEL. NOT for scripts. Use useradd in automation.
```

```
usermod -aG docker deploy
│       ││  │      └── the user to modify
│       ││  └── the group(s) to add
│       │└── -G = SET the supplementary group list
│       └── -a = APPEND  ◀◀◀ OMIT THIS AND YOU DESTROY THE EXISTING LIST
└── usermod — modify an existing account

  usermod -aG docker deploy  →  groups become: deploy, sudo, adm, docker   ✅
  usermod  -G docker deploy  →  groups become: deploy, docker              💀
                                sudo and adm are GONE. You may have just removed
                                your own ability to sudo.
```

```
chage -l deploy
│     │  └── the user
│     └── -l = LIST aging settings (a human-readable view of /etc/shadow fields 3–8)
└── chage — "CHange AGE"
  chage -E 2026-12-31 contractor  → account auto-expires that day (field 8)
  chage -M 90 deploy              → max password age 90 days (field 5)
  chage -d 0 newuser              → force a password change at next login (field 3 = 0)
```

```
getent passwd deploy
│      │      └── the key to look up (omit = dump the whole database)
│      └── the NSS database: passwd, group, shadow, hosts, services…
└── getent — "GET ENTries". Queries through NSS, the same path glibc uses.

WHY NOT `grep deploy /etc/passwd`?  Because /etc/passwd may not be the only source.
/etc/nsswitch.conf decides:
        passwd:  files systemd sss ldap
                 │     │       │   └── a corporate LDAP directory
                 │     │       └── SSSD (Active Directory)
                 │     └── systemd DynamicUser= services
                 └── the local /etc/passwd file
grep sees only the file. getent sees what the lookup ACTUALLY resolves. On any box
with LDAP/AD, grep lies. ALWAYS use getent in scripts.
```

```
sudo -u postgres psql
│    │  │        └── the command to run
│    └──┴── -u = run as THIS user (default: root)
└── sudo — "substitute user do". Needs YOUR password (su needs the TARGET's).

sudo -i    → full LOGIN as root: root's login shell, sources /root/.bashrc + .profile,
             cd's to /root, env RESET to root's.  ✅ PREFER THIS.
sudo su -  → sudo runs `su`, which runs a login shell. Same-ish result, but an extra
             process in the chain and su's own PAM stack. Redundant. Works. But why.
sudo -s    → root shell that KEEPS YOUR ENVIRONMENT ($PATH, $HOME=/home/deploy).
             ⚠ Dangerous: root now runs binaries found on YOUR PATH. If ~/bin is on it
             and someone dropped a malicious `ls` there — root just ran it.
sudo su    → non-login root shell. Worst of both. Avoid.
```

```
visudo -c    (-c = check syntax only)
└── visudo — edits /etc/sudoers with a LOCK and a SYNTAX CHECK BEFORE saving.

WHY YOU MUST USE IT: /etc/sudoers is re-parsed on EVERY sudo invocation. One syntax
error and sudo refuses to run AT ALL ("syntax error near line 28 … quitting"). On a
cloud VM with `PermitRootLogin no` you are now PERMANENTLY LOCKED OUT of admin. visudo
validates BEFORE writing — it CANNOT save a broken file, and it takes a lock.
BETTER: never touch /etc/sudoers. Use a drop-in (the file ends with `@includedir
/etc/sudoers.d`):   sudo visudo -f /etc/sudoers.d/deploy
```

```
The sudoers rule format:
deploy  ALL=(ALL:ALL)  NOPASSWD: /usr/bin/systemctl restart myapi
  │      │   │    │        │              └── COMMAND(S). ABSOLUTE path — else PATH tricks
  │      │   │    │        │                  let them run a different binary.
  │      │   │    │        └── NOPASSWD: no password prompt. SPARINGLY — for CI, not your login.
  │      │   │    └── …and as which GROUPS
  │      │   └── as which USERS they may run it
  │      └── on which HOSTS (a relic of shared sudoers files)
  └── WHO. A username, or %groupname for a group.

%sudo ALL=(ALL:ALL) ALL   ← Ubuntu default: anyone in the `sudo` group runs anything as
                            anyone. This is why `usermod -aG sudo alice` grants full admin.
```

---

## Example 1 — Basic

```bash
id                       # who am I, according to the KERNEL?
# uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),999(docker)
#     ▲ from getuid()      ▲ names looked up in /etc/passwd and /etc/group
id -u                    # 1000        ← just the number. The only part that's real.
whoami                   # ubuntu      ← equivalent to `id -un`; resolves the EFFECTIVE uid

getent passwd ubuntu     # the raw record, all 7 fields
# ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash

# All the SERVICE accounts (uid < 1000) and their shells:
awk -F: '$3 < 1000 {printf "%-12s uid=%-6s shell=%s\n", $1, $3, $7}' /etc/passwd
# root         uid=0      shell=/bin/bash
# daemon       uid=1      shell=/usr/sbin/nologin
# www-data     uid=33     shell=/usr/sbin/nologin    ← nginx workers run as this
# nobody       uid=65534  shell=/usr/sbin/nologin

# The hash is NOT in passwd. Prove it — this FAILS as a normal user:
cat /etc/shadow          # cat: /etc/shadow: Permission denied   ← exactly as designed
sudo grep ^ubuntu /etc/shadow
# ubuntu:$6$sHl2vR8u$K9dF…:19881:0:99999:7:::
#        └─ $6$ = SHA-512

groups                              # ubuntu adm sudo docker
grep -E '^(sudo|docker):' /etc/group
# sudo:x:27:ubuntu
# docker:x:999:ubuntu
#          │  └── member list (supplementary members ONLY)
#          └── the gid the kernel actually checks
```

---

## Example 2 — Production Scenario

**02:14.** PagerDuty fires. Your Node API on `prod-api-03` is 500ing. You SSH in as `deploy`.

```bash
$ pm2 logs api --lines 5
0|api  | Error: EACCES: permission denied, open '/var/log/myapp/app.log'
0|api  |     at Object.openSync (node:fs:596:3)
```

`EACCES` came from the **kernel**, not from Node (**01 — What Linux Actually Is**). The kernel compared numbers and said no. So read the numbers.

```bash
# Step 1: WHO is the process actually running as? Not who you think — who it IS.
$ ps -o pid,uid,user,cmd -C node
    PID   UID USER     CMD
   8412  1001 deploy   node /var/www/api/server.js      ← euid 1001

# Step 2: WHO owns the target?
$ ls -ld /var/log/myapp /var/log/myapp/app.log
drwxr-xr-x 2 root root  4096 Jul 12 02:10 /var/log/myapp
-rw-r----- 1 root adm  10240 Jul 12 02:10 /var/log/myapp/app.log
#            └─ uid 0   └─ gid 4 (adm),  mode 0640

# Step 3: run the KERNEL'S ALGORITHM BY HAND.
$ id deploy
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),27(sudo),999(docker)
#   euid 1001, not 0        → no root bypass
#   1001 == inode.uid (0)?  → NO  → skip owner bits
#   is gid 4 (adm) in [1001,27,999]? → NO  → skip group bits
#   fall through to OTHER bits: mode 0640 → other = ---  → DENIED. EACCES.
#   Mystery solved in 30 seconds, with zero guessing.

# Step 4: WHY did ownership change? Something ran as root and recreated the file.
$ sudo journalctl -u logrotate --since "1 hour ago" | tail -2
Jul 12 02:10:01 prod-api-03 logrotate[8390]: renaming /var/log/myapp/app.log to app.log.1
Jul 12 02:10:01 prod-api-03 logrotate[8390]: creating new /var/log/myapp/app.log mode 0640 uid 0 gid 4
#                                                                                    ▲▲▲▲▲▲▲▲▲▲
# THERE. logrotate runs as ROOT and stamped ITS OWN uid into the replacement inode.

# Step 5: STOP THE BLEEDING.
$ sudo chown deploy:deploy /var/log/myapp /var/log/myapp/app.log
$ pm2 restart api && curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/health
200

# Step 6: FIX IT FOREVER. /etc/logrotate.d/myapp had NO `create` directive, so
# logrotate defaulted to root:adm. Add one so the replacement is deploy-owned:
$ sudo tee -a /etc/logrotate.d/myapp <<'EOF'
    create 0640 deploy deploy    # recreate the rotated file OWNED BY DEPLOY
    copytruncate                 # truncate in place — Node's open fd stays valid
EOF
```

You didn't guess. You read the process's uid, read the inode's uid/gid, ran the kernel's algorithm in your head, found the number mismatch, then closed the root cause: **a root-owned cron job stamping its uid onto a file a non-root process needs.**

---

## The Docker Angle — Where UIDs Get Genuinely Dangerous

**There is ONE kernel and ONE uid space.** A container is just processes with a different filesystem view (namespaces + cgroups). uid 0 inside the container **is** uid 0 on the host — unless you've explicitly enabled user-namespace remapping, and you almost certainly haven't.

```
┌── HOST KERNEL (one kernel, ONE uid space) ─────────────────────────────────────┐
│  ┌── CONTAINER (mount ns + pid ns) ────────────┐                                │
│  │  $ whoami  → root      ← reads the CONTAINER's /etc/passwd. Says "root".     │
│  │  $ id -u   → 0         ← THE NUMBER. THE REAL, HOST-WIDE NUMBER.             │
│  │  $ touch /data/output.json                  │                                │
│  └───────────┬─────────────────────────────────┘                                │
│              │   bind mount: -v /home/dev/project:/data                          │
│              ▼                                                                   │
│   The kernel does ONE open(O_CREAT) and stamps the CALLING PROCESS'S euid into   │
│   the new inode. The euid is 0. THERE IS NO TRANSLATION LAYER.                   │
│                                                                                  │
│   HOST:  $ ls -l /home/dev/project/output.json                                   │
│          -rw-r--r-- 1 root root 0 Jul 12 03:02 output.json                        │
│                       └─ Your CI user (uid 1001) now cannot delete this.          │
│                          `git clean` fails. The next build fails.                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Three consequences:**

1. **Root-in-container + bind mount = root-owned files on the host.** The most common Docker permission bug there is. Every `node_modules` you've had to `sudo rm -rf` came from this.
2. **UID collisions are silent and total.** The `node:*` images create a `node` user at **uid 1000**. If your host's deploy user is *also* uid 1000, then container files written as `node` are owned by `deploy` on the host — and vice versa. As far as the kernel is concerned **they are the same user.** The names differ; the number doesn't, and the number is what counts.
3. **`USER` in the Dockerfile is the fix** — and it must come with matching ownership.

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev              # as root — it must write /app and the npm cache
COPY . .
RUN chown -R node:node /app        # node:* images ALREADY ship a `node` user at uid 1000
USER node                          # ◀◀◀ everything after this — INCLUDING CMD — is uid 1000.
                                   #     An RCE in Express now lands on an unprivileged uid.
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# PROVE IT: without USER you are root; --user takes RAW NUMBERS (no passwd entry needed).
$ docker run --rm node:18-alpine id           # uid=0(root) gid=0(root) …
$ docker run --rm --user 4242:4242 alpine id  # uid=4242 gid=4242 — no name in parens, runs fine

# PROVE IT: the root-owned-files bug, live — and the local-dev fix.
$ mkdir /tmp/bt && docker run --rm -v /tmp/bt:/data alpine touch /data/oops
$ ls -l /tmp/bt/oops                          # -rw-r--r-- 1 root root … ← root-owned. On YOUR host.
$ docker run --rm --user "$(id -u):$(id -g)" -v /tmp/bt:/data alpine touch /data/fine
$ ls -l /tmp/bt/fine                          # -rw-r--r-- 1 deploy deploy … ← yours. Deletable.
```

> **macOS trap:** Docker Desktop runs containers in a Linux VM whose file-sharing layer *silently fakes* ownership, so bind-mounted files often look like yours regardless of the container's uid. **This bug does not reproduce on your Mac — it only bites on the Linux CI runner or the prod host.** That's exactly why it reaches production.

---

## Common Mistakes

### Mistake 1 — `usermod -G` without the `-a`

**Wrong:** `sudo usermod -G docker deploy` **Right:** `sudo usermod -aG docker deploy`

**Root cause:** `-G` **sets** the complete supplementary list — a full replacement, not an addition. Without `-a`, `/etc/group` is rewritten with `deploy` removed from every group line except `docker`. Next login, glibc builds a group array of `[999]` only.

**Broken looks like:**
```bash
$ id deploy
uid=1001(deploy) gid=1001(deploy) groups=1001(deploy),999(docker)
#                                    ▲ sudo (27) and adm (4) are GONE
$ sudo ls
deploy is not in the sudoers file.  This incident will be reported.
```
You just removed your own admin rights. With root SSH disabled (as it should be), you are locked out.

**Fix:** from another root session or the cloud serial console: `usermod -aG sudo,adm,docker deploy`.

**Prevention:** always `-a` — or sidestep `usermod` entirely, since `gpasswd` is purely additive and *cannot* do this:
```bash
sudo gpasswd -a deploy docker    # add.    Physically cannot truncate.
sudo gpasswd -d deploy docker    # delete. Explicit.
```

---

### Mistake 2 — Editing `/etc/sudoers` with a plain editor

**Wrong:** `sudo nano /etc/sudoers` **Right:** `sudo visudo` (or `sudo visudo -f /etc/sudoers.d/deploy`)

**Root cause:** `/etc/sudoers` is re-parsed by `sudo` on **every invocation** — there's no compiled cache. A stray character makes the parse fail, and sudo's fail-closed design means it then refuses to run *anything*:
```
$ sudo whoami
>>> /etc/sudoers: syntax error near line 28 <<<
sudo: no valid sudoers sources found, quitting
```
Note the trap: **the tool you'd use to fix the file requires the file to be valid.** On a cloud VM with `PermitRootLogin no` (**35 — Security Basics**), there is no second path to root. Rescue-boot or rebuild.

**Fix (if you're already there):** use a root shell you still have open. If not — single-user mode, a rescue instance, or the provider's serial console.

**Prevention:** `visudo` locks the file and syntax-checks the result **before** writing; it physically cannot save a broken file. And **always keep a second root session open** while touching sudoers. That session is your undo button.

---

### Mistake 3 — "I added myself to `docker` but it still says permission denied"

**Wrong assumption:** group membership takes effect immediately.

**Root cause — the deepest lesson here.** Supplementary groups are baked into the process's `cred` struct **at login time** and inherited across `fork()`. They are **not** re-read from `/etc/group` per syscall — that would be catastrophically slow. Your running shell holds a *snapshot* from when `sshd` created it.

```
T0: sshd authenticates you → calls initgroups() → reads /etc/group NOW.
    Kernel stamps groups=[1001,27] into your shell's cred struct. bash forks from that.
T1: sudo usermod -aG docker deploy
    → /etc/group ON DISK now says docker:x:999:deploy
    → YOUR RUNNING SHELL'S cred struct: STILL [1001,27].  Nothing re-reads it.
T2: docker ps → open("/var/run/docker.sock")  (root:docker, mode 0660)
    → euid 1001 vs uid 0: no.  gid 999 in [1001,27]: NOT PRESENT.  other: ---
    → EACCES.
```

**Diagnosis — the two commands that disagree:**
```bash
$ groups                # what YOUR PROCESS HAS (the kernel's cred struct)
deploy sudo
$ id -nG deploy         # what the FILE SAYS (a fresh /etc/group lookup)
deploy sudo docker      #                     ▲ they disagree. That IS the bug.
```

**Fix — get a process with fresh credentials:**
```bash
exit && ssh deploy@host   # cleanest. sshd calls initgroups() fresh. ALWAYS works.
newgrp docker             # a NEW shell with docker as the PRIMARY gid (so new files
                          # get gid 999 — subtle side effect). Log out instead.
```

**Prevention:** do group changes in provisioning *before* the user ever gets a shell. Note `systemctl restart myapi` **does** pick up the change — because systemd (PID 1) re-reads `/etc/group` and spawns the new process with fresh groups. Your *shell* is the stale one, not the service.

---

### Mistake 4 — Running your Node app as root

**Wrong:** a systemd unit with no `User=` (it runs as root), or `sudo node server.js`.
**Right:**
```ini
[Service]
User=deploy
Group=deploy
ExecStart=/usr/bin/node /var/www/api/server.js
```

**Root cause at the kernel level:** the check is `if (euid == 0) allow`. A Node process with euid 0 has **no filesystem restrictions at all**. Now price a single RCE in one of your 1,400 transitive npm dependencies:

| App runs as | An RCE gives the attacker |
|---|---|
| `root` (uid 0) | Read `/etc/shadow` (every hash on the box). Write `/root/.ssh/authorized_keys` (permanent backdoor). Load a kernel module. Edit `/etc/sudoers`. **Total, irreversible compromise.** |
| `deploy` (uid 1001) | Read/write `/var/www/api` and `/home/deploy`. That's it. Can't read shadow, can't touch other users, can't persist outside its own home. **Contained** — restore from backup, rotate one credential set. |

**"But I need port 80."** No. Ports < 1024 are privileged (the kernel checks `CAP_NET_BIND_SERVICE` on `bind()`), and there are three correct answers, none of which is "run Node as root":
1. **Put nginx in front.** nginx starts as root, binds :80, then **drops privileges** to `www-data` for its workers. Node listens on 3000 as `deploy`. This is the standard architecture.
2. **Grant the capability, not the uid:** `sudo setcap 'cap_net_bind_service=+ep' $(which node)`. (Caveat: it applies to the node *binary*, so every node process gets it. Prefer #1.)
3. **systemd socket activation** — systemd binds the socket as root and hands the already-bound fd to your unprivileged process.

**Prevention:** make it structural (`User=deploy`; `USER node`), then **verify — never assume**:
```bash
$ ps -o user,uid,cmd -C node
USER       UID CMD
deploy    1001 node /var/www/api/server.js     ← if this says `root`, you have a bug
```

---

### Mistake 5 — `chmod 644 /etc/shadow` to "fix a permission error"

**Wrong:** `sudo chmod 644 /etc/shadow` **Right:** `sudo chmod 640 /etc/shadow && sudo chown root:shadow /etc/shadow`

**Root cause:** shadow is `0640 root:shadow` **specifically so non-root processes cannot read the hashes.** At `0644`, *every* process on the box — including your Node app, including a compromised www-data — can `cat` every hash, exfiltrate it, and crack it offline at leisure. `$6$` (SHA-512) is fast to compute; a GPU rig does billions of guesses/sec. Weak passwords fall in minutes.

**And there is no error message.** Nothing breaks. It just silently becomes catastrophic.

**Detect it:**
```bash
$ ls -l /etc/shadow
-rw-r--r-- 1 root root 1834 Jul 12 03:40 /etc/shadow
#     ▲▲▲  ▲▲▲▲ ▲▲▲▲   └─ group should be `shadow`
#     └─ OTHER can read. This is the emergency.
$ dpkg --verify passwd
??5?????? c /etc/shadow      ← the `5` = perms/checksum differ from the package
```

**Fix:** restore mode *and* group — then **rotate every password on the box**, because you must assume the hashes leaked.
```bash
sudo chown root:shadow /etc/shadow && sudo chmod 640 /etc/shadow && sudo pwck
```

**Prevention:** if a tool says "permission denied on /etc/shadow," the answer is *never* to loosen shadow — it's to run the tool as root, or use PAM. Add a hardening check:
```bash
[ "$(stat -c '%a %U %G' /etc/shadow)" = "640 root shadow" ] || echo "SHADOW PERMS WRONG"
```

---

## Hands-On Proof

```bash
# PROVE IT: to the kernel a user is a NUMBER. Names are optional decoration.
docker run --rm --user 4242:4242 alpine id
# uid=4242 gid=4242 groups=4242    ← no name in parens; no passwd entry; runs fine.

# PROVE IT: `id` gets NUMBERS from the kernel and NAMES from a FILE.
strace -e trace=getuid,getgroups,openat id 2>&1 | grep -E 'getuid|getgroups|passwd|group'
# getuid()                                            = 1000   ← from the KERNEL
# openat(AT_FDCWD, "/etc/passwd", O_RDONLY|O_CLOEXEC) = 3      ← from a FILE
# openat(AT_FDCWD, "/etc/group",  O_RDONLY|O_CLOEXEC) = 3      ← from a FILE

# PROVE IT: sudo works because of the setuid BIT (leading 4), not kernel favouritism.
stat -c '%a %U %G' /usr/bin/sudo               # 4755 root root  (see 05 — File Permissions)

# PROVE IT: real vs effective uid, live from /proc. Then run `sudo sleep 60 &` and
# check ITS pid — Uid becomes `0 0 0 0` because sudo called setresuid(0,0,0).
grep -E '^(Uid|Gid|Groups):' /proc/self/status
# Uid:  1000  1000  1000  1000     ← real, effective, saved, filesystem
# Groups: 4 27 999 1000            ← the supplementary ARRAY, straight from the kernel

# PROVE IT: passwd is world-readable and shadow is not — by design.
stat -c '%a %U:%G %n' /etc/passwd /etc/shadow
# 644 root:root   /etc/passwd     ← everyone reads (needed for uid→name)
# 640 root:shadow /etc/shadow     ← root + `shadow` group only. Hashes live here.

# PROVE IT: root is special ONLY because euid == 0. Same kernel, same inode.
sudo cat /etc/shadow > /dev/null && echo ok   # works
cat /etc/shadow                               # Permission denied — only the euid differed.

# PROVE IT: the kernel does NOT re-read /etc/group for a running process.
groups ; id -nG "$USER"
#   Add yourself to a group in ANOTHER terminal, then re-run both. THEY WILL DIFFER.

# PROVE IT: `nologin` is a real program, not a magic marker.
sudo -u www-data /usr/sbin/nologin ; echo "exit=$?"
# This account is currently not available.
# exit=1

# PROVE IT: which uid each service runs as; and that exactly ONE uid-0 account exists.
ps -eo user,uid,comm --sort=uid | awk '!seen[$1]++'   # www-data 33 nginx; deploy 1001 node
awk -F: '$3 == 0 {print $1}' /etc/passwd              # must print only: root (else = backdoor)
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
id ; id -u ; id -nG
getent passwd "$USER"
getent group sudo
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3}' /etc/passwd
stat -c '%a %U:%G' /etc/passwd /etc/shadow
```

**Questions:** What is your uid? Your primary gid? Which supplementary groups are you in? How many *human* users exist on this box? Why is `/etc/passwd` mode 644 but `/etc/shadow` mode 640 — name the specific attack that prevents. Paste your output.

---

### Exercise 2 — Medium

Build a locked-down service account the way a package installer would, and prove it cannot get a shell.

```bash
# 1. Create a SYSTEM user `apiworker` (uid < 1000), no home dir, /usr/sbin/nologin
#    as its shell, its own primary group.   (Hint: useradd --system --no-create-home -s …)
# 2. Verify the uid landed in the 1–999 range:      getent passwd apiworker
# 3. Prove it cannot log in:                        sudo su - apiworker
#    → What happened, and WHICH FIELD of /etc/passwd caused it?
# 4. Look at field 2 of its shadow entry:           sudo getent shadow apiworker
#    → What is the value, and what does that value MEAN?
# 5. sudo mkdir /srv/apiworker && sudo chown apiworker:apiworker /srv/apiworker
#    sudo -u apiworker touch /srv/apiworker/a      # should SUCCEED
#    touch /srv/apiworker/b                        # should FAIL
#    → Walk the kernel's permission algorithm BRANCH BY BRANCH to explain the denial.
# 6. Clean up:  sudo userdel -r apiworker ; sudo rm -rf /srv/apiworker
```

---

### Exercise 3 — Hard (Production Simulation)

**Scenario:** provision a fresh Ubuntu 22.04 box for a Node API behind nginx. CI must deploy and restart the app **without a password**, but must **not** be able to become root. All in the terminal.

```bash
# 1. Create a `deploy` user: home dir, bash shell, uid >= 1000.

# 2. Create an `app` GROUP. Add BOTH `deploy` and `www-data` to it.
#    (nginx must READ the build output; deploy must WRITE it.)

# 3. Create /var/www/api owned by deploy:app, mode 2750.
#    → Why 2750? What does the leading 2 do? (From 05: setgid on a DIRECTORY.)
#    → PROVE it: create a file in there as deploy and check the file's GROUP.

# 4. Grant deploy PASSWORDLESS sudo for EXACTLY these two commands and nothing else:
#         /usr/bin/systemctl restart myapi
#         /usr/bin/systemctl status  myapi
#    Use a DROP-IN in /etc/sudoers.d/. Use `visudo -f`. Use ABSOLUTE paths.

# 5. Verify:  sudo -l -U deploy
#    Then PROVE the restriction actually holds:
#         sudo -u deploy sudo systemctl restart myapi   # must SUCCEED, no password
#         sudo -u deploy sudo systemctl restart nginx   # must be DENIED
#         sudo -u deploy sudo -i                        # must be DENIED
#         sudo -u deploy sudo cat /etc/shadow           # must be DENIED

# 6. THE TRAP: open a shell as deploy and run BOTH:
#         groups          # and
#         id -nG deploy
#    They DISAGREE. Explain why at the level of the kernel's cred struct.
#    Then fix it WITHOUT rebooting.

# 7. Write /etc/systemd/system/myapi.service so Node runs as deploy:app, NOT root.
#    PROVE it:   ps -o user,uid,cmd -C node
```

**Deliverable:** terminal output for steps 3, 5, 6, 7. Step 5 must show **one success and three denials**. Step 7 must show `deploy` / `1001` — not `root` / `0`.

---

## Mental Model Checkpoint

1. **What does the kernel actually store to represent "who owns this file" and "who is this process"? What does it NOT store?**
2. **Name the 7 fields of `/etc/passwd`. Why is field 2 an `x` rather than a hash — what attack does that prevent?**
3. **In `$6$Xy9k$aB7c…`: what is the `6`, what is `Xy9k`, and why is it safe for the salt to sit in plaintext next to the hash?**
4. **Walk the kernel's check for a process with euid 1001 and groups [1001,27] opening a `root:adm` file with mode 0640. Outcome, and which branch decided it?**
5. **What exactly makes root able to do things others can't? (One sentence — it's one `if` statement.)**
6. **Why is `usermod -G docker deploy` (no `-a`) an outage, and what's the safer command?**
7. **Why does adding yourself to `docker` not take effect until you log out? Which data structure holds the stale copy?**
8. **A container running as root writes to a bind mount. Who owns the file on the host, and why is there no translation layer?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `id` | uid/gid/groups — kernel numbers + file-looked-up names | `-u` (uid), `-un` (name), `-nG` (group names), `-G` (gids) |
| `whoami` | Name for your **effective** uid | — (same as `id -un`) |
| `who` / `w` | Who is logged in / …**and what they're running** | `who -a` |
| `last` | Login history (`/var/log/wtmp`) | `-n 20`; `-f /var/log/btmp` (FAILED logins) |
| `groups` | Groups of your **running process** (may be stale vs `/etc/group`) | `<user>` forces a fresh lookup |
| `useradd` | Low-level, scriptable user creation | `-m` (make home), `-s` (shell), `-G` (supp. groups), `--system`, `-c` (GECOS) |
| `adduser` | Debian/Ubuntu **interactive** wrapper. Not on RHEL. Not for scripts | `--system`, `--disabled-password` |
| `usermod` | Modify a user | **`-aG`** (append — NEVER bare `-G`), `-s`, `-L`/`-U` (lock/unlock) |
| `gpasswd` | Safer, purely additive group membership | `-a <u> <g>` (add), `-d` (delete) |
| `userdel` | Delete a user | `-r` (**also** remove home + mail spool) |
| `groupadd` / `groupdel` | Create / delete a group | `-g` (specific gid), `--system` |
| `passwd` | Set/change a password (writes `/etc/shadow`) | `-l` (lock — prepends `!`), `-u`, `-S` (status), `-e` (expire now) |
| `chage` | Password **aging** (shadow fields 3–8) | `-l` (list), `-M` (max days), `-E` (expiry date), `-d 0` (force change) |
| `su` | Switch user — needs the **target's** password | `-` (full LOGIN shell: resets env, cd's home), `-c` (one command) |
| `sudo` | Run as another user — needs **your** password | `-u <user>`, `-i` (login shell as root), `-l` (list your rights), `-k` (forget cache) |
| `visudo` | **The only safe way** to edit sudoers — locks + syntax-checks | `-c` (check only), `-f <file>` (a drop-in under `/etc/sudoers.d/`) |
| `getent` | Query NSS (sees LDAP/AD, not just files) | `getent passwd <u>`, `group <g>`, `shadow <u>` (root) |
| `newgrp` | New shell with a different **primary** group | `<group>` |
| `pwck` / `grpck` | Verify passwd ↔ shadow consistency | `-r` (read-only) |

---

## When Would I Use This at Work?

### Scenario 1: The 2 AM `EACCES` on a log file
Node dies writing `/var/log/myapp/app.log`. You don't guess — you run `ps -o uid -C node` for the process's number (1001) and `ls -ln` for the inode's numbers (0/4), then run the kernel's algorithm in your head: 1001 ≠ 0, and 4 isn't in the process's group array, so the kernel used the "other" bits, which are `---`. Denied. Root cause: logrotate ran as root and recreated the file with its own uid. Fix: `create 0640 deploy deploy`. Three minutes, because you were reading numbers, not guessing at names.

### Scenario 2: `sudo rm -rf node_modules` on the CI runner
The build starts failing with `EACCES … rmdir 'node_modules'` and `ls` shows `drwxr-xr-x root root node_modules`. You know instantly: a `docker run -v $PWD:/app node:18 npm ci` ran as **uid 0 inside the container**, and the kernel — with exactly one uid space — stamped uid 0 into every inode on the host bind mount. The fix isn't `sudo` in the pipeline; it's `--user "$(id -u):$(id -g)"` on the run, or `USER node` in the Dockerfile.

### Scenario 3: Onboarding a contractor without handing over the keys
They need to deploy and restart one service, nothing else. You create their user, add them to a `deployers` group, and drop one file at `/etc/sudoers.d/deployers` (via `visudo -f`): `%deployers ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapi`. They can restart the app; they cannot `sudo -i`, cannot read `/etc/shadow`, cannot touch nginx. `chage -E 2026-09-30` auto-expires the account when the contract ends. **Least privilege**, built entirely from two numbers and one text file. (Full build-out in **32 — Deploy User Permissions**.)

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | **04 — Inodes in Depth** | The uid/gid the kernel compares live *in the inode* — not in the filename or the directory entry. That's why `chown` edits the inode, and why a hard link shares its owner. |
| **Builds on** | **05 — File Permissions in Depth** | 05 gave the *rwx* side; this doc gives the *identity* side. Together they are the whole check. `sudo` is unintelligible without 05's **setuid** bit, and Exercise 3's `2750` is 05's **setgid-on-a-directory**. |
| **Builds on** | **01 — What Linux Actually Is** | "Root skips permission checks" is the concrete form of 01's kernel/userspace boundary. `EACCES` comes from the kernel — never from Node. |
| **Next** | **07 — How the Shell Works** | The shell forks, and the child **inherits your uid/gid/groups** through `fork()`. Now that you know what identity IS, 07 shows how it rides into every process you launch. |
| **Used by** | **17 — Process Management** | `ps aux`'s USER column, and why you can only `kill` processes whose uid matches yours (unless you're root). |
| **Used by** | **22 — Services and systemd** | `User=` / `Group=` in a `.service` file are exactly this doc's `setuid()`/`setgid()`, made declarative. |
| **Used by** | **24 — SSH in Depth** | sshd **refuses** an `authorized_keys` file whose permissions are too loose — it enforces this doc's ownership rules before it will trust a key. |
| **Used by** | **32 — Deploy User Permissions** | The full production build-out: the `deploy` user, `sudoers.d` drop-ins, least-privilege CI. This doc is its foundation. |
| **Used by** | **35 — Security Basics** | Disabling root SSH, auditing shadow perms, and hunting stray uid-0 accounts (`awk -F: '$3==0' /etc/passwd` — there must be exactly ONE). |
