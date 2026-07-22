# 52 — Typing Express routes

## What is this?

Express itself is written in JavaScript and ships **no type information**. The types you use in a TypeScript Express app come from a separate package — `@types/express` — which describes the shape of everything Express hands you: the `Request` object, the `Response` object, the `next` callback, and the handler function signature that ties them together.

```ts
import express, { Request, Response, NextFunction, RequestHandler } from "express";

const app = express();

// The three arguments Express passes to every handler:
app.get("/health", (req: Request, res: Response, next: NextFunction) => {
  res.json({ status: "ok" });
});
```

The interesting part is that `Request` is **generic** — it takes four type parameters that describe the four untrusted, request-shaped blobs of data flowing into your handler:

```ts
Request<Params, ResBody, ReqBody, Query>
//      ↑       ↑        ↑        ↑
//      |       |        |        └── req.query   — the parsed query string
//      |       |        └─────────── req.body    — the parsed request body
//      |       └──────────────────── res.json()  — the response body type
//      └──────────────────────────── req.params  — the URL route parameters
```

Typing an Express route means filling those slots in so that `req.params.userId`, `req.body.email`, `req.query.page` and `res.json(...)` all become checked instead of `any`.

## Why does it matter?

An Express handler is the single place in a Node backend where the **outside world meets your code**. Everything crossing that boundary is untrusted and unshaped:

- `req.params.userId` is a **string**, always — even when your route is `/users/:userId` and the DB column is an integer. Forgetting `parseInt` is one of the most common Node bugs.
- `req.body` is whatever JSON the client sent — possibly `undefined` if `express.json()` middleware isn't mounted, possibly missing the fields you assume exist.
- `req.query.page` is `string | string[] | undefined` — the same route can be hit with `?page=1`, `?page=1&page=2`, or no query at all.
- `res.json(anything)` will happily serialise a shape that doesn't match your API contract — including a raw DB row with the `passwordHash` column still on it.

Untyped, each of these is `any`, and `any` propagates. One `req.body.email` flows into a `sendWelcomeEmail(email)` call that flows into a template string, and the compiler never once asks whether `email` exists. Typed Express routes turn the handler into a **contract checkpoint**: the router declares what it accepts and what it returns, and the compiler enforces both ends.

There's a second, subtler payoff: typed `res.json` means your route handlers and your API response types can't drift apart. Rename a field in `ApiResponse<User>` and every route that returns the old shape breaks at compile time — not in a client bug report three weeks later.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript ────────────────────────────────────────────────────────────────
// Everything is a mystery. The editor gives you nothing. The runtime punishes you.

const express = require("express");
const app = express();
app.use(express.json());

app.get("/users/:userId/orders", async (req, res) => {
  // req.params.userId — is it a string? a number? Nobody knows. (It's a string.)
  const orders = await db.findOrdersByUser(req.params.userId);
  //                                       ^ silently passes "42" to a numeric column

  // req.query.limit — string? array? undefined? All three are possible.
  const limit = req.query.limit || 20;
  orders.slice(0, limit);   // "10" as a slice index → silently returns []

  // Nothing checks the response shape:
  res.json(orders);         // leaks every DB column, including passwordHash on joins
});

app.post("/users", async (req, res) => {
  // req.body might be undefined if express.json() wasn't mounted.
  const { email, password } = req.body;   // TypeError: Cannot destructure ... of undefined

  // Typo? No warning. Ever.
  await createUser({ email, passwrod: password });

  // Forgot to return — code below keeps running and double-sends:
  if (!email) res.status(400).json({ error: "email required" });
  await sendWelcomeEmail(email);           // runs even on the 400 path
  res.status(201).json({ ok: true });      // ERR_HTTP_HEADERS_SENT
});
```

```ts
// ── TypeScript ────────────────────────────────────────────────────────────────
// Every boundary value has a declared shape; the compiler enforces both ends.

import express, { Request, Response } from "express";

const app = express();
app.use(express.json());

// Declare the four slots up front:
interface OrderParams  { userId: string }                    // params are ALWAYS strings
interface OrderQuery   { limit?: string; cursor?: string }   // query values are strings too
interface OrderListBody { orders: OrderDto[]; nextCursor: string | null }

app.get(
  "/users/:userId/orders",
  async (
    req: Request<OrderParams, OrderListBody, unknown, OrderQuery>,
    res: Response<OrderListBody>,
  ): Promise<void> => {
    const userId = Number(req.params.userId);   // ✅ compiler reminded us it's a string
    if (!Number.isInteger(userId)) {
      res.status(400).json({ orders: [], nextCursor: null });
      return;                                    // ✅ explicit return — no double-send
    }

    const limit = req.query.limit ? Number(req.query.limit) : 20; // ✅ limit?: string
    const orders = await orderRepo.findByUser(userId, limit);

    // ❌ res.json(orders) — Error: OrderRow[] is not assignable to OrderListBody
    res.json({ orders: orders.map(toOrderDto), nextCursor: null }); // ✅ exact contract
  },
);
```

The revelation: **`req.params.userId` being typed `string` is not a limitation — it's the compiler telling you the truth JavaScript hid.** Every `parseInt` you were supposed to write, but forgot, now shows up as a type error.

---

## Syntax

```ts
// ── Install ────────────────────────────────────────────────────────────────────
// npm i express
// npm i -D typescript @types/express @types/node ts-node
//   express       — the runtime library (plain JS)
//   @types/express — the type declarations (compile time only, erased in the build)

// ── Import the value and the types ─────────────────────────────────────────────
import express from "express";                                  // default export: the app factory
import type { Request, Response, NextFunction } from "express";  // types only — erased at build
import type { RequestHandler, ErrorRequestHandler, Router } from "express";

const app = express();          // Express.Application
app.use(express.json());        // body parser — REQUIRED for req.body to be populated

// ── The plainest possible typed handler ────────────────────────────────────────
app.get("/health", (req: Request, res: Response): void => {
  res.json({ status: "ok" });   // res.json accepts anything when ResBody is unspecified
});

// ── The four Request generic slots, in order ───────────────────────────────────
// Request<Params, ResBody, ReqBody, Query, Locals>
//   Params  → req.params   (route placeholders like /:userId)
//   ResBody → the body type res.json()/res.send() will accept
//   ReqBody → req.body     (after a body-parsing middleware)
//   Query   → req.query    (parsed query string)
//   Locals  → res.locals   (rarely typed; 5th slot)
app.post(
  "/users/:orgId/members",
  (
    req: Request<{ orgId: string }, MemberDto, CreateMemberBody, { notify?: string }>,
    res: Response<MemberDto>,
  ): void => {
    req.params.orgId;      // string
    req.body.email;        // string — from CreateMemberBody
    req.query.notify;      // string | undefined
    res.status(201).json(createdMember);  // must be MemberDto
  },
);

// ── RequestHandler — the same thing, typed on the function instead of the args ──
// RequestHandler<Params, ResBody, ReqBody, Query>
const getUser: RequestHandler<{ userId: string }, UserDto> = (req, res) => {
  req.params.userId;     // ✅ inferred string — no per-argument annotations needed
  res.json(userDto);     // ✅ must be UserDto
};
app.get("/users/:userId", getUser);

// ── next() and middleware ──────────────────────────────────────────────────────
const requestLogger = (req: Request, res: Response, next: NextFunction): void => {
  console.log(`${req.method} ${req.path}`);
  next();                 // pass control on; next(err) to jump to the error handler
};
app.use(requestLogger);

// ── Error handler — FOUR params, and the arity is what makes Express recognise it ─
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  res.status(500).json({ error: "Internal Server Error" });
};
app.use(errorHandler);   // must be registered LAST

// ── Router ─────────────────────────────────────────────────────────────────────
import { Router } from "express";
const userRouter: Router = Router();
userRouter.get("/:userId", getUser);
app.use("/users", userRouter);   // mounts at /users/:userId
```

---

## How it works — concept by concept

### Concept 1 — `@types/express` and where the types actually live

`express` is a JavaScript package. `@types/express` is a **declaration-only** package from DefinitelyTyped that describes it. Install both:

```bash
npm i express
npm i -D typescript @types/express @types/node
```

`@types/*` packages are picked up automatically by `tsc` — you don't import them. TypeScript scans `node_modules/@types/**` and loads every package it finds there, unless you restrict it in `tsconfig.json`:

```jsonc
{
  "compilerOptions": {
    "types": ["node", "express"],  // ← if present, ONLY these @types packages load
    "esModuleInterop": true,       // ← required for `import express from "express"`
    "strict": true
  }
}
```

The important structural fact: `@types/express` is mostly a **thin re-export**. The real definitions live in a second package, `@types/express-serve-static-core`, which `@types/express` depends on:

```ts
// Simplified from @types/express/index.d.ts:
import * as core from "express-serve-static-core";

declare namespace e {
  interface Request<P = core.ParamsDictionary, ResBody = any, ReqBody = any, ReqQuery = core.Query>
    extends core.Request<P, ResBody, ReqBody, ReqQuery> {}
  interface Response<ResBody = any> extends core.Response<ResBody> {}
}
```

This matters enormously for the next doc: when you want to add `req.user`, you augment `express-serve-static-core`, **not** `express`. (See **53 — Extending Express Request**.)

```ts
// Two equivalent ways to get the types — pick one style and stick to it:
import { Request, Response } from "express";                     // via the re-export
import { Request, Response } from "express-serve-static-core";   // the source (rarely used directly)
```

### Concept 2 — The four `Request` generic slots and their default values

The full signature, with defaults:

```ts
interface Request<
  P     = ParamsDictionary,   // req.params  — default: Record<string, string>
  ResBody = any,              // the res body type — default: any
  ReqBody = any,              // req.body    — default: any  ← the dangerous one
  ReqQuery = Query,           // req.query   — default: QueryString.ParsedQs
  Locals extends Record<string, any> = Record<string, any>,  // res.locals
> { /* ... */ }
```

What each default gives you if you leave it alone:

```ts
app.post("/orders", (req: Request, res: Response) => {
  req.params.anything;   // string       — ParamsDictionary is an index signature of string
  req.body;              // any          — ⚠️ no safety at all, this is the leak
  req.query.page;        // string | string[] | ParsedQs | ParsedQs[] | undefined  ⚠️ ugly union
  res.json(anything);    // ResBody = any — accepts literally anything
});
```

Filling the slots:

```ts
interface CreateOrderParams { /* no params on this route */ }
interface CreateOrderBody   { userId: number; items: { sku: string; qty: number }[] }
interface CreateOrderQuery  { dryRun?: string }        // ALWAYS string — see Concept 4
interface OrderDto          { orderId: string; totalCents: number; status: string }

app.post(
  "/orders",
  (
    req: Request<CreateOrderParams, OrderDto, CreateOrderBody, CreateOrderQuery>,
    res: Response<OrderDto>,
  ): void => {
    req.body.userId;      // number  ✅
    req.body.itmes;       // ❌ Error: Property 'itmes' does not exist — typo caught
    req.query.dryRun;     // string | undefined ✅
    res.json({ orderId: "o_1", totalCents: 4999, status: "pending" }); // ✅
  },
);
```

**The positional trap:** `ResBody` sits at slot 2, between `Params` and `ReqBody`. Almost everyone's first instinct is that slot 2 is the body. It isn't. If you only care about the request body, you must still pass something for the first two slots:

```ts
// ❌ WRONG — CreateOrderBody landed in the ResBody slot; req.body is still any:
Request<{}, CreateOrderBody>

// ✅ RIGHT — skip the first two with placeholders:
Request<{}, unknown, CreateOrderBody>
// or, when you also want the response typed:
Request<{}, OrderDto, CreateOrderBody>
```

Use `unknown` rather than `any` for slots you're not using — it keeps the "I haven't declared this" honest.

### Concept 3 — `RequestHandler`: typing the function, not the arguments

`RequestHandler` is a function type covering the whole handler. It takes the *same four* generic parameters in the *same order*, and lets you drop every per-parameter annotation:

```ts
// Simplified definition:
type RequestHandler<P = ParamsDictionary, ResBody = any, ReqBody = any, ReqQuery = Query> = (
  req:  Request<P, ResBody, ReqBody, ReqQuery>,
  res:  Response<ResBody>,
  next: NextFunction,
) => void;

// ── Annotating the arguments (verbose, but explicit) ───────────────────────────
const createOrderA = (
  req: Request<{}, OrderDto, CreateOrderBody>,
  res: Response<OrderDto>,
): void => { /* ... */ };

// ── Annotating with RequestHandler (preferred — one place, no duplication) ─────
const createOrderB: RequestHandler<{}, OrderDto, CreateOrderBody> = (req, res) => {
  req.body.userId;   // ✅ inferred — contextual typing flows into the parameters
  res.json(dto);     // ✅ Response<OrderDto> inferred too
};
```

The second form is better because `ResBody` is declared **once** and applies to both `req` and `res`. In the first form, nothing stops you writing `Request<{}, OrderDto, B>` and `Response<SomethingElse>` — they'd disagree silently.

`RequestHandler` also gives you a clean vocabulary for reusable middleware:

```ts
// Middleware with no route-specific knowledge:
const requestIdMiddleware: RequestHandler = (req, res, next) => {
  res.setHeader("x-request-id", randomUUID());
  next();
};

// Middleware that only makes sense on routes with :userId:
const loadUser: RequestHandler<{ userId: string }> = async (req, res, next) => {
  const user = await userRepo.findById(Number(req.params.userId));
  if (!user) { res.status(404).json({ error: "Not found" }); return; }
  next();
};

// A factory producing typed handlers:
function requireRole(role: "admin" | "member"): RequestHandler {
  return (req, res, next) => {
    // role captured in the closure
    next();
  };
}
```

### Concept 4 — `req.params` are always strings, and so is `req.query`

This is the single most valuable thing typed Express teaches a JS developer.

```ts
// Route: GET /users/:userId/orders/:orderId
const handler: RequestHandler<{ userId: string; orderId: string }> = (req, res) => {
  req.params.userId;   // string — "42", never 42
  req.params.orderId;  // string
};
```

There is no way for Express to know that `:userId` is numeric — it comes out of a URL, so it's text. Converting it is *your* job, and the conversion should validate:

```ts
// ── A reusable, safe numeric param parser ──────────────────────────────────────
function parseIntParam(raw: string, name: string): number {
  const value = Number(raw);
  if (!Number.isInteger(value) || value <= 0) {
    throw new BadRequestError(`Invalid ${name}: expected a positive integer, got "${raw}"`);
  }
  return value;
}

const getOrder: RequestHandler<{ userId: string; orderId: string }> = async (req, res) => {
  const userId  = parseIntParam(req.params.userId,  "userId");   // number ✅
  const orderId = parseIntParam(req.params.orderId, "orderId");  // number ✅
  const order   = await orderRepo.find(userId, orderId);
  res.json(order);
};
```

The default query type is deliberately paranoid, because Express's query parser really can produce all of these:

```ts
// GET /search?tag=node            → req.query.tag === "node"
// GET /search?tag=node&tag=ts     → req.query.tag === ["node", "ts"]
// GET /search?filter[min]=10      → req.query.filter === { min: "10" }  (extended parser)
// GET /search                     → req.query.tag === undefined

app.get("/search", (req: Request, res: Response) => {
  const tag = req.query.tag;
  // tag: string | string[] | ParsedQs | ParsedQs[] | undefined
  tag.toUpperCase();   // ❌ Error — and rightly so, it might be an array
});
```

You have two honest options. Narrow at runtime:

```ts
function firstString(value: unknown): string | undefined {
  if (typeof value === "string") return value;
  if (Array.isArray(value) && typeof value[0] === "string") return value[0];
  return undefined;
}

app.get("/search", (req: Request, res: Response) => {
  const tag = firstString(req.query.tag) ?? "all";  // string ✅
  res.json({ tag });
});
```

Or declare the query slot as the shape you *require*, and validate it with a schema before trusting it:

```ts
interface SearchQuery { tag?: string; page?: string; perPage?: string }

const search: RequestHandler<{}, SearchResults, unknown, SearchQuery> = (req, res) => {
  const page    = Number(req.query.page    ?? "1");
  const perPage = Math.min(Number(req.query.perPage ?? "20"), 100);
  // ...
};
```

Declaring `SearchQuery` is an *assertion*, not a guarantee — a client can still send `?page=1&page=2` and get an array at runtime. Treat the generic slot as documentation of intent, and pair it with real validation (Zod, class-validator, etc.) for anything security-relevant.

### Concept 5 — Typed `res.json`, `res.send`, and `res.status`

When you supply `ResBody`, `Response<ResBody>` locks down `json` and `send`:

```ts
interface UserDto { userId: number; email: string; displayName: string }

const getUser: RequestHandler<{ userId: string }, UserDto> = async (req, res) => {
  const row = await userRepo.findById(Number(req.params.userId));

  res.json(row);
  // ❌ Error: Type 'UserRow' is not assignable to 'UserDto'
  //    — UserRow has passwordHash, createdAt, and lacks displayName.
  //    The compiler just stopped a credential leak.

  res.json({ userId: row.id, email: row.email, displayName: row.name });  // ✅
};
```

`res.status()` returns `this`, so the chain keeps the type:

```ts
res.status(201).json(userDto);   // ✅ still Response<UserDto>
res.status(404).json(userDto);   // ✅ type-wise fine — status codes are not type-checked
```

Because status codes aren't part of the type, error responses fight the contract. The clean fix is a **union response type** — a discriminated union of success and error bodies (see **42 — Discriminated Unions**):

```ts
type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; code: string };

const getUser: RequestHandler<{ userId: string }, ApiResponse<UserDto>> = async (req, res) => {
  const user = await userRepo.findById(Number(req.params.userId));
  if (!user) {
    res.status(404).json({ ok: false, error: "User not found", code: "NOT_FOUND" }); // ✅
    return;
  }
  res.status(200).json({ ok: true, data: toUserDto(user) });                          // ✅
};
```

Now *both* branches are checked, and a route that forgets `code` on the error branch is a compile error.

Two extra notes on `Response`:

```ts
res.send("plain text");        // send accepts ResBody | string | Buffer — looser than json
res.sendStatus(204);           // sets status + sends the status text; no body typing
res.locals.requestId = "abc";  // res.locals is the 5th Request slot / 2nd Response slot
```

### Concept 6 — Async handlers and the `Promise<void>` return type

Express 4's `RequestHandler` declares its return type as `void`. An `async` function returns `Promise<void>`. TypeScript **allows** a `Promise<void>`-returning function where `void` is expected — this is a deliberate rule: a `void` return type means "the caller ignores whatever comes back," so any return value is assignable.

```ts
// ✅ Compiles fine — Promise<void> is assignable to void:
app.get("/users/:userId", async (req: Request, res: Response): Promise<void> => {
  const user = await userRepo.findById(Number(req.params.userId));
  res.json(user);
});
```

But "Express ignores the return value" is exactly the problem. In **Express 4**, a rejected promise is **not** caught — it becomes an unhandled rejection, the request hangs until it times out, and (on Node 15+) the process may crash:

```ts
// ❌ Express 4 — if userRepo throws, this request hangs forever:
app.get("/users/:userId", async (req, res) => {
  const user = await userRepo.findById(Number(req.params.userId)); // throws
  res.json(user);
});
```

The standard fix is an async wrapper that funnels rejections into `next()`:

```ts
import type { Request, Response, NextFunction, RequestHandler } from "express";

// Generic over all four slots so the wrapper doesn't erase your types:
function asyncHandler<P, ResBody, ReqBody, ReqQuery>(
  fn: (
    req: Request<P, ResBody, ReqBody, ReqQuery>,
    res: Response<ResBody>,
    next: NextFunction,
  ) => Promise<void>,
): RequestHandler<P, ResBody, ReqBody, ReqQuery> {
  return (req, res, next) => {
    // Promise.resolve handles both sync throws and async rejections:
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage — types survive the wrapper:
app.get(
  "/users/:userId",
  asyncHandler<{ userId: string }, ApiResponse<UserDto>, unknown, {}>(async (req, res) => {
    const user = await userRepo.findById(Number(req.params.userId)); // req.params typed ✅
    if (!user) { res.status(404).json({ ok: false, error: "Not found", code: "NOT_FOUND" }); return; }
    res.json({ ok: true, data: toUserDto(user) });
  }),
);
```

> **Express 5 difference:** Express 5 awaits handler return values and forwards rejections to the error middleware automatically, so `asyncHandler` becomes unnecessary. Its `@types/express@5` `RequestHandler` returns `void | Promise<void>`. If you're on Express 4 (`@types/express@4.x`), keep the wrapper. Express 5 also changed `req.query` to be a getter, dropped `res.send(status)`, and renamed `req.param()` out of existence — none of which affects the four generic slots.

Why you should still annotate `: Promise<void>` explicitly on async handlers: without it, TypeScript infers the return type from your `return` statements, and a stray `return res.json(...)` makes it `Promise<Response>`. That compiles, but it hides the real bug — see Mistake 3.

### Concept 7 — `NextFunction`, error handlers, and handler arity

```ts
// Simplified:
interface NextFunction {
  (err?: any): void;
  (deferToNext: "router"): void;   // skip the rest of this router
  (deferToNext: "route"):  void;   // skip the rest of THIS route's handler stack
}
```

Three distinct behaviours:

```ts
const authGuard: RequestHandler = (req, res, next) => {
  const authToken = req.headers.authorization;

  if (!authToken)             { next(new UnauthorizedError("Missing token")); return; } // → error handler
  if (authToken === "public") { next("route"); return; }   // → skip remaining handlers on this route
  next();                                                   // → continue normally
};
```

The error handler is identified by **arity** — Express counts `fn.length === 4`. This is a runtime check on the JavaScript function, which means TypeScript can't help you if you drop `next`:

```ts
// ❌ Three params — Express registers this as a NORMAL middleware, never an error handler:
app.use((err: Error, req: Request, res: Response) => {
  res.status(500).json({ error: err.message });
});

// ✅ Four params, and `next` must be present even if unused:
import type { ErrorRequestHandler } from "express";

const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) { next(err); return; }   // delegate to Express's default handler
  const status = err instanceof ApiError ? err.statusCode : 500;
  res.status(status).json({ ok: false, error: err.message, code: err.code ?? "INTERNAL" });
};

app.use(errorHandler);   // register AFTER all routes
```

With `noUnusedParameters: true` in tsconfig, the unused `next` triggers an error. Prefix it with `_` — TypeScript ignores underscore-prefixed unused parameters, and `fn.length` is unaffected:

```ts
const errorHandler: ErrorRequestHandler = (err, _req, res, _next) => { /* ... */ };
//                                              ↑            ↑ arity is still 4 ✅
```

### Concept 8 — Typing `Router` and composing route modules

`Router()` returns a `Router`, which has the same `get/post/put/delete/use` methods as `Application`:

```ts
// src/routes/users.routes.ts
import { Router } from "express";
import type { RequestHandler } from "express";

const router: Router = Router();

// A Router mounted at /users sees paths RELATIVE to the mount point:
router.get("/",         listUsers);          // GET  /users
router.get("/:userId",  getUser);            // GET  /users/:userId
router.post("/",        createUser);         // POST /users
router.patch("/:userId", updateUser);        // PATCH /users/:userId

export default router;
```

```ts
// src/app.ts
import express from "express";
import type { Application } from "express";
import userRouter from "./routes/users.routes";

const app: Application = express();
app.use(express.json());
app.use("/api/users", userRouter);   // final paths: /api/users, /api/users/:userId
export default app;
```

**Param inheritance gotcha:** by default a child router does **not** see the parent's params. Pass `mergeParams: true`:

```ts
// Parent: app.use("/orgs/:orgId/members", memberRouter)
const memberRouter = Router({ mergeParams: true });   // ← without this, req.params.orgId is missing

// The type doesn't know about the parent's params either — declare them yourself:
const listMembers: RequestHandler<{ orgId: string }> = (req, res) => {
  req.params.orgId;   // ✅ typed, and populated at runtime thanks to mergeParams
  res.json([]);
};
memberRouter.get("/", listMembers);
```

`router.route()` chains methods on one path and preserves types per-handler:

```ts
router
  .route("/:userId")
  .get(getUser satisfies RequestHandler<{ userId: string }, ApiResponse<UserDto>>)
  .patch(updateUser)
  .delete(deleteUser);
```

---

## Example 1 — basic

```ts
// A small, fully typed users API — every boundary value has a declared shape.

import express from "express";
import type { Request, Response, NextFunction, RequestHandler } from "express";

// ── Domain + DTO types ─────────────────────────────────────────────────────────

interface UserRow {
  id:           number;
  email:        string;
  display_name: string;
  password_hash: string;   // must NEVER leave the server
  created_at:   Date;
}

interface UserDto {
  userId:      number;
  email:       string;
  displayName: string;
  createdAt:   string;     // ISO 8601
}

function toUserDto(row: UserRow): UserDto {
  return {
    userId:      row.id,
    email:       row.email,
    displayName: row.display_name,
    createdAt:   row.created_at.toISOString(),
  };
}

// ── Response envelope — a discriminated union so errors are typed too ───────────

type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; code: string };

// ── Request-shaped types, one per route ────────────────────────────────────────

interface UserIdParams  { userId: string }             // params are always strings
interface CreateUserBody { email: string; password: string; displayName: string }
interface ListUsersQuery { page?: string; perPage?: string; search?: string }

// ── Handlers ───────────────────────────────────────────────────────────────────

// GET /users?page=1&perPage=20&search=ada
const listUsers: RequestHandler<
  {},                                  // Params  — none on this route
  ApiResponse<UserDto[]>,              // ResBody
  unknown,                             // ReqBody — GET has none
  ListUsersQuery                       // Query
> = async (req, res) => {
  const page    = Math.max(Number(req.query.page    ?? "1"), 1);
  const perPage = Math.min(Math.max(Number(req.query.perPage ?? "20"), 1), 100);
  const search  = req.query.search ?? "";              // string | undefined → string

  const rows = await userRepo.list({ offset: (page - 1) * perPage, limit: perPage, search });
  res.json({ ok: true, data: rows.map(toUserDto) });   // ✅ shape enforced
};

// GET /users/:userId
const getUser: RequestHandler<UserIdParams, ApiResponse<UserDto>> = async (req, res) => {
  const userId = Number(req.params.userId);            // ✅ compiler forced the conversion
  if (!Number.isInteger(userId) || userId <= 0) {
    res.status(400).json({ ok: false, error: "userId must be a positive integer", code: "INVALID_PARAM" });
    return;                                            // ✅ explicit return — never fall through
  }

  const row = await userRepo.findById(userId);
  if (!row) {
    res.status(404).json({ ok: false, error: "User not found", code: "NOT_FOUND" });
    return;
  }

  // res.json(row) would be an error here — UserRow ≠ UserDto, password_hash can't escape.
  res.json({ ok: true, data: toUserDto(row) });
};

// POST /users
const createUser: RequestHandler<{}, ApiResponse<UserDto>, CreateUserBody> = async (req, res) => {
  const { email, password, displayName } = req.body;   // ✅ all typed string

  if (!email.includes("@")) {
    res.status(422).json({ ok: false, error: "Invalid email", code: "VALIDATION_FAILED" });
    return;
  }
  if (password.length < 12) {
    res.status(422).json({ ok: false, error: "Password too short", code: "VALIDATION_FAILED" });
    return;
  }

  const existing = await userRepo.findByEmail(email);
  if (existing) {
    res.status(409).json({ ok: false, error: "Email already registered", code: "CONFLICT" });
    return;
  }

  const row = await userRepo.create({ email, password, displayName });
  res.status(201).json({ ok: true, data: toUserDto(row) });
};

// ── Middleware ─────────────────────────────────────────────────────────────────

const requestLogger: RequestHandler = (req: Request, res: Response, next: NextFunction): void => {
  const startedAt = Date.now();
  res.on("finish", () => {
    console.log(`${req.method} ${req.originalUrl} ${res.statusCode} ${Date.now() - startedAt}ms`);
  });
  next();
};

// ── Wiring ─────────────────────────────────────────────────────────────────────

const app = express();
app.use(express.json());     // populates req.body — without it, req.body is undefined
app.use(requestLogger);

app.get("/users",         listUsers);
app.get("/users/:userId", getUser);
app.post("/users",        createUser);

app.listen(3000, () => console.log("listening on :3000"));
```

---

## Example 2 — real world backend use case

```ts
// A production-shaped Express 4 setup: typed router modules, an async wrapper that
// preserves generics, typed validation middleware, and a typed error pipeline.

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/types.ts — shared HTTP types
// ═══════════════════════════════════════════════════════════════════════════════

import type { Request, Response, NextFunction, RequestHandler, ErrorRequestHandler } from "express";

export type ApiResponse<T> =
  | { ok: true;  data: T; meta?: { page: number; perPage: number; total: number } }
  | { ok: false; error: string; code: ErrorCode; details?: unknown };

export type ErrorCode =
  | "VALIDATION_FAILED"
  | "UNAUTHORIZED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "CONFLICT"
  | "RATE_LIMITED"
  | "INTERNAL";

// A handler whose response body is always our envelope:
export type ApiHandler<
  Params  = {},
  Data    = unknown,
  ReqBody = unknown,
  Query   = {},
> = RequestHandler<Params, ApiResponse<Data>, ReqBody, Query>;

// The async variant — note the Promise<void> return type:
export type AsyncApiHandler<
  Params  = {},
  Data    = unknown,
  ReqBody = unknown,
  Query   = {},
> = (
  req:  Request<Params, ApiResponse<Data>, ReqBody, Query>,
  res:  Response<ApiResponse<Data>>,
  next: NextFunction,
) => Promise<void>;

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/async-handler.ts — Express 4 rejection safety
// ═══════════════════════════════════════════════════════════════════════════════

// Express 4 does NOT catch rejected promises from handlers. This wrapper does.
// Express 5 forwards rejections automatically, so there you can drop it.
export function asyncHandler<Params, Data, ReqBody, Query>(
  fn: AsyncApiHandler<Params, Data, ReqBody, Query>,
): ApiHandler<Params, Data, ReqBody, Query> {
  return (req, res, next): void => {
    // Promise.resolve() also converts a synchronous throw into a rejection:
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/errors.ts — typed error class the error handler understands
// ═══════════════════════════════════════════════════════════════════════════════

export class ApiError extends Error {
  constructor(
    readonly statusCode: number,
    readonly code: ErrorCode,
    message: string,
    readonly details?: unknown,
  ) {
    super(message);
    this.name = "ApiError";
  }

  static badRequest(message: string, details?: unknown): ApiError {
    return new ApiError(400, "VALIDATION_FAILED", message, details);
  }
  static unauthorized(message = "Authentication required"): ApiError {
    return new ApiError(401, "UNAUTHORIZED", message);
  }
  static forbidden(message = "Insufficient permissions"): ApiError {
    return new ApiError(403, "FORBIDDEN", message);
  }
  static notFound(resource: string): ApiError {
    return new ApiError(404, "NOT_FOUND", `${resource} not found`);
  }
  static conflict(message: string): ApiError {
    return new ApiError(409, "CONFLICT", message);
  }
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/validate.ts — typed body validation middleware
// ═══════════════════════════════════════════════════════════════════════════════

// A minimal schema contract — swap for Zod's ZodType in a real project.
export interface Validator<T> {
  parse(input: unknown): T;    // throws on invalid input
}

// The generic T flows into the ReqBody slot of the returned handler, so downstream
// handlers registered on the same route see a typed req.body.
export function validateBody<T>(schema: Validator<T>): RequestHandler<{}, unknown, T> {
  return (req, res, next): void => {
    try {
      // Reassigning req.body is safe here — it's typed T on the way out.
      req.body = schema.parse(req.body);
      next();
    } catch (err) {
      next(ApiError.badRequest("Request body failed validation", err));
    }
  };
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/routes/orders.routes.ts — a typed router module
// ═══════════════════════════════════════════════════════════════════════════════

import { Router } from "express";

interface OrderDto {
  orderId:    string;
  userId:     number;
  status:     "pending" | "paid" | "shipped" | "cancelled";
  totalCents: number;
  createdAt:  string;
}

interface OrderIdParams   { orderId: string }
interface UserIdParams    { userId: string }
interface CreateOrderBody { userId: number; items: Array<{ sku: string; qty: number }> }
interface ListOrdersQuery { status?: OrderDto["status"]; page?: string; perPage?: string }

const orderRouter: Router = Router();

// ── GET /orders?status=paid&page=2 ─────────────────────────────────────────────
const listOrders: AsyncApiHandler<{}, OrderDto[], unknown, ListOrdersQuery> = async (req, res) => {
  const page    = Math.max(Number(req.query.page    ?? "1"), 1);
  const perPage = Math.min(Math.max(Number(req.query.perPage ?? "25"), 1), 100);
  const status  = req.query.status;   // "pending" | "paid" | ... | undefined

  const { rows, total } = await orderRepo.list({
    status,
    offset: (page - 1) * perPage,
    limit:  perPage,
  });

  res.json({
    ok:   true,
    data: rows.map(toOrderDto),
    meta: { page, perPage, total },   // ✅ meta is part of the success branch
  });
};

// ── GET /orders/:orderId ───────────────────────────────────────────────────────
const getOrder: AsyncApiHandler<OrderIdParams, OrderDto> = async (req, res) => {
  const { orderId } = req.params;                 // string ✅
  const order = await orderRepo.findById(orderId);
  if (!order) throw ApiError.notFound("Order");    // caught by asyncHandler → next(err)
  res.json({ ok: true, data: toOrderDto(order) });
};

// ── POST /orders ───────────────────────────────────────────────────────────────
const createOrder: AsyncApiHandler<{}, OrderDto, CreateOrderBody> = async (req, res) => {
  const { userId, items } = req.body;             // number, Array<{sku,qty}> ✅

  if (items.length === 0) throw ApiError.badRequest("Order must contain at least one item");

  const user = await userRepo.findById(userId);
  if (!user) throw ApiError.notFound("User");

  const order = await orderRepo.create({ userId, items });
  res.status(201).json({ ok: true, data: toOrderDto(order) });
};

// ── DELETE /orders/:orderId ────────────────────────────────────────────────────
const cancelOrder: AsyncApiHandler<OrderIdParams, OrderDto> = async (req, res) => {
  const order = await orderRepo.findById(req.params.orderId);
  if (!order)                     throw ApiError.notFound("Order");
  if (order.status === "shipped") throw ApiError.conflict("Cannot cancel a shipped order");

  const cancelled = await orderRepo.updateStatus(order.orderId, "cancelled");
  res.json({ ok: true, data: toOrderDto(cancelled) });
};

// ── Registration — asyncHandler preserves every generic slot ───────────────────
orderRouter.get("/",         asyncHandler(listOrders));
orderRouter.get("/:orderId", asyncHandler(getOrder));
orderRouter.post("/",        validateBody(createOrderSchema), asyncHandler(createOrder));
orderRouter.delete("/:orderId", asyncHandler(cancelOrder));

export default orderRouter;

// ═══════════════════════════════════════════════════════════════════════════════
// src/routes/user-orders.routes.ts — nested router with mergeParams
// ═══════════════════════════════════════════════════════════════════════════════

// Mounted at /users/:userId/orders — mergeParams is REQUIRED to see :userId.
const userOrderRouter: Router = Router({ mergeParams: true });

const listUserOrders: AsyncApiHandler<UserIdParams, OrderDto[]> = async (req, res) => {
  const userId = Number(req.params.userId);   // ✅ available thanks to mergeParams
  if (!Number.isInteger(userId)) throw ApiError.badRequest("userId must be an integer");

  const rows = await orderRepo.findByUser(userId);
  res.json({ ok: true, data: rows.map(toOrderDto) });
};

userOrderRouter.get("/", asyncHandler(listUserOrders));

export { userOrderRouter };

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/error-handler.ts — the terminal error middleware (4 params!)
// ═══════════════════════════════════════════════════════════════════════════════

export const notFoundHandler: RequestHandler = (req, res): void => {
  res.status(404).json({ ok: false, error: `No route for ${req.method} ${req.path}`, code: "NOT_FOUND" });
};

export const errorHandler: ErrorRequestHandler = (err, req, res, next): void => {
  // If the response already started streaming, Express's default handler must finish it:
  if (res.headersSent) { next(err); return; }

  if (err instanceof ApiError) {
    res.status(err.statusCode).json({
      ok:      false,
      error:   err.message,
      code:    err.code,
      details: err.details,
    });
    return;
  }

  // Unknown errors: log internally, expose nothing.
  console.error("Unhandled error", { path: req.path, method: req.method, err });
  res.status(500).json({ ok: false, error: "Internal Server Error", code: "INTERNAL" });
};

// ═══════════════════════════════════════════════════════════════════════════════
// src/app.ts — assembly. Order matters more than types here.
// ═══════════════════════════════════════════════════════════════════════════════

import express from "express";
import type { Application } from "express";

export function createApp(): Application {
  const app = express();

  app.use(express.json({ limit: "1mb" }));                 // 1. parse bodies
  app.use(express.urlencoded({ extended: true }));         // 2. form bodies

  app.use("/api/orders", orderRouter);                     // 3. routes
  app.use("/api/users/:userId/orders", userOrderRouter);

  app.use(notFoundHandler);                                // 4. 404 — after all routes
  app.use(errorHandler);                                   // 5. errors — absolutely last

  return app;
}
```

---

## Going deeper

### The `ResBody` slot is bivariant in practice — it does not stop every leak

`Response<ResBody>.json` is declared as `json(body?: ResBody): this`. Object assignability in TypeScript is structural, so a value with **extra** properties is assignable — unless it's a fresh object literal, which triggers excess property checking:

```ts
interface UserDto { userId: number; email: string }

const send: RequestHandler<{}, UserDto> = (req, res) => {
  // ❌ Excess property check fires — object literal has an unexpected key:
  res.json({ userId: 1, email: "a@b.c", passwordHash: "..." });

  // ⚠️ NO error — the value came from a variable, so excess properties pass:
  const row = { userId: 1, email: "a@b.c", passwordHash: "..." };
  res.json(row);   // compiles, and leaks passwordHash at runtime
};
```

Typed `res.json` catches inline mistakes, not laundered ones. The reliable defence is an explicit mapper (`toUserDto`) that constructs the DTO field by field, so extra columns physically can't reach the response.

### `RequestHandler`'s `void` return type hides double-sends

`RequestHandler` returns `void`, and TypeScript lets *anything* be returned where `void` is expected. That's why `return res.json(...)` compiles even though it returns a `Response`:

```ts
const handler: RequestHandler = (req, res) => {
  if (!req.headers.authorization) return res.status(401).json({ error: "no token" });
  //                              ^ compiles — Response assignable to void return position
  res.json({ ok: true });
};
```

This particular snippet is actually *correct* (the `return` prevents the double-send), but the pattern is fragile: as soon as someone drops the `return` keyword, nothing complains. Two ways to harden it:

```ts
// 1. Annotate the return type explicitly — now `return res.json()` is an error:
const handler = (req: Request, res: Response): void => {
  if (!req.headers.authorization) {
    res.status(401).json({ error: "no token" });
    return;                                        // ← the only correct shape
  }
  res.json({ ok: true });
};

// 2. Enable the ESLint rule `@typescript-eslint/no-confusing-void-expression`,
//    which flags `return someVoidReturningCall()`.
```

The `void`-accepts-anything rule exists so callbacks like `arr.forEach(x => map.set(x, 1))` work — `Map.set` returns the map, `forEach` wants `void`. It's a feature everywhere except Express handlers.

### Overload resolution: why `app.get` sometimes infers your params for free

`@types/express` declares `IRouterMatcher` with an overload that captures the path as a string literal and, in `@types/express` v4.17.21+, uses a template-literal type `RouteParameters<Route>` to extract `:param` names:

```ts
// With a literal path, params can be inferred without annotation:
app.get("/users/:userId/orders/:orderId", (req, res) => {
  req.params.userId;   // string  ✅ inferred from the path literal
  req.params.orderId;  // string  ✅
  req.params.nope;     // ❌ Error — not in the path
});

// But the moment you annotate the handler, YOUR type wins:
const h: RequestHandler<{ userId: string }> = (req, res) => { /* ... */ };
app.get("/users/:userId/orders/:orderId", h);   // orderId no longer visible on req.params
```

So: for inline handlers on literal paths, you often need no `Params` generic at all. For extracted named handlers, you must declare it. Mixing the two is the source of a lot of "why is `orderId` missing?" confusion.

Note the inference only works for the `Params` slot — `ReqBody` and `Query` can never be inferred from a path, so those always need explicit types.

### `express.json()` is what makes `req.body` exist at all

The `ReqBody` generic is a *claim*, not a runtime guarantee. If you forget `app.use(express.json())`, `req.body` is `undefined` and your typed destructure throws:

```ts
const create: RequestHandler<{}, unknown, CreateUserBody> = (req, res) => {
  const { email } = req.body;   // TypeError at runtime if express.json() isn't mounted
};
```

`@types/express` v4 types `req.body` as `ReqBody` with no `undefined`, which is optimistic. A defensive pattern for shared middleware:

```ts
function requireJsonBody<T>(req: Request<unknown, unknown, T>): T {
  if (req.body === undefined || req.body === null || typeof req.body !== "object") {
    throw ApiError.badRequest("Expected a JSON request body");
  }
  return req.body;
}
```

> **Express 5 difference:** in Express 5, `req.body` is `undefined` (not `{}`) when no body parser ran, and `@types/express@5` reflects that more honestly. Express 5 also removed the need for `body-parser` as a separate dependency (as did Express 4.16+).

### `req.query` is a getter in Express 5, and `ParsedQs` is recursive

The default `Query` type is `qs.ParsedQs`:

```ts
interface ParsedQs {
  [key: string]: undefined | string | string[] | ParsedQs | ParsedQs[];
}
```

It's recursive because the `extended` query parser supports nested brackets: `?filter[status][]=paid`. The union is genuinely what the runtime can produce, which is why narrowing helpers are mandatory rather than pedantic.

> **Express 5 difference:** `req.query` became a lazily-evaluated getter, and the default parser changed from `extended` to `simple`. `?filter[min]=10` yields the literal key `"filter[min]"` in Express 5 defaults. If you relied on nested query objects, set `app.set("query parser", "extended")`.

### `res.locals` is the fifth slot, and it's typed per-response

```ts
// Request<P, ResBody, ReqBody, Query, Locals> and Response<ResBody, Locals>
interface AppLocals { requestId: string; startedAt: number }

const withRequestId: RequestHandler<{}, unknown, unknown, {}, AppLocals> = (req, res, next) => {
  res.locals.requestId = randomUUID();   // ✅ typed
  res.locals.startedAt = Date.now();     // ✅ typed
  res.locals.typo = 1;                   // ❌ Error
  next();
};
```

The catch: `Locals` isn't propagated automatically between middleware — each handler declares its own. That makes `res.locals` a poor place for cross-cutting request state. `req.user` via declaration merging is the better tool, covered in **53 — Extending Express Request**.

### Compilation and DX traps

- **`esModuleInterop`** must be `true` for `import express from "express"` (Express uses `module.exports =`). Without it you need `import * as express from "express"`, which then breaks `express()` calls under some settings.
- **`"types"` in tsconfig is an allowlist, not an additive list.** Setting `"types": ["node"]` silently drops `@types/express` from the global scope. Either omit `types` entirely or list everything you need.
- **`skipLibCheck: true`** is near-mandatory in Express projects: `@types/express`, `@types/serve-static`, and `@types/express-serve-static-core` frequently disagree across minor versions, and errors inside `.d.ts` files you don't own are pure noise.
- **Version drift:** installing `express@5` with `@types/express@4` (or vice versa) produces baffling errors. Pin them together: `"express": "^4.19.2"` with `"@types/express": "^4.17.21"`, or `"express": "^5.1.0"` with `"@types/express": "^5.0.0"`.
- **Handlers get slower to type-check as generic depth grows.** `RouteParameters<"/a/:b/c/:d/e/:f">` is a recursive template-literal type; very long literal paths in a very large router file are a measurable `tsc` cost. Extracting `Params` interfaces sidesteps it.

---

## Common mistakes

### Mistake 1 — Putting the request body in the second generic slot

```ts
// ❌ WRONG — CreateUserBody landed in ResBody. req.body is still `any`, and
//    res.json() now demands a CreateUserBody. Everything is backwards.
const createUser = (req: Request<{}, CreateUserBody>, res: Response) => {
  req.body.email;              // any — no checking, typos silently pass
  res.json({ userId: 1 });     // ❌ Error: missing email/password/displayName
};
```

```ts
// ✅ RIGHT — the body is slot 3. Use `unknown` for slots you don't need.
const createUser = (
  req: Request<{}, ApiResponse<UserDto>, CreateUserBody>,
  res: Response<ApiResponse<UserDto>>,
) => {
  req.body.email;                                    // string ✅
  res.json({ ok: true, data: userDto });             // ✅
};

// ✅ Even better — declare it once with RequestHandler:
const createUser: RequestHandler<{}, ApiResponse<UserDto>, CreateUserBody> = (req, res) => {
  req.body.email;
  res.json({ ok: true, data: userDto });
};
```

### Mistake 2 — Treating `req.params` values as numbers

```ts
// ❌ WRONG — userId is a string; this comparison and this query are both broken.
const getUser: RequestHandler<{ userId: string }> = async (req, res) => {
  const userId = req.params.userId;
  if (userId === 42) { /* ❌ Error: comparison is unintentional, no overlap */ }
  const user = await userRepo.findById(userId);  // ❌ Error: string not assignable to number
  res.json(user);
};

// ❌ ALSO WRONG — lying to the compiler with an assertion:
const bad: RequestHandler<{ userId: number }> = async (req, res) => {
  //                              ^ params are never numbers at runtime
  await userRepo.findById(req.params.userId);   // compiles, passes "42" to the DB
};
```

```ts
// ✅ RIGHT — declare string, convert once, validate the conversion.
const getUser: RequestHandler<{ userId: string }, ApiResponse<UserDto>> = async (req, res) => {
  const userId = Number(req.params.userId);
  if (!Number.isInteger(userId) || userId <= 0) {
    res.status(400).json({ ok: false, error: "Invalid userId", code: "VALIDATION_FAILED" });
    return;
  }
  const user = await userRepo.findById(userId);   // ✅ number
  if (!user) {
    res.status(404).json({ ok: false, error: "User not found", code: "NOT_FOUND" });
    return;
  }
  res.json({ ok: true, data: toUserDto(user) });
};
```

### Mistake 3 — Unwrapped async handlers on Express 4

```ts
// ❌ WRONG — Express 4 does not catch the rejection. The client hangs until timeout,
//    the error handler never runs, and Node logs an unhandled rejection.
app.get("/orders/:orderId", async (req: Request<{ orderId: string }>, res: Response) => {
  const order = await orderRepo.findById(req.params.orderId);   // may throw
  if (!order) throw ApiError.notFound("Order");                 // definitely throws
  res.json(order);
});
```

```ts
// ✅ RIGHT (option A) — wrap it so rejections reach next():
app.get(
  "/orders/:orderId",
  asyncHandler<{ orderId: string }, OrderDto, unknown, {}>(async (req, res) => {
    const order = await orderRepo.findById(req.params.orderId);
    if (!order) throw ApiError.notFound("Order");
    res.json({ ok: true, data: toOrderDto(order) });
  }),
);

// ✅ RIGHT (option B) — try/catch inside, forwarding to next() manually:
app.get("/orders/:orderId", async (
  req: Request<{ orderId: string }>,
  res: Response,
  next: NextFunction,
): Promise<void> => {
  try {
    const order = await orderRepo.findById(req.params.orderId);
    if (!order) throw ApiError.notFound("Order");
    res.json(order);
  } catch (err) {
    next(err);          // ← the piece everyone forgets
  }
});

// ✅ RIGHT (option C) — upgrade to Express 5, which awaits handlers and
//    forwards rejections to the error middleware automatically.
```

### Mistake 4 — A three-parameter error handler

```ts
// ❌ WRONG — Express identifies error middleware by fn.length === 4.
//    With three params this is registered as ordinary middleware: `err` receives
//    the Request object, `req` receives the Response, and nothing works.
app.use((err: Error, req: Request, res: Response) => {
  res.status(500).json({ error: err.message });
});
```

```ts
// ✅ RIGHT — four parameters, typed via ErrorRequestHandler, `next` kept even if unused.
const errorHandler: ErrorRequestHandler = (err, req, res, next): void => {
  if (res.headersSent) { next(err); return; }
  const status = err instanceof ApiError ? err.statusCode : 500;
  res.status(status).json({ ok: false, error: err.message ?? "Internal Server Error" });
};
app.use(errorHandler);   // registered after every route
```

### Mistake 5 — Forgetting `return` after sending a response

```ts
// ❌ WRONG — no return, so execution continues and Express throws ERR_HTTP_HEADERS_SENT.
const createUser: RequestHandler<{}, ApiResponse<UserDto>, CreateUserBody> = async (req, res) => {
  if (!req.body.email) {
    res.status(422).json({ ok: false, error: "email required", code: "VALIDATION_FAILED" });
  }
  const user = await userRepo.create(req.body);   // runs even on the invalid path
  res.status(201).json({ ok: true, data: toUserDto(user) });   // second send → crash
};
```

```ts
// ✅ RIGHT — every send is followed by an explicit `return;`.
const createUser: RequestHandler<{}, ApiResponse<UserDto>, CreateUserBody> = async (req, res) => {
  if (!req.body.email) {
    res.status(422).json({ ok: false, error: "email required", code: "VALIDATION_FAILED" });
    return;
  }
  const user = await userRepo.create(req.body);
  res.status(201).json({ ok: true, data: toUserDto(user) });
};
```

### Mistake 6 — Forgetting `mergeParams` on a nested router

```ts
// ❌ WRONG — the type says orgId exists; at runtime it's undefined.
const memberRouter = Router();                       // no mergeParams
const listMembers: RequestHandler<{ orgId: string }> = (req, res) => {
  const orgId = req.params.orgId;                    // undefined at runtime ⚠️
  res.json({ orgId });
};
memberRouter.get("/", listMembers);
app.use("/orgs/:orgId/members", memberRouter);
```

```ts
// ✅ RIGHT — mergeParams makes the parent's params visible on the child router.
const memberRouter = Router({ mergeParams: true });
const listMembers: RequestHandler<{ orgId: string }> = (req, res) => {
  const orgId = req.params.orgId;                    // "acme" ✅
  res.json({ orgId });
};
memberRouter.get("/", listMembers);
app.use("/orgs/:orgId/members", memberRouter);
```

---

## Practice exercises

### Exercise 1 — easy

Build a fully typed "products" route module from scratch. You must define, without copying from the examples above:

1. A `ProductDto` interface: `{ productId: number; sku: string; name: string; priceCents: number; inStock: boolean }`.
2. A `ProductIdParams` interface for the route `GET /products/:productId`.
3. A `ListProductsQuery` interface supporting `?search=`, `?minPrice=`, `?maxPrice=` — remember what type query values really are.
4. A `CreateProductBody` interface for `POST /products`.

Then write three handlers using `RequestHandler` (not per-argument annotations):

- `listProducts` — returns `ProductDto[]`, applies the query filters after converting them.
- `getProduct` — converts `productId` from string to number, returns 400 on a non-integer, 404 when missing.
- `createProduct` — validates that `priceCents` is a positive integer, responds `201` with the created `ProductDto`.

Every handler must annotate its return type explicitly, and every response must be followed by an explicit `return;`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed pagination + response-envelope layer for an Express 4 API.

1. Define `type ApiResponse<T>` as a discriminated union with an `ok: true` branch carrying `data: T` and `meta: PageMeta`, and an `ok: false` branch carrying `error: string` and `code`.
2. Define `PageMeta` as `{ page: number; perPage: number; total: number; totalPages: number; hasNext: boolean }`.
3. Write `parsePagination(query: { page?: string; perPage?: string }): { page: number; perPage: number; offset: number }` that clamps `page >= 1` and `1 <= perPage <= 100`, and defaults to page 1 / perPage 20.
4. Write a generic `sendPage<T>(res: Response<ApiResponse<T[]>>, rows: T[], total: number, page: number, perPage: number): void` helper that computes `PageMeta` and sends it.
5. Write an `asyncHandler` wrapper that is generic over all four `Request` slots (Params, ResBody, ReqBody, Query) and does not erase them.
6. Use all of the above to implement `GET /users/:orgId/members?page=&perPage=&role=` on a router created with `mergeParams: true`, where `role` is `"admin" | "member" | undefined` after narrowing.

Add a comment on each `Request` generic slot naming which of `params`/`body`/`query`/response it controls.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **type-safe route registry** — a thin layer over Express where a route's params, body, query and response are declared once in a schema object, and the handler's types are derived from it.

Requirements:

1. Define a `RouteDefinition` type carrying `method`, `path`, and four phantom type slots (`Params`, `Query`, `Body`, `Response`).
2. Write `defineRoute<Params, Query, Body, Res>(def: { method: "get" | "post" | "patch" | "delete"; path: string; handler: (ctx: { params: Params; query: Query; body: Body }) => Promise<Res> })` that returns a `RouteDefinition`. The handler must receive a **plain context object**, never `req`/`res` directly — the goal is that domain code never touches Express types.
3. Write `registerRoutes(router: Router, routes: RouteDefinition[]): void` that adapts each definition into a real `RequestHandler`, wraps it for async rejection safety, and sends `{ ok: true, data: <handler result> }` with the right status (`201` for `post`, `200` otherwise).
4. Make thrown `ApiError`s flow to a single `ErrorRequestHandler` with four parameters.
5. Add a `validate` option per route: `{ params?: Validator<Params>; query?: Validator<Query>; body?: Validator<Body> }` where `Validator<T>` is `{ parse(input: unknown): T }`. When present, run it before the handler and convert failures into a 400.
6. Prove the types work by defining three routes — `GET /orders/:orderId`, `GET /orders?status=`, `POST /orders` — where `params.orderId` is a `number` (converted by the validator), `query.status` is a union literal, and `body` is a `CreateOrderBody`. A handler that returns the wrong shape must be a compile error.

Finally, write a short comment explaining what would change if the app were migrated from Express 4 to Express 5.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Install ────────────────────────────────────────────────────────────────────
// npm i express && npm i -D typescript @types/express @types/node

// ── Imports ────────────────────────────────────────────────────────────────────
import express, { Router } from "express";
import type {
  Request, Response, NextFunction,
  RequestHandler, ErrorRequestHandler, Application,
} from "express";

// ── The four slots (order matters!) ────────────────────────────────────────────
Request<Params, ResBody, ReqBody, Query, Locals>
RequestHandler<Params, ResBody, ReqBody, Query>
Response<ResBody, Locals>

// ── Typical handler ────────────────────────────────────────────────────────────
const getUser: RequestHandler<{ userId: string }, ApiResponse<UserDto>> = async (req, res) => {
  const userId = Number(req.params.userId);              // params are ALWAYS strings
  const user = await userRepo.findById(userId);
  if (!user) { res.status(404).json({ ok: false, error: "Not found", code: "NOT_FOUND" }); return; }
  res.json({ ok: true, data: toUserDto(user) });
};

// ── Async safety (Express 4) ───────────────────────────────────────────────────
const asyncHandler = <P, R, B, Q>(
  fn: (req: Request<P, R, B, Q>, res: Response<R>, next: NextFunction) => Promise<void>,
): RequestHandler<P, R, B, Q> => (req, res, next) => { Promise.resolve(fn(req, res, next)).catch(next); };

// ── Error handler — FOUR params ────────────────────────────────────────────────
const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  if (res.headersSent) { next(err); return; }
  res.status(500).json({ ok: false, error: "Internal Server Error" });
};

// ── Router ─────────────────────────────────────────────────────────────────────
const router: Router = Router({ mergeParams: true });   // mergeParams for nested params
router.get("/:userId", getUser);
app.use("/api/users", router);
```

| Thing | Type | Note |
|---|---|---|
| `req.params` | slot 1 — `Params` | Values are **always** `string`; convert + validate |
| `res.json()` body | slot 2 — `ResBody` | Also types `Response<ResBody>` |
| `req.body` | slot 3 — `ReqBody` | Requires `express.json()`; type is a claim, not a guarantee |
| `req.query` | slot 4 — `Query` | Default `ParsedQs`: `string \| string[] \| ParsedQs \| ...` |
| `res.locals` | slot 5 — `Locals` | Not propagated between middleware; prefer `req.user` |
| Handler function | `RequestHandler<P, ResBody, ReqBody, Q>` | Declares all slots in one place |
| Error middleware | `ErrorRequestHandler` | Must have **4** params; register last |
| `next` | `NextFunction` | `next()`, `next(err)`, `next("route")`, `next("router")` |
| Async return | `Promise<void>` | Express 4 ignores rejections — wrap with `asyncHandler` |
| Nested params | `Router({ mergeParams: true })` | Types alone won't populate parent params |
| Express 5 | `@types/express@5` | Awaits handlers; `req.query` is a getter; simple query parser |

---

## Connected topics

- **42 — Discriminated unions** — the `ApiResponse<T>` success/error envelope that makes typed `res.json` usable for both branches.
- **53 — Extending Express Request** — how to add `req.user`, `req.requestId` and other custom properties via declaration merging.
- **26 — What are generics** and **30 — Generic constraints** — the machinery behind `Request<Params, ResBody, ReqBody, Query>` and behind writing a generic `asyncHandler` that doesn't erase types.
- **20 — Function types** — `RequestHandler` and `ErrorRequestHandler` are just named function types; contextual typing is why annotating the handler types its parameters.
- **04 — TypeScript Node.js project setup** — `esModuleInterop`, `skipLibCheck`, and the `types` allowlist that decide whether `@types/express` loads at all.
