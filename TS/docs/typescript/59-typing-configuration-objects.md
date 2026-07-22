# 59 — Typing Configuration Objects

## What is this?

**Typing configuration objects** means treating your application's configuration the way you already treat your API contract: as *one* declared shape, built once at boot, validated once at boot, frozen, and then imported everywhere as a typed value — instead of a thousand scattered `process.env.SOMETHING` reads that each return `string | undefined` and each get parsed slightly differently.

The core artifact is a single module that exports one object:

```ts
// src/config/index.ts — the ONLY file in your app that reads process.env.
export const config: AppConfig = loadConfig(process.env);
Object.freeze(config);
```

Everything else in this doc exists to make that one line trustworthy:

- an `AppConfig` interface that describes every knob your service has,
- `as const` + `satisfies` so config *literals* keep their literal types while still being checked,
- `Record<Environment, AppConfig>` so `development` / `staging` / `production` are provably complete,
- `DeepReadonly<AppConfig>` so nothing mutates your config at 3am,
- `DeepPartial<AppConfig>` + a typed `deepMerge` so environment files only state their *differences*,
- feature flags as a typed record with an exhaustive key set,
- boot-time validation that turns a bad `DATABASE_URL` into a crash at second 0 instead of a `TypeError` at request 40,000.

---

## Why does it matter?

Untyped configuration is the single most common source of "it works on my machine" and of 3am pages. Not because config is hard, but because `process.env` is the worst-typed thing in Node:

```js
process.env.PORT              // string | undefined — always
process.env.MAX_RETRIES       // "3" — a string, and you WILL do "3" + 1 === "31"
process.env.ENABLE_BILLING    // "false" — and `Boolean("false") === true`
process.env.DATBASE_URL       // undefined, because you typo'd it. No error. Ever.
```

Every one of those failures is silent. `process.env.TYPO` is `undefined`, and `undefined` flows happily into a connection string, into `parseInt`, into a URL, and surfaces four layers away as `ECONNREFUSED ::1:NaN`.

Scattering these reads makes it worse:

- **You cannot enumerate your own config.** "What environment variables does this service need?" becomes a `grep` and a prayer.
- **Failures are lazy.** A missing `STRIPE_SECRET_KEY` is discovered the first time someone checks out — possibly days after deploy.
- **Parsing is inconsistent.** One file does `Number(process.env.PORT)`, another does `parseInt(process.env.PORT, 10)`, a third does `process.env.PORT || 3000` (a string!).
- **Tests are impossible to isolate.** Modules that read `process.env` at import time cannot be re-configured without `jest.resetModules()` gymnastics.
- **Nothing is documented.** The type *is* the documentation, and there isn't one.

A typed config module fixes all of that at once:

- **One shape** — `AppConfig` is the exhaustive, greppable list of every knob.
- **One read point** — exactly one file touches `process.env`; everything else imports a typed object.
- **Fail fast** — validation runs at boot. A misconfigured pod crash-loops instead of serving broken traffic.
- **Correct types downstream** — `config.server.port` is `number`, not `string | undefined`. No `!`, no `??`, no `Number(...)` at the call site.
- **Autocomplete and refactor safety** — rename `config.database.poolSize` and the compiler finds all 14 usages.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: process.env everywhere, parsed differently every time ────────

// src/db/pool.js
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,          // string | undefined → runtime crash
  max: process.env.DB_POOL_SIZE || 10,                 // "20" (a STRING) if set, 10 if not
  idleTimeoutMillis: parseInt(process.env.DB_IDLE_MS), // NaN if unset — silently disables timeout
});

// src/server.js
const port = process.env.PORT || 3000;                 // "8080" (string) or 3000 (number)
app.listen(port);                                      // works by accident, breaks on `port + 1`

// src/billing/stripe.js
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
// undefined in staging because nobody set it. Discovered by a customer.

// src/features.js
if (process.env.ENABLE_NEW_CHECKOUT) {                 // "false" is TRUTHY. Feature is always on.
  useNewCheckout();
}

// src/auth/jwt.js
const secret = process.env.JWT_SECRET ?? "dev-secret"; // 🔒 shipped to production. Nobody noticed.

// Questions you cannot answer without grepping the whole repo:
//   - Which env vars does this service require?
//   - Which ones have defaults? What are they?
//   - What happens in staging vs production?
```

```ts
// ── TypeScript: one declared shape, one load, one freeze ─────────────────────

// src/config/types.ts — the exhaustive contract.
export interface AppConfig {
  readonly environment: Environment;               // "development" | "staging" | "production"
  readonly server: {
    readonly port: number;                         // a NUMBER, parsed and validated once
    readonly requestTimeoutMs: number;
  };
  readonly database: {
    readonly url: string;                          // required — boot fails without it
    readonly poolSize: number;
    readonly idleTimeoutMs: number;
  };
  readonly auth: {
    readonly jwtSecret: string;                    // required in prod, no default
    readonly accessTokenTtlSeconds: number;
  };
  readonly features: FeatureFlags;                 // a typed record — no stray keys
}

// src/config/index.ts — THE ONLY place process.env is read.
export const config: DeepReadonly<AppConfig> = loadConfig(process.env);

// ── Everywhere else in the app ──────────────────────────────────────────────
import { config } from "./config";

const pool = new Pool({
  connectionString: config.database.url,           // ✅ string — guaranteed present
  max:              config.database.poolSize,      // ✅ number — already parsed
  idleTimeoutMillis: config.database.idleTimeoutMs,// ✅ number
});

app.listen(config.server.port);                    // ✅ number

if (config.features.newCheckout) {                 // ✅ boolean — "false" parsed correctly
  useNewCheckout();
}

// config.database.url = "postgres://evil";        // ❌ Cannot assign to 'url' — readonly
// config.databse.url;                             // ❌ Property 'databse' does not exist
```

The revelation: config is not "a bag of strings from the environment". It is a **value with a type**, produced by a function, at a known moment, and everything downstream should never know that `process.env` exists.

---

## Syntax

```ts
// ── 1. The environment union — closed, so a Record over it is exhaustive ────
export type Environment = "development" | "test" | "staging" | "production";

// ── 2. The config shape — deeply readonly by convention ─────────────────────
export interface AppConfig {
  readonly environment: Environment;
  readonly server: { readonly port: number; readonly host: string };
  readonly features: FeatureFlags;
}

// ── 3. `as const` — freeze a literal into its narrowest possible type ───────
const defaultConfig = {
  environment: "development",
  server: { port: 3000, host: "0.0.0.0" },
} as const;
// type: { readonly environment: "development"; readonly server: { readonly port: 3000; ... } }
//                                 ↑ literal "development", not string
//                                                              ↑ literal 3000, not number

// ── 4. `satisfies` — check against a type WITHOUT widening to it ────────────
const checkedConfig = {
  environment: "development",
  server: { port: 3000, host: "0.0.0.0" },
} as const satisfies AppConfigShape;
// ✅ checked against AppConfigShape (typos & missing keys error)
// ✅ but the inferred type is still the narrow literal type — best of both worlds

// ── 5. Environment-specific config as an exhaustive Record ─────────────────
const configByEnvironment: Record<Environment, AppConfig> = {
  development: { /* … */ },
  test:        { /* … */ },
  staging:     { /* … */ },
  production:  { /* … */ },
  // omit ANY key ⇒ compile error. Add a new Environment ⇒ compile error here.
};

// ── 6. DeepPartial — for overrides that state only their differences ───────
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// ── 7. DeepReadonly — for the frozen, exported result ─────────────────────
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// ── 8. Feature flags as a typed record over a closed key union ─────────────
export type FeatureFlagName = "newCheckout" | "betaSearch" | "auditLogging";
export type FeatureFlags = Readonly<Record<FeatureFlagName, boolean>>;

// ── 9. The single frozen export ───────────────────────────────────────────
export const config: DeepReadonly<AppConfig> = Object.freeze(loadConfig(process.env));
```

---

## How it works — concept by concept

### Concept 1 — One frozen module-level object, not scattered `process.env` reads

Start with the architectural decision, because every other technique depends on it.

There is exactly **one** module in your service that mentions `process.env`. It exports exactly **one** object. Nothing else in the codebase reads the environment. This is not stylistic preference — it buys four concrete properties:

```ts
// src/config/index.ts
// ══════════════════════════════════════════════════════════════════════════
//  This is the ONLY file in the repository allowed to touch process.env.
//  Enforce it with ESLint:
//    "no-restricted-properties": [
//      "error",
//      { object: "process", property: "env",
//        message: "Read config from src/config instead." }
//    ]
//  …with an override that re-enables it for this file only.
// ══════════════════════════════════════════════════════════════════════════

import { loadConfig } from "./load";
import type { AppConfig } from "./types";

/**
 * Built ONCE, at module load, before any request is served.
 * If the environment is invalid, this throws and the process exits — which is
 * exactly what you want: a crash-looping pod is visible; a silently
 * misconfigured pod serving 500s is not.
 */
export const config: AppConfig = loadConfig(process.env);

export type { AppConfig } from "./types";
```

The four properties you get:

1. **Enumerability.** `AppConfig` is a complete, greppable inventory of every knob. Onboarding a new engineer stops being an archaeology exercise.
2. **Fail-fast.** All parsing and validation happens at second 0. `DATABASE_URL` missing? The process never reaches `app.listen`.
3. **Single parsing policy.** `"8080"` becomes `8080` in one place, with one set of rules, tested once.
4. **Testability.** `loadConfig` is a pure function of an env-like object. Tests call `loadConfig({ ...baseEnv, NODE_ENV: "production" })` and assert — no module mocking, no global mutation.

Why *frozen*? Because a module-level object is shared mutable state, and shared mutable state is how you get "the pool size changed after a request handler tweaked config for a retry":

```ts
// Without freezing, this compiles at runtime and corrupts every later request:
config.database.poolSize = 100;   // TS blocks it via readonly, but a JS dependency,
                                  // a test helper, or an `as any` can still do it.

// Object.freeze makes it a runtime error in strict mode, a silent no-op otherwise.
```

`readonly` is a **compile-time** guarantee; `Object.freeze` is a **runtime** one. You want both. And `Object.freeze` is shallow, so you need a deep freeze — see Concept 6.

### Concept 2 — `as const`: keeping literals literal

By default TypeScript **widens** literals in a mutable position, because it assumes you might reassign them:

```ts
const defaults = {
  environment: "development",
  server: { port: 3000, host: "0.0.0.0" },
  logLevel: "info",
};

// Inferred type — everything widened:
// {
//   environment: string;
//   server: { port: number; host: string };
//   logLevel: string;
// }

type EnvOf = typeof defaults["environment"];   // string — useless
```

`string` is useless for config. You wanted `"development"`, so that `defaults.environment` can be checked against `Environment`, so that `defaults.logLevel` can feed a `Record<LogLevel, number>` lookup, so that a typo is a compile error.

`as const` turns off widening, recursively, and adds `readonly` everywhere:

```ts
const defaults = {
  environment: "development",
  server: { port: 3000, host: "0.0.0.0" },
  logLevel: "info",
  allowedOrigins: ["https://app.acme.com", "https://admin.acme.com"],
} as const;

// Inferred type:
// {
//   readonly environment: "development";
//   readonly server: { readonly port: 3000; readonly host: "0.0.0.0" };
//   readonly logLevel: "info";
//   readonly allowedOrigins: readonly ["https://app.acme.com", "https://admin.acme.com"];
// }
//                            ↑ a readonly TUPLE, not string[]

type EnvOf = typeof defaults["environment"];    // "development" ✅
```

Three things `as const` gives you that matter for config:

**(a) Deriving unions from values.** You write the data once and get the type for free:

```ts
export const SUPPORTED_CURRENCIES = ["USD", "EUR", "GBP", "INR"] as const;
export type Currency = typeof SUPPORTED_CURRENCIES[number];
//                     ↑ "USD" | "EUR" | "GBP" | "INR"

// Runtime check and compile-time type stay in sync forever:
export function isSupportedCurrency(value: string): value is Currency {
  return (SUPPORTED_CURRENCIES as readonly string[]).includes(value);
}
```

Without `as const`, `SUPPORTED_CURRENCIES` is `string[]` and `typeof SUPPORTED_CURRENCIES[number]` is `string`. The whole trick collapses.

**(b) Recursive readonly for free.** `as const` marks nested objects and arrays `readonly` all the way down — a poor man's `DeepReadonly` for literals you author by hand.

**(c) Exact keys for lookups.** `keyof typeof defaults` gives you the precise key union, which is what makes typed `get`/`has` helpers possible.

The catch — and it's a real one:

```ts
const defaults = { server: { port: 3000 } } as const;

interface ServerOptions { port: number }
function startServer(options: ServerOptions) { /* … */ }

startServer(defaults.server);          // ✅ readonly {port: 3000} → {port: number} is fine

interface MutableOptions { port: number }
const opts: MutableOptions = defaults.server;   // ✅ also fine — readonly is not enforced
                                                //    across assignment for properties

// But arrays DO bite:
const origins = ["https://app.acme.com"] as const;
function setOrigins(list: string[]) { /* … */ }
// setOrigins(origins);   // ❌ readonly ["…"] is not assignable to string[]
setOrigins([...origins]); // ✅ spread to a fresh mutable array
```

Rule of thumb: type your *consumers* as `readonly string[]` and the friction disappears.

### Concept 3 — `satisfies`: check without widening

`as const` gives you narrow types but **no checking**. A type annotation gives you checking but **widens**. `satisfies` gives you both.

Watch the three approaches fail and succeed on the same literal:

```ts
interface DatabaseConfig {
  readonly url: string;
  readonly poolSize: number;
  readonly sslMode: "disable" | "require" | "verify-full";
}

// ── (a) Annotation: checked ✅, but widened ❌ ──────────────────────────────
const dbA: DatabaseConfig = {
  url: "postgres://localhost:5432/acme",
  poolSize: 10,
  sslMode: "disable",
};
type SslA = typeof dbA["sslMode"];   // "disable" | "require" | "verify-full" — widened!
// You lost the information that THIS config uses "disable".

// ── (b) `as const`: narrow ✅, but unchecked ❌ ─────────────────────────────
const dbB = {
  url: "postgres://localhost:5432/acme",
  poolSize: 10,
  sslMoed: "disable",          // ← TYPO. Nothing complains. Ships to production.
} as const;
type SslB = typeof dbB["sslMoed"];   // "disable" — narrow, and wrong

// ── (c) `satisfies`: checked ✅ AND narrow ✅ ───────────────────────────────
const dbC = {
  url: "postgres://localhost:5432/acme",
  poolSize: 10,
  sslMode: "disable",
} satisfies DatabaseConfig;
type SslC = typeof dbC["sslMode"];   // "disable" ✅ narrow AND verified

// const dbD = {
//   url: "postgres://localhost:5432/acme",
//   poolSize: 10,
//   sslMoed: "disable",
// } satisfies DatabaseConfig;
// ❌ Object literal may only specify known properties, and 'sslMoed' does not
//    exist in type 'DatabaseConfig'.  ← the typo is now a build failure
// ❌ Property 'sslMode' is missing in type … but required in type 'DatabaseConfig'.
```

`as const satisfies T` combines them — narrowest possible inference, plus full checking, plus deep `readonly`:

```ts
const dbE = {
  url: "postgres://localhost:5432/acme",
  poolSize: 10,
  sslMode: "disable",
} as const satisfies DatabaseConfig;

// typeof dbE = {
//   readonly url: "postgres://localhost:5432/acme";
//   readonly poolSize: 10;
//   readonly sslMode: "disable";
// }
```

Where this pays off enormously is a **map of configs**, where you want per-key precision:

```ts
type Environment = "development" | "staging" | "production";

interface RegionConfig {
  readonly region: string;
  readonly replicaCount: number;
  readonly cdnEnabled: boolean;
}

// With an annotation you get a uniform, useless type:
const regionsAnnotated: Record<Environment, RegionConfig> = {
  development: { region: "local",      replicaCount: 1, cdnEnabled: false },
  staging:     { region: "us-east-1",  replicaCount: 2, cdnEnabled: false },
  production:  { region: "us-east-1",  replicaCount: 12, cdnEnabled: true },
};
type ProdReplicas = typeof regionsAnnotated["production"]["replicaCount"];  // number

// With `satisfies` you get per-key precision AND exhaustiveness checking:
const regions = {
  development: { region: "local",     replicaCount: 1,  cdnEnabled: false },
  staging:     { region: "us-east-1", replicaCount: 2,  cdnEnabled: false },
  production:  { region: "us-east-1", replicaCount: 12, cdnEnabled: true  },
} as const satisfies Record<Environment, RegionConfig>;

type ProdReplicas = typeof regions["production"]["replicaCount"];  // 12 ✅
type KnownEnvironments = keyof typeof regions;                     // "development" | "staging" | "production"

// And omitting one is still an error:
// const bad = {
//   development: { region: "local", replicaCount: 1, cdnEnabled: false },
// } as const satisfies Record<Environment, RegionConfig>;
// ❌ Type '{ development: … }' does not satisfy the expected type
//    'Record<Environment, RegionConfig>'. Property 'staging' is missing.
```

One more `satisfies` superpower: it validates *values* without collapsing *keys*. Compare:

```ts
type FeatureFlagName = "newCheckout" | "betaSearch" | "auditLogging";

// Annotation: keys are checked, but `keyof typeof` is the full union even if
// you only wrote some — no, actually Record<K,V> requires ALL keys. But the
// inferred type of the VALUE is Record<FeatureFlagName, boolean>, so you cannot
// tell which flags are on at the type level:
const flagsA: Record<FeatureFlagName, boolean> = {
  newCheckout: false, betaSearch: true, auditLogging: true,
};
type NewCheckoutA = typeof flagsA["newCheckout"];   // boolean

// satisfies: exhaustive AND you can read the literal value at the type level:
const flags = {
  newCheckout: false, betaSearch: true, auditLogging: true,
} as const satisfies Record<FeatureFlagName, boolean>;
type NewCheckoutB = typeof flags["newCheckout"];    // false ✅

// Which enables compile-time assertions about your own config:
type AuditMustBeOnInProd = typeof flags["auditLogging"] extends true ? true : never;
const _auditCheck: AuditMustBeOnInProd = true;      // ❌ breaks if someone flips it off
```

### Concept 4 — Environment-specific config via `Record<Environment, AppConfig>`

You have `development`, `test`, `staging`, `production`. The naive approach is a chain of `if`s. The typed approach is a `Record` keyed by a closed union, which is **exhaustive by construction**:

```ts
export type Environment = "development" | "test" | "staging" | "production";

export interface AppConfig {
  readonly environment: Environment;
  readonly server:   ServerConfig;
  readonly database: DatabaseConfig;
  readonly logging:  LoggingConfig;
  readonly features: FeatureFlags;
}

// ❌ The `if` chain: no exhaustiveness, easy to forget a branch.
function badGetConfig(env: string): AppConfig {
  if (env === "production") return productionConfig;
  if (env === "staging")    return stagingConfig;
  return developmentConfig;   // "test" silently gets dev config. Bug.
}

// ✅ The Record: adding "preview" to Environment breaks the build HERE,
//    which is the moment you actually want to be told.
const CONFIG_BY_ENVIRONMENT: Record<Environment, AppConfig> = {
  development: developmentConfig,
  test:        testConfig,
  staging:     stagingConfig,
  production:  productionConfig,
};

export function getConfigFor(environment: Environment): AppConfig {
  return CONFIG_BY_ENVIRONMENT[environment];   // total function — no undefined, no default
}
```

The critical detail: **`Record<K, V>` indexing returns `V`, not `V | undefined`**, precisely because the record is proven complete. That is only sound because `Environment` is a closed union. If you type the key as `string`, everything falls apart:

```ts
const loose: Record<string, AppConfig> = { development: developmentConfig };
const c = loose["staging"];        // typed AppConfig — but it's `undefined` at runtime! 🔥
                                   // Record<string, V> is a LIE without noUncheckedIndexedAccess.
```

Turn on `noUncheckedIndexedAccess` in `tsconfig.json` and `Record<string, V>` correctly yields `V | undefined`, while `Record<Environment, V>` still yields `V`. That single flag is worth the migration pain.

Getting from the raw env string to the union needs a guard, not a cast:

```ts
const ENVIRONMENTS = ["development", "test", "staging", "production"] as const;
export type Environment = typeof ENVIRONMENTS[number];

export function parseEnvironment(raw: string | undefined): Environment {
  if (raw === undefined) {
    throw new ConfigError("NODE_ENV is not set. Expected one of: " + ENVIRONMENTS.join(", "));
  }
  // The `includes` narrowing needs the widening cast; the return type re-narrows.
  if (!(ENVIRONMENTS as readonly string[]).includes(raw)) {
    throw new ConfigError(
      `NODE_ENV="${raw}" is not a known environment. Expected one of: ${ENVIRONMENTS.join(", ")}`,
    );
  }
  return raw as Environment;   // ✅ justified: we just proved membership at runtime
}

// ❌ Never do this instead:
// const environment = process.env.NODE_ENV as Environment;
//    A typo'd NODE_ENV="prodution" now flows in typed as "production" and
//    CONFIG_BY_ENVIRONMENT["prodution"] is `undefined` at runtime.
```

Now layer in the point of Concept 7: writing four *complete* configs is duplication. In practice `CONFIG_BY_ENVIRONMENT` holds four **overrides**, and each one is merged onto a shared base.

### Concept 5 — Feature flags as a typed record

Feature flags are config, and they rot faster than anything else. A typed record over a closed key union stops three specific problems: typo'd flag names, orphaned flags, and flags that silently default to "on".

```ts
// ── The closed set of flag names. This list IS the inventory. ───────────────
export const FEATURE_FLAG_NAMES = [
  "newCheckout",
  "betaSearch",
  "auditLogging",
  "asyncInvoiceExport",
  "strictRateLimiting",
] as const;

export type FeatureFlagName = typeof FEATURE_FLAG_NAMES[number];

// ── Every flag must have a value. No `?`, no partial. ──────────────────────
export type FeatureFlags = Readonly<Record<FeatureFlagName, boolean>>;

// ── Defaults: OFF. New code paths are opt-in, always. ──────────────────────
export const DEFAULT_FEATURE_FLAGS = {
  newCheckout:        false,
  betaSearch:         false,
  auditLogging:       true,    // safety feature: on by default, opt-OUT
  asyncInvoiceExport: false,
  strictRateLimiting: true,
} as const satisfies FeatureFlags;
// Add a name to FEATURE_FLAG_NAMES ⇒ this object errors until you give it a default. ✅
// Delete a name ⇒ this object errors for the now-unknown key. ✅ (orphan detection)

// ── Reading a flag: total, typed, no string keys at call sites ─────────────
import { config } from "../config";

export function isEnabled(flag: FeatureFlagName): boolean {
  return config.features[flag];         // ✅ boolean — Record over a closed union
}

if (isEnabled("newCheckout")) { /* … */ }
// if (isEnabled("newChekout")) { }     // ❌ Argument of type '"newChekout"' is not
//                                      //    assignable to parameter of type 'FeatureFlagName'
```

Parsing flags out of the environment is where typing earns its keep. Booleans from env vars are the classic footgun:

```ts
// ❌ Every one of these is wrong for FEATURE_NEW_CHECKOUT="false":
Boolean(process.env.FEATURE_NEW_CHECKOUT)          // true  — non-empty string
!!process.env.FEATURE_NEW_CHECKOUT                 // true
process.env.FEATURE_NEW_CHECKOUT ?? false          // "false" (a string, truthy)
JSON.parse(process.env.FEATURE_NEW_CHECKOUT)       // false ✅ but throws on "yes"

// ✅ One explicit parser, used for every flag:
const TRUTHY = new Set(["1", "true", "yes", "on"]);
const FALSY  = new Set(["0", "false", "no", "off"]);

export function parseBoolean(raw: string | undefined, fallback: boolean, varName: string): boolean {
  if (raw === undefined || raw === "") return fallback;
  const normalized = raw.trim().toLowerCase();
  if (TRUTHY.has(normalized)) return true;
  if (FALSY.has(normalized))  return false;
  throw new ConfigError(`${varName}="${raw}" is not a boolean. Use one of: true/false/1/0/yes/no/on/off`);
}

// ── Flags from env, derived mechanically from the name list ────────────────
/** "newCheckout" → "FEATURE_NEW_CHECKOUT" */
function envVarForFlag(name: FeatureFlagName): string {
  return "FEATURE_" + name.replace(/([A-Z])/g, "_$1").toUpperCase();
}

export function loadFeatureFlags(env: NodeJS.ProcessEnv, defaults: FeatureFlags): FeatureFlags {
  // Build via reduce over the NAME LIST, so a new flag is automatically supported.
  const entries = FEATURE_FLAG_NAMES.map((name) => {
    const varName = envVarForFlag(name);
    return [name, parseBoolean(env[varName], defaults[name], varName)] as const;
  });
  // `Object.fromEntries` loses the key precision, so re-assert against the type:
  return Object.fromEntries(entries) as FeatureFlags;
}
```

A more advanced variant: flags that carry *values*, not just booleans. Use a discriminated record so each flag's payload is exact:

```ts
export interface FeatureFlagSpec {
  readonly newCheckout:        boolean;
  readonly betaSearch:         boolean;
  readonly auditLogging:       boolean;
  readonly maxUploadMb:        number;               // a numeric knob
  readonly rolloutStrategy:    "off" | "percentage" | "allowlist";
  readonly rolloutPercentage:  number;               // 0–100
}

export type FeatureFlags = Readonly<FeatureFlagSpec>;

// A typed getter that returns the exact type per key — see `43 — Mapped types`.
export function getFlag<K extends keyof FeatureFlagSpec>(name: K): FeatureFlagSpec[K] {
  return config.features[name];
}

const maxUpload = getFlag("maxUploadMb");        // number ✅
const strategy  = getFlag("rolloutStrategy");    // "off" | "percentage" | "allowlist" ✅
```

### Concept 6 — Deep readonly config

`readonly` on an interface property is shallow. Nested objects stay mutable:

```ts
interface AppConfig {
  readonly database: { poolSize: number };
}

declare const config: AppConfig;
// config.database = { poolSize: 5 };     // ❌ blocked — the property is readonly
config.database.poolSize = 5;             // ✅ COMPILES — the nested object is not
```

You have three ways to fix this, and they compose.

**(a) Write `readonly` on every nested property by hand.** Verbose, explicit, and honestly fine for a config file you write once:

```ts
export interface AppConfig {
  readonly environment: Environment;
  readonly server: {
    readonly port: number;
    readonly host: string;
    readonly corsOrigins: readonly string[];   // note: readonly on the ARRAY too
  };
}
```

**(b) A `DeepReadonly<T>` mapped type.** One declaration, applied at the export boundary:

```ts
/**
 * Recursively marks every property readonly.
 * Arrays become `readonly T[]`; functions and primitives are left alone.
 */
export type DeepReadonly<T> =
  T extends (...args: readonly unknown[]) => unknown ? T                        // don't map functions
  : T extends readonly (infer Element)[] ? readonly DeepReadonly<Element>[]      // arrays & tuples
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }           // plain objects
  : T;                                                                          // primitives

interface MutableAppConfig {
  environment: Environment;
  server: { port: number; corsOrigins: string[] };
  database: { url: string; poolSize: number };
}

export type AppConfig = DeepReadonly<MutableAppConfig>;

declare const config: AppConfig;
// config.server.port = 4000;             // ❌ Cannot assign to 'port' — read-only property
// config.server.corsOrigins.push("x");   // ❌ Property 'push' does not exist on
//                                        //    'readonly DeepReadonly<string>[]'
```

Be careful with the built-in types you *don't* want mapped. `Date`, `Map`, `Set`, `RegExp` and `Buffer` all get their methods mangled by a naive `DeepReadonly`. Add an escape list:

```ts
type Immutable = string | number | boolean | bigint | symbol | null | undefined
               | Date | RegExp | ((...args: never[]) => unknown);

export type DeepReadonly<T> =
  T extends Immutable ? T
  : T extends ReadonlyMap<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends ReadonlySet<infer E> ? ReadonlySet<DeepReadonly<E>>
  : T extends readonly (infer E)[] ? readonly DeepReadonly<E>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

**(c) Runtime deep freeze.** Types vanish at runtime; a `JSON.parse` result, an `as any`, or a JS dependency can still mutate. Belt and braces:

```ts
/**
 * Recursively freezes an object and returns it typed as DeepReadonly.
 * Runs once, at boot, on a small object — the cost is irrelevant.
 */
export function deepFreeze<T>(value: T): DeepReadonly<T> {
  if (value !== null && typeof value === "object" && !Object.isFrozen(value)) {
    Object.freeze(value);
    for (const key of Object.getOwnPropertyNames(value)) {
      deepFreeze((value as Record<string, unknown>)[key]);
    }
  }
  return value as DeepReadonly<T>;
}

export const config = deepFreeze(loadConfig(process.env));
// In strict mode (which every ES module is), `config.server.port = 1` now THROWS.
```

Why bother with runtime freezing when `readonly` already errors? Because the failure you are guarding against is not the one your compiler sees. It is a test helper doing `(config as any).features.newCheckout = true`, leaking into the next test file. Frozen config makes that test fail loudly at the mutation site instead of mysteriously three files later.

### Concept 7 — Type-safe merging: `DeepPartial` + typed `deepMerge`

You do not want to write four complete configs. You want one **base** and three **overrides** that state only their differences. The type for "some subset of `AppConfig`, recursively" is `DeepPartial<AppConfig>`.

```ts
/**
 * Every property optional, recursively.
 * Arrays are treated as leaves — you replace them wholesale, never merge them
 * element-by-element (merging `corsOrigins` positionally would be nonsense).
 */
export type DeepPartial<T> =
  T extends Immutable ? T
  : T extends readonly (infer E)[] ? readonly E[]                     // arrays: replace, don't merge
  : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Now overrides are tiny and fully checked:
const productionOverrides: DeepPartial<AppConfig> = {
  server:   { port: 8080 },                 // ✅ `host` omitted — inherited from base
  database: { poolSize: 40, sslMode: "verify-full" },
  logging:  { level: "warn", pretty: false },
  features: { strictRateLimiting: true },
  // serevr: { port: 9090 },                // ❌ 'serevr' does not exist in DeepPartial<AppConfig>
  // database: { poolSize: "40" },          // ❌ Type 'string' is not assignable to 'number'
};
```

That last point is the whole reason to bother: an override file is checked against the *real* config shape, so a typo or a wrong-typed value is a build failure rather than a silently ignored key.

Now the merge itself. `Object.assign` and `{ ...base, ...override }` are **shallow** — they replace whole subtrees:

```ts
const base     = { server: { port: 3000, host: "0.0.0.0" }, database: { url: "…", poolSize: 10 } };
const override = { server: { port: 8080 } };

const shallow = { ...base, ...override };
// { server: { port: 8080 },  ← host is GONE. Your server binds to localhost by accident.
//   database: { url: "…", poolSize: 10 } }
```

A typed recursive merge:

```ts
/** True when `value` is a plain object we should recurse into (not array/Date/null). */
function isPlainObject(value: unknown): value is Record<string, unknown> {
  return (
    typeof value === "object" &&
    value !== null &&
    !Array.isArray(value) &&
    !(value instanceof Date) &&
    !(value instanceof RegExp) &&
    !(value instanceof Map) &&
    !(value instanceof Set)
  );
}

/**
 * Deeply merges `override` onto `base`.
 *
 * Rules, chosen deliberately:
 *  - plain objects   → merged recursively
 *  - arrays          → REPLACED wholesale (never concatenated, never index-merged)
 *  - `undefined`     → ignored, so `{ port: undefined }` does NOT erase the base value
 *  - everything else → replaced
 *
 * Returns a NEW object; neither argument is mutated.
 */
export function deepMerge<T>(base: T, override: DeepPartial<T>): T {
  if (!isPlainObject(base) || !isPlainObject(override)) {
    // Non-object base (or a scalar override): the override wins if it's defined.
    return (override === undefined ? base : (override as unknown as T));
  }

  const result: Record<string, unknown> = { ...base };

  for (const key of Object.keys(override)) {
    const overrideValue = (override as Record<string, unknown>)[key];
    const baseValue     = (base as Record<string, unknown>)[key];

    if (overrideValue === undefined) {
      continue;                                   // explicit `undefined` never erases
    }
    if (isPlainObject(baseValue) && isPlainObject(overrideValue)) {
      result[key] = deepMerge(baseValue, overrideValue as DeepPartial<typeof baseValue>);
    } else {
      result[key] = overrideValue;                // arrays and scalars: replace
    }
  }

  return result as T;
}

// ── Usage: base + environment override + local env-var override ────────────
const merged = deepMerge(baseConfig, productionOverrides);
// merged.server.host is still "0.0.0.0" ✅   merged.server.port is 8080 ✅
```

Note the return type: `deepMerge<T>(base: T, override: DeepPartial<T>): T`. This is the signature that makes the whole scheme work. The output is a *complete* `AppConfig`, because the base was complete and the override could only replace existing leaves. The compiler cannot verify the implementation's correctness (hence the internal casts), but it perfectly constrains the *boundary* — which is where all the call sites are.

You can fold a chain of overrides with the same function:

```ts
export function mergeAll<T>(base: T, ...overrides: readonly DeepPartial<T>[]): T {
  return overrides.reduce<T>((accumulated, layer) => deepMerge(accumulated, layer), base);
}

const config = mergeAll(
  baseConfig,                              // committed defaults
  CONFIG_BY_ENVIRONMENT[environment],      // per-environment differences
  overridesFromEnvVars(process.env),       // per-deployment / per-pod overrides
);
// Precedence, left to right, is now explicit and greppable.
```

### Concept 8 — Validating config at boot

Types describe what you *believe*. `process.env` supplies `string | undefined`. Everything between the two is an assumption that must be checked exactly once, at boot, with a loud failure.

Start with a dedicated error that collects *all* problems, not just the first:

```ts
export class ConfigError extends Error {
  constructor(readonly issues: readonly string[]) {
    super(`Invalid configuration:\n${issues.map((i) => `  • ${i}`).join("\n")}`);
    this.name = "ConfigError";
  }
}
```

Reporting one issue at a time makes deploying a fresh environment a game of whack-a-mole: fix `DATABASE_URL`, redeploy, discover `JWT_SECRET`, redeploy, discover `REDIS_URL`… Collect everything and print it once.

Then a small set of typed readers. Each one turns `string | undefined` into a *narrow* type or records an issue:

```ts
class EnvReader {
  private readonly issues: string[] = [];

  constructor(private readonly env: NodeJS.ProcessEnv) {}

  /** Required string. Records an issue if missing or blank. */
  requiredString(name: string): string {
    const raw = this.env[name]?.trim();
    if (!raw) {
      this.issues.push(`${name} is required but was not set`);
      return "";                                  // placeholder; we throw before it's used
    }
    return raw;
  }

  optionalString(name: string, fallback: string): string {
    const raw = this.env[name]?.trim();
    return raw ? raw : fallback;
  }

  /** Integer with optional bounds. */
  integer(name: string, fallback: number, min = -Infinity, max = Infinity): number {
    const raw = this.env[name]?.trim();
    if (!raw) return fallback;

    const parsed = Number(raw);                   // Number("") === 0, so the blank check above matters
    if (!Number.isInteger(parsed)) {
      this.issues.push(`${name}="${raw}" is not an integer`);
      return fallback;
    }
    if (parsed < min || parsed > max) {
      this.issues.push(`${name}=${parsed} is out of range (expected ${min}–${max})`);
      return fallback;
    }
    return parsed;
  }

  boolean(name: string, fallback: boolean): boolean {
    const raw = this.env[name]?.trim().toLowerCase();
    if (!raw) return fallback;
    if (["1", "true", "yes", "on"].includes(raw))  return true;
    if (["0", "false", "no", "off"].includes(raw)) return false;
    this.issues.push(`${name}="${raw}" is not a boolean (use true/false)`);
    return fallback;
  }

  /**
   * A value from a closed union. The return type is the union itself, so the
   * caller gets a narrow type without any cast at the call site.
   */
  oneOf<const T extends readonly string[]>(
    name: string,
    allowed: T,
    fallback: T[number],
  ): T[number] {
    const raw = this.env[name]?.trim();
    if (!raw) return fallback;
    if (!(allowed as readonly string[]).includes(raw)) {
      this.issues.push(`${name}="${raw}" is invalid (expected one of: ${allowed.join(", ")})`);
      return fallback;
    }
    return raw as T[number];
  }

  /** A URL with an allowed protocol set — catches "localhost:5432" (no scheme). */
  url(name: string, allowedProtocols: readonly string[]): string {
    const raw = this.requiredString(name);
    if (!raw) return raw;
    try {
      const parsed = new URL(raw);
      if (!allowedProtocols.includes(parsed.protocol.replace(":", ""))) {
        this.issues.push(
          `${name} has protocol "${parsed.protocol}" (expected one of: ${allowedProtocols.join(", ")})`,
        );
      }
    } catch {
      this.issues.push(`${name}="${raw}" is not a valid URL`);
    }
    return raw;
  }

  /** Adds a cross-field rule, e.g. "jwtSecret must be ≥32 chars in production". */
  assert(condition: boolean, message: string): void {
    if (!condition) this.issues.push(message);
  }

  /** Throws once, with every issue, or returns cleanly. */
  throwIfInvalid(): void {
    if (this.issues.length > 0) throw new ConfigError(this.issues);
  }
}
```

Note the `<const T extends readonly string[]>` on `oneOf`. The `const` type parameter modifier (TS 5.0+) makes the *call site* infer literals without needing `as const`:

```ts
const logLevel = reader.oneOf("LOG_LEVEL", ["trace", "debug", "info", "warn", "error"], "info");
//    ^ "trace" | "debug" | "info" | "warn" | "error"  — narrow, no `as const` needed
```

Now the loader ties it together, and — critically — validates **cross-field invariants** that no per-variable check can catch:

```ts
export function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  const reader = new EnvReader(env);

  const environment = reader.oneOf("NODE_ENV", ENVIRONMENTS, "development");
  const isProduction = environment === "production";

  const jwtSecret = isProduction
    ? reader.requiredString("JWT_SECRET")
    : reader.optionalString("JWT_SECRET", "insecure-development-secret");

  const draft: AppConfig = {
    environment,
    server: {
      port:             reader.integer("PORT", 3000, 1, 65535),
      host:             reader.optionalString("HOST", "0.0.0.0"),
      requestTimeoutMs: reader.integer("REQUEST_TIMEOUT_MS", 30_000, 1_000, 300_000),
    },
    database: {
      url:           reader.url("DATABASE_URL", ["postgres", "postgresql"]),
      poolSize:      reader.integer("DB_POOL_SIZE", 10, 1, 200),
      idleTimeoutMs: reader.integer("DB_IDLE_TIMEOUT_MS", 30_000, 1_000, 600_000),
      sslMode:       reader.oneOf("DB_SSL_MODE", ["disable", "require", "verify-full"], "require"),
    },
    auth: {
      jwtSecret,
      accessTokenTtlSeconds:  reader.integer("ACCESS_TOKEN_TTL_S", 900, 60, 86_400),
      refreshTokenTtlSeconds: reader.integer("REFRESH_TOKEN_TTL_S", 2_592_000, 3_600, 31_536_000),
    },
    features: loadFeatureFlags(env, DEFAULT_FEATURE_FLAGS),
  };

  // ── Cross-field invariants — the checks that actually prevent outages ────
  reader.assert(
    draft.auth.refreshTokenTtlSeconds > draft.auth.accessTokenTtlSeconds,
    "REFRESH_TOKEN_TTL_S must be greater than ACCESS_TOKEN_TTL_S",
  );
  reader.assert(
    !isProduction || draft.auth.jwtSecret.length >= 32,
    "JWT_SECRET must be at least 32 characters in production",
  );
  reader.assert(
    !isProduction || draft.database.sslMode !== "disable",
    "DB_SSL_MODE must not be 'disable' in production",
  );
  reader.assert(
    !isProduction || draft.database.poolSize >= 5,
    "DB_POOL_SIZE must be at least 5 in production",
  );

  reader.throwIfInvalid();     // ← throws ONCE, listing every problem
  return draft;
}
```

And the boot sequence prints the failure in a form a human can act on:

```ts
// src/index.ts
import { config } from "./config";     // ← may throw ConfigError at import time

async function main(): Promise<void> {
  logger.info("Starting service", {
    environment: config.environment,
    port:        config.server.port,
    features:    config.features,      // safe: booleans only, no secrets
  });
  // …
}

// Top-level: turn a ConfigError into a clean, actionable exit.
main().catch((error: unknown) => {
  if (error instanceof ConfigError) {
    // eslint-disable-next-line no-console -- logger may itself depend on config
    console.error(error.message);
    process.exit(78);                  // EX_CONFIG, from sysexits.h
  }
  throw error;
});
```

If you already use Zod (see `56 — Runtime validation with Zod`-style patterns), you can replace `EnvReader` with a schema and get `z.infer` to *generate* `AppConfig` for you. The structural point is identical: parse once, at the edge, into a typed value.

---

## Example 1 — basic

```ts
// ══════════════════════════════════════════════════════════════════════════
//  A minimal but complete typed config module for a small Node service.
// ══════════════════════════════════════════════════════════════════════════

// ── src/config/types.ts ────────────────────────────────────────────────────

export const ENVIRONMENTS = ["development", "test", "staging", "production"] as const;
export type Environment = typeof ENVIRONMENTS[number];

export const LOG_LEVELS = ["trace", "debug", "info", "warn", "error"] as const;
export type LogLevel = typeof LOG_LEVELS[number];

export interface AppConfig {
  readonly environment: Environment;
  readonly server: {
    readonly port: number;
    readonly host: string;
  };
  readonly database: {
    readonly url: string;
    readonly poolSize: number;
  };
  readonly logging: {
    readonly level: LogLevel;
    readonly pretty: boolean;
  };
}

// ── src/config/errors.ts ───────────────────────────────────────────────────

export class ConfigError extends Error {
  constructor(readonly issues: readonly string[]) {
    super(`Invalid configuration:\n${issues.map((issue) => `  • ${issue}`).join("\n")}`);
    this.name = "ConfigError";
  }
}

// ── src/config/load.ts ─────────────────────────────────────────────────────

import { ENVIRONMENTS, LOG_LEVELS, type AppConfig, type Environment, type LogLevel } from "./types";
import { ConfigError } from "./errors";

/** Accumulates every problem so boot reports them all at once. */
const issues: string[] = [];

function requiredString(env: NodeJS.ProcessEnv, name: string): string {
  const raw = env[name]?.trim();
  if (!raw) {
    issues.push(`${name} is required but was not set`);
    return "";
  }
  return raw;
}

function integer(env: NodeJS.ProcessEnv, name: string, fallback: number, min: number, max: number): number {
  const raw = env[name]?.trim();
  if (!raw) return fallback;
  const parsed = Number(raw);
  if (!Number.isInteger(parsed)) {
    issues.push(`${name}="${raw}" is not an integer`);
    return fallback;
  }
  if (parsed < min || parsed > max) {
    issues.push(`${name}=${parsed} is out of range (${min}–${max})`);
    return fallback;
  }
  return parsed;
}

function boolean(env: NodeJS.ProcessEnv, name: string, fallback: boolean): boolean {
  const raw = env[name]?.trim().toLowerCase();
  if (!raw) return fallback;
  if (["1", "true", "yes", "on"].includes(raw)) return true;
  if (["0", "false", "no", "off"].includes(raw)) return false;
  issues.push(`${name}="${raw}" is not a boolean`);
  return fallback;
}

/** Narrows a raw string to a member of a closed union, or records an issue. */
function oneOf<const T extends readonly string[]>(
  env: NodeJS.ProcessEnv,
  name: string,
  allowed: T,
  fallback: T[number],
): T[number] {
  const raw = env[name]?.trim();
  if (!raw) return fallback;
  if (!(allowed as readonly string[]).includes(raw)) {
    issues.push(`${name}="${raw}" is invalid (expected: ${allowed.join(", ")})`);
    return fallback;
  }
  return raw as T[number];
}

export function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  issues.length = 0;   // reset — this function is called in tests too

  const environment: Environment = oneOf(env, "NODE_ENV", ENVIRONMENTS, "development");
  const level: LogLevel = oneOf(env, "LOG_LEVEL", LOG_LEVELS, environment === "production" ? "info" : "debug");

  const loaded: AppConfig = {
    environment,
    server: {
      port: integer(env, "PORT", 3000, 1, 65535),
      host: env.HOST?.trim() || "0.0.0.0",
    },
    database: {
      url:      requiredString(env, "DATABASE_URL"),
      poolSize: integer(env, "DB_POOL_SIZE", environment === "production" ? 20 : 5, 1, 200),
    },
    logging: {
      level,
      pretty: boolean(env, "LOG_PRETTY", environment !== "production"),
    },
  };

  if (issues.length > 0) throw new ConfigError(issues);
  return loaded;
}

// ── src/config/index.ts — the ONE module everything else imports ───────────

import { loadConfig } from "./load";
import type { AppConfig } from "./types";

/** Built once, at import time. Frozen so nothing can mutate it later. */
export const config: AppConfig = Object.freeze(loadConfig(process.env));

export type { AppConfig, Environment, LogLevel } from "./types";
export { ConfigError } from "./errors";

// ── Usage anywhere else — no process.env, no parsing, no `!` ───────────────

import { config } from "./config";

const pool = new Pool({
  connectionString: config.database.url,     // string ✅
  max:              config.database.poolSize, // number ✅
});

app.listen(config.server.port, config.server.host, () => {
  logger.info(`Listening on ${config.server.host}:${config.server.port}`);
});

// config.server.port = 4000;                // ❌ Cannot assign to 'port' — read-only property
// config.databse.url;                       // ❌ Property 'databse' does not exist on 'AppConfig'
// config.logging.level = "verbose";         // ❌ '"verbose"' is not assignable to LogLevel

// ── A test — no module mocking needed, because loadConfig is pure ──────────

import { loadConfig, ConfigError } from "./config/load";

test("rejects a port outside the valid range", () => {
  expect(() =>
    loadConfig({ NODE_ENV: "test", DATABASE_URL: "postgres://localhost/acme", PORT: "99999" }),
  ).toThrow(ConfigError);
});

test("defaults pool size higher in production", () => {
  const produced = loadConfig({ NODE_ENV: "production", DATABASE_URL: "postgres://db/acme" });
  expect(produced.database.poolSize).toBe(20);
});
```

---

## Example 2 — real world backend use case

```ts
// ══════════════════════════════════════════════════════════════════════════
//  A full production config module:
//    base defaults → environment overrides → env-var overrides → validate →
//    deep-freeze → export one typed object.
//  Includes feature flags, secret redaction, and a boot-time summary log.
// ══════════════════════════════════════════════════════════════════════════

// ── src/config/types.ts ────────────────────────────────────────────────────

export const ENVIRONMENTS = ["development", "test", "staging", "production"] as const;
export type Environment = typeof ENVIRONMENTS[number];

export const LOG_LEVELS = ["trace", "debug", "info", "warn", "error"] as const;
export type LogLevel = typeof LOG_LEVELS[number];

export const SSL_MODES = ["disable", "require", "verify-full"] as const;
export type SslMode = typeof SSL_MODES[number];

export const FEATURE_FLAG_NAMES = [
  "newCheckout",
  "betaSearch",
  "auditLogging",
  "asyncInvoiceExport",
  "strictRateLimiting",
] as const;
export type FeatureFlagName = typeof FEATURE_FLAG_NAMES[number];
export type FeatureFlags = Readonly<Record<FeatureFlagName, boolean>>;

export interface ServerConfig {
  readonly port: number;
  readonly host: string;
  readonly requestTimeoutMs: number;
  readonly shutdownGraceMs: number;
  readonly trustProxy: boolean;
  readonly corsOrigins: readonly string[];
}

export interface DatabaseConfig {
  readonly url: string;                  // 🔒 contains a password — never log raw
  readonly poolSize: number;
  readonly idleTimeoutMs: number;
  readonly statementTimeoutMs: number;
  readonly sslMode: SslMode;
}

export interface RedisConfig {
  readonly url: string;                  // 🔒
  readonly keyPrefix: string;
  readonly commandTimeoutMs: number;
}

export interface AuthConfig {
  readonly jwtSecret: string;            // 🔒
  readonly jwtIssuer: string;
  readonly accessTokenTtlSeconds: number;
  readonly refreshTokenTtlSeconds: number;
  readonly bcryptRounds: number;
}

export interface RateLimitConfig {
  readonly windowMs: number;
  readonly maxRequestsPerWindow: number;
  readonly maxRequestsPerWindowAuthenticated: number;
}

export interface ObservabilityConfig {
  readonly logLevel: LogLevel;
  readonly prettyLogs: boolean;
  readonly otelEndpoint: string | null;   // null = tracing disabled
  readonly sampleRatio: number;           // 0.0 – 1.0
}

export interface AppConfig {
  readonly environment: Environment;
  readonly serviceName: string;
  readonly server:        ServerConfig;
  readonly database:      DatabaseConfig;
  readonly redis:         RedisConfig;
  readonly auth:          AuthConfig;
  readonly rateLimit:     RateLimitConfig;
  readonly observability: ObservabilityConfig;
  readonly features:      FeatureFlags;
}

// ── src/config/deep.ts — the two mapped types the whole scheme rests on ────

type Immutable =
  | string | number | boolean | bigint | symbol | null | undefined
  | Date | RegExp | ((...args: never[]) => unknown);

/** Every property optional, recursively. Arrays are leaves (replaced, not merged). */
export type DeepPartial<T> =
  T extends Immutable ? T
  : T extends readonly (infer Element)[] ? readonly Element[]
  : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

/** Every property readonly, recursively. */
export type DeepReadonly<T> =
  T extends Immutable ? T
  : T extends ReadonlyMap<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends ReadonlySet<infer E> ? ReadonlySet<DeepReadonly<E>>
  : T extends readonly (infer Element)[] ? readonly DeepReadonly<Element>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

function isPlainObject(value: unknown): value is Record<string, unknown> {
  return (
    typeof value === "object" && value !== null &&
    !Array.isArray(value) &&
    !(value instanceof Date) && !(value instanceof RegExp) &&
    !(value instanceof Map)  && !(value instanceof Set)
  );
}

/**
 * Merge `override` onto `base`, recursively.
 *  - plain objects merge; arrays and scalars replace
 *  - `undefined` in the override is ignored (does not erase a base value)
 *  - neither input is mutated
 */
export function deepMerge<T>(base: T, override: DeepPartial<T>): T {
  if (!isPlainObject(base) || !isPlainObject(override)) {
    return override === undefined ? base : (override as unknown as T);
  }
  const result: Record<string, unknown> = { ...base };
  for (const key of Object.keys(override)) {
    const overrideValue = (override as Record<string, unknown>)[key];
    const baseValue     = (base as Record<string, unknown>)[key];
    if (overrideValue === undefined) continue;
    result[key] =
      isPlainObject(baseValue) && isPlainObject(overrideValue)
        ? deepMerge(baseValue, overrideValue as DeepPartial<typeof baseValue>)
        : overrideValue;
  }
  return result as T;
}

/** Left-to-right precedence: later layers win. */
export function mergeAll<T>(base: T, ...layers: readonly DeepPartial<T>[]): T {
  return layers.reduce<T>((accumulated, layer) => deepMerge(accumulated, layer), base);
}

/** Runtime immutability — types are erased, this is not. */
export function deepFreeze<T>(value: T): DeepReadonly<T> {
  if (value !== null && typeof value === "object" && !Object.isFrozen(value)) {
    Object.freeze(value);
    for (const key of Object.getOwnPropertyNames(value)) {
      deepFreeze((value as Record<string, unknown>)[key]);
    }
  }
  return value as DeepReadonly<T>;
}

// ── src/config/base.ts — committed defaults, safe for local development ────

import type { AppConfig } from "./types";
import { FEATURE_FLAG_NAMES, type FeatureFlags } from "./types";

export const DEFAULT_FEATURE_FLAGS = {
  newCheckout:        false,   // opt-in: new code path
  betaSearch:         false,   // opt-in
  auditLogging:       true,    // safety: opt-OUT
  asyncInvoiceExport: false,   // opt-in
  strictRateLimiting: true,    // safety: opt-OUT
} as const satisfies FeatureFlags;
// Add a name to FEATURE_FLAG_NAMES and this object fails to compile until you
// give the new flag a default. That is exactly the review prompt you want.

/**
 * The base layer. Never contains a real secret — the placeholders below are
 * rejected by validation in staging/production.
 */
export const BASE_CONFIG = {
  environment: "development",
  serviceName: "acme-api",
  server: {
    port: 3000,
    host: "0.0.0.0",
    requestTimeoutMs: 30_000,
    shutdownGraceMs:  10_000,
    trustProxy: false,
    corsOrigins: ["http://localhost:5173"],
  },
  database: {
    url: "postgres://acme:acme@localhost:5432/acme_dev",
    poolSize: 5,
    idleTimeoutMs: 30_000,
    statementTimeoutMs: 15_000,
    sslMode: "disable",
  },
  redis: {
    url: "redis://localhost:6379",
    keyPrefix: "acme:dev:",
    commandTimeoutMs: 2_000,
  },
  auth: {
    jwtSecret: "insecure-development-secret-do-not-use",
    jwtIssuer: "https://api.acme.local",
    accessTokenTtlSeconds:  900,          // 15 minutes
    refreshTokenTtlSeconds: 2_592_000,    // 30 days
    bcryptRounds: 10,
  },
  rateLimit: {
    windowMs: 60_000,
    maxRequestsPerWindow: 1_000,
    maxRequestsPerWindowAuthenticated: 5_000,
  },
  observability: {
    logLevel: "debug",
    prettyLogs: true,
    otelEndpoint: null,
    sampleRatio: 1.0,
  },
  features: DEFAULT_FEATURE_FLAGS,
} as const satisfies AppConfig;
// ↑ `as const satisfies` = fully checked against AppConfig AND narrowly typed,
//   so `typeof BASE_CONFIG["server"]["port"]` is literally 3000.

// ── src/config/environments.ts — per-environment DIFFERENCES only ──────────

import type { AppConfig, Environment } from "./types";
import type { DeepPartial } from "./deep";

/**
 * Every key is required (it's a Record over the closed Environment union), but
 * every VALUE is a DeepPartial — so each entry states only what differs.
 * Adding an environment to the union breaks the build right here.
 */
export const OVERRIDES_BY_ENVIRONMENT: Record<Environment, DeepPartial<AppConfig>> = {
  development: {
    environment: "development",
    // everything else inherited from BASE_CONFIG
  },

  test: {
    environment: "test",
    server:   { port: 0 },                       // 0 = OS-assigned free port, for parallel tests
    database: { url: "postgres://acme:acme@localhost:5432/acme_test", poolSize: 2 },
    redis:    { keyPrefix: "acme:test:" },
    auth:     { bcryptRounds: 4 },               // fast hashing so tests aren't glacial
    observability: { logLevel: "error", prettyLogs: false, sampleRatio: 0 },
    features: { auditLogging: false },           // less noise in test assertions
  },

  staging: {
    environment: "staging",
    server: {
      port: 8080,
      trustProxy: true,
      corsOrigins: ["https://staging.acme.com"],  // arrays REPLACE, never merge
    },
    database: { poolSize: 10, sslMode: "require" },
    redis:    { keyPrefix: "acme:staging:" },
    auth:     { bcryptRounds: 12 },
    rateLimit: { maxRequestsPerWindow: 300 },
    observability: {
      logLevel: "info",
      prettyLogs: false,
      otelEndpoint: "http://otel-collector.observability:4318",
      sampleRatio: 0.5,
    },
    features: { newCheckout: true, betaSearch: true },   // staging tries things first
  },

  production: {
    environment: "production",
    server: {
      port: 8080,
      trustProxy: true,
      shutdownGraceMs: 30_000,                    // give k8s time to drain connections
      corsOrigins: ["https://app.acme.com", "https://admin.acme.com"],
    },
    database: {
      poolSize: 40,
      sslMode: "verify-full",
      statementTimeoutMs: 10_000,
    },
    redis:  { keyPrefix: "acme:prod:", commandTimeoutMs: 1_000 },
    auth:   { jwtIssuer: "https://api.acme.com", bcryptRounds: 12 },
    rateLimit: { maxRequestsPerWindow: 120, maxRequestsPerWindowAuthenticated: 1_200 },
    observability: {
      logLevel: "info",
      prettyLogs: false,
      otelEndpoint: "http://otel-collector.observability:4318",
      sampleRatio: 0.05,                          // 5% sampling at production volume
    },
    features: { auditLogging: true, strictRateLimiting: true },
  },
};

// ── src/config/fromEnv.ts — the deployment layer (highest precedence) ──────

import type { AppConfig, Environment, FeatureFlagName } from "./types";
import { ENVIRONMENTS, LOG_LEVELS, SSL_MODES, FEATURE_FLAG_NAMES } from "./types";
import type { DeepPartial } from "./deep";
import { ConfigError } from "./errors";

const issues: string[] = [];

function str(env: NodeJS.ProcessEnv, name: string): string | undefined {
  const raw = env[name]?.trim();
  return raw ? raw : undefined;
}

function int(env: NodeJS.ProcessEnv, name: string, min: number, max: number): number | undefined {
  const raw = str(env, name);
  if (raw === undefined) return undefined;
  const parsed = Number(raw);
  if (!Number.isInteger(parsed)) { issues.push(`${name}="${raw}" is not an integer`); return undefined; }
  if (parsed < min || parsed > max) { issues.push(`${name}=${parsed} out of range (${min}–${max})`); return undefined; }
  return parsed;
}

function num(env: NodeJS.ProcessEnv, name: string, min: number, max: number): number | undefined {
  const raw = str(env, name);
  if (raw === undefined) return undefined;
  const parsed = Number(raw);
  if (!Number.isFinite(parsed)) { issues.push(`${name}="${raw}" is not a number`); return undefined; }
  if (parsed < min || parsed > max) { issues.push(`${name}=${parsed} out of range (${min}–${max})`); return undefined; }
  return parsed;
}

function bool(env: NodeJS.ProcessEnv, name: string): boolean | undefined {
  const raw = str(env, name)?.toLowerCase();
  if (raw === undefined) return undefined;
  if (["1", "true", "yes", "on"].includes(raw))  return true;
  if (["0", "false", "no", "off"].includes(raw)) return false;
  issues.push(`${name}="${raw}" is not a boolean (use true/false)`);
  return undefined;
}

function pick<const T extends readonly string[]>(
  env: NodeJS.ProcessEnv, name: string, allowed: T,
): T[number] | undefined {
  const raw = str(env, name);
  if (raw === undefined) return undefined;
  if (!(allowed as readonly string[]).includes(raw)) {
    issues.push(`${name}="${raw}" is invalid (expected: ${allowed.join(", ")})`);
    return undefined;
  }
  return raw as T[number];
}

function csv(env: NodeJS.ProcessEnv, name: string): readonly string[] | undefined {
  const raw = str(env, name);
  return raw?.split(",").map((part) => part.trim()).filter(Boolean);
}

/** "newCheckout" → "FEATURE_NEW_CHECKOUT" */
function envVarForFlag(name: FeatureFlagName): string {
  return `FEATURE_${name.replace(/([A-Z])/g, "_$1").toUpperCase()}`;
}

function flagOverrides(env: NodeJS.ProcessEnv): Partial<Record<FeatureFlagName, boolean>> {
  const overrides: Partial<Record<FeatureFlagName, boolean>> = {};
  for (const name of FEATURE_FLAG_NAMES) {
    const parsed = bool(env, envVarForFlag(name));
    if (parsed !== undefined) overrides[name] = parsed;
  }
  return overrides;
}

/**
 * Build a DeepPartial<AppConfig> from environment variables.
 * Everything is `undefined` when unset — and `deepMerge` ignores `undefined`,
 * so an unset variable inherits from the layer below instead of erasing it.
 */
export function overridesFromEnv(env: NodeJS.ProcessEnv): DeepPartial<AppConfig> {
  return {
    serviceName: str(env, "SERVICE_NAME"),
    server: {
      port:             int(env, "PORT", 0, 65535),
      host:             str(env, "HOST"),
      requestTimeoutMs: int(env, "REQUEST_TIMEOUT_MS", 1_000, 300_000),
      shutdownGraceMs:  int(env, "SHUTDOWN_GRACE_MS", 0, 120_000),
      trustProxy:       bool(env, "TRUST_PROXY"),
      corsOrigins:      csv(env, "CORS_ORIGINS"),
    },
    database: {
      url:                str(env, "DATABASE_URL"),
      poolSize:           int(env, "DB_POOL_SIZE", 1, 200),
      idleTimeoutMs:      int(env, "DB_IDLE_TIMEOUT_MS", 1_000, 600_000),
      statementTimeoutMs: int(env, "DB_STATEMENT_TIMEOUT_MS", 100, 120_000),
      sslMode:            pick(env, "DB_SSL_MODE", SSL_MODES),
    },
    redis: {
      url:              str(env, "REDIS_URL"),
      keyPrefix:        str(env, "REDIS_KEY_PREFIX"),
      commandTimeoutMs: int(env, "REDIS_COMMAND_TIMEOUT_MS", 100, 30_000),
    },
    auth: {
      jwtSecret:              str(env, "JWT_SECRET"),
      jwtIssuer:              str(env, "JWT_ISSUER"),
      accessTokenTtlSeconds:  int(env, "ACCESS_TOKEN_TTL_S", 60, 86_400),
      refreshTokenTtlSeconds: int(env, "REFRESH_TOKEN_TTL_S", 3_600, 31_536_000),
      bcryptRounds:           int(env, "BCRYPT_ROUNDS", 4, 15),
    },
    rateLimit: {
      windowMs:                          int(env, "RATE_LIMIT_WINDOW_MS", 1_000, 3_600_000),
      maxRequestsPerWindow:              int(env, "RATE_LIMIT_MAX", 1, 1_000_000),
      maxRequestsPerWindowAuthenticated: int(env, "RATE_LIMIT_MAX_AUTHENTICATED", 1, 1_000_000),
    },
    observability: {
      logLevel:     pick(env, "LOG_LEVEL", LOG_LEVELS),
      prettyLogs:   bool(env, "LOG_PRETTY"),
      otelEndpoint: str(env, "OTEL_ENDPOINT"),
      sampleRatio:  num(env, "OTEL_SAMPLE_RATIO", 0, 1),
    },
    features: flagOverrides(env),
  };
}

export function takeEnvParseIssues(): readonly string[] {
  const taken = [...issues];
  issues.length = 0;
  return taken;
}

// ── src/config/validate.ts — cross-field invariants ───────────────────────

import type { AppConfig } from "./types";

const PLACEHOLDER_SECRETS = new Set([
  "insecure-development-secret-do-not-use",
  "changeme",
  "secret",
]);

/** Returns a list of problems. Empty list = valid. */
export function validateConfig(candidate: AppConfig): readonly string[] {
  const problems: string[] = [];
  const isDeployed = candidate.environment === "staging" || candidate.environment === "production";
  const isProduction = candidate.environment === "production";

  // ── Always true, in every environment ───────────────────────────────────
  if (candidate.auth.refreshTokenTtlSeconds <= candidate.auth.accessTokenTtlSeconds) {
    problems.push("auth.refreshTokenTtlSeconds must exceed auth.accessTokenTtlSeconds");
  }
  if (candidate.rateLimit.maxRequestsPerWindowAuthenticated < candidate.rateLimit.maxRequestsPerWindow) {
    problems.push("rateLimit: authenticated limit must be ≥ anonymous limit");
  }
  if (candidate.database.statementTimeoutMs >= candidate.server.requestTimeoutMs) {
    problems.push("database.statementTimeoutMs must be less than server.requestTimeoutMs");
  }

  // ── URL shapes ──────────────────────────────────────────────────────────
  for (const [label, raw, protocols] of [
    ["database.url", candidate.database.url, ["postgres:", "postgresql:"]],
    ["redis.url",    candidate.redis.url,    ["redis:", "rediss:"]],
  ] as const) {
    try {
      const parsed = new URL(raw);
      if (!protocols.includes(parsed.protocol as never)) {
        problems.push(`${label} has protocol "${parsed.protocol}" (expected ${protocols.join(" or ")})`);
      }
    } catch {
      problems.push(`${label}="${raw}" is not a valid URL`);
    }
  }

  // ── Deployed environments only ─────────────────────────────────────────
  if (isDeployed) {
    if (PLACEHOLDER_SECRETS.has(candidate.auth.jwtSecret)) {
      problems.push("auth.jwtSecret is still the development placeholder");
    }
    if (candidate.auth.jwtSecret.length < 32) {
      problems.push("auth.jwtSecret must be at least 32 characters outside development");
    }
    if (candidate.auth.bcryptRounds < 12) {
      problems.push("auth.bcryptRounds must be at least 12 outside development");
    }
    if (!candidate.server.trustProxy) {
      problems.push("server.trustProxy must be true behind the ingress load balancer");
    }
    if (candidate.observability.prettyLogs) {
      problems.push("observability.prettyLogs must be false outside development (breaks log ingestion)");
    }
    if (candidate.server.corsOrigins.some((origin) => origin.startsWith("http://"))) {
      problems.push("server.corsOrigins must all be https outside development");
    }
  }

  // ── Production only ────────────────────────────────────────────────────
  if (isProduction) {
    if (candidate.database.sslMode !== "verify-full") {
      problems.push('database.sslMode must be "verify-full" in production');
    }
    if (candidate.database.poolSize < 10) {
      problems.push("database.poolSize must be at least 10 in production");
    }
    if (candidate.observability.otelEndpoint === null) {
      problems.push("observability.otelEndpoint is required in production");
    }
    if (!candidate.features.auditLogging) {
      problems.push("features.auditLogging cannot be disabled in production");
    }
  }

  return problems;
}

// ── src/config/redact.ts — so you can safely log the config at boot ───────

import type { AppConfig } from "./types";

/** Keys whose values must never appear in a log line. */
const SECRET_PATHS = ["database.url", "redis.url", "auth.jwtSecret"] as const;

function redactUrlCredentials(raw: string): string {
  try {
    const parsed = new URL(raw);
    if (parsed.password) parsed.password = "***";
    if (parsed.username) parsed.username = "***";
    return parsed.toString();
  } catch {
    return "***";
  }
}

/** A log-safe view. Same shape, secrets masked. */
export function redactConfig(source: AppConfig): AppConfig {
  return {
    ...source,
    database: { ...source.database, url: redactUrlCredentials(source.database.url) },
    redis:    { ...source.redis,    url: redactUrlCredentials(source.redis.url) },
    auth:     { ...source.auth,     jwtSecret: `***(${source.auth.jwtSecret.length} chars)` },
  };
}

export { SECRET_PATHS };

// ── src/config/index.ts — assemble, validate, freeze, export ──────────────

import { BASE_CONFIG } from "./base";
import { OVERRIDES_BY_ENVIRONMENT } from "./environments";
import { overridesFromEnv, takeEnvParseIssues } from "./fromEnv";
import { validateConfig } from "./validate";
import { mergeAll, deepFreeze, type DeepReadonly } from "./deep";
import { ENVIRONMENTS, type AppConfig, type Environment } from "./types";
import { ConfigError } from "./errors";

function resolveEnvironment(raw: string | undefined): Environment {
  const candidate = raw?.trim();
  if (candidate === undefined || candidate === "") return "development";
  if (!(ENVIRONMENTS as readonly string[]).includes(candidate)) {
    throw new ConfigError([`NODE_ENV="${candidate}" is not a known environment (expected: ${ENVIRONMENTS.join(", ")})`]);
  }
  return candidate as Environment;
}

/**
 * Pure: takes an env-like object, returns a frozen config or throws.
 * Layer precedence, lowest to highest:
 *   1. BASE_CONFIG                     — committed defaults
 *   2. OVERRIDES_BY_ENVIRONMENT[env]   — committed per-environment differences
 *   3. overridesFromEnv(env)           — per-deployment / per-pod variables
 */
export function buildConfig(env: NodeJS.ProcessEnv): DeepReadonly<AppConfig> {
  const environment = resolveEnvironment(env.NODE_ENV);

  const merged: AppConfig = mergeAll<AppConfig>(
    BASE_CONFIG,
    OVERRIDES_BY_ENVIRONMENT[environment],   // ✅ Record over a closed union — never undefined
    overridesFromEnv(env),
  );

  const problems = [...takeEnvParseIssues(), ...validateConfig(merged)];
  if (problems.length > 0) throw new ConfigError(problems);

  return deepFreeze(merged);
}

/** The one object the rest of the application imports. */
export const config: DeepReadonly<AppConfig> = buildConfig(process.env);

export type { AppConfig, Environment, LogLevel, FeatureFlagName } from "./types";
export { ConfigError } from "./errors";
export { redactConfig } from "./redact";

// ── src/index.ts — boot ───────────────────────────────────────────────────

import { config, ConfigError, redactConfig } from "./config";
import { createLogger } from "./logging";
import { createServer } from "./server";

async function main(): Promise<void> {
  const logger = createLogger({
    level:  config.observability.logLevel,
    pretty: config.observability.prettyLogs,
    name:   config.serviceName,
  });

  // Safe to log: secrets are masked, everything else is genuinely useful in an incident.
  logger.info({ config: redactConfig(config as AppConfig) }, "Configuration resolved");

  const server = createServer(config);
  await server.listen(config.server.port, config.server.host);
  logger.info(`Listening on ${config.server.host}:${config.server.port}`);

  const shutdown = async (signal: string): Promise<void> => {
    logger.info(`${signal} received — draining for ${config.server.shutdownGraceMs}ms`);
    await server.close(config.server.shutdownGraceMs);
    process.exit(0);
  };
  process.on("SIGTERM", () => void shutdown("SIGTERM"));
  process.on("SIGINT",  () => void shutdown("SIGINT"));
}

main().catch((error: unknown) => {
  if (error instanceof ConfigError) {
    // eslint-disable-next-line no-console -- the logger itself depends on config
    console.error(error.message);
    process.exit(78);              // EX_CONFIG
  }
  // eslint-disable-next-line no-console
  console.error(error);
  process.exit(1);
});

// ── Consumers — typed, no parsing, no process.env ─────────────────────────

// src/db/pool.ts
import { Pool } from "pg";
import { config } from "../config";

export const pool = new Pool({
  connectionString:   config.database.url,
  max:                config.database.poolSize,
  idleTimeoutMillis:  config.database.idleTimeoutMs,
  statement_timeout:  config.database.statementTimeoutMs,
  ssl: config.database.sslMode === "disable" ? false : { rejectUnauthorized: config.database.sslMode === "verify-full" },
});

// src/auth/tokens.ts
import jwt from "jsonwebtoken";
import { config } from "../config";

export function issueAccessToken(userId: string): string {
  return jwt.sign({ sub: userId }, config.auth.jwtSecret, {
    issuer:    config.auth.jwtIssuer,
    expiresIn: config.auth.accessTokenTtlSeconds,
  });
}

// src/checkout/route.ts
import { config } from "../config";

export async function handleCheckout(userId: string, requestBody: CheckoutRequest): Promise<void> {
  if (config.features.newCheckout) {
    await newCheckoutPipeline(userId, requestBody);
  } else {
    await legacyCheckoutPipeline(userId, requestBody);
  }
}

// ── Tests — buildConfig is pure, so no module mocking is needed ───────────

import { buildConfig } from "./config";
import { ConfigError } from "./config/errors";

const PRODUCTION_ENV: NodeJS.ProcessEnv = {
  NODE_ENV: "production",
  DATABASE_URL: "postgres://acme:s3cret@db.internal:5432/acme",
  REDIS_URL: "redis://cache.internal:6379",
  JWT_SECRET: "a".repeat(48),
  TRUST_PROXY: "true",
  OTEL_ENDPOINT: "http://otel:4318",
  CORS_ORIGINS: "https://app.acme.com",
};

test("production config merges base + environment + env-var layers", () => {
  const produced = buildConfig(PRODUCTION_ENV);
  expect(produced.database.poolSize).toBe(40);          // from the production layer
  expect(produced.server.host).toBe("0.0.0.0");         // inherited from BASE_CONFIG
  expect(produced.database.sslMode).toBe("verify-full");
  expect(produced.features.auditLogging).toBe(true);
});

test("env vars beat the environment layer", () => {
  const produced = buildConfig({ ...PRODUCTION_ENV, DB_POOL_SIZE: "80" });
  expect(produced.database.poolSize).toBe(80);
});

test("rejects a placeholder secret in production", () => {
  expect(() =>
    buildConfig({ ...PRODUCTION_ENV, JWT_SECRET: "changeme" }),
  ).toThrow(ConfigError);
});

test("reports every problem at once", () => {
  try {
    buildConfig({ NODE_ENV: "production", DATABASE_URL: "not-a-url", JWT_SECRET: "short" });
    throw new Error("expected buildConfig to throw");
  } catch (error) {
    expect(error).toBeInstanceOf(ConfigError);
    expect((error as ConfigError).issues.length).toBeGreaterThan(3);
  }
});

test("the resolved config is deeply frozen", () => {
  const produced = buildConfig(PRODUCTION_ENV);
  expect(() => {
    (produced as unknown as { server: { port: number } }).server.port = 1;
  }).toThrow(TypeError);
});
```

---

## Going deeper

### 1 — `as const` vs `satisfies` vs an annotation: the decision table

These three are constantly confused. The difference is *which direction the type flows*.

```ts
interface ServerConfig { readonly port: number; readonly host: string }

// Annotation — the TYPE flows INTO the value. Value is widened to the type.
const a: ServerConfig = { port: 3000, host: "0.0.0.0" };
type PortA = typeof a["port"];        // number

// `as const` — the VALUE flows OUT. No type is imposed; nothing is checked.
const b = { port: 3000, host: "0.0.0.0" } as const;
type PortB = typeof b["port"];        // 3000 — but nothing verified this is a ServerConfig

// `satisfies` — the value is CHECKED against the type, then the VALUE's own
// (already-narrow-ish) inferred type is kept.
const c = { port: 3000, host: "0.0.0.0" } satisfies ServerConfig;
type PortC = typeof c["port"];        // number — satisfies alone doesn't stop widening

// `as const satisfies` — checked AND maximally narrow.
const d = { port: 3000, host: "0.0.0.0" } as const satisfies ServerConfig;
type PortD = typeof d["port"];        // 3000 ✅
```

Note `PortC` is `number`, not `3000` — a common surprise. `satisfies` does not itself prevent widening for a mutable property; it only preserves whatever the expression would have inferred on its own. You still need `as const` when you want literal precision.

The practical rule for config:

| You are writing | Use |
|---|---|
| a config literal you want checked and narrow | `as const satisfies AppConfig` |
| a list of names you'll derive a union from | `as const` |
| a value produced by a function at runtime | a return-type annotation |
| an override object | `: DeepPartial<AppConfig>` annotation (you *want* widening here) |

That last row matters: annotating overrides with `DeepPartial<AppConfig>` (rather than `satisfies`) is deliberate, because the merged output is what you care about, not the override's own literal types.

### 2 — Why `Object.freeze` and `readonly` are both necessary

They protect against different attackers.

```ts
const config: DeepReadonly<AppConfig> = deepFreeze(loadConfig(process.env));

// `readonly` catches the honest mistake at compile time:
// config.server.port = 4000;              // ❌ Cannot assign to 'port'

// `Object.freeze` catches the dishonest one at runtime:
(config as any).server.port = 4000;        // TypeError in strict mode (ES modules are strict)

// And it catches JS callers entirely — a plain-JS dependency, a REPL, a
// `require()`d legacy module, or a test helper that does:
Object.assign(config.features, { newCheckout: true });   // TypeError ✅
```

The freeze also documents intent at runtime. If someone *needs* mutable config — a hot-reload story, a feature-flag service that polls — the `TypeError` forces them to design it properly (see item 5 below) instead of poking the shared object.

Cost: `deepFreeze` walks the object once at boot. For a config with a few dozen leaves this is microseconds. Do not extend it to per-request data.

### 3 — `DeepPartial` has sharp edges you should know about

**Arrays.** A naive `DeepPartial` maps arrays element-wise:

```ts
type Naive<T> = { [K in keyof T]?: T[K] extends object ? Naive<T[K]> : T[K] };
type Bad = Naive<{ corsOrigins: string[] }>;
// { corsOrigins?: { [n: number]?: string; length?: number; push?: … } }  🤮
// Arrays are objects, so the mapped type mangles them into a partial of Array's own shape.
```

That is why the version in this doc special-cases arrays *before* the `object` branch and treats them as replaceable leaves.

**Optional properties that should stay absent.** `DeepPartial<T>` makes `undefined` assignable everywhere, which collides with `exactOptionalPropertyTypes`:

```ts
// With "exactOptionalPropertyTypes": true in tsconfig:
interface Config { otelEndpoint?: string }
const override: DeepPartial<Config> = { otelEndpoint: undefined };
// ❌ Type 'undefined' is not assignable to type 'string'
//    (the property may be ABSENT, but may not be present-and-undefined)
```

Fix by making "disabled" an explicit value in the type rather than an absence — `otelEndpoint: string | null` — which is what the real-world example above does. `null` means "explicitly off"; absent means "inherit". Two distinct concepts, two distinct encodings.

**Functions.** If your config carries callbacks (`onRetry`, a custom serializer), a naive `DeepPartial` will recurse into the function's properties. The `Immutable` escape list in the example prevents this.

### 4 — `deepMerge` is where types lie, so constrain the boundary

The implementation of `deepMerge` is full of `as` casts — unavoidable, because TypeScript cannot express "recursively merging two object trees yields the first tree's type". You have two honest options:

**(a) Accept the casts, but make the signature airtight.** `deepMerge<T>(base: T, override: DeepPartial<T>): T` is a *correct* contract even though the body is unverified. Test the body directly:

```ts
test("nested objects merge instead of replacing", () => {
  const merged = deepMerge(
    { server: { port: 3000, host: "0.0.0.0" } },
    { server: { port: 8080 } },
  );
  expect(merged).toEqual({ server: { port: 8080, host: "0.0.0.0" } });
});

test("arrays replace wholesale", () => {
  const merged = deepMerge({ corsOrigins: ["a", "b", "c"] }, { corsOrigins: ["z"] });
  expect(merged.corsOrigins).toEqual(["z"]);
});

test("undefined never erases a base value", () => {
  const merged = deepMerge({ server: { port: 3000 } }, { server: { port: undefined } });
  expect(merged.server.port).toBe(3000);
});

test("neither input is mutated", () => {
  const base = { server: { port: 3000 } };
  deepMerge(base, { server: { port: 8080 } });
  expect(base.server.port).toBe(3000);
});
```

**(b) Skip the generic merge entirely and write an explicit builder.** For a config with ~8 sections, hand-writing the merge is verbose but has *zero* unsound casts:

```ts
function mergeConfig(base: AppConfig, override: DeepPartial<AppConfig>): AppConfig {
  return {
    environment: override.environment ?? base.environment,
    serviceName: override.serviceName ?? base.serviceName,
    server: {
      port:             override.server?.port             ?? base.server.port,
      host:             override.server?.host             ?? base.server.host,
      requestTimeoutMs: override.server?.requestTimeoutMs  ?? base.server.requestTimeoutMs,
      shutdownGraceMs:  override.server?.shutdownGraceMs   ?? base.server.shutdownGraceMs,
      trustProxy:       override.server?.trustProxy        ?? base.server.trustProxy,
      corsOrigins:      override.server?.corsOrigins       ?? base.server.corsOrigins,
    },
    // …and so on. Fully checked; a new AppConfig field is a compile error here.
  };
}
```

The trade-off is honest: (a) is DRY but unverified inside; (b) is verbose but the compiler *forces* you to handle every new field. For a config that changes rarely and matters enormously, (b) is defensible. Watch out for `??` with booleans though — `override.server?.trustProxy ?? base.server.trustProxy` is correct (`false ?? x` is `false`), but `||` would be wrong.

### 5 — Config that changes at runtime needs a different shape

A frozen module-level object is right for ~95% of config. The exception is anything that must change without a restart — remote feature flags, dynamic rate limits, kill switches. Do not solve this by un-freezing config. Split the two concerns:

```ts
// Static: resolved once, frozen, imported directly.
export const config: DeepReadonly<AppConfig> = buildConfig(process.env);

// Dynamic: behind a function, explicitly not a plain object.
export interface FlagProvider {
  isEnabled(flag: FeatureFlagName, context?: { readonly userId?: string }): boolean;
  readonly lastRefreshedAt: Date;
}

export class RemoteFlagProvider implements FlagProvider {
  #current: FeatureFlags;

  constructor(private readonly fallback: FeatureFlags, private readonly client: FlagServiceClient) {
    this.#current = fallback;                // never start with nothing
  }

  isEnabled(flag: FeatureFlagName): boolean {
    return this.#current[flag];
  }

  /** Swap the WHOLE object atomically — never mutate the existing one. */
  async refresh(): Promise<void> {
    const fetched = await this.client.fetchFlags();
    this.#current = Object.freeze({ ...this.fallback, ...fetched });
  }

  readonly lastRefreshedAt = new Date();
}
```

The type-level lesson: a **value** (`config.database.poolSize`) promises stability; a **function** (`flags.isEnabled("newCheckout")`) promises nothing. Use the shape that tells the truth. If a caller can hold `config.features.newCheckout` in a local variable and it might go stale, the type is lying.

### 6 — Deriving env-var documentation from the type

Because your config is one typed object built by one function, you can generate the `.env.example` your README keeps forgetting to update:

```ts
interface EnvVarSpec {
  readonly name: string;
  readonly required: boolean;
  readonly description: string;
  readonly example: string;
  readonly secret: boolean;
}

export const ENV_VAR_SPECS = [
  { name: "NODE_ENV",     required: true,  secret: false, description: "Deployment environment",  example: "production" },
  { name: "PORT",         required: false, secret: false, description: "HTTP listen port",        example: "8080" },
  { name: "DATABASE_URL", required: true,  secret: true,  description: "Postgres connection URL", example: "postgres://user:pass@host:5432/db" },
  { name: "JWT_SECRET",   required: true,  secret: true,  description: "≥32 chars, HS256 key",    example: "<48 random bytes, base64>" },
] as const satisfies readonly EnvVarSpec[];

export type KnownEnvVar = typeof ENV_VAR_SPECS[number]["name"];
//    ↑ "NODE_ENV" | "PORT" | "DATABASE_URL" | "JWT_SECRET"

/** `npm run config:example > .env.example` */
export function renderEnvExample(): string {
  return ENV_VAR_SPECS.map((spec) =>
    [
      `# ${spec.description}${spec.required ? " (required)" : " (optional)"}`,
      `${spec.name}=${spec.secret ? "" : spec.example}`,
    ].join("\n"),
  ).join("\n\n");
}
```

You can go one step further and detect *unknown* env vars at boot — a `DATBASE_URL` typo in a Kubernetes manifest becomes a warning instead of a mystery:

```ts
export function warnOnUnknownAcmeVars(env: NodeJS.ProcessEnv, logger: Logger): void {
  const known = new Set<string>(ENV_VAR_SPECS.map((spec) => spec.name));
  for (const name of Object.keys(env)) {
    if (name.startsWith("ACME_") && !known.has(name)) {
      logger.warn(`Unknown configuration variable ${name} — is it a typo?`);
    }
  }
}
```

### 7 — Augmenting `ProcessEnv` is a trap

You will find this pattern online. Understand why it is dangerous:

```ts
// ❌ Global augmentation — a LIE the compiler will happily believe.
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;      // claims it's always present
      PORT: string;
      NODE_ENV: "development" | "production";
    }
  }
}

const url = process.env.DATABASE_URL;      // typed `string` — but it's `undefined` at runtime
new Pool({ connectionString: url });       // crashes with a useless error, four layers deep
```

The declaration asserts a fact nobody verified. Worse, it applies *globally*, so every file in the codebase now believes it can read `process.env.DATABASE_URL` safely — which actively encourages the scattering you were trying to eliminate.

The narrow, honest version of this idea is to type env vars as *possibly missing* and force a parse:

```ts
// ✅ If you must augment, augment truthfully — everything optional:
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      readonly DATABASE_URL?: string;
      readonly PORT?: string;
      readonly JWT_SECRET?: string;
    }
  }
}
// You still have to handle `undefined`, which is the point.
```

But even this is inferior to a `loadConfig(env)` function, because the augmentation gives you no place to put defaults, ranges, or cross-field rules.

### 8 — Secrets: what the type system can and cannot do for you

Types cannot keep a secret. They can, however, make leaking one *visible in review*.

```ts
// A branded string type for values that must never be logged or serialized:
declare const secretBrand: unique symbol;
export type Secret<T extends string = string> = T & { readonly [secretBrand]: "Secret" };

export function markSecret(value: string): Secret {
  return value as Secret;
}

export interface AuthConfig {
  readonly jwtSecret: Secret;      // ← visually flagged everywhere it's used
  readonly jwtIssuer: string;
}

// Now a function that logs config can refuse secrets structurally:
type Loggable<T> =
  T extends Secret ? never
  : T extends object ? { readonly [K in keyof T]: Loggable<T[K]> }
  : T;

declare function logConfig(value: Loggable<AppConfig>): void;
// logConfig(config);   // ❌ jwtSecret: Secret is not assignable to never
logConfig(redactConfig(config));   // ✅ redaction replaced it with a plain string
```

This is a *speed bump*, not a wall — `String(secret)` and template interpolation both erase the brand. Combine it with:

- a `redactConfig` used at every log site (as in Example 2),
- a logger serializer that scrubs known key names (`password`, `secret`, `token`, `authToken`),
- and never putting secrets in `BASE_CONFIG` at all — only placeholders that validation rejects outside development.

### 9 — Where config should *not* live: import-time side effects

`export const config = buildConfig(process.env)` runs at import time. That is deliberate (fail fast), but it has consequences:

```ts
// ⚠️ Any module that imports config transitively now throws on import if the
//    environment is invalid — including a unit test that never touches config.
import { config } from "../config";        // throws here, before your test body runs
```

Two mitigations, both worth knowing:

```ts
// (a) Keep buildConfig exported and pure, so tests bypass the singleton:
import { buildConfig } from "../config";   // pure — doesn't read process.env itself
const testConfig = buildConfig({ NODE_ENV: "test", DATABASE_URL: "postgres://…" });

// (b) Inject config rather than importing it, in anything you unit-test:
export function createUserService(deps: {
  readonly config: Pick<AppConfig, "auth">;    // ← only the slice this service needs
  readonly userRepo: UserRepository;
}) {
  return {
    async issueToken(userId: string): Promise<string> {
      return jwt.sign({ sub: userId }, deps.config.auth.jwtSecret, {
        expiresIn: deps.config.auth.accessTokenTtlSeconds,
      });
    },
  };
}
```

Note the `Pick<AppConfig, "auth">` — declaring the *slice* a module needs is better than passing the whole object, because it documents the dependency and makes the test fixture tiny. See `32 — Utility types`.

### 10 — `noUncheckedIndexedAccess` and config lookups

This tsconfig flag changes how index access is typed, and config code is where you feel it most:

```ts
// tsconfig.json: { "compilerOptions": { "noUncheckedIndexedAccess": true } }

const byEnvironment: Record<Environment, AppConfig> = { /* all four keys */ };
const c1 = byEnvironment["production"];    // AppConfig ✅ — closed union key, still not undefined

const loose: Record<string, string> = process.env as Record<string, string>;
const c2 = loose["PORT"];                  // string | undefined ✅ — the truth

const flags: FeatureFlags = config.features;
const f1 = flags.newCheckout;              // boolean ✅ — a known property, not an index

const names: readonly FeatureFlagName[] = FEATURE_FLAG_NAMES;
const first = names[0];                    // FeatureFlagName | undefined ✅ — arrays too
```

The flag is what makes `Record<Environment, AppConfig>` genuinely *better* than `Record<string, AppConfig>` rather than merely tidier. Without it both index as `AppConfig`, and only one of them is telling the truth.

---

## Common mistakes

### Mistake 1 — Reading `process.env` throughout the codebase

```ts
// ❌ WRONG — scattered, unparsed, undocumented, untestable.
// src/db/pool.ts
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,           // string | undefined
  max: Number(process.env.DB_POOL_SIZE) || 10,          // NaN → 10, silently
});

// src/server.ts
app.listen(process.env.PORT || 3000);                   // "8080" (string) or 3000 (number)

// src/billing.ts
const stripe = new Stripe(process.env.STRIPE_KEY!);     // `!` is a lie — it IS undefined in staging

// ✅ RIGHT — one typed module, read once, everything downstream gets real types.
// src/config/index.ts
export const config: DeepReadonly<AppConfig> = buildConfig(process.env);

// src/db/pool.ts
import { config } from "../config";
const pool = new Pool({
  connectionString: config.database.url,      // string, validated as a postgres:// URL
  max:              config.database.poolSize, // number, range-checked 1–200
});

// src/server.ts
app.listen(config.server.port);               // number, range-checked 1–65535
```

### Mistake 2 — Casting `process.env.NODE_ENV` instead of validating it

```ts
// ❌ WRONG — the cast asserts something nobody checked.
type Environment = "development" | "staging" | "production";
const environment = process.env.NODE_ENV as Environment;

const configByEnvironment: Record<Environment, AppConfig> = { /* … */ };
const active = configByEnvironment[environment];
// NODE_ENV="prodution" (typo in the deploy manifest) → `active` is undefined
// → every property access crashes with "Cannot read properties of undefined"

// ✅ RIGHT — validate at runtime, and the return TYPE does the narrowing.
const ENVIRONMENTS = ["development", "staging", "production"] as const;
type Environment = typeof ENVIRONMENTS[number];

function parseEnvironment(raw: string | undefined): Environment {
  if (raw !== undefined && (ENVIRONMENTS as readonly string[]).includes(raw)) {
    return raw as Environment;                 // ✅ justified — membership just proven
  }
  throw new ConfigError([
    `NODE_ENV="${raw ?? "(unset)"}" is invalid. Expected one of: ${ENVIRONMENTS.join(", ")}`,
  ]);
}

const environment = parseEnvironment(process.env.NODE_ENV);   // crashes at boot, loudly
```

### Mistake 3 — Shallow-merging nested config

```ts
// ❌ WRONG — spread and Object.assign replace whole subtrees.
const baseConfig = {
  server:   { port: 3000, host: "0.0.0.0", requestTimeoutMs: 30_000 },
  database: { url: "postgres://localhost/acme", poolSize: 5, sslMode: "disable" },
};
const productionOverrides = { server: { port: 8080 } };

const merged = { ...baseConfig, ...productionOverrides };
// merged.server === { port: 8080 }
//   → host and requestTimeoutMs are GONE.
//   → `app.listen(merged.server.port, merged.server.host)` binds to undefined,
//     which Node interprets as localhost. Your pod fails its health check.

// ❌ ALSO WRONG — Object.assign has the same shallow behaviour.
const alsoMerged = Object.assign({}, baseConfig, productionOverrides);

// ✅ RIGHT — a typed recursive merge, with arrays replaced and undefined ignored.
const merged2 = deepMerge(baseConfig, productionOverrides);
// { server: { port: 8080, host: "0.0.0.0", requestTimeoutMs: 30_000 }, database: { … } } ✅
```

### Mistake 4 — Treating env-var strings as their intended types

```ts
// ❌ WRONG — every one of these is a real production bug.
const port = process.env.PORT || 3000;
app.listen(port);                                     // "8080" — works, until `port + 1`

const retries = process.env.MAX_RETRIES ?? 3;
for (let i = 0; i < retries; i++) { }                 // "5" → `0 < "5"` is true, `1 < "5"` is true…
                                                      //   coerces, works by accident, breaks on "10"

if (process.env.ENABLE_BILLING) {                     // "false" is a non-empty string → TRUTHY
  chargeCustomer();                                   // billing runs when you disabled it 🔥
}

const ratio = parseFloat(process.env.SAMPLE_RATIO);   // NaN if unset — and NaN < 1 is FALSE,
if (Math.random() < ratio) trace();                   // so tracing silently never runs

// ✅ RIGHT — one parser per type, with defaults, ranges, and loud failure.
const port    = integer(env, "PORT", 3000, 1, 65535);              // number ✅
const retries = integer(env, "MAX_RETRIES", 3, 0, 10);             // number ✅
const billing = boolean(env, "ENABLE_BILLING", false);             // boolean ✅ ("false" → false)
const ratio   = number(env, "SAMPLE_RATIO", 1.0, 0, 1);            // number ✅ range-checked
```

### Mistake 5 — Shallow `readonly` and no runtime freeze

```ts
// ❌ WRONG — `readonly` on the outer property only.
interface AppConfig {
  readonly database: { poolSize: number; url: string };
  readonly features: Record<string, boolean>;
}
declare const config: AppConfig;

config.database.poolSize = 100;              // ✅ compiles — nested object is mutable 🔥
config.features.newCheckout = true;          // ✅ compiles — every later request sees it

// ❌ ALSO WRONG — deep readonly but no freeze. `as any` walks straight through.
const config2: DeepReadonly<AppConfig> = buildConfig(process.env);
(config2 as any).database.poolSize = 100;    // runtime mutation, no error

// ✅ RIGHT — DeepReadonly for the compiler, deepFreeze for the runtime.
export const config: DeepReadonly<AppConfig> = deepFreeze(buildConfig(process.env));
// config.database.poolSize = 100;           // ❌ Cannot assign to 'poolSize' — read-only
// (config as any).database.poolSize = 100;  // 💥 TypeError: Cannot assign to read only property
```

### Mistake 6 — Validating lazily, or one issue at a time

```ts
// ❌ WRONG — validation happens where the value is USED, so a misconfigured
//    service boots healthy and fails on the first checkout, days later.
export function chargeCustomer(userId: string, amountCents: number) {
  if (!process.env.STRIPE_SECRET_KEY) {
    throw new Error("STRIPE_SECRET_KEY is not set");   // discovered by a customer
  }
  // …
}

// ❌ ALSO WRONG — throws on the FIRST problem, so fixing a fresh environment is
//    a redeploy-per-variable game of whack-a-mole.
function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  if (!env.DATABASE_URL) throw new Error("DATABASE_URL is required");
  if (!env.REDIS_URL)    throw new Error("REDIS_URL is required");
  if (!env.JWT_SECRET)   throw new Error("JWT_SECRET is required");
  // …four more deploys to discover all four
}

// ✅ RIGHT — collect every issue, throw once, at boot, with an actionable message.
function loadConfig(env: NodeJS.ProcessEnv): AppConfig {
  const issues: string[] = [];
  if (!env.DATABASE_URL) issues.push("DATABASE_URL is required (postgres:// URL)");
  if (!env.REDIS_URL)    issues.push("REDIS_URL is required (redis:// URL)");
  if (!env.JWT_SECRET)   issues.push("JWT_SECRET is required (≥32 characters)");
  // …plus every range, enum and cross-field check
  if (issues.length > 0) throw new ConfigError(issues);
  // …
}
// Invalid configuration:
//   • DATABASE_URL is required (postgres:// URL)
//   • REDIS_URL is required (redis:// URL)
//   • JWT_SECRET is required (≥32 characters)
```

### Mistake 7 — Declaring `ProcessEnv` keys as required

```ts
// ❌ WRONG — a global lie. Nothing verified any of this.
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      JWT_SECRET:   string;
      NODE_ENV:     "development" | "production";
    }
  }
}
const secret = process.env.JWT_SECRET;    // typed `string`, actually `undefined` in staging
jwt.sign(payload, secret);                // "secretOrPrivateKey must have a value" at request time

// ✅ RIGHT — no augmentation. Parse once, in one function, into a real type.
export function loadAuthConfig(env: NodeJS.ProcessEnv, isProduction: boolean): AuthConfig {
  const jwtSecret = env.JWT_SECRET?.trim();
  if (isProduction && (!jwtSecret || jwtSecret.length < 32)) {
    throw new ConfigError(["JWT_SECRET must be set and at least 32 characters in production"]);
  }
  return {
    jwtSecret: jwtSecret ?? "insecure-development-secret",
    jwtIssuer: env.JWT_ISSUER?.trim() ?? "https://api.acme.local",
    accessTokenTtlSeconds:  900,
    refreshTokenTtlSeconds: 2_592_000,
    bcryptRounds: isProduction ? 12 : 4,
  };
}
```

### Mistake 8 — Using an `if` chain instead of a `Record` for environments

```ts
// ❌ WRONG — silently falls through, and adding an environment changes nothing here.
function getConfig(environment: Environment): AppConfig {
  if (environment === "production") return productionConfig;
  if (environment === "staging")    return stagingConfig;
  return developmentConfig;           // "test" gets dev config. Tests hit the dev database. 🔥
}

// ❌ ALSO WRONG — a Record keyed by `string` reintroduces undefined.
const byName: Record<string, AppConfig> = { development: devConfig, production: prodConfig };
const active = byName[environment];   // typed AppConfig, may be undefined at runtime

// ✅ RIGHT — Record over the closed union. Adding "preview" to Environment
//    breaks the build here, which is exactly when you want to be told.
const CONFIG_BY_ENVIRONMENT: Record<Environment, AppConfig> = {
  development: developmentConfig,
  test:        testConfig,
  staging:     stagingConfig,
  production:  productionConfig,
};
const active = CONFIG_BY_ENVIRONMENT[environment];   // AppConfig — total, no fallback needed
```

---

## Practice exercises

### Exercise 1 — easy

Build a minimal typed config module for a Node service from scratch.

Define:

1. `ENVIRONMENTS` as an `as const` array of `"development" | "test" | "staging" | "production"`, and derive `type Environment = typeof ENVIRONMENTS[number]`.
2. `LOG_LEVELS` as an `as const` array of `"debug" | "info" | "warn" | "error"`, and derive `type LogLevel`.
3. An `AppConfig` interface with `readonly` properties: `environment: Environment`, `server: { port: number; host: string }`, `database: { url: string; poolSize: number }`, `logging: { level: LogLevel; pretty: boolean }`.
4. A `ConfigError` class that takes `readonly issues: readonly string[]` and formats them into a bulleted `message`.
5. Three parser helpers — `requiredString`, `integer(name, fallback, min, max)` and `boolean(name, fallback)` — that push onto a shared `issues` array rather than throwing immediately. `boolean` must treat `"false"`, `"0"`, `"no"` and `"off"` as `false`.
6. `loadConfig(env: NodeJS.ProcessEnv): AppConfig` that uses them, throws a single `ConfigError` if any issue was recorded, and otherwise returns a complete `AppConfig`.
7. `export const config: AppConfig = Object.freeze(loadConfig(process.env))`.

Then prove three things to yourself:
- `config.server.port` is `number`, not `string | undefined`.
- `config.logging.level = "verbose"` is a compile error.
- `loadConfig({ NODE_ENV: "test", PORT: "abc" })` throws a `ConfigError` whose `issues` mention both the missing `DATABASE_URL` *and* the bad `PORT`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build the layered config: base defaults, environment overrides, and env-var overrides.

Write:

1. A `BASE_CONFIG` declared with `as const satisfies AppConfig`, containing safe development defaults for every field of the `AppConfig` from Exercise 1 plus a `features` section.
2. `FEATURE_FLAG_NAMES` as an `as const` array of at least `"newCheckout"`, `"betaSearch"`, `"auditLogging"`; derive `FeatureFlagName` and `FeatureFlags = Readonly<Record<FeatureFlagName, boolean>>`; and a `DEFAULT_FEATURE_FLAGS` object declared with `as const satisfies FeatureFlags`. Verify that adding a fourth flag name makes `DEFAULT_FEATURE_FLAGS` fail to compile.
3. A `DeepPartial<T>` mapped type that treats arrays, `Date` and functions as leaves.
4. `OVERRIDES_BY_ENVIRONMENT: Record<Environment, DeepPartial<AppConfig>>` where each entry states only its differences. Give `test` a pool size of 2, `staging` a port of 8080 and `betaSearch: true`, and `production` a pool size of 40 and `pretty: false`. Verify that adding `"preview"` to `Environment` breaks this object.
5. `deepMerge<T>(base: T, override: DeepPartial<T>): T` that merges plain objects recursively, replaces arrays wholesale, ignores `undefined` in the override, and mutates neither argument.
6. `mergeAll<T>(base: T, ...layers: readonly DeepPartial<T>[]): T`.
7. `overridesFromEnv(env: NodeJS.ProcessEnv): DeepPartial<AppConfig>` that returns `undefined` for every unset variable, including feature flags read from `FEATURE_<UPPER_SNAKE>` names derived mechanically from `FEATURE_FLAG_NAMES`.
8. `buildConfig(env): AppConfig` that composes all three layers in the right precedence order.

Then write tests proving: an unset `PORT` inherits the environment layer's value; a set `PORT` beats it; `server.host` survives a `server.port`-only override; and `CORS_ORIGINS` replaces rather than concatenates.

```ts
// Write your code here
```

### Exercise 3 — hard

Build the full production-grade config module, with deep immutability, cross-field validation, secret redaction, and compile-time guarantees.

Requirements:

1. **`DeepReadonly<T>`** that leaves `Date`, `RegExp`, functions and primitives alone, maps `ReadonlyMap`/`ReadonlySet` correctly, turns arrays into `readonly T[]`, and recurses into plain objects. Prove `config.server.corsOrigins.push("x")` is a compile error.

2. **`deepFreeze<T>(value: T): DeepReadonly<T>`** that recursively freezes and returns the value typed as `DeepReadonly<T>`. Prove that `(config as any).server.port = 1` throws a `TypeError`.

3. **A `validateConfig(candidate: AppConfig): readonly string[]`** implementing at least these cross-field rules:
   - `refreshTokenTtlSeconds > accessTokenTtlSeconds` (always)
   - `database.statementTimeoutMs < server.requestTimeoutMs` (always)
   - `database.url` parses as a URL with a `postgres:`/`postgresql:` protocol (always)
   - outside development: `jwtSecret.length >= 32`, `jwtSecret` is not a known placeholder, `bcryptRounds >= 12`, `trustProxy === true`, all `corsOrigins` are `https://`
   - production only: `sslMode === "verify-full"`, `poolSize >= 10`, `otelEndpoint !== null`, `features.auditLogging === true`

4. **A `Secret` branded string type** (`declare const secretBrand: unique symbol`) used for `auth.jwtSecret`, `database.url` and `redis.url`, plus a `Loggable<T>` conditional type that maps any `Secret` to `never`. Write `declare function logConfig(value: Loggable<AppConfig>): void` and prove that `logConfig(config)` fails to compile while `logConfig(redactConfig(config))` succeeds.

5. **`redactConfig(source: AppConfig)`** that masks URL credentials via the `URL` API (`parsed.password = "***"`) and replaces `jwtSecret` with `` `***(${length} chars)` ``.

6. **A compile-time assertion** that `DEFAULT_FEATURE_FLAGS` has `auditLogging: true`, using `as const satisfies` plus a conditional type that resolves to `never` when the flag is `false`. Verify the build breaks if someone flips it.

7. **`ENV_VAR_SPECS`** declared `as const satisfies readonly EnvVarSpec[]`, from which you derive `type KnownEnvVar = typeof ENV_VAR_SPECS[number]["name"]`, a `renderEnvExample()` generator, and a boot-time warning for any `ACME_`-prefixed variable not in the list.

8. **A `Pick`-based dependency slice**: write `createUserService(deps: { config: Pick<AppConfig, "auth" | "features">; userRepo: UserRepository })` and show that its unit test needs a fixture containing only `auth` and `features`, not the whole config.

9. **A `FlagProvider` interface** for dynamic flags whose `refresh()` swaps a whole frozen object atomically rather than mutating, and explain in a comment why `config.features.newCheckout` (a value) and `flags.isEnabled("newCheckout")` (a function) make different promises to the caller.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Derive a union from a value list ───────────────────────────────────────
const ENVIRONMENTS = ["development", "test", "staging", "production"] as const;
type Environment = typeof ENVIRONMENTS[number];   // "development" | "test" | ...

// ── as const satisfies: checked AND narrow ────────────────────────────────
const BASE_CONFIG = { server: { port: 3000, host: "0.0.0.0" } } as const satisfies AppConfig;
type BasePort = typeof BASE_CONFIG["server"]["port"];   // 3000 (not number)

// ── Exhaustive environment map (adding an Environment breaks this) ────────
const OVERRIDES_BY_ENVIRONMENT: Record<Environment, DeepPartial<AppConfig>> = {
  development: {}, test: {}, staging: {}, production: {},
};

// ── The two mapped types ──────────────────────────────────────────────────
type DeepPartial<T>  = T extends Immutable ? T
  : T extends readonly (infer E)[] ? readonly E[]
  : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

type DeepReadonly<T> = T extends Immutable ? T
  : T extends readonly (infer E)[] ? readonly DeepReadonly<E>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;

// ── Layered merge, lowest precedence first ────────────────────────────────
const merged = mergeAll(BASE_CONFIG, OVERRIDES_BY_ENVIRONMENT[environment], overridesFromEnv(env));

// ── Feature flags as a total record over a closed key union ──────────────
type FeatureFlagName = typeof FEATURE_FLAG_NAMES[number];
type FeatureFlags    = Readonly<Record<FeatureFlagName, boolean>>;
const enabled = config.features.newCheckout;      // boolean — no optionality, no undefined

// ── Env parsing: never trust a string ────────────────────────────────────
const port    = integer(env, "PORT", 3000, 1, 65535);      // NOT Number(env.PORT) || 3000
const enabled2 = boolean(env, "ENABLE_BILLING", false);    // NOT Boolean(env.ENABLE_BILLING)
const level   = pick(env, "LOG_LEVEL", LOG_LEVELS);        // NOT env.LOG_LEVEL as LogLevel

// ── Validate once, report everything, exit cleanly ───────────────────────
const problems = validateConfig(merged);
if (problems.length > 0) throw new ConfigError(problems);   // EX_CONFIG = exit code 78

// ── The single frozen export ─────────────────────────────────────────────
export const config: DeepReadonly<AppConfig> = deepFreeze(merged);
```

| Decision | Do this | Not this |
|---|---|---|
| Where env is read | one `src/config` module | `process.env` scattered everywhere |
| Config shape | an explicit `AppConfig` interface | implicit, discovered by grep |
| Config literals | `as const satisfies AppConfig` | bare `as const`, or a plain annotation |
| Deriving unions | `typeof LIST[number]` from an `as const` array | a hand-maintained union + array |
| Environment selection | `Record<Environment, T>` | an `if`/`else` chain |
| Environment key type | a closed union | `string` |
| Overrides | `DeepPartial<AppConfig>` | untyped partial objects |
| Merging | recursive `deepMerge` | `{ ...base, ...override }` |
| Arrays in a merge | replaced wholesale | concatenated or index-merged |
| `undefined` in overrides | ignored (inherits) | overwrites with `undefined` |
| Immutability | `DeepReadonly` **and** `deepFreeze` | shallow `readonly` only |
| Booleans from env | an explicit `"true"/"false"` parser | `Boolean(env.X)` or `!!env.X` |
| Numbers from env | parse + range check + `Number.isInteger` | `Number(env.X) \|\| fallback` |
| Enum-ish values | validate then narrow | `env.X as LogLevel` |
| Validation timing | once, at boot, before listening | lazily, at first use |
| Validation output | every issue collected, thrown once | throw on the first problem |
| `NodeJS.ProcessEnv` | leave it alone | augment keys as required `string` |
| Logging config | `redactConfig(config)` | the raw object |
| Testing | `buildConfig(fakeEnv)` — a pure function | mutating `process.env` + `resetModules` |
| Dynamic values | a `FlagProvider` function | un-freezing the config object |

---

## Connected topics

- **11 — Literal types** — `as const`, literal widening, and deriving `Environment` from `typeof ENVIRONMENTS[number]` are the foundation of everything here.
- **32 — Utility types** — `Record<Environment, AppConfig>`, `Readonly<T>`, `Partial<T>` and `Pick<AppConfig, "auth">` for dependency slices.
- **43 — Mapped types** — `DeepPartial<T>` and `DeepReadonly<T>` are mapped types with a recursive twist; `{ readonly [K in keyof T]: … }` is the core mechanism.
- **44 — Conditional types** — the `T extends Immutable ? T : …` chains that make `DeepReadonly` skip `Date`, arrays and functions correctly.
- **35 — Readonly class properties** — the compile-time half of immutability; `Object.freeze` is the runtime half, and you want both.
- **42 — Discriminated unions** — `Environment` as a closed union is what makes `Record<Environment, T>` exhaustive and total.
- **58 — Typing API responses** — the same discipline applied to the wire: one declared shape, built by helpers, validated at the boundary.
- **48 — unknown vs any** — `process.env` values are effectively `unknown`; the parsers in this doc are the narrowing step you must not skip with a cast.
