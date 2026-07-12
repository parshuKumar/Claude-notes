# 13 — Shell Scripting Fundamentals

## ELI5 — The Simple Analogy

A shell script is a **recipe card** you hand to a very fast, very literal line cook.

- The cook does **exactly** what the card says, in order, top to bottom. No common sense. If the card says "add salt to the pot" and there is no pot, the cook pours salt on the floor and keeps going — unless you wrote "STOP if anything goes wrong" at the top (`set -e`).
- The **first line** of the card says which cook to give it to: "This is a recipe for the *French* cook" (`#!/usr/bin/env bash`). Hand a French recipe to the *short-order* cook (`dash`) and half the instructions mean nothing — they'll silently skip the fancy steps and plate something wrong.
- **Ingredients** are variables. If you write "add cheese" but never bought cheese, a careless cook adds nothing and moves on (empty string). A *strict* cook stops and says "you never gave me cheese" (`set -u`) — which is exactly what stops them from "delete everything in `` `` " and wiping the whole kitchen.
- The card can have **sub-recipes** (functions), **"if the sauce is too thin, do X"** (conditionals), **"repeat for every onion"** (loops), and a **"no matter what, wash the pot when you're done"** note stuck to the bottom (`trap ... EXIT`).

A good recipe card is precise, refuses to continue when confused, quotes every ingredient name so "goat cheese" isn't read as two separate items, and always cleans up.

---

## Where This Lives in the Linux Stack

```
Hardware
  └── KERNEL
       │   • execve() reads the first 2 bytes of a file. Sees "#!" → runs the named interpreter.
       │                                                    ◀◀◀ THIS is why a shebang works
       └── System Calls: fork() execve() wait() exit()
            │
            └── C Standard Library
                 │
                 └── SHELL (bash) — reads your script line by line and IS the interpreter.
                      │            Variables, if/for/while, functions, [[ ]], $(( )) —
                      │            these are all BASH LANGUAGE FEATURES, not programs.  ◀◀◀ THIS TOPIC
                      └── External commands (grep, curl, systemctl) it forks & execs as needed
```

**The key distinction:** some things in a script are **bash built-ins / keywords** (`if`, `for`, `[[`, `cd`, `local`, `echo`) — bash runs them itself, no new process. Others are **external programs** (`grep`, `curl`, `[` believe it or not) — bash forks and execs them (topic 07). Knowing which is which explains half of all scripting bugs.

---

## What Is This?

A shell script is a plain text file containing shell commands, run top-to-bottom by an interpreter (bash). It is the glue of Unix: it wires together the small tools from topics 08–12 into automated, repeatable procedures — deploys, backups, health checks, cron jobs.

Bash is a real programming language — variables, functions, arrays, conditionals, loops — but a *quirky* one, full of whitespace-sensitive syntax and silent failure modes. This doc teaches you to write scripts that fail **loudly and safely** instead of silently corrupting production.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `set -euo pipefail` | A failed `cd` is ignored, the next line runs `rm -rf` in the **wrong directory**, and it's gone |
| `set -u` on unset vars | `rm -rf "$APPDIR/"` with an empty `$APPDIR` becomes `rm -rf /` — the whole disk |
| Quoting variables | A path with a space (`/my app/`) splits into two arguments and your command mangles files |
| `[` vs `[[` | `[ $x > 5 ]` silently **creates a file named 5** instead of comparing numbers |
| `for f in $(ls)` is broken | Filenames with spaces split; your loop processes half-filenames and corrupts data |
| `cmd \| while read` subshell | Your counter is 0 after the loop; you spend an hour thinking bash "forgot" your variable |
| Exit codes / 137 | Your Docker container dies with **exit 137** and you don't recognize it as the OOM killer |
| `trap ... EXIT` | Your temp files pile up in `/tmp` until the disk fills and takes the server down |

---

## The Physical Reality

### What `#!/usr/bin/env bash` actually does

The shebang is not a comment and not a bash feature — it is a **kernel** feature. When you `execve()` a file, the kernel reads its first bytes:

```
  You run:  ./deploy.sh   (first line of the file is  #!/usr/bin/env bash)
           │
           ▼
  KERNEL execve("./deploy.sh"):
    1. Reads first 2 bytes → sees 0x23 0x21 ("#!") → this is an INTERPRETER file
    2. Reads the rest of line 1 → "/usr/bin/env bash"
    3. REWRITES the exec into:  execve("/usr/bin/env", ["/usr/bin/env","bash","./deploy.sh"])
    4. env searches PATH, finds bash, and runs:  bash ./deploy.sh
    5. bash opens ./deploy.sh and interprets it line by line
```
(Proof that the kernel — not bash — routes on `#!` is in Hands-On below, using `#!/bin/cat`.)

### `#!/usr/bin/env bash` vs `#!/bin/bash`

| | `#!/bin/bash` | `#!/usr/bin/env bash` |
|---|---|---|
| Finds bash | Only at the hardcoded path `/bin/bash` | Searches `$PATH` — finds bash wherever it is |
| macOS (bash 3.2 at /bin, bash 5 via brew) | Gets the **ancient** system bash 3.2 | Gets your modern brew bash 5.x first on PATH |
| Alpine / minimal containers | `/bin/bash` may not exist at all | Still works if bash is anywhere on PATH |
| **Verdict** | Fragile across systems | **Portable — prefer this** |

### `#!/bin/sh` is a TRAP — it is NOT bash

On Debian and Ubuntu, `/bin/sh` is a symlink to **`dash`**, a minimal POSIX shell that is *faster* but does **not** understand "bashisms": `[[ ]]`, arrays, `${var^^}`, `source`, `function`, `<<<`, `set -o pipefail` (older versions), `local` semantics. A script with `#!/bin/sh` that uses these **silently breaks** or errors cryptically.

```bash
# PROVE IT: /bin/sh is dash on Debian/Ubuntu, and bashisms fail under it
ls -l /bin/sh              # /bin/sh -> dash
bash -c '[[ 1 == 1 ]] && echo bash-ok'     # bash-ok
sh   -c '[[ 1 == 1 ]] && echo sh-ok'       # sh: 1: [[: not found   ← the bashism failed!
```

**Rule:** if you use bash features, the shebang MUST be bash, and you must run it as `./script.sh` or `bash script.sh` — **never `sh script.sh`**, which ignores the shebang entirely and forces dash.

---

## The Unofficial Bash Strict Mode — learn this FIRST

Put this at the top of **every** script you write. It converts bash from "silently limp along after errors" to "stop the moment anything is wrong."

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

### `set -e` — exit immediately if any command fails (non-zero exit)

Without it, a script barrels past errors. With it, the first failure aborts the script.

```bash
cd /srv/app/releases/new    # if this fails (dir missing)...
rm -rf ./*                  # ...WITHOUT set -e, this runs in your CURRENT dir. Catastrophe.
```

**Caveats you MUST know** — `set -e` does **not** trigger when: the command is an `if`/`while`/`until` **condition** (a non-match isn't a script error); it's on the left of `&&`/`||` (`cmd || true` deliberately ignores failure); it's any pipeline stage **except the last** unless you add `pipefail`; or it's a function called in one of those contexts (`-e` is suppressed for the whole call).

### `set -u` — error on use of an unset variable

This is the guardrail that prevents disaster:

```bash
# WITHOUT set -u:  if DEPLOY_DIR is unset/typo'd, "$DEPLOY_DIR" expands to ""
rm -rf "$DEPLOY_DIR/"       # becomes  rm -rf /   ← DELETES THE ENTIRE FILESYSTEM
# WITH set -u:
rm -rf "$DEPLOY_DIR/"       # bash: DEPLOY_DIR: unbound variable → script aborts BEFORE the rm
```

### `set -o pipefail` — a pipeline fails if ANY stage fails (topic 12)

Without it, `curl ... | jq ...` reports success if `jq` succeeds even when `curl` died. With it, the pipeline's exit status is the rightmost non-zero. Essential with `set -e`.

### `IFS=$'\n\t'` — split words only on newline and tab, never spaces

The **Internal Field Separator** defaults to space-tab-newline. Setting it to just newline+tab means unquoted expansions won't split on spaces mid-filename. It's a belt-and-suspenders companion to always quoting.

### `set -x` — trace mode (debugging)

Prints every command (after expansion) with a `+` prefix before running it. Turn on with `set -x`, off with `set +x`. Invaluable for "what is this script actually doing?" (See the unset-variable proof in Hands-On.)

---

## Variables and Quoting

### Assignment — NO spaces around `=`

`name="prod"` is correct; `name = "prod"` fails with `name: command not found`. **Why:** bash parses `name = "prod"` as a command invocation (`name`) with two arguments — `=` is only an assignment operator when there's no whitespace around it. The single most common beginner error.

### `$var` vs `${var}` and scope

```bash
file="report"
echo "$file.log"       # report.log
echo "$file_v2"        # (empty!) bash looks for a variable named "file_v2"
echo "${file}_v2"      # report_v2   ← braces delimit the name explicitly
```

`local` confines a variable to a function; `readonly` makes it immutable; without `local`, all variables are **global** — a frequent source of bugs where a function clobbers a caller's variable.

### ALWAYS QUOTE YOUR VARIABLES — `"$var"`

An unquoted `$var` undergoes **word splitting** (on IFS) and then **globbing** (path expansion). Both silently mangle data.

```bash
f="my report.txt"        # a real filename with a space
rm $f                    # ❌ bash splits into TWO args: rm "my" "report.txt" → deletes wrong files / errors
rm "$f"                  # ✅ one argument, exactly the file you meant

pattern="*"              # value happens to contain a glob char
echo $pattern            # ❌ bash expands * to ALL filenames in the directory!
echo "$pattern"          # ✅ prints a literal *
```

**The one rule that prevents the most bugs:** quote every expansion — `"$var"`, `"$@"`, `"$(cmd)"` — unless you have a specific, deliberate reason not to. (Proof of the splitting in Hands-On below.)

---

## Parameter Expansion — the superpower

Pure bash string manipulation, no external `sed`/`cut` process needed:

```bash
name="${1:-default}"          # use $1, or "default" if $1 is unset OR empty
: "${DATABASE_URL:?must be set}"   # ABORT with this message if unset/empty (great for required env)
count="${count:=0}"           # assign 0 to count if it was unset, and use it
label="${DEBUG:+debug-mode}"  # expand to "debug-mode" ONLY if DEBUG is set (else empty)

path="/var/log/app.log"
echo "${#path}"               # 17         → length
echo "${path##*/}"            # app.log    → strip longest leading match of */  (basename)
echo "${path%/*}"             # /var/log   → strip shortest trailing /*         (dirname)
file="app.tar.gz"
echo "${file%.gz}"            # app.tar    → strip shortest trailing .gz
echo "${file%%.*}"            # app        → strip longest trailing .*  (all extensions)
echo "${file#*.}"             # tar.gz     → strip shortest leading *.

url="http://a/http/b"
echo "${url/http/https}"      # https://a/http/b   → replace FIRST match
echo "${url//http/https}"     # https://a/https/b  → replace ALL matches (double slash)

s="Prod"
echo "${s^^}"                 # PROD       → uppercase
echo "${s,,}"                 # prod       → lowercase
```

**The `:?` idiom is the cleanest way to demand required env vars** at the top of a deploy script: `: "${DEPLOY_HOST:?set DEPLOY_HOST}"`.

### Arrays

```bash
servers=("web-01" "web-02" "web with space")
echo "${servers[0]}"          # web-01
echo "${#servers[@]}"         # 3          → element count
for s in "${servers[@]}"; do echo "deploying $s"; done   # "${arr[@]}" = each element one word
# "${arr[*]}" joins into ONE string (via IFS's first char) — rarely what you want in a loop.
```

---

## Command Substitution and Arithmetic

```bash
now="$(date +%Y-%m-%d)"       # ✅ $(...) — nestable, readable, PREFERRED over `backticks`
count=$(( 2 + 3 * 4 ))        # 14   → arithmetic expansion
(( count++ ))                 # C-style arithmetic; increments count (no $ needed)
if (( count > 10 )); then echo big; fi    # numeric comparison with natural operators
```

---

## Conditionals — `[`, `[[`, and `((`

Three different things that look similar:

```
[ "$x" = "5" ]        →  [ is the TEST BINARY (/usr/bin/[ or a builtin). Needs SPACES around
│                        everything (they're arguments!). POSIX. Word-splits unquoted vars.
[[ "$x" == 5* ]]      →  [[ is a bash KEYWORD. Safer: no word splitting, supports ==, =~ regex,
│                        && || inside, and glob pattern matching. USE THIS in bash scripts.
(( x > 5 ))           →  arithmetic context. Bare variable names, C-style operators, no $.
```

**The `>` trap inside `[ ]`:**
```bash
[ "$a" > "$b" ]        # ❌ > is REDIRECTION here — this SILENTLY CREATES a file named "$b"!
[ "$a" -gt "$b" ]      # ✅ numeric greater-than
[[ "$a" > "$b" ]]      # ✅ string comparison ( > is safe inside [[ )
```

**Test operators:**

| Kind | Operators |
|---|---|
| String | `-z` (empty), `-n` (non-empty), `=` / `==` (equal), `!=`, `=~` (regex, `[[` only) |
| Numeric | `-eq -ne -lt -le -gt -ge` (use these in `[ ]`; use `< > == ` only in `(( ))`) |
| File | `-e` exists, `-f` regular file, `-d` dir, `-r/-w/-x` readable/writable/executable, `-s` non-empty, `-L` symlink |

```bash
if [[ -f "$config" && -r "$config" ]]; then source "$config"   # exists AND readable
elif [[ -z "${config:-}" ]]; then echo "no config given" >&2; exit 1   # empty/unset
fi

case "$1" in
  start)   systemctl start api ;;
  stop)    systemctl stop  api ;;
  restart) systemctl restart api ;;
  *)       echo "usage: $0 {start|stop|restart}" >&2; exit 2 ;;
esac
```

---

## Loops — and the traps that ruin them

```bash
for f in *.log; do echo "$f"; done          # ✅ glob — handles spaces, empty-safe with nullglob
for i in {1..5}; do echo "$i"; done         # brace range
for ((i=0; i<5; i++)); do echo "$i"; done   # C-style
```

**`for f in $(ls)` is BROKEN** — never do it. `$(ls)` produces one big string that bash **word-splits on spaces**, so `my file.log` becomes two iterations `my` and `file.log`. Use a glob (`for f in *.log`) or, for recursion, `find ... -print0 | while IFS= read -r -d '' f`.

**Reading a file line by line — the ONE correct incantation:**
```bash
while IFS= read -r line; do echo "processing: $line"; done < "$file"
```
`IFS=` → don't trim whitespace; `-r` → don't interpret backslashes (else `\n` in data is mangled); `< "$file"` → feed the file into the loop's stdin (topic 12).

### The subshell gotcha — "my counter is always 0"

```bash
count=0
grep ERROR app.log | while read -r line; do
  ((count++))                 # increments count... in a SUBSHELL
done
echo "$count"                 # 0  ← the subshell's variables VANISHED when the pipe ended
```
**Root cause:** each side of a `|` runs in its own **subshell** (a forked child — topic 07). Variables set in the child don't propagate back to the parent. **Fix** with process substitution so the loop runs in the *current* shell:
```bash
count=0
while read -r line; do ((count++)); done < <(grep ERROR app.log)
echo "$count"                 # 42  ✅
# (or: shopt -s lastpipe  — runs the last pipeline stage in the current shell)
```

---

## Functions

```bash
deploy() {                    # define
  local host="$1"             # local → confined to this function
  local ref="${2:-main}"      # default arg
  echo "deploying $ref to $host"
  return 0                    # return sets the EXIT CODE (0-255), NOT a value
}
deploy "web-01" "v2.1"        # call: args are $1, $2, ...
```

**Critical:** `return` returns an **exit code** (0–255), *not* data. To return a value, **echo it** on stdout and capture with `$()`: `current_release() { readlink /srv/app/current; }` then `rel="$(current_release)"`.

**Special variables inside functions/scripts:**

| Var | Meaning |
|---|---|
| `$?` | Exit code of the last command |
| `$0` | Script name |
| `$1`–`$9`, `${10}` | Positional arguments |
| `$#` | Number of arguments |
| `"$@"` | All args, **each as a separate quoted word** (almost always what you want) |
| `$*` | All args as one word (joined by IFS) |
| `$$` | PID of the current shell |
| `$!` | PID of the last backgrounded (`&`) command |
| `$_` | Last argument of the previous command |

**Why `"$@"` beats `$*`:** `"$@"` preserves argument boundaries so `myscript "a b" c` forwards exactly two arguments; `$*` mashes everything into one string and re-splits it. Use `"$@"` to forward args to another command.

---

## Exit Codes — and why 137 haunts you

Every command returns 0–255. **0 = success**, anything else = failure. Conventions:

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 2 | Shell builtin misuse (bad flags) |
| 126 | Command found but **not executable** (missing `chmod +x`) |
| 127 | **Command not found** (typo, or not on PATH) |
| **128 + N** | Killed by **signal N** |
| **130** | 128 + 2 = SIGINT — you pressed **Ctrl-C** |
| 137 | 128 + 9 = **SIGKILL** — the **OOM killer** killed you (or `docker stop` timeout) |
| 143 | 128 + 15 = SIGTERM — graceful stop requested |

**137 is the one every Node dev meets.** Your container shows `Exited (137)`: the Linux OOM killer (topic 01) sent SIGKILL because the process exceeded its memory limit. It is **not** a bug in your code's logic — it's memory. `docker inspect --format='{{.State.OOMKilled}}' <id>` confirms it. (Proof that signal N → exit 128+N is in Hands-On below.)

---

## trap, mktemp, getopts — writing robust scripts

### `trap` — guaranteed cleanup

```bash
tmp="$(mktemp)"                          # SAFE temp file — random name, no collisions
trap 'rm -f "$tmp"' EXIT                 # run this on ANY exit (success, error, or set -e abort)
trap 'echo "interrupted"; exit 130' INT TERM   # graceful shutdown on Ctrl-C / kill
# ... use "$tmp" freely; it WILL be cleaned up no matter how the script ends ...
```

**Never hardcode `/tmp/myfile`** — it's a symlink-attack vector and collides between concurrent runs. `mktemp` creates a uniquely-named file (or `mktemp -d` for a directory) with safe permissions.

### `getopts` — flag parsing

```bash
verbose=0; env="staging"
while getopts "ve:" opt; do
  case "$opt" in
    v) verbose=1 ;;
    e) env="$OPTARG" ;;         # OPTARG holds the value for flags that take one (the ':')
    *) echo "usage: $0 [-v] [-e env]" >&2; exit 2 ;;
  esac
done
shift $((OPTIND - 1))           # drop the parsed options, leaving positional args in $@
```

### Debugging

`bash -x script.sh` (or `set -x` inside) traces execution. Customize the trace prefix with `PS4='+ ${BASH_SOURCE}:${LINENO}: '` to show file and line numbers. And run **`shellcheck script.sh`** on everything — it statically catches unquoted vars, `[ ]` traps, and subshell bugs before they reach production. Install it; use it religiously.

---

## Exact Syntax Breakdown

```
while IFS= read -r line; do ... done < "$file"
│     │    │    │  │            │      └── redirect the file into the loop's stdin (fd 0)
│     │    │    │  └── the variable each line is read into
│     │    │    └── -r = raw: do NOT let backslashes escape characters
│     │    └── read = shell builtin that reads one line from stdin
│     └── IFS= = empty field separator FOR THIS COMMAND ONLY → don't trim whitespace
└── while = loop as long as read succeeds (read returns non-zero at EOF, ending the loop)
```

```
: "${DATABASE_URL:?must be set}"
│ │  │            ││
│ │  │            │└── the error message printed to stderr if the check fails
│ │  │            └── :? = "if unset OR empty, print message and EXIT non-zero"
│ │  └── the variable being checked
│ └── quotes so a value with spaces stays one word
└── the ":" builtin: a no-op whose only job is to evaluate its argument (the expansion)
```

```
trap 'rm -f "$tmp"' EXIT
│    │              └── the SIGNAL/event to trap on (EXIT = whenever the shell exits)
│    └── the command(s) to run when it fires (single-quoted so $tmp expands AT TRAP TIME)
└── trap = install a signal/event handler
```

---

## Example 1 — Basic

A commented script showing the core building blocks:

```bash
#!/usr/bin/env bash
set -euo pipefail                       # strict mode: fail loud, fail early
IFS=$'\n\t'

greeting="${1:-world}"                  # first arg, or "world" if none given
count="${2:-3}"                         # second arg, or 3

shout() {                               # a function that returns a value by ECHOING it
  local msg="$1"
  echo "${msg^^}!"                      # uppercase + exclamation
}

for (( i=1; i<=count; i++ )); do        # C-style loop, 'count' times
  echo "$i: $(shout "hello $greeting")" # capture the function's echoed value
done

if (( count > 5 )); then echo "that's a lot of greetings"; fi   # numeric conditional
```
```
$ ./greet.sh Ada 2
1: HELLO ADA!
2: HELLO ADA!
```

---

## Example 2 — Production Scenario: a real deploy script

**Situation:** you deploy a Node app to `/srv/api` behind systemd + nginx. A genuinely correct zero-downtime deploy: strict mode, required-env checks, trap cleanup, atomic symlink swap, health-check retry loop, automatic rollback.

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# ---- required configuration (fail immediately if missing) --------------------
: "${GIT_REF:?set GIT_REF (branch or tag to deploy)}"
APP_ROOT="/srv/api"
RELEASES="$APP_ROOT/releases"
CURRENT="$APP_ROOT/current"                       # symlink → the live release
HEALTH_URL="http://127.0.0.1:3000/health"
ts="$(date +%Y%m%d-%H%M%S)"
new_release="$RELEASES/$ts"

log() { echo "[$(date +%H:%M:%S)] $*"; }          # timestamped logging helper

# ---- cleanup runs on ANY exit, success or failure ----------------------------
cleanup() {
  local code=$?
  if (( code != 0 )); then
    log "FAILED (exit $code) — removing half-built release $new_release"
    rm -rf "$new_release"
  fi
}
trap cleanup EXIT

# ---- remember the current release so we can roll back ------------------------
previous="$(readlink -f "$CURRENT" 2>/dev/null || echo "")"

# ---- build the new release ---------------------------------------------------
log "creating $new_release"
mkdir -p "$new_release"
git -C "$APP_ROOT/repo" fetch --quiet origin
git -C "$APP_ROOT/repo" archive "$GIT_REF" | tar -x -C "$new_release"

log "installing dependencies"
( cd "$new_release" && npm ci --omit=dev )        # subshell so we don't change our own cwd
( cd "$new_release" && npm run build )

# ---- atomic switch: symlink swap is a single rename() syscall ----------------
log "activating release $ts"
ln -sfn "$new_release" "$CURRENT.tmp"              # -f force, -n don't follow existing symlink
mv -T "$CURRENT.tmp" "$CURRENT"                    # mv -T = atomic replace of the symlink
sudo systemctl restart api

# ---- health check with retry loop --------------------------------------------
log "waiting for health check"
healthy=0
for attempt in {1..10}; do
  if curl -fsS -o /dev/null "$HEALTH_URL"; then
    healthy=1; log "healthy on attempt $attempt"; break
  fi
  sleep 2
done

# ---- rollback on failure -----------------------------------------------------
if (( healthy == 0 )); then
  log "health check FAILED — rolling back"
  if [[ -n "$previous" ]]; then
    ln -sfn "$previous" "$CURRENT.tmp" && mv -T "$CURRENT.tmp" "$CURRENT"
    sudo systemctl restart api
    log "rolled back to $previous"
  fi
  exit 1                                           # trap cleanup removes the bad release
fi

log "deploy of $GIT_REF complete"
```

Realistic run:
```
[02:31:09] creating /srv/api/releases/20260712-023109
[02:31:14] installing dependencies
[02:31:52] activating release 20260712-023109
[02:31:59] healthy on attempt 3
[02:31:59] deploy of v2.4.0 complete
```

Every safety property is deliberate: strict mode aborts on the first failing command, `:?` refuses to run without `GIT_REF`, the `trap` guarantees the half-built release is cleaned up, and the symlink swap means the app is *never* pointed at a half-copied directory.

---

## Common Mistakes

### Mistake 1 — Spaces around `=` in assignment

**Wrong:** `PORT = 3000` → `bash: PORT: command not found`
**Right:** `PORT=3000`
**Root cause:** with spaces, bash parses `PORT` as a **command** and `=`/`3000` as its arguments. Assignment syntax requires *no* whitespace around `=`. **Prevention:** shellcheck flags this instantly.

### Mistake 2 — Unquoted variables

**Wrong:** `cp $src $dst` where `src="my file.txt"` → copies TWO files `my` and `file.txt`.
**Right:** `cp "$src" "$dst"`
**Root cause:** unquoted `$src` undergoes word-splitting (on IFS) then globbing. **Diagnose:** `set -x` shows the command expanding into extra arguments. **Prevention:** quote every expansion; run shellcheck.

### Mistake 3 — `[ $x -gt 5 ]` with an empty or spaced variable

**Wrong:** `if [ $count -gt 5 ]` — if count is empty this becomes `[ -gt 5 ]` → "unary operator expected" error.
**Right:** `if [ "$count" -gt 5 ]` or better `if (( count > 5 ))`.
**Root cause:** `[` is a **program** receiving arguments; an empty unquoted `$count` disappears entirely, leaving `[` with malformed arguments. **Prevention:** quote inside `[ ]`, or use `(( ))` for numbers and `[[ ]]` for strings.

### Mistake 4 — The `cmd | while read` subshell (counter stays 0)

**Wrong:** `cat f | while read l; do ((n++)); done; echo $n` → prints `0`.
**Root cause:** the `while` runs in a subshell (right side of the pipe — topic 07/12), so `n` is modified in a child that then exits. **Fix:** `while read l; do ((n++)); done < <(cat f)` (process substitution) or `shopt -s lastpipe`. **Prevention:** never pipe *into* a loop whose side effects you need afterward.

### Mistake 5 — Missing `set -e`/`set -u` → destructive chaining

**Wrong:** `cd "$build_dir"` (silently fails if empty/missing) then `rm -rf ./*` — which now deletes the **current** directory's contents.
**Root cause:** without `set -e` the failed `cd` is ignored and execution continues in the wrong directory; without `set -u` an empty `$build_dir` is silently accepted. **Fix:** `set -euo pipefail` at the top, and prefer `cd "$dir" || exit 1`. **Prevention:** strict mode on every script; run `bash -n` (syntax check) and shellcheck in CI.

---

## Hands-On Proof

```bash
# PROVE IT: the kernel reads the shebang and routes to the interpreter
printf '#!/bin/cat\nI am my own output\n' > /tmp/s && chmod +x /tmp/s && /tmp/s

# PROVE IT: /bin/sh is dash, and bashisms fail under it
ls -l /bin/sh
sh -c 'declare -A m 2>&1' ; echo "sh exit: $?"     # dash: declare: not found → non-zero

# PROVE IT: set -u stops the empty-variable disaster
bash -c 'set -u; dir=""; echo "would rm -rf ${dir}/"' 2>&1   # aborts? No — dir IS set(empty).
bash -c 'set -u; echo "${UNSET_VAR}/"' 2>&1                  # THIS aborts: unbound variable

# PROVE IT: word splitting vs quoting
touch "a b.c"; v="a b.c"
printf 'unquoted:'; for x in $v;   do printf ' [%s]' "$x"; done; echo   # [a] [b.c]
printf 'quoted:  '; for x in "$v"; do printf ' [%s]' "$x"; done; echo   # [a b.c]

# PROVE IT: the > trap inside [ ] silently creates a file
rm -f 5; [ 9 > 5 ]; ls -l 5      # a file named "5" now exists! Use -gt or (( )).

# PROVE IT: exit code 128 + N for signals
sleep 60 & kill -9 $!; wait $! 2>/dev/null; echo "SIGKILL → $?"    # 137
bash -c 'kill -INT $$'; echo "SIGINT → $?"                        # 130

# PROVE IT: the subshell counter bug and its fix
n=0; printf 'a\nb\nc\n' | while read -r x; do ((n++)); done; echo "piped n=$n"   # 0
n=0; while read -r x; do ((n++)); done < <(printf 'a\nb\nc\n'); echo "procsub n=$n"  # 3

# PROVE IT: trap fires even on error under set -e
bash -c 'trap "echo CLEANUP RAN" EXIT; set -e; false; echo "never prints"'
# CLEANUP RAN        ← the trap ran even though `false` aborted the script
```

---

## Practice Exercises

### Exercise 1 — Easy

Write `backup.sh` that: has a proper shebang, enables strict mode, takes a directory as `$1` (defaulting to `$HOME` with `${1:-$HOME}`), refuses to run if the directory doesn't exist (`[[ -d ]]`), and prints `"backing up <dir> (<N> files)"` where N is `$(ls "$dir" | wc -l)`. Run it with and without an argument, and against a nonexistent path.

### Exercise 2 — Medium

Write `logstats.sh <logfile>` that: uses `${1:?usage: logstats.sh <logfile>}` to require the argument, reads the file with `while IFS= read -r line`, counts total lines and lines containing "ERROR" (using two counters), and prints both plus the error percentage via `$(( ))`. Prove the counters are correct (they must NOT be 0 — so do **not** pipe into the loop). Then break it deliberately by piping `cat file |` into the loop and explain why the counts become 0.

### Exercise 3 — Hard (Production Simulation)

Write `healthcheck.sh` that parses flags with `getopts`: `-u <url>` (required), `-r <retries>` (default 5), `-v` (verbose). It curls the URL in a retry loop with a 2-second sleep, prints timestamped log lines, installs a `trap` that prints `"cleaning up"` on exit, and exits `0` on the first success or `1` after exhausting retries. Run `shellcheck healthcheck.sh` and fix every warning. Then run it against a URL that 500s and confirm the exit code with `echo $?`.

---

## Mental Model Checkpoint

1. **Who reads the shebang — bash or the kernel — and what exactly does `#!/usr/bin/env bash` do differently from `#!/bin/bash`?**
2. **What are the four parts of `set -euo pipefail` / `IFS`, and which one prevents `rm -rf "$X/"` from wiping the disk?**
3. **Why does `x = 1` fail but `x=1` work?**
4. **Show a filename that breaks `cp $src $dst` but works with `cp "$src" "$dst"`, and explain the two expansions involved.**
5. **What is the difference between `[`, `[[`, and `((`, and what does `[ $a > $b ]` silently do?**
6. **Why does `count` stay 0 after `cat f | while read; do ((count++)); done`, and how do you fix it?**
7. **Does `return 42` from a function give you the value 42? How do you actually return data from a function?**
8. **Your container exited 137. What killed it, what signal, and how do you confirm it?**

---

## Quick Reference Card

| Construct | What It Does |
|---|---|
| `#!/usr/bin/env bash` | Portable shebang — finds bash on PATH |
| `set -e` | Exit on any command failure |
| `set -u` | Error on unset variable use |
| `set -o pipefail` | Pipeline fails if any stage fails |
| `set -x` / `+x` | Turn command tracing on / off |
| `var="value"` | Assign (NO spaces around `=`) |
| `"$var"` `"$@"` | Quoted expansion (prevents splitting/globbing) |
| `${var:-default}` | Value or default if unset/empty |
| `${var:?msg}` | Abort with msg if unset/empty |
| `${var##*/}` / `${var%/*}` | basename / dirname via stripping |
| `${var//old/new}` | Replace all occurrences |
| `${var^^}` / `${var,,}` | Upper / lower case |
| `"${arr[@]}"` | All array elements, each a separate word |
| `$(cmd)` | Command substitution (nestable) |
| `$(( expr ))` / `(( expr ))` | Arithmetic |
| `[[ ... ]]` | Bash test keyword (safe: `=~`, `&&`, patterns) |
| `[ ... ]` | POSIX test binary (needs spaces; quote vars!) |
| `while IFS= read -r line; do…done < f` | Read a file line by line, correctly |
| `<(cmd)` | Process substitution (fixes the loop-subshell bug) |
| `$?` `$#` `"$@"` `$$` `$!` | exit code · arg count · args · PID · last bg PID |
| `trap 'cmd' EXIT` | Run cmd on exit (cleanup) |
| `mktemp` / `mktemp -d` | Safe temp file / directory |
| `getopts "ve:" opt` | Parse `-v` and `-e <arg>` flags |
| `shellcheck script.sh` | Static analysis — run on everything |

---

## When Would I Use This at Work?

### Scenario 1: A cron-driven database backup
A nightly script that dumps Postgres, gzips it, uploads to S3, and prunes old backups — with strict mode so a failed dump aborts *before* it deletes yesterday's backup, `mktemp` for the working file, and a `trap` to clean up on any exit. One silent `set -e` omission here means a failed dump plus a successful prune = **zero backups** the night you need one.

### Scenario 2: A deploy/rollback script (Example 2)
The atomic-symlink deploy above. You run it from CI or by hand. Strict mode, required-env checks, health-check retry loop, and automatic rollback turn "deploy" from a nail-biting manual ritual into one idempotent command that either succeeds or safely reverts.

### Scenario 3: An incident triage one-liner grown into a script
At 2am you keep typing the same five commands to check disk, memory, the app's health endpoint, recent errors in the journal, and open connections on port 3000. You wrap them in a `triage.sh` with `getopts` for `-v`, timestamped logging, and exit codes so it can also run from monitoring. Now anyone on-call runs one command instead of remembering five.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 07 — How the Shell Works | fork/exec, subshells, and why `cmd \| while` loses variables |
| **Builds on** | 11 — Text Processing | grep/sed/awk are the commands your scripts orchestrate |
| **Builds on** | 12 — Pipes and Redirection | `set -o pipefail`, `2>&1`, `< file`, process substitution `< <(cmd)` |
| **Next** | 14 — Environment Variables | `export`, `source`, `.bashrc` — how scripts inherit and share config |
| **Used by** | 17 — Process Management | signals and exit codes 128+N (137, 143) come from here |
| **Used by** | 20 — Cron and Scheduling | every cron job is a script; strict mode and `< /dev/null` matter most there |
| **Used by** | 22 — systemd / 33 — Node in Production | `ExecStart` scripts, graceful shutdown traps, health checks |
| **Used by** | 32 — Deploy User Permissions | the deploy script runs as a restricted user with specific sudoers rights |
