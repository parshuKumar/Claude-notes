# 30 — Generic Constraints

## What is this?

A **generic constraint** is a restriction placed on a type parameter using the `extends` keyword. Without a constraint, `T` is completely unknown — TypeScript treats it like `unknown`, and you can't access any properties on it. Adding `T extends SomeShape` tells TypeScript that `T` must have at least that shape, unlocking property access, method calls, and indexed access (`T[K]`) inside the generic function, class, or interface.

Constraints are not narrowing — they don't shrink `T` down to the constraint type. They are **minimum requirements**: `T` must have at least what the constraint specifies, and the full original type is still preserved everywhere `T` appears.

## Why does it matter?

Constraints are what make generics *useful* in real code rather than just theoretical. Every real-world generic function needs to access something on its inputs:

- A sort function needs `T` to be comparable.
- A repository needs `T` to have an `id`.
- A property-picker needs `K` to be a key of `T`.
- A serializer needs `T` to have a `toJSON()` method.

Without constraints you'd have to cast with `as any` at every access point — which completely defeats generics. With constraints, TypeScript knows exactly what operations are safe.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — no constraints, no type checking:
function sortByField(items, field) {
  return [...items].sort((a, b) => {
    if (a[field] < b[field]) return -1;
    if (a[field] > b[field]) return 1;
    return 0;
  });
}

// Works, but no protection:
sortByField([{ name: "Parsh" }], "nonExistentField"); // silent wrong behavior
sortByField("not an array", "name"); // runtime crash
```

```ts
// TypeScript with constraints — safe and fully typed:
function sortByField<TItem, TKey extends keyof TItem>(
  items: TItem[],                    // TItem must be an array
  field: TKey,                       // field must be an actual key of TItem
): TItem[] {
  return [...items].sort((a, b) => {
    const av = a[field];            // TItem[TKey] — exactly typed
    const bv = b[field];
    if (av < bv) return -1;
    if (av > bv) return 1;
    return 0;
  });
}

interface User { id: number; name: string; email: string; }
const users: User[] = [{ id: 2, name: "Asha", email: "a@dev.io" }, { id: 1, name: "Parsh", email: "p@dev.io" }];

sortByField(users, "name");   // ✅ TItem = User, TKey = "name"
sortByField(users, "id");     // ✅ TKey = "id"
sortByField(users, "password"); // ❌ Error: "password" is not keyof User
sortByField("not array", "name"); // ❌ Error: string is not TItem[]
```

---

## Syntax

```ts
// ── T must have a specific shape ──────────────────────────────────────────
function logId<T extends { id: number }>(entity: T): void {
  console.log(entity.id);
}

// ── T must be a specific primitive type ──────────────────────────────────
function add<T extends number | bigint>(a: T, b: T): T {
  return (Number(a) + Number(b)) as T;  // simplified
}

// ── T must extend another generic type ───────────────────────────────────
function firstItem<T extends Array<unknown>>(arr: T): T[0] | undefined {
  return arr[0];
}

// ── K must be a key of T ──────────────────────────────────────────────────
function getField<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// ── K must be a key of T, and T[K] must be a specific type ───────────────
function getStringField<T, K extends keyof T>(
  obj: T,
  key: K & { [P in K]: T[P] extends string ? P : never }[K],
): string {
  return obj[key] as string;
}

// ── Multiple constraints with intersection ────────────────────────────────
function save<T extends { id: number } & { updatedAt: Date }>(entity: T): T {
  entity.updatedAt = new Date();
  return entity;
}

// ── Constraint on one param relative to another ───────────────────────────
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  for (const key of keys) result[key] = obj[key];
  return result;
}

// ── Constructor constraint — T must be newable ────────────────────────────
function createInstance<T>(Constructor: new (...args: unknown[]) => T): T {
  return new Constructor();
}
```

---

## How it works — concept by concept

### Rule 1 — Without a constraint, `T` is `unknown`

When there is no constraint, TypeScript treats `T` as completely unknown. You cannot access any property, call any method, or use any operator on a value of type `T`:

```ts
function process<T>(value: T): void {
  console.log(value.id);       // ❌ Error: Property 'id' does not exist on type 'T'
  value.toString();            // ❌ Error: 'T' is possibly 'null' or 'undefined'
  const n = value + 1;        // ❌ Error: operator '+' cannot be applied to types 'T' and 'number'
}

// Fix: constrain T to what you need:
function process<T extends { id: number; toString(): string }>(value: T): void {
  console.log(value.id);       // ✅
  value.toString();            // ✅
}
```

### Rule 2 — Constraints are minimum requirements, not type narrowing

`T extends X` means T is *at least* X — but `T` still carries its full original type. The constraint only *unlocks* access to what X has. The full type is preserved in return values and elsewhere:

```ts
function enrich<T extends { id: number }>(entity: T): T & { fetchedAt: Date } {
  return { ...entity, fetchedAt: new Date() };
}

// Passes { id: 1, name: "Parsh", email: "p@dev.io", role: "admin" }
// T = { id: number; name: string; email: string; role: string }
// Return type = { id: number; name: string; email: string; role: string; fetchedAt: Date }
// ALL original properties are preserved — the constraint didn't strip name/email/role

const enriched = enrich({ id: 1, name: "Parsh", email: "p@dev.io", role: "admin" });
enriched.name;      // ✅ string — preserved
enriched.email;     // ✅ string — preserved
enriched.role;      // ✅ string — preserved
enriched.fetchedAt; // ✅ Date  — added by enrich
enriched.id;        // ✅ number — from constraint
```

### Rule 3 — `keyof T` constraint for property access

`K extends keyof T` guarantees that `K` is a valid property name of `T`. Combined with `T[K]`, you get the exact value type for that property — no `any`, no `unknown`:

```ts
function getField<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];  // TypeScript knows: obj[key] is T[K] — exact type
}

interface RequestConfig {
  timeout: number;
  retries: number;
  baseUrl: string;
  authToken: string;
}

const config: RequestConfig = { timeout: 5000, retries: 3, baseUrl: "https://api.dev.io", authToken: "Bearer xyz" };

const timeout   = getField(config, "timeout");    // timeout: number
const baseUrl   = getField(config, "baseUrl");    // baseUrl: string
const authToken = getField(config, "authToken");  // authToken: string

getField(config, "method"); // ❌ Error: "method" is not keyof RequestConfig

// TypeScript infers both T and K from the arguments:
// T = RequestConfig, K = "timeout" → return type = number
// T = RequestConfig, K = "baseUrl" → return type = string
```

### Rule 4 — Constraint propagation across related parameters

When multiple type parameters are related, constraining one relative to another creates powerful typed accessor patterns:

```ts
// K depends on T — T determines what keys K can be:
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map(item => item[key]);
}

// Both T and K are inferred from the arguments:
const users: User[] = [...];
const names  = pluck(users, "name");   // T = User, K = "name"  → string[]
const ids    = pluck(users, "id");     // T = User, K = "id"    → number[]
const emails = pluck(users, "email");  // T = User, K = "email" → string[]
pluck(users, "password");             // ❌ Error — "password" not in User
```

### Rule 5 — Intersection constraints (`T extends A & B`)

You can require `T` to satisfy multiple shapes simultaneously:

```ts
interface HasId       { id: number; }
interface HasTimestamps { createdAt: Date; updatedAt: Date; }
interface HasSoftDelete { deletedAt: Date | null; }

// T must have id AND timestamps:
function touch<T extends HasId & HasTimestamps>(entity: T): T {
  (entity as HasTimestamps).updatedAt = new Date();
  return entity;
}

// T must have all three:
function softDelete<T extends HasId & HasTimestamps & HasSoftDelete>(entity: T): T {
  entity.deletedAt  = new Date();
  entity.updatedAt  = new Date();
  return entity;
}

interface User extends HasId, HasTimestamps, HasSoftDelete {
  name: string;
  email: string;
  role: string;
}

const user: User = { id: 1, name: "Parsh", email: "p@dev.io", role: "admin", createdAt: new Date(), updatedAt: new Date(), deletedAt: null };

touch(user);       // ✅ User has id + timestamps
softDelete(user);  // ✅ User has id + timestamps + softDelete
```

### Rule 6 — Constraining to a union type

```ts
// T must be one of the allowed primitive types:
function serialize<T extends string | number | boolean | null>(value: T): string {
  return JSON.stringify(value);
}

serialize("hello");   // ✅ T = string
serialize(42);        // ✅ T = number
serialize(true);      // ✅ T = boolean
serialize(null);      // ✅ T = null
serialize({ id: 1 }); // ❌ Error: object is not assignable to string|number|boolean|null

// T must be an HTTP method string literal:
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

function makeRequest<T extends HttpMethod>(method: T, url: string): void {
  console.log(`${method} ${url}`);
}

makeRequest("GET", "/api/users");    // ✅ T = "GET"
makeRequest("POST", "/api/users");   // ✅ T = "POST"
makeRequest("CONNECT", "/api");      // ❌ Error: "CONNECT" is not assignable to HttpMethod
```

### Rule 7 — Constructor constraint (`new()`)

When you want to pass a class constructor as a value and call `new` on it:

```ts
// T is the instance type; Constructor must be callable with new and produce a T:
function createAndInit<T>(
  Constructor: new () => T,
  init: (instance: T) => void,
): T {
  const instance = new Constructor();  // ✅ TypeScript knows this produces T
  init(instance);
  return instance;
}

class UserService {
  name = "UserService";
  initialize(): void { console.log(`${this.name} initialized`); }
}

const service = createAndInit(UserService, s => s.initialize());
// service: UserService — fully typed

// With constructor arguments:
function instantiate<TArgs extends unknown[], TInstance>(
  Constructor: new (...args: TArgs) => TInstance,
  ...args: TArgs
): TInstance {
  return new Constructor(...args);
}
```

---

## Example 1 — basic

```ts
// Typed object utilities using constraints

// 1. deepGet — access a nested property safely with full type inference:
function deepGet<
  TObj,
  TK1 extends keyof TObj,
  TK2 extends keyof TObj[TK1],
>(obj: TObj, k1: TK1, k2: TK2): TObj[TK1][TK2] {
  return obj[k1][k2];
}

interface Config {
  database: { host: string; port: number; name: string };
  server:   { port: number; host: string; timeout: number };
  auth:     { secret: string; expiresIn: number };
}

const config: Config = {
  database: { host: "localhost", port: 5432, name: "appdb" },
  server:   { port: 3000, host: "0.0.0.0", timeout: 30000 },
  auth:     { secret: "s3cr3t", expiresIn: 86400 },
};

const dbHost    = deepGet(config, "database", "host");   // string
const authExp   = deepGet(config, "auth", "expiresIn");  // number
const srvPort   = deepGet(config, "server", "port");     // number

deepGet(config, "database", "password"); // ❌ Error — "password" not in database

// 2. setField — immutably update one field, return a new object with the field changed:
function setField<T, K extends keyof T>(obj: T, key: K, value: T[K]): T {
  return { ...obj, [key]: value };
}

interface UserProfile {
  id: number;
  name: string;
  email: string;
  role: "admin" | "editor" | "viewer";
}

const profile: UserProfile = { id: 1, name: "Parsh", email: "p@dev.io", role: "admin" };

const renamed  = setField(profile, "name", "Dev");              // ✅ T[K] = string
const demoted  = setField(profile, "role", "editor");          // ✅ T[K] = "admin"|"editor"|"viewer"
const badEmail = setField(profile, "email", 42);               // ❌ Error: 42 not assignable to string
const badKey   = setField(profile, "password", "x");           // ❌ Error: "password" not keyof UserProfile

// 3. filterByField — filter an array where a specific field matches a value:
function filterByField<T, K extends keyof T>(
  items: T[],
  key: K,
  value: T[K],
): T[] {
  return items.filter(item => item[key] === value);
}

const users: UserProfile[] = [
  { id: 1, name: "Parsh", email: "p@dev.io", role: "admin" },
  { id: 2, name: "Dev",   email: "d@dev.io", role: "editor" },
  { id: 3, name: "Asha",  email: "a@dev.io", role: "viewer" },
];

const admins  = filterByField(users, "role", "admin");   // ✅ UserProfile[]
const byEmail = filterByField(users, "email", "p@dev.io");// ✅ UserProfile[]
filterByField(users, "role", "superuser");               // ❌ "superuser" not in "admin"|"editor"|"viewer"
filterByField(users, "id", "not a number");              // ❌ "not a number" not assignable to number
```

---

## Example 2 — real world backend use case

```ts
// Type-safe validation and transformation pipeline using constraints

// ── Constraint: entity must be serializable to JSON ───────────────────────
interface JsonSerializable {
  toJSON(): Record<string, unknown>;
}

function serializeForCache<T extends JsonSerializable>(entity: T): string {
  return JSON.stringify(entity.toJSON());
}

// ── Constraint: entity must have id + timestamps (can be saved to DB) ─────
interface DbEntity {
  id: number;
  createdAt: Date;
  updatedAt: Date;
}

// Generic "upsert" — works for any entity type:
async function upsert<T extends DbEntity>(
  tableName: string,
  entity: T,
  conflictColumn: keyof T & string,
): Promise<T> {
  const now = new Date();
  const toSave: T = { ...entity, updatedAt: now };

  // Build the SQL — we know conflictColumn is a key of T:
  const columns = Object.keys(toSave).join(", ");
  const values  = Object.values(toSave).map(() => "?").join(", ");
  const sql = `INSERT INTO ${tableName} (${columns}) VALUES (${values})
               ON CONFLICT (${conflictColumn}) DO UPDATE SET updatedAt = ?`;

  console.log(sql);  // Stub — in production you'd execute this
  return toSave;
}

// ── Constraint: T must be comparable (for sorting/min/max) ────────────────
type Comparable = number | string | Date | bigint;

function minBy<T, TKey extends keyof T>(
  items: T[],
  key: TKey & { [K in TKey]: T[K] extends Comparable ? K : never }[TKey],
): T | undefined {
  if (items.length === 0) return undefined;
  return items.reduce((min, item) =>
    (item[key] as unknown as number) < (min[key] as unknown as number) ? item : min,
  );
}

function maxBy<T, TKey extends keyof T>(
  items: T[],
  key: TKey & { [K in TKey]: T[K] extends Comparable ? K : never }[TKey],
): T | undefined {
  if (items.length === 0) return undefined;
  return items.reduce((max, item) =>
    (item[key] as unknown as number) > (max[key] as unknown as number) ? item : max,
  );
}

// ── Constraint: validator that only accepts objects (not primitives) ───────
function validateRequiredFields<T extends Record<string, unknown>>(
  obj: T,
  requiredFields: Array<keyof T>,
): { valid: boolean; missingFields: string[] } {
  const missingFields: string[] = [];

  for (const field of requiredFields) {
    const value = obj[field];
    if (value === null || value === undefined || value === "") {
      missingFields.push(String(field));
    }
  }

  return { valid: missingFields.length === 0, missingFields };
}

// ── Constraint: merge partial updates — only allow known keys ─────────────
function applyUpdate<T extends DbEntity>(
  current: T,
  updates: Partial<Omit<T, "id" | "createdAt">>,
): T {
  return {
    ...current,
    ...updates,
    updatedAt: new Date(),
  };
}

// ── Usage ──────────────────────────────────────────────────────────────────

interface OrderItem {
  productId: number;
  quantity: number;
  unitPriceCents: number;
}

interface Order extends DbEntity {
  userId: number;
  status: "pending" | "processing" | "shipped" | "delivered" | "cancelled";
  items: OrderItem[];
  totalCents: number;
  shippedAt: Date | null;
}

const orders: Order[] = [
  { id: 1, userId: 1, status: "pending",    items: [], totalCents: 4999,  shippedAt: null,           createdAt: new Date("2026-01-01"), updatedAt: new Date("2026-01-01") },
  { id: 2, userId: 1, status: "shipped",    items: [], totalCents: 19900, shippedAt: new Date("2026-01-05"), createdAt: new Date("2026-01-02"), updatedAt: new Date("2026-01-05") },
  { id: 3, userId: 2, status: "delivered",  items: [], totalCents: 2500,  shippedAt: new Date("2026-01-03"), createdAt: new Date("2026-01-02"), updatedAt: new Date("2026-01-06") },
];

// All fully typed — return types inferred from constraints:
const cheapest = minBy(orders, "totalCents");      // Order | undefined
cheapest?.status;                                   // ✅ "pending"|"processing"|...
cheapest?.totalCents;                               // ✅ number

const newest = maxBy(orders, "createdAt");          // Order | undefined
newest?.userId;                                     // ✅ number

const validation = validateRequiredFields(
  { userId: 1, status: "pending", items: [], totalCents: 0 },
  ["userId", "status", "items", "totalCents"],      // ✅ all are keys of the object
);
// validation.valid: boolean, validation.missingFields: string[]

const currentOrder = orders[0];
const updatedOrder = applyUpdate(currentOrder, {    // Partial<Omit<Order,"id"|"createdAt">>
  status: "processing",
  totalCents: 5999,
});
// updatedOrder: Order — id and createdAt preserved, updatedAt refreshed
updatedOrder.status;     // ✅ "processing"
updatedOrder.id;         // ✅ 1 — unchanged
updatedOrder.updatedAt;  // ✅ Date — refreshed

applyUpdate(currentOrder, { id: 99 });              // ❌ Error: "id" is excluded from updates
applyUpdate(currentOrder, { nonExistent: true });   // ❌ Error: not a key of Order
```

---

## Common mistakes

### Mistake 1 — Over-constraining, losing genericity

```ts
// ❌ The constraint is the same as the full type — T is not adding value:
function processUser<T extends User>(user: T): T {
  console.log(user.name);
  return user;
}
// T is always User here — just write (user: User): User

// ✅ Keep generics when T can be a subtype of the constraint:
function processEntity<T extends { id: number; name: string }>(entity: T): T & { processed: true } {
  return { ...entity, processed: true };
}
// Now T can be User, Admin, Post — anything with id and name
// AND the return type preserves all extra properties of T
```

### Mistake 2 — Using `as any` instead of proper constraint

```ts
// ❌ Casting defeats the purpose of generics:
function sumField<T>(items: T[], field: string): number {
  return items.reduce((sum, item) => sum + (item as any)[field], 0);
}
// No safety — field could be non-numeric, or not even a key of T

// ✅ Constrain field to numeric keys of T:
function sumField<T, K extends keyof T>(
  items: T[],
  field: K & { [P in K]: T[P] extends number ? P : never }[K],
): number {
  return items.reduce((sum, item) => sum + (item[field] as unknown as number), 0);
}

// Or simpler — separate the number constraint:
function sumNumericField<T extends Record<K, number>, K extends keyof T>(
  items: T[],
  field: K,
): number {
  return items.reduce((sum, item) => sum + item[field], 0);
}

interface OrderLine { productId: number; quantity: number; unitPriceCents: number; description: string; }
const lines: OrderLine[] = [{ productId: 1, quantity: 2, unitPriceCents: 500, description: "Widget" }];

sumNumericField(lines, "quantity");       // ✅ K = "quantity", T[K] = number
sumNumericField(lines, "unitPriceCents"); // ✅ K = "unitPriceCents"
sumNumericField(lines, "description");   // ❌ Error: "description" maps to string, not number
```

### Mistake 3 — Forgetting that `keyof T` includes symbol and number keys

```ts
// keyof T is actually string | number | symbol — not just string:
function getStringKey<T, K extends keyof T>(obj: T, key: K): T[K] {
  // key could be a symbol — using it as a string would be wrong
  return obj[key];  // ✅ fine as long as you don't coerce key to string
}

// ❌ Dangerous coercion:
function logKey<T, K extends keyof T>(obj: T, key: K): void {
  console.log(`Key: ${String(key)}`);   // converts symbol → "Symbol(...)" — maybe not intended
}

// ✅ Narrow to string keys explicitly when you need string behavior:
function getStringKeyOnly<T, K extends keyof T & string>(obj: T, key: K): T[K] {
  console.log(`Key: ${key}`); // ✅ key is definitely a string
  return obj[key];
}
```

---

## Practice exercises

### Exercise 1 — easy

1. Write `getNestedField<T, K1 extends keyof T, K2 extends keyof T[K1]>(obj: T, k1: K1, k2: K2): T[K1][K2]`. Test it on a nested config object with at least 3 different nested paths, verifying each return type is correct.

2. Write `hasField<T, K extends keyof T>(obj: T, key: K, value: T[K]): boolean` — returns `true` if `obj[key] === value`. TypeScript must reject a value argument whose type doesn't match `T[K]`.

3. Write `sortByKey<T, K extends keyof T>(items: T[], key: K, direction: "asc" | "desc"): T[]` that sorts an array by any key. Test it on `User[]` by `"name"` and `"id"`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a type-safe `Validator<T>` class using constraints:

```ts
class Validator<T extends Record<string, unknown>> {
  private rules: Array<(obj: T) => string | null> = [];

  // Add a rule that checks a specific field:
  required<K extends keyof T & string>(field: K): this

  // Add a rule that checks a field is a number in a range:
  range<K extends keyof T>(
    field: K & { [P in K]: T[P] extends number ? P : never }[K],
    min: number,
    max: number,
  ): this

  // Add a rule that checks a string field matches a regex:
  pattern<K extends keyof T>(
    field: K & { [P in K]: T[P] extends string ? P : never }[K],
    regex: RegExp,
    message: string,
  ): this

  // Add a custom rule:
  custom(rule: (obj: T) => string | null): this

  // Run all rules and return results:
  validate(obj: T): { valid: boolean; errors: string[] }
}
```

Use it to validate a `CreateOrderInput`:

```ts
interface CreateOrderInput {
  userId: number;
  items: Array<{ productId: number; quantity: number }>;
  totalCents: number;
  couponCode: string | null;
  deliveryAddress: string;
}

const validator = new Validator<CreateOrderInput>()
  .required("userId")
  .required("deliveryAddress")
  .range("totalCents", 1, 1_000_000_00)
  .pattern("deliveryAddress", /\d{5}/, "Delivery address must include a zip code")
  .custom(order => order.items.length === 0 ? "Order must have at least one item" : null);

const result = validator.validate({ userId: 0, items: [], totalCents: 0, couponCode: null, deliveryAddress: "123 Main St" });
// result.valid: false, result.errors: ["userId is required", "Order must have at least one item", ...]
```

TypeScript must reject calls like `.required("nonExistentField")` or `.range("deliveryAddress", 0, 100)` (string field used in numeric range rule).

```ts
// Write your code here
```

### Exercise 3 — hard

Build a type-safe generic `ORM<TSchema>` where `TSchema` is a map of table names to entity types. The ORM uses constraints to ensure every operation references a valid table and valid columns:

```ts
interface DbSchema {
  users:    { id: number; name: string; email: string; role: "admin" | "editor" | "viewer"; createdAt: Date };
  posts:    { id: number; title: string; body: string; authorId: number; published: boolean; createdAt: Date };
  sessions: { id: string; userId: number; token: string; expiresAt: Date; createdAt: Date };
}

class ORM<TSchema extends Record<string, Record<string, unknown>>> {
  // Find by id in a table:
  findById<TTable extends keyof TSchema>(
    table: TTable,
    id: TSchema[TTable] extends { id: unknown } ? TSchema[TTable]["id"] : never,
  ): Promise<TSchema[TTable] | null>

  // Find all matching a filter:
  findWhere<TTable extends keyof TSchema>(
    table: TTable,
    filter: Partial<TSchema[TTable]>,
  ): Promise<TSchema[TTable][]>

  // Insert a record (id is auto-generated):
  insert<TTable extends keyof TSchema>(
    table: TTable,
    data: Omit<TSchema[TTable], "id" | "createdAt">,
  ): Promise<TSchema[TTable]>

  // Update specific fields:
  update<TTable extends keyof TSchema>(
    table: TTable,
    id: TSchema[TTable] extends { id: unknown } ? TSchema[TTable]["id"] : never,
    changes: Partial<Omit<TSchema[TTable], "id" | "createdAt">>,
  ): Promise<TSchema[TTable] | null>

  // Pluck a single column from matching rows:
  pluck<TTable extends keyof TSchema, TCol extends keyof TSchema[TTable]>(
    table: TTable,
    filter: Partial<TSchema[TTable]>,
    column: TCol,
  ): Promise<TSchema[TTable][TCol][]>
}

// Usage — TypeScript must enforce all constraints:
const orm = new ORM<DbSchema>();

const user = await orm.findById("users", 1);      // TSchema["users"] | null
user?.name;                                        // ✅ string
user?.published;                                   // ❌ not on users

const posts = await orm.findWhere("posts", { authorId: 1, published: true });
// posts: DbSchema["posts"][]

await orm.insert("users", { name: "Parsh", email: "p@dev.io", role: "admin" });
// ✅ id and createdAt excluded — correct

await orm.insert("sessions", { userId: 1, token: "tok_abc", expiresAt: new Date() });
// ✅ id (string here) and createdAt excluded

const emails = await orm.pluck("users", { role: "admin" }, "email");
// emails: string[]

await orm.pluck("posts", {}, "password");  // ❌ Error: "password" not in posts
```

Implement the ORM with an in-memory store backed by `Map<string, Map<unknown, unknown>>`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// Shape constraint:
function fn<T extends { id: number }>(x: T): T { return x; }

// keyof constraint — K must be a key of T:
function get<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }

// String-only keys:
function get<T, K extends keyof T & string>(obj: T, key: K): T[K] { return obj[key]; }

// Intersection constraint — T must satisfy multiple shapes:
function save<T extends HasId & HasTimestamps>(entity: T): T { return entity; }

// Union constraint — T must be one of these types:
function serialize<T extends string | number | boolean>(val: T): string { return String(val); }

// Constructor constraint:
function make<T>(Ctor: new () => T): T { return new Ctor(); }

// Constraint relative to another param:
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> { ... }
```

| Pattern | Purpose |
|---------|---------|
| `T extends { id: number }` | Access `.id` on T safely |
| `K extends keyof T` | K must be a valid property name of T |
| `T[K]` | Exact type of property K on T |
| `T extends A & B` | T must satisfy both shapes |
| `T extends string \| number` | T must be one of these primitives |
| `K extends keyof T & string` | K is a string key (excludes symbols/numbers) |
| `new () => T` | Constructor constraint — T can be instantiated |
| `T extends Record<string, unknown>` | T must be an object type |

---

## Connected topics

- **26 — What are generics** — foundational concept; why constraints are needed.
- **27 — Generic functions** — where constraints are most commonly applied.
- **28 — Generic interfaces** — constrained type parameters on interface declarations.
- **29 — Generic classes** — constrained class type parameters.
- **40 — Conditional types** — `T extends U ? X : Y` — constraints used in type-level branching.
- **41 — Mapped types** — iterating `keyof T` with constraints to transform every property.
- **29 — Utility types (built-in)** — `Pick<T,K>`, `Omit<T,K>`, `Record<K,V>` — all built on `K extends keyof T` constraints.
