# 10 — .dockerignore

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine you're mailing a box of your favorite toys to a friend.

Before you ship it, you have to **carry the whole box to the post office**. If
you dump your entire bedroom into the box — dirty socks, old homework, the
family cat — the box is huge, heavy, and slow to carry. And some of it (like
your diary with secrets) you really didn't want to send at all.

A `.dockerignore` file is a **note taped to your bedroom door** that says:
"When packing the box, skip the dirty socks, skip the homework, and definitely
skip the diary." Now you carry a small, light box with only the toys — fast to
move, and no secrets leaked.

In Docker, "carrying the box to the post office" = sending the **build context**
to the builder. `.dockerignore` tells Docker what to leave out of that box.

---

## The Linux kernel feature underneath

There's no exotic kernel feature here — the important mechanism is how the
**Docker client and daemon are separated**, and how files physically travel
between them.

Remember from Topic 03 (Docker Architecture): `docker` (the CLI/client) and
`dockerd` (the daemon) are **separate processes** that talk over a socket —
usually the Unix socket at `/var/run/docker.sock`. The daemon is what actually
builds images. The daemon **cannot see your project directory directly**; it
only sees files you send it.

So when you run `docker build .`, this happens at the OS level:

1. The CLI reads the directory you pointed at (`.` = the **build context root**).
2. It walks that directory tree, and for each file **not excluded** by
   `.dockerignore`, it **streams the bytes** into a tar archive.
3. That tar stream is written into the socket and read by `dockerd` (or, with
   BuildKit, sent to the `buildkitd` build process).
4. BuildKit unpacks it into a temporary location and uses it as the source for
   every `COPY`/`ADD`.

```
Your machine
┌─────────────────────────────────────────────────────────┐
│  ./orders-api/                                            │
│    ├── src/          ─┐                                   │
│    ├── package.json  ─┤ CLI tars ONLY non-ignored files  │
│    ├── node_modules/ ─┼──✗ excluded by .dockerignore     │
│    ├── .git/         ─┼──✗ excluded                       │
│    └── .env          ─┼──✗ excluded (secret!)            │
│                       │                                   │
│   docker CLI ─── tar stream ──▶ /var/run/docker.sock      │
└──────────────────────────────────────│──────────────────┘
                                        ▼
                              dockerd / buildkitd
                         unpacks context → used by COPY
```

The key kernel-adjacent truth: **every byte of the build context crosses a
socket and gets written to disk on the daemon side before the build even
starts.** If your context is 1.2 GB (because `node_modules` and `.git` came
along), you pay that transfer and disk-write cost on **every single build**,
even if your Dockerfile only `COPY`s three files. `.dockerignore` cuts that cost
at the source — the files never enter the tar in the first place.

---

## What is this?

`.dockerignore` is a plain text file at the **root of your build context**
(next to the file you build from). It lists glob patterns for files and
directories that the Docker CLI should **exclude when packing the build
context**. It works like `.gitignore`, but for what gets sent to the Docker
daemon.

---

## Why does it matter for a backend developer?

Skipping `.dockerignore` on your `orders-api` project causes four real
problems:

- **Slow, heavy builds.** Sending `node_modules` (often 300 MB–1 GB) and `.git`
  (can be hundreds of MB with history) on every build wastes seconds to minutes.
  You'll see `Sending build context to Docker daemon  1.2GB` scroll by before
  anything useful happens.
- **Cache thrash.** If `COPY . .` is in your Dockerfile and a junk file changes
  (a log file, a `.DS_Store`, an editor swap file), Docker sees the context
  changed and **busts your layer cache** (Topic 08), forcing an unnecessary
  rebuild of `npm ci` and everything after it.
- **Secret leakage.** `COPY . .` will happily copy `.env`, `.aws/credentials`,
  private keys, and `.git` (which contains your entire history, including
  secrets you thought you deleted) **into your image**. Anyone who pulls the
  image can extract them.
- **Broken/oversized images.** Copying a host `node_modules` built for macOS
  into a Linux image gives you native binaries compiled for the wrong OS —
  runtime crashes — plus bloats the image with files you should install fresh
  with `npm ci`.

One line of `.dockerignore` (`node_modules`) can cut a build from 90 seconds to
8 seconds and shave a gigabyte off the transfer.

---

## The physical reality

Watch what actually happens without and with `.dockerignore`.

**Without** `.dockerignore`, the legacy builder literally prints the context
size, and it's written to the daemon's temp dir:

```
$ docker build -t orders-api .
Sending build context to Docker daemon  1.24GB     ← node_modules + .git shipped
Step 1/7 : FROM node:20 ...
```

On the daemon side, the context is unpacked into a temporary build directory
(BuildKit keeps it in its own state dir):

```
/var/lib/docker/tmp/                      ← legacy builder unpacks context here
/var/lib/docker/buildkit/                 ← BuildKit's snapshot/cache state
```

**With** a good `.dockerignore`:

```
$ docker build -t orders-api .
[+] Building 0.4s (9/9) FINISHED
 => transferring context: 84.21kB          ← 1.24GB dropped to 84 KB
```

That number — `transferring context` (BuildKit) or `Sending build context`
(legacy) — is your report card. If it's large, your `.dockerignore` is missing
or wrong.

You can prove exactly what would be sent by building a "print the context"
image:

```bash
# See every file that WOULD be in the context (respects .dockerignore)
$ tar -cvf /dev/null . 2>/dev/null   # not context-aware; use the trick below
```

BuildKit-accurate way to inspect context contents:

```bash
$ echo -e "FROM busybox\nCOPY . /ctx\nRUN find /ctx -maxdepth 2" > Dockerfile.ctx
$ docker build -f Dockerfile.ctx --no-cache --progress=plain . 2>&1 | grep /ctx
# Anything listed here made it past .dockerignore
```

---

## How it works — step by step

Trace of `docker build -t orders-api .` with a `.dockerignore` present:

1. **CLI locates the context root.** The `.` argument means "the current
   directory is the build context root." The CLI looks for `.dockerignore` **in
   that root** (not next to the Dockerfile if the Dockerfile is elsewhere — it's
   the *context* root that matters).

2. **CLI parses `.dockerignore` into a pattern list.** Each non-empty,
   non-comment line becomes a pattern. Order matters because of negation (`!`)
   rules — later patterns can re-include what earlier ones excluded.

3. **CLI walks the directory tree.** For every path, it tests the path against
   the pattern list. If the path matches an exclude pattern (and isn't rescued
   by a later `!` negation), the CLI **skips it** — it's never read, never
   tarred.

4. **CLI builds the tar stream** from the surviving files and streams it to the
   daemon/BuildKit over the socket.

5. **BuildKit unpacks the context.** Now `COPY`/`ADD` instructions can only see
   files that survived step 3. If `.env` was excluded, then `COPY . .`
   **physically cannot** copy `.env` — it's not in the context at all.

6. **The Dockerfile and `.dockerignore` themselves.** By default the
   `Dockerfile` and `.dockerignore` are excluded from what `COPY . .` brings in
   (BuildKit special-cases them), so they don't end up inside your image.

Crucial mental note: `.dockerignore` filters the **context**, which is the
*input* to `COPY`. It does **not** filter what `COPY` targets — if a file made
it into the context, a `COPY that-file` still copies it. `.dockerignore` and
`COPY` are two gates in a row: context filter first, then COPY selection.

---

## Exact syntax breakdown

`.dockerignore` patterns use Go's `filepath.Match` semantics (similar to, but
not identical to, `.gitignore`).

### Basic exclude

```
node_modules
│
└─ match a file or dir named "node_modules" at ANY depth of the context
```

### Directory-scoped exclude

```
/build
│
└─ leading slash = anchor to context ROOT only. Matches ./build, not ./src/build
```

### Wildcards

```
*.log
│ │
│ └─ "log" literal extension
└─── * matches any run of non-separator characters → app.log, error.log
```

```
**/*.tmp
│  │
│  └─ any .tmp file
└──── ** matches any number of directories (deep match) → a/b/c/x.tmp
```

### Single-character wildcard

```
temp?
     │
     └─ ? matches exactly ONE character → temp1, tempA (not temp or temp10)
```

### Comment

```
# ignore local env files
│
└─ lines starting with # are comments, ignored by the parser
```

### Negation — re-include something you excluded

```
node_modules
!node_modules/.keep
│
└─ ! un-ignores a path a PREVIOUS pattern excluded. Order matters:
   the ! line must come AFTER the exclude it overrides.
```

### Exclude everything, then allow-list (safelist pattern)

```
*                    ← exclude EVERYTHING in the context
!package.json        ← but keep package.json
!package-lock.json   ← and the lockfile
!src/                ← and the src directory
│
└─ powerful for locked-down builds: start empty, add only what you need
```
Caveat: with `*` first, to re-include files *inside* a directory you must first
re-include the directory itself, e.g. `!src` then `!src/**` may be needed
depending on structure. Test with the inspection trick above.

---

## Example 1 — basic

A solid, minimal `.dockerignore` for `orders-api`. Every line explained.

```dockerignore
# --- dependencies: always reinstall inside the image with npm ci ---
node_modules          # host node_modules may be wrong-OS binaries + huge
npm-debug.log         # npm crash logs, useless in the image

# --- version control & CI ---
.git                  # entire repo history — big AND leaks old secrets
.gitignore
.github               # CI workflows, not needed at runtime

# --- secrets & local config (NEVER ship these) ---
.env                  # database URLs, API keys — must NOT enter the image
.env.*                # .env.local, .env.development, etc.
*.pem                 # private keys
*.key

# --- editor / OS junk (busts cache, adds nothing) ---
.DS_Store             # macOS Finder metadata
.vscode
.idea
*.swp

# --- build output & test noise ---
dist                  # rebuilt inside the image; host copy is stale
coverage              # test coverage reports
*.log
```

Build and confirm the context shrank:

```
$ docker build -t orders-api .
[+] Building ...
 => transferring context: 74.9kB     ← tiny, because node_modules/.git are excluded
```

---

## Example 2 — production scenario

**The situation:** A teammate reports "the `orders-api` image is 60 MB bigger
than the Dockerfile suggests, and CI leaked our staging database password."
Security did a scan and found `.env.staging` **inside** the shipped image, plus
the whole `.git` directory. Here's how it happened and how `.dockerignore`
fixes it.

The Dockerfile had:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .              # ← copies EVERYTHING in the context, incl. .env.staging + .git
RUN npm ci --omit=dev
CMD ["node", "dist/server.js"]
```

There was **no** `.dockerignore`. So `COPY . .` swept in `.env.staging` and
`.git/`. Anyone who pulled the image could run:

```
$ docker run --rm orders-api cat .env.staging
DATABASE_URL=postgres://app:SuperSecret@staging-db:5432/orders
REDIS_URL=redis://staging-redis:6379
```

Secret leaked. And `.git` let them reconstruct old commits.

**The fix** — add a strict `.dockerignore` (allow-list style for a locked-down
service):

```dockerignore
# Start by excluding EVERYTHING
*

# Re-include ONLY what the build genuinely needs
!package.json
!package-lock.json
!tsconfig.json
!src/**
```

Now rebuild and verify the secret is gone:

```
$ docker build -t orders-api .
 => transferring context: 41.3kB
$ docker run --rm orders-api sh -c 'ls -a | grep -E "\.env|\.git" || echo "clean ✓"'
clean ✓
```

**Defense in depth:** `.dockerignore` prevents secrets from entering the
context, but you should *also* pass real secrets at **runtime** via environment
variables or secret mounts (Topic 15), never bake them in. `.dockerignore` is
the safety net that stops accidental `COPY . .` leaks.

---

## Common mistakes

### Mistake 1 — No `.dockerignore` at all, with `COPY . .`

```
$ docker build -t orders-api .
Sending build context to Docker daemon  1.24GB
```
**Symptom:** slow builds, cache busted by junk files, secrets/`.git` in the
image.
**Root cause:** the CLI tars the whole directory because nothing tells it not
to. `COPY . .` then copies all of it into a layer.
**Fix:** add a `.dockerignore` that at minimum excludes `node_modules`, `.git`,
and `.env*`.

### Mistake 2 — Putting `.dockerignore` in the wrong directory

```
project/
├── docker/Dockerfile
├── .dockerignore          # ← wrong place if context root is project/app
└── app/
```
```
$ docker build -f docker/Dockerfile app/     # context root is app/
```
**Symptom:** your ignore rules seem ignored; context is still huge.
**Root cause:** `.dockerignore` must live at the **context root** (`app/`
here), not next to the Dockerfile.
**Fix:** put `.dockerignore` at whatever directory you pass as the build
context argument.

### Mistake 3 — Expecting `.dockerignore` to filter `COPY` targets

```dockerignore
secrets.json
```
```dockerfile
COPY secrets.json ./     # you explicitly named the excluded file
```
```
COPY failed: file not found in build context or excluded by .dockerignore:
  stat secrets.json: file does not exist
```
This one's actually correct behavior — but people get confused the *other* way:
they think `.dockerignore` will trim files out of a `COPY somedir ./` that
*includes* them. It will — the context filter runs first. The mistake is
assuming `.dockerignore` only affects `COPY . .`. It affects **all** copies,
because it removes files from the context entirely.

### Mistake 4 — Wrong wildcard depth

```dockerignore
*.log        # only matches .log files, but NOT logs/app.log at deeper paths?
```
Actually `*.log` matches at any depth in `.dockerignore` — but people often
write `/*.log` (anchored to root) and are surprised `src/debug.log` still ships.
**Fix:** use `**/*.log` when you truly mean "any `.log` anywhere," and reserve a
leading `/` for "root only."

### Mistake 5 — Negation order wrong

```dockerignore
!important.env      # re-include BEFORE the exclude — has no effect
.env*
```
**Symptom:** `important.env` still excluded.
**Root cause:** patterns are evaluated in order; the later `.env*` re-excludes
it. **Fix:** put the `!` negation **after** the broader exclude:
```dockerignore
.env*
!important.env
```

---

## Hands-on proof

```bash
# 1) Set up a project with junk that should NOT ship
mkdir -p ctx-demo/src && cd ctx-demo
echo 'console.log("orders-api")' > src/server.js
echo '{"name":"orders-api","version":"1.0.0"}' > package.json
echo 'DATABASE_URL=postgres://app:SECRET@db:5432/orders' > .env
git init -q && mkdir -p node_modules/junk && dd if=/dev/zero of=node_modules/junk/big bs=1m count=200 2>/dev/null
dd if=/dev/zero of=.git/bigblob bs=1m count=50 2>/dev/null

cat > Dockerfile <<'EOF'
FROM busybox
WORKDIR /app
COPY . .
CMD ["sh"]
EOF

# 2) Build WITHOUT .dockerignore — watch the context size
docker build -t ctx:before . 2>&1 | grep -i "transferring context"
#   => transferring context: 262.1MB     ← node_modules + .git shipped
docker run --rm ctx:before sh -c 'cat .env; ls -la .git 2>/dev/null | head -1'
#   SECRET is printed, .git is present — BAD

# 3) Add a .dockerignore
cat > .dockerignore <<'EOF'
node_modules
.git
.env
.env.*
*.log
EOF

# 4) Rebuild — context collapses, secret gone
docker build -t ctx:after . 2>&1 | grep -i "transferring context"
#   => transferring context: 3.1kB       ← 262MB -> 3KB
docker run --rm ctx:after sh -c 'ls -a | grep -E "\.env|\.git" || echo "clean ✓"'
#   clean ✓

# 5) Prove exactly what's in the context now
docker build --no-cache --progress=plain -t ctx:after . 2>&1 | grep -i "load build context"
```

You just watched a 262 MB context (with a leaked password) become a 3 KB, clean
context by adding five lines.

---

## Practice exercises

### Exercise 1 — easy
Create a project with `node_modules`, `.git`, and a `.env`. Build once with no
`.dockerignore` and record the `transferring context` size. Add a
`.dockerignore` excluding those three, rebuild, and record the new size.
Calculate the reduction factor.

### Exercise 2 — medium
Write an **allow-list** `.dockerignore` that starts with `*` and re-includes
only `package.json`, `package-lock.json`, and `src/`. Prove with the
build-context inspection trick (`FROM busybox` + `COPY . /ctx` + `find`) that
nothing else made it into the context. Then add a new required file
`tsconfig.json` and update the `.dockerignore` to include it.

### Exercise 3 — hard (production simulation)
Your `orders-api` repo has: `src/`, `test/`, `node_modules/`, `.git/`,
`.env.local`, `.env.production`, `dist/`, `coverage/`, and `docs/`. The
production image must contain **only** what runs the app (no tests, no docs, no
env files, no coverage). Write a `.dockerignore` that achieves this. Then modify
one junk file (e.g. touch `docs/README.md`) and prove — using
`docker build` twice and comparing — that the change does **not** bust the
`npm ci` layer cache (because docs never enter the context). Explain in two
sentences how `.dockerignore` protects your layer cache from Topic 08.

---

## Mental model checkpoint

Answer from memory:

1. What is the "build context," and where does it physically travel?
2. Why does a large context slow down **every** build even if `COPY` only grabs
   two files?
3. Where must the `.dockerignore` file live?
4. Name three things you should *always* exclude, and why.
5. Does `.dockerignore` filter what `COPY` targets, or what enters the context?
   How are those two gates ordered?
6. How does `.dockerignore` protect your layer cache?
7. What does the `!` prefix do, and why does its position in the file matter?

---

## Quick reference card

| Pattern | Meaning | Key detail |
|---|---|---|
| `node_modules` | Exclude by name at any depth | Most important single line |
| `/build` | Anchor to context root only | Leading `/` = root, not deep |
| `*.log` | Wildcard extension | `*` = non-separator chars |
| `**/*.tmp` | Match at any directory depth | `**` = any number of dirs |
| `temp?` | Single-char wildcard | `?` = exactly one char |
| `!keep.txt` | Re-include (negate) | Must come AFTER the exclude |
| `*` then `!x` | Allow-list style | Exclude all, opt back in |
| `# comment` | Ignored line | Documentation only |

---

## When would I use this at work?

1. **Cutting `orders-api` CI build time** from minutes to seconds by keeping
   `node_modules` and `.git` out of the context, so `docker build` in GitHub
   Actions (Topic 27) is fast and cache-stable.
2. **Preventing secret leaks** — a mandatory `.dockerignore` with `.env*`,
   `*.pem`, `*.key` as a safety net against a careless `COPY . .`.
3. **Stabilizing layer cache** so that editing a README, a test, or a
   `.DS_Store` never triggers a full rebuild of your dependency install layer.

---

## Connected topics

- **Study before:** Topic 03 (Docker Architecture) — the client/daemon split
  and why the context must be sent over the socket. Topic 06 (Dockerfile in
  Depth) — how `COPY`/`ADD` consume the context.
- **Study after:** Topic 08 (Layer Caching) — how a clean context keeps your
  cache stable. Topic 11 (Production Dockerfile for Node.js) — pairs
  `.dockerignore` with multi-stage builds. Topic 15 (Environment Variables and
  Secrets) — the right way to pass secrets at runtime instead of baking them in.
  Topic 23 (Image Security) — minimizing what ships.
