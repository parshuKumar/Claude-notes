# 43 — Mapped Types

## What is this?

A **mapped type** is a type that is *computed* by walking over the keys of another type and producing a new property for each one. It is a `for...in` loop, but at the type level, evaluated by the compiler instead of at runtime.

```ts
// For every key K in the keys of T, produce a property K whose type is T[K]:
type Clone<T> = {
  [K in keyof T]: T[K];
};

interface User {
  userId: number;
  email:  string;
  isAdmin: boolean;
}

type UserClone = Clone<User>;
// = { userId: number; email: string; isAdmin: boolean }
```

The `[K in keyof T]` part is the loop header. `K` is the loop variable — it takes on each key of `T` in turn (`"userId"`, then `"email"`, then `"isAdmin"`). The part after the colon is the value expression — it computes the type of that property.

Once you can loop over keys, you can transform them: make them all optional, all readonly, all nullable, all `string`, wrap them in promises, rename them, or drop them entirely. Every one of TypeScript's built-in transform utilities — `Partial`, `Required`, `Readonly`, `Record`, `Pick`, `Omit` — is a mapped type, written in about three lines.

## Why does it matter?

Backend code is full of **the same shape, in several dresses**:

- `User` is the domain model. `UserRow` is what the database returns — every column can be `NULL`. `CreateUserDto` is what the client sends — no `userId`, no `createdAt`. `UserPatchDto` is a PATCH body — every field optional. `UserResponse` is what you serialise — `Date` fields become ISO strings.
- An event bus has `EventMap` of event name → payload. The handler registry must be event name → `(payload) => void`. Same keys, transformed values.
- A config object has typed values (`port: number`, `debug: boolean`), but the environment gives you strings. You need `Stringify<AppConfig>` for parsing, `AppConfig` for use.
- A query builder needs `WhereClause<User>` — every field of `User` but wrapped in `{ eq?, gt?, in? }` operators.

Without mapped types you write each of those by hand and they drift apart. Someone adds `phoneNumber` to `User` and forgets `UserPatchDto`, and the PATCH endpoint silently ignores the field. With mapped types, the derived shapes are *functions of* the source type — add a field to `User` and every derived type updates in the same compile.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — "the same shape in several dresses" is pure convention and comments.
// You maintain five parallel object literals, and nothing links them.

// The domain shape (documented only in a comment):
// { userId, email, passwordHash, displayName, isAdmin, createdAt }

// The DB row (all nullable — you just have to remember):
function rowToUser(row) {
  return {
    userId:       row.user_id,
    email:        row.email,
    passwordHash: row.password_hash,
    displayName:  row.display_name,   // may be null — nothing tells you
    isAdmin:      row.is_admin,
    createdAt:    row.created_at,
  };
}

// The PATCH body validator (hand-written key list — drifts constantly):
const PATCHABLE = ["email", "displayName"]; // someone added phoneNumber to User... not here
function applyPatch(user, requestBody) {
  for (const key of PATCHABLE) {
    if (key in requestBody) user[key] = requestBody[key];
  }
  return user;
}

// The serialiser (dates -> strings, hand-written again):
function toResponse(user) {
  return {
    userId:      user.userId,
    email:       user.email,
    displayName: user.displayName,
    isAdmin:     user.isAdmin,
    createdAt:   user.createdAt.toISOString(),
    // passwordHash omitted... hopefully. Nothing enforces it.
  };
}
```

Five hand-maintained shapes, zero enforcement. Adding `phoneNumber` to the user means finding and editing four places you will not remember.

```ts
// TypeScript — write the shape ONCE. Derive every other dress from it.

interface User {
  userId:       number;
  email:        string;
  passwordHash: string;
  displayName:  string;
  isAdmin:      boolean;
  createdAt:    Date;
}

// Every column can come back NULL from the driver:
type Nullable<T> = { [K in keyof T]: T[K] | null };
type UserRow = Nullable<User>;
// = { userId: number | null; email: string | null; ... createdAt: Date | null }

// PATCH body — every field optional, and only the safe ones:
type UserPatchDto = Partial<Omit<User, "userId" | "passwordHash" | "createdAt">>;
// = { email?: string; displayName?: string; isAdmin?: boolean }

// Wire format — Dates become ISO strings, secrets removed:
type Serialized<T> = { [K in keyof T]: T[K] extends Date ? string : T[K] };
type UserResponse = Serialized<Omit<User, "passwordHash">>;
// = { userId: number; email: string; displayName: string; isAdmin: boolean; createdAt: string }

// Add `phoneNumber: string` to User and ALL THREE update in the same compile.
// Forget to handle it in a mapper? Compile error, not a 3am pager.
```

The revelation: you stop *maintaining* related types and start *deriving* them. `User` becomes the single source of truth, and every other shape is a pure function of it.

---

## Syntax

```ts
// ── The basic form ──────────────────────────────────────────────────────────
type Identity<T> = {
  [K in keyof T]: T[K];   // K = loop variable, keyof T = the key union, T[K] = indexed access
};

// ── Add the optional modifier to every key ──────────────────────────────────
type AllOptional<T> = {
  [K in keyof T]?: T[K];  // `?` after the bracket makes every property optional
};

// ── Add the readonly modifier to every key ──────────────────────────────────
type AllReadonly<T> = {
  readonly [K in keyof T]: T[K];  // `readonly` before the bracket
};

// ── REMOVE modifiers with the minus sign ────────────────────────────────────
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];  // strips readonly
};
type Concrete<T> = {
  [K in keyof T]-?: T[K];          // strips `?` (and removes undefined from the value type)
};

// ── Map over an arbitrary key union, not just keyof T ───────────────────────
type Flags<K extends string> = {
  [Key in K]: boolean;             // K can be any union of string | number | symbol
};
type FeatureFlags = Flags<"darkMode" | "betaApi">;
// = { darkMode: boolean; betaApi: boolean }

// ── Key remapping with `as` (TS 4.1+) — rename the produced key ─────────────
type Prefixed<T> = {
  [K in keyof T as `db_${string & K}`]: T[K];  // template literal builds the new key
};

// ── Filter keys out by remapping to `never` ─────────────────────────────────
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];  // never = key is dropped
};

// ── Transform the value type, not just the modifiers ────────────────────────
type Promised<T> = {
  [K in keyof T]: Promise<T[K]>;   // wrap every value in a Promise
};
```

---

## How it works — concept by concept

### Concept 1 — `keyof`, indexed access, and the loop

Three ingredients make a mapped type. Understand each on its own first.

```ts
interface User {
  userId:    number;
  email:     string;
  isActive:  boolean;
}

// (1) keyof T — the union of a type's keys, as string literal types:
type UserKeys = keyof User;
// = "userId" | "email" | "isActive"

// (2) Indexed access T[K] — the type of a property, looked up by key:
type EmailType = User["email"];              // string
type IdOrEmail = User["userId" | "email"];   // number | string  (distributes over the key union)
type AnyValue  = User[keyof User];           // number | string | boolean

// (3) The mapped loop — for each K in that union, emit a property:
type Rebuilt = {
  [K in keyof User]: User[K];
};
// K = "userId"   → userId:   User["userId"]   = number
// K = "email"    → email:    User["email"]    = string
// K = "isActive" → isActive: User["isActive"] = boolean
```

The key union does not have to come from `keyof`. Any union of `string | number | symbol` works:

```ts
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

type RouteHandlers = {
  [M in HttpMethod]: (requestBody: unknown) => Promise<void>;
};
// = { GET: (...) => ...; POST: (...) => ...; PUT: (...) => ...; DELETE: (...) => ... }

// This is exactly how Record is defined in lib.es5.d.ts:
type MyRecord<K extends keyof any, V> = {
  [P in K]: V;
};
type StatusCounts = MyRecord<"pending" | "failed" | "done", number>;
// = { pending: number; failed: number; done: number }
```

### Concept 2 — How `Partial`, `Required`, `Readonly`, `Pick` are actually built

Open `lib.es5.d.ts` and the famous utility types are four one-liners. Nothing magic — you can write them yourself in a minute.

```ts
// Partial — add `?` to every key:
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Required — remove `?` from every key (the `-?` modifier):
type MyRequired<T> = {
  [K in keyof T]-?: T[K];
};

// Readonly — add `readonly` to every key:
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Pick — loop over a SUBSET of keys instead of all of them:
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];
};

// Record — loop over an arbitrary key union with a fixed value type:
type MyRecord<K extends keyof any, V> = {
  [P in K]: V;
};

// Omit is NOT a mapped type by itself — it is Pick over a filtered key union:
type MyOmit<T, K extends keyof any> = MyPick<T, Exclude<keyof T, K>>;

// ── Proof they behave identically to the built-ins ──────────────────────────
interface AuthToken {
  token:      string;
  userId:     number;
  expiresAt:  Date;
  scopes:     string[];
}

type A = MyPartial<AuthToken>;             // { token?: string; userId?: number; ... }
type B = MyReadonly<AuthToken>;            // { readonly token: string; ... }
type C = MyPick<AuthToken, "token" | "userId">;  // { token: string; userId: number }
type D = MyOmit<AuthToken, "token">;       // { userId: number; expiresAt: Date; scopes: string[] }
```

Knowing this matters because the moment a built-in *almost* fits, you can write the variant you need instead of contorting your code around it.

### Concept 3 — Modifiers: `readonly`, `?`, and the `+` / `-` prefixes

Every mapped type can add or remove the two property modifiers. `+` is the default and is almost always omitted; `-` is the interesting one.

```ts
interface MutableConfig {
  port:      number;
  host:      string;
  debug?:    boolean;          // already optional
  readonly appName: string;    // already readonly
}

// ── ADD readonly to everything (the `+` is implicit) ────────────────────────
type FrozenConfig = { readonly [K in keyof MutableConfig]: MutableConfig[K] };
// = { readonly port: number; readonly host: string; readonly debug?: boolean; readonly appName: string }

// ── REMOVE readonly from everything ─────────────────────────────────────────
type ThawedConfig = { -readonly [K in keyof MutableConfig]: MutableConfig[K] };
// = { port: number; host: string; debug?: boolean; appName: string }   ← appName is writable now

// ── REMOVE optionality — note it also strips `undefined` from the value ─────
type FullConfig = { [K in keyof MutableConfig]-?: MutableConfig[K] };
// debug: boolean   (NOT boolean | undefined — the `-?` removes undefined too)

// ── ADD optionality ─────────────────────────────────────────────────────────
type ConfigPatch = { [K in keyof MutableConfig]?: MutableConfig[K] };

// ── Both at once — a fully frozen, fully required snapshot ──────────────────
type ConfigSnapshot = {
  readonly [K in keyof MutableConfig]-?: MutableConfig[K];
};
// = { readonly port: number; readonly host: string; readonly debug: boolean; readonly appName: string }

// ── The explicit + form (valid, rarely written) ─────────────────────────────
type Explicit<T> = { +readonly [K in keyof T]+?: T[K] };  // same as `readonly [K in keyof T]?:`
```

An important subtlety: `-?` removes `undefined` from the property type, but `?` does **not** add `undefined` to the property type under `exactOptionalPropertyTypes`. Without that flag, `{ debug?: boolean }` means `boolean | undefined`; with it, `debug` may be *absent* but may not be *explicitly `undefined`*.

```ts
// tsconfig: "exactOptionalPropertyTypes": false (default)
type Loose = { [K in keyof MutableConfig]?: MutableConfig[K] };
const a: Loose = { port: undefined };   // ✅ allowed — undefined is in the type

// tsconfig: "exactOptionalPropertyTypes": true
const b: Loose = { port: undefined };   // ❌ Error — the key may be missing, not undefined
const c: Loose = {};                    // ✅
```

### Concept 4 — Key remapping with `as`

Since TypeScript 4.1, you can rename the produced key with an `as` clause. The expression after `as` must evaluate to `string | number | symbol` — and if it evaluates to `never`, the key is **dropped entirely**.

```ts
interface User {
  userId:      number;
  email:       string;
  displayName: string;
  createdAt:   Date;
}

// ── Rename with a template literal type ─────────────────────────────────────
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<User>;
// = {
//     getUserId:      () => number;
//     getEmail:       () => string;
//     getDisplayName: () => string;
//     getCreatedAt:   () => Date;
//   }
// Note `string & K` — K is `keyof T` which is string|number|symbol; the intersection
// narrows it to the string part so Capitalize<> accepts it.

// ── Snake-case the keys for a DB layer ──────────────────────────────────────
type SnakeCase<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Tail extends Uncapitalize<Tail>
      ? `${Uncapitalize<Head>}${SnakeCase<Tail>}`
      : `${Uncapitalize<Head>}_${SnakeCase<Tail>}`
    : S;

type DbColumns<T> = {
  [K in keyof T as SnakeCase<string & K>]: T[K];
};
type UserColumns = DbColumns<User>;
// = { user_id: number; email: string; display_name: string; created_at: Date }

// ── DROP keys by remapping to never ─────────────────────────────────────────
type RemoveDates<T> = {
  [K in keyof T as T[K] extends Date ? never : K]: T[K];
};
type UserNoDates = RemoveDates<User>;
// = { userId: number; email: string; displayName: string }   ← createdAt gone

// ── Keep only keys whose name matches a pattern ─────────────────────────────
type IdFields<T> = {
  [K in keyof T as K extends `${string}Id` ? K : never]: T[K];
};
type UserIds = IdFields<User>;
// = { userId: number }

// ── Remap MANY keys to ONE (they merge, values union) ───────────────────────
type Collapse<T> = {
  [K in keyof T as "all"]: T[K];
};
type Collapsed = Collapse<User>;
// = { all: number | string | Date }   ← all four keys collapsed into one union-valued key
```

Key remapping is the feature that turns mapped types from "modifier tweaks" into a real transformation language.

### Concept 5 — Homomorphic mapped types (and why the shape survives)

A mapped type of the exact form `{ [K in keyof T]: ... }` — where the key source is literally `keyof T` for a type parameter `T` — is called **homomorphic**. The compiler treats it specially:

1. It **preserves existing modifiers** (`readonly` and `?` on the source stay unless you explicitly change them).
2. It **distributes over arrays and tuples** — `Partial<string[]>` gives `(string | undefined)[]`, not an object with `length?`.
3. It **distributes over unions** — `Partial<A | B>` gives `Partial<A> | Partial<B>`.
4. It **passes primitives through unchanged** — `Partial<number>` is `number`.

```ts
interface Source {
  readonly id: number;
  name?: string;
}

// ── Homomorphic: modifiers preserved ────────────────────────────────────────
type Homo<T> = { [K in keyof T]: T[K] };
type H = Homo<Source>;
// = { readonly id: number; name?: string }   ← readonly and ? survived

// ── NON-homomorphic: keys came from a different expression, modifiers LOST ──
type NonHomo<T> = { [K in keyof T & string]: T[K] };
type N = NonHomo<Source>;
// = { id: number; name: string | undefined }   ← readonly gone, ? gone

// ── Array / tuple distribution ──────────────────────────────────────────────
type P1 = Partial<string[]>;              // (string | undefined)[]    ← still an array
type P2 = Partial<[number, string]>;      // [(number | undefined)?, (string | undefined)?]
type P3 = NonHomo<string[]>;              // an OBJECT with number-ish keys — array-ness lost

// ── Union distribution ──────────────────────────────────────────────────────
type Success = { ok: true;  data: string };
type Failure = { ok: false; error: string };
type P4 = Readonly<Success | Failure>;
// = Readonly<Success> | Readonly<Failure>   ← still a discriminated union, still narrowable

// ── Primitive pass-through ──────────────────────────────────────────────────
type P5 = Partial<number>;                // number  (not {})
```

Adding an `as` clause keeps homomorphism as long as the key source is still `keyof T`:

```ts
type StillHomomorphic<T> = { [K in keyof T as `raw_${string & K}`]: T[K] };  // ✅ homomorphic
```

Practical rule: **write `[K in keyof T]` whenever you can.** Reach for `[K in keyof T & string]` or `[K in Exclude<keyof T, X>]` only when you specifically want to lose modifiers or filter keys — and know that you are giving up modifier preservation.

### Concept 6 — Deep / recursive mapped types

Mapped types can call themselves. This is how you build `DeepPartial`, `DeepReadonly`, and immutable config trees.

```ts
// ── DeepReadonly — freeze an entire nested structure ────────────────────────
type DeepReadonly<T> =
  T extends (infer E)[]                       // arrays → ReadonlyArray of deep-readonly elements
    ? ReadonlyArray<DeepReadonly<E>>
    : T extends Date | RegExp | Function       // leave built-ins alone
      ? T
      : T extends object
        ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
        : T;                                   // primitives pass through

interface AppConfig {
  server: { port: number; host: string; cors: { origins: string[] } };
  database: { url: string; poolSize: number };
  featureFlags: string[];
}

type FrozenAppConfig = DeepReadonly<AppConfig>;
declare const config: FrozenAppConfig;
config.server.port = 4000;             // ❌ Error: Cannot assign to 'port', it is read-only
config.server.cors.origins.push("x");  // ❌ Error: push does not exist on readonly string[]

// ── DeepPartial — for merging partial overrides into defaults ───────────────
type DeepPartial<T> =
  T extends (infer E)[]
    ? DeepPartial<E>[]
    : T extends Date | RegExp | Function
      ? T
      : T extends object
        ? { [K in keyof T]?: DeepPartial<T[K]> }
        : T;

function mergeConfig(defaults: AppConfig, overrides: DeepPartial<AppConfig>): AppConfig {
  return {
    server:   { ...defaults.server,   ...overrides.server,
                cors: { ...defaults.server.cors, ...overrides.server?.cors } },
    database: { ...defaults.database, ...overrides.database },
    featureFlags: overrides.featureFlags ?? defaults.featureFlags,
  };
}

// ✅ Only the leaf you care about:
mergeConfig(defaults, { server: { port: 8080 } });
// ✅ Nested leaf:
mergeConfig(defaults, { server: { cors: { origins: ["https://app.example.com"] } } });
// ❌ Typo still caught:
mergeConfig(defaults, { server: { prot: 8080 } });  // Error: 'prot' does not exist
```

The `T extends Date | RegExp | Function ? T` guard matters. Without it, `DeepReadonly<Date>` maps over `Date`'s methods and produces a useless object type that no real `Date` is assignable to.

### Concept 7 — Combining mapped types with conditional types

The value expression of a mapped type is an ordinary type expression, so conditional types (doc **44**) plug straight in. This is where the real power lives.

```ts
interface UserRecord {
  userId:      number;
  email:       string;
  createdAt:   Date;
  lastLoginAt: Date | null;
  tags:        string[];
  metadata:    Record<string, unknown>;
}

// ── Turn Dates into ISO strings for JSON serialisation ──────────────────────
type JsonSafe<T> = {
  [K in keyof T]: T[K] extends Date
    ? string
    : T[K] extends Date | null
      ? string | null
      : T[K];
};
type UserJson = JsonSafe<UserRecord>;
// = { userId: number; email: string; createdAt: string; lastLoginAt: string | null;
//     tags: string[]; metadata: Record<string, unknown> }

// ── Pick only the keys whose value matches a type ───────────────────────────
type KeysOfType<T, V> = { [K in keyof T]: T[K] extends V ? K : never }[keyof T];
//                       ^ build a map of key→(key|never), then index it with keyof T
//                         to collapse it into a union, dropping the nevers.

type StringKeys = KeysOfType<UserRecord, string>;   // "email"
type DateKeys   = KeysOfType<UserRecord, Date>;     // "createdAt"

// ── Same thing with `as` remapping — usually clearer ────────────────────────
type PickByType<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};
type OnlyDates = PickByType<UserRecord, Date>;      // { createdAt: Date }

// ── Split required vs optional keys ─────────────────────────────────────────
type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K
}[keyof T];
type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never
}[keyof T];

interface CreateUserDto { email: string; displayName?: string; isAdmin?: boolean }
type Req = RequiredKeys<CreateUserDto>;  // "email"
type Opt = OptionalKeys<CreateUserDto>;  // "displayName" | "isAdmin"
```

---

## Example 1 — basic

```ts
// Deriving every DTO an API needs from ONE domain model.

interface User {
  userId:       number;
  email:        string;
  passwordHash: string;
  displayName:  string;
  isAdmin:      boolean;
  createdAt:    Date;
  updatedAt:    Date;
}

// ── 1. What the client may send on POST /users ──────────────────────────────
// Server-owned fields removed; password comes in plaintext, not as a hash.
type CreateUserDto =
  Omit<User, "userId" | "passwordHash" | "createdAt" | "updatedAt">
  & { password: string };
// = { email: string; displayName: string; isAdmin: boolean; password: string }

// ── 2. What the client may send on PATCH /users/:userId ─────────────────────
// Every field optional — Partial is a mapped type adding `?` to each key.
type UpdateUserDto = Partial<Omit<User, "userId" | "passwordHash" | "createdAt" | "updatedAt">>;
// = { email?: string; displayName?: string; isAdmin?: boolean }

// ── 3. What the API sends back — no secrets, Dates as ISO strings ───────────
type Serialize<T> = {
  [K in keyof T]: T[K] extends Date ? string : T[K];
};
type UserResponse = Serialize<Omit<User, "passwordHash">>;
// = { userId: number; email: string; displayName: string; isAdmin: boolean;
//     createdAt: string; updatedAt: string }

// ── 4. What the DB driver hands back — every column may be NULL ─────────────
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};
type UserRow = Nullable<User>;

// ── 5. A frozen, deeply-immutable view for caches ───────────────────────────
type Frozen<T> = {
  readonly [K in keyof T]: T[K];
};
type CachedUser = Frozen<UserResponse>;

// ── Using them ──────────────────────────────────────────────────────────────

function toUserResponse(user: User): UserResponse {
  const { passwordHash, ...rest } = user;   // strip the secret
  return {
    ...rest,
    createdAt: rest.createdAt.toISOString(),
    updatedAt: rest.updatedAt.toISOString(),
  };
}

function applyUpdate(user: User, requestBody: UpdateUserDto): User {
  return {
    ...user,
    // `??` keeps the old value when the key is absent from the PATCH body:
    email:       requestBody.email       ?? user.email,
    displayName: requestBody.displayName ?? user.displayName,
    isAdmin:     requestBody.isAdmin     ?? user.isAdmin,
    updatedAt:   new Date(),
  };
}

function rowToUser(row: UserRow): User | null {
  // Every field is `| null` here, so the compiler forces you to prove it isn't:
  if (row.userId === null || row.email === null || row.passwordHash === null) return null;
  if (row.displayName === null || row.isAdmin === null) return null;
  if (row.createdAt === null || row.updatedAt === null) return null;

  return {
    userId:       row.userId,        // narrowed to number
    email:        row.email,         // narrowed to string
    passwordHash: row.passwordHash,
    displayName:  row.displayName,
    isAdmin:      row.isAdmin,
    createdAt:    row.createdAt,
    updatedAt:    row.updatedAt,
  };
}

// Add `phoneNumber: string` to User and every derived type above updates,
// and `toUserResponse` / `applyUpdate` become compile errors until you handle it.
```

---

## Example 2 — real world backend use case

```ts
// A fully type-safe event bus + a typed query builder, both built on mapped types.

// ════════════════════════════════════════════════════════════════════════════
// PART 1 — Typed event bus: one EventMap, everything else derived
// ════════════════════════════════════════════════════════════════════════════

// The single source of truth: event name → payload shape.
interface EventMap {
  "user.created":     { userId: number; email: string; createdAt: Date };
  "user.deleted":     { userId: number; deletedBy: number };
  "order.placed":     { orderId: string; userId: number; totalCents: number };
  "payment.failed":   { paymentId: string; userId: number; reason: string };
  "auth.login":       { userId: number; authToken: string; ipAddress: string };
}

// ── Derive the handler map: same keys, values become handler functions ──────
type EventHandlers = {
  [E in keyof EventMap]: (payload: EventMap[E]) => void | Promise<void>;
};

// ── Derive a registry that holds ARRAYS of handlers, all optional ───────────
type HandlerRegistry = {
  [E in keyof EventMap]?: Array<EventHandlers[E]>;
};

// ── Derive "on*" method names via key remapping ─────────────────────────────
type DotToPascal<S extends string> =
  S extends `${infer Head}.${infer Tail}`
    ? `${Capitalize<Head>}${DotToPascal<Tail>}`
    : Capitalize<S>;

type EventListenerApi = {
  [E in keyof EventMap as `on${DotToPascal<string & E>}`]:
    (handler: (payload: EventMap[E]) => void) => void;
};
// = {
//     onUserCreated:   (handler: (p: { userId: number; email: string; createdAt: Date }) => void) => void;
//     onUserDeleted:   (handler: (p: { userId: number; deletedBy: number }) => void) => void;
//     onOrderPlaced:   (...) => void;
//     onPaymentFailed: (...) => void;
//     onAuthLogin:     (...) => void;
//   }

class TypedEventBus {
  private readonly registry: HandlerRegistry = {};

  // Generic over the event name — payload type follows automatically.
  on<E extends keyof EventMap>(event: E, handler: EventHandlers[E]): () => void {
    const list = (this.registry[event] ??= []) as Array<EventHandlers[E]>;
    list.push(handler);
    return () => {
      const index = list.indexOf(handler);
      if (index >= 0) list.splice(index, 1);
    };
  }

  async emit<E extends keyof EventMap>(event: E, payload: EventMap[E]): Promise<void> {
    const handlers = this.registry[event] as Array<EventHandlers[E]> | undefined;
    if (!handlers) return;
    await Promise.all(handlers.map((h) => h(payload)));
  }
}

const bus = new TypedEventBus();

bus.on("user.created", (payload) => {
  // payload is inferred as { userId: number; email: string; createdAt: Date }
  console.log(`Welcome email queued for ${payload.email} (user ${payload.userId})`);
});

bus.on("payment.failed", (payload) => {
  console.warn(`Payment ${payload.paymentId} failed: ${payload.reason}`);
});

await bus.emit("order.placed", { orderId: "ord_9f2", userId: 42, totalCents: 4999 });
// ❌ await bus.emit("order.placed", { orderId: "ord_9f2" });      // missing fields
// ❌ await bus.emit("order.shipped", { /* ... */ });              // unknown event name

// ── Exhaustive handler table: the compiler forces a handler for EVERY event ──
const auditHandlers: EventHandlers = {
  "user.created":   (p) => auditLog("user.created",   p.userId, p.email),
  "user.deleted":   (p) => auditLog("user.deleted",   p.userId, String(p.deletedBy)),
  "order.placed":   (p) => auditLog("order.placed",   p.userId, p.orderId),
  "payment.failed": (p) => auditLog("payment.failed", p.userId, p.paymentId),
  "auth.login":     (p) => auditLog("auth.login",     p.userId, p.ipAddress),
  // Add a new key to EventMap → this object literal becomes a compile error.
};

declare function auditLog(event: string, userId: number, detail: string): void;

// ════════════════════════════════════════════════════════════════════════════
// PART 2 — Typed query builder: WHERE / ORDER BY / SELECT derived from the row
// ════════════════════════════════════════════════════════════════════════════

interface UserTable {
  userId:      number;
  email:       string;
  displayName: string;
  isAdmin:     boolean;
  createdAt:   Date;
}

// ── Comparison operators, restricted per value type ─────────────────────────
type Comparators<V> =
  V extends number | Date
    ? { eq?: V; neq?: V; gt?: V; gte?: V; lt?: V; lte?: V; in?: V[] }
    : V extends string
      ? { eq?: V; neq?: V; like?: string; in?: V[] }
      : { eq?: V; neq?: V };   // booleans: only equality makes sense

// ── WHERE clause: every column optional, value replaced by its operator set ─
type WhereClause<T> = {
  [K in keyof T]?: Comparators<T[K]> | T[K];   // allow shorthand `{ userId: 42 }` too
};

// ── ORDER BY: every column optional, value is a direction ───────────────────
type OrderBy<T> = {
  [K in keyof T]?: "ASC" | "DESC";
};

// ── SELECT: every column optional boolean; used to narrow the result type ───
type Projection<T> = {
  [K in keyof T]?: true;
};

// Given a projection, compute the row type it returns:
type Projected<T, P extends Projection<T>> = {
  [K in keyof T as P[K] extends true ? K : never]: T[K];
};

class QueryBuilder<T> {
  private whereClause: WhereClause<T> = {};
  private orderClause: OrderBy<T> = {};
  private limitValue = 100;

  constructor(private readonly tableName: string) {}

  where(clause: WhereClause<T>): this {
    this.whereClause = { ...this.whereClause, ...clause };
    return this;
  }

  orderBy(clause: OrderBy<T>): this {
    this.orderClause = { ...this.orderClause, ...clause };
    return this;
  }

  limit(count: number): this {
    this.limitValue = count;
    return this;
  }

  // The projection narrows the RESULT type, not just the SQL:
  async select<P extends Projection<T>>(projection: P): Promise<Array<Projected<T, P>>> {
    const columns = Object.keys(projection).join(", ");
    const sql = `SELECT ${columns} FROM ${this.tableName} ${this.buildWhere()} ${this.buildOrder()} LIMIT ${this.limitValue}`;
    return runQuery(sql) as Promise<Array<Projected<T, P>>>;
  }

  private buildWhere(): string {
    const parts = Object.entries(this.whereClause).map(([column, condition]) => {
      if (condition === null || typeof condition !== "object" || condition instanceof Date) {
        return `${column} = ${quote(condition)}`;
      }
      return Object.entries(condition as Record<string, unknown>)
        .map(([op, value]) => `${column} ${SQL_OPS[op] ?? "="} ${quote(value)}`)
        .join(" AND ");
    });
    return parts.length ? `WHERE ${parts.join(" AND ")}` : "";
  }

  private buildOrder(): string {
    const parts = Object.entries(this.orderClause).map(([c, d]) => `${c} ${d}`);
    return parts.length ? `ORDER BY ${parts.join(", ")}` : "";
  }
}

const SQL_OPS: Record<string, string> = {
  eq: "=", neq: "<>", gt: ">", gte: ">=", lt: "<", lte: "<=", like: "LIKE", in: "IN",
};

declare function quote(value: unknown): string;
declare function runQuery(sql: string): Promise<unknown[]>;

// ── Usage — everything checked, result type computed from the projection ────

const admins = await new QueryBuilder<UserTable>("users")
  .where({ isAdmin: true, createdAt: { gte: new Date("2026-01-01") } })
  .orderBy({ createdAt: "DESC" })
  .limit(50)
  .select({ userId: true, email: true });
// admins: Array<{ userId: number; email: string }>   ← displayName is NOT on the type

admins[0].email;        // ✅
// admins[0].displayName;  // ❌ Error: Property 'displayName' does not exist

// ❌ new QueryBuilder<UserTable>("users").where({ isAdmin: { gt: true } });
//    Error — Comparators<boolean> has no `gt`
// ❌ new QueryBuilder<UserTable>("users").where({ emial: { eq: "x" } });
//    Error — 'emial' does not exist in WhereClause<UserTable>
// ❌ new QueryBuilder<UserTable>("users").orderBy({ createdAt: "SIDEWAYS" });
//    Error — not "ASC" | "DESC"
```

---

## Going deeper

### The `[keyof T]` indexing trick — turning a map into a union

The single most useful non-obvious pattern: build a mapped type whose *values* are keys, then index it by `keyof T` to collapse the whole thing into a union. `never` values vanish from that union automatically.

```ts
interface ServiceRegistry {
  userService:    UserService;
  orderService:   OrderService;
  cacheClient:    RedisClient;
  logger:         Logger;
  metricsCounter: number;
}

// Step 1: map each key to either itself or never.
type Step1 = { [K in keyof ServiceRegistry]: ServiceRegistry[K] extends object ? K : never };
// = { userService: "userService"; orderService: "orderService"; cacheClient: "cacheClient";
//     logger: "logger"; metricsCounter: never }

// Step 2: index the whole map by keyof — the values become a union.
type ObjectKeys = Step1[keyof ServiceRegistry];
// = "userService" | "orderService" | "cacheClient" | "logger"
//   (never is absorbed into a union — `X | never` is just `X`)

// Both steps in one:
type KeysMatching<T, V> = { [K in keyof T]-?: T[K] extends V ? K : never }[keyof T];

type FunctionKeys<T> = KeysMatching<T, (...args: never[]) => unknown>;
type MethodsOf<T> = Pick<T, FunctionKeys<T>>;
```

The `-?` in `KeysMatching` is not cosmetic: without it, an optional property `K?` makes the value type `K | undefined`, and `undefined` leaks into the resulting union.

### `Pick<T, K>` is not homomorphic-safe with an arbitrary `K`

`Omit` is defined as `Pick<T, Exclude<keyof T, K>>`. Because `Exclude<...>` is not literally `keyof T`, `Omit` **is** still special-cased by the compiler to preserve modifiers in recent versions — but hand-rolled equivalents often are not.

```ts
interface Entity {
  readonly entityId: string;
  name?: string;
  version: number;
}

type ViaPick = Pick<Entity, "entityId" | "name">;
// = { readonly entityId: string; name?: string }   ← modifiers preserved (Pick is homomorphic in K)

// A hand-rolled "omit" that iterates keyof T with a filter is homomorphic and safe:
type SafeOmit<T, K extends keyof T> = { [P in keyof T as P extends K ? never : P]: T[P] };
type S = SafeOmit<Entity, "version">;
// = { readonly entityId: string; name?: string }   ← modifiers preserved

// A version that rebuilds from a computed key union loses them:
type UnsafeOmit<T, K extends keyof T> = { [P in Exclude<keyof T, K> & string]: T[P] };
type U = UnsafeOmit<Entity, "version">;
// = { entityId: string; name: string | undefined }   ← readonly and ? gone
```

### `Omit` does not check that the key exists

A real footgun. `Omit<T, K>` types `K` as `keyof any`, not `keyof T`, so typos silently do nothing.

```ts
interface User { userId: number; email: string; passwordHash: string }

type Safe = Omit<User, "passwordHsah">;   // typo!
// = { userId: number; email: string; passwordHash: string }
//   ← NO ERROR. The secret is still in the type. This has leaked real passwords.

// ✅ Write a strict version and use it everywhere:
type StrictOmit<T, K extends keyof T> = { [P in keyof T as P extends K ? never : P]: T[P] };
type Safe2 = StrictOmit<User, "passwordHsah">;
// ❌ Error: Type '"passwordHsah"' does not satisfy the constraint 'keyof User'
```

### Mapped types over index signatures and unions of keys

```ts
// Index signatures survive mapping:
type Headers = { [header: string]: string };
type OptionalHeaders = Partial<Headers>;   // { [header: string]: string | undefined }

// keyof on an index signature gives the index type, not literals:
type H = keyof Headers;                    // string | number  (number because "1" indexes too)

// A mapped type over a union type parameter distributes only if homomorphic:
type Wrapped<T> = { [K in keyof T]: { value: T[K] } };
type W = Wrapped<{ a: 1 } | { b: 2 }>;
// = { a: { value: 1 } } | { b: { value: 2 } }   ← distributed, stayed a union

// Non-homomorphic over the same union collapses to the shared keys (none):
type Collapsed<T> = { [K in keyof T & string]: T[K] };
type C = Collapsed<{ a: 1 } | { b: 2 }>;
// = {}   ← keyof (A|B) is the INTERSECTION of keys, which is never here
```

### Performance: mapped types are lazy but not free

The compiler evaluates mapped types on demand and caches instantiations, but deep recursion over wide types is where builds get slow.

```ts
// ⚠️ Recursion over a wide, deeply nested type instantiated at many call sites
// is the classic cause of "tsc is slow" in large backends.
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

// Mitigations that actually help:
// 1. Cache the result in a named type alias instead of re-instantiating inline:
type ConfigOverrides = DeepPartial<AppConfig>;         // computed once
function load(o: ConfigOverrides) {}                    // ✅
// function load(o: DeepPartial<AppConfig>) {}          // recomputed per signature site

// 2. Bound the recursion depth explicitly for very deep trees:
type DeepPartialN<T, D extends number = 5> =
  D extends 0 ? T
  : T extends object ? { [K in keyof T]?: DeepPartialN<T[K], Decrement<D>> }
  : T;
type Decrement<N extends number> =
  N extends 5 ? 4 : N extends 4 ? 3 : N extends 3 ? 2 : N extends 2 ? 1 : 0;

// 3. Add `interface` boundaries — named interfaces are cheaper to compare
//    than large anonymous object literal types produced by mapping.
```

There is also a hard limit: TypeScript errors with *"Type instantiation is excessively deep and possibly infinite"* at around 50 levels of recursive instantiation (100 in some code paths). Deep recursive mapped types over recursive data structures hit this.

### `as` remapping cannot produce duplicate keys with conflicting types

If two source keys remap to the same target key, the compiler unions the value types rather than erroring — which silently loses information.

```ts
interface Metrics {
  requestCountTotal:  number;
  requestCountFailed: number;
  latencyP99Ms:       string;
}

type StripSuffix<T> = {
  [K in keyof T as K extends `${infer Base}Total` | `${infer Base}Failed` ? Base : K]: T[K];
};
type M = StripSuffix<Metrics>;
// = { requestCount: number; latencyP99Ms: string }
//   Both source keys collapsed into `requestCount`. No error, no warning.
```

### Mapped types cannot add methods or call signatures

A mapped type produces only *properties*. A source type's call signatures, construct signatures, and index signature `readonly`ness behave differently:

```ts
type Fn = { (requestBody: string): void; timeoutMs: number };

type Mapped = { [K in keyof Fn]: Fn[K] };
// = { timeoutMs: number }   ← the CALL SIGNATURE is gone; only properties survive.

// If you need to preserve callability, intersect it back:
type MappedCallable = Mapped & ((requestBody: string) => void);
```

The same applies to `Partial<SomeClass>` — you get the properties and methods as properties, but you lose the constructor, `private` members become inaccessible in a different way, and the result is no longer `instanceof`-compatible.

### `satisfies` pairs beautifully with mapped types

```ts
type StatusHandlers = { [S in "pending" | "settled" | "failed"]: (paymentId: string) => void };

// `satisfies` checks exhaustiveness WITHOUT widening the inferred value types:
const handlers = {
  pending: (paymentId: string) => queueRetry(paymentId),
  settled: (paymentId: string) => markComplete(paymentId),
  failed:  (paymentId: string) => alertOncall(paymentId),
} satisfies StatusHandlers;

// handlers.pending keeps its exact inferred type, AND a missing key is an error.
declare function queueRetry(id: string): void;
declare function markComplete(id: string): void;
declare function alertOncall(id: string): void;
```

---

## Common mistakes

### Mistake 1 — Losing modifiers by breaking homomorphism

```ts
interface Session {
  readonly sessionId: string;
  userId: number;
  expiresAt?: Date;
}

// ❌ `keyof T & string` looks harmless but destroys readonly and ?:
type BadClone<T> = { [K in keyof T & string]: T[K] };
type Bad = BadClone<Session>;
// = { sessionId: string; userId: number; expiresAt: Date | undefined }
//   readonly gone (mutable session id!), optional gone (must now pass expiresAt)

const bad: Bad = { sessionId: "s1", userId: 1, expiresAt: undefined };
bad.sessionId = "hijacked";   // ✅ compiles — the readonly protection was silently dropped

// ✅ Keep the key source as literally `keyof T`:
type GoodClone<T> = { [K in keyof T]: T[K] };
type Good = GoodClone<Session>;
// = { readonly sessionId: string; userId: number; expiresAt?: Date }

const good: Good = { sessionId: "s1", userId: 1 };
// good.sessionId = "hijacked";  // ❌ Error: Cannot assign to 'sessionId', it is read-only
```

### Mistake 2 — Using `Omit` with an unchecked key (the leaked-secret bug)

```ts
interface User {
  userId:       number;
  email:        string;
  passwordHash: string;
  mfaSecret:    string;
}

// ❌ Omit accepts any key. A typo compiles and the secret stays in the response type:
type PublicUser = Omit<User, "passwordHash" | "mfaSecrets">;   // note the trailing 's'
// = { userId: number; email: string; mfaSecret: string }   ← LEAKED, no error

function toPublic(user: User): PublicUser {
  const { passwordHash, ...rest } = user;
  return rest;   // ✅ compiles — mfaSecret ships to the client
}

// ✅ Use a strict, key-checked omit built as a mapped type:
type StrictOmit<T, K extends keyof T> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

type PublicUser2 = StrictOmit<User, "passwordHash" | "mfaSecrets">;
// ❌ Error: Type '"passwordHash" | "mfaSecrets"' does not satisfy 'keyof User'

type PublicUser3 = StrictOmit<User, "passwordHash" | "mfaSecret">;
// = { userId: number; email: string }   ✅ correct, and typos are caught
```

### Mistake 3 — Forgetting `-?` when collecting keys by value type

```ts
interface UpdateUserDto {
  email?:       string;
  displayName?: string;
  isAdmin?:     boolean;
}

// ❌ Optional props make T[K] include undefined, so the extends check fails:
type BadStringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];
type Bad = BadStringKeys<UpdateUserDto>;
// = undefined
//   Because T["email"] is `string | undefined`, which does NOT extend `string`,
//   every branch produced `never`... and the optionality added `undefined` back.

// ✅ Strip optionality inside the map with -?:
type GoodStringKeys<T> = { [K in keyof T]-?: T[K] extends string ? K : never }[keyof T];
type Good = GoodStringKeys<UpdateUserDto>;
// = "email" | "displayName"   ✅
```

### Mistake 4 — Mapping over a type that contains `Date`, arrays, or functions without guarding

```ts
interface AuditEntry {
  entryId:   string;
  occurredAt: Date;
  changes:   string[];
  onCommit:  () => void;
}

// ❌ A naive deep map destroys Date and Function into useless property bags:
type BadDeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: BadDeepReadonly<T[K]> }
  : T;

type Bad = BadDeepReadonly<AuditEntry>;
// occurredAt becomes a readonly map of Date's METHODS — no real Date is assignable:
declare const bad: Bad;
// const d: Date = bad.occurredAt;   // ❌ Error — the Date-ness is gone
// bad.onCommit();                   // ❌ Error — call signature was dropped

// ✅ Short-circuit on built-ins and handle arrays explicitly:
type GoodDeepReadonly<T> =
  T extends (infer E)[]                                  ? ReadonlyArray<GoodDeepReadonly<E>>
  : T extends Date | RegExp | Function | Map<any, any>   ? T
  : T extends object                                     ? { readonly [K in keyof T]: GoodDeepReadonly<T[K]> }
  : T;

type Good = GoodDeepReadonly<AuditEntry>;
declare const good: Good;
const when: Date = good.occurredAt;   // ✅ still a Date
good.onCommit();                      // ✅ still callable
// good.changes.push("x");            // ❌ readonly string[] — correctly blocked
```

### Mistake 5 — Assuming `Partial<T>` makes a type safe to build incrementally

```ts
interface RequestContext {
  requestId: string;
  userId:    number;
  authToken: string;
}

// ❌ Partial gives you a mutable half-built object with no way to prove it's complete:
function buildContextBad(): RequestContext {
  const ctx: Partial<RequestContext> = {};
  ctx.requestId = crypto.randomUUID();
  ctx.userId = 42;
  // forgot authToken
  return ctx as RequestContext;   // the `as` cast lies — undefined at runtime
}

// ✅ Build it all at once so the compiler checks completeness:
function buildContextGood(userId: number, authToken: string): RequestContext {
  return {
    requestId: crypto.randomUUID(),
    userId,
    authToken,
  };   // ❌ missing any field is a compile error here
}

// ✅ Or use a builder whose type tracks which keys are filled:
class ContextBuilder<Filled extends keyof RequestContext = never> {
  private data: Partial<RequestContext> = {};

  set<K extends keyof RequestContext>(
    key: K,
    value: RequestContext[K],
  ): ContextBuilder<Filled | K> {
    this.data[key] = value;
    return this as unknown as ContextBuilder<Filled | K>;
  }

  // Only callable once every key has been set:
  build(this: ContextBuilder<keyof RequestContext>): RequestContext {
    return this.data as RequestContext;
  }
}

const ctx = new ContextBuilder()
  .set("requestId", "req_1")
  .set("userId", 42)
  .set("authToken", "tok_abc")
  .build();          // ✅ compiles

// new ContextBuilder().set("requestId", "req_1").build();
// ❌ Error: 'this' context is not assignable — authToken and userId still missing
```

---

## Practice exercises

### Exercise 1 — easy

Write these mapped types from scratch, without using any built-in utility type:

1. `Nullable<T>` — every property's type becomes `T[K] | null`
2. `Stringify<T>` — every property's type becomes `string` (for reading config from `process.env`)
3. `Optional<T>` — every property becomes optional, preserving `readonly`
4. `Mutable<T>` — strips `readonly` from every property
5. `EventFlags<K extends string>` — given a union of event names, produce `{ [name]: boolean }`

Then apply them to:

```ts
interface ServerConfig {
  readonly port:      number;
  readonly host:      string;
  maxConnections:     number;
  enableCompression:  boolean;
  shutdownTimeoutMs:  number;
}
```

and write `parseConfig(raw: Stringify<ServerConfig>): ServerConfig` that converts each string to the right runtime type and throws a descriptive error on a bad value.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a **validation schema type** derived from a domain model, using key remapping and conditional value types.

```ts
interface UserSchema {
  userId:      number;
  email:       string;
  displayName: string;
  age:         number;
  isAdmin:     boolean;
  createdAt:   Date;
  tags:        string[];
}

// Implement:
//
// 1. Validator<V> — a conditional type giving the allowed rule object per value type:
//      number  -> { type: "number";  min?: number; max?: number; integer?: boolean }
//      string  -> { type: "string";  minLength?: number; maxLength?: number; pattern?: RegExp }
//      boolean -> { type: "boolean" }
//      Date    -> { type: "date";    after?: Date; before?: Date }
//      T[]     -> { type: "array";   minItems?: number; itemValidator?: Validator<T> }
//
// 2. Schema<T> — a mapped type: every key of T maps to Validator<T[K]>,
//    plus a shared `required: boolean` on every rule.
//
// 3. ValidationErrors<T> — a mapped type where every key is optional and
//    the value is `string[]` (the messages for that field).
//
// 4. A `validate<T>(schema: Schema<T>, requestBody: unknown): ValidationErrors<T>` function.
//
// 5. WriteableSchema<T> — key-remap so every produced key is prefixed `validate`
//    and Capitalized: validateUserId, validateEmail, ... and the value is
//    `(value: T[K]) => string[]`.
//
// Then declare a full `userValidationSchema: Schema<UserSchema>` and show
// that omitting a key, or using `minLength` on `age`, is a compile error.
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **type-safe ORM entity layer** entirely out of mapped types.

```ts
// Given a table definition expressed as a plain type:

interface OrderTable {
  orderId:     string;
  userId:      number;
  totalCents:  number;
  status:      "pending" | "paid" | "shipped" | "cancelled";
  placedAt:    Date;
  shippedAt:   Date | null;
  notes:       string | null;
}

// Build ALL of the following as mapped types:
//
// 1. ColumnNames<T> — key-remapped to snake_case (orderId -> order_id) with the
//    same value types. Write the SnakeCase<S> template-literal helper yourself.
//
// 2. InsertShape<T, Generated extends keyof T> — columns in `Generated` become
//    optional; nullable columns (those whose type includes null) become optional;
//    everything else stays required.
//
// 3. UpdateShape<T, Immutable extends keyof T> — every column optional EXCEPT the
//    ones in `Immutable`, which are removed entirely (remap to never).
//
// 4. SelectShape<T, Columns extends keyof T> — Pick, but written by hand as a
//    homomorphic mapped type so modifiers are preserved.
//
// 5. WhereShape<T> — every column optional; value is `T[K] | { $in: T[K][] }`
//    and additionally `{ $gt/$gte/$lt/$lte: T[K] }` only when T[K] is number or Date,
//    and `{ $like: string }` only when T[K] is string.
//
// 6. RelationMap<T> — given
//      type Relations = { user: UserTable; items: OrderItemTable[] };
//    produce loaded/unloaded variants:
//      Loaded<T, R, K extends keyof R>  — T plus the relation keys in K, typed R[K]
//      i.e. `Loaded<OrderTable, Relations, "user">` has `user: UserTable`.
//
// 7. Repository<T> — a class generic over the table type exposing:
//      insert(row: InsertShape<T, "orderId" | "placedAt">): Promise<T>
//      update(id: string, patch: UpdateShape<T, "orderId" | "placedAt">): Promise<T>
//      find<C extends keyof T>(where: WhereShape<T>, columns: C[]): Promise<SelectShape<T, C>[]>
//      findOne(where: WhereShape<T>): Promise<T | null>
//
// Then instantiate `new Repository<OrderTable>("orders")` and demonstrate:
//   - inserting without orderId / placedAt / shippedAt / notes compiles
//   - attempting to update orderId is a compile error
//   - `find({ totalCents: { $gt: 1000 } }, ["orderId", "status"])` returns
//     Array<{ orderId: string; status: "pending" | "paid" | "shipped" | "cancelled" }>
//   - `find({ status: { $gt: "paid" } })` is a compile error (no $gt on a string column)
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Basic loop ──────────────────────────────────────────────────────────────
type Clone<T>    = { [K in keyof T]: T[K] };
type Values<T>   = T[keyof T];                       // union of all value types

// ── Modifiers ───────────────────────────────────────────────────────────────
type Opt<T>      = { [K in keyof T]?: T[K] };        // add ?          (= Partial<T>)
type Req<T>      = { [K in keyof T]-?: T[K] };       // remove ?       (= Required<T>)
type Ro<T>       = { readonly [K in keyof T]: T[K] };// add readonly   (= Readonly<T>)
type Mut<T>      = { -readonly [K in keyof T]: T[K] };// remove readonly

// ── Value transforms ────────────────────────────────────────────────────────
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Stringify<T>= { [K in keyof T]: string };
type Promised<T> = { [K in keyof T]: Promise<T[K]> };
type Getters<T>  = { [K in keyof T]: () => T[K] };

// ── Key remapping with `as` ─────────────────────────────────────────────────
type Prefix<T, P extends string> = { [K in keyof T as `${P}${Capitalize<string & K>}`]: T[K] };
type Drop<T, V>   = { [K in keyof T as T[K] extends V ? never : K]: T[K] };   // never = drop key
type KeepOnly<T, V> = { [K in keyof T as T[K] extends V ? K : never]: T[K] };

// ── Key union → union of key names (the [keyof T] trick) ────────────────────
type KeysMatching<T, V> = { [K in keyof T]-?: T[K] extends V ? K : never }[keyof T];

// ── Arbitrary key source ────────────────────────────────────────────────────
type Rec<K extends keyof any, V> = { [P in K]: V };  // = Record<K, V>

// ── Recursive ───────────────────────────────────────────────────────────────
type DeepReadonly<T> =
  T extends (infer E)[]                    ? ReadonlyArray<DeepReadonly<E>>
  : T extends Date | RegExp | Function      ? T
  : T extends object                        ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

| Construct | Meaning |
|---|---|
| `[K in keyof T]` | Loop over every key of `T` — **homomorphic**, preserves modifiers |
| `[K in "a" \| "b"]` | Loop over an explicit key union — non-homomorphic |
| `[K in keyof T as NewKey]` | Rename the produced key; `never` **drops** it |
| `?` / `-?` | Add / remove optionality (`-?` also removes `undefined`) |
| `readonly` / `-readonly` | Add / remove immutability |
| `+?`, `+readonly` | Explicit "add" form — same as the bare modifier |
| `T[K]` | Indexed access — the value type at key `K` |
| `T[keyof T]` | Union of all value types; collapses a key-map into a union |
| `Partial<T>` | `{ [K in keyof T]?: T[K] }` |
| `Required<T>` | `{ [K in keyof T]-?: T[K] }` |
| `Readonly<T>` | `{ readonly [K in keyof T]: T[K] }` |
| `Pick<T, K>` | `{ [P in K]: T[P] }` |
| `Record<K, V>` | `{ [P in K]: V }` |
| `Omit<T, K>` | `Pick<T, Exclude<keyof T, K>>` — **does not check `K` against `keyof T`** |
| Homomorphic bonus | Distributes over arrays, tuples, and unions; passes primitives through |

---

## Connected topics

- **32 — Utility types** — `Partial`, `Readonly`, `Pick`, `Record` are all mapped types; this doc shows how to build and extend them.
- **44 — Conditional types** — `T[K] extends U ? X : Y` inside a mapped type's value position is where transformations like `Serialize<T>` and `Comparators<V>` come from.
- **42 — Discriminated unions** — mapped types build exhaustive `Record<Kind, Handler>` tables from a union's tag values, so a new variant is a compile error.
- **31 — keyof and indexed access types** — `keyof T` and `T[K]` are the two primitives every mapped type is made of.
