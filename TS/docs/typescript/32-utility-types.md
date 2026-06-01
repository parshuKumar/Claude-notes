# 32 — Utility Types

## What is this?

**Utility types** are generic types built into TypeScript's standard library that transform an existing type into a new one. They are all defined using mapped types and conditional types under the hood (topics 41–42), but you don't need to understand their internals to use them effectively — you just apply them like functions at the type level.

They are organized into two main families:

1. **Object-transforming utilities** — modify the shape, optionality, or mutability of an object type: `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`.
2. **Type-set utilities** — work on union types and function types: `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `InstanceType`, `Awaited`, `ThisParameterType`, `OmitThisParameter`.

## Why does it matter?

Without utility types you'd define every variation of a type manually:

```ts
// Without utility types — write every variant by hand:
interface User          { id: number; name: string; email: string; role: string; createdAt: Date; }
interface UserUpdate    { name?: string; email?: string; role?: string; }          // Partial + Omit
interface UserPublic    { id: number; name: string; }                              // Pick
interface UserReadonly  { readonly id: number; readonly name: string; ... }        // Readonly
```

With utility types, you derive them from the source of truth — the original `User` interface. If `User` changes, all derived types update automatically:

```ts
type UserUpdate   = Partial<Omit<User, "id" | "createdAt">>;
type UserPublic   = Pick<User, "id" | "name">;
type UserReadonly = Readonly<User>;
```

This is DRY at the type level — and it is the standard pattern in every real TypeScript codebase.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no types, no transformations needed (or possible):
function updateUser(id, updates) {
  // updates could be anything — no constraint
}
```

```ts
// TypeScript with utility types — derived, safe, always in sync with source:
interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  passwordHash: string;
  createdAt: Date;
  updatedAt: Date;
}

// These are all derived from User — change User once, all update:
type CreateUserInput  = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; role: ...; passwordHash: string }

type UpdateUserInput  = Partial<Omit<User, "id" | "createdAt" | "updatedAt" | "passwordHash">>;
// { name?: string; email?: string; role?: ... }

type PublicUser       = Omit<User, "passwordHash">;
// { id, name, email, role, createdAt, updatedAt }

type UserSnapshot     = Readonly<User>;
// All properties become readonly

async function updateUser(id: number, input: UpdateUserInput): Promise<User> { ... }
// Only the allowed fields, all optional — TypeScript enforces it
```

---

## Syntax — every utility type

### Object-transforming utilities

```ts
// ── Partial<T> — all properties become optional ───────────────────────────
type UserUpdate = Partial<User>;
// { id?: number; name?: string; email?: string; ... }

// ── Required<T> — all optional properties become required ─────────────────
interface DraftPost { title?: string; body?: string; authorId?: number; }
type FullPost = Required<DraftPost>;
// { title: string; body: string; authorId: number }

// ── Readonly<T> — all properties become readonly ──────────────────────────
type ImmutableConfig = Readonly<AppConfig>;
// { readonly timeout: number; readonly baseUrl: string; ... }

// ── Pick<T, K> — keep only specified keys ─────────────────────────────────
type UserCredentials = Pick<User, "email" | "passwordHash">;
// { email: string; passwordHash: string }

// ── Omit<T, K> — remove specified keys ───────────────────────────────────
type PublicUser = Omit<User, "passwordHash">;
// { id, name, email, role, createdAt, updatedAt }

// ── Record<K, V> — map keys to a value type ──────────────────────────────
type RolePermissions  = Record<"admin" | "editor" | "viewer", string[]>;
type HttpStatusMessages = Record<number, string>;
type UserMap          = Record<string, User>;

// ── Required + Partial combination ───────────────────────────────────────
type PatchInput<T> = Partial<Omit<T, "id" | "createdAt" | "updatedAt">>;
type CreateInput<T> = Omit<T, "id" | "createdAt" | "updatedAt">;
```

### Type-set utilities

```ts
// ── Exclude<T, U> — remove members of U from union T ─────────────────────
type NonAdminRole = Exclude<"admin" | "editor" | "viewer", "admin">;
// "editor" | "viewer"

type StringOrNumber = Exclude<string | number | boolean | null, boolean | null>;
// string | number

// ── Extract<T, U> — keep only members of T that are assignable to U ───────
type NumericTypes = Extract<string | number | boolean | bigint, number | bigint>;
// number | bigint

type AdminOrEditor = Extract<"admin" | "editor" | "viewer", "admin" | "editor">;
// "admin" | "editor"

// ── NonNullable<T> — remove null and undefined ────────────────────────────
type DefiniteString = NonNullable<string | null | undefined>;
// string

type DefiniteUser = NonNullable<User | null | undefined>;
// User

// ── ReturnType<T> — extract the return type of a function type ─────────────
function getUser(): User { ... }
type UserResult = ReturnType<typeof getUser>;
// User

async function fetchPost(): Promise<Post> { ... }
type FetchPostReturn = ReturnType<typeof fetchPost>;
// Promise<Post>   ← use Awaited<> to unwrap:
type PostResult = Awaited<ReturnType<typeof fetchPost>>;
// Post

// ── Parameters<T> — extract the parameter types as a tuple ────────────────
function createOrder(userId: number, items: OrderItem[], coupon?: string): Order { ... }
type CreateOrderParams = Parameters<typeof createOrder>;
// [userId: number, items: OrderItem[], coupon?: string]

type FirstParam = Parameters<typeof createOrder>[0];
// number

// ── InstanceType<T> — extract the instance type of a constructor ──────────
class UserService { ... }
type UserServiceInstance = InstanceType<typeof UserService>;
// UserService

// ── Awaited<T> — unwrap Promise types (recursive) ─────────────────────────
type A = Awaited<Promise<string>>;                  // string
type B = Awaited<Promise<Promise<number>>>;         // number
type C = Awaited<string>;                           // string (non-Promise passes through)

// ── ThisParameterType<T> — extract the this parameter type ───────────────
function log(this: { prefix: string }, msg: string): void { ... }
type LogThis = ThisParameterType<typeof log>;
// { prefix: string }

// ── OmitThisParameter<T> — remove the this parameter from a function type ─
type BoundLog = OmitThisParameter<typeof log>;
// (msg: string) => void  (this removed)
```

---

## How it works — utility by utility

### `Partial<T>` and `Required<T>`

```ts
interface UserProfile {
  id: number;
  name: string;
  email: string;
  bio: string;
  avatarUrl: string;
  website: string;
}

// PATCH endpoint — all fields optional:
type UpdateProfileInput = Partial<Omit<UserProfile, "id">>;
// { name?: string; email?: string; bio?: string; avatarUrl?: string; website?: string }

async function updateProfile(userId: number, input: UpdateProfileInput): Promise<UserProfile> {
  // Only the provided fields are updated
  return { ...existingProfile, ...input, id: userId } as UserProfile;
}

// Draft → Published: draft has all optional, published requires them all:
interface ArticleDraft {
  title?: string;
  body?: string;
  authorId?: number;
  tags?: string[];
  coverImageUrl?: string;
}

type PublishedArticle = Required<ArticleDraft> & { id: number; publishedAt: Date };
// All fields required + id + publishedAt

function publish(draft: Required<ArticleDraft>): PublishedArticle {
  return { ...draft, id: Math.random(), publishedAt: new Date() };
}
```

### `Readonly<T>` — immutable DTOs and config

```ts
interface AppConfig {
  databaseUrl: string;
  port: number;
  jwtSecret: string;
  allowedOrigins: string[];
}

// Config should never be mutated after loading:
type ImmutableConfig = Readonly<AppConfig>;

function loadConfig(): ImmutableConfig {
  return {
    databaseUrl: process.env.DATABASE_URL ?? "postgres://localhost/app",
    port: parseInt(process.env.PORT ?? "3000", 10),
    jwtSecret: process.env.JWT_SECRET ?? "",
    allowedOrigins: ["https://app.dev.io"],
  };
}

const config = loadConfig();
config.port = 8080;  // ❌ Error: Cannot assign to 'port' because it is a read-only property
config.allowedOrigins.push("http://evil.com");  // ⚠️ Readonly is shallow — array contents are still mutable!
// Use `as const` or `Readonly<{ allowedOrigins: readonly string[] }>` for deep immutability
```

### `Pick<T, K>` and `Omit<T, K>` — the most-used utilities

```ts
interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  passwordHash: string;
  lastLoginAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

// Public API response — never expose passwordHash:
type PublicUser = Omit<User, "passwordHash">;

// Auth check — only what you need:
type UserAuth = Pick<User, "id" | "email" | "passwordHash" | "role">;

// JWT payload — minimal:
type JwtPayload = Pick<User, "id" | "email" | "role">;

// DB create — auto-generated fields excluded:
type CreateUserInput = Omit<User, "id" | "lastLoginAt" | "createdAt" | "updatedAt">;

// DB update — auto-generated fields excluded, everything optional:
type UpdateUserInput = Partial<Omit<User, "id" | "passwordHash" | "createdAt" | "updatedAt">>;

// Pick vs Omit: use Pick when you want a SMALL subset; use Omit when you want MOST fields minus a few
```

### `Record<K, V>` — typed dictionaries and lookup tables

```ts
// Permission lookup:
type Permission = "read" | "write" | "delete" | "admin";
type RolePermissions = Record<"admin" | "editor" | "viewer", Permission[]>;

const rolePermissions: RolePermissions = {
  admin:  ["read", "write", "delete", "admin"],
  editor: ["read", "write"],
  viewer: ["read"],
};

// All roles must be present — ❌ Error if any is missing:
const bad: RolePermissions = { admin: ["read"], editor: ["read"] };
// Error: Property 'viewer' is missing

// HTTP status code messages:
type StatusMessages = Record<number, string>;
const messages: StatusMessages = {
  200: "OK",
  201: "Created",
  404: "Not Found",
  500: "Internal Server Error",
};

// Cache keyed by string:
type UserCache = Record<string, User>;

// Indexed by a string literal union — more specific than Record<string, ...>:
type EndpointStats = Record<"GET /users" | "POST /users" | "DELETE /users/:id", {
  calls: number;
  avgMs: number;
}>;
```

### `Exclude<T, U>` and `Extract<T, U>` — filtering unions

```ts
type Role = "admin" | "editor" | "viewer" | "guest";

// Remove specific members:
type AuthenticatedRole = Exclude<Role, "guest">;   // "admin" | "editor" | "viewer"
type NonAdminRole      = Exclude<Role, "admin">;   // "editor" | "viewer" | "guest"

// Keep only matching members:
type PrivilegedRole = Extract<Role, "admin" | "editor">;  // "admin" | "editor"

// Remove nullables from a union:
type EventPayload = string | number | null | undefined | object;
type DefinitePayload = Exclude<EventPayload, null | undefined>;  // string | number | object

// Extract only object types from a union:
type ObjectPayload = Extract<EventPayload, object>;  // object
```

### `NonNullable<T>` — removing `null` and `undefined`

```ts
// Before fetching — type is possibly null:
async function findUser(id: number): Promise<User | null> { ... }

// After null-check — unwrap to User:
async function getUser(id: number): Promise<User> {
  const user = await findUser(id);
  if (!user) throw new Error(`User ${id} not found`);
  return user;  // user: User (narrowed) — but return type declared as User
}

// Use NonNullable in derived types:
type RequiredUserId = NonNullable<User["id"] | null | undefined>;
// number

// Utility to assert non-null:
function assertDefined<T>(value: T, name: string): NonNullable<T> {
  if (value === null || value === undefined) throw new Error(`${name} is required`);
  return value as NonNullable<T>;
}
```

### `ReturnType<T>` and `Parameters<T>` — extract function type info

```ts
// Great for when you don't own the type — infer it from the function:
function createJwtToken(userId: number, role: string, expiresIn: number): string { ... }
function verifyJwtToken(token: string): { userId: number; role: string; exp: number } | null { ... }

type TokenPayload   = ReturnType<typeof verifyJwtToken>;    // { userId: number; role: string; exp: number } | null
type TokenPayloadOk = NonNullable<ReturnType<typeof verifyJwtToken>>; // { userId: number; role: string; exp: number }

type CreateTokenArgs = Parameters<typeof createJwtToken>;   // [userId: number, role: string, expiresIn: number]
type UserId         = Parameters<typeof createJwtToken>[0]; // number

// Very useful for handler/middleware types:
import express from "express";
type RequestHandler = Parameters<typeof express.Router.prototype.get>[1];
// The callback type for Express route handlers — inferred from Express's types

// Capturing async return types:
async function fetchOrdersForUser(userId: number): Promise<Order[]> { ... }
type OrdersResult = Awaited<ReturnType<typeof fetchOrdersForUser>>;  // Order[]
```

### `InstanceType<T>` — extract what `new` produces

```ts
class DatabaseConnection {
  query<T>(sql: string): Promise<T[]> { ... }
  close(): Promise<void> { ... }
}

class RedisClient {
  get(key: string): Promise<string | null> { ... }
  set(key: string, value: string): Promise<void> { ... }
}

// When you have the class constructor but want the instance type:
type DbConnection = InstanceType<typeof DatabaseConnection>;
// DatabaseConnection

// Useful for factory functions that accept constructors:
function createAndTest<T extends abstract new (...args: unknown[]) => unknown>(
  Constructor: T,
): InstanceType<T> {
  return new (Constructor as new () => InstanceType<T>)();
}
```

---

## Example 1 — basic

```ts
// Full set of derived types for a User domain model

interface User {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
  passwordHash: string;
  bio: string | null;
  avatarUrl: string | null;
  lastLoginAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

// ── Create / Update inputs ────────────────────────────────────────────────

// POST /users — all fields required except auto-generated ones:
type CreateUserInput = Omit<User, "id" | "bio" | "avatarUrl" | "lastLoginAt" | "createdAt" | "updatedAt">;
// { name: string; email: string; role: ...; passwordHash: string }

// PATCH /users/:id — everything optional, sensitive fields excluded:
type UpdateUserInput = Partial<Pick<User, "name" | "email" | "role" | "bio" | "avatarUrl">>;
// { name?: string; email?: string; role?: ...; bio?: string|null; avatarUrl?: string|null }

// ── Response shapes ───────────────────────────────────────────────────────

// Public API — no password hash:
type PublicUser = Omit<User, "passwordHash">;

// List response — minimal fields:
type UserListItem = Pick<User, "id" | "name" | "email" | "role" | "createdAt">;

// Auth token payload — minimal and stable:
type JwtPayload = Pick<User, "id" | "email" | "role">;

// ── Immutable snapshots ───────────────────────────────────────────────────

type UserSnapshot = Readonly<User>;  // For caching — prevents accidental mutation

// ── Role operations ───────────────────────────────────────────────────────

type UserRole           = User["role"];                          // "admin" | "editor" | "viewer"
type PrivilegedRole     = Extract<UserRole, "admin" | "editor">; // "admin" | "editor"
type UnprivilegedRole   = Exclude<UserRole, "admin">;            // "editor" | "viewer"

// Permission map — every role must have an entry:
type PermissionMap = Record<UserRole, string[]>;
const permissions: PermissionMap = {
  admin:  ["user:read", "user:write", "user:delete", "post:read", "post:write", "post:delete"],
  editor: ["user:read", "post:read", "post:write"],
  viewer: ["user:read", "post:read"],
};

// ── Function type extraction ──────────────────────────────────────────────

function findUserByEmail(email: string): Promise<User | null> {
  return Promise.resolve(null);
}

function updateUserRole(userId: number, newRole: UserRole, updatedBy: number): Promise<User> {
  return Promise.resolve({} as User);
}

type FindUserResult  = Awaited<ReturnType<typeof findUserByEmail>>;   // User | null
type UpdateRoleArgs  = Parameters<typeof updateUserRole>;             // [number, UserRole, number]
type UpdatedByParam  = Parameters<typeof updateUserRole>[2];          // number
```

---

## Example 2 — real world backend use case

```ts
// Full Express-style API route handler pattern built entirely on utility types

interface Post {
  id: number;
  title: string;
  body: string;
  authorId: number;
  status: "draft" | "review" | "published" | "archived";
  tags: string[];
  viewCount: number;
  publishedAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

// ── All derived types — single source of truth ────────────────────────────

type CreatePostInput = Omit<Post, "id" | "viewCount" | "publishedAt" | "createdAt" | "updatedAt">;
// { title: string; body: string; authorId: number; status: "draft"|...; tags: string[] }

type UpdatePostInput = Partial<Pick<Post, "title" | "body" | "status" | "tags">>;
// { title?: string; body?: string; status?: ...; tags?: string[] }

type PostSummary = Pick<Post, "id" | "title" | "authorId" | "status" | "viewCount" | "publishedAt" | "createdAt">;
// Lightweight for list views

type PublishedPost = Post & { status: "published"; publishedAt: Date };
// Narrowed type — guaranteed published

type PostStatus    = Post["status"];                                    // "draft"|"review"|"published"|"archived"
type LiveStatus    = Extract<PostStatus, "published">;                  // "published"
type EditableStatus = Exclude<PostStatus, "published" | "archived">;   // "draft" | "review"

// ── Generic API response layer ────────────────────────────────────────────

interface ApiResponse<TData = unknown> {
  data: TData;
  status: number;
  message: string;
  timestamp: string;
}

interface PaginatedData<TItem> {
  items: TItem[];
  total: number;
  page: number;
  pageSize: number;
  hasNext: boolean;
}

// Endpoint-specific response types — all derived:
type PostResponse        = ApiResponse<Post>;
type PostSummaryResponse = ApiResponse<PaginatedData<PostSummary>>;
type CreatePostResponse  = ApiResponse<Post>;
type DeletePostResponse  = ApiResponse<null>;

// ── Service layer ─────────────────────────────────────────────────────────

interface PostService {
  getById(id: number): Promise<Post | null>;
  getAll(page: number, pageSize: number): Promise<{ items: Post[]; total: number }>;
  create(input: CreatePostInput): Promise<Post>;
  update(id: number, input: UpdatePostInput): Promise<Post | null>;
  publish(id: number): Promise<PublishedPost | null>;
  delete(id: number): Promise<boolean>;
}

// ── Infer handler types from service ─────────────────────────────────────

type GetByIdReturn   = Awaited<ReturnType<PostService["getById"]>>;    // Post | null
type GetAllReturn    = Awaited<ReturnType<PostService["getAll"]>>;     // { items: Post[]; total: number }
type CreateReturn    = Awaited<ReturnType<PostService["create"]>>;     // Post
type PublishReturn   = Awaited<ReturnType<PostService["publish"]>>;    // PublishedPost | null

type GetByIdParams   = Parameters<PostService["getById"]>;             // [id: number]
type CreateParams    = Parameters<PostService["create"]>;              // [input: CreatePostInput]

// ── HTTP controller — completely derived, zero manual type duplication ─────

class PostController {
  constructor(private readonly service: PostService) {}

  async getPost(id: number): Promise<PostResponse> {
    const post = await this.service.getById(id);
    if (!post) return { data: null as unknown as Post, status: 404, message: "Not found", timestamp: new Date().toISOString() };
    return { data: post, status: 200, message: "ok", timestamp: new Date().toISOString() };
  }

  async getPosts(page: number, pageSize: number): Promise<PostSummaryResponse> {
    const { items, total } = await this.service.getAll(page, pageSize);
    const summaries: PostSummary[] = items.map(p => ({
      id: p.id, title: p.title, authorId: p.authorId,
      status: p.status, viewCount: p.viewCount,
      publishedAt: p.publishedAt, createdAt: p.createdAt,
    }));
    return {
      data: { items: summaries, total, page, pageSize, hasNext: page * pageSize < total },
      status: 200, message: "ok", timestamp: new Date().toISOString(),
    };
  }

  async createPost(input: CreatePostInput): Promise<CreatePostResponse> {
    const post = await this.service.create(input);
    return { data: post, status: 201, message: "Created", timestamp: new Date().toISOString() };
  }

  async updatePost(id: number, input: UpdatePostInput): Promise<PostResponse> {
    const post = await this.service.update(id, input);
    if (!post) return { data: null as unknown as Post, status: 404, message: "Not found", timestamp: new Date().toISOString() };
    return { data: post, status: 200, message: "ok", timestamp: new Date().toISOString() };
  }
}
```

---

## Common mistakes

### Mistake 1 — Using `Partial<T>` on the whole entity including `id`

```ts
interface User { id: number; name: string; email: string; }

// ❌ Partial<User> makes id optional — update handlers shouldn't accept id changes:
async function updateUser(id: number, input: Partial<User>): Promise<User> {
  // input.id could be anything — caller could change the user's id!
  return {} as User;
}

// ✅ Omit id (and other immutable fields) first, then Partial:
async function updateUser(id: number, input: Partial<Omit<User, "id">>): Promise<User> {
  return {} as User;
}
```

### Mistake 2 — Confusing `Exclude` (works on unions) with `Omit` (works on object keys)

```ts
interface User { id: number; name: string; email: string; passwordHash: string; }
type Role = "admin" | "editor" | "viewer";

// ❌ Using Exclude to remove a property from an object type — WRONG:
type SafeUser = Exclude<User, "passwordHash">;
// SafeUser = never  (User is not assignable to "passwordHash")

// ✅ Omit removes properties from object types:
type SafeUser = Omit<User, "passwordHash">;
// { id: number; name: string; email: string }

// ❌ Using Omit to filter a union — WRONG:
type NonAdmin = Omit<Role, "admin">;
// { } — Omit treats Role as an object type, removes "admin" property — nonsensical result

// ✅ Exclude filters union members:
type NonAdmin = Exclude<Role, "admin">;
// "editor" | "viewer"
```

### Mistake 3 — `Readonly<T>` is shallow — nested objects are still mutable

```ts
interface Config {
  server: { port: number; host: string };
  db:     { url: string; poolSize: number };
}

const config: Readonly<Config> = {
  server: { port: 3000, host: "0.0.0.0" },
  db:     { url: "postgres://...", poolSize: 5 },
};

config.server = { port: 8080, host: "localhost" };  // ❌ Error — top level is readonly
config.server.port = 8080;  // ✅ NO error — nested object is still mutable!

// ✅ For deep immutability, use a recursive ReadonlyDeep or as const:
const frozenConfig = {
  server: { port: 3000, host: "0.0.0.0" },
  db:     { url: "postgres://...", poolSize: 5 },
} as const;
// frozenConfig.server.port = 8080; // ❌ Error — as const makes everything deeply readonly
```

---

## Practice exercises

### Exercise 1 — easy

Given this interface:

```ts
interface Product {
  id: number;
  name: string;
  description: string;
  priceCents: number;
  category: "electronics" | "clothing" | "food" | "books";
  inStock: boolean;
  stockCount: number;
  imageUrl: string | null;
  createdAt: Date;
  updatedAt: Date;
}
```

Using only utility types (no new `interface` or manual type definitions), derive:

1. `CreateProductInput` — fields a client sends when creating (no `id`, `inStock`, `stockCount`, `createdAt`, `updatedAt`)
2. `UpdateProductInput` — all fields optional, same exclusions as create
3. `ProductListItem` — only `id`, `name`, `priceCents`, `category`, `inStock` for list views
4. `ImmutableProduct` — readonly version of the full Product
5. `ProductCategory` — just the union of category values
6. `DigitalCategory` — only `"electronics"` | `"books"` from `ProductCategory`
7. `PhysicalCategory` — exclude digital from `ProductCategory`

Verify each derived type by writing a small function that accepts/returns it.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a generic CRUD API type generator using utility types:

```ts
// Given any entity type T, derive all the standard CRUD types:
type CrudTypes<T extends { id: number; createdAt: Date; updatedAt: Date }> = {
  Entity:      T;
  CreateInput: Omit<T, "id" | "createdAt" | "updatedAt">;
  UpdateInput: Partial<Omit<T, "id" | "createdAt" | "updatedAt">>;
  IdParam:     Pick<T, "id">;
  ListItem:    Pick<T, "id"> & Partial<Omit<T, "id" | "createdAt" | "updatedAt">>;
  Response:    ApiResponse<T>;
  ListResponse: ApiResponse<PaginatedData<T>>;
};
```

1. Define `CrudTypes` using the correct utility types.
2. Instantiate it for `User` and `Post` (define minimal versions of each).
3. Write a function `createCrudHandlers<T extends { id: number; createdAt: Date; updatedAt: Date }>(service: CrudService<T, CrudTypes<T>["CreateInput"], CrudTypes<T>["UpdateInput"]>)` that returns an object with handler functions for each CRUD operation. Each handler function should have its parameters and return types fully typed using `CrudTypes<T>`.

```ts
interface ApiResponse<TData> { data: TData; status: number; message: string; }
interface PaginatedData<T>   { items: T[]; total: number; page: number; pageSize: number; hasNext: boolean; }
interface CrudService<T, TCreate, TUpdate> {
  getById(id: number): Promise<T | null>;
  getAll(page: number, pageSize: number): Promise<{ items: T[]; total: number }>;
  create(input: TCreate): Promise<T>;
  update(id: number, input: TUpdate): Promise<T | null>;
  delete(id: number): Promise<boolean>;
}

// Write your implementation here
```

### Exercise 3 — hard

Build a typed middleware system where each middleware function's type is derived from the handler it wraps, using `ReturnType`, `Parameters`, and `Awaited`:

```ts
// A handler is any async function:
type AnyHandler = (...args: unknown[]) => Promise<unknown>;

// A middleware wraps a handler and can:
// 1. Inspect/transform the arguments (same type as handler's params)
// 2. Call the original handler
// 3. Inspect/transform the result (same type as handler's return value)
// 4. Catch errors from the handler

type Middleware<THandler extends AnyHandler> = (
  handler: THandler,
) => (...args: Parameters<THandler>) => ReturnType<THandler>;

// Implement these three middlewares using the Middleware<T> type:

// 1. loggingMiddleware — logs args, result, and timing:
function loggingMiddleware<T extends AnyHandler>(name: string): Middleware<T>

// 2. retryMiddleware — retries on failure up to maxAttempts times:
function retryMiddleware<T extends AnyHandler>(maxAttempts: number, delayMs: number): Middleware<T>

// 3. cacheMiddleware — caches results by a cache key derived from args:
function cacheMiddleware<T extends AnyHandler>(
  getKey: (...args: Parameters<T>) => string,
  ttlMs: number,
): Middleware<T>

// Compose middlewares:
function applyMiddleware<T extends AnyHandler>(
  handler: T,
  ...middlewares: Array<Middleware<T>>
): T
```

Then wrap a real async function with all three:

```ts
async function fetchUserWithPosts(
  userId: number,
  includeUnpublished: boolean,
): Promise<{ user: User; posts: Post[] }> {
  // stub
  return { user: {} as User, posts: [] };
}

const wrappedHandler = applyMiddleware(
  fetchUserWithPosts,
  loggingMiddleware("fetchUserWithPosts"),
  retryMiddleware(3, 500),
  cacheMiddleware((userId, includeUnpublished) => `user:${userId}:${includeUnpublished}`, 60_000),
);

// wrappedHandler must have the SAME signature as fetchUserWithPosts:
const result = await wrappedHandler(1, false);
// result: { user: User; posts: Post[] }  — fully typed
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Utility | What it does | Example |
|---------|-------------|---------|
| `Partial<T>` | All props optional | `Partial<User>` → `{ id?: number; name?: string; ... }` |
| `Required<T>` | All props required | `Required<DraftPost>` → all fields mandatory |
| `Readonly<T>` | All props readonly (shallow) | `Readonly<Config>` → `{ readonly port: number; ... }` |
| `Pick<T, K>` | Keep only K keys | `Pick<User, "id"\|"name">` → `{ id: number; name: string }` |
| `Omit<T, K>` | Remove K keys | `Omit<User, "passwordHash">` → all except password |
| `Record<K, V>` | Map keys to value type | `Record<Role, string[]>` → role → permissions |
| `Exclude<T, U>` | Remove U members from union T | `Exclude<Role, "admin">` → `"editor"\|"viewer"` |
| `Extract<T, U>` | Keep T members assignable to U | `Extract<Role, "admin"\|"editor">` → `"admin"\|"editor"` |
| `NonNullable<T>` | Remove null and undefined | `NonNullable<User\|null>` → `User` |
| `ReturnType<T>` | Return type of function T | `ReturnType<typeof getUser>` → `User` |
| `Parameters<T>` | Tuple of params of function T | `Parameters<typeof fn>` → `[number, string]` |
| `Awaited<T>` | Unwrap Promise (recursive) | `Awaited<Promise<User>>` → `User` |
| `InstanceType<T>` | Instance type of constructor T | `InstanceType<typeof UserService>` → `UserService` |

**Common compositions:**
```ts
// DB create input:
type CreateInput<T> = Omit<T, "id" | "createdAt" | "updatedAt">;

// DB update input (PATCH):
type UpdateInput<T> = Partial<Omit<T, "id" | "createdAt" | "updatedAt">>;

// Async return type:
type AsyncReturn<T extends (...a: unknown[]) => Promise<unknown>> = Awaited<ReturnType<T>>;

// Non-null async return:
type DefiniteReturn<T extends (...a: unknown[]) => Promise<unknown>> = NonNullable<Awaited<ReturnType<T>>>;
```

---

## Connected topics

- **26–31 — Generics** — utility types are all built on generic type parameters; understanding `<T>` is prerequisite.
- **40 — Conditional types** — `T extends U ? X : Y` — `Exclude`, `Extract`, `NonNullable` are all conditional types internally.
- **41 — Mapped types** — `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record` are all mapped types internally.
- **09 — Union types** — `Exclude` and `Extract` operate on unions.
- **10 — Intersection types** — composing utility types often uses `&`.
- **12 — Type aliases** — derived types from utility types are always type aliases.
