# 12 — Type Aliases

## What is this?

A type alias gives a **name to any type** — primitive, union, intersection, object shape, function signature, tuple, or anything else. You write `type MyType = ...` and from that point you can use `MyType` anywhere instead of repeating the full type definition. Type aliases are the primary tool for creating reusable, readable, self-documenting types throughout a TypeScript codebase.

## Why does it matter?

Without type aliases, complex types become inline noise repeated everywhere:

```ts
// Without aliases — the same ugly type written three times
function createUser(data: { name: string; email: string; role: "admin" | "editor" | "viewer" }): { id: number; name: string; email: string; role: "admin" | "editor" | "viewer"; createdAt: Date } { ... }
function updateUser(id: number, data: { name: string; email: string; role: "admin" | "editor" | "viewer" }): { id: number; name: string; email: string; role: "admin" | "editor" | "viewer"; createdAt: Date } { ... }
```

With type aliases, the code is clean and the type is defined once:

```ts
type UserRole = "admin" | "editor" | "viewer";
type CreateUserBody = { name: string; email: string; role: UserRole };
type User = { id: number; name: string; email: string; role: UserRole; createdAt: Date };

function createUser(data: CreateUserBody): User { ... }
function updateUser(id: number, data: Partial<CreateUserBody>): User { ... }
```

Any change to the type is made in one place and propagates everywhere automatically.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no types, no aliases, you document shapes in comments (if at all)
/**
 * @param {Object} data
 * @param {string} data.name
 * @param {string} data.email
 */
function createUser(data) {
  return { id: Date.now(), ...data };
}
// Comment can lie, drift out of sync, and gives no autocomplete
```

```ts
// TypeScript — the type alias IS the documentation, enforced by the compiler
type CreateUserBody = {
  name: string;
  email: string;
};

type User = {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
};

function createUser(data: CreateUserBody): User {
  return { id: Date.now(), ...data, createdAt: new Date() };
}
// The type IS the documentation — and it's enforced, not just text
```

---

## Syntax

```ts
// ── OBJECT TYPE ALIAS ─────────────────────────────────────────────────────
type User = {
  id: number;
  name: string;
  email: string;
  role: "admin" | "user";
  createdAt: Date;
};

// ── UNION TYPE ALIAS ──────────────────────────────────────────────────────
type UserId = string | number;
type UserRole = "admin" | "editor" | "viewer";
type NullableString = string | null;
type OptionalNumber = number | undefined;

// ── PRIMITIVE ALIAS ───────────────────────────────────────────────────────
type Email = string;          // gives semantic meaning to a plain string
type Milliseconds = number;   // documents the unit of measurement

// ── FUNCTION TYPE ALIAS ───────────────────────────────────────────────────
type RequestHandler = (req: Request, res: Response) => void;
type Predicate<T> = (value: T) => boolean;
type AsyncOperation<T> = () => Promise<T>;

// ── TUPLE TYPE ALIAS ──────────────────────────────────────────────────────
type Pair<T, U> = [T, U];
type Coordinates = [number, number];           // [latitude, longitude]
type HttpResult = [number, string];            // [statusCode, message]

// ── INTERSECTION TYPE ALIAS ───────────────────────────────────────────────
type AdminUser = User & { permissions: string[] };

// ── GENERIC TYPE ALIAS ────────────────────────────────────────────────────
type ApiResponse<T> = {
  success: boolean;
  data: T | null;
  error: string | null;
};
```

---

## How it works — line by line

```ts
type UserRole = "admin" | "editor" | "viewer";
//   ^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//   name       the type being named (a union of string literals)
```

- The `type` keyword starts the alias declaration.
- The name follows (by convention: `PascalCase` for object/complex types, sometimes `camelCase` for simple aliases).
- The `=` assigns the type definition.
- After the declaration, `UserRole` can be used anywhere a type is expected.

```ts
type ApiResponse<T> = {
  success: boolean;
  data: T | null;       // T is a type parameter — filled in at use site
  error: string | null;
};

// Using it:
type UserResponse = ApiResponse<User>;
// Expands to: { success: boolean; data: User | null; error: string | null }

type UserListResponse = ApiResponse<User[]>;
// Expands to: { success: boolean; data: User[] | null; error: string | null }
```

---

## Type aliases are just names — not new types

An important subtlety: a type alias does **not** create a distinct new type at runtime. It's purely a compile-time label. Two different aliases for the same shape are interchangeable:

```ts
type Email = string;
type Username = string;

function sendEmail(to: Email): void { console.log(to); }

const username: Username = "parsh";
sendEmail(username);   // ✅ — Email and Username are both string — fully interchangeable

// This is different from branded types (which do create distinctness — a later topic)
```

This is why `type Email = string` gives semantic meaning but not enforcement — it documents intent, but TypeScript treats it as `string` underneath. For true enforcement you need branded types (an advanced pattern).

---

## Naming conventions

```ts
// Object shapes — PascalCase:
type User = { ... };
type ApiResponse<T> = { ... };
type DatabaseConfig = { ... };

// Union types — PascalCase:
type UserRole = "admin" | "editor";
type HttpMethod = "GET" | "POST";
type HttpStatus = 200 | 404 | 500;

// Function types — PascalCase or descriptive:
type RequestHandler = (req: Request, res: Response) => void;
type Comparator<T> = (a: T, b: T) => number;

// Semantic primitives — PascalCase or camelCase:
type UserId = number;
type Email = string;
type ISODateString = string;

// Utility / transformation types — PascalCase:
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type MaybeArray<T> = T | T[];
```

---

## Type aliases for functions

```ts
// A typed callback:
type OnSuccess<T> = (data: T) => void;
type OnError = (error: Error) => void;

// A typed middleware function:
type Middleware = (req: Request, res: Response, next: NextFunction) => void;

// An async operation that returns a result:
type DbQuery<T> = (id: number) => Promise<T | null>;

// Using function type aliases:
function withErrorHandling(
  handler: RequestHandler,    // reuse the alias
  onError: OnError
): RequestHandler {
  return (req, res) => {
    try {
      handler(req, res);
    } catch (err) {
      onError(err instanceof Error ? err : new Error(String(err)));
    }
  };
}
```

---

## Building a shared types file

In a real project you create a `src/types/index.ts` that exports all shared types:

```ts
// src/types/index.ts

// ── Primitive aliases (semantic meaning) ─────────────────────────────────
export type UserId = number;
export type Email = string;
export type ISODateString = string;
export type JwtToken = string;

// ── Roles and permissions ─────────────────────────────────────────────────
export type UserRole = "admin" | "editor" | "viewer";
export type Permission = "read" | "write" | "delete" | "manage";

// ── Domain types ──────────────────────────────────────────────────────────
export type User = {
  id: UserId;
  email: Email;
  name: string;
  role: UserRole;
  createdAt: Date;
  deletedAt: Date | null;
};

export type CreateUserBody = Pick<User, "email" | "name" | "role">;
export type UpdateUserBody = Partial<Pick<User, "email" | "name" | "role">>;

// ── API contracts ─────────────────────────────────────────────────────────
export type ApiResponse<T> = {
  success: boolean;
  data: T | null;
  error: string | null;
  timestamp: ISODateString;
};

export type PaginatedResponse<T> = ApiResponse<{
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
}>;

// ── Request types ─────────────────────────────────────────────────────────
export type PaginationParams = {
  page: number;
  pageSize: number;
};

export type SortOrder = "asc" | "desc";

export type SortParams = {
  sortBy: string;
  sortOrder: SortOrder;
};
```

---

## Example 1 — basic

```ts
// All the types needed for a simple auth flow

type Email = string;
type HashedPassword = string;
type JwtToken = string;

type RegisterBody = {
  email: Email;
  password: string;         // plain text on the way in
  name: string;
};

type LoginBody = {
  email: Email;
  password: string;
};

type AuthUser = {
  id: number;
  email: Email;
  name: string;
  role: "admin" | "user";
};

type AuthResult = {
  user: AuthUser;
  token: JwtToken;
  expiresAt: Date;
};

type AuthError = {
  code: "INVALID_CREDENTIALS" | "USER_NOT_FOUND" | "ACCOUNT_LOCKED";
  message: string;
};

// Functions typed with aliases — clean and readable
function login(body: LoginBody): Promise<AuthResult | AuthError> {
  // ...implementation
  return Promise.resolve({ code: "INVALID_CREDENTIALS", message: "Wrong password" });
}

function register(body: RegisterBody): Promise<AuthUser> {
  return Promise.resolve({ id: 1, email: body.email, name: body.name, role: "user" });
}
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── Shared type foundation for a REST API ─────────────────────────────────

type UserId = number;
type PostId = number;
type ISODateString = string;

type User = {
  id: UserId;
  email: string;
  name: string;
  role: "admin" | "author" | "reader";
  createdAt: Date;
};

type Post = {
  id: PostId;
  title: string;
  content: string;
  authorId: UserId;
  status: "draft" | "published" | "archived";
  publishedAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
};

// ── Reusable API response wrapper ──────────────────────────────────────────

type ApiResponse<T> = {
  success: true;
  data: T;
  requestId: string;
  timestamp: ISODateString;
};

type ApiError = {
  success: false;
  error: string;
  code: string;
  requestId: string;
  timestamp: ISODateString;
};

type ApiResult<T> = ApiResponse<T> | ApiError;

// ── Route-specific request body types ────────────────────────────────────

type CreatePostBody = {
  title: string;
  content: string;
  status: "draft" | "published";
};

type UpdatePostBody = Partial<CreatePostBody>;   // all fields optional

type PostListQuery = {
  page?: number;
  pageSize?: number;
  status?: Post["status"];    // reuse the literal union from Post directly
  authorId?: UserId;
};

// ── Typed middleware using aliases ────────────────────────────────────────

type AuthenticatedRequest = Request & { user: User };

type RouteHandler = (req: Request, res: Response) => Promise<void>;
type AuthRouteHandler = (req: AuthenticatedRequest, res: Response) => Promise<void>;

// ── Factory functions using all the types ─────────────────────────────────

function buildApiResponse<T>(data: T, requestId: string): ApiResponse<T> {
  return {
    success: true,
    data,
    requestId,
    timestamp: new Date().toISOString(),
  };
}

function buildApiError(error: string, code: string, requestId: string): ApiError {
  return {
    success: false,
    error,
    code,
    requestId,
    timestamp: new Date().toISOString(),
  };
}

// ── A route handler — everything typed through aliases ────────────────────

const createPostHandler: AuthRouteHandler = async (req, res) => {
  const body = req.body as CreatePostBody;
  const requestId = (req as Request & { requestId: string }).requestId;

  const newPost: Post = {
    id: Date.now(),
    title: body.title,
    content: body.content,
    authorId: req.user.id,
    status: body.status,
    publishedAt: body.status === "published" ? new Date() : null,
    createdAt: new Date(),
    updatedAt: new Date(),
  };

  res.status(201).json(buildApiResponse(newPost, requestId));
};
```

---

## Common mistakes

### Mistake 1 — Redefining the same shape in multiple places instead of aliasing once

```ts
// ❌ WRONG — UserRole defined three separate times in three files
// controllers/userController.ts:
function createUser(data: { name: string; email: string; role: "admin" | "editor" | "viewer" }) { ... }

// middleware/auth.ts:
function checkRole(requiredRole: "admin" | "editor" | "viewer") { ... }

// models/userModel.ts:
type UserDoc = { role: "admin" | "editor" | "viewer"; /* ... */ };

// Add "moderator" to one and forget the other two → silent inconsistency

// ✅ RIGHT — define once, import everywhere
// types/index.ts:
export type UserRole = "admin" | "editor" | "viewer";

// Then in all other files:
import { UserRole } from '../types';
function createUser(data: { name: string; email: string; role: UserRole }) { ... }
function checkRole(requiredRole: UserRole) { ... }
type UserDoc = { role: UserRole };
```

### Mistake 2 — Using type aliases as interfaces for class-like structures

```ts
// ❌ NOT WRONG but suboptimal — type aliases can't be extended via declaration merging
// (covered in topic 15), and interfaces integrate better with classes

type UserService = {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
  delete(id: number): Promise<void>;
};

// ✅ BETTER for "contracts" that classes implement — use interface (topic 14)
interface UserService {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
  delete(id: number): Promise<void>;
}

// Type aliases are best for:
//   - Unions, intersections, primitives, tuples, template literals
//   - Complex computed types (Partial<T>, Pick<T, K>, etc.)
// Interfaces are best for:
//   - Object shapes that classes will implement
//   - Shapes that might need declaration merging
```

### Mistake 3 — Deeply nesting inline types instead of composing aliases

```ts
// ❌ WRONG — one giant inlined type that's impossible to read or reuse
type AppState = {
  user: {
    id: number;
    email: string;
    profile: {
      firstName: string;
      lastName: string;
      avatar: string | null;
      preferences: {
        theme: "light" | "dark";
        language: string;
        notifications: { email: boolean; sms: boolean; push: boolean };
      };
    };
  };
  posts: Array<{ id: number; title: string; createdAt: Date }>;
};

// ✅ RIGHT — compose from named pieces
type NotificationPrefs = { email: boolean; sms: boolean; push: boolean };
type UserPreferences = { theme: "light" | "dark"; language: string; notifications: NotificationPrefs };
type UserProfile = { firstName: string; lastName: string; avatar: string | null; preferences: UserPreferences };
type User = { id: number; email: string; profile: UserProfile };
type PostSummary = { id: number; title: string; createdAt: Date };
type AppState = { user: User; posts: PostSummary[] };
```

---

## Practice exercises

### Exercise 1 — easy

Create a `types/index.ts` for a simple product catalog API. Define:

- `ProductId` (number alias)
- `Currency` (string literal union: `"USD" | "EUR" | "GBP" | "INR"`)
- `ProductCategory` (at least 4 categories of your choice as a string literal union)
- `Product` type with: `id`, `name`, `description`, `price`, `currency`, `category`, `inStock`, `createdAt`
- `CreateProductBody` — same as Product but without `id` and `createdAt`
- `UpdateProductBody` — all fields of `CreateProductBody` are optional

Write a function `createProduct(body: CreateProductBody): Product` that fills in `id` (use `Date.now()`) and `createdAt`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed HTTP client using type aliases. Define:

- `HttpMethod` literal union
- `RequestConfig` type with: `url`, `method`, `headers` (optional, `Record<string, string>`), `body` (optional, `unknown`), `timeoutMs` (optional, number)
- `HttpResponse<T>` with: `statusCode`, `headers` (`Record<string, string>`), `data: T`, `durationMs: number`
- `RequestMiddleware` function type alias: takes a `RequestConfig` and returns `RequestConfig` (used to transform a request before sending)
- `ResponseMiddleware<T>` function type alias: takes `HttpResponse<T>` and returns `HttpResponse<T>`

Write a function `applyRequestMiddlewares(config: RequestConfig, middlewares: RequestMiddleware[]): RequestConfig` that reduces the middlewares over the config.

Write two concrete middleware functions:
1. `addAuthHeader(token: string): RequestMiddleware` — returns a middleware that adds `Authorization: Bearer ${token}` to headers
2. `addJsonContentType: RequestMiddleware` — adds `Content-Type: application/json` to headers

Demonstrate composing them.

```ts
// Write your code here
```

### Exercise 3 — hard

Design the complete type system for a multi-tenant SaaS API. All types should be in one file. Define:

- Semantic primitives: `TenantId`, `UserId`, `ResourceId` (all `number`), `ISODateString` (`string`), `JwtToken` (`string`)
- `Plan` literal union: `"free" | "pro" | "enterprise"`
- `Tenant` type: `id`, `name`, `plan`, `createdAt`, `isActive`
- `TenantUser` type: `userId`, `tenantId`, `role: "owner" | "admin" | "member"`, `joinedAt`
- `Resource` type: `id`, `tenantId`, `name`, `type: "document" | "image" | "video"`, `sizeBytes`, `uploadedBy`, `createdAt`
- `ApiResponse<T>` with success/error/data/timestamp
- `PaginatedData<T>` with items, total, page, pageSize, totalPages
- `AuditLog` type: `id`, `tenantId`, `userId`, `action: \`${TenantUser["role"]}:${"read" | "write" | "delete"}\``, `resourceId`, `timestamp`

Write three functions fully typed using these aliases:
1. `createTenant(name: string, plan: Plan): Tenant`
2. `addUserToTenant(tenantId: TenantId, userId: UserId, role: TenantUser["role"]): TenantUser`
3. `buildAuditLog(tenantId: TenantId, userId: UserId, action: AuditLog["action"], resourceId: ResourceId): AuditLog`

No `any`. Every function must have fully typed parameters and return types that use your aliases.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Pattern | Syntax |
|---------|--------|
| Object type | `type User = { id: number; name: string }` |
| Union alias | `type Role = "admin" \| "user"` |
| Primitive semantic alias | `type UserId = number` |
| Function type | `type Handler = (req: Request, res: Response) => void` |
| Generic type alias | `type ApiResponse<T> = { data: T; success: boolean }` |
| Intersection alias | `type AdminUser = User & { permissions: string[] }` |
| Tuple alias | `type Pair = [string, number]` |
| Reuse property type | `type PostStatus = Post["status"]` |
| Derive from utility type | `type CreateBody = Omit<User, "id" \| "createdAt">` |

| Convention | Rule |
|-----------|------|
| Casing | PascalCase for all type alias names |
| Location | Keep shared types in `src/types/index.ts` |
| Scope | Narrow types live near their usage; shared types in types/ |
| Composition | Build complex types from small named parts |
| vs interface | Use `type` for unions, intersections, primitives, computed types |

## Connected topics

- **14 — Interfaces** — the `interface` keyword; when to use `interface` vs `type`.
- **15 — type vs interface** — the actual differences, when each is the right choice.
- **32 — Utility types** — `Partial<T>`, `Pick<T, K>`, `Omit<T, K>` — built-in type aliases that transform other types.
- **28 — Generic interfaces** — parameterized types like `ApiResponse<T>` in full depth.
