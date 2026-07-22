# 45 — Template Literal Types

## What is this?

A **template literal type** is a type built the same way you build a template literal *string* in JavaScript — with backticks and `${}` — except the slots hold **types**, not values.

```ts
type Protocol = "http" | "https";
type Host     = "api.example.com";

type BaseUrl = `${Protocol}://${Host}`;
// = "http://api.example.com" | "https://api.example.com"
```

At the value level, `` `${protocol}://${host}` `` produces a string. At the type level, `` `${Protocol}://${Host}` `` produces the **set of all strings** that could result from every combination of the members of `Protocol` and `Host`.

Three things make them powerful:

1. They **distribute over unions** — a union in any slot multiplies the result (a cross product).
2. They can hold **wide types** (`string`, `number`, `bigint`, `boolean`) as an open-ended pattern — `` `/api/${string}` `` matches any string that starts with `/api/`.
3. They can be **pattern-matched** with `infer`, letting you pull structured type information back *out* of a string type.

```ts
type ApiRoute      = `/api/${string}`;                      // open-ended pattern
type UserIdParam   = "/users/:userId";
type ExtractParam<T extends string> =
  T extends `${string}:${infer P}` ? P : never;
type Param = ExtractParam<UserIdParam>;                     // "userId"
```

## Why does it matter?

Backend code is drowning in **strings that have structure**, and until template literal types existed, TypeScript treated all of them as the same featureless `string`:

- Route paths — `/api/v1/users/:userId` — the `:userId` segment is meaningful, but `string` doesn't know that.
- Event names — `user.created`, `order.placed`, `payment.failed` — a namespace, a dot, an action.
- Environment variables — `APP_DATABASE_URL`, `APP_REDIS_HOST` — a prefix plus a key.
- SQL/ORM column paths — `"users.email"`, `"orders.total_cents"` — a table, a dot, a column.
- Handler and hook names — a field `email` implies a hook `onEmailChange`.
- Redis/cache keys — `user:42:profile`, `session:abc123`.
- HTTP header names, feature flag keys, permission strings like `orders:read`.

Every one of these is a place where a typo compiles fine and blows up in production at 3am. Template literal types turn all of them into **compile-time-checked, autocompleted** values, derived from a single source of truth — usually a union you already have.

The second, bigger reason: template literal types are how you write types that **transform other types' names**. `Getters<UserSchema>` producing `{ getUserId(): number; getEmail(): string }` is not possible without them. Combined with mapped types (43) and conditional types (44) they turn TypeScript's type system into a real string-processing language.

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: every structured string is just an opaque blob ────────────────

const EVENT_PREFIX = "user";

// Emitting an event — the name is assembled by hand, nothing checks it:
function emit(eventName, payload) {
  bus.publish(eventName, payload);
}

emit(`${EVENT_PREFIX}.created`, { userId: 42 });   // ok
emit(`${EVENT_PREFIX}.creatd`, { userId: 42 });    // typo — silently never delivered
emit("usr.created", { userId: 42 });               // typo — silently never delivered
emit("user.created", { usrId: 42 });               // payload wrong too, nobody notices

// Subscribing — same problem, mirrored:
bus.subscribe("order.plced", handler);             // handler never fires. Debug that for a day.

// Config keys — prefix assembled by string concat, no validation:
function readEnv(key) {
  return process.env[`APP_${key}`];                // APP_DATABSE_URL → undefined → crash later
}
const dbUrl = readEnv("DATABSE_URL");              // undefined, no error until the pool opens

// Route registration — path params are documented in a comment, at best:
app.get("/api/v1/users/:userId/orders/:orderId", (req, res) => {
  const userId  = req.params.userId;               // any
  const orderId = req.params.ordrId;               // undefined — typo, no error
  const tenant  = req.params.tenantId;             // undefined — param doesn't even exist
});

// Column selection — the ORM takes strings, and hopes:
const rows = await db.select(["users.email", "users.emial", "orders.totl"]);
// Runtime SQL error, or worse: silently null columns.
```

```ts
// ── TypeScript: the structure of the string becomes part of the type ──────────

type EntityName = "user" | "order" | "payment";
type ActionName = "created" | "updated" | "deleted";

// One line generates all 9 valid event names — and only those 9:
type DomainEvent = `${EntityName}.${ActionName}`;
// "user.created" | "user.updated" | "user.deleted"
// | "order.created" | "order.updated" | "order.deleted"
// | "payment.created" | "payment.updated" | "payment.deleted"

declare function emit<E extends DomainEvent>(eventName: E, payload: EventPayload<E>): void;

emit("user.created", { userId: 42 });   // ✅ compiles, payload type checked against the name
// emit("user.creatd", { userId: 42 }); // ❌ Argument of type '"user.creatd"' is not
//                                      //    assignable to parameter of type DomainEvent
// emit("usr.created", { userId: 42 }); // ❌ same — caught at compile time

// Env vars — the prefix lives in the type, keys are checked and autocompleted:
type EnvKey = "DATABASE_URL" | "REDIS_HOST" | "JWT_SECRET" | "PORT";
type PrefixedEnvKey = `APP_${EnvKey}`;
// "APP_DATABASE_URL" | "APP_REDIS_HOST" | "APP_JWT_SECRET" | "APP_PORT"

declare const env: Record<PrefixedEnvKey, string>;
const databaseUrl = env.APP_DATABASE_URL;   // ✅ string, autocompleted
// const bad      = env.APP_DATABSE_URL;    // ❌ Property 'APP_DATABSE_URL' does not exist

// Route params — extracted from the path string itself, by the compiler:
type PathParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | PathParams<Rest>
    : Path extends `${string}:${infer Param}`
      ? Param
      : never;

type OrderRouteParams = PathParams<"/api/v1/users/:userId/orders/:orderId">;
// "userId" | "orderId"  ← derived from the literal path, no duplication

declare function get<Path extends string>(
  path: Path,
  handler: (req: { params: Record<PathParams<Path>, string> }) => void,
): void;

get("/api/v1/users/:userId/orders/:orderId", (req) => {
  const userId  = req.params.userId;    // ✅ string
  const orderId = req.params.orderId;   // ✅ string
  // const bad  = req.params.ordrId;    // ❌ Property 'ordrId' does not exist
});

// Column paths — generated from the schema, so they cannot drift from it:
interface UserSchema  { id: number; email: string; createdAt: Date }
interface OrderSchema { id: number; userId: number; totalCents: number }

type ColumnPath =
  | `users.${keyof UserSchema & string}`
  | `orders.${keyof OrderSchema & string}`;
// "users.id" | "users.email" | "users.createdAt" | "orders.id" | "orders.userId" | "orders.totalCents"

declare function select(columns: ColumnPath[]): Promise<unknown[]>;
select(["users.email", "orders.totalCents"]);  // ✅
// select(["users.emial"]);                    // ❌ caught at compile time
```

The revelation: you stop *writing* these strings and start *deriving* them. Add a field to `UserSchema` and `ColumnPath` grows automatically. Rename `totalCents` and every stale usage becomes a compile error instantly.

---

## Syntax

```ts
// ── Basic interpolation of literal types ────────────────────────────────────
type Version = "v1";                          // a string literal type
type Api = `/api/${Version}`;                 // "/api/v1"

// ── Union in a slot → distributes into a union of results ───────────────────
type Method = "GET" | "POST";                 // 2 members
type Route  = `${Method} /users`;             // "GET /users" | "POST /users"

// ── Multiple union slots → cross product (2 × 3 = 6 members) ────────────────
type Env  = "dev" | "prod";
type Svc  = "api" | "worker" | "cron";
type Host = `${Svc}.${Env}.internal`;         // 6 members

// ── Wide primitives make an open-ended pattern (not a finite union) ──────────
type AnyApiPath  = `/api/${string}`;          // any string starting with "/api/"
type CacheKey    = `user:${number}`;          // "user:1", "user:99999", ...
type FlagKey     = `flag_${boolean}`;         // "flag_true" | "flag_false"

const p1: AnyApiPath = "/api/users/42";       // ✅ matches the pattern
// const p2: AnyApiPath = "/users/42";        // ❌ does not start with "/api/"

// ── Intrinsic string manipulation types (built into the compiler) ───────────
type A = Uppercase<"user_id">;                // "USER_ID"
type B = Lowercase<"USER_ID">;                // "user_id"
type C = Capitalize<"email">;                 // "Email"
type D = Uncapitalize<"Email">;               // "email"

// ── Combining with mapped types — key remapping via `as` ────────────────────
type Getters<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
};
type UserGetters = Getters<{ userId: number; email: string }>;
// { getUserId: () => number; getEmail: () => string }

// ── Pattern matching with `infer` inside a template literal ─────────────────
type EventEntity<E extends string> =
  E extends `${infer Entity}.${string}` ? Entity : never;
type E1 = EventEntity<"order.placed">;        // "order"

// ── Constraining a generic to a pattern ─────────────────────────────────────
declare function fetchPath<P extends `/api/${string}`>(path: P): Promise<unknown>;
fetchPath("/api/users");                      // ✅
// fetchPath("/users");                       // ❌ not assignable to `/api/${string}`
```

---

## How it works — concept by concept

### Concept 1 — Interpolation and what is allowed in a slot

A template literal type is written with backticks; each `${...}` slot holds a **type**. The compiler produces the set of strings you get by substituting every possible value of every slot.

Slots may hold: string literal types, numeric literal types, boolean/`bigint` literal types, unions of those, and the wide primitives `string | number | bigint | boolean | null | undefined`. Object types are **not** allowed.

```ts
type Ok1 = `id-${string}`;       // open pattern
type Ok2 = `id-${number}`;       // "id-0" | "id-1" | ... (conceptually infinite; a pattern)
type Ok3 = `id-${boolean}`;      // "id-true" | "id-false" — boolean is a 2-member union
type Ok4 = `id-${null}`;         // "id-null"
type Ok5 = `id-${undefined}`;    // "id-undefined"

// ❌ Not allowed — objects have no string form the compiler will commit to:
// type Bad = `id-${{ userId: number }}`;
//   Error: Type '{ userId: number; }' is not assignable to type
//   'string | number | bigint | boolean | null | undefined'.

// Numeric literals are converted to their string form:
type StatusKey = `status_${200 | 404 | 500}`;   // "status_200" | "status_404" | "status_500"

// A wide slot swallows everything after it in terms of narrowing power:
type LooseKey = `user:${string}`;
const k1: LooseKey = "user:42";                 // ✅
const k2: LooseKey = "user:";                   // ✅ — `string` includes the empty string!
// const k3: LooseKey = "session:42";           // ❌
```

That last case matters: `` `user:${string}` `` accepts `"user:"` because `string` includes `""`. If you need "at least one character", you cannot express it directly — you validate at runtime or use a branded type (see 47 if present in your series, or a `unique symbol` brand).

### Concept 2 — Distribution over unions: the cross product

When more than one slot holds a union, the result is the **cartesian product** of all slots. This is the single most useful property of template literal types, and also the single easiest way to blow up your compiler.

```ts
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";      // 5
type Resource   = "users" | "orders" | "payments" | "sessions";     // 4

type RouteKey = `${HttpMethod} /api/${Resource}`;                   // 5 × 4 = 20 members
// "GET /api/users" | "GET /api/orders" | ... | "DELETE /api/sessions"

// The handler registry is now exhaustively typed:
type RouteRegistry = Record<RouteKey, (req: Request) => Promise<Response>>;
// Every one of the 20 combinations must be implemented — missing one is a compile error.

// Three unions → the product of all three:
type Env     = "dev" | "staging" | "prod";        // 3
type Service = "api" | "worker";                  // 2
type Region  = "us-east-1" | "eu-west-1";         // 2
type Cluster = `${Service}-${Env}-${Region}`;     // 3 × 2 × 2 = 12 members

// ⚠️ The product grows multiplicatively. This is 10 × 10 × 10 × 10 = 10,000 members:
type Digit = "0"|"1"|"2"|"3"|"4"|"5"|"6"|"7"|"8"|"9";
type FourDigitCode = `${Digit}${Digit}${Digit}${Digit}`;
// Legal, but TypeScript caps union sizes at 100,000 members. Add one more digit
// (100,000) and you get: "Expression produces a union type that is too complex to represent."
```

Distribution also happens when a **naked type parameter** is interpolated in a generic:

```ts
type Prefixed<T extends string> = `app.${T}`;
type Result = Prefixed<"users" | "orders">;    // "app.users" | "app.orders" — distributed
```

### Concept 3 — The four intrinsic string types

TypeScript ships four **intrinsic** types implemented in the compiler itself (they are declared in `lib.es5.d.ts` with the `intrinsic` keyword — you cannot write them yourself in TypeScript):

```ts
// type Uppercase<S extends string>    = intrinsic;
// type Lowercase<S extends string>    = intrinsic;
// type Capitalize<S extends string>   = intrinsic;
// type Uncapitalize<S extends string> = intrinsic;

type H1 = Uppercase<"content-type">;        // "CONTENT-TYPE"
type H2 = Lowercase<"X-Request-ID">;        // "x-request-id"
type H3 = Capitalize<"emailAddress">;       // "EmailAddress"
type H4 = Uncapitalize<"UserId">;           // "userId"

// They distribute over unions like any other operation on a union:
type Methods = Uppercase<"get" | "post" | "delete">;   // "GET" | "POST" | "DELETE"

// They compose with template literals — this is where they earn their keep:
type EventHandlerName<K extends string> = `on${Capitalize<K>}`;
type H = EventHandlerName<"connect" | "disconnect" | "error">;
// "onConnect" | "onDisconnect" | "onError"

// And they are transparent to wide `string`:
type Wide = Uppercase<string>;              // string  ← not an error, just no information

// Deferred over a type parameter — evaluated when the parameter is known:
type ScreamingKey<T extends string> = Uppercase<`app_${T}`>;
type SK = ScreamingKey<"database_url">;     // "APP_DATABASE_URL"
```

A crucial detail: these are **ASCII-ish, locale-independent** operations that mirror JavaScript's `toUpperCase()`/`toLowerCase()` for the common cases, but they are computed by the compiler on the type-level string. They do not run your runtime locale. Do not rely on them for Turkish `i`/`İ` or similar edge cases.

### Concept 4 — `infer` inside a template literal: pattern matching on strings

Conditional types (44) let you use `infer` to capture a piece of a type. Inside a template literal pattern, `infer` captures a **substring**:

```ts
// Split "entity.action" into its two halves:
type EventEntity<E extends string> = E extends `${infer Entity}.${infer _Action}` ? Entity : never;
type EventAction<E extends string> = E extends `${infer _Entity}.${infer Action}` ? Action : never;

type Ent = EventEntity<"payment.failed">;    // "payment"
type Act = EventAction<"payment.failed">;    // "failed"

// Strip a known prefix:
type StripApiPrefix<P extends string> = P extends `/api/${infer Rest}` ? Rest : P;
type S1 = StripApiPrefix<"/api/v1/users">;   // "v1/users"
type S2 = StripApiPrefix<"/health">;         // "/health" — no match, returns input

// Strip a known suffix:
type WithoutId<K extends string> = K extends `${infer Base}Id` ? Base : K;
type W1 = WithoutId<"userId">;               // "user"
type W2 = WithoutId<"email">;                // "email"

// ⚠️ Inference is GREEDY-LEFT: the first `infer` matches as LITTLE as possible.
type FirstSegment<P extends string> = P extends `${infer Head}/${infer Tail}` ? Head : P;
type F1 = FirstSegment<"api/v1/users">;      // "api"   ← Head took the shortest match
// Tail would be "v1/users" — the remainder, including further slashes.

// To match from the RIGHT you must recurse, consuming from the left until nothing is left:
type LastSegment<P extends string> =
  P extends `${string}/${infer Tail}` ? LastSegment<Tail> : P;
type L1 = LastSegment<"api/v1/users/:userId">;   // ":userId"

// `infer` can be constrained (TS 4.8+), which filters matches:
type ExtractStatusCode<S extends string> =
  S extends `HTTP ${infer Code extends number}` ? Code : never;
type C1 = ExtractStatusCode<"HTTP 404">;     // 404  ← a NUMBER literal, not "404"
type C2 = ExtractStatusCode<"HTTP OK">;      // never — "OK" is not a number
```

That last form — `infer Code extends number` — is how you convert a numeric *string* type into a numeric *literal* type. Before TS 4.8 this required ugly tuple-length tricks.

### Concept 5 — Recursive parsing: splitting, trimming, joining

Combine `infer` with recursion (46) and you can write real parsers at the type level.

```ts
// ── Split a delimited string into a tuple of segments ────────────────────────
type Split<S extends string, Delimiter extends string> =
  S extends `${infer Head}${Delimiter}${infer Tail}`
    ? [Head, ...Split<Tail, Delimiter>]
    : [S];

type Segments = Split<"api/v1/users/42", "/">;     // ["api", "v1", "users", "42"]
type CsvCols  = Split<"id,email,created_at", ",">; // ["id", "email", "created_at"]

// ── Join a tuple of strings back with a delimiter ────────────────────────────
type Join<T extends readonly string[], Delimiter extends string> =
  T extends readonly []                              ? ""
  : T extends readonly [infer Only extends string]   ? Only
  : T extends readonly [infer Head extends string, ...infer Rest extends string[]]
      ? `${Head}${Delimiter}${Join<Rest, Delimiter>}`
      : string;

type Path = Join<["api", "v1", "users"], "/">;     // "api/v1/users"

// ── Trim leading/trailing whitespace at the type level ──────────────────────
type TrimLeft<S extends string>  = S extends ` ${infer R}` ? TrimLeft<R> : S;
type TrimRight<S extends string> = S extends `${infer R} ` ? TrimRight<R> : S;
type Trim<S extends string>      = TrimLeft<TrimRight<S>>;

type T1 = Trim<"   application/json   ">;          // "application/json"

// ── Replace all occurrences ─────────────────────────────────────────────────
type ReplaceAll<S extends string, From extends string, To extends string> =
  From extends ""
    ? S
    : S extends `${infer Head}${From}${infer Tail}`
      ? `${Head}${To}${ReplaceAll<Tail, From, To>}`
      : S;

type R1 = ReplaceAll<"user_created_at", "_", "-">; // "user-created-at"

// ── snake_case → camelCase, the classic API-boundary transform ──────────────
type SnakeToCamel<S extends string> =
  S extends `${infer Head}_${infer Tail}`
    ? `${Head}${Capitalize<SnakeToCamel<Tail>>}`
    : S;

type SC1 = SnakeToCamel<"created_at">;             // "createdAt"
type SC2 = SnakeToCamel<"total_price_in_cents">;   // "totalPriceInCents"

// ── camelCase → snake_case, going the other way ─────────────────────────────
type CamelToSnake<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Head extends Uppercase<Head>
      ? Head extends Lowercase<Head>                // digits/symbols are both cases
        ? `${Head}${CamelToSnake<Tail>}`
        : `_${Lowercase<Head>}${CamelToSnake<Tail>}`
      : `${Head}${CamelToSnake<Tail>}`
    : S;

type CS1 = CamelToSnake<"createdAt">;              // "created_at"
type CS2 = CamelToSnake<"totalPriceInCents">;      // "total_price_in_cents"
```

The `Head extends Uppercase<Head>` trick is how you ask "is this character an uppercase letter?" — and the nested `Head extends Lowercase<Head>` filters out characters that have no case at all (digits, `_`, `-`), which are equal to both their uppercase and lowercase forms.

### Concept 6 — Template literals as key remappers in mapped types

Mapped types (43) support `as` clauses, and a template literal type in the `as` clause **renames** every key. This is the most common production use.

```ts
interface UserSchema {
  userId:    number;
  email:     string;
  isActive:  boolean;
  createdAt: Date;
}

// ── Getters ─────────────────────────────────────────────────────────────────
type Getters<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
};
type UserGetters = Getters<UserSchema>;
// {
//   getUserId:    () => number;
//   getEmail:     () => string;
//   getIsActive:  () => boolean;
//   getCreatedAt: () => Date;
// }

// ── Setters ─────────────────────────────────────────────────────────────────
type Setters<T> = {
  [K in keyof T & string as `set${Capitalize<K>}`]: (value: T[K]) => void;
};

// ── Change-event handlers ───────────────────────────────────────────────────
type ChangeHandlers<T> = {
  [K in keyof T & string as `on${Capitalize<K>}Change`]: (
    next: T[K],
    prev: T[K],
  ) => void;
};
type UserChangeHandlers = ChangeHandlers<UserSchema>;
// { onUserIdChange: (next: number, prev: number) => void; onEmailChange: ...; ... }

// ── Filtering keys by REMAPPING TO never ────────────────────────────────────
// A key mapped to `never` is dropped from the result entirely:
type StringKeysOnly<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
type OnlyStrings = StringKeysOnly<UserSchema>;      // { email: string }

// ── Namespacing every key ───────────────────────────────────────────────────
type Namespaced<T, Prefix extends string> = {
  [K in keyof T & string as `${Prefix}.${K}`]: T[K];
};
type NsUser = Namespaced<UserSchema, "user">;
// { "user.userId": number; "user.email": string; "user.isActive": boolean; "user.createdAt": Date }

// ── Renaming DB snake_case rows to camelCase domain objects ─────────────────
interface UserRow {
  user_id:    number;
  email:      string;
  is_active:  boolean;
  created_at: Date;
}
type CamelCaseKeys<T> = {
  [K in keyof T & string as SnakeToCamel<K>]: T[K];
};
type UserDomain = CamelCaseKeys<UserRow>;
// { userId: number; email: string; isActive: boolean; createdAt: Date }
```

### Concept 7 — Constraining function parameters to a pattern

You can use a template literal type as a **generic constraint** so a function only accepts strings of the right shape — while still capturing the exact literal for further inference.

```ts
// Only /api/... paths allowed:
declare function registerRoute<P extends `/api/${string}`>(
  path: P,
  handler: () => void,
): void;

registerRoute("/api/users", () => {});      // ✅
// registerRoute("/users", () => {});       // ❌ Argument of type '"/users"' is not
//                                          //   assignable to parameter of type `/api/${string}`

// Capture the literal AND derive information from it in one signature:
type TableName<Q extends string> =
  Q extends `SELECT ${string} FROM ${infer Table} ${string}` ? Table
  : Q extends `SELECT ${string} FROM ${infer Table}`         ? Table
  : never;

declare function query<Q extends string>(sql: Q): Promise<TableName<Q>[]>;

// The return type knows which table you queried:
const rows = query("SELECT id, email FROM users WHERE id = 1");
// Promise<"users"[]>  — a toy example, but the mechanism is real

// Autocomplete-preserving open unions — the `& {}` trick:
type KnownHeader = "content-type" | "authorization" | "x-request-id";
type AnyHeader   = KnownHeader | (string & {});
//                                ^^^^^^^^^^^^^ keeps the literal union visible to autocomplete
//                                              while still accepting arbitrary strings
declare function setHeader(name: AnyHeader, value: string): void;
setHeader("content-type", "application/json");   // ✅ autocompleted
setHeader("x-custom-tenant", "acme");            // ✅ still allowed
```

Without `(string & {})`, writing `KnownHeader | string` collapses immediately to `string` and you lose every suggestion. `string & {}` is a distinct-but-assignable type that survives union reduction — an ugly but universal idiom.

---

## Example 1 — basic

```ts
// A typed event bus where event names, and their payloads, are derived from one schema.

// ── The single source of truth ───────────────────────────────────────────────
interface EventPayloadMap {
  "user.created":     { userId: number; email: string };
  "user.deleted":     { userId: number; deletedBy: number };
  "order.placed":     { orderId: string; userId: number; totalCents: number };
  "order.cancelled":  { orderId: string; reason: string };
  "payment.failed":   { paymentId: string; orderId: string; errorCode: string };
}

type EventName = keyof EventPayloadMap & string;
// "user.created" | "user.deleted" | "order.placed" | "order.cancelled" | "payment.failed"

// ── Derive the entity and action from each name, purely at the type level ────
type EntityOf<E extends string> = E extends `${infer Entity}.${string}` ? Entity : never;
type ActionOf<E extends string> = E extends `${string}.${infer Action}` ? Action : never;

type Entities = EntityOf<EventName>;   // "user" | "order" | "payment"
type Actions  = ActionOf<EventName>;   // "created" | "deleted" | "placed" | "cancelled" | "failed"

// ── Derive handler method names: "user.created" → "onUserCreated" ───────────
type HandlerName<E extends string> =
  E extends `${infer Entity}.${infer Action}`
    ? `on${Capitalize<Entity>}${Capitalize<Action>}`
    : never;

type HandlerNames = HandlerName<EventName>;
// "onUserCreated" | "onUserDeleted" | "onOrderPlaced" | "onOrderCancelled" | "onPaymentFailed"

// ── A listener interface generated from the payload map ─────────────────────
type EventListeners = {
  [E in EventName as HandlerName<E>]?: (payload: EventPayloadMap[E]) => void | Promise<void>;
};
// {
//   onUserCreated?:    (payload: { userId: number; email: string }) => void | Promise<void>;
//   onUserDeleted?:    (payload: { userId: number; deletedBy: number }) => ...;
//   onOrderPlaced?:    (payload: { orderId: string; userId: number; totalCents: number }) => ...;
//   ...
// }

// ── The bus itself ──────────────────────────────────────────────────────────
class TypedEventBus {
  private handlers = new Map<EventName, Array<(payload: never) => void>>();

  on<E extends EventName>(eventName: E, handler: (payload: EventPayloadMap[E]) => void): void {
    const existing = this.handlers.get(eventName) ?? [];
    existing.push(handler as (payload: never) => void);
    this.handlers.set(eventName, existing);
  }

  emit<E extends EventName>(eventName: E, payload: EventPayloadMap[E]): void {
    for (const handler of this.handlers.get(eventName) ?? []) {
      (handler as (p: EventPayloadMap[E]) => void)(payload);
    }
  }

  // Subscribe to every event of one entity — "user.*" style, but typed:
  onEntity<Ent extends Entities>(
    entity: Ent,
    handler: (eventName: Extract<EventName, `${Ent}.${string}`>) => void,
  ): void {
    for (const name of this.handlers.keys()) {
      if (name.startsWith(`${entity}.`)) {
        handler(name as Extract<EventName, `${Ent}.${string}`>);
      }
    }
  }
}

// ── Usage ────────────────────────────────────────────────────────────────────
const bus = new TypedEventBus();

bus.on("user.created", (payload) => {
  // payload: { userId: number; email: string }  ← inferred from the event NAME
  console.log(`Welcome email queued for ${payload.email} (user ${payload.userId})`);
});

bus.on("payment.failed", (payload) => {
  // payload: { paymentId: string; orderId: string; errorCode: string }
  console.error(`Payment ${payload.paymentId} failed with ${payload.errorCode}`);
});

bus.emit("order.placed", { orderId: "ord_1", userId: 42, totalCents: 4999 });  // ✅

// ❌ bus.emit("order.placd", { orderId: "ord_1" });
//    Argument of type '"order.placd"' is not assignable to parameter of type EventName.

// ❌ bus.emit("order.placed", { orderId: "ord_1" });
//    Property 'userId' is missing — payload checked against the name.

// The listener object is fully typed and exhaustively named:
const listeners: EventListeners = {
  onUserCreated:   (payload) => console.log(payload.email),
  onOrderPlaced:   (payload) => console.log(payload.totalCents),
  onPaymentFailed: (payload) => console.log(payload.errorCode),
};
```

---

## Example 2 — real world backend use case

```ts
// A typed HTTP router + query builder + config loader, all driven by
// template literal types. This is close to what production frameworks do internally.

// ═══════════════════════════════════════════════════════════════════════════
// PART 1 — Route paths with typed params
// ═══════════════════════════════════════════════════════════════════════════

// Extract every ":param" segment out of a path literal, recursively.
type ExtractRouteParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<`/${Rest}`>
    : Path extends `${string}:${infer Param}`
      ? Param
      : never;

// Optional params written as ":param?" should be optional in the params object.
type RequiredParamKeys<Path extends string> =
  Exclude<ExtractRouteParams<Path>, `${string}?`>;

type OptionalParamKeys<Path extends string> =
  ExtractRouteParams<Path> extends infer P
    ? P extends `${infer Name}?` ? Name : never
    : never;

type RouteParams<Path extends string> =
  { [K in RequiredParamKeys<Path>]: string } &
  { [K in OptionalParamKeys<Path>]?: string };

type P1 = RouteParams<"/api/v1/users/:userId/orders/:orderId">;
// { userId: string; orderId: string }

type P2 = RouteParams<"/api/v1/reports/:reportId/:format?">;
// { reportId: string } & { format?: string }

// Query strings, if you want them typed too:
type ParseQuery<Q extends string> =
  Q extends `${infer Pair}&${infer Rest}`
    ? ParseQuery<Pair> & ParseQuery<Rest>
    : Q extends `${infer Key}=${string}`
      ? { [K in Key]: string }
      : {};

type Q1 = ParseQuery<"page=1&pageSize=20&sort=createdAt">;
// { page: string } & { pageSize: string } & { sort: string }

// ── The router ──────────────────────────────────────────────────────────────
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type ApiPath    = `/api/v${number}/${string}`;

interface TypedRequest<Path extends string, Body = unknown> {
  params:  RouteParams<Path>;
  body:    Body;
  headers: Record<string, string | undefined>;
  userId:  number | null;
}

interface TypedResponse {
  status(code: number): TypedResponse;
  json(payload: unknown): void;
}

type RouteKey<M extends HttpMethod, P extends string> = `${M} ${P}`;

class TypedRouter {
  private routes = new Map<string, (req: TypedRequest<string>, res: TypedResponse) => unknown>();

  register<M extends HttpMethod, P extends ApiPath, Body = unknown>(
    method: M,
    path: P,
    handler: (req: TypedRequest<P, Body>, res: TypedResponse) => Promise<void> | void,
  ): RouteKey<M, P> {
    const key = `${method} ${path}` as RouteKey<M, P>;
    this.routes.set(key, handler as (req: TypedRequest<string>, res: TypedResponse) => unknown);
    return key;
  }

  get<P extends ApiPath>(path: P, handler: (req: TypedRequest<P>, res: TypedResponse) => void) {
    return this.register("GET", path, handler);
  }

  post<P extends ApiPath, Body>(
    path: P,
    handler: (req: TypedRequest<P, Body>, res: TypedResponse) => void,
  ) {
    return this.register<"POST", P, Body>("POST", path, handler);
  }
}

const router = new TypedRouter();

router.get("/api/v1/users/:userId/orders/:orderId", (req, res) => {
  const userId  = req.params.userId;    // ✅ string — extracted from the path literal
  const orderId = req.params.orderId;   // ✅ string
  // const bad  = req.params.ordrId;    // ❌ Property 'ordrId' does not exist
  res.status(200).json({ userId, orderId });
});

// ❌ router.get("/v1/users", handler);
//    Argument of type '"/v1/users"' is not assignable to `/api/v${number}/${string}`

// ═══════════════════════════════════════════════════════════════════════════
// PART 2 — Typed column paths for a query builder
// ═══════════════════════════════════════════════════════════════════════════

interface DatabaseSchema {
  users: {
    id:         number;
    email:      string;
    full_name:  string;
    is_active:  boolean;
    created_at: Date;
  };
  orders: {
    id:          string;
    user_id:     number;
    total_cents: number;
    status:      "pending" | "paid" | "shipped" | "refunded";
    created_at:  Date;
  };
  payments: {
    id:         string;
    order_id:   string;
    provider:   "stripe" | "adyen";
    amount:     number;
    created_at: Date;
  };
}

type TableName = keyof DatabaseSchema & string;

// "users.id" | "users.email" | ... | "payments.created_at"
type ColumnPath = {
  [T in TableName]: `${T}.${keyof DatabaseSchema[T] & string}`;
}[TableName];

// Given a column path, recover its TypeScript type:
type ColumnType<Path extends ColumnPath> =
  Path extends `${infer T extends TableName}.${infer C}`
    ? C extends keyof DatabaseSchema[T]
      ? DatabaseSchema[T][C]
      : never
    : never;

type T1 = ColumnType<"orders.total_cents">;   // number
type T2 = ColumnType<"orders.status">;        // "pending" | "paid" | "shipped" | "refunded"
type T3 = ColumnType<"users.created_at">;     // Date

// Aliased columns: "users.email AS userEmail"
type AliasedColumn = `${ColumnPath} AS ${string}`;

// Sort directives: "orders.created_at DESC"
type SortDirective = `${ColumnPath} ${"ASC" | "DESC"}`;

// Join conditions between two real columns:
type JoinCondition = `${ColumnPath} = ${ColumnPath}`;

interface QuerySpec<T extends TableName> {
  from:   T;
  select: ColumnPath[];
  where?: Partial<{ [C in keyof DatabaseSchema[T] & string as `${T}.${C}`]: DatabaseSchema[T][C] }>;
  join?:  Array<{ table: TableName; on: JoinCondition }>;
  order?: SortDirective[];
  limit?: number;
}

declare function runQuery<T extends TableName>(spec: QuerySpec<T>): Promise<unknown[]>;

runQuery({
  from:   "orders",
  select: ["orders.id", "orders.total_cents", "users.email"],   // ✅ all valid paths
  where:  { "orders.status": "paid" },                          // ✅ value type checked
  join:   [{ table: "users", on: "orders.user_id = users.id" }],// ✅ both sides validated
  order:  ["orders.created_at DESC"],                           // ✅ direction constrained
  limit:  50,
});

// ❌ select: ["orders.totl_cents"]         — not a ColumnPath
// ❌ where:  { "orders.status": "paidd" }  — not in the status union
// ❌ order:  ["orders.created_at DES"]     — not "ASC" | "DESC"

// ═══════════════════════════════════════════════════════════════════════════
// PART 3 — Prefixed, validated environment configuration
// ═══════════════════════════════════════════════════════════════════════════

// Declare the config shape once, in camelCase, the way the app wants it:
interface AppConfig {
  databaseUrl:      string;
  redisHost:        string;
  redisPort:        number;
  jwtSecret:        string;
  jwtExpirySeconds: number;
  enableMetrics:    boolean;
  logLevel:         "debug" | "info" | "warn" | "error";
}

// camelCase → SCREAMING_SNAKE_CASE, at the type level:
type CamelToSnakeCase<S extends string> =
  S extends `${infer Head}${infer Tail}`
    ? Head extends Uppercase<Head>
      ? Head extends Lowercase<Head>
        ? `${Head}${CamelToSnakeCase<Tail>}`
        : `_${Lowercase<Head>}${CamelToSnakeCase<Tail>}`
      : `${Head}${CamelToSnakeCase<Tail>}`
    : S;

type EnvVarName<K extends string, Prefix extends string> =
  `${Prefix}_${Uppercase<CamelToSnakeCase<K>>}`;

type AppEnvVars = {
  [K in keyof AppConfig & string as EnvVarName<K, "APP">]: string | undefined;
};
// {
//   APP_DATABASE_URL?:       string | undefined;
//   APP_REDIS_HOST?:         string | undefined;
//   APP_REDIS_PORT?:         string | undefined;
//   APP_JWT_SECRET?:         string | undefined;
//   APP_JWT_EXPIRY_SECONDS?: string | undefined;
//   APP_ENABLE_METRICS?:     string | undefined;
//   APP_LOG_LEVEL?:          string | undefined;
// }

type EnvVarNameFor<K extends keyof AppConfig & string> = EnvVarName<K, "APP">;

// A parser table keyed by the config key, so each value is coerced correctly:
type Parsers = { [K in keyof AppConfig]: (raw: string) => AppConfig[K] };

const parsers: Parsers = {
  databaseUrl:      (raw) => raw,
  redisHost:        (raw) => raw,
  redisPort:        (raw) => Number.parseInt(raw, 10),
  jwtSecret:        (raw) => raw,
  jwtExpirySeconds: (raw) => Number.parseInt(raw, 10),
  enableMetrics:    (raw) => raw === "true" || raw === "1",
  logLevel:         (raw) => {
    if (raw === "debug" || raw === "info" || raw === "warn" || raw === "error") return raw;
    throw new Error(`Invalid APP_LOG_LEVEL: ${raw}`);
  },
};

function toEnvVarName<K extends keyof AppConfig & string>(key: K): EnvVarNameFor<K> {
  const snake = key.replace(/[A-Z]/g, (ch) => `_${ch.toLowerCase()}`);
  return `APP_${snake.toUpperCase()}` as EnvVarNameFor<K>;
}

function loadConfig(source: Record<string, string | undefined> = process.env): AppConfig {
  const keys = Object.keys(parsers) as Array<keyof AppConfig & string>;
  const config = {} as AppConfig;

  for (const key of keys) {
    const envVarName = toEnvVarName(key);        // e.g. "APP_JWT_EXPIRY_SECONDS"
    const raw = source[envVarName];
    if (raw === undefined) {
      throw new Error(`Missing required environment variable: ${envVarName}`);
    }
    // The cast is needed because TS cannot correlate `key` across the two mapped types:
    (config as Record<string, unknown>)[key] = parsers[key](raw);
  }

  return config;
}

const appConfig = loadConfig();
appConfig.redisPort;         // number
appConfig.logLevel;          // "debug" | "info" | "warn" | "error"
// appConfig.redisPrt;       // ❌ compile error

// ═══════════════════════════════════════════════════════════════════════════
// PART 4 — Permission strings and cache keys
// ═══════════════════════════════════════════════════════════════════════════

type PermissionResource = "users" | "orders" | "payments" | "reports";
type PermissionAction   = "read" | "write" | "delete" | "admin";
type Permission         = `${PermissionResource}:${PermissionAction}`;   // 16 members

type Scope = Permission | `${PermissionResource}:*` | "*:*";

function hasPermission(granted: readonly Scope[], required: Permission): boolean {
  const [resource] = required.split(":") as [PermissionResource, PermissionAction];
  return granted.includes(required)
      || granted.includes(`${resource}:*`)
      || granted.includes("*:*");
}

const authTokenScopes: Scope[] = ["orders:read", "users:*"];
hasPermission(authTokenScopes, "orders:read");     // true
hasPermission(authTokenScopes, "users:delete");    // true — matched by "users:*"
hasPermission(authTokenScopes, "payments:read");   // false
// hasPermission(authTokenScopes, "orders:reed");  // ❌ compile error

// Cache keys with a documented, enforced shape:
type CacheKey =
  | `user:${number}:profile`
  | `user:${number}:permissions`
  | `order:${string}:items`
  | `session:${string}`;

function buildUserProfileKey(userId: number): CacheKey {
  return `user:${userId}:profile`;
}

declare function cacheGet<T>(key: CacheKey): Promise<T | null>;
declare function cacheSet(key: CacheKey, value: unknown, ttlSeconds: number): Promise<void>;

await cacheSet(buildUserProfileKey(42), { email: "a@b.com" }, 300);   // ✅
// await cacheSet("usr:42:profile", {}, 300);                         // ❌ typo caught
```

---

## Going deeper

### Union size limits and compile-time cost

Template literal types produce unions, and unions have a hard ceiling. The compiler errors with **"Expression produces a union type that is too complex to represent"** at roughly **100,000** members, and gets slow long before that.

```ts
type Char = "a"|"b"|"c"|"d"|"e"|"f"|"g"|"h"|"i"|"j";   // 10
type Two   = `${Char}${Char}`;                          // 100
type Three = `${Char}${Char}${Char}`;                   // 1,000
type Four  = `${Char}${Char}${Char}${Char}`;            // 10,000 — already noticeably slow
// type Five = `${Char}${Char}${Char}${Char}${Char}`;    // 100,000 — ❌ too complex

// The practical rule: keep the product under a few thousand.
// 5 methods × 4 resources = 20 ✅
// 5 methods × 4 resources × 3 versions × 4 envs = 240 ✅
// 26 letters × 26 × 26 = 17,576 ⚠️ compiles, but your editor will lag
```

Crucially, a **wide** slot does not enumerate — `` `/api/${string}` `` is a single *pattern* type, not a union, and costs nothing. Prefer widening a slot to `string` when you don't actually need exhaustive enumeration.

### Pattern types vs finite unions: assignability differs

```ts
type Finite  = `/api/${"users" | "orders"}`;     // 2-member union
type Pattern = `/api/${string}`;                 // a pattern

declare let f: Finite;
declare let p: Pattern;

p = f;    // ✅ every Finite member matches the Pattern
// f = p; // ❌ Pattern is wider — not assignable to the 2-member union

// Pattern types narrow at runtime with startsWith (TS 4.9+ understands this
// only through explicit predicates — plain startsWith does NOT narrow):
function isApiPath(value: string): value is `/api/${string}` {
  return value.startsWith("/api/");
}

declare const rawPath: string;
if (isApiPath(rawPath)) {
  rawPath;   // `/api/${string}`
}
```

### `infer ... extends` for numeric conversion

```ts
// Convert a string type to a number type (TS 4.8+):
type ToNumber<S extends string> = S extends `${infer N extends number}` ? N : never;
type N1 = ToNumber<"42">;      // 42  (a number literal type)
type N2 = ToNumber<"abc">;     // never

// Useful for typed pagination or status codes read from route/config strings:
type StatusOf<S extends string> = S extends `HTTP/${string} ${infer C extends number} ${string}` ? C : never;
type S1 = StatusOf<"HTTP/1.1 404 Not Found">;   // 404

// ⚠️ Leading zeros and floats are lossy — "007" infers as 7, "1.50" as 1.5:
type N3 = ToNumber<"007">;     // 7
type N4 = ToNumber<"1.50">;    // 1.5
// If the exact original string matters, keep the string type around too.
```

### Recursion depth: the 1,000-level ceiling

Recursive template literal types (`Split`, `Trim`, `CamelToSnake`) recurse **once per character or segment**. TypeScript caps type instantiation depth at roughly 1,000 for non-tail-recursive types (and 1,000 iterations for tail-recursive conditional types since TS 4.5).

```ts
// This is fine — a route path has maybe 10 segments:
type Segs = Split<"/api/v1/users/:userId/orders/:orderId/items", "/">;

// This will explode — never run a character-by-character type over a long string:
// type Reversed = ReverseString<"...a 2000-character string...">;
//   Error: Type instantiation is excessively deep and possibly infinite.

// Mitigation: process per-SEGMENT (few iterations) rather than per-CHARACTER (many).
// CamelToSnake<S> is per-character — safe for identifiers (< 40 chars),
// dangerous for arbitrary user input types.
```

See **46 — Recursive types** for the tail-recursion rules that raise the ceiling.

### Distribution only happens for naked type parameters

```ts
type Wrap<T extends string> = `<${T}>`;
type W1 = Wrap<"a" | "b">;                 // "<a>" | "<b>"  — distributed

// But a union produced by a conditional in the slot behaves the same way,
// because the template literal itself is what distributes:
type W2 = `<${"a" | "b"}-${"x" | "y"}>`;   // "<a-x>" | "<a-y>" | "<b-x>" | "<b-y>"

// `never` in ANY slot annihilates the whole type — a very common silent bug:
type W3 = `prefix-${never}`;               // never
type W4 = `${string}-${never}-${string}`;  // never

// This bites when an `infer` fails and you feed the `never` forward:
type Entity<E extends string> = E extends `${infer Ent}.${string}` ? Ent : never;
type Bad = `handle_${Entity<"nodothere">}`;   // never — not "handle_" as you might hope
```

### Template literals do not validate at runtime

The type `` `user:${number}` `` is erased at compile time. Nothing stops a JSON body or a database read from producing `"user:abc"` and being cast into it.

```ts
// ❌ A lie the compiler cannot catch:
const key = JSON.parse(requestBody).key as `user:${number}`;

// ✅ Validate, then narrow with a predicate:
function isUserCacheKey(value: string): value is `user:${number}` {
  return /^user:\d+$/.test(value);
}

const raw: string = JSON.parse(requestBody).key;
if (!isUserCacheKey(raw)) throw new Error(`Malformed cache key: ${raw}`);
raw;   // `user:${number}`
```

Template literal types are a **compile-time contract**, exactly like every other type. At the I/O boundary you still need a runtime validator (Zod, io-ts, hand-written predicates).

### Editor performance and the `--generateTrace` escape hatch

Heavy template literal machinery is one of the top causes of a sluggish editor in large TypeScript codebases. If hovering a type takes seconds:

```bash
# Emit a compiler trace and look for the expensive type instantiations:
tsc --generateTrace ./trace-out --noEmit
# Then open trace-out/trace.json in Chrome's chrome://tracing or Perfetto.
```

Common fixes, in order of effectiveness: widen a slot to `string`; replace a recursive per-character type with a pre-computed union; move an expensive type behind a named type alias so it is cached; or accept `string` and validate at runtime instead.

### `Uppercase<T>` is not injective — beware round-trips

```ts
type Round = Lowercase<Uppercase<"UserId">>;   // "userid"  ← the camel hump is gone forever
// Uppercase/Lowercase lose information. Capitalize/Uncapitalize only touch
// the FIRST character, so they are safer for identifier transforms:
type Safe = Uncapitalize<Capitalize<"userId">>;  // "userId" — round-trips cleanly
```

---

## Common mistakes

### Mistake 1 — Assuming a union in a slot is "one of", not a cross product

```ts
// ❌ Expecting 3 members, getting 9:
type Entity = "user" | "order" | "payment";
type Action = "created" | "updated" | "deleted";

// The author wanted user.created, order.updated, payment.deleted (a zip).
// They got every combination (a cross product):
type WrongEvent = `${Entity}.${Action}`;   // 9 members, not 3

// ✅ If you want specific pairings, enumerate them or drive them from a map:
interface EventMap {
  user:    "created" | "updated" | "deleted";
  order:   "placed" | "cancelled";
  payment: "succeeded" | "failed";
}
type RightEvent = { [E in keyof EventMap & string]: `${E}.${EventMap[E]}` }[keyof EventMap & string];
// "user.created" | "user.updated" | "user.deleted"
// | "order.placed" | "order.cancelled"
// | "payment.succeeded" | "payment.failed"     ← 7 members, exactly the real ones
```

### Mistake 2 — Widening a literal by storing it in a `let` or an object property

```ts
declare function registerRoute(path: `/api/${string}`): void;

// ❌ `let` widens the literal to `string`:
let path = "/api/users";          // inferred as string, not "/api/users"
// registerRoute(path);           // ❌ Argument of type 'string' is not assignable
//                                //    to parameter of type `/api/${string}`

// ❌ Object properties widen too:
const routeConfig = { path: "/api/users" };   // { path: string }
// registerRoute(routeConfig.path);           // ❌ same error

// ✅ Use const, `as const`, or an explicit annotation:
const path1 = "/api/users";                             // "/api/users" (const narrows)
registerRoute(path1);                                   // ✅

const routeConfig2 = { path: "/api/users" } as const;   // { readonly path: "/api/users" }
registerRoute(routeConfig2.path);                       // ✅

const path3: `/api/${string}` = "/api/users";           // explicit annotation
registerRoute(path3);                                   // ✅

// ✅ Or make the function generic so it captures the literal:
declare function registerRoute2<P extends `/api/${string}`>(path: P): void;
```

### Mistake 3 — Forgetting `& string` when mapping over `keyof T`

```ts
interface UserSchema { userId: number; email: string }

// ❌ `keyof T` is `string | number | symbol`; symbols cannot go in a template literal:
type BadGetters<T> = {
  [K in keyof T as `get${Capitalize<K>}`]: () => T[K];
  //                          ^^^^^^^^^^^ Error: Type 'K' does not satisfy the
  //                                      constraint 'string'.
};

// ✅ Intersect with string to filter out number/symbol keys:
type GoodGetters<T> = {
  [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K];
};
type UG = GoodGetters<UserSchema>;   // { getUserId: () => number; getEmail: () => string }

// Note: `Extract<keyof T, string>` is equivalent and some codebases prefer it:
type AltGetters<T> = {
  [K in Extract<keyof T, string> as `get${Capitalize<K>}`]: () => T[K];
};
```

### Mistake 4 — Letting a failed `infer` produce `never` and swallow the whole type

```ts
type ActionOf<E extends string> = E extends `${string}.${infer A}` ? A : never;

// ❌ If any member of the input union has no dot, that member silently vanishes:
type Events = "user.created" | "healthcheck";
type Actions = ActionOf<Events>;               // "created"  ← "healthcheck" became never
type Handlers = `on${Capitalize<ActionOf<Events>>}`;  // "onCreated" only — silently incomplete

// ✅ Make the fallback explicit so bad input is visible:
type ActionOfStrict<E extends string> =
  E extends `${string}.${infer A}` ? A : ["ERROR: not a dotted event name", E];

type Bad = ActionOfStrict<"healthcheck">;
// ["ERROR: not a dotted event name", "healthcheck"]  ← loud, and breaks downstream

// ✅ Or constrain the input so malformed members cannot get in at all:
type DottedEvent = `${string}.${string}`;
type ActionOfSafe<E extends DottedEvent> = E extends `${string}.${infer A}` ? A : never;
// ActionOfSafe<"healthcheck">  → ❌ compile error at the call site, where it belongs
```

### Mistake 5 — Building an enormous union when a pattern would do

```ts
// ❌ Enumerating every possible request id — 36^8 combinations, instantly fatal:
// type Char36 = "a"|"b"|...|"z"|"0"|...|"9";
// type RequestId = `req_${Char36}${Char36}${Char36}${Char36}${Char36}${Char36}${Char36}${Char36}`;
//   Error: Expression produces a union type that is too complex to represent.

// ✅ Use a pattern — zero cost, still prevents "usr_..." typos:
type RequestId = `req_${string}`;

// ✅ And validate the exact shape at runtime where it actually matters:
function isRequestId(value: string): value is RequestId {
  return /^req_[a-z0-9]{8}$/.test(value);
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a small type-level toolkit for a notification service.

Given:

```ts
type Channel  = "email" | "sms" | "push" | "webhook";
type Priority = "low" | "normal" | "high" | "urgent";
```

Write, from scratch:

1. `NotificationTopic` — every `channel.priority` combination (16 members).
2. `QueueName` — every topic prefixed and uppercased, e.g. `"QUEUE_EMAIL_URGENT"` (hint: `Uppercase` plus a replacement of `.` with `_`, or build it directly).
3. `HandlerMethodName<C extends Channel>` — turns `"email"` into `"sendEmail"`, `"webhook"` into `"sendWebhook"`.
4. `NotificationHandlers` — an object type with one method per channel, named by the rule above, each taking `{ recipientId: number; body: string }` and returning `Promise<void>`.
5. `ChannelOf<T extends NotificationTopic>` — extracts `"email"` out of `"email.urgent"`.
6. A function `enqueue(topic: NotificationTopic, payload: { recipientId: number; body: string }): void` where passing `"emial.high"` is a compile error.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a **typed config accessor** with dot-path key support.

Given a nested config object type:

```ts
interface ServiceConfig {
  server:   { port: number; host: string; timeoutMs: number };
  database: { url: string; poolSize: number; ssl: { enabled: boolean; caPath: string } };
  auth:     { jwtSecret: string; expirySeconds: number; issuer: string };
  features: { enableBetaApi: boolean; enableAuditLog: boolean };
}
```

Write, from scratch:

1. `ConfigPath<T>` — a union of every valid dot path, including nested ones:
   `"server" | "server.port" | "server.host" | ... | "database.ssl.enabled" | ...`
   (Include intermediate object paths as well as leaf paths.)
2. `ConfigValue<T, P extends ConfigPath<T>>` — resolves the type at that path.
   `ConfigValue<ServiceConfig, "database.ssl.enabled">` must be `boolean`.
3. `LeafPath<T>` — only paths whose value is a primitive (no intermediate object paths).
4. A class `ConfigStore` with:
   - `get<P extends ConfigPath<ServiceConfig>>(path: P): ConfigValue<ServiceConfig, P>`
   - `set<P extends LeafPath<ServiceConfig>>(path: P, value: ConfigValue<ServiceConfig, P>): void`
   - `has(path: string): path is ConfigPath<ServiceConfig>` — a runtime-validating predicate
5. Demonstrate that `store.get("database.ssl.enabled")` is `boolean`, that
   `store.get("database.ssl.enabld")` is a compile error, and that
   `store.set("server.port", "8080")` is a compile error.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **type-level route path parser and typed API client** — the kind of thing tRPC and Hono do internally.

Requirements:

1. A route definition object, declared once with `as const`:

```ts
const routes = {
  "GET /api/v1/users/:userId":                   { response: {} as UserDto },
  "GET /api/v1/users/:userId/orders":            { response: {} as OrderDto[] },
  "POST /api/v1/users/:userId/orders":           { body: {} as CreateOrderDto, response: {} as OrderDto },
  "PATCH /api/v1/orders/:orderId/status":        { body: {} as { status: OrderStatus }, response: {} as OrderDto },
  "DELETE /api/v1/orders/:orderId":              { response: {} as void },
  "GET /api/v1/reports/:reportId/:format?":      { response: {} as Blob },
} as const;
```

2. Type-level machinery, all written by you:
   - `RouteKey` — `keyof typeof routes`
   - `MethodOf<K extends RouteKey>` — `"GET" | "POST" | ...` extracted from the key
   - `PathOf<K extends RouteKey>` — the path portion, extracted from the key
   - `PathParams<P>` — required params from `:name`, optional params from `:name?`,
     producing `{ userId: string } & { format?: string }`
   - `RoutesForMethod<M>` — filter `RouteKey` down to one HTTP method
   - `BodyOf<K>` / `ResponseOf<K>` — pull the declared body/response types out
   - `BuildUrl<P, Params>` — a function that substitutes params into the path and
     returns a `string` typed as narrowly as you can manage

3. A `TypedApiClient` class with:
   - `request<K extends RouteKey>(key: K, options: RequestOptions<K>): Promise<ResponseOf<K>>`
     where `RequestOptions<K>` requires `params` only if the path has params, and
     requires `body` only if the route declares one (use conditional optionality, not
     `params?: ...` on every route)
   - Convenience methods `get`, `post`, `patch`, `del`, each constrained to routes of
     that method so `client.get("POST /api/v1/users/:userId/orders", ...)` is a compile error
   - An `authToken` field applied as an `Authorization: Bearer ${string}` header, where
     the header value type is a template literal

4. Prove the following are compile errors:
   - Calling `request("GET /api/v1/users/:userId", { params: { userID: "1" } })` (wrong case)
   - Omitting `params` for a route that has them
   - Passing `body` to a route that declares none
   - `client.get` on a `POST` route

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Construction ────────────────────────────────────────────────────────────
type A = `/api/${string}`;                    // open pattern
type B = `${"GET" | "POST"} /users`;          // union distributes
type C = `${"a"|"b"}-${"x"|"y"}`;             // cross product: 4 members
type D = `port_${number}`;                    // numeric pattern
type E = `flag_${boolean}`;                   // "flag_true" | "flag_false"

// ── Intrinsics ──────────────────────────────────────────────────────────────
Uppercase<"a-b">      // "A-B"
Lowercase<"A-B">      // "a-b"
Capitalize<"userId">  // "UserId"
Uncapitalize<"UserId">// "userId"

// ── Pattern matching with infer ─────────────────────────────────────────────
type Head<S extends string> = S extends `${infer H}.${string}` ? H : never;
type Tail<S extends string> = S extends `${string}.${infer T}` ? T : never;
type Num<S extends string>  = S extends `${infer N extends number}` ? N : never;

// ── Key remapping in mapped types ───────────────────────────────────────────
type Getters<T> = { [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K] };
type Prefix<T, P extends string> = { [K in keyof T & string as `${P}.${K}`]: T[K] };
type Drop<T>    = { [K in keyof T as T[K] extends Function ? never : K]: T[K] };

// ── Recursive helpers ───────────────────────────────────────────────────────
type Split<S extends string, D extends string> =
  S extends `${infer H}${D}${infer T}` ? [H, ...Split<T, D>] : [S];
type Trim<S extends string> =
  S extends ` ${infer R}` ? Trim<R> : S extends `${infer R} ` ? Trim<R> : S;
type SnakeToCamel<S extends string> =
  S extends `${infer H}_${infer T}` ? `${H}${Capitalize<SnakeToCamel<T>>}` : S;

// ── Constraints & guards ────────────────────────────────────────────────────
declare function f<P extends `/api/${string}`>(path: P): void;   // constrain + capture
function isApiPath(v: string): v is `/api/${string}` {           // runtime narrowing
  return v.startsWith("/api/");
}
type OpenUnion = "known" | (string & {});                        // keep autocomplete
```

| Feature | Syntax | Result |
|---|---|---|
| Literal interpolation | `` `/api/${"v1"}` `` | `"/api/v1"` |
| Union distribution | `` `${"a"\|"b"}_x` `` | `"a_x" \| "b_x"` |
| Cross product | `` `${A}-${B}` `` | every combination — grows multiplicatively |
| Open pattern | `` `/api/${string}` `` | matches any string with that prefix; costs nothing |
| Numeric pattern | `` `user:${number}` `` | `"user:1"`, `"user:42"`, … |
| Uppercase / Lowercase | `Uppercase<S>` | whole-string case change; lossy |
| Capitalize / Uncapitalize | `Capitalize<S>` | first character only; round-trips |
| Substring capture | `` S extends `${infer H}.${string}` `` | leftmost-shortest match |
| String → number literal | `` S extends `${infer N extends number}` `` | numeric literal type (TS 4.8+) |
| Key remap | `[K in keyof T & string as \`get${Capitalize<K>}\`]` | renames keys |
| Drop a key | remap to `never` | key removed from the result |
| `never` in a slot | `` `x-${never}` `` | the whole type becomes `never` |
| Union size ceiling | ~100,000 members | "union type that is too complex to represent" |
| Recursion ceiling | ~1,000 instantiations | "type instantiation is excessively deep" |

---

## Connected topics

- **43 — Mapped types** — the `as` key-remapping clause is where template literal types do most of their production work (`Getters<T>`, `Namespaced<T>`).
- **44 — Conditional types** — `infer` inside a template literal is a conditional type; every parser in this doc is built on `T extends U ? X : Y`.
- **46 — Recursive types** — `Split`, `Trim`, and `SnakeToCamel` recurse; that doc covers the depth limits and tail-recursion rules that keep them compiling.
- **42 — Discriminated unions** — event-name unions like `` `${Entity}.${Action}` `` pair naturally with a discriminated payload union keyed by the same tag.
