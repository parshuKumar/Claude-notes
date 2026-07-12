# 20 — Cron and Scheduled Tasks

## ELI5 — The Simple Analogy

Imagine an **extremely literal-minded robot butler** who lives in your house.

Every single minute, on the minute, he wakes up, pulls out a laminated card, and reads every line on it. Each line says *when* to do a thing and *what* to do. If the current time matches a line, he does it. Then he goes back to sleep for the rest of the minute.

He is perfectly reliable. He is also **completely, catastrophically literal**:

- **He doesn't know where anything is.** You wrote "make coffee." He doesn't know where the coffee machine is, because he doesn't have your mental map of the house. He needs *"go to the kitchen, second cupboard on the left."* → **This is the `PATH` problem.** He has a nearly empty PATH and doesn't know where `node` lives, even though *you* can type `node` and it just works.

- **He always starts from the front hall**, never from wherever you happen to be standing. So "grab the file from the desk" fails, because *which* desk? → **This is the `CWD` problem.** Cron always starts in `$HOME`, so your relative paths and your `.env` file are nowhere to be found.

- **If something goes wrong, he writes you a polite note and posts it into a mailbox — a mailbox that was bricked over in 2003.** He is *certain* he told you. He did. Nobody has ever checked that mailbox. → **This is the silent-failure problem.** Cron mails you the output. Mail isn't configured. **Your job has been failing for three weeks and nobody knows.**

- **He never checks whether he's still doing the last job.** Told to mop every minute, and mopping takes five minutes, he will start a *new* mop every minute anyway. Within an hour there are sixty robots mopping, fighting, and the house is destroyed. → **This is the overlap problem.**

Cron is that butler. He is not smart. He is *punctual*. Every failure in this document comes from expecting him to be smart.

---

## Where This Lives in the Linux Stack

```
Hardware (the CMOS clock, the CPU timer interrupt)
  │
  └── KERNEL
       │   • Keeps wall-clock time (CLOCK_REALTIME)
       │   • Timer interrupt fires; kernel wakes sleeping processes
       │   • fork() + execve() — cron's ONLY mechanism for running your job
       │
       └── System Calls (nanosleep/timerfd, fork, execve, setuid, open, dup2)
            │
            └── C Library (glibc)
                 │
                 └── ┌─────────────────────────────────────────────────────┐
                     │  cron / crond  — a plain userspace DAEMON.  ◀◀◀     │
                     │  NOT part of the kernel. Just a long-running        │
                     │  program (PID ~900) doing:                          │
                     │     while(true) { sleep_until_next_minute();        │
                     │                   for job in crontabs:              │
                     │                       if matches(now): fork+exec }  │
                     │                                                     │
                     │  systemd timers — the modern replacement, run by    │
                     │  PID 1 itself.                            ◀◀◀       │
                     └─────────────────────────────────────────────────────┘
                          │
                          └── fork() + execve("/bin/sh", "-c", "<your command>")
                               │   ★ with a NEARLY EMPTY environment
                               │   ★ with CWD = $HOME
                               │   ★ with stdout/stderr piped to SENDMAIL
                               │
                               └── YOUR SCRIPT — which then fails, and you
                                   have no idea why.  ◀◀◀ THIS TOPIC
```

**Why it lives there:** cron is not privileged magic. It is an ordinary daemon that the init system starts at boot. Everything it does, you could write in 50 lines of C. It has **no relationship to your shell**, which is precisely why none of your shell's setup — PATH, nvm, aliases, `.bashrc`, `.env` — exists inside a cron job.

---

## What Is This?

`cron` is a **daemon** — a background process — that wakes up once every minute, reads a set of table files (**crontabs**), and for every line whose time specification matches the current minute, it `fork()`s and `exec()`s the command.

That's it. It is a `while` loop with a clock and a `fork()`.

**systemd timers** are the modern alternative: instead of one daemon parsing text tables, PID 1 (systemd) schedules the activation of a `.service` unit. Same idea, but with real logging, dependency ordering, resource limits, per-job status, and — crucially — the ability to run a job that was **missed** while the machine was off.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Cron's environment is nearly empty | `node: command not found` — for a command that works perfectly when you type it. You'll lose an hour to this. Everyone does |
| Cron's CWD is `$HOME` | `require('./config')` throws, `dotenv` finds no `.env`, your relative paths silently read the wrong file |
| Cron output is **mailed**, and mail isn't configured | **Your nightly backup has silently failed for three weeks.** You will discover this on the day you need the backup |
| `%` is special in crontab | `date +%Y-%m-%d` breaks your job in a way the error message will never explain |
| DOM/DOW is an **OR** | Your "1st of the month" job also runs every Monday. You will not notice for a year |
| No overlap protection | A job that takes 6 minutes, scheduled every minute, forks 6 copies of itself into a death spiral that eats the box |
| Cron uses the **system** timezone | Your "3am, low traffic" backup fires at 10pm, during peak |
| `crontab -r` is next to `crontab -e` | You will one day wipe every scheduled job on a production box, with **no confirmation prompt** and no undo |

---

## The Physical Reality

### Where crontabs actually live on disk

```
1. USER CRONTABS   /var/spool/cron/crontabs/<user>  (Debian/Ubuntu; RHEL/Alpine: /var/spool/cron/<user>)
     mode 600, group `crontab` — you cannot read another user's. ★ NEVER edit by hand; ALWAYS `crontab -e`
     (it validates syntax AND touches the dir so cron re-reads). Format: 5 time fields + COMMAND, run as owner.
        0 3 * * *  /srv/app/backup.sh

2. SYSTEM CRONTABS   /etc/crontab   /etc/cron.d/myapp   ← put package/deploy jobs here
     ★★ EXTRA 6th FIELD = THE USER TO RUN AS ★★.  Omit it → cron runs `deploy` as a command → silent
     failure (the #1 error in cron.d). Files must be root-owned, mode 644, filename with NO dots.
        0 3 * * *  deploy  /srv/app/backup.sh
                   ▲▲▲▲▲▲ the user

3. run-parts DIRS   /etc/cron.{hourly,daily,weekly,monthly}/   ← just drop an executable in (no time syntax)
     ★★ TRAP: run-parts IGNORES any filename containing a DOT ★★, and it must be chmod +x.
        /etc/cron.daily/backup.sh  ✗ silently never runs        /etc/cron.daily/backup  ✓ runs
     VERIFY:  run-parts --test /etc/cron.daily   (lists exactly what WOULD run)

4. ACCESS CONTROL   /etc/cron.allow (if it exists, ONLY listed users may use cron) then /etc/cron.deny.
     Neither present → (Debian/Ubuntu) everyone may use cron.
```

### What cron does, every 60 seconds, forever

```
the cron daemon (PID 912):
  sleep until top of next minute → re-read crontabs if mtime changed
  for each job line: (min,hour,dom,mon,dow) match NOW?  no → skip ;  yes → fork():
      child: setuid(job_owner)
             setenv(HOME, SHELL, PATH, LOGNAME)          ← ★ THAT'S ALL — 4 vars
             chdir($HOME)                                ← ★ NOT your app dir
             pipe stdout+stderr → sendmail               ← ★ THE BLACK HOLE (no MTA = discarded)
             execve("/bin/sh", ["-c", "<your command>"])
  syslog: "CRON[9182]: (deploy) CMD (...)"  ← ★ RECORDS ONLY THAT IT STARTED. No exit code.
                                              A job that instantly dies looks IDENTICAL to success.
  go back to sleep. Never waits. Never checks. Never cares if the last copy is still running.
```

**Burn those starred lines in.** Every classic cron failure is one of them.

---

## How It Works — Step by Step

```
COMMAND (in your crontab):  */5 * * * * /srv/app/bin/job.sh >> /var/log/job.log 2>&1

CRON DAEMON DOES:  wakes at :00, :05, :10 … — every minute the field spec matches
KERNEL DOES:       fork() → a child cron process
CHILD DOES:        setuid/setgid(job owner) → chdir($HOME) → sets HOME/SHELL/PATH/LOGNAME
                   → sets up fds: open("/var/log/job.log", O_APPEND) as fd 1, dup2(1,2) (topic 19)
SYSCALLS:          fork() → setresuid() → chdir() → open() → dup2() → execve("/bin/sh",["-c",cmd])
SHELL (sh -c):     parses the string; runs /srv/app/bin/job.sh (an ordinary program from here)
OUTPUT:            stdout+stderr → fd 1/2 → /var/log/job.log  (NOT mailed, because we redirected)
AFTER EXIT:        cron writes ONE syslog line ("CMD (...)"), discards the exit code, sleeps.
```

The whole daemon is the loop drawn above: `sleep → match → fork → setuid → chdir → exec → log-that-it-started → sleep`. There is no supervision, no retry, no success/failure tracking. Everything downstream — the empty environment, the `$HOME` cwd, the mailed-and-lost output — falls out of that `fork/setuid/chdir/exec` sequence.

---

## Crontab Syntax — The Five Fields

```
 ┌───────────── MINUTE          (0 - 59)
 │ ┌─────────── HOUR            (0 - 23)      ← 24-hour clock. No AM/PM. No 24.
 │ │ ┌───────── DAY OF MONTH    (1 - 31)      ← starts at 1, not 0
 │ │ │ ┌─────── MONTH           (1 - 12)      ← or jan,feb,mar … (case-insensitive)
 │ │ │ │ ┌───── DAY OF WEEK     (0 - 7)       ← ★ BOTH 0 AND 7 ARE SUNDAY ★
 │ │ │ │ │                                       or sun,mon,tue …
 │ │ │ │ │
 * * * * *  command to be executed
 ─────────  ──────────────────────
   WHEN            WHAT
                   (runs via `/bin/sh -c`, as the crontab's owner,
                    with CWD=$HOME and a nearly empty environment)
```

### The operators

| Operator | Meaning | Example | Fires |
|---|---|---|---|
| `*` | every value | `* * * * *` | every minute of every day |
| `N` | exactly N | `30 4 * * *` | 04:30 daily |
| `,` | a **list** | `0 9,13,17 * * *` | 09:00, 13:00, 17:00 |
| `-` | a **range** | `0 9-17 * * 1-5` | on the hour, 9am–5pm, Mon–Fri |
| `/` | a **step** | `*/5 * * * *` | every 5 minutes (:00,:05,:10…) |
| `/` on a range | step within range | `0 8-18/2 * * *` | every 2 hours from 08:00 to 18:00 |
| `,` + `-` + `/` | combine freely | `0,30 9-17 * * 1-5` | :00 and :30, 9–5, weekdays |

> ⚠ **`*/7` does NOT mean "every 7 minutes."** Steps are computed over the field's **full range** starting at its minimum, and the range **does not wrap**. `*/7` in the minute field fires at :00, :07, :14, :21, :28, :35, :42, :49, :56 — and then **:00 again, only 4 minutes later.** Steps only behave intuitively when they divide the range evenly (`*/1 /2 /3 /4 /5 /6 /10 /12 /15 /20 /30` for minutes).

### ★ The DOM/DOW OR-not-AND trap ★

This is the single most surprising rule in cron, and it is in the POSIX spec:

> **If BOTH day-of-month and day-of-week are restricted (neither is `*`), cron runs the job when EITHER matches — not both.**

```
   0 3 1 * *        →  3am on the 1st.                        (DOW is *, no ambiguity)
   0 3 * * 1        →  3am every Monday.                      (DOM is *, no ambiguity)

   0 3 1 * 1        →  ★ 3am on the 1st  ★OR★  3am EVERY MONDAY ★
                       NOT "the 1st, but only if it's a Monday."
                       This fires ~5 times a month, not once.
                       ↑ THIS IS ALMOST NEVER WHAT ANYONE WANTS.

   0 3 13 * 5       →  3am on the 13th, AND 3am every Friday.
                       (Not "Friday the 13th"!)
```

**Every other field ANDs. These two OR.** The historical reason: it lets you say "the 1st of the month **and** every Sunday" in one line. The practical result: people write `0 3 1 * 1` meaning "first Monday" and get 5 runs instead of 1.

**How to actually get "first Monday of the month":** you can't, in pure cron. Restrict DOM in cron and test DOW in the script:
```bash
0 3 1-7 * *  [ "$(date +\%u)" = "1" ] && /srv/app/monthly.sh
#      ▲▲▲    ▲                  ▲
#      │      │                  └── \% ESCAPED (see the % trap below!)
#      │      └── only days 1-7 can contain the first Monday
#      └── DOW left as * so no OR
```

### The `@` shortcuts

| Shortcut | Equivalent | Fires |
|---|---|---|
| `@yearly` / `@annually` | `0 0 1 1 *` | midnight, Jan 1 |
| `@monthly` | `0 0 1 * *` | midnight, 1st of month |
| `@weekly` | `0 0 * * 0` | midnight, Sunday |
| `@daily` / `@midnight` | `0 0 * * *` | midnight, every day |
| `@hourly` | `0 * * * *` | top of every hour |
| `@reboot` | — | **once, when cron starts at boot** |

> ⚠ **`@reboot` is not a process supervisor.** It fires once, when the cron daemon starts. If your process crashes at 3am, `@reboot` will not restart it. **Use a systemd service with `Restart=always` (topic 22).** `@reboot` is fine for one-shot boot chores (warm a cache, mount something); it is *not* how you keep a Node app alive.

> ⚠ **`@daily` means midnight, and midnight is when EVERY server on earth runs its jobs.** If you have 200 instances all `@daily`-ing a backup to the same S3 bucket at 00:00:00, you have built a thundering herd. Stagger them, or use a systemd timer with `RandomizedDelaySec=`.

---

## Exact Syntax Breakdown

```
crontab -e
│       │
│       └── -e = EDIT. Opens YOUR crontab in $EDITOR (or $VISUAL), in a TEMP file.
│              On save it VALIDATES THE SYNTAX, and only THEN installs it to
│              /var/spool/cron/crontabs/<you>, and touches the dir so cron re-reads.
│              ★ ALWAYS use -e. Never `vim /var/spool/cron/crontabs/you` — you'd
│                skip the validation AND cron might not notice the change.
└── crontab — the program that manages crontab files. Setuid-root, group `crontab`.

  crontab -l              # LIST your crontab to stdout
  crontab -l > ~/cron.bak # ★ BACK IT UP. DO THIS BEFORE EVERY EDIT. See -r. ★
  crontab -r              # ★★ REMOVE THE ENTIRE CRONTAB. NO CONFIRMATION. NO UNDO. ★★
  crontab -u deploy -l    # another user's crontab (root only)
  crontab myfile          # REPLACE your whole crontab with the contents of myfile
```

### ★ `crontab -r` — the classic disaster ★

```
                    e   ← EDIT my crontab
                    │
        q   w   e   r   t   y      ← LOOK AT YOUR KEYBOARD.
                        │
                        r   ← DELETE MY ENTIRE CRONTAB, INSTANTLY,
                                WITH NO CONFIRMATION AND NO UNDO.

   `crontab -r` and `crontab -e` are ONE KEY APART.
   There is no "are you sure?". There is no trash can. There is no backup.
   Every crontab on that box, for that user, is GONE.
```

**Defend yourself. Three layers:**
```bash
# 1. ALWAYS back up before editing:
crontab -l > ~/crontab.$(date +\%F).bak

# 2. Alias -r to the interactive version. (-i prompts before deleting.)
echo "alias crontab='crontab -i'" >> ~/.bashrc      # → topic 14/15
#   `crontab -i -r` prompts: "crontab: really delete deploy's crontab?"
#   NOTE: -i only takes effect ALONGSIDE -r. It changes nothing else.

# 3. THE REAL ANSWER: put jobs in /etc/cron.d/ or a systemd timer, IN GIT.
#    A file in version control cannot be destroyed by a typo. A spool file can.
```

```
0 3 * * * cd /srv/app && /usr/bin/flock -n /tmp/backup.lock ./backup.sh >> /var/log/backup.log 2>&1
│ │ │ │ │ │             │                  │                │            │       │
│ │ │ │ │ │             │                  │                │            │       └── 2>&1: send STDERR
│ │ │ │ │ │             │                  │                │            │           to the SAME place
│ │ │ │ │ │             │                  │                │            │           as stdout (topic 12/19).
│ │ │ │ │ │             │                  │                │            │           WITHOUT THIS, ERRORS
│ │ │ │ │ │             │                  │                │            │           GO TO THE MAIL BLACK HOLE.
│ │ │ │ │ │             │                  │                │            └── >> APPEND to a log
│ │ │ │ │ │             │                  │                └── the actual script
│ │ │ │ │ │             │                  └── the LOCK FILE. flock takes an exclusive
│ │ │ │ │ │             │                      lock on it (flock(2) on an fd — topic 19!)
│ │ │ │ │ │             └── -n = NON-BLOCKING. If the lock is already held (i.e. the
│ │ │ │ │ │                 previous run is STILL GOING), exit immediately instead of
│ │ │ │ │ │                 queueing. ★ THIS IS YOUR OVERLAP PROTECTION. ★
│ │ │ │ │ └── `cd` FIRST — cron starts in $HOME, not your app dir. (Failure #2.)
│ │ │ │ └── DOW: * = any day of the week
│ │ │ └── MONTH: * = every month
│ │ └── DOM: * = every day of the month
│ └── HOUR: 3 → 03:00 ★ IN THE SYSTEM'S TIMEZONE — usually UTC on a server! ★
└── MINUTE: 0
```

```
env -i /bin/sh -c '/srv/app/backup.sh'
│   │  │       │   │
│   │  │       │   └── your command, exactly as cron would run it
│   │  │       └── -c = run this string. ★ CRON USES `sh -c`, NOT BASH. ★
│   │  │             So bashisms ([[ ]], arrays, `source`) will FAIL under cron
│   │  │             on Debian/Ubuntu, where /bin/sh is DASH, not bash.
│   │  └── /bin/sh — NOT your login shell
│   └── -i = IGNORE THE ENVIRONMENT. Start with a COMPLETELY EMPTY env.
└── env — ★★ THE SINGLE BEST CRON DEBUGGING TRICK IN EXISTENCE. ★★
    This reproduces cron's bare environment IN YOUR TERMINAL, so you see the
    "command not found" immediately, interactively, instead of at 3am in silence.
```

---

## ★★ THE FOUR CLASSIC CRON FAILURES ★★

This section is the heart of the document. Every one of these will bite you.

---

### FAILURE 1 — PATH: `node: command not found`

**The symptom:** the command works perfectly when you type it. It fails in cron.

```bash
$ which node
/home/deploy/.nvm/versions/node/v20.11.0/bin/node        # ← installed via nvm

$ node /srv/app/job.js
✓ works fine

# crontab:
*/5 * * * * node /srv/app/job.js
# → /bin/sh: 1: node: not found
```

**Root cause — cron's environment is nearly EMPTY.** When cron forks your job, it sets **only**:

```
   YOUR INTERACTIVE SHELL              CRON'S CHILD PROCESS
   ──────────────────────              ────────────────────
   PATH=/home/deploy/.nvm/versions/    PATH=/usr/bin:/bin      ★ THAT'S IT ★
        node/v20.11.0/bin:/usr/local/  SHELL=/bin/sh
        sbin:/usr/local/bin:/usr/      HOME=/home/deploy
        sbin:/usr/bin:/sbin:/bin       LOGNAME=deploy
   NVM_DIR=/home/deploy/.nvm           (+ MAILTO if set)
   NODE_VERSION=v20.11.0
   NVM_BIN=…/bin                       ✗ no NVM_DIR
   DATABASE_URL=postgres://…           ✗ no NODE_VERSION
   AWS_PROFILE=prod                    ✗ no DATABASE_URL
   LANG=en_US.UTF-8                    ✗ no AWS_PROFILE
   USER=deploy                         ✗ no LANG (→ your script's locale/
   …150 more vars…                          sorting/date formats CHANGE)
```

**Why?** Your interactive PATH comes from `~/.bashrc` / `~/.profile` (→ topic 14). **Cron does not source them.** It is not a login shell. It is not an interactive shell. It is `sh -c`, forked from a daemon, with four environment variables.

And nvm is the worst offender: nvm is a **shell function** defined in `~/.bashrc`. There is no `node` binary in any standard PATH at all. Cron has **zero** chance of finding it.

**Three fixes, from worst to best:**

```bash
# FIX A — absolute paths. Works. Brittle: breaks on every nvm version bump.
*/5 * * * * /home/deploy/.nvm/versions/node/v20.11.0/bin/node /srv/app/job.js

# FIX B — set PATH at the top of the crontab. Applies to EVERY job below it.
PATH=/home/deploy/.nvm/versions/node/v20.11.0/bin:/usr/local/bin:/usr/bin:/bin
*/5 * * * * node /srv/app/job.js
#   ✓ Better. Still hardcodes a version.

# ★ FIX C — THE RIGHT ANSWER. ALWAYS CALL A WRAPPER SCRIPT. NEVER A BARE COMMAND. ★
*/5 * * * * /srv/app/bin/run-job.sh >> /var/log/job.log 2>&1
```
```bash
#!/usr/bin/env bash
# /srv/app/bin/run-job.sh   —   chmod +x this!
set -euo pipefail          # -e exit on error, -u error on unset var,
                           # -o pipefail: a failure ANYWHERE in a pipe fails the job
                           # (→ topic 13. Without this, `foo | tee log` "succeeds"
                           #  even when foo dies.)

# 1. Fix the environment — recreate what your shell would have done.
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"     # loads nvm; puts node on PATH
nvm use --silent 20                                  # pin the version explicitly

# 2. Fix the working directory (Failure #2).
cd /srv/app

# 3. Load your app config (Failure #2's cousin).
set -a; [ -f .env ] && . ./.env; set +a   # `set -a` = auto-export everything sourced

# 4. Do the work. Log with timestamps so future-you can debug it.
echo "[$(date -Is)] job starting (node $(node -v))"
node ./scripts/job.js
echo "[$(date -Is)] job finished ok"
```

**Why the wrapper is the right answer:** it is a **file, in git, that you can run by hand**. You can test it. You can `env -i` it. You can add logging, locking, timeouts, and error handling to it. A crontab line is a *string*; a wrapper is a *program*. **Never put logic in a crontab line.**

---

### FAILURE 2 — CWD: cron starts in `$HOME`, not your app directory

```bash
# crontab:
0 * * * * node /srv/app/scripts/report.js
```
```
Error: Cannot find module './config'
    at Module._resolveFilename (node:internal/modules/cjs/loader:1145:15)
```

**Root cause:** cron does `chdir($HOME)` before exec. Your process's CWD is `/home/deploy`, **not** `/srv/app`. Therefore:

- `require('./config')` resolves relative to the **script's** location (Node handles that fine) — but…
- `fs.readFileSync('./data/input.csv')` resolves relative to the **CWD** → `/home/deploy/data/input.csv` → **ENOENT**
- `require('dotenv').config()` looks for `.env` in the **CWD** → `/home/deploy/.env` → **not found, silently.** `dotenv` does not throw. Your `process.env.DATABASE_URL` is simply `undefined`, and your app connects to `localhost` instead of the RDS box, and you spend an hour staring at `ECONNREFUSED`.

**Prove it:**
```bash
# Add this to a crontab for one minute and read the output:
* * * * * pwd > /tmp/cron-cwd.txt 2>&1
# cat /tmp/cron-cwd.txt  →  /home/deploy      ← NOT your app dir
```

**Fix:**
```bash
# In the crontab (acceptable):
0 * * * * cd /srv/app && /usr/bin/node ./scripts/report.js >> /var/log/report.log 2>&1
#         ▲▲▲▲▲▲▲▲▲▲▲▲ ★ && not ; — if the cd FAILS, DO NOT run the command in
#                        the wrong directory. `;` would run it anyway. ★

# ★ In the wrapper (correct):
cd /srv/app || exit 1
# Even better — make the script location-independent:
cd "$(dirname "$0")/.." || exit 1
```

---

### FAILURE 3 — ★ OUTPUT IS EMAILED, AND IF MAIL ISN'T CONFIGURED IT IS SILENTLY LOST ★

**This is the one that costs you real money.**

Cron's design (from 1975, when every Unix box ran a local MTA) is: **any output a job produces on stdout or stderr gets mailed to the crontab's owner.** If the job produces no output, no mail is sent — "silence is success."

On a modern cloud server there is **no MTA installed**. `sendmail` doesn't exist. So cron tries to pipe your job's output into a program that isn't there, and the output goes **nowhere**.

```
   ┌─────────────────────────────────────────────────────────────────────┐
   │  YOUR JOB           cron                    /usr/sbin/sendmail      │
   │                                                                     │
   │  stderr: ────────▶  reads it, opens a  ──▶  ✗ ENOENT.               │
   │  "Error: ECONN-     pipe to sendmail        No such file.           │
   │   REFUSED"                                                          │
   │                                                                     │
   │                     Output is DISCARDED.                            │
   │                     Cron does NOT log it.                           │
   │                     Cron does NOT retry.                            │
   │                     Cron does NOT care.                             │
   │                                                                     │
   │  syslog says:  CRON[9182]: (deploy) CMD (/srv/app/backup.sh)        │
   │                ▲ IT STARTED. That is the ONLY thing recorded.       │
   │                  The exit code is NOT logged. The error is NOT       │
   │                  logged. A job that instantly died looks IDENTICAL   │
   │                  to one that succeeded perfectly.                    │
   └─────────────────────────────────────────────────────────────────────┘

   RESULT: your nightly backup has been failing since the 3rd.
           You find out on the 24th, when you need the backup.
```

**THE FIX — REDIRECT EVERY SINGLE CRON JOB. NO EXCEPTIONS.** (→ topic 12, topic 19)

```bash
# ✓ ALWAYS. THIS. Both streams, appended, to a file you own.
0 3 * * * /srv/app/bin/backup.sh >> /var/log/myapp/backup.log 2>&1
#                                 ▲▲                          ▲▲▲▲
#                                 │                           └── stderr → wherever
#                                 │                               stdout goes (dup2!)
#                                 └── >> APPEND. A single `>` TRUNCATES the log on
#                                     every run — you'd only ever keep the last run.
```

```bash
# The MAILTO directive (put it at the TOP of the crontab):
MAILTO=ops@example.com     # send output here (REQUIRES a working MTA — verify it!)
MAILTO=""                  # ★ explicitly DISABLE mail. Use this WITH redirection,
                           #   so cron doesn't even try, and no mail spool grows.
```

> ⚠ **`> /dev/null 2>&1` is NOT a fix — it is the same bug, written on purpose.** You have now *guaranteed* you will never see the error. Only use it for a job whose failure genuinely doesn't matter, and be honest with yourself about whether that's true. It almost never is.

**Beyond redirection — make failure LOUD:**
```bash
#!/usr/bin/env bash
set -euo pipefail
LOG=/var/log/myapp/backup.log

# Trap ANY error and actively alert. Silence is the enemy.
trap 'echo "[$(date -Is)] ✗ FAILED at line $LINENO (exit $?)" >> "$LOG";
      curl -fsS -m 10 --retry 3 "https://hc-ping.com/$HC_UUID/fail" >/dev/null;
      exit 1' ERR

echo "[$(date -Is)] starting" >> "$LOG"
# ... do the work ...
echo "[$(date -Is)] ✓ ok" >> "$LOG"

# A "dead man's switch": ping a service on SUCCESS. If the ping stops arriving,
# the service alerts YOU. This catches the case cron can never catch:
# ★ THE JOB DIDN'T RUN AT ALL. ★  (Because cron died, or the box was off, or
#   someone typed `crontab -r`.) No amount of logging inside the job detects that.
curl -fsS -m 10 --retry 3 "https://hc-ping.com/$HC_UUID" >/dev/null
```

---

### FAILURE 4 — `%` IS A NEWLINE IN CRONTAB

```bash
# crontab:
0 3 * * * pg_dump mydb > /backups/db-$(date +%Y-%m-%d).sql
```
**This does not do what you think.** In a crontab, an **unescaped `%` means NEWLINE**, and **everything after the FIRST unescaped `%` is fed to the command's STDIN.**

So cron parses that line as:

```
   COMMAND:  pg_dump mydb > /backups/db-$(date +
   STDIN:    Y-m-d).sql
             ▲ this is now INPUT PIPED TO THE COMMAND, not part of it.
```

The `date` subshell has an unterminated argument, the redirect target is garbage, and you get an empty or missing backup file — with a bizarre error, if you're even capturing errors (see Failure 3), which you probably aren't.

**Fix A — escape every `%` with a backslash:**
```bash
0 3 * * * pg_dump mydb > /backups/db-$(date +\%Y-\%m-\%d).sql
#                                            ▲▲  ▲▲  ▲▲
```

**★ Fix B — the right answer: put it in a script.** Inside a shell script, `%` has **no special meaning whatsoever** — it's only the *crontab parser* that treats it specially. This is one more reason **the crontab line should only ever call a wrapper.**
```bash
# crontab — trivially simple, nothing to get wrong:
0 3 * * * /srv/app/bin/backup.sh >> /var/log/backup.log 2>&1
```
```bash
# /srv/app/bin/backup.sh — % is completely ordinary here
DATE=$(date +%Y-%m-%d)          # ✓ works perfectly. No escaping.
pg_dump mydb > "/backups/db-${DATE}.sql"
```

**Useful side effect:** because `%` is a newline, you can feed stdin to a job deliberately:
```bash
0 3 * * * mysql -u root mydb%SELECT COUNT(*) FROM users;%
#                              ▲ everything after the first % becomes stdin
```
Cute. Never do this. Use a script.

---

## The Fifth Failure — NO OVERLAP PROTECTION

Cron `fork()`s and walks away. It **never** checks whether the previous run of the same job is still alive.

```
   Job scheduled: * * * * *   (every minute)
   Job actually takes: 5 minutes

   00:00  ├─ run #1 ──────────────────────────┤ (finishes 00:05)
   00:01  ├─ run #2 ──────────────────────────┤
   00:02  ├─ run #3 ──────────────────────────┤
   00:03  ├─ run #4 ──────────────────────────┤
   00:04  ├─ run #5 ──────────────────────────┤
   00:05  ├─ run #6 ─────────…
   ...
   01:00  ★ 60 CONCURRENT COPIES.
          Each one holds DB connections (→ fd exhaustion, topic 19!)
          Each one competes for the same disk (→ load spike, topic 18)
          Each one makes the others SLOWER, so they take even longer,
          so MORE of them pile up. This is a POSITIVE FEEDBACK LOOP.
          The box goes to load 80 and dies.
```

**Fix: `flock`.** It takes an exclusive lock (a real `flock(2)` syscall on an fd — topic 19) on a lockfile and refuses to start if it's held.

```bash
* * * * * /usr/bin/flock -n /tmp/sync.lock /srv/app/bin/sync.sh >> /var/log/sync.log 2>&1
#         │                │  │            │
#         │                │  │            └── the command to run UNDER the lock
#         │                │  └── the lockfile (created if absent; contents irrelevant)
#         │                └── -n = NON-BLOCKING: if the lock is held, EXIT IMMEDIATELY
#         │                    (exit code 1) rather than queueing up behind it.
#         │                    ★ Without -n, flock WAITS — and you build a QUEUE of
#         │                      waiting processes instead of a pile of running ones.
#         │                      Marginally better. Still a leak. USE -n.
#         └── absolute path — remember Failure #1!

# Variants worth knowing:
flock -n -E 0 /tmp/sync.lock ./sync.sh   # -E 0: exit 0 (not 1) when the lock is
                                          #       held, so "skipped" isn't logged as
                                          #       a failure by your monitoring
flock -w 30   /tmp/sync.lock ./sync.sh   # wait up to 30s for the lock, then give up
```

`flock` is **safe against crashes**: the lock is held on an *open fd*, and the kernel releases it automatically when the process dies — even on `SIGKILL`, even on OOM. A hand-rolled `if [ -f /tmp/lockfile ]` **is not** — a crashed job leaves a stale lockfile behind and your cron job never runs again. **Always use `flock`. Never write your own lockfile logic.**

Also consider a **timeout**, so a hung job can't hold the lock forever:
```bash
* * * * * /usr/bin/flock -n /tmp/sync.lock /usr/bin/timeout 300 /srv/app/bin/sync.sh >> /var/log/sync.log 2>&1
#                                          ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
#                                          SIGTERM after 300s. (Add -k 30 to follow up
#                                          with SIGKILL 30s later if it ignores TERM
#                                          — signals, topic 17.)
```

---

## Timezones — Your "3am" Job Is Not Running at 3am

**Cron uses the SYSTEM timezone.** And virtually every cloud server, Docker image, and CI runner is set to **UTC**.

```bash
timedatectl
#                Local time: Sat 2026-07-12 14:23:07 UTC
#            Universal time: Sat 2026-07-12 14:23:07 UTC
#                  RTC time: Sat 2026-07-12 14:23:07
#                 Time zone: Etc/UTC (UTC, +0000)      ★ THERE IT IS
# System clock synchronized: yes
#               NTP service: active
```

So `0 3 * * *` fires at **03:00 UTC** = 22:00 EDT (yesterday!) = 08:00 in London = 12:30 in Mumbai. If you're in New York and you scheduled a heavy backup for "3am, when nobody's using it", you actually scheduled it for **10pm — right in the middle of your evening traffic peak.**

**And it gets worse: DST.** If the box's timezone *does* observe daylight saving, then twice a year:
- **Spring forward:** 02:00→03:00 never happens. A job at `0 2 * * *` is **skipped entirely.** (Vixie cron tries to compensate for jobs in the 00:00-03:00 window, but don't rely on it.)
- **Fall back:** 02:00 happens **twice**. Your job may run **twice.** If it's a billing job, you just double-charged everyone.

**The rules:**
```bash
# 1. KEEP SERVERS ON UTC. Always. Do not "fix" this by changing the box's timezone —
#    you'd break log correlation, DB timestamps, and every other server you own.

# 2. Compute the UTC time you actually want:
#    "3am New York (EDT, UTC-4)"  →  07:00 UTC
0 7 * * * /srv/app/bin/backup.sh >> /var/log/backup.log 2>&1
#    ⚠ …but in WINTER, New York is EST (UTC-5), so 3am local = 08:00 UTC.
#      There is NO single cron line that means "3am New York, year-round".

# 3. ★ IF YOU NEED LOCAL-TIME SEMANTICS, USE A SYSTEMD TIMER. ★ It has native TZ support:
#      OnCalendar=*-*-* 03:00:00 America/New_York
#    systemd handles DST correctly. Cron cannot. This alone is a reason to switch.

# 4. Or: set CRON_TZ at the top of the crontab (Vixie cron / Debian's cron supports it):
CRON_TZ=America/New_York
0 3 * * * /srv/app/bin/backup.sh
#   ⚠ NOT portable. Not supported by every cron implementation (busybox crond ignores
#     it). Check `man 5 crontab` on YOUR box before trusting it in production.

# 5. And ALWAYS make the job itself timezone-aware if it does date math.
#    Do NOT let `date +%F` in a UTC container decide what "yesterday" means for a
#    report that a human in Chicago will read.
```

---

## Debugging Cron — The Complete Procedure

The job "isn't working." Work down this list, in order.

```bash
# STEP 1 — Is the cron DAEMON even running?
systemctl status cron          # Debian/Ubuntu (`crond` on RHEL/Fedora)
#  ⚠ IN DOCKER there is usually NO cron daemon — containers run ONE process. If your
#    Dockerfile didn't start cron, the crontab is INERT. Use a K8s CronJob / host timer.

# STEP 2 — Did cron ATTEMPT to run it?
grep CRON /var/log/syslog | tail -20     # (or `journalctl -u cron`; RHEL: /var/log/cron)
# CRON[9182]: (deploy) CMD (/srv/app/bin/backup.sh)
#   ★★ PROVES ONLY THAT CRON FORKED IT. No exit code. A 3ms "command not found" crash
#      produces this IDENTICAL line. ★★
#   ABSENT  → cron never matched: check syntax, timezone, `crontab -l`, /etc/cron.allow.
#   PRESENT → cron did its job; the bug is in YOUR script → step 3.
#   Also grep for cron's own confessions:
#     "(CRON) info (No MTA installed, discarding output)"   ← FAILURE #3, admitted
#     "(deploy) BAD FILE MODE" / "ORPHAN (no passwd entry)" / "Error: bad minute"

# STEP 3 — ★★ SIMULATE CRON'S BARE ENVIRONMENT — the single best trick in this doc ★★
env -i /bin/sh -c '/srv/app/bin/backup.sh'      # empty env, under `sh` — exactly like cron.
#   You see "command not found" IMMEDIATELY, in your terminal, instead of at 3am in silence.
env -i HOME=/home/deploy LOGNAME=deploy SHELL=/bin/sh PATH=/usr/bin:/bin \
    /bin/sh -c 'cd $HOME && /srv/app/bin/backup.sh'    # closer still (cron sets these four)

# STEP 4 — DUMP the exact env/cwd/uid cron gives you (add for 1 min, then remove):
#   * * * * * env > /tmp/e.txt 2>&1; pwd >> /tmp/e.txt; id >> /tmp/e.txt
# STEP 5 — Make the wrapper self-documenting: `set -x` + timestamped log lines.
```

---

## `at` and `batch` — One-Shot Jobs

Cron is for *recurring* jobs. `at` runs something **exactly once**, at a specified time. (Package: `at`; the daemon is `atd`.)

```bash
at now + 1 hour              # then type commands, Ctrl-D to finish
at 03:00 tomorrow
at 14:30 2026-08-01
at teatime                   # 16:00. Yes, really.
echo '/srv/app/bin/deploy.sh' | at now + 15 minutes    # non-interactive

atq                          # LIST pending jobs
# 7   Sat Jul 12 15:30:00 2026 a deploy
# ▲ job number
atrm 7                       # REMOVE job 7
at -c 7                      # ★ SHOW job 7's full script — INCLUDING the entire
                             #   environment `at` snapshotted for it

batch                        # run when the system LOAD AVERAGE drops below 1.5
                             # (→ topic 18!) Perfect for a heavy, non-urgent
                             # reindex you don't want competing with live traffic.
```

**★ `at` differs from cron in one crucial way: it SNAPSHOTS YOUR CURRENT ENVIRONMENT** (and CWD) at submit time and restores it at run time. So `at` jobs *do* have your PATH, your nvm, your env vars — which makes them far less surprising than cron. **But its output is still mailed**, so redirect it anyway.

---

## systemd Timers — The Modern Alternative

Everything cron does badly, systemd timers do well. On any box running systemd (which is: Ubuntu, Debian, RHEL, Fedora, Amazon Linux 2+ — i.e. **your production servers**), this is the better tool.

A timer is **two units**: a `.service` (what to run) and a `.timer` (when to run it). They must share a base name.

```ini
# /etc/systemd/system/db-backup.service
[Unit]
Description=Nightly Postgres backup to S3
# Don't even try if the network isn't up (cron has NO concept of this):
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot                       # run to completion and exit (not a daemon)
User=deploy
Group=deploy
WorkingDirectory=/srv/app          # ★ FAILURE #2, SOLVED DECLARATIVELY ★
EnvironmentFile=/etc/myapp/backup.env   # ★ real env vars, from a root-owned 600 file
Environment=PATH=/usr/local/bin:/usr/bin:/bin   # ★ FAILURE #1, SOLVED ★
ExecStart=/srv/app/bin/backup.sh    # absolute path, always

# Resource limits — cron cannot do ANY of this:
MemoryMax=1G                       # cgroup cap: OOM-kill the JOB, not the whole box
CPUQuota=50%                       # never let the backup starve the API
TimeoutStartSec=1800               # kill it after 30 min. No more hung jobs.
Nice=19                            # lowest scheduling priority
IOSchedulingClass=idle             # only use the disk when nothing else needs it
                                   #   (→ topic 18 — this is how you stop a backup
                                   #    from pinning your disk during peak traffic)

# Hardening — also impossible with cron:
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/srv/app/backups
NoNewPrivileges=true

# ★ OUTPUT GOES TO THE JOURNAL. AUTOMATICALLY. FAILURE #3 CANNOT HAPPEN. ★
# (StandardOutput=journal is the default. There is no mail. There is no black hole.)
```

```ini
# /etc/systemd/system/db-backup.timer      ← SAME BASE NAME as the .service
[Unit]
Description=Run the nightly DB backup

[Timer]
OnCalendar=*-*-* 03:00:00          # year-month-day HH:MM:SS. Also: daily, hourly, weekly,
                                   #   Mon..Fri 09:00, *-*-01 04:00:00, *:0/15 (every 15 min),
                                   #   Mon *-*-* 02:00:00 America/New_York  ★ NATIVE TIMEZONE ★
                                   # VERIFY: systemd-analyze calendar "Mon..Fri 09:00" --iterations=5
Persistent=true                    # ★★ KILLER FEATURE: box OFF when it should have fired?
                                   #    Run it IMMEDIATELY on next boot. CRON CANNOT DO THIS —
                                   #    a missed run is simply gone, silently. (anacron bolts this on.)
RandomizedDelaySec=1800            # ★ random 0-30min: 200 instances won't hit S3 at 03:00:00.000
AccuracySec=1m
# OnUnitActiveSec=1h               # fire 1h AFTER the last run FINISHED → cannot overlap itself,
#                                  #   for free (systemd also won't start a unit already active;
#                                  #   cron happily stacks 60 copies). OnBootSec=15min also exists.

[Install]
WantedBy=timers.target             # ← NOT multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now db-backup.timer      # ★ enable the .TIMER, not the .service

# ── Everything cron can't tell you ──────────────────────────────────────────
systemctl list-timers --all       # NEXT run + LAST run of every timer — cron answers NEITHER
# Sun 2026-07-13 03:00:00 UTC 12h left Sat 2026-07-12 03:14:22 UTC 11h ago db-backup.timer db-backup.service
systemctl status db-backup.service
#   Process: 9182 ExecStart=/srv/app/bin/backup.sh (code=exited, status=0/SUCCESS)  ← ★ THE EXIT CODE
journalctl -u db-backup.service -n 50 --no-pager           # ★ THE ACTUAL OUTPUT (add -p err for errors)
sudo systemctl start db-backup.service                     # ★ RUN IT NOW to test (cron: edit + wait)
```

### Why timers beat cron

| | **cron** | **systemd timer** |
|---|---|---|
| **Output/logging** | Mailed → **silently lost** | **Journal.** `journalctl -u foo` |
| **Exit code** | Never recorded | `systemctl status` shows it; `OnFailure=` can page you |
| **Missed run (box was off)** | **Gone forever, silently** | ★ `Persistent=true` runs it on next boot |
| **Timezone** | System TZ only (usually UTC); DST is broken | `OnCalendar=… America/New_York`, DST-correct |
| **Overlap** | 60 copies pile up | Won't start a unit that's already active; `OnUnitActiveSec=` |
| **Thundering herd** | Manual `sleep $RANDOM` | `RandomizedDelaySec=` |
| **Dependency ordering** | None. Runs before the network is up | `After=network-online.target`, `Requires=postgresql.service` |
| **Resource limits** | None | `MemoryMax=`, `CPUQuota=`, `IOSchedulingClass=`, `TimeoutStartSec=` |
| **Environment** | 4 vars, `sh -c` | `EnvironmentFile=`, `WorkingDirectory=`, `User=` — declarative |
| **Test a run now** | Edit the schedule and wait | `systemctl start foo.service` |
| **Validate the schedule** | Guess, and hope | `systemd-analyze calendar "Mon..Fri 09:00"` |
| **Wiped by one typo** | ★ `crontab -r` | Files on disk, in git |
| **Simplicity** | ✓ **One line. Universal. Works everywhere.** | ✗ Two files, more ceremony, needs systemd |

**The honest recommendation:** on a real production server, **use systemd timers** for anything that matters — backups, billing, cleanup, cert renewal. Use cron for trivial personal chores where a silent failure genuinely costs nothing. And if you inherit a cron-based box, at minimum: **redirect all output**, **wrap in a script**, **flock it**, and **add a dead man's switch**.

---

## Example 1 — Basic

```bash
crontab -l                                   # "no crontab for deploy" (probably)
crontab -l > ~/crontab.bak 2>/dev/null       # ★ BACK IT UP BEFORE EVERY EDIT ★
crontab -e                                   # then put this in the editor:
#   PATH=/usr/local/bin:/usr/bin:/bin        # cron's env is empty — set PATH once for all jobs
#   MAILTO=""                                # we redirect ourselves; don't try to mail
#   * * * * * echo "[$(date -Is)] pwd=$(pwd)" >> /tmp/cron-heartbeat.log 2>&1

sleep 70; cat /tmp/cron-heartbeat.log
#   [2026-07-12T14:24:01+00:00] pwd=/home/deploy   ← ★ CWD is $HOME, NOT your app dir ★
grep CRON /var/log/syslog | tail -1          # CRON[9182]: (deploy) CMD (...)  ← "started", not "succeeded"

env | wc -l ; env -i /bin/sh -c 'env | wc -l'    # ~62 in your shell vs ~1 under cron
env -i /bin/sh -c 'node -v'                       # sh: 1: node: not found  ← FAILURE #1, reproduced
crontab -e                                        # clean up by deleting the line (NOT `crontab -r`!)
```

---

## Example 2 — Production Scenario

**Monday, 09:15.** The CTO: *"Can you restore the users table from Friday's backup?"*

```bash
$ ls -lh /backups/
total 4.0K
-rw-r--r-- 1 deploy deploy 2.1G Jun 18 03:14 db-2026-06-18.sql.gz
```

**June 18th.** It is now **July 12th.** The nightly backup has not produced a file in **24 days**. Nobody knew. Nothing alerted. The cron job "ran" every single night.

```bash
$ crontab -l
0 3 * * * cd /srv/app && node scripts/backup.js > /backups/db-$(date +%Y-%m-%d).sql.gz

$ grep CRON /var/log/syslog | grep backup | tail -2
Jul 11 03:00:01 db-01 CRON[3021]: (deploy) CMD (cd /srv/app && node scripts/backup.js > /backups/db-$(date +)
Jul 12 03:00:01 db-01 CRON[4102]: (deploy) CMD (cd /srv/app && node scripts/backup.js > /backups/db-$(date +)
#                                                                                                    ▲▲▲▲▲▲▲▲
#                     ★ LOOK AT WHERE THE LOGGED COMMAND IS TRUNCATED. ★
#                       It stops dead at `$(date +`. Cron ATE the rest of the line
#                       at the first unescaped `%`. ★ FAILURE #4. ★
```

**Three bugs in one line.** Count them:

```bash
0 3 * * * cd /srv/app && node scripts/backup.js > /backups/db-$(date +%Y-%m-%d).sql.gz
#                        ▲▲▲▲                   ▲                          ▲
#                        │                      │                          │
#  FAILURE #1 ───────────┘                      │                          │
#  `node` is nvm-installed. Cron's PATH is      │                          │
#  /usr/bin:/bin. → `node: command not found`.  │                          │
#                                               │                          │
#  FAILURE #3 ──────────────────────────────────┘                          │
#  No `2>&1`. The "command not found" error went to STDERR → mailed →      │
#  no MTA installed → DISCARDED. Silence. For 24 days.                     │
#                                                                          │
#  FAILURE #4 ─────────────────────────────────────────────────────────────┘
#  Unescaped `%`. Cron truncated the command at `$(date +` and fed
#  "Y-m-d).sql.gz" to the job's STDIN. The redirect target is garbage.
#
#  BONUS: even when it "worked" back in June, it wrote a .sql.gz that was
#  never actually gzipped. Nobody checked. That backup is probably broken too.
```

**Verify each hypothesis before fixing anything.** Reproduce cron's world:

```bash
$ env -i /bin/sh -c 'cd /srv/app && node scripts/backup.js'
/bin/sh: 1: node: not found                      # ★ FAILURE #1 CONFIRMED, in 2 seconds.

$ ls /var/mail/ /var/spool/mail/ 2>&1
ls: cannot access '/var/mail/': No such file or directory     # ★ FAILURE #3 CONFIRMED.
$ which sendmail
                                                 # (nothing) — no MTA. Output = void.

$ grep -i 'No MTA' /var/log/syslog | tail -1
Jul 12 03:00:01 db-01 CRON[4102]: (CRON) info (No MTA installed, discarding output)
#                                          ★ CRON LITERALLY TOLD US. Every night.
#                                            In a log nobody reads. ★
```

**THE FIX.** Not a patched crontab line — a **wrapper script**, in **git**, driven by a **systemd timer** (units shown in the systemd section above; the wrapper's skeleton):

```bash
#!/usr/bin/env bash
# /srv/app/bin/backup.sh   (chmod 750)
set -euo pipefail                          # ★ fail loudly, immediately, on pipe errors too
LOG=/var/log/myapp/backup.log
exec >> "$LOG" 2>&1                         # everything below → the log (dup2 on 1&2, topic 19)
log() { echo "[$(date -Is)] $*"; }
trap 'log "✗ FAILED line $LINENO (exit $?)"; curl -fsS -m10 "https://hc-ping.com/$HC_UUID/fail"||true; exit 1' ERR

export NVM_DIR="$HOME/.nvm"; [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"   # FIX #1: env
cd /srv/app                                                                     # FIX #2: cwd
DATE=$(date -u +%Y-%m-%d)                   # FIX #4: `%` is ordinary inside a script
DEST="/backups/db-${DATE}.sql.gz"

# `set -o pipefail` makes a pg_dump failure FAIL THE JOB instead of gzip'ing an empty stream
# (that's how you get 24 nightly "backups" that are all 20 bytes long).
pg_dump --no-owner --format=custom "$DATABASE_URL" | gzip -9 > "$DEST"

SIZE=$(stat -c%s "$DEST")                    # ★ VERIFY THE ARTIFACT — an unverified backup
[ "$SIZE" -gt 10485760 ] || { log "✗ only ${SIZE}B"; exit 1; }   #   is not a backup
gzip -t "$DEST"                              # prove the stream is intact
aws s3 cp "$DEST" "s3://mycompany-backups/db/${DATE}.sql.gz"
find /backups -maxdepth 1 -name 'db-*.sql.gz' -type f -mtime +7 -delete   # prune (topic 08)
#             └ anchor the pattern & -maxdepth: a bare `-mtime +7 -delete` in the wrong dir
#               is how people delete their whole server
curl -fsS -m10 --retry 3 "https://hc-ping.com/$HC_UUID"   # ★ dead man's switch: the ONLY
log "✓ backup complete"                                   #   thing that catches "never ran"
```

The `.service` + `.timer` are exactly the pair shown in the systemd section (`Type=oneshot`, `Persistent=true`, `RandomizedDelaySec=900`, `IOSchedulingClass=idle`, `OnFailure=`). Install and — critically — **test it now**, don't wait for 3am:

```bash
$ sudo systemctl start db-backup.service
$ journalctl -u db-backup.service -n 5 --no-pager
Jul 12 09:41:02 db-01 backup.sh[9910]: [2026-07-12T09:41:02+00:00] starting → db-2026-07-12.sql.gz (v20.11.0)
Jul 12 09:44:51 db-01 systemd[1]: db-backup.service: Succeeded.   # ★ THE EXIT STATUS cron never gives you
$ systemctl list-timers db-backup.timer
NEXT                        LEFT     LAST                        UNIT             ACTIVATES
Sun 2026-07-13 03:07:33 UTC 17h left Sat 2026-07-12 09:41:02 UTC db-backup.timer  db-backup.service
#                        ▲ 03:07:33 = RandomizedDelaySec at work
```

**The point:** the original bug wasn't really the `%` or the PATH. **The bug was that a failure produced no signal.** A dead man's switch would have paged you on **June 19th**, not July 12th.

---

## Common Mistakes

The four classic failures above are the top four mistakes — here they are in Wrong→Right form, plus two more. (Root causes are detailed in the failures section; this is the quick-reference version.)

### Mistake 1: `node: command not found` — but it works when I type it
**Wrong:** `*/5 * * * * node /srv/app/job.js` · **Right:** `*/5 * * * * /srv/app/bin/run-job.sh >> /var/log/job.log 2>&1`
**Root cause:** cron execs `/bin/sh -c` with only `PATH=/usr/bin:/bin`, `SHELL`, `HOME`, `LOGNAME` — it never sources `~/.bashrc`/`~/.profile` (→ topic 14), and nvm is a shell *function* defined there, not a binary. **Diagnose:** `env -i /bin/sh -c 'node -v'`. **Prevent:** never put a bare command in a crontab; always call a wrapper that sets up its own environment.

### Mistake 2: Not redirecting output — the silent 3-week failure
**Wrong:** `0 3 * * * /srv/app/bin/backup.sh` · **Right:** append `>> /var/log/backup.log 2>&1`
**Root cause:** cron pipes stdout+stderr to `sendmail`; with no MTA the output is discarded, and syslog logs only that the job *started* — a 3ms crash and a perfect success produce the identical line. **Diagnose:** `grep -i 'No MTA' /var/log/syslog`. **Prevent:** redirect every job, plus a dead man's switch — only the switch catches "it never ran at all."

### Mistake 3: `date +%Y-%m-%d` inside a crontab line
**Wrong:** `... > /b/db-$(date +%Y-%m-%d).sql` · **Right:** escape as `\%Y-\%m-\%d`, or put it in a script.
**Root cause:** the crontab *parser* treats unescaped `%` as a newline; everything after the first `%` becomes the job's stdin, silently truncating your command. **Diagnose:** `grep CRON /var/log/syslog` — the logged command is cut off at the `%`. **Prevent:** never use `%` in a crontab line; inside a script it's ordinary.

### Mistake 4: A slow job every minute — cron stacks them
**Wrong:** `* * * * * /srv/app/bin/sync.sh` (takes 5 min) · **Right:** wrap in `flock -n /tmp/sync.lock` + `timeout 300`.
**Root cause:** cron keeps no state about running jobs; it forks a fresh copy every minute → a positive feedback loop → load 80 → dead box. **Diagnose:** `pgrep -fac sync.sh` → 47. **Prevent:** `flock -n` on every job that could run long (the kernel releases it even on SIGKILL — a hand-rolled lockfile does not). Or `OnUnitActiveSec=` in a timer.

### Mistake 5: `crontab -r` instead of `crontab -e`
**Wrong:** `crontab -r` (one key from `-e`) wipes the whole crontab — no prompt, no undo. **Right:** `crontab -l > ~/crontab.bak` before every edit; `alias crontab='crontab -i'`; keep jobs in `/etc/cron.d/` or a `.timer`, in git.
**Root cause:** `-r` `unlink()`s the spool file; per topic 19 the blocks are then actually freed. **Recover:** `~/crontab.bak`, backups, or reconstruct commands (not schedules) from `grep CRON /var/log/syslog`.

### Mistake 6: `0 3 1 * 1` — the DOM/DOW OR trap
**Wrong:** `0 3 1 * 1` intending "1st, if it's a Monday." **Right:** it fires on the 1st *and* every Monday (~5×/month).
**Root cause:** POSIX — when *both* DOM and DOW are restricted, the match is an OR (every other field pair ANDs). **Fix:** leave DOW as `*`, restrict DOM, test the weekday in the script: `0 3 1-7 * * [ "$(date +\%u)" = "1" ] && .../monthly.sh` — or a timer with `OnCalendar=Mon *-*-01..07 03:00:00`, provable via `systemd-analyze calendar`.

---

## Hands-On Proof

```bash
# PROVE IT: cron's env is nearly empty (add for 1 min, then read)
#   * * * * * env > /tmp/cron-env.txt 2>&1; pwd >> /tmp/cron-env.txt
cat /tmp/cron-env.txt      # SHELL,HOME,LOGNAME,PATH=/usr/bin:/bin,PWD=/home/deploy — that's it
env | wc -l                # 60+ in your shell; cron gives you ~5

# PROVE IT: `env -i` reproduces cron EXACTLY — the best debugging trick there is
env -i /bin/sh -c 'node -v'          # sh: 1: node: not found  ← the 3am failure, NOW, in your terminal

# PROVE IT: cron logs only that a job STARTED, never whether it worked
#   * * * * * /bin/false     ← then:  grep CRON /var/log/syslog | tail
# CRON[9182]: (deploy) CMD (/bin/false)   ← identical to a success. No exit code anywhere.

# PROVE IT: % is a newline that truncates your command
#   * * * * * echo "before%after" > /tmp/pct.txt      → file contains only "before"

# PROVE IT: flock prevents overlap
flock -n /tmp/demo.lock -c 'echo GOT; sleep 60'   &   # terminal 1 holds it
flock -n /tmp/demo.lock -c 'echo I RAN'; echo $?     # terminal 2 → prints nothing, exit 1

# PROVE IT: run-parts ignores filenames with a dot
sudo cp /bin/true /etc/cron.daily/mytest.sh && sudo chmod +x /etc/cron.daily/mytest.sh
run-parts --test /etc/cron.daily     # mytest.sh is ABSENT from the list — the dot killed it
sudo mv /etc/cron.daily/mytest.sh /etc/cron.daily/mytest
run-parts --test /etc/cron.daily | grep mytest   # NOW it appears
sudo rm /etc/cron.daily/mytest

# PROVE IT: spool file is real & mode 600 / server is UTC / systemd knows what cron can't
sudo ls -l /var/spool/cron/crontabs/      # -rw------- deploy crontab
timedatectl; date -u; date                # your "3am" is UTC, not local
systemd-analyze calendar "Mon..Fri 09:00" --iterations=3   # cron has no equivalent
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
crontab -l > ~/crontab.bak          # ★ ALWAYS FIRST ★
crontab -e
```
Add: `* * * * * date >> /tmp/tick.log 2>&1` and `* * * * * env > /tmp/e.txt; pwd >> /tmp/e.txt; id >> /tmp/e.txt`

**Questions:** After 3 minutes, how many lines are in `/tmp/tick.log`? How many environment variables did cron give you, versus `env | wc -l` in your shell? What is cron's `$PATH`, and what is its CWD? What user did the job run as? Now find the `CRON[...]` lines in `/var/log/syslog` — what information is *missing* from them that you'd want during an incident?

---

### Exercise 2 — Medium

**Reproduce all four classic failures deliberately, then fix each one.**

1. **PATH:** Install node via nvm. Write `job.js` that appends a timestamp to a file. Schedule `* * * * * node /path/job.js`. Confirm it fails. Prove *why* with `env -i /bin/sh -c 'node -v'`. Fix it three ways (absolute path; `PATH=` in the crontab; wrapper script) and confirm each works.
2. **CWD:** Make `job.js` do `fs.readFileSync('./data.txt')` with `data.txt` next to it. Watch it throw `ENOENT`. Prove the CWD is `$HOME` with a `pwd` cron line. Fix with `cd`.
3. **Silent output:** Schedule `* * * * * /bin/false` and `* * * * * this-command-does-not-exist`. Show that syslog reports them **identically to a successful job**. Then find cron's own confession: `grep -i 'No MTA' /var/log/syslog`. Fix with `>> log 2>&1`.
4. **The `%`:** Schedule `* * * * * echo "a%b" > /tmp/p.txt`. Show that only `a` lands in the file. Show the truncation in syslog. Fix with `\%`, then fix it *properly* with a script.
5. **Overlap:** Schedule `* * * * * /bin/sleep 300` for 5 minutes. Run `pgrep -c sleep` — how many? Add `flock -n` and repeat. How many now?

---

### Exercise 3 — Hard (Production Simulation)

Build a **production-grade nightly backup**, twice: once with cron, once with a systemd timer. Then break it, and prove your monitoring catches it.

1. **The wrapper** (`/srv/app/bin/backup.sh`): `set -euo pipefail`; source nvm; `cd` to the app; load `.env`; dump a DB (or `tar` a directory) to `/backups/db-$(date +%F).tar.gz`; **verify the artifact** (size > threshold, `gzip -t` passes); prune with `find /backups -maxdepth 1 -name 'db-*' -type f -mtime +7 -delete`; timestamp every log line; `trap ... ERR` to log and alert on failure; ping a dead man's switch on success.
2. **Version A — cron:** one line, calling only the wrapper, with `flock -n`, `timeout`, and `>> log 2>&1`. Confirm with `env -i` that it works in a bare environment.
3. **Version B — systemd timer:** `.service` + `.timer` with `Type=oneshot`, `User=`, `WorkingDirectory=`, `EnvironmentFile=`, `OnCalendar=*-*-* 03:00:00`, `Persistent=true`, `RandomizedDelaySec=900`, `IOSchedulingClass=idle`, `OnFailure=`. Verify with `systemd-analyze calendar` and `systemctl list-timers`. Run it now with `systemctl start` and read the output with `journalctl -u`.
4. **Prove `Persistent=true` works:** stop the timer, set the system clock forward past the scheduled time (or just wait a cycle with the timer masked), re-enable it, and show the missed job runs immediately. **Then prove cron cannot do this.**
5. **Break it on purpose:** `chmod -x` the script. Show exactly what each system does:
   - cron: does syslog show the failure? Does anything alert? (No. That's the lesson.)
   - systemd: `systemctl status` shows the non-zero exit; `journalctl` shows the error; `OnFailure=` fires.
6. Write a 6-line runbook entry: how to tell, at 9am, whether last night's backup ran — for each of the two systems. Which one can actually answer the question?

---

## Mental Model Checkpoint

Answer from memory.

1. **What exactly does the cron daemon do every 60 seconds?** Name the two syscalls it uses to run your job.
2. **Name the five fields, in order, with their ranges.** Which value is Sunday — and why are there two answers?
3. **`0 3 1 * 1` — when does this actually fire?** Why is that (almost certainly) not what the author intended?
4. **List the four classic cron failures.** For each: the symptom, the root cause, and the one-line fix.
5. **What is the single best command for debugging "it works in my terminal but not in cron"?** What does it do?
6. **A `CRON[9182]: (deploy) CMD (...)` line is in syslog. What does that prove — and, crucially, what does it NOT prove?**
7. **You have a job scheduled every minute that takes 5 minutes. What happens, and what is the exact fix?** Why is a hand-rolled lockfile check worse than `flock`?
8. **Name three things a systemd timer can do that cron fundamentally cannot.** Which of them would have saved the 24-day silent backup failure?

---

## Quick Reference Card

| Command / Syntax | What It Does | Key Notes |
|---|---|---|
| `crontab -e` | Edit your crontab | **Always use this.** It validates syntax + notifies cron |
| `crontab -l` | List your crontab | `crontab -l > ~/cron.bak` — **do this before every edit** |
| `crontab -r` | ☠ **DELETE the whole crontab** | **No confirmation. No undo.** One key from `-e`. Alias `crontab='crontab -i'` |
| `crontab -u deploy -l` | Another user's crontab | Root only |
| `* * * * * cmd` | min hour dom month dow | dow: `0` **and** `7` are Sunday |
| `*/5 * * * *` | Every 5 minutes | Steps don't wrap — `*/7` fires at :56 then :00 |
| `@reboot` / `@daily` / `@hourly` | Shortcuts | `@reboot` is **not** a process supervisor. Use systemd |
| `/etc/cron.d/myapp` | System crontab | ★ **Extra 6th field: the USER.** No dots in the filename |
| `/etc/cron.daily/` | Drop an executable in | ★ **No dots in filename**, must be `+x`. Test: `run-parts --test` |
| `/var/spool/cron/crontabs/` | Where user crontabs live | Mode 600. Never edit by hand |
| `MAILTO=""` | Disable cron's mail | Use **with** redirection, not instead of it |
| `>> /var/log/x.log 2>&1` | ★ **Capture stdout AND stderr** | **Without this, errors are silently discarded** |
| `\%` | Escape `%` in a crontab | Unescaped `%` = newline; truncates your command |
| `flock -n /tmp/x.lock cmd` | ★ **Overlap protection** | `-n` = skip if locked. Kernel releases it even on SIGKILL |
| `timeout 300 cmd` | Kill a hung job | `-k 30` to SIGKILL after SIGTERM |
| `env -i /bin/sh -c 'cmd'` | ★ **Simulate cron's bare env** | **The single best cron debugging trick** |
| `grep CRON /var/log/syslog` | Did cron *start* the job? | Proves it **started**. Says nothing about success |
| `journalctl -u cron` | Same, on systemd boxes | — |
| `at now + 1 hour` | One-shot job | `atq` list, `atrm N` remove, `at -c N` inspect. Snapshots your env |
| `batch` | Run when load < 1.5 | → topic 18 |
| `timedatectl` | System timezone | Servers are almost always **UTC** |
| **systemd timer** | `.timer` + `.service` | `OnCalendar=`, ★ `Persistent=true`, `RandomizedDelaySec=`, `OnUnitActiveSec=` |
| `systemctl list-timers --all` | ★ **Next & last run of every timer** | Cron can tell you neither |
| `systemd-analyze calendar "…"` | ★ **Validate a schedule before shipping** | Prints the next N firing times |
| `journalctl -u db-backup` | ★ **The job's actual output** | The whole reason to use timers |

---

## When Would I Use This at Work?

### Scenario 1: The nightly backup that hasn't run in a month
Someone asks for Friday's DB snapshot. The newest file in `/backups/` is 24 days old. The crontab line looked fine. It wasn't: `node` wasn't on cron's PATH, and with no `2>&1` the "command not found" was mailed into a void on a box with no MTA. `grep -i 'No MTA' /var/log/syslog` shows cron confessing every single night, in a log nobody reads. **Fix:** wrapper script that sources nvm, `>> log 2>&1`, a size+integrity check on the artifact, and a **dead man's switch** — because only the dead man's switch would have caught "it never ran at all", which is the failure that actually cost you the data.

### Scenario 2: Load average 60, and the culprit is your own log cleaner
`uptime` shows load 60 on a 4-core box (→ topic 18). `vmstat 1` shows `b`=48 — everything blocked on disk. `pgrep -fac cleanup.sh` → **51 copies running.** The job is scheduled every minute and started taking 8 minutes once the log directory grew past 400k files. Cron never checks; it just kept forking. Each copy made the disk slower, so each copy took longer, so more piled up. **Fix:** `pkill -f cleanup.sh`, then `flock -n /tmp/cleanup.lock` and `timeout 600` on the crontab line. **Prevent:** `flock` on every job that could ever run long — which is all of them.

### Scenario 3: The certbot renewal that fired at 3am — and took the site down
TLS renewal ran at "3am" per the crontab. The server is UTC; the team is in San Francisco. 03:00 UTC is **8pm Pacific** — peak traffic. The renewal reloaded nginx, dropping connections during the busiest hour of the day. Worse, 40 instances all fired at 03:00:00 and hit Let's Encrypt simultaneously, triggering rate limits. **Fix:** a systemd timer with `OnCalendar=*-*-* 03:00:00 America/Los_Angeles` (native TZ, DST-correct — **cron cannot do this at all**) plus `RandomizedDelaySec=3600` to spread the herd, plus `Persistent=true` so an instance that was down for maintenance still renews the moment it comes back.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 08 — Navigating the Filesystem | `find /backups -mtime +7 -delete` is how every cron log-pruner works |
| **Builds on** | 12 — Pipes and Redirection | `>> log 2>&1` is **the** fix for cron's silent-failure problem |
| **Builds on** | 13 — Shell Scripting | Every cron job should be a script with `set -euo pipefail` and a `trap ... ERR` |
| **Builds on** | 14 — Environment Variables | Cron doesn't source `.bashrc`/`.profile` — that is failure #1, entirely |
| **Builds on** | 16 — Processes in Depth | Cron is a daemon that does `fork()` + `execve()`. That's the whole design |
| **Builds on** | 17 — Process Management | `pkill -f`, `timeout`, and SIGTERM vs SIGKILL when a cron job hangs |
| **Builds on** | 18 — System Resource Monitoring | Piled-up cron jobs are a classic 3am load spike; `batch` waits for low load |
| **Builds on** | 19 — File Descriptors | `flock` is a lock on an **fd**; `>> file 2>&1` is `dup2()` on fds 1 and 2 |
| **Next** | 21 — Package Management | `apt` ships `/etc/cron.daily/` scripts — now you know what those are |
| **Used by** | 22 — systemd and Services | Timers ARE systemd units. `.timer` + `.service`, `journalctl`, `OnFailure=` |
| **Used by** | 23 — Logs and Log Management | `logrotate` is itself a cron/timer job; and cron logs go to syslog/journald |
| **Used by** | 33 — Node.js in Production | Scheduled jobs, graceful shutdown, and env-var loading in a non-interactive shell |
| **Used by** | 35 — Security Basics | `unattended-upgrades` and `certbot renew` are both scheduled jobs |
