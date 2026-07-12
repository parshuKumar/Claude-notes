# 21 — Package Management

## ELI5 — The Simple Analogy

Imagine you order a **flat-pack bookshelf** from a furniture store.

- **The box** is the package (a `.deb` file). It is not a bookshelf yet. It is a sealed container.
- **Inside the box** there are exactly two things: a **parts bag** (the actual wooden panels and screws — these are the *files*) and a **paper leaflet** (the name of the product, its version, the list of tools you need, and a set of instructions to run *before* and *after* assembly — this is the *metadata*).
- **The leaflet says "you also need a screwdriver and 4 wall anchors."** Those are **dependencies**.
- **The assembler guy** is `dpkg`. He is very good at exactly one thing: opening ONE box and putting the parts where they belong. If the leaflet says "you need a screwdriver" and you don't own a screwdriver, he **stops, throws his hands up, and leaves**. He will not go buy one. He does not know what a store is.
- **The store's logistics system** is `apt`. It reads the leaflet *before* anything arrives, works out the full shopping list (bookshelf + screwdriver + anchors + the box the anchors need), orders all of it from the **warehouse** (the repository), and then hands the boxes to the assembler **one at a time, in the correct order** so he never gets stuck.
- **The warehouse catalogue** is the package index. If your copy of the catalogue is from last year, you'll order item #4471 and the warehouse will say **"404 — we don't stock that anymore."** That is why you refresh the catalogue (`apt update`) before you order (`apt install`).
- **The wax seal on the catalogue** is the GPG signature. It proves the catalogue really came from the warehouse and nobody swapped in a page that says "the bookshelf now ships with a keylogger."

`dpkg` = the assembler. `apt` = the logistics system that keeps the assembler fed. That is the whole model.

---

## Where This Lives in the Linux Stack

```
Hardware (Disk, NIC)
  └── KERNEL (filesystem writes, network sockets, process creation)
       │      ▲ apt/dpkg are *heavy* syscall users: connect(), read(), write(),
       │      │ rename(), chmod(), unlink(), fork()+execve() for maintainer scripts
       └── System Calls
            │
            └── C Standard Library (glibc)
                 │
                 └── Shell (bash) ── invokes the tool, and dpkg *itself* invokes bash
                      │              to run preinst/postinst maintainer scripts
                      │
                      └── Commands: apt / apt-get / dpkg  ◀◀◀ THIS TOPIC
                                    │
                                    ├── apt   = HIGH level: dependency solver + HTTP downloader
                                    └── dpkg  = LOW level: unpacks ONE .deb onto the filesystem
```

Package management is **100% userspace**. The kernel has no idea what a "package" is. To the kernel, `apt install nginx` is just: some process opened a TCP socket, downloaded bytes, wrote files to disk, and forked a few shell scripts. Every guarantee a package manager gives you (versions, dependencies, "which package owns this file") is a **convention maintained in a database in `/var/lib/dpkg/`** — not a kernel feature.

---

## What Is This?

A **package** is an archive containing a program's files *plus* metadata describing what it is, what version it is, what else it needs, and what scripts to run when it is installed or removed. A **package manager** installs, upgrades, and removes those archives while keeping a database of what is currently on the system.

On Debian/Ubuntu there are **two layers**: `dpkg` (installs one `.deb`, no dependency resolution, no networking) and `apt` (resolves the dependency graph, downloads from remote repositories, then drives `dpkg`).

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `apt update` vs `apt upgrade` | Your Dockerfile builds fine for a month, then suddenly `E: Failed to fetch ... 404 Not Found` and CI is red |
| `dpkg` doesn't resolve deps | You `dpkg -i` a `.deb`, get "dependency problems", and think the package is broken |
| `remove` vs `purge` | You reinstall nginx to "reset it" and the broken config in `/etc/nginx` is still there |
| `signed-by=` vs `apt-key add` | You add a third-party repo that can now silently impersonate Ubuntu's own security updates |
| `apt` vs `apt-get` in scripts | Your deploy script breaks on a distro upgrade because apt's output format changed |
| `--no-install-recommends` | Your "slim" Docker image is 900 MB because installing `curl` dragged in 60 packages |
| The dpkg lock | You `rm` the lock file at 2am and corrupt the dpkg database, turning a 30-second wait into a rebuild |
| nvm vs system Node | Your app runs fine in SSH and dies instantly under systemd/cron with `node: command not found` |

---

## The Physical Reality — What a `.deb` Actually Is

A `.deb` is not a special format. It is an **`ar` archive** (the same ancient Unix archiver used for `.a` static libraries) containing exactly three members:

```
┌─────────────────────────────────────────────────────────────────────┐
│  nginx_1.18.0-6ubuntu14_amd64.deb   ← an `ar` archive               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. debian-binary        ← a 4-byte text file. Contains "2.0\n".    │
│                            The .deb format version. That's it.      │
│                                                                     │
│  2. control.tar.gz       ← THE METADATA (the leaflet)               │
│     ├── control            Package: nginx                           │
│     │                      Version: 1.18.0-6ubuntu14                │
│     │                      Architecture: amd64                      │
│     │                      Depends: libc6 (>= 2.34), nginx-core     │
│     │                      Maintainer: Ubuntu Developers <...>      │
│     │                      Description: small, powerful web server  │
│     ├── md5sums            checksum of every shipped file           │
│     ├── conffiles          list of files under /etc that are        │
│     │                      "config" — dpkg will NOT blindly         │
│     │                      overwrite these on upgrade               │
│     ├── preinst            ← shell script, runs BEFORE unpack       │
│     ├── postinst           ← shell script, runs AFTER unpack        │
│     ├── prerm              ← shell script, runs BEFORE removal      │
│     └── postrm             ← shell script, runs AFTER removal       │
│                                                                     │
│  3. data.tar.xz          ← THE ACTUAL FILES (the parts bag)         │
│     ./usr/sbin/nginx                                                │
│     ./etc/nginx/nginx.conf                                          │
│     ./lib/systemd/system/nginx.service                              │
│     ./usr/share/doc/nginx/changelog.Debian.gz                       │
│     ▲                                                               │
│     └── note the leading "./" — these are ABSOLUTE paths relative   │
│         to /. Installing = untarring data.tar.xz into /             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Installing a package is, at the syscall level, barely more than `tar -x` into `/` plus running four shell scripts and updating a text database.** That is genuinely all it is.

> **Compression note:** `data.tar` may be `.gz`, `.xz`, or `.zst`. Debian 12 defaults to **xz**; Ubuntu has shipped **zstd** (`data.tar.zst`) for its own packages since 21.10 because it decompresses much faster. Don't be surprised by the extension.

### The dpkg database — where "what's installed" actually lives

```
/var/lib/dpkg/
├── status              ← THE DATABASE. A plain-text file. Every installed
│                         package's control metadata, concatenated, separated
│                         by blank lines. `dpkg -l` just parses this.
├── status-old          ← backup of the previous version
├── available           ← legacy
├── lock                ← the low-level lock (dpkg holds this)
├── lock-frontend       ← the high-level lock (apt holds this)
└── info/
    ├── nginx.list      ← EVERY file this package installed, one per line
    ├── nginx.md5sums   ← checksums (so you can detect tampering)
    ├── nginx.conffiles
    ├── nginx.postinst  ← the maintainer scripts, saved for removal time
    └── nginx.prerm
```

`dpkg -L nginx` is literally `cat /var/lib/dpkg/info/nginx.list`.
`dpkg -S /usr/sbin/nginx` is literally `grep` across all `*.list` files.

**This is why the database matters more than the files.** If `/var/lib/dpkg/status` is corrupted, the files on disk are all still there — but the system no longer *knows* they are there. Package management is now broken and only a painful manual repair fixes it. (Remember this when you get to the lock-file mistake below.)

---

## How It Works — Step by Step

```
COMMAND: apt install nginx

STEP 1  apt reads /etc/apt/sources.list and /etc/apt/sources.list.d/*.list|*.sources
        → learns WHICH repositories exist.

STEP 2  apt reads the LOCAL INDEX in /var/lib/apt/lists/
        e.g. archive.ubuntu.com_ubuntu_dists_jammy_main_binary-amd64_Packages
        → a giant text file listing every package, version, dependency,
          filename, and SHA256 available in that repo.
        ⚠ THIS FILE IS A CACHED SNAPSHOT. `apt update` is what refreshes it.

STEP 3  DEPENDENCY RESOLUTION (this is the part dpkg cannot do)
        nginx Depends: nginx-core (>= 1.18.0)
          nginx-core Depends: libc6 (>= 2.34), libssl3, libpcre3, nginx-common
            libssl3 Depends: libc6
        → apt builds a graph, runs a SAT-style solver, and produces a
          TOPOLOGICALLY SORTED install order.

STEP 4  DOWNLOAD. apt forks /usr/lib/apt/methods/http (a separate binary!)
        which does socket() → connect() → GET /pool/main/n/nginx/nginx_...deb
        → files land in /var/cache/apt/archives/*.deb
        → each .deb's SHA256 is checked against the index.
        → the index itself was checked against the GPG signature in InRelease.
          THIS IS THE ENTIRE CHAIN OF TRUST:
              GPG key → signs InRelease → contains hash of Packages
              → Packages contains hash of the .deb → .deb verified.

STEP 5  apt calls dpkg, in order, once per package:
            dpkg --unpack libc6...deb
            dpkg --unpack nginx-common...deb
            dpkg --configure ...

STEP 6  For EACH .deb, dpkg does:
        a) run  preinst  (fork() + execve("/bin/sh", ["sh", "preinst", "install"]))
        b) extract data.tar.xz into /   → openat(O_CREAT), write(), chmod(), rename()
           (dpkg writes to a temp name then rename()s — rename() is ATOMIC, so a
            binary is never half-written. This is why you can upgrade a running app.)
        c) record every path into /var/lib/dpkg/info/<pkg>.list
        d) update /var/lib/dpkg/status  →  Status: install ok installed
        e) run  postinst  (this is where systemd units get enabled and services start)

STEP 7  apt releases the locks. Done.
```

**Now the key insight:** if you skip apt and run `dpkg -i nginx.deb` yourself, you jump straight to STEP 6. There is no STEP 1–4. No index, no network, no solver. dpkg reads `Depends:` from the control file, checks the database, sees `libssl3` is missing, and **stops with "dependency problems — leaving unconfigured."** It is not broken. It is doing exactly its job.

---

## Repositories — Where Packages Come From

### The classic one-line format (`/etc/apt/sources.list`)

```
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
│   │                                │     │
│   │                                │     └── COMPONENTS (one or more)
│   │                                └── SUITE / codename (the release pocket)
│   └── URI — the base URL of the repository
└── TYPE:  "deb" = binary packages.  "deb-src" = source packages (rarely needed;
           commenting these out makes `apt update` twice as fast)

With a signing key pinned (the MODERN, CORRECT form):

deb [signed-by=/usr/share/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main
    │
    └── ONLY this key may sign this repo. Nothing else on the system trusts it.
```

**Components** — Ubuntu splits its archive into four:

| Component | Meaning |
|---|---|
| `main` | Officially supported, free software. Canonical patches security bugs here. |
| `restricted` | Officially supported, **proprietary** (e.g. NVIDIA drivers). |
| `universe` | Community-maintained free software. **No guaranteed security support.** |
| `multiverse` | Non-free, unsupported. Legal restrictions may apply. |

**Pockets** — the same release has four separate suites:

| Suite | What it holds |
|---|---|
| `jammy` | The packages exactly as they shipped on release day. Frozen forever. |
| `jammy-security` | **Security patches only.** The one you must never disable. |
| `jammy-updates` | Bug fixes and non-security updates. |
| `jammy-backports` | Newer upstream versions backported. **Opt-in — not enabled by default for install.** |

So a real `sources.list` has four `deb` lines, one per pocket. When apt picks a version, it considers all of them and takes the highest-priority candidate.

### The new deb822 format (`/etc/apt/sources.list.d/*.sources`)

Ubuntu 24.04 ships this by default. Same information, structured:

```
Types: deb
URIs: https://deb.nodesource.com/node_20.x
Suites: nodistro
Components: main
Signed-By: /usr/share/keyrings/nodesource.gpg
```

Both formats work on Ubuntu 22.04. The `.sources` form is easier to script and can list multiple URIs/suites without repeating lines.

### GPG signing, and why `apt-key add` is dead

A repository serves an `InRelease` file: the hashes of every index file, **cryptographically signed**. `apt update` verifies that signature before trusting a single byte. Without it, anyone who can MITM your HTTP connection (or compromise a mirror) can hand you a `Packages` file pointing at a trojaned `openssh-server.deb` — and apt would install it as root without complaint.

```
┌──────────────────────────────────────────────────────────────────┐
│  THE OLD WAY — apt-key add  (DEPRECATED, removed in Ubuntu 24.04)│
├──────────────────────────────────────────────────────────────────┤
│  curl -s https://example.com/key.gpg | sudo apt-key add -        │
│                                                                  │
│  This appends the key to a GLOBAL trusted keyring.               │
│  RESULT: that key can now validly sign ANY repository —          │
│  including archive.ubuntu.com.                                   │
│                                                                  │
│  A sketchy third-party repo (or a compromised vendor key) can    │
│  now serve you a fake "libc6" or "openssh-server" and apt will   │
│  cheerfully install it as root. You gave a Docker-tutorial       │
│  vendor the power to impersonate Ubuntu Security.                │
└──────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────┐
│  THE MODERN WAY — one key, scoped to ONE repo, via signed-by=    │
├──────────────────────────────────────────────────────────────────┤
│  Key lives in /usr/share/keyrings/ (or /etc/apt/keyrings/)       │
│  and is referenced ONLY by the sources line that needs it.       │
│  That key is valid for that repo and NOTHING else.               │
└──────────────────────────────────────────────────────────────────┘
```

**The correct, current way to add the NodeSource repo (Node 20 on Ubuntu 22.04):**

```bash
# 1. Make sure we can fetch over HTTPS at all
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 2. Create the keyring directory (exists by default on 22.04+, harmless if it does)
sudo mkdir -p /etc/apt/keyrings

# 3. Download NodeSource's ASCII-armoured key and DEARMOR it into binary .gpg form
#    (--dearmor converts the -----BEGIN PGP PUBLIC KEY BLOCK----- text into the
#     binary keyring format that apt's signed-by= expects)
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

# 4. Write the sources line, PINNING that key to that repo only
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" \
  | sudo tee /etc/apt/sources.list.d/nodesource.list

# 5. Refresh the index (this is where the GPG signature gets verified)
sudo apt-get update

# 6. Install
sudo apt-get install -y nodejs

node --version   # v20.x.x
```

If step 5 prints `NO_PUBKEY` or `signature couldn't be verified`, the key is wrong or wasn't dearmoured — **do not "fix" it by adding `[trusted=yes]`.** That disables verification entirely, which is worse than `apt-key add`.

---

## `apt update` vs `apt upgrade` — Burn This Into Your Brain

This is the single most common Linux misunderstanding, and it costs people entire afternoons.

```
┌──────────────────────────────────────────────────────────────────────┐
│  apt update                                                          │
│  ────────────                                                        │
│  Downloads the CATALOGUE. Refreshes /var/lib/apt/lists/*.            │
│                                                                      │
│  CHANGES NOTHING ON YOUR SYSTEM. Not one installed package moves.    │
│  It only updates apt's knowledge of WHAT EXISTS and WHERE.           │
│                                                                      │
│  Output ends with:  "42 packages can be upgraded."   ← just INFO     │
└──────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────┐
│  apt upgrade                                                         │
│  ─────────────                                                       │
│  ACTUALLY INSTALLS newer versions of packages you already have,      │
│  using whatever the catalogue currently says.                        │
│                                                                      │
│  Will install NEW packages if needed as dependencies, but will       │
│  NEVER REMOVE an existing package. If an upgrade would require a     │
│  removal, it holds that package back ("The following packages have   │
│  been kept back:").                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

**Why you must ALWAYS `update` before `install`:**

The index in `/var/lib/apt/lists/` contains the exact **filename** of every `.deb` — including its version:
`pool/main/c/curl/curl_7.81.0-1ubuntu1.10_amd64.deb`

Repositories only keep the **current** version in the pool. When `7.81.0-1ubuntu1.15` is published, the `.10` file is **deleted from the mirror**. If your local index is stale, apt confidently requests the `.10` URL — and the server says:

```
E: Failed to fetch http://archive.ubuntu.com/ubuntu/pool/main/c/curl/curl_7.81.0-1ubuntu1.10_amd64.deb
   404  Not Found [IP: 185.125.190.36 80]
E: Unable to fetch some archives, maybe run apt-get update or use --fix-missing?
```

This is **not** a network problem. It is a **stale index** problem.

### The Docker layer-cache version of this bug

```dockerfile
# ❌ WRONG — this WILL break, and it will break weeks after you wrote it
FROM ubuntu:22.04
RUN apt-get update                     # ← layer A: cached forever by Docker
RUN apt-get install -y curl            # ← layer B: uses layer A's STALE index
```

Docker caches layer A. Three weeks later you change something below and rebuild: Docker reuses the *cached* layer A (an index from three weeks ago) and runs layer B against it. The `.deb` it names has been rotated off the mirror. **404.**

```dockerfile
# ✅ RIGHT — update and install in ONE layer, so they can never desynchronise
FROM ubuntu:22.04
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
#    │                              │
#    │                              └── delete the index we just downloaded — it's
#    │                                  ~40 MB of text that the image never needs
#    └── don't pull in "Recommends:" packages (docs, fonts, suggested extras)
```

`rm -rf /var/lib/apt/lists/*` in the **same RUN** is what actually shrinks the layer. Doing it in a *later* RUN does nothing — the bytes are already committed to the earlier layer.

---

## `apt` vs `apt-get` — Which One, When

| | `apt` | `apt-get` / `apt-cache` |
|---|---|---|
| Audience | **Humans, interactively** | **Scripts, Dockerfiles, CI** |
| Progress bar / colour | Yes | No |
| CLI stability | **Explicitly NOT guaranteed** | **Stable, guaranteed** |
| `apt list --upgradable` etc. | Yes | Split across `apt-cache`, `apt-mark` |
| Warns in pipes | `WARNING: apt does not have a stable CLI interface. Use with caution in scripts.` | Silent |

That warning is not decoration. The apt maintainers reserve the right to change apt's output format between releases. If your deploy script greps apt's output, it can break on a distro upgrade. **In scripts and Dockerfiles: `apt-get`. At a terminal: `apt`.**

---

## Exact Syntax Breakdown

```
sudo apt-get install -y --no-install-recommends nginx=1.18.0-6ubuntu14
│    │       │       │   │                      │     │
│    │       │       │   │                      │     └── exact version pin (optional)
│    │       │       │   │                      └── package name
│    │       │       │   └── skip "Recommends:" deps — installs ONLY hard "Depends:"
│    │       │       │       (this is the difference between a 90 MB and a 400 MB image)
│    │       │       └── assume yes to all prompts (mandatory in Docker/CI — without it
│    │       │           the build hangs forever waiting on stdin that doesn't exist)
│    │       └── the subcommand
│    └── the stable, script-safe front-end
└── package installs write to /usr and /etc — root required
```

```
dpkg -S /usr/bin/node
│    │  │
│    │  └── an absolute path (or a pattern)
│    └── --search : WHICH PACKAGE OWNS THIS FILE?
└── the low-level package tool

nodejs: /usr/bin/node        ← the package "nodejs" put that file there

# If it prints:
#   dpkg-query: no path found matching pattern /home/deploy/.nvm/.../bin/node
# ...then that node was NOT installed by a package. It's nvm. Remember this —
# it explains the systemd/cron failure at the bottom of this doc.
```

```
apt-cache policy nodejs
│         │      │
│         │      └── package name
│         └── show the version table and where each version comes from
└── the query half of apt-get

nodejs:
  Installed: 20.11.1-1nodesource1     ← what you HAVE
  Candidate: 20.12.0-1nodesource1     ← what apt WOULD install right now
  Version table:
     20.12.0-1nodesource1 500
        500 https://deb.nodesource.com/node_20.x nodistro/main amd64 Packages
 *** 20.11.1-1nodesource1 100         ← *** = currently installed
        100 /var/lib/dpkg/status      ← priority 100 = "installed, from the local db"
     12.22.9~dfsg-1ubuntu3 500
        500 http://archive.ubuntu.com/ubuntu jammy/universe amd64 Packages
                                       ▲
                                       └── Ubuntu's own ancient Node. THIS is what
                                           `apt install nodejs` gives you without
                                           the NodeSource repo.
```

**`apt-cache policy` is the single best debugging command in this topic.** It answers "what do I have, what would I get, and which repo is it coming from" in one shot.

```
dpkg -l | grep nginx
│    │
│    └── --list : list packages matching a pattern
└──
ii  nginx-common  1.18.0-6ubuntu14  all  small, powerful web server
││
││
│└── ACTUAL state:  i=installed  c=config-files-only  n=not-installed
│                   u=unpacked   f=half-configured    H=half-installed
└── DESIRED state:  i=install  h=hold  r=remove  p=purge  u=unknown

  "ii" = you want it installed, and it is. Normal.
  "rc" = you REMOVED it but its CONFIG FILES ARE STILL THERE.  ← the purge trap
  "iU" / "iF" = half-installed. Something died mid-transaction. Run `dpkg --configure -a`.
```

---

## Example 1 — Basic: Dissect a `.deb` With Your Own Hands

```bash
# Download a .deb WITHOUT installing it. Lands in the current directory.
apt-get download curl
# Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 curl amd64 7.81.0-1ubuntu1.15
ls
# curl_7.81.0-1ubuntu1.15_amd64.deb

# PROVE IT: a .deb is just an `ar` archive with 3 members
ar t curl_7.81.0-1ubuntu1.15_amd64.deb
# debian-binary
# control.tar.zst
# data.tar.zst

# PROVE IT: the first member is a 4-byte text file
ar p curl_7.81.0-1ubuntu1.15_amd64.deb debian-binary
# 2.0

# PROVE IT: control.tar holds the metadata (the "leaflet")
dpkg -I curl_7.81.0-1ubuntu1.15_amd64.deb        # -I = --info
#  Package: curl
#  Version: 7.81.0-1ubuntu1.15
#  Architecture: amd64
#  Depends: libc6 (>= 2.34), libcurl4 (= 7.81.0-1ubuntu1.15), zlib1g (>= 1:1.1.4)
#  Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>

# PROVE IT: data.tar holds the actual files, with absolute paths
dpkg -c curl_7.81.0-1ubuntu1.15_amd64.deb        # -c = --contents
# -rwxr-xr-x root/root  231208  ./usr/bin/curl
# drwxr-xr-x root/root       0  ./usr/share/man/man1/
# -rw-r--r-- root/root   28381  ./usr/share/man/man1/curl.1.gz
#                                ▲
#                                └── install = untar this into /

# Fully unpack it into a sandbox to see everything at once
mkdir -p /tmp/deb && dpkg-deb -R curl_*.deb /tmp/deb   # -R = --raw-extract
find /tmp/deb/DEBIAN -type f
# /tmp/deb/DEBIAN/control
# /tmp/deb/DEBIAN/md5sums
# /tmp/deb/DEBIAN/postinst    ← the maintainer script, a plain /bin/sh script
```

---

## Example 2 — Production Scenario

**Situation:** 2:14am. PagerDuty. Your Node API on `api-prod-01` is returning 502s. nginx is up; the Node app is dead. You SSH in.

```bash
$ systemctl status myapp
● myapp.service - Node API
     Active: failed (Result: exit-code) since Tue 2024-06-11 02:07:41 UTC; 6min ago
   Main PID: 21841 (code=exited, status=1/FAILURE)
Jun 11 02:07:41 api-prod-01 node[21841]: Error: The module '/srv/app/node_modules/better-sqlite3/build/Release/better_sqlite3.node'
Jun 11 02:07:41 api-prod-01 node[21841]: was compiled against a different Node.js version

# Somebody upgraded Node. Which version is on the box, and where did it come from?
$ node --version
v20.12.0

$ apt-cache policy nodejs
nodejs:
  Installed: 20.12.0-1nodesource1
  Candidate: 20.12.0-1nodesource1
  Version table:
 *** 20.12.0-1nodesource1 500
        500 https://deb.nodesource.com/node_20.x nodistro/main amd64 Packages
        100 /var/lib/dpkg/status

# When did that happen? dpkg keeps a full audit log.
$ grep " upgrade nodejs" /var/log/dpkg.log
2024-06-11 02:04:12 upgrade nodejs:amd64 20.11.1-1nodesource1 20.12.0-1nodesource1
#                                        ▲ from                ▲ to
# 02:04. Three minutes before the crash. That's our culprit.

# WHO did it? unattended-upgrades runs at night...
$ grep -A2 "nodejs" /var/log/unattended-upgrades/unattended-upgrades.log | tail -5
2024-06-11 02:04:09 Packages that will be upgraded: nodejs

# Confirmed: automatic security upgrade bumped Node's minor version, which changed
# the V8 module ABI, which invalidated a native addon compiled against the old one.

# ── FIX (60 seconds) ──────────────────────────────────────────────────
# Rebuild the native module against the Node that is now actually installed.
$ cd /srv/app && sudo -u deploy npm rebuild better-sqlite3
$ sudo systemctl restart myapp
$ systemctl is-active myapp
active                                      # 502s stop.

# ── PREVENT (the real fix) ────────────────────────────────────────────
# Pin Node so an unattended upgrade can never move it again without a human.
$ sudo apt-mark hold nodejs
nodejs set on hold.

$ apt-mark showhold
nodejs

# Verify the hold is real — apt now refuses to touch it:
$ sudo apt-get -s upgrade | grep -i nodejs
The following packages have been kept back:
  nodejs
```

**What you just did:** used `apt-cache policy` to find *where* the package came from, `/var/log/dpkg.log` to find *when* it changed, and `apt-mark hold` to make sure a machine never makes that decision for you again on a prod box.

---

## Common Mistakes

### Mistake 1 — `dpkg -i` and then declaring the package "broken"

**Wrong:**
```bash
$ sudo dpkg -i ./google-chrome-stable_current_amd64.deb
dpkg: dependency problems prevent configuration of google-chrome-stable:
 google-chrome-stable depends on libu2f-udev; however:
  Package libu2f-udev is not installed.
dpkg: error processing package google-chrome-stable (--install):
 dependency problems - leaving unconfigured
```
"This .deb is broken!" — No. dpkg **does not resolve dependencies**. It never has. That is apt's job.

**Root cause at the system level:** dpkg has now put the package into state `iU` (desired: install, actual: **unpacked but not configured**). The files are on disk. The `postinst` never ran. `/var/lib/dpkg/status` says `Status: install ok unpacked`. Your system is in a **half-transaction**.

**Right:**
```bash
# Let apt clean up the half-transaction by installing the missing deps
sudo apt-get -f install         # -f = --fix-broken

# Or, skip the whole dance: apt can install a LOCAL .deb and resolve deps itself
sudo apt-get install ./google-chrome-stable_current_amd64.deb
#                    ▲ the ./ is REQUIRED — without it apt looks for a package
#                      NAMED "google-chrome-stable_current_amd64.deb" in the repos
```

**Prevention:** never reach for `dpkg -i` on a `.deb` with dependencies. Use `apt install ./file.deb`.

---

### Mistake 2 — `apt remove` and expecting a clean slate

**Wrong:**
```bash
# nginx config is broken. "I'll just reinstall it."
sudo apt remove nginx
sudo apt install nginx
# ...the broken config is STILL THERE. Why?!
```

**Root cause:** `remove` deletes the files listed in `data.tar` **except** those listed in `conffiles` — everything under `/etc` that the maintainer marked as configuration. dpkg **deliberately preserves your edits**, because wiping an admin's config on every upgrade would be catastrophic. The package now shows as:

```bash
$ dpkg -l nginx-common
rc  nginx-common  1.18.0-6ubuntu14  all
││
│└── c = only CONFIG FILES remain
└── r = you asked for it to be removed
```

Reinstalling sees the conffiles already present, respects them, and leaves your broken config exactly where it was.

**Right:**
```bash
sudo apt purge nginx nginx-common     # purge = remove + DELETE the conffiles
sudo apt autoremove --purge           # and drop the orphaned dependencies too
sudo apt install nginx                # now you truly get a factory-fresh /etc/nginx

# Find every package sitting in the "rc" ghost state:
dpkg -l | awk '/^rc/ {print $2}'
```

---

### Mistake 3 — `rm`-ing the dpkg lock file

**Wrong:**
```bash
$ sudo apt install htop
E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 3218 (unattended-upgr)
E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?

# The internet says:
$ sudo rm /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock     # ☠️ NO
```

**Root cause:** that lock is not stale garbage. It is held by a **live process** that is *right now* in the middle of a dpkg transaction — halfway through unpacking `data.tar` into `/`, with `/var/lib/dpkg/status` in an inconsistent intermediate state. Deleting the lock lets a **second** dpkg start writing the same database file concurrently. Two writers, one text database, non-atomic updates. The result is a `status` file with truncated or interleaved records.

**What that failure actually looks like:**
```
E: Sub-process /usr/bin/dpkg returned an error code (2)
dpkg: unrecoverable fatal error, aborting:
 parsing file '/var/lib/dpkg/status' near line 74821 package 'libssl3':
 field name 'Descriptio' must be followed by a colon
```
Now **nothing** can be installed or removed until you hand-repair the database from `/var/lib/dpkg/status-old`. You have turned a 90-second wait into a two-hour recovery.

**Right:**
```bash
# 1. WHO holds it? (fuser prints the PID holding the file open)
sudo fuser -v /var/lib/dpkg/lock-frontend
#                      USER   PID  ACCESS COMMAND
# /var/lib/dpkg/lock-frontend:
#                      root  3218  F....  unattended-upgr

ps -p 3218 -o pid,etime,cmd
#   PID     ELAPSED CMD
#  3218       01:12 /usr/bin/python3 /usr/bin/unattended-upgrade --download-only

# 2. It's the nightly security updater. WAIT. It finishes in a minute or two.
#    Watch it go away:
while sudo fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 2; done; echo "lock free"

# 3. ONLY if a process is genuinely dead and the lock is orphaned (rare — verify
#    with fuser returning NOTHING) is it safe to clear state. And even then:
sudo dpkg --configure -a      # finish any half-done transaction FIRST
```

---

### Mistake 4 — Using `apt-key add` for a third-party repo

**Wrong:**
```bash
curl -sL https://some-vendor.example/gpg | sudo apt-key add -
echo "deb https://some-vendor.example/apt stable main" | sudo tee /etc/apt/sources.list.d/vendor.list
```

**Root cause:** `apt-key add` writes the key into `/etc/apt/trusted.gpg` (or drops it in `/etc/apt/trusted.gpg.d/`), which is the **global** trust store. apt does not remember *which repo* a key was added for. From that moment on, **that vendor's key is a valid signer for `archive.ubuntu.com` too**. If that vendor is ever compromised — or is simply malicious — they can serve you a modified `Packages` index for Ubuntu's own `main` and hand you a backdoored `openssh-server`, installed as root, with a valid signature. You have granted a random Docker blog post the authority of Ubuntu Security.

Ubuntu 22.04 warns loudly (`Warning: apt-key is deprecated`); Ubuntu 24.04 **removed the command**.

**Right:** scope the key with `signed-by=`, as shown in the NodeSource section above. One key, one repo. And **never** `[trusted=yes]` — that turns verification off entirely.

**Audit what you already trust globally:**
```bash
sudo apt-key list 2>/dev/null          # legacy global keys — should ideally be empty
ls -la /etc/apt/trusted.gpg.d/         # global keyrings — Ubuntu's own belong here
ls -la /usr/share/keyrings/ /etc/apt/keyrings/   # correctly SCOPED keys live here
grep -r "signed-by" /etc/apt/sources.list.d/     # which repos are properly pinned?
```

---

### Mistake 5 — Installing Node with nvm, then wondering why systemd and cron can't find it

**Wrong:**
```bash
# On the prod server, as user `deploy`:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
node --version    # v20.12.0 — works!
# ...then the systemd unit says: node: command not found
# ...and the cron job silently does nothing, forever.
```

**Root cause — and this is the trap that catches everyone:**

`nvm` is **not a program**. It is a **bash function** defined by `~/.nvm/nvm.sh`, which is sourced from `~/.bashrc`. It works by *rewriting `PATH`* inside your interactive shell.

```
When you SSH in:              login shell → reads ~/.bashrc → sources nvm.sh
                              → PATH gains /home/deploy/.nvm/versions/node/v20.12.0/bin
                              → `node` resolves. 

When systemd starts a unit:   systemd fork()s and execve()s DIRECTLY.
                              NO shell. NO ~/.bashrc. NO nvm.sh.
                              PATH is a minimal default: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
                              → `node` DOES NOT EXIST on that PATH. 

When cron runs a job:         cron uses PATH=/usr/bin:/bin. Same story. 
```

(This is exactly the environment-inheritance rule from **14 — Environment Variables**: non-interactive, non-login contexts do **not** read `.bashrc`. And **20 — Cron** and **22 — systemd** both bite you with it.)

**Proof:**
```bash
$ which node
/home/deploy/.nvm/versions/node/v20.12.0/bin/node

$ dpkg -S $(which node)
dpkg-query: no path found matching pattern /home/deploy/.nvm/versions/node/v20.12.0/bin/node
#           ▲ no package owns it. It is invisible to the system package manager,
#             and it lives in a HOME DIRECTORY — unreadable to other service users.

$ sudo env -i /home/deploy/.nvm/versions/node/v20.12.0/bin/node -v   # absolute path works
v20.12.0
$ sudo env -i node -v                                                # bare name does not
env: 'node': No such file or directory
```

**Right (pick one):**
- **For servers:** install Node **system-wide** from the NodeSource repo → binary at `/usr/bin/node`, on the default PATH, owned by a package, upgradeable with apt, visible to systemd and cron. This is the correct production answer.
- **If you must keep nvm:** use the **absolute path** everywhere — `ExecStart=/home/deploy/.nvm/versions/node/v20.12.0/bin/node /srv/app/server.js`. It works, but it hard-codes a version string that changes every time you `nvm install`, and your unit file silently breaks on the next upgrade.

---

## Installing Node.js Properly — An Honest Comparison

| Method | Where it lands | Visible to systemd/cron? | Multiple versions? | Verdict |
|---|---|---|---|---|
| `apt install nodejs` (Ubuntu repo) | `/usr/bin/node` | ✅ Yes | ❌ No | **Version is often ancient.** Ubuntu 22.04 ships **Node 12.22** in `universe` — EOL since 2022. Fine for a build dependency; wrong for running your app. |
| **NodeSource repo** | `/usr/bin/node` | ✅ Yes | ❌ One at a time | **The right answer for a production server.** Real apt package, real security updates, `apt-mark hold`-able. |
| **nvm** | `~/.nvm/versions/node/vX/bin/node` | ❌ **NO** — it's a bash function in `.bashrc` | ✅ Yes, instantly | **The right answer for local dev. A trap on servers.** |
| **`n`** | `/usr/local/bin/node` | ✅ Yes | ✅ Switches the system binary | A middle ground. Overwrites `/usr/local/bin` outside dpkg's knowledge — `dpkg -S` won't find it. |
| **Docker `node:20-slim`** | `/usr/local/bin/node` in the image | N/A | Per-image | If you're containerised, this is the real answer — pin the version in the tag. |

```bash
# Check what Ubuntu would actually give you BEFORE you install it:
apt-cache policy nodejs
#   Candidate: 12.22.9~dfsg-1ubuntu3     ← on Ubuntu 22.04. Node 12. From 2019.
```

---

## Other Things Worth Knowing

**Version strings.** `1:2.4.52-1ubuntu4.6`

```
1  :  2.4.52  -  1ubuntu4.6
│     │          │
│     │          └── DEBIAN REVISION — the packaging changed, upstream did not
│     └── UPSTREAM VERSION — the version the software's authors released
└── EPOCH (rare) — a manual override. "Trust me, this is newer."
    Needed when upstream RENUMBERS backwards (e.g. goes from 2020.03 to 1.0).
    An epoch of 1 beats a missing epoch (which means 0) no matter what follows.
```
apt compares these component-wise, with digits compared numerically and letters lexically — so **1.10 > 1.9** (unlike a string sort). Prove it:
```bash
dpkg --compare-versions 1.10 gt 1.9 && echo "1.10 is newer"   # prints
dpkg --compare-versions 1:1.0 gt 99.0 && echo "epoch wins"     # prints
```

**`apt full-upgrade` / `apt-get dist-upgrade`.** Unlike `upgrade`, it is **allowed to REMOVE packages** to satisfy a dependency change. On a laptop, that's usually fine. On a prod box, it can silently remove a package your app depends on. Always run `apt-get -s dist-upgrade` (`-s` = simulate) and **read the "The following packages will be REMOVED" list** before committing.

**`apt clean` — the 2am disk-full fix.** Every `.deb` apt downloads is kept in `/var/cache/apt/archives/`. On a long-lived server that quietly grows to gigabytes.
```bash
du -sh /var/cache/apt/archives/     # 3.1G
sudo apt clean                      # deletes all cached .debs — 100% safe, they're re-downloadable
sudo apt autoclean                  # gentler: only deletes .debs no longer in any repo
```

**`unattended-upgrades`.** Ubuntu enables automatic **security** updates by default. Config in `/etc/apt/apt.conf.d/50unattended-upgrades` and `20auto-upgrades`. It is the right default — but note two risks: it can **restart services** mid-traffic (`Unattended-Upgrade::Automatic-Reboot "false";` — verify this is false on prod), and it can bump a runtime's minor version and break a native module (see Example 2). Mitigation: `apt-mark hold` your runtime, and read `/var/log/unattended-upgrades/`.

**`update-alternatives`.** When several packages provide the same command (`python`, `editor`, `java`), a symlink farm in `/etc/alternatives/` decides which one wins.
```bash
ls -l /usr/bin/editor            # → /etc/alternatives/editor → /usr/bin/vim.basic
sudo update-alternatives --config editor
sudo update-alternatives --display java
```

**snap vs apt.** Snaps are self-contained, sandboxed, auto-updating bundles with their own runtime. They ignore your apt pins, auto-update on their own schedule (you cannot fully disable it), and mount a loop device per snap (`df -h` on Ubuntu is full of `/dev/loop*` — that's snaps). **On a server, prefer apt.** You want a package manager whose upgrades happen when *you* say so.

**Alpine (`apk`) — for Docker.**
```dockerfile
RUN apk add --no-cache curl tini
#          │
#          └── --no-cache = don't write the index to disk at all.
#              It is the Alpine equivalent of `apt-get update && ... && rm -rf /var/lib/apt/lists/*`,
#              collapsed into one flag. There is NO separate "apk update" step needed.
```
Alpine uses **musl**, not glibc (see **01 — Architecture**). Native npm modules with prebuilt glibc binaries will not load; they get recompiled from source, which is why `node:20-alpine` builds need `apk add --no-cache python3 make g++`.

---

## Hands-On Proof

```bash
# PROVE IT: `apt update` changes NOTHING except the index timestamps
ls -la --time-style=full-iso /var/lib/apt/lists/ | head -3
sudo apt-get update
ls -la --time-style=full-iso /var/lib/apt/lists/ | head -3   # ← timestamps moved
dpkg -l | wc -l                                              # ← count is IDENTICAL

# PROVE IT: the package index is just a giant text file
grep -A6 "^Package: curl$" /var/lib/apt/lists/*_jammy_main_binary-amd64_Packages | head -10
# Package: curl
# Architecture: amd64
# Version: 7.81.0-1ubuntu1
# Depends: libc6 (>= 2.34), libcurl4 (= 7.81.0-1ubuntu1), zlib1g (>= 1:1.1.4)
# Filename: pool/main/c/curl/curl_7.81.0-1ubuntu1_amd64.deb   ← the URL apt will fetch

# PROVE IT: the dpkg "database" is a plain text file you can read
grep -A5 "^Package: curl$" /var/lib/dpkg/status

# PROVE IT: dpkg knows every file it ever installed
dpkg -L curl | head
wc -l /var/lib/dpkg/info/curl.list       # ← dpkg -L just prints THIS FILE

# PROVE IT: reverse lookup — which package owns an arbitrary file?
dpkg -S /bin/ls        # coreutils: /bin/ls
dpkg -S $(which node)  # nodejs: /usr/bin/node   — OR "no path found" if it's nvm

# PROVE IT: apt fetches over the network; watch it happen
sudo strace -f -e trace=connect -o /tmp/apt.trace apt-get -s install curl >/dev/null 2>&1
grep -c connect /tmp/apt.trace

# PROVE IT: a simulated install shows the dependency graph without touching anything
apt-get -s install nginx        # -s = --simulate. Safe on ANY box, including prod.
# Inst nginx-common (1.18.0-6ubuntu14 Ubuntu:22.04/jammy [all])
# Inst nginx-core (...)
# Inst nginx (...)
# Conf nginx-common (...)      ← note: unpack ALL, then configure ALL

# PROVE IT: verify installed files haven't been tampered with (md5sums from control.tar)
sudo debsums -c        # apt install debsums first. Prints any file that CHANGED on disk.

# PROVE IT: every dpkg action is logged forever
tail -5 /var/log/dpkg.log
grep " install " /var/log/dpkg.log | tail -3
```

---

## Practice Exercises

### Exercise 1 — Easy

In a terminal (use a Docker container if you're on macOS: `docker run -it --rm ubuntu:22.04 bash`):

```bash
apt-get update
apt-get download curl
ar t curl_*.deb
dpkg -I curl_*.deb
dpkg -c curl_*.deb | head -20
```

**Answer from your output:** What are the three members of the `ar` archive? What does `curl` list under `Depends:`? What is the very first file it would install into `/`?

---

### Exercise 2 — Medium

```bash
# 1. Install nginx, then answer, using ONLY dpkg:
#      - How many files did the `nginx-common` package install?
#      - Which package owns /etc/nginx/nginx.conf ?
#      - Which package owns /usr/bin/env ?
# 2. Now edit /etc/nginx/nginx.conf (add a comment line at the top).
# 3. Run `apt remove nginx-common`, then check `dpkg -l | grep nginx-common`.
#    What is the two-letter state code? Does your edited nginx.conf still exist?
# 4. Now `apt purge nginx-common`. Is it gone?
```

**Question:** Explain, in terms of the `conffiles` list inside `control.tar.gz`, exactly why step 3 left the file behind.

---

### Exercise 3 — Hard (Production Simulation)

You are handed a fresh Ubuntu 22.04 box. Requirements:

1. Install **Node 20** system-wide from NodeSource, using `signed-by=` correctly. Do **not** use `apt-key`.
2. Prove the key is scoped: show that the sources line references your keyring, and that `/etc/apt/trusted.gpg.d/` did **not** gain a new file.
3. Prove Node came from a package: `dpkg -S $(which node)` must name a package.
4. Pin it so unattended-upgrades can never move it. Prove the pin works with a **simulated** upgrade.
5. Write a `Dockerfile` that installs `curl` and `ca-certificates` on `ubuntu:22.04` in a **single RUN layer**, with `--no-install-recommends` and `rm -rf /var/lib/apt/lists/*`. Build it, then compare `docker image ls` against a naive 3-RUN-layer version. Report the size difference.
6. Finally, break it on purpose: `sudo dpkg -i` a `.deb` with unmet dependencies, show the `iU` state in `dpkg -l`, then repair the system with `apt --fix-broken install` and show it back at `ii`.

Paste every command and its real output.

---

## Mental Model Checkpoint

1. **What are the three members inside a `.deb` `ar` archive, and what is in each?**
2. **`dpkg -i app.deb` prints "dependency problems." Is the package broken? What is dpkg's actual job, and what does apt do that dpkg does not?**
3. **What EXACTLY does `apt update` change on your system? Why does skipping it cause a 404 on a `.deb` URL?**
4. **Why must `apt-get update` and `apt-get install` be in the SAME `RUN` layer in a Dockerfile?**
5. **You removed a package but its `/etc` config is still there and `dpkg -l` shows `rc`. Why did dpkg keep it, and what command actually deletes it?**
6. **Why is `apt-key add` a security hole? What does `signed-by=` fix?**
7. **You get "Could not get lock /var/lib/dpkg/lock-frontend." What is the correct response, and what specifically breaks if you `rm` the lock?**
8. **You installed Node with nvm and it works over SSH. Why can't systemd or cron find it?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `apt update` | Refresh the package **index**. Changes nothing installed. | — |
| `apt upgrade` | Install newer versions. Never removes packages. | `-y` |
| `apt full-upgrade` | Upgrade, **allowed to remove** packages | `-s` (simulate — always do this on prod) |
| `apt install PKG` | Resolve deps, download, install | `-y`, `--no-install-recommends`, `PKG=version`, `./local.deb` |
| `apt remove PKG` | Remove files, **keep config in /etc** | — |
| `apt purge PKG` | Remove **including** config files | — |
| `apt autoremove` | Delete orphaned dependencies | `--purge` |
| `apt search` / `apt show` | Find / describe a package | — |
| `apt list --installed` | What's installed | `--upgradable` |
| **`apt-cache policy PKG`** | **Installed vs candidate version + which repo** | the best debug command |
| `apt-mark hold` / `unhold` | Pin a package against upgrades | `showhold` |
| `apt clean` | Empty `/var/cache/apt/archives` (disk-full fix) | `autoclean` (gentler) |
| `apt --fix-broken install` | Repair a half-done dpkg transaction | `-f` |
| `dpkg -i FILE.deb` | Install ONE .deb. **No dep resolution.** | — |
| `dpkg -l` | List packages + state codes (`ii`, `rc`, `iU`) | — |
| **`dpkg -L PKG`** | **What files did this package install?** | — |
| **`dpkg -S /path`** | **Which package owns this file?** | — |
| `dpkg -s PKG` | Show a package's status/metadata | — |
| `dpkg -I FILE.deb` | Metadata of an uninstalled `.deb` | `-c` (contents) |
| `dpkg --configure -a` | Finish all half-configured packages | — |
| `dpkg -r` / `-P` | Remove / Purge (low-level) | — |
| `dpkg --compare-versions` | Compare two version strings | `lt`, `gt`, `eq` |
| `update-alternatives` | Pick between competing providers | `--config`, `--display` |
| `apk add --no-cache` | Alpine install (Docker) | — |

---

## When Would I Use This at Work?

### Scenario 1: "Where did this binary come from?"
A prod box has a `node` you didn't install. `dpkg -S $(which node)` says `nodejs: /usr/bin/node` — good, it's a real package. `apt-cache policy nodejs` says it came from `deb.nodesource.com`. Now you know it's upgradeable with apt, is covered by `apt-mark hold`, and is visible to systemd. If `dpkg -S` had said "no path found," you'd know it was nvm or a manual tarball — invisible to the package manager, and about to break your systemd unit (topic 22).

### Scenario 2: Disk full at 2am, and the app can't write its logs
`df -h` shows `/` at 100%. `du -sh /var/* | sort -h` shows `/var/cache/apt/archives` at 3.1 GB. `sudo apt clean` reclaims it instantly with zero risk — every file there is a `.deb` that can be re-downloaded. That buys you the headroom to actually fix the real cause (topics 23 and 31).

### Scenario 3: Your Docker image is 1.2 GB and CI is timing out
You add `--no-install-recommends` (drops docs, fonts, and "suggested" extras), collapse three `RUN`s into one, and append `rm -rf /var/lib/apt/lists/*` **in the same layer**. Image drops to 180 MB. As a bonus, you've also permanently fixed the intermittent `404 Not Found` from the cached `apt-get update` layer.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | Distributions differ *precisely* in their package manager: apt/dpkg vs apk vs yum. Same kernel, different userspace. |
| **Builds on** | 02 — The Filesystem Hierarchy | `data.tar` unpacks into `/usr`, `/etc`, `/var`. Understanding *why* configs are in `/etc` is what makes `remove` vs `purge` obvious. |
| **Builds on** | 06 — Users and Groups | Package `postinst` scripts create system users (`www-data`, `postgres`). That's where those accounts come from. |
| **Builds on** | 14 — Environment Variables | The nvm trap is a PATH-inheritance trap. `.bashrc` is not read by non-interactive processes. |
| **Builds on** | 20 — Cron and Scheduled Tasks | Same nvm/PATH trap, different victim. |
| **Next** | 22 — Services and systemd | A package's `postinst` drops a `.service` file into `/lib/systemd/system` and enables it. Now you'll learn what that file *is* — and write one for your Node app. |
| **Used by** | 23 — Logs | `/var/log/dpkg.log` and `/var/log/unattended-upgrades/` are your audit trail for "what changed on this box, and when." |
| **Used by** | 33 — Node in Production | Choosing NodeSource over nvm on a server is a *package management* decision with direct systemd consequences. |
| **Used by** | 35 — Security Basics | `unattended-upgrades`, GPG trust scoping, and `apt list --upgradable` are the core of server patching. |
