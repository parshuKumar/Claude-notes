# 09 — TLS/SSL in Depth

> **Phase 2 — Topic 4 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine you want to send a secret message to a bank, but the only way to get it there is to hand it to a chain of strangers who pass it along — the coffee-shop WiFi, the ISP, a dozen routers. Any of them could read your message or swap it for a fake one.

So you do something clever:

1. The bank publishes an **open padlock** to the whole world (its **public key**). Anyone can snap it shut, but only the bank has the **key that opens it** (its **private key**).
2. You grab the bank's open padlock, put your secret note in a box, and click the padlock shut. Now *only the bank* can open it — not the WiFi, not the ISP.
3. But padlocks are slow and clunky, so inside that first locked box you don't send the real message — you send a **shared secret code**. From now on, both you and the bank scramble everything with that fast shared code instead.
4. And how do you know the padlock really belongs to the bank and not an impostor? The padlock comes with a **notarized ID card** (a **certificate**), signed by a notary (a **Certificate Authority**) whose signature your computer already trusts.

That's TLS. The slow padlock (**asymmetric encryption**) is used just long enough to agree on a fast shared code (**symmetric encryption**), and a trusted notary (**the CA**) vouches that you're talking to the real bank and not a man-in-the-middle.

---

## Where This Lives in the Network Stack

TLS sits *between* TCP and your application protocol. It is a wrapper: TCP gives you a reliable byte stream, TLS encrypts that byte stream, and HTTP rides inside the encrypted tunnel.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, gRPC, SMTP, IMAP)              │  ← your app data
│  ─────────────────────────────────────────────────────────────  │
│  Layer 6 — Presentation  (★ TLS/SSL lives here ★)             │  ← encrypt/decrypt
│  ─────────────────────────────────────────────────────────────  │
│  Layer 5 — Session       (TLS session/resumption tickets)       │
│  ─────────────────────────────────────────────────────────────  │
│  Layer 4 — Transport     (TCP — TLS needs a reliable stream)    │  ← TLS runs ON TOP of TCP
│  Layer 3 — Network       (IP)                                   │
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (fiber, copper, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

Two important truths:
- **TLS requires TCP first.** The TCP 3-way handshake (Topic 08) must complete *before* the TLS handshake starts. That's why HTTPS costs 2 round trips to set up (1 for TCP, 1 for TLS 1.3).
- **"HTTPS" = HTTP + TLS.** There is no separate "HTTPS protocol." It is literally plain HTTP sent inside a TLS tunnel, conventionally on port 443. (QUIC/HTTP3 folds TLS into the transport itself — Topic 12 — but that's the exception.)

---

## What Is This?

**TLS (Transport Layer Security)** is the protocol that turns a naked TCP connection into a secure one. Its predecessor was **SSL (Secure Sockets Layer)**. People still say "SSL" out of habit, but every version of SSL (1.0, 2.0, 3.0) is **broken and deprecated** — if someone says "SSL certificate" they almost always mean a TLS certificate.

```
SSL 2.0 (1995) ──► SSL 3.0 (1996) ──► TLS 1.0 (1999) ──► TLS 1.1 (2006)
   BROKEN            BROKEN (POODLE)     DEPRECATED         DEPRECATED
                                             │
                                             ▼
                            TLS 1.2 (2008) ──► TLS 1.3 (2018)
                            still common        the modern standard
```

TLS provides three guarantees, and you should be able to recite them:

| Guarantee | What it means | Without it |
|-----------|---------------|------------|
| **Confidentiality** | Nobody on the path can *read* your data | The coffee-shop WiFi reads your passwords |
| **Integrity** | Nobody can *modify* your data undetected | An ISP injects ads into your pages |
| **Authentication** | You're talking to the *real* server, not an impostor | You send your card number to a phishing proxy |

Plain HTTP provides **none** of these. That's why we say HTTP is "naked."

---

## Why Does It Matter for a Backend Developer?

You will spend real hours of your career debugging TLS. Not because it's badly designed — because it touches everything and fails loudly.

- **Certificate expiry** takes down more production systems than almost any other single cause. A cron job forgot to renew; at 3am your API starts returning `certificate has expired` and every client breaks at once.
- **"Works in the browser, fails in curl/Node"** — the classic **missing intermediate certificate** bug. Browsers paper over it; your backend HTTP client does not. You *will* hit this.
- **mTLS between microservices** (Topic 36) — internal service-to-service calls in a zero-trust network authenticate *both* sides with certificates. You need to understand chains to configure it.
- **Terminating TLS at a load balancer** (Topic 28) — knowing where decryption happens tells you where you can inspect traffic and where you can't.
- **Debugging latency** (Topic 27) — the TLS handshake is a measurable slice of your request time (`time_appconnect` in curl). Knowing 1-RTT vs 2-RTT tells you whether an upgrade to TLS 1.3 will help.
- **Security posture** — you'll be asked "are we PCI compliant?" and part of the answer is "we disabled TLS 1.0/1.1 and only offer forward-secret cipher suites." You need to know what those words mean.

---

## The Packet/Protocol Anatomy

Everything TLS sends is wrapped in a **TLS record**. The record is the envelope that rides inside TCP.

```
┌──────────────── TLS RECORD ────────────────────────────────┐
│ HEADER (5 bytes, ALWAYS plaintext)                         │
│   Content Type (1B) │ Legacy Version (2B) │ Length (2B)    │
│   0x16 = handshake  │ 0x0303 = "TLS 1.2"  │ up to 16 KB    │
│   0x17 = app data   │ (for middlebox compat)               │
│─────────────────────────────────────────────────────────── │
│ PAYLOAD (ENCRYPTED once the handshake completes)           │
│   handshake messages  OR  encrypted app data  OR  alert    │
│   In TLS 1.3 almost everything after the first two         │
│   messages is encrypted.                                   │
└────────────────────────────────────────────────────────────┘
```

**Content Types you'll see:**

| Byte | Type | When |
|------|------|------|
| `0x16` (22) | Handshake | Setting up the connection (ClientHello, etc.) |
| `0x17` (23) | Application Data | Your actual HTTP request/response, encrypted |
| `0x15` (21) | Alert | Something went wrong (bad cert, close_notify) |
| `0x14` (20) | ChangeCipherSpec | Legacy TLS 1.2 marker (dummy in TLS 1.3) |

### The two kinds of encryption TLS juggles

This is the single most important concept in the whole topic:

```
ASYMMETRIC (public/private key: RSA, ECDSA, ECDHE)
   ✓ solves the "how do we agree on a secret over an open wire" problem
   ✗ SLOW — ~1000x slower than symmetric. Unusable for bulk data.
   Used ONLY during the handshake.

SYMMETRIC (one shared key: AES-GCM, ChaCha20-Poly1305)
   ✓ FAST — hardware-accelerated, gigabytes per second
   ✗ requires both sides to already share the same secret key
   Used for ALL application data after the handshake.

The genius of TLS:  use SLOW asymmetric crypto for a few milliseconds
to safely agree on a FAST symmetric key, then throw the asymmetric
part away and encrypt everything else with the fast key.
```

### What an X.509 certificate actually contains

A certificate is a signed data structure. When you run `openssl x509 -text` you see:

```
┌─────────────── X.509 CERTIFICATE ──────────────────────────┐
│  Subject:     CN = api.myapp.com            ← WHO it's for  │
│  SAN:         DNS: api.myapp.com,                           │
│               DNS: www.myapp.com, *.myapp.com  ← REAL list  │
│                                                of valid names│
│  Issuer:      CN = Let's Encrypt R3         ← WHO signed it │
│  Validity:    Not Before 2026-05-01 / Not After 2026-07-30 │
│  Public Key:  ECDSA P-256 (or RSA 2048)     ← server's key  │
│  Signature:   SHA256withRSA  a3:f1:9c:...   ← CA signed all │
│               (issuer's PRIVATE key signed everything above)│
└────────────────────────────────────────────────────────────┘
```

**Critical detail:** modern clients ignore the `Subject CN` for hostname matching and **only check the SAN (Subject Alternative Name)**. A cert with `CN=api.myapp.com` but no matching SAN entry will be *rejected* by Chrome and Node. Always put the hostname in the SAN.

---

## How It Works — Step by Step

### The trust chain: why your computer believes a certificate

Your server's certificate isn't signed directly by a root authority. It's a **chain**:

```
┌───────────────┐  signs  ┌────────────────────┐  signs  ┌────────────────────┐
│   ROOT CA     │ ──────► │  INTERMEDIATE CA   │ ──────► │  LEAF (your server)│
│  ISRG Root X1 │         │  Let's Encrypt R3  │         │  api.myapp.com     │
│ Self-signed.  │         │ Signed by root.    │         │ Signed by interm.  │
│ In your OS/   │         │ Served BY YOUR     │         │ Served by your     │
│ browser TRUST │         │ SERVER alongside   │         │ server.            │
│ STORE.        │         │ the leaf.          │         │                    │
└───────────────┘         └────────────────────┘         └────────────────────┘
        ▲                                                          │
        └──────── client verifies the chain from leaf UP ─────────┘
   "Leaf signed by intermediate? Intermediate signed by a root I already
    trust in my OS store? Yes → connection is trusted."
```

**Why a chain at all?** Root CA private keys are so valuable they're kept offline in vaults. They sign a handful of intermediates; the intermediates do the day-to-day signing. If an intermediate is compromised, it can be revoked without touching the precious root.

**The trust store** is the anchor. Your OS (macOS Keychain, Windows Cert Store, Linux `/etc/ssl/certs`) and browsers ship a list of ~150 root CA certificates. Trust flows *down* from them. A **self-signed** certificate has no chain to any trusted root, which is why browsers scream at you — nobody vouched for it. **Let's Encrypt** (via the **ACME** protocol) is a free, automated CA that gives you a real CA-signed cert in ~30 seconds; that's why HTTPS is now universal.

### The TLS 1.3 handshake — 1 round trip

```
CLIENT                                                         SERVER
  │  (TCP handshake already done — SYN / SYN-ACK / ACK)         │
  │ ── ClientHello ────────────────────────────────────────►   │
  │      • supported versions (1.3) + cipher suites             │
  │      • key_share: my ephemeral ECDHE PUBLIC key ★          │
  │      • SNI: "api.myapp.com" ← which site (PLAINTEXT)        │
  │      • ALPN: "h2","http/1.1" ← protocols I speak            │
  │                       ◄──────────────────── ServerHello ──  │
  │                            • chosen cipher                   │
  │                            • key_share: server ephemeral ★  │
  │   ★ BOTH SIDES NOW COMPUTE THE SAME SHARED SECRET ★         │
  │   (ECDHE: my private + your public = same secret.           │
  │    Everything below this line is ENCRYPTED.)                 │
  │                       ◄─────────── {EncryptedExtensions} ──  │
  │                       ◄──── {Certificate} (leaf + interm.) ─ │
  │                       ◄──────────── {CertificateVerify} ──   │
  │                            (server SIGNS the handshake with  │
  │                             its cert's PRIVATE key — proves  │
  │                             it OWNS the cert, not just a copy)│
  │                       ◄─────────────────── {Finished} ──     │
  │   Client verifies chain vs trust store + SAN = "api.myapp.com"│
  │ ── {Finished} ──────────────────────────────────────────►  │
  │ ── {App Data: GET /users HTTP/2} ─────────────────────────►│
  │        (client sends app data immediately — 1-RTT total!)   │
```

`{...}` = encrypted. Notice: after just **one round trip** (ClientHello → ServerHello) the client can already send its HTTP request. Certificate and everything else are *encrypted*.

### Contrast: TLS 1.2 needed 2 round trips

```
TLS 1.2                                    TLS 1.3
─────────────────────                      ─────────────────────
ClientHello         ──►                    ClientHello+key_share ──►
      ◄── ServerHello                            ◄── ServerHello+key_share
      ◄── Certificate                                ◄── Cert (encrypted)
      ◄── ServerKeyExchange                          ◄── Finished
      ◄── ServerHelloDone                      Finished ──►  [DONE — 1 RTT]
ClientKeyExchange   ──►
ChangeCipherSpec    ──►
Finished            ──►
      ◄── ChangeCipherSpec
      ◄── Finished
[DONE — 2 RTT]

TLS 1.2 = 2 round trips before you can send data.
TLS 1.3 = 1 round trip. On a 200ms link, that's 200ms saved per new connection.
Also: in 1.3 the certificate is ENCRYPTED; in 1.2 it was sent in plaintext.
```

**0-RTT resumption (TLS 1.3):** if you've connected to a server *before*, it can hand you a **session ticket**. On the next connection you send your first HTTP request as **"early data"** *inside the very first ClientHello* — **zero round trips** before data flows. Blazing fast for reconnections.

```
First visit:   TCP + TLS 1.3 handshake  → 2 RTT, get a session ticket
Return visit:  send GET as early data in ClientHello → 0-RTT before app data
```

**The replay caveat — read this twice.** 0-RTT early data is **not forward-secret** and **can be replayed** by an attacker who captures it: they resend the exact same encrypted early-data packet and the server processes it again. That's harmless for an idempotent `GET /products`, but catastrophic for `POST /transfer?amount=1000`. **Rule: only ever send idempotent, side-effect-free requests as 0-RTT early data.** Servers that can't guarantee this should reject early data entirely.

### ECDHE and forward secrecy

The `key_share` in the handshake carries an **ephemeral** (temporary, one-time) Elliptic Curve Diffie-Hellman key. This is the whole game:

```
Diffie-Hellman lets two parties who have never met agree on a shared secret
over a fully public wire, WITHOUT ever transmitting the secret itself.

  Client private (a) ─┐                    ┌─ Server private (b)
  Client public  (A) ─┼── sent openly ────┼─ Server public (B)
                      │                    │
  shared = f(a, B) ───┘   ==   same    ────┘ shared = f(b, A)

An eavesdropper sees A and B but CANNOT derive the shared secret
(that's the discrete-log hard problem).

The "E" in ECDHE = EPHEMERAL. A fresh keypair every single connection.

▶ FORWARD SECRECY: because the ephemeral keys are thrown away after the
  handshake, even if an attacker records all your encrypted traffic today
  AND steals your server's private key next year, they STILL cannot decrypt
  the recorded sessions. The session keys no longer exist anywhere.
```

Old RSA key exchange (no "E") did not have this property: the client encrypted the session key with the server's public key, so anyone who later stole the private key could decrypt *all* past recorded traffic. TLS 1.3 removed RSA key exchange entirely — **all TLS 1.3 connections have forward secrecy.**

### SNI — how one IP serves a thousand HTTPS sites

```
PROBLEM: One server at 203.0.113.10 hosts 500 HTTPS sites, each needing a
DIFFERENT cert. But the server must send its cert BEFORE it sees any HTTP
Host header (HTTP is inside the tunnel; the tunnel needs the cert first).
Chicken and egg.

SOLUTION: SNI (Server Name Indication). The client names the host it wants
in the ClientHello — the very first message — so the server knows which
certificate to present.

  ClientHello → server_name: "shopB.com"  ◄── SNI, in PLAINTEXT
  Server: "shopB.com — I'll present shopB's certificate."
```

**The catch: SNI is sent in plaintext**, before encryption is established. So even though HTTPS hides *what* you request, the SNI field leaks *which hostname* you're connecting to. Your ISP can see `server_name: bank.com` even on an HTTPS connection.

```
┌──────────────────────────────────────────────────────────────┐
│ What an on-path observer (ISP, WiFi) can STILL see on HTTPS: │
│   • Destination IP address       (always — it's in IP header)│
│   • SNI hostname                 (plaintext in ClientHello)  │
│   • Rough size & timing of data  (traffic analysis)          │
│ What they CANNOT see:                                        │
│   • The URL path, headers, body  (all encrypted)             │
│   • Your cookies, passwords, JSON (all encrypted)            │
└──────────────────────────────────────────────────────────────┘

THE FIX: ECH (Encrypted Client Hello). Encrypts the ClientHello (including
SNI) using a public key the server publishes in DNS (HTTPS record). Still
rolling out in 2026 — supported by Cloudflare + Firefox/Chrome, not universal.
```

### ALPN — negotiating HTTP/2 vs HTTP/1.1 for free

Also in the ClientHello, **ALPN (Application-Layer Protocol Negotiation)** lets the client and server agree on which application protocol to speak *during* the TLS handshake, with no extra round trip:

```
Client ALPN offer:  ["h2", "http/1.1"]
Server ALPN pick:   "h2"     ← "we'll speak HTTP/2"

This is exactly how a browser and server decide to use HTTP/2 (Topic 11)
without an extra negotiation. gRPC (Topic 37) relies on ALPN = "h2" too.
```

---

## Exact Syntax Breakdown

The single most useful TLS debugging command is `openssl s_client`. Learn to read its output line by line.

```
openssl  s_client  -connect api.github.com:443  -servername api.github.com
│        │         │                            │
│        │         │                            └── SNI value sent in ClientHello
│        │         │                                (WITHOUT this, servers with
│        │         │                                 SNI-based virtual hosts give
│        │         │                                 you the WRONG cert or fail)
│        │         └── TCP target: host:port to connect to
│        └── "SSL/TLS client" mode — act as a TLS client and dump everything
└── the openssl toolkit
```

### Reading the output

```
CONNECTED(00000005)                          ← TCP connection established
depth=2 C=US, O=Internet Security..., CN=..  ← walking the chain: 2=root
depth=1 C=US, O=DigiCert Inc, CN=DigiCert..  ← 1=intermediate
depth=0 CN=api.github.com                    ← 0=leaf (the server's own cert)
verify return:1                              ← 1 = each cert verified OK

---
Certificate chain                            ← the certs the server SENT you
 0 s:CN=api.github.com                       ← s: = subject (leaf)
   i:C=US, O=DigiCert Inc, CN=DigiCert...    ← i: = issuer (the intermediate)
 1 s:C=US, O=DigiCert Inc, CN=DigiCert...    ← the intermediate itself
   i:C=US, O=DigiCert..., CN=DigiCert Global ← issued by the root
---
Server certificate
-----BEGIN CERTIFICATE-----                  ← the leaf cert in PEM format
MIIF...
-----END CERTIFICATE-----
subject=CN=api.github.com                    ← whose cert this is
issuer=C=US, O=DigiCert Inc, CN=DigiCert...  ← who signed it
---
SSL handshake has read 4500 bytes and written 400 bytes
---
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256  ← ★ protocol + cipher chosen
Server public key is 256 bit                    ← ECDSA P-256 key
Protocol  : TLSv1.3                              ← ★ TLS version negotiated
Cipher    : TLS_AES_128_GCM_SHA256              ← ★ symmetric cipher for data
ALPN protocol: h2                                ← ★ HTTP/2 negotiated
Verify return code: 0 (ok)                       ← ★★★ THE MONEY LINE ★★★
```

**The `Verify return code` is the line that matters most.** `0 (ok)` = the chain validated against your trust store. Anything else is a problem:

| Code | Meaning |
|------|---------|
| `0 (ok)` | Chain valid, cert trusted ✓ |
| `10` | certificate has expired |
| `18` | self-signed certificate |
| `19` | self-signed certificate in certificate chain (root not trusted) |
| `20` | unable to get local issuer certificate (**missing intermediate!**) |
| `21` | unable to verify the first certificate (**missing intermediate!**) |

### Decoding a certificate's contents

```
openssl x509 -in cert.pem -noout -text     # full human-readable dump
#            -noout = don't re-echo the encoded cert;  -text = parsed fields

openssl x509 -in cert.pem -noout -dates    # just the validity window
# notBefore=May  1 00:00:00 2026 GMT
# notAfter =Jul 30 23:59:59 2026 GMT

openssl x509 -in cert.pem -noout -ext subjectAltName   # just the valid hostnames
# X509v3 Subject Alternative Name:
#     DNS:api.myapp.com, DNS:www.myapp.com
```

---

## Example 1 — Basic

**Goal:** inspect a real site's TLS setup and confirm it's healthy.

```bash
# Pipe an empty input so s_client doesn't hang waiting for you to type
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Expected output:

```
subject=CN=github.com
issuer=C=GB, ST=Greater Manchester, L=Salford, O=Sectigo Limited, CN=Sectigo ECC Domain Validation Secure Server CA
notBefore=Feb  5 00:00:00 2026 GMT
notAfter =Feb  5 23:59:59 2027 GMT
```

Read it: the cert is **for** `github.com`, was **signed by** Sectigo, and is **valid** until Feb 2027. All three questions — who, by whom, until when — answered in one command.

Now watch the handshake summary:

```bash
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null \
  | grep -E "Protocol|Cipher|Verify"
```

```
Protocol  : TLSv1.3
Cipher    : TLS_AES_128_GCM_SHA256
Verify return code: 0 (ok)
```

Three facts confirmed: modern protocol (**1.3**), a strong AEAD cipher (**AES-128-GCM**), and a **trusted** chain (`0 (ok)`). This is what a healthy HTTPS endpoint looks like.

---

## Example 2 — Production Scenario

**The situation:** You deployed a new internal API at `https://payments.internal.myapp.com`. It works perfectly in your browser. But your Node.js service and a `curl` health check both fail:

```
Error: unable to verify the first certificate
    at TLSSocket.onConnectSecure (node:_tls_wrap:1670:34)
  code: 'UNABLE_TO_VERIFY_LEAF_SIGNATURE'
```

```bash
curl https://payments.internal.myapp.com/health
# curl: (60) SSL certificate problem: unable to get local issuer certificate
```

"Works in Chrome, fails in curl and Node." This is the **missing intermediate certificate** bug — the single most common TLS production failure. Let's prove it.

**Step 1 — See what the server actually sends:**

```bash
echo | openssl s_client -connect payments.internal.myapp.com:443 \
  -servername payments.internal.myapp.com 2>/dev/null | grep -A20 "Certificate chain"
```

```
Certificate chain
 0 s:CN=payments.internal.myapp.com
   i:CN=Let's Encrypt R3            ← leaf claims it was signed by "R3"
                                    ← ...but where IS R3's certificate?
                                    ← chain STOPS here. Only 1 cert sent.
```

**The smoking gun:** the chain has only entry `0` (the leaf). The server is *not* sending the intermediate (`Let's Encrypt R3`) cert. Compare a healthy server — it would show entries `0` AND `1`.

**Step 2 — Confirm the verify failure:**

```bash
echo | openssl s_client -connect payments.internal.myapp.com:443 \
  -servername payments.internal.myapp.com 2>/dev/null | grep "Verify return"
```

```
Verify return code: 21 (unable to verify the first certificate)
```

Code `21` / `20` = missing intermediate, confirmed.

**Why does Chrome work but curl/Node fail?** Browsers cache intermediate certs they've seen before and can fetch missing ones via the **AIA (Authority Information Access)** URL embedded in the cert. `curl` and Node do **not** do AIA fetching by default — they need the server to hand over the full chain. So the browser silently heals the misconfiguration and hides the bug from you.

**Step 3 — Prove it by supplying the intermediate manually:**

```bash
# Download the intermediate cert, then tell curl to use it as a trusted CA:
curl --cacert lets-encrypt-r3.pem https://payments.internal.myapp.com/health
# {"status":"ok"}   ← now it works. Confirms the ONLY problem was the missing chain.
```

**The fix:** configure your server to serve the **full chain** (leaf + intermediate), not just the leaf. With Let's Encrypt / certbot, use `fullchain.pem`, never `cert.pem`:

```nginx
# WRONG — leaf only, breaks curl/Node:
ssl_certificate     /etc/letsencrypt/live/myapp/cert.pem;

# RIGHT — leaf + intermediate:
ssl_certificate     /etc/letsencrypt/live/myapp/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/myapp/privkey.pem;
```

**Bonus — the other three failures you'll see** and how to spot them instantly with `curl -v`:

```bash
# EXPIRED CERTIFICATE
curl -v https://expired.badssl.com 2>&1 | grep -i "expire\|SSL cert"
#  curl: (60) SSL certificate problem: certificate has expired
#  → fix: renew the cert (check your certbot/ACME cron)

# HOSTNAME MISMATCH (cert is for a different name / SAN doesn't match)
curl -v https://wrong.host.badssl.com 2>&1 | grep -i "match\|CN\|SAN"
#  curl: (60) SSL: no alternative certificate subject name matches
#        target host name 'wrong.host.badssl.com'
#  → fix: reissue the cert with the right SAN, or connect to the right hostname

# SELF-SIGNED (no chain to any trusted root)
curl -v https://self-signed.badssl.com 2>&1 | grep -i "self"
#  curl: (60) SSL certificate problem: self-signed certificate
#  → fix: use a real CA-signed cert, or (internal only) add the CA to the trust store
```

The `badssl.com` family of subdomains is a live, safe playground for every one of these failure modes — bookmark it.

---

## Common Mistakes

### Mistake 1: Saying "SSL" when you mean "TLS" — and worse, still allowing SSL

```
"We need an SSL certificate."     → harmless slang; it's a TLS certificate.
"We support SSL 3.0 for old clients." → DANGEROUS. SSL 3.0 is broken (POODLE, 2014).
```
**Reality:** All SSL versions (1.0/2.0/3.0) and TLS 1.0/1.1 are deprecated and disabled in modern browsers. Configure your servers to accept **TLS 1.2 minimum**, ideally **1.3**. "Supporting old SSL for compatibility" is how you fail a security audit.

```nginx
ssl_protocols TLSv1.2 TLSv1.3;   # never list SSLv3, TLSv1, or TLSv1.1
```

---

### Mistake 2: Serving the leaf certificate without the intermediate chain

```
The bug: server sends ONLY its own cert, not the intermediate.
The tell: works in browsers, fails in curl / Node / Go / Python requests.
Why:      browsers cache/fetch intermediates; backend clients do not.
```
**Fix:** always serve `fullchain.pem` (leaf + intermediate), never `cert.pem` alone. Verify with `openssl s_client` — you must see chain entries `0` AND `1`. (This is Example 2 in full.)

---

### Mistake 3: Assuming self-signed certificates "just work" — or disabling verification to make them work

```javascript
// The panic fix that ends up in production and never leaves:
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';   // ☠️ DISABLES ALL CERT CHECKING
// or in requests: verify=False,  or curl -k / --insecure
```
**Why it's dangerous:** you've just turned off authentication *globally* for the whole process. Now *any* man-in-the-middle with *any* certificate is trusted. You wanted to skip one internal cert check and you disabled TLS's entire authentication guarantee.
**Fix:** for internal/self-signed CAs, *add the CA to the trust store* (`--cacert`, `NODE_EXTRA_CA_CERTS`, or the OS store) rather than disabling verification. Self-signed certs are fine for local dev, never for anything a real client touches.

---

### Mistake 4: Believing HTTPS hides *who* you're talking to

```
Myth:    "HTTPS is encrypted, so nobody knows which sites I visit."
Reality: The destination IP is in the plaintext IP header — always visible.
         The SNI hostname is plaintext in the ClientHello — visible (pre-ECH).
         Only the PATH, HEADERS, and BODY are encrypted.
```
**Consequence:** your ISP/corporate proxy can log every hostname you connect to over HTTPS, even though they can't read the content. ECH fixes the SNI leak but isn't universal in 2026. Don't design a system assuming the hostname is secret.

---

### Mistake 5: Letting certificates expire

```
04:00 — cert expires.
04:00 — every client on Earth simultaneously starts failing with
        "certificate has expired". Not a gradual degradation — a cliff.
```
**Why it happens:** someone got a 1-year cert, set a reminder, then left the company. Or the ACME renewal cron silently broke months ago and nobody noticed because the cert was still valid *until it wasn't*.
**Fix:** automate renewal (certbot/ACME, cloud-managed certs) **and** monitor expiry independently (`openssl ...-checkend`, an uptime probe, or a dashboard alert at 30/14/7 days out). Automation that fails silently is worse than a manual reminder.

---

### Mistake 6: Confusing where TLS is terminated

```
Client ══TLS══► Load Balancer ──plaintext HTTP──► Your App (127.0.0.1:3000)
                    ▲                                  ▲
              TLS terminated here            app sees UNENCRYPTED traffic
```
**The trap:** your app logs `http://` and port 80 and you panic that TLS is off. In fact the **load balancer terminated TLS** (Topic 28) and forwarded plaintext over the private network. That's normal. But it means (a) your app can't see the client cert unless the LB forwards it in a header, and (b) the LB↔app hop is only as safe as your private network. For zero-trust you re-encrypt that hop with **mTLS** (Topic 36).

---

## Hands-On Proof

```bash
# 1. Full handshake dump for any site — read the chain, protocol, cipher, verify code
echo | openssl s_client -connect cloudflare.com:443 -servername cloudflare.com 2>/dev/null \
  | grep -E "Protocol|Cipher|Verify|depth|s:|i:"

# 2. See ONLY the negotiated protocol + cipher (the fast summary)
echo | openssl s_client -connect google.com:443 -servername google.com 2>/dev/null \
  | grep -E "Protocol|Cipher|ALPN"

# 3. Force a TLS version — prove old versions are refused
openssl s_client -connect google.com:443 -tls1_1 2>&1 | grep -i "alert\|error\|Protocol"
#   → you'll see a handshake failure: TLS 1.1 is rejected by modern servers

# 4. Check how many days until a cert expires (great for a monitoring script)
echo | openssl s_client -connect github.com:443 -servername github.com 2>/dev/null \
  | openssl x509 -noout -checkend 2592000 && echo "OK: >30 days left" || echo "WARN: expires within 30 days"

# 5. Show the SAN list — the ACTUAL hostnames a cert is valid for
echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName

# 6. curl's TLS timing slice — how long did the handshake cost?
curl -o /dev/null -s -w "TCP connect: %{time_connect}s | TLS done: %{time_appconnect}s | TLS cost: %{time_appconnect}s (minus TCP)\n" \
  https://github.com

# 7. See ALPN pick HTTP/2, and watch the whole TLS conversation
curl -v --http2 https://github.com 2>&1 | grep -E "ALPN|SSL connection|subject|issuer|TLS"

# 8. Play with every failure mode safely (badssl.com is a live sandbox)
curl -v https://expired.badssl.com 2>&1 | grep -i "SSL cert"        # expired
curl -v https://self-signed.badssl.com 2>&1 | grep -i "self"        # self-signed
curl -v https://wrong.host.badssl.com 2>&1 | grep -i "match"        # hostname mismatch
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a live certificate

```bash
# Pick any HTTPS site and answer the questions from the output:
echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Answer:
# 1. What is the Subject (who is this cert for)?
# 2. Who is the Issuer (which CA signed it)? Is it a root or an intermediate?
# 3. On what date does it expire? How many days from today is that?
# 4. List every hostname in the SAN. Would this cert be valid for "en.wikipedia.org"?
```

### Exercise 2 — Medium: Prove forward secrecy exists in the negotiation

```bash
# Run this and look at the negotiated cipher suite and key exchange:
echo | openssl s_client -connect cloudflare.com:443 -servername cloudflare.com 2>/dev/null \
  | grep -E "Protocol|Cipher|Server Temp Key"

# The "Server Temp Key" line (e.g. "ECDH, X25519, 253 bits") is the EPHEMERAL
# key_share — its presence is the proof of forward secrecy.

# Answer:
# 1. Is the protocol TLS 1.3? If so, forward secrecy is GUARANTEED — why?
# 2. What does "Server Temp Key: ECDH, X25519" tell you about the key exchange?
# 3. If an attacker recorded this session AND stole Cloudflare's private key
#    tomorrow, could they decrypt what you sent today? Explain in one sentence.
```

### Exercise 3 — Hard (Production Simulation): Diagnose a broken chain from scratch

```bash
# Scenario: a teammate says "the API works in my browser but our Go service
# gets x509: certificate signed by unknown authority". Reproduce the diagnosis.

# Step 1 — dump the chain the server actually sends:
echo | openssl s_client -connect <YOUR_TEST_HOST>:443 -servername <YOUR_TEST_HOST> 2>/dev/null \
  | grep -A10 "Certificate chain"

# Step 2 — check the verify return code:
echo | openssl s_client -connect <YOUR_TEST_HOST>:443 -servername <YOUR_TEST_HOST> 2>/dev/null \
  | grep "Verify return"

# (You can use  untrusted-root.badssl.com  as YOUR_TEST_HOST to simulate a
#  chain that doesn't reach a trusted root.)

# Answer:
# 1. How many certs are in the chain (count the numbered "s:" lines)?
# 2. Does the chain reach a root your OS trusts? What's the Verify return code?
# 3. Is this a MISSING-INTERMEDIATE problem (code 20/21) or an
#    UNTRUSTED-ROOT problem (code 19)? How do you tell them apart?
# 4. Write the one-line fix for each case (hint: one is a server config change,
#    the other is a trust-store change on the client).
```

---

## Mental Model Checkpoint

Answer from memory. If you stumble, re-read the linked section.

1. **TLS uses both asymmetric and symmetric encryption. Which is used for what, and why not just use one?** (Think: speed vs the key-exchange problem.)

2. **You connect to `https://shop.example.com`. Walk through how your computer decides to *trust* the certificate it receives. What is the "trust store" and where does the chain end?**

3. **What single field in the certificate do modern clients check to confirm the hostname matches — and what field do they now *ignore*?**

4. **TLS 1.3 completes in 1 RTT; TLS 1.2 needed 2. On a Tokyo↔Virginia link (~200ms RTT), how much handshake time does upgrading save per new connection? What else did 1.3 encrypt that 1.2 sent in the clear?**

5. **Explain forward secrecy in one sentence. Why does the "E" (ephemeral) in ECDHE guarantee it, and why did old RSA key exchange not?**

6. **Your ISP is watching your HTTPS traffic to `bank.com`. Name two things they can still see and two things they cannot. What emerging technology closes the SNI gap?**

7. **A cert "works in Chrome but fails in curl and your Node service." What is the most likely single cause, which `openssl s_client` line confirms it, and what's the one-word fix (which file do you serve)?**

---

## Quick Reference Card

| Concept | What it is | Key fact to remember |
|---------|-----------|----------------------|
| TLS vs SSL | Transport Layer Security; SSL is its dead predecessor | All SSL + TLS 1.0/1.1 are deprecated. Use 1.2+ |
| Asymmetric crypto | Public/private key (RSA, ECDSA, ECDHE) | Slow; used only in the handshake to agree on a key |
| Symmetric crypto | One shared key (AES-GCM, ChaCha20) | Fast; encrypts all application data |
| X.509 certificate | Signed doc: subject, SAN, issuer, validity, public key | Hostname is matched against **SAN**, not CN |
| Certificate chain | leaf → intermediate → root CA | Serve leaf **+ intermediate** (`fullchain.pem`) |
| Trust store | OS/browser list of ~150 trusted root CAs | Trust anchors here; self-signed certs aren't in it |
| ACME / Let's Encrypt | Free automated CA | 90-day certs; automate renewal or you'll expire |
| TLS 1.3 handshake | ClientHello + ServerHello, then encrypted rest | **1 RTT**; cert is encrypted |
| ECDHE | Ephemeral elliptic-curve key exchange | Gives **forward secrecy** — past traffic stays safe |
| SNI | Hostname in ClientHello (plaintext) | Lets 1 IP serve many HTTPS sites; **leaks hostname** |
| ECH | Encrypted Client Hello | Fixes the SNI leak; not universal in 2026 |
| ALPN | Protocol negotiation in the handshake | Picks HTTP/2 (`h2`) vs HTTP/1.1 with no extra RTT |

**`openssl s_client` verify codes:**
```
0   = ok, chain trusted            ✓
10  = certificate has expired      → renew
18  = self-signed certificate      → use a real CA cert
19  = self-signed cert in chain    → root not trusted (add CA to store)
20/21 = can't get / verify issuer  → MISSING INTERMEDIATE (serve fullchain)
```

**Key commands:**
```bash
# Full handshake + chain + verify code
openssl s_client -connect HOST:443 -servername HOST

# Cert facts: who / by whom / until when
echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# Valid hostnames (SAN)
... | openssl x509 -noout -ext subjectAltName

# Expiry check for monitoring (exit 0 if >30 days left)
... | openssl x509 -noout -checkend 2592000

# See the failure reason quickly
curl -v https://HOST 2>&1 | grep -i "SSL\|certificate"

# TLS handshake cost
curl -o /dev/null -s -w "TLS done at: %{time_appconnect}s\n" https://HOST
```

**The three guarantees:** Confidentiality (can't read) · Integrity (can't modify) · Authentication (right server).

---

## When Would I Use This at Work?

### Scenario 1: "The API works in Postman/Chrome but our backend service throws a cert error"
Nine times out of ten this is a **missing intermediate certificate**. Run `openssl s_client -connect host:443 -servername host` and look at the `Certificate chain` block — if you only see entry `0` and the `Verify return code` is `20` or `21`, the server isn't serving its intermediate. Fix the server to send `fullchain.pem`. You just saved the afternoon someone else would have spent adding `rejectUnauthorized: false` to production.

### Scenario 2: "Set up HTTPS on our new service"
You reach for Let's Encrypt + certbot (ACME), configure nginx with `ssl_certificate fullchain.pem` and `ssl_protocols TLSv1.2 TLSv1.3`, and set up automated renewal. You verify with `openssl s_client` that the chain is complete and the protocol is 1.3, and you add an expiry monitor. You know *why* each of those steps exists, not just the copy-paste recipe.

### Scenario 3: "We need zero-trust between internal microservices"
Service-to-service **mTLS** (Topic 36) means *both* client and server present certificates. Understanding chains, private keys, and trust stores is exactly what lets you configure a service mesh (Envoy/Istio) or hand-roll mTLS in your HTTP clients — you provision each service with a cert from an internal CA and add that CA to every service's trust store.

### Scenario 4: "Is our TLS config compliant / secure?"
A pentest or PCI audit asks about your TLS posture. You know the answers: TLS 1.2 minimum, forward-secret cipher suites (ECDHE), no SSLv3/TLS1.0/1.1, valid unexpired certs with correct SANs, full chains served. You can prove each with a one-line `openssl` or `curl` command instead of guessing.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the end-to-end picture; TLS is Step 3 of the request lifecycle
- `08-tcp-in-depth.md` — TLS runs on top of TCP; the TCP handshake must finish before TLS starts

**Study next (in order):**
- `10-http1.1-in-depth.md` — the HTTP that rides *inside* the TLS tunnel
- `11-http2.md` — negotiated via ALPN (`h2`) during the TLS handshake
- `12-http3-and-quic.md` — where TLS 1.3 is folded directly into the transport (0-RTT setup)

**This topic connects forward to:**
- `16-authentication-over-http.md` — TLS protects the tokens/cookies; it is the layer *beneath* app-level auth
- `27-debugging-a-slow-api.md` — `time_appconnect` is the TLS slice of your latency budget
- `28-load-balancers.md` — where TLS is commonly *terminated* before traffic reaches your app
- `36-service-to-service-communication.md` — mutual TLS (mTLS) for zero-trust internal networks

---

*Doc saved: `/docs/networking/09-tls-ssl-in-depth.md`*
