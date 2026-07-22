# 65 — Readonly and Immutability Patterns

## What is this?

**Readonly** is TypeScript's way of saying "this value must not be reassigned after it is created". It is a **compile-time contract**, enforced by the type checker, and it comes in several flavours:

```ts
// 1. readonly property modifier — on interfaces, type aliases, and classes
interface UserSession {
  readonly userId:    number;   // cannot be reassigned after creation
  readonly authToken: string;
  lastSeenAt:         Date;     // mutable — no readonly
}

// 2. Readonly<T> — a mapped type that makes every top-level property readonly
type FrozenSession = Readonly<UserSession>;
// = { readonly userId: number; readonly authToken: string; readonly lastSeenAt: Date }

// 3. readonly arrays — no push/pop/splice/sort, no index assignment
const allowedOrigins: readonly string[] = ["https://app.example.com"];
const allowedMethods: ReadonlyArray<string> = ["GET", "POST"];  // identical meaning

// 4. as const — freezes a literal expression into its narrowest, deeply readonly type
const RETRY_DELAYS_MS = [100, 500, 2_000] as const;
// type: readonly [100, 500, 2000]

// 5. ReadonlyMap / ReadonlySet — read-only views over collections
const featureFlags: ReadonlyMap<string, boolean> = new Map([["newCheckout", true]]);
```

The critical thing to understand up front — and the thing most tutorials bury — is that **none of this exists at runtime**. `readonly` is erased when TypeScript compiles to JavaScript. It stops *your code* from mutating; it does not stop a third-party library, a `JSON.parse` result, or an `as any` cast. It is a design tool, not a security boundary.

## Why does it matter?

Backend code is where accidental mutation does the most damage, because objects live longer and are shared more widely than in a UI:

- A **config object** loaded once at boot and injected into forty modules. One module does `config.dbPool.max = 5` for a local test and now every module sees it.
- A **cached entity** returned from a repository. The caller does `user.email = user.email.toLowerCase()` for a comparison — the cache is now poisoned for every subsequent request.
- A **default options object** shared across calls. `Object.assign(DEFAULT_OPTS, userOpts)` mutates the module-level default, and request #2 inherits request #1's overrides.
- A **request context** passed down a middleware chain. Any middleware can silently rewrite `ctx.userId` and the auth decision made three layers up becomes a lie.
- **Shared constant arrays** — `ALLOWED_ROLES.sort()` in a logging helper mutates the array in place (`sort` is not pure) and reorders the permission list everyone else reads.

Every one of those bugs is invisible in review, non-deterministic in production, and reproduces only under load. `readonly` turns all of them into red squiggles in your editor.

There is a second, subtler payoff: **readonly is documentation the compiler enforces**. When a function takes `readonly UserRecord[]`, you know without reading the body that it will not reorder or splice your array. That guarantee is worth more than a comment, because comments rot.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: nothing stops mutation, so you defend everywhere ─────────────

// Module-level config, loaded once at boot:
const appConfig = {
  db:    { host: "localhost", port: 5432, poolMax: 20 },
  redis: { host: "localhost", port: 6379 },
  allowedOrigins: ["https://app.example.com", "https://admin.example.com"],
};

// Somewhere in a "harmless" helper, six files away:
function getSortedOrigins() {
  return appConfig.allowedOrigins.sort();   // 💥 sort() mutates IN PLACE
}

// Somewhere in a test setup that leaked into a shared module:
function useTestDb() {
  appConfig.db.poolMax = 1;                 // 💥 every consumer now has poolMax 1
}

// A repository that caches:
const userCache = new Map();
async function findUserById(userId) {
  if (userCache.has(userId)) return userCache.get(userId);
  const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);
  userCache.set(userId, user);
  return user;                              // returns the CACHED reference
}

// A caller that "just normalises" the email:
const user = await findUserById(42);
user.email = user.email.trim().toLowerCase();  // 💥 mutated the cached object
user.passwordHash = undefined;                 // 💥 deleted from the cache too

// Defensive JS: you clone everything, everywhere, forever:
function findUserSafely(userId) {
  return structuredClone(userCache.get(userId));  // allocation on every read
}
// ...and you still forget in 30% of call sites, and you pay the clone cost
// even in the 95% of call sites that never mutate anything.
```

```ts
// ── TypeScript: the compiler enforces it, at zero runtime cost ───────────────

interface DbConfig {
  readonly host:    string;
  readonly port:    number;
  readonly poolMax: number;
}

interface AppConfig {
  readonly db:             DbConfig;
  readonly redis:          { readonly host: string; readonly port: number };
  readonly allowedOrigins: readonly string[];
}

const appConfig: AppConfig = {
  db:    { host: "localhost", port: 5432, poolMax: 20 },
  redis: { host: "localhost", port: 6379 },
  allowedOrigins: ["https://app.example.com", "https://admin.example.com"],
};

function getSortedOrigins(): string[] {
  return appConfig.allowedOrigins.sort();
  // ❌ Error: Property 'sort' does not exist on type 'readonly string[]'.
  //    Did you mean 'toSorted'?
  //    ↑ the compiler literally tells you the non-mutating alternative
}

function getSortedOriginsFixed(): string[] {
  return [...appConfig.allowedOrigins].sort();  // ✅ copy first, then sort
}

function useTestDb(): void {
  appConfig.db.poolMax = 1;
  // ❌ Error: Cannot assign to 'poolMax' because it is a read-only property.
}

// Repository returns a readonly view — callers physically cannot corrupt the cache:
interface UserRecord {
  readonly userId:       number;
  readonly email:        string;
  readonly passwordHash: string;
  readonly createdAt:    Date;
}

const userCache = new Map<number, UserRecord>();

async function findUserById(userId: number): Promise<UserRecord | null> {
  const cached = userCache.get(userId);
  if (cached) return cached;                  // ✅ safe to hand out the reference
  const user = await db.queryOne<UserRecord>("SELECT * FROM users WHERE id = $1", [userId]);
  if (user) userCache.set(userId, user);
  return user;
}

const user = await findUserById(42);
if (user) {
  user.email = user.email.trim();
  // ❌ Error: Cannot assign to 'email' because it is a read-only property.

  // ✅ The right move — derive a new value instead of mutating:
  const normalisedEmail = user.email.trim().toLowerCase();
}
```

The revelation: in JavaScript you pay for safety with **runtime clones on every read**. In TypeScript you pay for it **once, at compile time, with zero bytes of emitted code**. The cache can hand out its own references forever, because no caller is capable of writing to them.

---

## Syntax

```ts
// ── readonly property on an interface ───────────────────────────────────────
interface AuthToken {
  readonly token:     string;   // assignable at creation only
  readonly expiresAt: Date;     // the Date OBJECT is still mutable — see Going deeper
  refreshCount:       number;   // mutable
}

// ── readonly on a type alias ────────────────────────────────────────────────
type RequestId = { readonly value: string };

// ── readonly class property ─────────────────────────────────────────────────
class UserService {
  readonly serviceName = "UserService";        // set at declaration
  private readonly repo: UserRepository;       // set in constructor only
  constructor(repo: UserRepository) {
    this.repo = repo;                          // ✅ allowed inside the constructor
  }
  reset(): void {
    // this.repo = other;                      // ❌ Error outside the constructor
  }
}

// ── Readonly<T> utility — shallow, top-level only ───────────────────────────
interface CreateUserBody { email: string; password: string; roles: string[] }
type FrozenBody = Readonly<CreateUserBody>;
// { readonly email: string; readonly password: string; readonly roles: string[] }
//                                                      ↑ the ARRAY is still mutable

// ── readonly arrays — two identical spellings ───────────────────────────────
let roles1: readonly string[]     = ["admin", "user"];   // shorthand (preferred)
let roles2: ReadonlyArray<string> = ["admin", "user"];   // generic form
// roles1.push("x");        // ❌ push does not exist
// roles1[0] = "x";         // ❌ index signature is read-only

// ── readonly tuples ─────────────────────────────────────────────────────────
type Coordinate = readonly [lat: number, lng: number];   // fixed length, no mutation

// ── as const — deep readonly + literal narrowing ────────────────────────────
const HTTP_STATUS = { OK: 200, NOT_FOUND: 404, SERVER_ERROR: 500 } as const;
// { readonly OK: 200; readonly NOT_FOUND: 404; readonly SERVER_ERROR: 500 }

// ── ReadonlyMap / ReadonlySet ───────────────────────────────────────────────
const flags: ReadonlyMap<string, boolean> = new Map([["betaCheckout", true]]);
// flags.set("x", true);    // ❌ set does not exist on ReadonlyMap
const admins: ReadonlySet<number> = new Set([1, 2, 3]);
// admins.add(4);           // ❌ add does not exist on ReadonlySet

// ── readonly index signature ────────────────────────────────────────────────
interface ReadonlyHeaders { readonly [headerName: string]: string }

// ── Mutable<T> — the inverse (not built in; you write it) ───────────────────
type Mutable<T> = { -readonly [K in keyof T]: T[K] };   // the '-' removes readonly
```

---

## How it works — concept by concept

### Concept 1 — `readonly` blocks assignment, not mutation

This is the single most misunderstood point. `readonly` protects the **binding**, not the **value behind it**.

```ts
interface SessionRecord {
  readonly userId:    number;
  readonly roles:     string[];        // readonly PROPERTY, mutable ARRAY
  readonly issuedAt:  Date;            // readonly PROPERTY, mutable DATE
  readonly metadata:  { ip: string };  // readonly PROPERTY, mutable OBJECT
}

const session: SessionRecord = {
  userId:   42,
  roles:    ["user"],
  issuedAt: new Date(),
  metadata: { ip: "10.0.0.1" },
};

session.userId = 99;              // ❌ Cannot assign to 'userId' — it is read-only
session.roles  = ["admin"];       // ❌ Cannot assign to 'roles'  — it is read-only

session.roles.push("admin");      // ✅ COMPILES — mutating the array, not the property
session.issuedAt.setFullYear(2030); // ✅ COMPILES — mutating the Date object
session.metadata.ip = "1.2.3.4";  // ✅ COMPILES — metadata's own props aren't readonly

// To actually protect the contents, the CONTENTS must be readonly too:
interface SafeSessionRecord {
  readonly userId:   number;
  readonly roles:    readonly string[];               // ← now push is gone
  readonly issuedAt: Date;                            // Date has no readonly variant
  readonly metadata: { readonly ip: string };         // ← now ip is protected
}

const safe: SafeSessionRecord = { userId: 42, roles: ["user"], issuedAt: new Date(), metadata: { ip: "10.0.0.1" } };
safe.roles.push("admin");   // ❌ Property 'push' does not exist on 'readonly string[]'
safe.metadata.ip = "x";     // ❌ Cannot assign to 'ip' — it is read-only
safe.issuedAt.setFullYear(2030); // ✅ still compiles — Date is inherently mutable
```

**Rule of thumb:** `readonly` is one level deep. Every level you care about needs its own `readonly`.

### Concept 2 — `Readonly<T>` is a shallow mapped type

`Readonly<T>` is not magic; it is four lines in `lib.es5.d.ts`:

```ts
// The actual definition from TypeScript's standard library:
type Readonly<T> = {
  readonly [K in keyof T]: T[K];   // map every key, add readonly, keep the value type
};
```

Because it maps only the **top level**, the value types pass through untouched:

```ts
interface OrderDraft {
  orderId:    string;
  items:      { sku: string; quantity: number }[];
  shipping:   { addressLine1: string; postcode: string };
  totalCents: number;
}

type FrozenDraft = Readonly<OrderDraft>;

declare const draft: FrozenDraft;
draft.orderId = "x";                 // ❌ read-only
draft.items = [];                    // ❌ read-only
draft.items.push({ sku: "A", quantity: 1 });  // ✅ compiles — array wasn't touched
draft.items[0].quantity = 99;                 // ✅ compiles — element wasn't touched
draft.shipping.postcode = "SW1A";             // ✅ compiles — nested object untouched
```

If you need depth, you write `DeepReadonly` yourself (Concept 6) or use `as const` on literals (Concept 4).

The mirror image, `Mutable<T>`, uses the **readonly-removing modifier** `-readonly`:

```ts
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

type EditableDraft = Mutable<FrozenDraft>;
// { orderId: string; items: ...[]; shipping: ...; totalCents: number }
```

You need `Mutable<T>` more often than you'd think — typically inside a builder or a hydration function that assembles an object field-by-field before handing out the frozen version.

### Concept 3 — `readonly T[]` vs `T[]` and the assignability quirk

Readonly arrays are not "arrays with a flag" — they are a **different type** with a smaller API surface. `ReadonlyArray<T>` has `length`, `concat`, `slice`, `map`, `filter`, `find`, `includes`, `indexOf`, `join`, `reduce`, `toSorted`, `toReversed`, `toSpliced`, `with`, and iteration. It does **not** have `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`, `fill`, `copyWithin`, or a writable index signature.

The assignability rule is asymmetric, and it surprises everyone once:

```ts
const mutableRoles:  string[]          = ["admin", "user"];
const readonlyRoles: readonly string[] = ["admin", "user"];

// ✅ mutable → readonly: ALLOWED. You are only removing capabilities.
const view: readonly string[] = mutableRoles;

// ❌ readonly → mutable: FORBIDDEN. You would be re-gaining push/sort.
const escaped: string[] = readonlyRoles;
// Error: The type 'readonly string[]' is 'readonly' and cannot be assigned
//        to the mutable type 'string[]'.
```

This propagates into function parameters, and it's where real friction shows up:

```ts
// A helper you wrote — takes a mutable array:
function summarise(values: number[]): number {
  return values.reduce((sum, v) => sum + v, 0);   // doesn't actually mutate!
}

const amounts: readonly number[] = [100, 250, 900];
summarise(amounts);
// ❌ Argument of type 'readonly number[]' is not assignable to parameter of type 'number[]'

// ✅ Fix 1 — widen the parameter. If you don't mutate, ACCEPT readonly:
function summariseFixed(values: readonly number[]): number {
  return values.reduce((sum, v) => sum + v, 0);
}
summariseFixed(amounts);       // ✅
summariseFixed([1, 2, 3]);     // ✅ mutable arrays still work — readonly is wider

// ✅ Fix 2 — copy at the boundary when the callee genuinely needs to mutate:
summarise([...amounts]);
```

**The practical guideline: every function parameter that is an array and is not mutated should be typed `readonly T[]`.** It costs nothing, accepts strictly more callers, and documents intent. This is the single highest-value readonly habit on a backend.

The same asymmetry applies to object properties, but *only* for the array/tuple case. For plain object properties, readonly-ness is **ignored** during assignability checks:

```ts
interface Writable  { userId: number }
interface ReadOnlyU { readonly userId: number }

declare const ro: ReadOnlyU;
const w: Writable = ro;   // ✅ COMPILES — and then w.userId = 1 mutates the same object!
```

That is a deliberate (and much-debated) design decision in TypeScript, made for structural-typing ergonomics. It means object-level `readonly` is a **guardrail, not a wall** — see Going deeper.

### Concept 4 — `as const` (const assertions)

`as const` does three things at once to a literal expression:

1. Widening is turned off — `"admin"` stays `"admin"` instead of becoming `string`.
2. Every object property becomes `readonly`.
3. Every array literal becomes a `readonly` **tuple** with exact length and per-index literal types.

Crucially, it applies **recursively**, all the way down.

```ts
// ── Without as const ────────────────────────────────────────────────────────
const config1 = {
  environment: "production",
  retryDelays: [100, 500, 2000],
  limits: { maxBodyBytes: 1_048_576 },
};
// type: {
//   environment: string;                  ← widened, useless for narrowing
//   retryDelays: number[];                ← mutable
//   limits: { maxBodyBytes: number };     ← mutable
// }

// ── With as const ───────────────────────────────────────────────────────────
const config2 = {
  environment: "production",
  retryDelays: [100, 500, 2000],
  limits: { maxBodyBytes: 1_048_576 },
} as const;
// type: {
//   readonly environment: "production";
//   readonly retryDelays: readonly [100, 500, 2000];
//   readonly limits: { readonly maxBodyBytes: 1048576 };
// }

config2.environment = "staging";     // ❌ read-only
config2.retryDelays.push(4000);      // ❌ push does not exist on readonly tuple
config2.limits.maxBodyBytes = 1;     // ❌ read-only — DEEP, unlike Readonly<T>
```

The most valuable side effect is **deriving union types from data**, which keeps a single source of truth:

```ts
const USER_ROLES = ["admin", "editor", "viewer", "billing"] as const;

type UserRole = typeof USER_ROLES[number];
// = "admin" | "editor" | "viewer" | "billing"
// Add a role to the array → the union updates automatically. One source of truth.

function hasRole(roles: readonly UserRole[], required: UserRole): boolean {
  return roles.includes(required);
}

const HTTP_ERROR_CODES = {
  BAD_REQUEST:  400,
  UNAUTHORIZED: 401,
  FORBIDDEN:    403,
  NOT_FOUND:    404,
  CONFLICT:     409,
} as const;

type ErrorCodeName  = keyof typeof HTTP_ERROR_CODES;         // "BAD_REQUEST" | "UNAUTHORIZED" | ...
type ErrorCodeValue = typeof HTTP_ERROR_CODES[ErrorCodeName]; // 400 | 401 | 403 | 404 | 409
```

`as const` also fixes the classic "argument widened to string" problem:

```ts
declare function setLogLevel(level: "debug" | "info" | "warn" | "error"): void;

const level = "debug";           // inferred as "debug" — const bindings don't widen for primitives
setLogLevel(level);              // ✅

const settings = { level: "debug" };   // property inferred as string
setLogLevel(settings.level);           // ❌ string is not assignable to the union

const settingsConst = { level: "debug" } as const;
setLogLevel(settingsConst.level);      // ✅ level is "debug"
```

### Concept 5 — `ReadonlyMap`, `ReadonlySet`, and read-only views

`ReadonlyMap<K, V>` and `ReadonlySet<T>` are interfaces in the standard library that expose only the reading half of the API:

```ts
// ReadonlyMap<K, V> has: get, has, size, forEach, keys, values, entries, [Symbol.iterator]
// It does NOT have: set, delete, clear
// ReadonlySet<T>    has: has, size, forEach, keys, values, entries, [Symbol.iterator]
// It does NOT have: add, delete, clear

class FeatureFlagRegistry {
  // Internally mutable — the class needs to write:
  private readonly flags = new Map<string, boolean>();

  register(flagName: string, enabled: boolean): void {
    this.flags.set(flagName, enabled);         // ✅ Map, not ReadonlyMap
  }

  // Externally read-only — callers get a VIEW, not a copy, and cannot write:
  getAll(): ReadonlyMap<string, boolean> {
    return this.flags;                         // ✅ Map is assignable to ReadonlyMap
  }

  isEnabled(flagName: string): boolean {
    return this.flags.get(flagName) ?? false;
  }
}

const registry = new FeatureFlagRegistry();
registry.register("newCheckout", true);

const all = registry.getAll();
all.get("newCheckout");        // ✅ true
all.set("newCheckout", false); // ❌ Property 'set' does not exist on type 'ReadonlyMap'
```

This is the **encapsulated-mutable / exposed-readonly** pattern, and it is the workhorse of immutability on a backend: mutate freely inside the module where you own the data; hand out readonly types at every public boundary. Zero copying, full protection.

Note the same asymmetry as arrays: `Map<K,V>` → `ReadonlyMap<K,V>` is allowed, the reverse is not.

### Concept 6 — `DeepReadonly<T>`

When you need genuine recursive protection over a type you didn't write with `as const`, you build a recursive mapped type:

```ts
// A production-grade DeepReadonly that handles arrays, tuples, Map, Set,
// functions, and primitives correctly:
type DeepReadonly<T> =
  T extends (...args: readonly unknown[]) => unknown ? T                       // leave functions alone
  : T extends ReadonlyArray<infer Element>          ? ReadonlyArray<DeepReadonly<Element>>
  : T extends ReadonlyMap<infer K, infer V>         ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends ReadonlySet<infer M>                  ? ReadonlySet<DeepReadonly<M>>
  : T extends Date                                  ? T                        // Date has no readonly form
  : T extends object                                ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;                                                                          // primitives

interface ServerConfig {
  http:  { port: number; host: string; corsOrigins: string[] };
  db:    { url: string; pool: { min: number; max: number } };
  redis: { url: string; keyPrefix: string };
  featureFlags: Record<string, boolean>;
}

type FrozenServerConfig = DeepReadonly<ServerConfig>;

declare const cfg: FrozenServerConfig;
cfg.http.port = 4000;                    // ❌ read-only (depth 2)
cfg.db.pool.max = 50;                    // ❌ read-only (depth 3)
cfg.http.corsOrigins.push("evil.com");   // ❌ push does not exist (arrays too)
cfg.featureFlags.newCheckout = true;     // ❌ read-only (index signature mapped too)

cfg.http.port;                            // ✅ reading is always fine
for (const origin of cfg.http.corsOrigins) { /* ✅ iteration works */ }
```

The `T extends (...args) => unknown` branch matters: without it, functions get mapped as objects and lose their call signatures. The `Date` branch matters because mapping `Date` produces a useless `{ readonly getTime: () => number; ... }` shape that is no longer assignable to `Date`.

**Cost warning:** `DeepReadonly` on a large, deeply-nested type is genuinely expensive for the type checker and shows up as editor lag. Reach for `as const` on literals first; keep `DeepReadonly` for a handful of high-value types like config.

### Concept 7 — `Object.freeze` and its typing

`Object.freeze` is the *runtime* half. TypeScript's declaration for it returns `Readonly<T>`, so it gives you both guarantees in one call:

```ts
// lib.es5.d.ts (simplified):
// freeze<T>(o: T): Readonly<T>;
// freeze<T extends Function>(f: T): T;
// freeze<T>(a: T[]): readonly T[];

const rateLimits = Object.freeze({
  windowMs:    60_000,
  maxRequests: 100,
  burst:       { size: 10, refillPerSec: 2 },   // ⚠️ NOT frozen — freeze is shallow
});
// type: Readonly<{ windowMs: number; maxRequests: number; burst: {...} }>

rateLimits.windowMs = 1;        // ❌ compile error AND silently ignored / throws at runtime
rateLimits.burst.size = 999;    // ✅ compiles, ✅ mutates — freeze is one level deep too
```

Two runtime gotchas you must know:

```ts
// 1. In non-strict-mode JS, writing to a frozen object fails SILENTLY.
//    In strict mode (which every ES module and every "use strict" file is),
//    it throws TypeError. TypeScript emits ES modules → you get the throw.

// 2. Deep freezing requires recursion, and it must be typed with DeepReadonly:
function deepFreeze<T>(value: T): DeepReadonly<T> {
  if (value !== null && typeof value === "object" && !Object.isFrozen(value)) {
    Object.freeze(value);
    for (const key of Object.getOwnPropertyNames(value)) {
      deepFreeze((value as Record<string, unknown>)[key]);
    }
  }
  return value as DeepReadonly<T>;
}

const frozenConfig = deepFreeze({
  http: { port: 3000, corsOrigins: ["https://app.example.com"] },
  db:   { url: "postgres://localhost/app", pool: { min: 2, max: 20 } },
});

frozenConfig.db.pool.max = 50;   // ❌ compile error AND ⚠️ runtime TypeError
```

**When to actually use `Object.freeze` on a backend:** almost never in hot paths, because frozen objects historically deoptimise property access in V8 and `deepFreeze` costs a full traversal. Use it for module-level config loaded once at boot, and for objects you hand to untrusted plugin code. For everything else, `readonly` types alone are enough — they cost nothing.

### Concept 8 — Immutable update patterns

If you cannot mutate, you must **derive**. The whole vocabulary:

```ts
interface UserProfile {
  readonly userId:      number;
  readonly email:       string;
  readonly displayName: string;
  readonly tags:        readonly string[];
  readonly preferences: {
    readonly theme:         "light" | "dark";
    readonly emailDigest:   boolean;
    readonly timezone:      string;
  };
}

declare const profile: UserProfile;

// ── Update one top-level field ──────────────────────────────────────────────
const renamed: UserProfile = { ...profile, displayName: "New Name" };

// ── Update a NESTED field — you must rebuild every level on the path ────────
const darkMode: UserProfile = {
  ...profile,
  preferences: { ...profile.preferences, theme: "dark" },
};

// ── Array: append ───────────────────────────────────────────────────────────
const tagged: UserProfile = { ...profile, tags: [...profile.tags, "beta-tester"] };

// ── Array: remove by value ──────────────────────────────────────────────────
const untagged: UserProfile = { ...profile, tags: profile.tags.filter((t) => t !== "beta-tester") };

// ── Array: replace at index (ES2023 `with`, or map) ─────────────────────────
const replaced: UserProfile = { ...profile, tags: profile.tags.with(0, "vip") };
const replacedCompat: UserProfile = {
  ...profile,
  tags: profile.tags.map((t, i) => (i === 0 ? "vip" : t)),
};

// ── Array: sort / reverse without mutating (ES2023) ─────────────────────────
const sortedTags: readonly string[] = profile.tags.toSorted();      // ES2023
const sortedCompat: readonly string[] = [...profile.tags].sort();   // works everywhere

// ── Remove a key entirely ───────────────────────────────────────────────────
const { email: _removed, ...withoutEmail } = profile;
// withoutEmail: Omit<UserProfile, "email">, still readonly on the remaining keys
```

For deep nesting, hand-written spreads become painful. A small typed helper covers 90% of cases:

```ts
// Type-safe single-level updater that preserves readonly on the result:
function updateIn<T, K extends keyof T>(source: T, key: K, updater: (current: T[K]) => T[K]): T {
  return { ...source, [key]: updater(source[key]) };
}

const updated = updateIn(profile, "preferences", (prefs) => ({ ...prefs, timezone: "Europe/London" }));
// updated: UserProfile — fully typed, no casts
```

Beyond three levels of nesting, use a library (`immer`'s `produce` gives you mutable-looking syntax over a `Draft<T>` and returns a frozen result) — but be aware `immer` adds a proxy layer and real runtime cost, so keep it out of per-request hot paths.

---

## Example 1 — basic

```ts
// A readonly application config loaded once at boot and shared everywhere.

// ── 1. The shape, with readonly at every level we care about ────────────────

interface DatabaseConfig {
  readonly host:            string;
  readonly port:            number;
  readonly database:        string;
  readonly poolMin:         number;
  readonly poolMax:         number;
  readonly statementTimeout: number;
}

interface HttpConfig {
  readonly port:           number;
  readonly bodyLimitBytes: number;
  readonly corsOrigins:    readonly string[];   // readonly ARRAY — no push/sort
}

interface AppConfig {
  readonly environment: "development" | "staging" | "production";
  readonly http:        HttpConfig;
  readonly database:    DatabaseConfig;
  readonly authSecret:  string;
}

// ── 2. Build it once from the environment ───────────────────────────────────

function loadConfig(): AppConfig {
  const environment = (process.env.NODE_ENV ?? "development") as AppConfig["environment"];

  return {
    environment,
    http: {
      port:           Number(process.env.PORT ?? 3000),
      bodyLimitBytes: 1_048_576,
      corsOrigins:    (process.env.CORS_ORIGINS ?? "").split(",").filter(Boolean),
    },
    database: {
      host:             process.env.DB_HOST ?? "localhost",
      port:             Number(process.env.DB_PORT ?? 5432),
      database:         process.env.DB_NAME ?? "app",
      poolMin:          2,
      poolMax:          Number(process.env.DB_POOL_MAX ?? 20),
      statementTimeout: 10_000,
    },
    authSecret: process.env.AUTH_SECRET ?? "dev-secret-do-not-use-in-prod",
  };
}

// Freeze at runtime too — this object is created exactly once, so the cost is nil:
export const appConfig: AppConfig = Object.freeze(loadConfig());

// ── 3. Consumers can read but never write ───────────────────────────────────

function buildConnectionString(config: AppConfig): string {
  const { host, port, database } = config.database;   // ✅ destructuring reads fine
  return `postgres://${host}:${port}/${database}`;
}

function isOriginAllowed(config: AppConfig, origin: string): boolean {
  return config.http.corsOrigins.includes(origin);    // ✅ includes is a read method
}

function listOriginsAlphabetically(config: AppConfig): string[] {
  // ❌ config.http.corsOrigins.sort();   — sort does not exist on readonly string[]
  return [...config.http.corsOrigins].sort();          // ✅ copy, then sort
}

// ── 4. The mistakes the compiler now catches for you ────────────────────────

function badTestSetup(config: AppConfig): void {
  config.database.poolMax = 1;
  // ❌ Cannot assign to 'poolMax' because it is a read-only property.

  config.http.corsOrigins.push("http://localhost:5173");
  // ❌ Property 'push' does not exist on type 'readonly string[]'.
}

// ── 5. Overriding for tests — derive a NEW config, never mutate the shared one ──

function withOverrides(base: AppConfig, overrides: {
  readonly database?: Partial<DatabaseConfig>;
  readonly http?:     Partial<HttpConfig>;
}): AppConfig {
  return {
    ...base,
    database: { ...base.database, ...overrides.database },
    http:     { ...base.http,     ...overrides.http },
  };
}

const testConfig = withOverrides(appConfig, { database: { poolMax: 1 } });
// testConfig.database.poolMax === 1, appConfig.database.poolMax still === 20 ✅
```

---

## Example 2 — real world backend use case

```ts
// An in-memory entity cache + immutable request context pipeline.
// This is where readonly earns its keep on a real Node service.

// ═══════════════════════════════════════════════════════════════════════════
// Part 1 — A cache that safely hands out its own references
// ═══════════════════════════════════════════════════════════════════════════

interface UserRecord {
  readonly userId:       number;
  readonly email:        string;
  readonly displayName:  string;
  readonly passwordHash: string;
  readonly roles:        readonly UserRole[];
  readonly createdAt:    Date;
  readonly updatedAt:    Date;
}

const USER_ROLES = ["admin", "editor", "viewer", "billing"] as const;
type UserRole = typeof USER_ROLES[number];   // "admin" | "editor" | "viewer" | "billing"

interface CacheEntry<T> {
  readonly value:     T;
  readonly cachedAt:  number;
  readonly expiresAt: number;
}

class UserCache {
  // Internal storage is mutable — this class owns it:
  private readonly entries = new Map<number, CacheEntry<UserRecord>>();

  constructor(private readonly ttlMs: number = 60_000) {}

  // Returns the CACHED REFERENCE with no clone. Safe, because UserRecord is
  // readonly at every level a caller could reach.
  get(userId: number): UserRecord | null {
    const entry = this.entries.get(userId);
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      this.entries.delete(userId);
      return null;
    }
    return entry.value;
  }

  set(user: UserRecord): void {
    const now = Date.now();
    this.entries.set(user.userId, { value: user, cachedAt: now, expiresAt: now + this.ttlMs });
  }

  // Expose a read-only view for diagnostics — no copy, no write access:
  inspect(): ReadonlyMap<number, CacheEntry<UserRecord>> {
    return this.entries;
  }

  invalidate(userId: number): void {
    this.entries.delete(userId);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// Part 2 — Immutable updates on cached entities
// ═══════════════════════════════════════════════════════════════════════════

class UserRepository {
  constructor(
    private readonly db:    DatabaseClient,
    private readonly cache: UserCache,
  ) {}

  async findById(userId: number): Promise<UserRecord | null> {
    const cached = this.cache.get(userId);
    if (cached) return cached;               // no defensive clone needed

    const row = await this.db.queryOne<UserRecord>(
      "SELECT id AS \"userId\", email, display_name AS \"displayName\", password_hash AS \"passwordHash\", roles, created_at AS \"createdAt\", updated_at AS \"updatedAt\" FROM users WHERE id = $1",
      [userId],
    );
    if (!row) return null;
    this.cache.set(row);
    return row;
  }

  // Updates DERIVE a new record — the cached one is never touched:
  async updateDisplayName(userId: number, displayName: string): Promise<UserRecord> {
    const existing = await this.findById(userId);
    if (!existing) throw new Error(`User ${userId} not found`);

    // ❌ existing.displayName = displayName;   — Cannot assign, read-only property
    const updated: UserRecord = { ...existing, displayName, updatedAt: new Date() };

    await this.db.query("UPDATE users SET display_name = $1, updated_at = $2 WHERE id = $3",
      [updated.displayName, updated.updatedAt, userId]);

    this.cache.set(updated);   // replace the cache entry with the new object
    return updated;
  }

  async grantRole(userId: number, role: UserRole): Promise<UserRecord> {
    const existing = await this.findById(userId);
    if (!existing) throw new Error(`User ${userId} not found`);
    if (existing.roles.includes(role)) return existing;   // no-op, same reference

    // ❌ existing.roles.push(role);   — push does not exist on readonly UserRole[]
    const updated: UserRecord = {
      ...existing,
      roles:     [...existing.roles, role],
      updatedAt: new Date(),
    };

    await this.db.query("UPDATE users SET roles = $1, updated_at = $2 WHERE id = $3",
      [updated.roles, updated.updatedAt, userId]);

    this.cache.set(updated);
    return updated;
  }

  async revokeRole(userId: number, role: UserRole): Promise<UserRecord> {
    const existing = await this.findById(userId);
    if (!existing) throw new Error(`User ${userId} not found`);

    const updated: UserRecord = {
      ...existing,
      roles:     existing.roles.filter((r) => r !== role),
      updatedAt: new Date(),
    };

    await this.db.query("UPDATE users SET roles = $1, updated_at = $2 WHERE id = $3",
      [updated.roles, updated.updatedAt, userId]);

    this.cache.set(updated);
    return updated;
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// Part 3 — An immutable request context threaded through middleware
// ═══════════════════════════════════════════════════════════════════════════

interface RequestContext {
  readonly requestId:  string;
  readonly startedAt:  number;
  readonly ipAddress:  string;
  readonly userAgent:  string;
  readonly authToken:  string | null;
  readonly actor:      UserRecord | null;      // populated by the auth middleware
  readonly traceTags:  readonly string[];
}

// Middleware never mutates the context — it RETURNS a new one.
// The signature makes tampering impossible: you cannot secretly rewrite actor.
type ContextMiddleware = (ctx: RequestContext) => Promise<RequestContext>;

const attachRequestId: ContextMiddleware = async (ctx) => ({
  ...ctx,
  requestId: ctx.requestId || crypto.randomUUID(),
});

const authenticate = (users: UserRepository): ContextMiddleware => async (ctx) => {
  if (!ctx.authToken) return ctx;                             // unchanged, same reference
  const claims = verifyJwt(ctx.authToken);
  if (!claims) return { ...ctx, traceTags: [...ctx.traceTags, "auth:invalid"] };

  const actor = await users.findById(claims.userId);
  return { ...ctx, actor, traceTags: [...ctx.traceTags, "auth:ok"] };
};

const tagInternalTraffic: ContextMiddleware = async (ctx) =>
  ctx.ipAddress.startsWith("10.")
    ? { ...ctx, traceTags: [...ctx.traceTags, "internal"] }
    : ctx;

async function runPipeline(initial: RequestContext, middleware: readonly ContextMiddleware[]): Promise<RequestContext> {
  let ctx = initial;                     // the BINDING is mutable; the VALUES are not
  for (const mw of middleware) ctx = await mw(ctx);
  return ctx;
}

// ═══════════════════════════════════════════════════════════════════════════
// Part 4 — Authorisation reads a context it knows nobody downstream rewrote
// ═══════════════════════════════════════════════════════════════════════════

const ROLE_PERMISSIONS = {
  admin:   ["user:read", "user:write", "billing:read", "billing:write"],
  editor:  ["user:read", "user:write"],
  viewer:  ["user:read"],
  billing: ["billing:read", "billing:write"],
} as const satisfies Record<UserRole, readonly string[]>;
//  ↑ `as const satisfies` — deep readonly AND checked against the expected shape

type Permission = typeof ROLE_PERMISSIONS[UserRole][number];
// = "user:read" | "user:write" | "billing:read" | "billing:write"

function permissionsFor(roles: readonly UserRole[]): ReadonlySet<Permission> {
  const granted = new Set<Permission>();                 // mutable while building
  for (const role of roles) {
    for (const permission of ROLE_PERMISSIONS[role]) granted.add(permission);
  }
  return granted;                                        // exposed as ReadonlySet
}

function requirePermission(ctx: RequestContext, permission: Permission): void {
  if (!ctx.actor) throw new Error("UNAUTHENTICATED");
  if (!permissionsFor(ctx.actor.roles).has(permission)) throw new Error("FORBIDDEN");
}

// ── Declarations used above ─────────────────────────────────────────────────
declare const crypto: { randomUUID(): string };
declare function verifyJwt(token: string): { userId: number } | null;
interface DatabaseClient {
  query(sql: string, params: readonly unknown[]): Promise<void>;
  queryOne<T>(sql: string, params: readonly unknown[]): Promise<T | null>;
}
```

---

## Going deeper

### `readonly` is erased — there is no runtime enforcement

Compile this:

```ts
interface Config { readonly port: number }
const config: Config = { port: 3000 };
```

The emitted JavaScript is:

```js
const config = { port: 3000 };
```

That's it. No `Object.freeze`, no getter, no proxy. Consequences:

```ts
const cfg: Config = { port: 3000 };

// 1. Any cast defeats it completely:
(cfg as { port: number }).port = 9999;      // ✅ compiles, ✅ mutates
(cfg as any).port = 9999;                   // ✅ compiles, ✅ mutates

// 2. Untyped JS callers ignore it entirely — a CommonJS require() from a
//    plain .js file sees a normal object and will happily write to it.

// 3. JSON.parse returns `any`, so readonly is not applied to parsed input
//    unless you assert or validate it into the readonly type.
const parsed: Config = JSON.parse(rawBody);  // no runtime guarantee whatsoever
```

If you need a *runtime* guarantee — because you're handing an object to plugin code, or exposing it across a package boundary you don't control — you need `Object.freeze` (shallow) or `deepFreeze` (recursive), and you should accept the traversal cost.

### The object-property assignability hole

This surprises people and it is worth internalising, because it means object-level `readonly` cannot be relied on as a security property:

```ts
interface FrozenUser  { readonly userId: number; readonly email: string }
interface MutableUser {          userId: number;          email: string }

declare const frozen: FrozenUser;

const escaped: MutableUser = frozen;   // ✅ COMPILES — readonly is not checked here
escaped.email = "attacker@example.com";
frozen.email;                          // → "attacker@example.com" — same object!
```

Why does TypeScript allow this? Because readonly-ness on properties is intentionally excluded from structural-assignability comparison. Including it would break enormous amounts of real code (e.g. passing a `Readonly<T>` to a library typed with `T`) with no way to opt out. Arrays are the exception — `readonly T[]` and `T[]` are genuinely *different types* with different members, so the member check naturally fails.

Practical takeaway: `readonly` object properties catch **accidental** mutation by an honest developer, which is 99% of real bugs. They do not catch **deliberate** circumvention. Use `readonly T[]` where you want a hard stop, and `Object.freeze` where you need a runtime one.

### `readonly` in mapped types: the `+`/`-` modifiers

Mapped types can add or remove `readonly` explicitly:

```ts
type MakeReadonly<T>   = { +readonly [K in keyof T]: T[K] };   // '+' is the default, usually omitted
type RemoveReadonly<T> = { -readonly [K in keyof T]: T[K] };   // strips readonly

// Combine with the optionality modifiers for a "fully relaxed" type:
type Draft<T> = { -readonly [K in keyof T]?: T[K] };           // mutable AND optional

interface OrderRecord {
  readonly orderId:    string;
  readonly userId:     number;
  readonly totalCents: number;
  readonly status:     "pending" | "paid" | "shipped";
}

type OrderDraft = Draft<OrderRecord>;
// { orderId?: string; userId?: number; totalCents?: number; status?: ... }
// ← exactly what a builder needs internally before producing the frozen record
```

`Partial<T>`, notably, **preserves** readonly (it only adds `?`), and `Required<T>` preserves it too. Only an explicit `-readonly` removes it.

### `as const` limitations

`as const` only applies to **literal expressions**. It cannot be used on anything computed:

```ts
const ok1 = { level: "debug" } as const;               // ✅ object literal
const ok2 = [1, 2, 3] as const;                        // ✅ array literal
const ok3 = "admin" as const;                          // ✅ string literal

const roles = ["admin", "user"];
const bad1 = roles as const;
// ❌ A 'const' assertion can only be applied to references to enum members,
//    or string, number, boolean, array, or object literals.

const bad2 = { count: 1 + 1 } as const;                // ✅ actually allowed — 2 is a literal result
const bad3 = { at: new Date() } as const;
// ✅ compiles, but `at` is just `Date` — as const cannot deepen a class instance
```

Also: `as const` on an array produces a **tuple with a fixed length**, which sometimes causes surprising errors:

```ts
const DEFAULTS = ["GET", "POST"] as const;   // readonly ["GET", "POST"] — length 2, fixed

function acceptsStringArray(methods: readonly string[]): void {}
acceptsStringArray(DEFAULTS);                // ✅ tuple is assignable to readonly array

function acceptsExactlyThree(m: readonly [string, string, string]): void {}
acceptsExactlyThree(DEFAULTS);               // ❌ Source has 2 element(s) but target requires 3
```

### `as const satisfies` — the modern best practice

Before TypeScript 4.9, you had to choose: annotate the type (losing literal narrowness) or use `as const` (gaining narrowness but losing shape checking). `satisfies` gives you both:

```ts
type RetryPolicy = { readonly maxAttempts: number; readonly backoffMs: readonly number[] };

// ❌ Annotation: shape is checked, but literal types are lost
const policyA: RetryPolicy = { maxAttempts: 3, backoffMs: [100, 500, 2000] };
policyA.maxAttempts;                 // number — not 3

// ❌ as const alone: literals kept, but a typo in a key is NOT caught
const policyB = { maxAttempts: 3, backofMs: [100] } as const;   // typo compiles fine

// ✅ as const satisfies: literals kept AND the shape is validated
const policyC = { maxAttempts: 3, backoffMs: [100, 500, 2000] } as const satisfies RetryPolicy;
policyC.maxAttempts;                 // 3 — the literal type survives
policyC.backoffMs;                   // readonly [100, 500, 2000]
// A typo'd key here → compile error, because it fails the `satisfies` check.
```

Use `as const satisfies SomeShape` for every constant lookup table on a backend. It is strictly better than either alternative.

### The readonly-array parameter rule, and why it's free

Widening a parameter from `T[]` to `readonly T[]`:

- Accepts **strictly more** arguments (every `T[]` is a valid `readonly T[]`).
- Costs zero runtime.
- Breaks only if the body actually mutates — in which case the compiler tells you, and you should have copied anyway.

The one friction point is passing that array on to a *library* function typed with `T[]`:

```ts
declare function legacySort(values: string[]): void;   // typed with a mutable array

function process(tags: readonly string[]): void {
  legacySort(tags);          // ❌ readonly string[] not assignable to string[]
  legacySort([...tags]);     // ✅ copy at the boundary — explicit and correct
  legacySort(tags as string[]);  // ⚠️ compiles, but you just lied; legacySort may mutate
}
```

Copy at the boundary. The cast is a lie and will eventually bite.

### Performance: readonly types are free, freezing is not

| Mechanism | Compile cost | Runtime cost | Actually prevents mutation at runtime |
|---|---|---|---|
| `readonly` property | ~0 | **0** — fully erased | No |
| `readonly T[]` | ~0 | **0** — fully erased | No |
| `as const` | ~0 | **0** — fully erased | No |
| `Readonly<T>` | negligible | **0** | No |
| `DeepReadonly<T>` | can be **significant** on big types | 0 | No |
| `Object.freeze` | ~0 | one shallow pass + possible V8 deopt | Yes, one level |
| `deepFreeze` | ~0 | full recursive traversal | Yes, all levels |
| `immer` `produce` | negligible | proxy allocation + copy-on-write | Yes (result is frozen in dev) |

Spread-based immutable updates allocate a new object per level of the update path. For a request-scoped context updated 5 times per request at 10k rps, that is 50k small short-lived allocations per second — well within what V8's young-generation collector handles trivially. For a hot loop mutating a 100k-element array per iteration, immutable updates are a real problem and you should use a local mutable buffer and only expose it as `readonly` at the end.

### `Readonly<T>` on arrays and functions — the trap

```ts
type A = Readonly<string[]>;
// = readonly string[]  ← works, TypeScript special-cases arrays

type F = Readonly<(userId: number) => string>;
// = {}  ← the call signature is DESTROYED; functions have no own enumerable keys
// This is why DeepReadonly needs the explicit function branch.
```

### When immutability is worth the friction on a backend

**Almost always worth it:**
- Module-level configuration and constant lookup tables. (`as const satisfies`, plus `Object.freeze` at boot.)
- Anything returned from a cache or a shared registry.
- Function parameters that are arrays and are not mutated. (Free, pure upside.)
- Request context / auth claims threaded through middleware.
- Domain events, audit records, and anything you also serialise to a log or a queue.
- Public API surfaces of a module — return `ReadonlyMap` / `readonly T[]` from getters.

**Often not worth it:**
- Local variables inside a function body. Nobody else can see them; `const` on the binding is enough.
- Accumulator objects and arrays being built in a loop. Build mutably, expose readonly.
- Large hot-path buffers, streams, and byte arrays where copying is the actual cost centre.
- ORM entity instances that the ORM itself expects to mutate for change tracking.

**The heuristic:** apply `readonly` at **boundaries** — where data crosses from one module, layer, or request scope to another. Inside a single function you own, mutate freely. The moment the value escapes, it should be readonly.

---

## Common mistakes

### Mistake 1 — Assuming `readonly` / `Readonly<T>` is deep

```ts
// ❌ Wrong — the arrays and nested objects are still fully mutable:
interface ServiceOptions {
  readonly retries:  number;
  readonly hosts:    string[];               // ← array itself is mutable
  readonly headers:  { auth: string };       // ← nested object is mutable
}

declare const opts: ServiceOptions;
opts.hosts.push("evil.example.com");         // ✅ compiles — mutation succeeds
opts.headers.auth = "Bearer stolen";         // ✅ compiles — mutation succeeds

// ✅ Right — readonly at every level you care about:
interface SafeServiceOptions {
  readonly retries: number;
  readonly hosts:   readonly string[];
  readonly headers: { readonly auth: string };
}

declare const safeOpts: SafeServiceOptions;
safeOpts.hosts.push("evil.example.com");     // ❌ push does not exist
safeOpts.headers.auth = "Bearer stolen";     // ❌ read-only property

// ✅ Or, for a type you don't control, apply DeepReadonly:
type DeeplySafeOptions = DeepReadonly<ServiceOptions>;
```

### Mistake 2 — Typing non-mutating parameters as mutable arrays

```ts
// ❌ Wrong — rejects every readonly caller for no reason:
function calculateOrderTotal(items: OrderItem[]): number {
  return items.reduce((sum, item) => sum + item.priceCents * item.quantity, 0);
}

const order: { readonly items: readonly OrderItem[] } = { items: [] };
calculateOrderTotal(order.items);
// ❌ Argument of type 'readonly OrderItem[]' is not assignable to 'OrderItem[]'
// You're now forced to write calculateOrderTotal([...order.items]) — a pointless copy.

// ✅ Right — widen the parameter. Accepts both, documents that you don't mutate:
function calculateOrderTotalFixed(items: readonly OrderItem[]): number {
  return items.reduce((sum, item) => sum + item.priceCents * item.quantity, 0);
}

calculateOrderTotalFixed(order.items);   // ✅ readonly caller works
calculateOrderTotalFixed([]);            // ✅ mutable caller still works

interface OrderItem { readonly sku: string; readonly priceCents: number; readonly quantity: number }
```

### Mistake 3 — Using `as` to escape readonly instead of copying

```ts
declare function sortInPlace(values: number[]): void;   // library fn that MUTATES

// ❌ Wrong — the cast silences the compiler and reintroduces the exact bug
//    readonly existed to prevent:
function reportAmounts(amounts: readonly number[]): void {
  sortInPlace(amounts as number[]);   // mutates the caller's shared array 💥
}

// Also wrong for the same reason:
function corruptCache(user: Readonly<UserRecord>): void {
  (user as { email: string }).email = "x";   // silently poisons a cached object
}

// ✅ Right — copy at the boundary:
function reportAmountsFixed(amounts: readonly number[]): void {
  const copy = [...amounts];
  sortInPlace(copy);
  console.log(copy);
}

// ✅ Or use the non-mutating ES2023 method and skip the copy entirely:
function reportAmountsModern(amounts: readonly number[]): void {
  console.log(amounts.toSorted((a, b) => a - b));
}
```

### Mistake 4 — Forgetting that `readonly` disappears at runtime

```ts
// ❌ Wrong — treating readonly as a security boundary for untrusted input:
interface WebhookPayload {
  readonly eventId: string;
  readonly amountCents: number;
}

app.post("/webhook", (req, res) => {
  const payload = req.body as WebhookPayload;   // ← a lie; req.body is `any`
  // readonly gives you ZERO validation here. amountCents could be a string,
  // an object, or absent. Nothing was checked.
  chargeAccount(payload.amountCents);           // 💥 NaN, or worse
});

// ✅ Right — validate at the boundary, THEN the readonly type is meaningful:
function parseWebhookPayload(raw: unknown): WebhookPayload | null {
  if (typeof raw !== "object" || raw === null) return null;
  const candidate = raw as Record<string, unknown>;
  if (typeof candidate.eventId !== "string") return null;
  if (typeof candidate.amountCents !== "number" || !Number.isFinite(candidate.amountCents)) return null;
  return Object.freeze({ eventId: candidate.eventId, amountCents: candidate.amountCents });
}
```

### Mistake 5 — Mutating a nested level while spreading only the top

```ts
interface UserSettings {
  readonly userId: number;
  readonly notifications: { readonly email: boolean; readonly sms: boolean; readonly push: boolean };
}

declare const settings: UserSettings;

// ❌ Wrong — the spread is SHALLOW, so `notifications` is the SAME object reference.
//    This compiles only because the inner object was cast away, and it corrupts
//    the original:
const broken = { ...settings };
(broken.notifications as { email: boolean }).email = false;
settings.notifications.email;   // → false. The "copy" shared the nested object.

// ✅ Right — rebuild every level on the path being changed:
const fixed: UserSettings = {
  ...settings,
  notifications: { ...settings.notifications, email: false },
};
settings.notifications.email;   // → unchanged ✅
```

---

## Practice exercises

### Exercise 1 — easy

Build a readonly rate-limit configuration module for an API gateway.

Requirements:

1. Define `RateLimitRule` with `readonly` on every property: `routePattern: string`, `windowMs: number`, `maxRequests: number`, `appliesToRoles: readonly UserRole[]`.
2. Define a constant `RATE_LIMIT_RULES` as an array of rules using `as const satisfies readonly RateLimitRule[]`, containing at least four rules for realistic routes (`/api/auth/login`, `/api/users`, `/api/reports/export`, `/api/webhooks/stripe`).
3. Derive a `RoutePattern` union type from the constant using `typeof ... [number]["routePattern"]`.
4. Write `findRule(routePattern: string): RateLimitRule | undefined`.
5. Write `rulesForRole(role: UserRole): readonly RateLimitRule[]` — filters the rules, returns a readonly array, and must not mutate the source.
6. Write `strictestRule(rules: readonly RateLimitRule[]): RateLimitRule | null` — returns the rule with the lowest `maxRequests`. It must sort *without* mutating the input array.
7. Prove the protection: add commented-out lines showing three distinct compile errors you get when you try to mutate the config.

```ts
// Write your code here
```

### Exercise 2 — medium

Build an immutable, event-sourced shopping cart.

Requirements:

1. Define `CartItem` (`readonly sku: string`, `readonly quantity: number`, `readonly unitPriceCents: number`) and `Cart` (`readonly cartId: string`, `readonly userId: number`, `readonly items: readonly CartItem[]`, `readonly appliedCouponCode: string | null`, `readonly updatedAt: Date`).
2. Define a discriminated union `CartEvent` with members: `{ type: "item_added"; sku; quantity; unitPriceCents }`, `{ type: "item_removed"; sku }`, `{ type: "quantity_changed"; sku; quantity }`, `{ type: "coupon_applied"; code }`, `{ type: "coupon_removed" }`, `{ type: "cart_cleared" }`.
3. Write `applyEvent(cart: Cart, event: CartEvent): Cart` — a **pure** function. It must never mutate `cart` or any nested value. Every branch returns a newly constructed `Cart`. Handle the case where `item_added` targets an SKU already in the cart (increment quantity immutably). Include an exhaustiveness check with `never`.
4. Write `replayEvents(initial: Cart, events: readonly CartEvent[]): Cart` using `reduce`.
5. Write `cartTotalCents(cart: Cart): number` and `itemCount(cart: Cart): number`, both taking readonly input.
6. Write `deepFreezeCart(cart: Cart): Cart` that freezes the cart, the items array, and each item at runtime — then write a test showing that a `(cart as any).items.push(...)` throws.
7. Write `diffCarts(before: Cart, after: Cart): readonly string[]` returning human-readable change descriptions.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a fully immutable, typed configuration system with layered overrides and deep-readonly guarantees.

Requirements:

1. Implement `DeepReadonly<T>` yourself, correctly handling: primitives, arrays, tuples, `Map`, `Set`, `Date`, functions, and plain objects. Do not use a library.
2. Implement the inverse `DeepMutable<T>` using the `-readonly` modifier recursively.
3. Implement `DeepPartial<T>` (recursively optional) for override layers.
4. Define a realistic `ServerConfig` interface at least three levels deep, covering: `http` (port, host, cors origins array, compression settings object), `database` (primary + an array of read replicas, each with host/port/pool settings), `cache` (redis url, ttl map), `auth` (jwt secret, token TTLs, an array of allowed issuers), `observability` (log level union, sample rate, an array of exporter configs).
5. Implement `mergeConfig(base: DeepReadonly<ServerConfig>, override: DeepPartial<ServerConfig>): DeepReadonly<ServerConfig>` — a **deep, immutable** merge. Objects merge recursively; arrays are replaced wholesale (not concatenated); `undefined` in the override means "keep the base value". It must be fully typed with no `any`.
6. Implement `deepFreeze<T>(value: T): DeepReadonly<T>` handling cycles safely (use a `WeakSet` of visited objects).
7. Build a `ConfigLoader` class that: takes a base config, accepts an ordered list of override layers (defaults → file → environment → runtime flags), computes the final config lazily on first access, deep-freezes it, caches it, and exposes `get(): DeepReadonly<ServerConfig>` plus a typed `select<R>(selector: (c: DeepReadonly<ServerConfig>) => R): R`.
8. Add a `describeOverrides(): readonly string[]` method that reports which paths each layer changed, computed by diffing consecutive layers.
9. Demonstrate with comments: at least five distinct compile errors from attempting mutation at different depths, and one runtime `TypeError` from the deep freeze.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Property level ──────────────────────────────────────────────────────────
interface T { readonly userId: number }            // cannot reassign the property
class C { private readonly repo: Repo }            // assignable only in constructor

// ── Whole-type level ────────────────────────────────────────────────────────
type R = Readonly<UserRecord>;                     // SHALLOW — top level only
type M = { -readonly [K in keyof T]: T[K] };       // remove readonly
type D = { -readonly [K in keyof T]?: T[K] };      // mutable + optional (draft)

// ── Arrays / tuples ─────────────────────────────────────────────────────────
readonly string[]              // shorthand — preferred
ReadonlyArray<string>          // identical
readonly [number, number]      // readonly tuple, fixed length
// mutable → readonly ✅ allowed;  readonly → mutable ❌ error

// ── Collections ─────────────────────────────────────────────────────────────
ReadonlyMap<K, V>              // get/has/size/forEach/keys/values/entries only
ReadonlySet<T>                 // has/size/forEach/keys/values/entries only

// ── as const ────────────────────────────────────────────────────────────────
const ROLES = ["admin", "viewer"] as const;        // readonly ["admin", "viewer"]
type Role = typeof ROLES[number];                  // "admin" | "viewer"
const CFG = { retries: 3 } as const satisfies RetryConfig;   // literals + shape check

// ── Runtime ─────────────────────────────────────────────────────────────────
Object.freeze(obj)             // → Readonly<T>, SHALLOW, throws on write in strict mode
deepFreeze(obj)                // → DeepReadonly<T>, recursive, costs a full traversal

// ── Immutable updates ───────────────────────────────────────────────────────
{ ...obj, key: value }                                // set top-level field
{ ...obj, nested: { ...obj.nested, key: v } }         // set nested field
[...arr, item]                                        // append
arr.filter(x => x.id !== id)                          // remove
arr.map((x, i) => i === idx ? next : x)               // replace at index
arr.with(idx, next)                                   // replace (ES2023)
arr.toSorted(cmp) / arr.toReversed() / arr.toSpliced() // non-mutating (ES2023)
const { removed, ...rest } = obj;                     // remove a key
```

| Construct | Depth | Runtime effect | Blocks `push`/`sort` | Assignable to mutable? |
|---|---|---|---|---|
| `readonly prop` | 1 level | none (erased) | n/a | **yes** (structural hole) |
| `Readonly<T>` | 1 level | none | no | **yes** |
| `readonly T[]` | array only | none | **yes** | **no** |
| `ReadonlyMap` / `ReadonlySet` | collection only | none | **yes** (`set`/`add` gone) | **no** |
| `as const` | **deep** | none | **yes** | no (for arrays) |
| `DeepReadonly<T>` | **deep** | none | **yes** | no (for arrays) |
| `Object.freeze` | 1 level | throws in strict mode | **yes** at runtime | yes |
| `deepFreeze` | **deep** | throws in strict mode | **yes** at runtime | depends on typing |

| Guideline | Why |
|---|---|
| Type non-mutating array params `readonly T[]` | Free, accepts strictly more callers, documents intent |
| Return `ReadonlyMap` / `readonly T[]` from getters | Hand out references with no defensive copy |
| Use `as const satisfies Shape` for lookup tables | Literal types **and** shape validation |
| Keep internals mutable, boundaries readonly | Build fast, expose safely |
| Never `as`-cast away readonly | Copy at the boundary instead |
| `Object.freeze` only at boot, not in hot paths | Runtime cost + possible V8 deopt |

---

## Connected topics

- **35 — Readonly class properties** — the class-specific side of `readonly`, constructor assignment rules, and how it combines with `private`/`protected`.
- **32 — Utility types** — `Readonly<T>`, `Partial<T>`, `Pick`, `Omit`, and how they compose with the `+`/`-readonly` mapped-type modifiers.
- **43 — Mapped types** — the machinery behind `Readonly<T>`, `Mutable<T>`, and your own `DeepReadonly<T>`.
- **11 — Literal types** — why `as const` matters: preventing literal widening is what makes `typeof ROLES[number]` produce a useful union.
- **13 — Arrays and tuples** — the `readonly T[]` vs `T[]` assignability rule and readonly tuple behaviour in depth.
