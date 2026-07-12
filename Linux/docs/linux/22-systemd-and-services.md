# 22 — Services and systemd

## ELI5 — The Simple Analogy

Think of a large apartment building with a **live-in superintendent**.

- He is the **first person hired** when the building opens. Everyone else — every tenant, every contractor — is let in by him. **He is employee #1.** (systemd is **PID 1**.)
- He doesn't just flip switches in a fixed order. He has a **wall of work orders** telling him what depends on what: the water pump must run before the boiler; the boiler before the radiators. Anything with no dependency between them he starts **at the same time**, in parallel — he doesn't stand around waiting.
- If the boiler dies at 3am, he **notices and relights it**. He doesn't wait for a tenant to complain. (Supervision + `Restart=`.)
- Every tenant's belongings are tracked **per apartment**. A tenant can't sneak a couch into a different flat and pretend it isn't theirs. When a tenant is evicted, *everything* in that apartment goes — including things they tried to disown. (**cgroups**: a service's child processes cannot escape.)
- He keeps a single **logbook** for the whole building. Every contractor's notes go in it, timestamped and searchable, instead of a hundred scraps of paper in a hundred drawers. (The **journal**.)
- The work orders come in **two binders**. The **manufacturer's binder** (`/lib/systemd/system`) gets replaced wholesale every time head office ships an update — **never scribble in it, your notes get shredded.** The **super's own binder** (`/etc/systemd/system`) is his, it always wins, and head office never touches it.
- And the thing that trips up every new super: **he works from memory.** Change a work order on the wall and he'll keep doing yesterday's version until you tell him **"go re-read the binder"** (`systemctl daemon-reload`).

That's systemd. Employee #1, a dependency graph, a supervisor, a cgroup accountant, and a logbook.

---

## Where This Lives in the Linux Stack

```
Hardware
  └── KERNEL
       │   ├─ boots, mounts the root filesystem
       │   ├─ creates PID 1 and execve()s /sbin/init  ──┐
       │   └─ provides CGROUPS (the kernel feature      │
       │      systemd uses to cage each service)        │
       │                                                │
       └── System Calls (fork, execve, kill, cgroup fs writes, sd_notify over a unix socket)
            │                                           │
            │    ┌──────────────────────────────────────┘
            │    ▼
            │  PID 1: systemd  ◀◀◀ THIS TOPIC
            │    │   The FIRST userspace process. The ancestor of everything.
            │    │   The REAPER of orphans (topic 16).
            │    │   If it dies, the kernel panics.
            │    │
            │    ├── sshd.service        ┐
            │    ├── nginx.service       │ every long-running process on the box
            │    ├── myapp.service       │ is a CHILD of PID 1, in its own cgroup
            │    └── cron.service        ┘
            │
            └── C Library (glibc)
                 └── Shell (bash)  ← systemd does NOT use one to start services!
                      └── Commands: systemctl, journalctl  ◀◀◀ THIS TOPIC (the client side)
```

**`systemctl` is not systemd.** `systemctl` is a small client program that talks to PID 1 over a **D-Bus / private socket**. When you type `systemctl restart myapp`, you are sending a *message* to PID 1 and asking *it* to do the work. That's why `systemctl` needs no privileges of its own to *read* state, but needs root (polkit) to *change* it.

---

## What Is This?

**systemd** is the **init system** — the first userspace process the kernel starts (PID 1), and the thing responsible for bringing the rest of the system up, keeping it up, and shutting it down. A **unit** is a declarative text file describing something systemd manages (a service, a socket, a timer, a mount point).

A **service** is a long-running background program (a daemon) — nginx, PostgreSQL, sshd, or **your Node.js app**. Writing a `.service` file is how you tell Linux: *"this program is important; start it at boot, restart it if it dies, run it as this user, and capture its logs."*

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `enable` vs `start` | You fix prod at 11pm, everything works, the box reboots at 4am, and **your app never comes back** |
| `daemon-reload` | You edit the unit, restart, and swear the change "didn't do anything" — because systemd used the cached old version |
| ExecStart needs an absolute path | `status=203/EXEC`, `node: command not found` — even though `node` works fine when you SSH in |
| `Type=` | You set `Type=forking` for a Node app; systemd hangs 90 seconds, then marks a perfectly healthy service `failed` |
| `TimeoutStopSec` + SIGTERM | Every deploy drops in-flight requests, because your app is SIGKILLed mid-response |
| `LimitNOFILE` | `EMFILE: too many open files` under load — and editing `/etc/security/limits.conf` **does nothing**, because that file does not apply to systemd services |
| `StartLimitBurst` | A crash-looping app gets permanently blocked with *"start request repeated too quickly"* and won't start even after you fix the bug |
| `journalctl -b -1` | The box rebooted unexpectedly and you have **no idea why** — you're looking at logs from *after* the reboot |
| Persistent journal | You reboot to "fix" it, and every log line proving what happened is **gone forever** (RAM-only journal) |

---

## The Physical Reality

### PID 1, and the cgroup cage

```
$ ps -p 1 -o pid,ppid,comm,args
  PID  PPID COMMAND         COMMAND
    1     0 systemd         /sbin/init splash
                            ▲
                            └── /sbin/init is a SYMLINK to /lib/systemd/systemd

$ ls -l /sbin/init
lrwxrwxrwx 1 root root 20 Apr 18 10:22 /sbin/init -> /lib/systemd/systemd
```

PPID **0** — nothing created it but the kernel itself. This is the process the kernel `execve()`s the moment the root filesystem is mounted. It never exits. **If PID 1 dies, the kernel panics** (`Kernel panic - not syncing: Attempted to kill init!`).

Because it is the ancestor of everything, it is also the **orphan reaper** from **16 — Processes in Depth**: when a parent dies before its child, the kernel re-parents the child to PID 1, and systemd `wait()`s on it so it never becomes a permanent zombie.

Now — the part that makes systemd fundamentally different from the old SysV init:

```
┌───────────────────────────────────────────────────────────────────┐
│  THE OLD WORLD (SysV init)                                        │
│                                                                   │
│  /etc/init.d/myapp start   →  a SHELL SCRIPT you wrote            │
│                            →  it backgrounds the daemon (fork/fork)│
│                            →  it writes a PID to /var/run/myapp.pid│
│                                                                   │
│  init's only handle on your process is that PID FILE.             │
│  DOUBLE-FORK: the daemon forks twice and its parent exits, so its │
│  PPID becomes 1. init has NO IDEA which processes belong to your  │
│  service. Stale PID file? Wrong PID? Now `stop` kills the wrong   │
│  process, or nothing at all. Child workers? Orphaned forever.     │
└───────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────────────────┐
│  THE SYSTEMD WORLD                                                │
│                                                                   │
│  Before exec'ing your process, systemd creates a KERNEL CGROUP:   │
│    /sys/fs/cgroup/system.slice/myapp.service/                     │
│  and puts your PID in it. Every process your app forks INHERITS   │
│  the cgroup. There is NO WAY OUT — it is enforced by the kernel.  │
│                                                                   │
│  $ systemd-cgls -u myapp.service                                  │
│  Unit myapp.service:                                              │
│  ├─1841 /usr/bin/node /srv/app/server.js                          │
│  ├─1866 /usr/bin/node /srv/app/worker.js      ← a child           │
│  └─1902 /bin/sh -c "imagemagick convert ..."  ← a grandchild      │
│                                                                   │
│  `systemctl stop myapp` signals THE WHOLE CGROUP. Double-forking  │
│  does not save you. Nothing leaks. Nothing is orphaned.           │
│  Resource limits (MemoryMax=, CPUQuota=) apply to the whole cage. │
│  Docker uses THE SAME KERNEL FEATURE. This is not a coincidence.  │
└───────────────────────────────────────────────────────────────────┘
```

### Where units live — and the override precedence

```
  LOWEST PRECEDENCE
        │
        │  /lib/systemd/system/          (= /usr/lib/systemd/system on some distros)
        │  ──────────────────────────────────────────────────────────────
        │  VENDOR / PACKAGE units. Shipped by `apt install nginx` (topic 21 —
        │  it's just a file in the .deb's data.tar).
        │  ⚠ DO NOT EDIT. The next `apt upgrade` overwrites it and your change
        │    is gone with zero warning.
        │
        │  /run/systemd/system/
        │  ──────────────────────────────────────────────────────────────
        │  RUNTIME units. tmpfs — vanishes on reboot. Rarely touched by hand.
        │
        ▼  /etc/systemd/system/
  HIGHEST  ──────────────────────────────────────────────────────────────
           THE ADMIN'S UNITS. Yours. Highest precedence. Survives upgrades.
           ✅ YOUR OWN units go here:  /etc/systemd/system/myapp.service
           ✅ Overrides of vendor units go here too, as DROP-INS:
                /etc/systemd/system/nginx.service.d/override.conf
```

**A file in `/etc/systemd/system/nginx.service` completely replaces** the vendor's `/lib/systemd/system/nginx.service`. That's usually a mistake — you've now forked the whole unit and you won't get the vendor's future fixes.

**A drop-in is the correct tool.** It *merges* on top of the vendor unit, so you change one directive and inherit everything else:

```bash
sudo systemctl edit nginx        # opens an editor, creates the drop-in for you
# → writes /etc/systemd/system/nginx.service.d/override.conf

sudo systemctl edit --full nginx # ⚠ forks the ENTIRE unit into /etc. Rarely what you want.

systemctl cat nginx              # shows the EFFECTIVE, MERGED unit — vendor + all drop-ins
# # /lib/systemd/system/nginx.service
# [Service]
# Type=forking
# ...
# # /etc/systemd/system/nginx.service.d/override.conf   ← your drop-in, shown appended
# [Service]
# LimitNOFILE=65535
```

> **Drop-in gotcha:** for most directives, the drop-in value *replaces* the vendor's. But for **list-type** directives (`ExecStart=`, `Environment=`, `After=`…), a drop-in **appends**. To *replace* an `ExecStart=`, you must first clear it with an empty assignment:
> ```ini
> [Service]
> ExecStart=
> ExecStart=/usr/bin/node /srv/app/server.js --flag
> ```
> Forgetting the blank `ExecStart=` gives you `Service has more than one ExecStart= setting, which is only allowed for Type=oneshot`.

---

## How It Works — Step by Step

```
COMMAND: sudo systemctl start myapp

STEP 1  systemctl connects to PID 1 over /run/systemd/private (a unix socket)
        and sends: StartUnit("myapp.service", "replace")

STEP 2  systemd loads myapp.service from its IN-MEMORY unit cache.
        ⚠ NOT from disk. From the cache built at the last daemon-reload.
        ⚠ THIS IS WHY YOU MUST RUN `daemon-reload` AFTER EDITING A UNIT FILE.

STEP 3  DEPENDENCY RESOLUTION. It walks Requires=/Wants=/After=/Before= and
        queues any needed units first (e.g. network.target).

STEP 4  systemd creates the CGROUP:
            mkdir /sys/fs/cgroup/system.slice/myapp.service
            echo 65535 > .../pids.max          (from TasksMax=)
            echo 512M  > .../memory.max        (from MemoryMax=)

STEP 5  fork()  →  in the CHILD, systemd sets up the execution context:
            setgid(deploy)  /  setuid(deploy)             ← User=/Group=
            chdir("/srv/app")                             ← WorkingDirectory=
            setrlimit(RLIMIT_NOFILE, 65535)               ← LimitNOFILE=
            reads /etc/myapp.env, builds the environ[]    ← EnvironmentFile=
            dup2() stdout/stderr onto a JOURNAL SOCKET    ← StandardOutput=journal
            prctl(PR_SET_NO_NEW_PRIVS, 1)                 ← NoNewPrivileges=
            (mount namespace tricks)                      ← PrivateTmp=/ProtectSystem=

STEP 6  execve("/usr/bin/node", ["/usr/bin/node","/srv/app/server.js"], environ)
        ▲
        └── DIRECT execve. NO SHELL IS INVOLVED. No ~/.bashrc. No PATH lookup of
            your login shell's PATH. This single fact causes half the bugs below.

STEP 7  Type=simple → systemd marks the unit ACTIVE THE INSTANT execve() returns.
        It does NOT know or care whether your app has bound port 3000 yet.

STEP 8  systemd now sits in its event loop waiting for SIGCHLD on that PID.
        If the process exits → consult Restart=  →  restart after RestartSec=,
        unless StartLimitBurst has been exceeded → mark `failed`, stop trying.
```

And the reverse:

```
COMMAND: sudo systemctl stop myapp   (also the first half of `restart`)

STEP 1  systemd sends KillSignal (default SIGTERM) to the MAIN PID.
        With the default KillMode=control-group, it sends SIGTERM to
        EVERY process in the cgroup.

STEP 2  Your Node app should now:
            process.on('SIGTERM', () => {
              server.close(() => process.exit(0));  // stop accepting new conns,
            });                                     // let in-flight ones finish
        (This is graceful shutdown from topic 17 — signals.)

STEP 3  systemd starts a stopwatch: TimeoutStopSec (default 90s).

STEP 4a If the process exits before the timeout → clean stop. ✅
STEP 4b If it does NOT → systemd sends SIGKILL to the whole cgroup. ☠️
        SIGKILL cannot be caught. In-flight HTTP requests die mid-response.
        Open DB transactions are severed. Clients see connection resets.

  ⇒ TimeoutStopSec is the CONTRACT between systemd and your shutdown handler.
    If your drain takes 30s, TimeoutStopSec must be > 30s. Otherwise every
    single deploy silently drops requests.
```

---

## Unit Types

| Type | What it is | You'll actually use it for |
|---|---|---|
| **`.service`** | A process systemd starts and supervises | **Your Node app.** nginx, postgres, sshd. 95% of your life. |
| `.socket` | A listening socket systemd owns; starts the service on first connection | Socket activation; also how `sshd.socket` works. Lets systemd hold port 80 while nginx restarts. |
| **`.timer`** | Runs a `.service` on a schedule | **The modern replacement for cron** (topic 20). Logs to the journal, has `Persistent=true` for missed runs, and inherits all of `.service`'s isolation. |
| **`.target`** | A **synchronization point** — a named group of units. **Replaces runlevels.** | `multi-user.target` (≈ old runlevel 3, no GUI — what a server boots to), `graphical.target` (≈ 5), `network-online.target`, `rescue.target` (≈ 1). |
| `.mount` / `.automount` | A filesystem mount (auto-generated from `/etc/fstab`) | Ordering a service after a data volume is mounted. |
| `.path` | Watches a file/directory with inotify; activates a service on change | Trigger a rebuild when a file drops in a folder. |
| `.device` | A kernel device, exposed by udev | Depend on a disk existing. |
| **`.slice`** | A cgroup branch for **resource grouping** | `system.slice` (all services), `user.slice`. Cap CPU across a *group* of services. |
| `.scope` | A cgroup of processes systemd did **not** start (it just adopted them) | Your SSH login session is a `.scope`. |

```bash
systemctl list-units --type=service           # what's running
systemctl list-units --type=target            # the sync points currently reached
systemctl get-default                         # graphical.target or multi-user.target
```

---

## The Centerpiece — A Production `.service` File for a Node App

`/etc/systemd/system/myapp.service`

```ini
[Unit]
# ─────────────────────────────────────────────────────────────────────────────
# Metadata + ORDERING/DEPENDENCY graph. Nothing here affects HOW the process runs.
# ─────────────────────────────────────────────────────────────────────────────

Description=Acme Node API Server
# ▲ Shown by `systemctl status`, in the journal, and on the boot console. Be specific.

Documentation=https://github.com/acme/api/blob/main/RUNBOOK.md
# ▲ Free real estate. The 2am you will thank the 2pm you.

After=network.target
# ▲ ORDERING ONLY. It means: "IF network.target is in this transaction, start me
#   AFTER it." It does NOT pull network.target in, and it does NOT require it to
#   succeed. After= is a *sequencing* directive, not a *dependency* directive.
#
#   ⚠ AND: network.target does NOT mean "the network is up and routable."
#     It only means "the networking SERVICE has been started." Your interface may
#     still be waiting on DHCP. If your app CRASHES ON BOOT with EADDRNOTAVAIL or
#     a DNS failure — but works fine when you start it by hand — this is why.
#
#   The fix, if you truly need a routable IP at start:
#       Wants=network-online.target
#       After=network-online.target
#   (requires systemd-networkd-wait-online or NetworkManager-wait-online enabled)
#
#   Honestly though: the *better* fix is to make your app retry its DB connection.
#   Ordering the boot is fragile; resilient apps are not.

Wants=postgresql.service
# ▲ SOFT dependency: "try to start postgres too, but if it FAILS, start me anyway."
#   Wants= is the right default for almost everything.

# Requires=postgresql.service
# ▲ HARD dependency: if postgres fails to start, *I* am not started either.
#   And if postgres is later STOPPED, I am stopped too. Use sparingly — a
#   Requires= chain turns one flaky unit into a cascading outage.

# BindsTo=  — like Requires=, but even stricter: if the other unit goes away for
#             ANY reason (even a crash), I am stopped immediately. Used for
#             hardware/device units.
# Conflicts= — "these two can never run at once." Starting me STOPS the other.
#             (This is how `systemctl isolate rescue.target` kills everything else.)

StartLimitIntervalSec=60
StartLimitBurst=5
# ▲ NOTE THE SECTION: these live in [Unit], not [Service]. (RestartSec= lives in
#   [Service]. Everyone gets this wrong once.)
#   Meaning: "if the unit is started more than 5 times within 60 seconds,
#   STOP TRYING and mark it failed."  See Mistake 4 — this is the rate limiter
#   that permanently blocks a crash-looping app until `systemctl reset-failed`.

# ─────────────────────────────────────────────────────────────────────────────
[Service]
# HOW the process runs. This is the section that actually matters.
# ─────────────────────────────────────────────────────────────────────────────

Type=simple
# ▲ THE DEFAULT, and the CORRECT choice for `node server.js`.
#   systemd fork()s + execve()s and considers the unit "started" the INSTANT
#   the exec succeeds. It does not wait for your app to be ready.
#
#   The full menu:
#     simple   — (default) the process stays in the foreground. Node. Python. Go.
#     exec     — like simple, but systemd waits for execve() to actually SUCCEED
#                before reporting started. Strictly better than simple: a typo'd
#                ExecStart fails IMMEDIATELY and loudly instead of "starting OK,
#                then dying." Use this if you're on systemd 240+.
#     forking  — ☠ FOR OLD DAEMONS THAT BACKGROUND THEMSELVES. systemd starts the
#                process and WAITS FOR THE PARENT TO EXIT before considering it up.
#                A Node app NEVER exits its parent — so systemd waits...
#                and waits... for TimeoutStartSec (90s)... then kills it and marks
#                it FAILED, even though your app was serving traffic the whole time.
#                THIS IS THE #1 COPY-PASTED MISTAKE IN NODE SYSTEMD UNITS.
#                Legit uses: old-school nginx/apache with a PIDFile=.
#     notify   — your app calls sd_notify(READY=1) when it's ACTUALLY ready
#                (npm i sd-notify). systemd holds dependent units until then.
#                The gold standard for zero-downtime ordering.
#     oneshot  — runs, exits, done. For scripts. Pair with RemainAfterExit=yes if
#                you want it to still count as "active" after exiting (e.g. a unit
#                whose job is to apply a sysctl).
#     idle     — like simple, but delays exec until all other jobs are done. Only
#                for cosmetic console-output reasons at boot. Never for a service.

ExecStart=/usr/bin/node /srv/app/server.js
# ▲ ⚠⚠ MUST BE AN ABSOLUTE PATH. `node server.js` will NOT work. `npm start` will
#   NOT work.  systemd does execve() DIRECTLY — there is NO SHELL:
#       • no ~/.bashrc, no ~/.profile        → NVM IS COMPLETELY INVISIBLE (topic 21!)
#       • no $PATH from your login shell
#       • no globs, no pipes, no &&, no $(...), no ~
#   Failure mode: `status=203/EXEC` and "Failed to locate executable node".
#
#   If you genuinely need shell features, be explicit about it:
#       ExecStart=/bin/bash -c '/usr/bin/node /srv/app/server.js >> /dev/null'
#   ...but you almost never should. Wrap the logic in a script instead.
#
#   Using nvm anyway? Then the path is the full, version-pinned monstrosity:
#       ExecStart=/home/deploy/.nvm/versions/node/v20.12.0/bin/node /srv/app/server.js
#   ...which silently breaks the next time anyone runs `nvm install`. Install Node
#   system-wide from NodeSource (topic 21) and be done with it.

ExecStartPre=/usr/bin/node /srv/app/scripts/migrate.js
# ▲ Runs BEFORE ExecStart. If it exits non-zero, the service does NOT start.
#   Prefix with "-" (ExecStartPre=-/usr/bin/foo) to IGNORE its failure.

ExecReload=/bin/kill -HUP $MAINPID
# ▲ This is what `systemctl reload myapp` runs. $MAINPID is expanded by systemd.
#   Without ExecReload=, `systemctl reload` errors out with "Job type reload is
#   not applicable". Your Node app must actually HANDLE SIGHUP for this to mean
#   anything (e.g. re-read config, reopen log files) — a bare kill -HUP with no
#   handler TERMINATES the process, which is worse than useless.

# ExecStop=  — usually omit it. The default (send KillSignal to the cgroup) is right.

WorkingDirectory=/srv/app
# ▲ chdir() before exec. Without this, CWD is `/`, and every relative path in your
#   app breaks: `require('./config')` is fine (module-relative), but
#   `fs.readFileSync('./.env')` and `dotenv.config()` are CWD-relative and will
#   silently read nothing.

User=deploy
Group=deploy
# ▲ setuid()/setgid() before exec. NEVER RUN YOUR APP AS ROOT (topics 06 / 32).
#   An RCE in an npm dependency now owns the whole box instead of one unprivileged
#   account.  Consequence: you CANNOT bind to port <1024 as a non-root user.
#   Correct answer: bind to 3000 and put nginx in front. (The hacky answer is
#   AmbientCapabilities=CAP_NET_BIND_SERVICE — works, but you rarely need it.)

Environment=NODE_ENV=production
Environment="NODE_OPTIONS=--max-old-space-size=512"
# ▲ Inline env vars. Fine for 1–3 non-secret values.

EnvironmentFile=/etc/myapp.env
# ▲ THE RIGHT WAY to inject DATABASE_URL, PORT, API keys.
#   Keep it OUT of the unit file and OUT of git:  chmod 600, chown root:deploy.
#
#   ⚠⚠ THIS FILE IS *NOT* A SHELL SCRIPT. systemd parses it itself:
#        ✅  DATABASE_URL=postgres://user:pass@localhost/db
#        ❌  export DATABASE_URL=...        ← "export" becomes part of the var NAME
#        ❌  PORT=$(cat /etc/port)          ← no command substitution. Literal string.
#        ❌  URL=http://x/$HOST             ← no variable expansion between lines
#        ⚠   PASS="my pass"                 ← quotes ARE stripped, but…
#            PASS=my pass                   ← …unquoted spaces are kept literally.
#            Rule: it's key=value, one per line, # for comments. Nothing else.
#   Prefix with "-" (EnvironmentFile=-/etc/myapp.env) to tolerate a missing file.

Restart=always
# ▲ no           — never restart. (The DEFAULT. Yes, really. If you don't set
#                  Restart=, a crash means your app stays down.)
#   on-failure   — restart on non-zero exit, an uncaught signal, a timeout, or a
#                  watchdog failure.  ⚠ NOT on a clean `process.exit(0)`.
#                  If your app has a bug that cleanly exits 0, it stays down forever.
#   on-abnormal  — signal/timeout only, not on a non-zero exit code.
#   always       — restart no matter how it exited, including exit 0.
#                  ✅ THE RIGHT CHOICE FOR A WEB SERVER. It should never exit. Ever.
#                  (It still respects `systemctl stop` — a deliberate stop is not
#                  a restart trigger.)

RestartSec=3
# ▲ Wait 3s between restarts. Default is 100ms — which will hammer your database
#   with reconnect storms and blow through StartLimitBurst in half a second.

TimeoutStartSec=30
TimeoutStopSec=30
# ▲ TimeoutStopSec: how long systemd waits after SIGTERM before it sends SIGKILL.
#   DEFAULT IS 90s. Set it to a value larger than your app's real drain time.
#   ⚠ This is the directive that decides whether your deploys drop requests.
#   Your Node app must actually handle SIGTERM (topic 17):
#       process.on('SIGTERM', () => server.close(() => process.exit(0)));
#   No handler? Node's default SIGTERM action is to die instantly, mid-request.

KillSignal=SIGTERM
# ▲ The default. Some apps want SIGQUIT (nginx uses SIGQUIT for graceful).
KillMode=control-group
# ▲ The default: signal EVERY process in the cgroup. Right for Node cluster mode —
#   your workers get SIGTERM too.
#   `mixed` = SIGTERM to the main PID only, then SIGKILL to everyone.
#   `process` = only ever touch the main PID (children get orphaned — usually wrong).

LimitNOFILE=65535
# ▲ setrlimit(RLIMIT_NOFILE). THE FIX FOR `EMFILE: too many open files` (topic 19).
#   ⚠⚠ CRITICAL: /etc/security/limits.conf DOES **NOT** APPLY HERE.
#      That file is read by PAM, which only runs for LOGIN sessions. systemd
#      services are not logins — PAM never runs. You can raise limits.conf to a
#      million and your service still gets the default 1024. It MUST be set here.
#   Prove it, live:   cat /proc/$(systemctl show -p MainPID --value myapp)/limits

MemoryMax=512M
# ▲ A cgroup hard cap. Exceed it → the KERNEL OOM-killer kills the process
#   (topic 01!) — but only YOUR service, not a random victim elsewhere on the box.
#   Bounding a leaky app is far kinder than letting it take the whole server down.
CPUQuota=80%
# ▲ Max 80% of ONE core. Use 200% for two cores' worth.
OOMPolicy=stop
# ▲ If the kernel OOM-kills anything in the cgroup, stop the whole unit (so
#   Restart= then applies cleanly) rather than leaving a half-dead service.

StandardOutput=journal
StandardError=journal
# ▲ THE MODERN LOGGING PATTERN, and it's a gift:
#     • Your app writes to stdout/stderr — console.log, pino, winston-to-stdout.
#     • systemd dup2()s those fds onto a journal socket before exec.
#     • Everything is captured, timestamped, indexed, rotated automatically.
#     • NO log files. NO logrotate config. NO "disk full because app.log hit 40GB."
#   (`journal` is the default on modern systemd, but STATE IT — it documents intent.)
#   Use StandardOutput=append:/var/log/myapp.log only if something external
#   genuinely needs a file. You almost certainly don't. (Topic 23.)

SyslogIdentifier=myapp
# ▲ The tag every line gets in the journal. Without it you get "node", and you
#   cannot tell your three Node services apart.

# ── HARDENING: cheap, high-value, no code changes ──────────────────────────────
NoNewPrivileges=true
# ▲ prctl(PR_SET_NO_NEW_PRIVS). The process and ALL its children can NEVER gain
#   privileges — setuid binaries silently lose their power. Kills a whole class of
#   privilege-escalation exploits. Set this on literally every service you write.
PrivateTmp=true
# ▲ A private /tmp in its own mount namespace. Your app's /tmp is invisible to
#   every other process, and is wiped on stop. Kills /tmp symlink-race attacks.
ProtectSystem=strict
# ▲ Mounts /usr, /boot, AND /etc READ-ONLY for this process. `strict` makes the
#   ENTIRE filesystem read-only except /dev, /proc, /sys.
#   (`full` = /usr + /boot + /etc read-only.  `true` = /usr + /boot only.)
ProtectHome=true
# ▲ /home, /root, /run/user become EMPTY for this process. It cannot read anyone's
#   SSH keys or dotfiles.  ⚠ If your app lives under /home, this breaks it — which
#   is one more reason to deploy to /srv or /opt, not a home directory.
ReadWritePaths=/srv/app/uploads /var/lib/myapp
# ▲ Punch specific, minimal write holes through ProtectSystem=strict.
PrivateDevices=true
# ▲ Only /dev/null, /dev/zero, /dev/random etc. No raw disks, no /dev/mem.

# ─────────────────────────────────────────────────────────────────────────────
[Install]
# Consulted ONLY by `systemctl enable` / `disable`. Ignored at runtime.
# ─────────────────────────────────────────────────────────────────────────────

WantedBy=multi-user.target
# ▲ THIS IS WHAT MAKES `systemctl enable` WORK. It tells enable WHERE to put the
#   symlink:
#       /etc/systemd/system/multi-user.target.wants/myapp.service
#           → /etc/systemd/system/myapp.service
#   At boot, systemd reaches multi-user.target (the normal server state, ≈ old
#   runlevel 3) and pulls in everything in that .wants/ directory. That is the
#   ENTIRE mechanism of "starts at boot." No magic, just a symlink.
#
#   ⚠⚠ A UNIT WITH NO [Install] SECTION CANNOT BE ENABLED:
#       $ sudo systemctl enable myapp
#       The unit files have no installation config (WantedBy=, RequiredBy=, Also=,
#       Alias= settings in the [Install] section, and DefaultInstance= for template
#       units). This means they are not meant to be enabled using systemctl.
#   You can still `start` it. It just will never come back after a reboot. This is
#   a genuinely confusing error message — now you know exactly what it means.
```

---

## `start` vs `enable` — The Distinction That Causes Outages

```
┌──────────────────────────────────────────────────────────────────────────┐
│  systemctl start myapp      →  RUN IT NOW.                               │
│                                Does NOTHING about boot. Nothing on disk  │
│                                changes. Reboot → it's gone.              │
│                                                                          │
│  systemctl enable myapp     →  CREATE THE BOOT SYMLINK.                  │
│                                Does NOT start it now. The app is still   │
│                                not running.                              │
│                                                                          │
│  systemctl enable --now myapp  →  BOTH. ✅ This is what you almost       │
│                                    always actually mean.                 │
└──────────────────────────────────────────────────────────────────────────┘
```

**The classic incident, in full:**

> 11:40pm. Prod is down. You SSH in, fix the config, `systemctl start myapp`. It comes up. Requests flow. `systemctl status` is green. You go to bed a hero.
>
> 4:02am. The hosting provider does maintenance and reboots the instance.
>
> 4:03am. systemd boots, reaches `multi-user.target`, and pulls in every unit symlinked into `multi-user.target.wants/`. **`myapp.service` is not one of them** — you never ran `enable`. There is no symlink. systemd has no reason to know your service exists.
>
> 4:04am – 7:30am. The site is down. Nobody is paged, because the *server* is perfectly healthy.

**Prove enable is nothing but a symlink:**
```bash
$ sudo systemctl enable myapp
Created symlink /etc/systemd/system/multi-user.target.wants/myapp.service → /etc/systemd/system/myapp.service.

$ ls -l /etc/systemd/system/multi-user.target.wants/myapp.service
lrwxrwxrwx 1 root root 34 Jun 11 09:12 myapp.service -> /etc/systemd/system/myapp.service

$ systemctl is-enabled myapp
enabled

# The one-liner you should type EVERY time, and the check you should always run:
$ sudo systemctl enable --now myapp
$ systemctl is-enabled myapp && systemctl is-active myapp
enabled
active
```

### `mask` — the nuclear override

```bash
sudo systemctl mask nginx
# Created symlink /etc/systemd/system/nginx.service → /dev/null

sudo systemctl start nginx
# Failed to start nginx.service: Unit nginx.service is masked.
```

`disable` only removes the boot symlink — the unit can still be started manually, **or pulled in as a dependency of something else**, or silently re-enabled by the next `apt upgrade`'s `postinst` (topic 21!). `mask` symlinks the unit to `/dev/null`, which makes it **structurally unstartable by anything, including systemd itself**.

**The real use case:** you run nginx from a Docker container or a custom build, but `apt install` of some package pulls in the distro nginx, which grabs port 80 on every boot and fights your real one. `systemctl mask nginx` ends the argument permanently. Undo with `systemctl unmask nginx`.

---

## Reading `systemctl status` — Line by Line

```
$ systemctl status myapp
● myapp.service - Acme Node API Server
│ │                └── your Description=
│ └── the unit name
└── ● white/green = running.  ● red = failed.  ○ = inactive.

     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; vendor preset: enabled)
             │      │                                   │
             │      │                                   └── ENABLED = will start at boot.
             │      │                                       disabled = won't.
             │      │                                       static   = no [Install] section;
             │      │                                                  can't be enabled.
             │      │                                       masked   = symlinked to /dev/null.
             │      └── WHICH FILE is loaded. Check this — if it says /lib/systemd/system
             │          but you edited /etc/systemd/system, you forgot daemon-reload.
             └── loaded / not-found / bad-setting / error

       Docs: https://github.com/acme/api/blob/main/RUNBOOK.md      ← your Documentation=

     Active: active (running) since Tue 2024-06-11 09:14:02 UTC; 2h 3min ago
             │                      │                              │
             │                      │                              └── UPTIME. If this is
             │                      │                                  "12s" and you didn't
             │                      │                                  just deploy → it is
             │                      │                                  CRASH-LOOPING.
             │                      └── when it last (re)started
             └── active (running)  = healthy
                 active (exited)   = ran and finished (oneshot + RemainAfterExit)
                 activating (start)= still starting (or HUNG — see Type=forking)
                 failed            = died. `Result:` tells you how.
                 inactive (dead)   = stopped, cleanly.

   Main PID: 1841 (node)
             │     └── the process NAME as the kernel sees it (comm)
             └── the PID. Feed it to /proc/1841/limits, /proc/1841/fd, strace…

      Tasks: 11 (limit: 4657)
             │             └── TasksMax — the cgroup's pids.max
             └── threads + processes IN THE CGROUP. Node's libuv threadpool lives here.

     Memory: 84.2M (max: 512.0M)     ← measured from the CGROUP, not RSS guesswork.
        CPU: 3min 12.443s            ← total CPU time consumed since start

     CGroup: /system.slice/myapp.service
             ├─1841 /usr/bin/node /srv/app/server.js
             └─1866 /usr/bin/node /srv/app/worker.js
             ▲
             └── THE CAGE. Every process here dies on `systemctl stop`. Nothing escapes.

Jun 11 11:15:44 api-prod-01 myapp[1841]: GET /health 200 1.2ms
Jun 11 11:16:02 api-prod-01 myapp[1841]: POST /orders 201 42ms
             ▲                  ▲
             │                  └── your SyslogIdentifier=
             └── the last 10 journal lines, for free. Often all the debugging you need.
```

### A failed one

```
× myapp.service - Acme Node API Server
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Tue 2024-06-11 09:20:14 UTC; 3s ago
    Process: 2104 ExecStart=/usr/bin/node /srv/app/server.js (code=exited, status=203/EXEC)
   Main PID: 2104 (code=exited, status=203/EXEC)
                                        ▲
                                        └── 203/EXEC = systemd COULD NOT EXECUTE THE BINARY.
                                            The file doesn't exist, isn't executable, or has
                                            a bad shebang. 99% of the time: your ExecStart
                                            path is wrong (hello, nvm).

Common exit codes you'll actually see:
  status=203/EXEC        binary not found / not executable      ← check ExecStart path
  status=200/CHDIR       WorkingDirectory doesn't exist
  status=217/USER        the User= doesn't exist
  status=209/STDOUT      couldn't set up StandardOutput=
  status=1/FAILURE       YOUR APP exited 1. It ran! Read the journal.
  Result: timeout        systemd gave up waiting  ← almost always Type=forking on a Node app
  Result: signal         killed by a signal (SIGKILL from the OOM killer?)
```

---

## `journalctl` — In Depth

```
journalctl -u myapp -f -n 100 --since "10 min ago" -p err -o cat
│          │       │  │       │                    │     │
│          │       │  │       │                    │     └── OUTPUT FORMAT:
│          │       │  │       │                    │         short (default) | cat (message
│          │       │  │       │                    │         only, no timestamps) |
│          │       │  │       │                    │         json-pretty (every field) |
│          │       │  │       │                    │         verbose
│          │       │  │       │                    └── PRIORITY ≤ err. The syslog levels:
│          │       │  │       │                        0 emerg  1 alert  2 crit  3 err
│          │       │  │       │                        4 warning 5 notice 6 info 7 debug
│          │       │  │       │                        `-p err` shows 0–3 only. Note: Node's
│          │       │  │       │                        stdout→journal is level 6 (info) and
│          │       │  │       │                        stderr→journal is level 3 (err).
│          │       │  │       └── TIME WINDOW. Accepts "2024-06-11 03:00", "yesterday",
│          │       │  │           "-1h", "10 min ago". Pair with --until.
│          │       │  └── last N lines (default 10 with -f)
│          │       └── FOLLOW, like `tail -f`
│          └── filter to ONE unit. The flag you'll type most.
└── the journal client
```

| Command | What it gives you |
|---|---|
| `journalctl -u myapp -n 50 --no-pager` | The last 50 lines. **Your first move when a service fails.** |
| `journalctl -u myapp -f` | Live tail. Watch a deploy happen. |
| `journalctl -u myapp --since "2024-06-11 03:00" --until "03:15"` | The 15 minutes around an incident. |
| `journalctl -u myapp -p err -b` | Errors only, this boot. |
| `journalctl -u myapp --grep "ECONNREFUSED"` | Server-side regex over messages. |
| **`journalctl -b -1 -p err`** | **Errors from the PREVIOUS boot.** The **only** way to see what happened before an unexpected reboot — `-b` is this boot, `-b -1` the one before, `-b -2` before that. `journalctl --list-boots` enumerates them. |
| `journalctl -k -b` | **Kernel** messages (= `dmesg`), this boot. Where the **OOM killer** confesses. |
| `journalctl -o json-pretty -u myapp -n 1` | Every hidden field: `_PID`, `_UID`, `_CMDLINE`, `_SYSTEMD_CGROUP`, `PRIORITY`… |
| `journalctl _PID=1841` | Everything from one PID, across all units. |
| `journalctl --disk-usage` | `Archived and active journals take up 3.8G in the file system.` |
| `journalctl --vacuum-size=200M` | **Delete oldest journals until only 200M remains. The disk-full fix.** |
| `journalctl --vacuum-time=7d` | Delete anything older than 7 days. |
| `journalctl --verify` | Check the journal files' integrity. |

### ⚠ Persistent vs volatile — the trap that erases your evidence

```bash
$ journalctl -b -1
Specifying boot ID or boot offset has no effect, no persistent journal was found.
```

**The journal is RAM-only unless `/var/log/journal/` exists as a directory.**

`/etc/systemd/journald.conf` defaults to `Storage=auto`, and `auto` means: *"persist to `/var/log/journal/` **if that directory exists**; otherwise keep everything in `/run/log/journal/` (a tmpfs) and **lose it all on reboot**."*

On a minimal/cloud/container image, that directory often does not exist. So: your app crashes the box, you reboot to recover, and **every log line that would have explained why is gone.**

```bash
# Which mode am I in?
ls -d /var/log/journal 2>/dev/null && echo "PERSISTENT ✅" || echo "RAM-ONLY — LOGS DIE ON REBOOT ❌"

# Fix it — do this on EVERY server you own, before you need it:
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal    # sets owner/perms correctly
sudo systemctl restart systemd-journald
journalctl --disk-usage

# And cap it so it can never eat the disk (/etc/systemd/journald.conf):
#   [Journal]
#   Storage=persistent
#   SystemMaxUse=500M
#   MaxRetentionSec=1month
```

### Slow boots

```bash
systemd-analyze                 # Startup finished in 3.1s (kernel) + 12.4s (userspace) = 15.6s
systemd-analyze blame           # every unit, sorted by how long IT took
                                #   9.881s  cloud-init.service
                                #   4.202s  snapd.service
                                #   1.104s  systemd-networkd-wait-online.service
systemd-analyze critical-chain  # the DEPENDENCY CHAIN that actually determined boot time.
                                # ⚠ Read THIS, not `blame` — a slow unit that nothing waits
                                #   on doesn't delay your boot at all.
systemd-analyze plot > boot.svg # a full parallel-timeline SVG
```

---

## Exact Syntax Breakdown

```
sudo systemctl enable --now myapp.service
│    │         │      │     │
│    │         │      │     └── the unit. ".service" is the default suffix and can be
│    │         │      │         omitted — but MUST be given for .timer, .socket, etc.
│    │         │      └── ...and start it right now, in the same command
│    │         └── create the boot symlink from [Install] WantedBy=
│    └── the client that messages PID 1
└── changing unit state requires root (or a polkit rule)
```

```
sudo systemctl daemon-reload
│              │
│              └── "PID 1: re-read every unit file from disk and rebuild your cache."
│                  ⚠ RUN THIS AFTER *ANY* EDIT TO ANY UNIT FILE. Always.
│                  It does NOT restart anything. It is safe on prod, any time.
│                  Without it, `systemctl restart` faithfully restarts the OLD version
│                  and you will lose 20 minutes wondering why nothing changed.
└──
  (`systemctl reload-daemon` is NOT a command. `daemon-reload` reloads systemd itself;
   `systemctl reload UNIT` reloads a service. Two completely different things.)
```

```
systemctl show -p MainPID -p MemoryCurrent -p Restart --value myapp
│         │    │                                       │
│         │    │                                       └── print bare values, no "KEY="
│         │    └── -p = the property to show (repeatable)
│         └── dump the FULLY-RESOLVED runtime properties (300+ of them without -p).
│             This is the ground truth — it shows the value systemd is ACTUALLY using,
│             after all drop-ins and defaults. When status and your file disagree,
│             `systemctl show` settles it.
└──
```

```
systemctl restart myapp    vs    systemctl reload myapp
│                                │
│                                └── runs ExecReload= (typically kill -HUP $MAINPID).
│                                    The process NEVER DIES. Zero dropped connections.
│                                    Only works if the unit HAS an ExecReload= and the app
│                                    handles SIGHUP. `nginx -s reload` works this way — it's
│                                    why you can reload nginx config with zero downtime.
└── SIGTERM → (wait ≤ TimeoutStopSec) → SIGKILL → start fresh.
    Connections ARE dropped unless your app drains on SIGTERM.
    `reload-or-restart` = reload if possible, else restart.
```

---

## Example 1 — Basic: Your First Unit, End to End

```bash
# 1. A trivial Node app
sudo mkdir -p /srv/hello
sudo tee /srv/hello/server.js >/dev/null <<'EOF'
const http = require('http');
const port = process.env.PORT || 3000;
const server = http.createServer((req, res) => {
  console.log(`${req.method} ${req.url}`);      // → goes straight to the journal
  res.end('hello\n');
});
server.listen(port, () => console.log(`listening on ${port}`));
process.on('SIGTERM', () => {                    // graceful shutdown (topic 17)
  console.log('SIGTERM received, draining...');
  server.close(() => { console.log('drained, exiting'); process.exit(0); });
});
EOF

# 2. A dedicated, unprivileged service user (topics 06 / 32) — no shell, no home
sudo useradd --system --no-create-home --shell /usr/sbin/nologin hello
sudo chown -R hello:hello /srv/hello

# 3. The unit
sudo tee /etc/systemd/system/hello.service >/dev/null <<'EOF'
[Unit]
Description=Hello Node App
After=network.target

[Service]
Type=simple
User=hello
WorkingDirectory=/srv/hello
ExecStart=/usr/bin/node /srv/hello/server.js
Environment=PORT=3000
Restart=always
RestartSec=3
SyslogIdentifier=hello
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

# 4. THE STEP EVERYONE FORGETS
sudo systemctl daemon-reload

# 5. Start it AND make it survive reboot — one command
sudo systemctl enable --now hello

# 6. Verify
systemctl status hello --no-pager
curl -s localhost:3000            # hello
journalctl -u hello -n 5 --no-pager
# Jun 11 09:14:02 box hello[1841]: listening on 3000
# Jun 11 09:14:19 box hello[1841]: GET /

# 7. PROVE the supervisor actually supervises: kill it and watch it come back
systemctl show -p MainPID --value hello       # 1841
sudo kill -9 1841                             # SIGKILL — no graceful anything
sleep 4
systemctl show -p MainPID --value hello       # 1902  ← NEW PID. systemd restarted it.
journalctl -u hello -n 3 --no-pager
# Jun 11 09:15:03 box systemd[1]: hello.service: Main process exited, code=killed, status=9/KILL
# Jun 11 09:15:03 box systemd[1]: hello.service: Scheduled restart job, restart counter is at 1.
# Jun 11 09:15:06 box hello[1902]: listening on 3000

# 8. PROVE the graceful path works
sudo systemctl stop hello
journalctl -u hello -n 4 --no-pager
# Jun 11 09:16:10 box systemd[1]: Stopping Hello Node App...
# Jun 11 09:16:10 box hello[1902]: SIGTERM received, draining...
# Jun 11 09:16:10 box hello[1902]: drained, exiting          ← your handler ran ✅
# Jun 11 09:16:10 box systemd[1]: hello.service: Deactivated successfully.
```

---

## Example 2 — Production Scenario

**Situation:** 02:47am. `myapp` is down. nginx returns 502. You SSH into `api-prod-01`.

```bash
$ systemctl status myapp --no-pager
× myapp.service - Acme Node API Server
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Wed 2024-06-12 02:41:09 UTC; 6min ago
    Process: 3312 ExecStart=/usr/bin/node /srv/app/server.js (code=exited, status=1/FAILURE)

Jun 12 02:41:09 api-prod-01 systemd[1]: myapp.service: Scheduled restart job, restart counter is at 5.
Jun 12 02:41:09 api-prod-01 systemd[1]: myapp.service: Start request repeated too quickly.
Jun 12 02:41:09 api-prod-01 systemd[1]: myapp.service: Failed with result 'exit-code'.
Jun 12 02:41:09 api-prod-01 systemd[1]: Failed to start Acme Node API Server.
#                                       ▲
#  READ THIS CAREFULLY: "Start request repeated too quickly" is NOT the bug. It is
#  systemd's RATE LIMITER, tripped because the app crashed 5 times in 60 seconds.
#  systemd has now GIVEN UP and will not retry — even if you fix the bug.
#  You must find the REAL error, which happened BEFORE the rate limiter kicked in.

$ journalctl -u myapp -n 30 --no-pager
Jun 12 02:40:52 api-prod-01 myapp[3298]: Error: connect ECONNREFUSED 10.0.1.42:5432
Jun 12 02:40:52 api-prod-01 myapp[3298]:     at TCPConnectWrap.afterConnect [as oncomplete]
Jun 12 02:40:52 api-prod-01 myapp[3298]: FATAL: could not reach database, exiting
#  ← THERE it is. The DB is unreachable. Our app exits 1 on DB failure, systemd
#    restarts it every 3s, it fails again, and after 5 tries in 60s → rate limited.

# Is the DB actually down, or is it us?
$ systemctl is-active postgresql
active
$ ss -tnp | grep 5432                # (topic 28)
# nothing.
$ nc -zv 10.0.1.42 5432
nc: connect to 10.0.1.42 port 5432 (tcp) failed: Connection refused

# The DB moved. Someone changed the DATABASE_URL in the config store but not on the box.
$ sudo cat /etc/myapp.env
NODE_ENV=production
PORT=3000
DATABASE_URL=postgres://api:xxx@10.0.1.42:5432/prod      ← the OLD IP

# ── FIX ────────────────────────────────────────────────────────────────────
$ sudo sed -i 's/10.0.1.42/10.0.1.77/' /etc/myapp.env

# ⚠ EnvironmentFile is read AT PROCESS START, not on daemon-reload. But we also
#   have to clear the rate limiter, or `restart` will refuse to do anything:
$ sudo systemctl start myapp
Job for myapp.service failed.
$ journalctl -u myapp -n 1 --no-pager
Jun 12 02:49:31 api-prod-01 systemd[1]: myapp.service: Start request repeated too quickly.
#   ← STILL BLOCKED. The failure counter has not been cleared.

$ sudo systemctl reset-failed myapp     # ← THE COMMAND. Clears the counter + failed state.
$ sudo systemctl start myapp
$ systemctl is-active myapp
active

$ journalctl -u myapp -f
Jun 12 02:49:58 api-prod-01 myapp[3401]: connected to postgres 10.0.1.77:5432
Jun 12 02:49:58 api-prod-01 myapp[3401]: listening on 3000
Jun 12 02:50:03 api-prod-01 myapp[3401]: GET /health 200 0.9ms      ← 502s stop.

# ── POST-INCIDENT: why did the DB IP change while we slept? Check the reboot.
$ journalctl --list-boots | tail -2
-1 8f2c... Wed 2024-06-12 01:02:11 UTC—Wed 2024-06-12 02:38:44 UTC
 0 a91d... Wed 2024-06-12 02:39:02 UTC—Wed 2024-06-12 02:51:10 UTC
#   ↑ the box DID reboot at 02:38. Only -b -1 can tell us why:
$ journalctl -b -1 -p err --no-pager | tail -5
Jun 12 02:38:41 api-prod-01 kernel: Out of memory: Killed process 2104 (node) total-vm:2411520kB
#   ← the OOM killer (topic 01/18) took Node down, the host rebooted, the DB failed
#     over to a new IP, and our hardcoded env file never followed it.
#     REAL FIX: MemoryMax= on the unit + a DNS name instead of an IP in DATABASE_URL.
```

**The permanent fixes, applied:**
```bash
sudo systemctl edit myapp        # a DROP-IN — does not touch the base unit
```
```ini
[Service]
MemoryMax=768M          # bound it — the OOM killer takes only US, not the whole box
Restart=always
RestartSec=10           # back off harder; a DB failover takes ~30s

[Unit]
StartLimitIntervalSec=300
StartLimitBurst=10      # tolerate a 5-minute outage without permanently giving up
```
```bash
sudo systemctl daemon-reload && sudo systemctl restart myapp
systemctl cat myapp     # confirm the drop-in is merged in
```

---

## Common Mistakes

### Mistake 1 — Editing a unit and not running `daemon-reload`

**Wrong:**
```bash
sudo vim /etc/systemd/system/myapp.service     # change ExecStart
sudo systemctl restart myapp                   # "why is it still running the old command?!"
```

**Root cause:** systemd parses every unit file **once**, at `daemon-reload` (and at boot), into an **in-memory object graph**. `systemctl restart` does not touch the disk at all — it operates on the cached object. Your edit is sitting on disk being ignored.

systemd *does* try to warn you, but the warning scrolls past in `status`:
```
Warning: The unit file, source configuration file or drop-ins of myapp.service changed
on disk. Run 'systemctl daemon-reload' to reload units.
```

**Right:**
```bash
sudo vim /etc/systemd/system/myapp.service
sudo systemctl daemon-reload      # ← ALWAYS. It is free, instant, and safe on prod.
sudo systemctl restart myapp
systemctl cat myapp               # ← and VERIFY the effective unit is what you think
```
**Prevention:** muscle-memory the pair `daemon-reload && restart`. And if a change ever "does nothing," run `systemctl cat` **first** — it prints the unit systemd is *actually* using.

---

### Mistake 2 — `Type=forking` on a Node app

**Wrong:**
```ini
[Service]
Type=forking
ExecStart=/usr/bin/node /srv/app/server.js
```
```
$ sudo systemctl start myapp
(...hangs for 90 seconds...)
Job for myapp.service failed because a timeout was exceeded.

$ systemctl status myapp
     Active: failed (Result: timeout)
Jun 11 09:20:14 box systemd[1]: myapp.service: start operation timed out. Terminating.
```
Meanwhile `curl localhost:3000` **worked fine the whole time**.

**Root cause:** `Type=forking` tells systemd *"the process I exec will fork a background daemon and then the PARENT will EXIT. Wait for that exit; that's my signal that startup is complete."* Node does no such thing — `node server.js` stays in the foreground **forever**, which is correct behaviour. So systemd waits for a parent exit that will never come, hits `TimeoutStartSec` (90s), declares the start a failure, **SIGKILLs your perfectly healthy server**, and marks the unit `failed`.

**Right:**
```ini
Type=simple      # or Type=exec (systemd 240+) — strictly better: it also verifies execve()
```
**Prevention:** `Type=forking` is for daemons that background *themselves* and write a `PIDFile=`. If your process stays in the foreground — Node, Python, Go, Java, and every modern runtime — you want `simple`/`exec`. Rule of thumb: **if it runs correctly in your terminal without `&`, it is `Type=simple`.**

---

### Mistake 3 — A relative `ExecStart`, and the nvm ambush

**Wrong:**
```ini
ExecStart=node /srv/app/server.js          # ❌ 203/EXEC
ExecStart=npm start                        # ❌ 203/EXEC
ExecStart=/usr/bin/node server.js > /var/log/app.log 2>&1   # ❌ ">" is not a file
```
```
Main PID: 2104 (code=exited, status=203/EXEC)
myapp.service: Failed to locate executable node: No such file or directory
```
...and yet `which node` in your SSH session prints a perfectly good path.

**Root cause:** systemd calls **`execve()` directly. There is no shell anywhere in the picture.**
- No `~/.bashrc` is sourced → **nvm, which is a bash function that rewrites `PATH`, does not exist** (topics 14 and 21).
- The unit's `PATH` is systemd's minimal default (`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`), not your login shell's.
- `>`, `|`, `&&`, `*`, `~`, `$(...)` are **shell syntax**. `execve()` passes them to your program as literal `argv` strings.

```
Your SSH session:  bash → sources ~/.bashrc → nvm.sh rewrites PATH → node found ✅
systemd:           execve("/usr/bin/node") — no bash, no .bashrc, no nvm      ❌
```

**Right:**
```ini
# The correct production answer: system-wide Node from NodeSource (topic 21)
ExecStart=/usr/bin/node /srv/app/server.js

# If you MUST use nvm — full absolute path (and accept that it breaks on nvm upgrade)
ExecStart=/home/deploy/.nvm/versions/node/v20.12.0/bin/node /srv/app/server.js

# If you genuinely need shell features, ask for a shell EXPLICITLY
ExecStart=/bin/bash -lc 'node /srv/app/server.js'

# Never do output redirection in ExecStart. Use the journal:
StandardOutput=journal
StandardError=journal
```
**Prevention:** before writing any unit, run `command -v node` **and** `dpkg -S $(command -v node)` (topic 21). If dpkg says *"no path found,"* your Node is invisible to systemd and you need to fix that first.

---

### Mistake 4 — Fighting "start request repeated too quickly"

**Wrong:** you fix the bug, run `systemctl restart myapp`, and systemd flatly refuses. You restart the whole *server* to clear it.

**Root cause:** `StartLimitIntervalSec` (default 10s) + `StartLimitBurst` (default 5). If a unit is started more than 5 times in 10 seconds, systemd concludes it is in a crash loop, **enters the `failed` state, and refuses all further start requests** — permanently — until the failure counter is explicitly cleared. Fixing the underlying bug does **not** clear the counter. Restarting the box does, which is why "reboot fixed it" is such a persistent myth here.

**Right:**
```bash
sudo systemctl reset-failed myapp     # clears the failed state AND the rate counter
sudo systemctl start myapp

systemctl reset-failed                # clear ALL failed units on the box
systemctl --failed                    # list everything currently failed
```
**Prevention:** set `RestartSec=5` or more (never the 100ms default — it burns your 5 attempts in half a second), and widen the window for services with a slow external dependency:
```ini
[Service]
RestartSec=10
[Unit]
StartLimitIntervalSec=300
StartLimitBurst=10
```

---

### Mistake 5 — Setting fd limits in `/etc/security/limits.conf`

**Wrong:**
```bash
# Node throws EMFILE: too many open files under load (topic 19).
$ sudo tee -a /etc/security/limits.conf <<< "deploy soft nofile 65535"
$ sudo systemctl restart myapp
# ...still EMFILE. Still 1024.
```

**Root cause:** `/etc/security/limits.conf` is read by **PAM** (`pam_limits.so`), and PAM only runs when a **login session** is created — SSH, `su`, a TTY login. **A systemd service is not a login.** No PAM, no `limits.conf`, no raised rlimit. Your service inherits systemd's own default (`DefaultLimitNOFILE`, commonly 1024 soft).

This is genuinely confusing because `ulimit -n` in your SSH session will happily report `65535` — you raised *your login's* limit, not your *service's*.

**Prove it — read the kernel's view of the actual running process:**
```bash
$ PID=$(systemctl show -p MainPID --value myapp)
$ grep "open files" /proc/$PID/limits
Max open files            1024                 4096                 files
#                         ▲ soft               ▲ hard      ← the truth, straight from the kernel
```

**Right:**
```ini
[Service]
LimitNOFILE=65535
```
```bash
sudo systemctl daemon-reload && sudo systemctl restart myapp
grep "open files" /proc/$(systemctl show -p MainPID --value myapp)/limits
# Max open files            65535                65535                files   ✅
```
**Prevention:** for **anything** started by systemd, the limit lives in the **unit**. `limits.conf` is only for interactive logins. (Set `DefaultLimitNOFILE=` in `/etc/systemd/system.conf` to change the fallback for every unit.)

---

## Hands-On Proof

```bash
# PROVE IT: systemd is PID 1, and its parent is the kernel itself
ps -p 1 -o pid,ppid,comm
ls -l /sbin/init                 # → /lib/systemd/systemd

# PROVE IT: every service is a child of PID 1, in its own cgroup
systemd-cgls | head -30
cat /proc/$(systemctl show -p MainPID --value ssh)/cgroup
# 0::/system.slice/ssh.service   ← the cage

# PROVE IT: `enable` is nothing but a symlink
systemctl enable --now hello
ls -l /etc/systemd/system/multi-user.target.wants/
readlink -f /etc/systemd/system/multi-user.target.wants/hello.service

# PROVE IT: `mask` is a symlink to /dev/null
sudo systemctl mask hello && ls -l /etc/systemd/system/hello.service
sudo systemctl start hello       # Unit hello.service is masked.
sudo systemctl unmask hello

# PROVE IT: systemd caches units in memory — daemon-reload is mandatory
sudo sed -i 's/Description=.*/Description=CHANGED/' /etc/systemd/system/hello.service
systemctl show -p Description --value hello     # ← still the OLD text!
sudo systemctl daemon-reload
systemctl show -p Description --value hello     # ← CHANGED

# PROVE IT: `systemctl cat` shows the merged, effective unit (base + drop-ins)
systemctl cat ssh

# PROVE IT: there is NO shell — systemd execve()s directly
sudo systemctl show -p ExecStart --value hello
# { path=/usr/bin/node ; argv[]=/usr/bin/node /srv/hello/server.js ; ... }

# PROVE IT: the rlimit really comes from the unit, not limits.conf
grep "open files" /proc/$(systemctl show -p MainPID --value hello)/limits

# PROVE IT: your app's console.log lands in the journal with zero config
journalctl -u hello -n 5 -o json-pretty | grep -E '"MESSAGE"|"PRIORITY"|"_PID"'

# PROVE IT: stdout is priority 6 (info), stderr is priority 3 (err)
journalctl -u hello -p err -n 5

# PROVE IT: is the journal persistent, or will it vanish on reboot?
ls -d /var/log/journal 2>/dev/null && echo PERSISTENT || echo "RAM-ONLY ❌"
journalctl --disk-usage
journalctl --list-boots

# PROVE IT: what did the last boot look like?
systemd-analyze
systemd-analyze critical-chain
```

---

## Practice Exercises

### Exercise 1 — Easy

On a Linux box (or `docker run -it --rm --privileged jrei/systemd-ubuntu:22.04`):

```bash
systemctl status ssh --no-pager
systemctl is-enabled ssh
systemctl is-active ssh
systemctl cat ssh
systemctl show -p MainPID -p Restart -p LimitNOFILE ssh
journalctl -u ssh -n 20 --no-pager
```

**Answer from your output:** Which *file* is the ssh unit loaded from — `/lib` or `/etc`? Is it enabled? What is its `Restart=` policy? What is its `LimitNOFILE`?

---

### Exercise 2 — Medium

Build the `hello.service` from Example 1, then:

1. `kill -9` the main PID. Prove from the journal that systemd restarted it and that the PID changed.
2. Add `EnvironmentFile=/etc/hello.env` with `PORT=4000` in it. Make the change take effect. Prove `curl localhost:4000` works and `localhost:3000` does not.
3. Now deliberately break it: put `export PORT=5000` in the env file. Restart. Read the journal. **Explain the exact error, and why `export` is illegal there.**
4. Add `LimitNOFILE=65535` **via a drop-in** (`systemctl edit hello` — do *not* edit the base file). Prove it took effect by reading `/proc/PID/limits`. Then show the merge with `systemctl cat hello`.

---

### Exercise 3 — Hard (Production Simulation)

Deploy a Node app the way you would on a real server, and then break it in three ways.

**Deploy:**
1. Create a system user `appsvc` (no shell, no home). App in `/srv/api`.
2. Write `/etc/systemd/system/api.service`: `Type=simple`, absolute `ExecStart`, `User=appsvc`, `WorkingDirectory=`, `EnvironmentFile=/etc/api.env` (mode `0600`, root-owned), `Restart=always`, `RestartSec=5`, `TimeoutStopSec=30`, `LimitNOFILE=65535`, `MemoryMax=256M`, `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=strict`, `ReadWritePaths=/srv/api/uploads`, `SyslogIdentifier=api`, `[Install] WantedBy=multi-user.target`.
3. `daemon-reload` → `enable --now` → `status` → `journalctl -u api -f`.
4. Implement `SIGTERM` → `server.close()` in the app. Run `systemctl restart api` while a slow request (`/slow`, 5s) is in flight. **Prove from the journal that the in-flight request completed before the process exited.**

**Then break it, and fix each from the journal alone:**
- **Break A:** change `ExecStart` to `node /srv/api/server.js` (no path). Predict the exit code *before* you run it. Confirm.
- **Break B:** set `Type=forking`. Time how long `systemctl start` hangs. Read what `status` says. Explain why `curl` worked during those 90 seconds.
- **Break C:** make the app `process.exit(1)` on startup. Restart it 6 times in 10 seconds. Get "start request repeated too quickly." Now fix the app — and show that `systemctl start` **still fails** until you run the one command that unblocks it.
- **Finally:** delete the `[Install]` section, `daemon-reload`, and run `systemctl enable api`. Quote the exact error. Explain it in one sentence.

Then: `journalctl -u api --since "10 min ago" -p err -o cat` and `systemd-analyze blame | head`.

---

## Mental Model Checkpoint

1. **What is PID 1, what creates it, and what happens if it dies?**
2. **`start` vs `enable` — what does each one physically do on disk? Which one saves you at 4am?**
3. **You edited a unit file and restarted the service, but nothing changed. What did you forget, and why is it necessary?**
4. **Why does `ExecStart=node server.js` fail with `203/EXEC` even though `node` works when you SSH in?**
5. **Why does `Type=forking` make a Node service hang for 90 seconds and then fail — while the app is serving traffic the whole time?**
6. **`After=network.target` — does it (a) start the network, (b) require the network, (c) guarantee the network is routable? What does it actually do?**
7. **Your app throws `EMFILE` and you raised the limit in `/etc/security/limits.conf`. Why did that change nothing, and where does the limit really belong?**
8. **The box rebooted unexpectedly. Which single `journalctl` flag shows you what happened *before* the reboot — and what must exist on disk for it to work at all?**
9. **What does `systemctl mask` do that `disable` does not, and when do you need it?**
10. **How does `TimeoutStopSec` decide whether your deploys drop in-flight HTTP requests?**

---

## Quick Reference Card

| Command | What It Does | Key Flags / Notes |
|---|---|---|
| `systemctl start / stop / restart` | Run / kill / cycle a unit **now** | Does **not** affect boot |
| `systemctl reload` | Run `ExecReload=` (SIGHUP) — **no downtime** | Needs `ExecReload=` in the unit |
| `systemctl reload-or-restart` | Reload if possible, else restart | Good for deploy scripts |
| **`systemctl enable`** | **Create the boot symlink from `[Install]`** | **`--now` = enable + start** |
| `systemctl disable` | Remove the boot symlink | Can still be started manually |
| **`systemctl mask` / `unmask`** | **Symlink unit → `/dev/null`; unstartable by anything** | The fix for a package that re-enables itself |
| **`systemctl daemon-reload`** | **Re-read all unit files from disk** | **After EVERY unit edit. Always. Safe on prod.** |
| `systemctl status` | State + PID + cgroup + last 10 log lines | `--no-pager`, `-l` |
| **`systemctl cat UNIT`** | **The effective, merged unit (base + drop-ins)** | Your first debugging move |
| **`systemctl edit UNIT`** | **Create a drop-in override** | `--full` forks the whole unit (rarely right) |
| `systemctl show -p X UNIT` | Fully-resolved runtime property | `--value` for bare output |
| **`systemctl reset-failed`** | **Clear failed state + the restart rate limiter** | The "repeated too quickly" unblocker |
| `systemctl is-active / is-enabled / is-failed` | Script-friendly checks (exit code!) | Use these in CI/health checks |
| `systemctl list-units --type=service` | What's loaded and running | `--all`, `--failed` |
| `systemctl list-unit-files` | Every unit on disk + enabled state | `--state=enabled` |
| `systemctl --failed` | Everything currently broken | First command after any incident |
| `systemctl kill -s SIGUSR2 UNIT` | Send an arbitrary signal to the cgroup | `--kill-who=main` |
| `systemd-cgls` / `systemd-cgtop` | The cgroup tree / live per-service resource use | |
| **`journalctl -u UNIT -n 50`** | **Last 50 lines for one unit** | **Your first move on a failed service** |
| `journalctl -u UNIT -f` | Live tail | |
| `journalctl --since / --until` | Time window | `"10 min ago"`, `"2024-06-11 03:00"` |
| `journalctl -p err` | Priority ≤ err | `emerg|alert|crit|err|warning|notice|info|debug` |
| **`journalctl -b -1`** | **The PREVIOUS boot** — the only way to debug a reboot | Needs a **persistent** journal |
| `journalctl -k` | Kernel messages (`dmesg`) — where the OOM killer confesses | |
| `journalctl --grep` | Regex over messages | |
| `journalctl -o json-pretty / cat` | Every field / message-only | |
| **`journalctl --vacuum-size=200M`** | **Shrink the journal — the disk-full fix** | `--vacuum-time=7d`, `--disk-usage` |
| `systemd-analyze blame / critical-chain` | Why boot is slow (read `critical-chain`) | `plot > boot.svg` |

---

## When Would I Use This at Work?

### Scenario 1: Deploying a Node app to a fresh server, properly
No pm2, no `nohup`, no `screen`. You write **one** `.service` file: it runs as an unprivileged `deploy` user, reads secrets from a root-owned `0600` `EnvironmentFile`, restarts on crash with a sane backoff, is bounded by `MemoryMax`, gets `LimitNOFILE=65535`, drains gracefully on SIGTERM within `TimeoutStopSec`, logs to the journal with zero logrotate config, and comes back automatically after a reboot because you ran `enable --now`. That is the entire "process manager" problem solved by the init system that is already on the box.

### Scenario 2: The 4am reboot that took the site down
You fixed prod at midnight with `systemctl start` and never ran `enable`. The box rebooted; `multi-user.target.wants/` had no symlink for your unit; the site stayed down for three hours. Afterwards, `journalctl -b -1 -p err` shows the OOM kill that caused the reboot in the first place, `systemctl is-enabled myapp` proves the gap, and `MemoryMax=` + `enable --now` mean it can't happen twice.

### Scenario 3: Every deploy drops requests, and nobody knows why
`systemctl restart myapp` sends SIGTERM to the cgroup. Your Node app has no SIGTERM handler, so it dies **instantly**, mid-response — every in-flight request becomes a connection reset. You add `process.on('SIGTERM', () => server.close(() => process.exit(0)))`, set `TimeoutStopSec=30` (longer than your drain), and prove it from the journal: `Stopping...` → `draining` → `drained, exiting` → `Deactivated successfully`. Zero dropped requests, forever.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 01 — What Linux Actually Is | The kernel `execve()`s PID 1. cgroups and the OOM killer are kernel features systemd merely *drives*. |
| **Builds on** | 06 — Users and Groups | `User=`/`Group=` are `setuid()`/`setgid()` before `execve()`. Never run your app as root. |
| **Builds on** | 14 — Environment Variables | systemd runs **no shell**: no `.bashrc`, no `PATH` from your login. This is why `ExecStart` must be absolute and why nvm is invisible. |
| **Builds on** | 16 — Processes in Depth | PID 1 is the orphan reaper. cgroups mean the double-fork trick can't hide a process from the supervisor. |
| **Builds on** | 17 — Process Management | `restart` = SIGTERM → wait `TimeoutStopSec` → SIGKILL. Your graceful-shutdown handler *is* the contract. |
| **Builds on** | 19 — File Descriptors | `LimitNOFILE=` is the **only** place a service's `EMFILE` limit can be raised. `limits.conf` does nothing here. |
| **Builds on** | 20 — Cron and Scheduling | `.timer` units are the modern cron — journal-logged, and they share every isolation directive above. |
| **Builds on** | 21 — Package Management | `apt install nginx` drops a unit into `/lib/systemd/system` and enables it in `postinst`. Vendor units vs `/etc` overrides. And nvm-vs-NodeSource is a *systemd* decision. |
| **Next** | 23 — Logs and Log Management | `StandardOutput=journal` means your logs *are* the journal. Now learn journald vs syslog vs logrotate, and what `--vacuum` really deletes. |
| **Used by** | 33 — Node in Production | This doc *is* the systemd half of that topic. pm2 vs systemd, zero-downtime reloads, graceful shutdown. |
| **Used by** | 34 — Performance Investigation | `systemd-cgtop`, `MemoryMax=`, and `journalctl -k` are the front door to a resource investigation. |
| **Used by** | 35 — Security Basics | `NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp`, `User=` — the cheapest hardening you will ever deploy. |
