# 15 — Environment Variables and Secrets

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine you build one toy robot in a factory. You want to sell the **exact same robot** to houses all over the world. But every house has a different Wi-Fi password.

You have two choices:

1. **Bake the password inside the robot at the factory.** Now every robot only works in one house, and worse — anyone who cracks the robot open can read the password printed on a chip inside. If you ship 10,000 robots, you've printed the password 10,000 times.

2. **Leave a little slot on the robot's back where the owner slides in a card with their Wi-Fi password when they turn it on.** Now the *same* robot works everywhere. The password lives on the card, not inside the robot. Throw the card away and the robot forgets it.

A **Docker image** is the robot made in the factory. **Environment variables** are the card you slide in when the container turns on. The image stays generic and reusable; the secret stays outside, injected at the last second.

Baking the secret into the image is choice #1 — and it's worse than it sounds, because a Docker image never truly forgets. Every change is saved as a permanent layer, like a robot that keeps a photo album of every chip that was ever installed. Delete the password chip later? The photo of it is still glued inside. That's the layer-history leak, and we'll dig into exactly where it lives on disk.

---

## The Linux kernel feature underneath

An environment variable is not a Docker invention. It is a **Unix process concept that predates Docker by decades.**

Every process on Linux has, in its own memory, a block called the **environment block** — a simple array of `KEY=VALUE` strings. When the kernel runs a new program via the `execve(2)` syscall, the signature is:

```
int execve(const char *pathname, char *const argv[], char *const envp[]);
                                          │                    │
                                          │                    └─ envp: the environment array,
                                          │                       e.g. ["PATH=/usr/bin", "PORT=3000", NULL]
                                          └─ argv: the command-line arguments, e.g. ["node", "server.js", NULL]
```

So when Docker starts your container's main process, it is literally calling `execve` with an `envp` array it built from your `-e` flags, your `--env-file`, and any `ENV` lines baked into the image. The kernel copies that array into the new process's memory.

You can see the raw environment block of **any** running process on Linux — it lives in the `/proc` virtual filesystem:

```
/proc/[PID]/environ
     │       │
     │       └─ a single "file" containing KEY=VALUE pairs separated by NUL (\0) bytes
     └─ the process ID
```

This is the single most important fact in this whole topic:

> **Environment variables are readable by anyone who can read `/proc/[PID]/environ` for that process.** They are NOT encrypted. They are NOT hidden. They sit in plain text in process memory and in that `/proc` file.

Two consequences that shape every decision below:

1. Inside the container, `/proc/1/environ` (PID 1 is your app) holds every env var you injected — including `DB_PASSWORD`. Any process in that container, and root on the host, can read it.
2. Child processes **inherit** the environment. When your Node app spawns a shell or a subprocess, that child gets a copy of the whole environment block automatically. This is why a leaked env var spreads.

Docker's job is just to *populate* that `envp` array correctly at `execve` time. Everything in this doc is about controlling **what goes into the array** and **whether the value also gets frozen into the image on disk**.

---

## What is this?

Environment variables are `KEY=VALUE` pairs injected into your container's main process at start time, used to configure behavior without changing the image. **Secrets** are a special, sensitive subset (passwords, API keys, tokens) that need extra care because plain env vars are visible in process memory, `docker inspect`, and — if baked into an image — permanently stored in layer history.

This topic covers how to inject config (`-e`, `--env-file`, `ENV`), the 12-factor principle of keeping config in the environment, *why* baking secrets into images leaks them, and the safer mechanisms (BuildKit build secrets and Docker Swarm secrets) that keep sensitive values off disk.

---

## Why does it matter for a backend developer?

Your `orders-api` needs a database URL, a Redis URL, a JWT signing key, and a Stripe API key. These four values change between your laptop, staging, and production. The Postgres password in production must **never** be the one on your laptop.

If you don't understand this topic, three concrete things break:

- **You bake `DATABASE_URL` into the Dockerfile with `ENV`.** Now your image only works in one environment, and the production password is sitting in the image on Docker Hub for anyone who runs `docker history`.
- **You paste the Stripe secret key into a `RUN` command** to test something during build. You delete the line the next day. But the secret is **permanently in a layer**, and that layer got pushed to your registry. Rotating the key is now the only fix — and you won't even know you need to.
- **You can't promote the same image from staging to production**, so you rebuild for each environment, meaning "the thing you tested" is not "the thing you shipped."

Understanding config-via-environment is what lets you **build one image and run it everywhere**, which is the entire foundation of reliable deploys and, later, Kubernetes ConfigMaps and Secrets (Topic 36).

---

## The physical reality

Let's look at where config actually lives when a container runs.

**1. Baked-in image config (`ENV` lines) — frozen on disk in image metadata:**

```
/var/lib/docker/
├── image/overlay2/imagedb/content/sha256/
│   └── <image-config-sha>          ← JSON blob. Contains "Env": ["PORT=3000", ...]
│                                       This is READ-ONLY and shipped with the image.
```

Run `docker inspect orders-api:1.0 --format '{{json .Config.Env}}'` and you see exactly this array. Anyone who pulls your image sees it too.

**2. Runtime config (`-e`, `--env-file`) — stored in the container's config, on the host:**

```
/var/lib/docker/containers/<container-id>/
├── config.v2.json                  ← contains the merged env array for THIS container instance
└── hostconfig.json
```

**3. The live process — in kernel-managed memory, exposed via /proc:**

```
Inside the container:
/proc/1/environ                     ← PID 1's environment block, NUL-separated KEY=VALUE pairs
```

Here is the crucial split, drawn out:

```
              ENV in Dockerfile                 -e / --env-file at run
                     │                                   │
                     ▼                                   ▼
        ┌────────────────────────┐          ┌────────────────────────┐
        │  Image config JSON      │          │  Container config JSON  │
        │  (in /var/lib/docker/   │          │  (per-container, host)  │
        │   image/...)            │          │                         │
        │                         │          │                         │
        │  SHIPPED with the image │          │  NEVER in the image      │
        │  → visible to everyone  │          │  → not in registry       │
        │    who pulls it         │          │  → gone when container   │
        │  → PERMANENT in layers  │          │    is removed            │
        └────────────────────────┘          └────────────────────────┘
                     │                                   │
                     └───────────────┬───────────────────┘
                                     ▼
                          Merged into one envp[] array
                                     │
                                     ▼
                        execve("node", argv, envp)
                                     │
                                     ▼
                       /proc/1/environ (plain text, in memory)
```

The lesson from the picture: `ENV` writes to the **left box** (permanent, shipped). `-e`/`--env-file` writes to the **right box** (per-run, ephemeral). Secrets must only ever touch the right box — or, better, not touch the environment at all (BuildKit secrets, below).

---

## How it works — step by step

Let's trace exactly what happens when you run:

```
docker run -e PORT=3000 --env-file ./prod.env myimage node server.js
```

1. **CLI parses flags.** The Docker CLI reads `-e PORT=3000` and reads every line of `./prod.env`, turning them into `KEY=VALUE` strings. `./prod.env` is read **on the host, by the CLI, right now** — the file is never sent into the container.
2. **CLI sends a create request to the daemon.** Over the Docker socket (`/var/run/docker.sock`, see Topic 03), it sends a JSON container-create request whose `Env` field is the combined list.
3. **Daemon merges environments in a defined order.** The daemon takes the image's baked-in `Env` (from the image config JSON) as the **base**, then overlays the request's env on top. **Runtime `-e`/`--env-file` values win over image `ENV` values for the same key.** The merged result is written to `/var/lib/docker/containers/<id>/config.v2.json`.
4. **Daemon asks containerd → runc to create the container** (Topic 03). It hands runc an OCI runtime spec (`config.json`) whose `process.env` array is that merged list.
5. **runc sets up namespaces and cgroups, then calls `execve`.** It calls `execve("/usr/local/bin/node", ["node","server.js"], envp)` where `envp` is the merged array. The kernel loads the array into the new PID 1's memory.
6. **Node reads the environment.** Node's V8 startup copies the environment block into the `process.env` object. Now `process.env.PORT === "3000"` in your JavaScript.
7. **`/proc/1/environ` is populated.** From this moment, the environment is visible in that file and inherited by every child process Node spawns.

Notice step 5: env vars are **read once, at exec time**. If you change `prod.env` later, a **running** container does NOT see it. Env vars are set at process birth. To change them you must recreate the container. (Contrast with a mounted config file, which you *can* change live — a real reason teams sometimes mount secrets as files instead.)

---

## Exact syntax breakdown

### The `-e` / `--env` flag

```
docker run -e DB_HOST=postgres -e PORT=3000 orders-api:1.0
           │  │       │
           │  │       └─ the value ("postgres")
           │  └─ the key ("DB_HOST"); everything after the first "=" is the value
           └─ -e (short for --env): inject ONE variable. Repeat the flag for more.
```

**Pass-through form (no `=`)** — take the value from the host's current shell:

```
docker run -e DB_PASSWORD orders-api:1.0
           │  │
           │  └─ no "=" sign. Docker reads $DB_PASSWORD from YOUR shell and forwards its value.
           └─ handy for CI: the secret lives in the CI env, never typed on the command line.
```

> Caution: any `-e KEY=VALUE` you type is saved in your shell history file (`~/.zsh_history`) and appears in `ps` and `docker inspect`. Prefer the pass-through form or `--env-file` for anything sensitive.

### The `--env-file` flag

```
docker run --env-file ./prod.env orders-api:1.0
           │          │
           │          └─ path to a file on the HOST, read by the CLI at run time
           └─ load many KEY=VALUE pairs from a file, one per line
```

The file format:

```
# prod.env — comments start with # and are ignored
DB_HOST=postgres
DB_PORT=5432
DB_USER=orders
DB_PASSWORD=s3cr3t-from-vault
REDIS_URL=redis://redis:6379
```

Rules that trip people up:

```
DB_PASSWORD=s3cr3t          ← value is the RAW rest of the line
DB_PASSWORD="s3cr3t"        ← the QUOTES become part of the value! → password is literally "s3cr3t"
KEY = value                 ← INVALID: no spaces around "=" allowed by --env-file
EMPTY=                      ← sets EMPTY to "" (empty string)
FROM_SHELL                  ← no "=": pulls FROM_SHELL from the CLI's environment (pass-through)
```

### The `ENV` Dockerfile instruction

```
ENV NODE_ENV=production PORT=3000
    │       │           │
    │       │           └─ a second KEY=VALUE on the same line (space-separated)
    │       └─ the value
    └─ writes into the IMAGE config → permanent, shipped, visible via docker history
```

Use `ENV` only for **non-secret defaults** that are genuinely part of the image's identity: `NODE_ENV=production`, `PORT=3000`, `PATH` additions. **Never** a password, token, or key.

### The `ARG` Dockerfile instruction (build-time only — but still not for secrets)

```
ARG NODE_VERSION=20
    │           │
    │           └─ default value if not overridden with --build-arg
    └─ a build-time variable: available during `docker build`, NOT in the running container
```

```
docker build --build-arg NODE_VERSION=20 -t orders-api:1.0 .
             │           │
             │           └─ overrides the ARG default for this build
             └─ pass a build-time value
```

`ARG` feels safer than `ENV` because it's not in the final container's environment — but its **value is stored in image history** (`docker history`). So `--build-arg STRIPE_KEY=...` still leaks. Not for secrets.

### BuildKit secret mount (the correct way to use a secret DURING build)

```
# syntax=docker/dockerfile:1
FROM node:20-alpine
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    │            │        │       │
    │            │        │       └─ where the secret file appears DURING this RUN only
    │            │        └─ the secret's name, matched to --secret id=npmrc on the CLI
    │            └─ this is a secret mount, not a bind or cache mount
    └─ mounts a secret into ONE RUN step; it is NEVER written to any layer
    npm ci
```

```
docker build --secret id=npmrc,src=$HOME/.npmrc -t orders-api:1.0 .
             │        │         │
             │        │         └─ the file on the host to expose as this secret
             │        └─ the id the Dockerfile references
             └─ hands the secret to BuildKit; it lives only in a tmpfs during the RUN, never in a layer
```

### Docker Swarm / Compose secret (the correct way to use a secret AT RUNTIME)

```yaml
services:
  orders-api:
    image: orders-api:1.0
    secrets:
      - db_password          # request this secret for this service
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password   # tell the app WHERE to read it
      │                    │
      │                    └─ secrets are mounted as FILES under /run/secrets/, not as env vars
      └─ note the "_FILE" convention: pass the PATH, app reads the file itself

secrets:
  db_password:
    file: ./db_password.txt   # source on the host (or "external: true" for Swarm-managed)
    │
    └─ where Docker reads the secret's value from
```

---

## Example 1 — basic

Configure `orders-api` with env vars, then prove the app reads them.

**`server.js`:**

```javascript
// server.js — a tiny orders-api that is configured ENTIRELY from the environment
const express = require('express');
const app = express();

// Read config from the environment. Provide sane defaults for local dev,
// but NEVER a default for a secret — missing secret should crash loudly.
const PORT = process.env.PORT || 3000;              // non-secret, safe default
const DB_HOST = process.env.DB_HOST || 'localhost'; // non-secret, safe default
const DB_PASSWORD = process.env.DB_PASSWORD;        // secret: NO default

if (!DB_PASSWORD) {
  // Fail fast: a container missing its secret should not start "half working".
  console.error('FATAL: DB_PASSWORD is not set');
  process.exit(1);
}

app.get('/config', (req, res) => {
  res.json({
    port: PORT,
    dbHost: DB_HOST,
    dbPasswordLength: DB_PASSWORD.length, // show we HAVE it, never log the value itself
  });
});

app.listen(PORT, () => console.log(`orders-api listening on ${PORT}`));
```

**`Dockerfile`:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
ENV NODE_ENV=production PORT=3000   # non-secret defaults ONLY — baked into the image is fine here
EXPOSE 3000
CMD ["node", "server.js"]
```

**Run it with runtime config:**

```
docker build -t orders-api:1.0 .

# -e passes the secret at run time; it never touches the image
docker run -d --name orders \
  -e DB_HOST=postgres \
  -e DB_PASSWORD=local-dev-password \
  -p 3000:3000 \
  orders-api:1.0

curl localhost:3000/config
# {"port":"3000","dbHost":"postgres","dbPasswordLength":19}
```

Now prove the two-box model from the physical-reality section:

```
# The image only shows the NON-secret ENV defaults:
docker inspect orders-api:1.0 --format '{{json .Config.Env}}'
# ["PATH=...","NODE_ENV=production","PORT=3000"]   ← no DB_PASSWORD here. Good.

# The running CONTAINER shows the merged env, including the secret:
docker inspect orders --format '{{json .Config.Env}}'
# [...,"DB_HOST=postgres","DB_PASSWORD=local-dev-password"]   ← secret is here (per-run only)
```

The secret is in the *container* config (ephemeral, host-only), never in the *image* config (shipped). That's the whole game.

---

## Example 2 — production scenario

**The situation:** Your team's `orders-api` needs a private npm package during build (from a private registry that needs an auth token in `.npmrc`), and needs the Postgres password at runtime. A junior engineer "made it work" like this:

```dockerfile
# ❌ THE DANGEROUS VERSION — do not do this
FROM node:20-alpine
WORKDIR /app
# baked the npm token into a RUN command to install private deps
RUN echo "//registry.npmjs.org/:_authToken=npm_AbCdEf123456" > /root/.npmrc && \
    npm ci && \
    rm /root/.npmrc                     # "I deleted it, so it's gone" — WRONG
COPY . .
ENV DB_PASSWORD=prod-postgres-9f3k2   # baked the DB password into the image — WRONG
CMD ["node", "server.js"]
```

Two secrets are now permanently leaked. Prove it:

```
docker history --no-trunc orders-api:1.0
# ...
# RUN echo "//registry.npmjs.org/:_authToken=npm_AbCdEf123456" > /root/.npmrc && npm ci && rm /root/.npmrc
# ENV DB_PASSWORD=prod-postgres-9f3k2
```

The `rm /root/.npmrc` runs in a **later** layer, but the **earlier** layer where the file was created is a separate, immutable filesystem diff sitting in `/var/lib/docker/overlay2/`. You can extract it:

```
docker save orders-api:1.0 -o img.tar
mkdir img && tar -xf img.tar -C img
# each layer is a tar of file DIFFS. The npmrc-creation layer still contains the token,
# even though a later layer "deleted" the file with a whiteout marker.
grep -r "authToken" img/    # → finds npm_AbCdEf123456
```

**Why deletion doesn't help:** overlay2 (Topic 02/04) deletes files by writing a **whiteout** entry in an upper layer that *hides* the file. The bytes in the lower layer are untouched. A layer is a permanent diff; you cannot edit history without rebuilding.

**The fix — the correct version:**

```dockerfile
# ✅ THE SAFE VERSION
# syntax=docker/dockerfile:1
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
# BuildKit secret: .npmrc exists only in a tmpfs during THIS RUN, never in any layer
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --omit=dev
COPY . .
ENV NODE_ENV=production PORT=3000   # non-secret defaults only
CMD ["node", "server.js"]
```

```
# build: hand the secret to BuildKit; it will not be stored
DOCKER_BUILDKIT=1 docker build \
  --secret id=npmrc,src=$HOME/.npmrc \
  -t orders-api:1.0 .

docker history --no-trunc orders-api:1.0 | grep -i token   # → nothing. Clean.
```

And the DB password goes in **at runtime**, from your secret store, never baked:

```
docker run -d --name orders \
  --env-file /etc/orders/prod.env \    # file readable only by root, holds DB_PASSWORD
  orders-api:1.0
```

Now the same image runs on staging and prod — only the `--env-file` differs. And no secret is on disk in the image or in the registry.

---

## Common mistakes

### Mistake 1 — Baking a secret with `ENV` or `ARG`

**What you did:**

```dockerfile
ENV STRIPE_KEY=sk_live_51H8xY...
```

**How you find out (the exact command):**

```
docker history orders-api:1.0
# IMAGE   CREATED   CREATED BY
# ...     ...       /bin/sh -c #(nop)  ENV STRIPE_KEY=sk_live_51H8xY...
```

**Root cause:** `ENV` writes into the image config JSON in `/var/lib/docker/image/...`, which is shipped with the image and shown by `docker history` and `docker inspect`. `ARG` values likewise land in history. **Right:** inject at runtime with `-e`/`--env-file`, or use a BuildKit secret if it's needed only during build.

### Mistake 2 — Thinking `rm secretfile` in a later RUN removes it

**What you did:** created a secret file, used it, then `rm`-ed it in the same or a later `RUN`.

**How you find out:**

```
docker save orders-api:1.0 -o img.tar && tar -xf img.tar && grep -r "sk_live" .
# → the key is still there, in the layer where the file was created
```

**Root cause:** overlay2 deletion is a whiteout in an upper layer; the lower layer's bytes are immutable. **Right:** never let a secret hit a layer at all — use `--mount=type=secret`.

### Mistake 3 — Quotes in `--env-file`

**What you did:**

```
# prod.env
DB_PASSWORD="my-password"
```

**The error you see** — your app fails to authenticate to Postgres:

```
error: password authentication failed for user "orders"
```

**Root cause:** `--env-file` does NOT strip quotes. The value literally became `"my-password"` (with quote characters). Postgres sends the wrong password. **Right:** write it bare: `DB_PASSWORD=my-password`.

### Mistake 4 — Expecting variable substitution inside `--env-file`

**What you did:**

```
# prod.env
DB_HOST=postgres
DATABASE_URL=postgres://orders@${DB_HOST}:5432/orders
```

**Result:** `DATABASE_URL` is the literal string with `${DB_HOST}` unexpanded — no substitution happens in `--env-file`.

```
docker run --env-file prod.env orders-api:1.0 sh -c 'echo $DATABASE_URL'
# postgres://orders@${DB_HOST}:5432/orders   ← not expanded
```

**Root cause:** `--env-file` is dumb key/value parsing, no shell, no interpolation. **Right:** build the full URL in the file, or interpolate in your shell before running, or use Compose (which *does* interpolate `${...}` from its own environment).

### Mistake 5 — Logging the environment

**What you did:**

```javascript
console.log('Starting with config:', process.env);  // ❌ dumps DB_PASSWORD, JWT_SECRET...
```

**How it bites you:** the secret is now in `docker logs`, in your `json-file` log on disk (`/var/lib/docker/containers/<id>/<id>-json.log`, Topic 17), and in your log aggregator (Datadog/CloudWatch) forever. **Root cause:** `process.env` is the whole environment block, secrets included. **Right:** log an explicit allowlist of non-secret fields only.

---

## Hands-on proof

Run these right now to see everything above with your own eyes.

```
# 1) See a process's raw environment block in /proc (the kernel reality)
docker run -d --name envtest -e SECRET=hello alpine sleep 1000
docker exec envtest cat /proc/1/environ | tr '\0' '\n'
#   → prints KEY=VALUE lines, including SECRET=hello. Note the NUL→newline translation.

# 2) Prove -e is per-container, not in the image
docker inspect alpine --format '{{json .Config.Env}}'      # image: just PATH
docker inspect envtest --format '{{json .Config.Env}}'     # container: PATH + SECRET=hello

# 3) Prove ENV leaks into history but -e does not
printf 'FROM alpine\nENV BAKED=leaked\n' > Dockerfile.leak
docker build -f Dockerfile.leak -t leaktest .
docker history leaktest        # → shows "ENV BAKED=leaked"
#   there is NO way to see the -e SECRET from step 1 in any image history — it was never in an image

# 4) Prove --env-file keeps quotes (the gotcha)
printf 'PW="quoted"\n' > demo.env
docker run --rm --env-file demo.env alpine sh -c 'echo [$PW]'
#   → [ "quoted" ]   the quotes are part of the value

# 5) Prove a "deleted" secret survives in a layer
printf 'FROM alpine\nRUN echo topsecret > /s && rm /s\n' > Dockerfile.rm
DOCKER_BUILDKIT=0 docker build -f Dockerfile.rm -t rmtest .
docker save rmtest -o rm.tar && mkdir -p rmx && tar -xf rm.tar -C rmx
grep -r topsecret rmx/ && echo "^ secret still on disk in a layer"

# cleanup
docker rm -f envtest; rm -f Dockerfile.leak Dockerfile.rm demo.env rm.tar; rm -rf rmx
```

What each proves: (1) env vars are plain text in `/proc`; (2) the two-box model; (3) `ENV` is permanent, `-e` is not; (4) quote gotcha; (5) layer-history leak is real.

---

## Practice exercises

### Exercise 1 — easy

Write a `hello.env` file with `GREETING=hi` and `NAME=orders`. Run `alpine` with `--env-file hello.env` and a command that prints `"$GREETING $NAME"`. Confirm the output is `hi orders`. Then add a line `BAD = spaced` and observe the error `--env-file` gives you. Explain in one sentence why spaces around `=` are rejected.

### Exercise 2 — medium

Take the `orders-api` from Example 1. Build it once as `orders-api:1.0`. Now run it **twice at the same time** on ports 3001 and 3002, giving each a *different* `DB_HOST` via `-e`, from the **same image**. Curl both `/config` endpoints and confirm they report different hosts. Write down which config box (image vs container) made this possible.

### Exercise 3 — hard (production simulation)

You inherited a Dockerfile that does `RUN echo "$AWS_SECRET_ACCESS_KEY" > /root/.aws/credentials && aws s3 cp ... && rm /root/.aws/credentials`, with `AWS_SECRET_ACCESS_KEY` passed via `--build-arg`. (a) Prove the secret leaked using `docker history` and by extracting layers with `docker save`. (b) Rewrite the build to use `--mount=type=secret` so the credentials file exists only during that one `RUN`. (c) Prove with `docker history --no-trunc` and a layer `grep` that the secret is no longer recoverable. (d) Write a one-paragraph incident note explaining why, even after your fix, the old key must be **rotated** (hint: it already reached the registry).

---

## Mental model checkpoint

Answer from memory:

1. Which syscall receives the environment array, and what is that argument called?
2. Where on the host is the *image's* baked-in `Env` stored versus a *running container's* env?
3. Why does `rm`-ing a secret file in a later layer NOT remove it from the image?
4. What's the difference between `ARG` and `ENV`, and why are **both** unsafe for secrets?
5. What does `--mount=type=secret` do differently that keeps the value off disk?
6. Why can't a running container pick up a changed `--env-file` without being recreated?
7. What's the convention behind mounting a secret as a file and passing `DB_PASSWORD_FILE` instead of `DB_PASSWORD`?

---

## Quick reference card

| Command / instruction | What it does | Key detail |
|---|---|---|
| `-e KEY=VALUE` | Inject one env var at run time | Per-container; shows in `docker inspect`, not in image |
| `-e KEY` (no `=`) | Forward `$KEY` from host shell | Keeps secret off the command line/history |
| `--env-file f` | Load many vars from a host file | No quote stripping, no `${}` interpolation, no spaces around `=` |
| `ENV KEY=VALUE` | Bake a var into the image | Permanent, shipped, in `docker history` — non-secrets only |
| `ARG KEY` | Build-time variable | Value lands in `docker history` — NOT for secrets |
| `--build-arg KEY=V` | Set an `ARG` at build | Leaks into image history |
| `--mount=type=secret` | Expose a secret to one `RUN` | Lives in tmpfs during build, never in a layer |
| `--secret id=..,src=..` | Provide the secret to BuildKit | Requires BuildKit (`DOCKER_BUILDKIT=1`) |
| `secrets:` (Compose/Swarm) | Mount a secret as a file at runtime | Appears under `/run/secrets/<name>` |
| `docker history --no-trunc` | Audit what's baked into layers | Your #1 leak-detection tool |
| `/proc/1/environ` | The live env block in the kernel | Plain text; proves env vars aren't secret |

---

## When would I use this at work?

1. **Promoting one image through environments.** You build `orders-api:1.4.2` once in CI, then run the *identical* image in staging and prod, differing only by `--env-file staging.env` vs `prod.env`. This is the 12-factor "one build, many deploys" rule — and it's why bugs you find in staging are the same binary that ships.
2. **Installing private dependencies without leaking the token.** Your build needs a private npm/GitHub token. You use `--mount=type=secret` so the token never lands in a layer and never reaches your registry, passing an image-scanning audit.
3. **Handing DB/Redis credentials to a container.** Production Postgres and Redis passwords come from Vault or your platform's secret store into an `--env-file` (or `/run/secrets` file) at deploy time. The image on Docker Hub stays generic and password-free, so a leaked image doesn't leak your database.

---

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — namespaces/overlay2, so you understand why layers are immutable), Topic 04 (Images and Layers — content-addressable storage and whiteouts), Topic 06 (Dockerfile in Depth — `ENV`/`ARG` semantics), Topic 12 (`docker run` in Depth — where `-e` fits among all flags).
- **Study after:** Topic 17 (Logging — so you never leak secrets into logs), Topic 20 (Multi-Container Apps — passing DB/Redis config in Compose), Topic 36 (ConfigMaps and Secrets — the Kubernetes version of this exact idea), Topic 50 (Secrets Management at Scale — Vault, External Secrets, sealed secrets).
