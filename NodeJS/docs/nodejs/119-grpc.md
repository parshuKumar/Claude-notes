# 119 — gRPC with Node.js

## What is this?

gRPC is a way for two services to call each other's functions directly over the network, as if they were local function calls — instead of building a URL, choosing a verb, and parsing JSON like you do with REST. It uses **Protocol Buffers** (a compact binary format) to define the data shapes and the available functions in one `.proto` file, then generates the client and server code from it. Think of it like a strict, pre-agreed phone script between two departments in a company: both sides know exactly what fields will be spoken and in what order, so there's no need to describe the conversation format every single call — it's faster and less error-prone than writing a free-form letter (REST/JSON) each time.

## Why does it matter for backend development?

Backend systems today are rarely a single server — they're a mesh of microservices that talk to each other constantly (order-service calling inventory-service calling payment-service). REST/JSON works, but it's verbose on the wire, has no strict contract enforcement, and doesn't support true bidirectional streaming easily. gRPC gives you a **strongly-typed contract** (the `.proto` file), **much smaller and faster binary payloads**, and **native streaming** (client, server, or both sides can stream data continuously over one connection). Companies like Google, Netflix, and most cloud-native/Kubernetes shops use gRPC for internal service-to-service communication, while still exposing REST or GraphQL to public-facing clients. A backend developer needs gRPC to build fast, reliable internal APIs and to work in any microservices-based company.

---

## Syntax / API

```proto
// File: protos/user.proto
// This is the CONTRACT — both client and server generate code from this file

syntax = "proto3";              // use proto3 syntax (the modern version)

package userpkg;                // namespace to avoid name collisions with other .proto files

// The service definition — lists the RPC methods this service exposes
service UserService {
  // Unary RPC: one request in, one response out (like a normal REST call)
  rpc GetUser (UserRequest) returns (UserResponse);

  // Server streaming RPC: one request in, a STREAM of responses out
  rpc ListUserOrders (UserRequest) returns (stream OrderResponse);
}

// Message = the shape of the data, like a TypeScript interface but for the wire
message UserRequest {
  string userId = 1;             // field name + a unique field NUMBER (not a default value)
}

message UserResponse {
  string userId = 1;             // field 1
  string email = 2;              // field 2
  string fullName = 3;           // field 3
}

message OrderResponse {
  string orderId = 1;            // field 1
  double amount = 2;             // field 2
}
```

```js
// File: src/grpc/server.js
// The Node.js gRPC SERVER — implements the contract defined above

const grpc = require('@grpc/grpc-js');          // core gRPC runtime for Node
const protoLoader = require('@grpc/proto-loader'); // turns .proto text into JS objects at runtime
const path = require('path');

// Load and parse the .proto file into a descriptor gRPC can use
const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '..', '..', 'protos', 'user.proto'),
  { keepCase: true, longs: String, enums: String, defaults: true } // formatting options
);

// Build the actual JS package object out of the descriptor
const userProto = grpc.loadPackageDefinition(packageDefinition).userpkg;

// Implementation of the unary RPC — mirrors an Express route handler
function getUser(call, callback) {
  const { userId } = call.request;              // incoming request fields, just a plain object
  const user = { userId, email: 'a@x.com', fullName: 'Ada Lovelace' }; // pretend DB lookup
  callback(null, user);                          // (error, response) — same shape as Node callbacks
}

// Create the gRPC server and register the service implementation
const server = new grpc.Server();
server.addService(userProto.UserService.service, { getUser }); // wire up method names to functions

// Bind to a port and start listening (insecure = no TLS, fine for local dev)
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  console.log('gRPC server running on port 50051');
});
```

---

## How it works — line by line

The `.proto` file is a plain text contract that both sides agree on ahead of time. `syntax = "proto3"` just picks the current version of the Protocol Buffers language. `service UserService` groups related remote functions together, similar to how a REST controller groups related routes. Each `rpc` line declares one callable function — `GetUser` takes a `UserRequest` and gives back one `UserResponse` (this is "unary," meaning one-and-done, just like a typical REST request/response). `rpc ListUserOrders` returns `stream OrderResponse`, meaning the server can keep sending multiple `OrderResponse` messages over the same connection until it decides it's done — useful for large result sets or live updates.

Each `message` block is a data shape, and each field inside it has a name and a **field number** (the `= 1`, `= 2`). That number — not the field name — is what actually gets sent over the wire in the compact binary encoding, which is why Protocol Buffers payloads are so much smaller than JSON: numbers replace repeated string keys.

On the server side, `proto-loader` reads that `.proto` file at runtime and turns it into a JavaScript description gRPC understands. `grpc.loadPackageDefinition` converts that description into an actual JS object tree (`userProto.UserService`) you can program against. The `getUser` function is the real implementation — it receives a `call` object (whose `.request` holds the incoming fields, already parsed into a plain JS object) and a `callback` you invoke exactly like a Node error-first callback: `callback(error, result)`. Registering it with `server.addService(...)` tells gRPC "when a client calls `GetUser`, run this function." Finally `bindAsync` opens a network port and starts the server, exactly like `app.listen()` in Express, except gRPC talks HTTP/2 with binary Protobuf frames instead of HTTP/1.1 with text/JSON.

---

## Example 1 — basic

```proto
// File: protos/greeting.proto
// A minimal one-method service, just to see the whole flow end to end

syntax = "proto3";                       // modern proto syntax
package greetpkg;                        // namespace

service GreetingService {
  rpc SayHello (HelloRequest) returns (HelloResponse); // one unary RPC
}

message HelloRequest {
  string name = 1;                       // field 1: the caller's name
}

message HelloResponse {
  string message = 2;                    // field 2: the greeting text
}
```

```js
// File: src/grpc/greetingServer.js
// Server: implements SayHello

const grpc = require('@grpc/grpc-js');           // gRPC runtime
const protoLoader = require('@grpc/proto-loader'); // proto file parser
const path = require('path');

// Parse the .proto file synchronously (fine — happens once at startup)
const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '..', '..', 'protos', 'greeting.proto')
);
const greetProto = grpc.loadPackageDefinition(packageDefinition).greetpkg; // access the package

// The actual handler function for SayHello
function sayHello(call, callback) {
  const requestBody = call.request;               // { name: '...' } sent by the client
  const responseMessage = `Hello, ${requestBody.name}!`; // build the reply text
  callback(null, { message: responseMessage });   // (error, response) — null error means success
}

const server = new grpc.Server();                                    // create the server instance
server.addService(greetProto.GreetingService.service, { sayHello }); // register method → handler

server.bindAsync('0.0.0.0:50052', grpc.ServerCredentials.createInsecure(), () => {
  console.log('Greeting gRPC server listening on 50052'); // confirm startup
});
```

```js
// File: src/grpc/greetingClient.js
// Client: calls SayHello on the server above

const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '..', '..', 'protos', 'greeting.proto')
);
const greetProto = grpc.loadPackageDefinition(packageDefinition).greetpkg;

// Create a client stub connected to the server's address
const client = new greetProto.GreetingService(
  'localhost:50052',
  grpc.credentials.createInsecure()               // no TLS, fine for local dev
);

// Call the remote method just like calling a local async function with a callback
client.sayHello({ name: 'Vishal' }, (err, response) => {
  if (err) {                                       // network error or server-side error
    console.error('gRPC call failed:', err.message);
    return;
  }
  console.log(response.message);                   // → "Hello, Vishal!"
});
```

---

## Example 2 — real world backend use case

```proto
// File: protos/order.proto
// A realistic internal service: order-service exposes order data to other microservices
// Includes a unary RPC AND a server-streaming RPC

syntax = "proto3";
package orderpkg;

service OrderService {
  rpc GetOrder (OrderIdRequest) returns (OrderResponse);            // fetch a single order
  rpc StreamOrdersForUser (UserIdRequest) returns (stream OrderResponse); // stream many orders
}

message OrderIdRequest {
  string orderId = 1;
}

message UserIdRequest {
  string userId = 1;
}

message OrderResponse {
  string orderId = 1;
  string userId = 2;
  double totalAmount = 3;
  string status = 4;             // e.g. "pending", "shipped", "delivered"
}
```

```js
// File: src/grpc/orderServer.js
// order-service — this is what inventory-service or a billing job would call internally

const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '..', '..', 'protos', 'order.proto'),
  { keepCase: true, longs: String, defaults: true }
);
const orderProto = grpc.loadPackageDefinition(packageDefinition).orderpkg;

// Pretend this is your real database layer (e.g. pg pool query in production)
const ordersDb = [
  { orderId: 'ord_1', userId: 'user_42', totalAmount: 59.99, status: 'shipped' },
  { orderId: 'ord_2', userId: 'user_42', totalAmount: 12.50, status: 'pending' },
  { orderId: 'ord_3', userId: 'user_77', totalAmount: 200.0, status: 'delivered' },
];

// Unary handler — fetch one order by ID
function getOrder(call, callback) {
  const { orderId } = call.request;                       // destructure the request
  const order = ordersDb.find((o) => o.orderId === orderId); // look it up

  if (!order) {
    // gRPC has its own status codes, similar in spirit to HTTP status codes
    return callback({
      code: grpc.status.NOT_FOUND,                         // maps roughly to HTTP 404
      message: `Order ${orderId} not found`,
    });
  }

  callback(null, order);                                   // success — send the order back
}

// Server-streaming handler — sends multiple messages over one call, no callback needed
function streamOrdersForUser(call) {
  const { userId } = call.request;                         // who we're fetching orders for
  const userOrders = ordersDb.filter((o) => o.userId === userId); // filter matching orders

  userOrders.forEach((order) => {
    call.write(order);                                     // push one order down the stream
  });

  call.end();                                              // signal "no more data, I'm done"
}

const server = new grpc.Server();
server.addService(orderProto.OrderService.service, {
  getOrder,                                                // wire up unary method
  streamOrdersForUser,                                     // wire up streaming method
});

server.bindAsync('0.0.0.0:50053', grpc.ServerCredentials.createInsecure(), () => {
  console.log('Order gRPC service listening on 50053');
});
```

```js
// File: src/grpc/orderClient.js
// billing-service (or any consumer) calling order-service internally

const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '..', '..', 'protos', 'order.proto')
);
const orderProto = grpc.loadPackageDefinition(packageDefinition).orderpkg;

const client = new orderProto.OrderService(
  'localhost:50053',
  grpc.credentials.createInsecure()
);

// Unary call — get a single order
client.getOrder({ orderId: 'ord_1' }, (err, order) => {
  if (err) {
    console.error(`getOrder failed [${err.code}]:`, err.message); // log gRPC status code + message
    return;
  }
  console.log('Fetched order:', order);
});

// Server-streaming call — listen for a stream of order events
const call = client.streamOrdersForUser({ userId: 'user_42' }); // returns a readable-stream-like object

call.on('data', (order) => {
  console.log('Received order chunk:', order);             // fires once per streamed message
});

call.on('end', () => {
  console.log('Stream finished — no more orders coming');   // server called call.end()
});

call.on('error', (err) => {
  console.error('Stream error:', err.message);              // network drop or server-side throw
});
```

---

## Common mistakes

### Mistake 1 — Mismatched field numbers between client and server proto files

```proto
// ❌ WRONG — server and client are using DIFFERENT .proto versions with reordered fields
// Server's message (deployed last week):
message UserResponse {
  string userId = 1;
  string email = 2;
}
// Client's message (older copy, someone reordered fields by hand):
message UserResponse {
  string email = 1;    // now field 1 means something different!
  string userId = 2;
}
// Result: client reads userId into the email field and vice versa — silent data corruption

// ✅ CORRECT — keep the .proto file in ONE shared location (a shared package or git submodule)
// and NEVER change existing field numbers — only ADD new fields with new numbers
message UserResponse {
  string userId = 1;   // never renumber existing fields
  string email = 2;
  string phone = 3;    // adding a new field is safe — old clients just ignore it
}
```

### Mistake 2 — Using unary calls for large result sets instead of streaming

```js
// ❌ WRONG — loading 100,000 rows into one giant response blocks memory and the connection
function listAllOrders(call, callback) {
  const allOrders = ordersDb.getAll();      // pulls everything into memory at once
  callback(null, { orders: allOrders });    // one huge message — slow, memory-heavy, times out
}

// ✅ CORRECT — use a server-streaming RPC and write rows as they're fetched
function streamAllOrders(call) {
  const cursor = ordersDb.streamCursor();   // e.g. a DB cursor / async generator
  cursor.on('row', (order) => call.write(order)); // send each row as it arrives
  cursor.on('end', () => call.end());       // close the stream once the cursor is exhausted
}
```

### Mistake 3 — Forgetting that gRPC errors are NOT thrown exceptions, they're callback errors with status codes

```js
// ❌ WRONG — try/catch around a callback-based gRPC call does nothing;
// the error never gets thrown, it's delivered via the callback's first argument
try {
  client.getUser({ userId: 'bad_id' }, (err, response) => {
    console.log(response.email); // crashes here if err was set and response is undefined
  });
} catch (e) {
  console.error('caught:', e);   // this block NEVER runs — no synchronous throw happens
}

// ✅ CORRECT — always check the callback's error argument first, using gRPC's status codes
client.getUser({ userId: 'bad_id' }, (err, response) => {
  if (err) {
    // grpc.status.NOT_FOUND, UNAUTHENTICATED, INTERNAL, etc. — check err.code
    console.error(`gRPC error [${err.code}]: ${err.message}`);
    return;
  }
  console.log(response.email);   // safe — we know response exists here
});
```

---

## Practice exercises

### Exercise 1 — easy

Create a `protos/calculator.proto` file defining a `CalculatorService` with one unary RPC called `Add` that takes a message with two numbers (`a`, `b`) and returns a message with one number (`result`). Then write a Node.js gRPC server that implements `Add` by returning `a + b`, and a separate client script that calls `Add(3, 5)` and logs the result.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an `inventory.proto` file with an `InventoryService` that has:
1. A unary RPC `GetStock` — takes a `productId`, returns `{ productId, quantity }`
2. A server-streaming RPC `StreamLowStockItems` — takes an empty request and streams back every product whose quantity is below 10

Implement the server with an in-memory array of at least 5 fake products (mix of low and high stock), and write a client that calls both RPCs and logs the results, including handling the `end` event on the stream properly.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design a `notification.proto` with a `NotificationService` exposing a **bidirectional streaming** RPC called `Chat` — the client can keep sending `{ userId, message }` messages, and the server keeps sending back `{ userId, message, timestamp }` acknowledgements on the same open connection, in real time, without either side closing the stream first. Implement:
1. The server side using `call.on('data', ...)` to receive incoming messages and `call.write(...)` to send acknowledgements back
2. A client that sends 3 messages spaced 1 second apart (`setTimeout`) and logs every acknowledgement it receives
3. Proper cleanup — the client should call `call.end()` after its last message, and the server should call `call.end()` once it detects the client's `end` event

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PACKAGES
  @grpc/grpc-js      → the gRPC runtime for Node (pure JS, no native bindings)
  @grpc/proto-loader → parses .proto files into JS descriptors at runtime

PROTO FILE BASICS
  syntax = "proto3";          → always use proto3 (modern syntax)
  message Foo { string x = 1; } → field NUMBERS matter, not just names — never reuse/reorder them
  service Foo { rpc Bar(Req) returns (Res); } → unary RPC
  rpc Bar(Req) returns (stream Res);          → server streaming
  rpc Bar(stream Req) returns (Res);          → client streaming
  rpc Bar(stream Req) returns (stream Res);   → bidirectional streaming

SERVER SETUP
  protoLoader.loadSync(path)                        → parse the .proto file
  grpc.loadPackageDefinition(def).pkgName            → get the JS package object
  new grpc.Server()                                  → create server instance
  server.addService(Proto.Service.service, { fn })   → register method implementations
  server.bindAsync(addr, credentials, callback)      → start listening

CLIENT SETUP
  new Proto.Service(address, grpc.credentials.createInsecure())
  client.methodName(request, (err, response) => {})  → unary call
  const call = client.streamMethod(request)          → server streaming — returns event emitter
  call.on('data' | 'end' | 'error', handler)

ERROR HANDLING
  Errors arrive via callback(err, ...) — NOT thrown exceptions
  err.code    → grpc.status.NOT_FOUND, UNAUTHENTICATED, INTERNAL, etc.
  err.message → human-readable description

gRPC vs REST — WHEN TO USE WHICH
  gRPC → internal service-to-service calls, need speed, need streaming, strict typed contracts
  REST → public APIs, browser clients, simple CRUD, human-readable debugging (curl/Postman)
  gRPC uses HTTP/2 + binary Protobuf   → smaller payloads, multiplexed connections, faster
  REST uses HTTP/1.1 (or 2) + JSON     → easier to inspect, wider tooling/browser support

GOTCHAS
  Browsers cannot call gRPC directly — need grpc-web or a REST gateway in front
  Field numbers in .proto are the wire contract — changing them breaks compatibility
  createInsecure() is dev-only — production needs TLS credentials
  Streaming calls behave like EventEmitters, not promises — no try/catch around callbacks
```

---

## Connected topics

- **20 — http module** — gRPC runs on top of HTTP/2, so understanding raw HTTP request/response handling in Node makes gRPC's transport layer easier to grasp.
- **37 — HTTP/2 in Node** — gRPC's multiplexing and streaming behavior comes directly from HTTP/2's capabilities; this topic explains that transport in depth.
- **120 — GraphQL with Node.js** — another alternative to plain REST for building APIs; comparing gRPC and GraphQL clarifies when to choose typed RPC vs flexible query-based APIs.
