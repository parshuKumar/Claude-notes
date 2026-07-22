# 54 — Typed middleware

## What is this?

**Middleware** is a function that sits between the incoming HTTP request and your route handler. It receives the request, may inspect or mutate it, and then either ends the response or calls `next()` to pass control down the chain.

In Express the shape is fixed:

```ts
import type { Request, Response, NextFunction } from "express";

function requestLogger(req: Request, res: Response, next: NextFunction): void {
  console.log(`${req.method} ${req.originalUrl}`);
  next();
}
```

**Typed middleware** means: instead of hand-writing `(req: Request, res: Response, next: NextFunction)` on every function, you use the types Express already ships — `RequestHandler` and `ErrorRequestHandler` — and you make them **generic** so the middleware can declare exactly what it reads from `req.params`, what it writes to `res.locals`, and what shape `req.body` has *after* it runs.

```ts
import type { RequestHandler, ErrorRequestHandler } from "express";

// One type annotation, all three parameters inferred:
const requestLogger: RequestHandler = (req, res, next) => {
  console.log(`${req.method} ${req.originalUrl}`);
  next();
};

// Error middleware — 4 parameters, a completely different type:
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  res.status(500).json({ error: "Internal Server Error" });
};
```

The interesting part is everything that hangs off those two types: generic parameters for `params`/`body`/`query`, middleware **factories** that return typed handlers, validation middleware that **narrows** `req.body` from `unknown` to `CreateUserBody`, and an `asyncHandler` wrapper that makes `async` middleware safe without losing types.

---

## Why does it matter?

Middleware is where a backend's cross-cutting concerns live: authentication, validation, rate limiting, request IDs, tenant resolution, logging, error formatting. It's also where the type system usually gets abandoned:

- `req.body` is `any` by default in older typings and `unknown`-ish in practice — every handler re-validates or, worse, doesn't
- Auth middleware attaches `req.user`, but nothing tells the route handler that `req.user` exists
- Validation middleware runs `UserSchema.parse(req.body)` but the handler still sees the untyped body
- Error middleware silently stops being error middleware if you delete an unused parameter — Express detects error handlers **by function arity**, and TypeScript will happily let you drop `next`
- `async` middleware that throws never reaches your error handler, because Express (v4) doesn't await promises

Every one of those is a type-system problem with a type-system fix. Getting middleware typing right means the compiler enforces the contract between the middleware chain and the handlers that depend on it.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: everything is a guess ────────────────────────────────────────

// Auth middleware attaches something to req...
function requireAuth(req, res, next) {
  const authToken = req.headers.authorization?.replace("Bearer ", "");
  if (!authToken) return res.status(401).json({ error: "Unauthorized" });
  req.user = verifyToken(authToken);   // ← what shape? nobody knows
  next();
}

// ...and the handler downstream has to trust it blindly:
app.get("/me", requireAuth, (req, res) => {
  res.json({ id: req.user.id, email: req.user.emailAddress });
  //                    ^^          ^^^^^^^^^^^^
  // Is it `id` or `userId`? Is it `emailAddress` or `email`?
  // Wrong guess = TypeError at runtime, in production, on the happy path.
});

// Validation middleware "validates" but hands nothing back:
function validateCreateUser(req, res, next) {
  if (!req.body.email) return res.status(422).json({ error: "email required" });
  if (!req.body.password) return res.status(422).json({ error: "password required" });
  next();
}

app.post("/users", validateCreateUser, (req, res) => {
  createUser(req.body.email, req.body.passwrod);  // typo — silently undefined
  //                                  ^^^^^^^^ no error, no warning, ships to prod
});

// Error middleware — one deleted parameter and it stops being error middleware:
function errorHandler(err, req, res, next) { /* 4 args → error handler */ }
function errorHandler(err, req, res)       { /* 3 args → NORMAL middleware! */ }
// Express literally counts fn.length. Nothing warns you. Errors now 404.

// Async middleware that rejects — Express 4 never sees it:
app.get("/users/:userId", async (req, res) => {
  const user = await db.findUser(req.params.userId);  // throws → unhandled rejection
  res.json(user);                                     // request hangs forever
});
```

```ts
// ── TypeScript: the chain is a contract the compiler checks ──────────────────

import type { RequestHandler, ErrorRequestHandler } from "express";

// 1. Declare what auth attaches, once, globally:
declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;   // now every handler knows the shape
    }
  }
}

interface AuthenticatedUser {
  userId: number;
  email:  string;
  roles:  readonly ("admin" | "editor" | "viewer")[];
}

const requireAuth: RequestHandler = (req, res, next) => {
  const authToken = req.headers.authorization?.replace("Bearer ", "");
  if (!authToken) { res.status(401).json({ error: "Unauthorized" }); return; }
  req.user = verifyToken(authToken);   // ✅ must match AuthenticatedUser
  next();
};

// 2. A handler that REQUIRES auth declares it in its own type:
type AuthedHandler = RequestHandler & { /* marker only, see Concept 6 */ };

app.get("/me", requireAuth, (req, res) => {
  res.json({ id: req.user!.userId, email: req.user!.email });
  //                     ✅ compiler knows userId/email exist — typos are errors
});

// 3. Validation middleware NARROWS req.body for downstream handlers:
interface CreateUserBody { email: string; password: string; displayName: string }

app.post(
  "/users",
  validateBody(CreateUserSchema),                    // ← runtime check
  (req: Request<{}, ApiResponse<User>, CreateUserBody>, res) => {
    createUser(req.body.email, req.body.passwrod);
    //                             ^^^^^^^^ ❌ Property 'passwrod' does not exist
  },
);

// 4. Error middleware is typed — 4 params enforced by ErrorRequestHandler:
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  //                                       ^^^ ❌ deleting a param is a type error
  res.status(500).json({ error: "Internal Server Error" });
};

// 5. Async handlers wrapped so rejections reach the error handler:
app.get("/users/:userId", asyncHandler(async (req, res) => {
  const user = await db.findUser(req.params.userId);
  res.json(user);
}));  // ✅ any rejection → next(err) → errorHandler
```

The revelation: middleware is not "loose glue code you can't type". It's the *most* valuable place to type, because a middleware's output is an invisible input to every handler behind it.

---

## Syntax

```ts
import type {
  Request,             // the request object
  Response,            // the response object
  NextFunction,        // (err?: any) => void
  RequestHandler,      // (req, res, next) => void  — normal middleware
  ErrorRequestHandler, // (err, req, res, next) => void — error middleware
} from "express";

// ── Hand-written signature (verbose, but explicit) ───────────────────────────
function requestLogger(req: Request, res: Response, next: NextFunction): void {
  next();                                        // pass control onward
}

// ── RequestHandler (preferred — parameters are inferred) ─────────────────────
const requestLogger2: RequestHandler = (req, res, next) => {
  next();                                        // req/res/next all typed for free
};

// ── RequestHandler's generic parameters, in order ────────────────────────────
// RequestHandler<Params, ResBody, ReqBody, ReqQuery, Locals>
const createUser: RequestHandler<
  { orgId: string },          // req.params
  ApiResponse<User>,          // res.json(...) argument type
  CreateUserBody,             // req.body
  { notify?: "true" | "false" }, // req.query
  { requestId: string }       // res.locals
> = (req, res, next) => {
  req.params.orgId;           // string
  req.body.email;             // string
  req.query.notify;           // "true" | "false" | undefined
  res.locals.requestId;       // string
  res.json({ ok: true, data: newUser });  // must match ApiResponse<User>
};

// ── Error middleware — MUST have 4 parameters ────────────────────────────────
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) { next(err); return; }    // delegate to Express default
  res.status(500).json({ error: "Internal Server Error" });
};

// ── Middleware factory — a function that RETURNS a RequestHandler ────────────
function requireRole(role: "admin" | "editor"): RequestHandler {
  return (req, res, next) => {                   // closure captures `role`
    if (!req.user?.roles.includes(role)) { res.status(403).end(); return; }
    next();
  };
}

// ── Registration ─────────────────────────────────────────────────────────────
app.use(requestLogger2);                         // global middleware
app.post("/orgs/:orgId/users", requireRole("admin"), createUser); // per-route
app.use(errorHandler);                           // error middleware goes LAST
```

---

## How it works — concept by concept

### Concept 1 — `RequestHandler` vs hand-written signatures

You can type middleware two ways. They are *not* equivalent in ergonomics.

```ts
import type { Request, Response, NextFunction, RequestHandler } from "express";

// ── Way A — annotate each parameter (contextual typing does NOT apply) ───────
function requestIdMiddleware(req: Request, res: Response, next: NextFunction): void {
  const requestId = req.header("x-request-id") ?? crypto.randomUUID();
  res.locals.requestId = requestId;   // res.locals is Record<string, any> by default
  res.setHeader("x-request-id", requestId);
  next();
}

// ── Way B — annotate the VARIABLE, parameters flow from the type ─────────────
const requestIdMiddleware2: RequestHandler = (req, res, next) => {
  const requestId = req.header("x-request-id") ?? crypto.randomUUID();
  res.locals.requestId = requestId;
  res.setHeader("x-request-id", requestId);
  next();
};
```

Why B is better:

```ts
// 1. Return type is enforced. RequestHandler returns void — accidentally
//    returning res.status(...) is caught if you enable it:
const bad: RequestHandler = (req, res, next) => {
  return res.status(401).json({ error: "Unauthorized" });
  // ❌ Type 'Response' is not assignable to type 'void'
  //    (only under some @types/express versions — see Going deeper)
};

const good: RequestHandler = (req, res, next) => {
  res.status(401).json({ error: "Unauthorized" });
  return;                                   // ✅ explicit early return, void
};

// 2. Generic parameters are available in one place:
const typed: RequestHandler<{ userId: string }> = (req, res, next) => {
  req.params.userId;                        // ✅ string, not `any`
  next();
};

// 3. It composes — you can build aliases:
type ApiHandler<TBody = unknown, TParams = Record<string, string>> =
  RequestHandler<TParams, ApiResponse<unknown>, TBody>;

const deleteUser: ApiHandler<never, { userId: string }> = (req, res) => {
  const userId = Number(req.params.userId);
  res.json({ ok: true, data: null });
};
```

Here is the actual definition from `@types/express-serve-static-core`, simplified:

```ts
// The five generic slots, with their defaults:
interface RequestHandler<
  P     = ParamsDictionary,          // req.params
  ResBody = any,                     // res.json / res.send argument
  ReqBody = any,                     // req.body
  ReqQuery = QueryString.ParsedQs,   // req.query
  LocalsObj extends Record<string, any> = Record<string, any>, // res.locals
> {
  (
    req:  Request<P, ResBody, ReqBody, ReqQuery, LocalsObj>,
    res:  Response<ResBody, LocalsObj>,
    next: NextFunction,
  ): void;
}
```

Two things to note:
- `ReqBody` defaults to `any`. That is why `req.body.anything` compiles by default. Fixing this is Concept 4's job.
- `P` (params) defaults to `ParamsDictionary = { [key: string]: string }`. Params are **always strings** — `req.params.userId` is `string`, never `number`, even for `/users/:userId(\\d+)`.

### Concept 2 — `NextFunction` and typing `next(err)`

`NextFunction` looks trivial but has two distinct call shapes:

```ts
// Simplified definition:
interface NextFunction {
  (err?: any): void;                    // pass an error, or nothing
  (deferToNext: "router"): void;        // skip the rest of THIS router
  (deferToNext: "route"): void;         // skip the rest of THIS route's handlers
}
```

```ts
const authMiddleware: RequestHandler = (req, res, next) => {
  const authToken = req.header("authorization");

  if (!authToken) {
    // ── Option 1 — respond directly and stop the chain ──────────────────────
    res.status(401).json({ error: "Missing authorization header" });
    return;                              // ⚠️ MUST return — no implicit stop
  }

  try {
    req.user = verifyToken(authToken);
    next();                              // ── Option 2 — continue the chain ──
  } catch (err) {
    next(err);                           // ── Option 3 — jump to error handler ─
  }
};

// `next("route")` — skip remaining handlers for this route, try the next route:
const skipIfCached: RequestHandler = (req, res, next) => {
  if (cache.has(req.originalUrl)) {
    res.json(cache.get(req.originalUrl));
    return;
  }
  next("route");                         // ✅ typed — literal "route" overload
};
```

The `err?: any` typing is a real weakness: `next(err)` accepts anything, so `next("oops")` and `next(404)` compile. If you want discipline, narrow it yourself:

```ts
// A strict `next` you use inside your own middleware helpers:
type StrictNext = (err?: Error) => void;

function typedNext(next: NextFunction): StrictNext {
  return (err) => next(err);             // narrows the input, delegates onward
}

const strictMiddleware: RequestHandler = (req, res, next) => {
  const safeNext = typedNext(next);
  safeNext(new ApiError(404, "Not found"));   // ✅
  // safeNext("Not found");                   // ❌ string is not assignable to Error
};
```

### Concept 3 — Error middleware, the 4-argument rule, and arity detection

Express decides whether a function is *error* middleware or *normal* middleware by reading `fn.length` — the number of declared parameters — at registration time.

```ts
// Inside Express's Layer constructor (conceptually):
//   this.length = fn.length;
// Inside Layer#handle_error:
//   if (fn.length !== 4) return next(error);   // not an error handler, skip it
```

Consequences:

```ts
// ✅ 4 parameters → error handler. Runs only when next(err) was called.
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  res.status(500).json({ error: "Internal Server Error" });
};

// ❌ 3 parameters → Express treats this as NORMAL middleware. Errors bypass it
//    entirely and fall through to Express's default HTML error page.
const brokenErrorHandler = (err: unknown, req: Request, res: Response) => {
  res.status(500).json({ error: "Internal Server Error" });
};

// ❌ Rest parameters → fn.length is 0. Also not an error handler.
const alsoBroken = (...args: [unknown, Request, Response, NextFunction]) => {};

// ❌ Default parameters truncate fn.length. `(err, req, res, next = noop)`
//    has fn.length === 3. Also not an error handler.
const subtlyBroken = (err: unknown, req: Request, res: Response, next: NextFunction = () => {}) => {};
```

This is exactly where `ErrorRequestHandler` earns its keep. Because it is a **call signature with four required parameters**, TypeScript enforces arity in the direction that matters:

```ts
import type { ErrorRequestHandler } from "express";

// TypeScript permits FEWER parameters (functions are bivariant on arity),
// so this still compiles — the type alone does not save you:
const stillCompiles: ErrorRequestHandler = (err, req, res) => {
  res.status(500).end();
};
// ⚠️ Compiles, but fn.length === 3 → Express will NOT call it on errors.
```

So the real rule: **`ErrorRequestHandler` documents intent; you still must physically write four parameters.** Enforce it with a lint rule or a wrapper that guarantees arity:

```ts
// A wrapper that always produces a 4-arity function no matter what you pass:
function makeErrorHandler(
  fn: (err: unknown, req: Request, res: Response, next: NextFunction) => void,
): ErrorRequestHandler {
  // Explicitly declaring 4 params here guarantees fn.length === 4:
  return (err, req, res, next) => fn(err, req, res, next);
}

const safeErrorHandler = makeErrorHandler((err, req, res) => {
  // You may ignore `next` here — the WRAPPER keeps the arity Express needs.
  res.status(500).json({ error: "Internal Server Error" });
});

app.use(safeErrorHandler);              // ✅ arity 4, guaranteed
```

Also note `err` is typed `any` in `ErrorRequestHandler`. Treat it as `unknown` immediately:

```ts
const errorHandler2: ErrorRequestHandler = (err, req, res, next) => {
  const error: unknown = err;           // ← re-widen to unknown, then narrow

  if (error instanceof ApiError) {
    res.status(error.statusCode).json({ error: error.message, code: error.code });
    return;
  }
  if (error instanceof Error) {
    req.log?.error({ err: error }, "Unhandled error");
    res.status(500).json({ error: "Internal Server Error" });
    return;
  }
  // Someone called next("string") or next(42):
  req.log?.error({ err: String(error) }, "Non-Error thrown");
  res.status(500).json({ error: "Internal Server Error" });
};
```

### Concept 4 — Typed validation middleware that narrows `req.body`

The default `req.body` is `any`. The goal: after validation middleware runs, downstream handlers see a **specific** body type — and TypeScript enforces it.

```ts
// ── The schema abstraction (works with zod, yup, or hand-rolled) ─────────────
interface Validator<T> {
  parse(input: unknown): T;             // throws on invalid input
}

// ── The middleware factory ──────────────────────────────────────────────────
function validateBody<TBody>(
  schema: Validator<TBody>,
): RequestHandler<Record<string, string>, unknown, TBody> {
  return (req, res, next) => {
    try {
      // Overwrite req.body with the PARSED value — coerced, defaulted, stripped:
      req.body = schema.parse(req.body);
      next();
    } catch (err) {
      next(new ApiError(422, "Validation failed", { cause: err }));
    }
  };
}
```

Now the key move — the *handler* declares the same body type:

```ts
interface CreateUserBody {
  email:       string;
  password:    string;
  displayName: string;
}

const CreateUserSchema: Validator<CreateUserBody> = { /* zod or manual */ } as never;

// The handler's RequestHandler generics say "my body is CreateUserBody":
const createUserHandler: RequestHandler<
  Record<string, string>,
  ApiResponse<PublicUser>,
  CreateUserBody
> = async (req, res, next) => {
  req.body.email;        // ✅ string
  req.body.password;     // ✅ string
  // req.body.passwrod;  // ❌ Property 'passwrod' does not exist on type 'CreateUserBody'

  const user = await userService.create(req.body);
  res.status(201).json({ ok: true, data: toPublicUser(user) });
};

app.post("/users", validateBody(CreateUserSchema), createUserHandler);
```

There is one honest caveat: Express does not *link* the middleware's `TBody` to the handler's `TBody`. Both must name the same type. You can close that gap with a helper that registers both at once:

```ts
// A route builder that ties the schema and the handler together:
function postWithBody<TBody, TRes>(
  app: Express,
  path: string,
  schema: Validator<TBody>,
  handler: RequestHandler<Record<string, string>, TRes, TBody>,
): void {
  app.post(path, validateBody(schema), handler as RequestHandler);
}

// Now the body type is inferred from the SCHEMA — impossible to disagree:
postWithBody(app, "/users", CreateUserSchema, async (req, res) => {
  req.body.displayName;      // ✅ inferred as string from CreateUserSchema
  res.status(201).json({ ok: true, data: await userService.create(req.body) });
});
```

### Concept 5 — Generic middleware and middleware factories

A **middleware factory** is a function returning a `RequestHandler`. It is the standard way to parameterise middleware, and making it generic is what turns it from "configurable" into "type-propagating".

```ts
// ── Non-generic factory — configuration only ────────────────────────────────
function rateLimit(options: { windowMs: number; max: number }): RequestHandler {
  const hits = new Map<string, { count: number; resetAt: number }>();

  return (req, res, next) => {
    const key = req.ip ?? "unknown";
    const now = Date.now();
    const entry = hits.get(key);

    if (!entry || entry.resetAt < now) {
      hits.set(key, { count: 1, resetAt: now + options.windowMs });
      next();
      return;
    }
    if (entry.count >= options.max) {
      res.status(429).json({ error: "Too Many Requests" });
      return;
    }
    entry.count += 1;
    next();
  };
}

app.use("/api", rateLimit({ windowMs: 60_000, max: 100 }));
```

```ts
// ── Generic factory — the type parameter flows into the handler ─────────────

// Validate and coerce route params into a typed object on res.locals:
function parseParams<TParams extends Record<string, unknown>>(
  parse: (raw: Record<string, string>) => TParams,
): RequestHandler<Record<string, string>, unknown, unknown, unknown, { params: TParams }> {
  return (req, res, next) => {
    try {
      res.locals.params = parse(req.params);   // ✅ typed as TParams
      next();
    } catch (err) {
      next(new ApiError(400, "Invalid route parameters", { cause: err }));
    }
  };
}

// Usage — TParams is inferred as { userId: number; includeOrders: boolean }:
const parseUserParams = parseParams((raw) => ({
  userId:        Number(raw.userId),
  includeOrders: raw.includeOrders === "true",
}));

const getUserHandler: RequestHandler<
  Record<string, string>,
  ApiResponse<PublicUser>,
  unknown,
  unknown,
  { params: { userId: number; includeOrders: boolean } }
> = async (req, res) => {
  const { userId, includeOrders } = res.locals.params;   // ✅ fully typed
  const user = await userService.findById(userId, { includeOrders });
  res.json({ ok: true, data: toPublicUser(user) });
};

app.get("/users/:userId", parseUserParams, getUserHandler);
```

Generic factories also let you build **role-parameterised** middleware where the role list is checked at compile time:

```ts
type Role = "admin" | "editor" | "viewer" | "billing";

// The rest parameter is a tuple of Role — misspellings are compile errors:
function requireAnyRole<const TRoles extends readonly Role[]>(
  ...roles: TRoles
): RequestHandler {
  const allowed = new Set<Role>(roles);
  return (req, res, next) => {
    const user = req.user;
    if (!user) { res.status(401).json({ error: "Unauthorized" }); return; }
    if (!user.roles.some((r) => allowed.has(r))) {
      res.status(403).json({ error: "Forbidden", required: roles });
      return;
    }
    next();
  };
}

app.delete("/users/:userId", requireAnyRole("admin", "billing"), deleteUserHandler);
// app.delete("/x", requireAnyRole("admn"));  // ❌ "admn" is not assignable to Role
```

### Concept 6 — Augmenting `Express.Request` so middleware output is visible

Middleware that attaches properties (`req.user`, `req.requestId`, `req.tenant`) must declare them, or downstream handlers can't see them.

```ts
// src/types/express.d.ts — a global declaration file (no top-level import/export
// in the `declare global` version, or use `export {}` to make it a module)

import type { Logger } from "pino";

declare global {
  namespace Express {
    interface Request {
      user?:      AuthenticatedUser;    // set by requireAuth
      requestId:  string;               // set by requestIdMiddleware (always present)
      log:        Logger;               // set by loggingMiddleware
      tenant?:    Tenant;               // set by tenantResolver
    }
  }
}

export {};   // makes this file a module so the import above is legal
```

Now the trade-off: `user?: AuthenticatedUser` means every handler must deal with `undefined`, even routes behind `requireAuth`. Three ways to handle it, worst to best:

```ts
// ── Option A — non-null assertion (fast, unsafe, silently wrong if you forget
//    to mount requireAuth on that route) ─────────────────────────────────────
app.get("/me", requireAuth, (req, res) => {
  res.json({ userId: req.user!.userId });
});

// ── Option B — a runtime assertion helper (safe, but repeated everywhere) ────
function assertAuthenticated(req: Request): asserts req is Request & { user: AuthenticatedUser } {
  if (!req.user) throw new ApiError(401, "Unauthorized");
}

app.get("/me", requireAuth, (req, res) => {
  assertAuthenticated(req);
  res.json({ userId: req.user.userId });     // ✅ narrowed, no `!`
});

// ── Option C — a typed wrapper that BAKES the guarantee into the signature ──
type AuthedRequest<
  P = Record<string, string>,
  ResBody = unknown,
  ReqBody = unknown,
> = Request<P, ResBody, ReqBody> & { user: AuthenticatedUser };

type AuthedRequestHandler<P = Record<string, string>, ResBody = unknown, ReqBody = unknown> =
  (req: AuthedRequest<P, ResBody, ReqBody>, res: Response<ResBody>, next: NextFunction) => void | Promise<void>;

// The wrapper performs the check AND supplies the narrowed type:
function authed<P, ResBody, ReqBody>(
  handler: AuthedRequestHandler<P, ResBody, ReqBody>,
): RequestHandler<P, ResBody, ReqBody> {
  return (req, res, next) => {
    if (!req.user) { res.status(401).json({ error: "Unauthorized" } as ResBody); return; }
    void handler(req as AuthedRequest<P, ResBody, ReqBody>, res, next);
  };
}

app.get("/me", requireAuth, authed((req, res) => {
  res.json({ userId: req.user.userId });     // ✅ no `!`, no assertion, no repetition
}));
```

### Concept 7 — `asyncHandler` — typing the async-to-next bridge

Express 4 does not await handlers. An `async` handler that rejects produces an **unhandled promise rejection** and a request that hangs until the client times out. The fix is a wrapper — and typing it correctly is the tricky part.

```ts
// ── Naive version — loses all type information ──────────────────────────────
function asyncHandlerBad(fn: any): RequestHandler {
  return (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
}
// req/res inside fn are `any`. Everything downstream is untyped.

// ── Correct version — generics mirror RequestHandler's generics ─────────────
import type { RequestHandler, Request, Response, NextFunction } from "express";
import type { ParamsDictionary, Query } from "express-serve-static-core";

type AsyncRequestHandler<
  P        = ParamsDictionary,
  ResBody  = unknown,
  ReqBody  = unknown,
  ReqQuery = Query,
  Locals extends Record<string, unknown> = Record<string, unknown>,
> = (
  req:  Request<P, ResBody, ReqBody, ReqQuery, Locals>,
  res:  Response<ResBody, Locals>,
  next: NextFunction,
) => Promise<void>;

function asyncHandler<
  P, ResBody, ReqBody, ReqQuery, Locals extends Record<string, unknown>,
>(
  fn: AsyncRequestHandler<P, ResBody, ReqBody, ReqQuery, Locals>,
): RequestHandler<P, ResBody, ReqBody, ReqQuery, Locals> {
  return (req, res, next) => {
    // Promise.resolve() also handles a sync throw inside fn:
    Promise.resolve(fn(req, res, next)).catch(next);
    //                                        ^^^^ next(err) → error middleware
  };
}
```

Why the generics must be *declared on the function*, not just on the parameter type: that is what lets TypeScript **infer** them from the call site.

```ts
// The generics are inferred from the annotated callback:
app.get(
  "/users/:userId",
  asyncHandler<{ userId: string }, ApiResponse<PublicUser>, never, Query, Record<string, unknown>>(
    async (req, res) => {
      const user = await userService.findById(Number(req.params.userId));
      res.json({ ok: true, data: toPublicUser(user) });
    },
  ),
);

// In practice you rarely spell them out — you let a route-builder do it (Example 2).
```

One subtlety worth internalising: `.catch(next)` passes the rejection value to `next`, which is typed `(err?: any) => void`. If your promise rejects with a non-Error (`throw "nope"`), your error middleware receives a string. Guard against that inside `asyncHandler` if you want a strong invariant:

```ts
function asyncHandlerStrict<P, ResBody, ReqBody>(
  fn: AsyncRequestHandler<P, ResBody, ReqBody>,
): RequestHandler<P, ResBody, ReqBody> {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch((err: unknown) => {
      // Normalise: the error middleware can now assume `Error`:
      next(err instanceof Error ? err : new Error(String(err), { cause: err }));
    });
  };
}
```

> Express **5** awaits handlers and forwards rejections automatically, so `asyncHandler` becomes optional there. The typed wrapper is still useful for normalising thrown non-Errors.

---

## Example 1 — basic

```ts
// A small, fully typed middleware stack: request id → logging → auth → route.

import express, {
  type Request,
  type Response,
  type NextFunction,
  type RequestHandler,
  type ErrorRequestHandler,
} from "express";
import { randomUUID } from "node:crypto";

// ── Domain types ────────────────────────────────────────────────────────────

interface AuthenticatedUser {
  userId: number;
  email:  string;
  roles:  readonly ("admin" | "editor" | "viewer")[];
}

type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; code: string };

// ── Request augmentation — declare what our middleware attaches ─────────────

declare global {
  namespace Express {
    interface Request {
      requestId: string;              // always set by requestIdMiddleware
      user?:     AuthenticatedUser;   // set only when requireAuth ran
    }
  }
}

// ── 1. Request ID middleware ────────────────────────────────────────────────

const requestIdMiddleware: RequestHandler = (req, res, next) => {
  // Reuse an upstream id (load balancer / gateway) or mint a new one:
  const incoming = req.header("x-request-id");
  req.requestId = incoming && incoming.length <= 64 ? incoming : randomUUID();
  res.setHeader("x-request-id", req.requestId);
  next();
};

// ── 2. Logging middleware — measures duration on response finish ────────────

const requestLogger: RequestHandler = (req, res, next) => {
  const startedAt = process.hrtime.bigint();

  res.on("finish", () => {
    const durationMs = Number(process.hrtime.bigint() - startedAt) / 1_000_000;
    console.log(
      JSON.stringify({
        requestId:  req.requestId,
        method:     req.method,
        path:       req.originalUrl,
        statusCode: res.statusCode,
        durationMs: Math.round(durationMs * 100) / 100,
        userId:     req.user?.userId ?? null,
      }),
    );
  });

  next();
};

// ── 3. Auth middleware ──────────────────────────────────────────────────────

function verifyToken(authToken: string): AuthenticatedUser {
  // Stand-in for jwt.verify — throws on invalid/expired tokens:
  const payload = JSON.parse(Buffer.from(authToken.split(".")[1]!, "base64url").toString()) as {
    sub: number; email: string; roles: AuthenticatedUser["roles"];
  };
  return { userId: payload.sub, email: payload.email, roles: payload.roles };
}

const requireAuth: RequestHandler = (req, res, next) => {
  const header = req.header("authorization");

  if (!header?.startsWith("Bearer ")) {
    res.status(401).json({ ok: false, error: "Missing bearer token", code: "UNAUTHENTICATED" });
    return;                                     // ⚠️ must return — next() is NOT called
  }

  try {
    req.user = verifyToken(header.slice("Bearer ".length));
    next();
  } catch (err) {
    next(new ApiError(401, "Invalid or expired token", "TOKEN_INVALID", { cause: err }));
  }
};

// ── 4. A typed application error ────────────────────────────────────────────

class ApiError extends Error {
  constructor(
    readonly statusCode: number,
    message: string,
    readonly code: string,
    options?: ErrorOptions,
  ) {
    super(message, options);
    this.name = "ApiError";
  }
}

// ── 5. Error middleware — FOUR parameters, always ───────────────────────────

const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  //                                                    ^^^^ unused, but required
  if (res.headersSent) { next(err); return; }   // Express default closes the socket

  const error: unknown = err;

  if (error instanceof ApiError) {
    res.status(error.statusCode).json({ ok: false, error: error.message, code: error.code });
    return;
  }

  console.error(JSON.stringify({ requestId: req.requestId, err: String(error) }));
  res.status(500).json({ ok: false, error: "Internal Server Error", code: "INTERNAL" });
};

// ── 6. Wiring — order matters, error handler is LAST ────────────────────────

const app = express();

app.use(express.json({ limit: "1mb" }));
app.use(requestIdMiddleware);
app.use(requestLogger);

app.get("/health", (req, res) => {
  res.json({ ok: true, data: { status: "up", requestId: req.requestId } });
});

app.get("/me", requireAuth, (req, res) => {
  // req.user is `AuthenticatedUser | undefined` — narrow before use:
  if (!req.user) { res.status(401).end(); return; }
  res.json({ ok: true, data: { userId: req.user.userId, email: req.user.email } });
});

app.use(errorHandler);                          // ← must come after all routes

app.listen(3000);
```

---

## Example 2 — real world backend use case

```ts
// A production middleware toolkit: schema validation that narrows req.body,
// role-based authorisation, async wrapping, and a typed route builder that
// ties the whole chain together so the compiler checks it end to end.

import express, {
  type Express,
  type Request,
  type Response,
  type NextFunction,
  type RequestHandler,
  type ErrorRequestHandler,
} from "express";
import type { ParamsDictionary, Query } from "express-serve-static-core";

// ═══════════════════════════════════════════════════════════════════════════
// Shared types
// ═══════════════════════════════════════════════════════════════════════════

type Role = "admin" | "editor" | "viewer" | "billing";

interface AuthenticatedUser {
  userId: number;
  email:  string;
  orgId:  number;
  roles:  readonly Role[];
}

type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; code: string; details?: unknown };

declare global {
  namespace Express {
    interface Request {
      requestId: string;
      user?:     AuthenticatedUser;
    }
  }
}

// A minimal validator interface — zod's ZodType satisfies it structurally:
interface Validator<T> {
  parse(input: unknown): T;
}

class ApiError extends Error {
  constructor(
    readonly statusCode: number,
    message: string,
    readonly code: string,
    readonly details?: unknown,
  ) {
    super(message);
    this.name = "ApiError";
  }

  static unauthorized(message = "Unauthorized"): ApiError {
    return new ApiError(401, message, "UNAUTHENTICATED");
  }
  static forbidden(message = "Forbidden"): ApiError {
    return new ApiError(403, message, "FORBIDDEN");
  }
  static notFound(resource: string): ApiError {
    return new ApiError(404, `${resource} not found`, "NOT_FOUND");
  }
  static validation(details: unknown): ApiError {
    return new ApiError(422, "Validation failed", "VALIDATION_ERROR", details);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 1. asyncHandler — bridges async functions to Express's callback error flow
// ═══════════════════════════════════════════════════════════════════════════

type AsyncRequestHandler<
  P        = ParamsDictionary,
  ResBody  = unknown,
  ReqBody  = unknown,
  ReqQuery = Query,
> = (
  req:  Request<P, ResBody, ReqBody, ReqQuery>,
  res:  Response<ResBody>,
  next: NextFunction,
) => Promise<void>;

function asyncHandler<P, ResBody, ReqBody, ReqQuery>(
  fn: AsyncRequestHandler<P, ResBody, ReqBody, ReqQuery>,
): RequestHandler<P, ResBody, ReqBody, ReqQuery> {
  return (req, res, next) => {
    // Promise.resolve captures both rejections AND synchronous throws:
    Promise.resolve(fn(req, res, next)).catch((err: unknown) => {
      // Normalise so the error middleware may assume `Error`:
      next(err instanceof Error ? err : new Error(String(err), { cause: err }));
    });
  };
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. Validation middleware — parses AND replaces req.body / req.query
// ═══════════════════════════════════════════════════════════════════════════

function validateBody<TBody>(
  schema: Validator<TBody>,
): RequestHandler<ParamsDictionary, unknown, TBody> {
  return (req, res, next) => {
    try {
      // Assign the PARSED result: defaults applied, unknown keys stripped,
      // numbers coerced. Downstream handlers see the clean value.
      req.body = schema.parse(req.body);
      next();
    } catch (err) {
      next(ApiError.validation(err));
    }
  };
}

function validateQuery<TQuery extends Query>(
  schema: Validator<TQuery>,
): RequestHandler<ParamsDictionary, unknown, unknown, TQuery> {
  return (req, res, next) => {
    try {
      // req.query is a getter in Express 5 — write through defineProperty:
      Object.defineProperty(req, "query", { value: schema.parse(req.query), writable: true });
      next();
    } catch (err) {
      next(ApiError.validation(err));
    }
  };
}

// ═══════════════════════════════════════════════════════════════════════════
// 3. Auth + authorisation middleware factories
// ═══════════════════════════════════════════════════════════════════════════

const requireAuth: RequestHandler = (req, res, next) => {
  const header = req.header("authorization");
  if (!header?.startsWith("Bearer ")) { next(ApiError.unauthorized("Missing bearer token")); return; }

  try {
    req.user = verifyAuthToken(header.slice(7));
    next();
  } catch {
    next(ApiError.unauthorized("Invalid or expired token"));
  }
};

// `const` type parameter keeps the literal tuple type, so `roles` in the 403
// payload is precise and typos are compile errors:
function requireAnyRole<const TRoles extends readonly Role[]>(...roles: TRoles): RequestHandler {
  const allowed = new Set<Role>(roles);
  return (req, res, next) => {
    if (!req.user) { next(ApiError.unauthorized()); return; }
    if (!req.user.roles.some((role) => allowed.has(role))) {
      next(new ApiError(403, "Insufficient permissions", "FORBIDDEN", { required: roles }));
      return;
    }
    next();
  };
}

// Tenant isolation — ensures the org in the path matches the token's org:
const requireSameOrg: RequestHandler<{ orgId: string }> = (req, res, next) => {
  if (!req.user) { next(ApiError.unauthorized()); return; }
  if (Number(req.params.orgId) !== req.user.orgId) {
    next(ApiError.forbidden("Cross-organisation access denied"));
    return;
  }
  next();
};

// ═══════════════════════════════════════════════════════════════════════════
// 4. The authed-handler wrapper — removes `req.user!` from every handler
// ═══════════════════════════════════════════════════════════════════════════

type AuthedRequest<P, ResBody, ReqBody, ReqQuery> =
  Request<P, ResBody, ReqBody, ReqQuery> & { user: AuthenticatedUser };

type AuthedAsyncHandler<P, ResBody, ReqBody, ReqQuery> = (
  req:  AuthedRequest<P, ResBody, ReqBody, ReqQuery>,
  res:  Response<ResBody>,
  next: NextFunction,
) => Promise<void>;

function authed<P, ResBody, ReqBody, ReqQuery>(
  handler: AuthedAsyncHandler<P, ResBody, ReqBody, ReqQuery>,
): RequestHandler<P, ResBody, ReqBody, ReqQuery> {
  return asyncHandler<P, ResBody, ReqBody, ReqQuery>(async (req, res, next) => {
    if (!req.user) throw ApiError.unauthorized();
    // The runtime check above justifies the assertion below:
    await handler(req as AuthedRequest<P, ResBody, ReqBody, ReqQuery>, res, next);
  });
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. A typed route builder — schema and handler body types cannot disagree
// ═══════════════════════════════════════════════════════════════════════════

interface RouteConfig<TParams, TBody, TQuery extends Query, TData> {
  method:    "get" | "post" | "patch" | "delete";
  path:      string;
  auth:      boolean;
  roles?:    readonly Role[];
  bodySchema?:  Validator<TBody>;
  querySchema?: Validator<TQuery>;
  handler:   AuthedAsyncHandler<TParams, ApiResponse<TData>, TBody, TQuery>;
}

function registerRoute<TParams, TBody, TQuery extends Query, TData>(
  app: Express,
  config: RouteConfig<TParams, TBody, TQuery, TData>,
): void {
  const chain: RequestHandler[] = [];

  if (config.auth)        chain.push(requireAuth);
  if (config.roles)       chain.push(requireAnyRole(...config.roles));
  if (config.bodySchema)  chain.push(validateBody(config.bodySchema) as RequestHandler);
  if (config.querySchema) chain.push(validateQuery(config.querySchema) as RequestHandler);

  chain.push(authed(config.handler) as RequestHandler);
  app[config.method](config.path, ...chain);
}

// ═══════════════════════════════════════════════════════════════════════════
// 6. Using it — the body type is INFERRED from the schema
// ═══════════════════════════════════════════════════════════════════════════

interface CreateUserBody {
  email:       string;
  password:    string;
  displayName: string;
  role:        Role;
}

interface PublicUser {
  userId:      number;
  email:       string;
  displayName: string;
  role:        Role;
  createdAt:   string;
}

declare const CreateUserSchema: Validator<CreateUserBody>;
declare const ListUsersQuerySchema: Validator<{ page: string; perPage: string; search?: string }>;
declare const userService: {
  create(orgId: number, input: CreateUserBody): Promise<PublicUser>;
  list(orgId: number, opts: { page: number; perPage: number; search?: string }): Promise<PublicUser[]>;
};
declare function verifyAuthToken(authToken: string): AuthenticatedUser;

const app = express();
app.use(express.json({ limit: "1mb" }));

registerRoute(app, {
  method: "post",
  path:   "/orgs/:orgId/users",
  auth:   true,
  roles:  ["admin"],
  bodySchema: CreateUserSchema,
  handler: async (req, res) => {
    req.user.orgId;          // ✅ no `!` — `authed` guaranteed it
    req.body.displayName;    // ✅ string — inferred from CreateUserSchema
    // req.body.dispalyName; // ❌ Property 'dispalyName' does not exist

    const user = await userService.create(req.user.orgId, req.body);
    res.status(201).json({ ok: true, data: user });
    //                                  ^^^^ must satisfy ApiResponse<PublicUser>
  },
});

registerRoute(app, {
  method: "get",
  path:   "/orgs/:orgId/users",
  auth:   true,
  roles:  ["admin", "viewer"],
  querySchema: ListUsersQuerySchema,
  handler: async (req, res) => {
    const users = await userService.list(req.user.orgId, {
      page:    Number(req.query.page),      // ✅ typed as string by the schema
      perPage: Number(req.query.perPage),
      search:  req.query.search,
    });
    res.json({ ok: true, data: users });
  },
});

// ═══════════════════════════════════════════════════════════════════════════
// 7. Terminal middleware — 404 then the error handler, in that order
// ═══════════════════════════════════════════════════════════════════════════

const notFoundHandler: RequestHandler = (req, res, next) => {
  next(ApiError.notFound(`Route ${req.method} ${req.originalUrl}`));
};

const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  // 4 parameters — non-negotiable. Express reads fn.length === 4.
  if (res.headersSent) { next(err); return; }

  const error: unknown = err;

  if (error instanceof ApiError) {
    res.status(error.statusCode).json({
      ok: false, error: error.message, code: error.code, details: error.details,
    });
    return;
  }

  console.error(JSON.stringify({
    requestId: req.requestId,
    message:   error instanceof Error ? error.message : String(error),
    stack:     error instanceof Error ? error.stack : undefined,
  }));

  res.status(500).json({ ok: false, error: "Internal Server Error", code: "INTERNAL" });
};

app.use(notFoundHandler);
app.use(errorHandler);
```

---

## Going deeper

### The `void` return type and why `return res.json(...)` sometimes errors

`RequestHandler`'s call signature returns `void`. A function returning something else *is* assignable to a `void`-returning type — that's a deliberate TypeScript rule ("return-type bivariance for void") so `arr.forEach(x => list.push(x))` works.

```ts
const handler: RequestHandler = (req, res) => {
  return res.status(404).json({ error: "Not found" });   // usually allowed
};
```

But it breaks when the handler is `async`, because `Promise<Response>` is not assignable to `void`:

```ts
const handler: RequestHandler = async (req, res) => {    // ❌ in strict setups
  return res.status(404).json({ error: "Not found" });
};
```

House rule that avoids the whole category: **never `return res.…`; write the response, then bare `return`.**

```ts
const handler: RequestHandler = (req, res) => {
  if (!req.user) { res.status(401).json({ error: "Unauthorized" }); return; }
  res.json({ ok: true, data: req.user });
};
```

### `RequestHandler`'s generics do not propagate along the chain

This surprises everyone: mounting `validateBody<CreateUserBody>(schema)` does **not** change the type of `req.body` in the *next* handler. Express types each handler independently.

```ts
app.post(
  "/users",
  validateBody(CreateUserSchema),          // TBody = CreateUserBody here...
  (req, res) => {
    req.body.email;                        // ...but req.body is `any` HERE
  },
);
```

There are only two real fixes:
1. Annotate the downstream handler with the same generics (manual, duplicated).
2. Use a route builder that accepts schema + handler together and infers the link (Example 2's `registerRoute`).

Frameworks like tRPC, Fastify (with type providers), NestJS, and Hono solve this natively. Plain Express requires you to build the bridge yourself.

### Params are always `string` — including numeric-looking ones

```ts
const handler: RequestHandler<{ userId: number }> = (req, res) => {
  req.params.userId;                       // typed as number...
  // ...but at RUNTIME it is the string "42". `typeof req.params.userId === "string"`.
};
```

Typing params as `number` is a **lie the compiler cannot catch**. Always type them as `string` and convert explicitly:

```ts
const handler: RequestHandler<{ userId: string }> = (req, res, next) => {
  const userId = Number.parseInt(req.params.userId, 10);
  if (!Number.isInteger(userId) || userId <= 0) {
    next(new ApiError(400, "userId must be a positive integer", "BAD_PARAM"));
    return;
  }
  // userId: number, validated
};
```

### `res.locals` is `Record<string, any>` by default

Anything you stash on `res.locals` is `any` unless you supply the `Locals` generic (5th slot) or augment `Express.Locals`:

```ts
declare global {
  namespace Express {
    interface Locals {
      requestId: string;
      startedAt: bigint;
    }
  }
}

const m: RequestHandler = (req, res, next) => {
  res.locals.requestId;    // ✅ string
  // res.locals.typo;      // ❌ Property 'typo' does not exist
  next();
};
```

Rule of thumb: use `req.*` for values a *handler* consumes; use `res.locals.*` for values a *view/serialiser* consumes. Augment whichever you use — never leave it `any`.

### Interface merging is global and one-way

`declare global { namespace Express { interface Request { … } } }` affects **every** file in the program, including third-party middleware's own type checks. It is additive-only — you cannot narrow `user?: User` to `user: User` for a subset of routes via augmentation. That's precisely why the `authed()` wrapper in Concept 6 exists.

Also: the augmentation file must be included by `tsconfig.json`. If it lives outside `include`/`files`, it silently does nothing. Verify with `tsc --listFiles | grep express.d.ts`.

### Error middleware ordering and the "already sent" trap

Express walks middleware in registration order. Error middleware only runs for errors raised **by handlers registered before it**.

```ts
app.use(errorHandler);          // ❌ registered FIRST — never sees route errors
app.get("/users", handler);

app.get("/users", handler);
app.use(notFoundHandler);       // ✅ catches unmatched routes
app.use(errorHandler);          // ✅ catches everything above
```

And if a response already started streaming when the error occurs, you cannot change the status:

```ts
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) {
    next(err);      // hand to Express's default handler → closes the connection
    return;
  }
  res.status(500).json({ error: "Internal Server Error" });
};
```

### Multiple error handlers form a chain

You can register several `ErrorRequestHandler`s. Each may handle its error class and `next(err)` the rest:

```ts
const validationErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (!(err instanceof ZodError)) { next(err); return; }   // not mine — pass on
  res.status(422).json({ ok: false, error: "Validation failed", details: err.issues });
};

const dbErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (!(err instanceof DatabaseError)) { next(err); return; }
  res.status(503).json({ ok: false, error: "Database unavailable", code: "DB_DOWN" });
};

const fallbackErrorHandler: ErrorRequestHandler = (err, req, res, next) => {
  res.status(500).json({ ok: false, error: "Internal Server Error", code: "INTERNAL" });
};

app.use(validationErrorHandler);
app.use(dbErrorHandler);
app.use(fallbackErrorHandler);      // last resort — never calls next(err)
```

Each of those must still physically declare four parameters.

### Performance note — middleware factories allocate per registration, not per request

```ts
// ✅ Set built ONCE at boot:
function requireAnyRole(...roles: Role[]): RequestHandler {
  const allowed = new Set(roles);          // outside the closure body
  return (req, res, next) => { /* uses `allowed` */ };
}

// ❌ Set rebuilt on EVERY request:
function requireAnyRoleSlow(...roles: Role[]): RequestHandler {
  return (req, res, next) => {
    const allowed = new Set(roles);        // allocation per request
  };
}
```

The type signature is identical; only the placement of work differs. Do the expensive setup in the factory body, not the returned handler.

---

## Common mistakes

### Mistake 1 — Error middleware with fewer than 4 parameters

```ts
// ❌ Three parameters. Express treats this as normal middleware — it never runs
//    on errors, and requests that error fall through to Express's HTML page.
const errorHandler = (err: unknown, req: Request, res: Response) => {
  res.status(500).json({ error: "Internal Server Error" });
};
app.use(errorHandler);

// ❌ Also broken — a default value truncates fn.length to 3:
const errorHandler2 = (err: unknown, req: Request, res: Response, next: NextFunction = () => {}) => {
  res.status(500).end();
};

// ✅ Four declared parameters, typed with ErrorRequestHandler:
const errorHandler3: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) { next(err); return; }
  res.status(500).json({ error: "Internal Server Error" });
};
app.use(errorHandler3);
```

If eslint flags `next` as unused, silence it (`// eslint-disable-next-line @typescript-eslint/no-unused-vars`) or name it `_next` with `argsIgnorePattern: "^_"` — never delete it.

### Mistake 2 — Forgetting `return` after responding

```ts
// ❌ Both branches execute — next() runs AFTER the 401 was already sent,
//    producing "Cannot set headers after they are sent to the client".
const requireAuth: RequestHandler = (req, res, next) => {
  if (!req.header("authorization")) {
    res.status(401).json({ error: "Unauthorized" });
  }
  req.user = verifyAuthToken(req.header("authorization")!);   // crashes here
  next();
};

// ✅ Return immediately after writing the response:
const requireAuth2: RequestHandler = (req, res, next) => {
  const header = req.header("authorization");
  if (!header) {
    res.status(401).json({ error: "Unauthorized" });
    return;                                  // ← stops the middleware
  }
  req.user = verifyAuthToken(header);
  next();
};
```

TypeScript cannot catch this — `res.json()` doesn't end the function. Make "respond then `return`" muscle memory.

### Mistake 3 — Unwrapped async middleware

```ts
// ❌ Rejection becomes an unhandled promise rejection. The client waits until
//    it times out. Nothing appears in the error handler.
app.get("/users/:userId", async (req, res) => {
  const user = await userRepository.findById(Number(req.params.userId));
  if (!user) throw ApiError.notFound("User");      // ← never reaches errorHandler
  res.json({ ok: true, data: user });
});

// ❌ Manual try/catch in every handler — correct, but noise you'll forget once:
app.get("/users/:userId", async (req, res, next) => {
  try {
    const user = await userRepository.findById(Number(req.params.userId));
    res.json({ ok: true, data: user });
  } catch (err) { next(err); }
});

// ✅ Wrap once, forget forever:
app.get("/users/:userId", asyncHandler(async (req, res) => {
  const user = await userRepository.findById(Number(req.params.userId));
  if (!user) throw ApiError.notFound("User");      // ✅ → next(err) → errorHandler
  res.json({ ok: true, data: user });
}));
```

### Mistake 4 — Typing route params as non-string

```ts
// ❌ The type says number; the runtime value is the string "42".
//    `req.params.userId * 2` produces NaN-free garbage like "4242".
const handler: RequestHandler<{ userId: number }> = (req, res) => {
  const doubled = req.params.userId * 2;
  res.json({ doubled });
};

// ✅ Type as string, parse and validate explicitly:
const handler2: RequestHandler<{ userId: string }> = (req, res, next) => {
  const userId = Number.parseInt(req.params.userId, 10);
  if (!Number.isSafeInteger(userId) || userId <= 0) {
    next(new ApiError(400, "userId must be a positive integer", "BAD_PARAM"));
    return;
  }
  res.json({ doubled: userId * 2 });
};
```

### Mistake 5 — Assuming validation middleware types the next handler

```ts
// ❌ validateBody's generic dies at the boundary — req.body here is `any`:
app.post("/users", validateBody(CreateUserSchema), (req, res) => {
  createUser(req.body.emial);              // typo compiles — `any` swallows it
});

// ✅ Either annotate the handler with the same generics...
app.post(
  "/users",
  validateBody(CreateUserSchema),
  (req: Request<ParamsDictionary, ApiResponse<PublicUser>, CreateUserBody>, res) => {
    createUser(req.body.emial);            // ❌ now a compile error
  },
);

// ✅ ...or use a builder that infers the link from the schema:
registerRoute(app, {
  method: "post", path: "/users", auth: true,
  bodySchema: CreateUserSchema,
  handler: async (req, res) => {
    createUser(req.body.emial);            // ❌ compile error, inferred from schema
  },
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a fully typed middleware trio for an Express app, using `RequestHandler` and `ErrorRequestHandler` (no hand-written `(req: Request, res: Response, …)` signatures):

1. `correlationId: RequestHandler` — reads `x-correlation-id` from the request headers, or generates a UUID; attaches it to `req.correlationId` and echoes it in the response header. Augment `Express.Request` so `correlationId` is a **required** `string`.
2. `requireApiKey(validKeys: readonly string[]): RequestHandler` — a factory. Reads the `x-api-key` header; responds `401` with `{ ok: false, error: "Invalid API key", code: "BAD_API_KEY" }` if missing or not in `validKeys`; otherwise calls `next()`. Build the lookup structure once in the factory, not per request.
3. `errorHandler: ErrorRequestHandler` — four parameters. If `res.headersSent`, delegate with `next(err)`. Narrow `err` to `unknown`, then handle `Error` vs non-`Error`, logging `req.correlationId` in both cases and responding `500` with a JSON body.

Then wire them in the correct order on an `express()` app with one `/health` route.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed validation middleware system without any external validation library.

```ts
// Define these:
// interface Validator<T> { parse(input: unknown): T }
// class ValidationError extends Error { constructor(readonly issues: { path: string; message: string }[]) }

// Implement combinator functions that return Validator<T>:
//   str(opts?: { minLength?: number; maxLength?: number; pattern?: RegExp }): Validator<string>
//   int(opts?: { min?: number; max?: number }): Validator<number>       // coerces "42" → 42
//   bool(): Validator<boolean>                                          // coerces "true" → true
//   optional<T>(inner: Validator<T>): Validator<T | undefined>
//   literalUnion<const T extends readonly string[]>(...values: T): Validator<T[number]>
//   object<TShape extends Record<string, Validator<unknown>>>(shape: TShape):
//       Validator<{ [K in keyof TShape]: TShape[K] extends Validator<infer U> ? U : never }>
//     — must collect ALL field errors, not just the first, and strip unknown keys

// Then:
//   validateBody<TBody>(schema: Validator<TBody>): RequestHandler<ParamsDictionary, unknown, TBody>
//   validateParams<TParams>(schema: Validator<TParams>): RequestHandler<TParams>
//   validateQuery<TQuery extends Query>(schema: Validator<TQuery>): RequestHandler<ParamsDictionary, unknown, unknown, TQuery>
// Each must call next(new ApiError(422, ..., issues)) on failure.

// Prove it: define
//   const CreateOrderSchema = object({ userId: int({ min: 1 }), currency: literalUnion("USD","EUR","GBP"), notes: optional(str({ maxLength: 500 })) });
// and show that the INFERRED type of the parsed body is
//   { userId: number; currency: "USD" | "EUR" | "GBP"; notes: string | undefined }
// without writing that type by hand anywhere.

// Finally, register a POST /orders route where the handler reads req.body.currency
// and a typo like req.body.currancy is a compile error.
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a complete typed middleware framework on top of Express with these requirements:

```ts
// 1. A `Middleware<TIn, TOut>` abstraction where TIn/TOut describe the CONTEXT
//    a middleware requires and produces (not Express's req/res):
//      type Ctx = Record<string, unknown>;
//      interface Middleware<TIn extends Ctx, TAdd extends Ctx> {
//        (req: Request, res: Response, ctx: TIn): Promise<TAdd>;
//      }
//    Middleware never touches `next` — it returns the context it adds, or throws.

// 2. A `chain()` builder with FULLY inferred accumulation:
//      chain()
//        .use(withRequestId)     // adds { requestId: string }
//        .use(withAuth)          // requires {} , adds { user: AuthenticatedUser }
//        .use(withOrg)           // REQUIRES { user: AuthenticatedUser }, adds { org: Organization }
//        .use(withBody(CreateUserSchema))  // adds { body: CreateUserBody }
//        .handle(async (req, res, ctx) => { ctx.user; ctx.org; ctx.body; ctx.requestId; })
//    - `.use(withOrg)` BEFORE `.use(withAuth)` must be a compile error
//      (its TIn requirement is not satisfied by the accumulated context).
//    - `.handle()` returns a plain `RequestHandler` that Express can mount.
//    - Every ctx property in the handler must be precisely typed, no `any`.

// 3. Error handling:
//    - Any middleware or the handler throwing must reach a single
//      ErrorRequestHandler (4 params) via next(err).
//    - Non-Error throws must be normalised to Error before next().
//    - Build a composable error pipeline: `errorPipeline(...handlers: ErrorRequestHandler[])`
//      returning ONE ErrorRequestHandler with fn.length === 4 that runs them in order,
//      stopping at the first that sends a response.

// 4. A `timing` middleware that records per-middleware durations into
//    ctx.timings: Record<string, number> and emits a Server-Timing header.
//    It must wrap the chain generically without breaking inference.

// 5. Demonstrate:
//    - the compile error from wrong middleware ordering
//    - the compile error from reading ctx.org when withOrg was never used
//    - a handler where req.body typos are compile errors
//    - an async throw inside the third middleware landing in the error pipeline
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
import type {
  Request, Response, NextFunction, RequestHandler, ErrorRequestHandler,
} from "express";

// ── Normal middleware ───────────────────────────────────────────────────────
const m: RequestHandler = (req, res, next) => { next(); };

// ── Generic slots: RequestHandler<Params, ResBody, ReqBody, ReqQuery, Locals>
const typed: RequestHandler<{ userId: string }, ApiResponse<User>, UpdateUserBody> =
  (req, res, next) => { req.params.userId; req.body; res.json({ ok: true, data: user }); };

// ── Error middleware — ALWAYS 4 parameters ──────────────────────────────────
const eh: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) { next(err); return; }
  res.status(500).json({ error: "Internal Server Error" });
};

// ── Middleware factory ──────────────────────────────────────────────────────
const requireRole = (role: Role): RequestHandler => (req, res, next) => {
  if (!req.user?.roles.includes(role)) { res.status(403).end(); return; }
  next();
};

// ── asyncHandler ────────────────────────────────────────────────────────────
const asyncHandler = <P, R, B, Q>(fn: AsyncRequestHandler<P, R, B, Q>): RequestHandler<P, R, B, Q> =>
  (req, res, next) => { Promise.resolve(fn(req, res, next)).catch(next); };

// ── Request augmentation ────────────────────────────────────────────────────
declare global {
  namespace Express {
    interface Request { user?: AuthenticatedUser; requestId: string }
    interface Locals  { startedAt: bigint }
  }
}
export {};

// ── next() call shapes ──────────────────────────────────────────────────────
next();            // continue
next(err);         // → error middleware
next("route");     // skip remaining handlers of this route
next("router");    // skip remaining handlers of this router

// ── Mount order ─────────────────────────────────────────────────────────────
app.use(bodyParser); app.use(requestId); app.use(logger);
app.use("/api", routes);
app.use(notFoundHandler);      // 404 → next(ApiError)
app.use(errorHandler);         // LAST
```

| Thing | Type | Key detail |
|---|---|---|
| Normal middleware | `RequestHandler` | 3 params; return `void`, not `res.json(...)` |
| Error middleware | `ErrorRequestHandler` | **4 declared params** — Express reads `fn.length` |
| `next` | `NextFunction` | `(err?: any)`, plus `"route"` / `"router"` overloads |
| `req.params` | `P` generic, default `{[k:string]:string}` | Always **strings** at runtime — never type as `number` |
| `req.body` | `ReqBody` generic, default `any` | Validate + reassign in middleware; annotate handler |
| `req.query` | `ReqQuery` generic, default `ParsedQs` | Values are `string \| string[] \| ParsedQs \| undefined` |
| `res.locals` | `Locals` generic, default `Record<string, any>` | Augment `Express.Locals` to type it |
| Attaching `req.user` | `declare global { namespace Express { interface Request … } }` | Global + additive; use a wrapper to remove optionality |
| Async handlers | `asyncHandler(fn)` | Express 4 ignores rejections; Express 5 forwards them |
| Factory setup work | Inside the factory body | Not inside the returned handler — that runs per request |

---

## Connected topics

- **20 — Function types** — `RequestHandler` is just a call-signature interface; everything about parameter/return typing applies here.
- **24 — Higher-order function types** — middleware factories and `asyncHandler` are higher-order functions whose generics must be declared on the *outer* function to be inferable.
- **30 — Generic constraints** — `TQuery extends Query` and `const TRoles extends readonly Role[]` are what make validation and role middleware precise.
- **41 — Type guards** — `assertAuthenticated(req): asserts req is Request & { user: AuthenticatedUser }` is the assertion-signature form used to strip `req.user`'s optionality.
- **42 — Discriminated unions** — `ApiResponse<T>` as `{ ok: true; data: T } | { ok: false; error: string }` is the response shape the error middleware and handlers both produce.
