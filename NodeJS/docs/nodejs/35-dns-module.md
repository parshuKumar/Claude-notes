# 35 — DNS module

## What is this?

The `dns` module lets Node.js turn a human-readable hostname like `api.stripe.com` into a machine-usable IP address like `54.187.174.169`, and do the reverse — turn an IP back into a hostname. Think of it like a phone contacts app: you type a person's name ("Mom"), and the phone looks up their actual phone number before dialing. Every time your Node app calls an external API, connects to a database by hostname, or sends an email, this name-to-address translation happens first — `dns` is the module that does it, or lets you do it yourself explicitly.

## Why does it matter for backend development?

Backend servers are constantly talking to other machines by name — a payment gateway, a Postgres host, a Redis cluster, a third-party API. Under the hood, every one of those connections starts with a DNS resolution step, and when it goes wrong (a typo'd hostname, an expired domain, a slow DNS server) your request hangs or fails with a cryptic error. Knowing the `dns` module lets you: validate hostnames before attempting a connection, build health checks that catch DNS failures early, verify a signup email's domain actually has mail servers (anti-fraud), debug "connection refused" issues by checking what IP a hostname actually resolves to, and understand why `dns.lookup()` and `dns.resolve()` — which look similar — can give you completely different answers.

---

## Syntax / API

```js
// Core module — built into Node, no install needed
const dns = require('dns');

// ── dns.lookup() — asks the OPERATING SYSTEM to resolve a hostname ─────────
// Behaves exactly like `ping hostname` — uses /etc/hosts, OS cache, nsswitch.conf
dns.lookup('api.example.com', (err, address, family) => {
  // err     → Error if resolution failed (e.g. ENOTFOUND)
  // address → the resolved IP string, e.g. "104.21.7.13"
  // family  → 4 (IPv4) or 6 (IPv6)
});

// Pass { all: true } to get every IP the hostname resolves to, not just one
dns.lookup('api.example.com', { all: true }, (err, addresses) => {
  // addresses → [{ address: '104.21.7.13', family: 4 }, { address: '...', family: 6 }]
});

// ── dns.resolve() — asks a DNS SERVER directly, bypasses OS/hosts file ─────
dns.resolve('api.example.com', 'A', (err, addresses) => {
  // addresses → array of IPv4 strings straight from the DNS server, e.g. ['104.21.7.13']
});

// Shortcuts for common record types — same idea as resolve() with a fixed rrtype
dns.resolve4('api.example.com', (err, addresses) => {});   // IPv4 addresses only
dns.resolve6('api.example.com', (err, addresses) => {});   // IPv6 addresses only
dns.resolveMx('gmail.com', (err, records) => {
  // records → [{ priority: 5, exchange: 'gmail-smtp-in.l.google.com' }, ...]
});
dns.resolveTxt('example.com', (err, records) => {});        // TXT records (SPF, verification)
dns.resolveCname('www.example.com', (err, addresses) => {}); // CNAME aliases

// ── dns.reverse() — IP → hostname(s), the opposite direction ──────────────
dns.reverse('142.250.183.14', (err, hostnames) => {
  // hostnames → ['maa03s28-in-f14.1e100.net'] — PTR record for that IP
});

// ── Promise-based API — same functions, no callbacks (Node 10.6+) ─────────
const dnsPromises = require('dns').promises;

async function resolveHost(hostname) {
  const address = await dnsPromises.lookup(hostname); // await instead of callback
  return address; // { address: '...', family: 4 }
}

// ── Changing which DNS servers Node's resolve* functions talk to ──────────
dns.setServers(['8.8.8.8', '1.1.1.1']); // use Google + Cloudflare instead of system default
```

---

## How it works — line by line

Node.js actually gives you **two different resolution systems** disguised as one module, and picking the wrong one is the single biggest source of confusion:

- **`dns.lookup()`** does not talk to a DNS server itself. It calls the operating system's own resolver — the exact same C function (`getaddrinfo`) that tools like `ping` and `curl` use. That means it respects your `/etc/hosts` file, your OS-level DNS cache, and any custom name resolution rules configured on the machine (like `nsswitch.conf` on Linux). Because `getaddrinfo` is a blocking system call, Node runs it on the **libuv thread pool** (the same pool used for file system operations) rather than the main event loop.

- **`dns.resolve()`** (and `resolve4`, `resolveMx`, etc.) skip the operating system entirely. They use a bundled library called **c-ares** to send actual DNS query packets over the network directly to a DNS server — by default whatever is in your OS's configured resolvers, or whatever you set with `dns.setServers()`. This means `/etc/hosts` is ignored, and you get the DNS server's raw answer, not an OS-cached one. It also does not use the thread pool — c-ares does its own non-blocking network I/O.

- **`dns.reverse()`** does a PTR lookup — given an IP, it asks "what hostname claims to own this IP?" This is commonly used for logging (showing where a request came from) or lightweight spam/bot filtering, though PTR records are optional and often missing, so `err` is common and expected.

- The **promise-based API** (`require('dns').promises`) exposes the exact same functions, just returning Promises instead of taking callbacks — this is the version you'll use in modern `async/await` code, matching the pattern of `fs.promises` covered in Topic 16.

In short: use `dns.lookup()` when you just need "what IP do I connect to" (mirrors what your OS would do). Use `dns.resolve()` and its variants when you need the actual DNS record data (MX, TXT, multiple A records) or want to bypass local overrides.

---

## Example 1 — basic

```js
// File: src/scripts/check-dns.js
// Basic demonstration of the core dns functions, no external dependencies.

const dns = require('dns').promises; // use the promise-based API — cleaner in modern code

async function inspectHostname(hostname) {
  try {
    // dns.lookup() — "what IP would my OS connect to for this name?"
    const lookupResult = await dns.lookup(hostname);
    console.log(`[lookup] ${hostname} →`, lookupResult); // { address, family }

    // dns.resolve4() — "what IPv4 addresses does the DNS server list?"
    const ipv4Addresses = await dns.resolve4(hostname);
    console.log(`[resolve4] ${hostname} →`, ipv4Addresses); // array of IPs, can be multiple

    // dns.resolveMx() — "what mail servers handle email for this domain?"
    const mxRecords = await dns.resolveMx(hostname);
    console.log(`[resolveMx] ${hostname} →`, mxRecords); // [{ priority, exchange }]
  } catch (err) {
    // ENOTFOUND / ENODATA are common — domain doesn't exist, or has no record of that type
    console.error(`[dns error] ${hostname}:`, err.code, err.message);
  }
}

// dns.reverse() — go the other way: IP → hostname
async function inspectIp(ip) {
  try {
    const hostnames = await dns.reverse(ip); // returns an array — an IP can map to many names
    console.log(`[reverse] ${ip} →`, hostnames);
  } catch (err) {
    console.error(`[reverse error] ${ip}:`, err.code); // PTR records are often missing — normal
  }
}

inspectHostname('gmail.com'); // real domain — has both A and MX records
inspectIp('8.8.8.8');          // Google's public DNS server — has a known PTR record
```

---

## Example 2 — real world backend use case

```js
// File: src/services/emailDomainValidator.js
// A signup-flow guard: reject registrations where the email domain has NO mail servers.
// This catches typos ("gmial.com") and disposable/fake domains before they hit your database.

const dns = require('dns').promises;

/**
 * Checks whether a domain can actually receive email, by verifying it has
 * MX records (or, as a fallback, at least an A record some mail setups use).
 */
async function domainAcceptsEmail(domain) {
  try {
    const mxRecords = await dns.resolveMx(domain); // ask DNS: "who handles mail for this domain?"
    return mxRecords.length > 0;                    // at least one mail exchange server exists
  } catch (mxError) {
    // ENODATA / ENOTFOUND → no MX records; some domains route mail via a plain A record instead
    if (mxError.code === 'ENODATA' || mxError.code === 'ENOTFOUND') {
      try {
        await dns.resolve4(domain); // fallback: does the domain resolve at all?
        return true;                 // domain exists — assume it can be configured to accept mail
      } catch (aError) {
        return false;                // domain doesn't exist at all — definitely invalid
      }
    }
    throw mxError; // an unexpected error (timeout, network issue) — let the caller decide
  }
}

// Express-style signup handler using the validator
async function handleSignup(requestBody, response) {
  const userEmail = requestBody.email;                 // e.g. "user@example.com"
  const emailDomain = userEmail.split('@')[1];          // extract "example.com"

  if (!emailDomain) {
    return response.status(400).json({ error: 'Invalid email format' });
  }

  const isValidDomain = await domainAcceptsEmail(emailDomain);

  if (!isValidDomain) {
    // Domain has no mail servers — likely a typo or disposable/fake address
    return response.status(400).json({ error: `Email domain "${emailDomain}" cannot receive mail` });
  }

  // Domain checks out — proceed to create the user record
  // await userRepository.create({ email: userEmail, ... });
  response.status(201).json({ message: 'Signup accepted', email: userEmail });
}

module.exports = { domainAcceptsEmail, handleSignup };

// Usage:
// domainAcceptsEmail('gmail.com').then(console.log);       // → true
// domainAcceptsEmail('thisdoesnotexist12345.com').then(console.log); // → false
```

---

## Common mistakes

### Mistake 1 — Using dns.lookup() when you actually need DNS record data

```js
// ❌ WRONG — dns.lookup() only ever returns an IP address + family.
// It cannot tell you about MX, TXT, CNAME, or multiple A records — that's not what it queries.
const dns = require('dns');
dns.lookup('gmail.com', (err, address) => {
  console.log('Mail servers:', address); // this is just ONE IP, not mail server info at all
});

// ✅ CORRECT — use dns.resolveMx() (or the relevant resolve* function) for actual record types
dns.resolveMx('gmail.com', (err, mxRecords) => {
  console.log('Mail servers:', mxRecords); // → [{ priority: 5, exchange: 'gmail-smtp-in...' }]
});
```

### Mistake 2 — Assuming DNS resolution always succeeds and skipping error handling

```js
// ❌ WRONG — treats DNS as infallible; crashes the process on a bad/mistyped hostname
const dns = require('dns').promises;

async function connectToUpstream(hostname) {
  const { address } = await dns.lookup(hostname); // throws if hostname doesn't resolve
  return address; // unhandled rejection if hostname was wrong — process may crash
}

// ✅ CORRECT — DNS failures (ENOTFOUND, ENODATA, timeouts) are routine and must be caught
async function connectToUpstream(hostname) {
  try {
    const { address } = await dns.lookup(hostname);
    return address;
  } catch (err) {
    // err.code tells you exactly what went wrong — log it, don't let it crash the process
    console.error(`DNS resolution failed for ${hostname}:`, err.code);
    throw new Error(`Cannot reach upstream host "${hostname}"`);
  }
}
```

### Mistake 3 — Calling dns.lookup() on every single request without any caching

```js
// ❌ WRONG — resolving the same hostname on every incoming request adds latency to
// every request and can exhaust the libuv thread pool under load (lookup uses it, not the network)
app.get('/proxy', async (req, res) => {
  const { address } = await dns.lookup('upstream-api.internal'); // repeated, wasted work
  const upstreamResponse = await fetch(`http://${address}/data`);
  res.json(await upstreamResponse.json());
});

// ✅ CORRECT — cache the resolved address and refresh it periodically (e.g. every 60s),
// since DNS records rarely change second-to-second in practice
let cachedAddress = null;
let cacheExpiresAt = 0;

async function getUpstreamAddress() {
  if (cachedAddress && Date.now() < cacheExpiresAt) {
    return cachedAddress; // reuse — no new lookup needed
  }
  const { address } = await dns.lookup('upstream-api.internal');
  cachedAddress = address;
  cacheExpiresAt = Date.now() + 60_000; // refresh at most once a minute
  return cachedAddress;
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that takes a hostname (hardcode `'nodejs.org'` for now) and:
1. Uses `dns.promises.lookup()` to print its IP address and IP family (4 or 6)
2. Uses `dns.promises.resolve4()` to print all of its IPv4 addresses
3. Wraps both calls in a `try/catch` and prints a clear error message if resolution fails
4. Also runs the whole thing against a hostname that does not exist (e.g. `'this-domain-does-not-exist-xyz123.com'`) and confirms the error is caught, not thrown uncaught

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `getMailServers(domain)` that:
1. Calls `dns.promises.resolveMx(domain)` and returns the array of MX records
2. Sorts the returned records by `priority` ascending (lowest priority number = most preferred mail server)
3. Returns just the array of `exchange` hostnames, in that sorted order
4. Handles the case where the domain has no MX records at all (return an empty array instead of throwing)

Test it against `'gmail.com'`, `'outlook.com'`, and a domain with no MX records, and log the results for each.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `DnsCache` class that wraps DNS lookups with caching and TTL, meant to sit in front of any code that repeatedly connects to the same set of hostnames (like a connection pool or an API client):

1. Constructor takes an optional `ttlMs` (default `60000`)
2. Has an async method `resolve(hostname)` that:
   - Returns the cached IP address if it exists and hasn't expired
   - Otherwise calls `dns.promises.lookup(hostname)`, stores the result with a timestamp, and returns it
3. Has a method `invalidate(hostname)` that manually clears one cached entry (useful if a connection attempt fails and you suspect a stale/changed IP)
4. Has a method `stats()` that returns `{ size, hostnames: [...] }` describing what's currently cached
5. If a lookup fails, it should NOT cache the failure — the next call should retry immediately

Test it:
```js
const cache = new DnsCache({ ttlMs: 5000 });
console.log(await cache.resolve('nodejs.org'));  // performs a real lookup
console.log(await cache.resolve('nodejs.org'));  // returns from cache, instantly
cache.invalidate('nodejs.org');
console.log(await cache.resolve('nodejs.org'));  // performs a real lookup again
console.log(cache.stats());
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE FUNCTIONS
  dns.lookup(hostname, cb)        → OS resolver (getaddrinfo). Uses /etc/hosts, OS cache.
                                    Returns { address, family }. Uses libuv thread pool.
  dns.resolve(hostname, rrtype)  → Direct DNS query via c-ares. Ignores /etc/hosts.
  dns.resolve4 / resolve6         → Shortcuts for A / AAAA records
  dns.resolveMx(hostname)         → Mail exchange records: [{ priority, exchange }]
  dns.resolveTxt(hostname)        → TXT records (SPF, domain verification, etc.)
  dns.resolveCname(hostname)      → CNAME alias chain
  dns.reverse(ip)                 → IP → hostname(s) (PTR record); often empty, that's normal
  dns.setServers([ips])           → change which DNS servers resolve*() functions query
  dns.getServers()                → list currently configured DNS servers

LOOKUP vs RESOLVE — THE KEY DIFFERENCE
  dns.lookup()   → asks the OS, same as `ping`/`curl` would. Honors /etc/hosts. Thread pool.
  dns.resolve()  → asks a DNS server directly over the network. Ignores /etc/hosts. c-ares.

  Use lookup()  → when you just need "what IP do I connect to"
  Use resolve() → when you need actual record data, or must bypass local host overrides

PROMISE API
  const dns = require('dns').promises;
  await dns.lookup(hostname)
  await dns.resolve4(hostname)
  → same functions, no callbacks, works naturally with async/await

COMMON ERROR CODES
  ENOTFOUND → hostname does not exist at all
  ENODATA   → hostname exists but has no record of the requested type
  ETIMEOUT  → DNS server did not respond in time
  ECONNREFUSED → couldn't reach the configured DNS server

GOTCHAS
  - dns.lookup() results can differ from dns.resolve() if /etc/hosts overrides a name
  - Reverse DNS (PTR) records are optional — expect errors/empty results often
  - DNS lookups add real latency — cache aggressively for hot-path/repeated hostnames
  - dns.lookup() consumes the libuv thread pool — heavy concurrent lookups can starve
    other thread-pool work (like fs operations) if the pool size isn't increased
```

---

## Connected topics

- **34 — TCP and UDP with net/dgram** — before `net.createServer()`/sockets can connect anywhere, the hostname must be resolved to an IP first, exactly what this module does
- **20 — http module** — every `http.request()` call to a hostname triggers a DNS lookup internally; understanding `dns` explains connection-time latency and `ENOTFOUND` errors you'll see from HTTP clients
- **22 — url module** — `new URL()` parses out the `hostname` piece of a URL, which is precisely the string you hand to `dns.lookup()` or `dns.resolve()`
