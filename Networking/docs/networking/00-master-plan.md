# 00 — Master Plan: Networking Deep Mastery Curriculum

> **Contract file.** Every topic, every filename, every milestone lives here.
> Before writing any topic doc, check this file. After finishing a topic, tick its checkbox.

---

## Phases Overview

| Phase | Theme | Topics |
|-------|-------|--------|
| 1 | How the Internet Actually Works | 01 – 05 |
| 2 | Core Protocols | 06 – 13 |
| 3 | HTTP for Backend Developers | 14 – 20 |
| 4 | Network Debugging and Tooling | 21 – 27 |
| 5 | Production Networking Concepts | 28 – 34 |
| 6 | Networking in System Design | 35 – 39 |

---

## Complete Curriculum & File List

### PHASE 1 — How the Internet Actually Works

| # | File | One-Line Description |
|---|------|----------------------|
| 01 | `01-how-the-internet-works.md` | The big picture: clients, servers, ISPs, undersea cables, data centers, and what physically happens when you open a browser |
| 02 | `02-osi-model.md` | All 7 OSI layers, what each layer does, a real protocol at each layer, and why layering exists at all |
| 03 | `03-tcp-ip-model.md` | The 4-layer TCP/IP model, how it maps to OSI, and which layers backend developers touch every day |
| 04 | `04-ip-addresses.md` | IPv4 vs IPv6, public vs private addresses, subnets, CIDR notation, NAT, and why 192.168.x.x is always private |
| 05 | `05-ports.md` | What a port number is, every well-known port a backend dev must memorize, ephemeral ports, and port conflicts |

---

### PHASE 2 — Core Protocols

| # | File | One-Line Description |
|---|------|----------------------|
| 06 | `06-dns-in-depth.md` | The full DNS resolution chain: recursive resolver → root → TLD → authoritative nameserver, caching, TTL, and every record type |
| 07 | `07-udp.md` | The UDP protocol, what "connectionless" really means, why UDP exists, and when to choose it over TCP |
| 08 | `08-tcp-in-depth.md` | The 3-way handshake, sequence/ack numbers, flow control, congestion control, and the 4-way connection teardown |
| 09 | `09-tls-ssl-in-depth.md` | Why HTTP is naked, asymmetric vs symmetric encryption, X.509 certificates, the full TLS 1.3 handshake, CAs, and SNI |
| 10 | `10-http1.1-in-depth.md` | HTTP/1.1 request/response format, all methods, status codes, headers, keep-alive, and pipelining problems |
| 11 | `11-http2.md` | Multiplexing, HPACK header compression, server push, binary framing, and exactly how HTTP/2 fixes HTTP/1.1 |
| 12 | `12-http3-and-quic.md` | QUIC built on UDP, 0-RTT connection setup, connection migration, head-of-line blocking fix, and where HTTP/3 stands today |
| 13 | `13-websockets.md` | The HTTP Upgrade handshake, full-duplex frames, the wire format, when to use WebSockets vs HTTP polling vs SSE |

---

### PHASE 3 — HTTP for Backend Developers

| # | File | One-Line Description |
|---|------|----------------------|
| 14 | `14-http-headers-in-depth.md` | Every important request/response header, content negotiation, caching headers (Cache-Control, ETag, Last-Modified), and security headers |
| 15 | `15-rest-over-http.md` | HTTP method semantics, idempotency rules, correct status code usage, and clean URL design |
| 16 | `16-authentication-over-http.md` | Basic auth, Bearer tokens, cookies (Set-Cookie, HttpOnly, Secure, SameSite), and session vs JWT over the wire |
| 17 | `17-cors-in-depth.md` | How the browser actually enforces CORS, the preflight OPTIONS request, every CORS header explained, and the most common mistakes |
| 18 | `18-rate-limiting-at-http-level.md` | X-RateLimit-* headers, Retry-After, the 429 status code, and how well-behaved clients should respond |
| 19 | `19-cookies-and-sessions.md` | What Set-Cookie actually puts on the wire, how browsers store and resend cookies, and every security implication |
| 20 | `20-caching-in-depth.md` | Browser cache, CDN cache, server cache, every Cache-Control directive, ETag validation flow, and stale-while-revalidate |

---

### PHASE 4 — Network Debugging and Tooling

| # | File | One-Line Description |
|---|------|----------------------|
| 21 | `21-curl-mastery.md` | Every curl flag a backend dev needs: GET/POST/PUT/DELETE, custom headers, auth, redirects, verbose mode, and the timing breakdown |
| 22 | `22-dig-and-dns-debugging.md` | Reading dig output line by line, tracing a full DNS resolution, and diagnosing the most common DNS problems |
| 23 | `23-tcpdump-and-wireshark.md` | Capturing live traffic, writing filters, reading packet output, and finding exactly where packets die |
| 24 | `24-netstat-and-ss.md` | Reading connection tables, every TCP state (LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT), and what each means for your app |
| 25 | `25-ping-traceroute-mtr.md` | Measuring latency hop by hop, reading traceroute output correctly, and using mtr for continuous path analysis |
| 26 | `26-nmap-basics.md` | Port scanning, service/version detection, and the legitimate backend-developer use cases for nmap |
| 27 | `27-debugging-a-slow-api.md` | Full methodology: break down DNS time, TCP connect time, TLS time, TTFB, transfer time — and know exactly where to look |

---

### PHASE 5 — Production Networking Concepts

| # | File | One-Line Description |
|---|------|----------------------|
| 28 | `28-load-balancers.md` | L4 vs L7 load balancing, routing algorithms, health checks, sticky sessions, and what changes at the TCP level |
| 29 | `29-reverse-proxies.md` | Nginx as a reverse proxy, which headers it injects (X-Forwarded-For, X-Real-IP), and why your app always sees 127.0.0.1 |
| 30 | `30-cdns.md` | How CDN edge nodes work, cache hit vs miss, cache purging, origin pull, and when a CDN actually helps |
| 31 | `31-vpns-and-private-networks.md` | What a VPN tunnel does at the packet level, why databases must live on private networks, and VPC basics |
| 32 | `32-firewalls-and-security-groups.md` | Stateful vs stateless firewalls, inbound vs outbound rules, what 0.0.0.0/0 means, and AWS security group patterns |
| 33 | `33-network-latency.md` | The physics of latency, speed-of-light limits, what you can vs cannot optimize, and the numbers every engineer must know |
| 34 | `34-api-gateways.md` | What an API gateway does at the network level: routing, rate limiting, auth offloading, and request transformation |

---

### PHASE 6 — Networking in System Design

| # | File | One-Line Description |
|---|------|----------------------|
| 35 | `35-network-in-system-design.md` | Latency numbers every engineer must memorize, bandwidth math, and how to reason about network trade-offs in design interviews |
| 36 | `36-service-to-service-communication.md` | Internal DNS, service mesh concepts (Envoy/Istio), and mutual TLS (mTLS) for zero-trust internal networks |
| 37 | `37-grpc-and-protocol-buffers.md` | How gRPC works over HTTP/2, Protocol Buffer wire format, and why gRPC is faster than REST for internal services |
| 38 | `38-message-queues-at-network-level.md` | How Kafka and RabbitMQ manage TCP connections, consumer group rebalancing over the wire, and persistence semantics |
| 39 | `39-database-connections-over-network.md` | Connection pooling mechanics, why 10,000 raw connections kill Postgres, how PgBouncer works at the TCP level |

---

## Progress Tracker

### PHASE 1 — How the Internet Actually Works
- [x] `01` — How the Internet Works
- [x] `02` — The OSI Model
- [x] `03` — The TCP/IP Model
- [x] `04` — IP Addresses in Depth
- [x] `05` — Ports

### PHASE 2 — Core Protocols
- [x] `06` — DNS in Depth
- [x] `07` — UDP
- [x] `08` — TCP in Depth
- [x] `09` — TLS/SSL in Depth
- [x] `10` — HTTP/1.1 in Depth
- [x] `11` — HTTP/2
- [x] `12` — HTTP/3 and QUIC
- [x] `13` — WebSockets

### PHASE 3 — HTTP for Backend Developers
- [x] `14` — HTTP Headers in Depth
- [x] `15` — REST over HTTP
- [x] `16` — Authentication over HTTP
- [x] `17` — CORS in Depth
- [x] `18` — Rate Limiting at the HTTP Level
- [x] `19` — Cookies and Sessions over the Wire
- [x] `20` — Caching in Depth

### PHASE 4 — Network Debugging and Tooling
- [x] `21` — curl Mastery
- [x] `22` — dig and DNS Debugging
- [x] `23` — tcpdump and Wireshark
- [x] `24` — netstat and ss
- [x] `25` — ping, traceroute, mtr
- [x] `26` — nmap Basics
- [x] `27` — Debugging a Slow API

### PHASE 5 — Production Networking Concepts
- [x] `28` — Load Balancers
- [x] `29` — Reverse Proxies
- [x] `30` — CDNs
- [x] `31` — VPNs and Private Networks
- [x] `32` — Firewalls and Security Groups
- [x] `33` — Network Latency
- [x] `34` — API Gateways

### PHASE 6 — Networking in System Design
- [x] `35` — Network in System Design
- [x] `36` — Service-to-Service Communication
- [x] `37` — gRPC and Protocol Buffers
- [x] `38` — Message Queues at the Network Level
- [x] `39` — Database Connections over the Network

---

## Commands (use these during the curriculum)

| Command | What happens |
|---------|-------------|
| `START` | Begin Topic 01 |
| `NEXT` | Move to the next topic (exercises must be done first) |
| `HINT` | One nudge on the current exercise |
| `PROGRESS` | Show this checklist with completed items ticked |
| `REDO [topic]` | Regenerate a specific topic doc |
| `SKIP [topic]` | Mark a topic done without exercises (use sparingly) |

---

## Doc Template (applied to every topic)

```
# [Number] — [Topic Name]

## ELI5 — The Simple Analogy
## Where this lives in the network stack
## What is this?
## Why does it matter for a backend developer?
## The packet/protocol anatomy
## How it works — step by step
## Exact syntax breakdown
## Example 1 — basic
## Example 2 — production scenario
## Common mistakes
## Hands-on proof
## Practice exercises
## Mental model checkpoint
## Quick reference card
## When would I use this at work?
## Connected topics
```

---

*Last updated: All 39 topics complete. Every phase (1–6) finished — the full Networking Deep Mastery Curriculum is written.*
