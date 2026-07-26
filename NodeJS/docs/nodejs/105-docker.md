# 105 — Dockerizing a Node.js app

## What is this?

Docker packages your Node.js app together with its exact runtime, dependencies, and OS libraries into a single portable unit called an **image**. A running instance of that image is a **container**. Think of it like a shipping container for cargo — it doesn't matter what's inside or what ship, truck, or crane moves it; the container has a standard shape that fits everywhere. Your app plus "works on my machine" excuses both get sealed inside, so it runs identically on your laptop, a teammate's laptop, and a production server.

## Why does it matter for backend development?

Backend apps depend on a specific Node version, specific npm packages, and sometimes native binaries or OS-level libraries. Without Docker, deploying means hoping the production server has the same setup as your dev machine — a common source of "it worked in dev" bugs. Docker eliminates that gap entirely: you build one image, and it runs the same way everywhere — your machine, CI, staging, and production. Every modern backend job expects you to write a Dockerfile, and it's the foundation for Kubernetes, CI/CD pipelines, and cloud deployment (topics 106–112 build directly on this).

---

## Syntax / API

```dockerfile
# ── Stage 1: build ───────────────────────────────────────────────────────────
FROM node:20-alpine AS builder
# node:20-alpine → small Debian-alternative image with Node 20 preinstalled
WORKDIR /app
# All following commands run inside /app inside the container
COPY package.json package-lock.json ./
# Copy ONLY dependency manifests first — enables Docker layer caching
RUN npm ci
# Clean, reproducible install using the lock file (faster & safer than npm install)
COPY . .
# Now copy the rest of the source code
RUN npm run build
# Compile TypeScript / bundle assets, if applicable

# ── Stage 2: production runtime ─────────────────────────────────────────────
FROM node:20-alpine AS production
# Fresh, clean base image — none of the build tools end up here
WORKDIR /app
ENV NODE_ENV=production
# Tell the app and libraries to run in production mode
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
# Install ONLY production dependencies — no devDependencies bloat
COPY --from=builder /app/dist ./dist
# Copy only the compiled output from the builder stage — not the source or node_modules
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
# Create a dedicated non-root user and group
USER appuser
# Switch to that user — the container no longer runs as root
EXPOSE 3000
# Documents which port the app listens on (does not actually publish it)
CMD ["node", "dist/server.js"]
# The command that starts the app when the container runs
```

```dockerignore
# .dockerignore — files never sent to the Docker build context
node_modules
npm-debug.log
.env
.git
.gitignore
Dockerfile
.dockerignore
dist
coverage
*.md
```

---

## How it works — line by line

A Dockerfile is a recipe: a list of steps Docker follows, top to bottom, to bake an image. Each instruction (`FROM`, `COPY`, `RUN`) creates a new **layer**, and Docker caches layers — if a layer's inputs haven't changed since the last build, Docker reuses the cached result instead of redoing the work. That's why `package.json` is copied and installed *before* the rest of the source code: your dependencies rarely change, but your source code changes constantly. If you copied everything first, every code change would force a full reinstall of every package.

**Multi-stage builds** use two (or more) `FROM` instructions in one file. The first stage (`builder`) has all the tools needed to install dependencies and compile code — including devDependencies and build tools that are large and unnecessary at runtime. The second stage starts completely fresh and copies over *only* the finished output (`COPY --from=builder`). The build tools, source TypeScript files, and dev packages never make it into the final image, which keeps it small and reduces the attack surface.

Running as a **non-root user** matters because containers share the host machine's kernel. If an attacker breaks out of a container running as `root`, they have root-level access to whatever that container can reach. Running as a low-privilege user (`appuser`) means a compromised container has far less power to do damage.

The `.dockerignore` file works exactly like `.gitignore` — it tells Docker which files to exclude when building the image. Without it, Docker would copy your entire `node_modules` folder (which may be built for the wrong OS/architecture), your `.env` secrets, and your `.git` history straight into the image — bloating it and leaking sensitive data.

---

## Example 1 — basic

```js
// File: server.js
// A minimal Express server we will containerize

const express = require('express');
const app = express();

// Read the port from an environment variable, fall back to 3000 for local dev
const PORT = process.env.PORT || 3000;

// Simple health endpoint — useful once the app is running inside a container
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });    // container orchestrators poll this
});

app.get('/', (req, res) => {
  res.send('Hello from a Dockerized Node.js app');
});

app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});
```

```dockerfile
# File: Dockerfile
# Single-stage Dockerfile — fine for a quick demo, not for production (see Example 2)
FROM node:20-alpine
# Small official Node.js image based on Alpine Linux
WORKDIR /app
# Working directory inside the container
COPY package.json package-lock.json ./
# Copy manifests first so npm install is cached when only source code changes
RUN npm ci --omit=dev
# Install dependencies exactly as locked, skipping devDependencies
COPY . .
# Copy the rest of the app's source code
EXPOSE 3000
# Document the port the app listens on
CMD ["node", "server.js"]
# Command executed when the container starts
```

```bash
# Build the image and tag it "myapp:1.0"
docker build -t myapp:1.0 .

# Run a container from that image, mapping host port 3000 to container port 3000
docker run -p 3000:3000 myapp:1.0

# Visit http://localhost:3000 — the app is now running inside a container
```

---

## Example 2 — real world backend use case

```dockerfile
# File: Dockerfile
# Production-grade multi-stage build for a TypeScript Express API
# Optimized for size, security, and rebuild speed

# ── Stage 1: install ALL deps and compile TypeScript ────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
# Full install (includes devDependencies like typescript, @types/*)
COPY tsconfig.json ./
COPY src ./src
RUN npm run build
# Compiles src/**/*.ts into dist/**/*.js

# ── Stage 2: install ONLY production deps ───────────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
# Smaller node_modules — no typescript, no test libraries, no dev tooling

# ── Stage 3: final lean runtime image ────────────────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production
# Bring in only what the running app actually needs
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
# Create a non-root user to run the app
RUN addgroup -S nodeapp && adduser -S nodeapp -G nodeapp
USER nodeapp
EXPOSE 4000
# Basic container-level health check — Docker marks the container unhealthy on failure
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD node -e "require('http').get('http://localhost:4000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"
CMD ["node", "dist/server.js"]
```

```js
// File: src/server.js — reads config via environment variables (topic 12)
const express = require('express');
const dbConnection = require('./db');   // your database connection module

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 4000;

// Health endpoint used by the HEALTHCHECK instruction and orchestrators (K8s, ECS)
app.get('/health', async (req, res) => {
  try {
    await dbConnection.ping();          // verify DB is reachable, not just "server up"
    res.status(200).json({ status: 'ok' });
  } catch (err) {
    res.status(503).json({ status: 'unavailable' });
  }
});

app.post('/api/v1/users', async (req, res) => {
  const requestBody = req.body;         // { name, email }
  const userId = await dbConnection.createUser(requestBody);
  res.status(201).json({ userId });
});

app.listen(PORT, () => {
  console.log(`API server listening on port ${PORT}`);
});
```

```bash
# Build, tag, and push to a registry so any server can pull it
docker build -t registry.example.com/backend-api:1.4.2 .
docker push registry.example.com/backend-api:1.4.2

# Run with secrets injected as environment variables, never baked into the image
docker run -d -p 4000:4000 \
  -e DATABASE_URL="postgres://user:pass@db-host:5432/mydb" \
  -e JWT_SECRET="$(cat ./secrets/jwt_secret.txt)" \
  --name backend-api registry.example.com/backend-api:1.4.2
```

---

## Common mistakes

### Mistake 1 — Copying everything before installing dependencies

```dockerfile
# ❌ WRONG — copies all source code first, so ANY code change invalidates
# the npm install layer, forcing a full reinstall on every rebuild
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

```dockerfile
# ✅ CORRECT — copy manifests first, install, THEN copy source code
# npm install is cached and skipped unless package.json/lock actually changes
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

### Mistake 2 — No .dockerignore, shipping node_modules and secrets into the image

```dockerfile
# ❌ WRONG — without a .dockerignore, `COPY . .` drags in the host's
# node_modules (wrong architecture, huge), .env secrets, and .git history
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

```
# ✅ CORRECT — add a .dockerignore file next to the Dockerfile, then COPY . .
# only brings in source code — node_modules is installed fresh inside the
# container, matching its OS/architecture, and secrets never travel with it
node_modules
.env
.env.*
.git
dist
*.log
```

### Mistake 3 — Running the container as root

```dockerfile
# ❌ WRONG — no USER instruction means the process runs as root inside
# the container by default; a container escape gives an attacker root access
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
CMD ["node", "server.js"]
```

```dockerfile
# ✅ CORRECT — create and switch to an unprivileged user before CMD runs
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --omit=dev
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
# Ensure the app's own files are readable by the new user
RUN chown -R appuser:appgroup /app
USER appuser
CMD ["node", "server.js"]
```

---

## Practice exercises

### Exercise 1 — easy

Create a minimal Express app with a single `GET /` route that returns `"Hello Docker"`. Write a `Dockerfile` (single stage is fine) that:
1. Uses `node:20-alpine` as the base image
2. Copies `package.json` and `package-lock.json` first, then runs `npm ci --omit=dev`
3. Copies the rest of the source code
4. Exposes port `3000`
5. Starts the app with `CMD`

Also write a `.dockerignore` file that excludes `node_modules`, `.git`, and `.env`. Build the image, run a container from it, and confirm `curl http://localhost:3000` returns the expected response.

```dockerfile
// Write your code here
```

---

### Exercise 2 — medium

Convert Exercise 1 into a **multi-stage build**:
1. Stage `builder` — installs all dependencies (including dev) and runs a build step (simulate one with `RUN echo "build step" > /app/build.log` if you don't have a real compile step)
2. Stage `production` — starts fresh from `node:20-alpine`, installs only production dependencies, and copies over just the app files needed to run (not the entire builder stage)
3. Add a non-root user and switch to it with `USER` before the final `CMD`
4. Add a `HEALTHCHECK` instruction that curls or requests a `/health` endpoint you add to the app

Build the image and run `docker inspect` on the resulting container to confirm the health check is configured and the process is not running as root.

```dockerfile
// Write your code here
```

---

### Exercise 3 — hard

Build a production-ready Dockerfile for a small TypeScript Express API with a database dependency, satisfying ALL of the following:
1. Three stages: `builder` (full install + `tsc` compile), `deps` (production-only install), `production` (final lean image)
2. Final image copies only `dist/`, `node_modules` (from the `deps` stage), and `package.json` — never the TypeScript source or devDependencies
3. Runs as a dedicated non-root user with correct file ownership (`chown`)
4. Reads `PORT`, `DATABASE_URL`, and `JWT_SECRET` from environment variables — nothing hardcoded, nothing baked into the image
5. Includes a `HEALTHCHECK` that fails if the app cannot reach its database (reuse the `/health` pattern that pings the DB)
6. A `.dockerignore` that excludes `node_modules`, `.git`, `.env*`, `dist`, and test files
7. After building, run `docker history <image>` and confirm no layer contains your `.env` file or raw TypeScript source

```dockerfile
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CONCEPTS
  Image      → the packaged, immutable blueprint (built from a Dockerfile)
  Container  → a running instance of an image
  Layer      → each Dockerfile instruction creates a cached, reusable layer
  Registry   → remote storage for images (Docker Hub, ECR, GCR, private registry)

KEY DOCKERFILE INSTRUCTIONS
  FROM <image>            → base image to start from
  WORKDIR <path>           → sets the working directory for following instructions
  COPY <src> <dest>        → copy files from host into the image
  RUN <command>            → executes a command at BUILD time, creates a layer
  ENV <key>=<value>        → sets an environment variable inside the image
  EXPOSE <port>            → documents the port (does not publish it — use -p at runtime)
  USER <user>              → switches to a non-root user for remaining instructions
  CMD ["executable", ...]  → the default command run when the container STARTS
  HEALTHCHECK              → command Docker runs periodically to verify the app is alive

LAYER CACHING RULE
  Order instructions from LEAST to MOST frequently changing:
  FROM → WORKDIR → COPY package*.json → RUN npm ci → COPY . . → CMD
  This way, code changes don't invalidate the (slow) npm install layer

MULTI-STAGE BUILDS
  FROM node:20-alpine AS builder    → build tools + devDependencies live here
  FROM node:20-alpine AS production → fresh image, COPY --from=builder only what's needed
  Result: final image has no compiler, no source TS, no dev packages — smaller & safer

NON-ROOT USER
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  RUN chown -R appuser:appgroup /app  &&  USER appuser
  → limits blast radius if the container is ever compromised

IMAGE OPTIMIZATION CHECKLIST
  [ ] Use *-alpine or *-slim base images, not the full node image
  [ ] .dockerignore excludes node_modules, .git, .env, dist, *.log
  [ ] npm ci (not npm install) — faster, reproducible, respects lock file
  [ ] npm ci --omit=dev in the final stage — no devDependencies at runtime
  [ ] Multi-stage build — build tools never reach production image
  [ ] Non-root USER before CMD
  [ ] Pin the base image tag (node:20-alpine, not node:latest)

COMMON COMMANDS
  docker build -t <name>:<tag> .        → build an image from Dockerfile in cwd
  docker run -p 3000:3000 <image>       → run a container, map host:container ports
  docker run -e KEY=value <image>       → pass an environment variable at runtime
  docker ps                             → list running containers
  docker logs <container>               → view container stdout/stderr
  docker exec -it <container> sh        → open a shell inside a running container
  docker images                         → list local images and their sizes
  docker history <image>                → inspect what each layer added
```

---

## Connected topics

- **106 — Docker Compose for local dev** — orchestrates this Dockerfile alongside a database and Redis container, with linked services, volumes, and shared env files
- **12 — Environment variables** — how `PORT`, `DATABASE_URL`, and `JWT_SECRET` get injected into the container at runtime instead of being hardcoded in the image
- **64 — Graceful shutdown** — containers receive `SIGTERM` on `docker stop`; your app must handle it to drain connections before the container is killed
