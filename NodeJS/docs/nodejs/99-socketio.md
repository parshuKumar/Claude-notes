# 99 — Socket.IO in depth

## What is this?

Socket.IO is a library that sits on top of WebSockets (with automatic fallback to HTTP long-polling) to give you real-time, two-way communication between a server and many connected clients. Instead of a client repeatedly asking "anything new?" (polling), the server can push data the instant something happens — a chat message, a notification, a live score update. Think of it like a phone line that stays open between the server and every connected browser or app, instead of the client mailing a letter and waiting for a reply every time.

## Why does it matter for backend development?

Any product feature that needs to feel "live" — chat apps, live dashboards, multiplayer games, collaborative editors, order-tracking screens, live notifications — needs a persistent connection instead of request/response. A backend developer is the one who designs the events (`message:sent`, `order:updated`), decides who receives them (rooms, namespaces), tracks who is currently online (presence), and makes sure it all still works when the app is running on 5 server instances behind a load balancer (the Redis adapter). Socket.IO is the most widely used tool in Node.js for exactly this job, and understanding it well is a core real-time backend skill.

---

## Syntax / API

```js
// server.js — minimal Socket.IO server setup
const { createServer } = require('http');           // Node's built-in HTTP server
const { Server } = require('socket.io');             // Socket.IO server class

const httpServer = createServer();                   // plain HTTP server Socket.IO attaches to
const io = new Server(httpServer, {
  cors: { origin: 'https://myapp.com' },              // allow this frontend origin to connect
});

// Fires once per NEW client connection — "socket" is that one client's connection object
io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);        // unique id Socket.IO assigns per connection

  // Listen for a custom event this specific client sends
  socket.on('message:send', (requestBody) => {
    console.log('Received:', requestBody);             // data the client sent along with the event
  });

  // Send an event back to ONLY this one client
  socket.emit('welcome', { text: 'Connected!' });

  // Send an event to EVERYONE except this client
  socket.broadcast.emit('user:joined', { userId: socket.id });

  // Join a "room" — a named group this socket now belongs to
  socket.join('room:general');

  // Send an event to everyone in a room (including or excluding sender)
  io.to('room:general').emit('room:message', { text: 'Hello room!' });

  // Fires when this client disconnects (closes tab, loses network, etc.)
  socket.on('disconnect', (reason) => {
    console.log('Client disconnected:', socket.id, reason);
  });
});

httpServer.listen(3000, () => {
  console.log('Socket.IO server listening on port 3000');
});
```

---

## How it works — line by line

- `createServer()` builds a normal Node HTTP server — Socket.IO does not replace your HTTP server, it attaches to it and hijacks the WebSocket upgrade request.
- `new Server(httpServer, options)` wraps that HTTP server with Socket.IO's real-time layer. The `cors` option controls which frontend URLs are allowed to open a connection, exactly like CORS for a normal REST API.
- `io.on('connection', callback)` is the single most important line — it runs once every time a new client successfully connects, and hands you a `socket` object that represents that one specific client's live connection.
- `socket.id` is a unique string Socket.IO generates per connection — think of it as that client's temporary "phone number" for this session.
- `socket.on('eventName', callback)` listens for a custom event that this particular client emits — you invent the event names (`message:send`, `order:updated`) to match your app's needs.
- `socket.emit('eventName', data)` sends an event back to that one client only — like replying directly to the person who spoke.
- `socket.broadcast.emit(...)` sends to every OTHER connected client except the sender — useful for "someone joined" style notifications.
- `socket.join('roomName')` adds this socket to a named group. Rooms are just labels Socket.IO tracks internally — a socket can be in many rooms at once.
- `io.to('roomName').emit(...)` sends an event to every socket currently in that room, regardless of which server process handles them (with the Redis adapter, covered later).
- `socket.on('disconnect', callback)` fires automatically when the client's connection drops for any reason — closed tab, lost internet, browser crash — so you can clean up (mark them offline, leave rooms).

---

## Example 1 — basic

```js
// File: server.js
// A minimal real-time echo + broadcast server

const { createServer } = require('http');
const { Server } = require('socket.io');

const httpServer = createServer();                    // base HTTP server
const io = new Server(httpServer, {
  cors: { origin: '*' },                                // allow any origin — dev only, never in prod
});

let onlineCount = 0;                                    // simple in-memory counter of connected clients

io.on('connection', (socket) => {
  onlineCount++;                                         // one more client just connected
  console.log(`Client connected: ${socket.id} (online: ${onlineCount})`);

  // Tell everyone (including the new client) the updated count
  io.emit('presence:count', { onlineCount });             // io.emit = send to ALL connected clients

  // Listen for a chat message event from this client
  socket.on('chat:message', (requestBody) => {
    // requestBody = { text: 'hello everyone' } sent from the client
    console.log(`Message from ${socket.id}:`, requestBody.text);

    // Re-broadcast the message to every client except the sender
    socket.broadcast.emit('chat:message', {
      from: socket.id,
      text: requestBody.text,
    });
  });

  // Cleanup when this client disconnects
  socket.on('disconnect', () => {
    onlineCount--;                                        // one fewer client online
    console.log(`Client disconnected: ${socket.id} (online: ${onlineCount})`);
    io.emit('presence:count', { onlineCount });            // notify everyone of the new count
  });
});

httpServer.listen(4000, () => {
  console.log('Echo/broadcast server running on port 4000');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/realtime/chatServer.js
// A multi-room chat backend with authenticated connections, rooms, and presence tracking.
// This is the pattern used in real support-chat / team-chat products.

const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');                    // verify auth tokens on connection

const JWT_SECRET = process.env.JWT_SECRET;               // signing secret — never hardcode this

// presenceStore: roomId -> Set of { userId, socketId } currently in that room
const presenceStore = new Map();

function attachChatServer(httpServer) {
  const io = new Server(httpServer, {
    cors: { origin: process.env.CLIENT_ORIGIN },          // only trust our own frontend
  });

  // Middleware runs BEFORE 'connection' — reject bad connections early
  io.use((socket, next) => {
    const authToken = socket.handshake.auth?.token;       // client sends token during handshake
    if (!authToken) {
      return next(new Error('Missing auth token'));       // reject connection with an error
    }
    try {
      const payload = jwt.verify(authToken, JWT_SECRET);   // throws if invalid/expired
      socket.userId = payload.userId;                       // attach userId onto the socket for later use
      next();                                                // allow the connection to proceed
    } catch (err) {
      next(new Error('Invalid auth token'));                // reject — client gets 'connect_error'
    }
  });

  io.on('connection', (socket) => {
    console.log(`User ${socket.userId} connected (${socket.id})`);

    // Client asks to join a specific chat room (e.g. a support ticket thread)
    socket.on('room:join', (requestBody) => {
      const { roomId } = requestBody;                      // e.g. "ticket:4821"
      socket.join(roomId);                                  // Socket.IO room join

      // Track presence for this room
      if (!presenceStore.has(roomId)) {
        presenceStore.set(roomId, new Map());
      }
      presenceStore.get(roomId).set(socket.userId, socket.id);

      // Tell everyone ELSE in the room that this user joined
      socket.to(roomId).emit('room:userJoined', { userId: socket.userId });

      // Send the joining user the current list of who else is present
      const onlineUserIds = [...presenceStore.get(roomId).keys()];
      socket.emit('room:presence', { roomId, onlineUserIds });
    });

    // Client sends a chat message into a room
    socket.on('message:send', (requestBody) => {
      const { roomId, text } = requestBody;

      const messagePayload = {
        userId: socket.userId,
        text,
        sentAt: new Date().toISOString(),
      };

      // Broadcast to everyone in the room, including the sender (so their own UI updates too)
      io.to(roomId).emit('message:new', messagePayload);

      // In a real app: also persist messagePayload to the database here
    });

    // Client explicitly leaves a room (e.g. closes that chat tab)
    socket.on('room:leave', (requestBody) => {
      const { roomId } = requestBody;
      socket.leave(roomId);
      presenceStore.get(roomId)?.delete(socket.userId);
      socket.to(roomId).emit('room:userLeft', { userId: socket.userId });
    });

    // Handle disconnects — remove this user from EVERY room's presence list
    socket.on('disconnect', () => {
      for (const [roomId, users] of presenceStore.entries()) {
        if (users.get(socket.userId) === socket.id) {
          users.delete(socket.userId);
          io.to(roomId).emit('room:userLeft', { userId: socket.userId });
        }
      }
      console.log(`User ${socket.userId} disconnected`);
    });
  });

  return io;
}

module.exports = { attachChatServer };
```

---

## Common mistakes

### Mistake 1 — Using `io.emit()` when you meant to exclude the sender

```js
// ❌ WRONG — io.emit() sends to EVERY client, including the person who just sent the message,
// causing their own message to appear twice on their screen (once optimistically, once from server)
socket.on('chat:message', (requestBody) => {
  io.emit('chat:message', requestBody);   // sender gets their own message echoed back
});

// ✅ CORRECT — socket.broadcast.emit() sends to everyone EXCEPT the sender
socket.on('chat:message', (requestBody) => {
  socket.broadcast.emit('chat:message', requestBody);   // sender already shows it locally
});
```

### Mistake 2 — Forgetting rooms are server-instance-local without the Redis adapter

```js
// ❌ WRONG — running Socket.IO across multiple server instances (e.g. behind PM2 cluster
// or multiple containers) WITHOUT an adapter — io.to(room).emit() only reaches clients
// connected to THIS process, silently dropping events for users on other instances
const io = new Server(httpServer);
io.on('connection', (socket) => {
  socket.join('room:general');
  io.to('room:general').emit('news', { text: 'Update!' });   // misses users on other instances
});

// ✅ CORRECT — attach the Redis adapter so rooms and broadcasts work across ALL instances
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);

const io = new Server(httpServer);
io.adapter(createAdapter(pubClient, subClient));   // now io.to(room).emit() reaches every instance
```

### Mistake 3 — Trusting client-sent data for authentication or identity

```js
// ❌ WRONG — trusting a userId the client just hands you on the socket — anyone can fake this
socket.on('room:join', (requestBody) => {
  const { userId, roomId } = requestBody;      // client claims to be any userId it wants
  socket.join(roomId);
  socket.userId = userId;                       // never verified — a security hole
});

// ✅ CORRECT — verify identity ONCE during the handshake with a signed token,
// then trust only the value your server derived from it
io.use((socket, next) => {
  const authToken = socket.handshake.auth?.token;      // sent once at connection time
  try {
    const payload = jwt.verify(authToken, process.env.JWT_SECRET);
    socket.userId = payload.userId;                     // server-derived, trustworthy
    next();
  } catch {
    next(new Error('Invalid auth token'));               // reject the connection outright
  }
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a minimal Socket.IO server that:
1. Listens on port `5000`.
2. Logs a message every time a client connects, showing `socket.id`.
3. Listens for a `ping` event from the client and replies to that SAME client only with a `pong` event containing the current timestamp.
4. Logs a message when a client disconnects, showing `socket.id`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a "live document" room system where:
1. Clients emit `doc:join` with `{ docId }` to join a room named `doc:{docId}`.
2. When a client joins, everyone else already in that room receives a `doc:userJoined` event with the new user's `socket.id`.
3. Clients emit `doc:edit` with `{ docId, change }` — the server broadcasts `doc:edit` to everyone else in that room (not back to the sender).
4. Maintain an in-memory `Map` of `docId -> Set of socket.id` representing who is currently editing each document.
5. Add a `doc:activeEditors` event, emitted to a client right after they join, containing the current list of active editor socket ids for that document.
6. On disconnect, remove the socket from every document's editor set and notify the remaining editors in each affected room with `doc:userLeft`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a production-style presence and notification system:
1. Use `io.use()` middleware to verify a JWT sent as `socket.handshake.auth.token`, rejecting the connection if invalid, and attach the decoded `userId` onto the socket.
2. On connection, add the user to a `namespace` at `/notifications` (create it with `io.of('/notifications')`), separate from your main default namespace used for chat.
3. Maintain a presence map of `userId -> array of socketIds` (a single user may have multiple tabs/devices open at once).
4. Expose a plain function `notifyUser(userId, payload)` that, given a `userId`, emits a `notification:new` event to EVERY socket currently connected for that user (all their open tabs), and does nothing (no error) if the user is not currently online.
5. On disconnect, remove only that specific `socketId` from the user's array — if it becomes empty, remove the `userId` key entirely and emit an internal `presence:userOffline` event on the main namespace.
6. Wire up the Redis adapter (`@socket.io/redis-adapter`) so this works correctly across multiple server instances, and add a short comment explaining what would break without it.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE OBJECTS
  io                    → the whole Socket.IO server (all namespaces, all clients)
  socket                → one single client's connection (has .id, .join(), .emit())

SENDING EVENTS
  socket.emit(event, data)              → to THIS client only
  socket.broadcast.emit(event, data)    → to everyone EXCEPT this client
  io.emit(event, data)                  → to EVERY connected client
  io.to(room).emit(event, data)         → to everyone in a room (all instances, with adapter)
  socket.to(room).emit(event, data)     → to everyone in a room EXCEPT this client

ROOMS
  socket.join(roomName)     → add this socket to a room (many rooms allowed per socket)
  socket.leave(roomName)    → remove this socket from a room
  Rooms are just server-side labels — clients don't know they're "in" a room
  Every socket auto-joins a room equal to its own socket.id

NAMESPACES
  io.of('/admin')            → a separate communication channel with its own events/rooms
  Default namespace is '/'   → what you get with io.on('connection', ...)
  Namespaces ≠ rooms: namespace = separate channel, room = group within a channel

CONNECTION LIFECYCLE
  io.use((socket, next) => {...})   → middleware, runs before 'connection' (auth checks here)
  io.on('connection', (socket) => {...})
  socket.on('disconnect', (reason) => {...})   → cleanup, presence removal

SCALING (MULTIPLE SERVER INSTANCES)
  npm install @socket.io/redis-adapter redis
  Without adapter → rooms/broadcasts only reach clients on the SAME process
  With adapter    → Redis pub/sub relays events between all instances transparently
  Need 2 Redis clients: one for publish, one for subscribe (subClient = pubClient.duplicate())

PRESENCE PATTERN
  Track online users in a Map: userId -> Set/array of socketIds (multi-tab support)
  Add on 'connection' (after auth), remove on 'disconnect'
  Never trust client-supplied identity — verify via JWT in io.use() middleware

COMMON GOTCHAS
  io.emit() when you meant broadcast.emit() → sender sees own message twice
  Forgetting the Redis adapter in a multi-instance deploy → events silently dropped
  Trusting client-sent userId/roomId without auth → security hole
```

---

## Connected topics

- **38 — WebSockets fundamentals** — Socket.IO is built on top of the WebSocket protocol (with polling fallback); understanding the raw upgrade handshake explains what Socket.IO is abstracting away.
- **92 — Connecting to Redis** — the Redis adapter used to scale Socket.IO across instances requires the same `ioredis`/`redis` connection concepts covered there.
- **101 — Pub/Sub with Redis** — the Redis adapter works by publishing every emitted event to a Redis channel that every server instance subscribes to — the exact pub/sub pattern taught in that topic.
