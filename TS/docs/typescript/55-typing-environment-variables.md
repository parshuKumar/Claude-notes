# 55 — Typing environment variables

## What is this?

Every Node process is handed a bag of strings by the operating system. Node exposes that bag as `process.env`:

```ts
process.env.DATABASE_URL;   // string | undefined
process.env.PORT;           // string | undefined
process.env.NODE_ENV;       // string | undefined
```

The type is not an accident and it is not TypeScript being pedantic. Here is the actual declaration shipped in `@types/node`:

```ts
// node_modules/@types/node/process.d.ts (simplified)
namespace NodeJS {
  interface ProcessEnv {
    [key: string]: string | undefined;    // ← an index signature, nothing more
  }
}
```

That index signature is the whole story:

- **`string`** — because environment variables are *always* strings. `PORT=3000` gives you `"3000"`, not `3000`. `FEATURE_BILLING=true` gives you `"true"`, not `true`. There is no JSON, no numbers, no booleans, no `null` — the kernel handed Node a `char*`.
- **`| undefined`** — because *any* key you ask for might not be set. `process.env.TOTALLY_MADE_UP_KEY` is a legal expression that returns `undefined` at runtime.

**Typing environment variables** means building a boundary: one module that reads `process.env` exactly once, validates every variable your application needs, coerces the strings into the types your code actually wants (`number`, `boolean`, union literals, URLs), fails loudly at process boot if anything is missing or malformed, and exports a **fully typed, frozen config object**. Everywhere else in the codebase imports that object and never touches `process.env` again.

```ts
// src/config/env.ts — the ONLY file that reads process.env
export const env = loadEnv();

// everywhere else:
import { env } from "./config/env.js";

env.PORT;          // number       — parsed, validated, guaranteed present
env.DATABASE_URL;  // string       — guaranteed non-empty
env.NODE_ENV;      // "development" | "test" | "production"
env.SENTRY_DSN;    // string | undefined — genuinely optional, and typed that way
```

This document covers why `!` and `as string` are lies, how to augment `NodeJS.ProcessEnv` (and why augmentation alone still validates nothing), how to hand-roll a validated env module, how to do the same with zod, dotenv loading order, per-environment defaults, and the single most important architectural rule: **`process.env` appears in exactly one file.**

---

## Why does it matter?

Environment variables are the seam between your code and the outside world, and they have three properties that make them uniquely dangerous:

1. **They are untyped.** Everything is a string. Every consumer has to remember to coerce, and every consumer coerces slightly differently. `Number(process.env.PORT)`, `parseInt(process.env.PORT)`, `+process.env.PORT`, `process.env.PORT * 1`. One of those returns `NaN` for `"3000abc"`, one returns `3000`, one crashes on `undefined`.

2. **They are set outside your repository.** In a Docker Compose file, a Kubernetes Secret, a Heroku dashboard, a CI job definition, a teammate's `.env` that isn't in git. The failure mode is: it works on your machine, it works in staging, and it explodes in production because someone forgot to add `STRIPE_WEBHOOK_SECRET` to the prod secret store.

3. **The failure is silent and late.** `undefined` does not throw when you read it. It throws — or worse, doesn't throw — when you *use* it:

```ts
// The bug is here...
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);   // undefined at boot

// ...but it surfaces here, at 2am, on a customer's first payment:
await stripe.charges.create({ amount: 4999, currency: "usd" });
// → StripeAuthenticationError: Invalid API Key provided: undefined
```

Or worse, it never throws at all:

```ts
const maxUploadBytes = Number(process.env.MAX_UPLOAD_BYTES);   // NaN
if (requestBody.size > maxUploadBytes) throw new Error("Too large");
// NaN comparisons are ALWAYS false → the limit is silently disabled.
// You now accept 4GB uploads. Nothing logged. Nothing crashed.
```

```ts
const enableRateLimit = Boolean(process.env.ENABLE_RATE_LIMIT);  // "false" → true!
// Every non-empty string is truthy. Setting ENABLE_RATE_LIMIT=false
// ENABLES rate limiting. This class of bug has taken down real systems.
```

A validated env module converts all three problems into a single, loud, immediate failure:

```
✖ Invalid environment configuration:
  • DATABASE_URL   — required, but not set
  • PORT           — expected an integer between 1 and 65535, got "three thousand"
  • NODE_ENV       — expected one of development | test | production, got "prod"
  • MAX_UPLOAD_MB  — expected a number, got "10mb"

Process exiting. Fix the environment and restart.
```

The container dies in 200ms, the orchestrator refuses to route traffic to it, the deploy rolls back, and nobody gets paged at 2am. **Fail fast at boot, or fail slow in production — those are the only two options.**

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: process.env everywhere, hope everywhere ──────────────────────

// src/db.js
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,   // undefined? → cryptic ECONNREFUSED
  max: process.env.DB_POOL_MAX || 10,           // ← "20" (a STRING) or 10 (a number)
  //                                                  the pool silently misbehaves
});

// src/server.js
const port = process.env.PORT || 3000;
app.listen(port);                    // listen("3000") works... by coincidence

// src/auth.js
const jwtSecret = process.env.JWT_SECRET;
jwt.sign({ userId }, jwtSecret);     // undefined secret → jsonwebtoken throws...
                                     // ...on the first login, not at boot

// src/features.js
const billingEnabled = process.env.ENABLE_BILLING;    // "false"
if (billingEnabled) chargeCustomer();                 // ← RUNS. "false" is truthy.

// src/upload.js
const maxBytes = parseInt(process.env.MAX_UPLOAD_BYTES);   // NaN if unset
if (size > maxBytes) reject();                             // NaN → never rejects

// And the classic: a typo you will never find.
mailer.send({ apiKey: process.env.SENDGRID_API_KEY });   // set in prod
mailer.send({ apiKey: process.env.SENDGRID_APIKEY });    // typo → undefined
//                                        ^^^^^^ no underscore. No error. Ever.
```

Every single line above is a landmine, and JavaScript gives you nothing: no autocomplete for variable names, no signal that a value might be missing, no indication that `DB_POOL_MAX` is a string when the pool wants a number, and absolutely no protection against typos.

Now the naive TypeScript "fix", which is worse than it looks:

```ts
// ── TypeScript, done badly: the compiler is told to shut up ──────────────────

const pool = new Pool({
  connectionString: process.env.DATABASE_URL!,     // ❌ `!` = "trust me"
  max: Number(process.env.DB_POOL_MAX),            // ❌ NaN if unset
});

const jwtSecret = process.env.JWT_SECRET as string;  // ❌ a cast is a LIE
jwt.sign({ userId }, jwtSecret);                     // still undefined at runtime

const port = parseInt(process.env.PORT ?? "3000", 10);  // ok-ish, but repeated
                                                        // in 5 files, differently
```

`!` and `as string` do **zero** runtime work. They erase at compile time. The variable is still `undefined`; you have only removed the compiler's ability to warn you. You have made the code *more* dangerous than the JavaScript version, because now it *looks* checked.

```ts
// ── TypeScript, done right: one validated boundary ───────────────────────────

// src/config/env.ts — the only file in the repo that mentions process.env
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV:         z.enum(["development", "test", "production"]).default("development"),
  PORT:             z.coerce.number().int().min(1).max(65_535).default(3000),
  DATABASE_URL:     z.string().url().startsWith("postgres"),
  DB_POOL_MAX:      z.coerce.number().int().min(1).max(100).default(10),
  JWT_SECRET:       z.string().min(32, "JWT_SECRET must be at least 32 chars"),
  ENABLE_BILLING:   z.enum(["true", "false"]).default("false").transform((v) => v === "true"),
  MAX_UPLOAD_BYTES: z.coerce.number().int().positive().default(5_242_880),
  SENTRY_DSN:       z.string().url().optional(),     // genuinely optional
});

const parsed = EnvSchema.safeParse(process.env);

if (!parsed.success) {
  console.error("✖ Invalid environment configuration:");
  for (const issue of parsed.error.issues) {
    console.error(`  • ${issue.path.join(".")} — ${issue.message}`);
  }
  process.exit(1);                     // fail fast, at boot, with a clear message
}

export const env = Object.freeze(parsed.data);
export type Env = typeof env;
```

```ts
// ── Every other file in the codebase ────────────────────────────────────────
import { env } from "./config/env.js";

const pool = new Pool({
  connectionString: env.DATABASE_URL,   // ✅ string, validated as a postgres URL
  max: env.DB_POOL_MAX,                 // ✅ number, 1–100
});

app.listen(env.PORT);                   // ✅ number

jwt.sign({ userId }, env.JWT_SECRET);   // ✅ string, ≥32 chars

if (env.ENABLE_BILLING) chargeCustomer();  // ✅ real boolean. "false" → false.

if (requestBody.size > env.MAX_UPLOAD_BYTES) reject();  // ✅ never NaN

env.SENDGRID_APIKEY;
//  ^^^^^^^^^^^^^^^ ❌ Property 'SENDGRID_APIKEY' does not exist on type 'Env'.
//                     Did you mean 'SENDGRID_API_KEY'?
```

The revelation: **you don't type `process.env` — you replace it.** The goal is not to make `process.env.PORT` return `number`; that's impossible, it's a string at runtime. The goal is to parse the untrusted string bag once, at the edge, into a trusted typed object, and then never look at the bag again. This is the exact same discipline you apply to `req.body` — env vars are just another untrusted input, they simply arrive earlier.

---

## Syntax

```ts
// ═══════════════════════════════════════════════════════════════════════════
// 1. What you get out of the box
// ═══════════════════════════════════════════════════════════════════════════

process.env.DATABASE_URL;        // string | undefined
process.env["DATABASE_URL"];     // string | undefined  (identical)
process.env.ANY_KEY_AT_ALL;      // string | undefined  (index signature, no typo check)

// ═══════════════════════════════════════════════════════════════════════════
// 2. The three unsafe escapes (know them so you can recognise them in reviews)
// ═══════════════════════════════════════════════════════════════════════════

process.env.DATABASE_URL!;                  // non-null assertion — no runtime check
process.env.DATABASE_URL as string;         // type assertion   — no runtime check
process.env.DATABASE_URL ?? "";             // silent default   — hides the failure

// ═══════════════════════════════════════════════════════════════════════════
// 3. Module augmentation — better autocomplete, still zero validation
// ═══════════════════════════════════════════════════════════════════════════

// src/types/env.d.ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV:      "development" | "test" | "production";
      PORT:          string;               // still a STRING — env vars always are
      DATABASE_URL:  string;
      SENTRY_DSN?:   string;               // `?` marks it optional
    }
  }
}
export {};                                 // makes this file a module

// ═══════════════════════════════════════════════════════════════════════════
// 4. Hand-rolled readers — validation that actually runs
// ═══════════════════════════════════════════════════════════════════════════

function requireString(key: string): string { /* throws if missing */ }
function optionalString(key: string): string | undefined { /* … */ }
function requireInt(key: string, fallback?: number): number { /* … */ }
function requireBool(key: string, fallback?: boolean): boolean { /* … */ }
function requireEnum<const T extends readonly string[]>(key: string, values: T): T[number];

// ═══════════════════════════════════════════════════════════════════════════
// 5. zod schema — declarative validation + inferred types
// ═══════════════════════════════════════════════════════════════════════════

import { z } from "zod";

const EnvSchema = z.object({
  PORT:          z.coerce.number().int().positive().default(3000),   // "3000" → 3000
  DATABASE_URL:  z.string().url(),
  NODE_ENV:      z.enum(["development", "test", "production"]),
  DEBUG:         z.enum(["true", "false"]).transform((v) => v === "true"),
  SENTRY_DSN:    z.string().url().optional(),
});

type Env = z.infer<typeof EnvSchema>;
// { PORT: number; DATABASE_URL: string; NODE_ENV: "development" | "test" | "production";
//   DEBUG: boolean; SENTRY_DSN?: string }

const result = EnvSchema.safeParse(process.env);   // never throws
if (!result.success) { /* print result.error.issues, process.exit(1) */ }
export const env: Env = Object.freeze(result.data);

// ═══════════════════════════════════════════════════════════════════════════
// 6. dotenv — load BEFORE anything reads process.env
// ═══════════════════════════════════════════════════════════════════════════

import "dotenv/config";                         // side-effect import, must be first
// or, explicitly:
import { config } from "dotenv";
config({ path: `.env.${process.env.NODE_ENV ?? "development"}` });
config({ path: ".env" });                       // does NOT overwrite already-set keys
```

---

## How it works — concept by concept

### Concept 1 — Why `process.env.X` is `string | undefined`

Open the type. It is four lines, and understanding them removes all the mystery:

```ts
// @types/node/process.d.ts
declare namespace NodeJS {
  interface ProcessEnv extends Dict<string> {}
}

interface Dict<T> {
  [key: string]: T | undefined;
}
```

So `ProcessEnv` is exactly `{ [key: string]: string | undefined }`. Three consequences follow mechanically:

**(a) Every property access type-checks, including typos.**

```ts
process.env.DATABASE_URL;             // string | undefined
process.env.DTABASE_URL;              // string | undefined — ✅ compiles!
process.env.I_AM_NOT_A_REAL_VARIABLE; // string | undefined — ✅ compiles!
```

An index signature says "any string key is valid". The compiler has no list of real keys to compare against, so it cannot catch a typo. This is the single biggest source of env-related production incidents.

**(b) The `| undefined` is mandatory and correct.**

```ts
const databaseUrl: string = process.env.DATABASE_URL;
// ❌ Type 'string | undefined' is not assignable to type 'string'.
//    Type 'undefined' is not assignable to type 'string'.
```

The compiler is right. At the moment that line runs, `DATABASE_URL` genuinely might not be set. There is no static information available that could prove otherwise.

> **Note:** the `| undefined` only appears when `strictNullChecks` (i.e. `strict: true`) is on. Under `"strict": false`, `process.env.DATABASE_URL` is just `string` and every one of these bugs compiles clean. See **62 — Strict null checks**.

**(c) Everything is a `string`, never a `number` or `boolean`.**

```bash
PORT=3000 ENABLE_CACHE=true node server.js
```

```ts
typeof process.env.PORT;          // "string"
process.env.PORT === 3000;        // false — "3000" !== 3000
process.env.PORT === "3000";      // true

typeof process.env.ENABLE_CACHE;  // "string"
process.env.ENABLE_CACHE === true;  // ❌ compile error, and false at runtime
```

There is also a fourth, less obvious consequence: **assignment is allowed and is coerced.**

```ts
process.env.PORT = 3000;
// ❌ Type 'number' is not assignable to type 'string | undefined'.

process.env.PORT = "3000";        // ✅ fine
process.env.PORT = String(3000);  // ✅ fine

// At runtime, Node coerces anything you assign:
(process.env as Record<string, unknown>).PORT = 3000;
typeof process.env.PORT;          // "string" — Node called String() on it
```

Writing to `process.env` mutates the real process environment (and is inherited by child processes). It's occasionally useful in tests, and almost never appropriate in application code.

### Concept 2 — Why `!` and `as string` are lies

Both operators are **compile-time-only**. TypeScript erases them completely; the emitted JavaScript is unchanged.

```ts
// ── What you write ──────────────────────────────────────────────────────────
const databaseUrl = process.env.DATABASE_URL!;
const jwtSecret   = process.env.JWT_SECRET as string;

// ── What actually ships (after tsc) ─────────────────────────────────────────
const databaseUrl = process.env.DATABASE_URL;    // ← identical to the JS version
const jwtSecret   = process.env.JWT_SECRET;      // ← identical to the JS version
```

You have changed nothing about the program's behaviour. You have only changed what the compiler is willing to tell you. That's the definition of a lie: the type says `string`, the value is `undefined`.

Watch how far the lie travels:

```ts
const jwtSecret: string = process.env.JWT_SECRET!;   // actually undefined

jwtSecret.length;                 // ✅ compiles → 💥 TypeError at runtime
jwtSecret.slice(0, 4);            // ✅ compiles → 💥 TypeError at runtime
jwtSecret.toUpperCase();          // ✅ compiles → 💥 TypeError at runtime

// And the truly evil case — no crash at all:
const signed = crypto
  .createHmac("sha256", jwtSecret)   // Node: undefined → throws? depends on version
  .update(payload)
  .digest("hex");

// Or with a template literal — undefined stringifies silently:
const redisUrl = `redis://${process.env.REDIS_HOST!}:6379`;
// → "redis://undefined:6379"
// DNS fails with a confusing ENOTFOUND for the literal hostname "undefined".
```

The `??` "fix" is often worse, because it hides the failure permanently:

```ts
// ❌ Now the app boots happily with a database it will never reach:
const databaseUrl = process.env.DATABASE_URL ?? "postgres://localhost:5432/dev";
// In production this silently points at a database that doesn't exist,
// or — much worse — at a real local one with stale data.

// ❌ The empty-string default: connects to nothing, throws nowhere useful:
const stripeKey = process.env.STRIPE_SECRET_KEY ?? "";

// ❌ The "dev secret" default — a genuine security hole if it reaches prod:
const jwtSecret = process.env.JWT_SECRET ?? "dev-secret";
// Anyone who reads your open-source repo can now forge tokens.
```

The rule that separates a safe default from a dangerous one:

> **A default is acceptable only when the default value is correct in production.**
> `PORT=3000` — fine, the orchestrator overrides it.
> `LOG_LEVEL=info` — fine.
> `JWT_SECRET=dev-secret` — never. `DATABASE_URL=localhost` — never.

The correct shape is always: **check at boot, throw if absent, narrow legitimately.**

```ts
function requireEnv(key: string): string {
  const value = process.env[key];
  // The runtime check is what earns the `string` return type:
  if (value === undefined || value.trim() === "") {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;                    // ✅ narrowed to string by control flow analysis
}

const databaseUrl = requireEnv("DATABASE_URL");   // string — and it's TRUE
```

No `!`, no `as`. The compiler narrows `string | undefined` to `string` because the `if` genuinely eliminated `undefined`. That is a *proof*, not an assertion. See **40 — Type narrowing**.

### Concept 3 — Augmenting `NodeJS.ProcessEnv`

You can replace the index signature's vagueness with a concrete list of keys through **declaration merging** on the global `NodeJS.ProcessEnv` interface.

```ts
// src/types/env.d.ts
export {};                         // ← makes this a module, so `declare global` is legal

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV:            "development" | "test" | "production";
      PORT:                string;      // still a string! do NOT write `number`
      DATABASE_URL:        string;
      REDIS_URL:           string;
      JWT_SECRET:          string;
      AWS_REGION:          string;
      SENTRY_DSN?:         string;      // optional → stays `string | undefined`
      LOG_LEVEL?:          "debug" | "info" | "warn" | "error";
    }
  }
}
```

What you gain:

```ts
process.env.DATABASE_URL;      // string          — no `| undefined`!
process.env.NODE_ENV;          // "development" | "test" | "production"
process.env.SENTRY_DSN;        // string | undefined
process.env.LOG_LEVEL;         // "debug" | "info" | "warn" | "error" | undefined

// Autocomplete now lists your real variables.
// And, if the interface has NO index signature left, typos are caught:
process.env.DTABASE_URL;
//          ^^^^^^^^^^^ Property 'DTABASE_URL' does not exist on type 'ProcessEnv'.
```

Three important caveats, in increasing order of severity.

**Caveat 1 — the file must be included by `tsconfig.json`.**

If `src/types/env.d.ts` falls outside your `include` globs, the augmentation silently does nothing and you'll swear TypeScript is broken.

```jsonc
{
  "include": ["src/**/*.ts", "src/**/*.d.ts"],   // ← .d.ts must be covered
}
```

Verify with `npx tsc --listFiles | grep env.d.ts`.

**Caveat 2 — the base index signature is still there.**

`@types/node` declares `interface ProcessEnv extends Dict<string>`. Interface merging is **additive** — your declaration adds members but cannot delete the inherited `[key: string]: string | undefined`. Depending on your `@types/node` version, typo detection may therefore *not* work:

```ts
process.env.COMPLETELY_MADE_UP;   // may still be `string | undefined`, no error
```

(Some versions of `@types/node` moved the index signature such that merging does shadow it; do not rely on either behaviour. Test it in your own repo.)

**Caveat 3 — and this is the big one — augmentation validates nothing.**

```ts
// You declared: DATABASE_URL: string
// You run:      node server.js         (with no DATABASE_URL set)

const databaseUrl = process.env.DATABASE_URL;
//    ^^^^^^^^^^^ TypeScript says: string
//                Runtime says:    undefined

databaseUrl.startsWith("postgres");   // ✅ compiles → 💥 TypeError
```

Declaration merging is **exactly as much of a lie as `as string`** — you have simply moved the lie into a `.d.ts` file where it applies to the entire codebase at once. You've upgraded from lying at one call site to lying everywhere, permanently, invisibly.

```ts
// This is why augmentation alone is a trap:
//
//   `as string`        →  lies about ONE expression
//   `ProcessEnv` aug.  →  lies about EVERY expression, forever, silently
//
// Neither runs a single line of validation code.
```

So what *is* augmentation good for? Two legitimate uses:

1. **Autocomplete and typo-catching while writing the env module itself.** Inside `src/config/env.ts`, having the key names listed is genuinely useful.
2. **Documenting the contract** — a `.d.ts` that enumerates every variable the service needs is a nice, greppable spec.

But it must be paired with runtime validation. Many teams skip augmentation entirely once they have a validated `env` module, because the exported object already provides better types than `ProcessEnv` ever could (`number` instead of `string`, real booleans, branded URLs).

```ts
// A safer augmentation style — keep everything OPTIONAL, matching reality.
// This gives you key-name autocomplete without lying about presence:
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV?:     "development" | "test" | "production";
      PORT?:         string;
      DATABASE_URL?: string;
      JWT_SECRET?:   string;
    }
  }
}
export {};
```

Now `process.env.DATABASE_URL` is `string | undefined` — true — and you still get autocomplete. The env module does the narrowing honestly.

### Concept 4 — A hand-rolled validated env module

No dependencies. The pattern is a set of small readers plus an error accumulator, so a single boot attempt reports **every** problem rather than one at a time.

```ts
// src/config/env.ts

// ═══════════════════════════════════════════════════════════════════════════
// Error accumulation — collect ALL problems, then report once
// ═══════════════════════════════════════════════════════════════════════════

interface EnvIssue {
  key:     string;
  message: string;
}

const issues: EnvIssue[] = [];

function fail(key: string, message: string): void {
  issues.push({ key, message });
}

// Read raw. Treat empty/whitespace-only as absent — a common CI mistake is
// `DATABASE_URL=` in a .env, which yields "" rather than undefined.
function raw(key: string): string | undefined {
  const value = process.env[key];
  if (value === undefined) return undefined;
  const trimmed = value.trim();
  return trimmed === "" ? undefined : trimmed;
}

// ═══════════════════════════════════════════════════════════════════════════
// Readers — each returns a plausible value on failure so parsing CONTINUES,
// which is what lets us report every issue in one pass.
// ═══════════════════════════════════════════════════════════════════════════

function requiredString(
  key: string,
  opts: { minLength?: number; pattern?: RegExp } = {},
): string {
  const value = raw(key);
  if (value === undefined) {
    fail(key, "required, but not set");
    return "";                                  // placeholder; we exit before use
  }
  if (opts.minLength !== undefined && value.length < opts.minLength) {
    fail(key, `must be at least ${opts.minLength} characters (got ${value.length})`);
  }
  if (opts.pattern && !opts.pattern.test(value)) {
    fail(key, `does not match ${String(opts.pattern)}`);
  }
  return value;
}

function optionalString(key: string): string | undefined {
  return raw(key);                              // absent is legal — no issue recorded
}

function requiredInt(
  key: string,
  opts: { min?: number; max?: number; default?: number } = {},
): number {
  const value = raw(key);

  if (value === undefined) {
    if (opts.default !== undefined) return opts.default;
    fail(key, "required, but not set");
    return 0;
  }

  // Number() rather than parseInt(): parseInt("3000abc") === 3000, which is
  // exactly the kind of silent acceptance we're trying to eliminate.
  const parsed = Number(value);

  if (!Number.isFinite(parsed) || !Number.isInteger(parsed)) {
    fail(key, `expected an integer, got ${JSON.stringify(value)}`);
    return 0;
  }
  if (opts.min !== undefined && parsed < opts.min) {
    fail(key, `must be >= ${opts.min} (got ${parsed})`);
  }
  if (opts.max !== undefined && parsed > opts.max) {
    fail(key, `must be <= ${opts.max} (got ${parsed})`);
  }
  return parsed;
}

const TRUTHY = new Set(["true", "1", "yes", "on"]);
const FALSY  = new Set(["false", "0", "no", "off"]);

function requiredBool(key: string, opts: { default?: boolean } = {}): boolean {
  const value = raw(key)?.toLowerCase();

  if (value === undefined) {
    if (opts.default !== undefined) return opts.default;
    fail(key, "required, but not set");
    return false;
  }
  if (TRUTHY.has(value)) return true;
  if (FALSY.has(value))  return false;

  // Critically: we do NOT fall back to Boolean(value). "flase" must be an error,
  // not a silent `true`.
  fail(key, `expected one of true|false|1|0|yes|no|on|off, got ${JSON.stringify(value)}`);
  return false;
}

// `const` type parameter keeps the literal union instead of widening to string[].
// See 30 — Generic constraints.
function requiredEnum<const TValues extends readonly string[]>(
  key: string,
  values: TValues,
  opts: { default?: TValues[number] } = {},
): TValues[number] {
  const value = raw(key);

  if (value === undefined) {
    if (opts.default !== undefined) return opts.default;
    fail(key, `required, but not set (expected one of ${values.join(" | ")})`);
    return values[0]!;
  }
  if (!values.includes(value)) {
    fail(key, `expected one of ${values.join(" | ")}, got ${JSON.stringify(value)}`);
    return values[0]!;
  }
  return value as TValues[number];
}

function requiredUrl(key: string, opts: { protocols?: readonly string[] } = {}): string {
  const value = requiredString(key);
  if (value === "") return value;              // already reported as missing

  let parsedUrl: URL;
  try {
    parsedUrl = new URL(value);
  } catch {
    fail(key, `expected a valid URL, got ${JSON.stringify(value)}`);
    return value;
  }
  if (opts.protocols && !opts.protocols.includes(parsedUrl.protocol.replace(":", ""))) {
    fail(key, `expected protocol ${opts.protocols.join(" or ")}, got ${parsedUrl.protocol}`);
  }
  return value;
}

// A comma-separated list — very common for CORS origins, feature flags, etc.
function requiredList(key: string, opts: { default?: readonly string[] } = {}): readonly string[] {
  const value = raw(key);
  if (value === undefined) {
    if (opts.default !== undefined) return opts.default;
    fail(key, "required, but not set");
    return [];
  }
  return value.split(",").map((part) => part.trim()).filter((part) => part !== "");
}
```

Now the schema itself — plain object literal, no magic:

```ts
// ═══════════════════════════════════════════════════════════════════════════
// The configuration object
// ═══════════════════════════════════════════════════════════════════════════

const nodeEnv = requiredEnum("NODE_ENV", ["development", "test", "production"], {
  default: "development",
});

const isProduction = nodeEnv === "production";

const config = {
  nodeEnv,
  isProduction,
  isTest: nodeEnv === "test",

  port:            requiredInt("PORT", { min: 1, max: 65_535, default: 3000 }),
  databaseUrl:     requiredUrl("DATABASE_URL", { protocols: ["postgres", "postgresql"] }),
  dbPoolMax:       requiredInt("DB_POOL_MAX", { min: 1, max: 200, default: 10 }),
  redisUrl:        requiredUrl("REDIS_URL", { protocols: ["redis", "rediss"] }),

  // Production demands a real secret; development may use a throwaway one.
  jwtSecret:       isProduction
                     ? requiredString("JWT_SECRET", { minLength: 32 })
                     : optionalString("JWT_SECRET") ?? "insecure-development-secret",
  jwtTtlSeconds:   requiredInt("JWT_TTL_SECONDS", { min: 60, max: 86_400, default: 3600 }),

  logLevel:        requiredEnum("LOG_LEVEL", ["debug", "info", "warn", "error"], {
                     default: isProduction ? "info" : "debug",
                   }),

  enableBilling:   requiredBool("ENABLE_BILLING", { default: false }),
  enableRateLimit: requiredBool("ENABLE_RATE_LIMIT", { default: isProduction }),

  corsOrigins:     requiredList("CORS_ORIGINS", { default: ["http://localhost:5173"] }),
  maxUploadBytes:  requiredInt("MAX_UPLOAD_BYTES", { min: 1024, default: 5_242_880 }),

  sentryDsn:       optionalString("SENTRY_DSN"),
  stripeSecretKey: isProduction ? requiredString("STRIPE_SECRET_KEY") : optionalString("STRIPE_SECRET_KEY"),
} as const;

// ═══════════════════════════════════════════════════════════════════════════
// Fail fast — the whole point of the module
// ═══════════════════════════════════════════════════════════════════════════

if (issues.length > 0) {
  const width = Math.max(...issues.map((issue) => issue.key.length));
  console.error("\n✖ Invalid environment configuration:\n");
  for (const issue of issues) {
    console.error(`  • ${issue.key.padEnd(width)}  —  ${issue.message}`);
  }
  console.error(`\n  ${issues.length} problem(s). See .env.example for the full list.\n`);
  process.exit(1);              // exit code 1 → orchestrator will not route traffic
}

export const env = Object.freeze(config);
export type Env = typeof env;
```

Two design decisions worth calling out:

- **Readers return a placeholder on failure instead of throwing.** That's what allows one boot to report all five missing variables rather than making the developer fix them one deploy at a time. The placeholder is never observed because `process.exit(1)` runs before `env` is exported.
- **`as const` + `Object.freeze`** gives you a readonly, literal-typed config. `env.nodeEnv` is `"development" | "test" | "production"`, not `string`, so `switch` statements over it are exhaustive. See **65 — Readonly and immutability patterns**.

The resulting type, inferred with no hand-written interface:

```ts
type Env = {
  readonly nodeEnv:         "development" | "test" | "production";
  readonly isProduction:    boolean;
  readonly isTest:          boolean;
  readonly port:            number;
  readonly databaseUrl:     string;
  readonly dbPoolMax:       number;
  readonly redisUrl:        string;
  readonly jwtSecret:       string;
  readonly jwtTtlSeconds:   number;
  readonly logLevel:        "debug" | "info" | "warn" | "error";
  readonly enableBilling:   boolean;
  readonly enableRateLimit: boolean;
  readonly corsOrigins:     readonly string[];
  readonly maxUploadBytes:  number;
  readonly sentryDsn:       string | undefined;
  readonly stripeSecretKey: string | undefined;
};
```

### Concept 5 — The same thing with zod

zod replaces the hand-rolled readers with a declarative schema, and — the real win — **derives the TypeScript type from the schema** so the two can never drift apart.

```ts
// src/config/env.ts
import { z } from "zod";

// ═══════════════════════════════════════════════════════════════════════════
// The schema — one source of truth for validation AND types
// ═══════════════════════════════════════════════════════════════════════════

// A reusable boolean parser. Env booleans are strings; be explicit about which
// strings are accepted rather than relying on truthiness.
const envBoolean = z
  .enum(["true", "false", "1", "0", "yes", "no", "on", "off"])
  .transform((value) => value === "true" || value === "1" || value === "yes" || value === "on");

// A comma-separated list parser:
const envList = z
  .string()
  .transform((value) => value.split(",").map((part) => part.trim()).filter(Boolean));

const EnvSchema = z.object({
  // ── Runtime mode ──────────────────────────────────────────────────────────
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),

  // ── Server ────────────────────────────────────────────────────────────────
  // z.coerce.number() runs Number(input) BEFORE the number checks. "3000" → 3000.
  PORT: z.coerce.number().int().min(1).max(65_535).default(3000),
  HOST: z.string().default("0.0.0.0"),

  // ── Database ──────────────────────────────────────────────────────────────
  DATABASE_URL: z
    .string()
    .url("must be a valid URL")
    .refine((value) => value.startsWith("postgres://") || value.startsWith("postgresql://"), {
      message: "must be a postgres:// or postgresql:// URL",
    }),
  DB_POOL_MAX:            z.coerce.number().int().min(1).max(200).default(10),
  DB_CONNECT_TIMEOUT_MS:  z.coerce.number().int().positive().default(5_000),

  // ── Cache ─────────────────────────────────────────────────────────────────
  REDIS_URL: z.string().url().default("redis://127.0.0.1:6379"),

  // ── Auth ──────────────────────────────────────────────────────────────────
  JWT_SECRET:      z.string().min(32, "must be at least 32 characters"),
  JWT_TTL_SECONDS: z.coerce.number().int().min(60).max(86_400).default(3_600),

  // ── Observability ─────────────────────────────────────────────────────────
  LOG_LEVEL:  z.enum(["debug", "info", "warn", "error"]).default("info"),
  SENTRY_DSN: z.string().url().optional(),          // absent is fine

  // ── Feature flags ─────────────────────────────────────────────────────────
  ENABLE_BILLING:    envBoolean.default("false"),
  ENABLE_RATE_LIMIT: envBoolean.default("true"),

  // ── Third-party ───────────────────────────────────────────────────────────
  STRIPE_SECRET_KEY:     z.string().startsWith("sk_").optional(),
  STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_").optional(),

  // ── Lists ─────────────────────────────────────────────────────────────────
  CORS_ORIGINS: envList.default("http://localhost:5173"),
})
  // Cross-field rules go on the object, after the shape:
  .superRefine((value, ctx) => {
    if (value.NODE_ENV !== "production") return;

    if (value.ENABLE_BILLING && !value.STRIPE_SECRET_KEY) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        path: ["STRIPE_SECRET_KEY"],
        message: "required when ENABLE_BILLING=true",
      });
    }
    if (value.JWT_SECRET === "insecure-development-secret") {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        path: ["JWT_SECRET"],
        message: "the development placeholder must not be used in production",
      });
    }
    if (value.CORS_ORIGINS.includes("*")) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        path: ["CORS_ORIGINS"],
        message: "wildcard origin is not allowed in production",
      });
    }
  });

// ═══════════════════════════════════════════════════════════════════════════
// The inferred type — never written by hand, never out of date
// ═══════════════════════════════════════════════════════════════════════════

export type Env = z.infer<typeof EnvSchema>;
// {
//   NODE_ENV: "development" | "test" | "production";
//   PORT: number;              ← number, because of z.coerce.number()
//   HOST: string;
//   DATABASE_URL: string;
//   DB_POOL_MAX: number;
//   DB_CONNECT_TIMEOUT_MS: number;
//   REDIS_URL: string;
//   JWT_SECRET: string;
//   JWT_TTL_SECONDS: number;
//   LOG_LEVEL: "debug" | "info" | "warn" | "error";
//   SENTRY_DSN?: string | undefined;
//   ENABLE_BILLING: boolean;   ← boolean, because of .transform()
//   ENABLE_RATE_LIMIT: boolean;
//   STRIPE_SECRET_KEY?: string | undefined;
//   STRIPE_WEBHOOK_SECRET?: string | undefined;
//   CORS_ORIGINS: string[];
// }

// ═══════════════════════════════════════════════════════════════════════════
// safeParse + fail fast
// ═══════════════════════════════════════════════════════════════════════════

const parsed = EnvSchema.safeParse(process.env);
//                        ^^^^^^^^^ returns a result object; never throws.
//                        .parse() would throw a ZodError — usable, but then you
//                        must try/catch to produce a readable report.

if (!parsed.success) {
  const width = Math.max(...parsed.error.issues.map((issue) => issue.path.join(".").length));
  console.error("\n✖ Invalid environment configuration:\n");
  for (const issue of parsed.error.issues) {
    console.error(`  • ${issue.path.join(".").padEnd(width)}  —  ${issue.message}`);
  }
  console.error(`\n  ${parsed.error.issues.length} problem(s). See .env.example.\n`);
  process.exit(1);
}

export const env: Env = Object.freeze(parsed.data);
```

The `safeParse` result is a **discriminated union** (see **42 — Discriminated unions**), which is why the `if (!parsed.success)` block narrows `parsed.data` for you afterwards:

```ts
type SafeParseReturnType<TOutput> =
  | { success: true;  data: TOutput }
  | { success: false; error: ZodError };

const parsed = EnvSchema.safeParse(process.env);

parsed.data;                 // ❌ Property 'data' does not exist on type '{ success: false; … }'

if (!parsed.success) {
  parsed.error.issues;       // ✅ narrowed to the failure branch
  process.exit(1);
}

parsed.data;                 // ✅ narrowed to the success branch — Env
```

Note that `process.exit(1)` has return type `never`, which is what lets control-flow analysis know the code below is only reachable on success — no `!` required.

**`z.coerce.number()` vs `z.number()`** deserves a close look, because it is the single most common zod-env mistake:

```ts
z.number().parse("3000");
// ❌ ZodError: Expected number, received string
//    Every env var is a string, so a bare z.number() ALWAYS fails.

z.coerce.number().parse("3000");    // ✅ 3000  — runs Number() first
z.coerce.number().parse("");        // ⚠️  0    — Number("") === 0 !!
z.coerce.number().parse("abc");     // ❌ ZodError: Expected number, received nan
z.coerce.number().parse(undefined); // ❌ Required (unless .default() is present)

// The empty-string trap is real: `PORT=` in a .env file yields "" → 0 → invalid port.
// Guard it:
const port = z.coerce.number().int().min(1).max(65_535);
z.string().min(1).pipe(port).parse("");   // ❌ String must contain at least 1 character
```

Similarly, `z.coerce.boolean()` is almost always wrong for env vars:

```ts
z.coerce.boolean().parse("false");   // ⚠️ true — it's just Boolean("false")
z.coerce.boolean().parse("0");       // ⚠️ true — Boolean("0") === true
z.coerce.boolean().parse("");        // false — the only falsy string

// ✅ Always use an explicit enum + transform:
const envBoolean = z.enum(["true", "false"]).transform((value) => value === "true");
envBoolean.parse("false");           // false ✅
envBoolean.parse("FALSE");           // ❌ ZodError — case matters; add .toLowerCase() upstream
```

### Concept 6 — dotenv and loading order

`dotenv` reads a `.env` file and copies its keys into `process.env`. Two rules govern everything:

**Rule 1 — dotenv never overwrites an existing variable.**

```ts
// .env contains:  PORT=3000
// Shell runs:     PORT=8080 node server.js

import "dotenv/config";
process.env.PORT;      // "8080" — the real environment wins
```

This is deliberate and correct: real environment variables (from Docker, Kubernetes, CI, your shell) must beat a file checked into a developer's machine. `override: true` exists but you almost never want it.

**Rule 2 — dotenv must run before *anything* reads `process.env`.**

This is where ESM bites people. `import` statements are **hoisted and evaluated before any statement in the module body**, so this does not work:

```ts
// ❌ BROKEN under ESM — env.js is evaluated BEFORE dotenv's config() call
import { config } from "dotenv";
import { env } from "./config/env.js";     // ← this module runs first!

config();                                  // too late; env.js already read process.env
```

The evaluation order is: all imports (depth-first, in order), then the body. `./config/env.js` reads `process.env` during *its* module evaluation, which happens before `config()` on line 4 ever runs.

Four correct approaches, best first:

```ts
// ── Approach A — Node's built-in flag (Node 20.6+). No import at all. ────────
//    package.json:  "start": "node --env-file=.env dist/server.js"
//    Node loads the file before ANY user code executes. Nothing to get wrong.
//    Node 22+ also has --env-file-if-exists=.env for optional files.

// ── Approach B — preload via -r / --import (works on every Node version) ─────
//    "start": "node -r dotenv/config dist/server.js"          (CJS)
//    "start": "node --import dotenv/config dist/server.js"    (ESM, Node 20.6+)

// ── Approach C — side-effect import, FIRST line of the entrypoint ────────────
import "dotenv/config";                    // ← must be the very first import
import { env } from "./config/env.js";     // now safe
import { createServer } from "./server.js";

// ── Approach D — load inside the env module itself, before reading ───────────
// src/config/env.ts
import { config } from "dotenv";
config();                                  // runs at module-eval time, before
const parsed = EnvSchema.safeParse(process.env);   // ← this line
```

Approach C has a sharp edge: import *sorting* tools (eslint `import/order`, some Prettier plugins, IDE "organize imports") will happily move `import "dotenv/config"` below your other imports and silently break your boot. Approaches A, B, and D are immune. If you must use C, pin it:

```ts
// eslint-disable-next-line import/order
import "dotenv/config";
```

**Multi-file loading order.** The convention (borrowed from Rails and adopted by Vite/Next) is a cascade, most specific first:

```
.env.production.local     ← machine-specific prod overrides   (gitignored)
.env.production           ← committed prod defaults           (no secrets!)
.env.local                ← machine-specific, all envs        (gitignored)
.env                      ← committed defaults                (no secrets!)
```

Because dotenv never overwrites, **load them in priority order — first file wins**:

```ts
// src/config/load-dotenv.ts
import { config } from "dotenv";
import { existsSync } from "node:fs";
import { resolve } from "node:path";

// NODE_ENV must be read from the REAL environment before any file is loaded —
// it decides which files to load, so it cannot come from a file.
const nodeEnv = process.env.NODE_ENV ?? "development";

// Highest priority first. dotenv will not overwrite keys already present,
// so the earliest file that defines a key wins.
const candidates = [
  `.env.${nodeEnv}.local`,
  `.env.${nodeEnv}`,
  // `.env.local` is intentionally skipped in tests so test runs are reproducible:
  ...(nodeEnv === "test" ? [] : [".env.local"]),
  ".env",
];

for (const fileName of candidates) {
  const fullPath = resolve(process.cwd(), fileName);
  if (existsSync(fullPath)) {
    config({ path: fullPath });          // silently no-ops for keys already set
  }
}
```

```ts
// src/config/env.ts
import "./load-dotenv.js";               // ← side-effect import, evaluated first
import { z } from "zod";

const EnvSchema = z.object({ /* … */ });
// …
```

Because `./load-dotenv.js` is imported *by* `env.ts`, ESM guarantees it finishes evaluating before `env.ts`'s body runs. That ordering is structural, not conventional — nothing can reorder it.

**In production, do not use dotenv at all.** Real deployments inject variables through the platform (Kubernetes Secrets, ECS task definitions, Fly secrets, Railway/Render dashboards). `.env` files are a *local development* convenience. Committing production secrets to a `.env` file is one of the most common ways credentials leak.

### Concept 7 — Per-environment defaults

Some variables should differ by environment, and there are two very different ways to express that.

**Approach A — defaults computed from `NODE_ENV`.** Simple, keeps one schema.

```ts
const nodeEnv = z
  .enum(["development", "test", "production"])
  .default("development")
  .parse(process.env.NODE_ENV);

const isProduction = nodeEnv === "production";
const isTest       = nodeEnv === "test";

const EnvSchema = z.object({
  NODE_ENV: z.literal(nodeEnv),

  // Chatty logs locally, structured info logs in prod, silence in tests:
  LOG_LEVEL: z
    .enum(["debug", "info", "warn", "error", "silent"])
    .default(isTest ? "silent" : isProduction ? "info" : "debug"),

  // Small pool locally, large pool in prod:
  DB_POOL_MAX: z.coerce.number().int().min(1).default(isProduction ? 50 : 5),

  // Rate limiting on in prod, off locally so you can hammer the API:
  ENABLE_RATE_LIMIT: envBoolean.default(isProduction ? "true" : "false"),

  // A local fallback URL is fine; in prod there must be no fallback at all:
  DATABASE_URL: isProduction
    ? z.string().url()
    : z.string().url().default("postgres://postgres:postgres@localhost:5432/app_dev"),
});
```

**Approach B — a base schema plus per-environment refinements.** More explicit; better when production genuinely requires *more* variables.

```ts
// Everything every environment needs:
const BaseEnvSchema = z.object({
  NODE_ENV:     z.enum(["development", "test", "production"]),
  PORT:         z.coerce.number().int().min(1).max(65_535).default(3000),
  DATABASE_URL: z.string().url(),
  LOG_LEVEL:    z.enum(["debug", "info", "warn", "error", "silent"]).default("info"),
});

// Production requires additional secrets that simply don't exist locally:
const ProductionEnvSchema = BaseEnvSchema.extend({
  NODE_ENV:              z.literal("production"),
  JWT_SECRET:            z.string().min(32),
  SENTRY_DSN:            z.string().url(),
  STRIPE_SECRET_KEY:     z.string().startsWith("sk_live_"),
  STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_"),
  AWS_REGION:            z.string().min(1),
  S3_BUCKET:             z.string().min(1),
});

const DevelopmentEnvSchema = BaseEnvSchema.extend({
  NODE_ENV:          z.literal("development"),
  JWT_SECRET:        z.string().min(8).default("dev-only-secret"),
  STRIPE_SECRET_KEY: z.string().startsWith("sk_test_").optional(),
});

const TestEnvSchema = BaseEnvSchema.extend({
  NODE_ENV:     z.literal("test"),
  JWT_SECRET:   z.string().default("test-secret"),
  DATABASE_URL: z.string().url().default("postgres://postgres:postgres@localhost:5432/app_test"),
  LOG_LEVEL:    z.literal("silent").default("silent"),
});

// A discriminated union — zod picks the branch by NODE_ENV and reports only
// the issues relevant to THAT environment:
const EnvSchema = z.discriminatedUnion("NODE_ENV", [
  ProductionEnvSchema,
  DevelopmentEnvSchema,
  TestEnvSchema,
]);

export type Env = z.infer<typeof EnvSchema>;
// A union! In production code you narrow on env.NODE_ENV to reach prod-only keys:
//
//   if (env.NODE_ENV === "production") {
//     env.SENTRY_DSN;        // ✅ string — only exists on the production branch
//   }
//   env.SENTRY_DSN;          // ❌ does not exist on all branches
```

The union type is honest but occasionally annoying. A common compromise is Approach B for *validation* and a flattened accessor for *consumption*:

```ts
const parsed = EnvSchema.safeParse(process.env);
if (!parsed.success) { /* report + exit */ }

export const env = Object.freeze({
  ...parsed.data,
  isProduction:  parsed.data.NODE_ENV === "production",
  isDevelopment: parsed.data.NODE_ENV === "development",
  isTest:        parsed.data.NODE_ENV === "test",
});
```

### Concept 8 — Never import `process.env` outside the env module

This is the architectural rule that makes everything above actually hold. Stated plainly:

> **`process.env` may appear in exactly one file: `src/config/env.ts`.**

Why it matters:

```ts
// ── Without the rule ────────────────────────────────────────────────────────
// src/mailer.ts
const apiKey = process.env.SENDGRID_API_KEY!;      // ← a second, unvalidated read

// Consequences:
// 1. It is NOT in the schema, so booting with it missing succeeds. The failure
//    happens on the first password-reset email, three days after deploy.
// 2. It is NOT in .env.example, so the next developer never sets it.
// 3. It is invisible to `grep -r "process.env" src/config` audits.
// 4. Mocking it in tests requires poking global state instead of a module.
// 5. Its type is `string`-by-assertion — a lie, again.
```

```ts
// ── With the rule ───────────────────────────────────────────────────────────
// src/config/env.ts  — add it to the schema:
SENDGRID_API_KEY: z.string().startsWith("SG.").optional(),

// src/mailer.ts
import { env } from "./config/env.js";
const apiKey = env.SENDGRID_API_KEY;               // string | undefined, honestly
if (!apiKey) throw new Error("Mailer disabled: SENDGRID_API_KEY not configured");
```

Enforce it mechanically — a convention nobody checks is not a convention.

```jsonc
// .eslintrc.json — no-restricted-properties
{
  "rules": {
    "no-restricted-properties": ["error", {
      "object": "process",
      "property": "env",
      "message": "Import { env } from '@/config/env' instead of reading process.env directly."
    }]
  },
  "overrides": [
    {
      // The env module is the one legal place:
      "files": ["src/config/env.ts", "src/config/load-dotenv.ts"],
      "rules": { "no-restricted-properties": "off" }
    }
  ]
}
```

A CI grep works too, and catches the case where someone disables the lint rule:

```bash
# fails the build if process.env appears outside the config directory
! grep -rn "process\.env" src --include="*.ts" | grep -v "^src/config/"
```

Beyond safety, there are three practical wins:

**Testability.** A module is trivially mockable; a global is not.

```ts
// tests/order-service.test.ts
import { vi, test, expect } from "vitest";

vi.mock("../src/config/env.js", () => ({
  env: {
    NODE_ENV: "test",
    PORT: 3000,
    DATABASE_URL: "postgres://localhost:5432/app_test",
    ENABLE_BILLING: false,             // ← flip a flag without touching globals
    MAX_UPLOAD_BYTES: 1024,
  },
}));

const { createOrder } = await import("../src/services/order-service.js");

test("skips charging when billing is disabled", async () => {
  const order = await createOrder({ userId: 42, amountCents: 4999 });
  expect(order.chargeId).toBeNull();
});
```

**Discoverability.** `src/config/env.ts` *is* the documentation. Generating `.env.example` from it is a five-line script:

```ts
// scripts/generate-env-example.ts
import { EnvSchema } from "../src/config/env.js";

const lines = Object.entries(EnvSchema.shape).map(([key, schema]) => {
  const isOptional = schema.isOptional();
  const description = schema.description ?? "";
  return `${isOptional ? "# " : ""}${key}=${description ? `  # ${description}` : ""}`;
});

console.log(lines.join("\n"));
```

**Refactorability.** Renaming `DATABASE_URL` to `POSTGRES_URL` touches one file, and every consumer keeps using `env.databaseUrl`. With `process.env` scattered across 40 files, it's a 40-file grep-and-pray.

---

## Example 1 — basic

```ts
// A minimal, dependency-free validated env module for a small Express service.
// src/config/env.ts

// ── Step 0: load .env files BEFORE reading process.env ──────────────────────
// (Prefer `node --env-file=.env` in package.json. Shown here for completeness.)
import { config as loadDotenv } from "dotenv";
loadDotenv();

// ── Step 1: an issue collector, so one boot reports EVERY problem ───────────

interface EnvIssue {
  key:     string;
  message: string;
}

const issues: EnvIssue[] = [];

// Treat "" and "   " as absent. `DATABASE_URL=` in a .env is a real mistake
// people make, and "" is not a usable value for anything.
function readRaw(key: string): string | undefined {
  const value = process.env[key]?.trim();
  return value === undefined || value === "" ? undefined : value;
}

// ── Step 2: typed readers ───────────────────────────────────────────────────

function requiredString(key: string, minLength = 1): string {
  const value = readRaw(key);
  if (value === undefined) {
    issues.push({ key, message: "required, but not set" });
    return "";
  }
  if (value.length < minLength) {
    issues.push({ key, message: `must be at least ${minLength} characters` });
  }
  return value;                       // ✅ narrowed to string by the check above
}

function optionalString(key: string): string | undefined {
  return readRaw(key);
}

function requiredInt(key: string, fallback: number, min = 1, max = Number.MAX_SAFE_INTEGER): number {
  const value = readRaw(key);
  if (value === undefined) return fallback;

  // Number(), not parseInt(): parseInt("3000abc") silently returns 3000.
  const parsed = Number(value);
  if (!Number.isInteger(parsed)) {
    issues.push({ key, message: `expected an integer, got ${JSON.stringify(value)}` });
    return fallback;
  }
  if (parsed < min || parsed > max) {
    issues.push({ key, message: `must be between ${min} and ${max}, got ${parsed}` });
    return fallback;
  }
  return parsed;
}

function requiredBool(key: string, fallback: boolean): boolean {
  const value = readRaw(key)?.toLowerCase();
  if (value === undefined) return fallback;
  if (value === "true" || value === "1")  return true;
  if (value === "false" || value === "0") return false;

  // Deliberately NOT Boolean(value) — "false" would become true.
  issues.push({ key, message: `expected true|false|1|0, got ${JSON.stringify(value)}` });
  return fallback;
}

// `const TValues` preserves the literal tuple so the return type is a union,
// not `string`. See 30 — Generic constraints.
function requiredEnum<const TValues extends readonly string[]>(
  key: string,
  allowed: TValues,
  fallback: TValues[number],
): TValues[number] {
  const value = readRaw(key);
  if (value === undefined) return fallback;
  if (!allowed.includes(value)) {
    issues.push({ key, message: `expected one of ${allowed.join(" | ")}, got ${JSON.stringify(value)}` });
    return fallback;
  }
  return value as TValues[number];
}

// ── Step 3: the configuration shape ─────────────────────────────────────────

const nodeEnv = requiredEnum("NODE_ENV", ["development", "test", "production"], "development");

const config = {
  nodeEnv,
  isProduction: nodeEnv === "production",

  port:        requiredInt("PORT", 3000, 1, 65_535),
  databaseUrl: requiredString("DATABASE_URL"),

  // In production a weak secret is a vulnerability, so raise the bar there:
  jwtSecret:   requiredString("JWT_SECRET", nodeEnv === "production" ? 32 : 8),

  logLevel:    requiredEnum("LOG_LEVEL", ["debug", "info", "warn", "error"], "info"),
  enableDocs:  requiredBool("ENABLE_DOCS", nodeEnv !== "production"),

  sentryDsn:   optionalString("SENTRY_DSN"),
} as const;

// ── Step 4: fail fast, loudly, with every problem listed ────────────────────

if (issues.length > 0) {
  const width = Math.max(...issues.map((issue) => issue.key.length));
  console.error("\n✖ Invalid environment configuration:\n");
  for (const issue of issues) {
    console.error(`  • ${issue.key.padEnd(width)}  —  ${issue.message}`);
  }
  console.error("");
  process.exit(1);           // `never` return type — TS knows code below is unreachable
}

export const env = Object.freeze(config);
export type Env = typeof env;
```

```ts
// ── Using it — src/server.ts ────────────────────────────────────────────────
import express from "express";
import { env } from "./config/env.js";

const app = express();

app.get("/health", (req, res) => {
  res.json({
    ok: true,
    data: {
      status:  "up",
      env:     env.nodeEnv,        // ✅ "development" | "test" | "production"
      version: process.env.npm_package_version ?? "unknown",
    },
  });
});

if (env.enableDocs) {             // ✅ a real boolean, not a truthy string
  app.get("/docs", (req, res) => { res.send("<h1>API docs</h1>"); });
}

app.listen(env.port, () => {      // ✅ a real number
  console.log(`Listening on :${env.port} in ${env.nodeEnv} mode`);
});

// env.databseUrl;
//     ^^^^^^^^^^ ❌ Property 'databseUrl' does not exist on type 'Env'.
//                   Did you mean 'databaseUrl'?
```

```
# What a bad boot looks like — 200ms, then dead:

$ node dist/server.js

✖ Invalid environment configuration:

  • DATABASE_URL  —  required, but not set
  • JWT_SECRET    —  must be at least 32 characters
  • PORT          —  expected an integer, got "three thousand"
  • LOG_LEVEL     —  expected one of debug | info | warn | error, got "verbose"

$ echo $?
1
```

---

## Example 2 — real world backend use case

```ts
// A production-grade configuration module for a multi-service Node backend:
// zod validation, per-environment schemas, secret redaction, a typed feature-flag
// helper, and a startup banner. This is the shape used in real services.

// ═══════════════════════════════════════════════════════════════════════════
// src/config/load-dotenv.ts — deterministic .env cascade
// ═══════════════════════════════════════════════════════════════════════════

import { config as loadDotenvFile } from "dotenv";
import { existsSync } from "node:fs";
import { resolve } from "node:path";

// NODE_ENV decides which files to load, so it must come from the real
// environment — never from a file.
const nodeEnvRaw = process.env.NODE_ENV ?? "development";

// Highest priority first; dotenv never overwrites an already-set key,
// so the FIRST file that defines a variable wins.
const dotenvFiles = [
  `.env.${nodeEnvRaw}.local`,          // machine-specific, gitignored
  `.env.${nodeEnvRaw}`,                // committed per-env defaults
  ...(nodeEnvRaw === "test" ? [] : [".env.local"]),   // tests stay reproducible
  ".env",                              // committed shared defaults
];

// In production, secrets come from the orchestrator, not from files. Loading
// .env there is at best pointless and at worst a way to ship stale values.
if (nodeEnvRaw !== "production") {
  for (const fileName of dotenvFiles) {
    const fullPath = resolve(process.cwd(), fileName);
    if (existsSync(fullPath)) loadDotenvFile({ path: fullPath });
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// src/config/env.ts — the single source of truth
// ═══════════════════════════════════════════════════════════════════════════

import "./load-dotenv.js";              // ← side-effect import; ESM guarantees it
                                        //   fully evaluates before this body runs
import { z } from "zod";

// ── Reusable env-specific primitives ────────────────────────────────────────

/** Env booleans are strings. Be explicit — never rely on truthiness. */
const envBoolean = z
  .string()
  .transform((value) => value.trim().toLowerCase())
  .pipe(z.enum(["true", "false", "1", "0", "yes", "no", "on", "off"]))
  .transform((value) => value === "true" || value === "1" || value === "yes" || value === "on");

/** Comma-separated list → readonly string[], empty entries dropped. */
const envList = z
  .string()
  .transform((value) => value.split(",").map((part) => part.trim()).filter(Boolean))
  .pipe(z.array(z.string().min(1)).readonly());

/** An integer that rejects "" (which Number() would turn into 0). */
const envInt = z.string().trim().min(1).pipe(z.coerce.number().int());

/** A duration like "30s", "5m", "2h" → milliseconds. */
const envDuration = z
  .string()
  .regex(/^\d+(ms|s|m|h|d)$/, "expected a duration like 30s, 5m, 2h, 7d")
  .transform((value) => {
    const amount = Number.parseInt(value, 10);
    const unit = value.replace(/^\d+/, "") as "ms" | "s" | "m" | "h" | "d";
    const multipliers = { ms: 1, s: 1_000, m: 60_000, h: 3_600_000, d: 86_400_000 } as const;
    return amount * multipliers[unit];
  });

// ── The base schema — everything every environment needs ────────────────────

const BaseEnvSchema = z.object({
  NODE_ENV:    z.enum(["development", "test", "production"]).default("development"),
  SERVICE_NAME: z.string().min(1).default("api"),
  GIT_SHA:     z.string().length(40).optional(),      // injected by CI

  // ── HTTP ──────────────────────────────────────────────────────────────────
  HOST:                 z.string().min(1).default("0.0.0.0"),
  PORT:                 envInt.pipe(z.number().min(1).max(65_535)).default(3000),
  REQUEST_TIMEOUT:      envDuration.default("30s"),
  BODY_LIMIT_BYTES:     envInt.pipe(z.number().positive()).default(1_048_576),
  TRUST_PROXY_HOPS:     envInt.pipe(z.number().min(0).max(10)).default(0),
  CORS_ORIGINS:         envList.default("http://localhost:5173"),

  // ── PostgreSQL ────────────────────────────────────────────────────────────
  DATABASE_URL: z
    .string()
    .url("must be a valid URL")
    .refine((value) => /^postgres(ql)?:\/\//.test(value), "must use the postgres:// scheme"),
  DB_POOL_MIN:          envInt.pipe(z.number().min(0).max(50)).default(2),
  DB_POOL_MAX:          envInt.pipe(z.number().min(1).max(200)).default(10),
  DB_STATEMENT_TIMEOUT: envDuration.default("10s"),
  DB_SSL:               envBoolean.default("false"),

  // ── Redis ─────────────────────────────────────────────────────────────────
  REDIS_URL:        z.string().url().default("redis://127.0.0.1:6379"),
  REDIS_KEY_PREFIX: z.string().default("api:"),

  // ── Auth ──────────────────────────────────────────────────────────────────
  JWT_SECRET:       z.string().min(32, "must be at least 32 characters"),
  JWT_ISSUER:       z.string().min(1).default("api.example.com"),
  ACCESS_TOKEN_TTL:  envDuration.default("15m"),
  REFRESH_TOKEN_TTL: envDuration.default("30d"),
  BCRYPT_ROUNDS:     envInt.pipe(z.number().min(10).max(15)).default(12),

  // ── Observability ─────────────────────────────────────────────────────────
  LOG_LEVEL:  z.enum(["trace", "debug", "info", "warn", "error", "silent"]).default("info"),
  LOG_PRETTY: envBoolean.default("false"),
  SENTRY_DSN: z.string().url().optional(),
  OTEL_EXPORTER_OTLP_ENDPOINT: z.string().url().optional(),

  // ── Feature flags ─────────────────────────────────────────────────────────
  ENABLE_BILLING:      envBoolean.default("false"),
  ENABLE_RATE_LIMIT:   envBoolean.default("true"),
  ENABLE_WEBHOOKS:     envBoolean.default("false"),
  ENABLE_SWAGGER_DOCS: envBoolean.default("true"),

  // ── Third-party ───────────────────────────────────────────────────────────
  STRIPE_SECRET_KEY:     z.string().startsWith("sk_").optional(),
  STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_").optional(),
  SENDGRID_API_KEY:      z.string().startsWith("SG.").optional(),
  AWS_REGION:            z.string().regex(/^[a-z]{2}-[a-z]+-\d$/).default("us-east-1"),
  S3_BUCKET:             z.string().min(3).optional(),
});

// ── Production tightenings — rules that only apply when it matters ──────────

const EnvSchema = BaseEnvSchema.superRefine((value, ctx) => {
  const require = (key: keyof typeof value, reason: string): void => {
    if (value[key] === undefined) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, path: [key], message: `required ${reason}` });
    }
  };

  if (value.ENABLE_BILLING) {
    require("STRIPE_SECRET_KEY", "when ENABLE_BILLING=true");
  }
  if (value.ENABLE_WEBHOOKS) {
    require("STRIPE_WEBHOOK_SECRET", "when ENABLE_WEBHOOKS=true");
  }

  if (value.DB_POOL_MIN > value.DB_POOL_MAX) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["DB_POOL_MIN"],
      message: `must be <= DB_POOL_MAX (${value.DB_POOL_MAX})`,
    });
  }

  if (value.NODE_ENV !== "production") return;

  // ── Production-only invariants ────────────────────────────────────────────
  require("SENTRY_DSN", "in production");
  require("GIT_SHA", "in production (set by CI)");

  if (!value.DB_SSL) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["DB_SSL"],
      message: "must be true in production",
    });
  }
  if (value.CORS_ORIGINS.includes("*")) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["CORS_ORIGINS"],
      message: "wildcard origin is forbidden in production",
    });
  }
  if (value.CORS_ORIGINS.some((origin) => origin.startsWith("http://"))) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["CORS_ORIGINS"],
      message: "all production origins must use https://",
    });
  }
  if (value.STRIPE_SECRET_KEY?.startsWith("sk_test_")) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["STRIPE_SECRET_KEY"],
      message: "a TEST key is configured in production",
    });
  }
  if (value.LOG_PRETTY) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["LOG_PRETTY"],
      message: "must be false in production (log aggregators need JSON)",
    });
  }
});

// ── Parse, report every issue, exit non-zero ────────────────────────────────

type RawEnv = z.infer<typeof EnvSchema>;

function parseEnv(source: NodeJS.ProcessEnv): RawEnv {
  const result = EnvSchema.safeParse(source);

  if (result.success) return result.data;

  const rows = result.error.issues.map((issue) => ({
    key:     issue.path.join(".") || "(root)",
    message: issue.message,
  }));
  const width = Math.max(...rows.map((row) => row.key.length));

  console.error("\n" + "═".repeat(72));
  console.error("  ✖  INVALID ENVIRONMENT CONFIGURATION — refusing to start");
  console.error("═".repeat(72) + "\n");
  for (const row of rows) {
    console.error(`  • ${row.key.padEnd(width)}   ${row.message}`);
  }
  console.error(`\n  ${rows.length} problem(s). See .env.example for the full list.`);
  console.error("═".repeat(72) + "\n");

  // Exit code 1 tells Docker/Kubernetes the container failed to start, so the
  // rollout halts instead of routing traffic to a broken instance.
  process.exit(1);
}

const raw = parseEnv(process.env);

// ── Derive a friendlier, grouped shape ──────────────────────────────────────

export const env = Object.freeze({
  nodeEnv:       raw.NODE_ENV,
  serviceName:   raw.SERVICE_NAME,
  gitSha:        raw.GIT_SHA,
  isProduction:  raw.NODE_ENV === "production",
  isDevelopment: raw.NODE_ENV === "development",
  isTest:        raw.NODE_ENV === "test",

  http: Object.freeze({
    host:            raw.HOST,
    port:            raw.PORT,
    requestTimeoutMs: raw.REQUEST_TIMEOUT,
    bodyLimitBytes:  raw.BODY_LIMIT_BYTES,
    trustProxyHops:  raw.TRUST_PROXY_HOPS,
    corsOrigins:     raw.CORS_ORIGINS,
  }),

  database: Object.freeze({
    url:                raw.DATABASE_URL,
    poolMin:            raw.DB_POOL_MIN,
    poolMax:            raw.DB_POOL_MAX,
    statementTimeoutMs: raw.DB_STATEMENT_TIMEOUT,
    ssl:                raw.DB_SSL,
  }),

  redis: Object.freeze({
    url:       raw.REDIS_URL,
    keyPrefix: raw.REDIS_KEY_PREFIX,
  }),

  auth: Object.freeze({
    jwtSecret:         raw.JWT_SECRET,
    jwtIssuer:         raw.JWT_ISSUER,
    accessTokenTtlMs:  raw.ACCESS_TOKEN_TTL,
    refreshTokenTtlMs: raw.REFRESH_TOKEN_TTL,
    bcryptRounds:      raw.BCRYPT_ROUNDS,
  }),

  logging: Object.freeze({
    level:  raw.LOG_LEVEL,
    pretty: raw.LOG_PRETTY,
  }),

  features: Object.freeze({
    billing:     raw.ENABLE_BILLING,
    rateLimit:   raw.ENABLE_RATE_LIMIT,
    webhooks:    raw.ENABLE_WEBHOOKS,
    swaggerDocs: raw.ENABLE_SWAGGER_DOCS,
  }),

  integrations: Object.freeze({
    sentryDsn:           raw.SENTRY_DSN,
    otlpEndpoint:        raw.OTEL_EXPORTER_OTLP_ENDPOINT,
    stripeSecretKey:     raw.STRIPE_SECRET_KEY,
    stripeWebhookSecret: raw.STRIPE_WEBHOOK_SECRET,
    sendgridApiKey:      raw.SENDGRID_API_KEY,
    awsRegion:           raw.AWS_REGION,
    s3Bucket:            raw.S3_BUCKET,
  }),
});

export type Env = typeof env;

// ── A typed feature-flag accessor ───────────────────────────────────────────

export type FeatureName = keyof Env["features"];

export function isFeatureEnabled(feature: FeatureName): boolean {
  return env.features[feature];
}
// isFeatureEnabled("billing");   ✅
// isFeatureEnabled("bulling");   ❌ Argument of type '"bulling"' is not assignable
//                                   to parameter of type 'FeatureName'

// ── Redaction — so the startup banner never leaks a secret ──────────────────

const SECRET_KEY_PATTERN = /secret|token|password|key|dsn|url/i;

function redact(value: unknown, keyName: string): unknown {
  if (typeof value !== "string") return value;
  if (!SECRET_KEY_PATTERN.test(keyName)) return value;
  if (value.length <= 8) return "«redacted»";
  return `${value.slice(0, 4)}…${value.slice(-2)} (${value.length} chars)`;
}

export function describeEnv(): Record<string, unknown> {
  const flatten = (obj: Record<string, unknown>, prefix = ""): Record<string, unknown> =>
    Object.entries(obj).reduce<Record<string, unknown>>((acc, [key, value]) => {
      const path = prefix ? `${prefix}.${key}` : key;
      if (value !== null && typeof value === "object" && !Array.isArray(value)) {
        Object.assign(acc, flatten(value as Record<string, unknown>, path));
      } else {
        acc[path] = redact(value, key);
      }
      return acc;
    }, {});

  return flatten(env as unknown as Record<string, unknown>);
}
```

```ts
// ═══════════════════════════════════════════════════════════════════════════
// src/server.ts — consuming it. Note: zero occurrences of `process.env`.
// ═══════════════════════════════════════════════════════════════════════════

import express from "express";
import { Pool } from "pg";
import { env, isFeatureEnabled, describeEnv } from "./config/env.js";

// ── Database — every option is a validated, correctly-typed value ───────────
const pool = new Pool({
  connectionString:       env.database.url,
  min:                    env.database.poolMin,        // number
  max:                    env.database.poolMax,        // number
  statement_timeout:      env.database.statementTimeoutMs,
  ssl:                    env.database.ssl ? { rejectUnauthorized: true } : false,
});

const app = express();

app.set("trust proxy", env.http.trustProxyHops);
app.use(express.json({ limit: env.http.bodyLimitBytes }));

// ── CORS from a validated list ──────────────────────────────────────────────
app.use((req, res, next) => {
  const requestOrigin = req.header("origin");
  if (requestOrigin && env.http.corsOrigins.includes(requestOrigin)) {
    res.setHeader("access-control-allow-origin", requestOrigin);
  }
  next();
});

// ── Feature-flagged routes ──────────────────────────────────────────────────
if (isFeatureEnabled("swaggerDocs")) {
  app.get("/docs", (req, res) => { res.send("<h1>API Docs</h1>"); });
}

if (isFeatureEnabled("billing")) {
  // Narrowing: the schema guarantees the key exists when the flag is on, but
  // the TYPE is still `string | undefined`, so narrow honestly rather than `!`.
  const stripeSecretKey = env.integrations.stripeSecretKey;
  if (!stripeSecretKey) throw new Error("unreachable: superRefine guarantees this");

  app.post("/billing/charge", async (req, res) => {
    // …use stripeSecretKey — `string` here, proven by the check above
    res.json({ ok: true, data: { chargeId: "ch_123" } });
  });
}

// ── Auth uses validated durations ───────────────────────────────────────────
function issueAccessToken(userId: number): { token: string; expiresAt: Date } {
  const expiresAt = new Date(Date.now() + env.auth.accessTokenTtlMs);   // number ✅
  return { token: signJwt({ sub: userId, iss: env.auth.jwtIssuer }), expiresAt };
}

// ── Health/readiness expose safe config, never secrets ──────────────────────
app.get("/health", (req, res) => {
  res.json({
    ok: true,
    data: {
      service: env.serviceName,
      env:     env.nodeEnv,
      version: env.gitSha?.slice(0, 7) ?? "dev",
      uptime:  Math.floor(process.uptime()),
    },
  });
});

// ── Startup banner — redacted, so logs are safe to share ────────────────────
app.listen(env.http.port, env.http.host, () => {
  console.log(
    JSON.stringify({
      level:   "info",
      message: `${env.serviceName} listening on ${env.http.host}:${env.http.port}`,
      config:  env.isProduction ? undefined : describeEnv(),
      //       ^^ full (redacted) config locally; nothing in prod
    }),
  );
});

declare function signJwt(payload: { sub: number; iss: string }): string;
```

```jsonc
// package.json — dotenv is never imported by application code
{
  "scripts": {
    "dev":   "node --env-file=.env --watch src/server.ts",
    "test":  "NODE_ENV=test vitest run",
    "build": "tsc -p tsconfig.build.json",
    "start": "NODE_ENV=production node dist/server.js",
    "check:env": "node dist/config/env.js && echo '✓ environment is valid'"
  }
}
```

The `check:env` script is a small but high-value trick: because `env.ts` validates and exits on import, running it standalone is a **configuration smoke test** you can run in CI or as a Kubernetes init container, before the real process starts.

---

## Going deeper

### `process.env` is a magic object, not a plain one

In Node, `process.env` is backed by a C++ interceptor. It behaves in ways plain objects do not:

```ts
// 1. All values are coerced to strings on write:
(process.env as Record<string, unknown>).PORT = 3000;
process.env.PORT;                 // "3000" — a string, silently

(process.env as Record<string, unknown>).FLAG = true;
process.env.FLAG;                 // "true"

(process.env as Record<string, unknown>).OBJ = { a: 1 };
process.env.OBJ;                  // "[object Object]" 😱

// 2. Assigning null/undefined does NOT delete — it stringifies:
(process.env as Record<string, unknown>).REDIS_URL = undefined;
process.env.REDIS_URL;            // "undefined" — the STRING. Truthy!
// Use `delete process.env.REDIS_URL` instead.

// 3. Spreading gives you a normal, decoupled object:
const snapshot = { ...process.env };    // plain object, no coercion magic
delete process.env.PORT;
snapshot.PORT;                          // still there — it's a copy

// 4. On Windows, keys are CASE-INSENSITIVE:
process.env.Path === process.env.PATH;  // true on Windows, false on Linux/macOS
```

Point 2 is a genuine trap in tests:

```ts
// ❌ Sets the string "undefined", which passes every truthiness check:
process.env.SENTRY_DSN = undefined as unknown as string;

// ✅ Actually removes it:
delete process.env.SENTRY_DSN;
```

Point 1 is why validating a `{ ...process.env }` snapshot is slightly safer than validating the live object — zod's `.transform()` results never accidentally get stringified back.

### The env module runs at import time — order still matters

`src/config/env.ts` does its work during **module evaluation**, not inside a function. That has consequences.

```ts
// The moment ANY module imports env.ts, validation runs and may exit the process.
import { env } from "./config/env.js";
```

This is usually exactly what you want (fail before anything else initialises), but it makes the module hard to test and impossible to reuse with a different source. The fix is to export both a function and the eager singleton:

```ts
// Testable core:
export function loadEnv(source: NodeJS.ProcessEnv = process.env): Env {
  const result = EnvSchema.safeParse(source);
  if (!result.success) throw new EnvValidationError(result.error.issues);
  return Object.freeze(buildEnv(result.data));
}

// Eager singleton for the app — exits rather than throws:
function loadEnvOrExit(): Env {
  try {
    return loadEnv();
  } catch (error) {
    if (error instanceof EnvValidationError) { error.print(); process.exit(1); }
    throw error;
  }
}

export const env = loadEnvOrExit();
```

```ts
// Now unit tests can exercise validation without killing the test runner:
import { loadEnv, EnvValidationError } from "../src/config/env.js";

test("rejects a non-numeric PORT", () => {
  expect(() => loadEnv({ ...validBaseEnv, PORT: "three thousand" }))
    .toThrow(EnvValidationError);
});

test("rejects http origins in production", () => {
  expect(() => loadEnv({ ...validBaseEnv, NODE_ENV: "production", CORS_ORIGINS: "http://x.com" }))
    .toThrow(/https/);
});
```

But note: importing `env.ts` at all still triggers the singleton. If you want *pure* testability, move the schema to `env.schema.ts` and keep the singleton in `env.ts`.

### `process.exit()` truncates stdout — sometimes

`process.exit()` terminates immediately. If stdout is a **pipe** (which it is under Docker, PM2, and most CI runners) writes are asynchronous, and your carefully formatted error report can be cut off.

```ts
// ⚠️ The error message may never appear when stdout is piped:
console.error(bigMultiLineReport);
process.exit(1);
```

Two robust fixes:

```ts
// ── Fix A — set the exit code and let Node drain naturally ──────────────────
console.error(bigMultiLineReport);
process.exitCode = 1;
throw new Error("Invalid environment configuration");   // unwinds, flushes, exits 1

// ── Fix B — write synchronously to fd 2 ─────────────────────────────────────
import { writeSync } from "node:fs";
writeSync(2, bigMultiLineReport + "\n");   // blocking write to stderr
process.exit(1);
```

Fix B is what serious config libraries do. For most apps, Fix A is enough, and it has the bonus of producing a stack trace.

### `z.coerce` runs `Number()`, with all its quirks

`z.coerce.number()` is literally `Number(input)` followed by the numeric checks. `Number()` has surprising behaviour on exactly the inputs env files produce:

```ts
Number("");            // 0        ← `PORT=` becomes 0
Number("  ");          // 0        ← whitespace too
Number("0x10");        // 16       ← hex is accepted
Number("1e3");         // 1000     ← exponent notation is accepted
Number("Infinity");    // Infinity ← .int() catches this, .positive() does not
Number("1_000");       // NaN      ← underscores are NOT accepted
Number("3,000");       // NaN
Number(null);          // 0        ← irrelevant here, but worth knowing
Number(undefined);     // NaN
```

So `z.coerce.number().int().positive()` accepts `"0x10"` and `"1e3"`. If you need strictly decimal input:

```ts
const strictInt = z
  .string()
  .regex(/^-?\d+$/, "must be a plain decimal integer")
  .transform(Number)
  .pipe(z.number().int());

strictInt.parse("3000");    // 3000 ✅
strictInt.parse("0x10");    // ❌ must be a plain decimal integer
strictInt.parse("1e3");     // ❌
strictInt.parse("");        // ❌
```

`.pipe()` is the general tool here: validate the *string* shape first, transform, then validate the *number*.

### `.default()` applies to `undefined`, not to `""`

A subtle and very common failure:

```ts
const schema = z.object({ PORT: z.coerce.number().default(3000) });

schema.parse({});                  // { PORT: 3000 } ✅ — key absent
schema.parse({ PORT: undefined }); // { PORT: 3000 } ✅
schema.parse({ PORT: "" });        // { PORT: 0 }    ⚠️ — "" is present, so the
                                   //                     default does NOT apply,
                                   //                     and Number("") === 0
```

A `.env` line of `PORT=` produces `""`, not `undefined`. Normalise empty strings to `undefined` **before** parsing:

```ts
function withoutEmptyValues(source: NodeJS.ProcessEnv): Record<string, string | undefined> {
  return Object.fromEntries(
    Object.entries(source).map(([key, value]) => [
      key,
      value === undefined || value.trim() === "" ? undefined : value.trim(),
    ]),
  );
}

const parsed = EnvSchema.safeParse(withoutEmptyValues(process.env));
```

This one preprocessing step eliminates an entire class of bug. Do it in every project.

### `z.infer` vs `z.input` — transforms make them differ

When a schema has `.transform()` or `.coerce`, the type going *in* differs from the type coming *out*:

```ts
const PortSchema = z.coerce.number().int();

type PortInput  = z.input<typeof PortSchema>;    // unknown (coerce accepts anything)
type PortOutput = z.output<typeof PortSchema>;   // number
type PortInfer  = z.infer<typeof PortSchema>;    // number  — `infer` === `output`

const FlagSchema = z.enum(["true", "false"]).transform((v) => v === "true");
type FlagInput  = z.input<typeof FlagSchema>;    // "true" | "false"
type FlagOutput = z.output<typeof FlagSchema>;   // boolean
```

For env work you almost always want `z.infer` / `z.output` — the parsed, coerced, usable type. `z.input` is occasionally useful for typing test fixtures that feed the schema raw strings:

```ts
const testEnv: z.input<typeof EnvSchema> = {
  NODE_ENV: "test",
  PORT: "3000",              // ✅ a string, as it would be in reality
  DATABASE_URL: "postgres://localhost/test",
  JWT_SECRET: "x".repeat(32),
};
```

### `Object.freeze` is shallow, and only enforced at runtime

```ts
export const env = Object.freeze({
  http: { port: 3000, corsOrigins: ["https://app.example.com"] },
});

env.http = { port: 1 };            // ❌ TypeError in strict mode (frozen)
env.http.port = 9999;              // ⚠️ WORKS — freeze is shallow!
env.http.corsOrigins.push("*");    // ⚠️ WORKS — arrays inside are not frozen
```

Freeze every level, and pair it with `readonly` types so the compiler helps too:

```ts
export const env = Object.freeze({
  http: Object.freeze({
    port:        3000,
    corsOrigins: Object.freeze(["https://app.example.com"]) as readonly string[],
  }),
});

env.http.port = 9999;
//       ^^^^ ❌ Cannot assign to 'port' because it is a read-only property
```

Note the compile-time `readonly` comes from `as const` or `Readonly<T>`, not from `Object.freeze` itself — although TypeScript does infer `readonly` for `Object.freeze` on object *literals*. See **65 — Readonly and immutability patterns**.

### Secrets in env vars leak through logs, errors, and `/proc`

Typing does not make a secret safe. Three specific leaks worth guarding:

```ts
// 1. Logging the whole config object:
console.log("Booting with", env);          // ⚠️ prints JWT_SECRET to your log
                                           //    aggregator, forever, searchable

// 2. Error messages that echo the value:
throw new Error(`Invalid DATABASE_URL: ${process.env.DATABASE_URL}`);
// ⚠️ that URL contains the database password. Now it's in Sentry.
// ✅ Never echo the value; name the key only:
throw new Error("DATABASE_URL is malformed (value redacted)");

// 3. Connection URLs in stack traces from pg/mongoose/redis clients.
//    Mitigate by parsing and passing components:
const dbUrl = new URL(env.database.url);
const pool = new Pool({
  host:     dbUrl.hostname,
  port:     Number(dbUrl.port || 5432),
  user:     decodeURIComponent(dbUrl.username),
  password: decodeURIComponent(dbUrl.password),   // never re-serialised into logs
  database: dbUrl.pathname.slice(1),
});
```

Also remember that on Linux, `/proc/<pid>/environ` exposes the whole environment to any process running as the same user, and `docker inspect` shows env vars set via `-e` or `ENV`. Env vars are convenient, not secret-grade. For high-value secrets, prefer mounted files or a secrets manager, and let the env var hold only the *path*:

```ts
const JWT_SECRET = z
  .string()
  .transform((path) => readFileSync(path, "utf8").trim())
  .pipe(z.string().min(32));
// JWT_SECRET_FILE=/run/secrets/jwt_secret   ← Docker/K8s secret mount
```

### Build-time vs run-time: bundlers inline `process.env`

If your Node code is bundled (esbuild, webpack, Next.js server bundles), `process.env.X` may be **statically replaced at build time** with a literal:

```ts
// Source:
if (process.env.NODE_ENV === "production") { /* … */ }

// After esbuild --define:process.env.NODE_ENV='"production"':
if ("production" === "production") { /* … */ }
```

That is fine for `NODE_ENV`, but catastrophic for secrets — a build-time-inlined value is **baked into the artifact** and cannot be changed by the runtime environment. Two rules:

- Never let a bundler inline anything but `NODE_ENV`.
- Read env vars through a **dynamic** access in bundled code so it can't be statically replaced: `process.env["DATABASE_URL"]` with a computed key, or simply keep the env module out of the bundle (`external`).

This is also why the "one env module" rule pays off: there is exactly one file to audit for bundler interference.

### Typing a partial `ProcessEnv` for tests

When you build test fixtures, `NodeJS.ProcessEnv` is inconvenient because it's an index signature. Define your own strict source type instead:

```ts
// The exact keys the schema reads, all as strings — matching reality:
type RawEnvSource = Partial<Record<keyof z.input<typeof EnvSchema>, string>>;

const baseTestEnv: RawEnvSource = {
  NODE_ENV:     "test",
  PORT:         "3000",
  DATABASE_URL: "postgres://postgres:postgres@localhost:5432/app_test",
  JWT_SECRET:   "0123456789abcdef0123456789abcdef",
};

test("PORT must be within the valid range", () => {
  expect(() => loadEnv({ ...baseTestEnv, PORT: "70000" })).toThrow(/65535/);
});
```

`keyof z.input<typeof EnvSchema>` derives the key list from the schema, so adding a variable automatically widens the fixture type. See **47 — keyof and typeof operators**.

---

## Common mistakes

### Mistake 1 — `!` or `as string` instead of a real check

```ts
// ❌ Both erase at compile time. The value is still undefined at runtime.
const databaseUrl = process.env.DATABASE_URL!;
const jwtSecret   = process.env.JWT_SECRET as string;
const redisUrl    = <string>process.env.REDIS_URL;

databaseUrl.startsWith("postgres");   // 💥 TypeError: Cannot read properties of undefined

// ❌ The `??` variant is worse — it hides the failure permanently:
const stripeKey = process.env.STRIPE_SECRET_KEY ?? "";
// boots fine, charges fail silently in production

// ✅ Check at boot; let control-flow analysis do the narrowing:
function requireEnv(key: string): string {
  const value = process.env[key];
  if (value === undefined || value.trim() === "") {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;                       // string — earned, not asserted
}

const databaseUrl2 = requireEnv("DATABASE_URL");   // ✅ genuinely a string
```

### Mistake 2 — declaring `ProcessEnv` members as `number` or `boolean`

```ts
// ❌ A lie that TypeScript will happily propagate through your whole codebase:
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      PORT: number;              // it is a STRING at runtime
      ENABLE_CACHE: boolean;     // it is a STRING at runtime
    }
  }
}

app.listen(process.env.PORT);            // "3000" passed where number expected
if (process.env.ENABLE_CACHE) { /* … */ } // "false" is truthy → always runs

const total = process.env.PORT + 1;      // TS says number; runtime gives "30001"

// ✅ Declare them as strings (the truth), and coerce in the env module:
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      PORT?: string;
      ENABLE_CACHE?: string;
    }
  }
}

export const env = {
  port:         Number(requireEnv("PORT")),
  enableCache:  requireEnv("ENABLE_CACHE") === "true",
} as const;
```

### Mistake 3 — `Boolean(process.env.X)` for feature flags

```ts
// ❌ Every non-empty string is truthy:
const enableBilling = Boolean(process.env.ENABLE_BILLING);
// ENABLE_BILLING=false  → true   😱
// ENABLE_BILLING=0      → true   😱
// ENABLE_BILLING=no     → true   😱
// ENABLE_BILLING=       → false  (the only falsy case)

// ❌ Same bug, different spelling:
if (process.env.ENABLE_BILLING) { chargeCustomer(); }
const enabled = !!process.env.ENABLE_BILLING;

// ❌ JSON.parse looks clever but throws on anything unexpected:
const flag = JSON.parse(process.env.ENABLE_BILLING ?? "false");
// ENABLE_BILLING=yes → SyntaxError at boot, with a useless message

// ✅ An explicit allow-list, with an error on anything else:
function parseBool(key: string, fallback: boolean): boolean {
  const value = process.env[key]?.trim().toLowerCase();
  if (value === undefined || value === "") return fallback;
  if (value === "true"  || value === "1" || value === "yes") return true;
  if (value === "false" || value === "0" || value === "no")  return false;
  throw new Error(`${key}: expected true|false|1|0|yes|no, got ${JSON.stringify(value)}`);
}

const enableBilling2 = parseBool("ENABLE_BILLING", false);   // ✅ real boolean
```

### Mistake 4 — `parseInt` without validating the result

```ts
// ❌ parseInt stops at the first non-digit and returns a partial number:
const port = parseInt(process.env.PORT ?? "");
// PORT unset       → NaN
// PORT="3000abc"   → 3000  (silently ignores the garbage)
// PORT="  3000  "  → 3000  (fine, but you didn't check)
// PORT="0x10"      → 0     (parseInt with radix 10 stops at 'x')

app.listen(port);                 // listen(NaN) → binds a RANDOM free port 😱

// ❌ NaN poisons comparisons without ever throwing:
const maxUploadBytes = parseInt(process.env.MAX_UPLOAD_BYTES ?? "");
if (requestBody.length > maxUploadBytes) reject();
// NaN comparisons are always false → the limit is silently disabled

// ✅ Number() + an explicit integer check + a range check:
function parseInteger(key: string, fallback: number, min: number, max: number): number {
  const raw = process.env[key]?.trim();
  if (raw === undefined || raw === "") return fallback;

  const parsed = Number(raw);
  if (!Number.isInteger(parsed)) {
    throw new Error(`${key}: expected an integer, got ${JSON.stringify(raw)}`);
  }
  if (parsed < min || parsed > max) {
    throw new Error(`${key}: must be between ${min} and ${max}, got ${parsed}`);
  }
  return parsed;
}

const port2 = parseInteger("PORT", 3000, 1, 65_535);   // ✅ never NaN, never 0
```

### Mistake 5 — reading `process.env` outside the env module

```ts
// ❌ src/services/mailer.ts — a second, unvalidated, undocumented read:
export async function sendPasswordResetEmail(emailAddress: string): Promise<void> {
  const apiKey = process.env.SENDGRID_API_KEY!;      // not in the schema
  await sendgrid.send({ apiKey, to: emailAddress, template: "reset" });
  // Boots fine with the key missing. Fails on the first password reset,
  // days later, in production, on a user-facing flow.
}

// ❌ Even "harmless" reads spread the rot:
const isProd = process.env.NODE_ENV === "production";   // duplicated in 12 files

// ✅ Add it to the schema, then import the validated value:
// src/config/env.ts
//   SENDGRID_API_KEY: z.string().startsWith("SG.").optional(),

import { env } from "../config/env.js";

export async function sendPasswordResetEmail2(emailAddress: string): Promise<void> {
  if (!env.integrations.sendgridApiKey) {
    throw new Error("Password reset unavailable: SENDGRID_API_KEY is not configured");
  }
  await sendgrid.send({
    apiKey: env.integrations.sendgridApiKey,      // ✅ narrowed to string
    to: emailAddress,
    template: "reset",
  });
}

// ✅ And derive booleans once:
if (env.isProduction) { /* … */ }
```

Back the rule with lint (`no-restricted-properties` on `process.env`) plus a CI grep. Conventions that only live in a README get violated within two sprints.

### Mistake 6 — loading dotenv after something already read `process.env`

```ts
// ❌ ESM hoists imports — ./config/env.js is fully evaluated BEFORE config() runs.
import { config } from "dotenv";
import { env } from "./config/env.js";
config();
// → "DATABASE_URL required, but not set", even though .env defines it.

// ❌ Same class of bug — an import-sorter moved dotenv below the others:
import { createServer } from "./server.js";   // reads env transitively
import "dotenv/config";                       // too late

// ✅ Preload via the runtime, so no import order can break it:
//    "start": "node --env-file=.env dist/server.js"

// ✅ Or load inside the env module, before the parse:
// src/config/env.ts
import "dotenv/config";                       // side-effect, evaluated first
import { z } from "zod";
const parsed = EnvSchema.safeParse(process.env);
```

### Mistake 7 — trusting `.default()` to cover empty strings

```ts
// .env contains:  PORT=
// (a real, extremely common typo)

const EnvSchema = z.object({ PORT: z.coerce.number().int().default(3000) });
EnvSchema.parse(process.env).PORT;    // 0  — not 3000!
// "" is present, so .default() is skipped; Number("") === 0.
app.listen(0);                        // binds a random port. Health checks fail.

// ✅ Normalise "" → undefined before parsing:
const cleaned = Object.fromEntries(
  Object.entries(process.env).map(([key, value]) => [
    key,
    value?.trim() === "" ? undefined : value,
  ]),
);
EnvSchema.parse(cleaned).PORT;        // 3000 ✅
```

---

## Practice exercises

### Exercise 1 — easy

Write a dependency-free, fully typed env module for a small Express service. No `zod`, no `!`, no `as`.

Requirements:

1. `requireString(key: string): string` — throws `Error("Missing required environment variable: KEY")` if the variable is absent, empty, or whitespace-only. The return type must be `string` because of a real runtime check, not an assertion.
2. `optionalString(key: string): string | undefined`.
3. `requireInt(key: string, fallback: number): number` — uses `Number()` (not `parseInt`), rejects non-integers with a clear message, and returns `fallback` when the variable is absent.
4. `requireBool(key: string, fallback: boolean): boolean` — accepts `true|false|1|0` case-insensitively, throws on anything else, returns `fallback` when absent. `ENABLE_DOCS=false` must produce `false`.
5. Export a frozen `env` object with: `nodeEnv` (`"development" | "test" | "production"`, default `"development"`), `port` (default `3000`), `databaseUrl` (required), `authTokenTtlSeconds` (default `3600`), `enableDocs` (default `true`), and `sentryDsn` (optional).
6. Export `type Env = typeof env` and verify by hovering that `nodeEnv` is a **union of literals**, not `string`, and that `port` is `number`.
7. In a separate `server.ts`, use `env.port` in `app.listen(...)` and confirm that `env.databseUrl` is a compile error.

```ts
// Write your code here
```

### Exercise 2 — medium

Rebuild Exercise 1 with `zod`, and add error accumulation plus per-environment behaviour.

```ts
// Requirements:
//
// 1. Build `EnvSchema` with z.object covering:
//      NODE_ENV        "development" | "test" | "production", default "development"
//      PORT            integer 1–65535, default 3000            (coerced from string)
//      DATABASE_URL    a valid URL that must start with postgres:// or postgresql://
//      DB_POOL_MAX     integer 1–100, default 10
//      JWT_SECRET      string, min 32 chars
//      LOG_LEVEL       "debug"|"info"|"warn"|"error", default "info"
//      ENABLE_BILLING  a REAL boolean parsed from "true"/"false"/"1"/"0"
//                      (z.coerce.boolean() is forbidden — explain in a comment why)
//      CORS_ORIGINS    comma-separated string → readonly string[]
//      SENTRY_DSN      optional URL
//
// 2. Write a `preprocessEnv(source: NodeJS.ProcessEnv)` helper that converts
//    empty/whitespace-only values to `undefined`, so `.default()` actually applies.
//    Prove it: `PORT=` must yield 3000, not 0.
//
// 3. Use `.superRefine()` to add production-only rules:
//      - SENTRY_DSN is required when NODE_ENV === "production"
//      - CORS_ORIGINS must not contain "*" in production
//      - every production origin must start with "https://"
//      - JWT_SECRET must not equal "dev-secret" in production
//
// 4. Use `safeParse` and, on failure, print EVERY issue in an aligned table:
//        • DATABASE_URL   —  must use the postgres:// scheme
//        • JWT_SECRET     —  must be at least 32 characters
//    then `process.exitCode = 1` and throw (so stdout flushes — see Going deeper).
//
// 5. Export `type Env = z.infer<typeof EnvSchema>` and a frozen `env`.
//    Confirm PORT is `number` and ENABLE_BILLING is `boolean` in the inferred type,
//    WITHOUT writing that type by hand.
//
// 6. Export a testable `loadEnv(source: NodeJS.ProcessEnv = process.env): Env`
//    that THROWS instead of exiting, plus the eager `env` singleton that exits.
//    Write three assertions against loadEnv: a bad PORT, a short JWT_SECRET,
//    and a production config missing SENTRY_DSN.
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a reusable, fully generic env-parsing library — no zod — where the resulting config type is **entirely inferred from a declarative spec**.

```ts
// 1. Define a spec DSL:
//
//      type EnvVarSpec<TOut> = {
//        parse:     (raw: string) => TOut;   // throws with a human message on failure
//        required:  boolean;
//        default?:  TOut;
//        secret?:   boolean;                 // controls redaction in describeEnv()
//        description?: string;               // used to generate .env.example
//      };
//
//    Provide builders that produce correctly typed specs:
//      envString(opts?: { minLength?: number; pattern?: RegExp })      → EnvVarSpec<string>
//      envInt(opts?: { min?: number; max?: number })                   → EnvVarSpec<number>
//      envBool()                                                       → EnvVarSpec<boolean>
//      envEnum<const T extends readonly string[]>(...values: T)        → EnvVarSpec<T[number]>
//      envUrl(opts?: { protocols?: readonly string[] })                → EnvVarSpec<string>
//      envList<T>(inner: EnvVarSpec<T>, separator?: string)            → EnvVarSpec<readonly T[]>
//      envDuration()   // "30s" | "5m" | "2h" | "7d" → milliseconds    → EnvVarSpec<number>
//      envJson<T>(validate: (value: unknown) => T)                     → EnvVarSpec<T>
//    Plus combinators:
//      optional<T>(spec: EnvVarSpec<T>): EnvVarSpec<T | undefined>
//      withDefault<T>(spec: EnvVarSpec<T>, value: T): EnvVarSpec<T>
//      secret<T>(spec: EnvVarSpec<T>): EnvVarSpec<T>
//
// 2. Write `defineEnv` with FULL inference. Given:
//
//      const env = defineEnv({
//        NODE_ENV:       envEnum("development", "test", "production"),
//        PORT:           withDefault(envInt({ min: 1, max: 65535 }), 3000),
//        DATABASE_URL:   secret(envUrl({ protocols: ["postgres", "postgresql"] })),
//        ACCESS_TTL:     withDefault(envDuration(), 900_000),
//        CORS_ORIGINS:   envList(envUrl()),
//        SENTRY_DSN:     optional(envUrl()),
//        FEATURE_LIMITS: envJson<{ maxUsers: number }>(validateLimits),
//      });
//
//    the inferred type of `env` must be EXACTLY:
//
//      {
//        readonly NODE_ENV:       "development" | "test" | "production";
//        readonly PORT:           number;
//        readonly DATABASE_URL:   string;
//        readonly ACCESS_TTL:     number;
//        readonly CORS_ORIGINS:   readonly string[];
//        readonly SENTRY_DSN:     string | undefined;
//        readonly FEATURE_LIMITS: { maxUsers: number };
//      }
//
//    with NO hand-written interface anywhere. Hint: a mapped type over the spec
//    object plus `infer` (see 43 — Mapped types).
//
// 3. Error accumulation: a single call must report EVERY failing variable at once,
//    formatted as an aligned table, and then exit with code 1 WITHOUT truncating
//    stdout when it is a pipe.
//
// 4. Add `defineEnv(spec, { source })` so tests can pass a fake source, and make
//    the test path THROW an `EnvValidationError` (carrying `issues: EnvIssue[]`)
//    instead of exiting.
//
// 5. Add per-environment overrides that type-check:
//      defineEnv(spec, {
//        overrides: {
//          test:        { PORT: 0, LOG_LEVEL: "silent" },
//          production:  { requiredExtra: ["SENTRY_DSN", "GIT_SHA"] },
//        },
//      })
//    An override value must be assignable to the spec's OUTPUT type — supplying
//    `{ test: { PORT: "0" } }` must be a compile error.
//
// 6. Generate `.env.example` from the spec: required vars uncommented with an
//    empty value, optional vars commented out, each preceded by its `description`
//    and its constraints ("integer, 1–65535, default 3000").
//
// 7. Implement `describeEnv()` that returns the config with every `secret: true`
//    value redacted to `"ab…yz (48 chars)"`, and prove that DATABASE_URL never
//    appears in full in the output.
//
// 8. Demonstrate all of the following:
//      - the compile error from `env.DATABSE_URL` (typo)
//      - the compile error from a wrongly-typed override
//      - a boot failure listing 4 problems at once
//      - a test that asserts `loadEnv({ PORT: "70000" })` throws with /65535/
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ═══ What Node gives you ════════════════════════════════════════════════════
process.env.ANYTHING;             // string | undefined  — index signature
typeof process.env.PORT;          // "string" — ALWAYS. Never number/boolean.
delete process.env.PORT;          // ✅ removes it
process.env.PORT = undefined;     // ❌ sets the STRING "undefined"

// ═══ Never do this ══════════════════════════════════════════════════════════
process.env.DATABASE_URL!;        // ❌ erased at compile time, no runtime check
process.env.JWT_SECRET as string; // ❌ same lie, different syntax
process.env.SECRET ?? "dev";      // ❌ dangerous default reaches production
Boolean(process.env.FLAG);        // ❌ "false" → true
parseInt(process.env.PORT!);      // ❌ NaN, or "3000abc" → 3000

// ═══ Do this instead ════════════════════════════════════════════════════════
function requireEnv(key: string): string {
  const value = process.env[key];
  if (value === undefined || value.trim() === "") {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;                   // ✅ narrowed by a REAL check
}

// ═══ ProcessEnv augmentation (autocomplete only — validates NOTHING) ════════
// src/types/env.d.ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV?:     "development" | "test" | "production";
      PORT?:         string;      // string — never number
      DATABASE_URL?: string;
    }
  }
}
export {};                        // required: makes the file a module

// ═══ zod env schema ═════════════════════════════════════════════════════════
import { z } from "zod";

const envBoolean = z.enum(["true", "false"]).transform((v) => v === "true");

const EnvSchema = z.object({
  NODE_ENV:     z.enum(["development", "test", "production"]).default("development"),
  PORT:         z.coerce.number().int().min(1).max(65_535).default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET:   z.string().min(32),
  ENABLE_DOCS:  envBoolean.default("true"),
  SENTRY_DSN:   z.string().url().optional(),
});

type Env = z.infer<typeof EnvSchema>;               // parsed/output type
const parsed = EnvSchema.safeParse(process.env);    // never throws
if (!parsed.success) {
  for (const issue of parsed.error.issues) {
    console.error(`  • ${issue.path.join(".")} — ${issue.message}`);
  }
  process.exit(1);                                  // returns `never` → narrows below
}
export const env = Object.freeze(parsed.data);

// ═══ Coercion gotchas ═══════════════════════════════════════════════════════
z.number().parse("3000");           // ❌ Expected number, received string
z.coerce.number().parse("3000");    // ✅ 3000
z.coerce.number().parse("");        // ⚠️ 0    — normalise "" → undefined first
z.coerce.boolean().parse("false");  // ⚠️ true — NEVER use for env flags
z.string().min(1).pipe(z.coerce.number().int());    // ✅ rejects ""

// ═══ dotenv loading order ═══════════════════════════════════════════════════
// Best:  node --env-file=.env dist/server.js      (Node 20.6+, nothing to break)
// Good:  node --import dotenv/config dist/server.js
// OK:    import "dotenv/config";  // FIRST import; import-sorters may move it
// OK:    load inside src/config/env.ts, before safeParse
// dotenv NEVER overwrites an already-set variable → real env always wins.
// Cascade, first match wins: .env.{NODE_ENV}.local → .env.{NODE_ENV} → .env.local → .env

// ═══ The one rule ═══════════════════════════════════════════════════════════
// `process.env` appears in exactly ONE file: src/config/env.ts
// Enforce with eslint no-restricted-properties + a CI grep.
import { env } from "./config/env.js";   // everywhere else
```

| Thing | Type / behaviour | Key detail |
|---|---|---|
| `process.env.X` | `string \| undefined` | Index signature — typos compile fine |
| `process.env` values | always `string` | `PORT=3000` → `"3000"`; writes are `String()`-coerced |
| `process.env.X = undefined` | sets `"undefined"` | Use `delete process.env.X` |
| `!` / `as string` | erased at compile time | Zero runtime effect — a lie, not a check |
| `?? "default"` | silent fallback | Only safe when the default is correct **in production** |
| `Boolean(env)` | truthy check | `"false"`, `"0"`, `"no"` are all `true` |
| `parseInt(env)` | partial parse | `"3000abc"` → `3000`; unset → `NaN` |
| `Number(env)` | strict-ish | `""` → `0`, `"1e3"` → `1000`, `"abc"` → `NaN` |
| `NodeJS.ProcessEnv` augmentation | declaration merging | Autocomplete only — **validates nothing** |
| Augmentation file | must be in `tsconfig` `include` | Verify: `tsc --listFiles \| grep env.d.ts` |
| `z.coerce.number()` | `Number()` + checks | `""` → `0`; normalise empty strings first |
| `z.coerce.boolean()` | `Boolean()` | Never use for env flags — use `z.enum` + `.transform` |
| `.default(v)` | applies to `undefined` only | `PORT=` gives `""`, so the default is skipped |
| `z.infer<T>` | = `z.output<T>` | Post-transform type; `z.input<T>` is the raw string type |
| `safeParse` | `{success:true,data} \| {success:false,error}` | Discriminated union — narrows after `if (!ok)` |
| `process.exit(1)` | returns `never` | May truncate piped stdout — `writeSync(2, …)` or `exitCode` + throw |
| `Object.freeze` | shallow | Freeze each level; use `as const` for compile-time `readonly` |
| dotenv | never overwrites | Real env beats `.env`; load before anything reads `process.env` |
| Production secrets | not from `.env` | Use the orchestrator's secret store; `.env` is dev-only |

---

## Connected topics

- **03 — tsconfig in depth** — `strict` / `strictNullChecks` is what puts the `| undefined` on `process.env.X` in the first place; without it every mistake in this document compiles clean.
- **07 — Special types** — `undefined` and `never` (the return type of `process.exit`, which is what lets the compiler narrow after a fail-fast block).
- **09 — Union types** — `string | undefined` is the union you spend this entire document eliminating honestly.
- **11 — Literal types** — `NODE_ENV: "development" | "test" | "production"` is what makes `switch` over the environment exhaustive.
- **17 — Index signatures** — `ProcessEnv extends Dict<string>` is an index signature, which is precisely why typos on `process.env` are never caught.
- **32 — Utility types** — `Readonly`, `Partial<Record<K, string>>` for test fixtures, and `Pick` for slicing config subsets per module.
- **40 — Type narrowing** — the `if (value === undefined) throw` pattern is what legitimately turns `string | undefined` into `string`.
- **41 — Type guards** — custom guards and assertion functions for validated values (URL, port, non-empty string) and branded config types.
- **42 — Discriminated unions** — zod's `safeParse` result and `z.discriminatedUnion("NODE_ENV", …)` for per-environment schemas.
- **43 — Mapped types** — how Exercise 3's `defineEnv` infers the whole config type from a spec object.
- **47 — keyof and typeof operators** — `type Env = typeof env` and `keyof Env["features"]` for a typed feature-flag accessor.
- **49 — Declaration files** — `declare global { namespace NodeJS { interface ProcessEnv … } }` lives in a `.d.ts`, and must be inside your `include`.
- **58 — Typing API responses** — the same parse-at-the-boundary discipline, applied to data arriving over HTTP instead of from the OS.
- **60 — Error handling in TypeScript** — `EnvValidationError` carrying structured `issues`, and why fail-fast beats defensive checks scattered through the codebase.
- **62 — Strict null checks** — the compiler flag that makes `process.env.DATABASE_URL` honest.
- **65 — Readonly and immutability patterns** — `as const` + `Object.freeze` so nothing can mutate config at runtime.
- **67 — Repository pattern** — repositories receive validated config through constructor injection rather than reading `process.env` themselves.
