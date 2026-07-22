# 66 — Builder Pattern in TypeScript

## What is this?

The **builder pattern** is a way to construct a complex value through a sequence of small, named, chained calls instead of one giant constructor call or one enormous options object.

```ts
const query = createQuery("users")
  .select("userId", "email", "createdAt")   // narrows the row type
  .where("status", "=", "active")           // accumulates a typed condition
  .where("createdAt", ">", new Date("2026-01-01"))
  .orderBy("createdAt", "desc")
  .limit(50)
  .build();                                 // only compiles if the shape is valid
```

In JavaScript that is just method chaining — every method returns `this`, and you find out at runtime whether you forgot something. In TypeScript, the builder becomes something categorically more powerful, because **the type of the builder can change with every call**:

```ts
// The type parameter tracks WHICH fields have been set so far.
declare const b0: RequestBuilder<never>;                       // nothing set
const b1 = b0.url("https://api.example.com/users");            // RequestBuilder<"url">
const b2 = b1.method("POST");                                  // RequestBuilder<"url" | "method">
const b3 = b2.json({ email: "ada@example.com" });              // RequestBuilder<"url" | "method" | "body">

b1.build();   // ❌ Error — 'method' has not been set yet
b3.build();   // ✅ compiles
```

That technique has a name: **type-state**. The builder's *value* is one object, but its *type* encodes a little state machine, and the compiler refuses transitions that don't make sense. Combine it with:

- **`this` return types** — so chaining survives subclassing,
- **generic result narrowing** — so `.select("email")` changes the type of what `.build()` eventually returns,
- **branded / phantom types** — so a "validated" builder is a genuinely different type from an unvalidated one,

...and you get APIs where "I forgot to set the table name" is a red squiggle in your editor rather than a 3 a.m. page.

This document is about all four techniques, plus the equally important skill of knowing when *not* to build a builder.

## Why does it matter?

On a backend, the things you construct are rarely simple:

- **SQL / query objects.** A query needs a table, optional columns, zero-or-more conditions, optional joins, ordering, pagination. Encoding "table is mandatory, everything else is optional, and `.offset()` without `.limit()` is meaningless" in a single constructor signature is miserable. In a builder it's natural.
- **HTTP clients.** `fetchJson(url, method, headers, body, timeout, retries, signal, auth)` — eight positional parameters, half of them optional, three of them interdependent (`body` is invalid on `GET`, `retries` is meaningless without `timeout`).
- **Response construction.** `ApiResponse` objects with status, headers, body, cache directives, and correlation IDs, where "you set a body but never a status" should be impossible.
- **Test fixtures.** `aUser().withRole("admin").withVerifiedEmail().build()` reads far better than a 20-key literal repeated in 200 test files.
- **Pipelines and middleware chains** where each `.use()` can *add* to a typed context object that the final handler sees.

The payoff has three parts.

**1. Discoverability.** After you type `.`, autocomplete shows exactly the operations legal *at that point*. With a plain options object, autocomplete shows all 20 keys at once with no ordering guidance.

**2. Impossible states become uncompilable.** The classic bug — a required field silently `undefined` — moves from runtime to compile time. Not by adding a runtime check (though you can keep one), but by making the call not typecheck at all.

**3. Progressive type refinement.** This is the part that has no JavaScript equivalent. `.select("userId", "email")` doesn't just record two strings; it *changes the type of the result rows* to `{ userId: number; email: string }`. The compiler tracks your data shape through the chain, so downstream code gets exact types with zero manual annotation.

The cost is real too: builders are more code, harder to read as an implementer, and produce type errors that can be genuinely cryptic. The last section of this doc is about when a plain options object wins.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript: a query builder that looks fine and lies constantly ──────────

class QueryBuilder {
  constructor() {
    this.table = null;
    this.columns = [];
    this.conditions = [];
    this.orderByClause = null;
    this.limitValue = null;
  }

  from(table)                 { this.table = table; return this; }
  select(...columns)          { this.columns = columns; return this; }
  where(column, op, value)    { this.conditions.push({ column, op, value }); return this; }
  orderBy(column, direction)  { this.orderByClause = { column, direction }; return this; }
  limit(n)                    { this.limitValue = n; return this; }

  build() {
    // Every guarantee lives here, at RUNTIME, hopefully:
    if (!this.table) throw new Error("No table specified");
    const cols = this.columns.length ? this.columns.join(", ") : "*";
    let sql = `SELECT ${cols} FROM ${this.table}`;
    // ...
    return { sql, params: this.conditions.map((c) => c.value) };
  }
}

// Failure mode 1 — forgot .from(). Discovered in production, at request time.
new QueryBuilder().select("email").limit(10).build();
// 💥 Error: No table specified

// Failure mode 2 — typo in a column name. Nothing catches it.
new QueryBuilder().from("users").select("emial").build();
// → SELECT emial FROM users  →  Postgres error 42703 at runtime

// Failure mode 3 — wrong type for a condition value. Nothing catches it.
new QueryBuilder().from("users").where("createdAt", ">", "yesterday").build();
// → invalid input syntax for type timestamp

// Failure mode 4 — the RESULT is untyped. Every caller re-invents the shape:
const rows = await db.run(new QueryBuilder().from("users").select("userId", "email").build());
rows[0].emailAddress;   // undefined. No error. Bug ships.

// Failure mode 5 — calling .build() twice, or mutating after build:
const shared = new QueryBuilder().from("users");
const activeQuery  = shared.where("status", "=", "active").build();
const deletedQuery = shared.where("status", "=", "deleted").build();
// 💥 deletedQuery has BOTH conditions — the builder is mutable and shared.
```

```ts
// ── TypeScript: the same five failures, all caught before the code runs ──────

// 1. Declare the table shapes once. This is the single source of truth.
interface UsersTable {
  userId:    number;
  email:     string;
  status:    "active" | "suspended" | "deleted";
  createdAt: Date;
}

interface Schema {
  users: UsersTable;
}

declare function createQuery<TableName extends keyof Schema>(
  table: TableName,
): QueryBuilder<Schema[TableName], keyof Schema[TableName]>;

// 2. The table is a CONSTRUCTOR argument — it cannot be forgotten.
//    Failure mode 1 is now structurally impossible.
const q = createQuery("users");

// 3. Column names are checked against the table type.
createQuery("users").select("emial");
// ❌ Argument of type '"emial"' is not assignable to parameter of type
//    'keyof UsersTable'. Did you mean '"email"'?

// 4. Condition values are checked against the COLUMN's type.
createQuery("users").where("createdAt", ">", "yesterday");
// ❌ Argument of type 'string' is not assignable to parameter of type 'Date'.

createQuery("users").where("status", "=", "activated");
// ❌ Type '"activated"' is not assignable to type '"active" | "suspended" | "deleted"'.

// 5. The RESULT type follows the selection automatically.
const built = createQuery("users")
  .select("userId", "email")
  .where("status", "=", "active")
  .build();

type Row = typeof built.__rowType;
// = { userId: number; email: string }
//   ↑ not `any`, not a hand-written interface — derived from the chain itself.

// 6. Immutable builder: each call returns a NEW builder, so branching is safe.
const base    = createQuery("users").select("userId", "email");
const active  = base.where("status", "=", "active").build();
const deleted = base.where("status", "=", "deleted").build();
// ✅ `active` has one condition, `deleted` has one condition. No cross-talk.
```

The revelation: in JavaScript, a builder is a *runtime validator wearing a fluent costume* — every guarantee is an `if (!x) throw` inside `build()`. In TypeScript, the guarantees move into the type of the builder itself. `build()` becomes a formality that mostly just assembles a string, because the compiler already proved the inputs were complete and well-typed.

---

## Syntax

```ts
// ═══════════════════════════════════════════════════════════════════════════
// 1. The `this` return type — polymorphic chaining that survives subclassing
// ═══════════════════════════════════════════════════════════════════════════

class BaseRequestBuilder {
  protected headers: Record<string, string> = {};

  header(name: string, value: string): this {   // `this`, not BaseRequestBuilder
    this.headers[name] = value;
    return this;
  }
}

class AuthedRequestBuilder extends BaseRequestBuilder {
  bearer(authToken: string): this {
    return this.header("Authorization", `Bearer ${authToken}`);
  }
}

new AuthedRequestBuilder().header("Accept", "application/json").bearer("tok_123");
//                        ↑ returns AuthedRequestBuilder, so .bearer() is visible ✅
// If `header` had returned `BaseRequestBuilder`, .bearer() would be an error.

// ═══════════════════════════════════════════════════════════════════════════
// 2. Type-state — a generic parameter recording which fields are set
// ═══════════════════════════════════════════════════════════════════════════

type RequiredField = "url" | "method";

declare class TypeStateBuilder<TSet extends RequiredField = never> {
  url(value: string): TypeStateBuilder<TSet | "url">;
  method(value: "GET" | "POST"): TypeStateBuilder<TSet | "method">;

  // The self-type constraint is the whole trick:
  build(this: TypeStateBuilder<RequiredField>): HttpRequest;
  //    ↑ `this` parameter — build() only exists when TSet covers everything
}

// ═══════════════════════════════════════════════════════════════════════════
// 3. Narrowing a result type with a generic method
// ═══════════════════════════════════════════════════════════════════════════

declare class RowBuilder<TRow, TSelected extends keyof TRow> {
  select<K extends keyof TRow>(...columns: K[]): RowBuilder<TRow, K>;
  //     ↑ K is inferred from the ARGUMENTS, and becomes the new selection
  build(): { rows: Pick<TRow, TSelected>[] };
}

// ═══════════════════════════════════════════════════════════════════════════
// 4. Accumulating typed conditions
// ═══════════════════════════════════════════════════════════════════════════

type ComparisonOperator = "=" | "!=" | ">" | ">=" | "<" | "<=" | "LIKE" | "IN";

interface Condition<TRow, K extends keyof TRow = keyof TRow> {
  readonly column:   K;
  readonly operator: ComparisonOperator;
  readonly value:    TRow[K];
}

declare class WhereBuilder<TRow> {
  where<K extends keyof TRow>(
    column: K,
    operator: ComparisonOperator,
    value: TRow[K],           // ← the value type FOLLOWS the column
  ): WhereBuilder<TRow>;
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. Immutable builder — copy-on-write via a private state object
// ═══════════════════════════════════════════════════════════════════════════

interface BuilderState {
  readonly url?:     string;
  readonly method?:  "GET" | "POST";
  readonly timeout?: number;
}

class ImmutableBuilder {
  private constructor(private readonly state: BuilderState) {}

  static create(): ImmutableBuilder {
    return new ImmutableBuilder({});
  }

  url(value: string): ImmutableBuilder {
    return new ImmutableBuilder({ ...this.state, url: value });   // new instance
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 6. Branded / phantom types — a build-time-only marker
// ═══════════════════════════════════════════════════════════════════════════

declare const brand: unique symbol;
type Brand<T, Tag extends string> = T & { readonly [brand]: Tag };

type ValidatedQuery = Brand<string, "ValidatedQuery">;

declare function validate(raw: string): ValidatedQuery;
declare function execute(sql: ValidatedQuery): Promise<unknown[]>;

// execute("SELECT * FROM users");   // ❌ plain string is not a ValidatedQuery
execute(validate("SELECT * FROM users"));   // ✅

// ═══════════════════════════════════════════════════════════════════════════
// 7. Function-style builder (no class) — often lighter and easier to type
// ═══════════════════════════════════════════════════════════════════════════

interface FnBuilder<TSet extends string> {
  readonly url: (value: string) => FnBuilder<TSet | "url">;
  readonly build: TSet extends "url" ? () => HttpRequest : never;
  //               ↑ conditional type gates the method's very existence
}

// ── Supporting declarations used above ──────────────────────────────────────
interface HttpRequest { url: string; method: string }
```

---

## How it works — concept by concept

### Concept 1 — `this` types and why `return this` is not enough

The naive fluent method annotates its own class:

```ts
class ResponseBuilder {
  protected statusCode = 200;
  protected headers: Record<string, string> = {};

  status(code: number): ResponseBuilder {     // ← explicit class name
    this.statusCode = code;
    return this;
  }

  header(name: string, value: string): ResponseBuilder {
    this.headers[name] = value;
    return this;
  }
}
```

This works until you subclass:

```ts
class JsonResponseBuilder extends ResponseBuilder {
  protected payload: unknown = null;

  json(body: unknown): JsonResponseBuilder {
    this.payload = body;
    this.headers["Content-Type"] = "application/json";
    return this;
  }
}

new JsonResponseBuilder()
  .status(201)          // returns ResponseBuilder — the subclass identity is LOST
  .json({ userId: 1 }); // ❌ Property 'json' does not exist on type 'ResponseBuilder'.
```

The fix is the **polymorphic `this` type**. `this` used as a type annotation means "the type of the actual receiver", which is `JsonResponseBuilder` when called on one:

```ts
class ResponseBuilderFixed {
  protected statusCode = 200;
  protected headers: Record<string, string> = {};

  status(code: number): this {                // ← polymorphic `this`
    this.statusCode = code;
    return this;
  }

  header(name: string, value: string): this {
    this.headers[name] = value;
    return this;
  }
}

class JsonResponseBuilderFixed extends ResponseBuilderFixed {
  protected payload: unknown = null;

  json(body: unknown): this {
    this.payload = body;
    return this.header("Content-Type", "application/json");
    //     ↑ header() returns `this` = JsonResponseBuilderFixed ✅
  }
}

new JsonResponseBuilderFixed()
  .status(201)
  .json({ userId: 1 })
  .header("X-Request-Id", "req_abc")
  .json({ retry: false });     // ✅ every step keeps the subclass type
```

Two things to know about `this` types:

```ts
// (a) `this` is also usable as a PARAMETER type, which is how type-state works:
class Guarded {
  private ready = false;
  markReady(): asserts this is ReadyGuarded { this.ready = true; }
}
interface ReadyGuarded extends Guarded { readonly ready: true }

// (b) `this` types make a class INVARIANT in tricky ways — a `this`-returning
//     method cannot be satisfied by a fixed-type method during assignability:
class A { self(): this { return this; } }
class B { self(): B    { return this; } }
declare const b: B;
const a: A = b;   // ❌ 'B' is not assignable to 'A': the `this` types differ
```

For a *mutable* builder, `this` is the right return type in almost every case. For an *immutable* builder (Concept 5) you cannot use `this`, because you are returning a new object — you return the class type or a generic instead.

### Concept 2 — Type-state: tracking accumulated state in a type parameter

The core idea: give the builder a type parameter that is a **union of the names of the fields set so far**, starting at `never` (the empty union). Every setter widens it.

```ts
// Which fields must be present before build() is legal:
type RequiredKey = "url" | "method";

class HttpBuilder<TSet extends RequiredKey = never> {
  private constructor(
    private readonly parts: {
      readonly url?:     string;
      readonly method?:  "GET" | "POST" | "PUT" | "DELETE";
      readonly timeout?: number;
    },
  ) {}

  static create(): HttpBuilder<never> {
    return new HttpBuilder<never>({});
  }

  url(value: string): HttpBuilder<TSet | "url"> {
    return new HttpBuilder<TSet | "url">({ ...this.parts, url: value });
  }

  method(value: "GET" | "POST" | "PUT" | "DELETE"): HttpBuilder<TSet | "method"> {
    return new HttpBuilder<TSet | "method">({ ...this.parts, method: value });
  }

  // Optional field — does NOT change the type-state:
  timeout(ms: number): HttpBuilder<TSet> {
    return new HttpBuilder<TSet>({ ...this.parts, timeout: ms });
  }

  // The gate. A `this` PARAMETER constrains the receiver's type:
  build(this: HttpBuilder<RequiredKey>): { url: string; method: string; timeout: number } {
    // Inside here we KNOW url and method are set, but TypeScript's control flow
    // cannot see that through the type parameter — so we still assert once:
    return {
      url:     this.parts.url!,
      method:  this.parts.method!,
      timeout: this.parts.timeout ?? 30_000,
    };
  }
}
```

Reading the `this` parameter is the whole trick. `build(this: HttpBuilder<RequiredKey>)` says: *this method may only be called on a receiver assignable to `HttpBuilder<"url" | "method">`*. Because `TSet` appears in a property position that makes the class covariant in `TSet`, `HttpBuilder<"url">` is **not** assignable to `HttpBuilder<"url" | "method">`... except that TypeScript infers variance structurally, and if `TSet` appears nowhere in the members, the class is bivariant and the check silently passes. So you must make `TSet` actually *used*. The standard trick is a phantom property:

```ts
class HttpBuilderSafe<TSet extends RequiredKey = never> {
  // Phantom field: never assigned, erased at runtime, but forces TSet to
  // participate in assignability checks.
  declare private readonly __set: TSet;
  //      ↑ `declare` means "type-only, emit nothing"

  // ...setters as before...
}
```

Without the phantom field, `HttpBuilder<never>` and `HttpBuilder<"url" | "method">` are structurally identical (both have the same methods after instantiation? — not quite, since the setters' return types mention `TSet`), and the gate can leak. Method return types *do* reference `TSet`, so in practice this class is checked correctly — but the moment you write a builder whose type parameter only appears in `build`'s `this`, the phantom field becomes mandatory. Add it by default; it costs nothing.

Usage:

```ts
const partial = HttpBuilder.create().url("https://api.example.com/users");
partial.build();
// ❌ The 'this' context of type 'HttpBuilder<"url">' is not assignable to
//    method's 'this' of type 'HttpBuilder<"url" | "method">'.

const complete = HttpBuilder.create()
  .url("https://api.example.com/users")
  .method("POST")
  .timeout(5_000);

complete.build();   // ✅
```

Note that order does not matter — `.method().url()` produces the same union — which is exactly right. Type-state encodes *what is set*, not *what order it was set in*. (If you genuinely need ordering, encode a state name instead of a set: `Builder<"empty">` → `Builder<"hasUrl">` → `Builder<"ready">`.)

### Concept 3 — Two ways to gate `build()`: `this` parameters vs conditional types

The `this`-parameter gate gives clean errors but requires classes. The alternative, which works for object-literal builders, is a **conditional return type**:

```ts
type RequiredKey2 = "url" | "method";

interface FluentRequest<TSet extends RequiredKey2> {
  url(value: string): FluentRequest<TSet | "url">;
  method(value: string): FluentRequest<TSet | "method">;

  // If TSet doesn't cover everything, build's type becomes `never` — uncallable.
  build: [Exclude<RequiredKey2, TSet>] extends [never]
    ? () => { url: string; method: string }
    : { readonly __error: `Missing: ${Exclude<RequiredKey2, TSet>}` };
}
```

`Exclude<RequiredKey2, TSet>` computes the *missing* keys. The `[...] extends [never]` wrapper prevents distribution over the union (a bare `X extends never` distributes and yields `never` for every member — a classic trap covered in **40 — Conditional types**).

The error message trick is worth stealing. Instead of `never`, make the failure branch an object with a descriptive property:

```ts
declare const bad: FluentRequest<"url">;
bad.build();
// ❌ This expression is not callable.
//    Type '{ readonly __error: "Missing: method"; }' has no call signatures.
//                              ↑ the missing field name is IN the error text
```

That is dramatically better than `Type 'never' has no call signatures`. A third variant embeds the message in a parameter:

```ts
type BuildGate<TMissing extends string> = [TMissing] extends [never]
  ? []
  : [error: `Cannot build — still required: ${TMissing}`];

interface GatedBuilder<TSet extends RequiredKey2> {
  build(...args: BuildGate<Exclude<RequiredKey2, TSet>>): { url: string; method: string };
}

declare const g: GatedBuilder<"url">;
g.build();
// ❌ Expected 1 arguments, but got 0.
//    (the parameter is named `error` and typed with the message)
```

Pick one style and stay consistent. The `this`-parameter version is the most idiomatic for classes; the conditional-type version is the most flexible and works for interfaces and factory functions.

### Concept 4 — Narrowing the result type with `select<K extends keyof T>()`

This is the concept with no JavaScript analogue at all: a method call that **changes what a later method returns**.

```ts
interface OrdersTable {
  orderId:     string;
  userId:      number;
  totalCents:  number;
  currency:    "usd" | "eur" | "gbp";
  status:      "pending" | "paid" | "refunded";
  createdAt:   Date;
  refundedAt:  Date | null;
}

class SelectBuilder<TRow, TSelected extends keyof TRow = keyof TRow> {
  private constructor(
    private readonly table:    string,
    private readonly columns:  readonly (keyof TRow)[],
  ) {}

  static forTable<T>(table: string): SelectBuilder<T, keyof T> {
    return new SelectBuilder<T, keyof T>(table, []);
  }

  // K is INFERRED from the arguments. The returned builder carries K forward.
  select<const K extends keyof TRow>(...columns: K[]): SelectBuilder<TRow, K> {
    return new SelectBuilder<TRow, K>(this.table, columns);
  }

  // Pick<TRow, TSelected> is the exact row shape the query will produce:
  async execute(): Promise<Pick<TRow, TSelected>[]> {
    const cols = this.columns.length ? this.columns.join(", ") : "*";
    return runSql(`SELECT ${cols} FROM ${this.table}`) as Promise<Pick<TRow, TSelected>[]>;
  }
}

declare function runSql(sql: string): Promise<unknown[]>;
```

Now watch the row type track the chain:

```ts
const all = await SelectBuilder.forTable<OrdersTable>("orders").execute();
// all: OrdersTable[]  — no select() means every column

const slim = await SelectBuilder.forTable<OrdersTable>("orders")
  .select("orderId", "totalCents", "currency")
  .execute();
// slim: { orderId: string; totalCents: number; currency: "usd" | "eur" | "gbp" }[]

slim[0].status;
// ❌ Property 'status' does not exist on type
//    '{ orderId: string; totalCents: number; currency: "usd" | "eur" | "gbp"; }'
//    ↑ You didn't select it, so you can't read it. The compiler knows.
```

Three details make this work properly:

**(a) `K extends keyof TRow` constrains the argument names.**

```ts
SelectBuilder.forTable<OrdersTable>("orders").select("totalCent");
// ❌ Argument of type '"totalCent"' is not assignable to parameter of type
//    'keyof OrdersTable'. Did you mean '"totalCents"'?
```

**(b) `...columns: K[]` (rest parameter) infers `K` as the *union* of all arguments.** With three arguments `"orderId" | "totalCents" | "currency"` is inferred, which is precisely what `Pick` needs. If you instead wrote `columns: K[]` taking an array argument, you'd need `as const` at the call site or `const K` to avoid widening to `string`.

**(c) The `const` type parameter modifier (TS 5.0+)** — `select<const K extends keyof TRow>` — makes literal arguments infer as literals without `as const`. For rest parameters of a `keyof` type it is usually unnecessary (the constraint already prevents widening to `string`), but it becomes essential when the parameter type is looser:

```ts
// Without `const`, arrays of literals widen:
declare function pick1<K extends string>(keys: K[]): K;
const r1 = pick1(["userId", "email"]);        // K = string  ✗

declare function pick2<const K extends string>(keys: K[]): K;
const r2 = pick2(["userId", "email"]);        // K = "userId" | "email"  ✓
```

**Composing selections.** A second `.select()` should *replace*, not intersect — otherwise chaining is confusing. If you want additive selection, use a differently-named method:

```ts
class AdditiveBuilder<TRow, TSelected extends keyof TRow = never> {
  addColumns<K extends keyof TRow>(...columns: K[]): AdditiveBuilder<TRow, TSelected | K> {
    return this as unknown as AdditiveBuilder<TRow, TSelected | K>;
  }
}
// .addColumns("orderId").addColumns("totalCents") → TSelected = "orderId" | "totalCents"
```

**Computed columns.** Real query builders also let you add derived columns, which means the result type is `Pick<TRow, TSelected> & TComputed`:

```ts
class ComputedBuilder<TRow, TSelected extends keyof TRow, TExtra extends object = {}> {
  countAs<TName extends string>(name: TName): ComputedBuilder<TRow, TSelected, TExtra & Record<TName, number>> {
    return this as unknown as ComputedBuilder<TRow, TSelected, TExtra & Record<TName, number>>;
  }

  declare execute: () => Promise<(Pick<TRow, TSelected> & TExtra)[]>;
}

declare const cb: ComputedBuilder<OrdersTable, "userId">;
const grouped = cb.countAs("orderCount");
// execute(): Promise<({ userId: number } & { orderCount: number })[]>
```

### Concept 5 — `.where()` and accumulating typed conditions

A condition has three parts: a column, an operator, and a value. The value's type must depend on the column:

```ts
type Operator = "=" | "!=" | ">" | ">=" | "<" | "<=" | "LIKE" | "IN" | "IS NULL";

// A condition is a discriminated-by-column union over the row's keys.
// The distributive form is what makes value's type follow column's type:
type Condition<TRow> = {
  [K in keyof TRow]: {
    readonly column:   K;
    readonly operator: Operator;
    readonly value:    TRow[K];
  }
}[keyof TRow];
// For OrdersTable this is:
//   { column: "orderId"; operator: Operator; value: string }
// | { column: "userId";  operator: Operator; value: number }
// | { column: "status";  operator: Operator; value: "pending" | "paid" | "refunded" }
// | ...
```

The builder method uses a generic to link the two arguments:

```ts
class ConditionBuilder<TRow> {
  constructor(private readonly conditions: readonly Condition<TRow>[] = []) {}

  where<K extends keyof TRow>(column: K, operator: Operator, value: TRow[K]): ConditionBuilder<TRow> {
    const next = { column, operator, value } as Condition<TRow>;
    return new ConditionBuilder<TRow>([...this.conditions, next]);
  }
}

declare const cond: ConditionBuilder<OrdersTable>;

cond.where("totalCents", ">=", 1000);          // ✅ number expected, number given
cond.where("currency",   "=",  "usd");         // ✅ narrowed to the literal union
cond.where("currency",   "=",  "inr");         // ❌ not assignable to "usd" | "eur" | "gbp"
cond.where("totalCents", ">=", "1000");        // ❌ string is not assignable to number
cond.where("total",      ">=", 1000);          // ❌ 'total' is not a key of OrdersTable
```

**Operator-aware value types.** You can go further and make the *operator* constrain the value too — `IN` takes an array, `IS NULL` takes nothing, `LIKE` only makes sense on strings:

```ts
type ValueFor<TValue, TOp extends Operator> =
    TOp extends "IN"      ? readonly TValue[]
  : TOp extends "IS NULL" ? null
  : TOp extends "LIKE"    ? (TValue extends string ? string : never)
  : TValue;

class SmartConditionBuilder<TRow> {
  where<K extends keyof TRow, TOp extends Operator>(
    column: K,
    operator: TOp,
    value: ValueFor<TRow[K], TOp>,
  ): SmartConditionBuilder<TRow> {
    return this;
  }
}

declare const smart: SmartConditionBuilder<OrdersTable>;

smart.where("status", "IN", ["paid", "refunded"]);   // ✅ array required for IN
smart.where("status", "IN", "paid");                 // ❌ readonly (...)[] expected
smart.where("orderId", "LIKE", "ord_%");             // ✅ string column, string pattern
smart.where("totalCents", "LIKE", "1%");             // ❌ value type is never
smart.where("refundedAt", "IS NULL", null);          // ✅
```

**Accumulating conditions into the TYPE, not just the value.** Sometimes you want the *set of filtered columns* visible in the type — for example to guarantee that a multi-tenant query always filters by `tenantId`:

```ts
class TenantSafeBuilder<TRow, TFiltered extends keyof TRow = never> {
  where<K extends keyof TRow>(column: K, operator: Operator, value: TRow[K]): TenantSafeBuilder<TRow, TFiltered | K> {
    return this as unknown as TenantSafeBuilder<TRow, TFiltered | K>;
  }

  // Only callable once tenantId has been filtered on:
  execute(this: TenantSafeBuilder<TRow, "tenantId" & keyof TRow>): Promise<TRow[]> {
    return runSql("...") as Promise<TRow[]>;
  }
}

interface TenantOrders extends OrdersTable { tenantId: string }

declare const t: TenantSafeBuilder<TenantOrders>;
t.where("status", "=", "paid").execute();
// ❌ 'this' context of type 'TenantSafeBuilder<TenantOrders, "status">' is not
//    assignable to method's 'this' of type 'TenantSafeBuilder<TenantOrders, "tenantId">'

t.where("tenantId", "=", "acme").where("status", "=", "paid").execute();   // ✅
```

That single pattern eliminates an entire class of production incident: the cross-tenant data leak caused by a forgotten `WHERE tenant_id = $1`.

### Concept 6 — Mutable vs immutable builders

Both are legitimate; they fail differently.

```ts
// ── Mutable: one object, methods return `this` ──────────────────────────────
class MutableQuery {
  private readonly conditions: string[] = [];

  where(clause: string): this {
    this.conditions.push(clause);      // mutates in place
    return this;
  }

  build(): string {
    return `WHERE ${this.conditions.join(" AND ")}`;
  }
}

// The aliasing hazard:
const base = new MutableQuery().where("status = 'active'");
const q1 = base.where("created_at > now() - interval '1 day'").build();
const q2 = base.where("created_at < now() - interval '1 year'").build();
// q1: "WHERE status = 'active' AND created_at > ..."
// q2: "WHERE status = 'active' AND created_at > ... AND created_at < ..."   💥
//     ↑ q2 inherited q1's condition. `base` was mutated by the first chain.
```

```ts
// ── Immutable: every method returns a NEW builder over a new state ──────────
class ImmutableQuery {
  private constructor(private readonly conditions: readonly string[]) {}

  static create(): ImmutableQuery {
    return new ImmutableQuery([]);
  }

  where(clause: string): ImmutableQuery {
    return new ImmutableQuery([...this.conditions, clause]);   // copy-on-write
  }

  build(): string {
    return `WHERE ${this.conditions.join(" AND ")}`;
  }
}

const base2 = ImmutableQuery.create().where("status = 'active'");
const q3 = base2.where("created_at > now() - interval '1 day'").build();
const q4 = base2.where("created_at < now() - interval '1 year'").build();
// q3 and q4 each have exactly two conditions. `base2` is untouched. ✅
```

| | Mutable | Immutable |
|---|---|---|
| Return type | `this` (subclass-safe, free) | the class or a generic instance |
| Reuse / branching | **unsafe** — shared prefix leaks | **safe** — prefixes are values |
| Allocation | one object total | one object per call |
| Type-state | awkward (`this` can't change type)* | natural (each call returns a new type) |
| Async / concurrent use | dangerous | safe |
| Typical use | short one-shot chains, test fixtures | reusable prefixes, DI, request pipelines |

\* You *can* do type-state on a mutable builder by casting `this` to the widened type, but you are then lying: the runtime object is shared, so two branches from the same prefix will see each other's fields even though their types disagree. If you want type-state, use an immutable builder.

**The pragmatic middle ground.** Immutable builders allocate one small object per call. A 6-call chain is 6 short-lived objects — completely irrelevant next to the cost of the database round trip that follows. Do not micro-optimise this. The one case where mutable wins on performance is a builder called in a tight loop hundreds of thousands of times (a serialisation buffer, a log line assembler); there, use a mutable builder and reset it.

There is also a **hybrid**: mutate internally, but copy on `fork()`:

```ts
class HybridQuery {
  private conditions: string[] = [];

  where(clause: string): this {
    this.conditions.push(clause);
    return this;
  }

  fork(): HybridQuery {
    const copy = new HybridQuery();
    copy.conditions = [...this.conditions];
    return copy;
  }
}
// Cheap chains, explicit branching. Requires discipline — the compiler
// cannot force you to call fork().
```

### Concept 7 — Branded and phantom types for build-time guarantees

A **brand** is a type-level tag with no runtime existence, used to make two structurally identical types incompatible.

```ts
declare const __brand: unique symbol;
type Brand<TBase, TTag extends string> = TBase & { readonly [__brand]: TTag };

type RawSql       = Brand<string, "RawSql">;
type SanitizedSql = Brand<string, "SanitizedSql">;
type UserId       = Brand<number, "UserId">;
type TenantId     = Brand<string, "TenantId">;

declare function executeSql(sql: SanitizedSql): Promise<unknown[]>;

const handWritten = `SELECT * FROM users WHERE id = ${1}`;
executeSql(handWritten);
// ❌ Type 'string' is not assignable to type 'SanitizedSql'.
//    Property '[__brand]' is missing.

// The ONLY way to obtain a SanitizedSql is through the builder:
declare function sanitize(sql: string, params: readonly unknown[]): SanitizedSql;
executeSql(sanitize("SELECT * FROM users WHERE id = $1", [1]));   // ✅
```

Because the brand uses a `unique symbol` declared in one module and never exported, nobody outside can forge the type without an explicit cast. That is the "build-time guarantee": the type system encodes *provenance*, not just shape.

Brands make ID mix-ups impossible too:

```ts
declare function loadUser(userId: UserId): Promise<unknown>;
declare function loadTenant(tenantId: TenantId): Promise<unknown>;

declare const someUserId: UserId;
loadTenant(someUserId);
// ❌ Type 'UserId' (number & brand) is not assignable to 'TenantId' (string & brand)
```

A **phantom type parameter** is the related idea applied to generics: a type parameter that appears in the type but corresponds to no runtime data.

```ts
// `TState` is phantom — no field of the class actually stores it.
declare const __state: unique symbol;

class Transaction<TState extends "open" | "committed" | "rolledBack"> {
  declare private readonly [__state]: TState;

  private constructor(private readonly id: string) {}

  static begin(): Transaction<"open"> {
    return new Transaction<"open">(crypto.randomUUID());
  }

  query(this: Transaction<"open">, sql: string): Promise<unknown[]> {
    return runSql(sql);
  }

  commit(this: Transaction<"open">): Transaction<"committed"> {
    return this as unknown as Transaction<"committed">;
  }

  rollback(this: Transaction<"open">): Transaction<"rolledBack"> {
    return this as unknown as Transaction<"rolledBack">;
  }
}

const tx = Transaction.begin();
const done = tx.commit();
done.query("SELECT 1");
// ❌ 'this' context of type 'Transaction<"committed">' is not assignable to
//    method's 'this' of type 'Transaction<"open">'
done.commit();
// ❌ same — you cannot commit twice
```

The `declare private readonly [__state]: TState` line is the phantom field. `declare` means TypeScript emits nothing for it (no `undefined` property at runtime), but it forces `TState` to participate in structural comparison. Without it, `Transaction<"open">` and `Transaction<"committed">` would be structurally identical and every gate would pass.

**Where brands and builders meet:** the output of `.build()` can be branded, so downstream functions can require *builder-produced* values:

```ts
type ValidatedRequest = Brand<{ url: string; method: string; headers: Record<string, string> }, "ValidatedRequest">;

declare class BrandedBuilder {
  build(): ValidatedRequest;
}

declare function send(request: ValidatedRequest): Promise<Response>;

send({ url: "https://x.example.com", method: "GET", headers: {} });
// ❌ hand-rolled object literal lacks the brand — must go through the builder
```

### Concept 8 — Building the builder: factory functions vs classes

Classes are the traditional home of builders, but a **closure-based factory** is often easier to type, especially with type-state, because you are not fighting `this`.

```ts
interface RetryPolicy {
  readonly maxAttempts:  number;
  readonly baseDelayMs:  number;
  readonly jitter:       boolean;
  readonly retryOn:      readonly number[];
}

type PolicyKey = "maxAttempts" | "baseDelayMs";

interface PolicyBuilder<TSet extends PolicyKey> {
  maxAttempts(n: number): PolicyBuilder<TSet | "maxAttempts">;
  baseDelayMs(ms: number): PolicyBuilder<TSet | "baseDelayMs">;
  jitter(enabled: boolean): PolicyBuilder<TSet>;
  retryOn(...statusCodes: number[]): PolicyBuilder<TSet>;
  build(this: PolicyBuilder<PolicyKey>): RetryPolicy;
}

function policyBuilder(state: Partial<RetryPolicy> = {}): PolicyBuilder<never> {
  const make = <TNext extends PolicyKey>(next: Partial<RetryPolicy>): PolicyBuilder<TNext> =>
    policyBuilder({ ...state, ...next }) as unknown as PolicyBuilder<TNext>;

  return {
    maxAttempts: (n) => make({ maxAttempts: n }),
    baseDelayMs: (ms) => make({ baseDelayMs: ms }),
    jitter:      (enabled) => make({ jitter: enabled }),
    retryOn:     (...statusCodes) => make({ retryOn: statusCodes }),
    build: () => ({
      maxAttempts: state.maxAttempts ?? 3,
      baseDelayMs: state.baseDelayMs ?? 100,
      jitter:      state.jitter ?? true,
      retryOn:     state.retryOn ?? [429, 502, 503, 504],
    }),
  } as PolicyBuilder<never>;
}

const policy = policyBuilder()
  .maxAttempts(5)
  .baseDelayMs(200)
  .retryOn(429, 503)
  .build();
// ✅ RetryPolicy

policyBuilder().jitter(true).build();
// ❌ 'this' context of type 'PolicyBuilder<never>' is not assignable to
//    method's 'this' of type 'PolicyBuilder<"maxAttempts" | "baseDelayMs">'
```

Notice the one unavoidable cast (`as unknown as PolicyBuilder<TNext>`). **Every type-state builder needs exactly one internal cast**, because the runtime object genuinely does not change shape — only its *type* does. That cast is the boundary of your trust: it is inside the builder, written once, reviewed carefully, and everything outside is fully checked. Do not let casts leak past it.

Trade-offs:

| | Class builder | Factory/closure builder |
|---|---|---|
| `this` return type | available, free subclass support | not applicable |
| Type-state | needs phantom field + `this` params | natural via return types |
| Private state | `private readonly` fields | closure variables (truly private) |
| Extension | `extends` | composition / spreading |
| Bundle size | slightly larger (prototype) | slightly smaller, but one object per call |
| Reads best when | 10+ methods, inheritance | 3–8 methods, heavy type-state |

---

## Example 1 — basic

```ts
// A small, complete, type-state HTTP request builder.
// Required: url + method. Optional: headers, query, timeout, body.
// build() is only callable once url and method are both set.

// ═══════════════════════════════════════════════════════════════════════════
// 1. The value the builder produces
// ═══════════════════════════════════════════════════════════════════════════

type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

interface HttpRequestSpec {
  readonly url:         string;
  readonly method:      HttpMethod;
  readonly headers:     Readonly<Record<string, string>>;
  readonly query:       Readonly<Record<string, string>>;
  readonly body:        string | null;
  readonly timeoutMs:   number;
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. Which fields the type-state tracks
// ═══════════════════════════════════════════════════════════════════════════

type RequiredRequestField = "url" | "method";

// Internal, partially-filled state. Mutable-looking but always copied.
interface RequestDraft {
  readonly url?:       string;
  readonly method?:    HttpMethod;
  readonly headers:    Readonly<Record<string, string>>;
  readonly query:      Readonly<Record<string, string>>;
  readonly body:       string | null;
  readonly timeoutMs:  number;
}

// ═══════════════════════════════════════════════════════════════════════════
// 3. The builder — immutable, one instance per call
// ═══════════════════════════════════════════════════════════════════════════

class RequestBuilder<TSet extends RequiredRequestField = never> {
  // Phantom field: type-only, emits nothing, forces TSet into variance checks.
  declare private readonly __set: TSet;

  private constructor(private readonly draft: RequestDraft) {}

  static create(): RequestBuilder<never> {
    return new RequestBuilder<never>({
      headers:   {},
      query:     {},
      body:      null,
      timeoutMs: 30_000,
    });
  }

  // ── Required setters: each WIDENS the type-state union ────────────────────

  url(value: string): RequestBuilder<TSet | "url"> {
    return new RequestBuilder<TSet | "url">({ ...this.draft, url: value });
  }

  method(value: HttpMethod): RequestBuilder<TSet | "method"> {
    return new RequestBuilder<TSet | "method">({ ...this.draft, method: value });
  }

  // ── Optional setters: type-state unchanged ────────────────────────────────

  header(name: string, value: string): RequestBuilder<TSet> {
    return new RequestBuilder<TSet>({
      ...this.draft,
      headers: { ...this.draft.headers, [name]: value },
    });
  }

  bearer(authToken: string): RequestBuilder<TSet> {
    return this.header("Authorization", `Bearer ${authToken}`);
  }

  param(name: string, value: string | number | boolean): RequestBuilder<TSet> {
    return new RequestBuilder<TSet>({
      ...this.draft,
      query: { ...this.draft.query, [name]: String(value) },
    });
  }

  json(payload: unknown): RequestBuilder<TSet> {
    return new RequestBuilder<TSet>({
      ...this.draft,
      body:    JSON.stringify(payload),
      headers: { ...this.draft.headers, "Content-Type": "application/json" },
    });
  }

  timeout(ms: number): RequestBuilder<TSet> {
    return new RequestBuilder<TSet>({ ...this.draft, timeoutMs: ms });
  }

  // ── The gate ──────────────────────────────────────────────────────────────

  build(this: RequestBuilder<RequiredRequestField>): HttpRequestSpec {
    // The `this` parameter proves url and method are set. The compiler still
    // sees them as optional in RequestDraft, so we narrow once, defensively.
    const { url, method } = this.draft;
    if (url === undefined || method === undefined) {
      // Unreachable given the type-state, but keeps the runtime honest if
      // someone casts around the compiler.
      throw new Error("RequestBuilder: url and method are required");
    }
    return {
      url,
      method,
      headers:   this.draft.headers,
      query:     this.draft.query,
      body:      this.draft.body,
      timeoutMs: this.draft.timeoutMs,
    };
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 4. Using it
// ═══════════════════════════════════════════════════════════════════════════

const listUsersRequest = RequestBuilder.create()
  .url("https://api.example.com/v1/users")
  .method("GET")
  .bearer("tok_live_9f2c")
  .param("status", "active")
  .param("limit", 50)
  .timeout(5_000)
  .build();

// listUsersRequest: HttpRequestSpec
// {
//   url: "https://api.example.com/v1/users",
//   method: "GET",
//   headers: { Authorization: "Bearer tok_live_9f2c" },
//   query: { status: "active", limit: "50" },
//   body: null,
//   timeoutMs: 5000,
// }

const createUserRequest = RequestBuilder.create()
  .url("https://api.example.com/v1/users")
  .method("POST")
  .bearer("tok_live_9f2c")
  .json({ email: "ada@example.com", displayName: "Ada Lovelace" })
  .build();

// ═══════════════════════════════════════════════════════════════════════════
// 5. The errors the compiler now catches
// ═══════════════════════════════════════════════════════════════════════════

RequestBuilder.create().url("https://api.example.com/v1/users").build();
// ❌ The 'this' context of type 'RequestBuilder<"url">' is not assignable to
//    method's 'this' of type 'RequestBuilder<"url" | "method">'.
//      Type '"url"' is not assignable to type '"url" | "method"'... (missing "method")

RequestBuilder.create().method("POST").build();
// ❌ same shape of error — 'url' was never set

RequestBuilder.create().url("...").method("FETCH");
// ❌ Argument of type '"FETCH"' is not assignable to parameter of type 'HttpMethod'.

// ═══════════════════════════════════════════════════════════════════════════
// 6. Immutability pays off: a shared, reusable prefix
// ═══════════════════════════════════════════════════════════════════════════

const apiBase = RequestBuilder.create()
  .bearer("tok_live_9f2c")
  .header("Accept", "application/json")
  .header("X-Client", "billing-worker/1.4.2")
  .timeout(10_000);
// apiBase: RequestBuilder<never> — no required field set yet, and that's fine

const getInvoice = apiBase.url("https://api.example.com/v1/invoices/inv_88").method("GET").build();
const voidInvoice = apiBase.url("https://api.example.com/v1/invoices/inv_88/void").method("POST").build();

// Both share the auth header and timeout; neither leaked into the other.
// With a MUTABLE builder, voidInvoice would have inherited getInvoice's url.
```

---

## Example 2 — real world backend use case

```ts
// A typed query builder + a typed HTTP client, as you'd actually ship them.
//
// Part 1 — schema-aware SQL query builder with select() narrowing,
//          typed where(), tenant-safety type-state, and a branded output.
// Part 2 — a typed HTTP client built on the request builder, with a
//          response-type parameter that flows into the awaited value.

// ═══════════════════════════════════════════════════════════════════════════
// PART 1 — Typed query builder
// ═══════════════════════════════════════════════════════════════════════════

// ── 1.1 The schema: one source of truth for every table ────────────────────

interface UsersRow {
  userId:      number;
  tenantId:    string;
  email:       string;
  displayName: string;
  status:      "active" | "suspended" | "deleted";
  createdAt:   Date;
  lastSeenAt:  Date | null;
}

interface OrdersRow {
  orderId:     string;
  tenantId:    string;
  userId:      number;
  totalCents:  number;
  currency:    "usd" | "eur" | "gbp";
  status:      "pending" | "paid" | "refunded";
  createdAt:   Date;
}

interface DatabaseSchema {
  users:  UsersRow;
  orders: OrdersRow;
}

// ── 1.2 Branded SQL so only the builder can produce executable statements ───

declare const sqlBrand: unique symbol;

interface CompiledQuery<TRow> {
  readonly [sqlBrand]: "CompiledQuery";
  readonly text:       string;
  readonly params:     readonly unknown[];
  /** Phantom: carries the row type. Never present at runtime. */
  readonly __row?:     TRow;
}

// ── 1.3 Condition + ordering types ─────────────────────────────────────────

type SqlOperator = "=" | "!=" | ">" | ">=" | "<" | "<=" | "LIKE" | "IN" | "IS NULL";

type OperandFor<TValue, TOp extends SqlOperator> =
    TOp extends "IN"      ? readonly TValue[]
  : TOp extends "IS NULL" ? null
  : TOp extends "LIKE"    ? (TValue extends string ? string : never)
  : TValue;

interface WhereClause {
  readonly column:   string;
  readonly operator: SqlOperator;
  readonly value:    unknown;
}

interface OrderClause {
  readonly column:    string;
  readonly direction: "asc" | "desc";
}

interface QueryState {
  readonly table:      string;
  readonly columns:    readonly string[];
  readonly conditions: readonly WhereClause[];
  readonly ordering:   readonly OrderClause[];
  readonly limitValue: number | null;
  readonly offsetValue: number | null;
}

// ── 1.4 The builder ────────────────────────────────────────────────────────
//
// Three type parameters:
//   TRow      — the full row type of the table
//   TSelected — which columns are in the result (defaults to all)
//   TFiltered — which columns have been filtered on (drives tenant safety)

class QueryBuilder<
  TRow,
  TSelected extends keyof TRow = keyof TRow,
  TFiltered extends keyof TRow = never,
> {
  declare private readonly __selected: TSelected;
  declare private readonly __filtered: TFiltered;

  private constructor(private readonly state: QueryState) {}

  static from<TName extends keyof DatabaseSchema>(
    table: TName,
  ): QueryBuilder<DatabaseSchema[TName], keyof DatabaseSchema[TName], never> {
    return new QueryBuilder({
      table:       String(table),
      columns:     [],
      conditions:  [],
      ordering:    [],
      limitValue:  null,
      offsetValue: null,
    });
  }

  // ── select(): narrows the result row type ────────────────────────────────
  select<const K extends keyof TRow>(...columns: K[]): QueryBuilder<TRow, K, TFiltered> {
    return new QueryBuilder<TRow, K, TFiltered>({
      ...this.state,
      columns: columns.map(String),
    });
  }

  // ── where(): value type follows the column, and records the column ───────
  where<K extends keyof TRow, TOp extends SqlOperator>(
    column: K,
    operator: TOp,
    value: OperandFor<TRow[K], TOp>,
  ): QueryBuilder<TRow, TSelected, TFiltered | K> {
    return new QueryBuilder<TRow, TSelected, TFiltered | K>({
      ...this.state,
      conditions: [...this.state.conditions, { column: String(column), operator, value }],
    });
  }

  orderBy<K extends keyof TRow>(column: K, direction: "asc" | "desc" = "asc"): this extends never ? never : QueryBuilder<TRow, TSelected, TFiltered> {
    return new QueryBuilder<TRow, TSelected, TFiltered>({
      ...this.state,
      ordering: [...this.state.ordering, { column: String(column), direction }],
    }) as QueryBuilder<TRow, TSelected, TFiltered>;
  }

  limit(count: number): QueryBuilder<TRow, TSelected, TFiltered> {
    return new QueryBuilder<TRow, TSelected, TFiltered>({ ...this.state, limitValue: count });
  }

  // offset() is only meaningful with a limit — but expressing that needs a
  // fourth type parameter, which is past the point of diminishing returns.
  // A runtime check in compile() is the pragmatic choice here.
  offset(count: number): QueryBuilder<TRow, TSelected, TFiltered> {
    return new QueryBuilder<TRow, TSelected, TFiltered>({ ...this.state, offsetValue: count });
  }

  // ── compile(): gated on tenantId having been filtered ────────────────────
  compile(
    this: QueryBuilder<TRow, TSelected, Extract<keyof TRow, "tenantId">>,
  ): CompiledQuery<Pick<TRow, TSelected>> {
    const s = this.state;
    const params: unknown[] = [];

    const columnList = s.columns.length > 0 ? s.columns.map(quoteIdent).join(", ") : "*";
    let text = `SELECT ${columnList} FROM ${quoteIdent(s.table)}`;

    if (s.conditions.length > 0) {
      const rendered = s.conditions.map((clause) => {
        if (clause.operator === "IS NULL") return `${quoteIdent(clause.column)} IS NULL`;
        if (clause.operator === "IN") {
          const values = clause.value as readonly unknown[];
          const placeholders = values.map((v) => `$${params.push(v)}`).join(", ");
          return `${quoteIdent(clause.column)} IN (${placeholders})`;
        }
        return `${quoteIdent(clause.column)} ${clause.operator} $${params.push(clause.value)}`;
      });
      text += ` WHERE ${rendered.join(" AND ")}`;
    }

    if (s.ordering.length > 0) {
      text += ` ORDER BY ${s.ordering.map((o) => `${quoteIdent(o.column)} ${o.direction.toUpperCase()}`).join(", ")}`;
    }
    if (s.limitValue !== null)  text += ` LIMIT $${params.push(s.limitValue)}`;
    if (s.offsetValue !== null) {
      if (s.limitValue === null) throw new Error("QueryBuilder: offset() requires limit()");
      text += ` OFFSET $${params.push(s.offsetValue)}`;
    }

    return { [sqlBrand]: "CompiledQuery", text, params } as CompiledQuery<Pick<TRow, TSelected>>;
  }
}

function quoteIdent(identifier: string): string {
  if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(identifier)) {
    throw new Error(`Unsafe SQL identifier: ${identifier}`);
  }
  return `"${identifier.replace(/([A-Z])/g, "_$1").toLowerCase()}"`;
}

// ── 1.5 Execution only accepts BRANDED compiled queries ────────────────────

interface DatabasePool {
  query(text: string, params: readonly unknown[]): Promise<{ rows: unknown[] }>;
}

async function runQuery<TRow>(pool: DatabasePool, query: CompiledQuery<TRow>): Promise<TRow[]> {
  const result = await pool.query(query.text, query.params);
  return result.rows as TRow[];
}

// ── 1.6 Using it ───────────────────────────────────────────────────────────

declare const pool: DatabasePool;

async function findRecentPaidOrders(tenantId: string) {
  const compiled = QueryBuilder.from("orders")
    .select("orderId", "userId", "totalCents", "currency")
    .where("tenantId", "=", tenantId)
    .where("status", "=", "paid")
    .where("totalCents", ">=", 5_000)
    .orderBy("createdAt", "desc")
    .limit(100)
    .compile();

  const rows = await runQuery(pool, compiled);
  // rows: { orderId: string; userId: number; totalCents: number; currency: "usd" | "eur" | "gbp" }[]

  return rows.map((row) => ({
    orderId: row.orderId,
    amount:  row.totalCents / 100,
    // row.status;  // ❌ Property 'status' does not exist — it wasn't selected
  }));
}

// Compile-time failures this design produces:

QueryBuilder.from("orders").select("orderId").where("status", "=", "paid").compile();
// ❌ 'this' context of type 'QueryBuilder<OrdersRow, "orderId", "status">' is not
//    assignable to method's 'this' of type 'QueryBuilder<OrdersRow, "orderId", "tenantId">'
//    ↑ MISSING TENANT FILTER — the cross-tenant leak is a compile error

QueryBuilder.from("orders").where("currency", "=", "inr");
// ❌ Type '"inr"' is not assignable to type '"usd" | "eur" | "gbp"'.

QueryBuilder.from("orders").where("createdAt", ">", "2026-01-01");
// ❌ Argument of type 'string' is not assignable to parameter of type 'Date'.

QueryBuilder.from("orders").where("status", "IN", "paid");
// ❌ Argument of type 'string' is not assignable to parameter of type
//    'readonly ("pending" | "paid" | "refunded")[]'.

QueryBuilder.from("users").select("emailAddress");
// ❌ Argument of type '"emailAddress"' is not assignable to parameter of type
//    'keyof UsersRow'. Did you mean '"email"'?

runQuery(pool, { text: "DELETE FROM users", params: [] });
// ❌ Property '[sqlBrand]' is missing — hand-written objects cannot be executed

// ═══════════════════════════════════════════════════════════════════════════
// PART 2 — Typed HTTP client built with a builder
// ═══════════════════════════════════════════════════════════════════════════

// ── 2.1 The API contract, declared once ────────────────────────────────────

interface ApiResponse<TData> {
  readonly data:      TData;
  readonly requestId: string;
}

interface UserDto {
  readonly userId:      number;
  readonly email:       string;
  readonly displayName: string;
  readonly createdAt:   string;
}

interface CreateUserBody {
  readonly email:       string;
  readonly displayName: string;
  readonly password:    string;
}

// ── 2.2 Type-state keys ────────────────────────────────────────────────────

type ClientRequiredKey = "path" | "method";

interface ClientState {
  readonly baseUrl:    string;
  readonly path?:      string;
  readonly method?:    HttpVerb;
  readonly headers:    Readonly<Record<string, string>>;
  readonly query:      Readonly<Record<string, string>>;
  readonly body?:      unknown;
  readonly timeoutMs:  number;
  readonly retries:    number;
}

type HttpVerb = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";

// ── 2.3 The client builder ─────────────────────────────────────────────────
//
// TResponse is the DECODED response type. It starts as `unknown` and is
// narrowed by .expect<T>() or by .decode(validator).

class ApiRequestBuilder<
  TSet extends ClientRequiredKey = never,
  TResponse = unknown,
> {
  declare private readonly __set: TSet;

  private constructor(private readonly state: ClientState) {}

  static forService(baseUrl: string): ApiRequestBuilder<never, unknown> {
    return new ApiRequestBuilder<never, unknown>({
      baseUrl,
      headers:   { Accept: "application/json" },
      query:     {},
      timeoutMs: 15_000,
      retries:   0,
    });
  }

  private next<TNextSet extends ClientRequiredKey, TNextResponse>(
    patch: Partial<ClientState>,
  ): ApiRequestBuilder<TNextSet, TNextResponse> {
    return new ApiRequestBuilder<TNextSet, TNextResponse>({ ...this.state, ...patch });
  }

  // ── Required ─────────────────────────────────────────────────────────────

  get(path: string):    ApiRequestBuilder<TSet | "path" | "method", TResponse> {
    return this.next({ path, method: "GET" });
  }
  post(path: string):   ApiRequestBuilder<TSet | "path" | "method", TResponse> {
    return this.next({ path, method: "POST" });
  }
  patch(path: string):  ApiRequestBuilder<TSet | "path" | "method", TResponse> {
    return this.next({ path, method: "PATCH" });
  }
  delete(path: string): ApiRequestBuilder<TSet | "path" | "method", TResponse> {
    return this.next({ path, method: "DELETE" });
  }

  // ── Optional ─────────────────────────────────────────────────────────────

  bearer(authToken: string): ApiRequestBuilder<TSet, TResponse> {
    return this.next({ headers: { ...this.state.headers, Authorization: `Bearer ${authToken}` } });
  }

  idempotencyKey(key: string): ApiRequestBuilder<TSet, TResponse> {
    return this.next({ headers: { ...this.state.headers, "Idempotency-Key": key } });
  }

  param(name: string, value: string | number | boolean): ApiRequestBuilder<TSet, TResponse> {
    return this.next({ query: { ...this.state.query, [name]: String(value) } });
  }

  json<TBody>(requestBody: TBody): ApiRequestBuilder<TSet, TResponse> {
    return this.next({
      body:    requestBody,
      headers: { ...this.state.headers, "Content-Type": "application/json" },
    });
  }

  timeout(ms: number): ApiRequestBuilder<TSet, TResponse> {
    return this.next({ timeoutMs: ms });
  }

  retry(attempts: number): ApiRequestBuilder<TSet, TResponse> {
    return this.next({ retries: attempts });
  }

  // ── Response narrowing: pure type-level, no runtime effect ───────────────

  expect<TExpected>(): ApiRequestBuilder<TSet, TExpected> {
    return this.next<TSet, TExpected>({});
  }

  // ── Response narrowing WITH runtime validation (preferred) ───────────────

  decode<TDecoded>(
    validator: (raw: unknown) => TDecoded,
  ): ApiRequestBuilder<TSet, TDecoded> & { readonly validator: (raw: unknown) => TDecoded } {
    const built = this.next<TSet, TDecoded>({});
    return Object.assign(built, { validator });
  }

  // ── The gate ─────────────────────────────────────────────────────────────

  async send(this: ApiRequestBuilder<ClientRequiredKey, TResponse>): Promise<TResponse> {
    const { baseUrl, path, method, headers, query, body, timeoutMs, retries } = this.state;
    if (path === undefined || method === undefined) {
      throw new Error("ApiRequestBuilder: path and method are required");
    }

    const url = new URL(path, baseUrl);
    for (const [key, value] of Object.entries(query)) url.searchParams.set(key, value);

    let lastError: unknown = null;
    for (let attempt = 0; attempt <= retries; attempt++) {
      const controller = new AbortController();
      const timer = setTimeout(() => controller.abort(), timeoutMs);
      try {
        const response = await fetch(url.toString(), {
          method,
          headers,
          body:   body === undefined ? undefined : JSON.stringify(body),
          signal: controller.signal,
        });
        if (!response.ok) {
          if (response.status >= 500 && attempt < retries) {
            lastError = new Error(`Upstream ${response.status}`);
            continue;
          }
          throw new HttpError(response.status, await response.text());
        }
        return (await response.json()) as TResponse;
      } catch (error) {
        lastError = error;
        if (attempt === retries) break;
      } finally {
        clearTimeout(timer);
      }
    }
    throw lastError instanceof Error ? lastError : new Error("Request failed");
  }
}

class HttpError extends Error {
  constructor(readonly status: number, readonly responseBody: string) {
    super(`HTTP ${status}`);
    this.name = "HttpError";
  }
}

// ── 2.4 A service client composed from a shared prefix ─────────────────────

class UserServiceClient {
  private readonly base: ApiRequestBuilder<never, unknown>;

  constructor(baseUrl: string, private readonly authToken: string) {
    this.base = ApiRequestBuilder.forService(baseUrl)
      .bearer(authToken)
      .timeout(8_000)
      .retry(2);
    // Because the builder is immutable, `base` can be reused for every call
    // below with zero risk of one request's headers leaking into another.
  }

  async listUsers(status: UsersRow["status"], limit = 50): Promise<ApiResponse<UserDto[]>> {
    return this.base
      .get("/v1/users")
      .param("status", status)
      .param("limit", limit)
      .expect<ApiResponse<UserDto[]>>()
      .send();
    // Return type is ApiResponse<UserDto[]> — inferred, not asserted at the call site.
  }

  async getUser(userId: number): Promise<ApiResponse<UserDto>> {
    return this.base.get(`/v1/users/${userId}`).expect<ApiResponse<UserDto>>().send();
  }

  async createUser(requestBody: CreateUserBody, idempotencyKey: string): Promise<ApiResponse<UserDto>> {
    return this.base
      .post("/v1/users")
      .idempotencyKey(idempotencyKey)
      .json(requestBody)
      .expect<ApiResponse<UserDto>>()
      .send();
  }

  async deleteUser(userId: number): Promise<void> {
    await this.base.delete(`/v1/users/${userId}`).expect<void>().send();
  }
}

// ── 2.5 Compile-time failures in the client ────────────────────────────────

ApiRequestBuilder.forService("https://users.internal").bearer("tok_1").send();
// ❌ 'this' context of type 'ApiRequestBuilder<never, unknown>' is not assignable
//    to method's 'this' of type 'ApiRequestBuilder<"path" | "method", unknown>'
//    ↑ no verb was chosen

async function misuse(client: UserServiceClient) {
  const result = await client.getUser(42);
  result.data.emailAddress;
  // ❌ Property 'emailAddress' does not exist on type 'UserDto'. Did you mean 'email'?
  result.data.email;      // ✅ string
  result.requestId;       // ✅ string
}
```

---

## Going deeper

### Why type-state needs a phantom field (variance, precisely)

TypeScript compares generic class types **structurally**, by comparing their members after substituting the type argument. If `TSet` never appears in any member's type, the two instantiations are identical and the gate silently passes:

```ts
class Leaky<TSet extends "a" | "b" = never> {
  private data: Record<string, unknown> = {};
  // TSet appears NOWHERE in a member type.
  markA(): Leaky<TSet | "a"> { return this as Leaky<TSet | "a"> }
  build(this: Leaky<"a" | "b">): string { return "ok" }
}
// `markA` mentions TSet in its RETURN type, which does put TSet into the
// structural comparison — so this particular class happens to be checked.
// But delete markA and the gate evaporates:

class ReallyLeaky<TSet extends "a" | "b" = never> {
  build(this: ReallyLeaky<"a" | "b">): string { return "ok" }
}
declare const rl: ReallyLeaky<never>;
rl.build();   // ✅ COMPILES — nothing distinguishes ReallyLeaky<never> from ReallyLeaky<"a"|"b">
```

The fix — always, unconditionally:

```ts
class Sealed<TSet extends "a" | "b" = never> {
  declare private readonly __set: TSet;    // phantom, type-only
  build(this: Sealed<"a" | "b">): string { return "ok" }
}
declare const s: Sealed<never>;
s.build();
// ❌ 'this' context of type 'Sealed<never>' is not assignable to 'Sealed<"a" | "b">'
```

Three properties of the phantom field matter:

1. **`declare`** — TypeScript emits no code for it (with `useDefineForClassFields`, a plain `private __set!: TSet` would emit `__set = undefined` and add a real property).
2. **`private`** — a private member makes the class *nominally* typed for assignability, so unrelated classes with the same shape are not interchangeable. That is usually what you want for a builder.
3. **`readonly`** — pure hygiene; nobody can write to a field that does not exist anyway.

You can also use an optional phantom on an interface: `readonly __set?: TSet`. Optional works but is weaker — `{ }` is assignable to `{ __set?: "a" }`.

### The one cast every type-state builder needs

An immutable builder can avoid casts entirely because it constructs a genuinely new instance with an explicit type argument:

```ts
url(value: string): RequestBuilder<TSet | "url"> {
  return new RequestBuilder<TSet | "url">({ ...this.draft, url: value });  // no cast ✅
}
```

A **mutable** type-state builder cannot, because `this` has type `Builder<TSet>` and you need to return `Builder<TSet | "url">`:

```ts
url(value: string): MutableTS<TSet | "url"> {
  this.draft.url = value;
  return this as unknown as MutableTS<TSet | "url">;   // unavoidable
}
```

`as unknown as` is required rather than a direct `as`, because `MutableTS<TSet>` and `MutableTS<TSet | "url">` do not overlap sufficiently for a single-step assertion once the phantom field is present. This is another argument for immutable builders: fewer lies.

### `build()` cannot narrow the internal draft

Even with a perfect type-state gate, the compiler does **not** propagate that knowledge into the draft object:

```ts
interface Draft { readonly url?: string; readonly method?: HttpMethod }

class B<TSet extends "url" | "method" = never> {
  declare private readonly __set: TSet;
  constructor(private readonly draft: Draft) {}

  build(this: B<"url" | "method">): { url: string; method: HttpMethod } {
    return { url: this.draft.url, method: this.draft.method };
    // ❌ Type 'string | undefined' is not assignable to type 'string'.
  }
}
```

The type-state lives in `TSet`; the optionality lives in `Draft`. Nothing connects them. Three ways out:

```ts
// (a) Non-null assertions — shortest, and safe *if* the gate is correct:
build(this: B<"url" | "method">) {
  return { url: this.draft.url!, method: this.draft.method! };
}

// (b) A runtime guard — safe even if someone casts around the gate:
build(this: B<"url" | "method">) {
  const { url, method } = this.draft;
  if (url === undefined || method === undefined) throw new Error("incomplete builder");
  return { url, method };
}

// (c) Link the draft type to TSet with a mapped type — most precise, most complex:
type DraftFor<TSet extends "url" | "method"> =
  { readonly [K in TSet & "url"]: string } &
  { readonly [K in TSet & "method"]: HttpMethod } &
  { readonly url?: string; readonly method?: HttpMethod };
```

(c) is elegant on paper and painful in practice — the intersection grows with every field and error messages become enormous. **Use (b).** The runtime check costs nanoseconds, documents the invariant, and protects against `as any` at the call site. Type-state is a developer-experience feature; keep one runtime assertion as the actual safety net.

### Method chaining and error message quality

Type-state errors point at the **whole chain**, not the missing call, and read badly by default:

```ts
RequestBuilder.create().url("https://x").build();
// ❌ The 'this' context of type 'RequestBuilder<"url">' is not assignable to
//    method's 'this' of type 'RequestBuilder<"url" | "method">'.
//      Type '"url"' is not assignable to type '"url" | "method"'.
```

A developer who has never seen type-state will not decode "Type '"url"' is not assignable to type '"url" | "method"'" as "you forgot `.method()`". Improve it by moving the gate into a parameter with a template-literal message:

```ts
type MissingMessage<TMissing extends string> = [TMissing] extends [never]
  ? []
  : [reason: `Cannot build: still need to call .${TMissing}()`];

declare class FriendlyBuilder<TSet extends "url" | "method" = never> {
  url(value: string): FriendlyBuilder<TSet | "url">;
  method(value: HttpMethod): FriendlyBuilder<TSet | "method">;
  build(...reason: MissingMessage<Exclude<"url" | "method", TSet>>): HttpRequestSpec;
}

declare const fb: FriendlyBuilder<"url">;
fb.build();
// ❌ Expected 1 arguments, but got 0.
//    Argument of type 'Cannot build: still need to call .method()' expected.
```

Still not perfect (the "Expected 1 arguments" prefix is noise), but the actionable sentence is now in the message. For a builder used by dozens of engineers, this investment pays back immediately.

### Type-parameter growth and compiler performance

Each accumulating type parameter costs the checker work on every call in the chain. A 12-call chain on a builder with four accumulating parameters instantiates the class type 12 times with progressively larger unions, and every `Pick<TRow, TSelected>` gets recomputed.

Practical limits from real codebases:

- **1–3 accumulating type parameters** is fine.
- **4+** and you start seeing measurable editor lag on long chains.
- **Deeply recursive conditional types inside the parameters** (e.g. computing a joined row shape from a tuple of join descriptors) is where query builders like Kysely and Drizzle spend their complexity budget — and where they occasionally hit `Type instantiation is excessively deep and possibly infinite`.

Mitigations:

```ts
// (a) Cache expensive computed types behind an interface instead of recomputing:
interface Selected<TRow, TKeys extends keyof TRow> { readonly rows: Pick<TRow, TKeys>[] }

// (b) Prefer unions of string keys (cheap) over object accumulation (expensive):
//     TSelected = "userId" | "email"        ✅ cheap
//     TSelected = { userId: number } & { email: string }   ❌ intersections pile up

// (c) Break very long chains — assign an intermediate const so the checker
//     resolves the type once instead of carrying a 12-deep instantiation:
const partialQuery = QueryBuilder.from("orders").select("orderId", "totalCents");
const finalQuery = partialQuery.where("tenantId", "=", "acme").limit(10).compile();
```

### Overloads vs conditional types for gating

You may be tempted to gate `build()` with overloads. It does not work the way you want:

```ts
declare class OverloadAttempt<TSet extends "a" | "b"> {
  build(this: OverloadAttempt<"a" | "b">): string;
  build(this: OverloadAttempt<never>): never;
  // Overload resolution picks the FIRST matching signature, and error messages
  // become "No overload matches this call" with N candidate errors listed.
  // Strictly worse DX than a single gated signature.
}
```

Single signature with a `this` parameter, or a single signature with a conditional parameter list. Never overloads.

### Interaction with `strictFunctionTypes` and method bivariance

`this` parameters and method parameters are checked **bivariantly** when declared as methods (`build(this: X): Y`) and **contravariantly** when declared as properties with function types (`build: (this: X) => Y`), under `strictFunctionTypes`. For type-state gating you want the *stricter* behaviour, which means bivariance can occasionally let something through:

```ts
class Bivariant<TSet extends "a" | "b" = never> {
  declare private readonly __set: TSet;
  build(this: Bivariant<"a" | "b">): string { return "x" }
}
// `this`-parameter checking is NOT subject to strictFunctionTypes bivariance —
// TypeScript checks `this` parameters contravariantly on methods specifically
// for this pattern. So the gate holds. But if you ever hand the method around
// as a free function, the check is done at extraction time, not call time:

declare const bv: Bivariant<never>;
const extracted = bv.build;   // ❌ error surfaces HERE, which is what you want
```

The takeaway: gate with `this` parameters on **methods**, and do not let callers destructure builder methods off the instance (`const { build } = builder`), because you lose the receiver and the check moves.

### Builders and `readonly` — they compose beautifully

The immutable builder's state should be `readonly` at every level, and the built value should be `readonly` too. See **65 — Readonly and immutability patterns**:

```ts
interface BuiltRequest {
  readonly url:     string;
  readonly method:  HttpVerb;
  readonly headers: Readonly<Record<string, string>>;
}

// The builder guarantees COMPLETENESS; readonly guarantees STABILITY after build.
// Together: you cannot construct an incomplete request, and you cannot corrupt
// a complete one. Neither guarantee alone is sufficient.
```

### Runtime cost of a builder, measured honestly

For an immutable builder with a 6-call chain producing one request:

- 6 builder instances + 6 spread copies of a small state object.
- Roughly 12 short-lived allocations, all young-generation, all collected in a scavenge.
- On a 10k rps service that's 120k tiny allocations/sec — V8 handles this without a blip; it is a rounding error next to `JSON.stringify` on the body.

Where it genuinely matters:

- **A builder inside a per-row loop** over a 500k-row result set. Use a mutable builder, or skip the builder and construct the object directly.
- **A builder whose state contains a large array copied on every call.** `[...this.conditions, next]` is O(n) per call, so a chain of n calls is O(n²). Fine for n < 50; measure beyond that. If you need thousands of conditions, accumulate into a mutable array inside a mutable builder and expose `readonly` at the end.

### The `satisfies` alternative for simple cases

Before reaching for a builder, check whether `satisfies` plus a well-typed options object already gives you what you want:

```ts
interface JobOptions {
  readonly queueName:   string;
  readonly maxAttempts: number;
  readonly backoff:     { readonly type: "exponential" | "fixed"; readonly delayMs: number };
  readonly priority?:   number;
}

const emailJobOptions = {
  queueName:   "transactional-email",
  maxAttempts: 5,
  backoff:     { type: "exponential", delayMs: 1_000 },
} as const satisfies JobOptions;
// ✅ required fields enforced, literal types preserved, typos caught, zero machinery
```

If required-field enforcement is all you need, the options object already does it. The builder earns its keep only when you need **progressive type refinement** (`select()` changing the row type) or **ordering/dependency constraints** (`offset()` requires `limit()`).

---

## Common mistakes

### Mistake 1 — Returning the class type instead of `this` from a chainable method

```ts
// ❌ Wrong — subclass methods vanish after the first base-class call:
class BaseQuery {
  protected parts: string[] = [];
  where(clause: string): BaseQuery {          // ← hard-coded class name
    this.parts.push(clause);
    return this;
  }
}

class PagedQuery extends BaseQuery {
  page(n: number): PagedQuery {
    this.parts.push(`OFFSET ${(n - 1) * 20}`);
    return this;
  }
}

new PagedQuery().where("status = 'active'").page(2);
// ❌ Property 'page' does not exist on type 'BaseQuery'.

// ✅ Right — polymorphic `this`:
class BaseQueryFixed {
  protected parts: string[] = [];
  where(clause: string): this {               // ← `this`
    this.parts.push(clause);
    return this;
  }
}

class PagedQueryFixed extends BaseQueryFixed {
  page(n: number): this {
    this.parts.push(`OFFSET ${(n - 1) * 20}`);
    return this;
  }
}

new PagedQueryFixed().where("status = 'active'").page(2).where("deleted_at IS NULL");
// ✅ every call keeps the PagedQueryFixed type
```

### Mistake 2 — Type-state on a mutable builder (the aliasing lie)

```ts
// ❌ Wrong — the TYPE forks but the OBJECT doesn't:
class MutableTypeState<TSet extends "url" | "method" = never> {
  declare private readonly __set: TSet;
  private draft: { url?: string; method?: string } = {};

  url(value: string): MutableTypeState<TSet | "url"> {
    this.draft.url = value;                                  // mutates shared state
    return this as unknown as MutableTypeState<TSet | "url">;
  }
  method(value: string): MutableTypeState<TSet | "method"> {
    this.draft.method = value;
    return this as unknown as MutableTypeState<TSet | "method">;
  }
  build(this: MutableTypeState<"url" | "method">) {
    return { ...this.draft };
  }
}

const shared = new MutableTypeState().url("https://api.example.com/a");
const getReq  = shared.method("GET").build();
const postReq = shared.method("POST").build();
// getReq.method === "POST"  💥 — one object, two "different" typed views.
// The type system said these were independent. They weren't.

// ✅ Right — immutable: each call constructs a new instance:
class ImmutableTypeState<TSet extends "url" | "method" = never> {
  declare private readonly __set: TSet;
  private constructor(private readonly draft: { readonly url?: string; readonly method?: string }) {}

  static create(): ImmutableTypeState<never> { return new ImmutableTypeState<never>({}) }

  url(value: string): ImmutableTypeState<TSet | "url"> {
    return new ImmutableTypeState<TSet | "url">({ ...this.draft, url: value });
  }
  method(value: string): ImmutableTypeState<TSet | "method"> {
    return new ImmutableTypeState<TSet | "method">({ ...this.draft, method: value });
  }
  build(this: ImmutableTypeState<"url" | "method">) {
    return { ...this.draft } as { url: string; method: string };
  }
}

const base = ImmutableTypeState.create().url("https://api.example.com/a");
const g = base.method("GET").build();    // { url: ".../a", method: "GET" }
const p = base.method("POST").build();   // { url: ".../a", method: "POST" }  ✅
```

**Rule: if the builder's type changes across calls, the builder's value must too.**

### Mistake 3 — Omitting the phantom field, so the gate does nothing

```ts
// ❌ Wrong — TSet appears in no member type, so all instantiations are identical:
class NoPhantom<TSet extends "path" | "verb" = never> {
  private state: Record<string, unknown> = {};
  build(this: NoPhantom<"path" | "verb">): string { return "compiled" }
}

declare const np: NoPhantom<never>;
np.build();   // ✅ COMPILES — the "gate" is decorative

// ✅ Right — a declared private phantom field puts TSet into the comparison:
class WithPhantom<TSet extends "path" | "verb" = never> {
  declare private readonly __set: TSet;
  private state: Record<string, unknown> = {};
  build(this: WithPhantom<"path" | "verb">): string { return "compiled" }
}

declare const wp: WithPhantom<never>;
wp.build();
// ❌ The 'this' context of type 'WithPhantom<never>' is not assignable to
//    method's 'this' of type 'WithPhantom<"path" | "verb">'.
```

**Always add the phantom field.** Even when the setters' return types happen to make the class comparable today, a refactor that removes or reshapes one method can silently disarm the gate — and nothing will fail.

### Mistake 4 — Widening the selection type by taking an array instead of a rest parameter

```ts
interface ProductsRow { productId: string; name: string; priceCents: number; sku: string }

// ❌ Wrong — the argument widens to string[], so K is `string`, and Pick breaks:
declare class ArrayBuilder<TRow, TSelected extends keyof TRow = keyof TRow> {
  select<K extends keyof TRow>(columns: K[]): ArrayBuilder<TRow, K>;
  rows(): Pick<TRow, TSelected>[];
}
declare const ab: ArrayBuilder<ProductsRow>;
const wrong = ab.select(["productId", "name"]).rows();
// K infers as "productId" | "name" here because of the keyof constraint — but
// the moment the constraint loosens (K extends string), you get K = string
// and Pick<TRow, string> collapses. And callers who pass a pre-built array lose it:
const requestedColumns = ["productId", "name"];   // string[]
ab.select(requestedColumns);
// ❌ Argument of type 'string[]' is not assignable to 'ProductsRow[keyof ...][]'

// ✅ Right — rest parameter (infers a literal union directly), plus `const`
//    type parameter for callers who must pass an array:
declare class RestBuilder<TRow, TSelected extends keyof TRow = keyof TRow> {
  select<const K extends keyof TRow>(...columns: K[]): RestBuilder<TRow, K>;
  selectAll<const K extends readonly (keyof TRow)[]>(columns: K): RestBuilder<TRow, K[number]>;
  rows(): Pick<TRow, TSelected>[];
}
declare const rb: RestBuilder<ProductsRow>;

const right1 = rb.select("productId", "name").rows();
// { productId: string; name: string }[]  ✅

const right2 = rb.selectAll(["productId", "priceCents"] as const).rows();
// { productId: string; priceCents: number }[]  ✅
```

### Mistake 5 — Forgetting the runtime check because "the types guarantee it"

```ts
// ❌ Wrong — one `as any` anywhere and build() returns a broken object:
class TrustingBuilder<TSet extends "url" | "method" = never> {
  declare private readonly __set: TSet;
  private constructor(private readonly draft: { readonly url?: string; readonly method?: string }) {}
  static create() { return new TrustingBuilder<never>({}) }
  url(v: string) { return new TrustingBuilder<TSet | "url">({ ...this.draft, url: v }) }
  build(this: TrustingBuilder<"url" | "method">) {
    return { url: this.draft.url!, method: this.draft.method! };   // trusts the gate
  }
}

const sneaky = TrustingBuilder.create().url("https://api.example.com") as any;
const broken = sneaky.build();
// { url: "https://api.example.com", method: undefined }  💥
// Sent to fetch() → TypeError deep inside undici, far from the cause.

// ✅ Right — keep ONE cheap runtime assertion at the gate:
build(this: TrustingBuilder<"url" | "method">) {
  const { url, method } = this.draft;
  if (url === undefined || method === undefined) {
    throw new Error(`Builder incomplete: missing ${[!url && "url", !method && "method"].filter(Boolean).join(", ")}`);
  }
  return { url, method };
}
// Fails immediately, at the builder, with the field name in the message.
```

Type-state is a **developer-experience** feature, not a runtime safety mechanism — types are erased. Boundaries that receive `any` (JSON, JS callers, dynamic dispatch) can always defeat it.

### Mistake 6 — Building a builder when an options object was enough

```ts
// ❌ Overkill — 40 lines of builder for a 4-field config with no ordering
//    constraints, no progressive typing, and no reusable prefix:
class CacheConfigBuilder {
  private ttlSeconds = 300;
  private maxEntries = 1_000;
  private keyPrefix = "";
  private staleWhileRevalidate = false;
  ttl(seconds: number): this { this.ttlSeconds = seconds; return this }
  max(entries: number): this { this.maxEntries = entries; return this }
  prefix(value: string): this { this.keyPrefix = value; return this }
  swr(enabled: boolean): this { this.staleWhileRevalidate = enabled; return this }
  build(): CacheConfig { /* ... */ return {} as CacheConfig }
}
new CacheConfigBuilder().ttl(60).max(500).prefix("users:").build();

// ✅ Right — a plain options object with defaults. Shorter, faster, and the
//    compiler already enforces required fields and rejects typos:
interface CacheConfig {
  readonly ttlSeconds:           number;
  readonly maxEntries?:          number;
  readonly keyPrefix?:           string;
  readonly staleWhileRevalidate?: boolean;
}

function createCache(options: CacheConfig) {
  const { ttlSeconds, maxEntries = 1_000, keyPrefix = "", staleWhileRevalidate = false } = options;
  /* ... */
}

createCache({ ttlSeconds: 60, maxEntries: 500, keyPrefix: "users:" });
createCache({ maxEntries: 500 });
// ❌ Property 'ttlSeconds' is missing — required-field enforcement, for free
```

---

## Practice exercises

### Exercise 1 — easy

Build a fluent, immutable `ApiResponseBuilder` for an Express-style backend.

Requirements:

1. Define `ApiResponse<TBody>` with `readonly status: number`, `readonly headers: Readonly<Record<string, string>>`, `readonly body: TBody`, `readonly requestId: string`.
2. Build an **immutable** builder class `ApiResponseBuilder<TBody = null>` whose private constructor takes a readonly state object and whose `static create(requestId: string)` is the only public entry point.
3. Chainable methods: `status(code: number)`, `header(name: string, value: string)`, `cacheFor(seconds: number)` (which sets `Cache-Control: public, max-age=N`), `noStore()` (sets `Cache-Control: no-store`), and `json<T>(body: T)` which must return a builder whose `TBody` is now `T`.
4. `build(): ApiResponse<TBody>` — defaults status to `200` and body to `null`.
5. Prove immutability: create a shared prefix builder with the request ID and a `X-Service: users-api` header, then derive two different responses from it and show in comments that neither affected the other.
6. Add commented-out lines showing two compile errors: reading a property that isn't on the built body type, and passing a non-numeric status.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a type-state `EmailMessageBuilder` for a transactional email service.

Requirements:

1. Required fields: `to`, `subject`, and exactly one of `templateId` or `htmlBody` (never both, never neither). Optional: `cc`, `bcc`, `replyTo`, `attachments`, `sendAt`, `tags`.
2. Track required fields in a type-state parameter `TSet` starting at `never`. Include a `declare private readonly __set: TSet` phantom field.
3. Model the "exactly one of templateId / htmlBody" constraint in the types: after `.template(id, vars)` the builder must expose no `.html()` method, and after `.html(markup)` it must expose no `.template()` method. Hint: use two builder interfaces and have the shared setters return `this`-preserving generics, or add a second type parameter `TContent extends "none" | "template" | "html"` and gate the two methods with `this` parameters.
4. `.to(address: string)` and `.cc(...addresses: string[])` should validate the shape of an email at the type level using a template literal type `` `${string}@${string}.${string}` ``.
5. `.template<TVars extends Record<string, string | number>>(templateId: string, variables: TVars)` should record `TVars` in the builder's type so the built message's `variables` field is exactly `TVars`, not `Record<string, unknown>`.
6. `send(this: Builder<AllRequired, "template" | "html", ...>): Promise<{ messageId: string }>` — gated so it only compiles when `to`, `subject`, and a content source are all present.
7. Keep the builder immutable. Show a shared prefix (from address + tags) used to derive two different emails.
8. Add at least five commented-out lines showing the distinct compile errors: missing `to`, missing subject, missing content, calling `.html()` after `.template()`, and passing `"not-an-email"` to `.to()`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a typed, multi-table query builder with joins, aggregation, and full type-state safety — the core of a mini ORM.

Requirements:

1. Define a `Schema` with at least four tables: `users`, `orders`, `orderItems`, `products`, each with a `tenantId` column and realistic columns and types (including a nullable column and a literal-union column).
2. Implement `QueryBuilder<TSchema, TFrom extends keyof TSchema, TJoined, TSelected, TFiltered, TGrouped>` — six type parameters. Yes, this is deliberately at the edge of what's reasonable; part of the exercise is feeling that edge.
3. `.from(table)` is a static entry point returning a builder whose row type is `TSchema[TFrom]`.
4. `.innerJoin(table, leftColumn, rightColumn)` must:
   - only accept column pairs whose **types are compatible** (e.g. `orders.userId: number` can join `users.userId: number`, but not `users.email: string`),
   - extend the available row shape to a namespaced union (e.g. keys become `` `users.email` | `orders.totalCents` ``, built with template literal types),
   - be reflected in `TJoined` so subsequent `.select()` and `.where()` accept the joined columns.
5. `.leftJoin(...)` must do the same but make every column of the joined table **nullable** in the result type.
6. `.select(...columns)` narrows the result row type using `Pick`-like logic over the namespaced key union, producing a flat object type with the namespaced keys.
7. `.where(column, operator, value)` must type the value from the column, with operator-aware types (`IN` → array, `IS NULL` → no value / null, `LIKE` → string columns only).
8. `.groupBy(...columns)` records `TGrouped`; `.count(alias)`, `.sum(column, alias)`, and `.avg(column, alias)` add computed numeric columns to the result type. `.sum()` and `.avg()` must reject non-numeric columns at compile time.
9. `.having(...)` must only be callable after `.groupBy(...)` — enforce with a `this` parameter.
10. `.compile()` must be gated on `tenantId` having been filtered (multi-tenant safety) and must return a **branded** `CompiledQuery<TResultRow>` that only `execute()` accepts.
11. `execute(pool, query)` returns `Promise<TResultRow[]>` with no casts at the call site.
12. Write a `describePlan(): readonly string[]` method returning a human-readable summary of the query built so far (usable before `compile()`, at any type-state).
13. Demonstrate with comments: at least **eight** distinct compile errors — bad column name, wrong value type for a column, incompatible join columns, `sum()` on a string column, `having()` before `groupBy()`, missing tenant filter, selecting a column from an unjoined table, and executing a hand-written (unbranded) query object.
14. Finally, write a short comment answering: at what point did the type machinery stop paying for itself, and which two of the six type parameters would you drop in a real codebase?

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Fluent chaining ─────────────────────────────────────────────────────────
method(x: T): this            // mutable builder — subclass-safe
method(x: T): Builder         // immutable builder — returns a NEW instance

// ── Type-state: track set fields in a union type parameter ──────────────────
class B<TSet extends Key = never> {
  declare private readonly __set: TSet;          // PHANTOM — required, always
  setA(v: A): B<TSet | "a"> { return new B<TSet | "a">({ ...this.s, a: v }) }
  build(this: B<Key>): Built { /* gate via `this` parameter */ }
}

// ── Gating build(): three styles ────────────────────────────────────────────
build(this: B<AllKeys>): Built;                            // `this` parameter (classes)
build: [Exclude<AllKeys, TSet>] extends [never] ? () => Built : never;   // conditional
build(...err: Missing<Exclude<AllKeys, TSet>>): Built;     // message in a parameter

type Missing<M extends string> = [M] extends [never] ? [] : [reason: `Need .${M}()`];

// ── Narrowing the result type ───────────────────────────────────────────────
select<const K extends keyof TRow>(...cols: K[]): Builder<TRow, K>;
execute(): Promise<Pick<TRow, TSelected>[]>;

// ── Typed conditions (value type follows the column) ────────────────────────
where<K extends keyof TRow>(col: K, op: Operator, value: TRow[K]): Builder<TRow>;

// Operator-aware:
type ValueFor<V, Op> = Op extends "IN" ? readonly V[]
                     : Op extends "IS NULL" ? null
                     : Op extends "LIKE" ? (V extends string ? string : never)
                     : V;

// ── Brands / phantom types ──────────────────────────────────────────────────
declare const brand: unique symbol;
type Brand<T, Tag extends string> = T & { readonly [brand]: Tag };
type CompiledSql = Brand<string, "CompiledSql">;   // only the builder can make one

declare private readonly __state: TState;          // phantom class field
//      ↑ `declare` = type-only, emits nothing

// ── Immutable copy-on-write ─────────────────────────────────────────────────
private constructor(private readonly state: State) {}
static create(): Builder<never> { return new Builder<never>(initialState) }
setX(v: X): Builder<TSet | "x"> { return new Builder<TSet | "x">({ ...this.state, x: v }) }
```

| Technique | What it buys you | Cost |
|---|---|---|
| `return this` + `: this` | chaining that survives subclassing | none |
| Mutable builder | one allocation, simple | aliasing bugs on reuse |
| Immutable builder | safe prefixes, real type-state | one small object per call |
| Type-state (`TSet` union) | `build()` won't compile if incomplete | phantom field + 1 internal cast |
| `this`-parameter gate | best error locality for classes | classes only |
| Conditional-type gate | works on interfaces/factories, custom messages | `[X] extends [never]` subtlety |
| `select<K extends keyof T>` | result rows typed from the chain | 1 extra type parameter |
| Typed `where()` | value type follows the column | none |
| Operator-aware `where()` | `IN` needs an array, `LIKE` needs a string | 1 conditional type |
| Brands on output | only builder-produced values are accepted | `unique symbol` + factory discipline |

| Situation | Use a builder? |
|---|---|
| 2–5 independent optional fields, no ordering rules | ❌ options object + `satisfies` |
| Required fields only | ❌ options object — the compiler already enforces them |
| Result type must change with the chain (`select`) | ✅ builder |
| Method B is only legal after method A | ✅ builder with type-state |
| Reusable partially-configured prefix (base client) | ✅ **immutable** builder |
| Test fixtures with sensible defaults | ✅ builder (`aUser().withRole("admin").build()`) |
| Called hundreds of thousands of times in a loop | ❌ construct the object directly |
| Config loaded once at boot | ❌ `as const satisfies ConfigShape` |
| Public library API used by many teams | ✅ builder — discoverability wins |
| 6+ accumulating type parameters | ⚠️ you are past the payoff; simplify |

---

## Connected topics

- **65 — Readonly and immutability patterns** — the `readonly` state objects, copy-on-write spreads, and `as const satisfies` that immutable builders are built out of.
- **40 — Conditional types** — `[Exclude<All, TSet>] extends [never]` gating, the distribution trap, and `never` as the "uncallable" signal.
- **43 — Mapped types** — `Pick`, `Omit`, and the distributive `{ [K in keyof T]: ... }[keyof T]` pattern behind typed `where()` conditions.
- **39 — Generic constraints** — `K extends keyof TRow`, the `const` type-parameter modifier, and inference from rest parameters.
- **35 — Readonly class properties** — `private readonly` fields, `declare` members, and constructor-only assignment in builder classes.
- **11 — Literal types** — why `as const` and `const` type parameters are what make `.select("userId", "email")` infer a union instead of `string`.
- **48 — Branded / nominal types** — `unique symbol` brands, phantom type parameters, and modelling provenance in the type system.
- **32 — Utility types** — `Partial`, `Pick`, `Exclude`, `Extract`, and `Record`, all of which appear in builder signatures.
- **13 — Arrays and tuples** — rest parameters, variadic tuple types, and the `readonly T[]` inference rules that `select(...columns)` depends on.
