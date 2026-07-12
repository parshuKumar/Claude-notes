# 35 — Security Basics

## ELI5 — The Simple Analogy

Imagine you build a house on a busy street corner and, the moment the roof goes on, you leave for vacation. You did not announce your address. You did not invite anyone. And yet — within minutes — someone walks up and tries the front door. Then the back door. Then every window. They try the key that's under the mat at *every* house ("admin/admin"). They come back every few seconds, forever, day and night.

This is not a movie villain who has chosen *you*. It is a **swarm of automated burglars** that walk down every street on Earth trying every door on every house, constantly, because trying a door is nearly free and one unlocked door in ten thousand pays for the whole operation.

Your server is that house. Boot a fresh cloud VM with a public IP and watch `/var/log/auth.log`: within **minutes**, before you've even deployed anything, bots from all over the world are trying to log in. They are not targeting you. You are just a door on the street.

The good news: because the attacks are boring and automated, **the defenses are boring and mechanical too.** You don't need to be a genius. You need to lock the doors, not hide a key under the mat, and not leave your safe (the database) sitting on the front lawn. Almost every real breach is one of five boring mistakes. This doc is the checklist that closes all five.

---

## Where This Lives in the Linux Stack

Security is not one layer — it is a **property of every layer**, and an attacker will happily walk down the stack looking for the one you forgot.

```
                                    THE ATTACKER WANTS TO GO DOWN THIS STACK ↓
Hardware / Cloud
  │   ◀ Cloud SECURITY GROUP — the outermost gate (a second firewall you can't
  │     bypass from inside the box). Your last line if the host firewall fails.
  │
  └── KERNEL
       │   ◀ netfilter/iptables (what ufw drives) lives HERE. So do
       │     capabilities, namespaces, seccomp, and the setuid mechanism —
       │     the things a container escape or privilege escalation abuses.
       │
       └── System Calls
            │   ◀ auditd traces security-relevant syscalls HERE
            │
            └── System Services (sshd, nginx, postgres, dockerd)
                 │   ◀ THIS is where 90% of attacks land. An exposed service
                 │     with weak or no auth. sshd is the #1 target.
                 │     fail2ban watches these services' LOGS.
                 │
                 └── Your Node app + its dependencies
                      │   ◀ npm dependencies are a LARGER attack surface than
                      │     the whole OS. One malicious postinstall script runs
                      │     as your user, with your database credentials.
                      │
                      └── Secrets (.env, keys, tokens)
                            ◀ the actual prize. Everything above exists to
                              protect these. Leak one and all the rest was theatre.
```

**The core discipline:** you are not trying to be un-hackable (nobody is). You are trying to not be the **easy** door. The bots have finite time; the moment your box costs more effort than the ten thousand un-hardened boxes next door, they move on.

---

## What Is This?

Server hardening is the practice of **systematically closing the boring, well-known holes** that automated attackers exploit: password logins, exposed services, unpatched software, leaked secrets, and over-privileged processes. It is checklist-shaped on purpose — the threats are mechanical, so the defenses are too.

This doc covers a **baseline for a small production server**. It is not a compliance program (SOC 2, PCI, HIPAA), not a substitute for a professional security review, and not enough for a high-value target. It is the difference between "owned in the first hour" and "boring and hardened," which is the difference that matters for 99% of the Node backends you will run.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| SSH password auth is brute-forceable | A bot guesses `deploy`/`password123` at 3am and now owns your box |
| A DB bound to 0.0.0.0 is public | Your Redis/Mongo gets found by a scanner in hours, dumped, and ransomed. This is the #1 recurring headline breach. |
| Docker bypasses ufw (topic 29) | You "firewalled" port 5432, but `-p 5432:5432` punched a hole straight through ufw to the internet |
| `git rm` doesn't remove a leaked secret | You "deleted" the AWS key — it's still in git history, being scraped. You must **rotate**, not delete. |
| The `docker` group == root (topic 32) | You added `deploy` to the docker group for convenience and handed them full root |
| A rooted box can't be cleaned | You "remove the malware" and reboot — the attacker's cron job / SSH key / systemd timer reinstalls it in 60 seconds |
| `npm install` runs arbitrary code | A typosquatted dependency's postinstall script exfiltrates your `.env` on `npm install` |
| Unpatched CVEs | A 6-month-old, one-line-to-fix CVE in your unpatched OpenSSL is how they got in |

---

## The Attacker's Playbook (so the defenses make sense)

Every defense in this doc maps to a step here. Learn the kill chain and the checklist stops being a list of random chores.

```
┌─────────────┐   ┌─────────────┐   ┌──────────────┐   ┌─────────────┐
│ 1. SCAN     │──►│ 2. FIND AN  │──►│ 3. BRUTE-    │──►│ 4. GET A    │
│ the whole   │   │ OPEN PORT   │   │ FORCE or     │   │ SHELL       │
│ internet    │   │ (22? 6379?  │   │ EXPLOIT      │   │ (as some    │
│ for IPs     │   │ 5432? 2375?)│   │ (weak pass / │   │ user)       │
│ (masscan)   │   │             │   │ known CVE)   │   │             │
└─────────────┘   └─────────────┘   └──────────────┘   └──────┬──────┘
   ▲ Defense:        ▲ Defense:         ▲ Defense:             │
   you can't stop    FIREWALL:          KEY-ONLY SSH kills     │
   scanning; make    expose ONLY        brute force.           │
   sure it finds     22/80/443.         PATCHING kills the     │
   nothing useful.   NEVER a DB port.   CVE. fail2ban adds     │
                                        friction + noise cut.  │
                                                               ▼
┌─────────────┐   ┌──────────────────────────────────┐   ┌─────────────┐
│ 7. PIVOT /  │◄──│ 6. PERSIST                        │◄──│ 5. ESCALATE │
│ EXFILTRATE  │   │ • an SSH key in authorized_keys   │   │ TO ROOT     │
│ • dump DB   │   │ • a cron job / at job             │   │ • a setuid  │
│ • steal     │   │ • a systemd unit or TIMER         │   │   binary    │
│   secrets   │   │ • a modified ~/.bashrc / .profile │   │ • a sudo    │
│ • ransom    │   │ • a malicious PAM module          │   │   misconfig │
│ • crypto-   │   └──────────────────────────────────┘   │ • a kernel  │
│   miner     │      ▲ Defense: LEAST PRIVILEGE means     │   exploit   │
└─────────────┘      even a shell can't reach root.       │ • the      │
   ▲ Defense:        DETECTION finds the persistence.     │   docker    │
   egress rules,     Rooted? You cannot clean it —        │   group     │
   secrets not on    REBUILD from a known-good image.     └─────────────┘
   the box.                                                ▲ Defense:
                                                           run as non-root,
                                                           drop caps, no
                                                           setuid surprises.
```

---

## SSH Hardening — The Highest-Value Work You Will Do

**Most attacks arrive at sshd.** It is the door on the street. Harden this one service and you eliminate the entire brute-force class of attack. (Topic 24 — SSH in Depth covers *how* SSH works; this is how you *lock it down*.) Config lives in `/etc/ssh/sshd_config` (and drop-ins under `/etc/ssh/sshd_config.d/`).

### The single most important change: key-only authentication

```
PasswordAuthentication no
```

**This one line eliminates the entire brute-force attack class.** A password can be guessed — bots try millions. An SSH key cannot; a 4096-bit RSA or an ed25519 key has more entropy than the bots could exhaust before the heat death of the sun. With passwords off, the endless `Failed password` swarm literally *cannot succeed* — they're knocking on a door with no keyhole. (Set up your key first — topic 24 — or you will lock yourself out. See the safety rule below.)

### Force the attacker to guess a username too

```
PermitRootLogin no
```

`root` exists on **every** Linux box. If root can SSH in, the attacker already knows the username — they only have to guess the password (or key). Disable it and they must guess **both** a valid username *and* its credential. Bonus: it forces everyone through a named user + `sudo`, which gives you an **audit trail** of who did what (topic 32) — `root` actions are anonymous, `alice` running `sudo` is not.

### The rest of the hardening block

```sshd_config
PasswordAuthentication no        # ◀ THE big one — kills brute force
PermitRootLogin no               # ◀ force a username guess + sudo audit trail
PubkeyAuthentication yes         # keys only
PermitEmptyPasswords no          # never allow a blank password
MaxAuthTries 3                   # drop the connection after 3 failed attempts
MaxSessions 5                    # limit multiplexed sessions per connection
LoginGraceTime 20                # 20s to authenticate, then disconnect
AllowUsers deploy alice          # ◀ EXPLICIT ALLOWLIST. Nobody else, ever.
                                 #   (or AllowGroups sshusers)
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no                 # you don't need it on a server
ClientAliveInterval 300          # drop dead/idle connections
ClientAliveCountMax 2
```

`AllowUsers`/`AllowGroups` is a **whitelist**: only listed users can authenticate over SSH at all, even with a valid key. It is the tightest control here — an attacker who somehow steals a key for a non-listed user still cannot log in.

### Changing the port — be honest about what it does

```sshd_config
Port 2222        # NOT 22
```

Let's be precise, because people oversell this:

- **Against a targeted attacker: it is SECURITY THEATER.** A port scan finds your SSH on 2222 in seconds. It also breaks tooling that assumes 22 (some deploy scripts, monitoring, `scp` defaults) and you'll forget it and lock yourself out of muscle memory.
- **But it has real operational value:** the automated swarm overwhelmingly hammers port **22** specifically. Moving off 22 cuts roughly **99% of the automated log noise** — your `auth.log` becomes readable, fail2ban has less to chew on, and genuine anomalies stop being buried under thousands of bot attempts.
- **It is NOT a substitute for key-only auth.** Do it *in addition to* `PasswordAuthentication no`, never *instead of* it. A weird port with passwords on is still owned; port 22 with keys-only is still safe.

Verdict: optional, cheap noise reduction; never your actual security.

### ►►► THE RULE THAT SAVES YOU FROM A TOTAL LOCKOUT ◀◀◀

You are about to change how you log in. Get it wrong and you are **locked out of your own production server** — no shell, at 2am, with the app down.

```bash
# 1. VALIDATE THE CONFIG SYNTAX BEFORE RESTARTING. This catches typos.
sudo sshd -t
#   No output = valid. Errors printed = DO NOT restart yet, you'll break sshd.

# 2. Reload/restart sshd:
sudo systemctl reload ssh     # (Ubuntu 22.04 unit is 'ssh', not 'sshd')

# 3. ►►► DO NOT CLOSE YOUR CURRENT SESSION. ◀◀◀
#    Open a SECOND, BRAND-NEW terminal and prove you can still get in:
ssh deploy@your-server
#    Only when the NEW session works do you close the old one.
#    Your existing session keeps working even with a broken config — that's
#    the trap. The broken config only bites the NEXT login.
```

**The rule: test a new SSH session BEFORE you close the old one.** Your current connection survives a bad config change; the *next* connection is the one that fails. If you close the working session first and the config is broken, you are locked out.

**Your escape hatch:** know your cloud provider's **serial console** or **rescue mode** (AWS EC2 Serial Console, DigitalOcean Recovery Console, GCP serial port) *before* you need it. It's an out-of-band way into the box that does not use sshd — the one thing that still works when you've locked yourself out of SSH.

---

## fail2ban — Automating the Ban

**What it does:** watches a log file → matches a regex for repeated failures from one IP → adds a temporary **firewall ban** for that IP → removes it after a timeout. It turns "1,000 password guesses" into "5 guesses, then banned for an hour."

```
   /var/log/auth.log                fail2ban                 nftables/iptables
   ┌────────────────┐    tail -f    ┌──────────┐   ban IP   ┌──────────────┐
   │ Failed password│──────────────►│ regex    │──────────►│ DROP all from │
   │ from 45.9.x.x  │  (5th time    │ matches  │  (via     │ 45.9.x.x for  │
   │ Failed password│   in 10 min)  │ "failregex"│ action)  │ 3600 seconds  │
   └────────────────┘               └──────────┘           └──────────────┘
```

It does **not** replace key-only auth (with passwords off, there's nothing to brute-force). But it: cuts log noise, blocks CVE-scanning bots hitting *any* service, and — its best use — protects **your own app's login endpoint**.

### Configuration — the one rule that trips everyone

```
/etc/fail2ban/jail.conf   ← the DEFAULTS. NEVER EDIT THIS. Overwritten on upgrade.
/etc/fail2ban/jail.local  ← YOUR overrides. Edit THIS. Survives upgrades.
/etc/fail2ban/jail.d/*.conf ← or drop-in files here.
```

**Never edit `jail.conf`.** A package upgrade replaces it and silently wipes your changes. Everything you set goes in `jail.local`.

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
bantime  = 1h          # how long an IP stays banned
findtime = 10m         # the window in which failures are counted
maxretry = 5           # this many failures within findtime → ban
ignoreip = 127.0.0.1/8 ::1 203.0.113.10    # ◀◀◀ YOUR OFFICE/VPN IP. CRITICAL.

[sshd]
enabled = true
port    = ssh          # or 2222 if you changed it
maxretry = 3
bantime  = 24h         # be harsher on SSH
```

### ►►► `ignoreip` — whitelist yourself or you WILL ban yourself ◀◀◀

Fat-finger your own SSH password a few times and fail2ban bans **your** IP. Now you're locked out by your own defense. Put your office/home/VPN IP in `ignoreip`. And know the recovery:

```bash
sudo fail2ban-client status sshd          # see banned IPs + stats
sudo fail2ban-client set sshd unbanip 1.2.3.4   # un-ban yourself
#   Locked out entirely? Use the cloud SERIAL CONSOLE to run the unban,
#   or your office IP is in ignoreip so it never happened.
```

### A custom jail for your Node app's login endpoint (the genuinely useful part)

This ties directly to topic 23 (logs). Have your app log failed logins in a greppable format, then teach fail2ban to watch it. (Topic 11's grep/regex reused for the filter.)

```
# Your Node app logs (to /var/log/myapp/access.log):
2026-07-12T14:03:11Z WARN auth failed for user=alice ip=45.9.201.44 reason=badpass
```

```ini
# /etc/fail2ban/filter.d/myapp-auth.conf
[Definition]
failregex = ^.*auth failed for user=\S+ ip=<HOST> reason=.*$
#   <HOST> is fail2ban's magic token — it captures the IP to ban.
```

```ini
# /etc/fail2ban/jail.local  — add:
[myapp-auth]
enabled  = true
filter   = myapp-auth
logpath  = /var/log/myapp/access.log
maxretry = 10
findtime = 5m
bantime  = 1h
port     = http,https
```

```bash
sudo fail2ban-client reload
sudo fail2ban-client status myapp-auth        # watch it work
# Test your regex against real log lines WITHOUT restarting:
fail2ban-regex /var/log/myapp/access.log /etc/fail2ban/filter.d/myapp-auth.conf
```

Now credential-stuffing your login API gets an IP banned at the **firewall** after 10 tries — the attacker never reaches your app again this hour.

---

## Reading /var/log/auth.log — Hands-On Forensics

This is where you *see* the attack. (Topic 23 — Logs; on systemd-only boxes the same events are in `journalctl -u ssh`; RHEL/CentOS use `/var/log/secure`.) Every SSH auth event lands here.

```bash
# ►► The SCALE of the noise. This number will genuinely shock you.
grep "Failed password" /var/log/auth.log | wc -l
#   4127     ← on a box that's been up a WEEK with password auth off, this is
#              4,127 bots that tried and failed. This is normal. This is the street.
```

```bash
# ►► The TOP ATTACKING IPs — topic 11's canonical pipeline, reused verbatim:
grep "Failed password" /var/log/auth.log \
  | awk '{print $(NF-3)}' \
  | sort | uniq -c | sort -rn | head
#     812 45.9.201.44
#     640 218.92.0.107
#     455 61.177.173.18
#   (grep the lines → awk out the IP field → sort → uniq -c to count →
#    sort -rn to rank → head for the worst offenders. This is topic 11.)
```

```bash
# ►►► THE LINE THAT ACTUALLY MATTERS: a SUCCESSFUL login. ◀◀◀
grep -E "Accepted (publickey|password)" /var/log/auth.log
#   Jul 12 09:14:02 web-1 sshd[2011]: Accepted publickey for deploy from
#     203.0.113.10 port 51224 ssh2: ED25519 SHA256:x9k...
#
#   ►► Failures are just noise — they FAILED. SUCCESSES are the whole game.
#      Read EVERY line here. Recognize every IP, every user, every key.
#      Anything on this list you do NOT recognize is an ACTIVE INCIDENT.
#      An 'Accepted password' line when you set PasswordAuthentication no?
#      Someone changed your config. That is an incident too.
```

Other forensic commands:

```bash
last            # successful logins, from /var/log/wtmp (who got in, from where, when)
lastb           # FAILED logins, from /var/log/btmp (needs sudo) — the brute-force log
lastlog         # last login time for EVERY user account
who             # who is logged in RIGHT NOW
w               # who's on + what they're running right now (spot the intruder's shell)
grep sudo /var/log/auth.log        # every sudo invocation — who ran what as root
grep "session opened" /var/log/auth.log   # every session start
```

---

## Firewall Posture — Close Everything, Open Three Ports

(Topic 29 — Firewalls covers ufw/iptables mechanics; this is the *policy*.)

### Deny by default, open only what you serve

```bash
sudo ufw default deny incoming      # ◀ the whole game: deny everything inbound
sudo ufw default allow outgoing     # your app needs to reach the world
sudo ufw allow 22/tcp               # SSH (or your custom port)
sudo ufw allow 80/tcp               # HTTP
sudo ufw allow 443/tcp              # HTTPS
sudo ufw enable
sudo ufw status verbose             # confirm ONLY 22/80/443 are open
```

That's it. Three ports. Everything else — including your database, your Redis, your app's raw port 3000, your metrics endpoint — is invisible from the internet.

### ►►► NEVER EXPOSE A DATABASE PORT TO THE INTERNET ◀◀◀

The **recurring headline breach** is not a sophisticated 0-day. It is an unauthenticated Redis / MongoDB / Elasticsearch / PostgreSQL bound to `0.0.0.0` with no password, found by a mass scanner in **hours**, dumped, deleted, and held for ransom. Shodan.io indexes hundreds of thousands of them at any moment.

```
   THE FIX IS TWO LAYERS, and you want BOTH:

   Layer 1 — bind the service to localhost or a private subnet, NOT 0.0.0.0:
     redis.conf:      bind 127.0.0.1 ::1          (or the private 10.x IP)
     postgresql.conf: listen_addresses = 'localhost'
     mongod.conf:     net: { bindIp: 127.0.0.1 }
     A service bound to 127.0.0.1 CANNOT be reached from off-box, period.

   Layer 2 — the firewall denies the port anyway (defense in depth):
     ufw default deny incoming already blocks 6379/5432/27017.
     If DB and app are on different hosts, allow ONLY the app's private IP:
       sudo ufw allow from 10.0.1.5 to any port 5432
```

### `ss -tulpn` — the audit command: "what am I ACTUALLY exposing?"

(Topic 28 — Ports and Sockets.) This is the single most important security-hygiene command. Run it after **every** deploy.

```bash
sudo ss -tulpn
Netid State  Local Address:Port   Process
tcp   LISTEN 127.0.0.1:5432       users:(("postgres",pid=811,fd=5))    ◀ good: localhost
tcp   LISTEN 0.0.0.0:3000         users:(("node",pid=2044,fd=20))      ◀◀◀ BAD!
tcp   LISTEN 0.0.0.0:22           users:(("sshd",pid=901,fd=3))        ◀ expected
tcp   LISTEN 0.0.0.0:6379         users:(("redis-server",pid=705))     ◀◀◀ CATASTROPHE
```
- `0.0.0.0:` = listening on **all interfaces** = reachable from the internet (if the firewall lets the port through).
- `127.0.0.1:` = localhost only = safe.
- **The discipline:** run `ss -tulpn` after every deploy and ask, for each `0.0.0.0` line, "do I *intend* for the whole internet to reach this?" Node on 3000 should be behind nginx (bind it to `127.0.0.1`). Redis on `0.0.0.0` is a five-alarm fire.

### ►►► THE DOCKER-BYPASSES-UFW TRAP (restated, because it WILL get you) ◀◀◀

(Topic 29.) This is the **most likely way you, specifically, accidentally expose a database.** Docker writes its own iptables rules **below** ufw's chain. So this:

```bash
docker run -d -p 5432:5432 postgres      # ◀ "-p" punches straight through ufw
```

...makes Postgres reachable from the **entire internet**, and `sudo ufw status` will cheerfully show `5432 DENY` — a lie. ufw never sees the packet; Docker's DNAT rule in the `nat` table redirects it before ufw's `INPUT` chain runs. The fixes:

```bash
docker run -d -p 127.0.0.1:5432:5432 postgres   # ◀ bind the published port to localhost
# or don't publish it at all — use a Docker network so only other containers reach it:
docker run -d --network app-net postgres         # no -p at all; app container reaches it by name
# and ALWAYS verify with the ground truth, not ufw status:
sudo ss -tulpn | grep 5432
```

**`ss -tulpn` is the ground truth. `ufw status` can lie when Docker is involved.**

### Cloud security groups — the second layer

Your cloud provider's security group (AWS SG, DO firewall, GCP firewall) is a firewall **outside** the box that a compromised host **cannot** disable from the inside. Set it to the same policy (allow 22/80/443 inbound, ideally 22 only from your office IP). It catches the Docker-ufw bypass too, since it filters before the packet ever reaches the instance. **Belt and suspenders: host firewall + cloud security group.**

---

## Patching — The Boring Fix for the Boring Breach

An unpatched, publicly-known CVE is one of the top ways boxes get owned. The fix is one command; the discipline is running it.

```bash
sudo apt update && sudo apt list --upgradable    # what's out of date?
sudo apt upgrade                                  # apply it
cat /var/run/reboot-required                      # did a kernel/libc update need a reboot?
#   *** System restart required ***   ← if this file exists, you're running old code in RAM
```

### unattended-upgrades — automate security patches

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades   # enable it
# Config: /etc/apt/apt.conf.d/50unattended-upgrades — by default applies the
# "-security" pocket only. You can enable automatic reboots (at a set time):
#   Unattended-Upgrade::Automatic-Reboot "true";
#   Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

**The honest trade-off:** automatic reboots at 4am can interrupt service (mitigate with a load balancer + rolling restarts / graceful shutdown, topic 33). But the alternative — a box running a **known-vulnerable** OpenSSL for three months because nobody scheduled a maintenance window — is worse. For a single small server, enable security auto-updates; for a fleet, patch through your deploy pipeline. Either way: **do not let security patches rot.**

### needrestart — the subtle one people miss

Updating a shared library (like OpenSSL) on disk does **not** update the copy already **loaded in RAM** by running services. Your `sshd`, `nginx`, and `node` are still executing the **old, vulnerable** library until you restart them.

```bash
sudo apt install needrestart
sudo needrestart          # lists every service still running old library code, offers to restart
```

`apt upgrade` runs `needrestart` automatically on modern Ubuntu. The lesson: **"I updated OpenSSL" is not done until the services holding the old one in memory have been RESTARTED.** A patched file on disk that nothing reloaded protects nobody.

### Your npm dependencies are a BIGGER attack surface than the OS

Sobering truth: a typical Node app has **hundreds** of transitive dependencies, any of which can run arbitrary code (a `postinstall` script) on `npm install`, as your user, with access to your `.env`. This is a far larger and more actively-attacked surface than the base OS.

```bash
npm audit                    # known CVEs in your dependency tree
npm audit fix                # patch what's safely patchable
npm ci                       # ◀ install EXACTLY the lockfile — reproducible, no surprise
                             #   version drift. Use this in CI/deploy, NOT `npm install`.
```
- **Commit your lockfile** (`package-lock.json`) and deploy with `npm ci`, so a dependency can't silently jump to a compromised new version between build and deploy.
- **Automate it:** Dependabot or Renovate to get PRs for dependency updates.
- **Never run `npm install` as root**, and never as root *inside* a Dockerfile — a malicious postinstall then runs as root. Use `USER node` (below).
- Consider `npm ci --ignore-scripts` in CI when you don't need native builds, to neuter postinstall entirely.

---

## Secrets Hygiene — Protecting the Actual Prize

(Topics 14 — Environment Variables, 32 — Deploy Permissions.) Everything else in this doc exists to protect these. Leak one and the hardening was theatre.

```
NEVER put a secret...

  ✗ in git                     → git history is FOREVER. `git rm` + commit does NOT
                                 remove it — it's in every past commit, on every clone,
                                 in every fork, and being scraped by bots watching GitHub
                                 in real time. A committed-then-deleted secret is STILL
                                 LEAKED. You must ROTATE it (issue a new one, revoke the
                                 old), not just delete it. Deletion is cosmetic.

  ✗ in a Dockerfile ENV/ARG    → baked into an image LAYER. `docker history <image>`
                                 and `docker inspect` reveal it. Anyone who pulls the
                                 image has it. Use runtime env or a secrets mount instead.

  ✗ on the command line        → visible to EVERY user via `ps aux` and
                                 /proc/PID/environ / /proc/PID/cmdline. `node app.js
                                 --db-pass=hunter2` leaks to every process on the box.

  ✗ world-readable on disk     → chmod 600 .env (owner read/write only), or 640 with a
                                 dedicated group. NEVER 644. Verify: ls -l .env
```

```bash
chmod 600 /srv/app/.env                 # owner-only. -rw-------
chown deploy:deploy /srv/app/.env
git rm --cached .env && echo ".env" >> .gitignore   # if it ever got tracked
# If a secret was EVER committed: ROTATE it now. Then optionally scrub history
# (git filter-repo / BFG) — but assume it's already stolen. Rotation is the real fix.
```

**Rotation is a first-class operation, not an emergency.** Rotate on any suspected leak, on employee offboarding, and on a schedule. A secret you cannot rotate quickly is a liability.

---

## Least Privilege — Recap for Production

(Topics 32 — Deploy Permissions, 33 — Node in Production.) The goal: if an attacker gets a shell in your app, they get as little as possible — ideally not root, ideally not even a useful filesystem.

```bash
# Run the app as a dedicated NON-ROOT, NON-LOGIN user (topic 33):
sudo useradd --system --no-create-home --shell /usr/sbin/nologin appuser
#   systemd unit: User=appuser  (topic 22). If the app is popped, the attacker is
#   'appuser', not root — they can't touch /etc, can't install packages, can't
#   read other users' files.
```

- **The `docker` group == root** (topic 32). Adding a user to `docker` lets them run `docker run -v /:/host ...` and read/write the entire host filesystem as root. **The docker group is root-equivalent — treat membership as granting full root.**
- In containers, drop everything you don't need:
```bash
docker run \
  --user node \                # ◀ don't run as root inside the container (USER node in Dockerfile)
  --cap-drop=ALL \             # drop ALL Linux capabilities, add back only what's needed
  --read-only \                # root filesystem is read-only; attacker can't write a payload
  --security-opt=no-new-privileges \   # block setuid escalation inside the container
  myapp
```
- **No `--privileged`** (it disables nearly all container isolation — a `--privileged` container is trivially a host root shell).
- **Never mount `/var/run/docker.sock`** into a container (topic 28/32) — the Docker socket **is** the Docker daemon's root API. A container with the socket can start a new privileged container mounting the host's `/`. It is a full host takeover in one bind mount.

---

## Detection & Audit — Finding the Persistence

If they got in, they left a way back. (Kill-chain step 6.) These are the classic persistence spots — sweep them.

```bash
# 1. Cron jobs for EVERY user + system cron (topic 20):
for u in $(cut -f1 -d: /etc/passwd); do echo "== $u =="; crontab -l -u "$u" 2>/dev/null; done
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/crontab
#   ►► An unexplained job curl-ing a script and piping to bash = a backdoor.

# 2. Unexpected systemd units and TIMERS (a common modern persistence spot):
systemctl list-units --type=service --state=running
systemctl list-timers --all
ls -la /etc/systemd/system/ ~/.config/systemd/user/
#   ►► A service/timer you don't recognize, or one running a script in /tmp, is a red flag.

# 3. ►► UNKNOWN SSH KEYS — a top-3 real persistence mechanism ◀◀
#    An attacker appends THEIR public key to authorized_keys → permanent silent access.
sudo find / -name "authorized_keys" 2>/dev/null -exec ls -la {} \; -exec cat {} \;
#   ►► Read EVERY key in EVERY user's ~/.ssh/authorized_keys. One you didn't add = owned.

# 4. Unexpected LISTENING ports (a backdoor shell, a crypto-miner's C2):
sudo ss -tulpn
#   ►► A process listening on a high port you don't recognize = investigate NOW.

# 5. ►► SETUID BINARY SWEEP (topic 05) — a classic root escalation/backdoor:
sudo find / -perm -4000 -type f 2>/dev/null
#   ►► setuid binaries run as their OWNER (usually root) regardless of who runs them.
#      A LEGIT list is short & stable (sudo, passwd, mount, su, ping...). A NEW or
#      UNEXPECTED setuid binary — especially in /tmp, /home, /dev/shm — is a red flag.
#      Baseline this list on a fresh box and diff against it later.

# 6. Unexpected USERS — especially DUPLICATE UID 0 (a hidden root account):
awk -F: '($3 == 0) {print $1}' /etc/passwd
#   ►► Should print ONLY "root". A SECOND uid-0 user IS a root backdoor.
awk -F: '($3 >= 1000) {print $1, $3}' /etc/passwd   # review human accounts

# 7. Who's been logging in, and unusual OUTBOUND connections (exfil / C2):
last -20
sudo ss -tp                 # ESTABLISHED connections + owning process — spot exfil
```

**Automated tooling (know the names):**
- **Lynis** — `sudo lynis audit system` — a fast automated hardening audit that scores your box and lists concrete fixes. Great starting point.
- **auditd** — the kernel audit framework; logs security-relevant syscalls (file access, exec, config changes) for after-the-fact forensics. Heavier; mention-level.
- **AIDE / Tripwire** — file-integrity monitoring. Baseline the hashes of `/etc`, `/bin`, `/usr/bin` on a clean box; they alert when a system binary changes (i.e., when a rootkit replaces `ls` or `sshd`).

---

## THE INCIDENT-RESPONSE MINI-PLAYBOOK (if you think you're owned)

```
┌────────────────────────────────────────────────────────────────────────┐
│  DO NOT JUST REBOOT.                                                     │
│  A reboot destroys volatile evidence (running processes, network         │
│  connections, memory) AND the malware almost certainly has persistence   │
│  (cron/systemd/SSH key) that reinstalls it on boot. You learn nothing    │
│  and fix nothing.                                                        │
└────────────────────────────────────────────────────────────────────────┘

  1. ISOLATE FIRST (contain the bleeding, before you investigate):
       • Cloud SECURITY GROUP → deny all inbound/outbound except your IP.
         (Do it at the SG, not on the box — the box may be compromised and
          its firewall can't be trusted.)
       • This stops exfiltration and C2 while preserving the box for forensics.

  2. SNAPSHOT THE DISK for forensics (cloud snapshot of the volume) BEFORE
       you change anything. You want evidence, and you may need to prove
       what was accessed (legal/compliance).

  3. INVESTIGATE the persistence spots above (cron, systemd, authorized_keys,
       setuid, uid-0 users, listening ports, last/lastb, outbound connections).
       Understand HOW they got in so you can close it in the rebuild.

  4. ►►► THE HARD TRUTH: YOU CANNOT RELIABLY CLEAN A ROOTED BOX. ◀◀◀
       Once an attacker had root, you can never again trust a single binary
       on that disk — `ls`, `ps`, `sshd`, even the kernel could be trojaned to
       hide the intruder. "Removing the malware" is a fantasy; you can only
       remove the malware you can SEE, and the whole point of a rootkit is you
       can't see it. So:
         → REBUILD FROM A KNOWN-GOOD IMAGE. Fresh OS, fresh install, restore
           only DATA (never binaries) from a backup that predates the breach.

  5. ►►► ROTATE EVERY CREDENTIAL THE BOX HAD ACCESS TO. ◀◀◀
       Assume everything on that box was stolen — because it was:
         • SSH keys (and remove the old pubkeys from every other box)
         • ALL database passwords
         • API tokens, third-party keys, webhook secrets
         • Cloud IAM keys / instance-role credentials (a huge one — a
           compromised instance can assume its IAM role)
         • Session-signing secrets (invalidate all user sessions)
       A rebuilt box with the OLD stolen credentials is still owned.
```

---

## Common Mistakes

### Mistake 1 — "Nobody knows my server's IP, so I'm fine"

**Wrong:** security through obscurity — an unadvertised IP is safe.
**Right:** the entire IPv4 space (4 billion addresses) is scanned continuously by everyone. `masscan` sweeps the whole internet in minutes.

**Root cause:** attacks are not targeted; they are **exhaustive and automated**. Boot a fresh VM and `tail -f /var/log/auth.log` — bot logins start within **minutes**, before you've told a soul the IP exists. You are not hidden. You are a door on a street that gets walked every second.

**Fix/Prevention:** assume you are being attacked from the moment of boot, because you are. Harden *before* you expose, not after.

---

### Mistake 2 — Deleting a leaked secret from git instead of rotating it

**Wrong:** committed AWS keys → `git rm .env`, commit "remove secret," breathe easy.
**Right:** the secret is in git **history** — every past commit, clone, and fork. It is already scraped (bots watch public GitHub pushes in real time and use leaked keys within *seconds*). **Rotate it: revoke the old credential, issue a new one.** Scrubbing history (BFG/filter-repo) is optional cleanup; it does **not** un-leak what was already public.

**Root cause:** git is an append-only history, not a current-state store. "Deleting" a file adds a new commit; the old blob with the secret is still reachable forever.

**Prevention:** a pre-commit secret scanner (gitleaks, git-secrets); `.env` in `.gitignore` from day one; secrets in a manager (Vault, AWS Secrets Manager, SSM), never in the repo.

---

### Mistake 3 — Firewalling a DB port with ufw while Docker publishes it anyway

**Wrong:** `ufw deny 5432` + `docker run -p 5432:5432 postgres` → "Postgres is firewalled."
**Right:** Docker's iptables DNAT rules run **before** ufw's INPUT chain (topic 29), so `-p 5432:5432` exposes Postgres to the whole internet and `ufw status` shows a reassuring lie.

**Root cause (kernel level):** ufw appends rules to the `filter` table's `INPUT` chain; Docker inserts rules into the `nat` table (`DOCKER` chain) that redirect the packet to the container before it ever reaches `INPUT`. Different tables, different order — ufw never sees the packet.

**Fix:** `-p 127.0.0.1:5432:5432` (bind published port to localhost), or use a Docker network with no `-p` at all. **Verify with `ss -tulpn`, never with `ufw status`.**

---

### Mistake 4 — Adding the deploy user to the docker group for convenience

**Wrong:** "the deploy user needs to run `docker` without sudo" → `usermod -aG docker deploy`.
**Right:** the `docker` group is **root-equivalent** (topic 32). A docker-group member runs `docker run -v /:/host -it alpine chroot /host` and now has a root shell on the host.

**Root cause:** the Docker daemon runs as root and its socket has no finer-grained authorization — anyone who can talk to it can ask it to mount the host's `/` and run as root. Group membership = full daemon access = full root.

**Fix/Prevention:** treat `docker` group membership exactly like handing out root. Prefer rootless Docker, or a tightly-scoped sudoers rule for the specific docker commands the deploy needs (topic 32), and audit `getent group docker` regularly.

---

### Mistake 5 — Locking yourself out while hardening SSH

**Wrong:** edit `sshd_config`, set `PasswordAuthentication no` before your key works (or fat-finger a directive), `systemctl restart ssh`, close the terminal. Next login: `Permission denied (publickey)`. You are locked out of production.
**Right:** `sshd -t` to validate syntax → reload → **open a NEW session and confirm it works** → only *then* close the old one. Set up and test your key *before* disabling passwords.

**Root cause:** your existing SSH connection keeps its already-authenticated session even after a broken config reload — the broken config only affects the **next** login. Closing the good session first removes your only way back in.

**Prevention:** the two-terminal rule, always. And know your cloud **serial/rescue console** *before* you need it — it's the out-of-band door that still opens when sshd won't.

---

## Hands-On Proof

```bash
# PROVE IT: the internet is attacking you right now
sudo grep "Failed password" /var/log/auth.log | wc -l    # thousands, on any public box
sudo lastb | head -20                                     # the failed-login log, live

# PROVE IT: see exactly what you're exposing to the world
sudo ss -tulpn | grep '0.0.0.0'   # every service reachable from outside. Audit each one.

# PROVE IT: a secret on the command line leaks to every process
sleep 300 --pass=SUPERSECRET &     # (sleep ignores it — we just want it in the args)
cat /proc/$!/cmdline | tr '\0' ' '; echo    # ►► "sleep 300 --pass=SUPERSECRET"
kill %1
#   Any user on the box could read that. THIS is why secrets never go on argv.

# PROVE IT: git history keeps "deleted" secrets forever
mkdir /tmp/leak && cd /tmp/leak && git init -q
echo "API_KEY=sk_live_abc123" > .env && git add .env && git commit -qm "add"
git rm -q .env && git commit -qm "remove secret"    # "deleted" it
git log -p --all | grep sk_live                     # ►► STILL THERE. In history. Forever.

# PROVE IT: find every setuid binary (topic 05) — baseline this list
sudo find / -perm -4000 -type f 2>/dev/null
#   Legit: /usr/bin/sudo, /usr/bin/passwd, /usr/bin/su, mount... Anything else, question.

# PROVE IT: only root should have uid 0
awk -F: '($3 == 0){print $1}' /etc/passwd    # ►► must print ONLY "root"

# PROVE IT: validate sshd config WITHOUT risking a restart
sudo sshd -t && echo "config OK"    # syntax-checks; prints nothing if fine

# PROVE IT: your firewall default is deny
sudo ufw status verbose | grep Default    # ►► "deny (incoming), allow (outgoing)"
```

---

## Practice Exercises

### Exercise 1 — Easy

On any Linux box you control, run the **exposure audit** and account audit:

```bash
sudo ss -tulpn                                   # what's listening, on which interface?
sudo ufw status verbose                          # what's the firewall policy?
awk -F: '($3==0){print}' /etc/passwd             # any surprise uid-0 users?
sudo find / -perm -4000 -type f 2>/dev/null      # the setuid baseline
```
For **every** `0.0.0.0` line in `ss` output, write down: what service it is, and whether you *intend* the whole internet to reach it. Any `0.0.0.0` database is a finding — fix it (bind to localhost).

---

### Exercise 2 — Medium

Harden SSH on a throwaway VM, **safely**:

1. Generate a key on your laptop (`ssh-keygen -t ed25519`), copy it up (`ssh-copy-id`), and confirm key login works.
2. Edit `sshd_config`: `PasswordAuthentication no`, `PermitRootLogin no`, `MaxAuthTries 3`, `AllowUsers <you>`.
3. `sudo sshd -t` (validate), then `sudo systemctl reload ssh`.
4. **Without closing your session**, open a second terminal and confirm you can still log in. Also confirm password login is now *refused* (`ssh -o PubkeyAuthentication=no you@host` → should fail).
5. Install fail2ban, put your own IP in `ignoreip`, enable the `sshd` jail, and confirm with `sudo fail2ban-client status sshd`.
6. **Prove the noise reduction:** `sudo grep "Failed password" /var/log/auth.log | wc -l` before and (a day later) after — and confirm no *successful* logins you don't recognize (`grep "Accepted" ...`).

---

### Exercise 3 — Hard (Production Simulation)

Run the full **new-server hardening checklist** (below) on a fresh Ubuntu cloud VM, from `create sudo user` to `test the restore`, and produce a short audit report proving each step:

1. Create a sudo user; disable password + root SSH; confirm with a fresh session.
2. `ufw` deny-in / allow 22,80,443; prove with `ufw status verbose` and `ss -tulpn`.
3. Deploy a Node app + Postgres. **Deliberately** run `docker run -p 5432:5432 postgres`, then use `ss -tulpn` to *prove* Postgres is exposed despite ufw. Fix it (`127.0.0.1:5432:5432`), and prove it's now local-only.
4. Install fail2ban + unattended-upgrades; show both active.
5. Run `sudo lynis audit system`; record the hardening score and pick three findings to remediate.
6. Take a backup of the DB, **destroy the box, rebuild from image, and restore the backup — and prove the data came back.** Write one sentence on why an untested backup is not a backup.

---

## Mental Model Checkpoint

1. **Within how long of booting a public server does automated attack traffic arrive, and what does that tell you about *when* to harden?**
2. **What single `sshd_config` change eliminates the entire brute-force attack class, and why does it work?**
3. **You `git rm` a committed API key and push. Are you safe? What is the ONLY real fix?**
4. **You ran `ufw deny 6379` but `ss -tulpn` shows Redis on `0.0.0.0:6379` and it was started with `docker run -p`. Explain what happened and how to fix it.**
5. **Why is `ss -tulpn` more trustworthy than `ufw status` for answering "what am I exposing?"**
6. **You confirm the box is compromised. Name the first action (before investigating) and explain why you must NOT just reboot.**
7. **After a confirmed breach you rebuild from a clean image. What critical step remains, and why is the rebuild worthless without it?**
8. **Why is the `docker` group equivalent to root?**

---

## Quick Reference Card

| Command / File | What It Does | Key Points |
|---|---|---|
| `/etc/ssh/sshd_config` | SSH server config | `PasswordAuthentication no`, `PermitRootLogin no`, `AllowUsers` |
| `sshd -t` | Validate sshd config syntax | Run BEFORE reload; never restart on an unvalidated config |
| `grep "Accepted" /var/log/auth.log` | **Successful** logins — the line that matters | Anything unrecognized = incident |
| `grep "Failed password" ... \| awk '{print $(NF-3)}' \| sort \| uniq -c \| sort -rn` | Top attacking IPs (topic 11 pipeline) | Shows the scale of the swarm |
| `last` / `lastb` / `lastlog` | Successful / failed / per-user last logins | `lastb` needs sudo |
| `fail2ban-client status sshd` | Show a jail's bans and stats | `set sshd unbanip 1.2.3.4` to recover |
| `/etc/fail2ban/jail.local` | Your fail2ban config | NEVER edit `jail.conf`; set `ignoreip`! |
| `ss -tulpn` | **What am I exposing?** — the audit command | `0.0.0.0`=public, `127.0.0.1`=safe; ground truth over `ufw status` |
| `ufw default deny incoming` + `allow 22,80,443` | Deny-by-default firewall | Docker `-p` bypasses it — verify with `ss` |
| `apt list --upgradable` / `unattended-upgrades` | See / auto-apply security patches | `/var/run/reboot-required`, run `needrestart` |
| `needrestart` | Services still running OLD library code | Patched-on-disk ≠ patched-in-memory |
| `npm audit` / `npm ci` | Dependency CVEs / reproducible install | Never `npm install` as root |
| `chmod 600 .env` | Owner-only secret file | Never 644; never in git; never in Dockerfile ENV |
| `find / -perm -4000 -type f` | Setuid-binary sweep (topic 05) | New/unexpected = red flag |
| `awk -F: '($3==0){print $1}' /etc/passwd` | uid-0 (root) accounts | Must be ONLY `root` |
| `lynis audit system` | Automated hardening audit + score | Great starting point |

---

## When Would I Use This at Work?

### Scenario 1: Standing up a new production server
You provision a fresh Ubuntu VM for the Node API. **Before** pointing DNS at it, you run the hardening checklist below top to bottom: sudo user, key-only SSH, root login off, ufw deny-in/allow-3, fail2ban, unattended-upgrades, app runs as `appuser`, Postgres bound to `127.0.0.1`, `.env` at 600. Fifteen minutes of checklist. You just closed all five of the boring holes that cause almost every breach — before the box ever faced the internet.

### Scenario 2: The 3am "is this a hack?" page
Your monitoring flags an unfamiliar outbound connection and CPU pinned at 100% (a miner). You `ss -tp` (the C2 connection + PID), `last` and `grep Accepted /var/log/auth.log` (how they got in — an `Accepted password` line means someone left password auth on), sweep cron / systemd timers / `authorized_keys` / setuid for persistence, snapshot the disk, isolate at the security group. Then — because you know a rooted box can't be cleaned — you **rebuild from image and rotate every credential**. You don't waste an hour "removing the virus."

### Scenario 3: The security review before a launch
Before a customer launch, you self-audit: `sudo lynis audit system` for a score and a punch-list; `ss -tulpn` to confirm only 22/80/443 face the world (and catch the Postgres someone exposed via `docker -p`); `npm audit` on the app; confirm secrets aren't in git (`git log -p | grep -i key`) and that `.env` is 600; verify backups **restore**. You fix the findings and walk into the launch review with evidence, not vibes — while being honest that this is a baseline, not a substitute for a professional pen-test.

---

## THE HARDENING CHECKLIST — A New Ubuntu Box

Run top to bottom on every new server. Each step has a one-line "why." This is the deliverable — the thing you actually *do*.

```
 1. Create a non-root sudo user            → never operate as root; get an audit trail
      adduser deploy && usermod -aG sudo deploy

 2. Set up SSH keys for that user          → key auth is unbrute-forceable
      ssh-copy-id deploy@host   (test it works BEFORE step 3!)

 3. Disable password + root SSH login      → kills the entire brute-force attack class
      sshd_config: PasswordAuthentication no; PermitRootLogin no; AllowUsers deploy
      sshd -t && systemctl reload ssh   → then TEST A NEW SESSION before closing this one

 4. Firewall: deny-in, allow 22/80/443     → the internet reaches only what you serve
      ufw default deny incoming; ufw allow 22,80,443/tcp; ufw enable

 5. Install fail2ban (+ ignoreip = you)    → auto-bans brute-forcers, cuts log noise
      apt install fail2ban; set ignoreip in jail.local; enable [sshd]

 6. Enable unattended-upgrades             → known-CVE patches apply themselves
      apt install unattended-upgrades; dpkg-reconfigure -plow unattended-upgrades

 7. Run the app as a NON-ROOT user         → a popped app can't become root
      useradd --system appuser; systemd User=appuser; USER node in Docker

 8. NO database ports exposed              → the #1 headline breach; don't be it
      bind 127.0.0.1 in redis/pg/mongo conf; verify with ss -tulpn (NOT ufw status)

 9. Secrets locked down                    → the actual prize; leak = game over
      chmod 600 .env; never in git/Dockerfile/argv; rotate on any suspicion

10. Automated backups — AND TEST RESTORE   → an untested backup is NOT a backup
      schedule dumps off-box; then actually RESTORE one and confirm the data returns

11. Monitoring & alerting                  → you can't respond to what you can't see
      alert on unfamiliar listening ports, failed-login spikes, disk full, new uid-0
```

**Scope, honestly:** this is a solid baseline for a small production Node server. It is **not** a compliance program (SOC 2 / PCI / ISO 27001), not a substitute for a professional security review or penetration test, and not sufficient for a high-value target (finance, health, large user data). It closes the boring holes that cause the overwhelming majority of real-world compromises — which, for most backends, is exactly the goal.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | — (this is the final topic) | You've reached the end. You can now operate a production Linux server with confidence. |
| **Builds on** | 24 — SSH in Depth | How SSH/keys work; this doc is how you lock SSH down |
| **Builds on** | 29 — Firewalls | ufw/iptables mechanics; this is the *policy* + the Docker-bypass trap |
| **Builds on** | 28 — Ports and Sockets | `ss -tulpn` — the "what am I exposing?" audit command |
| **Builds on** | 32 — Deploy Permissions | non-root users, sudoers, the docker-group-is-root trap, least privilege |
| **Builds on** | 33 — Node in Production | running as a non-root user, systemd hardening, secrets as env |
| **Builds on** | 23 — Logs | reading `/var/log/auth.log`; feeding your app's logs to a fail2ban jail |
| **Builds on** | 11 — Text Processing | the `grep \| awk \| sort \| uniq -c \| sort -rn` pipeline for top attacker IPs |
| **Builds on** | 05 — File Permissions | setuid bits — the setuid-binary sweep for escalation/persistence |
| **Builds on** | 14 — Environment Variables | secrets as env vars; why argv/Dockerfile ENV leak them |
| **Builds on** | 20 — Cron and Scheduling | cron as a persistence mechanism to sweep for |
| **Builds on** | 34 — Performance Investigation | same discipline: a checklist beats heroics; a miner shows up as pegged CPU |
| **Used by** | Every server you will ever run | Hardening is not a one-time task; it's an operating posture |
