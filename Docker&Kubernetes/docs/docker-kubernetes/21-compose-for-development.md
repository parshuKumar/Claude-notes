# 21 — Compose for Development
## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine you're a chef testing a new recipe. You've built a beautiful **sealed lunchbox** (your production image) with the meal already cooked, portioned, and shrink-wrapped. It's perfect for shipping to customers. But it's *terrible* for cooking, because every time you want to taste-and-tweak a pinch of salt, you'd have to throw away the whole lunchbox, cook a brand-new meal from scratch, seal it again, and only then taste it. Change one thing → 3 minutes of waiting. That's madness for development.

So while you're *developing* the recipe, you don't use the sealed lunchbox. You cook in an **open kitchen** where your recipe notebook is lying on the counter, and the moment you scribble "add basil," the pot on the stove reacts *instantly*. Same stove, same pot, same ingredients — but now the food is connected live to your notebook.

Development Compose is that open kitchen. Your source code lives on your laptop, and it's *wired straight into* the running container through a hole in the wall (a **bind mount**). You edit `orders.js` in your editor, hit save, and the `orders-api` process inside the container restarts and picks up the change in under a second. No rebuild. No re-seal. The container becomes a live window into the code on your disk.

This topic is about building that open kitchen correctly — because there's one classic trap (your laptop's `node_modules` leaking into the container and breaking everything) that everyone falls into once.

---

## The Linux kernel feature underneath

The magic that makes hot reload possible is the **VFS bind mount**, a Linux kernel feature. Remember from **Topic 02** that each container gets its own **mount namespace** (`CLONE_NEWNS`) — its own private view of the filesystem tree. A bind mount is the kernel taking one directory that already exists somewhere and making it *appear* at a second location, so both paths point at the **exact same inodes** on the same disk.

When you write in a Compose file:

```
volumes:
  - ./orders-api:/app
```

Docker asks the kernel — via `runc` and the `mount(2)` syscall with the `MS_BIND` flag — to bind-mount the host directory `./orders-api` onto `/app` inside the container's mount namespace:

```
mount("/home/you/orders-api", "/app", NULL, MS_BIND, NULL)
│      │                       │             │
│      │                       │             └─ MS_BIND: don't create a new filesystem,
│      │                       │                just make source appear at target
│      │                       └─ target: the path INSIDE the container's mount namespace
│      └─ source: the real directory on your laptop's disk
└─ the mount() syscall the kernel exposes
```

After this call, there is **no copy**. The bytes of `orders.js` exist once, on your laptop's SSD. Both `/home/you/orders-api/orders.js` (host) and `/app/orders.js` (container) are two doors into the *same room*. When your editor writes new bytes to the file, the inode's data changes, and the container — reading through the other door — sees the new bytes immediately. The kernel's page cache and inotify machinery do the rest: a file-watcher like **nodemon** inside the container asks the kernel "tell me when anything under `/app` changes" (the `inotify` API), the kernel fires an event the instant the inode is written, and nodemon restarts your Node process.

So "hot reload" is not a Docker feature at all. It's: **one bind mount** (so the container reads your live files) **plus one inotify watcher** (so a process notices the write) **plus one process restart**. Docker Compose just wires those three together for you.

The anonymous-volume `node_modules` trick we'll cover later is *also* just a mount — a second mount layered **on top of** the bind mount to hide one subdirectory. It's mounts all the way down.

---

## What is this?

**Compose for development** is the practice of writing your `docker-compose.yml` (and a companion `docker-compose.override.yml`) so that your source code is bind-mounted into the container, a file-watcher restarts your app on save, and dev-only settings (debug ports, verbose logging, seed data) are switched on — all without ever rebuilding the image.

It's the same Compose you learned in **Topics 18–20**, but tuned for the fast edit-save-see loop instead of for shipping. The central idea: in production the image *contains* your code; in development the image contains only the *environment*, and your code is mounted in live from your laptop.

---

## Why does it matter for a backend developer?

Because without it, your inner development loop is agony. Picture developing `orders-api` the "production way": every time you change a line, you run `docker compose build` (rebuild the image — copies code, runs `npm install`, 30–90 seconds) then `docker compose up` (recreate the container). Change a semicolon → wait a minute → check. Do that 200 times a day and you'll lose an hour to waiting and, worse, lose your train of thought every single time.

With development Compose, the loop is: **save file → 0.8 seconds → nodemon restarts → refresh.** You stay in flow. This is the difference between enjoying Docker and hating it.

Without understanding this you will:
- Rebuild the image for every code change and conclude "Docker is too slow for development" (a false conclusion — you were just holding it wrong).
- Bind-mount your whole project and get the infamous error where your Mac's compiled `bcrypt` or `sharp` binary crashes inside Linux Alpine (`Error: ... invalid ELF header`) because your host `node_modules` leaked in. You'll waste half a day on this exactly once.
- Accidentally ship development settings (bind mounts, debug ports, `NODE_ENV=development`) to production because you never separated dev config from prod config.
- Not know that `docker compose up` *automatically* merges `docker-compose.override.yml`, and be confused why your teammate's machine behaves differently from yours.

---

## The physical reality

Let's make it concrete. You're developing `orders-api` with this project on disk:

```
/home/you/orders-api/
├── docker-compose.yml            ← base config (shared, committed, prod-ish)
├── docker-compose.override.yml   ← dev overrides (committed, auto-merged)
├── Dockerfile
├── package.json
├── package-lock.json
├── node_modules/                 ← installed on your MAC → contains Mac/Darwin binaries
│   └── bcrypt/build/Release/bcrypt.node   ← a Mach-O binary, NOT a Linux ELF
└── src/
    └── orders.js
```

When the dev container is running, here is what physically exists in the kernel's mount table. Run `docker compose exec api mount` (or read `/proc/<PID>/mountinfo` on the host) and you'll see:

```
/dev/sda1 on /app type ext4 (rw)          ← bind mount: /app IS your host ./orders-api
tmpfs     on /app/node_modules type ...    ← anonymous volume HIDING host node_modules
```

Two facts to burn in:

**1. `/app` is not a copy — it's your laptop's directory.** Prove it:

```
$ docker compose exec api sh -c 'echo "// hi from container" >> /app/src/orders.js'
$ tail -1 /home/you/orders-api/src/orders.js
// hi from container            ← the container wrote to YOUR file on YOUR disk
```

**2. The `node_modules` you see inside the container is a *different*, container-only directory.** It is **not** your Mac's `node_modules`. It lives inside Docker's storage, not under `./orders-api`. This is the anonymous-volume trick — a mount stacked on top of the bind mount to shadow one subfolder. Physically it lives at:

```
/var/lib/docker/volumes/<random-64-hex-id>/_data/    ← the container's own node_modules
```

So on disk during development you have: **your source (one copy, shared live via bind mount)** + **a separate Linux-built `node_modules` (living in a Docker volume, invisible to your host)**. That separation is the whole game.

---

## How it works — step by step

Here's the full trace of `docker compose up` starting your dev environment, from the moment you press Enter.

1. **Compose discovers and merges files.** The Docker CLI looks in the current directory. It finds `docker-compose.yml` **and automatically finds `docker-compose.override.yml`**. It deep-merges them: the override file wins on conflicts, and list/map values are combined. (This auto-merge is the single most important dev behavior — you get prod-shaped base + dev tweaks with one command.)

2. **Compose computes the desired state.** After merging, it has a final spec for each service: image or build context, ports, environment, volumes. For `api` the merged spec now includes `build: .`, `volumes: [./:/app, /app/node_modules]`, `command: npm run dev`, and `NODE_ENV=development`.

3. **Compose builds the image if needed.** Because the merged `api` service has `build:`, Compose runs the Dockerfile (**Topic 06**), producing an image that has Node, dependencies installed via `npm ci`, and your code `COPY`ed in. Note: the code copied here will be *shadowed* by the bind mount at runtime — the image's copy of the code is basically a fallback; the live code comes from your disk.

4. **Compose creates the network and volumes.** It creates a bridge network `orders-api_default` (**Topic 13**) so `api`, `postgres`, and `redis` can reach each other by name. It creates the named volumes (Postgres data) and any anonymous volumes (the `node_modules` shadow).

5. **The container is created and `runc` sets up mounts — order matters.** When the container starts, `runc` performs mounts in path order. First it bind-mounts your host `./orders-api` → `/app`. **Then**, because `/app/node_modules` is a *longer path*, it mounts the anonymous volume on top of `/app/node_modules`, hiding whatever your host had there. Result: container sees your live source everywhere *except* `node_modules`, which is its own Linux-built copy from step 3.

6. **The container's command starts the file-watcher.** Instead of `node server.js`, the dev command is `npm run dev` → `nodemon server.js`. Nodemon starts your app *and* registers an `inotify` watch on `/app` with the kernel: "notify me on any write, create, or delete under this tree."

7. **You edit a file.** You save `src/orders.js` in your editor on your Mac. The kernel writes new bytes to that inode. Because `/app` is a bind mount to that same inode, the change is instantly visible inside the container.

8. **inotify fires; nodemon restarts.** The kernel delivers an inotify event to nodemon. Nodemon sends `SIGTERM` to the old Node process, waits for it to exit, then `execve`s a fresh `node server.js`. Your new code is now live. Total time: well under a second. No rebuild, no `compose up` re-run.

9. **Repeat forever.** Steps 7–8 loop for your entire coding session. You only go back to step 3 (rebuild) when you change something the image *bakes in* — like `package.json` dependencies or the Dockerfile itself.

---

## Exact syntax breakdown

### The dev-tuned `api` service in `docker-compose.override.yml`

```yaml
services:
  api:
    build: .
    command: npm run dev
    volumes:
      - ./:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
      LOG_LEVEL: debug
    ports:
      - "3000:3000"
      - "9229:9229"
```

```
services:
│
└─ api:
   │
   ├─ build: .
   │  │      │
   │  │      └─ "." → build context is the current directory; use ./Dockerfile.
   │  │         In dev we build locally instead of pulling a prebuilt image.
   │  └─ tells Compose to run docker build for this service (Topic 06)
   │
   ├─ command: npm run dev
   │  │        │
   │  │        └─ OVERRIDES the image's CMD. Runs nodemon (defined in package.json
   │  │           "dev" script) instead of plain "node server.js" (Topic 07)
   │  └─ the process that becomes PID 1 in the container
   │
   ├─ volumes:
   │  ├─ - ./:/app
   │  │    │  │
   │  │    │  └─ target: mount point INSIDE the container
   │  │    └─ source: "./" = your project dir on the host → BIND MOUNT.
   │  │       This is what makes hot reload possible (live source in container)
   │  │
   │  └─ - /app/node_modules
   │       │
   │       └─ ONE path, no colon, no source → ANONYMOUS VOLUME.
   │          Docker creates a fresh empty volume and mounts it here, ON TOP of
   │          the bind mount, HIDING the host's node_modules. This is THE trick.
   │
   ├─ environment:
   │  ├─ NODE_ENV: development   ← turns on dev behavior in Express/libraries
   │  └─ LOG_LEVEL: debug        ← verbose logs while developing (Topic 15)
   │
   └─ ports:
      ├─ - "3000:3000"   ← host:container — reach the API at localhost:3000
      │      │    │
      │      │    └─ container port the app listens on
      │      └─ host port published on your laptop
      │
      └─ - "9229:9229"   ← Node's debugger port — attach VS Code / Chrome DevTools
```

### The `volumes` line that trips everyone up — long vs short mount

```
- ./:/app                - /app/node_modules
  │  │                     │
  │  └─ has a COLON        └─ NO colon → anonymous volume (Docker-managed, empty)
  │     → host:container
  └─ source is a host path → bind mount (your live files)
```

Rule the kernel follows: **the longer (more specific) mount path wins for that subtree.** `/app/node_modules` is longer than `/app`, so it's mounted *after* and shadows the bind mount only under `node_modules`. Everything else under `/app` still shows your live host files.

### Choosing which Compose files to merge, explicitly

```
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
│              │  │                   │  │                        │
│              │  │                   │  │                        └─ up: create + start
│              │  │                   │  └─ second file, MERGED ON TOP of the first
│              │  │                   └─ another -f to add a file (order = merge order)
│              │  └─ the base file
│              └─ -f: specify a compose file explicitly (disables auto-merge of override)
└─ the compose subcommand
```

Key detail: the **moment you pass any `-f`, Compose stops auto-loading `docker-compose.override.yml`.** So `-f docker-compose.yml -f docker-compose.prod.yml` gives you base + prod, with *no* dev override. That's exactly how you run the "prod-shaped" config locally.

---

## Example 1 — basic

Let's build the minimal hot-reload setup for `orders-api` and watch it reload.

**`package.json`** (the `dev` script is what makes reload work):

```json
{
  "name": "orders-api",
  "scripts": {
    "start": "node server.js",       // production: run once, no watcher
    "dev": "nodemon server.js"        // development: watch files, restart on change
  },
  "dependencies": { "express": "^4.19.0" },
  "devDependencies": { "nodemon": "^3.1.0" }
}
```

**`server.js`:**

```js
const express = require('express');
const app = express();

// Change this string, save, and watch the container restart automatically.
app.get('/health', (req, res) => res.json({ status: 'ok', version: 'v1' }));

app.listen(3000, () => console.log('orders-api listening on :3000'));
```

**`docker-compose.yml`** (base — kept simple here):

```yaml
services:
  api:
    build: .                 # build from local Dockerfile
    ports:
      - "3000:3000"          # publish so we can curl localhost:3000
    volumes:
      - ./:/app              # BIND MOUNT: live source into the container
      - /app/node_modules    # ANONYMOUS VOLUME: keep container's own node_modules
    command: npm run dev     # run nodemon, not plain node
```

Run it and prove hot reload:

```bash
# 1. Start it (builds the image the first time, then runs nodemon).
docker compose up

# 2. In another terminal, check the app.
curl localhost:3000/health
# → {"status":"ok","version":"v1"}

# 3. Edit server.js: change 'v1' to 'v2' and SAVE. Do NOT rebuild.
#    Watch the first terminal — nodemon prints:
#    [nodemon] restarting due to changes...
#    orders-api listening on :3000

# 4. Curl again — new code, no rebuild.
curl localhost:3000/health
# → {"status":"ok","version":"v2"}     ← changed in <1 second, zero rebuild
```

You changed running code without rebuilding the image, because the container was reading `server.js` straight off your disk through the bind mount, and nodemon noticed the write via inotify.

---

## Example 2 — production scenario

**The situation.** Your team of five is building `orders-api` (Node 20 + Express), backed by Postgres and Redis. New hires clone the repo and it "doesn't work" on their machines in three different ways: one has the wrong Node version installed locally, one gets a `bcrypt` ELF-header crash, and one accidentally deploys their laptop's debug settings to staging. You're asked to make development "just work" with `docker compose up` while keeping production safe.

The fix is a **two-file split**: a base file that is prod-shaped, and an override that adds dev conveniences and is auto-merged only during development.

**`docker-compose.yml`** — the base. Safe, minimal, no dev leakage. This is close to what prod uses.

```yaml
services:
  api:
    image: registry.example.com/orders-api:${TAG:-latest}  # prod pulls a built image
    environment:
      NODE_ENV: production        # safe default
      DATABASE_URL: postgres://orders:secret@postgres:5432/orders
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres: { condition: service_healthy }   # startup order (Topic 20)
      redis:    { condition: service_started }
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: orders
    volumes:
      - pgdata:/var/lib/postgresql/data          # named volume: real persistence (Topic 14)
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

**`docker-compose.override.yml`** — auto-merged by `docker compose up`. Adds *only* dev conveniences on top of the base.

```yaml
services:
  api:
    build: .                    # dev BUILDS locally instead of using the registry image
    command: npm run dev        # nodemon instead of node
    environment:
      NODE_ENV: development     # OVERRIDES the base's "production"
      LOG_LEVEL: debug
    volumes:
      - ./:/app                 # live source for hot reload
      - /app/node_modules       # the node_modules shadow trick
    ports:
      - "3000:3000"             # reach the API
      - "9229:9229"             # attach a debugger

  postgres:
    ports:
      - "5432:5432"             # expose Postgres to your host so you can use a GUI (TablePlus)

  redis:
    ports:
      - "6379:6379"             # expose Redis so you can inspect keys locally
```

Now the two workflows:

```bash
# DEVELOPER on their laptop — one command, auto-merges base + override:
docker compose up
#   → api is BUILT locally, runs nodemon, hot reload on, debugger + DB ports exposed,
#     NODE_ENV=development. New hire types one command and everything works.

# CI / STAGING deploy — explicitly pick base + a prod file, which DISABLES the override:
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
#   → api PULLS the registry image, runs "node server.js", no bind mounts,
#     NODE_ENV=production, no debug ports. Your laptop's dev config can't leak in.
```

Now walk back through the three broken new-hires:
- **Wrong local Node version** — irrelevant now. Node 20 is baked into the image; their host Node is never used.
- **`bcrypt` ELF crash** — gone. The anonymous `node_modules` volume means the container uses *Linux-built* native modules from the image, never the Mac binaries sitting in the host `node_modules`.
- **Leaked debug settings** — impossible. Dev settings live only in `docker-compose.override.yml`, which the deploy command *doesn't load* because it passes explicit `-f` flags.

One `docker compose up` for devs; an explicit two-`-f` command for deploys; the two never contaminate each other. That's the whole pattern.

---

## Common mistakes

**Mistake 1 — Bind-mounting the project WITHOUT the `node_modules` anonymous volume.**

You write only `- ./:/app` and skip `- /app/node_modules`. Now your host `node_modules` (built on macOS) is visible inside the Linux container. Native modules explode:

```
$ docker compose up
api  | Error: /app/node_modules/bcrypt/lib/binding/napi-v3/bcrypt_lib.node:
api  |   invalid ELF header
api  | at Object.Module._extensions..node (node:internal/modules/cjs/loader)
```

Root cause: `bcrypt` (or `sharp`, `argon2`, `esbuild` — anything with a compiled native `.node` binary) was built for **Mach-O / Darwin** on your Mac. The Linux kernel in the container tries to `dlopen` it and rejects it — it's not a Linux **ELF** binary. The bind mount dragged the wrong-architecture binaries in.

Wrong:
```yaml
volumes:
  - ./:/app
```
Right:
```yaml
volumes:
  - ./:/app
  - /app/node_modules      # shadow host node_modules with the container's Linux-built copy
```
The anonymous volume mounts *over* `/app/node_modules`, hiding the Mac binaries so the container uses the Linux ones installed during `docker build`.

**Mistake 2 — Editing `package.json` and expecting the new dependency to appear.**

You add `axios` to `package.json`, save, and `require('axios')` throws:

```
api  | Error: Cannot find module 'axios'
```

Root cause: dependencies are installed **at image build time** (`npm ci` in the Dockerfile) into the container's `node_modules` volume. Hot reload only reloads *code*, not *dependencies* — the `node_modules` volume isn't touched by your edit. Right fix: rebuild so `npm ci` runs again:

```bash
docker compose up --build          # rebuild image, then start
# or, one-off, install into the running container's volume:
docker compose exec api npm install axios
```

**Mistake 3 — Assuming `-f` still auto-loads the override.**

You run `docker compose -f docker-compose.yml up` locally and hot reload is gone; your app runs `node server.js` and ignores your bind mounts.

Root cause: **passing any `-f` disables the automatic loading of `docker-compose.override.yml`.** You told Compose the exact file list, so it obeys you literally and never adds the override. Wrong (for dev):
```bash
docker compose -f docker-compose.yml up        # override IGNORED
```
Right (for dev), just let auto-merge happen:
```bash
docker compose up                              # base + override auto-merged
```
Or be explicit if you want dev:
```bash
docker compose -f docker-compose.yml -f docker-compose.override.yml up
```

**Mistake 4 — File changes not detected on Mac/Windows (no reload).**

You save a file and nodemon never restarts. This is common on Docker Desktop (Mac/Windows) because inotify events don't always cross the host↔VM filesystem boundary.

Root cause: on macOS/Windows, Docker runs a **Linux VM**, and the bind mount goes host FS → VM → container. The kernel `inotify` events from your Mac's filesystem don't always propagate through that boundary, so nodemon's watcher never fires. Right fix — tell the watcher to **poll** instead of relying on inotify:

```yaml
environment:
  CHOKIDAR_USEPOLLING: "true"    # nodemon/chokidar polls the filesystem every interval
```
or run `nodemon --legacy-watch server.js`. Polling is slightly heavier but reliably detects changes across the VM boundary.

**Mistake 5 — Shipping the override file's settings to production.**

Staging comes up with `NODE_ENV=development`, a bind mount to a directory that doesn't exist on the server, and an exposed debugger port. Attackers can attach to `9229`.

```
Error response from daemon: invalid mount config: bind source path does not exist: /home/ci/orders-api
```

Root cause: your deploy ran plain `docker compose up` on the server, which auto-merged `docker-compose.override.yml` (a dev-only file). Right fix: deploys must **explicitly** list prod files so the override is never loaded:
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
Rule: the override is for *your laptop only*. Never let a server run bare `docker compose up`.

---

## Hands-on proof

Run these **right now** to see the bind mount, the `node_modules` shadow, and the override merge with your own eyes.

```bash
# --- Setup: a tiny orders-api dev project ---
mkdir -p /tmp/orders-dev && cd /tmp/orders-dev

cat > package.json <<'EOF'
{ "name":"orders-api", "scripts":{ "dev":"node -e \"require('http').createServer((_,r)=>r.end(process.env.MSG||'v1')).listen(3000,()=>console.log('up'))\" && sleep infinity" } }
EOF

cat > Dockerfile <<'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev || true
COPY . .
CMD ["npm","run","dev"]
EOF

cat > docker-compose.yml <<'EOF'
services:
  api:
    build: .
    ports: ["3000:3000"]
    volumes:
      - ./:/app
      - /app/node_modules
EOF

cat > docker-compose.override.yml <<'EOF'
services:
  api:
    environment:
      MSG: "v2-from-override"
EOF

# 1. Show the FINAL merged config. Proof: override's MSG appears merged into base.
docker compose config
#    Look for  MSG: v2-from-override  under services.api.environment

# 2. Start it.
docker compose up -d --build

# 3. Prove /app is a live bind mount: write from the container, read on the host.
docker compose exec api sh -c 'echo BIND_MOUNT_WORKS > /app/proof.txt'
cat /tmp/orders-dev/proof.txt          # → BIND_MOUNT_WORKS  (container wrote to YOUR disk)

# 4. Prove node_modules is a SEPARATE volume, not your host dir.
#    Create a marker in the host node_modules, then look inside the container.
mkdir -p node_modules && echo HOST > node_modules/marker.txt
docker compose exec api ls /app/node_modules/marker.txt 2>&1
#    → No such file or directory  (the anonymous volume HIDES your host node_modules)

# 5. Prove the mount table shows two stacked mounts on /app and /app/node_modules.
docker compose exec api mount | grep /app

# 6. Prove override is auto-loaded vs disabled by -f:
docker compose config --services            # uses override (MSG=v2)
docker compose -f docker-compose.yml config  # base only — no MSG override

# 7. Clean up.
docker compose down -v
rm -rf /tmp/orders-dev
```

What you verified: `docker compose config` shows the *merged* result (step 1); `/app` is your live disk (step 3); `node_modules` inside the container is a separate Docker volume that shadows your host's (step 4–5); and `-f` disables the auto-override (step 6).

---

## Practice exercises

### Exercise 1 — easy
Create a two-file setup for `orders-api`: a `docker-compose.yml` with the `api` service using `image: node:20-alpine` and a `docker-compose.override.yml` that adds `command: node -v` and `environment: NODE_ENV: development`. Run `docker compose config` and identify: (a) which file contributed `NODE_ENV`, (b) what the final merged `command` is, and (c) what happens to `NODE_ENV` if you run `docker compose -f docker-compose.yml config` instead.

### Exercise 2 — medium
Build a real hot-reload loop. Write `server.js` (Express, one `/ping` route returning `"pong-1"`), a `package.json` with `"dev": "nodemon server.js"`, a `Dockerfile` that installs deps and runs `npm run dev`, and a Compose file with both the `./:/app` bind mount and the `/app/node_modules` anonymous volume. `docker compose up`, curl `/ping`, then change `"pong-1"` to `"pong-2"` and save. Without rebuilding, curl again and confirm it changed. Then run `docker compose exec api mount | grep /app` and write down why there are two mounts and which one wins for `node_modules`.

### Exercise 3 — hard (production simulation)
Reproduce the `bcrypt` ELF crash and fix it. On a Mac (or by first running `npm install bcrypt` on the host so host `node_modules` contains a native binary), write a Compose file that bind-mounts `./:/app` **without** the `/app/node_modules` anonymous volume, and a `server.js` that does `require('bcrypt')`. Run `docker compose up` and capture the `invalid ELF header` error. Now add the `- /app/node_modules` line, rebuild with `docker compose up --build`, and confirm the app starts. Explain in two sentences: which architecture the failing binary was built for, and exactly how the second mount fixed it.

---

## Mental model checkpoint

Answer these from memory:

1. What single Linux kernel feature makes hot reload possible, and what syscall flag creates it?
2. Why does `- ./:/app` alone break native modules on a Mac, and what does adding `- /app/node_modules` do about it?
3. When two mounts overlap (`/app` and `/app/node_modules`), which one wins for the `node_modules` subtree, and why?
4. What are the *two* things nodemon needs to reload your app, and which one does the kernel provide?
5. Does `docker compose up` load `docker-compose.override.yml`? What happens to that behavior the moment you pass `-f`?
6. You added a dependency to `package.json` and hot reload didn't pick it up. Why not, and what command fixes it?
7. How do you run the "production-shaped" config locally without the dev override merging in?

---

## Quick reference card

| Command / field | What it does | Key detail |
|---|---|---|
| `- ./:/app` | Bind-mount host project into container | Live source → hot reload; has a colon → bind mount |
| `- /app/node_modules` | Anonymous volume shadowing host `node_modules` | No colon → anonymous; longer path wins, hides host modules |
| `command: npm run dev` | Override image CMD to run nodemon | Reloads on file change via inotify (Topic 07) |
| `docker-compose.override.yml` | Dev overrides, auto-merged by `up` | Loaded automatically **only** when no `-f` is passed |
| `docker compose up` | Merge base + override, build/create/start | The everyday dev command |
| `docker compose -f a.yml -f b.yml up` | Merge exactly these files (later wins) | Passing `-f` **disables** auto-loading the override |
| `docker compose config` | Print the final merged config | Best way to debug "which value won" |
| `docker compose up --build` | Rebuild image then start | Needed after `package.json`/Dockerfile changes |
| `CHOKIDAR_USEPOLLING=true` | Make watcher poll instead of using inotify | Fixes "no reload" on Docker Desktop Mac/Windows |
| `9229:9229` port | Expose Node's inspector | Attach VS Code / Chrome DevTools debugger |

---

## When would I use this at work?

1. **Onboarding a new backend hire in five minutes.** Instead of a 3-page "install Node 20, Postgres, Redis, run these migrations" wiki, they `git clone` and `docker compose up`. The override file gives them hot reload, seeded DB, and exposed ports automatically — and their host's Node/Postgres versions are irrelevant.

2. **Debugging `orders-api` with breakpoints against real Postgres/Redis.** You expose `9229` in the override, run `nodemon --inspect=0.0.0.0:9229`, and attach VS Code. You step through the actual code while it talks to the containerized Postgres and Redis — production-like data flow, laptop-like debugging.

3. **Keeping dev and prod honest with one base file.** The `docker-compose.yml` base is the single source of truth for how services connect (env vars, DB URLs, healthchecks). Dev layers conveniences via the override; deploys layer `docker-compose.prod.yml`. Nobody maintains two divergent copies, and dev settings physically cannot leak to prod because deploys pass explicit `-f` flags.

---

## Connected topics

- **Study before:**
  - **Topic 14 — Volumes in Depth**: bind mounts vs named vs anonymous volumes — the mount mechanics this topic builds on.
  - **Topic 18 — Docker Compose in Depth**: the base Compose file structure.
  - **Topic 19 — Compose File Deep Dive**: every field (`build`, `volumes`, `environment`, `command`) used here.
  - **Topic 20 — Multi-Container Apps**: how `api`, `postgres`, `redis` find each other by name and order startup with `depends_on` + healthchecks.
- **Study after:**
  - **Topic 22 — Compose Commands**: the full command surface (`up`, `down`, `build`, `logs`, `exec`, `config`, profiles) you drive this dev loop with.
  - **Topic 11 — Production Dockerfile for Node.js**: how the *prod* image (no bind mounts, code baked in) is built.
  - **Topic 15 — Environment Variables and Secrets**: the `environment`/`--env-file` mechanics behind the dev/prod config split.
