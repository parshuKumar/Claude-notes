# 01 — What Is a Container
## Section: Docker Foundations

---

## ELI5 — The Simple Analogy

Imagine a huge apartment building. Everyone shares the **same foundation, plumbing, and electrical wiring** (that's the building). But each apartment has its **own locked door, own kitchen, own furniture**. You can't walk into your neighbor's apartment. You can't hear them. As far as you're concerned, you're alone in your own home — even though 200 other families are living in the same building at the same time.

A **container** is an apartment. The **building** is the Linux kernel of one machine. Many containers live on one machine, sharing the same kernel (the building), but each one *thinks* it has the whole computer to itself. It has its own files, its own processes, its own network — but there's only one real building underneath.

Now compare that to a **virtual machine (VM)**. A VM is not an apartment. A VM is like building a **whole second house on top of your house** — its own foundation, its own plumbing, its own everything — just to have one more kitchen. It works, but it's enormously wasteful.

That difference — apartment vs. a whole second house — is the entire reason the industry moved to containers.

---

## The Linux kernel feature underneath

A container is **not a real thing**. There is no "container" object in the Linux kernel. This is the single most important sentence in this whole document, so read it twice.

A container is just a **normal Linux process** (like your `node server.js` process) that the kernel has been told to **lie to**. The kernel wraps that process in three illusions:

1. **Namespaces** — "You can only *see* your own stuff." (isolation of what's visible: processes, network, mounts, hostname, users)
2. **Cgroups** (control groups) — "You can only *use* this much." (limits on CPU, memory, disk I/O)
3. **Union/overlay filesystem** — "Here is *your own* root filesystem `/`." (a fake `/` built from stacked read-only image layers plus a writable layer)

That's it. Those three kernel features, wrapped around an ordinary process, *is* a container. When you run `docker run`, Docker asks the kernel to start a process with these three illusions switched on. When people say "Docker container," they mean "a process that the Linux kernel is fooling using namespaces + cgroups + overlayfs."

We go deep on all three in **Topic 02 — Linux Foundations of Containers**. For this topic, just hold onto the headline: *a container is a process wearing a costume the kernel sewed for it.*

---

## What is this?

A **container** is a single (or small group of) Linux process that runs isolated from the rest of the machine — with its own filesystem, its own network stack, and its own process list — while still sharing the host machine's kernel.

It is **not** a lightweight virtual machine. A VM emulates a whole computer including its own kernel/operating system. A container shares the host's one kernel and only isolates the *userspace* around a process.

---

## Why does it matter for a backend developer?

You wrote `orders-api`, a Node.js Express service backed by Postgres and Redis. It runs perfectly on your MacBook. You send it to a teammate and it explodes: `Error: Node version 18 required, found 20`. You deploy to the server and it explodes again: `libssl.so.1.1: cannot open shared object file`.

This is the **"works on my machine"** disease. The root cause: your app secretly depends on *dozens* of things that live on your machine but not on theirs — the Node version, system libraries, environment variables, the exact Linux distro, a specific `openssl`, a timezone file, a locale.

A container **packages the app together with its entire userspace world** — the right Node binary, the right libc, the right OpenSSL, the right files — into one image. Then that image runs *identically* on your laptop, your teammate's laptop, the CI runner, and the production server, because all of them just start the same process with the same fake filesystem.

Without understanding containers you will:
- Waste days on "works on my machine" bugs.
- Not understand why your `orders-api` container was `OOMKilled` (a cgroup memory limit — Topic 25).
- Not understand why one container can't see another container's Postgres (network namespaces — Topic 13).
- Think Docker is "a lightweight VM" and make wrong decisions about security, performance, and networking forever.

---

## The physical reality

Let's kill the magic. When your `orders-api` container is running on a Linux host, here is what **actually physically exists**:

**1. A real process in the host's process table.** Run `ps` on the host and you'll literally see your Node process:

```
$ ps aux | grep node
root  84213  0.4  1.2  node /app/server.js     ← this IS the container
```

`84213` is a normal PID on the host. There is no separate "container" process hiding somewhere. The container *is* PID 84213 (plus any children it forks).

**2. Namespace files under `/proc`.** Every process has a `/proc/<PID>/ns/` directory listing the namespaces it belongs to:

```
$ ls -l /proc/84213/ns/
lrwxrwxrwx  cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx  ipc    -> 'ipc:[4026532567]'    ← its own IPC namespace
lrwxrwxrwx  mnt    -> 'mnt:[4026532565]'    ← its own mount (filesystem) namespace
lrwxrwxrwx  net    -> 'net:[4026532570]'    ← its own network namespace
lrwxrwxrwx  pid    -> 'pid:[4026532568]'    ← its own PID namespace
lrwxrwxrwx  user   -> 'user:[4026531837]'
lrwxrwxrwx  uts    -> 'uts:[4026532566]'    ← its own hostname namespace
```

Those numbers in brackets are namespace IDs. A different container will have *different* numbers for `net`, `mnt`, `pid`, etc. — proving they're isolated. That's the whole illusion, written down as symlinks in `/proc`.

**3. A cgroup directory controlling its limits.** On a cgroup v2 host:

```
$ cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.max
536870912          ← 512 MiB memory limit for orders-api, in bytes
```

**4. A stacked filesystem under `/var/lib/docker/`.** The container's fake `/` is assembled from image layers here:

```
/var/lib/docker/overlay2/<hash>/diff/     ← this layer's files
/var/lib/docker/overlay2/<hash>/merged/   ← the final combined /  the container sees
```

So a "container" on disk and in memory is: **one process + seven namespace symlinks + one cgroup directory + one stacked filesystem.** Nothing more mystical than that.

---

## How it works — step by step

Here is the full trace of what happens, at the kernel level, when you start `orders-api` as a container. (Topic 03 covers *which Docker component* does each step; here we focus on *what the kernel does*.)

1. **A parent process (`runc`) prepares the environment.** Docker eventually calls a low-level tool called `runc` (Topic 03). `runc` reads a config describing the desired namespaces, cgroups, and root filesystem.

2. **`runc` calls `clone()` / `unshare()` with namespace flags.** The kernel `clone()` syscall accepts flags like `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWNS`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWUSER`. Each flag tells the kernel: "give the new child process a **brand-new, empty** version of this namespace." The moment `clone()` returns, a new process exists that can no longer see the host's processes, network, or mounts.

3. **The kernel creates fresh namespaces.** For `CLONE_NEWPID`, the new process becomes **PID 1 inside its own PID namespace** (even though it's PID 84213 on the host). For `CLONE_NEWNET`, it gets an empty network stack with just a loopback — no eth0 yet.

4. **The process is placed into a cgroup.** `runc` writes the new process's PID into `/sys/fs/cgroup/.../cgroup.procs` and writes limits into `memory.max`, `cpu.max`, etc. From now on the kernel accounts every page of memory and every CPU slice this process uses, and kills it if it exceeds `memory.max`.

5. **`pivot_root` swaps the root filesystem.** Inside the new mount namespace, `runc` calls `pivot_root()` to make the overlay-mounted `merged/` directory the new `/`. Now when the process opens `/etc/hostname` or `/usr/bin/node`, it reads files from the **image**, not from the host. The host's real `/` has vanished from the process's view.

6. **Network plumbing is attached.** A virtual ethernet pair (`veth`) is created: one end inside the container's net namespace (appears as `eth0`), one end on the host attached to the `docker0` bridge (Topic 13). Now the container can talk to Postgres and Redis.

7. **`execve()` runs your actual program.** Finally `runc` calls `execve("/usr/local/bin/node", ["node", "/app/server.js"], envp)`. The child process **becomes** your Node app, keeping all the namespaces, cgroup, and root filesystem it was set up with.

8. **Your app runs, fully fooled.** `orders-api` starts, thinks it's PID 1 on a fresh machine called by its hostname, sees only `/app` and its dependencies, can use at most 512 MiB, and can reach Postgres/Redis over its private network. It has no idea 40 other containers are running two millimeters away on the same kernel.

The key insight: **there was never a moment where a "container" was created.** There was only a process, and a sequence of syscalls that fooled it.

---

## Exact syntax breakdown

The command that starts a container:

```
docker run --rm -p 3000:3000 --memory 512m --name orders-api node:20-alpine node server.js
```

```
docker run --rm -p 3000:3000 --memory 512m --name orders-api node:20-alpine node server.js
│      │    │    │            │              │             │              │
│      │    │    │            │              │             │              └─ command to run INSIDE the container:
│      │    │    │            │              │             │                 "node server.js" — becomes PID 1 in the container
│      │    │    │            │              │             │
│      │    │    │            │              │             └─ the IMAGE: the read-only stack of layers that
│      │    │    │            │              │                becomes the container's fake root filesystem /
│      │    │    │            │              │
│      │    │    │            │              └─ --name: a human label for this container (else Docker random-names it)
│      │    │    │            │
│      │    │    │            └─ --memory 512m: write 536870912 into the cgroup's memory.max.
│      │    │    │               Exceed it → kernel OOM-kills the process (Topic 25)
│      │    │    │
│      │    │    └─ -p 3000:3000: publish container port 3000 to host port 3000.
│      │    │       Sets up an iptables DNAT rule so host traffic reaches the container's net namespace (Topic 13)
│      │    │
│      │    └─ --rm: delete the container's writable layer + metadata when the process exits
│      │
│      └─ run: create namespaces + cgroup + rootfs, then start the process (the 8 steps above)
│
└─ docker: the CLI that sends this request to the Docker daemon (Topic 03)
```

Peeking at the namespaces of a running container:

```
lsns -p 84213
│    │  │
│    │  └─ -p 84213: show namespaces for process with PID 84213 (our orders-api)
│    │
│    └─ (list namespaces)
│
└─ lsns: a Linux tool that reads /proc/*/ns/ and prints which namespaces exist and who is in them
```

---

## Example 1 — basic

Run your `orders-api` and prove it's isolated but sharing the kernel.

```bash
# Start a simple container running a shell, isolated like orders-api would be.
docker run --rm -it --name demo alpine sh
#             │   │  │       │     │    │
#             │   │  │       │     │    └─ run the shell "sh" as PID 1 inside
#             │   │  │       │     └─ image: tiny Alpine Linux rootfs (~7 MB)
#             │   │  │       └─ name it "demo"
#             │   │  └─ -t: give it a terminal (TTY)
#             │   └─ -i: keep stdin open so we can type
#             └─ --rm: clean up on exit
```

Now, **inside** the container:

```sh
/ # ps aux
PID   USER     COMMAND
    1 root     sh              ← WE are PID 1. The host's systemd/init is invisible.
    7 root     ps aux          ← only OUR processes exist here
/ # hostname
3f9a2b1c4d5e                   ← our own hostname (UTS namespace), not the host's
/ # cat /etc/os-release
NAME="Alpine Linux"            ← we see Alpine's files, even if the host is Ubuntu
/ # ls /
bin  etc  lib  proc  root ...  ← our own tiny root filesystem, not the host's /
```

And **on the host**, in another terminal:

```bash
$ ps aux | grep '[s]h'
root  84213  ... sh            ← the SAME shell, but here it's PID 84213, not PID 1
$ uname -r
6.5.0-generic                  ← both host and container report the SAME kernel version
```

The `uname -r` proof is the punchline: the container reports the **same kernel** as the host, because there is only one kernel. That's what makes it a container and not a VM.

---

## Example 2 — production scenario

**The situation.** Your team runs `orders-api` (Node 20), a legacy `reports-api` (still on Node 14 because nobody dares upgrade it), and a `payments-worker` (Node 20 but needs a *specific* patched OpenSSL for a payment gateway). All three must run on **one** cloud VM to save money.

**Without containers**, this is a nightmare. You can only install one Node globally. You fight with `nvm`. The two OpenSSL versions clash. `reports-api` breaks whenever you patch the system for `payments-worker`. One machine, three apps, endless conflicts.

**With containers**, each app is an apartment:

```bash
docker run -d --name orders-api      --memory 512m orders-api:1.4.0        # Node 20 baked in
docker run -d --name reports-api     --memory 256m reports-api:0.9.2       # Node 14 baked in
docker run -d --name payments-worker --memory 384m payments-worker:2.1.0   # patched OpenSSL baked in
#          │
#          └─ -d: detached — run in the background, don't attach our terminal
```

Now:
- Each container has its **own filesystem** (mnt namespace), so each carries its *own* Node binary and *own* OpenSSL. Node 14 and Node 20 coexist with zero conflict, because neither can even see the other's files.
- Each has its **own memory limit** (cgroup). If `reports-api` leaks memory, the kernel OOM-kills *only reports-api* at 256 MiB. `orders-api` and `payments-worker` keep serving customers, untouched.
- Each has its **own network namespace**, so all three can bind to port 3000 internally without colliding — Docker maps each to a different host port.
- They all **share the one kernel**, so they boot in ~50 ms and use almost no extra RAM. Three VMs would need three guest OSes and gigabytes of overhead.

**The payoff:** one cheap VM safely runs three apps with three incompatible dependency worlds, each limited and isolated, each restartable independently. That is *exactly* the problem containers were invented to solve, and it's why the industry moved.

---

## Common mistakes

**Mistake 1 — "A container is a lightweight VM."**

Wrong mental model that leads to real bugs. A VM has its **own kernel**; a container **shares the host kernel**.

Consequence you'll actually hit: you try to run a **Windows** container on a **Linux** host, or run a container that needs a newer kernel feature than the host has:

```
$ docker run mycontainer
standard_init_linux.go: exec user process caused: exec format error
```

Root cause: there is no second kernel to translate for you. The container's process runs *directly* on the host kernel, so the CPU architecture and kernel must be compatible. Right mental model: *shared kernel, isolated userspace.*

**Mistake 2 — Expecting data to survive `--rm`.**

```bash
docker run --rm -v /app/data postgres   # writes go to the writable layer
# ... container exits ...
# your data is GONE
```

Root cause: the container's writable layer lives in `/var/lib/docker/overlay2/<id>/diff` and `--rm` deletes it on exit. The overlay writable layer is **ephemeral** by design. Right way: put stateful data in a **volume** (Topic 14), not the container filesystem.

**Mistake 3 — Thinking the container "boots an OS."**

New folks say "the container is starting Alpine Linux." It isn't. There is **no init, no kernel boot, no systemd** by default. The container just `execve()`s your one process directly (step 7 above). That's why:

```
$ docker run alpine service nginx start
/bin/sh: service: not found
```

Root cause: there's no service manager because nothing "booted." A container runs *one foreground process*, not a full OS. Right way: run the program directly, e.g. `docker run nginx nginx -g 'daemon off;'`.

**Mistake 4 — Assuming isolation means security equal to a VM.**

Because containers **share the kernel**, a kernel exploit inside a container can potentially escape to the host — something a VM's separate kernel makes far harder. Root cause: one shared kernel = one shared attack surface. Right mindset: containers isolate for *convenience and resource control*; for hard security boundaries you add user namespaces, seccomp, and sometimes VM-based sandboxing (gVisor, Kata). Covered in Topic 23.

**Mistake 5 — `PID 1` confusion (zombie processes).**

Your Node app is PID 1 inside the container. PID 1 has a special kernel duty: **reap zombie child processes**. Node doesn't do this by default, so if `orders-api` spawns child processes, you can leak zombies:

```
$ ps aux                       # inside container
PID  STAT  COMMAND
  1  Ss    node server.js
 23  Z     [sh] <defunct>      ← zombie never reaped
```

Root cause: being PID 1 in a PID namespace comes with responsibilities your app wasn't written for. Right way: use `docker run --init` (adds a tiny init that reaps zombies) or `tini` (Topic 11).

---

## Hands-on proof

Run these **right now** on a Linux host (or `docker run` on macOS/Windows, which uses a Linux VM under the hood) and watch the illusion appear and disappear.

```bash
# 1. Start orders-api-like container in the background.
docker run -d --name proof --memory 256m alpine sleep 600

# 2. Find its REAL host PID. Proof: it's a normal host process.
PID=$(docker inspect --format '{{.State.Pid}}' proof)
echo "Host PID is $PID"

# 3. Look at its namespaces on the HOST. Proof: it has its own net/mnt/pid/uts/ipc.
sudo ls -l /proc/$PID/ns/

# 4. Compare to YOUR shell's namespaces. Proof: the IDs are DIFFERENT.
ls -l /proc/$$/ns/          # $$ = your current shell's PID

# 5. Show its cgroup memory limit. Proof: 256m became a real kernel limit.
cat /sys/fs/cgroup/system.slice/docker-*$(docker inspect --format '{{.Id}}' proof)*.scope/memory.max 2>/dev/null \
  || docker exec proof cat /sys/fs/cgroup/memory.max

# 6. Prove shared kernel: same version inside and out.
uname -r                    # host kernel
docker exec proof uname -r  # container "kernel" — IDENTICAL

# 7. Clean up.
docker rm -f proof
```

What you verified: a container is **one host process** (step 2), with **its own namespaces** that differ from yours (steps 3–4), a **real cgroup limit** (step 5), running on the **same shared kernel** (step 6).

---

## Practice exercises

### Exercise 1 — easy
Start two containers: `docker run -d --name a alpine sleep 300` and `docker run -d --name b alpine sleep 300`. Using `docker exec a hostname` and `docker exec b hostname`, show they have different hostnames. Then run `docker exec a ps aux` — how many processes does container `a` see, and why can't it see container `b`'s `sleep`? (Hint: PID namespace.)

### Exercise 2 — medium
Run `docker run -d --name mem --memory 128m alpine sh -c 'while true; do :; done'`. On the host, find its PID with `docker inspect --format '{{.State.Pid}}' mem`, then `cat /proc/<PID>/cgroup`. Locate its cgroup directory under `/sys/fs/cgroup/` and read `memory.max` and `memory.current`. Explain in one sentence what the kernel will do if `memory.current` ever tries to pass `memory.max`.

### Exercise 3 — hard (production simulation)
Simulate the Example 2 scenario. Build (or fake with `alpine`) three containers named `orders-api`, `reports-api`, `payments-worker`, each with `--memory` limits of 512m/256m/128m. Inside `reports-api`, run a memory-hog: `docker exec reports-api sh -c 'yes | tr \\n x | head -c 500m | grep n'` to try to allocate ~500 MB. Watch it get **OOMKilled** (`docker inspect reports-api --format '{{.State.OOMKilled}}'` → `true`) while `orders-api` and `payments-worker` keep running. Write down: which kernel feature killed it, and why the other two survived.

---

## Mental model checkpoint

Answer these from memory:

1. A container is fundamentally a ______ that the kernel is fooling with three features. Name the three.
2. What is the one thing a container shares with the host that a VM does *not* share?
3. Where on disk does a container's stacked root filesystem live, and where do its namespace links appear?
4. Why does `uname -r` return the same value inside a container and on the host?
5. Why is data in a `--rm` container lost on exit, at the filesystem level?
6. Why can two containers both bind port 3000 internally without conflict?
7. Give one reason a container is *less* of a security boundary than a VM.

---

## Quick reference card

| Command / concept | What it does | Key detail |
|---|---|---|
| `docker run IMAGE CMD` | Create namespaces + cgroup + rootfs, then run CMD as the container's PID 1 | The "container" is just that process |
| Namespaces | Isolate what a process can *see* (pid, net, mnt, uts, ipc, user) | Symlinks in `/proc/<PID>/ns/` |
| Cgroups | Limit what a process can *use* (cpu, memory, io) | Files under `/sys/fs/cgroup/` |
| Overlay filesystem | Give the process a fake `/` from stacked layers | Under `/var/lib/docker/overlay2/` |
| `docker inspect --format '{{.State.Pid}}'` | Reveal the container's real host PID | Proves it's a normal process |
| `lsns -p <PID>` | List a process's namespaces | Reads `/proc/<PID>/ns/` |
| Container vs VM | Shared kernel vs own kernel | The single defining difference |
| `--rm` | Delete writable layer + metadata on exit | Data not in a volume is lost |

---

## When would I use this at work?

1. **Diagnosing "works on my machine."** When `orders-api` runs locally but crashes in CI, you now know the container packages its own userspace — so you check *which base image / Node version* the image was built on, not the CI host's Node.

2. **Explaining an OOMKill in a postmortem.** When ops says "orders-api got OOMKilled at 512 MiB but the VM had 16 GB free," you can explain it was the *cgroup* limit (a per-container kernel limit), not the host running out of memory.

3. **Right-sizing infrastructure.** When deciding "do we spin up 5 VMs or run 5 containers on one VM," you understand the tradeoff: containers share one kernel (cheap, dense, weaker isolation) vs VMs bring their own kernel (expensive, heavy, stronger isolation).

---

## Connected topics

- **Study before:** none — this is the entry point. Basic Linux CLI familiarity is assumed.
- **Study after:**
  - **Topic 02 — Linux Foundations of Containers**: the deep dive into the three kernel features (all 6 namespace types, cgroups v2, overlayfs) that this topic introduced.
  - **Topic 03 — Docker Architecture**: which component (CLI → dockerd → containerd → runc) actually performs the 8 steps above.
  - **Topic 13 — Networking in Docker**: how the `veth`/bridge from step 6 lets `orders-api` reach Postgres and Redis.
  - **Topic 25 — Resource Limits**: the full story of cgroups, `OOMKilled`, and CPU throttling from the mistakes above.
