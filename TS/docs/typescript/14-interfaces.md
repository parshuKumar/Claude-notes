# 14 — Interfaces

## What is this?

An interface defines the **shape of an object** — what properties it has, what types those properties are, and what methods it exposes. You declare an interface with the `interface` keyword. Unlike type aliases (which can describe anything), interfaces are specifically designed for describing object shapes, and they integrate directly with TypeScript's class system via the `implements` keyword. Think of an interface as a contract: "any object that calls itself this type must have exactly these properties."

## Why does it matter?

In JavaScript, you pass objects around everywhere with no guarantee about their shape. A function receives an object and you hope it has the right fields. In TypeScript, interfaces let you declare the exact contract an object must fulfill — if a field is missing, the wrong type, or misspelled, TypeScript catches it before your code runs. Interfaces also serve as architectural contracts between different parts of your codebase — a repository interface that multiple implementations must fulfill, a service interface used across controllers, a config interface shared across modules.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — you pass objects around and hope they have the right shape
function createUser(data) {
  // What fields does data need? You have to read the function body.
  // Is 'email' required? Is 'role' optional? No way to know without reading.
  return {
    id: Date.now(),
    email: data.email,
    name: data.name,
    role: data.role || "user",
    createdAt: new Date(),
  };
}

// These all silently "work" but produce broken objects:
createUser({ email: "parsh@dev.io" });                  // missing name — undefined
createUser({ email: "parsh@dev.io", nme: "Parsh" });   // typo in name — undefined
createUser({ emial: "parsh@dev.io", name: "Parsh" });  // typo in email — undefined
```

```ts
// TypeScript — the interface is the contract
interface CreateUserBody {
  email: string;
  name: string;
  role?: "admin" | "user";   // optional field
}

interface User {
  id: number;
  email: string;
  name: string;
  role: "admin" | "user";
  createdAt: Date;
}

function createUser(data: CreateUserBody): User {
  return {
    id: Date.now(),
    email: data.email,
    name: data.name,
    role: data.role ?? "user",
    createdAt: new Date(),
  };
}

createUser({ email: "parsh@dev.io" });                  // ❌ missing 'name'
createUser({ email: "parsh@dev.io", nme: "Parsh" });   // ❌ 'nme' not in interface, missing 'name'
createUser({ emial: "parsh@dev.io", name: "Parsh" });  // ❌ 'emial' not in interface, missing 'email'
createUser({ email: "parsh@dev.io", name: "Parsh" });  // ✅
```

---

## Syntax

```ts
// ── BASIC INTERFACE ───────────────────────────────────────────────────────
interface User {
  id: number;               // required property
  email: string;            // required property
  name: string;             // required property
}

// ── OPTIONAL PROPERTIES ───────────────────────────────────────────────────
interface UserProfile {
  id: number;
  name: string;
  bio?: string;             // optional — string | undefined
  avatarUrl?: string;       // optional — string | undefined
}

// ── READONLY PROPERTIES ───────────────────────────────────────────────────
interface DatabaseRecord {
  readonly id: number;      // can be set once, never changed
  readonly createdAt: Date; // immutable after creation
  updatedAt: Date;          // mutable
}

// ── INTERFACE WITH METHODS ────────────────────────────────────────────────
interface UserService {
  findById(id: number): Promise<User | null>;
  findAll(): Promise<User[]>;
  create(data: CreateUserBody): Promise<User>;
  update(id: number, data: Partial<User>): Promise<User | null>;
  delete(id: number): Promise<void>;
}

// ── INTERFACE WITH INDEX SIGNATURE ────────────────────────────────────────
interface StringMap {
  [key: string]: string;    // any string key → string value
}

// ── INTERFACE USED AS FUNCTION PARAM ─────────────────────────────────────
function sendEmail(user: User, subject: string): void {
  console.log(`Sending to ${user.email}: ${subject}`);
}
```

---

## How it works — property by property

### Required vs optional

```ts
interface CreatePostBody {
  title: string;              // required — must provide
  content: string;            // required — must provide
  status?: "draft" | "published"; // optional — may omit or provide
  tags?: string[];            // optional — may omit or provide
}

// Missing required field:
const body1: CreatePostBody = { title: "Hello" };
// ❌ Property 'content' is missing in type '{ title: string }'

// Extra field not in interface:
const body2: CreatePostBody = { title: "Hello", content: "World", extra: true };
// ❌ Object literal may only specify known properties — 'extra' doesn't exist

// All required fields, optional fields omitted:
const body3: CreatePostBody = { title: "Hello", content: "World" };
// ✅

// All fields provided:
const body4: CreatePostBody = {
  title: "Hello",
  content: "World",
  status: "published",
  tags: ["typescript", "backend"],
};
// ✅
```

### Readonly properties

```ts
interface User {
  readonly id: number;      // set once, immutable after
  readonly createdAt: Date; // set once, immutable after
  name: string;             // mutable
  email: string;            // mutable
}

const user: User = { id: 1, createdAt: new Date(), name: "Parsh", email: "p@dev.io" };

user.name = "Alice";        // ✅ mutable
user.email = "a@dev.io";    // ✅ mutable
user.id = 999;              // ❌ Cannot assign to 'id' because it is a read-only property
user.createdAt = new Date(); // ❌ readonly
```

### Method signatures in interfaces

```ts
// Two equivalent ways to declare methods on an interface:

interface Calculator {
  // Method syntax (preferred):
  add(a: number, b: number): number;
  subtract(a: number, b: number): number;

  // Property function syntax (equivalent):
  multiply: (a: number, b: number) => number;
}

// An object that satisfies this interface:
const calculator: Calculator = {
  add(a, b) { return a + b; },              // TypeScript infers param types from interface
  subtract(a, b) { return a - b; },
  multiply: (a, b) => a * b,
};
```

### Excess property checking — object literals only

TypeScript is stricter with **object literals** than with variables:

```ts
interface Config {
  host: string;
  port: number;
}

// Direct object literal — STRICT:
const config: Config = { host: "localhost", port: 3000, debug: true };
// ❌ Object literal may only specify known properties: 'debug' not in Config

// Variable assignment — LENIENT (structural subtyping):
const myConfig = { host: "localhost", port: 3000, debug: true };
const config2: Config = myConfig;   // ✅ myConfig has all Config properties (and more)
```

This is called **excess property checking** — it only applies to fresh object literals, not to variables.

---

## Interfaces vs type aliases — the key similarities

Both describe object shapes and are largely interchangeable for objects:

```ts
// These two are functionally equivalent:
interface User {
  id: number;
  name: string;
}

type User = {
  id: number;
  name: string;
};

// Both can be used as function params, return types, class contracts, etc.
```

The differences are covered in depth in topic 15. For now: **use `interface` for object shapes that define a contract** (services, repositories, request/response shapes), and `type` for unions, intersections, primitives, and computed types.

---

## Example 1 — basic

```ts
// A complete interface-driven user module

interface User {
  readonly id: number;
  email: string;
  name: string;
  role: "admin" | "editor" | "viewer";
  readonly createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

interface CreateUserBody {
  email: string;
  name: string;
  role?: "admin" | "editor" | "viewer";
}

interface UpdateUserBody {
  email?: string;
  name?: string;
  role?: "admin" | "editor" | "viewer";
}

// In-memory store typed to the interface:
const users: User[] = [];

function createUser(body: CreateUserBody): User {
  const newUser: User = {
    id: Date.now(),
    email: body.email,
    name: body.name,
    role: body.role ?? "viewer",
    createdAt: new Date(),
    updatedAt: new Date(),
    deletedAt: null,
  };
  users.push(newUser);
  return newUser;
}

function updateUser(id: number, body: UpdateUserBody): User | null {
  const user = users.find(u => u.id === id);
  if (!user) return null;

  if (body.email) user.email = body.email;
  if (body.name) user.name = body.name;
  if (body.role) user.role = body.role;
  user.updatedAt = new Date();

  return user;
}

function softDeleteUser(id: number): boolean {
  const user = users.find(u => u.id === id);
  if (!user) return false;
  user.deletedAt = new Date();
  return true;
}
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response, NextFunction } from 'express';

// ── Repository interface — the contract any DB implementation must fulfill ──

interface UserRepository {
  findById(id: number): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(options?: { page: number; pageSize: number }): Promise<User[]>;
  create(data: CreateUserBody): Promise<User>;
  update(id: number, data: UpdateUserBody): Promise<User | null>;
  delete(id: number): Promise<boolean>;
  count(): Promise<number>;
}

// ── Service interface — business logic layer ───────────────────────────────

interface UserService {
  getUser(id: number): Promise<User>;                     // throws if not found
  getUserByEmail(email: string): Promise<User | null>;
  listUsers(page: number, pageSize: number): Promise<{ users: User[]; total: number }>;
  createUser(body: CreateUserBody): Promise<User>;
  updateUser(id: number, body: UpdateUserBody): Promise<User>;
  deleteUser(id: number): Promise<void>;
}

// ── In-memory implementation of the repository ─────────────────────────────

class InMemoryUserRepository implements UserRepository {
  private store: User[] = [];

  async findById(id: number): Promise<User | null> {
    return this.store.find(u => u.id === id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.store.find(u => u.email === email) ?? null;
  }

  async findAll(options?: { page: number; pageSize: number }): Promise<User[]> {
    if (!options) return [...this.store];
    const start = (options.page - 1) * options.pageSize;
    return this.store.slice(start, start + options.pageSize);
  }

  async create(data: CreateUserBody): Promise<User> {
    const user: User = {
      id: Date.now(),
      email: data.email,
      name: data.name,
      role: data.role ?? "viewer",
      createdAt: new Date(),
      updatedAt: new Date(),
      deletedAt: null,
    };
    this.store.push(user);
    return user;
  }

  async update(id: number, data: UpdateUserBody): Promise<User | null> {
    const user = this.store.find(u => u.id === id);
    if (!user) return null;
    Object.assign(user, data, { updatedAt: new Date() });
    return user;
  }

  async delete(id: number): Promise<boolean> {
    const index = this.store.findIndex(u => u.id === id);
    if (index === -1) return false;
    this.store.splice(index, 1);
    return true;
  }

  async count(): Promise<number> {
    return this.store.length;
  }
}

// ── Service implementation wired to the repository ─────────────────────────

class UserServiceImpl implements UserService {
  constructor(private readonly repo: UserRepository) {}
  // Note: constructor(private readonly repo) uses the interface type, NOT the class
  // This means you could swap InMemoryUserRepository for a MongooseUserRepository
  // without changing a single line of service code — that's the power of interfaces

  async getUser(id: number): Promise<User> {
    const user = await this.repo.findById(id);
    if (!user) throw new Error(`User ${id} not found`);
    return user;
  }

  async getUserByEmail(email: string): Promise<User | null> {
    return this.repo.findByEmail(email);
  }

  async listUsers(page: number, pageSize: number): Promise<{ users: User[]; total: number }> {
    const [users, total] = await Promise.all([
      this.repo.findAll({ page, pageSize }),
      this.repo.count(),
    ]);
    return { users, total };
  }

  async createUser(body: CreateUserBody): Promise<User> {
    const existing = await this.repo.findByEmail(body.email);
    if (existing) throw new Error(`Email ${body.email} is already registered`);
    return this.repo.create(body);
  }

  async updateUser(id: number, body: UpdateUserBody): Promise<User> {
    const user = await this.repo.update(id, body);
    if (!user) throw new Error(`User ${id} not found`);
    return user;
  }

  async deleteUser(id: number): Promise<void> {
    const deleted = await this.repo.delete(id);
    if (!deleted) throw new Error(`User ${id} not found`);
  }
}

// ── Route handler using the service interface ──────────────────────────────

function makeUserHandlers(service: UserService) {
  // Takes UserService INTERFACE — works with any implementation
  return {
    async getUser(req: Request, res: Response): Promise<void> {
      try {
        const user = await service.getUser(parseInt(req.params.id, 10));
        res.json({ success: true, data: user });
      } catch (err) {
        res.status(404).json({ success: false, error: (err as Error).message });
      }
    },

    async createUser(req: Request, res: Response): Promise<void> {
      try {
        const user = await service.createUser(req.body as CreateUserBody);
        res.status(201).json({ success: true, data: user });
      } catch (err) {
        res.status(400).json({ success: false, error: (err as Error).message });
      }
    },
  };
}

// Wire it up:
const repo = new InMemoryUserRepository();
const userService = new UserServiceImpl(repo);
const userHandlers = makeUserHandlers(userService);
```

---

## Common mistakes

### Mistake 1 — Forgetting that optional properties are `type | undefined`

```ts
interface UserProfile {
  name: string;
  bio?: string;   // type is: string | undefined
}

function formatProfile(profile: UserProfile): string {
  // ❌ WRONG — bio might be undefined
  return `${profile.name}: ${profile.bio.toUpperCase()}`;
  // Error: Object is possibly 'undefined'

  // ✅ RIGHT — handle the undefined case
  const bio = profile.bio ?? "No bio provided";
  return `${profile.name}: ${bio.toUpperCase()}`;
}
```

### Mistake 2 — Trying to add extra properties to an object literal against an interface

```ts
interface Config {
  host: string;
  port: number;
}

// ❌ TypeScript catches extra properties on OBJECT LITERALS:
function connect(config: Config): void { ... }

connect({ host: "localhost", port: 3000, timeout: 5000 });
// ❌ Argument of type '{ ...; timeout: number }' is not assignable to 'Config'
// Object literal may only specify known properties

// ✅ If you genuinely need extra properties, add them to the interface:
interface Config {
  host: string;
  port: number;
  timeout?: number;   // add it explicitly
}

// Or use an index signature for truly dynamic extra properties:
interface FlexibleConfig {
  host: string;
  port: number;
  [key: string]: unknown;   // allows any additional properties
}
```

### Mistake 3 — Using interfaces where type aliases are the better fit

```ts
// ❌ AWKWARD — using interface for a union type (doesn't work)
interface UserId = string | number;   // ❌ Syntax error — interface can't be a union

// ❌ AWKWARD — using interface for a function type (works but is verbose)
interface Formatter {
  (value: string): string;   // call signature — valid but unusual for simple functions
}

// ✅ RIGHT — use type alias for unions, primitives, functions:
type UserId = string | number;
type Formatter = (value: string) => string;

// ✅ RIGHT — use interface for object shapes and class contracts:
interface UserService {
  findById(id: number): Promise<User | null>;
  create(data: CreateUserBody): Promise<User>;
}
```

---

## Practice exercises

### Exercise 1 — easy

Define an interface `ServerConfig` with:
- `host: string` (required)
- `port: number` (required)
- `protocol: "http" | "https"` (required)
- `maxConnections?: number` (optional)
- `readonly startedAt: Date` (readonly)

Write a function `formatServerUrl(config: ServerConfig): string` that returns `"https://localhost:3000"`.

Create two `ServerConfig` objects — one with only required fields, one with all fields. Verify that assigning to `startedAt` after creation causes a TypeScript error.

```ts
// Write your code here
```

### Exercise 2 — medium

Define these interfaces for a blog system:

```
Author — id, name, email, bio (optional)
Post — id, title, content, authorId, status ("draft" | "published"), tags (string[]), publishedAt (Date | null), createdAt, updatedAt
CreatePostBody — title, content, tags (optional), status (optional, defaults to "draft")
UpdatePostBody — all CreatePostBody fields optional
```

Write a `PostRepository` interface with: `findById`, `findByAuthor(authorId: number): Promise<Post[]>`, `findPublished(page: number, pageSize: number): Promise<Post[]>`, `create`, `update`, `delete`.

Write an in-memory class `InMemoryPostRepository` that implements `PostRepository` with a `private posts: Post[]` store. All methods must fully satisfy the interface contract.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed plugin system using interfaces. Define:

1. `PluginContext` interface — `appName: string`, `version: string`, `logger: { info(msg: string): void; error(msg: string): void; warn(msg: string): void }`, `config: Record<string, unknown>`
2. `Plugin` interface — `name: string`, `version: string`, `description: string`, `initialize(ctx: PluginContext): Promise<void>`, `shutdown(): Promise<void>`, `healthCheck(): Promise<{ healthy: boolean; message: string }>`
3. `PluginRegistry` interface — `register(plugin: Plugin): void`, `get(name: string): Plugin | undefined`, `getAll(): Plugin[]`, `initializeAll(ctx: PluginContext): Promise<void>`, `shutdownAll(): Promise<void>`, `healthCheckAll(): Promise<Array<{ plugin: string; healthy: boolean; message: string }>>`

Implement `PluginRegistryImpl` that implements `PluginRegistry` with a `private plugins: Map<string, Plugin>`.

Then write two concrete plugin classes (`LoggingPlugin` and `RateLimiterPlugin`) that implement `Plugin`. Both should have real `initialize`, `shutdown`, and `healthCheck` implementations (even if simulated).

Wire everything together: register both plugins, initialize them with a context, run health checks, then shut them down.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `id: number` | Required property |
| `bio?: string` | Optional — `string \| undefined` |
| `readonly id: number` | Immutable after initialization |
| `method(): void` | Method signature |
| `method: () => void` | Method as function property |
| `[key: string]: unknown` | Index signature — dynamic keys |
| `interface B extends A` | B inherits all of A's properties (topic 18) |
| `class C implements I` | C must fulfill interface I's contract (topic 19) |

| Rule | Notes |
|------|-------|
| Extra properties on literals | Not allowed — excess property checking |
| Extra properties on variables | Allowed — structural subtyping |
| Optional props accessed | Must handle `undefined` first |
| Readonly props | Only writable at initialization |
| Interface for unions | Not possible — use `type` alias |
| Interface merging | Declaring the same interface twice merges them (topic 15) |

## Connected topics

- **15 — type vs interface** — the real differences; when each is the right choice.
- **18 — Extending interfaces** — `interface Admin extends User { ... }`.
- **19 — Implementing interfaces in classes** — `class UserRepo implements UserRepository`.
- **32 — Utility types** — `Partial<T>`, `Required<T>`, `Pick<T, K>`, `Omit<T, K>` — transform interfaces.
