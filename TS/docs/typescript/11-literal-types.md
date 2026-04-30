# 11 — Literal Types

## What is this?

A literal type is a type that represents **one exact value** — not all strings, but specifically `"GET"`. Not all numbers, but specifically `200`. Not all booleans, but specifically `true`. Literal types let you constrain a variable or parameter to a specific set of exact values rather than a broad primitive type. Combined with union types, they become one of the most practically useful features in TypeScript for backend development.

## Why does it matter?

In JavaScript you rely on runtime checks (`if (method === "GET")`) to validate that values are correct. TypeScript literal types move that validation to compile time — if you declare a parameter type as `"GET" | "POST" | "PUT" | "DELETE"`, passing `"get"` (lowercase) or `"PATCH"` (unlisted) fails immediately in your editor. This eliminates an entire class of "invalid input" bugs in your API handlers, state machines, and configuration objects.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — method could be absolutely any string
function makeRequest(url, method) {
  if (method === "GTE") {   // typo — JS doesn't catch it
    return fetch(url, { method: "GET" });
  }
  return fetch(url, { method });
}

makeRequest("/api/users", "GTE");      // silently wrong — typo in "GET"
makeRequest("/api/users", "DELETED");  // silently wrong — should be "DELETE"
makeRequest("/api/users", "yolo");     // silently wrong — completely invalid
```

```ts
// TypeScript — method is constrained to valid values only
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

function makeRequest(url: string, method: HttpMethod): Promise<Response> {
  return fetch(url, { method });
}

makeRequest("/api/users", "GET");      // ✅
makeRequest("/api/users", "GTE");      // ❌ Argument of type '"GTE"' is not assignable to type 'HttpMethod'
makeRequest("/api/users", "DELETED");  // ❌
makeRequest("/api/users", "yolo");     // ❌
```

---

## Syntax

```ts
// ── STRING LITERAL TYPES ──────────────────────────────────────────────────
type Direction = "north" | "south" | "east" | "west";
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type Environment = "development" | "staging" | "production";
type UserRole = "admin" | "editor" | "viewer";

// ── NUMERIC LITERAL TYPES ─────────────────────────────────────────────────
type HttpStatus = 200 | 201 | 400 | 401 | 403 | 404 | 500;
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
type ApiVersion = 1 | 2 | 3;

// ── BOOLEAN LITERAL TYPES ─────────────────────────────────────────────────
type AlwaysTrue = true;
type AlwaysFalse = false;

// ── SINGLE LITERAL (not a union) ──────────────────────────────────────────
type OnlyPOST = "POST";           // only accepts "POST"

// ── LITERAL IN FUNCTION PARAMS ────────────────────────────────────────────
function setLogLevel(level: "debug" | "info" | "warn" | "error"): void { ... }

// ── LITERAL AS PROPERTY TYPE ──────────────────────────────────────────────
type SuccessResponse = {
  status: "success";               // this field can ONLY be "success"
  data: object;
};
```

---

## How it works — where literals come from

### `const` creates literal types automatically

```ts
// TypeScript infers a literal type for const declarations:
const method = "GET";        // inferred type: "GET"  (not string)
const port = 3000;           // inferred type: 3000   (not number)
const isEnabled = true;      // inferred type: true   (not boolean)

// let declarations infer the wider primitive type:
let method2 = "GET";         // inferred type: string (not "GET")

// This asymmetry matters for union assignment:
type HttpMethod = "GET" | "POST";

const m1 = "GET";            // type: "GET" — assignable to HttpMethod ✅
let m2 = "GET";              // type: string — NOT assignable to HttpMethod ❌

function request(method: HttpMethod): void { ... }
request(m1);   // ✅
request(m2);   // ❌ Argument of type 'string' is not assignable to type 'HttpMethod'
```

---

## `as const` — forcing literal types on any value

`as const` tells TypeScript: "treat every value in this structure as its most specific literal type, and make everything readonly."

```ts
// Without as const — properties infer their primitive types:
const config = {
  method: "GET",    // inferred: string
  port: 3000,       // inferred: number
  retries: 3,       // inferred: number
};
config.method = "POST";   // ✅ string is fine — mutable

// With as const — properties infer their literal types:
const config2 = {
  method: "GET",    // inferred: "GET"  (literal)
  port: 3000,       // inferred: 3000   (literal)
  retries: 3,       // inferred: 3      (literal)
} as const;
config2.method = "POST";  // ❌ Cannot assign to 'method' because it is a read-only property

// Full inferred type of config2:
// { readonly method: "GET"; readonly port: 3000; readonly retries: 3 }
```

### `as const` on arrays

```ts
// Without as const:
const methods = ["GET", "POST", "DELETE"];
// inferred: string[]  — elements can be anything

// With as const:
const methods = ["GET", "POST", "DELETE"] as const;
// inferred: readonly ["GET", "POST", "DELETE"]
// A tuple with specific literal types at each position

// This enables using the array values as a union type:
type AllowedMethod = typeof methods[number];
// type AllowedMethod = "GET" | "POST" | "DELETE"
```

---

## Template literal types

TypeScript lets you build new string literal types by combining other string literals with template syntax:

```ts
// Basic template literal type:
type EventName = `on${string}`;           // "onClick", "onChange", "onSubmit", etc.

// Combining with union types — produces every combination:
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ApiVersion = "v1" | "v2" | "v3";

type ApiEndpoint = `/${ApiVersion}/${string}`;
// Valid: "/v1/users", "/v2/products", "/v3/orders/123"

// Real use: building event names from a union:
type CrudEvent = "created" | "updated" | "deleted";
type Entity = "user" | "post" | "comment";

type DomainEvent = `${Entity}:${CrudEvent}`;
// "user:created" | "user:updated" | "user:deleted" |
// "post:created" | "post:updated" | "post:deleted" |
// "comment:created" | "comment:updated" | "comment:deleted"

function emitEvent(event: DomainEvent, payload: object): void {
  console.log(`Event emitted: ${event}`);
}

emitEvent("user:created", { userId: 1 });    // ✅
emitEvent("user:deleted", { userId: 1 });    // ✅
emitEvent("user:archived", { userId: 1 });   // ❌ "archived" is not in CrudEvent
emitEvent("admin:created", { userId: 1 });   // ❌ "admin" is not in Entity
```

### Template literals for route path typing

```ts
type UserId = `user-${number}`;       // "user-1", "user-42", "user-9999"
type ApiPath = `/api/${string}`;      // must start with /api/

function callEndpoint(path: ApiPath): void {
  fetch(path);
}

callEndpoint("/api/users");           // ✅
callEndpoint("/api/users/1");         // ✅
callEndpoint("/users");               // ❌ doesn't start with /api/
callEndpoint("api/users");            // ❌ doesn't start with /
```

---

## `const` assertions vs explicit literal annotations

```ts
// Option 1: explicit literal annotation
let role: "admin" = "admin";
role = "user";    // ❌ not "admin"

// Option 2: use const (most common)
const role = "admin";   // inferred as "admin" literal

// Option 3: as const on the value (useful for let)
let role2 = "admin" as const;  // inferred as "admin" literal
role2 = "user";                 // ❌

// Option 4: as const on an object (makes all properties readonly literals)
const HTTP_CODES = {
  OK: 200,
  CREATED: 201,
  NOT_FOUND: 404,
} as const;

// type: { readonly OK: 200; readonly CREATED: 201; readonly NOT_FOUND: 404 }
HTTP_CODES.OK = 999;  // ❌ readonly
```

---

## Using `typeof` to derive a union from an object's values

```ts
// Instead of maintaining both an object AND a union type separately:
// ❌ FRAGILE — two things to keep in sync
const ROLES_OBJECT = { ADMIN: "admin", EDITOR: "editor", VIEWER: "viewer" };
type UserRole = "admin" | "editor" | "viewer";  // must manually stay in sync

// ✅ BETTER — derive the type from the object:
const USER_ROLES = {
  ADMIN: "admin",
  EDITOR: "editor",
  VIEWER: "viewer",
} as const;

type UserRole = typeof USER_ROLES[keyof typeof USER_ROLES];
// typeof USER_ROLES = { readonly ADMIN: "admin"; readonly EDITOR: "editor"; readonly VIEWER: "viewer" }
// keyof typeof USER_ROLES = "ADMIN" | "EDITOR" | "VIEWER"
// typeof USER_ROLES[keyof typeof USER_ROLES] = "admin" | "editor" | "viewer"

function assignRole(userId: number, role: UserRole): void {
  console.log(`User ${userId} assigned role: ${role}`);
}

assignRole(1, USER_ROLES.ADMIN);   // ✅ "admin"
assignRole(1, "editor");           // ✅ literal string
assignRole(1, "superadmin");       // ❌ not in UserRole
```

---

## Example 1 — basic

```ts
// HTTP status codes as typed literals
type SuccessStatus = 200 | 201 | 204;
type ClientErrorStatus = 400 | 401 | 403 | 404 | 409 | 422;
type ServerErrorStatus = 500 | 502 | 503;
type HttpStatus = SuccessStatus | ClientErrorStatus | ServerErrorStatus;

// Typed log levels
type LogLevel = "debug" | "info" | "warn" | "error" | "fatal";

// A typed logger
function log(level: LogLevel, message: string): void {
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] [${level.toUpperCase()}] ${message}`);
}

log("info", "Server started");        // ✅
log("error", "Database unreachable"); // ✅
log("verbose", "Too much detail");    // ❌ "verbose" is not a LogLevel

// A typed response sender
function sendResponse(
  res: { status(code: number): { json(data: unknown): void } },
  statusCode: HttpStatus,
  body: unknown
): void {
  res.status(statusCode).json(body);
}

sendResponse(res, 200, { data: [] });   // ✅
sendResponse(res, 418, {});             // ❌ 418 is not in HttpStatus
```

---

## Example 2 — real world backend use case

```ts
// A fully typed event system using template literal types

// ── Domain entities ────────────────────────────────────────────────────────
const ENTITIES = ["user", "post", "comment", "order"] as const;
type Entity = typeof ENTITIES[number];   // "user" | "post" | "comment" | "order"

const CRUD_ACTIONS = ["created", "updated", "deleted", "restored"] as const;
type CrudAction = typeof CRUD_ACTIONS[number];

// ── Event name type (all combinations) ────────────────────────────────────
type DomainEvent = `${Entity}.${CrudAction}`;
// "user.created" | "user.updated" | "user.deleted" | "user.restored" |
// "post.created" | ... (16 total combinations)

// ── Typed event payload map ────────────────────────────────────────────────
type EventPayload = {
  "user.created": { userId: number; email: string; role: "admin" | "user" };
  "user.deleted": { userId: number; deletedBy: number };
  "order.created": { orderId: number; userId: number; total: number };
  "order.updated": { orderId: number; status: "processing" | "shipped" | "delivered" };
};

// Generic event emitter typed to the payload map:
type EventHandler<T extends DomainEvent> =
  T extends keyof EventPayload
    ? (payload: EventPayload[T]) => void
    : (payload: Record<string, unknown>) => void;

const eventHandlers = new Map<DomainEvent, Function>();

function on<T extends DomainEvent>(event: T, handler: EventHandler<T>): void {
  eventHandlers.set(event, handler);
}

function emit<T extends DomainEvent>(
  event: T,
  payload: T extends keyof EventPayload ? EventPayload[T] : Record<string, unknown>
): void {
  const handler = eventHandlers.get(event);
  if (handler) handler(payload);
}

// Usage — fully typed:
on("user.created", (payload) => {
  // TypeScript knows payload is { userId: number; email: string; role: "admin" | "user" }
  console.log(`New user: ${payload.email} with role ${payload.role}`);
});

emit("user.created", {
  userId: 1,
  email: "parsh@dev.io",
  role: "admin",
});  // ✅

emit("user.created", {
  userId: 1,
  email: "parsh@dev.io",
  role: "superadmin",   // ❌ "superadmin" is not "admin" | "user"
});

emit("user.archived", { userId: 1 });  // ❌ "user.archived" is not a DomainEvent
```

---

## Common mistakes

### Mistake 1 — Widening a literal type with `let`

```ts
// ❌ WRONG — let widens "GET" to string, breaking the function call
type HttpMethod = "GET" | "POST";

let method = "GET";              // inferred: string (widened)
function request(m: HttpMethod) { ... }
request(method);                 // ❌ Argument of type 'string' is not assignable to type 'HttpMethod'

// Three fixes:
// Fix 1 — use const
const method2 = "GET";           // inferred: "GET"
request(method2);                // ✅

// Fix 2 — annotate with the union type
let method3: HttpMethod = "GET";
request(method3);                // ✅

// Fix 3 — use as const on the value
let method4 = "GET" as const;    // inferred: "GET"
request(method4);                // ✅
```

### Mistake 2 — Building an enum-like object without `as const`

```ts
// ❌ WRONG — without as const, properties are widened to string
const HTTP_METHODS = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
};

type Method = typeof HTTP_METHODS[keyof typeof HTTP_METHODS];
// Inferred as: string  (not "GET" | "POST" | "PUT")
// because HTTP_METHODS.GET is string, not "GET"

// ✅ RIGHT — use as const
const HTTP_METHODS2 = {
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
} as const;

type Method2 = typeof HTTP_METHODS2[keyof typeof HTTP_METHODS2];
// Inferred as: "GET" | "POST" | "PUT"  ✅
```

### Mistake 3 — Template literal types matching too broadly

```ts
// ❌ TOO BROAD — `${string}` matches any string, defeating the purpose
type ApiPath = `/${string}`;
// "/api/users" ✅
// "/api/users/1" ✅
// "/literally/anything" ✅  — too permissive

// ✅ MORE SPECIFIC — enumerate the valid segments
type ApiVersion = "v1" | "v2";
type ApiResource = "users" | "posts" | "orders";
type ApiPath2 = `/api/${ApiVersion}/${ApiResource}`;
// Only valid: "/api/v1/users", "/api/v1/posts", etc. — exact combinations only

// Balance: use `${string}` for the parts you don't control (IDs, slugs),
// use unions for the parts you DO control (versions, resources):
type ResourcePath = `/api/${ApiVersion}/${ApiResource}/${number}`;
// "/api/v1/users/42" ✅
// "/api/v2/orders/100" ✅
// "/api/v1/admins/1" ❌ "admins" not in ApiResource
```

---

## Practice exercises

### Exercise 1 — easy

Define the following literal types for a REST API server:
- `HttpMethod` — the 5 standard methods
- `ContentType` — `"application/json"`, `"text/plain"`, `"multipart/form-data"`
- `CacheControl` — `"no-cache"`, `"no-store"`, `"max-age=3600"`, `"public"`, `"private"`

Write a function `buildRequestHeaders(method: HttpMethod, contentType: ContentType, cache: CacheControl): Record<string, string>` that returns a headers object. Call it with both valid and intentionally invalid arguments to verify the type errors.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed status code system using `as const` and `typeof`:

```ts
const STATUS_CODES = {
  OK: 200,
  CREATED: 201,
  NO_CONTENT: 204,
  BAD_REQUEST: 400,
  UNAUTHORIZED: 401,
  FORBIDDEN: 403,
  NOT_FOUND: 404,
  CONFLICT: 409,
  INTERNAL_ERROR: 500,
} as const;
```

1. Derive a `StatusCode` type from `STATUS_CODES` using `typeof STATUS_CODES[keyof typeof STATUS_CODES]`.
2. Derive a `StatusCodeName` type (the keys: `"OK" | "CREATED" | ...`) using `keyof typeof STATUS_CODES`.
3. Write a function `getStatusMessage(code: StatusCode): string` — use a switch statement that handles every status code. Add a `never` exhaustive check in the default case.
4. Write a function `getStatusCode(name: StatusCodeName): StatusCode` — returns `STATUS_CODES[name]`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed event bus system using template literal types.

1. Define these `as const` arrays: `RESOURCES = ["user", "product", "order"]` and `ACTIONS = ["created", "updated", "deleted", "listed"]`.
2. Derive `Resource` and `Action` types from them.
3. Create a `DomainEventName = `${Resource}:${Action}`` template literal type.
4. Define a `EventPayloadMap` — a type that maps at least 4 specific event names to their payload shapes (e.g., `"user:created"` → `{ userId: number; email: string }`, `"order:created"` → `{ orderId: number; total: number; userId: number }`).
5. Write a generic `EventBus` class with:
   - `on<T extends DomainEventName>(event: T, handler: (payload: T extends keyof EventPayloadMap ? EventPayloadMap[T] : Record<string, unknown>) => void): void`
   - `emit<T extends DomainEventName>(event: T, payload: T extends keyof EventPayloadMap ? EventPayloadMap[T] : Record<string, unknown>): void`
   - An internal `Map` to store handlers by event name
6. Demonstrate the class is correctly typed: subscribing to `"user:created"` should give the handler a typed payload with `userId` and `email`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Pattern | Syntax | Notes |
|---------|--------|-------|
| String literal union | `"GET" \| "POST" \| "DELETE"` | Most common literal pattern |
| Numeric literal union | `200 \| 404 \| 500` | Status codes, API versions |
| Lock variable to literal | `const x = "GET"` | const infers literals |
| Lock let to literal | `let x = "GET" as const` | Use `as const` |
| Readonly literal object | `{ ... } as const` | All values become readonly literals |
| Derive union from object | `typeof OBJ[keyof typeof OBJ]` | Single source of truth |
| Derive keys as union | `keyof typeof OBJ` | |
| Template literal type | `\`${Resource}:${Action}\`` | Auto-generates all combinations |
| Array element union | `typeof ARR[number]` | Works on `as const` arrays |

## Connected topics

- **08 — Type inference** — why `const` infers literal types and `let` doesn't.
- **09 — Union types** — literal types are almost always combined into unions.
- **42 — Discriminated unions** — the `kind: "pending"` discriminant field is a literal type.
- **43 — Mapped types** — iterating over literal union keys to build new types.
- **45 — Template literal types** — the deep dive into `${A}${B}` type-level string building.
