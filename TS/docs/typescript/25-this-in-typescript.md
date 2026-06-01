# 25 — `this` in TypeScript

## What is this?

`this` in JavaScript refers to the **execution context** — the object that "owns" the current call. It's one of the most notorious sources of bugs in JavaScript because `this` is determined at runtime by *how* a function is called, not where it's defined. Arrow functions capture `this` lexically from the surrounding scope at definition time.

TypeScript adds two things to help:

1. **The `this` parameter** — a fake first parameter named `this` that lets you declare what type `this` must be inside a function. It is erased at compile time and has no effect on the call signature.
2. **The `noImplicitThis` compiler option** — makes TypeScript error when `this` would have an implicit `any` type (i.e., you're accessing `this` inside a regular function with no annotation and no class context).

## Why does it matter?

The classic `this` bug:

```js
class UserService {
  constructor() {
    this.users = [];
  }
  getUsers() {
    return this.users;  // works fine called as service.getUsers()
  }
}

const service = new UserService();
const fn = service.getUsers;  // detached from the object
fn();  // this is now undefined or window — CRASH
```

This crashes silently in JavaScript. TypeScript with the `this` parameter can catch it at compile time. Beyond crash prevention, typing `this` explicitly documents which object a function expects to be called on, which is critical when you pass method references as callbacks, use mixins, or build fluent/chained APIs.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — this is decided at runtime, no compile-time safety
const button = {
  label: "Submit",
  onClick: function() {
    console.log(this.label); // depends on how onClick is called
  },
};

setTimeout(button.onClick, 100); // this = undefined (strict) or window — broken
```

```ts
// TypeScript — declare what 'this' must be
interface Button {
  label: string;
  onClick(this: Button): void;  // 'this' must be Button
}

const button: Button = {
  label: "Submit",
  onClick() {
    console.log(this.label);  // ✅ TypeScript knows this: Button
  },
};

// Passing as a detached callback:
setTimeout(button.onClick, 100);
// ❌ Error: The 'this' context of type 'void' is not assignable to type 'Button'
// TypeScript stops you before the runtime crash

// ✅ Correct — bind it:
setTimeout(button.onClick.bind(button), 100);
```

---

## Syntax

```ts
// ── this PARAMETER — fake first param, erased at compile time ────────────
function getUserName(this: User): string {
  return this.name;  // TypeScript knows this: User
}

// ── this in a method ──────────────────────────────────────────────────────
interface Logger {
  prefix: string;
  log(this: Logger, message: string): void;
}

// ── this: void — function must NOT use this ───────────────────────────────
function pureFunction(this: void, x: number): number {
  // this.anything  // ❌ TypeScript errors — void means no this context
  return x * 2;
}

// ── this in a class ───────────────────────────────────────────────────────
class QueryBuilder {
  private conditions: string[] = [];

  where(this: QueryBuilder, condition: string): this {
    // return type 'this' = the actual subclass instance — enables fluent subclassing
    this.conditions.push(condition);
    return this;
  }
}

// ── ThisType<T> utility — typing 'this' in object literals ───────────────
type Methods = {
  greet(): string;
  shout(): string;
};

const methods: Methods & ThisType<{ name: string }> = {
  greet() { return `Hello, ${this.name}!`; },   // this: { name: string }
  shout() { return `HEY ${this.name.toUpperCase()}!`; },
};
```

---

## How it works — rule by rule

### The `this` parameter is position 0 but not a real parameter

```ts
function formatUser(this: User, prefix: string): string {
  return `${prefix}: ${this.name} <${this.email}>`;
}

// Compiled JavaScript output (this parameter is stripped):
// function formatUser(prefix) {
//   return `${prefix}: ${this.name} <${this.email}>`;
// }

// To call it, you must use .call() or attach it to an object of the right type:
const user: User = { id: 1, name: "Parsh", email: "p@dev.io", role: "admin", createdAt: new Date(), updatedAt: new Date() };
formatUser.call(user, "User");  // ✅ this is User
```

### `noImplicitThis` — catching untyped `this` usage

Enable in `tsconfig.json`:
```json
{ "compilerOptions": { "noImplicitThis": true } }
```

```ts
// Without noImplicitThis — TypeScript silently gives this: any:
function logThis() {
  console.log(this.name); // this: any — no error, but no safety
}

// With noImplicitThis:
function logThis() {
  console.log(this.name);
  // ❌ Error: 'this' implicitly has type 'any' because it does not have a type annotation
}

// Fix: annotate this explicitly:
function logThis(this: { name: string }) {
  console.log(this.name); // ✅ this: { name: string }
}
```

### Arrow functions capture `this` lexically — no `this` parameter needed

Arrow functions don't have their own `this` — they capture it from the enclosing lexical scope. You cannot annotate a `this` parameter on an arrow function:

```ts
class RequestHandler {
  private requestCount = 0;

  // ❌ Regular function loses this when detached:
  handleRegular = function(req: Request): void {
    this.requestCount++;  // this: any — might not be RequestHandler
  };

  // ✅ Arrow function captures this from class instance:
  handleArrow = (req: Request): void => {
    this.requestCount++;  // this: RequestHandler — always correct
  };
}

const handler = new RequestHandler();
const fn = handler.handleArrow;
fn(req);  // ✅ this is still RequestHandler — captured at construction time

// ❌ You cannot put a this parameter on an arrow function:
const bad = (this: RequestHandler, req: Request): void => {};
// Error: An arrow function cannot have a 'this' parameter
```

### `this: void` — preventing accidental `this` usage

Use `this: void` to declare that a function must not depend on `this`:

```ts
// Express route handler should not use 'this':
function getUserHandler(this: void, req: Request, res: Response): void {
  // this.anything  // ❌ Error — this: void
  const id = parseInt(req.params.id, 10);
  res.json({ id });
}

// Callbacks passed to array methods should be pure:
function processItems(items: string[]): string[] {
  const fn: (this: void, item: string) => string = item => item.toUpperCase();
  return items.map(fn);
}
```

### Return type `this` — fluent / builder pattern

Returning `this` (the literal type, not `this: T`) lets subclasses chain methods and get back the **subclass type**, not the base class type:

```ts
class QueryBuilder {
  protected query: string = "SELECT * FROM";
  protected clauses: string[] = [];

  from(this: this, table: string): this {
    this.query += ` ${table}`;
    return this;
  }

  where(this: this, condition: string): this {
    this.clauses.push(`WHERE ${condition}`);
    return this;
  }

  limit(this: this, n: number): this {
    this.clauses.push(`LIMIT ${n}`);
    return this;
  }

  build(): string {
    return [this.query, ...this.clauses].join(" ");
  }
}

class UserQueryBuilder extends QueryBuilder {
  onlyActive(): this {
    return this.where("deleted_at IS NULL");
  }

  withRole(role: string): this {
    return this.where(`role = '${role}'`);
  }
}

const qb = new UserQueryBuilder();
const sql = qb
  .from("users")        // returns UserQueryBuilder (this) ✅
  .onlyActive()         // ✅ exists on UserQueryBuilder — wouldn't if return type was QueryBuilder
  .withRole("admin")    // ✅
  .limit(10)            // ✅
  .build();
// "SELECT * FROM users WHERE deleted_at IS NULL WHERE role = 'admin' LIMIT 10"
```

### `ThisType<T>` — typing `this` inside object literals with mixin patterns

When you compose an object from separate method definitions, `ThisType<T>` tells TypeScript what `this` refers to inside those methods:

```ts
interface DbRecord {
  id: number;
  tableName: string;
}

type ActiveRecordMethods = {
  save(): Promise<void>;
  destroy(): Promise<void>;
  reload(): Promise<void>;
};

function makeActiveRecord<T extends DbRecord>(
  record: T,
  methods: ActiveRecordMethods & ThisType<T & ActiveRecordMethods>,
): T & ActiveRecordMethods {
  return Object.assign(record, methods);
}

const user = makeActiveRecord(
  { id: 1, tableName: "users", name: "Parsh", email: "p@dev.io" },
  {
    async save() {
      // this: { id: number; tableName: string; name: string; email: string } & ActiveRecordMethods
      console.log(`Saving ${this.tableName} id=${this.id}`);
    },
    async destroy() {
      console.log(`Deleting from ${this.tableName} WHERE id=${this.id}`);
    },
    async reload() {
      console.log(`Reloading ${this.tableName} id=${this.id}`);
    },
  }
);

await user.save();    // ✅ this.tableName, this.id available and typed
```

---

## Example 1 — basic

```ts
// Fluent builder with typed this — SQL query builder

class SelectBuilder {
  private _table: string = "";
  private _conditions: string[] = [];
  private _orderBy: string[] = [];
  private _limitVal: number | null = null;
  private _offsetVal: number | null = null;
  private _columns: string[] = ["*"];

  from(table: string): this {
    this._table = table;
    return this;
  }

  select(...columns: string[]): this {
    this._columns = columns;
    return this;
  }

  where(condition: string): this {
    this._conditions.push(condition);
    return this;
  }

  orderBy(column: string, direction: "ASC" | "DESC" = "ASC"): this {
    this._orderBy.push(`${column} ${direction}`);
    return this;
  }

  limit(n: number): this {
    this._limitVal = n;
    return this;
  }

  offset(n: number): this {
    this._offsetVal = n;
    return this;
  }

  build(): string {
    const parts: string[] = [
      `SELECT ${this._columns.join(", ")} FROM ${this._table}`,
    ];
    if (this._conditions.length > 0) {
      parts.push(`WHERE ${this._conditions.join(" AND ")}`);
    }
    if (this._orderBy.length > 0) {
      parts.push(`ORDER BY ${this._orderBy.join(", ")}`);
    }
    if (this._limitVal !== null)  parts.push(`LIMIT ${this._limitVal}`);
    if (this._offsetVal !== null) parts.push(`OFFSET ${this._offsetVal}`);
    return parts.join(" ");
  }
}

// Subclass adds domain-specific methods — return type 'this' means they get UserSelectBuilder back:
class UserSelectBuilder extends SelectBuilder {
  onlyActive(): this {
    return this.where("deleted_at IS NULL");
  }

  withRole(role: "admin" | "editor" | "viewer"): this {
    return this.where(`role = '${role}'`);
  }

  createdAfter(date: Date): this {
    return this.where(`created_at > '${date.toISOString()}'`);
  }
}

const query = new UserSelectBuilder()
  .from("users")
  .select("id", "name", "email", "role")
  .onlyActive()           // ✅ UserSelectBuilder method — works because return type is 'this'
  .withRole("admin")      // ✅
  .createdAfter(new Date("2025-01-01"))
  .orderBy("name")
  .limit(50)
  .offset(0)
  .build();

console.log(query);
// SELECT id, name, email, role FROM users
// WHERE deleted_at IS NULL AND role = 'admin' AND created_at > '...'
// ORDER BY name ASC LIMIT 50 OFFSET 0
```

---

## Example 2 — real world backend use case

```ts
// Typed event emitter using 'this' parameter and arrow functions for safe callback binding

class TypedEventEmitter<TEvents extends Record<string, unknown>> {
  private listeners = new Map<keyof TEvents, Array<(payload: unknown) => void>>();

  // Arrow function as method — 'this' is always the class instance:
  on = <K extends keyof TEvents>(
    event: K,
    listener: (payload: TEvents[K]) => void,
  ): this => {
    const existing = this.listeners.get(event) ?? [];
    this.listeners.set(event, [...existing, listener as (p: unknown) => void]);
    return this;  // fluent — return this for chaining
  };

  off = <K extends keyof TEvents>(
    event: K,
    listener: (payload: TEvents[K]) => void,
  ): this => {
    const existing = this.listeners.get(event) ?? [];
    this.listeners.set(event, existing.filter(l => l !== listener));
    return this;
  };

  emit = <K extends keyof TEvents>(event: K, payload: TEvents[K]): this => {
    const handlers = this.listeners.get(event) ?? [];
    handlers.forEach(handler => handler(payload));
    return this;
  };
}

// Domain events:
type AppEvents = {
  "user.created":  { userId: number; email: string };
  "user.deleted":  { userId: number };
  "order.placed":  { orderId: number; total: number; userId: number };
  "payment.failed": { orderId: number; reason: string };
};

const emitter = new TypedEventEmitter<AppEvents>();

// Safe: arrow functions as methods — on/emit/off can be safely destructured:
const { on, emit } = emitter;

on("user.created", ({ userId, email }) => {
  console.log(`New user: ${email} (id=${userId})`);
});

on("order.placed", ({ orderId, total }) => {
  console.log(`Order ${orderId} placed for $${(total / 100).toFixed(2)}`);
});

// ✅ Fluent chaining:
emitter
  .on("payment.failed", ({ orderId, reason }) => {
    console.error(`Payment failed for order ${orderId}: ${reason}`);
  })
  .emit("user.created", { userId: 1, email: "p@dev.io" })
  .emit("order.placed",  { orderId: 42, total: 9900, userId: 1 });

// ❌ Wrong payload — TypeScript catches it:
emitter.emit("user.created", { userId: 1 });  // missing 'email'
emitter.emit("user.created", { userId: 1, email: "p@dev.io", extra: true }); // extra field
```

---

## Common mistakes

### Mistake 1 — Losing `this` by passing a method as a callback

```ts
class UserService {
  private prefix = "[UserService]";

  // Regular method — 'this' depends on how it's called:
  logUser(user: User): void {
    console.log(`${this.prefix} ${user.name}`);  // this.prefix WILL be undefined if detached
  }

  // Arrow method — 'this' is always the class instance:
  logUserSafe = (user: User): void => {
    console.log(`${this.prefix} ${user.name}`);  // always safe
  };
}

const service = new UserService();
const users: User[] = [];

// ❌ DANGEROUS — passing a regular method as a callback:
users.forEach(service.logUser);
// Error (with noImplicitThis) or silent undefined this at runtime

// ✅ SAFE — arrow method:
users.forEach(service.logUserSafe);  // this is always the class instance

// ✅ ALSO SAFE — bind explicitly:
users.forEach(service.logUser.bind(service));
```

### Mistake 2 — Using regular methods in fluent builders and losing the subclass type

```ts
class Base {
  setValue(v: string): Base {   // ❌ returns Base — not this
    return this;
  }
}

class Child extends Base {
  setExtra(v: string): Child {
    return this;
  }
}

const child = new Child();
child.setValue("x").setExtra("y");
//             ↑ returns Base — .setExtra doesn't exist on Base!
// ❌ Error: Property 'setExtra' does not exist on type 'Base'

// ✅ FIX — return type should be 'this':
class Base {
  setValue(v: string): this {   // ✅ returns the actual subclass instance
    return this;
  }
}

child.setValue("x").setExtra("y");  // ✅ TypeScript knows .setValue returns Child
```

### Mistake 3 — Putting `this` annotation on an arrow function

```ts
// ❌ Arrow functions cannot have a 'this' parameter:
const handler = (this: Request, res: Response): void => {
  console.log(this.method);
};
// Error: An arrow function cannot have a 'this' parameter

// ✅ Use a regular function when you need a 'this' annotation:
const handler = function(this: Request, res: Response): void {
  console.log(this.method);
};

// ✅ Or use an arrow function inside a class (captures 'this' lexically):
class Router {
  handle = (req: Request, res: Response): void => {
    // this = Router instance — no annotation needed
  };
}
```

---

## Practice exercises

### Exercise 1 — easy

1. Write a function `formatRecord(this: { id: number; name: string; createdAt: Date }): string` that returns `"[1] Parsh — created 2026-01-01"`. Call it with `.call()` on two different objects.

2. Write a `Counter` class with:
   - `count: number = 0`
   - `increment(): this` — increments and returns `this`
   - `decrement(): this`
   - `reset(): this`
   - `getValue(): number`
   
   Verify fluent chaining works: `counter.increment().increment().decrement().getValue()` → `1`

3. Write a `greet` function with `this: void` to prove it can't use `this`. Show the TypeScript error.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed fluent HTTP request builder:

```ts
class HttpRequestBuilder {
  // private fields: url, method, headers, body, timeout, retries
  
  // Methods (all return 'this'):
  get(url: string): this
  post(url: string): this
  put(url: string): this
  delete(url: string): this
  withHeader(key: string, value: string): this
  withBearerToken(token: string): this   // adds Authorization: Bearer <token>
  withBody(data: unknown): this
  withTimeout(ms: number): this
  withRetries(n: number): this
  
  // Terminal method (does not return this):
  async execute<T>(): Promise<{ status: number; data: T; headers: Record<string, string> }>
}
```

Implement it so all chainable methods return `this` correctly. The `execute` stub can return a fake response.

Then write a subclass `AuthenticatedRequestBuilder extends HttpRequestBuilder` that adds:
- `withApiKey(key: string): this` — adds an `x-api-key` header
- `asAdmin(): this` — adds an `x-admin: true` header

Demonstrate: create an `AuthenticatedRequestBuilder`, chain all methods including both subclass methods, call `execute<User>()`, and verify the result is typed as `{ status: number; data: User; headers: ... }`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed mixin/trait system using `ThisType<T>`. Mixins are objects containing methods that rely on `this` being a specific shape — they are later merged onto a target object.

1. Define a `Serializable` mixin:
   ```ts
   type SerializableMethods = { serialize(): string; deserialize(json: string): void };
   // 'this' inside these methods should have access to any object properties
   ```

2. Define a `Validatable` mixin:
   ```ts
   type ValidatableMethods = { validate(): string[]; isValid(): boolean };
   // validate() returns array of error messages; isValid() returns validate().length === 0
   ```

3. Define a `Timestampable` mixin:
   ```ts
   type TimestampableMethods = { touch(): void; getAge(): number };
   // touch() updates updatedAt to now; getAge() returns ms since createdAt
   // 'this' must have: createdAt: Date, updatedAt: Date
   ```

4. Write a `mixIn<T extends object, M>(target: T, methods: M & ThisType<T & M>): T & M` function.

5. Create a `UserRecord` object (plain object, not a class):
   ```ts
   const userRecord = { id: 1, name: "Parsh", email: "p@dev.io", createdAt: new Date(), updatedAt: new Date() }
   ```

6. Apply all three mixins to it using `mixIn`. The result should have all original properties plus all mixin methods, with `this` correctly typed inside each method.

7. Call `userRecord.serialize()`, `userRecord.validate()`, `userRecord.touch()`, `userRecord.getAge()` and verify TypeScript fully types all of them.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Syntax | Meaning |
|--------|---------|
| `fn(this: T, ...)` | `this` must be of type `T` when calling `fn` |
| `this: void` | Function must not use `this` at all |
| `return this` | Return the current instance (type: `this` = actual subclass) |
| Arrow method `= () => {}` | Captures `this` lexically — always the class instance |
| `ThisType<T>` | Sets the type of `this` inside an object literal's methods |
| `fn.call(obj, args)` | Call `fn` with `obj` as `this` |
| `fn.bind(obj)` | Returns a new function with `this` permanently bound to `obj` |

| Rule | Notes |
|------|-------|
| Regular method `this` = caller's context | Detaching a method breaks `this` |
| Arrow method `this` = class instance | Always safe to pass as callback |
| `this` parameter is erased in JS output | Compile-time only — zero runtime cost |
| `return this` in fluent builders | Use the literal type `this` not the class name — enables correct subclass chaining |
| `noImplicitThis` in tsconfig | Forces you to annotate `this` when it would otherwise be `any` |
| Arrow functions + `this` param | Illegal — arrow functions cannot declare a `this` parameter |

## Connected topics

- **20 — Function types** — `this` is part of the function type in TypeScript.
- **33 — Classes in TypeScript** — where `this` is most commonly used: class methods, arrow methods.
- **34 — Abstract classes** — `return this` in abstract base classes for fluent builder hierarchies.
- **19 — Implementing interfaces** — when a class implements an interface, method `this` types must be compatible.
- **32 — Utility types** — `ThisParameterType<T>` and `OmitThisParameter<T>` — utility types specifically for `this`.
