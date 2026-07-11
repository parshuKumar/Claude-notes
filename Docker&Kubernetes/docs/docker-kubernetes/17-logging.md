# 17 — Logging

## Section: Docker in Practice

---

## ELI5 — The Simple Analogy

Imagine your app is a chef in a kitchen. As the chef works, they call out everything they do: "Order in!", "Steak's burning!", "Table 4 served!" Those shout-outs are **logs** — the app narrating what's happening.

Now, where do those shout-outs go?

- In a normal kitchen, the chef shouts into the room and the words vanish into the air. That's a plain program: it prints to the screen and the words scroll away.
- Docker puts a **helper with a notepad** right next to the chef. Every time the chef shouts, the helper writes it down on the notepad, with a timestamp: "12:01 Order in! 12:03 Table 4 served!" That notepad is a **log file on disk**, and the helper is the **log driver**.
- `docker logs` is you walking over and reading the notepad. You can read it even after the chef goes home (container stopped), because the notepad is still lying there.

Two important details a kid gets immediately:

1. **The chef must shout out loud, not whisper into a drawer.** If the chef quietly writes notes into a private drawer (the app writes to its own log file *inside* the container), the helper never hears it, and you never see it in `docker logs`. Containers want you to shout — print to the screen (stdout/stderr).
2. **The notepad can fill up the whole kitchen.** If nobody ever tears off old pages, the notepad grows until it buries the kitchen (fills the disk). That tearing-off is **log rotation**.

The choice of *which helper with which notepad* — write it locally, ship it to a central office, hand it to the city records department — is the **log driver** choice.

---

## The Linux kernel feature underneath

Container logging is built on the oldest idea in Unix: **file descriptors and streams.**

Every Linux process is born with three open file descriptors:

```
fd 0  stdin   ← where input comes from
fd 1  stdout  ← normal output ("Order in!")     → this is what console.log() writes to
fd 2  stderr  ← error output ("Steak's burning!") → this is what console.error() writes to
```

A file descriptor is just an integer index into the kernel's per-process **open file table**. The magic of Unix is that fd 1 and fd 2 can point at *anything* — a terminal, a regular file, or a **pipe**. The process doesn't know or care; it just calls `write(1, "hello\n", 6)`.

**How Docker captures your logs — the pipe.** When Docker (via containerd's `shim`, Topic 03) starts your container, it does NOT connect fd 1/fd 2 to a terminal. Instead it creates two **pipes** (via the `pipe(2)` syscall) and wires the container process's fd 1 and fd 2 to the *write* ends. Docker's shim holds the *read* ends.

```
   Your Node process                         containerd-shim (on the host)
   ┌───────────────┐                         ┌──────────────────────────┐
   │ console.log() │                         │  reads from pipe          │
   │ write(1, ...) ─┼──── pipe (kernel) ─────┼─► tags each line with     │
   │ write(2, ...) ─┼──── pipe (kernel) ─────┼─► {time, stream: stdout}  │
   └───────────────┘                         │  hands to the LOG DRIVER  │
                                             └──────────────────────────┘
```

So the kernel primitive is: **anonymous pipes** connecting your process's stdout/stderr to the Docker shim. The shim reads every line, attaches a timestamp and a stream label (stdout vs stderr), and forwards the structured record to the configured **log driver**, which decides where it physically lands.

This is why the golden rule of container logging is **"log to stdout/stderr."** That's the only stream the kernel pipe captures. If your app writes to a file inside the container, those bytes go to the overlay2 writable layer (Topic 02/14), never through the pipe, and `docker logs` shows nothing.

One more kernel-adjacent fact: the default `json-file` driver writes to a real file on the host filesystem (`ext4`/`xfs`), and every `write()` to it consumes real disk blocks. Nothing rotates it unless you configure rotation. An unbounded pipe of logs becomes an unbounded file — and a full `/var` partition takes down the whole Docker daemon.

---

## What is this?

Logging is how a container's output is captured, stored, and read. Docker connects your process's stdout/stderr to a **log driver** — a pluggable backend (`json-file`, `journald`, `syslog`, `fluentd`, etc.) that decides where log lines physically go: a JSON file on the host, the systemd journal, a remote syslog server, or a log-shipping pipeline.

This topic covers `docker logs`, the main log drivers and their trade-offs, exactly where logs live on disk, why you should emit structured JSON logs from Node, how to rotate logs so they don't fill the disk, and how logging really works once you have many containers in production.

---

## Why does it matter for a backend developer?

When your `orders-api` returns a 500 at 3am, the log is often the *only* evidence of what happened. If you don't understand logging, these very specific things go wrong:

- **You can't find your logs.** Your app writes to `/app/logs/app.log` inside the container. `docker logs orders` is empty. You have no idea the app is even working. (Cause: not logging to stdout.)
- **The disk fills and the whole host dies.** A chatty container writes 5GB of logs to a `json-file` with no rotation. `/var/lib/docker` fills, and *every* container on that host fails to write — a total node outage from one noisy app.
- **Your logs are unsearchable.** You logged free-form strings like `"user 42 did a thing"`. In production you can't answer "show all errors for order 9981" because there's no structure to query. Structured JSON logs make every field searchable in Datadog/CloudWatch/Loki.
- **You lose logs when a container dies.** With the default `json-file` driver and no shipping, when you `docker rm` a crashed container its logs are deleted with it. You needed those logs to understand the crash.

Getting logging right is the difference between "I can see exactly what my API did" and "the API is a black box." It's also the foundation for Kubernetes observability (Topic 51).

---

## The physical reality

**Default `json-file` driver — where your logs actually live:**

```
/var/lib/docker/containers/<container-id>/
└── <container-id>-json.log
    │
    └─ one JSON object PER LINE (NDJSON). Example line:
       {"log":"orders-api up on 3000\n","stream":"stdout","time":"2026-07-12T09:01:22.14Z"}
        │                                │                 │
        │                                │                 └─ RFC3339 timestamp the shim attached
        │                                └─ which stream: "stdout" or "stderr"
        └─ the exact bytes your app wrote (newline included)
```

`docker logs` literally reads and reformats this file. That's why it works on a stopped container — the file persists until `docker rm`.

**With rotation configured**, you get numbered rotated files next to it:

```
/var/lib/docker/containers/<id>/
├── <id>-json.log        ← current (active) log, up to max-size
├── <id>-json.log.1      ← previous chunk
├── <id>-json.log.2
└── ...                  ← up to max-file files; oldest is deleted when the cap is hit
```

**`journald` driver** — logs go into the systemd journal instead:

```
/var/log/journal/<machine-id>/*.journal   ← binary journal; read with:
journalctl CONTAINER_NAME=orders           ← query by container
```

**`syslog` / `fluentd` / remote drivers** — nothing meaningful is stored in `/var/lib/docker/containers/<id>/`; the shim streams each record over a socket (UDP/TCP/unix) to the collector. If the collector is down, behavior depends on mode (blocking vs non-blocking — see below).

**See the whole picture on your host:**

```
# What driver is the daemon defaulting to?
docker info --format '{{.LoggingDriver}}'        # e.g. json-file

# What driver + options does THIS container use?
docker inspect -f '{{.HostConfig.LogConfig.Type}} {{json .HostConfig.LogConfig.Config}}' orders
# json-file {"max-size":"10m","max-file":"3"}

# How big is the log on disk right now?
sudo du -h /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' orders)/*-json.log
```

Physical takeaways: with `json-file`, logs are real host files that grow with every line and survive until `docker rm`; with remote drivers, logs leave the host immediately and local files are minimal. Rotation is the only thing standing between a chatty app and a full disk.

---

## How it works — step by step

Trace one `console.log('order created', id)` from your app to `docker logs`:

1. **App writes.** Node executes `process.stdout.write("order created 991\n")`, which is the syscall `write(1, "order created 991\n", 18)`.
2. **Kernel routes to the pipe.** fd 1 points at the write end of the pipe Docker set up. The bytes sit in the kernel pipe buffer.
3. **The shim reads.** `containerd-shim` reads the bytes from the read end, splits on newlines into a log *record*, and attaches metadata: `time` (now, RFC3339) and `stream` (`stdout`).
4. **The record goes to the log driver.** The daemon's configured driver for this container handles it:
   - `json-file`: appends `{"log":"order created 991\n","stream":"stdout","time":"..."}\n` to `/var/lib/docker/containers/<id>/<id>-json.log`. If `max-size` is exceeded, it rotates: renames current → `.1`, opens a fresh file, deletes the oldest if over `max-file`.
   - `journald`: writes a structured entry into the systemd journal with fields like `CONTAINER_ID`, `CONTAINER_NAME`.
   - `syslog`/`fluentd`: serializes the record and sends it over the network to the collector.
5. **You read it.** `docker logs orders` asks the daemon, which reads the `json-file` (or, for `journald`, the journal) and prints `order created 991` with optional `--timestamps`. **Important:** `docker logs` only works with `json-file`, `journald`, and `local` drivers. With `syslog`/`fluentd`/`gelf`/`awslogs`, `docker logs` returns an error — the daemon no longer has the logs; they're at the remote collector.

**Blocking vs non-blocking delivery (step 4, the part that can freeze your app):** each driver has a `mode`:

```
mode=blocking (default for json-file)  → if the driver can't keep up, the write BLOCKS your app.
                                          Your console.log() stalls until the log is accepted.
mode=non-blocking + max-buffer-size    → logs buffer in memory; if the buffer fills, NEW logs are
                                          DROPPED rather than blocking the app.
```

This matters: with a slow remote driver in blocking mode, a logging backlog can literally stall your request handlers. Production remote logging usually uses `mode=non-blocking` with a bounded buffer, accepting occasional dropped lines over a stalled API.

---

## Exact syntax breakdown

### `docker logs`

```
docker logs --follow --tail 100 --timestamps --since 10m orders
            │        │           │            │        │
            │        │           │            │        └─ container name/ID
            │        │           │            └─ only lines from the last 10 minutes (or an RFC3339 time)
            │        │           └─ prepend each line with its captured timestamp
            │        └─ show only the last 100 lines (then follow if -f)
            └─ -f: stream new lines live, like `tail -f`
```

### Set the driver + rotation on one container

```
docker run -d --name orders \
  --log-driver json-file \
  │           │
  │           └─ which log driver to use for THIS container
  └─ override the daemon default for this container
  --log-opt max-size=10m \
  │         │
  │         └─ rotate the active log file once it reaches 10 MB
  └─ pass an option to the driver (repeatable)
  --log-opt max-file=3 \
  │         │
  │         └─ keep at most 3 rotated files (10m × 3 = 30 MB cap per container)
  └─ another driver option
  orders-api:1.0
```

### Set the daemon-wide default (applies to all new containers)

```
/etc/docker/daemon.json
{
  "log-driver": "json-file",          ← default driver for every new container
  "log-opts": {
    "max-size": "10m",                ← default rotation size
    "max-file": "3",                  ← default number of rotated files
    "compress": "true"                ← gzip rotated files (.1.gz) to save space
  }
}
```

```
sudo systemctl restart docker   # reload daemon.json; affects NEW containers only, not running ones
```

### `syslog` driver

```
docker run -d --name orders \
  --log-driver syslog \
  --log-opt syslog-address=udp://logs.internal:514 \
  │         │              │   │                │
  │         │              │   │                └─ port of the syslog collector
  │         │              │   └─ the collector host
  │         │              └─ transport: udp:// | tcp:// | unix://
  │         └─ where to send syslog messages
  --log-opt tag="{{.Name}}" \    # label each message with the container name
  orders-api:1.0
```

### `fluentd` driver (ship to a Fluentd/Fluent Bit aggregator)

```
docker run -d --name orders \
  --log-driver fluentd \
  --log-opt fluentd-address=127.0.0.1:24224 \
  │         │               │
  │         │               └─ address of the local Fluentd forwarder
  │         └─ Fluentd forward-protocol endpoint
  --log-opt fluentd-async=true \      # non-blocking: don't stall the app if Fluentd is slow
  --log-opt tag=orders.api \          # routing tag Fluentd matches on
  orders-api:1.0
```

### `journald` driver

```
docker run -d --name orders --log-driver journald orders-api:1.0
# read with the systemd journal, richer querying than docker logs:
journalctl CONTAINER_NAME=orders -f -o json-pretty
           │                     │  │
           │                     │  └─ output format (json-pretty | short | cat ...)
           │                     └─ follow live
           └─ filter journal entries to this container
```

---

## Example 1 — basic

Prove the stdout rule and read logs from a stopped container.

```
# GOOD: app logs to stdout → docker logs sees it
docker run -d --name good alpine sh -c 'i=0; while true; do echo "tick $i"; i=$((i+1)); sleep 1; done'
docker logs --tail 3 --timestamps good
# 2026-07-12T09:10:01Z tick 7
# 2026-07-12T09:10:02Z tick 8
# 2026-07-12T09:10:03Z tick 9

# See the raw json-file on disk (the physical reality)
sudo tail -1 /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' good)/*-json.log
# {"log":"tick 9\n","stream":"stdout","time":"2026-07-12T09:10:03.02Z"}

# Logs SURVIVE a stop — read them from an exited container
docker stop good
docker logs --tail 1 good          # still works: "tick N"
docker rm good                      # NOW the json.log file is deleted

# BAD: app logs to a FILE inside the container → docker logs is empty
docker run -d --name bad alpine sh -c 'while true; do echo "hidden" >> /var/log/app.log; sleep 1; done'
docker logs bad                     # (empty!) — bytes went to overlay2, never through the pipe
docker exec bad tail -1 /var/log/app.log   # the logs exist, but only INSIDE the container
docker rm -f bad
```

Lesson: only stdout/stderr reach `docker logs`; and `json-file` logs persist until `docker rm`.

---

## Example 2 — production scenario

**The situation:** Your `orders-api` runs on a shared host with Postgres and Redis. Two incidents hit in one week: (1) a debugging deploy logged every request body, wrote 8GB in a day, filled `/var/lib/docker`, and took down *all three* containers; (2) when investigating a failed order, you couldn't search logs because they were free-form strings.

**Fix part A — structured JSON logging in Node (with `pino`):**

```javascript
// logger.js — structured logs to STDOUT, one JSON object per line
const pino = require('pino');

// pino writes NDJSON to stdout by default — exactly what Docker's pipe captures.
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // Add fields present on EVERY log line so they're queryable downstream.
  base: { service: 'orders-api', env: process.env.NODE_ENV },
});

module.exports = logger;
```

```javascript
// server.js
const express = require('express');
const logger = require('./logger');
const app = express();

app.post('/orders', async (req, res) => {
  const orderId = 991;
  // Structured: every field is independently searchable in Datadog/Loki/CloudWatch.
  logger.info({ orderId, userId: req.get('x-user'), amount: 4200 }, 'order created');
  res.json({ id: orderId });
});

app.use((err, req, res, next) => {
  // Errors go to stdout too (pino writes error level to stdout as JSON, not stderr, by default).
  logger.error({ err, path: req.path }, 'request failed');
  res.status(500).json({ error: 'internal' });
});

app.listen(3000, () => logger.info('orders-api up'));
```

A log line now looks like:

```
{"level":30,"time":1752312082140,"service":"orders-api","env":"production","orderId":991,"userId":"u_42","amount":4200,"msg":"order created"}
```

Now "show all logs where `orderId=991`" is a one-field query. That's the entire point of structured logging.

**Fix part B — bound the disk with rotation, host-wide.** Set it in `/etc/docker/daemon.json` so *every* container is capped and one chatty app can never fill the disk again:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "5", "compress": "true" }
}
```

```
sudo systemctl restart docker
# Each container now caps at 10m × 5 = 50MB (rotated files gzipped). Disk can't run away.
```

**Fix part C — ship logs off the host so they survive container death.** In real production you don't `docker logs` into each box; you run a log shipper (Fluent Bit) that tails the `json-file` files and forwards to a central store (Loki/Elasticsearch/CloudWatch). Two common shapes:

```
Option 1 — driver ships directly:
  orders-api  --log-driver fluentd --log-opt fluentd-async=true  → Fluentd → Loki
  (docker logs no longer works locally; logs live centrally)

Option 2 — keep json-file locally AND tail-ship (most common, most resilient):
  orders-api  --log-driver json-file (rotated)   → files on host
  fluent-bit (a DaemonSet-like agent) tails /var/lib/docker/containers/*/*-json.log → Loki
  (docker logs STILL works locally; central store has the durable copy)
```

Option 2 is usually preferred: local `docker logs` still works for quick debugging, and the central copy survives `docker rm` and node loss. Use `fluentd-async=true` (or `mode=non-blocking`) so a slow collector never stalls your request handlers (recall blocking mode from the step-by-step).

---

## Common mistakes

### Mistake 1 — Logging to a file inside the container

**Symptom:** `docker logs orders` is empty even though the app clearly works.

**Root cause:** the app writes to `/app/logs/app.log`, which lands in the overlay2 writable layer, not through the stdout pipe the shim reads. **Right:** log to stdout/stderr (`console.log`, `pino` to stdout). In containers, the platform captures and ships logs — the app should not manage log files itself. If a library insists on a file path, point it at `/dev/stdout`.

### Mistake 2 — No rotation on `json-file` → full disk

**The error others on the host start seeing:**

```
Error response from daemon: write /var/lib/docker/containers/<id>/<id>-json.log: no space left on device
```

**Root cause:** the default `json-file` driver has **no rotation by default**. One noisy container grows its log until `/var/lib/docker` is full, and then *every* container fails to write. **Right:** always set `max-size` and `max-file` (per container or, better, in `daemon.json` host-wide). Find the culprit with:

```
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail
```

### Mistake 3 — Expecting `docker logs` to work with a remote driver

**The error:**

```
docker logs orders
# Error response from daemon: configured logging driver does not support reading
```

**Root cause:** you set `--log-driver syslog` (or `fluentd`/`gelf`/`awslogs`). Those ship logs away; the daemon keeps no local copy to read. **Right:** read them at the collector (Kibana/Grafana/CloudWatch), or use the `local`/`json-file`/`journald` driver locally, or use the tail-ship pattern (Option 2 above) that keeps a local copy.

### Mistake 4 — Multi-line stack traces become many log entries

**Symptom:** a Node stack trace appears in your log store as 15 separate, unlinked log entries, breaking your error grouping.

**Root cause:** the shim splits on newlines; each `\n` in a stack trace is a separate record. **Right:** emit errors as **structured single-line JSON** — with `pino`, `logger.error({ err }, 'msg')` serializes the whole error (including stack) into one JSON object on one line. One error = one searchable record.

### Mistake 5 — Buffered stdout hides logs until crash

**Symptom:** logs are missing right up until the container dies, then appear late or not at all.

**Root cause:** if stdout is block-buffered (common when it's a pipe, not a TTY) and the process is killed with SIGKILL, the buffer is lost. **Right:** most Node logging is line-buffered/flushed on write, but avoid custom buffering; for other runtimes set unbuffered mode (e.g. `PYTHONUNBUFFERED=1`). Also ensure graceful shutdown (Topic 16) so buffers flush on SIGTERM.

---

## Hands-on proof

```
# 1) Prove the stdout→pipe→json-file path physically
docker run -d --name lg alpine sh -c 'echo hello-stdout; echo hello-stderr 1>&2; sleep 300'
docker logs lg                         # shows both lines
sudo cat /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' lg)/*-json.log
#   {"log":"hello-stdout\n","stream":"stdout",...}
#   {"log":"hello-stderr\n","stream":"stderr",...}   ← note the stream field differs
docker rm -f lg

# 2) Prove rotation caps disk usage
docker run -d --name spin --log-opt max-size=1m --log-opt max-file=2 \
  alpine sh -c 'while true; do head -c 20000 /dev/urandom | base64; done'
sleep 5
ls -lh /var/lib/docker/containers/$(docker inspect -f '{{.Id}}' spin)/ | grep json.log
#   -json.log      (≤1m)     ← never exceeds max-size
#   -json.log.1    (≤1m)     ← at most max-file files; total capped at ~2m
docker rm -f spin

# 3) Prove which driver a container uses
docker run -d --name jf alpine sleep 60
docker inspect -f '{{.HostConfig.LogConfig.Type}} {{json .HostConfig.LogConfig.Config}}' jf
docker rm -f jf

# 4) Prove docker logs works after stop, dies after rm
docker run --name once alpine echo "run once"
docker logs once            # "run once" — container already exited, log still here
docker rm once
docker logs once 2>&1       # Error: No such container — log deleted with the container

# 5) --since / --tail filtering
docker run -d --name t alpine sh -c 'i=0; while true; do echo line $i; i=$((i+1)); sleep 1; done'
sleep 5; docker logs --since 2s --timestamps t   # only the last ~2 seconds of lines
docker rm -f t
```

---

## Practice exercises

### Exercise 1 — easy

Run a container that prints one line to stdout and one to stderr. Use `docker logs` to view both, then read the raw `-json.log` on the host and identify the `stream` field for each line. Explain in one sentence why `console.log` and `console.error` end up with different `stream` values.

### Exercise 2 — medium

Configure a container with `--log-opt max-size=512k --log-opt max-file=3` running a loop that prints continuously. After a minute, list the log files in its container directory and compute the maximum disk the logs can ever occupy. Then move the same settings into `/etc/docker/daemon.json`, restart Docker, launch a *new* container with no `--log-opt`, and prove via `docker inspect` that it inherited the defaults.

### Exercise 3 — hard (production simulation)

Wire up local structured logging + shipping. (a) Add `pino` to the `orders-api` from Example 2 and emit a structured `order created` log with `orderId`, `userId`, and `amount`. (b) Run a `fluent/fluent-bit` container configured to tail `/var/lib/docker/containers/*/*-json.log` (bind-mount that directory read-only) and print parsed records to its own stdout. (c) Trigger a few orders, then show the JSON records arriving in Fluent Bit's output. (d) Kill and `docker rm` the orders-api container and confirm the records already shipped to Fluent Bit are still there — explain why this proves the value of shipping logs off the host.

---

## Mental model checkpoint

1. What kernel primitive connects your process's stdout to Docker, and what are fds 1 and 2?
2. Why does logging to a file inside the container make `docker logs` empty?
3. Where exactly does the default `json-file` driver write, and what's the shape of one line?
4. What two `--log-opt` settings bound disk usage, and how do you compute the cap?
5. Which drivers support `docker logs`, and why do `syslog`/`fluentd` not?
6. What's the difference between blocking and non-blocking log delivery, and why does it matter for latency?
7. Why is structured JSON logging better than free-form strings in production?

---

## Quick reference card

| Command / option | What it does | Key detail |
|---|---|---|
| `docker logs <c>` | Print a container's captured stdout/stderr | Works on stopped containers; only json-file/journald/local |
| `-f` / `--follow` | Stream new lines live | Like `tail -f` |
| `--tail N` | Show only the last N lines | Combine with `-f` |
| `--since` / `--until` | Time-window filter | RFC3339 or relative (`10m`) |
| `--timestamps` | Prepend the captured time | Time is what the shim attached |
| `--log-driver` | Choose the backend | json-file, local, journald, syslog, fluentd, gelf, awslogs |
| `--log-opt max-size` | Rotate at this size | e.g. `10m` |
| `--log-opt max-file` | Keep N rotated files | Total cap = max-size × max-file |
| `--log-opt compress=true` | gzip rotated files | Saves disk |
| `--log-opt mode=non-blocking` | Buffer instead of stall | Drops logs if buffer fills; protects latency |
| `daemon.json` `log-driver`/`log-opts` | Host-wide defaults | Applies to NEW containers after daemon restart |
| `journalctl CONTAINER_NAME=` | Read journald logs | Richer querying than docker logs |
| `<id>-json.log` | The on-disk log file | In `/var/lib/docker/containers/<id>/` |

---

## When would I use this at work?

1. **Debugging a production 500.** An order fails; you `docker logs --since 5m --timestamps orders` (or query Loki) and, because you log structured JSON, filter to `orderId=991` to see the exact error, the DB query that failed, and the user affected — in seconds.
2. **Preventing a full-disk outage.** Before shipping any service to a shared host, you set `max-size`/`max-file` in `daemon.json` so no single chatty container can fill `/var/lib/docker` and take down every other container on the node.
3. **Building the logging pipeline.** You stand up Fluent Bit to tail every container's `json-file` and forward to a central store, so logs survive container death and node loss, and the whole team can search one place instead of SSH-ing into hosts — the direct precursor to the EFK/Loki stack you'll run in Kubernetes (Topic 51).

---

## Connected topics

- **Study before:** Topic 02 (Linux Foundations — file descriptors, pipes, overlay2 writable layer), Topic 03 (Docker Architecture — containerd-shim, which captures the pipes), Topic 15 (Environment Variables and Secrets — so you never log a secret), Topic 16 (Container Lifecycle — logs persist across exit; flush buffers on graceful shutdown).
- **Study after:** Topic 22 (Compose Commands — `docker compose logs` across services), Topic 25 (Resource Limits — logging pressure and disk), Topic 51 (Observability in Kubernetes — Prometheus/Loki/OTEL and instrumenting a Node app, the cluster-scale version of this topic).
