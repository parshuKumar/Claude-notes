# 106 — Docker Compose for local dev

## What is this?

Docker Compose is a tool that lets you define and run **multiple containers as one system** using a single YAML file instead of typing long `docker run` commands by hand. A typical backend needs a Node.js API, a Postgres database, and a Redis cache running together — Compose describes all three, how they talk to each other, and what data they persist, in one place. Think of it like a recipe card for your whole local environment: instead of manually cooking each dish (container) separately and remembering the exact ingredients (flags), you write the recipe once and run `docker compose up` to cook the entire meal every time.

## Why does it matter for backend development?

Real backend apps are never just "one Node process" — they depend on a database, a cache, maybe a message queue, and sometimes other internal services. Without Compose, every teammate has to manually install Postgres, install Redis, configure ports, and hope their local setup matches everyone else's — a constant source of "works on my machine" bugs. Compose fixes this: a new developer clones the repo, runs one command, and gets the exact same API + database + cache stack running in seconds, fully isolated from whatever else is on their machine. This is also the direct stepping stone to Topic 108 (CI/CD), where the same `docker-compose.yml` (or its production equivalent) is used to spin up test dependencies in a pipeline.

---

## Syntax / API

```yaml
# File: docker-compose.yml
# Compose file format version — "3.8" is a stable, widely-supported syntax version
version: "3.8"

# services: — each key here becomes one running container
services:

  # ── Our Node.js API container ──────────────────────────────────────────────
  api:
    build: .                        # build the image from the Dockerfile in this folder
    ports:
      - "3000:3000"                 # map host port 3000 → container port 3000 (HOST:CONTAINER)
    environment:
      - NODE_ENV=development         # set env vars directly, OR use env_file below
    env_file:
      - .env                        # load additional secrets/config from a file (not committed to git)
    volumes:
      - .:/app                      # mount current folder into /app — code changes reflect instantly
      - /app/node_modules            # but DON'T overwrite node_modules with the host's (see Mistake 2)
    depends_on:
      - dbConnection                 # wait for the db service to start before starting api
      - cache                        # wait for the redis service to start before starting api
    command: npm run dev             # override the Dockerfile's default CMD for local dev (nodemon)

  # ── PostgreSQL database container ──────────────────────────────────────────
  dbConnection:
    image: postgres:16-alpine        # use the official Postgres image, alpine = smaller size
    environment:
      - POSTGRES_USER=app_user       # creates this DB user on first startup
      - POSTGRES_PASSWORD=app_pass   # password for that user (dev only — never do this in prod)
      - POSTGRES_DB=app_db           # creates this database automatically
    ports:
      - "5432:5432"                  # expose Postgres port so you can connect with a GUI tool too
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume — data survives container restarts

  # ── Redis cache container ───────────────────────────────────────────────────
  cache:
    image: redis:7-alpine            # official lightweight Redis image
    ports:
      - "6379:6379"                  # expose Redis port for local debugging tools

# named volumes must be declared at the top level to be reusable across restarts
volumes:
  pgdata:                            # Docker manages this storage location on the host
```

---

## How it works — line by line

- `version: "3.8"` tells Docker Compose which rules to use for reading the rest of the file — different versions support slightly different features.
- `services:` is the list of containers you want running together as one application.
- `api:` is a service name you chose — Compose turns this into both the container's hostname on the internal network and the label shown in logs.
- `build: .` means "don't pull a ready-made image, build one from the Dockerfile sitting in this same folder."
- `ports: "3000:3000"` opens a door from your laptop's port 3000 into the container's port 3000, so `http://localhost:3000` reaches the API.
- `environment:` sets environment variables directly inside the container, visible to Node as `process.env.NODE_ENV`.
- `env_file: .env` is a second way to load many environment variables at once from a file, kept out of version control.
- `volumes: .:/app` means "keep the container's `/app` folder in sync with my current project folder," so when you edit a file in your editor, the running container sees the change immediately — no rebuild needed.
- `/app/node_modules` (with no host path before the colon) tells Docker "give this folder its own separate storage," which prevents your host machine's `node_modules` from overwriting the one installed inside the Linux container.
- `depends_on:` tells Compose the startup order — start `dbConnection` and `cache` before starting `api` (note: it does not wait for Postgres to be *ready to accept connections*, only for the container to have *started*).
- `command: npm run dev` replaces whatever command the Dockerfile normally runs, letting you use a dev-only command (like `nodemon`) without touching the Dockerfile itself.
- `dbConnection:` is a second service using a pre-built public image instead of your own Dockerfile.
- `image: postgres:16-alpine` pulls a specific, pinned version of Postgres from Docker Hub — pinning avoids surprise breaking changes.
- The three `POSTGRES_*` environment variables are read by the official Postgres image's startup script to auto-create a user, password, and database the very first time the container runs.
- `volumes: pgdata:/var/lib/postgresql/data` points Postgres's internal data folder at a **named volume** (`pgdata`) that Docker manages outside the container, so your database survives even if you delete and recreate the container.
- `cache:` is the third service, running Redis the same simple way — just an image, no build needed.
- The top-level `volumes:` block declares `pgdata` so Compose knows to create and reuse that storage location across `docker compose up`/`down` cycles.

Two containers on the same Compose file can always reach each other by service name — inside the `api` container, connecting to `dbConnection:5432` and `cache:6379` works automatically because Compose creates a private network and registers each service name as a DNS hostname on it.

---

## Example 1 — basic

```yaml
# File: docker-compose.yml
# A minimal two-service setup: one Node API talking to one Redis cache.
version: "3.8"

services:
  api:
    build: .                    # build from the local Dockerfile
    ports:
      - "3000:3000"             # expose the API on localhost:3000
    environment:
      - REDIS_HOST=cache        # the API will connect using the SERVICE NAME, not localhost
      - REDIS_PORT=6379         # Redis's default port

  cache:
    image: redis:7-alpine       # small, official Redis image
    ports:
      - "6379:6379"             # optional — only needed if you want to inspect Redis from your host
```

```js
// File: src/cache.js
// Shows how the Node app inside "api" reaches the "cache" service by name.

const { createClient } = require('redis');   // npm install redis

// process.env.REDIS_HOST is "cache" — the Compose service name, resolved automatically
const redisClient = createClient({
  socket: {
    host: process.env.REDIS_HOST || 'localhost',   // falls back to localhost outside Docker
    port: Number(process.env.REDIS_PORT) || 6379,  // Redis's default port
  },
});

redisClient.on('error', (err) => console.error('Redis error:', err));   // log connection issues

async function connectCache() {
  await redisClient.connect();                     // open the connection
  console.log('Connected to Redis at', process.env.REDIS_HOST);
}

module.exports = { redisClient, connectCache };
```

Run it with:

```
docker compose up --build
# → builds the api image, starts both containers, streams logs from both to your terminal
```

---

## Example 2 — real world backend use case

```yaml
# File: docker-compose.yml
# A realistic local dev stack: Express API + Postgres + Redis, using env_file for secrets.
version: "3.8"

services:
  api:
    build:
      context: .                       # build context is the project root
      dockerfile: Dockerfile.dev        # a separate, dev-only Dockerfile (installs devDependencies too)
    ports:
      - "3000:3000"                    # expose API port
      - "9229:9229"                    # expose Node's debugger port for --inspect
    env_file:
      - .env.development                # DB credentials, JWT secret, etc. — never committed to git
    volumes:
      - .:/app                          # live-reload source code
      - /app/node_modules                 # protect container's own node_modules
    depends_on:
      - dbConnection
      - cache
    command: npm run dev                 # nodemon --inspect=0.0.0.0:9229 src/server.js

  dbConnection:
    image: postgres:16-alpine
    env_file:
      - .env.development                 # POSTGRES_USER / POSTGRES_PASSWORD / POSTGRES_DB live here too
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:                          # Compose can verify Postgres is actually ready, not just started
      test: ["CMD-SHELL", "pg_isready -U app_user"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data                   # persist Redis's append-only file across restarts

volumes:
  pgdata:
  redisdata:
```

```bash
# File: .env.development
# Loaded by env_file: — never commit this file, only commit .env.example

POSTGRES_USER=app_user
POSTGRES_PASSWORD=change_me_in_dev_only
POSTGRES_DB=app_db
DATABASE_URL=postgresql://app_user:change_me_in_dev_only@dbConnection:5432/app_db
JWT_SECRET=dev-only-secret-do-not-use-in-prod
```

```js
// File: src/db.js
// The Node API connects using the service name "dbConnection" as the host — resolved by Docker's DNS

const { Pool } = require('pg');   // npm install pg

// DATABASE_URL comes from .env.development via env_file in docker-compose.yml
const dbConnection = new Pool({
  connectionString: process.env.DATABASE_URL,   // pg parses host/port/user/pass/db from the URL
});

dbConnection.on('error', (err) => {
  console.error('Unexpected database error:', err);   // catch idle-client errors, don't crash silently
});

async function findUserById(userId) {
  // parameterized query — safe from SQL injection (Topic 76)
  const result = await dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  return result.rows[0];   // undefined if no matching user
}

module.exports = { dbConnection, findUserById };
```

Common day-to-day commands for this stack:

```
docker compose up --build      # build images (if changed) and start all services in the foreground
docker compose up -d           # same, but detached (runs in the background)
docker compose logs -f api     # follow just the api service's logs
docker compose exec api sh     # open a shell inside the running api container
docker compose down            # stop and remove containers (named volumes like pgdata survive)
docker compose down -v         # stop AND delete volumes — wipes the database, use carefully
```

---

## Common mistakes

### Mistake 1 — Connecting to `localhost` instead of the service name

```yaml
# ❌ WRONG — inside a container, "localhost" means THAT container itself, not the database container
environment:
  - DATABASE_URL=postgresql://app_user:app_pass@localhost:5432/app_db
```

```yaml
# ✅ CORRECT — use the Compose service name as the hostname; Docker's internal DNS resolves it
environment:
  - DATABASE_URL=postgresql://app_user:app_pass@dbConnection:5432/app_db
  # "dbConnection" here is the service name from docker-compose.yml, not a real domain
```

### Mistake 2 — Mounting the host folder over `node_modules`

```yaml
# ❌ WRONG — this overwrites the container's Linux-built node_modules with your host's
# (macOS/Windows) node_modules, causing native module errors (e.g. bcrypt, sharp)
volumes:
  - .:/app
  # missing the node_modules exclusion below
```

```yaml
# ✅ CORRECT — add an "anonymous volume" for node_modules so it stays container-only
volumes:
  - .:/app               # sync source code for live reload
  - /app/node_modules     # but keep node_modules as installed INSIDE the container
```

### Mistake 3 — Hardcoding secrets directly in docker-compose.yml

```yaml
# ❌ WRONG — docker-compose.yml is usually committed to git, so secrets leak into version control
services:
  api:
    environment:
      - JWT_SECRET=super-secret-production-key-12345
      - DATABASE_URL=postgresql://admin:realPassword@dbConnection:5432/app_db
```

```yaml
# ✅ CORRECT — keep secrets in a gitignored .env file, referenced only by name
services:
  api:
    env_file:
      - .env               # actual values live here, and .env is in .gitignore
```

```
# File: .gitignore
.env
.env.development
.env.production
```

---

## Practice exercises

### Exercise 1 — easy

Write a `docker-compose.yml` with exactly two services:
1. A `cache` service using the `redis:7-alpine` image, exposing port `6379`.
2. An `api` service that builds from a local `Dockerfile`, exposes port `4000`, sets an environment variable `REDIS_HOST=cache`, and uses `depends_on` so it starts after `cache`.

Then write the commands you would run to build and start the stack, and a separate command to stop it while keeping any volumes.

```yaml
// Write your code here
```

---

### Exercise 2 — medium

Extend Exercise 1 into a three-service stack for a small blog API:
1. `api` — builds from `./Dockerfile.dev`, exposes ports `5000` (app) and `9229` (debugger), mounts the current folder into `/app` with a `node_modules` exclusion, and loads secrets from a file named `.env.dev`.
2. `dbConnection` — uses `postgres:16-alpine`, sets `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` via the same `.env.dev` file, and persists data in a named volume called `blogdata`.
3. `cache` — uses `redis:7-alpine` with a named volume called `blogcache` for persistence.

Declare both named volumes at the bottom of the file. Also write the contents of `.env.dev` with realistic (fake) values for `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, and a `DATABASE_URL` that correctly references the `dbConnection` service name.

```yaml
// Write your code here
```

---

### Exercise 3 — hard

Design a `docker-compose.yml` for a multi-service order-processing backend with:
1. `api` — the main Express app, depending on `dbConnection`, `cache`, and `worker` all being started first.
2. `worker` — a second container built from the **same Dockerfile** as `api` but overriding `command` to run `npm run worker` instead of `npm run dev` (simulating a background job processor that shares code with the API but runs a different entrypoint).
3. `dbConnection` — Postgres, with a `healthcheck` block that runs `pg_isready`, and a `depends_on` on `api` and `worker` that waits for this healthcheck to pass (research the `condition: service_healthy` syntax) rather than just waiting for the container to start.
4. `cache` — Redis, with an anonymous or named volume for persistence.
5. All secrets loaded via a single shared `.env` file across every service that needs them.

Then, in a short comment block underneath the YAML, explain in your own words why `depends_on` alone (without `healthcheck` + `condition: service_healthy`) is not enough to guarantee Postgres is ready to accept queries when `api` starts.

```yaml
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE FILE STRUCTURE
  version: "3.8"          → syntax version for the compose file
  services:                → one entry per container
  volumes:                 → named volumes declared at the top level (reusable, persistent)

PER-SERVICE KEYS
  build: .                 → build image from local Dockerfile
  image: postgres:16-alpine → use a pre-built image instead of building
  ports: "HOST:CONTAINER"  → expose a container port to your machine
  environment:              → set env vars inline
  env_file:                 → load env vars from a file (keep secrets out of git)
  volumes:                  → mount host folders or named volumes into the container
  depends_on:                → control startup order between services
  command:                    → override the Dockerfile's default CMD
  healthcheck:                 → define how Compose checks if a service is truly ready

NETWORKING
  Every service name becomes a DNS hostname on Compose's private network
  Inside "api", reach Postgres via "dbConnection:5432", NOT "localhost:5432"
  "localhost" inside a container always means that container itself

VOLUMES
  .:/app                    → live-sync source code (bind mount)
  /app/node_modules           → anonymous volume, prevents host node_modules overwrite
  pgdata:/var/lib/postgresql/data → named volume, persists data across container recreation

COMMON COMMANDS
  docker compose up --build   → build (if needed) and start all services, foreground
  docker compose up -d        → start detached (background)
  docker compose down         → stop and remove containers, KEEP named volumes
  docker compose down -v      → stop, remove containers AND delete named volumes
  docker compose logs -f api  → follow logs for one service
  docker compose exec api sh  → shell into a running service
  docker compose ps           → list running services and their status

NEVER DO
  Connect to "localhost" for another service          → use the service name
  Hardcode passwords/secrets directly in the YAML      → use env_file + .gitignore
  Mount the host folder over node_modules unprotected → always exclude it with an anonymous volume
  Assume depends_on means "ready"                      → it only means "started"; use healthcheck for readiness
```

---

## Connected topics

- **105 — Dockerizing a Node.js app** — you need a working `Dockerfile` before Compose has anything to `build:`; this topic builds directly on it.
- **90 — Connecting to PostgreSQL** — the `pg` connection pool code shown here is exactly what Topic 90 covers in depth, including error handling and query patterns.
- **12 — Environment variables** — `env_file` and `.env` patterns used throughout this doc are the same dotenv conventions taught in Topic 12, now applied at the container level.
