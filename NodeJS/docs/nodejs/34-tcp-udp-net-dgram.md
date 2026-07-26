# 34 — TCP and UDP with net/dgram

## What is this?

`net` and `dgram` are Node's low-level networking modules — they let you talk directly to the transport layer of the internet, below HTTP. `net` builds **TCP** connections: reliable, ordered, connection-based pipes between two machines (like a phone call — you dial, both sides stay connected, every word arrives in order). `dgram` sends **UDP** datagrams: fire-and-forget packets with no connection and no guarantee of delivery or order (like dropping postcards in a mailbox — cheap and fast, but some might get lost or arrive out of sequence). Every HTTP request you've ever made actually rides on top of a TCP socket built with something very close to the `net` module.

## Why does it matter for backend development?

Express, Fastify, and even the built-in `http` module are all just **wrappers around `net.createServer()`** — HTTP is a text protocol layered on top of a raw TCP socket. Understanding `net` demystifies what "a server" actually is: a program listening on a port, accepting socket connections, and reading/writing bytes. Backend developers reach for `net` directly when building custom protocols (database drivers, message brokers, chat servers, IoT gateways) that don't fit the request/response shape of HTTP. `dgram` (UDP) matters for a different reason: it's what powers DNS lookups, video/voice streaming, game servers, and metrics systems (like StatsD) where speed matters more than guaranteed delivery — losing one frame of a video call is fine, but waiting for it to be retransmitted is not.

---

## Syntax / API

```js
// ── TCP server with net ──────────────────────────────────────────────────────
const net = require('net'); // core module, no install needed

// createServer() takes a callback that fires once PER CONNECTED CLIENT
const tcpServer = net.createServer((socket) => {
  // 'socket' is a Duplex stream — you can read from it AND write to it
  socket.write('Welcome to the TCP server\n'); // send bytes to this client

  socket.on('data', (chunk) => {
    // fires every time this client sends bytes — chunk is a Buffer
    console.log('Received:', chunk.toString());
  });

  socket.on('end', () => {
    // fires when the CLIENT closes their end of the connection
    console.log('Client disconnected');
  });

  socket.on('error', (err) => {
    // ALWAYS handle this — an unhandled socket error crashes the process
    console.error('Socket error:', err.message);
  });
});

tcpServer.listen(5000, () => {
  // server is now accepting connections on port 5000
  console.log('TCP server listening on port 5000');
});

// ── TCP client with net ──────────────────────────────────────────────────────
const client = net.createConnection({ port: 5000, host: '127.0.0.1' }, () => {
  // fires once the connection to the server is established
  client.write('Hello from the client\n'); // send bytes to the server
});

// ── UDP socket with dgram ────────────────────────────────────────────────────
const dgram = require('dgram'); // core module for UDP datagrams

const udpSocket = dgram.createSocket('udp4'); // 'udp4' = IPv4 UDP socket

udpSocket.on('message', (msg, rinfo) => {
  // fires when a datagram arrives — msg is a Buffer, rinfo has sender's address/port
  console.log(`Got "${msg}" from ${rinfo.address}:${rinfo.port}`);
});

udpSocket.bind(5001, () => {
  // start listening for incoming datagrams on port 5001
  console.log('UDP socket bound to port 5001');
});

// Sending a UDP datagram (no connection needed — just fire the packet)
const outgoing = dgram.createSocket('udp4');
const payload  = Buffer.from('ping'); // UDP sends raw Buffers, not strings
outgoing.send(payload, 5001, '127.0.0.1'); // (data, port, host)
```

---

## How it works — line by line

**TCP (`net`) side:**
`net.createServer()` doesn't accept connections itself — it hands you a fresh `socket` object every single time a new client connects. That socket is a two-way pipe: you `.write()` bytes into it to send data out, and you listen for `'data'` events to receive bytes coming in. The connection stays open until either side calls `.end()` or the network drops — this is what "connection-based" means. Because TCP guarantees ordered, reliable delivery, the OS handles retransmitting lost packets and reordering them behind the scenes; you never see that complexity, you just see bytes arriving in the right order eventually.

**UDP (`dgram`) side:**
There is no `createServer()`/`createConnection()` pair for UDP because there is no connection to create. `dgram.createSocket()` just gives you a socket that can send and receive independent packets called **datagrams**. `.bind(port)` tells the OS "give me any datagram sent to this port" and `.send(data, port, host)` fires a packet at a destination with zero handshake — the sender doesn't even know if anyone is listening. Each incoming datagram triggers one `'message'` event with the raw bytes and a `rinfo` object telling you exactly who sent it (useful because, unlike TCP, there's no persistent socket per client to track that).

**The key mental model:** TCP is a stateful conversation (a socket object living as long as the connection lives); UDP is stateless packet-throwing (one socket handles messages from anyone, with no per-client object).

---

## Example 1 — basic

```js
// File: src/net/tcp-echo-server.js
// A TCP server that echoes back whatever the client sends, in uppercase.

const net = require('net');

const server = net.createServer((socket) => {
  console.log('New client connected:', socket.remoteAddress, socket.remotePort);

  // Every time this specific client sends data, echo it back transformed
  socket.on('data', (chunk) => {
    const received = chunk.toString().trim();      // Buffer → string, trim newline
    const reply    = received.toUpperCase() + '\n'; // transform the message
    socket.write(reply);                            // send it back to the same client
  });

  // Clean up when the client disconnects
  socket.on('end', () => {
    console.log('Client', socket.remoteAddress, 'disconnected');
  });

  // Never skip this — prevents ECONNRESET from crashing the process
  socket.on('error', (err) => {
    console.error('Socket error:', err.message);
  });
});

// Start listening for incoming TCP connections on port 6000
server.listen(6000, () => {
  console.log('Echo server ready on port 6000');
});

// Test it from a terminal with: nc 127.0.0.1 6000
// Type "hello" and press enter → server replies "HELLO"
```

---

## Example 2 — real world backend use case

```js
// File: src/services/metrics-collector.js
// A UDP metrics collector — the exact pattern behind tools like StatsD.
// Application servers fire off metric packets without waiting for a reply,
// so recording a metric NEVER slows down the request that triggered it.

const dgram = require('dgram');

// ── The collector (runs as its own lightweight process) ─────────────────────
function startMetricsCollector(port) {
  const socket = dgram.createSocket('udp4');
  const counters = new Map(); // in-memory tally: metricName -> count

  socket.on('message', (msg, rinfo) => {
    // Expected format: "metricName:count" e.g. "api.request.count:1"
    const [metricName, rawCount] = msg.toString().split(':');
    const count = Number(rawCount) || 1;

    const current = counters.get(metricName) || 0;
    counters.set(metricName, current + count); // accumulate the tally

    // rinfo tells us which app server sent this — useful for debugging
    console.log(`+${count} ${metricName} (from ${rinfo.address})`);
  });

  socket.on('error', (err) => {
    // A malformed packet should never crash the collector
    console.error('Metrics socket error:', err.message);
    socket.close();
  });

  socket.bind(port, () => {
    console.log(`Metrics collector listening on UDP ${port}`);
  });

  // Expose current counters for a health/debug endpoint
  return { getCounters: () => Object.fromEntries(counters) };
}

// ── The client (called from inside your Express request handlers) ──────────
function recordMetric(metricName, count = 1) {
  const client  = dgram.createSocket('udp4');
  const payload = Buffer.from(`${metricName}:${count}`);

  // Fire and forget — we do NOT await a response, UDP has none.
  // If this packet is lost, the request that triggered it is unaffected.
  client.send(payload, 8125, '127.0.0.1', (err) => {
    if (err) console.error('Failed to send metric:', err.message); // log only
    client.close(); // free the ephemeral socket immediately
  });
}

module.exports = { startMetricsCollector, recordMetric };

// Usage inside an Express route handler:
// app.get('/api/orders/:orderId', (req, res) => {
//   recordMetric('api.orders.get.count');  // does not block the response
//   res.json({ orderId: req.params.orderId });
// });
```

---

## Common mistakes

### Mistake 1 — Forgetting to handle the `'error'` event on a socket

```js
// ❌ WRONG — an unhandled 'error' event on any EventEmitter throws and
// crashes the entire Node process, even inside a try/catch elsewhere
const net = require('net');
const server = net.createServer((socket) => {
  socket.on('data', (chunk) => socket.write(chunk));
  // no 'error' listener — a client that resets the connection (ECONNRESET)
  // will crash the whole server, taking down every other connection with it
});
server.listen(6000);

// ✅ CORRECT — always attach an 'error' listener to every socket
const serverFixed = net.createServer((socket) => {
  socket.on('data', (chunk) => socket.write(chunk));
  socket.on('error', (err) => {
    console.error(`Socket error [${socket.remoteAddress}]:`, err.message);
    // the process stays alive — only this one connection is affected
  });
});
serverFixed.listen(6000);
```

### Mistake 2 — Treating TCP `'data'` chunks as complete messages

```js
// ❌ WRONG — TCP is a BYTE STREAM, not a message stream.
// One socket.write() on the client can arrive as multiple 'data' events,
// or multiple writes can arrive merged into a single 'data' event.
socket.on('data', (chunk) => {
  const message = JSON.parse(chunk.toString()); // breaks if the JSON
  handleMessage(message);                        // was split across chunks
});

// ✅ CORRECT — buffer incoming bytes and split on a known delimiter
// (real protocols use a delimiter like '\n' or a length-prefix header)
let buffer = '';
socket.on('data', (chunk) => {
  buffer += chunk.toString();
  let boundary;
  while ((boundary = buffer.indexOf('\n')) !== -1) {
    const line = buffer.slice(0, boundary);   // one complete message
    buffer     = buffer.slice(boundary + 1);  // keep the leftover for next time
    handleMessage(JSON.parse(line));
  }
});
```

### Mistake 3 — Reaching for UDP (dgram) when you need guaranteed delivery

```js
// ❌ WRONG — using UDP to send something that MUST arrive, like a payment
// confirmation or an order event. UDP drops packets silently with no retry.
const dgram = require('dgram');
const socket = dgram.createSocket('udp4');
socket.send(Buffer.from(JSON.stringify({ orderId: 'order_9812', status: 'paid' })),
  5002, 'billing.internal'); // if this packet is lost, billing never finds out

// ✅ CORRECT — use TCP (net, or a higher-level protocol like HTTP/AMQP)
// for anything that must not be silently dropped
const net = require('net');
const client = net.createConnection({ port: 5002, host: 'billing.internal' }, () => {
  client.write(JSON.stringify({ orderId: 'order_9812', status: 'paid' }) + '\n');
  // TCP guarantees this arrives, in order, or the connection reports an error
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a TCP server using `net.createServer()` that listens on port `7000`. Every time a client connects:
1. Send the client a welcome message containing the current timestamp
2. Log to the console the client's `remoteAddress` and `remotePort`
3. When the client sends any data, log it to the console prefixed with `"Client said: "`
4. Handle the `'end'` and `'error'` events so the server never crashes

Test it by connecting with `nc 127.0.0.1 7000` from a terminal.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a UDP-based "presence ping" system with two functions in one file:
1. `startPresenceListener(port)` — creates a UDP socket, binds to `port`, and keeps a `Map` of `serviceName -> lastSeenTimestamp`. Every time a datagram arrives in the format `"serviceName:alive"`, update that service's timestamp in the map.
2. `sendPresencePing(serviceName, port)` — creates a short-lived UDP socket, sends a datagram `"serviceName:alive"` to `127.0.0.1:port`, and closes the socket after sending.
3. Add a third function `getDeadServices(map, thresholdMs)` that returns an array of service names whose `lastSeenTimestamp` is older than `thresholdMs` milliseconds ago (i.e. they've stopped pinging).

Test by starting the listener, calling `sendPresencePing('auth-service', port)` and `sendPresencePing('email-service', port)`, waiting a few seconds, and calling `getDeadServices` with a low threshold to see both flagged as dead.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small line-based TCP chat room server on port `9000`:
1. Keep an array of all currently connected client sockets.
2. When a new client connects, add its socket to the array and broadcast `"A new user has joined (N total)"` to every OTHER connected client, where N is the current count.
3. Correctly buffer incoming TCP data and split it on `'\n'` (do not assume one `'data'` event equals one message — reuse the buffering pattern from Common Mistake 2).
4. When a client sends a complete line, broadcast it to every OTHER connected socket, prefixed with that client's `remotePort` (e.g. `"[54213]: hello everyone"`).
5. When a client disconnects (`'end'` event), remove its socket from the array and broadcast `"A user has left (N total)"` with the updated count.
6. Handle the `'error'` event per-socket so one bad client doesn't crash the whole chat room.

Test by opening three terminal windows with `nc 127.0.0.1 9000` and typing messages back and forth.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
TCP (net module) — connection-based, reliable, ordered
  net.createServer(socketCallback)   → callback fires ONCE PER CLIENT connection
  server.listen(port, cb)            → start accepting connections
  net.createConnection({port,host})  → client connects to a TCP server
  socket.write(data)                 → send bytes (string or Buffer)
  socket.on('data', chunk => ...)    → bytes arrived (chunk is a Buffer)
  socket.on('end', ...)              → other side closed their end
  socket.on('close', ...)            → connection fully closed
  socket.on('error', ...)            → ALWAYS attach this or process crashes
  socket.end()                       → close your end of the connection
  socket.remoteAddress / remotePort  → who is on the other end

UDP (dgram module) — connectionless, fast, no delivery guarantee
  dgram.createSocket('udp4')         → create a socket (no server/client split)
  socket.bind(port, cb)               → start receiving datagrams on a port
  socket.send(buffer, port, host, cb) → fire a datagram at a destination
  socket.on('message', (msg, rinfo)) → datagram arrived; rinfo = {address, port}
  socket.close()                      → stop and release the socket

WHEN TO USE WHICH
  TCP (net)   → must arrive, must be in order: databases, chat, file transfer,
                anything HTTP-like, RPC protocols
  UDP (dgram) → speed over reliability: DNS, video/voice, game state updates,
                metrics/telemetry, service discovery pings

KEY GOTCHAS
  TCP is a byte STREAM, not a message stream — buffer and delimit yourself
  UDP has NO ordering and NO delivery guarantee — packets can vanish or arrive out of order
  Every socket needs an 'error' listener — unhandled errors crash the process
  UDP send() does not confirm the other side received anything, only that YOUR OS sent it
  http/https modules are built ON TOP of net — this is the layer just below them
```

---

## Connected topics

- **20 — http module** — the `http` module is implemented on top of `net.createServer()`; understanding raw TCP explains what an HTTP server actually is under the hood
- **26 — stream module in depth** — a TCP `socket` is a Duplex stream, so everything about backpressure, `.pipe()`, and stream events applies directly to sockets
- **38 — WebSockets fundamentals** — WebSockets start as an HTTP request that "upgrades" the same underlying TCP socket into a persistent bidirectional channel
