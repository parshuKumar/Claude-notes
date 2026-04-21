# Node.js Master Curriculum — Complete Learning Plan

> **Goal:** Go from JavaScript-comfortable to backend-expert in Node.js.  
> **Background:** JS fully complete (variables, functions, OOP, async/await, promises, modules, classes, error handling).  
> **Path after Node.js:** Databases → OOP/SOLID → Design Patterns → LLD  
> **Format:** One `.md` file per topic, numbered in order, saved to `/docs/nodejs/`

---

## How to use this plan

| Command | What it does |
|---|---|
| `NEXT` | Move to the next topic |
| `PROGRESS` | List all completed docs |
| `REDO [topic name]` | Regenerate that doc from scratch |
| `HINT` | One small nudge on a stuck exercise |

---

## PHASE 1 — How Node.js Actually Works (Topics 01–07)

> Understand the runtime before writing a single line of server code.  
> This is what separates developers who *use* Node from those who *understand* it.

| # | Topic | Core Concept |
|---|---|---|
| 01 | What is Node.js | Runtime vs browser JS, V8 engine, libuv, what Node adds on top of JS, real-world use |
| 02 | How Node.js works internally | Single thread, event-driven architecture, non-blocking I/O, comparison to Java/Python/Go |
| 03 | The event loop in depth | Call stack, libuv thread pool, event queue, microtask queue, macrotask queue, task priority |
| 04 | process.nextTick vs setImmediate vs setTimeout(0) | Exact execution order, when to use each, common gotchas |
| 05 | The process object | process.env, process.argv, process.cwd(), process.exit(), process.on(), process.pid |
| 06 | \_\_dirname and \_\_filename | What they are, why they exist, ESM equivalent (import.meta.url), common use cases |
| 07 | Errors and error handling in Node | Error types, error-first callbacks, try/catch with async, uncaughtException, unhandledRejection |

---

## PHASE 2 — Module Systems (Topics 08–13)

> Every Node.js app is made of modules. Know both systems cold.

| # | Topic | Core Concept |
|---|---|---|
| 08 | CommonJS modules | require(), module.exports, exports, path resolution, caching, circular dependencies |
| 09 | ES Modules in Node | import/export, .mjs vs .cjs, "type":"module", differences from CJS, when to use which |
| 10 | package.json in depth | name, version, scripts, dependencies, devDependencies, main, type, engines, semver (^, ~, exact) |
| 11 | npm in depth | install, run, init, npx, global vs local, package-lock.json, node_modules, audit, prune, outdated |
| 12 | Environment variables | .env files, dotenv package, process.env, never hardcoding secrets, .env.example, cross-env |
| 13 | Module resolution algorithm | Exactly how Node finds files — node_modules lookup chain, index.js fallback, core module priority |

---

## PHASE 3 — Core Built-in Modules (Topics 14–33)

> Node's standard library. These are the building blocks every backend dev must know.

| # | Topic | Core Concept |
|---|---|---|
| 14 | path module | join(), resolve(), dirname(), basename(), extname(), sep, why never string concatenate paths |
| 15 | fs module — callbacks | readFile, writeFile, appendFile, unlink, mkdir, rmdir, readdir, stat, existsSync |
| 16 | fs module — promises API | fs/promises, async/await with filesystem, error handling, parallel reads with Promise.all |
| 17 | fs module — streams and large files | createReadStream, createWriteStream, pipe(), why streams, handling large files without OOM |
| 18 | os module | platform(), cpus(), freemem(), totalmem(), homedir(), hostname(), real backend use cases |
| 19 | events module | EventEmitter, on(), emit(), once(), off(), removeAllListeners(), building custom emitters |
| 20 | http module | Raw HTTP server, req/res objects, manual routing, reading request body, headers, status codes |
| 21 | https module | Difference from http, TLS/SSL concept, when you need https in Node vs terminating at proxy |
| 22 | url module | new URL(), searchParams, parsing query strings, building URLs programmatically |
| 23 | crypto module | createHash (sha256, md5), HMAC, randomBytes, randomUUID, createCipheriv — auth use cases |
| 24 | util module | promisify(), format(), inspect(), util.types, converting callback APIs to promise-based |
| 25 | buffer module | What a Buffer is, binary data, Buffer.from(), alloc(), toString(), encodings (utf8, base64, hex) |
| 26 | stream module in depth | Readable, Writable, Duplex, Transform, pipeline(), backpressure, custom transform stream |
| 27 | child_process module | exec(), spawn(), execFile(), fork(), running shell commands from Node, stdout/stderr handling |
| 28 | worker_threads module | CPU-heavy tasks, creating workers, postMessage, SharedArrayBuffer, vs child_process |
| 29 | cluster module | Forking across CPU cores, master/worker process, connection handling, throughput improvement |
| 30 | timers in Node | setTimeout, setInterval, setImmediate, process.nextTick, clearTimeout, event loop interaction |
| 31 | readline module | Reading line by line from stdin/files, creating CLI prompts, Interface class |
| 32 | querystring module | parse(), stringify(), difference from URLSearchParams, legacy vs modern approach |
| 33 | zlib module | gzip, deflate, inflate, compressing HTTP responses, streaming compression |

---

## PHASE 4 — Networking and Protocols (Topics 34–39)

> What actually happens on the wire. Critical for REST APIs and real-time apps.

| # | Topic | Core Concept |
|---|---|---|
| 34 | TCP and UDP with net/dgram | net.createServer(), sockets, client-server communication, UDP datagrams, when to use each |
| 35 | DNS module | dns.lookup(), dns.resolve(), dns.reverse(), how name resolution works from Node |
| 36 | HTTP deep dive — headers, methods, status codes | Request lifecycle, content-type, accept, authorization headers, idempotency, REST semantics |
| 37 | HTTP/2 in Node | http2 module, multiplexing, server push, difference from HTTP/1.1, when to use |
| 38 | WebSockets fundamentals | What WebSockets are, the upgrade handshake, ws library, real-time bidirectional communication |
| 39 | Server-Sent Events (SSE) | One-way real-time from server to client, when to use SSE vs WebSockets vs polling |

---

## PHASE 5 — File System & Data Patterns (Topics 40–44)

> Reading, writing, watching, and processing data — at scale.

| # | Topic | Core Concept |
|---|---|---|
| 40 | Watching files — fs.watch and chokidar | fs.watch(), fs.watchFile(), chokidar library, building a live-reload system |
| 41 | Working with JSON at scale | JSON.parse / stringify edge cases, streaming JSON (JSONStream), large JSON file handling |
| 42 | CSV and file parsing | Reading CSV with streams, papaparse / csv-parse, transforming data rows |
| 43 | Temporary files and directories | os.tmpdir(), tmp package, cleanup patterns, safe temp file handling |
| 44 | File uploads | multipart/form-data, reading binary streams, storing files safely, path traversal prevention |

---

## PHASE 6 — Asynchronous Patterns in Node.js (Topics 45–51)

> Node's superpower is async. Master every pattern.

| # | Topic | Core Concept |
|---|---|---|
| 45 | Callback pattern deep dive | Error-first callbacks, callback hell, pyramid of doom, how Node core uses callbacks |
| 46 | Promises in Node context | Promise.all, Promise.allSettled, Promise.race, Promise.any, chaining in backend context |
| 47 | async/await in Node — advanced | Top-level await, sequential vs parallel, error boundaries, async iterators |
| 48 | Event-driven patterns | EventEmitter patterns, decoupling with events, event bus pattern |
| 49 | Streams as async iterables | for await...of on streams, async generators, composing async iterables |
| 50 | Concurrency control | p-limit, p-queue — controlling how many async ops run at once, rate limiting patterns |
| 51 | Retry and timeout patterns | Implementing retry with backoff, request timeouts, AbortController in Node |

---

## PHASE 7 — Building HTTP Servers (Topics 52–60)

> From raw http to production-grade Express/Fastify patterns.

| # | Topic | Core Concept |
|---|---|---|
| 52 | Building a REST API with raw http | Manual router, parsing JSON body, sending JSON responses, no framework |
| 53 | Express.js fundamentals | app, Router, middleware, req/res/next, error middleware, app.use() |
| 54 | Express routing in depth | Route params, query strings, route groups, express.Router, mounting sub-routers |
| 55 | Express middleware in depth | Built-in, third-party, custom middleware, middleware chain, order matters |
| 56 | Request validation | Joi / Zod / express-validator, validating body/params/query, sanitization |
| 57 | Authentication patterns — JWT | jsonwebtoken, signing, verifying, refresh tokens, storing tokens safely, auth middleware |
| 58 | Authentication patterns — Sessions | express-session, cookie-session, session stores (Redis), CSRF |
| 59 | Fastify introduction | Why Fastify, schema-based validation, lifecycle hooks, plugins, performance vs Express |
| 60 | API versioning and structure | /api/v1/, route organization, controller/service/router pattern, project folder structure |

---

## PHASE 8 — Error Handling and Reliability (Topics 61–66)

> Code that fails gracefully is production code.

| # | Topic | Core Concept |
|---|---|---|
| 61 | Operational vs programmer errors | What to crash on vs what to recover from, error classification |
| 62 | Centralized error handling | Global error handler in Express, AppError class, error codes, consistent error responses |
| 63 | Logging in Node | console vs winston vs pino, log levels, structured JSON logging, request logging |
| 64 | Graceful shutdown | SIGTERM/SIGINT, draining connections, cleanup before exit, health check endpoints |
| 65 | Health checks and readiness probes | /health, /ready, /live endpoints — what Kubernetes expects |
| 66 | Circuit breaker pattern | opossum library, preventing cascade failures, fallback strategies |

---

## PHASE 9 — Performance and Scaling (Topics 67–73)

> Make Node fast and keep it fast under load.

| # | Topic | Core Concept |
|---|---|---|
| 67 | Profiling Node.js | --inspect flag, Chrome DevTools, node --prof, clinic.js, finding CPU hotspots |
| 68 | Memory management | Heap, garbage collection, memory leaks, heap snapshots, tracking allocations |
| 69 | Caching strategies | In-memory cache (node-cache), Redis caching, cache-aside pattern, TTL, cache invalidation |
| 70 | Rate limiting | express-rate-limit, sliding window, token bucket, Redis-backed rate limiting |
| 71 | Compression and performance headers | compression middleware, ETags, Cache-Control, Last-Modified |
| 72 | Load testing | autocannon, artillery, k6 — writing load tests, reading results, finding bottlenecks |
| 73 | Horizontal scaling with PM2 | pm2 start, cluster mode, ecosystem.config.js, monitoring, zero-downtime reload |

---

## PHASE 10 — Security (Topics 74–81)

> OWASP Top 10 for Node. Non-negotiable for any production backend.

| # | Topic | Core Concept |
|---|---|---|
| 74 | Security fundamentals in Node | Attack surface, principle of least privilege, dependency security |
| 75 | Input validation and sanitization | Preventing injection, DOMPurify/sanitize-html, validator.js, never trust user input |
| 76 | SQL injection prevention | Parameterized queries, prepared statements, ORM protections |
| 77 | NoSQL injection | MongoDB operator injection, sanitizing queries, mongoose protections |
| 78 | XSS, CSRF, Clickjacking | helmet.js, CORS configuration, SameSite cookies, X-Frame-Options |
| 79 | Secrets management | Never in code, never in git, dotenv, AWS Secrets Manager pattern, secret rotation |
| 80 | HTTPS, TLS, certificates | Let's Encrypt, self-signed certs in dev, SSL termination at load balancer vs Node |
| 81 | Dependency auditing | npm audit, Snyk, checking CVEs, keeping dependencies updated safely |

---

## PHASE 11 — Testing (Topics 82–89)

> Code you can't test can't be trusted.

| # | Topic | Core Concept |
|---|---|---|
| 82 | Testing fundamentals | Unit, integration, e2e — what each tests, test pyramid, when to use each |
| 83 | Jest fundamentals | describe, it, expect, matchers, setup/teardown, test files structure |
| 84 | Testing async code | Testing promises, async/await in tests, handling rejections |
| 85 | Mocking and stubbing | jest.mock(), jest.spyOn(), mocking modules, mocking HTTP calls with nock |
| 86 | Testing Express routes | supertest, testing req/res, testing middleware, testing error handlers |
| 87 | Integration testing with a real DB | Test database setup, transactions for rollback, seeding test data |
| 88 | Test coverage | Istanbul/nyc, coverage thresholds, what coverage does and doesn't tell you |
| 89 | TDD workflow | Red-green-refactor, writing tests first, TDD for a REST endpoint |

---

## PHASE 12 — Working with Databases (Topics 90–98)

> Node + database = real backend. Foundation before moving to the Databases phase.

| # | Topic | Core Concept |
|---|---|---|
| 90 | Connecting to PostgreSQL | pg driver, connection pool, executing queries, handling errors |
| 91 | Connecting to MongoDB | mongoose, connection lifecycle, models, schemas, CRUD |
| 92 | Connecting to Redis | ioredis, basic get/set/del, expiry, pub/sub, using Redis as cache/session store |
| 93 | Connection pooling | Why pools exist, pool size tuning, pool exhaustion, pg-pool config |
| 94 | Database migrations | node-pg-migrate / knex migrations, up/down migrations, migration workflow |
| 95 | Query builders — Knex | knex.select().where().join(), transactions, batch inserts |
| 96 | ORM — Prisma | Schema, prisma generate, CRUD, relations, migrations with Prisma |
| 97 | Transactions | ACID, BEGIN/COMMIT/ROLLBACK, nested transactions, handling transaction errors |
| 98 | Database seeding and fixtures | Seeding dev/test data, factories, faker.js |

---

## PHASE 13 — Real-time and Messaging (Topics 99–104)

> Event-driven backend patterns used in production.

| # | Topic | Core Concept |
|---|---|---|
| 99 | Socket.IO in depth | Rooms, namespaces, events, broadcasting, presence, scaling with Redis adapter |
| 100 | Message queues — BullMQ | Job queues, workers, retries, delayed jobs, progress tracking, Redis-backed |
| 101 | Pub/Sub with Redis | publish, subscribe, channels, fan-out pattern |
| 102 | Event-driven architecture basics | Events vs commands, event sourcing intro, why decouple with events |
| 103 | Working with RabbitMQ | amqplib, queues, exchanges, routing keys, acknowledgements |
| 104 | Webhooks — sending and receiving | Outbound webhooks with retry, verifying incoming webhook signatures (HMAC) |

---

## PHASE 14 — Deployment and DevOps Basics (Topics 105–112)

> Get your Node app running in the real world.

| # | Topic | Core Concept |
|---|---|---|
| 105 | Dockerizing a Node.js app | Dockerfile, .dockerignore, multi-stage builds, non-root user, image optimization |
| 106 | Docker Compose for local dev | docker-compose.yml, service linking, volumes, environment files |
| 107 | Environment management | dev/test/staging/prod configs, NODE_ENV, config libraries (convict, dotenv-flow) |
| 108 | CI/CD basics with GitHub Actions | Workflow files, running tests on push, building and pushing Docker images |
| 109 | Reverse proxy with Nginx | Nginx config for Node, proxy_pass, SSL termination, static file serving |
| 110 | Process management in production | PM2 vs systemd, auto-restart, log rotation, monitoring |
| 111 | Monitoring and alerting | Prometheus + prom-client, Grafana basics, alerting rules, APM (New Relic / Datadog intro) |
| 112 | 12-Factor App methodology | Config, logs, stateless processes, backing services — applied to Node.js |

---

## PHASE 15 — Advanced Node.js Patterns (Topics 113–120)

> Architect-level Node.js knowledge.

| # | Topic | Core Concept |
|---|---|---|
| 113 | Building a CLI tool | process.argv parsing, commander.js, chalk, ora (spinners), publishing to npm |
| 114 | Building npm packages | Package authoring, exports field, dual CJS/ESM builds, publishing, versioning |
| 115 | Plugin and middleware architecture | Building extensible systems, plugin registration, hook systems |
| 116 | Dependency injection in Node | Manual DI containers, awilix, inversify-lite, testability benefits |
| 117 | Repository pattern in Node | Abstracting database access, swappable implementations, testing with mocks |
| 118 | Clean architecture in Node | Layers: controller → use case → domain → infrastructure, dependency rule |
| 119 | gRPC with Node.js | Protocol Buffers, @grpc/grpc-js, unary and streaming RPCs, when to use gRPC vs REST |
| 120 | GraphQL with Node.js | Apollo Server, type definitions, resolvers, DataLoader for N+1, subscriptions |

---

## Full topic count: 120 topics across 15 phases

---

## Learning Sequence — Recommended Order

```
Phase 1 (01–07)   → Core runtime understanding         [WEEK 1–2]
Phase 2 (08–13)   → Module systems                     [WEEK 2–3]
Phase 3 (14–33)   → Built-in modules                   [WEEK 3–6]
Phase 4 (34–39)   → Networking and protocols            [WEEK 6–7]
Phase 5 (40–44)   → File system patterns                [WEEK 7–8]
Phase 6 (45–51)   → Async patterns                      [WEEK 8–9]
Phase 7 (52–60)   → Building HTTP servers               [WEEK 9–12]
Phase 8 (61–66)   → Error handling and reliability      [WEEK 12–13]
Phase 9 (67–73)   → Performance and scaling             [WEEK 13–15]
Phase 10 (74–81)  → Security                            [WEEK 15–17]
Phase 11 (82–89)  → Testing                             [WEEK 17–19]
Phase 12 (90–98)  → Databases in Node                   [WEEK 19–22]
Phase 13 (99–104) → Real-time and messaging             [WEEK 22–24]
Phase 14 (105–112)→ Deployment and DevOps               [WEEK 24–27]
Phase 15 (113–120)→ Advanced patterns                   [WEEK 27–30]
```

---

## Progress Tracker

| Phase | Topics | Status |
|---|---|---|
| Phase 1 — How Node.js works | 01–07 | ⬜ Not started |
| Phase 2 — Module systems | 08–13 | ⬜ Not started |
| Phase 3 — Core built-ins | 14–33 | ⬜ Not started |
| Phase 4 — Networking | 34–39 | ⬜ Not started |
| Phase 5 — File system patterns | 40–44 | ⬜ Not started |
| Phase 6 — Async patterns | 45–51 | ⬜ Not started |
| Phase 7 — HTTP servers | 52–60 | ⬜ Not started |
| Phase 8 — Error handling | 61–66 | ⬜ Not started |
| Phase 9 — Performance | 67–73 | ⬜ Not started |
| Phase 10 — Security | 74–81 | ⬜ Not started |
| Phase 11 — Testing | 82–89 | ⬜ Not started |
| Phase 12 — Databases | 90–98 | ⬜ Not started |
| Phase 13 — Real-time | 99–104 | ⬜ Not started |
| Phase 14 — Deployment | 105–112 | ⬜ Not started |
| Phase 15 — Advanced patterns | 113–120 | ⬜ Not started |

---

## What was added beyond your original list

Your original curriculum had 27 topics. This plan has **120 topics**.  
Here is what was added and why:

| Addition | Why it matters |
|---|---|
| Topic 07 — Errors in Node | Error handling is a first-class citizen in Node. Knowing `uncaughtException` and `unhandledRejection` is critical before writing any server code. |
| Topic 13 — Module resolution algorithm | Knowing *exactly* how Node finds files prevents hours of debugging. |
| Topics 34–39 — Networking and protocols | You cannot build backends without understanding TCP, HTTP semantics, WebSockets. |
| Topics 40–44 — File system patterns | File uploads, CSV, watching — common backend tasks not covered in core module docs. |
| Topics 45–51 — Async patterns | You know async/await from JS. These go deeper: concurrency control, retry, abort. |
| Topics 52–60 — HTTP servers | Raw http → Express → Fastify → project structure. The real backend layer. |
| Topics 61–66 — Error handling and reliability | AppError class, centralized handler, graceful shutdown — production essentials. |
| Topics 67–73 — Performance | Profiling, memory leaks, caching, rate limiting, PM2. You need these before job interviews. |
| Topics 74–81 — Security | OWASP Top 10 applied to Node. Non-negotiable. |
| Topics 82–89 — Testing | Jest, supertest, mocking, TDD. Untested code is not production code. |
| Topics 90–98 — Databases in Node | Bridge to your Databases phase. Connection pooling, ORMs, migrations. |
| Topics 99–104 — Real-time and messaging | BullMQ, Socket.IO, Redis pub/sub — used in almost every real product. |
| Topics 105–112 — Deployment | Docker, Nginx, CI/CD, PM2, monitoring. Hire-ready skills. |
| Topics 113–120 — Advanced patterns | Clean architecture, DI, gRPC, GraphQL, plugin systems. Architect-level. |

---

*Plan created: 2026-04-21*  
*Start with: Type **NEXT** to begin Topic 01 — What is Node.js*
