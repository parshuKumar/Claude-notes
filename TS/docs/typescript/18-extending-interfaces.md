# 18 — Extending Interfaces

## What is this?

The `extends` keyword on an interface creates a **child interface** that inherits all properties and methods from one or more parent interfaces, and can add new ones on top. The child type is a structural superset of the parent — any object that satisfies the child interface automatically satisfies every parent interface too.

```ts
interface Child extends Parent {
  // inherits everything from Parent
  // plus adds new properties here
}
```

TypeScript supports:
- **Single inheritance** — `Child extends Parent`
- **Multiple inheritance** — `Child extends A, B, C`
- **Chaining** — `GrandChild extends Child` (which already `extends Parent`)

## Why does it matter?

Real-world data models share structure. Every database row has an `id` and `createdAt`. Every API response has `success`, `requestId`, and `timestamp`. Every entity in your system has audit fields. Without interface extension you copy these fields into every interface — and when they change you update them in twenty places. With `extends` you define the common base once, extend it everywhere, and change it in one place.

Extension also enforces hierarchy: a `CreateUserBody` doesn't need `id` or `createdAt` (they're assigned by the system), but a `User` returned from the database does. You can model this precisely with interfaces rather than duplicating or wrestling with optional fields everywhere.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no type system, no interface extension
// You just hope the objects you pass around have the right shape

// Common pattern: manually "extend" by copying fields:
function makeAdminUser(baseUser) {
  return {
    ...baseUser,     // copy all base user fields
    role: "admin",
    permissions: [],
    // Did we get all the required fields? No way to know.
  };
}
```

```ts
// TypeScript — interface extension ensures exact structural relationships

interface BaseEntity {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
}

interface User extends BaseEntity {
  email: string;
  name: string;
  role: "admin" | "editor" | "viewer";
}

interface AdminUser extends User {
  role: "admin";          // narrows the union from User — must still be "admin"
  permissions: string[];
  lastAdminActionAt: Date | null;
}

// AdminUser MUST satisfy: id, createdAt, updatedAt (BaseEntity)
//                      + email, name, role (User)
//                      + permissions, lastAdminActionAt (AdminUser)

// A function that accepts User also accepts AdminUser:
function sendWelcomeEmail(user: User): void {
  console.log(`Welcome, ${user.name}! (${user.email})`);
}

const admin: AdminUser = {
  id: 1, createdAt: new Date(), updatedAt: new Date(),
  email: "admin@dev.io", name: "Alice", role: "admin",
  permissions: ["users.read", "users.write"],
  lastAdminActionAt: null,
};

sendWelcomeEmail(admin); // ✅ AdminUser satisfies User — substitution works
```

---

## Syntax

```ts
// ── SINGLE EXTENDS ─────────────────────────────────────────────────────────
interface Child extends Parent {
  newProperty: string;
}

// ── MULTI EXTENDS ──────────────────────────────────────────────────────────
interface Child extends ParentA, ParentB, ParentC {
  newProperty: string;
}

// ── CHAINED INHERITANCE ────────────────────────────────────────────────────
interface GrandChild extends Child {
  // inherits from Child (which inherits from Parent)
  grandChildProperty: number;
}

// ── NARROWING A PARENT PROPERTY ────────────────────────────────────────────
interface BaseResponse {
  status: "success" | "error";
}
interface SuccessResponse extends BaseResponse {
  status: "success";     // narrows "success" | "error" → "success" only
  data: unknown;
}

// ── EXTENDING WITH GENERICS ────────────────────────────────────────────────
interface PaginatedResponse<T> extends BaseResponse {
  data: T[];
  total: number;
  page: number;
}

// ── EXTENDING A TYPE ALIAS ─────────────────────────────────────────────────
type Timestamped = { createdAt: Date; updatedAt: Date };

interface Post extends Timestamped {   // interface can extend a type alias
  id: number;
  title: string;
}
```

---

## How it works — rule by rule

### Structural superset

The child is always a superset of the parent — it has every parent property plus its own additions:

```ts
interface BaseEntity {
  readonly id: number;
  readonly createdAt: Date;
}

interface User extends BaseEntity {
  email: string;
  name: string;
}

// User must have: id, createdAt, email, name
// Anything assignable to BaseEntity accepts a User:
function logEntity(entity: BaseEntity): void {
  console.log(`Entity ${entity.id} created at ${entity.createdAt}`);
}

const user: User = { id: 1, createdAt: new Date(), email: "p@dev.io", name: "Parsh" };
logEntity(user);  // ✅ User satisfies BaseEntity
```

### Property narrowing in child

A child interface can **narrow** a parent property's type — but only to a subtype (a stricter version):

```ts
interface ApiResponse {
  success: boolean;
  requestId: string;
}

// ✅ Narrow boolean → true (literal subtype of boolean):
interface SuccessResponse extends ApiResponse {
  success: true;   // narrowed — this response always has success: true
  data: unknown;
}

// ✅ Narrow a union → one member:
interface BaseUser {
  role: "admin" | "editor" | "viewer";
}
interface AdminUser extends BaseUser {
  role: "admin";   // narrowed — admin users always have role: "admin"
  permissions: string[];
}

// ❌ Widening is NOT allowed — can't make a child property less specific:
interface Child extends BaseUser {
  role: string;    // ❌ 'string' is wider than 'admin' | 'editor' | 'viewer'
  // Error: 'string' is not assignable to type '"admin" | "editor" | "viewer"'
}
```

### Multi-inheritance — extending multiple parents

```ts
interface Timestamped {
  createdAt: Date;
  updatedAt: Date;
}

interface SoftDeletable {
  deletedAt: Date | null;
  isDeleted: boolean;
}

interface Auditable {
  createdBy: number;
  updatedBy: number;
}

// User inherits from all three:
interface User extends Timestamped, SoftDeletable, Auditable {
  id: number;
  email: string;
  name: string;
}

// User must have: createdAt, updatedAt, deletedAt, isDeleted,
//                createdBy, updatedBy, id, email, name
```

**Conflict rule in multi-inheritance:** if two parent interfaces declare the same property name with **incompatible types**, TypeScript raises an error:

```ts
interface A { value: string; }
interface B { value: number; }

interface C extends A, B { }
// ❌ Error: Named property 'value' of types 'A' and 'B' are not identical
//    'string' is not assignable to 'number'

// ✅ Both parents must agree on conflicting property types:
interface A { id: number; value: string; }
interface B { id: number; label: string; } // same 'id' type — OK
interface C extends A, B { }  // ✅ — id is number in both
```

### Extending a type alias

An interface can extend a `type` alias that describes an object shape:

```ts
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

// Interface extending a type alias — valid:
interface Post extends Timestamped {
  id: number;
  title: string;
  content: string;
}

// Type alias extending an interface — also valid, uses & intersection:
type EnrichedPost = Post & {
  authorName: string;    // denormalised for display
  commentCount: number;
};
```

### Generics with extends

```ts
interface Repository<T extends BaseEntity> {
  findById(id: number): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, "id" | "createdAt" | "updatedAt">): Promise<T>;
  update(id: number, data: Partial<T>): Promise<T | null>;
  delete(id: number): Promise<boolean>;
}

// The constraint T extends BaseEntity means T must have id, createdAt, updatedAt
// UserRepository uses T = User (which extends BaseEntity ✅):
interface UserRepository extends Repository<User> {
  findByEmail(email: string): Promise<User | null>;  // User-specific method
}
```

---

## Example 1 — basic

```ts
// Building a full entity hierarchy for a CMS

interface BaseEntity {
  readonly id: number;
  readonly createdAt: Date;
  updatedAt: Date;
}

interface SoftDeletable {
  deletedAt: Date | null;
}

interface Publishable {
  status: "draft" | "published" | "archived";
  publishedAt: Date | null;
}

// Author — just the base, no soft delete, no publish state
interface Author extends BaseEntity {
  email: string;
  name: string;
  bio: string;
  avatarUrl: string | null;
}

// Tag — simple, no soft delete
interface Tag extends BaseEntity {
  name: string;
  slug: string;
  color?: string;
}

// Post — base + soft delete + publishable
interface Post extends BaseEntity, SoftDeletable, Publishable {
  title: string;
  slug: string;
  content: string;
  excerpt: string;
  authorId: number;
  tags: Tag[];
  featuredImageUrl: string | null;
  viewCount: number;
}

// Comment — base + soft delete (no publish state)
interface Comment extends BaseEntity, SoftDeletable {
  postId: number;
  authorId: number;
  content: string;
  parentCommentId: number | null;   // for nested replies
  likeCount: number;
}

// Helper functions work with the bases:
function isPublished(item: Publishable): boolean {
  return item.status === "published" && item.publishedAt !== null;
}

function isDeleted(item: SoftDeletable): boolean {
  return item.deletedAt !== null;
}

function getEntityAge(entity: BaseEntity): number {
  return Date.now() - entity.createdAt.getTime();
}

// All work with Post (which satisfies all three base interfaces):
const post: Post = {
  id: 1, createdAt: new Date(), updatedAt: new Date(),
  deletedAt: null,
  status: "published", publishedAt: new Date(),
  title: "TypeScript Interfaces", slug: "ts-interfaces",
  content: "...", excerpt: "...",
  authorId: 42, tags: [], featuredImageUrl: null, viewCount: 0,
};

isPublished(post);    // ✅
isDeleted(post);      // ✅
getEntityAge(post);   // ✅
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response } from 'express';

// ── Base API response hierarchy ─────────────────────────────────────────────

interface BaseApiResponse {
  success: boolean;
  requestId: string;
  timestamp: Date;
  duration: number;  // ms
}

interface SuccessResponse<T> extends BaseApiResponse {
  success: true;     // narrowed
  data: T;
}

interface ErrorResponse extends BaseApiResponse {
  success: false;    // narrowed
  error: {
    code: string;
    message: string;
    field?: string;   // for validation errors
    details?: Record<string, string>;
  };
}

interface PaginatedSuccessResponse<T> extends SuccessResponse<T[]> {
  pagination: {
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPrevPage: boolean;
  };
}

// ── Repository hierarchy ────────────────────────────────────────────────────

interface ReadRepository<T> {
  findById(id: number): Promise<T | null>;
  findAll(options?: { page: number; pageSize: number }): Promise<T[]>;
  count(): Promise<number>;
}

interface WriteRepository<T, CreateBody, UpdateBody> {
  create(data: CreateBody): Promise<T>;
  update(id: number, data: UpdateBody): Promise<T | null>;
  delete(id: number): Promise<boolean>;
}

// Full CRUD extends both:
interface CrudRepository<T, CreateBody, UpdateBody>
  extends ReadRepository<T>,
          WriteRepository<T, CreateBody, UpdateBody> {}

// User repository adds a domain-specific method:
interface CreateUserBody { email: string; name: string; role?: "admin" | "editor" | "viewer"; }
interface UpdateUserBody { email?: string; name?: string; role?: "admin" | "editor" | "viewer"; }

interface UserRepository extends CrudRepository<User, CreateUserBody, UpdateUserBody> {
  findByEmail(email: string): Promise<User | null>;
  findByRole(role: User["role"]): Promise<User[]>;
}

// ── Implementation ──────────────────────────────────────────────────────────

class InMemoryUserRepository implements UserRepository {
  private store: User[] = [];

  async findById(id: number) { return this.store.find(u => u.id === id) ?? null; }
  async findAll(opts?: { page: number; pageSize: number }) {
    if (!opts) return [...this.store];
    const start = (opts.page - 1) * opts.pageSize;
    return this.store.slice(start, start + opts.pageSize);
  }
  async count() { return this.store.length; }
  async create(data: CreateUserBody): Promise<User> {
    const user: User = {
      id: Date.now(), createdAt: new Date(), updatedAt: new Date(),
      email: data.email, name: data.name, role: data.role ?? "viewer",
    };
    this.store.push(user);
    return user;
  }
  async update(id: number, data: UpdateUserBody) {
    const user = this.store.find(u => u.id === id);
    if (!user) return null;
    Object.assign(user, data, { updatedAt: new Date() });
    return user;
  }
  async delete(id: number) {
    const i = this.store.findIndex(u => u.id === id);
    if (i === -1) return false;
    this.store.splice(i, 1);
    return true;
  }
  async findByEmail(email: string) { return this.store.find(u => u.email === email) ?? null; }
  async findByRole(role: User["role"]) { return this.store.filter(u => u.role === role); }
}

// ── Controller using the response hierarchy ─────────────────────────────────

function sendSuccess<T>(
  res: Response,
  data: T,
  requestId: string,
  startTime: number,
  statusCode = 200,
): void {
  const response: SuccessResponse<T> = {
    success: true,
    data,
    requestId,
    timestamp: new Date(),
    duration: Date.now() - startTime,
  };
  res.status(statusCode).json(response);
}

function sendPaginated<T>(
  res: Response,
  items: T[],
  total: number,
  page: number,
  pageSize: number,
  requestId: string,
  startTime: number,
): void {
  const totalPages = Math.ceil(total / pageSize);
  const response: PaginatedSuccessResponse<T> = {
    success: true,
    data: items,
    requestId,
    timestamp: new Date(),
    duration: Date.now() - startTime,
    pagination: {
      total, page, pageSize, totalPages,
      hasNextPage: page < totalPages,
      hasPrevPage: page > 1,
    },
  };
  res.json(response);
}

function sendError(
  res: Response,
  code: string,
  message: string,
  requestId: string,
  startTime: number,
  statusCode = 400,
  field?: string,
): void {
  const response: ErrorResponse = {
    success: false,
    error: { code, message, field },
    requestId,
    timestamp: new Date(),
    duration: Date.now() - startTime,
  };
  res.status(statusCode).json(response);
}
```

---

## Common mistakes

### Mistake 1 — Trying to widen a property type in the child

```ts
interface Base {
  status: "active" | "inactive";
}

// ❌ Trying to widen the type in a child — not allowed:
interface Child extends Base {
  status: string;    // ❌ 'string' is not assignable to '"active" | "inactive"'
}

// The child can ONLY narrow (make more specific), never widen:
interface ActiveRecord extends Base {
  status: "active";  // ✅ narrowed — child is always active
}
```

### Mistake 2 — Conflicting property types from multiple parents

```ts
interface HasId {
  id: number;
}

interface HasStringId {
  id: string;   // same property name, different type
}

// ❌ Extending both — TypeScript catches the conflict immediately:
interface BadEntity extends HasId, HasStringId {
  name: string;
}
// Error: Named property 'id' of types 'HasId' and 'HasStringId' are not identical

// ✅ Fix: reconcile at the parent level, or pick one:
interface Entity extends HasId {
  name: string;
}
```

### Mistake 3 — Deep inheritance chains when composition is clearer

```ts
// ❌ BAD — 5-level inheritance chain, hard to reason about:
interface Entity extends Base extends Core extends Foundation extends Root {
  // What does this actually have? You have to read five interfaces.
}

// ✅ BETTER — compose with multi-extends at one level:
interface Entity extends Identifiable, Timestamped, SoftDeletable, Auditable {
  // All inherited shapes visible in one line
  name: string;
}

// Rule of thumb:
// - Use single-level multi-extends for "entity = these mixins"
// - Use single-chain for true IS-A hierarchies (AdminUser IS-A User IS-A BaseEntity)
// - Don't chain more than 3 levels deep
```

---

## Practice exercises

### Exercise 1 — easy

Build a type hierarchy for a notification system:

1. `BaseNotification` — `id: number`, `userId: number`, `createdAt: Date`, `readAt: Date | null`
2. `EmailNotification extends BaseNotification` — `type: "email"`, `subject: string`, `bodyHtml: string`, `toAddress: string`
3. `PushNotification extends BaseNotification` — `type: "push"`, `title: string`, `body: string`, `iconUrl?: string`, `actionUrl?: string`
4. `SmsNotification extends BaseNotification` — `type: "sms"`, `message: string`, `toPhoneNumber: string`

Write a function `isUnread(notification: BaseNotification): boolean`.

Write a function `markAsRead(notification: BaseNotification): BaseNotification` that returns a new object with `readAt` set to `new Date()` (don't mutate).

Write a function `formatNotificationSummary(n: EmailNotification | PushNotification | SmsNotification): string` that uses `n.type` to produce a type-appropriate summary line.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a repository hierarchy for a blog:

1. `BaseEntity` — `readonly id: number`, `readonly createdAt: Date`, `updatedAt: Date`
2. `Publishable` — `status: "draft" | "published" | "archived"`, `publishedAt: Date | null`
3. `Post extends BaseEntity, Publishable` — `title`, `slug`, `content`, `authorId`, `tags: string[]`
4. `ReadRepository<T>` interface — `findById`, `findAll(page, pageSize): Promise<T[]>`, `count`
5. `WriteRepository<T, C>` interface — `create(data: C): Promise<T>`, `update(id, data: Partial<T>): Promise<T | null>`, `delete(id): Promise<boolean>`
6. `PostRepository extends ReadRepository<Post>, WriteRepository<Post, CreatePostBody>` — add `findBySlug(slug: string): Promise<Post | null>` and `findPublished(page: number, pageSize: number): Promise<Post[]>`

Implement `InMemoryPostRepository` that fully satisfies `PostRepository`. Internally use `Post[]` as the store.

```ts
// Write your code here
```

### Exercise 3 — hard

Design a typed Express middleware chain using interface inheritance:

1. `BaseRequest` — the raw request shape: `method: string`, `path: string`, `headers: Record<string, string>`, `body: unknown`, `query: Record<string, string>`
2. `AuthenticatedRequest extends BaseRequest` — adds `userId: number`, `authToken: string`, `tokenExpiresAt: Date`
3. `ValidatedRequest<T> extends AuthenticatedRequest` — adds `validatedBody: T` (the parsed, validated body)
4. `RateLimitedRequest extends BaseRequest` — adds `clientIp: string`, `requestsRemaining: number`, `windowResetAt: Date`
5. `FullRequest<T> extends AuthenticatedRequest, RateLimitedRequest` — adds `validatedBody: T`, `requestId: string`, `startTime: number`

Write these middleware-style functions (pure functions, not Express middleware):
- `authenticate(req: BaseRequest): AuthenticatedRequest` — throws `Error("Unauthorized")` if `req.headers["authorization"]` doesn't start with `"Bearer "`; otherwise returns `AuthenticatedRequest` with fake values
- `validate<T>(req: AuthenticatedRequest, schema: (body: unknown) => T): ValidatedRequest<T>` — calls `schema(req.body)`, throws on failure, returns `ValidatedRequest<T>`
- `applyRateLimit(req: BaseRequest, limit: number): RateLimitedRequest` — always returns a `RateLimitedRequest` (use fake values)

Write a complete `processRequest<T>` function that chains all three steps and returns a `FullRequest<T>`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `interface B extends A` | B inherits all of A's properties |
| `interface C extends A, B` | C inherits from both A and B |
| `interface D extends C` | Chained — D inherits from C which inherits from A/B |
| `interface E extends MyType` | Interface extending a type alias (valid) |
| Child narrows parent property | Must be a subtype (more specific), not a supertype |
| Conflicting property types | Error at definition time (unlike `&` which silently creates `never`) |
| Child satisfies all parents | A `User` is assignable anywhere `BaseEntity` is expected |

| Rule | Notes |
|------|-------|
| Only object shapes can be extended | Can't `extends` a union, primitive, or mapped type |
| Parent properties are required in child | Unless they were already optional in the parent |
| `readonly` is inherited | Child can't remove `readonly` from a parent property |
| Optional → required in child | Child CAN make an optional parent property required |
| Required → optional in child | NOT allowed — would violate the parent contract |
| Max depth guideline | Keep chains ≤ 3 levels deep; prefer flat multi-extends |

## Connected topics

- **14 — Interfaces** — the foundation this topic builds on.
- **15 — type vs interface** — `extends` is the interface way; `&` is the type alias way, with different conflict semantics.
- **19 — Implementing interfaces in classes** — `class UserRepo implements UserRepository` — the other half of interface-driven design.
- **26 — Generic constraints** — `<T extends BaseEntity>` uses the same `extends` keyword in a different context.
- **32 — Utility types** — `Omit<T, K>`, `Pick<T, K>`, `Partial<T>` — transform inherited interfaces without re-declaring.
