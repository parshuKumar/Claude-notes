# 13 — Arrays and Tuples

## What is this?

Arrays in TypeScript work exactly like JavaScript arrays — but you declare what type of elements they hold. A `number[]` can only contain numbers; a `string[]` can only contain strings. Tuples are a special TypeScript array type where the **length is fixed** and **each position has its own specific type** — `[string, number]` is always exactly two elements: first a string, then a number. TypeScript enforces both.

## Why does it matter?

In JavaScript, arrays are completely untyped — you can push anything into any array. In TypeScript, typed arrays prevent you from accidentally mixing types, accessing missing elements, or calling array methods that don't exist for the element type. Tuples solve a specific problem: when you return multiple values from a function (like `[data, error]`), TypeScript knows exactly what type each position holds, giving you destructuring safety.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — arrays accept anything
const userIds = [];
userIds.push(1);
userIds.push("hello");   // accidentally pushed a string — no error
userIds.push(null);      // no error — now your number logic will break

// Returning multiple values via array — positions are ambiguous
function getUser(id) {
  if (!id) return [null, "ID required"];
  return [{ id, name: "Parsh" }, null];
}

const [user, error] = getUser(1);
user.nme;      // typo — JS says undefined, no error
error.msg;     // typo — JS says undefined, no error
```

```ts
// TypeScript — arrays are typed
const userIds: number[] = [];
userIds.push(1);         // ✅
userIds.push("hello");   // ❌ Argument of type 'string' is not assignable to type 'number'
userIds.push(null);      // ❌

// Tuple — positions are typed individually
type UserResult = [{ id: number; name: string } | null, string | null];

function getUser(id: number): UserResult {
  if (!id) return [null, "ID required"];
  return [{ id, name: "Parsh" }, null];
}

const [user, error] = getUser(1);
if (user) {
  user.nme;   // ❌ Property 'nme' does not exist — caught immediately
}
```

---

## Syntax

```ts
// ── ARRAY SYNTAX — two equivalent ways ───────────────────────────────────
let userIds: number[] = [1, 2, 3];          // shorthand (preferred)
let userIds2: Array<number> = [1, 2, 3];    // generic syntax

// ── COMMON TYPED ARRAYS ───────────────────────────────────────────────────
let names: string[] = ["Parsh", "Alice"];
let flags: boolean[] = [true, false, true];
let dates: Date[] = [new Date(), new Date()];
let mixed: (string | number)[] = [1, "hello", 2, "world"];  // union element type

// ── ARRAY OF OBJECTS ──────────────────────────────────────────────────────
type User = { id: number; name: string };
let users: User[] = [
  { id: 1, name: "Parsh" },
  { id: 2, name: "Alice" },
];

// ── READONLY ARRAYS ───────────────────────────────────────────────────────
const permissions: readonly string[] = ["read", "write"];
const ids: ReadonlyArray<number> = [1, 2, 3];   // equivalent

// ── TUPLE SYNTAX ──────────────────────────────────────────────────────────
let coordinate: [number, number] = [40.7128, -74.0060];  // [latitude, longitude]
let entry: [string, number] = ["Parsh", 101];            // [name, id]
let csvRow: [string, string, number, boolean] = ["Parsh", "parsh@dev.io", 25, true];

// ── NAMED TUPLE ELEMENTS ──────────────────────────────────────────────────
type Point = [x: number, y: number, z: number];
type HttpResult = [statusCode: number, message: string, body: unknown];

// ── OPTIONAL TUPLE ELEMENTS ───────────────────────────────────────────────
type OptionalTuple = [string, number?];   // second element is optional
const t1: OptionalTuple = ["hello"];          // ✅
const t2: OptionalTuple = ["hello", 42];      // ✅

// ── REST ELEMENT IN TUPLE ─────────────────────────────────────────────────
type AtLeastOne = [string, ...number[]];  // first is string, rest are numbers
```

---

## How it works — arrays

### Typed array methods

Once you declare an array type, TypeScript knows what every array method returns:

```ts
const userIds: number[] = [3, 1, 4, 1, 5, 9];

// TypeScript knows the return type of every method:
const doubled: number[] = userIds.map(id => id * 2);      // (id: number) inferred
const big: number[] = userIds.filter(id => id > 3);       // returns number[]
const sum: number = userIds.reduce((acc, id) => acc + id, 0); // inferred as number
const first: number | undefined = userIds.find(id => id > 3); // number | undefined

// TypeScript catches mistakes in callbacks:
userIds.map(id => id.toUpperCase());   // ❌ toUpperCase doesn't exist on number
```

### Array of objects — accessing properties safely

```ts
type Product = {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
};

const products: Product[] = [
  { id: 1, name: "Keyboard", price: 99.99, inStock: true },
  { id: 2, name: "Mouse", price: 49.99, inStock: false },
];

// TypeScript knows every element is a Product:
const names = products.map(p => p.name);         // string[]
const inStock = products.filter(p => p.inStock); // Product[]
const prices = products.map(p => p.pice);        // ❌ typo — caught immediately
```

### Empty arrays must be annotated

```ts
// ❌ PROBLEM — TypeScript infers never[] for empty arrays
const userIds = [];            // inferred: never[]
userIds.push(1);               // ❌ Argument of type 'number' is not assignable to 'never'

// ✅ Annotate empty arrays explicitly:
const userIds: number[] = [];
userIds.push(1);   // ✅
```

### Readonly arrays

```ts
// readonly prevents mutation — safe for config, constants, etc.
const allowedRoles: readonly string[] = ["admin", "editor", "viewer"];

allowedRoles.push("superadmin");     // ❌ Property 'push' does not exist on readonly array
allowedRoles[0] = "root";            // ❌ Cannot assign to index on readonly array

// You can still read:
allowedRoles[0];                     // ✅ "admin"
allowedRoles.includes("admin");      // ✅
allowedRoles.find(r => r === "admin"); // ✅ map/filter/find are fine — they don't mutate
```

---

## How it works — tuples

### Position-specific types

```ts
// Each position has its own type:
type NameAge = [string, number];

const person: NameAge = ["Parsh", 25];   // ✅
const broken: NameAge = [25, "Parsh"];   // ❌ number at pos 0, string at pos 1 — wrong order

// Destructuring gives each variable the correct type:
const [name, age] = person;
name.toUpperCase();   // ✅ TypeScript knows name is string
age.toFixed(2);       // ✅ TypeScript knows age is number
name.toFixed(2);      // ❌ toFixed doesn't exist on string
```

### Tuples enforce length

```ts
type Pair = [string, number];

const pair: Pair = ["hello", 42];        // ✅ exactly 2 elements
const tooMany: Pair = ["hello", 42, "extra"]; // ❌ tuple has 2 elements, not 3
const tooFew: Pair = ["hello"];           // ❌ tuple has 2 elements, not 1
```

### Named tuple elements — for readability

```ts
// Without names — confusing which is which:
type Geo = [number, number];  // is it [lat, lng] or [lng, lat]?

// With names — self-documenting:
type Geo2 = [latitude: number, longitude: number];
const nyc: Geo2 = [40.7128, -74.0060];   // ✅

// Named elements don't change the destructuring — names are only for documentation
const [lat, lng] = nyc;   // lat is number, lng is number
```

---

## The `[data, error]` tuple pattern

This is a powerful pattern inspired by Go's error handling and used heavily in TypeScript:

```ts
// Return [result, error] from functions — like Go's (value, err) pattern
type Success<T> = [T, null];
type Failure = [null, Error];
type Result<T> = Success<T> | Failure;

async function fetchUser(userId: number): Promise<Result<User>> {
  try {
    const user = await db.findById(userId);
    if (!user) return [null, new Error(`User ${userId} not found`)];
    return [user, null];
  } catch (err) {
    return [null, err instanceof Error ? err : new Error("Unknown error")];
  }
}

// Caller destructures and handles both cases:
const [user, error] = await fetchUser(42);

if (error) {
  console.error(error.message);  // TypeScript knows error is Error
  return;
}

// TypeScript knows user is User here (not null):
console.log(user.email);   // ✅
```

---

## Arrays vs Tuples — when to use which

| Use case | Use |
|----------|-----|
| A list of items of the same type | `string[]`, `User[]` |
| A collection that grows/shrinks | `string[]` |
| Fixed number of values with different types | Tuple `[string, number]` |
| Return multiple values from a function | Tuple `[User | null, Error | null]` |
| CSV row representation | Tuple `[string, string, number, boolean]` |
| Coordinates, key-value pairs | Tuple `[number, number]`, `[string, unknown]` |

---

## `as const` with arrays — making tuple types from literals

```ts
// Without as const — inferred as string[]:
const methods = ["GET", "POST", "DELETE"];
// type: string[]

// With as const — inferred as readonly tuple with literal types:
const methods = ["GET", "POST", "DELETE"] as const;
// type: readonly ["GET", "POST", "DELETE"]

// Extract a union type from it:
type HttpMethod = typeof methods[number];   // "GET" | "POST" | "DELETE"
```

---

## Example 1 — basic

```ts
// All the array and tuple patterns together

// Typed arrays:
const userIds: number[] = [1, 2, 3, 4, 5];
const emailList: string[] = ["parsh@dev.io", "alice@dev.io"];
const activeFlags: boolean[] = [true, false, true];

// Array of typed objects:
type Tag = { id: number; label: string; color: string };
const tags: Tag[] = [
  { id: 1, label: "backend", color: "#3b82f6" },
  { id: 2, label: "typescript", color: "#8b5cf6" },
];

// Readonly config array:
const allowedOrigins: readonly string[] = [
  "https://myapp.com",
  "https://staging.myapp.com",
];

// Tuples for structured data:
type Coordinate = [latitude: number, longitude: number];
const officeLocation: Coordinate = [28.6139, 77.2090];   // Delhi

// Destructure with correct types:
const [lat, lng] = officeLocation;
console.log(`Latitude: ${lat.toFixed(4)}`);   // ✅ TypeScript knows lat is number

// Tuple from a function:
function divideWithRemainder(dividend: number, divisor: number): [quotient: number, remainder: number] {
  return [Math.floor(dividend / divisor), dividend % divisor];
}

const [quotient, remainder] = divideWithRemainder(17, 5);
console.log(`17 / 5 = ${quotient} remainder ${remainder}`);  // "17 / 5 = 3 remainder 2"
```

---

## Example 2 — real world backend use case

```ts
import { Request, Response } from 'express';

// ── Typed DB query results using arrays ───────────────────────────────────

type UserRow = {
  id: number;
  email: string;
  name: string;
  role: "admin" | "user";
  created_at: string;
};

// Simulate a typed DB query result:
async function queryUsers(page: number, pageSize: number): Promise<UserRow[]> {
  // In reality: const result = await pool.query<UserRow>(...)
  // result.rows is UserRow[]
  return [];
}

// ── Tuple for DB operation results ────────────────────────────────────────

type DbSuccess<T> = [data: T, error: null];
type DbError = [data: null, error: Error];
type DbResult<T> = DbSuccess<T> | DbError;

async function safeQuery<T>(
  queryFn: () => Promise<T>
): Promise<DbResult<T>> {
  try {
    const result = await queryFn();
    return [result, null];
  } catch (err) {
    return [null, err instanceof Error ? err : new Error(String(err))];
  }
}

// ── Route handler using the tuple pattern ─────────────────────────────────

async function getUsersHandler(req: Request, res: Response): Promise<void> {
  const page = parseInt(req.query.page as string ?? "1", 10);
  const pageSize = parseInt(req.query.pageSize as string ?? "20", 10);

  const [users, error] = await safeQuery(() => queryUsers(page, pageSize));

  if (error) {
    res.status(500).json({ success: false, error: error.message });
    return;
  }

  // TypeScript knows users is UserRow[] here
  const responseData = users.map(u => ({
    id: u.id,
    email: u.email,
    name: u.name,
    role: u.role,
  }));

  res.json({
    success: true,
    data: responseData,
    pagination: { page, pageSize, total: users.length },
  });
}

// ── Readonly config array ──────────────────────────────────────────────────

const CORS_ORIGINS: readonly string[] = [
  "https://myapp.com",
  "https://admin.myapp.com",
  "https://staging.myapp.com",
] as const;

function isCorsAllowed(origin: string): boolean {
  return CORS_ORIGINS.includes(origin);
}

// ── Typed CSV parsing with tuples ──────────────────────────────────────────

type UserCsvRow = [
  id: string,
  name: string,
  email: string,
  role: string,
  createdAt: string
];

function parseCsvRow(raw: string): UserCsvRow {
  const parts = raw.split(",") as UserCsvRow;
  if (parts.length !== 5) throw new Error(`Invalid CSV row: "${raw}"`);
  return parts;
}

function csvRowToUser(row: UserCsvRow): UserRow {
  const [id, name, email, role, createdAt] = row;
  return {
    id: parseInt(id, 10),
    name: name.trim(),
    email: email.trim().toLowerCase(),
    role: (role.trim() === "admin" ? "admin" : "user"),
    created_at: createdAt.trim(),
  };
}
```

---

## Common mistakes

### Mistake 1 — Accessing a tuple element by out-of-bounds index

```ts
// ❌ WRONG — tuple has 2 elements, accessing index 2
type Pair = [string, number];
const pair: Pair = ["hello", 42];

const third = pair[2];   // TypeScript: type is 'undefined'
// With noUncheckedIndexedAccess: true → TypeScript warns about this
// Without it → TypeScript gives type undefined but no error

// ✅ RIGHT — destructure explicitly:
const [first, second] = pair;   // TypeScript knows exact types
// Or access only valid indices:
const elem0: string = pair[0];  // ✅
const elem1: number = pair[1];  // ✅
```

### Mistake 2 — Using a tuple where you need an array (immutable length gotcha)

```ts
// ❌ PROBLEM — tuples have fixed length; push/pop break the contract
type NameAge = [string, number];

const nameAge: NameAge = ["Parsh", 25];
nameAge.push(100);   // TypeScript allows this (known TS limitation with push/pop)
                     // but conceptually wrong — you now have 3 elements in a 2-tuple

// ✅ RIGHT — use readonly tuple to prevent mutation:
type ReadonlyNameAge = readonly [string, number];
const nameAge2: ReadonlyNameAge = ["Parsh", 25];
nameAge2.push(100);  // ❌ Properly caught: push doesn't exist on readonly tuple
```

### Mistake 3 — Forgetting Array<T> and T[] are identical but one looks wrong

```ts
// Both are exactly the same type — just different syntax:
let ids: number[] = [1, 2, 3];
let ids2: Array<number> = [1, 2, 3];

// Common mistake: using Array<T> with union types without parentheses:
let items: Array<string | number> = [1, "a", 2, "b"];  // ✅ clear
let items2: string | number[] = [1, "a"];               // ❌ WRONG! This is: string OR (number[])
                                                         // NOT (string | number)[]
// ✅ Correct shorthand for union element arrays:
let items3: (string | number)[] = [1, "a", 2, "b"];     // parens required
```

---

## Practice exercises

### Exercise 1 — easy

Create the following typed arrays:
1. An empty `string[]` called `pendingEmails`, then push three email strings into it.
2. A `readonly string[]` called `httpMethods` containing `"GET"`, `"POST"`, `"PUT"`, `"DELETE"` — try pushing to it and observe the error.
3. An array of objects typed as `{ id: number; score: number }[]` called `leaderboard` with at least 3 entries.

Write a function `getTopScorers(scores: { id: number; score: number }[], topN: number): { id: number; score: number }[]` that returns the top N entries sorted by score descending.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed CSV processor. Define a tuple type for a user import row:
```ts
type UserImportRow = [id: string, name: string, email: string, role: string, plan: string]
```

Write three functions:
1. `parseRow(raw: string): UserImportRow` — splits on comma, validates it has exactly 5 fields (throw if not), returns the tuple.
2. `validateRow(row: UserImportRow): { valid: boolean; errors: string[] }` — destructure using named positions and validate: id must parse to a positive number, email must contain `@`, role must be `"admin"` or `"user"`, plan must be `"free"` or `"pro"` or `"enterprise"`.
3. `processRows(rawRows: string[]): { valid: UserImportRow[]; invalid: Array<[row: string, errors: string[]]> }` — processes each row through `parseRow` and `validateRow`, collecting valid and invalid separately.

```ts
// Write your code here
```

### Exercise 3 — hard

Implement the `[data, error]` tuple result pattern as a full utility system.

1. Define these types:
   ```ts
   type Ok<T> = [data: T, error: null]
   type Err<E extends Error = Error> = [data: null, error: E]
   type Result<T, E extends Error = Error> = Ok<T> | Err<E>
   ```

2. Write a generic `wrap<T>(fn: () => Promise<T>): Promise<Result<T>>` function that catches errors and wraps them in the tuple pattern.

3. Write a generic `wrapSync<T>(fn: () => T): Result<T>` for synchronous operations.

4. Using these utilities, write three functions:
   - `parseJson<T>(raw: string): Result<T>` — uses `wrapSync` around `JSON.parse`
   - `readEnvVar(key: string): Result<string, Error>` — returns `Ok` if the env var exists and is non-empty, `Err` otherwise
   - `validatePositiveNumber(value: unknown): Result<number>` — returns `Ok` if value is a positive number, `Err` otherwise

5. Demonstrate calling all three and correctly handling both the success and error branches using destructuring. TypeScript must narrow correctly in both branches — no `any`.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

| Pattern | Syntax |
|---------|--------|
| Number array | `number[]` or `Array<number>` |
| String array | `string[]` |
| Object array | `User[]` |
| Union element array | `(string \| number)[]` — parens required |
| Readonly array | `readonly string[]` or `ReadonlyArray<string>` |
| Empty array | `const arr: number[] = []` — must annotate |
| 2-element tuple | `[string, number]` |
| Named tuple | `[name: string, age: number]` |
| Optional tuple elem | `[string, number?]` |
| Readonly tuple | `readonly [string, number]` |
| Array of tuples | `[string, number][]` |
| Derive union from array | `typeof arr[number]` (needs `as const`) |

| Rule | Notes |
|------|-------|
| `T[]` vs `Array<T>` | Identical — use `T[]` shorthand |
| Union in array | Must use parens: `(A \| B)[]` not `A \| B[]` |
| Empty array | Always annotate — otherwise `never[]` |
| Tuple for multiple return | `[result, error]` pattern |
| Immutable arrays | `readonly T[]` or `as const` |
| Tuples are fixed length | Don't push into tuples — use `readonly` to enforce |

## Connected topics

- **09 — Union types** — union element arrays `(string | number)[]`.
- **11 — Literal types** — `as const` on arrays to get literal tuple types.
- **26 — Generics** — `Array<T>` is a generic type — generics explained fully there.
- **32 — Utility types** — `ReadonlyArray<T>` and other built-in array utilities.
