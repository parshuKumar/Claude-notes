# 15 — type vs interface

## What is this?

TypeScript gives you two ways to name and describe a type: the `type` keyword (type alias) and the `interface` keyword. For simple object shapes they look nearly identical and are often interchangeable. But they have meaningful differences in capability, behaviour, and intent. This topic maps every difference precisely so you know exactly when to reach for each one.

A **type alias** (`type`) gives a name to *any* TypeScript type — primitives, unions, intersections, function signatures, tuples, conditionals, mapped types, and object shapes. It is a pure compile-time alias.

An **interface** (`interface`) declares a *named object type* specifically — properties, methods, index signatures, and call signatures. It participates in TypeScript's structural type system as a named entity that can be reopened, merged, and implemented by classes.

## Why does it matter?

The choice between `type` and `interface` affects:
- **Extensibility** — interfaces can be reopened (declaration merging); type aliases cannot.
- **Capability** — type aliases can express things interfaces simply cannot: unions, tuples, mapped types, conditional types.
- **Error messages** — TypeScript often shows interface names directly in errors (clearer); type alias errors can expand to their full structure (sometimes noisier).
- **Architectural intent** — `interface` signals a contract that classes will `implement`; `type` signals a shape or computed transformation.

Getting this wrong costs you: you either lose declaration merging you needed, or you write verbose interfaces where a two-line `type` would have done the job.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no type system, so none of this exists
// You just pass objects and hope for the best
function processRequest(req) {
  // req could be anything
}
```

```ts
// TypeScript — you choose type or interface to describe the shape

// With interface:
interface ApiRequest {
  userId: number;
  endpoint: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
}

// With type alias:
type ApiRequest = {
  userId: number;
  endpoint: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
};

// For a plain object shape: functionally identical.
// The differences appear when you extend, merge, or describe non-object types.
```

---

## Syntax

```ts
// ── TYPE ALIAS ─────────────────────────────────────────────────────────────
type UserId = number;                         // primitive alias
type ApiStatus = "success" | "error";         // union — interface CANNOT do this
type RequestHandler = (req: unknown) => void; // function — interface can but awkwardly
type Pair<T> = [T, T];                        // tuple — interface CANNOT do this
type StringOrNumber = string | number;        // union — interface CANNOT do this

type User = {                                 // object shape — same as interface
  id: number;
  name: string;
};

// ── INTERFACE ──────────────────────────────────────────────────────────────
interface User {                              // object shape — same as type alias
  id: number;
  name: string;
}

interface UserService {                       // method contracts — typical interface use
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
}
```

---

## How it works — the differences one by one

### Difference 1 — What they can describe

```ts
// TYPE ALIAS can describe anything:
type UserId = number;
type NullableString = string | null;
type ApiResponse<T> = { success: true; data: T } | { success: false; error: string };
type Middleware = (req: Request, res: Response, next: NextFunction) => void;
type Pair<T> = [T, T];
type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> }; // mapped type

// INTERFACE can only describe object shapes (including call/construct signatures):
interface UserShape { id: number; name: string; }  // ✅
interface CallableLogger { (msg: string): void; level: "info" | "error"; } // ✅ call sig

// These are IMPOSSIBLE with interface:
interface UserId = number;          // ❌ Syntax error
interface NullableString = string | null; // ❌ Syntax error — no unions
```

### Difference 2 — Declaration merging (interfaces only)

Interfaces with the same name in the same scope **automatically merge** into one type. Type aliases with the same name cause an error.

```ts
// ── DECLARATION MERGING — interface ───────────────────────────────────────
interface ServerConfig {
  host: string;
  port: number;
}

// Declared again — TypeScript MERGES them, not overrides:
interface ServerConfig {
  protocol: "http" | "https";
  timeout?: number;
}

// Resulting merged type:
// interface ServerConfig {
//   host: string;
//   port: number;
//   protocol: "http" | "https";
//   timeout?: number;
// }

const config: ServerConfig = {
  host: "localhost",
  port: 3000,
  protocol: "https",   // required from second declaration
};

// ── WHY THIS MATTERS — library augmentation ────────────────────────────────
// Express's Request interface is declared in @types/express.
// You can ADD properties to it in your own code:
declare global {
  namespace Express {
    interface Request {
      userId?: number;    // now req.userId is typed everywhere in your app
      authToken?: string;
    }
  }
}

// ── TYPE ALIAS — no merging, just an error ─────────────────────────────────
type ServerConfig = { host: string; port: number; };
type ServerConfig = { protocol: string; };  // ❌ Duplicate identifier 'ServerConfig'
```

### Difference 3 — Extending / composing

Both support extending but with different syntax and subtly different semantics.

```ts
// ── EXTENDING — interface uses 'extends' ────────────────────────────────────
interface Entity {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
}

interface User extends Entity {
  email: string;
  name: string;
}

interface AdminUser extends User {
  role: "admin";
  permissions: string[];
}

// Multi-extend:
interface AuditedUser extends User, Auditable {
  lastAuditedBy: number;
}

// ── EXTENDING — type uses intersection (&) ─────────────────────────────────
type Entity = {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
};

type User = Entity & {
  email: string;
  name: string;
};

type AdminUser = User & {
  role: "admin";
  permissions: string[];
};
```

**Key semantic difference when properties conflict:**

```ts
// ── CONFLICT WITH EXTENDS ──────────────────────────────────────────────────
interface Base {
  id: string;
}

interface Child extends Base {
  id: number;   // ❌ ERROR — 'number' is not assignable to 'string'
  // Interface extending with conflicting property type is a hard error
}

// ── CONFLICT WITH INTERSECTION ─────────────────────────────────────────────
type Base = { id: string };
type Child = Base & { id: number };
// No immediate error, BUT the resulting type of 'id' is: string & number = never
// This silently creates an unusable type — impossible to satisfy

const child: Child = { id: "hello" };  // ❌ 'string' not assignable to 'never'
const child2: Child = { id: 42 };       // ❌ 'number' not assignable to 'never'
// There is no value that satisfies this intersection
```

**Takeaway:** `extends` fails loudly and clearly. `&` may silently produce `never`. Use `extends` for inheritance hierarchies where you want the error caught early.

### Difference 4 — implements (interfaces only in practice)

Classes use `implements` to fulfill an interface contract. You can technically use `implements` with a type alias that describes an object shape, but only interfaces are the idiomatic choice because they're designed as contracts.

```ts
interface UserRepository {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
}

// ✅ Class implementing an interface:
class PostgresUserRepository implements UserRepository {
  async findById(id: number): Promise<User | null> { /* ... */ return null; }
  async create(data: CreateUserBody): Promise<User> { /* ... */ return {} as User; }
}

// ✅ Also works with type alias (but uncommon — prefer interface for this):
type IUserRepository = {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
};
class MongoUserRepository implements IUserRepository { /* ... */ }
```

### Difference 5 — Generic syntax and constraints

Both support generics with identical syntax:

```ts
// Identical generics syntax:
interface ApiResponse<T> {
  success: boolean;
  data: T;
  error?: string;
}

type ApiResponse<T> = {
  success: boolean;
  data: T;
  error?: string;
};

// ONLY type alias can use conditional generics and mapped types:
type NonNullable<T> = T extends null | undefined ? never : T;
type Partial<T> = { [K in keyof T]?: T[K] };       // mapped type — impossible with interface
type Readonly<T> = { readonly [K in keyof T]: T[K] }; // mapped type — impossible with interface
```

### Difference 6 — Error message clarity

```ts
interface User { id: number; name: string; }
type UserAlias = { id: number; name: string; };

function greetUser(user: User): string { return user.name; }
function greetAlias(user: UserAlias): string { return user.name; }

greetUser({ id: 1 });      // Error: Argument ... is missing 'name' (shows "User")
greetAlias({ id: 1 });     // Error: Argument ... is missing 'name' (shows "UserAlias")

// For simple types, nearly identical. But with complex generics:
type DeepPartial<T> = { [K in keyof T]?: DeepPartial<T[K]> };
// Errors involving DeepPartial may expand the whole mapped type inline — can be noisy
// Errors involving interface names stay as the named interface — cleaner
```

---

## The decision guide

| Situation | Use |
|-----------|-----|
| Describing an object shape that a class will `implement` | `interface` |
| Describing a service / repository contract | `interface` |
| Augmenting a third-party type (Express `Request`, etc.) | `interface` (declaration merging) |
| Library type that consumers should be able to extend | `interface` |
| Union type (`string \| number`, `"admin" \| "user"`) | `type` |
| Tuple type (`[string, number]`) | `type` |
| Primitive alias (`type UserId = number`) | `type` |
| Function type (`(req: Request) => void`) | `type` |
| Mapped type (`Partial<T>`, `Readonly<T>`) | `type` |
| Conditional type (`T extends U ? X : Y`) | `type` |
| Intersection of several shapes (quick composition) | `type` with `&` |
| Extending with property conflict detection | `interface` with `extends` |

**The pragmatic rule used by most TypeScript codebases:**
> Use `interface` for anything that defines a contract (objects, services, repositories, anything a class will implement or another interface will extend). Use `type` for everything else (unions, tuples, utility compositions, function types, mapped/conditional types).

---

## Example 1 — basic

```ts
// Showing both in the same codebase — where each belongs

// ── type aliases for non-object types ─────────────────────────────────────
type UserId = number;
type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type RouteHandler = (req: Request, res: Response) => Promise<void>;
type Result<T> = { success: true; data: T } | { success: false; error: string };

// ── interfaces for object shape contracts ─────────────────────────────────
interface RequestContext {
  userId: number;
  authToken: string;
  ipAddress: string;
  requestId: string;
}

interface PaginationOptions {
  page: number;
  pageSize: number;
}

interface PaginatedResult<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  hasNextPage: boolean;
}

// ── interface extends interface ─────────────────────────────────────────────
interface BaseEntity {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
}

interface User extends BaseEntity {
  email: string;
  name: string;
  role: "admin" | "editor" | "viewer";
  deletedAt: Date | null;
}

// ── type alias for a union of interfaces ──────────────────────────────────
// (can't do this with interface)
type AdminOrEditor = User & { role: "admin" | "editor" };

// ── functions use both ─────────────────────────────────────────────────────
async function listUsers(
  options: PaginationOptions,        // interface
  ctx: RequestContext                // interface
): Promise<PaginatedResult<User>> { // interface
  // Implementation
  return {
    items: [],
    total: 0,
    page: options.page,
    pageSize: options.pageSize,
    hasNextPage: false,
  };
}
```

---

## Example 2 — real world backend use case

```ts
// Augmenting Express Request — requires interface declaration merging

// ── src/types/express.d.ts ────────────────────────────────────────────────
import { User } from './user';

declare global {
  namespace Express {
    interface Request {
      user?: User;          // set by auth middleware
      requestId: string;    // set by logging middleware
      startTime: number;    // set by timing middleware
    }
  }
}

// This ONLY works because Express.Request is an interface — it can be merged.
// If Express had used a type alias, this augmentation would be impossible.

// ── src/middleware/auth.ts ────────────────────────────────────────────────
import { Request, Response, NextFunction } from 'express';
import { UserRepository } from './interfaces';

// Service contract — interface (will be implemented by class, used by middleware)
interface AuthService {
  verifyToken(token: string): Promise<{ userId: number; valid: boolean }>;
}

// Middleware factory — takes an interface, returns a RouteHandler (type alias)
type Middleware = (req: Request, res: Response, next: NextFunction) => Promise<void>;

function makeAuthMiddleware(authService: AuthService): Middleware {
  return async (req, res, next) => {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith("Bearer ")) {
      res.status(401).json({ error: "Missing auth token" });
      return;
    }

    const token = authHeader.slice(7);
    const result = await authService.verifyToken(token);

    if (!result.valid) {
      res.status(401).json({ error: "Invalid token" });
      return;
    }

    req.requestId = crypto.randomUUID(); // ✅ typed from declaration merging
    next();
  };
}

// ── Generic API response — type alias (union — interface can't do this) ────
type ApiResponse<T> =
  | { success: true; data: T; requestId: string }
  | { success: false; error: string; code: number; requestId: string };

// ── Utility helpers — type aliases (mapped/conditional — interface can't) ──
type CreateBody<T extends BaseEntity> = Omit<T, "id" | "createdAt" | "updatedAt">;
type UpdateBody<T extends BaseEntity> = Partial<Omit<T, "id" | "createdAt">>;

// Using both together:
interface UserController {
  getUser(req: Request, res: Response): Promise<void>;
  createUser(req: Request, res: Response): Promise<void>;
  updateUser(req: Request, res: Response): Promise<void>;
  deleteUser(req: Request, res: Response): Promise<void>;
}

// type aliases drive the shape of request bodies:
type CreateUserRequest = CreateBody<User>;   // { email, name, role, deletedAt }
type UpdateUserRequest = UpdateBody<User>;   // { email?, name?, role?, deletedAt? }
```

---

## Common mistakes

### Mistake 1 — Using interface where only type alias works

```ts
// ❌ Trying to make a union with interface:
interface Status = "active" | "inactive";  // ❌ Syntax error

interface RequestBody = CreateUserBody | UpdateUserBody; // ❌ Syntax error

// ✅ Use type alias for unions:
type Status = "active" | "inactive";
type RequestBody = CreateUserBody | UpdateUserBody;
```

### Mistake 2 — Using type alias when you need declaration merging

```ts
// Scenario: you use a library that puts config on the Express Request object
// and you want to add your own field

// ❌ If you try to augment with type alias — impossible
type Request = Express.Request & { userId: number }; // doesn't augment the original type
// This creates a NEW local type alias, doesn't change how Express.Request behaves globally

// ✅ You must use interface merging:
declare global {
  namespace Express {
    interface Request {
      userId?: number;   // merges into Express.Request globally
    }
  }
}
// Now every req.userId is typed everywhere — no local type alias can do this
```

### Mistake 3 — Intersection silently producing `never` vs extends catching it

```ts
// ❌ DANGEROUS — silent never from type intersection:
type A = { id: string };
type B = { id: number };
type C = A & B;

const c: C = { id: "hello" }; // ❌ 'string' not assignable to 'never'
const c2: C = { id: 42 };      // ❌ 'number' not assignable to 'never'
// TypeScript didn't warn you when you wrote `A & B` — only when you tried to use it

// ✅ Use extends to catch the conflict at definition time:
interface A { id: string; }
interface B extends A {
  id: number;   // ❌ IMMEDIATE error: 'number' is not assignable to type 'string'
}
// TypeScript stops you right here — no silent never
```

---

## Practice exercises

### Exercise 1 — easy

You have these two descriptions — decide which keyword to use for each, implement them, then write a short comment explaining your choice:

1. A named type for an HTTP status code: `200 | 201 | 400 | 401 | 403 | 404 | 500`
2. An object shape for a database row that a `UserRepo` class will implement
3. A function type: receives a `userId: number` and returns `Promise<boolean>`
4. An object shape for request pagination options (`page`, `pageSize`, optional `sortBy`, optional `sortOrder`)
5. A union of two different API response shapes: success with data, or failure with an error message

```ts
// Write your code here
```

### Exercise 2 — medium

Demonstrate declaration merging. You have a base `AppConfig` interface:

```ts
interface AppConfig {
  appName: string;
  version: string;
  port: number;
}
```

In a separate block (simulating a different file), merge in database config fields: `dbHost`, `dbPort`, `dbName`. In another block, merge in Redis config: `redisHost`, `redisPort`, optional `redisPassword`.

Show that a single `AppConfig` object must satisfy all three merged declarations.

Then show that the same thing is **impossible** with a `type` alias — write the attempt and the error comment.

Finally, write a function `printConfig(config: AppConfig): void` that logs all fields.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed event system using both `type` and `interface` where each is the correct choice:

1. Define a `type` alias `EventName` that is a union of all valid event strings: `"user.created"`, `"user.updated"`, `"user.deleted"`, `"order.placed"`, `"order.fulfilled"`, `"payment.received"`, `"payment.failed"`

2. Define `type` aliases for each event's payload:
   - `UserCreatedPayload` — `{ userId: number; email: string; name: string; }`
   - `UserUpdatedPayload` — `{ userId: number; changedFields: string[]; }`
   - `OrderPlacedPayload` — `{ orderId: number; userId: number; totalAmount: number; items: number; }`
   - (and so on for all 7 events)

3. Define a `type` alias `EventPayloadMap` that maps each `EventName` to its payload type (use an object type with all 7 keys)

4. Define an `interface` `EventHandler<T>` with a single method: `handle(payload: T): Promise<void>`

5. Define an `interface` `EventBus` with: `publish<E extends EventName>(event: E, payload: EventPayloadMap[E]): Promise<void>` and `subscribe<E extends EventName>(event: E, handler: EventHandler<EventPayloadMap[E]>): void`

6. Implement `InMemoryEventBus` that implements `EventBus`. Use a `Map<string, EventHandler<unknown>[]>` internally.

7. Write two concrete handler classes implementing `EventHandler`: one for `"user.created"` that logs the new user, one for `"order.placed"` that logs the order total.

8. Wire it all up: create the bus, subscribe both handlers, publish one `"user.created"` event and one `"order.placed"` event.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Capability | `type` | `interface` |
|------------|--------|-------------|
| Object shape | ✅ | ✅ |
| Union (`A \| B`) | ✅ | ❌ |
| Intersection (`A & B`) | ✅ | ❌ (use `extends`) |
| Tuple (`[A, B]`) | ✅ | ❌ |
| Primitive alias | ✅ | ❌ |
| Mapped type (`[K in keyof T]`) | ✅ | ❌ |
| Conditional type (`T extends U ?`) | ✅ | ❌ |
| Function type | ✅ | ✅ (call signature) |
| Declaration merging | ❌ | ✅ |
| `implements` by class | ✅ (works) | ✅ (idiomatic) |
| `extends` another type | ✅ via `&` | ✅ via `extends` |
| Conflict detection on extend | ❌ (silent `never`) | ✅ (immediate error) |
| Generic parameters | ✅ | ✅ |
| Recursive types | ✅ | ✅ |

## Connected topics

- **14 — Interfaces** — the full interface reference (optional, readonly, methods, index signatures).
- **10 — Intersection types** — how `A & B` works and the `never` trap.
- **09 — Union types** — why only `type` can express unions.
- **18 — Extending interfaces** — `interface Child extends Parent` in depth.
- **32 — Utility types** — `Partial`, `Pick`, `Omit`, `Record` — all built from mapped types (only possible with `type`).
- **40 — Mapped types** — building your own `Partial<T>` and `Readonly<T>`.
- **41 — Conditional types** — `T extends U ? X : Y` — only `type` can do this.
