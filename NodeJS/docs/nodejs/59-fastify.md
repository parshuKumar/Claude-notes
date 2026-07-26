# 59 — Fastify introduction

## What is this?

Fastify is a Node.js web framework, just like Express, that focuses on speed, low overhead, and built-in schema validation. Think of Express as a general-purpose toolbox that lets you plug in anything you want, while Fastify is more like a well-organized workshop with labeled drawers — it comes with a strict but fast plugin system, a lifecycle with well-defined hooks, and validation baked directly into how you define routes. Both frameworks solve the same problem (routing HTTP requests to handler functions), but Fastify was built from the ground up (starting 2016) with performance benchmarks and JSON Schema validation as first-class citizens rather than afterthoughts.

## Why does it matter for backend development?

As APIs grow, two things usually cause pain: requests crashing the server because nobody validated the input, and the framework itself becoming a bottleneck under high traffic. Fastify addresses both directly — every route can declare a JSON Schema for its body, query, params, and even response shape, so invalid data is rejected automatically before your handler code ever runs, and the same schema is used to serialize the response faster than `JSON.stringify()` would on its own. Companies choose Fastify for high-throughput APIs (payment gateways, high-traffic public APIs, microservices) where every millisecond of latency and every dropped invalid request matters. As a backend developer, knowing Fastify shows you understand that "which framework" is a real architectural decision, not just a taste preference — Express is easier to start with, Fastify is built for scale and correctness by design.

---

## Syntax / API

```js
// Install: npm install fastify

// Import the factory function — logger:true gives structured request logs for free
const fastify = require('fastify')({ logger: true });

// Define a route with a schema — Fastify validates AND serializes using this schema
fastify.get('/users/:userId', {
  schema: {
    // params schema — validates the :userId route parameter
    params: {
      type: 'object',
      properties: { userId: { type: 'string' } },
      required: ['userId'],
    },
    // response schema — Fastify uses this to fast-serialize the output JSON
    response: {
      200: {
        type: 'object',
        properties: {
          userId: { type: 'string' },
          email: { type: 'string' },
        },
      },
    },
  },
}, async (request, reply) => {
  // request.params is already validated to match the schema above
  const { userId } = request.params;
  return { userId, email: `${userId}@example.com` }; // returning an object = auto JSON response
});

// Lifecycle hook — runs before EVERY route handler, good for auth checks
fastify.addHook('onRequest', async (request, reply) => {
  request.log.info('Incoming request'); // request.log is the built-in Pino logger
});

// A plugin — a self-contained, reusable unit of routes/decorators/hooks
fastify.register(require('./routes/userRoutes'), { prefix: '/api/v1' });

// Start the server — listen() returns a Promise in modern Fastify
fastify.listen({ port: 3000, host: '0.0.0.0' })
  .then((address) => fastify.log.info(`Server ready at ${address}`))
  .catch((err) => {
    fastify.log.error(err); // log the startup error
    process.exit(1);        // exit if the server fails to bind
  });
```

---

## How it works — line by line

- `require('fastify')({ logger: true })` creates the app instance, same idea as `express()`, but it turns on structured logging (via the built-in Pino logger) with one flag.
- `fastify.get(path, options, handler)` registers a route. Unlike Express, the second argument can be an **options object** that includes a `schema` — this is where Fastify's validation engine plugs in.
- The `schema.params` block is a JSON Schema. Before your handler runs, Fastify checks the incoming `:userId` against this schema; if it fails, Fastify auto-responds with a `400 Bad Request` and your handler code never executes.
- The `schema.response` block does the opposite job: it tells Fastify exactly what shape a `200` response will have, so Fastify can pre-compile a fast serializer instead of calling the slower generic `JSON.stringify()` on every response.
- Inside the handler, `request.params` is already guaranteed to match the schema — no manual `if (!userId)` checks needed.
- Returning a plain object from an `async` handler is enough — Fastify automatically serializes it to JSON and sets the `Content-Type` header; there is no need to call `reply.send()` explicitly (though you can).
- `fastify.addHook('onRequest', ...)` registers a **lifecycle hook** — a function that runs at a specific stage of every request's journey through the framework, before routing/validation/handler execution.
- `fastify.register(plugin, options)` loads a **plugin** — a function that receives the Fastify instance and can add routes, decorators, or hooks, isolated in its own encapsulated context (covered more below).
- `fastify.listen({ port, host })` starts the HTTP server and returns a Promise, so you handle startup success/failure with `.then()`/`.catch()` instead of an error-event callback.

---

## Example 1 — basic

```js
// File: src/server.js
// A minimal Fastify server with one validated route.

const fastify = require('fastify')({ logger: true }); // create app with logging on

// Define a schema for creating a user — describes exactly what the body must look like
const createUserSchema = {
  body: {
    type: 'object',
    required: ['email', 'password'],       // both fields are mandatory
    properties: {
      email: { type: 'string', format: 'email' },  // must look like an email
      password: { type: 'string', minLength: 8 },   // must be at least 8 chars
    },
  },
};

// POST /users — schema is validated automatically before the handler runs
fastify.post('/users', { schema: createUserSchema }, async (request, reply) => {
  const { email, password } = request.body; // already validated — safe to use directly

  reply.code(201); // set status code 201 (Created)
  return {
    message: 'User created',
    email,                                  // never echo back the raw password
  };
});

// GET /health — a simple route with no schema, for a load balancer health check
fastify.get('/health', async () => {
  return { status: 'ok' }; // returned object becomes the JSON response body
});

// Start listening on port 3000
fastify.listen({ port: 3000 })
  .then(() => fastify.log.info('Server started on port 3000'))
  .catch((err) => {
    fastify.log.error(err);   // log any startup failure
    process.exit(1);          // exit the process so process managers can restart it
  });
```

---

## Example 2 — real world backend use case

```js
// File: src/plugins/authPlugin.js
// A reusable Fastify plugin that adds JWT authentication as a decorator + hook.
// This is the pattern real backend teams use to share auth logic across route files.

const jwt = require('jsonwebtoken');

// fastify-plugin wraps this so decorators are visible OUTSIDE this file's encapsulation
const fp = require('fastify-plugin');

async function authPlugin(fastify, options) {
  const apiKey = process.env.JWT_SECRET; // secret used to verify tokens, from .env

  // decorate() attaches a reusable method to every request object
  fastify.decorateRequest('authUser', null); // default value, overwritten per-request

  // Register an onRequest hook that runs before every route in this plugin's scope
  fastify.addHook('onRequest', async (request, reply) => {
    const authHeader = request.headers.authorization; // e.g. "Bearer eyJhbGciOi..."

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      reply.code(401);                       // Unauthorized
      throw new Error('Missing or malformed Authorization header');
    }

    const authToken = authHeader.split(' ')[1]; // extract the raw token string

    try {
      const decoded = jwt.verify(authToken, apiKey); // throws if invalid/expired
      request.authUser = { userId: decoded.userId };  // attach decoded user to request
    } catch (err) {
      reply.code(401);
      throw new Error('Invalid or expired token');
    }
  });
}

module.exports = fp(authPlugin); // fp() makes decorators visible to the parent app

// ── File: src/routes/orderRoutes.js ─────────────────────────────────────────
// Routes that consume the plugin above, with full schema validation.

async function orderRoutes(fastify, options) {
  fastify.register(require('../plugins/authPlugin')); // apply auth to this route group

  fastify.post('/orders', {
    schema: {
      body: {
        type: 'object',
        required: ['productId', 'quantity'],
        properties: {
          productId: { type: 'string' },
          quantity: { type: 'integer', minimum: 1 }, // reject zero/negative quantities
        },
      },
      response: {
        201: {
          type: 'object',
          properties: {
            orderId: { type: 'string' },
            userId: { type: 'string' },
          },
        },
      },
    },
  }, async (request, reply) => {
    const { productId, quantity } = request.body; // validated body
    const userId = request.authUser.userId;       // attached by the auth hook above

    const orderId = `order_${Date.now()}`; // placeholder ID generation
    // ... insert into database here using dbConnection ...

    reply.code(201);
    return { orderId, userId };
  });
}

module.exports = orderRoutes;

// ── File: src/server.js ──────────────────────────────────────────────────────
// const fastify = require('fastify')({ logger: true });
// fastify.register(require('./routes/orderRoutes'), { prefix: '/api/v1' });
// fastify.listen({ port: 3000 });
```

---

## Common mistakes

### Mistake 1 — Forgetting that returning a value already sends the response

```js
// ❌ WRONG — calling reply.send() AND returning a value causes a
// "FST_ERR_REP_ALREADY_SENT" error or double-response warning
fastify.get('/users/:userId', async (request, reply) => {
  reply.send({ userId: request.params.userId }); // sends response here
  return { userId: request.params.userId };      // ALSO tries to send — conflict
});

// ✅ CORRECT — pick ONE: either return, or call reply.send(), never both
fastify.get('/users/:userId', async (request, reply) => {
  return { userId: request.params.userId }; // return is enough in an async handler
});
```

### Mistake 2 — Writing schemas that silently strip fields you needed

```js
// ❌ WRONG — Fastify's default `additionalProperties: false` behavior means
// any body field NOT listed in `properties` gets silently removed, not rejected
const schema = {
  body: {
    type: 'object',
    properties: {
      email: { type: 'string' },
      // 'password' was forgotten here
    },
  },
};
// A request sending { email, password } arrives in the handler with password missing —
// no error, just silently gone. Hard bug to notice.

// ✅ CORRECT — list every field the client is allowed to send
const schema = {
  body: {
    type: 'object',
    required: ['email', 'password'],
    properties: {
      email: { type: 'string', format: 'email' },
      password: { type: 'string', minLength: 8 }, // now included and validated
    },
    additionalProperties: false, // be explicit: reject anything not listed
  },
};
```

### Mistake 3 — Registering plugins in the wrong scope, losing encapsulation benefits

```js
// ❌ WRONG — registering the auth plugin globally on the root instance
// means EVERY route (including public ones like /health) requires a token
const fastify = require('fastify')();
fastify.register(require('./plugins/authPlugin')); // applies to the whole app
fastify.get('/health', async () => ({ status: 'ok' })); // now blocked without a token!

// ✅ CORRECT — register auth only inside the route group that needs it
const fastify = require('fastify')();

fastify.get('/health', async () => ({ status: 'ok' })); // public, no auth needed

fastify.register(async function protectedRoutes(instance) {
  instance.register(require('./plugins/authPlugin')); // auth scoped to this group only
  instance.get('/orders', async (request) => ({ orders: [] }));
}, { prefix: '/api/v1' });
```

---

## Practice exercises

### Exercise 1 — easy

Create a Fastify server with two routes:
1. `GET /health` — returns `{ status: 'ok' }`, no schema needed
2. `GET /products/:productId` — has a `params` schema requiring `productId` to be a string, and returns `{ productId, name: 'Sample Product' }`

Start the server on port 4000 using `fastify.listen()`, and log a message once it starts successfully.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `POST /signup` route with:
1. A `body` schema requiring `email` (string, format `email`) and `password` (string, `minLength: 8`)
2. A `response` schema for status `201` that only exposes `email` (never the password)
3. A handler that returns status `201` with the created user's email
4. A manual test: send a request missing `password` and confirm Fastify auto-rejects it with a `400` before your handler code runs (add a `console.log` inside the handler to prove it never executes for invalid input)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small plugin-based Fastify app:
1. Create a plugin `requestTimerPlugin` that uses `onRequest` and `onResponse` hooks to measure how long each request took (store the start time on the `request` object in `onRequest`, calculate the duration in `onResponse`, and log it)
2. Create a plugin `apiKeyPlugin` that checks for a custom header `x-api-key` and rejects the request with `401` if it doesn't match a value from `process.env.API_KEY`
3. Create a route group under prefix `/api/v1` that registers BOTH plugins and exposes a `GET /reports` route returning a sample array of report objects
4. Register a separate public route `GET /status` OUTSIDE that group, with neither plugin applied, and confirm it works without an API key

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CREATE APP
  const fastify = require('fastify')({ logger: true });

ROUTES
  fastify.get(path, [options], handler)
  fastify.post(path, [options], handler)
  fastify.put/patch/delete(...)         → same pattern

SCHEMA KEYS (inside { schema: {...} })
  body       → validates request body (POST/PUT/PATCH)
  querystring→ validates ?query=params
  params     → validates :routeParams
  headers    → validates request headers
  response   → { 200: {...}, 404: {...} } — validates AND fast-serializes output

HANDLER RESPONSE
  return obj                → auto JSON response, status 200 by default
  reply.code(201).send(obj) → explicit status + send
  NEVER call both return AND reply.send() in the same handler

LIFECYCLE HOOKS (in execution order)
  onRequest → preParsing → preValidation → preHandler
  → [ROUTE HANDLER] →
  preSerialization → onSend → onResponse

  fastify.addHook('onRequest', async (request, reply) => { ... })

PLUGINS
  fastify.register(pluginFn, options)      → encapsulated (isolated) by default
  require('fastify-plugin')(pluginFn)      → breaks encapsulation, shares decorators up

DECORATORS
  fastify.decorate('name', value)          → add to fastify instance
  fastify.decorateRequest('name', default) → add to every request object
  fastify.decorateReply('name', default)   → add to every reply object

START SERVER
  fastify.listen({ port, host })  → returns a Promise (no error-event callback needed)

FASTIFY VS EXPRESS
  Validation     → built-in JSON Schema        vs  manual / external lib (Joi, Zod)
  Serialization  → schema-based, faster        vs  generic JSON.stringify()
  Plugins        → encapsulated by default     vs  global middleware (app.use)
  Throughput     → generally higher (benchmarks)vs  slightly lower, huge ecosystem
  Learning curve → schemas add upfront work     vs  simpler to start
```

---

## Connected topics

- **53 — Express.js fundamentals** — the framework Fastify is most often compared against; understanding Express's middleware model makes Fastify's hooks and plugin encapsulation easier to appreciate
- **56 — Request validation** — Fastify replaces external validators like Joi/Zod with built-in JSON Schema validation directly on the route definition
- **115 — Plugin and middleware architecture** — Fastify's `register()`/encapsulation model is a concrete real-world example of the plugin architecture patterns covered in that topic
