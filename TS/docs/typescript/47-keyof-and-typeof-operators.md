# 47 — keyof and typeof operators

## What is this?

`keyof` and `typeof` are the two **query operators** of TypeScript's type system. They let you derive types from things you already wrote instead of re-declaring them by hand.

- **`keyof T`** takes a *type* and produces the **union of its property key names** as string (or number/symbol) literal types.
- **`typeof value`** — used **in a type position** — takes a *runtime value* and produces **its inferred type**.

```ts
interface User {
  userId:    number;
  email:     string;
  createdAt: Date;
}

type UserKey = keyof User;          // "userId" | "email" | "createdAt"

const defaultUser = { userId: 0, email: "", createdAt: new Date() };
type DefaultUser = typeof defaultUser;  // { userId: number; email: string; createdAt: Date }
```

They are the bridge between the two worlds TypeScript keeps strictly separate:

- The **value world** — what exists at runtime: variables, functions, objects, classes.
- The **type world** — what exists only at compile time: interfaces, type aliases, generics.

`typeof` moves a *value* into the type world. `keyof` operates entirely inside the type world. Composed as `keyof typeof authConfig`, they turn a plain runtime object into a compile-time union of its keys — no duplication, no drift.

## Why does it matter?

Backend code is full of objects whose keys carry meaning: config maps, error-code tables, permission maps, column allow-lists, event handler registries. In JavaScript you write the object once and then re-type its keys by hand everywhere else — in JSDoc, in validation arrays, in switch statements. The two copies drift apart the first time someone adds a key.

`keyof typeof` kills that entire class of bug:

- Add a key to `ROLE_PERMISSIONS` → every function that accepts a role automatically accepts the new one.
- Remove an error code from `ERROR_CODES` → every call site referencing it becomes a **compile error**, not a 500 at 3am.
- A generic `getField(user, "emial")` typo is caught before the file is saved.
- ORM/query-builder column names are checked against your entity type instead of being loose strings.

Together with **indexed access types** (`T[K]`), these operators give you a *precise* generic getter: `get(user, "createdAt")` returns `Date`, not `unknown`, not `any`.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript ────────────────────────────────────────────────────────────────
// One config object, and then the same key names re-written by hand everywhere.

const ROLE_PERMISSIONS = {
  admin:  ["read", "write", "delete", "manage_users"],
  editor: ["read", "write"],
  viewer: ["read"],
};

// You have to maintain this list by hand — it WILL drift:
const VALID_ROLES = ["admin", "editor", "viewer"];

function hasPermission(role, permission) {
  const perms = ROLE_PERMISSIONS[role];   // undefined for a typo'd role
  return perms.includes(permission);      // 💥 TypeError: Cannot read 'includes' of undefined
}

hasPermission("editer", "write");   // typo — no warning, crashes at runtime
hasPermission("admin", "delet");    // typo — silently returns false, user is denied access

// Later somebody adds a role:
ROLE_PERMISSIONS.auditor = ["read", "export"];
// VALID_ROLES is now stale. The validation middleware rejects every auditor.
// Nothing told you. You find out from a support ticket.

// A generic getter is completely untyped:
function getField(obj, key) {
  return obj[key];   // returns "any" in your head, "undefined" in production
}
const email = getField(user, "emial");  // undefined — no error anywhere
email.toLowerCase();                    // 💥 TypeError
```

```ts
// ── TypeScript ────────────────────────────────────────────────────────────────
// Write the object ONCE. Derive every type from it. Nothing can drift.

const ROLE_PERMISSIONS = {
  admin:  ["read", "write", "delete", "manage_users"],
  editor: ["read", "write"],
  viewer: ["read"],
} as const;                                  // `as const` → literal types, readonly

type Role       = keyof typeof ROLE_PERMISSIONS;              // "admin" | "editor" | "viewer"
type Permission = (typeof ROLE_PERMISSIONS)[Role][number];    // "read" | "write" | "delete" | "manage_users"

function hasPermission(role: Role, permission: Permission): boolean {
  return (ROLE_PERMISSIONS[role] as readonly Permission[]).includes(permission);
}

hasPermission("editer", "write");   // ❌ Compile error: '"editer"' is not assignable to 'Role'
hasPermission("admin", "delet");    // ❌ Compile error: '"delet"' is not assignable to 'Permission'
hasPermission("admin", "delete");   // ✅

// Add a role to the object and `Role` widens automatically — zero other edits:
// auditor: ["read", "export"]  →  Role becomes ..."auditor", Permission gains "export"

// And the generic getter is exact:
function getField<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];                  // return type is T[K] — the ACTUAL property type
}

const email     = getField(user, "email");     // string  ✅
const createdAt = getField(user, "createdAt"); // Date    ✅ — not string, not any
const nope      = getField(user, "emial");     // ❌ Compile error: not a key of User
```

The revelation: **the object literal you already wrote is the single source of truth.** You never restate its keys. The compiler reads them for you.

---

## Syntax

```ts
// ── keyof — union of a type's keys ───────────────────────────────────────────
interface User { userId: number; email: string; isActive: boolean }
type UserKeys = keyof User;                 // "userId" | "email" | "isActive"

// ── typeof — a value's type, used in a TYPE position ─────────────────────────
const authConfig = { issuer: "api.example.com", ttlSeconds: 3600 };
type AuthConfig = typeof authConfig;        // { issuer: string; ttlSeconds: number }

// ── keyof typeof — keys of a runtime object ──────────────────────────────────
type AuthConfigKey = keyof typeof authConfig;   // "issuer" | "ttlSeconds"

// ── Indexed access type — the type AT a key ──────────────────────────────────
type Email     = User["email"];             // string
type IdOrEmail = User["userId" | "email"];  // number | string
type AnyValue  = User[keyof User];          // number | string | boolean

// ── Value union of a const object ────────────────────────────────────────────
const HTTP_STATUS = { OK: 200, NOT_FOUND: 404, SERVER_ERROR: 500 } as const;
type StatusCode = (typeof HTTP_STATUS)[keyof typeof HTTP_STATUS];   // 200 | 404 | 500

// ── Generic getter constrained by keyof ──────────────────────────────────────
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];                          // exact property type flows out
}

// ── Array element type via number index ──────────────────────────────────────
const REGIONS = ["us-east-1", "eu-west-1"] as const;
type Region = (typeof REGIONS)[number];     // "us-east-1" | "eu-west-1"

// ── typeof on a function → its full signature ────────────────────────────────
function createUser(email: string, role: string) { return { email, role }; }
type CreateUserFn = typeof createUser;      // (email: string, role: string) => { email: string; role: string }

// ── typeof on a class → the CONSTRUCTOR type, not the instance ───────────────
class UserRepository { findById(id: number) { /* ... */ } }
type RepoInstance = UserRepository;         // the instance type
type RepoClass    = typeof UserRepository;  // new () => UserRepository (the class itself)
```

---

## How it works — concept by concept

### Concept 1 — Two worlds: values and types

TypeScript keeps **value space** and **type space** in separate namespaces. The same identifier can mean different things depending on which space you're in.

```ts
// A VALUE — exists at runtime, can be logged, passed around:
const requestBody = { userId: 42, email: "dev@example.com" };

// A TYPE — erased at compile time, exists only for the checker:
interface RequestBody { userId: number; email: string }

// You CANNOT use a value where a type is expected:
// let a: requestBody;          // ❌ 'requestBody' refers to a value, but is being used as a type
// You CANNOT use a type where a value is expected:
// console.log(RequestBody);    // ❌ 'RequestBody' only refers to a type

// `typeof` is the ONE-WAY DOOR from value space into type space:
type InferredBody = typeof requestBody;   // ✅ { userId: number; email: string }

let another: InferredBody = { userId: 7, email: "ops@example.com" };  // ✅
```

Note the crucial distinction from JavaScript's runtime `typeof`:

```ts
const authToken = "eyJhbGciOi...";

// JavaScript's typeof — runs at RUNTIME, returns a string:
if (typeof authToken === "string") { /* narrowing, see 40 — Type narrowing */ }
console.log(typeof authToken);            // logs: "string"

// TypeScript's typeof — runs at COMPILE TIME, returns a TYPE:
type AuthToken = typeof authToken;        // "eyJhbGciOi..." (a string literal type — it's a const)
```

Same keyword, two completely different operators, disambiguated purely by position. If it's after a `:` or inside a `type` alias / generic argument, it's the type-level one.

### Concept 2 — What `keyof` actually produces

`keyof T` is a **union of literal key types**, not an array and not a string.

```ts
interface AuditLog {
  logId:     string;
  actorId:   number;
  action:    string;
  createdAt: Date;
}

type AuditLogKey = keyof AuditLog;   // "logId" | "actorId" | "action" | "createdAt"

// It behaves exactly like any other union — you can Exclude/Extract from it:
type MutableKey = Exclude<keyof AuditLog, "logId" | "createdAt">;  // "actorId" | "action"

// It's a type, so it can constrain a generic:
function pluck<K extends keyof AuditLog>(logs: AuditLog[], key: K): AuditLog[K][] {
  return logs.map((log) => log[key]);
}

const actorIds = pluck(auditLogs, "actorId");    // number[]  ✅
const dates    = pluck(auditLogs, "createdAt");  // Date[]    ✅
// pluck(auditLogs, "actor");                    // ❌ not a key of AuditLog
```

Numeric and symbol keys survive as their own literal types:

```ts
interface MixedKeys {
  0:              string;   // numeric key
  name:           string;   // string key
  [Symbol.iterator]: () => Iterator<string>;
}
type Mixed = keyof MixedKeys;   // 0 | "name" | typeof Symbol.iterator
```

And `keyof` on an empty object type gives `never` — a union of zero members:

```ts
type NoKeys = keyof {};         // never
type AnyKey = keyof any;        // string | number | symbol
type UnknownKey = keyof unknown; // never
```

### Concept 3 — Indexed access types: `T[K]`

Indexed access reads the **type stored at a key**. The syntax mirrors runtime property access, but with types.

```ts
interface Order {
  orderId:    string;
  userId:     number;
  items:      { sku: string; quantity: number }[];
  totalCents: number;
  status:     "pending" | "paid" | "shipped";
}

type OrderId     = Order["orderId"];    // string
type OrderStatus = Order["status"];     // "pending" | "paid" | "shipped"
type OrderItems  = Order["items"];      // { sku: string; quantity: number }[]

// Chain it — index into the array element, then into its property:
type OrderItem = Order["items"][number];       // { sku: string; quantity: number }
type Sku       = Order["items"][number]["sku"]; // string

// Index with a UNION of keys → union of the value types:
type IdFields = Order["orderId" | "userId"];   // string | number

// Index with `keyof` → union of ALL value types:
type AnyOrderValue = Order[keyof Order];
// string | number | { sku: string; quantity: number }[] | "pending" | "paid" | "shipped"
```

The key must actually exist — indexed access is checked:

```ts
// type Nope = Order["ordreId"];   // ❌ Property 'ordreId' does not exist on type 'Order'
```

This matters most when you want a nested type without exporting it separately. Instead of splitting `OrderItem` into its own interface just so another module can reference it, `Order["items"][number]` reaches in.

### Concept 4 — `keyof typeof` on const objects

The killer combination. You have a runtime object; you want its keys as a type.

```ts
// A lookup table you actually use at runtime:
const ERROR_CODES = {
  INVALID_INPUT:    { status: 400, message: "Invalid request body" },
  UNAUTHENTICATED:  { status: 401, message: "Missing or invalid auth token" },
  FORBIDDEN:        { status: 403, message: "Insufficient permissions" },
  NOT_FOUND:        { status: 404, message: "Resource not found" },
  CONFLICT:         { status: 409, message: "Resource already exists" },
  RATE_LIMITED:     { status: 429, message: "Too many requests" },
  INTERNAL:         { status: 500, message: "Internal server error" },
} as const;

// typeof ERROR_CODES → the object's type
// keyof (that)       → the union of its keys
type ErrorCode = keyof typeof ERROR_CODES;
// "INVALID_INPUT" | "UNAUTHENTICATED" | "FORBIDDEN" | "NOT_FOUND"
// | "CONFLICT" | "RATE_LIMITED" | "INTERNAL"

// Now every function that takes an error code is exhaustively typed:
function toHttpError(code: ErrorCode): { status: number; message: string; code: ErrorCode } {
  const entry = ERROR_CODES[code];      // ✅ no possible undefined — code is a real key
  return { status: entry.status, message: entry.message, code };
}

toHttpError("NOT_FOUND");   // ✅
// toHttpError("NOTFOUND"); // ❌ Argument of type '"NOTFOUND"' is not assignable to 'ErrorCode'
```

Adding `RATE_LIMITED` to the object automatically widened `ErrorCode`. Deleting `CONFLICT` would break every call site that used it — which is exactly what you want.

### Concept 5 — `typeof obj[keyof typeof obj]` — the value union

`keyof typeof` gives you the **keys**. To get the **values** as a union, index the type by its own keys.

```ts
const JOB_QUEUE = {
  EMAIL:        "queue:email",
  WEBHOOK:      "queue:webhook",
  REPORT_GEN:   "queue:report",
  IMAGE_RESIZE: "queue:image",
} as const;

type QueueName  = keyof typeof JOB_QUEUE;                     // "EMAIL" | "WEBHOOK" | ...
type QueueTopic = (typeof JOB_QUEUE)[keyof typeof JOB_QUEUE]; // "queue:email" | "queue:webhook" | ...

// Read it inside-out:
//   typeof JOB_QUEUE                        → { readonly EMAIL: "queue:email"; ... }
//   keyof typeof JOB_QUEUE                  → "EMAIL" | "WEBHOOK" | ...
//   (typeof JOB_QUEUE)[keyof typeof JOB_QUEUE] → "queue:email" | "queue:webhook" | ...

function publish(topic: QueueTopic, payload: unknown): void {
  redis.lpush(topic, JSON.stringify(payload));
}

publish(JOB_QUEUE.EMAIL, { userId: 42 });   // ✅
publish("queue:email", { userId: 42 });     // ✅ — the literal is in the union
// publish("queue:emails", {});             // ❌ typo caught
```

**`as const` is mandatory here.** Without it the values widen to `string` and the union collapses:

```ts
const LOOSE = { EMAIL: "queue:email", WEBHOOK: "queue:webhook" };      // no as const
type LooseTopic = (typeof LOOSE)[keyof typeof LOOSE];                  // string  ← useless

const STRICT = { EMAIL: "queue:email", WEBHOOK: "queue:webhook" } as const;
type StrictTopic = (typeof STRICT)[keyof typeof STRICT];               // "queue:email" | "queue:webhook"
```

This is the TypeScript-native replacement for enums — see **28 — Enums** for why const objects are usually preferable.

### Concept 6 — Generic getters: `get<T, K extends keyof T>`

The constraint `K extends keyof T` is what makes a generic accessor safe *and* precise.

```ts
// The canonical shape:
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

interface UserRecord {
  userId:    number;
  email:     string;
  isActive:  boolean;
  lastLogin: Date | null;
}

const userRecord: UserRecord = {
  userId: 42, email: "dev@example.com", isActive: true, lastLogin: null,
};

const id       = getProperty(userRecord, "userId");    // number
const active   = getProperty(userRecord, "isActive");  // boolean
const login    = getProperty(userRecord, "lastLogin"); // Date | null
// getProperty(userRecord, "userid");                  // ❌ wrong case → compile error

// The setter counterpart — value type must match the key's type:
function setProperty<T, K extends keyof T>(obj: T, key: K, value: T[K]): void {
  obj[key] = value;
}

setProperty(userRecord, "email", "new@example.com");  // ✅
// setProperty(userRecord, "email", 123);             // ❌ number not assignable to string
// setProperty(userRecord, "userId", "42");           // ❌ string not assignable to number

// Picking several keys at once:
function pick<T, K extends keyof T>(obj: T, keys: readonly K[]): Pick<T, K> {
  const out = {} as Pick<T, K>;
  for (const key of keys) out[key] = obj[key];
  return out;
}

const publicUser = pick(userRecord, ["userId", "email"] as const);
// { userId: number; email: string }  — isActive and lastLogin are gone from the TYPE too
```

Why two type parameters and not one? Because `K` must be inferred *per call* as the narrow literal (`"email"`), not widened to the whole `keyof T` union. If you wrote `key: keyof T`, the return type would be `T[keyof T]` — a union of every property type, which is almost as useless as `any`.

```ts
// ❌ Loses precision:
function getWide<T>(obj: T, key: keyof T): T[keyof T] { return obj[key]; }
const wide = getWide(userRecord, "email");   // number | string | boolean | Date | null 😞

// ✅ Keeps precision:
const narrow = getProperty(userRecord, "email");  // string 🎉
```

### Concept 7 — `keyof` on index signatures, arrays, and unions

`keyof` behaves differently for structures that aren't plain fixed-shape objects.

```ts
// ── Index signatures: keyof gives the index type, not literal keys ────────────
interface SessionStore { [sessionId: string]: { userId: number; expiresAt: Date } }
type SessionKey = keyof SessionStore;      // string | number
// ⚠ `number` is included: JS coerces obj[1] to obj["1"], so numeric keys are valid.

interface PortMap { [port: number]: string }
type PortKey = keyof PortMap;              // number

// A Record with a literal key union keeps literals:
type FeatureFlags = Record<"newCheckout" | "betaSearch", boolean>;
type FlagKey = keyof FeatureFlags;         // "newCheckout" | "betaSearch"

// ── Arrays: keyof includes ALL array members (methods too) ───────────────────
type ArrayKeys = keyof string[];
// number | "length" | "toString" | "push" | "pop" | "concat" | "join" | ... (everything)
// So `keyof someArray` is almost never what you want.

// To get the ELEMENT type, index by number instead:
const AUDIT_ACTIONS = ["create", "update", "delete"] as const;
type AuditAction = (typeof AUDIT_ACTIONS)[number];   // "create" | "update" | "delete"  ✅

// For tuples, numeric literals give you positional types:
type DbTuple = [host: string, port: number, ssl: boolean];
type Host = DbTuple[0];        // string
type Port = DbTuple[1];        // number
type Any  = DbTuple[number];   // string | number | boolean

// ── Unions: keyof distributes as an INTERSECTION (only shared keys) ──────────
interface EmailNotification { channel: "email"; to: string; subject: string }
interface SmsNotification   { channel: "sms";   to: string; message: string }
type Notification = EmailNotification | SmsNotification;

type NotificationKey = keyof Notification;    // "channel" | "to"   ← ONLY the common keys
// This is sound: given an unknown member, only shared keys are guaranteed to exist.

// To get the union of ALL keys across members, distribute manually:
type AllKeys<T> = T extends unknown ? keyof T : never;
type EveryNotificationKey = AllKeys<Notification>;
// "channel" | "to" | "subject" | "message"

// ── Intersections: keyof distributes as a UNION ──────────────────────────────
type WithTimestamps = { createdAt: Date; updatedAt: Date };
type Entity = UserRecord & WithTimestamps;
type EntityKey = keyof Entity;   // "userId" | "email" | "isActive" | "lastLogin" | "createdAt" | "updatedAt"
```

### Concept 8 — `typeof` on functions, classes, and modules

`typeof` isn't limited to object literals.

```ts
// ── Functions ────────────────────────────────────────────────────────────────
async function fetchUserById(userId: number, includeDeleted = false) {
  return { userId, email: "x@y.com", deletedAt: null as Date | null };
}

type FetchUserFn     = typeof fetchUserById;                   // (userId: number, includeDeleted?: boolean) => Promise<{...}>
type FetchUserParams = Parameters<typeof fetchUserById>;       // [userId: number, includeDeleted?: boolean]
type FetchUserResult = Awaited<ReturnType<typeof fetchUserById>>; // { userId: number; email: string; deletedAt: Date | null }

// Extremely useful for typing a mock/decorator without restating the signature:
const cachedFetchUser: typeof fetchUserById = async (userId, includeDeleted) => {
  const hit = cache.get(userId);
  if (hit) return hit;
  return fetchUserById(userId, includeDeleted);
};

// ── Classes: typeof Class is the CONSTRUCTOR side ────────────────────────────
class UserRepository {
  static readonly tableName = "users";
  constructor(private readonly db: Database) {}
  async findById(userId: number) { /* ... */ }
}

type RepoInstance = UserRepository;          // { findById(userId: number): Promise<...> }
type RepoCtor     = typeof UserRepository;   // { new (db: Database): UserRepository; tableName: "users" }

// A factory that accepts the class itself, not an instance:
function createRepo<T>(Ctor: new (db: Database) => T, db: Database): T {
  return new Ctor(db);
}
const repo = createRepo(UserRepository, db);   // UserRepository

// Static members live on the constructor type:
type TableName = (typeof UserRepository)["tableName"];   // "users"

// ── Modules (import-time typeof) ─────────────────────────────────────────────
// type ConfigModule = typeof import("./config");        // the whole module's shape
// type ConfigKey    = keyof typeof import("./config");  // its exported names
```

### Concept 9 — Composing with mapped types

`keyof` is the engine behind mapped types. Almost every utility type in `lib.es5.d.ts` is built from it.

```ts
// This IS how Partial, Readonly and Record are defined:
type MyPartial<T>  = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };

// Custom ones you'll actually write on a backend:

// Every field nullable — matching a DB row with LEFT JOINs:
type Nullable<T> = { [K in keyof T]: T[K] | null };
type NullableUser = Nullable<UserRecord>;
// { userId: number | null; email: string | null; isActive: boolean | null; lastLogin: Date | null }

// Only the keys whose value type is a string — useful for building search filters:
type StringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];
type SearchableUserField = StringKeys<UserRecord>;   // "email"

// A handler registry keyed by event name:
const WEBHOOK_EVENTS = {
  "user.created":   { userId: 0, email: "" },
  "order.placed":   { orderId: "", totalCents: 0 },
  "payment.failed": { paymentId: "", reason: "" },
} as const;

type WebhookEventName = keyof typeof WEBHOOK_EVENTS;
type WebhookPayload<E extends WebhookEventName> = (typeof WEBHOOK_EVENTS)[E];

type HandlerMap = { [E in WebhookEventName]: (payload: WebhookPayload<E>) => Promise<void> };

const handlers: HandlerMap = {
  "user.created":   async (payload) => { console.log(payload.email); },      // payload typed ✅
  "order.placed":   async (payload) => { console.log(payload.totalCents); }, // ✅
  "payment.failed": async (payload) => { console.log(payload.reason); },     // ✅
  // omit one → ❌ compile error: property missing
};
```

See **43 — Mapped types** for the full treatment.

---

## Example 1 — basic

```ts
// A typed application config with derived key/value types and a safe reader.

// ── The single source of truth: one plain object ─────────────────────────────
const appConfig = {
  port:            3000,
  host:            "0.0.0.0",
  databaseUrl:     "postgres://localhost:5432/api",
  redisUrl:        "redis://localhost:6379",
  jwtSecret:       "dev-secret-change-me",
  tokenTtlSeconds: 3600,
  logLevel:        "info",
  enableMetrics:   true,
} as const;

// ── Derived types — nothing restated by hand ─────────────────────────────────
type AppConfig      = typeof appConfig;                 // full shape, all readonly literals
type ConfigKey      = keyof typeof appConfig;           // "port" | "host" | "databaseUrl" | ...
type ConfigValue    = (typeof appConfig)[ConfigKey];    // 3000 | "0.0.0.0" | ... (literal union)

// ── A getter that returns the EXACT type per key ─────────────────────────────
function getConfig<K extends ConfigKey>(key: K): AppConfig[K] {
  return appConfig[key];
}

const port      = getConfig("port");             // 3000        (number literal)
const host      = getConfig("host");             // "0.0.0.0"   (string literal)
const metricsOn = getConfig("enableMetrics");    // true        (boolean literal)
// getConfig("prot");                            // ❌ Argument of type '"prot"' is not assignable

// ── Overriding from environment variables, key-checked ───────────────────────
const ENV_MAP = {
  PORT:          "port",
  HOST:          "host",
  DATABASE_URL:  "databaseUrl",
  REDIS_URL:     "redisUrl",
  JWT_SECRET:    "jwtSecret",
  LOG_LEVEL:     "logLevel",
} as const satisfies Record<string, ConfigKey>;   // every value must be a real config key

type EnvVarName = keyof typeof ENV_MAP;           // "PORT" | "HOST" | "DATABASE_URL" | ...

function readEnvOverrides(): Partial<Record<ConfigKey, string>> {
  const overrides: Partial<Record<ConfigKey, string>> = {};
  // Object.keys is typed as string[], so we assert to the known key union:
  for (const envName of Object.keys(ENV_MAP) as EnvVarName[]) {
    const raw = process.env[envName];
    if (raw !== undefined) overrides[ENV_MAP[envName]] = raw;
  }
  return overrides;
}

// ── Log-level ordering derived from an array, not a hand-written union ───────
const LOG_LEVELS = ["trace", "debug", "info", "warn", "error", "fatal"] as const;
type LogLevel = (typeof LOG_LEVELS)[number];     // "trace" | "debug" | ... | "fatal"

function shouldLog(messageLevel: LogLevel, minimumLevel: LogLevel): boolean {
  return LOG_LEVELS.indexOf(messageLevel) >= LOG_LEVELS.indexOf(minimumLevel);
}

shouldLog("warn", "info");        // true  ✅
// shouldLog("verbose", "info");  // ❌ "verbose" is not a LogLevel

// ── A redacted copy for logging — key list is checked against ConfigKey ──────
const SECRET_KEYS = ["jwtSecret", "databaseUrl", "redisUrl"] as const satisfies readonly ConfigKey[];
type SecretKey = (typeof SECRET_KEYS)[number];   // "jwtSecret" | "databaseUrl" | "redisUrl"

function redactedConfig(): Record<ConfigKey, unknown> {
  const out = { ...appConfig } as Record<ConfigKey, unknown>;
  for (const key of SECRET_KEYS) out[key] = "***redacted***";
  return out;
}

console.log(redactedConfig());
// { port: 3000, host: "0.0.0.0", databaseUrl: "***redacted***", ... }
```

---

## Example 2 — real world backend use case

```ts
// A type-safe query builder + serializer layer for a users table.
// Column names, sortable fields, filterable fields and the public API shape are
// ALL derived from one entity type. Nothing is typed twice.

// ── The entity — the single source of truth ──────────────────────────────────
interface UserEntity {
  userId:        number;
  email:         string;
  displayName:   string;
  passwordHash:  string;      // never leaves the server
  role:          "admin" | "editor" | "viewer";
  isActive:      boolean;
  loginCount:    number;
  lastLoginAt:   Date | null;
  createdAt:     Date;
  updatedAt:     Date;
  deletedAt:     Date | null;
}

type UserColumn = keyof UserEntity;   // union of every column name

// ── Column classification, checked against the entity ────────────────────────

// Fields that must never be serialized to a client:
const PRIVATE_COLUMNS = ["passwordHash", "deletedAt"] as const satisfies readonly UserColumn[];
type PrivateColumn = (typeof PRIVATE_COLUMNS)[number];

// The public API shape falls out of the entity automatically:
type PublicUser = Omit<UserEntity, PrivateColumn>;
// If someone adds `twoFactorSecret` to UserEntity and forgets to redact it,
// it appears in PublicUser — so make the redaction list the thing you review.

// Only scalar columns can be sorted by:
type SortableColumn = {
  [K in UserColumn]: UserEntity[K] extends string | number | boolean | Date | null ? K : never
}[UserColumn];
// "userId" | "email" | "displayName" | "passwordHash" | "role" | "isActive"
// | "loginCount" | "lastLoginAt" | "createdAt" | "updatedAt" | "deletedAt"

// Filterable = sortable minus private:
type FilterableColumn = Exclude<SortableColumn, PrivateColumn>;

// ── Query descriptor types ───────────────────────────────────────────────────

type SortDirection = "asc" | "desc";

interface SortSpec {
  column:    Exclude<SortableColumn, PrivateColumn>;
  direction: SortDirection;
}

// The operator set allowed depends on the column's VALUE type:
type OperatorFor<V> =
  V extends string        ? "eq" | "neq" | "like" | "in"
  : V extends number      ? "eq" | "neq" | "lt" | "lte" | "gt" | "gte" | "in"
  : V extends boolean     ? "eq"
  : V extends Date | null ? "eq" | "lt" | "gt" | "isNull"
  : never;

// A filter is a discriminated-by-column object where value type must match the column:
type Filter = {
  [K in FilterableColumn]: {
    column:   K;
    operator: OperatorFor<UserEntity[K]>;
    value:    UserEntity[K] | UserEntity[K][];
  }
}[FilterableColumn];

interface FindUsersQuery {
  select?: readonly Exclude<UserColumn, PrivateColumn>[];
  where?:  readonly Filter[];
  sort?:   readonly SortSpec[];
  limit?:  number;
  offset?: number;
}

// ── The repository ───────────────────────────────────────────────────────────

class UserRepository {
  constructor(private readonly db: Database) {}

  // Generic single-column read — return type is exactly the column's type:
  async getColumn<K extends UserColumn>(userId: number, column: K): Promise<UserEntity[K] | null> {
    const rows = await this.db.query<Pick<UserEntity, K>>(
      `SELECT ${String(column)} FROM users WHERE user_id = $1 LIMIT 1`,
      [userId],
    );
    return rows.length > 0 ? rows[0][column] : null;
  }

  // Multi-column read — the result type is Pick<UserEntity, K>:
  async getColumns<K extends UserColumn>(
    userId: number,
    columns: readonly K[],
  ): Promise<Pick<UserEntity, K> | null> {
    const list = columns.map((c) => toSnakeCase(String(c))).join(", ");
    const rows = await this.db.query<Pick<UserEntity, K>>(
      `SELECT ${list} FROM users WHERE user_id = $1 LIMIT 1`,
      [userId],
    );
    return rows[0] ?? null;
  }

  // Update — key/value pairs must agree per column:
  async updateColumn<K extends Exclude<UserColumn, "userId" | "createdAt">>(
    userId: number,
    column: K,
    value: UserEntity[K],
  ): Promise<void> {
    await this.db.query(
      `UPDATE users SET ${toSnakeCase(String(column))} = $1, updated_at = NOW() WHERE user_id = $2`,
      [value, userId],
    );
  }

  async find(query: FindUsersQuery): Promise<PublicUser[]> {
    const columns = query.select ?? (PUBLIC_COLUMNS as readonly Exclude<UserColumn, PrivateColumn>[]);
    const select  = columns.map((c) => toSnakeCase(String(c))).join(", ");

    const params: unknown[] = [];
    const clauses: string[] = [];

    for (const filter of query.where ?? []) {
      const col = toSnakeCase(String(filter.column));
      switch (filter.operator) {
        case "isNull":
          clauses.push(`${col} IS NULL`);
          break;
        case "in":
          params.push(filter.value);
          clauses.push(`${col} = ANY($${params.length})`);
          break;
        case "like":
          params.push(filter.value);
          clauses.push(`${col} ILIKE $${params.length}`);
          break;
        default: {
          const sqlOp = { eq: "=", neq: "<>", lt: "<", lte: "<=", gt: ">", gte: ">=" }[filter.operator];
          params.push(filter.value);
          clauses.push(`${col} ${sqlOp} $${params.length}`);
        }
      }
    }

    const where   = clauses.length ? `WHERE ${clauses.join(" AND ")}` : "";
    const orderBy = (query.sort ?? []).length
      ? `ORDER BY ${query.sort!.map((s) => `${toSnakeCase(String(s.column))} ${s.direction.toUpperCase()}`).join(", ")}`
      : "";

    params.push(query.limit ?? 50, query.offset ?? 0);
    const sql = `SELECT ${select} FROM users ${where} ${orderBy} LIMIT $${params.length - 1} OFFSET $${params.length}`;
    return this.db.query<PublicUser>(sql, params);
  }
}

// The public column list, verified against the entity at compile time:
const PUBLIC_COLUMNS = [
  "userId", "email", "displayName", "role",
  "isActive", "loginCount", "lastLoginAt", "createdAt", "updatedAt",
] as const satisfies readonly Exclude<UserColumn, PrivateColumn>[];

// ── Usage — every string below is checked ────────────────────────────────────

const userRepo = new UserRepository(db);

const email       = await userRepo.getColumn(42, "email");        // string | null
const lastLogin   = await userRepo.getColumn(42, "lastLoginAt");  // Date | null | null
const credentials = await userRepo.getColumns(42, ["email", "passwordHash"]);
//    credentials: { email: string; passwordHash: string } | null

await userRepo.updateColumn(42, "loginCount", 17);          // ✅ number for a number column
// await userRepo.updateColumn(42, "loginCount", "17");     // ❌ string not assignable to number
// await userRepo.updateColumn(42, "userId", 99);           // ❌ userId is excluded from updates
// await userRepo.getColumn(42, "emails");                  // ❌ not a column

const activeAdmins = await userRepo.find({
  select: ["userId", "email", "role"],
  where: [
    { column: "role",     operator: "eq", value: "admin" },   // ✅ value must be a Role
    { column: "isActive", operator: "eq", value: true },      // ✅ boolean column → only "eq"
    // { column: "isActive", operator: "gt", value: true },   // ❌ "gt" not allowed on boolean
    // { column: "loginCount", operator: "eq", value: "5" },  // ❌ string for a number column
    // { column: "passwordHash", operator: "eq", value: "" }, // ❌ private column not filterable
  ],
  sort:  [{ column: "createdAt", direction: "desc" }],
  limit: 25,
});

// ── Serializer — response shape derived from the entity as well ──────────────

type ApiResponse<T> = { data: T; meta: { count: number; requestId: string } };

function toApiResponse(users: PublicUser[], requestId: string): ApiResponse<PublicUser[]> {
  return { data: users, meta: { count: users.length, requestId } };
}

// Helper referenced above:
function toSnakeCase(input: string): string {
  return input.replace(/[A-Z]/g, (ch) => `_${ch.toLowerCase()}`);
}
```

---

## Going deeper

### 1. `Object.keys` returns `string[]`, not `(keyof T)[]` — and that is correct

This is the single most-complained-about `keyof` behaviour, and it's deliberate.

```ts
interface UserDto { userId: number; email: string }

const dto: UserDto = { userId: 1, email: "a@b.com" };
const keys = Object.keys(dto);       // string[]  ← not ("userId" | "email")[]

// Why? Because TypeScript types are OPEN. This is legal:
const extended = { userId: 1, email: "a@b.com", internalFlag: true };
const asDto: UserDto = extended;     // ✅ allowed — excess property check only hits literals
Object.keys(asDto);                  // at runtime: ["userId", "email", "internalFlag"]

// If Object.keys returned (keyof UserDto)[], the above would be a LIE.
```

So a cast is the pragmatic escape hatch, but understand what you're promising:

```ts
// The idiomatic helper — only sound when you know the object has no extra keys:
function typedKeys<T extends object>(obj: T): (keyof T)[] {
  return Object.keys(obj) as (keyof T)[];
}

// Same for entries:
function typedEntries<T extends object>(obj: T): { [K in keyof T]: [K, T[K]] }[keyof T][] {
  return Object.entries(obj) as never;
}

for (const [key, value] of typedEntries(dto)) {
  // key: "userId" | "email"; value: number | string  (correlated per key in the tuple union)
}
```

For `as const` object literals defined in the same file, the cast is safe — nothing can add keys.

### 2. `keyof` on a union is an intersection, and it surprises everyone

```ts
type A = { requestId: string; userId: number };
type B = { requestId: string; orderId: string };

type UnionKeys = keyof (A | B);        // "requestId"  ← ONLY the shared key
type InterKeys = keyof (A & B);        // "requestId" | "userId" | "orderId"
```

The rule: `keyof (A | B) = keyof A & keyof B`, and `keyof (A & B) = keyof A | keyof B`. This is De Morgan's law showing up in the type system, and it's sound — for a value that might be an `A` or a `B`, only keys present in both are guaranteed accessible.

When you actually want all keys, force distribution through a conditional type:

```ts
type DistributedKeyof<T> = T extends unknown ? keyof T : never;
type AllKeys = DistributedKeyof<A | B>;   // "requestId" | "userId" | "orderId"
```

See **44 — Conditional types** for why the naked `T extends unknown` triggers distribution.

### 3. `T[K]` where `K` is a generic stays deferred — and blocks some writes

Inside a generic function, `T[K]` is an *unresolved* indexed access. The compiler cannot know the concrete type, so certain operations are rejected even though they look obviously fine.

```ts
function updateField<T, K extends keyof T>(obj: T, key: K, value: T[K]): T {
  obj[key] = value;              // ✅ this direction works
  return obj;
}

function badMerge<T, K extends keyof T>(obj: T, key: K): void {
  // obj[key] = "some string";   // ❌ Type 'string' is not assignable to type 'T[K]'
  // Correct — even if T[K] happens to be string for your call, the compiler
  // must accept ALL possible T, and T[K] could be number for another caller.
}

// Mapped-type writes through a generic often need a helper signature:
function copyField<T, K extends keyof T>(from: T, to: T, key: K): void {
  to[key] = from[key];           // ✅ both sides are T[K]
}
```

Related trap: TypeScript will not let you *construct* a `Pick<T, K>` incrementally without a cast, because the object is incomplete mid-loop:

```ts
function pick<T extends object, K extends keyof T>(obj: T, keys: readonly K[]): Pick<T, K> {
  const out = {} as Pick<T, K>;   // the cast is necessary — {} is not yet a Pick
  for (const k of keys) out[k] = obj[k];
  return out;
}
```

### 4. `keyof` respects `private`/`protected` and optionality, but not `readonly`

```ts
class TokenService {
  public  readonly issuer = "api.example.com";
  private          signingKey = "secret";
  protected        clockSkewSeconds = 30;
  public sign(userId: number): string { return "..."; }
}

type TokenServiceKey = keyof TokenService;   // "issuer" | "sign"
// private and protected members are NOT in keyof — they're not part of the public shape.

interface PatchUserBody {
  email?:       string;
  displayName?: string;
  readonly userId: number;
}
type PatchKey = keyof PatchUserBody;   // "email" | "displayName" | "userId"
// Optional (?) keys ARE included — optionality affects the VALUE type (| undefined), not the key.
// readonly affects assignability, not membership.
```

To filter optional keys out you need a mapped type trick:

```ts
type RequiredKeys<T> = { [K in keyof T]-?: {} extends Pick<T, K> ? never : K }[keyof T];
type OptionalKeys<T> = { [K in keyof T]-?: {} extends Pick<T, K> ? K : never }[keyof T];

type Req = RequiredKeys<PatchUserBody>;   // "userId"
type Opt = OptionalKeys<PatchUserBody>;   // "email" | "displayName"
```

### 5. `typeof` on a `let` widens; on a `const` it doesn't

```ts
const constLevel = "info";
let   letLevel   = "info";

type A = typeof constLevel;   // "info"   ← literal type preserved
type B = typeof letLevel;     // string   ← widened, because let is reassignable

// Same rule inside objects — properties are always mutable, so they widen:
const config = { level: "info" };
type C = typeof config;             // { level: string }
type D = (typeof config)["level"];  // string  ← NOT "info"

const frozen = { level: "info" } as const;
type E = (typeof frozen)["level"];  // "info"  ✅
```

This is the #1 reason `keyof typeof X` "works" but `typeof X[keyof typeof X]` gives you `string`. Keys are always literal; **values are only literal under `as const`**.

### 6. Precedence: `typeof obj[key]` parses as `typeof (obj[key])`

```ts
const HTTP_STATUS = { OK: 200, NOT_FOUND: 404 } as const;

// ⚠ These are DIFFERENT in older mental models. In current TS both work,
// but the parenthesized form is unambiguous and what you should write:
type V1 = typeof HTTP_STATUS[keyof typeof HTTP_STATUS];    // 200 | 404 — works
type V2 = (typeof HTTP_STATUS)[keyof typeof HTTP_STATUS];  // 200 | 404 — identical, clearer

// Where it genuinely bites is with a value that has a computed index:
const statusKey = "OK";
// type V3 = typeof HTTP_STATUS[statusKey];  // ❌ 'statusKey' refers to a value, not a type
type V3 = (typeof HTTP_STATUS)[typeof statusKey];   // ✅ 200
```

Always parenthesize `(typeof X)[...]`. Some linters (`@typescript-eslint`) will not flag it, but readers of your code will thank you.

### 7. `satisfies` — the missing piece for const-object patterns

Before `satisfies` (TS 4.9) you had to choose between *validating* an object against a type (losing literal inference) and *inferring* precisely (losing validation).

```ts
type RouteHandler = (req: Request, res: Response) => Promise<void>;

// ❌ Annotation: validated, but keyof gives you the WIDE index type:
const routesA: Record<string, RouteHandler> = {
  "GET /users":     listUsers,
  "POST /users":    createUser,
};
type KeyA = keyof typeof routesA;   // string  ← useless

// ❌ No annotation: precise keys, but nothing checks the handler signatures:
const routesB = { "GET /users": listUsers, "POST /users": createUser };
type KeyB = keyof typeof routesB;   // "GET /users" | "POST /users"  ✅ but unvalidated

// ✅ satisfies: BOTH — validated against the type AND inferred as literals:
const routes = {
  "GET /users":      listUsers,
  "POST /users":     createUser,
  "DELETE /users/:id": deleteUser,
} satisfies Record<string, RouteHandler>;

type RouteKey = keyof typeof routes;   // "GET /users" | "POST /users" | "DELETE /users/:id" ✅
// and a handler with the wrong signature is a compile error ✅
```

`as const satisfies T` combines both: readonly literal inference plus validation. Use it for every lookup table.

### 8. Performance: huge `keyof` unions slow the compiler

`keyof` on a type with hundreds of keys, fed into a distributive conditional or a mapped type, can blow up type instantiation counts. Symptoms: editor lag, `tsc` taking minutes, "Type instantiation is excessively deep and possibly infinite."

Practical mitigations:

```ts
// ❌ Recursively mapping every key of a huge deeply-nested schema:
type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> };
// applied to a 400-field generated DB schema → very slow

// ✅ Cache the expensive derivation once behind a named alias, and export THAT.
// Named aliases are memoized by the checker; inline re-computation is not.
type UserColumn = keyof UserEntity;              // computed once
type Sortable   = Exclude<UserColumn, "passwordHash">;  // reuses the alias

// ✅ Avoid keyof on `any`-typed or generated-from-JSON types; annotate boundaries.
// ✅ Run `tsc --generateTrace ./trace` to find the offending type if it gets bad.
```

### 9. `keyof` and index signatures interact badly with `noUncheckedIndexedAccess`

```ts
// tsconfig: { "noUncheckedIndexedAccess": true }

interface SessionMap { [sessionId: string]: Session }
const sessions: SessionMap = {};
const s = sessions["abc"];       // Session | undefined  ← flag adds the undefined ✅

// But for a FIXED-key object, keyof access stays non-optional (correct):
const CONFIG = { port: 3000, host: "0.0.0.0" } as const;
function read<K extends keyof typeof CONFIG>(k: K) {
  return CONFIG[k];              // no `| undefined` — K is provably a real key ✅
}
```

Prefer `Record<LiteralUnion, V>` over an index signature whenever the key set is known; you get exhaustiveness *and* no phantom `undefined`.

### 10. Template literal types compose with `keyof`

```ts
interface UserEvents {
  created: { userId: number };
  updated: { userId: number; changes: string[] };
  deleted: { userId: number };
}

// Prefix every key — useful for namespaced event buses / Redis channels:
type PrefixedEvent = `user.${string & keyof UserEvents}`;
// "user.created" | "user.updated" | "user.deleted"

// Generate getter names from an entity's fields:
type Getters<T> = { [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K] };
type UserGetters = Getters<{ userId: number; email: string }>;
// { getUserId: () => number; getEmail: () => string }
```

Note the `& string` — `keyof T` can include `number | symbol`, which template literal types reject.

---

## Common mistakes

### Mistake 1 — Forgetting `as const`, so the value union collapses to `string`

```ts
// ❌ Values widen to string — the derived union is worthless:
const NOTIFICATION_CHANNELS = {
  EMAIL: "email",
  SMS:   "sms",
  PUSH:  "push",
};

type Channel = (typeof NOTIFICATION_CHANNELS)[keyof typeof NOTIFICATION_CHANNELS];
// string  ← every string is now a valid "channel"

function send(channel: Channel): void { /* ... */ }
send("emial");   // ✅ compiles. 💥 fails in production.
```

```ts
// ✅ `as const` freezes the literals:
const NOTIFICATION_CHANNELS = {
  EMAIL: "email",
  SMS:   "sms",
  PUSH:  "push",
} as const;

type Channel = (typeof NOTIFICATION_CHANNELS)[keyof typeof NOTIFICATION_CHANNELS];
// "email" | "sms" | "push"  ✅

function send(channel: Channel): void { /* ... */ }
// send("emial");  // ❌ compile error
send("email");     // ✅
```

### Mistake 2 — Using `key: keyof T` instead of a generic `K extends keyof T`

```ts
// ❌ Return type collapses to a union of every property type:
interface UserRecord { userId: number; email: string; isActive: boolean }

function getField(user: UserRecord, key: keyof UserRecord): UserRecord[keyof UserRecord] {
  return user[key];
}

const email = getField(user, "email");   // number | string | boolean  😞
email.toLowerCase();                     // ❌ Property 'toLowerCase' does not exist on
                                         //    type 'string | number | boolean'
```

```ts
// ✅ Capture the specific key in a type parameter:
function getField<K extends keyof UserRecord>(user: UserRecord, key: K): UserRecord[K] {
  return user[key];
}

const email = getField(user, "email");   // string  ✅
email.toLowerCase();                     // ✅
```

### Mistake 3 — Using `keyof` on an array expecting the element type

```ts
// ❌ keyof an array gives every array member, including methods:
const AUDIT_ACTIONS = ["create", "update", "delete"] as const;

type Action = keyof typeof AUDIT_ACTIONS;
// number | "length" | "toString" | "concat" | "join" | "slice" | "indexOf" | ... 😱

function log(action: Action): void { /* ... */ }
log("length");   // ✅ compiles — clearly wrong
```

```ts
// ✅ Index by `number` to get the element type:
const AUDIT_ACTIONS = ["create", "update", "delete"] as const;

type Action = (typeof AUDIT_ACTIONS)[number];   // "create" | "update" | "delete" ✅

function log(action: Action): void { /* ... */ }
// log("length");  // ❌ compile error
log("create");     // ✅
```

### Mistake 4 — Confusing runtime `typeof` with type-level `typeof`

```ts
// ❌ Trying to use the runtime result as a type:
const requestBody = { userId: 42, email: "dev@example.com" };

const bodyType = typeof requestBody;      // this is a VALUE: the string "object"
// let parsed: bodyType;                  // ❌ 'bodyType' refers to a value, not a type

// ❌ Trying to use a type in a runtime check:
interface RequestBody { userId: number; email: string }
// if (requestBody instanceof RequestBody) {}   // ❌ 'RequestBody' only refers to a type
```

```ts
// ✅ Type-level typeof goes in a TYPE position:
const requestBody = { userId: 42, email: "dev@example.com" };
type RequestBody = typeof requestBody;    // { userId: number; email: string }
let parsed: RequestBody = { userId: 1, email: "x@y.com" };   // ✅

// ✅ For runtime checks, write a type guard (see 41 — Type guards):
function isRequestBody(value: unknown): value is RequestBody {
  return typeof value === "object" && value !== null
    && "userId" in value && typeof (value as RequestBody).userId === "number"
    && "email"  in value && typeof (value as RequestBody).email  === "string";
}
```

### Mistake 5 — Assuming `Object.keys` is typed as `(keyof T)[]`

```ts
// ❌ This does not compile:
interface AppConfig { port: number; host: string }
const config: AppConfig = { port: 3000, host: "0.0.0.0" };

for (const key of Object.keys(config)) {
  console.log(config[key]);
  // ❌ Element implicitly has an 'any' type because expression of type 'string'
  //    can't be used to index type 'AppConfig'
}
```

```ts
// ✅ Assert once at the boundary with a named helper:
function typedKeys<T extends object>(obj: T): (keyof T)[] {
  return Object.keys(obj) as (keyof T)[];
}

for (const key of typedKeys(config)) {
  console.log(key, config[key]);   // key: "port" | "host"; value: number | string ✅
}
```

### Mistake 6 — Expecting `keyof` on a union to give all keys

```ts
// ❌ Only shared keys come out:
type EmailJob = { kind: "email"; to: string; subject: string };
type SmsJob   = { kind: "sms";   to: string; message: string };
type Job = EmailJob | SmsJob;

type JobField = keyof Job;   // "kind" | "to"   ← subject and message are missing
function readField(job: Job, field: JobField) { return job[field]; }
// readField(job, "subject");  // ❌ not assignable
```

```ts
// ✅ Distribute over the union when you want every key:
type AllKeys<T> = T extends unknown ? keyof T : never;
type JobField = AllKeys<Job>;   // "kind" | "to" | "subject" | "message" ✅

// …but then narrow before reading — a field may not exist on the actual member:
function readField(job: Job, field: JobField): string | undefined {
  return field in job ? (job as Record<string, string>)[field] : undefined;
}
```

---

## Practice exercises

### Exercise 1 — easy

Given this runtime constant table, derive all the types listed and write the helper functions. Do not restate any key or value literal by hand.

```ts
// Start from exactly this object (copy it, add `as const`):
// const HTTP_STATUS = {
//   OK: 200, CREATED: 201, NO_CONTENT: 204,
//   BAD_REQUEST: 400, UNAUTHORIZED: 401, FORBIDDEN: 403,
//   NOT_FOUND: 404, CONFLICT: 409, UNPROCESSABLE: 422,
//   TOO_MANY_REQUESTS: 429, INTERNAL_ERROR: 500,
// };
//
// Derive:
//   type StatusName    — the union of the KEY names
//   type StatusCode    — the union of the numeric VALUES
//   type SuccessCode   — only the 2xx codes (use Extract or a conditional)
//
// Then write:
//   codeFor(name: StatusName): StatusCode
//   nameFor(code: StatusCode): StatusName          — reverse lookup, no `any`
//   isSuccess(code: StatusCode): code is SuccessCode
//
// Prove: codeFor("NOT_FUND") and codeFor(404) must both be compile errors.
```

```ts
// Write your code here
```

### Exercise 2 — medium

Build a small type-safe in-memory cache keyed by a fixed set of entity types.

```ts
// Given a registry object that maps a cache namespace to a sample record shape:
//   const CACHE_SHAPES = {
//     user:    { userId: 0, email: "", role: "viewer" },
//     order:   { orderId: "", totalCents: 0, status: "pending" },
//     session: { sessionId: "", userId: 0, expiresAt: new Date() },
//   };
//
// Derive:
//   type CacheNamespace         — "user" | "order" | "session"
//   type CacheValue<N>          — the record shape for a given namespace
//
// Then implement a class TypedCache with:
//   set<N extends CacheNamespace>(namespace: N, key: string, value: CacheValue<N>, ttlMs: number): void
//   get<N extends CacheNamespace>(namespace: N, key: string): CacheValue<N> | null
//   getField<N extends CacheNamespace, K extends keyof CacheValue<N>>(
//     namespace: N, key: string, field: K
//   ): CacheValue<N>[K] | null
//   invalidate<N extends CacheNamespace>(namespace: N, key?: string): void
//   stats(): Record<CacheNamespace, { hits: number; misses: number; size: number }>
//
// Requirements:
//   - Storing an order shape under the "user" namespace must be a compile error.
//   - getField(cache, "user", "id", "totalCents") must be a compile error.
//   - stats() must be exhaustive — adding a namespace to CACHE_SHAPES without
//     updating stats() should fail to compile.
//   - No `any` anywhere. One internal cast at the Map boundary is acceptable.
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a fully type-safe event bus + audit-log projector for a backend, driven entirely by one const registry.

```ts
// Define ONE registry object describing every domain event and its payload shape:
//   user.registered, user.email_changed, user.deleted,
//   order.placed, order.paid, order.shipped, order.refunded,
//   payment.authorized, payment.failed
// Each payload should include at least: an entity id, an actorId, and event-specific fields.
//
// From that registry alone, derive:
//   type EventName                      — every event name
//   type PayloadOf<E extends EventName> — the payload for one event
//   type EventDomain                    — "user" | "order" | "payment"
//     (derive it from the event names with a template literal type — do NOT hand-write it)
//   type EventsInDomain<D>              — every event name in a domain
//   type HandlerMap                     — { [E in EventName]: Handler<E>[] }
//
// Implement class EventBus with:
//   on<E extends EventName>(event: E, handler: (payload: PayloadOf<E>, meta: EventMeta) => Promise<void>): () => void
//     — returns an unsubscribe function
//   onDomain<D extends EventDomain>(domain: D, handler: (event: EventsInDomain<D>, payload: unknown, meta: EventMeta) => Promise<void>): () => void
//   emit<E extends EventName>(event: E, payload: PayloadOf<E>): Promise<void>
//     — runs every matching handler, collects errors, never throws to the caller
//   waitFor<E extends EventName>(event: E, predicate: (p: PayloadOf<E>) => boolean, timeoutMs: number): Promise<PayloadOf<E>>
//
// Then implement an AuditProjector that:
//   - Subscribes to ALL events via a loop over the registry keys (typed, no `any`)
//   - Maps each event to an audit row: { eventName, domain, entityId, actorId, occurredAt, payloadJson }
//     — entityId must be extracted generically: derive a type-level mapping from
//       each event to the NAME of its id field (e.g. "order.paid" -> "orderId")
//       using a mapped type over the registry, and use it at runtime.
//   - Exposes a summary(): Record<EventDomain, number>  — exhaustive over domains
//
// Prove with commented-out lines:
//   - emit("order.paid", <user payload>) is a compile error
//   - on("order.pait", ...) is a compile error
//   - a handler that reads payload.nonExistentField is a compile error
//   - adding an event to the registry without updating AuditProjector's id-field map
//     is a compile error
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── keyof ─────────────────────────────────────────────────────────────────────
keyof User                       // union of User's public key names
keyof {}                         // never
keyof any                        // string | number | symbol
keyof (A | B)                    // keyof A & keyof B  (shared keys only)
keyof (A & B)                    // keyof A | keyof B  (all keys)
keyof string[]                   // number | "length" | "push" | ... (everything)
keyof Record<string, V>          // string | number
T extends unknown ? keyof T : never   // distribute keyof over a union

// ── typeof (TYPE position) ────────────────────────────────────────────────────
typeof appConfig                 // the object's inferred type
typeof handler                   // a function's full signature
typeof UserRepository            // the CLASS/constructor type, not the instance
typeof import("./config")        // a module's shape

// ── Indexed access ────────────────────────────────────────────────────────────
User["email"]                    // the type at one key
User["userId" | "email"]         // union of two value types
User[keyof User]                 // union of ALL value types
Order["items"][number]           // array element type
DbTuple[0]                       // tuple positional type
(typeof UserRepository)["tableName"]   // a static member's type

// ── The four canonical combos ─────────────────────────────────────────────────
type K = keyof typeof OBJ;                    // key union of a const object
type V = (typeof OBJ)[keyof typeof OBJ];      // value union of a const object
type E = (typeof ARR)[number];                // element union of a const array
function get<T, K extends keyof T>(o: T, k: K): T[K] { return o[k]; }   // exact getter

// ── Always pair const objects with as const / satisfies ───────────────────────
const T1 = { a: 1 };                              // values widen → number
const T2 = { a: 1 } as const;                     // values literal → 1, readonly
const T3 = { a: fn } satisfies Record<string, F>; // validated AND literal keys
const T4 = { a: 1 } as const satisfies Record<string, number>;  // both
```

| Expression | Yields | Note |
|---|---|---|
| `keyof T` | union of key literals | public members only; private/protected excluded |
| `typeof v` (type position) | `v`'s inferred type | compile-time; unrelated to runtime `typeof` |
| `keyof typeof obj` | keys of a runtime object | the workhorse combo |
| `(typeof obj)[keyof typeof obj]` | value union | needs `as const` or values widen |
| `(typeof arr)[number]` | element union | use this, never `keyof arr` |
| `T[K]` | type at key `K` | `K` must be a valid key; checked |
| `K extends keyof T` | generic key param | required for a precise `T[K]` return |
| `keyof (A \| B)` | shared keys only | intersection, not union |
| `Object.keys(o)` | `string[]` | not `(keyof T)[]` — types are open |
| `typeof SomeClass` | constructor type | `SomeClass` alone is the instance type |
| `as const` | literal + readonly | required for value unions |
| `satisfies T` | validate, keep literals | preferred over `: T` for lookup tables |

---

## Connected topics

- **32 — Utility types** — `Pick`, `Omit`, `Record`, `Extract` and `Exclude` are all defined in terms of `keyof` and indexed access.
- **43 — Mapped types** — `{ [K in keyof T]: ... }` is the direct consumer of `keyof`; every derived-shape trick starts here.
- **44 — Conditional types** — needed to distribute `keyof` over unions and to filter keys by their value type.
- **28 — Enums** — `as const` objects plus `keyof typeof` are the recommended alternative to TypeScript enums.
- **42 — Discriminated unions** — `keyof` on a union gives only shared keys, which is exactly why discriminants must be narrowed before field access.
