# 44 — Conditional Types

## What is this?

A **conditional type** is an `if / else` that runs in the type system instead of at runtime. It has exactly one form:

```ts
type IsString<T> = T extends string ? "yes" : "no";
//                 ^^^^^^^^^^^^^^^^   ^^^^^   ^^^^
//                 the condition      then    else

type A = IsString<string>;   // "yes"
type B = IsString<number>;   // "no"
```

Read `T extends U` as **"is `T` assignable to `U`?"** — not "does `T` inherit from `U`". It is the same assignability question the compiler asks when you pass an argument to a function. If the answer is yes, the whole type expression becomes the `true` branch; if no, the `false` branch.

That alone is mildly interesting. What makes conditional types the most powerful feature in the language is the two things bolted onto it:

```ts
// (1) `infer` — declare a type variable INSIDE the condition and capture whatever
//     the compiler had to match to make the condition true.
type ElementOf<T> = T extends (infer Element)[] ? Element : never;
type UserIds = ElementOf<number[]>;   // number

// (2) DISTRIBUTION — when the checked type is a naked type parameter and the
//     argument is a union, the conditional runs once PER MEMBER and re-unions.
type NonNullish<T> = T extends null | undefined ? never : T;
type Cleaned = NonNullish<string | null | number | undefined>;   // string | number
```

`infer` is pattern matching. Distribution is `Array.prototype.filter` / `.map` for unions. Between them you can pull the return type out of a function, the resolved value out of a `Promise`, the element out of an array, the payload out of a discriminated union member, and filter or transform unions member by member. Every "magic" utility type in `lib.es5.d.ts` — `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `InstanceType`, `Awaited` — is a conditional type of three to six lines, and by the end of this doc you will have written all of them yourself.

## Why does it matter?

Backend TypeScript is mostly **plumbing types between layers**, and every joint in that plumbing is a conditional type:

- A route handler returns `Promise<UserResponse>`. Your test helper, your OpenAPI generator, and your client SDK all need `UserResponse` — the *unwrapped* type. Hand-writing it means it drifts. `Awaited<ReturnType<typeof getUser>>` never drifts.
- A repository method's argument type should follow from the table type. `find(where: WhereClause<UserTable>)` needs `Comparators<T[K]>` — different operator sets for `number` columns, `string` columns, and `boolean` columns. That is a conditional type in a mapped type's value slot.
- A middleware chain narrows `Request` progressively: `Request` → `Request & { userId: number }` → `Request & { userId: number; tenantId: string }`. Expressing "the request type after this middleware" is a conditional type over the middleware's own type.
- An event union has fifteen members and you want only the ones whose `type` starts with `"payment."`. That is distribution plus a template-literal check.
- You have `JSON.parse` returning `unknown` and a Zod-ish schema object; the *output type* of the schema is a recursive conditional type over the schema's shape.
- A function should return `T` when given `T`, but `T | null` when given a `findOne`-style flag. The return type is conditional on a generic parameter.

Without conditional types you write those relationships as comments and casts, and casts are where production bugs are born. With them, the relationship is *checked*: change the function, and every derived type changes with it or fails the build.

The other reason it matters: **you will read them long before you write them**. Every serious library — Express typings, Prisma, tRPC, Zod, TypeORM, Fastify — is built out of conditional types. Being unable to read `T extends (...args: infer P) => infer R ? ... : ...` means every type error from those libraries is an unreadable wall.

## The JavaScript way vs the TypeScript way

There is no JavaScript equivalent of a conditional type — JavaScript has no types to branch on. The honest comparison is against **what a JS developer does instead**: runtime `typeof` checks, hand-written duplicate signatures, JSDoc that lies, and defensive casts.

```js
// JavaScript — every type relationship is a comment, a convention, or a runtime check.

// ── Problem 1: "what does this function return?" ─────────────────────────────
async function getUser(userId) {
  const row = await db.query("SELECT * FROM users WHERE user_id = $1", [userId]);
  if (!row) return null;
  return {
    userId:      row.user_id,
    email:       row.email,
    displayName: row.display_name,
    createdAt:   new Date(row.created_at),
  };
}

// Now a caller wants to build a response DTO from it. They have to RE-DECLARE
// the shape in a comment and hope it stays true:
/**
 * @param user  { userId, email, displayName, createdAt }   <-- copy-pasted, will drift
 */
function toResponse(user) {
  return { ...user, createdAt: user.createdAt.toISOString() };
}
// Someone renames displayName -> name inside getUser. Nothing breaks at build time.
// It breaks at 3am, in production, for one customer, in a field nobody reads.

// ── Problem 2: "unwrap this, whatever it is" ─────────────────────────────────
// You cannot ask "is this a promise?" statically, so you check at runtime, forever:
async function resolveMaybe(value) {
  return value && typeof value.then === "function" ? await value : value;
}
// The caller has no idea what came out. It's `any`, spiritually.

// ── Problem 3: "filter this list of allowed values" ──────────────────────────
const ALL_EVENTS   = ["user.created", "user.deleted", "payment.failed", "payment.settled"];
const PAYMENT_ONLY = ALL_EVENTS.filter((e) => e.startsWith("payment."));
// PAYMENT_ONLY is correct at runtime. But a function that only accepts payment
// events cannot express that — it takes `string`, and typos sail through.
function handlePayment(eventName) {
  // eventName could be "usre.created", "", 42, or undefined. No help at all.
}

// ── Problem 4: "the return type depends on an argument" ──────────────────────
function findUsers(where, { single = false } = {}) {
  const rows = db.queryAll(where);
  return single ? rows[0] ?? null : rows;   // object|null OR an array. Caller guesses.
}
const a = findUsers({ isAdmin: true });                  // array
const b = findUsers({ userId: 1 }, { single: true });    // object | null
b.length;   // undefined at runtime. No warning. Ships.
```

Four problems, four kinds of drift, zero enforcement. Now the same four with conditional types:

```ts
// TypeScript — every relationship above becomes a compiler-checked derivation.

interface UserRow {
  user_id:      number;
  email:        string;
  display_name: string;
  created_at:   string;
}

async function getUser(userId: number): Promise<{
  userId:      number;
  email:       string;
  displayName: string;
  createdAt:   Date;
} | null> {
  const row = await db.query<UserRow>("SELECT * FROM users WHERE user_id = $1", [userId]);
  if (!row) return null;
  return {
    userId:      row.user_id,
    email:       row.email,
    displayName: row.display_name,
    createdAt:   new Date(row.created_at),
  };
}

// ── Fix 1: derive the shape instead of copying it ───────────────────────────
type User = NonNullable<Awaited<ReturnType<typeof getUser>>>;
//          ^ strip null   ^ unwrap Promise  ^ pull the return type out
// = { userId: number; email: string; displayName: string; createdAt: Date }

// Rename displayName inside getUser and THIS LINE breaks — at build time:
function toResponse(user: User) {
  return { ...user, createdAt: user.createdAt.toISOString() };
}

// ── Fix 2: unwrap statically, keeping the type ──────────────────────────────
type Unwrap<T> = T extends Promise<infer Resolved> ? Resolved : T;
type R1 = Unwrap<Promise<User>>;   // User
type R2 = Unwrap<User>;            // User   ← non-promises pass through unchanged

// ── Fix 3: filter a union of literals by pattern ────────────────────────────
type EventName =
  | "user.created" | "user.deleted"
  | "payment.failed" | "payment.settled";

type PaymentEvent = Extract<EventName, `payment.${string}`>;
// = "payment.failed" | "payment.settled"

function handlePayment(eventName: PaymentEvent): void { /* ... */ }
handlePayment("payment.failed");   // ✅
// handlePayment("user.created");  // ❌ Error, at build time
// handlePayment("payment.faild"); // ❌ Error, typo caught

// ── Fix 4: return type that depends on an argument ──────────────────────────
type FindResult<Single extends boolean> = Single extends true ? User | null : User[];

declare function findUsers<Single extends boolean = false>(
  where: Partial<User>,
  options?: { single?: Single },
): Promise<FindResult<Single>>;

const many = await findUsers({ isAdmin: true } as Partial<User>);   // User[]
const one  = await findUsers({ userId: 1 }, { single: true });      // User | null
// one.length;   // ❌ Error: Property 'length' does not exist on type 'User | null'

declare const db: {
  query<T>(sql: string, params: unknown[]): Promise<T | null>;
  queryAll<T>(where: unknown): T[];
};
```

The revelation: relationships between types stop being documentation and become **computation**. `User` is not a thing you maintain — it is a value computed from `getUser`, recomputed on every build.

---

## Syntax

```ts
// ── The one and only form ───────────────────────────────────────────────────
type Cond<T> = T extends SomeType ? TrueBranch : FalseBranch;
//             ^^^^^^^^^^^^^^^^^^ "is T assignable to SomeType?"

// ── Nesting = else-if chains ────────────────────────────────────────────────
type Kind<T> =
  T extends string  ? "string"
  : T extends number  ? "number"
  : T extends boolean ? "boolean"
  : T extends unknown[] ? "array"
  : "object";
// Formatting convention: leading `:` aligned in a column reads like a switch.

// ── `infer` — capture a type from inside the pattern ────────────────────────
type ElementOf<T>     = T extends (infer Element)[]        ? Element  : never;
type Resolved<T>      = T extends Promise<infer Value>     ? Value    : never;
type FirstArg<T>      = T extends (first: infer A, ...rest: never[]) => unknown ? A : never;
type Returned<T>      = T extends (...args: never[]) => infer R ? R : never;
type CtorInstance<T>  = T extends new (...args: never[]) => infer I ? I : never;

// ── Multiple infers in one pattern ──────────────────────────────────────────
type SplitFn<T> = T extends (...args: infer Args) => infer Ret
  ? { args: Args; ret: Ret }
  : never;

// ── infer with a constraint (TS 4.7+) ───────────────────────────────────────
type FirstNumeric<T> = T extends [infer Head extends number, ...unknown[]] ? Head : never;

// ── Distribution: happens when the checked type is a NAKED type parameter ───
type Boxed<T> = T extends unknown ? { value: T } : never;
type B = Boxed<string | number>;
// = { value: string } | { value: number }        ← ran once per member

// ── Blocking distribution with tuple brackets ───────────────────────────────
type BoxedWhole<T> = [T] extends [unknown] ? { value: T } : never;
type BW = BoxedWhole<string | number>;
// = { value: string | number }                   ← ran once, on the whole union

// ── `never` is the empty union, so distributing over it gives never ─────────
type D = Boxed<never>;        // never   (zero members to iterate)
type E = BoxedWhole<never>;   // { value: never }

// ── Conditional types in a function's return position ───────────────────────
declare function parseBody<Strict extends boolean>(
  raw: string,
  strict: Strict,
): Strict extends true ? Record<string, unknown> : unknown;

// ── Recursive conditional types (TS 4.1+) ───────────────────────────────────
type DeepAwaited<T> = T extends Promise<infer Inner> ? DeepAwaited<Inner> : T;
type F = DeepAwaited<Promise<Promise<Promise<string>>>>;   // string
```

---

## How it works — concept by concept

### Concept 1 — `extends` means "assignable to", not "inherits from"

This is the single most common source of confusion. `A extends B` asks: *could I pass a value of type `A` where a `B` is expected?* Nothing to do with classes.

```ts
// ── Literals extend their primitive ─────────────────────────────────────────
type T1 = "payment.failed" extends string ? true : false;   // true
type T2 = 42 extends number ? true : false;                 // true
type T3 = true extends boolean ? true : false;              // true
type T4 = string extends "payment.failed" ? true : false;   // false  ← other direction fails

// ── Objects: MORE properties means MORE specific, so it extends ─────────────
interface HttpRequest {
  method:  string;
  url:     string;
}
interface AuthedRequest extends HttpRequest {
  userId:    number;
  authToken: string;
}

type T5 = AuthedRequest extends HttpRequest ? true : false;   // true  (has everything + more)
type T6 = HttpRequest extends AuthedRequest ? true : false;   // false (missing userId)

// It is STRUCTURAL — the `extends` keyword on the interface is irrelevant:
interface LooseAuthed { method: string; url: string; userId: number; authToken: string }
type T7 = LooseAuthed extends HttpRequest ? true : false;     // true, no declared relationship

// ── Unions: a union extends U only if EVERY member does ─────────────────────
type T8  = ("a" | "b") extends string ? true : false;         // true
type T9  = ("a" | 1)   extends string ? true : false;         // false ← 1 doesn't
type T10 = string extends string | number ? true : false;     // true  ← the TARGET may be wider

// ── never extends everything; everything extends unknown ────────────────────
type T11 = never extends string ? true : false;     // true  (never is the bottom type)
type T12 = string extends unknown ? true : false;   // true  (unknown is the top type)
type T13 = unknown extends string ? true : false;   // false

// ── any is the chaos agent: it matches BOTH branches ────────────────────────
type T14 = any extends string ? "yes" : "no";       // "yes" | "no"  ← a UNION of both!
// Because `any` is assignable to everything and everything is assignable to `any`,
// the compiler cannot decide, so it returns both branches unioned. Remember this
// when a utility type mysteriously returns a union you did not expect.

// ── Functions: parameters are compared CONTRAVARIANTLY ──────────────────────
type Handler       = (request: HttpRequest) => void;
type AuthedHandler = (request: AuthedRequest) => void;

type T15 = Handler extends AuthedHandler ? true : false;
// true — a function taking the WIDER parameter can stand in where the narrower
// one is expected. Parameter positions flip the direction. (See doc 46.)
type T16 = AuthedHandler extends Handler ? true : false;
// false under strictFunctionTypes for function TYPE positions.
```

Practical framing when reading library types: mentally replace `X extends Y ?` with `if (canAssign(X, Y))`.

### Concept 2 — `infer`: pattern matching inside the condition

`infer Name` declares a fresh type variable **inside the `extends` clause**. The compiler tries to match the checked type against the pattern; whatever lands in the `infer` slot is bound to that name and is usable in the true branch only.

```ts
// ── Unwrap an array ─────────────────────────────────────────────────────────
type ElementOf<T> = T extends (infer Element)[] ? Element : never;

type E1 = ElementOf<string[]>;                       // string
type E2 = ElementOf<Array<{ userId: number }>>;      // { userId: number }
type E3 = ElementOf<readonly number[]>;              // never  ← readonly arrays don't match `(...)[]`

// Handle readonly too:
type ElementOfAny<T> = T extends readonly (infer Element)[] ? Element : never;
type E4 = ElementOfAny<readonly number[]>;           // number
type E5 = ElementOfAny<string[]>;                    // string  (mutable arrays ARE readonly arrays)

// ── Unwrap a Promise ────────────────────────────────────────────────────────
type ResolvedValue<T> = T extends Promise<infer Value> ? Value : never;
type P1 = ResolvedValue<Promise<UserRecord>>;        // UserRecord
type P2 = ResolvedValue<UserRecord>;                 // never

// ── Function pieces ─────────────────────────────────────────────────────────
type ReturnOf<T>  = T extends (...args: never[]) => infer R ? R : never;
type ParamsOf<T>  = T extends (...args: infer P) => unknown ? P : never;

declare function createSession(
  userId: number,
  authToken: string,
  ttlSeconds?: number,
): { sessionId: string; expiresAt: Date };

type SessionResult = ReturnOf<typeof createSession>;
// = { sessionId: string; expiresAt: Date }
type SessionParams = ParamsOf<typeof createSession>;
// = [userId: number, authToken: string, ttlSeconds?: number]   ← a labelled tuple

// ── Multiple infers, and reusing one name ───────────────────────────────────
type SplitRoute<T> = T extends `${infer Method} ${infer Path}`
  ? { method: Method; path: Path }
  : never;
type Route = SplitRoute<"POST /api/users/:userId">;
// = { method: "POST"; path: "/api/users/:userId" }

// Same name inferred twice in a COVARIANT position → the compiler unions them:
type BothElements<T> = T extends [infer X, infer X] ? X : never;
type BE = BothElements<[string, number]>;            // string | number

// ── infer on object properties ──────────────────────────────────────────────
type PayloadOf<T> = T extends { payload: infer P } ? P : never;
type Pay = PayloadOf<{ type: "user.created"; payload: { userId: number } }>;
// = { userId: number }

// ── Constrained infer (TS 4.7+) — filter while matching ─────────────────────
type StatusCodeOf<T> = T extends { status: infer S extends number } ? S : never;
type SC1 = StatusCodeOf<{ status: 404 }>;            // 404
type SC2 = StatusCodeOf<{ status: "404" }>;          // never  ← constraint failed

// Without the constraint you'd need a nested conditional:
type StatusCodeOld<T> = T extends { status: infer S }
  ? S extends number ? S : never
  : never;

interface UserRecord { userId: number; email: string }
```

The mental model: `infer` is a hole in a shape. You describe the shape you expect, punch a hole where you want the value, and the compiler fills it in — or the whole match fails and you take the false branch.

### Concept 3 — Distributive conditional types

This is the behaviour that surprises everyone once and then becomes indispensable.

**Rule:** if the checked type is a *naked type parameter* (just `T`, not `T[]`, not `[T]`, not `{ v: T }`) and the type argument is a **union**, the conditional is applied to each member separately and the results are unioned back together.

```ts
type ToArray<T> = T extends unknown ? T[] : never;

type A1 = ToArray<string | number>;
// NOT (string | number)[]
// Instead: ToArray<string> | ToArray<number>  =  string[] | number[]

// Step by step, this is what the compiler does:
//   1. T is naked, argument is a union of 2 members → distribute
//   2. member "string":  string extends unknown ? string[] : never  →  string[]
//   3. member "number":  number extends unknown ? number[] : never  →  number[]
//   4. union the results: string[] | number[]
```

Distribution makes conditional types behave like `filter` and `map` over unions:

```ts
type EventName =
  | "user.created"
  | "user.deleted"
  | "payment.failed"
  | "payment.settled"
  | "order.placed";

// ── FILTER: keep members matching a pattern (this is `Extract`) ─────────────
type Keep<T, U> = T extends U ? T : never;
type PaymentEvents = Keep<EventName, `payment.${string}`>;
// member by member:
//   "user.created"    extends `payment.${string}` ? ... : never  → never
//   "user.deleted"    → never
//   "payment.failed"  → "payment.failed"
//   "payment.settled" → "payment.settled"
//   "order.placed"    → never
// union: never | never | "payment.failed" | "payment.settled" | never
//      = "payment.failed" | "payment.settled"     ← never vanishes from unions

// ── FILTER OUT (this is `Exclude`) ──────────────────────────────────────────
type Drop<T, U> = T extends U ? never : T;
type NonPaymentEvents = Drop<EventName, `payment.${string}`>;
// = "user.created" | "user.deleted" | "order.placed"

// ── MAP: transform each member ──────────────────────────────────────────────
type Namespaced<T extends string> = T extends `${infer Domain}.${infer Action}`
  ? { domain: Domain; action: Action }
  : never;
type Parsed = Namespaced<"user.created" | "payment.failed">;
// = { domain: "user"; action: "created" } | { domain: "payment"; action: "failed" }

// ── Distribution over a discriminated union of objects ──────────────────────
type ApiEvent =
  | { type: "user.created";   payload: { userId: number; email: string } }
  | { type: "user.deleted";   payload: { userId: number; deletedBy: number } }
  | { type: "payment.failed"; payload: { paymentId: string; reason: string } };

// Look up ONE member's payload by its tag — the workhorse of typed event buses:
type PayloadFor<Type extends ApiEvent["type"]> =
  Extract<ApiEvent, { type: Type }>["payload"];

type P = PayloadFor<"payment.failed">;   // { paymentId: string; reason: string }

// ── The three critical facts about distribution ─────────────────────────────

// (a) `never` is the EMPTY union — distributing over it produces never:
type Distributed<T> = T extends string ? "matched" : "unmatched";
type N1 = Distributed<never>;      // never   ← NOT "unmatched"! Zero iterations.

// (b) Only NAKED type parameters distribute. These do not:
type NotNaked1<T> = T[] extends unknown[] ? true : false;
type NotNaked2<T> = [T] extends [unknown] ? true : false;
type NotNaked3<T> = { v: T } extends { v: unknown } ? true : false;

// (c) A concrete union written inline (not via a parameter) does NOT distribute:
type Inline = (string | number) extends string ? true : false;   // false — checked as a whole
```

`boolean` deserves its own warning, because it is secretly `true | false`:

```ts
type Flip<T> = T extends true ? "on" : "off";
type F1 = Flip<boolean>;
// boolean = true | false, so it distributes:
//   true  → "on"
//   false → "off"
// = "on" | "off"     ← not "off" as you might expect

// Block it if you meant "is this type exactly boolean":
type FlipWhole<T> = [T] extends [true] ? "on" : "off";
type F2 = FlipWhole<boolean>;    // "off"
type F3 = FlipWhole<true>;       // "on"
```

### Concept 4 — Blocking distribution with `[T] extends [U]`

Wrapping both sides in a one-element tuple makes the checked type non-naked, so the conditional evaluates once against the whole union. This is the standard idiom and you will see it constantly in library code.

```ts
// ── Problem: "does T contain a string?" ─────────────────────────────────────
type ContainsStringBad<T> = T extends string ? true : false;
type C1 = ContainsStringBad<string | number>;   // boolean  ← true | false, useless

type ContainsStringGood<T> = [T] extends [string] ? true : false;
type C2 = ContainsStringGood<string | number>;  // false  ← the whole union isn't string
type C3 = ContainsStringGood<string>;           // true

// ── The canonical use: detecting `never` ────────────────────────────────────
type IsNeverBad<T> = T extends never ? true : false;
type IN1 = IsNeverBad<never>;      // never   ← distribution over zero members

type IsNever<T> = [T] extends [never] ? true : false;
type IN2 = IsNever<never>;         // true    ✅
type IN3 = IsNever<string>;        // false

// This matters because `never` is what a failed lookup produces, and you often
// want to turn that into a friendly error type instead of silently propagating.
type LookupOrFail<T, K> = [Extract<T, { type: K }>] extends [never]
  ? { error: "no such event type"; requested: K }
  : Extract<T, { type: K }>;

// ── Detecting `any` ─────────────────────────────────────────────────────────
// Trick: only `any` makes `0 extends (1 & T)` true, because `1 & any` is `any`.
type IsAny<T> = 0 extends 1 & T ? true : false;
type IA1 = IsAny<any>;        // true
type IA2 = IsAny<unknown>;    // false
type IA3 = IsAny<string>;     // false

// ── Detecting `unknown` ─────────────────────────────────────────────────────
type IsUnknown<T> = IsAny<T> extends true ? false : unknown extends T ? true : false;
type IU1 = IsUnknown<unknown>;   // true
type IU2 = IsUnknown<any>;       // false
type IU3 = IsUnknown<string>;    // false

// ── Mutual assignability = type equality (approximately) ────────────────────
type IsExactly<A, B> = [A] extends [B] ? ([B] extends [A] ? true : false) : false;
type EQ1 = IsExactly<{ userId: number }, { userId: number }>;   // true
type EQ2 = IsExactly<string, string | never>;                    // true
type EQ3 = IsExactly<string, string | number>;                   // false
// (A stricter version used in test libraries exploits conditional-type
//  identity checking; see "Going deeper".)

// ── Union emptiness / membership helpers built on the same idiom ────────────
type IsEmptyUnion<T> = [T] extends [never] ? true : false;
type HasMember<T, M> = [Extract<T, M>] extends [never] ? false : true;

type HM1 = HasMember<"GET" | "POST", "POST">;    // true
type HM2 = HasMember<"GET" | "POST", "PATCH">;   // false
```

### Concept 5 — Hand-building the standard library

Every conditional utility type in `lib.es5.d.ts` fits on one screen. Writing them yourself is the fastest way to internalise the feature.

```ts
// ══════════════════════════════════════════════════════════════════════════
// Exclude / Extract / NonNullable — pure distribution, no infer
// ══════════════════════════════════════════════════════════════════════════

type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T & {};     // the real lib uses this since TS 4.8
type MyNonNullableClassic<T> = T extends null | undefined ? never : T;

type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
type ReadMethods  = MyExtract<HttpMethod, "GET">;                     // "GET"
type WriteMethods = MyExclude<HttpMethod, "GET">;                     // "POST"|"PUT"|"PATCH"|"DELETE"
type Cleaned      = MyNonNullableClassic<string | null | undefined>;  // string

// ══════════════════════════════════════════════════════════════════════════
// ReturnType / Parameters / ConstructorParameters / InstanceType — infer
// ══════════════════════════════════════════════════════════════════════════

type MyReturnType<T extends (...args: never[]) => unknown> =
  T extends (...args: never[]) => infer R ? R : never;

type MyParameters<T extends (...args: never[]) => unknown> =
  T extends (...args: infer P) => unknown ? P : never;

type MyConstructorParameters<T extends abstract new (...args: never[]) => unknown> =
  T extends abstract new (...args: infer P) => unknown ? P : never;

type MyInstanceType<T extends abstract new (...args: never[]) => unknown> =
  T extends abstract new (...args: never[]) => infer I ? I : never;

// (The real lib uses `any[]` in the constraint; `never[]` is stricter and
//  works for reading types, but rejects some generic functions. Use `any[]`
//  if you hit "not assignable to the constraint" errors on generic callbacks.)

declare function issueAuthToken(
  userId: number,
  scopes: string[],
): { authToken: string; expiresAt: Date };

type Issued = MyReturnType<typeof issueAuthToken>;
// = { authToken: string; expiresAt: Date }
type IssueArgs = MyParameters<typeof issueAuthToken>;
// = [userId: number, scopes: string[]]

class RedisCacheClient {
  constructor(public readonly url: string, public readonly ttlSeconds: number) {}
  get(key: string): Promise<string | null> { return Promise.resolve(null); }
}
type CacheCtorArgs = MyConstructorParameters<typeof RedisCacheClient>;  // [url: string, ttlSeconds: number]
type Cache         = MyInstanceType<typeof RedisCacheClient>;           // RedisCacheClient

// ══════════════════════════════════════════════════════════════════════════
// Awaited — RECURSIVE, and handles thenables, not just Promise
// ══════════════════════════════════════════════════════════════════════════

// The naive version, which is what most people write first:
type NaiveAwaited<T> = T extends Promise<infer V> ? NaiveAwaited<V> : T;

type NA1 = NaiveAwaited<Promise<Promise<string>>>;   // string
type NA2 = NaiveAwaited<string>;                     // string

// The real one also handles any object with a `.then(onfulfilled)` method,
// because `await` unwraps thenables, not just real Promises:
type MyAwaited<T> =
  T extends null | undefined ? T
  : T extends object & { then(onfulfilled: infer F, ...args: never): unknown }
    ? F extends (value: infer V, ...args: never) => unknown
      ? MyAwaited<V>
      : never
    : T;

type A1 = MyAwaited<Promise<Promise<number[]>>>;                 // number[]
type A2 = MyAwaited<{ then(cb: (v: string) => void): void }>;    // string
type A3 = MyAwaited<number>;                                      // number

// ══════════════════════════════════════════════════════════════════════════
// The combination you will actually use every day
// ══════════════════════════════════════════════════════════════════════════

async function fetchUserProfile(userId: number) {
  return {
    userId,
    email:       "ada@example.com",
    displayName: "Ada",
    roles:       ["admin", "billing"] as const,
    lastLoginAt: new Date(),
  };
}

type UserProfile = Awaited<ReturnType<typeof fetchUserProfile>>;
// = { userId: number; email: string; displayName: string;
//     roles: readonly ["admin", "billing"]; lastLoginAt: Date }

// Now every downstream type is derived, not copied:
type ProfileField = keyof UserProfile;                    // "userId" | "email" | ...
type Role         = UserProfile["roles"][number];         // "admin" | "billing"
type ProfilePatch = Partial<Omit<UserProfile, "userId">>;
```

### Concept 6 — Nested conditionals: type-level `switch`

Chained conditionals are how you write a decision table. The formatting convention below (leading `:` in a column) makes them readable.

```ts
// ── A type-level typeof ─────────────────────────────────────────────────────
type TypeName<T> =
  T extends string      ? "string"
  : T extends number    ? "number"
  : T extends boolean   ? "boolean"
  : T extends undefined ? "undefined"
  : T extends null      ? "null"
  : T extends Date      ? "date"
  : T extends readonly unknown[] ? "array"
  : T extends Function  ? "function"
  : "object";

type TN1 = TypeName<string>;       // "string"
type TN2 = TypeName<Date>;         // "date"
type TN3 = TypeName<string[]>;     // "array"
type TN4 = TypeName<{ a: 1 }>;     // "object"
type TN5 = TypeName<string | number>;   // "string" | "number"  ← distributed!

// ⚠️ ORDER MATTERS — put narrower checks first. `Date extends object` is true,
// so if `T extends object` came before `T extends Date`, dates would be "object".

// ── Column-mapping: SQL column type from a TS type ──────────────────────────
type SqlColumnType<T> =
  [T] extends [never]      ? never
  : T extends boolean      ? "BOOLEAN"
  : T extends number       ? "INTEGER"
  : T extends bigint       ? "BIGINT"
  : T extends Date         ? "TIMESTAMPTZ"
  : T extends string       ? "TEXT"
  : T extends readonly unknown[] ? "JSONB"
  : T extends object       ? "JSONB"
  : "TEXT";

interface OrderTable {
  orderId:    string;
  userId:     number;
  totalCents: number;
  isPaid:     boolean;
  placedAt:   Date;
  metadata:   Record<string, unknown>;
}

// Feed it through a mapped type (doc 43) to describe the whole schema:
type DdlOf<T> = { [K in keyof T]: SqlColumnType<NonNullable<T[K]>> };
type OrderDdl = DdlOf<OrderTable>;
// = { orderId: "TEXT"; userId: "INTEGER"; totalCents: "INTEGER";
//     isPaid: "BOOLEAN"; placedAt: "TIMESTAMPTZ"; metadata: "JSONB" }

// ── Operator sets per column type (a real query-builder need) ───────────────
type Comparators<V> =
  V extends number | bigint | Date
    ? { eq?: V; neq?: V; gt?: V; gte?: V; lt?: V; lte?: V; in?: V[] }
  : V extends string
    ? { eq?: V; neq?: V; like?: string; startsWith?: string; in?: V[] }
  : V extends boolean
    ? { eq?: V }
  : { eq?: V; neq?: V };

type NumberOps = Comparators<number>;   // has gt/gte/lt/lte
type TextOps   = Comparators<string>;   // has like/startsWith, no gt
type BoolOps   = Comparators<boolean>;  // ⚠️ boolean distributes → Comparators<true>|Comparators<false>
                                         //    = { eq?: true } | { eq?: false }
// Fix with the tuple trick if you want a single { eq?: boolean }:
type ComparatorsFixed<V> =
  [V] extends [boolean] ? { eq?: boolean }
  : [V] extends [number | bigint | Date] ? { eq?: V; gt?: V; gte?: V; lt?: V; lte?: V; in?: V[] }
  : [V] extends [string] ? { eq?: V; like?: string; in?: V[] }
  : { eq?: V };
```

### Concept 7 — Conditional types in generic function return positions

A conditional type in a return position lets one function have a return type that *follows from* its arguments. This is what replaces overloads in modern code — but it comes with a real limitation you must understand.

```ts
// ── The basic pattern: return type keyed off a literal argument ─────────────

type ResponseFor<Format extends "json" | "text" | "buffer"> =
  Format extends "json"   ? Record<string, unknown>
  : Format extends "text" ? string
  : Buffer;

declare function readResponseBody<Format extends "json" | "text" | "buffer">(
  response: Response,
  format: Format,
): Promise<ResponseFor<Format>>;

declare const response: Response;
const asJson = await readResponseBody(response, "json");     // Record<string, unknown>
const asText = await readResponseBody(response, "text");     // string
const asBuf  = await readResponseBody(response, "buffer");   // Buffer

// ── Optional-vs-required results ────────────────────────────────────────────

interface UserEntity { userId: number; email: string; isAdmin: boolean }

type QueryResult<Mode extends "one" | "many" | "oneOrFail"> =
  Mode extends "many"      ? UserEntity[]
  : Mode extends "one"     ? UserEntity | null
  : UserEntity;   // "oneOrFail" throws instead of returning null

declare function queryUsers<Mode extends "one" | "many" | "oneOrFail" = "many">(
  where: Partial<UserEntity>,
  mode?: Mode,
): Promise<QueryResult<Mode>>;

const list  = await queryUsers({ isAdmin: true });                    // UserEntity[]
const maybe = await queryUsers({ userId: 1 }, "one");                 // UserEntity | null
const sure  = await queryUsers({ userId: 1 }, "oneOrFail");           // UserEntity
sure.email;      // ✅ no null check needed
// maybe.email;  // ❌ Error: possibly null — exactly the pressure you want

// ── THE LIMITATION: inside the function body, the conditional is UNRESOLVED ──

function readBodyImpl<Format extends "json" | "text">(
  raw: string,
  format: Format,
): ResponseFor<Format> {
  if (format === "json") {
    // return JSON.parse(raw);
    // ❌ Error: Type 'any' is not assignable to type 'ResponseFor<Format>'
    //    The compiler cannot prove that narrowing `format` narrows `Format`.
  }
  // ...
  throw new Error("unreachable");
}

// ✅ Fix A — assert once at the return, keeping the public signature honest:
function readBodyA<Format extends "json" | "text">(
  raw: string,
  format: Format,
): ResponseFor<Format> {
  const value = format === "json" ? (JSON.parse(raw) as unknown) : raw;
  return value as ResponseFor<Format>;   // one contained lie, at the boundary
}

// ✅ Fix B — overload signatures for callers, a loose implementation signature:
function readBodyB(raw: string, format: "json"): Record<string, unknown>;
function readBodyB(raw: string, format: "text"): string;
function readBodyB(raw: string, format: "json" | "text"): unknown {
  return format === "json" ? JSON.parse(raw) : raw;
}
const j = readBodyB("{}", "json");   // Record<string, unknown>
const t = readBodyB("hi", "text");   // string

// Rule of thumb: conditional return types are excellent for CALLERS and
// awkward for IMPLEMENTERS. Use them on declared APIs and wrappers; use
// overloads or a single internal cast where the logic actually lives.

// ── Deferred conditionals: when the argument is still generic ───────────────

type Boxed<T> = T extends string ? { text: T } : { value: T };

function box<T>(input: T): Boxed<T> {
  // `Boxed<T>` is DEFERRED here — T is unresolved, so no branch is taken.
  return (typeof input === "string" ? { text: input } : { value: input }) as Boxed<T>;
}

const b1 = box("req_123");    // { text: string }   ← resolved at the call site
const b2 = box(42);           // { value: number }

declare const Buffer: { new (): unknown; prototype: unknown };
type Buffer = unknown;
declare const Response: { new (): unknown };
type Response = unknown;
```

### Concept 8 — Recursive conditional types

Since TypeScript 4.1 a conditional type may reference itself in its branches. This is how you unwrap arbitrary nesting, walk tuples, and parse strings at the type level.

```ts
// ── Unwrap nested promises to any depth ─────────────────────────────────────
type DeepAwaited<T> = T extends Promise<infer Inner> ? DeepAwaited<Inner> : T;
type DA = DeepAwaited<Promise<Promise<Promise<{ userId: number }>>>>;
// = { userId: number }

// ── Flatten nested arrays ───────────────────────────────────────────────────
type DeepFlat<T> = T extends readonly (infer Element)[] ? DeepFlat<Element> : T;
type DF = DeepFlat<number[][][]>;   // number

// ── Walk a tuple ────────────────────────────────────────────────────────────
type Reverse<T extends readonly unknown[]> =
  T extends readonly [infer Head, ...infer Tail] ? [...Reverse<Tail>, Head] : [];
type RV = Reverse<[1, 2, 3]>;   // [3, 2, 1]

type LastOf<T extends readonly unknown[]> =
  T extends readonly [...unknown[], infer Last] ? Last : never;
type LO = LastOf<[string, number, boolean]>;   // boolean

// ── Parse a route path into its params (this is how tRPC-ish routers work) ──
type PathParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | PathParams<`/${Rest}`>
  : Path extends `${string}:${infer Param}`
    ? Param
    : never;

type P1 = PathParams<"/api/users/:userId/orders/:orderId">;   // "userId" | "orderId"
type P2 = PathParams<"/api/health">;                          // never

// Turn those into a typed params object:
type ParamsObject<Path extends string> =
  [PathParams<Path>] extends [never]
    ? Record<string, never>
    : { [K in PathParams<Path>]: string };

type PO = ParamsObject<"/api/users/:userId/orders/:orderId">;
// = { userId: string; orderId: string }

// ── Dotted path access into a nested config ─────────────────────────────────
type ValueAtPath<T, Path extends string> =
  Path extends `${infer Head}.${infer Rest}`
    ? Head extends keyof T ? ValueAtPath<T[Head], Rest> : never
    : Path extends keyof T ? T[Path] : never;

interface AppConfig {
  server:   { port: number; host: string; tls: { certPath: string } };
  database: { url: string; poolSize: number };
}

type V1 = ValueAtPath<AppConfig, "server.port">;          // number
type V2 = ValueAtPath<AppConfig, "server.tls.certPath">;  // string
type V3 = ValueAtPath<AppConfig, "server.prot">;          // never

declare function getConfig<Path extends string>(path: Path): ValueAtPath<AppConfig, Path>;
const port = getConfig("server.port");                    // number
const cert = getConfig("server.tls.certPath");            // string

// ── Recursion has a limit ───────────────────────────────────────────────────
// Around 50 levels of instantiation depth (1000 for tail-recursive forms since
// TS 4.5) the compiler bails with:
//   "Type instantiation is excessively deep and possibly infinite. ts(2589)"
// Tail-recursive shapes — where the recursive call is the ENTIRE branch, as in
// DeepAwaited above — get the higher limit. Accumulator-style tuple recursion
// like Reverse<> also qualifies.
```

---

## Example 1 — basic

```ts
// Building the six utility types you use most, from nothing, and using them.

// ════════════════════════════════════════════════════════════════════════════
// The building blocks
// ════════════════════════════════════════════════════════════════════════════

type Unwrap<T>       = T extends Promise<infer V> ? Unwrap<V> : T;
type ElementOf<T>    = T extends readonly (infer E)[] ? E : never;
type ReturnOf<T>     = T extends (...args: never[]) => infer R ? R : never;
type ParamsOf<T>     = T extends (...args: infer P) => unknown ? P : never;
type Only<T, U>      = T extends U ? T : never;      // = Extract
type Without<T, U>   = T extends U ? never : T;      // = Exclude
type Defined<T>      = T extends null | undefined ? never : T;   // = NonNullable

// ════════════════════════════════════════════════════════════════════════════
// A tiny service module — the "source of truth"
// ════════════════════════════════════════════════════════════════════════════

interface UserEntity {
  userId:       number;
  email:        string;
  displayName:  string;
  passwordHash: string;
  isAdmin:      boolean;
  createdAt:    Date;
  deletedAt:    Date | null;
}

async function findUserById(userId: number): Promise<UserEntity | null> {
  return db.users.find(userId);
}

async function listActiveUsers(limit = 50): Promise<UserEntity[]> {
  return db.users.list({ deletedAt: null, limit });
}

function toPublicUser(user: UserEntity) {
  return {
    userId:      user.userId,
    email:       user.email,
    displayName: user.displayName,
    isAdmin:     user.isAdmin,
    createdAt:   user.createdAt.toISOString(),
  };
}

// ════════════════════════════════════════════════════════════════════════════
// Everything downstream is DERIVED — no shape is written twice
// ════════════════════════════════════════════════════════════════════════════

// 1. What findUserById resolves to, minus the null:
type FoundUser = Defined<Unwrap<ReturnOf<typeof findUserById>>>;
// = UserEntity

// 2. One element of the list endpoint's result:
type ListedUser = ElementOf<Unwrap<ReturnOf<typeof listActiveUsers>>>;
// = UserEntity

// 3. The exact JSON the API sends — inferred from the serialiser, never copied:
type PublicUser = ReturnOf<typeof toPublicUser>;
// = { userId: number; email: string; displayName: string;
//     isAdmin: boolean; createdAt: string }

// 4. The arguments of the list endpoint, for a retry wrapper:
type ListArgs = ParamsOf<typeof listActiveUsers>;
// = [limit?: number | undefined]

// 5. Only the Date-valued fields of the entity, as a union of key names:
type FieldsWithDates = {
  [K in keyof UserEntity]-?: Defined<UserEntity[K]> extends Date ? K : never;
}[keyof UserEntity];
// = "createdAt" | "deletedAt"

// 6. Which fields must never be serialised:
type SecretField = Only<keyof UserEntity, "passwordHash" | `${string}Secret`>;
// = "passwordHash"
type SafeField   = Without<keyof UserEntity, SecretField>;
// = "userId" | "email" | "displayName" | "isAdmin" | "createdAt" | "deletedAt"

// ════════════════════════════════════════════════════════════════════════════
// Using the derived types — the compiler now enforces the relationships
// ════════════════════════════════════════════════════════════════════════════

// A generic retry wrapper whose types follow the wrapped function exactly:
async function withRetry<Fn extends (...args: never[]) => Promise<unknown>>(
  fn: Fn,
  args: ParamsOf<Fn>,
  attempts = 3,
): Promise<Unwrap<ReturnOf<Fn>>> {
  let lastError: unknown;
  for (let attempt = 1; attempt <= attempts; attempt++) {
    try {
      return (await fn(...(args as never[]))) as Unwrap<ReturnOf<Fn>>;
    } catch (error) {
      lastError = error;
      await delay(2 ** attempt * 100);
    }
  }
  throw lastError;
}

const users = await withRetry(listActiveUsers, [100]);
// users: UserEntity[]   ← inferred all the way through
// await withRetry(listActiveUsers, ["100"]);   // ❌ Error: string is not number

// A serialiser that is guaranteed not to touch a secret field:
function pickSafe<K extends SafeField>(user: UserEntity, fields: K[]): Pick<UserEntity, K> {
  const output = {} as Pick<UserEntity, K>;
  for (const field of fields) output[field] = user[field];
  return output;
}

const summary = pickSafe(userRecord, ["userId", "email", "isAdmin"]);
// summary: { userId: number; email: string; isAdmin: boolean }
// pickSafe(userRecord, ["passwordHash"]);   // ❌ Error: not assignable to SafeField

// Rename `displayName` to `name` in UserEntity, and PublicUser, FoundUser,
// ListedUser, SafeField and every call site above break in the same compile.

declare const db: {
  users: {
    find(userId: number): Promise<UserEntity | null>;
    list(options: { deletedAt: null; limit: number }): Promise<UserEntity[]>;
  };
};
declare const userRecord: UserEntity;
declare function delay(ms: number): Promise<void>;
```

---

## Example 2 — real world backend use case

```ts
// A fully typed HTTP router + job queue, where every type is COMPUTED from a
// single route table. Conditional types do all the work.

// ════════════════════════════════════════════════════════════════════════════
// PART 1 — The route table: one declaration, everything derived
// ════════════════════════════════════════════════════════════════════════════

interface UserResource   { userId: number; email: string; displayName: string; isAdmin: boolean }
interface OrderResource  { orderId: string; userId: number; totalCents: number; status: OrderStatus }
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

interface ApiError { error: string; code: number }

// The single source of truth. Keys are "METHOD /path" with :params inline.
interface RouteTable {
  "GET /api/users":                   { body: never;              query: { limit?: number; cursor?: string }; response: { items: UserResource[]; nextCursor: string | null } };
  "GET /api/users/:userId":           { body: never;              query: never;                                response: UserResource };
  "POST /api/users":                  { body: CreateUserBody;     query: never;                                response: UserResource };
  "PATCH /api/users/:userId":         { body: PatchUserBody;      query: never;                                response: UserResource };
  "DELETE /api/users/:userId":        { body: never;              query: never;                                response: { deleted: true } };
  "GET /api/users/:userId/orders":    { body: never;              query: { status?: OrderStatus };              response: OrderResource[] };
  "POST /api/orders/:orderId/refund": { body: { reasonCode: string }; query: never;                             response: OrderResource };
}

interface CreateUserBody { email: string; displayName: string; password: string; isAdmin?: boolean }
interface PatchUserBody  { email?: string; displayName?: string; isAdmin?: boolean }

// ── Derive: split "METHOD /path" into its parts ─────────────────────────────
type RouteKey = keyof RouteTable;

type MethodOf<K extends string> = K extends `${infer M} ${string}` ? M : never;
type PathOf<K extends string>   = K extends `${string} ${infer P}` ? P : never;

type M1 = MethodOf<"POST /api/users">;    // "POST"
type P1 = PathOf<"POST /api/users">;      // "/api/users"

// ── Derive: which routes use which verb (distribution as a filter) ──────────
type RoutesWithMethod<M extends string> = Extract<RouteKey, `${M} ${string}`>;
type MutatingRoutes = RoutesWithMethod<"POST" | "PATCH" | "DELETE">;
// = "POST /api/users" | "PATCH /api/users/:userId"
//   | "DELETE /api/users/:userId" | "POST /api/orders/:orderId/refund"

// ── Derive: path params, recursively, from the path string ──────────────────
type ExtractParams<Path extends string> =
  Path extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParams<`/${Rest}`>
  : Path extends `${string}:${infer Param}`
    ? Param
    : never;

// Path params are strings on the wire; coerce known numeric ones:
type ParamValue<Name extends string> = Name extends `${string}Id`
  ? Name extends "userId" ? number : string
  : string;

type ParamsOfRoute<K extends RouteKey> =
  [ExtractParams<PathOf<K>>] extends [never]
    ? Record<string, never>
    : { [Name in ExtractParams<PathOf<K>>]: ParamValue<Name> };

type PA1 = ParamsOfRoute<"GET /api/users/:userId">;
// = { userId: number }
type PA2 = ParamsOfRoute<"POST /api/orders/:orderId/refund">;
// = { orderId: string }
type PA3 = ParamsOfRoute<"GET /api/users">;
// = Record<string, never>   ← no params

// ── Derive: the handler's context object, per route ─────────────────────────
// `never` body/query fields are omitted entirely rather than being unusable.
type OmitNever<T> = { [K in keyof T as [T[K]] extends [never] ? never : K]: T[K] };

type HandlerContext<K extends RouteKey> = OmitNever<{
  params:  ParamsOfRoute<K>;
  body:    RouteTable[K]["body"];
  query:   RouteTable[K]["query"];
}> & {
  userId:    number;
  authToken: string;
  requestId: string;
};

type HC1 = HandlerContext<"POST /api/users">;
// = { params: Record<string, never>; body: CreateUserBody;
//     userId: number; authToken: string; requestId: string }
//   ← `query` is gone because it was never

type HC2 = HandlerContext<"GET /api/users/:userId">;
// = { params: { userId: number }; userId: number; authToken: string; requestId: string }

// ── Derive: the handler signature and the whole handler map ─────────────────
type RouteHandler<K extends RouteKey> = (
  context: HandlerContext<K>,
) => Promise<RouteTable[K]["response"]>;

type HandlerMap = { [K in RouteKey]: RouteHandler<K> };

// ════════════════════════════════════════════════════════════════════════════
// PART 2 — Implementing it. Every handler is checked against the table.
// ════════════════════════════════════════════════════════════════════════════

const handlers: HandlerMap = {
  "GET /api/users": async (ctx) => {
    // ctx.query is { limit?: number; cursor?: string }; ctx.body does not exist.
    const items = await userRepo.list(ctx.query.limit ?? 25, ctx.query.cursor);
    return { items, nextCursor: items.length ? String(items[items.length - 1]!.userId) : null };
  },

  "GET /api/users/:userId": async (ctx) => {
    // ctx.params.userId is a NUMBER, not a string — ParamValue said so.
    const user = await userRepo.findById(ctx.params.userId);
    if (!user) throw new HttpError(404, "user not found");
    return user;
  },

  "POST /api/users": async (ctx) => {
    // ctx.body is CreateUserBody — password is required, isAdmin optional.
    return userRepo.create(ctx.body);
  },

  "PATCH /api/users/:userId": async (ctx) => {
    return userRepo.update(ctx.params.userId, ctx.body);
  },

  "DELETE /api/users/:userId": async (ctx) => {
    await userRepo.softDelete(ctx.params.userId, ctx.userId);
    return { deleted: true };   // ❌ `{ deleted: false }` would not compile
  },

  "GET /api/users/:userId/orders": async (ctx) => {
    return orderRepo.listForUser(ctx.params.userId, ctx.query.status);
  },

  "POST /api/orders/:orderId/refund": async (ctx) => {
    return orderRepo.refund(ctx.params.orderId, ctx.body.reasonCode, ctx.userId);
  },

  // Add a route to RouteTable and this object literal fails to compile until
  // you implement it. Remove one and its handler becomes an excess property.
};

// ════════════════════════════════════════════════════════════════════════════
// PART 3 — A type-safe client, derived from the SAME table
// ════════════════════════════════════════════════════════════════════════════

// Build the argument object the caller must supply, omitting empty pieces:
type RequestArgs<K extends RouteKey> = OmitNever<{
  params: ParamsOfRoute<K> extends Record<string, never> ? never : ParamsOfRoute<K>;
  body:   RouteTable[K]["body"];
  query:  RouteTable[K]["query"];
}>;

// If a route needs nothing at all, allow calling with no second argument:
type CallArgs<K extends RouteKey> =
  keyof RequestArgs<K> extends never ? [] : [args: RequestArgs<K>];

declare function apiCall<K extends RouteKey>(
  route: K,
  ...args: CallArgs<K>
): Promise<RouteTable[K]["response"]>;

// ── Usage: everything inferred, everything checked ──────────────────────────

const page = await apiCall("GET /api/users", { query: { limit: 10 } });
// page: { items: UserResource[]; nextCursor: string | null }

const one = await apiCall("GET /api/users/:userId", { params: { userId: 42 } });
// one: UserResource
one.email;   // ✅

const created = await apiCall("POST /api/users", {
  body: { email: "ada@example.com", displayName: "Ada", password: "s3cret" },
});
// created: UserResource

// ❌ await apiCall("GET /api/users/:userId", { params: { userId: "42" } });
//    Error: string is not assignable to number
// ❌ await apiCall("POST /api/users", { body: { email: "a@b.c" } });
//    Error: missing displayName and password
// ❌ await apiCall("GET /api/uesrs");
//    Error: not assignable to RouteKey — typo caught at the call site
// ❌ (await apiCall("DELETE /api/users/:userId", { params: { userId: 1 } })).deleted === false;
//    Error: this comparison appears unintentional — the type is `true`

// ════════════════════════════════════════════════════════════════════════════
// PART 4 — A job queue whose payloads are derived from the handlers themselves
// ════════════════════════════════════════════════════════════════════════════

// Job processors, written as ordinary functions:
const jobProcessors = {
  sendWelcomeEmail: async (payload: { userId: number; email: string }) => {
    await mailer.send(payload.email, "welcome");
    return { messageId: "msg_1" };
  },
  reconcilePayment: async (payload: { paymentId: string; attempt: number }) => {
    return { settled: true as const, attempt: payload.attempt };
  },
  rebuildSearchIndex: async (payload: { since: Date; batchSize?: number }) => {
    return { indexed: 0 };
  },
};

type JobName = keyof typeof jobProcessors;

// Derive the payload type of each job from its processor's first parameter:
type PayloadOf<Name extends JobName> =
  (typeof jobProcessors)[Name] extends (payload: infer P) => unknown ? P : never;

type WelcomePayload = PayloadOf<"sendWelcomeEmail">;   // { userId: number; email: string }

// Derive the result type the same way:
type JobResult<Name extends JobName> = Awaited<ReturnType<(typeof jobProcessors)[Name]>>;
type WelcomeResult = JobResult<"sendWelcomeEmail">;    // { messageId: string }

// Jobs whose payload has a required `userId` — used to attribute queue metrics:
type UserScopedJob = {
  [Name in JobName]: PayloadOf<Name> extends { userId: number } ? Name : never;
}[JobName];
// = "sendWelcomeEmail"

// The enqueue API — payload type follows the job name:
declare function enqueue<Name extends JobName>(
  name: Name,
  payload: PayloadOf<Name>,
  options?: { delayMs?: number; attempts?: number },
): Promise<{ jobId: string }>;

await enqueue("sendWelcomeEmail", { userId: 42, email: "ada@example.com" });
await enqueue("rebuildSearchIndex", { since: new Date(), batchSize: 500 });
// ❌ await enqueue("sendWelcomeEmail", { userId: 42 });
//    Error: missing email
// ❌ await enqueue("sendWelcomeEmial", { userId: 42, email: "x" });
//    Error: unknown job name

// Run a job by name with a fully typed result:
async function runJob<Name extends JobName>(
  name: Name,
  payload: PayloadOf<Name>,
): Promise<JobResult<Name>> {
  const processor = jobProcessors[name] as (p: PayloadOf<Name>) => Promise<JobResult<Name>>;
  return processor(payload);
}

const result = await runJob("reconcilePayment", { paymentId: "pay_1", attempt: 2 });
// result: { settled: true; attempt: number }

// ── Supporting declarations ─────────────────────────────────────────────────
class HttpError extends Error {
  constructor(public readonly status: number, message: string) { super(message); }
}
declare const userRepo: {
  list(limit: number, cursor?: string): Promise<UserResource[]>;
  findById(userId: number): Promise<UserResource | null>;
  create(body: CreateUserBody): Promise<UserResource>;
  update(userId: number, patch: PatchUserBody): Promise<UserResource>;
  softDelete(userId: number, byUserId: number): Promise<void>;
};
declare const orderRepo: {
  listForUser(userId: number, status?: OrderStatus): Promise<OrderResource[]>;
  refund(orderId: string, reasonCode: string, byUserId: number): Promise<OrderResource>;
};
declare const mailer: { send(to: string, template: string): Promise<void> };
```

---

## Going deeper

### `never` in a union disappears — and that is the whole trick

`Exclude`, `Extract`, key filtering, and `OmitNever` all rest on one algebraic fact: `X | never` is `X`. `never` is the empty union, and unioning with it adds nothing.

```ts
type Demo = string | never | number | never;   // string | number

// Which is why "map to never to delete" works everywhere:
type ActiveOnly<T> = T extends { deletedAt: Date } ? never : T;
type Rows =
  | { userId: number; deletedAt: Date }
  | { userId: number; deletedAt: null };
type Active = ActiveOnly<Rows>;   // { userId: number; deletedAt: null }

// ⚠️ But `never` in an INTERSECTION or a property position does NOT disappear —
// it poisons:
type Poisoned = { userId: number; email: never };
// const p: Poisoned = { userId: 1, email: "x" };   // ❌ nothing is assignable to never
// You literally cannot construct this object. That is why OmitNever exists.
```

### Conditional types are *deferred* while a type parameter is unresolved

Inside a generic function or generic type, a conditional over an unresolved `T` is not evaluated — it is stored as a deferred type and resolved at the instantiation site. That is why the compiler cannot let you `return` from inside the branches.

```ts
type Wrapped<T> = T extends string ? { text: T } : { value: T };

function tryToImplement<T>(input: T): Wrapped<T> {
  if (typeof input === "string") {
    // `input` narrows to `T & string`, but `Wrapped<T>` does NOT narrow to `{ text: ... }`.
    // return { text: input };
    // ❌ Error: Type '{ text: T & string }' is not assignable to type 'Wrapped<T>'
  }
  return { value: input } as Wrapped<T>;   // the escape hatch
}

// Deferred conditionals also affect ASSIGNABILITY between two conditionals.
// TS can prove one conditional assignable to another only when the checked
// type, the extends type, and both branches are identical:
type SameA<T> = T extends string ? number : boolean;
type SameB<T> = T extends string ? number : boolean;
declare function takesB<T>(x: SameB<T>): void;
declare function passesA<T>(x: SameA<T>): void;
// takesB<T>(a) works for identical structures; anything less and you need a cast.
```

### `any` matches both branches; `never` matches zero

Two special inputs break the mental model, and both show up in real code (`any` from untyped libraries, `never` from failed lookups).

```ts
type Test<T> = T extends string ? "string" : "not-string";

type Anything = Test<any>;     // "string" | "not-string"   ← BOTH
type Nothing  = Test<never>;   // never                     ← NEITHER

// Real consequence: a utility built on a conditional silently degrades when a
// value flows in as `any` from an untyped dependency.
type IsArray<T> = T extends readonly unknown[] ? true : false;
declare const fromUntypedLib: any;
type Result = IsArray<typeof fromUntypedLib>;   // boolean — not an answer

// Defend explicitly if it matters:
type IsAny<T> = 0 extends 1 & T ? true : false;
type SafeIsArray<T> = IsAny<T> extends true
  ? never   // or "unknown" — make the ambiguity visible
  : T extends readonly unknown[] ? true : false;
```

### `infer` variance: covariant positions union, contravariant positions intersect

When the same `infer` name appears more than once, how the results combine depends on position.

```ts
// Covariant (return / property positions) → the candidates are UNIONED:
type Covariant<T> = T extends { a: infer U; b: infer U } ? U : never;
type CV = Covariant<{ a: string; b: number }>;   // string | number

// Contravariant (parameter positions) → the candidates are INTERSECTED:
type Contravariant<T> = T extends { a: (x: infer U) => void; b: (x: infer U) => void } ? U : never;
type CT = Contravariant<{ a: (x: string) => void; b: (x: number) => void }>;
// = string & number  = never

// This is the basis of the famous UnionToIntersection hack:
type UnionToIntersection<U> =
  (U extends unknown ? (arg: U) => void : never) extends (arg: infer I) => void ? I : never;
//  ^ distribute U into a union of FUNCTIONS taking each member,
//    then infer the single parameter → contravariance intersects them.

type Merged = UnionToIntersection<{ userId: number } | { authToken: string }>;
// = { userId: number } & { authToken: string }

// Practical use: merging plugin context types contributed by middleware.
type MiddlewareContribs =
  | { userId: number }
  | { tenantId: string }
  | { traceId: string };
type FullContext = UnionToIntersection<MiddlewareContribs>;
// = { userId: number } & { tenantId: string } & { traceId: string }
```

### Overload resolution and `ReturnType`: only the LAST signature is seen

`infer` on an overloaded function picks the last overload. This bites constantly with library functions.

```ts
declare function parseId(raw: string): number;
declare function parseId(raw: string, allowNull: true): number | null;

type Parsed = ReturnType<typeof parseId>;
// = number | null    ← the LAST overload won, not the one you probably meant

// If you need a specific overload's return type, index a tuple of signatures
// or restructure the API to use a conditional return type instead of overloads.
```

### Distribution and `keyof`: filtering keys by value type

The `[keyof T]` collapse from doc 43 depends entirely on distribution and on `never` vanishing.

```ts
interface ServiceContainer {
  userRepo:       { findById(userId: number): Promise<unknown> };
  orderRepo:      { findById(orderId: string): Promise<unknown> };
  cacheClient:    { get(key: string): Promise<string | null> };
  requestTimeout: number;
  serviceName:    string;
}

type KeysMatching<T, V> = { [K in keyof T]-?: T[K] extends V ? K : never }[keyof T];

type ObjectDeps  = KeysMatching<ServiceContainer, object>;
// = "userRepo" | "orderRepo" | "cacheClient"
type ScalarDeps  = Exclude<keyof ServiceContainer, ObjectDeps>;
// = "requestTimeout" | "serviceName"

// The `-?` is load-bearing: an optional property makes T[K] include undefined,
// which fails `extends V` and silently drops the key.

// The same via key remapping, usually easier to read:
type PickMatching<T, V> = { [K in keyof T as T[K] extends V ? K : never]: T[K] };
type Injectables = PickMatching<ServiceContainer, object>;
// = { userRepo: ...; orderRepo: ...; cacheClient: ... }
```

### Strict type equality (what test libraries actually use)

`[A] extends [B] ? [B] extends [A] ? true : false : false` treats `any` and `{}`-ish cases loosely. The strict version exploits the fact that the compiler compares *deferred conditional types* by identity, which is sensitive to `any`.

```ts
type Equals<A, B> =
  (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;

type E1 = Equals<string, string>;                 // true
type E2 = Equals<string, any>;                    // false  ← the loose version says true
type E3 = Equals<{ a: 1 }, { a: 1 }>;             // true
type E4 = Equals<{ a: 1 }, { a: 1; b?: 2 }>;      // false

// Use it to assert derived types in a "type test" file that fails the build:
type Expect<T extends true> = T;
type _check1 = Expect<Equals<Awaited<Promise<string>>, string>>;
type _check2 = Expect<Equals<Exclude<"a" | "b" | "c", "b">, "a" | "c">>;
// type _bad = Expect<Equals<Exclude<"a" | "b", "b">, "b">>;   // ❌ compile error
```

### Optional properties, `undefined`, and conditional checks

`exactOptionalPropertyTypes` changes what `T[K]` is, and therefore what your conditional sees.

```ts
interface PatchBody {
  email?:       string;
  displayName?: string;
  isAdmin?:     boolean;
}

// Without -?, T["email"] is `string | undefined`, which does NOT extend `string`:
type Bad = { [K in keyof PatchBody]: PatchBody[K] extends string ? K : never }[keyof PatchBody];
// = undefined   ← surprising and useless

type Good = { [K in keyof PatchBody]-?: PatchBody[K] extends string ? K : never }[keyof PatchBody];
// = "email" | "displayName"   ✅

// Or strip undefined in the condition instead:
type Good2 = {
  [K in keyof PatchBody]-?: NonNullable<PatchBody[K]> extends string ? K : never
}[keyof PatchBody];
```

### Performance: conditional types are cached, recursion is not free

```ts
// ⚠️ Each distinct instantiation is computed and cached. Deep recursion over a
// wide union multiplies: a 20-member union through a 10-deep recursive
// conditional is 200 instantiations, per call site that isn't cached.

// Mitigations that measurably help:

// 1. Name the result. `type Ctx = HandlerContext<K>` computes once; writing
//    HandlerContext<K> inline in ten signatures computes ten times.

// 2. Prefer tail-recursive shapes — the recursive call IS the whole branch:
type TailOk<T>  = T extends Promise<infer V> ? TailOk<V> : T;              // ✅ 1000 depth
type NotTail<T> = T extends Promise<infer V> ? [TailOk<V>] : [T];          // ⚠️ 50 depth

// 3. Bound recursion explicitly when the input depth is unbounded:
type Depth = [never, 0, 1, 2, 3, 4, 5];
type BoundedFlat<T, D extends number = 5> =
  D extends 0 ? T
  : T extends readonly (infer E)[] ? BoundedFlat<E, Depth[D]> : T;

// 4. Put an `interface` boundary around big computed object types — named
//    interfaces compare faster than large anonymous structures.

// The hard limits: ts(2589) "Type instantiation is excessively deep and
// possibly infinite" at ~50 levels (~1000 for tail-recursive conditionals),
// and ts(2590) "expression produces a union type that is too complex" when a
// distributed conditional generates more than ~100_000 union members.
```

### `infer` in template literal types

String parsing at the type level is just `infer` inside a template literal pattern.

```ts
type ParseAuthHeader<H extends string> =
  H extends `Bearer ${infer Token}` ? { scheme: "bearer"; authToken: Token }
  : H extends `Basic ${infer Encoded}` ? { scheme: "basic"; credentials: Encoded }
  : { scheme: "unknown" };

type AH1 = ParseAuthHeader<"Bearer eyJhbGciOi">;   // { scheme: "bearer"; authToken: "eyJhbGciOi" }
type AH2 = ParseAuthHeader<"Basic YWRtaW4=">;      // { scheme: "basic"; credentials: "YWRtaW4=" }
type AH3 = ParseAuthHeader<"Weird xyz">;           // { scheme: "unknown" }

// Splitting on a separator, recursively:
type Split<S extends string, Sep extends string> =
  S extends `${infer Head}${Sep}${infer Tail}` ? [Head, ...Split<Tail, Sep>] : [S];

type Segments = Split<"api/users/42/orders", "/">;   // ["api", "users", "42", "orders"]

// Numeric-literal parsing needs a lookup table — TS has no arithmetic on
// string digits, so `${infer N extends number}` (TS 4.8+) is the shortcut:
type PortOf<S extends string> = S extends `${string}:${infer P extends number}` ? P : never;
type PO = PortOf<"localhost:5432">;   // 5432
```

---

## Common mistakes

### Mistake 1 — Expecting `T extends U` not to distribute over a union

```ts
type EventPayload =
  | { type: "user.created"; userId: number }
  | { type: "payment.failed"; paymentId: string };

// ❌ "Does this union have a userId?" — distribution makes this meaningless:
type HasUserIdBad<T> = T extends { userId: number } ? true : false;
type Bad1 = HasUserIdBad<EventPayload>;
// = boolean   (true from the first member, false from the second, unioned)
// You will then write `if (someFlag)` against a `boolean` you thought was `true`.

// ❌ Same bug in a filter that was meant to be a check:
type RequireUserIdBad<T> = T extends { userId: number } ? T : never;
type Bad2 = RequireUserIdBad<EventPayload>;
// = { type: "user.created"; userId: number }
//   The payment member was SILENTLY DELETED, not rejected.

// ✅ Block distribution when you mean "the whole type, as one":
type HasUserId<T> = [T] extends [{ userId: number }] ? true : false;
type Good1 = HasUserId<EventPayload>;                          // false ✅
type Good2 = HasUserId<{ type: "user.created"; userId: number }>;  // true ✅

// ✅ And when you DO want a filter, name it like one so nobody is surprised:
type OnlyWithUserId<T> = T extends { userId: number } ? T : never;
```

### Mistake 2 — Checking for `never` with a naked type parameter

```ts
type LookupPayload<Events, Type> = Extract<Events, { type: Type }>;

// ❌ This never reports the failure, because distributing over `never` yields `never`:
type ValidateBad<T> = T extends never ? "no such event" : T;
type B1 = ValidateBad<LookupPayload<EventPayload, "order.placed">>;
// = never    ← you wanted "no such event"; you got silence

// The downstream effect: a function parameter typed `never` accepts nothing,
// and the error message points at the call site, not at the bad event name.

// ✅ Wrap in tuples to check the union as a whole:
type Validate<T> = [T] extends [never]
  ? { error: "no matching event type" }
  : T;
type G1 = Validate<LookupPayload<EventPayload, "order.placed">>;
// = { error: "no matching event type" }   ✅ a readable failure
type G2 = Validate<LookupPayload<EventPayload, "user.created">>;
// = { type: "user.created"; userId: number }   ✅
```

### Mistake 3 — Forgetting that `boolean` is `true | false`

```ts
interface FeatureFlags {
  enableBetaApi:  boolean;
  enableTracing:  boolean;
  maxRetries:     number;
}

// ❌ Intended: "give boolean flags an on/off toggle type". Actually distributes:
type ToggleBad<V> = V extends boolean ? { on: V; off: V } : V;
type TB = ToggleBad<boolean>;
// = { on: true; off: true } | { on: false; off: false }
//   Two useless variants. `{ on: true, off: false }` does not fit either one.

const bad: TB = { on: true, off: false };   // ❌ Error, and the message is baffling

// ✅ Block distribution:
type Toggle<V> = [V] extends [boolean] ? { on: boolean; off: boolean } : V;
type TG = Toggle<boolean>;                  // { on: boolean; off: boolean }
const good: TG = { on: true, off: false };  // ✅

// ❌ The same bug in a "which keys are booleans" query:
type BoolKeysBad<T> = { [K in keyof T]-?: T[K] extends true ? K : never }[keyof T];
type BKB = BoolKeysBad<FeatureFlags>;       // "enableBetaApi" | "enableTracing"
//   Works by ACCIDENT (each distributes and `true` matches one branch), but
//   `T[K] extends false` would also "work", which tells you it's not doing
//   what you think.

// ✅ Explicit and distribution-proof:
type BoolKeys<T> = { [K in keyof T]-?: [T[K]] extends [boolean] ? K : never }[keyof T];
type BK = BoolKeys<FeatureFlags>;           // "enableBetaApi" | "enableTracing" ✅
```

### Mistake 4 — Ordering nested conditionals from general to specific

```ts
// ❌ `Date`, arrays and functions are all objects, so this catches everything:
type SerializedBad<T> =
  T extends object   ? Record<string, unknown>
  : T extends Date   ? string
  : T extends unknown[] ? unknown[]
  : T;

type SB1 = SerializedBad<Date>;       // Record<string, unknown>   ← wrong
type SB2 = SerializedBad<string[]>;   // Record<string, unknown>   ← wrong

// ✅ Narrowest first, `object` last:
type Serialized<T> =
  T extends Date              ? string
  : T extends readonly (infer E)[] ? Serialized<E>[]
  : T extends Map<infer K, infer V> ? Array<[Serialized<K>, Serialized<V>]>
  : T extends Set<infer E>    ? Serialized<E>[]
  : T extends Function        ? never
  : T extends object          ? { [K in keyof T]: Serialized<T[K]> }
  : T;

interface AuditRecord {
  auditId:    string;
  occurredAt: Date;
  actor:      { userId: number; roles: Set<string> };
  changes:    Array<{ field: string; before: unknown; after: unknown }>;
}

type WireAudit = Serialized<AuditRecord>;
// = { auditId: string; occurredAt: string;
//     actor: { userId: number; roles: string[] };
//     changes: { field: string; before: unknown; after: unknown }[] }
```

### Mistake 5 — Trying to implement a conditional return type inside the function

```ts
type ParsedBody<Strict extends boolean> =
  Strict extends true ? Record<string, unknown> : unknown;

// ❌ Narrowing the VALUE does not narrow the TYPE PARAMETER:
function parseBad<Strict extends boolean>(raw: string, strict: Strict): ParsedBody<Strict> {
  const parsed = JSON.parse(raw) as unknown;
  if (strict) {
    if (typeof parsed !== "object" || parsed === null) throw new Error("expected object");
    // return parsed;
    // ❌ Error: 'Record<string, unknown>' is not assignable to 'ParsedBody<Strict>'
  }
  // return parsed;
  // ❌ Error: 'unknown' is not assignable to 'ParsedBody<Strict>'
  throw new Error("unreachable");
}

// ✅ Option A — one cast at the boundary, correct signature for callers:
function parseA<Strict extends boolean>(raw: string, strict: Strict): ParsedBody<Strict> {
  const parsed = JSON.parse(raw) as unknown;
  if (strict && (typeof parsed !== "object" || parsed === null)) {
    throw new HttpError(400, "request body must be a JSON object");
  }
  return parsed as ParsedBody<Strict>;
}

// ✅ Option B — overloads: precise for callers, loose inside:
function parseB(raw: string, strict: true): Record<string, unknown>;
function parseB(raw: string, strict: false): unknown;
function parseB(raw: string, strict: boolean): unknown {
  const parsed: unknown = JSON.parse(raw);
  if (strict && (typeof parsed !== "object" || parsed === null)) {
    throw new HttpError(400, "request body must be a JSON object");
  }
  return parsed;
}

// ✅ Option C — no conditional at all; two functions with honest names:
function parseJsonObject(raw: string): Record<string, unknown> { /* ... */ return {}; }
function parseJsonValue(raw: string): unknown { return JSON.parse(raw); }
// Often the best answer. A conditional return type earns its complexity only
// when the branch count is high or the type is derived from a table.
```

### Mistake 6 — Using `Extract`/`Exclude` where the constraint should be `keyof`

```ts
interface UserEntity2 {
  userId:       number;
  email:        string;
  passwordHash: string;
  mfaSecret:    string;
}

// ❌ Exclude accepts ANY type for U — a typo silently excludes nothing:
type PublicKeysBad = Exclude<keyof UserEntity2, "passwordHash" | "mfaSecrets">;
//                                                                       ^ typo
// = "userId" | "email" | "mfaSecret"   ← the secret is still in the type. No error.

// ✅ Constrain the exclusion set to the actual keys:
type ExcludeKeys<T, K extends keyof T> = Exclude<keyof T, K>;
// type PublicKeysBad2 = ExcludeKeys<UserEntity2, "passwordHash" | "mfaSecrets">;
// ❌ Error: '"passwordHash" | "mfaSecrets"' does not satisfy 'keyof UserEntity2'  ✅

type PublicKeys = ExcludeKeys<UserEntity2, "passwordHash" | "mfaSecret">;
// = "userId" | "email"   ✅ and typos are now compile errors
```

### Mistake 7 — Assuming `ReturnType` sees the overload you care about

```ts
declare function sendNotification(userId: number): Promise<{ messageId: string }>;
declare function sendNotification(userIds: number[]): Promise<{ messageIds: string[] }>;

// ❌ Only the last overload is inferred:
type Sent = ReturnType<typeof sendNotification>;
// = Promise<{ messageIds: string[] }>
//   Downstream code that expected `messageId` fails in a confusing place.

// ✅ Don't derive from overloaded functions. Name the shapes explicitly:
interface SingleNotificationResult { messageId: string }
interface BatchNotificationResult  { messageIds: string[] }

declare function sendNotification2(userId: number): Promise<SingleNotificationResult>;
declare function sendNotification2(userIds: number[]): Promise<BatchNotificationResult>;
// Now callers derive from the named interfaces, not from `typeof`.

// ✅ Or replace the overloads with a conditional return type, which IS derivable:
type NotificationResult<Target> = Target extends readonly number[]
  ? BatchNotificationResult
  : SingleNotificationResult;

declare function sendNotification3<Target extends number | readonly number[]>(
  target: Target,
): Promise<NotificationResult<Target>>;

type S1 = Awaited<ReturnType<typeof sendNotification3<number>>>;      // SingleNotificationResult
type S2 = Awaited<ReturnType<typeof sendNotification3<number[]>>>;    // BatchNotificationResult
```

---

## Practice exercises

### Exercise 1 — easy

Write each of these conditional types from scratch. Do **not** use any built-in utility type (no `Exclude`, `Extract`, `ReturnType`, `Awaited`, `NonNullable`).

```ts
// 1.  MyExclude<T, U>      — remove from union T every member assignable to U
// 2.  MyExtract<T, U>      — keep only members assignable to U
// 3.  MyNonNullable<T>     — remove null and undefined from a union
// 4.  MyReturnType<T>      — the return type of a function type
// 5.  MyParameters<T>      — the parameter tuple of a function type
// 6.  MyAwaited<T>         — recursively unwrap nested Promises
// 7.  ElementOf<T>         — the element type of an array OR readonly array
// 8.  IsNever<T>           — true only when T is exactly never
// 9.  IsExactly<A, B>      — true when A and B are mutually assignable
// 10. FirstParam<T>        — the first parameter's type, or never

// Then verify each against this material:

interface SessionRecord {
  sessionId:  string;
  userId:     number;
  authToken:  string;
  expiresAt:  Date;
  revokedAt:  Date | null;
}

type HttpStatus = 200 | 201 | 400 | 401 | 403 | 404 | 500;

declare function createSession(
  userId: number,
  ttlSeconds: number,
): Promise<SessionRecord>;

declare function listSessions(userId: number): Promise<SessionRecord[]>;

// Prove that:
//   MyExclude<HttpStatus, 200 | 201>            is 400 | 401 | 403 | 404 | 500
//   MyExtract<HttpStatus, 400 | 401 | 403>      is 400 | 401 | 403
//   MyNonNullable<SessionRecord["revokedAt"]>   is Date
//   MyAwaited<MyReturnType<typeof createSession>>          is SessionRecord
//   ElementOf<MyAwaited<MyReturnType<typeof listSessions>>> is SessionRecord
//   MyParameters<typeof createSession>          is [userId: number, ttlSeconds: number]
//   FirstParam<typeof createSession>            is number
//   IsNever<MyExtract<HttpStatus, 418>>         is true
//   IsExactly<MyExclude<HttpStatus, never>, HttpStatus> is true
//
// Finally, write a helper `assertType<Expected, Actual>()` that fails to
// compile when the two are not exactly equal, and use it for all of the above.
```

```ts
// Write your code here
```

### Exercise 2 — medium

Build a **type-safe cache layer** where the value type of every cache key is derived from the loader function that produces it, and where key strings are parsed at the type level.

```ts
// Given these loaders (write more if you like):

const cacheLoaders = {
  "user": async (userId: number) => ({
    userId,
    email:       "ada@example.com",
    displayName: "Ada",
    isAdmin:     false,
  }),
  "order": async (orderId: string) => ({
    orderId,
    userId:     42,
    totalCents: 4999,
    status:     "paid" as const,
  }),
  "session": async (sessionId: string) => ({
    sessionId,
    userId:    42,
    authToken: "tok_abc",
    expiresAt: new Date(),
  }),
  "feature-flags": async () => ({
    enableBetaApi: true,
    enableTracing: false,
  }),
};

// Implement:
//
// 1. LoaderName            — the union of loader keys.
//
// 2. LoaderArgs<Name>      — the parameter tuple of that loader (via infer).
//
// 3. CachedValue<Name>     — the resolved value type of that loader.
//    Must recursively unwrap Promises; do not use the built-in Awaited.
//
// 4. CacheKey<Name>        — a template literal type `${Name}:${string}` for
//    loaders that take an argument, and exactly `${Name}` for loaders that
//    take none. (Hint: `LoaderArgs<Name> extends [] ? ... : ...`)
//
// 5. ParseCacheKey<Key>    — given a literal key string like "user:42", produce
//    { name: "user"; id: "42" }, and for "feature-flags" produce
//    { name: "feature-flags"; id: never }. Handle the fact that some loader
//    names contain a hyphen but never a colon.
//
// 6. ValueForKey<Key>      — given a literal key string, produce the cached
//    value type, by combining ParseCacheKey and CachedValue. If the name is
//    not a real loader, produce { error: "unknown cache key"; key: Key }
//    instead of never (use the [T] extends [never] idiom).
//
// 7. A `TypedCache` class with:
//        get<Key extends string>(key: Key): Promise<ValueForKey<Key> | null>
//        set<Key extends string>(key: Key, value: ValueForKey<Key>, ttlSeconds?: number): Promise<void>
//        load<Name extends LoaderName>(name: Name, ...args: LoaderArgs<Name>): Promise<CachedValue<Name>>
//        invalidate<Name extends LoaderName>(name: Name): Promise<number>
//
// 8. StaleWhileRevalidate<Name> — a conditional type producing
//        { value: CachedValue<Name>; stale: boolean; refreshedAt: Date }
//    but ONLY for loaders whose value is an object; for any other value type
//    it should produce a compile-visible error type.
//
// Demonstrate:
//   - cache.get("user:42")            resolves to the user shape | null
//   - cache.get("feature-flags")      resolves to the flags shape | null
//   - cache.get("nonsense:1")         has the error shape (not never)
//   - cache.load("order", "ord_1")    is checked: passing a number is an error
//   - cache.load("feature-flags")     compiles with NO extra arguments
//   - cache.set("user:42", { userId: 1 })  is a compile error (incomplete value)
```

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **type-level SQL query result inferencer**: given a table schema and a query description, compute the exact row type the query returns — including joins, nullable columns from LEFT JOINs, aliases, and aggregates.

```ts
// Start from these schemas:

interface UsersTable {
  user_id:      number;
  email:        string;
  display_name: string;
  is_admin:     boolean;
  created_at:   Date;
  deleted_at:   Date | null;
}

interface OrdersTable {
  order_id:     string;
  user_id:      number;
  total_cents:  number;
  status:       "pending" | "paid" | "shipped" | "cancelled";
  placed_at:    Date;
  shipped_at:   Date | null;
}

interface OrderItemsTable {
  order_item_id: number;
  order_id:      string;
  sku:           string;
  quantity:      number;
  unit_cents:    number;
}

interface Database {
  users:       UsersTable;
  orders:      OrdersTable;
  order_items: OrderItemsTable;
}

// Build ALL of the following using conditional types and infer:
//
// 1. SnakeToCamel<S>       — "display_name" -> "displayName", recursively, at the
//    type level. Then CamelRow<T> — a mapped type applying it to every key.
//
// 2. ColumnRef<DB, Table>  — the union of `${Table}.${keyof DB[Table] & string}`
//    for a given table name.
//
// 3. ResolveColumn<DB, Ref> — given "orders.total_cents", produce number.
//    Given a bad reference, produce { error: "unknown column"; ref: Ref }.
//
// 4. AliasedSelection      — a selection is a readonly array whose entries are
//    either a plain ColumnRef, or `${ColumnRef} AS ${string}`. Write
//    ParseSelection<Entry> producing { ref: ...; alias: ... } where alias
//    defaults to the column name (camel-cased) when no AS clause is present.
//
// 5. SelectionRow<DB, Sel> — fold a readonly tuple of selection entries into a
//    single object type: { [alias]: ResolveColumn<...> }. You will need
//    recursion over the tuple plus UnionToIntersection or an accumulator.
//
// 6. JoinKind = "INNER" | "LEFT" | "RIGHT". Write
//    ApplyJoinNullability<Row, Kind, Side> so that a LEFT join makes every
//    column of the RIGHT-hand table `| null`, a RIGHT join does the reverse,
//    and INNER changes nothing.
//
// 7. Aggregate handling: a selection entry may be
//      `COUNT(${ColumnRef})` -> number
//      `SUM(${ColumnRef})`   -> number | null
//      `MAX(${ColumnRef})`   -> the column's type | null
//    Extend ParseSelection and ResolveColumn to handle these, and require an
//    alias for aggregate entries (no alias => a compile-visible error type).
//
// 8. A `query()` function:
//      declare function query<
//        Table extends keyof Database,
//        Sel extends readonly string[],
//        Joins extends readonly JoinSpec[] = [],
//      >(spec: { from: Table; select: Sel; joins?: Joins; where?: ... }):
//        Promise<CamelRow<SelectionRow<Database, Sel, Joins>>[]>;
//
// Demonstrate that:
//
//   query({ from: "users", select: ["users.user_id", "users.email"] })
//     -> Array<{ userId: number; email: string }>
//
//   query({
//     from: "orders",
//     select: ["orders.order_id", "orders.total_cents AS amount", "users.email"],
//     joins: [{ kind: "LEFT", table: "users", on: ["orders.user_id", "users.user_id"] }],
//   })
//     -> Array<{ orderId: string; amount: number; email: string | null }>
//        (email is nullable because of the LEFT join)
//
//   query({ from: "orders", select: ["COUNT(orders.order_id) AS orderCount"] })
//     -> Array<{ orderCount: number }>
//
//   query({ from: "users", select: ["users.emial"] })
//     -> a compile error mentioning "unknown column", NOT a silent never
//
//   query({ from: "orders", select: ["SUM(orders.total_cents)"] })
//     -> a compile error demanding an alias
//
// Bonus: add ORDER BY validation — the order key must be one of the produced
// aliases, checked with Extract and reported with a friendly error type when not.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── The form ────────────────────────────────────────────────────────────────
type Cond<T> = T extends U ? TrueBranch : FalseBranch;   // "is T assignable to U?"

// ── Nesting = switch ────────────────────────────────────────────────────────
type Kind<T> =
  T extends string  ? "string"
  : T extends number ? "number"
  : "other";                       // narrowest checks FIRST

// ── infer patterns you will use weekly ──────────────────────────────────────
type ElementOf<T>  = T extends readonly (infer E)[]              ? E : never;
type ResolvedOf<T> = T extends Promise<infer V>                  ? V : never;
type ReturnOf<T>   = T extends (...args: never[]) => infer R     ? R : never;
type ParamsOf<T>   = T extends (...args: infer P) => unknown     ? P : never;
type FirstOf<T>    = T extends (first: infer A, ...rest: never[]) => unknown ? A : never;
type InstanceOf<T> = T extends abstract new (...a: never[]) => infer I ? I : never;
type PayloadOf<T>  = T extends { payload: infer P }              ? P : never;
type HeadOf<T>     = T extends readonly [infer H, ...unknown[]]  ? H : never;
type TailOf<T>     = T extends readonly [unknown, ...infer R]    ? R : never;
type LastOf<T>     = T extends readonly [...unknown[], infer L]  ? L : never;

// ── Distribution: naked T + union argument = runs per member ────────────────
type Filter<T, U>  = T extends U ? T : never;       // = Extract<T, U>
type Reject<T, U>  = T extends U ? never : T;       // = Exclude<T, U>
type Map_<T>       = T extends string ? `id_${T}` : never;

// ── Block distribution with tuple brackets ──────────────────────────────────
type Whole<T>      = [T] extends [U] ? A : B;
type IsNever<T>    = [T] extends [never]   ? true : false;
type IsBool<T>     = [T] extends [boolean] ? true : false;

// ── Special inputs ──────────────────────────────────────────────────────────
// any   → returns BOTH branches unioned
// never → returns never (zero distribution steps)
// boolean → distributes as true | false
type IsAny<T>     = 0 extends 1 & T ? true : false;
type IsUnknown<T> = IsAny<T> extends true ? false : unknown extends T ? true : false;
type Equals<A, B> = (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;

// ── Hand-built standard library ─────────────────────────────────────────────
type MyExclude<T, U>     = T extends U ? never : T;
type MyExtract<T, U>     = T extends U ? T : never;
type MyNonNullable<T>    = T extends null | undefined ? never : T;
type MyReturnType<T>     = T extends (...a: never[]) => infer R ? R : never;
type MyParameters<T>     = T extends (...a: infer P) => unknown ? P : never;
type MyAwaited<T>        = T extends Promise<infer V> ? MyAwaited<V> : T;

// ── Recursive ───────────────────────────────────────────────────────────────
type DeepFlat<T>   = T extends readonly (infer E)[] ? DeepFlat<E> : T;
type Split<S extends string, Sep extends string> =
  S extends `${infer H}${Sep}${infer T}` ? [H, ...Split<T, Sep>] : [S];

// ── Conditional return types ────────────────────────────────────────────────
declare function find<Single extends boolean = false>(
  where: object, single?: Single,
): Promise<Single extends true ? Row | null : Row[]>;
// Great for callers. Inside the body the conditional is DEFERRED — cast once
// at the return, or use overloads.

// ── The union → intersection hack ───────────────────────────────────────────
type UnionToIntersection<U> =
  (U extends unknown ? (a: U) => void : never) extends (a: infer I) => void ? I : never;

type U = unknown; type A = unknown; type B = unknown; type Row = unknown;
```

| Construct | Meaning |
|---|---|
| `T extends U ? X : Y` | Is `T` **assignable to** `U`? Not inheritance — structural assignability |
| `infer N` | Declare a type variable in the pattern; usable in the **true branch only** |
| `infer N extends C` | Constrained inference (TS 4.7+) — fails the match if `N` isn't a `C` |
| Naked `T` + union arg | **Distributes**: runs once per member, results unioned |
| `[T] extends [U]` | **Blocks** distribution — checks the whole union at once |
| `T extends U ? T : never` | `Extract<T, U>` — filter a union |
| `T extends U ? never : T` | `Exclude<T, U>` — reject from a union |
| `X \| never` | Just `X` — which is why `never` deletes union members |
| `{ k: never }` | **Not** deleted — poisons the object; use `as never` key remap to drop it |
| `any` as the checked type | Returns **both** branches, unioned |
| `never` as the checked type | Returns `never` (zero distribution steps) |
| `boolean` as the checked type | Distributes as `true \| false` |
| Same `infer` twice, covariant | Candidates **unioned** |
| Same `infer` twice, contravariant | Candidates **intersected** (basis of `UnionToIntersection`) |
| `ReturnType` on overloads | Sees only the **last** overload signature |
| Recursion depth | ~50 levels; ~1000 when the recursive call is the entire branch |
| Error `ts(2589)` | "Type instantiation is excessively deep" — bound the recursion |
| Error `ts(2590)` | Union too complex — distribution produced too many members |

---

## Connected topics

- **43 — Mapped types** — conditional types live in a mapped type's value slot (`Serialize<T>`, `Comparators<V>`) and in its `as` clause to drop keys; the two features are almost always used together.
- **32 — Utility types** — `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `InstanceType` and `Awaited` are the conditional types you get for free; this doc rebuilds every one of them.
- **31 — keyof and indexed access types** — `keyof T`, `T[K]` and the `{...}[keyof T]` collapse are what conditional filtering operates on.
- **42 — Discriminated unions** — `Extract<Event, { type: "payment.failed" }>` is the standard way to look up one variant by its tag; distribution is what makes it work.
- **45 — Template literal types** — `infer` inside a template pattern is how route params, cache keys and header parsing are done at the type level.
- **46 — Variance and assignability** — why `extends` answers differently in parameter positions, and why the same `infer` name intersects there instead of unioning.
- **41 — Generics and constraints** — conditional return types only pay off when the type parameter is inferred from an argument; constraints control what can reach the condition.
