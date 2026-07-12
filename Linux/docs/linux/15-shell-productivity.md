# 15 — Shell Shortcuts & Productivity

## ELI5 — The Simple Analogy

Watch a chef at a busy line versus someone cooking at home.

The home cook walks to the drawer, opens it, finds the knife, walks back, cuts one onion, walks to the bin, comes back. The line chef never looks at their hands — knife, board, pan, and every ingredient are in muscle memory. Same knife, same onion. The chef is 20x faster because **the motions are automatic and the tools are exactly where the hands expect them.**

This doc is about becoming the line chef *of the terminal*. At 2am with production down, the developer who fumbles — retyping long paths, re-running the whole command to fix one typo, losing the working command they ran ten minutes ago — loses time they don't have. The developer whose hands already know Ctrl-R, `Alt-.`, and `tmux` moves at the speed of thought. None of it is talent. It's a small set of reflexes you install once.

---

## Where This Lives in the Linux Stack

```
Hardware (keyboard)
  └── KERNEL — the TTY/PTY driver. Runs in "canonical mode" so the LINE editor works.
       │       Handles a few keys itself: Ctrl-C→SIGINT, Ctrl-Z→SIGTSTP, Ctrl-S/Q→flow.
       └── System Calls — read()/write() on fd 0/1; ioctl() to configure the terminal
            └── C Library
                 └── READLINE (GNU library)  ◀◀◀ THIS TOPIC. Linked into bash, psql,
                      │   the python & node REPLs, gdb, mysql...  ONE library, so the
                      │   SAME keys (Ctrl-A, Ctrl-R, Alt-.) work in ALL of them.
                      └── SHELL (bash/zsh) — history, aliases, completion, brace expansion
                           └── tmux / screen — a terminal MULTIPLEXER wrapping all of it
```

The magic fact of this whole topic: **Readline is a library, not a bash feature.** Any program that links it gets identical line-editing. That's why the keys you learn here pay off in `psql`, the Node REPL, `gdb`, and dozens more — you learn them once and they work everywhere.

---

## What Is This?

Productivity in the shell comes from four systems: **Readline** (in-line editing — moving and changing the command you're typing), **history** (recalling and reusing past commands), **shell conveniences** (aliases, functions, tab-completion, brace expansion, fast navigation), and **terminal multiplexing** (tmux/screen — keeping work alive across disconnects). Each is a small investment with an outsized, permanent return.

---

## Why Does This Matter for a Backend Developer?

| If you don't know this... | This is the cost... |
|---|---|
| Ctrl-R reverse search | You retype a 90-character `docker run` from memory, get it subtly wrong, and debug the wrong thing |
| `Alt-.` (last argument) | You re-type the long path you just used, introduce a typo, and `cd` into nowhere |
| History persistence | You close a terminal and lose the exact migration command you'll need to reproduce the incident |
| `HISTCONTROL=ignorespace` | You leak `DB_PASSWORD=...` into `~/.bash_history` in plaintext (topic 14 callback) |
| tmux | Your SSH drops mid-`npm install`, SIGHUP kills it, and you return to a half-installed, corrupt `node_modules` |
| Functions vs aliases | You write `alias mkcd='mkdir $1 && cd $1'`, it silently ignores the argument, and you can't figure out why |
| Ctrl-S froze your terminal | You think the server hung, panic, and kill a healthy process |

---

## The Physical Reality — What Readline Actually Does

When you type at a bash prompt, you are **not** talking to bash character by character. The kernel's terminal driver is in **canonical (cooked) mode**, and bash has handed control to **Readline**, which keeps an in-memory edit buffer:

```
   You press keys           THE READLINE EDIT BUFFER (in bash's memory)
   ─────────────            ┌──────────────────────────────────────────┐
   d o c k e r   r u ...    │ docker run -it ubuntu bash                │
                            │              ▲                            │
   Ctrl-A  ───────────────▶ │ point (cursor) moves to column 0         │
   Alt-.   ───────────────▶ │ inserts last arg of the previous command │
   Ctrl-W  ───────────────▶ │ deletes a word, saves it to the KILL RING│
   Ctrl-Y  ───────────────▶ │ pastes (yanks) from the kill ring        │
   Enter   ───────────────▶ │ NOW the whole line is handed to the shell│
   └──────────────────────────────────────────────────────────────────┘
        Nothing runs until Enter. Everything before it is pure editing.

   THE KILL RING (Readline's clipboard — separate from the OS clipboard):
   ┌─────────────┬──────────────┬─────────────┐
   │ "ubuntu"    │ "server.js"  │ "/var/log"  │  ← Ctrl-W / Ctrl-U / Ctrl-K push here
   └─────────────┴──────────────┴─────────────┘     Ctrl-Y yanks the most recent
```

**Emacs mode** is the default keymap (there's also vi mode, below). "Kill" means cut-to-kill-ring; "yank" means paste-from-kill-ring — old Emacs vocabulary that Readline inherited.

```bash
# PROVE IT: readline is a shared library, linked into many programs
ldd /bin/bash | grep readline
# libreadline.so.8 => /lib/x86_64-linux-gnu/libreadline.so.8   ← that's the library
# Now open `psql`, or `python3`, or `node`, and try Ctrl-A / Ctrl-E / Ctrl-R.
# Same keys. Same library. (Node/psql use it or a work-alike; the bindings match.)
```

---

## How It Works — Readline Emacs Keybindings (Learn These Cold)

### Moving the cursor (nothing is deleted)

```
   $ node --max-old-space-size=2048 server.js --port 3000
     ▲          ▲    ▲                    ▲
   Ctrl-A     Alt-B Ctrl-B              Ctrl-E
   (start)   (word (char             (end of line)
             back)  back)
```

| Key | Moves | Mnemonic |
|---|---|---|
| **Ctrl-A** | Start of line | **A** = first letter |
| **Ctrl-E** | End of line | **E** = End |
| **Ctrl-B** / **Ctrl-F** | Back / Forward one **char** | **B**ack, **F**orward |
| **Alt-B** / **Alt-F** | Back / Forward one **word** | Alt = bigger jumps |

### Editing (these delete — and killed text goes to the kill ring)

| Key | Deletes | Goes to kill ring? |
|---|---|---|
| **Ctrl-W** | Word **before** cursor (whitespace-delimited) | ✅ yank it back with Ctrl-Y |
| **Alt-D** | Word **after** cursor | ✅ |
| **Ctrl-U** | In bash: from cursor **to start of line** | ✅ (great: type password wrong → Ctrl-U → retype) |
| **Ctrl-K** | From cursor **to end of line** | ✅ |
| **Ctrl-Y** | **Yanks** (pastes) the most recently killed text | — |
| **Ctrl-T** | **Transpose** the two chars around the cursor (fix `sl`→`ls`) | — |
| **Ctrl-_** | **Undo** the last edit (repeatable) | — |

> **Note:** Ctrl-U in bash's default (emacs) mode kills to the *start of line*, not the whole line. In the kernel's cooked-mode line discipline (before readline, e.g. at a raw password prompt) Ctrl-U kills the whole line. Same key, two different owners.

### The single highest-value shortcut: `Alt-.`

**`Alt-.` (or `Esc` then `.`) inserts the LAST ARGUMENT of the previous command.** Learn this one until it's a reflex.

```bash
$ mkdir -p /srv/app/releases/2026-07-12/config
$ cd <Alt-.>          # expands to:  cd /srv/app/releases/2026-07-12/config
#                        You did NOT retype the path. Press Alt-. AGAIN to walk back
#                        through the last arg of EARLIER commands.
```

Why it beats `!$` (covered below): `Alt-.` **expands the text right there on your command line, before you press Enter, so you SEE it.** `!$` expands only as the command runs — you're trusting, not looking.

### The second highest-value: Ctrl-R reverse-i-search

**Ctrl-R** searches your history *backwards* as you type a fragment.

```
$ <Ctrl-R>
(reverse-i-search)`mig': npm run migrate:latest -- --env production
        │           │     └── the best match so far, ready to run on Enter
        │           └── what you've typed
        └── you are now searching backwards through history
```

- Type a few chars of any command you ran before. The best match appears.
- **Ctrl-R again** → jump to the *next older* match with the same fragment.
- **Enter** → run it. **→ (right arrow) or Ctrl-E/Ctrl-A** → put it on the line to *edit* first (safer).
- **Ctrl-G** → abort, leave the line empty. **Ctrl-S** → search *forward* again (only if flow-control is disabled — see below).

### The control keys the KERNEL owns (not readline)

These are handled by the terminal driver, so they work even when readline isn't running (e.g. inside `node`, `top`, a hung program):

| Key | Signal / effect | What it's for |
|---|---|---|
| **Ctrl-C** | **SIGINT** — interrupt | Stop the running foreground program (topic 17) |
| **Ctrl-D** | **EOF** — *not a signal!* Closes stdin | Ends input. At an empty bash prompt it logs you out; in `node`/`python` it exits the REPL; it ends a `cat > file` heredoc (topic 12/19 callback) |
| **Ctrl-Z** | **SIGTSTP** — suspend | Pause the foreground job; resume with `fg` or background it with `bg` (topic 17) |
| **Ctrl-S** / **Ctrl-Q** | **XOFF / XON** flow control | **THE "my terminal froze" mystery** — see below |
| **Ctrl-L** | Clear screen (redraw) | A readline redraw — **not** the same as `clear` (see mistakes) |

**The frozen-terminal mystery, solved:** you hit **Ctrl-S** by accident (reaching for save). The terminal driver reads it as **XOFF** and *pauses all output* — the process is fine and still running, the tty just stopped painting. Everything looks hung. **Press Ctrl-Q (XON) to unfreeze it.** Knowing this saves you from killing a healthy process in a panic. (You can disable this entirely with `stty -ixon`, which also frees Ctrl-S for forward-search.)

### vi mode & ~/.inputrc

Prefer modal editing? `set -o vi` (put it in `~/.bashrc`) switches Readline to vi keybindings: you start in *insert* mode, press **Esc** for *command* mode, then use `h j k l`, `w`, `b`, `dd`, `cw`, etc., and `/pattern` to search history. `set -o emacs` switches back.

Readline reads **`~/.inputrc`** at startup for customisation:

```bash
# ~/.inputrc
set editing-mode emacs                 # or vi
set completion-ignore-case on          # tab-complete regardless of case
set show-all-if-ambiguous on           # single Tab lists matches (no double-tap)
"\e[A": history-search-backward        # Up arrow = search history by current prefix
"\e[B": history-search-forward         # Down arrow = same, forward
set bell-style none                    # stop the terminal beeping at you
```

That Up/Down binding is a life-changer: type `git ` then press Up to cycle only through *past commands starting with `git `*.

---

## How History Actually Works

```
   ┌──────────────────────────┐            ┌──────────────────────────┐
   │  IN-MEMORY history list  │            │   ~/.bash_history (disk)  │
   │  (this shell session)    │            │   loaded at startup,      │
   │  1002  cd /srv/app       │  written   │   written on exit         │
   │  1003  npm run migrate   │ ─────────▶ │   (or with `history -a`)  │
   │  1004  pm2 restart api   │  on EXIT   │                           │
   └──────────────────────────┘            └──────────────────────────┘
```

Each interactive shell holds history **in memory** and, by default, only **writes it to `~/.bash_history` on clean exit** (`history -w`). Two consequences bite immediately:

- **Kill a terminal** (crash, `kill -9`, closed laptop) → that session's history is **never written** → gone.
- **Multiple terminals open** → each writes on exit and, by default, **overwrites** the file → whichever closes *last* wins, and the others' history vanishes.

The fix for both — put this in `~/.bashrc`:

```bash
shopt -s histappend                       # APPEND on exit instead of overwriting
export PROMPT_COMMAND='history -a'        # write each command to the file IMMEDIATELY,
#                                           after every prompt — so nothing is ever lost
export HISTSIZE=100000                     # commands kept in MEMORY this session
export HISTFILESIZE=200000                 # commands kept in the FILE on disk
export HISTCONTROL=ignoreboth              # = ignoredups + ignorespace (see below)
export HISTTIMEFORMAT='%F %T  '            # timestamp every entry: 2026-07-12 02:14:07
export HISTIGNORE='ls:ll:cd:pwd:clear:history'   # don't record trivial noise
```

| Setting | What it does |
|---|---|
| `HISTSIZE` | How many commands are kept **in memory** for this session |
| `HISTFILESIZE` | How many lines are kept **in the file** on disk |
| `HISTCONTROL=ignoredups` | Don't store a command identical to the one before it |
| `HISTCONTROL=ignorespace` | **Don't store any command that starts with a SPACE** |
| `HISTCONTROL=ignoreboth` | Both of the above |
| `HISTIGNORE` | Colon-list of patterns never to record |
| `HISTTIMEFORMAT` | Adds timestamps — **invaluable for incident forensics** |

**The `ignorespace` secret trick (topic 14 callback):** with `HISTCONTROL` including `ignorespace`, prefix any sensitive command with a **leading space** and it never enters history:

```bash
$  export STRIPE_KEY=sk_live_abc123      # ← note the leading space. Never recorded.
$ history 1                              # ...it's not there. Your secret stayed private.
```

**`HISTTIMEFORMAT` for forensics:** after an incident, `history` with timestamps answers "what exactly did I run at 02:14, and in what order?" — the difference between a real post-mortem and guesswork.

### History expansion (the `!` bang commands)

| Expansion | Means | Example |
|---|---|---|
| `!!` | The entire previous command | `sudo !!` — re-run the last command with sudo (the canonical use) |
| `!$` | Last **argument** of the previous command | `mkdir /a/b/c` → `cd !$` |
| `!*` | **All** arguments of the previous command | `touch a b c` → `chmod +x !*` |
| `!n` | History entry number *n* | `!1003` |
| `!string` | Most recent command **starting with** `string` | `!npm` re-runs your last npm command |
| `!?string?` | Most recent command **containing** `string` | `!?migrate?` |
| `^old^new^` | Re-run last command, replacing first `old` with `new` | typo fix: `^prod^staging^` |
| `!!:gs/old/new/` | Re-run last command, **g**lobally **s**ubstituting | `!!:gs/dev/prod/` |

**⚠ The critical warning:** history expansion happens **before you can review it** — `!!` and `!$` expand *as the command executes*, so you're trusting your memory of what "the last command" was. **`Alt-.` and Ctrl-R are safer habits** because they put the real text on your command line where you *see it* before pressing Enter. Also: a `!` inside **double quotes** triggers expansion and will bite you (`echo "hello!"` may error or mangle) — use single quotes, or `set +H` to disable expansion entirely.

---

## Aliases, Functions, and Completion

### Aliases — and why they can't take arguments

```
alias gs='git status'
│     │  │
│     │  └── the replacement text (quote it). Simple textual substitution at line START.
│     └── the alias name you'll type
└── a shell builtin. `unalias gs` removes it. `alias` alone lists them all.
```

Aliases are **pure text substitution at the start of a command** — so they **cannot take positional arguments** (`$1`, `$2`). When you need arguments, use a **function**:

```bash
# ❌ WRONG — the $1 is literal text; the alias just PREPENDS "mkdir -p $1 && cd $1"
alias mkcd='mkdir -p $1 && cd $1'      # `mkcd foo` → runs `mkdir -p $1 && cd $1 foo`

# ✅ RIGHT — a function takes real arguments
mkcd() { mkdir -p "$1" && cd "$1"; }   # `mkcd foo` → makes foo, enters it
```

**Where aliases go:** `~/.bashrc`, **not** `~/.bash_profile` — because aliases are **not exported to child shells** (topic 14: only environment variables are inherited), so they must be defined in the file that runs for *every* interactive shell. To **bypass** an alias for one run, prefix a backslash or use `command`:

```bash
alias rm='rm -i'          # (see the danger below)
\rm file                  # bypass the alias — run the real rm
command rm file           # same effect
```

**The dangerous-alias antipattern:** `alias rm='rm -i'` *builds a habit* of typing `rm *` expecting a confirmation prompt. The day you're on a fresh server, a container, or a coworker's box **without** that alias, `rm *` deletes silently and instantly. Safety aliases that don't travel with you are worse than none. Prefer real habits (`ls` before `rm`, use `trash-cli`, use `-I` which prompts once for bulk deletes).

### Genuinely useful aliases & functions for a backend dev

```bash
# ~/.bashrc
alias ll='ls -alhF'                                  # long, human sizes, type markers
alias gs='git status'
alias gl='git log --oneline --graph --decorate -20'
alias ports='ss -tulpn'                              # what's LISTENING (topic 28)
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'

dexec() { docker exec -it "$1" "${2:-bash}"; }       # dexec api   → shell into container
logs()  { journalctl -u "$1" -f --no-pager; }        # logs my-api → tail a service log
tailapp() { tail -f /var/log/"$1"/*.log; }
mkcd()  { mkdir -p "$1" && cd "$1"; }
```

### Tab completion

Press **Tab** to complete a command, path, variable, or hostname; press **Tab twice** to list all matches when it's ambiguous.

```bash
$ cd /var/lo<Tab>            # → /var/log/
$ echo $HO<Tab>             # → $HOME  (variable completion)
$ cat file<Tab><Tab>        # lists file1.txt file2.txt ... (double-tap = show options)
```

**Programmable completion** goes far beyond filenames. The `bash-completion` package teaches bash the *semantics* of specific tools:

```bash
sudo apt install bash-completion        # Debian/Ubuntu; sourced from ~/.bashrc usually
git checkout <Tab>                      # completes BRANCH NAMES, not files
git log --<Tab>                         # completes git's flags
docker <Tab>                            # completes subcommands, then image/container names
kubectl get pods <Tab>                  # completes live pod names from the cluster
# Tools often ship their own:
source <(kubectl completion bash)
source <(npm completion)
```

---

## Brace Expansion — Small Idioms, Big Wins

Brace expansion is done by the shell **before** the command runs, generating multiple strings from one:

```
cp nginx.conf{,.bak}   →   cp nginx.conf nginx.conf.bak
   │            ││
   │            │└── second variant: append ".bak"
   │            └── first variant: nothing (empty)
   └── the base string, reused for every variant
```

The `{,.bak}` idiom is *the* beloved one-liner for making a backup — memorise it. More:

```bash
cp app.conf{,.bak}                 # instant backup — no retyping the filename
mv report.txt{,.old}               # rename by appending
mkdir -p src/{api,web,shared}/dist # make src/api/dist, src/web/dist, src/shared/dist
touch log_{1..10}.txt              # log_1.txt ... log_10.txt  (numeric range)
echo {a..e}                        # a b c d e                 (alphabetic range)
diff <(sort a.txt) <(sort b.txt)   # (process substitution — a cousin, topic 12)
```

---

## Fast Navigation & Fixing Monster Commands

```bash
cd -                 # jump BACK to the previous directory ($OLDPWD) — a toggle
pushd /var/log       # cd there AND push the old dir onto a stack
popd                 # pop the stack — return to where pushd started
dirs -v              # show the directory stack, numbered
```

```bash
# Put these in ~/.bashrc for effortless navigation:
shopt -s autocd      # type a directory NAME alone (no `cd`) to enter it: `/var/log` ⏎
shopt -s cdspell     # auto-correct minor typos in cd:  cd /var/lgo  → /var/log
shopt -s dirspell    # spell-correct directory names during completion
export CDPATH=".:$HOME/projects:/srv"   # `cd api` searches these — cd from anywhere
```

**Ctrl-X Ctrl-E — edit the command line in `$EDITOR`.** When you're staring at a 200-character `awk | sed | while read` monster and cursor-shuffling isn't cutting it, press **Ctrl-X Ctrl-E**: bash opens the current line in `$EDITOR` (topic 14 — set `EDITOR=vim`), you edit it with full editor power, save and quit, and bash runs the result. Indispensable for one-liners that grew too big.

### Job control quick recap (full detail in topic 17)

```bash
long_command          # running in the foreground
<Ctrl-Z>              # SIGTSTP — suspend it; you get your prompt back
jobs                  # list suspended/background jobs:  [1]+  Stopped  long_command
bg                    # resume job 1 in the BACKGROUND
fg                    # bring it back to the FOREGROUND
long_command &        # start it in the background from the outset
```

---

## Terminal Multiplexing — tmux (Non-Negotiable in Production)

**The problem it solves:** you SSH into a server and start `npm install` (or a DB migration, or a long `rsync`). Your laptop sleeps, the WiFi hiccups, the VPN drops. The SSH connection dies, the kernel sends **SIGHUP** ("hang up") to your shell and everything it started, and your process is **killed mid-write** — a half-installed `node_modules`, a partially-applied migration, a corrupt deploy. This is a genuine career-scarring event.

**tmux** (terminal multiplexer) runs a **persistent session on the server** that your SSH client merely *attaches* to. When your connection drops, the session keeps running detached; you reconnect and *reattach* exactly where you left off.

```
   YOUR LAPTOP                         SERVER
   ───────────                         ──────
   ssh ────────────────────────▶  ┌──────────────────────────┐
   tmux attach ─────────────────▶ │  tmux SESSION "deploy"    │
                                   │   └─ npm install (running)│
   ✗ WiFi drops — SSH dies         │   KEEPS RUNNING detached  │  ← survives SIGHUP
                                   │                           │
   ssh again ──────────────────▶  │                           │
   tmux attach -t deploy ───────▶ │  ...still installing. 78%. │  ← you're back
                                   └──────────────────────────┘
```

### The tmux commands you actually need

```bash
tmux new -s deploy         # start a NEW named session "deploy"
#   ...run your long command...
#   Ctrl-B  then  D        → DETACH (leave it running, drop back to your normal shell)
tmux ls                    # list sessions:  deploy: 1 windows (attached)
tmux attach -t deploy      # RE-ATTACH to "deploy" after you reconnect
tmux kill-session -t deploy   # when you're truly done with it
```

**The prefix key is `Ctrl-B`.** You press `Ctrl-B`, release, then a command key:

| After `Ctrl-B` | Does |
|---|---|
| `D` | **Detach** (the one you'll use constantly) |
| `C` | Create a new window (tab) |
| `N` / `P` | Next / previous window |
| `"` | Split pane horizontally |
| `%` | Split pane vertically |
| `arrow` | Move between panes |
| `[` | Enter scroll/copy mode (scroll back through output; `q` to exit) |

`screen` is the older equivalent (`screen -S deploy`, `Ctrl-A D` to detach, `screen -r deploy` to reattach) — same idea, use whichever is installed.

**Lighter alternatives (topic 17 link)** when you just need *one* command to survive a disconnect and don't need to reattach:

```bash
nohup long_command &          # ignore SIGHUP; output → nohup.out
long_command & disown         # start it, then detach it from the shell's job table
setsid long_command           # run in a new session, fully divorced from the terminal
```

Use `nohup`/`disown` to fire-and-forget; use **tmux when you want to come back and watch**.

---

## Common Mistakes

### Mistake 1 — Ctrl-L is not `clear`

**Wrong:** believing `Ctrl-L` and `clear` are identical.
**Right:** `clear` erases the scrollback too (on most terminals it emits the "clear + reset scrollback" sequence); **Ctrl-L is a Readline redraw** — it moves your current prompt to the top of the screen but your history is still there when you scroll up. Handy: Ctrl-L works *while you're mid-typing* a command (it repaints the current line at top), whereas running `clear` would need you to abandon the line.

### Mistake 2 — `alias mkcd='... $1 ...'` silently ignores the argument

**Wrong:** `alias mkcd='mkdir -p $1 && cd $1'`. Running `mkcd foo` expands to `mkdir -p $1 && cd $1 foo` — the `foo` is just appended to the end, `$1` is empty, and you get a confusing error.
**Root cause:** aliases are **textual substitution**, not callable code — they have no parameter list, so `$1` is meaningless (it's the alias's own — empty — positional param). **Right:** use a function: `mkcd() { mkdir -p "$1" && cd "$1"; }`. Rule of thumb: **need an argument in the middle → function; simple prefix → alias.**

### Mistake 3 — Losing history with multiple terminals

**Wrong:** default settings, three terminals open. You close them one by one; each overwrites `~/.bash_history` on exit; only the last survives; the commands from the first two are gone — including the one you needed for the post-mortem.
**Root cause:** by default bash **writes (overwrites)** the whole history file on exit, once per shell, with no merging.
**Right:** `shopt -s histappend` + `PROMPT_COMMAND='history -a'` (append each command immediately). Now every shell contributes and nothing is lost. **Prevention:** put it in `~/.bashrc` on day one of any new machine.

### Mistake 4 — Trusting `!!`/`!$` blind, or a `!` in double quotes

**Wrong:** `rm -rf !$` when you *think* the last argument was `/tmp/build` but it was actually `/` from a different command two lines up. History expansion runs with no preview.
Also **wrong:** `echo "It works!"` in an interactive shell → `bash: !": event not found`, because `!"` triggered history expansion inside the double quotes.
**Root cause:** `!` expansion is resolved by the shell *before execution* and *before* you can eyeball the result; double quotes do **not** protect it (only single quotes do).
**Right:** build the reflex of **`Alt-.` and Ctrl-R** instead — they place the real text on the line for you to see. For literal bangs, use single quotes (`echo 'It works!'`) or `set +H` to switch history expansion off.

### Mistake 5 — Panicking at a "frozen" terminal (Ctrl-S)

**Wrong:** output stops after you fat-finger Ctrl-S; you assume the process hung or the server died; you open another session and `kill -9` a perfectly healthy process, corrupting its work.
**Root cause:** Ctrl-S is **XOFF** — the terminal driver paused *display*, not the program. The program is running fine, its output is just buffered.
**Right:** press **Ctrl-Q** (XON) and everything resumes. **Prevention:** `stty -ixon` in `~/.bashrc` disables software flow control entirely (and frees Ctrl-S to become forward-search).

### Mistake 6 — Running long jobs over bare SSH

**Wrong:** `ssh prod` then `npm ci && npm run migrate` directly. The link drops → SIGHUP → both die halfway.
**Root cause:** when the pty closes, the kernel sends SIGHUP to the foreground process group; nothing was protecting it.
**Right:** `tmux new -s deploy` first, run inside it, `Ctrl-B D` to detach whenever you like. **Prevention:** make "any command longer than ~10 seconds on a remote host goes in tmux" an unbreakable habit.

---

## Hands-On Proof

```bash
# PROVE IT: readline is a shared library shared by many programs
ldd /bin/bash | grep readline        # libreadline.so.8 => ...
# Open psql / python3 / node and confirm Ctrl-A, Ctrl-E, Ctrl-R behave identically.

# PROVE IT: Alt-. really inserts the last argument (no retyping)
mkdir -p /tmp/proof/a/b/c
cd /tmp/proof/a/b/c            # now type:  ls <Alt-.>   → becomes  ls /tmp/proof/a/b/c

# PROVE IT: the kill ring works like a clipboard
#   type:  echo hello world foo
#   Ctrl-W (kills "foo") Ctrl-W (kills "world") — then Ctrl-Y pastes "world" back.

# PROVE IT: ignorespace keeps a command out of history
export HISTCONTROL=ignoreboth
 echo "this line starts with a space"     # ← leading space
history 3 | tail -3                        # the spaced command is ABSENT

# PROVE IT: timestamps land in history for forensics
export HISTTIMEFORMAT='%F %T  '
whoami
history 2                                  #  1050  2026-07-12 14:07:31  whoami

# PROVE IT: brace expansion happens in the shell, before the command
echo cp app.conf{,.bak}                    # cp app.conf app.conf.bak   (echo shows it)
echo mkdir -p src/{api,web}/dist           # mkdir -p src/api/dist src/web/dist

# PROVE IT: Ctrl-Z suspends, bg/fg resume (topic 17)
sleep 120                                  # then press Ctrl-Z
jobs                                       # [1]+  Stopped   sleep 120
bg ; jobs                                  # [1]+  Running   sleep 120 &
kill %1                                    # clean up

# PROVE IT: tmux survives a disconnect
tmux new -s test                           # inside: run `top`, then Ctrl-B then D
tmux ls                                    # test: 1 windows
tmux attach -t test                        # you're back in `top`
#   (in the pane) press q, then: exit      # closes the session

# PROVE IT: history expansion has NO preview (why Alt-./Ctrl-R are safer)
echo one two three
echo !$                                    # runs `echo three` — you didn't see it first
```

---

## Practice Exercises

### Exercise 1 — Easy (Readline reflexes)

```bash
# Type this line but DO NOT press Enter:
#   docker run -it --rm -v /srv/app:/app -w /app node:18 npm test
# Now, using ONLY keyboard shortcuts (no arrow-holding, no backspace-spamming):
# 1. Jump to the start of the line (one key).
# 2. Jump to the end (one key).
# 3. Delete the word "test" using Alt-Backspace or Ctrl-W, then Ctrl-Y to paste it back.
# 4. Clear the entire line to the start with Ctrl-U.
# 5. Retrieve it from the kill ring with Ctrl-Y.
```
**Question:** which keys moved the cursor by *word* vs by *character*?

### Exercise 2 — Medium (History that never lets you down)

```bash
# 1. Add to ~/.bashrc: histappend, PROMPT_COMMAND='history -a', HISTSIZE/HISTFILESIZE,
#    HISTCONTROL=ignoreboth, HISTTIMEFORMAT. Open a NEW shell.
# 2. Open a SECOND terminal. Run a distinctive command in each.
# 3. In terminal ONE, run `history | tail`. Confirm you can see terminal TWO's command
#    (thanks to `history -a`). Before your change, you could not.
# 4. Run a fake "secret":   export FAKE_KEY=abc   with a LEADING SPACE. Prove with
#    `history` that it was NOT recorded.
# 5. Use Ctrl-R to find and re-run one of your earlier commands WITHOUT retyping it.
```
**Deliverable:** the `~/.bashrc` block, and the `history | tail` proving cross-terminal capture.

### Exercise 3 — Hard (Production Simulation)

```bash
# You must run a 20-minute migration on a remote server over a flaky connection, and
# be able to walk away and reconnect.
# 1. SSH to a host (or use localhost). Start a tmux session named "migration".
# 2. Inside it, simulate the job:  for i in $(seq 1 1200); do echo "step $i"; sleep 1; done
# 3. DETACH (Ctrl-B D). Confirm with `tmux ls` that it's still running.
# 4. Fully disconnect: close the terminal / drop the SSH session entirely.
# 5. Reconnect and `tmux attach -t migration`. Prove the loop kept counting while you
#    were gone (the step number jumped).
# 6. Write ONE navigation function + ONE alias you'd genuinely add to your dotfiles, and
#    a `\`-bypass demonstration that runs the real `ls` past an `alias ls='ls --color'`.
```
**Deliverable:** `tmux ls` output before and after reconnect, and your function + alias.

---

## Mental Model Checkpoint

1. **Why do Ctrl-A, Ctrl-E, and Ctrl-R work identically in bash, `psql`, and the Node REPL?**
2. **What does `Alt-.` do, and why is it safer than `!$`?**
3. **Your terminal output froze after you pressed a key near "S". What happened, and which key un-freezes it?**
4. **Ctrl-D is not a signal — so what is it, and why does it exit the Node REPL and log you out of bash?**
5. **Why can't an alias take arguments, and what do you use instead? Write `mkcd` correctly.**
6. **You had three terminals open and lost most of your history. What are the two `~/.bashrc` settings that fix this?**
7. **Your SSH drops during `npm install` and the install dies. What tool prevents this, and what are the three commands (new / detach / reattach)?**
8. **What does `cp nginx.conf{,.bak}` expand to, and who performs that expansion?**

---

## Quick Reference Card

| Category | Key / Command | What it does |
|---|---|---|
| **Move** | `Ctrl-A` / `Ctrl-E` | Start / end of line |
| | `Alt-B` / `Alt-F` | Back / forward one **word** |
| **Edit** | `Ctrl-W` / `Alt-D` | Kill word before / after cursor |
| | `Ctrl-U` / `Ctrl-K` | Kill to start / end of line |
| | `Ctrl-Y` | Yank (paste) from kill ring |
| | `Ctrl-_` | Undo |
| | `Alt-.` | **Insert last argument of previous command** |
| **History** | `Ctrl-R` | **Reverse-i-search** (Ctrl-R again = older, Ctrl-G = abort) |
| | `!!` / `!$` / `!*` | Last command / last arg / all args |
| | `!string` / `^old^new^` | Last cmd starting with `string` / quick typo fix |
| **Control** | `Ctrl-C` / `Ctrl-Z` | SIGINT (interrupt) / SIGTSTP (suspend) |
| | `Ctrl-D` / `Ctrl-S` / `Ctrl-Q` | EOF / freeze (XOFF) / unfreeze (XON) |
| | `Ctrl-L` / `Ctrl-X Ctrl-E` | Redraw screen / edit line in `$EDITOR` |
| **Shell** | `alias` / `unalias` | Define / remove alias (`\cmd` bypasses) |
| | `cd -` / `pushd` / `popd` | Toggle dir / push / pop dir stack |
| | `{a,b}` / `{1..9}` / `f{,.bak}` | Brace expansion |
| | `Tab` / `Tab Tab` | Complete / list matches |
| **tmux** | `tmux new -s NAME` | New named session |
| | `Ctrl-B D` / `tmux attach -t NAME` | Detach / reattach |
| | `tmux ls` | List sessions |
| **Detach lite** | `nohup CMD &` / `CMD & disown` | Survive SIGHUP without tmux |

---

## Make-Your-Shell-Fast Checklist

Drop this into `~/.bashrc` on every machine you touch, then never fumble again:

```bash
# ── History: never lose a command ──
shopt -s histappend
export PROMPT_COMMAND='history -a'
export HISTSIZE=100000 HISTFILESIZE=200000
export HISTCONTROL=ignoreboth               # ignoredups + ignorespace (secret-hiding)
export HISTTIMEFORMAT='%F %T  '             # timestamps for incident forensics
export HISTIGNORE='ls:ll:cd:pwd:clear:history'

# ── Navigation: move without thinking ──
shopt -s autocd cdspell dirspell
export CDPATH=".:$HOME/projects:/srv"

# ── Editing: sane readline ──
export EDITOR=vim                           # powers Ctrl-X Ctrl-E, git, crontab -e
stty -ixon                                  # free Ctrl-S; kill the freeze trap

# ── ~/.inputrc: prefix history search on arrows ──
#   "\e[A": history-search-backward
#   "\e[B": history-search-forward
#   set completion-ignore-case on

# ── Aliases & functions (NOT in .bash_profile — aliases aren't inherited) ──
alias ll='ls -alhF'  gs='git status'  ports='ss -tulpn'
mkcd(){ mkdir -p "$1" && cd "$1"; }
dexec(){ docker exec -it "$1" "${2:-bash}"; }

# ── Habit, not config: tmux for anything long on a remote host ──
```

The four reflexes that matter most, in order: **Ctrl-R**, **`Alt-.`**, **tmux detach/reattach**, and **a leading space to hide a secret**. Install those four and you already move like the line chef.

---

## When Would I Use This at Work?

### Scenario 1: 2am incident, reconstructing what you ran
The API is throwing 500s after a deploy. You need the *exact* migration and restart commands you ran an hour ago to reproduce and roll back. With `HISTTIMEFORMAT` and cross-terminal `history -a`, you run `history | grep migrate` and read the precise commands **with timestamps** — no guessing. Without it, you're reconstructing from memory under pressure, and memory is exactly what fails at 2am.

### Scenario 2: A long deploy over a shaky VPN
You're running a 15-minute `docker build` + push on a bastion host from a coffee-shop connection. You start it inside `tmux new -s build`, `Ctrl-B D`, and close your laptop to catch the train. On the train you reconnect, `tmux attach -t build`, and watch it finish. The disconnect that would have killed a bare-SSH build was a non-event.

### Scenario 3: Iterating on a gnarly log-parsing one-liner
You're building `journalctl -u api | grep ERROR | awk '...' | sort | uniq -c | sort -rn` and it's grown unwieldy. Instead of cursor-crawling, **Ctrl-X Ctrl-E** opens it in vim, you restructure the pipeline with real editing, save, and it runs. Then **`Alt-.`** feeds the last file path into the next command, and **Ctrl-R** recalls the working version after you experiment. Three reflexes turn a frustrating grind into a fast loop.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 14 — Environment Variables | `HISTCONTROL`, `HISTSIZE`, `PROMPT_COMMAND`, `EDITOR`, `CDPATH` are all environment/shell variables; `ignorespace` is the secret-hiding trick from topic 14 |
| **Builds on** | 07 — How the Shell Works | Aliases, functions, and expansion all happen in the shell's parse step before fork/exec |
| **Builds on** | 12 — Pipes and Redirection | Ctrl-D sends EOF on stdin; process substitution (`<(...)`) extends brace/expansion idioms |
| **Uses** | 17 — Process Management | Ctrl-C/Ctrl-Z/Ctrl-D map to SIGINT/SIGTSTP/EOF; `jobs`/`bg`/`fg`/`nohup`/`disown` and SIGHUP are the reason tmux exists |
| **Uses** | 19 — File Descriptors | Ctrl-D closes stdin (fd 0), which is why it exits REPLs and ends `cat >` |
| **Used by** | 24 — SSH in Depth | tmux lives on the far side of an SSH session; SIGHUP-on-disconnect is an SSH behaviour |
| **Used by** | 23 — Logs and Log Management | `journalctl -f`, `tail -f`, and log-tailing aliases/functions are daily productivity wins |
| **Used by** | 33 — Node.js in Production | Fast, reliable shell work under pressure is the whole job when a production Node app is down |
