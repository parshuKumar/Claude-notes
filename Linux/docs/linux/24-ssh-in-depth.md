# 24 — SSH in Depth

## ELI5 — The Simple Analogy

Imagine you need to talk privately to a bank teller, but you're shouting across a crowded market square. Everyone can hear you.

- **Step 1 — you invent a secret language, out loud, in front of everyone.** This sounds impossible, but it isn't: you and the teller each shout a *mixed paint colour*, each secretly adds your own private colour, and you both end up holding the exact same final shade — while the eavesdroppers, who heard every shouted colour, cannot reproduce it. That's **Diffie-Hellman key exchange**: you agree on a **shared secret over a public channel**, and *the secret itself is never spoken*.
- **Step 2 — but wait: is that even the real teller?** Maybe a con artist intercepted you. So the teller shows you their **signature stamp** — one you've seen before and wrote down in your notebook. If the stamp doesn't match what's in your notebook, you walk away. That's the **host key** and `~/.ssh/known_hosts`.
- **Step 3 — now the teller wants to know YOU are you.** You don't hand over your passport (that could be photocopied). Instead the teller writes a random number on a slip, and you **sign it with your personal seal** — a seal only you possess, which never leaves your pocket. The teller checks the signature against the seal-impression they have on file. That's **public-key authentication**: your private key **never leaves your machine**, ever.
- **Step 4 — the secret language is fast, so you use it for the whole conversation.** The fancy stamps and seals were only needed to *bootstrap trust*. That's **asymmetric crypto to establish the session, then symmetric crypto for the bulk data.**

SSH is that entire dance, and it happens in about 200 milliseconds every time you type `ssh`.

---

## Where This Lives in the Linux Stack

```
Hardware (NIC — the packets on the wire, which anyone can capture)
  └── KERNEL
       │    • TCP/IP stack: the 3-way handshake to port 22
       │    • AF_INET sockets, and AF_UNIX sockets (ssh-agent!)
       │    • Random number source /dev/urandom (used for key generation)
       │
       └── System Calls
            │    socket(), connect(), bind(), listen(), accept()
            │    read()/write() on the socket — CIPHERTEXT only
            │    fork(), execve()  ← sshd forks a child per connection
            │    setuid()/setgid() ← sshd drops privileges to YOUR user after auth
            │
            └── C Library + OpenSSL/libcrypto
                 │
                 └── OPENSSH  ◀◀◀ THIS TOPIC (a pure userspace program!)
                      │    • CLIENT: /usr/bin/ssh   → reads ~/.ssh/config
                      │    • SERVER: /usr/sbin/sshd → reads /etc/ssh/sshd_config
                      │    • ssh-agent, ssh-keygen, ssh-copy-id, scp, sftp
                      │    • rsync rides ON TOP of ssh as its transport
                      │
                      └── Your shell, on the far end
```

**Key insight:** SSH is **not** in the kernel. It is an ordinary userspace program that opens a TCP socket and does cryptography in userspace. The kernel just moves bytes and has no idea they're encrypted. This is why `tcpdump` on port 22 shows you nothing useful — the ciphertext is opaque, and that's the whole point.

---

## What Is This?

**SSH (Secure Shell)** is a protocol for getting an authenticated, encrypted, integrity-checked byte stream to a remote machine. Usually that stream is a shell, but it can carry file transfers (`scp`, `sftp`, `rsync`), arbitrary TCP forwarding (tunnels), and agent connections.

It replaced `telnet` and `rlogin`, which sent your **password in plaintext** across the wire.

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| The handshake | You'll blindly type `yes` to host-key prompts and have no idea what you just trusted |
| "REMOTE HOST IDENTIFICATION HAS CHANGED" | You'll panic, or worse — `rm ~/.ssh/known_hosts` and destroy every host you'd ever verified |
| Key permissions | `Permissions 0644 for 'id_rsa' are too open` on your first day, and you'll `chmod 777` it and make it worse |
| `authorized_keys` perms + StrictModes | "My key isn't working" — 90% of the time it's a group-writable `$HOME`, and the error message doesn't tell you |
| `PasswordAuthentication no` | You'll disable passwords, restart sshd, and **lock yourself permanently out of a production server** |
| `~/.ssh/config` | You'll type `ssh -i ~/.ssh/prod.pem -p 2222 -J bastion@1.2.3.4 ubuntu@10.0.1.7` fifty times a day |
| Port forwarding | You'll expose Postgres to the internet "so pgAdmin can reach it" — instead of tunnelling |
| rsync's trailing slash | You'll create `/srv/app/app/app` or nuke the wrong directory with `--delete` |
| `ServerAliveInterval` | Your `npm ci` dies at 4 minutes because the NAT gateway killed the idle connection |

---

## The Physical Reality — What Exists on Disk

### On YOUR laptop (the client)

```
~/.ssh/
├── id_ed25519          -rw-------  (0600)  ◀◀ PRIVATE KEY. Encrypted with your passphrase.
│                                              NEVER LEAVES THIS MACHINE. Never uploaded.
│                                              SSH REFUSES TO USE IT if perms are looser.
├── id_ed25519.pub      -rw-r--r--  (0644)  ◀◀ PUBLIC key. Safe to email, post, commit.
├── known_hosts         -rw-r--r--  (0644)  ◀◀ Fingerprints of SERVERS you've trusted.
├── config              -rw-------  (0600)  ◀◀ Your per-host settings (the superpower).
└── (dir itself)        drwx------  (0700)
```

A private key file is literally just text:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABDy...
      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ "aes256-ctr" + "bcrypt" → THIS KEY HAS A PASSPHRASE.
      If you see "none" there, it's UNENCRYPTED at rest.
-----END OPENSSH PRIVATE KEY-----
```

### On the SERVER

```
/etc/ssh/
├── sshd_config              ◀◀ SERVER config  (note the "d" — DAEMON)
├── ssh_config               ◀◀ system-wide CLIENT defaults (people mix these up CONSTANTLY)
├── ssh_host_ed25519_key     ◀◀ the SERVER's private HOST key (0600, root). This is what
│                                proves "I am really web-1". Regenerated on reimage → the
│                                dreaded "IDENTIFICATION HAS CHANGED" warning.
├── ssh_host_ed25519_key.pub ◀◀ what your client hashes into a fingerprint
├── ssh_host_rsa_key(.pub)
└── sshd_config.d/*.conf     ◀◀ drop-ins (cloud images put PasswordAuthentication here!)

/home/deploy/.ssh/
├── authorized_keys          -rw-------  (0600)   ◀◀ a list of PUBLIC keys allowed to log in
└── (dir)                    drwx------  (0700)
/home/deploy                 drwxr-xr-x  ← MUST NOT be group- or other-writable (StrictModes)
```

---

## How It Works — Step by Step (the full handshake)

```
CLIENT (your laptop)                                    SERVER (web-1, sshd on :22)
════════════════════                                    ═══════════════════════════

 1. TCP CONNECT
    socket(); connect(1.2.3.4:22) ─────── SYN ─────────►
                                 ◄──── SYN-ACK ─────────
                                 ────── ACK ───────────►
    (kernel work. Nothing secret yet. Everything below is on this one TCP stream.)

 2. PROTOCOL VERSION EXCHANGE          ← PLAINTEXT, in the clear
    "SSH-2.0-OpenSSH_9.6"    ─────────►
                             ◄───────── "SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6"
    (this is why `nc host 22` prints the sshd version — and why scanners fingerprint you)

 3. ALGORITHM NEGOTIATION (KEXINIT)    ← PLAINTEXT
    Each side sends its ordered lists: key-exchange algs, host-key types,
    ciphers, MACs, compression. They pick the first mutual match.
      kex:      curve25519-sha256
      hostkey:  ssh-ed25519
      cipher:   chacha20-poly1305@openssh.com
      mac:      (implicit, AEAD)

 4. KEY EXCHANGE — DIFFIE-HELLMAN / ECDH    ◀◀◀ THE MAGIC
    ┌──────────────────────────────────────────────────────────────────────┐
    │  client picks random  a  → sends  A = g^a   (public, in the clear)   │
    │  server picks random  b  → sends  B = g^b   (public, in the clear)   │
    │                                                                       │
    │  client computes:  K = B^a                                            │
    │  server computes:  K = A^b     ← THE SAME K. Mathematically identical.│
    │                                                                       │
    │  An eavesdropper saw g, A, and B — and CANNOT compute K.             │
    │  ▶ K IS NEVER TRANSMITTED. It is DERIVED, independently, on both ends.│
    └──────────────────────────────────────────────────────────────────────┘
    K → hashed into the SESSION KEYS (encryption key, MAC key, IVs — separate
        keys for each direction).

 5. SERVER AUTHENTICATION (host key)   ◀◀◀ "are you really web-1?"
    Server SIGNS the exchange hash H with its PRIVATE host key
    (/etc/ssh/ssh_host_ed25519_key) and sends the signature + its PUBLIC host key.

    CLIENT: verifies the signature with that public key   ✓ math checks out
    CLIENT: hashes the public key → a FINGERPRINT
            SHA256:x9dK3f...
    CLIENT: looks it up in ~/.ssh/known_hosts
            ├── found & matches      → silently proceed
            ├── NOT found            → "The authenticity of host ... can't be
            │                           established. Are you sure? (yes/no)"   ← TOFU
            └── found & DIFFERENT    → "@@@ WARNING: REMOTE HOST IDENTIFICATION
                                        HAS CHANGED! @@@"  → REFUSES to connect
    ⚠ NOTE: this step proves the SERVER's identity. You are NOT authenticated yet.

 6. SWITCH TO ENCRYPTION  (NEWKEYS)
    ══════════════════════════════════════════════════════════════════════════
    Everything from this point is SYMMETRICALLY encrypted with the session key.
    ══════════════════════════════════════════════════════════════════════════
    ▶ WHY: asymmetric crypto (ed25519/RSA) is SLOW — it's used ONLY to
      bootstrap trust. Symmetric crypto (ChaCha20/AES) is ~1000x faster and
      carries ALL the actual data. This "hybrid" design is the core idea
      behind SSH, TLS, and basically all modern secure transport.

 7. CLIENT AUTHENTICATION  ← only NOW does the server care who you are
    ┌─ publickey (the good way) ────────────────────────────────────────────┐
    │  CLIENT: "I'd like to use this public key: <blob>"                     │
    │  SERVER: reads ~deploy/.ssh/authorized_keys. Is that blob in there?    │
    │          (checks perms first — StrictModes! see mistakes)              │
    │          → "yes, that key is allowed. Prove you have the private half."│
    │  SERVER: sends a CHALLENGE (derived from the session ID — so it's      │
    │          unique per-session and can't be replayed)                     │
    │  CLIENT: SIGNS the challenge with the PRIVATE key                      │
    │          ▶ THE PRIVATE KEY IS NEVER SENT. Not encrypted, not hashed,   │
    │            NOT AT ALL. Only a signature crosses the wire.              │
    │  SERVER: verifies the signature with the public key from               │
    │          authorized_keys.  ✓ → AUTHENTICATED                           │
    └───────────────────────────────────────────────────────────────────────┘
    ┌─ password (the bad way) ──────────────────────────────────────────────┐
    │  CLIENT sends the password over the (already-encrypted) channel.       │
    │  Safe from eavesdropping — but brute-forceable, phishable, reusable,   │
    │  and the server now sees your secret. Turn it OFF.                     │
    └───────────────────────────────────────────────────────────────────────┘

 8. SESSION
    sshd forks a child, setuid()s to `deploy`, allocates a PTY,
    execve()s /bin/bash. You get a prompt. Every keystroke is a small
    encrypted packet.
```

---

## Asymmetric Crypto — Properly

A keypair is two mathematically-linked numbers.

| | Public key | Private key |
|---|---|---|
| Who has it | Everyone. The server. GitHub. Your blog. | **Only you. It never leaves your machine.** |
| **Encrypt** | ✅ anyone can encrypt *to* you | ❌ |
| **Decrypt** | ❌ | ✅ only you can open it |
| **Sign** | ❌ | ✅ only you can produce a valid signature |
| **Verify** | ✅ anyone can check your signature | ❌ |

SSH authentication uses **sign / verify**, not encrypt / decrypt:

```
   SERVER                                        CLIENT
   ──────                                        ──────
   "prove it. Here's a random challenge C"  ──►
                                                 sig = SIGN(C, private_key)
                                            ◄──  sig
   VERIFY(C, sig, public_key) == true?
   ✓ → you must hold the private key.
       And I still don't have it. And I never will.
```

**Consequences you should internalise:**
- Putting your `.pub` on 500 servers is completely safe.
- A compromised server **cannot** steal your key from an SSH login — it only ever saw a signature.
- If someone steals your *private key file*, they have your key — **unless it has a passphrase**, which encrypts it at rest.

---

## Key Generation

```
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519
│          │           │                    │
│          │           │                    └── -f: output file (default ~/.ssh/id_<type>)
│          │           └── -C: a COMMENT. Purely a label — it ends up at the end of the
│          │                   .pub file so you can tell which key is which in
│          │                   authorized_keys. It is NOT an identity or an email check.
│          └── -t: key TYPE.  ed25519 ◀◀ USE THIS
│                             rsa     → if you must (old hardware/AWS), then -b 4096
│                             ecdsa   → avoid (NIST curve, murkier provenance)
│                             dsa     → DEAD. Removed from OpenSSH. Never.
└── the key generator

  It will ask:  "Enter passphrase (empty for no passphrase):"
                 ◀◀ USE ONE. It encrypts the PRIVATE KEY FILE AT REST with
                    bcrypt-KDF + AES. A stolen laptop ≠ a compromised server.
                    ssh-agent means you type it once per day, not once per ssh.
```

### Why ed25519 over RSA

| | ed25519 | RSA-4096 |
|---|---|---|
| Public key size | 68 bytes (one short line) | ~730 bytes |
| Signing speed | Very fast | Slow |
| Security level | ~128-bit, modern EdDSA | ~128-bit-ish, ancient design |
| Bad-randomness resistance | **Deterministic signatures — immune to the "reused nonce" class of catastrophic bugs** | Sensitive to implementation |
| Support | OpenSSH 6.5+ (2014). Universal now. | Universal, incl. ancient boxes |

**Default to `ed25519`. Use RSA-4096 only when something old refuses ed25519.**

### The permissions rule that bites everyone on day one

```bash
$ ssh deploy@web-1
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/home/you/.ssh/id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

SSH **refuses** to use it. This is not a warning you can shrug at — the client will not even offer the key. (This bites constantly when a `.pem` file comes out of a download or a `git clone`, both of which give 0644.)

```bash
chmod 600 ~/.ssh/id_ed25519      # -rw------- : only you can read it
chmod 700 ~/.ssh                 # drwx------
```

---

## known_hosts and TOFU

**TOFU = Trust On First Use.** The first time you connect, SSH has no idea who the server is, so it asks you:

```
The authenticity of host 'web-1 (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:x9dK3fVv7Lq2pR8mN4tYcXe1bZ5wQhJ0sA6uD7iG9kM.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

**What you are being asked:** "Here is a hash of this server's public host key. Does it match the one your cloud provider / colleague / provisioning output told you to expect?"

**What you should do:** compare it against the fingerprint printed in your instance's console output, or run on the server (via the cloud console): `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`.

**What ~100% of engineers actually do:** type `yes`.

**What a MITM looks like:** an attacker on the path (compromised WiFi, hostile ISP, ARP-poisoned LAN, hijacked BGP) intercepts your TCP connection and terminates it themselves. They present **their own host key**, you type `yes`, and now they hold two SSH sessions — one with you, one with the real server — and they read/modify **everything in the clear** between them. The *only* thing standing between you and this is you checking that fingerprint.

After you say `yes`, the key is appended to `~/.ssh/known_hosts` and **every future connection is verified silently and automatically**. TOFU is weak on the first connection and strong forever after.

### "REMOTE HOST IDENTIFICATION HAS CHANGED!"

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
...
Offending ECDSA key in /home/you/.ssh/known_hosts:14
```

**What it literally means:** the host key this server just presented does **not** match the one recorded on line 14 of your `known_hosts`.

**Two possible causes:**

| Cause | Likelihood | What it means |
|---|---|---|
| You rebuilt / reimaged / recreated the VM, or the IP got recycled to a different box | **~99%** | New machine → new host keys generated at first boot. Totally expected. |
| An actual man-in-the-middle | ~1% | Someone is impersonating your server. |

**The correct fix (after you've convinced yourself it's cause #1):**

```bash
ssh-keygen -R web-1                 # remove JUST that host's entry
ssh-keygen -R 203.0.113.10          # and its IP, if you connect by IP too
#          │
#          └── -R = Remove a host from known_hosts. It rewrites the file and
#                  keeps a .old backup. SURGICAL.
```

**The wrong fix:** `rm ~/.ssh/known_hosts` — you just discarded every host you have ever verified, and re-armed the TOFU window for all of them at once.
**The much worse fix:** `StrictHostKeyChecking no` in your config, permanently. You have now disabled the single defence against MITM, everywhere, forever.

---

## `~/.ssh/authorized_keys` — and Why Your Key "Isn't Working"

On the **server**, in the target user's home:

```bash
# The right way — ssh-copy-id gets the perms right FOR you
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@web-1
#           │
#           └── the PUBLIC key (.pub!). It appends it to
#               ~deploy/.ssh/authorized_keys, creating .ssh (0700) and
#               authorized_keys (0600) if needed.

# The manual way (when ssh-copy-id isn't available, e.g. on macOS without brew)
cat ~/.ssh/id_ed25519.pub | ssh deploy@web-1 \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

### ⚠️ StrictModes — the 90% cause of "my key isn't working"

`sshd` has `StrictModes yes` by default. Before it will even *read* `authorized_keys`, it checks:

```
/home/deploy               drwxr-xr-x   ✅  must NOT be group- or other-WRITABLE
/home/deploy/.ssh          drwx------   ✅  0700
/home/deploy/.ssh/authorized_keys  -rw-------  ✅  0600
   ...and ALL of them must be owned by deploy (or root).
```

If **any** of those fail, sshd **silently ignores the file** and falls back to asking for a password. The client-side error is a useless `Permission denied (publickey)`. The truth is only on the server:

```bash
# ON THE SERVER, in another terminal:
sudo journalctl -u ssh -f
# Jul 12 14:31:02 web-1 sshd[4412]: Authentication refused: bad ownership or modes
#                                    for directory /home/deploy
#   ◀◀ THERE IT IS. Someone ran `chmod 775 /home/deploy` or the home dir is
#      group-writable from a bad useradd. That's the whole bug.
```

**The fix, every time:**
```bash
chmod 755 /home/deploy       # or 700
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

---

## ssh-agent

**Problem:** your key has a passphrase (it should). Typing it on every `ssh`, every `git push`, every `rsync` is unbearable.

**ssh-agent** is a small background process that holds your **decrypted private key in memory** and performs signatures on your behalf. It exposes an AF_UNIX socket; the `ssh` client talks to it via `$SSH_AUTH_SOCK`.

```bash
eval "$(ssh-agent -s)"        # start it; exports SSH_AUTH_SOCK + SSH_AGENT_PID
ssh-add ~/.ssh/id_ed25519     # decrypt it ONCE — type the passphrase here
ssh-add -l                    # list loaded keys (fingerprints)
ssh-add -L                    # list loaded keys (full public-key blobs)
ssh-add -D                    # delete all keys from the agent
ssh-add -t 3600 ~/.ssh/prod   # load with a 1-hour lifetime, then auto-forget

# In ~/.ssh/config — never think about it again:
Host *
    AddKeysToAgent yes        # auto-add a key to the agent the first time it's used
    UseKeychain yes           # macOS ONLY: store/read the passphrase in the Keychain
                              # (⚠ Linux does NOT have this option — it'll error)
```

```bash
# PROVE IT: the agent is a socket, and the key never leaves it
echo $SSH_AUTH_SOCK           # /private/tmp/com.apple.launchd.xyz/Listeners
ssh-add -l                    # 256 SHA256:x9dK... you@example.com (ED25519)
# The agent will SIGN for you. It will NEVER hand out the key material itself.
```

### Agent forwarding (`-A` / `ForwardAgent yes`) — and why to avoid it

Agent forwarding makes your *local* agent socket available on the *remote* host, so you can hop from `bastion` → `web-1` using the key on your laptop, without ever copying the key to the bastion.

```
laptop ──[agent socket forwarded over the ssh channel]──► bastion ──ssh──► web-1
   │                                                        │
   └── your key lives HERE, and stays here                  └── has a SOCKET that
                                                                can ASK your laptop
                                                                to sign things
```

**⚠️ THE DANGER:** anyone with **root on the bastion** (or a compromised process running as your user there) can read `$SSH_AUTH_SOCK` from your process environment, connect to that socket, and **use your key to authenticate as you, to anywhere** — for as long as your session is open. They can't steal the key, but they don't need to. They can just borrow it.

**Prefer `ProxyJump`.** It tunnels a *new, direct, end-to-end encrypted* SSH session to `web-1` **through** the bastion. The bastion only ever sees ciphertext, and your agent is never exposed to it.

```
   -A  (ForwardAgent)              -J  (ProxyJump)  ◀◀ USE THIS
   ────────────────────            ───────────────────────────────
   laptop → bastion (auth #1)      laptop ──── TCP tunnel through bastion ────► web-1
             ↓ agent exposed              ONE ssh session, end-to-end encrypted,
           bastion → web-1                bastion is a dumb pipe and sees nothing.
```

---

## `~/.ssh/config` — The Superpower

Everything you'd type as a flag can live here. **This file is the difference between an SSH novice and someone who is fast.**

```
# ~/.ssh/config      (chmod 600)

# ── The bastion / jump host ────────────────────────────────────
Host bastion
    HostName      bastion.example.com
    User          ubuntu
    Port          22
    IdentityFile  ~/.ssh/id_ed25519
    IdentitiesOnly yes

# ── Production app servers, reached THROUGH the bastion ────────
Host prod-web-*
    HostName      %h.internal.example.com   # %h = the literal host you typed
    User          deploy
    IdentityFile  ~/.ssh/id_prod_ed25519
    IdentitiesOnly yes
    ProxyJump     bastion                   # ◀◀ THE MODERN WAY THROUGH A BASTION

# ── A box on a nonstandard port with a .pem ───────────────────
Host legacy-db
    HostName      10.0.4.19
    User          ec2-user
    Port          2222
    IdentityFile  ~/.ssh/legacy.pem
    IdentitiesOnly yes
    ProxyJump     bastion
    # bring the remote Postgres to MY laptop on :5433 automatically
    LocalForward  5433 127.0.0.1:5432

# ── Global defaults (LAST — first match wins for each keyword!) ─
Host *
    AddKeysToAgent      yes
    ServerAliveInterval 60      # send a keepalive every 60s...
    ServerAliveCountMax 3       # ...give up after 3 unanswered (=180s)
    ControlMaster       auto    # reuse ONE TCP connection for many sessions
    ControlPath         ~/.ssh/cm-%r@%h:%p
    ControlPersist      10m     # keep the master alive 10 min after the last session
    HashKnownHosts      yes
    StrictHostKeyChecking ask   # the default. Do NOT set this to "no".
```

| Directive | Why you want it |
|---|---|
| `Host <alias>` | `ssh prod-web-1` instead of a 90-character command |
| `HostName` | The real DNS name/IP the alias resolves to |
| `User` / `Port` / `IdentityFile` | The obvious ones |
| **`IdentitiesOnly yes`** | **Without this, the client offers EVERY key in your agent, in order. The server counts each rejection against `MaxAuthTries` (default 6) and disconnects: `Too many authentication failures`. With 8 keys in your agent, you can fail to log in with a perfectly good key.** |
| **`ProxyJump bastion`** (`-J`) | End-to-end encrypted hop through a bastion. Replaces the old `ProxyCommand ssh -W %h:%p bastion` — same idea, one word. |
| **`ServerAliveInterval 60`** | **Stops your session freezing on idle.** NAT gateways, load balancers, and corporate firewalls silently drop idle TCP flows after ~5 min. Without keepalives your terminal just... hangs. Forever. This is what kills your `npm install` halfway through. |
| **`ControlMaster` + `ControlPath` + `ControlPersist`** | **Connection multiplexing.** The first `ssh` opens a real TCP+crypto handshake and leaves a master socket behind. Every *subsequent* `ssh`/`scp`/`rsync`/`git push` to that host reuses it — **connecting in ~10ms instead of ~400ms.** This makes ansible-style loops and `git push` over SSH dramatically faster. |
| `ForwardAgent yes` | Only where you truly need it. Prefer ProxyJump. |
| `LocalForward` / `RemoteForward` / `DynamicForward` | Tunnels, declared once (see below) |
| `Host *` block | **Put it LAST.** SSH uses **first-match-wins per keyword** — a value set in an earlier block cannot be overridden by a later one. |

```bash
# PROVE IT: multiplexing works
time ssh prod-web-1 true    # 0.42s  ← full handshake, opens the master
time ssh prod-web-1 true    # 0.01s  ← reused the existing connection ⚡
ssh -O check prod-web-1     # Master running (pid=4831)
ssh -O exit  prod-web-1     # tear the master down
```

---

## Port Forwarding — All Three, With Diagrams

These are enormously useful and almost universally misunderstood. The trick is to always ask: **"which machine does the new listening port appear on?"**

### 1. LOCAL forward (`-L`) — "bring a REMOTE port to MY machine"

```
ssh -L 8080:localhost:5432 deploy@web-1
    │  │    │         │
    │  │    │         └── the PORT to connect to, AS SEEN FROM web-1
    │  │    └── the HOST to connect to, AS SEEN FROM web-1 ("localhost" = web-1 itself!)
    │  └── the port to OPEN ON MY LAPTOP
    └── Local forward
```

```
  YOUR LAPTOP                              WEB-1                        DB
  ┌───────────────┐                  ┌───────────────┐        ┌──────────────────┐
  │ TablePlus     │                  │               │        │ Postgres         │
  │   ↓ connects  │                  │               │        │ listening ONLY   │
  │ 127.0.0.1:8080│══════════════════╪═══════════════╪═══════►│ on 127.0.0.1:5432│
  │  (ssh listens)│  ENCRYPTED SSH   │  ssh opens a  │        │ FIREWALLED from  │
  └───────────────┘    TUNNEL        │  plain TCP    │        │ the internet     │
                                     │  conn to      │        └──────────────────┘
                                     │  localhost:5432│
                                     └───────────────┘
```

**The killer use case:** your production Postgres listens on `127.0.0.1:5432` only, and the firewall (topic 29) drops 5432 from the world — exactly as it should. But you need to point pgAdmin/TablePlus at it *right now*. So:

```bash
ssh -N -L 8080:localhost:5432 deploy@web-1
#    │
#    └── -N = "do NOT run a remote command." Just hold the tunnel open. No shell.
#        (add -f to background it: ssh -f -N -L ... )

# Now point TablePlus at:  host=127.0.0.1  port=8080  ← it thinks it's talking to
#                                                        a local DB. It's talking to prod.
psql -h 127.0.0.1 -p 8080 -U app mydb    # works
```

**Variant — reaching a *third* machine (a private RDS instance):**

```bash
ssh -N -L 5433:mydb.abc123.us-east-1.rds.amazonaws.com:5432 deploy@bastion
#            │  └── resolved and connected FROM THE BASTION, inside the VPC.
#            │      Your laptop can't even resolve this name. It doesn't need to.
#            └── on your laptop
```
This is *the* standard way to reach a private RDS/ElastiCache from your laptop.

### 2. REMOTE forward (`-R`) — "expose MY local port ON the remote"

```
ssh -R 9000:localhost:3000 deploy@web-1
    │  │    │         │
    │  │    │         └── port on MY LAPTOP to forward TO
    │  │    └── host as seen from MY LAPTOP
    │  └── the port to OPEN ON WEB-1
    └── Remote forward
```

```
  YOUR LAPTOP                                    WEB-1
  ┌────────────────────┐                  ┌─────────────────────────┐
  │ node server.js     │                  │  ssh LISTENS on :9000   │◄── a colleague
  │ on 127.0.0.1:3000  │◄═════════════════╪══ (or a webhook from    │    curl web-1:9000
  │ (your dev box!)    │  ENCRYPTED       │    Stripe hits it)      │
  └────────────────────┘  SSH TUNNEL      └─────────────────────────┘
```

**Use cases:** demo your local dev server to a colleague; receive a **webhook** (Stripe/GitHub) on your laptop without deploying; give a build server temporary access to something local.

**⚠ `GatewayPorts`:** by default the remote listener binds to `127.0.0.1:9000` on web-1 — so only someone **already on web-1** can reach it. To let the *outside world* hit `web-1:9000`, the **server** must have `GatewayPorts yes` (or `clientspecified`) in `sshd_config`, and you use `-R 0.0.0.0:9000:localhost:3000`. Think hard before enabling this — you're punching a hole through your firewall from the inside.

### 3. DYNAMIC forward (`-D`) — a SOCKS proxy

```
ssh -D 1080 -N deploy@bastion
    │  │
    │  └── open a SOCKS5 proxy on MY laptop, port 1080
    └── Dynamic
```

```
  Browser / curl                       BASTION                    ANYWHERE in the VPC
  ┌──────────────────┐          ┌──────────────────┐        ┌────────────────────┐
  │ SOCKS5 →         │══════════│ ssh resolves DNS │───────►│ 10.0.3.9:8080      │
  │ 127.0.0.1:1080   │ ENCRYPTED│ and connects for │───────►│ internal-admin:80  │
  └──────────────────┘          │ EACH request     │───────►│ ...anything...     │
                                └──────────────────┘        └────────────────────┘
```

Unlike `-L` (one fixed destination), `-D` proxies **arbitrary** destinations chosen per-connection by the client. It's a general-purpose "put my browser inside the VPC" button.

```bash
curl --socks5-hostname 127.0.0.1:1080 http://10.0.3.9:8080/admin
#          ^^^^^^^^^^^ resolve DNS on the BASTION side, not locally — important
#                      for internal-only hostnames
```

| Flag | Mnemonic | Listening port appears on | Use it for |
|---|---|---|---|
| `-L local:host:port` | **L**ocal → pull it to me | **Your laptop** | Reaching a firewalled prod DB from a GUI client |
| `-R remote:host:port` | **R**emote → push mine out | **The server** | Webhooks, demoing localhost |
| `-D port` | **D**ynamic → SOCKS | **Your laptop** | Browsing a private network |
| `-N` | **N**o command | — | Tunnel only, no shell |
| `-f` | Fork to background | — | Combine: `ssh -fN -L ...` |

---

## Server Config — `/etc/ssh/sshd_config`

> **`sshd_config` (SERVER, the daemon) vs `ssh_config` (CLIENT).** People mix these up constantly. If you're editing the box you're *logging into*, it's `sshd_config`. Note also: on Ubuntu 22.04+ cloud images, `/etc/ssh/sshd_config.d/50-cloud-init.conf` often sets `PasswordAuthentication yes` and **overrides** the main file. Check the drop-ins.

```
Port 22                          # change it and you'll cut bot noise ~99% (but it is
                                 #   obscurity, not security — the firewall is security)
PermitRootLogin no               # ◀◀ ALWAYS. Log in as a user, then sudo (topics 06/32).
                                 #   Root login = no audit trail of WHO did it.
PasswordAuthentication no        # ◀◀ THE SINGLE HIGHEST-VALUE HARDENING STEP.
                                 #   Kills every SSH brute-force attack, permanently.
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
StrictModes yes                  # enforce the perm checks (leave it on)
MaxAuthTries 3                   # disconnect after 3 failed attempts
MaxSessions 10
AllowUsers deploy admin          # ◀◀ a whitelist. Nobody else can even TRY. (Or AllowGroups sshusers)
ClientAliveInterval 300          # server-side keepalive: ping an idle client every 300s
ClientAliveCountMax 2            #   ...disconnect after 2 unanswered (reaps dead sessions)
PermitEmptyPasswords no
X11Forwarding no                 # you're on a server. You don't need it.
LogLevel VERBOSE                 # ◀◀ logs the key FINGERPRINT used for each login →
                                 #   you can tell WHICH key logged in. Great for auditing.
```

### 🔥 THE LOCKOUT — read this before you touch anything

**If you set `PasswordAuthentication no` and restart sshd *before* confirming your key works, and your key does not work, you are permanently locked out of that server.** No password. No key. Your only recovery is a cloud provider serial/VNC console, a rescue instance, or rebuilding the box.

**The safe procedure, every single time:**

```bash
# 1. On the SERVER: edit the config
sudo vim /etc/ssh/sshd_config
sudo grep -r PasswordAuthentication /etc/ssh/sshd_config.d/   # ← check the drop-ins too!

# 2. TEST THE CONFIG SYNTAX — a typo here means sshd won't start at all
sudo sshd -t
#         │
#         └── -t = TEST mode. Parses the config, prints errors, exits. Changes nothing.
#             It prints NOTHING on success. Do this EVERY time.

# 3. Reload (does NOT kill your current session — sshd's children survive)
sudo systemctl reload ssh

# 4. ⚠️⚠️ KEEP THIS SESSION OPEN. DO NOT CLOSE IT. ⚠️⚠️
#    Open a SECOND terminal on your laptop and prove you can get back in:
ssh -v deploy@web-1
#   ✅ It worked → NOW you may close the first session.
#   ❌ It failed → you still have the first session. Fix it. Undo it. You are saved.
```

Say it out loud: **always keep a working session open until a second, fresh login succeeds.**

---

## Debugging — `ssh -v`

```bash
ssh -v deploy@web-1      # -v, -vv, -vvv — more v's, more detail
```

Read it like this:

```
debug1: Connecting to web-1 [203.0.113.10] port 22.
debug1: Connection established.                          ← TCP is fine (not a firewall issue)
debug1: Local version string SSH-2.0-OpenSSH_9.6
debug1: Remote protocol version 2.0, remote software version OpenSSH_8.9p1 Ubuntu
debug1: kex: algorithm: curve25519-sha256                 ← key exchange chosen
debug1: Server host key: ssh-ed25519 SHA256:x9dK3f...     ← the HOST KEY fingerprint
debug1: Host 'web-1' is known and matches the ED25519 host key.   ← known_hosts ✅
debug1: Authentications that can continue: publickey,password    ← what the server ALLOWS
debug1: Offering public key: /home/you/.ssh/id_rsa RSA SHA256:aaa...       ← tried
debug1: Authentications that can continue: publickey,password             ←   REJECTED
debug1: Offering public key: /home/you/.ssh/id_work ED25519 SHA256:bbb...  ← tried
debug1: Authentications that can continue: publickey,password             ←   REJECTED
debug1: Offering public key: /home/you/.ssh/id_prod ED25519 SHA256:ccc...  ← tried
debug1: Server accepts key: /home/you/.ssh/id_prod ED25519 SHA256:ccc...   ← ✅ ACCEPTED
debug1: Authentication succeeded (publickey).
```

**Reading it tells you:**
- **Did TCP connect?** No → firewall/security group/wrong port. Not an SSH problem at all.
- **Which keys did I offer?** If your key isn't in the list, SSH never even tried it → wrong `IdentityFile`, or it's not in the agent, or the perms are too open (SSH silently skips it).
- **Was any key accepted?** If all were rejected and it fell back to `password`, the key isn't in `authorized_keys` — **or StrictModes rejected the file.** Check the server.

**Server-side is where the truth is:**

```bash
sudo journalctl -u ssh -f          # live
sudo tail -F /var/log/auth.log     # same thing, text (topic 23)

# Jul 12 14:31:02 web-1 sshd[4412]: Authentication refused: bad ownership or modes for directory /home/deploy
# Jul 12 14:31:02 web-1 sshd[4412]: Connection closed by authenticating user deploy 203.0.113.7 port 51422 [preauth]
#   ◀◀ The client says "Permission denied (publickey)". The SERVER says exactly why.
```

---

## scp and rsync

### scp — simple, and legacy

```
scp -r ./dist deploy@web-1:/srv/app/
│   │  │      │            │
│   │  │      │            └── destination path on the remote
│   │  │      └── user@host
│   │  └── local source
│   └── -r = recursive
└── secure copy
```

```bash
scp file.tar.gz deploy@web-1:/tmp/           # push
scp deploy@web-1:/var/log/app.log ./         # pull
scp -P 2222 file deploy@web-1:/tmp/          # ⚠ CAPITAL -P for port. ssh uses lowercase -p. Yes, really.
```

> **scp is deprecated.** The old SCP protocol had a design flaw (a malicious server could write files you didn't ask for). Modern OpenSSH (9.0+) quietly implements `scp` on top of **SFTP** instead. Prefer `rsync` (smarter) or `sftp` (safer) for anything nontrivial. `scp` is still fine for "one file, right now."

### rsync — and why it exists

**The delta-transfer algorithm.** rsync splits the destination file into blocks, computes a rolling checksum + a strong hash for each, sends those to the sender, and the sender then transmits **only the blocks that differ**, plus instructions to reuse the rest.

```
  Re-syncing a 2 GB directory after changing 3 files:
    scp    → transfers 2 GB.  ~4 minutes.
    rsync  → transfers ~8 MB. ~3 seconds.
```

It also skips files entirely whose **size + mtime** already match (the fast default path), which is why a no-op `rsync` of a huge tree takes a second.

```
rsync -avz --progress --delete --exclude='node_modules' ./dist/ deploy@web-1:/srv/app/
│     ││││  │          │        │                        │       │
│     ││││  │          │        │                        │       └── destination
│     ││││  │          │        │                        └── source (NOTE THE TRAILING SLASH)
│     ││││  │          │        └── skip these paths (repeatable; or --exclude-from=file)
│     ││││  │          └── DELETE files in dest that don't exist in src → dest MIRRORS src
│     ││││  └── show a live per-file progress bar
│     │││└── -z: compress DURING TRANSFER (not on disk). Great over WAN, pointless on LAN/localhost.
│     ││└── -v: verbose
│     │└── -a: ARCHIVE = -rlptgoD
│     │        r recursive   l copy symlinks AS symlinks   p preserve permissions
│     │        t preserve mtimes ◀◀ CRITICAL — without -t, rsync can't tell what changed
│     │        g preserve group   o preserve owner (needs root)   D devices+specials
│     └── (the flags cluster)
└── rsync
```

### ⭐ THE TRAILING SLASH RULE — this trips up literally everyone

```
rsync -a src/ dest/      →  copies the CONTENTS of src INTO dest
                            result: dest/file1  dest/file2

rsync -a src  dest/      →  copies the DIRECTORY src INTO dest
                            result: dest/src/file1  dest/src/file2
                                         ^^^^ an extra level you didn't want
```

**Mnemonic:** a trailing slash on the **source** means *"the stuff inside"*. No slash means *"the folder itself"*. The trailing slash on the **destination** never matters.

Run it twice without the slash and you get `dest/src/src`. Combine "no trailing slash" with `--delete` and you can mirror-delete an entire directory you meant to fill.

### ⚠️ `--delete` — the footgun

```bash
# ❌ DISASTER: a typo (or an empty variable!) in the source path
rsync -a --delete /srv/app-typo/ deploy@web-1:/srv/app/
#                  ^^^^^^^^^^^^ doesn't exist locally → rsync errors out, fine.
# But:
SRC=""                                    # a bug in your deploy script
rsync -a --delete "$SRC/" web-1:/srv/app/ # → syncs "/" ... or an empty dir.
#                                            EVERYTHING IN /srv/app IS DELETED.

# ✅ ALWAYS dry-run first. ALWAYS.
rsync -avz --delete --dry-run ./dist/ deploy@web-1:/srv/app/
#                   ^^^^^^^^^ (= -n) shows EXACTLY what it WOULD transfer and
#                             WOULD delete. Changes nothing. Read the output.
#                             THEN remove --dry-run and run it for real.
```

### Other rsync flags worth knowing

| Flag | What |
|---|---|
| `-n` / `--dry-run` | **Do this first. Every time you use `--delete`.** |
| `--progress` / `-P` | `-P` = `--partial --progress` |
| `--partial --append-verify` | **Resume a big transfer over a flaky link** — keeps the partial file and appends, verifying the existing prefix's checksum |
| `--bwlimit=5000` | Cap at ~5 MB/s so you don't saturate the prod NIC and cause an incident |
| `-e 'ssh -p 2222'` | Use a non-default SSH port (or any ssh options: `-e 'ssh -J bastion'`) |
| `--exclude-from=.rsyncignore` | A file of exclude patterns |
| `--chown=deploy:deploy` | Set ownership at the destination |
| `-h` | Human-readable sizes in the output |
| `--checksum` / `-c` | Compare by checksum, not size+mtime (slow, but catches same-size same-mtime changes) |

---

## Example 1 — Basic

```bash
# 1. Generate a key (accept the default path; SET A PASSPHRASE)
ssh-keygen -t ed25519 -C "you@example.com"

# 2. Look at what you made
ls -la ~/.ssh/
# -rw------- 1 you you  464 Jul 12 14:10 id_ed25519       ← 0600. PRIVATE.
# -rw-r--r-- 1 you you  102 Jul 12 14:10 id_ed25519.pub   ← 0644. Public.

cat ~/.ssh/id_ed25519.pub
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH8xk...q3F you@example.com
#     ↑type      ↑the actual public key (base64)              ↑the -C comment

# 3. See its fingerprint — this is what the server logs and what you compare
ssh-keygen -lf ~/.ssh/id_ed25519.pub
# 256 SHA256:x9dK3fVv7Lq2pR8mN4tYcXe1bZ5wQhJ0sA6uD7iG9kM you@example.com (ED25519)

# 4. Load it into the agent (type the passphrase ONCE)
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l                      # confirm it's loaded

# 5. Install the PUBLIC key on the server
ssh-copy-id deploy@web-1        # (asks for the password ONE last time)

# 6. Log in with the key — no password
ssh -v deploy@web-1 'hostname; whoami'
# ... debug1: Server accepts key: /home/you/.ssh/id_ed25519 ED25519 SHA256:x9dK...
# web-1
# deploy

# 7. Confirm the server's host key fingerprint matches what your client trusted
ssh deploy@web-1 'ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub'
```

---

## Example 2 — Production Scenario

**Friday 16:40. Deploy the built Node app to `prod-web-1`, which is only reachable through the bastion. Then pull a DB dump. Then debug a slow query with your local GUI.**

```bash
# ── ~/.ssh/config already has: ──────────────────────────────────
# Host bastion
#     HostName bastion.example.com
#     User ubuntu
#     IdentityFile ~/.ssh/id_ed25519
#     IdentitiesOnly yes
# Host prod-web-1
#     HostName 10.0.1.7
#     User deploy
#     IdentityFile ~/.ssh/id_prod
#     IdentitiesOnly yes
#     ProxyJump bastion
# Host *
#     ControlMaster auto
#     ControlPath ~/.ssh/cm-%r@%h:%p
#     ControlPersist 10m
#     ServerAliveInterval 60
# ────────────────────────────────────────────────────────────────

# ── 1. DEPLOY: dry-run FIRST. Always. ──────────────────────────
$ npm run build

$ rsync -avz --delete --dry-run \
    --exclude='node_modules' --exclude='.git' --exclude='.env' \
    ./dist/ prod-web-1:/srv/app/current/
sending incremental file list
deleting current/old-bundle.a1b2.js          ◀◀ read this. Is deleting it correct? YES.
./
server.js
assets/main.9f3c.js
sent 1,204 bytes  received 88 bytes  861.33 bytes/sec
total size is 4,102,884  speedup is 3,175.22        ◀◀ only the CHANGED files. delta-transfer.

# Looks right (it's only touching /srv/app/current, only deleting the stale bundle).
$ rsync -avz --delete --progress \
    --exclude='node_modules' --exclude='.git' --exclude='.env' \
    ./dist/ prod-web-1:/srv/app/current/
#            ^^^^                    ^
#            TRAILING SLASH on the source → copy the CONTENTS of dist/.
#            Without it you'd create /srv/app/current/dist/  ← the classic bug.

$ ssh prod-web-1 'cd /srv/app/current && npm ci --omit=dev && sudo systemctl restart myapp'
#   ⚡ this connects INSTANTLY — ControlMaster reused the socket from the rsync above.

$ ssh prod-web-1 'systemctl is-active myapp && curl -sf localhost:3000/healthz'
active
{"status":"ok","version":"1.4.2"}          ✅ deployed

# ── 2. PULL A 3GB PRODUCTION DB DUMP, SAFELY ───────────────────
$ ssh prod-web-1 'pg_dump -Fc mydb | gzip > /tmp/mydb.dump.gz'   # topic 25 — stream it
$ ssh prod-web-1 'ls -lh /tmp/mydb.dump.gz'
-rw-r--r-- 1 deploy deploy 2.9G Jul 12 16:44 /tmp/mydb.dump.gz

# --bwlimit so we don't saturate prod's NIC and cause an incident on a Friday.
# --partial --append-verify so a dropped connection doesn't cost us 40 minutes.
$ rsync -avhP --bwlimit=5000 --partial --append-verify \
    prod-web-1:/tmp/mydb.dump.gz ./backups/
mydb.dump.gz
      2.90G 100%    4.77MB/s    0:10:23 (xfr#1, to-chk=0/1)
$ ssh prod-web-1 'rm /tmp/mydb.dump.gz'     # don't leave a 3GB dump on prod's disk (topic 23!)

# ── 3. TUNNEL TO THE PRIVATE RDS INSTANCE FROM YOUR GUI ────────
# The DB is in a private subnet. It has NO public IP. Your laptop cannot route to it.
# The bastion can. So: make the bastion do the connecting.
$ ssh -f -N -L 5433:mydb.abc123.us-east-1.rds.amazonaws.com:5432 bastion
#     │  │  │    │                                          │
#     │  │  │    │                                          └── port on the RDS box
#     │  │  │    └── resolved & connected FROM THE BASTION (inside the VPC).
#     │  │  │        Your laptop can't even resolve this hostname. It doesn't need to.
#     │  │  └── the port that appears on YOUR LAPTOP
#     │  └── no remote command — just hold the tunnel
#     └── background it

$ psql -h 127.0.0.1 -p 5433 -U readonly mydb -c '\dt'
#      ^^^^^^^^^ your laptop. The bytes go: psql → ssh → bastion → RDS. Encrypted the whole way.
#      Point TablePlus/pgAdmin at 127.0.0.1:5433 and it Just Works.

$ pkill -f 'ssh -f -N -L 5433'     # tear the tunnel down when you're done
```

---

## Common Mistakes

### Mistake 1 — `chmod 777 ~/.ssh/id_rsa` to "fix" the permissions error

**Wrong:** You see `Permissions 0644 for 'id_rsa' are too open`, so you `chmod 777`.
**Root cause:** the error says **too open**. You made it *maximally* open. SSH still refuses (and now every user on the box can read your private key).
**Diagnose:** `ls -l ~/.ssh/id_rsa` → anything other than `-rw-------`.
**Fix:** `chmod 600 ~/.ssh/id_ed25519 && chmod 700 ~/.ssh`
**Prevent:** a downloaded `.pem` from AWS always arrives 0644. `chmod 600` it the moment it lands.

---

### Mistake 2 — Disabling password auth before testing the key → **permanent lockout**

**Wrong:**
```bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
exit                      # ◀◀ and now you're locked out forever
```
**Root cause:** your key wasn't actually working (group-writable `$HOME`, wrong user, key in the wrong `authorized_keys`, or a `sshd_config.d/` drop-in overriding you). Passwords are now off. There is no way in.
**What the broken state looks like:** `Permission denied (publickey).` — and nothing else. No prompt. No recourse.
**Fix:** cloud provider serial console / VNC, or detach the volume and mount it on a rescue instance. Hours of your life.
**Prevent:** **Keep the working session open. Open a SECOND terminal. Prove you can log in. Only then close the first.** Also run `sudo sshd -t` before every reload, and `grep -r PasswordAuthentication /etc/ssh/sshd_config.d/` to catch cloud-init drop-ins.

---

### Mistake 3 — `Permission denied (publickey)` from a group-writable home directory

**Wrong:** you `ssh-copy-id`'d correctly, the key is in `authorized_keys`, and it still fails.
**Root cause:** `sshd` has `StrictModes yes`. It `stat()`s `/home/deploy`, `/home/deploy/.ssh`, and `authorized_keys`. If **any** is writable by group or other, sshd **refuses to read the file at all** — reasoning that another user could add their own key to it. The client-side error is generic and unhelpful.
**Diagnose (server-side — this is the only place the truth lives):**
```bash
sudo journalctl -u ssh -n 20
# sshd[4412]: Authentication refused: bad ownership or modes for directory /home/deploy
```
**Fix:**
```bash
chmod 755 /home/deploy && chmod 700 /home/deploy/.ssh && chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```
**Prevent:** use `ssh-copy-id`, which sets everything correctly. And when `publickey` fails, **always** read the server log before touching the client.

---

### Mistake 4 — `rm ~/.ssh/known_hosts` after "IDENTIFICATION HAS CHANGED"

**Wrong:** the warning is scary, so you nuke the file.
**Root cause:** you rebuilt the VM; new host keys were generated at first boot. Deleting the whole file discards the verified fingerprints of **every other server you've ever used** — and re-opens the TOFU window on all of them simultaneously.
**Even worse:** adding `StrictHostKeyChecking no` to `Host *`. That permanently disables the *only* MITM defence SSH has.
**Right:**
```bash
ssh-keygen -R web-1              # remove ONLY that host
ssh-keygen -R 203.0.113.10       # ...and its IP
# then reconnect and VERIFY the new fingerprint against the cloud console output
```
**Prevent:** in ephemeral/CI environments where hosts really are recreated constantly, scope it narrowly:
```
Host 10.0.*
    StrictHostKeyChecking accept-new     # accept a NEW host silently, but STILL refuse a CHANGED one
    UserKnownHostsFile /dev/null
```
`accept-new` is the sane middle ground — never use a bare `no`.

---

### Mistake 5 — `Too many authentication failures`

**Wrong:** you have 9 keys in your agent and a `.pem` in your config; the server disconnects before it ever tries the right one.
**Root cause:** the client offers keys **one at a time**, in order. The server counts each rejection against `MaxAuthTries` (default 6) and **disconnects on the 7th attempt** — possibly before reaching your correct key.
**Diagnose:** `ssh -v host` → count the `Offering public key:` lines. If you see 6 and then a disconnect, that's it.
**Fix / Prevent:**
```
Host web-1
    IdentityFile ~/.ssh/id_prod
    IdentitiesOnly yes        # ◀◀ "offer ONLY the IdentityFile above, and nothing from the agent"
```

---

### Mistake 6 — The rsync trailing slash / `--delete` disaster

**Wrong:** `rsync -a --delete ./dist prod-web-1:/srv/app/`
**Root cause:** no trailing slash on the source → rsync copies the *directory* `dist` into the destination, creating `/srv/app/dist/`. Then `--delete` **removes everything else in `/srv/app/`** because it isn't in the source — including `/srv/app/current`, which was your running app.
**What the broken state looks like:** the app is gone, systemd is restart-looping, and `/srv/app/` contains one folder called `dist`.
**Right:** `rsync -a --delete ./dist/ prod-web-1:/srv/app/current/` — trailing slash on the source, an explicit destination subdir.
**Prevent:** **`--dry-run` is mandatory whenever `--delete` is present.** Read every `deleting ...` line before you commit.

---

## Hands-On Proof

```bash
# PROVE IT: your private key NEVER crosses the wire — only a signature does
ssh -vvv deploy@web-1 2>&1 | grep -iE "sign|offering|accepts"
# debug1: Offering public key: /home/you/.ssh/id_ed25519 ED25519 SHA256:x9d...
# debug1: Server accepts key: ...
# debug3: sign_and_send_pubkey: ED25519 SHA256:x9d...   ◀◀ SIGN and send. Not "send key".

# PROVE IT: the passphrase encrypts the key file at rest
head -2 ~/.ssh/id_ed25519 | base64 -d 2>/dev/null | strings | head -3
# aes256-ctr   ◀◀ the private key is AES-encrypted
# bcrypt       ◀◀ ...with a bcrypt-derived key from your passphrase
# (if you see "none", your key is stored in PLAINTEXT — a stolen laptop = a stolen key)

# PROVE IT: the server's host key is a file, and its fingerprint is what you trusted
ssh deploy@web-1 'ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub'
# 256 SHA256:aB3dEf... root@web-1 (ED25519)
ssh-keygen -F web-1 | tail -1 | ssh-keygen -lf -    # what YOUR known_hosts recorded
#  ↑ these two fingerprints MUST match. That's the whole security model.

# PROVE IT: the agent holds the key and signs on demand — it never releases it
ssh-add -l                       # 256 SHA256:x9dK... (ED25519)
ls -l $SSH_AUTH_SOCK             # srwx------ ... it's an AF_UNIX SOCKET
# There is NO ssh-add command that prints your private key. By design.

# PROVE IT: SSH is a userspace program talking TCP on port 22
sudo ss -tlnp | grep :22
# LISTEN 0  128  0.0.0.0:22  0.0.0.0:*  users:(("sshd",pid=901,fd=3))
sudo lsof -p $(pgrep -f 'sshd.*deploy') | head    # topic 19 — its file descriptors

# PROVE IT: a local forward opens a listener ON YOUR LAPTOP
ssh -fN -L 8080:localhost:5432 deploy@web-1
lsof -iTCP:8080 -sTCP:LISTEN            # ssh (not postgres!) is listening on YOUR machine
# ssh  4831 you  5u  IPv4 ... TCP localhost:http-alt (LISTEN)

# PROVE IT: multiplexing makes the 2nd connection ~40x faster
time ssh web-1 true ; time ssh web-1 true
ssh -O check web-1               # Master running (pid=4831)

# PROVE IT: rsync's delta transfer only sends what changed
dd if=/dev/urandom of=/tmp/big bs=1M count=100 2>/dev/null
rsync -a --stats /tmp/big web-1:/tmp/     | grep "Total transferred"
echo "one more line" >> /tmp/big
rsync -a --stats /tmp/big web-1:/tmp/     | grep "Total transferred"
#  ↑ the second run transfers a few KB, not 100 MB.
```

---

## Practice Exercises

### Exercise 1 — Easy

In the terminal:
1. Generate a **new** ed25519 key at `~/.ssh/practice_ed25519` **with a passphrase**.
2. Print its fingerprint with `ssh-keygen -lf`.
3. `chmod 644` the private key, then try `ssh -i ~/.ssh/practice_ed25519 localhost`. Capture the exact error.
4. `chmod 600` it and confirm the error is gone.
5. Start `ssh-agent`, `ssh-add` the key, and show it with `ssh-add -l`. Confirm no command exists that will print the private key from the agent.

Paste your terminal output.

---

### Exercise 2 — Medium

Set up a real `~/.ssh/config` and prove each feature works. Use any host you have (a VM, a Docker container running `sshd`, or `localhost`).

1. Write a `Host` block with an alias, `HostName`, `User`, `IdentityFile`, and `IdentitiesOnly yes`.
2. Add `ControlMaster auto` / `ControlPath` / `ControlPersist 10m`. Run `time ssh <alias> true` **twice** and show the second is dramatically faster. Then `ssh -O check <alias>`.
3. Run `ssh -v <alias>` and identify, from the output: (a) which host key algorithm was used, (b) which keys were *offered*, (c) which key was *accepted*.
4. Open a local forward: `ssh -fN -L 8080:localhost:22 <alias>`. Prove with `lsof -iTCP:8080 -sTCP:LISTEN` that `ssh` is listening **on your machine**. Then `ssh -p 8080 localhost` and land on the remote — through your own tunnel.

Paste your config and output.

---

### Exercise 3 — Hard (Production Simulation)

Build a two-hop bastion setup with Docker, then deploy through it.

1. Run two containers on a Docker network: `bastion` and `app`, both running `sshd`. Only `bastion` publishes port 22 to your host; `app` is reachable **only** from `bastion`.
2. Generate a key, install the public key into both containers' `authorized_keys` (get the perms right — 0700/.ssh, 0600/authorized_keys, and a non-group-writable home).
3. Write a `~/.ssh/config` with `Host app` → `ProxyJump bastion`. Prove `ssh app` works in one command, and prove with `ssh -v` that you never authenticated *to the app* from the bastion — it's end-to-end.
4. **Break it deliberately:** `chmod 777 /home/deploy` inside `app`. Watch the login fail. Find the real reason in the container's `journalctl -u ssh` / `/var/log/auth.log`. Fix it.
5. `rsync -avz --delete --dry-run ./somedir/ app:/srv/app/` **through the ProxyJump**. Read the dry-run output. Then run it for real. Then run it **without** the trailing slash and observe the `/srv/app/somedir/` bug.
6. Open a local forward through the bastion to a service running only on `app`'s loopback. Curl it from your laptop.

Paste your full terminal session.

---

## Mental Model Checkpoint

Answer from memory.

1. **Walk through the 7 steps of the SSH handshake. At which step is the SERVER authenticated, and at which step are YOU?**
2. **Why is asymmetric crypto used only at the start, and symmetric crypto for everything after?**
3. **Does your private key ever leave your machine during an SSH login? What actually crosses the wire to prove you hold it?**
4. **What does `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` mean literally, what are the two causes, and what is the correct fix?**
5. **Your key is in `authorized_keys` and you still get `Permission denied (publickey)`. Name the most likely cause and the exact command that reveals it.**
6. **Explain `-L 8080:localhost:5432 user@server` in one sentence. On which machine does port 8080 open?**
7. **Why is `ProxyJump` safer than `ForwardAgent` when hopping through a bastion?**
8. **What does the trailing slash change in `rsync -a src/ dest/` vs `rsync -a src dest/`?**
9. **What is the one thing you must do before setting `PasswordAuthentication no` and reloading sshd?**

---

## Quick Reference Card

| Command | What it does | Key flags |
|---|---|---|
| `ssh` | Connect | `-v/-vv/-vvv` (debug), `-i key`, `-p port`, `-J bastion` (ProxyJump), `-N` (no cmd), `-f` (background), `-A` (forward agent — careful), `-o Opt=val` |
| `ssh -L L:host:R` | Local forward — pull a remote port to me | Combine with `-N -f` |
| `ssh -R R:host:L` | Remote forward — push my port out there | Needs `GatewayPorts` to expose publicly |
| `ssh -D 1080` | SOCKS5 proxy | `curl --socks5-hostname 127.0.0.1:1080` |
| `ssh-keygen` | Make/inspect keys | `-t ed25519`, `-b 4096` (RSA), `-C comment`, `-l -f key` (fingerprint), **`-R host` (remove from known_hosts)**, `-F host` (find), `-p` (change passphrase), `-y -f key` (recover .pub from private) |
| `ssh-copy-id` | Install your `.pub` on a server, with correct perms | `-i key.pub` |
| `ssh-agent` / `ssh-add` | Hold decrypted keys in memory | `ssh-add -l`, `-L`, `-D` (drop all), `-t 3600` (TTL) |
| `sshd -t` | **Test sshd_config syntax before reloading** | Prints nothing on success |
| `scp` | Copy files (legacy) | `-r`, **`-P port` (CAPITAL P!)** |
| `sftp` | Interactive/scriptable file transfer | `get`, `put` |
| `rsync` | Smart sync (delta transfer) | `-avz`, `-P`, **`-n`/`--dry-run`**, `--delete`, `--exclude=`, `-e 'ssh -J bastion'`, `--bwlimit=`, `--partial --append-verify` |
| Client config | `~/.ssh/config` | `Host`, `HostName`, `User`, `IdentityFile`, **`IdentitiesOnly yes`**, **`ProxyJump`**, **`ServerAliveInterval 60`**, **`ControlMaster auto`** |
| Server config | `/etc/ssh/sshd_config` (+ `sshd_config.d/`) | **`PermitRootLogin no`**, **`PasswordAuthentication no`**, `AllowUsers`, `MaxAuthTries`, `StrictModes`, `LogLevel VERBOSE` |
| Server logs | Where the truth lives | `journalctl -u ssh -f`, `/var/log/auth.log` |

---

## When Would I Use This at Work?

### Scenario 1: Deploying a Node app through a bastion
Prod is in a private subnet. You put `ProxyJump bastion` and `ControlMaster auto` in `~/.ssh/config`, then `rsync -avz --delete --dry-run ./dist/ prod-web-1:/srv/app/current/`, read the deletions, run it for real, and `ssh prod-web-1 'sudo systemctl restart myapp'` — which connects instantly on the multiplexed socket. Total: three commands, ~8 seconds.

### Scenario 2: Debugging a production query from your laptop's GUI
Postgres listens on `127.0.0.1:5432` and the security group blocks 5432 from the world, correctly. Instead of weakening any of that, you run `ssh -fN -L 5433:localhost:5432 prod-db` and point TablePlus at `127.0.0.1:5433`. You get a full GUI against production, over an encrypted tunnel, with zero new attack surface.

### Scenario 3: A new engineer's key "doesn't work"
They send you a screenshot: `Permission denied (publickey)`. You don't guess. You run `sudo journalctl -u ssh -n 20` on the server and read `Authentication refused: bad ownership or modes for directory /home/newdev`. Someone created the home dir with a bad umask. `chmod 755 /home/newdev` and they're in — 90 seconds, no guesswork, because you know where the truth is logged (topic 23).

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Builds on** | 05 — File Permissions | 0600 keys, 0700 `.ssh`, StrictModes — SSH is a permissions exam |
| **Builds on** | 06 — Users and Groups | `authorized_keys` is per-user; `PermitRootLogin no` + sudo |
| **Builds on** | 19 — File Descriptors | The agent socket, the ControlMaster socket |
| **Builds on** | 22 — systemd | `systemctl reload ssh`, `journalctl -u ssh` |
| **Builds on** | 23 — Logs | `/var/log/auth.log` is where every SSH truth is written |
| **Next** | 25 — Compression & Archiving | `tar -czf - dir \| ssh host 'tar -xzf -'` — SSH as a pipe |
| **Used by** | 28 — Ports and Sockets | Port forwarding is socket plumbing; `ss -tlnp` to see your tunnels |
| **Used by** | 29 — Firewalls | Tunnelling is what you do *instead of* opening a port |
| **Used by** | 32 — Deploy User Permissions | The deploy user's `authorized_keys` and sudoers rules |
| **Used by** | 35 — Security Basics | `PasswordAuthentication no` + `PermitRootLogin no` + fail2ban is the hardening core |
