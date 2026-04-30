# 10 — Intersection Types

## What is this?

An intersection type combines **multiple types into one** using the `&` operator. The resulting type has **all properties from all combined types** — you must satisfy every type in the intersection simultaneously. Where union (`|`) means "one of these types," intersection (`&`) means "all of these types at once." Think of it as merging multiple shapes into a single merged shape.

## Why does it matter?

In backend development you constantly compose objects — a user object that is both a database record AND has been authenticated, a request that has the base properties AND has been extended with middleware-injected data, a config that merges base settings AND environment-specific overrides. Intersection types let you express these compositions in a reusable, type-safe way without duplicating property definitions.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — you manually merge objects with spread, no type safety
function mergeUserWithSession(user, session) {
  return { ...user, ...session };
  // Result shape? Unknown. TypeScript can't help here.
  // If user has .name and session has .token, the result has both — or maybe one overrides the other
}

const enrichedUser = mergeUserWithSession(
  { id: 1, name: "Parsh" },
  { token: "Bearer xyz", expiresAt: new Date() }
);

enrichedUser.nme;  // typo — JS says undefined, no error
```

```ts
// TypeScript — intersection precisely defines the merged shape
type UserRecord = { id: number; name: string; email: string };
type AuthSession = { token: string; expiresAt: Date; role: "admin" | "user" };

type AuthenticatedUser = UserRecord & AuthSession;
// Result: { id: number; name: string; email: string; token: string; expiresAt: Date; role: "admin" | "user" }

const enrichedUser: AuthenticatedUser = {
  id: 1,
  name: "Parsh",
  email: "parsh@dev.io",
  token: "Bearer xyz",
  expiresAt: new Date(),
  role: "admin",
};

enrichedUser.nme;      // ❌ Property 'nme' does not exist — caught immediately
enrichedUser.name;     // ✅
enrichedUser.token;    // ✅
```

---

## Syntax

```ts
// ── BASIC INTERSECTION ────────────────────────────────────────────────────
type A = { name: string };
type B = { age: number };
type C = A & B;  // { name: string; age: number }

// ── INTERSECTION IN FUNCTION PARAMS ──────────────────────────────────────
function processUser(user: UserRecord & AuthSession): void { ... }

// ── INLINE INTERSECTION ───────────────────────────────────────────────────
function greet(entity: { name: string } & { greeting: string }): string {
  return `${entity.greeting}, ${entity.name}`;
}

// ── THREE-WAY INTERSECTION ────────────────────────────────────────────────
type FullUserProfile = UserRecord & AuthSession & { preferences: UserPreferences };

// ── INTERSECTION WITH PRIMITIVE — produces never ──────────────────────────
type Impossible = string & number;  // type: never — a value can't be both string AND number
```

---

## How it works — line by line

### Properties are merged

```ts
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type SoftDeletable = {
  deletedAt: Date | null;
};

type BaseModel = Timestamped & SoftDeletable;
// Equivalent to writing:
// type BaseModel = {
//   createdAt: Date;
//   updatedAt: Date;
//   deletedAt: Date | null;
// }

// Any object typed as BaseModel must have ALL three properties:
const record: BaseModel = {
  createdAt: new Date(),
  updatedAt: new Date(),
  deletedAt: null,
};
```

### Conflicting property types produce `never`

```ts
// If two intersected types have the same key with different types:
type HasStringId = { id: string };
type HasNumberId = { id: number };

type Conflict = HasStringId & HasNumberId;
// id is: string & number = never
// No value can satisfy both string AND number simultaneously

const broken: Conflict = {
  id: "hello",  // ❌ string is not assignable to type 'never'
};

// Practical takeaway: never intersect types that share a key with incompatible types
```

### Intersection vs spreading

```ts
type A = { x: number };
type B = { y: string };

// Intersection type — both must be present at the same time:
function takeAandB(val: A & B): void {
  console.log(val.x, val.y);  // TypeScript knows both exist
}

// Object spread creates a new object — the types aren't automatically intersected:
const a: A = { x: 1 };
const b: B = { y: "hello" };
const merged = { ...a, ...b };
// TypeScript infers: { x: number; y: string } — effectively A & B, but via inference
```

---

## Intersection vs Union — the key difference

```ts
type Cat = { meow(): void };
type Dog = { bark(): void };

// Union — the value is ONE of the types:
type Pet = Cat | Dog;
function playCat(pet: Pet): void {
  pet.meow();   // ❌ can't call — pet might be a Dog
  pet.bark();   // ❌ can't call — pet might be a Cat
  // Must narrow first
}

// Intersection — the value is ALL of the types simultaneously:
type CatDog = Cat & Dog;
function playCatDog(pet: CatDog): void {
  pet.meow();   // ✅ guaranteed to exist
  pet.bark();   // ✅ guaranteed to exist
}
```

---

## Real patterns where intersections shine

### Pattern 1: Adding metadata to a base type

```ts
// Base DB record shape — every table has these:
type DbRecord = {
  id: number;
  createdAt: Date;
  updatedAt: Date;
};

// Your domain types don't need to repeat timestamp fields:
type User = DbRecord & {
  email: string;
  name: string;
  role: "admin" | "user";
};

type Product = DbRecord & {
  name: string;
  price: number;
  stockCount: number;
};

// Result: User has id, createdAt, updatedAt, email, name, role
// Result: Product has id, createdAt, updatedAt, name, price, stockCount
```

### Pattern 2: Middleware-enriched Express Request

```ts
// Express Request doesn't have req.user by default.
// A common pattern: extend it via intersection in the middleware chain.

import { Request, Response, NextFunction } from 'express';

type AuthenticatedUser = { id: number; email: string; role: "admin" | "user" };

// Type for a request that has been through the auth middleware:
type AuthenticatedRequest = Request & { user: AuthenticatedUser };

// Auth middleware adds req.user:
function authMiddleware(req: Request, res: Response, next: NextFunction): void {
  const token = req.headers.authorization;
  if (!token) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }
  // Validate token — if valid, attach user:
  (req as AuthenticatedRequest).user = { id: 1, email: "parsh@dev.io", role: "admin" };
  next();
}

// Protected route handlers receive the enriched request:
function getProfileHandler(req: AuthenticatedRequest, res: Response): void {
  const { id, email, role } = req.user;   // ✅ TypeScript knows req.user exists
  res.json({ id, email, role });
}
```

### Pattern 3: Configuration merging

```ts
type BaseConfig = {
  port: number;
  host: string;
  logLevel: "debug" | "info" | "warn" | "error";
};

type DatabaseConfig = {
  dbUrl: string;
  dbPoolSize: number;
  dbTimeout: number;
};

type CacheConfig = {
  redisUrl: string;
  cacheTtl: number;
};

// The full app config is all three:
type AppConfig = BaseConfig & DatabaseConfig & CacheConfig;

// You can now pass different subsets to functions that only need part of the config:
function startServer(config: BaseConfig): void {
  console.log(`Starting on ${config.host}:${config.port}`);
}

function connectDatabase(config: DatabaseConfig): void {
  console.log(`Connecting to ${config.dbUrl}`);
}

// And the full config satisfies both:
const fullConfig: AppConfig = {
  port: 3000,
  host: "localhost",
  logLevel: "info",
  dbUrl: "postgres://localhost/mydb",
  dbPoolSize: 10,
  dbTimeout: 5000,
  redisUrl: "redis://localhost:6379",
  cacheTtl: 3600,
};

startServer(fullConfig);      // ✅ AppConfig satisfies BaseConfig
connectDatabase(fullConfig);  // ✅ AppConfig satisfies DatabaseConfig
```

---

## Example 1 — basic

```ts
// Building a layered user type using intersections

type HasId = { id: number };

type HasTimestamps = {
  createdAt: Date;
  updatedAt: Date;
};

type HasSoftDelete = {
  deletedAt: Date | null;
};

type UserProfile = {
  name: string;
  email: string;
  avatarUrl: string | null;
};

type UserPermissions = {
  role: "admin" | "editor" | "viewer";
  canPublish: boolean;
  canDelete: boolean;
};

// Compose the full user from parts:
type FullUser = HasId & HasTimestamps & HasSoftDelete & UserProfile & UserPermissions;

// TypeScript requires ALL fields to be present:
const user: FullUser = {
  id: 1,
  createdAt: new Date("2026-01-01"),
  updatedAt: new Date("2026-04-01"),
  deletedAt: null,
  name: "Parsh",
  email: "parsh@dev.io",
  avatarUrl: null,
  role: "admin",
  canPublish: true,
  canDelete: true,
};

// Each part's properties are accessible:
console.log(user.id);           // ✅ from HasId
console.log(user.createdAt);    // ✅ from HasTimestamps
console.log(user.role);         // ✅ from UserPermissions
console.log(user.name);         // ✅ from UserProfile
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── Typed middleware enrichment chain ─────────────────────────────────────────

// Step 1: base request (from Express)
// Step 2: auth middleware adds req.user
// Step 3: rate limiter middleware adds req.rateLimit
// Step 4: request logger adds req.requestId

type RequestUser = {
  userId: number;
  email: string;
  role: "admin" | "user";
};

type RateLimitInfo = {
  limit: number;
  remaining: number;
  resetAt: Date;
};

// Intersection builds up the enriched request type:
type AuthenticatedRequest = Request & { user: RequestUser };
type RateLimitedRequest = AuthenticatedRequest & { rateLimit: RateLimitInfo };
type TrackedRequest = RateLimitedRequest & { requestId: string };

// Each middleware adds its piece:
function attachRequestId(req: Request, _res: Response, next: NextFunction): void {
  (req as TrackedRequest).requestId = `req-${Date.now()}-${Math.random().toString(36).slice(2)}`;
  next();
}

function authMiddleware(req: Request, res: Response, next: NextFunction): void {
  const token = req.headers.authorization;
  if (!token) {
    res.status(401).json({ error: "Unauthorized" });
    return;
  }
  (req as AuthenticatedRequest).user = { userId: 1, email: "parsh@dev.io", role: "admin" };
  next();
}

// Route handler that requires the fully enriched request:
async function createPostHandler(req: TrackedRequest, res: Response): Promise<void> {
  const { userId, role } = req.user;      // ✅ from auth middleware
  const { remaining } = req.rateLimit;    // ✅ from rate limiter
  const { requestId } = req;              // ✅ from request tracker

  if (role !== "admin" && remaining === 0) {
    res.status(429).json({ error: "Rate limit exceeded", requestId });
    return;
  }

  // ... create post
  res.status(201).json({ requestId, createdBy: userId });
}

// ── Intersection for repository pattern ───────────────────────────────────────

type Identifiable = { id: number };
type Creatable<T> = { create(data: T): Promise<Identifiable & T> };
type Findable<T> = { findById(id: number): Promise<(Identifiable & T) | null> };
type Deletable = { delete(id: number): Promise<void> };

// A full repository implements all operations:
type UserRepository = Creatable<UserProfile> & Findable<UserProfile> & Deletable;

// An implementation:
const userRepo: UserRepository = {
  async create(data: UserProfile) {
    return { id: Date.now(), ...data };
  },
  async findById(id: number) {
    if (id !== 1) return null;
    return { id: 1, name: "Parsh", email: "parsh@dev.io", avatarUrl: null };
  },
  async delete(_id: number) {
    // remove from DB
  },
};
```

---

## Common mistakes

### Mistake 1 — Using intersection when you mean union

```ts
// ❌ WRONG — intersection means ALL types at once, not one of them
function handleInput(value: string & number): void {
  // value must be BOTH string AND number — impossible!
  // This is type 'never' — no value can ever be passed here
}

// ✅ RIGHT — if you mean "string or number", use union
function handleInput(value: string | number): void {
  if (typeof value === "string") return;
  value.toFixed(2);
}
```

### Mistake 2 — Intersecting conflicting property types

```ts
// ❌ WRONG — different types for the same key produces never
type HasStringId = { id: string; name: string };
type HasNumberId = { id: number; age: number };

type Merged = HasStringId & HasNumberId;
// id is: string & number = never
// You cannot create a valid Merged value

const broken: Merged = {
  id: 1,     // ❌ Type 'number' is not assignable to type 'never'
  name: "Parsh",
  age: 25,
};

// ✅ RIGHT — ensure shared keys have compatible types, or use different key names
type UserBase = { userId: string; name: string };
type UserStats = { userId: string; age: number };  // same type for userId

type UserFull = UserBase & UserStats;  // userId: string & string = string ✅
```

### Mistake 3 — Over-using intersection where interface `extends` is cleaner

```ts
// ❌ AWKWARD — intersection of many small types is hard to read
type Entity = HasId & HasTimestamps & HasSoftDelete & { name: string; email: string };

// ✅ BETTER for class-like structures — use interface extends (topic 18)
interface BaseEntity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface UserEntity extends BaseEntity {
  name: string;
  email: string;
}

// Intersections are best for:
//   - Composing independently maintained types
//   - Extending third-party types you don't own (like Request)
//   - Creating utility types programmatically
```

---

## Practice exercises

### Exercise 1 — easy

Define three separate types:
- `HasTimestamps` — `createdAt: Date`, `updatedAt: Date`
- `HasOwner` — `ownerId: number`, `ownerEmail: string`
- `BlogPost` — `title: string`, `content: string`, `isPublished: boolean`

Create an intersection type `FullBlogPost = BlogPost & HasTimestamps & HasOwner`.

Write a function `createBlogPost(title: string, content: string, ownerId: number, ownerEmail: string): FullBlogPost` that fills in all fields (set `isPublished: false`, set both timestamps to `new Date()`).

Call the function and verify you can access every field on the result.

```ts
// Write your code here
```

### Exercise 2 — medium

You're building a permission-checking system. Define:

- `BaseUser` — `id: number`, `email: string`
- `WithRole` — `role: "admin" | "editor" | "viewer"`
- `WithPermissions` — `permissions: string[]` (e.g. `["read", "write", "delete"]`)
- `AuthenticatedUser = BaseUser & WithRole & WithPermissions`

Write two functions:
1. `hasPermission(user: AuthenticatedUser, permission: string): boolean` — returns true if `permission` is in `user.permissions`
2. `requireAdmin(user: AuthenticatedUser): void` — throws an `Error("Forbidden: admin role required")` if `user.role !== "admin"`

Then write a mock `RequestContext` type that is `Request & { currentUser: AuthenticatedUser }` (you can import Request from express or mock it as `type Request = { headers: Record<string, string> }`). Write a route handler function typed as `(ctx: RequestContext, res: { json: (data: unknown) => void }) => void` that calls `requireAdmin` and returns the user's permissions if they are an admin.

```ts
// Write your code here
```

### Exercise 3 — hard

You're implementing a typed repository pattern. Define:

```ts
type Paginated<T> = { items: T[]; total: number; page: number; pageSize: number }
```

Define an intersection-based repository interface for a `UserRecord` type (at minimum: `id: number`, `email: string`, `name: string`, `role: "admin" | "user"`, `createdAt: Date`):

```ts
type ReadRepository<T extends { id: number }> = {
  findById(id: number): Promise<T | null>
  findAll(page: number, pageSize: number): Promise<Paginated<T>>
  findWhere(predicate: (item: T) => boolean): Promise<T[]>
}

type WriteRepository<T extends { id: number }, TCreate> = {
  create(data: TCreate): Promise<T>
  update(id: number, data: Partial<TCreate>): Promise<T | null>
  delete(id: number): Promise<boolean>
}

type UserRepository = ReadRepository<UserRecord> & WriteRepository<UserRecord, Omit<UserRecord, "id" | "createdAt">>
```

Implement `userRepository: UserRepository` as a plain object with an in-memory array store. All methods must be fully typed — no `any`. Implement the `Paginated` logic in `findAll` correctly (slice the array based on `page` and `pageSize`). Test by calling `create`, `findById`, `findAll`, and `delete`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Concept | Syntax | Result |
|---------|--------|--------|
| Basic intersection | `A & B` | Has all properties of A and B |
| Three-way | `A & B & C` | Has all properties of A, B, and C |
| Inline | `{ name: string } & { age: number }` | `{ name: string; age: number }` |
| Conflicting key | `{ id: string } & { id: number }` | `id: never` — avoid! |
| Primitives | `string & number` | `never` — impossible |

| vs Union | |
|----------|--|
| Union `\|` | "One of these types" — must narrow before using |
| Intersection `&` | "All of these types" — all properties guaranteed |

| Common patterns | Use case |
|----------------|---------|
| `DbRecord & DomainType` | Add timestamps/id to any model |
| `Request & { user: User }` | Extend Express request with middleware data |
| `BaseConfig & EnvConfig` | Merge configuration objects |
| `Readable<T> & Writable<T>` | Compose repository operations |

## Connected topics

- **09 — Union types** — `|` is the opposite of `&` — understand both together.
- **14 — Interfaces** — `interface extends` is often cleaner than `&` for object hierarchies.
- **18 — Extending interfaces** — inheritance-style composition vs intersection composition.
- **30 — Generic constraints** — `T extends A & B` to require a type satisfies multiple shapes.
