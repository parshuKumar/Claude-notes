# 17 — Index Signatures

## What is this?

An **index signature** lets you type an object whose property keys are not known in advance — only the *type* of the key and the *type* of the value are known. The syntax is:

```ts
{ [key: string]: ValueType }
```

This tells TypeScript: "this object can have any string key, and every value at those keys will be of `ValueType`." Index signatures are how you type dictionaries, maps, caches, feature-flag objects, metadata bags, HTTP header collections — any structure where the exact property names are decided at runtime.

## Why does it matter?

JavaScript objects are frequently used as key-value stores. Without index signatures you have two bad options: type the object as `object` (loses all value type information) or type it as `any` (disables the type system entirely). Index signatures give you the middle path: you don't know the keys, but you still enforce that every value is the right type. TypeScript will catch you if you store a `number` in a `Record<string, string>` cache, or if you access a value and forget it might be `undefined`.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — plain objects used as dictionaries, no type safety
const userSessions = {};
userSessions["user_123"] = { token: "abc", expiresAt: Date.now() + 3600000 };
userSessions["user_456"] = { token: "def", expiresAt: Date.now() + 3600000 };

// TypeScript has no idea what's in here:
const session = userSessions["user_789"]; // could be undefined — no warning
session.token;                             // crashes at runtime if session is undefined
```

```ts
// TypeScript — index signature types every value
interface SessionStore {
  [userId: string]: {
    token: string;
    expiresAt: number;
  };
}

const userSessions: SessionStore = {};
userSessions["user_123"] = { token: "abc", expiresAt: Date.now() + 3600000 };
userSessions["user_456"] = { token: "def", expiresAt: Date.now() + 3600000 };

const session = userSessions["user_789"]; // type: { token: string; expiresAt: number }
// TypeScript doesn't warn about missing keys by default — but you can enable noUncheckedIndexedAccess
// which makes the type: { token: string; expiresAt: number } | undefined
session.token; // fine without noUncheckedIndexedAccess, requires guard with it
```

---

## Syntax

```ts
// ── STRING key index signature ─────────────────────────────────────────────
interface StringToString {
  [key: string]: string;
}

// ── NUMBER key index signature ─────────────────────────────────────────────
interface NumberToUser {
  [index: number]: User;
}

// ── COMBINING fixed properties + index signature ───────────────────────────
interface ResponseHeaders {
  "content-type": string;       // fixed, required
  "cache-control"?: string;     // fixed, optional
  [headerName: string]: string | undefined;  // dynamic headers
  // Note: fixed properties MUST be compatible with the index signature value type
}

// ── type alias version (identical capability) ─────────────────────────────
type FeatureFlags = {
  [flagName: string]: boolean;
};

// ── Record<K, V> — shorthand for index signatures ─────────────────────────
type FeatureFlags = Record<string, boolean>;           // equivalent
type UserCache = Record<string, User | undefined>;     // safer: undefined on miss
type RouteHandlers = Record<string, RequestHandler>;   // handler per route string
```

---

## How it works — rule by rule

### The key parameter name is just a label

```ts
interface HttpHeaders {
  [headerName: string]: string; // 'headerName' is a label — could be anything
}

// Functionally identical:
interface HttpHeaders {
  [key: string]: string;
  [k: string]: string;
  [name: string]: string;
}
// The label has no effect on the type — it's documentation only
```

### Valid key types: `string`, `number`, `symbol`, template literals

```ts
interface StringKeyed  { [k: string]: unknown }
interface NumberKeyed  { [k: number]: unknown }
interface SymbolKeyed  { [k: symbol]: unknown }

// Template literal key (TypeScript 4.4+):
type EventMap = {
  [K in `on${string}`]: (...args: unknown[]) => void;
};
// Allows: onConnect, onClick, onUserCreated — any string starting with "on"
```

**Important:** a `number` key index signature is actually a subset of `string` key. In JavaScript, `obj[0]` and `obj["0"]` are the same — the number is always converted to a string. TypeScript knows this and enforces that the number-keyed value type must be compatible with the string-keyed value type if both exist.

### Fixed properties + index signature — the compatibility rule

When you combine fixed properties with an index signature, **every fixed property's type must be assignable to the index signature's value type**:

```ts
// ❌ BROKEN — fixed property 'count' is number, but index is string:
interface Broken {
  count: number;
  [key: string]: string;
}
// Error: Property 'count' of type 'number' is not assignable to 'string' index type 'string'

// ✅ FIXED — use a union value type that covers all fixed property types:
interface RequestMetrics {
  requestId: string;        // string ✅ assignable to string | number
  duration: number;         // number ✅ assignable to string | number
  [key: string]: string | number;  // value union covers both fixed types
}

// ✅ ALSO FIXED — use unknown or any (less safe but valid):
interface FlexibleMeta {
  requestId: string;
  [key: string]: unknown;  // unknown covers string and everything else
}
```

### `noUncheckedIndexedAccess` — the safety flag

By default, indexing into `{ [k: string]: string }` returns `string`, not `string | undefined`. That's a lie — the key might not exist. Enable `noUncheckedIndexedAccess` in `tsconfig.json` for honesty:

```json
{
  "compilerOptions": {
    "noUncheckedIndexedAccess": true
  }
}
```

```ts
interface Cache {
  [key: string]: string;
}

const cache: Cache = {};

// Without noUncheckedIndexedAccess:
const value = cache["missing"];   // type: string (LIE — actually undefined at runtime)
value.toUpperCase();               // no error, crashes at runtime

// With noUncheckedIndexedAccess:
const value = cache["missing"];   // type: string | undefined (HONEST)
value.toUpperCase();               // ❌ Error: Object is possibly 'undefined'
value?.toUpperCase();              // ✅ safe access
```

### Index signature vs `Record<K, V>`

`Record<K, V>` is a utility type built on mapped types — it is equivalent to an index signature for most purposes:

```ts
type A = { [key: string]: number };
type B = Record<string, number>;
// A and B are structurally identical — you can use either

// Where Record is better:
type FeatureFlags = Record<string, boolean>;  // shorter, reads naturally
type UserMap = Record<number, User>;          // cleaner than index signature syntax

// Where index signature is better:
interface Config {
  host: string;
  port: number;
  [key: string]: string | number;  // mixing fixed + dynamic requires interface/type literal
}
// Record can't mix fixed properties and dynamic keys in one definition
```

### When to reach for `Map` instead

Index signatures type plain objects as key-value stores. JavaScript's `Map` is often better for runtime dictionaries:

```ts
// Index signature — plain object:
const cache: Record<string, User> = {};
cache["user_123"] = user;
delete cache["user_123"];

// Map — more appropriate when:
// - keys are non-string (numbers, objects, symbols)
// - you need .size, .has(), .delete(), iteration order guarantee
// - you're inserting/deleting frequently
const userMap = new Map<string, User>();
userMap.set("user_123", user);
userMap.has("user_123");
userMap.delete("user_123");
userMap.size;

// Typed Map (no index signature needed):
const sessionStore = new Map<string, { token: string; expiresAt: number }>();
```

---

## Example 1 — basic

```ts
// Feature flags, HTTP headers, and a per-route config store

// ── Feature flags ─────────────────────────────────────────────────────────
type FeatureFlags = Record<string, boolean>;

const flags: FeatureFlags = {
  "new-dashboard": true,
  "beta-checkout": false,
  "ai-recommendations": true,
};

function isEnabled(flags: FeatureFlags, flagName: string): boolean {
  return flags[flagName] ?? false;
}

// ── HTTP headers (mixed fixed + dynamic) ──────────────────────────────────
interface ResponseHeaders {
  "content-type": string;
  "cache-control"?: string;
  [header: string]: string | undefined;
}

function buildJsonHeaders(cacheMaxAge?: number): ResponseHeaders {
  return {
    "content-type": "application/json",
    "cache-control": cacheMaxAge ? `max-age=${cacheMaxAge}` : undefined,
    "x-request-id": crypto.randomUUID(),
  };
}

// ── Validation error map — field name → error message ─────────────────────
type ValidationErrors = Record<string, string>;

function validateCreateUserBody(body: unknown): ValidationErrors {
  const errors: ValidationErrors = {};

  if (!body || typeof body !== "object") {
    errors["body"] = "Request body must be a JSON object";
    return errors;
  }

  const b = body as Record<string, unknown>;

  if (typeof b["email"] !== "string" || !b["email"].includes("@")) {
    errors["email"] = "Must be a valid email address";
  }
  if (typeof b["name"] !== "string" || b["name"].trim().length < 2) {
    errors["name"] = "Name must be at least 2 characters";
  }

  return errors;
}

// ── Per-route middleware config ────────────────────────────────────────────
interface RouteConfig {
  rateLimit: number;     // requests per minute
  requiresAuth: boolean;
  cacheTtl?: number;     // seconds
}

const routeConfigs: Record<string, RouteConfig> = {
  "GET /api/users":        { rateLimit: 100, requiresAuth: true, cacheTtl: 60 },
  "POST /api/users":       { rateLimit: 20,  requiresAuth: true },
  "GET /api/products":     { rateLimit: 200, requiresAuth: false, cacheTtl: 300 },
  "DELETE /api/users/:id": { rateLimit: 10,  requiresAuth: true },
};

function getRouteConfig(method: string, path: string): RouteConfig | undefined {
  return routeConfigs[`${method} ${path}`];
}
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── In-memory cache with typed index signature ─────────────────────────────

interface CacheEntry<T> {
  value: T;
  expiresAt: number;  // Date.now() + ttlMs
  hits: number;
}

interface TypedCache<T> {
  [key: string]: CacheEntry<T> | undefined;  // undefined: honest about missing keys
}

class InMemoryCache<T> {
  private store: TypedCache<T> = {};

  set(key: string, value: T, ttlMs: number): void {
    this.store[key] = {
      value,
      expiresAt: Date.now() + ttlMs,
      hits: 0,
    };
  }

  get(key: string): T | null {
    const entry = this.store[key];
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      delete this.store[key];
      return null;
    }
    entry.hits++;
    return entry.value;
  }

  has(key: string): boolean {
    return this.get(key) !== null;
  }

  delete(key: string): void {
    delete this.store[key];
  }

  stats(): { size: number; keys: string[] } {
    const keys = Object.keys(this.store).filter(k => this.has(k));
    return { size: keys.length, keys };
  }
}

// ── Per-request context bag — dynamic metadata ─────────────────────────────

interface RequestContext {
  requestId: string;
  userId?: number;
  startTime: number;
  [metaKey: string]: unknown;  // dynamic — middleware can attach anything
}

function createRequestContext(): RequestContext {
  return {
    requestId: crypto.randomUUID(),
    startTime: Date.now(),
  };
}

function requestContextMiddleware(req: Request, res: Response, next: NextFunction): void {
  (req as Request & { ctx: RequestContext }).ctx = createRequestContext();
  next();
}

// ── Config registry — dynamic service config by service name ──────────────

interface DatabaseConfig {
  host: string;
  port: number;
  name: string;
}

interface ServiceRegistry {
  databases: Record<string, DatabaseConfig>;   // serviceName → DB config
  endpoints: Record<string, string>;           // serviceName → base URL
  timeouts: Record<string, number>;            // serviceName → ms
}

const registry: ServiceRegistry = {
  databases: {
    users:    { host: "db-users.internal",    port: 5432, name: "users_db" },
    orders:   { host: "db-orders.internal",   port: 5432, name: "orders_db" },
    products: { host: "db-products.internal", port: 5432, name: "products_db" },
  },
  endpoints: {
    "payment-service":      "http://payments.internal:4000",
    "notification-service": "http://notifications.internal:4001",
    "search-service":       "http://search.internal:4002",
  },
  timeouts: {
    "payment-service":      5000,
    "notification-service": 3000,
    "search-service":       2000,
  },
};

function getServiceEndpoint(registry: ServiceRegistry, serviceName: string): string {
  const url = registry.endpoints[serviceName];
  if (!url) throw new Error(`Unknown service: ${serviceName}`);
  return url;
}

// ── Translation / i18n store ───────────────────────────────────────────────

type TranslationKey = string;
type Locale = "en" | "es" | "fr" | "de" | "hi";

type TranslationStore = Record<Locale, Record<TranslationKey, string>>;

const translations: TranslationStore = {
  en: { "welcome": "Welcome", "logout": "Log out", "error.notFound": "Not found" },
  es: { "welcome": "Bienvenido", "logout": "Cerrar sesión", "error.notFound": "No encontrado" },
  fr: { "welcome": "Bienvenue", "logout": "Se déconnecter", "error.notFound": "Introuvable" },
  de: { "welcome": "Willkommen", "logout": "Abmelden", "error.notFound": "Nicht gefunden" },
  hi: { "welcome": "स्वागत है", "logout": "लॉग आउट", "error.notFound": "नहीं मिला" },
};

function t(locale: Locale, key: TranslationKey): string {
  return translations[locale][key] ?? translations["en"][key] ?? key;
}
```

---

## Common mistakes

### Mistake 1 — Fixed property type incompatible with index signature value type

```ts
// ❌ BROKEN — 'count' is number but index signature expects string:
interface Broken {
  requestId: string;
  count: number;
  [key: string]: string;  // number is not assignable to string
}
// Error: Property 'count' of type 'number' is not assignable to 'string' index type

// ✅ FIXED — widen the index signature to a union:
interface RequestMetrics {
  requestId: string;
  count: number;
  [key: string]: string | number;  // covers both fixed property types
}

// ✅ ALSO FINE — use 'unknown' if you want maximum flexibility:
interface RequestMetrics {
  requestId: string;
  count: number;
  [key: string]: unknown;
}
```

### Mistake 2 — Not handling `undefined` on index access (the silent crash)

```ts
const userConfig: Record<string, string> = {
  theme: "dark",
  language: "en",
};

// ❌ DANGEROUS — key might not exist, type says string but runtime is undefined:
const value = userConfig["nonExistent"];
console.log(value.toUpperCase()); // CRASHES: Cannot read properties of undefined

// ✅ SAFE — check before using:
const value = userConfig["nonExistent"];
if (value !== undefined) {
  console.log(value.toUpperCase());
}

// ✅ SAFE — nullish fallback:
const value = userConfig["nonExistent"] ?? "default";

// ✅ BEST — enable noUncheckedIndexedAccess in tsconfig to make TypeScript
// report the type as 'string | undefined' and force you to handle it
```

### Mistake 3 — Using index signature when a union of literal keys is the right fit

```ts
// ❌ WRONG — index signature when keys are known:
interface HttpMethod {
  [method: string]: boolean; // allows "FLARB": true — nonsense
}

// ✅ RIGHT — when keys are finite and known, use a mapped type or explicit properties:
type HttpMethodMap = Record<"GET" | "POST" | "PUT" | "PATCH" | "DELETE", boolean>;

// Or:
interface AllowedMethods {
  GET: boolean;
  POST: boolean;
  PUT: boolean;
  PATCH: boolean;
  DELETE: boolean;
}

// Index signatures are for GENUINELY DYNAMIC keys — if you know all the keys,
// enumerate them explicitly so TypeScript can catch typos and missing keys.
```

---

## Practice exercises

### Exercise 1 — easy

Define these types using index signatures or `Record`:

1. `HttpHeaderMap` — any string key → string value (for HTTP request/response headers)
2. `EnvVariables` — any string key → string value (for `process.env`-style objects)
3. `RateLimitConfig` — maps a string route key (`"POST /api/login"`) to a number (requests per minute)
4. `PermissionMap` — maps a string permission name to a boolean (granted or not)

Write a function `hasPermission(permissions: PermissionMap, action: string): boolean` that safely checks if a permission is granted (returns `false` if key doesn't exist).

Write a function `mergeHeaders(base: HttpHeaderMap, overrides: HttpHeaderMap): HttpHeaderMap` that returns a new header map with the override values taking precedence.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed in-memory session store. Define:

```
SessionData:
  - userId: number
  - email: string
  - role: "admin" | "user"
  - createdAt: number   (timestamp)
  - expiresAt: number   (timestamp)
  - [key: string]: unknown  (allow extra middleware-attached data)

SessionStore interface with methods:
  - create(sessionId: string, data: SessionData): void
  - get(sessionId: string): SessionData | null   (null if missing or expired)
  - refresh(sessionId: string, ttlMs: number): boolean  (returns false if session not found)
  - destroy(sessionId: string): void
  - purgeExpired(): number   (returns count of sessions removed)
  - count(): number
```

Implement `InMemorySessionStore` satisfying the interface. Use `Record<string, SessionData | undefined>` as the internal store.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed middleware pipeline registry. Each route can register multiple middlewares. Define:

```ts
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type RoutePath = string;   // e.g. "/api/users/:id"
type RouteKey = `${HttpMethod} ${RoutePath}`;  // template literal key
```

1. Define a `MiddlewareFn` type: `(req: Request, res: Response, next: NextFunction) => void | Promise<void>`
2. Define `RouteMiddlewareRegistry` — maps `RouteKey` → `MiddlewareFn[]`
3. Define a `PipelineRegistry` interface with:
   - `register(method: HttpMethod, path: RoutePath, ...fns: MiddlewareFn[]): void`
   - `get(method: HttpMethod, path: RoutePath): MiddlewareFn[]`
   - `getAll(): RouteMiddlewareRegistry`
   - `has(method: HttpMethod, path: RoutePath): boolean`
   - `remove(method: HttpMethod, path: RoutePath): void`
4. Implement `MiddlewarePipelineRegistry` satisfying `PipelineRegistry`
5. Write 3 concrete `MiddlewareFn` functions: `logRequest`, `requireAuth`, `validateJsonBody`
6. Register them on at least 3 routes and demonstrate `get()` returns the correct pipeline

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `{ [key: string]: V }` | Any string key → value of type V |
| `{ [key: number]: V }` | Any number key → value of type V |
| `Record<string, V>` | Shorthand for `{ [key: string]: V }` |
| `Record<K, V>` where K is a union | Maps each member of K to V |
| `Record<string, V \| undefined>` | Honest — access may return undefined |
| Fixed props + index sig | Fixed prop type must be ⊆ index value type |
| `[key: string]: unknown` | Widest safe index — accepts any value |
| `[K in \`on${string}\`]` | Template literal index — keys matching pattern |

| Rule | Notes |
|------|-------|
| Key label is documentation only | `[key: string]` and `[k: string]` are identical |
| `number` keys convert to `string` at runtime | JS always coerces numeric keys |
| Fixed + index: type compatibility required | Fixed value types must be assignable to index value type |
| Default: no undefined on access | Enable `noUncheckedIndexedAccess` for safety |
| Known keys: don't use index signature | Use literal union or explicit properties instead |
| Dynamic runtime keys: use index sig or `Map` | If inserting/deleting heavily, prefer `Map` |

## Connected topics

- **15 — type vs interface** — `Record<K, V>` is a mapped type (`type` territory); index signatures work in both.
- **09 — Union types** — frequently used as the value type in index signatures.
- **32 — Utility types** — `Record<K, V>`, `Partial<T>`, `Required<T>` all relate to index signatures.
- **40 — Mapped types** — `{ [K in keyof T]: V }` — the generic version of index signatures.
- **43 — Indexed access types** — `T["key"]` pulling values from typed objects.
- **tsconfig — noUncheckedIndexedAccess** — the compiler flag that makes index access honest about `undefined`.
