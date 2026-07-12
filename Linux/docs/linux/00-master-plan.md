# Linux & The Command Line — Deep Mastery Curriculum
## Master Plan & Progress Tracker

---

## Phases Overview

| Phase | Title | Topics | Focus |
|-------|-------|--------|-------|
| 1 | Linux Architecture & Orientation | 01–06 | How Linux is built, what files really are, permissions, users |
| 2 | The Shell & Command Line | 07–15 | How the shell works, navigating, file ops, text processing, scripting |
| 3 | Processes & System Resources | 16–20 | How processes live and die, monitoring, scheduling |
| 4 | System Administration | 21–25 | Packages, services, logs, SSH, compression |
| 5 | Networking from the Linux Side | 26–30 | Interfaces, DNS, ports, firewalls, HTTP from CLI |
| 6 | Production Server Skills | 31–35 | Disk, deploy users, Node in production, performance, security |

---

## Complete Curriculum — Numbered Topics

### PHASE 1 — Linux Architecture & Orientation

| # | File | Topic | Description |
|---|------|-------|-------------|
| 01 | `01-linux-architecture.md` | What Linux Actually Is | Kernel vs shell vs userspace, what a distribution is, why every server runs Linux |
| 02 | `02-filesystem-hierarchy.md` | The Filesystem Hierarchy | Every top-level directory (/, /etc, /var, /usr, /home, /proc, /sys, /dev, /tmp) — what lives where and why |
| 03 | `03-everything-is-a-file.md` | Everything Is a File | Regular files, directories, symlinks, hard links, device files, sockets, pipes — all unified under one abstraction |
| 04 | `04-inodes-in-depth.md` | Inodes in Depth | What an inode is, what it stores, what it does NOT store, the inode table, how filenames map to data |
| 05 | `05-file-permissions.md` | File Permissions in Depth | rwx bits, octal notation, user/group/other, umask, setuid, setgid, sticky bit — the full permission system |
| 06 | `06-users-and-groups.md` | Users and Groups | /etc/passwd, /etc/shadow, /etc/group, uid/gid, root vs regular user, sudo, how the kernel checks identity |

### PHASE 2 — The Shell & Command Line

| # | File | Topic | Description |
|---|------|-------|-------------|
| 07 | `07-how-the-shell-works.md` | How the Shell Works | The fork-exec model, the shell loop, PATH resolution, how typing a command becomes a running process |
| 08 | `08-navigating-the-filesystem.md` | Navigating the Filesystem | cd, ls, pwd, find, locate, which, whereis — moving around and finding things with full option depth |
| 09 | `09-working-with-files.md` | Working with Files | touch, cp, mv, rm, mkdir, rmdir, file, stat, du, df — creating, copying, moving, deleting, and measuring |
| 10 | `10-viewing-and-reading-files.md` | Viewing and Reading Files | cat, less, more, head, tail, tail -f, wc — every way to look at file contents and when to use each |
| 11 | `11-text-processing.md` | Text Processing Powertools | grep (with regex), sed, awk, cut, sort, uniq, tr, xargs — slicing and transforming text in pipelines |
| 12 | `12-pipes-and-redirection.md` | Pipes and Redirection | stdin/stdout/stderr, |, >, >>, 2>, 2>&1, /dev/null, tee — the Unix philosophy of composable tools |
| 13 | `13-shell-scripting.md` | Shell Scripting Fundamentals | Shebang, variables, conditionals, loops, functions, exit codes, $?, $@, $# — writing real scripts |
| 14 | `14-environment-variables.md` | Environment Variables | PATH, HOME, USER, export, source, .bashrc vs .bash_profile vs .profile — how your shell knows things |
| 15 | `15-shell-productivity.md` | Shell Shortcuts & Productivity | history, !!, Ctrl shortcuts, aliases, tab completion, readline — working at speed |

### PHASE 3 — Processes & System Resources

| # | File | Topic | Description |
|---|------|-------|-------------|
| 16 | `16-processes-in-depth.md` | Processes in Depth | PID, PPID, fork, exec, zombie processes, orphan processes, the process tree, /proc/PID |
| 17 | `17-process-management.md` | Process Management | ps, top, htop, kill, killall, pkill, signals (SIGTERM vs SIGKILL vs SIGHUP), jobs, bg, fg, nohup, & |
| 18 | `18-system-resources.md` | System Resource Monitoring | free, vmstat, iostat, sar, /proc/meminfo, /proc/cpuinfo, load average — understanding what "busy" means |
| 19 | `19-file-descriptors.md` | File Descriptors | What they are, /proc/PID/fd, lsof, the 0/1/2 standard descriptors, fd limits (ulimit), fd inheritance |
| 20 | `20-cron-and-scheduling.md` | Cron and Scheduled Tasks | crontab syntax, /etc/cron.d, /etc/cron.daily, at command, systemd timers — running things on schedule |

### PHASE 4 — System Administration

| # | File | Topic | Description |
|---|------|-------|-------------|
| 21 | `21-package-management.md` | Package Management | apt, dpkg, how .deb packages work, dependencies, repositories, /etc/apt/sources.list, pinning |
| 22 | `22-systemd-and-services.md` | Services and systemd | Units, systemctl (start/stop/enable/disable/status), journalctl, writing a .service file for a Node.js app |
| 23 | `23-logs-and-log-management.md` | Logs and Log Management | /var/log, syslog, journald, logrotate, reading and filtering logs when your Node app is misbehaving |
| 24 | `24-ssh-in-depth.md` | SSH in Depth | How SSH works (key exchange, asymmetric crypto), ssh-keygen, authorized_keys, config, tunneling, scp, rsync |
| 25 | `25-compression-and-archiving.md` | File Compression & Archiving | tar, gzip, bzip2, xz, zip — when to use each, extracting in production, streaming compression |

### PHASE 5 — Networking from the Linux Side

| # | File | Topic | Description |
|---|------|-------|-------------|
| 26 | `26-network-interfaces.md` | Network Interfaces | ip addr, ifconfig, lo, eth0, network namespaces, how Docker uses them, routing basics |
| 27 | `27-dns-from-cli.md` | DNS from the Command Line | dig, nslookup, /etc/resolv.conf, /etc/hosts, how resolution works step by step, DNS caching |
| 28 | `28-ports-and-sockets.md` | Ports and Sockets | netstat, ss, lsof -i, LISTEN vs ESTABLISHED, checking what is on port 3000, socket files vs network sockets |
| 29 | `29-firewalls.md` | Firewalls | ufw, iptables basics, chains (INPUT/OUTPUT/FORWARD), opening ports, rules order, checking status |
| 30 | `30-curl-and-wget.md` | curl and wget in Depth | HTTP requests from terminal, headers, POST/PUT, auth, following redirects, --verbose, scripting with curl |

### PHASE 6 — Production Server Skills

| # | File | Topic | Description |
|---|------|-------|-------------|
| 31 | `31-disk-management.md` | Disk Management | fdisk, lsblk, mount, /etc/fstab, df, du, checking disk space before it kills your server, LVM basics |
| 32 | `32-deploy-user-permissions.md` | User & Permission Management for Deployments | Creating a deploy user, restricting permissions, sudoers file, principle of least privilege |
| 33 | `33-node-in-production.md` | Environment & Process Management for Node.js | Running Node in production, PM2, environment variables, systemd for Node, graceful shutdown |
| 34 | `34-performance-investigation.md` | Performance Investigation | Finding what is killing CPU/memory, strace, perf basics, tracing a slow process, diagnosing OOM kills |
| 35 | `35-security-basics.md` | Security Basics | fail2ban, disabling root SSH, checking open ports, reading auth.log, unattended-upgrades, hardening checklist |

---

## Complete File List

```
docs/linux/
├── 00-master-plan.md              ← You are here
├── 01-linux-architecture.md
├── 02-filesystem-hierarchy.md
├── 03-everything-is-a-file.md
├── 04-inodes-in-depth.md
├── 05-file-permissions.md
├── 06-users-and-groups.md
├── 07-how-the-shell-works.md
├── 08-navigating-the-filesystem.md
├── 09-working-with-files.md
├── 10-viewing-and-reading-files.md
├── 11-text-processing.md
├── 12-pipes-and-redirection.md
├── 13-shell-scripting.md
├── 14-environment-variables.md
├── 15-shell-productivity.md
├── 16-processes-in-depth.md
├── 17-process-management.md
├── 18-system-resources.md
├── 19-file-descriptors.md
├── 20-cron-and-scheduling.md
├── 21-package-management.md
├── 22-systemd-and-services.md
├── 23-logs-and-log-management.md
├── 24-ssh-in-depth.md
├── 25-compression-and-archiving.md
├── 26-network-interfaces.md
├── 27-dns-from-cli.md
├── 28-ports-and-sockets.md
├── 29-firewalls.md
├── 30-curl-and-wget.md
├── 31-disk-management.md
├── 32-deploy-user-permissions.md
├── 33-node-in-production.md
├── 34-performance-investigation.md
└── 35-security-basics.md
```

---

## Progress Tracker

### Phase 1 — Linux Architecture & Orientation
- [x] 01 — What Linux Actually Is
- [x] 02 — The Filesystem Hierarchy
- [x] 03 — Everything Is a File
- [x] 04 — Inodes in Depth
- [x] 05 — File Permissions in Depth
- [x] 06 — Users and Groups

### Phase 2 — The Shell & Command Line
- [x] 07 — How the Shell Works
- [x] 08 — Navigating the Filesystem
- [x] 09 — Working with Files
- [x] 10 — Viewing and Reading Files
- [x] 11 — Text Processing Powertools
- [x] 12 — Pipes and Redirection
- [x] 13 — Shell Scripting Fundamentals
- [x] 14 — Environment Variables
- [x] 15 — Shell Shortcuts & Productivity

### Phase 3 — Processes & System Resources
- [x] 16 — Processes in Depth
- [x] 17 — Process Management
- [x] 18 — System Resource Monitoring
- [x] 19 — File Descriptors
- [x] 20 — Cron and Scheduled Tasks

### Phase 4 — System Administration
- [x] 21 — Package Management
- [x] 22 — Services and systemd
- [x] 23 — Logs and Log Management
- [x] 24 — SSH in Depth
- [x] 25 — File Compression & Archiving

### Phase 5 — Networking from the Linux Side
- [x] 26 — Network Interfaces
- [x] 27 — DNS from the Command Line
- [x] 28 — Ports and Sockets
- [x] 29 — Firewalls
- [x] 30 — curl and wget in Depth

### Phase 6 — Production Server Skills
- [x] 31 — Disk Management
- [x] 32 — User & Permission Management for Deployments
- [x] 33 — Environment & Process Management for Node.js
- [x] 34 — Performance Investigation
- [x] 35 — Security Basics

---

## How to Use This Curriculum

| Command | Action |
|---------|--------|
| `START` | Begin Topic 01 |
| `NEXT` | Move to the next topic (after exercises pass) |
| `PROGRESS` | Show checklist with completed topics ticked |
| `HINT` | Get one nudge for a stuck exercise |
| `REDO [topic#]` | Regenerate that topic's doc |

---

## Learning Path Dependencies

```
01 Architecture ──→ 02 Filesystem ──→ 03 Everything is a File ──→ 04 Inodes
                                              │
                                              ▼
                                    05 Permissions ──→ 06 Users & Groups
                                              │
                                              ▼
                              07 How Shell Works ──→ 08 Navigating ──→ 09 Files
                                              │
                                              ▼
                              10 Viewing Files ──→ 11 Text Processing ──→ 12 Pipes
                                              │
                                              ▼
                              13 Scripting ──→ 14 Env Vars ──→ 15 Productivity
                                              │
                                              ▼
                              16 Processes ──→ 17 Process Mgmt ──→ 18 Resources
                                              │
                                              ▼
                              19 File Descriptors ──→ 20 Cron
                                              │
                                              ▼
                              21 Packages ──→ 22 systemd ──→ 23 Logs
                                              │
                                              ▼
                              24 SSH ──→ 25 Compression
                                              │
                                              ▼
                              26 Network ──→ 27 DNS ──→ 28 Ports ──→ 29 Firewalls
                                              │
                                              ▼
                              30 curl/wget ──→ 31 Disk ──→ 32 Deploy Users
                                              │
                                              ▼
                              33 Node Prod ──→ 34 Performance ──→ 35 Security
```

---

*Total: 35 topics across 6 phases. Each topic = one deep-dive doc following the mandatory template.*
