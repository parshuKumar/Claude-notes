# 51 — Module augmentation

## What is this?

**Module augmentation** is the mechanism by which you reach into a module you do not own — `express`, `jsonwebtoken`, `socket.io`, `@types/node` — and **add** to the types it already declares, without forking it, without patching `node_modules`, and without casting at every call site.

```ts
// src/types/express.d.ts
import "express-serve-static-core";           // load the module you are augmenting

declare module "express-serve-static-core" {  // re-open it
  interface Request {                          // merge into its existing Request
    user?: AuthenticatedUser;                  // ← the property you are adding
  }
}

export interface AuthenticatedUser {
  userId: number;
  email:  string;
  roles:  readonly ("admin" | "member" | "billing")[];
}
```

From that moment, every file in your project sees `req.user` as `AuthenticatedUser | undefined` — in your middleware, in your route handlers, in your tests, in the router of a package you imported. You changed a type you did not write.

Three facts define module augmentation:

1. **It is built on declaration merging.** Two `interface Request` declarations with the same name in the same declaration space become *one* interface with the union of their members. Augmentation is just declaration merging performed across module boundaries on purpose.
2. **It only ever adds.** You can add new members to an interface, new overloads to a function, new members to a namespace. You **cannot** change the type of a member that already exists, and you cannot remove anything. TypeScript will reject the attempt with `TS2717`.
3. **It is global to the compilation.** An augmentation is not scoped to the file that declares it. If any file in the program contains it, the whole program — including files that never import your augmentation — sees the change. That is what makes it powerful, and exactly what makes it dangerous.

## Why does it matter?

Almost every non-trivial Node backend needs it within the first week.

- **`req.user`.** Authentication middleware attaches the decoded user to the request. `@types/express` has no idea your app has users. Without augmentation, every handler that reads `req.user` either casts (`(req as any).user`) or invents a parallel `AuthedRequest` type that fights the framework's own handler signatures.
- **`process.env`.** Node types `process.env` as `Record<string, string | undefined>`. Every config read is `string | undefined`, every typo (`DATBASE_URL`) silently compiles and yields `undefined`, and you discover it when the pod crashes on boot. Augmenting `NodeJS.ProcessEnv` turns your environment contract into a checked type.
- **Third-party payload shapes.** `jsonwebtoken`'s `JwtPayload` is deliberately loose. `socket.io` gives you `socket.data` typed as `any` unless you tell it otherwise. `fastify`'s `FastifyRequest`, `koa`'s `Context`, `passport`'s `Express.User` — every one of these frameworks *expects* you to augment.
- **Plugin ecosystems.** When a library says "this plugin adds `app.redis`", the only way that becomes visible to `tsc` is that the plugin's own `.d.ts` augments the host library's interface. If you write internal plugins, you write augmentations.

The alternative to augmentation is casting. Casting is not a smaller version of augmentation — it is the total absence of checking at exactly the point where your security-critical data (the authenticated user, the token payload, the environment secrets) enters your code. Augmentation is how you type the seams of your application.

---

## The JavaScript way vs the TypeScript way

Start with the pain, because it is the pain you have already lived through.

```js
// ═══ The JavaScript way — Express auth middleware, plain Node ═════════════════
// src/middleware/authenticate.js
const jwt = require("jsonwebtoken");

function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader) return res.status(401).json({ error: "Missing token" });

  try {
    const payload = jwt.verify(authHeader.slice(7), process.env.JWT_SECRET);
    req.user = {                       // ← we just invented a property on Request
      userId: payload.sub,
      email:  payload.email,
      roles:  payload.roles,
    };
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
}
```

```js
// src/routes/orders.js — a different file, written by a different person, 3 months later
router.get("/orders", authenticate, async (req, res) => {
  const orders = await orderRepo.listForUser(req.user.id);      // ❌ it's `userId`, not `id`
  //                                              ^ undefined → listForUser(undefined)
  //                                                → SELECT ... WHERE user_id IS NULL
  //                                                → returns every orphaned row in the table

  if (req.user.roles.includes("adming")) {                      // ❌ typo → always false
    return res.json({ orders, internal: true });                //    admins silently lose a feature
  }

  res.json({ orders });
});

router.get("/profile", async (req, res) => {                    // ❌ forgot `authenticate`
  res.json({ email: req.user.email });                          // 💥 TypeError: Cannot read
  //                                                            //    properties of undefined
});
```

Three separate production bugs, none of which JavaScript can see:

- `req.user.id` is `undefined`, and a query with `WHERE user_id IS NULL` is a **data leak**, not a crash.
- `"adming"` is a typo in a string literal — `roles.includes` happily returns `false` forever.
- `/profile` has no `authenticate` middleware, so `req.user` is `undefined` and the route 500s the first time an unauthenticated request hits it — usually in production, because your tests always send a token.

Now, TypeScript **without** augmentation. It does not get better on its own:

```ts
// ❌ TypeScript, no augmentation — the compiler agrees with @types/express
import type { Request, Response, NextFunction } from "express";

export function authenticate(req: Request, res: Response, next: NextFunction): void {
  req.user = { userId: 42, email: "a@b.c", roles: ["member"] };
  //  ^^^^ TS2339: Property 'user' does not exist on type 'Request<...>'.
}
```

So people reach for the two bad escapes:

```ts
// ❌ Escape 1 — cast at every use. Now you have zero checking, everywhere, forever.
(req as any).user = { userId: 42 };
const userId = (req as any).user.id;          // still the wrong property, still silent
                                              // and now the cast is copy-pasted into 60 files

// ❌ Escape 2 — a parallel request type. Looks principled; fights the framework.
interface AuthedRequest extends Request {
  user: AuthenticatedUser;
}

router.get("/orders", authenticate, (req: AuthedRequest, res: Response) => { /* ... */ });
//                                  ^^^ TS2769: No overload matches this call.
//   express's RequestHandler expects (req: Request, ...) — AuthedRequest is not assignable
//   to the parameter position because parameters are checked bivariantly-but-not-always,
//   and every generic (params/body/query) now has to be threaded manually.
// You end up writing `as unknown as RequestHandler` — a cast with extra steps.
```

Now the TypeScript way. **One file, eleven lines, written once:**

```ts
// ✅ src/types/express.d.ts
import "express-serve-static-core";

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}

export interface AuthenticatedUser {
  userId: number;
  email:  string;
  roles:  readonly ("admin" | "member" | "billing")[];
}
```

Every one of the original bugs is now a compile error, in the *original* code, with no changes to the call sites:

```ts
// ✅ src/routes/orders.ts — untouched shape, fully checked
import { Router, type Request, type Response } from "express";
import { authenticate } from "../middleware/authenticate.js";

const router = Router();

router.get("/orders", authenticate, async (req: Request, res: Response) => {
  const orders = await orderRepo.listForUser(req.user.id);
  //                                         ^^^^^^^^ TS18048: 'req.user' is possibly 'undefined'.
  //                                                  ^^ TS2339: Property 'id' does not exist on
  //                                                     type 'AuthenticatedUser'.
  //                                                     Did you mean 'userId'?

  if (req.user.roles.includes("adming")) { /* ... */ }
  //                          ^^^^^^^^ TS2345: Argument of type '"adming"' is not assignable to
  //                                   parameter of type '"admin" | "member" | "billing"'.
});

// ✅ And the forgotten-middleware bug becomes visible too, because `user` is OPTIONAL:
router.get("/profile", async (req: Request, res: Response) => {
  res.json({ email: req.user.email });
  //                ^^^^^^^^ TS18048: 'req.user' is possibly 'undefined'.
});

// ✅ The fix the compiler forces you into is the correct one:
router.get("/profile", authenticate, async (req: Request, res: Response) => {
  if (!req.user) return res.status(401).json({ error: "Unauthenticated" });
  res.json({ email: req.user.email });   // ✅ narrowed to AuthenticatedUser
});
```

That is the whole satisfaction of augmentation: you did not restructure your app, you did not wrap the framework, you did not cast. You told the compiler one true fact about a type you don't own, and it propagated to every file in the program.

---

## Syntax

```ts
// ═══ 1. AUGMENTING A MODULE — the canonical form ═══════════════════════════════
// The file MUST be a module (must have a top-level import or export) for
// `declare module "x"` to mean "augment x" rather than "declare a new module x".

import "express-serve-static-core";          // ← the import that makes this file a module
                                             //   AND loads the target module's declarations

declare module "express-serve-static-core" { // ← specifier must resolve to a REAL module
  interface Request {                        // ← merges with the existing Request
    requestId: string;                       // ← new member: legal
  }
  interface Response {
    sendApiError(statusCode: number, code: string): void;   // new method: legal
  }
}


// ═══ 2. AUGMENTING A GLOBAL NAMESPACE — `declare global` ═══════════════════════
export {};                                   // ← makes the file a module (no import needed)

declare global {
  namespace NodeJS {
    interface ProcessEnv {                   // merges with @types/node's ProcessEnv
      DATABASE_URL: string;
      JWT_SECRET:   string;
      NODE_ENV:     "development" | "test" | "production";
      PORT?:        string;
    }
  }

  // Adding a genuinely global value (rare, but this is how):
  // eslint-disable-next-line no-var
  var requestContext: { requestId: string; userId?: number } | undefined;
  //  ^^^ must be `var` — `let`/`const` do not create properties on globalThis
}


// ═══ 3. AUGMENTING INSIDE A .ts FILE (not a .d.ts) — same syntax ═══════════════
// Any module file can carry an augmentation. Common for library plugins.
import { FastifyInstance } from "fastify";

declare module "fastify" {
  interface FastifyInstance {
    redis: RedisClient;
  }
}


// ═══ 4. WHAT YOU MAY ADD ═══════════════════════════════════════════════════════
declare module "some-lib" {
  interface ExistingInterface { newProp: string }     // ✅ new interface member
  namespace ExistingNamespace { const NEW: number }   // ✅ new namespace member
  interface BrandNewInterface { a: number }           // ✅ a wholly new exported type
  type BrandNewAlias = string;                        // ✅ a wholly new type alias
  function existingFn(x: number): void;               // ✅ a new OVERLOAD of a function
}

// ═══ 5. WHAT YOU MAY NOT ADD ═══════════════════════════════════════════════════
declare module "some-lib" {
  interface ExistingInterface { existingProp: number } // ❌ TS2717 if the type differs
  export default class NewDefault {}                   // ❌ TS2669: cannot add default export
  export const NEW_VALUE: string;                      // ⚠️  legal syntax, but the value does
                                                       //     NOT exist at runtime — you lied
}


// ═══ 6. TSCONFIG — making sure tsc actually loads the augmentation ═════════════
// {
//   "compilerOptions": {
//     "strict":        true,
//     "types":         ["node"],       // ⚠️ this does NOT control your own .d.ts files
//     "typeRoots":     ["./node_modules/@types"]
//   },
//   "include": ["src/**/*.ts"]         // ← must match src/types/express.d.ts
// }

// Prove it loaded:
//   npx tsc --noEmit --listFiles | grep 'src/types'
```

---

## How it works — concept by concept

### Concept 1 — Declaration merging is the engine

Before module augmentation makes sense, you need the rule it is built on: **two declarations with the same name in the same declaration space merge.**

```ts
// ── Interfaces merge. This is legal and produces ONE interface. ───────────────
interface OrderRecord {
  orderId:    string;
  userId:     number;
}

interface OrderRecord {
  totalCents: number;
  createdAt:  Date;
}

// The compiler now sees a single type:
const order: OrderRecord = {
  orderId:    "ord_9f2",
  userId:     41,
  totalCents: 12_50,
  createdAt:  new Date(),
};   // ✅ all four members required
```

```ts
// ── Type ALIASES do not merge. This is a hard error. ─────────────────────────
type PaymentRecord = { paymentId: string };
type PaymentRecord = { amountCents: number };
//   ^^^^^^^^^^^^^ TS2300: Duplicate identifier 'PaymentRecord'.
```

This single asymmetry explains a design decision you have probably wondered about: **why do library authors declare public shapes as `interface` and not `type`?** Because `interface` is the only form that consumers can augment. `@types/express` uses `interface Request` precisely so you can add `user` to it. If it were `type Request = {...}`, module augmentation of Express would be impossible.

Four things merge in TypeScript:

| Declaration | Merges with | Result |
|---|---|---|
| `interface` | `interface` | Union of members (conflicting members must match exactly) |
| `namespace` | `namespace` | Union of exported members |
| `namespace` | `function` / `class` / `enum` | Static-side properties added to the value |
| `enum` | `enum` | Union of members (all but one must have initializers) |

Module augmentation is exactly rule 1 and rule 2 applied **across module boundaries**. `declare module "express-serve-static-core" { interface Request { ... } }` says "put this interface declaration into that module's declaration space, where it will merge with the one already there."

```ts
// ── The merge rule for conflicting members ────────────────────────────────────
interface SessionRecord { expiresAt: Date }
interface SessionRecord { expiresAt: Date }        // ✅ identical type → fine, merges
interface SessionRecord { expiresAt: number }      // ❌ TS2717: Subsequent property
//                        ^^^^^^^^^                   declarations must have the same type.
//                                                    Property 'expiresAt' must be of type
//                                                    'Date', but here has type 'number'.
```

Remember `TS2717`. It is the error you will see when you try to *change* something instead of *add* something, and it is the error you will see when two copies of `@types/node` end up in your dependency tree.

### Concept 2 — Why the file must be a module (the single biggest trap)

`declare module "x" { ... }` means **two completely different things** depending on whether the enclosing file is a module or a script.

TypeScript's rule: **a file is a module if and only if it contains a top-level `import` or `export`.** Otherwise it is a *global script*.

```ts
// ── CASE A: file has NO top-level import/export → it is a SCRIPT ─────────────
// src/types/express.d.ts

declare module "express-serve-static-core" {
  interface Request {
    user?: { userId: number };
  }
}

// ⚠️ This does NOT augment express-serve-static-core.
// In a script file, `declare module "x"` is an AMBIENT MODULE DECLARATION:
// "there exists a module named 'express-serve-static-core' and here is its ENTIRE type."
// It SHADOWS the real one. Result: express-serve-static-core now has exactly one
// exported thing — an interface Request with only `user`. Everything else vanishes.
//
// Symptom: `import express from "express"` still works (different specifier),
// but `req.params`, `req.body`, `req.headers` all become errors, or the whole
// module silently resolves to `any`. Deeply confusing.
```

```ts
// ── CASE B: file HAS a top-level import/export → it is a MODULE ──────────────
// src/types/express.d.ts

import "express-serve-static-core";      // ← this one line changes everything

declare module "express-serve-static-core" {
  interface Request {
    user?: { userId: number };
  }
}

// ✅ Now `declare module` means AUGMENT: merge these declarations into the
//    existing module. Everything @types/express declared is still there.
```

```ts
// ── CASE C: `export {}` also works, when you have nothing to import ──────────
// src/types/env.d.ts
export {};                                // ← the "make this a module" idiom

declare global {
  namespace NodeJS {
    interface ProcessEnv { DATABASE_URL: string }
  }
}
// Without `export {}`, the file is already a script — so `declare global` is
// redundant AND an error: TS2669 "Augmentations for the global scope can only be
// directly nested in external modules or ambient module declarations."
```

The rule in one line:

> **To augment, the file must be a module. To declare a brand-new module that doesn't exist, the file must be a script.**

Two different jobs, one keyword, opposite requirements. This is the #1 cause of "my augmentation doesn't work" and "my augmentation broke everything."

Import forms that make a file a module — pick whichever is honest:

```ts
import "express-serve-static-core";                    // side-effect import: loads + marks module
import type { Request } from "express";                // type-only import: also marks module ✅
import { Router } from "express";                      // value import: marks module
export {};                                             // empty export: marks module
export type AuthenticatedUser = { userId: number };    // any export: marks module
```

> ⚠️ One subtlety: `import type` marks the file as a module, but under `verbatimModuleSyntax` a type-only import is fully erased — that's fine for `.d.ts` files (nothing is emitted anyway) but in a `.ts` file it means no runtime import happens. For augmentation purposes, either form works; `import "x";` is the clearest signal of intent.

### Concept 3 — The module specifier must resolve to a real module

`declare module "some-string"` in a module file does not fuzzy-match. The string is resolved with the **exact same module resolution** as an `import` from that file. If it resolves to nothing, you have not augmented anything — you have silently created a *new* ambient module (or gotten an error, depending on config).

```ts
// ❌ WRONG — "express" is a re-export barrel; Request does not live there
import "express";

declare module "express" {
  interface Request {
    user?: AuthenticatedUser;      // creates a NEW, unrelated `Request` interface
  }                                //  inside the express module. req.user still errors.
}
```

Why? Look at what `@types/express` actually does:

```ts
// node_modules/@types/express/index.d.ts (abridged)
import * as core from "express-serve-static-core";

declare namespace e {
  export interface Request<P = core.ParamsDictionary, ResBody = any, ReqBody = any>
    extends core.Request<P, ResBody, ReqBody> {}
  // ↑ express's `Request` is an EMPTY interface EXTENDING the real one
}
```

`express`'s `Request` merely **extends** `express-serve-static-core`'s `Request`. Augmenting `express` adds a member to the empty derived interface — which is a *different* interface from the one Express's own `RequestHandler` signature refers to. So your handler parameter, typed by Express as `core.Request`, never sees your member.

```ts
// ✅ RIGHT — augment where the interface is actually declared
import "express-serve-static-core";

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
```

**How to find the right specifier, every time:**

```bash
# 1. Ctrl-click / "Go to definition" on the type in your editor. Look at the FILE PATH.
#    node_modules/@types/express-serve-static-core/index.d.ts  →  specifier is
#    "express-serve-static-core"

# 2. Or grep for the declaring interface:
grep -rn "interface Request" node_modules/@types/express-serve-static-core/index.d.ts

# 3. Then confirm tsc agrees the specifier resolves:
npx tsc --noEmit --traceResolution 2>&1 | grep -A3 "express-serve-static-core"
```

Common real-world specifiers on a Node backend:

| You want to augment | Correct specifier | Notes |
|---|---|---|
| Express `req` / `res` | `"express-serve-static-core"` | not `"express"` |
| Express `Express.User` (passport) | `declare global { namespace Express { ... } }` | passport uses the global namespace |
| `process.env` | `declare global { namespace NodeJS { ... } }` | global, not a module |
| Fastify request/instance | `"fastify"` | fastify declares them directly |
| Koa context | `"koa"` | |
| `jsonwebtoken` payload | `"jsonwebtoken"` | interface `JwtPayload` |
| socket.io `socket.data` | prefer **generics**, not augmentation | see Concept 6 |
| `@types/node` globals | `declare global { ... }` | |

### Concept 4 — Augmenting `NodeJS.ProcessEnv`

`process.env` is the highest-value augmentation in a Node backend, because it is where untyped strings become your entire runtime configuration.

Here is what `@types/node` gives you:

```ts
// node_modules/@types/node/process.d.ts (abridged)
declare global {
  namespace NodeJS {
    interface ProcessEnv extends Dict<string> {}
    //                            ^^^^^^^^^^^ index signature: [key: string]: string | undefined
  }
  var process: NodeJS.Process;
}
```

So out of the box:

```ts
const databaseUrl = process.env.DATABASE_URL;   // string | undefined
const port        = process.env.PORT;           // string | undefined
const typo        = process.env.DATBASE_URL;    // string | undefined — NO ERROR. Index signature.
```

The index signature is the problem: **every key is valid**, so typos are invisible. Augment it:

```ts
// ── src/types/env.d.ts ───────────────────────────────────────────────────────
export {};                                   // ← module marker (Concept 2)

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      // Required — the process must not boot without these.
      NODE_ENV:         "development" | "test" | "production";
      DATABASE_URL:     string;
      JWT_SECRET:       string;
      REDIS_URL:        string;

      // Optional — has a sensible default in code.
      PORT?:            string;
      LOG_LEVEL?:       "debug" | "info" | "warn" | "error";
      SENTRY_DSN?:      string;
      AWS_REGION?:      string;
    }
  }
}
```

```ts
// ── src/config.ts — what you get ─────────────────────────────────────────────
const nodeEnv     = process.env.NODE_ENV;      // "development" | "test" | "production"  ✅
const databaseUrl = process.env.DATABASE_URL;  // string  ✅ — no `| undefined`
const port        = process.env.PORT;          // string | undefined  ✅
const typo        = process.env.DATBASE_URL;   // string | undefined — still no error ⚠️

if (process.env.NODE_ENV === "prod") { }
//                          ^^^^^^ TS2367: This comparison appears unintentional because the
//                                 types have no overlap. ✅ literal union catches it
```

Two things to understand deeply here.

**(a) The typo is still not caught, and cannot be.** `ProcessEnv extends Dict<string>` carries an index signature, and merging your interface members into it does not remove that index signature. Any string key remains legal. You get autocomplete and literal-union checking for the keys you declared; you do not get "unknown key" errors. That is a hard limit of this technique.

**(b) `DATABASE_URL: string` is a *lie you choose to tell*.** Node will happily give you `undefined` if the variable is unset. You have declared it non-optional. The declaration is only true if you *make* it true, at boot, with a runtime check:

```ts
// ── src/config.ts — the honest pattern: augment + validate at startup ────────
const REQUIRED_ENV_VARS = [
  "NODE_ENV",
  "DATABASE_URL",
  "JWT_SECRET",
  "REDIS_URL",
] as const satisfies readonly (keyof NodeJS.ProcessEnv)[];
//    ^^^^^^^^^ `satisfies` ties the runtime list to the augmented type:
//              add a key to the list that isn't declared → compile error.

export function assertEnv(): void {
  const missing = REQUIRED_ENV_VARS.filter((key) => !process.env[key]);
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(", ")}`);
  }
}

assertEnv();                                   // run this FIRST, before any other import side-effect

export const config = {
  nodeEnv:     process.env.NODE_ENV,           // "development" | "test" | "production"
  databaseUrl: process.env.DATABASE_URL,       // string
  jwtSecret:   process.env.JWT_SECRET,         // string
  port:        Number(process.env.PORT ?? 3000),
  logLevel:    process.env.LOG_LEVEL ?? "info",
} as const;
```

The augmentation gives you *compile-time* names and literal types. `assertEnv()` makes the augmentation *true*. Neither alone is sufficient; together they are the standard professional setup. (A schema library like `zod` can do both jobs at once — see Going Deeper.)

### Concept 5 — Augmenting Express: the full, correct pattern

This is the one you will actually write. There are three decisions worth making deliberately.

**Decision 1 — optional or required `user`?**

```ts
// ── Option A: optional (recommended) ─────────────────────────────────────────
declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
// ✅ Truthful: a Request genuinely might not have gone through auth middleware.
// ✅ Forces a null check at every use → the "forgot the middleware" bug is caught.
// ⚠️ Costs you a guard in every authenticated handler.

// ── Option B: required ───────────────────────────────────────────────────────
declare module "express-serve-static-core" {
  interface Request {
    user: AuthenticatedUser;      // no `?`
  }
}
// ✅ No guards needed. Reads nicely.
// ❌ A LIE for every unauthenticated route — /health, /login, /webhooks/stripe.
// ❌ req.user.userId on an unauthenticated route compiles and throws at runtime.
```

Use **Option A**. The extra guard is three characters and it is the guard that catches the missing-middleware class of bug. If the ceremony bothers you, encode the narrowing once in a helper:

```ts
// ── src/http/require-user.ts — narrow once, reuse everywhere ─────────────────
import type { Request } from "express";
import { ApiError } from "../errors.js";

/** Type predicate: narrows Request to one that definitely has a user. */
export function isAuthenticated(
  req: Request,
): req is Request & { user: AuthenticatedUser } {
  return req.user !== undefined;
}

/** Throws a 401 if unauthenticated; returns a non-optional user otherwise. */
export function requireUser(req: Request): AuthenticatedUser {
  if (!req.user) throw ApiError.unauthorized("Authentication required");
  return req.user;                             // ✅ narrowed to AuthenticatedUser
}
```

```ts
// ── Usage — one line, no casts, fully typed ─────────────────────────────────
router.get("/orders", authenticate, async (req, res) => {
  const user = requireUser(req);               // AuthenticatedUser
  const orders = await orderRepo.listForUser(user.userId);
  res.json({ orders });
});
```

**Decision 2 — where the `AuthenticatedUser` type comes from.**

Two styles, both correct, with different tradeoffs:

```ts
// ── Style 1: import the type into the augmentation file ─────────────────────
// src/types/express.d.ts
import type { AuthenticatedUser } from "../auth/authenticated-user.js";
// ⚠️ This import makes the file a module, so `declare module` = augment. Good.
// ⚠️ BUT: inside `declare module`, names resolve in the OUTER file scope, so this works.

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}

// ── Style 2: declare the type inline and export it from the augmentation file ─
// src/types/express.d.ts
import "express-serve-static-core";

export interface AuthenticatedUser {
  userId: number;
  email:  string;
  roles:  readonly AuthRole[];
}
export type AuthRole = "admin" | "member" | "billing";

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
```

Style 1 keeps the domain type where the domain lives; Style 2 keeps the whole contract in one file. Style 1 scales better once `AuthenticatedUser` is used by services, repositories, and tests — you do not want your domain model living in a `.d.ts`.

**Decision 3 — the full augmentation for a real service.**

```ts
// ── src/types/express.d.ts ──────────────────────────────────────────────────
import type { AuthenticatedUser } from "../auth/authenticated-user.js";
import type { Logger } from "pino";

declare module "express-serve-static-core" {
  interface Request {
    /** Set by `authenticate` middleware. Undefined on public routes. */
    user?: AuthenticatedUser;

    /** Set by `requestId` middleware. Always present after it runs. */
    requestId: string;

    /** Child logger pre-bound with requestId and userId. */
    log: Logger;

    /** Monotonic timestamp captured at the start of the request. */
    startedAtMs: number;
  }

  interface Response {
    /** Convenience wrapper that serialises an ApiError with the requestId. */
    sendApiError(error: unknown): void;
  }

  interface Locals {
    /** Populated by the tenancy middleware from the subdomain. */
    tenantId?: string;
  }
}
```

Note `requestId: string` is **required** while `user?` is optional. That is a deliberate statement: the requestId middleware is mounted globally at the top of the stack, so it genuinely always runs; auth is per-route, so it genuinely might not.

### Concept 6 — Augmenting third-party libraries: `jsonwebtoken` and `socket.io`

**`jsonwebtoken`.** `jwt.verify()` returns `string | JwtPayload`, and `JwtPayload` is the loose set of registered claims plus an index signature:

```ts
// node_modules/@types/jsonwebtoken/index.d.ts (abridged)
export interface JwtPayload {
  [key: string]: any;
  iss?: string;
  sub?: string;
  aud?: string | string[];
  exp?: number;
  nbf?: number;
  iat?: number;
  jti?: string;
}
```

Because of `[key: string]: any`, `payload.rolez` compiles and is `any`. You can augment it:

```ts
// ── src/types/jsonwebtoken.d.ts ─────────────────────────────────────────────
import "jsonwebtoken";

declare module "jsonwebtoken" {
  interface JwtPayload {
    /** Our custom claims. Namespaced to avoid collisions with other issuers. */
    "https://acme.dev/userId"?: number;
    "https://acme.dev/roles"?: readonly ("admin" | "member" | "billing")[];
    "https://acme.dev/tenantId"?: string;
  }
}

// ❌ You CANNOT do this — sub is already declared as `string | undefined`:
declare module "jsonwebtoken" {
  interface JwtPayload {
    sub: number;
    // ^^^ TS2717: Subsequent property declarations must have the same type.
    //     Property 'sub' must be of type 'string | undefined',
    //     but here has type 'number'.
  }
}
```

But **for `jsonwebtoken` specifically, augmentation is usually the wrong tool**, and this is an important judgement call. The library gives you a generic escape and, more importantly, `verify` is a **trust boundary** — the payload is attacker-influenced data. The professional pattern is: narrow the return type, then validate:

```ts
// ── src/auth/verify-token.ts — validate, don't just assert ──────────────────
import jwt, { type JwtPayload } from "jsonwebtoken";
import { z } from "zod";

const AccessTokenClaims = z.object({
  sub:      z.string().regex(/^\d+$/),
  email:    z.string().email(),
  roles:    z.array(z.enum(["admin", "member", "billing"])).readonly(),
  tenantId: z.string().uuid(),
  exp:      z.number().int(),
});

export type AccessTokenClaims = z.infer<typeof AccessTokenClaims>;

export function verifyAccessToken(authToken: string): AccessTokenClaims {
  const decoded: string | JwtPayload = jwt.verify(authToken, process.env.JWT_SECRET, {
    algorithms: ["HS256"],
    issuer:     "https://auth.acme.dev",
  });

  if (typeof decoded === "string") {
    throw ApiError.unauthorized("Token payload is not an object");
  }

  // ✅ The only honest way across a trust boundary: parse, don't cast.
  const parsed = AccessTokenClaims.safeParse(decoded);
  if (!parsed.success) throw ApiError.unauthorized("Malformed token claims");
  return parsed.data;
}
```

The rule: **augment for ergonomics on data you produce; validate for safety on data you receive.** Augmenting `JwtPayload` makes autocomplete nicer when you *sign* tokens. It does nothing to guarantee an incoming token actually has those claims.

**`socket.io`.** Socket.IO v4 deliberately made everything generic rather than augmentable, and understanding why teaches you when *not* to augment:

```ts
// ── ❌ The augmentation approach — works, but is global and unscoped ─────────
import "socket.io";

declare module "socket.io" {
  interface Socket {
    data: { userId: number; sessionId: string };
    //   ^^^ TS2717 in practice — Socket<..., SocketData>['data'] is already declared
    //       as `SocketData`, a type parameter. You cannot re-declare it.
  }
}

// ── ✅ The generic approach — what socket.io actually wants ──────────────────
import { Server, type Socket } from "socket.io";

interface ClientToServerEvents {
  "order:subscribe": (orderId: string, ack: (ok: boolean) => void) => void;
  "chat:message":    (payload: { roomId: string; text: string }) => void;
}

interface ServerToClientEvents {
  "order:updated":   (order: { orderId: string; status: string }) => void;
  "server:shutdown": (reason: string) => void;
}

interface InterServerEvents {
  ping: () => void;
}

interface SocketData {
  userId:    number;
  sessionId: string;
  roles:     readonly ("admin" | "member")[];
}

const io = new Server<
  ClientToServerEvents,
  ServerToClientEvents,
  InterServerEvents,
  SocketData
>(httpServer, { cors: { origin: process.env.CORS_ORIGIN ?? "*" } });

io.use(async (socket, next) => {
  const authToken = socket.handshake.auth.token as string | undefined;
  if (!authToken) return next(new Error("Unauthorized"));
  const claims = verifyAccessToken(authToken);
  socket.data.userId    = Number(claims.sub);     // ✅ typed as number
  socket.data.sessionId = claims.tenantId;        // ✅ typed as string
  socket.data.roles     = claims.roles;
  next();
});

io.on("connection", (socket) => {
  socket.on("order:subscribe", (orderId, ack) => {
    //                          ^^^^^^^ string, inferred from ClientToServerEvents ✅
    void socket.join(`order:${orderId}`);
    ack(true);                                     // ✅ ack signature checked
  });

  socket.emit("order:updated", { orderId: "ord_1", status: "shipped" });   // ✅
  socket.emit("order:updated", { orderId: "ord_1" });
  //                            ^^^^^^^^^^^^^^^^^^ TS2345: Property 'status' is missing.
});
```

**The judgement rule:** if the library exposes a type parameter for the thing you want to shape, use the type parameter. Generics are scoped to the instance; augmentation is global to the program. Reach for augmentation only when there is no generic — which is exactly the Express case, because `RequestHandler`'s `req` has no slot for "your app's user."

### Concept 7 — Where the file lives, and how tsc finds it

An augmentation only exists if the compiler reads the file. This is where most "it works in VS Code but fails in CI" reports come from.

**The rule:** an augmentation file is included in the program if it matches `files`, `include`, or is transitively imported by something that does. It has nothing to do with `typeRoots` or `types` unless you deliberately structure it as a types package.

```jsonc
// ✅ The simplest correct setup — file under src/, glob covers it
// tsconfig.json
{
  "compilerOptions": {
    "strict":           true,
    "module":           "NodeNext",
    "moduleResolution": "NodeNext",
    "types":            ["node"],
    "skipLibCheck":     true,
    "outDir":           "./dist",
    "rootDir":          "./src"
  },
  "include": ["src/**/*.ts"]     // ← matches src/types/express.d.ts (the glob covers .d.ts)
}
```

```text
src/
├── types/
│   ├── express.d.ts          ← augments express-serve-static-core
│   ├── env.d.ts              ← augments NodeJS.ProcessEnv
│   └── jsonwebtoken.d.ts     ← augments JwtPayload
├── middleware/
│   └── authenticate.ts
├── routes/
│   └── orders.ts
└── server.ts
```

Four traps in this area, each with a symptom you can recognise:

```jsonc
// ── Trap 1: the file lives OUTSIDE the include glob ──────────────────────────
{
  "include": ["src/**/*.ts"]
}
// types/express.d.ts   ← at the repo root, NOT under src/ → never loaded
// Symptom: TS2339 "Property 'user' does not exist on type 'Request'".
//
// ✅ Fix A: move it under src/
// ✅ Fix B: widen the glob
{ "include": ["src/**/*.ts", "types/**/*.d.ts"] }
// ✅ Fix C: force it, immune to any glob change
{ "files": ["types/express.d.ts"], "include": ["src/**/*.ts"] }
```

```jsonc
// ── Trap 2: rootDir/outDir confusion when the .d.ts lives outside rootDir ────
{ "compilerOptions": { "rootDir": "./src", "outDir": "./dist" },
  "include": ["src/**/*.ts", "types/**/*.d.ts"] }
// ❌ TS6059: File 'types/express.d.ts' is not under 'rootDir' 'src'.
// (.d.ts files emit nothing, but they still participate in rootDir computation
//  in some configurations.)
// ✅ Fix: keep augmentation files under src/. Always. It removes a whole class of bugs.
```

```jsonc
// ── Trap 3: "types": [] or a narrow types array — a red herring ──────────────
{ "compilerOptions": { "types": ["node"] }, "include": ["src/**/*.ts"] }
// `types` controls which node_modules/@types/* packages are auto-included GLOBALLY.
// It has NO effect on your own src/types/*.d.ts files — those come in via `include`.
// People add "src/types" to `typeRoots` and are surprised nothing changes.
// typeRoots entries must be structured like packages: src/types/<name>/index.d.ts
```

```bash
# ── Trap 4: works in the editor, fails in CI ────────────────────────────────
# VS Code may be using a different tsconfig (the nearest one) than your build.
# Prove which files the BUILD loads:
npx tsc -p tsconfig.json --noEmit --listFiles | grep 'src/types'
# → prints the file: it's loaded ✅
# → prints nothing:  it's not in the program ❌
#
# And check which tsconfig the editor picked:
#   VS Code command palette → "TypeScript: Go to Project Configuration"
```

**Monorepo note.** With project references, each package needs its own copy of the augmentation or must depend on a package that provides it. An augmentation in `packages/api/src/types/express.d.ts` is **not** visible to `packages/worker` unless `worker` includes it too. Put shared augmentations in a small internal package (`@acme/express-types`) and have every service import it once from its entrypoint.

### Concept 8 — What you can and cannot change

The rules, stated precisely, because "you can only add" is a useful slogan but not the whole truth.

```ts
declare module "express-serve-static-core" {
  interface Request {
    // ✅ ADD a member that does not exist
    tenantId?: string;

    // ✅ RE-DECLARE a member with the EXACT same type (legal, pointless)
    method: string;

    // ❌ RE-DECLARE with a different type
    method: "GET" | "POST";
    // TS2717: Subsequent property declarations must have the same type.
    //         Property 'method' must be of type 'string',
    //         but here has type '"GET" | "POST"'.

    // ✅ ADD an OVERLOAD to an existing method (new signature wins for matching calls)
    header(name: "x-acme-tenant"): string | undefined;

    // ❌ REMOVE a member — there is no syntax for this. None. Not possible.
  }

  // ✅ ADD a brand-new exported interface/type to the module's namespace
  interface AcmeRequestContext {
    requestId: string;
    startedAtMs: number;
  }

  // ❌ ADD a default export
  export default class Foo {}
  // TS2669 / TS2665: cannot add a default export to an augmentation.

  // ⚠️ ADD a value declaration — syntactically legal, semantically a lie
  export const ACME_VERSION: string;
  // The compiler now believes `import { ACME_VERSION } from "express-serve-static-core"`
  // works. At runtime it is `undefined`. Never do this.
}
```

Additional constraints worth memorising:

- **You cannot augment a `type` alias.** Only `interface`, `namespace`, `class` (interface side), and `enum` participate in merging. If a library exports `export type Config = {...}`, it is sealed against augmentation. Nothing you can do.
- **You cannot augment a default export directly.** `export default interface Foo` in the target can be augmented only if the interface also has a name in the module's scope.
- **Optionality can be added but not removed.** Declaring `user: AuthenticatedUser` when the original had `user?: AuthenticatedUser` is a TS2717 conflict (`AuthenticatedUser` vs `AuthenticatedUser | undefined`).
- **Generic interfaces must be augmented with matching type parameters**, and the parameter *names* must match, because the merged declaration is textually combined:

```ts
// The target: interface Request<P = ParamsDictionary, ResBody = any, ReqBody = any> { ... }

// ✅ Omitting type parameters entirely is allowed and is what you almost always want:
declare module "express-serve-static-core" {
  interface Request { user?: AuthenticatedUser }
}

// ⚠️ If you DO write them, they must match arity and constraints exactly:
declare module "express-serve-static-core" {
  interface Request<P, ResBody, ReqBody> { user?: AuthenticatedUser }
  // TS2428: All declarations of 'Request' must have identical type parameters.
  // (defaults differ → mismatch)
}
```

---

## Example 1 — basic

```ts
// ══════════════════════════════════════════════════════════════════════════════
// Adding `req.requestId` and a typed `process.env` to a small Express service.
// Three files: the augmentation, the middleware, the config.
// ══════════════════════════════════════════════════════════════════════════════

// ─────────────────────────────────────────────────────────────────────────────
// 1. src/types/express.d.ts — augment Express's Request
// ─────────────────────────────────────────────────────────────────────────────
import "express-serve-static-core";     // ← makes this file a MODULE (critical, Concept 2)
                                        //   and loads the declarations we are merging into

declare module "express-serve-static-core" {
  interface Request {
    /** UUID assigned by requestId middleware. Present on every request. */
    requestId: string;
    /** Monotonic clock reading captured when the request entered the app. */
    startedAtMs: number;
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 2. src/types/env.d.ts — augment NodeJS.ProcessEnv
// ─────────────────────────────────────────────────────────────────────────────
export {};                              // ← module marker; `declare global` requires it

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV:     "development" | "test" | "production";
      DATABASE_URL: string;
      PORT?:        string;
      LOG_LEVEL?:   "debug" | "info" | "warn" | "error";
    }
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 3. src/middleware/request-context.ts — the middleware that makes it true
// ─────────────────────────────────────────────────────────────────────────────
import { randomUUID } from "node:crypto";
import type { Request, Response, NextFunction } from "express";

export function requestContext(req: Request, res: Response, next: NextFunction): void {
  req.requestId   = req.header("x-request-id") ?? randomUUID();   // ✅ typed as string
  req.startedAtMs = performance.now();                            // ✅ typed as number

  res.setHeader("x-request-id", req.requestId);

  res.on("finish", () => {
    const durationMs = Math.round(performance.now() - req.startedAtMs);
    console.log(
      JSON.stringify({
        requestId:  req.requestId,
        method:     req.method,
        path:       req.path,
        statusCode: res.statusCode,
        durationMs,
      }),
    );
  });

  next();
}

// ─────────────────────────────────────────────────────────────────────────────
// 4. src/config.ts — the augmented env in use
// ─────────────────────────────────────────────────────────────────────────────
export const config = {
  nodeEnv:     process.env.NODE_ENV,                    // ✅ "development"|"test"|"production"
  databaseUrl: process.env.DATABASE_URL,                // ✅ string (not string | undefined)
  port:        Number(process.env.PORT ?? "3000"),      // ✅ PORT is string | undefined
  logLevel:    process.env.LOG_LEVEL ?? "info",         // ✅ narrow union with a default
  isProd:      process.env.NODE_ENV === "production",   // ✅ comparison is checked
} as const;

// ❌ Errors the augmentations now produce:
// req.requestID;                       TS2551: Did you mean 'requestId'?
// req.requestId = 42;                  TS2322: Type 'number' is not assignable to 'string'.
// config.port.toUpperCase();           TS2339: Property 'toUpperCase' does not exist on 'number'.
// if (process.env.NODE_ENV === "prod") TS2367: This comparison appears to be unintentional
//                                             because the types have no overlap.
// process.env.LOG_LEVEL = "verbose";   TS2322: Type '"verbose"' is not assignable to
//                                             '"debug" | "info" | "warn" | "error" | undefined'.

// ─────────────────────────────────────────────────────────────────────────────
// 5. tsconfig.json — the part that makes any of this load
// ─────────────────────────────────────────────────────────────────────────────
// {
//   "compilerOptions": {
//     "strict":           true,
//     "module":           "NodeNext",
//     "moduleResolution": "NodeNext",
//     "types":            ["node"],
//     "skipLibCheck":     true
//   },
//   "include": ["src/**/*.ts"]      // ← covers src/types/*.d.ts
// }
//
// Verify:  npx tsc --noEmit --listFiles | grep 'src/types'
```

---

## Example 2 — real world backend use case

```ts
// ══════════════════════════════════════════════════════════════════════════════
// A production Express + JWT + multi-tenant service.
// Augmentations: express-serve-static-core (Request, Response, Locals),
//                NodeJS.ProcessEnv, and jsonwebtoken's JwtPayload for SIGNING.
// Everything below is one service; file boundaries are marked.
// ══════════════════════════════════════════════════════════════════════════════

// ─────────────────────────────────────────────────────────────────────────────
// 1. src/auth/authenticated-user.ts — the domain type (NOT in a .d.ts)
// ─────────────────────────────────────────────────────────────────────────────
export type AuthRole = "admin" | "member" | "billing" | "support";

export interface AuthenticatedUser {
  userId:    number;
  email:     string;
  tenantId:  string;
  roles:     readonly AuthRole[];
  tokenId:   string;
  expiresAt: Date;
}

export function hasRole(user: AuthenticatedUser, role: AuthRole): boolean {
  return user.roles.includes(role);
}

// ─────────────────────────────────────────────────────────────────────────────
// 2. src/errors.ts — the error type used by the Response augmentation
// ─────────────────────────────────────────────────────────────────────────────
export type ErrorCode =
  | "BAD_REQUEST" | "UNAUTHORIZED" | "FORBIDDEN"
  | "NOT_FOUND"   | "CONFLICT"     | "INTERNAL";

export class ApiError extends Error {
  readonly code:       ErrorCode;
  readonly statusCode: number;
  readonly details?:   Record<string, unknown>;

  private constructor(code: ErrorCode, statusCode: number, message: string,
                      details?: Record<string, unknown>) {
    super(message);
    this.name       = "ApiError";
    this.code       = code;
    this.statusCode = statusCode;
    this.details    = details;
  }

  static badRequest(message: string, details?: Record<string, unknown>): ApiError {
    return new ApiError("BAD_REQUEST", 400, message, details);
  }
  static unauthorized(message = "Authentication required"): ApiError {
    return new ApiError("UNAUTHORIZED", 401, message);
  }
  static forbidden(message = "Insufficient permissions"): ApiError {
    return new ApiError("FORBIDDEN", 403, message);
  }
  static notFound(resource: string): ApiError {
    return new ApiError("NOT_FOUND", 404, `${resource} not found`);
  }
  static internal(message = "Internal server error"): ApiError {
    return new ApiError("INTERNAL", 500, message);
  }
}

export interface ApiResponse<TBody> {
  data:      TBody;
  requestId: string;
}

// ─────────────────────────────────────────────────────────────────────────────
// 3. src/types/express.d.ts — THE AUGMENTATION
// ─────────────────────────────────────────────────────────────────────────────
import type { AuthenticatedUser } from "../auth/authenticated-user.js";
import type { Logger } from "pino";
//     ^ these type-only imports make the file a MODULE → `declare module` = AUGMENT

declare module "express-serve-static-core" {
  interface Request {
    /**
     * The authenticated principal.
     * OPTIONAL on purpose: public routes (/health, /login, /webhooks/*) never set it.
     * Use `requireUser(req)` to get a non-optional value with a 401 on failure.
     */
    user?: AuthenticatedUser;

    /** Correlation id. Required — `requestContext` is mounted before every route. */
    requestId: string;

    /** Request-scoped logger, pre-bound with requestId and (if present) userId. */
    log: Logger;

    /** performance.now() at request entry — used for the duration metric. */
    startedAtMs: number;

    /** Raw body buffer, populated only for routes needing signature verification. */
    rawBody?: Buffer;
  }

  interface Response {
    /** Send a success envelope with the requestId attached. */
    sendData<TBody>(statusCode: number, data: TBody): void;

    /** Serialise any thrown value into the standard error envelope. */
    sendApiError(error: unknown): void;
  }

  interface Locals {
    /** Resolved by the tenancy middleware from the subdomain or the JWT claim. */
    tenantId?: string;
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 4. src/types/env.d.ts — the environment contract
// ─────────────────────────────────────────────────────────────────────────────
export {};

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV:              "development" | "test" | "production";
      PORT?:                 string;

      DATABASE_URL:          string;
      REDIS_URL:             string;

      JWT_SECRET:            string;
      JWT_ISSUER:            string;
      JWT_AUDIENCE:          string;

      STRIPE_WEBHOOK_SECRET: string;

      LOG_LEVEL?:            "trace" | "debug" | "info" | "warn" | "error";
      SENTRY_DSN?:           string;
      OTEL_ENDPOINT?:        string;
    }
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 5. src/types/jsonwebtoken.d.ts — augment the payload for SIGNING
// ─────────────────────────────────────────────────────────────────────────────
import "jsonwebtoken";
import type { AuthRole } from "../auth/authenticated-user.js";

declare module "jsonwebtoken" {
  /**
   * Custom claims we EMIT. Namespaced URIs avoid collision with claims minted by
   * other issuers whose tokens might also flow through this service.
   * All optional — augmenting cannot make an existing optional claim required,
   * and an incoming token may legitimately lack them.
   */
  interface JwtPayload {
    "https://acme.dev/userId"?:   number;
    "https://acme.dev/email"?:    string;
    "https://acme.dev/tenantId"?: string;
    "https://acme.dev/roles"?:    readonly AuthRole[];
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 6. src/config.ts — assert the env augmentation is actually true
// ─────────────────────────────────────────────────────────────────────────────
const REQUIRED_ENV = [
  "NODE_ENV", "DATABASE_URL", "REDIS_URL",
  "JWT_SECRET", "JWT_ISSUER", "JWT_AUDIENCE",
  "STRIPE_WEBHOOK_SECRET",
] as const satisfies readonly (keyof NodeJS.ProcessEnv)[];
//  ^ `satisfies` binds the runtime list to the augmented type: misspell a key here
//    and it's a compile error, not a silent no-op check.

export function assertEnv(): void {
  const missing = REQUIRED_ENV.filter((key) => !process.env[key]);
  if (missing.length > 0) {
    throw new Error(
      `Refusing to start: missing environment variables ${missing.join(", ")}`,
    );
  }
}

assertEnv();

export const config = {
  nodeEnv:             process.env.NODE_ENV,
  port:                Number(process.env.PORT ?? "8080"),
  databaseUrl:         process.env.DATABASE_URL,
  redisUrl:            process.env.REDIS_URL,
  jwtSecret:           process.env.JWT_SECRET,
  jwtIssuer:           process.env.JWT_ISSUER,
  jwtAudience:         process.env.JWT_AUDIENCE,
  stripeWebhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
  logLevel:            process.env.LOG_LEVEL ?? "info",
} as const;

// ─────────────────────────────────────────────────────────────────────────────
// 7. src/auth/tokens.ts — signing uses the augmentation; verifying validates
// ─────────────────────────────────────────────────────────────────────────────
import jwt, { type JwtPayload, type SignOptions } from "jsonwebtoken";
import { z } from "zod";
import { config } from "../config.js";
import { ApiError } from "../errors.js";
import type { AuthenticatedUser, AuthRole } from "./authenticated-user.js";

/** SIGNING — the augmented JwtPayload gives us autocomplete + key checking. */
export function signAccessToken(user: AuthenticatedUser, ttlSeconds = 900): string {
  const claims: JwtPayload = {
    sub:                          String(user.userId),
    jti:                          user.tokenId,
    "https://acme.dev/userId":    user.userId,      // ✅ augmented, typed number
    "https://acme.dev/email":     user.email,
    "https://acme.dev/tenantId":  user.tenantId,
    "https://acme.dev/roles":     user.roles,       // ✅ readonly AuthRole[]
  };

  // ❌ "https://acme.dev/userid": user.userId
  //    → no error! JwtPayload has [key: string]: any, so unknown keys are legal.
  //      Augmentation buys autocomplete and value-type checking on KNOWN keys only.

  const options: SignOptions = {
    algorithm: "HS256",
    issuer:    config.jwtIssuer,
    audience:  config.jwtAudience,
    expiresIn: ttlSeconds,
  };
  return jwt.sign(claims, config.jwtSecret, options);
}

/** VERIFYING — a trust boundary. Parse, never assert. */
const AccessClaims = z.object({
  sub: z.string(),
  jti: z.string(),
  exp: z.number().int(),
  "https://acme.dev/userId":   z.number().int().positive(),
  "https://acme.dev/email":    z.string().email(),
  "https://acme.dev/tenantId": z.string().uuid(),
  "https://acme.dev/roles":    z.array(
    z.enum(["admin", "member", "billing", "support"]),
  ),
});

export function verifyAccessToken(authToken: string): AuthenticatedUser {
  let decoded: string | JwtPayload;
  try {
    decoded = jwt.verify(authToken, config.jwtSecret, {
      algorithms: ["HS256"],
      issuer:     config.jwtIssuer,
      audience:   config.jwtAudience,
    });
  } catch {
    throw ApiError.unauthorized("Invalid or expired token");
  }

  if (typeof decoded === "string") throw ApiError.unauthorized("Malformed token payload");

  const parsed = AccessClaims.safeParse(decoded);
  if (!parsed.success) throw ApiError.unauthorized("Token is missing required claims");

  const c = parsed.data;
  return {
    userId:    c["https://acme.dev/userId"],
    email:     c["https://acme.dev/email"],
    tenantId:  c["https://acme.dev/tenantId"],
    roles:     c["https://acme.dev/roles"] as readonly AuthRole[],
    tokenId:   c.jti,
    expiresAt: new Date(c.exp * 1000),
  };
}

// ─────────────────────────────────────────────────────────────────────────────
// 8. src/middleware/authenticate.ts — the middleware that fills req.user
// ─────────────────────────────────────────────────────────────────────────────
import type { Request, Response, NextFunction, RequestHandler } from "express";
import { ApiError } from "../errors.js";
import { verifyAccessToken } from "../auth/tokens.js";
import type { AuthenticatedUser, AuthRole } from "../auth/authenticated-user.js";

export const authenticate: RequestHandler = (req, res, next) => {
  const header = req.header("authorization");
  if (!header?.startsWith("Bearer ")) {
    return res.sendApiError(ApiError.unauthorized("Missing bearer token"));
  }
  try {
    req.user = verifyAccessToken(header.slice(7));    // ✅ typed by the augmentation
    req.log  = req.log.child({ userId: req.user.userId });
    res.locals.tenantId = req.user.tenantId;          // ✅ Locals augmentation
    next();
  } catch (error) {
    res.sendApiError(error);
  }
};

/** Type predicate — narrows Request to one that definitely carries a user. */
export function isAuthenticated(
  req: Request,
): req is Request & { user: AuthenticatedUser } {
  return req.user !== undefined;
}

/** Throwing accessor — the ergonomic escape from `user?`. */
export function requireUser(req: Request): AuthenticatedUser {
  if (!req.user) throw ApiError.unauthorized();
  return req.user;
}

/** Role gate, built on the augmented Request. */
export function requireRole(...allowed: readonly AuthRole[]): RequestHandler {
  return (req, res, next) => {
    const user = req.user;
    if (!user) return res.sendApiError(ApiError.unauthorized());
    if (!allowed.some((role) => user.roles.includes(role))) {
      return res.sendApiError(ApiError.forbidden(`Requires one of: ${allowed.join(", ")}`));
    }
    next();
  };
}

// ─────────────────────────────────────────────────────────────────────────────
// 9. src/middleware/response-helpers.ts — makes the Response augmentation TRUE
// ─────────────────────────────────────────────────────────────────────────────
import type { RequestHandler } from "express";
import { ApiError } from "../errors.js";

/**
 * The augmentation DECLARED res.sendData / res.sendApiError.
 * Declaring is not implementing — this middleware must run before any route,
 * or every call is a runtime TypeError with a clean compile.
 */
export const responseHelpers: RequestHandler = (req, res, next) => {
  res.sendData = function <TBody>(statusCode: number, data: TBody): void {
    res.status(statusCode).json({ data, requestId: req.requestId });
  };

  res.sendApiError = function (error: unknown): void {
    if (error instanceof ApiError) {
      req.log.warn({ code: error.code, err: error }, "request failed");
      res.status(error.statusCode).json({
        error: { code: error.code, message: error.message, details: error.details },
        requestId: req.requestId,
      });
      return;
    }
    req.log.error({ err: error }, "unhandled error");
    res.status(500).json({
      error: { code: "INTERNAL", message: "Internal server error" },
      requestId: req.requestId,
    });
  };

  next();
};

// ─────────────────────────────────────────────────────────────────────────────
// 10. src/routes/orders.ts — the payoff: zero casts, full checking
// ─────────────────────────────────────────────────────────────────────────────
import { Router } from "express";
import { z } from "zod";
import { authenticate, requireRole, requireUser } from "../middleware/authenticate.js";
import { ApiError } from "../errors.js";

interface Order {
  orderId:    string;
  userId:     number;
  tenantId:   string;
  totalCents: number;
  status:     "pending" | "paid" | "shipped" | "cancelled";
  createdAt:  Date;
}

const CreateOrderBody = z.object({
  sku:      z.string().min(1),
  quantity: z.number().int().positive().max(999),
});

export const orderRoutes = Router();

orderRoutes.get("/orders", authenticate, async (req, res) => {
  const user = requireUser(req);                       // AuthenticatedUser ✅
  req.log.info({ requestId: req.requestId }, "listing orders");   // ✅ augmented log

  const orders = await orderRepo.listForUser(user.userId, user.tenantId);
  res.sendData<readonly Order[]>(200, orders);         // ✅ augmented Response method
});

orderRoutes.post("/orders", authenticate, async (req, res) => {
  const user = requireUser(req);

  const parsed = CreateOrderBody.safeParse(req.body);  // req.body is `any` from express
  if (!parsed.success) {
    return res.sendApiError(
      ApiError.badRequest("Invalid request body", { issues: parsed.error.issues }),
    );
  }

  const order = await orderRepo.create({
    userId:   user.userId,
    tenantId: user.tenantId,
    sku:      parsed.data.sku,
    quantity: parsed.data.quantity,
  });
  res.sendData<Order>(201, order);
});

orderRoutes.delete(
  "/orders/:orderId",
  authenticate,
  requireRole("admin", "support"),                     // ✅ AuthRole union checked
  async (req, res) => {
    const user  = requireUser(req);
    const order = await orderRepo.findById(req.params.orderId);
    if (!order || order.tenantId !== user.tenantId) {
      return res.sendApiError(ApiError.notFound("Order"));
    }
    await orderRepo.cancel(order.orderId);
    res.sendData(200, { orderId: order.orderId, status: "cancelled" as const });
  },
);

// ❌ Every one of these is now a compile error, in ordinary handler code:
//
// req.user.userId
//   TS18048: 'req.user' is possibly 'undefined'.       ← forgot `authenticate`
//
// requireUser(req).userID
//   TS2551: Property 'userID' does not exist. Did you mean 'userId'?
//
// requireRole("adming")
//   TS2345: Argument of type '"adming"' is not assignable to parameter of type 'AuthRole'.
//
// res.sendData(200)
//   TS2554: Expected 2 arguments, but got 1.
//
// req.requestID
//   TS2551: Property 'requestID' does not exist. Did you mean 'requestId'?
//
// process.env.STRIPE_WEBHOOK_SECRET.slice(0, 4)   ← ✅ compiles, because we declared it
//                                                    required AND assertEnv() proves it

// ─────────────────────────────────────────────────────────────────────────────
// 11. src/server.ts — ordering matters: helpers before routes
// ─────────────────────────────────────────────────────────────────────────────
import express from "express";
import { config } from "./config.js";
import { requestContext } from "./middleware/request-context.js";
import { responseHelpers } from "./middleware/response-helpers.js";
import { orderRoutes } from "./routes/orders.js";

const app = express();

app.use(express.json({ limit: "1mb" }));
app.use(requestContext);      // ← makes req.requestId / req.log / req.startedAtMs true
app.use(responseHelpers);     // ← makes res.sendData / res.sendApiError true
app.use("/v1", orderRoutes);

app.listen(config.port, () => {
  console.log(`listening on :${config.port} (${config.nodeEnv})`);
});

// ⚠️ THE ONE RISK OF AUGMENTATION, stated plainly:
// The augmentation promises `req.requestId: string` and `res.sendData(...)`.
// If `requestContext` or `responseHelpers` is not mounted — or is mounted AFTER
// a route — the promise is false and you get a runtime TypeError with a clean build.
// Augmentation moves the burden of proof from the compiler to your app wiring.
// Cover it with one integration test that boots the real app and hits one route.
```

---

## Going deeper

### The augmentation is global — that is the whole design, and the whole hazard

An augmentation is not scoped to the importing file. Once *any* file in the program declares it, the change applies to **every** file, including files in `node_modules` that the compiler is checking.

```ts
// src/types/express.d.ts — this file is never imported by anything
import "express-serve-static-core";
declare module "express-serve-static-core" {
  interface Request { user?: AuthenticatedUser }
}
```

```ts
// src/routes/health.ts — no import of the augmentation file anywhere
import type { Request } from "express";

export function health(req: Request) {
  return { ok: true, user: req.user };   // ✅ req.user is visible here. No import needed.
}
```

That is a feature for an application. It is a **serious problem for a library**:

```ts
// ❌ Inside a published package @acme/express-middleware/src/index.ts
import "express-serve-static-core";

declare module "express-serve-static-core" {
  interface Request {
    user?: { userId: number };     // ← ships in the package's .d.ts
  }
}
```

Now every consumer of `@acme/express-middleware` gets `req.user?: { userId: number }` injected into *their* Express types. If their app already augments `Request` with `user?: AuthenticatedUser`, the two collide:

```text
error TS2717: Subsequent property declarations must have the same type.
  Property 'user' must be of type '{ userId: number; } | undefined',
  but here has type 'AuthenticatedUser | undefined'.
```

Neither side can fix it without the other changing. This is the "augmentation leaking across packages" problem, and it is why `@types/passport` was so painful for years.

The professional rules for library authors:

```ts
// ✅ Rule 1 — Use a namespaced property nobody else will claim.
declare module "express-serve-static-core" {
  interface Request {
    acmeRateLimit?: { remaining: number; resetAt: Date };
  }
}

// ✅ Rule 2 — Point at a type YOU export, so consumers can widen it themselves.
declare module "express-serve-static-core" {
  interface Request {
    acmeContext?: import("@acme/express-middleware").AcmeContext;
  }
}

// ✅ Rule 3 — Better: prefer a generic or an explicit accessor over augmentation.
export function getAcmeContext(req: Request): AcmeContext | undefined {
  return (req as { [ACME_KEY]?: AcmeContext })[ACME_KEY];
}
// No global mutation of anyone's types. Slightly less ergonomic. Zero collisions.

// ✅ Rule 4 — If you MUST augment, ship it in an OPT-IN entrypoint.
// package.json exports:  "./augment": { "types": "./dist/augment.d.ts", ... }
// Consumers write:       import "@acme/express-middleware/augment";
```

### `declare module` vs `declare global` vs ambient module declaration

Three syntaxes, constantly confused. Here they are side by side with the decision rule.

```ts
// ═══ A. AUGMENT AN EXISTING MODULE ═════════════════════════════════════════
// File MUST be a module. Specifier MUST resolve.
import "express-serve-static-core";
declare module "express-serve-static-core" {
  interface Request { user?: AuthenticatedUser }
}
// Effect: MERGE into the existing module's declarations.


// ═══ B. DECLARE A MODULE THAT HAS NO TYPES AT ALL ══════════════════════════
// File MUST be a script (no top-level import/export).
declare module "@acme/legacy-cache" {
  export function get(key: string): string | null;
}
// Effect: CREATE the module's entire type. Shadows anything real.
// Only correct when the package genuinely ships no types.


// ═══ C. AUGMENT THE GLOBAL SCOPE ═══════════════════════════════════════════
// File MUST be a module (that's why `export {}` is there).
export {};
declare global {
  namespace NodeJS {
    interface ProcessEnv { DATABASE_URL: string }
  }
  interface Array<T> { last(): T | undefined }   // ⚠️ prototype pollution, be careful
  var appStartedAtMs: number;                    // must be `var`, not `let`/`const`
}
// Effect: MERGE into the global declaration space.
```

The decision rule, in three questions:

1. Does the thing I'm changing live inside a module I can `import`? → **A**.
2. Does the package have no types whatsoever? → **B** (and see `50 — Ambient declarations`).
3. Is it a global — `process`, `globalThis`, `Array.prototype`, `Express.User`? → **C**.

The killer detail: **A and B use identical syntax and are distinguished only by whether the file is a module.** Adding an `import` to a B-file silently converts it into an A-file that augments a module that may not exist. Removing an `import` from an A-file silently converts it into a B-file that obliterates the real module's types. Both changes look innocuous in a diff.

```bash
# The diagnostic: if the augmentation resolves, tsc says nothing.
# If the specifier does NOT resolve in a MODULE file, tsc says:
#   error TS2664: Invalid module name in augmentation, module
#   'express-serve-static-cor' cannot be found.
# TS2664 is your friend — it only appears in the augment case, and it proves
# your file is correctly a module.
```

### Interface merging: member ordering, overloads, and which declaration wins

Merging is not a simple set union — order matters for **call and construct signatures**.

```ts
// Later declarations' overloads are ordered FIRST in the merged interface,
// so an augmentation's overloads take priority over the original's.

// node_modules/@types/express-serve-static-core/index.d.ts
interface Request {
  header(name: "set-cookie"): string[] | undefined;
  header(name: string): string | undefined;
}

// your augmentation
declare module "express-serve-static-core" {
  interface Request {
    header(name: "x-acme-tenant"): string;   // non-optional return for OUR header
  }
}

// Merged overload resolution order:
//   1. header(name: "x-acme-tenant"): string          ← yours, checked first
//   2. header(name: "set-cookie"): string[] | undefined
//   3. header(name: string): string | undefined

const tenantId = req.header("x-acme-tenant");   // string          ✅ your overload wins
const cookies  = req.header("set-cookie");      // string[] | undefined
const other    = req.header("x-forwarded-for"); // string | undefined
```

Within a *single* interface body, the source order of overloads is preserved. Across merged declarations, **the last-declared interface's members come first**. This is documented behaviour and is what makes overload-adding augmentations useful — but it also means a careless augmentation can shadow a library's carefully ordered overloads.

Non-function members have no ordering semantics; they must simply be *identical* or you get TS2717.

```ts
// A subtlety with generics and merged members:
interface Repository<TEntity> { findById(id: string): Promise<TEntity | null> }

declare module "@acme/data" {
  interface Repository<TEntity> {              // ✅ type parameter name must match
    findManyById(ids: readonly string[]): Promise<readonly TEntity[]>;
  }
  interface Repository<TRecord> {              // ❌ TS2428: All declarations of
    count(): Promise<number>;                  //    'Repository' must have identical
  }                                            //    type parameters.
}
```

### Augmenting `Express.User` — the passport special case

If you use `passport`, you will hit a different pattern, because `@types/passport` augments a **global namespace**, not a module:

```ts
// node_modules/@types/passport/index.d.ts (abridged)
declare global {
  namespace Express {
    interface AuthInfo {}
    interface User {}                    // ← deliberately empty, for YOU to fill
    interface Request {
      authInfo?: AuthInfo | undefined;
      user?: User | undefined;           // ← so req.user's type IS Express.User
      login(user: User, done: (err: any) => void): void;
      logout(options: any, done: (err: any) => void): void;
      isAuthenticated(): this is AuthenticatedRequest;
    }
  }
}
```

```ts
// ✅ src/types/passport.d.ts — fill in the empty interface, don't fight it
export {};

declare global {
  namespace Express {
    interface User {
      userId:   number;
      email:    string;
      tenantId: string;
      roles:    readonly ("admin" | "member" | "billing")[];
    }
  }
}
```

```ts
// Now req.user is Express.User — and passport's own signatures agree:
app.get("/me", (req, res) => {
  if (!req.isAuthenticated()) return res.sendStatus(401);
  res.json({ userId: req.user.userId });     // ✅ narrowed by the type predicate
});
```

**The collision you will eventually hit:** if you *also* augment `express-serve-static-core`'s `Request` with `user?: AuthenticatedUser`, and passport declares `user?: Express.User`, you get TS2717 — two different types for the same property. Pick **one** mechanism. With passport installed, fill `Express.User`. Without it, augment `express-serve-static-core`.

```bash
# Diagnose a TS2717 on `user` — find every declaration of the property:
grep -rn "user?:" node_modules/@types/express-serve-static-core/index.d.ts \
                  node_modules/@types/passport/index.d.ts \
                  src/types/
```

### Duplicate `@types/node` and TS2717 storms

The most alarming failure mode looks like your augmentation broke everything:

```text
node_modules/@types/node/globals.d.ts:72:13 - error TS2717: Subsequent property
  declarations must have the same type. Property 'process' must be of type
  'Process', but here has type 'Process'.
```

Two identical-looking types that are not the same type, because they come from **two copies of `@types/node`** in the tree. Merging requires *identity*, and types from different files are different types.

```bash
# Diagnose:
npm ls @types/node          # look for more than one version
npm ls @types/express

# Fix — force a single version:
#   npm:  "overrides":   { "@types/node": "^22.10.0" }
#   pnpm: "pnpm": { "overrides": { "@types/node": "^22.10.0" } }
#   yarn: "resolutions": { "@types/node": "^22.10.0" }

# Mitigate (does not fix, but stops the noise):
#   "skipLibCheck": true   ← suppresses errors INSIDE .d.ts files
```

`skipLibCheck: true` is the standard mitigation and is why most Node projects run with it on. Be aware of what it hides: it also stops checking *your own* augmentation files for internal consistency. An augmentation with a genuine TS2717 conflict against a library type will be silently ignored under `skipLibCheck`, and `req.user` will just... not be there, with no error explaining why. When an augmentation mysteriously does nothing, temporarily flip `skipLibCheck` off:

```bash
npx tsc --noEmit --skipLibCheck false 2>&1 | head -50
```

### Augmentation vs. the alternatives — a decision table

Augmentation is one of five ways to attach data to a framework object. Choosing correctly matters more than executing correctly.

| Technique | When it's right | Cost |
|---|---|---|
| **Module augmentation** | Framework has no generic slot (Express `Request`), your app owns the whole program | Global; collides across packages; declaration ≠ implementation |
| **Generic type parameter** | Library exposes one (`socket.io`, `fastify` `RouteGeneric`, `knex` `Tables`) | None — always prefer this |
| **`AsyncLocalStorage`** | Request-scoped context you don't want on the request object at all | Slight perf cost; must be typed manually |
| **Explicit accessor + `WeakMap`** | Library code that must not pollute consumer types | Less ergonomic; no `req.x` syntax |
| **Wrapper type + adapter** | You control every call site and want zero global effects | Verbose; fights framework handler signatures |

```ts
// ── The AsyncLocalStorage alternative, fully typed — no augmentation at all ──
import { AsyncLocalStorage } from "node:async_hooks";
import type { RequestHandler } from "express";

export interface RequestScope {
  requestId: string;
  user?:     AuthenticatedUser;
  startedAtMs: number;
}

const storage = new AsyncLocalStorage<RequestScope>();

export const scopeMiddleware: RequestHandler = (req, res, next) => {
  storage.run(
    { requestId: randomUUID(), startedAtMs: performance.now() },
    () => next(),
  );
};

/** Throws rather than returning undefined — the scope must exist inside a request. */
export function currentScope(): RequestScope {
  const scope = storage.getStore();
  if (!scope) throw new Error("currentScope() called outside a request");
  return scope;
}

// Usage — no req.user, no augmentation, no global type mutation:
orderRoutes.get("/orders", authenticate, async (_req, res) => {
  const { requestId, user } = currentScope();
  if (!user) return res.sendStatus(401);
  res.json({ requestId, orders: await orderRepo.listForUser(user.userId) });
});
```

This is genuinely better for library code and for deep call stacks (a repository three layers down can read the requestId without threading it through six signatures). It is worse for route handlers, where `req.user` is simply the idiom everyone reads fluently. Real services use both.

### Augmentation declares; it does not implement

Worth stating as its own section because it is the bug that reaches production.

```ts
// The augmentation:
declare module "express-serve-static-core" {
  interface Response {
    sendData<TBody>(statusCode: number, data: TBody): void;
  }
}
```

```ts
// ✅ Compiles perfectly:
res.sendData(200, { orderId: "ord_1" });

// 💥 Runtime, if the responseHelpers middleware was never mounted:
// TypeError: res.sendData is not a function
```

The compiler cannot verify that any JavaScript ever assigns `sendData`. This is the same asymmetry as any `.d.ts` (see `49 — Declaration files`): the declaration is a promise you make and must keep. Two habits close the gap:

```ts
// ✅ Habit 1 — implement immediately adjacent to the augmentation, and mount first.
//    Keep `src/types/express.d.ts` and `src/middleware/response-helpers.ts` in the
//    same PR, and put the mount at the very top of server.ts with a comment.

// ✅ Habit 2 — one smoke test that boots the real app and exercises each augmented member.
import request from "supertest";
import { createApp } from "../src/app.js";

describe("request/response augmentations are actually implemented", () => {
  it("sets req.requestId and echoes it via res.sendData", async () => {
    const response = await request(createApp()).get("/v1/health");
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty("requestId");
    expect(response.headers["x-request-id"]).toBeDefined();
  });
});
```

---

## Common mistakes

### Mistake 1 — The augmentation file is not a module (no import/export)

```ts
// ❌ src/types/express.d.ts — no top-level import or export
declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
```

This file is a **script**, so `declare module` is an *ambient module declaration*, not an augmentation. It replaces `express-serve-static-core`'s entire type surface with one interface. Symptoms are bizarre and rarely point at this file:

```ts
router.get("/orders", (req, res) => {
  req.user;                 // maybe works
  req.params.orderId;       // ❌ TS2339: Property 'params' does not exist on type 'Request'
  req.body;                 // ❌ TS2339
  res.status(200);          // ❌ TS2339: Property 'status' does not exist on type 'Response'
});
// "Why did adding one property delete the entire Express type?"
```

```ts
// ✅ Add the import that makes it a module — one line, whole problem solved
import "express-serve-static-core";

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
```

```ts
// ✅ Or, if you don't want to import the target, any export works:
export {};
import "express-serve-static-core";   // still needed to LOAD the target's declarations
```

**How to tell instantly:** a correct augmentation with a *typo* in the specifier produces `TS2664: Invalid module name in augmentation`. A script-file "augmentation" with a typo produces **no error at all** — it just declares a new module nobody imports. If you deliberately misspell the specifier and get no error, your file is a script.

### Mistake 2 — Augmenting `"express"` instead of `"express-serve-static-core"`

```ts
// ❌ Looks right, does nothing useful
import "express";

declare module "express" {
  interface Request {
    user?: AuthenticatedUser;
  }
}

// Symptom: req.user still errors inside handlers
router.get("/orders", (req, res) => {
  req.user;   // ❌ TS2339: Property 'user' does not exist on type
              //    'Request<ParamsDictionary, any, any, ParsedQs, Record<string, any>>'
});
```

The handler's `req` is typed as `core.Request` from `express-serve-static-core`; `express`'s `Request` is a separate empty interface that merely *extends* it. Augmenting the derived interface does not add members to the base.

```ts
// ✅ Augment where the interface is declared
import "express-serve-static-core";

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
```

```bash
# ✅ The general procedure for ANY library — never guess the specifier:
# 1. Go-to-definition on the type in your editor; read the file path.
# 2. Or grep for the declaration:
grep -rln "interface Request" node_modules/@types/express*/
#    → node_modules/@types/express-serve-static-core/index.d.ts   ← the specifier
# 3. Verify with a deliberate typo: if you don't get TS2664, something else is wrong.
```

### Mistake 3 — Declaring `req.user` as required when it isn't

```ts
// ❌ The "convenient" augmentation
declare module "express-serve-static-core" {
  interface Request {
    user: AuthenticatedUser;      // no `?` — no guards needed anywhere!
  }
}
```

```ts
// Compiles clean, crashes in production on every public route:
app.get("/health", (req, res) => {
  res.json({ ok: true, tenantId: req.user.tenantId });
  //                              ^^^^^^^^ compiles ✅ — undefined at runtime 💥
});

app.post("/auth/login", (req, res) => {
  logLoginAttempt(req.user.email);        // 💥 TypeError on the LOGIN route
});

app.post("/webhooks/stripe", (req, res) => {
  audit(req.user.userId, "stripe.webhook");   // 💥 Stripe has no bearer token
});
```

You did not make the code safer; you removed the compiler's ability to tell you that a route is missing its auth middleware — the single highest-severity bug class in this area.

```ts
// ✅ Optional in the augmentation, non-optional at the point of use
declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}

export function requireUser(req: Request): AuthenticatedUser {
  if (!req.user) throw ApiError.unauthorized();
  return req.user;
}

// One call, no ceremony, and the public routes stay honest:
app.get("/orders", authenticate, (req, res) => {
  const user = requireUser(req);           // AuthenticatedUser
  res.json({ userId: user.userId });
});

app.get("/health", (req, res) => {
  res.json({ ok: true });                  // no user access → no error, correct by construction
});
```

### Mistake 4 — Trying to change an existing member instead of adding one

```ts
// ❌ "process.env.PORT should be a number, let me just fix it"
export {};
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      PORT: number;
      //    ^^^^^^ TS2717: Subsequent property declarations must have the same type.
      //           Property 'PORT' must be of type 'string | undefined',
      //           but here has type 'number'.
    }
  }
}
```

```ts
// ❌ Same category: narrowing an Express member
declare module "express-serve-static-core" {
  interface Request {
    method: "GET" | "POST" | "PUT" | "DELETE";
    //      TS2717 — the original declares `method: string`
  }
}
```

Augmentation is **additive only**. You cannot retype, narrow, widen, or delete an existing member. The workaround is always: add a *new* member with the type you want, and derive it in code.

```ts
// ✅ Add, then convert in your config layer
export {};
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      PORT?: string;              // matches reality — env vars are always strings
    }
  }
}

// src/config.ts — the conversion lives in code, where it belongs
export const config = {
  port: Number.parseInt(process.env.PORT ?? "8080", 10),   // number ✅
} as const;

// ✅ For a narrowed method, add a derived helper rather than retyping:
declare module "express-serve-static-core" {
  interface Request {
    httpMethod?: "GET" | "POST" | "PUT" | "PATCH" | "DELETE";   // NEW member, no conflict
  }
}
```

### Mistake 5 — Shipping an augmentation from a published package

```ts
// ❌ @acme/request-logger/src/index.ts — augments on import, for everyone
import "express-serve-static-core";

declare module "express-serve-static-core" {
  interface Request {
    user?: { id: number };          // generic name + your own shape = guaranteed collision
    log:   Logger;                   // required, but only true if the middleware is mounted
  }
}
```

Consumers get:

```text
error TS2717: Subsequent property declarations must have the same type.
  Property 'user' must be of type '{ id: number; } | undefined',
  but here has type 'AuthenticatedUser | undefined'.
```

...in *their* app, from a package they installed for logging. They cannot fix it. Neither can you without a breaking release.

```ts
// ✅ Fix 1 — namespace the property so collisions are impossible
declare module "express-serve-static-core" {
  interface Request {
    acmeRequestLogger?: Logger;
  }
}

// ✅ Fix 2 — make the augmentation OPT-IN via a separate entrypoint
// src/augment.ts  (only this file augments)
import "express-serve-static-core";
declare module "express-serve-static-core" {
  interface Request { acmeRequestLogger?: Logger }
}
// package.json:
//   "exports": {
//     ".":        { "types": "./dist/index.d.ts",   "default": "./dist/index.js" },
//     "./augment":{ "types": "./dist/augment.d.ts", "default": "./dist/augment.js" }
//   }
// Consumers write, once, in their entrypoint:
//   import "@acme/request-logger/augment";

// ✅ Fix 3 — best: no augmentation at all. Export an accessor.
const LOGGER_KEY = Symbol.for("acme.request-logger");

export function attachLogger(req: Request, logger: Logger): void {
  (req as Record<symbol, unknown>)[LOGGER_KEY] = logger;
}

export function getLogger(req: Request): Logger | undefined {
  return (req as Record<symbol, unknown>)[LOGGER_KEY] as Logger | undefined;
}
// Zero effect on consumer types. Slightly less pretty. Always safe.
```

### Mistake 6 — The augmentation file exists but tsc never loads it

```jsonc
// ❌ tsconfig.json
{
  "compilerOptions": { "strict": true, "types": ["node"] },
  "include": ["src/**/*.ts"]
}
// File is at ./types/express.d.ts — outside the glob → never in the program.
// Symptom: TS2339 'user' does not exist. Editor may or may not agree, depending
// on which tsconfig it picked up.
```

```jsonc
// ✅ Fix A — move it under src/ (do this; it is the least surprising)
//    src/types/express.d.ts

// ✅ Fix B — widen include
{ "include": ["src/**/*.ts", "types/**/*.d.ts"] }

// ✅ Fix C — pin it with `files`, immune to glob edits
{ "files": ["types/express.d.ts"], "include": ["src/**/*.ts"] }
```

```bash
# ✅ Never assume — prove it:
npx tsc -p tsconfig.json --noEmit --listFiles | grep -E 'types/(express|env)'
# no output → the file is not in the program

# Also check you're not chasing an editor/CI mismatch:
#   VS Code → "TypeScript: Go to Project Configuration"
#   and confirm the version:  npx tsc --version  vs  the editor's TS version
```

A related trap: adding `"src/types"` to `typeRoots` and expecting it to help. `typeRoots` entries must be **directories of packages** (`src/types/<pkgname>/index.d.ts`); a bare `.d.ts` file there is ignored. `include` is the mechanism you want.

---

## Practice exercises

### Exercise 1 — easy

Your service currently reads configuration like this, and it has bitten you twice:

```ts
// src/config.ts — current state
export const config = {
  databaseUrl: process.env.DATABASE_URL,        // string | undefined
  redisUrl:    process.env.REDIS_URL,           // string | undefined
  port:        Number(process.env.PORT),        // NaN when unset
  logLevel:    process.env.LOG_LEVEL,           // string | undefined, any value accepted
};
```

Write, from scratch:

1. `src/types/env.d.ts` — an augmentation of `NodeJS.ProcessEnv` declaring:
   - `NODE_ENV` as the literal union `"development" | "test" | "production"` (required)
   - `DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET` as required `string`
   - `PORT` and `SENTRY_DSN` as optional `string`
   - `LOG_LEVEL` as an optional literal union `"debug" | "info" | "warn" | "error"`

   Make sure the file is a **module**, and write a one-line comment explaining which construct achieves that and what would happen without it.

2. `src/config.ts` — rewritten to use the augmentation, exporting a `config` object where `port` is a `number` and `logLevel` has a default. Include an `assertEnv()` function that checks the required variables at boot, with the required-key list typed as `readonly (keyof NodeJS.ProcessEnv)[]` using `satisfies`.

3. Four commented compile errors the augmentation now catches, with their TS error codes.

```ts
// Write your code here
```

### Exercise 2 — medium

Build the complete typed-request setup for an Express service with **API-key** authentication (not JWT) and per-key rate limiting.

Write, from scratch:

1. `src/auth/api-key.ts` — the domain types:
   - `ApiKeyScope = "orders:read" | "orders:write" | "reports:read" | "admin"`
   - `interface ApiKeyPrincipal { keyId: string; tenantId: string; scopes: readonly ApiKeyScope[]; rateLimitPerMinute: number; createdAt: Date }`

2. `src/types/express.d.ts` — a single augmentation of the **correct** module specifier that adds to `Request`:
   - `principal?: ApiKeyPrincipal` (optional — explain in a comment why)
   - `requestId: string` (required — explain in a comment why)
   - `rateLimit?: { remaining: number; resetAt: Date }`
   and adds to `Response`:
   - `sendApiResponse<TBody>(statusCode: number, body: TBody): void`
   - `sendApiError(error: unknown): void`

3. `src/middleware/api-key-auth.ts` — a `RequestHandler` that reads the `x-api-key` header, looks the principal up (stub the lookup), assigns `req.principal`, and 401s otherwise. Plus:
   - a type predicate `hasPrincipal(req: Request): req is Request & { principal: ApiKeyPrincipal }`
   - a `requireScope(...scopes: readonly ApiKeyScope[]): RequestHandler` gate

4. `src/middleware/response-helpers.ts` — the middleware that **implements** the two `Response` methods declared in step 2. Add a comment explaining what happens if this middleware is not mounted, and why the compiler cannot catch it.

5. `src/routes/reports.ts` — two routes using all of the above with zero casts, plus at least five commented compile errors (including one `TS18048` and one `TS2345` on a scope literal).

Finally, in comments: explain why augmenting `"express"` would have silently failed, and give the exact command you would run to prove your `.d.ts` is in the program.

```ts
// Write your code here
```

### Exercise 3 — hard

You maintain `@acme/tenancy`, an internal npm package consumed by nine backend services. It provides Express middleware that resolves a tenant from the subdomain and attaches it to the request, plus a Socket.IO handshake guard.

Three of the nine services already augment `express-serve-static-core`'s `Request` with their own `user?: AuthenticatedUser`. Two of them have a property called `tenant`. Your package must not break any of them.

Deliver all of the following:

1. **The safe published augmentation** — `src/augment.ts` in the package, augmenting `express-serve-static-core` with a **collision-proof** property carrying a type you export. Wire the `package.json` `exports` map so the augmentation is a **separate, opt-in entrypoint** (`@acme/tenancy/augment`) with `types` ordered first, and the main entrypoint carries **no** augmentation.

2. **The zero-augmentation alternative** — `src/context.ts` exposing `attachTenantContext(req, ctx)` and `getTenantContext(req): TenantContext | undefined` backed by a module-level `WeakMap<Request, TenantContext>`, fully typed with no `any`. Explain in comments the two concrete advantages over augmentation and the one ergonomic cost.

3. **The Socket.IO side, done with generics not augmentation** — declare `ClientToServerEvents`, `ServerToClientEvents`, `InterServerEvents`, and `SocketData` (carrying `tenantId`, `keyId`, `scopes`), instantiate `Server<...>` with all four, and write a handshake `io.use(...)` guard that populates `socket.data`. In comments, explain precisely why augmenting `socket.io`'s `Socket` interface to retype `data` produces `TS2717`, and what property of the `Socket<..., SocketData>` declaration causes it.

4. **A conflict post-mortem, in comments.** A consuming service reports:

   ```text
   error TS2717: Subsequent property declarations must have the same type.
     Property 'tenant' must be of type 'TenantContext | undefined',
     but here has type '{ id: string; name: string } | undefined'.
   ```

   Explain: (a) the exact mechanism producing this error, including why interface merging requires type *identity* rather than compatibility; (b) why neither party can fix it unilaterally; (c) three distinct resolutions ranked by preference, with the tradeoff of each; (d) why `skipLibCheck: true` will *not* fix it here even though it suppresses similar-looking errors from `node_modules`.

5. **A consumer-side integration test** proving the augmentation is not merely declared but actually implemented at runtime — booting the real Express app with `supertest` and asserting on the tenant-scoped behaviour of one route.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── AUGMENT A MODULE (file must be a MODULE) ─────────────────────────────────
import "express-serve-static-core";          // ← makes the file a module + loads target
declare module "express-serve-static-core" {
  interface Request  { user?: AuthenticatedUser; requestId: string }
  interface Response { sendApiError(error: unknown): void }
  interface Locals   { tenantId?: string }
}

// ── AUGMENT THE GLOBAL SCOPE (file must be a MODULE) ─────────────────────────
export {};                                    // ← the module marker
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV:     "development" | "test" | "production";
      DATABASE_URL: string;
      PORT?:        string;
    }
  }
  var appStartedAtMs: number;                 // globals must be `var`, not `let`/`const`
}

// ── DECLARE A MODULE WITH NO TYPES (file must be a SCRIPT — no import/export) ─
declare module "@acme/legacy-cache" {
  export function get(key: string): string | null;
}

// ── NARROW `user?` AT THE POINT OF USE ───────────────────────────────────────
export function requireUser(req: Request): AuthenticatedUser {
  if (!req.user) throw ApiError.unauthorized();
  return req.user;
}
export function isAuthenticated(req: Request): req is Request & { user: AuthenticatedUser } {
  return req.user !== undefined;
}
```

| Rule | Detail |
|---|---|
| File must be a **module** to augment | Needs a top-level `import` or `export` (`export {}` suffices) |
| File must be a **script** to declare a new ambient module | No top-level `import`/`export` |
| Specifier must **resolve** | Same resolution as an `import`; typo → `TS2664` in a module file |
| Express | Augment `"express-serve-static-core"`, **not** `"express"` |
| passport installed | Fill the empty global `Express.User` instead |
| `process.env` | `declare global { namespace NodeJS { interface ProcessEnv { … } } }` |
| Only `interface` / `namespace` / `enum` merge | `type` aliases cannot be augmented — sealed |
| Additive only | Add members and overloads; cannot retype, narrow, or remove |
| Same-name member must match **exactly** | Otherwise `TS2717` |
| Generic interfaces | Omit type params, or match arity + names exactly (`TS2428`) |
| Overload priority | Later declarations' overloads are ordered **first** in the merge |
| Augmentation is **global** | Applies program-wide; never ship one from a library without opt-in |
| Declaring ≠ implementing | Mount the middleware that makes the declaration true; cover with a test |
| `?` on `req.user` | Keep it — it is what catches the missing-middleware bug |

| Error | Meaning |
|---|---|
| `TS2339` Property does not exist | Augmentation not loaded, or wrong module specifier |
| `TS2664` Invalid module name in augmentation | Specifier doesn't resolve — and proves the file IS a module |
| `TS2669` Augmentations for the global scope… | `declare global` in a script file — add `export {}` |
| `TS2717` Subsequent property declarations… | Type conflict on a merged member, or duplicate `@types/*` |
| `TS2428` All declarations must have identical type parameters | Generic arity/name mismatch on a merged interface |
| `TS18048` `'req.user' is possibly 'undefined'` | Working as intended — narrow it |

```bash
# ── Diagnostics ──────────────────────────────────────────────────────────────
npx tsc --noEmit --listFiles | grep 'src/types'     # is the .d.ts in the program?
npx tsc --noEmit --skipLibCheck false | head -50    # is my augmentation silently conflicting?
npx tsc --noEmit --traceResolution | grep express   # where did the specifier resolve to?
npm ls @types/node @types/express                   # duplicate @types → TS2717 storms
grep -rn "interface Request" node_modules/@types/express-serve-static-core/index.d.ts
```

---

## Connected topics

- **49 — Declaration files** — `.d.ts` mechanics, `@types/*` resolution, and the same declare-vs-implement asymmetry that makes augmentation a promise rather than a proof.
- **50 — Ambient declarations** — `declare`, `declare global`, and the script-vs-module distinction that decides whether `declare module "x"` augments or replaces.
- **14 — Interfaces** — declaration merging, the reason libraries expose `interface` rather than `type`, and why only `interface` is augmentable.
- **03 — tsconfig in depth** — `include`, `files`, `typeRoots`, `types`, and `skipLibCheck`: everything that decides whether your augmentation file is in the program.
- **22 — Type predicates and narrowing** — `req is Request & { user: AuthenticatedUser }`, the tool that makes an optional augmented property ergonomic.
- **31 — Generics in depth** — type parameters as the alternative to augmentation, and why `socket.io` and `fastify` chose them.
- **44 — Declaration merging** — the general rules for interface, namespace, function, class, and enum merging that module augmentation builds on.
- **56 — Runtime validation with zod** — pairing an augmented `ProcessEnv` or `JwtPayload` with parsing that makes the declaration actually true.
