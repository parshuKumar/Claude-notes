# 01 — What Linux Actually Is

## ELI5 — The Simple Analogy

Imagine a restaurant.

- **The kitchen** is the kernel. It has direct access to all the raw ingredients (hardware — CPU, RAM, disk). No customer ever walks into the kitchen. The kitchen takes orders, manages limited resources (burners, ovens, counter space), and delivers finished dishes.
- **The waiters** are system calls. They are the ONLY way to communicate between the dining room and the kitchen. You can't yell at the chef directly — you tell the waiter what you want, and the waiter carries it back.
- **The dining room** is userspace. This is where all programs live — your Node.js app, your terminal, `ls`, `grep`, everything. They sit at tables, order through waiters, and never touch the stove.
- **The menu** is the shell. It's how you (the human) figure out what to order and communicate with the waiter. You could also write your order on a piece of paper (a script).
- **The restaurant chain** is the distribution (Ubuntu, Debian, Arch). Same kitchen equipment, same waiter protocol, but different menus, different décor, different default dishes.

Linux is the **kitchen** — the kernel. Everything else is built on top.

---

## Where This Lives in the Linux Stack

```
Hardware (CPU, RAM, Disk, NIC, GPU)
  └── KERNEL ◀◀◀ THIS IS LINUX. This is what Linus Torvalds wrote.
       │         Manages ALL hardware. Schedules ALL processes.
       │         Controls ALL memory. Enforces ALL permissions.
       │
       └── System Calls (≈400 functions — the ONLY door into the kernel)
            │
            └── C Standard Library (glibc/musl — wraps syscalls into friendly functions)
                 │
                 └── Shell (bash/zsh — interprets your typed commands)
                      │
                      └── Commands & Applications (ls, node, nginx — programs you run)
```

**This entire topic is about understanding these layers and how they relate.**

---

## What Is This?

Linux is an operating system **kernel** — a program that starts when your computer boots and manages everything between your software and your hardware. It is NOT an application you run. It IS the thing that lets all other applications run.

When people say "Linux" casually, they usually mean a **Linux distribution** — a complete operating system built by packaging the Linux kernel together with userspace tools (GNU utilities, package manager, init system, shell).

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| Kernel vs userspace boundary | You won't understand why your Node app can't access a file (permission denied happens in the kernel, not in Node) |
| What a distribution is | You'll be confused why commands differ between Ubuntu and Alpine Docker images |
| How the layers connect | You can't debug "why is my server slow" because you don't know where to look |
| What the shell actually does | You'll treat the terminal as magic instead of a predictable system |

---

## The Physical Reality

When a Linux server boots, here is what actually exists in memory:

```
┌─────────────────────────────────────────────────────┐
│                    RAM (Physical Memory)              │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │           KERNEL SPACE                       │     │
│  │  (Ring 0 — full hardware access)            │     │
│  │                                              │     │
│  │  • Process scheduler                         │     │
│  │  • Memory manager (virtual memory, paging)   │     │
│  │  • Filesystem drivers (ext4, xfs, etc.)      │     │
│  │  • Network stack (TCP/IP)                    │     │
│  │  • Device drivers (disk, NIC, USB)           │     │
│  │  • Security modules (SELinux, AppArmor)      │     │
│  └─────────────────────────────────────────────┘     │
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │           USER SPACE                         │     │
│  │  (Ring 3 — restricted, no direct HW access) │     │
│  │                                              │     │
│  │  • PID 1: init/systemd (first process)       │     │
│  │  • sshd (SSH server)                         │     │
│  │  • bash (your shell)                         │     │
│  │  • node server.js (your app)                 │     │
│  │  • nginx (reverse proxy)                     │     │
│  │  • postgres (database)                       │     │
│  └─────────────────────────────────────────────┘     │
│                                                       │
└─────────────────────────────────────────────────────┘
```

**Key fact:** Userspace programs CANNOT directly touch hardware. They must ask the kernel via system calls. This is enforced by the CPU itself (Ring 0 vs Ring 3 protection).

---

## How It Works — Step by Step

### What happens when you type `node server.js` on a Linux server:

```
Step 1: YOU type "node server.js" and press Enter

Step 2: SHELL (bash) reads the string "node server.js"
        - Parses it: command = "node", argument = "server.js"
        - Searches PATH for the "node" binary
        - Finds it at /usr/bin/node

Step 3: SHELL calls fork() system call
        - Kernel creates an exact copy of the shell process
        - Now there are TWO bash processes (parent and child)

Step 4: CHILD process calls execve("/usr/bin/node", ["node", "server.js"], env)
        - Kernel replaces the child's memory with the Node.js binary
        - The child is no longer bash — it IS node now

Step 5: KERNEL sets up the new process:
        - Assigns a PID (process ID)
        - Allocates virtual memory
        - Opens file descriptors 0 (stdin), 1 (stdout), 2 (stderr)
        - Adds it to the scheduler queue

Step 6: NODE starts executing
        - Reads server.js (open() syscall → kernel reads from disk)
        - Binds to port 3000 (socket() + bind() syscalls → kernel manages network)
        - Listens for connections (listen() syscall)

Step 7: PARENT (bash) waits for the child to finish
        - Actually calls waitpid() syscall
        - Shell prompt doesn't return until node exits (unless you use &)
```

### The system call boundary in action:

```
Your Node.js code:          fs.readFileSync('/etc/hostname')
                                     │
                                     ▼
Node.js internals:          calls libuv → calls C read() function
                                     │
                                     ▼
C Library (glibc):          packages it into a read() syscall
                                     │
                                     ▼
═══════════════════ SYSCALL BOUNDARY (Ring 3 → Ring 0) ═══════════════════
                                     │
                                     ▼
Kernel:                     • Checks: does this process have permission?
                            • Finds the file's inode
                            • Reads data blocks from disk (or cache)
                            • Copies data to process's memory buffer
                                     │
                                     ▼
═══════════════════ RETURN TO USERSPACE (Ring 0 → Ring 3) ═══════════════
                                     │
                                     ▼
Your Node.js code:          receives the string "my-server-hostname\n"
```

---

## The Kernel — What It Actually Does

The kernel has exactly **5 core jobs**:

### 1. Process Management
- Creates, schedules, and destroys processes
- Decides which process runs on which CPU core, and for how long
- Every program you run is a process managed by the kernel

### 2. Memory Management
- Gives each process its own virtual address space (the process thinks it has all the RAM)
- Maps virtual addresses to physical RAM pages
- Handles swap (moving memory to disk when RAM is full)
- This is why one process can't read another process's memory

### 3. Filesystem Management
- Provides the `/` directory tree abstraction
- Translates "read file /var/log/app.log" into disk block operations
- Manages different filesystem types (ext4, xfs, tmpfs, procfs)
- Enforces file permissions

### 4. Device Management
- Talks to hardware through drivers (disk controllers, network cards, etc.)
- Exposes devices as files in `/dev` (everything is a file!)
- Manages interrupts from hardware

### 5. Network Management
- Implements the entire TCP/IP stack
- Manages sockets, connections, routing
- When Node.js calls `http.createServer()`, the kernel does the real work

---

## Distributions — What They Are and Why They Matter

A **distribution** = Linux kernel + choices about everything else:

```
┌──────────────────────────────────────────────────────────────┐
│                    Linux Distribution                          │
├──────────────────────────────────────────────────────────────┤
│  Component          │ Ubuntu 22.04    │ Alpine 3.18          │
├─────────────────────┼─────────────────┼──────────────────────┤
│  Kernel             │ Linux 5.15      │ Linux 6.1            │
│  C Library          │ glibc           │ musl (smaller)       │
│  Package Manager    │ apt (dpkg)      │ apk                  │
│  Init System        │ systemd         │ OpenRC               │
│  Shell              │ bash            │ ash (BusyBox)        │
│  Core Utilities     │ GNU coreutils   │ BusyBox              │
│  Default size       │ ~700MB          │ ~5MB                 │
└─────────────────────┴─────────────────┴──────────────────────┘
```

**Why this matters to you:**
- Your Docker images probably use Alpine (small) or Debian (compatible)
- Commands may behave slightly differently (`sed` on Alpine vs Ubuntu)
- Package names differ between distributions
- The kernel underneath is always Linux — same syscalls, same fundamentals

---

## The Shell — Your Interface to Everything

The shell is just a program. It is NOT the kernel. It is NOT the operating system. It's a userspace program like any other, whose job is:

1. Print a prompt (`$`)
2. Read your input
3. Parse it into command + arguments
4. Find the program on disk
5. Ask the kernel to run it (fork + exec)
6. Wait for it to finish
7. Go back to step 1

```
The Shell Loop (simplified):

while (true) {
    print_prompt();              // Display "user@host:~$ "
    input = read_line();         // Read what you typed
    args = parse(input);         // Split into command + arguments
    
    if (is_builtin(args[0])) {  // cd, export, etc. — shell handles these itself
        run_builtin(args);
    } else {
        pid = fork();           // Create child process
        if (pid == 0) {         // In child:
            execvp(args[0], args);  // Replace child with the command
        } else {                // In parent:
            waitpid(pid);       // Wait for child to finish
        }
    }
}
```

---

## Exact Syntax Breakdown

```
uname -a
│      │
│      └── -a = --all — print ALL system information
└── uname = "Unix Name" — prints system information

Output: Linux myserver 5.15.0-72-generic #79-Ubuntu SMP x86_64 GNU/Linux
        │     │        │                  │           │       │
        │     │        │                  │           │       └── OS name
        │     │        │                  │           └── CPU architecture
        │     │        │                  └── kernel build info
        │     │        └── kernel version (5.15.0, patch 72, generic config)
        │     └── hostname
        └── kernel name (always "Linux" on Linux)
```

```
cat /etc/os-release
│   │
│   └── file containing distribution identity info
└── concatenate and print file contents

Output includes:
  NAME="Ubuntu"
  VERSION="22.04.2 LTS (Jammy Jellyfish)"
  ID=ubuntu
  ID_LIKE=debian
```

---

## Example 1 — Basic: Identify Your System

```bash
# What kernel are you running?
uname -r
# Output: 5.15.0-72-generic

# What distribution is this?
cat /etc/os-release
# Output: NAME="Ubuntu", VERSION="22.04" ...

# How long has this system been running?
uptime
# Output: 14:23:07 up 42 days, 3:15, 2 users, load average: 0.15, 0.10, 0.08

# What CPU does the kernel see?
lscpu | head -15
# Shows: Architecture, CPU(s), Model name, MHz

# How much RAM does the kernel manage?
free -h
# Output: total 7.8G, used 3.2G, free 1.1G, available 4.3G

# What processes is the kernel running right now?
ps aux | wc -l
# Output: 187 (number of running processes)
```

---

## Example 2 — Production Scenario

**Situation:** You SSH into your production server. Your Node.js API is returning 500 errors. You need to understand the system before debugging.

```bash
# Step 1: What system am I on? (Matters for which commands are available)
cat /etc/os-release | grep -E "^(NAME|VERSION)="
# NAME="Ubuntu"
# VERSION="22.04.2 LTS (Jammy Jellyfish)"

# Step 2: How long has it been running? (Did it recently reboot unexpectedly?)
uptime
# 14:23:07 up 42 days, 3:15, 2 users, load average: 12.50, 8.30, 4.15
#                                                     ^^^^^ ← HIGH! Something is wrong

# Step 3: Is the system overloaded? (Load average > number of CPUs = overloaded)
nproc
# 4  ← we have 4 CPU cores, but load average is 12.5 — 3x overloaded!

# Step 4: What is the kernel struggling with?
dmesg | tail -20
# Shows kernel messages — look for OOM (Out Of Memory) kills, disk errors

# Step 5: Is the kernel killing processes due to low memory?
dmesg | grep -i "killed process"
# [423842.123] Out of memory: Killed process 8234 (node) total-vm:2048000kB
# ← The kernel's OOM killer terminated your Node.js process!

# Step 6: Check available memory right now
free -h
#               total   used   free   shared  buff/cache  available
# Mem:          7.8G    7.5G   52M    124M    312M        180M
# ← Almost no memory left. That's why the kernel killed node.
```

**What you just did:** Used your understanding of the kernel's job (memory management, process management) to diagnose WHY your app crashed. The kernel killed it because it ran out of memory to manage.

---

## Common Mistakes

### Mistake 1: "Linux is Ubuntu"

**Wrong:** "I'm running Linux Ubuntu"
**Right:** "I'm running the Ubuntu distribution, which uses the Linux kernel"

**Why it matters:** When your Docker container uses `FROM alpine:3.18` and something breaks, you need to know that Alpine uses musl instead of glibc, and BusyBox instead of GNU coreutils. The KERNEL is the same — but everything around it differs.

---

### Mistake 2: "The terminal IS the operating system"

**Wrong:** Thinking that closing the terminal shuts down Linux
**Right:** The terminal is just one userspace program. The kernel runs hundreds of other processes.

**Root cause:** The shell is a process like any other. `ps aux` will show you hundreds of processes that have nothing to do with your terminal. The kernel manages them all.

---

### Mistake 3: "My Node app accesses files directly"

**Wrong:** `fs.readFile()` reads directly from disk
**Right:** `fs.readFile()` → libuv → read() syscall → kernel checks permissions → kernel reads disk → returns data to your process

**Why it matters:** When you get `EACCES: permission denied`, the error comes from the KERNEL, not from Node. The kernel checked the file's permission bits against your process's UID and said "no." Understanding this means you know to fix permissions, not your code.

---

### Mistake 4: "All Linux servers are the same"

**Wrong:** A command that works on Ubuntu will work on Alpine
**Right:** Kernel syscalls are the same, but userspace tools differ between distros

```bash
# Ubuntu (GNU sed):
sed -i 's/old/new/' file.txt

# Alpine/macOS (BSD sed) — needs empty string after -i:
sed -i '' 's/old/new/' file.txt
```

---

### Mistake 5: "Root can do anything"

**Wrong:** The root user bypasses everything
**Right:** Root bypasses PERMISSION CHECKS, but the kernel still enforces other rules (you can't write to a read-only filesystem, you can't exceed physical RAM, you can't use a network interface that doesn't exist)

---

## Hands-On Proof

```bash
# PROVE IT: The kernel is a specific file on disk
ls -la /boot/vmlinuz*
# You'll see the actual kernel binary file. This gets loaded into RAM at boot.

# PROVE IT: See what kernel version you're running
uname -r
cat /proc/version
# Both show the kernel version — /proc/version is the kernel reporting about itself

# PROVE IT: The kernel exposes itself through /proc
ls /proc | head -20
# Numbers = PIDs of running processes. The kernel creates this "virtual filesystem"

# PROVE IT: See a system call happen in real time
strace echo "hello" 2>&1 | head -5
# strace shows EVERY system call a program makes
# You'll see execve(), then write(1, "hello\n", 6) — that's echo asking the kernel
# to write "hello\n" (6 bytes) to file descriptor 1 (stdout)

# PROVE IT: The kernel manages all memory
cat /proc/meminfo | head -5
# The kernel tracks every byte of RAM

# PROVE IT: See the boundary between kernel space and user space
cat /proc/1/maps | head -10
# Shows the virtual memory map of PID 1 (init/systemd)
# The kernel assigns each process its own memory layout

# PROVE IT: The shell is just a process, nothing special
echo $$
# Shows YOUR shell's PID — it's just a process with a number
ps -p $$ -o comm=
# Shows "bash" or "zsh" — just a program name
```

---

## Practice Exercises

### Exercise 1 — Easy

Run the following commands and answer:

```bash
uname -r
uname -a
cat /etc/os-release
uptime
nproc
free -h
```

**Question:** What kernel version are you running? What distribution? How many CPU cores does the kernel see? How much total RAM?

Paste your terminal output.

---

### Exercise 2 — Medium

Use `strace` to observe the kernel at work:

```bash
strace -c ls /tmp
```

The `-c` flag gives a SUMMARY of all system calls made.

**Question:** What were the top 3 most-called system calls? What do you think each one does? (Guess based on the name — we'll learn them properly later.)

Paste your terminal output.

---

### Exercise 3 — Hard (Production Simulation)

You are the sysadmin. Get a complete system report:

```bash
# Create a system report with:
# 1. Kernel version
# 2. Distribution name and version
# 3. CPU count
# 4. Total RAM
# 5. System uptime
# 6. Number of running processes
# 7. Disk usage of / (root filesystem)

# Write ONE compound command using && that prints all of this
# Example start: echo "=== KERNEL ===" && uname -r && ...
```

Build the full command. Paste your terminal output.

---

## Mental Model Checkpoint

Answer these from memory. If you can't, re-read the relevant section.

1. **What are the 5 core jobs of the Linux kernel?**
2. **What is the ONLY way a userspace program can ask the kernel to do something?**
3. **What happens at the CPU hardware level to prevent userspace programs from touching hardware directly?**
4. **When you type `node server.js`, what are the two system calls the shell uses to create the new process?**
5. **What is the difference between the Linux kernel and a Linux distribution?**
6. **If your Node.js app gets `EACCES: permission denied` when reading a file, which layer (kernel or Node.js) generated that error?**
7. **Why does a command that works on Ubuntu sometimes fail on Alpine?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---------|-------------|-----------|
| `uname` | Print system/kernel info | `-r` (kernel version), `-a` (all info) |
| `cat /etc/os-release` | Show distribution identity | — |
| `uptime` | System uptime + load average | — |
| `nproc` | Number of CPU cores | — |
| `free` | Memory usage | `-h` (human-readable) |
| `lscpu` | CPU architecture details | — |
| `ps aux` | List all running processes | `aux` = all users, full format |
| `dmesg` | Kernel ring buffer (kernel messages) | `\| tail` (recent messages) |
| `strace` | Trace system calls of a process | `-c` (summary), `-e` (filter) |
| `/proc/` | Virtual filesystem — kernel exposes info here | Browse with `ls`, read with `cat` |

---

## When Would I Use This at Work?

### Scenario 1: Your Docker build fails on Alpine
Your `Dockerfile` uses `FROM node:18-alpine`. A native npm package fails to compile. You now know WHY: Alpine uses musl (not glibc) and doesn't have GNU build tools by default. Fix: `RUN apk add --no-cache build-base python3`.

### Scenario 2: Your Node app keeps crashing with no error
You SSH in and run `dmesg | grep -i kill`. You see the kernel's OOM killer terminated your process. You now know the kernel did it — not a bug in your code. Fix: increase RAM or fix a memory leak.

### Scenario 3: "Permission denied" in production
Your deploy script can't write to `/var/www/app`. You know this isn't a Node bug — it's the KERNEL checking permissions. You check `ls -la /var/www/app`, see the owner is root, and your app runs as user `deploy`. Fix: `chown deploy:deploy /var/www/app`.

---

## Connected Topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Next** | 02 — The Filesystem Hierarchy | Now that you know the kernel manages filesystems, learn HOW the directory tree is organized |
| **Builds on** | Nothing — this is the foundation | Everything else in this curriculum sits on top of these concepts |
| **Used by** | Every single subsequent topic | The kernel/userspace/syscall mental model is referenced in ALL future docs |
