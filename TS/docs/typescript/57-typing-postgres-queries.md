# 57 — Typing database queries with pg (PostgreSQL)

## What is this?

**`node-postgres`** (the `pg` package) is the driver almost every Node backend uses to talk to PostgreSQL. Unlike an ORM, it has no schema, no models, and no migrations — you hand it a SQL string and an array of parameters, and it hands you back rows.

In JavaScript, those rows are `any`. In TypeScript, `pg` gives you a **generic** you can fill in:

```ts
import { Pool, QueryResult, QueryResultRow } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// The row shape you claim the query returns:
interface UserRow {
  id:            string;
  email:         string;
  password_hash: string;
  created_at:    Date;
}

// QueryResult<UserRow> — rows: UserRow[], rowCount: number | null, fields: FieldDef[]
const result: QueryResult<UserRow> = await pool.query<UserRow>(
  "SELECT id, email, password_hash, created_at FROM users WHERE email = $1",
  ["ada@example.com"],
);

const user = result.rows[0];   // UserRow | undefined (with noUncheckedIndexedAccess)
```

That is the whole API surface: `pool.query<T>(sql, params)` returns `Promise<QueryResult<T>>`, and `QueryResult<T>.rows` is `T[]`.

But here is the sentence that this entire document exists to make you feel in your bones:

> **`T` is an unchecked assertion. PostgreSQL never sees it. TypeScript never verifies it. It is exactly as trustworthy as `as UserRow`.**

Typing `pg` well is therefore not about "adding types" — it's about building a **narrow, deliberate boundary** where that assertion is made once, in as few places as possible, ideally backed by a runtime check.

## Why does it matter?

The `pg` boundary is the most dangerous seam in a typical Node backend, because it *looks* typed while being completely unverified.

- You write `SELECT id, email FROM users` but type the result as a row with `password_hash`. The compiler is happy. `user.password_hash` is `undefined` at runtime, and your `bcrypt.compare(password, undefined)` throws — or worse, your custom comparison returns `true`.
- Postgres columns are `snake_case`. Your domain objects are `camelCase`. You declare `createdAt: Date` on the row interface and get `undefined` forever, because the actual key is `created_at`.
- A `bigint` / `int8` column comes back as a **string** from `pg`, not a `number`. You typed it `number`, and now `totalCents + shippingCents` silently produces `"1200300"`.
- A `numeric` column also comes back as a string. Your revenue report is string concatenation.
- `rows[0]` is typed `UserRow` without `noUncheckedIndexedAccess`, so the "user not found" case looks impossible to the compiler and blows up in production.
- You add a column to the SQL but forget the interface — no error. You *remove* a column from the SQL but leave it in the interface — no error. The interface and the query drift apart permanently and silently.
- A migration renames `is_active` to `status`. Zero compile errors anywhere in your codebase. You find out from your users.

Typing `pg` correctly means: one generic `query<T>()` helper, one place per query where the assertion lives, explicit column lists, a mapper from `snake_case` row to `camelCase` domain object, and — for anything that matters — a zod parse at the boundary so the assertion becomes a *proof*.

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript — the query, the row shape, and the caller all drift apart ────
const { Pool } = require("pg");
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function findUserByEmail(email) {
  const result = await pool.query(
    "SELECT id, email, created_at FROM users WHERE email = $1",
    [email],
  );
  return result.rows[0];
  // What is this? An object? undefined? What keys does it have?
  // Nothing in the file, the editor, or the type system tells you.
}

async function authenticate(email, password) {
  const user = await findUserByEmail(email);
  if (!user) return null;

  // ❌ password_hash was never SELECTed. undefined at runtime.
  const valid = await bcrypt.compare(password, user.password_hash);
  if (!valid) return null;

  // ❌ camelCase guess. The real key is created_at. undefined.
  return { userId: user.id, since: user.createdAt.toISOString() };
  //                                    ^ TypeError: Cannot read properties of undefined
}

async function orgRevenue(orgId) {
  const result = await pool.query(
    "SELECT SUM(total_cents) AS total FROM orders WHERE org_id = $1",
    [orgId],
  );
  // ❌ SUM() over bigint returns NUMERIC → pg gives you a STRING.
  //    total is "482300", not 482300.
  return result.rows[0].total + 1000;   // "4823001000"
}

async function deleteUser(userId) {
  // ❌ String interpolation. One quote in userId and your table is gone.
  await pool.query(`DELETE FROM users WHERE id = '${userId}'`);
}
```

```ts
// ── TypeScript — one assertion per query, made deliberately and visibly ──────
import { Pool, QueryResultRow } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// The row interface mirrors the SELECT list EXACTLY — snake_case included:
interface UserAuthRow extends QueryResultRow {
  id:            string;
  email:         string;
  password_hash: string;
  created_at:    Date;
}

async function findUserForAuth(email: string): Promise<UserAuthRow | undefined> {
  const result = await pool.query<UserAuthRow>(
    // Every column in the interface appears here. Explicit list, never SELECT *.
    `SELECT id, email, password_hash, created_at
       FROM users
      WHERE email = $1`,
    [email],                       // parameters — never interpolation
  );
  return result.rows[0];           // UserAuthRow | undefined under noUncheckedIndexedAccess
}

interface AuthResult { userId: string; authToken: string; since: string }

async function authenticate(email: string, password: string): Promise<AuthResult | null> {
  const row = await findUserForAuth(email);
  if (!row) return null;           // ✅ the compiler forced this check

  const valid = await bcrypt.compare(password, row.password_hash);  // ✅ known string
  if (!valid) return null;

  return {
    userId:    row.id,
    authToken: signJwt({ userId: row.id }),
    since:     row.created_at.toISOString(),  // ✅ Date, not undefined
    // since:  row.createdAt.toISOString(),   // ❌ Property 'createdAt' does not exist
  };
}

// bigint / numeric aggregates come back as strings — the TYPE says so, so you
// are forced to convert:
interface RevenueRow extends QueryResultRow { total: string | null }

async function orgRevenue(orgId: string): Promise<number> {
  const result = await pool.query<RevenueRow>(
    "SELECT SUM(total_cents)::text AS total FROM orders WHERE org_id = $1",
    [orgId],
  );
  const total = result.rows[0]?.total ?? "0";
  // return total + 1000;   ❌ Operator '+' cannot be applied to 'string' and 'number'
  return Number(total);     // ✅ forced to be explicit
}
```

The revelation: TypeScript cannot check your SQL, but it **can** force every mismatch between "what I believe this query returns" and "how I use it" into a single, visible, reviewable place — the row interface. Once you also feed that interface through a `zod` parse (see Concept 8), the belief becomes a fact.

---

## Syntax

```ts
import {
  Pool, PoolClient, PoolConfig,
  Client, QueryResult, QueryResultRow, QueryConfig, DatabaseError,
} from "pg";

// ── 1. The pool ─────────────────────────────────────────────────────────────
const pool: Pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
} satisfies PoolConfig);

// ── 2. A row interface — mirrors the SELECT list, snake_case ────────────────
//    `extends QueryResultRow` is the constraint pg's generic requires:
//    type QueryResultRow = { [column: string]: any }
interface UserRow extends QueryResultRow {
  id:         string;      // uuid  → string
  email:      string;      // text  → string
  is_active:  boolean;     // bool  → boolean
  login_count: number;     // int4  → number
  balance_cents: string;   // int8  → STRING (not number!)
  created_at: Date;        // timestamptz → Date
  deleted_at: Date | null; // nullable column → | null
}

// ── 3. Querying — the generic goes on .query<T>() ───────────────────────────
const result: QueryResult<UserRow> = await pool.query<UserRow>(
  "SELECT id, email, is_active, login_count, balance_cents, created_at, deleted_at " +
  "FROM users WHERE is_active = $1 LIMIT $2",
  [true, 20],
);

result.rows;      // UserRow[]
result.rowCount;  // number | null  ← nullable in pg 8.11+
result.command;   // "SELECT" | "INSERT" | ...
result.fields;    // FieldDef[] — column metadata (name, dataTypeID, ...)

// ── 4. QueryConfig — the object form, useful for named/prepared queries ─────
const config: QueryConfig<[string]> = {
  name: "find-user-by-email",   // named → Postgres prepares & caches the plan
  text: "SELECT id, email FROM users WHERE email = $1",
  values: ["ada@example.com"],
};
const r2 = await pool.query<UserRow, [string]>(config);

// ── 5. A single client from the pool — required for transactions ───────────
const client: PoolClient = await pool.connect();
try {
  await client.query("BEGIN");
  await client.query("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2",
                     [100, "acct_1"]);
  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();            // ALWAYS release, or you leak the connection
}

// ── 6. Narrowing driver errors ─────────────────────────────────────────────
try {
  await pool.query("INSERT INTO users (email) VALUES ($1)", ["dupe@example.com"]);
} catch (error) {
  if (error instanceof DatabaseError && error.code === "23505") {
    // 23505 = unique_violation
  }
  throw error;
}
```

---

## How it works — concept by concept

### Concept 1 — `QueryResult<T>` and `QueryResultRow`: what the types actually say

`pg`'s type surface is tiny. Here it is, essentially verbatim:

```ts
// From @types/pg (pg ships its own types in v8.11+ via @types/pg):

interface QueryResultRow {
  [column: string]: any;        // ← an index signature. This is the constraint.
}

interface QueryResult<R extends QueryResultRow = any> {
  rows:     R[];                // ← the only field you use 95% of the time
  rowCount: number | null;      // ← NULLABLE. Not every command reports a count.
  command:  string;             // "SELECT" | "INSERT" | "UPDATE" | "DELETE" | ...
  oid:      number;
  fields:   FieldDef[];         // per-column metadata from the wire protocol
}

interface FieldDef {
  name:         string;         // column name as returned
  tableID:      number;
  columnID:     number;
  dataTypeID:   number;         // the Postgres OID — 23 = int4, 25 = text, ...
  dataTypeSize: number;
  dataTypeModifier: number;
  format:       string;
}
```

Three consequences fall straight out of this.

**(a) `QueryResultRow` has an index signature, so your row interfaces must be index-signature-compatible.** Interfaces without an index signature are *not* automatically assignable to one, which is why you sometimes see an error like `Index signature for type 'string' is missing in type 'UserRow'`:

```ts
// ❌ Depending on your pg/@types version, a plain interface can fail the constraint:
interface UserRow { id: string; email: string }
await pool.query<UserRow>(sql);
//               ~~~~~~~ Type 'UserRow' does not satisfy the constraint 'QueryResultRow'

// ✅ Three fixes, in order of preference:
interface UserRow2 extends QueryResultRow { id: string; email: string }  // explicit extends
type UserRow3 = { id: string; email: string };                           // type aliases pass
interface UserRow4 { id: string; email: string; [column: string]: unknown } // manual index sig
```

Prefer `extends QueryResultRow` — it documents intent and survives version bumps. (Why do `type` aliases pass where `interface` fails? Because object *type aliases* get implicit index signatures during assignability checks; interfaces do not, since they're open to declaration merging. See *18 — Interfaces vs type aliases*.)

**(b) `rowCount` is `number | null`.** In older `@types/pg` it was `number`. It is nullable because not every command reports a row count.

```ts
const result = await pool.query("UPDATE users SET is_active = false WHERE org_id = $1", [orgId]);

// ❌ Object is possibly 'null':
if (result.rowCount > 0) { /* ... */ }

// ✅
if ((result.rowCount ?? 0) > 0) { /* ... */ }
```

**(c) `fields` is the only *real* runtime information you get about the shape.** You could, in principle, validate your generic against `fields[].name` — and that's exactly the trick used in Concept 8's dev-mode column checker.

### Concept 2 — The crucial warning: the row generic is an unchecked assertion

This is the single most important idea in this document, so it gets its own concept.

```ts
interface UserRow extends QueryResultRow {
  id:            string;
  email:         string;
  password_hash: string;
  is_admin:      boolean;
  created_at:    Date;
}

// The query only selects TWO columns. The generic claims FIVE.
const result = await pool.query<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);

const user = result.rows[0];
user?.password_hash;   // TypeScript: string.   Runtime: undefined.
user?.is_admin;        // TypeScript: boolean.  Runtime: undefined.
user?.created_at;      // TypeScript: Date.     Runtime: undefined.

if (user && !user.is_admin) throw new Error("Forbidden");
// ⚠️ undefined is falsy → !undefined is true → this ALWAYS throws.
// Or invert the check and it always PASSES. This is how authorization bugs ship.
```

There is **no** mechanism by which TypeScript could catch this. `pool.query<T>()` is declared roughly as:

```ts
query<R extends QueryResultRow = any, I = any[]>(
  queryText: string,
  values?: I,
): Promise<QueryResult<R>>;
```

`queryText` is a `string`. TypeScript does not parse SQL, does not know your schema, and does not connect to your database at compile time. `R` is pure declaration. It is *literally* the same trust level as:

```ts
const rows = (await pool.query("SELECT id, email FROM users")).rows as UserRow[];
```

Internalise this list of things the generic does **not** protect you from:

| Mismatch | Compiler | Runtime |
|---|---|---|
| Column not in the SELECT list | ✅ silent | `undefined` |
| Column renamed by a migration | ✅ silent | `undefined` |
| `snake_case` column typed as `camelCase` | ✅ silent | `undefined` |
| `int8`/`numeric` typed as `number` | ✅ silent | a `string` |
| Nullable column typed non-nullable | ✅ silent | `null` |
| `SELECT *` (shape depends on the live schema) | ✅ silent | whatever the DB has today |
| Typo in the SQL column alias | ✅ silent | `undefined` |
| A `JOIN` producing duplicate column names | ✅ silent | last one wins |

Three practical defences, all covered later:

1. **Never `SELECT *`.** Write explicit column lists that mirror the interface line for line.
2. **Keep the interface next to the query.** One row interface per query function, in the same file, ideally directly above it.
3. **Parse at the boundary** with zod for anything security-relevant, externally-shaped, or aggregate-derived (Concept 8).

### Concept 3 — A generic `query<T>()` helper

Calling `pool.query<T>()` directly everywhere scatters the assertion across your codebase and gives you nowhere to add logging, timing, error mapping, or validation. Wrap it once:

```ts
import { Pool, PoolClient, QueryResult, QueryResultRow, DatabaseError } from "pg";

export const pool = new Pool({ connectionString: process.env.DATABASE_URL });

/** Anything you can run a query on: the pool itself, or a checked-out client. */
export type Queryable = Pick<Pool, "query"> | Pick<PoolClient, "query">;

/**
 * Run a query and return the rows.
 *
 * ⚠️ `TRow` is an UNCHECKED ASSERTION about what the SQL returns.
 *    Keep the SELECT list and TRow in sync by hand, or use `queryValidated`.
 */
export async function query<TRow extends QueryResultRow>(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<TRow[]> {
  const startedAt = Date.now();
  try {
    const result: QueryResult<TRow> = await executor.query<TRow>(sql, params as unknown[]);
    return result.rows;
  } catch (error) {
    if (error instanceof DatabaseError) {
      logger.error("query failed", { code: error.code, detail: error.detail, sql });
    }
    throw error;
  } finally {
    logger.debug("query", { ms: Date.now() - startedAt, sql });
  }
}

/** Exactly one row expected. Throws if 0 or >1. Return type is NOT nullable. */
export async function queryOne<TRow extends QueryResultRow>(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<TRow> {
  const rows = await query<TRow>(sql, params, executor);
  if (rows.length !== 1) {
    throw new Error(`Expected exactly 1 row, got ${rows.length}`);
  }
  return rows[0] as TRow;   // length checked above
}

/** Zero or one row. Nullable return — the caller must handle the miss. */
export async function queryMaybeOne<TRow extends QueryResultRow>(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<TRow | null> {
  const rows = await query<TRow>(sql, params, executor);
  if (rows.length > 1) throw new Error(`Expected at most 1 row, got ${rows.length}`);
  return rows[0] ?? null;
}

/** For INSERT/UPDATE/DELETE where you only care how many rows changed. */
export async function execute(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<number> {
  const result = await executor.query(sql, params as unknown[]);
  return result.rowCount ?? 0;
}
```

Now the call sites are clean and the nullability is *carried by the function you pick*, not by an `rows[0]` you might forget to check:

```ts
interface UserRow extends QueryResultRow { id: string; email: string; created_at: Date }

const allUsers   = await query<UserRow>("SELECT id, email, created_at FROM users");           // UserRow[]
const maybeUser  = await queryMaybeOne<UserRow>("SELECT id, email, created_at FROM users WHERE id = $1", [userId]);  // UserRow | null
const definitely = await queryOne<UserRow>("SELECT id, email, created_at FROM users WHERE id = $1", [userId]);       // UserRow (throws on miss)
const removed    = await execute("DELETE FROM sessions WHERE user_id = $1", [userId]);        // number
```

Note the `executor` parameter defaulting to `pool`. That single design decision is what lets every one of these helpers participate in a transaction (Concept 6) without a second copy of the code.

### Concept 4 — Row interfaces vs `snake_case` columns, and camelCase mapping types

Postgres convention is `snake_case`. TypeScript/JS convention is `camelCase`. You have three options, and only two of them are good.

**Option A — Row interfaces stay snake_case, mappers convert to camelCase domain types.** This is the honest, boring, correct default.

```ts
// The ROW type mirrors the database exactly. It is an infrastructure detail.
interface UserRow extends QueryResultRow {
  id:            string;
  org_id:        string;
  email:         string;
  display_name:  string;
  is_active:     boolean;
  last_login_at: Date | null;
  created_at:    Date;
}

// The DOMAIN type is what the rest of your app sees. It is camelCase and clean.
export interface User {
  userId:      string;
  orgId:       string;
  email:       string;
  displayName: string;
  isActive:    boolean;
  lastLoginAt: Date | null;
  createdAt:   Date;
}

// One mapper. It is the ONLY place snake_case exists outside the SQL string.
function toUser(row: UserRow): User {
  return {
    userId:      row.id,
    orgId:       row.org_id,
    email:       row.email,
    displayName: row.display_name,
    isActive:    row.is_active,
    lastLoginAt: row.last_login_at,
    createdAt:   row.created_at,
  };
}
```

Verbose, yes — but every field rename in the database produces exactly one compile error, in exactly one file, and the fix is obvious.

**Option B — Alias in SQL, so the driver returns camelCase directly.** Postgres preserves case only inside double quotes:

```ts
interface UserCamelRow extends QueryResultRow {
  userId:      string;
  displayName: string;
  isActive:    boolean;
  createdAt:   Date;
}

const users = await query<UserCamelRow>(`
  SELECT id            AS "userId",
         display_name  AS "displayName",
         is_active     AS "isActive",
         created_at    AS "createdAt"
    FROM users
   WHERE org_id = $1
`, [orgId]);
```

⚠️ The double quotes are **mandatory**. `SELECT display_name AS displayName` returns a column literally named `displayname` (Postgres downcases unquoted identifiers), and your `row.displayName` is `undefined` — a classic silent failure.

**Option C — a `SnakeToCamel` mapped type, so the domain type is *derived* from the row type.** This is where TypeScript gets fun, and it removes the possibility of the two types drifting:

```ts
// ── Convert a snake_case string literal type to camelCase ──────────────────
// "last_login_at" → "lastLoginAt"
type SnakeToCamel<S extends string> =
  S extends `${infer Head}_${infer Tail}`
    ? `${Head}${Capitalize<SnakeToCamel<Tail>>}`
    : S;

type A = SnakeToCamel<"last_login_at">;   // "lastLoginAt"
type B = SnakeToCamel<"email">;           // "email"
type C = SnakeToCamel<"org_id">;          // "orgId"

// ── Remap every key of a row type ─────────────────────────────────────────
type CamelCaseKeys<T> = {
  [K in keyof T as K extends string ? SnakeToCamel<K> : K]: T[K];
};

type UserDomain = CamelCaseKeys<UserRow>;
// {
//   id:          string;
//   orgId:       string;
//   email:       string;
//   displayName: string;
//   isActive:    boolean;
//   lastLoginAt: Date | null;
//   createdAt:   Date;
// }
```

And the runtime mapper that matches it — one function for every table:

```ts
function snakeToCamel(key: string): string {
  return key.replace(/_([a-z0-9])/g, (_match, char: string) => char.toUpperCase());
}

/** Runtime counterpart of CamelCaseKeys<T>. Shallow — intentionally. */
export function camelizeRow<T extends QueryResultRow>(row: T): CamelCaseKeys<T> {
  const out: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(row)) {
    out[snakeToCamel(key)] = value;
  }
  return out as CamelCaseKeys<T>;   // ⚠️ assertion — the type and the loop must agree
}

export function camelizeRows<T extends QueryResultRow>(rows: T[]): CamelCaseKeys<T>[] {
  return rows.map(camelizeRow);
}

const rows   = await query<UserRow>("SELECT id, org_id, email, display_name, is_active, last_login_at, created_at FROM users");
const users  = camelizeRows(rows);
users[0]?.displayName;   // ✅ string
users[0]?.display_name;  // ❌ Property 'display_name' does not exist
```

Note the honest `as` inside `camelizeRow` — the mapped type and the regex are two independent descriptions of the same transformation, and TypeScript cannot prove they agree. That's acceptable *because it's one function you test once*, rather than one assertion per query.

There is a fourth option that people reach for and shouldn't: setting a global `pg` type parser or a `camelCase` post-processor on the pool. It works, but it makes the row-type/SQL relationship invisible and breaks the "the interface mirrors the SELECT list" discipline that keeps you honest.

### Concept 5 — Typed parameter arrays: the second generic

`pool.query` has a *second* generic for the values array:

```ts
query<R extends QueryResultRow = any, I = any[]>(
  queryTextOrConfig: string | QueryConfig<I>,
  values?: I,
): Promise<QueryResult<R>>;
```

By default `I = any[]`, so this compiles:

```ts
await pool.query("SELECT * FROM users WHERE id = $1 AND is_active = $2", [userId]);
//                                                              ^^ $2 has no value → runtime error
await pool.query("SELECT id FROM users WHERE id = $1", [userId, "extra", 42, {}]);
//                                                              ^^^^^^^^^^^^^^ ignored... or an error
```

Pin the tuple type and the arity becomes checked:

```ts
type FindUserParams = [userId: string];    // a labelled tuple — labels show in tooltips

const result = await pool.query<UserRow, FindUserParams>(
  "SELECT id, email, created_at FROM users WHERE id = $1",
  [userId],
);

// ❌ Source has 2 elements but target allows only 1:
await pool.query<UserRow, FindUserParams>("SELECT ...", [userId, "oops"]);

// ❌ Type 'number' is not assignable to type 'string':
await pool.query<UserRow, FindUserParams>("SELECT ...", [42]);
```

To get this ergonomically through your own helper, add the params generic there too:

```ts
export async function queryT<
  TRow extends QueryResultRow,
  TParams extends readonly unknown[] = readonly unknown[],
>(
  sql: string,
  params: TParams,
  executor: Queryable = pool,
): Promise<TRow[]> {
  const result = await executor.query<TRow>(sql, params as unknown[]);
  return result.rows;
}

type ListOrdersParams = [orgId: string, status: OrderStatus, limit: number];

const orders = await queryT<OrderRow, ListOrdersParams>(
  `SELECT id, org_id, status, total_cents, created_at
     FROM orders
    WHERE org_id = $1 AND status = $2
    ORDER BY created_at DESC
    LIMIT $3`,
  [orgId, "paid", 50],
);
```

**The strongest version of this pattern is to bind the SQL and the params tuple together in one object**, so a query can never be called with the wrong arguments:

```ts
interface TypedQuery<TRow extends QueryResultRow, TParams extends readonly unknown[]> {
  readonly text: string;
  readonly rowType?: TRow;      // phantom — only carries the type, never assigned
  readonly _params?: TParams;   // phantom
}

function defineQuery<TRow extends QueryResultRow, TParams extends readonly unknown[]>(
  text: string,
): TypedQuery<TRow, TParams> {
  return { text };
}

const findUserById = defineQuery<UserRow, [userId: string]>(
  `SELECT id, org_id, email, display_name, is_active, last_login_at, created_at
     FROM users WHERE id = $1`,
);

async function run<TRow extends QueryResultRow, TParams extends readonly unknown[]>(
  q: TypedQuery<TRow, TParams>,
  params: TParams,
  executor: Queryable = pool,
): Promise<TRow[]> {
  const result = await executor.query<TRow>(q.text, params as unknown[]);
  return result.rows;
}

const found = await run(findUserById, [userId]);   // ✅ TRow and TParams both inferred
// await run(findUserById, [userId, "x"]);         // ❌ too many arguments
// await run(findUserById, [123]);                 // ❌ number not assignable to string
```

Two hard rules about parameters that types cannot enforce, so you must:

```ts
// ✅ Values are parameterised — the driver sends them out-of-band. Injection-proof.
await query<UserRow>("SELECT id FROM users WHERE email = $1", [requestBody.email]);

// ❌ NEVER interpolate. $1 is not string formatting; it's a protocol-level placeholder.
await query<UserRow>(`SELECT id FROM users WHERE email = '${requestBody.email}'`);

// ⚠️ IDENTIFIERS (table/column names) CANNOT be parameterised. $1 only works for VALUES.
//    If you need a dynamic ORDER BY column, use an allowlist keyed by a union type:
const SORTABLE = { createdAt: "created_at", email: "email", total: "total_cents" } as const;
type SortKey = keyof typeof SORTABLE;

function buildOrderBy(sortKey: SortKey, direction: "asc" | "desc"): string {
  return `ORDER BY ${SORTABLE[sortKey]} ${direction === "asc" ? "ASC" : "DESC"}`;
  // Safe: both halves come from closed unions, never from user strings.
}
```

### Concept 6 — Transactions with a typed client and a `withTransaction` helper

A transaction must run every statement on the **same physical connection**. `pool.query()` grabs an arbitrary idle client per call, so `pool.query("BEGIN")` followed by `pool.query("UPDATE ...")` may run on two different connections — the update escapes the transaction entirely. This is the number one `pg` bug.

```ts
// ❌ BROKEN — three statements, possibly three different connections:
await pool.query("BEGIN");
await pool.query("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2", [100, from]);
await pool.query("COMMIT");
```

You must check out a `PoolClient` and use it for all statements:

```ts
import { Pool, PoolClient } from "pg";

export async function withTransaction<T>(
  fn: (tx: PoolClient) => Promise<T>,
  executorPool: Pool = pool,
): Promise<T> {
  const client: PoolClient = await executorPool.connect();
  try {
    await client.query("BEGIN");
    const result = await fn(client);      // ← everything inside uses `tx`
    await client.query("COMMIT");
    return result;
  } catch (error) {
    // Rollback can itself fail (e.g. the connection died). Never let it mask
    // the original error:
    try {
      await client.query("ROLLBACK");
    } catch (rollbackError) {
      logger.error("rollback failed", { rollbackError });
    }
    throw error;
  } finally {
    client.release();                     // ALWAYS — a leaked client exhausts the pool
  }
}
```

Because our `query`/`queryOne`/`execute` helpers take an `executor`, they slot straight in:

```ts
interface AccountRow extends QueryResultRow { id: string; balance_cents: string }

export async function transfer(fromId: string, toId: string, amountCents: number): Promise<void> {
  await withTransaction(async (tx) => {
    // FOR UPDATE takes a row lock for the duration of the transaction:
    const from = await queryOne<AccountRow>(
      "SELECT id, balance_cents FROM accounts WHERE id = $1 FOR UPDATE",
      [fromId],
      tx,                                  // ← the client, not the pool
    );

    if (Number(from.balance_cents) < amountCents) {
      throw new InsufficientFundsError(fromId);   // throwing rolls back
    }

    await execute("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2",
                  [amountCents, fromId], tx);
    await execute("UPDATE accounts SET balance_cents = balance_cents + $1 WHERE id = $2",
                  [amountCents, toId], tx);
    await execute("INSERT INTO ledger (from_account_id, to_account_id, amount_cents) VALUES ($1, $2, $3)",
                  [fromId, toId, amountCents], tx);
  });
}
```

**Making the client-vs-pool distinction impossible to get wrong at the type level** is worth the extra ten lines. Use a branded type (see *38 — Branded types*) so a transactional repository method *cannot* be called with the pool:

```ts
declare const txBrand: unique symbol;

/** A PoolClient that is known to be inside BEGIN...COMMIT. */
export type Tx = PoolClient & { readonly [txBrand]: true };

export async function withTransaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const result = await fn(client as Tx);   // the ONE place the brand is applied
    await client.query("COMMIT");
    return result;
  } catch (error) {
    try { await client.query("ROLLBACK"); } catch { /* logged elsewhere */ }
    throw error;
  } finally {
    client.release();
  }
}

// A function that MUST be transactional now says so in its signature:
async function debitAccount(tx: Tx, accountId: string, amountCents: number): Promise<void> {
  await execute("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2",
                [amountCents, accountId], tx);
}

// await debitAccount(pool, id, 100);
//                    ~~~~ ❌ Argument of type 'Pool' is not assignable to parameter of type 'Tx'
await withTransaction((tx) => debitAccount(tx, accountId, 100));   // ✅
```

**Savepoints** for nested transactions — the same helper, one level down:

```ts
export async function withSavepoint<T>(tx: Tx, name: string, fn: () => Promise<T>): Promise<T> {
  // The name is an identifier, so it CANNOT be parameterised — validate it hard:
  if (!/^[a-z_][a-z0-9_]{0,30}$/.test(name)) throw new Error(`Invalid savepoint name: ${name}`);
  await tx.query(`SAVEPOINT ${name}`);
  try {
    const result = await fn();
    await tx.query(`RELEASE SAVEPOINT ${name}`);
    return result;
  } catch (error) {
    await tx.query(`ROLLBACK TO SAVEPOINT ${name}`);
    throw error;
  }
}
```

And **isolation levels**, typed as a closed union rather than a free string:

```ts
export type IsolationLevel = "READ COMMITTED" | "REPEATABLE READ" | "SERIALIZABLE";

export async function withTransactionAt<T>(
  level: IsolationLevel,
  fn: (tx: Tx) => Promise<T>,
): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query(`BEGIN ISOLATION LEVEL ${level}`);   // safe: closed union
    const result = await fn(client as Tx);
    await client.query("COMMIT");
    return result;
  } catch (error) {
    try { await client.query("ROLLBACK"); } catch { /* ignore */ }
    throw error;
  } finally {
    client.release();
  }
}
```

### Concept 7 — `rows[0]` and `noUncheckedIndexedAccess`

By default, TypeScript lies about array indexing:

```ts
const rows = await query<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);
const user = rows[0];        // UserRow          ← a LIE if the array is empty
console.log(user.email);     // compiles fine, TypeError at runtime
```

Turn on the compiler flag that fixes this:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true   // ← every T[number] access becomes T | undefined
  }
}
```

Now:

```ts
const user = rows[0];        // UserRow | undefined  ← the truth
console.log(user.email);     // ❌ 'user' is possibly 'undefined'

// ✅ Handle it — three idiomatic shapes:
if (!user) throw new NotFoundError(`User ${userId} not found`);
console.log(user.email);                       // narrowed to UserRow

const email = rows[0]?.email ?? "unknown";     // optional chaining + default

const [first] = rows;                          // UserRow | undefined — same rule applies
```

The flag has knock-on effects worth knowing before you enable it repo-wide:

```ts
// It also applies to Record<string, T> index access:
const columnByName: Record<string, string> = { id: "uuid" };
const t = columnByName["id"];   // string | undefined  (even though you "know" it's there)

// And to for-loop indexing:
for (let i = 0; i < rows.length; i++) {
  const row = rows[i];          // UserRow | undefined — despite the bounds check
  if (!row) continue;           // annoying but honest
  process(row);
}

// ✅ Prefer iteration forms that don't index — these stay non-optional:
for (const row of rows)   process(row);        // row: UserRow
rows.forEach((row) => process(row));           // row: UserRow
const emails = rows.map((row) => row.email);   // row: UserRow
```

That last block is the practical takeaway: **with `noUncheckedIndexedAccess`, `for...of` and `.map()` are strictly nicer than indexed loops.** And the ugly `rows[i]` friction pushes you toward the `queryOne` / `queryMaybeOne` helpers from Concept 3, which encode "exactly one" and "zero or one" in the *return type* instead of at every call site.

A related trap — `.find()` returns `T | undefined` **regardless** of the flag, and people forget:

```ts
const admin = rows.find((row) => row.is_admin);   // UserRow | undefined, always
```

### Concept 8 — Runtime validation at the DB boundary with zod

Everything so far makes your beliefs *explicit*. zod makes them *true*.

```ts
import { z } from "zod";

// ── 1. The schema describes what you EXPECT the DB to hand back ────────────
const userRowSchema = z.object({
  id:            z.string().uuid(),
  org_id:        z.string().uuid(),
  email:         z.string().email(),
  display_name:  z.string().min(1),
  is_active:     z.boolean(),
  // int8/bigint arrives as a STRING — accept it and coerce, don't pretend:
  balance_cents: z.string().regex(/^-?\d+$/),
  last_login_at: z.coerce.date().nullable(),
  created_at:    z.coerce.date(),
});

// ── 2. The row type is DERIVED, so schema and type can never drift ─────────
type UserRow = z.infer<typeof userRowSchema>;

// ── 3. And the domain type is a separate, transformed shape ────────────────
const userSchema = userRowSchema.transform((row) => ({
  userId:       row.id,
  orgId:        row.org_id,
  email:        row.email,
  displayName:  row.display_name,
  isActive:     row.is_active,
  balanceCents: Number(row.balance_cents),   // string → number, validated above
  lastLoginAt:  row.last_login_at,
  createdAt:    row.created_at,
}));

export type User = z.infer<typeof userSchema>;
```

Then a validating query helper. Note the return type is inferred *from the schema*, so there is no generic left for you to get wrong:

```ts
import { z } from "zod";

export async function queryValidated<TSchema extends z.ZodTypeAny>(
  schema: TSchema,
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<z.infer<TSchema>[]> {
  const result = await executor.query(sql, params as unknown[]);
  return result.rows.map((row: unknown, index: number) => {
    const parsed = schema.safeParse(row);
    if (!parsed.success) {
      // A schema drift is an operational incident, not a user error —
      // log the SQL and the exact field path, then fail loudly.
      logger.error("row failed validation", {
        sql,
        rowIndex: index,
        issues: parsed.error.issues,
      });
      throw new Error(
        `Row ${index} did not match schema: ${parsed.error.issues
          .map((i) => `${i.path.join(".")}: ${i.message}`)
          .join("; ")}`,
      );
    }
    return parsed.data as z.infer<TSchema>;
  });
}

export async function queryOneValidated<TSchema extends z.ZodTypeAny>(
  schema: TSchema,
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<z.infer<TSchema>> {
  const rows = await queryValidated(schema, sql, params, executor);
  const first = rows[0];
  if (rows.length !== 1 || first === undefined) {
    throw new Error(`Expected exactly 1 row, got ${rows.length}`);
  }
  return first;
}
```

Usage — and notice the generic has vanished from the call site entirely:

```ts
const users: User[] = await queryValidated(
  userSchema,
  `SELECT id, org_id, email, display_name, is_active, balance_cents, last_login_at, created_at
     FROM users WHERE org_id = $1`,
  [orgId],
);
users[0]?.balanceCents;   // ✅ number, validated, camelCase
```

Now delete a column from the SELECT list and run the code: you get a precise, immediate `balance_cents: Required` error naming the exact query — instead of an `undefined` propagating three layers up.

**Where to apply this, honestly.** zod-parsing every row of a 50,000-row analytics scan is real overhead. A pragmatic policy:

| Query | Validate? |
|---|---|
| Auth / permissions / billing rows | ✅ always |
| Single-row reads that drive branching logic | ✅ always |
| Data crossing into an HTTP response | ✅ always (or validate the response DTO) |
| Aggregates (`SUM`, `COUNT`, `json_agg`) | ✅ always — types are least predictable here |
| Bulk internal batch scans | ⚠️ validate the first row only, or sample |
| Anything you `SELECT *` from | 🚨 stop `SELECT *`-ing, then validate |

A cheap middle ground: **validate the shape once per query, not once per row.**

```ts
export async function queryValidatedShape<TSchema extends z.ZodTypeAny>(
  schema: TSchema,
  sql: string,
  params: ReadonlyArray<unknown> = [],
): Promise<z.infer<TSchema>[]> {
  const result = await pool.query(sql, params as unknown[]);
  const first = result.rows[0];
  if (first !== undefined) schema.parse(first);   // one parse, catches shape drift
  return result.rows as z.infer<TSchema>[];       // ⚠️ the rest is asserted
}
```

And a **dev-only column checker** that uses `QueryResult.fields` — the one piece of genuine runtime schema information the driver gives you:

```ts
/** In development, assert the returned columns exactly match the schema keys. */
function assertColumns(result: QueryResult, expected: readonly string[], sql: string): void {
  if (process.env.NODE_ENV === "production") return;
  const actual  = result.fields.map((field) => field.name).sort();
  const wanted  = [...expected].sort();
  const missing = wanted.filter((c) => !actual.includes(c));
  const extra   = actual.filter((c) => !wanted.includes(c));
  if (missing.length > 0 || extra.length > 0) {
    throw new Error(
      `Column mismatch for query:\n${sql}\n  missing: ${missing.join(", ") || "none"}` +
      `\n  unexpected: ${extra.join(", ") || "none"}`,
    );
  }
}

// With a zod object schema you get `expected` for free:
assertColumns(result, Object.keys(userRowSchema.shape), sql);
```

This catches the *entire* class of "the generic lied" bugs on the very first request in dev, with zero production cost.

### Concept 9 — Postgres types → TypeScript types (the conversion table that bites)

`pg` parses wire-format values into JS values using `pg-types`. The mapping is not always what you'd guess, and every surprise below has cost someone a production incident.

| Postgres type | OID | JS value from `pg` | Correct TS type | Note |
|---|---|---|---|---|
| `int2`, `int4` | 21, 23 | `number` | `number` | safe |
| `int8` / `bigint` | 20 | **`string`** | `string` | would lose precision as `number` |
| `numeric` / `decimal` | 1700 | **`string`** | `string` | arbitrary precision |
| `float4`, `float8` | 700, 701 | `number` | `number` | |
| `bool` | 16 | `boolean` | `boolean` | |
| `text`, `varchar`, `uuid` | 25, 1043, 2950 | `string` | `string` | |
| `date` | 1082 | `Date` | `Date` | midnight in **server local time** |
| `timestamptz` | 1184 | `Date` | `Date` | correct instant |
| `timestamp` (no tz) | 1114 | `Date` | `Date` | ⚠️ interpreted as local time |
| `json`, `jsonb` | 114, 3802 | parsed object | a declared interface | already parsed — don't `JSON.parse` |
| `int4[]`, `text[]` | 1007, 1009 | `number[]` / `string[]` | `T[]` | arrays are parsed |
| `bytea` | 17 | `Buffer` | `Buffer` | |
| `interval` | 1186 | `PostgresInterval` object | that object | not a number of ms |
| `COUNT(*)` | int8 | **`string`** | `string` | the classic |
| `SUM(int)` | int8 | **`string`** | `string` | |
| `SUM(numeric)` | numeric | **`string`** | `string` | |

The two you will hit this week:

```ts
// ── COUNT(*) is bigint → string ────────────────────────────────────────────
interface CountRow extends QueryResultRow { count: string }   // ← string, not number

const [row] = await query<CountRow>("SELECT COUNT(*) AS count FROM users WHERE org_id = $1", [orgId]);
const total: number = Number(row?.count ?? "0");

// ❌ If you had typed it `count: number`, this compiles and is wrong:
//    row.count > 100  → "9" > 100 → false, because "9" is a string.

// ── NULL aggregates ────────────────────────────────────────────────────────
// SUM over zero rows returns NULL, not 0:
interface SumRow extends QueryResultRow { total: string | null }
const [sumRow] = await query<SumRow>("SELECT SUM(total_cents) AS total FROM orders WHERE org_id = $1", [orgId]);
const revenue = Number(sumRow?.total ?? 0);
// ✅ Or push the default into SQL, and drop the null from the type:
//    SELECT COALESCE(SUM(total_cents), 0)::bigint AS total
```

You *can* override the parsers globally, but do it deliberately and centrally:

```ts
import pgTypes from "pg-types";

// Parse int8 as a JS number. ⚠️ ONLY safe if you are certain no value exceeds
// Number.MAX_SAFE_INTEGER (9_007_199_254_740_991). Ids and money in cents are
// usually fine; event counters and Snowflake ids are NOT.
pgTypes.setTypeParser(pgTypes.builtins.INT8, (value: string) => Number(value));

// Keep DATE as a plain "YYYY-MM-DD" string instead of a timezone-shifted Date:
pgTypes.setTypeParser(pgTypes.builtins.DATE, (value: string) => value);
```

If you do this, the change is invisible to the type system — so write it once, in your db bootstrap module, next to a comment explaining which row interfaces now say `number` instead of `string`.

### Concept 10 — Typing `jsonb`, arrays, and `RETURNING`

**`jsonb`** columns come back already parsed. `pg` gives them to you as `any`, so the generic is doing all the work — which means it's a prime candidate for zod:

```ts
interface RequestMetadata {
  ip:        string;
  userAgent: string;
  traceId:   string;
}

interface AuditRow extends QueryResultRow {
  id:       string;
  user_id:  string;
  action:   string;
  metadata: RequestMetadata;      // ⚠️ pure assertion — jsonb is `any` at runtime
  tags:     string[];             // text[] — parsed into a real array
}

// ✅ Writing jsonb: pass the OBJECT and let pg serialise it. Do NOT JSON.stringify
//    into a jsonb parameter — you'd store a JSON *string* inside the jsonb.
await execute(
  "INSERT INTO audit_log (user_id, action, metadata, tags) VALUES ($1, $2, $3, $4)",
  [userId, "user.login", { ip, userAgent, traceId } satisfies RequestMetadata, ["auth", "web"]],
);

// ⚠️ The exception: `json`/`jsonb` columns holding a top-level ARRAY need an
//    explicit cast, because pg would send a JS array as a Postgres array:
await execute(
  "INSERT INTO audit_log (user_id, action, metadata) VALUES ($1, $2, $3::jsonb)",
  [userId, "bulk.import", JSON.stringify([{ sku: "A1" }, { sku: "B2" }])],
);
```

**`RETURNING`** turns writes into reads — and it's the cleanest way to get generated values back:

```ts
interface InsertedUserRow extends QueryResultRow {
  id:         string;
  email:      string;
  created_at: Date;
}

export async function createUser(email: string, passwordHash: string): Promise<InsertedUserRow> {
  return queryOne<InsertedUserRow>(
    `INSERT INTO users (email, password_hash)
          VALUES ($1, $2)
       RETURNING id, email, created_at`,   // ← mirrors InsertedUserRow exactly
    [email, passwordHash],
  );
}

// Upsert with RETURNING — note that ON CONFLICT DO NOTHING returns ZERO rows
// on conflict, so the return type must be nullable:
export async function ensureTag(orgId: string, label: string): Promise<{ id: string } | null> {
  return queryMaybeOne<{ id: string } & QueryResultRow>(
    `INSERT INTO tags (org_id, label) VALUES ($1, $2)
       ON CONFLICT (org_id, label) DO NOTHING
       RETURNING id`,
    [orgId, label],
  );
}
```

**Bulk inserts** with `UNNEST` — one round trip, one parameter per column, fully typed:

```ts
interface OrderItemInput { sku: string; quantity: number; unitCents: number }

export async function insertOrderItems(
  orderId: string,
  items: readonly OrderItemInput[],
  tx: Tx,
): Promise<number> {
  if (items.length === 0) return 0;
  return execute(
    `INSERT INTO order_items (order_id, sku, quantity, unit_cents)
     SELECT $1, * FROM UNNEST($2::text[], $3::int[], $4::int[])`,
    [
      orderId,
      items.map((item) => item.sku),
      items.map((item) => item.quantity),
      items.map((item) => item.unitCents),
    ],
    tx,
  );
}
```

---

## Example 1 — basic

```ts
// A minimal, fully typed users data-access module: pool, helpers, row types,
// domain types, and a mapper. Everything a small service actually needs.

import { Pool, PoolClient, QueryResult, QueryResultRow, DatabaseError } from "pg";

// ═══ 1. The pool ══════════════════════════════════════════════════════════

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});

pool.on("error", (error: Error) => {
  // Idle clients can die (network blips, DB restarts). Without this handler
  // Node emits an unhandled 'error' event and crashes the process.
  console.error("Unexpected pg pool error", error);
});

// ═══ 2. Helpers ═══════════════════════════════════════════════════════════

export type Queryable = Pick<Pool, "query"> | Pick<PoolClient, "query">;

/** ⚠️ TRow is an UNCHECKED assertion about the SELECT list. Keep them in sync. */
export async function query<TRow extends QueryResultRow>(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<TRow[]> {
  const result: QueryResult<TRow> = await executor.query<TRow>(sql, params as unknown[]);
  return result.rows;
}

export async function queryMaybeOne<TRow extends QueryResultRow>(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<TRow | null> {
  const rows = await query<TRow>(sql, params, executor);
  if (rows.length > 1) throw new Error(`Expected at most 1 row, got ${rows.length}`);
  return rows[0] ?? null;
}

export async function execute(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<number> {
  const result = await executor.query(sql, params as unknown[]);
  return result.rowCount ?? 0;   // rowCount is number | null
}

// ═══ 3. Row types — snake_case, mirroring the columns ═════════════════════

interface UserRow extends QueryResultRow {
  id:            string;        // uuid
  email:         string;        // text
  display_name:  string;        // text
  password_hash: string;        // text
  is_active:     boolean;       // bool
  login_count:   number;        // int4  → number
  last_login_at: Date | null;   // timestamptz, nullable
  created_at:    Date;          // timestamptz
}

/** Every column, in one place, reused by every SELECT below. */
const USER_COLUMNS = `
  id, email, display_name, password_hash, is_active, login_count, last_login_at, created_at
` as const;

// ═══ 4. Domain type — camelCase, no password hash, what the app sees ══════

export interface User {
  userId:      string;
  email:       string;
  displayName: string;
  isActive:    boolean;
  loginCount:  number;
  lastLoginAt: Date | null;
  createdAt:   Date;
}

function toUser(row: UserRow): User {
  return {
    userId:      row.id,
    email:       row.email,
    displayName: row.display_name,
    isActive:    row.is_active,
    loginCount:  row.login_count,
    lastLoginAt: row.last_login_at,
    createdAt:   row.created_at,
    // password_hash deliberately NOT mapped — it cannot leak past this line.
  };
}

// ═══ 5. Queries ═══════════════════════════════════════════════════════════

export async function findUserById(userId: string): Promise<User | null> {
  const row = await queryMaybeOne<UserRow>(
    `SELECT ${USER_COLUMNS} FROM users WHERE id = $1`,
    [userId],
  );
  return row ? toUser(row) : null;
}

export async function findUserByEmail(email: string): Promise<User | null> {
  const row = await queryMaybeOne<UserRow>(
    `SELECT ${USER_COLUMNS} FROM users WHERE lower(email) = lower($1)`,
    [email],
  );
  return row ? toUser(row) : null;
}

/** The ONE function allowed to see password_hash. */
export async function findCredentialsByEmail(
  email: string,
): Promise<{ userId: string; passwordHash: string; isActive: boolean } | null> {
  interface CredentialRow extends QueryResultRow {
    id: string;
    password_hash: string;
    is_active: boolean;
  }
  const row = await queryMaybeOne<CredentialRow>(
    "SELECT id, password_hash, is_active FROM users WHERE lower(email) = lower($1)",
    [email],
  );
  return row ? { userId: row.id, passwordHash: row.password_hash, isActive: row.is_active } : null;
}

export interface ListUsersOptions {
  orgId:  string;
  limit?: number;
  offset?: number;
  activeOnly?: boolean;
}

export async function listUsers(options: ListUsersOptions): Promise<User[]> {
  const { orgId, limit = 50, offset = 0, activeOnly = true } = options;
  const rows = await query<UserRow>(
    `SELECT ${USER_COLUMNS}
       FROM users
      WHERE org_id = $1
        AND ($2::boolean IS FALSE OR is_active = TRUE)
      ORDER BY created_at DESC
      LIMIT $3 OFFSET $4`,
    [orgId, activeOnly, limit, offset],
  );
  return rows.map(toUser);
}

export async function createUser(input: {
  orgId: string;
  email: string;
  displayName: string;
  passwordHash: string;
}): Promise<User> {
  try {
    const row = await queryMaybeOne<UserRow>(
      `INSERT INTO users (org_id, email, display_name, password_hash)
            VALUES ($1, $2, $3, $4)
         RETURNING ${USER_COLUMNS}`,
      [input.orgId, input.email, input.displayName, input.passwordHash],
    );
    if (!row) throw new Error("INSERT ... RETURNING produced no row");
    return toUser(row);
  } catch (error) {
    if (error instanceof DatabaseError && error.code === "23505") {
      throw new Error(`Email already registered: ${input.email}`);
    }
    throw error;
  }
}

export async function recordLogin(userId: string): Promise<void> {
  const changed = await execute(
    "UPDATE users SET login_count = login_count + 1, last_login_at = now() WHERE id = $1",
    [userId],
  );
  if (changed === 0) throw new Error(`User ${userId} not found`);
}

export async function deactivateUser(userId: string): Promise<boolean> {
  const changed = await execute(
    "UPDATE users SET is_active = FALSE WHERE id = $1 AND is_active = TRUE",
    [userId],
  );
  return changed > 0;
}
```

---

## Example 2 — real world backend use case

```ts
// A multi-tenant orders service on raw pg:
//   - a branded Tx type so transactional code cannot run on the pool
//   - zod-validated row schemas at the DB boundary
//   - explicit column lists, snake_case rows, camelCase domain objects
//   - bigint/numeric handled as strings and converted deliberately
//   - a dynamic filter builder with a typed, injection-proof parameter list
//   - an Express handler showing the full request → SQL → ApiResponse path

import { Pool, PoolClient, QueryResultRow, DatabaseError } from "pg";
import { z } from "zod";
import type { Request, Response } from "express";

// ═══════════════════════════════════════════════════════════════════════════
// 1. Infrastructure — pool, Tx brand, helpers
// ═══════════════════════════════════════════════════════════════════════════

export const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 20 });
pool.on("error", (error) => logger.error("pg pool error", { error }));

declare const txBrand: unique symbol;
/** A PoolClient proven to be inside BEGIN…COMMIT. Only withTransaction can mint one. */
export type Tx = PoolClient & { readonly [txBrand]: true };

export type Queryable = Pick<Pool, "query"> | Pick<PoolClient, "query">;

export async function withTransaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const result = await fn(client as Tx);
    await client.query("COMMIT");
    return result;
  } catch (error) {
    try {
      await client.query("ROLLBACK");
    } catch (rollbackError) {
      logger.error("rollback failed", { rollbackError });
    }
    throw error;
  } finally {
    client.release();
  }
}

/** Validating query. The return type is inferred from the zod schema. */
export async function queryValidated<TSchema extends z.ZodTypeAny>(
  schema: TSchema,
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<z.infer<TSchema>[]> {
  const result = await executor.query(sql, params as unknown[]);
  return result.rows.map((row: unknown, index: number) => {
    const parsed = schema.safeParse(row);
    if (!parsed.success) {
      throw new SchemaDriftError(sql, index, parsed.error.issues.map((i) => `${i.path.join(".")}: ${i.message}`));
    }
    return parsed.data as z.infer<TSchema>;
  });
}

export async function queryOneValidated<TSchema extends z.ZodTypeAny>(
  schema: TSchema,
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<z.infer<TSchema> | null> {
  const rows = await queryValidated(schema, sql, params, executor);
  if (rows.length > 1) throw new Error(`Expected at most 1 row, got ${rows.length}`);
  return rows[0] ?? null;
}

export async function execute(
  sql: string,
  params: ReadonlyArray<unknown> = [],
  executor: Queryable = pool,
): Promise<number> {
  const result = await executor.query(sql, params as unknown[]);
  return result.rowCount ?? 0;
}

export class SchemaDriftError extends Error {
  constructor(readonly sql: string, readonly rowIndex: number, readonly issues: string[]) {
    super(`Row ${rowIndex} did not match schema: ${issues.join("; ")}`);
    this.name = "SchemaDriftError";
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. Schemas — row shape (snake_case, DB types) → domain shape (camelCase)
// ═══════════════════════════════════════════════════════════════════════════

export const ORDER_STATUSES = ["pending", "paid", "shipped", "cancelled"] as const;
export type OrderStatus = (typeof ORDER_STATUSES)[number];

/** A numeric-string column (int8 / numeric) converted to a safe integer. */
const centsFromDb = z
  .union([z.string().regex(/^-?\d+$/), z.number().int()])
  .transform((value) => {
    const n = typeof value === "string" ? Number(value) : value;
    if (!Number.isSafeInteger(n)) throw new Error(`Amount out of safe range: ${value}`);
    return n;
  });

const orderRowSchema = z.object({
  id:               z.string().uuid(),
  org_id:           z.string().uuid(),
  user_id:          z.string().uuid(),
  status:           z.enum(ORDER_STATUSES),
  total_cents:      centsFromDb,          // int8 → number, validated
  payment_id:       z.string().nullable(),
  metadata:         z.object({ source: z.string(), campaignId: z.string().nullable() }), // jsonb
  created_at:       z.coerce.date(),
  updated_at:       z.coerce.date(),
  // joined columns:
  user_email:       z.string().email(),
  user_display_name: z.string(),
  item_count:       centsFromDb,          // COUNT(*) is int8 → string too
});

/** The camelCase domain object the rest of the app consumes. */
export const orderSchema = orderRowSchema.transform((row) => ({
  orderId:    row.id,
  orgId:      row.org_id,
  status:     row.status,
  totalCents: row.total_cents,
  paymentId:  row.payment_id,
  metadata:   row.metadata,
  createdAt:  row.created_at,
  updatedAt:  row.updated_at,
  user: {
    userId:      row.user_id,
    email:       row.user_email,
    displayName: row.user_display_name,
  },
  itemCount: row.item_count,
}));

export type Order = z.infer<typeof orderSchema>;

const orderItemRowSchema = z.object({
  id:         z.string().uuid(),
  order_id:   z.string().uuid(),
  sku:        z.string(),
  quantity:   z.number().int().positive(),
  unit_cents: centsFromDb,
});

export const orderItemSchema = orderItemRowSchema.transform((row) => ({
  orderItemId: row.id,
  sku:         row.sku,
  quantity:    row.quantity,
  unitCents:   row.unit_cents,
}));

export type OrderItem = z.infer<typeof orderItemSchema>;

// ═══════════════════════════════════════════════════════════════════════════
// 3. The shared SELECT — one definition, reused, mirroring orderRowSchema
// ═══════════════════════════════════════════════════════════════════════════

const ORDER_SELECT = `
  SELECT o.id,
         o.org_id,
         o.user_id,
         o.status,
         o.total_cents::text        AS total_cents,
         o.payment_id,
         o.metadata,
         o.created_at,
         o.updated_at,
         u.email                    AS user_email,
         u.display_name             AS user_display_name,
         COALESCE(i.item_count, 0)::text AS item_count
    FROM orders o
    JOIN users u ON u.id = o.user_id
    LEFT JOIN (
      SELECT order_id, COUNT(*) AS item_count
        FROM order_items
       GROUP BY order_id
    ) i ON i.order_id = o.id
`;

// ═══════════════════════════════════════════════════════════════════════════
// 4. Repository
// ═══════════════════════════════════════════════════════════════════════════

export interface ListOrdersFilter {
  orgId:      string;
  status?:    OrderStatus;
  userId?:    string;
  createdAfter?: Date;
  minTotalCents?: number;
  sortBy?:    "createdAt" | "totalCents";
  direction?: "asc" | "desc";
  limit?:     number;
  offset?:    number;
}

/** Allowlist: the ONLY columns that may appear in ORDER BY. */
const SORT_COLUMNS = {
  createdAt:  "o.created_at",
  totalCents: "o.total_cents",
} as const satisfies Record<string, string>;

export class OrderRepository {
  /**
   * Dynamic WHERE built from a typed filter. Every user value goes through the
   * params array; only allowlisted identifiers are ever interpolated.
   */
  async list(filter: ListOrdersFilter): Promise<Order[]> {
    const conditions: string[] = ["o.org_id = $1"];
    const params: unknown[] = [filter.orgId];

    if (filter.status !== undefined) {
      params.push(filter.status);
      conditions.push(`o.status = $${params.length}`);
    }
    if (filter.userId !== undefined) {
      params.push(filter.userId);
      conditions.push(`o.user_id = $${params.length}`);
    }
    if (filter.createdAfter !== undefined) {
      params.push(filter.createdAfter);
      conditions.push(`o.created_at > $${params.length}`);
    }
    if (filter.minTotalCents !== undefined) {
      params.push(filter.minTotalCents);
      conditions.push(`o.total_cents >= $${params.length}`);
    }

    const sortColumn = SORT_COLUMNS[filter.sortBy ?? "createdAt"];       // ✅ closed union
    const direction  = filter.direction === "asc" ? "ASC" : "DESC";      // ✅ closed union

    params.push(Math.min(filter.limit ?? 50, 200));
    const limitPlaceholder = `$${params.length}`;
    params.push(filter.offset ?? 0);
    const offsetPlaceholder = `$${params.length}`;

    return queryValidated(
      orderSchema,
      `${ORDER_SELECT}
        WHERE ${conditions.join(" AND ")}
        ORDER BY ${sortColumn} ${direction}
        LIMIT ${limitPlaceholder} OFFSET ${offsetPlaceholder}`,
      params,
    );
  }

  async findById(orgId: string, orderId: string): Promise<Order | null> {
    return queryOneValidated(
      orderSchema,
      `${ORDER_SELECT} WHERE o.org_id = $1 AND o.id = $2`,
      [orgId, orderId],
    );
  }

  async itemsFor(orderId: string, executor: Queryable = pool): Promise<OrderItem[]> {
    return queryValidated(
      orderItemSchema,
      `SELECT id, order_id, sku, quantity, unit_cents::text AS unit_cents
         FROM order_items WHERE order_id = $1 ORDER BY sku`,
      [orderId],
      executor,
    );
  }

  /**
   * Create an order and its items atomically, decrementing stock with a
   * conditional UPDATE so oversell is impossible even under concurrency.
   */
  async create(input: {
    orgId: string;
    userId: string;
    items: ReadonlyArray<{ sku: string; quantity: number }>;
    source: string;
  }): Promise<Order> {
    if (input.items.length === 0) throw new Error("An order needs at least one item");

    return withTransaction(async (tx) => {
      // (a) Lock and price the products in one round trip.
      const priceRowSchema = z.object({
        sku:         z.string(),
        price_cents: centsFromDb,
        stock_count: z.number().int(),
      });

      const skus = input.items.map((item) => item.sku);
      const priced = await queryValidated(
        priceRowSchema,
        `SELECT sku, price_cents::text AS price_cents, stock_count
           FROM products
          WHERE org_id = $1 AND sku = ANY($2::text[])
          ORDER BY sku
            FOR UPDATE`,
        [input.orgId, skus],
        tx,
      );

      const priceBySku = new Map(priced.map((row) => [row.sku, row]));
      for (const item of input.items) {
        if (!priceBySku.has(item.sku)) throw new Error(`Unknown SKU: ${item.sku}`);
      }

      const totalCents = input.items.reduce((sum, item) => {
        const product = priceBySku.get(item.sku);
        if (!product) throw new Error(`Unknown SKU: ${item.sku}`);   // narrows for TS
        return sum + product.price_cents * item.quantity;
      }, 0);

      // (b) Insert the order head.
      const insertedSchema = z.object({ id: z.string().uuid() });
      const inserted = await queryValidated(
        insertedSchema,
        `INSERT INTO orders (org_id, user_id, status, total_cents, metadata)
              VALUES ($1, $2, 'pending', $3, $4)
           RETURNING id`,
        [input.orgId, input.userId, totalCents, { source: input.source, campaignId: null }],
        tx,
      );
      const orderId = inserted[0]?.id;
      if (orderId === undefined) throw new Error("INSERT ... RETURNING produced no row");

      // (c) Bulk-insert the items with UNNEST — one statement, N rows.
      await execute(
        `INSERT INTO order_items (order_id, sku, quantity, unit_cents)
         SELECT $1, sku, quantity, unit_cents
           FROM UNNEST($2::text[], $3::int[], $4::bigint[])
             AS t(sku, quantity, unit_cents)`,
        [
          orderId,
          input.items.map((item) => item.sku),
          input.items.map((item) => item.quantity),
          input.items.map((item) => priceBySku.get(item.sku)?.price_cents ?? 0),
        ],
        tx,
      );

      // (d) Decrement stock conditionally — rowCount tells us if it succeeded.
      for (const item of input.items) {
        const updated = await execute(
          `UPDATE products
              SET stock_count = stock_count - $1
            WHERE org_id = $2 AND sku = $3 AND stock_count >= $1`,
          [item.quantity, input.orgId, item.sku],
          tx,
        );
        if (updated === 0) throw new OutOfStockError(item.sku);   // rolls the whole tx back
      }

      const order = await queryOneValidated(
        orderSchema,
        `${ORDER_SELECT} WHERE o.id = $1`,
        [orderId],
        tx,
      );
      if (!order) throw new Error("Newly created order not readable");
      return order;
    });
  }

  /** Idempotent payment capture: the status guard is IN the WHERE clause. */
  async markPaid(orgId: string, orderId: string, paymentId: string): Promise<Order> {
    return withTransaction(async (tx) => {
      const changed = await execute(
        `UPDATE orders
            SET status = 'paid', payment_id = $1, updated_at = now()
          WHERE org_id = $2 AND id = $3 AND status = 'pending'`,
        [paymentId, orgId, orderId],
        tx,
      );

      if (changed === 0) {
        // Either it doesn't exist, or it isn't pending. Distinguish the two:
        const current = await queryOneValidated(
          orderSchema,
          `${ORDER_SELECT} WHERE o.org_id = $1 AND o.id = $2`,
          [orgId, orderId],
          tx,
        );
        if (!current) throw new NotFoundError(`Order ${orderId} not found`);
        if (current.status === "paid" && current.paymentId === paymentId) return current; // idempotent replay
        throw new ConflictError(`Order ${orderId} is ${current.status}, cannot pay`);
      }

      const order = await queryOneValidated(
        orderSchema,
        `${ORDER_SELECT} WHERE o.id = $1`,
        [orderId],
        tx,
      );
      if (!order) throw new NotFoundError(`Order ${orderId} not found`);
      return order;
    });
  }

  /** Aggregate: every numeric column comes back as a string, so cast + validate. */
  async revenueByStatus(orgId: string): Promise<Array<{ status: OrderStatus; totalCents: number; count: number }>> {
    const rowSchema = z.object({
      status: z.enum(ORDER_STATUSES),
      total:  centsFromDb,
      count:  z.coerce.number().int(),
    });

    const rows = await queryValidated(
      rowSchema,
      `SELECT status,
              COALESCE(SUM(total_cents), 0)::text AS total,
              COUNT(*)::text                      AS count
         FROM orders
        WHERE org_id = $1
        GROUP BY status`,
      [orgId],
    );

    return rows.map((row) => ({ status: row.status, totalCents: row.total, count: row.count }));
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. HTTP layer — request validation in, ApiResponse out
// ═══════════════════════════════════════════════════════════════════════════

export type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: { code: string; message: string } };

const createOrderBodySchema = z.object({
  userId: z.string().uuid(),
  source: z.string().min(1).max(64),
  items: z
    .array(z.object({ sku: z.string().min(1), quantity: z.number().int().positive().max(999) }))
    .min(1)
    .max(100),
});

type CreateOrderBody = z.infer<typeof createOrderBodySchema>;

const orders = new OrderRepository();

export async function createOrderHandler(
  req: Request,
  res: Response<ApiResponse<Order>>,
): Promise<void> {
  const orgId = req.auth?.orgId;                     // set by auth middleware
  if (!orgId) {
    res.status(401).json({ ok: false, error: { code: "unauthenticated", message: "Missing token" } });
    return;
  }

  // req.body is `any` — parse it before it touches SQL. Same discipline as rows.
  const parsed = createOrderBodySchema.safeParse(req.body);
  if (!parsed.success) {
    res.status(400).json({
      ok: false,
      error: { code: "invalid_body", message: parsed.error.issues.map((i) => i.message).join("; ") },
    });
    return;
  }
  const requestBody: CreateOrderBody = parsed.data;

  try {
    const order = await orders.create({
      orgId,
      userId: requestBody.userId,
      items:  requestBody.items,
      source: requestBody.source,
    });
    res.status(201).json({ ok: true, data: order });
  } catch (error) {
    if (error instanceof OutOfStockError) {
      res.status(409).json({ ok: false, error: { code: "out_of_stock", message: error.message } });
      return;
    }
    if (error instanceof SchemaDriftError) {
      // A row didn't match its schema — this is OUR bug, page someone.
      logger.error("db schema drift", { sql: error.sql, issues: error.issues });
      res.status(500).json({ ok: false, error: { code: "internal", message: "Internal error" } });
      return;
    }
    if (error instanceof DatabaseError) {
      const message = error.code === "23505" ? "Duplicate order" : "Database error";
      res.status(error.code === "23505" ? 409 : 500).json({ ok: false, error: { code: error.code ?? "db", message } });
      return;
    }
    throw error;
  }
}

export class OutOfStockError extends Error {
  constructor(readonly sku: string) {
    super(`Out of stock: ${sku}`);
    this.name = "OutOfStockError";
  }
}
export class NotFoundError extends Error {}
export class ConflictError extends Error {}
```

---

## Going deeper

### The generic is an assertion — a proof by contradiction

If you remember one experiment from this document, make it this one. Run it against a real database:

```ts
interface ImaginaryRow extends QueryResultRow {
  totallyFakeColumn:  string;
  anotherInvention:   number;
  neverExisted:       Date;
}

const rows = await pool.query<ImaginaryRow>("SELECT 1 AS one");
console.log(rows.rows[0]?.totallyFakeColumn);   // undefined
console.log(rows.rows[0]?.anotherInvention);    // undefined
// TypeScript compiles this with zero complaints. The generic is a comment
// that happens to affect autocomplete.
```

The corollary: **treat `pool.query<T>` with the exact same code-review suspicion you'd give an `as T`.** A PR that changes a SQL string without changing the row interface deserves the same scrutiny as a PR that adds a cast.

### `rowCount` semantics per command

| Command | `rowCount` |
|---|---|
| `SELECT` | rows returned |
| `INSERT` without `RETURNING` | rows inserted |
| `INSERT ... ON CONFLICT DO NOTHING` (conflict hit) | `0` |
| `UPDATE` where the new values equal the old | still counts the row as updated |
| `DELETE` | rows deleted |
| `CREATE TABLE`, `BEGIN`, `SET` | `null` |
| Multi-statement string | only the **last** statement's count |

That last row matters more than it looks:

```ts
// pg allows multiple statements in one string ONLY when there are no parameters.
// You get a QueryResult[] instead of a QueryResult — and the type doesn't say so:
const result = await pool.query("SELECT 1; SELECT 2;");
// At runtime this is an ARRAY of results in some pg versions. Don't do this.
// Use a transaction with separate .query() calls instead.
```

### `UPDATE` reporting a row it didn't change

`rowCount` counts rows *matched and written*, not rows whose values differed. `UPDATE users SET is_active = TRUE WHERE id = $1` on an already-active user returns `1`. If you need "did anything actually change", say so in SQL:

```ts
const changed = await execute(
  "UPDATE users SET is_active = TRUE WHERE id = $1 AND is_active IS DISTINCT FROM TRUE",
  [userId],
);
// changed === 1 now genuinely means "the value changed"
```

### Nullability from `LEFT JOIN` is invisible to the type system

A `LEFT JOIN` makes every column from the joined table nullable — but only in the *result*, not in your interface:

```ts
// ❌ The interface came from the users table definition, where email is NOT NULL:
interface OrderWithUserRow extends QueryResultRow {
  id:         string;
  user_email: string;   // ← a lie under LEFT JOIN
}

await query<OrderWithUserRow>(`
  SELECT o.id, u.email AS user_email
    FROM orders o
    LEFT JOIN users u ON u.id = o.user_id
`);
// If an order has a dangling user_id, user_email is null and every
// `row.user_email.toLowerCase()` throws.

// ✅ The join dictates the nullability, not the table:
interface OrderWithUserRow2 extends QueryResultRow {
  id:         string;
  user_email: string | null;
}
```

Rule: **for every `LEFT JOIN`, every column from the right-hand table gets `| null` in the row interface.** No exceptions, no "but the FK is enforced" — because that's precisely the reasoning that produces the outage.

### `SELECT *` is a moving target

`SELECT *` binds your code to whatever the live schema happens to be. Two specific failure modes:

```ts
// (a) A migration ADDS a column. Your interface doesn't have it. Harmless —
//     but you're now shipping a column (maybe `internal_notes`, maybe
//     `password_hash`) across the wire and possibly straight into JSON.stringify.
const rows = await query<UserRow>("SELECT * FROM users WHERE id = $1", [userId]);
res.json(rows[0]);   // 🚨 leaks every column added since you wrote this line

// (b) A JOIN with SELECT * produces DUPLICATE column names. In pg, later
//     columns overwrite earlier ones in the row object:
await query("SELECT * FROM orders o JOIN users u ON u.id = o.user_id");
// Both tables have `id` and `created_at`. You get the USER's id and created_at.
// Silently. Your order ids are now user ids.
```

Explicit column lists cost twenty seconds and remove both.

### `pg` pool exhaustion: the class of bug types can't see

```ts
// ❌ Client never released on the throw path — 20 requests later the pool is dead
//    and every subsequent query hangs forever (no timeout by default).
async function broken(userId: string): Promise<void> {
  const client = await pool.connect();
  const rows = await client.query("SELECT ...", [userId]);   // throws?
  if (rows.rowCount === 0) throw new Error("nope");          // client leaked
  client.release();
}

// ✅ try/finally, always. Or never call pool.connect() outside withTransaction().
```

Two hardening settings worth having so a leak fails fast instead of hanging:

```ts
new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  connectionTimeoutMillis: 5_000,   // fail if no client available in 5s
  idleTimeoutMillis: 30_000,
  statement_timeout: 10_000,        // server-side: kill runaway queries
  query_timeout: 10_000,            // client-side backstop
});
```

The strongest structural fix is to make `pool.connect()` private to your db module and expose only `query` / `withTransaction`. Then leaking a client is not something application code *can* do.

### `DatabaseError` and typed error codes

`pg` throws `DatabaseError`, which carries the Postgres `SQLSTATE` in `.code`. Model the codes you handle as a union rather than sprinkling magic strings:

```ts
import { DatabaseError } from "pg";

export const PG_ERROR = {
  uniqueViolation:     "23505",
  foreignKeyViolation: "23503",
  notNullViolation:    "23502",
  checkViolation:      "23514",
  serializationFailure:"40001",   // retryable under SERIALIZABLE
  deadlockDetected:    "40P01",   // retryable
  lockNotAvailable:    "55P03",
} as const;

export type PgErrorCode = (typeof PG_ERROR)[keyof typeof PG_ERROR];

export function isPgError(error: unknown, code: PgErrorCode): error is DatabaseError {
  return error instanceof DatabaseError && error.code === code;
}

// Retryable transactions become expressible:
export async function withRetryableTransaction<T>(
  fn: (tx: Tx) => Promise<T>,
  attempts = 3,
): Promise<T> {
  for (let attempt = 1; attempt <= attempts; attempt++) {
    try {
      return await withTransaction(fn);
    } catch (error) {
      const retryable =
        isPgError(error, PG_ERROR.serializationFailure) || isPgError(error, PG_ERROR.deadlockDetected);
      if (!retryable || attempt === attempts) throw error;
      await new Promise((resolve) => setTimeout(resolve, 50 * 2 ** attempt));
    }
  }
  throw new Error("unreachable");
}
```

Also note `error` in a `catch` is `unknown` under `useUnknownInCatchVariables` (implied by `strict`) — the `instanceof DatabaseError` check is what narrows it. See *41 — Type guards*.

### Template-literal types can *almost* check your SQL

You can get partial column checking with template-literal types — enough to catch a `SELECT *`, not enough to validate a real query:

```ts
type NoSelectStar<S extends string> =
  Lowercase<S> extends `${string}select *${string}` ? never : S;

async function safeQuery<TRow extends QueryResultRow, S extends string>(
  sql: NoSelectStar<S>,
  params: readonly unknown[] = [],
): Promise<TRow[]> {
  const result = await pool.query<TRow>(sql, params as unknown[]);
  return result.rows;
}

await safeQuery<UserRow, "SELECT id, email FROM users">("SELECT id, email FROM users");  // ✅
// await safeQuery("SELECT * FROM users");
//                 ~~~~~~~~~~~~~~~~~~~~~ ❌ not assignable to 'never'
```

Cute, and genuinely useful as a lint. But full SQL parsing in the type system is not a road you want to walk by hand — which brings us to the tools that already did.

### What Kysely, Prisma and pgtyped actually solve

All three attack the *same single problem*: **the row generic is unverified**. They just attack it from different directions.

| Tool | Approach | Source of truth | You still write SQL? | Verifies your query against the real schema? |
|---|---|---|---|---|
| **raw `pg`** | you assert `T` | your head | yes | ❌ never |
| **pgtyped** | reads your `.sql` files, connects to the DB at build time, emits row + param types | the live database | ✅ yes, real SQL | ✅ at build time |
| **Kysely** | a typed query builder; the SQL is constructed from a `Database` interface | a TS `Database` interface (usually codegen'd from the DB) | no — you build it fluently | ✅ at compile time |
| **Prisma** | full ORM + its own schema language + generated client | `schema.prisma` | rarely (`$queryRaw` escape hatch) | ✅ at compile time |
| **Drizzle** | typed query builder close to SQL, schema declared in TS | TS schema files | SQL-like builder | ✅ at compile time |

What each buys you, concretely:

```ts
// ── pgtyped: you keep writing SQL; the types are GENERATED from it ─────────
/* queries/findUserByEmail.sql
   -- @name findUserByEmail
   SELECT id, email, created_at FROM users WHERE email = :email!;
*/
import { findUserByEmail } from "./queries/findUserByEmail.queries";
const rows = await findUserByEmail.run({ email: "ada@example.com" }, pool);
// rows: Array<{ id: string; email: string; created_at: Date }>  ← DERIVED, not asserted.
// Rename a column in a migration → regeneration produces compile errors everywhere.

// ── Kysely: the SQL is BUILT, so the result type is computed ───────────────
import { Kysely } from "kysely";
interface Database {
  users:  { id: string; email: string; org_id: string; created_at: Date };
  orders: { id: string; user_id: string; total_cents: string; status: OrderStatus };
}
const db = new Kysely<Database>({ /* dialect */ });

const rows2 = await db
  .selectFrom("orders")
  .innerJoin("users", "users.id", "orders.user_id")
  .select(["orders.id", "orders.total_cents", "users.email"])
  .where("orders.status", "=", "paid")
  .execute();
// rows2: Array<{ id: string; total_cents: string; email: string }> ← INFERRED from
// the select list. Add a column to .select() and the type changes automatically.
// Typo a column name and it's a compile error.

// ── Prisma: no SQL at all; the client is generated from schema.prisma ──────
const order = await prisma.order.findUnique({
  where:   { id: orderId },
  include: { user: true, items: true },
});
// order: (Order & { user: User; items: OrderItem[] }) | null — fully derived.
```

Honest guidance:

- **Stay on raw `pg`** when your queries are complex/hand-tuned, the team knows SQL, and you're willing to pay for discipline (explicit columns + zod at the boundary). This document is that path done properly.
- **Add pgtyped** if you love your SQL files but hate maintaining row interfaces. It is the smallest possible step from here — you keep every query you already wrote.
- **Move to Kysely** if you want compile-time-verified queries without an ORM's opinions, and you're comfortable with a fluent builder. It's the sweet spot for most typed-SQL teams.
- **Move to Prisma/Drizzle** if you want migrations, relations and the client generated for you and your query shapes are mostly CRUD.

Whatever you choose, the raw-`pg` skills here remain load-bearing: every one of these tools has a `$queryRaw` / `sql` escape hatch for the query the builder can't express — and inside that escape hatch, you are back to an unchecked row generic.

---

## Common mistakes

### Mistake 1 — Trusting the row generic instead of the SELECT list

```ts
// ❌ The interface claims columns the query never selects:
interface UserRow extends QueryResultRow {
  id: string; email: string; is_admin: boolean; password_hash: string;
}

const rows = await pool.query<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);
const user = rows.rows[0];
if (user && !user.is_admin) throw new ForbiddenError();
// is_admin is undefined → !undefined is true → EVERY user is rejected.
// Flip the condition and every user is ADMITTED. No compile error either way.

// ✅ The SELECT list and the interface are written together, and validated:
const adminCheckSchema = z.object({ id: z.string().uuid(), is_admin: z.boolean() });

const [row] = await queryValidated(
  adminCheckSchema,
  "SELECT id, is_admin FROM users WHERE id = $1",   // mirrors the schema exactly
  [userId],
);
if (!row) throw new NotFoundError(`User ${userId} not found`);
if (!row.is_admin) throw new ForbiddenError();      // is_admin is PROVEN to be a boolean
```

### Mistake 2 — Running a transaction on the pool instead of a client

```ts
// ❌ Each pool.query() may grab a DIFFERENT connection. The UPDATE can land
//    outside the transaction, and the COMMIT may commit nothing.
export async function transferBroken(from: string, to: string, cents: number): Promise<void> {
  await pool.query("BEGIN");
  await pool.query("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2", [cents, from]);
  await pool.query("UPDATE accounts SET balance_cents = balance_cents + $1 WHERE id = $2", [cents, to]);
  await pool.query("COMMIT");
}

// ✅ One checked-out client for every statement, released in `finally`:
export async function transfer(from: string, to: string, cents: number): Promise<void> {
  await withTransaction(async (tx) => {
    await execute("UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2", [cents, from], tx);
    await execute("UPDATE accounts SET balance_cents = balance_cents + $1 WHERE id = $2", [cents, to], tx);
  });
}

// ✅✅ And with the branded Tx type, the broken version doesn't even compile:
async function debit(tx: Tx, accountId: string, cents: number): Promise<void> { /* ... */ }
// await debit(pool, "acct_1", 100);
//             ~~~~ Argument of type 'Pool' is not assignable to parameter of type 'Tx'
```

### Mistake 3 — Typing `bigint` / `numeric` / `COUNT(*)` as `number`

```ts
// ❌ pg returns int8 and numeric as STRINGS. This type is a lie:
interface StatsRow extends QueryResultRow { user_count: number; revenue_cents: number }

const [stats] = await query<StatsRow>(
  "SELECT COUNT(*) AS user_count, SUM(total_cents) AS revenue_cents FROM orders WHERE org_id = $1",
  [orgId],
);
const average = (stats?.revenue_cents ?? 0) / (stats?.user_count ?? 1);
// "482300" / "9" → JS coerces on `/`, so this ACCIDENTALLY works...
const total = (stats?.revenue_cents ?? 0) + 500;
// ...but `+` concatenates: "482300500". Silently, in a billing report.

// ✅ Type them as they arrive, convert once, explicitly:
interface StatsRow2 extends QueryResultRow { user_count: string; revenue_cents: string | null }

const [stats2] = await query<StatsRow2>(
  `SELECT COUNT(*)::text                      AS user_count,
          COALESCE(SUM(total_cents), 0)::text AS revenue_cents
     FROM orders WHERE org_id = $1`,
  [orgId],
);
const userCount    = Number(stats2?.user_count ?? "0");
const revenueCents = Number(stats2?.revenue_cents ?? "0");
// const bad = revenueCents + "500";  ❌ now a type error if you slip
```

### Mistake 4 — Assuming `rows[0]` exists

```ts
// ❌ Without noUncheckedIndexedAccess this compiles and crashes on an empty result:
export async function getUserEmail(userId: string): Promise<string> {
  const rows = await query<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);
  return rows[0].email;   // TypeError: Cannot read properties of undefined
}

// ✅ Enable noUncheckedIndexedAccess, then handle the miss explicitly:
export async function getUserEmail(userId: string): Promise<string> {
  const rows = await query<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);
  const user = rows[0];                       // UserRow | undefined
  if (!user) throw new NotFoundError(`User ${userId} not found`);
  return user.email;
}

// ✅✅ Better: encode the cardinality in the helper you call, not at the call site:
export async function getUserEmail2(userId: string): Promise<string> {
  const user = await queryOne<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);
  return user.email;                          // queryOne throws if not exactly 1
}
```

### Mistake 5 — Interpolating values into SQL

```ts
// ❌ SQL injection, and it defeats query-plan caching too:
const email = requestBody.email;   // "'; DROP TABLE users; --"
await pool.query(`SELECT id FROM users WHERE email = '${email}'`);

// ❌ Still wrong — the `sql` tag here is just template concatenation:
await pool.query(`DELETE FROM sessions WHERE user_id = ${userId}`);

// ✅ Parameters. The values travel out-of-band; the server never parses them as SQL:
await pool.query("SELECT id FROM users WHERE email = $1", [email]);
await pool.query("DELETE FROM sessions WHERE user_id = $1", [userId]);

// ✅ Lists use = ANY($1::type[]) — NOT `IN (${ids.join(",")})`:
await pool.query("SELECT id, email FROM users WHERE id = ANY($1::uuid[])", [userIds]);

// ⚠️ Identifiers cannot be parameterised. Allowlist them through a closed union:
const SORT_COLUMNS = { createdAt: "created_at", email: "email" } as const;
type SortKey = keyof typeof SORT_COLUMNS;
function orderBy(key: SortKey, dir: "asc" | "desc"): string {
  return `ORDER BY ${SORT_COLUMNS[key]} ${dir === "asc" ? "ASC" : "DESC"}`;
}
```

### Mistake 6 — Forgetting `| null` on nullable columns

```ts
// ❌ The column is `last_login_at timestamptz NULL`, but the interface says Date:
interface UserRow extends QueryResultRow { id: string; last_login_at: Date }

const user = await queryOne<UserRow>("SELECT id, last_login_at FROM users WHERE id = $1", [userId]);
const daysSince = (Date.now() - user.last_login_at.getTime()) / 86_400_000;
// TypeError on a user who has never logged in. Compiler saw nothing wrong.

// ✅ Nullable in the DB → `| null` in the interface, and the compiler forces a branch:
interface UserRow2 extends QueryResultRow { id: string; last_login_at: Date | null }

const user2 = await queryOne<UserRow2>("SELECT id, last_login_at FROM users WHERE id = $1", [userId]);
const daysSince2 = user2.last_login_at
  ? (Date.now() - user2.last_login_at.getTime()) / 86_400_000
  : null;

// ✅ Or push the default into SQL and legitimately keep the type non-nullable:
//    SELECT id, COALESCE(last_login_at, created_at) AS last_login_at FROM users ...
```

### Mistake 7 — Leaking a client on the error path

```ts
// ❌ Every throw between connect() and release() permanently removes one
//    connection from the pool. After `max` failures, the app hangs.
async function report(orgId: string): Promise<number> {
  const client = await pool.connect();
  const result = await client.query("SELECT COUNT(*) AS c FROM orders WHERE org_id = $1", [orgId]);
  const count = Number(result.rows[0].c);   // throws on empty → client never released
  client.release();
  return count;
}

// ✅ try/finally — release is unconditional:
async function report2(orgId: string): Promise<number> {
  const client = await pool.connect();
  try {
    const result = await client.query<{ c: string }>(
      "SELECT COUNT(*) AS c FROM orders WHERE org_id = $1", [orgId],
    );
    return Number(result.rows[0]?.c ?? "0");
  } finally {
    client.release();
  }
}

// ✅✅ Best: don't check out clients in application code at all. Use pool.query()
//     for single statements and withTransaction() for everything else.
```

---

## Practice exercises

### Exercise 1 — easy

Build a **typed sessions table access layer** on raw `pg`.

Requirements:

1. Write a `SessionRow` interface that `extends QueryResultRow` and mirrors this table exactly, in `snake_case`:
   ```sql
   CREATE TABLE sessions (
     id           uuid PRIMARY KEY,
     user_id      uuid NOT NULL,
     auth_token   text NOT NULL UNIQUE,
     ip_address   text,                       -- nullable
     expires_at   timestamptz NOT NULL,
     revoked_at   timestamptz,                -- nullable
     created_at   timestamptz NOT NULL DEFAULT now()
   );
   ```
2. Write a camelCase domain interface `Session` with `sessionId`, `userId`, `authToken`, `ipAddress`, `expiresAt`, `revokedAt`, `createdAt` and a `toSession(row: SessionRow): Session` mapper.
3. Write a `SESSION_COLUMNS` string constant containing the explicit column list, and use it in every query (no `SELECT *`).
4. Implement, each with a fully annotated signature:
   - `createSession(userId: string, authToken: string, ipAddress: string | null, ttlSeconds: number): Promise<Session>` using `INSERT ... RETURNING`.
   - `findActiveSession(authToken: string): Promise<Session | null>` — must exclude revoked and expired rows in SQL, not in JS.
   - `revokeSession(sessionId: string): Promise<boolean>` — return whether a row was actually revoked, using `rowCount ?? 0`.
   - `countActiveSessionsForUser(userId: string): Promise<number>` — remember `COUNT(*)` returns a **string**.
5. Enable `noUncheckedIndexedAccess` mentally: no `rows[0].x` without a guard, and no `!` assertions anywhere.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a **validated, camelCased query layer** with zod and a transaction helper.

Requirements:

1. Write a generic helper `query<TRow extends QueryResultRow>(sql, params, executor)` plus `queryOne`, `queryMaybeOne` and `execute`, all accepting an optional `executor: Queryable` defaulting to the pool.
2. Write `withTransaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T>` where `Tx` is a **branded** `PoolClient`, with `BEGIN`/`COMMIT`/`ROLLBACK` and a `finally { client.release() }`.
3. Write the type-level helpers `SnakeToCamel<S extends string>` and `CamelCaseKeys<T>` (template-literal + mapped types), plus the runtime `camelizeRow` / `camelizeRows` counterparts.
4. Define zod schemas for this `invoices` table and derive the row type with `z.infer`:
   ```sql
   invoices(id uuid, org_id uuid, customer_id uuid, status text
            CHECK (status IN ('draft','sent','paid','void')),
            amount_cents bigint NOT NULL,      -- arrives as a STRING
            tax_cents bigint NOT NULL,
            due_date date, paid_at timestamptz, line_items jsonb NOT NULL,
            created_at timestamptz NOT NULL)
   ```
   `line_items` is `Array<{ description: string; quantity: number; unitCents: number }>`.
5. Write `queryValidated(schema, sql, params, executor)` that `safeParse`s every row and throws a `SchemaDriftError` naming the failing field path and row index.
6. Implement `issueInvoice(orgId, customerId, lineItems)` that, **in one transaction**: inserts the invoice with a computed `amount_cents` and `tax_cents`, inserts an `invoice_events` audit row, and returns the validated camelCase `Invoice`. Throwing anywhere must roll back everything.
7. Prove three compile errors: passing `pool` where `Tx` is required; reading `invoice.amount_cents` on the camelCase domain type; and adding an extra element to a fixed-arity params tuple.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **type-safe query registry** where the SQL, the parameter tuple, the row schema and the domain mapper are declared together and cannot drift.

Requirements:

1. Define:
   ```ts
   interface QueryDefinition<TParams extends readonly unknown[], TDomain> {
     readonly name:   string;                // used as the pg prepared-statement name
     readonly text:   string;                // the SQL, with $1..$n
     readonly params: (input: never) => TParams;   // you decide the real signature
     readonly rowSchema: z.ZodTypeAny;       // validates each raw row
     readonly toDomain: (row: unknown) => TDomain;
     readonly cardinality: "many" | "one" | "maybeOne" | "none";
   }
   ```
   Refine these types yourself so that `defineQuery(...)` **infers** `TParams` and `TDomain` from the arguments with no manual annotation at the call site.
2. Write an overloaded `runQuery` whose **return type depends on `cardinality`**: `"many"` → `TDomain[]`, `"one"` → `TDomain` (throws on 0 or >1), `"maybeOne"` → `TDomain | null`, `"none"` → `number` (the row count). Use a conditional type, not four separate functions. (See *44 — Conditional types*.)
3. Add a dev-mode guard: after the first execution of each definition, compare `QueryResult.fields.map(f => f.name)` against the keys of `rowSchema` (a `z.ZodObject`) and throw a descriptive error listing missing and unexpected columns. It must be a no-op when `NODE_ENV === "production"`.
4. Add a `paramCount` compile-time sanity check: write a template-literal type `HighestPlaceholder<S extends string>` that extracts the largest `$n` in the SQL string, and constrain `TParams["length"]` to equal it. (Partial credit: get it working for `$1`–`$9`.)
5. Build the registry:
   ```ts
   export const queries = {
     findUserByAuthToken: defineQuery({ /* ... */ }),
     listOrdersForOrg:    defineQuery({ /* ... */ }),
     markOrderPaid:       defineQuery({ /* ... */ }),
   } as const;
   ```
   and a `runAll` type test proving that `runQuery(queries.findUserByAuthToken, [authToken])` is `Promise<User | null>` while `runQuery(queries.listOrdersForOrg, [orgId, 50])` is `Promise<Order[]>`.
6. Make every definition usable inside `withTransaction` by threading the optional `Tx` executor through `runQuery`, and write a `payOrder(orderId, paymentId)` that uses three registry queries in one transaction.
7. Finally, write a short comparison in comments: for each of steps 1–4, note which of **pgtyped**, **Kysely** and **Prisma** gives you that guarantee for free, and what you gave up to hand-roll it.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
import {
  Pool, PoolClient, Client, QueryResult, QueryResultRow,
  QueryConfig, DatabaseError, types as pgTypes,
} from "pg";

// ── Pool ────────────────────────────────────────────────────────────────────
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20, idleTimeoutMillis: 30_000, connectionTimeoutMillis: 5_000,
});
pool.on("error", (error) => logger.error("idle client error", { error }));

// ── Row types ───────────────────────────────────────────────────────────────
interface UserRow extends QueryResultRow {   // `extends QueryResultRow` = index signature
  id:         string;        // uuid        → string
  email:      string;        // text        → string
  balance:    string;        // int8/numeric→ STRING (not number)
  is_active:  boolean;       // bool        → boolean
  metadata:   { source: string };  // jsonb  → already parsed (typed by ASSERTION)
  deleted_at: Date | null;   // nullable    → | null
}

// ── Querying ────────────────────────────────────────────────────────────────
const r = await pool.query<UserRow>("SELECT id, email FROM users WHERE id = $1", [userId]);
r.rows;       // UserRow[]        ⚠️ UNCHECKED ASSERTION — same trust as `as UserRow[]`
r.rowCount;   // number | null    ← nullable; use `?? 0`
r.fields;     // FieldDef[]       ← the only real runtime shape info

// Typed params tuple (second generic):
await pool.query<UserRow, [userId: string]>("SELECT id FROM users WHERE id = $1", [userId]);

// ── Helpers ─────────────────────────────────────────────────────────────────
query<T>(sql, params, executor?)        // → T[]
queryOne<T>(sql, params, executor?)     // → T        (throws unless exactly 1)
queryMaybeOne<T>(sql, params, executor?)// → T | null
execute(sql, params, executor?)         // → number   (rowCount ?? 0)

// ── Transactions ────────────────────────────────────────────────────────────
declare const txBrand: unique symbol;
type Tx = PoolClient & { readonly [txBrand]: true };

await withTransaction(async (tx) => {
  await execute("UPDATE accounts SET balance = balance - $1 WHERE id = $2", [100, from], tx);
  await execute("UPDATE accounts SET balance = balance + $1 WHERE id = $2", [100, to], tx);
});
// BEGIN → fn(tx) → COMMIT | catch → ROLLBACK | finally → client.release()
// ❌ NEVER pool.query("BEGIN") — statements may land on different connections.

// ── Validation at the boundary ──────────────────────────────────────────────
const rowSchema = z.object({ id: z.string().uuid(), created_at: z.coerce.date() });
const rows = await queryValidated(rowSchema, "SELECT id, created_at FROM users");
// → z.infer<typeof rowSchema>[]  ✅ PROVEN, not asserted

// ── snake_case → camelCase at the type level ────────────────────────────────
type SnakeToCamel<S extends string> =
  S extends `${infer H}_${infer T}` ? `${H}${Capitalize<SnakeToCamel<T>>}` : S;
type CamelCaseKeys<T> = { [K in keyof T as K extends string ? SnakeToCamel<K> : K]: T[K] };

// ── Postgres → JS type surprises ────────────────────────────────────────────
// int8 / bigint / numeric / COUNT(*) / SUM(...)  → string
// timestamptz / timestamp / date                → Date
// json / jsonb                                  → parsed object (typed by assertion)
// int4[] / text[]                               → number[] / string[]
// bytea                                         → Buffer
// SUM over zero rows                            → null (use COALESCE)

// ── Error codes ─────────────────────────────────────────────────────────────
error instanceof DatabaseError && error.code === "23505"  // unique_violation
// 23503 fk_violation · 23502 not_null · 23514 check · 40001 serialization · 40P01 deadlock

// ── tsconfig ────────────────────────────────────────────────────────────────
// { "strict": true, "noUncheckedIndexedAccess": true }
//   → rows[0] is T | undefined  ✅  prefer for...of and .map() over indexed loops
```

| Symbol | What it is | Gotcha |
|---|---|---|
| `QueryResult<T>` | `{ rows: T[]; rowCount: number \| null; fields: FieldDef[] }` | `rowCount` is nullable |
| `QueryResultRow` | `{ [column: string]: any }` — the generic's constraint | interfaces need `extends QueryResultRow` |
| `pool.query<T>()` | Runs SQL, returns `QueryResult<T>` | **`T` is an unchecked assertion** |
| `pool.query<T, P>()` | Second generic pins the params tuple | default `P = any[]` checks nothing |
| `PoolClient` | One checked-out connection | required for transactions; must be `release()`d |
| `QueryConfig` | `{ name, text, values }` object form | `name` enables server-side prepared statements |
| `DatabaseError` | The thrown error type | `.code` is the SQLSTATE string |
| `FieldDef` | Per-column metadata on the result | the only runtime shape info you get |
| `noUncheckedIndexedAccess` | Makes `rows[0]` be `T \| undefined` | also affects `Record` access and `arr[i]` |
| `z.infer<typeof schema>` | Type derived from a zod schema | schema and type can no longer drift |

---

## Connected topics

- **56 — Typing database models with Mongoose** — the same boundary problem on the NoSQL side; `.lean<T>()` and `aggregate<T>()` are unchecked assertions for exactly the same reason `pool.query<T>()` is.
- **28 — Generic interfaces** — `QueryResult<T>` and the `query<TRow>()` helper are generic interfaces and generic functions; understanding the constraint `TRow extends QueryResultRow` makes pg's error messages readable.
- **32 — Utility types** — `Pick`, `Omit` and `Record` build the row-subset types for projections, insert inputs and the sort-column allowlists.
- **41 — Type guards** — `isPgError(error, code): error is DatabaseError` is how you narrow an `unknown` catch variable into a typed Postgres error.
- **38 — Branded types** — the `Tx = PoolClient & { readonly [txBrand]: true }` trick that makes "this must run inside a transaction" a compile-time guarantee.
- **44 — Conditional types** — used for the cardinality-dependent return type of `runQuery` in Exercise 3.
- **43 — Mapped types** and **46 — Template literal types** — together they give you `CamelCaseKeys<T>` and `SnakeToCamel<S>`, converting a snake_case row type into a camelCase domain type with zero duplication.
- **50 — Runtime validation with zod** — `queryValidated` turns the row generic from a belief into a proof; the same schema style validates `requestBody` at the HTTP edge.
