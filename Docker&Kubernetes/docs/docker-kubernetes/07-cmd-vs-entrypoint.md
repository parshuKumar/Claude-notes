# 07 — CMD vs ENTRYPOINT
## Section: Docker in Practice

## ELI5 — The Simple Analogy

Imagine a **coffee machine** in an office.

- **ENTRYPOINT** is the machine itself, bolted to the counter: "I am a coffee maker. That is what I always do."
- **CMD** is the *default drink* printed on the front: "Unless you press another button, I'll make a latte."

If you walk up and press "espresso", the machine still makes coffee (ENTRYPOINT is fixed) but now it makes an espresso instead of the default latte (CMD was the default and you overrode it).

Now imagine a different machine with **no ENTRYPOINT** — just a big screen showing "latte" (only CMD). If you press "espresso", the screen changes and it makes espresso. But if you type "make toast", it will try to make toast, because there is no fixed job — you can replace the whole thing.

That is the entire difference:
- **ENTRYPOINT** = the fixed job the container always does.
- **CMD** = the default arguments/command, easily swapped.

## The Linux kernel feature underneath

When a container starts, `runc` (Topic 03) performs a real Linux `execve()` syscall. `execve()` takes an **argument vector** — `argv[]` — an array of strings where `argv[0]` is the program and the rest are its arguments. Whatever ends up in that array becomes the container's **PID 1** inside the new PID namespace (Topic 02).

CMD and ENTRYPOINT are just two halves of how Docker builds that final `argv[]`:

```
final argv[]  =  ENTRYPOINT array  ++  CMD-or-runtime-args array
```

So the kernel does not know the words "CMD" or "ENTRYPOINT". It only sees the concatenated array Docker hands to `execve()`. Everything else — override rules, exec vs shell form — is Docker deciding *how to assemble that one array*.

Two consequences that dominate this topic:

1. **PID 1 is special.** In a PID namespace, PID 1 is the "init" process. The kernel gives PID 1 special signal behavior: signals like `SIGTERM` have **no default action** for PID 1 unless the program explicitly installs a handler. This is why signal handling depends entirely on *what becomes PID 1* — your Node process, or a shell wrapping it.

2. **`execve()` replaces the process image.** When exec form is used, your program *is* PID 1. When shell form is used, `/bin/sh` is PID 1 and your program is a child — a completely different signal situation.

## What is this?

`CMD` and `ENTRYPOINT` are the two Dockerfile instructions that decide **what process runs when a container starts**. `ENTRYPOINT` defines the fixed executable; `CMD` provides default arguments (or a default command). They can be used alone or combined, and each has a **shell form** and an **exec form** that behave very differently.

## Why does it matter for a backend developer?

Because these two instructions control three things you hit constantly:

- **Graceful shutdown.** If your `orders-api` doesn't receive `SIGTERM`, it can't finish in-flight requests, flush logs, or close Postgres/Redis connections before dying. Deploys and `docker stop` then take the full grace period and kill the process hard, dropping requests. The wrong form of CMD/ENTRYPOINT is the #1 cause.
- **Overridability.** Do you want `docker run orders-api npm test` to run your tests, or to be ignored? That depends on whether you used CMD, ENTRYPOINT, or both.
- **Building a CLI-like image.** If your image is really a tool (say a DB migration runner), ENTRYPOINT makes `docker run migrator up` and `docker run migrator down` feel natural.

Getting this wrong produces containers that won't stop cleanly, ignore the arguments you pass, or run the wrong thing entirely.

## The physical reality

Both CMD and ENTRYPOINT are **metadata** — they live in the image config JSON (Topic 06), not in any filesystem layer. You can read them directly:

```bash
docker inspect orders-api:1.0 --format 'ENTRYPOINT={{json .Config.Entrypoint}} CMD={{json .Config.Cmd}}'
# ENTRYPOINT=["node"] CMD=["server.js"]
```

At runtime, look at what actually became PID 1 inside the container:

```bash
docker exec orders sh -c 'cat /proc/1/cmdline | tr "\0" " "; echo'
# node server.js          ← exec form: node IS pid 1  ✅
# /bin/sh -c node server.js   ← shell form: sh is pid 1, node is a child ❌
```

`/proc/1/cmdline` is the kernel's record of PID 1's argument vector. This single file tells you the whole truth about which form you used.

## How it works — step by step

When you run `docker run [flags] IMAGE [ARGS...]`, Docker builds the final `argv[]` like this:

1. **Read the image config** for its `Entrypoint` and `Cmd` arrays.
2. **Did you pass ARGS after the image name?**
   - If yes → those ARGS **replace `Cmd`**. `Entrypoint` is untouched.
   - If no → `Cmd` from the image is used.
3. **Concatenate:** `final = Entrypoint (if any) ++ effective_Cmd`.
4. **Shell form wrapping (per instruction):** if an instruction was written in shell form (`CMD node server.js`), Docker stored it as `["/bin/sh","-c","node server.js"]`. So shell form silently prepends `/bin/sh -c`.
5. **`runc` calls `execve(final[0], final)`** inside the container's namespaces. `final[0]` becomes **PID 1**.
6. **Signals:** `docker stop` sends `SIGTERM` to PID 1, waits 10s (default, `--time`), then `SIGKILL`. Whether your app sees the `SIGTERM` depends entirely on whether it *is* PID 1.

Let's make the concatenation concrete:

```
Dockerfile:
  ENTRYPOINT ["node"]
  CMD ["server.js"]

docker run orders-api                → execve: ["node","server.js"]      (CMD default)
docker run orders-api app.js         → execve: ["node","app.js"]         (arg replaced CMD)
docker run orders-api --version      → execve: ["node","--version"]      (arg replaced CMD)
docker run --entrypoint sh orders-api -c 'echo hi'  → execve: ["sh","-c","echo hi"]  (ENTRYPOINT overridden)
```

## Exact syntax breakdown

### The two forms — this is the crux

**Exec form** — a JSON array. No shell involved. Your program is PID 1.

```
CMD ["node", "server.js"]
│   ││     │  │        ││
│   ││     │  │        │└─ closing bracket
│   ││     │  └─────────── second element = first argument
│   ││     └────────────── comma-separated
│   │└──────────────────── first element = the executable (becomes argv[0])
│   └───────────────────── JSON array = EXEC form (double quotes are REQUIRED)
└───────────────────────── runs node DIRECTLY: execve("node", ["node","server.js"])
```

**Shell form** — a bare string. Docker wraps it in `/bin/sh -c`. The shell is PID 1.

```
CMD node server.js
│   │
│   └─ a plain string (no brackets, no quotes)
└───── SHELL form: stored as ["/bin/sh","-c","node server.js"] → sh is PID 1, node is its child
```

Key behavioral differences:

| | Exec form `["node","server.js"]` | Shell form `node server.js` |
|---|---|---|
| PID 1 | `node` (your app) ✅ | `/bin/sh` ❌ |
| Receives SIGTERM | Yes | No (sh doesn't forward it) |
| `$VAR` expansion | No (no shell) | Yes (shell expands) |
| `&&`, pipes, `>` | No | Yes |
| Needs a shell in image | No (works on distroless) | Yes (needs `/bin/sh`) |

### ENTRYPOINT + CMD combined — the recommended pattern

```
ENTRYPOINT ["node"]        ← the fixed executable
CMD        ["server.js"]   ← default argument, overridable at runtime
```

```
Result of concatenation:
  ENTRYPOINT ["node"]  +  CMD ["server.js"]  =  execve(["node","server.js"])
  │                       │
  │                       └─ default arg; `docker run img app.js` swaps it → ["node","app.js"]
  └────────────────────── always runs; unchanged by run-time args
```

### Override rules at a glance

```
docker run IMAGE                    → ENTRYPOINT + CMD(default)
docker run IMAGE arg1 arg2          → ENTRYPOINT + [arg1, arg2]   (CMD replaced)
docker run --entrypoint EXE IMAGE   → EXE + CMD(default)          (ENTRYPOINT replaced)
docker run --entrypoint EXE IMAGE a → EXE + [a]                   (both replaced)
```

## Example 1 — basic

Three tiny images to feel the difference. Build each and observe.

**A) CMD only (overridable command):**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js ./
CMD ["node", "server.js"]     # default; fully replaceable
```

```bash
docker run img-a                 # runs: node server.js
docker run img-a node --version  # runs: node --version   (CMD fully replaced)
docker run img-a echo hello      # runs: echo hello        (CMD fully replaced)
```

**B) ENTRYPOINT only (fixed command):**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js ./
ENTRYPOINT ["node", "server.js"]   # always runs node server.js
```

```bash
docker run img-b                 # runs: node server.js
docker run img-b echo hello      # runs: node server.js echo hello  ← args APPENDED, not replaced!
docker run --entrypoint sh img-b # runs: sh   (only --entrypoint overrides it)
```

**C) ENTRYPOINT + CMD (fixed exe, default arg):**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY server.js ./
ENTRYPOINT ["node"]        # always node
CMD ["server.js"]          # default script, replaceable
```

```bash
docker run img-c                 # runs: node server.js
docker run img-c --version       # runs: node --version   (CMD replaced by --version)
docker run img-c other.js        # runs: node other.js
```

Notice B: passing `echo hello` did NOT replace the entrypoint — it got appended, producing the nonsense `node server.js echo hello`. That surprise is exactly why you must understand the combination.

## Example 2 — production scenario

**The scenario:** Your `orders-api` runs fine, but every deploy is slow and lossy. During a rolling update, Kubernetes (or `docker stop`) sends `SIGTERM`, waits 30s, then `SIGKILL`s the pod. Users see dropped connections. Your app *has* a graceful shutdown handler, but it never fires. Logs show no "shutting down" message.

Your app code:

```js
// server.js — orders-api
const server = app.listen(3000, () => console.log('orders-api up'));

process.on('SIGTERM', () => {          // ← this handler NEVER runs in the broken setup
  console.log('SIGTERM received, draining...');
  server.close(async () => {
    await pgPool.end();                // close Postgres connections
    await redis.quit();                // close Redis
    console.log('shutdown complete');
    process.exit(0);
  });
});
```

The broken Dockerfile:

```dockerfile
CMD npm start          # ❌ shell form. And npm spawns node as ITS child too.
```

What actually happens at the kernel level:

```
PID 1: /bin/sh -c "npm start"
  └─ PID 7:  npm
       └─ PID 18: node server.js      ← your app, TWO levels below PID 1

docker stop → SIGTERM to PID 1 (sh) → sh ignores/exits, never forwards → node never sees it
→ 30s later → SIGKILL to everything → in-flight requests dropped, no drain
```

The fix — exec form so `node` is PID 1 directly:

```dockerfile
CMD ["node", "server.js"]     # ✅ exec form: node IS pid 1, receives SIGTERM
```

Now:

```
PID 1: node server.js
docker stop → SIGTERM to PID 1 (node) → handler fires → drains → exits 0 in ~200ms
```

**But what if you genuinely need `npm start`** (or a shell for `$VAR`)? Two correct options:

1. Use an init like `tini` so signals are forwarded to children:
   ```dockerfile
   RUN apk add --no-cache tini
   ENTRYPOINT ["/sbin/tini", "--"]   # tini becomes PID 1 and forwards SIGTERM to node
   CMD ["node", "server.js"]
   ```
   Or simply `docker run --init` (Topic 12), which injects a tiny init as PID 1.

2. Best: skip `npm start` entirely and exec `node` directly. `npm` adds a process layer with no benefit in production.

Result: graceful shutdown works, drains finish in milliseconds, and rolling updates (Topic 46) are truly zero-downtime.

## Common mistakes

**1. Shell form breaks SIGTERM (the classic)**

```dockerfile
CMD node server.js
```

Symptom:

```
$ time docker stop orders
orders
real    0m10.3s          ← waited the full grace period, then SIGKILL
```

Root cause: `/bin/sh` is PID 1; it doesn't forward `SIGTERM` to `node`. Fix: `CMD ["node","server.js"]`, or add `tini`/`--init` if you truly need a shell.

**2. Expecting `docker run img arg` to replace an ENTRYPOINT**

```dockerfile
ENTRYPOINT ["node", "server.js"]
```
```
$ docker run orders-api --help
# runs: node server.js --help   ← --help appended, not a replacement
```

Root cause: run-time args replace CMD, never ENTRYPOINT. Fix: split into `ENTRYPOINT ["node"]` + `CMD ["server.js"]`, or override with `--entrypoint`.

**3. Exec form with a variable that never expands**

```dockerfile
ENV PORT=3000
CMD ["node", "server.js", "--port=$PORT"]
```

Symptom: your app literally sees the string `$PORT`, not `3000`.

```
Error: invalid port "$PORT"
```

Root cause: exec form does NOT run a shell, so `$PORT` is not expanded. Fix options: read `process.env.PORT` in code (best), or use shell form `CMD node server.js --port=$PORT`, or `CMD ["sh","-c","node server.js --port=$PORT"]`.

**4. JSON array with single quotes (silent fallback to shell form)**

```dockerfile
CMD ['node', 'server.js']
```

Symptom: Docker can't parse it as JSON (JSON requires double quotes), so it treats the whole thing as **shell form** — reintroducing the PID-1 problem. Older/strict builders warn:

```
JSONArgsRecommended: JSON arguments recommended for CMD to prevent unintended behavior
```

Root cause: JSON arrays require double quotes. Fix: `CMD ["node", "server.js"]`.

**5. Two CMDs (or two ENTRYPOINTs) — only the last wins**

```dockerfile
CMD ["node", "a.js"]
CMD ["node", "b.js"]     # only THIS one takes effect; the first is dead code
```

Root cause: each is overwritten by the next. Fix: keep exactly one of each.

## Hands-on proof

```bash
# Setup
mkdir cmd-vs-entry && cd cmd-vs-entry
cat > server.js <<'EOF'
const s = require('http').createServer((_, res) => res.end('ok')).listen(3000);
console.log('up, pid', process.pid);
process.on('SIGTERM', () => { console.log('SIGTERM received, exiting'); process.exit(0); });
EOF

# --- Broken: shell form ---
printf 'FROM node:20-alpine\nWORKDIR /app\nCOPY server.js ./\nCMD node server.js\n' > Dockerfile
docker build -t sig-shell . >/dev/null
docker run -d --name shell sig-shell >/dev/null
docker exec shell sh -c 'cat /proc/1/cmdline | tr "\0" " "; echo'
#   → /bin/sh -c node server.js        (sh is PID 1)
time docker stop shell
#   → takes ~10s: node never saw SIGTERM

# --- Fixed: exec form ---
printf 'FROM node:20-alpine\nWORKDIR /app\nCOPY server.js ./\nCMD ["node","server.js"]\n' > Dockerfile
docker build -t sig-exec . >/dev/null
docker run -d --name exec sig-exec >/dev/null
docker exec exec sh -c 'cat /proc/1/cmdline | tr "\0" " "; echo'
#   → node server.js                   (node is PID 1)
time docker stop exec
#   → ~0.2s, and: docker logs exec  shows "SIGTERM received, exiting"

# --- Override rules ---
docker inspect sig-exec --format 'ENTRYPOINT={{json .Config.Entrypoint}} CMD={{json .Config.Cmd}}'
docker run --rm sig-exec node --version     # CMD replaced → prints node version
docker run --rm --entrypoint sh sig-exec -c 'echo overridden'   # entrypoint replaced

# Cleanup
docker rm -f shell exec >/dev/null; docker rmi sig-shell sig-exec >/dev/null
```

You directly observed: shell form makes `sh` PID 1 and blocks SIGTERM (10s stop), exec form makes `node` PID 1 and delivers SIGTERM (instant graceful stop).

## Practice exercises

### Exercise 1 — easy
Given `ENTRYPOINT ["node"]` and `CMD ["server.js"]`, predict the final command for: (a) `docker run img`, (b) `docker run img app.js`, (c) `docker run img --version`, (d) `docker run --entrypoint sh img -c 'ls'`. Then build the image and verify each with `docker run`.

### Exercise 2 — medium
You have `CMD npm start`. Reproduce the 10-second `docker stop` delay, confirm with `/proc/1/cmdline` that a shell is PID 1, then fix it two different ways: (1) exec-form `node`, and (2) keep a shell/npm but add `tini` (or `--init`) so SIGTERM reaches node. Prove both fixes with a `time docker stop` under 1 second and a "SIGTERM received" log line.

### Exercise 3 — hard (production simulation)
Turn `orders-api` into a small CLI-style image: `ENTRYPOINT` runs a `node cli.js`, and `CMD` defaults to `serve`. It should support `docker run orders migrate`, `docker run orders serve`, and `docker run orders --help`, all by *appending* the subcommand as an argument to `cli.js`. Ensure the served process is PID 1 (verify via `/proc/1/cmdline`) and handles SIGTERM. Then show how a teammate can still get a debug shell with `--entrypoint sh`.

## Mental model checkpoint

1. In terms of `argv[]`, how does Docker combine ENTRYPOINT and CMD?
2. What replaces CMD at runtime? What replaces ENTRYPOINT?
3. Why does shell-form CMD break `SIGTERM` handling, in PID-namespace terms?
4. Why does `$PORT` not expand in exec form, and what are three fixes?
5. If you pass `docker run img extra-arg` and the image has only an ENTRYPOINT, what runs?
6. When would you deliberately add `tini` or `--init`?
7. How do you read the effective ENTRYPOINT and CMD out of an image?

## Quick reference card

| Situation | Result |
|-----------|--------|
| `CMD ["a","b"]` only | Runs `a b`; `docker run img x y` → runs `x y` (fully replaced) |
| `ENTRYPOINT ["a","b"]` only | Runs `a b`; `docker run img x` → runs `a b x` (appended) |
| `ENTRYPOINT ["a"]` + `CMD ["b"]` | Runs `a b`; `docker run img c` → runs `a c` (CMD replaced) |
| Exec form `["node","server.js"]` | `node` is PID 1; gets SIGTERM; no `$VAR`; no shell needed |
| Shell form `node server.js` | `/bin/sh` is PID 1; blocks SIGTERM; `$VAR` works; needs a shell |
| `--entrypoint EXE` | Overrides ENTRYPOINT; args after image become CMD |
| Two CMDs / two ENTRYPOINTs | Only the last one applies |
| `docker run --init` / `tini` | Injects a real init as PID 1 that forwards signals |

## When would I use this at work?

1. **Fixing lossy deploys.** Rollouts drop requests and take the full grace period. You find shell-form CMD, switch to exec form, and graceful shutdown (drain Postgres/Redis) starts firing — deploys become zero-downtime (Topic 46).
2. **Building a task/CLI image.** You containerize a DB migration tool. `ENTRYPOINT ["node","migrate.js"]` + `CMD ["up"]` lets ops run `docker run migrator up` and `docker run migrator down` naturally, while `--entrypoint sh` still gives a debug shell.
3. **Supporting flexible startup.** Your image should run the server by default but occasionally run a one-off script. `ENTRYPOINT ["node"]` + `CMD ["server.js"]` means `docker run img seed.js` reuses the same image to run a seeding script — no second image needed.

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — PID namespaces, signals), Topic 03 (Docker Architecture — runc/execve), Topic 06 (Dockerfile in Depth — where CMD/ENTRYPOINT sit).
- **Study after:** Topic 11 (Production Dockerfile for Node.js — signal handling in the full template), Topic 12 (docker run in Depth — `--init`, `--entrypoint`, `--time`), Topic 16 (Container Lifecycle — stop/kill states), Topic 26 (Health Checks), Topic 46 (Rolling Updates — why graceful shutdown matters at scale).
