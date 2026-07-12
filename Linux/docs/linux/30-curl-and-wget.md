# 30 — curl and wget in Depth

## ELI5 — The Simple Analogy

Imagine you want something from a warehouse across town.

**wget is a delivery van.** You give it an address, it drives there, picks up the box, and **puts it in your garage**. If the road is blocked it waits and tries again. If it got halfway through loading and the power went out, it comes back and *resumes from where it stopped*. It can even be told "bring me everything in aisle 7 and all the sub-shelves" and it will make trip after trip. It is a **fetcher**. You use it to get *things*.

**curl is a conversation.** curl walks up to the warehouse counter, hands over a form ("I'd like item 42, here's my ID badge, here's my membership number, I accept it in JSON"), listens to exactly what the clerk says back, and then **reads the entire exchange out loud to you in the street** — the form it handed over, the clerk's reply, the receipt, the timestamps. By default it doesn't even put the item in your garage; it just holds it up so you can look at it. It is a **conversationalist**. You use it to *talk to* a server and understand what it says.

That's the whole distinction:

- **wget writes to a FILE by default.** Its job is to *get things*.
- **curl writes to STDOUT by default.** Its job is to *have a conversation you can inspect and pipe*.

When you're debugging a 502 from nginx at 2am, you want the conversationalist.
When you're downloading a 4GB release tarball over a flaky connection, you want the van.

---

## Where This Lives in the Linux Stack

```
Hardware (NIC)
  └── KERNEL
       ├── netfilter (topic 29 — can this packet even leave/arrive?)
       ├── TCP/IP stack — the kernel builds the TCP connection
       └── Sockets
            │
            └── System Calls: socket() → connect() → write() → read() → close()
                 │   (plus getaddrinfo() → which reads /etc/nsswitch.conf,
                 │    /etc/hosts, /etc/resolv.conf → topic 27, DNS)
                 │
                 └── C Library (glibc) + libssl/OpenSSL (the TLS handshake)
                      │
                      └── libcurl ◀◀◀ the actual engine. A C library.
                          │           Node's `undici`, PHP, Python's pycurl —
                          │           half the internet links against this.
                          │
                          └── Shell
                               │
                               └── curl / wget ◀◀◀ THIS TOPIC
                                   (thin CLI wrappers over the library)
```

**Key insight:** `curl` the command is a ~30KB wrapper around `libcurl`, which is the real thing. When you run `curl -v`, you are watching a *library* narrate its own socket calls. That's why `-v` output is so precise — it isn't a log, it's the actual protocol.

---

## What Is This?

`curl` is a data-transfer tool. It speaks HTTP, HTTPS, FTP, SFTP, SCP, SMTP, IMAP, LDAP, and ~20 other protocols. It writes to stdout, so it composes with pipes (topic 12). It shows you the raw protocol. **It is the single most important debugging tool a backend developer has**, because it lets you make *exactly* the request you want and see *exactly* what came back — with no browser, no framework, and no client library in between to lie to you.

`wget` is a non-interactive network downloader. It writes to files, resumes broken transfers, retries robustly, and can recursively mirror a site. It is the right tool for "fetch this tarball onto the server."

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `-d` implies `Content-Type: x-www-form-urlencoded` | Your JSON API returns 400 and you'll spend an hour blaming your API code |
| curl does not follow redirects by default | You'll get an empty body, a 301, and conclude "the endpoint is broken" |
| curl exits **0** on an HTTP 500 | Your deploy script's health check will "pass" against a completely broken server, and you'll ship the outage |
| `-w '%{time_*}'` timing breakdown | "The API is slow" will stay a mystery instead of "DNS takes 4 seconds because one resolver is dead" |
| No `--max-time` in scripts | A hung request will pin your cron job forever, and tomorrow's cron will pile on top of it |
| 502 vs 504 vs 499 | You'll debug the wrong layer. 502 = your Node app is *down*. 504 = your Node app is *slow*. 499 = the *client* left |
| `-v` reading | You'll be unable to answer "did the header actually get sent?" — which is 80% of auth bugs |
| Putting a token in the URL | Your API key ends up in nginx access logs, shell history, and a Grafana dashboard |

---

## The Physical Reality — What Actually Happens on the Wire

```
COMMAND: curl -v https://api.example.com/users/42

  1. curl parses the URL:  scheme=https  host=api.example.com  path=/users/42  port=443 (implied)

  2. DNS RESOLUTION  ────────────────────────────────► getaddrinfo("api.example.com")
     reads /etc/nsswitch.conf → /etc/hosts → /etc/resolv.conf → UDP :53 to the resolver
     ◀── 93.184.216.34                          [ time_namelookup ends here ]

  3. TCP HANDSHAKE   ─── socket() → connect() ───────► SYN
                                                  ◀── SYN-ACK
                                                  ───► ACK
                                                 [ time_connect ends here ]

  4. TLS HANDSHAKE   ───────────────────────────────► ClientHello (SNI: api.example.com)
                                                  ◀── ServerHello + Certificate
     curl VERIFIES the cert chain against /etc/ssl/certs/ca-certificates.crt
     (-k/--insecure SKIPS THIS STEP — see the warning below)
                                                  ───► Finished
                                                 [ time_appconnect ends here ]

  5. HTTP REQUEST    ───────────────────────────────►
     ┌──────────────────────────────────────────┐
     │ GET /users/42 HTTP/1.1        ← request line: METHOD PATH VERSION
     │ Host: api.example.com         ← headers
     │ User-Agent: curl/7.81.0       │
     │ Accept: */*                   │
     │                               ← BLANK LINE (this is what ends the headers)
     │ (no body for GET)             ← body
     └──────────────────────────────────────────┘
                                                 [ time_pretransfer ends here ]

  6. SERVER THINKS ... (queries the DB, runs your Express handler)

  7. HTTP RESPONSE   ◀───────────────────────────────
     ┌──────────────────────────────────────────┐
     │ HTTP/1.1 200 OK               ← status line
     │ Content-Type: application/json│
     │ Content-Length: 47            │
     │                               ← BLANK LINE
     │ {"id":42,"name":"Ada"}        ← body  [ time_starttransfer = FIRST BYTE = TTFB ]
     └──────────────────────────────────────────┘
                                                 [ time_total ends here ]
  8. curl writes the BODY to fd 1 (stdout). Headers go nowhere unless you ask (-i / -v).
```

**Burn this into your head:** an HTTP message is `start line` + `headers` + `blank line` + `body`. That's it. That's the whole protocol. Everything curl does is constructing that top half and showing you the bottom half.

---

## How It Works — Step by Step

```
COMMAND:  curl -sS -f -X POST https://api.example.com/orders \
               -H "Authorization: Bearer $TOKEN" \
               -H "Content-Type: application/json" \
               -d '{"item":"widget","qty":2}'

SHELL DOES:    expands $TOKEN from the environment (topic 14), forks, execs /usr/bin/curl
CURL DOES:     parses argv; -d sets the body AND flips the method to POST (so -X POST is redundant)
               -d also sets Content-Type: application/x-www-form-urlencoded...
               ...but your explicit -H OVERRIDES it. This is the crux of the #1 curl mistake.
SYSCALLS:      getaddrinfo() → socket(AF_INET, SOCK_STREAM) → connect() →
               [TLS: many write()/read() pairs] → write(fd, "POST /orders HTTP/1.1\r\n...")
               → read(fd, ...) → write(1, body, n) → close(fd)
KERNEL DOES:   builds the TCP segments, runs them through netfilter's OUTPUT chain (topic 29),
               tracks the connection in conntrack, hands them to the NIC driver
OUTPUT:        the response body → stdout (fd 1). Errors → stderr (fd 2).
EXIT CODE:     0 if the transfer worked. With -f: 22 if the server said 4xx/5xx.
```

**The three things beginners get wrong right there:**
1. `-X POST` is redundant when you use `-d`.
2. Without that explicit `Content-Type` header, the server gets `x-www-form-urlencoded` and your Express `express.json()` middleware doesn't parse the body → `req.body` is `{}` → 400.
3. Without `-f`, if the server returns 500, curl prints the error page and **exits 0** — and your deploy script marches on.

---

## Exact Syntax Breakdown

### The POST that every backend dev needs

```
curl -sS -f -X POST https://api.example.com/orders \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"item":"widget","qty":2}' \
     --max-time 10
│    │  │  │       │                 │                          │           │
│    │  │  │       │                 │                          │           └── kill the whole
│    │  │  │       │                 │                          │               transfer after 10s.
│    │  │  │       │                 │                          │               NEVER omit in a script.
│    │  │  │       │                 │                          └── the request BODY.
│    │  │  │       │                 │                              -d IMPLIES: method=POST, and
│    │  │  │       │                 │                              Content-Type: x-www-form-urlencoded
│    │  │  │       │                 │                              (which our -H above overrides)
│    │  │  │       │                 └── a request header. REPEATABLE — use -H as many times as needed.
│    │  │  │       │                     The secret goes HERE, not in the URL.
│    │  │  │       └── the URL. https → curl will do a full TLS handshake + cert verification.
│    │  │  └── -X = set the HTTP method explicitly. REDUNDANT here (-d already made it POST).
│    │  │      ⚠️ `-X GET -d '...'` is a FOOTGUN: you get a GET *with a body*, which many
│    │  │         servers and proxies silently discard. Use -G instead (see below).
│    │  └── -f/--fail = exit NON-ZERO (22) on HTTP >= 400. In a script this is MANDATORY.
│    │          Without it, curl exits 0 on a 500 and your CI thinks the deploy worked.
│    └── -S/--show-error = still print errors to stderr even though -s silenced everything.
└── -s/--silent = no progress meter, no stats. (-sS is THE correct scripting combo.)
```

### Reading `curl -v` — the debugging workhorse

```
curl -v https://api.example.com/health

* Host api.example.com:443 was resolved.               ← "*" = curl's own commentary.
* IPv4: 93.184.216.34                                       DNS, TCP, TLS — all the plumbing.
*   Trying 93.184.216.34:443...
* Connected to api.example.com (93.184.216.34) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: CN=api.example.com
*  start date: Jun  1 00:00:00 2026 GMT
*  expire date: Aug 30 23:59:59 2026 GMT     ← ⚠️ check this when TLS "suddenly breaks"
*  issuer: C=US; O=Let's Encrypt; CN=R3
*  SSL certificate verify ok.
> GET /health HTTP/2                          ← ">" = what curl SENT. The request.
> Host: api.example.com                            If your header isn't in a ">" line,
> user-agent: curl/8.5.0                           IT WAS NEVER SENT. This settles 80%
> accept: */*                                      of "but I set the auth header!" bugs.
>
< HTTP/2 200                                  ← "<" = what the server SENT BACK. The response.
< content-type: application/json
< x-request-id: 7f3a-91bc                          ← grab this for your log search
< server: nginx/1.24.0
<
{"status":"ok","uptime":91223}                ← the BODY (no prefix — it's the payload)
* Connection #0 to host api.example.com left intact
```

**The three prefixes are the whole skill:**

| Prefix | Meaning | Use it for |
|---|---|---|
| `*` | curl's own narration | DNS, TCP, TLS, cert expiry, connection reuse |
| `>` | **Sent** to the server | "Did my `Authorization` header actually go out?" |
| `<` | **Received** from the server | Status, `Location:` on a redirect, `Set-Cookie`, `Retry-After` |

### -i vs -I vs -v — three different things people confuse

```
curl -i URL     Print RESPONSE HEADERS + BODY to stdout. Still a GET. Pipeable.
                → Use when you want the headers as *data*.

curl -I URL     Send a HEAD request. Server returns headers ONLY, no body.
                → Use to check Content-Length / Content-Type / a redirect target
                  WITHOUT downloading a 2GB file.
                ⚠️ Some APIs don't implement HEAD and return 405. Then use -sI... or -o /dev/null -D -.

curl -v URL     Print the FULL exchange (request headers too) — but to STDERR.
                → Use for debugging, not for piping. The body still goes to stdout,
                  so `curl -v URL | jq` actually works fine.
```

### The redirect trap

```
curl https://example.com/old
# (empty output. exit code 0. "the endpoint is broken?!")

curl -i https://example.com/old
# HTTP/1.1 301 Moved Permanently
# Location: https://example.com/new           ← AH. There's the answer.
# Content-Length: 0
#                                             ← the body IS empty. curl told the truth.

curl -L https://example.com/old
│    │
│    └── -L/--location = FOLLOW the Location: header. Curl does NOT do this by default.
│        --max-redirs 5   ← cap it, or a redirect loop spins forever
│        --location-trusted ← also resend auth headers across hosts (⚠️ leaks credentials — rare)
└── the #1 "why does my curl return nothing" bug, solved.
```

### -d variants — the data flags, precisely

```
-d 'a=1&b=2'            urlencoded body. Implies POST + Content-Type: x-www-form-urlencoded.
-d @payload.json        read the body from a FILE. ⚠️ STRIPS NEWLINES AND CARRIAGE RETURNS.
                        Harmless for JSON. FATAL for a signed payload or a binary blob.
--data-raw 'a=1'        same as -d but does NOT treat a leading @ as a filename.
                        (use this when your literal data starts with @, e.g. an email)
--data-binary @file.json  read the file BYTE FOR BYTE. No newline stripping. ◀── the correct
                          way to POST a JSON file, and the only way for binary.
--data-urlencode 'q=a b&c'  URL-encode the value for you. Great for query params with spaces.
--json '{"a":1}'        modern shorthand (curl 7.82+). Equivalent to:
                          --data-binary '{"a":1}'
                          -H 'Content-Type: application/json'
                          -H 'Accept: application/json'
                        ◀── just use this for JSON APIs on a modern curl.
-F 'file=@./avatar.png' multipart/form-data. THE way to upload a file.
                        -F 'user=ada' -F 'file=@./a.png;type=image/png'
-G                      take the -d data and put it in the QUERY STRING instead of the body,
                        and use GET. → `curl -G -d 'q=linux' -d 'page=2' https://x/search`
                        results in GET /search?q=linux&page=2
```

### Timing: the poor man's APM

```
curl -o /dev/null -s -w '%{http_code} %{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total}\n' https://api.example.com/orders
     │            │  │
     │            │  └── -w/--write-out: print these VARIABLES after the transfer, to stdout.
     │            │      They are filled in by libcurl from its own internal timers.
     │            └── -s: kill the progress bar so the output is clean
     └── -o /dev/null: throw the BODY away. We only want the timings.
```

Every timing variable is **cumulative from the start of the transfer**, so you get the phases by subtraction:

```
 0 ──── time_namelookup ──── time_connect ──── time_appconnect ──── time_starttransfer ──── time_total
 │           │                    │                 │                       │                     │
 │◀── DNS ──▶│                    │                 │                       │                     │
 │           │◀─── TCP handshake ▶│                 │                       │                     │
 │           │                    │◀─ TLS handshake▶│                       │                     │
 │           │                    │                 │◀── SERVER THINK TIME ▶│                     │
 │           │                    │                 │      (this is TTFB)   │◀── downloading ────▶│
 │           │                    │                 │                       │                     │
 └───────────┴────────────────────┴─────────────────┴───────────────────────┴─────────────────────┘
     "DNS is         "the network       "TLS is           "THE APP IS SLOW"      "the response
      slow"           is slow /          slow / cert       ← your problem.        is huge / the
                      the server         chain is                                 network is
                      is far away"       big"                                     slow"
```

**This single command tells you, in 200ms, whether "the API is slow" is DNS, the network, TLS, the app, or the payload size.** It is the highest-value thing in this entire document. Make a template file:

```bash
cat > ~/curl-format.txt <<'EOF'
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
   time_pretransfer:  %{time_pretransfer}s\n
      time_redirect:  %{time_redirect}s\n
 time_starttransfer:  %{time_starttransfer}s\n
                    ----------\n
         time_total:  %{time_total}s\n
          http_code:  %{http_code}\n
       size_download:  %{size_download} bytes\n
          remote_ip:  %{remote_ip}\n
EOF

curl -w "@$HOME/curl-format.txt" -o /dev/null -s https://api.example.com/orders
#     time_namelookup:  4.021041s      ◀── ⚠️ FOUR SECONDS OF DNS.
#        time_connect:  4.043112s          The app is fine. Your resolver is dying.
#     time_appconnect:  4.098821s          Go look at /etc/resolv.conf (topic 27).
#    time_pretransfer:  4.098901s
#       time_redirect:  0.000000s
#  time_starttransfer:  4.121004s
#                     ----------
#          time_total:  4.122880s
#           http_code:  200
#       size_download:  1847 bytes
#           remote_ip:  93.184.216.34
```

---

## curl — The Full Flag Reference, Annotated

| Flag | What it does | The gotcha |
|---|---|---|
| `-X METHOD` | Set the HTTP method | Redundant with `-d`. `-X GET -d` is a footgun — use `-G` |
| `-d DATA` | Request body | **Implies POST + urlencoded Content-Type.** Add `-H 'Content-Type: application/json'` for JSON |
| `-d @file` | Body from file | **Strips newlines.** Use `--data-binary @file` |
| `--data-binary @f` | Body from file, byte-exact | The correct one for JSON files and binaries |
| `--json '{}'` | JSON shorthand | curl 7.82+. Sets Content-Type AND Accept |
| `-F k=v` / `-F 'f=@x'` | multipart/form-data | The only way to upload a file to a form endpoint |
| `-G` | Turn `-d` into a query string | For GETs with params |
| `-H 'K: V'` | Add a header | Repeatable. `-H 'K:'` *removes* a default header |
| `-i` | Include response headers in stdout | Body still there — pipeable |
| `-I` | HEAD request (headers only) | Some servers 405 on HEAD |
| `-v` | Full exchange to **stderr** | `>` sent, `<` received, `*` curl's commentary |
| `--trace-ascii -` | Even deeper: full byte dump | When `-v` isn't enough (protocol-level weirdness) |
| `-L` | Follow redirects | **NOT the default.** `--max-redirs N` |
| `-o FILE` | Write body to a file | `-o /dev/null` to discard |
| `-O` | Write to a file named after the remote path | `curl -O https://x/y/app.tar.gz` → `./app.tar.gz` |
| `-s` | Silent (no progress, no errors) | Hides errors too — always pair with `-S` |
| `-S` | Show errors even when silent | **`-sS` is the scripting combo** |
| `-f` / `--fail` | Exit 22 on HTTP >= 400 | **Without this, curl exits 0 on a 500** |
| `--fail-with-body` | Same, but still print the error body | Use when the API returns useful JSON errors |
| `-u user:pass` | HTTP Basic auth | Ends up in `ps` output — prefer `-u user` and let curl prompt, or a `.netrc` |
| `--oauth2-bearer TOK` | Bearer token | Same as `-H "Authorization: Bearer TOK"` |
| `-b` / `-c` | Send / save cookies | `-c jar.txt` then `-b jar.txt` for a login flow |
| `-k` / `--insecure` | **Disable TLS cert verification** | See the stern warning below |
| `--cacert FILE` | Trust a specific CA bundle | **The right fix** for a self-signed/internal CA |
| `--resolve H:P:IP` | Fake a DNS entry for this request | Test a new server **before** you cut DNS over |
| `--connect-timeout N` | Give up if TCP connect takes > N sec | |
| `--max-time N` | Give up on the WHOLE transfer after N sec | **Always set in scripts and cron** |
| `--retry N` | Retry on transient failures | `--retry-delay`, `--retry-max-time` |
| `--retry-all-errors` | Also retry on things curl normally won't | Pair with `-f` to retry 5xx |
| `--compressed` | Send `Accept-Encoding: gzip` and decompress | Cuts transfer time on big JSON |
| `-A 'agent'` | Set User-Agent | Some APIs/CDNs reject the default `curl/8.x` |
| `-x http://proxy:8080` | Use a proxy | Also honors `HTTP_PROXY`/`HTTPS_PROXY` env vars |
| `--http2` / `--http1.1` | Force a protocol version | Useful when debugging an h2-specific bug |
| `-Z` | Parallel transfers | With multiple URLs |
| `-w FMT` | Print variables after the transfer | The timing breakdown. See above |
| `-D FILE` | Dump response headers to a file | `-D -` sends them to stdout |

### `-k` / `--insecure` — read this before you type it

```bash
curl -k https://internal-api.corp/health     # "it works now!"
```

**What you actually just turned off:** curl no longer checks that (a) the certificate is signed by a CA you trust, (b) the hostname on the cert matches the hostname you asked for, (c) the cert hasn't expired. **You are now vulnerable to a trivial man-in-the-middle.** Anyone on the path can present any cert and curl will happily send them your `Authorization: Bearer` token.

`-k` is acceptable for a 10-second local diagnostic. It is **never** acceptable in a script, a Dockerfile, a CI job, or anything that carries a secret.

**The right fixes:**
```bash
# 1. Install your org's CA into the system trust store (best):
sudo cp corp-ca.crt /usr/local/share/ca-certificates/corp-ca.crt
sudo update-ca-certificates
# → now plain `curl https://internal-api.corp` just works. Forever. For everything.

# 2. Or point curl at the CA for this request:
curl --cacert /path/to/corp-ca.crt https://internal-api.corp/health

# 3. If the error is "certificate has expired" — the fix is to renew the cert,
#    not to stop checking it. `curl -vI` shows you the expiry date.
```

### `--resolve` — the professional's DNS-cutover trick

You've stood up a new server at `203.0.113.99` and you want to test that vhost `www.example.com` works there — **before** you point DNS at it.

```bash
curl --resolve www.example.com:443:203.0.113.99 https://www.example.com/health -v
│      │            │           │
│      │            │           └── the IP to actually connect to
│      │            └── the port this override applies to
│      └── the hostname to override
└── curl skips DNS for this host:port and uses YOUR IP.
    But it STILL sends `Host: www.example.com` and STILL uses that name for
    TLS SNI and cert verification.
```

This is exactly what a real client will do after the DNS change — same Host header, same SNI, same cert check — **against the new box**. If it returns 200, your cutover is safe. If it 404s, you have a vhost misconfiguration and you just avoided a 20-minute outage. Editing `/etc/hosts` does the same thing but is stateful and easy to forget to undo; `--resolve` is per-request.

---

## Exit Codes — Use Them or Your Scripts Are Lies

```bash
curl -sS -f --max-time 10 https://api.example.com/health
echo "exit: $?"
```

| Code | Meaning | What it tells you |
|---|---|---|
| **0** | OK | The transfer succeeded. **Note: without `-f`, this includes a 500 response.** |
| **6** | Couldn't resolve host | DNS. Check `/etc/resolv.conf`, `dig` (topic 27) |
| **7** | Failed to connect | TCP refused or unreachable. The port is closed, the app is down, or the **firewall REJECTed** (topic 29) |
| **22** | HTTP error (only with `-f`) | The server answered, with 4xx/5xx |
| **28** | Operation timed out | `--connect-timeout` or `--max-time` fired. Or the **firewall DROPped** you (topic 29 — DROP hangs, REJECT gives you a 7) |
| **35** | SSL connect error | TLS handshake failed. Protocol/cipher mismatch |
| **56** | Failure receiving network data | Connection reset mid-transfer |
| **60** | Peer certificate cannot be authenticated | Untrusted CA, expired cert, or hostname mismatch. **This is what `-k` silences** |

**Exit 7 vs exit 28 is a free firewall diagnosis.** A REJECT gives you an instant connection-refused (7). A DROP gives you a timeout (28). You can tell which kind of block you hit without ever logging into the server.

---

## Example 1 — Basic

```bash
# ── The absolute basics ─────────────────────────────────────────────
curl https://api.github.com/zen
# "Design for failure."
# ↑ body goes to stdout. No headers. No file written.

# See the headers too
curl -i https://api.github.com/zen
# HTTP/2 200
# content-type: text/plain;charset=utf-8
# x-ratelimit-remaining: 59
#
# "Approachable is better than simple."

# Just the headers — no body downloaded at all
curl -I https://api.github.com/zen

# Watch the FULL conversation (request AND response)
curl -v https://api.github.com/zen 2>&1 | head -30

# ── Save it ─────────────────────────────────────────────────────────
curl -o zen.txt https://api.github.com/zen     # name it yourself
curl -O https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-x64.tar.xz  # use remote name

# ── POST JSON correctly ─────────────────────────────────────────────
# ✗ WRONG — server sees Content-Type: application/x-www-form-urlencoded → 400
curl -d '{"name":"ada"}' https://api.example.com/users

# ✓ RIGHT — explicitly declare JSON
curl -H 'Content-Type: application/json' -d '{"name":"ada"}' https://api.example.com/users

# ✓ RIGHT (modern, curl 7.82+) — one flag does both
curl --json '{"name":"ada"}' https://api.example.com/users

# ✓ RIGHT — body from a file, byte-exact
curl -H 'Content-Type: application/json' --data-binary @payload.json https://api.example.com/users

# ── Auth: the token goes in a HEADER, never the URL ─────────────────
export TOKEN='eyJhbGciOi...'
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/me
# ✗ NEVER: curl 'https://api.example.com/me?token=eyJhbG...'
#   → that URL lands in: nginx access logs, your ~/.bash_history, the CDN's logs,
#     any Referer header, and the browser address bar. Treat it as a leaked credential.

# ── Redirects ───────────────────────────────────────────────────────
curl -L --max-redirs 5 http://example.com     # follow, but not into a loop

# ── Pipe to jq ──────────────────────────────────────────────────────
curl -s https://api.example.com/orders | jq '.data[] | select(.status=="error") | .id'
# "ord_9f2a"
# "ord_31bc"
# ↑ -s is essential: without it the progress meter corrupts the pipe. Well — it goes to
#   stderr so jq survives, but it garbles your terminal. Always -s when piping.

# ── wget: the downloader ────────────────────────────────────────────
wget https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-x64.tar.xz
# node-v20.11.0-linux-x64.tar.xz  100%[==========>]  22.5M  8.2MB/s  in 2.8s
# ↑ writes to a FILE, shows a progress bar. curl would have vomited binary into your terminal.

# The killer wget feature: RESUME a broken download
wget -c https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-x64.tar.xz
# ↑ -c/--continue. Sends `Range: bytes=8388608-`. Picks up exactly where it stopped.
#   This is why you use wget for big files on a flaky connection.
```

---

## Example 2 — Production Scenario

**2:41am.** Alerts: `api.yourco.com` is returning **502 Bad Gateway**. You SSH in. Architecture is: internet → nginx (:443) → Node app (:3000) → Postgres.

```bash
# ═══ 1. Confirm it from the OUTSIDE, and get the timing ═══════════════════
curl -o /dev/null -s -w 'code=%{http_code} total=%{time_total}s ttfb=%{time_starttransfer}s\n' \
     https://api.yourco.com/health
# code=502 total=0.094s ttfb=0.093s
#
# ★ 502 + FAST. That combination is the diagnosis, right there.
#   502 = nginx could not talk to its upstream. And it failed in 90ms —
#   nginx got an instant connection-REFUSED from :3000. It didn't time out.
#   → The Node process is DOWN or not listening. Not slow. DOWN.
#
#   (If it were 504 + ~60s, the story would be the opposite: Node is UP but
#    hung, and nginx gave up after proxy_read_timeout.)

# ═══ 2. Prove it: bypass nginx and hit the app directly ══════════════════
curl -sS -o /dev/null -w '%{http_code}\n' --max-time 5 http://127.0.0.1:3000/health
# curl: (7) Failed to connect to 127.0.0.1 port 3000 after 0 ms: Connection refused
#
# ★ Exit code 7. Connection REFUSED — instantly.
#   Nothing is listening on 3000. This is not a firewall (a firewall DROP would
#   have given us exit 28, a timeout). The socket simply does not exist.
#   → nginx is innocent. The app is dead. CONFIRMED in two commands.

# ═══ 3. Cross-check with the socket table (topic 28) ═════════════════════
ss -tulpn | grep 3000
# (no output — nothing is listening. Confirmed.)

# ═══ 4. Why did it die? (topic 22, 23) ═══════════════════════════════════
systemctl status api --no-pager | head -8
# ● api.service - Node API
#    Active: failed (Result: oom-kill) since Sun 2026-07-12 02:39:41 UTC; 2min ago
#                    ^^^^^^^^^^^^^^^^ the OOM killer got it (topic 01, 18)

# ═══ 5. Restart, and VERIFY WITH THE RIGHT CURL ══════════════════════════
sudo systemctl restart api

# ✓ A health check that actually checks: -f fails on 5xx, --retry rides out the boot,
#   --max-time bounds it, -sS is quiet but reports real errors.
curl -sS -f --retry 10 --retry-delay 2 --retry-all-errors \
     --connect-timeout 3 --max-time 5 \
     http://127.0.0.1:3000/health && echo "APP OK" || echo "APP STILL DOWN (exit $?)"
# {"status":"ok"}APP OK

# ═══ 6. Now through the whole stack, with the full timing breakdown ══════
curl -w "@$HOME/curl-format.txt" -o /dev/null -s https://api.yourco.com/health
#     time_namelookup:  0.004211s
#        time_connect:  0.021880s     (17ms of TCP — fine)
#     time_appconnect:  0.062044s     (40ms of TLS — fine)
#  time_starttransfer:  0.089301s     (27ms of server think time — the app is healthy)
#          time_total:  0.089510s
#           http_code:  200            ◀── ✓ RESOLVED
```

### The nginx status triad — memorize this

You saw it above. It is the single most useful piece of knowledge for a Node dev behind a reverse proxy:

```
   client ────▶ nginx ────▶ your Node app (:3000)

┌────────┬──────────────────────────────────────────────┬───────────────────────────┐
│ Status │ What nginx is telling you                     │ Where the failure IS       │
├────────┼──────────────────────────────────────────────┼───────────────────────────┤
│  502   │ "I tried to connect to the upstream and it    │ ★ YOUR NODE APP IS DOWN.   │
│  Bad   │  REFUSED / RESET / wasn't there."             │   Crashed, OOM-killed,     │
│Gateway │  Comes back FAST (milliseconds).              │   not listening, wrong     │
│        │                                              │   port. Check systemctl.   │
├────────┼──────────────────────────────────────────────┼───────────────────────────┤
│  504   │ "I connected fine, sent the request, and the  │ ★ YOUR NODE APP IS SLOW.   │
│Gateway │  upstream never answered in time."            │   Blocked event loop, a    │
│Timeout │  Comes back SLOW (== proxy_read_timeout,      │   hanging DB query, an     │
│        │  default 60s).                                │   upstream API with no     │
│        │                                              │   timeout. It is UP.       │
├────────┼──────────────────────────────────────────────┼───────────────────────────┤
│  499   │ "The CLIENT closed the connection before I    │ ★ THE CLIENT GAVE UP.      │
│(nginx- │  could answer." (nginx's own non-standard     │   Usually because your app │
│specific│  code — appears in access.log, never sent     │   is slow and the caller's  │
│ )      │  to a client, because there's no one left)    │   timeout is shorter than  │
│        │                                              │   yours. A 499 flood == a  │
│        │                                              │   latency problem.         │
└────────┴──────────────────────────────────────────────┴───────────────────────────┘
```

**502 = down. 504 = slow. 499 = the caller left.** And the `time_total` from `-w` tells you which one you're in before you even read the number.

---

## HTTP Status Codes a Backend Dev Must Recognize Instantly

| Class | Meaning |
|---|---|
| **2xx** | Success |
| **3xx** | Go somewhere else |
| **4xx** | **YOU broke it** (the client sent something wrong) |
| **5xx** | **THE SERVER broke** (your code, or something it depends on) |

| Code | Name | What it actually means to you |
|---|---|---|
| 200 / 201 / 204 | OK / Created / No Content | 204 = success, and **there is no body** — don't `jq` it |
| **301** | Moved Permanently | Cached by browsers **forever**. `-L` follows. **⚠️ Old clients may change POST → GET when following** |
| **302** | Found | Temporary. **Same POST → GET method-change quirk** (a historical browser bug that got codified) |
| **307** | Temporary Redirect | Like 302 but the **method and body are PRESERVED**. A POST stays a POST |
| **308** | Permanent Redirect | Like 301 but **method preserved**. Use 307/308 for APIs; 301/302 only for browsers |
| **401** | Unauthorized | **"I don't know who you are."** Missing/invalid/expired credentials. Re-auth might help |
| **403** | Forbidden | **"I know exactly who you are, and you're not allowed."** Re-auth will NOT help |
| **404** | Not Found | The route doesn't exist. Also, sometimes, a 403 in disguise (hiding resource existence) |
| **405** | Method Not Allowed | The route exists but not for this verb. **Classic symptom of `curl -I` against an API that only handles GET** |
| **413** | Payload Too Large | nginx `client_max_body_size` (default **1 MB**) or Express `limit`. Your file upload dies here |
| **429** | Too Many Requests | Rate-limited. **Read the `Retry-After` header** and honor it. `curl -i` to see it |
| **499** | (nginx) Client Closed Request | The caller hung up first |
| **500** | Internal Server Error | Your unhandled exception |
| **502** | Bad Gateway | **The upstream is DOWN / refused the connection** |
| **503** | Service Unavailable | Overloaded or in maintenance. Often has `Retry-After` |
| **504** | Gateway Timeout | **The upstream is UP but TOO SLOW** — exceeded `proxy_read_timeout` |

**The method-change quirk, concretely:**
```bash
# Server 301s /api/v1/orders → /api/v2/orders
curl -L -X POST -d '{"a":1}' https://x/api/v1/orders
# → curl follows the 301 but downgrades to GET, drops your body.
# → the server sees GET /api/v2/orders and returns 405 or a list.
# → you spend an hour convinced your POST is broken.
# FIX: server should use 308. Client workaround: curl's --post301 / --post302 / --post303
curl -L --post301 -X POST -d '{"a":1}' https://x/api/v1/orders
```

---

## wget — The Downloader

```
wget -c -q --show-progress --tries=5 --timeout=30 -P /opt/releases https://cdn.co/app-v2.tar.gz
│    │  │  │                │          │            │
│    │  │  │                │          │            └── -P = the DIRECTORY to save into
│    │  │  │                │          └── give up on a stalled read after 30s
│    │  │  │                └── retry 5 times on failure (0 = infinite)
│    │  │  └── keep the progress bar even in quiet mode
│    │  └── -q = quiet. (-nv = "not verbose": one summary line. Perfect for CI logs.)
│    └── -c = CONTINUE. Resume a partial download via an HTTP Range request.
│           ◀── THE reason to reach for wget on a 4GB file over a flaky link.
└── the wget binary
```

| Flag | What it does |
|---|---|
| `-O file` | Write to this filename. **`-O -` writes to stdout** (wget pretending to be curl) |
| `-c` | **Continue/resume** a partial download |
| `-q` | Quiet. `-nv` = one summary line — the right level for a deploy script |
| `-P dir` | Save into this directory |
| `--tries=N` | Retries (default 20). `--timeout=N` sets connect+read+dns timeouts at once |
| `--spider` | Don't download — just check the URL exists. A poor-man's link checker |
| `-r` | Recursive (mirror a site) |
| `-np` | `--no-parent` — don't climb above the starting directory. **Essential with `-r`** |
| `-k` | `--convert-links` — rewrite links for offline browsing |
| `-l N` | Recursion depth |

```bash
# Mirror a docs site for offline reading
wget -r -np -k -l 3 -P ./docs-mirror https://docs.example.com/guide/

# Check a URL exists without downloading it
wget --spider -q https://cdn.co/app-v2.tar.gz && echo "exists" || echo "404"

# wget acting like curl (rarely what you want, but useful in a minimal container)
wget -qO- https://api.example.com/health | jq .
#      ││└── the URL
#      │└── "-" = stdout
#      └── -O + -q combined
```

**Container gotcha:** `alpine` images ship **BusyBox wget**, not GNU wget. It supports `-O`, `-q`, and basic fetching, but **not** `-c`, `--spider`, or `-r`. And many minimal images (`node:20-slim`, `debian:slim`) ship **neither** curl nor wget. Your `HEALTHCHECK` line will silently fail. Either `RUN apt-get install -y curl` or use Node itself: `HEALTHCHECK CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"`.

---

## Common Mistakes

### Mistake 1: POSTing JSON without `Content-Type` — the eternal 400

**Wrong:**
```bash
curl -X POST -d '{"name":"ada"}' http://localhost:3000/users
# {"error":"Invalid request body"}   ← and you go read your Express code for an hour
```

**Root cause:** `-d` sets `Content-Type: application/x-www-form-urlencoded`. Express's `express.json()` middleware **only parses bodies whose Content-Type is `application/json`**. It sees urlencoded, skips parsing, and `req.body` is `{}`. Your validator rejects it. **Your API is fine. Your curl is wrong.**

**Prove it in one command:**
```bash
curl -v -d '{"name":"ada"}' http://localhost:3000/users 2>&1 | grep -i content-type
# > Content-Type: application/x-www-form-urlencoded     ← ">" = what YOU sent. There it is.
```

**Right:**
```bash
curl -H 'Content-Type: application/json' -d '{"name":"ada"}' http://localhost:3000/users
# or, modern:
curl --json '{"name":"ada"}' http://localhost:3000/users
```

**Prevention:** make `--json` your default muscle memory for any API. And when an API returns 400, `curl -v | grep '^>'` **before** you open your editor.

---

### Mistake 2: No `-f` in a deploy health check — shipping a broken build

**Wrong:**
```bash
#!/usr/bin/env bash
set -euo pipefail
systemctl restart api
sleep 5
curl -s http://localhost:3000/health     # "checking the health"
echo "Deploy successful! 🎉"
# ... and it prints "Deploy successful" even when the app returns a 500 error page.
```

**Root cause:** **curl exits 0 whenever the HTTP transfer succeeded** — and a 500 response *is* a successful transfer. curl did its job: it delivered a request and received a reply. It doesn't care that the reply was terrible. So `set -e` never fires and your script marches happily past a completely broken server.

**What the broken state looks like:** green CI, green deploy log, and a production outage. Nobody knows for 20 minutes.

**Right:**
```bash
#!/usr/bin/env bash
set -euo pipefail

systemctl restart api

# -f            → exit 22 on any 4xx/5xx
# --retry 15    → keep trying while the app boots
# --retry-delay 2 --retry-all-errors → also retry on 5xx, not just connection errors
# --max-time 5  → bound each individual attempt
if curl -sS -f --retry 15 --retry-delay 2 --retry-all-errors \
        --connect-timeout 2 --max-time 5 \
        -o /dev/null http://127.0.0.1:3000/health; then
    echo "✓ health check passed"
else
    echo "✗ health check FAILED (curl exit $?) — rolling back"
    systemctl stop api
    ./rollback.sh
    exit 1
fi
```
(This is the health check your topic-13 deploy script should be calling.)

**Prevention:** grep your repo for `curl` inside any `.sh`, `Dockerfile`, or CI yaml, and make sure every single one has `-f` and `--max-time`. There is no legitimate reason to omit them in automation.

---

### Mistake 3: No timeouts in cron — the pileup

**Wrong:**
```cron
*/5 * * * * curl -s https://api.partner.com/sync >> /var/log/sync.log
```

**Root cause:** The partner's endpoint hangs (their firewall DROPs, or their LB is wedged). curl has **no default `--max-time`** — it will wait essentially forever. Cron fires again in 5 minutes and starts a *second* hung curl. Then a third. After a day you have 288 zombie curls, each holding a socket and a file descriptor (topic 19). Eventually you hit the fd limit and *other, unrelated things* start failing with `EMFILE: too many open files`. The postmortem takes hours because the symptom is nowhere near the cause.

**Right:**
```cron
*/5 * * * * /usr/bin/curl -sS -f --connect-timeout 5 --max-time 60 https://api.partner.com/sync >> /var/log/sync.log 2>&1
```
Even better, wrap it in `flock` so a slow run can never overlap the next one:
```cron
*/5 * * * * /usr/bin/flock -n /tmp/sync.lock /usr/local/bin/sync.sh
```

**Prevention:** `--connect-timeout` + `--max-time` on **every** curl that runs unattended. No exceptions. Ever.

---

### Mistake 4: `-k` as a reflex

**Wrong:** TLS error → add `-k` → it works → commit it → it's in production for three years.

**Root cause:** You didn't fix anything. You **disabled certificate verification**, which means curl no longer validates the CA chain, the hostname, or the expiry. Every request now trusts whatever certificate is presented — including one from an attacker on the network path. And you're sending a `Bearer` token over it.

**Diagnose the real problem first — `-v` tells you exactly which of the three failed:**
```bash
curl -v https://internal.corp/api 2>&1 | grep -Ei 'SSL|certificate|subject|expire|issuer'
# * SSL certificate problem: unable to get local issuer certificate
#   → an internal CA that's not in the system trust store. FIX: install the CA.
#
# * SSL: certificate subject name 'old.corp' does not match target host 'internal.corp'
#   → the cert is for the wrong hostname. FIX: reissue the cert (or use --resolve + the right name).
#
# *  expire date: Mar 12 00:00:00 2026 GMT
# * SSL certificate problem: certificate has expired
#   → FIX: RENEW THE CERT. This is a real, ticking production bug. -k hides it until
#     a client you don't control (a browser, a partner's server) starts failing too.
```

**Right:** install the CA (`update-ca-certificates`), fix the hostname, or renew the cert.

---

### Mistake 5: Assuming `-I` works on an API

**Wrong:**
```bash
curl -I https://api.example.com/orders
# HTTP/1.1 405 Method Not Allowed
# "the API is down!"
```

**Root cause:** `-I` sends a **HEAD** request, not a GET. Most frameworks route HEAD automatically, but plenty of hand-rolled API routers (and many Express routes with an explicit `app.get(...)` — which *does* handle HEAD, but a `app.post(...)`-only route does not) return **405**. The API is completely fine. You asked it a question it doesn't answer.

**Right — get the headers from a real GET, and throw the body away:**
```bash
curl -sS -o /dev/null -D - https://api.example.com/orders
#         │            │
#         │            └── -D - = dump response headers to stdout
#         └── discard the body
```

---

## Hands-On Proof

```bash
# PROVE IT: -d really does imply POST and set a Content-Type you didn't ask for
curl -v -d 'a=1' http://localhost:3000/anything 2>&1 | grep '^>'
# > POST /anything HTTP/1.1                       ← the method changed. You never said -X POST.
# > Content-Type: application/x-www-form-urlencoded  ← and there's the header you didn't set.

# PROVE IT: curl exits 0 on a 500 without -f
curl -s -o /dev/null -w '%{http_code}\n' https://httpbin.org/status/500 ; echo "exit=$?"
# 500
# exit=0        ◀── THE BUG. Your script would sail right past this.
curl -sS -f -o /dev/null https://httpbin.org/status/500 ; echo "exit=$?"
# curl: (22) The requested URL returned error: 500
# exit=22       ◀── now the script can actually fail.

# PROVE IT: curl does not follow redirects by default
curl -s -o /dev/null -w '%{http_code} %{num_redirects}\n' http://github.com
# 301 0        ← zero redirects followed. Empty body. This is the #1 confusion.
curl -sL -o /dev/null -w '%{http_code} %{num_redirects} → %{url_effective}\n' http://github.com
# 200 1 → https://github.com/

# PROVE IT: the timing breakdown is real — compare a near host to a far one
curl -o /dev/null -s -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer}\n' https://example.com

# PROVE IT: --resolve overrides DNS but keeps the Host header and SNI
curl -v --resolve example.com:443:93.184.216.34 https://example.com 2>&1 | grep -E 'Trying|Host:|subject:'
# *   Trying 93.184.216.34:443...      ← connected to the IP you forced
# *  subject: CN=example.com            ← but TLS still verified the real name
# > Host: example.com                   ← and the Host header is still correct

# PROVE IT: curl is really making socket syscalls
strace -f -e trace=socket,connect,sendto,write curl -s -o /dev/null http://example.com 2>&1 | head -8
# socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC, IPPROTO_TCP) = 5
# connect(5, {sa_family=AF_INET, sin_port=htons(80), sin_addr=...}, 16) = ...
# ↑ exactly the syscalls from the stack diagram at the top. Nothing magic.

# PROVE IT: a firewall DROP gives you exit 28; a REJECT gives you exit 7 (topic 29)
curl --connect-timeout 3 http://192.0.2.1:9999 ; echo "exit=$?"   # unroutable → 28 (timeout)
curl --connect-timeout 3 http://127.0.0.1:9999 ; echo "exit=$?"   # nothing listening → 7 (refused)

# PROVE IT: wget resumes; curl (by default) starts over
wget -O /tmp/big.iso https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso &
sleep 3 && kill %1
ls -l /tmp/big.iso                       # partial file, some MB
wget -c -O /tmp/big.iso https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso
# HTTP request sent, awaiting response... 206 Partial Content
#                                         ^^^^^^^^^^^^^^^^^^^ it sent a Range: header
# Length: 2104408064 (2.0G), 2098012160 (2.0G) remaining
#                            ^^^^^^^^^^^^^^^^^^^^^^^^^ resumed. Did not re-download.

# PROVE IT: 204 has no body — don't pipe it to jq
curl -s -o /dev/null -w '%{http_code} %{size_download}\n' -X DELETE https://api.example.com/x/1
# 204 0        ← zero bytes. `| jq .` on this gives you a parse error, not an API bug.
```

---

## Practice Exercises

### Exercise 1 — Easy

Against any public API (`https://httpbin.org` echoes your request back, which is perfect):

```bash
# 1. GET a URL and see ONLY the response headers.
# 2. GET it and see the FULL exchange — request headers AND response headers.
#    Identify which lines start with ">", "<", and "*", and say what each prefix means.
# 3. POST '{"hello":"world"}' as JSON, correctly. Then POST it INCORRECTLY (no
#    Content-Type) and use httpbin's echo to show exactly what the server received
#    in each case.
# 4. Hit http://github.com (plain http) with and without -L. Explain the difference
#    in the output and in %{http_code}.
```

---

### Exercise 2 — Medium

**Build the timing template and find the slow layer.**

```bash
# 1. Create ~/curl-format.txt with all six time_* variables plus http_code.
# 2. Time these three and explain WHICH PHASE dominates each, and why:
#      - https://example.com                 (a fast CDN-fronted static page)
#      - https://api.github.com/zen          (a real API, TLS, some think time)
#      - a URL on a host in another continent
# 3. Now force a slow DNS: temporarily add a dead nameserver as the FIRST entry
#    in /etc/resolv.conf (back it up!). Re-run. Show that time_namelookup
#    explodes while ttfb-minus-appconnect stays flat.
#    → This is exactly the shape of "the API got slow but the app didn't change."
# 4. Restore /etc/resolv.conf.
```

---

### Exercise 3 — Hard (Production Simulation)

**Write the deploy health-check that a real rollout depends on.**

```bash
# Requirements — a single bash script, `healthcheck.sh <url>`:
#
#  1. Poll <url> until it returns 200, for up to 60 seconds total.
#  2. Use -f so a 500 is a FAILURE, not a success. (This is the whole point.)
#  3. Bound every individual attempt with --connect-timeout and --max-time.
#  4. Distinguish and REPORT DIFFERENTLY, using curl's EXIT CODE:
#       - exit 6  → "DNS failure: cannot resolve host"
#       - exit 7  → "Connection refused: the app is not listening"   (502 territory)
#       - exit 28 → "Timeout: the app is up but not responding"      (504 territory)
#       - exit 22 → "HTTP error: the app answered with <code>"       (use --write-out
#                    or --fail-with-body to capture the actual status and body)
#       - exit 60 → "TLS: certificate not trusted / expired"
#  5. On success, print the full timing breakdown (-w) so the deploy log has a
#     latency baseline for every release.
#  6. Exit 0 on success, non-zero on failure, so `set -e` in the deploy script works.
#
# Then TEST all five failure modes for real:
#   - point it at a bogus hostname            → 6
#   - point it at localhost:9999 (nothing there) → 7
#   - point it at a route that sleeps 30s + --max-time 5 → 28
#   - point it at an endpoint that returns 500 → 22
#   - point it at https://expired.badssl.com  → 60
#
# Finally: run it against your actual Node app, restart the app mid-check, and
# confirm the --retry logic rides out the restart and reports success.
```

---

## Mental Model Checkpoint

1. **You `curl -d '{"a":1}' https://api/x` and get a 400. The API works from Postman. What is wrong, and what single `curl -v | grep '^>'` line would prove it?**
2. **Your deploy script runs `curl -s http://localhost:3000/health` and reports success — but the app is returning 500s. Why did the script pass? What one flag fixes it, and what will curl's exit code become?**
3. **`curl https://x/old` prints nothing and exits 0. Is the endpoint broken? What flag reveals the truth, and what flag fixes it?**
4. **Draw the `-w` timing timeline.** Which interval is "DNS", which is "TLS", and which is "the server was slow"? If `time_starttransfer` minus `time_appconnect` is 4 seconds, whose problem is it?
5. **nginx returns a 502 in 40ms. Then, later, a 504 after 60 seconds. What is different about the state of your Node app in each case?** And what does a 499 in the access log tell you?
6. **curl exits 7 vs curl exits 28.** What is the difference, and what does each one tell you about the firewall rule that blocked you (topic 29)?
7. **You need to download a 3GB tarball onto a server over a connection that keeps dropping.** curl or wget, which flag, and why?

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---|---|---|
| `curl URL` | GET, body → stdout | |
| `curl -i URL` | Response headers + body | vs `-I` (HEAD only), vs `-v` (full exchange to stderr) |
| `curl -v URL` | **The debugging workhorse** | `>` sent, `<` received, `*` curl's commentary |
| `curl -L URL` | Follow redirects | **NOT the default.** `--max-redirs N` |
| `curl --json '{}' URL` | POST JSON, correctly | Sets Content-Type AND Accept. curl 7.82+ |
| `curl -H 'Content-Type: application/json' -d '{}' URL` | POST JSON, the compatible way | `-d` alone sends urlencoded → 400 |
| `curl --data-binary @f.json URL` | POST a JSON **file** | `-d @f` strips newlines — use `--data-binary` |
| `curl -F 'file=@x.png' URL` | Upload a file (multipart) | |
| `curl -sS -f URL` | **The scripting combo** | `-s` quiet, `-S` show errors, `-f` non-zero exit on 4xx/5xx |
| `curl --max-time 10 --connect-timeout 3` | **Mandatory in cron/CI** | Without these, a hung request hangs forever |
| `curl --retry 5 --retry-delay 2 --retry-all-errors` | Ride out a restart | Pair with `-f` |
| `curl -o /dev/null -s -w '%{http_code} %{time_total}\n'` | Status + latency, nothing else | The one-liner you'll type most |
| `curl -w "@curl-format.txt" -o /dev/null -s URL` | **The poor-man's APM** | DNS vs TCP vs TLS vs TTFB vs transfer |
| `curl --resolve host:443:IP URL` | Test a new server before DNS cutover | Keeps the Host header + SNI + cert check |
| `curl -H "Authorization: Bearer $TOKEN"` | Auth | **Never put the token in the URL** |
| `curl -k URL` | ⚠️ Disable cert verification | Use `--cacert` or install the CA instead |
| `curl -s URL \| jq '.data[]'` | Parse JSON | `-s` is essential when piping |
| `wget URL` | Download → file | |
| `wget -c URL` | **Resume a broken download** | The reason wget exists |
| `wget -qO- URL` | wget pretending to be curl | For containers without curl |
| `wget --spider URL` | Check existence, don't download | |
| `wget -r -np -k URL` | Mirror a site | `-np` = don't climb above the start dir |

**Exit codes:** `0` ok (⚠️ *including* a 500 without `-f`) · `6` DNS · `7` connection refused · `22` HTTP 4xx/5xx (with `-f`) · `28` timeout · `35` TLS handshake · `60` cert not trusted

**Alternative:** [httpie](https://httpie.io) (`http POST :3000/users name=ada`) is far friendlier — it defaults to JSON, colorizes output, and shows headers by default. Great for interactive exploration. But **learn curl anyway**: it is on every server, in every container, in every CI image, and every bug report and API doc on earth is written in curl.

---

## When Would I Use This at Work?

### Scenario 1: "The API is slow" — finding out *which* layer is slow

Support says the mobile app is timing out. Your APM says the Express handler runs in 12ms. Somebody is lying. You run `curl -w "@curl-format.txt" -o /dev/null -s https://api.yourco.com/orders` from three places: your laptop, the app server itself, and a box in the same VPC. From the VPC: `time_total 0.031s`. From your laptop: `time_namelookup 3.9s`, everything else normal. **It's DNS.** One of the two resolvers in `/etc/resolv.conf` on the client side is a black hole, and every other request eats a 5-second timeout before failing over. Your Express handler was never the problem, and no amount of code profiling would have found it.

### Scenario 2: Isolating nginx from your app during a 502 storm

`https://api.yourco.com` returns 502. Two commands settle it: `curl -sS -o /dev/null -w '%{http_code} %{time_total}\n' https://api.yourco.com/health` → `502 0.09s` (fast → refused, not timed out), then `curl -sS --max-time 5 http://127.0.0.1:3000/health` on the box → `curl: (7) Connection refused`. Exit 7. The Node process is dead. nginx is blameless. You've gone from "the site is down" to "restart the app service and find out why it OOM'd" in under a minute — and you can tell the team confidently that it isn't a proxy config problem, so nobody wastes time in `nginx.conf`.

### Scenario 3: A deploy health check that actually protects you

Your rollout script (topic 13) restarts the systemd service and then runs a health check. The naive version — `curl -s localhost:3000/health` — passes on a 500 and green-lights a broken release. You replace it with `curl -sS -f --retry 15 --retry-delay 2 --retry-all-errors --connect-timeout 2 --max-time 5 -o /dev/null http://127.0.0.1:3000/health`, which retries while the app boots, **fails hard on any 4xx/5xx**, bounds every attempt, and returns a non-zero exit code that `set -e` will catch — triggering your automatic rollback. One flag (`-f`) is the difference between a deploy system that protects you and one that lies to you.

---

## Connected Topics

| Direction | Topic | Why |
|---|---|---|
| **Next** | 31 — Disk Management | The other half of "the server is broken at 2am": not the network, the disk |
| **Builds on** | 12 — Pipes and Redirection | `curl -s ... \| jq`, `-o file`, `2>&1`, `-w` writing to stdout while `-v` writes to stderr — all of it is fds 0/1/2 |
| **Builds on** | 27 — DNS from the CLI | `time_namelookup` is `getaddrinfo()` reading `/etc/resolv.conf`. A slow DNS phase sends you straight to `dig` |
| **Builds on** | 28 — Ports and Sockets | curl exit 7 = nothing is LISTENING. Confirm with `ss -tulpn` |
| **Builds on** | 29 — Firewalls | curl exit 7 (instant refuse) = REJECT or nothing listening. curl exit 28 (timeout) = DROP. curl is how you *test* the firewall change you just made |
| **Builds on** | 13 — Shell Scripting | Exit codes, `set -e`, `$?` — the whole reason `-f` matters |
| **Builds on** | 20 — Cron | The `--max-time` lesson: an unbounded curl in cron will pile up until you exhaust file descriptors |
| **Used by** | 33 — Node in Production | Health checks, readiness probes, and `HEALTHCHECK` in your Dockerfile are all curl |
| **Used by** | 34 — Performance Investigation | `-w` timing is the first tool you reach for before you ever touch `strace` or a profiler |
