# 67 — Repository pattern in TypeScript

## What is this?

The **repository pattern** puts a single, typed, database-agnostic interface between your business logic and your persistence layer. Your service code says *"give me the user with this id"*; it never says *"run this Mongoose query against this collection with this projection"*.

In TypeScript the pattern becomes far more valuable than in plain JavaScript, because the interface is not a convention or a comment — it is a **compile-time contract**:

```ts
// The contract. Two type parameters: the entity, and the type of its identifier.
interface Repository<T, ID> {
  findById(id: ID): Promise<T | null>;
  findMany(filter: Filter<T>, options?: QueryOptions<T>): Promise<T[]>;
  create(input: CreateInput<T>): Promise<T>;
  update(id: ID, patch: UpdatePatch<T>): Promise<T | null>;
  delete(id: ID): Promise<boolean>;
  count(filter: Filter<T>): Promise<number>;
}
```

Anything that satisfies `Repository<User, UserId>` can be handed to your `UserService` — a Mongoose implementation in production, a Postgres implementation after a migration, an in-memory array in tests. The service does not change, does not know, and cannot tell the difference.

## Why does it matter?

A backend without a repository layer leaks database concerns everywhere:

- Your `UserService` imports the Mongoose model, so **unit tests need a real MongoDB**.
- Mongoose documents carry `_id`, `__v`, `save()`, `toObject()` — all of that ends up in your route handlers and eventually in your JSON responses.
- Swapping Mongo for Postgres means touching 60 files instead of 1.
- Query logic is duplicated: three different places build "find active users created in the last 30 days" slightly differently, and two of them forget the `deletedAt: null` clause.
- There is no single place to add caching, soft deletes, tenant scoping, or audit logging.

With a typed repository:

- The **domain entity** (`User`) is a plain object you fully control — no `_id`, no ODM machinery.
- The **persistence model** (`UserDocument`) stays inside one file.
- Tests use `new InMemoryUserRepository()` and run in microseconds with no Docker.
- Filters and pagination are typed against the entity's real fields, so `{ emial: "x" }` is a compile error.
- Cross-cutting behaviour (caching, tenancy, metrics) is a decorator that implements the same interface.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: the model IS the data layer, and it's everywhere ──────────────
const User = require("../models/user.model"); // Mongoose model imported into a service

async function getActiveUsers(orgId, page) {
  // Query shape is invented on the spot. Nothing validates the field names.
  const docs = await User.find({
    orgId,
    status: "active",
    delete_at: null,          // typo — the real field is deletedAt. Silently matches nothing.
  })
    .sort({ createdAt: -1 })
    .skip(page * 20)
    .limit(20)
    .lean();

  // Callers now receive raw Mongoose shapes: _id, __v, nested ObjectIds.
  return docs;
}

async function promoteUser(userId, patch) {
  // patch came from req.body. Nothing stops it containing { role: "admin" }
  // or { passwordHash: "..." } or { _id: someOtherId }.
  await User.updateOne({ _id: userId }, { $set: patch });
}

// Testing getActiveUsers requires a live MongoDB, or a mocking library that
// stubs a fluent chain of .find().sort().skip().limit().lean() — brittle either way.
```

Everything above is *plausible* and *wrong*. The typo compiles. The mass-assignment compiles. The `_id` leak compiles. You find out in production.

```ts
// ── TypeScript: one contract, checked at compile time ────────────────────────

// The domain entity — what your business logic actually cares about.
interface User {
  readonly userId:    UserId;      // branded string, not a raw string
  readonly orgId:     OrgId;
  readonly email:     string;
  readonly status:    "active" | "suspended" | "deleted";
  readonly role:      "member" | "admin" | "owner";
  readonly createdAt: Date;
}

// The contract. No Mongoose types appear anywhere in it.
interface UserRepository extends Repository<User, UserId> {
  findByEmail(email: string): Promise<User | null>;
}

// The service depends on the interface, never on a model.
class UserService {
  constructor(private readonly users: UserRepository) {}

  async getActiveUsers(orgId: OrgId, page: number): Promise<Page<User>> {
    return this.users.findPage(
      { orgId, status: "active" },              // ✅ `delete_at` would be a compile error
      { sort: { createdAt: "desc" }, page, pageSize: 20 },
    );
  }

  // The patch type is explicit — role and email are NOT assignable here.
  async renameUser(userId: UserId, displayName: string): Promise<User | null> {
    return this.users.update(userId, { displayName });
  }
}

// Tests: `new UserService(new InMemoryUserRepository([alice, bob]))`. No DB. No mocks.
```

The revelation is not "now I have an interface". It is that **the wrong query can no longer be written**, the wrong field can no longer be updated, and the database no longer exists as far as 95% of your code is concerned.

---

## Syntax

```ts
// ── A branded ID — a string at runtime, a distinct type at compile time ──────
declare const brand: unique symbol;                       // never actually exists
type Brand<T, B extends string> = T & { readonly [brand]: B };
type UserId  = Brand<string, "UserId">;                   // not interchangeable with OrgId
type OrgId   = Brand<string, "OrgId">;

// ── Constructor / parser for the branded type ───────────────────────────────
const toUserId = (raw: string): UserId => raw as UserId;  // the ONLY cast, in one place

// ── The generic contract ────────────────────────────────────────────────────
interface Repository<T, ID> {
  findById(id: ID): Promise<T | null>;                    // null, never undefined
  create(input: Omit<T, "userId">): Promise<T>;           // id assigned by the repo
  update(id: ID, patch: Partial<T>): Promise<T | null>;   // ⚠ blind Partial — see below
  delete(id: ID): Promise<boolean>;                       // true if a row was removed
}

// ── Implementing it ─────────────────────────────────────────────────────────
class InMemoryUserRepository implements Repository<User, UserId> {
  //                                    ^ compile error if any method is missing/wrong
  private readonly rows = new Map<UserId, User>();        // keyed by branded id

  async findById(id: UserId): Promise<User | null> {
    return this.rows.get(id) ?? null;                     // Map.get returns undefined
  }
  // ... rest of the interface
}

// ── Consuming it — depend on the interface, not the class ───────────────────
function makeUserService(users: Repository<User, UserId>) { /* ... */ }
```

---

## How it works — concept by concept

### Concept 1 — The entity is not the document

This is the single most important split in the pattern, and the one most tutorials skip.

A **domain entity** is a plain, serialisable, framework-free object. A **persistence model** is whatever your database driver needs. They are usually *similar* — which is exactly why people conflate them, and why the conflation hurts later.

```ts
// ── domain/user.entity.ts — zero imports from mongoose/pg/prisma ─────────────
export interface User {
  readonly userId:      UserId;
  readonly orgId:       OrgId;
  readonly email:       string;
  readonly displayName: string;
  readonly status:      "active" | "suspended" | "deleted";
  readonly role:        "member" | "admin" | "owner";
  readonly createdAt:   Date;
  readonly updatedAt:   Date;
}

// ── infra/user.model.ts — the database's view of the same thing ─────────────
import { Schema, model, type Types, type HydratedDocument } from "mongoose";

export interface UserDocProps {
  _id:          Types.ObjectId;     // Mongo's id, not our UserId
  org:          Types.ObjectId;     // a REFERENCE, named differently
  emailLower:   string;             // denormalised for a case-insensitive index
  displayName:  string;
  status:       "active" | "suspended" | "deleted";
  role:         "member" | "admin" | "owner";
  passwordHash: string;             // ⚠ exists in the DB, must NEVER reach the domain
  deletedAt:    Date | null;        // soft-delete marker, an infra concern
  createdAt:    Date;
  updatedAt:    Date;
}

export type UserDocument = HydratedDocument<UserDocProps>;

// ── The mapper — the ONLY place that knows both shapes ──────────────────────
export function toDomain(doc: UserDocProps): User {
  return {
    userId:      toUserId(doc._id.toHexString()),   // ObjectId → branded string
    orgId:       toOrgId(doc.org.toHexString()),
    email:       doc.emailLower,
    displayName: doc.displayName,
    status:      doc.status,
    role:        doc.role,
    createdAt:   doc.createdAt,
    updatedAt:   doc.updatedAt,
    // passwordHash is simply not copied — it cannot leak, structurally.
  };
}
```

Note what the mapper bought you: `passwordHash` can never be accidentally `res.json()`-ed, because it does not exist on the type the rest of the app sees. That is a **structural** guarantee, not a discipline-based one.

### Concept 2 — Typed filters instead of `Record<string, unknown>`

A filter typed as `object` or `any` is a filter that accepts typos. Build the filter type *from* the entity.

```ts
// The set of operators a single field supports, parameterised by that field's type.
type FieldFilter<V> =
  | V                                    // shorthand equality: { status: "active" }
  | { $eq?: V; $ne?: V }                 // explicit equality
  | { $in?: readonly V[]; $nin?: readonly V[] }
  | (V extends number | Date             // comparisons only for ordered types
      ? { $gt?: V; $gte?: V; $lt?: V; $lte?: V }
      : never);

// A filter over an entity: every key optional, each value a FieldFilter of that key's type.
type Filter<T> = {
  [K in keyof T]?: FieldFilter<T[K]>;
};

// Usage — fully checked:
const f1: Filter<User> = { status: "active" };                    // ✅
const f2: Filter<User> = { status: { $in: ["active", "suspended"] } }; // ✅
const f3: Filter<User> = { createdAt: { $gte: new Date("2026-01-01") } }; // ✅
// const bad1: Filter<User> = { statuss: "active" };               // ❌ unknown key
// const bad2: Filter<User> = { status: "activ" };                 // ❌ not in the union
// const bad3: Filter<User> = { email: { $gt: "a" } };             // ❌ $gt not allowed on string
```

`keyof T` drives everything, so renaming `status` → `accountStatus` on the entity produces compile errors at every call site rather than silently-empty result sets.

### Concept 3 — Typed sorting and pagination

```ts
// Sort keys must be real entity keys; direction is a literal union.
type SortDirection = "asc" | "desc";
type Sort<T> = { readonly [K in keyof T]?: SortDirection };

// Two pagination styles — offset (simple) and cursor (correct at scale).
interface OffsetPage {
  readonly page:     number;   // 0-based
  readonly pageSize: number;
}
interface CursorPage<T> {
  readonly after?:   string;   // opaque, encodes the sort key of the last row
  readonly limit:    number;
  readonly sortKey:  keyof T & string;
}

interface QueryOptions<T> {
  readonly sort?:    Sort<T>;
  readonly limit?:   number;
  readonly skip?:    number;
  readonly select?:  readonly (keyof T)[];   // projection, typed
}

// The paged result — generic over the entity so callers keep full typing.
interface Page<T> {
  readonly items:      readonly T[];
  readonly total:      number;
  readonly page:       number;
  readonly pageSize:   number;
  readonly hasMore:    boolean;
}
```

A subtle but valuable trick: when `select` is provided, the return type should narrow to only those keys. That is expressible:

```ts
// Overload: with a `select`, you get Pick<T, K>; without, you get T.
interface Readable<T, ID> {
  findById(id: ID): Promise<T | null>;
  findById<K extends keyof T>(id: ID, select: readonly K[]): Promise<Pick<T, K> | null>;
}

declare const users: Readable<User, UserId>;
const full    = await users.findById(toUserId("u_1"));                 // User | null
const partial = await users.findById(toUserId("u_1"), ["userId", "email"]);
// partial: Pick<User, "userId" | "email"> | null — accessing .role is a compile error
```

### Concept 4 — `Partial<T>` for updates, and why blind `Partial` is dangerous

`update(id, patch: Partial<User>)` reads beautifully and is a security hole.

```ts
// ❌ The naive contract:
interface NaiveRepo {
  update(id: UserId, patch: Partial<User>): Promise<User | null>;
}

// Three separate problems:

// 1) Mass assignment. If `patch` is ever derived from request input, this compiles:
await repo.update(currentUserId, requestBody as Partial<User>);
//    requestBody = { "role": "owner" }  → privilege escalation, zero type errors.

// 2) Identity/audit fields are writable. `Partial<User>` includes userId, orgId, createdAt.
await repo.update(userId, { userId: someOtherUserId }); // ✅ compiles. Corrupts the row.

// 3) `Partial` erases the difference between "leave alone" and "set to null".
//    Partial<{ deletedAt: Date | null }> gives `deletedAt?: Date | null`, so
//    `undefined` (skip) and `null` (clear it) both type-check and mean different things
//    to different drivers. Mongoose $set: { deletedAt: undefined } is a no-op;
//    a naive SQL builder writes NULL. Same code, two behaviours.
```

The fix is to define the mutable surface explicitly, per entity.

```ts
// ── Step 1: name the immutable fields once ──────────────────────────────────
type ImmutableUserFields = "userId" | "orgId" | "createdAt" | "updatedAt";

// ── Step 2: derive the patch type from the entity, minus the immutables ─────
//    `-readonly` strips the readonly modifiers so the patch object is assignable.
type UpdatePatch<T, Immutable extends keyof T> = {
  -readonly [K in Exclude<keyof T, Immutable>]?: T[K];
};

type UserPatch = UpdatePatch<User, ImmutableUserFields>;
// = { email?: string; displayName?: string; status?: ...; role?: ... }

// ── Step 3: narrow further per use case ─────────────────────────────────────
//    What an end user may change about themselves:
type SelfEditablePatch = Pick<UserPatch, "displayName" | "email">;
//    What an admin may change:
type AdminEditablePatch = Pick<UserPatch, "displayName" | "email" | "status" | "role">;

// ── Step 4: the repository takes the narrow type ────────────────────────────
interface UserRepository {
  update(id: UserId, patch: UserPatch): Promise<User | null>;
}

class UserService {
  constructor(private readonly users: UserRepository) {}

  // The route handler can only ever reach this — the wide patch is unreachable.
  async updateOwnProfile(userId: UserId, patch: SelfEditablePatch): Promise<User | null> {
    return this.users.update(userId, patch);   // ✅ SelfEditablePatch ⊆ UserPatch
  }
}

// const escalate: SelfEditablePatch = { role: "owner" }; // ❌ Object literal may only
//                                                        //    specify known properties
```

For the explicit-null problem, model it explicitly instead of relying on `undefined`:

```ts
// A three-state field update: absent = leave, { set: v } = write, { clear: true } = null.
type FieldUpdate<V> = { set: V } | { clear: true };
type ExplicitPatch<T> = { [K in keyof T]?: FieldUpdate<NonNullable<T[K]>> };

// Now "don't touch" and "set to null" are different values, not different shades of undefined.
const p1: ExplicitPatch<User> = { displayName: { set: "Ada" } };
const p2: ExplicitPatch<{ deletedAt: Date | null }> = { deletedAt: { clear: true } };
```

Use `ExplicitPatch` only where nullability genuinely matters (soft deletes, optional foreign keys). Elsewhere the narrowed `UpdatePatch` is enough and far less noisy.

### Concept 5 — Branded IDs stop the classic argument swap

Every backend has a function like `transfer(fromAccountId, toAccountId)` or `addMember(orgId, userId)`. With `string` ids, swapping the arguments compiles and ships.

```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type UserId    = Brand<string, "UserId">;
export type OrgId     = Brand<string, "OrgId">;
export type SessionId = Brand<string, "SessionId">;

// Parsers: validate once at the boundary, cast once, trust everywhere after.
export function parseUserId(raw: string): UserId {
  if (!/^usr_[0-9a-f]{24}$/.test(raw)) throw new Error(`Malformed userId: ${raw}`);
  return raw as UserId;                       // the single sanctioned assertion
}
export function parseOrgId(raw: string): OrgId {
  if (!/^org_[0-9a-f]{24}$/.test(raw)) throw new Error(`Malformed orgId: ${raw}`);
  return raw as OrgId;
}

declare function addMember(orgId: OrgId, userId: UserId): Promise<void>;

const userId = parseUserId("usr_5f1a2b3c4d5e6f7a8b9c0d1e");
const orgId  = parseOrgId("org_1111222233334444aaaabbbb");

await addMember(orgId, userId);   // ✅
// await addMember(userId, orgId); // ❌ Argument of type 'UserId' is not assignable to 'OrgId'

// Branded ids are erased at runtime — this is still just a string:
console.log(typeof userId);       // "string"
await redis.get(`session:${userId}`); // template literals work unchanged
```

Branded ids compose with the repository generic parameter: `Repository<User, UserId>` and `Repository<Org, OrgId>` are incompatible types, so you cannot pass the wrong repository to the wrong service.

### Concept 6 — A generic base class to kill the boilerplate

Seven entities × six CRUD methods = 42 near-identical methods. Push them into one generic base.

```ts
import type { Model, FilterQuery, SortOrder } from "mongoose";

// The three things every concrete repo must supply.
interface MongoRepositoryConfig<TEntity, TDoc> {
  readonly model:     Model<TDoc>;
  readonly toDomain:  (doc: TDoc) => TEntity;
  readonly toPersist: (patch: Record<string, unknown>) => Record<string, unknown>;
}

abstract class MongoRepository<TEntity, TDoc, ID extends string>
  implements Repository<TEntity, ID>
{
  protected constructor(protected readonly cfg: MongoRepositoryConfig<TEntity, TDoc>) {}

  // Subclasses translate a domain Filter<TEntity> into a driver FilterQuery<TDoc>.
  protected abstract toQuery(filter: Filter<TEntity>): FilterQuery<TDoc>;

  async findById(id: ID): Promise<TEntity | null> {
    const doc = await this.cfg.model.findById(id).lean<TDoc>().exec();
    return doc ? this.cfg.toDomain(doc) : null;   // null, never undefined
  }

  async findMany(filter: Filter<TEntity>, options: QueryOptions<TEntity> = {}): Promise<TEntity[]> {
    const sort: Record<string, SortOrder> = {};
    for (const [key, dir] of Object.entries(options.sort ?? {})) {
      sort[key] = dir === "desc" ? -1 : 1;
    }
    const docs = await this.cfg.model
      .find(this.toQuery(filter))
      .sort(sort)
      .skip(options.skip ?? 0)
      .limit(options.limit ?? 100)              // ALWAYS bound the limit
      .lean<TDoc[]>()
      .exec();
    return docs.map(this.cfg.toDomain);
  }

  async count(filter: Filter<TEntity>): Promise<number> {
    return this.cfg.model.countDocuments(this.toQuery(filter)).exec();
  }

  async delete(id: ID): Promise<boolean> {
    const res = await this.cfg.model.deleteOne({ _id: id }).exec();
    return res.deletedCount === 1;
  }
}
```

The base class handles the mechanical 80%; each concrete repository supplies the mapper, the query translator, and its own domain-specific finders.

### Concept 7 — Decorating a repository with cross-cutting behaviour

Because the contract is an interface, you can wrap any implementation in another implementation. This is where the pattern really earns its keep.

```ts
// A caching layer that is itself a UserRepository — callers cannot tell.
class CachedUserRepository implements UserRepository {
  constructor(
    private readonly inner: UserRepository,        // the real one
    private readonly cache: CacheClient,
    private readonly ttlSeconds = 60,
  ) {}

  async findById(userId: UserId): Promise<User | null> {
    const key    = `user:${userId}`;
    const cached = await this.cache.get<User>(key);
    if (cached) return cached;

    const fresh = await this.inner.findById(userId);
    if (fresh) await this.cache.set(key, fresh, this.ttlSeconds);
    return fresh;
  }

  async update(userId: UserId, patch: UserPatch): Promise<User | null> {
    const updated = await this.inner.update(userId, patch);
    await this.cache.del(`user:${userId}`);        // invalidate on write
    return updated;
  }

  // Everything else delegates unchanged.
  findByEmail(email: string): Promise<User | null> { return this.inner.findByEmail(email); }
  findMany(f: Filter<User>, o?: QueryOptions<User>): Promise<User[]> { return this.inner.findMany(f, o); }
  create(input: CreateUserInput): Promise<User> { return this.inner.create(input); }
  delete(userId: UserId): Promise<boolean> { return this.inner.delete(userId); }
  count(f: Filter<User>): Promise<number> { return this.inner.count(f); }
}

// Composition — the service still receives "a UserRepository":
const repo: UserRepository = new CachedUserRepository(
  new MongoUserRepository(UserModel),
  redisCache,
);
```

The same shape gives you tenant scoping (`TenantScopedUserRepository` that injects `orgId` into every filter), metrics, and audit logging — none of which the service ever learns about.

---

## Example 1 — basic

```ts
// A complete, minimal, compile-valid repository: interface, entity, in-memory impl.

// ── Branded id ──────────────────────────────────────────────────────────────
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type UserId = Brand<string, "UserId">;
export const parseUserId = (raw: string): UserId => {
  if (raw.length === 0) throw new Error("userId must not be empty");
  return raw as UserId;
};

// ── Entity ──────────────────────────────────────────────────────────────────
export interface User {
  readonly userId:      UserId;
  readonly email:       string;
  readonly displayName: string;
  readonly status:      "active" | "suspended";
  readonly createdAt:   Date;
}

// ── Derived input/patch types — never hand-written, always derived ──────────
export type CreateUserInput = Omit<User, "userId" | "createdAt">;
export type UserPatch       = { -readonly [K in "email" | "displayName" | "status"]?: User[K] };

// ── Query types ─────────────────────────────────────────────────────────────
export type Filter<T> = { [K in keyof T]?: T[K] | { $in: readonly T[K][] } };
export type Sort<T>   = { readonly [K in keyof T]?: "asc" | "desc" };

export interface QueryOptions<T> {
  readonly sort?:  Sort<T>;
  readonly skip?:  number;
  readonly limit?: number;
}

// ── The contract ────────────────────────────────────────────────────────────
export interface UserRepository {
  findById(userId: UserId): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findMany(filter: Filter<User>, options?: QueryOptions<User>): Promise<User[]>;
  create(input: CreateUserInput): Promise<User>;
  update(userId: UserId, patch: UserPatch): Promise<User | null>;
  delete(userId: UserId): Promise<boolean>;
  count(filter: Filter<User>): Promise<number>;
}

// ── In-memory implementation — the test double, and a good spec of behaviour ─
export class InMemoryUserRepository implements UserRepository {
  private readonly rows = new Map<UserId, User>();
  private nextId = 1;

  constructor(seed: readonly User[] = []) {
    for (const user of seed) this.rows.set(user.userId, user);
  }

  async findById(userId: UserId): Promise<User | null> {
    return this.rows.get(userId) ?? null;                 // normalise undefined → null
  }

  async findByEmail(email: string): Promise<User | null> {
    const needle = email.toLowerCase();
    for (const user of this.rows.values()) {
      if (user.email.toLowerCase() === needle) return user;
    }
    return null;
  }

  async findMany(filter: Filter<User>, options: QueryOptions<User> = {}): Promise<User[]> {
    let result = [...this.rows.values()].filter((user) => matches(user, filter));

    // Apply a single-key sort (enough for an in-memory fake).
    const sortEntries = Object.entries(options.sort ?? {}) as [keyof User, "asc" | "desc"][];
    for (const [key, dir] of sortEntries.reverse()) {
      result.sort((a, b) => compare(a[key], b[key]) * (dir === "desc" ? -1 : 1));
    }

    const skip = options.skip ?? 0;
    result = result.slice(skip, skip + (options.limit ?? result.length));
    return result;
  }

  async create(input: CreateUserInput): Promise<User> {
    const user: User = {
      ...input,
      userId:    parseUserId(`usr_${this.nextId++}`),
      createdAt: new Date(),
    };
    this.rows.set(user.userId, user);
    return user;
  }

  async update(userId: UserId, patch: UserPatch): Promise<User | null> {
    const existing = this.rows.get(userId);
    if (!existing) return null;

    // Drop undefined keys so "absent" never overwrites a real value.
    const defined = Object.fromEntries(
      Object.entries(patch).filter(([, v]) => v !== undefined),
    ) as UserPatch;

    const updated: User = { ...existing, ...defined };
    this.rows.set(userId, updated);
    return updated;
  }

  async delete(userId: UserId): Promise<boolean> {
    return this.rows.delete(userId);
  }

  async count(filter: Filter<User>): Promise<number> {
    return [...this.rows.values()].filter((user) => matches(user, filter)).length;
  }
}

// ── Tiny helpers used by the fake ───────────────────────────────────────────
function matches(user: User, filter: Filter<User>): boolean {
  return (Object.entries(filter) as [keyof User, unknown][]).every(([key, expected]) => {
    const actual = user[key];
    if (expected !== null && typeof expected === "object" && "$in" in expected) {
      return (expected as { $in: readonly unknown[] }).$in.includes(actual);
    }
    return actual === expected;
  });
}

function compare(a: unknown, b: unknown): number {
  if (a instanceof Date && b instanceof Date) return a.getTime() - b.getTime();
  if (typeof a === "number" && typeof b === "number") return a - b;
  return String(a).localeCompare(String(b));
}

// ── Using it ────────────────────────────────────────────────────────────────
const users: UserRepository = new InMemoryUserRepository();

const created = await users.create({
  email:       "ada@example.com",
  displayName: "Ada Lovelace",
  status:      "active",
});

await users.update(created.userId, { displayName: "Ada L." });      // ✅
// await users.update(created.userId, { userId: created.userId });  // ❌ not in UserPatch
// await users.findMany({ statuss: "active" });                     // ❌ unknown key
```

---

## Example 2 — real world backend use case

```ts
// Production wiring: domain entity, Mongoose document, mapper, concrete repository,
// an in-memory fake for tests, and a service that never mentions Mongo.

import { Schema, model, Types, type Model, type FilterQuery, type SortOrder } from "mongoose";

// ════════════════════════════════════════════════════════════════════════════
// 1. Domain layer — no framework imports allowed in this file
// ════════════════════════════════════════════════════════════════════════════

declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type OrderId = Brand<string, "OrderId">;
export type UserId  = Brand<string, "UserId">;

export const parseOrderId = (raw: string): OrderId => {
  if (!Types.ObjectId.isValid(raw)) throw new Error(`Malformed orderId: ${raw}`);
  return raw as OrderId;
};
export const parseUserId = (raw: string): UserId => {
  if (!Types.ObjectId.isValid(raw)) throw new Error(`Malformed userId: ${raw}`);
  return raw as UserId;
};

export interface OrderLine {
  readonly sku:        string;
  readonly quantity:   number;
  readonly priceCents: number;
}

export interface Order {
  readonly orderId:     OrderId;
  readonly userId:      UserId;
  readonly lines:       readonly OrderLine[];
  readonly totalCents:  number;
  readonly status:      "pending" | "paid" | "shipped" | "cancelled";
  readonly currency:    "USD" | "EUR" | "INR";
  readonly placedAt:    Date;
  readonly updatedAt:   Date;
}

// Inputs and patches are DERIVED, so entity changes propagate automatically.
export type CreateOrderInput = Omit<Order, "orderId" | "placedAt" | "updatedAt" | "totalCents">;
export type OrderPatch       = { -readonly [K in "status" | "lines"]?: Order[K] };

export type Filter<T> = {
  [K in keyof T]?:
    | T[K]
    | { $in?: readonly T[K][]; $ne?: T[K] }
    | (T[K] extends number | Date ? { $gte?: T[K]; $lte?: T[K] } : never);
};
export type Sort<T> = { readonly [K in keyof T]?: "asc" | "desc" };

export interface QueryOptions<T> {
  readonly sort?:  Sort<T>;
  readonly skip?:  number;
  readonly limit?: number;
}

export interface Page<T> {
  readonly items:    readonly T[];
  readonly total:    number;
  readonly page:     number;
  readonly pageSize: number;
  readonly hasMore:  boolean;
}

// ════════════════════════════════════════════════════════════════════════════
// 2. The repository contract — the seam between domain and infrastructure
// ════════════════════════════════════════════════════════════════════════════

export interface OrderRepository {
  findById(orderId: OrderId): Promise<Order | null>;
  findMany(filter: Filter<Order>, options?: QueryOptions<Order>): Promise<Order[]>;
  findPage(filter: Filter<Order>, page: number, pageSize: number): Promise<Page<Order>>;
  create(input: CreateOrderInput): Promise<Order>;
  update(orderId: OrderId, patch: OrderPatch): Promise<Order | null>;
  delete(orderId: OrderId): Promise<boolean>;
  count(filter: Filter<Order>): Promise<number>;

  // Domain-specific finders live on the concrete interface, not the generic base.
  findPendingOlderThan(cutoff: Date, limit: number): Promise<Order[]>;
  sumRevenueCents(userId: UserId): Promise<number>;
}

// ════════════════════════════════════════════════════════════════════════════
// 3. Persistence layer — Mongoose lives here and nowhere else
// ════════════════════════════════════════════════════════════════════════════

interface OrderDoc {
  _id:        Types.ObjectId;
  user:       Types.ObjectId;                  // note the different field name
  lines:      { sku: string; qty: number; price_cents: number }[];  // snake_case in DB
  totalCents: number;
  status:     Order["status"];
  currency:   Order["currency"];
  deletedAt:  Date | null;                     // soft delete — infra concern only
  createdAt:  Date;
  updatedAt:  Date;
}

const orderSchema = new Schema<OrderDoc>(
  {
    user:  { type: Schema.Types.ObjectId, ref: "User", required: true, index: true },
    lines: [
      {
        sku:         { type: String, required: true },
        qty:         { type: Number, required: true, min: 1 },
        price_cents: { type: Number, required: true, min: 0 },
      },
    ],
    totalCents: { type: Number, required: true, min: 0 },
    status:     { type: String, enum: ["pending", "paid", "shipped", "cancelled"], required: true },
    currency:   { type: String, enum: ["USD", "EUR", "INR"], required: true },
    deletedAt:  { type: Date, default: null },
  },
  { timestamps: true },
);

export const OrderModel: Model<OrderDoc> = model<OrderDoc>("Order", orderSchema);

// ── Mappers — the translation boundary ──────────────────────────────────────
function toDomain(doc: OrderDoc): Order {
  return {
    orderId:    parseOrderId(doc._id.toHexString()),
    userId:     parseUserId(doc.user.toHexString()),
    lines:      doc.lines.map((l) => ({ sku: l.sku, quantity: l.qty, priceCents: l.price_cents })),
    totalCents: doc.totalCents,
    status:     doc.status,
    currency:   doc.currency,
    placedAt:   doc.createdAt,
    updatedAt:  doc.updatedAt,
    // deletedAt is deliberately not exposed to the domain.
  };
}

// Domain filter → driver query. Field renames are handled once, here.
function toQuery(filter: Filter<Order>): FilterQuery<OrderDoc> {
  const query: FilterQuery<OrderDoc> = { deletedAt: null };  // soft delete always applied

  for (const [key, value] of Object.entries(filter)) {
    switch (key) {
      case "orderId": query._id  = value as unknown as Types.ObjectId; break;
      case "userId":  query.user = value as unknown as Types.ObjectId; break;
      case "placedAt": {
        // domain `placedAt` maps to persistence `createdAt`
        query.createdAt = value as OrderDoc["createdAt"];
        break;
      }
      default:
        (query as Record<string, unknown>)[key] = value;
    }
  }
  return query;
}

function toSort(sort: Sort<Order> | undefined): Record<string, SortOrder> {
  const out: Record<string, SortOrder> = {};
  for (const [key, dir] of Object.entries(sort ?? {})) {
    const field = key === "placedAt" ? "createdAt" : key;
    out[field] = dir === "desc" ? -1 : 1;
  }
  return out;
}

// ════════════════════════════════════════════════════════════════════════════
// 4. Concrete Mongoose repository
// ════════════════════════════════════════════════════════════════════════════

export class MongoOrderRepository implements OrderRepository {
  private static readonly MAX_LIMIT = 200;     // never let a caller ask for everything

  constructor(private readonly orders: Model<OrderDoc> = OrderModel) {}

  async findById(orderId: OrderId): Promise<Order | null> {
    const doc = await this.orders.findOne({ _id: orderId, deletedAt: null }).lean<OrderDoc>().exec();
    return doc ? toDomain(doc) : null;
  }

  async findMany(filter: Filter<Order>, options: QueryOptions<Order> = {}): Promise<Order[]> {
    const limit = Math.min(options.limit ?? 50, MongoOrderRepository.MAX_LIMIT);
    const docs  = await this.orders
      .find(toQuery(filter))
      .sort(toSort(options.sort))
      .skip(options.skip ?? 0)
      .limit(limit)
      .lean<OrderDoc[]>()
      .exec();
    return docs.map(toDomain);
  }

  async findPage(filter: Filter<Order>, page: number, pageSize: number): Promise<Page<Order>> {
    const safeSize = Math.min(Math.max(pageSize, 1), MongoOrderRepository.MAX_LIMIT);
    const query    = toQuery(filter);

    // One round trip for the page, one for the count — run them concurrently.
    const [docs, total] = await Promise.all([
      this.orders.find(query).sort({ createdAt: -1 }).skip(page * safeSize).limit(safeSize)
        .lean<OrderDoc[]>().exec(),
      this.orders.countDocuments(query).exec(),
    ]);

    const items = docs.map(toDomain);
    return {
      items,
      total,
      page,
      pageSize: safeSize,
      hasMore:  page * safeSize + items.length < total,
    };
  }

  async create(input: CreateOrderInput): Promise<Order> {
    const totalCents = input.lines.reduce((sum, l) => sum + l.priceCents * l.quantity, 0);
    const doc = await this.orders.create({
      user:  new Types.ObjectId(input.userId),
      lines: input.lines.map((l) => ({ sku: l.sku, qty: l.quantity, price_cents: l.priceCents })),
      totalCents,
      status:    input.status,
      currency:  input.currency,
      deletedAt: null,
    });
    return toDomain(doc.toObject<OrderDoc>());
  }

  async update(orderId: OrderId, patch: OrderPatch): Promise<Order | null> {
    // Build $set explicitly. Nothing outside OrderPatch can reach the database.
    const $set: Record<string, unknown> = {};
    if (patch.status !== undefined) $set.status = patch.status;
    if (patch.lines  !== undefined) {
      $set.lines      = patch.lines.map((l) => ({ sku: l.sku, qty: l.quantity, price_cents: l.priceCents }));
      $set.totalCents = patch.lines.reduce((sum, l) => sum + l.priceCents * l.quantity, 0);
    }
    if (Object.keys($set).length === 0) return this.findById(orderId);   // no-op patch

    const doc = await this.orders
      .findOneAndUpdate({ _id: orderId, deletedAt: null }, { $set }, { new: true })
      .lean<OrderDoc>()
      .exec();
    return doc ? toDomain(doc) : null;
  }

  async delete(orderId: OrderId): Promise<boolean> {
    // Soft delete — the caller doesn't know or care.
    const res = await this.orders
      .updateOne({ _id: orderId, deletedAt: null }, { $set: { deletedAt: new Date() } })
      .exec();
    return res.modifiedCount === 1;
  }

  async count(filter: Filter<Order>): Promise<number> {
    return this.orders.countDocuments(toQuery(filter)).exec();
  }

  async findPendingOlderThan(cutoff: Date, limit: number): Promise<Order[]> {
    const docs = await this.orders
      .find({ status: "pending", createdAt: { $lt: cutoff }, deletedAt: null })
      .sort({ createdAt: 1 })
      .limit(Math.min(limit, MongoOrderRepository.MAX_LIMIT))
      .lean<OrderDoc[]>()
      .exec();
    return docs.map(toDomain);
  }

  async sumRevenueCents(userId: UserId): Promise<number> {
    const [row] = await this.orders.aggregate<{ total: number }>([
      { $match: { user: new Types.ObjectId(userId), status: { $in: ["paid", "shipped"] }, deletedAt: null } },
      { $group: { _id: null, total: { $sum: "$totalCents" } } },
    ]);
    return row?.total ?? 0;
  }
}

// ════════════════════════════════════════════════════════════════════════════
// 5. In-memory fake — same contract, no database
// ════════════════════════════════════════════════════════════════════════════

export class InMemoryOrderRepository implements OrderRepository {
  private readonly rows = new Map<OrderId, Order>();
  private seq = 0;

  constructor(seed: readonly Order[] = []) {
    for (const order of seed) this.rows.set(order.orderId, order);
  }

  private all(): Order[] { return [...this.rows.values()]; }

  private select(filter: Filter<Order>): Order[] {
    return this.all().filter((order) =>
      (Object.entries(filter) as [keyof Order, unknown][]).every(([key, expected]) => {
        const actual = order[key];
        if (expected && typeof expected === "object" && !(expected instanceof Date)) {
          const op = expected as { $in?: unknown[]; $ne?: unknown; $gte?: Date; $lte?: Date };
          if (op.$in  !== undefined) return op.$in.includes(actual);
          if (op.$ne  !== undefined) return actual !== op.$ne;
          if (op.$gte !== undefined && (actual as Date) < op.$gte) return false;
          if (op.$lte !== undefined && (actual as Date) > op.$lte) return false;
          return true;
        }
        return actual === expected;
      }),
    );
  }

  async findById(orderId: OrderId): Promise<Order | null> {
    return this.rows.get(orderId) ?? null;
  }

  async findMany(filter: Filter<Order>, options: QueryOptions<Order> = {}): Promise<Order[]> {
    const rows = this.select(filter);
    const skip = options.skip ?? 0;
    return rows.slice(skip, skip + (options.limit ?? rows.length));
  }

  async findPage(filter: Filter<Order>, page: number, pageSize: number): Promise<Page<Order>> {
    const rows  = this.select(filter).sort((a, b) => b.placedAt.getTime() - a.placedAt.getTime());
    const items = rows.slice(page * pageSize, page * pageSize + pageSize);
    return { items, total: rows.length, page, pageSize, hasMore: page * pageSize + items.length < rows.length };
  }

  async create(input: CreateOrderInput): Promise<Order> {
    const now   = new Date();
    const order: Order = {
      ...input,
      orderId:    parseOrderId(new Types.ObjectId().toHexString()),
      totalCents: input.lines.reduce((sum, l) => sum + l.priceCents * l.quantity, 0),
      placedAt:   now,
      updatedAt:  now,
    };
    this.seq += 1;
    this.rows.set(order.orderId, order);
    return order;
  }

  async update(orderId: OrderId, patch: OrderPatch): Promise<Order | null> {
    const existing = this.rows.get(orderId);
    if (!existing) return null;

    const lines      = patch.lines ?? existing.lines;
    const totalCents = lines.reduce((sum, l) => sum + l.priceCents * l.quantity, 0);
    const updated: Order = {
      ...existing,
      status: patch.status ?? existing.status,
      lines,
      totalCents,
      updatedAt: new Date(),
    };
    this.rows.set(orderId, updated);
    return updated;
  }

  async delete(orderId: OrderId): Promise<boolean> { return this.rows.delete(orderId); }
  async count(filter: Filter<Order>): Promise<number> { return this.select(filter).length; }

  async findPendingOlderThan(cutoff: Date, limit: number): Promise<Order[]> {
    return this.all()
      .filter((o) => o.status === "pending" && o.placedAt < cutoff)
      .sort((a, b) => a.placedAt.getTime() - b.placedAt.getTime())
      .slice(0, limit);
  }

  async sumRevenueCents(userId: UserId): Promise<number> {
    return this.all()
      .filter((o) => o.userId === userId && (o.status === "paid" || o.status === "shipped"))
      .reduce((sum, o) => sum + o.totalCents, 0);
  }
}

// ════════════════════════════════════════════════════════════════════════════
// 6. The service — depends only on the interface
// ════════════════════════════════════════════════════════════════════════════

export interface Clock { now(): Date; }

export class OrderService {
  constructor(
    private readonly orders: OrderRepository,
    private readonly clock:  Clock,
  ) {}

  async placeOrder(userId: UserId, lines: readonly OrderLine[], currency: Order["currency"]): Promise<Order> {
    if (lines.length === 0) throw new Error("An order needs at least one line");
    return this.orders.create({ userId, lines, status: "pending", currency });
  }

  async markPaid(orderId: OrderId): Promise<Order> {
    const order = await this.orders.findById(orderId);
    if (!order) throw new Error(`Order not found: ${orderId}`);
    if (order.status !== "pending") throw new Error(`Cannot pay an order in status ${order.status}`);

    const updated = await this.orders.update(orderId, { status: "paid" });
    if (!updated) throw new Error(`Order vanished during update: ${orderId}`);
    return updated;
  }

  // A cron job: cancel orders left pending for more than 24h.
  async expireStaleOrders(): Promise<number> {
    const cutoff = new Date(this.clock.now().getTime() - 24 * 60 * 60 * 1000);
    const stale  = await this.orders.findPendingOlderThan(cutoff, 100);
    for (const order of stale) {
      await this.orders.update(order.orderId, { status: "cancelled" });
    }
    return stale.length;
  }
}

// ════════════════════════════════════════════════════════════════════════════
// 7. Tests — no Mongo, no Docker, no mocking library, deterministic time
// ════════════════════════════════════════════════════════════════════════════

// const fixedClock: Clock = { now: () => new Date("2026-07-22T00:00:00Z") };
// const repo    = new InMemoryOrderRepository();
// const service = new OrderService(repo, fixedClock);
//
// const order = await service.placeOrder(
//   parseUserId("507f1f77bcf86cd799439011"),
//   [{ sku: "SKU-1", quantity: 2, priceCents: 1500 }],
//   "USD",
// );
// assert.equal(order.totalCents, 3000);
// assert.equal((await service.markPaid(order.orderId)).status, "paid");
//
// Production wiring is the only place the concrete class is named:
// const service = new OrderService(new MongoOrderRepository(OrderModel), { now: () => new Date() });
```

---

## Going deeper

### `implements` does not narrow — it only checks

A classic surprise: `implements` is a *check*, not an inference source. Method parameters do **not** get contextually typed from the interface.

```ts
interface UserRepository { findById(userId: UserId): Promise<User | null>; }

class MongoUserRepository implements UserRepository {
  // ❌ under `noImplicitAny`: Parameter 'userId' implicitly has an 'any' type.
  // async findById(userId) { ... }

  // ✅ you must annotate:
  async findById(userId: UserId): Promise<User | null> { /* ... */ return null; }
}

// If you want inference, type the VARIABLE instead of using `implements`:
const repo: UserRepository = {
  async findById(userId) {          // ✅ userId inferred as UserId from the annotation
    return null;
  },
};
```

### Method bivariance lets a wrong implementation slip through

Parameters of methods declared with **method syntax** are checked bivariantly, even under `strictFunctionTypes`. Only *property* syntax gets strict contravariant checking.

```ts
interface RepoMethodSyntax   { update(id: UserId, patch: UserPatch): Promise<void>; }
interface RepoPropertySyntax { update: (id: UserId, patch: UserPatch) => Promise<void>; }

// ❗ This compiles — method syntax is bivariant:
const sloppy: RepoMethodSyntax = {
  async update(id: UserId, patch: { status: "active" }) {},  // narrower than UserPatch!
};

// ✅ This is correctly rejected — property syntax is contravariant:
// const strict: RepoPropertySyntax = {
//   async update(id: UserId, patch: { status: "active" }) {}, // ❌ not assignable
// };
```

If a repository interface is a genuine safety boundary, declaring its members with property/arrow syntax buys you real variance checking.

### `null` vs `undefined` for "not found" — pick one and enforce it

Mongoose returns `null`. `Map.get` returns `undefined`. `Array.find` returns `undefined`. Prisma returns `null`. If your interface says `Promise<User | null>` and an implementation returns `undefined`, `strictNullChecks` will catch it — but only if you actually annotate the return type. Always normalise at the repository boundary:

```ts
return this.rows.get(userId) ?? null;             // ✅ normalise once, here
return docs.find((d) => d.sku === sku) ?? null;   // ✅
```

Standardising on `null` also survives JSON round-trips, which `undefined` does not (`JSON.stringify({ a: undefined })` → `"{}"`).

### `Omit` silently accepts keys that don't exist

`Omit<T, K>` does not constrain `K` to `keyof T`. A typo in an omit list produces a type that is quietly wrong, not an error.

```ts
type CreateUserInput = Omit<User, "userIdd">;   // ✅ compiles! userId is still present.

// ✅ A strict version that rejects unknown keys:
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
// type Bad = StrictOmit<User, "userIdd">;      // ❌ correctly errors
```

Use `StrictOmit` everywhere you derive input types — this bug is extremely easy to ship because both sides look reasonable in review.

### `Partial<T>` and `exactOptionalPropertyTypes`

By default `Partial<{ status: "active" }>` is `{ status?: "active" | undefined }` — you can *explicitly* pass `undefined`. Under `exactOptionalPropertyTypes: true` (tsconfig), it becomes `{ status?: "active" }`: the key may be absent, but you cannot pass `undefined` for it. That flag is worth enabling for repository patch types specifically, because it removes an entire class of "I meant to skip it but wrote undefined" bugs.

```ts
// With exactOptionalPropertyTypes: true
declare function update(patch: Partial<{ status: "active" | "suspended" }>): void;
update({});                       // ✅
update({ status: "active" });     // ✅
// update({ status: undefined }); // ❌ now rejected
```

### The N+1 trap the pattern makes easy to write

A repository that only exposes `findById` invites this:

```ts
// ❌ 1 + N queries. Looks innocent, kills your database at 500 orders.
for (const order of orders) {
  const user = await userRepo.findById(order.userId);
}

// ✅ Add a batch method to the contract and use it:
interface UserRepository {
  findByIds(userIds: readonly UserId[]): Promise<Map<UserId, User>>;
}

const usersById = await userRepo.findByIds(orders.map((o) => o.userId));
for (const order of orders) {
  const user = usersById.get(order.userId);   // User | undefined
}
```

Returning a `Map<ID, T>` rather than `T[]` is deliberate: it forces the caller to handle misses and makes the lookup O(1).

### Transactions do not fit the naive interface

`findById` and `update` on separate repositories cannot participate in one transaction unless something carries the session. Two workable typed approaches:

```ts
// Option A — an explicit unit-of-work handle threaded through the calls.
interface UnitOfWork { readonly txId: symbol; }

interface OrderRepository {
  update(orderId: OrderId, patch: OrderPatch, uow?: UnitOfWork): Promise<Order | null>;
}

interface TransactionRunner {
  run<T>(fn: (uow: UnitOfWork) => Promise<T>): Promise<T>;
}

// Option B — a repository factory scoped to a transaction.
interface RepositoryFactory {
  orders(uow?: UnitOfWork): OrderRepository;
  users(uow?: UnitOfWork): UserRepository;
}
```

Option B keeps method signatures clean at the cost of one extra indirection. Whichever you pick, decide before you have twenty repositories.

### Performance: `.lean()` matters more than the abstraction

The repository layer adds one object allocation per row (the mapper). Hydrating full Mongoose documents costs roughly an order of magnitude more. `.lean<TDoc>()` returns plain objects and skips document construction entirely — combined with a mapper, you get both speed and a clean domain type. The mapper is not your bottleneck; the ODM you avoided is.

### When the pattern is over-engineering

Be honest about this. Skip repositories when:

- The project is a single-purpose script, a Lambda with three handlers, or a prototype you expect to delete.
- You use Prisma or Drizzle and are genuinely happy coupling to them — both already generate typed models and typed queries, so a hand-written repository mostly re-implements what you already have. A thin `UserQueries` module of named functions gives you 80% of the value for 10% of the code.
- Every repository method is a one-line passthrough to the ORM and you have exactly one implementation and no tests that need a fake. That is a layer of indirection buying nothing.
- Your "repository" ends up exposing driver types (`FilterQuery`, `PrismaClient`, `QueryBuilder`) in its signatures. At that point it is not an abstraction, it is an alias — and it costs more than it saves.

The pattern pays for itself when there is (a) real domain logic to keep clean, (b) a second implementation such as a test fake or a cache decorator, or (c) a persistence shape that meaningfully differs from the domain shape. If none of the three is true, write the query.

---

## Common mistakes

### Mistake 1 — Leaking persistence types through the interface

```ts
// ❌ The "abstraction" imports Mongoose. Every consumer now transitively depends on it,
//    and no non-Mongo implementation can ever satisfy the contract.
import type { FilterQuery, HydratedDocument } from "mongoose";

interface UserRepository {
  findMany(query: FilterQuery<UserDoc>): Promise<HydratedDocument<UserDoc>[]>;
  findById(id: string): Promise<UserDocument | null>;
}

// ✅ Domain types only. Mongoose appears in the implementation file, nowhere else.
interface UserRepository {
  findMany(filter: Filter<User>, options?: QueryOptions<User>): Promise<User[]>;
  findById(userId: UserId): Promise<User | null>;
}
```

### Mistake 2 — `update(id, patch: Partial<T>)` fed from request input

```ts
// ❌ Mass assignment. Compiles perfectly. Grants admin to anyone who asks nicely.
interface UserRepository {
  update(userId: UserId, patch: Partial<User>): Promise<User | null>;
}

app.patch("/users/:id", async (req, res) => {
  const updated = await userRepo.update(parseUserId(req.params.id), req.body as Partial<User>);
  res.json(updated);
});
// POST body: { "role": "owner", "status": "active" }  → privilege escalation

// ✅ Narrow the patch type, and validate the body into that exact type.
type SelfEditablePatch = { displayName?: string; email?: string };

const SelfEditSchema = z.object({
  displayName: z.string().min(1).max(80).optional(),
  email:       z.string().email().optional(),
}).strict();                                  // .strict() rejects unknown keys at runtime

app.patch("/users/:id", async (req, res) => {
  const patch: SelfEditablePatch = SelfEditSchema.parse(req.body);
  const updated = await userRepo.update(parseUserId(req.params.id), patch);
  res.json(updated);
});
```

The compile-time narrowing and the runtime `.strict()` schema are both required — the first stops your own code, the second stops the network.

### Mistake 3 — Returning the entity from `create` without the generated fields

```ts
// ❌ The input type is returned, so callers don't get the id the database assigned.
interface UserRepository {
  create(input: CreateUserInput): Promise<CreateUserInput>;   // no userId, no createdAt
}
const created = await userRepo.create({ email, displayName, status: "active" });
// created.userId  → ❌ Property 'userId' does not exist

// ✅ Take the input type, return the full entity.
interface UserRepository {
  create(input: CreateUserInput): Promise<User>;
}
const created = await userRepo.create({ email, displayName, status: "active" });
created.userId;     // ✅ UserId
created.createdAt;  // ✅ Date
```

### Mistake 4 — A generic base class that forces every entity into the same shape

```ts
// ❌ Assumes every entity has a numeric `id` field. Breaks for composite keys,
//    branded ids, UUIDs, and any entity whose key is named differently.
abstract class BaseRepository<T extends { id: number }> {
  abstract findById(id: number): Promise<T | null>;
}
// class OrderRepo extends BaseRepository<Order> {}  // ❌ Order has orderId: OrderId

// ✅ Parameterise the ID type; do not assume the key's name or type.
abstract class BaseRepository<T, ID> {
  abstract findById(id: ID): Promise<T | null>;
}
class OrderRepo extends BaseRepository<Order, OrderId> {
  async findById(orderId: OrderId): Promise<Order | null> { /* ... */ return null; }
}
```

### Mistake 5 — Unbounded queries in the contract

```ts
// ❌ Nothing stops a caller loading two million rows into memory.
interface OrderRepository {
  findAll(): Promise<Order[]>;
}

// ✅ Make the limit part of the type, and cap it in the implementation.
interface OrderRepository {
  findMany(filter: Filter<Order>, options: QueryOptions<Order> & { limit: number }): Promise<Order[]>;
  streamAll(filter: Filter<Order>): AsyncIterable<Order>;    // for genuine bulk work
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a typed repository for a `Session` entity backed by an in-memory `Map`.

Requirements:

- Define a branded `SessionId` type and a `parseSessionId(raw: string): SessionId` parser that rejects anything not matching `sess_` followed by 32 hex characters.
- Define the entity:
  `Session = { sessionId, userId, authToken, ipAddress, userAgent, createdAt, expiresAt, revokedAt: Date | null }` — all fields `readonly`.
- Derive `CreateSessionInput` from the entity (omit `sessionId`, `createdAt`, `revokedAt`) and `SessionPatch` allowing only `expiresAt` and `revokedAt`.
- Define `SessionRepository` with: `findById`, `findByAuthToken`, `findActiveByUserId(userId): Promise<Session[]>`, `create`, `revoke(sessionId): Promise<boolean>`, `deleteExpired(now: Date): Promise<number>`.
- Implement `InMemorySessionRepository` satisfying the interface. "Active" means `revokedAt === null` **and** `expiresAt > now`.
- Show one line that fails to compile because it tries to patch `userId`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a reusable generic `InMemoryRepository<T, ID>` base class and use it for two different entities.

Requirements:

- `abstract class InMemoryRepository<TEntity, TId extends string>` implementing `findById`, `findMany`, `create`, `update`, `delete`, `count`.
- The subclass supplies the id field name and the id generator via two abstract members:
  `protected abstract readonly idKey: keyof TEntity;` and `protected abstract generateId(): TId;`
- Implement a typed `Filter<T>` supporting shorthand equality, `$in`, `$ne`, and (for `number | Date` fields only) `$gte` / `$lte` — comparison operators must be a compile error on `string` fields.
- Implement `QueryOptions<T>` with a multi-key `sort` (later keys break ties), `skip`, `limit`, and a typed `select` that narrows the return type to `Pick<T, K>[]`.
- Extend it twice: `InMemoryUserRepository` (entity `User`, id `UserId`) and `InMemoryInvoiceRepository` (entity `Invoice`, id `InvoiceId`), each adding one domain-specific finder.
- The `update` method must take a per-entity patch type, not `Partial<TEntity>` — show how the base class stays generic while subclasses narrow the patch.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **layered repository stack** for a multi-tenant SaaS, where each layer implements the same interface.

Requirements:

- Domain: `Project = { projectId, orgId, name, ownerId, archivedAt: Date | null, createdAt, updatedAt }` with branded `ProjectId`, `OrgId`, `UserId`.
- `ProjectRepository` interface with `findById`, `findPage`, `create`, `update`, `archive`, `count`, and `findByIds(ids): Promise<Map<ProjectId, Project>>`.
- Layer 1 — `MongoProjectRepository`: a persistence model whose field names differ from the domain (`org`, `owner`, `deleted_at`), a `toDomain` mapper, and a `toQuery` translator. Soft deletes are applied automatically.
- Layer 2 — `TenantScopedProjectRepository`: wraps any `ProjectRepository` plus a `RequestContext { orgId: OrgId }`. It must inject `orgId` into every filter and **return `null` from `findById` if the found project belongs to a different org**. Make it structurally impossible for a caller to bypass the scope.
- Layer 3 — `CachedProjectRepository`: wraps any `ProjectRepository` plus a `CacheClient` interface you define. Caches `findById` and `findByIds`, invalidates on `update` and `archive`, and never caches `findPage`.
- Layer 4 — `InstrumentedProjectRepository`: wraps any `ProjectRepository` plus a `Metrics { timing(name: string, ms: number): void }`, recording per-method latency. Write it so adding a method to `ProjectRepository` forces you to update this class (hint: `implements` plus explicit delegation, no `Proxy`).
- Write `composeProjectRepository(deps): ProjectRepository` as the composition root, wiring all four layers in the right order and returning only the interface type.
- Write an `InMemoryProjectRepository` and a test showing that swapping it into `composeProjectRepository` requires changing exactly one line.
- Finally: add `restore(projectId): Promise<Project | null>` to the interface and list every file that now fails to compile. Explain why that is the point.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Branded id ──────────────────────────────────────────────────────────────
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };
type UserId = Brand<string, "UserId">;
const parseUserId = (raw: string): UserId => raw as UserId;   // one cast, at the boundary

// ── Derived input/patch types ───────────────────────────────────────────────
type StrictOmit<T, K extends keyof T> = Omit<T, K>;           // Omit that catches typos
type CreateInput = StrictOmit<User, "userId" | "createdAt">;
type Patch = { -readonly [K in "email" | "displayName"]?: User[K] };  // explicit surface

// ── Typed filter / sort / options ───────────────────────────────────────────
type Filter<T> = { [K in keyof T]?: T[K] | { $in?: readonly T[K][]; $ne?: T[K] } };
type Sort<T>   = { readonly [K in keyof T]?: "asc" | "desc" };
interface QueryOptions<T> { sort?: Sort<T>; skip?: number; limit?: number; select?: (keyof T)[] }
interface Page<T> { items: readonly T[]; total: number; page: number; pageSize: number; hasMore: boolean }

// ── The contract ────────────────────────────────────────────────────────────
interface Repository<T, ID> {
  findById(id: ID): Promise<T | null>;
  findByIds(ids: readonly ID[]): Promise<Map<ID, T>>;         // batch — avoids N+1
  findMany(filter: Filter<T>, options?: QueryOptions<T>): Promise<T[]>;
  create(input: CreateInput): Promise<T>;
  update(id: ID, patch: Patch): Promise<T | null>;
  delete(id: ID): Promise<boolean>;
  count(filter: Filter<T>): Promise<number>;
}

// ── Decorating ──────────────────────────────────────────────────────────────
const repo: UserRepository = new CachedUserRepository(new MongoUserRepository(UserModel), redis);
```

| Rule | Detail |
|---|---|
| Entity ≠ document | Domain type has no `_id`, no `__v`, no `passwordHash`, no ODM methods |
| Interface imports nothing from the driver | `FilterQuery`/`HydratedDocument` in a contract = not an abstraction |
| Never `patch: Partial<T>` from request input | Derive an explicit patch type; validate with `.strict()` at runtime |
| Strip identity fields from patches | `Exclude<keyof T, "userId" \| "createdAt">` with `-readonly` |
| Brand your IDs | `Brand<string, "UserId">` — stops argument swaps at compile time |
| Normalise "not found" to `null` | `map.get(id) ?? null`, `doc ? toDomain(doc) : null` |
| Bound every query | Cap `limit` inside the implementation; no `findAll()` |
| Batch, don't loop | `findByIds(): Promise<Map<ID, T>>` instead of N × `findById` |
| Use `StrictOmit`, not `Omit` | `Omit<T, "typoo">` compiles and is silently wrong |
| Fakes are first-class | The in-memory implementation is the executable spec of the contract |
| Skip the pattern when | One implementation, no tests, Prisma/Drizzle already typed, throwaway code |

---

## Connected topics

- **26 — Interfaces** — the repository contract is an interface; `implements` checks a class against it without contextually typing its parameters.
- **31 — Generics** — `Repository<T, ID>` and the generic base class are the payoff for understanding type parameters and constraints.
- **32 — Utility types** — `Omit`, `Partial`, `Pick`, `Exclude`, `Record` build every input, patch, and projection type here.
- **43 — Mapped types** — `Filter<T>`, `Sort<T>`, and `UpdatePatch<T>` are all mapped types over `keyof T`, including the `-readonly` modifier.
- **68 — Dependency injection basics** — the natural sequel: how the composition root wires a concrete repository into a service, and swaps a fake in tests.
