# 63 — Logging in Node

## What is this?

Logging is the practice of recording what your application is doing while it runs — who requested what, what succeeded, what failed, and why. Think of it as a ship's log book: the captain doesn't remember every wave and course correction, but the log book records timestamps, events, and decisions so anyone can reconstruct exactly what happened during a voyage, especially after something goes wrong. In Node.js, logging ranges from a simple `console.log()` to structured, leveled, machine-readable logs produced by dedicated libraries like **winston** and **pino**.

## Why does it matter for backend development?

A backend server runs unattended, often on a remote machine you never directly watch. When a user reports "the checkout failed at 3am," `console.log` output scattered in a terminal you can't see is useless — you need **structured logs** that a log aggregator (Datadog, ELK, CloudWatch) can search, filter by request ID, and alert on. Production logging also needs **log levels** (so you can silence noisy debug logs in prod but keep errors), **structured JSON** (so machines can parse fields like `userId` or `statusCode`), and **request logging** (so every HTTP call is traceable end-to-end). Without proper logging, debugging a production incident is like investigating a crime with no witnesses and no camera footage.

---

## Syntax / API

```js
// ── console — built-in, zero setup, fine for local dev only ──────────────────
console.log('Server started');          // stdout — general info
console.error('Connection failed');     // stderr — errors, separate stream
console.warn('Deprecated route used');  // stderr — warnings
console.info('Cache warmed');           // stdout — alias for console.log

// ── pino — fastest structured JSON logger, minimal API ────────────────────────
const pino = require('pino');           // npm install pino
const logger = pino();                  // default logger, logs JSON to stdout

logger.info('Server started');                       // { level, time, msg: "Server started" }
logger.info({ userId: 'user_42' }, 'User logged in'); // merges object + message into JSON
logger.error({ err: new Error('DB down') }, 'Query failed'); // logs error with stack trace
logger.debug('Cache miss for key: %s', 'session_123'); // only shown if level allows debug

// ── winston — flexible, config-heavy, multiple transports ────────────────────
const winston = require('winston');     // npm install winston

const winstonLogger = winston.createLogger({
  level: 'info',                                    // minimum level to log
  format: winston.format.json(),                    // output as structured JSON
  transports: [                                     // where logs are sent
    new winston.transports.Console(),               // print to terminal
    new winston.transports.File({ filename: 'error.log', level: 'error' }), // errors to file
  ],
});

winstonLogger.info('Server started');               // logged as JSON
winstonLogger.error('Payment failed', { orderId: 'ord_9' }); // extra metadata attached
```

---

## How it works — line by line

`console.log` writes directly to `stdout` and `console.error`/`console.warn` write to `stderr` — two separate output streams. This is fine for local development but has no concept of **levels** (you cannot turn off "debug" noise without deleting code) and outputs plain text, which is hard for machines to parse at scale.

**Log levels** are a ranked severity scale, from most to least severe: `fatal` → `error` → `warn` → `info` → `debug` → `trace`. Every logging library lets you set a **minimum level** (e.g. `'info'`) — only messages at that level or more severe get printed. In production you typically run at `'info'`, and switch to `'debug'` temporarily while investigating an issue.

**Pino** is built for speed: it serializes objects to JSON as fast as possible and does almost no formatting work itself (you pipe its output through `pino-pretty` in dev for human-readable colors). It is the go-to choice when logging volume is high and performance matters, such as logging every incoming HTTP request.

**Winston** trades some speed for flexibility: it supports multiple **transports** (console, file, HTTP endpoint, database) at once, each with its own level and format, and lets you combine formatters (timestamp, colorize, JSON) declaratively. It is the go-to choice when you need one log call to simultaneously print to the console *and* write errors to a file *and* ship warnings to a remote service.

**Structured JSON logging** means every log line is a JSON object with consistent fields (`level`, `time`, `msg`, and custom fields like `userId` or `requestId`) instead of a free-form string. This lets log aggregators query "show me all errors for `userId: user_42` in the last hour" instead of grepping text.

**Request logging** means automatically logging metadata about every incoming HTTP request (method, URL, status code, response time) — usually via middleware that wraps every route, so you never have to remember to log it manually per endpoint.

---

## Example 1 — basic

```js
// File: src/utils/logger.js
// Setting up a pino logger with human-readable output in dev, JSON in prod.

const pino = require('pino'); // npm install pino pino-pretty

// process.env.NODE_ENV distinguishes dev vs production behavior (Topic 12)
const isDev = process.env.NODE_ENV !== 'production';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info', // default to 'info' if not set
  // In dev, pretty-print with colors; in prod, emit raw JSON for log shippers
  transport: isDev
    ? { target: 'pino-pretty', options: { colorize: true } } // human-friendly console
    : undefined,                                             // undefined = raw JSON to stdout
});

// Basic messages at different levels
logger.debug('This only shows if LOG_LEVEL=debug'); // suppressed by default at 'info'
logger.info('Server booting up');                   // always visible at 'info' level
logger.warn('Cache miss — falling back to database'); // non-fatal issue
logger.error(new Error('Failed to connect to Redis')); // captures message + stack trace

// Attaching structured context — merges into the JSON log line
logger.info({ userId: 'user_42', action: 'login' }, 'User authenticated');
// → {"level":30,"time":...,"userId":"user_42","action":"login","msg":"User authenticated"}

module.exports = logger; // exported once, reused across the whole app
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/requestLogger.js
// Express middleware that logs every request with a unique request ID,
// status code, and response time — the standard pattern for API observability.

const crypto = require('crypto');       // for generating a request ID (Topic 23)
const logger = require('../utils/logger'); // shared pino instance from Example 1

function requestLogger(req, res, next) {
  const requestId = crypto.randomUUID(); // unique ID to trace this request across logs
  const startTime = Date.now();          // capture start time to measure duration

  req.requestId = requestId;             // attach to req so other middleware/handlers can use it
  req.log = logger.child({ requestId }); // child logger auto-includes requestId in every call

  req.log.info(
    { method: req.method, url: req.originalUrl, ip: req.ip }, // structured request metadata
    'Incoming request'                                         // human-readable message
  );

  // Hook into the 'finish' event — fires after the response has been fully sent
  res.on('finish', () => {
    const durationMs = Date.now() - startTime; // total time to handle this request

    req.log.info(
      {
        statusCode: res.statusCode,   // e.g. 200, 404, 500
        durationMs,                   // how long the request took
      },
      'Request completed'
    );
  });

  next(); // pass control to the next middleware/route handler
}

module.exports = requestLogger;

// Usage in server.js:
// const requestLogger = require('./middleware/requestLogger');
// app.use(requestLogger); // mount before routes so every request is logged

// Inside a route handler you can now use the per-request logger:
// app.get('/api/orders/:orderId', (req, res) => {
//   req.log.info({ orderId: req.params.orderId }, 'Fetching order');
//   res.json({ orderId: req.params.orderId, status: 'shipped' });
// });
```

---

## Common mistakes

### Mistake 1 — Using console.log everywhere in production

```js
// ❌ WRONG — no levels, no structure, cannot be filtered or shipped to a log aggregator
console.log('User ' + userId + ' placed order ' + orderId);
// Impossible to search "all orders for userId X" without regex-parsing free text

// ✅ CORRECT — structured, leveled, machine-parseable
const logger = require('./utils/logger');
logger.info({ userId, orderId }, 'Order placed');
// Log aggregators can now query on the "userId" and "orderId" fields directly
```

### Mistake 2 — Logging sensitive data (passwords, tokens, card numbers)

```js
// ❌ WRONG — secrets end up in log files, log aggregators, and backups forever
logger.info({ authToken, password: requestBody.password }, 'Login attempt');
// Anyone with log access (including third-party log SaaS) now has live credentials

// ✅ CORRECT — redact or omit sensitive fields before logging
logger.info(
  { userId: requestBody.userId, hasPassword: Boolean(requestBody.password) },
  'Login attempt'
);
// Or use a redaction option: pino({ redact: ['requestBody.password', 'authToken'] })
```

### Mistake 3 — Logging errors without enough context to debug them

```js
// ❌ WRONG — just the message, no stack trace, no request context, no idea which user
try {
  await dbConnection.query(sql, params);
} catch (err) {
  console.log('Something went wrong'); // useless — which query? which request?
}

// ✅ CORRECT — log the actual error object (stack trace) plus surrounding context
try {
  await dbConnection.query(sql, params);
} catch (err) {
  req.log.error(
    { err, sql, requestId: req.requestId }, // pino/winston serialize err.stack automatically
    'Database query failed'
  );
  throw err; // re-throw so centralized error handling (Topic 62) can respond to the client
}
```

---

## Practice exercises

### Exercise 1 — easy

Set up a pino logger in a standalone file (`logger.js`) that:
1. Reads its minimum log level from `process.env.LOG_LEVEL`, defaulting to `'info'` if not set
2. Exports the logger with `module.exports`
3. In a separate script, require the logger and log one message at each level: `debug`, `info`, `warn`, `error`
4. Run the script once with `LOG_LEVEL=debug node script.js` and once with no env var set, and observe which messages appear in each run

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express request-logging middleware (without a library — plain `console` is fine for this exercise) that:
1. Generates a short random request ID for every incoming request (e.g. using `crypto.randomUUID()`)
2. Logs the method, URL, and request ID when the request comes in
3. Logs the status code, request ID, and duration in milliseconds when the response finishes (use the `res.on('finish', ...)` event)
4. Attaches the request ID to `req` so route handlers can reference it
5. Mount it on a small Express app with two routes and verify both log lines share the same request ID for a given call

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small logging module that supports **structured JSON logging with levels and redaction**, without using any external library:
1. A `createLogger(options)` factory that accepts `{ level, redactKeys }`
2. Support levels `debug < info < warn < error`, where a message is only printed if its level is >= the configured `level`
3. Each log call takes `(level, meta, message)` and prints one line of JSON containing `timestamp`, `level`, `message`, and the `meta` object
4. Before printing, recursively walk the `meta` object and replace the value of any key listed in `redactKeys` with `"[REDACTED]"` (e.g. redact `password`, `authToken`)
5. Add a `.child(extraMeta)` method that returns a new logger which automatically merges `extraMeta` (e.g. `{ requestId }`) into every subsequent log call
6. Test it by logging a login attempt that includes a `password` field and confirming it prints as `[REDACTED]`, and by creating a child logger with a `requestId` and confirming it appears on every log line

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
LOG LEVELS (least → most severe)
  trace < debug < info < warn < error < fatal
  Setting level: "info" means debug/trace are suppressed, warn/error/fatal still show

WHEN TO USE WHAT
  console.log/error         → quick local debugging only, never ship to prod as-is
  pino                      → high-throughput apps, fastest JSON serialization, needs pino-pretty in dev
  winston                   → need multiple transports (console + file + remote) or custom formats

STRUCTURED JSON LOGGING
  logger.info({ userId, orderId }, 'Order placed')
  → { "level": 30, "time": 169..., "userId": "u1", "orderId": "o9", "msg": "Order placed" }
  Machine-parseable → searchable/filterable in log aggregators (ELK, Datadog, CloudWatch)

REQUEST LOGGING PATTERN
  1. Generate a requestId (crypto.randomUUID()) per incoming request
  2. Log "incoming request" with method/url/requestId
  3. On res.on('finish'), log statusCode + durationMs + requestId
  4. Use logger.child({ requestId }) so every log in that request auto-includes it

NEVER LOG
  Passwords, auth tokens, API keys, full card numbers, session secrets
  → redact them, or log only booleans like hasPassword: true

PINO QUICK API
  pino()                          → JSON logger to stdout
  pino({ level: 'debug' })        → set minimum level
  logger.child({ requestId })     → scoped logger with extra fields baked in
  { target: 'pino-pretty' }       → human-readable dev output (install pino-pretty)

WINSTON QUICK API
  winston.createLogger({ level, format, transports })
  winston.format.json() / winston.format.combine(...)
  new winston.transports.Console() / new winston.transports.File({ filename })

GOTCHAS
  console.error/warn write to stderr, console.log/info write to stdout — different streams
  Logging is I/O — extremely high-frequency sync logging can slow down hot paths
  Always log the Error OBJECT (not just err.message) to keep the stack trace
```

---

## Connected topics

- **62 — Centralized error handling** — the global error handler is where caught errors get logged before a response is sent to the client
- **55 — Express middleware in depth** — request logging is implemented as middleware, run for every route before the handler executes
- **111 — Monitoring and alerting** — structured logs are the raw data that monitoring tools (Prometheus/Grafana, APM) parse to generate metrics and alerts
