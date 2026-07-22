# 53 — Extending Express Request

## What is this?

Express hands every handler the same `Request` object, and by the time your route runs, that object has usually been **decorated by middleware**. Your auth middleware verified the `authToken`, loaded the account, and stuck it somewhere:

```js
// The JS reality — this is what every Express codebase does:
req.user = await userRepo.findById(payload.userId);
```

TypeScript's `Request` interface, as declared in `@types/express-serve-static-core`, has no `user` property. So the moment you port that code:

```ts
req.user = await userRepo.findById(payload.userId);
// ❌ Error: Property 'user' does not exist on type 'Request<...>'.
```

**Extending Express Request** is the set of techniques for teaching the compiler about properties that middleware adds at runtime. The canonical one is **declaration merging**: you re-open the `Request` interface from a `.d.ts` file and add fields to it.

```ts
// src/types/express.d.ts
import type { AuthenticatedUser } from "../auth/types";

declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;
      requestId: string;
    }
  }
}

export {};
```

After that file is in the compilation, `req.user` exists on **every** `Request` in the project — typed, autocompleted, checked.

The rest of this doc is about the three hard parts that follow: *why* `declare global { namespace Express }` is the incantation that works, *where* to put the file so `tsc` actually picks it up, and the genuinely difficult design question — the property is **optional** in the type (`AuthenticatedUser | undefined`) but **guaranteed** at runtime inside any route mounted behind `requireAuth`. Reconciling those two facts is where most Express+TS codebases go wrong.

---

## Why does it matter?

Three reasons, in increasing order of importance.

**1. Middleware-added state is the backbone of every real Express app.** Authentication, request IDs, tenant resolution, feature flags, rate-limit counters, DB transactions, the parsed and validated body — all of it gets attached to `req` and read later. If `req.user` is untyped, then the single most security-relevant value in your application (*who is making this request?*) is an `any` flowing through your whole codebase. `req.user.id`, `req.user.orgId`, `req.user.role === "admin"` — none of that is checked. A rename of `role` to `roles` breaks authorization silently.

**2. The alternatives are all worse.** Faced with the error, the untrained instinct is `(req as any).user` — which compiles, propagates `any` into everything downstream, and has to be repeated at every single access site. The second instinct is a custom `AuthenticatedRequest` interface, which works but collides with Express's own handler signatures in ways that produce spectacular error messages. Understanding declaration merging is what lets you pick deliberately rather than flailing.

**3. It's the clearest real-world example of a genuinely unusual TypeScript feature.** Declaration merging lets you **modify a type you don't own, from outside, without touching its source**. Nothing in JavaScript is analogous. It's how `@types/*` packages extend each other, how `express-session` adds `req.session`, how `passport` adds `req.login()`, and how Multer adds `req.file`. Once you see the mechanism, a whole category of "how did that type get there?" questions answers itself.

There's a fourth, quieter payoff. A well-typed `req.user` turns authorization into something the compiler participates in. If `user` is a discriminated union (`{ kind: "user" } | { kind: "service" }`), a handler that reads `req.user.orgId` without narrowing `kind` first is a compile error — a real class of multi-tenant bug caught before it ships.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript ────────────────────────────────────────────────────────────────
// Nothing declares what lives on `req`. The contract is oral tradition.

const express = require("express");
const jwt = require("jsonwebtoken");
const app = express();

// Auth middleware decorates the request:
app.use(async (req, res, next) => {
  const authToken = req.headers.authorization?.replace("Bearer ", "");
  if (!authToken) return next();

  try {
    const payload = jwt.verify(authToken, process.env.JWT_SECRET);
    req.user = await userRepo.findById(payload.sub);   // ← invented out of thin air
    req.currentOrg = await orgRepo.findById(payload.org);
  } catch {
    // swallow — anonymous request
  }
  next();
});

// A route, written six months later by a different person:
app.get("/me/invoices", async (req, res) => {
  // Is it req.user? req.currentUser? req.auth.user? res.locals.user?
  // Only grep knows. And grep finds all four, because all four exist
  // in different corners of this codebase.
  const invoices = await invoiceRepo.findByUser(req.user.id);
  //                                            ^^^^^^^^^^^^
  // TypeError: Cannot read properties of undefined (reading 'id')
  // — because this route was mounted BEFORE the auth middleware,
  //   or because the token was missing and we silently called next().

  res.json(invoices);
});

// A different route, with a subtler bug:
app.get("/me/profile", requireAuth, async (req, res) => {
  res.json({
    userId: req.user.id,
    email: req.user.email,
    isAdmin: req.user.role === "adminstrator",   // typo — always false
    //       ^^^^^^^^^^^^ nothing checks this string. The admin check
    //       silently fails closed today; when someone "fixes" the
    //       typo by renaming the DB value instead, it fails OPEN.
  });
});
```

Three failures, all invisible until production: the property name is undiscoverable, the *presence* of the property is unverifiable, and the *shape* of the property is unchecked.

```ts
// ── TypeScript ────────────────────────────────────────────────────────────────
// One declaration file makes `req.user` real, named, and checked everywhere.

// ═══ src/auth/types.ts ═══════════════════════════════════════════════════════
export type UserRole = "owner" | "admin" | "member" | "readonly";

export interface AuthenticatedUser {
  userId: number;
  email: string;
  orgId: number;
  role: UserRole;
  scopes: readonly string[];
}

// ═══ src/types/express.d.ts ══════════════════════════════════════════════════
import type { AuthenticatedUser } from "../auth/types";

declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;   // optional: not every route is authenticated
      requestId: string;          // required: set by the very first middleware
    }
  }
}

export {};   // makes this file a module so the top-level import is legal

// ═══ src/auth/middleware.ts ══════════════════════════════════════════════════
import type { RequestHandler } from "express";

export const attachUser: RequestHandler = async (req, res, next) => {
  const authToken = req.headers.authorization?.replace("Bearer ", "");
  if (!authToken) { next(); return; }

  const payload = await verifyToken(authToken);
  req.user = {                       // ✅ assignment is checked field-by-field
    userId: payload.sub,
    email:  payload.email,
    orgId:  payload.org,
    role:   payload.role,            // ❌ error if payload.role isn't a UserRole
    scopes: payload.scopes,
  };
  next();
};

// ═══ src/routes/me.routes.ts ═════════════════════════════════════════════════
app.get("/me/profile", requireAuth, (req, res) => {
  req.user.email;
  // ❌ Error: 'req.user' is possibly 'undefined'.
  //    ← the compiler is telling you the literal truth: nothing in the TYPE
  //      system knows requireAuth ran. This is the real problem, and the
  //      whole second half of this doc is about solving it properly.

  res.json({
    userId:  req.user!.userId,
    email:   req.user!.email,
    isAdmin: req.user!.role === "adminstrator",
    // ❌ Error: This comparison appears to be unintentional because the types
    //    'UserRole' and '"adminstrator"' have no overlap.  ← typo caught 🎉
  });
});
```

The satisfying part is the last error. In JavaScript, `req.user.role === "adminstrator"` is a permanently-false authorization check that no test will catch unless someone thought to write it. In TypeScript, with `role` typed as a four-member union, it is a red squiggle the instant you type it.

The *unsatisfying* part — `req.user` being `possibly undefined` even inside a route guarded by `requireAuth` — is not a flaw in the technique. It's TypeScript accurately reporting that middleware ordering is a runtime fact the type system cannot see. Sections **Concept 5** onward are entirely about the four ways to close that gap.

---

## Syntax

```ts
// ═══════════════════════════════════════════════════════════════════════════════
// FORM 1 — global namespace augmentation (the standard, recommended form)
// File: src/types/express.d.ts
// ═══════════════════════════════════════════════════════════════════════════════

import type { AuthenticatedUser } from "../auth/types";

declare global {                 // escape into the global scope from a module file
  namespace Express {            // Express's own global namespace (see Concept 2)
    interface Request {          // merges with the existing Request interface
      user?: AuthenticatedUser;
      requestId: string;
      startedAtMs: number;
    }
    interface Response {         // Response can be augmented the same way
      sendApiError?(status: number, code: string, message: string): void;
    }
  }
}

export {};   // REQUIRED when the file has no other top-level import/export


// ═══════════════════════════════════════════════════════════════════════════════
// FORM 2 — module augmentation of express-serve-static-core (equivalent, explicit)
// File: src/types/express.d.ts
// ═══════════════════════════════════════════════════════════════════════════════

import type { AuthenticatedUser } from "../auth/types";

declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
    requestId: string;
  }
}
// no `export {}` needed — the top-level `import` already makes this a module


// ═══════════════════════════════════════════════════════════════════════════════
// FORM 3 — WRONG: augmenting "express" itself
// ═══════════════════════════════════════════════════════════════════════════════

declare module "express" {
  interface Request { user?: AuthenticatedUser }   // ⚠️ usually does nothing useful
}
// `@types/express` re-exports interfaces that EXTEND the ones in
// express-serve-static-core. Merging here creates a THIRD Request, and the one
// your handlers actually receive is still core's. See Concept 3.


// ═══════════════════════════════════════════════════════════════════════════════
// Getting tsc to include the .d.ts file
// ═══════════════════════════════════════════════════════════════════════════════

// tsconfig.json — option A: let `include` cover it (simplest)
{
  "compilerOptions": { "strict": true, "outDir": "dist", "rootDir": "src" },
  "include": ["src/**/*.ts", "src/**/*.d.ts"]     // ← .d.ts must be reachable
}

// tsconfig.json — option B: name it explicitly with `files` (bulletproof)
{
  "compilerOptions": { "strict": true },
  "files": ["src/types/express.d.ts"],
  "include": ["src/**/*.ts"]
}

// tsconfig.json — option C: typeRoots-style, for a folder of ambient types
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./src/types"]
  }
}
// with src/types/express/index.d.ts  ← note the folder+index.d.ts layout


// ═══════════════════════════════════════════════════════════════════════════════
// The alternatives to declaration merging (all covered later)
// ═══════════════════════════════════════════════════════════════════════════════

// A. Custom request interface
interface AuthenticatedRequest extends Request {
  user: AuthenticatedUser;        // REQUIRED, not optional
}

// B. Typed handler wrapper
type AuthedHandler<P = {}, Res = unknown, B = unknown, Q = {}> = (
  req: Request<P, Res, B, Q> & { user: AuthenticatedUser },
  res: Response<Res>,
  next: NextFunction,
) => Promise<void> | void;

// C. Per-route generic slot (the 6th, undocumented-ish slot doesn't exist —
//    instead intersect the Request type at the call site)
const handler: RequestHandler<{ orderId: string }> = (req, res) => { /* ... */ };
```

---

## How it works — concept by concept

### Concept 1 — Declaration merging: two interfaces with the same name become one

TypeScript's rule: **if two `interface` declarations share a name in the same scope, their members are merged into a single interface.** This is unique to interfaces (and namespaces, and enums) — `type` aliases cannot merge and produce a "Duplicate identifier" error instead.

```ts
// ── The core mechanic, in isolation ────────────────────────────────────────────
interface ApiRequestContext {
  requestId: string;
}

interface ApiRequestContext {          // NOT an error — this merges
  authToken: string;
}

// One interface, both members:
const ctx: ApiRequestContext = {
  requestId: "req_01H8X",
  authToken: "eyJhbGci...",
};

ctx.requestId;   // string ✅
ctx.authToken;   // string ✅
```

```ts
// ── Contrast: type aliases do NOT merge ────────────────────────────────────────
type SessionInfo = { userId: number };
type SessionInfo = { orgId: number };
// ❌ Error: Duplicate identifier 'SessionInfo'.
```

The merge is **member-wise union**, not override. Two declarations of the *same* member must agree exactly, or you get an error:

```ts
interface RequestMeta { retryCount: number }
interface RequestMeta { retryCount: number }   // ✅ identical — allowed
interface RequestMeta { retryCount: string }   // ❌ Error: Subsequent property
                                               //    declarations must have the
                                               //    same type.
```

This is why you cannot "fix" `user?: AuthenticatedUser` into `user: AuthenticatedUser` with a second merge later. Once a member is declared optional in one merged declaration, it is optional, full stop.

Two more merge rules that matter in practice:

```ts
// Methods merge as OVERLOADS, and later declarations win priority order:
interface ApiLogger { log(message: string): void }
interface ApiLogger { log(payload: { message: string; level: string }): void }

// Effective type — overload set, with the LAST-declared file's overloads first:
//   log(payload: { message: string; level: string }): void
//   log(message: string): void
```

```ts
// Merging works ACROSS FILES, and across package boundaries — that's the point.
// node_modules/@types/express-serve-static-core/index.d.ts declares Request.
// YOUR src/types/express.d.ts declares Request again.
// tsc merges them because they resolve to the same declaration space.
```

The mental model: **an interface is not a closed record, it's an open, append-only set of claims about a name.** Every `.d.ts` in the program contributes claims. `tsc` unions them before checking anything.

### Concept 2 — Why `declare global { namespace Express { interface Request } }` works

This is the incantation everybody copies and nobody explains. It has three layers, and each one is load-bearing.

**Layer 1: `declare global` — escaping module scope.**

A `.ts`/`.d.ts` file that contains a top-level `import` or `export` is a **module**. Everything declared inside it is scoped to that module and invisible elsewhere. A file with no top-level import/export is a **script**, and its declarations land directly in the global scope.

```ts
// src/types/express.d.ts
import type { AuthenticatedUser } from "../auth/types";   // ← makes this a MODULE

interface Request {          // ⚠️ this is a LOCAL interface named Request.
  user?: AuthenticatedUser;  //    It merges with nothing. It does nothing.
}
```

`declare global { ... }` is the escape hatch: from inside a module, it says "the following declarations belong to the global scope, not to me." It is **only legal inside a module** — in a script file it's an error, because you're already global.

```ts
import type { AuthenticatedUser } from "../auth/types";

declare global {
  // everything in here lands in the GLOBAL declaration space
}

export {};   // if you ever delete the import above, this keeps the file a module
```

That `export {}` line is not cargo cult. Without any top-level import or export, the file is a script, `declare global` becomes illegal ("Augmentations for the global scope can only be directly nested in external modules or ambient module declarations"), and you get a confusing error.

**Layer 2: `namespace Express` — the global namespace Express deliberately created.**

`@types/express-serve-static-core` contains this, at global scope:

```ts
// node_modules/@types/express-serve-static-core/index.d.ts (simplified)
declare global {
  namespace Express {
    // These three interfaces are INTENTIONALLY EMPTY.
    // They exist purely as extension points for you and for other packages.
    interface Request {}
    interface Response {}
    interface Application {}
  }
}
```

That's the whole trick. The Express typings authors deliberately created a global `Express` namespace containing empty `Request`/`Response`/`Application` interfaces, precisely so that consumers can merge into them. Your `declare global { namespace Express { interface Request { user?: ... } } }` merges with *that* empty interface.

Namespaces merge by the same rule interfaces do — two `namespace Express` blocks anywhere in the program become one namespace, and their same-named interfaces merge.

**Layer 3: how `Express.Request` reaches the `Request` you actually use.**

The core `Request` interface — the one your handler receives — *extends* the global one:

```ts
// node_modules/@types/express-serve-static-core/index.d.ts (simplified)
export interface Request<
  P = ParamsDictionary,
  ResBody = any,
  ReqBody = any,
  ReqQuery = QueryString.ParsedQs,
  LocalsObj extends Record<string, any> = Record<string, any>,
>
  extends http.IncomingMessage,
          Express.Request        // ← ✨ THE LINK ✨
{
  params: P;
  body: ReqBody;
  query: ReqQuery;
  // ...
}
```

So the chain is:

```
your src/types/express.d.ts
        │  declare global { namespace Express { interface Request { user?: … } } }
        ▼
global Express.Request              ← merged: now has `user`
        │  extended by
        ▼
express-serve-static-core.Request<P, ResBody, ReqBody, Query, Locals>
        │  extended by
        ▼
express.Request  (the re-export you import)
        │  passed to
        ▼
your handler's `req` parameter        ← `req.user` resolves ✅
```

Two consequences worth internalising:

1. **Generics don't flow into the global interface.** `Express.Request` is non-generic. So you cannot write `interface Request<T> { user?: T }` in the augmentation — the merge would fail on arity. Whatever you add is the same for every route.
2. **`Express.Response` and `Express.Application` work identically.** `res.locals` typing is separate (it's a generic slot), but adding a custom `res.sendApiError()` method uses the same mechanism.

### Concept 3 — Augmenting `express-serve-static-core` vs augmenting `express`

Form 2 from the Syntax section is module augmentation:

```ts
declare module "express-serve-static-core" {
  interface Request {
    user?: AuthenticatedUser;
  }
}
```

`declare module "some-package" { ... }` inside a module file means "**add these declarations to the existing module `some-package`**." Because `express-serve-static-core` declares `export interface Request<...>` at module scope, your `interface Request` merges directly with it — one hop shorter than going through the global namespace.

Why does the *global* form (Form 1) still work then? Because both routes converge: augmenting global `Express.Request` adds members that `core.Request` inherits via `extends`; augmenting `express-serve-static-core` adds members directly. The `req` object satisfies both.

**Why augmenting `"express"` usually fails.** `@types/express` doesn't define `Request` — it re-declares an interface that extends core's:

```ts
// node_modules/@types/express/index.d.ts (simplified)
import * as core from "express-serve-static-core";

declare namespace e {
  interface Request<P = core.ParamsDictionary, ResBody = any, ReqBody = any, ReqQuery = core.Query>
    extends core.Request<P, ResBody, ReqBody, ReqQuery> {}
  interface Response<ResBody = any> extends core.Response<ResBody> {}
}
```

`Request` there lives inside `declare namespace e`, not at the module's top level. So:

```ts
declare module "express" {
  interface Request { user?: AuthenticatedUser }
  // Creates a NEW top-level `Request` in the express module, unrelated to e.Request.
  // Your handlers receive core.Request (via e.Request), which never sees this.
}
```

It compiles, produces no error, and does nothing — the worst possible failure mode. You then spend an hour wondering why `req.user` is still an error.

There is one wrinkle: **which form works can depend on how you import.** If you write `import { Request } from "express"`, you get `e.Request`, which extends `core.Request`, which extends `Express.Request`. All three augmentation targets that are *reachable through that chain* work. The global form is reachable from everything, which is why it's the recommended default.

```ts
// ── Practical rule ─────────────────────────────────────────────────────────────
// Use the GLOBAL form (Form 1) unless you have a specific reason not to:
//   ✅ works regardless of which package you import Request from
//   ✅ works for Express 4 and Express 5 typings
//   ✅ it's what express-session, passport, and multer all do
//   ✅ survives @types/express reorganisations
//
// Use `declare module "express-serve-static-core"` when:
//   • you want the augmentation scoped to a package build (a library you publish)
//   • you're augmenting something that isn't mirrored into the Express namespace
//     (e.g. adding a method to Router or to the core Response generic)
```

### Concept 4 — Placing the `.d.ts` file and getting `tsc` to include it

This is where more time is lost than anywhere else in this topic. The augmentation is correct; `tsc` simply never reads the file.

**Rule: an augmentation only applies if the declaration file is part of the compilation.** TypeScript does not scan your project for `.d.ts` files "just in case." A file enters the program if it is:

1. matched by `include`/`files` in `tsconfig.json`, **or**
2. imported (directly or transitively) by a file that is, **or**
3. found under a `typeRoots` directory as `<name>/index.d.ts`, **or**
4. referenced by a `/// <reference path="..." />` comment.

Nothing else. In particular, `.d.ts` files are **never** imported by your source — so if `include` doesn't cover them, they're invisible.

```jsonc
// ── Layout A — recommended for applications ────────────────────────────────────
// src/
//   types/
//     express.d.ts          ← the augmentation
//   auth/
//     types.ts
//     middleware.ts
//   app.ts
// tsconfig.json

{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*"],      // ← covers .ts AND .d.ts. This is enough.
  "exclude": ["node_modules", "dist"]
}
```

`"include": ["src/**/*"]` picks up `.ts`, `.tsx`, and `.d.ts` by default. `"include": ["src/**/*.ts"]` **also** matches `.d.ts` (because `express.d.ts` ends in `.ts`) — so both work. What does *not* work:

```jsonc
// ❌ BROKEN — src/types is outside the include glob
{ "include": ["src/routes/**/*.ts", "src/services/**/*.ts"] }

// ❌ BROKEN — a "types" folder at the repo root, with include scoped to src
{ "include": ["src/**/*"] }        // and the file lives at ./types/express.d.ts

// ❌ BROKEN — the exclude wins
{ "include": ["src/**/*"], "exclude": ["**/*.d.ts"] }
```

**Belt and braces: use `files`.** If you want zero ambiguity, name the augmentation explicitly. `files` entries are always added to the program:

```jsonc
{
  "compilerOptions": { "strict": true },
  "files": ["src/types/express.d.ts"],
  "include": ["src/**/*.ts"]
}
```

**The `typeRoots` layout.** If you prefer the `@types`-style convention:

```
src/
  types/
    express/
      index.d.ts        ← note: FOLDER named `express`, file named `index.d.ts`
```

```jsonc
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./src/types"]
  }
}
```

Two traps here. First, once you set `typeRoots`, the default `./node_modules/@types` is **replaced**, not extended — you must list it yourself or lose `@types/node` and `@types/express`. Second, `typeRoots` interacts with the `types` allowlist: if `"types": ["node", "express"]` is present, only those packages load, and your `src/types/express` folder is... actually named `express`, so it may or may not resolve depending on ordering. This layout is fragile. Prefer `include`/`files`.

**Verifying it worked.** Three commands, in order of usefulness:

```bash
# 1. Is the file in the program at all?
npx tsc --listFiles | grep express.d.ts

# 2. What does tsc think req.user is? (create a scratch file and hover, or:)
npx tsc --noEmit

# 3. Full resolution trace when things are truly baffling:
npx tsc --traceResolution | grep -i "express-serve-static-core"
```

If `--listFiles` doesn't show your file, nothing else matters — fix `include` first.

**`ts-node`, `tsx`, `swc`, and `esbuild`.** Runtime transpilers that skip type checking (`tsx`, `esbuild`, `swc`) don't care about your `.d.ts` at all — it's erased. That's fine; type checking is `tsc --noEmit`'s job. But `ts-node` *does* type check by default, and it respects `tsconfig.json`'s `files`/`include` **only** when `--files` is passed:

```bash
# ❌ ts-node ignores `files`/`include` by default — starts from the entry file
#    and follows imports. Your .d.ts is never imported, so it's never loaded.
npx ts-node src/app.ts        # → "Property 'user' does not exist"

# ✅ Tell ts-node to honour tsconfig's file list:
npx ts-node --files src/app.ts
```

Or set it in the config:

```jsonc
{
  "ts-node": { "files": true },
  "compilerOptions": { /* ... */ },
  "include": ["src/**/*"]
}
```

This single flag is responsible for an enormous amount of "it compiles but doesn't run" / "it runs but doesn't compile" confusion.

### Concept 5 — The optional-vs-required `user` problem

Here is the central design tension of this entire topic.

```ts
declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;   // optional — because /health and /login have no user
    }
  }
}
```

The `?` is honest. `Request` is one interface shared by *every* route in the app, and most apps have unauthenticated routes. Declaring `user: AuthenticatedUser` (required) would be a lie that lets you read `req.user.userId` on `/health` and crash.

But the `?` is also *maddening*, because inside a route mounted behind `requireAuth`, the value is guaranteed:

```ts
export const requireAuth: RequestHandler = (req, res, next) => {
  if (!req.user) {
    res.status(401).json({ ok: false, error: "Authentication required", code: "UNAUTHORIZED" });
    return;      // ← the request STOPS here. Downstream handlers cannot run.
  }
  next();        // ← past this line, req.user is definitely an AuthenticatedUser
};

router.get("/me/invoices", requireAuth, async (req, res) => {
  const invoices = await invoiceRepo.findByUser(req.user.userId);
  //                                            ^^^^^^^^
  // ❌ 'req.user' is possibly 'undefined'.
});
```

**Why the compiler can't see it.** `router.get(path, mw1, mw2, handler)` is, to the type system, a function call with several independent function arguments. There is no type-level relationship between "`mw1` returned without calling `next()` when `user` was absent" and "`handler`'s `req` has `user`". Control-flow analysis works *within* a function body; it cannot cross the boundary between two callbacks passed to the same function, let alone model Express's runtime dispatch chain.

Type predicates don't help either. A signature like `(req: Request) => asserts req is AuthenticatedRequest` narrows `req` **at the call site inside the middleware's own body** — the narrowing dies when that function returns.

So you have exactly four options. Here they are with honest trade-offs; the recommendation comes at the end of Concept 9.

**Option 0 — the non-null assertion (`!`).**

```ts
router.get("/me/invoices", requireAuth, async (req, res) => {
  const invoices = await invoiceRepo.findByUser(req.user!.userId);
  //                                                   ^ "trust me"
  res.json({ ok: true, data: invoices });
});
```

- ✅ Zero infrastructure. Works immediately.
- ❌ Repeated at every access. A handler touching `req.user` five times has five `!`s.
- ❌ Silently wrong if someone removes `requireAuth` from the route — the `!` keeps compiling and you get a runtime `TypeError` in production instead of a compile error.
- ❌ Trains everyone on the team to reach for `!` reflexively, which is how `!` ends up on things that really can be null.

**Option 0.5 — narrow once at the top.**

```ts
router.get("/me/invoices", requireAuth, async (req, res) => {
  const { user } = req;
  if (!user) {                                   // one runtime check, then TS knows
    res.status(401).json({ ok: false, error: "Unauthenticated", code: "UNAUTHORIZED" });
    return;
  }
  // `user` is AuthenticatedUser for the rest of the body ✅
  const invoices = await invoiceRepo.findByUser(user.userId);
  res.json({ ok: true, data: invoices, orgId: user.orgId });
});
```

- ✅ No assertions, no lies. Honest narrowing via control flow.
- ✅ If `requireAuth` is removed, the route degrades to a clean 401 instead of a crash.
- ❌ Four boilerplate lines in every authenticated handler.
- ❌ Duplicates `requireAuth`'s job — two places now know how to reject anonymous requests.

This is the *safest* option and a perfectly defensible house style for a small app. The remaining three options are about removing the boilerplate without removing the safety.

### Concept 6 — Alternative A: a custom `AuthenticatedRequest` type

Declare a second interface where `user` is required, and use it on authenticated routes only:

```ts
// src/auth/types.ts
import type { Request } from "express";
import type { ParamsDictionary, Query } from "express-serve-static-core";

// Keep all four Request generic slots so you don't lose params/body/query typing:
export interface AuthenticatedRequest<
  P = ParamsDictionary,
  ResBody = unknown,
  ReqBody = unknown,
  ReqQuery = Query,
> extends Request<P, ResBody, ReqBody, ReqQuery> {
  user: AuthenticatedUser;      // ← REQUIRED here, overriding the optional global
}
```

**Wait — can you do that?** Yes, and it's important to understand why. This is *not* declaration merging (which would error on the conflicting optionality). It's **interface extension with a narrowed member**, and TypeScript allows a derived interface to redeclare an inherited member as long as the new type is *assignable to* the old one. `AuthenticatedUser` is assignable to `AuthenticatedUser | undefined`, so removing the `?` is legal.

```ts
// The same trick with an intersection, if you prefer types to interfaces:
export type AuthenticatedRequest<P = ParamsDictionary, ResBody = unknown, ReqBody = unknown, ReqQuery = Query> =
  Request<P, ResBody, ReqBody, ReqQuery> & { user: AuthenticatedUser };
```

Usage:

```ts
import type { Response, NextFunction } from "express";

// ✅ req.user is AuthenticatedUser — no `!`, no narrowing:
export async function listMyInvoices(
  req: AuthenticatedRequest<{}, ApiResponse<InvoiceDto[]>>,
  res: Response<ApiResponse<InvoiceDto[]>>,
): Promise<void> {
  const invoices = await invoiceRepo.findByUser(req.user.userId, req.user.orgId);
  res.json({ ok: true, data: invoices.map(toInvoiceDto) });
}
```

Now the problem: **registering it.**

```ts
router.get("/me/invoices", requireAuth, listMyInvoices);
// ❌ Error: Argument of type '(req: AuthenticatedRequest<...>, res: ...) => Promise<void>'
//    is not assignable to parameter of type 'RequestHandler<...>'.
//      Types of parameters 'req' and 'req' are incompatible.
//        Property 'user' is optional in type 'Request<...>' but required in
//        type 'AuthenticatedRequest<...>'.
```

Express's `router.get` expects a `RequestHandler`, whose `req` is a plain `Request`. A function demanding a *more specific* parameter is not assignable to one accepting a *less specific* one — that's contravariance, and it's exactly right: Express really could call your handler with a `req` that has no `user`.

Three ways people work around this, in decreasing order of awfulness:

```ts
// ❌ WORST — `as any` at the registration site. Erases everything.
router.get("/me/invoices", requireAuth, listMyInvoices as any);

// ⚠️ BAD — double assertion. Loses no other typing, but still a lie:
router.get("/me/invoices", requireAuth, listMyInvoices as unknown as RequestHandler);

// ⚠️ MEDIOCRE — a single named cast helper, so at least the lie is in one place:
export function authed<P, ResBody, ReqBody, Q>(
  handler: (
    req: AuthenticatedRequest<P, ResBody, ReqBody, Q>,
    res: Response<ResBody>,
    next: NextFunction,
  ) => void | Promise<void>,
): RequestHandler<P, ResBody, ReqBody, Q> {
  // The cast is confined to this one function. Every call site is clean.
  return handler as RequestHandler<P, ResBody, ReqBody, Q>;
}

router.get("/me/invoices", requireAuth, authed(listMyInvoices));   // ✅ compiles
```

That last one is already halfway to Alternative B.

**Trade-offs of `AuthenticatedRequest`:**

- ✅ Handler bodies are completely clean — `req.user.orgId` with no ceremony.
- ✅ Reads well; the type *documents* that the route requires auth.
- ✅ Works with any number of generic slots if you thread them through.
- ❌ Requires a cast at every registration site (or a helper that hides one).
- ❌ The cast is unchecked: nothing stops you registering an `AuthenticatedRequest` handler on a route with **no** `requireAuth` middleware. The safety is by convention, not by the compiler.
- ❌ Multiplies if you have more than one flavour of decoration (`AuthenticatedRequest`, `TenantRequest`, `AuthenticatedTenantRequest`, …).

### Concept 7 — Alternative B: a generic typed handler wrapper

Instead of typing the *request*, type the *registration*. A wrapper function takes a handler that requires `user`, performs the runtime check itself, and returns a plain `RequestHandler` that Express accepts.

```ts
// src/http/authed-handler.ts
import type { Request, Response, NextFunction, RequestHandler } from "express";
import type { ParamsDictionary, Query } from "express-serve-static-core";
import type { AuthenticatedUser } from "../auth/types";
import { ApiError } from "./errors";

// The shape a handler sees once auth is guaranteed:
export type AuthedRequest<P, ResBody, ReqBody, ReqQuery> =
  Request<P, ResBody, ReqBody, ReqQuery> & { user: AuthenticatedUser };

export type AuthedHandler<
  P = ParamsDictionary,
  ResBody = unknown,
  ReqBody = unknown,
  ReqQuery = Query,
> = (
  req: AuthedRequest<P, ResBody, ReqBody, ReqQuery>,
  res: Response<ResBody>,
  next: NextFunction,
) => Promise<void> | void;

/**
 * Wraps a handler that REQUIRES req.user.
 *   • Performs the runtime guarantee itself (401 if absent).
 *   • Narrows the type for the inner handler with ONE cast, here, not at call sites.
 *   • Forwards async rejections to next() — Express 4 doesn't do this for you.
 */
export function withAuth<P, ResBody, ReqBody, ReqQuery>(
  handler: AuthedHandler<P, ResBody, ReqBody, ReqQuery>,
): RequestHandler<P, ResBody, ReqBody, ReqQuery> {
  return (req, res, next): void => {
    if (!req.user) {
      // Runtime check and type narrowing happen at the SAME place. This is the
      // key property: the cast below is justified by the line above it.
      next(ApiError.unauthorized());
      return;
    }

    const authedReq = req as AuthedRequest<P, ResBody, ReqBody, ReqQuery>;
    Promise.resolve(handler(authedReq, res, next)).catch(next);
  };
}
```

Usage — nothing at the call site knows a cast happened:

```ts
interface InvoiceIdParams { invoiceId: string }

const getInvoice: AuthedHandler<InvoiceIdParams, ApiResponse<InvoiceDto>> = async (req, res) => {
  // req.user      → AuthenticatedUser ✅ (no `?`, no `!`)
  // req.params    → InvoiceIdParams   ✅ (generics survived the wrapper)
  // res.json(...) → ApiResponse<InvoiceDto> ✅
  const invoice = await invoiceRepo.findById(req.params.invoiceId);

  if (!invoice)                        throw ApiError.notFound("Invoice");
  if (invoice.orgId !== req.user.orgId) throw ApiError.forbidden();   // tenant isolation ✅

  res.json({ ok: true, data: toInvoiceDto(invoice) });
};

router.get("/invoices/:invoiceId", withAuth(getInvoice));
//                                 ^^^^^^^^ no separate requireAuth needed —
//                                          the wrapper IS the guard.
```

**Why this is better than Alternative A:** the cast and the runtime check live in the same three lines. It is impossible to register an `AuthedHandler` without the 401 guard, because `withAuth` is the only thing that produces a registrable handler from one. The type and the runtime behaviour cannot drift.

You can extend the same pattern to role requirements, and get compile-time-visible authorization:

```ts
export function withRole<P, ResBody, ReqBody, ReqQuery>(
  allowed: readonly UserRole[],
  handler: AuthedHandler<P, ResBody, ReqBody, ReqQuery>,
): RequestHandler<P, ResBody, ReqBody, ReqQuery> {
  return withAuth<P, ResBody, ReqBody, ReqQuery>((req, res, next) => {
    if (!allowed.includes(req.user.role)) {
      next(ApiError.forbidden(`Requires one of: ${allowed.join(", ")}`));
      return;
    }
    return handler(req, res, next);
  });
}

router.delete("/orgs/:orgId/members/:userId", withRole(["owner", "admin"], removeMember));
```

Or make the *scope* requirement part of the type, so a handler declares what it needs:

```ts
export type ScopedHandler<S extends string, P = {}, R = unknown, B = unknown, Q = {}> = (
  req: AuthedRequest<P, R, B, Q> & { user: { scopes: readonly S[] } },
  res: Response<R>,
  next: NextFunction,
) => Promise<void> | void;

export function withScope<S extends string, P, R, B, Q>(
  scope: S,
  handler: ScopedHandler<S, P, R, B, Q>,
): RequestHandler<P, R, B, Q> { /* runtime scope check + cast */ }
```

**Trade-offs:**

- ✅ One cast, in one audited function, next to the runtime check that justifies it.
- ✅ Impossible to forget the guard — you literally can't register the handler otherwise.
- ✅ Composes: `withAuth`, `withRole`, `withScope`, `withTenant` stack cleanly.
- ✅ Doubles as the Express 4 async-rejection wrapper, so you need only one wrapper, not two.
- ❌ Generic inference can need help. TypeScript infers `P`, `ResBody`, etc. from the handler's declared type; if the handler is an inline arrow with no annotation, everything infers as `unknown`. Annotate the handler (`const h: AuthedHandler<Params, Res> = ...`) or pass explicit type arguments (`withAuth<Params, Res, unknown, {}>(...)`).
- ❌ Slightly more indirection for someone reading the route table for the first time.

### Concept 8 — Alternative C: per-route generics (intersect at the call site)

If you only need `user` on a handful of routes, you can skip the extra types entirely and intersect at the point of use.

```ts
import type { RequestHandler } from "express";
import type { AuthenticatedUser } from "../auth/types";

// A tiny alias that adds `user` to any RequestHandler's req:
type WithUser<H> = H extends RequestHandler<infer P, infer R, infer B, infer Q>
  ? (req: Request<P, R, B, Q> & { user: AuthenticatedUser }, res: Response<R>, next: NextFunction) => void
  : never;
```

In practice the simpler, non-conditional version is what people actually write:

```ts
const getMyOrders: RequestHandler<{}, ApiResponse<OrderDto[]>> = async (req, res) => {
  const user = req.user as AuthenticatedUser;   // scoped, local, one line
  const orders = await orderRepo.findByUser(user.userId);
  res.json({ ok: true, data: orders.map(toOrderDto) });
};
```

Or, using the fifth generic slot (`Locals`) instead of `req` — attaching auth state to `res.locals`, which *is* generic per-handler:

```ts
interface AuthLocals extends Record<string, any> {
  user: AuthenticatedUser;
  requestId: string;
}

// The guard populates res.locals:
const requireAuth: RequestHandler<{}, unknown, unknown, {}, AuthLocals> = (req, res, next) => {
  if (!req.user) { res.status(401).json({ ok: false, error: "Unauthenticated" }); return; }
  res.locals.user = req.user;      // ✅ typed via the Locals slot
  next();
};

// The route declares the same Locals type and reads it without any assertion:
const listMyOrders: RequestHandler<{}, ApiResponse<OrderDto[]>, unknown, {}, AuthLocals> =
  async (req, res) => {
    const orders = await orderRepo.findByUser(res.locals.user.userId);   // ✅ no `!`
    res.json({ ok: true, data: orders.map(toOrderDto) });
  };

router.get("/me/orders", requireAuth, listMyOrders);   // ✅ registers with NO cast
```

This is genuinely interesting: because `Locals` is a *generic parameter* rather than a merged global, each handler declares what it expects, and no augmentation is needed at all. And crucially, **it registers without a cast** — `RequestHandler<..., AuthLocals>` is still a `RequestHandler`.

But the safety is illusory in the same way as Alternative A: nothing verifies that `requireAuth` actually ran. Declaring `AuthLocals` on a route is a claim, not a proof. And you've moved the state off `req` (where every Express library expects it — `passport`, `express-session`, request loggers) onto `res`, which reads backwards. See doc **52 — Typing Express routes**, Concept 5, for why `res.locals` doesn't propagate well.

**Trade-offs:**

- ✅ No `.d.ts` file, no build configuration, no global mutation.
- ✅ (Locals variant) No cast needed at registration.
- ✅ Per-route precision — different routes can require different decorations.
- ❌ Verbose: five generic slots on every handler declaration.
- ❌ The claim is unverified in every variant.
- ❌ `res.locals` is a poor home for identity; libraries and middleware conventions all use `req`.
- ❌ Doesn't help third-party middleware that writes to `req.user` (passport does exactly this).

### Concept 9 — Recommendation, and combining the pieces

The techniques are not mutually exclusive. The setup that holds up best in production combines three of them:

```
1. Declaration merging (global Express.Request), with `user?:` OPTIONAL
   → gives every file in the codebase a single, discoverable, correctly-typed
     `req.user`. Optional, because it is genuinely optional on public routes.

2. A `withAuth` wrapper (Alternative B) for every authenticated handler
   → converts the optional into a required, with the runtime 401 check and
     the type narrowing sitting on adjacent lines.

3. Handlers written against `AuthedHandler<...>` (the type from step 2)
   → bodies read `req.user.orgId` with zero assertions and zero boilerplate.
```

Explicitly **not** recommended: making `user` required in the global augmentation. It's tempting, it removes all the friction at once, and it is a lie that will produce a `Cannot read properties of undefined` in a `/health` check or a login route eventually.

```ts
// ❌ Do not do this:
declare global {
  namespace Express {
    interface Request { user: AuthenticatedUser }   // required — lies about /login
  }
}
```

A concrete decision table:

| Situation | Use |
|---|---|
| Small app, few authed routes | Narrow at the top of the handler (Option 0.5) |
| Standard backend, most routes authed | Declaration merging + `withAuth` wrapper |
| Library published to npm | `declare module "express-serve-static-core"`, never global |
| Migrating a large JS codebase incrementally | Declaration merging with `user?:` + `!` at first, tighten later |
| A handful of routes need extra decoration | Per-route `Locals` generics |
| Third-party middleware already writes `req.user` (passport) | Declaration merging is mandatory — you don't control the writer |

---

## Example 1 — basic

```ts
// A minimal but complete setup: one augmentation file, one auth middleware,
// two routes — one public, one authenticated.

// ═══════════════════════════════════════════════════════════════════════════════
// src/auth/types.ts — the domain type that goes on the request
// ═══════════════════════════════════════════════════════════════════════════════

export type UserRole = "owner" | "admin" | "member" | "readonly";

export interface AuthenticatedUser {
  userId: number;
  email:  string;
  orgId:  number;
  role:   UserRole;
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/types/express.d.ts — THE AUGMENTATION
// ═══════════════════════════════════════════════════════════════════════════════

import type { AuthenticatedUser } from "../auth/types";

declare global {
  namespace Express {
    interface Request {
      // Optional: /health and /auth/login are legitimately unauthenticated.
      user?: AuthenticatedUser;

      // Required: assigned by the very first middleware in the chain, so by the
      // time any other code sees a Request, it is always present.
      requestId: string;
    }
  }
}

// Makes this file a module, which is what makes `declare global` legal.
// (The `import type` above already does it — this is belt and braces, and it
//  keeps working if someone deletes the import.)
export {};

// ═══════════════════════════════════════════════════════════════════════════════
// tsconfig.json
// ═══════════════════════════════════════════════════════════════════════════════
// {
//   "compilerOptions": {
//     "target": "ES2022",
//     "module": "commonjs",
//     "strict": true,               ← without this, `user?` never errors anyway
//     "esModuleInterop": true,
//     "skipLibCheck": true,
//     "outDir": "dist",
//     "rootDir": "src"
//   },
//   "include": ["src/**/*"],        ← MUST cover src/types/express.d.ts
//   "ts-node": { "files": true }    ← MUST be set if you run with ts-node
// }

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/request-id.ts — the middleware that satisfies `requestId: string`
// ═══════════════════════════════════════════════════════════════════════════════

import { randomUUID } from "node:crypto";
import type { RequestHandler } from "express";

// Register this FIRST, before anything reads req.requestId.
export const attachRequestId: RequestHandler = (req, res, next): void => {
  const incoming = req.headers["x-request-id"];
  req.requestId = typeof incoming === "string" ? incoming : randomUUID();
  //  ^^^^^^^^^ ✅ exists thanks to the augmentation; typed string
  res.setHeader("x-request-id", req.requestId);
  next();
};

// ═══════════════════════════════════════════════════════════════════════════════
// src/auth/middleware.ts — decorate the request with `user`
// ═══════════════════════════════════════════════════════════════════════════════

import type { RequestHandler } from "express";
import type { AuthenticatedUser, UserRole } from "./types";

interface TokenPayload {
  sub:   number;
  email: string;
  org:   number;
  role:  string;      // untrusted — comes off a JWT
}

const VALID_ROLES: readonly UserRole[] = ["owner", "admin", "member", "readonly"];

function isUserRole(value: string): value is UserRole {
  return (VALID_ROLES as readonly string[]).includes(value);
}

// Populates req.user when a valid token is present. Does NOT reject — that's
// requireAuth's job. This split lets routes be optionally-authenticated.
export const attachUser: RequestHandler = async (req, res, next): Promise<void> => {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) { next(); return; }

  const authToken = header.slice("Bearer ".length);

  try {
    const payload = await verifyJwt<TokenPayload>(authToken);
    if (!isUserRole(payload.role)) {
      // The type system forced us to validate the role string. In JS this
      // would have quietly become an unknown role that fails every check.
      next(); return;
    }

    req.user = {                      // ✅ checked against AuthenticatedUser
      userId: payload.sub,
      email:  payload.email,
      orgId:  payload.org,
      role:   payload.role,           // ✅ narrowed to UserRole by isUserRole
    };
  } catch {
    // Invalid/expired token → treat as anonymous.
  }

  next();
};

// Rejects anonymous requests. After this runs, req.user is guaranteed at
// RUNTIME — but the TYPE still says `AuthenticatedUser | undefined`.
export const requireAuth: RequestHandler = (req, res, next): void => {
  if (!req.user) {
    res.status(401).json({ ok: false, error: "Authentication required", code: "UNAUTHORIZED" });
    return;
  }
  next();
};

// ═══════════════════════════════════════════════════════════════════════════════
// src/routes/me.routes.ts — reading the decorated request
// ═══════════════════════════════════════════════════════════════════════════════

import { Router } from "express";
import type { RequestHandler } from "express";

type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; code: string };

interface ProfileDto {
  userId:      number;
  email:       string;
  orgId:       number;
  role:        UserRole;
  requestId:   string;
}

const router: Router = Router();

// ── Public route: no user expected, and the type reflects that ─────────────────
const health: RequestHandler<{}, { status: string; requestId: string }> = (req, res): void => {
  res.json({ status: "ok", requestId: req.requestId });   // ✅ requestId is required
  // req.user.email  → ❌ 'req.user' is possibly 'undefined'  ← correct and useful
};

// ── Authenticated route: narrow once at the top (Option 0.5 from Concept 5) ────
const getProfile: RequestHandler<{}, ApiResponse<ProfileDto>> = (req, res): void => {
  const { user } = req;
  if (!user) {
    // Defensive: requireAuth should have caught this, but the type demands it
    // and the check costs nothing.
    res.status(401).json({ ok: false, error: "Authentication required", code: "UNAUTHORIZED" });
    return;
  }

  // From here on, `user` is AuthenticatedUser — narrowed by control flow.
  res.json({
    ok: true,
    data: {
      userId:    user.userId,
      email:     user.email,
      orgId:     user.orgId,
      role:      user.role,
      requestId: req.requestId,
    },
  });
};

// ── Optionally-authenticated route: the union is the POINT here ────────────────
const listArticles: RequestHandler<{}, ApiResponse<ArticleDto[]>> = async (req, res): Promise<void> => {
  // Anonymous visitors see public articles; signed-in users also see their org's.
  const articles = req.user
    ? await articleRepo.listForOrg(req.user.orgId)   // ✅ narrowed inside the branch
    : await articleRepo.listPublic();

  res.json({ ok: true, data: articles.map(toArticleDto) });
};

router.get("/health",   health);
router.get("/me",       requireAuth, getProfile);
router.get("/articles", listArticles);            // no requireAuth — optional auth

export default router;

// ═══════════════════════════════════════════════════════════════════════════════
// src/app.ts — ORDER IS EVERYTHING
// ═══════════════════════════════════════════════════════════════════════════════

import express from "express";
import type { Application } from "express";

export function createApp(): Application {
  const app = express();

  app.use(express.json());
  app.use(attachRequestId);    // 1. satisfies `requestId: string` for everything below
  app.use(attachUser);         // 2. populates req.user when a token is present
  app.use("/api", router);     // 3. routes — requireAuth applied per-route

  return app;
}
```

---

## Example 2 — real world backend use case

```ts
// A production-shaped multi-tenant API. Declaration merging supplies the shared
// vocabulary; a `withAuth` wrapper converts optional → required with a runtime
// check; role and tenant checks compose on top. Express 4 typings throughout,
// with Express 5 notes inline.

// ═══════════════════════════════════════════════════════════════════════════════
// src/auth/types.ts
// ═══════════════════════════════════════════════════════════════════════════════

export type UserRole = "owner" | "admin" | "member" | "readonly";
export type ApiScope = "orders:read" | "orders:write" | "invoices:read" | "users:admin";

// A discriminated union — human sessions and machine tokens are NOT the same
// thing, and the compiler will now force you to distinguish them.
export type Principal =
  | {
      kind:   "user";
      userId: number;
      email:  string;
      orgId:  number;
      role:   UserRole;
      scopes: readonly ApiScope[];
    }
  | {
      kind:      "service";
      serviceId: string;
      orgId:     number;
      scopes:    readonly ApiScope[];
    };

export interface RequestTenant {
  orgId: number;
  slug:  string;
  plan:  "free" | "pro" | "enterprise";
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/types/express.d.ts — the single augmentation for the whole app
// ═══════════════════════════════════════════════════════════════════════════════

import type { Principal, RequestTenant } from "../auth/types";

declare global {
  namespace Express {
    interface Request {
      /** Set by attachPrincipal when a valid credential is presented. */
      user?: Principal;

      /** Set by resolveTenant on routes under /orgs/:orgSlug. */
      tenant?: RequestTenant;

      /** Set by attachRequestId — the first middleware, so always present. */
      requestId: string;

      /** Set by attachRequestId — used for latency logging. */
      startedAtMs: number;

      /** Populated by rate-limit middleware; absent on unmetered routes. */
      rateLimit?: { remaining: number; resetAtMs: number };
    }

    interface Response {
      /** Convenience helper installed by the responseHelpers middleware. */
      apiError?(status: number, code: string, message: string): void;
    }
  }
}

export {};

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/types.ts
// ═══════════════════════════════════════════════════════════════════════════════

import type { Request, Response, NextFunction, RequestHandler } from "express";
import type { ParamsDictionary, Query } from "express-serve-static-core";
import type { Principal, RequestTenant } from "../auth/types";

export type ErrorCode =
  | "VALIDATION_FAILED" | "UNAUTHORIZED" | "FORBIDDEN"
  | "NOT_FOUND" | "CONFLICT" | "RATE_LIMITED" | "INTERNAL";

export type ApiResponse<T> =
  | { ok: true;  data: T; meta?: Record<string, unknown> }
  | { ok: false; error: string; code: ErrorCode; requestId: string };

// ── The narrowed request shapes ────────────────────────────────────────────────
// Intersections, not interfaces, so they compose: you can ask for a request that
// is both authenticated AND tenant-resolved without declaring a third interface.

export type AuthedRequest<P, ResBody, ReqBody, ReqQuery> =
  Request<P, ResBody, ReqBody, ReqQuery> & { user: Principal };

export type TenantRequest<P, ResBody, ReqBody, ReqQuery> =
  AuthedRequest<P, ResBody, ReqBody, ReqQuery> & { tenant: RequestTenant };

// ── The handler types that go with them ────────────────────────────────────────

export type AuthedHandler<
  P        = ParamsDictionary,
  Data     = unknown,
  ReqBody  = unknown,
  ReqQuery = Query,
> = (
  req:  AuthedRequest<P, ApiResponse<Data>, ReqBody, ReqQuery>,
  res:  Response<ApiResponse<Data>>,
  next: NextFunction,
) => Promise<void> | void;

export type TenantHandler<
  P        = ParamsDictionary,
  Data     = unknown,
  ReqBody  = unknown,
  ReqQuery = Query,
> = (
  req:  TenantRequest<P, ApiResponse<Data>, ReqBody, ReqQuery>,
  res:  Response<ApiResponse<Data>>,
  next: NextFunction,
) => Promise<void> | void;

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/errors.ts
// ═══════════════════════════════════════════════════════════════════════════════

export class ApiError extends Error {
  constructor(
    readonly statusCode: number,
    readonly code: ErrorCode,
    message: string,
  ) {
    super(message);
    this.name = "ApiError";
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
  static badRequest(message: string): ApiError {
    return new ApiError(400, "VALIDATION_FAILED", message);
  }
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/wrappers.ts — THE CENTRAL PIECE
// Every unchecked cast in this application lives in this one file.
// ═══════════════════════════════════════════════════════════════════════════════

import type { RequestHandler } from "express";
import { ApiError } from "./errors";
import type {
  ApiResponse, AuthedHandler, AuthedRequest, TenantHandler, TenantRequest,
} from "./types";
import type { ApiScope, UserRole } from "../auth/types";

/**
 * withAuth — the optional→required converter.
 *
 * The runtime guarantee (the `if (!req.user)` bail-out) and the type assertion
 * (the `as AuthedRequest`) are three lines apart. That adjacency is the whole
 * argument for this pattern: you can audit its correctness at a glance, and it
 * is IMPOSSIBLE to register an AuthedHandler without going through here.
 *
 * It also doubles as the Express 4 async-rejection wrapper.
 */
export function withAuth<P, Data, ReqBody, ReqQuery>(
  handler: AuthedHandler<P, Data, ReqBody, ReqQuery>,
): RequestHandler<P, ApiResponse<Data>, ReqBody, ReqQuery> {
  return (req, res, next): void => {
    if (!req.user) {
      next(ApiError.unauthorized());
      return;
    }

    const authedReq = req as AuthedRequest<P, ApiResponse<Data>, ReqBody, ReqQuery>;

    // Promise.resolve turns a synchronous throw into a rejection too.
    // Express 5 does this natively — there you could `return handler(...)`.
    Promise.resolve(handler(authedReq, res, next)).catch(next);
  };
}

/** withRole — layered on withAuth, so the auth guarantee is never bypassed. */
export function withRole<P, Data, ReqBody, ReqQuery>(
  allowed: readonly UserRole[],
  handler: AuthedHandler<P, Data, ReqBody, ReqQuery>,
): RequestHandler<P, ApiResponse<Data>, ReqBody, ReqQuery> {
  return withAuth<P, Data, ReqBody, ReqQuery>((req, res, next) => {
    // Principal is a discriminated union — the compiler FORCES this check.
    // Without it, `req.user.role` is an error: `role` doesn't exist on the
    // "service" branch. That is a real multi-tenant bug being prevented.
    if (req.user.kind !== "user") {
      next(ApiError.forbidden("This endpoint requires a user credential"));
      return;
    }
    if (!allowed.includes(req.user.role)) {
      next(ApiError.forbidden(`Requires role: ${allowed.join(" | ")}`));
      return;
    }
    return handler(req, res, next);
  });
}

/** withScope — works for both user and service principals. */
export function withScope<P, Data, ReqBody, ReqQuery>(
  scope: ApiScope,
  handler: AuthedHandler<P, Data, ReqBody, ReqQuery>,
): RequestHandler<P, ApiResponse<Data>, ReqBody, ReqQuery> {
  return withAuth<P, Data, ReqBody, ReqQuery>((req, res, next) => {
    if (!req.user.scopes.includes(scope)) {
      next(ApiError.forbidden(`Missing scope: ${scope}`));
      return;
    }
    return handler(req, res, next);
  });
}

/** withTenant — requires BOTH user and tenant. Note the composed intersection. */
export function withTenant<P, Data, ReqBody, ReqQuery>(
  handler: TenantHandler<P, Data, ReqBody, ReqQuery>,
): RequestHandler<P, ApiResponse<Data>, ReqBody, ReqQuery> {
  return withAuth<P, Data, ReqBody, ReqQuery>((req, res, next) => {
    if (!req.tenant) {
      next(ApiError.badRequest("Tenant could not be resolved for this route"));
      return;
    }
    // req is already AuthedRequest; adding the tenant guarantee is one more step.
    return handler(
      req as TenantRequest<P, ApiResponse<Data>, ReqBody, ReqQuery>,
      res,
      next,
    );
  });
}

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/middleware.ts — the writers
// ═══════════════════════════════════════════════════════════════════════════════

import { randomUUID } from "node:crypto";
import type { RequestHandler } from "express";

// MUST be first: everything below assumes requestId/startedAtMs are populated,
// and the augmentation declares them as REQUIRED (no `?`).
export const attachRequestId: RequestHandler = (req, res, next): void => {
  const incoming = req.headers["x-request-id"];
  req.requestId   = typeof incoming === "string" ? incoming : randomUUID();
  req.startedAtMs = Date.now();
  res.setHeader("x-request-id", req.requestId);
  next();
};

export const attachPrincipal: RequestHandler = async (req, res, next): Promise<void> => {
  const header = req.headers.authorization;

  try {
    if (header?.startsWith("Bearer ")) {
      const claims = await verifyJwt(header.slice(7));
      req.user = {
        kind:   "user",
        userId: claims.sub,
        email:  claims.email,
        orgId:  claims.org,
        role:   claims.role,
        scopes: claims.scopes,
      };
    } else if (header?.startsWith("ApiKey ")) {
      const key = await apiKeyRepo.verify(header.slice(7));
      if (key) {
        req.user = {
          kind:      "service",
          serviceId: key.serviceId,
          orgId:     key.orgId,
          scopes:    key.scopes,
        };
        // Note: no `role`, no `email`. The union makes it impossible to read
        // them without narrowing on `kind` first.
      }
    }
  } catch {
    // Invalid credential → anonymous. requireAuth/withAuth will 401.
  }

  next();
};

// Resolves :orgSlug into req.tenant, and enforces that the principal belongs to it.
export const resolveTenant: RequestHandler<{ orgSlug: string }> = async (req, res, next) => {
  const org = await orgRepo.findBySlug(req.params.orgSlug);
  if (!org) { next(ApiError.notFound("Organisation")); return; }

  // Tenant isolation: a valid token for org A must not read org B.
  // `req.user` is `Principal | undefined` — both branches carry `orgId`, so
  // this reads without narrowing on `kind`. That's the union working for us.
  if (req.user && req.user.orgId !== org.orgId) {
    next(ApiError.forbidden("Credential does not belong to this organisation"));
    return;
  }

  req.tenant = { orgId: org.orgId, slug: org.slug, plan: org.plan };
  next();
};

// ═══════════════════════════════════════════════════════════════════════════════
// src/routes/orders.routes.ts — handlers with zero assertions
// ═══════════════════════════════════════════════════════════════════════════════

import { Router } from "express";

interface OrderDto {
  orderId:    string;
  orgId:      number;
  status:     "pending" | "paid" | "shipped" | "cancelled";
  totalCents: number;
  createdBy:  string;
}

interface OrderIdParams   { orderId: string }
interface OrgSlugParams   { orgSlug: string }
interface ListOrdersQuery { status?: OrderDto["status"]; page?: string }
interface CreateOrderBody { items: Array<{ sku: string; qty: number }>; note?: string }

// ── GET /orgs/:orgSlug/orders ──────────────────────────────────────────────────
const listOrders: TenantHandler<OrgSlugParams, OrderDto[], unknown, ListOrdersQuery> =
  async (req, res) => {
    // req.user    → Principal          (required — no `?`, no `!`) ✅
    // req.tenant  → RequestTenant      (required) ✅
    // req.params  → OrgSlugParams      ✅ generics survived both wrappers
    // req.query   → ListOrdersQuery    ✅
    const page = Math.max(Number(req.query.page ?? "1"), 1);

    const rows = await orderRepo.list({
      orgId:  req.tenant.orgId,          // scoped by tenant, not by client input
      status: req.query.status,
      offset: (page - 1) * 25,
      limit:  25,
    });

    res.json({ ok: true, data: rows.map(toOrderDto), meta: { page } });
  };

// ── GET /orgs/:orgSlug/orders/:orderId ─────────────────────────────────────────
const getOrder: TenantHandler<OrgSlugParams & OrderIdParams, OrderDto> = async (req, res) => {
  const order = await orderRepo.findById(req.params.orderId);
  if (!order) throw ApiError.notFound("Order");

  // Defence in depth: the row itself must belong to the resolved tenant.
  if (order.orgId !== req.tenant.orgId) throw ApiError.notFound("Order");

  res.json({ ok: true, data: toOrderDto(order) });
};

// ── POST /orgs/:orgSlug/orders ─────────────────────────────────────────────────
const createOrder: TenantHandler<OrgSlugParams, OrderDto, CreateOrderBody> = async (req, res) => {
  if (req.body.items.length === 0) throw ApiError.badRequest("Order requires at least one item");

  // The discriminated union forces us to say who created it, per principal kind:
  const createdBy =
    req.user.kind === "user" ? req.user.email : `service:${req.user.serviceId}`;
  //                            ^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //  Both branches type-check ONLY because we narrowed on `kind` first.
  //  `req.user.email` outside this narrowing is a compile error. ✅

  const order = await orderRepo.create({
    orgId: req.tenant.orgId,
    items: req.body.items,
    note:  req.body.note,
    createdBy,
  });

  res.status(201).json({ ok: true, data: toOrderDto(order) });
};

// ── DELETE /orgs/:orgSlug/orders/:orderId — admins only ────────────────────────
const cancelOrder: AuthedHandler<OrderIdParams, OrderDto> = async (req, res) => {
  const order = await orderRepo.findById(req.params.orderId);
  if (!order)                          throw ApiError.notFound("Order");
  if (order.orgId !== req.user.orgId)  throw ApiError.notFound("Order");
  if (order.status === "shipped")      throw new ApiError(409, "CONFLICT", "Already shipped");

  const cancelled = await orderRepo.updateStatus(order.orderId, "cancelled");
  res.json({ ok: true, data: toOrderDto(cancelled) });
};

// ── Registration — every route states its requirements in one visible token ────
const orderRouter: Router = Router({ mergeParams: true });   // needs :orgSlug from parent

orderRouter.get("/",                withTenant(listOrders));
orderRouter.get("/:orderId",        withTenant(getOrder));
orderRouter.post("/",               withScope("orders:write", createOrder as AuthedHandler<any, any, any, any>));
orderRouter.delete("/:orderId",     withRole(["owner", "admin"], cancelOrder));

export default orderRouter;

// ═══════════════════════════════════════════════════════════════════════════════
// src/http/error-handler.ts — reads req.requestId, which the augmentation
// declares as REQUIRED, so no narrowing is needed here either.
// ═══════════════════════════════════════════════════════════════════════════════

import type { ErrorRequestHandler } from "express";

export const errorHandler: ErrorRequestHandler = (err, req, res, next): void => {
  if (res.headersSent) { next(err); return; }

  const durationMs = Date.now() - req.startedAtMs;   // ✅ required field, no `!`

  if (err instanceof ApiError) {
    console.warn("api_error", {
      requestId: req.requestId,
      // Careful: req.user is optional HERE — the error handler runs for
      // unauthenticated failures too. The compiler makes you handle it.
      principal: req.user?.kind === "user" ? req.user.email : req.user?.kind ?? "anonymous",
      code:      err.code,
      durationMs,
    });
    res.status(err.statusCode).json({
      ok:        false,
      error:     err.message,
      code:      err.code,
      requestId: req.requestId,
    });
    return;
  }

  console.error("unhandled_error", { requestId: req.requestId, durationMs, err });
  res.status(500).json({
    ok:        false,
    error:     "Internal Server Error",
    code:      "INTERNAL",
    requestId: req.requestId,
  });
};

// ═══════════════════════════════════════════════════════════════════════════════
// src/app.ts — assembly. The middleware ORDER is what makes the required
// fields in the augmentation truthful.
// ═══════════════════════════════════════════════════════════════════════════════

import express from "express";
import type { Application } from "express";

export function createApp(): Application {
  const app = express();

  app.use(express.json({ limit: "1mb" }));
  app.use(attachRequestId);      // 1. MUST be first — requestId/startedAtMs are required
  app.use(attachPrincipal);      // 2. optional user
  app.use("/api/orgs/:orgSlug", resolveTenant);   // 3. optional tenant, scoped by path
  app.use("/api/orgs/:orgSlug/orders", orderRouter);
  app.use(errorHandler);         // 4. last, four params

  return app;
}

// ── Express 5 notes for this file ──────────────────────────────────────────────
// • Async rejections are forwarded automatically, so `withAuth` could `return
//   handler(...)` instead of `Promise.resolve(...).catch(next)`. Keeping the
//   wrapper is harmless and keeps the code portable.
// • `@types/express@5` still exposes the same `declare global { namespace Express }`
//   extension point — the augmentation file needs no changes at all.
// • `req.query` is a lazy getter in Express 5 and the default parser is "simple";
//   ListOrdersQuery is unaffected because it has no nested keys.
// • Path patterns changed: "/orgs/:orgSlug" still works, but "*" must be "/*splat"
//   and optional params are ":name?" → "{/:name}".
```

---

## Going deeper

### An interface augmentation is global and unconditional — there is no scoping

Once your `.d.ts` is in the program, **every** `Request` in **every** file has `user`. There is no way to say "only in the routes folder." That has two consequences.

First, the augmentation becomes part of your project's public vocabulary, so name things carefully. `req.user` is fine. `req.data` or `req.ctx` is a landmine — the next library you install may want the same name.

Second, and worse: **two packages augmenting the same property with different types is a hard error you cannot fix from your code.**

```ts
// @types/passport declares:
declare global {
  namespace Express {
    interface User {}                 // an empty extension-point interface
    interface Request { user?: User } // ← passport claims `user`
  }
}

// Your src/types/express.d.ts:
declare global {
  namespace Express {
    interface Request { user?: AuthenticatedUser }
  }
}
// ❌ Error: Subsequent property declarations must have the same type.
//    Property 'user' must be of type 'User | undefined', but here has type
//    'AuthenticatedUser | undefined'.
```

The fix is to augment `Express.User` — the extension point passport deliberately provided — rather than redeclaring `Request.user`:

```ts
// ✅ src/types/express.d.ts — cooperate with passport's design
import type { UserRole } from "../auth/types";

declare global {
  namespace Express {
    interface User {                 // merges into passport's empty interface
      userId: number;
      email:  string;
      orgId:  number;
      role:   UserRole;
    }
  }
}

export {};

// req.user is now `Express.User | undefined` with YOUR members. ✅
```

The general lesson: before augmenting, check whether the library already declares an empty interface for the purpose. `express-session` provides `SessionData`; `multer` provides `Express.Multer.File`; `passport` provides `Express.User`. Merging into their extension point is always better than fighting over `Request`.

### The augmentation is erased — it changes nothing at runtime

```ts
declare global {
  namespace Express {
    interface Request { requestId: string }   // "required"
  }
}
```

Compiles to **nothing**. There is no runtime check, no default value, no initialisation. If you forget to register `attachRequestId`, `req.requestId` is `undefined` at runtime while the type insists it's a `string`:

```ts
const handler: RequestHandler = (req, res) => {
  res.setHeader("x-request-id", req.requestId);
  // TypeError: Invalid value "undefined" for header "x-request-id"
  // ...even though `req.requestId` has type `string` and strictNullChecks is on.
};
```

Declaring a member as required is a **promise you make to the compiler**, and the compiler has no way to verify it. The rule of thumb: only make a field required if it is set by a middleware registered *before every route*, at the very top of `app.use` chain, unconditionally. Everything conditional gets `?`.

A useful hardening trick — make the writer the only thing that can satisfy the type, and check it once in development:

```ts
// src/http/assert-decorations.ts — dev-only sanity middleware, registered
// immediately after the decorators in non-production builds.
export const assertDecorations: RequestHandler = (req, res, next): void => {
  if (typeof req.requestId !== "string") {
    throw new Error("attachRequestId middleware is not registered — req.requestId is missing");
  }
  next();
};
```

### `strict` / `strictNullChecks` is what makes any of this useful

With `"strict": false` (or `strictNullChecks: false`), `user?: AuthenticatedUser` gives you a type of `AuthenticatedUser` that happens to be optional in object literals — reading `req.user.email` produces **no error at all**. The entire optional-vs-required discussion evaporates, and so does the safety.

```jsonc
// Without this, `req.user.email` compiles on a route with no auth middleware:
{ "compilerOptions": { "strict": true } }
```

If you're migrating a JS codebase incrementally and can't turn on `strict` yet, at minimum turn on `strictNullChecks` — it's the flag that makes `?` mean something.

### Interfaces merge; `type` aliases and generics do not

Three related limitations that surprise people:

```ts
// 1. You cannot use a type alias for the augmentation:
declare global {
  namespace Express {
    type Request = { user?: AuthenticatedUser };
    // ❌ Error: Duplicate identifier 'Request'.
  }
}

// 2. You cannot make the augmentation generic:
declare global {
  namespace Express {
    interface Request<TUser> { user?: TUser }
    // ❌ Error: All declarations of 'Request' must have identical type parameters.
    //    (Express.Request is declared with zero type parameters.)
  }
}

// 3. You cannot change the optionality or type of an existing member:
declare global {
  namespace Express {
    interface Request { body: MyBody }
    // ❌ Error: Subsequent property declarations must have the same type.
    //    `body` is declared as ReqBody (a generic parameter) in core.Request.
  }
}
```

Limitation 2 is why "one global `req.user` type for the whole app" is unavoidable with this technique. If different route groups genuinely need different principal shapes, model it as a **discriminated union** on one type (as Example 2 does with `Principal`) rather than trying to parameterise the interface.

### Declaration merging in a monorepo, and in published libraries

If you publish an Express middleware to npm, **never** use `declare global`. A global augmentation from a dependency silently rewrites `Request` for every consumer, and if two dependencies do it with conflicting types, the consumer's build breaks with an error in `node_modules` that they cannot fix.

```ts
// ✅ In a published library — augment the module, and DOCUMENT it:
declare module "express-serve-static-core" {
  interface Request {
    /** Set by `@acme/rate-limit`. Present only on routes the limiter guards. */
    acmeRateLimit?: { remaining: number; resetAtMs: number };
  }
}
```

Two conventions that avoid collisions: namespace-prefix your property (`acmeRateLimit`, not `rateLimit`), and make it optional always, since consumers control middleware ordering, not you.

In a **monorepo**, an augmentation in `packages/api/src/types/express.d.ts` applies to every package compiled in the same `tsc` program. With project references (`composite: true`), each package is its own program — so `packages/worker` will **not** see `packages/api`'s augmentation unless it's exported from a shared package that `worker` references. The usual fix is a `packages/http-types` package whose `index.d.ts` holds the augmentation, imported (for side effects) by every consumer:

```ts
// packages/api/src/app.ts
import "@acme/http-types";     // side-effect import pulls the augmentation in
```

### Why `export {}` and not `export default {}`

Any top-level `import` or `export` makes the file a module. `export {}` is the minimal, zero-cost way of saying so — it exports nothing, emits nothing, and doesn't pollute the module's shape. `export default {}` would work but declares a real runtime value in a file that's supposed to be types-only, which some bundlers will then try to emit.

If your `.d.ts` already has `import type { ... }` at the top, `export {}` is redundant — but it's cheap insurance against someone later deleting the last import and being greeted by:

```
error TS2669: Augmentations for the global scope can only be directly nested
in external modules or ambient module declarations.
```

That message is the number-one symptom of a `.d.ts` that has accidentally become a script.

### `req.user` and `res.locals` are not interchangeable

`res.locals` looks like a rival home for request state, and Express's own docs suggest it. Three reasons `req` wins for identity:

1. **Direction.** The user is an attribute *of the request*. Putting it on the response is semantically backwards, and it reads badly in every log line and helper signature.
2. **Ecosystem.** `passport`, `express-jwt`, `express-openid-connect`, and every tutorial write to `req.user`. If any of them is in your stack, you're merging into `Request` regardless.
3. **Propagation.** The `Locals` generic slot isn't inherited between middleware — every handler restates it (see **52 — Typing Express routes**, Concept 5). `req.user` via merging is declared once.

`res.locals` remains genuinely good for **view-render data** (its original purpose) and for per-response values that only the response pipeline cares about, like a computed `Cache-Control` policy.

### Overwriting `req.body` after validation

A related and very common augmentation-adjacent trick: validation middleware that replaces `req.body` with the parsed, typed value. You *cannot* merge a new `body` type (limitation 3 above), because `body` is a generic slot on `core.Request`. Instead, use the generic slot:

```ts
export function validateBody<T>(schema: { parse(input: unknown): T }): RequestHandler<
  ParamsDictionary, unknown, T
> {
  return (req, res, next): void => {
    try {
      req.body = schema.parse(req.body);   // req.body is typed T on the way out
      next();
    } catch {
      next(ApiError.badRequest("Request body failed validation"));
    }
  };
}
```

The downstream handler must independently declare the same `T` in its own `ReqBody` slot — the type does not flow between middleware any more than `Locals` does. That is a real limitation of Express's typing model, and the reason frameworks like tRPC, Fastify (with type providers) and Hono exist.

### `skipLibCheck` and augmentation errors

`skipLibCheck: true` suppresses type errors *inside* `.d.ts` files — but **not** errors caused by conflicting merges, because those are reported at the merge site. If you see "Subsequent property declarations must have the same type" pointing into `node_modules`, `skipLibCheck` won't silence it. The only fixes are: rename your property, augment the library's intended extension point, or (last resort) pin the conflicting `@types` version.

### Editor vs `tsc` disagreements

If VS Code shows `req.user` fine but `npx tsc --noEmit` errors (or vice versa), the two are using different programs. Checklist:

```bash
# 1. Is the editor using the workspace TypeScript, or its bundled one?
#    VS Code: Cmd+Shift+P → "TypeScript: Select TypeScript Version" → Use Workspace Version

# 2. Is the editor picking up the right tsconfig? (multiple tsconfigs in a monorepo)
#    Cmd+Shift+P → "TypeScript: Go to Project Configuration"

# 3. Restart the language server after adding a new .d.ts — it does not always
#    notice new files that aren't imported by anything:
#    Cmd+Shift+P → "TypeScript: Restart TS Server"

# 4. Confirm the file is in the program:
npx tsc --listFiles | grep -i "types/express"
```

Step 3 is the most common cause. Adding a `.d.ts` that nothing imports is exactly the case where the language server's incremental file watcher can miss it.

---

## Common mistakes

### Mistake 1 — Reaching for `as any` instead of augmenting

```ts
// ❌ WRONG — one cast per access site, and `any` leaks into everything downstream.
app.use(async (req, res, next) => {
  (req as any).user = await userRepo.findById(payload.sub);
  next();
});

app.get("/me/invoices", async (req, res) => {
  const userId = (req as any).user.id;           // any — no checking at all
  //                              ^^ note: `.id`, but the middleware wrote a
  //                              row with `.userId`. Nothing catches this.
  const invoices = await invoiceRepo.findByUser(userId);   // any flows in
  res.json(invoices);
});

// Variants that are equally bad:
const user = req["user"];                        // implicit any via index access
declare const req: any;                          // giving up entirely
// @ts-expect-error                              // suppressing the real signal
```

```ts
// ✅ RIGHT — one .d.ts file, and every access site is typed forever.

// src/types/express.d.ts
import type { AuthenticatedUser } from "../auth/types";
declare global {
  namespace Express {
    interface Request { user?: AuthenticatedUser }
  }
}
export {};

// Anywhere in the app:
app.use(async (req, res, next) => {
  req.user = await loadUser(payload.sub);   // ✅ checked against AuthenticatedUser
  next();
});

app.get("/me/invoices", async (req, res) => {
  const { user } = req;
  if (!user) { res.status(401).json({ ok: false, error: "Unauthenticated" }); return; }
  const invoices = await invoiceRepo.findByUser(user.userId);   // ✅ number
  res.json({ ok: true, data: invoices });
});
```

### Mistake 2 — Augmenting `"express"` instead of `express-serve-static-core` / the global namespace

```ts
// ❌ WRONG — compiles, produces no error, and has no effect.
//    `@types/express` declares Request inside `declare namespace e`, not at
//    module top level, so your interface merges with nothing.
declare module "express" {
  interface Request {
    user?: AuthenticatedUser;
  }
}

// Elsewhere:
req.user;   // ❌ Property 'user' does not exist on type 'Request<...>'
//              ...and you have no idea why, because the augmentation looks right.
```

```ts
// ✅ RIGHT (recommended) — the global Express namespace that core.Request extends:
import type { AuthenticatedUser } from "../auth/types";

declare global {
  namespace Express {
    interface Request { user?: AuthenticatedUser }
  }
}
export {};

// ✅ ALSO RIGHT — direct module augmentation of the package that owns Request:
declare module "express-serve-static-core" {
  interface Request { user?: AuthenticatedUser }
}
```

### Mistake 3 — Writing the `.d.ts` but never getting it into the compilation

```ts
// The file is perfect. It is also invisible.

// ❌ WRONG — tsconfig.json doesn't reach it:
// {
//   "include": ["src/routes/**/*.ts", "src/services/**/*.ts"]
// }
// with the augmentation at src/types/express.d.ts

// ❌ WRONG — the file lives outside rootDir:
// {
//   "compilerOptions": { "rootDir": "src" },
//   "include": ["src/**/*"]
// }
// with the augmentation at ./types/express.d.ts   ← repo root, not src/

// ❌ WRONG — ts-node ignores `include`, so it type-checks a smaller program:
//   npx ts-node src/app.ts    → "Property 'user' does not exist"
//   (tsc --noEmit passes, ts-node fails. Baffling until you know.)

// ❌ WRONG — the file is a SCRIPT, not a module, so `declare global` is illegal:
//   (no import, no export anywhere in the file)
declare global {
  namespace Express { interface Request { user?: AuthenticatedUser } }
}
// error TS2669: Augmentations for the global scope can only be directly nested
// in external modules or ambient module declarations.
```

```ts
// ✅ RIGHT — three things together:

// 1. The file is a module (import type OR export {} — ideally both):
import type { AuthenticatedUser } from "../auth/types";
declare global {
  namespace Express { interface Request { user?: AuthenticatedUser } }
}
export {};

// 2. tsconfig.json reaches it, and names it explicitly for good measure:
// {
//   "compilerOptions": { "strict": true, "rootDir": "src", "outDir": "dist" },
//   "files":   ["src/types/express.d.ts"],
//   "include": ["src/**/*.ts"],
//   "ts-node": { "files": true }
// }

// 3. Verify:
//   npx tsc --listFiles | grep express.d.ts   ← must print the path
```

### Mistake 4 — Declaring `user` as required in the global augmentation

```ts
// ❌ WRONG — a lie. `Request` is shared by /health, /login, /webhooks/stripe,
//    and every 404. On all of them, req.user is undefined at runtime.
declare global {
  namespace Express {
    interface Request { user: AuthenticatedUser }   // no `?`
  }
}
export {};

// The consequence, three months later:
app.get("/health", (req, res) => {
  console.log("health check by", req.user.email);
  // ✅ compiles — the type says `user` is always there
  // ❌ TypeError: Cannot read properties of undefined (reading 'email')
});

// Worse, the compiler now HELPS you write the bug — autocomplete offers
// `req.user.` on every route, authenticated or not.
```

```ts
// ✅ RIGHT — optional in the global type, made required by a wrapper that
//    performs the runtime check:

declare global {
  namespace Express {
    interface Request { user?: AuthenticatedUser }   // honest
  }
}
export {};

// src/http/wrappers.ts — the ONE place the optional becomes required:
export function withAuth<P, ResBody, ReqBody, Q>(
  handler: (req: Request<P, ResBody, ReqBody, Q> & { user: AuthenticatedUser },
            res: Response<ResBody>, next: NextFunction) => void | Promise<void>,
): RequestHandler<P, ResBody, ReqBody, Q> {
  return (req, res, next): void => {
    if (!req.user) { next(ApiError.unauthorized()); return; }   // runtime guarantee
    Promise.resolve(
      handler(req as Request<P, ResBody, ReqBody, Q> & { user: AuthenticatedUser }, res, next),
    ).catch(next);
  };
}

// Handlers are clean, and the guard can't be forgotten:
router.get("/me", withAuth((req, res) => {
  res.json({ ok: true, data: { email: req.user.email } });   // ✅ no `!`
}));

// And /health still can't touch req.user without narrowing. ✅
```

### Mistake 5 — Registering an `AuthenticatedRequest` handler with `as any`

```ts
// ❌ WRONG — the cast erases every other generic slot too.
interface AuthenticatedRequest extends Request {
  user: AuthenticatedUser;
}

async function getMyOrders(req: AuthenticatedRequest, res: Response): Promise<void> {
  const orders = await orderRepo.findByUser(req.user.userId);
  res.json(orders);
}

router.get("/me/orders", requireAuth, getMyOrders as any);
//                                                 ^^^^^^
// • `req.params`, `req.query`, and `res.json`'s body type are now unchecked.
// • Nothing verifies that `requireAuth` is actually in the chain — delete it
//   and this still compiles, then crashes at runtime.
// • Change getMyOrders' signature to take three params in the wrong order and
//   `as any` happily accepts that too.
```

```ts
// ✅ RIGHT — a typed wrapper that does the check and keeps all four generic slots:
router.get("/me/orders", withAuth<{}, OrderDto[], unknown, {}>(async (req, res) => {
  const orders = await orderRepo.findByUser(req.user.userId);   // ✅ user required
  res.json({ ok: true, data: orders.map(toOrderDto) });         // ✅ body checked
}));

// ✅ ALSO ACCEPTABLE — if you keep AuthenticatedRequest, confine the cast to
//    one named helper rather than sprinkling `as any` at every route:
export function authed<P, ResBody, ReqBody, Q>(
  handler: (req: AuthenticatedRequest<P, ResBody, ReqBody, Q>,
            res: Response<ResBody>, next: NextFunction) => void | Promise<void>,
): RequestHandler<P, ResBody, ReqBody, Q> {
  return handler as RequestHandler<P, ResBody, ReqBody, Q>;
}
router.get("/me/orders", requireAuth, authed(getMyOrders));
// ⚠️ Note: this variant still doesn't verify that requireAuth is present.
//    withAuth() does. Prefer withAuth().
```

### Mistake 6 — Colliding with a library's augmentation

```ts
// ❌ WRONG — @types/passport already declares `Request.user?: Express.User`.
//    Redeclaring it with a different type is an unfixable merge conflict.
import type { AuthenticatedUser } from "../auth/types";

declare global {
  namespace Express {
    interface Request { user?: AuthenticatedUser }
  }
}
export {};

// error TS2717: Subsequent property declarations must have the same type.
// Property 'user' must be of type 'User | undefined', but here has type
// 'AuthenticatedUser | undefined'.
```

```ts
// ✅ RIGHT — merge into the extension point the library provided:
declare global {
  namespace Express {
    // passport declares `interface User {}` (empty) for exactly this purpose.
    interface User {
      userId: number;
      email:  string;
      orgId:  number;
      role:   "owner" | "admin" | "member" | "readonly";
    }
  }
}
export {};

// req.user is now `Express.User | undefined` with your members. ✅

// ✅ SAME IDEA for express-session:
declare module "express-session" {
  interface SessionData {          // the library's documented extension point
    userId?: number;
    csrfToken?: string;
  }
}
// req.session.userId ✅

// ✅ SAME IDEA when publishing your own middleware — prefix, and stay optional:
declare module "express-serve-static-core" {
  interface Request {
    /** Set by @acme/tenant-resolver. Optional: consumers control ordering. */
    acmeTenant?: { orgId: number; plan: "free" | "pro" };
  }
}
```

### Mistake 7 — Forgetting that the required fields are only true if the middleware ran

```ts
// ❌ WRONG — the augmentation promises `requestId: string`, but nothing
//    enforces registration order. Here the router is mounted BEFORE the
//    decorator, so requestId is undefined inside every route.
const app = express();

app.use("/api", apiRouter);        // ← routes first
app.use(attachRequestId);          // ← decorator second: too late, never runs
                                   //   for /api requests

// Inside apiRouter:
res.setHeader("x-request-id", req.requestId);
// ✅ compiles (type is `string`)
// ❌ TypeError: Invalid value "undefined" for header "x-request-id"
```

```ts
// ✅ RIGHT — decorators first, unconditionally, before any route:
const app = express();

app.use(express.json());
app.use(attachRequestId);          // 1. satisfies `requestId: string`
app.use(attachPrincipal);          // 2. populates optional `user`
app.use("/api", apiRouter);        // 3. routes
app.use(errorHandler);             // 4. errors, last

// ✅ AND — only declare a field required when it's set by an unconditional,
//    first-in-chain middleware. Everything conditional gets `?`:
declare global {
  namespace Express {
    interface Request {
      requestId: string;                // ✅ set by app.use() #1, always
      user?: Principal;                 // ✅ optional — depends on the token
      tenant?: RequestTenant;           // ✅ optional — only on /orgs/:slug routes
      rateLimit?: { remaining: number }; // ✅ optional — only on limited routes
    }
  }
}
export {};
```

---

## Practice exercises

### Exercise 1 — easy

Set up request decoration from scratch for a small API. Do not copy the examples above — write it yourself.

1. Create `src/auth/types.ts` exporting an `AuthenticatedUser` interface with `userId: number`, `email: string`, `role: "admin" | "member"`, and a `SessionInfo` interface with `sessionId: string` and `issuedAtMs: number`.
2. Create `src/types/express.d.ts` that augments the global `Express.Request` interface with:
   - `user?: AuthenticatedUser` (optional)
   - `session?: SessionInfo` (optional)
   - `requestId: string` (required)
   Make sure the file is a module.
3. Write the `tsconfig.json` snippet that guarantees `tsc` includes the file — use `files` as well as `include`, and add the `ts-node` setting.
4. Write `attachRequestId: RequestHandler` that reads the `x-request-id` header (remember: header values are `string | string[] | undefined`) and falls back to `randomUUID()`.
5. Write `attachUser: RequestHandler` that reads a `Bearer` token from `req.headers.authorization`, calls a hypothetical `verifyToken(token): Promise<{ sub: number; email: string; role: string }>`, validates that `role` is one of the two literals using a type predicate, and assigns `req.user`.
6. Write one route, `GET /whoami`, that responds with `{ requestId, email }` when a user is present and `401` otherwise — using control-flow narrowing, no `!` and no `as`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build the `withAuth` wrapper pattern end to end.

1. Define `type ApiResponse<T>` as a discriminated union: `{ ok: true; data: T }` | `{ ok: false; error: string; code: ErrorCode; requestId: string }`.
2. Define `AuthedRequest<P, ResBody, ReqBody, Q>` as an intersection of Express's `Request` with `{ user: AuthenticatedUser }`, and `AuthedHandler<P, Data, ReqBody, Q>` as the handler function type that receives it.
3. Write `withAuth` so that it:
   - is generic over all four `Request` slots and does not erase them,
   - responds/`next()`s with a 401 when `req.user` is absent,
   - performs exactly one type assertion, immediately after the check,
   - forwards async rejections to `next` (Express 4 behaviour),
   - returns a value assignable to `RequestHandler<P, ApiResponse<Data>, ReqBody, Q>`.
4. Write `withRole(allowed: readonly UserRole[], handler)` by *composing* `withAuth` — it must not repeat the `req.user` check or the cast.
5. Implement three routes on a `Router({ mergeParams: true })` mounted at `/orgs/:orgSlug/projects`:
   - `GET /` — returns `ProjectDto[]`, reads `req.params.orgSlug` and `req.query.archived` (remember what type query values are).
   - `GET /:projectId` — 404s when missing, 403s when `project.orgId !== req.user.orgId`.
   - `DELETE /:projectId` — wrapped in `withRole(["admin"])`.
6. Prove the wrapper preserves types by adding a comment on each handler naming what `req.params`, `req.query`, `req.body` and `req.user` resolve to.
7. Add a comment explaining what would change under Express 5.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **capability-typed request pipeline**: a system where each middleware declares what it *adds* to the request, each handler declares what it *requires*, and the compiler rejects a route whose handler requires a capability that no wrapper in its chain supplies.

Requirements:

1. Define a `Capabilities` interface mapping capability names to the shape they add:
   ```
   interface Capabilities {
     auth:      { user: Principal };
     tenant:    { tenant: RequestTenant };
     rateLimit: { rateLimit: { remaining: number; resetAtMs: number } };
     idempotency: { idempotencyKey: string };
   }
   ```
2. Define `type RequestWith<K extends keyof Capabilities, P, ResBody, ReqBody, Q>` that intersects Express's `Request` with the union of the shapes for every capability in `K`. (Hint: you need a mapped-type-to-intersection helper — `UnionToIntersection` or an indexed mapped type.)
3. Define `type PipelineHandler<K extends keyof Capabilities, P, Data, ReqBody, Q>` accordingly.
4. Write `pipeline()` — a builder with `.use(capabilityName, middleware)` and `.handle(handler)`, where:
   - `.use("auth", ...)` adds `"auth"` to the accumulated capability union in the *type* of the builder,
   - `.handle(handler)` only accepts a handler whose required capability set is a **subset** of the accumulated set — a handler needing `"tenant"` after only `.use("auth")` must be a **compile error**,
   - `.handle()` returns a plain `RequestHandler` that Express can register with no cast at the call site.
5. Each `.use()` middleware performs the runtime check and, on failure, calls `next(ApiError)`. All type assertions in the whole system must live inside the builder — zero at any call site.
6. Prove it works with four routes:
   - a public route needing nothing,
   - `GET /orgs/:orgSlug/orders` needing `auth` + `tenant`,
   - `POST /orgs/:orgSlug/orders` needing `auth` + `tenant` + `idempotency`,
   - a deliberately broken route where the handler needs `tenant` but the pipeline only supplies `auth` — leave it commented out with the expected error text.
7. Write the `src/types/express.d.ts` that backs it: every capability field must be **optional** in the global augmentation, because the builder is what makes them required.
8. Finish with a short comment answering: what does this buy you over plain `withAuth`/`withTenant` wrappers, and what does it cost in compile time, error-message readability, and onboarding?

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ═══ THE AUGMENTATION — src/types/express.d.ts ════════════════════════════════
import type { AuthenticatedUser } from "../auth/types";

declare global {              // escape module scope (only legal inside a module)
  namespace Express {         // the global namespace @types/express provides
    interface Request {       // merges with the empty Express.Request
      user?: AuthenticatedUser;   // optional — most apps have public routes
      requestId: string;          // required — set by the FIRST middleware
    }
  }
}
export {};                    // makes the file a module → `declare global` legal

// ═══ EQUIVALENT, EXPLICIT FORM ════════════════════════════════════════════════
declare module "express-serve-static-core" {
  interface Request { user?: AuthenticatedUser }
}
// ⚠️ NOT `declare module "express"` — that merges with nothing.

// ═══ TSCONFIG — the file must be IN the program ═══════════════════════════════
// {
//   "compilerOptions": { "strict": true, "rootDir": "src", "outDir": "dist" },
//   "files":   ["src/types/express.d.ts"],
//   "include": ["src/**/*.ts"],
//   "ts-node": { "files": true }        ← ts-node ignores `files` without this
// }
// Verify:  npx tsc --listFiles | grep express.d.ts

// ═══ THE OPTIONAL→REQUIRED CONVERTER ══════════════════════════════════════════
type AuthedRequest<P, R, B, Q> = Request<P, R, B, Q> & { user: AuthenticatedUser };

export function withAuth<P, R, B, Q>(
  handler: (req: AuthedRequest<P, R, B, Q>, res: Response<R>, next: NextFunction)
    => void | Promise<void>,
): RequestHandler<P, R, B, Q> {
  return (req, res, next): void => {
    if (!req.user) { next(ApiError.unauthorized()); return; }    // runtime check
    Promise.resolve(handler(req as AuthedRequest<P, R, B, Q>, res, next)).catch(next);
  };                                    // ↑ the ONE cast, next to its justification
}

router.get("/me", withAuth<{}, ProfileDto>(async (req, res) => {
  res.json({ ok: true, data: toProfile(req.user) });   // req.user required ✅
}));

// ═══ NARROW-AT-THE-TOP (no infrastructure needed) ═════════════════════════════
const handler: RequestHandler = (req, res) => {
  const { user } = req;
  if (!user) { res.status(401).json({ ok: false, error: "Unauthenticated" }); return; }
  user.userId;   // AuthenticatedUser ✅
};

// ═══ LIBRARY EXTENSION POINTS — merge here, not into Request ══════════════════
declare global { namespace Express { interface User { userId: number } } }  // passport
declare module "express-session" { interface SessionData { userId?: number } }
// multer: Express.Multer.File is already declared for you
```

| Task | Do this |
|---|---|
| Add `req.user` | `declare global { namespace Express { interface Request { user?: T } } }` |
| Make the file a module | `export {};` at the bottom (or any top-level `import`/`export`) |
| Get `tsc` to see it | `include: ["src/**/*"]` covering it, plus `files: [...]` for certainty |
| Get `ts-node` to see it | `"ts-node": { "files": true }` or `ts-node --files` |
| Verify it's loaded | `npx tsc --listFiles \| grep express.d.ts` |
| Wrong augmentation target | `declare module "express"` — compiles, does nothing |
| Right augmentation targets | global `Express` namespace, or `"express-serve-static-core"` |
| Optional vs required | Optional in the type; required via a wrapper that checks at runtime |
| Handler needs `user` | `withAuth(handler)` — one cast, next to the 401 check |
| Registering `AuthenticatedRequest` | Needs a cast; prefer `withAuth` which justifies its own |
| Conflicts with passport | Augment `Express.User`, not `Request.user` |
| Publishing a library | `declare module "express-serve-static-core"`, prefixed + optional; never `declare global` |
| Different principal shapes | One discriminated union type — the interface can't be generic |
| `strict: false` | The whole `?` discussion is meaningless; turn on `strictNullChecks` first |
| Express 5 | Same augmentation, unchanged; drop the `Promise.resolve().catch(next)` if you like |

---

## Connected topics

- **52 — Typing Express routes** — the four `Request` generic slots (`Params`, `ResBody`, `ReqBody`, `Query`) that every wrapper in this doc has to thread through, plus `res.locals` as the fifth.
- **42 — Discriminated unions** — `Principal` as `{ kind: "user" } | { kind: "service" }`, and the `ApiResponse<T>` envelope; narrowing on `kind` is what makes multi-tenant checks compiler-enforced.
- **26 — What are generics** and **30 — Generic constraints** — the machinery behind `withAuth<P, ResBody, ReqBody, Q>` and behind the capability builder in Exercise 3.
- **19 — Interfaces vs type aliases** — why only `interface` can merge, and why the augmentation cannot be a `type`.
- **34 — Intersection types** — `Request<...> & { user: Principal }` and how capability intersections compose.
- **31 — Type narrowing and control flow analysis** — `if (!user) return;` and `req.user.kind === "user"`; narrowing is what makes the optional field usable without assertions.
- **37 — Type predicates and assertion functions** — `isUserRole(value): value is UserRole`, and why an `asserts req is AuthenticatedRequest` predicate cannot cross the middleware boundary.
- **04 — TypeScript Node.js project setup** — `include`, `files`, `typeRoots`, `types`, and `skipLibCheck`: the settings that decide whether your `.d.ts` is in the program at all.
- **09 — strictNullChecks and optional properties** — the flag that makes `user?:` mean something, and the `!` operator you should mostly avoid.
- **50 — Ambient declarations and .d.ts files** — the broader story of declaration files, `declare module`, and describing untyped JavaScript.
